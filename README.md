# Spark Repair Estimator

A field-ready, single-file PWA for home repair estimating. Designed for property inspectors and real estate agents to quickly estimate repair costs during walkthroughs, capture photos of equipment serial numbers, run AI-based damage detection, and generate professional Excel reports — all offline.

## Quick Start

Open `App.html` in any browser. No server, build, or install needed.

```
open App.html
```

For PWA features (installable, offline caching), serve via HTTP:

```
python3 -m http.server 8080
```

## Architecture

| Aspect | Detail |
|--------|--------|
| **Framework** | None — vanilla JS |
| **Styling** | Tailwind CSS (CDN, runtime) |
| **Libraries** | `xlsx-js-style` (Excel export), `JSZip` (photo bundling), `@tensorflow/tfjs` + `@tensorflow-models/coco-ssd` (AI damage detection) |
| **Structure** | Single file: HTML + CSS + JS (~1950 lines) |
| **State** | Global `state` object, full `innerHTML` re-render via `render()` |
| **Events** | Delegation on `document.body` via `data-action` attributes |
| **Persistence** | `localStorage` — auto-save on 800ms debounce + page lifecycle events |

## Key Features

### Project Management

- **Side panel** — slide-out project list with project name, save date, active indicator, rename, and delete
- **Panel search** — filter projects by name or date via inline search bar inside the panel
- **New Project** — create a new blank estimate with a custom name
- **Switch projects** — tap any project in the panel to load it
- **Rename** — pencil button in header opens a rename modal
- **Delete** — trash button on inactive projects with confirmation
- **Auto-load** — last active project loads automatically on boot
- **Start New Estimate** — clears all entries with confirmation

### Category Navigation

- **Tab bar** — horizontal scrollable tabs for all 7 categories with cost/check badges
- **Previous / Next** — fixed bottom navigation buttons; "View Summary" replaces Next on the last category
- **Swipe** — touch swipe left/right (≥50px threshold) changes category
- **Scroll reset** — viewport scrolls to top on category switch

### Line Item Management

- **Check/uncheck** — circular toggle; quantity input auto-focuses when checked
- **Quantity controls** — `−` and `+` buttons, manual numeric input; totals update in-place (no full re-render)
- **Default quantity** — items with `defaultQty` pre-fill when checked
- **Minimum enforcement** — items with `min` property floor the total (e.g., `ig-14` min $500)
- **No-action items** — placeholder "No action needed" items at $0, inside group cards
- **Group cards** — related items grouped under labeled cards; collapse/expand with group total + check count badge; expansion state preserved
- **Inline totals** — each checked item shows computed cost (qty × unit cost, floored at min)
- **Metadata** — shows unit, notes, and minimum cost hints below item name

### Multi-Instance Categories (Bathroom, Bedroom, Living/Common)

- **Default instances** — Bathroom starts with 2, Living/Common with 1, Bedroom with 0
- **Sub-tabs** — horizontal scrollable instance tabs with totals and check counts
- **Add / Remove / Rename** — dynamic instance management from the sub-tab bar; remove requires confirmation, minimum 1 instance enforced
- **Composite state keys** — items identified as `b1:ba-01`, `bed1:br-03`, `lc1:lc-09`

### Search

- **Always-visible search bar** — inline input between the header and category tabs; no toggle needed
- **Live filtering** — results update in real-time as you type via partial DOM swap (no focus loss)
- **Global scope** — searches across all 7 categories, including custom items (excludes no-action placeholders)
- **Results view** — hides category tabs during search; results grouped by category in styled cards
- **Instant navigation** — tapping a result switches to the correct category tab (scrolled into view), collapses all other groups, expands only the matching item's group, clears the query, and scrolls the item card into view
- **✕ clear** — resets the query and restores the normal category view

### Pricing & Cost Overrides

Three-tier priority: **project override** → **global CSV override** → **default price**

- **Default** — hardcoded in `CATEGORIES` array (sourced from `Pricing List.csv`)
- **Global CSV** — Settings → upload CSV with `id` and `cost` columns; applied to all projects
- **Project-level** — tap the unit cost label on any item → inline prompt edit → per-project only
- **Price status** — Settings shows "Using default prices" or "Custom prices active — N items overridden"
- **Reset global prices** — button in Settings to clear all overrides
- **Custom item cost** — editable numeric input directly on custom items

### Custom Items

- **Add custom item** — button at bottom of each category opens sequential prompts (name, cost, unit)
- **Auto-ID** — IDs generated as `{catId}-cus-{N}` (e.g., `ig-cus-3`)
- **Delete** — "Remove" link on each custom item with confirmation
- **Multi-instance support** — custom items instantiated across all rooms of multi-instance categories
- **Manage UI** — placeholder in Settings for future global management

### Notes & Voice Dictation

