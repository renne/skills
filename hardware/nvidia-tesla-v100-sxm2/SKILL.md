---
name: nvidia-tesla-v100-sxm2
description: NVIDIA Tesla V100 SXM2 16 GB compute GPU. Hardware specs, VBIOS, driver installation (Ubuntu/CUDA), testing procedures, NVLink, PCIe requirements, and homelab integration via TNS-2SXM2-4P54 baseboard on Machinist MRA9. Use this skill when working with V100 SXM2 GPUs, CUDA compute, GPU passthrough, or NVIDIA data-centre driver setup.
---

# NVIDIA Tesla V100 SXM2 16 GB

## Hardware Specifications

| Field | Value |
|---|---|
| **Full name** | NVIDIA Tesla V100 SXM2 16 GB |
| **PCI Device ID** | `10de:1db1` (GV100GL) |
| **PCI Subsystem** | `10de:1212` |
| **GPU die** | GV100 (chip ID `140000a1`, rev `a1`) |
| **Architecture** | Volta (sm_70) |
| **VRAM** | 16 384 MiB HBM2 (4 stacks × 4 GB) |
| **Memory bandwidth** | 900 GB/s |
| **FP64 performance** | 7.8 TFLOPS |
| **FP32 performance** | 15.7 TFLOPS |
| **FP16 (Tensor Core)** | 125 TFLOPS |
| **Tensor Cores** | 640 (1st-gen) |
| **CUDA cores** | 5 120 |
| **Form factor** | SXM2 (board-edge socket, not PCIe add-in card) |
| **TDP** | 300 W |
| **NVLink** | NVLink 2.0 — 6 links × 25 GB/s = 300 GB/s bidirectional (requires NVLink bridge) |
| **PCIe interface** | PCIe 3.0 x16 via SXM2 baseboard |
| **VBIOS version** | `88.00.8D.00.03` (copyright 1996–2019 NVIDIA) |
| **VBIOS ROM size** | 268 800 bytes (262.5 KiB) |
| **Max payload size** | 256 bytes |
| **Max read request** | 512 bytes |

## Confirmed Live System State (mra9, 2026-03-22)

| GPU | PCI BDF | Physical Slot | IOMMU Group | PCIe Link | Idle Temp |
|---|---|---|---|---|---|
| GPU 0 | `03:00.0` | Slot 6 (baseboard label) | 29 | Gen3 x16 @ 8 GT/s | 29 °C |
| GPU 1 | `04:00.0` | Slot 4 (baseboard label) | 30 | Gen3 x16 @ 8 GT/s | 31 °C |

Both GPUs occupy **separate IOMMU groups** (no shared upstream devices) — ideal for VFIO GPU passthrough.

Memory regions (Above 4G Decoding active):

| GPU | BAR0 (32-bit, 16 MiB) | BAR1 (64-bit, 16 GiB, prefetchable) | BAR3 (64-bit, 32 MiB, prefetchable) |
|---|---|---|---|
| GPU 0 | `fa000000` | `383800000000` | `383c00000000` |
| GPU 1 | `f9000000` | `383000000000` | `383400000000` |

## Physical Installation

The V100 SXM2 modules are installed in a **TNS-2SXM2-4P54** dual-SXM2 carrier baseboard (see `hardware/tns-2sxm2-4p54/SKILL.md`), connected to the host via **2× RTE162P54B-2UR** PCIe risers (see `hardware/rte162p54b-2ur/SKILL.md`).

The complete stack on **Machinist MRA9 v1.0** is:

```
V100 SXM2 ×2
    └── TNS-2SXM2-4P54 baseboard  (4× SFF-8654-8i / SlimSAS x8)
            └── 2× RTE162P54B-2UR risers  (SFF-8654-8i → PCIe x16 slot)
                    └── MRA9 PCIe x16 slots (CPU-attached, PCIE slots 4 & 6)
```

## Required BIOS Settings (Machinist MRA9 / AMI X99MA011)

All three settings **must** be configured before the V100s will enumerate correctly. See `hardware/machinist-mra9-v1/SKILL.md` for the full navigation paths.

