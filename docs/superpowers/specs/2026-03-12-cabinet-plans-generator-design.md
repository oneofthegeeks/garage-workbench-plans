# Cabinet Plans Generator — Design Spec

**Date:** 2026-03-12
**Status:** Approved for implementation
**Repo:** New repo (separate from garage-workbench-plans)
**Hosting:** GitHub Pages (static, no backend)

---

## Overview

A React + Vite single-page application that guides a user through a 4-step wizard to define a custom base cabinet wall. As the user fills in each step, a live SVG front elevation updates in real time on the right. At the end the app generates a printable blueprint-style plans page (same aesthetic as the garage-workbench-plans site), a downloadable CSV cut list, and a shareable URL that encodes the full project config in the URL hash.

No backend. No accounts. No build-time data. Everything runs in the browser.

---

## Tech Stack

| Layer | Choice | Reason |
|-------|--------|--------|
| Framework | React 18 + Vite | Component model suits a multi-step wizard with live preview; Vite builds to static files for GitHub Pages |
| Language | JavaScript (ES2022) | No build complexity; user can edit directly |
| CSS | CSS Modules | Scoped styles per component; no extra dependencies |
| State | Custom hook (`useCabinetConfig`) | Linear wizard flow doesn't need a state library |
| Deployment | GitHub Pages via `gh-pages` npm package | One command to deploy |

---

## Application Layout

```
┌─────────────────────────────────────────────────────┐
│  TOPBAR: title · step indicators · project name     │
├───────────────────────┬─────────────────────────────┤
│                       │                             │
│   WIZARD PANEL        │   LIVE PREVIEW PANEL        │
│   (380px fixed)       │   (remaining width)         │
│                       │                             │
│   Current step form   │   ElevationSVG(config)      │
│   + step controls     │   updates on every change   │
│                       │                             │
│                       ├─────────────────────────────┤
│                       │  OUTPUT BAR                 │
│                       │  Share · CSV · Generate     │
├───────────────────────┴─────────────────────────────┤
│  WIZARD FOOTER: Back · Next (or Generate on step 4) │
└─────────────────────────────────────────────────────┘
```

---

## Data Model

The `config` object is the single source of truth. It flows from `useCabinetConfig` → wizard step components → pure calculators → outputs.

```js
{
  meta: {
    name: String,          // e.g. "Garage Workbench"
    preset: String | null, // preset key used, or null
  },
  dimensions: {
    wallWidth: Number,     // inches — target total width
    surfaceHeight: Number, // inches, default 36
    depth: Number,         // inches, default 30
    toeKickHeight: Number, // inches, default 4
    topThickness: Number,  // inches, default 1.5
  },
  materials: {
    box: String,           // e.g. "3/4\" MDF"
    top: String,           // e.g. "3/4\" plywood (2 layers)"
    toeKick: String,       // e.g. "3/4\" plywood"
    drawerBox: String,     // e.g. "1/2\" plywood"
  },
  sections: [
    {
      id: String,          // stable UUID for React keys
      name: String,        // user-defined label
      width: Number,       // inches
      color: String,       // hex color, used in SVG and UI
      interior: {
        type: String,      // "drawers" | "shelves" | "open" | "custom"
        // type === "drawers":
        drawers: [{ height: Number }],
        // type === "shelves":
        shelfCount: Number,
        shelfMount: String, // "pocket-screw" | "shelf-pins"
        // type === "open":
        label: String,
        // type === "custom":
        label: String,
        surfaceOffset: Number, // inches, negative = lower (e.g. -3 for miter bay)
        openBelow: Boolean,
        note: String,
      }
    }
  ]
}
```

**Invariant:** `sections.reduce((sum, s) => sum + s.width, 0) === dimensions.wallWidth`
The wizard enforces this — Step 2 cannot advance until the total matches.

---

## Component Tree

```
App
├── LandingScreen          (shown before wizard if no URL hash)
│   └── PresetCard × 4
└── WizardShell            (shown when wizard is active)
    ├── TopBar
    ├── WizardPanel
    │   ├── StepBasics
    │   ├── StepSections
    │   │   ├── SectionRow (draggable)
    │   │   └── AddSectionModal
    │   ├── StepDetails
    │   │   └── SectionDetailForm (one per section, paginated)
    │   └── StepReview
    │       └── CutListPreview
    ├── PreviewPanel
    │   ├── ElevationSVG   (pure — takes config, returns SVG)
    │   └── OutputBar
    └── WizardFooter
```

---

## Pure Calculators (no UI)

These functions are completely independent of the React tree. They take a `config` and return data. They can be developed, tested, and reasoned about without touching the UI.

### `buildElevationSVG(config) → SVGElement`
Renders the front elevation. Scales to fit the container. Color-codes sections by `section.color`. Handles `interior.surfaceOffset` for custom sections. Shows dimension annotations (widths, overall width, height, toe kick). Highlights the active section when passed an optional `activeId`.

### `buildCutList(config) → CutListData`
Returns a structured object:
```js
{
  sections: [
    {
      sectionName: String,
      material: String,
      parts: [{ name, width, height, qty, notes }]
    }
  ],
  hardware: [{ item, qty, notes }],
  sheetSummary: [{ material, size, qty }]
}
```
Calculates all panel dimensions from the config (interior width = section width − 2×material thickness, etc.). Handles each `interior.type` independently.

