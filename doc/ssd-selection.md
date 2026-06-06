# SSD-Auswahl: Yocto-Scratch auf ms-a2-01

*Snapshot 29.05.2026 — Preise und Modellverfügbarkeit ändern sich monatlich. Vor Bestellung in Geizhals/Idealo gegenchecken.*

## Kontext

Für die dritte M.2-Disk auf `ms-a2-01` (Minisforum MS-A2) wird eine 4-TB-Gen4-NVMe als
*lokales volatile scratch* gesucht — primär für Yocto/OCI-Builds. Niemals Ceph, niemals
mirroring. Build-Artefakte werden durch explizite CI-Schritte in Registries gepushed
(siehe `CLAUDE.md` → "Storage Tiering").

Die beiden anderen M.2-Slots halten 2× Samsung 990 PRO 4 TB als Rook-Ceph OSDs. Die
Scratch-Disk muss per CEL `diskSelector` davon getrennt werden — siehe unten.

## Hardware-Constraints MS-A2

Der MS-A2 hat **3× M.2 PCIe 4.0** Slots, aber das Handbuch warnt: bei voller Bestückung
fallen typischerweise zwei der drei Slots thermisch auf **PCIe 3.0** zurück (kein
aktives M.2-Kühlsystem, gemeinsame Wärmezone unter dem Bodenblech). Das ist explizit
*kein* Defekt, sondern Designentscheidung.

Konsequenzen für die SSD-Wahl:

- **Gen5 sofort raus** — wäre selbst ohne Throttling thermisch unmöglich in dem
  Gehäuse, und Gen5-Bandbreite ist für *keine* Workload hier nutzbar.
- **Gen4 vs. Gen3 ist für Ceph irrelevant**: CephFS-Replikation läuft über das
  Netz, nicht über lokale SSD-Bandbreite. Aktueller Stand: 1-GbE-Switch
  (Zyxel GS2220-24), f5a-01 hat ohnehin nur GbE. Der MS-A2 hat zwar 2× 10 GbE,
  die aber bis zu einem Switch-Upgrade auf 1 GbE auflaufen.
  - **1 GbE ≈ 125 MB/s** ← aktuelle Realität
  - 2,5 GbE ≈ 300 MB/s
  - 10 GbE ≈ 1,25 GB/s
  - Gen3 NVMe ≈ 3,5 GB/s, Gen4 NVMe ≈ 7 GB/s
  → Das Netz ist immer der Bottleneck. Die OSDs sind nie SSD-limited.
  Bei 1 GbE ist selbst eine Gen3-SSD ~28× schneller als die Replikation
  je leisten kann; selbst Gen5 würde daran nichts ändern.
- **Gen4 vs. Gen3 ist für Yocto ebenfalls weitgehend irrelevant**: Yocto ist
  IOPS-/Latenz-limitiert, nicht bandbreiten-limitiert. NAND-Zugriffszeit und
  FTL-Lookups dominieren — die ändern sich nicht zwischen PCIe-Generationen.
  Sequential-Reads (Tarball-Extract) profitieren marginal von Gen4, aber das ist
  ein Bruchteil der Gesamt-Build-Zeit.

→ Praktisches Fazit: **Jeder Gen4-4TB-NVMe reicht.** Selbst wenn er in einem
Slot auf Gen3 zurückfällt, ist die Real-World-Performance für unsere Workloads
unverändert. Modellwahl darf rein nach Preis/Reife/DRAM-Architektur getroffen
werden, ohne Gedanken an "schade um die verlorene Gen4-Bandbreite".

## Yocto-Workload-Profil

Yocto-Builds sind **massiv random-access und metadata-dominiert** — nicht sequentiell.
Wichtige Phasen und ihr I/O-Charakter:

