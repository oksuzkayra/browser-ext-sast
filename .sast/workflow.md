# Extension Security Assessment: Canonical Workflow

Trigger phrase: "run vulnerability scan"

---

## Steps

| # | Name | Skill file | Input | Output | Skip if exists |
|---|---|---|---|---|---|
| 1 | Recon | `.claude/skills/ext-recon/SKILL.md` | codebase | `scan/index.md`, `scan/beautified/` (minified files only) | `scan/index.md` |
| 2a | Manifest detection | `.claude/skills/ext-manifest/SKILL.md` | `scan/index.md` | `scan/manifest-results.md` | `scan/manifest-results.md` |
| 2b | Messaging detection | `.claude/skills/ext-messaging/SKILL.md` | `scan/index.md` | `scan/messaging-results.md` | `scan/messaging-results.md` |
| 2c | Injection detection | `.claude/skills/ext-injection/SKILL.md` | `scan/index.md` | `scan/injection-results.md` | `scan/injection-results.md` |
| 2d | Data detection | `.claude/skills/ext-data/SKILL.md` | `scan/index.md` | `scan/data-results.md` | `scan/data-results.md` |
| 3 | Triage | `.claude/skills/ext-triage/SKILL.md` | `scan/*-results.md` | `scan/confirmed.md`, `scan/dropped.md` | `scan/confirmed.md` |
| 4 | Report | `.claude/skills/ext-report/SKILL.md` | `scan/confirmed.md` | `scan/final-report.md` | `scan/final-report.md` |

## Ordering rules

- Step 1 must complete before any Step 2 skill starts.
- Steps 2a–2d are independent of each other (run in parallel when supported).
- Step 3 must wait for all Step 2 outputs.
- Step 4 must wait for Step 3.

## Re-running after a fix

Delete the output file(s) for the step you want to re-run. Also delete all
downstream output files to prevent stale results reaching the report.

Downstream order: `index.md` → `*-results.md` → `confirmed.md` / `dropped.md` → `final-report.md`

## Notes on minified extensions

When ext-recon detects minified/bundled JS, it produces beautified copies under
`scan/beautified/` and maps them in index.md. All downstream skills read
`path:line` from the beautified copies — never from the originals. If
`scan/beautified/` is stale (original files changed), delete `scan/index.md`
and re-run from Step 1.
