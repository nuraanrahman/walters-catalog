# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-page fabric catalog for Walters ("Smart Digital Catalog"): browse 20 sample books across two categories (Sofa Upholstery, Curtains & Drapery), drill into a book to see its colorways, and inquire about a specific swatch via WhatsApp. There is no build step and no server — it's opened directly as a static HTML file.

## Running it

Open `Walters Catalog.dc.html` directly in a browser (e.g. `open "Walters Catalog.dc.html"`). There is no dev server, bundler, package.json, or test suite in this repo.

## Architecture

The app is built on **dc-runtime**, a small React-based template runtime vendored into `support.js` (a generated, do-not-edit bundle — the comment at its top points to `dc-runtime/src/*.ts`, which is not part of this repo). All application code lives in the single file `Walters Catalog.dc.html`, split into two parts:

- **`<x-dc>...</x-dc>`** — the HTML template. dc-runtime parses this and renders it with React under the hood.
- **`<script type="text/x-dc" data-dc-script>`** — a plain-JS `class Component extends DCLogic { ... }`. `DCLogic` is the documented base class (aliased at runtime to the framework's internal `StreamableLogic`).

Template/logic contract:
- `state` is the component's local state; `this.setState({...})` triggers a re-render.
- `renderVals()` runs on every render and returns the plain object whose keys are referenced in the template via `{{ key }}` (including inside `style="..."` attribute values, e.g. `background: {{ book.tint }};`) and as event handlers, e.g. `onclick="{{ book.onClick }}"`.
- `<sc-if value="{{ cond }}">` / `<sc-for list="{{ list }}" as="item">` are the only control-flow tags (plus `sc-else`, unused here). `hint-placeholder-val` / `hint-placeholder-count` are design-time-only hints for the empty/loading skeleton and don't affect runtime behavior.
- `style-hover="{ prop: 'value' }"` declares hover-state style overrides inline (there's no separate CSS class for hover); it's parsed by dc-runtime, not the browser.
- There is no client-side router — `state.view` (`'home'` | `'book'`) switches between the two top-level `<sc-if>` blocks in place.

Styling is almost entirely **inline `style="..."` attributes** (dc-runtime supports interpolation inside them), not stylesheet classes. The one real `<style>` block in `<helmet>` holds only: CSS custom-property color tokens on `:root`, `@keyframes`, `::selection`, a couple of `:focus-visible`/`::placeholder` rules that inline styles can't express, and the responsive `@media` overrides (which use `!important` to beat the inline styles at narrower breakpoints — this is intentional and required given the inline-first styling approach, not a hack to clean up). When adding new responsive behavior, follow that same pattern: give the element a `class`, then override it in the appropriate `@media` block.

`image-slot.js` defines a `<image-slot>` custom element (a drag-and-drop image placeholder component from an "omelette" starter scaffold) used for the book's application photo. It accepts a `src` fallback plus `shape`/`fit`/`placeholder` attributes; see the usage doc comment at the top of that file for the full attribute contract. Its normal drag-and-drop persistence only works inside the omelette design runtime — in a plain browser it just displays `src` read-only.

### Data flow

`catalog-data.js` sets `window.CATALOG_DATA`, an array of book objects:
```
{ id, name, material, category, tag, composition, rubLabel, rub, care, application, heroImage?, swatches: [{ name, code, hex, img? }, ...] }
```
`category` is either `"Sofa Upholstery"` or `"Curtains & Drapery"`. `heroImage` and per-swatch `img` are optional — when present they override the shared placeholder photo / flat hex chip (see "Real book data" below); books without them still render fine from `hex` alone plus `assets/fabric-application-placeholder.jpg`. `Component.renderVals()` filters/maps this array directly (no external state store) into the view-model consumed by the template — e.g. `buildCard()` builds home-grid cards, and the book-detail path derives `book`/`selected` from `state.bookId`/`state.swatchIndex`.

### Real book data (replacing placeholders)

Books are being converted one at a time from placeholder data to real data, sourced from each book's physical swatch-card PDF (a photographed page per colorway: fabric photo + a printed tag with Shade No., SR. NO., Composition, Martindale rub count, wash-care icons, End Use icons). When converting a book:

**Always use the PDF's SR. NO. (not the Shade No.) when extracting swatches.** Every tag prints two different numbers — Shade No. is an internal, non-sequential dye-lot reference shared across Walters' whole catalog, while SR. NO. is this specific book's own 01, 02, 03... sequence. The SR. NO. is what's used everywhere in the app's data:

- **Naming**: the printed tag has no descriptive color name — use the SR. NO. for both fields: `name: "SR {SRNo, zero-padded}"` (e.g. `"SR 01"`) and `code: "{BookPrefix}-{SRNo, zero-padded}"` (e.g. `"FL-01"`). Don't use the Shade No. anywhere in the display data, and don't invent a descriptive name like "Royal Blue".
- **Composition/rub/care**: transcribe from the printed tag, don't keep generic boilerplate. Wash-care is icon-only (no text) — read the icons carefully (tub=wash temp, triangle=bleach, iron, square=tumble dry, circle=dry-clean solvent code) and state what was inferred so it can be sanity-checked; don't guess if unclear.
- **Application**: if the tag's End Use icons show more than one (e.g. both a sofa and a curtain icon), reflect that in `application` rather than assuming it matches the book's single `category` tab.
- **Swatch photos**: extract the embedded page image per PDF page (e.g. via PyMuPDF), crop out just the fabric area (exclude the printed label strip), and save to `assets/<sofa|curtains>/<book-id>/swatches/<CODE>.jpg`. Photographed catalog pages have inconsistent shadow bands (page-curl/binding shadow) and glare depending on how that page sat when photographed — don't use one fixed percentage crop for every swatch in a book. Instead scan for the flattest/most evenly-lit horizontal band within the safe zone (below any top shadow, above the label strip) and crop that, then resize every swatch in the book to identical output pixel dimensions so the colorway grid reads as one consistent set, not a mix of zoom levels/aspect ratios.
- **Hero photo**: pull one clean application photo out of the PDF's cover collage (multiple product shots composited into one page image — crop out just one), save to `assets/<sofa|curtains>/<book-id>/hero.jpg`, set `heroImage`. Once a hero photo has been approved, treat it as locked — don't re-crop or replace it on later passes (e.g. when only fixing swatch-photo cropping) unless explicitly asked to.
- Per-swatch `hex` should be sampled from the final cropped swatch photo (not eyeballed), so the color chips/tabs match the photo actually shown.
- **New/generated hero photos must be 4:5 portrait**, product centered both axes, filling ~55-70% of frame height, with generous even margin on all four sides (the mobile hero system below depends on this exact ratio). Reusable generation brief:
  ```
  Aspect ratio: 4:5 portrait, high resolution (2000px+ short side).
  Subject (sofa/armchair/curtain) centered horizontally and vertically,
  filling ~55-70% of frame height, even ~15-20% margin on all four sides
  — nothing important near the edges, since that margin is what gets
  cropped on other screen shapes. Avoid wide panoramic room shots; favor
  a tighter, more direct view of the product. Soft natural daylight,
  neutral/warm interior, photorealistic editorial furniture-catalog style.
  ```

### Extraction pipeline (technical steps)

The PDF-to-assets conversion described above follows a repeatable script-driven pipeline. This system has no Python deps installed globally — set up a scratch venv once per session:
```
python3 -m venv <scratchpad>/venv && source <scratchpad>/venv/bin/activate
pip install PyMuPDF Pillow numpy
```
Steps, in order:
1. **Extract page images**: `fitz.open(pdf)`, then for each page `page.get_images(full=True)[0][0]` → `fitz.Pixmap(doc, xref)` → save as PNG. Page 0 is the cover collage; pages 1..N are one swatch each; the last page is usually the spec/back cover (skip it). Page index `i` maps to `SR. NO.` sequentially (page01→SR01, page02→SR02, ...) — confirm this against a few visible SR. NO. labels before trusting it for the whole book, since a book can have a divider or extra page that shifts the mapping.
2. **Read tag fields visually** (Read tool on each page PNG, not OCR) — Shade, SR. NO., Composition, Martindale, Wash Care icons, End Use icons. Zoom-crop the label strip with Pillow if icons are too small to read at full-page scale. **Verify End Use icons against the book's assumed `category`/`application` before writing any data** — a "SOFA FABRIC" filename or an existing (placeholder) category is not proof; the icons are ground truth and have caught at least one mis-tagged category (Florida was filed as Curtains & Drapery but the tag only shows sofa+cushion icons).
3. **Crop each swatch photo — avoid the aspect-ratio stretch trap**: pick a "safe band" (a vertical slice below any top binding shadow, above the printed label strip; scan row-mean brightness to find where the white label starts if unsure). That safe band's width:height ratio will almost never match your target output ratio (e.g. 4:3) — **do not `resize()` straight to fixed output dimensions**, that non-uniformly stretches the fabric pattern (this happened to Florida's ikat swatches: patterns looked visibly squished/distorted until re-cropped). Instead, first crop *within* the safe band to the exact target aspect ratio (center a width- or height-constrained window, whichever the band is oversized in), *then* resize — resize should only ever scale uniformly, never reshape. Render a contact sheet (grid of all crops via Pillow, one `Read` call) to sanity-check the whole book at once before writing files into `assets/`, rather than reviewing swatches one image at a time.
4. **Preview in the real app before declaring done**: `file://` URLs are blocked by the claude-in-chrome extension, so serve the repo (`python3 -m http.server <port>` from the project root) and navigate to `http://localhost:<port>/Walters%20Catalog.dc.html?book=<id>` — the `?book=` deep link jumps straight to the converted book. Screenshot the colorway grid and at least one swatch detail panel; close the tab and kill the http.server process when done.

### Mobile hero image system

The book-detail hero (`.hero-bg-fixed`) is `position: fixed` full-bleed on desktop (intentional — it's meant to feel pinned behind the scrolling content). Naively keeping it `inset: 0` (i.e. sized to the full viewport) on a phone forces `fit="cover"` to crop hard into a portrait 4:5 photo, since a phone viewport is far taller/narrower than 4:5 — this cropped sofas/products almost entirely out of frame and took several wrong turns to fix (making the reveal window taller just showed more of the *same* crop — it didn't reduce the crop, because the fixed layer's crop is determined purely by viewport aspect vs image aspect, not by how much of it `.hero-spacer` reveals).

The fix, in the `@media (max-width: 768px)` block: keep `.hero-bg-fixed` `position: fixed` (preserve the pinned/parallax feel) but constrain its **height to `aspect-ratio: 4 / 5`** instead of stretching to the full viewport — matching the hero photo brief above means `cover` has nothing left to crop, so the whole product shows. `.hero-spacer`'s `min-height` is set to `125vw` (i.e. `100vw × 5/4`) to match that same rendered height exactly, so the title/tag sit right at the bottom edge of the photo with no dead gap before the colorways card. Keep this pairing (`hero-bg-fixed` aspect-ratio + `hero-spacer` min-height in matching units) in sync if either changes — they're solving the same crop together, not two independent tweaks.

### Page structure

- **Home** (`state.view === 'home'`): header + search + category tabs, then two `sc-for` grids (sofa/curtain books) filtered by `state.query`/`state.category`.
- **Book detail** (`state.view === 'book'`): a `position: fixed` full-bleed hero image (the book's application photo, via `<image-slot>`) with a dark scrim, a fixed "Back" pill and WhatsApp FAB layered above it, and a single scrolling content column (`.book-scroll`, `z-index: 1`) that sits over the fixed hero — title/meta over the photo, then one card (`.book-card`) containing the colorways grid, the selected-swatch chip + details, and the WhatsApp CTA.

### Design system

The palette is **"Mill Ledger"**: warm ledger-paper cream (`--bg`/`--surface`/`--surface-raised`), ink text, and a single madder-red `--accent` — chosen deliberately over a generic dark-charcoal-+-gold "AI luxury" look. Fonts are system stacks via `--font-display` (Georgia/serif), `--font-body` (Avenir Next/sans), `--font-mono` (SF Mono, used for swatch codes) — no webfont imports. The visual identity leans on *qualities* borrowed from the products rather than literal iconography (a literal grommet-ring / tufted-button icon pass was tried and explicitly rejected): a soft uneven SVG "drape" seam where the book-detail content card overlaps the hero photo, layered soft shadows with real hover-lift motion on cards/buttons ("comfort"), and a thin hand-drawn stitched divider line instead of hard rules ("flow"). Keep new UI in this vocabulary rather than reaching for dark+gold or literal fabric-hardware imagery.

The catalog also supports deep-linking: `?book=<id>` in the URL opens straight to that book (read in the `state` initializer and on `popstate`), and `goHome`/`openBook` push/pop that query param via `history.pushState` — this is what lets a QR code point straight at one book.

### Assets

`assets/walters-logo.png` is the current logo — transparent background, placed directly on the page (no white badge chip; that was removed because a white card looked out of place against the cream background once the logo itself already has no background to blend in). `assets/fabric-application-placeholder.jpg` is a temporary stand-in application/hero photo used by any book that doesn't yet have a real `heroImage`. Real per-book photography goes under `assets/sofa/<book-id>/` or `assets/curtains/<book-id>/` (see "Real book data" above).
