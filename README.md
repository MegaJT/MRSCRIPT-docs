# Tablix / MRScript documentation site

A [Material for MkDocs](https://squidfunk.github.io/mkdocs-material/) static site:
a tutorial + how-to recipes + the full MRScript language reference, written for DP
users of all levels.

## Edit & preview locally

```powershell
# from this folder (docs-site/)
pip install -r requirements.txt
mkdocs serve            # live preview at http://127.0.0.1:8000 (auto-reloads on save)
```

Edit any `.md` file under `docs/`; the browser reloads as you type.

## Build the static site

```powershell
mkdocs build            # renders into ./site  (git-ignored)
```

## Publishing

Pushing to `main` triggers `.github/workflows/docs.yml`, which builds this site and
deploys it to **GitHub Pages**. No files need to be sent to anyone — update the
Markdown, push, and the live site refreshes in about a minute.

> One-time setup: in the GitHub repo, go to **Settings → Pages → Build and deployment
> → Source = GitHub Actions**. After the first successful run the site is live at the
> `site_url` set in `mkdocs.yml`.

## Layout

```
docs-site/
├── mkdocs.yml              # site config, theme, navigation
├── requirements.txt        # mkdocs-material + extensions
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
`docs/stylesheets/extra.css` and adapt to light/dark mode. When you add a new
keyword to the language, add it to the keyword list in that JS file too.

## Source of truth

The reference pages were ported from `../docs/MRScript_Reference.txt`. When the
language changes, update the reference pages here (and, if you still keep it, the
`.txt`).
