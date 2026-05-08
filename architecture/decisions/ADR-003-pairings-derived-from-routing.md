# ADR-003 — Implementer-reviewer pairings derive from viv-routing by default

**Status:** Accepted
**Date:** 2026-05-08
**Category:** viv-workflows local

## Context

ADR-RD-005 lists `implementer-reviewer-pairings.json` as one of the five workflow rule files. Initial design considered an explicit map:

```json
{ "backend-implementer": "backend-reviewer",
  "frontend-implementer": "frontend-reviewer", ... }
```

But viv-routing already encodes pairings: each route entry has `implementer` AND `reviewer`. Duplicating this map creates the same drift risk that motivated [ADR-RD-004](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-004-classifier-folded.md) (eliminating artifact-classifier).

## Decision

`implementer-reviewer-pairings.json` declares a **default rule**, not an explicit map.

```json
{
  "default_rule": "from-routing-table",
  "overrides": []
}
```

Semantics:
- `from-routing-table`: hooks resolve the reviewer for an implementer by scanning `routing-table.json` for the route(s) where `route.implementer == X` and using the corresponding `route.reviewer`.
- `explicit-only`: ignore routing-table; use only `overrides`.
- `overrides`: optional per-implementer entries that win over `default_rule` when present.

The file is small (a few lines) but **necessary** to express:
- The directive that pairing IS computed from routing (not hardcoded in hooks)
- The override mechanism for special cases (e.g. an implementer that should skip review)

## Rationale

| Concern | How this satisfies |
|---|---|
| Single source of truth | Routing remains the only place that knows which agent handles which path AND which reviewer pairs with which implementer |
| Drift prevention | No duplicated map to maintain; pairings auto-update when routing changes |
| Override flexibility | `overrides` allows special cases without polluting routing |
| Explicitness | The file's existence makes the derivation rule discoverable; hooks aren't silently consulting routing |

## Consequences

- Hooks consume both `implementer-reviewer-pairings.json` AND `routing-table.json` to resolve a pairing — the workflows file points at routing.
- A consumer that uses `default_rule: "explicit-only"` opts out of routing-derived pairings entirely (e.g. for projects without a routing table).
- The schema permits an empty `overrides: []` — the default-rule alone is sufficient for most consumers.

## Alternatives considered

- **Drop `implementer-reviewer-pairings.json` entirely; pairings are routing-only:** rejected — hides the design intent; consumers wouldn't know hooks consult routing for this purpose. Explicit small file documents the contract.
- **Duplicate the full map here:** rejected — drift risk; same problem as duplicated artifact-classifier.

## Related

- ADR-RD-004 (eliminating duplication; same principle applied)
- viv-routing field-level SRP (ADR-RD-003)
