---
name: ext-recon
description: >-
  Scan a browser extension codebase once and produce scan/index.md — a compact
  structured index of manifest fields, file roles, sources, sinks, and message
  handlers. All detection skills read only this index; nothing re-scans the
  codebase. Step 1 of the "run vulnerability scan" pipeline. Run before any
  detection skill; triggered when scan/index.md does not exist.
---

# Extension Recon

You are performing the **sole codebase scan** for a browser extension security
assessment. Your output — `scan/index.md` — is the only file detection skills
will read. Make it complete and accurate; detection quality depends entirely on
this index.

Create the `scan/` directory if it doesn't exist.

---

## Token rules

- You are the ONLY phase that reads source files broadly.
- Read manifest first, then entry points, then follow imports/references.
- `scan/index.md` must use tables/lists — no prose paragraphs.
- Every source, sink, and handler entry requires a `path:line` reference.
- For minified files, all `path:line` references must point to the beautified
  copy under `scan/beautified/`. Never emit `:1` for a minified file unless
  beautification failed (mark those with `[minified, line# unreliable]`).
- Omit sections that have no entries (e.g., no native messaging → omit that row).

---

## Phase 1: Manifest

Read `manifest.json` at the extension root. Extract:

| Field | What to record |
|---|---|
| `manifest_version` | 2 or 3 |
| `permissions` | full array |
| `host_permissions` | full array (MV3) or `http://`/`https://` entries from `permissions` (MV2) |
| `content_security_policy` | full string or object value |
| `web_accessible_resources` | full array |
| `externally_connectable` | `matches` array |
| `content_scripts` | `matches` + `js` arrays for each entry |
| `background` | `service_worker` path / `scripts` array, `persistent` flag (MV2) |
| `name`, `version` | for report header |

---

## Phase 2: File Inventory

Walk all `.js`, `.html`, `.ts` files. Classify each by role:

| Role | How to identify |
|---|---|
| `content_script` | listed under `content_scripts[*].js` in manifest |
| `background` | `background.service_worker` or `background.scripts` |
| `popup` | `browser_action.default_popup` / `action.default_popup`, or referenced from popup HTML |
| `options` | `options_ui.page` or `options_page` |
| `injected` | target of `executeScript` / `scripting.executeScript` calls anywhere in the codebase |
| `lib` | vendor/utility (note name + version if identifiable) |

---

## Phase 2.5: Minification Detection & Beautification

After classifying files, check each `.js` file for minification or bundling.
A file is minified/bundled if ANY of these is true:

- Single-line file with more than 500 characters on that line.
- Fewer than 10 newlines for a file larger than 5 KB.
- Obvious bundler header (`/******/ (function(modules)`, `!function(e,t)`,
  `define(["require"]`, webpack/rollup/esbuild banners).

**For each minified file:**

1. Run a beautifier to produce a readable copy. Try in order:
   - `npx prettier --write --parser babel <file>` (output in place, then copy)
   - `js-beautify <file>` if installed
   - `python3 -c "import jsbeautifier; ..."` as last resort
   Produce the beautified output at `scan/beautified/<original-relative-path>`,
   preserving the directory structure (e.g. `src/background.js` →
   `scan/beautified/src/background.js`).

2. If no beautifier is available, skip beautification and record the file as
   minified with `:1` warnings in index.md — do NOT fabricate line numbers.

**All SOURCE, SINK, and handler entries in index.md for minified files must
reference the BEAUTIFIED path and its real line number**, never `original.js:1`.
If beautification failed, record the original path with a `[minified]` tag and
`:1` to signal the line number is unreliable.

---

## Phase 3: Sources

Sources are locations where untrusted / attacker-controlled data enters the
extension. Grep each file in inventory for the patterns below and record every
hit as `path:line`.

**Content scripts:**
- `window.addEventListener('message', ...)` — reads `event.data` from web page
- `location.href`, `location.hash`, `location.search` — URL-derived
- `document.referrer`, `document.URL`
- DOM reads that capture page-authored content: `element.innerHTML`,
  `element.textContent`, `element.value` on elements the extension did not write
- `window.name`

