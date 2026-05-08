# abcd

**Agent Build Context Data** — the runtime for contexts.

Contexts are prompt templates that declare dependencies on data sources and each other. abcd resolves the dependency graph, binds references to their values, and materializes artifacts. Only reruns what's stale.

## Quick start

Define sources and contexts:

```yaml
# abcd.yml
name: my-project

sources:
  transcript:
    description: "The full meeting transcript from the product sync on 2026-05-05"
  open_tickets:
    description: "Query Jira project ATLAS for tickets with status 'In Progress'"
```

```markdown
<!-- contexts/extraction/summary.md -->
---
description: "Extracts a summary from the meeting transcript"
---

Extract a structured summary of the meeting.

<source name="transcript" />
```

```markdown
<!-- contexts/extraction/action_items.md -->
---
description: "Extracts action items from the meeting transcript and summary"
expect: json
---

Extract action items.

<source name="transcript" />
<ref name="summary" />
```

```markdown
<!-- contexts/generation/email_draft.md -->
---
description: "Writes a follow-up email from the summary and action items"
---

Write a follow-up email.

<ref name="summary" />
<ref name="action_items" />
```

Run:

```
$ abcd run email_draft

  transcript      ● spec injected
  summary         ● bound → running... ✓ fresh
  action_items    ● bound → running... ✓ fresh
  email_draft     ● bound → running... ✓ fresh
```

## Concepts

| Concept | What it is |
|---|---|
| **Context** | A prompt template with dependencies. The primary unit of work. |
| **Source** | A spec — describes expected data, not the data itself. Fulfilled by the executor at runtime. |
| **Ref** | `<ref name="..." />` — injects the materialized artifact of another context. Makes the DAG explicit. |
| **Artifact** | The output of a context. Text or JSON on disk. |
| **Freshness** | Tracks whether an artifact is still valid. Stale when inputs or context definition changed. |

## CLI

```
abcd run <context>              Bind + execute + materialize
abcd run <context> --input ...  Record externally-produced artifact (agent-driven)
abcd bind <context>             Resolve references, produce bound prompt
abcd status <context>           Show freshness: current state, why stale, what changed
abcd list                       List all contexts
abcd show <context>             Context metadata and current artifact
abcd graph                      Print the dependency graph
abcd validate                   Check graph without executing
abcd init <name>                Scaffold new project
```

## Docs

- [CONTEXT.md](docs/CONTEXT.md) — domain language, glossary, and decisions
- [SPEC.md](docs/SPEC.md) — detailed design specification
- [ADR-0001](docs/adr/0001-python-over-go.md) — Python over Go
- [ADR-0002](docs/adr/0002-xml-tags-over-jinja.md) — XML tags over Jinja
