---
name: vpn
description: FRITZ!Box 7690 VPN setup for WireGuard (remote access and site-to-site) and IPSec (MyFRITZ! remote access). Use when configuring secure remote access to the home network, setting up WireGuard tunnels, creating VPN users, connecting from Windows/macOS/Linux/Android/iOS, or setting up VPN through a cascaded FRITZ!Box.
---
# FRITZ!Box 7690 VPN

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 supports two VPN technologies:
- **WireGuard** – modern, fast protocol for remote access and site-to-site connections (recommended; requires FRITZ!OS 7.50+)
- **IPSec** – classic VPN for compatibility with older clients and site-to-site scenarios

VPN settings are located at **Internet → Permit Access → VPN (WireGuard)** or **VPN (IPSec)** in the web interface.

**Prerequisite:** For remote VPN access, the FRITZ!Box must be reachable from the internet. Use **MyFRITZ!** (a free DynDNS service by AVM) or a static public IP: **Internet → Permit Access → MyFRITZ! Account**.

---

## WireGuard – Remote Access for a Single Device

Connect a phone, laptop, or any device securely to your home network from anywhere.

### Step 1: Set up MyFRITZ! or a DynDNS name

1. Go to **Internet → Permit Access → MyFRITZ! Account**.
2. Create a free MyFRITZ! account – the FRITZ!Box gets a hostname like `abcdef1234.myfritz.net`.

### Step 2: Create a WireGuard connection

1. Go to **Internet → Permit Access → VPN (WireGuard)**.
2. Click **Add VPN Connection**.
3. Select **Connect a single device** (remote access).
4. Enter a name (e.g., `My Laptop`).
5. Click **Finish**. The FRITZ!Box generates:
   - A WireGuard configuration file (`.conf`)
   - A QR code for mobile devices

### Step 3: Import on the client device

| Platform | Instructions |
|----------|-------------|
| Windows / macOS / Linux | Download the `.conf` file; import into the [WireGuard app](https://www.wireguard.com/install/) |
| Android / iOS | Scan the QR code using the WireGuard app |

### Step 4: Activate the tunnel

In the WireGuard app on your device, toggle the VPN connection on. All traffic to your home network is now encrypted and routed through the FRITZ!Box.

---

## WireGuard – Site-to-Site (Link Two Networks)

Connect two FRITZ!Boxes (e.g., home and office) so devices on both networks can communicate.

### On FRITZ!Box A (exporter):

1. **Internet → Permit Access → VPN (WireGuard) → Add VPN Connection**.
2. Select **Link networks** (site-to-site).
3. Enter a name for this connection.
4. Click **Finish** and download the configuration file.

### On FRITZ!Box B (importer):

1. **Internet → Permit Access → VPN (WireGuard) → Add VPN Connection**.
2. Select **Import configuration**.
3. Upload the configuration file from FRITZ!Box A.
4. The tunnel is established automatically.

Devices on the networks of both FRITZ!Boxes can now reach each other using their local IP addresses.

---

## IPSec – Remote Access via FRITZ!Box Users

Classic IPSec VPN using FRITZ!Box user accounts:

### Create a VPN user

1. Go to **System → FRITZ!Box Users → Add User**.
2. Set username and password.
3. Enable **VPN** permission.
4. Click **Apply**. The FRITZ!Box generates IPSec settings (pre-shared key, remote gateway, etc.).
5. Click the **Show VPN Settings** link for the user to view the configuration.

### Configure the client (Windows/macOS/iOS/Android)

Use the settings provided in the FRITZ!Box user VPN configuration:
- **Remote gateway**: your FRITZ!Box hostname (e.g., `abcdef1234.myfritz.net`)
- **Username** and **password**
- **Shared key (PSK)**

For Windows, AVM provides the free **FRITZ!VPN** client for easy configuration. WireGuard is generally preferred for new setups.

---

## VPN in a Cascaded FRITZ!Box

If the FRITZ!Box 7690 is behind another router (double NAT), additional steps are needed:

### Option A: Port forwarding on the upstream router
1. Forward UDP port **500** and **4500** (IPSec) or the WireGuard port (default: **51820**) from the upstream router to the FRITZ!Box 7690.
2. The upstream router must have a public IP or itself be reachable from the internet.

### Option B: DMZ / exposed host
1. On the upstream router, set the FRITZ!Box 7690 as the DMZ/exposed host.
2. All incoming traffic is forwarded to the FRITZ!Box, which handles VPN termination.

### Option C: FRITZ!Box as IP client
If the upstream device is also a FRITZ!Box, enable the Mesh Master relationship so the downstream FRITZ!Box inherits VPN capabilities through the Mesh Master.

---

## Split Tunneling vs. Full Tunnel

When creating a WireGuard connection:
- **Route all traffic through FRITZ!Box** – all internet traffic from the client goes through the home network (full tunnel; enables parental controls and firewall for remote devices).
- **Route only home network traffic** – only traffic to home network addresses uses the VPN; other traffic goes directly to the internet (split tunnel; better performance for internet access).

Configure under the WireGuard connection settings: **Allowed IPs**.

---

## MyFRITZ! (Remote Access without Full VPN)

MyFRITZ! provides secure web-based remote access to specific FRITZ!Box services (NAS, smart home, cameras) without setting up a full VPN:
- Access at `https://myfritz.net` after creating a MyFRITZ! account.
- Access the FRITZ!Box web interface, FRITZ!NAS, and smart home remotely.
- End-to-end encrypted; no port forwarding required.

### Setting up a MyFRITZ! account

1. In the FRITZ!Box web interface, click **Internet → MyFRITZ! Account**.
2. Enter your email address and click **Next**.
3. Open the confirmation email from MyFRITZ!Net and click **Register Your FRITZ!Box**.
4. Set a MyFRITZ! password on the myfritz.net website and click **Conclude Submission**.

### Enabling FRITZ!Box remote access via MyFRITZ!

1. In the FRITZ!Box web interface, click **Internet → MyFRITZ! Account**.
2. Click **Configure MyFRITZ! Internet Access**.
3. Select or create a FRITZ!Box user account for remote access and click **Apply**.

Your FRITZ!Box is now reachable at your personal `*.myfritz.net` hostname. This hostname is also used as the **Remote Gateway** when setting up WireGuard or IPSec VPN connections.

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| VPN tunnel won't connect | Verify MyFRITZ!/DynDNS hostname resolves; check firewall/port forwarding |
| Can't reach home devices via VPN | Confirm VPN connection is active; check routing (split vs. full tunnel) |
| WireGuard QR code not shown | Update FRITZ!OS to 7.50+ |
| Slow VPN throughput | Use WireGuard (faster than IPSec); check upload bandwidth at home |
| VPN drops after a few minutes | Check ISP for IP address changes; verify MyFRITZ! is active |

---

## References

- [VPN with FRITZ!Box overview](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/3448_VPN-with-FRITZ-Box)
- [Creating a MyFRITZ! account and setting it up in the FRITZ!Box](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/966_Creating-a-MyFRITZ-account-and-setting-it-up-in-the-FRITZ-Box/)
- [Setting up VPN connections in a cascading FRITZ!Box](https://en.avm.de/service/knowledge-base/dok/FRITZ-Box-7690/3766_Setting-up-VPN-connections-in-a-FRITZ-Box-used-as-a-cascading-router/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
