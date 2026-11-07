---
name: ext-manifest
description: >-
  Detect manifest-level security issues in a browser extension: excessive
  permissions, weak CSP (unsafe-eval/unsafe-inline), dangerous
  web_accessible_resources, overly broad externally_connectable, MV2 remnants
  in an MV3 extension. Reads scan/index.md only — does not rescan source files.
  Outputs candidates to scan/manifest-results.md. Step 2a of the "run
  vulnerability scan" pipeline.
---

# ext-manifest: Manifest Security Analysis

**Prerequisite**: `scan/index.md` must exist. Run `ext-recon` first.

Read `scan/index.md`. Analyze the **Manifest Summary** section.
Open source files only when the index entry is ambiguous and you need to confirm
the exact field value — open only the specific `path:line` ± 5 lines.

---

## Token rules

- Read `scan/index.md` once. No full-file scans.
- Source file reads only for confirmation — specific line ± 5 lines max.
- Output only flagged candidates — no commentary on clean fields.
- No candidates → `No manifest candidates.` and stop.

---

## What to flag

### 1. Excessive / dangerous permissions

Flag any permission that provides broad capability beyond the extension's
evident purpose. Use File Inventory role labels from the index to infer purpose.

| Permission | Risk |
|---|---|
| `<all_urls>` / `*://*/*` | broadest host access |
| `tabs` | reads URLs of all open tabs |
| `webRequest` / `webRequestBlocking` | intercepts all network traffic |
| `nativeMessaging` | spawns host process |
| `debugger` | full script execution in any tab |
| `proxy` | reroutes all browser traffic |
| `cookies` (+ host access) | cross-origin session theft |
| `history` | full browsing history |
| `clipboardRead` | clipboard monitoring without user gesture |
| `management` | can disable/enable other extensions |

Flag when permission is present **and** the extension's evident function does
not require it.

### 2. Weak Content Security Policy

**Critical:**
- `unsafe-eval` in `script-src` — allows `eval()`, `new Function()`, etc.
- `unsafe-inline` in `script-src` — allows inline scripts; amplifies any XSS

**High:**
- CSP not set at all
- Wildcard source `*` or `https:` in `script-src`
- Remote script host in `script-src` (e.g., `https://cdn.example.com`) — remote
  code runs with extension privileges

**MV3 note**: MV3 bans `unsafe-eval` by spec. If present, it's a misconfiguration
and will cause the extension to fail on Chrome, but flag it anyway.

### 3. Dangerous web_accessible_resources

Flag when:
- MV3: `matches` includes `"<all_urls>"` or `"*"` — any page can embed/load the
  resource; enables extension fingerprinting and UI-spoofing attacks
- Sensitive scripts or pages are exposed (e.g., `background.js` or option pages
  listed in `web_accessible_resources`)

Absence of `web_accessible_resources` = more secure default; don't flag absence.

### 4. Overly broad externally_connectable

Flag when `matches` includes:
- `*://*/*` — any website can call `chrome.runtime.sendMessage` to the background
- `*://*.example.com/*` — entire domain family (note if domain is not fully
  controlled by the developer)

Absence of `externally_connectable` = safer default (only the extension can
send external messages); do not flag absence.

### 5. MV2 remnants in an MV3 extension

If `manifest_version` is 3, flag:
- `background.persistent: true` — MV2-only field
- `browser_action` / `page_action` instead of `action`
- `background.scripts` array instead of `background.service_worker`
- `webRequestBlocking` in permissions (removed in MV3)
- `content_security_policy` as a plain string instead of an object

---

## Output format

Write to `scan/manifest-results.md`:

```markdown
# Manifest Analysis Candidates

## [CANDIDATE] <short title>
- **path:line**: manifest.json (field: `permissions`)
- **issue**: <one sentence — what it is and why it's risky>
- **evidence**: `"permissions": ["<all_urls>", "debugger"]`

[repeat per candidate]
```

If no candidates: `No manifest candidates.`
