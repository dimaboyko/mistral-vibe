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

**Appendices**
- [A. Built-in Tool Schemas (Exact)](#appendix-a--built-in-tool-schemas-exact)
- [B. LLM Wire Format & Tool-Call Assembly](#appendix-b--llm-wire-format--tool-call-assembly)
- [C. TUI Subsystems Reference](#appendix-c--tui-subsystems-reference)
- [D. Auxiliary Subsystems & Exact Wire / Text Content](#appendix-d--auxiliary-subsystems--exact-wire--text-content)
- [E. Errata & Confirmations Against §1–§28](#appendix-e--errata--confirmations-against-1-28)

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

**File format** (`~/.vibe/trusted_folders.toml`) — TOML with two top-level **arrays of absolute path strings**:

```toml
trusted = [
  "/home/user/project1",
  "/home/user/project2",
]
untrusted = [
  "/home/user/sketchy",
]
```

Created and updated by `TrustedFoldersManager` in `vibe/core/trusted_folders.py`. Empty file is bootstrapped on first read if missing or malformed.

**Trustable file detection** (`find_trustable_files(path)`): returns the set of *relative* paths that, if present in `path`, would cause Vibe to read user-supplied configuration. Triggers the trust prompt when the directory is undecided. The detector reports:
- `AGENTS.md` (file in `path`).
- Any directory returned by `walk_local_config_dirs(path).config_dirs` — i.e. a found `.vibe/` (containing config / prompts / tools / skills / agents) or `.agents/` (containing `skills/`).

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

---

# Appendices — Deep-Dive Reference

The following appendices give wire-level / line-level detail for areas that the body of the spec describes structurally. They are intended to remove every remaining ambiguity for a reimplementer. Where a numeric default or string literal is given, it is normative; a port is non-conformant if it diverges silently.

## Appendix A — Built-in Tool Schemas (Exact)

Each subsection gives the canonical Pydantic schema, defaults, errors, and prompt-file summary for one built-in tool. All tools live in `vibe/core/tools/builtins/`. Tool prompt files (`vibe/core/tools/builtins/prompts/<name>.md`) are sent to the LLM as the tool description body.

### A.1 `read_file`

- **LLM description (`ClassVar`).** "Read a text file (encoding detected safely), returning content from a specific line range. Reading is capped by a byte limit for safety."
- **Args (`ReadFileArgs`).** `path: str`, `offset: int = 0` (0-indexed inclusive line), `limit: int | None = None`.
- **Result (`ReadFileResult`).** `path: str`, `content: str`, `offset: int`, `lines_read: int`, `was_truncated: bool`.
- **Config (`ReadFileToolConfig`).** `permission = ALWAYS`, `sensitive_patterns = ["**/.env", "**/.env.*"]`, `max_read_bytes = 64_000`.
- **State.** `injected_agents_md: set[str]` — tracks which `AGENTS.md` files have been injected via `get_result_extra()` to avoid duplicates.
- **Prompt summary.** Use `offset`/`limit` for large files; avoid 3+ calls on the same file; never read model checkpoints or binary files.
- **Errors (verbatim).** `"Path cannot be empty"`, `"Offset cannot be negative"`, `"Limit, if provided, must be a positive number"`, `"Security error: Cannot read path '...' outside of the project directory"`, `"File not found at: ..."`, `"Path is a directory, not a file: ..."`, `"Error reading {file_path}: {exc}"`.
- **Permission resolution.** `resolve_file_tool_permission(path, allowlist, denylist, sensitive_patterns)` — the same helper used by `write_file`, `search_replace`, and `grep`. Sensitive patterns force `ASK` even when default is `ALWAYS`.
- **Side effect.** On success, may inject `AGENTS.md` content found in subdirectories of `path` via `get_result_extra()` wrapped in warning tags (one-shot per file path per session).
- **Display.** `format_call_display`: `Reading {path}` (with `(from line X, limit Y)` and `(scratchpad)` suffixes when applicable). `get_result_display`: `Read N line(s) from {filename}` (`(truncated)` suffix on truncation). `get_status_text`: `Reading file`.

### A.2 `write_file`

- **LLM description.** "Create or overwrite a UTF-8 file. Fails if file exists unless 'overwrite=True'."
- **Args.** `path: str`, `content: str`, `overwrite: bool = False`.
- **Result.** `path: str`, `bytes_written: int`, `file_existed: bool`, `content: str`.
- **Config.** `permission = ASK`, `sensitive_patterns = ["**/.env", "**/.env.*"]`, `max_write_bytes = 64_000`, `create_parent_dirs = True`.
- **Errors.** `"Path cannot be empty"`, `"Content exceeds {max_write_bytes} bytes limit"`, `"File '{path}' exists. Set overwrite=True to replace."`, `"Parent directory does not exist: {parent}"`, `"Error writing {file_path}: {e}"`.
- **Snapshot.** Calls `get_file_snapshot_for_path()` so the rewind feature can restore overwritten content.
- **Display.** `Writing {path}` (with `(overwrite)` and `(scratchpad)` flags); result `Created` / `Overwritten {filename}`.

### A.3 `search_replace`

- **LLM description.** Tells the model to send one or more SEARCH/REPLACE blocks:

  ```
  <<<<<<< SEARCH
  [text to find — exact whitespace]
  =======
  [replacement text]
  >>>>>>> REPLACE
  ```

  Five or more `=` characters are required between SEARCH and REPLACE. The body is parsed by two regexes (`SEARCH_REPLACE_BLOCK_RE` and `SEARCH_REPLACE_BLOCK_WITH_FENCE_RE`), so blocks may appear inside or outside ` ``` ` fences.
- **Args.** `file_path: str`, `content: str` (one or more blocks).
- **Result.** `file: str`, `blocks_applied: int`, `lines_changed: int`, `content: str` (echo), `warnings: list[str]`.
- **Config.** `permission = ASK`, `sensitive_patterns = same as write`, `max_content_size = 100_000`, `create_backup = False`, `fuzzy_threshold = 0.9`.
- **Match semantics.** Literal exact text match (whitespace included). Only the first occurrence is replaced per block — multiple matches produce a warning. On miss, the tool runs a `difflib.SequenceMatcher` window over the file and surfaces the closest match (≥ `fuzzy_threshold`) plus a unified diff truncated to 2000 chars in the error message.
- **Errors.** `"File path cannot be empty"`, `"Content size ({size} bytes) exceeds max_content_size ({max} bytes)"`, `"Empty content provided"`, `"File does not exist: {path}"`, `"Path is not a file: {path}"`, `"No valid SEARCH/REPLACE blocks found..."`, `"SEARCH/REPLACE blocks failed:\n{errors}"`, `"Permission denied reading file: {path}"`, `"OS error reading/writing..."`.

### A.4 `bash`

- **LLM description.** "Run a one-off bash command and capture its output."
- **Args.** `command: str`, `timeout: int | None`.
- **Result.** `command: str`, `stdout: str` (truncated), `stderr: str` (truncated), `returncode: int`.
- **Config.** `permission = ASK`, `max_output_bytes = 16_000`, `default_timeout = 300`, `allowlist`/`denylist`/`denylist_standalone` (defaults below), `sensitive_patterns = ["sudo"]`.
- **Default allowlist** (Unix): `cd, echo, git diff, git log, git status, tree, whoami, cat, file, find, head, ls, pwd, stat, tail, uname, wc, which`. Windows: `cd, echo, git diff, git log, git status, tree, whoami, dir, findstr, more, type, ver, where`.
- **Default denylist** (Unix): `gdb, pdb, passwd, nano, vim, vi, emacs, bash -i, sh -i, zsh -i, fish -i, dash -i, screen, tmux`. Windows adds `cmd /k, powershell -NoExit, pwsh -NoExit, notepad`.
- **`denylist_standalone`** (Unix): `python, python3, ipython, bash, sh, nohup, vi, vim, emacs, nano, su` — denied only when invoked with no arguments.
- **Composite-command parsing.** `_extract_commands` walks the tree-sitter-bash AST collecting nodes of type `command`. Each child node of type `command_name | word | string | raw_string | concatenation` is concatenated to form the sub-command string. Pipes, `&&`, `||`, semicolons, and subshells produce multiple sub-commands.
- **Permission flow** (`resolve_permission`):
  1. Windows ⇒ return `None` (no analysis).
  2. `_resolve_guardrail_permission`: per-sub-command — denylist match ⇒ `NEVER` with reason; standalone-denylist ⇒ `NEVER`; `find … -exec/-execdir/-ok/-okdir` ⇒ adds a `COMMAND_PATTERN` `RequiredPermission`.
  3. `_collect_outside_dirs`: for sub-commands whose first token is in `_PATH_COMMANDS` (`cat, cd, chmod, chown, cp, head, ls, mkdir, mv, rm, stat, tail, touch, wc`), inspect each path-like argument (starts with `/`, `~`, `.`, or contains `/`). If it resolves outside the workdir and is not inside a scratchpad ⇒ collect parent dir.
  4. `_is_unconditionally_allowed`: returns False if any sub-command matches `sensitive_patterns`; returns False if config permission ≠ `ALWAYS`; otherwise True iff every sub-command is allowlisted AND `outside_dirs` is empty.
  5. `_build_required_permissions`: builds `COMMAND_PATTERN` requirements for sensitive and non-allowlisted sub-commands (via `build_session_pattern`) and `OUTSIDE_DIRECTORY` requirements (`{dir}/*` glob) for collected outside dirs.
- **Subprocess.** `asyncio.create_subprocess_shell(stdin=DEVNULL, env=_get_base_env(), executable=os.environ.get("SHELL"), start_new_session=True (Unix))`. Output decoded with platform-specific encoding (UTF-8 Unix; `cp{OEMCP}` Windows).
- **Errors.** `"Command timed out after {timeout}s: {command!r}"`, `"Command failed: {command!r}\nReturn code: {returncode}\nStderr: …\nStdout: …"`, `"Error running command {command!r}: {exc}"`.

### A.5 `grep`

- **LLM description.** "Recursively search files for a regex pattern using ripgrep (rg) or grep. Respects .gitignore and .codeignore files by default when using ripgrep."
- **Args.** `pattern: str`, `path: str = "."`, `max_matches: int | None`, `use_default_ignore: bool = True`.
- **Result.** `matches: str` (newline-joined `path:line:content`), `match_count: int`, `was_truncated: bool`. Property `parsed_matches: list[GrepMatch]` parses the string back (handles Windows drive letters such as `C:\repo\file.py:10:…`).
- **Config.** `permission = ALWAYS`, `sensitive_patterns = ["**/.env", "**/.env.*"]`, `max_output_bytes = 64_000`, `default_max_matches = 100`, `default_timeout = 60`, `exclude_patterns` (long curated list), `codeignore_file = ".vibeignore"`.
- **Backend selection.** `shutil.which("rg")` ⇒ ripgrep; else `shutil.which("grep")` ⇒ GNU grep; else `ToolError`.
- **Ripgrep cmdline.** `rg --line-number --no-heading --smart-case --no-binary --max-count {max+1} [--no-ignore] [--glob !pattern] -e {pattern} {path}`.
- **GNU grep cmdline.** `grep -r -n -I -E --max-count={max+1} [-i] [--exclude-dir=…] [--exclude=…] -e {pattern} {path}`.

### A.6 `task`

- **LLM description.** "Delegate a task to a subagent for independent execution. Useful for exploration, research, or parallel work that doesn't require user interaction. The subagent runs in-memory and saves interaction logs."
- **Args.** `task: str`, `agent: str = "explore"`.
- **Result.** `response: str`, `turns_used: int`, `completed: bool`.
- **Config.** `permission = ASK`, `allowlist = ["explore"]` (only `explore` is auto-approved by default).
- **Permission resolution.** `args.agent` matched against denylist (`fnmatch`) ⇒ `NEVER`; against allowlist ⇒ `ALWAYS`; else `None`.
- **Errors.** `"Task tool requires agent_manager in context"`, `"Unknown agent: {name}"`, `"Agent '{name}' is a {type} agent. Only subagents can be used..."`.
- **Subagent invocation.** Creates a new `AgentLoop(is_subagent=True)`, injects scratchpad info if available, streams `AssistantEvent` and `ToolResultEvent`, marks `completed = False` if any event is `stopped_by_middleware` or `skipped`. Each subagent tool result is forwarded as a `ToolStreamEvent` so the parent sees progress.

### A.7 `todo`

- **LLM description.** "Manage todos. Use action='read' to view, action='write' with complete list to update."
- **Args.** `action: "read" | "write"`, `todos: list[TodoItem] | None`.
- **TodoItem.** `id: str`, `content: str`, `status: PENDING | IN_PROGRESS | COMPLETED | CANCELLED`, `priority: LOW | MEDIUM | HIGH`.
- **Result.** `message: str`, `todos: list[TodoItem]`, `total_count: int`.
- **Config.** `permission = ALWAYS`, `max_todos = 100`.
- **State.** In-memory list (per session, not persisted to disk).
- **Errors.** `"Cannot store more than {max_todos} todos"`, `"Todo IDs must be unique"`.

### A.8 `ask_user_question`

- **LLM description.** "Ask the user one or more questions and wait for their responses. Each question has 2-4 choices plus an automatic 'Other' option for free text."
- **Args (`AskUserQuestionArgs`).** `questions: list[Question]` (1–4), `content_preview: str | None`.
- **Question.** `question: str`, `header: str` (≤ 12 chars), `options: list[Choice]` (2–4), `multi_select: bool = False`, `hide_other: bool = False`.
- **Choice.** `label: str`, `description: str = ""`.
- **Answer.** `question: str`, `answer: str`, `is_other: bool = False`.
- **Errors.** `"User input not available. This tool requires an interactive UI."` (when `user_input_callback` missing — i.e. programmatic mode).

### A.9 `exit_plan_mode`

- **LLM description.** "Signal that your plan is complete and you are ready to start implementing. This will ask the user to confirm switching from plan mode to accept-edits mode."
- **Args.** Empty.
- **Result.** `switched: bool`, `message: str`.
- **Permission.** `ALWAYS`.
- **Behaviour.** Reads the plan file via `ctx.plan_file_path` (when set) and shows it as `content_preview` to the user. Internally calls `user_input_callback` with three options: *Yes, auto-approve edits* (switches to `accept-edits`), *Yes, require approval* (switches to `default`), *No* (stays in `plan`). Switch is performed via `ctx.switch_agent_callback(name)` if available, otherwise `agent_manager.switch_profile(name)`.
- **Errors.** `"ExitPlanMode requires an agent manager context."`, `"ExitPlanMode can only be used in plan mode."`, `"ExitPlanMode requires an interactive UI."`, `"Failed to read plan file at {path}: {e}"`.

### A.10 `webfetch`

- **LLM description.** "Fetch content from a URL. Converts HTML to markdown for readability."
- **Args.** `url: str`, `timeout: int | None` (capped at `max_timeout`).
- **Result.** `url: str`, `content: str`, `content_type: str`, `was_truncated: bool`.
- **Config.** `permission = ASK`, `default_timeout = 30`, `max_timeout = 120`, `max_content_bytes = 120_000`, `user_agent =` modern Chrome UA (with bot-detection fallback to `vibe-cli`).
- **URL normalisation.** Protocol-relative `//example.com` and bare `example.com` get an `https://` prefix.
- **Permission resolution.** When the static permission is `ASK`, returns a `URL_PATTERN` `RequiredPermission` for the host domain.
- **HTML→Markdown.** Custom `markdownify.MarkdownConverter` removes `script/style/noscript/iframe/object/embed`.
- **Bot-detection retry.** On 403 with `cf-mitigated: challenge`, retry once with the honest `vibe-cli` User-Agent.

### A.11 `websearch`

- **LLM description.** "Search the web for current information using Mistral's web search."
- **Args.** `query: str` (min length 1).
- **Result.** `answer: str`, `sources: list[{title, url}]` (deduped by URL).
- **Config.** `permission = ASK`, `timeout = 120`, `model = "mistral-vibe-cli-with-tools"`.
- **Availability.** Tool only registered if `MISTRAL_API_KEY` is in env.
- **Backend.** `client.beta.conversations.start_async()` with the `web_search` tool — extracts `TextChunk` (answer) and `ToolReferenceChunk` (sources).

### A.12 `skill`

- **LLM description.** "Load a specialized skill that provides domain-specific instructions and workflows. … The skill will inject detailed instructions, workflows, and access to bundled resources (scripts, references, templates) into the conversation context."
- **Args.** `name: str`.
- **Result.** `name: str`, `content: str` (XML-wrapped), `skill_dir: str | None`.
- **Permission.** `ALWAYS`.
- **Behaviour.** `skill_manager.get_skill(name)` lookup; collect up to 10 bundled files via `skill_dir.rglob("*")` (skipping `SKILL.md`); wrap in `<skill_content name="…">` and `<skill_files>` sections (relative paths from skill base dir).
- **Errors.** `"Skill manager not available"`, `'Skill "{name}" not found. Available skills: {comma_separated}'`.

---

## Appendix B — LLM Wire Format & Tool-Call Assembly

### B.1 `APIToolFormatHandler` (`vibe/core/llm/format.py`)

- `process_api_response_message(provider_msg)` ⇒ canonical `LLMMessage`. Tool calls are translated to `ToolCall(id, index, type, function={name, arguments: str})`. Reasoning fields are propagated when the provider emits them.
- `parse_message(LLMMessage)` ⇒ list of `ParsedToolCall(tool_name, raw_args: dict, call_id)`. JSON-decodes each `function.arguments`; **on `JSONDecodeError`, defaults to `{}` silently** so a malformed stream still surfaces a recoverable failure downstream.
- `resolve_tool_calls(parsed, tool_manager)` ⇒ list of `ResolvedToolCall | FailedToolCall`.
  - Unknown tool ⇒ `FailedToolCall(error="Unknown tool '{name}'")`.
  - Pydantic `ValidationError` ⇒ `FailedToolCall(error=f"Invalid arguments: {e}")` (full error detail included).
- `create_tool_response_message(call, result_text)` ⇒ `LLMMessage(role=tool, tool_call_id, name, content)`.

### B.2 `LLMMessage.__add__` accumulation rules

Strict invariants — violation raises `ValueError`:
- Different `role` ⇒ `"Can't accumulate messages with different roles"`.
- Different `name` ⇒ `"Can't accumulate messages with different names"`.
- Different `tool_call_id` ⇒ `"Can't accumulate messages with different tool_call_ids"`.

Tool-call merge:
- `tool_calls` are merged in an `OrderedDict` keyed by `index`; missing index ⇒ `"Tool call chunk missing index"`.
- New chunk extends an existing tool-call by **string-concatenating `function.arguments`** (i.e. partial JSON pieces are appended in order).
- Conflicting non-empty `function.name` between chunks ⇒ `"Can't accumulate messages with different tool call names"`. New name is accepted if old was empty.

Validation is deferred until the *complete* message has accumulated; the parser always runs against the joined argument string.

### B.3 Mistral backend (`vibe/core/llm/backend/mistral.py`)

- Message conversion: `MistralMapper.prepare_message`:
  - `system` ⇒ `SystemMessage(content=msg.content or "")`.
  - `user` ⇒ `UserMessage(content=msg.content)`.
  - `assistant` with `reasoning_content` ⇒ content list `[ThinkChunk(thinking=[TextChunk(reasoning_content)]), TextChunk(content?)]`. Without reasoning, content is plain string.
  - `tool` ⇒ `ToolMessage(content, tool_call_id, name)`.
- Tool conversion: `Tool(type="function", function=Function(name, description, parameters))`.
- Retry config: exponential backoff, 500 ms initial, 30 s max, exponent 1.5, max elapsed 300 s, retries on connection errors.
- SSL: `build_ssl_context()` (cached LRU 1) — starts from `certifi.where()`, additively loads `SSL_CERT_FILE` / `SSL_CERT_DIR`, logs and continues on error.
- Streaming: extracts `mistral-correlation-id` response header, yields `LLMChunk(message, usage, correlation_id)` per delta.
- `count_tokens()`: implemented by calling `complete(max_tokens=1)` and reading `usage.prompt_tokens`. **No dedicated count-tokens endpoint.**
- Error mapping: `BackendErrorBuilder.build_http_error()` (SDKError) and `build_request_error()` (httpx.RequestError) wrap into `BackendError` with provider, endpoint, status, headers, body, model, payload summary (msg count, char count, model, temperature, tools enabled).

### B.4 Generic backend (`vibe/core/llm/backend/generic.py`) and adapters

`APIAdapter` protocol:
```python
endpoint: ClassVar[str]
def prepare_request(...) -> PreparedRequest
def parse_response(data, provider) -> LLMChunk
```

OpenAI-compatible adapter:
- Tool format: `{type: "function", function: {name, description, parameters}}`.
- `excludes` `message_id`, `reasoning_message_id`, `reasoning_state`, `injected` from outgoing message JSON.
- Reasoning field name remapped via `provider.reasoning_field_name`.
- Streaming payload includes `stream: true`, `stream_options: {include_usage: true, stream_tool_calls: true}` (Mistral-specific extension).
- SSE parsing: lines starting with `data:`, terminated by `[DONE]` sentinel; comment lines (`:…`) and blank lines skipped.

Anthropic adapter:
- Tool format: `{name, description, input_schema}`.
- System message extracted as a separate top-level string.
- User content = `[{type: "text", text: "…"}]`; tool results appended to last user message or create one.
- Streaming events: `message_start`, `content_block_start` (initialises tool call with id+name), `input_json_delta` (appends partial JSON to args), `content_block_delta`, `message_delta`, `message_stop`. Thinking blocks have type `thinking` with nested text blocks.

Vertex adapter:
- Inherits all conversion from the Anthropic adapter.
- Endpoint: `/v1/projects/{project_id}/locations/{region}/publishers/anthropic/models/{model}:{rawPredict|streamRawPredict}`.
- Base URL: `https://{region}-aiplatform.googleapis.com` (or `…/aiplatform.googleapis.com` for global).
- Auth: Google ADC. `VertexCredentials.access_token` calls `google.auth.default(scopes=["https://www.googleapis.com/auth/cloud-platform"])`, refreshes when stale (thread-safe via lock), sends `Authorization: Bearer {access_token}`.

### B.5 Reasoning adapter (`reasoning_adapter.py`)

Not a true wrapper — it is a full adapter. `_parse_content_blocks(content)`:
- string ⇒ `(content, None)`
- list of blocks ⇒ concatenate `text` blocks into content; concatenate inner text blocks of `thinking` blocks into `reasoning_content`.

### B.6 Backend exceptions (`vibe/core/llm/exceptions.py`)

`BackendError(RuntimeError)` carries `provider`, `endpoint`, `status`, `reason`, `headers` (lowercased keys), `body_text`, `parsed_error`, `model`, `payload_summary`.

`is_context_too_long`:
- Only `True` when `status == 400` AND the (lowercased) body contains any of:
  - `"context too long"`
  - `"maximum context length"`
  - `"input too large"`
  - `"couldn't fit with truncation"`
  - `"prompt is too long"`

User-facing formatting:
- `401` ⇒ `"Invalid API key. Please check your API key and try again."`
- `429` ⇒ `"Rate limit exceeded. Please wait a moment before trying again."`
- otherwise: multi-line block including status, reason, request id, endpoint, model, provider message, body excerpt (400 chars max with `…` overflow), payload summary JSON.

The agent loop maps:
- `status == 429` ⇒ `RateLimitError(provider, model)` (re-raised; no internal retry).
- `is_context_too_long` ⇒ `ContextTooLongError(provider, model)`.
- non-retryable Temporal errors ⇒ re-raised verbatim.
- everything else ⇒ `RuntimeError(f"API error from {provider}…")`.

---

## Appendix C — TUI Subsystems Reference

### C.1 CLI cache (`vibe/cli/cache.py`)

TOML file with sections (e.g. `[updates]`, `[whats_new]`).
- `read_cache(path)` ⇒ `dict` (empty on `OSError`/`TOMLDecodeError`).
- `write_cache(path, section, data)` ⇒ merges `data` into `existing[section]`, creates parent dirs, silent no-op on `OSError`. Default location: `$VIBE_HOME/cache.toml`.

### C.2 Profiler (`vibe/cli/profiler.py`)

- Activation: `VIBE_PROFILE=1`.
- API: `profiler.start("startup")` then `profiler.stop_and_print()`.
- Output: `{label}-profile.html` and `{label}-profile.txt` in cwd; coloured text summary to stderr.
- No-op when env var unset or `pyinstrument` not installed.

### C.3 Stderr guard (`vibe/cli/stderr_guard.py`)

A context manager that, on Unix TTYs:
1. `dup`s fd 2 to a new fd pointing at the real terminal.
2. `dup2`s `/dev/null` over fd 2 (absorbs stray native writes).
3. Reassigns `sys.__stderr__` and `sys.stderr` to a Python file object wrapping the dup'd fd, so Textual continues rendering correctly.
4. Restores everything on exit.

No-op on Windows or when fd 2 is not a TTY.

### C.4 Terminal detect (`vibe/cli/terminal_detect.py`)

`detect_terminal()` returns a `Terminal` enum value. Detection order:
1. `TERM_PROGRAM == "vscode"` ⇒ check Cursor env vars (`VSCODE_GIT_ASKPASS_*`, `VSCODE_IPC_HOOK_CLI`, `VSCODE_NLS_CONFIG` containing `"cursor"`) ⇒ `CURSOR`; else parse `TERM_PROGRAM_VERSION` for `-insider` suffix ⇒ `VSCODE_INSIDERS` or `VSCODE`.
2. `TERM_PROGRAM` map: `iterm.app → ITERM2`, `wezterm → WEZTERM`, `ghostty → GHOSTTY`, `alacritty → ALACRITTY`, `kitty → KITTY`, `hyper → HYPER`.
3. Env markers: `WEZTERM_PANE`, `GHOSTTY_RESOURCES_DIR`, `KITTY_WINDOW_ID`, `ALACRITTY_SOCKET`, `ALACRITTY_LOG`, `WT_SESSION → WINDOWS_TERMINAL`, `TERMINAL_EMULATOR` containing `"jetbrains" → JETBRAINS`.
4. Else ⇒ `UNKNOWN`.

### C.5 Clipboard (`vibe/cli/clipboard.py`)

Copy strategies attempted in order until one verifies:
1. **OSC 52** escape sequence (works over SSH, tmux-aware via `\033Ptmux;\033 … \033\\` wrapping).
2. **pyperclip**.
3. `pbcopy` (macOS), `xclip -selection clipboard` (X11), `wl-copy` (Wayland) — only if the binary is present.

Paste: `pyperclip`, `pbpaste`, `xclip -o -selection clipboard`, `wl-paste`.

`copy_selection_to_clipboard(app)` collects `text_selection` from focused widgets; toast "Copied to clipboard" or "Failed to copy - clipboard not available".

### C.6 File indexer (`vibe/core/autocompletion/file_indexer/`)

Default ignore patterns (`.gitignore`-style; `(pattern, is_exclude)` tuples) — used by `@`-autocomplete:

```
.git/, __pycache__/, node_modules/, .DS_Store, *.pyc, *.log, .vscode/, .idea/,
/build/, dist/, target/, .next/, .nuxt/, coverage/, .nyc_output/, *.egg-info,
.pytest_cache/, .tox/, vendor/, third_party/, deps/, *.min.js, *.min.css,
*.bundle.js, *.chunk.js, .cache/, tmp/, temp/, logs/, .uv-cache/, .ruff_cache/,
.venv/, venv/, .mypy_cache/, htmlcov/, .coverage
```

`WALK_SKIP_DIR_NAMES` is the frozenset of *directory-only, name-only, non-anchored* defaults — used by `walk_local_config_dirs` (project trust scan) AND by the indexer.

`IndexEntry` carries `rel`, `rel_lower`, `name`, `path`, `is_dir`, and an `ascii_mask` bitset for fast ASCII pre-filtering.

The indexer keeps a single-worker thread pool and tracks per-root rebuild tasks. The `watcher.py` (when enabled by config `file_watcher_for_autocomplete = true`) uses `watchfiles.watch()` to apply `Change.added | deleted | modified` events incrementally; ≥ 200 changes in one batch triggers a full rebuild.

### C.7 Fuzzy matcher (`vibe/core/autocompletion/fuzzy.py`)

Subsequence scoring with multipliers and bonuses:
- Base score 100 + positional bonus/penalty.
- ×2.0 prefix match, ×1.8 word-boundary match (`/`, `-`, `_`, `.`, uppercase transitions), ×1.3 consecutive match.
- +2.0 per matching case, +10.0 per adjacent match, +5.0 per word boundary, +3.0 per uppercase boundary, ×1.5 per gap (penalty).
- Empty pattern matches everything with score `0.0`. Returns `MatchResult(matched: bool, score: float, matched_indices: list[int])`.

### C.8 ANSI markdown (`vibe/cli/textual_ui/ansi_markdown.py`)

`AnsiMarkdown` extends Textual's `Markdown` to override `MarkdownFence.highlight()` with a custom `AnsiHighlightTheme` that maps Pygments tokens to Textual ANSI styles (e.g. `ansi_red`, `ansi_green`, `ansi_cyan`). Inline markdown (bold, italic, code) is rendered normally; fenced code blocks use Pygments with the ANSI theme.

### C.9 Debug console (`vibe/cli/textual_ui/widgets/debug_console.py`)

- Toggled with `Ctrl+\` (and `/debug`).
- Displays `LogEntry` items from `LogReader`, level-coloured (`DEBUG=dim, INFO=cyan, WARNING=yellow, ERROR=red, CRITICAL=bold red`).
- Initial page fills the viewport; scrolling up requests an earlier page from the file.
- LRU render cache (1024 entries).

### C.10 Themes

No runtime theme switcher. Styling is via `vibe/cli/textual_ui/app.tcss`, with `$mistral_orange = #FF8205` as the brand accent. The Textual theme name is `"textual-ansi"` (hardcoded in `app.py`). Code-fence syntax highlighting uses the custom `AnsiHighlightTheme` (see C.8).

### C.11 Modal apps / pickers

| Modal | Triggered by | Purpose | Persistence |
|-------|--------------|---------|-------------|
| `ConfigApp` | `/config` | Toggle UX flags (autocopy, file watcher). Buttons open Model/Thinking pickers. | Saves changes via `VibeConfig.save_updates()` |
| `ModelPickerApp` | `/model` or ConfigApp | OptionList of model aliases; current marked with `›` | Persists `active_model` in config |
| `ThinkingPickerApp` | `/thinking` | OptionList of thinking levels | Persists `models[i].thinking` via `set_thinking()` |
| `MCPApp` | `/mcp`, `/connectors`, banner | Hierarchical view of servers/connectors; per-tool enable/disable; OAuth prompts | Updates config + secrets |
| `VoiceApp` | `/voice` | Toggle voice/narrator | Saves to config |
| `SessionPickerApp` | `/resume` (no args) | OptionList of resumable sessions (time ago, source, id, latest message preview) | Read-only |
| `ApprovalApp` | Tool needing approval | 4-option picker: *Yes (one-time), Always (this tool, this session), Always (permanent), No* | "Always permanent" persists into `tools.<name>.allowlist` config |
| `ProxySetupApp` | `/proxy-setup` | Edit proxy env vars | Writes to `$VIBE_HOME/.env` |
| `RewindApp` | `/rewind` or Alt+↑ | Highlight a prior user message; pick rewind variant | In-memory only |
| `ConnectorAuthApp` | OAuth-required connector | Browser flow for connector auth | Saves token via secrets |

### C.12 Scheduled loop runner (`vibe/cli/textual_ui/scheduled_loop_runner.py`, `vibe/core/loop.py`)

- **Command syntax.** `/loop <interval> <prompt>` schedules; `/loop list` shows table; `/loop cancel <id|all>` cancels.
- **Interval format.** `\d+[smhd]` (seconds/minutes/hours/days); minimum **30 seconds**.
- **Persistence.** Loops are stored in `SessionMetadata.loops` (each: `id, interval_seconds, prompt, next_fire_at, created_at`) and serialised through `session_logger.persist_loops()`. `restore_from_session()` reloads them on resume.
- **Polling.** Sleeps `max(0.05, min(time_until_next_due, 1.0))` seconds. On each tick: if `can_fire()` returns `True`, pop loops with `next_fire_at <= now`, fire each prompt as a new user message, append a `UserCommandMessage` to the chat, then update `next_fire_at`.

### C.13 Recording UI (`vibe/cli/textual_ui/recording/recording_indicator.py`)

- `RecordingIndicator` reads `VoiceManagerPort.transcribe_state` (IDLE / RECORDING / FLUSHING).
- RECORDING: peak meter — polls `peak` every 50 ms, maps `[0,1]` to one of `▁▂▃▄▅▆▇█`.
- FLUSHING: animation through `▏▎▍▌▋▊▉█` every 100 ms.
- IDLE: hidden / empty.

### C.14 Quit manager (`vibe/cli/textual_ui/quit_manager.py`)

First Ctrl+C or Ctrl+D opens a 1.0 s confirmation window; `PathDisplay` shows `Press [key] again to quit`. Second matching keystroke within 1.0 s exits; mismatched / late keystroke resets.

### C.15 Session exit (`vibe/cli/textual_ui/session_exit.py`)

On graceful exit prints token usage summary (input / output / total) and the resume command (`vibe --continue` or `vibe --resume <id>`). `SessionLogger` has already flushed metadata.

### C.16 External editor (`vibe/cli/textual_ui/external_editor.py`)

`tempfile.mkstemp(suffix=".md", prefix="vibe_")` ⇒ write current input ⇒ `subprocess.run([editor_argv, file])` (blocking) ⇒ read back ⇒ unlink in `finally` ⇒ return None if unchanged or subprocess failed.

### C.17 Windowing (`vibe/cli/textual_ui/windowing/`)

Lazy-loads chat history on resume so resuming a 1000-message session doesn't render everything at once.
- `SessionWindowing` tracks visible range and backfill state.
- `HistoryResumePlan` carries `tail_messages`, `backfill_messages`, `tool_call_map`.
- `create_resume_plan(history)` splits into tail (rendered immediately) and backfill (loaded on scroll-up).
- `sync_backfill_state()` keeps it consistent when history mutates mid-session.

### C.18 Notifications

`NotificationContext.ACTION_REQUIRED` and `NotificationContext.COMPLETE`. Title is set with OSC 0 (`\x1b]0;{title}\x07`). Throttle: 1 notification / 1.0 s. Bell only fires while the app is unfocused; default title is restored on focus.

### C.19 Remote (`vibe/cli/textual_ui/remote/`)

`RemoteSessionManager` attaches to a Mistral-hosted (Vibe Code) session. Subscribes to a `RemoteEventsSource` that yields `AssistantEvent`, `ReasoningEvent`, `ToolCallEvent`, etc., and translates them to TUI updates. Handles `WaitingForInputEvent` (blocks the remote agent until the user responds via the question picker, up to 4 choices).

### C.20 Banner (`vibe/cli/textual_ui/widgets/banner/`)

ASCII art `petit chat` plus an info block:

1. `Mistral Vibe v{version} · {model}[{thinking}] {plan_cta}`
2. `{N} models · {N} connectors · {N} MCP servers · {N} skill[s]` (sections omitted when zero)
3. `Type /help for more information`

`PetitChat` animates on mount unless `disable_welcome_banner_animation = true`. `freeze_animation()` is called when the first user message arrives so the chat area becomes the focus.

### C.21 Chat input (`vibe/cli/textual_ui/widgets/chat_input/text_area.py`)

- Submit on Enter (without modifier).
- Shift+Enter or Ctrl+J inserts a newline.
- First non-whitespace character classifies the input mode: `>` chat, `!` shell command, `/` slash command, `&` reserved/experimental.
- History up/down navigates `~/.vibe/vibehistory` (skips multi-line edits in progress).
- Ctrl+G ⇒ external editor (see C.16).
- Paste is detected via the `_cursor_moved_since_load` flag to allow correct multi-line pastes.

### C.22 Update notifier UI

A background async task (`_schedule_update_notification`) polls the configured gateway (PyPI / GitHub) at start and every check interval. When `UpdateAvailability.should_notify` is true, an inline notification is mounted with current/latest version and a hint to run the install/upgrade command.

### C.23 What's New widget

If `cache.seen_whats_new_version != current_version` AND `vibe/whats_new.md` is non-empty: load it, mount a `WhatsNewMessage(content)` widget styled with the `whats-new-message` class, and immediately call `mark_version_as_seen(current_version)` so it does not reappear next launch.

---

## Appendix D — Auxiliary Subsystems & Exact Wire / Text Content

### D.1 Built-in prompt summaries (verbatim files in `vibe/core/prompts/`)

| File | Length | Purpose | Notable directives |
|------|--------|---------|--------------------|
| `cli.md` | 110 lines | Default agent persona | "Most tasks need <150 words." Three-phase workflow (Orient/Plan/Execute). Explicit "Never say" / "Never use" word lists. Zero-emoji rule. |
| `compact.md` | 49 lines | Compaction utility | Demands a 7-section summary: Goals, Timeline, Technical Context, Files & Code Changes, Active Work, Unresolved Issues, Immediate Next Step |
| `turn_summary.md` | 11 lines | Turn-summary utility | 2–4 sentences covering: ask, actions, outcome, optional open questions |
| `explore.md` | 51 lines | Explore subagent | "CODE/DIAGRAM FIRST." Bans greetings, announcements, hedging, puffery |
| `lean.md` | 158 lines | Lean 4 specialist | Same Orient/Plan/Execute phases; Lean-specific commands (`lake exe cache get`, `lake build`, `grind` tactic, `trace_state` debug); long Hard Rules section |
| `tests.md` | 1 line | Test fixture stub | `"You are Vibe, a super useful programming assistant."` |
| `project_context.md` | 4 lines | Template injected into system prompt | Variables: `$abs_path`, `$git_status` |
| `agents_doc.md` | 5 lines | AGENTS.md wrapper | Variable `$sections`. Declares precedence: project > user; closer-to-cwd > farther |
| `dangerous_directory.md` | 5 lines | Project-context disable warning | Variables: `$reason`, `$abs_path` |

### D.2 HarnessFilesManager (`vibe/core/config/harness_files/`)

Singleton initialised by `init_harness_files_manager(*sources)`. `sources` is an ordered tuple, e.g. `("user", "project")`. The `"project"` source is suppressed when the working directory has not been (or cannot be) trusted.

Per-source paths:
- **user** — `~/.vibe/config.toml`, `~/.vibe/hooks.toml`, `~/.vibe/{tools,skills,agents,prompts}/`.
- **project** — `<cwd>/.vibe/config.toml`, `<cwd>/.vibe/hooks.toml`, plus all `walk_local_config_dirs(cwd)` entries for `tools/`, `skills/`, `agents/`. Project AGENTS.md files are walked from cwd up to the trust root and concatenated outermost-first.

Important properties:
- `trusted_workdir` ⇒ `cwd` if `"project"` is enabled and the folder is trusted, else `None`.
- `persist_allowed` ⇒ `True` iff `"user"` is in `sources` (controls whether `VibeConfig.save_updates()` actually writes).
- `config_file` ⇒ project path if trusted-and-enabled, else user path.
- `hook_files` ⇒ list of one or two hook TOML paths.

### D.3 Browser sign-in (`vibe/setup/auth/`)

Exact endpoints (relative to `provider.browser_auth_api_base_url`, default `https://console.mistral.ai/api`):

1. `POST {api_base_url}/vibe/sign-in` — body `{code_challenge, code_challenge_method: "S256"}`. Response `CreateProcessPayload {process_id, sign_in_url, poll_url, expires_at}`.
2. Open `sign_in_url` in the user's browser.
3. `GET {poll_url}` repeatedly. Response `PollPayload {status: "pending"|"completed"|"expired"|"denied"|"error", exchange_token?, message?}`. Default poll interval 3.0 s; up to 3 consecutive failures tolerated; `410 Gone` ⇒ expired.
4. `POST {api_base_url}/vibe/sign-in/{process_id}/exchange` — body `{exchange_token, code_verifier}`. Response `ExchangePayload {api_key}`.

PKCE: SHA256-based code challenge.

### D.4 Plan offer / WhoAmI (`vibe/cli/plan_offer/`)

- Endpoint: `GET {provider.browser_auth_api_base_url}/vibe/whoami` (default base `https://console.mistral.ai/api`) — Vibe also computes a derived URL via `get_server_url_from_api_base()` when needed.
- Header: `Authorization: Bearer {api_key}`.
- Response: `{plan_type: str, plan_name: str, prompt_switching_to_pro_plan?: bool}`.

`WhoAmIPlanType` enum: `API`, `CHAT`, `MISTRAL_CODE`, `UNKNOWN`, `UNAUTHORIZED`. `MistralCodePlanName`: `"F"` (free), `"E"` (enterprise).

`PlanInfo` predicates:
- `is_paid_api_plan()` — `API` and not free.
- `is_free_api_plan()` — `API` with `"FREE"` in name.
- `is_chat_pro_plan()` — `CHAT`.
- `is_teleport_eligible()` — Chat Pro AND not `prompt_switching_to_pro_plan`.
- `is_free_mistral_code_plan()` — `MISTRAL_CODE` with `plan_name == "F"`.
- `is_mistral_code_enterprise_plan()` — `MISTRAL_CODE` with `plan_name == "E"`.

`plan_offer_cta()` returns banner CTA markdown:
- If `prompt_switching_to_pro_plan`: "Switch to your [Le Chat Pro API key](https://console.mistral.ai/codestral/cli)".
- If `API` / `UNAUTHORIZED` / free Mistral Code: "Unlock more with Vibe — [Upgrade to Le Chat Pro](https://console.mistral.ai/codestral/cli)".

### D.5 Lean install / uninstall

`/leanstall` and `/unleanstall` mutate `installed_agents` in `config.toml` (adding/removing the `lean` entry) via `VibeConfig.save_updates()`. The mutation reloads the agent manager. **No external dependency installation is performed by Vibe** — the user is expected to have `lake`/`elan` available; the prompt embedded in `lean.md` instructs the agent how to bootstrap a Lean project. ACP also propagates the change through `set_config_option`.

### D.6 MCP registry (`vibe/core/tools/mcp/registry.py`)

- Each server's effective config is hashed (SHA256 of `model_dump_json`) into a fingerprint key.
- `_cache: dict[fingerprint, dict[tool_name, proxy_class]]` survives agent switches; new tools are discovered only when the fingerprint changes.
- HTTP / streamable-HTTP discovery: `list_tools_http(url, headers, startup_timeout_sec)`.
- stdio discovery: `list_tools_stdio(cmd, env, cwd, startup_timeout_sec)`.
- Each `RemoteTool` becomes a `MCPTool` proxy class via `create_mcp_http_proxy_tool_class()` or `create_mcp_stdio_proxy_tool_class()`.
- Sampling: respects `sampling_enabled` per server. When the remote requests `sampling/createMessage`, the agent loop's active backend services it.

### D.7 Connector registry (`vibe/core/tools/connectors/connector_registry.py`)

- Bootstrap endpoint: `GET {server}/v1/connectors/bootstrap` with `Authorization: Bearer {MISTRAL_API_KEY}`.
- Tools surfaced only when the connector entry has `status.is_ready == true`.
- Per-connector `name` is normalised to `[a-zA-Z0-9_-]`, max 256 chars; collisions are deduped with a numeric suffix.
- Tool name format: `connector_{alias}_{remote_name}`.
- Tool execution endpoint: `POST {server}/v1/experimental/connectors/{connector_id}/mcp` (Mistral-side proxy that re-uses MCP semantics).

### D.8 Vibe Code / Nuage workflow client (`vibe/core/nuage/client.py`)

`WorkflowsClient` is an async HTTP client (`httpx.AsyncClient` with `build_ssl_context()`).

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `{base}/v1/workflows/{workflow_id}/execute` | POST | Start workflow with `WorkflowParams` |
| `{base}/v1/workflows/events/stream` | GET (SSE) | Stream `StreamEvent`s |
| `{base}/v1/workflows/executions/{execution_id}/signals` | POST | Send a workflow signal |
| `{base}/v1/workflows/executions/{execution_id}/updates` | POST | Send a workflow update |
| `{base}/v1/workflows/runs` | GET | List runs (pagination, status filter) |

SSE parser ignores comment lines (`:…`) and blank lines, switches state on `event:` lines, and JSON-parses each `data:` block into `StreamEvent.model_validate(...)`. Errors yield `WorkflowsException`.

### D.9 Teleport flow (`vibe/core/teleport/`)

Ordered events (each a `BaseEvent` subclass):

1. `TeleportCheckingGitEvent` — fetch remote, inspect status.
2. `TeleportPushRequiredEvent(unpushed_count)` — surfaces if local is ahead. The user replies via `TeleportPushResponseEvent(approved: bool)`.
3. `TeleportPushingEvent` — performs `git push`.
4. `TeleportStartingWorkflowEvent` — calls `nuage_client.start_workflow()`.
5. `TeleportWaitingForGitHubEvent` — polls until GitHub connection ready.
6. `TeleportAuthRequiredEvent(oauth_url)` — when auth is needed; user signs in.
7. `TeleportAuthCompleteEvent`.
8. `TeleportFetchingUrlEvent`.
9. `TeleportCompleteEvent(url)`.

The git helpers cover `fetch()`, `is_commit_pushed()`, `is_branch_pushed()`, `get_unpushed_commit_count()`. Auth scope is whatever GitHub OAuth scope is configured upstream by the Nuage workflow; Vibe just opens the URL and polls.

### D.10 Update notifier — `do_update`

`vibe/cli/update_notifier/update.py::do_update()` tries, in order:
1. `uv tool upgrade mistral-vibe`.
2. `brew upgrade mistral-vibe`.

Returns `True` on the first success. stdin is `DEVNULL`; stdout/stderr captured. **Only fired on user action** (no silent automatic upgrade) — the cache and notification just inform the user that an upgrade is available.

### D.11 Data retention message (`vibe/core/data_retention.py`)

Verbatim:

```
## Your Data Helps Improve Mistral AI

At Mistral AI, we're committed to delivering the best possible experience. When you use Mistral models on our API, your interactions may be collected to improve our models, ensuring they stay cutting-edge, accurate, and helpful.

Manage your data settings [here](https://admin.mistral.ai/plateforme/privacy)
```

### D.12 What's New (`vibe/cli/update_notifier/whats_new.py`, `vibe/whats_new.md`)

`should_show_whats_new(current_version, repository)` — true iff `cache.seen_whats_new_version != current_version`. `load_whats_new_content()` reads `VIBE_ROOT/whats_new.md` (an empty marker file in the current release). On show, `mark_version_as_seen(current_version)` is called immediately.

### D.13 Trustable file detection (`vibe/core/trusted_folders.py`)

`find_trustable_files(path)` returns a list of relative names that, if present, would cause Vibe to read configuration: `AGENTS.md` and the relative paths of every directory returned by `walk_local_config_dirs(path).config_dirs`. The CLI calls this on startup; if non-empty AND the dir's trust state is `None`, it prompts via `ask_trust_folder()`.

### D.14 Programmatic stdin handling (`vibe/cli/cli.py`)

- `get_prompt_from_stdin()` reads all of stdin if not a TTY, then re-opens `/dev/tty` for future interactive use, strips whitespace, returns `None` if empty.
- Precedence in programmatic mode: `--prompt VALUE` > stdin. If `-p` is given without value, stdin is required.
- In interactive mode, stdin is used as the initial prompt when no positional argument is given.

### D.15 Output formatter event handling

- `TextOutputFormatter`: collects `LLMMessage`s; tracks `_final_response` from the last `AssistantEvent`; prints teleport status lines synchronously. `finalize()` prints `_final_response` (or the teleport URL on completion).
- `JsonOutputFormatter`: collects messages; ignores events; `finalize()` prints `json.dumps(messages, indent=2)`.
- `StreamingJsonOutputFormatter`: writes each `LLMMessage` as a single JSON line on `on_message_added`; ignores events; `finalize()` returns `None`.

Only `LLMMessage` objects make it into `--output json|streaming`. Mid-stream events (`ToolCallEvent`, `CompactStartEvent`, …) influence the *text* surface only.

### D.16 ACP exception code map (verbatim)

```python
UNAUTHENTICATED = -32000
INVALID_PARAMS = -32602
INTERNAL_ERROR = -32603
METHOD_NOT_FOUND = -32601
RATE_LIMITED = -31001
CONFIGURATION_ERROR = -31002
CONVERSATION_LIMIT = -31003
CONTEXT_TOO_LONG = -31004
```

All raised as subclasses of `VibeRequestError(acp.RequestError)`: `UnauthenticatedError`, `NotImplementedMethodError`, `InvalidRequestError`, `SessionNotFoundError`, `SessionLoadError`, `RateLimitError`, `ContextTooLongError`, plus `ConversationLimitError`, `InternalError`.

### D.17 Connector / MCP / Bash environment scrubbing

The `bash` tool sets these env vars before running a subprocess:

| Variable | Unix | Windows |
|----------|------|---------|
| `CI` | `true` | `true` |
| `NONINTERACTIVE` | `1` | `1` |
| `NO_TTY` | `1` | `1` |
| `TERM` | `dumb` | (unchanged) |
| `DEBIAN_FRONTEND` | `noninteractive` | — |
| `GIT_PAGER` | `cat` | `more` |
| `PAGER` | `cat` | `more` |
| `LESS` | `-FX` | — |
| `LC_ALL` | `en_US.UTF-8` | — |

This guarantees deterministic, paginator-free output and avoids interactive prompts.

---

## Appendix E — Errata & Confirmations Against §1–§28

This appendix amends or sharpens parts of the body where deeper inspection produced a more accurate statement.

1. **Trusted-folders TOML format** (§7) — corrected inline. The file holds **TOML lists of strings under `trusted` and `untrusted` keys**, NOT tables of `path = ""` mappings.
2. **`read_file` sensitive paths** (§11.1) — clarified: triggered by `**/.env` and `**/.env.*` glob patterns matched against the resolved path.
3. **`task` default allowlist** (§11.6) — the only auto-approved subagent is `explore`; any other subagent (e.g. installed `lean` if used as a subagent) goes through `ASK`.
4. **`websearch` model** (§11.11) — confirmed: `"mistral-vibe-cli-with-tools"`. Tool is hidden when no `MISTRAL_API_KEY`.
5. **Auto-update** (§25) — corrected: there is no silent auto-upgrade; `do_update()` is only invoked on user action and tries `uv tool upgrade mistral-vibe`, then `brew upgrade mistral-vibe`.
6. **Skill prompt injection** (§14) — `skill` tool wraps result in `<skill_content name="…">…</skill_content>` and `<skill_files>…</skill_files>` XML; up to 10 bundled non-`SKILL.md` files are listed by relative path.
7. **`exit_plan_mode` UX** (§11.9) — clarified: it asks the user with three options — *Yes, auto-approve edits* ⇒ `accept-edits`; *Yes, require approval* ⇒ `default`; *No* ⇒ stay in `plan`.
8. **`/loop` minimum interval** (§15) — minimum is **30 s**; format is `\d+[smhd]`.
9. **Plan WhoAmI endpoint** (§16.7) — exact path is `/vibe/whoami` under the provider's `browser_auth_api_base_url` (default `https://console.mistral.ai/api`).
10. **Browser sign-in endpoints** (§23.2) — exact endpoints are `/vibe/sign-in`, the polling URL returned by the server, and `/vibe/sign-in/{process_id}/exchange`. Default poll interval 3.0 s, max 3 consecutive failures tolerated, `HTTP 410` ⇒ expired.
11. **Backend `count_tokens`** (§9.1) — the Mistral backend implements it by calling `complete(max_tokens=1)` and reading `usage.prompt_tokens`. It does **not** call a dedicated count-tokens endpoint.
12. **Streaming tool-call assembly** (§9.5) — chunks of a single tool call are merged by `index`, with `function.arguments` string-concatenated; validation against the Pydantic args model is deferred until the message has fully accumulated. Invalid JSON falls back to `{}` silently before validation.
13. **`bash` allowlist drift across versions** (§5.7 migration) — `find` was added to the default allowlist via migration; trailing ` *` patterns are stripped.

---

*End of specification — Appendices A–E complete the reproduction reference.*

