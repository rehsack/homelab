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

Voraussetzungen wie oben. Schematic-IDs sind per-node in
`talos/<node>-schematics.yaml` gepinnt; gleiche Extension-Sets ergeben denselben
Hash (content-addressed), divergieren aber, sobald ein Node eigene Kernel-Args
braucht (z.B. ms-a2-01 mit dem `usb-storage`-UAS-Quirk für die RTL9210-Bridge).

### f5a-01 (Control Plane, single-node Cluster-Init)

**Hardware**: Acemagic F5A (AMD HX370). Boot vom externen USB-C-SSD
(Pyrite 2400, 512 GB). NIC `enp197s0` am Switch.

**Patches**: `talos/f5a-01-machine.yaml`, `talos/f5a-01-volumes.yaml`,
`talos/homelab-endpoint.yaml`, `talos/homelab-k8s-images.yaml` (CP-only),
`talos/homelab-cilium.yaml`.

**Schematic**: `talos/f5a-01-schematics.yaml`.

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
        --config-patch @talos/homelab-cilium.yaml \
        --config-patch @talos/f5a-01-machine.yaml \
        --config-patch @talos/f5a-01-volumes.yaml \
        --config-patch @talos/homelab-endpoint.yaml \
        --config-patch @talos/homelab-k8s-images.yaml \
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

# 7) Die generierte Config wird NICHT committet (regenerierbar aus secrets +
#    Patches). Optional als lokale Referenz nach talos/.gen/ (gitignored):
mv /tmp/f5a-01.yaml talos/.gen/controlplane.yaml
#    Committet werden nur die Quellen:
git add talos/secrets.yaml talos/talosconfig.yaml
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

**Patches**: `talos/ms-a2-01-machine.yaml`, `talos/ms-a2-01-hostname.yaml`,
`talos/ms-a2-01-volumes.yaml`, `talos/homelab-cilium.yaml`.

**Schematic**: `talos/ms-a2-01-schematics.yaml` (eigener Hash —
`usb-storage.quirks=0bda:9210:u` für die RTL9210-Boot-Bridge, weicht damit
bewusst von f5a-01 ab; nicht mehr derselbe content-addressed Hash).

