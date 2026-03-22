---
name: NVIDIA GeForce GT 710 (GK208B)
description: NVIDIA GeForce GT 710 — GK208B GPU, PCIe 2.0 x8/x16, 1–2 GB DDR3/GDDR5 — VBIOS variants (legacy vs UEFI GOP), Above 4G Decoding black screen issue, nvflash procedure
---

# NVIDIA GeForce GT 710 (GK208B)

## Overview

| Field | Value |
|-------|-------|
| Model | GeForce GT 710 |
| GPU | GK208B (Kepler architecture, 28 nm) |
| PCI Vendor ID | `0x10DE` (NVIDIA) |
| PCI Device ID | `0x128B` |
| Interface | PCIe 2.0 x1/x8/x16 (x1 is sufficient; x16 slot compatible) |
| VRAM | 1 GB or 2 GB DDR3 or GDDR5 (varies by SKU) |
| TDP | ~19 W (passively cooled on most SKUs) |
| PCIe power | **None required** — powered entirely from the PCIe slot |
| Display outputs | Varies by AIB partner (common: HDMI / DVI / VGA) |
| Driver support | NVIDIA legacy driver branch 470.xx (Linux); 474.x (Windows) |

## Known SKUs / Board Variants

| AIB | Subsystem | VRAM | VBIOS type | Notes |
|-----|-----------|------|------------|-------|
| ZOTAC | `19da:5360` | 1 GB DDR3 | **Legacy-only** (see below) | Ships without UEFI GOP; causes POST black screen with Above 4G Decoding enabled |
| Various | — | 2 GB DDR3/GDDR5 | Varies | Check GPU-Z "UEFI" field to confirm |

## VBIOS Variants

### ⚠️ Legacy-Only VBIOS (ships on some GT 710 cards, e.g. ZOTAC 19da:5360)

| Field | Value |
|-------|-------|
| VBIOS type | Legacy-only — PCIR `code_type=0`, x86 real-mode only, **no UEFI EFI/GOP partition** |
| Version | v80.28.A6.00.10 |
| Size | **59,904 bytes** (≈ 58.5 KB) |
| BIOS POST behavior | Works fine without Above 4G Decoding. **Black screen** during POST when Above 4G Decoding is enabled (see below). |
| OS behavior | GPU initializes normally via nvidia / nouveau driver regardless — OS is unaffected |

Verify with:
```bash
strings <rom>.rom | grep -iE "uefi|efi|gop"
# No output = legacy-only
```

---

### ✅ UEFI GOP VBIOS — `vbios/190890.rom`

Replacement VBIOS with UEFI GOP module, enabling correct operation with Above 4G Decoding enabled.

