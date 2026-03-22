---
name: tns-2sxm2-4p54
description: TNS-2SXM2-4P54 dual-SXM2 GPU carrier baseboard. Hosts 2× NVIDIA SXM2 GPUs (V100 SXM2 confirmed; DRIVE A100/PG199 SXM2 physically compatible) and connects to a host via 4× SFF-8654-8i (SlimSAS x8) PCIe 4.0 ports through 2× RTE162P54B-2UR risers. Use this skill when working with this baseboard, SXM2 GPU installation, NVLink, V100/A100 homelab setups.
---
# TNS-2SXM2-4P54 — Dual-SXM2 GPU Carrier Baseboard

| Field | Value |
|---|---|
| **Model** | TNS-2SXM2-4P54 |
| **P/N** | 97.R529V07 |
| **S/N** | YR25111801002? (partially legible) |
| **Type** | Dual-SXM2 GPU carrier / baseboard |
| **GPU Slots** | 2× SXM2 (NVIDIA SXM2 form factor; each socket has 2 rectangular connector halves) |
| **Host Interface** | 4× SFF-8654-8i (SlimSAS x8, PCIe 4.0) on left edge, labelled X16/X8 — "4P54" in model name |
| **PCIe bandwidth** | PCIe 4.0, 4× x8 = ~64 GB/s aggregate to host |
| **Power** | 2× large PSU connectors (~20-24 pin) + 2× 8-pin PCIe power + aux connectors on top edge |
| **Companion riser** | RTE162P54B-2UR — 2× per baseboard (each riser carries 2× SFF-8654-8i) |
| **Host platform** | Intel Xeon server (confirmed via `IntelRCSetup` tab in AMI Aptio BIOS v2.18.1263) |
| **GPUs installed** | 2× NVIDIA V100 SXM2 (confirmed; see assembled photo below) |
| **Coolers** | 2× SXM2 heatsink — aluminum fin stack with copper heat pipes |

## Physical Description
- Large black PCB, SXM2 sockets arranged vertically (top + bottom), power headers along top edge
- Left edge: 4× SFF-8654-8i stacked in two pairs; silkscreen shows X16 and X8 lane designations per group
- Back/solder side: 2× metal SXM2 socket retainer frames with rectangular cutouts; sparse SMD population
- Right edge: board version/revision silkscreen; CE/FCC marks

## Photos

### Board — Front (component side)
![TNS-2SXM2-4P54 front PCB, component side](photos/PXL_20260314_104840199.jpg)

### Board — Back (solder side / SXM2 socket frames)
![TNS-2SXM2-4P54 back side with SXM2 socket retainer frames](photos/PXL_20260314_104823252.jpg)

### Assembled — 2× V100 SXM2 + Heatsinks
![TNS-2SXM2-4P54 assembled with 2× NVIDIA V100 SXM2 and heatsinks in shipping box](photos/PXL_20260314_180902214.jpg)

### BIOS — PCI Resource Error
![BIOS screen showing PCI resource exhaustion / Above 4G Decoding issue](photos/PXL_20260322_114203660.MP.jpg)

### BIOS — CSM Configuration
![BIOS CSM (Compatibility Support Module) configuration screen](photos/PXL_20260322_125720017.jpg)

## Model Name Breakdown
- **TNS** — product line prefix (anonymous Chinese OEM)
- **2SXM2** — 2× SXM2 GPU slots
- **4P54** — 4× SFF-8654 (SlimSAS) ports; same "P54" family identifier as on RTE162P54B-2UR risers

## Kit Contents (eBay listing 136861481597)
- 1× TNS-2SXM2-4P54 baseboard
- 2× RTE162P54B-2UR risers (PCIe 3.0 x16 → 2× SFF-8654, with full-height and half-height brackets)
- 4× SFF-8654 cables

## Board Features
- 6× fan headers — supports both DC and PWM fans
- AT/MT power override switch — selects between PCIe-controlled power or always-on
- Onboard power regulation, fan controllers, and thermal/temperature probes
- Debug headers

