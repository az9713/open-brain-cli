# ob CLI

A lightweight command-line interface for Open Brain personal memory. Uses `curl` and `jq` to talk directly to Supabase and OpenRouter — no MCP, no Edge Function, no runtime dependencies.

## Prerequisites

- `curl` and `jq` installed
- A Supabase project with the `thoughts` table, `match_thoughts()` function, and indexes (see `docs/01-getting-started.md` Steps 1–3)
- An OpenRouter API key

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
| `ob stats` | Knowledge base statistics |
| `ob version` | Version and config status |

## Verification

```bash
ob version   # Should show version and config status
ob stats     # Should show thought count (requires configured env vars)
```

## More Information

See [docs/CLI_DIRECT_APPROACH.md](../../docs/CLI_DIRECT_APPROACH.md) for full architecture context, CLI AI tool configuration, troubleshooting, and the comparison between MCP and CLI-Direct approaches.
