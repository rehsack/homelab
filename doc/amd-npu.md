# AMD NPU (XDNA) für LLM-Workloads auf f5a-01

*Snapshot 19.06.2026 — Recherche-Zwischenstand. Der Linux-NPU-LLM-Stack ist erst seit März 2026 nutzbar und bewegt sich
schnell; Kernel-/Treiber-/SHIM-Versionen vor Umsetzung gegenchecken.*

## Kernaussage

`f5a-01` (Acemagic F5A, AMD Ryzen AI 9 HX 370) kann LLM-Workloads auf **zwei unabhängigen Pfaden** aufnehmen:

1. **iGPU / Vulkan** (Radeon 890M, `gfx1150`) — funktioniert **heute**, keine Kernel-Änderung nötig. Siehe unten
   „iGPU-Pfad".
2. **NPU / FastFlowLM** (XDNA2) — **ein Kernel-Config-Flip entfernt**. Userspace und Firmware sind bereits da; es fehlt
   nur das gebaute Kernelmodul.

Die anderen Nodes haben **keinen** NPU: `ms-a2-01` (Fire Range) und `ser6` (Rembrandt+) sind NPU-los. Der NPU-Pfad ist
exklusiv `f5a-01`.

## Hardware-Fakten (live verifiziert)

| Einheit      | Befund                                                                                      | Quelle                                   |
| ------------ | ------------------------------------------------------------------------------------------- | ---------------------------------------- |
| iGPU         | Radeon 880M/890M, `gfx_v11_0` (RDNA 3.5), DevID `0x1002:0x150e`                             | `talosctl get pcidevices`, `dmesg`       |
| NPU          | **XDNA2**, „NPU Strix" (STXA), PCI `1022:17f0`                                              | `pcidevices`, amdxdna `npu4_regs.c`      |
| iGPU-Treiber | `amdgpu` gebunden, `/dev/dri/{card0,renderD128}`, KFD-Compute-Node `added device 1002:150e` | `dmesg`, `ls /dev/dri`, `/sys/class/kfd` |
| NPU-Treiber  | **nicht gebunden** — kein `/dev/accel`, kein `amdxdna` in `dmesg`                           | `ls /dev/accel` (fehlt)                  |
| Speicher     | 4 GB UMA-VRAM + **46 GB GTT** (von 96 GB RAM)                                               | `dmesg` amdgpu                           |

**XDNA1 vs. XDNA2 — nicht verwechseln:** HX 370 ist Strix Point = **XDNA2** (~50 TOPS, Copilot+-Klasse), PCI `17f0`.
XDNA1 (Phoenix/Hawk Point, PCI `1502`, ~10–16 TOPS) wird vom Linux-NPU-LLM-Stack **nicht** unterstützt — uns betrifft
das nicht. Strix *Halo* hat dieselbe XDNA2-NPU wie Strix *Point*; „fett" ist bei Halo die iGPU/Speicherbandbreite, nicht
der NPU.

## Stand des Linux-NPU-LLM-Stacks (Juni 2026)

- **Treiber**: `amdxdna.ko` ist **mainline seit Kernel 6.14** (`drivers/accel/amdxdna`). Repo `amd/xdna-driver` liefert
  zusätzlich die **XRT-SHIM**-Userspace-Bibliothek.
- **Runtime**: **FastFlowLM (FLM)** — Ollama-artiger, NPU-first-Runtime, **ausschließlich XDNA2** (Strix, Strix Halo,
  Kraken, Gorgon). Läuft vollständig auf dem NPU, ~10× energieeffizienter als CPU, Kontext bis 256k.
- **Integration**: **Lemonade 10.0** (11.03.2026) bündelt FLM als Linux-NPU-Backend → erster End-to-End-LLM-auf-NPU-Pfad
  unter Linux.
- **Container**: `hpenedones/fastflowlm-docker` existiert → direkt k8s-relevant. FLM lädt vorkompilierte **Binär-Kernels
  pro Modell** von HuggingFace (Blobs).
- **llama.cpp**: hat **keinen** NPU-Backend (zwei Feature-Requests closed/stale). Der iGPU-Pfad in llama.cpp ist
  Vulkan/ROCm, nicht NPU.

## Talos-Gap-Analyse (NPU-Pfad)

Die gesamte Lücke zwischen „NPU tot" und „LLM läuft auf dem NPU" ist **ein Kernel-Config-Flip**:

