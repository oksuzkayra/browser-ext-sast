# browser-ext-sast

Browser extension (MV2/MV3) security assessment skills for Claude Code and
other coding assistants (opencode, Cursor, Codex, etc.).

> **Disclaimer**: This project is intended for authorized security testing
> and research only. Only scan browser extensions you own or have explicit
> permission to test. The authors accept no liability for misuse.

## Design

Single recon pass → compressed index → 4 parallel detection skills → triage gate → report.

Detection skills never re-scan the codebase. They read `scan/index.md` and do
targeted `path:line` reads only. This eliminates repeated full-file reads that
multi-skill SAST setups typically produce.

## Workflow

```
ext-recon   →  scan/index.md
                    ↓ (parallel)
ext-manifest →  scan/manifest-results.md
ext-messaging → scan/messaging-results.md
ext-injection → scan/injection-results.md
ext-data     →  scan/data-results.md
                    ↓
ext-triage   →  scan/confirmed.md + scan/dropped.md
                    ↓
ext-report   →  scan/final-report.md
```

Triggered by: **"run vulnerability scan"**

## Skills

| Skill | Reads | Writes | Purpose |
|---|---|---|---|
| ext-recon | codebase (once) | scan/index.md | manifest, file inventory, sources, sinks, message handlers |
| ext-manifest | scan/index.md | scan/manifest-results.md | permissions, CSP, web_accessible_resources, externally_connectable, MV2 remnants |
| ext-messaging | scan/index.md + path:line | scan/messaging-results.md | sender/origin validation, privilege escalation chains |
| ext-injection | scan/index.md + path:line | scan/injection-results.md | DOM XSS, eval, dynamic executeScript, remote code load |
| ext-data | scan/index.md + grep + path:line | scan/data-results.md | hardcoded secrets, sensitive storage, token leakage |
| ext-triage | scan/*-results.md + path:line | scan/confirmed.md, scan/dropped.md | KEEP/DROP gate — eliminates FPs and informational |
| ext-report | scan/confirmed.md | scan/final-report.md | severity-sorted deliverable report |

## Installation and usage

Copy the entire `browser-ext-sast/` tree (excluding `.git/`) into the root
of the browser extension you want to assess. Then say **"run vulnerability scan"**.

Which files are read depends on your assistant:

| File | Read by | Purpose |
|---|---|---|
| `CLAUDE.md` | Claude Code | Triggers skill invocation; launches Step 2 as parallel subagents |
| `AGENTS.md` | opencode, Cursor, Codex, Windsurf, etc. | Triggers sequential (or parallel) execution by reading SKILL.md files directly |
| `.sast/workflow.md` | both (via CLAUDE.md / AGENTS.md) | Canonical step table, file names, skip logic — single source of truth |
| `.claude/skills/*/SKILL.md` | both | Detailed instructions for each skill step |

Both `CLAUDE.md` and `AGENTS.md` reference `.sast/workflow.md` and the SKILL.md
files — no content is duplicated between them.

## Re-running after a fix

Delete the output file(s) for the step(s) you want to re-run, plus all
downstream files. Downstream order and skip logic: `.sast/workflow.md`.

## Directory structure

```
browser-ext-sast/
├── CLAUDE.md                    Claude Code orchestration (thin wrapper)
├── AGENTS.md                    Other assistants orchestration (thin wrapper)
├── README.md
├── .sast/
│   └── workflow.md              Canonical workflow — single source of truth
└── .claude/
    └── skills/
        ├── ext-recon/SKILL.md
        ├── ext-manifest/SKILL.md
        ├── ext-messaging/SKILL.md
        ├── ext-injection/SKILL.md
        ├── ext-data/SKILL.md
        ├── ext-triage/SKILL.md
        └── ext-report/SKILL.md
```

---

Inspired by [utkusen/sast-skills](https://github.com/utkusen/sast-skills).
This project reimplements the idea for browser extensions with a
token-optimized, single-index architecture.
