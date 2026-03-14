---
name: ob-capture
description: Save thoughts to the user's Open Brain. Use this skill whenever the user wants to remember, save, note, capture, or log any information — meeting notes, person details, decisions, ideas, observations, action items, or references. Triggers on phrases like "remember this", "save this", "note that", "don't forget", "capture", "log this", "put this in my brain", or when the user shares information prefixed with "Meeting with...", "Decision:", "Insight:", or person names followed by details. Even casual phrasing like "oh that's worth remembering" or "I should write that down" should trigger this skill.
---

# Open Brain Capture

## What This Does

Takes something the user said and saves it to their Open Brain database using `ob capture`. The CLI handles the hard parts automatically — generating a vector embedding for semantic search and extracting structured metadata (people, topics, action items, thought type) via LLM.

The reason formatting matters: the metadata extractor (gpt-4o-mini) produces better tags when the input has clear structure. A well-formatted capture is easier to find later.

## Before First Capture

Run `ob check` once per session to verify connectivity. If it fails, the user needs to set `OB_SUPABASE_URL`, `OB_SUPABASE_KEY`, and `OB_OPENROUTER_KEY` — either via a `.env` file (copy `.env.example` from repo root and set `OB_ENV_FILE`) or shell exports. Point them to `docs/01-getting-started.md`.

## How to Format Thoughts

Transform the user's raw input into a capture-ready string. The goal is to preserve their intent while giving the metadata extractor clear signals.

**Lead with names** when the thought is about a person:
```
"Marcus — overwhelmed since the reorg, wants to move to platform team. Wife just had a baby."
```
Not: "someone at work is stressed." Names trigger person extraction.

**Structure meetings** with attendees, decisions, and actions:
```
"Meeting with Sarah and Tom about dashboard redesign. Decided to cut sidebar panels. Action: I send API spec by Thursday."
```

**Make decisions explicit** with reasoning:
```
"Decision: Moving launch to March 15. Reason: QA found 3 blockers in payment flow."
```

**Separate action items** with owners and dates:
```
"Action: Design team sends revised mockups by Monday."
```

**Add context** to observations — what makes it searchable later:
```
"Sarah mentioned thinking about leaving her job to start a consulting business"
```
Not: "Sarah might leave." The context (consulting, career change) creates richer embedding.

## Executing the Capture

```bash
ob capture "the formatted thought text"
```

The output is JSON with `id`, `content`, `metadata`, and `created_at`. Show the user the extracted metadata (topics, people, type) so they can confirm it looks right.

## When to Split Into Multiple Captures

One idea per capture produces tighter embeddings and better search. If the user gives you a wall of text with multiple topics, split it:

```bash
ob capture "Meeting with design team: decided to cut sidebar panels and keep revenue chart"
ob capture "Action: I send API spec to design team by Thursday"
ob capture "Action: Design team sends revised mockups by Monday"
```

Splitting guideline: if two pieces of information would answer different search queries, they should be separate captures.

## Metadata Types

The extractor classifies each thought as one of: `observation`, `task`, `idea`, `reference`, `person_note`. It also extracts `people` (names mentioned), `topics` (1-3 tags), and `action_items` (implied to-dos). All thoughts get tagged with `"source": "ob-cli"` automatically.

## If Something Goes Wrong

- **"embedding failed"** — OpenRouter key or credits issue. Run `ob check`.
- **"Failed to insert thought"** — Supabase connection. Check URL and key.
- **Metadata says "uncategorized"** — The text was too vague. Recapture with specifics.
- **Wrong type/topics** — Not worth fixing. The embedding (which powers search) works regardless of metadata classification.
