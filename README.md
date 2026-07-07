<p align="center"><img src="https://raw.githubusercontent.com/go-lsp-bridge/brand/main/social/go-lsp-bridge.png" alt="go-lsp-bridge/docs" width="720"></p>

# go-lsp-bridge/docs

Versioned documentation for [go-lsp-bridge](https://github.com/go-lsp-bridge),
built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and
versioned with [mike](https://github.com/jimporter/mike). Published to the
`gh-pages` branch and served at <https://go-lsp-bridge.github.io/docs/>.

The organization landing page ([go-lsp-bridge.github.io](https://go-lsp-bridge.github.io))
links here.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000 (current sources)
mike serve                         # preview the versioned site
```
