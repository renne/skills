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
| Getting Started (Overview, features, plans, quickstart) | https://docs.github.com/en/copilot |
| Chat (IDE, GitHub.com, Mobile, slash commands, MCP, AI models) | https://docs.github.com/en/copilot/concepts/chat |
| Code Suggestions (inline completions, next edit suggestions, supported languages) | https://docs.github.com/en/copilot/concepts/completions/code-suggestions |
| Coding Agent (autonomous PR creation, custom agents, hooks, MCP, security) | https://docs.github.com/en/copilot/concepts/agents/coding-agent/about-coding-agent |
| Customization (custom instructions, prompt engineering, response tuning) | https://docs.github.com/en/copilot/concepts/prompting/response-customization |
| CLI (terminal assistant, installation, modes, shortcuts, slash commands, skills, MCP, LSP) | https://docs.github.com/en/copilot/how-tos/use-copilot-agents/use-copilot-cli |

### LimeSurvey

| Skill | Source |
|-------|--------|
| Contributing (PHP backend, JS frontend, Twig/SCSS themes, OpenAPI, GitHub Actions CI, Cypress E2E, MySQL/PostgreSQL/MSSQL) | https://github.com/LimeSurvey/LimeSurvey |
| Survey Administration (LSS format, groups/branching/questions, RemoteControl API, import/export, permissions) | https://limesurvey.svc.external.tba-hosting.de |

### Moodle

| Skill | Source |
|-------|--------|
| PHP Coding Style | https://moodledev.io/general/development/policies/codingstyle |
| Moodle App Coding Style | https://moodledev.io/general/development/policies/codingstyle-moodleapp |
| Moodle App Development Guide | https://moodledev.io/general/app/development/development-guide |
| Moodle App Plugins Development Guide | https://moodledev.io/general/app/development/plugins-development-guide |
| HVP→H5P Migration (tool_migratehvp2h5p, c=0 vs c=1 permission bug, DB diagnostics, re-migration fix) | https://moodle.org/plugins/tool_migratehvp2h5p |

### Nextcloud

| Skill | Source |
|-------|--------|
| Coding Style & General Guidelines | https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/index.html |
| PHP Coding Standards | https://docs.nextcloud.com/server/stable/developer_manual/getting_started/coding_standards/php.html |
| App Development Guide | https://docs.nextcloud.com/server/19/developer_manual/app/index.html |
| OIDC Identity Provider (user_oidc app, registering clients, oauth2-proxy, email_verified quirk) | https://github.com/nextcloud/user_oidc |

### Docker Compose

| Skill | Source |
|-------|--------|
| Overview & Getting Started | https://docs.docker.com/compose/ |
| Compose File Reference | https://docs.docker.com/reference/compose-file/ |
| How-Tos (Networking, Env Vars, Secrets, Profiles, Watch, Multiple Files) | https://docs.docker.com/compose/how-tos/ |

### CoreDNS

| Skill | Source |
|-------|--------|
| Overview & Configuration (Corefile, server blocks, snippets, env vars) | https://coredns.io/manual/toc/ |
| Plugins Reference (forward, cache, rewrite, file, kubernetes, acl, dnssec, hosts, template, and more) | https://coredns.io/plugins/ |
| Kubernetes (cluster DNS, ConfigMap, stub zones, debugging, monitoring) | https://coredns.io/manual/toc/ |
### NetBird

| Skill | Source |
|-------|--------|
| Getting Started (Installation, CLI, Setup Keys) | https://docs.netbird.io/get-started |
| Access Control (Groups, Policies, Posture Checks) | https://docs.netbird.io/manage/access-control |
| Networks and DNS (Routes, Site-to-Site, Nameservers) | https://docs.netbird.io/manage/networks |
| DNS Management (Nameservers, Custom Zones, DNS Settings, Aliases, Troubleshooting) | https://docs.netbird.io/manage/dns |
| Self-Hosted Deployment (Docker Compose, Config, IdP) | https://docs.netbird.io/selfhosted/selfhosted-quickstart |
| MCP Server (AI Management via mcp-netbird) | https://github.com/XNet-NGO/mcp-netbird |
| DNS + CoreDNS Integration (Custom Zones vs CoreDNS, forwarder, internal zone routing, Corefile patterns) | https://docs.netbird.io/manage/dns |
| SSH Server (dashboard SSH, --allow-server-ssh flags, OAuth2 re-auth root cause, container apply, --disable-ssh-auth) | https://docs.netbird.io/manage/ssh |
| Reverse Proxy (services, targets, domains, auth SSO/password/PIN, TLS, HA, custom domains, backend config, access logs) | https://docs.netbird.io/manage/reverse-proxy |
| Reverse Proxy — Expose from CLI (netbird expose, flags, session lifecycle, troubleshooting) | https://docs.netbird.io/manage/reverse-proxy/expose-from-cli |
| Public API (authentication, accounts, users, tokens, peers, setup keys, groups, networks, policies, routes, DNS, events) | https://docs.netbird.io/api |
| Docker Compose Integration (shared namespace sidecar pattern, subnet routing, 1:N services, port conflicts, --scale limitations) | https://docs.netbird.io/manage/networks |

