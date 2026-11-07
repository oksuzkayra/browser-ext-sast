---
name: ext-injection
description: >-
  Detect code and DOM injection vulnerabilities in a browser extension: DOM XSS
  sinks (innerHTML, insertAdjacentHTML, document.write, outerHTML), eval /
  new Function / setTimeout(string), dynamic executeScript with external data,
  and remote code loading. Reads scan/index.md plus targeted path:line reads.
  Outputs candidates to scan/injection-results.md. Step 2c of the "run
  vulnerability scan" pipeline.
---

# ext-injection: Injection Vulnerability Analysis

**Prerequisite**: `scan/index.md` must exist. Run `ext-recon` first.

Read `scan/index.md`. Use the **Sinks** and **Sources** sections as your
starting point. For each sink entry in the index, open `path:line` ± 5 lines
to verify what variable is being written and where it comes from.

---

## Token rules

- Read `scan/index.md` once.
- For each sink: open the exact `path:line` ± 5 lines max.
- Trace at most 2 hops upstream if origin is unclear (one targeted read per hop,
  ≤ 20 lines each).
- Flag only when attacker-controlled data from the Sources section demonstrably
  reaches a sink.
- Same sink flagged by multiple buckets → note the cross-ref; don't duplicate.
- No candidates → `No injection candidates.` and stop.

---

## Sink patterns to flag

### 1. innerHTML / outerHTML / insertAdjacentHTML with untrusted data

DOM XSS in a content script is elevated-severity: if the extension runs on
`<all_urls>`, the XSS executes on every page the user visits.

**Flag when** (from index Sinks):
- `sink_type` is `innerHTML`, `outerHTML`, or `insertAdjacentHTML`
- AND the origin note traces back to a Source entry: postMessage data,
  URL fragment, page DOM read, `onMessage` data from an external origin

**Elevated risk pattern**: background receives external data → sends to content
script via `tabs.sendMessage` → content sets `element.innerHTML = msg.html`

---

### 2. eval / new Function / setTimeout(string) / setInterval(string)

Code execution sink. Impact depends on context:
- In content script: script execution in page context (content script privileges)
- In background/popup: script execution with extension privileges

`setTimeout("codeString", n)` and `setInterval("codeString", n)` with a string
argument are equivalent to `eval`.

**Flag when** (from index Sinks):
- `sink_type` is `eval`, `Function`, or `setTimeout-string`
- AND origin traces back to any external Source (postMessage, URL, onMessage
  from untrusted origin, remote fetch response)

---

### 3. Dynamic executeScript / scripting.executeScript with external data

`chrome.tabs.executeScript({code: variable})` or
`chrome.scripting.executeScript({func: ..., args: [variable]})` where `code`
or the args come from web page messages, URL parameters, or storage values
written from an external source.

This is **Critical** — it is arbitrary script execution in the target tab's
context with content-script-level (or higher) privileges.

**Flag when** (from index Sinks):
- `sink_type` is `executeScript`
- AND the note indicates the `code` argument is dynamic and traces to an
  external Source

---

### 4. Remote script loading

Patterns that load and execute code from an external URL:
- `importScripts(remoteUrl)` in a service worker where `remoteUrl` is dynamic
- Creating `<script src="...">` in a content script where the `src` is from
  external data
- `fetch(url).then(r => r.text()).then(eval)` — remote code execution pattern

**Flag when** (from index Sinks):
- `sink_type` is `importScripts` or dynamic `<script>` injection
- AND the URL is derived from external data (not a hardcoded extension resource)

---

### 5. document.write / document.writeln

Older pattern but still present in MV2 extensions.
`document.write()` in a content script that includes data from page messages
or URL can overwrite the entire page DOM, enabling XSS.

---

## Severity hints

| Pattern | Context | Severity hint |
|---|---|---|
| executeScript with external code | background | Critical |
| eval with external data | background/popup | Critical |
| eval with external data | content script | High |
| innerHTML from postMessage (no origin check) | content script | High |
| innerHTML from onMessage | popup/options | Medium |
| document.write from URL hash | content script | Medium |

---

## Output format

Write to `scan/injection-results.md`:

```markdown
# Injection Analysis Candidates

## [CANDIDATE] <short title>
- **path:line**: <sink location from index>
- **sink**: `innerHTML` / `eval` / `executeScript` / `importScripts` / etc.
- **source path:line**: <origin location from index> (<source type>)
- **data flow**: <one-line trace — e.g., "event.data.html (postMessage) → msg → element.innerHTML">
- **context**: content script / background / popup / options
- **severity-hint**: Critical / High / Medium

[repeat per candidate]
```

If no candidates: `No injection candidates.`
