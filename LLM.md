# LLM.md — homelab Kubernetes Cluster

> **Sources of truth — read these BEFORE reasoning or claiming anything about the
> cluster.** This file is an operational quick-reference, NOT the authority on
> architecture or deployed state. When a question touches either, open the actual
> source first; do not reconstruct from memory or a partial read:
> - **Architecture & the "why"** → `doc/arc42-homelab.adoc` (ADRs, quality goals,
>   building-block view). Authoritative for design decisions.
> - **Deployed node state** → `talos/<node>-machine.yaml`, `talos/homelab-*.yaml`
>   patches, `talos/<node>-schematics.yaml`. Authoritative for what a node actually
>   runs (install image, kubelet, NIC, labels, endpoint, k8s component images).
> - **Live cluster** → `kubectl` / `talosctl get` / `ceph -s`. Ground truth; beats
>   any doc when they disagree — then fix the doc.

## Cluster Overview

Two-node Talos Linux cluster (1 CP + 1 worker) on Acemagic F5A (AMD HX370, 96GB
RAM) + Minisforum MS-A2, expanding to 3 nodes.

> **Version currency rule**: The values below (Talos/K8s/Cilium versions, node
> count, OSD count, endpoint, failure domain) are load-bearing operational facts,
> NOT decoration. On EVERY component upgrade (e.g. the pending Talos 1.13.5) or
> topology change, update this table and the affected sections in the SAME change
> — never let CLAUDE.md drift behind reality. Verify against the live cluster
> (`kubectl get nodes -o wide`, `ceph -s`, `talosctl version`) before editing.

| Component       | Details                                                    |
|-----------------|------------------------------------------------------------|
| OS              | Talos Linux 1.13.5                                         |
| Kubernetes      | 1.36.2                                                     |
| CNI             | Cilium 1.19.1 (kube-proxy replacement, WireGuard encryption, Hubble UI) |
| Storage         | Rook-Ceph, 4× Samsung 990 PRO 4TB NVMe as OSDs (2 per node)|
| Filesystem      | CephFS, StorageClass `rook-cephfs`                         |
| Ceph replication| `size: 2`, failure domain `host` (transitioned from `osd` when ms-a2-01 joined) |
| Nodes           | `f5a-01` at `192.168.47.40` (CP), `ms-a2-01` at `192.168.47.39` (worker) |
| Control Plane   | Single CP f5a-01. Canonical endpoint `https://192.168.47.40:6443` (SSOT: `talos/homelab-endpoint.yaml`). VIP `192.168.47.240` is configured on enp197s0 and usable as a client access path — but NOT promoted to the canonical endpoint until it survives a deliberate cold-start test (see Operational Notes) |
| LoadBalancer    | Cilium LB-IPAM + L2 Announcements, pool `192.168.47.200-209` |
| Ingress         | None yet                                                   |
| Secrets         | SOPS + age (field-level encryption in Git); cluster-side decryption TBD |

- **Talos 1.13 reproducible images**: Same build → identical disk image (raw images;
  VHD/VMDK excluded). Reinforces the schematic-pinned upgrade workflow.
- **Talos 1.13 enables CDI by default**; extension services can drop CDI specs under
  /run/cdi. This is the path for exposing the AMD iGPU/NPU to LLM pods (llama.cpp +
  ROCm/Vulkan) on HX-class nodes. Kernel is now 6.18.x (good for recent amdgpu).
- **f5a-01 LLM paths (HX 370)** — two independent options, see `doc/amd-npu.md`:
  - *iGPU/Vulkan* (Radeon 890M, `gfx1150`): works today, `amdgpu` bound,
    `/dev/dri/renderD128` + KFD compute node present. No kernel change needed.
  - *NPU/FastFlowLM* (XDNA2, Strix Point, PCI `1022:17f0`): one kernel-config flip
    away. Firmware (`amdnpu/17f0_10`) ships in the Talos base, XRT userspace runs
    in the FLM container — but `amdxdna.ko` is NOT built (`CONFIG_DRM_ACCEL_AMDXDNA`
    unset). XDNA2 is the FastFlowLM-supported tier; XDNA1 (Phoenix, PCI `1502`) is not.

### Secrets Management

* SOPS + age for field-level encryption — see `doc/secrets-management.md`
* `.sops.yaml` in repo root defines encryption rules per path
* age private key at `~/.config/sops/age/keys.txt` (never in repo)

## Network

- LAN: `192.168.47.0/24`
- Gateway/DNS: FritzBox at `192.168.47.1`
- DHCP range: `.20` – `.199`
- Cilium LB-IPAM pool: `192.168.47.200` – `192.168.47.209`
- Virtual IPs: `192.168.47.240` – `192.168.47.249` (Talos VIP etc.)
- Control Plane VIP: `192.168.47.240`

## Storage Tiering: Persistent vs. Volatile (per-disk, not per-node)

The cluster's core storage rule is NOT "storage nodes vs. compute nodes". It is:

- **Persistent / CephFS-friendly**: Services with replicated state (Forgejo, Nexus,
  PostgreSQL, SonarQube, registries). Live on Ceph OSDs.
