---
name: ymzx-4gpu-q1
description: YMZX-4GPU-Q1 quad-SXM2 GPU carrier baseboard. Hosts 4× NVIDIA SXM2 GPUs (V100, P100 confirmed compatible) with NVLink. Alternative to the 2-slot TNS-2SXM2-4P54 for building high-VRAM homelab AI clusters. Use this skill when working with this baseboard, SXM2 GPU installation, NVLink 4-GPU setups.
---
# YMZX-4GPU-Q1 — Quad-SXM2 GPU Carrier Baseboard

| Field | Value |
|---|---|
| **Model** | YMZX-4GPU-Q1 |
| **Manufacturer** | YMZX (Guangzhou) AI Technology Co., Ltd. (广州意锟智能科技有限公司) |
| **Website** | https://nusgrtii.cn |
| **Founded** | ~2020–2021 |
| **Type** | Quad-SXM2 GPU carrier / baseboard |
| **GPU Slots** | 4× SXM2 (NVIDIA SXM2 form factor) |
| **NVLink** | Supported — requires on-board mux/NVSwitch chips for 4-GPU topology |
| **Host Interface** | Likely 4× SFF-8654-8i SlimSAS x8 PCIe 3.0 (confirmed on 2-slot TNS variant; unconfirmed for YMZX-4GPU-Q1) |
| **Power** | 12V DC per SXM2 module; on-board VRMs per slot |
| **Connectors** | Amphenol Meg-Array (PCIe + NVLink per module) |
| **Price (2026)** | ~$818 USD (eBay) |
| **eBay listing** | https://www.ebay.com/itm/298083941971 |
| **Origin** | China (Shenzhen/Guangdong) |
| **Related board** | TNS-2SXM2-4P54 (2-slot variant; same BIOS requirements and connector family) |
| **Software ecosystem** | 1CatAI / 1Cat-vLLM (https://github.com/1CatAI/1Cat-vLLM) |

## Photos

![YMZX-4GPU-Q1 board — quad-SXM2 GPU carrier (source: angrysysadmins.tech)](photos/YMZX-4GPU-Q1.jpg)

## Supported GPUs

| GPU | VRAM | NVLink | Notes |
|---|---|---|---|
| NVIDIA Tesla V100 SXM2 16 GB | 16 GB HBM2 | Yes | Confirmed compatible class |
| NVIDIA Tesla V100 SXM2 32 GB | 32 GB HBM2 | Yes | Confirmed compatible class |
| NVIDIA DRIVE A100 / PG199 | 32 GB HBM2e | Yes | Physically compatible (SXM2 socket); requires 1200W PSU |
| NVIDIA Tesla P100 SXM2 | 16 GB HBM | Likely | Same SXM2 socket family |

> **Note:** 4-GPU NVLink scaling requires on-board NVSwitch or mux chips. Whether YMZX-4GPU-Q1
> implements full NVSwitch mesh topology is unconfirmed. The blog author notes uncertainty:
> *"I wonder how well NVLink scales past 2 GPU's"* (angrysysadmins.tech, 2026-03).

## BIOS / System Requirements

Same requirements as TNS-2SXM2-4P54:

- **UEFI boot** enabled; Legacy CSM boot disabled
- Host OS booted via UEFI (Windows or Linux)
- **Above 4G Decoding** enabled
- **Resizable BAR** (Large Bar Support) enabled
- Recommended: SR-IOV, PCIe ARI, Enhanced PCIe error reporting

## Power Requirements

| GPU Config | Recommended PSU |
|---|---|
| 4× V100 16 GB or 32 GB | ATX3.1 1600W+ (Gold or better) |
| 4× PG199 (DRIVE A100) | ATX3.1 2000W+ (Platinum recommended) |

> SXM2 VRMs are on-board; board draws 12V from EPS/ATX connectors and regulates internally.
> Connect ATX 24-pin + sufficient 8-pin EPS/CPU connectors (count per board manual).

## Cooling

- Requires active cooling per SXM2 module
- Standard SXM2 heatsink + fan approach (same as TNS-2SXM2-4P54 assembly)
- 4× 80mm fans (e.g., ARCTIC P8 3000 RPM) per module recommended
- Ambient temp: keep modules below 55°C under load

## NVLink Topology Notes

- **2-GPU NVLink**: Direct connection, no mux chips needed — fully confirmed on 2-slot boards
- **4-GPU NVLink**: Requires on-board NVSwitch/mux chips for full mesh; YMZX-4GPU-Q1 likely includes these but NVLink scaling effectiveness at 4× is unconfirmed in homelab testing
- NVLink 2 (V100/SXM2): 150 GB/s read + 150 GB/s write = 300 GB/s total bidirectional per link
- Compare: PCIe 3.0 ×16 ≈ 16 GB/s total — NVLink is ~19× faster for GPU-GPU bandwidth

## Host OS & Driver Notes

Same as TNS-2SXM2-4P54:

```
# Kernel cmdline — prevent framebuffer conflict at boot
initcall_blacklist=simpledrm_platform_driver_init

# /etc/modprobe.d/nvidia-drm.conf — disable framebuffer for headless GPU nodes
options nvidia-drm fbdev=0

# /etc/modprobe.d/nvidia-power.conf — enable dynamic power management
options nvidia NVreg_DynamicPowerManagement=0x02
```

- **Driver**: Use `nvidia-580xx-dkms` (or equivalent closed-source); `nvidia-open` does **not** work with V100 (GV100) or PG199/GA100
- Confirmed working on: CachyOS (Arch-based), any modern Linux with UEFI boot

## Known Issues / Limitations

| Issue | Details |
|---|---|
| Dell iDRAC9 fan runaway | iDRAC ramps fans to max for SXM2 PCIe thermal readings; cannot be overridden — avoid Dell servers |
| NVLink 4-GPU scaling | Unconfirmed in homelab; NVSwitch presence and topology not publicly documented |
| Closed-source driver required | `nvidia-open` incompatible with GV100 (V100) and GA100 (PG199/DRIVE A100) |
| No official documentation | No public spec sheet; board from Chinese OEM with minimal support |

## Real-World Performance (2× V100 SXM2 32 GB on TNS-2SXM2-4P54)

Source: angrysysadmins.tech blog (AMD Threadripper 2950X host):

| Model | Generation Speed (tok/s) |
|---|---|
| qwen3-coder:30b | 98.84 |
| nemotron-3-nano:30b | 122.51 |
| lfm2:24b | 138.48 |
| gpt-oss:20b | 119.43 |
| llama3.2:3b | 207.62 |

> Powered by [1Cat-vLLM](https://github.com/1CatAI/1Cat-vLLM) — fork of vLLM for Tesla V100 (SM70) with AWQ 4-bit quantization, CUDA 12.8, and vLLM v0.9.1.

## Comparison: YMZX-4GPU-Q1 vs TNS-2SXM2-4P54

| Feature | YMZX-4GPU-Q1 | TNS-2SXM2-4P54 |
|---|---|---|
| GPU slots | 4× SXM2 | 2× SXM2 |
| Max VRAM (V100 32G) | 128 GB | 64 GB |
| NVLink GPUs | 4 (mux required) | 2 (direct) |
| Host interface | Likely 4× SFF-8654-8i (unconfirmed) | 4× SFF-8654-8i (confirmed) |
| Companion riser | Unknown | 2× RTE162P54B-2UR |
| Price (2026) | ~$818 | Kit ~$200–400 |
| NVLink tested | Unconfirmed at 4-GPU | Confirmed 2-GPU |

## References

- Blog post with first mention: https://angrysysadmins.tech/index.php/2026/03/grassyloki/cheapish-ai-homelab-on-a-budget-v100s-custom-boards-and-nvlink/
- eBay listing: https://www.ebay.com/itm/298083941971
- Board photo source: https://angrysysadmins.tech/wp-content/uploads/2026/03/YMZX-4GPU-Q1.jpg
- SXM2 reverse engineering: https://github.com/l4rz/running-nvidia-sxm-gpus-in-consumer-pcs
- SXM socket info: https://en.wikipedia.org/wiki/SXM_(socket)
- YMZX company profile: https://nusgrtii.cn
- 1CatAI vLLM (V100-optimized): https://github.com/1CatAI/1Cat-vLLM

## TODO / Unresolved

- [ ] Confirm exact PCIe host interface (connector type and count)
- [ ] Confirm NVLink topology at 4 GPUs (NVSwitch chip presence)
- [ ] Confirm power connector count and pinout
- [ ] Obtain board-specific BIOS/firmware flash procedure if applicable
- [ ] Test with DRIVE A100 / PG199 (physically compatible, untested)
