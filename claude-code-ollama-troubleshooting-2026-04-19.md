# Claude Code + Ollama local models troubleshooting

**Date:** 2026-04-19
**Environment:** M3 Pro 36 GB, macOS Tahoe 26.4.1, Ollama 0.21.0, Claude Code 2.1.114
**Status:** ✅ Resolved

## Initial problem

Followed a guide to set up local models for Claude Code with three aliases: `cl-qwen`, `cl-deep`, `cl-fast` (all using `ollama launch claude` pattern). Models loaded and responded, but outputs were malformed — raw JSON/XML blocks instead of actual tool invocations. The `cl` alias (Sonnet 4.6 via Claude Enterprise) worked normally.

Observed symptom on `cl-fast` (qwen2.5-coder:7b):

```
⏺ {"name": "Bash", "arguments": {"command": "git log -1 --pretty=%B"}}
⏺ {"name": "Bash", "arguments": {"command": "git log -1 --pretty=%B"}}
```

Duplicate raw JSON instead of command execution.

## Root cause

Tool-call translation failure between Ollama's Anthropic-compat endpoint (`/v1/messages`) and Claude Code. Three compounding issues:

**1. Model output format mismatches built-in parsers.** Qwen3-Coder emits Claude-style XML (`<function=Bash><parameter=command>`), Qwen2.5-Coder-tools emits raw JSON. Ollama's built-in parsers (`qwen3-coder`, `glm-4.7`) expect specific wrapper tags. When the format doesn't match, the tool output passes through as plain text and Claude Code never sees a `tool_use` content block.

