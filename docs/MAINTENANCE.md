# Maintenance Guide

Technical reference for anyone (human or AI) maintaining this codebase. Read this before making changes.

## 1. Project shape

The entire application is a **single self-contained file**: `html-to-markdown.html`. There is no build step, no bundler, no package manager, and no runtime dependencies. HTML, CSS, and JavaScript all live in that one file. The favicon is embedded as a base64 data URI in the `<head>`, so the file is fully portable — open it from disk and it works.

```
qdvc-html-to-markdown-SPA/
├── html-to-markdown.html   ← the whole app
├── README.md               ← user-facing cover page
└── docs/
    └── MAINTENANCE.md       ← this file
```

Design constraint to preserve: **keep it a single file with zero dependencies.** The value proposition is "download one file, open it, done." Do not introduce npm packages, external script/CDN tags, or a build pipeline without a deliberate decision to change that contract.

## 2. Anatomy of the file

Reading top to bottom:

1. **`<head>`** — meta tags, `<title>`, and the embedded SVG favicon (a crimson down-arrow on a beige circle, matching the app palette).
2. **`<style>`** — all CSS. The palette is defined as CSS custom properties in `:root` (see §5). Layout is a two-pane CSS grid that collapses to stacked panes under 760px.
3. **`<body>` markup** — header (wordmark + tagline), `<main>` with two `.pane` sections (input textarea, output area), and a footer with live character counts and the GitHub / open-source attribution.
4. **`<script>`** — one IIFE (`(function () { "use strict"; ... })();`) containing all logic. It has three parts, separated by comment banners:
   - the **HTML → Markdown converter** (top),
   - the **Markdown → HTML renderer** used only for the Preview pane (`// --- minimal markdown -> html renderer ---`),
   - the **DOM wiring** (`// --- wiring ---`).

## 3. HTML → Markdown converter (the core)

This is the important part. It parses pasted HTML with the browser's own `DOMParser` (`new DOMParser().parseFromString(html, "text/html")`) and walks the resulting DOM tree, emitting Markdown. Never write a regex-based HTML parser here; the DOM parser is the whole point and it handles malformed input gracefully.

### Two rendering modes

The converter distinguishes **inline** context from **block** context, because Markdown treats them differently.

- **`inline(node)` / `renderInline(n)`** — produce inline Markdown with no block spacing. Handles `BR`, `STRONG`/`B`, `EM`/`I`, `DEL`/`S`/`STRIKE`, `CODE`, `A`, `IMG`, and a set of pass-through wrappers (`SPAN`, `FONT`, `SUB`, `SUP`, `U`, `MARK`, `SMALL`, `ABBR`, `TIME`, `CITE`, `Q`) whose children are rendered but whose tags are dropped. Unknown inline tags fall through to rendering their children.
- **`block(node, indent)` / `renderBlock(n, indent)`** — produce block-level Markdown, joining children with blank lines. Handles headings, `P`, `HR`, `BLOCKQUOTE`, `PRE`, `UL`/`OL`, `DL`, `TABLE`, `FIGURE`, and strips `SCRIPT`/`STYLE`/`NOSCRIPT`/`TEMPLATE`. Unknown tags are classified via `hasBlockChild` / the `BLOCK` set and recursed into as either a block container or an inline run.

`convert(html)` is the entry point: parse → `block(doc.body, "")` → collapse 3+ newlines to 2 → trim.

### The `BLOCK` set and `hasBlockChild`

`BLOCK` is a `Set` of tag names treated as block-level containers. When `renderBlock` hits an unrecognized element, it uses `BLOCK.has(tag) || hasBlockChild(n)` to decide whether to descend as a block (blank-line-separated children) or flatten to an inline run. If you add support for a new block element, add its tag to `BLOCK` and/or give it an explicit `case` in `renderBlock`.

### Escaping — read this before touching it

This is the subtlest and most bug-prone area, and the behavior is **intentional**. There are two escaping functions:

