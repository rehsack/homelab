# Disaster Recovery — f5a-01 Total Failure

*Stand 29.05.2026 — Aktualisieren wenn sich CP-Topologie ändert.*

## Szenario

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
