---
name: building
description: Building the Matter (formerly Project CHIP) SDK from source, including environment setup, prerequisites, GN/ninja build system, ZAP tool installation, and cross-platform compilation with build_examples.py. Use when setting up a Matter SDK development environment, compiling Matter examples, or troubleshooting build issues.
---
# Building Matter

Source: https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html

## Overview

Matter uses [GN](https://gn.googlesource.com/gn/) as its meta-build system and [ninja](https://ninja-build.org/) as the build executor. Both are downloaded automatically when you bootstrap the environment.

**Tested Operating Systems:**
- macOS 10.15+
- Debian 11 (64-bit required)
- Ubuntu 22.04 LTS (requires Python 3.11+)
- Ubuntu 24.04 LTS

---

## Checking Out the Code

### All Platforms (Recommended)

```bash
git clone --recurse-submodules git@github.com:project-chip/connectedhomeip.git
```

### Specific Platforms (Reduces Size)

```bash
# Shallow clone the top-level repo
git clone --depth=1 git@github.com:project-chip/connectedhomeip.git

# Check out only specific platform submodules
python3 scripts/checkout_submodules.py --shallow --platform linux
# or
python3 scripts/checkout_submodules.py --shallow --platform darwin
# or multiple platforms
python3 scripts/checkout_submodules.py --shallow --platform linux,esp32,nrf
```

### Updating an Existing Checkout

```bash
git pull
git submodule update --init
```

---

## Prerequisites

### Linux (Debian/Ubuntu)

```bash
sudo apt-get install git gcc g++ pkg-config cmake curl libssl-dev libdbus-1-dev \
     libglib2.0-dev libavahi-client-dev ninja-build python3-venv python3-dev \
     python3-pip unzip libgirepository1.0-dev libcairo2-dev libreadline-dev \
     libevent-dev default-jre
```

#### Python 3.11 on Ubuntu 22.04

Ubuntu 22.04 ships Python 3.10; Matter SDK requires Python 3.11+:

```bash
sudo apt-get install python3.11 python3.11-dev python3.11-venv
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.10 1
sudo update-alternatives --install /usr/bin/python3 python3 /usr/bin/python3.11 2
python3 --version  # Should print 3.11.x
```

#### Optional: NFC builds

```bash
sudo apt-get install libpcsclite-dev
```

#### Optional: UI builds (SDL2)

```bash
sudo apt-get install libsdl2-dev
```

### macOS

Install Xcode from the Mac App Store.

```bash
# Optional: for -with-ui builds
brew install sdl2
```

### Raspberry Pi 4

1. Install Ubuntu 22.04 64-bit server OS using `rpi-imager`
2. Follow Linux prerequisites above
3. Install Raspberry Pi-specific packages:
   ```bash
   sudo apt-get install bluez pi-bluetooth avahi-utils
   ```
4. Enable experimental Bluetooth and disable battery plugin:
   ```bash
   sudo systemctl edit bluetooth.service
   # Add:
   [Service]
   ExecStart=
   ExecStart=/usr/libexec/bluetooth/bluetoothd -E -P battery
   
   sudo systemctl restart bluetooth.service
   ```

---

## Environment Setup

### Bootstrap (First Time or When Stale)

```bash
source scripts/bootstrap.sh
```

This downloads GN, ninja, and sets up a Python virtual environment. It can take several minutes.

### Activate Environment (Subsequent Sessions)

```bash
source scripts/activate.sh
```

Always run this before building. If the environment is out of date, it will prompt you to re-bootstrap.

---

## Building for the Host (Linux or macOS)

```bash
source scripts/activate.sh

# Generate build files (debug by default)
gn gen out/host

# Build
ninja -C out/host

# Optimized build
gn gen out/host --args='is_debug=false'
ninja -C out/host
```

> **Tip:** The `out/host` directory is a convention. Any directory works, and multiple build directories can coexist for different configurations.

### Run All Tests

```bash
ninja -C out/host check
```

### Run Tests in a Specific Directory

```bash
ninja -C out/host src/inet/tests:tests_run
```

### Run a Single Unit Test

```bash
# Build and run a specific test
ninja -C out/debug mac_arm64_gcc/tests/TestSessionManagerDispatch

# Or enter build directory and run
cd out/debug
ninja linux_x64_clang/phony/src/transport/tests/TestSessionManagerDispatch.run
```

---

## Using `build_examples.py`

The `scripts/build/build_examples.py` script provides a uniform interface to build for any supported platform:

```bash
# List all supported build targets
./scripts/build/build_examples.py targets

# Build and run all tests on host
./scripts/build/build_examples.py --target linux-x64-tests build

# Build an ESP32 example
./scripts/build/build_examples.py --target esp32-m5stack-all-clusters build

# Build an nRF example
./scripts/build/build_examples.py --target nrf-nrf5340dk-pump build

# Build with clang and AddressSanitizer (for fuzzing)
./scripts/build/build_examples.py --target linux-x64-tests-clang-asan-libfuzzer build
```

---

## ZAP Tool

The ZAP tool is used to edit cluster configuration (`.zap`) files and drive code generation.

### Automatic Installation (Recommended)

ZAP is installed automatically via CIPD when you run `scripts/bootstrap.sh`. It becomes available as `zap-cli` in `$PATH` within the activated environment.

### Custom ZAP Installation

Download from [ZAP releases](https://github.com/project-chip/zap/releases) or build from source:

```bash
mkdir -p /opt/zap-${ZAP_VERSION}
git clone https://github.com/project-chip/zap.git /opt/zap-${ZAP_VERSION}
cd /opt/zap-${ZAP_VERSION}
git checkout ${ZAP_VERSION}
npm config set user 0
npm ci
export ZAP_DEVELOPMENT_PATH=/opt/zap-${ZAP_VERSION}
```

### ZAP Environment Variables (in order of precedence)

| Variable | Description |
|----------|-------------|
| `$ZAP_DEVELOPMENT_PATH` | Path to ZAP source checkout (for ZAP development) |
| `$ZAP_INSTALL_PATH` | Path where ZAP release zip was unpacked |
| *(none)* | Scripts assume `zap-cli`/`zap` is in `$PATH` |

### Using the ZAP UI

```bash
# Edit a .zap file with the ZAP GUI
./scripts/tools/zap/run_zaptool.sh examples/lighting-app/lighting-common/lighting-app.zap
```

---

## Common Build Configurations

| Target | Command |
|--------|---------|
| Linux x64 (host) | `gn gen out/host && ninja -C out/host` |
| Linux x64 tests | `./scripts/build/build_examples.py --target linux-x64-tests build` |
| ESP32 all-clusters | `./scripts/build/build_examples.py --target esp32-m5stack-all-clusters build` |
| nRF Connect SDK | `./scripts/build/build_examples.py --target nrf-nrf5340dk-pump build` |
| Raspberry Pi | `./scripts/build/build_examples.py --target linux-arm64-all-clusters build` |

---

## Introspection and Formatting

```bash
# Describe a build target
gn desc out/host //src/lib/core:core

# List all targets
gn ls out/host

# Format GN build files
gn format path/to/BUILD.gn
```

## References

- [Building Matter Guide](https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html)
- [ZAP Releases](https://github.com/project-chip/zap/releases)
- [GN Documentation](https://gn.googlesource.com/gn/)
- [Ninja Build System](https://ninja-build.org/)
