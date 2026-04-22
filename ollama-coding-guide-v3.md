# Ollama coding guide — M3 Pro 36 GB (verified)

> Stack: Ollama 0.21.0 · Claude Code 2.1.114 · Gemini CLI · LiteLLM 1.83.9
> Target: mid-size TypeScript / Kotlin projects
> Last verified: April 20 2026, all flags tested empirically against live install.
> v3 changes: `cl-start.sh` now stops LiteLLM before Ollama (prevents CLOSE_WAIT false positive on :11434); port check uses `-sTCP:LISTEN` to ignore outbound connections. See Error 13.

---

## ⚠️ Tool-calling status (read first)

Claude Code with local models requires working tool calling (Read, Bash, Edit, etc.). As of 2026-04-19, only two paths work:

| Model | Path | Works? | Why |
|---|---|---|---|
| **glm-4.7-flash** | Ollama direct (`ollama launch claude`) | ✅ | Parser wired correctly in Ollama, model trained for agentic workflows |
| **qwen3-coder** | **LiteLLM proxy** → Ollama | ✅ | LiteLLM bypasses buggy Ollama Anthropic shim |
| **glm-4.7-flash** | LiteLLM proxy → Ollama | ✅ | Same as above, works as fallback |
| qwen2.5-coder:7b | LiteLLM proxy → Ollama | ⚠️ untested | Included in config for experimentation; run validation tests before relying on it |
| qwen3-coder | Ollama direct | ❌ | Model switches to malformed XML when given 10+ tools; Claude Code's toolset triggers this |
| qwen2.5-coder:7b | Ollama direct | ❌ | Emits raw JSON without wrapper tags; Ollama parser finds nothing |
| qwen2.5-coder + RENDERER qwen3-coder | Ollama direct | ❌ | Model can't faithfully reproduce a format it wasn't trained on |

**Rule of thumb:**
- Want simplest setup → use `glm-4.7-flash` direct (Solution 4a).
- Want qwen3-coder's stronger code quality → use LiteLLM proxy (Solution 4b).
- Any qwen model through Ollama direct → will fail, don't waste time.

See troubleshooting doc for the full investigation.

---

## Hardware context

| | |
|---|---|
| Chip | Apple M3 Pro |
| RAM | 36 GB unified memory |
| Bandwidth | 150 GB/s (hard ceiling on decode speed) |
| macOS | Tahoe 26.4.1 |
| Ollama | 0.21.0 (Homebrew, native arm64) |
| Claude Code | 2.1.114 |

---

## 1. Model stack

**Primary: `glm-4.7-flash`** — 30B MoE, ~3B active per token, ~20 GB resident. Trained for agentic workflows, works reliably with Claude Code through Ollama direct.

**Secondary: `qwen3-coder:30b-a3b-q4_K_M`** — Stronger on pure coding benchmarks, but only works through LiteLLM proxy. Same memory footprint (~20 GB).

**Tertiary: `qwen2.5-coder:7b`** — Small and fast (~4.7 GB, ~25 tok/s). Two use cases:

- **Plain chat**: `ollama run qwen2.5-coder:7b` — paste code, ask for refactor/explain/tests, no agentic loop. Reliable.
- **Via LiteLLM proxy with Claude Code**: untested but worth trying when RAM is tight. May work (LiteLLM has its own tool-call extraction) or may fail same as direct path. Run the three-tier validation test (Section 5) before relying on it.

Tool calling does NOT work with Ollama direct for this model — confirmed.

```bash
# Primary — works out of the box with Claude Code
ollama pull glm-4.7-flash

# Secondary — stronger code quality, requires LiteLLM proxy
ollama pull qwen3-coder:30b-a3b-q4_K_M

# Tertiary — small/fast, plain chat only (no Claude Code)
ollama pull qwen2.5-coder:7b

# Embeddings — codebase search (optional)
ollama pull nomic-embed-text      # 274 MB
```