```bash
# 0) USB-Stick mit der Factory-ISO am MS-A2 booten, Maintenance Mode verifizieren:
talosctl version -n 192.168.47.39 -e 192.168.47.39 --insecure

# 1) Disks verifizieren — model-Strings müssen mit den CEL-Selectors matchen:
talosctl get disks -n 192.168.47.39 -e 192.168.47.39 --insecure -o yaml
#    Erwartet (siehe CLAUDE.md → Storage Tiering):
#      nvme0n1, nvme1n1 = "Samsung SSD 990 PRO 4TB" (Ceph-OSDs, intern)
#      nvme2n1          = "Samsung SSD 970 EVO Plus 2TB" (Yocto-Scratch,
#                         3. interner M.2-Slot; UserVolumeConfig — Kühlkörper
#                         GEKLEBT, bleibt intern, nie als USB-Boot umfunktionieren)
#      sda              = "CT1000P5..." (Crucial P5 1TB im UGREEN-USB-C-Gehäuse
#                         mit RTL9210B-Bridge, Install-Target)
#      sdb              = "SanDisk 3.2Gen1" (Boot-Stick, USB-A)
#
#    Caveat 1: die Reihenfolge sda/sdb hängt von der USB-Enumeration ab.
#    Immer per `talosctl get disks` verifizieren, NICHT raten. Bei
#    abweichender Reihenfolge `install.disk` im machine-Patch anpassen,
#    BEVOR apply-config.
#
#    Caveat 2: USB-NVMe-Bridges sind nicht alle gleich. ASMedia-Bridges
#    (ASM2362, ASM2364) hatten auf AMD-Ryzen-USB4-Hosts (Fire Range /
#    Rembrandt / Phoenix) reproduzierbare USB4-Negotiation-Loops mit
#    Disconnect-Storm. Empirisch validiert: UGREEN-Gehäuse mit RTL9210B
#    (Modell 15512) am rückseitigen USB 3.2 Gen 2x1 Port (NICHT am USB4
#    vorne). Siehe `doc/ssd-selection.md` für Marktstand.
#
# 1a) Recovery-Hinweis: wenn Talos beim Boot vom Stick in einer Schleife
#     hängt mit Console-Output "task haltIfInstalled: Talos is already
#     installed to disk but booted from another media", dann hat ein
#     vorheriger (möglicherweise gescheiterter) Install META-Reste auf
#     einer der Disks hinterlassen. Zwei Auswege:
#     - Wenn die "installed" Disk physisch ziehbar ist: abziehen, neu
#       booten — Talos sieht keine bestehende Installation mehr und
#       geht sauber in Maintenance Mode.
#     - Wenn nicht (z.B. interne NVMe): am Bootloader-Menü vom Stick
#       `e` drücken zum Edit der Kernel-Cmdline. Der Parameter
#       `talos.halt_if_installed=1` steht bereits in der `linux`-Zeile
#       (Default vom Schematic-Build). Den Wert auf `0` ändern — also
#       `=1` zu `=0` umschreiben, NICHT zusätzlich anhängen. Mit Ctrl-X
#       (GRUB) oder F10 booten. Talos ignoriert dann den existing-install
#       für diesen einen Boot und geht sauber in Maintenance Mode.
#       Bei den nächsten Boots ist die Cmdline wieder Default — also
#       entweder erneut editieren oder (besser) die "installed" Disk
#       per Maintenance-API wipen:
#         talosctl wipe disk <kurzname> -i -n IP -e IP
#       (Kurzname = z.B. `nvme2n1`, nicht `/dev/nvme2n1`. Default-Wipe-
#       Methode FAST reicht, schreibt nur Schlüsselsektoren leer.)
#       Damit triggert kein zukünftiger Boot mehr `halt_if_installed`.

# 2) Worker-Config mit Patches generieren (sops exec-file dechiffriert
#    secrets.yaml in einen tmp file, kein plaintext auf Disk):
SOPS_AGE_KEY_FILE=~/.config/sops/age/keys.txt \
sops exec-file talos/secrets.yaml \
    'talosctl gen config homelab https://192.168.47.40:6443 \
        --with-secrets {} \
        --output-types worker \
        --output /tmp/ms-a2-01.yaml \
        --config-patch @talos/homelab-cilium.yaml \
        --config-patch @talos/ms-a2-01-machine.yaml \
        --config-patch @talos/ms-a2-01-hostname.yaml \
        --config-patch @talos/ms-a2-01-volumes.yaml \
        --force'

# 2a) Workaround für siderolabs/talos#12573:
#     Die auto-generierte HostnameConfig hat `auto: stable` als Default. Unser
#     ms-a2-01-hostname.yaml Patch FÜGT `hostname: ms-a2-01` strategic-merged
#     HINZU, kann aber `auto: stable` nicht ENTFERNEN. Talos lehnt beide
#     gemeinsam ab ("'auto' and 'hostname' cannot be set at the same time").
#     JSON-6902-Patches sind für multi-document configs nicht unterstützt
#     ("JSON6902 patches are not supported for multi-document machine
#     configuration"). Lösung: nach gen-config die `auto:`-Zeile direkt aus
#     dem HostnameConfig-Block streichen.
sed -i.bak '/^kind: HostnameConfig$/,/^---$/ { /^auto:/d; }' /tmp/ms-a2-01.yaml
# Verify (sollte nur `hostname:` zeigen, kein `auto:`):
sed -n '/^kind: HostnameConfig$/,/^---$/p' /tmp/ms-a2-01.yaml

# Note: derselbe Workaround gilt analog für f5a-01, sobald wir dessen Patches
# beim 1.13.3-Upgrade nachziehen. f5a-01 setzt hostname aktuell noch über
# machine.network.hostname (v1alpha1-Mechanismus), das wird beim Re-Patch
# voraussichtlich denselben Konflikt mit der HostnameConfig erzeugen.

# 3) Sanity-Check vor Apply (kritische Felder):
grep -n -E "hostname:|disk:|wipe:|image:|interface:|homelab.local|node.kubernetes.io" /tmp/ms-a2-01.yaml
#    Erwartet:
#      hostname: ms-a2-01
#      install.disk: /dev/sda
#      install.wipe: true
#      install.image: factory.../v1.13.x (passend zum ms-a2-01-schematics.yaml)
#      interface: enp3s0
#      nodeLabels: homelab.local/{ceph-osd,llm-capable,scratch-nvme}
#                  + node.kubernetes.io/role: worker

# 4) Apply ans Maintenance-Mode-API:
talosctl apply-config --insecure -n 192.168.47.39 -e 192.168.47.39 \
    --file /tmp/ms-a2-01.yaml
#    → Talos installiert auf /dev/sda (wipe vorab), rebootet
#    → Maintenance Mode endet, SanDisk-Boot-Stick darf raus

# 5) Post-install verify (KEIN --insecure mehr, talosconfig benutzen):
talosctl --talosconfig talos/talosconfig.yaml version -n 192.168.47.39
kubectl --kubeconfig ./kubeconfig get nodes
#    Erwartet: ms-a2-01 Ready, Labels:
#      homelab.local/ceph-osd=true,
#      homelab.local/llm-capable=false,
#      homelab.local/scratch-nvme=true,
#      node.kubernetes.io/role=worker
#    Hinweis: IP kann sich nach Reboot ändern (neuer DHCP-Lease möglich)

# 6) Die generierte Config wird NICHT committet (regenerierbar aus secrets +
#    Patches). Optional als lokale Referenz nach talos/.gen/ (gitignored):
mv /tmp/ms-a2-01.yaml talos/.gen/ms-a2-01.yaml
#    (keine neuen Quellen zu committen — secrets/talosconfig bestehen bereits)
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
#    Vorlage: die ms-a2-01-Patches (worker-Variante), Typ auf controlplane
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

## Operative Recovery — Container-Image-Cache-Korruption nach Disk-Failure

**Szenario**: Eine vorübergehende Boot-Disk-Failure (z.B. USB-NVMe-Bridge zickt,
sda4 in EIO, XFS-Recovery klärt Metadata aber Daten-Blöcke der OSI-Layer-Files
sind partiell ungeschrieben) hinterlässt korrupte Container-Image-Blobs im
containerd-Content-Store. Pods scheitern mit `exec format error` (ENOEXEC) oder
sporadischem EIO beim Container-Start, *obwohl* dmesg keine neuen Disk-Errors
mehr zeigt und der `image ls` das erwartete Image listet. Der `sha256`-Hash des
Blobs stimmt im containerd-Index, weil der Hash beim Write-Time gegen Memory
geprüft wurde — die später beobachtete Diskrepanz ist eine Block-Layer-Realität
ohne Buchhaltung in containerd.

**Diagnose-Schritte:**

1. Pod-Container-Status anschauen — welche Container im Pod failen, welche laufen?
   Wenn nur EIN Sidecar versagt (z.B. csi-snapshotter), während andere Container
   im selben Pod laufen, ist es nicht "die ganze Image-Familie kaputt", sondern
   ein einzelnes Image.

2. Pull-Timestamp des verdächtigen Images vergleichen — Pull-Time in der bekannten
   Disk-Failure-Phase ist die rauchende Pistole:
   ```bash
   talosctl image ls -n <node> | grep <image>
   # Falls Pull-Zeitstempel mit Failure-Phase überlappt → korrupter Cache hochwahrscheinlich
   ```

3. Cross-Check gegen einen *gesunden* Node desselben Clusters — wenn dort dasselbe
   Image (gleicher sha256) ohne Issues läuft, ist's definitiv ein Node-lokaler
   Cache-Issue, nicht ein generelles Image-Problem.

### Eskalations-Stufen (least to most disruptive)

**Stufe 1 — Reference-Removal + Force-Pull** (surgical Probe für ein einzelnes
Image, *funktioniert nicht immer*):

```bash
talosctl image remove <image:tag> -n <node>
talosctl image remove <image>@sha256:<digest> -n <node>
talosctl image pull <image:tag> -n <node>
```

Erwartung: wenn der Pull instant zurückkommt (z.B. <1s für 40 MB), ist der
orphan Blob noch im Content-Store und containerd re-linked nur Tag+Digest auf
denselben kaputten Bytes. Echte fresh-Download dauert proportional zur
Image-Größe. **Wenn instant: zur Stufe 2 — aber vorher batch-entwirren, siehe
unten.**

Ressourcen-Impact: keine. Risiko: keine.

**Stufe 1b — Batch-Vorbereitung VOR dem Reboot** (wichtiger Schritt, sonst
Reboot-Loop):

Pull-Timestamps aller Images mit der Disk-Failure-Phase abgleichen. Verdächtig
ist alles, dessen `CREATED`-Timestamp in der Failure-Phase liegt *oder kurz
davor* (Layer-Extraktion bei großen Images kann mehrere Sekunden bis Minuten
laufen und Failure-Window-Overlap erzeugen, auch wenn der Pull-Befehl vorher
startete). Großes Image mit Pull-Start 2 Minuten vor Failure = wahrscheinlich
mitten in der Extraktion zum Failure-Zeitpunkt → korrupt-verdächtig.

```bash
# Beispiel: Failure-Phase 11:42-12:07 → Verdächtig ist alles 11:39-12:10
talosctl image ls -n <node> | awk '/T11:3[9]|T11:4[0-9]|T11:5[0-9]|T12:0[0-9]/ {print}'
```

Für JEDES verdächtige Image: Tag UND Digest entfernen. Loop dafür:

```bash
for ref in <image1:tag> <image2:tag> ... <image1>@<digest> <image2>@<digest> ...; do
  talosctl image remove "$ref" -n <node>
