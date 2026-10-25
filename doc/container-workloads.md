# Container-Workloads — Lokale Entwicklungsumgebung

Stand: 2026-03-21

## Täglicher Bedarf

| Container          | Quelle                                                             | Zweck                                                                     | Bind Mounts?                          | Compose? |
| ------------------ | ------------------------------------------------------------------ | ------------------------------------------------------------------------- | ------------------------------------- | -------- |
| **doctoolchain**   | `doctoolchain/doctoolchain:v3.4.2`                                 | Dokumentations-Pipeline (AsciiDoc → diverse Formate)                      | Ja (Projektverzeichnisse)             | Nein     |
| **TeXLive**        | `danteev/texlive:latest`                                           | LaTeX-Builds                                                              | Ja (Projektverzeichnisse)             | Nein     |
| **tpo42-asciidoc** | Eigenbau, `~/data/Projects/OSS/container/tpo42-asciidoc-container` | AsciiDoc-Toolchain (Ruby/Asciidoctor, pre-commit), wird zunehmend wichtig | Ja (`/workspace`, `/build`, `/cache`) | Nein     |

## Kubernetes / Homelab (nicht lokal laufen)

| Container | Zweck                                                           | Ziel                                            |
| --------- | --------------------------------------------------------------- | ----------------------------------------------- |
| **Kroki** | `yuzutech/kroki` — Diagramm-Rendering (PlantUML, Mermaid, etc.) | K8s-Deployment im Homelab, Anbindung an Jenkins |

## Kommende Woche — Delphin

| Container               | Zweck                                                                                 | Hinweise                                                                                                                                  |
| ----------------------- | ------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------- |
| **Yocto-SDK-Container** | Fertige Toolchain für Cortex-M3, Cortex-M8 und Host (x86_64, perspektivisch Apple M5) | Vorbild: `~/data/devel/Windhoff/yocto-show-case/docker-yocto-build/`. Nächste Stufe: vorinstalliertes SDK. Jenkins-CI-Einsatz bei Delphin |

## Entsorgt / Alles kann weg

| Container / Volume                                             | Grund                                                               |
| -------------------------------------------------------------- | ------------------------------------------------------------------- |
| `gitlab-ext.sigma-chemnitz.de:5000/vizaar/*` (2 Images, 25 GB) | viZaar ist Geschichte                                               |
| `build-serialportcom-native*` Volumes                          | Altes viZaar-Image, Altlast                                         |
| `dementia-app_*` Volumes (8 Stück)                             | Waren Compose-Experimente, Projekt wird von Grund auf neu aufgebaut |
| Anonyme Volumes (8 Hex-Hash-Volumes)                           | Altlast                                                             |
| `vscode` Volume                                                | Vermutlich VS Code Dev Container Integration                        |

## Cleanup

`~/bin/cleanup-docker` — ein Script, zwei Modi:

| Aufruf                          | Modus  | Zweck                                                        |
| ------------------------------- | ------ | ------------------------------------------------------------ |
| `cleanup-docker`                | medium | Während der Arbeit: Build-Cruft weg, getaggte Images bleiben |
| `cleanup-docker-full` (Symlink) | full   | Alles stoppen und wegräumen                                  |
| `cleanup-docker --full`         | full   | Alternativ per Flag                                          |

Beide verstehen `--quiet` für Cron. Cron-Beispiel (Sonntag 03:00):

```
0 3 * * 0 /Users/sno/bin/cleanup-docker-full --quiet
```

### Warmup nach Full-Cleanup

Nach full-Cleanup werden essentielle Images neu geholt/gebaut, damit Montagmorgen sofort arbeitsfähig:

| Image              | Aktion                                                             |
| ------------------ | ------------------------------------------------------------------ |
| **TeXLive**        | `docker pull danteev/texlive:latest`                               |
| **doctoolchain**   | `dtcw docker pull` (PR nötig, siehe TODO)                          |
| **tpo42-asciidoc** | `~/data/Projects/OSS/container/tpo42-asciidoc-container/bin/adcbw` |

## Lokales Kubernetes

Docker Desktop bringt lokales K8s mit. Wird gebraucht wenn unterwegs (Homelab nicht erreichbar). Full-Cleanup räumt
K8s-Container mit auf; Docker Desktop zieht sie bei Bedarf wieder (~1 GB).

## Mittelfristig: Colima als Docker-Desktop-Ersatz

- **Colima** v0.10.1 (Lima v2.1.0 Backend) unterstützt VZ + VirtioFS (Default auf macOS 13+)
- Auf Intel Macs: VZ hat Netzwerk-Probleme → QEMU + sshfs nötig
- Auf Apple Silicon (Kerstins M4): VZ + VirtioFS performant
- Maintainer: Jan Dubois (persönlicher Kontakt aus CPAN-Zeiten)
- **Apple Container** (macOS 26, Apple Silicon only): v0.1.x, kein Compose, keine Bind Mounts — zu früh
- Entscheidung: Nicht jetzt, aber Option für später

## TODO

- [ ] **dtcw PR** — `docToolchain/docToolchain` forken, `docker pull` in dtcw einbauen, PR an Ralf, eigenes dtcw
  deployen nach `/usr/local/bin/dtcw`
- [ ] **cleanup-docker** fertigstellen — Warmup-Block einbauen, testen (medium + full)
- [ ] Cron-Job für `cleanup-docker-full --quiet` einrichten (Sonntag 03:00)
- [ ] Braucht Kerstin eigenes Container-Setup? (Colima auf M4 wäre naheliegend)
