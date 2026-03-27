## Security Rules

**This is a public GitHub repository. Never commit passwords, tokens, API keys, or other secrets into any file.** If an example requires credentials, use placeholder values like `<your-token>`, `<api-key>`, or `$TOKEN` (shell variable).

This rule applies to:
- API tokens (e.g. NetBird `nbp_...`, GitHub PATs, cloud provider keys)
- Passwords and passphrases
- Private keys or certificates
- Any value that would grant access if disclosed

If a skill currently contains a real credential, remove it immediately, rotate the credential, and replace with a placeholder.


