# LLM.md — homelab Kubernetes Cluster

## Cluster Overview

Single-node Talos Linux cluster on Acemagic F5A (AMD HX370, 96GB RAM), expanding to 3 nodes shortly.

| Component       | Details                                                    |
|-----------------|------------------------------------------------------------|
| OS              | Talos Linux 1.13.2                                         |
| Kubernetes      | 1.35.0                                                     |
| CNI             | Cilium 1.19.1 (kube-proxy replacement, WireGuard encryption, Hubble UI) |
| Storage         | Rook-Ceph, 2× Samsung 990 PRO 4TB NVMe as OSDs            |
| Filesystem      | CephFS, StorageClass `rook-cephfs`                         |
| Ceph replication| `size: 2`, failure domain `osd` (transitions to `host` with Node 2) |
| Node            | `f5a-01` at `192.168.47.40`                                |
| Control Plane   | VIP `192.168.47.240` (active), endpoint `https://192.168.47.40:6443` (transitions to VIP with Node 2) |
| LoadBalancer    | Cilium LB-IPAM + L2 Announcements, pool `192.168.47.200-209` |
| Ingress         | None yet                                                   |
| Secrets         | SOPS + age (field-level encryption in Git); cluster-side decryption TBD |

- **Talos 1.13 reproducible images**: Same build → identical disk image (raw images;
  VHD/VMDK excluded). Reinforces the schematic-pinned upgrade workflow.
- **Talos 1.13 enables CDI by default**; extension services can drop CDI specs under
  /run/cdi. This is the path for exposing the AMD iGPU/NPU to LLM pods (llama.cpp +
  ROCm/Vulkan) on HX-class nodes. Kernel is now 6.18.x (good for recent amdgpu).

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

> **History note**: The original Phase 2 plan (Intel NUC6i7KYK as second OSD host)
> is VOID. The NUC6i7KYK failed (dead display output, unclearable CMOS error,
> no DHCP lease — retired to the "repair someday" pile, not scrap). Its two Samsung
> 990 PRO 4TB drives were removed for reuse in the replacement node. This forced a
> cleaner architecture (see "Storage Tiering" below): the per-node "storage vs.
> compute" split was replaced by a per-disk "persistent vs. volatile" model.

- **Phase 2 (active): Minisforum MS-A2** (Ryzen 9 9955HX, Zen 5 Fire Range,
  16C/32T, Radeon 610M iGPU, *no XDNA NPU*). Dual SO-DIMM DDR5-5600, 3× M.2
  PCIe 4.0 (manual warns: full bestückung → 2 of 3 throttle to Gen3 thermally;
  irrelevant for our workloads — Ceph is net-bound, Yocto is IOPS-bound, see
  `doc/ssd-selection.md`). NICs verified on arrival: 2× Intel i226-V (`igc`,
  enp3s0/enp4s0) + 2× Marvell AQC113C (`atlantic`, enp5s0f0np0/f1np1) — all
  detected by stock 6.18 kernel, no driver extension needed.
  - **K8s role: worker** (NOT CP). Avoids the 2-node etcd quorum trap (where
    loss of either CP would freeze the cluster — 2 members need 2 votes to make
    progress). Promotion to CP happens in Phase 3 coordinated with SER6, going
    directly 1→3 etcd members in one step.
  - **Storage role**: 2× Samsung 990 PRO 4TB (ex-NUC) as Ceph OSDs;
    1× differently-modelled NVMe (TBD, next week) in the 3rd M.2 as LOCAL
    volatile scratch (UserVolumeConfig, never a Ceph OSD). Boot from external
    USB-C NVMe enclosure (ASMedia bridge): use the *non-USB4* Type-C port —
    the USB4 port enters a renegotiation loop with this bridge chip.
  - **Compute role**: persistent workloads (Forgejo, PostgreSQL, SonarQube,
    Nexus, registries) + volatile builds (Yocto, OCI). *No LLM inference* —
    Fire Range has no NPU and the Radeon 610M iGPU is too weak.
  - With Node 2 online: Ceph failure domain transitions `osd` → `host`.
    Control plane endpoint stays at `.40` (no VIP transition, no 2-CP trap).
- **Phase 3: Beelink SER6 Pro** (Ryzen 7 7735HS, Zen3+, 32GB) — pure VOLATILE
  COMPUTE node (Yocto/OCI builds on local NVMe). Weak 680M iGPU → not an LLM
  node. Joins **together with promoting MS-A2 to CP** — 1→3 etcd in one
  coordinated step, skipping the 2-node quorum trap. Control plane endpoint
  transitions to VIP `192.168.47.240` once 3-CP topology is healthy.
- **Phase 4 (tentative): Minisforum MS-02 Ultra** or successor — additional
  Ceph OSDs toward `size: 3`. With 4-node topology the typical pattern is
  3 CPs + 1 worker (etcd stays at 3, MS-02 joins as pure worker). POSSIBLE
  second dual-purpose node IF the MS-A2 CEL per-disk separation proves stable
  in practice. Bonus: boots from internal M.2, no USB-boot needed. Do NOT
  plan around this until the MS-A2 setup proves itself.

**Disaster recovery**: see `doc/disaster-recovery.md` — MS-A2 can be promoted
to CP under f5a-01 total failure; multiple paths documented, none requiring
new hardware.

## Repo Structure

```
homelab/
├── talos/              # Talos machine configs, patches, secrets, schematics — NOT applied via kubectl
│   └── schematics.yaml # Image Factory schematic IDs and extensions per node
├── infrastructure/     # Cluster infrastructure (storage, networking)
│   ├── rook-ceph/      # Rook-Ceph operator, CephCluster, StorageClass
│   └── cilium-lb/          # LB-IPAM pool + L2 announcement policy
└── applications/       # Workloads
    └── forgejo/        # Git forge (first workload)
```

## Conventions

- **Plain YAML manifests**, applied with `kubectl apply -f <directory>/`. No Helm, no Kustomize.
- **Labels**: Use `app.kubernetes.io/name`, `app.kubernetes.io/component`, `app.kubernetes.io/part-of`.
- **No nodeSelector/nodeAffinity** on workloads — pods must be freely schedulable across nodes.
- **StorageClass**: Always `rook-cephfs` for PVCs.
- **Secrets**: Real values in YAML, field-level encrypted via SOPS+age. Edit with `sops <file>`, decrypt with `sops -d <file>`.
- **Namespaces**: `soho` for business/SOHO workloads (Forgejo, CI). Future: per-customer namespaces for VPN isolation.
- **Talos upgrades**: Always use the per-node Image Factory installer from `talos/schematics.yaml`, never the generic `ghcr.io/siderolabs/installer`. This preserves extensions and the schematic ID in META.
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
- **talosctl version pinning**: Pin `siderolabs/tap/talosctl` in brew (`brew pin`) to match the server version. Cross-version gRPC keepalive issues between 1.13.x client and 1.12.x server cause upgrade failures.
- **Talos VIP on single-node**: VIP works but the operator resigns during (even failed) upgrades and doesn't re-campaign in 1.12.x. Fixed in 1.13.x. Never point the endpoint to VIP until VIP has survived a full reboot cycle.
