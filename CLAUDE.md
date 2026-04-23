# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This repository documents setup and troubleshooting for running Claude Code with local LLMs via Ollama on macOS. It serves as a single-user reference guide tested empirically against live installations.

All configurations and commands have been verified against specific hardware/software versions (M3 Pro 36 GB, Ollama 0.21.0, Claude Code 2.1.114) and should be updated when versions change.

## Repository Structure

- **ollama-coding-guide-v3.md** — Primary comprehensive guide
  - Model stack and selection (GLM-4.7-flash, qwen3-coder, qwen2.5-coder)
  - Environment variables and Modelfile configuration
  - Claude Code integration (direct Ollama vs LiteLLM proxy)
  - Validation tests for tool-calling (three-tier smoke test)
  - Daily startup script (`cl-start.sh`) with comprehensive documentation
  - GPU placement verification, throughput measurement
  - Common errors and gotchas (Errors 1–13 with specific fixes)

- **claude-code-ollama-troubleshooting-2026-04-19.md** — Diagnostic reference
  - Root cause analysis of tool-calling failures with local models
  - What Modelfile changes can/cannot fix
  - Two working solutions (direct GLM vs LiteLLM proxy)
  - Validation test methodology and results
  - When to use which alias for different scenarios
  - Open follow-ups for future updates

## Key Concepts & Gotchas

### Tool-Calling Failure Modes

Local model tool-calling is fragile due to three compounding issues:

1. **Model output format mismatches** — Different models emit different tool-call formats (raw JSON vs Claude XML). Ollama's parsers expect specific wrapper tags.
2. **Behavioral switch at scale** — qwen3-coder switches from JSON to malformed XML when given 10+ tools. Claude Code always ships 10+ tools, always triggering this.
3. **Ollama Anthropic shim bugs** — Multiple open issues in Ollama's `/v1/messages` endpoint state machine (renderer broken, contentIndex collision, truncation errors).

**Current working paths:**
- glm-4.7-flash → Ollama direct ✅ (parser wired correctly, model trained for agentic workflows)
- qwen3-coder → LiteLLM proxy ✅ (LiteLLM bypasses Ollama's buggy Anthropic shim)
- qwen3-coder → Ollama direct ❌ (emits malformed XML with 10+ tools)

### Critical Configuration Details

**LiteLLM integration specifics:**

- Use `ollama_chat/` prefix, not `ollama/` — only `ollama_chat` uses `/api/chat` with tool support
- Set `drop_params: true` — Claude Code passes `context_management` parameter that Ollama rejects
- Include haiku alias in config — Claude Code makes background preflight requests; without alias, logs 400 errors

**Startup script (`cl-start.sh`) critical behavior:**

- Stops LiteLLM **before** Ollama to avoid CLOSE_WAIT false positives on port check
- Uses `-sTCP:LISTEN` flag in lsof to ignore outbound connections (Error 13)
- Pre-loads primary model to trigger Metal shader compile once instead of on first request
- Respects `PRIMARY_MODEL`, `START_LITELLM`, `LITELLM_CONFIG`, `LITELLM_PORT` env vars for override

### When to Update Guides

- **Ollama version changes** — Verify shim bugs fixed, re-run three-tier validation tests
- **Claude Code version changes** — Confirm tool-calling still works, update version in hardware context
- **Model registry changes** — If qwen3-coder or glm-4.7-flash models removed or renamed, update pull commands
- **Python/LiteLLM updates** — Test with new Python version, confirm LiteLLM still runs correctly

Always test changes empirically before documenting. Run the three-tier validation test (Read, Random Detector, Chain) after any significant config changes.

## Validation Test (Three-Tier Smoke Test)

Used to verify that any local setup actually works end-to-end:

1. **Read test** — `Read package.json and tell me the name field.` Checks single tool call, verifiable against real file.
2. **Random detector** — `Run: echo $((RANDOM * RANDOM)) and report the exact number.` Impossible to hallucinate correctly; detects real vs fake tool execution.
3. **Chain test** — `Create /tmp/claude-test-$(date +%s).txt with content "works", then ls -la /tmp/claude-test-*.txt`. Verifies multi-step round-trip; filesystem evidence persists.

All three must pass. If any fails, tool calling is broken — do not trust that setup.

**Success indicators:**
- Visible `⏺ Tool(args)` invocation followed by `⎿` result block
- Real numbers/file listing (not hallucinated)
- No raw JSON/XML blocks in output

## Common Maintenance Tasks

### Checking if a setup still works after updates

```bash
# Pull latest model
ollama pull glm-4.7-flash

# Run three-tier test
cl-glm
# Then in session: (1) Read test (2) Random detector (3) Chain test
```

### Upgrading Ollama/Claude Code versions

1. Update version number in "Hardware context" section
2. Test both direct and LiteLLM-proxy paths
3. Run three-tier validation on each path
4. Update any Error sections if behavior changed
5. Update README.md "Last updated" date

### Troubleshooting tool-calling regression

1. Confirm you're running on the right path (direct Ollama vs LiteLLM proxy)
2. Run three-tier validation test to isolate which step fails
3. Check Error sections for matching symptoms
4. If Ollama-related, consult Ollama issue tracker; if LiteLLM, check LiteLLM logs
5. Document findings in troubleshooting guide and update Error section

## Hardware Context (for updates)

Current verified setup:
- **Chip:** Apple M3 Pro
- **RAM:** 36 GB unified memory
- **Bandwidth:** 150 GB/s (hard ceiling on decode speed)
- **macOS:** Tahoe 26.4
- **Ollama:** 0.21.0 (Homebrew, native arm64)
- **Claude Code:** 2.1.114
- **Verified date:** April 20, 2026

When Claude Code updates, test against new version and update version number above.
When Ollama updates, re-verify GPU placement, throughput, and three-tier test.
