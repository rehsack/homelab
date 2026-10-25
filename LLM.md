# LLM.md — homelab Kubernetes Cluster

> **Sources of truth — read these BEFORE reasoning or claiming anything about the cluster.** This file is an operational
> quick-reference, NOT the authority on architecture or deployed state. When a question touches either, open the actual
> source first; do not reconstruct from memory or a partial read:
>
> - **Architecture & the "why"** → `doc/arc42-homelab.adoc` (ADRs, quality goals, building-block view). Authoritative
>   for design decisions.
> - **Deployed node state** → `talos/<node>-machine.yaml`, `talos/homelab-*.yaml` patches,
>   `talos/<node>-schematics.yaml`. Authoritative for what a node actually runs (install image, kubelet, NIC, labels,
>   endpoint, k8s component images).
> - **Live cluster** → `kubectl` / `talosctl get` / `ceph -s`. Ground truth; beats any doc when they disagree — then fix
>   the doc.

## Cluster Overview

Two-node Talos Linux cluster (1 CP + 1 worker) on Acemagic F5A (AMD HX370, 96GB RAM) + Minisforum MS-A2, expanding to 3
nodes.

> **Version currency rule**: The values below (Talos/K8s/Cilium versions, node count, OSD count, endpoint, failure
> domain) are load-bearing operational facts, NOT decoration. On EVERY component upgrade (e.g. the pending Talos 1.13.5)
> or topology change, update this table and the affected sections in the SAME change — never let CLAUDE.md drift behind
> reality. Verify against the live cluster (`kubectl get nodes -o wide`, `ceph -s`, `talosctl version`) before editing.

| Component        | Details                                                                               |
| ---------------- | ------------------------------------------------------------------------------------- |
| OS               | Talos Linux 1.13.12                                                                   |
| Kubernetes       | 1.36.2                                                                                |
| CNI              | Cilium 1.20.4 (kube-proxy replacement, WireGuard encryption, Hubble UI)               |
| Storage          | Rook-Ceph, 4× Samsung 990 PRO 4TB NVMe as OSDs (2 per node)                           |
| StorageClasses   | `rook-cephfs` + `rook-ceph-block` — which one when: arc42 ADR-H-009                   |
| Ceph replication | `size: 2`, failure domain `host` (transitioned from `osd` when ms-a2-01 joined)       |
| Nodes            | `f5a-01` at `192.168.47.40` (CP), `ms-a2-01` at `192.168.47.39` (worker)              |
| Control Plane    | Single CP f5a-01, endpoint `192.168.47.40:6443` (SSOT: `talos/homelab-endpoint.yaml`) |
| LoadBalancer     | Cilium LB-IPAM + L2 Announcements, pool `192.168.47.200-209`                          |
| Ingress          | None yet                                                                              |
| Secrets          | SOPS + age (field-level encryption in Git); cluster-side decryption TBD               |

- **Talos 1.13 reproducible images**: Same build → identical disk image (raw images; VHD/VMDK excluded). Reinforces the
  schematic-pinned upgrade workflow.
- **Talos 1.13 enables CDI by default**; extension services can drop CDI specs under /run/cdi. This is the path for
  exposing the AMD iGPU/NPU to LLM pods (llama.cpp + ROCm/Vulkan) on HX-class nodes. Kernel is now 6.18.x (good for
  recent amdgpu).
- **f5a-01 LLM paths (HX 370)** — two independent options, see `doc/amd-npu.md`:
  - *iGPU/Vulkan* (Radeon 890M, `gfx1150`): works today, `amdgpu` bound, `/dev/dri/renderD128` + KFD compute node
    present. No kernel change needed.
  - *NPU/FastFlowLM* (XDNA2, Strix Point, PCI `1022:17f0`): one kernel-config flip away. Firmware (`amdnpu/17f0_10`)
    ships in the Talos base, XRT userspace runs in the FLM container — but `amdxdna.ko` is NOT built
    (`CONFIG_DRM_ACCEL_AMDXDNA` unset). XDNA2 is the FastFlowLM-supported tier; XDNA1 (Phoenix, PCI `1502`) is not.

### Secrets Management

- SOPS + age for field-level encryption — see `doc/secrets-management.md`
- `.sops.yaml` in repo root defines encryption rules per path
- age private key at `~/.config/sops/age/keys.txt` (never in repo)

## Network

- LAN: `192.168.47.0/24`
- Gateway/DNS: FritzBox at `192.168.47.1`
- DHCP range: `.20` – `.199`
- Cilium LB-IPAM pool: `192.168.47.200` – `192.168.47.209`
- Virtual IPs: `192.168.47.240` – `192.168.47.249` (Talos VIP etc.)
- Control Plane VIP: `192.168.47.240`

## Storage Tiering: Persistent vs. Volatile (per-disk, not per-node)

The rule and its mechanics live in `doc/arc42-homelab.adoc` (Ebene 2 + ADR-H-001 for the per-disk boundary, ADR-H-002
for the full CEL `diskSelector` how-to). Operational essence:

