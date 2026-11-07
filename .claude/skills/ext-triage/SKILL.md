---
name: ext-triage
description: >-
  Triage all browser extension scan candidates from scan/*-results.md. For each
  candidate, verify exploitability by opening the exact path:line in source code
  and deciding KEEP or DROP. KEEP = exploitable, trust boundary crossed, concrete
  PoC possible. DROP = false positive, unreachable, informational, or no PoC.
  Outputs scan/confirmed.md (KEEP only) and scan/dropped.md. Step 3 of the
  "run vulnerability scan" pipeline; mandatory gate before ext-report.
---

# ext-triage: Candidate Triage

**Prerequisite**: At least one `scan/*-results.md` file must exist.

Read all existing `scan/*-results.md` files. For each `[CANDIDATE]` entry,
open the exact `path:line` from the index and make a final KEEP / DROP decision.

---

## Token rules

- Read each results file once.
- For each candidate: open `path:line` ± 10 lines max.
- If taint trace requires following one function call: one additional targeted
  read, ≤ 20 lines.
- Do not re-read files already read in this session.
- KEEP count will typically be less than DROP count. If everything is KEEP,
  you are not filtering enough.
- No PoC → DROP, no exceptions.

---

## KEEP criteria — all three must be true

1. **Reachable**: Code path is live. Not dead code, not behind `if (DEBUG)`,
   not test fixture, not commented out.
2. **Attacker-controlled input**: An external source (web page, URL param,
   external message, remote fetch response) demonstrably reaches the sink or
   risky operation. Not the extension itself controlling its own data.
3. **Exploitable consequence**: Crosses a meaningful trust boundary OR exposes
   real user data. Must have a concrete PoC you can write in 2–3 steps.

---

## DROP criteria — any one is sufficient

- Sink exists but source is fully controlled by the extension itself
  (hardcoded constant, server-generated value, sanitized before reaching sink).
- Origin / sender check IS present in the actual code — even if the index
  marked it NO (index can be wrong; the code is authoritative).
- Dead code / disabled feature / `if (process.env.NODE_ENV === 'test')` branch.
- Only a best-practice / hardening concern with no direct exploitability
  (e.g., CSP missing but zero XSS vectors exist in the codebase; `tabs`
  permission present but never used to access sensitive URLs).
- Cannot write a step-by-step PoC — if you can't describe the exact attacker
  action and outcome, it is DROP.
- Informational observation with no security impact on user data or privilege.

**When in doubt: DROP.** "Might be exploitable" / "theoretically possible" /
"worth investigating" are all DROP.

---

## Hard-DROP overrides — these override KEEP regardless of everything else

### postMessage cross-origin rule

A `window.postMessage(data, "*")` call targets **the same window (self)**. A
cross-origin child iframe does NOT receive a `window.postMessage` from the
parent's own window context unless the iframe's `contentWindow` is the explicit
target. Only these call forms can cross a window boundary:

- `window.parent.postMessage(...)`
- `opener.postMessage(...)`
- `iframeRef.contentWindow.postMessage(...)`

**Rule:** DROP any postMessage finding unless the code you read **proves** the
recipient is a different, cross-origin window via one of the forms above. If the
target is `window` itself (same window) AND the data already belongs to the
page's own origin (e.g. the page initiated the operation whose result is being
posted), DROP as false positive.

### Tool-self-impact rule

DROP (treat as informational) if the **only** impact is corruption of the
extension tool's own output, recording, or internal data quality. "Breaks the
extension's own data" is not a reportable finding — there must be security
impact to a third party, another origin, or another user.

### Unverified-assumption rule

If a finding's exploitability depends on an assumption about code that triage
has **not actually read** (e.g. "the background signs requests without user
presence", "the token is likely stored unencrypted"), triage **must** open that
code and confirm the assumption before keeping the finding. Use the allowed
targeted read (≤ 20 lines). If the assumption cannot be confirmed from the
code, DROP — do not keep a finding on assumed behavior.

---

## PoC requirement for every KEEP

Must include at minimum:
1. Who is the attacker (a web page the user visits, an external site, the
   extension itself, etc.)
2. The specific action they take (exact message payload, URL crafted,
   API call made)
3. The concrete result (script execution, data exfiltration, privilege
   escalation, credential exposure)

No PoC = DROP, regardless of how obvious the vulnerability seems.

---

## Severity scale for KEEP entries

| Severity | Criteria |
|---|---|
| Critical | RCE via `executeScript` with external code; `eval` of external data in background |
| High | DOM XSS in content script on broad host permissions; auth token leaked to web page; `eval` in content script; privilege escalation chain (web→content→background) |
| Medium | DOM XSS in popup/options; `onMessage` without sender check but limited privileged action; broad `externally_connectable` with real exploit path |
| Low | `console.log` of sensitive value; informational permission flag with demonstrated (not theoretical) risk |

---

## Output files

### scan/confirmed.md

```markdown
# Confirmed Findings

## [FINDING] <title>
- **bucket**: manifest / messaging / injection / data
- **path:line**: <exact location>
- **severity**: Critical / High / Medium / Low
- **boundary**: web page → content / content → background / external → background / etc.
- **PoC**:
  1. <attacker action>
  2. <what happens internally>
  3. <impact / result>
- **remediation**: <one sentence — specific fix, not generic advice>

[repeat per KEEP]
```

If no KEEP entries: `No confirmed findings.`

---

### scan/dropped.md

One line per dropped candidate:

```markdown
- `<bucket> / <title>` — DROP: <one-phrase reason> (`path:line`)
```

Examples:
```
- `injection / innerHTML in search results` — DROP: variable is a hardcoded template string, no external source (content.js:42)
- `messaging / onMessage no sender check` — DROP: handler only reads tabs count, no privileged action possible (background.js:5)
- `manifest / tabs permission` — DROP: permission present but never used to access URL data (manifest.json)
```
