# Subagent Mapping: DeepSeek Cost Routing

How subagents inherit model routing — and how to configure the system so the right model runs for each task tier.

## How Inheritance Works

Claude Code subagents **inherit env vars from their parent session**. This is the key.

When you launch a session via `ds-pro`, the shell sets:
```
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
ANTHROPIC_API_KEY=sk-...
```

Every subagent Claude spawns inside that session reads the same env vars → routes to DeepSeek automatically. You don't configure subagents individually.

```
ds-pro (terminal launch)
  └── Agent: smart-searcher    → DeepSeek v4-flash (inherits base URL)
  └── Agent: task-orchestrator → DeepSeek v4-flash (inherits base URL)
  └── Agent: lg-researcher     → DeepSeek v4-pro   (inherits base URL)
```

If you launch with `claude` (Anthropic session), all subagents use Anthropic. The routing decision is made at session start, not per-agent.

## Model Tier Mapping

| Agent Role | Claude Model | DeepSeek Equivalent | Cost Reduction |
|-----------|-------------|-------------------|---------------|
| Search / triage / file ops | claude-haiku-4-5 | deepseek-v4-flash | ~60x |
| Routing / orchestration | claude-sonnet-4-6 | deepseek-v4-flash | ~40x |
| Research / reasoning / planning | claude-opus-4-6 | deepseek-v4-pro | ~30x |

DeepSeek's Anthropic-compatible endpoint accepts Claude model names and maps them server-side. You don't need to change agent `.md` files — the base URL override handles routing.

## Usage-Based Throttle

The throttle automatically redirects Opus-tier sessions to DeepSeek once weekly spend crosses your threshold.

```powershell
# Set your weekly budget in the PS profile
$budget = 700  # USD — adjust to your plan

# When spend > $budget, claude-opus and cs redirect to ds-pro automatically
claude-opus  # → ds-pro if throttled, claude-opus-4-6 if not
cs           # → ds-pro if throttled, claude-sonnet-4-6 if not
```

**Usage resets Monday.** Reads local Claude Code usage history via `ccusage` — no API calls, no external service.

Check status anytime:
```powershell
usage
# Week: $749.72 / ~$875 [THROTTLED → DeepSeek]
# [#################] 85.7%
```

## Session Launch Reference

| Command | Model | When to Use |
|---------|-------|-------------|
| `ds-pro` | DeepSeek v4-pro | Primary workhorse — all subagents on DeepSeek |
| `ds-flash` | DeepSeek v4-flash | Cheap triage/search sessions |
| `cs` | Sonnet or ds-pro (throttle-aware) | Smart default — picks for you |
| `claude-opus` | Opus 4.6 or ds-pro (throttle-aware) | Heavy reasoning when budget allows |
| `claude-sonnet` | Sonnet 4.6, always Anthropic | Never throttled |
| `deepseek-pro` | DeepSeek v4-pro | One-shot non-interactive (scripting) |
| `deepseek-flash` | DeepSeek v4-flash | One-shot non-interactive (scripting) |

## What the Throttle Does NOT Cover

- Sessions launched via `claude` directly (bypasses PS functions)
- External agent systems (Clay Claygent, custom API wrappers, standalone scripts)
- Subagents within a non-throttled Anthropic session

For full coverage: always launch via `cs` or `claude-opus` rather than `claude` directly.

## Agent File Pinning

Pin specific agents to always use Anthropic (e.g. voice-critical content writers):

```yaml
---
name: lg-writer
model: claude-sonnet-4-6
env:
  ANTHROPIC_BASE_URL: ""  # forces Anthropic even inside a DeepSeek session
---
```

This overrides the session env var for that agent only.

## Tool Use Limitations in DeepSeek Sessions

Subagent tool use works but has important caveats. See [TOOL-USE-GUIDE.md](TOOL-USE-GUIDE.md) for the full breakdown.

**Key issues affecting subagents:**

1. **Thinking block round-trip** — DeepSeek requires thinking content blocks passed back in subsequent requests during multi-turn tool conversations. If Claude Code strips these, subagent tool loops will fail with HTTP 400. This is the primary cause of "can't invoke subagents" errors.

2. **`is_error` ignored** — Tool failures aren't signaled to the model. Subagents may proceed on bad data after a tool error instead of retrying.

3. **No multimodal** — Agents that use screenshots or document reading must be pinned to Anthropic.

**Recommended pinning for complex agents:**

```yaml
---
name: task-orchestrator
model: claude-sonnet-4-6
env:
  ANTHROPIC_BASE_URL: ""  # force Anthropic for multi-step orchestration
---
```

Simple agents (smart-searcher, file-only operations) work fine on DeepSeek without pinning.

---

## Setting Your Budget Threshold

Edit `$budget` in `Get-ThrottleStatus` inside your PowerShell profile. Rule of thumb: 80% of your weekly Anthropic spend cap. Leaves headroom for sessions that genuinely need Anthropic (voice work, production copy, anything requiring full Claude fidelity).
