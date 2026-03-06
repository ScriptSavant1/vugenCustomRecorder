# VuGen HAR Script Generator — Technical Reference

> **Audience:** Developers maintaining or extending `VuGen-Recorder.html` and `VuGen-Script-Studio.html`.
> Covers problem context, architecture, every implemented feature, and how to add new ones.

---

## Table of Contents

1. [Why This Exists](#1-why-this-exists)
2. [Solution Overview](#2-solution-overview)
3. [Shared Concepts](#3-shared-concepts)
4. [VuGen-Recorder.html — Architecture](#4-vugen-recorderhtml--architecture)
5. [VuGen-Script-Studio.html — Architecture](#5-vugen-script-studiohtml--architecture)
6. [Correlation Engine (Script Studio)](#6-correlation-engine-script-studio)
7. [Authentication Detection](#7-authentication-detection)
8. [Header Logic](#8-header-logic)
9. [Parameterization](#9-parameterization)
10. [Script Generation](#10-script-generation)
11. [NetLog Support](#11-netlog-support)
12. [SSO / OAuth / Redirect Chain Handling](#12-sso--oauth--redirect-chain-handling)
13. [ZIP Download](#13-zip-download)
14. [How to Add a New Feature](#14-how-to-add-a-new-feature)
15. [Known Limitations](#15-known-limitations)
16. [Testing Guide](#16-testing-guide)

---

## 1. Why This Exists

### The Problem

VuGen (LoadRunner's script recorder) captures HTTP traffic by installing **WinPcap or NPCAP** — kernel-level network capture drivers. In enterprise environments, these drivers are blocked by:
- Endpoint protection / AV software
- Group Policy / org security policy
- Users without admin rights

When the driver installation fails, VuGen shows a cryptic "proxy recording failed" or "no network interface available" error. The user can open VuGen but cannot record any traffic.

### The Failed Alternatives

| Option | Why it fails |
|---|---|
| VuGen Proxy recording | Requires system proxy settings — blocked by corporate proxy / SSL inspection |
| Browser Extension recording | Requires extension install — IT-managed browsers often block unsigned extensions |
| Fiddler / Charles | Require installation, admin rights |
| VuGen TruClient | Can record but output is JavaScript simulation, not HTTP replay |

### The Root Cause of the Solution

**DevTools HAR export** works in any browser with zero installation. `F12 → Network → Export HAR` captures every HTTP request the browser makes, including headers, bodies, timing, cookies, and response content. This is standard browser functionality — no admin rights, no extension, no driver.

The gap: a HAR file is raw data. Converting it to a working VuGen script (either **Web HTTP/HTML** C language or **DevWeb** JavaScript) requires:
- Filtering noise (analytics, static assets, OPTIONS preflight)
- Detecting transaction boundaries
- Identifying dynamic values (session tokens, CSRF tokens, IDs) that must be correlated
- Generating syntactically correct VuGen API calls
- Packaging into a valid VuGen project ZIP

That conversion is what these two tools do.

---

## 2. Solution Overview

### Two Tools, Two Stages

```
Stage 1: VuGen-Recorder.html
  Input:  .har file OR chrome://net-export/ .json
  Output: Complete VuGen project ZIP (basic script, no correlation)
  Use for: Quick first-cut scripts, verifying your HAR capture works

Stage 2: VuGen-Script-Studio.html
  Input:  1 or 2 HAR files (or NetLog JSON)
  Output: Complete VuGen project ZIP (correlated, parameterized, exception-handled)
  Use for: Production-ready scripts that work under load with multiple users
```

### Why Two Tools?

The Recorder is intentionally simple — it gives users a script in under 30 seconds. The Studio is complex because full correlation requires either two recordings (to diff dynamic values) or sophisticated pattern detection. Keeping them separate means users can start with the Recorder without being overwhelmed.

### Technology Choices

| Decision | Choice | Reason |
|---|---|---|
| Delivery | Single `.html` file | Zero installation, works everywhere |
| Dependencies | None (JSZip inlined) | Can't rely on CDN availability in enterprise |
| State management | Global `S` object | Simple, debuggable in browser console |
| ZIP creation | JSZip v3.10.1 (inlined) | Proven library, ~100KB minified |
| Languages | Pure HTML/CSS/ES6 JS | No transpilation, no build step |
| Correlation | Two-HAR diff algorithm | 100% deterministic, no AI/ML, offline |

---

## 3. Shared Concepts

### Entry Shape

Both tools normalize all input (HAR or NetLog) into the same entry object shape:

```javascript
entry = {
  id,           // unique integer
  url,          // full URL string
  method,       // 'GET', 'POST', etc.
  status,       // HTTP response status code (or 0 for markers)
  ct,           // response Content-Type (short, e.g. 'application/json')
  hdrsMap,      // { 'header-name': 'value' } — request headers (lowercased keys)
  reqHdrs,      // [ { name, value } ] — request headers array (for ordered iteration)
  respHdrsMap,  // { 'header-name': 'value' } — response headers (lowercased keys)
  body,         // { text: string, mimeType: string } — request body
  respBody,     // response body text (for correlation back-tracing)
  referer,      // Referer header value (shortcut)
  txn,          // transaction name this entry belongs to (or null)
  filtered,     // boolean — excluded from script output
  isMarker,     // boolean — bookmarklet START/END marker entry
}
```

### Transaction Markers

Users drag bookmarklets to their browser toolbar. Each bookmarklet fires:
```javascript
fetch("https://START-T01_Login.invalid");
fetch("https://END-T01_Login.invalid");
```

These generate entries in the HAR with `.invalid` TLD (unreachable, status 0 or CORS error). The tools detect them by regex:
```
/START[-_](.+?)\.invalid/i   or   /(.+?)[-_]START\.invalid/i
/END[-_](.+?)\.invalid/i     or   /(.+?)[-_]END\.invalid/i
```

Hyphens in transaction names are converted to underscores (C identifier constraint). A transaction named `T01-Login` becomes `T01_Login` in generated code.

### Filtering

`applyFilters()` marks `entry.filtered = true` for:
- **Static assets**: images, fonts, CSS, JS (by Content-Type and URL extension)
- **Noise domains**: Google Analytics, Hotjar, Intercom, Sentry, etc. (`NOISY` regex)
- **OPTIONS preflight**: browser CORS pre-flight requests (not replayed by VuGen)
- **CSP violation reports**: browser auto-sends these when `.invalid` URLs are blocked by app CSP
- **WebSocket upgrades**: `wss://` and `ws://` URLs (VuGen Web HTTP/HTML can't replay them)
- **Domain filter**: hostnames the user has unchecked in the domain sidebar

Marker entries (`isMarker=true`) are **never filtered** — they define transaction boundaries.

### Protocol Output Formats

**Web HTTP/HTML (C language)** — VuGen's classic protocol:
- `Action.c` — main iterated script
- `vuser_init.c` — one-time init per vuser
- `vuser_end.c` — one-time cleanup per vuser
- `globals.h` — standard includes
- Plus config files: `.usr`, `default.cfg`, `default.usp`, `ParameterFile.prm`, `collection_data.dat`, `ScriptUploadMetadata.xml`

**DevWeb (JavaScript)** — VuGen's modern protocol:
- `main.js` — single entry point with `load.initialize()`, `load.action()`, `load.finalize()`
- Plus config files: `rts.yml`, `scenario.yml`, `parameters.yml`, `collection_data.csv`, `tsconfig.json`, `package.json`, `DevWebSdk.d.ts`

---

## 4. VuGen-Recorder.html — Architecture

### State Object

```javascript
S = {
  entries[],        // normalized entries from HAR or NetLog
  txns[],           // [{name, color}] detected transactions
  colorMap{},       // txnName → color CSS string
  selMode,          // boolean: manual select mode
  selIds,           // Set of selected entry IDs (manual mode)
  tab,              // active preview tab: 'ac'|'mj'|'vi'|'ve'|'gh'
  scripts{},        // generated script strings keyed by tab
  format,           // 'webhttp' | 'devweb' | 'both'
  pendingHar,       // parsed HAR JSON waiting while format modal is open
  pendingNetLog,    // parsed NetLog JSON waiting while format modal is open
  isNetLogSource,   // true when source is chrome://net-export/ JSON
  domainFilter{},   // hostname → boolean (false = excluded)
  domainStats{},    // hostname → {count, size}
  auth,             // {type, realm, host} — detected auth scheme
  serverHost,       // {host, proto, prefix} — dominant hostname
  collapsed{},      // txnName → boolean for collapse/expand state
}
```

### Processing Pipeline

```
User drops file
  readFile(file)
    isNetLog(obj)?
      → S.pendingNetLog = obj
    else
      → S.pendingHar = obj
    openFmtModal()         ← user selects Web HTTP/HTML / DevWeb / Both
      onFmtConfirm()
        processHAR(har) or processNetLog(netlog)
          parseHAR(har) or parseNetLog(netlog)  ← normalize to entry shape
          detectMarkers(entries)                ← find .invalid URLs
          detectAuth(entries)                   ← auth scheme detection
          detectServerHost(entries)             ← dominant hostname
          buildDomainStats(entries)
          renderDomainPanel()                   ← left sidebar checkboxes
          applyFilters(entries)                 ← mark filtered=true
          buildScripts()
            genActionC()                        ← if format includes webhttp
            genMainJS()                         ← if format includes devweb
            genVuserInit() / genVuserEnd() / genGlobalsH()
          renderTable(entries)                  ← request table with txn bands
          showScript()                          ← display active tab
```

### UI Components

| Element | Purpose |
|---|---|
| Drop zone / file input | Accepts `.har` and `.json` |
| Format modal | User picks Web HTTP/HTML / DevWeb / Both |
| Domain panel (left) | Checkbox per hostname — toggle inclusion |
| Left resizer | Drag to resize domain panel |
| Request table (center) | Rows grouped by transaction with color bands |
| Collapse/Expand controls | `⊖ Collapse All` / `⊕ Expand All` + per-transaction toggle |
| Preview tabs (right) | `Action.c`, `main.js`, `vuser_init.c`, `vuser_end.c`, `globals.h` |
| Right resizer | Drag to resize code panel |
| NetLog banner | Amber warning bar when source is NetLog (POST bodies missing) |
| Auth badge | Shows detected auth scheme in toolbar |
| Download button | Triggers ZIP generation and download |

### Key Functions

| Function | What it does |
|---|---|
| `readFile(file)` | Detects HAR vs NetLog, stores pending, opens format modal |
| `parseHAR(har)` | Converts HAR entries to normalized entry shape |
| `parseNetLog(netlog)` | Parses chrome://net-export/ JSON to entry shape |
| `detectMarkers(entries)` | Finds START/END .invalid URLs, assigns txn names |
| `detectAuth(entries)` | Returns {type, realm, host} for authentication |
| `detectServerHost(entries)` | Returns dominant hostname (>35% of requests) |
| `applyFilters(entries)` | Sets entry.filtered based on all filter rules |
| `genActionC()` | Generates Action.c source string |
| `genMainJS()` | Generates main.js source string |
| `buildZip()` | Creates JSZip object, adds all files, triggers download |
| `renderTable(entries)` | Builds HTML for the request table |
| `toggleDomain(host, checked)` | Updates domainFilter, re-runs applyFilters + buildScripts |
| `toggleTxn(row)` | Collapses/expands a transaction group |

---

## 5. VuGen-Script-Studio.html — Architecture

### State Object

```javascript
S = {
  har1, har2,                // raw parsed HAR/NetLog objects
  entries1, entries2,        // normalized entry arrays
  txns,                      // [{name, color}] from entries1
  correlations,              // [{name, sourceIdx, extractorType, extractorConfig, usages[]}]
  candidates,                // [{name, hint, usages[]}] — unresolved dynamic values
  params,                    // [{key, csvKey, values[]}] — parameterized fields
  harWarning,                // string — warning message shown in UI
  scripts,                   // {ac: string, mj: string} — generated script text
  format,                    // 'webhttp' | 'devweb' | 'both'
  mode,                      // 'single' | 'two' — HAR count
  tab,                       // active preview tab
  auth,                      // {type, realm, host}
  serverHost,                // {host, proto, prefix}
  isNetLog1, isNetLog2,      // boolean flags for NetLog source
}
```

### Processing Pipeline

```
User drops 1 or 2 files
  loadFile(file, slot)       ← slot = 1 or 2
  [Analyze & Generate] clicked
  analyze()
    parseHAR/parseNetLog → entries1, entries2
    detectAuth(entries1)
    detectServerHost(entries1)
    applyFilters(entries1, entries2)
    mode = two-HAR if entries2 present, else single
    if two-HAR:
      twoHarCorrelate(entries1, entries2)
        matchPairs()                    ← by method + normalizeUrlKey(url)
        diffChangedVals()               ← find values that differ between recordings
        findValueBefore()               ← back-trace to source response in HAR1
        fallback chain (1→2→2b→2c→2d→3) ← when source not found directly
        generateExtractors()
    if single-HAR:
      singleHarCorrelate(entries1)
        patternScan()                   ← JWT Bearer, CSRF headers, session cookies
        redirectChainScan()             ← 3xx Location → next URL dynamic segments
    detectParams(entries1, correlations)
    SSO warning detection
    buildScripts()
      genMainJS(entries1, correlations)
      genActionC(entries1, correlations)
    renderResults()
    renderCorrelationsPanel()
    renderParamsPanel()
```

### Correlation Object Shape

```javascript
correlation = {
  name,           // 'AuthToken', 'JSessionId', 'CsrfToken', etc.
  sourceIdx,      // index in entries1 of the response that returns the value
                  // -1 for generate-type (client-side generated)
  extractorType,  // 'jsonpath' | 'boundary' | 'boundary_header' | 'html' | 'cookie' | 'generate'
  extractorConfig,// type-specific config object (see below)
  usages,         // [{reqIdx, location, key, originalValue, tokenValue}]
}
```

**Extractor configs:**

| Type | Config shape | Meaning |
|---|---|---|
| `jsonpath` | `{expression: '$.access_token'}` | JSON path in response body |
| `boundary` | `{lb: 'name="csrf" value="', rb: '"'}` | Left/right boundary in response body |
| `boundary_header` | `{lb: 'X-Token: ', rb: '\r\n'}` | Left/right boundary in response headers |
| `html` | `{lb: 'name="__VIEWSTATE" value="', rb: '"'}` | HTML hidden input |
| `cookie` | `{cookieName: 'JSESSIONID'}` | Set-Cookie response header |
| `generate` | `{pattern: 'uuid'|'hex', length: 32}` | Client-side generated — no server extraction |

### Usage Object Shape

```javascript
usage = {
  reqIdx,         // index in entries1 of the request that sends the value
  location,       // 'url_path' | 'url_path_seg' | 'query' | 'header' | 'cookie' | 'body_form' | 'body_json' | 'body_xml'
  key,            // field name / header name / query param name
  originalValue,  // the raw value from HAR1 (used to find and replace)
  tokenValue,     // same as originalValue usually; used for matching in body substitution
}
```

---

## 6. Correlation Engine (Script Studio)

### Why Two HARs?

Record the exact same user flow twice, in two completely fresh browser sessions. Values that **differ** between the two recordings are **dynamic** (session tokens, IDs, timestamps, CSRF tokens) — they must be extracted at runtime. Values that are **the same** are **static** — hardcode them.

This approach is 100% deterministic, requires no AI, and works completely offline.

### Step 1: Match Request Pairs

For each request in HAR1, find its counterpart in HAR2 using:
```
match key = HTTP method + normalizeUrlKey(url)
```

`normalizeUrlKey(url)` replaces dynamic segments in the URL before matching:
- UUIDs → `{uuid}`
- hex32/hex64/hex16 tokens → `{hex}`
- JWTs → `{jwt}`
- Numeric IDs (4+ digits) → `{id}`
- Long tokens (32+ chars) → `{tok}`
- Mid-length mixed alphanumeric IDs (10+ chars, ≥25% digits or has separator) → `{mid}`

This allows REST APIs like `/orders/12345/items` to correctly match `/orders/67890/items`.

### Step 2: Diff Changed Values

For each matched pair (HAR1 request ↔ HAR2 request), compare all of:

| Location | What is compared |
|---|---|
| `url_path` | `;key=value` matrix params (e.g. `;jsessionid=xxx`) |
| `url_path_seg` | Individual path segments that are dynamic |
| `query` | URL query string `?key=value` pairs |
| `header` | All request headers (except SKIP_HDRS) |
| `cookie` | Individual cookies parsed from `Cookie:` header |
| `body_form` | `application/x-www-form-urlencoded` body fields |
| `body_json` | JSON body fields (recursive via `jsonDiffFlat()`) |
| `body_xml` | XML body leaf text nodes `<tag>value</tag>` |

A changed value is recorded only if `isDynamic(value)` returns true (minimum length 8 chars):

| Pattern | Covers |
|---|---|
| JWT | `eyJ...` three-part base64url tokens |
| UUID | 8-4-4-4-12 hex format |
| hex32/hex64 | MD5 hashes, SHA-256 hashes |
| hex16 | Short hex tokens |
| longToken (≥32 chars) | OAuth tokens, base64url encoded secrets |
| midToken (12–35 chars) | Request/trace/correlation IDs, instance IDs |
| numericId (15+ digits) | Snowflake IDs, GCP instance IDs |
| compound | `;` or `|` separated multi-segment keys |

### Step 3: Back-Trace to Source (`findValueBefore`)

For each changed value (HAR1 value), search backwards through all previous responses in HAR1 to find where it was set.

`findValueInResponse(entry, value)` selects the **most precise extractor** based on where the value is found and the response content-type. Extractor type is chosen for both correctness and performance (specific extractors are faster/more reliable than generic boundary scanning):

| Priority | Location | Extractor type | Why chosen |
|---|---|---|---|
| 1 | `Set-Cookie` header | `cookie` | O(1) cookie jar lookup — most specific |
| 2 | 3xx `Location` header | `boundary_header` | Header-only scan; OAuth2/OIDC redirect values |
| 3 | Other response headers | `boundary_header` | Header-only scan; skips body entirely |
| 4 | JSON body | `jsonpath` | Compiled JSONPath; type-safe; fastest body lookup |
| 5 | XML body | `boundary` (scope:Body) | No XmlPathExtractor in SDK; boundary is safe |
| 6 | HTML `<input name=x>` | `html` | DOM-parsed CSS selector; immune to whitespace variations |
| 7 | Vue/React `:prop=` | `boundary` (scope:Body) | Non-standard HTML — CSS selector cannot target `:prop=` |
| 8 | HTML `data-*` attribute | `html` | DOM-parsed; `[data-csrf]` CSS selector is reliable |
| 9 | JS `var x = "VALUE"` | `boundary` (scope:Body) | Inside `<script>` — no DOM selector available |
| 10 | HTML `<meta name=x>` | `html` | DOM-parsed; `meta[name='csrf-token']` CSS selector |
| 11 | Universal fallback | `boundary` (scope:Body) | Last resort; 8-char context around value |

**Why `HtmlExtractor` is preferred over `BoundaryExtractor` for HTML elements:** `HtmlExtractor` uses proper DOM parsing (like `querySelector`), making it immune to attribute ordering, whitespace, and encoding variations. `BoundaryExtractor(scope:Body)` is a raw string scan that could match the wrong occurrence if the same text appears elsewhere on the page.

### Step 4: Fallback Chain

When `findValueBefore()` returns null (value not found in any earlier response):

**Fallback 1 — Cross-HAR:** Look for the value in `entries2` (HAR2). If found there, reuse the extractor type. (Covers the case where HAR1 had a cached response with empty body.)

**Fallback 2 — Pattern-based hidden fields:** If the field name matches `KNOWN_HIDDEN_FIELD` regex (covers: `__VIEWSTATE`, `__EVENTVALIDATION`, `csrfmiddlewaretoken`, `_token`, `form_key`, `authenticity_token`, `_csrf`, `_sourcePage`, `__fp`, `javax.faces.ViewState`, `__RequestVerificationToken`), generate an `html` extractor on the last GET request before this POST.

**Fallback 2b — Session cookie in URL path:** If value matches a session cookie pattern (`JSESSIONID`, `PHPSESSID`, `ASP.NET_SessionId`, etc.) and it's used in a URL path, extract from the first `Set-Cookie` in the first real response.

**Fallback 2c — CSRF double-submit:** If the value is used in a CSRF-like header AND the same value is present in the `Cookie:` request header, generate a cookie extractor. Works even when `Set-Cookie` is absent (browser cached the login response).

**Fallback 2d — Client-generated tokens:** If the value matches UUID/hex patterns and is used in a CSRF-like header, and all other fallbacks fail, classify as `extractorType: 'generate'`. VuGen generates a fresh random token each iteration using `lr_param_sprintf()` (Web HTTP/HTML) or `load.utils.uuid()` / `load.utils.randomString(n, {hex: true})` (DevWeb — VuGen native API, NOT Node.js `require('crypto')`).

**Fallback 3 — Candidate:** Push to `S.candidates`. Script emits a TODO comment. User must add the extractor manually.

### Single-HAR Mode

When only one HAR is provided, no diff is possible. Instead, `singleHarCorrelate()` uses pattern detection:

1. **JWT Bearer tokens** — `Authorization: Bearer eyJ...` → `jsonpath` or `boundary` from the response that issued the token
2. **CSRF headers** — headers matching `CSRF_HEADER_PATTERN` (`/csrf|xsrf|antiforg|request.?verif/i`) → cookie double-submit check → generate-type fallback
3. **Session cookies** — `JSESSIONID`, `PHPSESSID`, etc. in `Set-Cookie` responses
4. **Redirect chain** *(NEW — Phase 14)* — 3xx responses with dynamic path segments or query params in their `Location` header, if those values appear in a subsequent request URL or body

---

## 7. Authentication Detection

`detectAuth(entries)` scans all entries and returns the highest-priority auth type found:

| Type | Detection | Priority | Code generated |
|---|---|---|---|
| Kerberos | `Authorization: Negotiate <token>` where decoded token ≠ NTLMSSP | 7 | `web_set_user()` + Kerberos runtime settings |
| NTLM | `Authorization: Negotiate/NTLM <NTLMSSP token>` | 6 | `web_set_user()` + NTLM runtime settings |
| Negotiate | `WWW-Authenticate: Negotiate` in 401 response | 5 | `web_set_user()` |
| Digest | `WWW-Authenticate: Digest realm="..."` | 4 | `web_set_user()` |
| Basic | `WWW-Authenticate: Basic realm="..."` | 3 | `web_set_user()` |
| Bearer | `Authorization: Bearer <token>` in request | 2 | Dynamic extraction (not web_set_user) |
| SAML | `SAMLResponse` or `SAMLRequest` in POST body | 1 | TODO comment |

**Kerberos vs NTLM discrimination:** The Negotiate token is base64-decoded. If the first 8 bytes spell `NTLMSSP\x00`, it is NTLM; otherwise Kerberos.

**Generated Web HTTP/HTML code (Kerberos example):**
```c
// Kerberos Authentication — credentials supplied via parameter file
web_set_user("{AuthUsername}", "{AuthPassword}", "app.company.com:443");
// Runtime Settings → Internet Protocol → Preferences → Authentication:
//   [x] Enable Integrated Authentication
//   [x] Use canonical name in SPN (Kerberos only)
```

**Generated DevWeb code:**
```javascript
load.setUserCredentials({
    username: load.params.AuthUsername,
    password: load.params.AuthPassword,
    host: "app.company.com:443"  // "*" for Basic/Digest (any host)
});
```

**Runtime settings auto-patching:**
- `genDefaultCfg(auth)` — modifies `default.cfg` INI string: sets `IntegratedAuthentication=1`, `SPNCNameLookup=1` (Kerberos), `UseNativeNTLM=1`, `OverrideNTLMCreds=1` (NTLM)
- `genRtsYml(auth)` — modifies `rts.yml`: sets `enableIntegratedAuthentication: true`

**Parameter file:** `AuthUsername` and `AuthPassword` are automatically added to `S.params` so they appear in the CSV and parameter file.

---

## 8. Header Logic

Correct header generation is critical — VuGen auto-adds many headers at runtime (Accept-Encoding, Host, Connection, etc.). Over-emitting headers causes duplicates; under-emitting causes replay failures.

### Web HTTP/HTML — `web_add_auto_header` vs `web_add_header`

- **`web_add_auto_header("Name", "Value")`** — applies to ALL subsequent requests until overridden
- **`web_add_header("Name", "Value")`** — applies to the NEXT request only

### Algorithm

**Step 1 — Compute key-presence frequency:**
```javascript
// Count how many entries HAVE this header key (any value)
hdrKeyFreq[k] = count of entries where k is present
```

**Step 2 — Find global headers (autoHdrs):**
```javascript
threshold = ceil(visEntries.length * 0.8)  // 80% of visible entries
if hdrKeyFreq[k] >= threshold:
  autoHdrs[k] = most_common_value(hdrFreq[k])
```

Why key-presence (not value-equality): If `Accept` has two values (`text/html` for navigation requests, `application/json` for XHR), neither value reaches 80% frequency. But the key `accept` appears on 100% of entries → it IS a global header. The most common value (`application/json` for an API-heavy app) becomes the global default; navigation requests emit an override.

**Step 3 — Force-global headers:**
```javascript
// user-agent and accept-language are always browser session constants
// Force them to global regardless of count
for fk in ['user-agent', 'accept-language']:
  if fk in hdrFreq and fk not in autoHdrs:
    autoHdrs[fk] = most_common_value(hdrFreq[fk])
```

**Step 4 — Per-request headers (overrides only):**
```javascript
for each request header h:
  if h.name in SKIP_HDRS: continue              // never emit
  if autoHdrs[h.name] === h.value: continue    // global covers this → skip
  if h.name === 'accept' and h.value === '*/*': continue  // DevWeb/VuGen default
  if h.name === 'authorization' and auth.type in [kerberos, ntlm, ...]: continue
  emit web_add_header(h.name, h.value)         // value differs from global → override
```

### Skipped Headers (SKIP_HDR_AC / SKIP_HDRS)

These are auto-managed by VuGen/browser at runtime — emitting them causes duplicates or errors:

```
host, content-length, connection, keep-alive, transfer-encoding, te, trailer,
expect, via, accept-encoding, cookie, cookie2,
sec-ch-ua, sec-ch-ua-mobile, sec-ch-ua-platform,
sec-fetch-dest, sec-fetch-mode, sec-fetch-site, sec-fetch-user,
upgrade-insecure-requests, dnt, priority,
if-none-match, if-modified-since, if-unmodified-since, if-match,
cache-control, pragma, x-forwarded-for,
referer, content-type  (Web HTTP/HTML only — VuGen handles these)
```

**`cookie` and `cookie2`** are excluded because VuGen's cookie jar manages them automatically. Emitting a static `Cookie:` header would bypass the jar and cause failures after iteration 1.

---

## 9. Parameterization

`detectParams(entries1, correlations)` scans POST body fields against `PARAM_KEYS_MAP` patterns:

| Field regex | CSV column | Example |
|---|---|---|
| `username\|_username\|loginName` | `Username` | `john.doe` |
| `password\|_password\|passwd` | `Password` | `secret123` |
| `fromPort\|depart\|from_city\|origin` | `FromCity` | `JFK` |
| `toPort\|arrive\|to_city\|destination` | `ToCity` | `LHR` |
| `outDate\|depart_date\|departure` | `DepartDate` | `2024-06-15` |
| `returnDate\|return_date\|arrival_date` | `ReturnDate` | `2024-06-22` |
| `firstName\|first_name` | `FirstName` | `John` |
| `lastName\|last_name` | `LastName` | `Doe` |
| `creditCard\|credit_card\|cardNumber` | `CardNumber` | `4111111111111111` |
| `email\|e_mail` | `Email` | `user@example.com` |

**Pre-populated special params (always added when detected):**
- `AuthUsername`, `AuthPassword` — when auth detected
- `ServerHost` — when dominant hostname found

**Exclusion rules:**
- Value already in `corrValues` (correlation handles it)
- Value matches `isDynamic()` AND field name doesn't match `PARAM_KEYS_MAP` (credit card is a param, random session token is not)

**Substitution:**
- DevWeb: `${load.params.CsvKey}` in URL and body template literals
- Web HTTP/HTML: `{CsvKey}` in URL strings and body

**`processField(key, value)` helper** strips dot-prefix (`order.creditCard` → `creditCard`) and colon-prefix (Oracle ADF/SAP) before matching patterns.

### Parameter File Formats

**`ParameterFile.prm` (Web HTTP/HTML):**
```ini
[parameter:Username]
ColumnName="Username"
Delimiter=","
GenerateNewVal="EachIteration"
ParamName="Username"
SelectNextRow="Sequential"
StartRow="1"
Table="collection_data.dat"
TableLocation="Local"
Type="Table"
OutOfRangePolicy="ContinueWithLast"
auto_allocate_block_size="1"
value_for_each_vuser=""
OriginalValue="testuser"
```

Critical rules:
- `Type="Table"` NOT `"File"` — VuGen won't recognise "File"
- No leading spaces on any key
- `ColumnName` must match CSV header exactly

**`parameters.yml` (DevWeb):**
```yaml
parameters:
  - name: Username
    type: csv
    fileName: collection_data.csv
    columnName: Username
    nextValue: iteration
    nextRow: sequential
    onEnd: loop
```

---

## 10. Script Generation

### Web HTTP/HTML — `genActionC()`

Structure:
```c
#include "globals.h"

void gen_XCSRFToken() { /* generate-type helper if needed */ }

Action() {
    web_set_sockets_option("SSL_VERSION", "AUTO");
    web_set_user("{AuthUsername}", "{AuthPassword}", "host:port");  // if auth detected
    lr_save_string("actual_host", "ServerHost");                   // if serverHost detected
    web_add_auto_header("User-Agent", "...");  // global headers
    web_add_auto_header("Accept-Language", "...");

    lr_start_transaction("T01_Login");

    // HTTP 302 redirect — VuGen follows subsequent redirects automatically (if 3xx)
    web_reg_save_param("AuthToken", "LB=...", "RB=...", "Ord=1", LAST);  // BEFORE the source request
    web_url("stepName",
        "URL={ServerHost}/path",
        "Resource=0",
        "RecContentType=application/json",
        "Referer=",
        "Snapshot=t1.inf",
        "Mode=HTML",
        LAST);

    web_custom_request("apiCall",
        "URL={ServerHost}/api/login",
        "Method=POST",
        "Resource=0",
        "RecContentType=application/json",
        "Referer=",
        "Snapshot=t2.inf",
        "Mode=HTML",
        "EncType=application/json",
        "BodyBinary={\"username\":\"{Username}\",\"token\":\"{AuthToken}\"}",
        LAST);

    lr_end_transaction("T01_Login", LR_AUTO);
    return 0;
}
```

**`BodyBinary=` rule:** ALL POST bodies use `BodyBinary=` not `Body=`. VuGen's attribute parser chokes on `Body=` when the body contains `{`, `}`, `:`, or other special chars (common in JSON). `BodyBinary=` treats the value as a raw string — no parsing.

**Body chunking:** Bodies longer than 200 chars are split across adjacent C string literals (no comma, just juxtaposition — C string literal concatenation). Split points are escape-sequence-aware: never split inside `\xHH` hex escape sequences.

### DevWeb — `genMainJS()`

Structure:
```javascript
// VuGen DevWeb script — main.js
// Generated by VuGen-Script-Studio.html

load.initialize("Initialize", async function () {
    load.WebRequest.defaults.returnBody = false;
    load.WebRequest.defaults.headers = {
        "User-Agent": "...",
        "Accept-Language": "..."
    };
    load.setUserCredentials({ ... });  // if auth detected
    // load.global.X declarations for correlation values
});

load.action("Action", async function () {
    let _activeTxn = null;
    try {
        const T01 = new load.Transaction("T01_Login");
        T01.start(); _activeTxn = T01;

        // HTTP 302 redirect — VuGen follows subsequent redirects automatically (if 3xx)
        const webResponse_01 = await new load.WebRequest({
            id: 1,
            url: `https://${load.params.ServerHost}/path`,
            method: "GET",
            headers: { "Accept": "text/html" },  // only if differs from default
            returnBody: true,
            extractors: [
                new load.BoundaryExtractor("AuthToken", {leftBoundary: "...", rightBoundary: "...", scope: load.ExtractorScope.Body})
            ]
        }).send();
        load.global.AuthToken = webResponse_01.extractors.AuthToken;

        T01.stop(load.TransactionStatus.Passed); _activeTxn = null;
    } catch(e) {
        if (_activeTxn) _activeTxn.stop(load.TransactionStatus.Failed);
        load.log(`Action failed: ${e.message}`, load.LogLevel.error);
        load.exit(load.ExitType.iteration, "Action failed");
    }
});

load.finalize("Finalize", async function () {
});
```

**Key rules:**
- `await .send()` not `sendSync()` — action functions are `async`
- `returnBody: true` only on requests where response body is needed (extractors or validation)
- Form POST bodies rendered as objects: `body: { username: "...", token: "..." }`
- JSON/XML/other bodies rendered as template literals

### Extractor Code Generation — Both Outputs

The same `extractorType` value drives code generation for both outputs via `devwebExtractorCode()` and `webHttpCorrCode()`:

| `extractorType` | DevWeb (`devwebExtractorCode`) | Web HTTP/HTML (`webHttpCorrCode`) |
|---|---|---|
| `jsonpath` | `new load.JsonPathExtractor("name", "$.path")` | `web_reg_save_param_json("name", "QueryString=$.path", "Ord=1", LAST)` |
| `cookie` | `new load.CookieExtractor("name", {cookieName: "x"})` | `web_reg_save_param("name", "LB=x=", "RB=;", "Search=Headers", "Ord=1", LAST)` |
| `html` | `new load.HtmlExtractor("name", "selector", "attr")` | `web_reg_save_param` with boundary derived from selector (input→`name="x" value="`, meta→`name="x" content="`, data-*→`data-x="`) |
| `boundary_header` | `new load.BoundaryExtractor("name", {leftBoundary, rightBoundary, scope: load.ExtractorScope.Headers})` | `web_reg_save_param("name", "LB=...", "RB=\r\n", "Search=Headers", "Ord=1", LAST)` |
| `boundary` | `new load.BoundaryExtractor("name", {leftBoundary, rightBoundary, scope: load.ExtractorScope.Body})` | `web_reg_save_param("name", "LB=...", "RB=...", "Ord=1", LAST)` |
| `generate` | `load.utils.uuid()` or `load.utils.randomString(n, {hex: true})` in a `gen_NAME()` helper | `lr_param_sprintf` in a `void gen_NAME()` helper |

**Important notes:**
- `BoundaryExtractor` always uses the **options form** with explicit `scope` — the 3-arg positional form scans both headers and body (ambiguous); options form is precise
- `web_reg_save_param` (NOT `web_reg_save_param_ex`) is used for all body/boundary extraction in Web HTTP/HTML — `_ex` does not accept `Search=Body`
- All extractors are emitted **BEFORE** the request that returns the value (VuGen registration rule)

---

## 11. NetLog Support

### Problem

DevTools (F12) captures only the current browser tab. If the application opens **popup windows** or **new tabs** (common in SSO flows, OAuth authorization pages, and form submissions that open new windows), those requests are NOT captured in the HAR.

### Solution

`chrome://net-export/` captures ALL traffic at the Chrome network stack level simultaneously — all tabs, all windows, background requests. Users navigate to this page, click "Start Logging to Disk", perform their workflow, click "Stop", and export a `.json` file.

### NetLog vs HAR differences

| Aspect | HAR | NetLog |
|---|---|---|
| Multi-tab | Single tab only | All tabs |
| Request bodies | Captured | Captured (GET only for response bodies) |
| POST bodies | Captured | **NOT captured** |
| Response bodies | Captured | **NOT captured** |
| Format | Standard JSON (`.har`) | Chrome-specific JSON (`.json`) |

### Implementation

`isNetLog(obj)` detects the format:
```javascript
return obj?.constants?.logEventTypes && Array.isArray(obj?.events)
```

`parseNetLog(netlog)` groups events by `source.id`, extracts:
- URL and method from `URL_REQUEST_START_JOB` events
- Request headers from `SEND_REQUEST_HEADERS` events
- Response headers from `READ_RESPONSE_HEADERS` events
- Sorts by `tickOffset + event.time`
- Returns entries in the same shape as `parseHAR()`

**Bookmarklet markers** (`fetch("https://START-T01.invalid")`) ARE captured in NetLog — they fire from any tab.

**Missing POST bodies:** When `entry.body === null` from NetLog and method is POST/PUT/PATCH, both generators emit:
```javascript
// TODO: POST body not available in NetLog recording.
// Use DevTools HAR recording if you need the request body captured.
```

**UI indicator:** An amber banner `#netlog-banner` is displayed when source is NetLog, reminding users that POST bodies are unavailable.

---

## 12. SSO / OAuth / Redirect Chain Handling

### The Problem

Enterprise applications frequently use SSO (Single Sign-On) via OAuth2/OIDC or SAML. These flows produce chains of HTTP redirects. A typical PingFederate / ADFS / Okta flow in HAR looks like:

```
GET /app/page
  → 302 Location: /as/authorisation.oauth2?response_type=id_token&client_id=...
GET /as/authorisation.oauth2?...
  → 302 Location: /as/MB7gGaMbSW/resume/as/authorisation.oauth2?...
GET /as/MB7gGaMbSW/resume/as/authorisation.oauth2?...
  → 200 (HTML page with auto-submit form containing id_token and state)
POST /pa/oidc/cb  body: id_token=eyJ...&state=abc123
  → 302 Location: /app/page (back to app)
```

Dynamic values:
- `MB7gGaMbSW` — PingFederate AS session ID, in step 2's `Location` header, used in step 3's URL
- `id_token` — JWT, in step 3's HTML form hidden input, used in step 4's POST body
- `state` — short random value, in step 3's HTML form, used in step 4's POST body

### What the Tool Does

**All 4 requests are included in the generated script** — VuGen needs them all for correct replay.

**3xx redirect comment:** Every 3xx entry is preceded by:
```c
// HTTP 302 redirect — VuGen follows subsequent redirects automatically
```
This reminds developers that VuGen does auto-follow GET 302s in HTML mode. The request is included for correlation completeness.

**Dynamic token correlation:**

1. `findValueInResponse()` section 1b (NEW — Phase 14):
   - When entry is 3xx, searches the `Location` response header for the value
   - If value is a path segment: LB = preceding path separator context, RB = `/` or `?`
   - If value is a query param: LB = `paramName=`, RB = `&` or `\r\n`
   - Returns `{type: 'boundary_header', config: {lb, rb}}`

2. `singleHarCorrelate()` redirect chain pass (NEW — Phase 14):
   - Iterates all 3xx entries
   - For each dynamic path segment in `Location` URL, checks if any later request uses it
   - For each dynamic query param, checks if any later request URL or body contains it
   - Generates `boundary_header` extractor from the Location header
   - Adds to correlations list

**SSO warning:** `analyze()` detects SSO flows via:
```javascript
SSO_URL_PATTERN = /\/oauth2?\/|\/oidc\/|\/as\/authoris|\/sso\/|\/saml\/|\/connect\/token|\.ping$|authorization\.oauth|\/login\?|\/auth\?|\.okta\.|\/adfs\/|\/realms\//i
```
If matched, a warning is added to `S.harWarning` explaining the redirect chain and confirming it is handled.

### `id_token` in form_post

The `id_token` is a JWT (`eyJ...`). It matches `isDynamic()` via the `jwt` pattern. `findValueInResponse()` finds it in the step 3 HTML response body as an `<input type="hidden" name="id_token" value="eyJ...">` → `html` extractor. The POST body substitution replaces it with `{IdToken}` (Web) or `${load.global.IdToken}` (DevWeb).

---

## 13. ZIP Download

Both tools use **JSZip v3.10.1** (fully inlined — no external network request). The entire ~100KB minified library is embedded as a string literal.

### `buildZip()` — Recorder

```javascript
const zip = new JSZip();
const folder = zip.folder(scriptName);
folder.file("Action.c", genActionC());
folder.file("vuser_init.c", genVuserInit());
folder.file("vuser_end.c", genVuserEnd());
folder.file("globals.h", genGlobalsH());
folder.file(scriptName + ".usr", genUsrFile(scriptName, S.params));
folder.file("default.cfg", genDefaultCfg(S.auth));
folder.file("default.usp", DEFAULT_USP);
folder.file("ScriptUploadMetadata.xml", genScriptUploadMetadata(scriptName, S.params));
if (S.params.length > 0) {
  folder.file("ParameterFile.prm", genParamFilePrm(S.params));
  folder.file("collection_data.dat", genCollectionDataCsv(S.params));
}
const dataFolder = folder.folder("data");
dataFolder.file("HostNames.dat", genHostNamesDat(S.entries));
dataFolder.file("cross_correlation_parameters.txt", "Delimiter:=\n");
// snapshot stubs
for (let i = 1; i <= snap; i++) dataFolder.file(`t${i}.inf`, "");
zip.generateAsync({type:"blob"}).then(blob => saveAs(blob, scriptName + ".zip"));
```

### VuGen Project Config Files

**`.usr` file** (INI format — critical fields):
```ini
[General]
ScriptName=MyScript
ParameterFile=ParameterFile.prm   ← REQUIRED when params exist (not empty string!)
ParamLeftBrace={
ParamRightBrace=}
ScriptLanguage=C
[Actions]
Action=Action.c
vuser_init=vuser_init.c
vuser_end=vuser_end.c
```

**`ScriptUploadMetadata.xml`** (must list param files when they exist):
```xml
<GeneralFiles>
  <FileEntry Name="ParameterFile.prm" Filter="4" />
  <FileEntry Name="collection_data.dat" Filter="4" />
</GeneralFiles>
```

---

## 14. How to Add a New Feature

### Pattern A — New filter rule

Add a condition to `applyFilters()`:
```javascript
// Example: filter out GraphQL introspection queries
if (e.body?.text?.includes('"__schema"')) { e.filtered = true; continue; }
```
Available in both files. No other changes needed — filtered entries are skipped by all generators.

### Pattern B — New correlation location (twoHarCorrelate)

In `diffChangedVals()`, add a new diff pass:
```javascript
// Example: diff gRPC metadata headers
for (const [k, v] of Object.entries(e1.grpcMetadata || {})) {
  const v2 = (e2.grpcMetadata || {})[k];
  if (v !== v2 && isDynamic(v)) {
    changedVals.push({ location: 'grpc_metadata', key: k, value1: v, value2: v2, reqIdx2 });
  }
}
```

Then handle the new location in `buildUsage()` and in both generators (substitution logic).

### Pattern C — New extractor type in `findValueInResponse()`

Add a new detection block between the existing sections:
```javascript
// Example: detect value in JWT payload (base64-decoded middle segment)
const parts = (entry.respBody || '').match(/eyJ[\w-]+\.([\w-]+)\./);
if (parts) {
  try {
    const payload = JSON.parse(atob(parts[1].replace(/-/g,'+').replace(/_/g,'/')));
    if (jsonSearchRecursive(payload, v)) {
      return { type: 'jwt_payload', config: { claim: foundPath } };
    }
  } catch {}
}
```

Then handle `type: 'jwt_payload'` in `webHttpCorrCode()` and `devwebCorrCode()`.

### Pattern D — New parameter key pattern

Add to `PARAM_KEYS_MAP`:
```javascript
const PARAM_KEYS_MAP = [
  // existing entries...
  { pattern: /^(search_term|q|query)$/i, csvKey: 'SearchTerm' },
  { pattern: /^(product_id|productId|item_id)$/i, csvKey: 'ProductId' },
];
```

No other changes needed — `detectParams()` applies all patterns automatically.

### Pattern E — New VuGen config file

In `buildZip()`:
```javascript
folder.file("custom_config.ini", genCustomConfig(S));
```

Add the generator function following the INI or YAML format.

### Pattern F — New entry-level metadata

Add detection in `processHAR()` or `parseNetLog()`:
```javascript
entry.isGraphQL = entry.url.includes('/graphql') && entry.method === 'POST';
```

Then reference `entry.isGraphQL` in generators to emit specialized code.

### Pattern G — New SSO pattern detection

Extend `SSO_URL_PATTERN` in `analyze()`:
```javascript
const SSO_URL_PATTERN = /\/oauth2?\/|\/oidc\/|...|\/your-new-idp\//i;
```

Or extend `singleHarCorrelate()` redirect chain pass to handle new token shapes.

---

## 15. Known Limitations

| Limitation | Impact | Workaround |
|---|---|---|
| NetLog missing POST bodies | Scripts missing POST payloads | Re-record with DevTools HAR if POST bodies needed |
| Browser-cached response bodies | Correlation fallbacks less accurate | Enable "Disable cache" in DevTools before recording |
| Same-session re-recording | Two-HAR produces 0 correlations | Use fresh Incognito window for HAR2 |
| Multipart file uploads | TODO comment emitted | Use `web_add_body_part()` (C) or `load.FormData()` (DevWeb) manually |
| WebSocket connections | Not captured | Script manually using `load.WebSocket` API |
| SAML IdP-initiated SSO | Redirect chain may need manual completion | Script the SAML assertion POST manually |
| GraphQL subscriptions | Use WebSocket transport — same as above | Script manually |
| Large .NET VIEWSTATE (single HAR) | TODO comment unless two-HAR used | Use two-HAR mode to auto-correlate |
| Client-side JS that triggers requests | HAR doesn't explain why request was sent | Check Initiator column in DevTools |
| Binary request bodies | Emitted as-is from HAR (base64) | Manual review required |

---

## 16. Testing Guide

### Test Files

| File | Purpose |
|---|---|
| `C:\Users\karrir\Downloads\perfmatrix.com.har` | Basic — 409 entries, 3 transactions with bookmarklet markers |
| `C:\Users\karrir\Downloads\WebTours.har` | Two-HAR test HAR1 — Web Tours travel booking app |
| `C:\Users\karrir\Downloads\WebTours1.har` | Two-HAR test HAR2 — same workflow, fresh session |
| `C:\Users\karrir\Downloads\opensource-demo.orangehrmlive.com.har` | OrangeHRM — Vue.js SPA, CSRF token as Vue prop |
| `C:\Users\karrir\Downloads\opensource-demo.orangehrmlive.com1111.har` | OrangeHRM HAR2 — for two-HAR mode |

### What to Verify

**VuGen-Recorder.html:**
- [ ] DROP HAR → format modal appears
- [ ] Transaction color bands shown correctly
- [ ] Domain panel checkboxes toggle requests
- [ ] Collapse/expand transaction groups
- [ ] Download ZIP → unzip → open in VuGen → script compiles
- [ ] NetLog JSON accepted (drop .json file)
- [ ] Auth badge shown for authenticated apps
- [ ] ServerHost appears as `{ServerHost}` in URLs

**VuGen-Script-Studio.html (two-HAR mode):**
- [ ] Drop HAR1 + HAR2 → Analyze → correlations panel shows detected values
- [ ] Params panel shows username, password (WebTours)
- [ ] Generated Action.c has `web_reg_save_param` BEFORE the source request
- [ ] Generated main.js has extractors on the correct response
- [ ] `{ParamName}` tokens appear in POST bodies where values differed
- [ ] Download ZIP → open in VuGen → replay works for at least 2 iterations

**VuGen-Script-Studio.html (single-HAR mode):**
- [ ] Drop 1 HAR → Analyze → some correlations detected (JWT, session cookies)
- [ ] CSRF token detected (if present in headers)
- [ ] SSO warning shown (if OAuth/OIDC redirects present)
- [ ] 3xx entries have redirect comment in generated script

### Browser Console Debugging

All state is in `S` — open DevTools console and inspect:
```javascript
S.entries1                // all entries
S.correlations            // detected correlations
S.candidates              // unresolved dynamic values
S.params                  // parameterized fields
S.auth                    // detected auth type
S.serverHost              // dominant hostname
S.harWarning              // any warning message
S.scripts.ac              // generated Action.c
S.scripts.mj              // generated main.js
```

To re-run script generation without re-uploading:
```javascript
buildScripts()            // Recorder
// Studio: re-click Analyze
```

---

*Last updated: 2026-03-06*
*Maintained by LRE Admin Team*
