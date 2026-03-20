---
name: acer-vn7-593g
description: Hardware reference for the Acer Aspire VN7-593G (Nitro Black Edition) gaming laptop. Use this skill when working on, configuring, or troubleshooting this specific machine (hostname: aspire). Covers CPU, GPU, memory, storage, display, networking, Thunderbolt dock, battery, BIOS, and software stack (Ubuntu 24.04 LTS / ZFS).
---
# Acer Aspire VN7-593G — Hardware Skill

## System Identity

| Field | Value |
|---|---|
| **Product** | Acer Aspire VN7-593G (Aspire V Nitro Black Edition) |
| **Hostname** | `aspire` |
| **Board** | KBL / Pluto_KLS, Revision V1.11 |
| **Chassis** | Type 10 = Notebook |
| **BIOS** | Insyde Corp. V1.11 (2018-08-01) |
| **OS** | Ubuntu 24.04.4 LTS (Noble Numbat) |
| **Kernel** | 6.17.0-19-generic (x86_64) |

---

## CPU

| Field | Value |
|---|---|
| **Model** | Intel Core i7-7700HQ (Kaby Lake) |
| **Cores / Threads** | 4 / 8 |
| **Base / Boost** | 2.80 GHz / 3.80 GHz |
| **TDP** | 45 W (mobile H-series) |
| **Cache** | L1d 128 KiB, L1i 128 KiB, L2 1 MiB, L3 6 MiB |
| **CPUID** | Family 6, Model 158 (0x9E), Stepping 9 |
| **Virtualization** | VT-x + VT-d (IOMMU) |
| **Notable extensions** | AVX2, AES-NI, Intel PT, TSX, SGX, MPX, SSE4.1/4.2 |
| **Spectre/Meltdown** | Mitigations active (old BIOS = no microcode update for some variants) |

---

## Chipset

| PCI ID | Component |
|---|---|
| `8086:a152` | Intel HM175 LPC/eSPI Controller (100/200 Series) |
| `8086:a123` | Intel 100/200 Series SMBus |
| `8086:a13a` | Intel ME Interface (CSME 11.x) |
| `8086:a131` | Intel 100 Series Platform Thermal Controller |
| `8086:a103` | Intel HM170/QM170 AHCI SATA Controller |

---

## Memory

| Field | Value |
|---|---|
| **Total installed** | 64 GB |
| **Slots** | 2× SO-DIMM DDR4 |
| **Speed** | DDR4-2400 (i7-7700HQ max) |
| **Max per slot** | 32 GB (official max 32 GB total, but 64 GB via 2×32 GB works) |
| **ECC** | No |

> ⚠️ The official Acer spec lists 32 GB max, but 2×32 GB SO-DIMMs work and the OS correctly reports 64 GB. `dmidecode` is needed (with `sudo`) to retrieve exact part numbers and SPD data.

---

## Graphics

### dGPU — NVIDIA GeForce GTX 1060 Mobile

| Field | Value |
|---|---|
| **PCI ID** | `10de:1c20` (GP106M) |
| **Bus** | PCIe x16 via Intel PCIe bridge `8086:1901` |
| **VRAM** | 6 GB GDDR5 |
| **Audio** | NVIDIA GP106 HDAC `10de:10f1` (HDMI/DP audio) |
| **Driver** | NVIDIA proprietary (production), or `nouveau` (open-source) |

### iGPU — Intel HD Graphics 630

| Field | Value |
|---|---|
| **PCI ID** | `8086:591b` (GT2, Kaby Lake) |
| **Driver** | `i915` |
| **Features** | Intel Quick Sync, HDCP, eDP/LVDS (internal panel) |
| **Audio** | Intel CM238 HD Audio `8086:a171` |

### Display Setup (PRIME / Optimus)

- Laptop uses NVIDIA Optimus: iGPU drives internal panel (eDP-1), dGPU handles external outputs.
- With the lid closed and Thunderbolt dock active, only `DP-2` is active under the current X session.
- DRM connectors on card1 (Intel i915): `eDP-1`, `DP-1`, `DP-2`, `HDMI-A-1`, `HDMI-A-2`.

---

## Internal Display

| Field | Value |
|---|---|
| **Panel** | AUO N156DCE-GA1 |
| **Size** | 15.6" IPS |
| **Resolution** | 3840×2160 (4K UHD) |
| **Connector** | eDP-1 (internal) |
| **Status** | Connected (eDP-1 present); lid may be closed when docked |

---

## Storage