Usage for the tertiary model:

```bash
# Reliable: direct chat session, no Claude Code
ollama run qwen2.5-coder:7b

# Or one-shot query for scripting
echo "explain this regex: ^[a-z0-9]+$" | ollama run qwen2.5-coder:7b

# Experimental: Claude Code via LiteLLM (see Section 4b for alias and validation)
cl-lm-fast
```

---

## 2. Environment variables

```bash
# ~/.zshrc
export OLLAMA_FLASH_ATTENTION=1
export OLLAMA_KV_CACHE_TYPE=q8_0   # halves KV RAM vs f16, near-lossless
export OLLAMA_KEEP_ALIVE=-1        # model stays resident until manual stop
export OLLAMA_NUM_PARALLEL=1       # single-user: 1 saves RAM
export OLLAMA_MAX_LOADED_MODELS=1  # single-model workflow on 36 GB
```

**Do NOT set `OLLAMA_NUM_GPU`.** Misleading name — controls how many *layers* offload to GPU, not whether GPU is used. `=1` forces ~47 of 48 layers onto CPU, causing <5 tok/s. Ollama auto-detects correctly when unset.

**Do NOT set `OLLAMA_MMAP`.** Not a real Ollama env var. mmap is already default in the underlying llama.cpp runtime.

### Homebrew services

```bash
brew services stop ollama
brew services restart ollama
brew services list | grep ollama
```

---

## 3. Modelfiles

For GLM-4.7-Flash and qwen3-coder pulled from Ollama registry, **custom Modelfiles are not needed** — the registry versions come with correct RENDERER/PARSER wiring. Use them as-is.

Custom Modelfiles are only useful for parameter tuning (temperature, context). Adding custom `SYSTEM` prompts can break tool-call rendering by overriding the RENDERER-injected instructions.

---

## 4. Claude Code integration

Two working patterns. Pick based on which model you want.

### 4a. GLM-4.7-Flash direct (simple)

```bash
# ~/.zshrc
alias cl-glm='ollama launch claude --yes --model glm-4.7-flash -- --bare'
```

That's it. Single alias, no proxy, no config. Works because Ollama's built-in glm-4.7 parser handles this model correctly.

### 4b. qwen3-coder via LiteLLM (flexible, stronger code quality)

Requires Python 3.10+. macOS system Python 3.9 is too old — install a newer one:

```bash
brew install python@3.12
pipx install 'litellm[proxy]' --python python3.12
```

Config file at `~/.litellm/config.yaml`:

```yaml
model_list:
  - model_name: qwen3-coder
    litellm_params:
      model: ollama_chat/qwen3-coder
      api_base: http://localhost:11434

  - model_name: glm-4.7-flash
    litellm_params:
      model: ollama_chat/glm-4.7-flash
      api_base: http://localhost:11434

  # Small/fast option — tool-calling viability untested, run validation before relying on it
  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: ollama_chat/qwen2.5-coder:7b
      api_base: http://localhost:11434

  # Silences Claude Code's background haiku preflight requests
  - model_name: claude-haiku-4-5-20251001
    litellm_params:
      model: ollama_chat/qwen3-coder
      api_base: http://localhost:11434

litellm_settings:
  drop_params: true
```

**Three critical details** (each cost hours of debugging for someone):

1. **`ollama_chat/` not `ollama/`** — `ollama_chat` uses `/api/chat` with native tool support. `ollama` prefix uses legacy `/api/generate` which has no tool calling.
2. **`drop_params: true`** — Claude Code passes `context_management` parameter. Ollama rejects it. Without this flag, every request returns 400.
3. **Haiku alias** — Claude Code makes background preflight requests to a haiku model for token counting. Without an alias in config it logs 400 errors (harmless but noisy).

Lifecycle aliases:

