---
name: machinist-mra9-v1
description: Hardware reference for the Machinist X99 MRA9 / E5-MR9A hardware revision v1.0 — an ATX LGA 2011-3 budget motherboard supporting Xeon E5 v3/v4 CPUs with DDR4 quad-channel memory. Use when managing, troubleshooting, or configuring any system running on this board: covers specs, CPU/RAM compatibility, BIOS quirks, storage, PCIe layout, fan control, virtualization (IOMMU/VT-d), and known v1.0-specific issues.
---
# Machinist X99 MRA9 / E5-MR9A — Hardware Revision v1.0

Also marketed as: **X99-MR9A**, **E5-MR9A**, **Machinist MRA9**  
Revision v1.0 is the original production run — no debug display, no onboard power/reset buttons, no TPM 2.0 header, no integrated I/O shield.

## Board Overview

| Parameter | Value |
|---|---|
| Manufacturer | Machinist (Chinese budget OEM) |
| Model | MRA9 / MR9A (hardware rev v1.0) |
| Form Factor | ATX (~216–280 × 215–283 mm) |
| Socket | LGA 2011-3 |
| Chipset | Intel B85 (most units); Q87 or C226 on some variants |
| BIOS Chip | Winbond 25Q128FV |
| BIOS Type | Legacy + UEFI (AMI) |

> **Note:** Despite "X99" marketing, this board uses a **desktop chipset (B85/Q87/C226)**, NOT Intel X99 or C612. This limits ECC, PCIe bifurcation, VROC, IPMI, and overclocking.

## CPU Support

| Parameter | Value |
|---|---|
| Architecture | LGA 2011-3 (Haswell-EP / Broadwell-EP) |
| Supported CPUs | Intel Xeon E5-26xx v3, E5-26xx v4; Core i7-58x0X / i7-68x0X |
| Recommended CPUs | E5-2620 v3/v4, E5-2650 v3/v4, E5-2680 v3/v4, E5-2699 v3/v4 |
| TDP headroom | 4-phase VRM (8 virtual via doublers); active VRM cooling recommended for >120 W CPUs |
| Overclocking | None — desktop chipset does not support multiplier adjustment |
| Turbo unlock | Available via patched Huananzhi BIOS (see BIOS section) |
| Temperature sensor | **Unreliable on v1.0** — commonly reads 120 °C or random values; do not trust |

### VRM Details (v1.0)
- MOSFETs: SM4503NHKP (high-side) + SM4508NHKP (low-side)
- 4 physical phases, 8 virtual via doublers
- Add a 40–60 mm fan aimed at VRMs for sustained high-TDP Xeon loads (E5-2680 v4 and above)

## Memory

| Parameter | Value |
|---|---|
| Slots | 4× DDR4 DIMM |
| Channels | Quad-channel (populate all 4 for quad-channel) |
| Capacity | Up to 128 GB total |
| Speed | DDR4-2133 / 2400 (JEDEC) |
| ECC support | ECC modules POST and boot; **ECC error correction is NOT functionally enabled** (desktop chipset limitation) |
| Non-ECC | Supported |
| RDIMM/LRDIMM | Registered DIMMs work; LRDIMM compatibility varies |

> **ECC Caveat:** Linux `dmesg` / `edac-util` will report no ECC support. This is expected — the hardware accepts ECC DIMMs electrically but the chipset/BIOS does not enable error correction logic.

## Storage

| Interface | Quantity | Speed | Notes |
|---|---|---|---|
| SATA 3.0 (6 Gbps) | 4 | 6 Gbps | Primary storage |
| SATA 2.0 (3 Gbps) | 2 | 3 Gbps | Secondary/legacy |
| M.2 (NVMe) | 1 | PCIe 3.0 x4, up to 32 Gbps | 2280 form factor; CPU lanes |
| M.2 (NGFF/SATA) | 1 | SATA | SATA M.2 drives only |

## PCIe Expansion

| Slot | Lanes | Gen | Source | Notes |
|---|---|---|---|---|
| PCIe x16 #1 | x16 | Gen 3 | CPU | Primary GPU slot |
| PCIe x16 #2 | x16 | Gen 3 | CPU | Secondary GPU / add-in card |
| PCIe x4 | x4 | Gen 3 | CPU | M.2 / add-in cards |
| PCIe x1 | x1 | Gen 2 | Chipset | Chipset-attached |

- **No SLI / CrossFireX** support
- **No PCIe bifurcation** in stock BIOS — quad-NVMe adapters require a modded BIOS
- Bifurcation (x8+x8 or x4+x4+x4+x4) may be possible with the patched Huananzhi BIOS but is untested/unsupported

## Rear I/O

| Connector | Quantity |
|---|---|
| PS/2 (keyboard/mouse combo) | 2 |
| USB 2.0 | 6 |
| USB 3.0 | 2 |
| RJ45 (1 GbE) | 1 |
| Audio jacks (5.1 ch) | 3 |

- LAN chip: Realtek RTL8111 / RTL8168
- Audio chip: Realtek ALC897 (5.1 channel)

