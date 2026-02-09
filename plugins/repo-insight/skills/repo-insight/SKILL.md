---
name: repo-insight
description: Generate or update AI-consumable project documentation (docs/ai/*). Automatically detects whether to run in Generate mode (new project) or Cognize mode (existing docs). Uses token-budget tiers and smart extraction to efficiently analyze codebases of any size.
---

This skill produces a suite of AI-oriented project documentation under `docs/ai/`. It operates in two modes detected automatically, applies token-budget controls, and writes each document to disk immediately after generation.

## Mode Detection

Use the Read tool to directly check `<current working directory>/docs/ai/INDEX.md`. Do NOT use Glob for this check — use the absolute path constructed from the current working directory.

- **Not found** -> **Generate mode** (full documentation creation)
- **Found** -> **Cognize mode** (incremental update / Q&A / regenerate)

---

## Generate Mode

Execute these steps in order. At each file-write step, use the Write tool immediately — do not batch writes.

### Step 1: Project Scan

1. Run `ls` on the project root to get top-level structure.
2. Use Glob to find configuration/manifest files:
   - `**/package.json`, `**/tsconfig.json`, `**/pyproject.toml`, `**/setup.py`, `**/setup.cfg`
   - `**/go.mod`, `**/Cargo.toml`, `**/pom.xml`, `**/build.gradle`, `**/Makefile`
   - `**/.env.example`, `**/docker-compose.yml`, `**/Dockerfile`
3. Read discovered config files to detect language, framework, and dependencies.

### Step 2: Detect Project Profile

Determine:
- **Primary language(s)** and framework(s)
- **Project type**: library / web app / API service / CLI tool / monorepo / other
- **Scale**: count source files (exclude ignored dirs). Use Glob with language-appropriate patterns (e.g. `**/*.ts`, `**/*.py`).

### Step 3: Recommend Token Budget — Wait for User Confirmation

Present the detection results and recommend a budget tier:

| Tier | When to recommend | What gets read | Documents produced |
|------|------------------|----------------|-------------------|
| **Quick** | < 30 source files | Entry points + config files only | `INDEX.md` + `00-overview` |
| **Standard** (default) | 30–150 source files | + core module signatures + architecture | + `01-architecture` through `04-data-model` (skip N/A) |
| **Deep** | > 150 source files | + full reads of core modules | + `05-build-run-test` through `99-open-questions` (skip N/A) |

Use AskUserQuestion to let the user choose a tier. Recommend the tier matching the scale.

### Step 4: Smart Extraction (per chosen tier)

#### Ignore rules — ALWAYS skip these paths:

`node_modules`, `dist`, `build`, `out`, `.next`, `.nuxt`, `.git`, `vendor`, `coverage`, `__pycache__`, `.mypy_cache`, `.pytest_cache`, `*.lock`, `*.min.js`, `*.min.css`, `*.map`, `.DS_Store`, `*.pyc`, `.env`

#### All tiers — read these:

- Entry point files (e.g. `src/index.*`, `src/main.*`, `app.*`, `main.*`, `cmd/main.*`)
- All discovered config/manifest files from Step 1

#### Standard and Deep — additionally extract module signatures:

Use Grep with these language-specific patterns to extract public interfaces WITHOUT reading entire files:

- **JS/TS**: `^export\s+(default\s+)?(function|class|const|interface|type|enum)\s+\w+`
- **Python**: `^(class\s+\w+|def\s+\w+|async\s+def\s+\w+)`
- **Go**: `^func\s+(\(\w+\s+\*?\w+\)\s+)?\w+`
- **Rust**: `^pub\s+(fn|struct|enum|trait|mod|type)\s+\w+`
- **Java/Kotlin**: `^(public|protected)\s+(class|interface|enum|record|fun)\s+\w+`

Search in `src/`, `lib/`, `pkg/`, `internal/`, `app/`, `api/`, `cmd/`, `core/`, `services/`, `models/`, `handlers/`, `controllers/`, `routes/`, or equivalent directories found in Step 1.

#### Deep only — additionally:

- Full Read of core business modules (identified by import frequency or directory name: `core/`, `services/`, `domain/`, `engine/`, `lib/`)
- Read test directory structure (file list only — do NOT read test file contents unless specifically relevant)

#### Long file rule:

If a source file exceeds 500 lines, only read the first 100 lines, then Grep for signatures in the rest. Never read a 500+ line file in full unless it is the primary entry point.

#### Test files:

Count test files and report stats. Do NOT read test file contents (except in Deep tier for critical integration tests).

### Step 5: Generate Documents — Write Each Immediately

Generate documents one at a time. After generating each file, **immediately write it to disk** using the Write tool. Then keep only a 1–2 sentence TL;DR summary in your working memory — discard the full content from context.

#### YAML front-matter for every document:

```yaml
---
generated_by: repo-insight
version: 1
created: YYYY-MM-DD
last_updated: YYYY-MM-DD
source_commit: <git rev-parse --short HEAD, or "n/a" if not a git repo>
coverage: quick|standard|deep
---
```

#### Standard sections for every document:

1. **Purpose** — one sentence explaining what this document covers
2. **TL;DR** — 5–10 bullet points of the most important facts
3. **Canonical Facts** — detailed factual content with source refs
4. **Interfaces / Dependencies** — relevant APIs, imports, contracts (or N/A)
5. **Unknowns & Risks** — anything uncertain, explicitly marked
6. **Source Refs** — file paths (with line numbers where useful)

#### Document catalog:

| File | Content | Min tier |
|------|---------|----------|
| `docs/ai/00-system-overview.md` | Project purpose, tech stack, repo layout, entry points | Quick |
| `docs/ai/01-architecture-map.md` | Architecture style, component diagram, data flow, runtime flow | Standard |
| `docs/ai/02-module-cheatsheet.md` | Module-by-module index: path, role, public interfaces, deps | Standard |
| `docs/ai/03-api-contracts.md` | REST/GraphQL/gRPC/CLI/Event contracts, auth, errors | Standard |
| `docs/ai/04-data-model.md` | Entities, fields, relations, constraints, storage, migrations | Standard |
| `docs/ai/05-build-run-test.md` | Build/run/test/lint/CI commands, env vars, deploy notes | Deep |
| `docs/ai/06-decision-log.md` | Key technical decisions, alternatives considered, trade-offs | Deep |
| `docs/ai/07-glossary.md` | Project-specific terms, definitions, aliases | Deep |
| `docs/ai/99-open-questions.md` | Unresolved questions, gaps, blocking unknowns | Deep |

**Skip any document that would have no meaningful content** — do not generate empty shells.

### Step 6: Generate INDEX.md (last)

After all individual documents are written, generate `docs/ai/INDEX.md` containing:

- Project name and one-line summary
- YAML metadata (same schema as above)
- Table of all generated documents with one-line descriptions
- Recommended reading order
- Coverage tier used
- Generation timestamp

Write to disk immediately.

### Step 7: Offer .gitignore Addition

Ask the user whether to add `docs/ai/` to `.gitignore`. If yes, append the entry. If the project has no `.gitignore`, create one with just that entry.

---

## Cognize Mode

Entered when `docs/ai/INDEX.md` already exists.

### Step 1: Read INDEX.md Metadata

Read `docs/ai/INDEX.md`. Parse the YAML front-matter to extract `source_commit` and `coverage`.

### Step 2: Detect Changes (git repos only)

If the project is a git repo and `source_commit` is not `n/a`:

```bash
git diff <source_commit>..HEAD --name-only
```

Collect the list of changed files.

### Step 3: Present Options to User

Use AskUserQuestion with three choices:

1. **Incremental update** — only re-analyze changed files and update affected documents
2. **Project Q&A** — answer questions about the project using existing docs (load docs on-demand, not all at once)
3. **Full regeneration** — discard existing docs and run Generate mode from scratch

### Incremental Update Flow

1. Map changed files to affected documents:
   - Config/manifest changes -> `00-overview`, `05-build-run-test`
   - Source module changes -> `01-architecture`, `02-module-cheatsheet`
   - API route changes -> `03-api-contracts`
   - Schema/model changes -> `04-data-model`
   - Test changes -> `05-build-run-test`
   - Any change -> `INDEX.md` (update `source_commit` and `last_updated`)
2. For each affected document, re-read relevant source files and regenerate that document only.
3. Write updated documents to disk immediately, updating `last_updated` and `source_commit` in YAML front-matter.
4. Update `INDEX.md` metadata.

### Project Q&A Flow

1. Accept the user's question.
2. Read only the specific `docs/ai/*.md` file(s) relevant to the question — do NOT load all docs.
3. If the docs are insufficient, selectively Read source files to fill gaps.
4. Answer the question with source references.
5. Stay in Q&A mode until the user exits or asks for a different operation.

---

## Token Optimization Rules (mandatory)

These rules are non-negotiable and must be followed throughout execution:

1. **Never read ignored directories** — always apply the ignore list before any Glob or Read.
2. **Signatures before full reads** — always Grep for signatures first; only do full Read when specifically needed.
3. **Write-and-forget** — after writing a document to disk, retain only a 1–2 sentence summary. Do not keep full document text in context.
4. **On-demand loading in Cognize mode** — never bulk-read all docs; read only what the current question or update requires.
5. **Skip empty documents** — if a document would have no meaningful content for this project, do not generate it.
6. **Long file truncation** — files > 500 lines: read first 100 lines + Grep signatures only.
7. **Test files: stats only** — count and list test files but do not read their contents (Deep tier may read critical integration tests).

## Writing Guidelines

- Target audience is **other AI models**, not human beginners.
- Prefer: bullet lists, tables, short declarative sentences, stable terminology.
- Clearly distinguish: **verified fact** vs. **inference** vs. **unknown**.
- Every key claim must include a `source_refs` (file path, optionally with line number).
- Use the project's own terminology consistently; define terms in the glossary.
