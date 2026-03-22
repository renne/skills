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
| GPU (display) | NVIDIA GeForce GT 710 (GK208B rev a1, ZOTAC 19da:5360, 1 GB DDR3) — PCIe x1 slot (PCH-attached), PCIe addr `08:00.0` |
| GPU (compute) | 2× NVIDIA Tesla V100 SXM2 16 GB — installed in TNS-2SXM2-4P54 carrier, CPU-attached PCIe x16 slots (addrs `03:00.0`, `04:00.0`) |
| NIC | Realtek RTL8111/8168/8211/8411 GbE (`enp6s0`, PCIe `06:00.0`) |
| Internal storage | Ortial ON-750-256 NVMe SSD (256 GB, SM2263EN) — M.2 slot, PCIe addr `02:00.0` |
| OS | Ubuntu 24.04.4 LTS (installed on NVMe; kernel 6.17.0-19-generic) |

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
| M.2 #1 (NVMe) | 1 | PCIe 3.0 x4, up to 32 Gbps | 2280 form factor; CPU lanes |
| M.2 #2 (NVMe) | 1 | PCIe 3.0 x4, up to 32 Gbps | 2280 form factor; as advertised — both slots are NVMe |

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
- **Live ROM identity:** Version `X99MA011`, date `2020-10-15` (original Machinist AMI BIOS — no Miyconst/Huananzhi markers in firmware code; DMI strings `HUANANZHI X99-8M-F V1.1` come from NVRAM variables, not firmware)
- The HUANANZHI DMI strings are stored in NVRAM (not the flash code). The `bios-roms/LIVE-*.rom` is the actual live firmware dump. See [BIOS ROM Files](#bios-rom-files-0x8008mr9a-github-archive) for all variants.

### Key BIOS Settings

| Setting | Menu Path | Recommendation |
|---|---|---|
| Load Optimized Defaults | F6 key on main BIOS screen | **Required** after any BIOS flash |
| CPU C-State → C6 | Chipset → Processor Configuration → CPU C State Control / Package C State limit | **Disable C6** — causes random instability / boot failures on this board |
| Above 4G Decoding | Advanced → PCI Subsystem Settings → Above 4G Decoding | **Enable** when using GPUs with large BARs (Tesla V100, RX 5700+, RTX series) |
| CSM Support | Advanced → CSM Configuration → CSM Support | **Disable** for UEFI-only GPU / clean UEFI boot; requires UEFI GOP VBIOS on all GPUs |
| Video Option ROM | Advanced → CSM Configuration → Option ROM Execution → Video | **UEFI** (with CSM off or UEFI GOP GPU); **Legacy** only if GPU has no UEFI GOP |
| Intel VT-d (IOMMU) | Chipset → IIO Configuration → Intel VT for Directed I/O (VT-d) | **Enable** for GPU passthrough / IOMMU |
| Hyper-Threading | Chipset → Processor Configuration → Hyper-Threading [ALL] | Enable (default) for workloads; disable for latency-sensitive single-threaded use |
| Turbo Mode | Chipset → Processor Configuration → Turbo Mode | **Enable** (default); required for TBU to have effect |
| SR-IOV Support | Advanced → PCI Subsystem Settings → PCI Express Settings → SR-IOV Support | Enable if using NIC SR-IOV or GPU virtual functions |
| Secure Boot | Security → Secure Boot | Disable for Linux/Proxmox (unsigned kernels) |
| Memory XMP/OC | N/A | **Not supported**; runs at JEDEC defaults only |

### Patched Huananzhi BIOS (community-recommended alternative)

The live SPI dump from this machine (`LIVE-machinist-x99-mr9a-20201015-X99MA011.rom`) is the **original Machinist AMI BIOS** (version X99MA011, 2020-10-15) — not a Huananzhi cross-flash. The `HUANANZHI X99-8M-F V1.1` string visible in DMI/lshw comes from NVRAM variables, not the firmware code.

However, the community-patched Huananzhi X99-8M-F firmware from the [0x8008/mr9a](https://github.com/0x8008/mr9a) archive is the **recommended upgrade** for MRA9 v1.0 boards due to superior stability and feature support compared to stock.

The default stock BIOS lacks sleep state support, memory timing control, Turbo Boost, and overclocking. Flashing the Huananzhi X99-8M-F firmware resolves all of these. The board is sold as both "X99-MR9A" and "E5-MR9A" — they are (nearly) identical; all BIOS files work on both.

**Advantages over stock BIOS:**
- Turbo Boost Unlock (full CPU turbo multipliers)
- Undervolting support (e.g., −50 mV)
- Resizable BAR (ReBAR) support via [xCuri0/ReBarUEFI](https://github.com/xCuri0/ReBarUEFI)
- Improved S3/S4 sleep state reliability
- Better memory timing/compatibility

### BIOS ROM Files (0x8008/mr9a GitHub archive)

Repository: https://github.com/0x8008/mr9a

These are confirmed-working BIOS dumps from an MR9A board. Variants:

ROM files are stored locally in `bios-roms/` (this skill directory).

| File | Description | MD5 | Direct URL |
|---|---|---|---|
| `STOCK-machinist-x99-mr9a-20240226-203747.rom` | Original stock BIOS dump — keep as recovery backup | `03d7859f2ac7958f458acafa2ff78aaa` | [download](https://raw.githubusercontent.com/0x8008/mr9a/main/STOCK-machinist-x99-mr9a-20240226-203747.rom) |
| `TBU-machinist-x99-mr9a-20240227-102043.rom` | **Recommended** — Huananzhi X99-8M-F + Turbo Boost Unlock + undervolting (−50 mV) via Mi899 | `9893df2d90b57e163881272a7730685d` | [download](https://raw.githubusercontent.com/0x8008/mr9a/main/TBU-machinist-x99-mr9a-20240227-102043.rom) |
| `REBAR-machinist-x99-mr9a-20240227-102043.rom` | TBU + undervolting + **Resizable BAR** (ReBarUEFI) | `022050ad32de1a00b448dd50fab6b210` | [download](https://raw.githubusercontent.com/0x8008/mr9a/main/REBAR-machinist-x99-mr9a-20240227-102043.rom) |
| `LOGO-machinist-x99-mr9a-20240227-102043.rom` | Same as REBAR + custom boot logo (no "Huananzhi" splash) | `8dd01baf00de2a0c7c265de67893c50e` | [download](https://raw.githubusercontent.com/0x8008/mr9a/main/LOGO-machinist-x99-mr9a-20240227-102043.rom) |
| `LIVE-machinist-x99-mr9a-20201015-X99MA011.rom` | **Live SPI dump from this machine** — version `X99MA011`, date `2020-10-15`; no Miyconst markers — original Machinist AMI BIOS, distinct from all four archived variants above | MD5: `b8adb1746988b3c0849ebad7926982e2` SHA1: `aa415e4746a5575b34949fad2eac3120f52ae8e1` | (local only — dumped via `flashrom -p internal`) |

All files are 16,777,216 bytes (16 MiB — matches the Winbond 25Q128FV SPI flash chip).

> **Warning (from 0x8008):** These BIOSes are only confirmed to work on the author's board. They will most likely work on yours too, but proceed at your own risk.

### Mi899 Flashing Tool (Windows GUI)

Mi899 is a Windows GUI for managing, patching, and flashing BIOSes for Machinist/Huananzhi boards. It bundles Intel FPT internally.

- Project page: https://github.com/miyconst/Mi899
- **Latest release (v1.4.6):** https://github.com/miyconst/Mi899/releases/tag/1.4.6
- **Direct download:** https://github.com/miyconst/Mi899/releases/download/1.4.6/Mi899-1.4.6.zip
- Additional BIOS ROM collection: https://archive.org/download/machinist-e5-mr9a-pro-bios

### Flashing Procedure

1. **Back up current BIOS first** — use Mi899 → "Read BIOS" or Intel FPT: `fpt.exe -d backup.rom`
2. Keep a **CH341a SPI programmer** on hand for recovery (hardware reflash if board won't POST)
3. Boot into Windows, run Mi899 or use `fpt.exe -f <rom_file>` from a command prompt
4. **BIOS Lock workaround (if FPT fails with "Error 26" or access denied):**
   - Reboot to BIOS → IntelRCSetup → PCH Configuration → Security Configuration → **BIOS Lock**
   - Toggle it to **Disable** (even if it already appears Disabled) → Save & reboot to Windows → retry FPT
5. After flashing: reboot, press **F6 → "Load Optimized Defaults"** — mandatory after any BIOS change
6. After loading defaults: disable **CPU C6 states** (C-states configuration → C6 = **Disabled**) to prevent random boot failures
7. Optionally set **Above 4G Decoding** if you have GPUs with large BARs (e.g., Tesla V100)

### Alternative: iEngineer Custom BIOS

An independent custom BIOS by **iEngineer** is available as an alternative to the Huananzhi-patched firmware. Miyconst covers it in the MR9A review video (chapter 04:32).

- Video review (MR9A PRO MAX variant, same flashing process): https://www.youtube.com/watch?v=RI5bpvRTA_E
- Offers similar TBU/OC unlocks; use the same Mi899 + FPT flashing procedure and CH341a recovery precaution.

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
| **Xid 74 NVLink fatal error + full system freeze** | **Critical** | Triggered by SIGKILL to V100 CUDA processes mid-NVLink DMA. Recovery: hard power cycle only. See V100 section above. |

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
| Model | NVIDIA Tesla V100 SXM2 16 GB |
| Quantity | 2 — installed via TNS-2SXM2-4P54 carrier + 2× RTE162P54B-2UR risers |
| PCIe slots | CPU-attached x16 slots (PCIe Gen 3, 16× lanes from CPU) |
| PCIe addresses | GPU0 `03:00.0`, GPU1 `04:00.0` |
| IOMMU groups | GPU0 → group 29, GPU1 → group 30 (each isolated — ideal for passthrough) |
| VFIO PCI IDs | GPU function `10de:1db1`, Audio function `10de:1db3` |
| BAR size | 64-bit prefetchable BAR up to 16 GB — **requires Above 4G Decoding** |
| Purpose | CUDA compute only (no display output) |
| Serial numbers | GPU0: `1321920049451`, GPU1: `1321620017216` |

> **V100 and "PCI Out of Resources":** Both Tesla V100s request large 64-bit prefetchable memory BARs. Without "Above 4G Decoding" enabled in BIOS, the firmware cannot map these into the 32-bit MMIO window (0–4 GB) and logs "PCI Out of Resources" errors. The BARs are left unmapped and the GPUs are unusable for CUDA. **Additionally, the NVMe SSD (`02:00.0`) also becomes invisible to the OS when Above 4G Decoding is disabled on this board — always keep it enabled.**

### Validated test results (mra9, 2026-03-22)

All tests run with **driver 580.126.09**, **CUDA toolkit 12.0**, **NCCL 2.29.7**.

| Test | Result | Details |
|---|---|---|
| FP64 burn (gpu-burn, 300 s) | ✅ PASSED | Peak 83/82 °C, 0 errors, no throttling |
| VRAM memtest (cuda_memtest, 3 passes) | ✅ PASSED | 0 errors both GPUs; temps 41–45 °C |
| NVLink status | ✅ ACTIVE | All 6 links × 25.781 GB/s per GPU |
| NCCL all_reduce bandwidth | ⏳ PENDING | Blocked — nccl-tests rebuild needs nvcc ≥ 12.2 |

### Required driver setup (working configuration)

```bash
# 1. Blacklist nouveau in initramfs (MANDATORY before driver install)
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u && sudo reboot

# 2. Install server driver
sudo apt install -y nvidia-driver-580-server

# 3. Add NVIDIA CUDA apt repo + install NCCL
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb && sudo apt update
sudo apt install -y libnccl2=2.29.7-1+cuda12.9 libnccl-dev=2.29.7-1+cuda12.9

# 4. Kernel parameters (already in /etc/default/grub on mra9)
# GRUB_CMDLINE_LINUX_DEFAULT="quiet splash iommu=pt pcie_aspm=off"
```

### ⚠️ GT 710 + GDM blocks NVIDIA driver unload

GDM/X11 holds `nvidia_drm` via the GT 710 display GPU, preventing `rmmod` of the entire NVIDIA driver stack. If you need to unload the driver (e.g., for VFIO rebinding):

```bash
sudo systemctl stop gdm   # must do this FIRST — releases nvidia_drm
sudo rmmod nvidia_drm nvidia_modeset nvidia_uvm nvidia
```

Skipping `systemctl stop gdm` causes `rmmod nvidia_drm` to fail with "Module is in use".

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

**nvflash binary is stored in this skill repository:**  
`tools/nvflash_5.867_linux.zip` — MD5: `dc4775fdefa3d4cbf2b5ab9178a4720e`  
Contains: `x64/nvflash` (Linux x86_64, 15.2 MB), `x86/`, `aarch64/`, `ppc64/` variants.  
Source: TechPowerUp, dated 2024-12-10.

**Flash procedure (on mra9 via SSH, Ubuntu live environment):**
```bash
# Copy files from local machine first
# nvflash is stored in the skill repo: ~/.copilot/skills/hardware/machinist-mra9-v1/tools/nvflash_5.867_linux.zip
scp ~/.copilot/skills/hardware/machinist-mra9-v1/tools/nvflash_5.867_linux.zip ~/Downloads/gt710_uefi.rom mra9:/tmp/

# On mra9:
cd /tmp
unzip nvflash_5.867_linux.zip
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

### BIOS Menu Reference (Live ROM — X99MA011, 2020-10-15)

Full menu tree extracted via IFR opcode parsing from the live SPI dump (`LIVE-machinist-x99-mr9a-20201015-X99MA011.rom`).
Extraction method: Setup DXE FFS (GUID `899407D7-99FE-43D8-9A21-79EC328CAC21`) → LZMA decompression → IFR opcode stream + HII string package (989 strings).

Top-level tabs: **Main → Advanced → Chipset → Boot → Security → Save & Exit**

> **Chipset form note:** The Chipset tab (`IntelRCSetup`, Form 0x2713) is **intentionally empty in the Setup DXE module** — this is standard AMI behavior. The Chipset DXE module registers its menus into the HII database at runtime, so they only appear on the live system. The Chipset submenu structure (Processor Configuration, IIO Configuration, Memory Map, etc.) was confirmed by extracting UCS-2 strings from the Chipset DXE module at ROM offset `0x009070E8`.

#### Main

- BIOS Information (Vendor, Core Version, Compliancy, Board Name, BIOS Version, Build Date/Time)
- Memory Information
- System Language

#### Advanced

- **Trusted Computing** — TPM device support (Security Device Support, TPM State, Pending operation; TPM 1.2/2.0/TCM selectable)
- **ACPI Settings** — Enable ACPI Auto Configuration, Enable Hibernation, ACPI Sleep State (S1/S3), Lock Legacy Resources
- **NCT5532D SSIO Configuration** — AC Power Loss setting; Serial Port 1 Configuration (enable/disable, I/O address)
- **Hardware Monitor (PC Health Status)** — CPU temps, fan RPM, voltages (display only)
- **Smart Fan Function** — Smart Fan 1 & 2: mode (Manual PWM / Smart), temperature thresholds (4 points), PWM levels, critical temperature and duty, tolerance
- **Serial Port Console Redirection** — COM0 console redirect (Terminal Type, baud, flow control); Legacy Console Redirection; EMS (Windows Emergency Management)
- **PCI Subsystem Settings** (Form 0x2740)
  - PCI Latency Timer, PCI-X Latency Timer, VGA Palette Snoop, PERR#/SERR# Generation
  - **Above 4G Decoding** ← ★ enable for Tesla V100 / large-BAR GPUs
  - SR-IOV Support, BME DMA Mitigation
  - PCI Express Settings → ASPM Support, Relaxed Ordering, Extended Tag, No Snoop, Max Payload/Read Request, Extended Synch, Link Training Retry/Timeout
  - PCI Express GEN 2 Settings → Completion Timeout, ARI Forwarding, AtomicOp, IDO, LTR, Target Link Speed, Clock Power Management, HW Auto Width/Speed
- **Network Stack Configuration** — Network Stack enable, IPv4/IPv6 PXE support, PXE boot wait time, Media detect count, LAN Wake-up Control
- **CSM Configuration** (Form 0x2747)
  - **CSM Support** ← ★ disable for UEFI-only boot (requires UEFI GOP on all GPUs)
  - GateA20 Active, Option ROM Messages, Boot option filter (UEFI+Legacy / Legacy only / UEFI only)
  - Option ROM Execution:
    - Network — Do not launch / UEFI / Legacy
    - Storage — Do not launch / UEFI / Legacy (set UEFI for NVMe)
    - **Video** — Do not launch / UEFI / **Legacy** (change to UEFI after GOP flash; irrelevant when CSM off)
    - Other PCI devices
  - (When CSM disabled: "CSM configuration is disabled — Compatibility Support Module is not loaded due to active UEFI Secure Boot mode.")
- **NVMe Configuration** — NVMe controller and drive information (shows "No NVME Device Found" if none present)
- **USB Configuration** — USB Support, Legacy USB Support, USB 2.0 Controller Mode, XHCI Legacy/Hand-off, EHCI Hand-off, USB Mass Storage Driver, Port 60/64 Emulation, transfer/reset/power-up delay timeouts; per-port device type overrides

#### Chipset (IntelRCSetup — runtime-populated by Chipset DXE)

*The following structure was confirmed from Chipset DXE UCS-2 string extraction; exact sub-options may vary at runtime.*

- **Processor Configuration**
  - Hyper-Threading [ALL]
  - Enhanced Halt State (C1E)
  - Execute Disable Bit (XD/NX)
  - EIST (Enhanced SpeedStep / P-states)
  - **Turbo Mode** — required for TBU multipliers
  - **CPU C State Control / Package C State limit** — set to **C1E or Disabled** (never C6 — causes boot failures)
  - Energy Efficient Turbo, Turbo Power Limit Lock
  - Non-Turbo Mode Processor Core Ratio Multiplier
  - Turbo-XE Mode TDC/TDP Limit Override
- **IIO Configuration** (IIO General Configuration)
  - **Intel VT for Directed I/O (VT-d)** ← ★ enable for IOMMU/GPU passthrough (Proxmox, KVM)
    - Interrupt Remapping
  - IIO PCIe Root Port settings (per-port enable/disable, speed, ASPM)
- **Memory Map** — memory mapping configuration
- **PCH Configuration** — PCH-level settings (USB, SATA, PCH PCIe ports, etc.)

#### Boot

- Boot Configuration: Setup Prompt Timeout, Bootup NumLock State
- **Fast Boot** — SATA Support, VGA Support, USB Support, PS2 Devices Support, Network Stack Driver Support, Redirection Support
- **Quiet Boot** — suppress POST details / show logo
- Built-in EFI Shell option
- Boot Option Priorities (`Boot Option #N`)
- Driver Option Priorities

#### Security

- Administrator Password / User Password
- Secure Boot menu:
  - Vendor Keys, **Secure Boot** (must be **Disabled** for Linux/Proxmox unsigned kernels), Secure Boot Mode
  - Key Management: Provision Factory Default keys, Delete/Enroll/Save all Secure Boot variables (PK, KEK, db, dbx, dbt)

#### Save & Exit

- Save Changes and Exit / Discard Changes and Exit
- Save Changes and Reset / Discard Changes and Reset
- Save Changes / Discard Changes
- **Restore Defaults** — also **F6** key from any screen (mandatory after BIOS flash)
- Save as User Defaults / Restore User Defaults
- Boot Override — one-time boot device selection

> **CSM / OpROM note:** The default "Option ROM Execution → Video = Legacy" runs the GPU's legacy x86 option ROM at POST. After flashing a UEFI GOP VBIOS onto the GT-710 (or any GPU), change this to **UEFI**. When CSM is fully Disabled, this setting has no effect — the GOP EFI driver initializes all displays natively.

> **VT-x note:** On Xeon E5 (Haswell-EP) processors, Intel VT-x (VMX) is always enabled in hardware. There is no separate VT-x toggle in this BIOS — it is permanently active. Only VT-d has an explicit BIOS toggle.

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
