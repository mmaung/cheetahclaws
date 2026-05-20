# 7-Day Codebase Onboarding Plan

CheetahClaws is a ~45K LOC Python AI assistant framework — multi-provider REPL, daemon mode, bridges (Telegram/Slack/WeChat), web UI, plugin system, and research lab.

---

## Day 1 — Orient & Run It

- Read [README.md](README.md) for full context
- Install: `pip install -e ".[all]"`, set `ANTHROPIC_API_KEY`, run `cheetahclaws`
- **Goal**: have a working REPL conversation

---

## Day 2 — Core Loop

Read these 4 files in order:

| File | LOC | What it does |
|------|-----|--------------|
| [cheetahclaws.py](cheetahclaws.py) | 1904 | REPL entry point, slash dispatch, permission UI |
| [agent.py](agent.py) | 742 | Generator-based multi-turn agent loop |
| [providers.py](providers.py) | 1587 | Provider abstraction + unified streaming |
| [tool_registry.py](tool_registry.py) | 180 | Tool dispatch + permission gating |

**Mental model**: `main()` → REPL → `agent.run()` generator → provider streaming → tool registry → permission gate → tool execution.

---

## Day 3 — Tools & Commands

- Browse [tools/](tools/) (12 files): Bash, file I/O, web, Jupyter, diagnostics
- Browse [commands/](commands/) (14 files): `/save`, `/agent`, `/mcp`, `/memory`, etc.
- **Exercise**: trace one slash command end-to-end from dispatch → handler → tool

---

## Day 4 — Context Management

| File | What it does |
|------|--------------|
| [context.py](context.py) | System prompt assembly |
| [compaction.py](compaction.py) | Context window snipping + LLM summarization |
| [session_store.py](session_store.py) | On-disk session history |
| [cc_config.py](cc_config.py) | `~/.cheetahclaws/config.json` load/save |

---

## Day 5 — Daemon & Bridges

- [cc_daemon/](cc_daemon/) (25 files): subprocess supervision (F-1..F-9), RPC, quota-pause
- [bridges/](bridges/) (9 files): Telegram, Slack, WeChat adapters + message routing
- **Exercise**: run `cheetahclaws serve` and inspect the RPC interface

---

## Day 6 — Agent-OS Kernel & Web UI

- [cc_kernel/](cc_kernel/) (27 files): sandbox, orchestrator, scheduler, ledger — v1.0 milestone
- [web/](web/) (19 files): FastAPI + xterm.js terminal, multi-user auth, CSRF protection
- Read [docs/architecture.md](docs/architecture.md) and relevant [docs/RFC/](docs/RFC/) entries

---

## Day 7 — Tests & Extensibility

- Run `pytest tests/` — understand coverage across 129 test files
- [plugin/](plugin/) + [skill/](skill/): git-based plugin discovery, Markdown skill templates
- [memory/](memory/) (9 files): persistent cross-session memory + semantic search
- [research/](research/) (16 dirs): 20+ source aggregation (arxiv, stocks, news, social)
- Read [docs/roadmap/](docs/roadmap/) to understand design decisions and what's coming

---

## Key Architecture at a Glance

```
cheetahclaws.main()
  └── REPL (prompt_toolkit)
        └── agent.run()  ← generator, yields typed events
              ├── providers.py  ← routes to Anthropic/OpenAI/Ollama/etc.
              └── tool_registry.py  ← permission-gated tool dispatch
                    └── tools/  ← Bash, file I/O, web, Jupyter, ...

Daemon layer (cc_daemon/):
  cheetahclaws serve → RPC server → runner subprocesses + bridge pollers

Agent-OS (cc_kernel/):
  sandbox + orchestrator + scheduler + ledger + observability
```

## Dependency Flow Rule

Imports go **downward only** — `tools/` never imports `cheetahclaws.py`. Circular refs are broken via lazy imports inside functions.

## Extension Points

| Mechanism | Where | How |
|-----------|-------|-----|
| Plugins | `plugin/` | Git-based discovery, registered at runtime |
| Skills | `skill/` | Markdown templates, inline or fork execution |
| MCP servers | `cc_mcp/` | stdio/SSE/HTTP JSON-RPC |
| Custom tools | `tool_registry.py` | Register a `ToolDef` |
