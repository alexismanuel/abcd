# abcd — Agent Build Context Data

## The Problem

You're working with an agent. You need the right context at the right time — a meeting summary, a list of action items, a drafted email. Each piece depends on upstream data or other derived context. Today you:

- Manually assemble prompts by copying artifacts between files
- Re-derive everything when a source changes because nothing tracks dependencies
- Have no record of what was produced, from what, or whether it's still current
- Duplicate prompt patterns across projects because there's no reusable structure

Context engineering is ad-hoc, manual, and fragile. You need a runtime for it.

---

## What abcd Does

abcd is the runtime for contexts. It models knowledge as a dependency graph of **sources** (data specs) and **contexts** (prompt templates that reference upstream artifacts). Given a context name, it:

1. Resolves what upstream data is needed
2. Binds all references to their values, producing a self-contained prompt
3. Invokes an executor to produce the artifact
4. Tracks freshness — what was produced, when, and whether it's still valid

**Why abcd instead of nothing:**

| Without abcd | With abcd |
|---|---|
| Manually copy-paste artifacts into prompts | `abcd run summary` — refs resolve automatically |
| Re-derive everything on source changes | Freshness tracking skips what's still valid |
| No visibility into what depends on what | `abcd graph` shows the full DAG |
| Prompt patterns live in chat history, not files | Contexts are versioned, composable, shareable |
| No way to reproduce a context from a clean state | `abcd run --full-refresh` from any point |
| Knowledge leaves with the person who built it | Shared contexts let the team invoke the same knowledge |
| New team members have no idea what's available | `abcd list` — discover every invocable context instantly |

---

## Architecture

### Primitives

| Primitive | What |
|---|---|
| **Sources** | Data specs — described in natural language, not fetched by abcd. The executor fulfills them. |
| **Contexts** | Prompt templates that reference upstream sources and contexts. Composable, discoverable, shareable. |
| **Manifest** | `target/manifest.json`. The state ledger — tracks freshness, not history. |

### Two Modes

**Standalone (for production orchestrators):**
```
abcd run summary
```
abcd resolves the DAG, checks freshness, binds each context, invokes the configured executor subprocess (stdin → stdout), writes artifacts, updates manifest. Designed for schedulers like Dagster or Airflow calling abcd as a step in a pipeline.

**Agent-driven (for AI agents):**
```
abcd bind summary --stdout    # agent gets the bound prompt
# agent does its thing
abcd run summary --input result.txt  # abcd records the result
```
abcd still resolves the DAG and checks freshness. The agent is the executor — it receives bound prompts, produces artifacts, hands them back. The caller agent asks for a target context and receives the final artifact. It never sees the chain.

In both modes, abcd owns the DAG. The executor (subprocess or agent) owns execution.

### The Seam

abcd knows **what** and **when** — what dependencies exist, what order to resolve them, what's stale. The executor knows **how** — how to fetch from Jira, how to call GitHub, how to reason. The executor receives a bound prompt containing all source specs and ref content. It doesn't know the DAG, just executes what it's given.

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
- Context names: filename minus extension. Unique across the project.

---

### Config — `abcd.yml`

```yaml
name: my-project

executor: pi --print

defaults:
  model: openrouter/anthropic/claude-sonnet-4-20250514
  temperature: 0.3
  max_tokens: 4096

sources:
  transcript:
    description: "Read the file at sources/transcript.md — the full meeting transcript from the product sync on 2026-05-05"
  open_tickets:
    description: "Query Jira project ATLAS for tickets with status 'In Progress'. Use Jira skills to fetch current ticket summaries, priorities, and assignees."
```

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Project name. |
| `executor` | No | Command to invoke in standalone mode. Receives bound prompt on stdin, artifact expected on stdout. Default: `pi --print`. |
| `defaults` | No | Default executor config (model, temperature, max_tokens). Passed to executor via env vars or arguments (executor-specific). Only used in standalone mode — in agent-driven mode, the agent decides. |
| `sources` | No | Named source specs. Each has a freeform `description` — a spec that tells the executor what data is expected. |

---

### Context Files

Markdown with YAML frontmatter + prompt body.

```markdown
---
description: "Extracts a summary from the meeting transcript"
expect: json
---

Extract a structured summary of the meeting.

<source name="transcript" />
```

| Field | Required | Description |
|---|---|---|
| `description` | **Yes** | What this context produces. The public interface — shown in `list`, used by agents and humans to decide whether to invoke it. |
| `expect` | No | Structural acceptance check. `non-empty` or `json`. Fails the run if output doesn't match. Default: no expectations. |

Dependencies are **inferred** from the body. `<ref name="..." />` and `<source name="..." />` are parsed at load time. No manual declaration.

Frontmatter is intentionally thin — only context-level concerns. Model, temperature, max_tokens, and system prompt belong to the executor, not the context.

---

### Template Syntax

Two tags:

| Tag | Expansion |
|---|---|
| `<ref name="..." />` | The produced artifact content from `target/`. Baked in — it's already been generated and accepted. |
| `<source name="..." />` | The source spec from config. Not the data itself — instructions for the executor to fulfill. |

