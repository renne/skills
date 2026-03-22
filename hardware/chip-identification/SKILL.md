---
name: chip-identification
description: Reference for identified chips/ICs on PCBs from macro photos. Use this skill when identifying or looking up chips found on circuit boards in the user's hardware collection.
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
| **Origin** | TWN (Taiwan), week 23, year 2012 |
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