| Device | Model | Interface | PCI ID / Controller | Size | Notes |
|---|---|---|---|---|---|
| `nvme0n1` | Samsung SSD 990 PRO 2TB | NVMe | `144d:a80c` (Samsung S4LV008) | ~1.86 TiB | ZFS `rpool` (OS root + home) |
| `sda` | WDC WDS200T2B0B-00YS70 | SATA | Intel AHCI `8086:a103` | ~1.86 TiB | EFI partition + swap + ZFS `bpool` |

### ZFS Layout

| Pool | Device | Mount | Notes |
|---|---|---|---|
| `rpool` | `nvme0n1` | `/` | Ubuntu root + home, ZFS native encryption |
| `bpool` | `sda` (part) | `/boot` | Boot pool on SATA SSD |
| `keystore-rpool` | `zd0` (zvol, 500 MB) | — | ZFS encryption keystore device |
| — | `sda1` (512 MB) | `/boot/efi` | FAT32 EFI System Partition |
| `cryptoswap` | `sda2` | swap | LUKS-encrypted swap on SATA SSD |

---

## Networking

| Interface | Device | PCI/USB ID | Notes |
|---|---|---|---|
| `enp3s0` | Realtek RTL8111/8168 Gigabit | `10ec:8168` | Built-in Ethernet, state UP / active |
| `wlp2s0` | Intel Wi-Fi 6 AX200 | `8086:2723` | Built-in WiFi (upgraded from original?); state DOWN when docked |
| `enxa453eed5e996` | Realtek RTL8153 Gigabit (USB) | `0bda:8153` | Via Thunderbolt dock; NO-CARRIER when disconnected |
| `virbr0` | libvirt bridge | — | KVM/libvirt virtual network |

---

## Thunderbolt

| Component | PCI ID |
|---|---|
| Intel JHL6340 Alpine Ridge 2C (TB3 controller) | `8086:15da` |
| Intel Alpine Ridge USB 3.1 Controller | `8086:15db` |

- Thunderbolt 3 via USB-C port (left side)
- Dock presents large USB hub tree (see USB section below)

---

## Audio

| Sound Card | Driver / Controller |
|---|---|
| HDA NVidia (card 0) | `snd_hda_intel` via `10de:10f1` — HDMI audio from GTX 1060 |
| HDA Intel PCH (card 1) | `snd_hda_intel` via `8086:a171` — Internal speakers, headphone jack, mic |

---

## Battery

| Field | Value |
|---|---|
| **Model** | LGC AC16A8N |
| **Chemistry** | Li-ion |
| **Cells** | 4-cell |
| **Nominal voltage** | 15.2 V |
| **Design capacity** | 4605 mAh (~70 Wh) |
| **Current full charge capacity** | ~2292 mAh (~35 Wh) |
| **Health** | ~50% (significantly degraded) |
| **Replacement SKU** | AC16A8N (third-party) or AP16A5K (some variants) |

> ⚠️ Battery is degraded to ~50% of original capacity. Runtime on battery is reduced. Consider replacement.

---

## Built-in USB Devices

| Device | USB ID | Notes |
|---|---|---|
| Chicony HD WebCam | `04f2:b571` | 720p/1080p internal webcam |
| Realtek RTS5129 Card Reader | `0bda:0129` | SD card slot (internal) |
| Elan WBF Fingerprint Sensor | `04f3:0c03` | Fingerprint reader (works with `fprintd`) |
| Intel AX200 Bluetooth | `8087:0029` | Bluetooth 5.x (co-located with WiFi card) |

---

## Thunderbolt Dock — USB Devices

The connected Thunderbolt dock presents the following hub tree:

| Device | USB ID | Notes |
|---|---|---|
| VIA Labs USB 3.1 Gen 1 Hub | `2109:0822` | Primary SuperSpeed hub |
| VIA Labs USB 3.0 Hub | `2109:0817` | Secondary SS hub |
| VIA Labs USB 2.0 Hub | `2109:2822` | HS companion hub |
| VIA Labs USB 2.0 Hub | `2109:2817` | HS companion hub |
| Genesys Logic GL3523 USB 3.2 Hub | `05e3:0610` | Additional hub in dock |
| Genesys Logic GL3523 SD Reader | `05e3:0749` | SD card slot in dock |
| Realtek RTL8153 Gigabit Ethernet | `0bda:8153` | Dock Ethernet port → `enxa453eed5e996` |
| ASIX AX68004 USB Ethernet ×2 | `0b95:6804` / `d31f:ac60` | Additional network ports |
| Terminus FE 1.1 USB Hub ×2 | `1a40:0101` | Small passive hubs |
| Logitech USB Receiver | `046d:c52b` | Unifying Receiver (keyboard/mouse) |
| Hypersecu HyperFIDO | `2ccf:0854` | FIDO2 / U2F security key |

