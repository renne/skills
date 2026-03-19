# skills
AI skills collection

Clone this repository with:
```bash
git clone --recurse-submodules https://github.com/renne/skills
```

Each skill is a directory containing a `SKILL.md` file following the [VS Code Agent Skills](https://code.visualstudio.com/docs/copilot/customization/agent-skills) open standard. See [AGENTS.md](AGENTS.md) for contribution rules.

## Skills

### GitHub Copilot

| Skill | Source |
|-------|--------|
| [Getting Started (Overview, features, plans, quickstart)](github-copilot/getting-started/SKILL.md) | https://docs.github.com/en/copilot |
| [Chat (IDE, GitHub.com, Mobile, slash commands, MCP, AI models)](github-copilot/chat/SKILL.md) | https://docs.github.com/en/copilot/concepts/chat |
| [Code Suggestions (inline completions, next edit suggestions, supported languages)](github-copilot/code-suggestions/SKILL.md) | https://docs.github.com/en/copilot/concepts/completions/code-suggestions |
| [Coding Agent (autonomous PR creation, custom agents, hooks, MCP, security)](github-copilot/coding-agent/SKILL.md) | https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent |
| [Customization (custom instructions, prompt engineering, response tuning)](github-copilot/customization/SKILL.md) | https://docs.github.com/en/copilot/concepts/prompting/response-customization |
| [CLI (terminal assistant, installation, modes, shortcuts, slash commands, skills, MCP, LSP)](github-copilot/cli/SKILL.md) | https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli |

### LimeSurvey

| Skill | Source |
|-------|--------|
| [Contributing (PHP backend, JS frontend, Twig/SCSS themes, OpenAPI, GitHub Actions CI, Cypress E2E, MySQL/PostgreSQL/MSSQL)](limesurvey/contributing/SKILL.md) | https://github.com/LimeSurvey/LimeSurvey |

### Moodle

| Skill | Source |
|-------|--------|
| [PHP Coding Style](moodle/php-coding-style/SKILL.md) | https://moodledev.io/general/development/policies/codingstyle |
| [Moodle App Coding Style](moodle/app-coding-style/SKILL.md) | https://moodledev.io/general/development/policies/codingstyle-moodleapp |
| [Moodle App Development Guide](moodle/app-development-guide/SKILL.md) | https://moodledev.io/general/app/development/development-guide |
| [Moodle App Plugins Development Guide](moodle/app-plugins-development-guide/SKILL.md) | https://moodledev.io/general/app/development/plugins-development-guide |

### Nextcloud

| Skill | Source |
|-------|--------|
| [Coding Style & General Guidelines](nextcloud/coding-standards/SKILL.md) | https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/index.html |
| [PHP Coding Standards](nextcloud/php-coding-standards/SKILL.md) | https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/php.html |
| [App Development Guide](nextcloud/app-development/SKILL.md) | https://docs.nextcloud.com/server/19/developer_manual/app/index.html |

### Docker Compose

| Skill | Source |
|-------|--------|
| [Overview & Getting Started](docker-compose/overview/SKILL.md) | https://docs.docker.com/compose/ |
| [Compose File Reference](docker-compose/compose-file-reference/SKILL.md) | https://docs.docker.com/reference/compose-file/ |
| [How-Tos (Networking, Env Vars, Secrets, Profiles, Watch, Multiple Files)](docker-compose/how-tos/SKILL.md) | https://docs.docker.com/compose/how-tos/ |

### CoreDNS

| Skill | Source |
|-------|--------|
| [Overview & Configuration (Corefile, server blocks, snippets, env vars)](coredns/overview/SKILL.md) | https://coredns.io/manual/toc/ |
| [Plugins Reference (forward, cache, rewrite, file, kubernetes, acl, dnssec, hosts, template, and more)](coredns/plugins/SKILL.md) | https://coredns.io/plugins/ |
| [Kubernetes (cluster DNS, ConfigMap, stub zones, debugging, monitoring)](coredns/kubernetes/SKILL.md) | https://coredns.io/manual/toc/ |
### NetBird

| Skill | Source |
|-------|--------|
| [Getting Started (Installation, CLI, Setup Keys)](netbird/getting-started/SKILL.md) | https://docs.netbird.io/get-started |
| [Access Control (Groups, Policies, Posture Checks)](netbird/access-control/SKILL.md) | https://docs.netbird.io/manage/access-control |
| [Networks and DNS (Routes, Site-to-Site, Nameservers)](netbird/networks-and-dns/SKILL.md) | https://docs.netbird.io/manage/networks |
| [DNS Management (Nameservers, Custom Zones, DNS Settings, Aliases, Troubleshooting)](netbird/dns/SKILL.md) | https://docs.netbird.io/manage/dns |
| [Self-Hosted Deployment (Docker Compose, Config, IdP)](netbird/self-hosted/SKILL.md) | https://docs.netbird.io/selfhosted/selfhosted-quickstart |
| [MCP Server (AI Management via mcp-netbird)](netbird/mcp-netbird/SKILL.md) | https://github.com/XNet-NGO/mcp-netbird |
| [DNS + CoreDNS Integration (Custom Zones vs CoreDNS, forwarder, internal zone routing, Corefile patterns)](netbird/dns-coredns-integration/SKILL.md) | https://docs.netbird.io/manage/dns |
| [SSH Server (dashboard SSH, --allow-server-ssh flags, OAuth2 re-auth root cause, container apply, --disable-ssh-auth)](netbird/ssh/SKILL.md) | https://docs.netbird.io/manage/ssh |
| [Reverse Proxy (services, targets, domains, auth SSO/password/PIN, TLS, HA, custom domains, backend config, access logs)](netbird/reverse-proxy/SKILL.md) | https://docs.netbird.io/manage/reverse-proxy |
| [Reverse Proxy — Expose from CLI (netbird expose, flags, session lifecycle, troubleshooting)](netbird/reverse-proxy-expose-cli/SKILL.md) | https://docs.netbird.io/manage/reverse-proxy/expose-from-cli |
| [Public API (authentication, accounts, users, tokens, peers, setup keys, groups, networks, policies, routes, DNS, events)](netbird/api/SKILL.md) | https://docs.netbird.io/api |

### Proxmox VE

| Skill | Source |
|-------|--------|
| [Installation & Host System Administration (Debian base, networking, packages, disk management)](proxmox/installation/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [QEMU/KVM Virtual Machines (qm CLI, hardware config, cloud-init, PCI passthrough, snapshots)](proxmox/virtual-machines/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [LXC Containers (pct CLI, templates, networking, mount points, resource limits)](proxmox/containers/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [Storage (ZFS, Ceph RBD/CephFS, LVM, LVM-Thin, NFS, iSCSI, CIFS, pvesm)](proxmox/storage/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [Cluster Manager, Networking & Firewall (pvecm, SDN, bridges, VLANs, bonds, HA firewall)](proxmox/cluster-networking/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [User Management (realms, users, groups, roles, permissions, API tokens, 2FA)](proxmox/user-management/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [Backup & Restore (vzdump, backup modes, PBS integration, scheduled jobs, retention)](proxmox/backup-restore/SKILL.md) | https://pve.proxmox.com/pve-docs/ |
| [High Availability (HA manager, fencing, HA groups, resource states, failover)](proxmox/high-availability/SKILL.md) | https://pve.proxmox.com/pve-docs/ |

### Gramps Web

| Skill | Source |
|-------|--------|
| [Setup & Deployment (Docker Compose, configuration, user roles, OIDC, AI chat)](gramps-web/setup/SKILL.md) | https://www.grampsweb.org/install_setup/setup/ |
| [Frontend Development (Lit/LitElement components, API integration, views, state management)](gramps-web/frontend-development/SKILL.md) | https://github.com/gramps-project/gramps-web |
| [REST API (endpoints, authentication, data model, search, LLM/chat)](gramps-web/api/SKILL.md) | https://github.com/gramps-project/gramps-web-api |

### Vaultwarden

| Skill | Source |
|-------|--------|
| [Installation & Deployment (Docker images, docker run, Docker Compose, Caddy)](vaultwarden/installation/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki |
| [Configuration (environment variables, ENV_FILE, config precedence, DOMAIN, signups)](vaultwarden/configuration/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview |
| [HTTPS & Reverse Proxy (Caddy, nginx, Let's Encrypt, Cloudflare, OCSP)](vaultwarden/https-and-reverse-proxy/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS |
| [Admin Page (ADMIN_TOKEN, argon2id hashing, user management, organizations)](vaultwarden/admin-page/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page |
| [SMTP Configuration (email delivery, Gmail, Outlook, SendGrid, OAuth2 proxy)](vaultwarden/smtp-configuration/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration |
| [Backup & Restore (SQLite backup, data folder, automated backups, restore)](vaultwarden/backup-and-restore/SKILL.md) | https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault |

### Anthropics (submodule)

Skills from [anthropics/skills](https://github.com/anthropics/skills) are included as a Git submodule at [`anthropics/skills`](anthropics/skills).

### VoltAgent (submodule)

Skills from [VoltAgent/awesome-agent-skills](https://github.com/VoltAgent/awesome-agent-skills) are included as a Git submodule at [`voltagent/awesome-agent-skills`](voltagent/awesome-agent-skills).

### Traefik

| Skill | Source |
|-------|--------|
| [Overview & Getting Started (core concepts, request lifecycle, installation, router rules)](traefik/overview/SKILL.md) | https://doc.traefik.io/traefik/ |
| [Routing (HTTP/TCP/UDP routers, rules, services, load balancing, entrypoints)](traefik/routing/SKILL.md) | https://doc.traefik.io/traefik/routing/overview/ |
| [Middlewares (BasicAuth, ForwardAuth, Headers, RateLimit, Compress, StripPrefix, CircuitBreaker, and more)](traefik/middlewares/SKILL.md) | https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/ |
| [HTTPS & TLS (Let's Encrypt ACME, HTTP-01/DNS-01/TLS-ALPN-01 challenges, wildcard certs, TLS options, mTLS)](traefik/https-tls/SKILL.md) | https://doc.traefik.io/traefik/https/overview/ |
| [Providers (Docker labels, Kubernetes IngressRoute CRDs, file provider, Consul/etcd)](traefik/providers/SKILL.md) | https://doc.traefik.io/traefik/providers/overview/ |
| [Observability (Prometheus metrics, OpenTelemetry/Jaeger tracing, access logs, API dashboard)](traefik/observability/SKILL.md) | https://doc.traefik.io/traefik/observe/overview/ |

### INWX

| Skill | Source |
|-------|--------|
| [DNS and ACME Migration (DomRobot API, delegation changes, DNSSEC recovery, Proxmox/Traefik DNS-01)](inwx/dns-and-acme-migration/SKILL.md) | https://www.inwx.de/en/help/apicommands |

### Matter (Connected Home over IP)

| Skill | Source |
|-------|--------|
| [Overview (architecture, stack layers, nodes, clusters, fabrics, commissioning)](matter/overview/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/ |
| [Building (GN/ninja, prerequisites, ZAP tool, build_examples.py)](matter/building/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html |
| [Access Control (ACLs, privileges, AuthModes, subjects, targets, chip-tool)](matter/access-control/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/guides/access-control-guide.html |
| [Writing Clusters (C++ implementation, ServerClusterInterface, build integration, Ember migration)](matter/writing-clusters/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/guides/writing_clusters.html |
| [Code Generation (ZAP, .matter IDL, codegen.py, pre-generation, IDL tooling)](matter/code-generation/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/zap_and_codegen/code_generation.html |
| [Testing (unit tests pw_unit_test, Python Mobly integration tests, YAML tests, CI)](matter/testing/SKILL.md) | https://project-chip.github.io/connectedhomeip-doc/testing/index.html |

### FRITZ!Box 7690

| Skill | Source |
|-------|--------|
| [Internet Setup (DSL, VDSL, FTTH/fiber, external modem, cascading)](fritzbox-7690/internet-setup/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [Wi-Fi and Mesh (Wi-Fi 7, mesh networking, guest Wi-Fi, WPS, scheduling)](fritzbox-7690/wifi-and-mesh/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [Telephony (DECT phones, VoIP/SIP, answering machines, call blocking, fax)](fritzbox-7690/telephony/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [Smart Home (Zigbee, DECT ULE/HAN FUN, FRITZ!Smart Gateway, routines)](fritzbox-7690/smart-home/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [VPN (WireGuard remote access and site-to-site, IPSec, MyFRITZ!)](fritzbox-7690/vpn/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [Network Security (Firewall, parental controls, access profiles, device blocking)](fritzbox-7690/network-security/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [NAS and Storage (FRITZ!NAS, USB storage, SMB, media server, remote access)](fritzbox-7690/nas-and-storage/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| [Device Management (FRITZ!OS updates, backup/restore, diagnostics, factory reset)](fritzbox-7690/device-management/SKILL.md) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |

### Watchtower

| Skill | Source |
|-------|--------|
| [Deployment (Docker Compose, label-enable mode, Docker API version fix, scheduling, labeling stacks)](watchtower/deployment/SKILL.md) | https://containrrr.dev/watchtower/ |

### Home Assistant

| Skill | Source |
|-------|--------|
| [Automations (Triggers, Conditions, Actions, Blueprints)](home-assistant/automations/SKILL.md) | https://www.home-assistant.io/docs/automation/ |
| [Scripts (Sequences, Variables, Modes)](home-assistant/scripts/SKILL.md) | https://www.home-assistant.io/docs/scripts/ |
| [Templates (Jinja2, States, Filters, Template Entities)](home-assistant/templates/SKILL.md) | https://www.home-assistant.io/docs/configuration/templating/ |
| [Configuration (configuration.yaml, Secrets, Packages, Includes)](home-assistant/configuration/SKILL.md) | https://www.home-assistant.io/docs/configuration/ |
| [Dashboard (Lovelace Cards, YAML Mode, Custom Cards)](home-assistant/dashboard/SKILL.md) | https://www.home-assistant.io/docs/dashboard/ |
| [Scenes (Definition, Activation, Ad-hoc Apply)](home-assistant/scenes/SKILL.md) | https://www.home-assistant.io/docs/scene/ |
| [Helpers (input_boolean, input_number, input_select, input_text, input_datetime, input_button, counter, timer, schedule)](home-assistant/helpers/SKILL.md) | https://www.home-assistant.io/docs/configuration/helpers/ |
| [Areas & Floors (Rooms, Floors, Labels, Zones)](home-assistant/areas-floors/SKILL.md) | https://www.home-assistant.io/docs/organizing/ |
| [Entities & Device Registry (Entity IDs, States, Renaming, Disabling, Device Management)](home-assistant/entities/SKILL.md) | https://www.home-assistant.io/docs/configuration/customizing-devices/ |
| [MCP Server (ha-mcp: AI control via Claude Desktop, Claude Code, VS Code, Cursor, and 15+ clients)](home-assistant/ha-mcp/SKILL.md) | https://github.com/homeassistant-ai/ha-mcp |
