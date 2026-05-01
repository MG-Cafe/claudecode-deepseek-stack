# Tool Use & Subagent Guide for DeepSeek Sessions

How tool use works (and breaks) when routing Claude Code through DeepSeek's Anthropic-compatible endpoint.

## TL;DR

Basic tool use works. Multi-turn tool loops with thinking mode enabled will fail unless your client round-trips thinking blocks. Subagent spawning depends on this working correctly.

---

## What Works

| Feature | Status | Notes |
|---------|--------|-------|
| `tools` parameter in Messages API | Full support | `name`, `input_schema`, `description` all work |
| `tool_choice` (`none`, `auto`, `any`, `tool`) | Supported | |
| `tool_use` content blocks in responses | Full support | `id`, `input`, `name` all present |
| `tool_result` content blocks in requests | Full support | `tool_use_id` + `content` work |
| Nested/complex JSON schemas | Supported | `enum`, `anyOf`, `$ref`, `$def` all work |
| Read/Write/Edit/Grep/Glob/Bash tools | Work | Standard Claude Code tool loop functions |
| Subagent env var inheritance | Works | `ANTHROPIC_BASE_URL` + `ANTHROPIC_API_KEY` pass to child agents |

**Both `deepseek-v4-pro` and `deepseek-v4-flash` support tool use.**

---

## What Breaks

### 1. Thinking Blocks Round-Trip (Critical)

**This is the #1 cause of subagent failures on DeepSeek.**

When thinking mode is active and the model makes tool calls, DeepSeek requires all `thinking` content blocks to be included in subsequent API requests. Native Anthropic does not require this.

If your client strips thinking blocks between turns:
```
HTTP 400: "The content[].thinking in the thinking mode must be passed back to the API."
```

**Impact:** Multi-turn tool-calling conversations die mid-conversation. Since subagents are multi-turn tool-calling conversations by definition, this breaks agent spawning.

**Workaround:** Disable extended thinking for DeepSeek sessions, or ensure your client preserves thinking blocks in conversation history. Claude Code's `--model` flag controls this, but the thinking behavior is tied to the model tier, not a separate flag.

### 2. `is_error` on `tool_result` (Degraded)

DeepSeek silently ignores the `is_error` field on `tool_result` blocks. Every tool result is treated as successful.

**Impact:** When a tool call fails (file not found, command errors, permission denied), the model doesn't know it failed. It proceeds with incorrect assumptions instead of retrying or adjusting.

**Workaround:** Prefix error tool results with a clear text marker like `ERROR:` or `FAILED:` in the content string so the model can detect failures from content alone.

### 3. `disable_parallel_tool_use` (Ignored)

The `disable_parallel_tool_use` field on `tool_choice` is ignored. DeepSeek decides independently whether to emit parallel tool calls.

**Impact:** Claude Code may expect sequential tool execution but receive parallel calls, or vice versa. Usually benign, but can cause ordering issues in dependent operations.

### 4. No Prompt Caching

All `cache_control` fields are ignored. Tool schemas are re-processed on every request.

**Impact:** Higher input token costs on long conversations with many tools. No way to amortize the cost of large tool definitions across turns.

### 5. No Image/Document Content

Image and document content blocks in messages are not supported.

**Impact:** Screenshots, PDF reading, and multimodal tool results won't work. The `Read` tool can't display images. Browser automation tools that return screenshots will fail.

---

## Subagent Configuration

### How Routing Works

Subagents inherit `ANTHROPIC_BASE_URL` and `ANTHROPIC_API_KEY` from the parent session. No per-agent configuration needed.

```
ds-pro session
  └── Agent: smart-searcher    → DeepSeek (inherited)
  └── Agent: lg-researcher     → DeepSeek (inherited)
  └── Agent: lg-writer         → DeepSeek (inherited)
```

### Pinning an Agent to Anthropic

For agents that need full Anthropic capabilities (image processing, error recovery, thinking + tools):

```yaml
---
name: agent-name
model: claude-sonnet-4-6
env:
  ANTHROPIC_BASE_URL: ""  # clears the DeepSeek override
---
```

### Recommended Pinning Strategy

| Agent Type | Pin to Anthropic? | Why |
|-----------|-------------------|-----|
| Search/triage (smart-searcher) | No | Simple tool loops, rarely fails |
| Content writers (lg-writer) | Optional | Voice fidelity may matter |
| Research agents (lg-researcher) | No | Web search + file reads work fine |
| Browser automation | Yes | Needs screenshot/image support |
| Multi-step orchestrators | Yes | Complex tool chains + error recovery |

---

## Practical Recommendations

### For Session Launchers

Add a `--no-thinking` or equivalent flag to DeepSeek session launchers if your Claude Code version supports it. This avoids the thinking block round-trip issue entirely.

If `--no-thinking` isn't available, consider:
1. Using `deepseek-v4-flash` (less likely to trigger extended thinking)
2. Keeping complex multi-agent orchestration on Anthropic sessions
3. Using DeepSeek for single-agent, low-tool-use tasks (writing, simple edits)

### For Agent Authors

When writing agents that need to work on both backends:

1. **Don't rely on `is_error`** — include error context in the tool result content string
2. **Keep tool chains short** — fewer turns = fewer chances for thinking block issues
3. **Avoid image/document tools** — no multimodal support on DeepSeek
4. **Test on both backends** — what works on Anthropic may silently degrade on DeepSeek

### Checking Which Backend You're On

```
# In a Claude Code session, the header shows:
# "API Usage Billing" → DeepSeek
# "Claude Max" (or plan name) → Anthropic
```

---

## DeepSeek API Reference

- [Anthropic API Compatibility](https://api-docs.deepseek.com/guides/anthropic_api)
- [Tool Calls Documentation](https://api-docs.deepseek.com/guides/tool_calls)
- [Thinking Mode Documentation](https://api-docs.deepseek.com/guides/thinking_mode)
- [Claude Code Integration Guide](https://api-docs.deepseek.com/guides/agent_integrations/claude_code)
- [V4 Model Specs](https://api-docs.deepseek.com/news/news260424)
