---
name: ASRock Radeon RX 550 2GB
description: ASRock Radeon RX 550 2GB (AMD Lexa PRO, 1002:699f) — vBIOS structure (dual-image: x86 legacy + UEFI GOP), GPU passthrough / VFIO, AMD reset bug, ROM dump procedure, driver unbind method
---

# ASRock Radeon RX 550 2GB (AMD Lexa PRO)

## Overview

| Field | Value |
|-------|-------|
| Model | ASRock Radeon RX 550 2GB |
| GPU | AMD Lexa PRO (Lexa architecture, 14 nm) |
| PCI Vendor ID | `0x1002` (AMD) |
| PCI Device ID | `0x699f` rev `0xc7` |
| Audio function | `1002:aae0` (Baffin HDMI/DP Audio — AMD internal naming quirk; part of Lexa chip) |
| VRAM | 2 GB GDDR5 |
| TDP | ~50 W |
| PCIe | PCIe 3.0 x8/x16 |
| Config hint | `ASROCK_LEXA_D09001_PRO_G5_2G_GAMING\config.h` |
| Part number | `113-TIC32700-100` |

### Deployment on `mra9`

| Field | Value |
|-------|-------|
| Host | `mra9` (Machinist X99 MRA9 v1) |
| PCIe address | `03:00.0` (GPU), `03:00.1` (audio) |
| PCIe root port | CPU IIO `00:02.0`, slot 6 |
| DRI device | `dri/1` (amdgpu driver; DRI card1 = `/sys/kernel/debug/dri/1`) |
| Purpose | Dedicated GPU passthrough / VFIO testing |

---

## vBIOS

### Summary

The ASRock RX 550 2GB ships with a **dual-image vBIOS**: a legacy x86 PCI option ROM followed by a UEFI GOP image. Both images are present on-chip in the same flash ROM.

> **Key finding (confirmed 2025):** The `amdgpu` driver's debugfs interface (`/sys/kernel/debug/dri/N/amdgpu_vbios`) exposes **only the x86 legacy image** — the UEFI GOP image is invisible there. The full dual-image ROM is only accessible by reading the physical ROM BAR directly after unbinding the driver.

### vBIOS identification

| Field | Value |
|-------|-------|
| Version string | `ATOMBIOSBK-AMD VER015.050.002.001.032700` |
| Date | `03/27/18` |
| Part number | `113-TIC32700-100` |
| Total ROM size | **119,808 bytes** (234 × 512-byte blocks) |

### ROM image structure (full dump analysis)

| # | Offset | Size | Code Type | Description |
|---|--------|------|-----------|-------------|
| 1 | `0x0000` | 60,928 bytes (119 blocks) | `0x00` — x86 legacy | PC-AT real-mode initialization; last-image = **False** |
| 2 | `0xEE00` | 58,880 bytes (115 blocks) | `0x03` — UEFI/EFI | **UEFI GOP driver**; last-image = **True** |

#### Image 2 (UEFI GOP) — PCIR details

| Field | Value |
|-------|-------|
| PCIR signature | `PCIR` |
| Vendor ID | `0x1002` (AMD) |
| Device ID | `0x699f` |
| Code type | `0x03` = UEFI/EFI |
| Last image | `True` |
| GOP banner string | `GOP AMD REV: x.x.x.x.x` at absolute offset `0xEE34` |
| EFI payload | Compressed (compression byte `0x73` — AMD-proprietary scheme); no bare MZ/PE headers visible |

**UEFI GOP conclusion: ✅ CONFIRMED** — This card has a UEFI GOP driver in its stock vBIOS and does not need a vBIOS flash for UEFI GOP support.

---

## ROM Dump Procedure

The standard sysfs ROM interface fails while the `amdgpu` driver is active. The correct procedure is:

### Prerequisites

- A second GPU providing display (on `mra9`: GT 710 at `08:00.0`) so unbinding amdgpu is safe.
- Root / sudo access.

### Method: Unbind driver, read physical ROM BAR

```bash
# 1. Find the ROM BAR address
lspci -vvv -s 03:00.0 | grep "Expansion ROM"
# Example output: Expansion ROM at fb240000 [disabled] [size=128K]

# 2. Unbind the amdgpu driver
echo "0000:03:00.0" | sudo tee /sys/bus/pci/drivers/amdgpu/unbind

# 3. Enable the ROM BAR via sysfs
echo 1 | sudo tee /sys/bus/pci/devices/0000:03:00.0/rom

# 4. Dump the full ROM (use /proc/bus/pci or sysfs rom node after enable)
sudo cat /sys/bus/pci/devices/0000:03:00.0/rom > /tmp/rx550_fullrom.bin
echo 1 | sudo tee /sys/bus/pci/devices/0000:03:00.0/rom  # disable again

# 5. Rebind amdgpu
echo "0000:03:00.0" | sudo tee /sys/bus/pci/drivers/amdgpu/bind

# Verify: expect 119808 bytes (119808 = 234 × 512)
ls -la /tmp/rx550_fullrom.bin
xxd /tmp/rx550_fullrom.bin | head -1   # should start: 55 aa ...
```