Input is a self-closing tag; expansion wraps the content:

**Ref (baked artifact):**
```xml
<!-- input -->
<ref name="summary" />

<!-- expanded output -->
<ref name="summary">
[full content of target/extraction/summary.md]
</ref>
```

**Source (spec):**
```xml
<!-- input -->
<source name="transcript" />

<!-- expanded output -->
<source name="transcript">
Read the file at sources/transcript.md — the full meeting transcript from the product sync on 2026-05-05
</source>
```

Sources and refs are fundamentally different:

- **Refs** are baked from materialized artifacts. Deterministic, tracked for freshness. An artifact changes → downstream refs are stale.
- **Sources** are specs. The executor fulfills them fresh at every execution. Always live by structure — abcd never holds source data, only the spec.

---

### DAG Resolution

1. Parse source declarations from `abcd.yml`.
2. Scan `contexts/` — every markdown file is a context node.
3. Parse `<ref />` and `<source />` tags from each context body.
4. Build directed graph: sources are leaves, contexts are internal nodes.

**Validation (fails before any operation):**

- Cycles
- Missing ref/source (with suggestion)
- Duplicate context names
- Source referenced in template but not declared in config

**Execution:** Topological sort. Independent branches run concurrently.

---

### Freshness

An artifact is **fresh** when nothing that affects its output has changed since it was last accepted. Freshness is tracked per-context via content hashes stored in the manifest.

**Invalidation reasons:**

| Reason | Meaning |
|---|---|
| `new` | Never produced |
| `upstream_changed: <name>` | An upstream artifact or source spec changed |
| `prompt_changed` | Context prompt body was edited |
| `config_changed` | Context frontmatter changed |
| `forced` | `--full-refresh` |

Freshness is not a cache. It's a record of accepted decisions — the artifact was produced, accepted, and nothing has invalidated it since. `--full-refresh` overrides freshness and forces re-execution.

Sources are excluded from freshness tracking because they are always live — the executor fulfills them fresh at every run.

---

### Acceptance

A context can optionally declare structural expectations on its output via `expect` in frontmatter:

- `non-empty` — artifact must contain content
- `json` — artifact must be valid JSON

If the produced artifact doesn't match, the run fails for that context. No semantic validation — the executor decides if the output is *good*, abcd only checks if it's *shaped right*. Default: no expectations.

---

### Manifest

`target/manifest.json` — the state ledger.

Per context: content hash, input hashes, context definition hash, timestamp, executor used, freshness reason.

Not a cache. No history. Current state only. Versioning is git's job.

---

## CLI

Agent-first. Progressive disclosure.

**Output modes:** default (rich TTY), `--verbose`, `--json`, `--quiet`.
**Exit codes:** 0 success, 1 failure.
**Default behavior:** commands write to `target/` and print the path. `--stdout` prints content.

### Core Commands

```
abcd bind <context> [--stdout]
```
Resolve all refs and sources. Write bound prompt to `target/`. Print path (or content with `--stdout`). No execution.

```
abcd run <context>
```
Bind + execute + materialize. Resolves the DAG, checks freshness for the target and all upstream contexts, executes stale contexts in order, writes artifacts, updates manifest. This is the standard end-to-end command.

```
abcd run <context> --input <path>
abcd run <context> --stdin
```
Record an externally-produced artifact. Used in agent-driven mode: the agent executes the bound prompt and records the result via this command. Validates against `expect` if declared, writes to `target/`, updates manifest.

```
abcd status <context>
```
Show freshness for a given context: current state, whether stale, why, what upstream changed.

### Inspection Commands

```
abcd list
```
Discover available contexts. Name and description only — like listing skills.

```
abcd show <context>
```
Full context metadata: description, frontmatter config, refs, sources, prompt body, materialization status.

```
abcd graph             # DAG as tree
abcd validate          # Check graph without executing
```

### Project Commands

```
abcd init <name>       # Scaffold new project
```

### Selectors

Available on `run`:

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
| `--full-refresh` | Ignore freshness, rerun everything |
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
| Language | Python |
| CLI | typer or click |
| Terminal | rich |
| Config | PyYAML |
| Concurrency | asyncio |
| Executor | Subprocess invocation (configurable command) |
| Distribution | pip / uv tool |

---

## Scope — What abcd Does NOT Do

- No LLM client — execution is delegated to the configured executor or agent
- No provider protocol — sources are specs, fulfilled by the executor
- No agent orchestration — abcd owns the DAG, the executor owns reasoning
- No actions on the world — abcd produces text artifacts
- No vector retrieval / RAG
- No streaming output
- No interactive prompts or TUI
- No scheduling or proactive triggers
- No multi-user, permissions, or governance
- No schema validation or structured output in v1
- No prompt optimization
- No cost tracking in v1

---

## Skill

abcd ships with an agent skill that teaches agents how to author contexts well. Beyond syntax, it guides agents to behave like domain experts: asking clarifying questions, defining terms, aligning on language, challenging assumptions before writing prompts. The discipline of building shared understanding before building context.
