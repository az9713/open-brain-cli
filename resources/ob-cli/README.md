# ob CLI

A lightweight command-line interface for Open Brain personal memory. Uses `curl` and `jq` to talk directly to Supabase and OpenRouter — no MCP, no Edge Function, no runtime dependencies.

## Prerequisites

- `curl` and `jq` installed
- A Supabase project with the `thoughts` table, `match_thoughts()` function, and indexes (see `docs/01-getting-started.md` Steps 1–3)
- An OpenRouter API key
- **Windows:** Requires Git Bash, WSL, or similar bash environment

## Installation

```bash
# Copy to a directory in your PATH
mkdir -p ~/.local/bin
cp resources/ob-cli/ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob

# Add to PATH if needed (in ~/.bashrc or ~/.zshrc)
export PATH="$HOME/.local/bin:$PATH"
```

## Configuration

Set these environment variables (in `~/.bashrc`, `~/.zshrc`, or `~/.profile`):

```bash
export OB_SUPABASE_URL="https://your-project-ref.supabase.co"
export OB_SUPABASE_KEY="your-service-role-key"
export OB_OPENROUTER_KEY="your-openrouter-key"
```

Optional:

| Variable | Default | Description |
|---|---|---|
| `OB_THRESHOLD` | `0.7` | Similarity threshold for search |
| `OB_COUNT` | `10` | Default result count |

## Commands

| Command | Description |
|---|---|
| `ob capture "thought text"` | Save a thought with embedding + metadata |
| `ob search "query" [--threshold N] [--count N] [--json]` | Semantic search |
| `ob recent [count]` | List recent thoughts |
| `ob delete <thought-id>` | Delete a thought by ID |
| `ob stats` | Knowledge base statistics |
| `ob check` | Verify connectivity to Supabase + OpenRouter |
| `ob version` | Version and config status |

## Verification

After installation and configuration:

```bash
# 1. Check syntax
bash -n ~/.local/bin/ob

# 2. Verify version and config
ob version

# 3. Test connectivity
ob check

# 4. Capture a test thought
ob capture "Testing my Open Brain CLI setup"

# 5. Search for it
ob search "CLI setup"

# 6. View recent thoughts
ob recent 5
```

## Exit Codes

| Code | Meaning |
|---|---|
| `0` | Success |
| `1` | Error (missing config, API failure, not found) |

## Source Tagging

Thoughts captured via the `ob` CLI include `"source": "ob-cli"` in their metadata, so you can distinguish them from MCP-captured thoughts.

## Natural Language Usage (AI Tools)

If you use Claude Code, Codex, or Gemini CLI, you don't need to type `ob` commands directly. This repo includes 6 skills in `.claude/skills/` that translate natural language into `ob` commands:

- **"Remember this: Sarah is thinking about consulting"** → `ob capture "Sarah — thinking about leaving job to start consulting business"`
- **"What do I know about the API redesign?"** → `ob search "API redesign decisions"`
- **"Weekly review"** → `ob recent 50` + `ob stats` + targeted searches + synthesis
- **"Is my brain working?"** → `ob check` + `ob stats`
- **"Import my Notion notes"** → structured chunking + batch `ob capture`
- **"Find duplicates"** → `ob search` with high threshold + `ob delete`

Skills load automatically when you open the repo in Claude Code.

## More Information

See [docs/CLI_DIRECT_APPROACH.md](../../docs/CLI_DIRECT_APPROACH.md) for full architecture context, CLI AI tool configuration, troubleshooting, and the comparison between MCP and CLI-Direct approaches.
