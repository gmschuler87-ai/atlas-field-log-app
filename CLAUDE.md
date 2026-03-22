# CLAUDE.md — Atlas Daily Field Log App

## Overview

This is a **zero-dependency, single-file Progressive Web App (PWA)** for Atlas field technicians. The entire application lives in `index.html` (~209 KB, ~2063 lines). There is no build system, no npm, no backend, and no framework. All state persists in the browser's `localStorage`.

---

## Repository Structure

```
atlas-field-log-app/
├── index.html    # Complete application (HTML + CSS + JS in one file)
└── README.md     # Minimal title-only readme
```

No `package.json`, no `node_modules`, no config files of any kind.

---

## Technology Stack

| Layer        | Technology                              |
|--------------|----------------------------------------|
| Language     | Vanilla JavaScript (ES5-compatible)     |
| UI           | HTML5 + CSS3 (no framework)            |
| Persistence  | `localStorage` (no backend)            |
| State        | Global JS objects                      |
| External API | Open-Meteo (weather, optional)          |
| Build        | None — open `index.html` directly      |
| Tests        | None                                    |
| PWA          | Mobile-installable (meta tags only)    |

---

## Running the App

No build step is needed. Open `index.html` directly in a browser:

```bash
# Option 1: open directly
open index.html

# Option 2: serve locally (avoids potential localStorage CORS quirks)
python3 -m http.server 8080
# then navigate to http://localhost:8080
```

---

## Architecture

### File Layout Inside `index.html`

The file is structured as:

1. **`<head>`** — meta tags (PWA, viewport, theme-color), and `<style>` block with all CSS
2. **`<body>`** — minimal shell div (`#app`)
3. **`<script>`** — all application JavaScript

### Global State (lines ~175–182)

```js
var activeLid = null;      // ID of currently displayed log tab
var logOrder  = [];         // Ordered array of log IDs
var logs      = {};         // Map of lid → log data object
var library   = [];         // Archived/generated reports

var store = {
  names: { atlas: [], sub: [], visitor: [], client: [] },
  visitorOrgs: {},
  affiliations: {},
  projects: [],
  fieldOptions: { proj:[], phase:[], loc:[], well:[], miles:[], atlasscope:[], contractorscope:[] },
  equip: { items: [...], checked: [...] },
  quickPhrases: [...]
};
```

### localStorage Keys

| Key           | Contents                                  |
|---------------|-------------------------------------------|
| `atlas-store` | Serialized `store` object (config/templates) |
| `atlas-logs`  | Serialized `logs` + `logOrder`            |
| `atlas-library` | Serialized `library` array              |
| `atlas-dark`  | `'1'` if dark mode is enabled             |

### Persistence Functions (lines ~317–360)

```js
async function loadStore()    // Read store from localStorage on boot
async function persist()      // Write store to localStorage
async function saveLogs()     // Write logs to localStorage
async function saveLibrary()  // Write library to localStorage
```

All storage calls are wrapped with a `storageSet`/`storageGet` abstraction that falls back gracefully when `localStorage` quota is exceeded.

---

## Three-Stage Workflow

Each log tab has three stages navigable via stage tabs:

### Stage 1 — Field Notes
Data entry: date, project ID, phase, office, project manager, location, well, work scopes, Atlas/sub/visitor crew, equipment checklist, weather (optional API fetch), site conditions, timestamped log entries, photos (base64 in localStorage), and quick-phrase chips.

### Stage 2 — Review
Each entry is shown as a card (`ri` class). Users mark entries as **kept**, **modified**, or **discarded**. Photos can be captioned here.

### Stage 3 — Final Report
`buildReport(lid)` (line ~945) generates a full HTML report in a new tab/window. The report can be printed to PDF (`exportPDF(lid)`, line ~1905) or emailed via native `mailto`.

---

## Key Functions Reference

| Function | Line | Purpose |
|---|---|---|
| `loadStore()` | ~317 | Boot: load config from localStorage |
| `persist()` | ~334 | Save store to localStorage |
| `saveLogs()` | ~336 | Save all log data to localStorage |
| `createLog(label, projData)` | ~1297 | Create a new log tab |
| `buildLogDOM(lid, projData)` | ~1305 | Render full log UI into DOM |
| `makeEntryRow(timeVal, noteVal)` | ~1488 | Create a timestamped note row |
| `fetchWeather(lid)` | ~730 | Call Open-Meteo API for weather data |
| `skipReview(lid)` | ~794 | Advance log to review stage |
| `buildReport(lid)` | ~945 | Generate final HTML report |
| `saveToLibrary(lid, html, isDraft)` | ~1125 | Archive report to library |
| `exportPDF(lid)` | ~1905 | Trigger browser print dialog for PDF |
| `showConfirm(msg, onYes)` | ~205 | Generic confirm modal |
| `showAlert(msg, onClose)` | ~215 | Generic alert modal |
| `goStage(lid, n)` | — | Navigate to stage 1/2/3 for a log |
| `renderTabs()` | — | Re-render the top log tab bar |
| `switchToLog(lid)` | — | Activate a log tab |

