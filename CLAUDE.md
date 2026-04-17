# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build & Development

```bash
# Setup (Python 3.11+, uv required)
uv venv venv --python 3.11
source venv/bin/activate
uv pip install -e ".[all,dev]"

# Run the CLI
hermes                    # interactive TUI
hermes chat -q "Hello"    # one-shot
hermes doctor             # diagnostics

# Run the gateway (messaging platforms)
hermes gateway

# Entry points: hermes (CLI), hermes-agent (standalone agent loop), hermes-acp (editor integration)
```

## Testing

```bash
# Unit tests (default — skips integration/e2e, runs in parallel)
python -m pytest tests/ -q --ignore=tests/integration --ignore=tests/e2e -n auto

# Single test file
python -m pytest tests/tools/test_terminal_tool.py -v

# Single test
python -m pytest tests/tools/test_terminal_tool.py::test_function_name -v

# E2E tests (separate suite)
python -m pytest tests/e2e/ -v

# Integration tests (need API keys)
python -m pytest tests/integration/ -v
```

pytest config is in `pyproject.toml` — default markers skip `integration` tests, `-n auto` enables parallel via pytest-xdist.

## Architecture

### Core Loop

```
User message → AIAgent._run_agent_loop() (run_agent.py)
  ├── Build system prompt (agent/prompt_builder.py) — identity + skills + context + memory
  ├── Call LLM via OpenAI-compatible API (provider-agnostic)
  ├── If tool_calls → registry dispatch → results → loop back to LLM
  ├── If text response → persist session → return
  └── Context compression if near token limit (agent/context_compressor.py)
```

Messages follow OpenAI format (`role: system/user/assistant/tool`). The agent is synchronous — async tools are bridged via `_run_async()`.

### File Dependency Chain

```
tools/registry.py  (no deps — imported by all tool files)
       ↑
tools/*.py  (each calls registry.register() at import time)
       ↑
model_tools.py  (imports registry + triggers tool discovery)
       ↑
run_agent.py, cli.py, batch_runner.py
```

### Key Modules

- **`run_agent.py`** — `AIAgent` class, core conversation loop, tool dispatch, session persistence
- **`cli.py`** — `HermesCLI` class, interactive TUI via prompt_toolkit
- **`hermes_cli/main.py`** — All `hermes` subcommands, setup wizard, config management
- **`hermes_cli/auth.py`** — Provider credential resolution (Nous Portal, OpenRouter, OpenAI, Anthropic, Bedrock, custom endpoints)
- **`model_tools.py`** — Tool orchestration layer, `_discover_tools()`, `handle_function_call()`
- **`toolsets.py`** — Tool grouping definitions, platform presets (`_HERMES_CORE_TOOLS`)
- **`hermes_state.py`** — SQLite session DB with FTS5 full-text search
- **`tools/registry.py`** — Central tool registry (schemas, handlers, dispatch, availability)

### Tool System

Tools self-register at import time. Adding a tool requires 3 files:

1. **`tools/your_tool.py`** — schema + handler + `registry.register()` call
2. **`model_tools.py`** — add import to `_discover_tools()` `_modules` list
3. **`toolsets.py`** — add to existing toolset or create new one

All tool handlers MUST return a JSON string. Use `check_fn` for conditional availability and `requires_env` for API key gating.

### Skill System

Skills are markdown files (`SKILL.md`) with YAML frontmatter in `skills/` (bundled) or `optional-skills/` (official but not default). Most new capabilities should be skills, not tools — prefer skills unless the capability needs custom Python, binary data handling, or precise execution guarantees.

### Slash Command Registry

All slash commands are defined in `hermes_cli/commands.py` as `CommandDef` objects. Adding an alias only requires modifying the `aliases` tuple — dispatch, help, Telegram menu, Slack mapping, and autocomplete all derive from the central registry automatically.

### Gateway

`gateway/run.py` (`GatewayRunner`) handles multi-platform messaging — Telegram, Discord, Slack, WhatsApp, Signal, etc. Platform adapters are in `gateway/platforms/`. Each conversation gets an isolated `AIAgent` session.

### Config System

User config lives in `~/.hermes/` (overridable via `HERMES_HOME`):
- `config.yaml` — settings (model, tools, terminal, compression, memory, display)
- `.env` — API keys and secrets
- `skills/` — active skills
- `state.db` — session database
- `memories/` — persistent memory (MEMORY.md, USER.md)

Three config loaders exist: `load_cli_config()` (CLI), `load_config()` (setup/tools commands), direct YAML load (gateway). Adding a config option: add to `DEFAULT_CONFIG` in `hermes_cli/config.py` and bump `_config_version`.

### Environment Backends

Terminal execution is pluggable via `tools/environments/` — `BaseEnvironment` ABC with implementations: local, docker, ssh, singularity, modal, daytona. Use `get_hermes_home()` for state file paths (never hardcode `~/.hermes`).

## Code Conventions

- **PEP 8** with practical exceptions (no strict line length enforcement)
- **Cross-platform**: never assume Unix — guard `termios`/`fcntl` imports, handle paths portably
- **Error handling**: catch specific exceptions, log with `logger.warning()`/`logger.error()` and `exc_info=True` for unexpected errors
- **Ephemeral injection**: system prompts and prefill messages injected at API call time, never persisted to DB or logs
- **Profile-aware paths**: use `get_hermes_home()` / `display_hermes_home()` for file paths, never `Path.home() / ".hermes"`

## AWS Bedrock Provider

Bedrock uses the Anthropic SDK's `AnthropicBedrock` class (not boto3 Converse API) to preserve streaming, prompt caching, and reasoning support.

**Activation** (env var or config.yaml):
```bash
export CLAUDE_CODE_USE_BEDROCK=1        # temporary override
# or config.yaml: model.provider: bedrock  (permanent)

export AWS_REGION=us-east-1
export AWS_BEARER_TOKEN_BEDROCK=...     # bearer token auth
# or AWS_ACCESS_KEY_ID + AWS_SECRET_ACCESS_KEY  (SigV4 auth)

export ANTHROPIC_MODEL=us.anthropic.claude-sonnet-4-6
export ANTHROPIC_SMALL_FAST_MODEL=...   # auxiliary model for compression/vision
export DISABLE_PROMPT_CACHING=0         # set to 1 to disable
```

**Key integration points**:
- `agent/anthropic_adapter.py` — `build_bedrock_client()` creates `AnthropicBedrock` (SigV4) or `AnthropicBedrock(api_key=...)` (bearer). `is_bedrock_model_id()` detects ARNs and vendor-prefixed IDs.
- `hermes_cli/runtime_provider.py` — Bedrock resolution branch (before Anthropic), `CLAUDE_CODE_USE_BEDROCK` detection in `resolve_requested_provider()`
- `hermes_cli/auth.py` — `bedrock` entry in `PROVIDER_REGISTRY` with aliases (aws, aws-bedrock, amazon-bedrock)
- `run_agent.py` — Bedrock client init and `switch_model()` both in the `api_mode == "anthropic_messages"` path, gated on `provider == "bedrock"`
- `hermes_cli/model_normalize.py` — `bedrock` in `_AUTHORITATIVE_NATIVE_PROVIDERS` so model IDs pass through unchanged

**Model IDs**: Bedrock accepts full ARNs (`arn:aws:bedrock:...`), regional prefixes (`us.anthropic.claude-*`), and vendor prefixes (`anthropic.claude-*`). These must NOT be normalized (no dot-to-hyphen conversion). `is_bedrock_model_id()` guards this.

**Dependency**: `AnthropicBedrock` requires `botocore` at runtime (installed via `pip install boto3`).
