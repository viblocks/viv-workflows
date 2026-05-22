# Audit — TTL batch sizing & implicit fan-out in existing rules

**Date:** 2026-05-22
**Scope:** All schemas, templates, and examples in `viv-workflows` at HEAD of `main`
**Reason:** Issue [#2](https://github.com/viblocks/viv-workflows/issues/2) — identify rules that prescribe fan-out with N > ceiling × 0.8 (TTL safety).
**Reference policy:** [viv-orchestration-rules `rules/common/ttl-batch-sizing.md`](https://github.com/viblocks/viv-orchestration-rules/blob/main/rules/common/ttl-batch-sizing.md) (ADR-005).

## TL;DR

**No current rule in this repo prescribes implicit fan-out.** The five existing rules (`post-implementation-chain`, `evidence-schema`, `fix-intent-pattern`, `audit-trail-pattern`, `implementer-reviewer-pairings`) all operate at **per-dispatch granularity**, not per-item granularity. None declares a multi-item batch.

The TTL truncation risk lives one layer above (the orchestrator that dispatches `N` items per workflow) and one layer below (the typed agent that iterates over the brief's items). Neither layer is this repo's concern.

This audit adds:
1. A new declarative rule `ttl-batch-sizing-policy` so consumers can read the policy as data from this repo instead of round-tripping to `viv-orchestration-rules`.
2. An optional `dispatch_constraints.max_batch_size_ref` field on `post-implementation-chain.schema.json` for future rules that DO prescribe fan-out (currently none) — the field is a schema-level pre-emption against drift.

## Method

Greps performed against `**/*.json` and `**/*.md` at `viv-workflows` `main`:

```
grep -rinE "batch|fan.?out|\bN[ \.]"
grep -rin "items"
grep -rin "dispatch"
```

Each hit was inspected in context.

## Per-rule findings

### `post-implementation-chain` (schema + template + examples)

| Property | Finding |
|---|---|
| Granularity | Per dispatch. The chain runs once after a typed-implementer dispatch completes (verification → domain-review → security-review → commit). |
| Multi-item declaration | None. No field declares `N` or batch size. |
| Implicit fan-out | None. Each stage runs as a single sub-dispatch. |
| Risk | Zero for this rule shape. If a chain ever adds a "for-each-item" stage, the new schema property `dispatch_constraints.max_batch_size_ref` MUST be set. |

### `evidence-schema` (schema + template + examples)

| Property | Finding |
|---|---|
| Granularity | Per close-comment. Verifies fields in an issue-close evidence block. |
| Multi-item declaration | None. |
| `N` mentions | Two occurrences in `format` strings: `"PASS\|FAIL — N/N tests, build OK\|FAIL"` and `"<reviewer-agent> — PASS \| N issues (severity)"`. These are **prose placeholders** in human-readable format strings, NOT batch counts. No risk. |
| Risk | Zero. |

### `fix-intent-pattern` (schema + template + examples)

| Property | Finding |
|---|---|
| Granularity | Per Agent-dispatch detection. |
| Multi-item declaration | None. Pattern lists are matched against a single prompt string. |
| Risk | Zero. |

### `audit-trail-pattern` (schema + template + examples)

| Property | Finding |
|---|---|
| Granularity | Per `git commit`. |
| Multi-item declaration | None. One trailer per commit. |
| Risk | Zero. |

### `implementer-reviewer-pairings` (schema + template + examples)

| Property | Finding |
|---|---|
| Granularity | Per dispatch resolution. |
| Multi-item declaration | None. Pairings are 1-to-1 (implementer → reviewer). |
| Risk | Zero. |

## Conclusion

The audit confirms **the absence of implicit fan-out in this repo's current rules**. No template, no example, no schema field expresses or implies "do this N times in one dispatch."

The risk surface is upstream (orchestrator: [`viv-aidlc-orchestrator#12`](https://github.com/viblocks/viv-aidlc-orchestrator/issues/12)) and downstream (agent iteration: [`viv-typed-agents#16`](https://github.com/viblocks/viv-typed-agents/issues/16)). This repo's contribution is **defensive**:

1. **Publish the policy as data** — `rules/ttl-batch-sizing-policy.template.json` — so any consumer (hooks, orchestrator scripts) can read the ceiling/margin/cap values from a single declarative source. Mirrors the prose policy in [viv-orchestration-rules ADR-005](https://github.com/viblocks/viv-orchestration-rules/blob/main/architecture/decisions/ADR-005-ttl-safety-batch-sizing.md).
2. **Pre-empt schema drift** — add optional `dispatch_constraints.max_batch_size_ref` to `post-implementation-chain.schema.json`. Future rules that DO want to express fan-out (none exist yet) must reference the policy by id, ensuring the cap is honored declaratively at the schema layer.

Both additions are pure data + schema (no code), consistent with [ADR-RD-008](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-008-pure-descriptors.md) and [ADR-001 here](../decisions/ADR-001-gate-vs-hook-boundary.md).

## Sibling work

| Repo | Issue / PR | Purpose |
|---|---|---|
| viv-orchestration-rules | #3 / #4 | Policy parent (prose rule + ADR-005) |
| viv-typed-agents | #16 / #17 | Agent contract (`commit_item` + `validate_item`) |
| viv-workflows (here) | #2 (this PR) | Audit + declarative policy data + schema pre-emption |
| viv-aidlc-orchestrator | #12 | Pre-dispatch splitter, telemetry, e2e test |

## Re-audit trigger

This audit MUST be re-run when:

- A new rule schema is added that includes any property semantically equivalent to "for each X do Y."
- A template introduces a new `condition.kind` that implies iteration.
- A consumer (hook or orchestrator) reports observing fan-out at the rule layer.
