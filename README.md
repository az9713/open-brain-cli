<p align="center">
  <strong>CLI edition of <a href="https://github.com/NateBJones-Projects/OB1">Open Brain (OB1)</a> — the personal AI memory system by Nate B. Jones.</strong><br>
  This repo adds a lightweight CLI tool (<code>ob</code>) for terminal-first workflows and strips the MCP requirement for core usage.
</p>

<p align="center">
  <img src=".github/ob1-logo-wide.png" alt="Open Brain CLI" width="600">
</p>

<h1 align="center">Open Brain CLI</h1>

A CLI-first personal AI memory system. One database (Supabase + pgvector), one command-line tool (`ob`), any AI. Capture thoughts, search by meaning, and give every AI tool you use a shared persistent memory — all from your terminal.

```bash
ob capture "Sarah mentioned she's thinking about leaving her job to start a consulting business"
ob search "career changes"    # finds Sarah's thought by meaning, not keywords
ob recent 5                   # latest thoughts
ob stats                      # knowledge base overview
```

No MCP server required. No Edge Function deployment. No Node.js. Just `curl`, `jq`, and three environment variables.

> **Note:** The `ob` CLI is under active development and testing. Expect rough edges — feedback and bug reports are welcome via [GitHub Issues](https://github.com/az9713/open-brain-cli/issues).

> Open Brain was originally created by [Nate B. Jones](https://natesnewsletter.substack.com/). This repository is the CLI edition, maintained at [github.com/az9713/open-brain-cli](https://github.com/az9713/open-brain-cli).

## How It Works

```
You type:  ob capture "Met with design team, decided to cut sidebar panels"
           │
           ├─► OpenRouter generates a 1536-dim embedding (meaning vector)
           ├─► gpt-4o-mini extracts metadata (topics, people, action items)
           └─► Both stored in Supabase PostgreSQL (thoughts table + pgvector)

You type:  ob search "dashboard redesign decisions"
           │
           ├─► Your query gets embedded
           └─► pgvector finds semantically similar thoughts (not keyword matching)
```

## Setup

Follow the **[Setup Guide](docs/01-getting-started.md)** — it covers everything from creating your Supabase database to capturing your first thought. Takes about 10 minutes.

Already set up? Here's the install shortcut:

```bash
git clone https://github.com/az9713/open-brain-cli.git
mkdir -p ~/.local/bin
cp open-brain-cli/resources/ob-cli/ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob

# Set env vars — choose one method:

# Option A: Copy .env.example and fill in your values
cp open-brain-cli/.env.example ~/.ob.env
# Edit ~/.ob.env with your actual keys, then:
export OB_ENV_FILE="$HOME/.ob.env"  # add to ~/.bashrc or ~/.zshrc

# Option B: Export directly (add to ~/.bashrc or ~/.zshrc)
export OB_SUPABASE_URL="https://your-project-ref.supabase.co"
export OB_SUPABASE_KEY="your-service-role-key"
export OB_OPENROUTER_KEY="your-openrouter-key"

# Verify
ob check
```

## Commands

| Command | Description |
|---|---|
| `ob capture "text"` | Save a thought with embedding + metadata |
| `ob search "query" [--threshold N] [--count N] [--json]` | Semantic search across thoughts |
| `ob recent [count]` | List recent thoughts (default: 10) |
| `ob delete <thought-id>` | Delete a thought by ID |
| `ob stats` | Knowledge base statistics |
| `ob check` | Verify connectivity to Supabase + OpenRouter |
| `ob version` | Print version and config status |

### Environment Variables

The `ob` CLI automatically loads a `.env` file from the current directory if one exists. You can also point to a specific file with `OB_ENV_FILE`. Shell exports take precedence over `.env` values.

| Variable | Required | Default | Description |
|---|---|---|---|
| `OB_SUPABASE_URL` | Yes | — | Supabase project URL |
| `OB_SUPABASE_KEY` | Yes | — | Supabase service role key |
| `OB_OPENROUTER_KEY` | Yes | — | OpenRouter API key |
| `OB_THRESHOLD` | No | `0.7` | Similarity threshold for search |
| `OB_COUNT` | No | `10` | Default result count |
| `OB_ENV_FILE` | No | `.env` | Path to environment file |

## Using with AI Tools (Claude Code, Codex, Gemini CLI)

This repo includes 6 skills that let AI tools use Open Brain via natural language — no need to memorize CLI commands.

### What You Can Say

| Instead of... | Just say... | Skill |
|---|---|---|
| `ob capture "Sarah mentioned..."` | "Remember this: Sarah mentioned she's thinking about starting a consulting business" | ob-capture |
| `ob search "career changes"` | "What do I know about career changes?" | ob-recall |
| `ob stats` + `ob recent 50` | "Give me a weekly review of my brain" | ob-review |
| `ob check` + `ob stats` | "Is my Open Brain working?" | ob-status |
| (manual process) | "Import my notes from Notion into Open Brain" | ob-migrate |
| `ob search` + `ob delete` | "Find and remove duplicate thoughts" | ob-cleanup |

### More Examples

**Capturing naturally:**
- "Save this: met with design team, decided to cut sidebar panels, API spec due Thursday"
- "Don't forget — Marcus wants to move to the platform team"
- "Note: the living room paint is Sherwin Williams Sea Salt SW 6204"

**Recalling by meaning:**
- "What did I capture about the API redesign?"
- "What do I know about Marcus?"
- "Find my notes on dashboard decisions"

**Reviews and synthesis:**
- "What action items do I have open?"
- "What themes have I been thinking about this week?"
- "Give me a monthly review"

**Maintenance:**
- "That last capture was wrong — fix it"
- "Clean up my brain — find any duplicates"
- "Delete the test thought I captured earlier"

### How Skills Work

Skills are loaded automatically when you clone this repo and open it in Claude Code. When you say something like "remember this meeting note," Claude reads the `ob-capture` skill and translates your request into the right `ob` CLI commands with proper formatting for good metadata extraction.

The skills live in `.claude/skills/` — you can read or customize them.

## GUI Clients (Optional)

If you also use GUI-based AI tools (Claude Desktop, ChatGPT, Cursor), you can deploy an MCP server alongside the CLI. Both paths read and write the same `thoughts` table. See the [original Open Brain repo](https://github.com/NateBJones-Projects/OB1) for MCP deployment instructions.

## Documentation

| Document | What It Covers |
| -------- | -------------- |
| [Setup Guide](docs/01-getting-started.md) | Complete setup: database, API keys, CLI install, verification |
| [CLI Reference](resources/ob-cli/README.md) | Full `ob` CLI documentation, flags, exit codes |
| [Quick Start Guide](docs/QUICKSTART.md) | First 10 wins with your Open Brain |
| [User Guide](docs/USER_GUIDE.md) | Use cases, tips, and troubleshooting |
| [API Reference](docs/API_REFERENCE.md) | `ob` CLI reference, MCP tools, database schemas |
| [Architecture](docs/ARCHITECTURE.md) | System design, data flows, security boundaries |
| [Developer Guide](docs/DEVELOPER_GUIDE.md) | Build extensions, recipes, and integrations |
| [Companion Prompts](docs/02-companion-prompts.md) | Memory migration, use case discovery, weekly review |
| [FAQ](docs/03-faq.md) | Common issues and architecture questions |

## Extensions

Build these in order. Each one adds a new capability to your Open Brain. Extensions use their own MCP servers for GUI clients, but all read from the same Supabase database.

| # | Extension | What You Build | Difficulty |
| --- | --------- | -------------- | ---------- |
| 1 | [Household Knowledge Base](extensions/household-knowledge/) | Home facts your agent can recall instantly | Beginner |
| 2 | [Home Maintenance Tracker](extensions/home-maintenance/) | Scheduling and history for home upkeep | Beginner |
| 3 | [Family Calendar](extensions/family-calendar/) | Multi-person schedule coordination | Intermediate |
| 4 | [Meal Planning](extensions/meal-planning/) | Recipes, meal plans, shared grocery lists | Intermediate |
| 5 | [Professional CRM](extensions/professional-crm/) | Contact tracking wired into your thoughts | Intermediate |
| 6 | [Job Hunt Pipeline](extensions/job-hunt/) | Application tracking and interview pipeline | Advanced |

## Community Contributions

| Directory | What's Inside | Status |
|---|---|---|
| [`/recipes`](recipes/) | Step-by-step capability builds (email import, ChatGPT import, daily digest) | Open |
| [`/schemas`](schemas/) | Database table extensions (CRM contacts, preferences, reading list) | Open |
| [`/dashboards`](dashboards/) | Frontend templates for Vercel/Netlify | Open |
| [`/integrations`](integrations/) | Capture sources and webhook receivers | Open |
| [`/primitives`](primitives/) | Reusable concepts (RLS, shared MCP server) | Curated |

See [CONTRIBUTING.md](CONTRIBUTING.md) for contribution guidelines.

## Credits

Open Brain was originally created by [Nate B. Jones](https://natesnewsletter.substack.com/). This CLI edition is maintained by [az9713](https://github.com/az9713).

## License

[FSL-1.1-MIT](LICENSE.md)
