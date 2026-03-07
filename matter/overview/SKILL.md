---
name: overview
description: Matter (formerly Project CHIP) SDK overview covering architecture, stack layers, goals, and project structure. Use when learning about Matter, explaining the protocol stack, understanding Matter design principles, or getting started with connected home device development.
---
# Matter SDK Overview

Source: https://project-chip.github.io/connectedhomeip-doc/

## What is Matter?

Matter is a unified, open-source application-layer connectivity standard built to enable developers and device manufacturers to connect and build reliable, secure ecosystems and increase compatibility among connected home devices. It is built on Internet Protocol (IP) and is compatible with Thread and Wi-Fi network transports.

Matter was developed by the Connectivity Standards Alliance (CSA) Working Group and is royalty-free. Visit [buildwithmatter.com](https://buildwithmatter.com) for the latest news.

---

## Design Goals

| Goal | Description |
|------|-------------|
| **Unifying** | Built on top of market-tested, existing technologies |
| **Interoperable** | Communication between any Matter-certified device (subject to user permission) |
| **Secure** | Leverages modern security practices and protocols |
| **User Control** | End user controls authorization for device interactions |
| **Federated** | No single entity serves as a throttle or single point of failure for root of trust |
| **Robust** | Complete device lifecycle: out-of-box experience, operational protocols, system management |
| **Low Overhead** | Implementable on resource-constrained MCUs |
| **Pervasive** | Broadly deployable over IP, accessible on low-capability devices |
| **Ecosystem-Flexible** | Flexible enough for deployment across ecosystems with differing policies |
| **Easy to Use** | Smooth, cohesive provisioning and out-of-box experience |
| **Open** | Design and technical processes are open and transparent |

---

## Architecture Overview

Matter uses a universal IPv6-based communication protocol. The architecture is layered to separate responsibilities and provide encapsulation:

```
┌─────────────────────────────────────┐
│           Application               │  High-order business logic (e.g., lighting)
├─────────────────────────────────────┤
│           Data Model                │  Data and verb elements supporting app functionality
├─────────────────────────────────────┤
│        Interaction Model            │  Read/write/subscribe/invoke interactions
├─────────────────────────────────────┤
│         Action Framing              │  Serialization into packed binary format
├─────────────────────────────────────┤
│            Security                 │  Encryption and signing of payloads
├─────────────────────────────────────┤
│    Message Framing & Routing        │  Header fields, message properties, routing info
├─────────────────────────────────────┤
│  IP Framing & Transport Management  │  Underlying transport protocol (UDP/TCP over IP)
└─────────────────────────────────────┘
```

### Layer Descriptions

1. **Application** – High-order business logic of a device (e.g., turning on/off a bulb, controlling color).

2. **Data Model** – Data structures and verbs (commands/attributes/events) that represent the device's functionality. Applications operate on these structures.

3. **Interaction Model** – Defines interactions between client and server devices:
   - Reading or subscribing to attributes/events
   - Writing attributes
   - Invoking commands

4. **Action Framing** – Serializes Interaction Model actions into a prescribed packed binary format for network transmission.

5. **Security** – Encrypts and signs encoded frames to ensure data security and authentication.

6. **Message Framing & Routing** – Constructs the payload with required and optional header fields specifying message properties and routing information.

7. **IP Framing & Transport Management** – Passes the final payload to the underlying transport protocol for IP management.

---

## Network Transports

Matter supports two primary wireless network transports:

- **Wi-Fi** – For devices with higher bandwidth needs
- **Thread** – For low-power, mesh-networked devices

Both use IPv6 as the network layer, ensuring Matter devices can communicate across transport types.

---

## Repository Structure

The [connectedhomeip](https://github.com/project-chip/connectedhomeip) repository contains:

- **`src/`** – Core SDK source code (app layer, clusters, transport, security, etc.)
- **`examples/`** – Example applications for various device types and platforms
- **`scripts/`** – Build, test, and tooling scripts
- **`data_model/`** – CSA XML cluster definitions scraped from the Matter specification
- **`docs/`** – Documentation source

### Key Subdirectories

| Directory | Contents |
|-----------|----------|
| `src/app/clusters/` | Cluster server implementations |
| `src/controller/` | Controller (commissioner) implementation |
| `src/lib/` | Core library utilities |
| `src/platform/` | Platform abstraction layer |
| `src/transport/` | Transport layer |
| `examples/` | Example applications (lighting, lock, thermostat, etc.) |
| `scripts/py_matter_idl/` | Python tools for `.matter` IDL files |

---

## Key Concepts

### Nodes and Endpoints

- A **Node** is a uniquely addressable device on a Matter fabric.
- Each node has one or more **Endpoints** (numbered starting from 0).
- **Endpoint 0** is reserved for the root node descriptor and fabric-wide clusters (e.g., Access Control, Basic Information).

### Clusters

Clusters are the fundamental unit of device functionality in Matter, analogous to services or profiles. Each cluster has:

- **Attributes** – Readable/writable state values
- **Commands** – Invocable operations
- **Events** – Reportable occurrences

### Fabrics

A **fabric** is a shared security domain. Devices on the same fabric can communicate with each other. A device can be part of multiple fabrics simultaneously (multi-admin).

### Commissioning

Commissioning is the process of adding a device to a fabric:
1. Open commissioning window (using discriminator and passcode)
2. Establish a PASE (Password-Authenticated Session Establishment) session
3. Commissioner provisions the device with operational credentials (NOC via `AddNOC`)
4. CASE (Certificate-Authenticated Session Establishment) used for subsequent communication

---

## Build System

Matter uses [GN](https://gn.googlesource.com/gn/) (Generate Ninja) as its meta-build system with [ninja](https://ninja-build.org/) as the build executor.

```bash
# Bootstrap environment (first time or when stale)
source scripts/bootstrap.sh

# Activate environment (subsequent times)
source scripts/activate.sh

# Generate and build for host
gn gen out/host
ninja -C out/host
```

See the [Building](https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html) guide for full details.

---

## License

Matter SDK is licensed under the **Apache License 2.0**.

## References

- [Matter SDK Documentation](https://project-chip.github.io/connectedhomeip-doc/)
- [Matter SDK GitHub Repository](https://github.com/project-chip/connectedhomeip)
- [Connectivity Standards Alliance](https://csa-iot.org/)
- [Matter Specification Download](https://csa-iot.org/developer-resource/specifications-download-request/)
- [Build with Matter](https://buildwithmatter.com)
