# Preservation audit

What was extracted into viv-workflows vs. what stayed behind in viblocks-ai.

## Extracted (lives in viv-workflows)

| Content | Source | Where it lives now |
|---|---|---|
| Evidence gate fields (4 markers) | `settings.json` PreToolUse | `examples/viblocks-style/evidence-schema.json` `required_fields` |
| N/A justification rule | `settings.json` PreToolUse | `examples/viblocks-style/evidence-schema.json` `validations[0]` |
| Fix-intent EN keywords (13) | `settings.json` PreToolUse | `examples/viblocks-style/fix-intent-pattern.json` `intent_keywords[0]` |
| Fix-intent ES keywords (8) | `settings.json` PreToolUse | `examples/viblocks-style/fix-intent-pattern.json` `intent_keywords[1]` |
| Required tokens (Root cause:/Causa raíz:/Causa raiz:) | `settings.json` PreToolUse | `examples/viblocks-style/fix-intent-pattern.json` `required_tokens` |
| Audit-Trail trailer name + regex | `pretooluse-bash-commit.sh` | `examples/viblocks-style/audit-trail-pattern.json` `required_trailer` |
| Editor-mode block policy for Class A | `pretooluse-bash-commit.sh` | schema `editor_mode_policy` (default `block`) |
| Post-Impl Chain stage order | `CLAUDE.md` prose | `examples/viblocks-style/post-implementation-chain.json` |
| Security review path triggers | `CLAUDE.md` prose | `examples/viblocks-style/post-implementation-chain.json` security-review condition |
| Pairing derivation rule | implicit in routing-table | `examples/viblocks-style/implementer-reviewer-pairings.json` `default_rule: from-routing-table` |

## Stayed behind (viblocks-ai-specific)

| Content | Why it stays |
|---|---|
| Blacklist-domain detector keywords (TronPoll, EthereumPoll, PolygonPoll, DetectionEnrich, Guardian, targetAddress, raw-detections, domain-events, blacklist.pipeline) | Project-specific advisor for viblocks' product domain; not a workflow gate |
| `VI-` issue ID prefix | Linear team identifier; consumer-defined |
| Specific security-sensitive paths (`**/chain/**`, `**/enrichment/**`) | Project-specific; preserved in viblocks-style example for reference |
| `Skill(blacklist-monitoring)` invocation | Project-specific skill; viblocks owns it |
| `AIDLC_ENFORCEMENT_MODE` env var name | viblocks-coined; may be renamed by consumers |

## Knowledge loss check

- [x] All 4 evidence markers preserved (Verification, Code review, Security review, Commits)
- [x] N/A justification special case preserved as structured validation
- [x] All 21 fix-intent keywords preserved (13 EN + 8 ES)
- [x] All 3 required tokens preserved (Root cause:, Causa raíz:, Causa raiz:)
- [x] Audit-Trail regex semantics preserved (VI-XXX OR adhoc-<id>) in viblocks-style example
- [x] Editor-mode commit policy preserved (block for Class A)
- [x] Chain stage order preserved (verify → domain review → security review → commit)
- [x] Security-review conditional trigger preserved (paths-match)
- [x] Pairing derivation preserved (routing-table is source of truth)

## Verification

Reproduce viblocks-ai's behavior using viv-workflows files:

| viblocks-ai gate behavior | viv-workflows equivalent |
|---|---|
| `gh issue close` blocks if `**Verification**` marker missing | `evidence-schema.required_fields[0]` with `marker: "**Verification**"` |
| `gh issue close` blocks if Security review = N/A but no justification | `evidence-schema.validations[0]` with rule on `**Security review**` |
| Agent dispatch with `subagent_type *implementer$` AND prompt matches "fix" AND no "Root cause:" → blocked | `fix-intent-pattern.trigger.agent_type_pattern: "implementer$"` + `intent_keywords` + `required_tokens` |
| `git commit` of Class A files without `-m`/`-F` → blocked | `audit-trail-pattern.editor_mode_policy: "block"` |
| `git commit` of Class A files with `-m` but no `Audit-Trail:` → blocked | `audit-trail-pattern.required_trailer.value_pattern` |
| Domain reviewer dispatched after typed implementer | `post-implementation-chain.stages[1].kind: "domain-review"` + `pairings.default_rule: "from-routing-table"` |
| `security-reviewer` skipped when no security-sensitive paths | `post-implementation-chain.stages[2].condition.kind: "paths-match"` |

All viblocks-ai gate behaviors map to viv-workflows constructs. No behavioral coverage loss.

## Identified during post-extraction review

(Reserved for findings discovered after initial extraction. No findings yet.)
