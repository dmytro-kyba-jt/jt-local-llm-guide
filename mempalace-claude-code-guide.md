# MemPalace + Claude Code: Install & Usage Guide

Based on the actual pitfalls we hit setting this up on macOS ARM64. Everything here is a lesson learned through a crash, a misfire, or a wrong-turn recommendation that got corrected.

---

## TL;DR — The Canonical Setup

```bash
# 1. Install mempalace under Python 3.12 via pipx
pyenv install 3.12 && pyenv global 3.12
pipx install --python "$(pyenv which python)" mempalace

# 2. Pin ChromaDB <1 to dodge the ARM64 segfault
pipx inject --force mempalace 'chromadb<1'

# 3. Create the palace
mempalace init --yes ~/my-palace

# 4. Alias it so you don't repeat --palace every time
echo 'alias mempalace="mempalace --palace \"$HOME/my-palace\""' >> ~/.zshrc
exec zsh

# 5. Register the MCP server at USER scope
claude mcp add mempalace -s user -- \
  ~/.local/pipx/venvs/mempalace/bin/python -m mempalace.mcp_server

# 6. Add write-path hooks at USER scope (see §4)
# 7. Add the Memory rule to ~/.claude/CLAUDE.md (see §5)
# 8. Backfill the palace with mempalace mine (see §6)
```

Everything below is the why and the how-to-verify.

---

## 1. Prerequisites

- **macOS ARM64** (Apple Silicon) — the guide assumes this; ChromaDB's prebuilt wheels behave differently on other platforms.
- **pyenv** installed (`brew install pyenv`). Pyenv becomes the single source of truth for Python so pipx venvs don't drift.
- **pipx** installed (`brew install pipx && pipx ensurepath`).
- **Claude Code** installed and working (`claude --version`).

### Why Python 3.12 specifically

- **Python 3.14** — `ensurepip` dies with `SIGKILL` on macOS; `venv` creation fails. Too early.
- **Python 3.13** — works, but `chromadb<1` needs to compile `hnswlib` from source on ARM64 because prebuilt wheels don't exist for 3.13+. You'll hit C++ build errors (`iostream file not found`).
- **Python 3.12** — has prebuilt `hnswlib` wheels, no compilation needed. This is the sweet spot.

Set it as the pyenv global and make sure `which python3.12` resolves to the pyenv shim before proceeding.

---

## 2. Installation

### 2.1 Install mempalace under Python 3.12

```bash
# Explicit Python pin — don't let pipx pick whatever is default
pipx install --python "$(pyenv which python)" mempalace
```

Verify:

```bash
pipx list
# should show: package mempalace X.Y.Z, installed using Python 3.12.x
```

### 2.2 Downgrade ChromaDB to <1 (the critical step)

ChromaDB `1.5.7` ships a Rust binding (`chromadb_rust_bindings.abi3.so`) with a null-pointer-plus-offset bug in HNSW index traversal. On macOS ARM64 it segfaults the moment you run `mempalace wake-up` or any search:

```
KERN_INVALID_ADDRESS at 0x88
```

Fix by downgrading to the pre-1.x line inside the mempalace pipx venv:

```bash
pipx inject --force mempalace 'chromadb<1'
```

You'll get `chromadb 0.6.3`. The `--force` flag is mandatory — pipx otherwise sees chromadb is already present (as a transitive dep) and refuses to modify it.

Verify the downgrade stuck:

```bash
MP="$HOME/.local/bin/mempalace"
PY=$(head -1 "$MP" | sed 's|#!||')
"$PY" -m pip show chromadb | grep -E 'Name|Version'
# expect: Version: 0.6.3
```

### 2.3 The "upgrade will bite you" warning

If you ever run `pipx upgrade mempalace` or `pipx upgrade-all`, pipx may pull chromadb back to 1.x because mempalace's metadata allows it. The segfault returns. Fix:

```bash
pipx inject --force mempalace 'chromadb<1'
```

Consider pinning tighter to make intent explicit:

```bash
pipx inject --force mempalace 'chromadb==0.6.3'
```

> **0.x → 1.x is an on-disk format break.** If you ever upgrade chromadb past 1.0, the existing palace is unreadable. You'd have to `rm -rf ~/my-palace && mempalace init ~/my-palace && mempalace mine ...` to rebuild.

---

## 3. Palace Initialization

```bash
mempalace init --yes ~/my-palace
```

> Subcommands require the palace directory as a positional argument, even when you also pass `--palace`. The `init` command won't accept just the alias — it needs the `dir` positional.

### Set up the alias properly

Put this in `~/.zshrc`:

```bash
alias mempalace='mempalace --palace "$HOME/my-palace"'
```

Now `mempalace status`, `mempalace mine <dir>`, etc. all use your palace without repeating the flag. The `init` case still takes a dir argument — that's one-time, not worth fighting.

---

## 4. Claude Code Integration — The Right Wiring

The integration has two paths: **write** (Claude saves to palace) and **read** (Claude queries palace). They use different mechanisms. Getting them crossed is the #1 reason things "don't work."

