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

### `cubesql-webadmin`
Browser admin for the in-cluster cubeSQL server (`applications/cubesql/`). Use it
to register the free license, manage databases/backups, and change `admin/admin`.

```sh
docker compose up cubesql-webadmin -d
# open http://localhost:8080  → host/port/user are prefilled; type the password
docker compose down
```

The compose deliberately carries NO password — it lives in the password manager
(Strongbox); enter it at the web admin's login form. The connection uses AES256
(`CUBESQL_ENCRYPTION`).

Note: the helper container must be able to reach `192.168.47.201` on the LAN. With
Colima that normally works for outbound LAN traffic; if not, check Colima's
network mode.