- **Per-category/instance notes** — textarea at the bottom ofa each category view
- **Clear button** — `×` to clear note text
- **Speech-to-text** — microphone button triggers Web Speech API; transcribed text appended to the note
- **Dictation feedback** — orange highlight on mic + animated pulse dots while recording
- **Unsupported fallback** — alert shown if SpeechRecognition is unavailable
- **Notes in summary** — notes displayed below each section in the summary view

### Photo Capture

- **Camera / file picker** — opens device camera via `<input capture="environment">` for items with `hasYear: true` (furnace, condensing unit, HWH, roof)
- **Compression** — images compressed to 1600px max dimension at 82% JPEG quality via canvas
- **Barcode/QR detection** — `BarcodeDetector` API (Chromium) auto-extracts serial numbers from barcodes/QR codes on capture
- **Manual serial entry** — text field per photo for serial number input
- **Remove photo** — `×` button on each photo thumbnail
- **Summary gallery** — all captured photos grouped by item with serial numbers and AI labels
- **Export inclusion** — photos bundled into ZIP with serial-numbered filenames

### AI Damage Detection

- **COCO-SSD object detection** — TensorFlow.js + MobileNet v2 detects objects (people, vehicles, appliances) at >25% confidence
- **Canvas-based anomaly detection** — pixel-level analysis samples ~8k pixels for dark/green/yellow discolored areas; computes damage score 0–100
- **Damage thresholds** — >12% → "⚠ Anomalous areas", >35% → "⚠ Inspect closely", <12% → "✓ Surface looks normal"
- **Color-coded badges** — green / yellow / gray labels on each photo showing detected objects and damage score
- **Lazy loading** — model loads on first photo capture, not at boot
- **Graceful degradation** — silent error handling if TensorFlow/CDN is unavailable

### Deal Analyzer

In the summary view: inputs for ARV (After Repair Value), Purchase Price, Closing Costs, Holding Costs. Computes total cost basis, estimated profit, and ROI percentage with color-coded feedback and qualitative assessment (green >15%, yellow 0–15%, red negative). Persisted per-project. Partial in-place update preserves input focus during typing.

### Summary View

- **Total estimate** — large formatted figure at the top
- **Per-section breakdown** — each category/instance listed with items, quantities, unit costs, and section totals
- **Year badges** — `hasYear` items show the year in a badge
- **Notes** — per-category notes displayed below items
- **Photo gallery** — all captured photos with serial numbers and AI results
- **Empty state** — "No items selected" placeholder when nothing is checked
- **Summary as a tab** — accessible as the last tab in the category bar for direct one-tap access without tabbing through every category
- **Back to Edit** — returns to the main editing view

### Export

- **Excel (.xlsx)** — styled workbook with dark headers, orange section headers, item rows, category totals, and grand total
- **ZIP bundle** — if photos exist, creates `.zip` containing `.xlsx` + `photos/` folder with serial-numbered image files
- **PDF** — print-to-PDF via `window.print()` with branded header, strips buttons/inputs via `@media print` CSS
- **Share** — plain-text estimate breakdown via Web Share API; clipboard fallback on unsupported devices
- **File naming** — `Repair-Estimate-{projectName}-{date}.xlsx` or `.zip`

### Progress Tracking

- **Running total** — live cost total in the header
- **Project name + active sections** — `{projectName} · N active sections` label
- **Progress bar** — orange bar showing completion % based on checked groups / standalone items
- **Group-based units** — groups count as one unit if any item is checked; standalone items count individually

### PWA

- `manifest.json` — installable on Android/iOS
- `sw.js` — cache-first service worker (offline support)
- Apple touch icon + status bar styling + theme-color meta tag
- Service worker registered on load

### Auto-Save & Persistence

- **localStorage keys** — `sparkRepairProjects_v1` (projects) and `sparkRepairPrices_v1` (global price overrides)
- **Debounced auto-save** — 800ms debounce via `scheduleSave()` after any state change
- **Lifecycle hooks** — saves on `visibilitychange`, `pagehide`, and `freeze`
- **Storage quota handling** — if save fails (quota exceeded), retries without photos
- **Modified timestamp** — `savedAt` stored with each project

### Settings

- **Settings modal** — bottom sheet overlay with gear icon toggle
- **Price Schedule** — section explaining CSV-based global price overrides
- **CSV upload** — file input parses `id` and `cost` columns; updates global prices
- **Reset prices** — clears all global overrides (conditionally visible)
- **Custom Items** — placeholder for future global custom item management

### Partial DOM Updates

Two mechanisms avoid full re-renders on frequent interactions:

**`updateTotalsInPlace()`** — updates only:
- The affected item's total
- The parent group's total
- The header's running total
- The current category tab's badge
- The active section count label

**Search live filter** — typing in the search bar swaps only the `#scroll-area` innerHTML (results or category view) and toggles the tabs' visibility, preserving input focus during continuous typing.

