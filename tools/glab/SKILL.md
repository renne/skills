---
name: glab-cli
description: Using glab CLI to interact with self-hosted GitLab instances — authentication, issues, MRs, notes
---

# glab CLI

`glab` is the official GitLab CLI, analogous to `gh` for GitHub.

## Installation

```bash
sudo apt install glab          # Ubuntu/Debian (may be slightly outdated)
# Or download latest from https://gitlab.com/gitlab-org/cli/-/releases
```

## Authentication

### Login

```bash
glab auth login --hostname <gitlab.example.com>
```

- Choose **Token** when prompted (paste a Personal Access Token)
- Required token scopes: `api`, `write_repository`
- Choose **SSH** as default git protocol (uses existing SSH keys for git ops)
- API calls always use HTTPS + token regardless of git protocol

### Keyring on WSL2

`--use-keyring` fails on WSL2:
```
The name org.freedesktop.secrets was not provided by any .service files
```
Omit `--use-keyring` — token is stored in `~/.config/glab-cli/config.yml` (plain text).

### Verify

```bash
glab auth status --hostname <gitlab.example.com>
```

## Repo targeting

Always specify the full host+namespace+repo when not inside a git repo:

```bash
glab -R gitlab.example.com/namespace/repo <command>
```

Do NOT use `--hostname` on subcommands — that flag only exists on `glab auth`.

## Issue Templates

> **Always follow the project's issue template.**
> Before creating or updating an issue, check whether the project defines issue templates and use the appropriate one to structure the issue body.
>
> ```bash
> # List available issue templates via the GitLab REST API
> curl -s --header "PRIVATE-TOKEN: <token>" \
>   "https://gitlab.example.com/api/v4/projects/<id>/templates/issues" \
>   | jq '.[].name'
>
> # Read a specific template by name
> curl -s --header "PRIVATE-TOKEN: <token>" \
>   "https://gitlab.example.com/api/v4/projects/<id>/templates/issues/<template-name>" \
>   | jq -r '.content'
> ```
>
> Fill in every section of the template. Do not omit or reorder sections.
>
> GitLab issue templates live in `.gitlab/issue_templates/` in the repository.

## Issue commands

```bash
# List issues
glab -R gitlab.example.com/ns/repo issue list

# View issue
glab -R gitlab.example.com/ns/repo issue view <id>

# Post a comment/note
glab -R gitlab.example.com/ns/repo issue note <id> --message "..."

# Create issue
glab -R gitlab.example.com/ns/repo issue create --title "..." --description "..."
```

## MR commands

```bash
glab -R gitlab.example.com/ns/repo mr list
glab -R gitlab.example.com/ns/repo mr view <id>
glab -R gitlab.example.com/ns/repo mr note <id> --message "..."
```

## Wiki

The glab CLI does not have a built-in wiki command. Fetch wiki pages via the GitLab REST API:

```bash
curl -s --header "PRIVATE-TOKEN: <token>" \
  "https://gitlab.example.com/api/v4/projects/<id>/wikis/<slug>"
```

Or read wiki markdown directly via git:

```bash
git clone git@gitlab.example.com:namespace/repo.wiki.git
```

## Common pitfalls

| Problem | Cause | Fix |
|---|---|---|
| `unknown flag: --hostname` on subcommands | `--hostname` only valid for `glab auth` | Use `-R host/ns/repo` instead |
| Copy-paste invisible U+FEFF prefix | Terminal/chat copy artifact | Re-type the command |
| Token auth error after revocation | Old token revoked in GitLab UI | Re-run `glab auth login` |
