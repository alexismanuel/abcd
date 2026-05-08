# abcd — Domain Context

The runtime for contexts. Anyone wanting to organize knowledge using a language defines contexts; abcd resolves, binds, and materializes them.

## Language

**Context**:
A prompt template that declares dependencies on upstream sources and other contexts. The primary unit of work — composable, discoverable, shareable. Defined as a markdown file in `contexts/`.
_Avoid_: Model (reserved for LLM models), prompt, template

**Artifact**:
The materialized output of a context. Text or JSON on disk in `target/`. Current state only — no history, no versioning inside abcd.
_Avoid_: Output, result, response

**Source**:
A data spec — describes the shape and location of expected data, not the data itself. Like an interface, not an implementation. Fulfilled fresh by the executor at every execution. abcd never holds source data, only the spec. Sources are structurally always live.
_Avoid_: Input, data, dependency, fetch instruction

**Spec**:
The natural-language description inside a source. Can be anything from "the meeting transcript from the product sync on 2026-05-05" to "use `gh issue view 42` to fetch the GitHub issue." Doesn't prescribe the method, only the expected result. The executor reads the spec and produces data that satisfies it.

**Ref**:
A reference from one context to another context's artifact. Makes the DAG explicit. Baked from materialized artifacts — deterministic, tracked for freshness.

**Bind**:
The core operation. Take an unbound context (template with unresolved references) and resolve each ref and source to its value. Produces a **bound prompt** — self-contained, dependency-free, ready for execution.
_Avoid_: Compile, render, assemble, hydrate

**Bound prompt**:
The output of binding. A complete prompt with all refs expanded to artifact content and all sources expanded to specs. Ready for the executor.
_Avoid_: Assembled prompt, hydrated prompt, compiled prompt

**Freshness**:
Whether an artifact is still valid. An artifact is **fresh** when nothing that affects its output has changed since last acceptance. Stale when upstream changed, context definition changed, or never produced. Not a cache — a record of accepted decisions.
_Avoid_: Cache, cache hit, cached

**Stale**:
An artifact whose inputs have changed since it was last accepted. Needs re-execution.
_Avoid_: Invalid, expired, outdated

**Acceptance**:
A structural check on an artifact. A context optionally declares `expect` (e.g. `non-empty`, `json`). abcd checks the shape — the executor decides if the content is good.
_Avoid_: Validation, verification

**Manifest**:
`target/manifest.json`. The state ledger — tracks freshness per context: content hashes, input hashes, timestamps. Not a cache, not a history. Current state only.
_Avoid_: Cache, state file, metadata

**Executor**:
The thing that takes a bound prompt and produces an artifact. In standalone mode, a subprocess (stdin → stdout). In agent-driven mode, an AI agent. abcd never reasons — the executor does.
_Avoid_: Agent (when referring to the generic concept), runner, provider

**The seam**:
The boundary between abcd and the executor. abcd knows **what** and **when** (dependencies, order, freshness). The executor knows **how** (fetching, reasoning, producing). The executor receives a bound prompt and doesn't see the DAG.

## Relationships

- A **Context** depends on zero or more **Sources** and zero or more other **Contexts** (via **Refs**)
- A **Source** contains exactly one **Spec**
- A **Source** is always live — structurally guaranteed, not tracked
- A **Ref** is always stale-tracked — it points to a materialized **Artifact**
- Binding a **Context** produces a **Bound prompt**
- Running a **Context** produces an **Artifact**
- An **Artifact** passes **Acceptance** checks before being written to **target/**
- The **Manifest** records the freshness state of every **Artifact**

## Example dialogue

> **Dev:** "When I run `email_draft`, does it re-execute `summary`?"
> **Domain expert:** "Only if `summary` is **stale** — if its **artifact** changed, or if the `email_draft` prompt body was edited. If everything's **fresh**, it skips."
>
> **Dev:** "What if the Notion doc changed since last run?"
> **Domain expert:** "That depends on whether the Notion doc is a **source** or a **ref**. If it's a source, it's always live — the **executor** fetches it fresh every time anyway. If it's a ref to another context's artifact, freshness tracking catches it."

## Flagged ambiguities

- "Model" is ambiguous — it means LLM model in this system. Use **context** for the prompt template concept. Resolved.
- "Cache" implies a transparent performance layer. Use **freshness** for the validity concept. Resolved.
- "Assemble" / "hydrate" / "compile" all mean the same thing. Canonical term: **bind**. Resolved.
- "Put" / "record" / "publish" for recording artifacts. Canonical term: **run** (with `--input`). Resolved.
- "Source" could mean the data itself or the spec. In abcd, a source is the **spec** — the data is the executor's concern. Resolved.
