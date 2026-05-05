# Talos Control Plane VIP — Setup-Anleitung

Stand: 2026-05-05

## Warum VIP von Anfang an?

Ohne VIP zeigt `cluster.controlPlane.endpoint` auf die IP eines einzelnen Nodes.
Das hat zwei Konsequenzen:

1. **Kein HA**: Fällt dieser Node aus, ist die K8s API unerreichbar
2. **Nachträgliches Umstellen ist riskant**: Endpoint-Änderung auf laufendem Cluster
   erfordert, dass die VIP bereits antwortet, bevor der Endpoint umgestellt wird

## Empfohlene Reihenfolge bei Neuinstallation

```
1. IP-Plan festlegen:
   - Node-IPs:        per DHCP-Reservation (Netzwerk-Infrastruktur vergibt)
   - VIP:             feste IP außerhalb DHCP-Range, im selben Subnet
   - LB-IPAM-Pool:    separater Bereich für Service-LoadBalancer

2. talosctl gen config mit VIP-Endpoint:
   talosctl gen config homelab https://<VIP>:6443 \
     --with-secrets secrets.yaml \
     --config-patch @patches/common.yaml

3. Im Machine-Patch die VIP auf dem physischen Interface definieren:
   machine:
     network:
       interfaces:
         - interface: <phy-interface>   # z.B. enp197s0, eth0
           dhcp: true                   # IP kommt vom DHCP-Server
           vip:
             ip: <VIP>                  # z.B. 192.168.47.240

4. Ersten Node bootstrappen — VIP wird sofort auf diesem Node aktiv
5. Weitere CP Nodes: gleiche VIP-Config, Talos handelt Failover automatisch
```

## IP-Bereichsplanung (Beispiel homelab)

| Bereich | Verwendung |
|---|---|
| .1 | Gateway (FritzBox) |
| .20–.199 | DHCP-Range (Nodes, Clients) |
| .200–.209 | Cilium LB-IPAM (Service LoadBalancer) |
| .240–.249 | Virtual IPs (Talos VIP, ggf. weitere) |

## Nachträgliche Umstellung (was wir gemacht haben)

Falls ein Cluster bereits ohne VIP läuft:

```bash
# 1. VIP-Interface aktivieren (kein Reboot nötig)
talosctl patch machineconfig --patch '{
  "machine":{"network":{"interfaces":[{
    "interface":"<phy-interface>",
    "dhcp":true,
    "vip":{"ip":"<VIP>"}
  }]}}
}'

# 2. Verifizieren
ping <VIP>
curl -sk https://<VIP>:6443/livez
talosctl -e <VIP> -n <NODE-IP> version

# 3. Endpoint umstellen
talosctl patch machineconfig --patch '{
  "cluster":{"controlPlane":{"endpoint":"https://<VIP>:6443"}}
}'

# 4. kubeconfig aktualisieren
kubectl config set-cluster <name> --server=https://<VIP>:6443

# 5. talosconfig endpoints aktualisieren
```

## Wichtig

- **DHCP beibehalten**: `dhcp: true` in der Interface-Config. Die Node-IP kommt
  weiterhin vom Netzwerk-DHCP-Server. Talos soll keine IPs selbst vergeben.
- **VIP ≠ Node-IP**: Die VIP ist eine zusätzliche IP, die zwischen CP Nodes wandert.
- **talosconfig.endpoints**: Zeigt auf die VIP (für Management). `nodes` bleibt
  die tatsächliche Node-IP (für node-spezifische Operationen).
- **Repo = Source of Truth**: Jede Live-Änderung muss sofort in die YAML-Dateien
  im Git-Repo zurückfließen.
