# CheetahClaws — Codebase Overview

**Version**: 3.05.79 (May 2026) | **License**: MIT | **Python 3.10+**

## What It Is

A Python-native, production-ready personal AI assistant that operates autonomously 24/7. It's a full-featured reimplementation of tools like Claude Code, supporting multiple AI providers and complex autonomous workflows.

---

## High-Level Architecture

```
User Input (Terminal / Web UI / Telegram / Slack)
    │
    ├─ cheetahclaws.py  — REPL, slash commands, streaming render
    ├─ agent.py         — Core multi-turn reasoning loop
    ├─ providers.py     — Adapters for Anthropic, OpenAI, Gemini, Ollama, etc.
    ├─ tool_registry.py — Central tool dispatch
    └─ compaction.py    — Context window management
```

Dependencies flow downward only. Nothing in feature packages imports from core at module load time; circular refs are broken with lazy imports.

---

## Key Subsystems

| Component | Purpose |
|-----------|---------|
| `tools/` | 40+ built-in LLM-callable tools (fs, shell, web, browser, email…) |
| `cc_daemon/` | Background daemon for 24/7 autonomous operation |
| `cc_kernel/` | Agent OS layer (orchestrator, scheduler, sandbox) |
| `web/` | FastAPI + xterm.js web UI with multi-user auth |
| `bridges/` | Telegram/Slack/WeChat messaging integrations |
| `memory/` | Persistent memory store with LLM-based consolidation |
| `multi_agent/` | Sub-agent spawning with ThreadPoolExecutor + git worktree isolation |
| `monitor/` | Subscription-based monitoring (arxiv, stocks, crypto, news) |
| `research/` | Research lab with 20+ academic/finance/tech sources |
| `skill/` | Markdown-based skill templates |
| `cc_mcp/` | Model Context Protocol (MCP) client |
| `task/` | In-session task list (thread-safe CRUD) |
| `checkpoint/` | Conversation snapshots (auto after turns) |
| `plugin/` | Plugin discovery and marketplace recommendation |

---

## Tech Stack

**Core**: Python 3.10+, `anthropic`, `openai`, `httpx`, `rich`, FastAPI, SQLite, SQLAlchemy, JWT/bcrypt

**Optional extras** (pip install):
- `browser` — playwright (web automation)
- `voice` — faster-whisper / sounddevice (STT + recording)
- `files` — pymupdf, openpyxl (PDF/Excel)
- `trading` — yfinance, rank-bm25 (stock/crypto analysis)
- `web` — full web UI + auth
- `litellm` — multi-provider routing (AWS Bedrock, Azure, Vertex AI)

---

## Agent Loop (`agent.py`)

1. User message enters the REPL
2. Agent streams from the selected LLM provider
3. Tool calls are intercepted → permission checked → executed
4. Results are fed back into the conversation as tool_result messages
5. Loop continues until `end_turn` or context limit triggers compaction

---

## Multi-Provider Support (`providers.py`)

- Central `PROVIDERS` registry with endpoint URLs, API keys, context limits
- Auto-detection from model string (e.g. `claude-opus-4-6` → anthropic, `gpt-4o` → openai)
- Streaming adapters normalize response formats across providers
- Cost tracking: per-provider token pricing, cache hit/write tokens counted
- Error mapping: `ProviderInvalidRequest` vs `ProviderUnavailable` for retry logic

---

## System Prompt Assembly (`context.py`)

- Base template: `prompts/base/default.md`
- Overlay appended per model family: `prompts/overlays/<family>.md`
- Fragments injected conditionally (plan status, tmux context, memory index)
- Injection threat scanner detects prompt-hijack attempts in user input

---

## Context Window Management (`compaction.py`)

- Cheap snip: truncate old messages by token count
- LLM summarization: if snipping loses too much, call auxiliary model to summarize
- Per-provider limits from providers registry
- Message coalescing: merge adjacent same-role turns to save tokens

---

## Daemon Mode (`cc_daemon/`)

- `/agent start <template>` spawns `agent_runner.py --pipe` as child process
- IPC via JsonLineChannel (logs, tool results, notifications)
- `runner_supervisor.py` monitors for crashes, restarts on policy
- Persistence: iteration logs → `agent_runs` / `agent_iterations` SQLite tables
- Bridge notifications send results to Telegram/Slack/WeChat

---

## Slash Command Ecosystem (`commands/`)

| File | Commands |
|------|---------|
| `config_cmd.py` | `/model`, `/config`, `/permissions` |
| `session.py` | `/save`, `/load`, `/resume` |
| `advanced.py` | `/brainstorm`, `/worker`, `/memory`, `/agents`, `/skills`, `/mcp`, `/plugin`, `/tasks` |
| `checkpoint_plan.py` | `/checkpoint`, `/rewind`, `/plan` |
| `agent_cmd.py` | `/agent start/stop/list` |
| `monitor_cmd.py` | `/subscribe`, `/monitor`, `/unsubscribe` |
| `daemon_cmd.py` | `/daemon status/stop/logs` |

---

## Security (May 2026 hardening)

- Bot tokens via env var only (never in conversation history)
- Web CSRF double-submit cookies
- Terminal session JWT owner-binding
- Bash hard-denylist (cannot be bypassed with `--accept-all`)
- Credential-path denylist (SSH keys, AWS creds, `/etc/shadow`)
- Plugin/MCP/FS sandboxing
- macOS daemon peer-cred auth via `getpeereid`

---

## Testing

- **2,347 passing tests** as of May 12, 2026
- E2E coverage for live LLM providers (skipif-gated on env vars)
- CI/CD via GitHub Actions