---

## DOM Helpers (lines ~184–202)

Short utility functions used throughout for DOM construction:

```js
el(tag, cls)         // createElement with optional className
div(cls)             // shorthand for el('div', cls)
btn(txt, cls)        // <button class="btn [cls]">
inp(val, ph, cap)    // <input> with optional auto-capitalize binding
ta(val, rows, ph)    // <textarea>
lbl(txt)             // <label>
fg(labelTxt, child)  // .fg wrapper div (label + input)
uid()                // generate unique ID (timestamp + random)
esc(s)               // HTML-escape a string
fv(lid, field)       // get value of input #l{lid}-{field}
gv(id)               // get value of input by id
```

---

## CSS Conventions

All styles are in the `<style>` block in `<head>`. CSS uses:

- **CSS custom properties** for theming:
  - `--color-text-primary` / `--color-text-secondary`
  - `--color-background-primary` / `--color-background-secondary` / `--color-background-tertiary`
  - `--color-border-secondary` / `--color-border-tertiary`
- **Dark mode**: toggled by adding class `dark` to `<html>`, persisted to `localStorage`
- **Brand color**: `#1a3a5c` (deep navy — used for active tabs, banners, primary buttons)
- **Green accent**: `#3B6D11` / `#5a9e3a` (kept entries, success states)
- **Filled fields** get class `has-value` → green border + light green background
- **`.card`** — the main content container (white, rounded, bordered)
- **`.btn-primary`** — navy background, white text
- **`.btn-danger`** — red text, red border, red hover background

### Element ID Conventions

Log-scoped elements follow the pattern:
```
l{lid}-{fieldname}
```
For example: `l1a2b3c-proj`, `l1a2b3c-date`, `l1a2b3c-stage-1`, `l1a2b3c-visitor-crew`.

---

## External APIs

Only used for optional weather fetching (not required for core functionality):

```
GET https://geocoding-api.open-meteo.com/v1/search?name={location}&count=1
GET https://api.zippopotam.us/us/{zipcode}          (fallback geocoding)
GET https://api.open-meteo.com/v1/forecast?latitude=...&longitude=...&...
```

No API keys required. All calls are fire-and-forget with user-visible status messages.

---

## Photo Handling

Photos are stored as **base64 data URLs** directly in `localStorage`. Because this can fill quota quickly:
- A storage quota warning is shown if saving photos approaches the limit
- Photos are attached to log entries as `logs[lid].photos` array
- Captions are set in the Review stage via `showPhotoLinkModal()`

---

## Development Guidelines

### Making Changes
- **Edit `index.html` only.** There are no other source files.
- The CSS and JS are intentionally kept in one file for zero-dependency portability.
- Keep JavaScript **ES5-compatible** (no arrow functions, `const`/`let`, template literals, etc.) unless you are certain the target browsers all support modern JS. The existing code uses `var`, `function`, and string concatenation throughout.
- Do not add a build system, npm, or external dependencies without explicit agreement — the single-file constraint is intentional.

### State Mutations
- Always call `persist()` after mutating `store`
- Always call `saveLogs()` after mutating `logs`
- Always call `saveLibrary()` after mutating `library`

### Adding UI
- Reuse the existing DOM helpers (`div`, `btn`, `inp`, etc.)
- Apply existing CSS classes rather than adding inline styles where possible
- Log-scoped element IDs must follow the `l{lid}-{fieldname}` pattern
- Use `has-value` class on inputs to show the green-filled state

### Modal Pattern
Use `showConfirm(msg, callback)` or `showAlert(msg, callback)` for user prompts rather than `window.confirm` / `window.alert`.

### No Testing Framework
There are no automated tests. Test changes manually in a browser. Verify:
1. Data survives a page reload (localStorage round-trip)
2. All three stages render correctly for new and loaded logs
3. Report generation produces valid HTML
4. Dark mode toggle works

---

## Git Workflow

Active development branch: `claude/add-claude-documentation-q8eQ0`

```bash
git add index.html CLAUDE.md
git commit -m "Description of change"
git push -u origin claude/add-claude-documentation-q8eQ0
```

The `master` branch tracks the canonical upstream. The `main` remote branch is the GitHub default.

---

## Deployment

Deploy by copying `index.html` to any static file host (GitHub Pages, S3, Netlify, a local server). No server-side processing, databases, or environment variables are required.
