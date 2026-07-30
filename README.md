# Mini Claude Code

> A terminal-based AI coding agent for the JVM. Mini Claude Code turns natural-language
> requests into real actions in your codebase — reading and editing files,
> searching code, running commands, browsing the web, and orchestrating
> multi-step plans — all from an interactive command line.

## What is Mini Claude Code?

Mini Claude Code is a Java 17 command-line agent that pairs a large language model
with a rich tool layer, so it can actually *do* work in your project instead of
only talking about it. It ships three execution strategies that all share the
same tools, memory, and safety layer:

- **ReAct** — a single reasoning/acting loop for direct, interactive tasks.
- **Plan-and-Execute** — a planner that decomposes a request into a dependency
  graph (DAG) and runs the steps in order, with a confirmation gate before
  execution.
- **Multi-Agent** — a Planner / Worker / Reviewer team coordinated by an
  orchestrator, with review-and-retry on failed quality checks.

On top of that, Mini Claude Code provides multiple LLM providers, retrieval-augmented code
understanding, the Model Context Protocol (MCP) for pluggable external tools,
headless browser automation, human-in-the-loop approvals, a skill system, and a
modern streaming terminal UI.

```text
   ████████    Mini Claude Code  v16.1.0
     ██  ██    Model glm-5.1 (glm)
     ██  ██    MCP 4/4 · 61 tools · 2/2 skills · ReAct
     ██  ██    ReAct · Plan · MCP · Browser · Image
     ██  ██

Tips for getting started:
1. Type / for commands and Tab completion
2. Ask coding questions, edit code or run commands
3. Attach context with @path or @image:
```

## Key Features

### Agent execution modes

- **ReAct (default):** think → act → observe loop driven by streaming model
  output. Best for simple or single-step tasks.
- **Plan-and-Execute (`/plan`):** decomposes complex work into a DAG of tasks,
  shows the plan for confirmation, then executes independent tasks in parallel
  batches. Trivial requests still get a minimal plan instead of padded steps.
- **Multi-Agent (`/team`):** a Planner decomposes the task, Workers execute, and
  a Reviewer checks quality; failed reviews retry with feedback (up to twice),
  and conflicts are resolved automatically.

All three modes share one `ToolRegistry`, `MemoryManager`, snapshot service, MCP
server pool, skill registry, and HITL handler — no mode creates isolated,
orphaned capabilities.

### Built-in tools

- `read_file` / `write_file` / `list_dir` — file operations
- `glob_files` — filename-glob search across the project (read-only, skips
  common build/dependency directories)
- `grep_code` — keyword/regex code search, prefers `ripgrep` when installed and
  falls back automatically; returns file, line, optional context, partial state,
  and suggested reads
- `execute_command` — short-lived shell commands in the project directory
  (60-second timeout, destructive-command blocklist)
- `create_project` — scaffold a Java / Python / Node project
- `search_code` — semantic (RAG) code retrieval for fuzzy, natural-language
  queries; exact lookups prefer `glob_files` / `grep_code` / `read_file`
- `web_search` — real-time internet search
- `web_fetch` — fetch a URL and extract the main content as Markdown
- `revert_turn` — restore the most recent pre-turn snapshot (gated by HITL and
  auditing)
- `mcp__{server}__{tool}` — external tools provided dynamically by MCP servers
- `mcp__{server}__list_resources` / `read_resource` — virtual tools for MCP
  servers that expose resources

When the model returns several tool calls in one turn, Mini Claude Code executes them in
parallel while preserving the original call order when feeding results back.

### Memory & context engineering

- **Short-term memory** tracks the current conversation and tool results.
- **Long-term memory** stores stable facts only, via `/save <fact>` or a
  `save_memory` call when you explicitly ask the agent to remember something.
  Default scope is per-project; cross-project preferences use `--global`.
- **Project memory** is Markdown-based: `PAI.md` / `.paicli/PAI.md` hold
  team-shared rules (commit them to git), while `PAI.local.md` is a local-only
  override.
- Conversations are automatically summarized and compressed as they approach the
  token budget.
- **Long-context modes** (short / balanced / long) adjust the budget, skip
  compression in long mode, and raise semantic-retrieval top-K.

### Codebase understanding (RAG)

- Code embeddings via local Ollama or a remote API
- SQLite-backed vector store with cosine-similarity retrieval
- Code chunking at file / class / method granularity with AST parsing
- A code relationship graph (extends / implements / imports / calls / contains)
- `/index`, `/search`, and `/graph` commands

### Multi-model support