### Proxmox VE

| Skill | Source |
|-------|--------|
| Installation & Host System Administration (Debian base, networking, packages, disk management) | https://pve.proxmox.com/pve-docs/ |
| QEMU/KVM Virtual Machines (qm CLI, hardware config, cloud-init, PCI passthrough, snapshots) | https://pve.proxmox.com/pve-docs/ |
| LXC Containers (pct CLI, templates, networking, mount points, resource limits) | https://pve.proxmox.com/pve-docs/ |
| Storage (ZFS, Ceph RBD/CephFS, LVM, LVM-Thin, NFS, iSCSI, CIFS, pvesm) | https://pve.proxmox.com/pve-docs/ |
| Cluster Manager, Networking & Firewall (pvecm, SDN, bridges, VLANs, bonds, HA firewall) | https://pve.proxmox.com/pve-docs/ |
| User Management (realms, users, groups, roles, permissions, API tokens, 2FA) | https://pve.proxmox.com/pve-docs/ |
| Backup & Restore (vzdump, backup modes, PBS integration, scheduled jobs, retention) | https://pve.proxmox.com/pve-docs/ |
| High Availability (HA manager, fencing, HA groups, resource states, failover) | https://pve.proxmox.com/pve-docs/ |

### Gramps Web

| Skill | Source |
|-------|--------|
| Setup & Deployment (Docker Compose, configuration, user roles, OIDC, AI chat) | https://www.grampsweb.org/install_setup/setup/ |
| Frontend Development (Lit/LitElement components, API integration, views, state management) | https://github.com/gramps-project/gramps-web |
| REST API (endpoints, authentication, data model, search, LLM/chat) | https://github.com/gramps-project/gramps-web-api |

### Vaultwarden