done
```

Verifikation per `talosctl image ls | grep <substring>` — sollte leer sein für
alle behandelten Images. **Erst dann zur Stufe 2 (Reboot).** containerd's
Boot-GC räumt *alle* unreferenzierten Blobs auf einmal weg, daher ein Reboot
genügt für N Images, vorausgesetzt deren Refs sind vorher alle entfernt.

Resourcen-Impact: keine (außer Tipparbeit). Risiko: keine. Wenn versehentlich
auch ein *sauberes* Image entfernt wird, kostet das nur einen Re-Pull beim
nächsten Pod-Schedule.

**Stufe 2 — Node-Reboot, containerd-GC** (preserves /var/log, ~2 Min Pause):

**Vorbedingung**: Stufe 1b (Batch-Reference-Removal aller Verdächtigen) muss
durchgeführt sein. Sonst räumt der Reboot-GC nur die Blobs weg, deren
Referenzen wir explizit entfernt haben — andere korrupte Layer bleiben
unverändert und tauchen beim nächsten Pod-Schedule wieder als Failure auf =
Reboot-Loop.

```bash
talosctl reboot -n <node>
# warten ~60-90s bis node wieder Ready
# nach Boot räumt containerd orphan Blobs per GC weg, fresh Pull beim
# nächsten Pod-Schedule pullt echten neuen Content
```

Ressourcen-Impact: Node kurz NotReady, Pods werden anderswo rescheduled bzw.
restartet beim Comeback. **`/var/log` bleibt erhalten** — wichtig für
forensische Analyse späterer Incidents. Container-Image-Cache bleibt
weitgehend erhalten (nur orphaned Blobs werden gepurgt).

Risiko: gering. Wenn etwas anderes auf dem Node im selben Zustand corruption
hat (z.B. weitere Layer mehrerer Images), würde ein Reboot diese nicht
adressieren — dann eskalieren.

`talosctl service containerd restart` als noch-leichtere Alternative ist
**nicht verfügbar** (Talos-API: "service containerd doesn't support restart
operation via API"). Reboot ist der billigste Weg, GC zu triggern.

**Stufe 3 — EPHEMERAL-Wipe** (last resort, *verliert /var/log*):

```bash
talosctl reset --system-labels-to-wipe EPHEMERAL --reboot -n <node>
```

Was passiert:
- `--graceful` (default): vorab cordon + drain
- Talos schreibt Wipe-Direktive in META, rebootet
- Boot-Sequence formatiert sda4 frisch (komplettes XFS), erzeugt leeres /var
- Node kommt mit unveränderter Identity (Hostname, IP, K8s-Labels — alles in
  STATE/META) zurück
- kubelet pullt *alle* benötigten Images frisch nach (~5-15 Min je nach
  Image-Volumen — typisch 2-3 GB für Rook-Ceph + Cilium + CSI-Familie)
- Rook reconciled von selbst, OSD-prepare läuft mit sauberem State

Was *nicht* passiert: keine Re-Provision, kein Re-Bootstrap, kein Manuelles
Re-Apply. Cluster-State (etcd) ist unberührt. Internal NVMes (Ceph-OSDs,
Yocto-Scratch) sind nicht auf sda → unberührt.

Ressourcen-Impact: Node ~10 Min NotReady, ~3 GB Image-Re-Downloads, /var/log
weg (deshalb last resort — Logs sind in einer Folge-Inzident-Analyse wertvoll).

Risiko: gering (cluster-state bleibt komplett), aber **verlierst die
forensische Spur** zum Original-Incident. Wenn die Disk-Failure-Ursache nicht
klar ist, BACKUP der relevanten /var/log-Dateien vor dem Wipe machen
(via `talosctl read /var/log/<file>` oder rsync via Pod).

### Vorbeugung

- USB-Boot-Hardware mit *stabilen* Bridge-Chips wählen (siehe
  `doc/ssd-selection.md` für die Bridge-Chip-Liste und die ASMedia-Falle auf
  AMD-USB4-Hosts).
- USB-Boot über *USB-A* statt USB-C, wo möglich — umgeht den USB-C/USB4-
  Negotiation-Pfad mit weniger Failure-Modes.
- Nach einem überstandenen Disk-Failure-Incident: *präventiv* die in dieser
  Zeit gepullten Images auf alle Nodes erneuern (Stufe 1+2 dieser Doc), bevor
  Pods sie das nächste Mal anfassen. Spart Diagnose-Zeit beim nächsten Mal.

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