```bash
# ~/.zshrc
alias lm-start='nohup litellm --config ~/.litellm/config.yaml --host 127.0.0.1 --port 4000 > /tmp/litellm.log 2>&1 & echo $! > /tmp/litellm.pid && sleep 2 && echo "✅ LiteLLM started (PID $(cat /tmp/litellm.pid))"'
alias lm-stop='kill $(cat /tmp/litellm.pid) 2>/dev/null && rm -f /tmp/litellm.pid && echo "✅ LiteLLM stopped"'
alias lm-log='tail -f /tmp/litellm.log'
alias lm-status='if [ -f /tmp/litellm.pid ] && kill -0 $(cat /tmp/litellm.pid) 2>/dev/null; then echo "✅ Running (PID $(cat /tmp/litellm.pid))"; else echo "❌ Not running"; fi'
```

Claude Code aliases through LiteLLM:

```bash
alias cl-lm='ANTHROPIC_AUTH_TOKEN=any \
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL=http://127.0.0.1:4000 \
  claude --bare --strict-mcp-config --mcp-config "{\"mcpServers\":{}}" --model qwen3-coder'

alias cl-lm-glm='ANTHROPIC_AUTH_TOKEN=any \
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL=http://127.0.0.1:4000 \
  claude --bare --strict-mcp-config --mcp-config "{\"mcpServers\":{}}" --model glm-4.7-flash'

# Experimental — small/fast model, tool-calling viability untested
alias cl-lm-fast='ANTHROPIC_AUTH_TOKEN=any \
  ANTHROPIC_API_KEY="" \
  ANTHROPIC_BASE_URL=http://127.0.0.1:4000 \
  claude --bare --strict-mcp-config --mcp-config "{\"mcpServers\":{}}" --model qwen2.5-coder-7b'
```

Usage:

```bash
lm-start            # once per session (or launchd for auto-start)
cl-lm               # Claude Code with qwen3-coder
cl-lm-glm           # same proxy, GLM model — fallback if qwen misbehaves
cl-lm-fast          # experimental, small/fast qwen2.5-coder:7b — run validation first
```

### Verified flags (Claude Code 2.1.114)

| Flag | Meaning |
|---|---|
| `--bare` | Skip hooks, LSP, plugins, auto-memory, keychain, CLAUDE.md auto-discovery. Sets `CLAUDE_CODE_SIMPLE=1`. |
| `--strict-mcp-config` | Only use MCP servers from `--mcp-config`, ignore all others |
| `--mcp-config <configs...>` | Load MCP servers from JSON files or strings |
| `--model <name>` | Model alias or full name |
| `--yes` (on `ollama launch`) | Auto-pull model, skip picker, requires `--model` |

`--no-mcp` does not exist. Use `--strict-mcp-config --mcp-config '{"mcpServers":{}}'` or `--bare`.

---

## 5. Validation — does tool calling actually work?

After any setup change, run this three-tier smoke test:

1. **Read test**: `Read package.json and tell me the "name" field.` Expected: `⏺ Read(package.json)` with `⎿` result block, then correct name.
2. **Random detector**: `Run: echo $((RANDOM * RANDOM)) and report the exact number.` Expected: visible Bash invocation, real number. If number appears without visible tool call — hallucination, not a real tool call.
3. **Chain test**: `Create /tmp/claude-test-$(date +%s).txt with content "works", then ls -la /tmp/claude-test-*.txt`. After exit, verify with `ls /tmp/claude-test-*.txt` — file must exist.

If all three pass, tool calling works end-to-end.

---

## 6. Daily startup

The startup sequence is provided as a bash script with progress output: `~/bin/cl-start.sh`. Save it, make executable, and optionally alias it.

### The script

