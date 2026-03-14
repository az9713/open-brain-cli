# How Open Brain Works: A Technical Deep Dive

This document explains exactly what Open Brain is, what every piece does, and how they
connect — from the PostgreSQL row to the AI client response. It assumes strong technical
fundamentals (you know databases, APIs, and how software systems fit together) but zero
prior exposure to the specific technologies Open Brain uses: Supabase, pgvector, MCP,
Edge Functions, or OpenRouter.

---

## Table of Contents

1. [The One-Sentence Summary](#the-one-sentence-summary)
2. [The Problem Open Brain Solves](#the-problem-open-brain-solves)
3. [System Architecture Overview](#system-architecture-overview)
4. [Layer 1: PostgreSQL + pgvector — The Storage Engine](#layer-1-postgresql--pgvector--the-storage-engine)
5. [Layer 2: Supabase — The Platform Layer](#layer-2-supabase--the-platform-layer)
6. [Layer 3: OpenRouter — The AI Gateway](#layer-3-openrouter--the-ai-gateway)
7. [Layer 4: MCP — The Protocol Layer](#layer-4-mcp--the-protocol-layer)
8. [Layer 5: AI Clients — The User Interface](#layer-5-ai-clients--the-user-interface)
9. [The Capture Pipeline in Detail](#the-capture-pipeline-in-detail)
10. [The Search Pipeline in Detail](#the-search-pipeline-in-detail)
11. [Extension Architecture](#extension-architecture)
12. [Integration Architecture (Slack/Discord)](#integration-architecture-slackdiscord)
13. [Security Model](#security-model)
14. [What Runs Where](#what-runs-where)
15. [Cost Model](#cost-model)
16. [Common Misconceptions](#common-misconceptions)

---

## The One-Sentence Summary

Open Brain is a PostgreSQL database with vector embeddings, exposed to AI clients
through a serverless MCP server, so that every AI tool you use shares one persistent
memory.

## The Problem Open Brain Solves

Every AI conversation starts from zero. Claude does not know what you told ChatGPT
yesterday. ChatGPT does not know what you discussed with Cursor this morning. Each
tool has its own ephemeral context window that evaporates when the session ends.

Open Brain solves this by putting a database between you and your AI tools. The
database stores your thoughts as text + vector embedding + structured metadata. An
MCP server sits in front of the database and exposes four tools (capture, search,
browse, stats) over a standard protocol. Any AI client that speaks MCP can read from
and write to the same database. One brain, many clients.

## System Architecture Overview

```
+--------------------------------------------------------------------+
|                        AI CLIENTS (Layer 5)                        |
|  Claude Desktop  |  ChatGPT  |  Claude Code  |  Cursor  |  etc.  |
+--------+---------+-----+-----+-------+-------+----+-----+---------+
         |               |             |             |
         |   MCP Protocol (HTTP or stdio)            |
         |               |             |             |
+--------v---------------v-------------v-------------v---------+
|                     MCP SERVERS (Layer 4)                     |
|                                                               |
|  +---------------------------+  +---------------------------+ |
|  | Core: open-brain-mcp      |  | Extension: household-     | |
|  | (Supabase Edge Function)  |  | knowledge (Node.js stdio) | |
|  |                           |  |                           | |
|  | Tools:                    |  | Tools:                    | |
|  |  - capture_thought        |  |  - add_household_item     | |
|  |  - search_thoughts        |  |  - search_household_items | |
|  |  - list_thoughts          |  |  - get_item_details       | |
|  |  - stats                  |  |  - add_vendor             | |
|  +------------+--------------+  |  - list_vendors           | |
|               |                 +------------+--------------+ |
+---------------|-----------------------------|--+--------------+
                |                             |  |
    +-----------v----+              +---------v--v-------+
    | OpenRouter API |              |                    |
    | (Layer 3)      |              |  Supabase          |
    |                |              |  (Layer 2)         |
    | Embedding:     +------+       |                    |
    |  text-embedding|      |       |  +----------------+|
    |  -3-small      |      +------->  | PostgreSQL     ||
    |                |              |  | + pgvector     ||
    | Metadata:      |              |  | (Layer 1)      ||
    |  gpt-4o-mini   |              |  |                ||
    +----------------+              |  | Tables:        ||
                                    |  |  thoughts      ||
                                    |  |  household_*   ||
                                    |  |  maintenance_* ||
                                    |  |  family_*      ||
                                    |  |  recipes, ...  ||
                                    |  +----------------+|
                                    |                    |
                                    |  Edge Functions    |
                                    |  (Deno runtime)    |
                                    +--------------------+
```

Five layers, bottom to top:

| Layer | What | Technology | Where it runs |
|-------|------|-----------|---------------|
| 1 | Storage | PostgreSQL 14+ with pgvector | Supabase cloud |
| 2 | Platform | Supabase (auth, REST API, Edge Functions) | Supabase cloud |
| 3 | AI Services | OpenRouter (embeddings + LLM) | OpenRouter cloud |
| 4 | Protocol | MCP servers (HTTP Edge Function + stdio Node.js) | Supabase cloud + local |
| 5 | Interface | Claude Desktop, ChatGPT, Cursor, etc. | User's machine |

No piece depends on another vendor's proprietary system. You could replace Supabase
with raw PostgreSQL, replace OpenRouter with direct OpenAI calls, and replace Claude
with any MCP-compatible client. The architecture is deliberately vendor-agnostic.

---

## Layer 1: PostgreSQL + pgvector — The Storage Engine

### What pgvector is

pgvector is a PostgreSQL extension that adds a `VECTOR` column type and distance
operators. It lets you store arrays of floating-point numbers (vectors) as first-class
column values and perform similarity search on them using SQL.

If you have used PostGIS for geospatial queries, pgvector is the same idea applied to
high-dimensional vector space instead of 2D/3D coordinates.

### The thoughts table

This is the only table the core system requires:

```sql
CREATE TABLE thoughts (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  content     TEXT NOT NULL,                    -- the raw text
  embedding   VECTOR(1536),                     -- 1536-float semantic vector
  metadata    JSONB DEFAULT '{}'::jsonb,         -- structured extraction
  created_at  TIMESTAMPTZ DEFAULT now(),
  updated_at  TIMESTAMPTZ DEFAULT now()
);
```

Each row is one thought. The `content` column is what the user typed. The `embedding`
column is a 1536-dimensional vector produced by the `text-embedding-3-small` model —
it is a numerical encoding of the semantic meaning of the text. The `metadata` column
is a JSONB object produced by `gpt-4o-mini` that contains structured fields like
topics, people mentioned, action items, and a type classification.

> **Source code:** The `CREATE TABLE thoughts` DDL is in
> `docs/01-getting-started.md:124-157` (Step 2, "Create the Thoughts Table").
> You paste this SQL directly into the Supabase SQL Editor.

### Three indexes and why each matters

```sql
-- 1. HNSW index for approximate nearest neighbor search on embeddings
CREATE INDEX ON thoughts USING hnsw (embedding vector_cosine_ops);

-- 2. GIN index for JSONB containment queries on metadata
CREATE INDEX ON thoughts USING gin (metadata);

-- 3. B-tree index for date-ordered browsing
CREATE INDEX ON thoughts (created_at DESC);
```

**HNSW (Hierarchical Navigable Small World)** is an approximate nearest neighbor (ANN)
algorithm. Without it, finding the closest vectors to a query requires comparing against
every row (O(n)). HNSW builds a multi-layer graph structure that allows O(log n) lookup.
The tradeoff: slightly imperfect recall (it may miss the absolute nearest neighbor in
rare cases) in exchange for dramatically faster queries. For Open Brain's use case
(personal knowledge, hundreds to tens of thousands of rows) the recall is effectively
perfect.

`vector_cosine_ops` tells pgvector to use cosine distance as the comparison metric.
Cosine distance measures the angle between two vectors, ignoring magnitude. Two texts
about the same topic will have vectors pointing in similar directions regardless of
text length.

**GIN (Generalized Inverted Index)** accelerates JSONB containment queries. When you
filter by `metadata @> '{"type": "task"}'`, the GIN index makes this O(log n) instead
of a full table scan.

> **Source code:** All three index definitions are part of the same DDL block in
> `docs/01-getting-started.md:135-142`. The HNSW index is on line 136, the GIN
> index on line 139, and the B-tree index on line 142.

### The match_thoughts function

This is the semantic search engine:

```sql
CREATE OR REPLACE FUNCTION match_thoughts(
  query_embedding VECTOR(1536),
  match_threshold FLOAT DEFAULT 0.7,
  match_count     INT DEFAULT 10,
  filter          JSONB DEFAULT '{}'::jsonb
)
RETURNS TABLE (
  id          UUID,
  content     TEXT,
  metadata    JSONB,
  similarity  FLOAT,
  created_at  TIMESTAMPTZ
)
LANGUAGE plpgsql
AS $$
BEGIN
  RETURN QUERY
  SELECT
    t.id,
    t.content,
    t.metadata,
    1 - (t.embedding <=> query_embedding) AS similarity,
    t.created_at
  FROM thoughts t
  WHERE 1 - (t.embedding <=> query_embedding) > match_threshold
    AND (filter = '{}'::jsonb OR t.metadata @> filter)
  ORDER BY t.embedding <=> query_embedding
  LIMIT match_count;
END;
$$;
```

Key details:

- `<=>` is pgvector's cosine distance operator. It returns a value between 0 (identical)
  and 2 (opposite). `1 - distance` converts this to a similarity score between -1 and 1,
  where 1 means identical.

- `match_threshold` defaults to 0.7 — only results with >70% cosine similarity are
  returned. This filters out weak matches.

- `filter` is an optional JSONB argument. If provided, results must also satisfy a
  JSONB containment check (`@>`) against the metadata column. This enables hybrid search:
  vector similarity narrowed by structured filters (e.g., "find similar thoughts that
  are also tagged as tasks").

- The `ORDER BY t.embedding <=> query_embedding` clause uses the HNSW index for
  efficient retrieval.

> **Source code:** The `match_thoughts()` function is defined in
> `docs/01-getting-started.md:164-194` (Step 2, "Create the Search Function").
> This is pasted as a separate SQL query in the Supabase SQL Editor after the
> table creation.

### Row Level Security

RLS is enabled on the thoughts table with a single policy:

```sql
ALTER TABLE thoughts ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Service role full access"
  ON thoughts FOR ALL
  USING (auth.role() = 'service_role');
```

This means: only connections using the `service_role` key can read or write. The
anonymous key and authenticated user keys see zero rows. The MCP server uses the
service role key, so it has full access. Nobody else can query the table directly
through the Supabase REST API.

> **Source code:** The RLS policy for the core thoughts table is in
> `docs/01-getting-started.md:201-209` (Step 2, "Lock Down Security").
> Compare with the extension-level RLS policies — for example, the user-scoped
> policies in `extensions/household-knowledge/schema.sql:42-56` and the
> household-scoped policies in `extensions/meal-planning/schema.sql:54-113`.
> The RLS primitive guide at `primitives/rls/README.md` covers all three
> patterns with full worked examples.

---

## Layer 2: Supabase — The Platform Layer

### What Supabase actually is

Supabase is a managed PostgreSQL service with batteries included. When you create a
Supabase project, you get:

- A PostgreSQL 14+ instance with pgvector pre-installed
- An auto-generated REST API (PostgREST) that maps every table to HTTP endpoints
- An auth system (JWT-based, supports email/password, OAuth, magic link)
- Edge Functions (Deno-based serverless compute)
- A dashboard with a SQL editor, table editor, and log viewer
- Two API keys: an `anon` key (respects RLS) and a `service_role` key (bypasses RLS)

If you are familiar with Firebase, Supabase is the PostgreSQL equivalent. If you are
familiar with Heroku Postgres, Supabase adds the application layer on top.

### Edge Functions

Edge Functions are the serverless compute layer. They are Deno programs (TypeScript
running on the Deno runtime) that receive HTTP requests and return HTTP responses.
They are deployed to Supabase's infrastructure and run close to the database.

The core Open Brain MCP server is an Edge Function. It receives MCP protocol messages
over HTTP, processes them, and returns results. It has automatic access to two
environment variables:

- `SUPABASE_URL` — the project's REST API base URL
- `SUPABASE_SERVICE_ROLE_KEY` — the full-access key

Additional secrets (like `OPENROUTER_API_KEY` and `MCP_ACCESS_KEY`) are set via the
Supabase CLI:

```bash
supabase secrets set OPENROUTER_API_KEY=sk-or-...
supabase secrets set MCP_ACCESS_KEY=a3f8b2c1d4e5...
```

Deployment is a single command:

```bash
supabase functions deploy open-brain-mcp --no-verify-jwt
```

`--no-verify-jwt` disables the default JWT requirement on the Edge Function endpoint,
because the MCP server handles its own authentication via the access key.

### The Supabase client library

Extension MCP servers (the ones that run locally, not as Edge Functions) use the
`@supabase/supabase-js` client library to communicate with the database:

```typescript
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY,
  {
    auth: {
      autoRefreshToken: false,
      persistSession: false,
    },
  }
);
```

This creates an HTTP client that sends requests to the PostgREST API. The
`service_role` key bypasses RLS, giving full CRUD access. The query builder provides a
fluent API:

```typescript
// SELECT * FROM household_items WHERE user_id = ? AND category ILIKE '%paint%'
const { data, error } = await supabase
  .from("household_items")
  .select("*")
  .eq("user_id", userId)
  .ilike("category", "%paint%")
  .order("created_at", { ascending: false });
```

Under the hood, this translates to HTTP:

```
GET /rest/v1/household_items?user_id=eq.{userId}&category=ilike.%25paint%25&order=created_at.desc
Authorization: Bearer {service_role_key}
apikey: {service_role_key}
```

> **Source code:** The Supabase client initialization is at
> `extensions/household-knowledge/index.ts:30-39`. Every extension server
> follows this same pattern — compare with `extensions/job-hunt/index.ts` or
> `extensions/meal-planning/index.ts` to see identical initialization blocks.
> The query builder fluent API is used in every handler function; for example,
> the `search_household_items` handler at
> `extensions/household-knowledge/index.ts:244-277` shows `.eq()`, `.ilike()`,
> `.or()`, and `.order()` chained together. The `handleAddVendor` function at
> `extensions/household-knowledge/index.ts:303-331` demonstrates `.insert()`
> with `.select().single()`.

---

## Layer 3: OpenRouter — The AI Gateway

### What OpenRouter is

OpenRouter is a unified API gateway for AI models. One API key gives you access to
models from OpenAI, Anthropic, Google, Meta, Mistral, and others. The request format
mirrors the OpenAI API specification, so switching models is a one-line change.

Open Brain uses OpenRouter for two operations:

### Operation 1: Embedding generation

```typescript
async function getEmbedding(text: string): Promise<number[]> {
  const response = await fetch("https://openrouter.ai/api/v1/embeddings", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/text-embedding-3-small",
      input: text,
    }),
  });
  const data = await response.json();
  return data.data[0].embedding; // number[] with 1536 elements
}
```

`text-embedding-3-small` is OpenAI's embedding model. It takes a string and returns a
1536-dimensional vector of floats. The vector encodes semantic meaning: texts about
similar topics produce vectors that are close together in cosine distance, even if
they share no keywords.

**Why this matters:** "Sarah is thinking about leaving her job to start consulting"
and "What did I capture about career changes?" share zero keywords. But their
embedding vectors are close in cosine distance because the underlying meanings
overlap. This is what makes semantic search work.

### Operation 2: Metadata extraction

```typescript
async function extractMetadata(text: string): Promise<Record<string, unknown>> {
  const response = await fetch("https://openrouter.ai/api/v1/chat/completions", {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${OPENROUTER_API_KEY}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: "openai/gpt-4o-mini",
      response_format: { type: "json_object" },
      messages: [
        {
          role: "system",
          content: `Extract metadata from the user's captured thought. Return JSON with:
- "people": array of people mentioned
- "action_items": array of implied to-dos
- "topics": array of 1-3 short topic tags
- "type": one of "observation", "task", "idea", "reference", "person_note"
Only extract what's explicitly there.`
        },
        { role: "user", content: text },
      ],
    }),
  });
  const data = await response.json();
  return JSON.parse(data.choices[0].message.content);
}
```

`gpt-4o-mini` is a small, fast, cheap LLM. It reads the user's text and extracts
structured metadata: who was mentioned, what topics are covered, whether there are
action items, and what type of thought this is. The `response_format: { type:
"json_object" }` parameter forces the model to return valid JSON.

This metadata serves two purposes:
1. It enables filtered search (e.g., "show me all my person notes about Sarah")
2. It provides the AI client with structured context alongside search results

**Cost:** Embedding costs ~$0.02 per million tokens. Metadata extraction costs ~$0.15
per million input tokens. For 20 thoughts/day, the total is roughly $0.10-0.30/month.

> **Source code:** Both `getEmbedding()` and `extractMetadata()` are implemented
> in the Slack capture Edge Function at
> `integrations/slack-capture/README.md:125-157`. The system prompt for metadata
> extraction (lines 143-149) defines the exact JSON schema the LLM must return.
> The same two functions are used in the core MCP Edge Function — the setup guide
> at `docs/01-getting-started.md:341` references the full server code. The
> `Promise.all` parallelization is at
> `integrations/slack-capture/README.md:185-188`.

---

## Layer 4: MCP — The Protocol Layer

### What MCP is — starting from scratch

If you have never heard of MCP, start here. If you have built REST APIs, gRPC
services, or even shell scripts that other programs call, you already understand the
core idea. MCP is just a standardized way for an AI chatbot to call functions that
live outside of it.

**The problem MCP solves:** Claude, ChatGPT, and other AI models are text-in,
text-out systems. They cannot query a database, call an API, read a file, or do
anything in the real world on their own. They can only generate text. To do useful
things, they need a way to say "I want to call this function with these arguments"
and get the result back. MCP is the protocol that standardizes how that works.

**Before MCP existed:** Every AI platform had its own proprietary way of doing this.
OpenAI called it "function calling." Anthropic called it "tool use." Each had a
different message format, a different way to define tool schemas, and a different way
to return results. If you wanted your database to be accessible from both Claude and
ChatGPT, you had to write two separate integrations.

**What MCP standardizes:** One protocol, one message format, one way to define tools.
Any AI client that speaks MCP can connect to any MCP server. You write the server
once, and Claude Desktop, ChatGPT, Claude Code, Cursor, and every future MCP client
can all use it.

### The analogy that makes it click

Think of a restaurant:

```
+------------------+     +------------------+     +------------------+
|                  |     |                  |     |                  |
|    CUSTOMER      |     |    WAITER        |     |    KITCHEN       |
|    (You)         |     |    (AI Client)   |     |    (MCP Server)  |
|                  |     |                  |     |                  |
|  "I'd like the   |---->|  Reads the menu  |---->|  Receives the    |
|   pasta, please" |     |  (ListTools),    |     |  order (CallTool)|
|                  |     |  takes your      |     |  cooks the dish, |
|                  |<----|  order, brings   |<----|  sends it out    |
|                  |     |  your food       |     |                  |
+------------------+     +------------------+     +------------------+
```

- **You** speak natural language. You do not need to know what is in the kitchen.
- **The waiter** (AI client) understands you AND knows the menu (available tools).
  The waiter decides which dish to order based on what you asked for.
- **The kitchen** (MCP server) has a fixed menu of dishes it can make (tools). It
  does not decide what to cook — it waits for orders and fulfills them.

In Open Brain:
- **You** say "What did I capture about career changes?"
- **Claude** (the waiter) reads the menu of MCP tools, sees `search_thoughts`, and
  decides to order it with `{query: "career changes"}`
- **The MCP server** (the kitchen) takes that order, queries the database, and sends
  back the results
- **Claude** reads the results and tells you what it found, in plain English

You never interact with the MCP server. You never see the JSON messages. You just
talk to Claude (or ChatGPT, or Cursor) and it handles the rest.

### The actual protocol — what is on the wire

MCP uses JSON-RPC 2.0 — a simple request/response format over either HTTP or
stdin/stdout. There are only two operations that matter:

**Operation 1: "What tools do you have?" (ListTools)**

When an AI client first connects to an MCP server, it asks what tools are available:

```json
// Request (AI client → MCP server)
{"jsonrpc": "2.0", "method": "tools/list", "id": 1}

// Response (MCP server → AI client)
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "search_thoughts",
        "description": "Search your stored thoughts by semantic similarity",
        "inputSchema": {
          "type": "object",
          "properties": {
            "query": {"type": "string", "description": "What to search for"},
            "threshold": {"type": "number", "description": "Minimum similarity (0-1)"}
          },
          "required": ["query"]
        }
      },
      {
        "name": "capture_thought",
        "description": "Save a new thought to your Open Brain",
        "inputSchema": {
          "type": "object",
          "properties": {
            "content": {"type": "string", "description": "The thought to save"}
          },
          "required": ["content"]
        }
      }
    ]
  }
}
```

The AI client reads these tool definitions and now knows: "I have a `search_thoughts`
tool that takes a `query` string, and a `capture_thought` tool that takes a `content`
string." It stores this in its context and uses it to decide when to call tools during
the conversation.

**The `description` field is critical.** This is what the AI reads to decide whether
to use a tool. If the description says "Search your stored thoughts by semantic
similarity" and the user asks "What did I note about career changes?", the AI matches
the intent to the description and decides to call `search_thoughts`. A vague or
misleading description means the AI will not call the tool at the right time.

**Operation 2: "Call this tool" (CallTool)**

When the AI decides to use a tool, it sends a call request:

```json
// Request (AI client → MCP server)
{
  "jsonrpc": "2.0",
  "method": "tools/call",
  "id": 2,
  "params": {
    "name": "search_thoughts",
    "arguments": {
      "query": "career changes",
      "threshold": 0.7
    }
  }
}

// Response (MCP server → AI client)
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": "{\"results\": [{\"content\": \"Sarah mentioned she wants to start consulting\", \"similarity\": 0.84}]}"
      }
    ]
  }
}
```

The AI receives the result as a JSON string inside a `content` array. It reads this,
interprets the results, and formulates a natural language response for you.

### The complete conversation flow

Here is what happens from your perspective versus what happens on the wire:

```
YOU                          CLAUDE (AI Client)                 MCP SERVER
 |                               |                                |
 | "What did I capture           |                                |
 |  about career changes?"       |                                |
 |------------------------------>|                                |
 |                               |                                |
 |                               | [Claude thinks: "The user      |
 |                               |  wants to search their stored  |
 |                               |  thoughts. I have a tool       |
 |                               |  called search_thoughts that   |
 |                               |  does semantic search. I'll    |
 |                               |  call it with query='career    |
 |                               |  changes'."]                   |
 |                               |                                |
 |                               |--- CallTool ------------------>|
 |                               |    name: "search_thoughts"     |
 |                               |    args: {query: "career       |
 |                               |           changes"}            |
 |                               |                                |
 |                               |                    [MCP server |
 |                               |                     generates  |
 |                               |                     embedding, |
 |                               |                     queries    |
 |                               |                     PostgreSQL,|
 |                               |                     returns    |
 |                               |                     results]   |
 |                               |                                |
 |                               |<-- CallTool response ----------|
 |                               |    [{content: "Sarah mentioned |
 |                               |      she wants to start        |
 |                               |      consulting",              |
 |                               |      similarity: 0.84}]        |
 |                               |                                |
 |                               | [Claude reads results and      |
 |                               |  formulates response]          |
 |                               |                                |
 | "I found a thought from last  |                                |
 |  week: Sarah mentioned she's  |                                |
 |  thinking about leaving her   |                                |
 |  job to start consulting.     |                                |
 |  Similarity: 84%."            |                                |
 |<------------------------------|                                |
```

**The key insight:** The AI client (Claude) is the decision-maker. It reads your
natural language, decides which tool to call, constructs the arguments, and
interprets the results. The MCP server is passive — it is a function library that
waits to be called. It never initiates communication. It never decides what to do.
It just executes what it is asked and returns the result.

### Two transport mechanisms — how messages physically travel

The JSON-RPC messages above need a way to get from the AI client to the MCP server.
MCP supports two transport mechanisms:

```
Transport 1: HTTP (used by the core MCP server)
+------------------+          HTTPS           +------------------+
|                  |  POST /functions/v1/     |                  |
|  Claude Desktop  |  open-brain-mcp          |  Supabase Edge   |
|  (your laptop)   |------------------------>|  Function         |
|                  |                          |  (cloud)          |
|                  |<------------------------|                  |
|                  |     JSON response        |                  |
+------------------+                          +------------------+

Transport 2: stdio (used by extension MCP servers)
+------------------+     stdin/stdout pipe    +------------------+
|                  |                          |                  |
|  Claude Desktop  |  {"jsonrpc":"2.0",...}   |  Node.js process |
|  (your laptop)   |  written to stdin  ---->|  (your laptop)   |
|                  |                          |                  |
|                  |  {"jsonrpc":"2.0",...}   |                  |
|                  |  read from stdout <-----|                  |
+------------------+                          +------------------+
```

**HTTP transport:** The AI client sends an HTTP POST request to a URL. The body is a
JSON-RPC message. The response body is a JSON-RPC response. This is how the core
Open Brain MCP server works — it is a Supabase Edge Function with a public URL. Any
AI client on the internet can reach it (with the right access key).

**stdio transport:** The AI client spawns the MCP server as a child process on the
same machine. It writes JSON-RPC messages to the child's stdin. It reads JSON-RPC
responses from the child's stdout. No network involved — it is inter-process
communication via pipes. This is how all six extension MCP servers work — they are
Node.js programs that Claude Desktop launches locally.

**Why two transports?** The core server needs to be reachable from anywhere (your
laptop, your phone via ChatGPT, a CI system via Claude Code). HTTP makes it a
public endpoint. Extension servers only need to be reachable from the AI client on
your machine. stdio is simpler — no URL, no port, no firewall, no TLS certificates.

### What MCP servers exist in this codebase

Open Brain has **seven MCP servers** (one core + six extensions). Here is what each
one does, who it talks to, and where it runs:

```
+------------------------------------------------------------------+
|                     YOUR AI CLIENT                                |
|   (Claude Desktop, ChatGPT, Claude Code, Cursor)                 |
|                                                                   |
|   Connected to 1 or more MCP servers simultaneously:              |
+-------+-------+-------+-------+-------+-------+---------+--------+
        |       |       |       |       |       |         |
        v       v       v       v       v       v         v
   +--------+------+------+------+------+------+   +-----------+
   | Core   | Ext  | Ext  | Ext  | Ext  | Ext  |   | Ext 6:    |
   | MCP    | 1:   | 2:   | 3:   | 4:   | 5:   |   | Job Hunt  |
   | Server | House| Maint| Cal  | Meal | CRM  |   |           |
   +---+----+--+---+--+---+--+---+--+---+--+---+   +-----+-----+
       |       |      |      |      |      |              |
       | HTTP  | stdio| stdio| stdio| stdio| stdio        | stdio
       |       |      |      |      |      |              |
       v       v      v      v      v      v              v
   +------+ +--------------------------------------------+--------+
   |OpenR.| |                                                     |
   |API   | |              SUPABASE POSTGRESQL                    |
   +------+ |                                                     |
             |  thoughts | household_* | maintenance_* | family_* |
             |  recipes  | meal_plans  | contacts      | companies|
             |  ... (all tables in one database)                  |
             +----------------------------------------------------+
```

| MCP Server | Transport | Runs where | Tools it provides | Tables it queries | Source file |
|-----------|-----------|-----------|-------------------|------------------|------------|
| **Core: open-brain-mcp** | HTTP | Supabase cloud | `capture_thought`, `search_thoughts`, `list_thoughts`, `stats` | `thoughts` | `docs/01-getting-started.md:341` |
| **Ext 1: household-knowledge** | stdio | Your machine | `add_household_item`, `search_household_items`, `get_item_details`, `add_vendor`, `list_vendors` | `household_items`, `household_vendors` | `extensions/household-knowledge/index.ts` |
| **Ext 2: home-maintenance** | stdio | Your machine | Tasks + logs CRUD | `maintenance_tasks`, `maintenance_logs` | `extensions/home-maintenance/index.ts` |
| **Ext 3: family-calendar** | stdio | Your machine | Members + events CRUD, conflict detection | `family_members`, `family_events` | `extensions/family-calendar/index.ts` |
| **Ext 4: meal-planning** | stdio | Your machine | Recipes, meal plans, shopping lists | `recipes`, `meal_plans`, `shopping_lists` | `extensions/meal-planning/index.ts` |
| **Ext 5: professional-crm** | stdio | Your machine | Contacts, interactions, opportunities | `contacts`, `interactions`, `opportunities` | `extensions/professional-crm/index.ts` |
| **Ext 6: job-hunt** | stdio | Your machine | Companies, postings, applications, interviews, contacts | `companies`, `job_postings`, `applications`, `interviews`, `job_contacts` | `extensions/job-hunt/index.ts` |

**Every MCP server talks to the same PostgreSQL database** (your Supabase instance).
They are separate servers only because they expose different tool sets. The core
server gives the AI the ability to capture and search thoughts. Each extension server
gives the AI the ability to work with a specific domain (household facts, maintenance
schedules, meal plans, etc.).

**The AI client connects to multiple MCP servers simultaneously.** When you configure
Claude Desktop, you give it the core server URL AND the local paths to whichever
extension servers you have built. Claude sees all tools from all connected servers in
one unified list. When you ask "What's for dinner this week and do I have any
overdue maintenance?", Claude can call `search_meal_plans` from the meal-planning
server AND `list_overdue_tasks` from the maintenance server in the same conversation.

### Why MCP matters for Open Brain specifically

Without MCP, Open Brain would be just a database. You could query it with SQL. You
could build a REST API on top. But no AI client would know how to use it.

MCP is the layer that makes the database **AI-native**. It tells Claude: "You have a
tool called `capture_thought`. When the user says 'remember this' or 'save this note',
call it. You have a tool called `search_thoughts`. When the user asks about something
they noted before, call it." The AI reads these descriptions and acts accordingly —
without you writing prompt instructions, without you explaining how the database
works, without you knowing SQL.

This is the core value proposition: **the user talks in natural language, MCP
translates that into database operations, and the results come back as natural
language.** The database, the embeddings, the SQL, the HTTP calls — all invisible.

### The core MCP server

The core server is a Supabase Edge Function that exposes four tools:

| Tool | Purpose | Key input |
|------|---------|-----------|
| `capture_thought` | Store a new thought | `content` (string) |
| `search_thoughts` | Semantic search | `query` (string), `threshold`, `count`, `filter` |
| `list_thoughts` | Browse recent | `count`, `offset` |
| `stats` | Overview statistics | (none) |

The server authenticates requests using a custom access key, passed either as a URL
query parameter (`?key=...`) or as an HTTP header (`x-brain-key: ...`).

> **Source code:** The core MCP server setup is described in
> `docs/01-getting-started.md:269-365` (Step 6). The dependency configuration
> (`deno.json`) is at lines 328-337. The full server source code is referenced
> at line 341 — it links to the original guide. The access key generation is in
> Step 5 (`docs/01-getting-started.md:246-259`), and the secret is stored via
> `supabase secrets set MCP_ACCESS_KEY=...` at line 265. The two authentication
> methods (URL query parameter vs header) are documented in Step 7
> (`docs/01-getting-started.md:369-443`).

### Extension MCP servers

Each of the six extensions provides its own MCP server as a standalone Node.js
process using the stdio transport. Here is the anatomy of every extension server:

```typescript
// 1. Imports
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import { CallToolRequestSchema, ListToolsRequestSchema, Tool }
  from "@modelcontextprotocol/sdk/types.js";
import { createClient } from "@supabase/supabase-js";

// 2. Environment validation
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_SERVICE_ROLE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;
if (!SUPABASE_URL || !SUPABASE_SERVICE_ROLE_KEY) { process.exit(1); }

// 3. Supabase client
const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_ROLE_KEY, { ... });

// 4. Tool definitions (array of Tool objects)
const TOOLS: Tool[] = [
  {
    name: "add_household_item",
    description: "Add a new household item",
    inputSchema: { type: "object", properties: { ... }, required: ["name"] },
  },
  // ... more tools
];

// 5. Tool handler functions (async, query Supabase, return JSON string)
async function handleAddHouseholdItem(args: any): Promise<string> {
  const { data, error } = await supabase.from("household_items").insert({ ... });
  if (error) throw new Error(error.message);
  return JSON.stringify({ success: true, item: data });
}

// 6. Server instance
const server = new Server({ name: "household-knowledge", version: "1.0.0" },
                          { capabilities: { tools: {} } });

// 7. Request handler registration
server.setRequestHandler(ListToolsRequestSchema, async () => ({ tools: TOOLS }));
server.setRequestHandler(CallToolRequestSchema, async (request) => {
  switch (request.params.name) {
    case "add_household_item":
      return { content: [{ type: "text", text: await handleAddHouseholdItem(args) }] };
    // ... more cases
    default: throw new Error(`Unknown tool: ${request.params.name}`);
  }
});

// 8. Main
const transport = new StdioServerTransport();
await server.connect(transport);
```

This pattern is identical across all six extensions. The only things that change are
the tool definitions and handler implementations.

> **Source code:** The complete reference implementation is
> `extensions/household-knowledge/index.ts` (414 lines). Here is where each
> section maps to in that file:
>
> | Section | Lines | What to look at |
> |---------|-------|-----------------|
> | Imports | 11-18 | MCP SDK + Supabase imports |
> | Environment validation | 21-27 | `process.env` checks with early exit |
> | Supabase client | 30-39 | `createClient()` with auth options |
> | Type definitions | 42-66 | `HouseholdItem` and `HouseholdVendor` interfaces |
> | Tool definitions | 69-214 | `TOOLS: Tool[]` array with 5 tools, each with JSON Schema |
> | Handler: add item | 217-242 | `.insert()` → `.select().single()` |
> | Handler: search items | 244-277 | `.eq()`, `.ilike()`, `.or()` filter chain |
> | Handler: get details | 279-301 | `.eq("id")` + `.eq("user_id")` + `.single()` |
> | Handler: add vendor | 303-331 | Insert with 8 nullable fields |
> | Handler: list vendors | 333-356 | Optional filter + `.order("name")` |
> | Server setup | 358-369 | `new Server()` with capabilities |
> | Request handlers | 372-401 | `ListToolsRequestSchema` + `CallToolRequestSchema` |
> | Main function | 404-413 | `StdioServerTransport` + `server.connect()` |
>
> The other five extensions follow this exact structure. To compare:
> - `extensions/meal-planning/index.ts` — adds shared MCP server tools
> - `extensions/job-hunt/index.ts` — most complex, 5 tables, cross-extension FK
> - `extensions/professional-crm/index.ts` — demonstrates thought integration

---

## Layer 5: AI Clients — The User Interface

The user never interacts with the database or MCP server directly. They talk to an AI
client (Claude Desktop, ChatGPT, Claude Code, Cursor) in natural language. The AI
client does the translation:

1. User says: "What did I capture about career changes?"
2. AI reads available MCP tools and decides `search_thoughts` is appropriate
3. AI constructs: `{name: "search_thoughts", arguments: {query: "career changes"}}`
4. MCP server executes the search and returns ranked results
5. AI reads the results and formulates a natural language response

The user does not need to know that MCP exists, that embeddings are being generated,
or that cosine similarity is being computed. They just talk.

### Connection methods

| Client | Transport | Configuration |
|--------|-----------|--------------|
| Claude Desktop | HTTP | Settings → Connectors → paste MCP Connection URL |
| ChatGPT | HTTP | Settings → Apps & Connectors → Create → paste URL |
| Claude Code | HTTP | `claude mcp add --transport http open-brain URL --header "x-brain-key: KEY"` |
| Cursor/VS Code | stdio via mcp-remote | JSON config with `npx mcp-remote` bridge |

> **Source code:** All four connection methods are documented with exact
> configuration syntax in `docs/01-getting-started.md:369-443` (Step 7,
> "Connect to Your AI"). The `mcp-remote` JSON config with the `--header`
> no-space gotcha is at lines 424-441. Claude Desktop's connector setup
> is at lines 375-381. ChatGPT's Developer Mode requirement and connector
> setup are at lines 387-406.

---

## The Capture Pipeline in Detail

When a user says "Remember this: Sarah mentioned she wants to start consulting":

```
User types in AI client
        |
        v
AI client calls capture_thought MCP tool
  with {content: "Sarah mentioned she wants to start consulting"}
        |
        v
MCP Edge Function receives the request
        |
        v
+-------+-------+
|               |                    <-- Promise.all (parallel execution)
v               v
getEmbedding()  extractMetadata()
|               |
v               v
OpenRouter      OpenRouter
embedding API   chat API
|               |
v               v
[1536 floats]   {type: "person_note",
                 topics: ["career", "consulting"],
                 people: ["Sarah"],
                 action_items: ["Check in with Sarah"]}
|               |
+-------+-------+
        |
        v
supabase.from("thoughts").insert({
  content: "Sarah mentioned she wants to start consulting",
  embedding: [0.012, -0.034, 0.078, ...],  // 1536 values
  metadata: {
    type: "person_note",
    topics: ["career", "consulting"],
    people: ["Sarah"],
    action_items: ["Check in with Sarah"],
    source: "mcp"
  }
})
        |
        v
Confirmation returned to AI client
        |
        v
AI tells user: "Captured. Tagged as person_note about career/consulting.
                People: Sarah. Action item: Check in with Sarah."
```

**Critical detail:** The embedding and metadata extraction run in parallel via
`Promise.all([getEmbedding(text), extractMetadata(text)])`. This halves the
latency — both API calls take roughly 500-1500ms, but they execute concurrently
instead of sequentially.

> **Source code:** The `Promise.all` parallelization is implemented in the Slack
> capture Edge Function at `integrations/slack-capture/README.md:185-188`:
> ```typescript
> const [embedding, metadata] = await Promise.all([
>   getEmbedding(messageText),
>   extractMetadata(messageText),
> ]);
> ```
> The subsequent database insert is at lines 190-194. The same pattern is used
> in the core MCP server's `capture_thought` tool handler. The confirmation
> reply to the user (building the metadata display string) is at lines 202-211.

---

## The Search Pipeline in Detail

When a user asks "What did I capture about career changes?":

```
User asks question in AI client
        |
        v
AI client calls search_thoughts MCP tool
  with {query: "career changes", threshold: 0.7, count: 10}
        |
        v
MCP Edge Function receives the request
        |
        v
getEmbedding("career changes")
        |
        v
OpenRouter returns query embedding: [0.023, -0.045, 0.091, ...]
        |
        v
supabase.rpc("match_thoughts", {
  query_embedding: [0.023, -0.045, 0.091, ...],
  match_threshold: 0.7,
  match_count: 10,
  filter: {}
})
        |
        v
PostgreSQL executes:
  SELECT id, content, metadata,
         1 - (embedding <=> query_embedding) AS similarity
  FROM thoughts
  WHERE 1 - (embedding <=> query_embedding) > 0.7
  ORDER BY embedding <=> query_embedding
  LIMIT 10
        |
        v
Results (ranked by similarity):
  1. "Sarah mentioned she wants to start consulting"  (similarity: 0.84)
  2. "Marcus wants to move to the platform team"      (similarity: 0.76)
  3. "Thinking about whether to stay at current job"   (similarity: 0.73)
        |
        v
Results returned to AI client as JSON
        |
        v
AI formulates natural language response incorporating the results
```

**Why "career changes" finds "Sarah wants to start consulting":** Both phrases encode
the concept of professional transition. The embedding model maps them to nearby points
in 1536-dimensional space. The cosine similarity of 0.84 reflects strong semantic
overlap despite sharing zero keywords. This is the fundamental value proposition of
vector search over keyword search.

> **Source code:** The search flow uses `supabase.rpc("match_thoughts", ...)`
> which calls the PLpgSQL function defined in `docs/01-getting-started.md:164-194`.
> The key line is 205: `1 - (t.embedding <=> query_embedding) AS similarity` —
> this is where pgvector's cosine distance operator converts distance to
> similarity. The `WHERE` clause on line 208 applies the threshold filter, and
> line 209 applies the optional JSONB metadata filter.

---

## Extension Architecture

The repo contains six extensions that form a progressive learning path. Each extension
adds new tables to the same Supabase database and provides its own local MCP server
with domain-specific tools.

### Extension dependency graph

```
Extension 1: Household Knowledge (Beginner)
  Tables: household_items, household_vendors
  Concepts: Basic CRUD, JSONB, ILIKE search
  Primitives: none
        |
        v
Extension 2: Home Maintenance (Beginner)
  Tables: maintenance_tasks, maintenance_logs
  Concepts: Date scheduling, one-to-many FK
  Primitives: none
        |
        v
Extension 3: Family Calendar (Intermediate)
  Tables: family_members, family_events
  Concepts: Multi-person scheduling, conflict detection
  Primitives: none
        |
        v
Extension 4: Meal Planning (Intermediate)
  Tables: recipes, meal_plans, shopping_lists
  Concepts: JSONB arrays, household access, shared servers
  Primitives: rls, shared-mcp   <-- first use of RLS and shared MCP
        |
        v
Extension 5: Professional CRM (Intermediate)
  Tables: contacts, interactions, opportunities
  Concepts: Thought integration, relationship tracking
  Primitives: rls
        |
        v
Extension 6: Job Hunt Pipeline (Advanced)
  Tables: companies, job_postings, applications, interviews, job_contacts
  Concepts: 5-table schema, pipeline tracking, cross-extension FKs
  Primitives: rls
  Cross-ref: job_contacts.professional_crm_contact_id -> Extension 5
```

### What each extension contains

```
extensions/household-knowledge/
  ├── README.md           Setup guide and learning outcomes
  ├── metadata.json       Structured metadata for the contribution system
  ├── schema.sql          PostgreSQL DDL (tables, indexes, RLS, triggers)
  ├── index.ts            MCP server implementation (TypeScript)
  ├── package.json        Node.js dependencies
  └── tsconfig.json       TypeScript compiler config
```

Every extension follows this exact structure. The schema.sql creates tables in the
same Supabase database alongside the core `thoughts` table. The index.ts runs as a
separate local MCP server that the AI client connects to independently.

> **Source code:** To trace a complete extension, start with the simplest one:
>
> - **Schema:** `extensions/household-knowledge/schema.sql` — 78 lines. Two
>   tables (`household_items` at line 6, `household_vendors` at line 20),
>   two indexes (lines 35-39), RLS policies (lines 42-56), a trigger
>   function (lines 59-65), and a trigger (lines 68-72).
>
> - **Server:** `extensions/household-knowledge/index.ts` — 414 lines. See
>   the line-by-line table in the MCP section above.
>
> - **Config:** `extensions/household-knowledge/package.json` — lists
>   `@modelcontextprotocol/sdk` and `@supabase/supabase-js` as dependencies.
>   `extensions/household-knowledge/tsconfig.json` — targets ES2022 with
>   Node16 module resolution.
>
> - **Metadata:** `extensions/household-knowledge/metadata.json` — structured
>   fields validated by `.github/metadata.schema.json` (116-line JSON Schema).
>
> Then compare with the most complex extension:
>
> - **Schema:** `extensions/job-hunt/schema.sql` — 175 lines. Five tables
>   (`companies` at line 6, `job_postings` at line 23, `applications` at
>   line 43, `interviews` at line 60, `job_contacts` at line 78). Seven
>   indexes (lines 95-115). RLS on all five tables (lines 118-148). The
>   soft cross-extension FK is at line 88:
>   `professional_crm_contact_id UUID` (no `REFERENCES` constraint).
>
> - **Schema:** `extensions/meal-planning/schema.sql` — 113 lines. Three
>   tables with household-scoped RLS policies (lines 59-113) that use
>   JWT role checking: `auth.jwt() ->> 'role' = 'household_member'`.

### How extensions relate to the core

Extensions do NOT modify the core `thoughts` table. They add new tables that exist
alongside it. The AI client connects to both the core MCP server (for thoughts) and
extension MCP servers (for domain data). When the user asks a question, the AI can
query across all connected servers to find relevant information.

The cross-extension reference in Extension 6 (`job_contacts.professional_crm_contact_id`)
is a soft foreign key — it stores the UUID of a contact from Extension 5's table but
does not enforce it with a database constraint. This allows extensions to be built
independently while still enabling cross-references.

---

## Integration Architecture (Slack/Discord)

Integrations provide alternative capture paths. Instead of going through an AI client's
MCP connection, they capture data directly into the `thoughts` table from external
sources.

### Slack capture flow

```
User types in Slack #capture channel
        |
        v
Slack sends HTTP POST to Edge Function URL
  (Event Subscriptions webhook)
        |
        v
ingest-thought Edge Function
  1. Validates: is this a message? in the right channel? not from a bot?
  2. Promise.all([getEmbedding(text), extractMetadata(text)])
  3. INSERT INTO thoughts (content, embedding, metadata)
     with metadata.source = "slack"
  4. Reply in Slack thread with confirmation
        |
        v
Row in thoughts table, searchable by any MCP client
```

The Slack integration uses Slack's Event Subscriptions API. When a message is posted
to the configured channel, Slack sends an HTTP POST with the message content to the
Edge Function URL. The Edge Function processes it identically to the MCP capture tool —
embedding and metadata extraction in parallel, then insert.

The first request from Slack is a URL verification challenge (Slack sends
`{type: "url_verification", challenge: "..."}` and expects the challenge echoed back).
The Edge Function handles this automatically.

> **Source code:** The complete Slack capture Edge Function is in
> `integrations/slack-capture/README.md:113-217`. Key sections:
>
> | Lines | What it does |
> |-------|-------------|
> | 114-118 | Environment variable setup (`SUPABASE_URL`, `OPENROUTER_API_KEY`, `SLACK_BOT_TOKEN`, `SLACK_CAPTURE_CHANNEL`) |
> | 125-133 | `getEmbedding()` — calls OpenRouter embedding API |
> | 135-157 | `extractMetadata()` — calls OpenRouter chat API with JSON schema prompt |
> | 159-165 | `replyInSlack()` — posts threaded confirmation via Slack API |
> | 167-170 | URL verification challenge handler (returns `body.challenge`) |
> | 176-183 | Message validation (ignores bot messages, wrong channels, subtypes) |
> | 185-188 | `Promise.all` — parallel embedding + metadata extraction |
> | 190-194 | Database insert with `source: "slack"` in metadata |
> | 202-211 | Confirmation message construction (type, topics, people, action items) |
>
> The Slack app setup (OAuth scopes, Event Subscriptions, bot invitation) is
> documented in `integrations/slack-capture/README.md:57-77` (Step 2).
> The `message.channels` + `message.groups` gotcha (you need both for
> public and private channels) is at lines 253-256.
>
> The Discord capture integration follows a similar pattern:
> `integrations/discord-capture/README.md`.
>
> The shared MCP server primitive (for giving family members scoped access)
> is fully implemented in `primitives/shared-mcp/README.md:119-268`, with a
> complete TypeScript server, boundary test script, and deployment config.

---

## Security Model

### Defense in depth

```
+-------------------------------------------------------+
| Layer 1: Network — HTTPS everywhere                   |
|                                                       |
|  +---------------------------------------------------+|
|  | Layer 2: MCP Access Key                           ||
|  | (query param or header, checked on every request) ||
|  |                                                   ||
|  |  +-----------------------------------------------+||
|  |  | Layer 3: Supabase Service Role Key            |||
|  |  | (full database access, never exposed to user) |||
|  |  |                                               |||
|  |  |  +-----------------------------------------+  |||
|  |  |  | Layer 4: Row Level Security             |  |||
|  |  |  | (per-table, per-user access policies)   |  |||
|  |  |  +-----------------------------------------+  |||
|  |  +-----------------------------------------------+|||
|  +---------------------------------------------------+||
+-------------------------------------------------------+
```

1. **Network:** All communication is over HTTPS. Edge Functions are served over TLS.
   Supabase API endpoints require HTTPS.

2. **MCP Access Key:** A random 64-character hex string generated during setup. The
   core MCP server checks this on every request. Without it, the server returns 401.
   Passed either as `?key=...` in the URL or as `x-brain-key: ...` header.

3. **Service Role Key:** The Supabase service role key bypasses RLS and grants full
   database access. It is stored as a Supabase secret, never exposed to the client.
   The MCP server uses it internally to query the database.

4. **Row Level Security:** Extension tables use RLS policies to control per-user access.
   Three patterns are implemented:

   - **User-scoped:** `USING (auth.uid() = user_id)` — each user sees only their own
     rows. Used by Extensions 1, 2, 5, 6.

   - **Household-scoped:** Subquery checking `household_members` junction table — family
     members see shared data. Used by Extension 4 (Meal Planning).

   - **Public + Private:** `USING (visibility = 'public' OR auth.uid() = user_id)` —
     some rows are visible to all. Documented in `primitives/rls/`.

### CI/CD security enforcement

The GitHub Actions workflow (`.github/workflows/ob1-review.yml`) enforces security
rules on every PR:

- **Rule 4:** Scans code for credential patterns (API key formats, service role keys)
- **Rule 5:** Blocks `DROP TABLE`, `TRUNCATE`, unqualified `DELETE FROM`, and
  modifications to core `thoughts` table columns
- **Rule 8:** Rejects binary blobs over 1MB

These rules prevent accidental secret exposure and destructive SQL from reaching the
repository.

> **Source code:** The complete CI/CD workflow is at
> `.github/workflows/ob1-review.yml` (594 lines). The 13 automated checks are
> implemented as a single bash script in the `review` step (lines 46-553):
>
> | Rule | Lines | What it checks |
> |------|-------|---------------|
> | 1. Folder structure | 83-99 | Files in allowed directories |
> | 2. Required files | 101-119 | README.md + metadata.json exist |
> | 3. Metadata valid | 122-176 | JSON parses, required fields present |
> | 4. No credentials | 179-208 | Regex scan for API key patterns |
> | 5. SQL safety | 210-248 | No DROP/TRUNCATE, no core table mods |
> | 6. Category artifacts | 250-318 | Code/SQL files per category |
> | 7. PR format | 321-326 | Title starts with `[category]` |
> | 8. No binary blobs | 328-353 | File size < 1MB, no banned extensions |
> | 9. README completeness | 355-385 | Prerequisites, steps, expected outcome |
> | 10. Primitive deps | 387-422 | Declared primitives exist and are linked |
> | 11. LLM clarity | 424-428 | Stub (always passes, planned for v2) |
> | 12. Scope check | 432-457 | Changes within contribution folder only |
> | 13. Internal links | 459-488 | Relative links resolve to real files |
>
> The PR comment bot (lines 555-587) uses `actions/github-script` to post or
> update a review comment. The metadata JSON Schema is at
> `.github/metadata.schema.json` (116 lines).

---

## What Runs Where

| Component | Runtime | Location | Persistent? |
|-----------|---------|----------|-------------|
| PostgreSQL + pgvector | PostgreSQL 14+ | Supabase cloud | Yes |
| Core MCP server | Deno (Edge Function) | Supabase cloud | No (serverless) |
| Slack capture | Deno (Edge Function) | Supabase cloud | No (serverless) |
| Extension MCP servers | Node.js 18+ | User's local machine | No (process) |
| AI clients | Various | User's local machine | N/A |
| OpenRouter API | External service | OpenRouter cloud | N/A |

The only persistent state is in PostgreSQL. Everything else is stateless and
replaceable. If the Edge Function crashes, redeploying fixes it instantly. If a local
MCP server dies, restarting it loses nothing because all state is in the database.

---

## Cost Model

| Component | Cost |
|-----------|------|
| Supabase free tier | $0 (500MB database, 50K monthly active users) |
| OpenRouter embeddings (text-embedding-3-small) | ~$0.02 per million tokens |
| OpenRouter metadata extraction (gpt-4o-mini) | ~$0.15 per million input tokens |
| Vercel hosting (optional dashboard) | $0 (hobby tier) |

For a typical user capturing 20 thoughts per day: **$0.10-0.30/month** in API costs.

The system is designed to fit entirely within free tiers. The only variable cost is
the OpenRouter API usage for embeddings and metadata extraction, which is minimal.

---

## Common Misconceptions

### "Open Brain is an app I install"

No. Open Brain is a set of guides, SQL schemas, and server code. You build the system
yourself by creating a Supabase project, running SQL, deploying an Edge Function, and
connecting your AI client. You own every piece.

### "The MCP server is the brain"

No. The MCP server is a thin translation layer. It converts MCP protocol messages into
Supabase queries and OpenRouter API calls. The actual intelligence comes from:
(a) the embedding model that encodes meaning, (b) pgvector that performs similarity
search, and (c) the AI client that decides when to use the tools.

### "I need all six extensions"

No. The extensions are independent optional add-ons. The core system (thoughts table +
MCP server) works on its own. Extensions add domain-specific tables and tools for
household knowledge, maintenance tracking, calendaring, meal planning, CRM, and job
hunting. Build whichever ones match your life.

### "Semantic search requires thousands of entries to work"

No, but it gets better with more data. Even with 10-20 entries, semantic search
correctly matches queries to relevant thoughts. With hundreds of entries, the relative
ranking becomes more meaningful because there are more candidates to differentiate.

### "The LLM metadata extraction is critical for retrieval"

No. The embedding is the primary retrieval mechanism. Metadata is a convenience layer
for structured filtering (show me all person_notes, show me tasks with action items).
If the metadata extraction fails or produces inaccurate results, semantic search via
embeddings still works correctly.

### "I need OpenClaw or an autonomous agent"

No. Open Brain works with any MCP-compatible AI client. Claude Desktop and ChatGPT
provide the "pull" model — you ask, the agent queries. Autonomous agents add the
"push" model — they scan proactively. Both are valid. The system is designed to work
with conversational AI first, autonomous agents optionally.

---

## Source File Index

Every file referenced in this document, organized by what it implements:

### Core System

| File | Lines | What it contains |
|------|-------|-----------------|
| `docs/01-getting-started.md` | 124-157 | `thoughts` table DDL + indexes + trigger |
| `docs/01-getting-started.md` | 164-194 | `match_thoughts()` semantic search function |
| `docs/01-getting-started.md` | 201-209 | Core RLS policy (service role only) |
| `docs/01-getting-started.md` | 246-265 | MCP access key generation + secret storage |
| `docs/01-getting-started.md` | 269-365 | Core MCP Edge Function setup + deployment |
| `docs/01-getting-started.md` | 328-337 | `deno.json` dependency configuration |
| `docs/01-getting-started.md` | 369-443 | AI client connection (Claude, ChatGPT, Cursor, etc.) |
| `docs/01-getting-started.md` | 512-518 | "How It Works Under the Hood" — capture and search explained |

### Extension 1: Household Knowledge Base (reference implementation)

| File | Lines | What it contains |
|------|-------|-----------------|
| `extensions/household-knowledge/schema.sql` | 6-16 | `household_items` table |
| `extensions/household-knowledge/schema.sql` | 20-32 | `household_vendors` table |
| `extensions/household-knowledge/schema.sql` | 35-39 | Composite indexes |
| `extensions/household-knowledge/schema.sql` | 42-56 | User-scoped RLS policies |
| `extensions/household-knowledge/schema.sql` | 59-72 | `update_updated_at` trigger |
| `extensions/household-knowledge/index.ts` | 11-18 | MCP SDK + Supabase imports |
| `extensions/household-knowledge/index.ts` | 21-27 | Environment variable validation |
| `extensions/household-knowledge/index.ts` | 30-39 | Supabase client initialization |
| `extensions/household-knowledge/index.ts` | 42-66 | TypeScript type interfaces |
| `extensions/household-knowledge/index.ts` | 69-214 | Tool definitions (5 tools with JSON Schema) |
| `extensions/household-knowledge/index.ts` | 217-242 | `handleAddHouseholdItem` — insert handler |
| `extensions/household-knowledge/index.ts` | 244-277 | `handleSearchHouseholdItems` — search with filters |
| `extensions/household-knowledge/index.ts` | 279-301 | `handleGetItemDetails` — single-row lookup |
| `extensions/household-knowledge/index.ts` | 303-331 | `handleAddVendor` — vendor insert |
| `extensions/household-knowledge/index.ts` | 333-356 | `handleListVendors` — filtered list |
| `extensions/household-knowledge/index.ts` | 358-401 | Server setup + request handler registration |
| `extensions/household-knowledge/index.ts` | 404-413 | Main function (stdio transport) |

### Extension 4: Meal Planning (RLS + shared access)

| File | Lines | What it contains |
|------|-------|-----------------|
| `extensions/meal-planning/schema.sql` | 5-20 | `recipes` table with JSONB ingredients/instructions |
| `extensions/meal-planning/schema.sql` | 23-34 | `meal_plans` table with day/meal constraints |
| `extensions/meal-planning/schema.sql` | 37-45 | `shopping_lists` table with JSONB items |
| `extensions/meal-planning/schema.sql` | 48-51 | GIN index on recipe tags array |
| `extensions/meal-planning/schema.sql` | 54-113 | Household-scoped RLS (JWT role checking) |

### Extension 6: Job Hunt Pipeline (most complex)

| File | Lines | What it contains |
|------|-------|-----------------|
| `extensions/job-hunt/schema.sql` | 6-19 | `companies` table with size/remote CHECK constraints |
| `extensions/job-hunt/schema.sql` | 23-39 | `job_postings` table with salary range + source enum |
| `extensions/job-hunt/schema.sql` | 43-56 | `applications` table with 8-state status pipeline |
| `extensions/job-hunt/schema.sql` | 60-74 | `interviews` table with 7 interview types |
| `extensions/job-hunt/schema.sql` | 78-92 | `job_contacts` table with soft FK to CRM (line 88) |
| `extensions/job-hunt/schema.sql` | 95-115 | 7 indexes including partial index (line 112) |
| `extensions/job-hunt/schema.sql` | 118-148 | User-scoped RLS on all 5 tables |

### Primitives

| File | Lines | What it contains |
|------|-------|-----------------|
| `primitives/rls/README.md` | 40-77 | Pattern 1: User-scoped RLS (full worked example) |
| `primitives/rls/README.md` | 84-187 | Pattern 2: Household-scoped RLS with junction table |
| `primitives/rls/README.md` | 192-245 | Pattern 3: Public + Private visibility |
| `primitives/rls/README.md` | 249-293 | Step-by-step RLS setup guide |
| `primitives/rls/README.md` | 296-331 | Troubleshooting (3 common issues) |
| `primitives/shared-mcp/README.md` | 20-39 | Three-layer security model |
| `primitives/shared-mcp/README.md` | 82-111 | Scoped database role creation (SQL) |
| `primitives/shared-mcp/README.md` | 119-268 | Complete shared MCP server (TypeScript) |
| `primitives/shared-mcp/README.md` | 334-373 | Boundary test script |

### Integrations

| File | Lines | What it contains |
|------|-------|-----------------|
| `integrations/slack-capture/README.md` | 43-50 | Channel creation + Channel ID retrieval |
| `integrations/slack-capture/README.md` | 57-77 | Slack app creation + OAuth scopes |
| `integrations/slack-capture/README.md` | 113-217 | Complete Edge Function (Deno/TypeScript) |
| `integrations/slack-capture/README.md` | 125-133 | `getEmbedding()` implementation |
| `integrations/slack-capture/README.md` | 135-157 | `extractMetadata()` implementation |
| `integrations/slack-capture/README.md` | 167-170 | Slack URL verification handler |
| `integrations/slack-capture/README.md` | 185-188 | `Promise.all` parallel execution |
| `integrations/slack-capture/README.md` | 249-256 | Event Subscriptions setup |

### CI/CD

| File | Lines | What it contains |
|------|-------|-----------------|
| `.github/workflows/ob1-review.yml` | 9-11 | Trigger configuration (PR events) |
| `.github/workflows/ob1-review.yml` | 46-553 | All 13 automated review checks |
| `.github/workflows/ob1-review.yml` | 555-587 | PR comment bot (post/update review) |
| `.github/metadata.schema.json` | 1-116 | JSON Schema for metadata.json validation |
| `.github/PULL_REQUEST_TEMPLATE.md` | — | PR description template |

### Configuration & Governance

| File | What it contains |
|------|-----------------|
| `CONTRIBUTING.md` | Contribution rules, metadata template, 11 review rules explained |
| `CLAUDE.md` | AI agent instructions, guard rails, PR standards |
| `LICENSE.md` | FSL-1.1-MIT terms (no commercial derivatives for 2 years) |
| `CODE_OF_CONDUCT.md` | Contributor Covenant community standards |
| `SECURITY.md` | Vulnerability reporting policy |
