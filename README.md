# Spark Repair Estimator

A field-ready, single-file PWA for home repair estimating. Designed for property inspectors and real estate agents to quickly estimate repair costs during walkthroughs, capture photos of equipment serial numbers, and generate professional Excel reports — all offline.

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
| **Structure** | Single file: HTML + CSS + JS (~1850 lines) |
| **State** | Global `state` object, full `innerHTML` re-render via `render()` |
| **Events** | Delegation on `document.body` for click + input |
| **Persistence** | `localStorage` — auto-save on 800ms debounce + page lifecycle events |

## Data / Persistence

| Key | Content |
|-----|---------|
| `sparkRepairProjects_v1` | Project list with all data (items, photos, rooms, notes, custom items, price overrides, deal) |
| `sparkRepairPrices_v1` | Global CSV price override map |

Photos stored as base64 data URLs (compressed to 1600px / 82% JPEG quality).

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

## Key Features

### Line Item Management
- Check/uncheck items; quantities auto-toggle visibility
- Groups collapse related items (27 groups across all categories)
- No-action items display as "skip" placeholders inside groups
- Items with `min` enforce minimum totals
- Items with `hasYear` (furnace, condensing unit, HWH) show year + photo inputs

### Pricing
Three-tier priority: **project override** → **global CSV override** → **default price**
- **Default**: Hardcoded in `CATEGORIES` array (sourced from `Pricing List.csv`)
- **Global CSV**: Upload via Settings → replaces prices for all projects
- **Project-level**: Tap cost label → inline edit → per-project only
- **Custom items**: Direct number input for cost

### Photo Capture
- Opens device camera via `<input capture="environment">`
- Compresses to 1600px / 82% quality
- Auto-detects **barcodes/QR codes** via `BarcodeDetector` API (Chromium)
- Manual serial number entry per photo
- **AI damage detection** — TensorFlow.js + COCO-SSD analyzes every photo for objects (people, vehicles, appliances) at >25% confidence; color-coded badges (green/yellow/gray). Canvas-based anomaly detection samples ~8k pixels for dark/green/yellow patches → damageScore 0–100 with flags
- Summary gallery grouped by item with AI results
- Export includes photos folder in ZIP

### Voice Dictation
- Per-category/instance notes textarea
- Speech-to-text via Web Speech API (mic button → transcript appended)
- Animated indicator during recording

### Deal Analyzer
In summary view: inputs for ARV, purchase price, closing costs, holding costs.
Calculates cost basis, profit, ROI with color-coded feedback (green/yellow/red).

### Export
- **Excel** (`.xlsx`) with styled workbook — section headers, item rows, totals
- **ZIP** bundle — Excel + `photos/` folder (photo filenames include detected serial numbers)
- **Share** — plain-text breakdown via Web Share API or clipboard

### PWA
- `manifest.json` — installable on Android/iOS
- `sw.js` — cache-first service worker (offline support)
- Apple touch icon + status bar styling

## Item ID Conventions

| Pattern | Example | Meaning |
|---------|---------|---------|
| `<cat>-<N>` | `ig-01`, `ba-07` | Built-in item |
| `<cat>-cus-<N>` | `ig-cus-1` | Custom item |
| `<cat>-nan-<N>` | `ig-nan-1` | No-action item (skip placeholder) |
| `<instId>:<itemId>` | `b1:ba-01` | Multi-instance composite state key |

## Project Structure

```
App.html             — Entire application
Pricing List.csv     — Default line-item prices
manifest.json        — PWA manifest
sw.js                — Service worker
logo.png             — Brand logo (favicon)
icon-180.png         — Apple touch icon
icon-192.png         — PWA install icon
icon-512.png         — PWA splash icon
Contest Briefing.docx — Original challenge instructions
Spark Group - Logo.png  — Full-size brand logo
```

## Modifying Pricing

1. Edit `Pricing List.csv` (source of truth)
2. Update the `CATEGORIES` array in `App.html` to match
3. CSV `id` values must match JS item `id` values exactly

## Adding a New Multi-Instance Category

1. Add a `CATEGORIES` entry with `type: 'multi'`
2. Add `GROUPS` entries with appropriate prefix
3. Item state initialization is fully generic — no other code changes needed
