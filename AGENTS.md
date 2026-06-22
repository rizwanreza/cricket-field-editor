# AGENTS.md - Cricket Field Editor

Guidelines for AI agents working on this codebase.

## Project Overview

A client-side HTML/JavaScript cricket field editor. A captain can paste/upload a
JSON of fielders (by cricket position name) — a **single object, or an array of
them for multiple stacked diagrams** — and the app places everyone automatically,
then shares each as an image or prints them (one per page). Uses:
- **Konva.js** - Canvas graphics library
- **jQuery** - DOM manipulation (+ jquery-touch-events for mobile drag)
- **FileSaver.js** - Image/JSON download

No build system, bundler, or package manager. All dependencies loaded via CDN.
No Bootstrap, no html2canvas, no ads — styling is a custom stylesheet and image
export uses Konva's native `stage.toDataURL`.

## Files

- `index.html` - The entire app (single self-contained file: HTML + CSS + JS).
  **All work happens here.**
- `fielding_metric.html` - A tiny redirect to `index.html`, kept only so older
  links/bookmarks don't 404. Do not put app logic here.
- `README.md` - Project documentation

## Field setup JSON schema

This is one diagram. The input may be a single such object **or an array of them**
(→ multiple stacked, scrollable diagrams). **Save JSON** emits an object for one
diagram or an array for several, so saved files round-trip through the loader:

```json
{
  "title": "vs Australia — 2nd innings",   // optional, shown in header/print/image
  "bowler": "Ashwin",                       // bowler's name
  "handedness": "right",                    // "right"|"left" batter (aliases rh/lh/r/l)
  "keeper": "Pant",                         // optional
  "fielders": [                             // up to 9; "name" optional per entry
    { "position": "1st slip", "name": "Rohit" },
    { "position": "gully" }
  ]
}
```

There is no batter name — only the batter's handedness matters (it mirrors the
field). The bowler name and handedness are the important top-level fields. The
ready-to-paste **LLM prompt** (`LLM_PROMPT`, built from `SCHEMA_TEXT` plus the full
list of valid position names) is copied by the single "Copy LLM prompt" button;
the schema is no longer rendered on screen.

Position names resolve through `resolvePosition()`: normalized lookup against
`allpos`, then an `ALIASES` table (e.g. `slip`→`1st slip`, `cow corner`→
`deep mid-wicket`). Unknown names raise a `showAlert` (prefixed with the diagram
number) with a Levenshtein suggestion and are skipped (the rest still place).
Handedness is **per diagram**: each field derives its own `coords` from the shared,
never-mutated `BASE_COORD` (`mirrorCoords()` for left-handers). The older positional
save format (`fielders` as `[x,y,name]` triples) still loads via `applyLegacy()`.

## Commands

### Local Development

Since this is vanilla HTML/JS with no build step, serve files with any static server:

```bash
# Python 3
python -m http.server 8000

# Node.js (if npx available)
npx serve .

# PHP
php -S localhost:8000
```

Then open `http://localhost:8000` in browser.

### Testing

No automated tests exist. Manual testing required:
1. Open file in browser
2. Click **Load sample** → one diagram, all 9 fielders at named positions
3. Click **Sample (multiple)** → two stacked diagrams with different handedness,
   each mirrored correctly (verifies per-field `coords`, no shared mutation)
4. Drag a fielder in one diagram → only that diagram's labels/coverage update
5. Unknown/duplicate position and <9 fielders → alert names the diagram; rest place
6. Toggle display checkboxes → all diagrams update at once
7. **Save JSON** (array for several), then re-upload → set is restored
8. Per-card **Download/Copy**, **Download all** (one PNG each), **Print**
   (Cmd+P → one diagram per page with legend)

A quick headless smoke test (Chrome must be installed):

```bash
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --headless=new \
  --virtual-time-budget=6000 --screenshot=/tmp/cfe.png \
  "file://$PWD/index.html"
```

### Linting

No linter configured. Follow code style guidelines below.

## Code Style Guidelines

### JavaScript

**Variable Declaration:**
- Use `var` for consistency with existing code (legacy style)
- Declare variables at top of function scope
- Use camelCase for variables and functions

```javascript
// Good
var playerName = "John";
var playerGroup = getPlayerGroup(x, y);

// Avoid
const playerName = "John";  // Don't use const/let
let x = 10;                // Stick to var
```

**Functions:**
- Use traditional function declarations
- Name functions descriptively with camelCase
- Group related functions together