**2. Qwen3-Coder behavior switch at 5-6+ tools.** Documented in [block/goose#6883](https://github.com/block/goose/issues/6883): the model switches from native JSON tool_calls to XML format inside content when given many tools. Claude Code ships 10+ tools (Bash, Read, Edit, Write, Glob, Grep, Task, TodoWrite, WebSearch, WebFetch), always triggering the failure mode.

**3. Ollama Anthropic shim state machine bugs.** Multiple open issues — [#14493](https://github.com/ollama/ollama/issues/14493) (renderer broken for Qwen3.5), [#14816](https://github.com/ollama/ollama/issues/14816) (contentIndex collision on thinking+tool_use), [#14570](https://github.com/ollama/ollama/issues/14570) (500 on truncated tool JSON).

## What was tested

| Attempt | Model | Path | Result |
|---|---|---|---|
| Original aliases | qwen3-coder, qwen2.5-coder:7b, deep-coder | Ollama direct | ❌ Raw XML/JSON, no tool execution |
| Direct base tag, no custom Modelfile | qwen3-coder:30b-a3b-q4_K_M | Ollama direct | ❌ Same failure — not a Modelfile issue |
| SFT-tuned tool variant | hhao/qwen2.5-coder-tools:7b | Ollama direct | ❌ Clean JSON but no wrapper → parser finds nothing |
| Force qwen3-coder parser on 2.5 | qwen2.5-coder:7b + RENDERER/PARSER qwen3-coder | Ollama direct | ❌ Malformed XML, parser rejects, infinite retry |
| Switch to GLM | glm-4.7-flash | Ollama direct | ✅ Works end-to-end |
| LiteLLM proxy | qwen3-coder | LiteLLM → Ollama | ✅ Works end-to-end |
| LiteLLM proxy | glm-4.7-flash | LiteLLM → Ollama | ✅ Works end-to-end |

## What Modelfile changes can and cannot fix

**Cannot fix:**

- Model behavior with 10+ tools (requires retraining)
- Ollama shim state machine bugs (Ollama code)
- Missing `input_json_delta` events in SSE stream (protocol-level)

**Can help marginally:**

- Removing custom `SYSTEM` prompts that conflict with RENDERER-injected instructions
- Matching HuggingFace chat template exactly
- Adding correct `RENDERER`/`PARSER` directives when a model-specific parser exists

Modelfile tuning is cosmetic on top of a fundamentally unstable integration. The real fix is either switching models (GLM) or changing the translation layer (LiteLLM).

## The two working solutions

### Solution A — GLM-4.7-Flash direct (simple)

Works because Ollama's `glm-4.7` parser is correctly wired and the model is trained for agentic workflows.

```bash
ollama pull glm-4.7-flash
alias cl-glm='ollama launch claude --yes --model glm-4.7-flash -- --bare'
```

Pros: no extra process, one-liner setup.
Cons: locked to GLM; Qwen3-Coder's stronger code quality unavailable.

### Solution B — LiteLLM proxy (flexible)

LiteLLM implements its own Anthropic ↔ OpenAI translation layer that bypasses Ollama's buggy Anthropic shim. Works with qwen3-coder, glm-4.7-flash, or any other Ollama model.

Setup (Python 3.10+ required — Apple system Python 3.9 won't work):

```bash
brew install python@3.12
pipx install 'litellm[proxy]' --python python3.12

# Config file at ~/.litellm/config.yaml
cat > ~/.litellm/config.yaml << 'EOF'
model_list:
  - model_name: qwen3-coder
    litellm_params:
      model: ollama_chat/qwen3-coder
      api_base: http://localhost:11434

  - model_name: glm-4.7-flash
    litellm_params:
      model: ollama_chat/glm-4.7-flash
      api_base: http://localhost:11434

  # Silences Claude Code's background haiku preflight requests
  - model_name: claude-haiku-4-5-20251001
    litellm_params:
      model: ollama_chat/qwen3-coder
      api_base: http://localhost:11434

litellm_settings:
  drop_params: true
EOF
```

Critical details:

- **`ollama_chat/` not `ollama/`** — `ollama_chat` uses `/api/chat` with native tool support; `ollama` uses legacy `/api/generate` without tools.
- **`drop_params: true`** — Claude Code passes `context_management` parameter that Ollama doesn't support; without this, every request errors out.
- **Haiku alias** — Claude Code makes background preflight requests to a haiku model for token counting. Without an alias it returns 400 errors (harmless but noisy in logs).

Process management:

```bash
# ~/.zshrc
alias lm-start='nohup litellm --config ~/.litellm/config.yaml --host 127.0.0.1 --port 4000 > /tmp/litellm.log 2>&1 & echo $! > /tmp/litellm.pid && sleep 2 && echo "✅ LiteLLM started (PID $(cat /tmp/litellm.pid))"'
alias lm-stop='kill $(cat /tmp/litellm.pid) 2>/dev/null && rm -f /tmp/litellm.pid && echo "✅ LiteLLM stopped"'
alias lm-log='tail -f /tmp/litellm.log'
alias lm-status='if [ -f /tmp/litellm.pid ] && kill -0 $(cat /tmp/litellm.pid) 2>/dev/null; then echo "✅ Running (PID $(cat /tmp/litellm.pid))"; else echo "❌ Not running"; fi'
```

## Key diagnostic signals

Working tool call:

```
⏺ Bash(git log -1 --pretty=%B)
  ⎿ Migrate to new version of platform api
```

Broken:

- Raw JSON/XML blocks in output
- Missing `⎿` result block
- Model hallucinating plausible-looking answers without invoking the tool
- Duplicate tool-call blocks (model didn't get `tool_result` back, retried)

## Validation tests

Three-tier smoke test to verify any local setup:

1. **Read test**: "Read package.json and tell me the name field" — single tool call, verifiable against real file
2. **Random detector**: "Run: `echo $((RANDOM * RANDOM))` and report the number" — impossible to hallucinate correctly
3. **Chain test**: "Create `/tmp/claude-test-$(date +%s).txt` with 'hello', then ls it" — multi-step round-trip, leaves verifiable filesystem evidence

If all three pass, tool calling works end-to-end. Verified for: `cl-glm`, `cl-lm` (qwen3-coder), `cl-lm-glm`.

## Final configuration

```bash
# Cloud — primary workhorse
alias cl='claude'
alias cl-q='claude --strict-mcp-config --mcp-config "{}"'

# Local — direct Ollama (simple, GLM only)
alias cl-glm='ollama launch claude --yes --model glm-4.7-flash -- --bare'

# Local — via LiteLLM proxy (flexible, better code quality with qwen3-coder)
alias cl-lm='ANTHROPIC_AUTH_TOKEN=any \
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL=http://127.0.0.1:4000 \
  claude --bare --strict-mcp-config --mcp-config "{\"mcpServers\":{}}" --model qwen3-coder'

alias cl-lm-glm='ANTHROPIC_AUTH_TOKEN=any \
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL=http://127.0.0.1:4000 \
  claude --bare --strict-mcp-config --mcp-config "{\"mcpServers\":{}}" --model glm-4.7-flash'

# LiteLLM lifecycle (see Solution B for full definitions)
alias lm-start='...'
alias lm-stop='...'
alias lm-log='...'
alias lm-status='...'
```

Cleanup of failed experiments:

```bash
ollama rm fast-coder deep-coder fast-coder-tools fast-coder-parser-test
# Keep: qwen3-coder:30b-a3b-q4_K_M (used via LiteLLM), glm-4.7-flash
```

## When to use which

| Alias | Use case | Notes |
|---|---|---|
| `cl` | Daily work, real tasks | Sonnet, fastest, highest quality |
| `cl-glm` | Offline/privacy, quick local | No extra process, simplest |
| `cl-lm` | Offline + want best local code quality | Needs LiteLLM running; qwen3-coder > glm on pure coding |
| `cl-lm-glm` | Fallback if qwen3-coder misbehaves with many tools | Same proxy, different model |

Reference: identical codebase investigation task — **1m 13s cloud Sonnet vs 1h 22m local, ~68× slower** ([paddo.dev](https://paddo.dev/blog/claude-code-local-ollama/)). Local is for offline/privacy, not daily productivity.

## Open follow-ups

- `brew upgrade ollama` to pull recent state-machine fixes (not yet done this session)
- Update CLAUDE.md / `.docs/guides/` with findings
- Record in MemPalace project wing to avoid repeating this diagnostic cycle
- Back to the real work: Clara contact rate collapse investigation