### 4.1 Write path — Hooks (at user scope)

Two hooks only. Put them in **`~/.claude/settings.json`** (user scope — not project scope, unless you want to inflict mempalace on teammates):

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "mempalace hook run --hook stop --harness claude-code"
          }
        ]
      }
    ],
    "PreCompact": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "mempalace hook run --hook precompact --harness claude-code"
          }
        ]
      }
    ]
  }
}
```

- **`Stop`** fires when a conversation ends — saves the session to the palace.
- **`PreCompact`** fires before Claude Code compresses context — saves before memory is lost.
- **No `SessionStart` hook.** This is the trap. `mempalace wake-up` is a CLI command intended for local models (Qwen, etc.) that don't speak MCP. It outputs human-readable text which Claude Code logs as *"Hook output does not start with `{`, treating as plain text"* — it does not change Claude's tool-selection behavior. Remove any `SessionStart: mempalace wake-up` you have committed.

### 4.2 Read path — MCP server (at user scope)

```bash
claude mcp add mempalace -s user -- \
  ~/.local/pipx/venvs/mempalace/bin/python -m mempalace.mcp_server
```

The `-s user` is critical. If you register it at local/project scope, your global `~/.claude/CLAUDE.md` Memory rule will reference `mempalace_status` in every project — but the tool only exists in the project where you registered it. You get confusing "unknown tool" failures everywhere else.

Verify:

```bash
claude mcp get mempalace
# Scope: User config (visible across all your projects)
```

---

## 5. The CLAUDE.md Memory Rule

MCP tools sit dormant unless Claude is told when to call them. This is the read-path trigger. Add to **`~/.claude/CLAUDE.md`** (user scope):

```markdown
## Memory (MemPalace)

- Before answering about past work, decisions, or people: call `mempalace_search`.
  If first call of session, also call `mempalace_status` first to load the protocol.
- On "remember this" or similar: call `mempalace_add_drawer`
  (and `mempalace_kg_add` for discrete facts).
```

### Heading level matters

Use `##`, **not** `###`. An `###`-level Memory rule gets parsed as preamble under whatever `##` section came before it, and Claude often ignores it. Promote it to a top-level `##` so it stands as its own instruction block.

### Why "first call of session" wording

An unconditional `mempalace_status` on every recall query burns ~170 tokens per call. Gating it on "first call of session" means you pay once per session, not per query. The `mempalace_search` still fires every time — that's where the value is.

---

## 6. Backfilling the Palace — `mempalace mine`

An empty palace is the silent killer. The wiring works, tools fire, but searches return near-empty because there are only ~2 drawers. It looks broken; it's actually just empty.

### Mine both the project and the existing Claude Code convo history

```bash
# 1. Index project source/docs
mempalace mine ~/Projects/my-project --mode projects --wing my-project

# 2. Index existing Claude Code conversation memory (the goldmine)
#    The directory is a slugified version of your project path.
#    Find it with: ls ~/.claude/projects/ | grep <your-project>
mempalace mine \
  ~/.claude/projects/<slugified-project-path>/ \
  --mode convos --wing my-project

# 3. Verify drawer count jumped
mempalace status
```

The `.claude/projects/<path-slug>/memory/` directory is Claude Code's native conversation store. It often has 10–30 files of real historical context (decisions, past investigations, PR writeups). Mining it into the palace turns "I don't remember" into immediately useful search results.

### There is no file watcher

`mempalace mine` is a pull, not a subscription. When project docs change, you re-run `mempalace mine`. The `Stop`/`PreCompact` hooks only save *conversation* memories, not source files.

The community-recommended alternative is: let Claude create and update drawers incrementally during sessions via `mempalace_add_drawer`, rather than bulk re-mining. Works well once the palace has momentum.

### Per-wing, per-project

The `--wing` flag isolates palaces per project. Drawers from one project don't pollute searches in another. Use one wing per project; name it the same as the repo for mental mapping.

---

## 7. Verification — End-to-End Smoke Test

### Read path

```bash
# 1. MCP is registered at user scope
claude mcp list | grep mempalace

# 2. Start a session, then in the TUI:
#    /mcp    — mempalace should show `connected`, tool count > 0
#    /context — MCP tools listed under "loaded on-demand"

# 3. Palace has content
mempalace status   # drawer count per wing should be > ~10 for any useful project

# 4. Force a recall-shaped query with Ctrl+O verbose ON:
#    > "What did we decide about <specific past topic>?"
#    Expect: mempalace_status (first call) → mempalace_search → answer
```

If tools never fire on a recall query, the read path is broken. Check in order:

1. Heading level in `~/.claude/CLAUDE.md` (must be `##`).
2. MCP server scope (must be user, not local).
3. `claude --debug 2>&1 | tee /tmp/cc.log` then `grep 'mempalace.*tools/call' /tmp/cc.log` — if you see `tools/list` but no `tools/call`, Claude sees the tools but isn't triggered to use them. Re-read the Memory rule wording.

### Write path

```bash
# 1. Hooks are at user scope
jq '.hooks | keys' ~/.claude/settings.json
# expect: ["PreCompact","Stop"]

# 2. Note drawer count
mempalace status | grep -A1 <your-wing>

# 3. Run a normal Claude Code session, chat for 15+ messages, exit.
# 4. Re-check drawer count — should have grown.
```

If drawer count doesn't grow, the `Stop` hook isn't firing. Some older docs show `echo "$INPUT" | mempalace hook run ...` piping JSON from stdin. Current integrations don't need this — the bare command works — but if you're on an older mempalace version, add the stdin pipe.

---

## 8. Session Config — `cl` vs `cl-qwen` / `cl-dev`

Different aliases, different memory retrieval modes:

| Alias | Backend | MCP | Memory retrieval |
|---|---|---|---|
| `cl` | Anthropic API | Enabled | Auto via `mempalace_status` + `mempalace_search` (MCP tools) |
| `cl-qwen`, `cl-dev` | Local models | `CLAUDE_CODE_SIMPLE=1` | Manual: run `mempalace wake-up --wing <project>` before session |

Local models can't do MCP tool calls reliably, so you inject the context up front with `wake-up`. That's ~170 tokens of pre-loaded palace protocol + wing summary.

If you want seamless local retrieval too, the upgrade path is a separate `~/.claude/mcp-local.json` containing only the mempalace MCP server, pointed at by the local aliases. Reserves full tool-call overhead for `cl` while still giving local models palace access.

---

## 9. Troubleshooting Cheatsheet

| Symptom | Cause | Fix |
|---|---|---|
| `mempalace wake-up` crashes `KERN_INVALID_ADDRESS at 0x88` | ChromaDB 1.5.7 Rust binding bug | `pipx inject --force mempalace 'chromadb<1'` |
| `pipx inject` says "already injected, not modifying" | Chromadb present as transitive dep | Add `--force` |
| `python3 -m venv` dies with `SIGKILL` | Python 3.14 broken on machine | Switch to Python 3.12 via pyenv |
| `chromadb<1` install tries to compile hnswlib (fails) | Python 3.13+ has no prebuilt wheel | Switch to Python 3.12 |
| Claude Code ignores mempalace, tools "connected" but never called | Weak/absent CLAUDE.md Memory rule, or rule at `###` level | Promote to `##`, use explicit recall triggers |
| `Hook output does not start with {, treating as plain text` in debug log | Using `mempalace wake-up` as SessionStart hook | Remove it. Wake-up is CLI-only. Use Stop + PreCompact instead. |
| Searches always return near-empty | Palace has ~2 drawers | Backfill with `mempalace mine` against both project dir and `.claude/projects/...` |
| `mempalace_status` works in one project but fails in another | MCP registered at local scope | `claude mcp remove mempalace -s local && claude mcp add mempalace -s user -- ...` |
| Teammates see hook errors on every session start | Hooks committed to project `.claude/settings.json` | Move to `~/.claude/settings.json` (user scope) |
| Segfault returns after `pipx upgrade` | ChromaDB got pulled back to 1.x | Re-run `pipx inject --force mempalace 'chromadb<1'` |

---

## 10. What NOT to Do (Hard-Learned)

- **Don't `pip install mempalace`.** System python, PEP 668 errors, and version chaos. Always pipx.
- **Don't install under Python 3.9 or 3.14.** 3.9 is too old for ChromaDB's Rust binding; 3.14 is too new.
- **Don't use a `SessionStart` hook with `mempalace wake-up`.** It's CLI output, not JSON; Claude Code logs a warning and ignores it.
- **Don't commit hooks or mempalace MCP registration to project-level `.claude/settings.json`.** Teammates without mempalace see errors every session.
- **Don't uninstall "extra" Homebrew Pythons.** Other formulae (awscli, pipx, mlx, etc.) silently depend on them. Check with `brew uses --installed python@X.Y` first.
- **Don't run `pipx upgrade` without re-pinning chromadb.** You'll resurrect the ARM64 segfault.

---

## Appendix: Useful Commands Reference

```bash
# Palace inspection
mempalace status                         # wings, drawer counts, kg stats
mempalace status --wing <your-wing>       # just one wing

# Manual retrieval (for local models or debugging)
mempalace wake-up --wing <your-wing>      # prints palace protocol + wing summary

# Indexing
mempalace mine <dir> --mode projects --wing <name>   # source/docs
mempalace mine <dir> --mode convos   --wing <name>   # Claude Code convo histories

# Health checks
claude mcp list                           # all MCP servers
claude mcp get mempalace                  # scope + command
jq '.hooks | keys' ~/.claude/settings.json   # hook names
claude --debug 2>&1 | tee /tmp/cc.log    # full session log

# Reinstall cleanly (nuclear option)
pipx uninstall mempalace
rm -rf ~/my-palace
pipx install --python "$(pyenv which python)" mempalace
pipx inject --force mempalace 'chromadb<1'
mempalace init --yes ~/my-palace
mempalace mine ~/Projects/<project> --mode projects --wing <project>
```
