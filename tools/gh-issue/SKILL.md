---
name: gh-issue
description: Create, search, and manage GitHub issues using the gh CLI. Covers authentication (OAuth device flow, classic PAT vs fine-grained PAT), creating issues from a file, pre-filling issue forms in the browser, WSL2 browser/passkey workarounds, and common permission errors. Use when filing a new issue, searching for existing issues, or programmatically managing issues on any GitHub repository.
---
# GitHub Issue Management with `gh`

## Authentication

### Recommended: OAuth Device Flow (no token needed)

```bash
gh auth login --hostname github.com --git-protocol ssh --web
```

- Select **Skip** when asked to upload an SSH key (unless you want to).
- A **one-time code** is printed (e.g. `43FE-530A`).
- Open **https://github.com/login/device** in a browser and enter the code.
- Authorize — `gh` detects approval automatically and saves the OAuth token.

#### ⚠️ WSL2 + Windows Passkey

`gh auth login --web` opens a browser **inside WSL2**, which cannot access the Windows passkey store. Instead:

1. Note the one-time code printed in the terminal.
2. Open **https://github.com/login/device** manually in the **Windows** browser.
3. Enter the code and authorize with Windows Hello / passkey.
4. The `gh` process in WSL2 detects the approval automatically.

### Alternative: Classic PAT

Fine-grained PATs **cannot create issues on repositories you do not own**, even with Issues: Read and write permission granted.

Use a **classic PAT** with the `public_repo` scope for public repos:

```bash
gh auth login --with-token <<< "ghp_YOUR_CLASSIC_TOKEN"
```

⚠️ Never commit tokens to files. Revoke any token that was accidentally pasted into a chat or shared in plain text.

---

## Creating an Issue

> **Always follow the project's issue template.**
> Before creating or updating an issue, check whether the repository defines issue templates and use the appropriate one to structure the issue body.
>
> ```bash
> # List available templates
> gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE --jq '.[].name'
>
> # Read a specific template (e.g. bug_report.md)
> gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE/bug_report.md \
>   --jq '.content' | base64 -d
> ```
>
> Fill in every section of the template. Do not omit or reorder sections.

### From a Markdown file (recommended for long bodies)

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "Your issue title" \
  --body-file issue-body.md \
  --label "feature-request"
```

### Inline body

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "Your issue title" \
  --body "Short description here." \
  --label "bug"
```

### Open browser form pre-filled (`--web`)

```bash
gh issue create \
  --repo <owner>/<repo> \
  --title "Your issue title" \
  --body-file issue-body.md \
  --label "feature-request" \
  --web
```

⚠️ The `--web` flag encodes the body into the URL. If the body is longer than ~4000 characters, GitHub will show **"Whoops, something went wrong!"**. Use direct `gh issue create` (without `--web`) for long issue bodies and edit on GitHub afterwards if needed.

---

## Searching for Existing Issues

```bash
# Search issues by keyword
gh issue list --repo <owner>/<repo> --search "traefik labels expose" --state open

# Or use the API search
gh api "search/issues?q=repo:<owner>/<repo>+traefik+labels&type=issues" \
  --jq '.items[] | "#\(.number) [\(.state)] \(.title)\n  \(.html_url)"'
```

---

## Issue Templates

Check what templates a repo provides:

```bash
gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE \
  --jq '.[].name'
```

Read the feature request template:

```bash
gh api repos/<owner>/<repo>/contents/.github/ISSUE_TEMPLATE/feature_request.md \
  --jq '.content' | base64 -d
```

---

## Known Quirks and Limitations

- ⚠️ **Fine-grained PATs cannot create issues on third-party repos.** Even with "All repositories" and "Issues: Read and write", `createIssue` returns `Resource not accessible by personal access token`. Use a classic PAT with `public_repo` or OAuth instead.
- ⚠️ **`--web` URL length limit.** GitHub issue new-form URLs longer than ~4000–5000 characters cause a generic error page. Use `--body-file` without `--web` for substantial issue bodies.
- ⚠️ **WSL2 browser cannot use Windows passkeys.** Open https://github.com/login/device in the Windows host browser when using the device flow from WSL2.
- The `admin:public_key` scope is required to upload SSH keys via PAT. Missing this scope causes a 403 during `gh auth login` but does not affect issue creation — authentication still succeeds for other operations.

## References

- [GitHub CLI manual — gh issue create](https://cli.github.com/manual/gh_issue_create)
- [GitHub CLI manual — gh auth login](https://cli.github.com/manual/gh_auth_login)
- [GitHub docs — Fine-grained PATs](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens#fine-grained-personal-access-tokens)
- [GitHub docs — Device authorization flow](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps#device-flow)
