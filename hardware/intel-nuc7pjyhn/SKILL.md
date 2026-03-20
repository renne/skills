---
name: intel-nuc7pjyhn
description: Hardware reference for the Intel NUC7PJYHN (NUC Kit with Pentium Silver J5040, Gemini Lake). Use when managing or troubleshooting pve2, the Proxmox VE hypervisor at Friedensstraße that runs on this NUC hardware — covers CPU, memory, storage, networking, BIOS, and PVE-specific kernel/IOMMU configuration.
---
# Intel NUC7PJYHN — Hardware Reference

Host using this hardware: **pve2** (Proxmox VE hypervisor, Friedensstraße)

## Board / System

| Parameter | Value |
|---|---|
| Product Name | NUC7PJYHN |
| Board Name | NUC7JYB |
| Board Vendor | Intel Corporation |
| Board Version | M37329-601 |
| Product Version | M37320-601 |
| Form Factor | Intel NUC (Mini PC) |
| BIOS Vendor | Intel Corp. |
| BIOS Version | JYGLKCPX.86A.0067.2021.1111.1431 |
| BIOS Date | 2021-11-11 |
| BIOS Access | F2 = BIOS Setup, F10 = Boot Menu |

## CPU

| Parameter | Value |
|---|---|
| Model | Intel Pentium Silver J5040 |
| Architecture | Gemini Lake (SoC) |
| Cores / Threads | 4C / 4T (no Hyper-Threading) |
| Base / Boost | 2.0 GHz / 3.2 GHz |
| TDP | 10 W |
| VT-x | ✅ Yes |
| VT-d | ❌ No (J-series Pentium Silver lacks VT-d) |
| L2 Cache | 4 MiB (shared) |
| L1d | 96 KiB (4× 24 KiB) |
| L1i | 128 KiB (4× 32 KiB) |

> ⚠️ **No VT-d:** PCIe passthrough (VFIO) is **not supported** on pve2 despite `intel_iommu=on pcie_acs_override` being set in the kernel cmdline. Passthrough attempts will fail.

## Memory

| Parameter | Value |
|---|---|
| Slots | 2× SO-DIMM (DDR4) |
| Installed | 2× 8 GB = 16 GB total |
| Part Number | Micron 8ATF1G64HZ-2G3B1 |
| Speed | DDR4-2400 (2400 MT/s configured) |
| ECC | No |
| Max Supported | 16 GB (2× 8 GB; board does not support 32 GB modules) |
| Channels | Channel A Slot 0 (SODIMM1) + Channel B Slot 0 (SODIMM2) |

## Graphics

| Component | Value |
|---|---|
| iGPU | Intel UHD Graphics 605 (Gemini Lake) |
| PCI ID | `8086:3185` |
| Display Output | HDMI 2.0a, Mini DisplayPort 1.2 |
| GVT-g | Not supported on Gemini Lake (GVT requires Broadwell/Skylake–Ice Lake) |

## Storage

| Device | Interface | Size | Model | Use |
|---|---|---|---|---|
| `/dev/sda` | SATA (internal 2.5") | ~233 GB | Samsung SSD 840 Series | ZFS `rpool` (single-disk) |

### ZFS Pool

| Pool | Topology | Size | Used | Health |
|---|---|---|---|---|
| `rpool` | Single disk (sda3) | 230 GB | ~25 GB | ONLINE |

- Root filesystem: `rpool/ROOT/pve-1`
- Boot: EFI partition on sda2

> ⚠️ No redundancy — single-disk ZFS pool. Disk failure = data loss.

## Networking

| Interface | MAC | Type | State | Notes |
|---|---|---|---|---|
| `eno1` / `enp2s0` | `88:ae:dd:06:4d:07` | Realtek RTL8168 (1 GbE) | UP | Bridged to `vmbr0` |
| `wlo2` / `wlp0s12f0` | `50:84:92:87:3c:dc` | Intel CNVi Wi-Fi (Gemini Lake) | DOWN | Unused |

## Kernel & PVE Configuration

| Parameter | Value |
|---|---|
| PVE Version | pve-manager/9.1.6 |
| Kernel | 6.17.4-1-pve |
| Boot cmdline | `intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction` |

> `intel_iommu=on` is set but has no practical effect — J5040 has no VT-d hardware. `pcie_acs_override` is also a no-op here.

## PCI Devices

| Bus | Device | Description |
|---|---|---|
| 00:00.0 | Host bridge | Intel Gemini Lake Host Bridge |
| 00:02.0 | VGA | Intel UHD Graphics 605 |
| 00:0c.0 | Network controller | Intel Gemini Lake CNVi Wi-Fi |
| 00:0e.0 | Audio | Intel Celeron/Pentium Silver HD Audio |
| 00:12.0 | SATA | Intel Celeron/Pentium Silver SATA Controller |
| 00:15.0 | USB | Intel Celeron/Pentium Silver USB 3.0 xHCI |
| 01:00.0 | Card reader | Realtek RTS522A PCIe Card Reader |
| 02:00.0 | Ethernet | Realtek RTL8111 GbE |

## Proxmox VE Guests

See `~/.copilot/networks/friedensstrasse/pve2/SKILL.md` for full VM/LXC inventory.

Summary:
- VM 105 `haos-whm` — Home Assistant OS (2 GB RAM, 32 GB disk)
- LXC 104 `homegear` — HomeMatic gateway (2 GB RAM, 16 GB disk)
- LXC 111 `nb1` — Netbird routing peer for 192.168.178.0/24

## Power

| Parameter | Value |
|---|---|
| Adapter | 19V DC (NUC slim tip) |
| Typical TDP | ~10–15 W under load |
| Max | ~35 W |

## Known Quirks

- **No VT-d / No PCIe passthrough**: The J5040 SoC does not expose IOMMU groups. Do not attempt USB or PCIe device passthrough.
- **16 GB RAM ceiling**: NUC7JYB only supports up to 16 GB total (2× 8 GB SO-DIMM). 32 GB SO-DIMMs are not recognized.
- **Single-disk ZFS**: No redundancy. Consider periodic snapshots or PBS backups.
- **CNVi Wi-Fi disabled**: The integrated Wi-Fi is administratively down; the NUC uses wired Ethernet only.

## References

- [Intel NUC7PJYHN Product Brief](https://www.intel.com/content/www/us/en/products/sku/126143/intel-nuc-kit-nuc7pjyhn/specifications.html)
- [Intel NUC7JYB Board Specs](https://www.intel.com/content/www/us/en/products/sku/130394/intel-nuc-board-nuc7jyb/specifications.html)
- Network skill: `~/.copilot/networks/friedensstrasse/pve2/SKILL.md`
