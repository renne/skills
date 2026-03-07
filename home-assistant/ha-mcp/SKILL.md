---
name: ha-mcp
description: Install, configure, and use the ha-mcp MCP server to let AI assistants (Claude Desktop, Claude Code, ChatGPT, VS Code, Cursor, and 15+ other MCP clients) control a Home Assistant installation through natural language or tool calls. Use when setting up AI-driven Home Assistant management, looking up the 97 available tool names and parameters (entity search, service calls, automations, scripts, helpers, dashboards, areas, calendars, history, backups, and more), choosing a deployment method (Home Assistant OS add-on, macOS/Windows quick-install, pip/uvx, or Docker), or configuring a specific MCP client to connect to the server.
---
# Home Assistant MCP Server (ha-mcp)

Source: https://github.com/homeassistant-ai/ha-mcp

## Overview

`ha-mcp` is a [Model Context Protocol](https://modelcontextprotocol.io) (MCP) server that lets AI assistants (Claude Desktop, Claude Code, ChatGPT, VS Code Copilot, Cursor, Open WebUI, Gemini CLI, and more) control a Home Assistant installation through natural language or programmatic tool calls.

It provides **97 tools** covering every major Home Assistant subsystem:

| Category | Tools |
|----------|-------|
| **Search & Discovery** | `ha_search_entities`, `ha_deep_search`, `ha_get_overview`, `ha_get_state` |
| **Service & Device Control** | `ha_call_service`, `ha_bulk_control`, `ha_get_operation_status`, `ha_get_bulk_status`, `ha_list_services` |
| **Automations** | `ha_config_get_automation`, `ha_config_set_automation`, `ha_config_remove_automation` |
| **Scripts** | `ha_config_get_script`, `ha_config_set_script`, `ha_config_remove_script` |
| **Helper Entities** | `ha_config_list_helpers`, `ha_config_set_helper`, `ha_config_remove_helper` |
| **Dashboards** | `ha_config_get_dashboard`, `ha_config_set_dashboard`, `ha_config_delete_dashboard`, `ha_get_dashboard_guide`, `ha_get_card_documentation` |
| **Areas & Floors** | `ha_config_list_areas`, `ha_config_set_area`, `ha_config_remove_area`, `ha_config_list_floors`, `ha_config_set_floor`, `ha_config_remove_floor` |
| **Labels** | `ha_config_get_label`, `ha_config_set_label`, `ha_config_remove_label`, `ha_manage_entity_labels` |
| **Zones** | `ha_get_zone`, `ha_create_zone`, `ha_update_zone`, `ha_delete_zone` |
| **Groups** | `ha_config_list_groups`, `ha_config_set_group`, `ha_config_remove_group` |
| **Todo Lists** | `ha_get_todo`, `ha_add_todo_item`, `ha_update_todo_item`, `ha_remove_todo_item` |
| **Calendar** | `ha_config_get_calendar_events`, `ha_config_set_calendar_event`, `ha_config_remove_calendar_event` |
| **Blueprints** | `ha_list_blueprints`, `ha_get_blueprint`, `ha_import_blueprint` |
| **Device Registry** | `ha_get_device`, `ha_update_device`, `ha_remove_device`, `ha_rename_entity` |
| **ZHA & Integrations** | `ha_get_zha_devices`, `ha_get_entity_integration_source` |
| **Add-ons** | `ha_get_addon` |
| **Camera** | `ha_get_camera_image` |
| **History & Statistics** | `ha_get_history`, `ha_get_statistics` |
| **Automation Traces** | `ha_get_automation_traces` |
| **System & Updates** | `ha_check_config`, `ha_restart`, `ha_reload_core`, `ha_get_system_info`, `ha_get_system_health`, `ha_get_updates` |
| **Backup & Restore** | `ha_backup_create`, `ha_backup_restore` |
| **Utility** | `ha_get_logbook`, `ha_eval_template`, `ha_get_domain_docs`, `ha_get_integration` |

---

## Installation

### Option 1: Home Assistant OS Add-on (recommended for HAOS users)

No token or credential setup needed — the add-on connects automatically.

1. Add the repository to Home Assistant:

   [![Add Repository](https://my.home-assistant.io/badges/supervisor_add_addon_repository.svg)](https://my.home-assistant.io/redirect/supervisor_add_addon_repository/?repository_url=https%3A%2F%2Fgithub.com%2Fhomeassistant-ai%2Fha-mcp)

   Or manually add this URL in **Supervisor → Add-on Store → ⋮ → Repositories**:
   ```
   https://github.com/homeassistant-ai/ha-mcp
   ```

2. Install **"Home Assistant MCP Server"** from the Add-on Store and wait for it to complete.
3. Click **Start**, then open the **Logs** tab to find your unique MCP server URL:
   ```
   🔐 MCP Server URL: http://192.168.1.100:9583/private_zctpwlX7ZkIAr7oqdfLPxw
   ```
4. Configure your AI client with that URL (see [Client Configuration](#client-configuration)).

The add-on listens on port **9583** and is accessible on the local network only by default.

### Option 2: Quick Install (macOS / Windows)

Connects to a demo Home Assistant environment — swap the URL for your own instance afterwards.

**macOS:**
```bash
curl -LsSf https://raw.githubusercontent.com/homeassistant-ai/ha-mcp/master/scripts/install-macos.sh | sh
```
Then restart Claude Desktop and ask: _"Can you see my Home Assistant?"_

**Windows (PowerShell):**
```powershell
irm https://raw.githubusercontent.com/homeassistant-ai/ha-mcp/master/scripts/install-windows.ps1 | iex
```
Then restart Claude Desktop and ask: _"Can you see my Home Assistant?"_

### Option 3: pip / uvx

```bash
# Using uv (recommended)
uvx ha-mcp

# Using pip
pip install ha-mcp
ha-mcp
```

Set the following environment variables before running:

| Variable | Description |
|----------|-------------|
| `HA_URL` | Home Assistant base URL, e.g. `http://192.168.1.100:8123` |
| `HA_TOKEN` | Long-lived access token (Settings → Profile → Long-Lived Access Tokens) |

### Option 4: Docker

```bash
docker run -d \
  --name ha-mcp \
  -p 9583:9583 \
  -e HA_URL=http://192.168.1.100:8123 \
  -e HA_TOKEN=<YOUR_LONG_LIVED_ACCESS_TOKEN> \
  ghcr.io/homeassistant-ai/ha-mcp:latest
```

Or with Docker Compose:

```yaml
services:
  ha-mcp:
    image: ghcr.io/homeassistant-ai/ha-mcp:latest
    restart: unless-stopped
    ports:
      - "9583:9583"
    environment:
      HA_URL: http://192.168.1.100:8123
      HA_TOKEN: <YOUR_LONG_LIVED_ACCESS_TOKEN>
```

---

## Client Configuration

Replace `<MCP_URL>` with the full URL shown in the add-on logs or your server address, e.g.:
```
http://192.168.1.100:9583/private_zctpwlX7ZkIAr7oqdfLPxw
```

### Claude Desktop

Claude Desktop requires **mcp-proxy** to connect to HTTP MCP servers.

1. Install mcp-proxy:
   ```bash
   uv tool install mcp-proxy
   # or
   pipx install mcp-proxy
   ```

2. Add to the Claude Desktop configuration file:
   - **macOS:** `~/Library/Application Support/Claude/claude_desktop_config.json`
   - **Windows:** `%APPDATA%\Claude\claude_desktop_config.json`

   ```json
   {
     "mcpServers": {
       "home-assistant": {
         "command": "mcp-proxy",
         "args": ["--transport", "streamablehttp", "<MCP_URL>"]
       }
     }
   }
   ```

3. Restart Claude Desktop.

### Claude Code

```bash
claude mcp add-json home-assistant '{
  "url": "<MCP_URL>",
  "transport": "http"
}'
```

Restart Claude Code after adding the configuration.

### VS Code (GitHub Copilot)

Add to `.vscode/mcp.json` in your workspace or to your user settings:

```json
{
  "servers": {
    "home-assistant": {
      "url": "<MCP_URL>",
      "type": "http"
    }
  }
}
```

### Cursor

Add to `~/.cursor/mcp.json`:

```json
{
  "mcpServers": {
    "home-assistant": {
      "url": "<MCP_URL>",
      "transport": "http"
    }
  }
}
```

### Open WebUI

In Open WebUI, go to **Settings → Tools → Add MCP Server** and paste `<MCP_URL>`.

### Other Clients (Setup Wizard)

A Setup Wizard covering 15+ clients (Gemini CLI, ChatGPT, Windsurf, Zed, Continue, etc.) is available at:
https://homeassistant-ai.github.io/ha-mcp/setup/

---

## Add-on Configuration Options

The add-on has minimal configuration. Most settings are automatic.

| Option | Default | Description |
|--------|---------|-------------|
| `backup_hint` | `normal` | When to suggest backups: `normal` (before irreversible ops), `strong` (before first modification per session), `weak` (rarely), `auto` (future) |
| `secret_path` | *(auto)* | Custom secret path override. Leave empty to auto-generate a secure 128-bit random path |

To see advanced options in the UI, enable **"Show unused optional configuration options"** in the add-on configuration.

---

## Remote Access

To use web-based clients (claude.ai, ChatGPT) or access from outside the local network, expose the MCP server securely using the **Cloudflared add-on**:

1. Install the [Cloudflared add-on](https://github.com/brenner-tobias/addon-cloudflared).
2. Add to its configuration:
   ```yaml
   additional_hosts:
     - hostname: ha-mcp.yourdomain.com
       service: http://localhost:9583
   ```
3. The full MCP URL becomes:
   ```
   https://ha-mcp.yourdomain.com/private_zctpwlX7ZkIAr7oqdfLPxw
   ```

Never expose port 9583 directly to the internet.

---

## Usage Examples

### Natural Language Prompts

After connecting your MCP client, ask naturally:

| Prompt | What happens |
|--------|-------------|
| *"Create an automation that turns on the porch light at sunset"* | Creates the automation with proper triggers and actions |
| *"Add a weather card to my dashboard"* | Updates the Lovelace dashboard |
| *"The motion sensor automation isn't working, debug it"* | Analyzes execution traces, identifies the issue |
| *"Make my morning routine automation also turn on the coffee maker"* | Reads the existing automation, adds the new action |
| *"Create a script that sets movie mode: dim lights, close blinds, turn on TV"* | Creates a reusable script |
| *"Show me all lights in the living room"* | Uses fuzzy search to list matching entities |
| *"Turn off all lights in the house"* | Uses `ha_bulk_control` across all light entities |

### Tool Call Examples

**Search for entities:**
```json
ha_search_entities({ "query": "living room light" })
```

**Get entity state:**
```json
ha_get_state({ "entity_id": "light.living_room" })
```

**Call a service:**
```json
ha_call_service({
  "domain": "light",
  "service": "turn_on",
  "target": { "entity_id": "light.living_room" },
  "service_data": { "brightness_pct": 80, "color_temp": 300 }
})
```

**Bulk control:**
```json
ha_bulk_control({
  "domain": "light",
  "service": "turn_off",
  "area_id": "living_room"
})
```

**Get an automation:**
```json
ha_config_get_automation({ "automation_id": "motion_light" })
```

**Create or update an automation:**
```json
ha_config_set_automation({
  "automation_id": "sunset_porch",
  "config": {
    "alias": "Porch light at sunset",
    "triggers": [{ "trigger": "sun", "event": "sunset" }],
    "actions": [{ "action": "light.turn_on", "target": { "entity_id": "light.porch" } }]
  }
})
```

**Create a backup:**
```json
ha_backup_create({ "name": "Before automation changes" })
```

**Evaluate a template:**
```json
ha_eval_template({ "template": "{{ states('sensor.temperature') | float | round(1) }} °C" })
```

---

## Bundled Agent Skills

Skills from `homeassistant-ai/skills` are bundled inside the MCP server and served as [MCP resources](https://modelcontextprotocol.io/docs/concepts/resources) via `skill://` URIs. MCP clients that support resources can discover them automatically — no manual installation needed.

| Setting | Default | Description |
|---------|---------|-------------|
| `ENABLE_SKILLS` | `true` | Serve skills as MCP resources |
| `ENABLE_SKILLS_AS_TOOLS` | `false` | Also expose skills via `list_resources`/`read_resource` tools (for clients that don't support MCP resources natively) |

---

## Security

- **Auto-generated secret path** — each installation gets a unique, unpredictable URL segment using 128-bit cryptographic entropy, persisted to `/data/secret_path.txt`.
- **Local network only by default** — the add-on listens on port 9583 and is not reachable from the internet unless you explicitly expose it.
- **No token required for the add-on** — uses Home Assistant Supervisor's built-in authentication automatically.

---

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Tools not appearing in the client | Restart the MCP client after saving configuration |
| Can't connect to MCP server | Verify the add-on is running; confirm the full URL including secret path; check for firewall blocking port 9583 |
| Lost the secret URL | Restart the add-on and check logs; or read `/data/secret_path.txt` via the Terminal & SSH add-on |
| `401 Unauthorized` | For non-add-on installs, verify `HA_TOKEN` is a valid long-lived access token |
| Operations failing on wrong entity | Use `ha_search_entities` to find the correct entity ID before acting |
| Add-on won't start | Check logs for port conflicts (9583 already in use) or configuration errors |

---

## References

- [ha-mcp on GitHub](https://github.com/homeassistant-ai/ha-mcp)
- [Add-on documentation](https://github.com/homeassistant-ai/ha-mcp/blob/master/homeassistant-addon/DOCS.md)
- [Setup Wizard (15+ clients)](https://homeassistant-ai.github.io/ha-mcp/setup/)
- [FAQ & Troubleshooting](https://homeassistant-ai.github.io/ha-mcp/faq/)
- [Model Context Protocol](https://modelcontextprotocol.io)
- [Home Assistant Long-Lived Access Tokens](https://www.home-assistant.io/docs/authentication/#your-account-profile)
