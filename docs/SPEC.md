# abcd — Agent Build Context Data

## Specification

### Overview

`abcd` is a declarative DAG runner for composing LLM calls into materialized artifacts. Developers define **sources** (raw inputs) and **contexts** (prompt templates referencing upstream artifacts). The system resolves execution order, caches what hasn't changed, and only recomputes what's needed.

The CLI is designed for agent consumption first, with progressive disclosure for human readability.

---

### 1. Project Structure

```
my-project/
├── abcd.yml                  # Project config
├── contexts/
│   ├── extraction/
│   │   ├── summary.md
│   │   └── action_items.md
│   └── generation/
│       ├── email_draft.md
│       └── slack_message.md
├── target/                   # Materialized artifacts (gitignored)
│   ├── manifest.json
│   ├── extraction/
│   │   ├── summary.md
│   │   └── action_items.md
│   └── generation/
│       ├── email_draft.md
│       └── slack_message.md
└── .gitignore
```

- `contexts/` contains markdown files, one per context. Arbitrary subdirectories allowed.
- `target/` mirrors the `contexts/` directory structure.
- Context names are flat and unique — derived from filename minus extension, regardless of subdirectory.
- `target/` is gitignored. `target/manifest.json` tracks build state.

---

### 2. Project Config — `abcd.yml`

```yaml
name: my-project

defaults:
  model: openrouter/anthropic/claude-sonnet-4-20250514
  temperature: 0.3
  max_tokens: 4096

vars:
  tone: "professional"
  language: "english"

providers:
  file: abcd.providers.file.FileProvider

sources:
  transcript:
    provider: file
    path: ./data/meeting.txt
  tone:
    provider: file
    content: "professional and concise"
```

**Fields:**

| Field | Required | Description |
|---|---|---|
| `name` | Yes | Project name. Used for display and namespacing. |
| `defaults` | No | Default model config. Contexts override per-field. |
| `defaults.model` | No | Default LLM model string. |
| `defaults.temperature` | No | Default temperature. |
| `defaults.max_tokens` | No | Default max tokens. |
| `vars` | No | Project-level variables. Accessible via `{{ var('key') }}`. |
| `providers` | No | Provider registration. Key is provider name, value is Python import path. `file` provider is built-in. |
| `sources` | No | Named source declarations. Each source references a provider and provider-specific config. |

---

### 3. Context Files

Each context is a markdown file with YAML frontmatter and a prompt body.

```markdown
---
model: openrouter/openai/gpt-4o
temperature: 0.7
system: "You are a helpful assistant that writes concise emails."
description: "Generates a follow-up email from meeting summary and action items"
---

Write a follow-up email using the summary and action items.

{{ ref('summary') }}

{{ ref('action_items') }}
```

**Frontmatter fields:**

| Field | Required | Description |
|---|---|---|
| `model` | No | LLM model string. Falls back to `abcd.yml` defaults. |
| `temperature` | No | Sampling temperature. Falls back to defaults. |
| `max_tokens` | No | Max output tokens. Falls back to defaults. |
| `system` | No | System prompt sent as the system message. |
| `description` | No | Human/agent-readable description. Used in `abcd list`. |

Refs are **inferred** from the body — all `{{ ref('name') }}` and `{{ source('name') }}` calls are parsed at load time to build the DAG. No manual declaration needed.

**Context naming:** Filename minus extension. Must be unique across the entire project. Loading fails on duplicates.

---

### 4. Template Engine

Custom parser. Supports three functions:

| Function | Description |
|---|---|
| `{{ ref('name') }}` | Injects the materialized content of an upstream context. |
| `{{ source('name') }}` | Injects the content of a declared source. |
| `{{ var('key') }}` | Injects a project-level variable from `abcd.yml`. |

**Structured injection:** Refs and sources are expanded into XML-tagged blocks at call time:

```
Write a follow-up email using the summary and action items.

<ref name="summary">
[materialized content of summary]
</ref>

<ref name="action_items">
[materialized content of action_items]
</ref>
```

The user writes `{{ ref('summary') }}` in their markdown. The tagged expansion is a rendering concern — the LLM receives structured input it can distinguish from instructions.

---

### 5. Source Providers

Sources are the entry points of the DAG — data not produced by `abcd`.

**Provider protocol:**

```python
from typing import Protocol

class SourceProvider(Protocol):
    async def fetch(self, config: dict) -> str: ...
```

- `config` is the source declaration from `abcd.yml` (everything under the source key except `provider`).
- Returns the content as a string.

**Built-in provider: `file`**

Supports two config keys:

```yaml
sources:
  transcript:
    provider: file
    path: ./data/meeting.txt      # file path

  tone:
    provider: file
    content: "professional"        # inline string
```

**Third-party providers:**

Registered in `abcd.yml` via Python import path:

