---
name: network-security
description: FRITZ!Box 7690 network security including the stateful packet inspection firewall, parental controls and access profiles (time budgets, website filters, device blocks), stealth mode, WPA3 Wi-Fi security, and port sharing. Use when configuring parental controls, blocking websites or apps, restricting internet access by schedule, setting up device-level access rules, or hardening the FRITZ!Box firewall.
---
# FRITZ!Box 7690 Network Security

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 provides multi-layer network security:
- **Stateful packet inspection (SPI) firewall** – all unsolicited inbound traffic from the internet is blocked by default
- **Parental controls** – per-device internet access profiles with time budgets, schedules, and content filters
- **Device blocking** – instantly cut any device off from the internet
- **Wi-Fi security** – WPA2/WPA3 encryption
- **Stealth mode** – makes the FRITZ!Box invisible to port scans from the internet

---

## Firewall

### Default behavior
- All **inbound traffic** from the internet is blocked unless explicitly allowed via port sharing.
- All **outbound traffic** from the home network is allowed.
- The firewall uses **NAT** (Network Address Translation) – home network devices are not directly accessible from the internet.

### Stealth mode
Enable stealth mode to make the FRITZ!Box invisible to internet probes:
1. Go to **Internet → Filter → Lists → Global Filter Settings**.
2. Enable **Stealth Mode** (discard all unsolicited incoming packets).

### Port sharing (port forwarding)
Open specific ports for services that need to be accessible from the internet:
1. Go to **Internet → Permit Access → Port Sharing**.
2. Click **Add Device for Sharing**.
3. Select the device and the protocol/port.
4. Examples:
   - Port 443 TCP for a home web server
   - Port 51820 UDP for WireGuard VPN
5. Port sharing only exposes the specific port/protocol – all other ports remain blocked.

---

## Parental Controls (Access Profiles)

### What access profiles control
- **Time budgets** – how many hours per day/week a device can use the internet
- **Online schedules** – which hours of the day internet is allowed
- **Content filters** – block categories of websites (requires DNS-based filtering or configuring a filter list)
- **Blocked times** – absolute off-times regardless of remaining budget

### Creating and assigning access profiles
1. Go to **Internet → Filter → Access Profiles**.
2. Click **Add Access Profile** or edit an existing one.
3. Configure the profile:
   - **Online time**: set daily time budgets (e.g., 2 hours/day for children)
   - **Schedule**: define allowed online hours (e.g., 08:00–21:00)
   - **Website filter**: enable and select filter categories or add custom block/allow lists
4. Go to **Internet → Filter → Parental Controls**.
5. Assign the profile to one or more devices by selecting the device and the profile.

### Built-in profiles

| Profile | Default behavior |
|---------|-----------------|
| **Standard** | Unrestricted – full internet access |
| **Guest** | Applied to guest Wi-Fi devices |
| **No internet access** | Blocks all internet access for the device |

### Device-level blocking
Quick block/unblock any device without configuring a profile:
- Go to **Home Network → Network → Network Connections**.
- Click the edit icon next to a device and set **Internet Access** to **Off**.
- Or go to **Internet → Filter → Parental Controls** and click the block button next to a device.

---

## Website Filters

### Block specific websites
1. Go to **Internet → Filter → Lists → Global Filter Settings**.
2. Under **Blocked Websites**, add domain names to block for all devices (or per profile).

### Allow-list (whitelist) mode
For stricter control (e.g., children's devices):
1. In the access profile, enable **Permit only allowed websites**.
2. Add allowed domains to the allow list – everything else is blocked.

### Blocking network applications by port
Block specific applications (e.g., online games, streaming services):
1. Go to **Internet → Filter → Lists → Network Applications**.
2. Add a new network application rule with the protocol and port range.
3. Assign the block rule to a specific access profile.

---

## Wi-Fi Security

### Encryption settings
- The FRITZ!Box 7690 supports **WPA2** and **WPA3** (recommended: WPA2+WPA3 mixed mode for compatibility).
- Configure at **Wi-Fi → Wi-Fi Network → Security**.

### Changing the Wi-Fi password
1. Go to **Wi-Fi → Wi-Fi Network**.
2. Click **Edit** on the Wi-Fi network.
3. Enter a new password (minimum 8 characters; 12+ recommended).
4. Click **Apply** – all currently connected devices will need to reconnect with the new password.

### Guest Wi-Fi isolation
- Guest devices are isolated from the home network by default.
- Enable **Block guest access to Fritz!Box settings** to prevent guests from accessing the router interface.

---

## FRITZ!Box Password and Security Recommendations

- Set a strong FRITZ!Box admin password: **System → FRITZ!Box Users → admin user → Edit**.
- Enable **Two-factor verification** for the FRITZ!Box login: **System → FRITZ!Box Users → admin → Two-factor Verification**.
- Disable remote access to the FRITZ!Box web interface from the internet (only use MyFRITZ! or VPN for remote management).
- Regularly update FRITZ!OS: **System → Update**.

---

## Security Notifications

The FRITZ!Box can send push mail notifications for security events:
1. Go to **System → Push Services**.
2. Configure an email address for notifications.
3. Enable notification types:
   - Failed login attempts
   - New device connected
   - Certificate changes
   - Firmware updates available

---

## Important Limitations

- Parental controls and the firewall are **only active in router mode** (not in IP client mode or when used as a Mesh Repeater behind another FRITZ!Box).
- In IP client mode, use the primary router for firewall and parental control functions.
- Devices set to use a static IP (instead of DHCP) may bypass parental controls – ensure DHCP is enforced.

---

## References

- [Security functions (firewall) of the FRITZ!Box](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/57_Security-functions-firewall-of-the-FRITZ-Box)
- [Restricting internet use with FRITZ!Box parental controls](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7632/8_Restricting-internet-use-with-the-FRITZ-Box-parental-controls)
- [FRITZ!Box parental controls incorrectly block internet access](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/214_FRITZ-Box-parental-controls-incorrectly-block-internet-access/)
- [Complete control: Parental controls and device blocks](https://fritz.com/en/blogs/discover-fritz/security-complete-control-parental-controls-and-device-blocks-secure-your-home-network)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
