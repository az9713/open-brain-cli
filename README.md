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

## Quick Start

### Prerequisites

- A Supabase project (free tier) with the `thoughts` table and `match_thoughts()` function — see [Database Setup](docs/01-getting-started.md#step-1-create-your-supabase-project) (Steps 1-3)
- An [OpenRouter](https://openrouter.ai) API key ($5 credit lasts months)
- `curl` and `jq` installed
- **Windows:** Requires Git Bash, WSL, or similar bash environment

### Install

```bash
# Copy the CLI to your PATH
mkdir -p ~/.local/bin
cp resources/ob-cli/ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob

# Or clone and install
git clone https://github.com/az9713/open-brain-cli.git
cp open-brain-cli/resources/ob-cli/ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob
```

### Configure

Add to `~/.bashrc` or `~/.zshrc`:

```bash
export OB_SUPABASE_URL="https://your-project-ref.supabase.co"
export OB_SUPABASE_KEY="your-service-role-key"
export OB_OPENROUTER_KEY="your-openrouter-key"
```

### Verify

```bash
ob check     # test connectivity to Supabase + OpenRouter
ob capture "My first Open Brain thought"
ob search "first thought"
ob recent
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

| Variable | Required | Default | Description |
|---|---|---|---|
| `OB_SUPABASE_URL` | Yes | — | Supabase project URL |
| `OB_SUPABASE_KEY` | Yes | — | Supabase service role key |
| `OB_OPENROUTER_KEY` | Yes | — | OpenRouter API key |
| `OB_THRESHOLD` | No | `0.7` | Similarity threshold for search |
| `OB_COUNT` | No | `10` | Default result count |

## GUI Clients (Optional)

If you also use GUI-based AI tools (Claude Desktop, ChatGPT, Cursor), you can deploy the MCP server alongside the CLI. Both paths read and write the same `thoughts` table. See the [Full Setup Guide](docs/01-getting-started.md) for MCP deployment.

## Documentation

| Document | What It Covers |
| -------- | -------------- |
| [CLI Reference](resources/ob-cli/README.md) | Full `ob` CLI documentation |
| [CLI Architecture](docs/CLI_DIRECT_APPROACH.md) | CLI-Direct architecture, AI tool configuration |
| [Quick Start Guide](docs/QUICKSTART.md) | First 10 wins with your Open Brain |
| [User Guide](docs/USER_GUIDE.md) | Use cases, tips, and troubleshooting |
| [API Reference](docs/API_REFERENCE.md) | `ob` CLI reference, MCP tools, database schemas |
| [Architecture](docs/ARCHITECTURE.md) | System design, data flows, security boundaries |
| [Developer Guide](docs/DEVELOPER_GUIDE.md) | Build extensions, recipes, and integrations |
| [Setup Guide](docs/01-getting-started.md) | Full system build with MCP (for GUI clients) |
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
