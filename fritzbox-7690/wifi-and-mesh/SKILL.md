---
name: wifi-and-mesh
description: FRITZ!Box 7690 Wi-Fi 7 configuration, mesh networking with FRITZ! devices, guest Wi-Fi, WPS, Wi-Fi scheduling, and wireless troubleshooting. Use when setting up Wi-Fi connections, configuring the mesh network with repeaters or powerline adapters, enabling guest access, optimizing wireless performance, or troubleshooting Wi-Fi connectivity.
---
# FRITZ!Box 7690 Wi-Fi and Mesh

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 features **Wi-Fi 7** (IEEE 802.11be) with:
- **2.4 GHz** band (Wi-Fi 4/5/6/7 compatible, up to ~688 Mbit/s)
- **5 GHz** band (Wi-Fi 5/6/7 compatible, higher throughput)
- Integrated **mesh master** for centralized network management

Wi-Fi credentials (SSID and password) are printed on the device label and are pre-configured at the factory.

---

## Connecting Devices to Wi-Fi

### Manual connection
1. On your device, scan for Wi-Fi networks and select the FRITZ!Box SSID (e.g., `FRITZ!Box 7690`).
2. Enter the Wi-Fi password from the device label or from the web interface: **Wi-Fi → Wi-Fi Network**.

### Via QR code
- In the FRITZ!Box web interface at **Wi-Fi → Wi-Fi Network**, a QR code is displayed. Scan it with a smartphone camera to connect automatically.

### Via WPS (Wi-Fi Protected Setup)
1. Press the **WPS button** on the FRITZ!Box (the button with the Wi-Fi symbol).
2. Within 2 minutes, activate WPS on your device.
3. The devices connect automatically without entering a password.

---

## Wi-Fi Settings

Access at **Wi-Fi → Wi-Fi Network** in the FRITZ!Box web interface.

| Setting | Recommendation |
|---------|---------------|
| SSID | Customizable; avoid personal information |
| Wi-Fi password | Minimum 12 characters (WPA3 recommended) |
| Encryption | WPA2+WPA3 mixed for best compatibility |
| Channel | Auto (FRITZ!Box selects optimal channel) |
| Channel width | 80/160 MHz for 5 GHz (higher throughput) |
| Band steering | On – guides dual-band devices to the best band |

### Separate band SSIDs
By default, both 2.4 GHz and 5 GHz use the same SSID. To assign separate SSIDs:
- **Wi-Fi → Wi-Fi Network → Advanced Settings** → uncheck **Use the same name and password for all Wi-Fi bands**.

---

## Guest Wi-Fi

The FRITZ!Box supports a separate guest Wi-Fi network that isolates guest devices from the home network.

1. Go to **Wi-Fi → Guest Access**.
2. Enable **Guest Access**.
3. Set an SSID (e.g., `MyFRITZ-Guest`) and password.
4. Optionally enable:
   - **Time limit** – guest Wi-Fi turns off automatically after a set period.
   - **Block private LAN** – guests cannot access home network devices (recommended).
   - **Restrict access for apps** – limits apps guests can use.
5. Share the guest Wi-Fi via QR code or push the WPS button with **Guest Access** mode active.

---

## Mesh Networking with FRITZ!

The FRITZ!Box 7690 acts as the **Mesh Master** – the central control point for all FRITZ! devices in the mesh.

### Supported mesh devices
- FRITZ!Repeater (wireless or via LAN backhaul)
- FRITZ!Powerline adapters
- Additional FRITZ!Box devices (as Mesh Repeaters)

### Adding a FRITZ! device to the mesh

**Method 1: WPS**
1. Press the WPS button on the FRITZ!Box 7690.
2. Within 2 minutes, press the WPS/Connect button on the FRITZ!Repeater or FRITZ!Powerline adapter.
3. The device joins the mesh automatically and receives all Wi-Fi settings.

**Method 2: Mesh ticket (via LAN)**
1. Connect the FRITZ! device to the FRITZ!Box via a LAN cable.
2. In the FRITZ!Box web interface, go to **Home Network → Mesh**.
3. The connected device appears and can be adopted as a mesh node.

### Mesh Overview
View all mesh devices at **Home Network → Mesh**:
- Connection type (wireless/wired backhaul)
- Signal strength and throughput to each node
- Firmware update status of all nodes
- Update all mesh devices simultaneously from the Mesh Master

### Mesh steering
The mesh master uses band steering and fast roaming (802.11r/k/v) to guide devices to the best access point and band automatically.

---

## Wi-Fi Scheduling (Night Mode)

Automatically disable Wi-Fi at set times:
1. Go to **Wi-Fi → Wi-Fi Channel**.
2. Under **Wi-Fi Schedule**, add time windows when Wi-Fi should be **off**.
3. Devices with an active LAN connection remain connected during Wi-Fi off periods.

---

## FRITZ!Box as Mesh Repeater

The FRITZ!Box 7690 can also operate as a **Mesh Repeater** (wireless node) for a different FRITZ!Box acting as the Mesh Master:

1. Reset the 7690 to factory defaults or use the **Mesh Repeater** setup option.
2. Connect it to the primary FRITZ!Box via WPS or LAN cable.
3. In this mode, the 7690 takes its Wi-Fi settings from the Mesh Master.

---

## Troubleshooting Wi-Fi

| Symptom | Solution |
|---------|----------|
| Device cannot find SSID | Check Wi-Fi is enabled at **Wi-Fi → Wi-Fi Network**; verify the device supports 2.4/5 GHz |
| Slow Wi-Fi throughput | Check channel utilization; try manual channel selection; reduce interference from other networks |
| Frequent disconnections | Enable band steering; move device closer to FRITZ!Box; update device drivers |
| WPS not working | Some devices don't support WPS – use manual password connection |
| Mesh node shows offline | Check power and LAN/wireless backhaul; update firmware via **Home Network → Mesh** |

---

## References

- [Setting up a Wi-Fi connection to the FRITZ!Box (FRITZ!Box 7690)](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/244_Setting-up-a-Wi-Fi-connection-to-the-FRITZ-Box/)
- [Mesh with FRITZ!](https://en.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/3329_Mesh-with-FRITZ/)
- [Setting up the FRITZ!Box as a Mesh Repeater](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/3354_Setting-up-the-FRITZ-Box-as-a-Mesh-Repeater/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