## Data / Persistence

| Key | Content |
|-----|---------|
| `sparkRepairProjects_v1` | Project list with all data (items, photos, rooms, notes, custom items, price overrides, deal) |
| `sparkRepairPrices_v1` | Global CSV price override map |

Photos stored as base64 data URLs (compressed to 1600px / 82% JPEG quality). If localStorage quota is exceeded, photos are omitted from the save.

## Categories

| ID | Name | Type | Items | Default Instances |
|----|------|------|-------|-------------------|
| `interior-general` | Interior / General | single | 28 items + 4 no-action | — |
| `kitchen` | Kitchen | single | 17 items + 3 no-action | — |
| `bathroom` | Bathroom | **multi** | 16 items + 3 no-action | Bathroom 1, Bathroom 2 |
| `bedroom` | Bedroom | **multi** | 12 items + 3 no-action | None (user adds) |
| `living-common` | Living / Common | **multi** | 11 items + 3 no-action | Living Room |
| `systems` | Systems & Structure | single | 24 items + 4 no-action | — |
| `exterior` | Exterior | single | 23 items + 5 no-action | — |

**Multi-instance** categories support dynamic add/remove/rename of room instances. Item state uses composite keys (`b1:ba-01`, `bed1:br-03`, `lc1:lc-09`).

## Item ID Conventions

| Pattern | Example | Meaning |
|---------|---------|---------|
| `<cat>-<N>` | `ig-01`, `ba-07` | Built-in item |
| `<cat>-cus-<N>` | `ig-cus-1` | Custom item |
| `<cat>-nan-<N>` | `ig-nan-1` | No-action item (skip placeholder) |
| `<instId>:<itemId>` | `b1:ba-01` | Multi-instance composite state key |

## UI / UX Details

- **Safe area insets** — `safe-bottom` and `pt-safe` classes for notch/bar avoidance on modern devices
- **Touch-optimized** — `touch-action: pan-y` on scroll areas, `overscroll-behavior: none`, no number spinner arrows
- **Smooth panel animation** — side panel slides with `cubic-bezier(.4,0,.2,1)` timing
- **Modal system** — generic rename modal with auto-focus, Enter to confirm, Escape to dismiss, backdrop click to close
- **Swipe gestures** — touch navigation between categories
- **SVG icons** — all icons are inline SVGs (no icon library dependency)

## Project Structure

```
App.html             — Entire application (HTML + CSS + JS)
Pricing List.csv     — Default line-item prices (source of truth)
manifest.json        — PWA manifest
sw.js                — Service worker
logo.png             — Brand logo (favicon)
icon-180.png         — Apple touch icon
icon-192.png         — PWA install icon
icon-512.png         — PWA splash icon
Contest Briefing.docx — Original challenge instructions
Spark Group - Logo.png  — Full-size brand logo
AGENTS.md            — Developer reference
```

## Modifying Pricing

1. Edit `Pricing List.csv` (source of truth)
2. Update the `CATEGORIES` array in `App.html` to match
3. CSV `id` values must match JS item `id` values exactly
4. For new categories, add items to the array and `GROUPS` entries

## Adding a New Multi-Instance Category

1. Add a `CATEGORIES` entry with `type: 'multi'`
2. Add `GROUPS` entries with appropriate prefix (e.g., `br:`, `lc:`)
3. Item state initialization is fully generic — no other code changes needed

## Event Reference

All user interactions use the `data-action` attribute pattern:

| data-action | Purpose |
|-------------|---------|
| `search-input` | Live filter search results |
| `search-clear` | Clear search query and restore category view |
| `search-select` | Navigate to item: switch tab, expand group, scroll to card |
| `open-panel` | Open side panel |
| `open-settings` | Open settings modal |
| `export-excel` | Trigger Excel/ZIP export |
| `toggle-item` | Check/uncheck a line item |
| `switch-cat` | Switch category tab |
| `prev-cat` / `next-cat` | Navigate categories |
| `toggle-group` | Expand/collapse group card |
| `switch-instance` | Switch multi-instance subtab |
| `edit-instance-name` | Rename room instance |
| `add-instance` / `remove-instance` | Manage room instances |
| `rename-project` | Rename current project |
| `add-photo` / `remove-photo` | Manage photos |
| `show-summary` / `back-to-edit` | Toggle summary view |
| `reset` | Start new estimate |
| `edit-cost` | Edit line item unit cost |
| `inc-qty` / `dec-qty` | Adjust quantity |
| `share-project` | Share estimate |
| `clear-note` / `dictate-note` | Manage notes |
| `add-item` / `delete-item` | Manage custom items |
| `update-qty` / `update-year` / `update-note` / `update-serial` / `update-custom-cost` / `update-deal` | Input handlers |