- **Volatile / CephFS-hostile**: Compile jobs (Yocto, OCI image builds) generate
  millions of small temp files / hardlink churn — toxic to CephFS MDS and replication.
  MUST live on LOCAL scratch storage, NEVER on Ceph. Never mirrored.
- **Compute-heavy / CephFS-neutral**: LLM inference. Model weights are read-once from
  Ceph (RWX, fine), then resident in RAM. Memory-bandwidth-bound, no persistent write.
- **Disk selection for volumes uses CEL `diskSelector`** (Talos 1.13). Match the three
  internal NVMe by stable properties — and ALWAYS scope with `disk.transport == "nvme"`
  so the USB-C boot SSD (`/dev/sda`) can never be matched by an OSD or scratch selector.
  Within the NVMe set: match by `disk.model` to separate the two Samsung 990 PRO (OSDs)
  from the differently-modelled scratch NVMe; use `disk.serial`/`disk.wwid` to
  distinguish the two identical 990 PRO from each other. Size constants need the
  unsigned suffix `u` (e.g. `disk.size > 100u * GiB`). Inspect real values first with
  `talosctl get disks -o yaml`.

KEY INSIGHT: This boundary runs between DISKS, not between MACHINES. A single node may
host Ceph OSDs (persistent) AND a dedicated local NVMe (volatile scratch) simultaneously,
as long as the disks are deterministically separated via CEL diskSelector. Build
artifacts are persisted by an EXPLICIT CI step (push to registry) — never by an
automatic cluster mechanism. The cluster never "knows" about the volatile cache.

OPEN TENSION (to resolve when scheduling volatile workloads): a build pod needing the
local scratch NVMe must land on the node that physically has it — which conflicts with
the "No nodeSelector/nodeAffinity" rule in Conventions. Local-disk affinity vs. pod
mobility is an unresolved design point. Flag, don't silently violate.

## Node Expansion

Full roster, hardware specs, and rationale (ADR-H-003…007, incl. the voided NUC
plan) live in `doc/arc42-homelab.adoc` (Bausteinsicht Ebene 1 + ADRs). Only
operational status + the bits not captured there:

- **Phase 2 — MS-A2 (`ms-a2-01`): DONE.** Worker, not CP (avoids the 2-node etcd
  quorum trap; CP promotion happens in Phase 3, 1→3 in one step). 2× 990 PRO as
  OSDs; 3rd M.2 reserved for local volatile scratch (UserVolumeConfig, never a
  Ceph OSD). Cluster NIC = Realtek `enp3s0` (ADR-H-005), DHCP fixlease `.39` bound
  to its MAC. Boot: external USB-C NVMe (RTL9210 bridge) — use the *non-USB4*
  Type-C port; the `usb-storage.quirks` kernel arg is pinned in
  `talos/ms-a2-01-schematics.yaml` (USB4 port renegotiation-loops with the bridge).
  Result: Ceph failure domain transitioned `osd` → `host`; still single CP
  (endpoint/VIP status → Cluster Overview table).
- **Phase 3 — SER6 Pro:** pending. Joins *together with* promoting MS-A2 to CP
  (1→3 etcd in one step). Canonical endpoint moves to VIP `.240` once 3-CP is
  healthy AND the VIP has passed a cold-start test.
- **Phase 4 — MS-02 Ultra (tentative):** more OSDs toward `size: 3`. Don't plan
  around it until the MS-A2 setup proves itself.

**Disaster recovery**: `doc/disaster-recovery.md` — MS-A2 promotable to CP under
f5a-01 total failure; no new hardware required.

## Repo Structure

```
homelab/
├── talos/              # Talos machine configs, patches, secrets, schematics — NOT applied via kubectl
│   └── <node>-schematics.yaml # Image Factory schematic IDs/extensions per node
├── infrastructure/     # Cluster infrastructure (storage, networking)
│   ├── rook-ceph/      # Rook-Ceph operator, CephCluster, StorageClass
│   └── cilium-lb/          # LB-IPAM pool + L2 announcement policy
└── applications/       # Workloads
    └── forgejo/        # Git forge (first workload)
```

## Conventions

- **Deployment mechanism per component type**:
  - *Helm* for complex operator/CNI charts — **Cilium** (`infrastructure/cilium/`)
    and **Rook-Ceph operator** (`infrastructure/rook-ceph/`), future Ingress. Chart
    version pinned, `values.yaml` committed as SSOT; each dir's `README.md` carries
    the exact `helm upgrade --install` command.
  - *Plain YAML* (`kubectl apply -f <directory>/`) for our own CRs and config —
    CephCluster/CephFilesystem/StorageClass, LB-IPAM pool, L2 policy, workloads.
  - *Kustomize* is allowed where it earns its keep, not mandated.