## Vendor & Sourcing
- Manufacturer: anonymous Chinese OEM ("made by who knows") — no official documentation
- Primary source: eBay item **136861481597**
- Knowledge source: community reverse-engineering and homelab blog posts
- Blog reference: [angrysysadmins.tech — Cheap-ish AI Homelab: V100s, Custom Boards, and NVLink](https://angrysysadmins.tech/index.php/2026/03/grassyloki/cheap-ish-ai-homelab-on-a-budget-v100s-custom-boards-and-nvlink/)

## Compatible GPUs

| GPU | Compatibility | Notes |
|---|---|---|
| **NVIDIA Tesla V100 SXM2** | ✅ Confirmed | 16 GB or 32 GB HBM2; photos show confirmed installation |
| **NVIDIA DRIVE A100 (PG199) SXM2** | ⚠️ Likely compatible | SXM2 socket is physically identical; electrically confirmed on generic SXM2 boards; **not specifically tested on TNS-2SXM2-4P54** |
| **NVIDIA P100 SXM2** | ⚠️ Probably compatible | Same SXM2 socket; untested on this board |
| SXM3, SXM4, SXM6 | ❌ Not compatible | Different socket/connector |

### PG199 (NVIDIA DRIVE A100) — Compatibility Notes
- **PG199** = NVIDIA DRIVE A100 Automotive, Ampere GA100 die, SXM2 physical socket, **32 GB HBM2e**, ~400 W TDP
- Part number: `900-6G199-0000-C00`
- **NOT** the datacenter A100 SXM4 (PG506) — that uses a different socket and is not compatible
- SXM2 pinout is largely identical between V100 and PG199 for PCIe signal/power; confirmed working in generic SXM2 adapters and non-OEM boards
- Some SXM2 adapter projects (OSHWHub, GitHub) explicitly list P100/V100/PG199 "电气兼容" (electrically compatible)
- Community reports (ServeTheHome) show PG199 working in third-party SXM2 carrier boards; **no specific confirmation for TNS-2SXM2-4P54 found**
- **Power**: PG199 draws up to ~400 W; verify TNS-2SXM2-4P54 power headers can deliver this (vs V100 ~300 W)
- **Cooling**: PG199 has no integrated heatsink — requires custom water block or third-party air cooler; standard SXM4 heatsinks do not fit
- **NVLink**: Status on TNS-2SXM2-4P54 with PG199 is unknown; NVLink is often absent/non-functional on PG199 outside DGX systems
- **Drivers**: Use closed-source `nvidia` driver — `nvidia-open` does **not** support GA100/PG199. Linux strongly preferred (Windows support unreliable)
- Sources: [ServeTheHome DRIVE A100 thread](https://forums.servethehome.com/index.php?threads/automotive-a100-sxm2-for-fsd-nvidia-drive-a100.43196/), [l4rz SXM guide](https://l4rz.net/running-nvidia-sxm-gpus-in-consumer-pcs/), [Léo's automotive GPU blog](https://leikoe.github.io/posts/automotive-gpu-maxxxing), [LiuXinyu12378/SXM2_to_PCIE_adapter](https://github.com/LiuXinyu12378/SXM2_to_PCIE_adapter)

## V100 SXM2 Heatsink Assembly Procedure
Labels printed on each heatsink lid (both heatsinks identical):

| Step | Order |
|---|---|
| Heat sink assembly | A → B → C → D |
| Heat sink disassembly | D → C → B → A |
| GPU screw assembly | 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 |
| GPU screw disassembly | 8 → 7 → 6 → 5 → 4 → 3 → 2 → 1 |

- Corner screw numbers on heatsink lid: **5** (top-left), **7** (top-right), **8** (bottom-left), **6** (bottom-right)
- A/B/C/D labels indicate heatsink retention bracket screw positions
- 1–8 GPU screws mount the SXM2 module die to the socket; cross-pattern tightening sequence

## NVLink Topology
- **2× V100 SXM2**: Direct NVLink connection — 150 GB/s per direction (half-duplex)
- **4+ GPUs**: Requires NVSwitch chips for full-mesh — not present on this board
- 4-GPU variant with potential NVLink scaling: **YMZX-4GPU-Q1** (different model, eBay item 298083941971)

## BIOS / Platform Requirements
| Requirement | Notes |
|---|---|
| UEFI boot | Legacy/CSM must be **disabled** |
| Above 4G Decoding | Required for high-BAR GPUs like V100 |
| Resizable BAR / Large Bar Support | Required |
| SR-IOV | Recommended |
| PCIe ARI | Recommended |
| Enhanced PCIe error reporting | Recommended where supported |

## Host Platform Compatibility
- **Recommended**: AMD Threadripper (X399 platform) or similar non-proprietary server/workstation with PCIe bifurcation support
- **Dell R740 (iDRAC9) — INCOMPATIBLE**: iDRAC9 fans ramp to 100% ~5 min after boot; thermal controls for SXM2 PCIe slot cannot be suppressed via iDRAC — not usable if noise is a concern
- PCIe bifurcation (x8/x8) required for 2-riser setup; PLX-based riser card available if motherboard lacks native bifurcation
- M.2 M-key to PCIe adapter also viable for Mini PC deployments
- **PCI Resource Exhaustion (BAR space)**: Enable **Above 4G Decoding** under Advanced → PCI Configuration. On Intel Xeon platforms: also check **IntelRCSetup** → PCIe resource allocation settings.
- **CSM**: "Above 4G Decoding" typically requires UEFI boot — verify compatibility if CSM is active.

## Real-World Performance (Ollama, 2× V100 SXM2 32 GB, AMD Threadripper 2950X)
Tested with ollama 0.18.0, script: [ollama-benchmark](https://github.com/aidatatools/ollama-benchmark)

| Model | Gen Speed (tok/s) |
|---|---|
| qwen3-coder:30b | 98.84 |
| nemotron-3-nano:30b | 122.51 |
| lfm2:24b | 138.48 |
| gpt-oss:20b | 119.43 |
| llama3.2:3b | 207.62 |

## OS & Driver Configuration

### V100 SXM2 (Volta/GV100)
- Driver: `nvidia-580xx-dkms` (last CUDA 12-supporting branch for V100)
- Do **not** use `nvidia-open` — not compatible with V100 (Volta/GV100)
- Recommended kernel parameters: `initcall_blacklist=simpledrm_platform_driver_init nvidia-drm.fbdev=0`
- Add to `/etc/modprobe.d/nvidia.conf`: `options nvidia NVreg_DynamicPowerManagement=0x02`

### PG199 DRIVE A100 (Ampere/GA100)
- Driver: use closed-source `nvidia` driver (e.g. `nvidia-dkms`) — `nvidia-open` **incompatible** with GA100/PG199
- Linux strongly preferred; Windows support for PG199 is unreliable
- Same kernel parameters as V100 apply
- May need to explicitly disable ECC for better throughput: `nvidia-smi -e 0`
- CUDA compute capability: 8.0 (vs V100 = 7.0) — broader model/tool compatibility

## Official Specifications (from product manual)

| Parameter | Value |
|---|---|
| **PCB** | FR4 black, 1 oz copper, Rev 1.2 |
| **Dimensions** | 236 × 170 mm |
| **Host Interface** | PCIe x16 (4× SFF-8654-8i) |
| **Upstream ports** | 4× SFF-8654-8i (SlimSAS x8) |
| **Downstream (GPU0)** | PCIe x16 (slot 1) or x8 (bifurcated) |
| **Downstream (GPU1)** | PCIe x16 (slot 2) or x8 (bifurcated) |
| **NVLink** | NVHS — 300 GB/s bidirectional |
| **Power connectors** | ATX 24-pin + 2× CPU 8-pin + 2× GPU 8-pin |
| **Fan headers** | 4× SFF-TA-1016 Rev1.3 (DC and PWM) |
| **Idle power** | ~5 W (board only, no GPUs) |

> Note: The blog post mentions "6× fan headers" but the official manual specifies 4× SFF-TA-1016.

## PCIe Connection Modes (from manual page 2)

Four operating modes selectable based on riser/host PCIe configuration:

| Mode | Bifurcation | Port 1 → GPU | Port 2 → GPU | Notes |
|---|---|---|---|---|
| **Internal X16** | x16 | Port 1 → GPU0 @ x8 | Port 2 → GPU1 @ x16 | Single riser, GPU1 gets full x16 |
| **External X16** | x16 | Port 1 → GPU1 @ x16 | Port 2 → GPU0 @ x8 | Single riser, GPU0 gets full x16 |
| **Internal X8** | x8/x8 | Port 1 → GPU0 @ x8 | Port 2 → GPU1 @ x8 | Both GPUs x8; use with bifurcated host |
| **External X8** | x8/x8 | Port 2 → GPU0 @ x8 | Port 1 → GPU1 @ x8 | Both GPUs x8; alternate cable routing |

## Expansion Configurations (from manual page 3)

| Config | Risers | Baseboards | Description |
|---|---|---|---|
| **Dual Card x16** | 1× RTE162P54B-2UR | 1 | Single riser → both x8 ports; each GPU gets x16 downstream |
| **Four Card x8** | 2× RTE162P54B-2UR | 2 (stacked) | Two risers → two stacked baseboards; x8/x8 bifurcation per riser |

### On-Board Switches

| Switch | Setting | Meaning |
|---|---|---|
| **Auto_SW** | AT | Host Power Follow — board powers when host PCIe power is present |
| **Auto_SW** | MT | Manual Power Control — use On_SW to control independently |
| **On_SW** | ON | GPU board power on |
| **On_SW** | OFF | GPU board power off |

### LED Indicators

| LED | Label | Meaning |
|---|---|---|
| 01 | GPU Insertion | GPU module physically seated |
| 02 | Self-Check OK | Board self-test passed |
| 03 | GPU Power On | GPU slot powered |
| 04 | GPU Overheat | Thermal alert |
| 05 | Baseboard Power | Main board rail active |
| 06 | ATX Working | ATX 24-pin rail active |
| 07 | ATX Standby | ATX 5VSB standby present |
| 08–11 | (repeat per GPU bank) | Same indicators for second GPU bank |

## Power Wiring Modes (from manual page 4)

### Mode 1 — Independent ATX (Recommended for high-wattage GPUs)

1. Connect ATX 24-pin to the baseboard
2. Connect any **3 of the 4** 8-pin connectors (CPU/GPU mix)
3. Board operates independently from host PSU

### Mode 2 — Shared with Motherboard

1. Leave ATX 24-pin **unplugged**
2. Connect **all 4** 8-pin connectors (CPU + GPU)
3. Set `Auto_SW` to **AT** (Host Power Follow)
4. Board shares host PSU; suitable for lower-wattage configurations only

### PSU Sizing Recommendations

| GPU Config | Minimum PSU | Recommended PSU |
|---|---|---|
| 2× V100 SXM2 16 GB or 32 GB | ATX3.1 850 W | Great Wall E8 850 W Gold |
| 2× PG199 (DRIVE A100) | ATX3.1 1200 W | Great Wall N12 1200 W Platinum |

> Great Wall (长城) E8/N12 series explicitly listed in TNS manual as reference PSUs.

## Known Gotchas
- SXM2 modules idle at ~42 W each (no meaningful power saving in manual mode)
- Some AI models/tools require Turing (RTX 20xx) or Ampere (RTX 30xx) — not compatible with V100
- Physical footprint is large; no rack-mount option in the kit
- "TNS" vendor still unidentified; no official documentation or support
