# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

**Read `PROJECT-PLAN.md` at the start of every session.** It contains all architectural decisions, confirmed API syntax for both output protocols, the two-HAR correlation algorithm design, and the full build plan. It is the single source of truth.

This repo builds tools that replace VuGen's blocked network recorder for LRE (LoadRunner Enterprise) performance testing. The root cause of VuGen failure is WinPcap/NPCAP kernel driver installation blocked by enterprise AV/org policy. The solution is browser-based HAR recording with zero installation.

## No Build Step

There is no package manager, no compilation, no test suite. `VuGen-Recorder.html` is a self-contained single-file tool. Open it directly in Chrome/Edge/Firefox — that is the development and production workflow.

To test changes: open `VuGen-Recorder.html` in a browser and drop in `C:\Users\karrir\Downloads\perfmatrix.com.har` (409 entries, 3 transactions with bookmarklet markers).

## Architecture of VuGen-Recorder.html

The entire application is ~1200 lines of HTML/CSS/JS in one file with no external dependencies. All state lives in a single object `S`:

```
S = {
  entries[],      // parsed HAR entries (normalized)
  txns[],         // detected transactions [{name, color}]
  colorMap{},     // txnName → color object
  selMode,        // boolean: manual select mode active
  selIds,         // Set of selected entry IDs
  tab,            // active preview tab: 'ac'|'mj'|'vi'|'ve'|'gh'
  scripts{},      // generated script strings keyed by tab name
  format,         // 'webhttp' | 'devweb' | 'both'
  pendingHar,     // parsed HAR JSON waiting for format modal
  domainFilter{}, // hostname → true/false
  domainStats{}   // hostname → {count, size}
}
```

**Processing pipeline** (triggered on HAR load):
```
readHAR(file)
  → openFmtModal()          user picks format
  → processHAR(har)
      → detectMarkers()     finds START-TxnName.invalid / END-TxnName.invalid URLs
      → buildDomainStats()  counts requests + bytes per hostname
      → renderDomainPanel() left sidebar with checkboxes
      → applyFilters()      sets entry.filtered based on checkboxes + domainFilter
      → buildScripts()      calls genActionC() and/or genMainJS()
      → renderTable()       the request rows with transaction colour bands
      → showScript()        shows active tab in the code preview
```

**Script generators:**
- `genActionC()` → `Action.c` (Web HTTP/HTML, C language): `web_url()` for GET, `web_custom_request()` for POST/PUT/DELETE, wrapped in `lr_start/end_transaction()`
- `genMainJS()` → `main.js` (DevWeb, JavaScript): `new load.WebRequest({...}).sendSync()` wrapped in `new load.Transaction()` blocks
- `genVuserInit()`, `genVuserEnd()`, `genGlobalsH()` → stub files

**Marker detection regex** (both directions supported):
```
//START[-_]TxnName.invalid  OR  //TxnName[-_]START.invalid
//END[-_]TxnName.invalid    OR  //TxnName[-_]END.invalid
```
Hyphens in transaction names are auto-converted to underscores (C/JS identifier constraint).

**Domain filter bug pattern to avoid:** Never wrap `<input type=checkbox>` inside `<label>` — browser double-fires the toggle. Use `<div onclick="onDpClick(event, domain)">` with `onclick="event.stopPropagation(); toggleDomain(domain, this.checked)"` on the checkbox itself.

**Panel resizing:** Left resizer grows `domain-pane` (previousElementSibling), right resizer grows `code-pane` (nextElementSibling, inverted delta). After first drag, set `pane.style.flex = 'none'` and control via `pane.style.width`.

## Output Formats

**Web HTTP/HTML** — generates 5 files: `Action.c`, `vuser_init.c`, `vuser_end.c`, `globals.h` (current: individual downloads; planned: complete project ZIP)

**DevWeb** — generates 1 file: `main.js` (planned: complete project ZIP with `rts.yml`, `scenario.yml`, `parameters.yml`, `tsconfig.json`, `DevWebSdk.d.ts`)

The complete VuGen project ZIP file structures are documented in `PROJECT-PLAN.md` § 3.

## Protocol API — Critical Rules

**Web HTTP/HTML (C):**
- `web_url()` for GET/HEAD; `web_custom_request()` for all other methods
- Correlation functions (`web_reg_save_param_json`, `web_reg_save_param_ex`, `web_reg_save_param_cookie`) must be placed **BEFORE** the request that returns the value
- Long C string bodies must be split at ~1000 chars (compiler limit) — use adjacent string literal concatenation (no comma)
- Correlation values used as `{ParamName}` tokens in subsequent requests

**DevWeb (JavaScript):**
- `load` is a **global object** — no import statement
- Single entry file: `main.js` only (no separate init/finalize files)
- `returnBody: true` is required to read `response.body` or `response.jsonBody`
- Exit types are **lowercase**: `load.ExitType.stop`, `load.ExitType.iteration`
- Think time: `load.sleep(3)` or `load.thinkTime(3)` (both work)
- Stop iteration gracefully: `return false` from the action function
- `TextCheckExtractor` with `failOn: false` returns a boolean; without `failOn` it throws if text not found
- Global defaults apply to all subsequent requests: `load.WebRequest.defaults.returnBody`, `.headers`, `.handleHTTPError`, `.extractors`

Full syntax reference: `C:\Users\karrir\.claude\projects\c--Workspace-Vugen-Recorder\memory\protocols.md`

## Reference Directories

| Path | Purpose |
|---|---|
| `WebHttpHtml3/` | Empty VuGen Web HTTP/HTML project — reference for exact file formats |
| `WebHttpHtml3_After_recording/` | Same project after recording + parameterization |
| `examples/` | 45 vendor-provided DevWeb example scripts — use as syntax reference |
| `examples/EmptyScript/DevWebSdk.d.ts` | TypeScript SDK definitions to embed in generated DevWeb projects |

## Planned Next Tools

**Phase 1** — Upgrade `VuGen-Recorder.html` to download complete VuGen project ZIPs instead of loose files (uses JSZip, already available as a browser library).

**Phase 2** — New `VuGen-Script-Studio.html`: accepts 1–2 HAR files, runs two-HAR diff correlation algorithm, generates complete projects with correlations + parameterization + response validation + exception handling. Algorithm design is in `PROJECT-PLAN.md` § 7.

## User Workflow

1. User opens `VuGen-Recorder.html`, drags bookmarklets to Bookmarks Bar (one-time setup)
2. User presses F12, records HAR using bookmarklet markers for transaction boundaries
3. User drops `.har` into the tool → selects format → reviews request table → downloads scripts
4. Scripts are copied into VuGen/LRE project folder and replayed for load testing

## GitHub

Repo: `git@github.com:ScriptSavant1/vugenCustomRecorder.git` (branch: `main`)
