# Run GLM-5.2 in Claude Code

This guide routes Claude Code's API calls through **Z.ai GLM-5.2 instead of Anthropic** -- same Claude Code UI, same skills, same workflow.

It uses Z.ai's Anthropic-compatible endpoint, so Claude Code can talk to GLM directly. No local proxy is required.

Most importantly: **your normal Claude Code stays untouched**. Plain `claude` keeps using your usual backend, such as Opus on Vertex AI. GLM only activates when you pass `--bare --settings`.

---

## Setup

### 1. Get a Z.ai API key

1. Go to https://docs.z.ai
2. Sign in to Z.ai / Zhipu AI
3. Create an API key
4. Copy the key for the local setup below

### 2. Save the key locally

```bash
mkdir -p ~/.config/mg-glm && chmod 700 ~/.config/mg-glm
echo 'export ZAI_API_KEY="your-zai-key-here"' > ~/.config/mg-glm/key.env
chmod 600 ~/.config/mg-glm/key.env
source ~/.config/mg-glm/key.env
```

Do not paste your real key into this repo, a README, a video description, or your shell history if you are recording.

### 3. Create a Claude Code settings override

Most Claude Code installs already have a provider configured: Anthropic login, Vertex AI, AWS Bedrock, or an API key. Inline env-var prefixes can lose that precedence fight. The reliable approach is a dedicated `--settings` override file.

```bash
cat > ~/.config/mg-glm/claude-glm-settings.json <<EOF
{
  "env": {
    "CLAUDE_CODE_USE_VERTEX": "",
    "ANTHROPIC_VERTEX_PROJECT_ID": "",
    "CLOUD_ML_REGION": "",
    "ANTHROPIC_BASE_URL": "https://api.z.ai/api/anthropic",
    "ANTHROPIC_AUTH_TOKEN": "$ZAI_API_KEY",
    "ANTHROPIC_API_KEY": "$ZAI_API_KEY",
    "ANTHROPIC_DEFAULT_HAIKU_MODEL": "glm-4.7",
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.2",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.2",
    "API_TIMEOUT_MS": "3000000"
  }
}
EOF
chmod 600 ~/.config/mg-glm/claude-glm-settings.json
```

The empty strings neutralize Vertex / Bedrock defaults for this one command. `ANTHROPIC_BASE_URL` points Claude Code at Z.ai, and the model keys map Claude Code's normal aliases to GLM.

### 4. Run Claude Code on GLM-5.2

```bash
claude --bare \
  --settings ~/.config/mg-glm/claude-glm-settings.json \
  --model sonnet \
  "Build me a REST API with auth, JWT, and tests"
```

That command talks to GLM-5.2. Plain `claude` without those flags keeps using your normal backend.

`--bare` is required. It tells Claude Code to read auth from the settings file instead of your OAuth keychain or auto-detected providers.

### 5. Use max effort for harder tasks

```bash
claude --bare \
  --settings ~/.config/mg-glm/claude-glm-settings.json \
  --model sonnet \
  --effort max \
  "Refactor this codebase and add tests"
```

Z.ai's Claude Code docs also mention `/effort max` and `/effort ultracode` for interactive sessions.

### 6. Verify the swap actually fired

```bash
claude --bare --settings ~/.config/mg-glm/claude-glm-settings.json \
  --debug-file /tmp/glm-check.log \
  --model sonnet \
  -p "Reply with: SWAP_TEST" < /dev/null

grep -E "api.z.ai|glm-5.2|/api/anthropic/v1/messages" /tmp/glm-check.log | head -5
```

You should see `glm-5.2`, `/api/anthropic/v1/messages`, or `api.z.ai` in the debug output. If you see a Vertex URL like `projects/.../publishers/anthropic`, your normal backend is still winning and you should re-check `--bare`, `--settings`, and the empty Vertex keys in the JSON.

---

## Optional: 1M context model

Z.ai documents a long-context GLM-5.2 variant for Claude Code:

```json
{
  "env": {
    "ANTHROPIC_DEFAULT_SONNET_MODEL": "glm-5.2[1m]",
    "ANTHROPIC_DEFAULT_OPUS_MODEL": "glm-5.2[1m]",
    "CLAUDE_CODE_AUTO_COMPACT_WINDOW": "1000000"
  }
}
```

If `glm-5.2[1m]` returns an unknown-model error, use the default `glm-5.2` settings above.

---

## Run both backends side by side

Two terminals, two backends, no conflicts:

```bash
# Terminal 1 -- your normal Claude Code, unchanged
claude --model opus "Architect a multi-region failover system"

# Terminal 2 -- GLM-5.2 through Z.ai
claude --bare --settings ~/.config/mg-glm/claude-glm-settings.json \
  --model sonnet \
  "Refactor this codebase to async/await"
```

This is the whole trick: different settings sources, different backends, both alive.

## Optional: shorten with an alias

```bash
# Add to your .zshrc or .bashrc
alias claude-glm='claude --bare --settings ~/.config/mg-glm/claude-glm-settings.json --model sonnet'
```

Then:

- `claude-glm "Refactor this"` -> GLM-5.2
- `claude "Hard problem"` -> your default Claude Code backend

Do not alias plain `claude` unless you intentionally want GLM to become your default.

---

## How it works

Claude Code respects `ANTHROPIC_BASE_URL` plus `ANTHROPIC_AUTH_TOKEN` / `ANTHROPIC_API_KEY`. Z.ai exposes an Anthropic-compatible endpoint at `https://api.z.ai/api/anthropic`, so this guide points Claude Code there and maps Claude's model aliases to GLM models.

The setup is isolated because the provider details live in `~/.config/mg-glm/claude-glm-settings.json`, and you only use them when you pass `--bare --settings`. Your global `~/.claude/settings.json`, shell profile, Vertex AI setup, and normal Opus workflow are not modified.

---

## Troubleshooting

**`1211 Unknown Model`**

Use `glm-5.2` instead of `glm-5.2[1m]`. The long-context alias may depend on the current Z.ai plan, endpoint, or Claude Code integration state.

**`Auth conflict` warning**

Add `--bare`. It prevents Claude Code from mixing this key with managed login/keychain auth.

**Still hitting Vertex or Bedrock**

Run with `--debug-file /tmp/glm-check.log`, then inspect the URL:

```bash
grep -E "api.z.ai|publishers/anthropic|bedrock-runtime" /tmp/glm-check.log
```

If you see Vertex or Bedrock, re-check the settings JSON and make sure you launched with both `--bare` and `--settings`.

**Do not put these exports in `.zshrc`**

Keep the provider switch in the settings file. Global exports make it too easy to accidentally route normal Claude Code sessions away from your preferred backend.

---

## Revert / uninstall

```bash
rm -rf ~/.config/mg-glm
```

Nothing else is required because this guide does not edit your global Claude Code settings.

---

## Resources

- Z.ai docs: https://docs.z.ai
- Z.ai Claude Code guide: https://docs.z.ai/devpack/tool/claude
- Z.ai GLM-5.2 guide: https://docs.z.ai/guides/llm/glm-5.2
- Claude Code docs: https://docs.anthropic.com/en/docs/claude-code
