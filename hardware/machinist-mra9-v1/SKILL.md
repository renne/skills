---
name: machinist-mra9-v1
description: Hardware reference for the Machinist X99 MRA9 / E5-MR9A hardware revision v1.0 — an ATX LGA 2011-3 budget motherboard supporting Xeon E5 v3/v4 CPUs with DDR4 quad-channel memory. The `mra9` machine runs with the Huananzhi X99-8M-F BIOS cross-flashed (so DMI reports HUANANZHI branding). Use when managing, troubleshooting, or configuring any system running on this board: covers specs, CPU/RAM compatibility (including RDIMM), BIOS quirks, storage, PCIe layout, fan control, virtualization (IOMMU/VT-d), and known v1.0-specific issues.
---
# Machinist X99 MRA9 / E5-MR9A — Hardware Revision v1.0

Also marketed as: **X99-MR9A**, **E5-MR9A**, **Machinist MRA9**  
Revision v1.0 is the original production run — no debug display, no onboard power/reset buttons, no TPM 2.0 header, no integrated I/O shield.

> **Branding note:** The machine named `mra9` has DMI reporting `Manufacturer: HUANANZHI`, `Product Name: X99-8M-F V1.1`. This is because the **Huananzhi X99-8M-F BIOS has been cross-flashed** onto the Machinist board — the BIOS ROM contains HUANANZHI DMI strings which overwrite the original Machinist entries. The physical board is a Machinist MRA9 v1.0.

## Board Overview

| Parameter | Value |
|---|---|
| Manufacturer | Machinist (Chinese budget OEM) |
| Model | MRA9 / MR9A (hardware rev v1.0) — DMI may report HUANANZHI X99-8M-F V1.1 after BIOS cross-flash |
| Form Factor | ATX (~216–280 × 215–283 mm) |
| Socket | LGA 2011-3 |
| Chipset | Intel **Q87 Express** (confirmed on `mra9`; B85 or C226 on other production batches) |
| BIOS Chip | Winbond 25Q128FV |
| BIOS Type | Legacy + UEFI (AMI) |
| BIOS Version | AMI 5.11 (Huananzhi-patched), released 10/15/2020 |

> **Note:** Despite "X99" marketing, this board uses a **desktop chipset (B85/Q87/C226)**, NOT Intel X99 or C612. This limits ECC, PCIe bifurcation, VROC, IPMI, and overclocking.

## Installed Configuration (`mra9`)

Confirmed hardware as of live system inspection (Ubuntu 24.04.4 LTS live via Ventoy, kernel 6.17.0-14-generic):

| Component | Details |
|---|---|
| CPU | Intel Xeon E5-2620 v3 @ 2.40 GHz (6C/12T, max turbo 3.2 GHz) |
| RAM | 1× 8 GB DDR4 (Corsair CMK8GX4M1A2400C16) in slot **DIMM_B1**, running at **1866 MT/s** (JEDEC default; XMP not supported) |
| GPU (display) | NVIDIA GeForce GT 710 (GK208B rev a1, ZOTAC 19da:5360, 1 GB DDR3) — PCIe x1 slot (PCH-attached), PCIe addr `05:00.0` |
| GPU (compute) | 2× NVIDIA Tesla V100 (SXM2 or PCIe) — currently **removed** pending Above 4G Decoding fix; intended for CPU-attached PCIe x16 slots |
| NIC | Realtek RTL8111/8168/8211/8411 GbE (`enp6s0`, PCIe `06:00.0`) |
| Internal storage | **None installed** — system currently boots via Ventoy USB (SanDisk 57.3 GB, `/dev/sda`) |
| OS | Ubuntu 24.04.4 LTS (live environment from Ventoy) |

> **RAM slot note:** On `mra9` (Machinist board with Huananzhi BIOS cross-flashed), SMBIOS reports the populated single-DIMM slot as **DIMM_B1** (Node 1). The physical silkscreen on the Machinist board labels this slot **DIMM1**. The Huananzhi BIOS relabels it B1 in its SMBIOS tables. Single-DIMM in the physical DIMM1 slot works correctly.

