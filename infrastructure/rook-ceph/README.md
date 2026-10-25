# Rook-Ceph — Helm charts + plain-YAML CRs

Two layers, deliberately split:

1. **Helm releases** (namespace `rook-ceph`), values in [`helm/`](./helm/) as SSOT:
   - `rook-ceph` — the Rook operator plus the ceph-csi operator as subchart. Runs on chart defaults;
     [`helm/operator-values.yaml`](./helm/operator-values.yaml) pins "no overrides".
   - `ceph-csi-drivers` — the CSI drivers (RBD, CephFS) the ceph-csi operator runs; since Rook 1.20 no longer Rook's
     job. [`helm/csi-drivers-values.yaml`](./helm/csi-drivers-values.yaml) explains where it deviates from Rook's
     recommended values.
2. **Cluster resources** — plain-YAML **CRs** applied with `kubectl apply` on top of the running operator:
   [`ceph-cluster.yaml`](./ceph-cluster.yaml), [`ceph-filesystem.yaml`](./ceph-filesystem.yaml),
   [`ceph-blockpool.yaml`](./ceph-blockpool.yaml), [`storageclass-cephfs.yaml`](./storageclass-cephfs.yaml),
   [`toolbox.yaml`](./toolbox.yaml), [`namespace.yaml`](./namespace.yaml).

|                                  |                                                                               |
| -------------------------------- | ----------------------------------------------------------------------------- |
| Chart repos                      | `https://charts.rook.io/release`, `https://ceph.github.io/ceph-csi-operator`  |
| Operator chart version (running) | `rook-ceph-v1.20.9` / app `v1.20.9` (revision 5)                              |
| CSI drivers chart (running)      | `ceph-csi-drivers-1.0.4` (revision 2) — matches the operator subchart of Rook |
| Ceph version (running)           | `19.2.7` squid (pinned in `ceph-cluster.yaml`)                                |

## Reproduce / reconcile the current state

```bash
helm repo add rook-release https://charts.rook.io/release
helm repo add ceph-csi-operator https://ceph.github.io/ceph-csi-operator
helm repo update
helm upgrade --install rook-ceph rook-release/rook-ceph \
  --namespace rook-ceph --create-namespace \
  --version v1.20.9 \
  -f infrastructure/rook-ceph/helm/operator-values.yaml
helm upgrade --install ceph-csi-drivers ceph-csi-operator/ceph-csi-drivers \
  --namespace rook-ceph \
  --version 1.0.4 \
  -f infrastructure/rook-ceph/helm/csi-drivers-values.yaml
# then the CRs (operator must be Ready first):
kubectl apply -f infrastructure/rook-ceph/
```

## Upgrade

Rook upgrades in two independent tracks — **do not bundle them**:

- **Operator** (Rook): bump `--version` and re-run the `helm upgrade` (stay on the latest 1.20.x patch). The operator
  then reconciles mon/mgr/osd/mds itself; a patch may leave the daemons untouched (compatible sidecars). A minor is a
  separate step — follow the Rook upgrade guide, one minor at a time. Note: the operator image pulls from docker.io over
  the 25 Mbit WAN — a fresh version can sit in `ContainerCreating` for minutes; that's the uplink, not a fault. Node
  reboots leave many `ContainerStatusUnknown` daemon pods behind — clean them up afterwards.
  - Pre-pull first: image extraction on f5a-01 stalls etcd (arc42 Risiken). A throwaway DaemonSet running `pause`, with
    every new image as an image volume (`volumes[].image`), pulls without executing anything — the CSI sidecars are
    distroless. Take the images from the rendered chart (`rook/ceph`, ceph-csi operator,
    `rook-csi-operator-image-set-configmap`); delete the DaemonSet once READY on all nodes.
  - Then the `ceph-csi-drivers` chart, on the version of the `ceph-csi-operator` subchart that the new `rook-ceph` chart
    pins (`helm show chart rook-release/rook-ceph --version <new>`). Diff its rendered `Driver`/`OperatorConfig` against
    the live CRs first — chart defaults change behaviour (log rotation, replicas, snapshotter). Run it right after the
    `rook-ceph` upgrade: from 1.20 on, `rook-ceph` drops the CSI service accounts, and until the drivers chart recreates
    them the CSI pods are gone (mounted volumes keep working, new mounts wait).
  - Helm 4 applies server-side. Fields an operator wrote earlier (`rook` field manager) make the apply fail with
    `conflict occurred while applying object`; seen once, at the 1.19 → 1.20 handover (image-set ConfigMap,
    `Driver`/`OperatorConfig`). The chart is the owner from 1.20 on — re-run with `--force-conflicts`, which takes over
    only the conflicting fields.
  - The CSI plugins are rolled by the separate ceph-csi operator (`ceph-csi-controller-manager`). It does not react when
    Rook rewrites `rook-csi-operator-image-set-configmap`; if the CSI pods keep the old `cephcsi` image,
    `kubectl -n rook-ceph rollout restart deploy/ceph-csi-controller-manager`.
- **Ceph** (the data plane): bumped by editing `spec.cephVersion.image` in `ceph-cluster.yaml` and `toolbox.yaml`
  together, never a floating tag. Pre-pull `quay.io/ceph/ceph:<new>` the same way as above, then `kubectl apply`.
  - CephX keys (CVE-2025-30156): daemon keys run on AES256K since 19.2.7 (`spec.security.cephx.daemon`). A rotation
    raises `HEALTH_ERR` (`AUTH_INSECURE_SERVICE_*`) until the daemons have restarted — expected; the
    `AUTH_INSECURE_ROTATING_SERVICE_KEY_TYPE` warning clears 2–3 h later. Done when `status.cephx` of the CephCluster
    and the CephFilesystem show the new `keyGeneration`. CSI keys stay `aes`, the three warnings this keeps alive are
    muted in `ceph-cluster.yaml`: kernel mounts need Linux 7.0+ for AES256K (Talos 1.14 ships 6.18) — rotation guide:
    <https://rook.io/docs/rook/latest/Storage-Configuration/Advanced/cephx-keys/>.
- **Toolbox** (`toolbox.yaml`): our own lean variant, not Rook's `deploy/examples/toolbox.yaml`. It reads the mon
  address and the admin key once at start (env from the `rook-ceph-config` and `rook-ceph-mon` Secrets).
  - Before a Rook bump: check that the target release still provides these Secrets and keys — the upstream manifest of
    the target tag shows what Rook ships.
  - After an admin key rotation or a mon move: `kubectl -n rook-ceph rollout restart deploy/rook-ceph-tools`.

Precondition for any node reboot during these upgrades: `ceph -s` = `HEALTH_OK`, and scale CephFS-consuming workloads
(e.g. `soho/forgejo`) to 0 first to avoid the mount-hang. Clean up stuck pods afterwards.
