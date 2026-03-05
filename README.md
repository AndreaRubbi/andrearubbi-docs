# Andrea Rubbi Docs Hub

[![Deploy Docs](https://github.com/AndreaRubbi/andrearubbi-docs/actions/workflows/deploy.yml/badge.svg)](https://github.com/AndreaRubbi/andrearubbi-docs/actions/workflows/deploy.yml)

Documentation hub for my open-source projects, built with [MkDocs Material](https://squidfunk.github.io/mkdocs-material/).

**Live site**: [docs.andrearubbi.com](https://docs.andrearubbi.com)

## Projects

| Project | Description | Docs |
|---------|-------------|------|
| **GGE** | Gene Expression Generative Model Evaluation | [docs.andrearubbi.com/gge](https://docs.andrearubbi.com/gge/) |
| **MixFlow** | Flow-based generative models | [docs.andrearubbi.com/mixflow](https://docs.andrearubbi.com/mixflow/) |
| **Pear** | Coming soon | [docs.andrearubbi.com/pear](https://docs.andrearubbi.com/pear/) |

## How It Works

This hub aggregates documentation from multiple project repositories:

1. Each project maintains its own `docs/` folder
2. GitHub Actions clones each project's docs during CI
3. MkDocs builds a unified site
4. Deployed to GitHub Pages

## Adding a New Project

See [CONTRIBUTING.md](CONTRIBUTING.md) for instructions.

## Local Development

```bash
pip install mkdocs-material pymdown-extensions
mkdocs serve
```

## License

MIT