## Internal Headers / Connectors

| Header | Quantity | Notes |
|---|---|---|
| 24-pin ATX power | 1 | Main power |
| 8-pin EPS CPU power | 1 | CPU power |
| CPU FAN (4-pin PWM) | 1 | Full PWM smart fan control |
| SYS FAN (4-pin) | 1 | Basic voltage control only |
| CASE FAN (3-pin) | 2 | Runs at full 12 V — no speed control |
| SATA data | 6 | — |
| USB 3.0 header | 1 | — |
| USB 2.0 headers | 2 | — |
| Front panel (ATX std) | 1 | PWR SW, RESET, HDD LED, POWER LED |
| TPM 2.0 (14-pin LPC) | **No** | Absent on v1.0; present on some v2.0/PRO |
| POST code display | **No** | Absent on v1.0 |
| Onboard PWR/RESET btn | **No** | Absent on v1.0 |

## Fan Control

- Only **CPU_FAN1** (4-pin) supports full PWM/smart fan control via BIOS
- SYS_FAN (4-pin) has basic voltage-step control
- 3-pin headers are always-on at 12 V
- `lm-sensors` / `fancontrol` will typically report no fan speed data due to broken sensor support
- Workaround: use a PWM fan controller card or external fan controller

## BIOS

- Vendor: AMI UEFI (minimal feature set)
- Both Legacy (CSM) and UEFI boot modes supported
- **This system uses the patched Huananzhi X99-8M-F BIOS** — strongly recommended over stock for stability

### Key BIOS Settings

| Setting | Recommendation |
|---|---|
| CPU C-States → C6 | **Disable** — can cause random instability / boot failures |
| Load Optimized Defaults | Always run after any BIOS flash (F6 or equivalent) |
| Memory XMP/OC | Not supported; runs at JEDEC defaults |

### Patched Huananzhi BIOS

This system runs a **community-patched BIOS based on the Huananzhi X99-8M-F firmware**. This is the recommended approach for MRA9 v1.0 boards due to superior stability and feature support compared to stock.

**Advantages over stock BIOS:**
- Turbo Boost Unlock (full CPU turbo multipliers)
- Undervolting support
- Resizable BAR (ReBAR) for modern GPUs
- Improved S3/S4 sleep state reliability
- Better memory timing/compatibility

**Flashing procedure:**
1. Flash with **Intel FPT** (Flash Programming Tool) from a Windows or DOS bootable USB, or use Miyconst's **Mi899** GUI utility (Windows)
2. **Always back up current BIOS before flashing**
3. Keep a **CH341a programmer** on hand for recovery if the board won't POST after a failed flash
4. After flashing: always load BIOS defaults (F6 → "Load Optimized Defaults")

**BIOS sources:**
- Miyconst Mi899 utility: https://github.com/miyconst/Mi899
- BIOS ROM collection: https://archive.org/download/machinist-e5-mr9a-pro-bios
- MR9A BIOS GitHub archive: https://github.com/0x8008/mr9a

### Alternative: iEngineer Custom BIOS

An independent custom BIOS by **iEngineer** is available as an alternative to the Huananzhi-patched firmware. Miyconst covers it in the MR9A review video (chapter 04:32). See also: https://www.youtube.com/watch?v=RI5bpvRTA_E (for MR9A PRO MAX variant). Offers similar stability/OC unlocks; use the same flashing procedure and CH341a recovery precaution.

## Virtualization & IOMMU

- VT-x: Supported (enable in BIOS)
- VT-d (IOMMU): Supported on Xeon E5 CPUs — enable in BIOS
- IOMMU group isolation is **not as clean** as server boards (desktop chipset)
- GPU passthrough (Proxmox/KVM) is possible but may require `iommu=pt` kernel parameter
- Some users report poor device isolation in IOMMU groups — test before relying on SR-IOV
- ESXi and Proxmox home labs are community-validated use cases

## Known Issues (v1.0 Specific)

| Issue | Severity | Notes |
|---|---|---|
| **CPU fan (4-pin) required to boot** | **Critical** | Board will NOT initialize BIOS/CPU without a 4-pin fan on CPU_FAN1 — hard firmware requirement |
| Temperature sensor reads 120 °C or random values | High | Hardware defect in v1.0; do not use for alerts/automation |
| USB 3.0 instability under heavy load | High | Can cause system lockups; improved in v2.0 |
| No ECC error correction despite ECC DIMMs | Medium | Expected — desktop chipset limitation |
| No TPM 2.0 header | Medium | Windows 11 TPM requirement cannot be met natively |
| No POST debug display | Low | Must use speaker beep codes for boot failures |
| Stock BIOS lacks C-state controls | Medium | Patched Huananzhi BIOS exposes and allows disabling C6 |
| Fan speed reporting broken | Low | Workaround: external PWM controller |
| S3/S4 sleep unreliable on stock BIOS | Medium | Patched Huananzhi BIOS resolves in most cases |

## v1.0 vs v2.0 / PRO Differences

