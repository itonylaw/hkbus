# CLAUDE.md — HKBus Codebase Guide

## Project Overview

**HKBus** (九巴到站) is a zero-dependency, single-page mobile web application for checking real-time Kowloon Motor Bus (KMB) arrival times in Hong Kong. The entire application is a single `index.html` file (~863 lines) with embedded CSS and JavaScript — there is no build system, no package manager, and no server-side code.

---

## Repository Structure

```
/
├── index.html      # The entire application (HTML + CSS + JS, ~863 lines)
└── README.md       # Minimal project description
```

That's it. There are no subdirectories, no configuration files, no dependencies, and no build artifacts.

---

## Technology Stack

| Concern        | Technology                                  |
|----------------|---------------------------------------------|
| Language       | Vanilla HTML5 / CSS3 / JavaScript (ES2017+) |
| Fonts          | Google Fonts CDN (DM Mono, Noto Sans HK)    |
| Data source    | Hong Kong KMB ETA public API                |
| Persistence    | Browser `localStorage`                      |
| Build system   | None — open `index.html` directly           |
| Package manager| None                                        |
| Test framework | None                                        |
| CI/CD          | None                                        |

---

## Running the Application

Open `index.html` directly in any modern browser. No server, no install step, no build step required. For the API calls to work, internet access is needed (CORS is open on the public API).

Alternatively, serve it with any static file server:
```bash
python3 -m http.server 8080
# then visit http://localhost:8080
```

---

## Application Architecture

### Single-File Structure

`index.html` is divided into three logical sections:

1. **`<style>` (lines 9–483)** — All CSS, including CSS custom properties (design tokens), component styles, animations, and scrollbar customisation.
2. **`<body>` HTML (lines 486–539)** — Structural markup for the 4-screen app: screen containers, static elements, and the header.
3. **`<script>` (lines 541–861)** — All application logic: state management, navigation, API calls, DOM manipulation, and localStorage persistence.

### Screen Flow (4-step wizard)

```
s1 (Route input) → s2 (Direction) → s3 (Stop select) → s4 (ETA display)
```

Navigation is stack-based (`screenHistory` array). The back button pops the stack. Screen transitions use CSS class swapping (`active`, `exit-left`, `entry-right`) with a 0.28s cubic-bezier animation.

### Global State Object

```javascript
let state = {
  route: '',          // e.g. "74B"
  direction: '',      // "outbound" or "inbound"
  serviceType: '1',   // KMB service type identifier
  stopId: '',         // KMB stop UUID
  stopName: '',       // Stop name in Traditional Chinese
  destTc: '',         // Destination in Traditional Chinese
  routeData: [],      // All route variants for the searched route number
  stopData: [],       // Stop objects with enriched info
  etaInterval: null   // setInterval handle for ETA polling
};
```

All mutable application state lives in this one object. There is no framework, no reactive system — DOM is updated imperatively.

---

## External API

**Base URL:** `https://data.etabus.gov.hk/v1/transport/kmb`

| Endpoint | Purpose |
|----------|---------|
| `GET /route` | List all routes (used to find direction variants) |
| `GET /route/{route}/outbound/1` | Validate that a route exists |
| `GET /route-stop/{route}/{direction}/{serviceType}` | Get ordered stop list for a route |
| `GET /stop/{stopId}` | Get stop details (name, coordinates) |
| `GET /eta/{stopId}/{route}/{serviceType}` | Get real-time ETA data |

**Key API data fields:**
- `bound`: `"O"` = outbound (去程), `"I"` = inbound (回程)
- `dir`: same `"O"/"I"` filter on ETA responses
- `eta`: ISO 8601 timestamp string or `null`
- `rmk_tc`: Remark text in Traditional Chinese (e.g. "原定班次")
- `name_tc`: Stop name in Traditional Chinese
- `dest_tc`: Destination name in Traditional Chinese

---

## Key Conventions

### Naming

- **JS variables/functions:** camelCase (`screenHistory`, `loadETA`, `fetchSavedETA`)
- **CSS classes:** kebab-case (`.save-btn`, `.eta-row`, `.dir-card`)
- **HTML IDs:** short, sometimes single-character prefix (screens: `s1`–`s4`; stop items: `si_<stopId>`; ETA badges: `badge_<idx>`)
- **Section comments in JS:** `// ─── Section Name ───...` dividers to visually separate logical blocks

### Code Style

