# abcd

**Agent Build Context Data** — dbt for context engineering.

You compose LLM calls into a DAG. Each model references upstream outputs, runs a prompt, and materializes an artifact. The system resolves execution order, caches what hasn't changed, and only recomputes what's needed.

## Why

Everyone hand-wires multi-step LLM calls in scripts. No hashing, no caching, no reuse. Every run is from scratch. `abcd` treats prompt chains like a build system: declarative models, content-addressed caching, selective execution.

## How it works

Define sources and models in YAML:

```yaml
sources:
  transcript: ./meeting.txt

models:
  summary:
    prompt: "Summarize: {{ ref('transcript') }}"

  action_items:
    prompt: "Extract action items from: {{ ref('summary') }}"

  email_draft:
    prompt: "Write a follow-up email using {{ ref('summary') }} and {{ ref('action_items') }}"
```

Run the DAG:

```
$ abcd run

  transcript    ✓ cached
  summary       ● running... ✓ materialized
  action_items  ● running... ✓ materialized
  email_draft   ● running... ✓ materialized
```

Only rerun what changed:

```
$ abcd run --select email_draft

  transcript    ✓ cached
  summary       ✓ cached
  action_items  ✓ cached
  email_draft   ● running... ✓ materialized
```

## Concepts

| Concept | What it is |
|---|---|
| **Source** | Raw input — a file, text, API response. Declared, not produced by the system. |
| **Model** | A prompt template + references to upstream artifacts. Running a model calls an LLM and materializes the output. |
| **Ref** | `{{ ref('name') }}` injects the materialized content of another artifact. Makes the DAG explicit. |
| **Artifact** | The materialized output of a model. Text or JSON on disk. Diffable, version-controllable. |
| **Cache** | Content-hashed. A model reruns only when its prompt or any upstream artifact changes. |

## Core features

- **Declarative models.** YAML + `{{ ref() }}` replaces ad-hoc prompt chaining scripts.
- **DAG resolution.** Define models, the system figures out execution order.
- **Content-hashed caching.** Change a source? Only downstream models rerun.
- **Selective execution.** `--select` runs only what's needed to produce a target artifact.
- **Plain artifacts.** Output is text/JSON on disk. Composable, diffable, version-controllable.

## What it doesn't do

No knowledge graph. No vector retrieval. No schema inference. No auto-suggested refs. Just a deterministic build system for LLM context.

## CLI

```
abcd run                    Run all models in dependency order
abcd run --select <model>   Run only what's needed to produce target
abcd run --full-refresh     Ignore cache, rerun everything
abcd list                   List all sources and models
abcd show <model>           Display a model's materialized artifact
abcd graph                  Print the dependency graph
```

## Status

Early concept. See [SPEC.md](docs/SPEC.md) for detailed design.
