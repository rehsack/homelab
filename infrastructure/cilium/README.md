# Cilium (CNI) — Helm-managed

Cilium is installed via **Helm** (release `cilium`, namespace `kube-system`).
`values.yaml` in this directory is the **SSOT** for the release configuration —
it was reconstructed from the live cluster on 2026-07-03 (the release predates
this repo capture; see git history).

| | |
|---|---|
| Chart repo | `https://helm.cilium.io/` |
| Chart / app version (running) | `cilium-1.19.1` / `v1.19.1` (revision 2) |
| Values | [`values.yaml`](./values.yaml) |
| Related plain-YAML CRs | [`../cilium-lb/`](../cilium-lb/) (LB-IPAM pool + L2 announcement policy) |

## Reproduce / reconcile the current state

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.19.1 \
  -f infrastructure/cilium/values.yaml
```

## Upgrade

Bump `--version` and re-run the command above — **next target `1.19.5`** (latest
1.19.x patch, same minor → low risk). Then verify:

```bash
cilium status --wait
kubectl -n kube-system rollout status ds/cilium
```

Afterwards, clean up any stuck/terminating pods. Do not jump minors blindly —
review the Cilium upgrade guide before `1.19 → 1.20`.
