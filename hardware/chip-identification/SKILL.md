---
name: chip-identification
description: Reference for identified chips/ICs, cards, and GPU carrier systems from PCB photos. Use this skill when identifying or looking up chips, riser cards, or GPU baseboards found in the user's hardware collection.
---
# Chip Identification Reference

A log of chips identified from macro PCB photos, including part numbers, manufacturers, and board context.

---

## ICS 9ZXL1950DKIL

| Field | Value |
|---|---|
| **Manufacturer** | ICS (Integrated Circuit Systems, now Renesas/IDT) |
| **Part Number** | 9ZXL1950DKIL |
| **Function** | Spread-spectrum clock generator / synthesizer |
| **Package** | QFN (observed ~72-pin) |
| **Date Code** | 2403538 |
| **Origin** | TWN (Taiwan), year 2024 (date code 2403538) |
| **PCB** | RBE162548_V52 — RTE162P54B-2UR SFF-8654-8i riser for SXM2 baseboard |
| **Photo** | `PXL_20260322_133229089.MACRO_FOCUS.jpg` |

### Notes
- Used on SFF-8654-8i riser cards to generate reference clocks for PCIe 4.0 / SerDes interfaces.
- The `DKIL` suffix indicates a specific speed/spread variant.
- Identified 2026-03-22 from macro photo of the back side of the RTE162P54B-2UR riser card.

---

## RTE162P54B-2UR — SFF-8654-8i Riser Card for SXM2 Baseboard

| Field | Value |
|---|---|
| **Model** | RTE162P54B-2UR (alias: RBS-16G4-2P54) |
| **Board** | RBE162548_V52 |
| **P/N** | 97.R566V01 |
| **S/N** | YR260130180151 |
| **Function** | SFF-8654-8i riser card — connects SXM2 GPU baseboard via SlimSAS PCIe 4.0 |
| **Connectors** | 2× SFF-8654-8i (SlimSAS x8, internal), bracket labels SLIM6A51 / SLIM6A52 |
| **PCIe bandwidth** | PCIe 4.0 at 16 GT/s per lane; x8 per connector = ~16 GB/s, x16 total = ~32 GB/s |
| **Form factor** | ~147 × 68.5 mm, 2U rack, ~1W, 3.3V PCIe slot powered |
| **Certifications** | CE, FCC, RoHS, UL E511912, 94V-0, date code 0226 |
| **Photos** | `PXL_20260322_133210597.jpg` (back/bracket), `PXL_20260322_133229089.MACRO_FOCUS.jpg` (ICS chip) |

### Notes
- Part of the **P54 connector family** — same SFF-8654-8i SlimSAS connectors as on the TNS-2SXM2-4P54 300G dual-SXM2 baseboard.
- The "SLIM" in SLIM6A51/SLIM6A52 bracket silkscreen refers to SlimSAS (SFF-8654).
- The "P54" suffix in the model number identifies the SFF-8654 (SlimSAS) connector family.
- Primary ASIC (PCIe switch/bridge) on front side not yet photographed — unknown.
- Identified 2026-03-22 from photos of the bracket/back side.

---

## TNS-2SXM2-4P54 — Dual-SXM2 GPU Carrier Baseboard

| Field | Value |
|---|---|
| **Model** | TNS-2SXM2-4P54 |
| **Type** | Dual-SXM2 GPU carrier / baseboard |
| **GPU Slots** | 2× SXM2 (NVIDIA SXM2 form factor) |
| **Host Interface** | 4× SFF-8654-8i (SlimSAS x8, PCIe 4.0) — "4P54" in model name |
| **PCIe bandwidth** | PCIe 4.0, 4× x8 = ~64 GB/s aggregate to host |
| **Companion riser** | RTE162P54B-2UR — 2× per baseboard (each riser carries 2× SFF-8654-8i) |
| **Host platform** | Intel Xeon server (confirmed via `IntelRCSetup` tab in AMI Aptio BIOS v2.18.1263) |
| **Photos** | `PXL_20260322_114203660.MP.jpg` (BIOS: PCI resource error), `PXL_20260322_125720017.jpg` (BIOS: CSM config) |

### Model Name Breakdown
- **TNS** — product line prefix
- **2SXM2** — 2× SXM2 GPU slots
- **4P54** — 4× SFF-8654 (SlimSAS) ports; same "P54" family identifier as on RTE162P54B-2UR risers

### Installation Notes
- **PCI Resource Exhaustion (BAR space)**: Populating both SXM2 slots with high-BAR GPUs (e.g., NVIDIA V100 SXM2) alongside other PCIe cards can trigger "Insufficient PCI Resources Detected" in AMI Aptio BIOS.
  - Fix: Enable **Above 4G Decoding** under Advanced → PCI Configuration.
  - On Intel Xeon platforms: also check **IntelRCSetup** → PCIe resource allocation settings.
  - Observed during 2026-03-22 installation session (BIOS v2.18.1263, AMI copyright 2020).
- **CSM**: Host BIOS had CSM enabled with Storage/Video in Legacy mode, Other PCI in UEFI mode. Note that "Above 4G Decoding" typically requires UEFI boot — verify compatibility if CSM is active.
- Baseboard not yet directly photographed; entry based on riser card analysis (RTE162P54B-2UR) and host BIOS screenshots.
