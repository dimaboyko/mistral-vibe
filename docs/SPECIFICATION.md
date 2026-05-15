# Mistral Vibe — Software Specification

**Document purpose.** This is a stack-agnostic, reproduction-grade specification for *Mistral Vibe*, an open-source CLI coding assistant currently implemented in Python 3.12. The intent is that a team starting from scratch — in any language or runtime — can rebuild a functionally equivalent product from this document.

The reference implementation lives at `https://github.com/mistralai/mistral-vibe`; everywhere in this spec the *current source* paths in that repository are cited so reviewers can cross-check intent against behaviour. The spec describes the **what** and **how it behaves**; concrete language choices, frameworks, and identifiers are noted where they are part of the externally observable contract (e.g. configuration file names, protocol message shapes, command-line flags) and otherwise should be considered illustrative.

Version covered: `2.9.6` (per `pyproject.toml` and `distribution/zed/extension.toml`).

---

## Table of Contents

1. [Product Overview](#1-product-overview)
2. [System Architecture](#2-system-architecture)
3. [Distribution & Installation](#3-distribution--installation)
4. [Entry Points & CLI Surface](#4-entry-points--cli-surface)
5. [Configuration Subsystem](#5-configuration-subsystem)
6. [Filesystem Layout & Paths](#6-filesystem-layout--paths)
7. [Trust Folder System](#7-trust-folder-system)
8. [The Agent Loop](#8-the-agent-loop)
9. [LLM Backend Abstraction](#9-llm-backend-abstraction)
10. [Tool System & Permission Model](#10-tool-system--permission-model)
11. [Built-in Tools — Detailed Specs](#11-built-in-tools--detailed-specs)
12. [MCP & Connector Tools](#12-mcp--connector-tools)
13. [Agent Profiles](#13-agent-profiles)
14. [Skills System](#14-skills-system)
15. [Slash Commands](#15-slash-commands)
16. [Interactive TUI](#16-interactive-tui)
17. [Programmatic Mode & Output Formats](#17-programmatic-mode--output-formats)
18. [ACP (Agent Client Protocol) Bridge](#18-acp-agent-client-protocol-bridge)
19. [Voice & Audio (TTS / STT / Narrator)](#19-voice--audio-tts--stt--narrator)
20. [Sessions & Persistence](#20-sessions--persistence)
21. [System Prompts & AGENTS.md Discovery](#21-system-prompts--agentsmd-discovery)
22. [Hooks (Experimental)](#22-hooks-experimental)
23. [Onboarding & Authentication](#23-onboarding--authentication)
24. [Telemetry & Tracing](#24-telemetry--tracing)
25. [Update Notifier](#25-update-notifier)
26. [Auxiliary Modules](#26-auxiliary-modules)
27. [Event Catalog & Data Models](#27-event-catalog--data-models)
28. [Reproduction Notes for Reimplementers](#28-reproduction-notes-for-reimplementers)

---

## 1. Product Overview

Mistral Vibe is a terminal coding assistant. It provides a chat REPL in which an LLM-backed agent reads, edits, runs, and reasons about code in the developer's working directory, mediated by a fixed set of tools (file I/O, shell, search, web fetch, sub-agent delegation, interactive questions, etc.). Vibe ships three execution surfaces from a single codebase:

| Surface | Binary | Purpose |
|---------|--------|---------|
| Interactive TUI | `vibe` | Foreground terminal app with a rich Textual UI |
| Programmatic / headless | `vibe -p "<prompt>"` | Non-interactive single-shot or pipeline use |
| ACP server | `vibe-acp` | Long-running JSON-RPC server speaking the Agent Client Protocol — used by IDEs (Zed, JetBrains, etc.) |

Hard design tenets observable in the codebase:

- **Tool execution is gated by permissions** (`ALWAYS`, `ASK`, `NEVER`) — never silently destructive in `default` agent.
- **Trust is explicit per-folder** — project-local `.vibe/` config / hooks / skills are ignored until the folder is trusted.
- **Sessions are durable** — every conversation can be replayed, continued, or branched.
- **Provider-agnostic** — backends for Mistral, OpenAI-compatible, Anthropic, and Vertex are all present, selected per-model via a provider table.
- **Extensible** — third parties extend Vibe via Skills, custom Agents, MCP servers, and (experimentally) hooks.

---

## 2. System Architecture

### 2.1 Process model

A single Python process hosts the agent runtime. Concurrency is asyncio-based; the agent loop is an `AsyncGenerator[Event, None]`. Tool execution runs in async tasks; the TUI runs Textual's own asyncio event loop.

For the ACP surface the same process is a JSON-RPC server reading stdin and writing stdout (line-buffered), capable of managing N concurrent agent sessions in parallel.

### 2.2 Layered packages

Top-level packages (1:1 with directories under `vibe/`):

```
vibe/
├── core/    # Engine: agent loop, tools, LLMs, sessions, hooks, paths, config
├── cli/     # Interactive Textual TUI (the `vibe` binary)
├── acp/     # ACP server adapter (the `vibe-acp` binary)
└── setup/   # First-run wizard, auth, trust dialog
```

`core` is the only package that may be imported by `cli`, `acp`, or `setup`; `cli` / `acp` / `setup` MUST NOT import each other. `core` itself has no runtime dependency on any presentation layer (it accepts callbacks for I/O).

### 2.3 Ports & adapters

Several subsystems use the hexagonal "port" idiom: an abstract interface (`*_port.py`) consumed by the agent loop, with one or more concrete adapters provided at the presentation layer. The pattern appears for:

- `voice_manager` — STT input pipeline.
- `narrator_manager` — TTS output pipeline.
- `turn_summary` — async turn summarization.
- `update_notifier` — version-check gateways (PyPI / GitHub) and cache repository.
- `plan_offer` — `WhoAmIGateway` for plan eligibility detection.
- `tts` and `transcribe` — pluggable speech backends (current default: Mistral).
- `notifications` — desktop/terminal notification adapter.

### 2.4 Component diagram

```
                   ┌──────────────────────────────────────────────┐
       stdin ─────▶│ Entry point (cli/entrypoint.py |             │◀── argv
                   │  acp/entrypoint.py)                          │
                   └────────────────────┬─────────────────────────┘
                                        │ loads
                                        ▼
        ┌───────────────────────────────────────────────────────────────┐
        │ VibeConfig (core/config) ◀── .env, env vars, config.toml      │
        └───────────────────────────────────────────────────────────────┘
                                        │ injected into
                                        ▼
                  ┌───────────────────────────────────────────────┐
                  │ AgentLoop (core/agent_loop)                   │
                  │  - messages: MessageList                      │
                  │  - middleware pipeline                        │
                  │  - stats: AgentStats                          │
                  │  - hooks_manager, skill_manager,              │
                  │    agent_manager, tool_manager                │
                  └────────────┬──────────────────────────────────┘
                               │ yields BaseEvent
                  ┌────────────┴──────────────┐
                  ▼                           ▼
        ┌────────────────────┐    ┌──────────────────────────┐
        │ Textual TUI        │    │ ACP server / programmatic │
        │ (cli/textual_ui)   │    │ (acp/, core/programmatic) │
        └────────────────────┘    └──────────────────────────┘
                  │                           │
                  │  Tool invocations call BaseTool.invoke()
                  ▼                           ▼
        ┌─────────────────────────────────────────────────────┐
        │ Tool Manager (core/tools/manager)                    │
        │   - builtins (read_file, write_file, bash, …)        │
        │   - MCP-discovered tools                             │
        │   - Mistral connectors                               │
        │   - user-defined tools (config tool_paths)           │
        └─────────────────────────────────────────────────────┘
                                        │
                                        ▼
                  ┌─────────────────────────────────────────┐
                  │ Backend (core/llm/backend)              │
                  │  Mistral / Generic (OpenAI-compat,      │
                  │  Anthropic, Vertex, OpenAI Responses)   │
                  └─────────────────────────────────────────┘
                                        │
                                        ▼ httpx.AsyncClient
                                  ┌────────────┐
                                  │ LLM API    │
                                  └────────────┘
```

---

## 3. Distribution & Installation

### 3.1 Installation channels

1. **One-line installer** (`scripts/install.sh`): bootstraps `uv` if missing, then `uv tool install mistral-vibe`. The hosted endpoint is `https://mistral.ai/vibe/install.sh`.
2. **`uv tool install mistral-vibe`** — recommended path.
3. **`pip install mistral-vibe`** from PyPI.
4. **Pre-built `vibe-acp` binaries** via PyInstaller — published per release to GitHub Releases for `darwin-aarch64`, `darwin-x86_64`, `linux-aarch64`, `linux-x86_64`, `windows-x86_64` (see `vibe-acp.spec`).
5. **Zed extension** (`distribution/zed/extension.toml`) — declares the per-platform binary archive and the command (`./vibe-acp`).
6. **GitHub Action** (`action.yml`) — `mistralai/mistral-vibe` composite action runs `vibe -p "<prompt>"` in CI; inputs: `prompt`, `MISTRAL_API_KEY`, `install_python` (default `true`), `python_version` (optional).

A reimplementation should ship at least: a CLI binary that registers `vibe` and `vibe-acp` in `PATH`, and pre-built single-file binaries for the ACP server so IDE extensions can declare them as a download URL.

### 3.2 Build artefacts

- Wheel (`vibe/` package) for PyPI / `pip`.
- PyInstaller "onedir" bundle for `vibe-acp` (`vibe-acp.spec`): includes `vibe/core/prompts/*.md`, `vibe/core/tools/builtins/prompts/*.md`, `vibe/setup/*`, all tool modules in `vibe/core/tools/builtins/*.py` and `vibe/acp/tools/builtins/*.py`, plus a `truststore` runtime hook and `rich._unicode_data` submodules.

---

## 4. Entry Points & CLI Surface

### 4.1 The `vibe` command

Parsed flags (see `vibe/cli/cli.py`):

| Flag / arg | Type | Default | Effect |
|------------|------|---------|--------|
| `PROMPT` (positional) | str (optional) | — | Initial prompt for interactive session |
| `-v`, `--version` | flag | — | Print version, exit |
| `-p`, `--prompt [TEXT]` | str / flag | — | Programmatic mode; if flag given without value, reads prompt from stdin |
| `--max-turns N` | int | — | Cap on assistant turns (programmatic only) |
| `--max-price DOLLARS` | float | — | Halt when accumulated session cost exceeds (programmatic only) |
| `--enabled-tools TOOL` | str (multi) | — | Whitelist tool names/patterns; supports glob and `re:` regex |
| `--output {text|json|streaming}` | str | `text` | Output format (programmatic only) |
| `--agent NAME` | str | — | Agent profile to use |
| `--setup` | flag | — | Run onboarding wizard then exit |
| `--workdir DIR` | path | — | `cd` to this directory before running |
| `--trust` | flag | — | Trust the working directory for this invocation only |
| `-c`, `--continue` | flag | — | Resume the most recent saved session |
| `--resume [SESSION_ID]` | str / flag | — | Resume a specific session by ID; without an ID, opens a picker. Mutually exclusive with `-c` |

Startup order (the reimplementer must follow this sequence — observable behaviours depend on it):

1. Parse args.
2. If `--workdir` set, `chdir` and validate; abort if the dir is missing.
3. If `--trust` set, mark the workdir as session-trusted (not persisted).
4. In interactive mode, run `check_and_resolve_trusted_folder()` — discover trustable files (`.claude/settings.json`, `.vibe/config.toml`, etc.), prompt the user if the trust state is unknown, persist the answer to `~/.vibe/trusted_folders.toml`.
5. Initialize the *harness files manager* (which knows whether to load project-level config based on trust).
6. Load `.env` from `~/.vibe/.env` into the process environment.
7. Bootstrap defaults: create `~/.vibe/config.toml` and `~/.vibe/vibehistory` if absent.
8. If `--setup`, run onboarding and exit.
9. `VibeConfig.load()` — see §5. On `MissingAPIKeyError`, trigger onboarding and re-load; on other errors, exit.
10. Pick the initial agent profile: programmatic ⇒ `auto-approve` (overriding `default_agent`); interactive ⇒ `config.default_agent` unless `--agent` is set.
11. Load hooks from disk (no-op unless `enable_experimental_hooks` is true).
12. `setup_tracing(config)` (silently swallows errors).
13. Apply `--enabled-tools` override.
14. If `-c`/`--resume`, load the prior session messages/metadata.
15. If `-p`, branch to programmatic mode (§17); otherwise launch the TUI (§16).

### 4.2 The `vibe-acp` command

Parses `--setup` only (registers API key via terminal flow). Always starts the ACP JSON-RPC server on stdio. Honours `VIBE_ACP_LOGGING_ENABLED=true` to record all incoming/outgoing messages to `~/.vibe/logs/acp/messages.jsonl` (rotating, 1 MB × 3 backups).

### 4.3 Environment variables

| Variable | Read by | Effect |
|----------|---------|--------|
| `VIBE_HOME` | `core/paths/_vibe_home.py` | Override `~/.vibe` base directory |
| `MISTRAL_API_KEY` | default provider | API key for Mistral |
| `VIBE_*` (any name) | `pydantic-settings` | Override any top-level `VibeConfig` field (e.g. `VIBE_ACTIVE_MODEL`, `VIBE_ENABLE_TELEMETRY`) |
| `LOG_LEVEL` | `core/logger.py` | Default `WARNING` |
| `LOG_MAX_BYTES` | `core/logger.py` | Rotating file handler size |
| `VIBE_ACP_LOGGING_ENABLED` | `acp/acp_logger.py` | Enable ACP wire log |
| `HTTP_PROXY`/`HTTPS_PROXY`/`ALL_PROXY`/`NO_PROXY`/`SSL_CERT_FILE`/`SSL_CERT_DIR` | HTTP client | Proxy support |
| `VISUAL` / `EDITOR` | external editor (Ctrl+G) | Fallback `nano` |
| `SHELL` | bash tool | External shell on Unix |
| Provider-specific keys per `ProviderConfig.api_key_env_var` | LLM backends | Auth |

---

## 5. Configuration Subsystem

### 5.1 File precedence

`VibeConfig` (see `vibe/core/config/_settings.py`) merges sources in this order (highest wins):

1. Init kwargs (programmatic instantiation).
2. Environment variables prefixed `VIBE_` (case-insensitive, `extra="ignore"`).
3. The active TOML config file — chosen by the *harness files manager*:
   - If project `.vibe/config.toml` exists **and the folder is trusted** ⇒ use it.
   - Else ⇒ `~/.vibe/config.toml` (i.e. `$VIBE_HOME/config.toml`).
4. `file_secret_settings` (defaults).

**Note:** `.env` is **not** a pydantic source. It is loaded explicitly into `os.environ` at startup via `load_dotenv_values()`, so its values can be picked up by both `VIBE_*` env settings and by provider `api_key_env_var` lookups.

### 5.2 Full config schema

The following table catalogues every top-level field in `VibeConfig`. Types and defaults are normative.

| Field | Type | Default | Notes |
|-------|------|---------|-------|
| `active_model` | string | `mistral-medium-3.5` | Must match an alias in `models` |
| `vim_keybindings` | bool | `false` | Editor keybindings in the TUI input |
| `disable_welcome_banner_animation` | bool | `false` | Skip banner intro |
| `autocopy_to_clipboard` | bool | `true` | Copy last assistant message on `/copy` |
| `file_watcher_for_autocomplete` | bool | `false` | Use watchfiles for `@` completion |
| `displayed_workdir` | string | `""` | Override of CWD displayed in the bottom bar |
| `context_warnings` | bool | `false` | Emit reminder at 50 % of context |
| `voice_mode_enabled` | bool | `false` | Voice STT default |
| `narrator_enabled` | bool | `false` | TTS narration default |
| `active_transcribe_model` | string | `voxtral-realtime` | Must match alias in `transcribe_models` |
| `active_tts_model` | string | `voxtral-tts` | Must match alias in `tts_models` |
| `bypass_tool_permissions` | bool | `false` | Skip all approval prompts |
| `enable_telemetry` | bool | `true` | Send events to Mistral telemetry endpoint |
| `system_prompt_id` | string | `cli` | Built-in id or `~/.vibe/prompts/<id>.md` |
| `include_commit_signature` | bool | `true` | Add Co-Authored-By trailer when committing |
| `include_model_info` | bool | `true` | Inject model info into system prompt |
| `include_project_context` | bool | `true` | Inject git status / cwd into system prompt |
| `include_prompt_detail` | bool | `true` | Inject OS, tool, skill, agent info into prompt |
| `enable_update_checks` | bool | `true` | Check for new versions |
| `enable_auto_update` | bool | `true` | Auto-upgrade in place |
| `enable_notifications` | bool | `true` | OS/terminal bell on action required |
| `api_timeout` | float | `720.0` | Per-request timeout, seconds |
| `auto_compact_threshold` | int | `200_000` | Token threshold for auto-compact (per-model override possible) |
| `providers` | array of `ProviderConfig` | `[mistral, llamacpp]` | LLM HTTP endpoints |
| `models` | array of `ModelConfig` | 3 defaults | Inventory of usable models |
| `compaction_model` | `ModelConfig` or null | null | Specific model used for `/compact` |
| `transcribe_providers` / `transcribe_models` | arrays | Mistral voxtral defaults | STT |
| `tts_providers` / `tts_models` | arrays | Mistral voxtral defaults | TTS |
| `project_context` | object | `{default_commit_count: 5, timeout_seconds: 2.0}` | git context limits |
| `session_logging` | object | `{enabled: true, save_dir: ~/.vibe/logs/session, session_prefix: "session"}` | Session log behaviour |
| `tools` | dict[name → dict] | `{}` | Per-tool config; supports `permission`, `allowlist`, plus tool-specific keys |
| `tool_paths` | array of paths | `[]` | Extra directories to scan for tools |
| `mcp_servers` | array | `[]` | MCP integrations (see §12) |
| `connectors` | array of `ConnectorConfig` | `[]` | Mistral connector tools |
| `enabled_tools` / `disabled_tools` | arrays of patterns | `[]` | Allowlist / denylist with `re:` regex + glob support |
| `agent_paths` | array of paths | `[]` | Extra agent profile directories |
| `enabled_agents` / `disabled_agents` / `installed_agents` | arrays | `[]` | Same pattern semantics |
| `default_agent` | string | `default` | Initial agent profile in interactive mode |
| `skill_paths` | array of paths | `[]` | Extra skill directories |
| `enabled_skills` / `disabled_skills` | arrays | `[]` | Skill filters |

**Excluded-from-dump (internal) fields:** `vibe_code_enabled`, `vibe_code_base_url`, `vibe_code_workflow_id`, `vibe_code_task_queue`, `vibe_code_api_key_env_var`, `vibe_code_project_name`, `enable_otel`, `otel_endpoint`, `enable_experimental_hooks`.

### 5.3 ProviderConfig

```toml
[[providers]]
name = "mistral"              # unique alias
api_base = "https://api.mistral.ai/v1"
api_key_env_var = "MISTRAL_API_KEY"
browser_auth_base_url = "https://console.mistral.ai"
browser_auth_api_base_url = "https://console.mistral.ai/api"
api_style = "openai"          # "openai" | "anthropic" | "vertex" | "openai-responses"
backend = "mistral"           # "mistral" | "generic"
reasoning_field_name = "reasoning_content"
project_id = ""               # vertex
region = ""                   # vertex
extra_headers = {}
```

`backend` selects the implementation class in `core/llm/backend/factory.py`. `api_style` (within the Generic backend) selects the request/response adapter.

### 5.4 ModelConfig

```toml
[[models]]
name = "mistral-medium-3.5"   # vendor model id sent to the API
provider = "mistral"           # foreign key to providers[].name
alias = "mistral-medium-3.5"   # local lookup name (must be unique)
temperature = 1.0
input_price = 1.5              # $ per million input tokens
output_price = 7.5             # $ per million output tokens
thinking = "high"              # "off" | "low" | "medium" | "high" | "max"
auto_compact_threshold = 200_000
```

If `alias` is omitted it defaults to `name`. Duplicate aliases are rejected by validator. The "default models" inventory bootstrapped on first run is:
- `mistral-medium-3.5` (provider `mistral`, alias same as name, thinking `high`).
- `devstral-small-latest` ⇒ alias `devstral-small`.
- `devstral` (provider `llamacpp`, alias `local`).

### 5.5 Tool sub-table

```toml
[tools.bash]
permission = "ask"             # "always" | "ask" | "never"
allowlist = ["git status", "ls", "pwd"]
denylist = ["rm -rf"]
default_timeout = 300
max_output_bytes = 16000
sensitive_patterns = ["sudo"]

[tools.read_file]
permission = "always"
```

Tool config fields vary per tool (see §11). Built-in tools auto-discover their default config from a `BaseToolConfig` subclass.

### 5.6 MCP server table

See §12 for transport-specific fields. Discriminator key is `transport` ∈ {`http`, `streamable-http`, `stdio`}.

### 5.7 Persistence APIs

- `VibeConfig.save_updates({...})` — deep-merge a patch into the on-disk TOML.
- `VibeConfig.dump_config({...})` — full rewrite.
- `VibeConfig.add_tool_allowlist_patterns(tool_name, [...])` — used when the user approves "always" for a specific tool/pattern.
- `VibeConfig.set_thinking(level)` — persist a thinking-level change for the active model.
- `VibeConfig._migrate()` — runs on every `load()`; current migrations:
  - Add `find` to `tools.bash.allowlist` if missing.
  - Strip legacy trailing ` *` from bash allowlist patterns.
  - Rename old model alias `devstral-2` → `mistral-medium-3.5` and set new pricing/thinking defaults.

### 5.8 Merge metadata (schema introspection)

For composite multi-layer configs (used by ACP fork/merge), each `ConfigSchema` field declares a merge strategy via Pydantic Annotated metadata (`schema.py`):

- `WithReplaceMerge` — higher layer wins.
- `WithConcatMerge` — list concatenation.
- `WithUnionMerge(merge_key="alias")` — list union by key.
- `WithShallowMerge` — dict shallow merge.
- `WithConflictMerge` — error if multiple layers set the field.

A reimplementation can ignore this if it does not support layered merging; it is internal to ACP's `set_config_option` semantics.

---

## 6. Filesystem Layout & Paths

`vibe/core/paths/_vibe_home.py` defines the canonical paths (all derived lazily from `VIBE_HOME`):

```
$VIBE_HOME/                       (default: ~/.vibe/)
├── config.toml                  # User-level TOML config
├── .env                         # API keys & secrets (dotenv)
├── trusted_folders.toml         # Trust DB
├── vibehistory                  # Input history (JSON-per-line)
├── cache.toml                   # Update-check cache + "what's new" pointer
├── hooks.toml                   # User-level hook definitions (experimental)
├── agents/                      # Custom agent TOML files
├── prompts/                     # Custom system prompt markdown
├── skills/                      # User-level skills (each is dir + SKILL.md)
├── tools/                       # Custom tool implementations
├── plans/                       # Per-session plan markdown files
└── logs/
    ├── vibe.log                 # Rolling app log (StructuredLogFormatter)
    ├── session/                 # One subdir per session
    │   └── session_<timestamp>_<short_id>/
    │       ├── meta.json         # SessionMetadata
    │       └── messages.jsonl    # One LLMMessage per line
    └── acp/messages.jsonl       # ACP wire log (rotating, 1MB × 3)
```

### 6.1 Project-local discovery

When the project root is trusted, Vibe also scans for project-level configuration. Discovery uses a **bounded BFS** from the working directory (`_local_config_walk.py`):
- max depth 4
- max 2000 directories visited
- skips hidden directories and `WALK_SKIP_DIR_NAMES`

Found:

```
<project>/.vibe/
├── config.toml
├── hooks.toml
├── tools/
├── skills/
├── agents/
└── prompts/

<project>/.agents/
└── skills/                      # Agent Skills standard path
```

The set of found `config_dirs`, `tools`, `skills`, `agents` is exposed via `ConfigWalkResult`.

### 6.2 Constant filenames

- `AGENTS.md` — system prompt extension files (see §21).
- `SKILL.md` — frontmatter+content for each skill (§14).
- `whats_new.md` — embedded in the package, shown after upgrade.

---

## 7. Trust Folder System

**Purpose.** Refuse to load user-supplied code or config from a workspace until the user has opted in.

**File format** (`~/.vibe/trusted_folders.toml`):

```toml
[trusted]
"/home/user/project1" = ""

[untrusted]
"/home/user/sketchy" = ""
```

**Resolution algorithm** (`core/trusted_folders.py`):
1. Resolve the path to absolute form.
2. Walk up the parent chain to `/`.
3. The first ancestor present in `trusted`, session-trusted (in-memory), or `untrusted` decides.
4. Return `bool | None`. `None` means "no decision yet" — caller should prompt.

**When trust is consulted.**
- Loading project `.vibe/config.toml` and `.vibe/hooks.toml`.
- Discovering skills in `.vibe/skills/` and `.agents/skills/`.
- Discovering tools in `.vibe/tools/`.
- Discovering agents in `.vibe/agents/`.
- Reading project `AGENTS.md` files (§21).

**Trust dialog** (`vibe/setup/trusted_folders/trust_folder_dialog.py`): on `None`, a Textual modal asks: Trust permanently? Trust for this session? Deny? The choice is persisted into `trusted_folders.toml` (except "session only" which is in-memory).

`--trust` flag bypasses the dialog and applies session trust.

---

## 8. The Agent Loop

`AgentLoop` (`vibe/core/agent_loop.py`) is the heart of the runtime. Its public contract is an async generator of `BaseEvent` instances.

### 8.1 Turn structure

```
user.send(text)
    └── append UserMessageEvent to MessageList
         └── middleware.before_turn(...)
              ├── STOP        ⇒ yield AssistantEvent(stopped_by_middleware=True); return
              ├── COMPACT     ⇒ run /compact in line; restart turn
              ├── INJECT_MSG  ⇒ append synthetic user msg; continue
              └── CONTINUE
         └── backend.complete_streaming(...)   # or backend.complete(...)
              └── yield ReasoningEvent / AssistantEvent chunks
         └── APIToolFormatHandler.parse_message
              └── ResolvedMessage:
                  - valid tool calls
                  - failed tool calls (schema invalid)
         └── if tool calls:
              └── _run_tools_concurrently
                  ├── yield ToolCallEvent
                  ├── permission check (see §10)
                  ├── tool.invoke ⇒ AsyncGenerator
                  │     └── yield ToolStreamEvent[*]
                  ├── yield ToolResultEvent
                  └── append tool-role LLMMessage
         └── go around again
    last msg has no tool calls? exit loop.
└── hooks.run(POST_AGENT_TURN, session_id, ...)
    if hook returns an INJECTED_MESSAGE, restart loop.
```

### 8.2 Middleware pipeline

Implemented as an ordered list of objects each exposing `async before_turn(ctx) -> MiddlewareResult` (`vibe/core/middleware.py`). Actions:

- `CONTINUE` — proceed.
- `STOP` — terminate the loop with a reason string.
- `COMPACT` — trigger compaction (metadata carries the threshold/used tokens).
- `INJECT_MESSAGE` — prepend a synthetic user-role message.

Built-in middleware (must be present in a reimplementation):

| Middleware | Trigger | Action |
|------------|---------|--------|
| `TurnLimitMiddleware` | `stats.steps - 1 >= max_turns` | STOP |
| `PriceLimitMiddleware` | `stats.session_cost > max_price` | STOP |
| `AutoCompactMiddleware` | `stats.context_tokens >= model.auto_compact_threshold` | COMPACT |
| `ContextWarningMiddleware` | crosses 50 % once | INJECT one-time reminder |
| `ReadOnlyAgentMiddleware` | profile is `plan` or `chat` | INJECT reminder on enter; INJECT exit message on leave |

### 8.3 Compaction algorithm

1. Yield `CompactStartEvent(current_context_tokens, threshold, tool_call_id)`.
2. Save the current session to disk before mutating.
3. Append a synthetic user message containing the `compact.md` system-utility prompt (and any user-supplied tail).
4. Call `backend.complete` with the `compaction_model` (falls back to the active model).
5. Replace `MessageList` contents with `[system_prompt, assistant_summary]`.
6. Re-count tokens.
7. Yield `CompactEndEvent(old_context_tokens, new_context_tokens, summary_length, old_session_id, new_session_id, tool_call_id)`.
8. Generate a new session ID (preserving the stable suffix — see §20).

### 8.4 Stop conditions

- Middleware STOP.
- Last assistant message has no tool calls.
- `asyncio.CancelledError` (user pressed Ctrl+C or Escape) — cancels in-flight tool tasks, marks them `cancelled=True`, yields `ToolResultEvent(cancelled=True)`.
- `ContextTooLongError` (backend returned context-limit error) — surface to UI, ask user to `/rewind` or `/compact`.
- `RateLimitError` — surface, wait, no retry by the loop (provider SDK already handles 429 with backoff).

### 8.5 Stats (`AgentStats`)

Tracked per session, exposed to the TUI via observer pattern (`add_listener`):

- `steps`, `session_prompt_tokens`, `session_completion_tokens`, `context_tokens`
- `last_turn_*` snapshots (tokens, duration, tokens/sec)
- `tool_calls_*` counters (`agreed`, `rejected`, `failed`, `succeeded`)
- `input_price_per_million`, `output_price_per_million` (per active model)
- Computed: `session_total_llm_tokens`, `last_turn_total_tokens`, `session_cost`

---

## 9. LLM Backend Abstraction

### 9.1 Protocol

`BackendLike` (`vibe/core/llm/types.py`) is an async context manager exposing three methods:

```python
async def complete(*, model, messages, temperature, tools, max_tokens,
                   tool_choice, extra_headers, metadata) -> LLMChunk

def complete_streaming(...same kwargs...) -> AsyncGenerator[LLMChunk, None]

async def count_tokens(*, model, messages, temperature, tools,
                       tool_choice, extra_headers, metadata) -> int
```

`LLMChunk` is `(message: LLMMessage, usage: LLMUsage | None, correlation_id: str | None)` and supports `__add__` for accumulating streamed deltas.

### 9.2 Selection

`AgentLoop._select_backend()` reads `config.get_active_provider().backend` (∈ `MISTRAL`, `GENERIC`) and instantiates the corresponding class from `BACKEND_FACTORY`.

### 9.3 Mistral backend

Wraps the Mistral Python SDK with a custom `httpx`-style client (SSL via `certifi`, optional `truststore`). Translation:

- System message ⇒ `SystemMessage`.
- User text ⇒ `UserMessage` (string content).
- Assistant ⇒ `AssistantMessage`, with reasoning split into `ThinkChunk` content parts when present.
- Tool ⇒ `ToolMessage`.

Streaming: `client.chat.stream_async` yields deltas — accumulated via `LLMMessage.__add__`.

Retries: exponential backoff (500 ms initial, 30 s max, multiplier 1.5).

Errors are wrapped via `BackendErrorBuilder` into `BackendError` (with `is_context_too_long`, `status`).

### 9.4 Generic backend

Selects an `APIAdapter` based on `provider.api_style`:

- `openai` (default) — `core/llm/backend/generic.py`
- `openai-responses` — `openai_responses.py` (uses the Responses API shape)
- `anthropic` — `anthropic.py`
- `vertex` — `vertex.py` (Google ADC auth via `google-auth`)

Each adapter implements:

```python
def prepare_request(model, messages, ...) -> PreparedRequest  # url, headers, body
def parse_response(http_response) -> LLMChunk                  # to internal shape
def parse_streaming_chunk(line) -> LLMChunk | None             # SSE parsing
```

A `ReasoningAdapter` wrapper extracts model-supplied "thinking"/"reasoning" tokens into the dedicated `reasoning_content` field. The provider declares which field name the thinking content arrives under via `provider.reasoning_field_name`.

### 9.5 Reasoning / thinking

`LLMMessage` carries four reasoning fields:

- `reasoning_content` (text)
- `reasoning_state` (list[str], internal state segments)
- `reasoning_signature` (provider signature blob)
- `reasoning_message_id` (UUID; assigned client-side)

Per-model thinking level (`thinking ∈ {off, low, medium, high, max}`) is sent through to backends that support it (Mistral, Anthropic). Backends that don't support thinking ignore the field.

---

## 10. Tool System & Permission Model

### 10.1 BaseTool

`BaseTool` is a generic class parameterised by four type variables:

```python
BaseTool[ToolArgs: BaseModel,
         ToolResult: BaseModel,
         ToolConfig: BaseToolConfig,
         ToolState: BaseToolState]
```

Subclasses provide:

- `description: ClassVar[str]` — schema for the LLM
- `name` derived from class name by snake_case conversion
- `args_schema` from the `ToolArgs` type param
- `async def run(args, ctx) -> AsyncGenerator[ToolStreamEvent | ToolResult, None]` — yields progress and a single terminal result
- Optional: `resolve_permission(args)` — per-invocation override
- Optional: `format_call_display(args)` / `get_result_display(event)` — UI hints
- Optional: `get_file_snapshot()` — for rewind/replay
- Optional class method `discover_default_config()` — to seed default `[tools.<name>]` on first run

### 10.2 InvokeContext

Passed to every `run()`:

```
tool_call_id: str
approval_callback: ApprovalCallback | None
agent_manager: AgentManager | None
user_input_callback: UserInputCallback | None
sampling_callback: MCPSamplingHandler | None       # for MCP sampling/createMessage
session_dir: Path | None
entrypoint_metadata: EntrypointMetadata | None
plan_file_path: Path | None
switch_agent_callback: SwitchAgentCallback | None
skill_manager: SkillManager | None
scratchpad_dir: Path | None
```

### 10.3 Permission model

`ToolPermission` enum: `ALWAYS` | `ASK` | `NEVER`.

Per-invocation a tool may build a `PermissionContext`:

```python
PermissionContext(
    permission: ToolPermission,        # ALWAYS, ASK, NEVER
    required_permissions: list[RequiredPermission],
    reason: str | None,                # shown to user / LLM on NEVER
)
```

`RequiredPermission` is a scope-typed pattern that the user may "approve always" for the rest of the session:

```python
RequiredPermission(
    scope: PermissionScope,            # COMMAND_PATTERN | OUTSIDE_DIRECTORY |
                                       # FILE_PATTERN | URL_PATTERN
    invocation_pattern: str,           # what this call needs
    session_pattern: str,              # what gets remembered if user says "always"
    label: str,                        # human-readable
)
```

Session-approved rules live in `ApprovedRule[]` on the agent loop. On every tool call the loop calls `_is_permission_covered(required, approved_rules)` (glob/prefix match per scope).

Approval flow:

1. Tool returns `PermissionContext`.
2. If `permission == ALWAYS` ⇒ run.
3. If `permission == NEVER` ⇒ skip, feed `reason` back to the model as the tool result.
4. If `permission == ASK`:
   - If any required permissions are already covered by session rules ⇒ run.
   - Else call `approval_callback(tool_name, args, summary, required_permissions)` ⇒ `(ApprovalResponse.YES | NO, "always" | None)`.
   - On YES + "always", persist new rules; on NO ⇒ skip.

If `bypass_tool_permissions=true` in the active agent profile ⇒ skip the check entirely (this is how `auto-approve` and `chat` agents work).

### 10.4 Discovery

`ToolManager` (`vibe/core/tools/manager.py`) loads tools from:

1. `vibe/core/tools/builtins/` (built-in, always available).
2. Each path in `config.tool_paths`.
3. Project `.vibe/tools/` (if trusted).
4. User `~/.vibe/tools/`.

Loading: import each Python file, enumerate `BaseTool` subclasses, register by canonical snake_case name. Conflicts are resolved by *last writer wins*, with custom paths overriding built-ins.

Discovered MCP servers and Mistral connectors are integrated at the same registry layer (§12).

`enabled_tools` / `disabled_tools` filtering applies last. Supported pattern forms (matched against full tool name):

- Exact (`bash`)
- Glob (`serena_*`)
- Regex (`re:^serena_.*$`) — `fullmatch`, case-insensitive

---

## 11. Built-in Tools — Detailed Specs

All built-ins live in `vibe/core/tools/builtins/`. Each one has both a Python implementation and a markdown system-prompt description (`vibe/core/tools/builtins/prompts/<name>.md`) injected as the tool description sent to the LLM.

Notation: `args` and `result` are pydantic-style schemas; default permission is what ships if no `[tools.<name>]` override exists.

### 11.1 `read_file`
- **Description.** "Read text content from a file."
- **Args.** `path: str` (relative or abs); `offset: int = 0` (line offset); `limit: int | None = None` (line count cap).
- **Result.** `path: str; content: str; offset: int; lines_read: int; was_truncated: bool`.
- **Default permission.** `ALWAYS` — except for paths matching `**/.env`, `**/.env.*`, etc., which require ASK (see `sensitive_patterns` config).
- **Decoding.** Uses `read_safe`: UTF-8 → BOM → locale → `charset_normalizer`; replaces undecodable bytes with U+FFFD by default.
- **Truncation.** Capped at 64 KB by default; sets `was_truncated=true` when truncated.

### 11.2 `write_file`
- **Description.** "Write content to a file, overwriting if it exists."
- **Args.** `path: str; content: str`.
- **Result.** `path: str; num_lines_written: int`.
- **Default permission.** `ASK`. Creates parent dirs as needed. Refuses paths outside the working directory unless explicitly approved (`OUTSIDE_DIRECTORY` scope).

### 11.3 `search_replace`
- **Description.** "Replace literal/regex text in a file in-place."
- **Args.** `path: str; old_text: str; new_text: str; count: int | None`.
- **Result.** `path: str; num_replacements: int; content: str` (post-edit).
- **Default permission.** `ASK`. Supports dry-run via internal API.

### 11.4 `bash`
- **Description.** "Run a one-off bash command and capture its output." Stateless per call.
- **Args.** `command: str; timeout: int | None` (override default).
- **Result.** `command: str; stdout: str; stderr: str; returncode: int`.
- **Default permission.** `ASK`.
- **Config keys.** `max_output_bytes` (16000), `default_timeout` (300 s), `allowlist`, `denylist`, `denylist_standalone`, `sensitive_patterns`.
- **Default allowlist** (Unix). `cd, echo, git diff, git log, git status, tree, whoami, cat, file, find, head, ls, pwd, stat, tail, uname, wc, which`.
- **Default denylist.** `gdb, pdb, passwd, nano, vim, vi, emacs, bash -i, sh -i, …, screen, tmux` and Windows equivalents.
- **Standalone denylist.** `python, python3, ipython, bash, sh, nohup, vi, …, su` (denied only when run with no arguments).
- **Composite-command parsing.** Uses `tree-sitter-bash` to split a multi-command line into individual commands. Each sub-command is independently evaluated:
  - Allowlist hit ⇒ no approval needed (provided no `OUTSIDE_DIRECTORY` issue).
  - Denylist hit ⇒ `NEVER`.
  - `sensitive_patterns` (default `sudo`) ⇒ always ASK regardless of allowlist.
  - `find` with `-exec`/`-execdir`/`-ok`/`-okdir` ⇒ ASK with command pattern.
  - Path references outside CWD ⇒ ASK with `OUTSIDE_DIRECTORY` scope.
- **Process env.** Sets `CI=true`, `NONINTERACTIVE=1`, `NO_TTY=1`, `PAGER=cat`, `GIT_PAGER=cat`, `TERM=dumb`, `LC_ALL=en_US.UTF-8`, `DEBIAN_FRONTEND=noninteractive` (Unix). On Windows: `PAGER=more`, `GIT_PAGER=more`.

### 11.5 `grep`
- **Description.** "Search for a regex pattern in files."
- **Args.** `pattern: str; path: str = "."; recursive: bool = true; max_matches: int = 100`.
- **Result.** `matches: list[Match]` where `Match: {path, line_number, line_text, context_before, context_after}`.
- **Default permission.** `ALWAYS`.
- **Implementation.** Uses `ripgrep` if available; falls back to Python regex.

### 11.6 `task`
- **Description.** "Delegate work to a subagent."
- **Args.** `task: str; agent: str = "explore"` (subagent name).
- **Result.** Tool-dependent; carries the subagent's final assistant message and aggregated tool-call summary.
- **Default permission.** `ALWAYS`.
- **Behaviour.** Spawns a new `AgentLoop` instance configured with the named subagent profile (`agent_type=SUBAGENT`), runs to completion, returns the final result.

### 11.7 `todo`
- **Description.** "Manage a simple checklist."
- **Args.** `item: str; operation: "add" | "check" | "list"`.
- **Result.** `items: list[{id, text, done}]`.
- **Default permission.** `ALWAYS`. State is per-session.

### 11.8 `ask_user_question`
- **Description.** "Ask the user a multi-choice or free-text question."
- **Args.** `questions: list[Question]` where `Question = {question: str, options?: list[{label, description}]}`.
- **Result.** `answers: list[str]` (one per question).
- **Default permission.** `ALWAYS`.
- **TUI.** Multiple questions render as tabs; each gets 2–4 options plus an automatic "Other" free-text option.
- **Disabled in programmatic mode.**

### 11.9 `exit_plan_mode`
- **Description.** "Exit plan agent and switch to a write-capable profile."
- **Args.** none.
- **Default permission.** `ALWAYS`. Calls `ctx.switch_agent_callback("default")`.
- **Disabled by `default`, `accept-edits`, `auto-approve`, `lean` profiles** via `base_disabled = ["exit_plan_mode"]`.

### 11.10 `webfetch`
- **Description.** "Fetch a URL and return its text content, optionally extracting based on a prompt."
- **Args.** `url: str; prompt: str | None`.
- **Result.** `url: str; content: str; extracted_content: str | None`.
- **Default permission.** `ASK` (`URL_PATTERN` scope).
- **Implementation.** httpx + markdownify; optional follow-up LLM extraction when `prompt` is supplied.

### 11.11 `websearch`
- **Description.** "Search the web."
- **Args.** `query: str; num_results: int = 10`.
- **Result.** `query: str; results: list[{url, title, snippet}]`.
- **Default permission.** `ASK`.

### 11.12 `skill`
- **Description.** "Invoke a registered skill (custom prompt + tool allowlist)."
- **Args.** `skill: str; …extra kwargs forwarded to the skill prompt template`.
- **Result.** Tool-dependent.
- **Default permission.** `ALWAYS`. Calls `ctx.skill_manager.invoke(...)`.

---

## 12. MCP & Connector Tools

### 12.1 MCP

[Model Context Protocol](https://modelcontextprotocol.io) servers extend the tool registry at runtime.

Config (discriminated by `transport`):

```toml
# HTTP
[[mcp_servers]]
name = "my_http_server"
transport = "http"
url = "http://localhost:8000"
headers = { Authorization = "Bearer my_token" }
api_key_env = "MY_API_KEY"
api_key_header = "Authorization"
api_key_format = "Bearer {token}"
startup_timeout_sec = 10
tool_timeout_sec = 60
sampling_enabled = true
disabled = false
disabled_tools = []

# Streamable HTTP — same fields as HTTP plus streaming endpoints
[[mcp_servers]]
name = "stream"
transport = "streamable-http"
url = "http://localhost:8001"

# stdio
[[mcp_servers]]
name = "fetch_server"
transport = "stdio"
command = "uvx"            # or list[str]
args = ["mcp-server-fetch"]
env = { DEBUG = "1" }
cwd = "/optional/working/dir"
```

The `name` is normalised to `[a-zA-Z0-9_-]`, max 256 chars. Each discovered remote tool is exposed as `{server_name}_{tool_name}`. Permissions can be set per remote tool via `[tools.<server>_<tool>]` (same shape as built-in tool config).

Sampling: if a server requests `sampling/createMessage`, the agent loop uses the active backend to respond (subject to `sampling_enabled`).

### 12.2 Mistral Connectors

Discovered from the Mistral API. Per-connector config:

```toml
[[connectors]]
name = "github"
disabled = false
disabled_tools = []
```

Connector tools have names like `connector_<name>_<tool>` and are invoked via HTTP calls to the Mistral connector API. Only available when the active provider is Mistral and the API key is present.

---

## 13. Agent Profiles

### 13.1 Profile structure

```python
AgentProfile(
    name: str,
    display_name: str,
    description: str,
    safety: "safe" | "neutral" | "destructive" | "yolo",
    agent_type: "agent" | "subagent",
    overrides: dict,             # deep-merged into VibeConfig
    install_required: bool,
)
```

`overrides` is applied via deep merge of `model_dump()`. The special key `base_disabled` is a list of tools that should be added to `disabled_tools` (i.e. additive denial).

If the active agent has `enabled_tools`, environment-level `disabled_tools` always wins (post-filter).

### 13.2 Built-in profiles

| Name | Safety | Tools | Notes |
|------|--------|-------|-------|
| `default` | neutral | all except `exit_plan_mode` | Ask permission for sensitive tools |
| `plan` | safe | all; `write_file` / `search_replace` permission set to `never` outside `~/.vibe/plans/*` | Read-only mode for exploration |
| `chat` | safe | only `grep`, `read_file`, `ask_user_question`, `task`; `bypass_tool_permissions=true` | Pure conversation mode |
| `accept-edits` | destructive | `write_file`/`search_replace` auto-approved; `exit_plan_mode` disabled | Refactoring workflows |
| `auto-approve` | yolo | All tools auto-approved (`bypass_tool_permissions=true`); `exit_plan_mode` disabled | Use with caution |
| `explore` | safe | `grep`, `read_file` only; system prompt `explore` | **Subagent** (used by `task` tool) |
| `lean` | neutral | Standard set with `bash.default_timeout=1200`; uses `leanstral` model; system prompt `lean` | **Install-required** profile for Lean 4 work |

### 13.3 Custom agents

Place `<name>.toml` in `~/.vibe/agents/` or `.vibe/agents/` (latter requires trust). Example:

```toml
display_name = "Red Team"
description = "Audit for security issues"
safety = "neutral"
agent_type = "agent"
active_model = "mistral-medium-3.5"
system_prompt_id = "redteam"
disabled_tools = ["search_replace", "write_file"]

[tools.bash]
permission = "always"

[tools.read_file]
permission = "always"
```

A custom agent whose name collides with a built-in *replaces* the built-in.

### 13.4 Lifecycle

- `AgentManager.switch_profile(name)` invalidates the cached merged config and the system prompt.
- A profile change mid-turn emits `AgentProfileChangedEvent`.
- The TUI cycles through profiles via `Shift+Tab`.

---

## 14. Skills System

Skills implement the [Agent Skills](https://agentskills.io/specification) v1 format.

### 14.1 SKILL.md

```markdown
---
name: code-review
description: Perform automated code reviews
license: MIT
compatibility: Python 3.12+
user-invocable: true
allowed-tools:
  - read_file
  - grep
  - ask_user_question
metadata:
  category: quality
---

# Body — used as the system prompt augmentation when the skill is invoked
```

Frontmatter schema (`vibe/core/skills/models.py`):

| Field | Type | Default | Constraint |
|-------|------|---------|------------|
| `name` | string | — | `^[a-z0-9]+(-[a-z0-9]+)*$`, 1-64 chars |
| `description` | string | — | 1-1024 chars |
| `license` | string \| null | null | — |
| `compatibility` | string \| null | null | ≤ 500 chars |
| `metadata` | dict[str, str] | `{}` | — |
| `allowed-tools` | string \| list[string] | `[]` | Space-delimited string accepted |
| `user-invocable` | bool | `true` | Controls slash-menu visibility |

### 14.2 Discovery order

1. Built-ins from `vibe/core/skills/builtins/` (currently the internal `vibe` skill — non-user-invocable; documents the CLI to the agent).
2. Each path in `config.skill_paths`.
3. `<project>/.vibe/skills/` (if trusted).
4. `<project>/.agents/skills/` (if trusted) — the Agent Skills standard path.
5. `~/.vibe/skills/`.

First match by `name` wins. `enabled_skills` / `disabled_skills` patterns then filter the result.

### 14.3 Invocation

- User-invocable skills appear in the slash menu as `/<skill-name>`.
- The agent may invoke any skill via the `skill` tool.
- Invocation prepends the skill's prompt body to the next assistant turn and (experimentally) constrains tool use to `allowed-tools`.

---

## 15. Slash Commands

Slash commands trigger meta-actions (configuration, navigation, etc.) — they do NOT go to the LLM. Implementation: `vibe/cli/commands.py`. Each command declares `aliases`, `description`, a handler method on the TUI app, `exits` (whether the app quits), and optional `is_available(ctx)`.

### 15.1 Built-in commands

| Command | Aliases | Args | Behaviour |
|---------|---------|------|-----------|
| `/help` | — | — | Show keybindings + command list |
| `/config` | — | — | Open config editor modal |
| `/model` | — | — | Model picker |
| `/thinking` | — | — | Thinking level picker (per active model) |
| `/reload` | — | — | Re-read config, agents, skills from disk |
| `/clear` | — | — | Clear conversation, preserve session id |
| `/copy` | — | — | Copy last assistant message to clipboard |
| `/log` | — | — | Show current `messages.jsonl` path |
| `/debug` | — | — | Toggle debug console |
| `/compact` | — | `[instructions]` | Run compaction now |
| `/exit` | — | — | Quit |
| `/status` | — | — | Show `AgentStats` summary |
| `/teleport` | — | — | Push session to Vibe Code (cloud); gated by plan |
| `/proxy-setup` | — | `[KEY [value]]` | View / set / unset proxy env vars in `.env` |
| `/resume`, `/continue` | both | — | Open session picker |
| `/rename` | — | — | Set session title |
| `/mcp`, `/connectors` | both | `[server]` | List MCP servers / connectors / their tools |
| `/voice` | — | — | Toggle voice mode and configure |
| `/leanstall` / `/unleanstall` | — | — | Install / remove the Lean 4 agent |
| `/rewind` | — | — | Enter rewind mode (jump to prior message) |
| `/loop` | — | `<interval> <prompt>` \| `list` \| `cancel <id|all>` | Recurring scheduled prompts |
| `/data-retention` | — | — | Show data-retention notice |

Availability filtering: each command may declare `is_available(ctx: CommandAvailabilityContext) -> bool`. The context exposes `vibe_code_enabled`, `is_active_model_mistral`, and the current `plan_info`.

### 15.2 Skill commands

User-invocable skills appear in the same autocomplete menu as built-in commands (typically lower in the list). Invoking `/my-skill <args>` causes the skill's prompt to be appended to the next user message with `<args>` interpolated.

---

## 16. Interactive TUI

The TUI is built on `textual==8.2.4`. The reimplementer can choose any equivalent TUI framework; what follows is the externally observable contract.

### 16.1 Layout

```
┌────────────────────────────────────────────────────────────┐
│ Banner: ascii art, version, model, skills/MCP/connectors    │
│ Chat area (scrollable):                                     │
│   ▸ UserMessage                                             │
│   ▸ AssistantMessage (streamed, supports markdown+ANSI)     │
│   ▸ ReasoningMessage (spinner while thinking)               │
│   ▸ ToolCall / ToolResult cards                             │
│   ▸ BashOutputMessage                                       │
│   ▸ CompactMessage, TeleportMessage, ErrorMessage, …        │
│   ▸ WhatsNewMessage (on first run after upgrade)            │
├────────────────────────────────────────────────────────────┤
│ Narrator status │ Loading area │ FeedbackBar                │
├────────────────────────────────────────────────────────────┤
│ ChatInput:                                                 │
│   - ChatTextArea (text input, ANSI/markdown render)        │
│   - CompletionPopup (overlay for @ paths and / commands)    │
│   - VoiceRecordingIndicator (when voice mode is on)        │
├────────────────────────────────────────────────────────────┤
│ Bottom bar: cwd │ context-progress (tokens / model)        │
└────────────────────────────────────────────────────────────┘
```

### 16.2 Keybindings

Global (Textual `BINDINGS`):

| Key | Action |
|-----|--------|
| Ctrl+C | Interrupt agent, else quit |
| Ctrl+D | Delete-right in input, else quit |
| Ctrl+Z | Suspend |
| Escape | Interrupt running tool / agent |
| Ctrl+O | Toggle tool output panel |
| Ctrl+Y / Ctrl+Shift+C | Copy selected text |
| Shift+Tab | Cycle agent profile |
| Shift+Up / Shift+Down | Scroll chat |
| Ctrl+\ | Toggle debug console |
| Ctrl+T | Toggle todo view (per README) |
| Alt+↑ / Ctrl+P | Rewind: previous message |
| Alt+↓ / Ctrl+N | Rewind: next message |

Input-area only:

| Key | Action |
|-----|--------|
| Enter | Submit |
| Ctrl+J or Shift+Enter | Newline |
| Ctrl+G | Open `$VISUAL`/`$EDITOR` (fallback `nano`) on a temp `.md` file |
| `!<cmd>` | Run `<cmd>` directly in the shell, bypassing the agent |
| `/<…>` prefix | Activates slash-command autocomplete |
| `@<…>` prefix | Activates file-path autocomplete |

Voice mode (when active):

| Key | Action |
|-----|--------|
| Ctrl+R | Start recording |
| Any key | Stop recording |
| Escape / Ctrl+C | Cancel recording |

### 16.3 Autocomplete

- **Slash commands** — synchronous; queries the `CommandRegistry`. Up to 10 suggestions.
- **`@` paths** — runs in a dedicated 1-worker thread (`path-completion`). Fuzzy match against an indexed file tree (built via `vibe/core/autocompletion/file_indexer/`). Skips hidden directories and `.gitignore` entries.

### 16.4 Notifications

`NotificationPort` interface; default adapter:
- Triggers only when the terminal does NOT have focus.
- Sets terminal title and rings the bell.
- Throttled to ≤ 1 notification/second.
- Contexts: `ACTION_REQUIRED` (awaiting approval/question) and `COMPLETE` (turn done).

### 16.5 Rewind

`/rewind` (or Alt+↑) enters rewind mode: highlights a prior user message; user picks one of two actions:
- *Rewind & restore input*: trim history, repopulate input with that message's text.
- *Rewind without restore*: trim history, leave input empty.

### 16.6 Turn summary

Background task per turn: collects the user request + assistant response + any error, calls `backend.complete` with the `turn_summary.md` utility prompt (max 512 tokens by default), surfaces the result via `on_summary` (used to set session titles and for nuage workflows).

### 16.7 Plan-offer banner

On startup the TUI calls `whoami` on the Mistral API to determine the user's plan (`UNAUTHORIZED`, `UNKNOWN`, `API` paid/free, `CHAT` pro/free, `MISTRAL_CODE` enterprise/free). The banner shows plan label; `/teleport` is gated on `is_teleport_eligible`.

### 16.8 Input history

Stored at `~/.vibe/vibehistory` as one JSON-encoded entry per line. Capped at 100 entries (configurable). Atomic write via `tempfile` + `os.replace`. Navigation: Up/Down in the input box; first Up captures the current draft so it can be restored on the way back.

### 16.9 External editor

Ctrl+G: spawn `$VISUAL || $EDITOR || nano` on a temp `.md` file pre-populated with current input. On editor exit, replace input with file content (unless unchanged).

---

## 17. Programmatic Mode & Output Formats

Enter via `vibe -p [PROMPT]` (or piped stdin). Behaviour:

- Default agent is `auto-approve` regardless of `default_agent`.
- `ask_user_question` and `exit_plan_mode` tools are **disabled** (cannot interact).
- Supports `--max-turns`, `--max-price`, `--enabled-tools`, `--output`.
- Honours `--continue` / `--resume` to bootstrap from prior messages.

### 17.1 Output formats

`OutputFormat` enum: `TEXT`, `JSON`, `STREAMING`.

**TEXT** (default):
- Accumulates events.
- Prints teleport progress lines if applicable.
- At end, prints the final assistant message content (or teleport URL).

**JSON**:
- Buffers all `LLMMessage`s.
- At end, writes a single pretty-printed JSON array to stdout.

**STREAMING** (NDJSON):
- On each `on_message_added(message)`, writes one JSON object + newline immediately.
- Each line is a serialized `LLMMessage` (system, user, assistant, or tool).

The shape of an `LLMMessage` JSON object (see `core/types.py`):

```json
{
  "role": "system" | "user" | "assistant" | "tool",
  "content": "string or null",
  "injected": false,
  "reasoning_content": "string or null",
  "reasoning_state": ["…"] | null,
  "reasoning_signature": "string or null",
  "reasoning_message_id": "uuid or null",
  "tool_calls": [
    {
      "id": "string",
      "index": 0,
      "type": "function",
      "function": { "name": "string", "arguments": "JSON-encoded string" }
    }
  ] | null,
  "name": "string or null",
  "tool_call_id": "string or null",
  "message_id": "uuid or null"
}
```

---

## 18. ACP (Agent Client Protocol) Bridge

The `vibe-acp` binary implements the JSON-RPC [Agent Client Protocol](https://agentclientprotocol.com) v0.9.0 over stdio. It is the integration surface for IDEs.

### 18.1 Methods (agent ← client)

| Method | Purpose |
|--------|---------|
| `initialize` | Capability negotiation |
| `authenticate` | Trigger interactive auth |
| `new_session` | Create session with a CWD |
| `load_session` | Resume by id |
| `fork_session` | Branch from an existing session |
| `list_sessions` | Enumerate available sessions |
| `prompt` | Send user content to agent; streams `session_update`s back |
| `cancel` | Cancel the active prompt |
| `set_session_mode` | Switch agent profile |
| `set_session_model` | Switch active model |
| `set_config_option` | Update a config value (layered) |
| `close_session` | Tear down |
| `ext_method` | Vibe extension: `session/set_title` |
| `ext_notification` | Vibe extension: `telemetry/send` |

### 18.2 Session updates (agent → client)

`SessionUpdate` is a tagged union:
- `agent_message_chunk` — streamed assistant text
- `tool_call` — `ToolCallStart` (id, name, kind, arguments)
- `tool_call_update` — `ToolCallProgress` (status: pending → in_progress → success/failed/cancelled; optional content)
- `usage_update` — token counts
- `config_option_update` — server-side config changed
- `session_info_update` — id, title, cwd

Tool kind mapping (used by IDEs to render appropriate UI):

| Vibe tool | ACP kind |
|-----------|----------|
| `read_file` | `read` |
| `bash` | `execute` |
| `write_file`, `search_replace` | `edit` |
| `grep`, `websearch` | `search` |
| `webfetch` | `fetch` |
| `skill` | `read` |

### 18.3 Bash via ACP terminals

The ACP `bash` tool does not execute locally. Instead it calls `client.create_terminal(command, …)`, waits for exit via `client.wait_for_terminal_exit`, fetches output via `client.terminal_output`, and releases the terminal. This delegates execution to the IDE's integrated terminal so the user sees and can interrupt the process.

### 18.4 Error codes

Vibe-specific JSON-RPC error codes:

| Code | Name |
|------|------|
| -31001 | RATE_LIMITED |
| -31002 | CONFIGURATION_ERROR |
| -31003 | CONVERSATION_LIMIT |
| -31004 | CONTEXT_TOO_LONG |
| -32000 | UNAUTHENTICATED |
| -32601 | METHOD_NOT_FOUND |
| -32602 | INVALID_PARAMS |
| -32603 | INTERNAL_ERROR |

### 18.5 Available ACP commands

A subset of the TUI slash commands is exposed via the ACP `availableCommands` mechanism: `help`, `compact`, `reload`, `log`, `proxy-setup`, `leanstall`, `unleanstall`, `data-retention`. Skills with `user-invocable=true` are advertised alongside them.

### 18.6 Wire logging

When `VIBE_ACP_LOGGING_ENABLED=true`, all incoming/outgoing messages are appended to `~/.vibe/logs/acp/messages.jsonl`. Each line: `{timestamp, direction: "in"|"out", session_id, message: {…}}`.

---

## 19. Voice & Audio (TTS / STT / Narrator)

### 19.1 Pipeline

```
mic → AudioRecorder ───► VoiceManager ─► TranscribeClient ─► insert text into ChatInput
                                                             (streamed deltas)

assistant turn end ─► NarratorManager ─► summarize turn ─► TTSClient ─► AudioPlayer ─► speaker
```

### 19.2 STT (transcribe)

- Default: Mistral realtime transcription over a websocket (`wss://api.mistral.ai`).
- Model: `voxtral-mini-transcribe-realtime-2602` (alias `voxtral-realtime`).
- Audio format: PCM s16le, sample rate per model config (default 16000 Hz), mono.
- Stream events: `TranscribeSessionCreated`, `TranscribeTextDelta` (incremental text), `TranscribeDone`, `TranscribeError`.
- `VoiceManager` exposes states `IDLE`, `RECORDING`, `FLUSHING` and a `peak` float (0–1).

### 19.3 TTS

- Default: Mistral speech API.
- Model: `voxtral-mini-tts-latest` (alias `voxtral-tts`).
- Voice: `gb_jane_neutral` (default).
- Response format: WAV (or MP3 per config).
- `NarratorManager` is opt-in (`narrator_enabled = true`). On each turn-end it summarises and speaks the response.

### 19.4 Audio I/O

- `sounddevice` is the cross-platform backend.
- Recorder supports buffer mode (returns WAV bytes) or stream mode (async generator).
- Player decodes WAV header and streams PCM via `RawOutputStream` callback (low-latency).
- Exceptions surface as typed errors: `NoAudioInputDeviceError`, `NoAudioOutputDeviceError`, `AudioBackendUnavailableError`, `AlreadyRecordingError`, `AlreadyPlayingError`, `IncompatibleSampleRateError`, `UnsupportedAudioFormatError`.

### 19.5 Telemetry events emitted

- `vibe.audio.transcription.start` / `.done` / `.cancel_recording` / `.error`
- `vibe.read_aloud.requested` / `.play_started` / `.ended`
- Carry `recording_id`, `duration`, `transcript_length`, `time_to_first_read_s`.

---

## 20. Sessions & Persistence

### 20.1 Session ID

UUID-shaped string `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` where the **last segment (12 hex)** is a "stable suffix" preserved across compact / fork / rewind operations so the same logical thread can be tracked. Helper: `shorten_session_id` returns the last 8 chars.

### 20.2 On-disk layout

```
$VIBE_HOME/logs/session/
└── session_20260514_153012_abcdef12/
    ├── meta.json           # SessionMetadata
    └── messages.jsonl      # one LLMMessage JSON object per line
```

The folder name is `<session_prefix>_<UTC timestamp YYYYMMDD_HHMMSS>_<short_id>`.

### 20.3 SessionMetadata schema

```json
{
  "session_id": "uuid",
  "parent_session_id": "uuid or null",
  "start_time": "ISO-8601 UTC",
  "end_time": "ISO-8601 UTC or null",
  "git_commit": "string or null",
  "git_branch": "string or null",
  "environment": {
    "working_directory": "/abs/path",
    "user_agent": "vibe/2.9.6",
    "...": "..."
  },
  "username": "string",
  "loops": [
    { "id": "...", "interval_seconds": 600, "prompt": "...",
      "next_fire_at": 0.0, "created_at": 0.0 }
  ],
  "title": "string or null",
  "title_source": "auto" | "manual"
}
```

### 20.4 Validity check

A session is "valid" iff (a) both files exist, (b) `meta.json` decodes to a dict, (c) `messages.jsonl` has at least one line and every line is a JSON object. When resuming with a `working_directory` filter, the session's `environment.working_directory` must equal the current `cwd`.

### 20.5 Continue / Resume

- `--continue` ⇒ `SessionLoader.latest_session(session_dirs, working_directory=cwd)` — newest valid session whose cwd matches.
- `--resume ABC123` ⇒ matches by short or full id; opens picker if multiple.

`resume_sessions.py` adds remote (`nuage`) sessions to the picker when Vibe Code is reachable.

### 20.6 Migration

`session_migration.py` upgrades pre-2.0 single-file `session_*.json` files into the per-directory format.

### 20.7 Logging behaviour

- `[session_logging] enabled = true` is the default. When false, all session APIs are no-ops and `--continue`/`--resume` are unavailable.
- Auto-saves a metadata snapshot every time the loop returns to "waiting for user".
- Stable suffix is preserved through `/compact` so consumers can chain summaries.

---

## 21. System Prompts & AGENTS.md Discovery

### 21.1 Resolution

The active system prompt is the result of:

1. Look up `system_prompt_id` in the project prompts dirs (each trusted project + harness layers).
2. Then in user prompts dirs (`~/.vibe/prompts/`).
3. Then in the built-in `SystemPrompt` enum (mapped from `vibe/core/prompts/*.md`).
4. Else: `MissingPromptFileError`.

The resolved markdown is then *augmented* (unless disabled by `include_*` config flags) with:
- Tool catalogue.
- Active model info.
- Available skills.
- Available agents (`agents_doc.md`).
- Project context (cwd, git branch/commit, recent commits — bounded by `project_context.timeout_seconds`).
- OS / shell info.
- `AGENTS.md` contents (see below).

### 21.2 Built-in prompts

| File | Purpose |
|------|---------|
| `cli.md` | Main agent persona — 110 lines, strict verbosity rules |
| `explore.md` | Explore subagent — read-only investigator |
| `lean.md` | Lean 4 specialist agent persona |
| `compact.md` | Utility prompt sent to the compaction LLM call |
| `turn_summary.md` | Utility prompt for turn-summary generation |
| `project_context.md` | Template for cwd / git info injection |
| `agents_doc.md` | Auto-injected catalogue of available agents |
| `tests.md` | Testing-specific guidance |
| `dangerous_directory.md` | Warning shown when running in sensitive dirs |

### 21.3 AGENTS.md

Markdown files named exactly `AGENTS.md` are discovered:

1. `~/.vibe/AGENTS.md` — user-level baseline.
2. Walk upward from CWD to the trust root, collecting any `AGENTS.md` files encountered.

All are concatenated; closer-to-CWD files take priority (their content appears later, after the global file). Only loaded when the containing folder is trusted. AGENTS.md is treated as *additional instructions* prepended/appended to the active system prompt — it does **not** replace it.

### 21.4 Custom prompts

Place `<id>.md` in `~/.vibe/prompts/` (or `.vibe/prompts/`) and set `system_prompt_id = "<id>"` in config. This *fully replaces* the built-in `cli.md` content.

---

## 22. Hooks (Experimental)

Gated by `enable_experimental_hooks = true`.

### 22.1 Configuration

User scope: `~/.vibe/hooks.toml`. Project scope: `<project>/.vibe/hooks.toml` (requires trust).

```toml
[[hooks]]
name = "post-turn-linter"
type = "post_agent_turn"
command = "ruff check ."
timeout = 30.0
description = "Run linter after every turn"
```

Duplicate `name`s across scopes are rejected as issues (validation runs at load).

### 22.2 Execution

- Trigger: `POST_AGENT_TURN` — fires after every assistant turn completes.
- Each hook is invoked as a subprocess (`shlex.split(command)`), receiving a JSON payload on stdin (`HookInvocation`: `session_id`, `transcript_path`, `cwd`, `hook_event_name`).
- Result: `HookExecutionResult` (`exit_code`, `stdout`, `stderr`, `timed_out`).
- Exit code `2` means *retry*; the loop will inject a synthetic user message with the hook's stdout and re-run the turn, up to 3 times.

### 22.3 UI events

`HookRunStartEvent`, `HookRunEndEvent`, `HookStartEvent(hook_name)`, `HookEndEvent(hook_name, status: ok|warning|error, content)`.

---

## 23. Onboarding & Authentication

### 23.1 First-run

`vibe/setup/onboarding/` is a small Textual app launched when:
- `--setup` is passed, or
- `VibeConfig.load()` raises `MissingAPIKeyError`.

Screens:
1. `welcome.py` — intro + ASCII logo.
2. `api_key.py` — pick the provider (Mistral by default), then either:
   - paste an API key, or
   - **browser sign-in** flow (Mistral providers only).

Result: `None` (cancelled) | `"completed"` | error string. On success, the API key is appended to `~/.vibe/.env`.

### 23.2 Browser sign-in

`vibe/setup/auth/browser_sign_in.py`:
1. Generate PKCE verifier+challenge.
2. POST to `<browser_auth_api_base_url>/cli/start` with challenge; receive `code` + `auth_url`.
3. Open `auth_url` in the user's browser.
4. Poll `<browser_auth_api_base_url>/cli/exchange` with the code + verifier (exponential backoff, capped wait).
5. Receive an API key; persist via dotenv.

The gateway interface (`BrowserSignInGateway`) abstracts the HTTP calls; `HttpBrowserSignInGateway` is the default implementation.

### 23.3 API key resolution order

For any provider, the active key is the first of:

1. `os.environ[provider.api_key_env_var]`.
2. The same env var after `~/.vibe/.env` has been loaded by `load_dotenv_values`.

(No keyring is consulted in the current implementation despite the `keyring` dependency.)

---

## 24. Telemetry & Tracing

### 24.1 Telemetry (`vibe/core/telemetry/`)

- Endpoint: `<mistral api_base>/v1/datalake/events`.
- Gated by `enable_telemetry=true` AND a valid Mistral API key.
- Async batched, non-blocking.
- Payload includes session/parent_session ids, entrypoint metadata (`{name: "vibe"|"vibe-acp", version: "2.9.6", …}`), and event-specific properties.
- Event names follow `vibe.<surface>.<verb>` (e.g. `vibe.audio.transcription.start`, `vibe.at_mention_inserted`).

### 24.2 OpenTelemetry (`vibe/core/tracing.py`)

- Disabled unless `enable_otel = true`.
- Uses OTLP HTTP/protobuf exporter (default Mistral endpoint `<server>/telemetry`).
- If `otel_endpoint` is explicitly set, authentication is the user's responsibility (`OTEL_EXPORTER_OTLP_*` env vars).
- Default span: `agent_span()` per turn, attributes:
  - `gen_ai.operation.name = "invoke_agent"`
  - `gen_ai.provider.name = "mistral_ai"`
  - `gen_ai.request.model = <name>`
  - `gen_ai.session.id`, custom `vibe.session.id`
- Spans capture exceptions and set status accordingly.

### 24.3 Log file

`~/.vibe/logs/vibe.log`. Rotating, format: ISO timestamp, ppid, pid, level, message. Format keys are positional `%s` (so log records are grep-friendly). Default level `WARNING`, override via `LOG_LEVEL`.

---

## 25. Update Notifier

- Gateway protocol: `UpdateGateway.fetch_update()` returns the latest version or None. Adapters: `PyPIUpdateGateway`, `GitHubUpdateGateway`.
- Cache: `~/.vibe/cache.toml` (key: `update_cache`). Schema: `latest_version`, `stored_at_timestamp`, `seen_whats_new_version`. TTL 24 h.
- Version parsing via `packaging.version.Version` (tolerates dashes as prereleases).
- `whats_new.md` is bundled with the package (`vibe/whats_new.md`); shown once after the version on disk advances, by storing the seen version in the cache.
- Auto-update (gated by `enable_auto_update=true`) runs `uv tool upgrade mistral-vibe` in the background.

---

## 26. Auxiliary Modules

### 26.1 Scratchpad (`vibe/core/scratchpad.py`)

Per-session temp dir (`vibe-scratchpad-<short_id>-<tmp>`) for intermediate files. APIs:

- `init_scratchpad(session_id)` — idempotent create.
- `get_scratchpad_dir(session_id)` — lookup.
- `is_scratchpad_path(path)` — validates a path is inside *any* active scratchpad (uses `.resolve()` to defeat symlink tricks). The `bash` tool consults this to allow path operations against scratchpads even though they sit outside CWD.

### 26.2 Plan session (`vibe/core/plan_session.py`)

Manages per-session markdown files in `~/.vibe/plans/`. Used by the `plan` agent profile (writes/reads only into `PLANS_DIR.path / "*"`).

### 26.3 Teleport (`vibe/core/teleport/`) & Nuage (`vibe/core/nuage/`)

"Teleport" pushes the current local session + uncommitted changes to Mistral's *Vibe Code* cloud service (a workflow on the Nuage runtime). Flow:

1. `TeleportCheckingGitEvent`.
2. If unauthenticated ⇒ `TeleportAuthRequiredEvent(oauth_url)`.
3. If unpushed commits ⇒ `TeleportPushRequiredEvent(unpushed_count)` → push.
4. `TeleportPushingEvent`, `TeleportWaitingForGitHubEvent`, `TeleportFetchingUrlEvent`.
5. `TeleportCompleteEvent(url)`.

Gated by plan eligibility (`is_teleport_eligible`).

Nuage exposes a `WorkflowsClient` (HTTP + SSE) for starting / streaming / signalling workflows on `vibe_code_base_url` (default `https://api.mistral.ai`), task queue `shared-vibe-nuage`.

### 26.4 Proxy setup (`vibe/core/proxy_setup.py`)

Reads/writes `HTTP_PROXY`, `HTTPS_PROXY`, `ALL_PROXY`, `NO_PROXY`, `SSL_CERT_FILE`, `SSL_CERT_DIR` in `~/.vibe/.env`. Drives the `/proxy-setup` command.

### 26.5 Data retention (`vibe/core/data_retention.py`)

One-shot informational message about training data usage; user is directed to `https://admin.mistral.ai/plateforme/privacy` to manage settings.

### 26.6 Log reader (`vibe/core/log_reader.py`)

Reads `~/.vibe/logs/vibe.log` with pagination and (optional) tail-following for the debug console.

---

## 27. Event Catalog & Data Models

All events derive from `BaseEvent` (`vibe/core/types.py`).

```text
BaseEvent
├── UserMessageEvent          { content, message_id }
├── AssistantEvent            { content, message_id, stopped_by_middleware }
├── ReasoningEvent            { content, message_id }
├── ToolCallEvent             { tool_call_id, tool_name, tool_class, tool_call_index, args }
├── ToolResultEvent           { tool_call_id, tool_name, tool_class, result, error,
│                               skipped, skip_reason, cancelled, duration }
├── ToolStreamEvent           { tool_call_id, tool_name, message }
├── WaitingForInputEvent      { task_id, label, predefined_answers }
├── CompactStartEvent         { current_context_tokens, threshold, tool_call_id }
├── CompactEndEvent           { old_context_tokens, new_context_tokens, summary_length,
│                               old_session_id, new_session_id, tool_call_id }
├── AgentProfileChangedEvent  { agent_name }
└── HookEvent
    ├── HookRunStartEvent
    ├── HookRunEndEvent
    ├── HookStartEvent        { hook_name }
    └── HookEndEvent          { hook_name, status, content }
```

Teleport adds its own family (`TeleportCheckingGitEvent`, …).

Key domain models:

- `LLMMessage` — see §17.1 for JSON shape.
- `LLMUsage`: `{ prompt_tokens, completion_tokens }`.
- `LLMChunk`: `{ message, usage, correlation_id }` — supports `+` for streaming accumulation.
- `AgentStats` — §8.5.
- `SessionMetadata` — §20.3.
- `AgentProfile` — §13.1.
- `SkillInfo` — §14.
- `RequiredPermission`, `PermissionContext`, `ApprovedRule` — §10.3.
- `ScheduledLoop`: `{ id, interval_seconds, prompt, next_fire_at, created_at }`.

---

## 28. Reproduction Notes for Reimplementers

### 28.1 Mandatory observable behaviours

If you intend the new build to be drop-in replaceable for Vibe (e.g. for an IDE that already integrates the ACP server), you MUST preserve:

- `~/.vibe/` layout (paths in §6).
- Config TOML field names, types, and defaults (§5).
- `LLMMessage` JSON shape — IDE integrations parse this directly.
- ACP method names, error codes, and `SessionUpdate` tagged-union members (§18).
- Tool *names* and their JSON arg schemas, since these are quoted into prompts and IDE UIs.
- Trust prompts before reading any project-local file.
- `--continue` / `--resume` semantics including the "stable suffix" of session ids.

### 28.2 Stack-agnostic choices

Things that can change freely:

- TUI framework — any with chat-style rendering, autocomplete overlays, and async support will do.
- HTTP client — must support streaming, custom CA bundles, and proxy env vars.
- Async runtime — `asyncio` is convenient because of structured cancellation; equivalents exist (Kotlin coroutines, Go contexts, Rust tokio, …).
- Subprocess library — must support timeouts, output capture, and SIGINT propagation on cancel.
- TOML / JSON / JSONL parsers — standard.
- Tree-sitter bash — used solely to split composite commands for the bash tool's permission analysis. A simpler tokenizer would suffice if it handles `&&`, `;`, `|`, subshells, and quoted strings.

### 28.3 Recommended build order

1. **Paths + config** — get `VibeConfig.load()` reading `~/.vibe/config.toml` and env vars.
2. **Mistral backend** — minimal non-streaming `complete()`.
3. **Built-in tools** — start with `read_file`, `grep`, `bash`.
4. **Agent loop** — a turn-by-turn loop without middleware.
5. **Permission model** — wire `ASK` callback to a stub that always says yes.
6. **TUI MVP** — message list + input box.
7. **Sessions** — write `messages.jsonl` + `meta.json`; implement `--continue`.
8. **Streaming** — add `complete_streaming` + chunk accumulation.
9. **Slash commands** — `/help`, `/clear`, `/exit`, `/status`.
10. **Programmatic mode** — `-p` + `--output text|json|streaming`.
11. **MCP** — wire stdio transport first; HTTP later.
12. **ACP** — last, since it builds on a stable agent-loop contract.
13. **Auxiliaries** — voice, telemetry, teleport, etc.

### 28.4 Test parity targets

The reference repo has 267 test files across these areas: agent loop, tool execution, permission, autocompletion, ACP, audio, banner, browser sign-in, CLI, core, e2e (programmatic-mode end-to-end runs), MCP, narrator, onboarding, plan offer, session, skills, snapshots (Textual UI), TTS, transcribe, update notifier, voice manager. A reimplementation that meets the same test scenarios is conformant. Notable invariants to enforce in tests:

- Message merging semantics on streamed deltas (`LLMMessage.__add__`).
- Tool permission flow including session-rule promotion ("approve always").
- Auto-compact preserves the stable session-id suffix.
- Programmatic mode auto-approves and disables `ask_user_question` / `exit_plan_mode`.
- Bash composite-command splitting honours denylist on every sub-command.
- Trust prompt fires exactly once per untrusted directory, then persists.
- `--continue` matches a prior session by the CWD recorded in `meta.json.environment.working_directory`.
- Session messages survive a compaction with a single summary entry plus the system prompt.

### 28.5 Out-of-scope / vendor-specific

These pieces are tightly coupled to Mistral infrastructure and may be replaced wholesale:

- Telemetry endpoint and event names.
- OTel default endpoint and headers.
- Browser sign-in URLs.
- Teleport / Nuage workflows.
- Plan eligibility checks.
- The `vibe` self-doc built-in skill (it documents this exact product; rewrite for your own).

---

*End of specification.*
