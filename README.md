# Make Claude Code Free with DeepSeek V4

Replace Anthropic's $200/month Claude Code billing with DeepSeek V4 for **~$7/month**. Same Claude Code CLI you already use. Two environment variables. No third-party tool.

> **Live-tested April 2026** — same 3D Three.js prompt: Anthropic Opus 4.7 = **$1.14**, DeepSeek V4 Flash = **$0.012**. **95× cheaper**, both produce working playable demos.

---

## Setup (5 minutes)

### 1. Get a DeepSeek API key

1. Go to https://platform.deepseek.com
2. Sign up (free, $10 starter credit — runs months of coding)
3. Create an API key in the dashboard
4. Copy the key (starts with `sk-`)

DeepSeek's API is currently **75% off until May 5, 2026** — cheapest moment to try this.

### 2. Save the key locally

```bash
mkdir -p ~/.config/mg-deepseek && chmod 700 ~/.config/mg-deepseek
echo 'export DEEPSEEK_API_KEY="sk-your-key-here"' > ~/.config/mg-deepseek/key.env
chmod 600 ~/.config/mg-deepseek/key.env
source ~/.config/mg-deepseek/key.env
```

Mode 600 = only your user can read it. Don't paste it on camera or commit it.

### 3. Run Claude Code with DeepSeek

Inline-prefix the `claude` command with two env vars:

```bash
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic \
ANTHROPIC_AUTH_TOKEN="$DEEPSEEK_API_KEY" \
claude --model sonnet "Build me a REST API with auth, JWT, and tests"
```

That's it. Claude Code now talks to DeepSeek V4 Flash for that one command. Your shell config is untouched.

---

## How it works

- DeepSeek exposes an **Anthropic-compatible API** at `https://api.deepseek.com/anthropic`
- Claude Code respects two env vars:
  - `ANTHROPIC_BASE_URL` — where to send the API request
  - `ANTHROPIC_AUTH_TOKEN` — the auth header to attach
- Inline-prefixing them in front of `claude` applies them to **that one command only** — no `export`, no shell config changes, no install
- Use `--model sonnet` (Claude Code validates against Anthropic's allowlist; DeepSeek's compat endpoint maps the alias to V4 Flash)

---

## The 80/20 hybrid

For most coding (refactors, tests, scaffolding, docs) — DeepSeek matches or beats Opus at ~1% of the cost. For the genuinely hard 20% (architecture, complex debugging) — keep using your normal Claude Code.

```bash
# Easy 80%: cheap stack
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic \
ANTHROPIC_AUTH_TOKEN="$DEEPSEEK_API_KEY" \
claude --model sonnet "Refactor this codebase to use async/await"

# Hard 20%: plain claude (your normal setup, unchanged)
claude --model opus "Architect a multi-region failover system"
```

### Optional: shorten the prefix with an alias

```bash
# Add to your .zshrc or .bashrc
alias claude-cheap='ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic ANTHROPIC_AUTH_TOKEN="$DEEPSEEK_API_KEY" claude --model sonnet'
```

Then:
- `claude-cheap "Write tests"` → DeepSeek (~1¢)
- `claude "Hard problem"` → your default Claude Code

---

## Live-tested cost data

| Task | Backend | Time | Cost |
|---|---|---|---|
| REST API + tests (FastAPI + JWT + bcrypt) | DeepSeek Flash | 1m 15s | **$0.0094** |
| 3D Three.js walking world (WASD + collision + skybox) | Anthropic Opus 4.7 | 1m 30s | **$1.14** |
| 3D Three.js walking world (same prompt) | DeepSeek Flash | 2m 02s | **$0.012** |

Cost ratio: **95× cheaper** for the same outcome. Both produced working playable 3D demos with identical functionality.

Heavy daily use lands at **~$7/MONTH** total on DeepSeek with the 75% off promo. (Math check: that's per *month*, not per day. $7×30 = $210, which would be more than Claude — the savings only work at the monthly figure.)

---

## Notes

- **Model name:** Use `--model sonnet` (or `opus`). Claude Code rejects `deepseek-v4-pro` directly because of its hardcoded Anthropic-only allowlist. The `sonnet` alias gets mapped server-side to V4 Flash.
- **DeepSeek V4 Pro:** if you specifically need Pro (better reasoning, slower), call DeepSeek's API directly via `curl` or Python — that bypasses Claude Code's allowlist.
- **No `export` to `.zshrc`** for the `ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` vars. Inline-prefix only — that way the swap applies per-command and your existing Claude Code setup keeps working when you don't want DeepSeek.

---

## Troubleshooting

**`There's an issue with the selected model (deepseek-v4-pro)`**
Use `--model sonnet` or `--model opus`, not the DeepSeek model name. Claude Code only accepts Anthropic aliases.

**`401 Unauthorized`**
Your DeepSeek key isn't being passed. Re-check `echo $DEEPSEEK_API_KEY` shows the key, then re-source `~/.config/mg-deepseek/key.env`.

**Want to verify the swap actually fired?**
```bash
ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic \
ANTHROPIC_AUTH_TOKEN="$DEEPSEEK_API_KEY" \
claude --print --model sonnet "Reply with: SWAP_CONFIRMED"
```
If you see `SWAP_CONFIRMED` and your DeepSeek dashboard shows ~$0.0001 spent, it worked.

---

## Resources

- DeepSeek API docs: https://api-docs.deepseek.com
- DeepSeek 75% off pricing page: https://api-docs.deepseek.com/models-pricing
- Anthropic Claude Code docs: https://docs.anthropic.com/claude/docs/claude-code