- **Persistent** (replicated state: Forgejo, PostgreSQL, Nexus, SonarQube, registries) → Ceph OSDs. **Volatile**
  (Yocto/OCI build scratch — millions of small files, hardlink churn, toxic to CephFS MDS) → LOCAL scratch NVMe, NEVER
  Ceph, never mirrored. **LLM inference** → weights read-once from Ceph (RWX), then RAM-resident; CephFS-neutral.
- The boundary runs between DISKS, not machines: one node may host OSDs AND local scratch, kept apart via CEL
  `diskSelector`. ALWAYS scope `disk.transport == "nvme"` so the USB-C boot SSD (`/dev/sda`) can't be matched. Inspect
  real values first: `talosctl get disks -o yaml`.
- Build artifacts are persisted by an EXPLICIT CI step (registry push), never by an automatic cluster mechanism — the
  cluster never "knows" the volatile cache.

**OPEN TENSION** (arc42 Risiken): a build pod needing the local scratch NVMe must land on the node that physically has
it — conflicts with the no-nodeSelector/ affinity rule. Unresolved; flag, don't silently violate.

## Node Expansion

Full roster, hardware specs, and rationale (ADR-H-003…007, incl. the voided NUC plan) live in `doc/arc42-homelab.adoc`
(Bausteinsicht Ebene 1 + ADRs). Only operational status + the bits not captured there:

- **Phase 2 — MS-A2 (`ms-a2-01`): DONE.** Worker, not CP (avoids the 2-node etcd quorum trap; CP promotion happens in
  Phase 3, 1→3 in one step). 2× 990 PRO as OSDs; 3rd M.2 reserved for local volatile scratch (UserVolumeConfig, never a
  Ceph OSD). Cluster NIC = Realtek `enp3s0` (ADR-H-005), DHCP fixlease `.39` bound to its MAC. Boot: external USB-C NVMe
  (RTL9210 bridge) — use the *non-USB4* Type-C port; the `usb-storage.quirks` kernel arg is pinned in
  `talos/ms-a2-01-schematics.yaml` (USB4 port renegotiation-loops with the bridge). Result: Ceph failure domain
  transitioned `osd` → `host`; still single CP.
- **Phase 3 — SER6 Pro:** pending. Joins *together with* promoting MS-A2 to CP (1→3 etcd in one step). Canonical
  endpoint moves to VIP `.240` once 3-CP is healthy (VIP cold-start gate already passed).
- **Phase 4 — MS-02 Ultra (tentative):** more OSDs toward `size: 3`. Don't plan around it until the MS-A2 setup proves
  itself.

**Disaster recovery**: `doc/disaster-recovery.md` — MS-A2 promotable to CP under f5a-01 total failure; no new hardware
required.

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
  - *Helm* for complex operator/CNI charts — **Cilium** (`infrastructure/cilium/`) and **Rook-Ceph operator**
    (`infrastructure/rook-ceph/`), future Ingress. Chart version pinned, `values.yaml` committed as SSOT; each dir's
    `README.md` carries the exact `helm upgrade --install` command.
  - *Plain YAML* (`kubectl apply -f <directory>/`) for our own CRs and config — CephCluster/CephFilesystem/StorageClass,
    LB-IPAM pool, L2 policy, workloads.
  - *Kustomize* is allowed where it earns its keep, not mandated.
- **Labels**: Use `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`.
- **No nodeSelector/nodeAffinity** on workloads — pods must be freely schedulable across nodes.
- **StorageClass**: chosen per PVC by access pattern — rule in arc42 ADR-H-009.
- **Secrets**: Real values in YAML, field-level encrypted via SOPS+age. Edit with `sops <file>`, decrypt with
  `sops -d <file>`.
- **Namespaces**: `soho` for business/SOHO workloads (Forgejo, CI). Future: per-customer namespaces for VPN isolation.
- **Talos upgrades**: Start every upgrade with the currency sweep (Upgrade hygiene / arc42 ADR-H-008, below). Always use
  the per-node Image Factory installer from the node's `talos/<node>-schematics.yaml`, never the generic
  `ghcr.io/siderolabs/installer`. This preserves extensions and the schematic ID in META (verify: the node reports its
  schematic ID back).
- **Upgrade hygiene (arc42 ADR-H-008)**: on ANY update trigger, FIRST run a currency sweep over all pinned components,
  then decide per finding — take the patch into this window, or put a minor/major on the roadmap; never ignore. Criteria
  \+ rationale (self-determination, avoid EOL Zugzwang → "Wissen ist Macht") live in ADR-H-008; the Overview version
  table is the checklist anchor.
  ```
  gh release list -R siderolabs/talos -L 10   # Talos (+ its bundled k8s)
  gh release list -R cilium/cilium   -L 10    # Cilium
  gh release list -R rook/rook       -L 10    # Rook (Ceph tracked in ceph-cluster.yaml)
  ```
  Hard gate: a K8s *minor* change means verifying Cilium AND Rook support it first.
