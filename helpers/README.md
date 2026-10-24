# helpers/ — ephemeral local helper tools

Docker-Compose definitions for tools we only need *temporarily* and that do NOT
belong in the cluster (admin UIs, one-off utilities). They run on the workstation
(Docker / Colima) and reach cluster services over the LAN via their LoadBalancer
IPs (`192.168.47.200-209`).

```sh
cd helpers
docker compose up <service> -d      # start just what you need
docker compose down                 # stop it again
```

Keeping these out of the cluster avoids permanently exposing admin interfaces
(often with default credentials) and keeps the cluster to actual workloads.

## Services

None currently.
