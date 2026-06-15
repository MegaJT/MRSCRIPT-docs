# Tablix / MRScript documentation

The public documentation site for **MRScript** — the tabulation DSL behind Tablix.
A tutorial + how-to recipes + the full language reference, built with
[Material for MkDocs](https://squidfunk.github.io/mkdocs-material/).

**Live site:** https://megajt.github.io/MRSCRIPT-docs/

> This repository contains **documentation only** — the Tablix engine source lives in
> a separate, private repository.

## Edit & preview locally

```powershell
pip install -r requirements.txt
mkdocs serve            # live preview at http://127.0.0.1:8000 (auto-reloads on save)
```

Edit any `.md` file under `docs/`; the browser reloads as you type.

## Publishing

Pushing to `main` triggers `.github/workflows/docs.yml`, which builds this site and
deploys it to **GitHub Pages**. Update the Markdown, push, and the live site refreshes
in about a minute — no files to send to anyone.

> One-time setup: **Settings → Pages → Build and deployment → Source = GitHub Actions**.

## Layout

```
.
├── mkdocs.yml              # site config, theme, navigation
├── requirements.txt        # mkdocs-material + extensions
├── .github/workflows/      # docs.yml — build + deploy to GitHub Pages
└── docs/
    ├── index.md            # home / landing
    ├── getting-started/    # what Tablix is, install, first run
    ├── tutorial/           # learn-by-doing walkthrough
    ├── guide/              # how-to recipes (task-oriented)
    ├── reference/          # the complete language reference (Parts 1–8)
    ├── assets/             # logo / favicon
    ├── stylesheets/        # extra.css (incl. MRScript syntax colours)
    └── javascripts/        # mrscript.js (highlight.js language definition)
```

## MRScript syntax highlighting

Code blocks tagged ` ```mrs ` (or `mrscript`) are coloured by a small custom
highlight.js grammar in `docs/javascripts/mrscript.js`; token colours live in
`docs/stylesheets/extra.css` and adapt to light/dark mode. When the language gains a
new keyword, add it to the keyword list in that JS file too.