**Background / service worker:**
- `chrome.runtime.onMessage.addListener(...)` — from content scripts or popups
- `chrome.runtime.onMessageExternal.addListener(...)` — from external origins
- `chrome.tabs.onUpdated` / URL-derived values
- `fetch()` / `XMLHttpRequest` responses used as data
- Native messaging: `chrome.runtime.connectNative`, `port.onMessage`

**Popup / options:**
- URL params from `location.search` or `location.hash`
- `chrome.storage.get(...)` / `browser.storage.get(...)` reads that flow into DOM

---

## Phase 4: Sinks

Sinks are locations where data could cause harm if attacker-controlled.

**DOM injection sinks:**
- `element.innerHTML = ...`
- `element.outerHTML = ...`
- `element.insertAdjacentHTML(...)`
- `document.write(...)`, `document.writeln(...)`

**Code execution sinks:**
- `eval(...)`, `new Function(...)`, `setTimeout(<string>, ...)`, `setInterval(<string>, ...)`
- `chrome.tabs.executeScript(...)` / `chrome.scripting.executeScript(...)` with dynamic `code` argument
- `importScripts(...)` with a non-literal URL

**Data exfiltration / storage sinks:**
- `fetch(url)` where `url` is a variable
- `XMLHttpRequest.open(method, url)` where `url` is a variable
- `chrome.storage.set(...)` / `localStorage.setItem(...)` with values from messages
- `console.log(...)` with potentially sensitive arguments (tokens, passwords, PII)

Record each as:
```
path:line  <sink_type>  <- <variable/expression>  (<brief origin note>)
```

---

## Phase 5: Message Handlers

For every message handler, record whether sender/origin validation is present.

**How to check for sender validation:**
- `onMessage`: look for `sender.id === chrome.runtime.id` or `sender.origin` check
  within the same handler function.
- `window.message`: look for `event.origin === "https://expected.com"` or allowlist
  check immediately after the event listener fires.

Classify as:
- `YES` — explicit check present
- `NO` — no check found
- `PARTIAL` — check exists but can be bypassed (e.g., `sender.id.includes(...)`)

---

## Output: scan/index.md

Write using exactly this template (omit rows/sections with no data):

```markdown
# Extension Index: [name v.version]

## Manifest Summary

| Field | Value |
|---|---|
| manifest_version | 2 or 3 |
| permissions | [...] |
| host_permissions | [...] |
| CSP | "..." (or "not set") |
| web_accessible_resources | [...] |
| externally_connectable | matches: [...] (or "not set") |
| content_scripts | matches: [...] → js: [...] |
| background | service_worker: bg.js / scripts: [bg.js], persistent: false |

## File Inventory

| Role | Path |
|---|---|
| content_script | content.js |
| background | background.js |
| popup | popup.html, popup.js |
| injected | inject.js (via executeScript in background.js:34) |

## Beautified Files

(Omit section if no minified files were detected.)

| Original path | Beautified path | Note |
|---|---|---|
| dist/background.js | scan/beautified/dist/background.js | webpack bundle, single-line 142 KB |
| dist/content.js | scan/beautified/dist/content.js | rollup bundle |

All source/sink/handler references below use the beautified paths.

## Sources

| path:line | type | note |
|---|---|---|
| content.js:12 | window.message | reads event.data from web page |
| background.js:5 | onMessage | from content scripts, no sender check |

## Sinks

| path:line | sink_type | note |
|---|---|---|
| content.js:42 | innerHTML | <- message.payload (from web page postMessage) |
| background.js:88 | eval | <- tab.url.hash (URL fragment) |
| popup.js:17 | executeScript | dynamic code from storage value |

## Message Handlers

| path:line | handler_type | sender_check | origin_check | notes |
|---|---|---|---|---|
| background.js:5 | onMessage | NO | n/a | routes action to tabs.executeScript |
| content.js:10 | window.message | n/a | NO | reads event.data.action |
| background.js:72 | onMessageExternal | YES | n/a | id allowlist: ["com.example"] |

## Trust Boundaries

- web page → content (postMessage, no origin check in content.js:10)
- content → background (sendMessage, no sender validation in background.js:5)
- background → content (tabs.sendMessage with dynamic code in background.js:34)
```
