# Browser Extension Security Assessment

When the user says **"run vulnerability scan"**, the full pipeline runs
automatically: ext-recon → 4 detection skills → ext-triage → ext-report.
Steps whose output file already exists are skipped.

Workflow definition (steps, files, skip logic): `.sast/workflow.md`

---

## Claude Code execution

**Step 1** — Run `ext-recon` in-session. Wait for completion before Step 2.

**Step 2** — Launch `ext-manifest`, `ext-messaging`, `ext-injection`, `ext-data`
as **parallel subagents**. Subagent instruction:
> Read `scan/index.md`. Run the named skill. Write candidates to the output
> file. Use only targeted `path:line` reads — do not scan the full codebase.

Skip any bucket whose output file already exists. Wait for all subagents before Step 3.

**Step 3** — Run `ext-triage` in-session.

**Step 4** — Run `ext-report` in-session.
