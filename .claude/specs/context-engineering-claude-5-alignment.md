# Spec: align the auditor with "The new rules of context engineering for Claude 5 generation models"

Source: https://claude.com/blog/the-new-rules-of-context-engineering-for-claude-5-generation-models
(Thariq Shihipar, 24 Jul 2026). Rules extracted:

1. *Then: give Claude rules → Now: let Claude use judgement.* Over-constraining costs
   quality; conflicting directives across system prompt / CLAUDE.md / skills force the
   model to arbitrate before working.
2. *Then: give Claude examples → Now: design interfaces.* Examples now constrain the
   exploration space.
3. *Then: put it all upfront → Now: progressive disclosure.* (already covered by
   `OVER-500-LINES` / `NO-PROGRESSIVE-DISCLOSURE` / `CHAINED-REF`)
4. *Then: repeat yourself → Now: say it once, in the right place.*
5. *Then: memory in CLAUDE.md → Now: auto-memory.*
6. *CLAUDE.md: lightweight; spend the tokens on gotchas; avoid stating the obvious
   things Claude can see from the file system.*

## Objective + success criteria

- The auditor no longer enforces the retired "every SKILL.md needs Examples /
  Troubleshooting" rule → `NO-EXAMPLES` / `NO-TROUBLESHOOTING` gone from the command,
  the tag set, the report map, and the verification table.
- Four new deterministic tags fire on planted fixtures and stay silent on the clean
  fixture: `OVER-CONSTRAINED`, `INSTRUCTION-DUPLICATED`, `CLAUDEMD-OBVIOUS`,
  `CLAUDEMD-MEMORY-DRIFT`.
- One new judgment tag `RULE-CONFLICT` with an evidence recipe (quote both sides) and
  an `llm-rubric` eval.
- `bash tests/run.sh` green (existing 214 scanner + 13 history assertions + the new cases).
- This repo's own command file loses its duplicated instructions (dogfooding rule 4).

## Files to create / change

- `commands/scripts/validate-skills.sh` — 4 new checks
- `commands/claude-markdown-health-check.md` — Phase 5/8/12 text, new Phase 27, depth
  table, tag set, de-duplication of its own repeated instructions
- `commands/claude-markdown-health-check/references/report-format.md` — tag→domain map
- `commands/claude-markdown-health-check/references/finding-verification.md` — fast-path
  list, `RULE-CONFLICT` row, removal of the two retired rows
- `commands/claude-markdown-health-check/references/claude-md-quality.md` — new
  deterministic rows, "avoid the obvious" red flag, re-weighted score table
- `tests/fixtures/{over-constrained,instruction-duplicated,claudemd-obvious,claudemd-memory-drift,rule-conflict}/`
- `commands/claude-markdown-health-check/evals/{86..90}-*.json`
- `README.md`, `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json`

## Open questions / risks

- False positives on `OVER-CONSTRAINED`: calibrated against real trees before choosing
  thresholds (see step 1).
- `RULE-CONFLICT` cannot be made deterministic without antonym guessing → stays a
  judgment finding behind the evidence-grounding gate.
- Removing `NO-EXAMPLES` is a contract change for anyone grepping the tag; it is not
  referenced by any eval, so no test churn.

## Steps

1. Calibrate the over-constraint threshold.
   - key decision: what density is "over-constrained" without flagging normal skills.
   - default: ALL-CAPS absolutes only (`NEVER|ALWAYS|MUST|MUST NOT|DO NOT|DON'T|
     MANDATORY|STRICTLY FORBIDDEN|ABSOLUTE`), counted outside frontmatter and fenced
     code; fire at ≥ 12 per 100 body lines with ≥ 8 hits and ≥ 40 body lines.
     Measured: real skills 0–7.3/100; an "ABSOLUTE RULE"-style CLAUDE.md 18.2/100.
   - verify: measurement table over `~/.claude/skills/*/SKILL.md` + CLAUDE.md files.
   - affected files: none (analysis)
2. Implement the four deterministic checks in `validate-skills.sh`.
   - key decision: precision over recall — every check needs a disproof-resistant signal.
   - default: `OVER-CONSTRAINED` = density above; `INSTRUCTION-DUPLICATED` = a ≥ 40-char
     normalized directive line appearing verbatim in ≥ 2 files; `CLAUDEMD-OBVIOUS` = a
     run of ≥ 6 bare-path lines of which ≥ 3 resolve on disk; `CLAUDEMD-MEMORY-DRIFT` =
     ≥ 3 memory-shaped bullets.
   - verify: `bash tests/run.sh` new cases pass, `01-clean` still clean.
   - affected files: `commands/scripts/validate-skills.sh`
3. Fixtures + evals 86–90.
   - key decision: one planted defect per fixture, plus false-positive guards.
   - default: code-graded for the 4 deterministic tags, `llm-rubric` for `RULE-CONFLICT`.
   - verify: `bash tests/run.sh`, `validate-evals.sh` schema gate.
   - affected files: `tests/fixtures/**`, `commands/claude-markdown-health-check/evals/*`
4. Retire the examples mandate + wire the new tags into the command and references.
   - key decision: which domain owns the cross-cutting tags.
   - default: `Context` domain for `OVER-CONSTRAINED` / `INSTRUCTION-DUPLICATED` /
     `RULE-CONFLICT`; `CLAUDE.md` domain for the two CLAUDEMD tags.
   - verify: grep shows no `NO-EXAMPLES` / `NO-TROUBLESHOOTING` outside git history; every
     new tag appears in the tag set, the domain map, and the verification doc.
   - affected files: command file + 3 reference docs
5. Dogfood: remove this repo's own repeated instructions.
   - key decision: what is repetition vs a deliberate safety restatement.
   - default: collapse duplicated render/resolution instructions; keep every autonomy,
     no-write, and privacy rule verbatim.
   - verify: `validate-skills.sh` on this repo reports no `INSTRUCTION-DUPLICATED` for the
     collapsed lines; the autonomy-gate eval text is untouched.
   - affected files: `commands/claude-markdown-health-check.md`
6. Docs + version.
   - key decision: minor or patch bump.
   - default: minor (0.11.0 → 0.12.0) — new tags are a contract addition.
   - verify: README "What it checks" and phase table mention the new checks; both plugin
     manifests carry the same version.
   - affected files: `README.md`, `.claude-plugin/*.json`

## Final verification gate

`bash tests/run.sh` green, and `bash commands/scripts/validate-skills.sh <this repo>`
runs clean of the new tags after step 5.
