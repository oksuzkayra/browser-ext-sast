# Browser Extension Security Assessment

When the user says **"run vulnerability scan"**, execute the full pipeline:
ext-recon → 4 detection skills → ext-triage → ext-report.
Steps whose output file already exists are skipped.

Workflow definition (steps, file names, skill paths, skip logic): `.sast/workflow.md`

---

## Execution (generic coding assistant)

For each step listed in `.sast/workflow.md`: read the SKILL.md at the path shown
in the "Skill file" column and follow its instructions. Skip any step whose
output file already exists.

Steps 2a–2d are independent — run them in parallel if your environment supports
it; otherwise run sequentially in any order. Do not re-scan the full codebase in
Steps 2–4: each skill reads only `scan/index.md` plus targeted `path:line` reads.