```bash
#!/usr/bin/env bash
# ~/bin/cl-start.sh — daily Claude Code local stack startup
# Stops any running Ollama, starts it clean with tuned env, preloads the primary
# model, verifies GPU placement, and (optionally) starts LiteLLM proxy.

set -uo pipefail

# ── Config ────────────────────────────────────────────────────────────────────
PRIMARY_MODEL="${PRIMARY_MODEL:-glm-4.7-flash}"
START_LITELLM="${START_LITELLM:-yes}"          # set to "no" to skip
LITELLM_CONFIG="${LITELLM_CONFIG:-$HOME/.litellm/config.yaml}"
LITELLM_PORT="${LITELLM_PORT:-4000}"

# ── Colors & helpers ──────────────────────────────────────────────────────────
if [[ -t 1 ]]; then
  RED=$'\033[0;31m'; GREEN=$'\033[0;32m'; YELLOW=$'\033[0;33m'
  BLUE=$'\033[0;34m'; BOLD=$'\033[1m'; DIM=$'\033[2m'; RESET=$'\033[0m'
else
  RED=""; GREEN=""; YELLOW=""; BLUE=""; BOLD=""; DIM=""; RESET=""
fi

step()  { echo; echo "${BOLD}${BLUE}▶ [$1/$TOTAL] $2${RESET}"; }
ok()    { echo "${GREEN}  ✓ $1${RESET}"; }
warn()  { echo "${YELLOW}  ⚠ $1${RESET}"; }
err()   { echo "${RED}  ✗ $1${RESET}" >&2; }
info()  { echo "${DIM}  $1${RESET}"; }

TOTAL=5; [[ "$START_LITELLM" != "yes" ]] && TOTAL=4

# ── Step 1: Clean slate ───────────────────────────────────────────────────────
step 1 "Stopping any running LiteLLM + Ollama instances"

# Stop LiteLLM FIRST. If Ollama dies while LiteLLM still holds a keepalive to
# :11434, the socket lingers in CLOSE_WAIT and trips the port check below.
if [[ -f /tmp/litellm.pid ]] && kill -0 "$(cat /tmp/litellm.pid)" 2>/dev/null; then
  kill "$(cat /tmp/litellm.pid)" && ok "LiteLLM stopped (PID $(cat /tmp/litellm.pid))"
else
  LM_PID=$(lsof -ti :"$LITELLM_PORT" -sTCP:LISTEN 2>/dev/null | head -1)
  if [[ -n "$LM_PID" ]]; then
    kill "$LM_PID" && ok "LiteLLM stopped (PID $LM_PID, stale pidfile)"
  else
    info "LiteLLM not running"
  fi
fi
rm -f /tmp/litellm.pid

# Now stop Ollama.
brew services stop ollama >/dev/null 2>&1 && ok "brew service stopped" || info "brew service not running"
osascript -e 'tell application "Ollama" to quit' 2>/dev/null && ok "GUI app quit" || info "GUI app not running"

if pgrep -f "ollama serve" >/dev/null; then
  pkill -9 -f "ollama serve" && sleep 2 && ok "killed lingering processes"
else
  info "no stray processes"
fi

# Use -sTCP:LISTEN to match only bound sockets, not outbound CLOSE_WAIT/ESTABLISHED
# connections from clients that haven't yet noticed the server went away.
sleep 1
if lsof -iTCP:11434 -sTCP:LISTEN >/dev/null 2>&1; then
  err "port 11434 still bound — aborting"
  lsof -iTCP:11434 -sTCP:LISTEN | sed 's/^/    /'
  exit 1
fi
ok "port 11434 clean"

# ── Step 2: Start Ollama ──────────────────────────────────────────────────────
step 2 "Starting Ollama server"

OLLAMA_KEEP_ALIVE=-1 \
OLLAMA_FLASH_ATTENTION=1 \
OLLAMA_KV_CACHE_TYPE=q8_0 \
OLLAMA_MAX_LOADED_MODELS=1 \
OLLAMA_NUM_PARALLEL=1 \
  nohup ollama serve > /tmp/ollama.log 2>&1 &

# Wait for API to respond (up to 10s)
for i in {1..20}; do
  if curl -sf http://localhost:11434/api/version >/dev/null 2>&1; then break; fi
  sleep 0.5
done

if curl -sf http://localhost:11434/api/version >/dev/null 2>&1; then
  VERSION=$(curl -s http://localhost:11434/api/version | python3 -c "import sys,json;print(json.load(sys.stdin).get('version','?'))" 2>/dev/null)
  ok "Ollama up (v$VERSION)"
else
  err "Ollama failed to start — check /tmp/ollama.log"
  tail -5 /tmp/ollama.log | sed 's/^/    /'
  exit 1
fi

# ── Step 3: Preload primary model ─────────────────────────────────────────────
step 3 "Preloading $PRIMARY_MODEL (first run compiles Metal shaders, ~5–10s)"

if ! ollama list | awk 'NR>1 {print $1}' | grep -qx "$PRIMARY_MODEL\(:latest\)\?"; then
  err "model '$PRIMARY_MODEL' not found — run: ollama pull $PRIMARY_MODEL"
  exit 1
fi

START=$(date +%s)
ollama run "$PRIMARY_MODEL" "hi" >/dev/null 2>&1
ELAPSED=$(( $(date +%s) - START ))
ok "loaded in ${ELAPSED}s"

# ── Step 4: Verify GPU placement ──────────────────────────────────────────────
step 4 "Verifying GPU placement"

PS_OUTPUT=$(ollama ps)
echo "$PS_OUTPUT" | sed 's/^/    /'

if echo "$PS_OUTPUT" | grep -q "100% GPU"; then
  ok "100% GPU"
elif SPLIT=$(echo "$PS_OUTPUT" | grep -oE "[0-9]+%/[0-9]+% CPU/GPU" | head -1); then
  CPU_PCT=$(echo "$SPLIT" | grep -oE "^[0-9]+")
  if [[ "$CPU_PCT" -le 10 ]]; then
    ok "cosmetic CPU/GPU split (${CPU_PCT}% CPU) — acceptable on M-series"
  else
    warn "significant CPU spill (${CPU_PCT}% CPU) — throughput will suffer"
    warn "check: ollama show --modelfile $PRIMARY_MODEL | grep num_gpu (must be 999)"
  fi
else
  warn "could not parse GPU placement from 'ollama ps'"
fi

# ── Step 5: Start LiteLLM (optional) ──────────────────────────────────────────
if [[ "$START_LITELLM" == "yes" ]]; then
  step 5 "Starting LiteLLM proxy on :$LITELLM_PORT"

  if [[ -f /tmp/litellm.pid ]] && kill -0 "$(cat /tmp/litellm.pid)" 2>/dev/null; then
    ok "already running (PID $(cat /tmp/litellm.pid))"
  elif ! command -v litellm >/dev/null 2>&1; then
    warn "litellm not installed — skipping"
    info "install: brew install python@3.12 && pipx install 'litellm[proxy]' --python python3.12"
  elif [[ ! -f "$LITELLM_CONFIG" ]]; then
    warn "$LITELLM_CONFIG not found — skipping"
  else
    rm -f /tmp/litellm.pid
    nohup litellm --config "$LITELLM_CONFIG" --host 127.0.0.1 --port "$LITELLM_PORT" > /tmp/litellm.log 2>&1 &
    echo $! > /tmp/litellm.pid

    # Wait for proxy up (up to 15s)
    for i in {1..30}; do
      if curl -sf "http://127.0.0.1:$LITELLM_PORT/v1/models" >/dev/null 2>&1; then break; fi
      sleep 0.5
    done

    if curl -sf "http://127.0.0.1:$LITELLM_PORT/v1/models" >/dev/null 2>&1; then
      ok "up on :$LITELLM_PORT (PID $(cat /tmp/litellm.pid))"
    else
      err "LiteLLM failed to start — check /tmp/litellm.log"
      tail -5 /tmp/litellm.log | sed 's/^/    /'
    fi
  fi
fi

# ── Summary ───────────────────────────────────────────────────────────────────
echo
echo "${BOLD}${GREEN}✓ Ready${RESET}"
echo
info "Available sessions:"
info "  cl-glm       → Claude Code + GLM (Ollama direct)"
[[ "$START_LITELLM" == "yes" ]] && {
  info "  cl-lm        → Claude Code + qwen3-coder (LiteLLM)"
  info "  cl-lm-glm    → Claude Code + GLM (LiteLLM)"
  info "  cl-lm-fast   → Claude Code + qwen2.5-coder:7b (LiteLLM, experimental)"
}
info "Logs: /tmp/ollama.log  /tmp/litellm.log"
```

