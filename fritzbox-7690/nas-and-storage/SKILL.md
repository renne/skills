---
name: nas-and-storage
description: FRITZ!Box 7690 FRITZ!NAS setup for USB storage devices including configuration, SMB/CIFS network sharing, FTP access, user permissions, media server (DLNA/UPnP), and remote access via MyFRITZ!. Use when setting up a network-attached storage share, enabling media streaming, configuring file access permissions, or connecting to the FRITZ!Box from Windows/macOS/Linux.
---
# FRITZ!Box 7690 NAS and Storage (FRITZ!NAS)

Source: https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690

## Overview

The FRITZ!Box 7690 can act as a basic **NAS (Network-Attached Storage)** using USB-connected storage devices:
- **USB 2.0 port** (type A) for storage devices
- Supports: USB sticks, USB hard drives, USB SSDs
- Network protocols: **SMB/CIFS** (Windows file sharing), **FTP**
- Integrated **media server** (DLNA/UPnP) for streaming to smart TVs and media players
- Remote access via **MyFRITZ!**

Access NAS settings at **Home Network → USB / Storage** in the web interface (http://fritz.box).

---

## Supported File Systems

| File System | Support |
|-------------|---------|
| NTFS | ✓ Read/write |
| exFAT | ✓ Read/write |
| FAT32 | ✓ Read/write |
| ext2 / ext3 / ext4 | ✓ Read/write |
| HFS+ (macOS) | Read-only |

Up to **4 partitions** per USB device, each up to **4 TB**.

---

## Setup: Connect and Enable NAS

1. Plug a USB storage device into the **USB port** on the back of the FRITZ!Box 7690.
2. In the web interface, go to **Home Network → USB / Storage**.
3. Verify the storage device is recognized (it appears in the device list).
4. Enable **Storage (NAS)**: turn on **Activate network storage (NAS)**.
5. Choose sharing options:
   - **Access via network drive (SMB)** – for Windows/macOS/Linux file sharing
   - **Access via FTP** – for FTP client access

---

## User Accounts and Permissions

Control who can access the NAS and with what rights:

1. Go to **System → FRITZ!Box Users**.
2. Click **Add User** or edit an existing user.
3. Enable **Access to NAS contents**.
4. Configure access rights:
   - **Read and write access** – full access
   - **Read-only access** – view files only
   - Restrict access to **specific folders** only
5. Click **Apply**.

Each user needs a username and password to connect to the NAS. The FRITZ!Box admin account also has NAS access by default.

---

## Connecting from Windows

### Via Explorer (SMB)
1. Open File Explorer and type `\\fritz.box` in the address bar (or `\\192.168.178.1`).
2. Enter the FRITZ!Box NAS username and password when prompted.
3. The NAS shares appear as folders.

### Map a network drive (persistent)
1. Right-click **This PC → Map Network Drive**.
2. Enter `\\fritz.box\<share-name>`.
3. Check **Reconnect at sign-in** and enter credentials.

---

## Connecting from macOS

1. In Finder, press **Cmd+K** (Go → Connect to Server).
2. Enter `smb://fritz.box` and click **Connect**.
3. Enter the FRITZ!Box NAS username and password.
4. Select the shared folder(s) to mount.

---

## Connecting from Linux

Mount via command line:

```bash
sudo mount -t cifs //fritz.box/<share-name> /mnt/fritznas \
  -o username=<user>,password=<password>,vers=3.0
```

Or add to `/etc/fstab` for automatic mounting at boot:

```
//fritz.box/<share-name> /mnt/fritznas cifs username=<user>,password=<password>,vers=3.0 0 0
```

---

## Connecting from iOS / Android

Use any SMB-capable file manager app (e.g., **Files** app on iOS, **Total Commander**, **Solid Explorer**):
- Protocol: SMB
- Server: `fritz.box` or `192.168.178.1`
- Username and password: NAS user credentials

For media streaming, use a DLNA-compatible app (e.g., **VLC**, **Infuse**, **Kodi**).

---

## Media Server (DLNA/UPnP)

Stream music, videos, and photos to DLNA-compatible devices (smart TVs, game consoles, media players):

1. Go to **Home Network → USB / Storage → Media Server** tab.
2. Enable **Media Server**.
3. Select the folders to share (e.g., `Music`, `Videos`, `Photos`).
4. DLNA-compatible devices on the home network automatically discover the media server.

### Force media library refresh
If new files are not appearing on DLNA clients:
1. Go to **Home Network → USB / Storage → Media Server**.
2. Click **Rebuild Media Library** (or **Update**).

### Supported media formats
The media server streams most common formats. Actual playback support depends on the receiving device.

---

## Share Links (FRITZ!OS 8+)

FRITZ!OS 8 introduced **share links** – shareable links to specific folders or files:
1. In **Home Network → USB / Storage**, browse to a folder.
2. Click **Create Share Link** on the folder.
3. Share the generated link with others. Recipients can download files without needing a NAS account.
4. Share links can be time-limited or revoked at any time.

---

## Remote Access via MyFRITZ!

Access FRITZ!NAS from outside the home network:
1. Enable **MyFRITZ!** at **Internet → Permit Access → MyFRITZ! Account** (free AVM service).
2. Log in at `https://myfritz.net` with your MyFRITZ! account.
3. Select **NAS** to browse files remotely via the secure web interface.

Alternatively, enable remote access via VPN (see the **VPN** skill) for full network-drive-level access.

---

## Performance Tips

- Use **USB 3.0** devices for better throughput (the FRITZ!Box 7690 has a USB 2.0 port, but a USB 3.0 device works at USB 2.0 speeds on this port).
- Prefer **SSD** over spinning hard drives for more consistent performance.
- Use **NTFS or ext4** for large files (FAT32 has a 4 GB file size limit).
- The FRITZ!Box NAS is suitable for light home use; for heavy workloads, consider a dedicated NAS device.

---

## Troubleshooting

| Symptom | Solution |
|---------|----------|
| USB device not recognized | Try a different USB port or cable; ensure the drive is formatted with a supported file system; check power requirements |
| Cannot connect from Windows | Verify SMB is enabled; check username/password; try `\\192.168.178.1` |
| Media not appearing on TV | Rebuild media library; verify the media server is enabled; check if the TV supports DLNA |
| Slow file transfer | Use USB 3.0 device; check Wi-Fi signal strength if accessing wirelessly |
| Remote access failing | Confirm MyFRITZ! is active; check internet connection |

---

## References

- [Configuring a USB storage device connected to FRITZ!Box](https://uk.fritz.com/service/knowledge-base/dok/FRITZ-Box-7690/26_Configuring-a-USB-storage-device-connected-to-FRITZ-Box/)
- [FRITZ!Box 7690 Service & Support](https://fritz.com/en/pages/service-fritz-box-7690)
- [FRITZ!Box 7690 User Manual (PDF, English)](https://www.manua.ls/avm/fritzbox-7690/manual)
