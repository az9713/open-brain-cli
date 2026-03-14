---
name: ob-review
description: Run a structured review of the user's Open Brain to surface themes, patterns, open action items, and connections between thoughts. Use this skill when the user asks for a "weekly review", "brain review", "what patterns do I see", "review my thoughts", "what action items are open", "what have I been thinking about", "summarize my recent captures", "end of week review", or wants any kind of synthesis or reflection over their captured thoughts. Also triggers on "what themes am I seeing", "give me an overview", or before planning sessions when the user wants to take stock.
---

# Open Brain Review

## What This Does

Pulls recent thoughts and statistics from Open Brain, then synthesizes them into a review that surfaces patterns, open action items, connections, and gaps the user might not notice on their own. Think of it as a conversation with someone who's read all your notes.

## The Review Process

### 1. Gather Data

```bash
ob recent 50
ob stats
```

Adjust the count based on how active the user is. For weekly reviews, 50 is usually enough. For monthly, try 200.

### 2. Search for Review Themes

```bash
ob search "action items and tasks to do" --count 20 --threshold 0.5
ob search "decisions made recently" --count 15 --threshold 0.5
ob search "people and relationships" --count 15 --threshold 0.5
ob search "ideas and opportunities" --count 15 --threshold 0.5
```

### 3. Synthesize Into Sections

**Themes:** What topics keep showing up? Name them. "You've captured 8 thoughts about the API redesign in the past 2 weeks — clearly top of mind."

**Open Action Items:** Extract from recent thoughts. Flag anything overdue based on mentioned dates:
- [ ] Send API spec to design team (mentioned March 10)
- [ ] Follow up with Marcus about platform team transfer

**People:** Who appears most? What's the context? Useful before 1:1s.

**Connections:** Non-obvious links between thoughts. "You mentioned Sarah is considering consulting AND you need a freelance designer for project X — is there a connection?"

**Gaps:** What's missing? Lots of work thoughts but no personal ones? Many observations but no decisions? Note it without judgment.

**Numbers:** Thoughts this period, top topics, most mentioned people, capture streak.

### 4. Suggest Next Captures

Based on gaps: "You mentioned a meeting with the design team but didn't capture the outcomes — worth saving?"

## Quick Review

If the user just wants a pulse check, not a full review:

```bash
ob stats
ob recent 10
```

Summarize in 3-4 sentences: count, top themes, any urgent action items.

## Monthly Review

Widen scope and add trajectory analysis:

```bash
ob recent 200
ob search "goals and priorities" --count 20 --threshold 0.4
ob search "wins and accomplishments" --count 20 --threshold 0.4
ob search "frustrations and blockers" --count 20 --threshold 0.4
```

## Tone

Present conversationally, not as a report. Quote from captured thoughts to ground the analysis. The user should feel like they're talking to someone who's been paying attention.
