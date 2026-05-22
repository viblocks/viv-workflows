# ADR-004 — Publish TTL batch-sizing policy as declarative data + pre-empt fan-out drift in schemas

**Status:** Accepted
**Date:** 2026-05-22
**Category:** viv-workflows local

## Context

[`viv-orchestration-rules` ADR-005](https://github.com/viblocks/viv-orchestration-rules/blob/main/architecture/decisions/ADR-005-ttl-safety-batch-sizing.md) defines a structural policy: workers have a 600s TTL, and orchestrators MUST cap per-dispatch item count at `floor(ceiling × safety_margin)`. The prose policy lives in that repo.

Three downstream consumers need the policy values (`ttl_seconds`, `ceiling`, `safety_margin`, the derived cap):

1. The orchestrator's pre-dispatch splitter ([`viv-aidlc-orchestrator#12`](https://github.com/viblocks/viv-aidlc-orchestrator/issues/12)).
2. Hooks in `viv-hooks` that may validate dispatches against the cap.
3. Telemetry consumers that need to know whether a dispatch was inside cap when truncation was observed.

If each consumer hard-codes the numbers, calibration updates (e.g. raising the ceiling once telemetry shows headroom) require coordinated edits across repos. That's brittle.

Separately, the [2026-05-22 audit](../audits/2026-05-22-ttl-batch-sizing-fanout.md) confirms that **no current rule in `viv-workflows` prescribes fan-out**. The five existing rules operate per-dispatch, not per-item. But schemas evolve — a future rule could introduce a "for-each-item" stage, and without a schema-layer pre-emption, such a rule could silently violate the cap.

## Decision

**Two additions, both pure data + schema (no code).**

### 1. Publish the policy as a declarative rule

Add a sixth rule: `ttl-batch-sizing-policy`.

- Schema: [`schemas/ttl-batch-sizing-policy.schema.json`](../../schemas/ttl-batch-sizing-policy.schema.json)
- Template: [`rules/ttl-batch-sizing-policy.template.json`](../../rules/ttl-batch-sizing-policy.template.json)
- Examples: [`examples/minimal/ttl-batch-sizing-policy.json`](../../examples/minimal/ttl-batch-sizing-policy.json), [`examples/viblocks-style/ttl-batch-sizing-policy.json`](../../examples/viblocks-style/ttl-batch-sizing-policy.json)

Shape captures: runtime TTL, empirical ceiling with calibration metadata, safety margin, cap formula (single allowed value to prevent formula drift), failure policies (`abort`/`skip`/`retry` per [`viv-typed-agents` ADR-RD-013](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-013-per-item-atomicity.md)), evidence-field names, telemetry hints.

Consumers read this rule by `id` (e.g. `ttl-batch-sizing-policy/default`) and compute `cap = floor(ceiling.empirical_value × safety_margin)`. Calibration updates touch one file.

### 2. Pre-empt fan-out drift in `post-implementation-chain` schema

Extend [`post-implementation-chain.schema.json`](../../schemas/post-implementation-chain.schema.json) with an optional top-level `dispatch_constraints` object:

```json
{
  "dispatch_constraints": {
    "max_batch_size_ref": "ttl-batch-sizing-policy/default",
    "max_batch_size_override": 10,
    "override_justification": "Brief; explains why a smaller cap applies to this chain."
  }
}
```

The field is **optional** — no existing template needs to set it (none currently prescribes fan-out). Its presence in the schema makes the contract explicit: future fan-out rules MUST reference a policy by id, and any numeric override MUST be accompanied by justification prose.

## Rationale

| Concern | How this satisfies |
|---|---|
| Single source of truth | Calibration values live in one file; consumers reference by id. |
| Declarative purity | Both additions are JSON Schema + JSON data. No bash, no code. Honors [ADR-001](ADR-001-gate-vs-hook-boundary.md) and [ADR-RD-008](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-008-pure-descriptors.md). |
| Pre-emption against drift | Schema-layer constraint means a future fan-out rule cannot land without explicitly referencing the cap. |
| Audit hygiene | A re-run of the [2026-05-22 audit](../audits/2026-05-22-ttl-batch-sizing-fanout.md) is now mechanically gated by schema evolution. |
| Mirroring policy parent | The prose source remains [`viv-orchestration-rules ADR-005`](https://github.com/viblocks/viv-orchestration-rules/blob/main/architecture/decisions/ADR-005-ttl-safety-batch-sizing.md); this repo publishes the data form. |

## Consequences

### What changes

- A sixth rule joins the five in `README.md`.
- The new rule has no consumer hook in `viv-hooks` yet — that lands as part of [`viv-aidlc-orchestrator#12`](https://github.com/viblocks/viv-aidlc-orchestrator/issues/12) and a corresponding hook PR (out of scope for this repo).
- The `post-implementation-chain` schema gains an optional field. No existing template needs to set it (additive, non-breaking).

### What does NOT change

- No existing rule's behavior changes.
- No new code anywhere in this repo. Audit trail confirms purity.
- Consumers that don't care about TTL constraints don't need to read the new rule (ISP preserved).

### Trade-offs

- **Some duplication** between this rule's prose (`note` fields, `calibration_basis`) and `viv-orchestration-rules`' prose ADR. Accepted: prose redundancy keeps the data file self-explanatory; the binding to the ADR is by URL link.
- **`cap_formula` as a constrained enum** (single allowed value) instead of a free string is intentional. Allowing arbitrary formula strings would smuggle logic into rule data, defeating [ADR-001](ADR-001-gate-vs-hook-boundary.md). Future formula changes go through ADR amendment.

## Alternatives considered

- **"Just link to viv-orchestration-rules; don't publish data here."** Rejected — consumers (hooks, orchestrator) would need to fetch a markdown file and parse prose for numeric values. That's the kind of brittleness the data form eliminates.
- **"Put the policy in viv-orchestration-rules as JSON instead of here."** Rejected — that repo is for behavioral playbooks and prose rules; gate rules as data live here (ADR-RD-005). Co-locating the JSON next to the prose ADR makes calibration updates require two coordinated edits, the opposite of the goal.
- **"Mandate `dispatch_constraints` on all rules, not just `post-implementation-chain`."** Rejected — premature; the audit confirms only chain-like rules could plausibly grow fan-out. Other rule shapes (evidence, fix-intent, audit-trail, pairings) operate per-event by definition. We can extend later if drift is observed in those shapes.
- **"Inline the cap as a plain integer in `post-implementation-chain` instead of a reference."** Rejected — defeats the single-source-of-truth goal; calibration changes would require touching every rule that hard-coded the integer.

## Related

- [`viv-orchestration-rules` ADR-005](https://github.com/viblocks/viv-orchestration-rules/blob/main/architecture/decisions/ADR-005-ttl-safety-batch-sizing.md) — policy parent (prose).
- [`viv-typed-agents` ADR-RD-013](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-013-per-item-atomicity.md) — sibling contract; `failure_policies` here mirrors its `abort`/`skip`/`retry`.
- [`viv-aidlc-orchestrator#12`](https://github.com/viblocks/viv-aidlc-orchestrator/issues/12) — first consumer of the data form (splitter + telemetry).
- [2026-05-22 audit](../audits/2026-05-22-ttl-batch-sizing-fanout.md) — basis for the additions.
- [ADR-001](ADR-001-gate-vs-hook-boundary.md) (local) — gate-vs-hook boundary; this ADR honors it.