| Feature | v1.0 | v2.0 / PRO |
|---|---|---|
| TPM 2.0 header | ✗ | ✓ |
| POST code display | ✗ | ✓ |
| Onboard power/reset buttons | ✗ | ✓ |
| Integrated I/O shield | ✗ | ✓ |
| Temperature sensor accuracy | Faulty | Improved |
| USB 3.0 reliability | Problematic | Improved |
| Smart fan headers | Limited | Slightly expanded |
| VRM heatsink | Minimal | Better heatsink, improved cooling |
| PCB layers | Fewer | 10-layer (better signal integrity) |

> **Note:** Features vary by production batch even within the same "revision". Verify before purchase.

## Linux Notes

- Boots and runs all major Linux distributions without issues
- `lm-sensors` will report no usable temperature or fan data (broken hardware sensors)
- `edac-util` will show no ECC support (expected)
- USB 3.0 issues may manifest as `xhci_hcd` errors in `dmesg` — avoid heavy USB 3.0 use on v1.0
- Sleep (S3): unreliable on stock BIOS; test after patched Huananzhi BIOS flash
- No special kernel parameters required beyond VT-d if using virtualization:
  `intel_iommu=on iommu=pt`

## No-POST Troubleshooting

v1.0 has **no POST code display** — rely on speaker beep codes and the following checklist.

### Quick Checklist

1. **CPU fan (4-pin)** — **A 4-pin CPU fan must be connected to the CPU_FAN1 header.** The board will NOT start the BIOS, initialize the CPU, or boot without it — this is a hard firmware requirement, not just a warning.
2. **RAM slot** — Single DIMM must be in **DDR4_A1** (closest to CPU). Wrong slot = silent no-POST.
3. **RAM reseating** — Remove, clean contacts, firmly re-insert until both clips lock.
4. **8-pin CPU power** — Must be fully seated. Missing = no POST.
5. **Minimal config** — Remove all cards/drives. Boot with only CPU + 1× RAM + CPU power + 24-pin + CPU fan.
6. **CMOS clear** — Short the JCMOS1 jumper (3-pin near CMOS battery) for 10 s with power off; or remove battery for 5 min.
7. **GPU slot** — Primary display output is from **PCIe x16 #1 (top slot)**. BIOS will NOT output video from the lower x16 slot. Move GPU to top slot.
8. **CPU socket pins** — Board was stored? Inspect LGA 2011-3 socket under good light for bent pins.
9. **Speaker** — Attach a PC speaker to the front-panel header to hear beep codes (v1.0 has no other debug output).
10. **RAM compatibility** — ECC RDIMMs may or may not POST depending on CPU microcode. Try non-ECC unbuffered if in doubt.
11. **Breadboard** — Test outside the case to rule out case shorts.

### RAM Slot Population Rules

| Config | Slots to use |
|---|---|
| 1 DIMM | A1 |
| 2 DIMM (dual-channel) | A1 + B1 |
| 4 DIMM (quad-channel) | A1 + A2 + B1 + B2 |

Populating **any other combination** may result in no POST or reduced stability.

**Physical slot order (left to right, starting closest to CPU socket):**  
`DDR4_A1` → `DDR4_A2` → `DDR4_B1` → `DDR4_B2`  
Slot labels are silk-screened on the PCB. **A1 is always the slot nearest the LGA 2011-3 socket.**

## Community Resources

| Resource | URL |
|---|---|
| Miyconst YouTube — "MR9A (Pro): my favorite LGA 2011-3 board" (PCIe routing, defects, iEngineer BIOS, i7-5820K OC) | https://www.youtube.com/watch?v=kEn30kEkQes |
| Miyconst YouTube — overclocking guide | https://www.youtube.com/watch?v=cC93MGua-ow |
| iEngineer BIOS for MR9A PRO MAX (YouTube, alternative custom BIOS) | https://www.youtube.com/watch?v=RI5bpvRTA_E |
| Mi899 BIOS utility (GitHub, Miyconst) | https://github.com/miyconst/Mi899 |
| GitHub — BIOS archives, quirks, ROM dumps | https://github.com/0x8008/mr9a |
| Archive.org — modded BIOS ROM collection | https://archive.org/download/machinist-e5-mr9a-pro-bios |
| OldRigRevive full review | https://oldrigrevive.com/lga-2011-3/motherboards/machinist-x99-mr9a-pro/ |
| Win-Raid BIOS mod guide | https://winraid.level1techs.com/t/guide-overclock-bios-mods-for-chinese-x99-mbs/104683 |
| Linus Tech Tips community thread | https://linustechtips.com/topic/1480885-thread-for-those-of-you-with-chinese-x99x79x58etc-boards/ |
| MR9A PRO manual PDF (Amazon CDN) | https://m.media-amazon.com/images/I/B19Zb42p7uL.pdf |
| MR9A PRO MAX manual PDF (Amazon CDN) | https://m.media-amazon.com/images/I/B1NgOunvmGL.pdf |
| Machinist official product page (MR9A) | https://machinist.site/mr9a |