A `LlmClient` interface plus a template base class (`AbstractOpenAiCompatibleClient`)
keeps each provider to a thin (~20-line) implementation. Built-in providers:

| Provider | Example models | Switch command |
|---|---|---|
| GLM | `glm-5.1`, `glm-5v-turbo` (multimodal) | `/model glm-5.1`, `/model glm-5v-turbo` |
| DeepSeek | DeepSeek V4 | `/model deepseek` |
| StepFun | `step-3.5-flash`, `step-3.7-flash` | `/model step` |
| Kimi / Moonshot | `kimi-k2.6` | `/model kimi` |
| FreeLLMAPI | OpenAI-compatible gateway | `/model freellmapi` |
| Agnes AI | `agnes-2.0-flash` | `/model agnes` |

The default model is persisted to `~/.paicli/config.json`; API keys are read from
config, environment variables, or a `.env` file. Each client declares its own
capabilities (context window, prompt caching, image input) so the request
serializer adapts automatically.

### Model Context Protocol (MCP)

- Supports both stdio subprocess servers and Streamable HTTP remote servers.
- Reads `~/.paicli/mcp.json` and `.paicli/mcp.json`; project config overrides
  user config per server name.
- `${VAR}` interpolation from environment variables, system properties, project
  `.env`, and `~/.env`.
- MCP tools are auto-registered as `mcp__{server}__{tool}`; parameter schemas are
  sanitized (`$ref` / `anyOf` / overlong descriptions) to reduce call failures.
- All MCP tools default to HITL approval and auditing, with credentials redacted
  from audit logs.
- Supports MCP resources and prompts, plus passive handling of
  `list_changed` / `updated` notifications.
- Reference a resource inline with `@server:protocol://path`.

### Browser automation

- Integrates a Chrome DevTools MCP by default, exposing browser tools such as
  `navigate_page`, `take_snapshot`, `click`, and `fill_form`.
- Used for SPA / JS-rendered / anti-scraping / form-interaction pages that
  `web_fetch` cannot handle.
- Runs an isolated temporary browser profile by default; `/browser connect`
  switches to a shared, already-authenticated Chrome to reuse login sessions.
- Sensitive pages force single-step HITL approval for mutating browser tools
  (`click`, `fill_form`, `evaluate_script`) while read-only tools stay available.

### Safety: Human-in-the-Loop + policy layer

- **HITL** flags dangerous operations (`write_file`, `execute_command`,
  `create_project`, `revert_turn`) across three risk levels; approve / approve-all
  / reject / skip / edit-then-run. Off by default; enable with `/hitl on`.
- **PathGuard** confines file tools to the project root, blocking absolute-path
  escapes, `..` traversal, and symlink escapes.
- **CommandGuard** fast-rejects destructive commands (`sudo`, whole-disk
  `rm -rf`, `mkfs`, `dd of=/dev`, fork bombs, `curl | sh`, `find /`,
  `chmod 777 /`, `shutdown`) before they ever reach HITL.
- **AuditLog** writes structured JSONL to `~/.paicli/audit/` with outcome
  (`allow` / `deny` / `error`) and approver (`hitl` / `policy` / `none`).
- Resource limits: `write_file` capped at 5 MB; `execute_command` at 60 seconds
  with output truncation.

This is a defense-in-depth model at the *tool boundary* — approval, path fencing,
command rejection, and auditing — not a sandbox. A local coding agent runs with
real access to your working directory by design; the guard rails focus on
catching destructive operations and keeping an audit trail.

### Skills

Skills package "how the agent should think" into reusable units instead of
hard-coding it into the system prompt. Each skill is a directory: `SKILL.md`
(the decision playbook) plus optional `references/` and `scripts/`.

- Three-layer loading (later overrides earlier): jar built-in <
  user-level `~/.paicli/skills/` < project-level `.paicli/skills/`.
- Enabled skill names + descriptions are injected into the system prompt; the
  agent calls `load_skill(name)` to pull in the full playbook on demand.
- Ships a built-in **web-access** skill with per-site browsing playbooks.
- Manage with `/skill list|show|on|off|reload`.

### Terminal UI

A `Renderer` interface with three implementations:

| Renderer | How to enable | Style |
|---|---|---|
| **inline streaming** (default) | run directly / `PAICLI_RENDERER=inline` | colorized splash, live thinking preview, a JLine-managed bottom status dock, foldable tool blocks, inline git diffs, single-character HITL prompts |
| **full-screen (lanterna)** | `PAICLI_RENDERER=lanterna` | three-pane layout: file tree + conversation + status bar + input, with modal HITL dialogs |
| **plain** | `PAICLI_RENDERER=plain` | plain output, no folding or status bar |

