# Maintenance Guide

Technical reference for anyone (human or AI) maintaining this codebase. Read this before making changes.

## 1. Project shape

The entire application is a **single self-contained file**, served for GitHub Pages as `docs/index.html`. There is no build step, no bundler, no package manager, and no runtime dependencies. HTML, CSS, and JavaScript all live in that one file. The favicon is embedded as a base64 data URI in the `<head>`, so the file is fully portable — open it from disk and it works.

```
qdvc-html-to-markdown-SPA/
├── README.md                  ← user-facing cover page
├── docs/
│   └── index.html             ← the whole app (GitHub Pages serves from /docs)
├── maintenance/
│   └── MAINTENANCE.md         ← this file
└── vibe-coding/               ← session logs
```

Note: the app file was previously `html-to-markdown.html` at the repo root; it moved to `docs/index.html` so GitHub Pages can serve it. If you see older references to `html-to-markdown.html`, they mean this file.

Design constraint to preserve: **keep it a single file with zero dependencies.** The value proposition is "download one file, open it, done." Do not introduce npm packages, external script/CDN tags, or a build pipeline without a deliberate decision to change that contract.

## 2. Anatomy of the file

Reading top to bottom:

1. **`<head>`** — meta tags, `<title>`, and the embedded SVG favicon (a crimson down-arrow on a beige circle, matching the app palette).
2. **`<style>`** — all CSS. The palette is defined as CSS custom properties in `:root` (see §5). Layout is a two-pane CSS grid that collapses to stacked panes under 760px.
3. **`<body>` markup** — header (wordmark + tagline + a **Settings** gear button), `<main>` with two `.pane` sections (input textarea; output area with **Links**, **Preview**, **Copy** buttons), the **settings modal** (`#settings-overlay`, hidden by default), and a footer with live character counts and the GitHub / open-source attribution.
4. **`<script>`** — one IIFE (`(function () { "use strict"; ... })();`) containing all logic. At the top of the IIFE is the `SETTINGS` object and the `emphasize` helper (see §3.5). Below that, the converter, the URL scanner (`extractUrls` / `buildLinksReport`, see §4.5), the preview renderer, and three parts separated by comment banners:
   - the **HTML → Markdown converter** (top),
   - the **Markdown → HTML renderer** used only for the Preview pane (`// --- minimal markdown -> html renderer ---`),
   - the **DOM wiring** (`// --- wiring ---`), which also handles opening/closing the modal, tab switching, reacting to settings changes, and the three-way output view state (see §4.5).

## 3. HTML → Markdown converter (the core)

This is the important part. It parses pasted HTML with the browser's own `DOMParser` (`new DOMParser().parseFromString(html, "text/html")`) and walks the resulting DOM tree, emitting Markdown. Never write a regex-based HTML parser here; the DOM parser is the whole point and it handles malformed input gracefully.

### Two rendering modes

The converter distinguishes **inline** context from **block** context, because Markdown treats them differently.

- **`inline(node)` / `renderInline(n)`** — produce inline Markdown with no block spacing. Handles `BR`, `STRONG`/`B`, `EM`/`I`, `DEL`/`S`/`STRIKE`, `CODE`, `A`, `IMG`, and a set of pass-through wrappers (`SPAN`, `FONT`, `SUB`, `SUP`, `U`, `MARK`, `SMALL`, `ABBR`, `TIME`, `CITE`, `Q`) whose children are rendered but whose tags are dropped. Unknown inline tags fall through to rendering their children. Two cases are **settings-dependent**: emphasis tags route through `emphasize()` and consult the nested-emphasis depth counters, and `IMG` is dropped entirely when `SETTINGS.discardImages` is true (see §3.5).
- **`block(node, indent)` / `renderBlock(n, indent)`** — produce block-level Markdown, joining children with blank lines. Handles headings, `P`, `HR`, `BLOCKQUOTE`, `PRE`, `UL`/`OL`, `DL`, `TABLE`, `FIGURE`, and strips `SCRIPT`/`STYLE`/`NOSCRIPT`/`TEMPLATE`. Unknown tags are classified via `hasBlockChild` / the `BLOCK` set and recursed into as either a block container or an inline run.