| Task           | Pattern                                                            |
|----------------|--------------------------------------------------------------------|
| `do_unpack`    | Tarball-Extract: viel kleine Files erzeugen (metadata-heavy)       |
| `do_configure` | autoconf/cmake/meson: hochfrequente `stat`/`open`, kleine R/W      |
| `do_compile`   | gcc tiefe Header-Ketten → random reads pro TU, kleine .o-writes    |
| `do_install`   | mkdir/cp/hardlink-Stürme, pseudo-DB updates — metadata-heavy       |
| `do_package`   | File-Scanning, Dedup, Package-Build — random reads + writes        |
| sstate lookup  | Random reads kleiner Index-Files                                   |
| sstate write   | tar.zst-Erstellung — sequentiell-ish                               |
| `rm_work`      | massive Unlink-Stürme — metadata-dominiert                         |

→ FTL wird unter dieser Last hart beansprucht. DRAM-equipped NVMes sind *spürbar*
besser als DRAM-less + HMB, weil On-Board-DRAM-Lookups sub-µs sind, während HMB
über PCIe round-trip läuft. SLC-Cache puffert Burst-Writes, kann bei mehreren parallelen
Builds aber überlaufen.

## Marktlage 4-TB-NVMe (Mai 2026)

### Generationsstand

- **Gen5 ist Premium-Segment**: Samsung 9100 PRO, SanDisk WD_BLACK SN8100, Crucial T710,
  Lexar NM1090 PRO. 4 TB liegt bei €560-640.
- **Gen5 in Mini-PCs aktiv problematisch**: ohne aktive M.2-Kühlung Thermal-Throttling
  >85 °C, kein Yocto-Vorteil (Workload ist nicht durchsatzlimitiert).
- **Gen4 bleibt Sweet Spot** für Workstation-Mini-PCs wie MS-A2 (3× M.2 PCIe 4.0).
  4 TB liegt bei €220-450.

### Markenverschiebungen

- **Western Digital ist Oktober 2025 aus dem SSD-Geschäft ausgestiegen.** Marke
  "WD_BLACK" gehört jetzt SanDisk:
  - **SN850X**: nur noch Lagerräumung, strategisch tot → nicht mehr empfehlen
  - **SN7100**: aktueller Gen4-Pick (DRAM-less, energieeffizient)
  - **SN8100**: Gen5-Flagship
- **Kingston KC3000** ist Phison-E18-Generation, gegenüber neueren E25/E26-Designs
  preislich nicht mehr wettbewerbsfähig. Nachfolger Fury Renegade.
- **Lexar NM790 / TeamGroup MP44 / Crucial T500** sind weiterhin current Gen4-Produkte
  (NM1090 PRO, P510 sind Gen5 in separater Klasse).

### Gen4-4TB-Vergleich (Geizhals.de, 29.05.2026)

| Modell                                  | DRAM | TBW  | Preis ab | Controller / NAND              |
|-----------------------------------------|------|------|----------|--------------------------------|
| TeamGroup MP44 4TB                      | nein (HMB) | 3000 | €223,90  | Maxio MAP1602 + YMTC TLC       |
| WD_BLACK SN7100 4TB (retail)            | nein (HMB) | 2400 | €289,99  | SanDisk in-house + BiCS6 TLC   |
| Crucial T500 4TB (ohne Heatsink)        | **ja**     | 2400 | €355,50  | Phison E25 + Micron TLC        |
| Lexar NM790 4TB                         | nein (HMB) | 3000 | €367,90  | Maxio MAP1602 + YMTC TLC       |
| ~~WD_BLACK SN850X 4TB~~                 | ja         | 2400 | €399-450 | EOL-Marke, nicht mehr empfehlen|
| ~~Kingston KC3000 4TB~~                 | ja         | 3200 | €482,90  | Generation überholt            |

## Empfehlung

Da Yocto random-dominant ist, ist DRAM ein realer Vorteil — die ursprüngliche
"DRAM-less ist OK weil Yocto sequenziell"-Annahme ist *falsch*.

