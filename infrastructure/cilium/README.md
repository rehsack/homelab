# Cilium (CNI) — Helm-managed

Cilium is installed via **Helm** (release `cilium`, namespace `kube-system`). `values.yaml` in this directory is the
**SSOT** for the release configuration — it was reconstructed from the live cluster on 2026-07-03 (the release predates
this repo capture; see git history).

|                               |                                                                          |
| ----------------------------- | ------------------------------------------------------------------------ |
| Chart repo                    | `https://helm.cilium.io/`                                                |
| Chart / app version (running) | `cilium-1.20.4` / `v1.20.4` (revision 5)                                 |
| Values                        | [`values.yaml`](./values.yaml)                                           |
| Related plain-YAML CRs        | [`../cilium-lb/`](../cilium-lb/) (LB-IPAM pool + L2 announcement policy) |

## Reproduce / reconcile the current state

```bash
helm repo add cilium https://helm.cilium.io/
helm repo update
helm upgrade --install cilium cilium/cilium \
  --namespace kube-system \
  --version 1.20.4 \
  -f infrastructure/cilium/values.yaml
```

## Upgrade

Bump `--version` and re-run the command above (stay on the latest 1.20.x patch). Then verify:

```bash
kubectl -n kube-system rollout status ds/cilium
kubectl -n kube-system get pods -l k8s-app=cilium
cilium status
```

The DaemonSet rolls agents one node at a time (`updateStrategy` in `values.yaml`); the datapath persists across an agent
restart, but L2-announcement leases may briefly fail to renew mid-rollout (transient). Afterwards clean up dead pods
left by node reboots (`ContainerStatusUnknown`/`Error`, `Completed`).

### Minor upgrade

Cilium tests only consecutive minors, starting from the latest patch of the running one: patch first, then the minor.

1. Read the version's upgrade notes (`Documentation/operations/upgrade-current.inc` at the target tag) against the
   cluster, and diff the rendered `cilium-config` of both versions with our values (`helm template … | yq`) — that shows
   every default change that actually takes effect.

2. Pre-pull the images with the pre-flight check. Image extraction on f5a-01 stalls etcd (arc42 Risiken), so it should
   not coincide with an agent restart. Render it with our values, but apply **only** the `pre-flight` objects — the full
   render also carries Hubble and the `cilium-ca` Secret:

   ```bash
   helm template cilium cilium/cilium --version <new> --namespace kube-system -f infrastructure/cilium/values.yaml \
     --set preflight.enabled=true --set agent=false --set operator.enabled=false \
     | yq 'select(.metadata.name | test("pre-flight"))' > preflight.yaml
   kubectl create -f preflight.yaml    # wait for READY on all nodes
   kubectl delete -f preflight.yaml
   ```

3. `helm upgrade` as above.
