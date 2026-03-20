---
name: fujitsu-d3643-h1
description: Hardware reference for the Fujitsu CELSIUS W550 / Mainboard D3643-H1 (Intel Celeron G4900, Coffee Lake). Use when managing or troubleshooting pve3, the Proxmox VE hypervisor at Weißdornweg that runs on this Fujitsu workstation hardware — covers CPU, memory, storage (ZFS mirror NVMe), networking (I350 4-port GbE), BIOS, IOMMU/GVT, and PVE-specific configuration.
---
# Fujitsu D3643-H1 — Hardware Reference

Host using this hardware: **pve3** (Proxmox VE hypervisor, Weißdornweg)

## Board / System

| Parameter | Value |
|---|---|
| Product Name | Fujitsu CELSIUS W550 workstation |
| Board Name | D3643-H1 |
| Board Vendor | FUJITSU |
| Part Number | S26361-D3643-H1 |
| BIOS Vendor | FUJITSU // American Megatrends Inc. |
| BIOS Version | V5.0.0.13 R1.5.0 for D3643-H1x |
| BIOS Date | 2019-02-21 |
| Form Factor | Micro-Tower / Desktop workstation |

## CPU

| Parameter | Value |
|---|---|
| Model | Intel Celeron G4900 |
| Architecture | Coffee Lake (LGA 1151v2) |
| Cores / Threads | 2C / 2T (no Hyper-Threading) |
| Base Clock | 3.10 GHz |
| TDP | 54 W |
| VT-x | ✅ Yes |
| VT-d | ✅ Yes (enabled via `intel_iommu=on`) |
| L1d | 64 KiB (2× 32 KiB) |
| L1i | 64 KiB (2× 32 KiB) |
| L2 | 512 KiB (2× 256 KiB) |
| L3 | 2 MiB (shared) |

## Memory

| Parameter | Value |
|---|---|
| Slots | 4× DIMM (DDR4, dual-channel) |
| Installed | 4× 16 GB = **64 GB total** |
| Part Number | Corsair CMK32GX4M2A2666C16 (Vengeance LPX 32 GB kit) |
| Rated Speed | DDR4-2666 (XMP) |
| Configured Speed | **2133 MT/s** (JEDEC default; XMP not enabled in BIOS) |
| ECC | No |
| Slots Used | DIMM CHA1, CHA3, CHB2, CHB4 (all 4 slots populated) |

> ⚠️ RAM runs at 2133 MT/s despite Corsair XMP rating of 2666. The D3643-H1 BIOS or the Celeron G4900 may cap at 2133. Enable XMP in BIOS if higher speed is desired (verify stability).

## Graphics

| Component | Value |
|---|---|
| iGPU | Intel UHD Graphics 610 (Coffee Lake GT1) |
| PCI ID | `8086:3e92` |
| GVT-g | ✅ Enabled (`i915.enable_gvt=1` in kernel cmdline) — allows iGPU mediated passthrough |

## Storage

| Device | Interface | Size | Model | Use |
|---|---|---|---|---|
| `nvme0n1` | NVMe (PCIe x4) | 3.6 TB | Samsung SSD 990 PRO 4TB | ZFS `rpool` mirror leg |
| `nvme1n1` | NVMe (PCIe x4) | 3.6 TB | Samsung SSD 990 PRO 4TB | ZFS `rpool` mirror leg |
| `nvme2n1` | NVMe (PCIe x4) | 465.8 GB | Samsung SSD 970 EVO 500GB | Standalone (no partitions; available for passthrough or future use) |

### ZFS Pool

| Pool | Topology | Raw Size | Usable | Used | Health |
|---|---|---|---|---|---|
| `rpool` | `mirror-0` (nvme0n1p3 + nvme1n1p3) | 2× 3.6 TB | 3.63 TB | ~1.31 TB | ONLINE |

- Root filesystem: `rpool/ROOT/pve-1`
- Boot: EFI partition on each NVMe drive (nvme0n1p2, nvme1n1p2)
- Last scrub: 2026-03-08, no errors
- Pool upgrade pending (`zpool upgrade` needed for latest feature flags)

### Proxmox Storage Pools

| Name | Type | Total | Used | Notes |
|---|---|---|---|---|
| `local` | dir (`/var/lib/vz`) | 2.74 TB | ~367 GB | On ZFS rpool |
| `local-zfs` | ZFS pool | 2.88 TB | ~510 GB | `rpool` directly |
| `backup` | PBS | 2.93 TB | ~560 GB | Proxmox Backup Server (remote) |

## Networking

| Interface | MAC | Type | State | Notes |
|---|---|---|---|---|
| `i219lm` (onboard) | — | Intel I219-LM (1 GbE) | not bridged | Available, unused in current config |
| `i350_0` | `a0:36:9f:85:e2:c4` | Intel I350 4-port GbE | DOWN | Bridged to `vmbr0` (link down) |
| `i350_1` | `a0:36:9f:85:e2:c5` | Intel I350 4-port GbE | DOWN | Bridged to `vmbr0` (link down) |
| `i350_2` | `a0:36:9f:85:e2:c6` | Intel I350 4-port GbE | DOWN | Bridged to `vmbr0` (link down) |
| `i350_3` | `a0:36:9f:85:e2:c7` | Intel I350 4-port GbE | **UP** | Bridged to `vmbr0` — **active uplink** |

