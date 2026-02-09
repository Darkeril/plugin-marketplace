# Darkeril-plugins

A collection of Claude Code plugins.

## Available Plugins

| Plugin | Description | Version |
|--------|-------------|---------|
| [repo-insight](plugins/repo-insight/) | Generate or update AI-consumable project documentation | 1.0.0 |

## Installation

### 1. Add this marketplace

```
/plugin marketplace add https://github.com/Darkeril/plugin-marketplace
```

### 2. Install a plugin

```
/plugin install repo-insight@Darkeril-plugins
```

## Plugin Details

### repo-insight

Automatically generates a suite of AI-oriented project documentation under `docs/ai/`.

**Features:**

- **Auto mode detection** — detects whether to create new docs (Generate) or update existing ones (Cognize)
- **Token-budget tiers** — Quick / Standard / Deep, matched to project scale
- **Smart extraction** — uses signatures and truncation to minimize token usage
- **Incremental updates** — only regenerates docs affected by code changes
- **Project Q&A** — answer questions about the project using generated docs

**Generated documents:**

| File | Content |
|------|---------|
| `00-system-overview.md` | Tech stack, repo layout, entry points |
| `01-architecture-map.md` | Architecture style, component diagram, data flow |
| `02-module-cheatsheet.md` | Module index with public interfaces |
| `03-api-contracts.md` | REST/GraphQL/gRPC/CLI contracts |
| `04-data-model.md` | Entities, relations, migrations |
| `05-build-run-test.md` | Build, run, test, deploy commands |
| `06-decision-log.md` | Key technical decisions |
| `07-glossary.md` | Project-specific terms |
| `99-open-questions.md` | Unresolved questions |

## License

MIT