| Setting | Required Value | BIOS Path |
|---|---|---|
| Above 4G Decoding | **Enabled** | `Advanced → PCI Subsystem Settings → Above 4G Decoding` |
| CSM Support | **Disabled** | `Advanced → CSM Configuration → CSM Support` |
| Intel VT-d | **Enabled** (for IOMMU/passthrough) | `Chipset → IIO Configuration → Intel VT for Directed I/O (VT-d)` |

> Without **Above 4G Decoding**, the 16 GiB BAR1 cannot be mapped and the card will fail to initialise ("PCI Out of Resources").  
> Without **CSM Disabled**, the display card (GT-710, now flashed with UEFI VBIOS) may revert to legacy mode.

## ⚠️ CRITICAL: Nouveau Unbind Causes Kernel Oops on V100 SXM2

**NEVER attempt to unbind the V100 from nouveau at runtime.** Calling `echo BDF > /sys/bus/pci/drivers/nouveau/unbind` on a V100 SXM2 triggers a kernel oops in `nve0_bo_move_copy` (nouveau's buffer-object DMA move path), corrupts V100 MMIO state, and may hang SSH. The system requires a full reboot to recover.

**Root cause:** nouveau's `nve0_bo_move_copy` fires during device teardown; GV100 DMA descriptors in an active state cause a kernel NULL-pointer dereference when the nouveau drm driver tries to clean up in-flight BOs.

### Mandatory driver sequence — installed Ubuntu (safe, correct order)

1. **Blacklist nouveau in initramfs BEFORE first boot with nvidia driver:**
   ```bash
   echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
   sudo update-initramfs -u    # bakes blacklist into initrd — nouveau never loads
   sudo reboot
   ```
2. After reboot, confirm nouveau is not loaded: `lsmod | grep nouveau` → empty
3. Install driver (driver will load cleanly because nouveau never claimed the device)
4. `nvidia-smi` → both GPUs present

### Live-system / Ubuntu Desktop Live (problematic — do NOT attempt)

On Ubuntu Live (casper/squashfs), `update-initramfs` is a no-op (`running on read-only media`). Nouveau loads at boot and claims the V100s before any blacklist takes effect. **Attempting runtime unbind will kernel-oops.** Use a fully installed Ubuntu instead.

### Emergency recovery if V100 MMIO corrupts (SBR via CF8/CFC I/O ports)

If MMIO becomes corrupt (nvidia module hangs, `setpci` via MMCONFIG deadlocks), use the legacy Intel CF8/CFC I/O port path to issue a Secondary Bus Reset:

```bash
# V100 #1 is downstream of root port 00:02.0, V100 #2 downstream of 00:03.0
# Assert SBR (bit 6 of Bridge Control register 0x3e)
setpci -A intel-conf1 -s 00:02.0 3e.w=0052; sleep 0.1; setpci -A intel-conf1 -s 00:02.0 3e.w=0012
setpci -A intel-conf1 -s 00:03.0 3e.w=0052; sleep 0.1; setpci -A intel-conf1 -s 00:03.0 3e.w=0012
```

**MUST use `-A intel-conf1`** — regular MMCONFIG (`-A mmio`) hangs when V100 MMIO is already corrupt. `intel-conf1` uses legacy CF8/CFC I/O ports which bypass MMCONFIG.

## Driver Installation (Ubuntu 24.04 LTS — Installed System)

The proprietary NVIDIA driver is required for all compute workloads. Nouveau has very limited support for GV100 (no power management, no CUDA, no PMU firmware).

### Install NVIDIA Data Centre Driver

```bash
# Step 1: Blacklist nouveau and bake into initrd BEFORE installing driver
echo -e "blacklist nouveau\noptions nouveau modeset=0" | sudo tee /etc/modprobe.d/blacklist-nouveau.conf
sudo update-initramfs -u

# Step 2: Install server driver (Ubuntu package — confirmed working: 570.211.01)
sudo apt update
sudo apt install -y nvidia-driver-570-server

# Reboot to activate — nouveau will NOT load, nvidia claims V100s cleanly
sudo reboot
```

### Install CUDA Toolkit

```bash
# CUDA 12.x via network installer (recommended)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
sudo apt install -y cuda-toolkit-12-6

# Add to PATH
echo 'export PATH=/usr/local/cuda/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

### Verify Installation

```bash
nvidia-smi
# Expected: two GV100 devices, 16 160 MiB free VRAM each, temp ~30–40 °C idle

nvidia-smi -q | grep -E 'Product|Serial|VBIOS|UUID|Temp|Power|P-State'

# List CUDA devices
nvidia-smi -L
```

## Health Check & Diagnostics

### Quick system checks (no driver needed)

```bash
# Confirm both GPUs enumerate
lspci -d 10de:1db1

# Check PCIe link speed (must be Gen3 x16)
sudo lspci -vvnn -d 10de:1db1 | grep -E 'LnkSta:|LnkCap:'

# Idle temperatures via nouveau hwmon (millidegrees Celsius)
cat /sys/bus/pci/devices/0000:03:00.0/hwmon/hwmon*/temp1_input   # GPU 0
cat /sys/bus/pci/devices/0000:04:00.0/hwmon/hwmon*/temp1_input   # GPU 1

# AER error counters
sudo lspci -vvnn -d 10de:1db1 | grep -E 'CESta|UESta|DevSta'
# DevSta: CorrErr+ is normal (can occur during enumeration with nouveau)
```

### Post-driver tests (requires NVIDIA driver + CUDA)

```bash
# 1. Basic GPU info
nvidia-smi -q

# 2. Memory bandwidth (built into CUDA samples)
/usr/local/cuda/samples/1_Utilities/bandwidthTest/bandwidthTest

# 3. Device query
/usr/local/cuda/samples/1_Utilities/deviceQuery/deviceQuery

# 4. GPU burn-in (Volta native PTX — MUST override arch)
sudo apt install -y git build-essential
git clone https://github.com/wilicc/gpu-burn
cd gpu-burn
# Default Makefile uses -arch=compute_75 (Turing); override for Volta sm_70 native PTX:
sed -i 's/-arch=compute_75/-arch=sm_70/' Makefile
make
sudo ./gpu_burn 300   # 300-second stress; watch temps and errors

# 5. VRAM memtest (all tests, multiple passes)
git clone https://github.com/ComputationalRadiationPhysics/cuda_memtest
cd cuda_memtest && cmake -DCMAKE_CUDA_ARCHITECTURES=70 . && make -j$(nproc)
# Run BOTH GPUs in parallel — must use sudo and CUDA_VISIBLE_DEVICES (--gpu_num flag absent)
sudo CUDA_VISIBLE_DEVICES=0 ./cuda_memtest --num_passes 3 &
sudo CUDA_VISIBLE_DEVICES=1 ./cuda_memtest --num_passes 3 &
wait
# ⚠️ DO NOT kill these with SIGKILL — see Xid 74 warning below

# 6. NCCL all-reduce bandwidth test (NVLink validation)
# See "NCCL Compatibility" section for build instructions and version requirements
sudo /tmp/nccl-tests/build/all_reduce_perf -g 2 -b 8 -e 4G -f 2
# Expected busbw: ~50 GB/s (NVLink active), ~12 GB/s (PCIe fallback = NVLink broken)

# 7. ECC error check
nvidia-smi --query-gpu=ecc.errors.corrected.aggregate.total,ecc.errors.uncorrected.aggregate.total --format=csv
```

### Expected healthy values

| Metric | Expected |
|---|---|
| Idle temperature | 25–45 °C |
| Max load temperature | ≤ 83 °C (throttle threshold 87 °C, shutdown 90 °C) |
| PCIe link | Gen3 x16 @ 8 GT/s |
| ECC correctable errors | Low / zero on new run |
| ECC uncorrectable errors | **0** (any UE = card fault) |
| gpu-burn FP64 | ~7.8 TFLOPS per card |
| VRAM available | 16 160 MiB (16 384 MiB − driver overhead) |
| NVLink per-link bandwidth | 25.781 GB/s (all 6 links per GPU) |
| NCCL all_reduce busbw | ~50 GB/s (NVLink active) |

### Confirmed Test Results (mra9 pre-move validation, 2026-03-22)

| Test | Result | Notes |
|---|---|---|
| FP64 burn (gpu-burn, 300 s) | ✅ PASSED | Peak 83/82 °C; no throttling (threshold 87 °C); ECC 0 post-burn |
| VRAM memtest (cuda_memtest, 3 passes) | ✅ PASSED | Zero errors; temps ~41–45 °C during test |
| NVLink link status | ✅ PASSED | All 6 links on both GPUs at 25.781 GB/s post-reboot |
| NCCL all_reduce | ⏳ PENDING | Blocked by NCCL/nvcc version mismatch — see NCCL section |

## ⚠️ Xid 74 — NVLink Fatal Error (SIGKILL on cuda_memtest)

**Trigger:** Abruptly killing `cuda_memtest` processes with SIGKILL (or sending SIGTERM that escalates to kill) while they hold active NVLink state causes:
- **Xid 74** on GPU0 `03:00.0`: `NVLink: fatal error detected on link 4` (error code `0x10000`)
- **Xid 62** on GPU1 `04:00.0`: Internal micro-controller halt (NVLink cascade from GPU0)
- Both `nvidia-smi` and kernel threads enter D-state (uninterruptible sleep); SSH freezes

**Recovery:** **Hard power cycle only.** There is no software recovery path once Xid 74 fires. `SysRq+B` is the last resort if console is accessible. Reboot is clean — post-cycle `dmesg` shows no Xid errors and NVLink returns to 25.781 GB/s.

**Prevention:**
- Let `cuda_memtest` finish its passes naturally
- If you must interrupt, use `SIGINT` (Ctrl+C) and wait for graceful teardown
- Do **NOT** `kill -9` GPU compute processes that hold NVLink state

**GPU serial numbers:**
- GPU0 `03:00.0`: `1321920049451`
- GPU1 `04:00.0`: `1321620017216`

## NVLink

The V100 SXM2 supports **NVLink 2.0** (6 links, 300 GB/s bidirectional) but NVLink requires a **NVLink bridge** connecting both SXM2 sockets on the TNS baseboard. Without the bridge, GPUs communicate over PCIe only (~32 GB/s bidirectional through the MRA9 CPU DMI).

NVLink bridge P/N for dual-V100 SXM2: `NVIDIA NVLink Bridge for Tesla V100 SXM2` (part of NVLink SXM2 accessories kit). The TNS-2SXM2-4P54 baseboard has pads for an NVLink bridge connector between the two SXM2 sockets.

To check NVLink status after driver install:
```bash
nvidia-smi nvlink --status -i 0
nvidia-smi nvlink --capabilities -i 0
# Healthy output: all 6 links "Active" at 25.781 GB/s
```

## NCCL Compatibility — Driver 580.x / CUDA 13.0

### NCCL 2.18 + CUDA 13.0 driver (580.x) — BROKEN

NCCL 2.18.5 calls `cuStreamBatchMemOp` (`misc/strongstream.cc:60`) which was **removed/stubbed** in the CUDA 13.0 driver (580.x). Symptoms:
- NCCL init succeeds fully (12 P2P/direct pointer channels via NVLink are established and logged)
- First `ncclAllReduce` call fails: `CUDA driver is a stub library`
- Disabling NVLS (`NCCL_NVLS_ENABLE=0`) does NOT help — different code path

### Fix: Upgrade to NCCL 2.29.7

```bash
# Add NVIDIA CUDA apt repo (if not already present)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install NCCL 2.29.7 (compatible with CUDA 13.0 driver)
sudo apt install -y libnccl2=2.29.7-1+cuda12.9 libnccl-dev=2.29.7-1+cuda12.9
```

### nccl-tests build — nvcc version requirement

**NCCL 2.29.7 device headers require nvcc ≥ 12.2.** The Ubuntu 24.04 repo only ships `nvidia-cuda-toolkit` 12.0.140. Building nccl-tests against NCCL 2.29.7 with nvcc 12.0 fails in `nccl_device/impl/vector__funcs.h` with:

```
error: identifier "__bfloat1622float2" is undefined
error: identifier "__float22bfloat162_rn" is undefined
error: no suitable user-defined conversion from "__nv_bfloat162" to "__half2" exists
```

**Fix options:**
1. Install newer CUDA toolkit from NVIDIA repo: `sudo apt install -y cuda-toolkit-12-8` (recommended)
2. Downgrade NCCL to last version compatible with nvcc 12.0 (≈ 2.20.x)

### nccl-tests build command

```bash
# After clean source + correct NCCL + nvcc ≥ 12.2:
cd /tmp/nccl-tests
make clean
make MPI=0 CUDA_HOME=/usr -j$(nproc)

# Run all-reduce bandwidth test
sudo ./build/all_reduce_perf -g 2 -b 8 -e 4G -f 2
# Expected: busbw ~50 GB/s (NVLink); ~12 GB/s = NVLink disabled/broken
```

### ABI mismatch warning

Running nccl-tests binary compiled against NCCL 2.18 headers against NCCL 2.29.7 library produces `misaligned address` in the validation step. Always `make clean` before rebuilding after a NCCL library upgrade.

## VFIO / GPU Passthrough

Each V100 is in its own IOMMU group (29 and 30) with no other devices — ideal for passthrough to a VM.

```bash
# Bind a GPU to vfio-pci (example for GPU 0)
echo "10de 1db1" | sudo tee /sys/bus/pci/drivers/vfio-pci/new_id

# Or by BDF:
echo 0000:03:00.0 | sudo tee /sys/bus/pci/devices/0000:03:00.0/driver/unbind
echo 0000:03:00.0 | sudo tee /sys/bus/pci/drivers/vfio-pci/bind

# Verify
ls -la /sys/bus/pci/devices/0000:03:00.0/driver   # should point to vfio-pci
```

## Validated Test Results (mra9, 2026-03-22)

### FP64 Burn Test — PASSED

- Tool: `wilicc/gpu-burn` compiled with `-arch=sm_70` (Volta native PTX)
- Duration: 300 seconds, both GPUs simultaneously
- Result: exit 0, 0 errors, both GPUs at 100% utilisation
- Peak temps: 83 °C (GPU0) / 82 °C (GPU1) — 4–5 °C below throttle threshold (87 °C)
- No thermal throttling observed; post-burn ECC uncorrected = 0/0

### VRAM Memtest — 3 Passes PASSED

- Tool: `ComputationalRadiationPhysics/cuda_memtest` built with `-DCMAKE_CUDA_ARCHITECTURES=70`
- Ran both GPUs in parallel: `sudo CUDA_VISIBLE_DEVICES=0 ./cuda_memtest` and `sudo CUDA_VISIBLE_DEVICES=1 ./cuda_memtest`
- 3 full passes (Test0–Test8, Test10; Test9 skipped by tool), zero errors on both GPUs
- Temps during memtest: ~41–45 °C; throttle code 0x0 throughout

> ⚠️ **SIGKILL after memtest triggers Xid 74 — see below**

### NVLink Status — CONFIRMED ACTIVE

```
nvidia-smi nvlink --status -i 0
```
All 6 links on both GPUs at **25.781 GB/s** ✅

GPU serial numbers:
- GPU0 `03:00.0`: `1321920049451`
- GPU1 `04:00.0`: `1321620017216`

## Known Issues / Quirks

| Issue | Details |
|---|---|
| `pmu: firmware unavailable` (nouveau) | Normal — nouveau cannot load NVIDIA PMU firmware for GV100; harmless with nouveau, requires NVIDIA proprietary driver for full function |
| `BIOS Certificate Check Failed!!!` (nouveau) | Normal warning from nouveau during VBIOS parse — not a hardware fault |
| `DevSta: CorrErr+` | Correctable PCIe errors can accumulate at enumeration time with nouveau; reset after driver load |
| VPD not readable | `sysfs_read_vpd: read failed: No such device` — V100 SXM2 does not expose VPD via standard PCIe VPD capability |
| No serial number via lspci | Serials accessible only via `nvidia-smi -q` after driver load |
| pstate empty (nouveau) | Nouveau has no power-state management for GV100; driver required |
| No display output | V100 SXM2 has no video output — always pair with a display card (GT-710 on this system) |
| SXM2 ≠ PCIe card | Cannot be used in a standard PCIe x16 slot; requires SXM2 baseboard (TNS-2SXM2-4P54 or similar) |

## ⚠️ Xid 74 / Xid 62 — NVLink Fatal Error After SIGKILL

### Trigger

Abruptly killing `cuda_memtest` (or any CUDA process doing active DMA over NVLink) with SIGKILL causes:

- **Xid 74** on GPU0 (`03:00.0`): `NVLink: fatal error detected on link 4` (error code `0x10000`)
- **Xid 62** on GPU1 (`04:00.0`): Internal micro-controller halt — secondary cascade from GPU0's NVLink fault

`dmesg` output:
```
NVRM: GPU Board Serial Number: [N/A]
NVRM: Xid (PCI:0000:03:00): 74, pid=..., NVLink: Fatal error detected on link 4
NVRM: Xid (PCI:0000:04:00): 62, pid=0, 09d4 00000000 ...
```

After Xid 74, `nvidia-smi` hangs indefinitely in D-state. **SSH also hangs.** The system is completely frozen.

### Recovery

**Only recovery: hard power cycle.** Cold boot (power off → power on).

- After hard power cycle: dmesg is clean — **no Xid 74, no NVLink errors**
- `nvidia-smi` health: ECC uncorrected = 0, temps nominal, throttle code `0x1` (idle/normal)
- NVLink status: all 12 lanes back at 25.781 GB/s ✅
- No hardware damage from the Xid 74 event itself

### Prevention

Terminate CUDA processes **gracefully** (SIGTERM / wait for clean exit) rather than SIGKILL. When force-stopping cuda_memtest, use `kill -SIGINT <PID>` and wait for the process to exit cleanly before removing the driver or rebooting.

## NCCL — Driver 580.x / CUDA 13.0 Incompatibility

### NCCL 2.18.x + driver 580.x fails at runtime

NCCL 2.18 calls `cuStreamBatchMemOp` internally (`misc/strongstream.cc:60`). This API was **removed/stubbed** in the CUDA 13.0 driver (580.x series). NCCL init succeeds completely (12 P2P/direct pointer NVLink channels confirmed), but the first `ncclAllReduce` enqueue fails:

```
NCCL WARN misc/strongstream.cc:60 CUDA failure 'driver is a stub library'
```

Disabling NVLS (`NCCL_NVLS_ENABLE=0`) does **not** help.

### Fix: upgrade NCCL to 2.29.7+

```bash
# Add NVIDIA official CUDA apt repo (if not already present)
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update

# Install NCCL 2.29.7 (compatible with CUDA 13.0 / driver 580.x)
sudo apt install -y libnccl2=2.29.7-1+cuda12.9 libnccl-dev=2.29.7-1+cuda12.9
```

### nccl-tests rebuild requires nvcc ≥ 12.2

After upgrading to NCCL 2.29.7, rebuilding `NVIDIA/nccl-tests` with Ubuntu 24.04's bundled `nvidia-cuda-toolkit` (12.0.140) **fails** with errors in `nccl_device/impl/vector__funcs.h`:

```
error: identifier "__bfloat1622float2" is undefined
error: identifier "__float22bfloat162_rn" is undefined
error: no suitable user-defined conversion from "__nv_bfloat162" to "__half2" exists
```

**Root cause:** NCCL 2.29.7 device headers use bfloat16 intrinsics (`__bfloat1622float2`, `__float22bfloat162_rn`) that require nvcc ≥ 12.2. Ubuntu 24.04 repos only ship 12.0.140.

**Fix:** Install CUDA toolkit 12.2+ from the NVIDIA CUDA apt repo:
```bash
sudo apt install -y cuda-toolkit-12-8   # or 12-6, 12-4 — any ≥ 12.2
export CUDA_HOME=/usr/local/cuda
make clean && make MPI=0 CUDA_HOME=$CUDA_HOME -j$(nproc)
```

### ABI mismatch warning

Running an nccl-tests binary compiled against NCCL 2.18 headers against the 2.29.7 library produces a `misaligned address` in validation. Always `make clean` before rebuilding after an NCCL library upgrade.

### Expected healthy NVLink bandwidth

| Test | Expected busbw | NVLink broken indicator |
|---|---|---|
| `all_reduce_perf -g 2` | ≥ 50 GB/s | ~12 GB/s (PCIe only) |

## Useful nvidia-smi One-Liners

```bash
# Continuous monitoring (1s refresh)
nvidia-smi dmon -s pucvmet

# Power limit (default 300 W; can be reduced for efficiency)
nvidia-smi -pl 250   # set 250 W limit

# Enable persistence mode (keeps driver loaded between jobs)
nvidia-smi -pm 1

# Reset GPU (clears correctable errors counter)
nvidia-smi --gpu-reset -i 0

# ECC mode (enabled by default on Tesla cards)
nvidia-smi --ecc-config=1 -i 0   # enable
nvidia-smi --ecc-config=0 -i 0   # disable (needs reboot)
```