> **Why not use debugfs?** `/sys/kernel/debug/dri/1/amdgpu_vbios` returns only 60,928 bytes — the x86 legacy image only. The driver loads this portion into its working buffer at init time; the UEFI image is not included.

> **Why not read /dev/mem?** The ROM BAR is `[disabled]` while the driver is active — memory at that physical address returns all-zeros. The driver-unbind + sysfs enable method is required.

### Quick PCIR parser (Python snippet)

```python
import struct

def parse_option_roms(data):
    offset = 0
    while offset < len(data):
        if data[offset:offset+2] != b'\x55\xaa':
            break
        img_size = data[offset+2] * 512
        pcir_off = offset + struct.unpack_from('<H', data, offset+0x18)[0]
        if data[pcir_off:pcir_off+4] == b'PCIR':
            code_type = data[pcir_off+0x14]
            last = bool(data[pcir_off+0x15] & 0x80)
            vid = struct.unpack_from('<H', data, pcir_off+4)[0]
            did = struct.unpack_from('<H', data, pcir_off+6)[0]
            print(f"  Offset 0x{offset:04X}: {img_size} bytes, code_type=0x{code_type:02X}, last={last}, VID={vid:#06x}, DID={did:#06x}")
        offset += img_size

with open('/tmp/rx550_fullrom.bin', 'rb') as f:
    parse_option_roms(f.read())
```

---

## GPU Passthrough / VFIO

### IOMMU group isolation

On `mra9`, the RX 550 at `03:00.0` shares the CPU IIO root port `00:02.0` with nothing else (the V100s are removed when the RX 550 is installed). GPU and audio function are in the same IOMMU group — both must be passed through together.

PCI IDs for VFIO:
```
options vfio-pci ids=1002:699f,1002:aae0
```

### AMD reset bug

| Field | Value |
|-------|-------|
| Affected | **Yes** — Lexa/Polaris family is affected by the AMD GPU reset bug |
| Root cause | Hardware FLR (Function Level Reset) does not fully reinitialize the GPU; subsequent VM boot leaves the GPU in a broken state |
| vBIOS update fix? | **❌ No** — the reset bug is a silicon-level FLR issue, completely independent of vBIOS content. Updating vBIOS cannot fix it. |

#### Fixes

| Method | Reliability | Notes |
|--------|-------------|-------|
| `vendor-reset` kernel module | ✅ Most reliable | [https://github.com/gnif/vendor-reset](https://github.com/gnif/vendor-reset) — implements proper reset sequences for affected AMD GPUs |
| `amdgpu.reset_method=4` | ✅ Works for many setups | Kernel parameter; uses SMU reset path instead of FLR; add to `/etc/default/grub` |
| VFIO `no_iommu` mode | ⚠️ Security tradeoff | Not a reset bug fix; separate concern |

```bash
# Install vendor-reset (Ubuntu)
sudo apt install dkms linux-headers-$(uname -r)
git clone https://github.com/gnif/vendor-reset
cd vendor-reset && sudo dkms install .
sudo modprobe vendor-reset
# Add to /etc/modules for persistence
echo vendor-reset | sudo tee -a /etc/modules
```

Or kernel parameter approach:
```bash
# /etc/default/grub — add amdgpu.reset_method=4 to GRUB_CMDLINE_LINUX_DEFAULT
sudo sed -i 's/GRUB_CMDLINE_LINUX_DEFAULT="\(.*\)"/GRUB_CMDLINE_LINUX_DEFAULT="\1 amdgpu.reset_method=4"/' /etc/default/grub
sudo update-grub && sudo reboot
```

### UEFI GOP and guest display

Since the vBIOS **already has UEFI GOP** (confirmed), no vBIOS flash is required for:
- OVMF/UEFI guest boot with correct display output from the passed-through GPU
- BIOS-level display during POST (if used as display GPU in a UEFI system with CSM disabled)

---

## Troubleshooting

### sysfs ROM read returns I/O error or 0xFFFF signature

Cause: `amdgpu` driver holds the ROM BAR active; the PCI ROM BAR is set `[disabled]` by the OS while the driver is running.

Fix: Use the **driver unbind method** above.

### amdgpu_vbios debugfs dump is smaller than expected

Expected: 60,928 bytes (x86 image only) — this is correct and by design. The driver loads only the x86 legacy portion into its internal vbios buffer. Use the ROM BAR method for the full 119,808-byte dual-image dump.

### GPU not recognized after VM shutdown (reset bug symptom)

Symptom: GPU shows as non-functional; `amdgpu` reports errors on reload; subsequent VM won't boot GPU.

Fix: Load `vendor-reset` module (see above), or add `amdgpu.reset_method=4` kernel parameter.
