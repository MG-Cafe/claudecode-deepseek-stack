# Make Claude Code Free/Cheaper with DeepSeek V4

Routes Claude Code through DeepSeek V4 instead of Anthropic — same UI, same skills, same workflow. ~$7/week vs $200+.

**Windows/PowerShell users:** see [docs/POWERSHELL-SETUP.md](docs/POWERSHELL-SETUP.md) for the full setup including throttle-aware session launchers.
**Subagent routing:** see [docs/SUBAGENT-MAPPING.md](docs/SUBAGENT-MAPPING.md) to map your agent tiers to DeepSeek models.
**Tool use & subagents:** see [docs/TOOL-USE-GUIDE.md](docs/TOOL-USE-GUIDE.md) for what works, what breaks, and how to fix subagent spawning on DeepSeek.

## Quick Launch (after setup)

| Command | Alias | Model | Cost tier |
|---------|-------|-------|-----------|
| `ds-pro` | `dsp` | DeepSeek v4-pro | Opus-equivalent, ~30x cheaper |
| `ds-flash` | `dsf` | DeepSeek v4-flash | Haiku-equivalent, ~60x cheaper |
| `cs` | — | Auto (Sonnet or ds-pro) | Throttle-aware smart default |
| `claude-opus` | — | Opus 4.6 or ds-pro | Throttle-aware, no 4.7 |
| `claude-sonnet` | — | Sonnet 4.6 | Always Anthropic |

```powershell
dsp    # DeepSeek v4-pro session
dsf    # DeepSeek v4-flash session
cs     # smart default
```

**How to confirm you're on DeepSeek:** The Claude Code header shows `API Usage Billing` for DeepSeek sessions and `Claude Max` (or your plan name) for Anthropic sessions. The model name displayed (`Opus 4.6`) is what Claude Code uses for client-side validation — the actual inference runs on DeepSeek's backend.

---

## Setup (5 minutes)

### 1. Get a DeepSeek API key

1. Go to https://platform.deepseek.com
2. Sign up (free, $10 starter credit)
3. Create an API key, copy the value (starts with `sk-`)

### 2. Save the key locally

```bash
mkdir -p ~/.config/mg-deepseek && chmod 700 ~/.config/mg-deepseek
echo 'export DEEPSEEK_API_KEY="sk-your-key-here"' > ~/.config/mg-deepseek/key.env
chmod 600 ~/.config/mg-deepseek/key.env
source ~/.config/mg-deepseek/key.env
```

### 3. Create a Claude Code settings override

Most Claude Code installs already have something configured (Anthropic OAuth login, Vertex AI, AWS Bedrock, or an API key). Inline env-var prefixes get overridden by those configs. The reliable approach is a `--settings` override file.

```bash
cat > ~/.config/mg-deepseek/claude-deepseek-settings.json <<EOF
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX": "",
    "ANTHROPIC_VERTEX_PROJECT_ID": "",
    "CLOUD_ML_REGION": "",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "",
    "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "$DEEPSEEK_API_KEY",
    "ANTHROPIC_API_KEY": "$DEEPSEEK_API_KEY"
  }
}
EOF
chmod 600 ~/.config/mg-deepseek/claude-deepseek-settings.json
```

The empty strings on the first 4 keys neutralize Vertex / Bedrock / OAuth-driven defaults. The bottom 3 keys point Claude Code at DeepSeek's Anthropic-compatible endpoint.

### 4. Run Claude Code on DeepSeek

```bash
claude --bare \
  --settings ~/.config/mg-deepseek/claude-deepseek-settings.json \
  --model sonnet \
  "Build me a REST API with auth, JWT, and tests"
```

That's it. Claude Code now talks to DeepSeek V4 Flash (ignore that it says sonnet) for **that one command**. Plain `claude` (without the flags) keeps using your normal backend.

The `--bare` flag is required — it tells Claude Code to read auth strictly from settings, not from OAuth keychain or other auto-detected providers.

### 5. Verify the swap actually fired

```bash
claude --bare --settings ~/.config/mg-deepseek/claude-deepseek-settings.json \
  --debug-file /tmp/check.log --model sonnet "Reply with: SWAP_TEST"
grep "API REQUEST" /tmp/check.log | head -1
```

You should see `/anthropic/v1/messages` in the URL. If you see `/v1/projects/.../publishers/anthropic/...` you're still hitting Vertex; if you see `bedrock-runtime` you're on AWS Bedrock — re-check the JSON file has empty strings for the Vertex / project keys.

