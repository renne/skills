---
name: getting-started
description: NetBird getting started guide covering installation on Linux, macOS, Windows, Docker, and other platforms, client CLI commands, setup keys for headless enrollment, and initial peer connection. Use when installing NetBird, connecting devices to a private mesh network, or looking up CLI commands for managing the NetBird client.
---
# NetBird Getting Started

Source: https://docs.netbird.io/get-started

## What is NetBird?

NetBird is an open-source, zero-trust networking platform that creates encrypted peer-to-peer mesh networks using WireGuard. It automates key exchange, peer discovery, NAT traversal, and IP assignment so devices can securely communicate regardless of their network location.

### How It Works

- **WireGuard** provides the encrypted data plane between peers.
- **Management server** handles peer registration, authentication, policy distribution, and network maps.
- **Signaling server** facilitates ICE (Interactive Connectivity Establishment) negotiation for direct P2P connections.
- **STUN** allows peers to discover their public IP/port; **TURN/relay** provides fallback routing when direct connections are blocked by symmetric NAT or restrictive firewalls.
- All connections are outbound — no open inbound ports are required.

---

## Onboarding Wizard

When you first log in to the NetBird dashboard, an onboarding wizard guides setup through two modes:

### Mode 1: Peer-to-Peer Network

Connect laptops, desktops, and servers directly in a mesh. Steps:
1. Install the NetBird client on each device.
2. Sign in with your identity provider (SSO) — the peer appears in the dashboard automatically.
3. The default policy allows all peers to communicate. For Zero Trust, delete this and create targeted policies.

### Mode 2: Remote Network Access

Provides VPN-like access to a private subnet or cloud VPC without requiring NetBird on every host. Steps:
1. Install NetBird on a **routing peer** inside the target network.
2. Register it with a **Setup Key** (for headless/automated enrollment).
3. Define a **Network** resource (e.g., `192.168.1.0/24`).
4. Create an **Access Policy** granting specific peer groups access to that network.
5. Users can access devices on the subnet using their NetBird IP as a gateway.

**Demo of zero-trust in the onboarding**: the wizard lets you toggle the Default Policy on/off to instantly see how deny-by-default works before adding your first real policy.

### IDP Sync for User Groups

If your identity provider (e.g., Okta, Azure AD) has user groups, NetBird can sync them automatically. Synced groups appear in **Access Control → Groups** and can be used directly in policies.

---

## Installation

### Linux (script)

```bash
curl -fsSL https://pkgs.netbird.io/install.sh | sh
```

### Linux (APT)

```bash
sudo apt-get install ca-certificates curl gnupg -y
curl -sSL https://pkgs.netbird.io/debian/public.key \
  | sudo gpg --dearmor -o /usr/share/keyrings/netbird-keyring.gpg
echo 'deb [signed-by=/usr/share/keyrings/netbird-keyring.gpg] https://pkgs.netbird.io/debian stable main' \
  | sudo tee /etc/apt/sources.list.d/netbird.list
sudo apt-get update
sudo apt-get install netbird -y
```

### macOS (Homebrew)

```bash
brew install netbirdio/tap/netbird
sudo netbird service install
sudo netbird service start
```

Download the `.pkg` installer from the NetBird dashboard as an alternative.

### Windows

Download the `.exe` or `.msi` installer from the NetBird dashboard and follow the setup wizard.

### Docker

```bash
docker run --rm -it --cap-add=NET_ADMIN --cap-add=SYS_ADMIN --cap-add=SYS_RESOURCE \
  -v netbird-client:/etc/netbird \
  --name netbird \
  netbirdio/netbird:latest
```

### Other Platforms

Official packages are also available for Android, iOS, Synology NAS, TrueNAS, FreeBSD, OpenWRT, and Raspberry Pi. See https://docs.netbird.io/get-started/install for platform-specific instructions.

---

## Connecting Your First Device

### Interactive Login (GUI/Desktop)

Start the client and authenticate through your browser:

```bash
netbird up
```

A browser window opens for SSO login. After authentication the peer appears in the NetBird dashboard.

### Headless / Server Enrollment with a Setup Key

For servers and automated deployments, generate a **Setup Key** in the dashboard (Dashboard → Setup Keys → Add Key) and pass it on the command line:

```bash
netbird up --setup-key <SETUP_KEY>
```

Setup keys can be:
- **One-time**: Used once; expires after first use.
- **Reusable**: Can enroll multiple peers; optionally auto-assigns groups.

### Specifying a Management URL (self-hosted)

```bash
netbird up --setup-key <SETUP_KEY> --management-url https://netbird.example.com:443
```

---

## CLI Reference

| Command | Description |
|---------|-------------|
| `netbird up` | Start client and connect to the network |
| `netbird up --setup-key <KEY>` | Connect using a setup key (headless) |
| `netbird down` | Disconnect and remove the WireGuard interface |
| `netbird status` | Show current connection state |
| `netbird status --detail` | Detailed status: peers, routes, relays, management |
| `netbird networks list` | List configured networks |
| `netbird networks enable <id>` | Enable a specific network |
| `netbird networks disable <id>` | Disable a specific network |
| `netbird service install` | Install the system service |
| `netbird service start` | Start the system service |
| `netbird service stop` | Stop the system service |
| `netbird version` | Print the client version |
| `netbird --help` | Show all available commands |

---

## Verifying the Connection

After running `netbird up`, verify connectivity:

```bash
# Check the assigned NetBird IP and peer list
netbird status --detail

# Ping another peer by its NetBird IP (e.g., 100.64.x.x)
ping 100.64.0.1
```

All peers in the same network and policy group can reach each other by their NetBird IP addresses and automatically assigned DNS names (e.g., `my-peer.netbird.cloud`).

---

## Setup Keys

Setup keys are found in the dashboard under **Setup Keys**. Key options:

| Option | Description |
|--------|-------------|
| **Name** | Human-readable label |
| **Type** | `one-time` (single use) or `reusable` |
| **Expiration** | Optional expiry date |
| **Auto-groups** | Groups automatically assigned to enrolled peers |
| **Ephemeral peers** | Peers are deleted from the network when they disconnect |

---

## Tips

- Peers automatically reconnect after network changes or reboots when the service is installed.
- Use `netbird status --detail` to diagnose connectivity issues; it shows relay vs. direct connections.
- Remove a peer from the network in the dashboard under **Peers** → select peer → **Delete**.

## References

- [Getting Started](https://docs.netbird.io/get-started)
- [Install NetBird](https://docs.netbird.io/get-started/install)
- [Linux Installation](https://docs.netbird.io/get-started/install/linux)
- [Windows Installation](https://docs.netbird.io/get-started/install/windows)
- [CLI Reference](https://docs.netbird.io/get-started/cli)
- [How NetBird Works](https://docs.netbird.io/about-netbird/how-netbird-works)
- [Understanding NAT and Connectivity](https://docs.netbird.io/about-netbird/understanding-nat-and-connectivity)
