# EDE Framework — Documentation

Live site: **<https://the-source-core.github.io/ede-docs/>**

This repository is an **auto-synced mirror** of the documentation that lives inside the private `the-framework/ede-framework` code repo. A GitHub Action in the code repo pushes `docs/` + `mkdocs.yml` here on every commit to `main`. Pages then rebuilds the site from MkDocs.

## Do not hand-edit `docs/` or `mkdocs.yml` here

Any direct edit to those files will be **overwritten** by the next sync. Open a PR or commit against the private code repo instead — the change flows back here automatically.

The only files maintained directly in this repo are:

-   `.github/workflows/pages.yml` — the Pages build pipeline.
-   `README.md` — this file.

Everything else under `docs/`, `mkdocs.yml`, and `overrides/` is overwritten on each sync.

## How the pipeline works

```
ede-framework (private)              ede-docs (this repo, public)
    │                                     │
    │  push to main                       │
    │  (docs/** or mkdocs.yml)            │
    │                                     │
    ▼                                     │
sync-docs.yml ─────► clones this repo ────►  commits mirror of docs/ + mkdocs.yml
                                          │
                                          ▼
                                     pages.yml builds MkDocs site
                                          │
                                          ▼
                                     GitHub Pages serves it
```

## Local preview

If you want to preview a change locally before it's synced:

```bash
git clone git@github.com:the-source-core/ede-docs.git
cd ede-docs
pip install mkdocs-material pymdown-extensions
mkdocs serve     # http://127.0.0.1:8000
```

Remember: any change you commit here will be overwritten on the next sync. Use this only to *inspect* what's already published.