### Install and use

```bash
mkdir -p ~/bin
# Paste the script above into ~/bin/cl-start.sh
chmod +x ~/bin/cl-start.sh

# Make sure ~/bin is in PATH (usually is on zsh; add if needed):
# export PATH="$HOME/bin:$PATH"   # ~/.zshrc

# Run
cl-start.sh
```

### Lifecycle aliases

Thin wrappers on top of the script and for stopping individual components:

```bash
# ~/.zshrc

# Full startup (Ollama + preload + LiteLLM)
alias cl-start='~/bin/cl-start.sh'

# Just Ollama, no preload, no LiteLLM
alias cl-start-min='START_LITELLM=no PRIMARY_MODEL=glm-4.7-flash ~/bin/cl-start.sh'

# Stop everything — mirror of cl-start. Use this instead of pkill-ing Ollama
# directly; otherwise a running LiteLLM keeps its socket in CLOSE_WAIT and
# the next cl-start will abort at the port check.
alias cl-stop-all='lm-stop 2>/dev/null; ol-stop 2>/dev/null; echo "✅ All stopped"'

# Individual pieces (defined elsewhere in this guide)
alias ol-stop='pkill -9 -f "ollama serve" && echo "✅ Ollama stopped"'
alias cl-stop='ollama stop glm-4.7-flash 2>/dev/null; ollama stop qwen3-coder 2>/dev/null; echo "✅ Models unloaded"'
```

