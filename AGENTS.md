# Repair Estimator Challenge

## Overview

Single-file vanilla JS SPA (`App.html`) — no build tools, no package manager, no tests. Fully client-side PWA, served by opening in a browser or any static file server.

## Project structure

```
App.html             — entire app (HTML, CSS, JS in ~2087 lines)
Pricing List.csv     — default line-item prices (source of truth for costs)
manifest.json        — PWA manifest (installable on Android/iOS)
sw.js                — service worker (caches app for offline use)
logo.png             — brand logo
icon-180.png         — Apple touch icon
icon-192.png         — PWA install icon
icon-512.png         — PWA splash icon
Contest Briefing.docx — challenge instructions
Spark Group - Logo.png
README.md            — full feature documentation
writeup.md           — one-page challenge writeup
AGENTS.md            — developer reference
```

## How to run

Open `App.html` in any browser. No server, build, or install needed. PWA features (service worker, manifest) work when served from an HTTP server.

## Architecture

- **Framework**: None — vanilla JS, CDN Tailwind CSS (`cdn.tailwindcss.com`), CDN XLSX (`xlsx-js-style`), CDN JSZip, CDN TensorFlow.js + COCO-SSD
- **State**: Single `state` object with all data (items, photos, project metadata, price overrides, rooms, deal, UI state)
- **Rendering**: Full `innerHTML` re-render via `render()`, except qty inputs use `updateTotalsInPlace()` and search uses targeted `#scroll-area` swap
- **Events**: Delegation on `document.body` for `#app` content; direct listeners for elements outside `#app` (panel, modal, settings)
- **Search**: Always-visible inline bar, live filtering via partial DOM update (preserves input focus), instant navigation on result click
- **No routing, no components, no virtual DOM**

## Data / persistence

- **localStorage** key `sparkRepairProjects_v1` — project list with items, photos, rooms, custom items, price overrides, deal data
- **localStorage** key `sparkRepairPrices_v1` — global price override map (from CSV upload)
- Auto-save on 800ms debounce (`scheduleSave`), also on `visibilitychange`, `pagehide`, `freeze`
- Photos stored as base64 data URLs in localStorage (compressed to 1600px / 82% quality via canvas)

## Pricing

- Default prices defined in `Pricing List.csv` (columns: `id`, `name`, `cost`, `unit`)
- Hardcoded in `App.html` as the `CATEGORIES` array
- **Global overrides**: Settings → Upload CSV (only `id` and `cost` columns matter), stored in `state.priceOverrides` → localStorage key `sparkRepairPrices_v1`
- **Per-project overrides**: tap the cost label on any line item to edit inline, stored in `state.projectPriceOverrides` → per-project in `sparkRepairProjects_v1`
- Priority: `projectPriceOverrides` → `priceOverrides` → `item.cost`

## Key conventions

- **Line item IDs** follow pattern: `<category>-<number>` (e.g. `ig-01`, `br-07`, `lc-10`, `ba-12`, `as-14`, `ex-14`)
- **Custom items** use `-cus-` suffix (e.g. `ig-cus-1`), stored in `state.customItems[catId]`
- **No-action items** use `-nan-` suffix (e.g. `ig-nan-1`), flagged with `noAction: true`
- **Multi-instance items** use composite state IDs: `{instanceId}:{itemId}` (e.g. `b1:ba-01`, `bed1:br-03`, `lc1:lc-09`)
- **Items with `hasYear: true`** (furnace, condensing unit, HWH) require year + photo input
- **`min` property** on some items enforces a minimum total (e.g. `ig-14` has `min: 500`)
- **`defaultQty`** pre-fills quantity when item is checked
- **Groups** (`GROUPS` object) collapse related items under one expandable card
- **Multi-instance categories** (bathroom, bedroom, living-common) dynamically create per-instance item state

## Categories

| Category ID  | Name                   | Type        | Items      | Default instances            |
|-------------|------------------------|-------------|-----------|------------------------------|
| interior-general | Interior / General | single      | 28 items  | —                            |
| kitchen     | Kitchen                | single      | 17 items  | —                            |
| bathroom    | Bathroom               | **multi**   | 16 items  | Bathroom 1, Bathroom 2       |
| bedroom     | Bedroom                | **multi**   | 12 items  | None (added by user)         |
| living-common | Living / Common      | **multi**   | 11 items  | Living Room                  |
| systems     | Systems & Structure    | single      | 24 items  | —                            |
| exterior    | Exterior               | single      | 23 items  | —                            |

All multi-instance categories support add/remove/rename of instances. Room instances are stored in `state.rooms[catId].instances`.

## Search

- **Always-visible** — inline search bar between header and category tabs, no toggle needed
- **Live filtering** — results update in real-time via targeted `#scroll-area` innerHTML swap (no focus loss)
- **Tabs hidden during search** — `#category-tabs` display set to `none` when query is non-empty
- **Instant navigation** — clicking a result switches category tab (scrolled into view), collapses all other groups, expands only matching group, scrolls item card into view
- **Scope** — searches across all 7 categories including custom items (excludes no-action placeholders)
- **Project panel search** — separate search bar in side panel filters projects by name or date

## Summary as a Tab

Summary is rendered as the last tab in the category bar, giving users a direct one-tap path to review/export without tabbing through every category. Clicking any other tab exits summary and returns to the edit view.

## Creative Additions

- **AI Damage Detection** — TensorFlow.js COCO-SSD detects objects + canvas-based pixel analysis computes damage score (0–100) on captured photos. No server upload.
- **Deal Analyzer** — ARV, purchase price, closing costs, holding costs → profit/ROI with color-coded feedback in summary view
- **Global Search** — always-visible inline bar with live filtering across all categories, instant navigation to matching items
- **Voice Dictation** — Web Speech API transcribed notes with pulse-dot recording indicator
- **Barcode/QR Auto-Detection** — automatic serial extraction from photos via `BarcodeDetector` API
- **PDF Export** — `window.print()` with `@media print` CSS stripping interactive elements, producing client-ready PDFs with zero dependencies

## Export

- **Export to Excel** — generates .xlsx with styled workbook; if photos exist, bundles .xlsx + photos folder into .zip via JSZip
- **Download as PDF** — print-to-PDF with branded header, hides buttons/inputs via @media print CSS
- **Share** — plain-text estimate via Web Share API (clipboard fallback)
- File naming: `Repair-Estimate-{projectName}-{date}.xlsx` or `.zip`

## Partial DOM Updates

To avoid full re-renders on frequent interactions:

**`updateTotalsInPlace()`** — updates only the affected item's total, parent group total, header running total, category tab badge, and active section count.

**Search live filter** — swaps `#scroll-area` innerHTML and toggles `#category-tabs` visibility, preserving input focus during continuous typing.

## Modifying pricing

1. Edit `Pricing List.csv` for the single source of truth
2. Update the `CATEGORIES` array in `App.html` to match
3. CSV `id` values must match JS `CATEGORIES` item `id` values exactly
4. For new categories, add items to the array and GROUPS entries

## Adding a new multi-instance category

1. Add category to `CATEGORIES` with `type: 'multi'`
2. Add GROUPS entries with appropriate prefix (e.g. `br:`, `lc:`)
3. No other code changes needed — instances, subtabs, and state init are fully generic
