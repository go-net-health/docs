<p align="center"><img src="https://raw.githubusercontent.com/go-net-health/brand/main/social/go-net-health.png" alt="go-net-health/docs" width="720"></p>

# go-net-health/docs

Versioned documentation for [go-net-health](https://github.com/go-net-health),
built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/) and
versioned with [mike](https://github.com/jimporter/mike). Published to the
`gh-pages` branch and served at <https://go-net-health.github.io/docs/>.

## Local preview

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements.txt
mkdocs serve                       # http://localhost:8000
mike serve                         # preview the versioned site
```

## Releasing a new docs version

```bash
mike deploy --push --update-aliases <version> latest
mike set-default --push latest
```
