# Repair Estimator — One-Page Writeup

## 1. Most Interesting Design/UX Decision

**Inline search bar with instant navigation.** Rather than a modal overlay or separate search page, the search bar lives permanently below the header — always visible, zero taps to reach. As you type, results filter in real-time via targeted DOM swaps (not a full re-render), preserving input focus. Tapping a result switches categories, collapses all groups except the matching one, scrolls the active tab into view, then scrolls the item card into view. This turns a 7-tab, ~130-item tree into a single-keystroke lookup — critical for field agents who need to find "water heater" or "drywall" fast while walking a property.

## 2. What's Broken or Fragile

- **Single-file monolith** — all HTML/CSS/JS in one 2000+ line file. No module boundaries, no type checking, hard to test.
- **`innerHTML` full re-render** — most state changes wipe and rebuild the DOM. Focus, scroll position, and transient UI state are lost unless explicitly preserved (mitigated by partial DOM updates for search and totals, but fragile).
- **localStorage for photos** — base64 data URLs consume ~1.37× the binary size; a property with 50+ photos hits the ~5–10 MB quota quickly. No compression beyond canvas resize.
- **CDN dependencies** — Tailwind, TensorFlow.js, XLSX, JSZip all load from CDN. First load (or any offline launch) fails silently if uncached.
- **Chromium-only barcode detection** — `BarcodeDetector` API is unsupported on Safari/Firefox.
- **TensorFlow.js is heavy** — ~2 MB model download on first photo capture; no loading indicator, can freeze the UI.

## 3. Creative Additions

**AI Damage Detection** — When a user captures a photo of damaged property, TensorFlow.js (COCO-SSD MobileNet v2) runs client-side object detection to identify items (appliances, vehicles, structural elements) at >25% confidence. Simultaneously, a canvas-based pixel analysis samples ~8k pixels for dark, green, or yellow discoloration and computes a 0–100 damage score. Results appear as color-coded labels on each photo: green ✓ for clean surfaces, yellow ⚠ for anomalous areas (>12%), red ⚠ for severe damage (>35%). No server upload needed — all inference runs in the browser. To use: tap the camera button on any equipment item (furnace, HWH, roof, condensing unit), take a photo, and the damage analysis appears automatically on the thumbnail in the summary gallery.

**Deal Analyzer** — Built into the summary view, inputs for ARV, purchase price, closing costs, and holding costs compute total cost basis, estimated profit, and ROI with color-coded feedback (green ≥15%, yellow 0–15%, red negative). Target users need to know *whether the deal works*, not just the repair cost. Combining both in one view eliminates context-switching.

**Voice Dictation** — A microphone button on each category note uses the Web Speech API to transcribe speech-to-text. The transcribed text is appended to the existing note with visual pulse-dot feedback while recording. Designed for hands-free field use — agents can dictate observations while walking a property.

**Barcode/QR Auto-Detection** — When a photo is captured (furnace, HWH, condensing unit, roof), the `BarcodeDetector` API (Chromium) automatically scans the image for barcodes and QR codes. Detected serial numbers are extracted and stored alongside the photo — no separate scan button, no extra step.

**Summary as a Tab** — Summary is rendered as the last tab in the category bar alongside Interior, Kitchen, Bathroom, etc., giving users a persistent one-tap path to the review/export view without needing to tab through every category.

**Project Panel Search** — A search bar in the side project panel filters projects by name or date in real-time, essential for agents managing dozens of estimates across properties.

**Download as PDF** — A PDF button in the summary header triggers `window.print()` with a `@media print` stylesheet that strips interactive elements (buttons, inputs) and preserves the branded header and formatting, letting users save a client-ready PDF from the system print dialog. Zero additional dependencies.

## 4. What I'd Ship Next (Two More Days)

- **IndexedDB photo storage** — migrate base64 photos to IndexedDB blobs to bypass localStorage quota limits and enable chunked upload.
- **Estimate templates** — save a configured estimate (checked items, quantities, custom items) as a template; apply to new projects for recurring property types (e.g., "Standard Flip Checklist").
- **Offline-first with sync** — lightweight backend (or Cloudflare Durable Objects) for multi-device access; syncs when connectivity returns.
- **Cost breakdown charts** — canvas-based pie/bar charts in the summary view showing cost distribution by category.
- **Print-optimized summary** — a print-specific CSS layout that produces a client-ready PDF without debug UI.

## AI Tools Role

AI assistants (Claude) were used throughout: generating initial component structure, refactoring the header layout, implementing the search system with partial DOM updates, and debugging CSS/event-handling issues. The AI served as a pair programmer — suggesting approaches, writing the first pass of most features, and catching edge cases (e.g., multi-instance item IDs in search results, focus preservation during live filtering). All AI-generated code was reviewed, tested, and iterated on. The architectural decisions (state management, rendering strategy, persistence model) were human-driven.
