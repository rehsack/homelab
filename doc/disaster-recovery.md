# Node Bootstrap & Disaster Recovery

*Stand 30.05.2026 — Aktualisieren wenn sich CP-Topologie oder Patches-Struktur ändert.*

Diese Doc deckt zwei verwandte Szenarien ab, die dieselbe Infrastruktur
nutzen (`talos/`-Patches, SOPS-encrypted `secrets.yaml`, Image-Factory-Schematics):

1. **Initialer Bring-up oder Re-Provisionierung eines Knotens** — siehe
   *Per-Node Bootstrap Runbooks* weiter unten. Anwendungsfall: erstes Mal
   Talos installieren, oder einen Node nach Hardware-Tausch wiederherstellen.
2. **Cluster-Level Recovery** nach Verlust des Control Plane — siehe
   *Pfad 1 / 2 / 3* weiter unten. Anwendungsfall: f5a-01 fällt aus.

Beide Szenarien teilen sich die gleichen Voraussetzungen.

## Cluster-Level Recovery — Szenario

f5a-01 (aktuell einziger Control Plane) fällt katastrophal aus, MS-A2 ist
gesund und hat in seiner Worker-Rolle die Hälfte der Ceph-OSDs gehostet.
Ziel: Cluster auf MS-A2 als neuen alleinigen CP wiederherstellen, ohne neue
Hardware bestellen zu müssen.

Dieser Pfad ist eine Folge der bewussten Phase-2-Entscheidung "MS-A2 als
worker" — siehe `CLAUDE.md` → Node Expansion.

## Voraussetzungen vor Disaster

Müssen *vor* dem Incident bestehen, sonst ist Recovery zumindest aufwändiger:

- **Talos-Secrets-Bundle** vom ursprünglichen Cluster-Init erhalten (typisch
  `talos/secrets.yaml`, SOPS-encrypted im Repo).
- **Age-Schlüssel** für SOPS verfügbar (`~/.config/sops/age/keys.txt`),
  *off-cluster* gespeichert.
- **Repo `homelab/`** mit aktuellem Manifest-Stand verfügbar (Git lokal oder
  von einem externen Forgejo-Mirror clonebar).
- **Ceph failure-domain ist `host`** (nicht `osd`) — sonst sind nicht garantiert
  vollständige Daten-Repliken auf MS-A2. Diese Transition gehört zur
  MS-A2-Inbetriebnahme, nicht aufschiebbar.

Wenn auch nur eines davon fehlt → Pfad 3 ist möglicherweise die einzige
Option.

## Per-Node Bootstrap Runbooks

Diese Runbooks dienen zwei Zwecken:
1. **Initialer Bring-up** eines Knotens vom Image-Factory-USB-Stick aus.
2. **Re-Provisionierung** nach Hardware-Tausch oder als Teil eines Recovery-Pfads.

Voraussetzungen wie oben. Schematic-IDs sind in `talos/schematics.yaml` gepinnt;
das identische Schematic auf zwei Nodes ergibt denselben Hash (content-addressed).

### f5a-01 (Control Plane, single-node Cluster-Init)

**Hardware**: Acemagic F5A (AMD HX370). Boot vom externen USB-C-SSD
(Pyrite 2400, 512 GB). NIC `enp197s0` am Switch.

**Patches**: `talos/patch-machine.yaml`, `talos/patch-volumes.yaml`,
`talos/patch-cluster.yaml`, `talos/cilium-patch.yaml`.

**Schematic**: `talos/schematics.yaml` → `f5a-01`.

