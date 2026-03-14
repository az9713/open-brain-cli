---
name: ob-migrate
description: Import memories and knowledge into the user's Open Brain from other sources — AI conversation history, existing notes, documents, or any structured/unstructured text the user wants to preserve. Use this skill when the user says "migrate my memories", "import my notes", "move my knowledge to Open Brain", "bring over my notes from Notion/Obsidian/Apple Notes", "extract what you know about me", "save everything you know about me to my brain", "transfer my memories", or wants to bulk-capture information from an external source into their Open Brain.
---

# Open Brain Migration

## What This Does

Helps the user move existing knowledge — from AI memory, note-taking apps, documents, or conversation history — into their Open Brain by structuring it into well-formatted captures.

## Migration from AI Memory (This Conversation)

When the user asks you to extract what you know about them and save it:

### 1. Gather What You Know

Pull together everything from the current conversation and any context you have about the user:
- Their role, job, projects
- People they've mentioned (colleagues, family, contacts)
- Decisions they've made
- Preferences and workflows
- Key facts and reference information

### 2. Organize Into Categories

Group by type:
- **People** — one capture per person with details, role, relationship
- **Projects** — one capture per active project with status, decisions, stakeholders
- **Preferences** — workflow preferences, tool choices, communication style
- **Decisions** — key decisions with reasoning
- **Facts** — reference information (account details, config values, etc.)

### 3. Capture Each One

```bash
ob capture "Marcus — colleague, overwhelmed since the reorg. Wants to move to platform team. Wife just had a baby."
ob capture "Project: API redesign. Status: in progress. Design team cutting sidebar panels, keeping revenue chart. API spec due Thursday."
ob capture "Preference: Uses Claude Code for development, prefers CLI over GUI tools."
```

Present each batch to the user before capturing — they should approve what gets saved. Don't capture anything the user hasn't confirmed.

### 4. Verify

```bash
ob search "Marcus"
ob stats
```

Confirm the captures landed and are searchable.

## Migration from Notes (Notion, Obsidian, Apple Notes, etc.)

When the user provides exported notes or pastes content from another system:

### 1. Read the Source

Ask the user to paste content or provide a file path. Read and understand the structure.

### 2. Chunk Into Atomic Thoughts

Each capture should represent one idea, one fact, or one note. Long documents should be split at natural boundaries — headings, paragraphs, or topic changes.

The chunking principle: if two pieces of information would answer different search queries, they should be separate captures.

### 3. Enrich for Metadata

Add structure that helps the metadata extractor:
- Include people's names explicitly
- Add context about what project or area this relates to
- Make action items explicit with "Action:" prefix

**Before:** "Need to update the homepage"
**After:** "Action: Update the company homepage with new pricing. Part of the Q1 marketing refresh."

### 4. Capture in Batches

Don't fire off 100 captures without checking. Do 5-10 at a time, verify they look right with `ob recent 10`, then continue.

### 5. Report Progress

After each batch:
- How many captures so far
- Any that had weird metadata (suggest recapturing)
- Estimated remaining

## Migration from Conversation Exports (ChatGPT, Claude)

If the user has a JSON or text export from another AI:

1. Read the export file
2. Extract the substantive content — skip small talk, focus on decisions, facts, people, insights
3. Group by topic
4. Format each as a capture-ready thought
5. Present to user for approval
6. Capture in batches

## Important Rules

- **Always get user approval** before capturing. Show them what you're about to save.
- **Don't capture duplicates.** Run `ob search` for key terms before capturing to check if the thought already exists.
- **Preserve the user's voice.** Keep their phrasing when possible — just add structure for better metadata extraction.
- **One idea per capture.** Resist the urge to combine — tighter captures = better search.
