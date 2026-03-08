---
name: internet-setup
description: FRITZ!Box 7690 internet connection setup for DSL, VDSL, supervectoring, FTTH/fiber (via ONT), cable (via external modem), and cascaded router scenarios. Use when configuring the internet connection, troubleshooting internet access, setting up provider-specific parameters (PPPoE, IPoE, VLAN), or connecting the FRITZ!Box behind another router.
---
# FRITZ!Box 7690 Internet Setup

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 is a combination modem and router that supports:
- **DSL/VDSL/supervectoring** – up to 300 Mbit/s downstream via the built-in DSL modem
- **FTTH/fiber** – connected via an external ONT (optical network terminal) to the WAN or LAN 1 port
- **External modem / cable / LTE** – router-only mode behind any upstream device
- **Cascading** – behind another router (double NAT or IP client mode)

Access the FRITZ!Box web interface at **http://fritz.box** (or 192.168.178.1) and log in with the FRITZ!Box password (printed on the device label).

---

## DSL / VDSL / Supervectoring Setup

1. Connect the DSL cable from the **DSL** port on the FRITZ!Box to the telephone wall socket (TAE-F in Germany, or via a DSL splitter).
2. Connect the power supply and wait for the FRITZ!Box to boot (Power LED solid).
3. In the web interface, open the setup wizard: **Wizards → Configure the Internet Connection**.
4. Select your country and ISP from the list. If not listed, choose **Other Internet Service Provider**.
5. Enter the DSL account credentials (username/password) provided by your ISP for PPPoE connections.
6. For **VDSL2 vectoring/supervectoring** the FRITZ!Box handles the modem automatically; no extra settings are needed.
7. Click **Apply**. The FRITZ!Box attempts to establish the connection. A successful connection is confirmed by a solid **Internet** LED.

### Manual entry (Internet → Account Information)

| Field | Notes |
|-------|-------|
| Connection type | PPPoE (most DSL providers) or IPoE |
| Username | Provided by ISP (e.g., `12345678901@t-online.de`) |
| Password | DSL account password |
| VLAN ID | Required by some ISPs (e.g., Telekom: VLAN 7) |

---

## FTTH / Fiber (via ONT)

The FRITZ!Box 7690 does **not** have a built-in fiber port. It connects to the ISP's ONT:

1. Connect a LAN cable from the ONT's Ethernet port to the **WAN** port (or LAN 1, if no dedicated WAN port is used) on the FRITZ!Box.
2. In the web interface: **Internet → Account Information → Connection Settings**.
3. Set the **Internet connection via** to **WAN port** (or the correct LAN port).
4. Choose your connection type:
   - **PPPoE** – enter username and password from your ISP.
   - **IPoE (DHCP)** – the FRITZ!Box gets an IP address automatically.
5. Some fiber ISPs require a **VLAN tag** on the WAN interface. Set **Internet connection VLAN ID** accordingly.
6. The **2.5 Gigabit LAN port** on the FRITZ!Box 7690 supports up to 2.5 Gbit/s for multi-gigabit fiber connections.

---

## External Modem / Cable / LTE (Router Mode)

Use this when the FRITZ!Box operates behind a cable modem, LTE router, or any other upstream device:

1. Connect a LAN cable from the upstream device to the **WAN** port of the FRITZ!Box.
2. In the web interface: **Internet → Account Information**.
3. Set connection type to **External modem or router**.
4. The FRITZ!Box receives an IP address from the upstream device via DHCP.

---

## Cascading Router (IP Client Mode)

If another router already provides DHCP and NAT for your network, the FRITZ!Box can operate in **IP client mode**:

1. **Internet → Account Information → Connection Settings** → set to **IP client (without NAT)** or connect as an IP client.
2. The FRITZ!Box receives a private IP from the upstream router and acts as a bridge/switch + Wi-Fi access point.
3. Note: In IP client mode, the FRITZ!Box firewall, parental controls, and VPN are not available.

For VPN access through a cascaded router, refer to the **VPN** skill.

---

## Internet Failover (FRITZ! Failsafe)

FRITZ!OS 8 introduced automatic internet failover (FRITZ! Failsafe):
- If the primary connection fails, the FRITZ!Box automatically switches to a backup connection (e.g., LTE via USB dongle or secondary DSL line).
- Configure under **Internet → Account Information → Backup Connection**.

---

## Online Monitor

View real-time bandwidth usage per device:
- **Overview → Internet** shows the online monitor.
- FRITZ!OS 8.20+ displays a graphical view with top bandwidth-consuming devices.

---

## Troubleshooting

| Symptom | Check |
|---------|-------|
| No internet connection | Verify DSL cable, ISP credentials, line activation |
| Wrong VLAN | Check ISP documentation for VLAN ID requirement |
| Slow VDSL speeds | Ensure supervectoring profile is active; check DSL signal stats under **Internet → DSL Information** |
| FTTH: No IP address | Confirm ONT is powered and link is up; try IPoE if PPPoE fails |
| Double NAT (cascading) | Consider IP client mode or DMZ on the upstream router |

---

## Provider-Specific Setup Guides

AVM provides dedicated step-by-step setup guides for major ISPs in Germany, Austria, Belgium, Italy, the Netherlands, Spain, Switzerland, and other countries. These guides cover provider-specific settings such as PPPoE credentials format, required VLAN IDs, and any non-standard configuration.

To find the guide for your ISP, visit:
- [Setting up the FRITZ!Box: Guides for different providers](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/3588_Setting-up-the-FRITZ-Box-Guides-for-different-providers/)

Examples of providers with dedicated guides: Deutsche Telekom, Vodafone, o2, 1&1, Proximus, Fastweb, KPN, MásMóvil, Swisscom, Sunrise, A1, and many more.

---

## References

- [Setting up FRITZ!Box for use with a DSL line](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/635_Setting-up-FRITZ-Box-for-use-with-a-DSL-line)
- [Setting up the FRITZ!Box: Guides for different providers](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/3588_Setting-up-the-FRITZ-Box-Guides-for-different-providers/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [VPN connections in a cascading FRITZ!Box](https://en.avm.de/service/knowledge-base/dok/FRITZ-Box-7690/3766_Setting-up-VPN-connections-in-a-FRITZ-Box-used-as-a-cascading-router/)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
