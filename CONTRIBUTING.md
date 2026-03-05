# Contributing to the Docs Hub

This documentation hub aggregates docs from multiple project repositories.

## How to Add a New Project

Adding a new project requires **two changes**:

### 1. Update the workflow (`.github/workflows/deploy.yml`)

Add your project to the `PROJECTS` environment variable:

```yaml
env:
  PROJECTS: "GGE:gge,MixFlow:mixflow,Pear:pear,NewProject:newproject"
```

Format: `RepoName:subpath`
- `RepoName` = GitHub repo slug (under AndreaRubbi/)
- `subpath` = folder name under `docs/` (lowercase recommended)

### 2. Update navigation (`mkdocs.yml`)

Add an entry to the `nav` section:

```yaml
nav:
  - Home: index.md
  - Projects:
    - GGE: gge/index.md
    - MixFlow: mixflow/index.md
    - Pear: pear/index.md
    - NewProject: newproject/index.md  # Add this line
```

### 3. (Optional) Update landing page (`docs/index.md`)

Add a card for the new project on the landing page.

## Project Documentation Requirements

Each project repository should have:

```
your-project/
├── docs/
│   ├── index.md          # Required: main documentation page
│   ├── getting-started.md  # Optional
│   └── ...
└── ...
```

The hub will automatically:
- Clone your project's `docs/` folder during CI
- Create a placeholder if `docs/` or `index.md` is missing

## Local Development

To test the docs site locally:

```bash
# Install dependencies
pip install mkdocs-material pymdown-extensions

# Serve locally (without syncing external projects)
mkdocs serve

# Or build static site
mkdocs build
```

Note: Local builds won't include synced project docs unless you manually copy them.

## Deployment

The site deploys automatically via GitHub Actions when you push to `main`.

Manual deployment can be triggered from the Actions tab.