`NO_COLOR=1` disables ANSI colors while keeping layout; `PAICLI_NO_STATUSBAR=true`
disables the bottom dock in inline mode.

### Snapshots & rollback

- Each turn creates a `pre-turn` snapshot before running and a `post-turn`
  snapshot after.
- Snapshots use a pure-Java JGit repository stored under `~/.paicli/snapshots/`
  — they never touch your project's own `.git`.
- `/snapshot`, `/snapshot status`, `/snapshot clean`, and `/restore <N>` manage
  and roll back changes; the agent's `revert_turn` tool is HITL- and
  audit-gated.

### Background tasks & Runtime API

- A `DurableTaskManager` persists a background task queue in SQLite
  (`~/.paicli/tasks/tasks.db`) with an `enqueued → running → completed/failed/
  canceled` lifecycle, driven by a worker pool.
- Manage tasks with `/task`, `/task add`, `/task cancel`, `/task log`.
- A local Runtime API server exposes `POST /v1/threads`,
  `POST /v1/threads/{id}/turns`, and `GET /v1/threads/{id}/events`, and requires
  an API key:

  ```bash
  java -jar target/paicli-1.0-SNAPSHOT.jar serve --http --port 8080
  ```

### Additional capabilities

- **LSP diagnostics injection:** after a successful `write_file`, lightweight
  post-edit diagnostics (JavaParser-based for Java) are injected into the next
  request without blocking the tool flow.
- **Image input:** paste images with Ctrl+V or reference them with
  `@image:/path.png`; multimodal models (e.g. `glm-5v-turbo`) receive them as
  image blocks, while text-only providers get a text placeholder.
- **WeChat channel:** an optional WeChat iLink text channel bound via QR-code
  login (`/wechat`), running as a separate opt-in channel with a
  default-deny tool policy.

## Tech stack

- Java 17, Maven
- OkHttp (HTTP), Jackson (JSON)
- JLine 4 (terminal input, status area, interactive widgets)
- Lanterna (full-screen TUI)
- SQLite (vector store and graph persistence)
- JavaParser (AST analysis)
- JGit (side-history snapshots, no system git required)
- Jsoup (HTML extraction for `web_fetch`)
- ZXing (terminal QR-code rendering)
- Ollama (local embeddings)

## Quick start

### 1. Configure an API key

Copy `.env.example` to `.env` and fill in a provider key:

```bash
cp .env.example .env
# edit .env and add your API key
```

Or set it via environment variables:

```bash
export GLM_API_KEY=your_api_key_here
# or
export STEP_API_KEY=your_step_api_key_here
export STEP_MODEL=step-3.5-flash
# or
export KIMI_API_KEY=your_kimi_api_key_here
export KIMI_MODEL=kimi-k2.6
# or
export AGNES_API_KEY=your_agnes_api_key_here
export AGNES_MODEL=agnes-2.0-flash
export AGNES_BASE_URL=https://apihub.agnes-ai.com/v1
```

Or write config from inside Mini Claude Code (persisted to `~/.paicli/config.json`):

```text
/config provider agnes --api-key <key> --model agnes-2.0-flash --default
/model agnes
```

Default storage locations:

- Long-term memory: `~/.paicli/memory/long_term_memory.json`
- Code index (RAG): `~/.paicli/rag/codebase.db`
- Rolling debug logs: `~/.paicli/logs/paicli.log`

Any of these can be redirected with system properties, e.g.
`-Dpaicli.memory.dir=...`, `-Dpaicli.rag.dir=...`, `-Dpaicli.log.dir=...`.

### 2. (Optional) Configure MCP servers

The MCP subsystem is on by default. If `~/.paicli/mcp.json` does not exist,
Mini Claude Code creates a default `chrome-devtools` config. To add more servers, edit
`~/.paicli/mcp.json` or the project-level `.paicli/mcp.json`:

```json
{
  "mcpServers": {
    "fetch": {
      "command": "uvx",
      "args": ["mcp-server-fetch"]
    },
    "git": {
      "command": "uvx",
      "args": ["mcp-server-git", "--repository", "${PROJECT_DIR}"]
    },
    "remote-demo": {
      "url": "https://mcp.example.com/v1",
      "headers": {"Authorization": "Bearer ${REMOTE_TOKEN}"}
    }
  }
}
```

`command` denotes a stdio server; `url` denotes a Streamable HTTP server.
`${PROJECT_DIR}` and `${HOME}` are built-in; other `${VAR}` values come from the
environment and are reported at startup if missing.

### 3. Build & run

