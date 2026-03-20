---
name: huananzhi-h12d-8d
description: Hardware reference for the Huananzhi H12D-8D single-socket AMD EPYC server mainboard. Use this skill when asked about memory slot numbering, DIMM population order, board layout, supported CPUs, or other hardware specifications for this board.
---
# Huananzhi H12D-8D Hardware Reference

## Overview

The **Huananzhi H12D-8D** (华南金牌 H12D-8D) is a **single-socket** AMD EPYC server mainboard.

- **Form factor:** ATX (305 × 245 mm)
- **CPU sockets:** 1× SP3 (AMD EPYC)
- **Supported CPUs:** AMD EPYC 7002 (Rome), 7003 (Milan)
- **Memory type:** DDR4 ECC RDIMM / LRDIMM, up to 3200 MHz
- **Memory slots:** 8 total — all serving the single CPU
- **Memory configuration:** True octa-channel (8-channel) from one CPU
- **Max RAM:** up to 2 TB

## Expansion Slots & Storage

- 4× PCIe 4.0 x16 slots
- 3× M.2 NVMe PCIe 4.0 x4
- 4× SATA3 ports
- 3× SFF-8643 (Mini-SAS HD) ports

## Networking

- 2× Intel I226-V 2.5 GbE
- 1× dedicated IPMI/BMC port (BMC module required separately)

## Power

- 1× 24-pin ATX
- 2× 8-pin EPS (CPU power)

## Memory Slot Numbering

Slots are **labeled MM1–MM8** on the board silkscreen. All 8 slots belong to the single CPU.

| Slot | Channel | Notes                              |
|------|---------|------------------------------------|
| MM1  | A       | **First slot — install here first** |
| MM2  | B       |                                    |
| MM3  | C       |                                    |
| MM4  | D       |                                    |
| MM5  | E       |                                    |
| MM6  | F       |                                    |
| MM7  | G       |                                    |
| MM8  | H       |                                    |

**MM1 is the first slot** on this board.

## DIMM Population Rules

For optimal bandwidth, spread DIMMs across channels before doubling up in any channel.

| Number of DIMMs | Recommended slots            |
|-----------------|------------------------------|
| 1               | MM1                          |
| 2               | MM1, MM5 (channels A + E)    |
| 4               | MM1, MM3, MM5, MM7           |
| 8 (full/optimal)| MM1–MM8                      |

- Use identical DIMMs (capacity, speed, rank) for best compatibility.
- All 8 DIMMs populated gives maximum bandwidth (true 8-channel interleaving).

## Known Quirks and Limitations

- ⚠️ The official Huananzhi PDF manual (`http://www.huananzhi.com/uploads/file/20260212/1770877734213650.pdf`) is slow to load and frequently times out. Use the Manualzz mirror instead.
- BIOS update may be needed for EPYC 7003 (Milan) support — check the Huananzhi product page for current BIOS versions.
- Despite the "dual" in H12**D**, this board has only **one** CPU socket (single-socket).

## References

- [User Manual on Manualzz](https://manualzz.com/doc/84402668/huananzhi-h12d-8d-user-manual) — English mirror of official manual
- [Official Huananzhi product page (EN)](http://www.huananzhi.com/en/list_6/183.html)
- [Official manual PDF (CN)](http://www.huananzhi.com/uploads/file/20260212/1770877734213650.pdf) — may time out
