---
name: device-management
description: FRITZ!Box 7690 device management covering FRITZ!OS firmware updates (automatic and manual), configuration backup and restore, system event logs, diagnostics, energy settings, factory reset, and access management. Use when updating firmware, backing up or restoring settings, diagnosing problems, resetting the device, or managing FRITZ!Box administrator access.
---
# FRITZ!Box 7690 Device Management

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

Device management tasks on the FRITZ!Box 7690 are handled under the **System** menu in the web interface (http://fritz.box). Key areas:
- **Update** – FRITZ!OS firmware updates
- **Backup** – settings export/import
- **Events** – system event log
- **Push Services** – email/push notifications
- **Energy Monitor** – power consumption
- **Reboot / Reset** – restart or factory reset

---

## FRITZ!OS Firmware Updates

Keeping FRITZ!OS up to date provides new features, bug fixes, and security patches.

### Automatic update (recommended)
1. Go to **System → Update → Auto Update**.
2. Select the update option:
   - **Install recommended updates automatically** – updates install overnight
   - **Notify only** – the FRITZ!Box sends an email/push notification when an update is available, but does not install automatically
3. Click **Apply**.

### Online update (manual trigger)
1. Go to **System → Update**.
2. Click **Check for New FRITZ!OS**.
3. If an update is available, click **Start Update**.
4. The FRITZ!Box downloads and installs the firmware. A reboot follows automatically (takes 2–5 minutes).
5. Do not disconnect power during the update.

### Manual update (offline / specific version)
For installing a specific FRITZ!OS version or updating without internet access:

1. Download the correct firmware image from AVM's official server:
   ```
   https://download.avm.de/fritzbox/fritzbox-7690/deutschland/fritz.os/
   ```
   Choose the correct `.image` file for your FRITZ!Box 7690 variant.
2. Go to **System → Update → FRITZ!OS File**.
3. Click **Choose File** and select the downloaded `.image` file.
4. Click **Start Update**.

### Recent FRITZ!OS versions (FRITZ!Box 7690)

| Version | Notable changes |
|---------|----------------|
| FRITZ!OS 8.22 | Bug fixes for external modem interoperability; performance improvements |
| FRITZ!OS 8.20 | Online monitor with top bandwidth devices; FRITZ! Failsafe (internet failover); improved mesh optimization |
| FRITZ!OS 8.10 | Smart home routines improvements; energy monitoring enhancements |
| FRITZ!OS 8.00 | Initial FRITZ!OS 8 release for FRITZ!Box 7690; extended Zigbee support |

---

## Configuration Backup and Restore

### Export (backup) settings
1. Go to **System → Backup**.
2. Click **Save Settings**.
3. Optionally, set a password to protect the backup file.
4. Save the `.export` file to a safe location.

**Always back up before a firmware update or major configuration change.**

### Import (restore) settings
1. Go to **System → Backup**.
2. Click **Restore Settings**.
3. Select the `.export` file and enter the password if it was set.
4. Click **Restore** – the FRITZ!Box restores the configuration and reboots.

---

## System Event Log

The event log records system events, errors, and informational messages.

1. Go to **System → Events**.
2. Filter by category:
   - System
   - Internet connection
   - Telephone
   - Wi-Fi
   - USB
   - Smart Home
3. Click **Save** to download the log as a text file for support analysis.

---

## Push Services (Notifications)

Configure email or push notifications for FRITZ!Box events:

1. Go to **System → Push Services**.
2. Enter a recipient email address.
3. Enable notification categories:
   - New FRITZ!OS available
   - Wi-Fi: new device connected
   - Smart Home events (temperature alerts, device failures)
   - Security alerts (failed login attempts)
   - Missed calls
   - New answering machine messages

---

## Energy Monitor

View the FRITZ!Box power consumption and connected device statistics:
- Go to **System → Energy Monitor**.
- View current power draw and historical usage.
- Enable **Energy Saving Mode** to reduce transmission power during low-traffic periods.

---

## FRITZ!Box Users and Access Management

### Adding or editing users
1. Go to **System → FRITZ!Box Users**.
2. Click **Add User** or edit an existing user.
3. Set username, password, and permissions:
   - **FRITZ!Box settings** – can log into the web interface
   - **NAS contents** – file access
   - **VPN** – VPN access (IPSec)
   - **Smart Home** – control smart home devices remotely
   - **Call list** – view call history

### Two-factor authentication
Enable an additional authentication step for web interface logins:
1. Go to **System → FRITZ!Box Users → admin → Two-factor Verification**.
2. Choose a method (TOTP authenticator app or hardware token).
3. Follow the setup wizard.

---

## Reboot

Restart the FRITZ!Box without losing configuration:
1. Go to **System → Reboot**.
2. Click **Reboot**.
3. The FRITZ!Box restarts (takes ~60 seconds).

Alternatively, briefly press the power button on the device.

---

## Factory Reset

Restore the FRITZ!Box to factory settings (deletes all configuration):

**Via web interface:**
1. Go to **System → Backup**.
2. Click **Factory Settings** → **OK** to confirm.

**Via hardware button (if web interface is inaccessible):**
1. With the device powered on, press and hold the **WPS/WLAN** button for about 15 seconds until all LEDs flash.
2. Release the button. The FRITZ!Box resets and reboots.
3. After reset, reconfigure the device from scratch. Default Wi-Fi credentials are on the device label.

---

## FRITZ! Labor (Beta Firmware)

AVM offers **FRITZ! Labor** – beta firmware versions with upcoming features for early adopters:
- Download from the AVM website: [avm.de/fritz-labor](https://avm.de/fritz-labor) (select FRITZ!Box 7690 variant).
- Install via manual firmware update (see above).
- Beta firmware may contain bugs; maintain a backup before installing.
- To return to stable FRITZ!OS: perform a factory reset and install the stable firmware.

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| Power LED flashing red | Indicates a firmware error; perform a manual firmware update or factory reset |
| Web interface inaccessible | Try `192.168.178.1` directly; reboot the FRITZ!Box; check LAN cable |
| Update fails (online) | Check internet connection; try manual update with a downloaded image file |
| Can't restore backup | Ensure the backup file matches the FRITZ!Box model; check the backup password |
| Forgot FRITZ!Box password | Perform a factory reset (all settings will be lost) |

---

## References

- [Updating FRITZ!OS of the FRITZ!Box](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/2_Updating-FRITZ-OS-of-the-FRITZ-Box/)
- [Updating FRITZ!OS manually](https://fritz.com/en/apps/knowledge-base/FRITZ-Box-7690/1574_Updating-FRITZ-OS-manually/)
- [FRITZ!OS Update News (FRITZ!Box 7690)](https://fritz.com/en/pages/update-news?product_group=FRITZ%21Box&product=fritzbox-7690)
- [AVM FRITZ!OS download (FRITZ!Box 7690)](https://download.avm.de/fritzbox/fritzbox-7690/deutschland/fritz.os/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