## CPU Support

| Parameter | Value |
|---|---|
| Architecture | LGA 2011-3 (Haswell-EP / Broadwell-EP) |
| Supported CPUs | Intel Xeon E5-26xx v3, E5-26xx v4; Core i7-58x0X / i7-68x0X |
| Recommended CPUs | E5-2620 v3/v4, E5-2650 v3/v4, E5-2680 v3/v4, E5-2699 v3/v4 |
| Installed (`mra9`) | **Intel Xeon E5-2620 v3** — 6C/12T (HT), 2.40 GHz base, 3.20 GHz turbo, 85 W TDP, 15 MiB L3 |
| TDP headroom | 4-phase VRM (8 virtual via doublers); active VRM cooling recommended for >120 W CPUs |
| Overclocking | None — desktop chipset does not support multiplier adjustment |
| Turbo unlock | Available via patched Huananzhi BIOS (see BIOS section) |
| Temperature sensor | **Unreliable on v1.0** — commonly reads 120 °C or random values; do not trust |

### Security Vulnerabilities (E5-2620 v3)

| Vulnerability | Status |
|---|---|
| Meltdown | Mitigated (PTI) |
| Spectre v1/v2 | Mitigated |
| MDS (Microarchitectural Data Sampling) | **Vulnerable — no microcode fix available for this stepping** |
| MMIO Stale Data | **Vulnerable — no microcode fix available** |

> No remediation exists at OS level for MDS on this CPU. Do not run untrusted workloads in shared VMs without physical isolation.

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
| Speed | DDR4-2133 / 2400 (JEDEC); actual running speed may be **1866 MT/s** (see note) |
| ECC support | ECC modules POST and boot; **ECC error correction is NOT functionally enabled** (desktop chipset limitation) |
| Non-ECC | Supported |
| RDIMM | **Yes** — the E5-2620 v3's integrated memory controller (IMC) natively supports RDIMM. The Q87 PCH is irrelevant to memory type on LGA 2011-3; all memory signaling goes through the CPU IMC, not the chipset. Community-validated on this board. |
| LRDIMM | Compatibility varies by module; some work, some fail to POST. |

> **RDIMM & Chipset clarification:** On LGA 2011-3, the Q87 PCH does **not** control memory. The Intel E5 v3 CPU has a fully integrated DDR4 memory controller supporting UDIMM, RDIMM, and LRDIMM. The Q87 in this system handles chipset I/O (SATA, USB, PCIe from PCH lanes) — memory type support depends solely on the CPU IMC and the board's slot wiring, both of which accept registered DIMMs.

> **ECC Caveat:** Linux `dmesg` / `edac-util` will report no ECC support. This is expected — the hardware accepts ECC/RDIMM modules electrically but the Q87 desktop PCH does not enable error correction logic.

> **Speed note:** On the `mra9` machine, a DDR4-2400 module (Corsair CMK8GX4M1A2400C16) runs at **1866 MT/s** — the board enforces JEDEC sub-profiles and does not support XMP. This is normal and expected.

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
- **Confirmed on `mra9`:** AMI version **5.11**, release date **10/15/2020**

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
2. **RAM slot** — Single DIMM must be in **DIMM1** (closest to CPU). Wrong slot = silent no-POST.
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
| 1 DIMM | **DIMM1** (Machinist silkscreen) / SMBIOS reports as `DIMM_B1` when Huananzhi BIOS is flashed |
| 2 DIMM (dual-channel) | DIMM1 + DIMM2 |
| 4 DIMM (quad-channel) | DIMM1 + DIMM2 + DIMM3 + DIMM4 |

Populating **any other combination** may result in no POST or reduced stability.

