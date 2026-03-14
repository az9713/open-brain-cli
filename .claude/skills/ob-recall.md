---
name: ob-recall
description: Search and retrieve information from the user's Open Brain. Use this skill whenever the user wants to find, recall, search, look up, or retrieve something they previously captured. Triggers on "what do I know about", "find my notes on", "recall", "search my brain", "what did I capture about", "what was that thing about", "who mentioned", "do I have anything on", or any question that might be answered by their stored thoughts. Also use when the user needs context before a meeting, wants to prepare for a conversation, or asks about a person or topic they've discussed before. Even vague prompts like "wasn't there something about X?" should trigger this.
---

# Open Brain Recall

## What This Does

Searches the user's Open Brain using `ob search`, which performs semantic similarity matching — it finds thoughts by meaning, not by keyword. The query "career changes" will find "Sarah is thinking about leaving her job to start consulting" even though they share no words.

## Formulating Good Queries

The search query gets embedded and compared against all stored thoughts. Natural language works best:

- "career changes and people leaving" > "career leave job" (more semantic context)
- "API redesign decisions from design team" > "API stuff" (specificity matters)
- Include people's names when relevant: "Sarah" as a query finds all Sarah-related thoughts

## Running a Search

```bash
ob search "the search query"
```

Default: up to 10 results with similarity >= 70%.

**Adjusting results:**
```bash
ob search "query" --threshold 0.4 --count 20   # wider net
ob search "query" --threshold 0.85 --count 5    # high-confidence only
ob search "query" --json                         # raw JSON for processing
```

## Presenting Results

Don't just dump the raw output. Synthesize:
- Group related thoughts together
- Highlight people, dates, and action items
- High scores (80%+) = strong matches; 50-70% = tangential connections
- If multiple thoughts paint a picture, combine them into a coherent answer

## When Search Isn't Enough

**Try recent thoughts** if the user remembers *when* but not *what*:
```bash
ob recent 20
```

**Run multiple searches** for broad questions ("what's going on with project X?"):
```bash
ob search "project X decisions and status"
ob search "project X action items and deadlines"
ob search "project X people involved"
```

**Pre-meeting prep** — pull everything about a person and related context:
```bash
ob search "Marcus"
ob search "platform team reorg"
```
Then combine into a briefing.

## Answering vs. Listing

- **Factual question** ("what paint color is in the living room?") — search, find the answer, state it directly
- **Open question** ("what have I been thinking about careers?") — search and present relevant thoughts with context
- **Nothing found** — say so honestly. Don't fabricate. Suggest the user capture the information if they know it. Try lowering threshold to 0.3 before giving up.

## If Something Goes Wrong

- **No results** — try `--threshold 0.3`. If still empty, the brain may not have relevant captures yet.
- **Irrelevant results** — the query may be too vague. Help the user refine with more specific terms.
- **"Failed to generate query embedding"** — OpenRouter issue. Run `ob check`.
