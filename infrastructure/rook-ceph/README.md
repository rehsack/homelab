# Rook-Ceph — Helm operator + plain-YAML CRs

Two layers, deliberately split:

1. **Operator** — installed via **Helm** (release `rook-ceph`, namespace `rook-ceph`). Runs on chart defaults;
   [`operator-values.yaml`](./operator-values.yaml) is the SSOT pinning "no overrides".
2. **Cluster resources** — plain-YAML **CRs** applied with `kubectl apply` on top of the running operator:
   [`ceph-cluster.yaml`](./ceph-cluster.yaml), [`ceph-filesystem.yaml`](./ceph-filesystem.yaml),
   [`storageclass-cephfs.yaml`](./storageclass-cephfs.yaml), [`toolbox.yaml`](./toolbox.yaml),
   [`namespace.yaml`](./namespace.yaml).

|                                  |                                                    |
| -------------------------------- | -------------------------------------------------- |
| Chart repo                       | `https://charts.rook.io/release`                   |
| Operator chart version (running) | `rook-ceph-v1.19.12` / app `v1.19.12` (revision 3) |
| Ceph version (running)           | `19.2.7` squid (pinned in `ceph-cluster.yaml`)     |

## Reproduce / reconcile the current state

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo update
helm upgrade --install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph --create-namespace \
  --version v1.19.12 \
  -f infrastructure/rook-ceph/operator-values.yaml
# then the CRs (operator must be Ready first):
kubectl apply -f infrastructure/rook-ceph/
```

## Upgrade

Rook upgrades in two independent tracks — **do not bundle them**:

- **Operator** (Rook): bump `--version` and re-run the `helm upgrade` (stay on the latest 1.19.x patch). The operator
  then reconciles mon/mgr/osd/mds itself; a patch may leave the daemons untouched (compatible sidecars). `v1.19 → v1.20`
  is a separate minor step — follow the Rook upgrade guide, one minor at a time. Note: the operator image pulls from
  docker.io over the 25 Mbit WAN — a fresh version can sit in `ContainerCreating` for minutes; that's the uplink, not a
  fault. Node reboots leave many `ContainerStatusUnknown` daemon pods behind — clean them up afterwards.
  - Pre-pull first: image extraction on f5a-01 stalls etcd (arc42 Risiken). A throwaway DaemonSet whose init containers
    run `true` in `rook/ceph:<new>` and the new `cephcsi` image (from the chart's `csi` values) does it; delete it once
    READY on all nodes.
  - The CSI plugins are rolled by the separate ceph-csi operator (`ceph-csi-controller-manager`). It does not react when
    Rook rewrites `rook-csi-operator-image-set-configmap`; if the CSI pods keep the old `cephcsi` image,
    `kubectl -n rook-ceph rollout restart deploy/ceph-csi-controller-manager`.
- **Ceph** (the data plane): bumped by editing `spec.cephVersion.image` in `ceph-cluster.yaml` and `toolbox.yaml`
  together, never a floating tag. Pre-pull `quay.io/ceph/ceph:<new>` the same way as above, then `kubectl apply`.
  - CephX keys (CVE-2025-30156): daemon keys run on AES256K since 19.2.7 (`spec.security.cephx.daemon`). A rotation
    raises `HEALTH_ERR` (`AUTH_INSECURE_SERVICE_*`) until the daemons have restarted — expected; the
    `AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE` warning clears 2–3 h later. Done when `status.cephx` of the CephCluster
    and the CephFilesystem show the new `keyGeneration`. CSI keys stay `aes` and `AUTH_INSECURE_CLIENT_KEY_TYPE` stays:
    kernel mounts need Linux 7.0+ for AES256K (Talos 1.14 ships 6.18) — rotation guide:
    <https://rook.io/docs/rook/latest/Storage-Configuration/Advanced/cephx-keys/>.
- **Toolbox** (`toolbox.yaml`): our own lean variant, not Rook's `deploy/examples/toolbox.yaml`. It reads the mon
  address and the admin key once at start (env from the `rook-ceph-config` and `rook-ceph-mon` Secrets).
  - Before a Rook bump: check that the target release still provides these Secrets and keys — the upstream manifest of
    the target tag shows what Rook ships.
  - After an admin key rotation or a mon move: `kubectl -n rook-ceph rollout restart deploy/rook-ceph-tools`.

Precondition for any node reboot during these upgrades: `ceph -s` = `HEALTH_OK`, and scale CephFS-consuming workloads
(e.g. `soho/forgejo`) to 0 first to avoid the mount-hang. Clean up stuck pods afterwards.