**Physical slot order (left to right, starting closest to CPU socket):**  
`DIMM1` → `DIMM2` → `DIMM3` → `DIMM4`  
Slot labels are silk-screened on the PCB. **DIMM1 is always the slot nearest the LGA 2011-3 socket.**

> **Huananzhi BIOS cross-flash note:** When the Huananzhi X99-8M-F BIOS is flashed, SMBIOS reports the slots as `DIMM_A1/A2/B1/B2` — `DIMM1` maps to `DIMM_B1`. If no-POST occurs with single DIMM in DIMM1, verify it is seated correctly; no need to move to another slot.

## GPU Configuration

### Display GPU: NVIDIA GeForce GT 710 (ZOTAC)

| Parameter | Value |
|---|---|
| Model | NVIDIA GeForce GT 710 |
| GPU die | GK208B (Kepler, 28 nm) |
| Device ID | `10de:128b` rev a1 |
| Subsystem | ZOTAC `19da:5360` |
| VRAM | 1 GB DDR3 |
| PCIe slot | x1 slot (PCH-attached), PCIe address `05:00.0` |
| VBIOS version | v80.28.A6.00.10, 59,904 bytes |
| VBIOS type | **Legacy-only** (PCIR code_type=0, x86 real mode, no EFI/GOP partition) |
| VBIOS backup | `~/Downloads/gt710_current.rom` (local machine, persistent) |
| Driver | `nouveau` (open source) or proprietary NVIDIA 470.xx branch |

> **VBIOS note:** The ZOTAC GT-710 1 GB DDR3 ships with a **legacy-only VBIOS** — no UEFI GOP module. This causes a black screen during BIOS POST when "Above 4G Decoding" is enabled (see section below). The OS will initialize the GPU correctly via driver regardless.

### Compute GPUs: NVIDIA Tesla V100 (×2)

| Parameter | Value |
|---|---|
| Model | NVIDIA Tesla V100 |
| Quantity | 2 (currently **removed** — pending Above 4G Decoding fix) |
| PCIe slots | CPU-attached x16 slots (PCIe Gen 3, 16× lanes from CPU) |
| BAR size | 64-bit prefetchable BAR up to 16 GB — **requires Above 4G Decoding** |
| Purpose | CUDA compute only (no display output) |

> **V100 and "PCI Out of Resources":** Both Tesla V100s request large 64-bit prefetchable memory BARs. Without "Above 4G Decoding" enabled in BIOS, the firmware cannot map these into the 32-bit MMIO window (0–4 GB) and logs "PCI Out of Resources" errors. The BARs are left unmapped and the GPUs are unusable for CUDA.

## Above 4G Decoding — Root Cause, Diagnosis, and Fix

### Problem

Enabling "Above 4G Decoding" in BIOS causes the GT-710 to output a **black screen from the first POST** with zero video output. This is **not** a hang — the system still boots; it is a POST-display-only issue.

### Root Cause (Confirmed)

When Above 4G Decoding is active, the BIOS MMIO resource allocator remaps the GT-710's 64-bit prefetchable BARs (Region 1: 128 MB framebuffer, Region 3: 32 MB) to addresses **above 4 GB**. The GT-710's legacy VBIOS runs in **16-bit real mode** during POST initialization and cannot address MMIO above 4 GB. The framebuffer becomes unreachable → black screen during POST.

This affects **BIOS/POST display only**. The Linux OS (nouveau or NVIDIA proprietary driver) runs in 64-bit protected mode and can address any BAR location. **Display returns to normal once the OS boots and loads the driver.**

### Confirmed non-causes (all ruled out via BIOS inspection)

| BIOS Setting | Path | Confirmed value | Effect |
|---|---|---|---|
| Active Video | IntelRCSetup → Miscellaneous Configuration → Active Video | Offboard | Correct — no change needed |
| Option ROM Execution → Video | CSM section | Legacy | Already Legacy — not the cause |
| PCH Display | PCH section | Enabled | Correct |
| Legacy VGA socket | IIO section | Socket 0 | Correct |
| IIO ID0 Gen setting | PCIe generation config | Gen2 (x1 PCH slot) | Irrelevant to display |