| Skill | Source |
|-------|--------|
| Installation & Deployment (Docker images, docker run, Docker Compose, Caddy) | https://github.com/dani-garcia/vaultwarden/wiki |
| Configuration (environment variables, ENV_FILE, config precedence, DOMAIN, signups) | https://github.com/dani-garcia/vaultwarden/wiki/Configuration-overview |
| HTTPS & Reverse Proxy (Caddy, nginx, Let's Encrypt, Cloudflare, OCSP) | https://github.com/dani-garcia/vaultwarden/wiki/Enabling-HTTPS |
| Admin Page (ADMIN_TOKEN, argon2id hashing, user management, organizations) | https://github.com/dani-garcia/vaultwarden/wiki/Enabling-admin-page |
| SMTP Configuration (email delivery, Gmail, Outlook, SendGrid, OAuth2 proxy) | https://github.com/dani-garcia/vaultwarden/wiki/SMTP-Configuration |
| Backup & Restore (SQLite backup, data folder, automated backups, restore) | https://github.com/dani-garcia/vaultwarden/wiki/Backing-up-your-vault |

### Anthropics

| Skill | Source |
|-------|--------|
| [Skills directory (algorithmic-art, brand-guidelines, canvas-design, claude-api, doc-coauthoring, docx, frontend-design, mcp-builder, pdf, pptx, webapp-testing, xlsx, and more)](https://github.com/anthropics/skills/tree/main/skills) | https://github.com/anthropics/skills |

### VoltAgent

| Skill | Source |
|-------|--------|
| [Awesome Agent Skills](https://github.com/VoltAgent/awesome-agent-skills) | https://github.com/VoltAgent/awesome-agent-skills |

### Traefik

| Skill | Source |
|-------|--------|
| Overview & Getting Started (core concepts, request lifecycle, installation, router rules) | https://doc.traefik.io/traefik/ |
| Routing (HTTP/TCP/UDP routers, rules, services, load balancing, entrypoints) | https://doc.traefik.io/traefik/routing/overview/ |
| Middlewares (BasicAuth, ForwardAuth, Headers, RateLimit, Compress, StripPrefix, CircuitBreaker, and more) | https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/ |
| HTTPS & TLS (Let's Encrypt ACME, HTTP-01/DNS-01/TLS-ALPN-01 challenges, wildcard certs, TLS options, mTLS) | https://doc.traefik.io/traefik/https/overview/ |
| Providers (Docker labels, Kubernetes IngressRoute CRDs, file provider, Consul/etcd) | https://doc.traefik.io/traefik/providers/overview/ |
| Observability (Prometheus metrics, OpenTelemetry/Jaeger tracing, access logs, API dashboard) | https://doc.traefik.io/traefik/observe/overview/ |

### INWX

| Skill | Source |
|-------|--------|
| DNS and ACME Migration (DomRobot API, delegation changes, DNSSEC recovery, Proxmox/Traefik DNS-01) | https://www.inwx.de/en/help/apicommands |

### Matter (Connected Home over IP)

| Skill | Source |
|-------|--------|
| Overview (architecture, stack layers, nodes, clusters, fabrics, commissioning) | https://project-chip.github.io/connectedhomeip-doc/ |
| Building (GN/ninja, prerequisites, ZAP tool, build_examples.py) | https://project-chip.github.io/connectedhomeip-doc/guides/BUILDING.html |
| Access Control (ACLs, privileges, AuthModes, subjects, targets, chip-tool) | https://project-chip.github.io/connectedhomeip-doc/guides/access-control-guide.html |
| Writing Clusters (C++ implementation, ServerClusterInterface, build integration, Ember migration) | https://project-chip.github.io/connectedhomeip-doc/guides/writing_clusters.html |
| Code Generation (ZAP, .matter IDL, codegen.py, pre-generation, IDL tooling) | https://project-chip.github.io/connectedhomeip-doc/zap_and_codegen/code_generation.html |
| Testing (unit tests pw_unit_test, Python Mobly integration tests, YAML tests, CI) | https://project-chip.github.io/connectedhomeip-doc/testing/index.html |

### FRITZ!Box 7690

| Skill | Source |
|-------|--------|
| Internet Setup (DSL, VDSL, FTTH/fiber, external modem, cascading) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| Wi-Fi and Mesh (Wi-Fi 7, mesh networking, guest Wi-Fi, WPS, scheduling) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| Telephony (DECT phones, VoIP/SIP, answering machines, call blocking, fax) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| Smart Home (Zigbee, DECT ULE/HAN FUN, FRITZ!Smart Gateway, routines) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| VPN (WireGuard remote access and site-to-site, IPSec, MyFRITZ!) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| Network Security (Firewall, parental controls, access profiles, device blocking) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| NAS and Storage (FRITZ!NAS, USB storage, SMB, media server, remote access) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |
| Device Management (FRITZ!OS updates, backup/restore, diagnostics, factory reset) | https://fritz.com/pages/knowledge-base?product=FRITZ-Box-7690 |

### Watchtower

| Skill | Source |
|-------|--------|
| Deployment (Docker Compose, label-enable mode, Docker API version fix, scheduling, labeling stacks) | https://containrrr.dev/watchtower/ |

### Home Assistant

| Skill | Source |
|-------|--------|
| Automations (Triggers, Conditions, Actions, Blueprints) | https://www.home-assistant.io/docs/automation/ |
| Scripts (Sequences, Variables, Modes) | https://www.home-assistant.io/docs/scripts/ |
| Templates (Jinja2, States, Filters, Template Entities) | https://www.home-assistant.io/docs/configuration/templating/ |
| Configuration (configuration.yaml, Secrets, Packages, Includes) | https://www.home-assistant.io/docs/configuration/ |
| Dashboard (Lovelace Cards, YAML Mode, Custom Cards) | https://www.home-assistant.io/docs/dashboard/ |
| Scenes (Definition, Activation, Ad-hoc Apply) | https://www.home-assistant.io/docs/scene/ |
| Helpers (input_boolean, input_number, input_select, input_text, input_datetime, input_button, counter, timer, schedule) | https://www.home-assistant.io/docs/configuration/helpers/ |
| Areas & Floors (Rooms, Floors, Labels, Zones) | https://www.home-assistant.io/docs/organizing/ |
| Entities & Device Registry (Entity IDs, States, Renaming, Disabling, Device Management) | https://www.home-assistant.io/docs/configuration/customizing-devices/ |
| MCP Server (ha-mcp: AI control via Claude Desktop, Claude Code, VS Code, Cursor, and 15+ clients) | https://github.com/homeassistant-ai/ha-mcp |

### Tools

| Skill | Source |
|-------|--------|
| gh-issue (Create, search and manage GitHub issues via gh CLI; OAuth device flow, classic vs fine-grained PAT, WSL2 workaround, --web URL length limit) | https://cli.github.com/manual/gh_issue_create |
| glab CLI (Authentication, issues, MRs, notes on self-hosted GitLab; WSL2 keyring workaround, -R flag usage) | https://gitlab.com/gitlab-org/cli |
| cq (mozilla-ai/cq shared agent knowledge commons: install for Claude Code/OpenCode, deploy team API via Docker Compose, MCP tools query/propose/confirm/flag/reflect/status, agent protocol) | https://github.com/mozilla-ai/cq |

### Hardware

| Skill | Source |
|-------|--------|
| Huananzhi H12D-8D (Single-socket AMD EPYC, memory slot MM1–MM8, DIMM population) | http://www.huananzhi.com/en/list_6/183.html |
| Acer Aspire VN7-593G (Intel i7-7700HQ, GTX 1060, 64GB RAM, 4K IPS, ZFS, Ubuntu) | https://www.acer.com/ac/en/US/content/series/aspirevinitrobeblack |
| Intel NUC7PJYHN (Pentium Silver J5040 Gemini Lake, 16GB, SATA ZFS — pve2) | https://www.intel.com/content/www/us/en/products/sku/126143/intel-nuc-kit-nuc7pjyhn/specifications.html |
| Fujitsu D3643-H1 (Celeron G4900 Coffee Lake, 64GB, 2×4TB NVMe ZFS mirror, I350 — pve3) | https://www.fujitsu.com/global/products/computing/peripheral/accessories/d3643-h1.html |