---

## Run both backends side-by-side

Two terminals, two backends, simultaneously:

```bash
# Terminal 1 — your normal Claude Code (Vertex / Bedrock / Anthropic API key — unchanged)
claude --model opus "Architect a multi-region failover system"

# Terminal 2 — DeepSeek backend
claude --bare --settings ~/.config/mg-deepseek/claude-deepseek-settings.json \
  --model sonnet "Refactor this codebase to async/await"
```

No conflicts. Different settings sources, different backends, both alive.

---

## Optional: shorten the prefix with an alias

```bash
# Add to your .zshrc or .bashrc
alias claude-cheap='claude --bare --settings ~/.config/mg-deepseek/claude-deepseek-settings.json --model sonnet'
```

Then:

- `claude-cheap "Refactor this"` → DeepSeek V4 Flash (~1¢)
- `claude "Hard problem"` → your default backend

Two commands, two backends, side-by-side in your shell history.

---

## How it works (one paragraph)

Claude Code respects two configuration keys: `ANTHROPIC_BASE_URL` (where to send the API request) and `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY` (the auth header). DeepSeek exposes an Anthropic-compatible endpoint at `https://api.deepseek.com/anthropic` that speaks the same JSON dialect Claude expects. By providing those keys via `--settings` (which overrides the user-level `~/.claude/settings.json`), you can route a single `claude` invocation to DeepSeek without touching any global config. The `--bare` flag is required because OAuth keychain and auto-detected providers (Vertex AI, AWS Bedrock, Anthropic API key) otherwise win the precedence fight against the env settings.

---

## Tool use & subagents

DeepSeek's Anthropic-compatible endpoint supports tool use (function calling), but with caveats that affect subagent spawning. The critical issue: **thinking blocks must be round-tripped** in multi-turn tool conversations, or DeepSeek returns HTTP 400 errors mid-conversation.

See [docs/TOOL-USE-GUIDE.md](docs/TOOL-USE-GUIDE.md) for the full breakdown of what works, what breaks, and recommended agent configurations.

Quick summary:
- Basic tool use (Read, Write, Bash, Grep) works on both v4-pro and v4-flash
- **Agent tool (subagent spawning) requires system prompt injection** — DeepSeek doesn't natively know how to use Claude Code's Agent tool. The `config/agent-boost-prompt.md` file teaches it. Launchers inject this automatically via `--append-system-prompt-file`
- `is_error` on tool results is ignored — prefix error content with `ERROR:` instead
- No image/document/multimodal support — pin browser automation agents to Anthropic
- Subagents inherit env vars from parent session automatically

---

## Notes & troubleshooting

**Model names:** Both `--model claude-opus-4-6` and `--model claude-haiku-4-5-20251001` work directly. The original Claude model aliases (`sonnet`, `opus`) also work — DeepSeek maps them server-side. Use the explicit DeepSeek names for predictable tier routing.

**v4-pro vs v4-flash:** Pro is the reasoning tier (Opus-equivalent), Flash is the fast/cheap tier (Haiku-equivalent). For most coding tasks, Flash is fine. Use Pro for planning, architecture, and complex reasoning.

**`Auth conflict` warning when running interactively?** That means Claude Code sees both an env token and a managed login. The `--bare` flag fixes it by ignoring OAuth/keychain — make sure it's in your command.

**Still getting Anthropic responses?** Check the actual API URL with `--debug-file`:
```bash
claude --bare --settings ~/.config/mg-deepseek/claude-deepseek-settings.json \
  --debug-file /tmp/check.log --model sonnet "test"
grep "API REQUEST" /tmp/check.log
```
If the URL contains `projects/.../publishers/anthropic` you're hitting Vertex; if it contains `bedrock-runtime` you're on AWS Bedrock. The override settings file should produce `/anthropic/v1/messages` — if not, the empty-string keys at the top of the JSON aren't doing their job (check JSON syntax, restart shell, re-source the key file).

**Don't `export ANTHROPIC_BASE_URL=...` in `.zshrc`.** Inline env-var prefixes don't actually win the precedence fight when other backends are configured. The `--bare --settings` flag combo is what works reliably.

---

## Resources

- DeepSeek API docs: https://api-docs.deepseek.com
- DeepSeek 75% off pricing page: https://api-docs.deepseek.com/models-pricing
- Claude Code docs: https://docs.anthropic.com/claude/docs/claude-code