```yaml
providers:
  jira: my_abcd_jira.JiraProvider

sources:
  tickets:
    provider: jira
    project: ATLAS
    status: In Progress
```

`abcd` does not ship providers for external services. It provides the harness — community builds and owns providers.

**Determinism:** All source output is treated identically — content-hashed and compared. Providers may declare a `deterministic: bool` metadata field. When `false`, `abcd` warns the user that source output may vary between runs.

---

### 6. DAG Resolution

**Building the graph:**

1. Parse all context files in `contexts/` (recursive).
2. Extract `{{ ref() }}` and `{{ source() }}` calls from each body.
3. Build a directed graph: sources are leaf nodes, contexts are internal nodes.
4. Edges: context → its refs/sources.

**Validation (fails before any LLM call):**

- Cycles detected → error with cycle path.
- Missing ref/source → error with suggestion ("did you mean 'summary'?").
- Duplicate context names → error with both file paths.

**Execution order:** Topological sort. Independent branches run concurrently via `asyncio`.

---

### 7. Caching

**Cache key per context:** Hash of:

- Prompt template body (markdown content)
- Frontmatter config (model, temperature, max_tokens, system)
- Content hashes of all upstream artifacts (both refs and sources)
- Source provider configs

**How it works:**

1. On `abcd run`, compute current cache key for each context.
2. Compare to cache key stored in `manifest.json`.
3. If match → `cached`. Skip LLM call.
4. If mismatch → re-materialize. Write new artifact and update manifest.

**Manifest stores a `reason` field per context:**

| Reason | Meaning |
|---|---|
| `new` | Never materialized before |
| `cached` | No upstream or config changes |
| `upstream_changed: <name>` | Specific upstream artifact changed |
| `prompt_changed` | Template body was edited |
| `config_changed: <field>` | Frontmatter config field changed |
| `forced` | `--full-refresh` flag |

---

### 8. LLM Integration

**Provider:** OpenRouter first (OpenAI-compatible API). `OPENROUTER_API_KEY` env var for auth.

**Model strings:** Passed directly to the API. E.g. `openrouter/anthropic/claude-sonnet-4-20250514`, `openrouter/openai/gpt-4o`.

**Context size warning:** Before calling the LLM, estimate input token count (refs + source + prompt body + system prompt). Warn if estimated tokens approach or exceed the model's context window. Do not truncate. Still attempt the call — let the API return its own error if exceeded.

**No streaming in v1.** Wait for full response, materialize, update manifest.

---

### 9. CLI

Designed for agents first. Progressive disclosure.

**Output layers:**

| Mode | When | Output |
|---|---|---|
| Default | TTY | Rich-formatted. Status dots, context names, pass/fail. |
| `--verbose` | Flag | Timestamps, token estimates, model used per context, reason for (re)materialization. |
| `--json` | Flag | Full structured JSON. All state machine-readable. |
| `--quiet` | Flag | Suppress all stdout. Only exit code. |

**Exit codes:**

| Code | Meaning |
|---|---|
| 0 | All contexts materialized successfully |
| 1 | One or more failures (fail-fast or final report with `--keep-going`) |

**Commands:**

#### `abcd init <name>`

Create a new project with example context and `.gitignore`.

```
$ abcd init my-project
Created my-project/
├── abcd.yml
├── contexts/
│   └── example.md
└── .gitignore
```

#### `abcd run [selectors...] [flags]`

Run the DAG. Materialize all contexts or selected subset.

**Selectors (dbt-style):**

| Selector | What runs |
|---|---|
| `summary` | Just that context |
| `+summary` | summary + all upstream |
| `summary+` | summary + all downstream |
| `+summary+` | summary + upstream + downstream |
| `2+summary` | summary + 2 levels upstream |
| `summary+3` | summary + 3 levels downstream |

Multiple selectors can be combined: `abcd run +email_draft +slack_message`

**Flags:**

| Flag | Description |
|---|---|
| `--select <selector>` | Alternative syntax for selectors |
| `--full-refresh` | Ignore cache, rerun everything |
| `--keep-going` | Continue on failure, skip dependents, report all errors at end |
| `--project-dir <path>` | Project root directory (default: cwd) |
| `--verbose` | Detailed output |
| `--json` | Machine-readable JSON output |
| `--quiet` | Suppress stdout |

**Default output (TTY):**

```
$ abcd run

  source transcript    ✓ cached
  summary              ● running... ✓ (3.2s)
  action_items         ● running... ✓ (2.8s)
  email_draft          ● running... ✓ (3.1s)

  3 materialized, 1 cached, 0 errors (9.1s)
```

**`--json` output:**

