# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**nanobot** is an ultra-lightweight personal AI assistant framework (~4,000 LOC in core agent). It supports multi-channel communication (Telegram, Discord, Slack, Matrix, WhatsApp, Email, Feishu, DingTalk, QQ, Mochat), tool execution, long-term memory, cron scheduling, and multiple LLM providers via LiteLLM.

## Commands

### Install (development)
```bash
pip install -e .
# or
uv tool install nanobot-ai
```

### Run
```bash
nanobot onboard        # Initialize config and workspace (~/.nanobot/)
nanobot agent          # CLI chat (REPL or single-turn)
nanobot gateway        # Multi-channel server mode
nanobot cron           # Manage scheduled tasks
```

### Lint
```bash
ruff check nanobot tests
ruff format nanobot tests
ruff check --fix nanobot tests
```
Ruff is configured for Python 3.11+, line length 100, ignores E501. Rules: E, F, I, N, W.

### Test
```bash
pytest                               # All tests
pytest tests/test_commands.py        # Single file
pytest tests/test_commands.py::test_function_name  # Single test
pytest -v --cov=nanobot              # With coverage
```
Tests use `pytest-asyncio` with `asyncio_mode = "auto"`. Test files live in `tests/` (24 files).

## Architecture

### Core Data Flow
```
Channel (Telegram/Discord/...) 
    → InboundMessage → MessageBus.inbound queue
        → AgentLoop._process_message()
            → ContextBuilder (system prompt + memory + skills)
            → LLMProvider.call() 
            → ToolRegistry.execute() (may loop up to 40 iterations)
            → MemoryConsolidation (when history grows)
        → OutboundMessage → MessageBus.outbound queue
    → Channel.send()
```

### Key Modules

| Module | Path | Purpose |
|--------|------|---------|
| AgentLoop | `nanobot/agent/loop.py` | Core agentic reasoning loop |
| MessageBus | `nanobot/bus/queue.py` | Async queue decoupling channels from agent |
| BaseChannel | `nanobot/channels/base.py` | Abstract interface all channels implement |
| LLMProvider | `nanobot/providers/` | Abstract LLM backend; LiteLLM is the default |
| ToolRegistry | `nanobot/agent/tools/registry.py` | Dynamic tool management and execution |
| ContextBuilder | `nanobot/agent/context.py` | Assembles system prompt from Markdown docs |
| MemoryManager | `nanobot/agent/memory.py` | Two-layer memory (MEMORY.md + HISTORY.md) |
| SessionManager | `nanobot/session/manager.py` | Append-only JSONL conversation persistence |
| CronService | `nanobot/cron/service.py` | Task scheduling (at/every/cron expressions) |
| HeartbeatService | `nanobot/heartbeat/service.py` | Periodic LLM-guided decision making |
| Config | `nanobot/config/schema.py` | Pydantic v2 models for all configuration |

### Extension Points

**Adding a channel**: Subclass `BaseChannel` in `nanobot/channels/`, implement `start()`, `stop()`, `send()`, `is_allowed()`. Register in `nanobot/channels/__init__.py`.

**Adding a tool**: Subclass `Tool` in `nanobot/agent/tools/`, define `name`, `description`, `parameters` (JSON Schema), and async `execute()`. Register via `ToolRegistry.register()`.

**Adding a provider**: Subclass `LLMProvider` in `nanobot/providers/`, implement `call()` and `get_default_model()`. Register in `nanobot/providers/registry.py`.

### Configuration
- **Storage**: `~/.nanobot/config.json` (auto-created by `nanobot onboard`)
- **Schema**: Pydantic v2 with camelCase aliases (`alias_generator=to_camel`)
- **Paths module**: `nanobot/config/paths.py` — all workspace paths defined here
- **Auto-migration**: `_migrate_config()` in the config loader upgrades old formats

### Memory System
- `MEMORY.md` — Long-term facts (written by agent via `save_memory` tool)
- `HISTORY.md` — Grep-searchable timestamped log (each line: `[YYYY-MM-DD HH:MM] ...`)
- Consolidation: LLM summarizes old `session/*.jsonl` messages into MEMORY.md + HISTORY.md when `memory_window` is exceeded. Only unconsolidated messages are sent to the LLM.

### Session Persistence
Sessions stored as append-only JSONL at `~/.nanobot/sessions/{channel}:{chat_id}.jsonl`. Messages are never deleted (supports prompt caching). `last_consolidated` index tracks the consolidation offset.

### Skills System
Skills live in `nanobot/skills/` as directories with a `SKILL.md` containing YAML frontmatter and Markdown instructions. The frontmatter `always_available: true` causes the skill to be loaded into every system prompt. Skills marked `always_available: false` appear as a listed capability the agent can invoke on demand.

### Templates
`nanobot/templates/` contains the Markdown files assembled into the system prompt:
- `AGENTS.md` — Agent behavior instructions
- `SOUL.md` — Personality definition  
- `USER.md` — User context
- `TOOLS.md` — Tool usage documentation
- `HEARTBEAT.md` — Periodic task list

### Security Model
- Channel access uses `allow_from` lists; empty = deny all, `["*"]` = allow all
- Shell tool blocks dangerous patterns (`rm -rf`, `mkfs`, `dd`, fork bombs)
- File tool enforces path traversal protection
- Command execution has a configurable timeout (default 60s) with 10KB output truncation
