# Migration from viblocks-ai

This document records how viv-workflows was extracted from viblocks-ai's inline-hook rules.

## Source artifacts (viblocks-ai)

Three rules lived embedded in viblocks-ai code:

1. **Evidence gate** — inline in `.claude/settings.json` PreToolUse hook (gh issue close)
2. **Fix-intent gate** — inline in `.claude/settings.json` PreToolUse hook (Agent dispatch)
3. **Audit-trail gate** — coded in `.claude/hooks/pretooluse-bash-commit.sh` (git commit)

Plus two derived from CLAUDE.md prose:

4. **Post-implementation chain** — described in `CLAUDE.md` "Post-Implementation Chain (MANDATORY)" section
5. **Implementer-reviewer pairings** — implicit in `routing-table.json`; CLAUDE.md says "Reviewer derived from path"

## Transformations applied

### 1. Extracted inline rule logic into JSON (per ADR-RD-005)

Each inline bash rule became a JSON file. Hooks (in `viv-hooks`) will be refactored to consume these.

| viblocks-ai source | viv-workflows artifact |
|---|---|
| `settings.json` evidence-gate command | `examples/viblocks-style/evidence-schema.json` |
| `settings.json` fix-intent-gate command | `examples/viblocks-style/fix-intent-pattern.json` |
| `pretooluse-bash-commit.sh` trailer regex | `examples/viblocks-style/audit-trail-pattern.json` |
| `CLAUDE.md` Post-Impl Chain prose | `examples/viblocks-style/post-implementation-chain.json` |
| `routing-table.json` implementer/reviewer fields | `examples/viblocks-style/implementer-reviewer-pairings.json` (with `default_rule: from-routing-table`) |

### 2. Sanitized project-specific tokens

| viblocks-ai value | Template placeholder |
|---|---|
| `VI-[0-9]+` (Linear issue ID) | `<ISSUE_ID_PATTERN>` in audit-trail template |
| `VI-186`, `VI-XXX` examples | `<PROJECT-PREFIX>-123` in template |
| Specific security-sensitive paths (`**/chain/**`, `**/enrichment/**`) | `<SECURITY_SENSITIVE_PATHS>` in chain template |
| Spanish-only error messages | violation_message templates kept English-default; viblocks-style example preserves Spanish |

Concrete viblocks values are preserved in `examples/viblocks-style/` for reference.

### 3. Restructured fix-intent keywords (ADR-002)

viblocks' flat alternation regex split into language-segmented arrays. Both EN and ES keyword sets preserved verbatim.

### 4. Pairings as a derived rule (ADR-003)

Instead of duplicating routing-table content, the pairings file declares `default_rule: "from-routing-table"`. Hooks consult both files to resolve a reviewer for a given implementer.

### 5. Promoted N/A justification rule to a structured validation

viblocks' evidence gate has a special-case check: "if Security review = N/A, must include ' -- justification'". This was inline in the bash. Promoted to a `validations` array entry with `id`, `rule`, `violation_message`.

### 6. Removed viblocks-specific blacklist domain detector

viblocks' settings.json includes a parallel command that warns when `gh issue create/edit/comment` references blacklist-domain keywords (TronPoll, EthereumPoll, etc.). This is a **project-specific advisor** (not a workflow gate) and stays in viblocks-ai per the cross-component preservation policy. Documented in `viv-typed-agents/migration/from-viblocks.md`.

### 7. Editor-mode commit policy made explicit

viblocks' `pretooluse-bash-commit.sh` blocks editor-mode commits (no `-m`/`-F`) for Class A. Promoted this to a schema-level `editor_mode_policy` enum (`block`/`warn`/`allow`) so consumers can configure it.

## Sanitization checklist

- [x] No project-specific issue ID patterns in template (placeholder used)
- [x] No project-specific security paths in template
- [x] Spanish-only messages restricted to viblocks-style example
- [x] Blacklist domain detector NOT extracted (project-specific advisor)
- [x] Schemas validate templates and viblocks-style examples (verified by ajv-cli)
- [x] No executable code; pure JSON + MD

## Re-vendor plan (future)

When viv-workflows is vendored back into viblocks-ai:

1. Copy `examples/viblocks-style/*.json` to `viblocks-ai/.claude/workflows/`
2. Update `.claude/settings.json` — remove inline evidence-gate and fix-intent-gate commands; replace with calls to viv-hooks consumer scripts
3. Update `.claude/hooks/pretooluse-bash-commit.sh` to read `audit-trail-pattern.json` instead of hardcoded regex
4. Update `CLAUDE.md` to reference the chain JSON as source of truth
5. Keep blacklist-domain detector in `.claude/settings.json` (it's project-specific)
6. Run full test suite; verify gate behavior unchanged

This re-vendor is gated on viv-hooks extraction.
