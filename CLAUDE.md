# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Context

**Read `PROJECT-PLAN.md` at the start of every session.** It contains all architectural decisions, confirmed API syntax for both output protocols, the two-HAR correlation algorithm design, and all completed phases. It is the single source of truth.

This repo builds tools that replace VuGen's blocked network recorder for LRE (LoadRunner Enterprise) performance testing. The root cause of VuGen failure is WinPcap/NPCAP kernel driver installation blocked by enterprise AV/org policy. The solution is browser-based HAR recording with zero installation.

## No Build Step

There is no package manager, no compilation, no test suite. Both HTML files are self-contained single-file tools. Open directly in Chrome/Edge/Firefox — that is the development and production workflow.

**Test files:**
- `VuGen-Recorder.html` → drop `C:\Users\karrir\Downloads\perfmatrix.com.har` (409 entries, 3 transactions with bookmarklet markers)
- `VuGen-Script-Studio.html` → drop `C:\Users\karrir\Downloads\WebTours.har` + `WebTours1.har` (two-HAR mode), or `opensource-demo.orangehrmlive.com.har` + `opensource-demo.orangehrmlive.com1111.har`

## Architecture of VuGen-Recorder.html

~1200 lines of HTML/CSS/JS. All state in `S`:

```
S = {
  entries[],        // parsed HAR/NetLog entries (normalized)
  txns[],           // detected transactions [{name, color}]
  colorMap{},       // txnName → color object
  selMode,          // boolean: manual select mode active
  selIds,           // Set of selected entry IDs
  tab,              // active preview tab: 'ac'|'mj'|'vi'|'ve'|'gh'
  scripts{},        // generated script strings keyed by tab name
  format,           // 'webhttp' | 'devweb' | 'both'
  pendingHar,       // parsed HAR JSON waiting for format modal
  pendingNetLog,    // parsed NetLog JSON waiting for format modal
  isNetLogSource,   // true when source is chrome://net-export/
  domainFilter{},   // hostname → true/false
  domainStats{},    // hostname → {count, size}
  auth,             // {type, realm, host} from detectAuth()
  serverHost        // {host, proto, prefix} from detectServerHost()
}
```

**Processing pipeline:**
```
readFile(file)            // detects HAR vs NetLog
  → openFmtModal()        user picks format
  → processHAR(har) OR processNetLog(netlog)
      → detectMarkers()   finds START-TxnName.invalid / END-TxnName.invalid
      → detectAuth()      Kerberos/NTLM/Negotiate/Digest/Basic/Bearer/SAML
      → detectServerHost() dominant hostname → ServerHost param
      → buildDomainStats()
      → renderDomainPanel()
      → applyFilters()    sets entry.filtered
      → buildScripts()    calls genActionC() and/or genMainJS()
      → renderTable()
      → showScript()
```

**Script generators:**
- `genActionC()` → `Action.c` (Web HTTP/HTML, C): `web_url()` for GET, `web_custom_request()` for POST/PUT/DELETE
- `genMainJS()` → `main.js` (DevWeb, JS): `await new load.WebRequest({...}).send()` in async action functions
- `genVuserInit()`, `genVuserEnd()`, `genGlobalsH()` → stub files
- Downloads: complete ZIP with JSZip (inlined)

**Header generation (both generators):**
- `autoHdrs`: KEY PRESENCE threshold — if ≥80% of entries have the header key, most common value = global auto-header
- Force-global: `user-agent`, `accept-language` always session-wide regardless of count
- Per-request: emit override ONLY when value differs from global (`if(autoHdrs[k]===h.value) continue`)
- `Accept: */*` always suppressed (VuGen/DevWeb default)
- `Authorization` suppressed for non-Bearer auth (`web_set_user` / `load.setUserCredentials` handles it)
- `SKIP_HDR_AC` (Web) and `SKIP_HDRS` (DevWeb) exclude: referer, content-length, host, connection, keep-alive, accept-encoding, expect, priority, sec-*, cache-control, cookie, cookie2, if-* headers