---

## External Display (via Thunderbolt Dock)

| Field | Value |
|---|---|
| **Connector** | DP-2 (DisplayPort over Thunderbolt dock) |
| **EDID Name** | MULTIVIEWER |
| **Resolution** | 1920×1080 (active, primary output) |
| **Physical size** | 1150mm × 650mm (~53" — large screen or AV multiviewer unit) |

---

## BIOS / Firmware Notes

- **BIOS vendor:** Insyde Corp. (common for Acer)
- **Version:** V1.11 (2018-08-01) — no updates available as of late 2024
- **Insyde BIOS access:** F2 at POST
- **Boot menu:** F12 at POST
- **Secure Boot:** Available in BIOS (check current state with `mokutil --sb-state`)
- **VT-x / VT-d:** Enabled (confirmed via `/proc/cpuinfo`)

---

## Software Environment

| Component | Detail |
|---|---|
| **OS** | Ubuntu 24.04.4 LTS |
| **Kernel** | 6.17.0-19-generic (Ubuntu mainline HWE) |
| **Filesystem** | ZFS (OpenZFS via `zfsutils-linux`) |
| **Virtualization** | KVM/libvirt (`virbr0` bridge present) |
| **GPU mode** | PRIME/Optimus (NVIDIA proprietary likely with `prime-select`) |
| **Display server** | X11 (xrandr used successfully) or Wayland (check `$XDG_SESSION_TYPE`) |
| **Swap** | LUKS-encrypted swap on `sda2` (`cryptoswap`) |

---

## Thermal Zones

| Zone | Type | Temp at scan |
|---|---|---|
| thermal_zone0 | acpitz | 67 °C |
| thermal_zone1 | acpitz | 42 °C |
| thermal_zone2 | pch_skylake | N/A |

- hwmon0: ADP1 (AC adapter)
- hwmon1: acpitz
- hwmon2: BAT0 (battery)
- hwmon3: nvme (NVMe drive temperature)

---

## Known Quirks and Limitations

- ⚠️ **64 GB RAM**: The official Acer VN7-593G spec says 32 GB max. 2×32 GB SO-DIMMs work in practice.
- ⚠️ **Battery health**: Degraded to ~50% (4605 mAh design → ~2292 mAh actual). Replace with AC16A8N.
- ⚠️ **BIOS**: Insyde V1.11 (2018). No newer BIOS update known. Some Spectre/Meltdown mitigations may be limited.
- ⚠️ **PRIME**: The internal 4K eDP display (N156DCE-GA1) is driven by the Intel iGPU; the Thunderbolt/DP outputs go through the NVIDIA dGPU. When lid is closed and dock is in use, only `DP-2` is active.
- ⚠️ **WiFi card appears upgraded**: Intel AX200 (Wi-Fi 6) is not the original card for this 2016/2017 model. Original was likely Intel 7265 or similar. The AX200 works with `iwlwifi`.
- ⚠️ **ZFS encryption**: `rpool` uses ZFS native encryption; keystore lives on a 500 MB zvol (`zd0`). Ensure keystore is unlocked on boot.
- ⚠️ **Samsung 990 PRO NVMe**: Installed as aftermarket upgrade (original likely had a smaller SSD). Works perfectly under Linux with `nvme` driver.

---

## References

- [Acer Aspire VN7-593G product page](https://www.acer.com/ac/en/US/content/series/aspirevinitrobeblack)
- [Intel Core i7-7700HQ specifications](https://ark.intel.com/content/www/us/en/ark/products/97185/intel-core-i7-7700hq-processor-6m-cache-up-to-3-80-ghz.html)
- [Samsung 990 PRO NVMe](https://semiconductor.samsung.com/us/consumer-storage/internal-ssd/990-pro/)
- [Intel Wi-Fi 6 AX200 product brief](https://www.intel.com/content/www/us/en/products/sku/189347/intel-wi-fi-6-ax200/specifications.html)
- [AUO N156DCE-GA1 panel datasheet](https://www.panelook.com/N156DCE-GA1_AUO_15.6_LCM_overview_38073.html)
- [LGC AC16A8N battery replacement](https://www.ifixit.com/) (search: Acer Aspire V Nitro VN7-593G battery)