1. **Beste Wahl für Yocto-spezifisch**: **Crucial T500 4TB** (€356). DRAM, Phison E25,
   guter sustained random Durchsatz. ~€130 Aufpreis gegenüber MP44 ist gerechtfertigt.
2. **Budget-Pick / "Rotation statt Premium"-Strategie**: TeamGroup MP44 4TB (€224 wenn
   Preis stabil). DRAM-less, aber 3000 TBW und bewährter Maxio-Stack. Akzeptabler
   Kompromiss, wenn Preis-Lücke groß bleibt.
3. **Mittelweg**: WD_BLACK SN7100 4TB (€290). Aktuelles SanDisk-Design, DRAM-less aber
   reife Firmware. Wenn DRAM-less akzeptabel, vor NM790 wählen (Hardware-zwilling der
   MP44, aber teurer).

**TBW-Realität**: Selbst die niedrigste TBW dieser Liste (2400) ergibt bei 100 GB/Tag
~65 Jahre, bei 500 GB/Tag ~13 Jahre. TBW ist für ein Solo-Setup kein praktischer Limiter
— Rotation kommt nicht aus Verschleiß, sondern aus Hardware-Updates.

## CEL DiskSelector (Talos 1.13)

Erst nach Hardware-Erhalt mit `talosctl get disks -o yaml --insecure` echten
`disk.model`-String prüfen (Samsung mischt manchmal Trailing-Spaces oder
"MZ-V..."-Varianten rein).

```yaml
# Ceph OSDs (Rook-Ceph)
diskSelector:
  match: disk.transport == "nvme" && disk.model == "Samsung SSD 990 PRO 4TB"

# Yocto-Scratch (UserVolumeConfig)
diskSelector:
  match: disk.transport == "nvme" && disk.model != "Samsung SSD 990 PRO 4TB"
```

**Vorbehalt**: Funktioniert nur, wenn die Scratch-Disk *nicht* auch eine 990 PRO 4TB
wird. Fallback bei gleichem Modell: Serial-Pinning über `disk.serial == "..."`
(weniger elegant, robust). Daher Empfehlung: bewusst anderes Modell wählen.

## Filesystem

Talos UserVolumeConfig unterstützt `xfs` (default) und `ext4`. Btrfs nicht in
UserVolume-Spec.

- **Empfehlung: xfs** — besseres Verhalten bei sehr vielen Files + Mass-Unlinks
  (`rm_work`-Stürme).
- Mount-Optionen wichtiger als FS-Wahl:
  - `noatime,nodiratime` — atime-Updates pro Read killen sonst SSD-Endurance bei
    Yocto-File-Count
  - `lazytime` — reduziert weitere Metadata-Writes
  - `discard` (oder `fstrim`-Timer) — moderne NVMes wollen TRIM sehen

## Quellen (Stand 29.05.2026)