`convert(html)` is the entry point: parse → `block(doc.body, "")` → collapse 3+ newlines to 2 → trim.

### The `BLOCK` set and `hasBlockChild`

`BLOCK` is a `Set` of tag names treated as block-level containers. When `renderBlock` hits an unrecognized element, it uses `BLOCK.has(tag) || hasBlockChild(n)` to decide whether to descend as a block (blank-line-separated children) or flatten to an inline run. If you add support for a new block element, add its tag to `BLOCK` and/or give it an explicit `case` in `renderBlock`.

### Escaping — read this before touching it

This is the subtlest and most bug-prone area, and the behavior is **intentional**. There are two escaping functions:

- **`esc(t)`** escapes only characters that are special *mid-text*: `` \ ` * _ [ ] < > `` and stray entity ampersands. It deliberately does **not** escape `.`, `-`, `(`, `)`, `#`, `!`, `|`, `~`, etc., because those are only meaningful at specific positions and escaping them everywhere produces ugly, hard-to-read Markdown (e.g. `Berners\-Lee, T\. \(1989\)`).
- **`escLineStart(line)`** escapes a character only when it appears as the **first token of a line**, where it would otherwise be parsed as a block construct: leading bullet markers (`- + *`), ordered-list markers (`1.` / `1)`), ATX headings (`#`), blockquotes (`>`), and setext/HR rules (`===`, `---`, `___`). It is applied to plain text nodes and to paragraph output in `renderBlock`.

**Rule of thumb:** if a converted document shows backslashes inside ordinary prose (citations, DOIs, sentences), the fix is almost always in `esc`, not `escLineStart`. If raw Markdown syntax leaks through and gets misrendered (a sentence starting with "1. " turning into a list), the fix is in `escLineStart`. Do not revert to blanket escaping.

### 3.5. Settings, emphasis, and whitespace at emphasis boundaries

Conversion behavior is driven by a `SETTINGS` object at the top of the IIFE:

| Key | Values | Default | Effect |
|---|---|---|---|
| `discardImages` | `true` / `false` | `true` | When true, `<img>` emits nothing. When false, it emits `![alt](src "title")`. |
| `nestedEmphasis` | `"discard"` / `"stack"` / `"reverse"` | `"discard"` | How to resolve emphasis of the same kind nested inside itself. |

The settings modal (§6.5) mutates `SETTINGS` and calls `run()`, so changes apply live. `SETTINGS` is in-memory only — it resets on reload; there is intentionally no persistence (no `localStorage`).

**Emphasis is emitted through `emphasize(inner, kind, marker, combined)`, not by string concatenation in the tag cases.** This exists to fix two problems and must not be bypassed:

1. **Whitespace at boundaries.** Markdown delimiters must hug non-space text — `* text *` does *not* italicize, and two adjacent emphasized runs written as `*a**b*` collide. `emphasize` splits leading/trailing whitespace off the inner content and emits it *outside* the markers, so `<i>a </i><i>b</i>` becomes `*a* *b*` (space preserved, both runs valid) rather than `*a**b*`. This was the original bug behind mangled output like `with*[*callback`. The old code called `.trim()` inside each tag case, which destroyed these boundary spaces — do not reintroduce that.

