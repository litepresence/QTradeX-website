# QTradeX Website

Landing page + documentation site for the [QTradeX Algo Trading SDK](https://github.com/squidKid-deluxe/QTradeX-Algo-Trading-SDK).

## Structure

```
├── index.html, main.css, icons/   → Landing page (served at /)
├── raw-docs/                       → MkDocs source (docs markdown)
│   ├── mkdocs.yml
│   ├── getting-started.md, api/*, ...
│   └── extra.css
└── dist/                           → Build artifact (gitignored, ephemeral)
    ├── index.html, icons/...       → Landing page copied from root
    └── docs/                       → MkDocs output (served at /docs/)
```

## Development

```bash
pip install mkdocs mkdocs-material
mkdocs build --strict -f raw-docs/mkdocs.yml
```

The built site lands in `dist/`. Open `dist/index.html` to preview the landing page, `dist/docs/index.html` for docs.

## Deployment

Push to `main`. GitHub Actions builds everything and deploys to GitHub Pages at [qtradex.litepresence.com](https://qtradex.litepresence.com/).

- `/` — landing page
- `/docs/` — documentation

Static assets are copied into the deploy artifact alongside the MkDocs output. No `gh-pages` branch, no committed build artifacts.

## License

WTFPL