| Baustein                                       | Status                                                                      | Aktion                                                                           |
| ---------------------------------------------- | --------------------------------------------------------------------------- | -------------------------------------------------------------------------------- |
| Firmware `amdnpu/17f0_10/npu.sbin` (≥ 1.1.0.0) | ✅ schon im Talos-Base (`linux-firmware`-Pkg macht ungefiltert `cp -R usr`) | —                                                                                |
| XRT-SHIM-Userspace                             | ✅ baut der FLM-Container intern                                            | —                                                                                |
| Kernel ≥ 6.11                                  | ✅ Talos hat 6.18                                                           | —                                                                                |
| `amdxdna.ko`                                   | ❌ Talos-Kernelconfig: `# CONFIG_DRM_ACCEL_AMDXDNA is not set`              | **`=m` setzen in `siderolabs/pkgs` `kernel/build/config-amd64`**                 |
| Extension                                      | ❌ nur `drm/intel-npu` existiert                                            | neues `drm/amd-xdna` nach `intel-npu`-Vorlage (`amdxdna.ko` + `firmware/amdnpu`) |
| `/dev/accel/accel0` im Pod                     | TBD                                                                         | CDI-Spec / Device-Plugin + `--ulimit memlock=-1:-1`                              |

**Kconfig-Deps** (`AMD_IOMMU`, `DRM_ACCEL`, `PCI`, `HAS_IOMEM`, `X86_64`) sind im Talos-Kernel alle erfüllt — der Flip
ist technisch ein Einzeiler. Dass er offen ist, liegt am Zeitpunkt: der nutzbare Userspace entstand erst 03/2026, und
die Schnittmenge {Strix} ∩ {Talos} ∩ {NPU-LLM} ist winzig. Kein bestehendes Issue/PR in `siderolabs/pkgs` oder
`siderolabs/extensions`.

## iGPU-Pfad (Referenz, funktioniert heute)

- amdgpu gebunden, `/dev/dri/renderD128` vorhanden, KFD-Compute-Node aktiv.
- **Vulkan**: llama.cpp läuft direkt — Container braucht nur RADV-ICD + `/dev/dri`.
- **ROCm/HIP**: Kernelseite bereit (KFD-Node). Userspace-Vorbehalt: `gfx1150` braucht ROCm ≥ 6.4 oder
  `HSA_OVERRIDE_GFX_VERSION=11.0.0` (Container-Sache).
- Modellgewichte passen über GTT (bis ~46 GB von 96 GB RAM).

## Caveats (vor Umsetzung verifizieren)

1. **ABI-Match Treiber ↔ SHIM**: in-Kernel-`amdxdna` (6.18) muss zur `libxrt-npu`-/ SHIM-Version passen, die FLM
   erwartet. Quellen nennen „Kernel 7.0 oder amdxdna-Backports" wegen Last-Minute-Treiberänderungen — **das ist das
   reale Integrationsrisiko**, gerade bei Talos' festem Kernel.
2. **Opake NPU-Kernels**: FLM lädt pro Modell vorkompilierte Blobs von HuggingFace. Frei nutzbar, aber gegen die
   Declarative-/Daten-Integritäts-Linie ein bewusster Kompromiss + ein Offline-/Air-Gap-Thema.
3. **memlock unlimited** + NPU als Single-Device (kein Pod-Sharing) — fürs Homelab unkritisch, aber zu konfigurieren.

## Offene Recherche-Stränge

- **(A) ABI-Realität**: Läuft FLM gegen den 6.18-`amdxdna`? Issues bei `FastFlowLM/FastFlowLM` und `amd/xdna-driver` zu
  Kernel 6.15–6.18 / SHIM-Mismatch.
- **(B) Talos-Weg**: hat jemand `amdxdna` auf Talos/immutable Distros gefahren? PR-Entwurf `CONFIG_DRM_ACCEL_AMDXDNA=m`
  \+ `drm/amd-xdna`-Extension.
- **(C) Modell-/Perf-Realität**: FLM-Modellliste + Benchmarks vs. iGPU/Vulkan für Ziel-Workloads (MCP-Kontext,
  Doku-Rendering).

## Quellen

- Phoronix: <https://www.phoronix.com/news/AMD-Ryzen-AI-NPUs-Linux-LLMs>
- FastFlowLM: <https://github.com/FastFlowLM/FastFlowLM>
- Lemonade (Linux NPU): <https://lemonade-server.ai/flm_npu_linux.html>
- fastflowlm-docker: <https://github.com/hpenedones/fastflowlm-docker>
- amd/xdna-driver: <https://github.com/amd/xdna-driver>
- Ryzen AI Software Docs: <https://ryzenai.docs.amd.com/en/latest/relnotes.html>