```bash
# 0) USB-Stick mit der Factory-ISO am F5A booten, Maintenance Mode verifizieren:
talosctl version -n <F5A-IP> -e <F5A-IP> --insecure

# 1) Secrets-Bundle generieren — NUR beim allerersten Cluster-Init!
#    Bei Re-Provisionierung in bestehenden Cluster das existierende
#    SOPS-verschlüsselte talos/secrets.yaml wiederverwenden.
talosctl gen secrets -o talos/secrets.yaml
# danach SOPS-encrypten:
sops -e -i talos/secrets.yaml

# 2) ControlPlane-Config mit allen Patches generieren:
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
sops exec-file talos/secrets.yaml \
    'talosctl gen config homelab https://<F5A-IP>:6443 \
        --with-secrets {} \
        --output-types controlplane \
        --output /tmp/f5a-01.yaml \
        --config-patch @talos/cilium-patch.yaml \
        --config-patch @talos/patch-machine.yaml \
        --config-patch @talos/patch-volumes.yaml \
        --config-patch @talos/patch-cluster.yaml \
        --force'

# 3) Sanity-Check vor Apply (kritische Felder verifizieren):
grep -n -E "hostname:|disk:|wipe:|image:|interface:" /tmp/f5a-01.yaml

# 4) Config ans Maintenance-Mode-API:
talosctl apply-config --insecure -n <F5A-IP> -e <F5A-IP> --file /tmp/f5a-01.yaml
#    → Talos installiert auf Boot-Device, rebootet aus dem Maintenance Mode

# 5) etcd bootstrappen — NUR beim allerersten Node eines Clusters!
#    Bei Re-Provisionierung als CP in bestehenden Cluster NICHT ausführen
#    (siehe Pfad 1/2 für Recovery-spezifische Bootstrap-Pfade).
talosctl bootstrap -n <F5A-IP>

# 6) Kubeconfig holen, talosconfig für zukünftige Aufrufe konfigurieren:
talosctl kubeconfig -n <F5A-IP> ./kubeconfig
talosctl --talosconfig talos/talosconfig.yaml config endpoint <F5A-IP>
talosctl --talosconfig talos/talosconfig.yaml config node <F5A-IP>

# 7) Generierte Config SOPS-encrypten und committen:
sops -e -i /tmp/f5a-01.yaml
mv /tmp/f5a-01.yaml talos/controlplane.yaml
git add talos/controlplane.yaml talos/secrets.yaml talos/talosconfig.yaml
```

Danach (separate Runbooks): Cilium installieren, dann Rook-Ceph,
dann Workloads — siehe `infrastructure/` und `applications/`.

### ms-a2-01 (Worker, joining existing cluster)

**Hardware**: Minisforum MS-A2 (Ryzen 9 9955HX, Zen 5 Fire Range).
Boot vom Image-Factory-USB-Stick (SanDisk 3.2Gen1). Install-Target ist
externes USB-C-NVMe-Gehäuse (ASMedia-Bridge, am *non-USB4* Type-C-Port —
der USB4-Port pendelt mit dieser Bridge in einer Renegotiation-Loop).
NIC `enp3s0` am Switch.

**Pre-flight**: USB-C-NVMe-Gehäuse muss stabil enumeriert sein (vorher
Port-Wechsel testen, falls dmesg-Disconnects am Bildschirm sichtbar).

**Patches**: `talos/ms-a2-01-machine.yaml`, `talos/ms-a2-01-volumes.yaml`,
`talos/cilium-patch.yaml`.

**Schematic**: `talos/schematics.yaml` → `ms-a2-01` (identischer Hash wie
f5a-01 dank content-addressing — gleiche Extension-Set).

```bash
# 0) USB-Stick mit der Factory-ISO am MS-A2 booten, Maintenance Mode verifizieren:
talosctl version -n 192.168.47.39 -e 192.168.47.39 --insecure

# 1) Disks verifizieren — model-Strings müssen mit den CEL-Selectors matchen:
talosctl get disks -n 192.168.47.39 -e 192.168.47.39 --insecure -o yaml
#    Erwartet (siehe CLAUDE.md → Storage Tiering):
#      nvme0n1, nvme1n1 = "Samsung SSD 990 PRO 4TB" (Ceph-OSDs)
#      sda = "SanDisk 3.2Gen1" (Boot-Stick)
#      sdb = "M.2 NVME SSD" (USB-C-Gehäuse, Install-Target)

# 2) Worker-Config mit Patches generieren (sops exec-file dechiffriert
#    secrets.yaml in einen tmp file, kein plaintext auf Disk):
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
sops exec-file talos/secrets.yaml \
    'talosctl gen config homelab https://192.168.47.40:6443 \
        --with-secrets {} \
        --output-types worker \
        --output /tmp/ms-a2-01.yaml \
        --config-patch @talos/cilium-patch.yaml \
        --config-patch @talos/ms-a2-01-machine.yaml \
        --config-patch @talos/ms-a2-01-volumes.yaml \
        --force'

# 3) Sanity-Check vor Apply (kritische Felder):
grep -n -E "hostname:|disk:|wipe:|image:|interface:|homelab.local|node.kubernetes.io" /tmp/ms-a2-01.yaml
#    Erwartet:
#      hostname: ms-a2-01
#      install.disk: /dev/sdb
#      install.wipe: true
#      install.image: factory.../v1.13.2 (passend zur schematics.yaml)
#      interface: enp3s0
#      nodeLabels: homelab.local/{ceph-osd,llm-capable,scratch-nvme}
#                  + node.kubernetes.io/role: worker

# 4) Apply ans Maintenance-Mode-API:
talosctl apply-config --insecure -n 192.168.47.39 -e 192.168.47.39 \
    --file /tmp/ms-a2-01.yaml
#    → Talos installiert auf /dev/sdb (wipe vorab), rebootet
#    → Maintenance Mode endet, SanDisk-Boot-Stick darf raus

# 5) Post-install verify (KEIN --insecure mehr, talosconfig benutzen):
talosctl --talosconfig talos/talosconfig.yaml version -n 192.168.47.39
kubectl --kubeconfig ./kubeconfig get nodes
#    Erwartet: ms-a2-01 Ready, Labels:
#      homelab.local/ceph-osd=true,
#      homelab.local/llm-capable=false,
#      homelab.local/scratch-nvme=false,
#      node.kubernetes.io/role=worker
#    Hinweis: IP kann sich nach Reboot ändern (neuer DHCP-Lease möglich)

# 6) Generierte Config SOPS-encrypten und committen:
sops -e -i /tmp/ms-a2-01.yaml
mv /tmp/ms-a2-01.yaml talos/ms-a2-01.yaml
git add talos/ms-a2-01.yaml
```