- [TeamGroup MP44 4TB — Geizhals.de](https://geizhals.de/teamgroup-mp44-4tb-tm8fpw004t0c101-a2966530.html)
- [WD_BLACK SN7100 4TB retail — Geizhals.de](https://geizhals.de/western-digital-wd-black-sn7100-nvme-ssd-4tb-wdbelg0040bbk-a3458986.html)
- [Crucial T500 4TB (CT4000T500SSD8) — Geizhals.de](https://geizhals.de/crucial-t500-ssd-4tb-ct4000t500ssd8-a3270045.html)
- [Lexar NM790 4TB — Geizhals.de](https://geizhals.de/lexar-nm790-4tb-lnm790x004t-rnnn-a2980705.html)
- [WD_BLACK SN850X 4TB — Geizhals.de](https://geizhals.de/western-digital-wd-black-sn850x-nvme-ssd-4tb-wds400t2x0e-00bca0-a2770181.html)
- [WD_BLACK SN8100 4TB — Geizhals.de](https://geizhals.de/western-digital-wd-black-sn8100-nvme-ssd-4tb-wds400t1x0m-00cmt0-a3483434.html)
- [Samsung 9100 PRO 4TB — Geizhals.de](https://geizhals.de/samsung-ssd-9100-pro-4tb-mz-vap4t0bw-a3427123.html)
- [Lexar NM1090 PRO 4TB Review — Tom's Hardware](https://www.tomshardware.com/pc-components/ssds/lexar-nm1090-pro-4tb-ssd-review)
- [WD Black SN7100 Review — Tom's Hardware](https://www.tomshardware.com/pc-components/ssds/wd-black-sn7100-ssd-review)

---

## Postskriptum — was tatsächlich gekauft / verbaut wurde (Juni 2026)

Dieses Dokument war als Markt-Recherche für die geplante 3. interne NVMe
(Yocto-Scratch) gedacht. In der praktischen Umsetzung sind zwei Sachen
abweichend gelaufen:

**Yocto-Scratch: gar nicht gekauft, sondern recycelt.** Die ex-NUC Samsung
SSD 970 EVO Plus 2 TB (originally Boot-Disk der gescheiterten NUC-Reanimation,
extern in einem USB-C-Gehäuse als ms-a2-01-Boot-Drive eingesetzt, dort
wegen Bridge-Inkompatibilität gescheitert) ist in den 3. M.2-Slot intern
verbaut. `disk.model = "Samsung SSD 970 EVO Plus 2TB"` matched die
CEL-Negation `!= "Samsung SSD 990 PRO 4TB"` sauber — keine Architektur-
Anpassung nötig. Der ganze 4-TB-Markt-Snapshot oben ist damit historische
Referenz, brauchbar falls die 970 EVO Plus mal ausfällt und ein Neukauf ansteht.

**Boot-Disk: gekauft, aber USB-C, nicht intern.** Nach zwei Bridge-Failures
(ASMedia ASM2362-basierte Gehäuse, beide am USB4-Front-Port des MS-A2 in
Disconnect-Loops gelaufen) wurde die Architektur auf eine UGREEN-USB-C-NVMe-
Bridge (Modell 15512, RTL9210B-Chip, Gen 2x1) + SanDisk PLUS 250 GB Gen3
NVMe gewechselt. ~€80 netto Gesamt. Verbaut am *rückseitigen* USB 3.2 Gen 2x1
Port (NICHT am USB4 vorne — Mode-Negotiation-Komplexität umgangen). Läuft.

**Bridge-Chip-Lehre für die Akten:**

- **ASMedia ASM2362 / ASM2364**: auf AMD-Ryzen-USB4-Hosts (Fire Range / Phoenix /
  Rembrandt) reproduzierbare USB4-Mode-Negotiation-Loops, Disconnect-Storm,
  unbenutzbar. **Nicht kaufen** für Hosts mit USB4-Ports.
- **ASMedia ASM2464PD** (USB4-nativ): laut egpu.io / Arch-Forums hat dieselbe
  Linux-Compat-Problematik auf AMD-USB4-Hosts wie ASM2362. Premium-Preis,
  keine Verbesserung. **Auch nicht kaufen** für AMD-USB4.
- **JMicron JMS583**: chip-level mature, aber Board-/Kühl-Qualität variiert
  (ICY BOX IB-1817M-C31 hatte hier zickige Connection/Lesefehler nach Wärme).
- **Realtek RTL9210B** (Gen 2x1, 10 Gbps): empirisch stabil, in UGREEN-Gehäusen
  bewährt. **Aktuelle Empfehlung** für USB-C-NVMe-Boot.

**Marktkontext zum Zeitpunkt des Kaufs**: NAND-Preise im Q2 2026 in massiver
Aufwärtsbewegung (Tom's Hardware: +246 % seit Q1/2025, +70 % in 60 Tagen,
NAND-Kapazität wird in HBM für AI-GPUs umgeleitet). Forecast bis Ende 2027
weiter steigend. Wenn ein erneuter Kauf ansteht, ist der Markt-Snapshot
oben spätestens nach 4 Wochen obsolet — re-recherchieren auf Geizhals.
