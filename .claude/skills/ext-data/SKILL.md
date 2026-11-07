---
name: ext-data
description: >-
  Detect data security issues in a browser extension: hardcoded secrets and API
  keys in JS/HTML files, sensitive data stored in chrome.storage or
  localStorage, auth token leakage to web pages, and sensitive values written to
  console.log. Reads scan/index.md plus targeted grep/reads on files in the
  File Inventory. Outputs candidates to scan/data-results.md. Step 2d of the
  "run vulnerability scan" pipeline.
---

# ext-data: Data Security Analysis

**Prerequisite**: `scan/index.md` must exist. Run `ext-recon` first.

Read `scan/index.md`. Use the **File Inventory** (for the secrets grep scope),
**Sinks** (storage and console entries), and **Sources** sections.

---

## Token rules

- Read `scan/index.md` once.
- Secrets grep: run `rg` scoped to files listed in File Inventory only —
  not a full filesystem scan.
- For each grep hit or sink entry: open `path:line` ± 3 lines to confirm
  it is a real value (not a placeholder or test fixture).
- Storage/log candidates: use index Sinks, open only flagged `path:line`.
- No candidates → `No data candidates.` and stop.

---

## What to flag

### 1. Hardcoded secrets and API keys

Run targeted grep on files from the File Inventory:

```
rg -n -i "(api[_-]?key|apikey|api[_-]?secret|client[_-]?secret|access[_-]?token|\
auth[_-]?token|bearer|private[_-]?key|password|passwd|secret)\s*[:=]\s*['\"][^'\"]{8,}['\"]" \
<files from File Inventory>
```

Also search for format-specific patterns:
- AWS access key: `AKIA[0-9A-Z]{16}`
- Google API key: `AIza[0-9A-Za-z\-_]{35}`
- Stripe live secret: `sk_live_[0-9a-zA-Z]{24}`
- PEM private key: `-----BEGIN (RSA |EC )?PRIVATE KEY-----`
- GitHub token: `ghp_[0-9a-zA-Z]{36}` or `github_pat_`
- Slack token: `xox[baprs]-[0-9a-zA-Z\-]+`

**Confirm before flagging**: open the specific line. If the value is a
placeholder (`YOUR_API_KEY`, `xxxx`, `<replace>`, `TODO`, empty string), DROP
it immediately. If it matches the expected entropy/format of a real credential,
flag it as a candidate.

---

### 2. Sensitive data in chrome.storage / localStorage

Look at index Sinks for `chrome.storage.set` / `localStorage.setItem` entries.
Open `path:line` ± 5 lines. Flag when the stored value contains:
- Authentication tokens, session IDs, JWT
- Passwords or PINs
- Full PII (name, email, SSN, card number, address)
- Private keys or signing secrets

**Storage risk notes:**
- `chrome.storage.sync` is synced to Google's servers — treat as external
- `chrome.storage.local` is accessible to any external origin listed in
  `externally_connectable.matches` — check the index for that field
- `localStorage` in a popup/options page persists across browser sessions in
  the extension's origin

---

### 3. Auth token / sensitive value leakage to web page

Pattern: extension holds an auth token (from storage or hardcoded), and a
content script sends it to the web page or exposes it in the DOM.

Leak vectors:
- `window.postMessage(token, '*')` — broadcasted to all listeners on the page
- `document.body.dataset.token = token` — readable by page scripts
- `element.setAttribute('data-...', token)` — same
- `chrome.scripting.executeScript({code: 'window.__token = ' + token})`

**Check**: cross-reference index Sources (token read from storage) with Sinks
(postMessage / DOM attribute / executeScript) in content script files.

---

### 4. Sensitive data in console.log

Look at index Sinks for `console.log` entries.
Open `path:line` ± 3 lines.
Flag when the logged argument is an auth token, password, PII, or private key.

In production extensions DevTools is accessible to the user AND to any page
content in the same process — moderate severity but trivially fixed.

---

## Output format

Write to `scan/data-results.md`:

```markdown
# Data Security Candidates

## [CANDIDATE] <short title>
- **path:line**: <location>
- **issue**: hardcoded-secret / sensitive-storage / token-leak / sensitive-log
- **evidence**: `const API_KEY = "AIza..."` (or describe what was found and why it looks real)
- **sensitivity**: Critical (private key / password) / High (API key / token) / Medium (PII) / Low (console log)

[repeat per candidate]
```

If no candidates: `No data candidates.`
