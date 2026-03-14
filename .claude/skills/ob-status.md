---
name: ob-status
description: Check the health and status of the user's Open Brain — connectivity, configuration, statistics, and overall brain health. Use this skill when the user asks "is my brain working", "check my setup", "how's my brain", "brain status", "how many thoughts do I have", "am I connected", "is Open Brain configured", or reports any error from the ob CLI. Also use proactively at the start of a session when the user intends to work with Open Brain, to verify everything is connected before they hit an error.
---

# Open Brain Status

## What This Does

Checks whether the `ob` CLI is properly configured and connected, and provides a snapshot of the user's brain health — how many thoughts, what they're about, and whether the system is working.

## Connectivity Check

```bash
ob check
```

This verifies:
- All 3 env vars are set (`OB_SUPABASE_URL`, `OB_SUPABASE_KEY`, `OB_OPENROUTER_KEY`)
- Supabase is reachable (HTTP 200)
- OpenRouter is reachable (HTTP 200)

If anything fails, tell the user exactly what's wrong and how to fix it:
- Missing env vars → add to `~/.bashrc` or `~/.zshrc`, then `source` it
- Supabase fails → check the URL (should be `https://your-ref.supabase.co`) and key
- OpenRouter fails → check the key at openrouter.ai/keys, verify credits remain

## Brain Statistics

```bash
ob stats
```

Shows: total thoughts, date range (oldest to newest), top 5 topics, top 5 people mentioned, and type distribution (observation, task, idea, etc.).

Present this as a health dashboard:
- **Healthy brain:** Regular captures across multiple topics and types
- **Cold brain:** No recent captures — encourage the user to build the habit
- **Narrow brain:** All one type or topic — suggest diversifying captures

## Version Info

```bash
ob version
```

Shows version number and which services are configured. Useful for debugging.

## When to Use Proactively

If the user says something like "let me save this to my brain" or "search my Open Brain for..." and you haven't verified connectivity yet in this session, run `ob check` first. A 2-second check saves the user from a cryptic error message mid-capture.

## Troubleshooting Common Issues

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "OB_SUPABASE_URL is not set" | Env vars not loaded | `source ~/.bashrc` or add exports |
| Supabase HTTP 401 | Wrong key | Use the Secret/service role key, not the anon key |
| OpenRouter HTTP 401 | Invalid API key | Regenerate at openrouter.ai/keys |
| OpenRouter HTTP 402 | No credits | Add credits at openrouter.ai |
| "curl is required" | Missing dependency | Install curl via package manager |
| "jq is required" | Missing dependency | Install jq via package manager |