- **`esc(t)`** escapes only characters that are special *mid-text*: `` \ ` * _ [ ] < > `` and stray entity ampersands. It deliberately does **not** escape `.`, `-`, `(`, `)`, `#`, `!`, `|`, `~`, etc., because those are only meaningful at specific positions and escaping them everywhere produces ugly, hard-to-read Markdown (e.g. `Berners\-Lee, T\. \(1989\)`).
- **`escLineStart(line)`** escapes a character only when it appears as the **first token of a line**, where it would otherwise be parsed as a block construct: leading bullet markers (`- + *`), ordered-list markers (`1.` / `1)`), ATX headings (`#`), blockquotes (`>`), and setext/HR rules (`===`, `---`, `___`). It is applied to plain text nodes and to paragraph output in `renderBlock`.

**Rule of thumb:** if a converted document shows backslashes inside ordinary prose (citations, DOIs, sentences), the fix is almost always in `esc`, not `escLineStart`. If raw Markdown syntax leaks through and gets misrendered (a sentence starting with "1. " turning into a list), the fix is in `escLineStart`. Do not revert to blanket escaping.

### Lists, tables, definition lists

- **`list(node, indent, ordered)`** handles `UL`/`OL`, including nesting. Nested `UL`/`OL` inside an `LI` recurse with an increased `indent` (padded to the width of the marker). Block children of an `LI` other than `P` are recursed via `renderBlock`; `P` and inline content are flattened into the item's first line.
- **`table(node)`** builds a GitHub-flavored Markdown pipe table. First row is treated as the header; a separator row is synthesized; cell contents are rendered inline with `|` escaped and newlines flattened. Ragged rows are padded to the max column count.
- **`descList(node)`** maps `DL`/`DT`/`DD` to bold term + `: ` definition lines (a common convention, not standard Markdown).
- **`PRE`** detects a language from a `language-xxx` class on the `<pre>` or its child `<code>` and emits a fenced block.

## 4. Markdown → HTML renderer (Preview only)

`renderMarkdown(md)` and its helpers (`renderSpans`, `renderTable`, `htmlEsc`, `unescapeMd`) exist **only to power the Preview toggle**. They render the Markdown *we just generated*, so the renderer only needs to handle the subset of Markdown this app produces — it is **not** a general-purpose CommonMark implementation and should not be treated as one.

Key mechanics: `renderSpans` protects code spans by swapping them out for `\u0000N\u0000` placeholders before HTML-escaping and applying inline rules, then restores them. `renderMarkdown` is a line scanner handling headings, fenced code, HR, blockquotes, lists (with 2-space-indented nesting), pipe tables, and paragraphs.

If you extend the converter to emit a new Markdown construct, extend this renderer too, or Preview will show it as literal text.

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

## 7. Testing changes

There is no test harness in the repo, but the converter functions are pure and can be exercised in Node with `jsdom` providing `DOMParser` and `document`. A quick approach: extract the script text, slice off everything from the `// --- wiring ---` banner onward (that part needs real DOM elements), `eval` the pure-function portion, and call `convert(...)` on sample HTML. Always test at least: a citation/DOI-heavy paragraph (guards against escaping regressions), nested lists, a table, a fenced code block, and a paragraph beginning with `1. ` or `- ` (guards `escLineStart`).

Manual smoke test before shipping: open the file, click **Sample**, confirm the output pane looks right, toggle **Preview**, and click **Copy**.

## 8. Commit conventions

Commit titles are tagged by change type: `[feat]` for new features, `[fix]` for bug fixes, `[refactor]` for internal-only changes, `[docs]` for documentation-only changes, else `[chore]`. Include a short description body and co-author attribution where relevant.

## 9. Common tasks, quick pointers

- **A tag isn't converting** → add a `case` in `renderInline` (inline tag) or `renderBlock` (block tag); update `BLOCK` if it's a new container.
- **Output has spurious backslashes** → look at `esc`, not `escLineStart`.
- **Raw Markdown leaking into prose** → look at `escLineStart`.
- **Preview shows literal Markdown** → extend `renderMarkdown` / `renderSpans`.
- **Palette change** → edit `:root` variables and regenerate the favicon.
- **New button/control** → add markup in the relevant `.pane-head`, style it, and wire it in the `// --- wiring ---` section; remember `run()` re-runs conversion and syncs Preview when active.
