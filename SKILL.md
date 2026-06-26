---
name: claude-max-subscription-setup
description: "Set up and optimize Hermes to use Claude Max subscription billing instead of API credits. Use when the user wants to configure claude -p subagent calls to draw from their Max plan, or wants to reduce API token costs through auxiliary model and system prompt optimization."
version: 1.0.0
author: Jeff Nichols / Hermes Agent
license: MIT
platforms: [macos, linux]
metadata:
  hermes:
    tags: [Setup, Cost-Optimization, Claude-Max, Subscription, Billing]
    related_skills: [claude-code, hermes-cost-and-auth]
---

# Claude Max Subscription Setup for Hermes

This skill covers two things:

1. **Routing `claude -p` subagent calls to your Max subscription** instead of API credits
2. **Reducing Hermes API token costs** through model and system prompt optimization

Both are independent — you can do either or both.

---

## Background: How Billing Works

Hermes itself always uses your configured API provider (typically Anthropic API with `ANTHROPIC_API_KEY`). That billing is unavoidable for the main session.

However, when Hermes calls the `claude` CLI via `terminal()` for coding subagent work, those calls *can* draw from a Claude Max subscription instead — saving significant cost for heavy compute tasks that use Opus.

**The credential priority order (highest wins):**
1. `ANTHROPIC_API_KEY` env var
2. `CLAUDE_CODE_OAUTH_TOKEN` env var
3. Stored OAuth credentials (`~/.claude/.credentials.json`)

This means if `ANTHROPIC_API_KEY` is set (it usually is for Hermes), it silently overrides the subscription OAuth token. The fix is to unset it before each `claude -p` call.

**Note on `total_cost_usd` in `claude -p` JSON output:** This field always shows a value regardless of billing method — it's a known cosmetic display bug (GitHub issue #50786). Actual charges go to your subscription quota, not your API account. Anthropic confirmed in June 2026 that `claude -p` continues to work with subscription billing.

**Is this officially supported?** Yes. In May 2026 Anthropic announced they would move `claude -p` and Agent SDK usage off subscription billing onto a separate credit system. They reversed that decision and sent a follow-up email to subscribers in June 2026 stating:

> *"Nothing changes for now. Agent SDK, `claude -p`, and third-party app usage continues to work with your subscription exactly as it did before today, and there's no credit to claim. Your subscription limits are unchanged."*

The subject line was "We're pausing the Agent SDK credit change." This pattern is intentional and supported by Anthropic.

---

## Part 1: Route `claude -p` to Max Subscription

### Step 1: Generate a long-lived OAuth token

Run this in a real terminal (not via Hermes — it requires an interactive browser flow):

```bash
claude setup-token
```

A URL will appear. Open it in your browser, authorize it, and paste the code back when prompted. The CLI will display a token starting with `sk-ant-oat01-...`. **Copy it immediately** — it's shown only once.

Store it securely (1Password or similar). It's valid for 1 year.

### Step 2: Add the token to Hermes

Add to `~/.hermes/.env`:

```bash
CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-...your-token-here...
```

### Step 3: Unset `ANTHROPIC_API_KEY` before `claude -p` calls

In any script or invocation that calls `claude -p`, unset the API key first:

```bash
# In a shell script:
unset ANTHROPIC_API_KEY
claude -p "Your task here" --max-turns 10 --output-format json

# Or inline:
env -u ANTHROPIC_API_KEY claude -p "Your task here" --max-turns 10 --output-format json
```

In Hermes via `terminal()`:
```python
terminal(
    command='env -u ANTHROPIC_API_KEY claude -p "Your task" --max-turns 10 --output-format json',
    workdir="/path/to/project",
    timeout=120
)
```

### Step 4: Verify it's working

```bash
claude auth status --text
```

Should show `Login method: Claude Max account`. If it shows API key, `ANTHROPIC_API_KEY` is still overriding — check your environment.

Run a test call:
```bash
env -u ANTHROPIC_API_KEY claude -p "Reply with just PONG" --max-turns 1 --output-format json
```

Check that `modelUsage` in the output shows `claude-opus-4-8[1m]` — that's the Max subscription model.

### Recommended: Add DISABLE_AUTOUPDATER to all unattended scripts

The `claude` auto-updater rewrites `~/.claude/.credentials.json` mid-flight, which can break long-running sessions. Always add this to unattended scripts:

```bash
export DISABLE_AUTOUPDATER=1
unset ANTHROPIC_API_KEY
```

---

## Part 2: Reduce Hermes API Token Costs

These changes affect the main Hermes session (API billing). Each is independent.