The Intel I350 quad-GbE card is connected via an ASMedia ASM2806 4-Port PCIe x2 Gen3 switch, which in turn is connected to the CPU via a PCIe x16 slot (downclocked to x2 per port due to the switch).

> **Only i350_3 is cabled.** The other three ports are available for future use (bonding, redundancy, additional VMs).

## PCI Devices

| Bus | Device | Description |
|---|---|---|
| 00:02.0 | VGA | Intel UHD Graphics 610 |
| 00:1f.6 | Ethernet | Intel I219-LM (onboard GbE) |
| 01:00.0 | PCI bridge | ASMedia ASM2806 PCIe x2 Gen3 Switch (upstream) |
| 02:00.0–0e.0 | PCI bridges | ASMedia ASM2806 downstream ports |
| 04:00.0 | NVMe | Samsung SSD 990 PRO 4TB (nvme1) |
| 05:00.0 | NVMe | Samsung SSD 990 PRO 4TB (nvme0) |
| 07:00.0 | NVMe | Samsung SSD 970 EVO 500GB (nvme2) |
| 08:00.0–3 | Ethernet | Intel I350 4-port GbE (i350_0 to i350_3) |

## Kernel & PVE Configuration

| Parameter | Value |
|---|---|
| PVE Version | pve-manager/8.4.17 |
| Kernel | 6.8.12-19-pve |
| Boot cmdline | `intel_iommu=on intel_iommu=pt i915.enable_gvt=1` |

- `intel_iommu=on`: Enables IOMMU for VT-d passthrough
- `intel_iommu=pt`: Pass-through mode (devices not used by host bypass IOMMU for performance)
- `i915.enable_gvt=1`: Enables GVT-g mediated passthrough for Intel UHD 610

## Proxmox VE Guests

See `~/.copilot/networks/weissdornweg/pve3/SKILL.md` for full VM/LXC inventory.

Summary:
- VM 101 `openwrt` — OpenWrt router (stopped)
- VM 103 `docker` — Docker host (32 GB RAM, 1 TB disk, running)
- VM 106 `homeassistant` — Home Assistant OS (8 GB RAM, 32 GB disk, running)
- VM 109 `Raspberrymatic` — HomeMatic CCU (⚠️ DO NOT DELETE — contains HomeMatic network key)
- VM 110 `pc` — Desktop PC VM (16 GB RAM, ~465 GB disk, stopped)
- LXC 107 `dns1` — CoreDNS + Netbird routing peer (10.0.0.7)
- LXC 108 `dns2` — CoreDNS + Netbird routing peer (10.0.0.8)

## Power

| Parameter | Value |
|---|---|
| PSU | Standard ATX (Fujitsu CELSIUS W550 internal) |
| CPU TDP | 54 W |
| Estimated system power | 80–120 W under typical PVE load |

## Known Quirks

- **RAM at 2133 instead of 2666**: Corsair Vengeance LPX XMP profile not active; runs at JEDEC 2133 MT/s. Enable XMP in BIOS → Advanced → Memory Config if 2666 is desired.
- **ZFS pool upgrade pending**: `zpool upgrade rpool` needed to enable latest feature flags. Do this during a maintenance window (pool will be unavailable to older ZFS versions after upgrade).
- **ASM2806 PCIe bandwidth**: The I350 quad-GbE card shares PCIe x2 Gen3 bandwidth (~1.97 GB/s) across up to 4 ports via the ASMedia switch. Full-duplex 4× 1 GbE (~4 Gbps) is possible but would approach the switch bandwidth limit.
- **nvme2n1 (970 EVO) unpartitioned**: No partitions visible — device is attached but not currently used by any storage pool. Available for passthrough to a VM or addition to ZFS as cache/log.
- **pve3 hostname is `pve` in DMI/OS** but the Proxmox node name and SSH hostname are both `pve3`. The DMI product string just shows the board name, not the customized hostname.
- **ACME certificates**: pve3 uses Let's Encrypt production with INWX DNS plugin for `pve.bartschnet.de` and `pve1.bartschnet.de`.

## References

- [Fujitsu D3643-H1 Product Page](https://www.fujitsu.com/global/products/computing/peripheral/accessories/d3643-h1.html)
- [Intel Celeron G4900 Specs](https://ark.intel.com/content/www/us/en/ark/products/129909/intel-celeron-processor-g4900-2m-cache-3-10-ghz.html)
- [Intel I350-T4 Datasheet](https://www.intel.com/content/www/us/en/ethernet-products/gigabit-server-adapters/ethernet-server-adapter-i350-brief.html)
- Network skill: `~/.copilot/networks/weissdornweg/pve3/SKILL.md`