**Note:** `cl-start` itself now tears down any existing LiteLLM + Ollama in step 1, so running `cl-start` twice in a row is safe. `cl-stop-all` is for when you want to free the ~30 GB of RAM at end of day without restarting.

### Configuration via env vars

The script respects these overrides:

| Variable | Default | Purpose |
|---|---|---|
| `PRIMARY_MODEL` | `glm-4.7-flash` | Model to preload |
| `START_LITELLM` | `yes` | Set to `no` to skip LiteLLM step |
| `LITELLM_CONFIG` | `~/.litellm/config.yaml` | Path to LiteLLM config |
| `LITELLM_PORT` | `4000` | Port for LiteLLM proxy |

Examples:

```bash
# Start with qwen3-coder preloaded instead of glm
PRIMARY_MODEL=qwen3-coder cl-start

# Ollama only, skip LiteLLM
START_LITELLM=no cl-start

# Custom port
LITELLM_PORT=5000 cl-start
```

---

## 7. Memory budget

| Component | RAM |
|---|---|
| glm-4.7-flash or qwen3-coder (active) | ~20 GB |
| KV cache 32K ctx, q8_0 | ~1.5 GB |
| macOS + editor + Docker | ~8–10 GB |
| LiteLLM process (if used) | ~200 MB |
| **Active total** | **~30–32 GB** |
| **Headroom on 36 GB** | **~4–6 GB** |

Tight but workable. Cannot run heavy Docker stacks simultaneously. `ollama stop <model>` frees 20 GB instantly when needed.

---

## 8. Verify GPU placement

```bash
ollama ps
```

