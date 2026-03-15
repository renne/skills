---
name: mcp-netbird
description: Configure and use the mcp-netbird MCP server to let AI assistants (Kiro, Claude Desktop, and other MCP clients) manage a NetBird VPN infrastructure through natural language or tool calls. Use when setting up AI-driven NetBird administration, looking up the 50+ available tool names and parameters (peers, groups, policies, networks, DNS, setup keys, users, posture checks, port allocations), choosing a deployment mode (Docker MCP Gateway, local STDIO, or remote SSE), or building automated NetBird workflows with an AI agent.
---
# NetBird MCP Server (mcp-netbird)

Source: https://github.com/XNet-NGO/mcp-netbird

## Overview

`mcp-netbird` is a [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that lets AI assistants (Kiro, Claude Desktop, and any other MCP client) manage NetBird VPN infrastructure through natural language or programmatic tool calls.

Use the MCP server or the supported NetBird management API for NetBird changes whenever possible. Do **not** modify NetBird's backing database directly for operational changes such as peers, groups, policies, reverse-proxy configuration, DNS, routes, or account state. Direct database edits bypass validation and can leave the management state inconsistent.

It provides **50+ tools** for complete CRUD operations on all NetBird resources:

| Resource | Operations |
|----------|-----------|
| Peers | list, get, update, delete |
| Groups | list, get, create, update, delete |
| Policies | list, get, create, update, delete |
| Networks | list, get, create, update, delete |
| Network Resources | list, get, create, update, delete |
| Network Routers | list, get, create, update, delete |
| Nameservers | list, get, create, update, delete |
| Routes (legacy) | list, get, create, update, delete |
| Setup Keys | list, get, create, update, delete |
| Users | list, get, invite, update, delete |
| Posture Checks | list, get, create, update, delete |
| Port Allocations | list, get, create, update, delete |
| Account | get, update |

### Helper Tools

- **list_policies_by_group** – find all policies referencing a specific group
- **replace_group_in_policies** – bulk-replace a group across all policies
- **get_policy_template** – retrieve example policy structures with inline documentation

---

## Token & Endpoint Location (this installation)

When MCP tools are unavailable or insufficient (e.g. for raw `curl` calls to endpoints the MCP server doesn't expose), retrieve the live credentials from:

```
~/.copilot/mcp-config.json
```

Relevant fields:

| Field | Path in JSON |
|-------|-------------|
| API token (`nbp_…`) | `.mcpServers.netbird.headers["X-Netbird-API-Token"]` |
| MCP SSE endpoint | `.mcpServers.netbird.url` |

The MCP SSE server runs on vps1 (`mcp-netbird` Docker container, WireGuard IP `100.115.218.142`, port `8001`).

**Never copy the token value into skill files or commit it to git.** Always read it at runtime:

```bash
NB_TOKEN=$(jq -r '.mcpServers.netbird.headers["X-Netbird-API-Token"]' ~/.copilot/mcp-config.json)
curl -sf -H "Authorization: Token $NB_TOKEN" https://netbird.bartschnet.de/api/...
```

---

## Prerequisites

A NetBird API token is required before any deployment:

1. Log in to your NetBird dashboard (<https://app.netbird.io> or your self-hosted instance).
2. Go to **Settings → Access Tokens**.
3. Create a **Service User** (recommended) or use a personal token; assign the **admin** or **network_admin** role.
4. Copy the generated token (starts with `nbp_`).

---

## Deployment Options

### Option 1: Docker MCP Gateway (Recommended)

The Docker MCP Gateway translates between the local STDIO protocol expected by desktop MCP clients and the remote SSE protocol served by the container.

**1. Enable the server** – create or edit `~/.docker/mcp/config.yaml`:

```yaml
netbird-mcp-server:
  netbird_host: api.netbird.io   # or your self-hosted domain
  enabled: true
```

**2. Set the API token** – create or edit `~/.docker/mcp/.env`:

```env
NETBIRD_API_TOKEN=nbp_your_token_here
```

**3. Configure your MCP client**

For **Kiro** (`~/.kiro/settings/mcp.json`):

```json
{
  "mcpServers": {
    "MCP_DOCKER": {
      "command": "docker",
      "args": ["mcp", "gateway", "run", "--servers=netbird-mcp-server"],
      "disabled": false,
      "autoApprove": ["*"]
    }
  }
}
```

For **Claude Desktop** (`~/Library/Application Support/Claude/claude_desktop_config.json` on macOS):

```json
{
  "mcpServers": {
    "MCP_DOCKER": {
      "command": "docker",
      "args": ["mcp", "gateway", "run", "--servers=netbird-mcp-server"]
    }
  }
}
```

Tools are exposed with the prefix `mcp_MCP_DOCKER_` (e.g., `mcp_MCP_DOCKER_list_netbird_peers`).

---

### Option 2: Local STDIO Mode

Run the server as a local process for development or single-user use.

**For Kiro** (`~/.kiro/settings/mcp.json`):

```json
{
  "mcpServers": {
    "netbird": {
      "command": "docker",
      "args": ["run", "--rm", "-i", "xnetadmin/mcp-netbird:latest", "-t", "stdio"],
      "env": {
        "NETBIRD_API_TOKEN": "nbp_your_token_here",
        "NETBIRD_API_HOST": "api.netbird.io"
      },
      "disabled": false
    }
  }
}
```

---

### Option 3: Remote SSE Server

Deploy as a shared service for team collaboration.

**`docker-compose.yml`**:

```yaml
services:
  mcp-netbird:
    image: xnetadmin/mcp-netbird:latest
    container_name: mcp-netbird-server
    restart: unless-stopped
    command: ["-t", "sse", "-sse-address", "0.0.0.0:8001"]
    environment:
      - NETBIRD_API_TOKEN=nbp_your_token_here
      - NETBIRD_API_HOST=api.netbird.io
    ports:
      - "8001:8001"
```

Then update the Docker MCP Gateway config to point at the remote URL:

```yaml
netbird-mcp-server:
  url: https://mcp.example.com/sse
  transport: sse
  enabled: true
```

---

## Configuration Priority

`CLI Arguments > HTTP Headers > Environment Variables`

| Source | Example |
|--------|---------|
| Environment variable | `NETBIRD_API_TOKEN=nbp_…` |
| HTTP header | `X-Netbird-API-Token: nbp_…` |
| CLI argument | `--api-token nbp_…` |

---

## Usage Examples

### Natural Language (AI Assistant)

After connecting your MCP client, ask:

- "Can you explain my NetBird peers, groups, and policies?"
- "Create a new group called 'developers' and generate a reusable setup key for it"
- "Show me all policies that reference the admin group"
- "Configure DNS to use Cloudflare for all peers"
- "Delete the peer named old-server"

### Tool Calls

**List all peers**:

```js
mcp_MCP_DOCKER_list_netbird_peers()
// Returns: array of peer objects with IP, status, groups, etc.
```

**Create a group**:

```js
mcp_MCP_DOCKER_create_netbird_group({
  name: "developers",
  peers: []
})
```

**Create a setup key**:

```js
mcp_MCP_DOCKER_create_netbird_setup_key({
  name: "dev-team-key",
  type: "reusable",
  expires_in: 2592000,   // 30 days in seconds
  auto_groups: ["dev-group-id"],
  usage_limit: 50
})
```

**Create a policy (SSH access)**:

```js
mcp_MCP_DOCKER_create_netbird_policy({
  name: "Admin SSH Access",
  description: "Allow admins to SSH into servers",
  enabled: true,
  rules: [{
    name: "SSH Rule",
    enabled: true,
    action: "accept",
    bidirectional: false,
    protocol: "tcp",
    sources: ["admin-group-id"],
    destinations: ["server-group-id"],
    port_ranges: [{ start: 22, end: 22 }]
  }]
})
```

**Find policies by group**:

```js
mcp_MCP_DOCKER_list_policies_by_group({ group_id: "d535b93ngf8s73892nng" })
// Returns: all policies that reference this group
```

**Bulk replace a group across all policies**:

```js
mcp_MCP_DOCKER_replace_group_in_policies({
  old_group_id: "old-group-id",
  new_group_id: "new-group-id"
})
```

**Add a DNS nameserver**:

```js
mcp_MCP_DOCKER_create_netbird_nameserver({
  name: "Cloudflare DNS",
  nameservers: [
    { ip: "1.1.1.1", ns_type: "udp", port: 53 },
    { ip: "1.0.0.1", ns_type: "udp", port: 53 }
  ],
  enabled: true,
  groups: ["all-group-id"],
  primary: true,
  domains: [],
  search_domains_enabled: false
})
```

---

## Policy Rule Format

When creating or updating policies, each rule must follow this structure:

```json
{
  "name": "Rule Name",
  "description": "Optional description",
  "enabled": true,
  "action": "accept",
  "bidirectional": true,
  "protocol": "all",
  "sources": ["group-id-1"],
  "destinations": ["group-id-2"],
  "port_ranges": [
    { "start": 80, "end": 80 },
    { "start": 443, "end": 443 }
  ]
}
```

**Required fields**: `name`, `enabled`, `action`, `bidirectional`, `protocol`, at least one source, at least one destination.  
**Optional fields**: `description`, `port_ranges` (TCP/UDP only), `authorized_groups`.  
**`protocol`** values: `all`, `tcp`, `udp`, `icmp`.

Retrieve annotated example structures:

```js
mcp_MCP_DOCKER_get_policy_template()
```

---

## Building from Source

```bash
git clone https://github.com/XNet-NGO/mcp-netbird
cd mcp-netbird
make install
# or
go install github.com/XNet-NGO/mcp-netbird/cmd/mcp-netbird@latest
```

---

## Troubleshooting

| Problem | Action |
|---------|--------|
| Tools not appearing | Restart the MCP client after configuration changes |
| Connection timeout | Verify the API token is valid and has appropriate permissions |
| 401 Unauthorized | Check that the API token has not expired |
| Self-hosted instance | Set `NETBIRD_API_HOST` to your domain (e.g., `api.yourdomain.com`) |

---

## References

- [mcp-netbird on GitHub](https://github.com/XNet-NGO/mcp-netbird)
- [mcp-netbird on Docker Hub](https://hub.docker.com/r/xnetadmin/mcp-netbird)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [NetBird API Reference](https://docs.netbird.io/api)