Danach (separate, nicht Teil dieses Runbooks):
- Ceph: `failure domain` von `osd` auf `host` umstellen (CephCluster-CR
  patchen). Die zwei NVMe auf MS-A2 werden über `deviceFilter: "^nvme"`
  automatisch als OSDs adoptiert.
- (Phase 3) MS-A2 zur CP-Rolle promoten — koordiniert mit SER6-Beitritt.

### SER6 Pro (Worker → CP-Promotion, Phase 3)

Wenn SER6 ankommt: zuerst als Worker einbinden (analog zu ms-a2-01),
dann *koordiniert* mit MS-A2 zur Control Plane befördern, sodass etcd
in einem Schritt von 1 → 3 Membern wächst (kein 2-Member-Quorum-Loch).

Detail-Runbook folgt sobald die Hardware da ist und der MachineConfig-Patch
geschrieben ist.

## Pfad 1 — etcd-Snapshot Restore (empfohlen, sobald Snapshots laufen)

**Status heute (29.05.2026)**: Snapshot-Automatisierung steht noch aus, daher
ist Pfad 2 vorerst der reale Pfad. Diese Sektion beschreibt das Soll-Verfahren,
sobald die TODO-Liste am Ende abgearbeitet ist.

**Zusätzliche Voraussetzung**: aktueller etcd-Snapshot (`< 24 h alt`), off-cluster
gespeichert.

Grober Ablauf (Details: Talos Disaster-Recovery-Doku):

```bash
# 1) MS-A2 in Maintenance Mode bringen — Boot vom USB-Stick (sollte noch da sein).

# 2) Control-plane MachineConfig für MS-A2 generieren mit den ORIGINAL-Secrets
#    aus dem Repo (gleiche Cluster-CA, etcd-Keys), damit der Snapshot passt:
talosctl gen config homelab https://192.168.47.39:6443 \
    --with-secrets talos/secrets.yaml \
    --output-types controlplane

# 3) Bootstrap mit Snapshot-Restore:
talosctl apply-config --insecure -n 192.168.47.39 --file controlplane.yaml
talosctl bootstrap -n 192.168.47.39 \
    --recover-from <pfad-zum-snapshot.db>

# 4) Cluster läuft, MS-A2 ist alleiniger CP. K8s-State (Workloads, RBAC,
#    Secrets, PVCs) kommen aus dem Snapshot zurück.
```

Danach Ceph-Reparatur (siehe unten "Ceph-Reanimation").

Erwarteter Aufwand: 1–2 h.

## Pfad 2 — Bootstrap + Git-Replay + Ceph-Recovery (ohne Snapshot)

Aktuell der realistische Pfad. Verliert K8s-State (Workload-Definitionen,
RBAC, Secrets), aber Daten in CephFS bleiben erhalten — wenn die OSDs
adoptiert werden können.

