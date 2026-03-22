---
name: rte162p54b-2ur
description: RTE162P54B-2UR SFF-8654-8i PCIe riser card — converts PCIe 3.0 x16 to 2× SFF-8654-8i (SlimSAS x8) for SXM2 GPU baseboard connections. Use this skill when working with this riser card or the P54/SlimSAS connector family.
---
# RTE162P54B-2UR — SFF-8654-8i Riser Card

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
| **Photos** | See [Photos](#photos) section below |

## Notes
- Part of the **P54 connector family** — same SFF-8654-8i SlimSAS connectors as on the TNS-2SXM2-4P54 dual-SXM2 baseboard.
- The "SLIM" in SLIM6A51/SLIM6A52 bracket silkscreen refers to SlimSAS (SFF-8654).
- The "P54" suffix in the model number identifies the SFF-8654 (SlimSAS) connector family.
- Contains an **ICS 9ZXL1950DKIL** spread-spectrum clock generator on the back side (generates PCIe 4.0 / SerDes reference clocks). See [chip-identification skill](../chip-identification/SKILL.md).
- Primary ASIC (PCIe switch/bridge) on front side not yet photographed — unknown.
- Identified 2026-03-22 from photos of the bracket/back side.

## Photos

### Back side / bracket / SlimSAS connectors
![RTE162P54B-2UR back side showing bracket and SFF-8654-8i SlimSAS connectors](photos/PXL_20260322_133210597.jpg)

### ICS 9ZXL1950DKIL clock generator (macro)
![ICS 9ZXL1950DKIL spread-spectrum clock generator chip on PCB back](photos/PXL_20260322_133229089.MACRO_FOCUS.jpg)

## Usage with TNS-2SXM2-4P54
- 2× risers are included in the TNS-2SXM2-4P54 kit (eBay 136861481597)
- Each riser carries 2× SFF-8654-8i, connects to one side of the baseboard's X16/X8 port group
- Full-height and half-height brackets included
