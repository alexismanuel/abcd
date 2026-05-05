# abcd — Agent Build Context Data

## The Problem

You're working with an agent. You need the right context at the right time — a meeting summary, a list of action items, a drafted email. Each piece depends on upstream data or other derived context. Today you:

- Manually assemble prompts by copying artifacts between files
- Re-derive everything when a source changes because nothing tracks dependencies
- Have no record of what was produced, from what, or whether it's still current
- Duplicate prompt patterns across projects because there's no reusable structure

Context engineering is ad-hoc, manual, and fragile. You need a build system for it.

---

## What abcd Does

`abcd` is a context build system. It models knowledge as a dependency graph of **sources** (raw data) and **contexts** (prompt templates that reference upstream artifacts). Given a context name, it:

1. Resolves what upstream data is needed
2. Assembles the fully hydrated prompt
3. Invokes an executor to produce the artifact
4. Tracks what was produced, when, and from what inputs

It's Make for context. Make resolves C file dependencies and invokes gcc. abcd resolves context dependencies and invokes an agent. The agent does the reasoning — abcd does the plumbing.

**Why abcd instead of nothing:**

| Without abcd | With abcd |
|---|---|
| Manually copy-paste artifacts into prompts | `abcd build summary` — refs resolve automatically |
| Re-derive everything on source changes | Content-addressed cache skips what's current |
| No visibility into what depends on what | `abcd graph` shows the full DAG |
| Prompt patterns live in chat history, not files | Contexts are versioned, composable, shareable |
| No way to reproduce a context from a clean state | `abcd build --full-refresh` from any point |
| Knowledge leaves with the person who built it | Shared contexts let the team invoke the same knowledge |
| New team members have no idea what's available | `abcd list` — discover every invocable context instantly |

---

## Architecture

### Primitives

| Primitive | What |
|---|---|
| **Sources** | Declarative data dependencies. Described in English — what the data is, where to find it, how to fetch it. Not fetched by abcd. |
| **Contexts** | Invocable units of knowledge. Prompt templates that reference upstream sources and contexts. The core product — composable, discoverable, shareable. |
| **Manifest** | `target/manifest.json`. Tracks what's been produced, from what inputs, when. |

### Two Modes

**Standalone (build command):**
```
abcd build summary
```
abcd assembles the prompt, invokes the configured executor (default: `pi --print`), writes the result to `target/`, updates the manifest.

**Agent-driven (assemble + put):**
```
abcd assemble summary --stdout    # agent gets the prompt
# agent does its thing
abcd put summary --in result.txt  # agent records the result
```
The agent handles execution. abcd handles graph resolution and state tracking.

---

### Project Structure

```
my-project/
├── abcd.yml
├── contexts/
│   ├── extraction/
│   │   ├── summary.md
│   │   └── action_items.md
│   └── generation/
│       └── email_draft.md
├── target/
│   ├── manifest.json
│   ├── extraction/
│   │   ├── summary.md
│   │   └── action_items.md
│   └── generation/
│       └── email_draft.md
└── .gitignore
```

- `contexts/` — prompt templates. One markdown file per context. Arbitrary subdirectories.
- `target/` — materialized artifacts. Mirrors `contexts/`. Gitignored.
- Context names: filename minus extension. Flat, unique across the project.

---

### Config — `abcd.yml`

```yaml
name: my-project

executor: pi --print

defaults:
  model: openrouter/anthropic/claude-sonnet-4-20250514
  temperature: 0.3
  max_tokens: 4096

vars:
  tone: "professional"

sources:
  transcript:
    description: "Read the file at sources/transcript.md — the full meeting transcript from the product sync on 2026-05-05"
  open_tickets:
    description: "Query Jira project ATLAS for tickets with status 'In Progress'. Use Jira skills to fetch current ticket summaries, priorities, and assignees."
```

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Project name. |
| `executor` | No | Command to invoke for `build`. Receives assembled prompt on stdin, artifact expected on stdout. Default: `pi --print`. |
| `defaults` | No | Default model config. Passed to executor via env vars or arguments (executor-specific). |
| `vars` | No | Project variables. Accessible via `{{ var('key') }}`. |
| `sources` | No | Named source declarations. Each has a freeform `description` — English instructions for the agent on what data to fetch and how. abcd includes this description in assembled prompts. |

Sources are not fetched by abcd. The description is a contract: it tells the executing agent what data is expected. The agent fetches using whatever tools or skills it has. This keeps abcd decoupled from the external world — no provider protocol, no API clients, no plugin system.

---

### Context Files

Markdown with YAML frontmatter + prompt body.

```markdown
---
model: openrouter/openai/gpt-4o
temperature: 0.7
system: "You extract structured information from meeting transcripts."
description: "Extracts a summary from the meeting transcript"
---

Extract a structured summary of the meeting.

{{ source('transcript') }}
```

| Field | Required | Description |
|---|---|---|
| `description` | **Yes** | What this context produces. The public interface — shown in `list`, used by agents and humans to decide whether to invoke it. |
| `model` | No | Model hint. Passed to executor. Falls back to defaults. |
| `system` | No | System prompt. |
| `temperature` | No | Sampling temperature. Falls back to defaults. |
| `max_tokens` | No | Max output tokens. Falls back to defaults. |