2. **Nested same-kind emphasis is ambiguous** and resolved per `SETTINGS.nestedEmphasis`. Nesting is tracked by the module-level `emphasisDepth` counters (`star`, `strong`, `strike`), incremented around the recursive `inline()` call in each emphasis case and reset at the top of `convert()`. When `emphasize` runs with `emphasisDepth[kind] > 0` it applies:
   - **`discard`** (default) — emit the inner core unwrapped; a single level of emphasis remains. Result: `<i>x <i>y</i> z</i>` → `*x y z*`.
   - **`stack`** — emit the inner core with the `combined` marker (`***` for italic/bold), giving second-order (bold-italic) emphasis. Result: `*x ***y*** z*`.
   - **`reverse`** — the inner should toggle *back* to plain text. Markdown can't express `*x *y* z*` as "y is plain" (CommonMark re-nests it), so the inner core is fenced with a sentinel `REV` (`U+0001`). When the enclosing wrapper wraps its core and finds `REV` fences, it **closes the emphasis before each fenced span and reopens after**, with per-segment whitespace trimming so every delimiter hugs text. Result: `*x* y *z*` (y renders roman). `convert()` strips any stray `REV` sentinels that never met an outer wrapper.

If you add a new emphasis-like tag, give it a depth counter key, increment/decrement around its `inline()` call, and route it through `emphasize` — do not hand-roll the markers.

### Lists, tables, definition lists

- **`list(node, indent, ordered)`** handles `UL`/`OL`, including nesting. Nested `UL`/`OL` inside an `LI` recurse with an increased `indent` (padded to the width of the marker). Block children of an `LI` other than `P` are recursed via `renderBlock`; `P` and inline content are flattened into the item's first line.
- **`table(node)`** builds a GitHub-flavored Markdown pipe table. First row is treated as the header; a separator row is synthesized; cell contents are rendered inline with `|` escaped and newlines flattened. Ragged rows are padded to the max column count.
- **`descList(node)`** maps `DL`/`DT`/`DD` to bold term + `: ` definition lines (a common convention, not standard Markdown).
- **`PRE`** detects a language from a `language-xxx` class on the `<pre>` or its child `<code>` and emits a fenced block.

## 4. Markdown → HTML renderer (Preview only)

`renderMarkdown(md)` and its helpers (`renderSpans`, `renderTable`, `htmlEsc`, `unescapeMd`) exist **only to power the Preview toggle**. They render the Markdown *we just generated*, so the renderer only needs to handle the subset of Markdown this app produces — it is **not** a general-purpose CommonMark implementation and should not be treated as one.

Key mechanics: `renderSpans` protects code spans by swapping them out for `\u0000N\u0000` placeholders before HTML-escaping and applying inline rules, then restores them. `renderMarkdown` is a line scanner handling headings, fenced code, HR, blockquotes, lists (with 2-space-indented nesting), pipe tables, and paragraphs.

If you extend the converter to emit a new Markdown construct, extend this renderer too, or Preview will show it as literal text.

## 4.5. URL scanner and the three output views

**URL scanner.** `extractUrls(text)` scans the **raw input** (not the DOM, not the converted Markdown) for URL-like runs, so it catches URLs anywhere — in `href`/`src` attributes, plain text, comments, even malformed tag soup. The pattern is a scheme (`[a-z][a-z0-9+.\-]*`) followed by `://` and a run of non-whitespace, non-`<>"'\`` characters. Matches with common trailing punctuation (`.,;:!?)]}>'"`) are trimmed, then de-duplicated case-insensitively and sorted alphabetically (case-insensitive `localeCompare`). Note this intentionally only matches `scheme://…` forms, so schemeless-authority URLs and `mailto:`/`tel:` (no `//`) are **not** reported. `buildLinksReport(text)` wraps the result as a Markdown report: a heading line, then one `` - `url` `` bullet per URL (URLs are backtick-wrapped code spans, deliberately **not** clickable links). Empty input yields a "No URLs detected" line.

