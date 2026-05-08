# ADR-002 — Fix-intent keywords are i18n-segmented arrays, not a flat regex

**Status:** Accepted
**Date:** 2026-05-08
**Category:** viv-workflows local

## Context

viblocks-ai's fix-intent gate uses one big alternation regex:

```
fix|bug|fail|broken|crash|...|falla|corregir|arreglar|roto|no corre|no pasa|no funciona|rompi
```

English and Spanish keywords are interleaved. Adding a third language (e.g. Portuguese) requires editing the same regex and risks breaking existing matches with backslash mishandling.

## Decision

Keywords are stored as **language-segmented arrays**:

```json
{
  "intent_keywords": [
    { "language": "en", "keywords": ["fix", "bug", ...] },
    { "language": "es", "keywords": ["falla", "corregir", ...] }
  ]
}
```

Hooks compose the final regex/match-set from these arrays. ANY match across ANY language triggers the rule.

Adding a language = adding a new entry. Editing one language never touches another.

## Rationale

| Concern | How this satisfies |
|---|---|
| OCP | New language adds a new entry; existing entries untouched |
| Reviewability | Reviewer can see "what's in Spanish" or "what's in English" without parsing alternation |
| Operational simplicity | A change to one language can be reviewed by a speaker of that language only |
| Tooling forward-compat | Hooks can later support per-language case-folding rules without schema change |

## Consequences

- Hooks combine the language arrays at runtime. The hook is responsible for case-folding, anchoring, and word-boundaries — not the rule.
- The `language` field uses ISO 639-1 codes by convention but accepts any descriptive label. Tooling MUST NOT depend on language codes for behavior.
- For projects with a single-language team, only one entry is needed. The minimal example demonstrates this.

## Alternatives considered

- **Flat regex string:** rejected — viblocks' current pattern; brittle to extend
- **One file per language:** rejected — file explosion; harder to validate "fix-intent rule is complete"
- **Map keyed by language:** rejected — harder to extend with per-language metadata (e.g. case-folding flags)

## Related

- ADR-001 (rules describe; hooks enforce — case-folding is hook concern)
