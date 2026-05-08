# ADR-001 — Gate-vs-hook boundary: rules describe; hooks enforce

**Status:** Accepted
**Date:** 2026-05-08
**Category:** viv-workflows local

## Context

Per [ADR-RD-005](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-005-workflow-gates-as-data.md), workflow rules are data and hooks are consumers. But the boundary needs to be sharp — otherwise rule files can drift toward containing executable logic, defeating the SRP win.

## Decision

A workflow rule file in this repo MAY contain:
- **Literal patterns** (markers, regex strings, glob patterns)
- **Enums** (allowed values, kinds)
- **Conditions** as structured data (path globs, agent-type patterns)
- **Human-readable notes and violation messages**

A workflow rule file MAY NOT contain:
- **Bash, Python, JavaScript, or any language code**
- **Imperative steps** (e.g. "first do X then do Y in shell")
- **References to specific runtime tools** (jq, grep flags, etc.) — those are hook concerns

When a rule needs computation (e.g. "fixtures path matches longest-prefix"), the rule **declares the requirement**; the hook **implements the algorithm**.

## Rationale

| Concern | How this satisfies |
|---|---|
| SRP | Rule = policy (changes when policy changes). Hook = mechanism (changes when implementation changes). |
| Reviewability | Non-engineers (PMs, security) can review JSON diffs. Bash diffs require engineering review. |
| Testability | Hook tests cover algorithm correctness; rule files validated by JSON Schema. |
| Portability | A consumer can swap the bash hooks for Python hooks; rules don't change. |

## Consequences

- The schemas in `schemas/` enforce field-level constraints (enum, regex, additionalProperties:false) but cannot enforce "no code" — that's a review-time invariant.
- The `note` and `violation_message` fields are explicitly free-form prose. Tooling MUST NOT execute these; they are human-facing.
- When a rule needs new vocabulary (e.g. a new condition kind), extend the schema (additive change) rather than smuggling logic into a string field.

## Alternatives considered

- **Allow rule files to include shell snippets:** rejected — collapses the SRP separation; the original viblocks-ai problem.
- **Define a tiny rule DSL:** rejected — premature; structured data + named enums cover all observed cases.

## Related

- ADR-RD-005 (system-wide workflow-as-data principle)
- ADR-RD-008 (pure descriptors; reinforced here at field level)
