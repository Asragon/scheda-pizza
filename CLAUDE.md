# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

"Scheda Pizza" — a single-file, client-side pizza recipe card editor. The entire application (markup, styles, and logic) lives in one file: `index.html`. There is no build step, no package manager, no bundler, and no test suite. Development is done by editing `index.html` directly and opening it in a browser (as a `file://` URL or via GitHub Pages).

There is no local dev server config, linter, or test runner in this repo — "running" the app means opening `index.html` in a browser.

## Architecture

`index.html` is organized top-to-bottom as:

1. **`<style>`** — all CSS, using custom properties defined in `:root` (colors: `--rosso`, `--ocra`, `--verde`, `--crema`, etc.). Key structural pieces:
   - `.page` — the recipe sheet itself, sized as an A4 page (`210mm` desktop) that gets visually scaled via a dynamic `style.zoom` (see Zoom below), not a real responsive layout, on desktop.
   - `@media (max-width: 640px)` — a separate mobile layout that switches `.page` to fluid width and stacks the two-column grids into one column. Mobile does **not** use the desktop zoom-to-fit mechanism.
   - `body.exporting-image` — a class toggled temporarily by JS during PNG export (see below) that forces the same single-column stacked layout as the mobile breakpoint, regardless of actual viewport, so exported images are always a consistent "long" single-column strip. When editing the mobile breakpoint's structural rules (grid columns, `.page` sizing), keep the corresponding `body.exporting-image` rules in sync — they are intentionally duplicated rather than shared, since this file has no CSS preprocessor.
   - `@media print` — hides the toolbar and edit-only affordances (`.btn-add`, `.btn-remove`, `.modal-overlay`, `.add-ing-row`) for clean printing.

2. **`<body>`** — three top-level pieces:
   - `.toolbar` — sticky (desktop) / static (mobile) action bar: save/load JSON, print, export image, zoom controls.
   - `.page` — the actual recipe card: header (title/date), stagione (season) selector, then a `.main-grid` of cards (ingredienti, parametri, fasi di lavorazione, schema lievitazione, cottura, note). Most text fields are native `contenteditable` elements (`[data-k="..."]` attributes key them into the save/load state), not `<input>`s.
   - `.modal-overlay#modal-fase` — the "add fase" modal, populated from `FASE_PRESETS`.

3. **One `<script>` block** — no modules, everything is global functions/state. Key state variables: `sezioni` (impasto sections/ingredients), `parametri`, `fasi`, `stagioneSel`. Rendering is manual/imperative: mutate the state array, then call the matching `render*()` function (`renderSezioni()`, `renderParametri()`, `renderFasi()`, `renderStagioneUI()`) to rebuild that section's DOM from scratch — there is no reactive framework.

### Data model & persistence

- `statoCorrente()` serializes all state (contenteditable field values via `[data-k]`, `sezioni`, `parametri`, `fasi`, stagione, forno/pietra selects, date) into a plain object; `applicaStato(s)` does the reverse. This is the single source of truth for what "a recipe" contains.
- "Save" (`salvaRicetta()`) and "Load" (`caricaRicetta()`) work by downloading/reading a `.json` file via `Blob`/`FileReader` — there is no backend, no localStorage. Filenames follow the pattern `{slug(titolo)}_{slug(nome)}_{data}.json`, built with an inline `slug()` helper duplicated wherever a filename is generated (also used by image export).
- Preset data (`ING_HINTS`, `FASE_PRESETS`, `FORNO_PRESETS`) drives both placeholder hints and the defaults filled in when a type is selected — when adding a new ingredient/fase/forno type, add it to the relevant preset object rather than hardcoding markup.

### Date input

The date field (`#data-ricetta`) is a native `<input type="date">` kept functionally intact but visually hidden (`opacity: 0`) underneath a styled `<span id="data-ricetta-display">` that shows the value formatted as Italian `gg/mm/aaaa` (native date inputs otherwise render in the browser/OS locale format, which isn't controllable via CSS). Clicking calls `apriDatePicker()` → `input.showPicker()`. Any code path that sets the date value programmatically (`applicaStato`, the `change` listener) must also update `#data-ricetta-display`'s text via `formatDataIt()` — the two are not automatically in sync.

### Zoom (desktop "fit to window")

Because `.page` is a fixed-size A4 sheet, desktop uses a CSS `zoom` trick (`zoomFit()`, `zoomDelta()`, `applyZoom()`) to scale the whole sheet to fit the viewport, tracked in `zoomLevel`/`fitMode`. This is disabled below `MOBILE_BREAKPOINT` (640px), where the fluid mobile CSS layout takes over instead. Any feature that measures or captures `.page` (e.g. image export) must neutralize `style.zoom` first — html2canvas does not reliably support the non-standard CSS `zoom` property.

### Image export

`esportaImmagine()` uses `html2canvas` (loaded from a CDN `<script>` tag in `<head>` — the only external script dependency) to rasterize `.page` into a PNG. It temporarily adds `body.exporting-image` (forcing the single-column layout, see above) and clears `.page`'s `style.zoom`, waits two `requestAnimationFrame`s for reflow, captures at `scale: 2`, then tries `navigator.share()` (native share sheet, mainly for mobile) before falling back to a plain `<a download>` blob download. Both `exporting-image` class and `style.zoom` are restored in a `finally` block.

## Working in this repo

- Everything is in one file — there's no risk of forgetting to update a second location for markup, but CSS/JS changes should be made in the correct one of the three sections (styles / markup / script) rather than inlined ad hoc.
- No automated tests exist. Verify changes by opening `index.html` in a browser and exercising the UI directly (add/remove ingredients, save/load a JSON file, print preview, resize to mobile width, export image).
- The mobile breakpoint (`@media (max-width: 640px)`) and the `body.exporting-image` block intentionally duplicate the same structural CSS rules for two different trigger conditions — update both together when changing layout for stacked/mobile views.