### `buildPlansHTML(config) → String`
Assembles a complete, self-contained HTML document (same blueprint aesthetic as the garage plans page) using `buildElevationSVG` and `buildCutList` as inputs. Returns a string that can be opened in a new tab via a data URL or Blob URL.

### `encodeConfig(config) → String`
`JSON.stringify(config)` → `btoa()` → URL-safe base64 string.

### `decodeConfig(hash) → config | null`
`atob(hash)` → `JSON.parse()` → validates shape → returns config or null on failure.

---

## Wizard Steps — Detailed

### Step 1: Basics
Fields:
- Project name (text)
- Wall width in inches (number, required)
- Surface height (number, default 36)
- Depth (number, default 30)
- Toe kick height (number, default 4)
- Top thickness (select: 0.75" single layer / 1.5" double layer)
- Box material (select: 3/4" MDF / 3/4" plywood / 1/2" plywood)
- Top material (select: 3/4" plywood / 3/4" MDF / solid wood)

Validation: wall width must be > 0. All fields must be filled.

### Step 2: Sections
User builds the section list. Each section row shows: drag handle, color swatch, name, width, interior type summary, delete button.

**Add Section modal** fields: name, width (number), interior type (select), color (color picker with 6 presets).

**Width progress bar:** running total / wall width. Green = exact match. Amber = over or under. Step cannot advance until exact match.

Sections can be reordered by drag (using HTML5 drag-and-drop, no library needed).

### Step 3: Details
Steps through sections one at a time with Prev/Next section controls (separate from the main Back/Next wizard buttons). The preview panel highlights the active section with a dashed outline.

**Interior type forms:**

- **Drawers:** Shows current drawer list with height fields. "Auto-distribute" button divides available interior height equally. User can add/remove drawers. Live validation: drawer heights must sum to interior height.
- **Shelves:** Shelf count (number). Shelf mount method (pocket-screw / shelf-pins). Displays calculated bay heights.
- **Open:** Label field only. Nothing else to configure.
- **Custom:** Label, surface offset (number, can be negative), open below (checkbox), notes (text area).

### Step 4: Review
- Full-width elevation SVG at top
- Section summary table
- Cut list preview (first 5 rows of each section, "Show all" toggle)
- Sheet count summary
- Hardware list
- Three output actions: Generate Plans, Download CSV, Copy Share Link

---

## Outputs

### Plans Page
`buildPlansHTML(config)` returns a complete HTML string. On click, the app opens it with:
```js
const blob = new Blob([html], { type: 'text/html' });
window.open(URL.createObjectURL(blob));
```
The generated page matches the blueprint aesthetic of the existing garage plans site: dark navy background, IBM Plex Mono for dimensions, Oswald for headers, green/blue/orange section color coding.

### CSV Download
```js
const csv = buildCSV(cutListData); // joins rows with commas, adds headers
const blob = new Blob([csv], { type: 'text/csv' });
const a = document.createElement('a');
a.href = URL.createObjectURL(blob);
a.download = `${config.meta.name}-cut-list.csv`;
a.click();
```

### Shareable URL
On "Copy Share Link":
```js
window.location.hash = encodeConfig(config);
navigator.clipboard.writeText(window.location.href);
```
On page load:
```js
const hash = window.location.hash.slice(1);
if (hash) {
  const config = decodeConfig(hash);
  if (config) loadConfig(config); else showWizard();
}
```

---

## Presets

Defined as a static array in `src/presets.js`. Each preset is a complete `config` object.

| Key | Name | Wall Width | Sections |
|-----|------|-----------|----------|
| `garage-workbench` | Garage Workbench | 108" | 24" drawers · 20" shelves · 20" miter · 20" shelves · 24" drawers |
| `shop-storage` | Shop Storage Wall | 96" | 24" drawers · 24" shelves · 24" shelves · 24" drawers |
| `laundry-room` | Laundry Room | 72" | 24" drawers · 24" open · 24" drawers |
| `blank` | Start from Scratch | — | Empty sections, user sets wall width first |

---

## GitHub Pages Deployment

```
vite.config.js:  base: '/cabinet-plans-generator/'
package.json:    "deploy": "vite build && gh-pages -d dist"
```

Running `npm run deploy` builds and pushes the `dist/` folder to the `gh-pages` branch. GitHub Pages serves from that branch.

Live URL: `https://oneofthegeeks.github.io/cabinet-plans-generator/`

---

## File Structure

```
cabinet-plans-generator/
├── index.html
├── vite.config.js
├── package.json
├── CLAUDE.md
└── src/
    ├── main.jsx
    ├── App.jsx
    ├── presets.js
    ├── hooks/
    │   └── useCabinetConfig.js
    ├── calculators/
    │   ├── buildElevationSVG.js
    │   ├── buildCutList.js
    │   ├── buildPlansHTML.js
    │   └── encodeConfig.js
    ├── components/
    │   ├── LandingScreen/
    │   ├── WizardShell/
    │   ├── TopBar/
    │   ├── WizardPanel/
    │   │   ├── StepBasics/
    │   │   ├── StepSections/
    │   │   ├── StepDetails/
    │   │   └── StepReview/
    │   ├── PreviewPanel/
    │   │   ├── ElevationSVG/
    │   │   └── OutputBar/
    │   └── WizardFooter/
    └── styles/
        └── globals.css
```
