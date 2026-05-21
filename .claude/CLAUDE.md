# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

Source of truth for dependencies is `pyproject.toml`; `requirements.txt` mirrors it.

```bash
# Dev install from source
pip install -r requirements.txt        # or: pip install -e ".[all]"
python cheetahclaws.py                  # run the REPL without installing

# After packaging changes, verify pip install still ships everything
pip install --force-reinstall .

# Tests (pytest picks up both test_*.py and e2e_*.py — see [tool.pytest.ini_options])
python -m pytest tests/ -x -q
python -m pytest tests/test_providers.py            # one file
python -m pytest tests/test_providers.py::test_name # one test
python -m pytest tests/ -k "compaction"             # by keyword
python -m pytest tests/ --ignore=tests/e2e_daemon_skeleton.py   # skip unix-only e2e on Windows
```

Live-LLM e2e tests are skipif-gated on env vars (e.g. `CC_LITELLM_E2E=1`, `ANTHROPIC_API_KEY`); they will silently skip without those set. There is no separate linter target — CI is just pytest on Python 3.10–3.13.

## Architectural invariants

These are load-bearing; breaking them is a bug, not a style issue.

### `config` dict vs `RuntimeContext` — never mix them

- **`config`** (dict, loaded from `~/.cheetahclaws/config.json`) is serializable user settings only. `save_config()` strips any `_`-prefixed key before writing.
- **`RuntimeContext`** (`runtime.py`, keyed by `_session_id`) holds per-session live state: threads, bridge flags, plan-mode pointer, pending images, streaming hooks. Never persisted.
- The **only** `_`-prefixed key allowed in `config` is `_session_id`. Per-turn transients (`_depth`, `_system_prompt`, `_worktree_cwd`) are injected by `agent.run()` into a local copy and never persisted.

```python
# CORRECT
sctx = runtime.get_ctx(config)
sctx.plan_file = path
# WRONG
config["_plan_file"] = path
```

### Tool registration is the single extension point

Everything the model can call ends up in `tool_registry._registry`. Plugins, MCP servers, skills, and feature packages all compose through it without knowing about each other.

- **Plugin tools** export a module-level `TOOL_DEFS = [ToolDef(...)]` list — the loader (`plugin/loader.py::register_plugin_tools`) registers them. Do **not** call `register_tool()` directly from plugin code; it bypasses scoping/resolution.
- **Plugin commands** export `COMMAND_DEFS = {"name": {"func": ..., "help": (desc, [subcommands])}}`.
- Plugins resolve from `~/.cheetahclaws/plugins/<name>/` (user) or `.cheetahclaws/plugins/<name>/` (project) via `plugin.json` manifest. `CHEETAHCLAWS_DISABLE_PLUGINS` and `CHEETAHCLAWS_PLUGIN_ALLOWLIST` gate loading.

### Bootstrap order, no import-time side effects

`bootstrap.py` is the single place startup side effects happen, in a fixed order: logging → tool registry → health server. Do not add `register_tool()` (or anything similar) at module top level — register via `_EXTENSION_MODULES` or the modular ecosystem.

### Provider abstraction: neutral message format

Every subsystem speaks one canonical message format. Providers (`providers.py`) adapt only at the boundary (`stream_anthropic`, `stream_openai`, `stream_litellm`, …). Never let provider-specific shapes leak into the pipeline. Auto-detection routes models from the `PROVIDERS` registry; cost tracking and `ProviderInvalidRequest` vs `ProviderUnavailable` mapping happen there.

### Dependency flow is downward only

Feature packages (`tools/`, `commands/`, `bridges/`, `cc_daemon/`, …) **never** import `cheetahclaws.py`. Circular refs are broken with lazy imports inside functions, not by reshuffling modules.

### Renamed modules — do not "fix"

- `config.py` → **`cc_config.py`** (stdlib collision). Always `import cc_config`.
- `mcp/` → **`cc_mcp/`** (package collision). Always `from cc_mcp import …`.
- Plan mode writes to `.nano_claude/plans/<session>.md` in cwd — `.nano_claude` is historical and intentional. Don't rename it without updating plan-mode code. Other runtime state lives under `~/.cheetahclaws/`.

### Packaging discipline (issue #97)