- **Async/await** for all API calls with try/catch error handling
- **Inline event handlers** on HTML elements (`onclick`, `oninput`, `onkeydown`) — do not refactor to `addEventListener` without good reason
- **Template literals** for dynamic HTML construction in JS
- **No abstraction for one-off logic** — the code is intentionally direct and flat

### CSS Design Tokens (CSS Custom Properties)

All colours and key values are defined on `:root`:

| Variable        | Usage                                        |
|-----------------|----------------------------------------------|
| `--bg`          | Page background (`#0d0f0e`)                  |
| `--bg2`         | Card/input background (`#161918`)            |
| `--bg3`         | Hover state background (`#1e2220`)           |
| `--amber`       | Primary accent, route numbers (`#f5a623`)    |
| `--amber-dim`   | Amber focus/hover borders (`#a06a10`)        |
| `--text`        | Primary text (`#e8e4dc`)                     |
| `--text-muted`  | Secondary/hint text (`#7a7872`)              |
| `--text-dim`    | Placeholder / disabled text (`#3d3d39`)      |
| `--green`       | ETA > 3 min (`#3ddc84`)                      |
| `--amber`       | ETA 1–3 min (reused, with `pulse` animation) |
| `--red`         | ETA ≤ 0 min / errors (`#ff5f5f`)             |
| `--border`      | Card borders (`#252824`)                     |

### ETA Colour Logic

```
diff <= 0 min  → "即將" (arriving), red + 0.7s pulse animation
diff 1–3 min   → "{n} 分鐘", amber + 1s pulse animation
diff > 3 min   → "{n} 分鐘", green, no animation
no eta data    → show rmk_tc remark or "–"
```

### LocalStorage

- Key: `kmb_saved`
- Format: JSON array of saved route objects (max 10 entries)
- Each entry: `{ route, direction, serviceType, stopId, stopName, destTc }`
- Most-recent save is prepended (index 0); oldest are trimmed beyond 10

---

## Performance Patterns

### Batch Stop Name Fetching (Screen 3)

Stop names are fetched in parallel batches of 8 to avoid overwhelming the API while still being fast. Placeholder items (showing the raw stop ID) render immediately, then update in-place as each batch resolves:

```javascript
const batchSize = 8;
for (let i = 0; i < routeStops.length; i += batchSize) {
  const batch = routeStops.slice(i, i + batchSize);
  await Promise.all(batch.map(item => fetch(...).then(d => updateDOM(item.stop, d))));
}
```

### ETA Auto-Refresh

On Screen 4, `loadETA()` is called immediately and then every 30 seconds via `setInterval`. The interval is cleared when navigating back from Screen 4 (`goBack()` checks `cur === 's4'`). Always clear `state.etaInterval` before setting a new one to prevent double-polling.

---

## UI/UX Conventions

- **Language:** Traditional Chinese (zh-HK) throughout the interface
- **Max width:** 480px — mobile-first, centred on desktop
- **Fonts:** `DM Mono` for numbers/codes/labels; `Noto Sans HK` for Chinese text
- **Dark theme only** — no light mode toggle
- **Input auto-uppercase:** Route input forces uppercase via `oninput` and `autocapitalize="characters"`
- **Touch targets:** Minimum 14px padding on interactive elements

---

## What NOT to Do

- Do not introduce a build system, bundler, or package.json unless the project scope fundamentally changes.
- Do not add a JavaScript framework (React, Vue, etc.) — the current vanilla approach is intentional for simplicity and zero-dependency deployment.
- Do not split the single file into multiple files without a clear reason and a serving strategy.
- Do not add error handling for impossible states or defensive code for internal invariants.
- Do not add TypeScript without setting up a proper build pipeline.
- Do not commit `node_modules`, build output, or `.env` files.

---

## Making Changes

Since there is no build step:

1. Edit `index.html` directly.
2. Refresh the browser to test.
3. For CSS changes, the `:root` custom properties are the single source of truth for colours — prefer updating tokens over hardcoding values.
4. For new screens, follow the existing pattern: add a `<div class="screen entry-right" id="sN">` in the HTML, add a step label in the `steps` map in `updateHeader()`, and manage screen transitions via `showScreen('sN')`.
5. For new API calls, follow the async/await + try/catch pattern used in `loadStops()` and `loadETA()`.

---

## Git Workflow

The project uses GitHub with a `main` branch. Development branches follow the pattern `claude/<description>-<id>`. There are no branch protection rules documented, no required reviewers, and no CI checks.

Commit messages are plain English imperative sentences (e.g. "Add HKBus section to README").
