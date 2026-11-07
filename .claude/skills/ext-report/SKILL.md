---
name: ext-report
description: >-
  Generate the final security report for a browser extension assessment. Reads
  only scan/confirmed.md. Produces scan/final-report.md sorted by severity.
  Only confirmed, exploitable findings are included — the report is directly
  deliverable. Step 4 (final) of the "run vulnerability scan" pipeline.
  Run after ext-triage when scan/confirmed.md exists.
---

# ext-report: Final Report Generation

**Prerequisite**: `scan/confirmed.md` must exist. Run `ext-triage` first.

Read `scan/confirmed.md` only. No other files are needed — all evidence is
already captured in confirmed.md.

---

## Token rules

- Read `scan/confirmed.md` once. No extension source file reads.
- Read each `scan/*-results.md` once (to count candidates for the coverage table)
  — do not re-analyse their content.
- If confirmed.md says `No confirmed findings.`, write that in the report and stop.
- Do not re-derive severity — use the severity recorded in confirmed.md.
- Report is directly deliverable: no FPs, no informational, no hedging language.

---

## Sort order

1. Critical
2. High
3. Medium
4. Low

Tiebreaker within same severity: injection > messaging > manifest > data
(reflects exploitation complexity — injection is most direct).

---

## Output: scan/final-report.md

```markdown
# Browser Extension Security Report

**Extension**: [name and version — from confirmed.md context or infer]
**Date**: [current date]
**Findings**: [N total — N Critical, N High, N Medium, N Low]

---

## Summary

| # | Title | Severity | Bucket | Location |
|---|---|---|---|---|
| 1 | ... | Critical | injection | content.js:42 |
| 2 | ... | High | messaging | background.js:5 |
| 3 | ... | Medium | manifest | manifest.json |

---

## Findings

### [Critical] <title>

- **Location**: `path:line`
- **Bucket**: injection / messaging / manifest / data
- **Trust boundary**: web page → content → background (etc.)
- **Description**: <what the vulnerability is — 1–2 sentences>
- **PoC**:
  1. <attacker action>
  2. <what happens>
  3. <impact>
- **Remediation**: <specific fix — one sentence>

---

[repeat, grouped by severity — omit a severity section entirely if no findings
fall in that tier]

---

## Scan Coverage

| Bucket | Results file | Candidates | Confirmed |
|---|---|---|---|
| ext-manifest | scan/manifest-results.md | — | — |
| ext-messaging | scan/messaging-results.md | — | — |
| ext-injection | scan/injection-results.md | — | — |
| ext-data | scan/data-results.md | — | — |
```

Fill the Candidates and Confirmed columns by counting entries in each
`*-results.md` file and in `confirmed.md`. If a results file doesn't exist,
write `not run` in the Status column.