```bash
# 1) MS-A2 in Maintenance Mode bringen.

# 2) Frische Cluster-Bootstrap: NEUE Secrets, MS-A2 als alleiniger CP.
talosctl gen config homelab https://192.168.47.39:6443 \
    --output-types controlplane

# 3) MachineConfig anpassen (install-disk, hostname, schematic-installer),
#    Vorlage: talos/ms-a2-01.yaml (worker-Variante), Typ auf controlplane
#    umstellen.

# 4) Apply + Bootstrap:
talosctl apply-config --insecure -n 192.168.47.39 \
    --file ms-a2-01-controlplane.yaml
# warten bis Install + Reboot durch
talosctl bootstrap -n 192.168.47.39

# 5) Kubeconfig holen:
talosctl kubeconfig -n 192.168.47.39 ./kubeconfig-recovery
```

Manifests neu apply'en (Reihenfolge wichtig: CNI vor Storage vor Apps):

```bash
kubectl apply -f infrastructure/cilium/
# warten bis cilium-DaemonSet ready
kubectl apply -f infrastructure/rook-ceph/operator/
# warten bis Operator ready
kubectl apply -f infrastructure/rook-ceph/cluster/  # mit OSD-Adoption!
# warten bis CephCluster wieder healthy
kubectl apply -f applications/
```

Erwarteter Aufwand: halber Tag bis Tag, je nach Komplexität.

### Ceph-Reanimation (gemeinsam für Pfad 1 und 2)

Die OSDs auf MS-A2 sind LVM-Volumes mit Rook-Metadaten. Sie müssen
*adoptiert* werden, **nicht** wegformatiert:

- Rook-CephCluster CR: `cleanupPolicy.confirmation` darf **nicht** gesetzt sein
  (das würde die Adoption unterbinden und Daten wegwerfen).
- Mon-DB neu aufbauen — die Original-Mons waren auf f5a-01 und sind weg. Rook
  hat dafür eine "Mon-Quorum-Recovery"-Prozedur dokumentiert (Suchwort:
  *"restore quorum from single mon"* oder *"adopt OSDs after Mon loss"*).
- Vorher mit `rook-ceph-tools`-Pod den OSD-Tree inspizieren, um sicher zu sein
  dass die OSDs intakt sind.

Wegen der Komplexität dieses Schrittes ist die Trockenübung (siehe TODO unten)
besonders wichtig.

## Pfad 3 — Worst Case: Cluster komplett neu

Wenn MS-A2's OSDs *auch* unbrauchbar sind (z.B. Stromschaden, der beide Nodes
erwischt hat), oder die Voraussetzungen aus den ersten Abschnitten nicht
erfüllbar sind:

- Cluster neu bootstrappen.
- Workload-Daten in CephFS sind verloren, sofern kein externes Backup existiert.
- Manifests aus Git restorabel.

Dies ist das Szenario, das wir durch Pfad 1 und 2 *vermeiden* — aber es ist
realistisch genug, dass externe Backups der wichtigsten Workload-Daten
(z.B. Forgejo-Repos via `git push` auf einen externen Mirror) sinnvoll sind.

## Warum MS-A2's OSDs eine vollständige Datenkopie halten

Konfiguration nach MS-A2-Beitritt: `size: 2`, `failure domain: host`, 2 Hosts.
Jede Placement Group hat genau eine Replika pro Host. f5a-01-Verlust →
MS-A2 hat eine vollständige Kopie aller Daten.

Gegenbeispiel: bei `failure domain: osd` (heute auf Single-Host) könnten
beide Repliken einer PG auf zwei OSDs *desselben* Hosts liegen — host-Verlust
würde dann beide treffen. Genau deshalb ist die Transition zu `host`-Domain
*nicht* aufschiebbar, sobald MS-A2 läuft.

## TODO (nach heute Abend, vor dem ersten realen Incident)

- [ ] `talosctl etcd snapshot`-Automatisierung einrichten — Talos-Service oder
      externer Cron-Job
- [ ] Snapshots off-cluster persistieren — SOPS-encrypted im Repo, oder auf
      FritzBox-USB, oder NAS
- [ ] Rook-Ceph Mon-Topologie für 2-Host-Setup überdenken — aktuell vermutlich
      alle 3 Mons auf f5a-01. Mindestens einer sollte auf MS-A2 wandern, sobald
      Worker stabil läuft (Mon-Pod-Verteilung über `mon.allowMultiplePerNode:
      false` und Anti-Affinity)
- [ ] Pfad 2 einmal als Trockenübung durchspielen — gezielt f5a-01 herunterfahren
      und die OSD-Adoption auf MS-A2 üben. Ein Recovery-Pfad ist nur verlässlich,
      wenn er mindestens einmal manuell ausgeführt wurde.
