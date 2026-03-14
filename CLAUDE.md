# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Repo Is

Open Brain CLI is a CLI-first personal AI memory system — one database (Supabase + pgvector), one CLI tool (`ob`), any AI. This repo provides the `ob` CLI for direct terminal access, plus extensions, recipes, schemas, dashboards, integrations, and comprehensive docs. Originally based on [Open Brain by Nate B. Jones](https://github.com/NateBJones-Projects/OB1).

**License:** FSL-1.1-MIT. No commercial derivative works.

## Repo Structure

```
extensions/     — Curated, ordered learning path (6 builds). Do NOT add without maintainer approval.
primitives/     — Reusable concept guides (must be referenced by 2+ extensions). Curated.
recipes/        — Standalone capability builds. Open for community contributions.
schemas/        — Database table extensions. Open.
dashboards/     — Frontend templates (Vercel/Netlify). Open.
integrations/   — MCP extensions, webhooks, capture sources. Open.
docs/           — Setup guides, FAQ, companion prompts.
resources/      — Claude Skill, companion files, ob CLI tool.
```

Every contribution in recipes/schemas/dashboards/integrations/extensions/primitives must live in its own subfolder and include `README.md` + `metadata.json`.

## Environment Setup

Required environment variables (see `.env.example` template):

```
OB_SUPABASE_URL    — Supabase project URL
OB_SUPABASE_KEY    — Supabase service role key
OB_OPENROUTER_KEY  — OpenRouter API key
```

Optional: `OB_THRESHOLD` (default 0.7), `OB_COUNT` (default 10).

## Common Commands

```bash
# Verify connectivity
ob check

# Test the CLI
ob capture "test thought"
ob search "test"
ob recent 5
ob stats
ob delete <thought-id>

# Validate a contribution's metadata.json against schema
# (no build system — this is a docs/scripts repo, not an app)
```

There is no build step, test runner, or linter for the repo itself. The `ob` CLI (`resources/ob-cli/ob`) is a standalone bash script with only `curl` and `jq` as dependencies. Validate changes by running `ob check` and testing commands manually.

GitHub Actions run `markdown-lint` on PRs (`.github/workflows/markdown-lint.yml`).

## Claude Skills

Seven skills in `.claude/skills/` let Claude translate natural language into `ob` CLI commands:

- `ob-capture` — "Remember this: ..." → `ob capture`
- `ob-recall` — "What do I know about ...?" → `ob search`
- `ob-review` — "Give me a weekly review" → `ob stats` + `ob recent`
- `ob-status` — "Is my Open Brain working?" → `ob check` + `ob stats`
- `ob-migrate` — "Import my notes from ..." → bulk `ob capture`
- `ob-cleanup` — "Find duplicates" → `ob search` + `ob delete`
- `review-pr` — Automated PR review against contribution rules

## Guard Rails

- **Never modify the core `thoughts` table structure.** Adding columns is fine; altering or dropping existing ones is not.
- **No credentials, API keys, or secrets in any file.** Use environment variables.
- **No binary blobs** over 1MB. No `.exe`, `.dmg`, `.zip`, `.tar.gz`.
- **No `DROP TABLE`, `DROP DATABASE`, `TRUNCATE`, or unqualified `DELETE FROM`** in SQL files.

## Architecture — The Two Access Paths

Open Brain has two ways to access the same Supabase PostgreSQL database:

```
MCP path (GUI clients like Claude Desktop, ChatGPT):
  AI Client → MCP Server (Edge Function) → OpenRouter + Supabase PostgreSQL

CLI-Direct path (terminal tools like Claude Code, Codex, Gemini CLI):
  AI Tool → ob CLI (bash, curl+jq) → OpenRouter + Supabase PostgreSQL
```

Both paths read/write the same `thoughts` table and are fully interoperable. The `ob` CLI tool lives at `resources/ob-cli/ob` and provides 7 commands: `capture`, `search`, `recent`, `delete`, `stats`, `check`, `version`. CLI-captured thoughts include `"source": "ob-cli"` in metadata.

The core data flow for both paths: user text → OpenRouter embedding (text-embedding-3-small) + metadata extraction (gpt-4o-mini) → INSERT into `thoughts` table with pgvector embedding.

## Automated PR Review

`.github/workflows/ob1-review.yml` runs 13 checks on every PR to `main`. Key rules:

- PR title must start with `[category]` (e.g., `[recipes]`, `[docs]`)
- Branch convention: `contrib/<github-username>/<short-description>`
- Commit prefixes: `[category]` matching contribution type
- Contributions need `README.md` + valid `metadata.json` (see `.github/metadata.schema.json` and `CONTRIBUTING.md` for required fields)
- `[docs]` prefix or PRs that don't touch contribution dirs skip contribution checks
- Contribution README must include: Prerequisites, step-by-step instructions, expected outcome sections
- Category-specific: schemas need `.sql`, dashboards need frontend code, extensions need both `.sql` and code, primitives need 200+ word README
- Declared `requires_primitives` must exist in `primitives/` and be linked in README

## Key Files

- `CONTRIBUTING.md` — Source of truth for contribution rules, metadata format, review process, 11+ automated checks
- `.github/workflows/ob1-review.yml` — Automated PR review
- `.github/metadata.schema.json` — JSON schema for metadata.json validation
- `docs/CLI_DIRECT_APPROACH.md` — Full CLI-Direct architecture guide with ob CLI design
- `docs/ARCHITECTURE.md` — System architecture with ASCII diagrams
- `docs/API_REFERENCE.md` — `ob` CLI reference, all 35+ MCP tools, database schemas, environment variables
- `docs/DEVELOPER_GUIDE.md` — Developer guide for building extensions/recipes/integrations
- `resources/ob-cli/ob` — The ob CLI bash script (curl + jq, no other runtime deps)