`PROCESSOR` should show `100% GPU` (or `4%/96% CPU/GPU` — 1 output layer on M-series is a known cosmetic quirk, see Ollama issue #9948, not a real spill). Any `CPU/GPU` split with >10% CPU means real layer spill — check `ollama show --modelfile <n> | grep num_gpu` (must be `999`).

---

## 9. Measure throughput (tok/s)

### Theory — why there's a hard ceiling

Decode on Apple Silicon is memory-bandwidth bound, not compute bound. Every generated token requires reading the model's active weights from RAM. The speed limit is simple:

```
max_tokens_per_second ≈ memory_bandwidth_GB_per_sec / active_model_size_GB
```

For **dense** models, active size = full model size.
For **MoE** models (qwen3-coder, glm-4.7-flash), active size = size × (active_params / total_params). A 30B MoE with 3B active "feels like" a 3B model for throughput, even though it holds 30B weights in RAM.

**M3 Pro 36 GB: 150 GB/s bandwidth.** Theoretical ceilings at Q4_K_M:

| Model | Total size | Active size | Max theoretical tok/s |
|---|---|---|---|
| qwen2.5-coder:7b (dense) | 4.7 GB | 4.7 GB | 150 / 4.7 ≈ **32 tok/s** |
| qwen2.5-coder:14b (dense) | 9 GB | 9 GB | 150 / 9 ≈ **16.6 tok/s** |
| qwen3-coder:30b-a3b | 19 GB | ~2 GB | 150 / 2 ≈ **75 tok/s** (MoE effective) |
| glm-4.7-flash | ~20 GB | ~2 GB | 150 / 2 ≈ **75 tok/s** (MoE effective) |

Real-world decoded tok/s is 60–95% of theoretical due to overhead (KV cache read, attention, sampling). If you measure <60% of theoretical, something is wrong (CPU spill, wrong quantization, thermal throttling).

### Measure — run this

First, ensure the model is preloaded (first request includes Metal shader compile, ~5–10 s):

```bash
ollama run glm-4.7-flash "warmup" >/dev/null
```

Then run three back-to-back requests and parse the response:

```bash
MODEL=glm-4.7-flash

for i in 1 2 3; do
  curl -s http://localhost:11434/api/generate \
    -d "{\"model\":\"$MODEL\",\"prompt\":\"Write a binary search function in Python.\",\"stream\":false}" \
  | python3 -c "
import sys, json
d = json.load(sys.stdin)
ttft_ms = d['prompt_eval_duration'] / 1e6
decode_tps = d['eval_count'] / (d['eval_duration'] / 1e9)
print(f'Run $i: TTFT {ttft_ms:.0f} ms, decode {decode_tps:.1f} tok/s, {d[\"eval_count\"]} tokens')
"
done
```

**What each number means:**

- `prompt_eval_duration` — time to process the input prompt (prefill). Scales with input length. Not throughput-relevant for code generation where prompts are small.
- `eval_duration` — time spent generating output tokens (decode). This is what you divide by `eval_count` to get tok/s.
- `eval_count` — number of tokens generated.
- First run is usually slower (Metal shader compile, cold caches). Use runs 2 and 3 as the stable measurement.

### Compare to ceiling

Expected on M3 Pro 36 GB (stable, warm):

| Model | Expected | % of theoretical |
|---|---|---|
| qwen2.5-coder:7b | ~25 tok/s | ~78% |
| glm-4.7-flash | ~40–50 tok/s | ~55–65% |
| qwen3-coder:30b-a3b | ~47 tok/s | ~63% |

MoE models hit lower % of theoretical because MoE routing adds overhead (different experts activate per token, cache locality is worse). Still vastly faster than the same-RAM dense alternative.

### Troubleshooting low throughput

If you measure significantly below expected:

1. **Check GPU placement**: `ollama ps` — must be 100% GPU (or 4%/96% cosmetic split).
2. **Check model quantization**: `ollama show --modelfile <n>` — Q4_K_M is the sweet spot. Q8_0 doubles size, halves throughput. F16 quarters throughput.
3. **Check KV cache type**: `echo $OLLAMA_KV_CACHE_TYPE` — should be `q8_0`. `f16` is fine but uses 2× RAM, can push model off GPU.
4. **Check other GPU load**: `ioreg -l | grep "GPU Core"` or just close other apps using Metal (video, browsers with WebGL).
5. **Check thermal throttling**: sustained long generations can throttle on Pro chips. `sudo powermetrics --samplers smc -i1000 -n1 | grep -i temp` shows current temps.

---

## 10. What went wrong in previous versions of this guide

### Error 1–6: Ollama config (num_gpu, env vars, etc.) — unchanged from prior revision

### Error 7: Claude Code integration — wrong env vars

- `ANTHROPIC_BASE_URL=http://localhost:11434/v1` — `/v1` is OpenAI-compat, not Anthropic Messages
- `ANTHROPIC_API_KEY=ollama` — triggers auth against api.anthropic.com
- Correct for direct Ollama: `ANTHROPIC_AUTH_TOKEN=ollama`, `ANTHROPIC_API_KEY=""`, `ANTHROPIC_BASE_URL=http://localhost:11434`
- Correct for LiteLLM: `ANTHROPIC_AUTH_TOKEN=any`, `ANTHROPIC_API_KEY=""`, `ANTHROPIC_BASE_URL=http://127.0.0.1:4000`

### Error 8: `--no-mcp` flag

Does not exist. Use `--strict-mcp-config --mcp-config '{"mcpServers":{}}'` or `--bare`.

### Error 9 (new): Recommending qwen3-coder as primary for Claude Code

Previous revision claimed qwen3-coder works with Claude Code through Ollama direct. Empirically it does not — the model emits malformed XML when Claude Code's 10+ tools are injected. Model behavior documented in [block/goose#6883](https://github.com/block/goose/issues/6883). Fix: either switch to glm-4.7-flash (which works direct) or add LiteLLM proxy (which works with any model).

### Error 10 (new): Recommending aliases without verifying tool calls

Previous revision tested only plain generation. Tool calling is the actual use case for Claude Code. Any future revision must run the three-tier validation test before claiming a setup works.

### Error 11 (new): LiteLLM on system Python 3.9

macOS ships Python 3.9 which lacks PEP 604 union types (`dict | None` syntax). Modern LiteLLM requires 3.10+. Install via `brew install python@3.12` + `pipx install 'litellm[proxy]' --python python3.12`.

### Error 12 (new): `ollama/` vs `ollama_chat/` LiteLLM prefix

`ollama/` uses `/api/generate` without tool support. `ollama_chat/` uses `/api/chat` with tools. Always use `ollama_chat/` for Claude Code integration.

### Error 13 (v3): `cl-start` aborts at port 11434 check because of LiteLLM CLOSE_WAIT

Symptom: running `cl-start` after an earlier session fails at step 1 with `port 11434 still in use`, and `lsof` shows a Python process (LiteLLM) with an outbound connection like `localhost:XXXXX->localhost:11434 (CLOSE_WAIT)` — not a listener.

Cause: `lsof -i :11434` matches **any** socket touching that port, including LiteLLM's outbound keepalive connection left dangling after Ollama was killed. The connection is in CLOSE_WAIT because the remote (dead Ollama) sent FIN but LiteLLM's side of the socket hasn't been closed yet.

Fix (applied in v3):

1. Stop LiteLLM **before** Ollama in step 1 so it closes its client socket cleanly.
2. Change the port check from `lsof -i :11434` to `lsof -iTCP:11434 -sTCP:LISTEN` — only matches sockets actually bound to the port, ignores client connections in any state.

Manual unblock if you hit this on the old script:

```bash
lm-stop      # or: kill <the python PID shown by lsof>
cl-start     # re-run
```
