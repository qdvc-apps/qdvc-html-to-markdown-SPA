# qdvc-html-to-markdown-SPA

**Quick and Dirty, Vibe-Coded (QDVC)** single-page app. Paste HTML, get clean Markdown. Runs entirely in your browser — no server, no upload, no tracking.

- Vibe-coding process is fully documented in [vibe-coding/](vibe-coding/)
- Technical maintenance notes in [docs/MAINTENANCE.md](docs/MAINTENANCE.md)

## What it does

Drop in an HTML snippet and the app converts it to readable Markdown as you type. It handles the things you actually run into:

- Headings, paragraphs, and line breaks
- **Bold**, *italic*, ~~strikethrough~~, and `inline code`
- Links and images (with titles)
- Ordered, unordered, and nested lists
- Fenced code blocks with language detection
- Blockquotes, tables, horizontal rules, and figures with captions

The output is tuned to stay **human-readable** — it won't sprinkle backslashes through your citations, DOIs, or sentences the way naive converters do. It only escapes characters that would genuinely be misread as Markdown.

## Using it

Just open the page.

1. Paste your HTML into the left pane.
2. Read the Markdown on the right.
3. Hit **Preview** to see it rendered, or **Copy** to grab it.

There's a **Sample** button if you want to see it in action first.

## Running locally

It's one file. Download `html-to-markdown.html` and open it in any modern browser. That's the whole install.

## Good to know

- Everything happens client-side — your content never leaves your machine.
- Best-effort by design: unusual or malformed HTML degrades gracefully instead of breaking.
