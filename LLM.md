# CLAUDE.md — homelab Kubernetes Cluster

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

## Node Expansion (imminent)

- **Phase 2**: Intel NUC6i7KYK (32GB RAM, 2× Samsung 990 PRO 4TB NVMe) — lightweight workloads, 2 Ceph OSDs. Boot from external USB-SSD (NOT a Ceph OSD).
- **Phase 3**: Beelink SER6 Pro (32GB RAM) — third control plane node, no dedicated Ceph OSDs.
- **Phase 4**: Minisforum MS-02 Ultra (Intel 285HX, up to 256GB ECC RAM, 4× M.2) — additional Ceph OSDs, Ceph reaches `size: 3`.

## Repo Structure

```
homelab/
├── talos/              # Talos machine configs, patches, secrets — NOT applied via kubectl
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
- Refer to existing YAMLs in `infrastructure/rook-ceph/` for formatting style.

## Design Principles

- **Cattle, not pets**: Every node re-provisionable from config. No manual state on nodes.
- **Pod mobility**: Workloads must survive node rescheduling. Use LoadBalancer services (Cilium LB-IPAM), not NodePort.
- **CNI before storage**: Cilium must be in place before Rook-Ceph. Never replace CNI while Ceph holds data.
- **Boot drives ≠ Ceph OSDs**: Boot media is always separate from Ceph storage.
- **Declarative**: All config in git. Secrets encrypted (SOPS+age). Target: `git clone` + age key + `sops -d | kubectl apply` = running cluster.

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