Refs are **inferred** from the body. `{{ ref() }}` and `{{ source() }}` parsed at load time. No manual declaration.

---

### Template Engine

Three functions:

| Function | Expansion |
|---|---|
| `{{ ref('name') }}` | The produced artifact content from `target/`. Baked in — it's already been generated and versioned. |
| `{{ source('name') }}` | The source description from config. Not the data itself — instructions for the agent to fetch it. |
| `{{ var('key') }}` | The variable value from config. |

Refs and sources expand differently:

**Ref (baked artifact):**
```xml
<ref name="summary">
[full content of target/extraction/summary.md]
</ref>
```

**Source (fetch instruction):**
```xml
<source name="transcript">
Read the file at sources/transcript.md — the full meeting transcript from the product sync on 2026-05-05
</source>
```

This split is intentional. Sources are live data — their content may change between runs, and freshness is the agent's responsibility. Baking stale content into the prompt defeats the purpose. Refs are produced artifacts — they're deterministic, versioned in the manifest, and safe to include verbatim.

---

### DAG Resolution

1. Parse source declarations from `abcd.yml`.
2. Scan `contexts/` — every markdown file is a context node.
3. Parse template calls from each context body.
4. Build directed graph: sources are leaves, contexts are internal nodes.

**Validation (fails before any operation):**

- Cycles
- Missing ref/source (with suggestion)
- Duplicate context names
- Source referenced in template but not declared in config

**Execution:** Topological sort. Independent branches run concurrently.

---

### Caching

Cache key per context: hash of prompt body + frontmatter config + upstream artifact hashes + source descriptions.

Match → skip. Mismatch → re-execute. `put` always writes (explicit decision to record).

**Reasons:**

| Reason | Meaning |
|---|---|
| `new` | Never produced |
| `cached` | No changes detected |
| `upstream_changed: <name>` | Upstream artifact or source changed |
| `prompt_changed` | Template body edited |
| `config_changed: <field>` | Frontmatter config changed |
| `produced_externally` | Written via `put` |
| `forced` | `--full-refresh` |

---

### Manifest

`target/manifest.json` — single source of truth for build state.

Per context: cache key, content hash, reason, model used, timestamp, source (executor or external), upstream hashes.

---

## CLI

Agent-first. Progressive disclosure.

**Output modes:** default (rich TTY), `--verbose`, `--json`, `--quiet`.
**Exit codes:** 0 success, 1 failure.
**Default behavior:** commands write to `target/` and print the path. `--stdout` prints content.

### Core Commands

```
abcd assemble <context> [--stdout]
```
Hydrate all refs and sources. Write assembled prompt to `target/`. Print path (or content with `--stdout`). No execution.

```
abcd build <context>
```
Assemble + invoke executor + write result to `target/` + update manifest. This is the standard end-to-end command for standalone usage.

```
abcd put <context> --in <path>
abcd put <context> --stdin
```
Record an externally-produced result. Validate it exists, write to `target/`, update manifest. Used in agent-driven workflows.

```
abcd status
```
Show what's current, what's stale, what's missing, and why. The starting point for any agent deciding what to produce.

### Inspection Commands

```
abcd list
```
Discover available contexts. Name and description only — like listing skills. The entry point for anyone new to the project.

```
abcd show <context>
```
Full context metadata: description, frontmatter config, refs, sources, prompt body, materialization status. Like inspecting a skill. The detail view behind `list`.

```
abcd graph             # DAG as ASCII tree
abcd validate          # Check graph without executing
```

### Project Commands

```
abcd init <name>       # Scaffold new project
```

### Selectors (dbt-style)

Available on `build`:

| Selector | What runs |
|---|---|
| `summary` | Just that context |
| `+summary` | summary + all upstream |
| `summary+` | summary + all downstream |
| `+summary+` | summary + upstream + downstream |
| `2+summary` | summary + 2 levels upstream |
| `summary+3` | summary + 3 levels downstream |

### Flags

| Flag | Description |
|---|---|
| `--full-refresh` | Ignore cache |
| `--keep-going` | Continue on failure |
| `--project-dir <path>` | Project root (default: cwd) |
| `--stdout` | Print content instead of path |
| `--verbose` | Detailed output |
| `--json` | Machine-readable |
| `--quiet` | Suppress stdout |

---

## Implementation

| Component | Choice |
|---|---|
| Language | Go |
| CLI | cobra |
| Terminal | bubbletea + lipgloss |
| Config | gopkg.in/yaml.v3 |
| Concurrency | goroutines |
| Executor | Subprocess invocation (configurable command) |
| Distribution | Single binary |

---

## Scope — What abcd Does NOT Do

- No LLM client — execution is delegated to the configured executor
- No provider protocol — sources are described in English, fetched by agents
- No agent orchestration — abcd never calls an agent (it calls an executor)
- No actions on the world — abcd produces text artifacts
- No vector retrieval / RAG
- No streaming output
- No interactive prompts or TUI
- No scheduling or proactive triggers
- No multi-user, permissions, or governance
- No schema validation or structured output in v1
- No prompt optimization