- **Labels**: Use `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`.
- **No nodeSelector/nodeAffinity** on workloads — pods must be freely schedulable across nodes.
- **StorageClass**: Always `rook-cephfs` for PVCs.
- **Secrets**: Real values in YAML, field-level encrypted via SOPS+age. Edit with `sops <file>`, decrypt with `sops -d <file>`.
- **Namespaces**: `soho` for business/SOHO workloads (Forgejo, CI). Future: per-customer namespaces for VPN isolation.
- **Talos upgrades**: Always use the per-node Image Factory installer from the node's `talos/<node>-schematics.yaml`, never the generic `ghcr.io/siderolabs/installer`. This preserves extensions and the schematic ID in META.
- **Node labels for storage tiering**: every node declares its resource-capability
  axes via three boolean `homelab.local/*` labels set in its MachineConfig
  `nodeLabels`. This makes the per-disk storage tiering visible from the K8s API
  without violating the no-nodeSelector/affinity rule on workloads — the labels
  are descriptive metadata, scheduling mechanism (taints? PV node-affinity?)
  remains TBD per the OPEN TENSION above.
  - `homelab.local/ceph-osd`: node hosts Ceph OSDs (persistent storage)
  - `homelab.local/scratch-nvme`: node has dedicated local volatile NVMe
  - `homelab.local/llm-capable`: usable LLM inference path (iGPU and/or NPU). On
    f5a-01 this is the Radeon 890M iGPU (Vulkan today) AND the XDNA2 NPU (one kernel
    flip away via FastFlowLM); see `doc/amd-npu.md`. NOT NPU-gated — ms-a2/ser6 are
    NPU-less. iGPU/memory-bandwidth alone qualifies.
  Current matrix (ceph-osd, scratch-nvme, llm-capable):
  f5a-01 = (true, false, true); ms-a2-01 = (true, true, false);
  ser6 = (false, true, false); ms-02 = tbd.
- Refer to existing YAMLs in `infrastructure/rook-ceph/` for formatting style.

## Design Principles

- **Cattle, not pets**: Every node re-provisionable from config. No manual state on nodes.
- **Pod mobility**: Workloads must survive node rescheduling. Use LoadBalancer services (Cilium LB-IPAM), not NodePort.
- **CNI before storage**: Cilium must be in place before Rook-Ceph. Never replace CNI while Ceph holds data.
- **Boot drives ≠ Ceph OSDs**: Boot media is always separate from Ceph storage.
- **Declarative**: All config in git. Secrets encrypted (SOPS+age). Target: `git clone` + age key + `sops -d | kubectl apply` = running cluster.
- **CP topology is orthogonal to storage tiering**: a node's K8s role (control plane vs worker) is driven by etcd quorum needs (odd numbers, currently 1→3 jump), not by what disks it holds. Storage tiering rules apply equally to CP and worker nodes.

## Current Deployment Order

1. ✅ Talos + Cilium
2. ✅ Rook-Ceph + CephFS StorageClass
3. ⬜ Cilium LB-IPAM + L2 Announcements (`infrastructure/cilium-lb/`)
4. ⬜ Forgejo + PostgreSQL (`applications/forgejo/`)
5. ⬜ MCP pods (concept: Git repos as mounted context, no RAG/vector stores)
6. ⬜ Per-tenant web spaces for rendered documentation

## Useful Commands

```bash
# Talos node management
talosctl -n 192.168.47.40 get disks
talosctl -n 192.168.47.40 get discoveredvolumes

# Ceph status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph status
kubectl -n rook-ceph exec deploy/rook-ceph-tools -- ceph osd tree

# Watch pod status
kubectl get pods -A -w
```

## Operational Notes

- `talosctl disks` is deprecated — use `talosctl get disks/discoveredvolumes/systemdisk`.
- `--insecure` flag required for `talosctl` in Maintenance Mode.
- Talos install disk config uses `disk: /dev/sda`, not CEL expressions.
- Mixed patch types in a single `--config-patch` file are unreliable in Talos.
- **Reboot on single-node Ceph**: `talosctl reboot` hangs if CephFS volumes are mounted — the mount waits for the local mon which is already shutting down. Workaround: drain CephFS pods first, or use ACPI power-off (short press power button). Never change the control plane endpoint to an unproven address on a running cluster.
- **`talosctl upgrade` MUST use `--drain=false` on Ceph mon/CP nodes**: the default `--drain=true` cordon+evict hangs forever on a node hosting a Ceph mon — the mon's PodDisruptionBudget blocks eviction (can't drop the only mon without losing quorum), so the upgrade never reaches image-pull/reboot. Symptom: machine stage stays `running`, no installer activity, `talosctl upgrade` produces no output. With `--drain=false` Talos stops pods via its own shutdown sequence instead. Still drain *workload* CephFS pods (e.g. Forgejo `scale --replicas=0`) manually beforehand to avoid the mount-hang above; the rook-ceph daemons (mon/mds/osd) come back on their own after the reboot.
- **talosctl version pinning**: Pin `siderolabs/tap/talosctl` in brew (`brew pin`) to match the server version. Cross-version gRPC keepalive issues between 1.13.x client and 1.12.x server cause upgrade failures.
- **Talos VIP on single-node**: VIP works but the operator resigns during (even failed) upgrades and doesn't re-campaign in 1.12.x. Fixed in 1.13.x. Never point the endpoint to VIP until VIP has survived a full reboot cycle.