```json
{
  "project": "my-project",
  "contexts": {
    "summary": {
      "status": "materialized",
      "reason": "upstream_changed: transcript",
      "duration_ms": 3200,
      "model": "openrouter/anthropic/claude-sonnet-4-20250514",
      "tokens_in": 1200,
      "tokens_out": 450,
      "hash": "sha256:abc123..."
    },
    "action_items": {
      "status": "materialized",
      "reason": "upstream_changed: summary",
      "duration_ms": 2800,
      "model": "openrouter/anthropic/claude-sonnet-4-20250514",
      "tokens_in": 800,
      "tokens_out": 200,
      "hash": "sha256:def456..."
    },
    "email_draft": {
      "status": "cached",
      "reason": "cached",
      "hash": "sha256:ghi789..."
    }
  },
  "duration_ms": 9100,
  "errors": []
}
```

#### `abcd list`

Show project inventory.

```
$ abcd list

  Sources:
    transcript (file)

  Contexts:
    summary          → [transcript]
    action_items     → [summary]
    email_draft      → [summary, action_items]
    slack_message    → [summary]
```

`--json` returns full state:

```json
{
  "sources": {
    "transcript": {
      "provider": "file",
      "cached": true,
      "hash": "sha256:..."
    }
  },
  "contexts": {
    "summary": {
      "refs": ["transcript"],
      "materialized": true,
      "hash": "sha256:...",
      "description": "Summarizes meeting transcript"
    },
    "email_draft": {
      "refs": ["summary", "action_items"],
      "materialized": false,
      "description": "Generates a follow-up email"
    }
  }
}
```

#### `abcd graph`

Render the dependency DAG as an ASCII tree.

```
$ abcd graph

transcript (source)
├── summary
│   ├── action_items
│   │   └── email_draft
│   └── email_draft
└── slack_message
```

#### `abcd show <context>`

Print materialized artifact content to stdout. Raw text, no formatting.

```
$ abcd show summary
[outputs content of target/extraction/summary.md]
```

#### `abcd validate`

Validate the DAG without running. Checks for cycles, missing refs, duplicate names.

---

### 10. Manifest — `target/manifest.json`

```json
{
  "project": "my-project",
  "created_at": "2026-04-30T14:32:00Z",
  "updated_at": "2026-04-30T14:32:10Z",
  "contexts": {
    "summary": {
      "hash": "sha256:abc123...",
      "cache_key": "sha256:def456...",
      "reason": "upstream_changed: transcript",
      "model": "openrouter/anthropic/claude-sonnet-4-20250514",
      "materialized_at": "2026-04-30T14:32:04Z",
      "upstream_hashes": {
        "transcript": "sha256:src789..."
      }
    }
  },
  "sources": {
    "transcript": {
      "hash": "sha256:src789...",
      "provider": "file",
      "fetched_at": "2026-04-30T14:32:01Z"
    }
  }
}
```

---

### 11. Parallelism

Independent branches of the DAG run concurrently using `asyncio`. LLM API calls are IO-bound — no reason to wait sequentially when two contexts share no dependency relationship.

A `--sequential` flag may be added for debugging.

---

### 12. Error Handling

**Default: fail fast.** First error stops the run. Already-materialized artifacts are kept.

**`--keep-going`:** Continue running independent branches. Skip contexts whose upstream failed. Report all errors at the end.

**DAG validation errors** (cycles, missing refs, duplicate names) always fail before any LLM call — regardless of flags.

---

### 13. Authentication

Single env var: `OPENROUTER_API_KEY`.

No config file for secrets. No interactive prompts.

---

### 14. Technical Stack

| Component | Choice |
|---|---|
| Language | Python 3.12+ |
| Package manager | uv |
| CLI framework | Click |
| Terminal output | Rich |
| LLM client | OpenAI SDK (OpenRouter-compatible) |
| Config parsing | PyYAML |
| Template engine | Custom parser (regex-based, ~20 lines) |
| Async runtime | asyncio |

**Dependencies:**

- `click`
- `rich`
- `openai`
- `pyyaml`

Everything else is stdlib: `asyncio`, `hashlib`, `pathlib`, `json`, `re`, `fnmatch`.

---

### 15. `abcd init` Template

**`abcd.yml`:**

```yaml
name: ${PROJECT_NAME}

defaults:
  model: openrouter/anthropic/claude-sonnet-4-20250514
  temperature: 0.3

providers:
  file: abcd.providers.file.FileProvider
```

**`contexts/example.md`:**

```markdown
---
description: "An example context — edit or replace me"
system: "You are a helpful assistant."
---

Summarize the following text concisely.

{{ source('input') }}
```

**`.gitignore`:**

```
target/
```

---

### 16. Scope — What abcd Does NOT Do

- No knowledge graph
- No vector retrieval / RAG
- No schema inference or output validation
- No auto-suggested refs
- No streaming output
- No built-in providers for external services (only `file`)
- No pure transformations (every context calls an LLM)
- No interactive prompts or TUI
- No sub-agents or agent orchestration

abcd is a deterministic build system for LLM context. DAG resolution, content-addressed caching, selective execution. That's the product.
