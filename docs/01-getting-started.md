# Open Brain CLI — Setup Guide

Get your Open Brain running in about 10 minutes. You'll set up a database, get an API key, install the `ob` CLI, and verify everything works.

## What You're Building

A database that stores your thoughts with vector embeddings, plus a CLI tool that lets you capture and search from your terminal. Your AI tools (Claude Code, Codex, Gemini CLI) can use `ob` directly — no MCP server needed.

```
You → ob capture "thought" → OpenRouter (embedding + metadata) → Supabase PostgreSQL
You → ob search "query"    → OpenRouter (query embedding)      → pgvector similarity search
```

## What You Need

- About 10 minutes
- A terminal with `curl` and `jq` installed
- **Windows:** Git Bash, WSL, or similar bash environment

### Services (All Free Tier)

| Service | What It Does | Cost |
|---------|-------------|------|
| [Supabase](https://supabase.com) | Your database — stores everything | $0 (free tier) |
| [OpenRouter](https://openrouter.ai) | AI gateway — generates embeddings and extracts metadata | ~$0.10–0.30/month |

### If You Get Stuck

Supabase has a free built-in AI assistant in every project dashboard (chat icon, bottom-right). It knows all of Supabase's documentation and can help with database setup, SQL errors, and configuration issues.

---

## Credential Tracker

You'll generate keys across two services. Copy this into a text editor and fill it in as you go:

```text
OPEN BRAIN CLI -- CREDENTIAL TRACKER
--------------------------------------

SUPABASE
  Database password:  ____________ <- Step 1
  Project ref:        ____________ <- Step 1
  Project URL:        ____________ <- Step 2
  Secret key:         ____________ <- Step 2

OPENROUTER
  API key:            ____________ <- Step 3

--------------------------------------
```

---

## Step 1: Create Your Supabase Project

Supabase is your database. It stores your thoughts as raw text, vector embeddings, and structured metadata — and gives you a REST API automatically.

1. Go to [supabase.com](https://supabase.com) and sign up (GitHub login is fastest)
2. Click **New Project**
3. Pick your organization (default is fine)
4. Set Project name: `open-brain` (or whatever you want)
5. Generate a strong Database password — paste into your credential tracker NOW
6. Pick the Region closest to you
7. Click **Create new project** and wait 1–2 minutes

> Grab your **Project ref** — it's the random string in your dashboard URL: `supabase.com/dashboard/project/THIS_PART`. Paste it into the tracker.

### Set Up the Database

Three SQL commands, pasted one at a time into the Supabase SQL Editor.

**Enable the vector extension:** In the left sidebar: **Database → Extensions** → search for "vector" → flip **pgvector ON**.

**Create the thoughts table:** In the left sidebar: **SQL Editor → New query** → paste and Run:

```sql
-- Create the thoughts table
create table thoughts (
  id uuid default gen_random_uuid() primary key,
  content text not null,
  embedding vector(1536),
  metadata jsonb default '{}'::jsonb,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Index for fast vector similarity search
create index on thoughts
  using hnsw (embedding vector_cosine_ops);

-- Index for filtering by metadata fields
create index on thoughts using gin (metadata);

-- Index for date range queries
create index on thoughts (created_at desc);

-- Auto-update the updated_at timestamp
create or replace function update_updated_at()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger thoughts_updated_at
  before update on thoughts
  for each row
  execute function update_updated_at();
```

**Create the search function:** New query → paste and Run:

```sql
-- Semantic search function
create or replace function match_thoughts(
  query_embedding vector(1536),
  match_threshold float default 0.7,
  match_count int default 10,
  filter jsonb default '{}'::jsonb
)
returns table (
  id uuid,
  content text,
  metadata jsonb,
  similarity float,
  created_at timestamptz
)
language plpgsql
as $$
begin
  return query
  select
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) as similarity,
    t.created_at
  from thoughts t
  where 1 - (t.embedding <=> query_embedding) > match_threshold
    and (filter = '{}'::jsonb or t.metadata @> filter)
  order by t.embedding <=> query_embedding
  limit match_count;
end;
$$;
```

**Lock down security:** One more new query:

```sql
-- Enable Row Level Security
alter table thoughts enable row level security;

-- Service role full access only
create policy "Service role full access"
  on thoughts
  for all
  using (auth.role() = 'service_role');
```

**Verify:** Table Editor should show the `thoughts` table with columns: id, content, embedding, metadata, created_at, updated_at. Database → Functions should show `match_thoughts`.

---

## Step 2: Save Your Connection Details

In the left sidebar: **Settings** (gear icon) → **API**. Copy these into your credential tracker:

- **Project URL** — Listed under "Project URL"
- **Secret key** — Under "API keys," click reveal on the key formerly labeled "Service role key." Same key, new name.

> Treat the Secret key like a password. Anyone with it has full access to your data.

---

## Step 3: Get an OpenRouter API Key

OpenRouter is a universal AI gateway — one key for embeddings and metadata extraction.

1. Go to [openrouter.ai](https://openrouter.ai) and sign up
2. Go to [openrouter.ai/keys](https://openrouter.ai/keys)
3. Click **Create Key**, name it `open-brain`
4. Copy the key into your credential tracker
5. Add $5 in credits under Credits (lasts months)

---

## Step 4: Install the `ob` CLI

```bash
# Clone the repo
git clone https://github.com/az9713/open-brain-cli.git

# Copy to your PATH
mkdir -p ~/.local/bin
cp open-brain-cli/resources/ob-cli/ob ~/.local/bin/ob
chmod +x ~/.local/bin/ob

# Add to PATH if needed (add this line to ~/.bashrc or ~/.zshrc)
export PATH="$HOME/.local/bin:$PATH"
```

### Set Environment Variables

Add to your `~/.bashrc`, `~/.zshrc`, or `~/.profile`:

```bash
export OB_SUPABASE_URL="your-project-url-from-step-2"
export OB_SUPABASE_KEY="your-secret-key-from-step-2"
export OB_OPENROUTER_KEY="your-openrouter-key-from-step-3"
```

Reload your shell:

```bash
source ~/.bashrc   # or ~/.zshrc
```

---

## Step 5: Verify

```bash
# Check connectivity
ob check

# Capture your first thought
ob capture "Sarah mentioned she's thinking about leaving her job to start a consulting business"

# Search by meaning (not keywords)
ob search "career changes"

# View recent thoughts
ob recent

# Check stats
ob stats
```

Open your Supabase dashboard → Table Editor → `thoughts`. You should see your captured thought with its embedding and extracted metadata.

**You're done.** Your Open Brain is live.

---

## Commands Reference

| Command | Description |
|---|---|
| `ob capture "text"` | Save a thought with embedding + metadata |
| `ob search "query" [--threshold N] [--count N] [--json]` | Semantic search |
| `ob recent [count]` | List recent thoughts (default: 10) |
| `ob delete <thought-id>` | Delete a thought by ID |
| `ob stats` | Knowledge base statistics |
| `ob check` | Verify connectivity |
| `ob version` | Version and config status |

See the full [CLI Reference](../resources/ob-cli/README.md) for details on flags, exit codes, and environment variables.

---

## What to Do Next

- **[Quick Start Guide](QUICKSTART.md)** — 10 quick wins to build the capture habit
- **[Companion Prompts](02-companion-prompts.md)** — Memory migration, use case discovery, weekly review
- **[Extensions](../extensions/)** — Add household knowledge, meal planning, CRM, and more
- **[User Guide](USER_GUIDE.md)** — Full guide with use cases and tips

---

## Troubleshooting

**`ob check` fails on Supabase**

Double-check your `OB_SUPABASE_URL` and `OB_SUPABASE_KEY`. The URL should be `https://your-ref.supabase.co` (no trailing slash). The key is the Secret/service role key, not the anon key.

**`ob capture` fails with "embedding failed"**

Your OpenRouter key may be invalid or out of credits. Check at [openrouter.ai/keys](https://openrouter.ai/keys). Run `ob check` to test connectivity.

**`ob search` returns no results**

Make sure you've captured at least one thought first. Try lowering the threshold: `ob search "your query" --threshold 0.3`. Semantic search gets more accurate as you add more thoughts.

**Metadata is wrong or incomplete**

Metadata extraction is best-effort — the LLM guesses based on your text. The embedding (which powers search) works regardless of metadata quality. For better metadata, be specific when capturing: include names, dates, and action items explicitly.

**Windows: "command not found" or "permission denied"**

The `ob` script requires a bash environment. Use Git Bash, WSL, or similar. Make sure the script is executable: `chmod +x ~/.local/bin/ob`.

---

## Optional: MCP Server for GUI Clients

If you also use GUI-based AI tools (Claude Desktop, ChatGPT, Cursor), you can deploy an MCP server alongside the CLI. Both paths read and write the same `thoughts` table.

MCP setup requires deploying a Supabase Edge Function and takes an additional 20–30 minutes. See the [original Open Brain guide](https://github.com/NateBJones-Projects/OB1) for MCP deployment instructions.

---

*Originally created by [Nate B. Jones](https://natesnewsletter.substack.com/). CLI edition maintained at [github.com/az9713/open-brain-cli](https://github.com/az9713/open-brain-cli).*
