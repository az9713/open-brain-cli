---
name: ob-cleanup
description: Maintain and clean up the user's Open Brain — delete thoughts, find duplicates, identify low-quality captures, and keep the knowledge base healthy. Use this skill when the user says "clean up my brain", "delete that thought", "remove", "find duplicates", "deduplicate", "brain maintenance", "clean up old thoughts", "that capture was wrong — delete it", or when they want to manage the quality of their stored thoughts. Also triggers on "fix that last capture" or "undo that save".
---

# Open Brain Cleanup

## What This Does

Helps the user maintain their Open Brain by deleting unwanted thoughts, finding duplicates, and identifying captures that should be improved or removed.

## Deleting a Thought

When the user wants to remove a specific thought:

### 1. Find It

If the user describes what they want to delete but doesn't have the ID:

```bash
ob search "the content they described" --json
```

Or check recent captures:

```bash
ob recent 10
```

The `--json` flag on search gives you the `id` field for each result.

### 2. Confirm With the User

Show them the thought content and ask for confirmation before deleting. Deletion is permanent.

### 3. Delete

```bash
ob delete "the-thought-uuid"
```

Exit code 0 = deleted. Exit code 1 = not found (wrong ID).

## Fixing a Bad Capture

If the user says "that last one was wrong" or "fix that capture":

1. Find the thought via `ob recent 1` or `ob search`
2. Delete the bad version: `ob delete "id"`
3. Recapture with corrected content: `ob capture "improved version"`

There's no edit/update command — delete + recapture is the workflow.

## Finding Duplicates

When the user suspects they've captured the same thing multiple times:

### 1. Search for the Topic

```bash
ob search "the suspected duplicate topic" --threshold 0.8 --count 20
```

High threshold (0.8+) finds near-identical content.

### 2. Compare Results

Look at the results for thoughts that say essentially the same thing. Present the duplicates to the user with their IDs and ask which to keep.

### 3. Delete the Extras

```bash
ob delete "duplicate-id-1"
ob delete "duplicate-id-2"
```

Keep the version with the best metadata or most complete content.

## Brain Maintenance Audit

For a broader cleanup session:

### 1. Get an Overview

```bash
ob stats
ob recent 50
```

### 2. Look for Issues

- **Near-duplicates:** Search key topics with high threshold
- **Vague captures:** Thoughts with "uncategorized" topics or generic content that won't be useful in search
- **Test captures:** Early setup tests like "My first Open Brain thought" that can be deleted
- **Stale action items:** Action items with past dates that were never completed or followed up

### 3. Present Findings

Show the user what you found and let them decide what to do with each. Never delete without asking.

### 4. Execute

Delete approved items, recapture any that need improvement.

## Important Rules

- **Always confirm before deleting.** Show the content, get explicit approval.
- **Deletion is permanent.** There's no undo. Make sure the user understands this.
- **When in doubt, keep it.** A slightly redundant brain is better than one with gaps. Only delete clear duplicates or things the user explicitly wants removed.
- **Recapture after delete** if the thought had value but needed improvement. Don't just delete — replace.
