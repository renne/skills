---
name: frontend-development
description: Gramps Web frontend development guide covering the Lit/LitElement component architecture, app state management, API integration, adding new views and components, coding conventions, and local development setup. Use when contributing to the Gramps Web frontend, creating new views/components, or understanding the frontend codebase.
---
# Gramps Web – Frontend Development

Source: https://github.com/gramps-project/gramps-web

## Tech Stack

- **[Lit](https://lit.dev/) / LitElement** – Web components framework (ES modules, no build step in development)
- **[Material Web Components](https://github.com/material-components/material-web)** (`@material/mwc-*`, `@material/web`) – UI component library
- **[D3](https://d3js.org/)** – Data-driven charts and genealogical tree visualizations
- **[MapLibre GL](https://maplibre.org/)** – Interactive maps for place visualizations
- **[Rollup](https://rollupjs.org/)** – Production bundler
- **Prettier + ESLint** – Code formatting and linting (`@open-wc/eslint-config`)

## Local Development Setup

```bash
# Clone both repos side-by-side
git clone https://github.com/gramps-project/gramps-web
cd gramps-web

# Install dependencies
npm install

# Start the dev server (requires a running backend)
npm start
# → available at http://localhost:8001

# Or use the Docker Compose development setup (starts API + frontend hot-reload):
docker compose up
# → available at http://localhost:5555
```

The Docker Compose dev setup (`docker-compose.yml`) proxies the frontend dev server through nginx alongside a pre-seeded demo API backend, so you can work on UI without setting up a full Gramps database.

```bash
# Lint
npm run lint

# Run tests
npm test

# Production build (output in dist/)
npm run build
```

## Project Layout

```
src/
├── GrampsJs.js          # Root application element (<gramps-js>)
├── api.js               # Auth token management and API helpers
├── appState.js          # Centralised application state definition
├── SharedStyles.js      # Shared CSS (token-based design system)
├── strings.js           # i18n string keys
├── icons.js             # MDI icon SVG paths
├── util.js              # Utility helpers (fireEvent, etc.)
├── components/          # Reusable UI components
├── views/               # Full-page view components
├── charts/              # D3 genealogical chart components
└── mixins/              # Reusable LitElement mixins
```

## Component Architecture

### Base Classes and Mixins

All views and most components inherit from two base classes:

1. **`GrampsjsAppStateMixin`** – Adds the `appState` property and the `_(key)` i18n helper method.
2. **`GrampsjsView`** (in `src/views/GrampsjsView.js`) – Extends `GrampsjsAppStateMixin(LitElement)`. Provides `active`, `loading`, `error` properties and fires `progress:on`/`progress:off` events.

### Naming Convention

All components are named with a `Grampsjs` prefix:
- **Views** (full pages): `GrampsjsViewPerson`, `GrampsjsViewPeople`, …
- **Components** (reusable): `GrampsjsPersonCard`, `GrampsjsTimeline`, …
- **Mixins**: `GrampsjsAppStateMixin`, `GrampsjsStaleDataMixin`, …

Custom element tag names use kebab-case: `grampsjs-view-person`, `grampsjs-person-card`.

### Minimal View Example

```javascript
// src/views/GrampsjsViewExample.js
import {html, css} from 'lit'
import {GrampsjsView} from './GrampsjsView.js'
import {apiGet} from '../api.js'

export class GrampsjsViewExample extends GrampsjsView {
  static get styles() {
    return [
      ...super.styles,
      css`
        h2 {
          color: var(--md-sys-color-primary);
        }
      `,
    ]
  }

  static get properties() {
    return {
      ...super.properties,
      _data: {type: Array},
    }
  }

  constructor() {
    super()
    this._data = []
  }

  connectedCallback() {
    super.connectedCallback()
    this._fetchData()
  }

  async _fetchData() {
    this.loading = true
    const result = await apiGet('/api/people/?keys=handle,name_given,name_surname')
    this.loading = false
    if (result.ok) {
      this._data = result.data
    } else {
      this.error = true
      this._errorMessage = result.statusText
    }
  }

  renderContent() {
    return html`
      <h2>${this._('People')}</h2>
      ${this._data.map(
        person => html`<p>${person.name_given} ${person.name_surname}</p>`
      )}
    `
  }
}

customElements.define('grampsjs-view-example', GrampsjsViewExample)
```

### Minimal Reusable Component Example

```javascript
// src/components/GrampsjsExampleCard.js
import {LitElement, html, css} from 'lit'
import {GrampsjsAppStateMixin} from '../mixins/GrampsjsAppStateMixin.js'
import {sharedStyles} from '../SharedStyles.js'

export class GrampsjsExampleCard extends GrampsjsAppStateMixin(LitElement) {
  static get styles() {
    return [sharedStyles, css`:host { display: block; }`]
  }

  static get properties() {
    return {
      ...super.properties,
      handle: {type: String},
      label: {type: String},
    }
  }

  constructor() {
    super()
    this.handle = ''
    this.label = ''
  }

  render() {
    return html`<span>${this._(this.label)}</span>`
  }
}

customElements.define('grampsjs-example-card', GrampsjsExampleCard)
```

## Application State (`appState`)

The root `<gramps-js>` element maintains centralised state and passes it down as the `appState` property to all child views and components via Lit property bindings.

Key `appState` fields:

| Field | Type | Description |
|-------|------|-------------|
| `appState.i18n.strings` | Object | i18n translation strings map |
| `appState.permissions` | Object | Current user's permissions (`canAdd`, `canEdit`, `canUseChat`, …) |
| `appState.dbInfo` | Object | Backend metadata (Gramps version, tree name, feature flags) |
| `appState.dbInfo.server.chat` | Boolean | Whether AI chat is enabled on the server |
| `appState.settings` | Object | Per-tree user settings stored in `localStorage` |
| `appState.homePersonHandle` | String | Handle of the tree's home person |

Access in any component via `this.appState`:
```javascript
const canEdit = this.appState.permissions?.canEdit
const treeName = this.appState.dbInfo?.database?.name
```

### i18n

Use `this._(key)` (provided by `GrampsjsAppStateMixin`) to translate strings:
```javascript
// Simple
html`<span>${this._('Add person')}</span>`

// With substitution (%s)
html`<span>${this._('Hello %s', userName)}</span>`
```

String keys are defined in `src/strings.js`.

## API Layer (`src/api.js`)

All REST API calls go through helpers in `src/api.js`. The JWT access token is read from `localStorage` and injected automatically.

### Core Helpers

```javascript
import {apiGet, apiPost, apiPut, apiDelete, apiPatch} from '../api.js'

// GET /api/people/?page=1&pagesize=20
const result = await apiGet('/api/people/', {page: 1, pagesize: 20})
if (result.ok) {
  const people = result.data   // parsed JSON
  const total = result.total   // X-Total-Count header value
}

// POST /api/people/
const result = await apiPost('/api/people/', {
  _class: 'Person',
  primary_name: {_class: 'Name', first_name: 'John', ...}
})

// PATCH (update) /api/people/<handle>
const result = await apiPatch(`/api/people/${handle}`, changes)

// DELETE /api/people/<handle>
const result = await apiDelete(`/api/people/${handle}`)
```

All helpers return an object `{ok, status, statusText, data, total}`.

### Auth Helpers

```javascript
import {
  getTreeId,
  getPermissions,
  getSettings,
  updateSettings,
  doLogout,
} from '../api.js'

const treeId = getTreeId()          // from JWT claims
const perms = getPermissions()      // from JWT claims
const settings = getSettings()      // from localStorage
```

## Events

Gramps Web uses custom DOM events for cross-component communication. Fire events using `fireEvent` from `src/util.js`:

```javascript
import {fireEvent} from '../util.js'

// Dispatch a data-update notification (causes global refresh)
fireEvent(this, 'db:changed')

// Show a snackbar notification
fireEvent(this, 'grampsjs:notification', {message: 'Saved!', status: 'success'})

// Show an error
fireEvent(this, 'grampsjs:error', {message: 'Something went wrong'})

// Update URL/navigation
fireEvent(this, 'nav', {path: `/person/${handle}`})
```

Common events:

| Event | Description |
|-------|-------------|
| `db:changed` | Database was modified; triggers global data refresh |
| `progress:on` / `progress:off` | Show/hide the linear progress indicator |
| `grampsjs:notification` | Show a snack-bar notification |
| `grampsjs:error` | Show an error notification |
| `nav` | Programmatic navigation |
| `user:loggedout` | User was logged out |
| `settings:changed` | User settings changed |

## Coding Conventions

- **Single quotes**, no semicolons, 2-space indentation (enforced by Prettier)
- Arrow functions: avoid parentheses for single parameters (`x => x + 1`)
- **No unused variables or imports** (ESLint will catch them)
- CSS uses **CSS custom properties** from the Material Design token system (e.g. `var(--md-sys-color-primary)`)
- Templates use **tagged template literals** (`html\`...\``, `css\`...\``)
- Properties must be declared in the static `properties` getter; always call `super()` in constructor and always initialize all properties
- Use `connectedCallback()` + `firstUpdated()` to fetch data when a view becomes active
- Fire `grampsjs:error` events instead of `console.error` for user-facing errors

### File Structure for New Components

1. Create `src/components/GrampsjsMyComponent.js` or `src/views/GrampsjsViewMyPage.js`
2. Register the custom element at the bottom: `customElements.define('grampsjs-my-component', GrampsjsMyComponent)`
3. Import and use in the parent view or in `src/GrampsJs.js` for top-level views
4. Add any new i18n strings to `src/strings.js`

## References

- [Gramps Web Frontend Repository](https://github.com/gramps-project/gramps-web)
- [Gramps Web Developer Documentation](https://www.grampsweb.org/development/dev/)
- [Lit Documentation](https://lit.dev/docs/)
- [Material Web Components](https://github.com/material-components/material-web)
- [Open WC Guides](https://open-wc.org/guides/)
