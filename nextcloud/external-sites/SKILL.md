---
name: external-sites
description: Working with the Nextcloud External Sites app — configuring entries, managing icons/favicons, and known limitations. Use when asked about embedding external links or icons in Nextcloud navigation.
---
# Nextcloud External Sites App

The **External Sites** app (`nextcloud/external`) adds custom links to the Nextcloud top navigation bar. Each entry can have a name, URL, icon, and display options.

## Icon / Favicon Handling

### Current Behaviour (as of 2026-03)

Icons must be provided **manually**. There is **no built-in auto-fetch** of favicons from the linked website. Options:

| Method | How |
|--------|-----|
| Upload a file | Upload an image via the admin UI |
| Provide a URL | Enter an icon URL directly in the admin UI or via the OCS API |
| Third-party favicon proxy | Use a service URL such as `https://favicon.im/<domain>` or `https://favicon.is/<domain>` as the icon URL |

### Open Feature Request

A feature request for automatic favicon fetching has been filed:
**[nextcloud/external#1051](https://github.com/nextcloud/external/issues/1051)**

The requested behaviour:
1. When an entry is saved, fetch the linked URL's root domain.
2. Detect the favicon via `<link rel="icon">` tags or `/favicon.ico` fallback.
3. Store the resolved icon server-side (to avoid leaking admin browser requests to external sites).
4. Alternatively, expose a "Fetch icon" button in the UI.

### Workaround: Favicon Proxy

Until a native solution exists, use a favicon proxy as the icon URL:

```
https://favicon.im/<domain>
# e.g. https://favicon.im/github.com
```

This is fetched server-side by Nextcloud when rendering the nav icon, so no direct request
is made from the admin's browser to the target site.

## Related Issues

- **#198** (open, enhancement) — Browser tab favicon does not update to match the configured icon when navigating to the embedded page. Separate from auto-fetch; affects the `<iframe>` page title/favicon inheritance.
- **#1051** (open, feature request) — Auto-fetch favicon from linked website URL (filed 2026-03).