| Field | Value |
|-------|-------|
| Source | [TechPowerUp VBIOS Database #190890](https://www.techpowerup.com/vgabios/190890/190890) |
| Version | 80.28.A6.00.10 |
| VBIOS type | **UEFI GOP enabled** |
| Board string | `GK208 P2132 SKU 14 VGA BIOS` |
| Board detail | `GK208 Board - 21320014` |
| PCI Vendor ID | `0x10DE` (NVIDIA Corp.) |
| PCI Device ID | `0x128B` (GeForce GT 710) |
| Board SKU | P2132 SKU 14 |
| Copyright | 1996–2015 NVIDIA Corp. |
| File size | **169,472 bytes** (166 KB — much larger than legacy ROM due to EFI module) |
| MD5 | `4565d08a37b448d6c869e7b834dc04a4` |

Verify ROM before flashing:
```bash
python3 -c "
import struct
data = open('vbios/190890.rom','rb').read()
idx = data.find(b'PCIR')
vendor = struct.unpack_from('<H', data, idx+4)[0]
device = struct.unpack_from('<H', data, idx+6)[0]
print(f'VendorID=0x{vendor:04x} DeviceID=0x{device:04x}')
# Expect: VendorID=0x10de DeviceID=0x128b
"

lspci -nn | grep -i nvidia
# Expect: [10de:128b]
```

## Above 4G Decoding — Black Screen Issue

### Problem

On cards with **legacy-only VBIOS**, enabling "Above 4G Decoding" in motherboard BIOS causes a **black screen from the first POST frame**. The system is **not frozen** — it continues to boot normally — but there is no video output until the OS loads its GPU driver.

### Root Cause (Confirmed on Machinist MRA9 v1.0 / Huananzhi X99-8M-F)

When Above 4G Decoding is active, the BIOS MMIO allocator remaps the GT-710's 64-bit prefetchable BARs (Region 1: 128 MB framebuffer, Region 3: 32 MB) to addresses **above 4 GB**. The legacy VBIOS executes in **16-bit real mode** during POST and cannot address MMIO above 4 GB → framebuffer is unreachable → black screen.

This does **not** affect OS operation. Linux `nouveau` and the proprietary NVIDIA 470.xx driver run in 64-bit protected mode and re-map BARs correctly. Display returns to normal after driver load.

### Why Above 4G Decoding Matters

Cards with large 64-bit prefetchable BARs (e.g. **Tesla V100**: BAR up to 16 GB) **require** Above 4G Decoding. Without it, the firmware logs `PCI Out of Resources` and leaves the large BARs unmapped → GPU unusable for CUDA.

### Confirmed Non-Causes (ruled out by BIOS inspection on MRA9)

| BIOS Setting | Value | Finding |
|---|---|---|
| Active Video | Offboard | Correct — not the cause |
| Option ROM Execution → Video | Legacy | Already Legacy — not the cause |
| PCH Display | Enabled | Correct |
| Legacy VGA socket | Socket 0 (IIO) | Correct |

### Fix Option 1: Flash UEFI GOP VBIOS ✅ (Recommended)

Flash `vbios/190890.rom` (see above) to replace the legacy VBIOS with a UEFI GOP version. The card will then initialize properly during BIOS POST even with Above 4G Decoding enabled.

**Flash procedure (Linux, nvflash 5.867):**

> **nvflash binary:** `nvflash_5.867_linux.zip` is stored in this skill directory.
> The ZIP contains binaries for x64, x86, aarch64, and ppc64 — use `x64/nvflash` for mra9.
> Source: [TechPowerUp nvflash download page](https://www.techpowerup.com/download/nvidia-nvflash/)
> (Direct download requires a browser — Cloudflare JS challenge blocks curl/wget.)

```bash
# Copy nvflash to the target machine (Ubuntu live system)
scp ~/.copilot/skills/hardware/nvidia-gt710/nvflash_5.867_linux.zip mra9:/tmp/

# On mra9:
cd /tmp && unzip nvflash_5.867_linux.zip
chmod +x x64/nvflash

# Backup current VBIOS first (keep this safe!)
sudo x64/nvflash --save gt710_current.rom

# Verify tool sees the card
sudo x64/nvflash --list

# Unload driver (required)
sudo rmmod nouveau   # or: sudo rmmod nvidia

# Disable write protection, flash with subsystem override
sudo x64/nvflash --protectoff
sudo x64/nvflash -6 /path/to/190890.rom

# Reboot, then enable Above 4G Decoding in BIOS
sudo reboot
```

> **`-6` flag:** Overrides subsystem ID mismatch warning. Required when flashing a ROM with a different subsystem ID (e.g. generic P2132 ROM onto a ZOTAC `19da:5360` card).

> **Recovery:** If flash fails or display is lost:
> ```bash
> sudo x64/nvflash -6 /path/to/gt710_current.rom
> ```
> Keep `gt710_current.rom` backup in a safe location before flashing.

After flashing:
- In BIOS CSM settings, change **Option ROM Execution → Video** from `Legacy` to `UEFI` (or `Do Not Execute`) to let GOP initialize the card in UEFI mode.

### Fix Option 2: Accept Black BIOS Screen (Practical workaround)

If VBIOS flashing is not desired, you can operate with Above 4G Decoding enabled by managing the system headlessly:

- **BIOS POST / setup screens:** no display (navigate by memory or use IPMI/BMC if available)
- **OS boot and runtime:** display works normally once GPU driver loads
- **SSH:** always available regardless of display state
- **Compute GPUs:** fully functional (V100 BARs mapped correctly)

**Testing this approach:**
1. Enable Above 4G Decoding in BIOS (navigate blind)
2. Wait ~3 minutes for OS to boot
3. `ssh <hostname>` — if it connects, the system is functional
4. Verify no "PCI Out of Resources" errors: `dmesg | grep -i 'pci\|bar\|resource'`

## Driver Support

| OS | Driver |
|----|--------|
| Windows 10/11 | GeForce Game Ready 474.x (legacy Kepler support) |
| Linux | `nvidia` 470.xx branch (last version supporting Kepler) |
| Linux | `nouveau` open-source (limited 3D, lower performance) |

```bash
# Install legacy driver on Debian/Ubuntu
sudo apt install nvidia-legacy-470xx-driver
```

> **`nvidia-open` incompatibility:** Kepler architecture (GK208) is **not** supported by the `nvidia-open` kernel module. Must use the closed-source `nvidia` driver.

## Use Case: Secondary Display GPU in Compute Servers

The GT 710 is popular as a dedicated **display/management GPU** in servers housing compute-only cards (Tesla V100, A100, etc.) because:

- Draws no PCIe slot power beyond 25 W limit — no auxiliary connector needed
- Passive cooling — silent, reliable
- Fits in x1 PCH-attached slots, leaving CPU-attached x16 slots for compute
- Works as the CSM/legacy display adapter so compute GPUs (CUDA-only, no display output) don't need to handle POST

> After flashing UEFI GOP VBIOS, it also works correctly with Above 4G Decoding and Secure Boot enabled.