```bash
# build (tests are skipped by default for a faster, hand-verifiable jar)
mvn clean package

# run (requires a local Ollama with nomic-embed-text pulled;
# grep_code prefers ripgrep and falls back automatically if it is missing)
java -jar target/paicli-1.0-SNAPSHOT.jar
```

Or run directly from source:

```bash
mvn clean compile exec:java -Dexec.mainClass="com.paicli.cli.Main"
```

### 4. Entering Plan mode

The default mode is ReAct. To use Plan-and-Execute for a single task:

```text
/plan create a demo project, then read pom.xml, then verify the structure
```

After the plan is generated, the CLI pauses for confirmation:

- **Enter** — run the plan as-is
- **Ctrl+O** — expand the full plan
- **Esc** — collapse the plan or cancel it
- **I** — add extra requirements and re-plan

Execution returns to the default ReAct mode when the task completes.

## Testing

Daily development does not require the full test suite. `mvn clean package`
skips tests by default; run regressions by scope:

```bash
# terminal / TUI / inline renderer smoke tests
mvn test -Pphase16-smoke

# fast regression, skipping slow external-process / network / command paths
mvn test -Pquick

# a single test class
mvn test -Dtest=CodeSearchGoldenSetTest -DskipTests=false

# full suite (before a release or large refactor)
mvn test -DskipTests=false
```

## Commands

Interactive slash commands:

- `/plan [task]` — use Plan-and-Execute mode for the next task (or this one)
- `/team [task]` — use Multi-Agent mode for the next task (or this one)
- `/cancel` — request cancellation of the running task
- `/hitl on` / `/hitl off` / `/hitl` — toggle or inspect human-in-the-loop
  approval
- `/mcp` — list MCP server status
- `/mcp restart|logs|disable|enable|resources|prompts <name>` — manage a server
- `/policy` — view the security policy state
- `/audit [N]` — show the last N dangerous-tool audit records (default 10)
- `/snapshot` / `/snapshot status` / `/snapshot clean` — manage side-git snapshots
- `/restore <N>` — restore the Nth most recent pre-turn snapshot
- `/memory` / `/memory list|search|delete|clear` — inspect and manage memory
- `/save <fact>` — save a project-level fact (`--global` for cross-project)
- `/init` — generate a concise project `PAI.md` (`--force` to overwrite)
- `/index [path]` — index the codebase (defaults to the current directory)
- `/search <query>` — semantic (RAG) code search
- `/graph <class>` — view the code relationship graph
- `/model <name>` — switch model / provider
- `/config` — configure providers and defaults
- `/context` — show context mode, prompt-cache mode, RAG top-K, resource status
- `/skill list|show|on|off|reload` — manage skills
- `/browser status|connect|disconnect|tabs` — manage the browser session
- `/task` / `/task add|cancel|log` — manage background tasks
- `/wechat` / `/wechat setup|status|stop` — bind and run the WeChat channel
- `/export` — export the current session as Markdown
- `/clear` — clear the current conversation and short-term memory
- `/exit` / `/quit` — exit

Process-level entry points:

- `paicli serve --http --port 8080` — start the local Runtime API
- `paicli wechat setup|start|status` — set up and run the WeChat channel
- `paicli wechat daemon start|stop|restart|status|logs` — manage the WeChat
  background process

## Project structure

```
src/main/java/com/paicli
├── agent/
│   ├── Agent.java              # ReAct agent
│   ├── PlanExecuteAgent.java   # Plan-and-Execute agent
│   ├── AgentRole.java          # agent role enum
│   ├── AgentMessage.java       # inter-agent messages
│   ├── SubAgent.java           # configurable sub-agent
│   └── AgentOrchestrator.java  # Multi-Agent orchestrator
├── cli/
│   ├── Main.java               # CLI entry point
│   ├── CliCommandParser.java   # command parsing
│   └── PlanReviewInputParser.java
├── llm/
│   ├── GLMClient.java          # GLM client (coding + multimodal endpoints)
│   ├── DeepSeekClient.java
│   ├── StepClient.java         # StepFun client
│   ├── KimiClient.java         # Kimi / Moonshot client
│   ├── FreeLlmApiClient.java   # local OpenAI-compatible gateway
│   └── AgnesClient.java        # Agnes AI client
├── context/                    # context modes, model profiles, token/cost display
├── memory/                     # short-term, long-term, compression, budget, retrieval
├── plan/                       # Task, ExecutionPlan, Planner
├── rag/                        # embeddings, vector store, chunking, AST, code graph
└── tool/
    └── ToolRegistry.java       # tool registry
```
