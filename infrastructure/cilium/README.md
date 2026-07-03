# Cilium (CNI) — Helm-managed

Cilium is installed via **Helm** (release `cilium`, namespace `kube-system`).
`values.yaml` in this directory is the **SSOT** for the release configuration —
it was reconstructed from the live cluster on 2026-07-03 (the release predates
this repo capture; see git history).

| | |
|---|---|
| Chart repo | `https://helm.cilium.io/` |
| Chart / app version (running) | `cilium-1.19.5` / `v1.19.5` (revision 3) |
| Values | [`values.yaml`](./values.yaml) |
| Related plain-YAML CRs | [`../cilium-lb/`](../cilium-lb/) (LB-IPAM pool + L2 announcement policy) |

## Reproduce / reconcile the current state

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.19.5 \
  -f infrastructure/cilium/values.yaml
```

## Upgrade

Bump `--version` and re-run the command above (stay on the latest 1.19.x patch).
Then verify:

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system get pods -l k8s-app=cilium
cilium status
```

The DaemonSet rolls agents one node at a time; the datapath persists across an
agent restart, but L2-announcement leases may briefly fail to renew mid-rollout
(transient). Afterwards clean up dead pods left by node reboots
(`ContainerStatusUnknown`/`Error`). Do not jump minors blindly — review the
Cilium upgrade guide before `1.19 → 1.20`.