**Three output views.** The output pane can show one of three things, tracked by a single `view` variable in the wiring: `"source"` (the converted Markdown, default), `"preview"` (rendered HTML via §4), or `"links"` (the report above). The **Preview** and **Links** buttons each toggle their view on/off against `"source"`; they are mutually exclusive because they share one `view` variable — turning one on turns the other off. `applyView()` sets element visibility and button state (`.done` class, Preview's label flips to "Source") and then calls `run()`. `run()` branches on `view` to decide whether to render `buildLinksReport(src)` or `convert(src)` into the `<pre>`, and refreshes the preview pane when `view === "preview"`. Because every input/settings/clear/sample handler calls `run()`, all three views stay live as the user types or changes settings. When adding a fourth view, extend the `view` enum, add a toggle button + handler that flips `view` and calls `applyView()`, and teach `run()`/`applyView()` how to render it.

## 5. Styling and theme

All colors are CSS custom properties in `:root`:

| Variable | Hex | Role |
|---|---|---|
| `--ink` | `#1c1a17` | text / borders |
| `--paper` | `#efe9dd` | page background (beige) |
| `--panel` | `#fbf8f1` | pane-head / footer background |
| `--rule` | `#d8cfbd` | hairline dividers |
| `--accent` | `#3a5a40` | green accents, focus rings |
| `--accent-soft` | `#6a8a5f` | blockquote rule in preview |
| `--amber` | `#b3541e` | crimson/amber highlight, links, favicon arrow |
| `--mute` | `#7b7365` | secondary text |

The favicon uses `--paper` and `--amber` and `--ink` values directly. If you change the palette, regenerate the favicon SVG to match and re-embed it (see §6).

CSS gotcha noted in the design skill and worth repeating: watch selector specificity between type-based (`.section`) and element-based selectors so padding/margin rules don't silently cancel. Keep the accessibility floor: visible keyboard focus (`:focus-visible` outlines), reduced-motion friendliness, and a working mobile layout.

## 6. Regenerating the favicon

The favicon is an inline SVG, base64-encoded into a `data:image/svg+xml;base64,...` URI on the `<link rel="icon">`. To change it:

1. Edit the SVG (circle `fill`/`stroke`, arrow color, geometry).
2. Base64-encode it, e.g. `python3 -c "import base64;print(base64.b64encode(open('favicon.svg','rb').read()).decode())"`.
3. Replace the string after `base64,` in the `<link rel="icon">` tag.

## 6.5. Settings modal

The modal markup is `#settings-overlay` (a fixed full-screen `.overlay` containing a `.modal`), hidden via the `hidden` attribute by default. It is opened by the header **Settings** gear (`#open-settings`). Structure:

- **Tabs** (`.tabs` / `.tab`, ARIA `role="tablist"`) — two tabs: **Images** (`#tabbtn-images` → `#tab-images`) and **Nested emphasis** (`#tabbtn-emphasis` → `#tab-emphasis`). Images is the first/default tab. `selectTab(idx)` toggles the `active` class and `aria-selected`, and shows/hides the matching `.tab-panel`. The tabs are styled as **real tab shapes**: bordered with rounded top corners, inactive tabs recessed (darker fill, pushed down via `top`), and the active tab raised so its background merges into the `.tab-body` below and its bottom border visually disappears into the panel. The panels live inside a `.tab-body` wrapper whose top border the active tab overlaps (`margin-bottom: -2px`).
- **Stable height (do not remove).** `.tab-panel` has a fixed `min-height` (currently `327px`) sized to the *taller* panel (nested emphasis, 3 options). Without it, switching to the shorter Images panel would shrink the modal and slide the tab bar out from under the cursor. If you add options to either panel, re-measure the taller panel's natural `scrollHeight` and update this `min-height` so both tabs keep the modal the same size.
- **Options** are radio groups: `name="images"` (`discard` default / `keep`) and `name="emphasis"` (`reverse` / `stack` / `discard` default). Their `change` handlers set the corresponding `SETTINGS` key and call `run()` for a live re-convert.
- **Close paths** — the ✕ button, the **Done** button, clicking the backdrop, and the Escape key all call `closeSettings()`, which re-hides the overlay and restores focus to the gear.

The defaults in the HTML (`checked` attributes) must stay in sync with the defaults in the `SETTINGS` object: images `discard`, emphasis `discard`. If you add a setting, add it to `SETTINGS`, add a tab/panel or option here, wire a `change` handler that updates `SETTINGS` and calls `run()`, and document it in §3.5.

## 7. Testing changes

There is no test harness in the repo, but the converter functions are pure and can be exercised in Node with `jsdom` providing `DOMParser` and `document`. A quick approach: extract the script text, slice off everything from the `// --- wiring ---` banner onward (that part needs real DOM elements), `eval` the pure-function portion, and call `convert(...)` on sample HTML. Toggle behavior by setting `SETTINGS.discardImages` / `SETTINGS.nestedEmphasis` between calls. `extractUrls` / `buildLinksReport` are pure too — test them with tag soup containing duplicate URLs, mixed schemes (`http`, `https`, `ftp`, `smb`), URLs in attributes vs plain text, and trailing punctuation, expecting a de-duplicated, alphabetically sorted, backtick-wrapped report. Always test at least: a citation/DOI-heavy paragraph (guards against escaping regressions), nested lists, a table, a fenced code block, a paragraph beginning with `1. ` or `- ` (guards `escLineStart`), **adjacent emphasized runs like `<i>a </i><i>b</i>` (guards the boundary-whitespace fix — expect `*a* *b*`, never `*a**b*`), and same-kind nested emphasis under all three modes**. Piping the output through a real CommonMark parser (`commonmark` on npm) is the best way to confirm the Markdown actually renders as intended.

For the modal, output views, and settings wiring, load the whole file in `jsdom` with `runScripts: "dangerously"`, then drive it via `.click()` and dispatched `input`/`change` events — this catches wiring regressions the pure-function tests can't. Worth asserting: toggling **Links** then **Preview** leaves exactly one view active, and editing input while a non-source view is active updates that view. For the modal's stable-height guarantee, a headless browser (Playwright) can compare `.modal` bounding-box heights across both tabs — they must be equal.

Manual smoke test before shipping: open the file, click **Sample**, confirm the output pane looks right, toggle **Links** and **Preview**, click **Copy**, then open **Settings**, switch tabs (confirming the modal doesn't resize), and flip each option while watching the output update.

## 8. Commit conventions

Commit titles are tagged by change type: `[feat]` for new features, `[fix]` for bug fixes, `[refactor]` for internal-only changes, `[docs]` for documentation-only changes, else `[chore]`. Include a short description body and co-author attribution where relevant.

## 9. Common tasks, quick pointers

- **A tag isn't converting** → add a `case` in `renderInline` (inline tag) or `renderBlock` (block tag); update `BLOCK` if it's a new container.
- **Output has spurious backslashes** → look at `esc`, not `escLineStart`.
- **Raw Markdown leaking into prose** → look at `escLineStart`.
- **Emphasis markers touching adjacent text / lost spaces** → the fix belongs in `emphasize`, not the tag cases; never `.trim()` inside a tag case.
- **Changing how nested emphasis or images behave** → adjust `SETTINGS` defaults and/or the branches in `emphasize` and the `IMG` case; keep the modal's `checked` radios in sync.
- **Preview shows literal Markdown** → extend `renderMarkdown` / `renderSpans`.
- **URLs missing/extra in the Links report** → adjust the scheme/character pattern or trailing-punctuation trim in `extractUrls` (§4.5).
- **Adding another output view** → extend the `view` enum, add a toggle button + handler flipping `view` and calling `applyView()`, and teach `run()`/`applyView()` to render it (§4.5).
- **Modal resizes when switching tabs** → the `.tab-panel` `min-height` no longer covers the taller panel; re-measure and update it (§6.5).
- **Palette change** → edit `:root` variables and regenerate the favicon.
- **New button/control** → add markup in the relevant `.pane-head` (or the modal), style it, and wire it in the `// --- wiring ---` section; remember `run()` re-runs conversion and syncs the active view.
- **New setting** → §6.5 has the checklist (SETTINGS key, UI, `change` handler → `run()`, docs).