**3xx redirect entries:** All generators emit `// HTTP 3xx redirect — VuGen follows subsequent redirects automatically` before each 3xx request.

**Marker detection regex** (both directions supported):
```
//START[-_]TxnName.invalid  OR  //TxnName[-_]START.invalid
//END[-_]TxnName.invalid    OR  //TxnName[-_]END.invalid
```
Hyphens in transaction names are auto-converted to underscores.

**Domain filter bug pattern to avoid:** Never wrap `<input type=checkbox>` inside `<label>` — browser double-fires the toggle. Use `<div onclick="onDpClick(event, domain)">` with `onclick="event.stopPropagation(); toggleDomain(domain, this.checked)"` on the checkbox itself.

**Panel resizing:** Left resizer grows `domain-pane` (previousElementSibling), right resizer grows `code-pane` (nextElementSibling, inverted delta). After first drag, set `pane.style.flex = 'none'` and control via `pane.style.width`.

## Architecture of VuGen-Script-Studio.html

~3400+ lines of HTML/CSS/JS. State:
```
S = {har1, har2, entries1, entries2, txns, correlations, candidates, params,
     harWarning, scripts, format, mode, tab, auth, serverHost, isNetLog1, isNetLog2}
```

**UI:** 3 phases — Upload → Processing spinner → Results (preview + download)

**Key functions:**
- `analyze()` — orchestrates: detectAuth → detectServerHost → applyFilters → route to single/two-HAR → detectParams → buildScripts → SSO warning
- `twoHarCorrelate(e1, e2)` — diff engine: match pairs, diff all locations, back-trace, generate extractors
- `singleHarCorrelate(entries)` — pattern detection (JWT, CSRF headers, session cookies) + redirect chain detection
- `findValueInResponse(entry, value)` — searches response for a value: Set-Cookie → resp headers (incl. Location for 3xx) → JSON → XML → HTML hidden input → Vue/React prop → data-* → JS var → meta → boundary fallback
- `detectParams(entries, correlations)` — scans POST body fields matching PARAM_KEYS_MAP
- `genActionC(entries, correlations)` → `Action.c`
- `genMainJS(entries, correlations)` → `main.js`

**SSO/OAuth redirect chain (Studio only):**
- `findValueInResponse()` section 1b: tight LB/RB extraction from Location headers of 3xx responses (path segment and query param aware)
- `singleHarCorrelate()` redirect pass at end: 3xx → Location → next URL dynamic segment detection
- `analyze()` SSO warning: `SSO_URL_PATTERN` detects oauth2/oidc/sso/saml/Okta/ADFS/Keycloak in URLs and Location headers

## Output Formats

**Web HTTP/HTML** — ZIP contains: `Action.c`, `vuser_init.c`, `vuser_end.c`, `globals.h`, `ScriptName.usr`, `default.cfg`, `default.usp`, `ParameterFile.prm`, `collection_data.dat`, `ScriptUploadMetadata.xml`, `data/` folder

**DevWeb** — ZIP contains: `main.js`, `rts.yml`, `scenario.yml`, `parameters.yml`, `collection_data.csv`, `tsconfig.json`, `package.json`, `DevWebSdk.d.ts`

Full structure documented in `PROJECT-PLAN.md` § 3.

## Protocol API — Critical Rules

**Web HTTP/HTML (C):**
- `web_url()` for GET/HEAD; `web_custom_request()` for all other methods
- `web_custom_request()` always uses `BodyBinary=` — NEVER `Body=` (JSON braces break VuGen parser)
- Correlation functions must be placed **BEFORE** the request that returns the value
- NEVER use `web_reg_save_param_ex` for body/HTML — use `web_reg_save_param("Name","LB=...","RB=...",LAST)`
- `web_reg_save_param_cookie` does NOT exist — use `web_reg_save_param` with `Search=Headers`
- `web_reg_save_param_json` uses `QueryString=` not `QueryParam=`
- Long C string bodies split at ~200 chars (CHUNK=200) using adjacent string literal concatenation; split points are escape-sequence-aware
- Correlation values used as `{ParamName}` tokens in subsequent requests