- **Node labels for storage tiering**: every node declares its resource-capability axes via three boolean
  `homelab.local/*` labels set in its MachineConfig `nodeLabels`. This makes the per-disk storage tiering visible from
  the K8s API without violating the no-nodeSelector/affinity rule on workloads — the labels are descriptive metadata,
  scheduling mechanism (taints? PV node-affinity?) remains TBD per the OPEN TENSION above.
  - `homelab.local/ceph-osd`: node hosts Ceph OSDs (persistent storage)
  - `homelab.local/scratch-nvme`: node has dedicated local volatile NVMe
  - `homelab.local/llm-capable`: usable LLM inference path (iGPU and/or NPU). On f5a-01 this is the Radeon 890M iGPU
    (Vulkan today) AND the XDNA2 NPU (one kernel flip away via FastFlowLM); see `doc/amd-npu.md`. NOT NPU-gated —
    ms-a2/ser6 are NPU-less. iGPU/memory-bandwidth alone qualifies. Current matrix (ceph-osd, scratch-nvme,
    llm-capable): f5a-01 = (true, false, true); ms-a2-01 = (true, true, false); ser6 = (false, true, false); ms-02 =
    tbd.
- Refer to existing YAMLs in `infrastructure/rook-ceph/` for formatting style.
- **QA gates**: `lefthook.yml` (lefthook-qa-gates contract + AsciiDoc); run them locally before every commit — setup in
  `helpers/README.md`, "QA fleet".

## Design Principles

Quality goals and headline constraints (cattle-not-pets, pod mobility, declarative/GitOps) live in
`doc/arc42-homelab.adoc` (Qualitätsziele + Randbedingungen + Lösungsstrategie). The operational rules that aren't
obvious from there:

- **CNI before storage**: Cilium must be in place before Rook-Ceph. Never replace the CNI while Ceph holds data.
- **Boot drives ≠ Ceph OSDs**: boot media always separate from Ceph storage.
- **Pod mobility is concrete**: use LoadBalancer services (Cilium LB-IPAM), not NodePort; no nodeSelector/affinity on
  workloads (one OPEN TENSION, see above).
- **CP topology is orthogonal to storage tiering**: a node's K8s role (CP vs worker) is driven by etcd quorum (odd
  numbers, currently the 1→3 jump), not by what disks it holds. Tiering rules apply equally to CP and worker nodes.

## Roadmap

Open items, their order and triggers: `doc/req42-homelab.adoc`, chapter 11 (Roadmap & Backlog) — the SSOT. Don't keep a
second list here.

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

- **Cold-start acceptance test**: a node/infra change is *accepted* only after a full power-cycle cold start — NOT the
  kexec that `talosctl upgrade`/`reboot` do by default. Real test: `talosctl reboot --mode powercycle`, one node at a
  time (scale CephFS workloads to 0 first on the mon node). Rationale + constraint → arc42 Randbedingungen.
- `talosctl disks` is deprecated — use `talosctl get disks/discoveredvolumes/systemdisk`.
- `--insecure` flag required for `talosctl` in Maintenance Mode.
- Talos install disk config uses `disk: /dev/sda`, not CEL expressions.
- Mixed patch types in a single `--config-patch` file are unreliable in Talos.
- **Reboot on single-node Ceph**: `talosctl reboot` hangs if CephFS volumes are mounted — the mount waits for the local
  mon which is already shutting down. Workaround: drain CephFS pods first, or use ACPI power-off (short press power
  button). Never change the control plane endpoint to an unproven address on a running cluster.
- **`talosctl upgrade` MUST use `--drain=false` on Ceph mon/CP nodes**: the default `--drain=true` cordon+evict hangs
  forever on a node hosting a Ceph mon — the mon's PodDisruptionBudget blocks eviction (can't drop the only mon without
  losing quorum), so the upgrade never reaches image-pull/reboot. Symptom: machine stage stays `running`, no installer
  activity, `talosctl upgrade` produces no output. With `--drain=false` Talos stops pods via its own shutdown sequence
  instead. Still drain *workload* CephFS pods (e.g. Forgejo `scale --replicas=0`) manually beforehand to avoid the
  mount-hang above; the rook-ceph daemons (mon/mds/osd) come back on their own after the reboot. `kubectl cordon` the
  node first (`uncordon` once Ready): `--drain=false` skips the cordon too, and the scheduler keeps placing replacement
  pods on the node going down — hundreds of `NodeShutdown` rejects pile up in seconds. Cordoning evicts nothing, so the
  mon PDB doesn't block.
- **talosctl version pinning**: Pin `siderolabs/tap/talosctl` in brew (`brew pin`) to match the server version.
  Cross-version gRPC keepalive issues between 1.13.x client and 1.12.x server cause upgrade failures.
- **Talos VIP on single-node**: VIP works but the operator resigns during (even failed) upgrades and doesn't re-campaign
  in 1.12.x. Fixed in 1.13.x. Never point the endpoint to VIP until VIP has survived a full reboot cycle.
