---
name: ext-messaging
description: >-
  Detect messaging security issues in a browser extension: missing sender/origin
  validation in onMessage/onMessageExternal handlers, content↔background
  privilege escalation chains, postMessage without origin check, native messaging
  with unsanitized input. Reads scan/index.md plus targeted path:line reads.
  Outputs candidates to scan/messaging-results.md. Step 2b of the "run
  vulnerability scan" pipeline.
---

# ext-messaging: Messaging Security Analysis

**Prerequisite**: `scan/index.md` must exist. Run `ext-recon` first.

Read `scan/index.md`. Use the **Message Handlers**, **Sources**, and
**Trust Boundaries** sections as your starting point. Open only the specific
`path:line` entries listed in the index — do not scan files broadly.

---

## Token rules

- Read `scan/index.md` once.
- For each handler in the index: open `path:line` ± 5 lines for initial context.
- If tracing a data flow: one additional targeted read per hop, ≤ 20 lines.
- Candidate = concrete path where attacker controls data crossing a trust boundary.
- If a handler is not in the index, it doesn't exist — do not search for it.
- No candidates → `No messaging candidates.` and stop.

---

## Attack patterns to flag

### 1. onMessage without sender validation → background privilege escalation

`chrome.runtime.onMessage` in a background script/service worker can receive
from content scripts (which run in an untrusted web page context) and from the
extension's own popup/options.

**Flag when** (from index):
- `sender_check: NO` on an `onMessage` handler in a background file
- AND the handler dispatches to a privileged operation:
  `executeScript`, `scripting.executeScript`, `fetch` to an internal/sensitive
  URL, `chrome.storage.set`, native messaging port write, or `tabs.update`

A compromised content script (or a page exploiting the content script) becomes a
relay to perform privileged actions in the background context.

**Confirm**: open `path:line` ± 5 lines. Verify the privileged action exists in
the same dispatch path.

---

### 2. onMessageExternal without strict sender allowlist

`chrome.runtime.onMessageExternal` receives messages from origins listed in
`externally_connectable.matches`. If matches is broad (`*://*/*`) AND the handler
has no `sender.id` or `sender.origin` check, any website can invoke privileged
extension actions.

**Flag when** (from index):
- `onMessageExternal` with `sender_check: NO`
- AND `externally_connectable.matches` includes a wildcard or broad pattern

---

### 3. window.addEventListener('message') without origin check → postMessage injection

Content scripts listening to `postMessage` without verifying `event.origin` can
be manipulated by any page including malicious iframes.

**Flag when** (from index):
- `window.message` handler with `origin_check: NO`
- AND the handler reads `event.data` and passes it to: a DOM sink, a
  `chrome.runtime.sendMessage` call, a storage write, or any other sensitive op

**Classic attack sketch**: attacker page sends
`window.postMessage({action:'exec', code:'...'}, '*')` → content script relays
message to background → background calls `executeScript`.

---

### 4. Two-hop privilege escalation chain

This requires BOTH conditions:
1. Content script's `window.message` handler has `origin_check: NO`
2. Background's `onMessage` handler has `sender_check: NO`
3. The background handler performs a privileged action

Each condition alone is a weaker finding; both together is a full chain from
attacker-controlled web page input to privileged extension behavior.

**Check**: cross-reference Trust Boundaries in index — look for
`web page → content (no origin check)` + `content → background (no sender check)`
appearing in the same extension.

---

### 5. Native messaging with unsanitized input

`chrome.runtime.connectNative` / `port.postMessage` to a native host.
If the message payload is derived from web page input with no sanitization, the
native host's attack surface expands significantly.

**Flag when** (from index):
- A source entry traces to a native messaging sink
- No sanitization step is evident between source and sink

---

## Output format

Write to `scan/messaging-results.md`:

```markdown
# Messaging Analysis Candidates

## [CANDIDATE] <short title>
- **path:line**: <handler location from index>
- **handler**: `chrome.runtime.onMessage` / `window.message` / `onMessageExternal` / etc.
- **boundary**: web page → content / content → background / external → background
- **missing check**: `sender.id` / `event.origin` / none
- **privileged action**: `executeScript` / `fetch` / `storage.set` / native port / etc.
- **attack path**: <one sentence — e.g., "attacker page postMessage → content relays → background executeScript">

[repeat per candidate]
```

If no candidates: `No messaging candidates.`
