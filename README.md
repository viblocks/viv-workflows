# viv-workflows

> ⚠️ **Internal component of [viv-typed-agents](https://github.com/viblocks/viv-typed-agents).**
>
> The recommended install path is the typed-agents product, not this repo standalone:
> ```bash
> git clone https://github.com/viblocks/viv-typed-agents
> ./viv-typed-agents/scripts/install.sh /path/to/your-project --tier 3
> ```
>
> This repo is public for transparency and as a surgical-use escape hatch (`cp -r` a single rule file). See [ADR-RD-010](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-010-product-composition.md) for product composition rationale.

Declarative workflow gate rules for the typed-agents strategy.

Per [ADR-RD-005](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-005-workflow-gates-as-data.md), workflow rules are **data**, not code. Hooks in `viv-hooks` are rule consumers; this repo is the rule producer.

Per [ADR-RD-008](https://github.com/viblocks/viv-typed-agents/blob/main/architecture/decisions/ADR-RD-008-pure-descriptors.md), this repo ships only `.json` and `.md`. No executable code.

## Contents

```
viv-workflows/
├── README.md
├── schemas/
│   ├── post-implementation-chain.schema.json
│   ├── evidence-schema.schema.json
│   ├── fix-intent-pattern.schema.json
│   ├── audit-trail-pattern.schema.json
│   └── implementer-reviewer-pairings.schema.json
├── rules/
│   ├── post-implementation-chain.template.json
│   ├── evidence-schema.template.json
│   ├── fix-intent-pattern.template.json
│   ├── audit-trail-pattern.template.json
│   └── implementer-reviewer-pairings.template.json
├── examples/
│   ├── viblocks-style/         ← concrete rules mirroring viblocks-ai
│   └── minimal/                ← smallest viable rule set
├── architecture/
│   └── decisions/
│       ├── ADR-001-gate-vs-hook-boundary.md
│       ├── ADR-002-i18n-fix-intent.md
│       └── ADR-003-pairings-derived-from-routing.md
└── migration/
    ├── from-viblocks.md
    └── preservation-audit.md
```

## The five rules

| Rule file | Trigger | What it gates |
|---|---|---|
| `post-implementation-chain.json` | After typed implementer completes | Defines ordered stages (verify → review → security → commit) |
| `evidence-schema.json` | `gh issue close` | Required evidence fields in close comment |
| `fix-intent-pattern.json` | `Agent` dispatch with `*-implementer` | Detects fix/bug intent; requires `Root cause:` token |
| `audit-trail-pattern.json` | `git commit` on Class A files | Required commit trailer (`Audit-Trail: <id>`) |
| `implementer-reviewer-pairings.json` | After typed implementer completes | Which reviewer follows which implementer (derived from routing) |

## Quick start (consumer)

1. **Vendor**: `cp -r viv-workflows/ my-project/.claude/workflows/`
2. **Pick**: copy templates from `rules/` (or one of the `examples/`) into `my-project/.claude/workflows/`
3. **Customize**: fill placeholders for project-specific values (issue ID format, language, etc.)
4. **Validate**: each rule has a JSON Schema in `schemas/`

## Companion repos

- [viv-routing](https://github.com/viblocks/viv-routing) — the `enforced` field there determines Class A scope used by `audit-trail-pattern`
- [viv-agents](https://github.com/viblocks/viv-agents) — agents referenced in pairings + chain stages
- [viv-typed-agents](https://github.com/viblocks/viv-typed-agents) — strategy spec + cross-component ADRs
- viv-hooks (planned) — consumer that reads these rules and enforces them

## Status

Initial extraction (2026-05-08). Not yet vendored back to viblocks-ai. Pending: viv-orchestration-rules.