- Top-level `.py` files **must** be added to `pyproject.toml` `py-modules`, or `pip install .` silently drops them.
- New sub-packages under an already-tracked top-level package auto-discover via `[tool.setuptools.packages.find]`.
- New top-level packages need a wildcard entry (`"newpkg*"`) in the `include` list.
- **Never** use the same name in `py-modules` *and* as a directory package (e.g., a `memory.py` shim next to a `memory/` package). On Windows + Python 3.13 + setuptools ≥ 75 this triggers a silent package-drop during wheel build. Regression test: `tests/test_packaging.py::test_pyproject_no_module_package_collision`. Existing shims (`memory.py`, `skills.py`, `subagent.py`) are listed — don't delete them from `py-modules` without also deleting the shim file.

## End-to-end request flow

```
cheetahclaws.main() → REPL (prompt_toolkit)
  → agent.run()  (generator, yields ToolStart/ToolEnd/PermissionRequest/TurnDone)
      → context.py builds system prompt (base/default.md + overlays/<family>.md + fragments)
      → compaction.maybe_compact() if near token limit
      → quota.check_quota() + circuit_breaker wrap providers.stream()
      → providers.stream() routes by model string → vendor adapter
      → tool_call → agent._check_permission → tool_registry.execute_tool
          → checkpoint hook snapshots pre-edit file (Write/Edit/NotebookEdit only)
          → tools/<category>.py runs
      → tool_result fed back; loop until end_turn
  → post-turn: checkpoint.snapshot_session() + session_store.save_latest()
```

Daemon mode (`cheetahclaws serve`) replaces the REPL with an RPC server; `/agent` runners optionally spawn as `python -m agent_runner --pipe` subprocesses supervised by `cc_daemon/runner_supervisor.py` (toggle `CHEETAHCLAWS_ENABLE_F4=1`). Bridges (`bridges/` — Telegram/Slack/WeChat) poll for messages and route through `RuntimeContext.run_query`; bridge state lives in `RuntimeContext`, never `config`.

## File-encoding discipline (`tools/fs.py`)

`_read` / `_write` / `_edit` force `encoding="utf-8"` and `newline=""`. `_edit` additionally detects pure-CRLF files and restores original line endings after the edit; mixed-ending files are left alone. Any new file-writing tool must mirror this.

## Prompt assembly

`prompts/base/default.md` is the single shared baseline. Vendor-documented quirks go in `prompts/overlays/<family>.md` only — there is no per-family base file. Two regression tests (`test_dead_family_base_files_are_gone`, `test_overlay_cites_source`) prevent drift. See `prompts/README.md` for the overlay-admission policy.

## Hooks system

There is no generic event-based hooks system. `checkpoint/hooks.py` wraps Write/Edit/NotebookEdit to snapshot files before mutation. There is no `hooks.json`, no `hook_session_start`/`hook_stop` events. Don't add one without an RFC.

## Security defaults (May 2026 hardening)

- Bash tool has a hard-denylist (`rm -rf /`, fork bomb, `mkfs`, `dd of=/dev/sd…`) that `permission_mode=accept-all` **cannot** bypass.
- `Read`/`Write`/`Edit` deny credential paths by default (`~/.ssh/id_*`, `~/.aws`, `/etc/shadow`, etc.).
- Bot tokens (`$TELEGRAM_BOT_TOKEN`, `$SLACK_BOT_TOKEN`) come from env only; never persisted to config or readline history.
- MCP server configs cannot inject `LD_PRELOAD` / `PYTHONPATH` / `DYLD_*` / `NODE_OPTIONS` into subprocess env.
- `permission_mode=accept-all` is session-scoped, never written to disk.

## Where to look

- Onboarding tour: `notes/ONBOARDING.md` (7-day plan), `notes/codebase-overview.md`.
- Architecture deep-dive: `docs/architecture.md` (invariants, end-to-end example, gotchas).
- Contributor conventions: `CONTRIBUTING.md` (PR checklist, plugin/skill/MCP shape).
- News/changelog with feature flags (`CHEETAHCLAWS_ENABLE_F4`/F6/F7/F8 etc.): `docs/news.md`.
- RFCs (daemon roadmap F-1..F-9): `docs/RFC/`.
