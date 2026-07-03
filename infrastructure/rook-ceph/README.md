# Rook-Ceph — Helm operator + plain-YAML CRs

Two layers, deliberately split:

1. **Operator** — installed via **Helm** (release `rook-ceph`, namespace
   `rook-ceph`). Runs on chart defaults; [`operator-values.yaml`](./operator-values.yaml)
   is the SSOT pinning "no overrides".
2. **Cluster resources** — plain-YAML **CRs** applied with `kubectl apply` on top
   of the running operator: [`ceph-cluster.yaml`](./ceph-cluster.yaml),
   [`ceph-filesystem.yaml`](./ceph-filesystem.yaml),
   [`storageclass-cephfs.yaml`](./storageclass-cephfs.yaml),
   [`toolbox.yaml`](./toolbox.yaml), [`namespace.yaml`](./namespace.yaml).

| | |
|---|---|
| Chart repo | `https://charts.rook.io/release` |
| Operator chart version (running) | `rook-ceph-v1.19.7` / app `v1.19.7` (revision 2) |
| Ceph version (running) | `19.2.4` squid (pinned in `ceph-cluster.yaml`) |

## Reproduce / reconcile the current state

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
helm upgrade --install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph --create-namespace \
  --version v1.19.7 \
  -f infrastructure/rook-ceph/operator-values.yaml
# then the CRs (operator must be Ready first):
kubectl apply -f infrastructure/rook-ceph/
```

## Upgrade

Rook upgrades in two independent tracks — **do not bundle them**:

- **Operator** (Rook): bump `--version` and re-run the `helm upgrade` (stay on the
  latest 1.19.x patch). The operator then reconciles mon/mgr/osd/mds itself; a
  patch may leave the daemons untouched (compatible sidecars). `v1.19 → v1.20` is
  a separate minor step — follow the Rook upgrade guide, one minor at a time.
  Note: the operator image pulls from docker.io over the 25 Mbit WAN — a fresh
  version can sit in `ContainerCreating` for minutes; that's the uplink, not a
  fault. Node reboots leave many `ContainerStatusUnknown` daemon pods behind —
  clean them up afterwards.
- **Ceph** (the data plane, currently 19.2.4 squid): bumped by editing
  `spec.cephVersion.image` in `ceph-cluster.yaml`, never a floating tag.

Precondition for any node reboot during these upgrades: `ceph -s` = `HEALTH_OK`,
and scale CephFS-consuming workloads (e.g. `soho/forgejo`) to 0 first to avoid
the mount-hang. Clean up stuck pods afterwards.