**DevWeb (JavaScript):**
- `load` is a **global object** — no import statement
- Single entry file: `main.js` only (no separate init/finalize files)
- Use `await new load.WebRequest({...}).send()` in async action functions (NOT `sendSync()`)
- `returnBody: true` is required to read `response.body` or `response.jsonBody`
- Exit types are **lowercase**: `load.ExitType.stop`, `load.ExitType.iteration`
- Think time: `load.sleep(3)` or `load.thinkTime(3)` (both work)
- Stop iteration gracefully: `return false` from the action function
- Global defaults apply to all subsequent requests: `load.WebRequest.defaults.returnBody`, `.headers`, `.handleHTTPError`, `.extractors`

**Extractor type selection (response-type-driven, both protocols):**
- `cookie` → `CookieExtractor` / `web_reg_save_param(Search=Headers, LB=name=, RB=;)`
- `boundary_header` → `BoundaryExtractor({scope:Headers})` / `web_reg_save_param(Search=Headers)`
- `jsonpath` → `JsonPathExtractor("name","$.path")` / `web_reg_save_param_json(QueryString=$.path)`
- `html` → `HtmlExtractor("name","selector","attr")` / `web_reg_save_param` with selector-derived LB/RB
  - `input[name='x']` → `LB=name="x" value="` `RB="`
  - `meta[name='x']` → `LB=name="x" content="` `RB="`
  - `[data-x]` → `LB=data-x="` `RB="`
- `boundary` → `BoundaryExtractor({scope:Body})` / `web_reg_save_param(LB=...,RB=...)`
- `generate` → `load.utils.uuid()` / `load.utils.randomString(n,{hex:true})` / `lr_param_sprintf`
- `BoundaryExtractor` ALWAYS uses options form with explicit `scope` — NEVER the 3-arg positional form (it scans both headers and body ambiguously)

Full syntax reference: `C:\Users\karrir\.claude\projects\c--Workspace-Vugen-Recorder\memory\protocols.md`

## Reference Directories

| Path | Purpose |
|---|---|
| `WebHttpHtml3/` | Empty VuGen Web HTTP/HTML project — reference for exact file formats |
| `WebHttpHtml3_After_recording/` | Same project after recording + parameterization |
| `examples/` | 45 vendor-provided DevWeb example scripts — use as syntax reference |
| `examples/EmptyScript/DevWebSdk.d.ts` | TypeScript SDK definitions to embed in generated DevWeb projects |

## Both Tools Are Complete

Both `VuGen-Recorder.html` and `VuGen-Script-Studio.html` are fully implemented and deployed to GitHub at `git@github.com:ScriptSavant1/vugenCustomRecorder.git` (branch: `main`).

**Feature summary:**
- HAR and NetLog (`chrome://net-export/`) input
- Authentication detection and code generation (Kerberos, NTLM, Negotiate, Digest, Basic, Bearer, SAML)
- ServerHost parameterization
- Header deduplication (global auto-headers + per-request overrides)
- Two-HAR correlation engine (Studio) with all diff locations
- Single-HAR pattern detection (Studio)
- SSO/OAuth redirect chain correlation (Studio)
- Exception handling (Studio)
- Parameterization — username, password, dates, travel fields, card numbers (Studio)
- Complete ZIP download for both protocols

## User Workflow

**Tool 1 (VuGen-Recorder.html):**
1. Open app in browser, drag bookmarklets to Bookmarks Bar (one-time setup)
2. Navigate to app first (bookmarklets blocked on chrome:// pages)
3. Press F12 → record HAR OR use `chrome://net-export/` for multi-tab recording
4. Drop `.har` or `.json` into tool → pick format → review table → download ZIP

**Tool 2 (VuGen-Script-Studio.html):**
1. Record two HAR files using fresh sessions (Incognito for HAR2 to ensure different tokens)
2. Drop both HARs into tool → Analyze → review correlations panel → download ZIP

## GitHub

Repo: `git@github.com:ScriptSavant1/vugenCustomRecorder.git` (branch: `main`)