### Fix Option 1: Flash UEFI GOP VBIOS (Recommended for BIOS display)

Replace the legacy-only VBIOS with a UEFI GOP-enabled VBIOS so the card can initialize in 64-bit protected mode during BIOS POST.

**Target UEFI VBIOS:** TechPowerUp entry #190890  
URL: `https://www.techpowerup.com/vgabios/190890/190890`  
(ZOTAC GT-710 1024MB DDR3, UEFI=Yes, v80.28.A6.00.10)

**Flash tool required:** nvflash 5.867 for Linux x64  
Download: `https://www.techpowerup.com/download/nvidia-nvflash/`  
(Must be downloaded manually via browser — TechPowerUp uses Cloudflare JS challenge that blocks all programmatic access.)

**Flash procedure (on mra9 via SSH, Ubuntu live environment):**
```bash
# Copy files from local machine first
scp ~/Downloads/nvflash_5.867_Linux_x64.zip ~/Downloads/gt710_uefi.rom mra9:/tmp/

# On mra9:
cd /tmp
unzip nvflash_5.867_Linux_x64.zip
chmod +x x64/nvflash

# Verify tool sees the GPU
sudo x64/nvflash --list

# Flash (--protectoff removes write protection; -6 overrides subsystem ID mismatch if needed)
sudo rmmod nouveau
sudo x64/nvflash --protectoff
sudo x64/nvflash -6 /tmp/gt710_uefi.rom

# Reboot, then re-enable Above 4G Decoding in BIOS
sudo reboot
```

> **Recovery:** If flash fails or display is lost, restore original VBIOS:
> `sudo x64/nvflash -6 /tmp/gt710_current.rom`
> Backup file: `~/Downloads/gt710_current.rom` (local, persistent — keep this safe)

### Fix Option 2: Accept black BIOS screen (Practical workaround)

Since the OS brings up display normally, you can operate with Above 4G Decoding enabled **without** flashing the VBIOS:
- BIOS POST and BIOS setup screens: no display (navigate blind using memorized layout)
- OS boot and runtime: display works normally via GPU driver
- SSH management: always available regardless of display state
- V100 CUDA compute: fully functional with Above 4G Decoding enabled

**To test if this approach works:**
1. Enable Above 4G Decoding in BIOS (navigate blind — you know the layout)
2. Wait ~3 minutes for Ubuntu live to boot
3. Run `ssh mra9` — if it connects, the system is fully functional
4. Install V100s, verify no "PCI Out of Resources" errors: `dmesg | grep -i 'pci\|bar\|resource'`

### BIOS Navigation (Huananzhi X99-8M-F, Machinist MRA9 v1.0)

These menus are confirmed to exist in the Huananzhi-patched BIOS on this board:

| Path | Location |
|---|---|
| Above 4G Decoding | Main BIOS setup → Advanced → PCIe/PCI settings (or similar top-level PCIe menu) |
| Active Video (offboard/onboard) | IntelRCSetup → Miscellaneous Configuration → Active Video |
| Option ROM Execution → Video | BIOS → Advanced → CSM Configuration → Option ROM Execution → Video |
| PCH Display | PCH settings section |
| Legacy VGA socket | IntelRCSetup → IIO Configuration → Legacy VGA Socket |
| Load Optimized Defaults | F6 key from main BIOS screen (essential after any BIOS flash) |

> **CSM / OpROM note:** "Option ROM Execution → Video = Legacy" is the **default** setting in this BIOS. It means the BIOS will try to run the GPU's legacy x86 option ROM during POST. This is correct for legacy VBIOSes. After flashing UEFI GOP VBIOS, change this to "UEFI" (or "Do Not Execute" if the GPU initializes via GOP without being called as legacy ROM).

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