```javascript
function calculateDistance(pointA, pointB) {
  // implementation
}
```

**Indentation:**
- Use 2 spaces (not tabs)
- Be consistent with existing code

**Quotes:**
- Prefer single quotes for strings
- Use double quotes for HTML attributes

```javascript
var message = 'Player position updated';
var html = '<div class="alert">' + message + '</div>';
```

### HTML

**Structure:**
- Custom flex layout (`.layout` / `.field-col` / `.panel`); no CSS framework
- Mark non-diagram chrome with `.no-print`; print-only blocks use `.print-only`
- Include all CDN scripts in `<head>`
- Keep styling in the `<style>` block (CSS variables drive the palette)

**Script Placement:**
- Place JavaScript at end of `<body>`
- Use IIFE pattern if adding new script sections

### CSS

**Naming:**
- Use kebab-case for class names
- Prefix custom classes to avoid Bootstrap conflicts

```css
.field-coverage-area {
  /* styles */
}
```

## Architecture Patterns

**One factory per diagram (DRY):** `createField(parent)` builds one diagram —
its own DOM `.diagram-card`, `Konva.Stage`, the six layers, bat, and player groups —
and returns a field object `f` holding all that per-field state plus `coords`,
`isLeft`, `title`, `bowlerName`, `keeperName` and methods (`applySpec`, `serialize`,
`fit`, `destroy`). The controller keeps a `fields = []` array; `renderAll(specs)`
destroys old fields and creates one per spec.

- **Shared & immutable (module-level):** geometry constants, `allpos`,
  `BASE_COORD` (never mutated), position resolution (`normalizePos`,
  `resolvePosition`, `ALIASES`, `suggest`), node factories, coverage **math**, and
  the `LLM_PROMPT`. These are reused by every field.
- **Per-field functions take `f`:** `applySpec(f, spec)`, `drawGround(f, node)`,
  `drawCatch(f, node)`, `applyDisplayTo(f)`, `centerLabel(f, id)`, `fieldToBlob(f, cb)`.
  Never reach for a global `stage`/`layer` — use the passed field.

**Layers (per stage, drawn bottom→top):** `layerground` (backdrop + grass) /
`layerstripes` (mowing, clipped) / `layerfeatures` (30-yd circle, pitch, creases,
bat) / `layercover` (boundary, clipped) / `layercatch` (catch, clipped) / `layer`
(markers, labels, header pills).

**Player Groups:** each is a `Konva.Group` of `[circle, nameText, posLabel]`
(indices 0/1/2). The header pills are `Konva.Label`s (also Groups) — iterate
`f.playerGroups`, never `layer.find(Konva.Group)`, to avoid catching them.

**Controls:** display toggles are **global** (`refreshDisplay()` loops `fields`);
**handedness is per-diagram** from each spec. There is no manual name panel and no
global left-handed toggle.

**Event Handling:**
- jQuery for DOM events; Konva `stage.on('dragend', …)` per field (relabels the
  nearest position from `f.coords` and redraws that field's coverage + legend).

## Adding New Features

1. **Field Positions:** Add to `allpos` and `BASE_COORD` (and an `ALIASES` entry if
   it has common shorthand). Never mutate `BASE_COORD` at runtime.
2. **UI Controls:** Add a `.toggle`/`.field-input`/`.btn` in the `.panel`; global
   display toggles call `refreshDisplay()`.
3. **Visual Elements:** Add Konva shapes inside `createField` on the right layer;
   per-field draw helpers must take `f`.
4. **Persistence:** Update `serializeField(f)` (emits the schema) and
   `applySpec(f, spec)` / `applyLegacy(f, spec)` (the loaders).

## Error Handling

Use the existing `showAlert()` function for user-facing errors:

```javascript
try {
  // risky operation
} catch (error) {
  showAlert('Failed to load field configuration');
  console.error(error);
}
```

## Dependencies

All loaded via CDN in HTML `<head>`:
- `konva@2.4.2`
- `jquery@3.3.1` (+ `jquery-touch-events` for mobile drag)
- `FileSaver.js`

**Do not add npm packages or bundlers.** Keep it simple.

## Browser Compatibility

Target modern browsers. Test in:
- Chrome/Edge (primary)
- Firefox
- Safari

## Git Workflow

1. Make atomic commits with descriptive messages
2. Test `index.html` before committing (`fielding_metric.html` is just a redirect)
3. No pre-commit hooks or CI configured
4. Main branch is `main`
