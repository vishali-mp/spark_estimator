# Repair Estimator Challenge

## Overview

Single-file vanilla JS SPA (`Example App.html`) — no build tools, no package manager, no tests. Fully client-side PWA, served by opening in a browser or any static file server.

## Project structure

```
Example App.html     — entire app (HTML, CSS, JS in one 1645-line file)
Pricing List.csv     — default line-item prices (source of truth for costs)
manifest.json        — PWA manifest (installable on Android/iOS)
sw.js                — service worker (caches app for offline use)
Contest Briefing.docx — challenge instructions
Spark Group - Logo.png
```

## How to run

Open `Example App.html` in any browser. No server, build, or install needed. PWA features (service worker, manifest) work when served from an HTTP server.

## Architecture

- **Framework**: None — vanilla JS, CDN Tailwind CSS (`cdn.tailwindcss.com`), CDN XLSX (`xlsx-js-style`), CDN JSZip
- **State**: Single `state` object with all data (items, photos, project metadata, price overrides, rooms, deal, UI state)
- **Rendering**: Full `innerHTML` re-render via `render()`, except qty inputs use `updateTotalsInPlace()` partial update
- **Events**: Delegation on `document.body` for `#app` content; direct listeners for elements outside `#app` (panel, modal, settings)
- **No routing, no components, no virtual DOM**

## Data / persistence

- **localStorage** key `sparkRepairProjects_v1` — project list with items, photos, rooms, custom items, price overrides, deal data
- **localStorage** key `sparkRepairPrices_v1` — global price override map (from CSV upload)
- Auto-save on 800ms debounce (`scheduleSave`), also on `visibilitychange`, `pagehide`, `freeze`
- Photos stored as base64 data URLs in localStorage (compressed to 1600px / 82% quality via canvas)

## Pricing

- Default prices defined in `Pricing List.csv` (columns: `id`, `name`, `cost`, `unit`)
- Hardcoded in `Example App.html` as the `CATEGORIES` array
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

## Creative addition: Deal Analyzer

In the summary view, a **Deal Analyzer** section lets agents input ARV (After Repair Value), purchase price, closing costs, and holding costs. It calculates total cost basis, estimated profit, and ROI percentage with color-coded feedback. Persisted per-project.

## Export

- **Export to Excel** (button in header) generates .xlsx with styled workbook
- If photos exist, bundles .xlsx + photos folder into .zip via JSZip
- File naming: `Repair-Estimate-{projectName}-{date}.xlsx` or `.zip`

## Modifying pricing

1. Edit `Pricing List.csv` for the single source of truth
2. Update the `CATEGORIES` array in `Example App.html` to match
3. CSV `id` values must match JS `CATEGORIES` item `id` values exactly
4. For new categories, add items to the array and GROUPS entries

## Adding a new multi-instance category

1. Add category to `CATEGORIES` with `type: 'multi'`
2. Add GROUPS entries with appropriate prefix (e.g. `br:`, `lc:`)
3. No other code changes needed — instances, subtabs, and state init are fully generic