### 2a: Increase prompt cache TTL (biggest single win)

The default 5-minute cache TTL means any pause longer than 5 minutes causes Hermes to re-pay the cache write cost on the next turn. Increasing to 1 hour keeps the cache warm across natural breaks.

```bash
hermes config set prompt_caching.cache_ttl 1h
```

**Impact:** At ~20k system prompt tokens × $3.75/M write cost, each avoided re-write saves ~$0.075. In a working day with breaks this adds up fast.

### 2b: Switch auxiliary models to Haiku

Hermes runs several background model calls for lightweight tasks. By default they all inherit your main model (Sonnet). These are safe to run on Haiku (~20× cheaper):

```bash
hermes config set auxiliary.title_generation.model claude-haiku-3-5
hermes config set auxiliary.approval.model claude-haiku-3-5
hermes config set auxiliary.triage_specifier.model claude-haiku-3-5
hermes config set auxiliary.monitor.model claude-haiku-3-5
hermes config set auxiliary.tts_audio_tags.model claude-haiku-3-5
hermes config set auxiliary.skills_hub.model claude-haiku-3-5
```

**Keep these on Sonnet** (quality-sensitive):
- `compression` — summarizes your conversation history; bad summaries lose context
- `curator` — manages skills and memory
- `background_review` — post-turn self-improvement analysis

### 2c: Disable unused MCP servers

Each MCP server adds its full tool schema to every API call. Disable servers you rarely use:

```bash
hermes config set mcp_servers.cloudflare.disabled true
# Repeat for any other servers you don't use regularly
hermes mcp list   # to see what's loaded
```

**Impact:** The Cloudflare MCP server alone adds ~60 tool schemas (~15,000 tokens) to every turn.

### 2d: Trim the skills index

Every skill's name and description is sent in the system prompt on every turn. Audit and disable skills you don't use via the Hermes dashboard (`/skills` page) or by adding skill names to `~/.hermes/skills/.disabled` (one name per line).

```bash
# Check which skills are loaded and how many
hermes skills list | wc -l
```

**Impact:** ~23 tokens per skill. 100 unused skills = ~2,300 tokens saved per turn.

---

## Part 3: Using Haiku for `claude -p` Tasks That Don't Need Opus

Not every `claude -p` task needs Opus reasoning. Wiki summarization, simple edits, and classification tasks run fine on Haiku at a fraction of the quota:

```bash
# Lighter tasks: Haiku
env -u ANTHROPIC_API_KEY claude -p "Summarize these PRs into wiki docs" \
  --model claude-haiku-3-5 --max-turns 10

# Heavier tasks: let it default to Opus (subscription default)
env -u ANTHROPIC_API_KEY claude -p "Debug this production error and propose a fix" \
  --max-turns 15
```

---

## Quick Reference

```bash
# One-time setup
claude setup-token                                    # generate OAuth token
echo "CLAUDE_CODE_OAUTH_TOKEN=sk-ant-oat01-..." >> ~/.hermes/.env

# Every claude -p call
env -u ANTHROPIC_API_KEY claude -p "task" --max-turns 10 --output-format json

# Cost optimization (run once)
hermes config set prompt_caching.cache_ttl 1h
hermes config set auxiliary.title_generation.model claude-haiku-3-5
hermes config set auxiliary.approval.model claude-haiku-3-5
hermes config set auxiliary.triage_specifier.model claude-haiku-3-5
hermes config set auxiliary.monitor.model claude-haiku-3-5
hermes config set auxiliary.tts_audio_tags.model claude-haiku-3-5
hermes config set auxiliary.skills_hub.model claude-haiku-3-5
```

---

## Pitfalls

1. **`ANTHROPIC_API_KEY` silently overrides everything** — always unset it before `claude -p` calls. Stale env vars in shell profiles are the most common cause of unexpected API billing.

2. **`total_cost_usd` in JSON output is always populated** — this is a display bug, not evidence of API billing. Check `claude auth status --text` to confirm auth method.

3. **`claude setup-token` output is shown only once** — the CLI masks it after display. Store it immediately.

4. **The auto-updater rewrites credentials** — `DISABLE_AUTOUPDATER=1` is essential for any unattended script that calls the `claude` binary, not just ones using the OAuth token.

5. **Max subscription has rate limits** — it's not unlimited. Heavy sustained `claude -p` usage (many parallel calls) will hit the 5-hour rolling window. Fall back to API billing for burst workloads.

6. **`claude setup-token` requires an interactive terminal** — it can't be run headlessly or via Hermes `terminal()`. Run it directly in a terminal session.
