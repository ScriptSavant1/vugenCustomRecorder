# FEATURES.md — VuGen Recorder & Script Studio

Comprehensive reference for all features and rules implemented in both tools.

---

## Table of Contents

1. [Overview](#overview)
2. [VuGen-Recorder.html](#vugen-recorderhtml)
   - [Architecture](#recorder-architecture)
   - [Input Handling](#recorder-input-handling)
   - [Processing Pipeline](#recorder-processing-pipeline)
   - [Transaction Marker Detection](#recorder-transaction-marker-detection)
   - [Authentication Detection](#recorder-authentication-detection)
   - [Server Host Parameterization](#recorder-server-host-parameterization)
   - [Hostname Substitution Coverage](#recorder-hostname-substitution-coverage)
   - [Header Generation](#recorder-header-generation)
   - [3xx Redirect Handling](#recorder-3xx-redirect-handling)
   - [Domain Filter Panel](#recorder-domain-filter-panel)
   - [Generated Output — DevWeb (main.js)](#recorder-generated-output--devweb-mainjs)
   - [Generated Output — Web HTTP/HTML (Action.c)](#recorder-generated-output--web-httphtml-actionc)
   - [ZIP Download Contents](#recorder-zip-download-contents)
3. [VuGen-Script-Studio.html](#vugen-script-studiohtml)
   - [Architecture](#studio-architecture)
   - [Input Handling](#studio-input-handling)
   - [Processing Pipeline](#studio-processing-pipeline)
   - [Authentication Detection](#studio-authentication-detection)
   - [Two-HAR Correlation Engine](#studio-two-har-correlation-engine)
   - [Extractor Type Selection](#studio-extractor-type-selection)
   - [Single-HAR Correlation](#studio-single-har-correlation)
   - [SSO/OAuth Detection](#studio-ssooauth-detection)
   - [Parameterization](#studio-parameterization)
   - [Generated Output — DevWeb (main.js)](#studio-generated-output--devweb-mainjs)
   - [Generated Output — Web HTTP/HTML (Action.c)](#studio-generated-output--web-httphtml-actionc)
   - [Correlation Placement Rules (C)](#studio-correlation-placement-rules-c)
   - [Parameter File Rules](#studio-parameter-file-rules)
   - [ZIP Download Contents](#studio-zip-download-contents)
4. [Shared Rules Reference](#shared-rules-reference)
   - [Transaction Naming Convention](#transaction-naming-convention)
   - [Header Skip Lists](#header-skip-lists)
   - [Protocol API Quick Reference](#protocol-api-quick-reference)

---

## Overview

Both tools are self-contained single HTML files. No build step, no package manager, no installation. Open directly in Chrome, Edge, or Firefox.

| Tool | File | Purpose | Lines |
|---|---|---|---|
| VuGen Recorder | `VuGen-Recorder.html` | Single HAR/NetLog → LoadRunner script (no correlation) | ~2500 |
| VuGen Script Studio | `VuGen-Script-Studio.html` | Two-HAR diff → correlated LoadRunner script | ~4500+ |

Both tools output scripts for two protocols:

- **Web HTTP/HTML** — C language (`Action.c`), used with the classic VuGen engine
- **DevWeb** — JavaScript (`main.js`), used with the modern DevWeb engine

---

## VuGen-Recorder.html

### Recorder Architecture

All application state lives in a single object `S`:

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

### Recorder Input Handling

- Drag-and-drop `.har` files (standard HAR format) onto the drop zone
- Drag-and-drop Chrome NetLog `.json` files exported from `chrome://net-export/`
- **NetLog source detection**: presence of `URL_REQUEST_START_JOB` event identifies NetLog source; handles Chrome 120+ where event types may be strings or numeric values
- After file drop: format modal prompts user to choose **Web HTTP/HTML**, **DevWeb**, or **Both**

### Recorder Processing Pipeline

```
readFile(file)
  → openFmtModal()             user picks output format
  → processHAR(har)
    OR processNetLog(netlog)
      → detectMarkers()        finds START/END bookmarklet markers
      → detectAuth()           scans for WWW-Authenticate / Bearer / SAML
      → detectCorporateAuth()  fallback for corporate Kerberos sites
      → detectServerHost()     dominant hostname → SERVER_HOST variable
      → buildDomainStats()     count + bytes per hostname
      → renderDomainPanel()    draws filter checkboxes
      → applyFilters()         sets entry.filtered flag
      → buildScripts()         calls genActionC() and/or genMainJS()
      → renderTable()          draws entry rows with transaction colors
      → showScript()           displays code in preview pane
```

### Recorder Transaction Marker Detection

Bookmarklet-based markers are injected as fake HTTP requests into the HAR during recording. Both directions of the marker name are supported.

**Supported marker URL patterns:**

| Direction | Start marker | End marker |
|---|---|---|
| Forward | `START-TxnName.invalid` | `END-TxnName.invalid` |
| Reverse | `TxnName-START.invalid` | `TxnName-END.invalid` |
| Underscore variants | `START_TxnName.invalid` | `END_TxnName.invalid` |

**Transformation rules:**
- Hyphens in transaction names are automatically converted to underscores
- `T01_` or `t01-` prefixes are stripped
- `SC01_XX_` is prepended, name is uppercased
- Example: `T01_login` → `SC01_01_LOGIN`

See [Transaction Naming Convention](#transaction-naming-convention) for the full rule.

### Recorder Authentication Detection

**`detectAuth(entries)`** — scans response headers:

| Signal | Detected type |
|---|---|
| `WWW-Authenticate: Negotiate` | `negotiate` (Kerberos/NTLM) |
| `WWW-Authenticate: NTLM` | `ntlm` |
| `WWW-Authenticate: Kerberos` | `kerberos` |
| `WWW-Authenticate: Digest` | `digest` |
| `WWW-Authenticate: Basic` | `basic` |
| `Authorization: Bearer ...` in requests | `bearer` |
| SAML assertion in POST body | `saml` |

**`detectCorporateAuth(entries, currentAuth)`** — fallback for corporate Kerberos sites where Chrome omits Negotiate headers from HAR:

- Checks if any request URL hostname is NOT a public TLD (`.com`, `.org`, `.net`, `.io`, etc.) and is NOT localhost or an IP address
- If a corporate-internal hostname is found, returns `{type:'negotiate'}` for that host
- Also catches Azure AD hostnames (`login.microsoftonline.com`, etc.)

**Generated auth code:**

| Protocol | Output |
|---|---|
| Web HTTP/HTML | `web_set_user("username","password","realm");` at start of Action |
| DevWeb | `load.setUserCredentials({username: load.params.AuthUsername, password: load.params.AuthPassword, domain: load.params.AuthDomain, type: load.CredentialType.kerberos})` inside action |

Auth parameters `AuthUsername`, `AuthPassword`, `AuthDomain` are added to the CSV and parameter file when auth is detected.

### Recorder Server Host Parameterization

**`detectServerHost(entries)`** — identifies the dominant hostname (the one with the most requests) as `SERVER_HOST`.

**`buildHdrHostMap(entries, primaryHost)`** — scans ALL request header values for URL patterns to find secondary hostnames, generating `SERVER_HOST1`, `SERVER_HOST2`, etc.

**Generated code:**

| Protocol | Output |
|---|---|
| Web HTTP/HTML (C) | `lr_save_string("hostname.com", "ServerHost");` at start of Action; URLs use `{ServerHost}` |
| DevWeb (JS) | `let SERVER_HOST = 'https://hostname.com';` at module level (NOT from `load.params`); URLs use `${SERVER_HOST}` |

`ServerHost` is **NOT** added to `parameters.yml` or `collection_data.csv`.

### Recorder Hostname Substitution Coverage

Hostnames are substituted in all of the following locations:

| Location | DevWeb | Web HTTP/HTML (C) |
|---|---|---|
| Request URLs (`url:` / `URL=`) | Yes | Yes |
| Request header values (referer, origin, etc.) | Yes — `subHdrValMj()` | Yes — `subHdrValC()` |
| Query string values | Yes — `subHdrValMj()` | Partial |
| POST body form field values (decoded) | Yes — `subHdrValMj()` | Not supported |
| POST body JSON/raw text | Yes — `subRawMj()` | Not supported |

Both raw (`https://hostname`) and URL-encoded (`https%3A%2F%2Fhostname`) forms are handled.

Note: The C generator uses `BodyBinary=` with hex encoding, so in-body hostname substitution is not supported for the Recorder C generator. Use VuGen-Script-Studio for full body substitution.

### Recorder Header Generation

**Global auto-headers (`autoHdrs`):**

- **Key-presence threshold**: if ≥80% of requests include a header key, the most common value for that key becomes the global auto-header
- **Force-global exceptions**: `user-agent` and `accept-language` are always emitted as session-wide globals regardless of percentage

**Per-request headers:**

- An override is emitted ONLY when the request's value differs from the global auto-header value
- If the value matches the global, no per-request header is emitted (deduplication)

**Always suppressed:**

| Header | Reason |
|---|---|
| `Accept: */*` | VuGen/DevWeb default — never emit |
| `Authorization` (non-Bearer) | `web_set_user` / `load.setUserCredentials` handles it |

**Skip lists (never emitted, either protocol):**

`referer`, `content-length`, `host`, `connection`, `keep-alive`, `accept-encoding`, `expect`, `priority`, `sec-*`, `cache-control`, `cookie`, `cookie2`, `if-*` headers

**Generated code:**

| Scope | Web HTTP/HTML | DevWeb |
|---|---|---|
| Global | `web_add_auto_header("Key", "Value");` | `load.WebRequest.defaults.headers = {...}` |
| Per-request | `web_add_header("Key", "Value");` | `headers: { Key: "Value" }` on WebRequest |

### Recorder 3xx Redirect Handling

**`buildAutoFollowMap()`** — identifies redirect chains and marks them for omission from script output. VuGen and DevWeb both automatically follow redirects, so emitting the redirect target as a separate request is incorrect.

**Comments emitted:**

- On the originating request (before the redirect):
  ```
  // Note: VuGen auto-follows N redirect(s) after this request — extractors scan the final response
  ```
- On the omitted redirect target entry:
  ```
  // HTTP 3xx redirect — VuGen follows subsequent redirects automatically
  ```

### Recorder Domain Filter Panel

- Displays all domains found in the HAR with request count and transfer size
- Checkboxes allow individual domains to be included or excluded
- Entries from excluded domains are skipped during script generation
- Panels are resizable (drag the divider)
- Implementation note: checkboxes use `onclick="event.stopPropagation(); toggleDomain(domain, this.checked)"` on the checkbox and `<div onclick="onDpClick(event, domain)">` on the row to avoid double-fire bugs

### Recorder Generated Output — DevWeb (main.js)

**Module-level declarations (in order):**

```javascript
let SERVER_HOST = 'https://detected-hostname.com';  // hardcoded, NOT load.params

load.WebRequest.defaults.returnBody = false;
load.WebRequest.defaults.downloadHtmlStaticResources = true;
load.WebRequest.defaults.headers = { /* global auto-headers */ };

let TS01 = new load.Transaction("SC01_01_NAME");
// ... more transaction declarations ...

let think_time = 1;
```

**Block structure:**

```javascript
load.initialize("initialize", async function() {
  load.log("Initializing script", load.LogLevel.debug);
});

load.action("action", async function() {
  TS01.start();
  await new load.WebRequest({ ... }).send();
  // ... more requests ...
  TS01.stop();
  load.sleep(think_time);
  // ... next transaction ...
});
```

**WebRequest rules:**

| Rule | Detail |
|---|---|
| Keys unquoted | `id:`, `url:`, `method:`, `headers:`, `body:`, `queryString:`, `returnBody:`, `extractors:` |
| URL splitting | `url: "base"` + `queryString: { key: "value" }` object when query params are present |
| JSON bodies | Pretty-printed as JS object literal (4-space indent) |
| Form bodies | `body: { key: "value" }` object literal with decoded key-value pairs |
| Think time | `load.sleep(think_time)` only after `TS.stop()` — never per-request |
| `TS.stop()` | No arguments — no `TransactionStatus.Passed`/`Failed` |
| Log level | `load.LogLevel.debug` (never `info`) for all log calls |
| Log messages | No emoji prefixes |

### Recorder Generated Output — Web HTTP/HTML (Action.c)

**Action start:**
```c
lr_save_string("hostname.com", "ServerHost");
web_set_user("username", "password", "realm");  // if auth detected
```

**Request functions:**

| Method | Function |
|---|---|
| GET, HEAD | `web_url()` |
| POST, PUT, DELETE, PATCH, all others | `web_custom_request()` |

**`web_custom_request()` rules:**

- Always uses `BodyBinary=` — NEVER `Body=` (JSON braces break the VuGen parser)
- Non-ASCII bytes are hex-encoded via `escBodyBinary()`
- Long bodies split at ~200 chars using adjacent string literal concatenation; split points are escape-sequence-aware to avoid splitting in the middle of an escape sequence

**Transaction structure:**
```c
lr_start_transaction("SC01_01_NAME");
web_url(/* ... */);
// ... more requests ...
lr_end_transaction("SC01_01_NAME", LR_AUTO);
lr_think_time(3);
```

- `LR_AUTO` only — never `LR_PASS` or `LR_FAIL`
- `lr_think_time()` only after `lr_end_transaction()`, never per-request

### Recorder ZIP Download Contents

**Web HTTP/HTML ZIP:**

| File | Description |
|---|---|
| `Action.c` | Generated script |
| `vuser_init.c` | Stub init file |
| `vuser_end.c` | Stub end file |
| `globals.h` | Stub globals header |
| `ScriptName.usr` | VuGen project file |
| `default.cfg` | Default run configuration |
| `default.usp` | User script properties |
| `ParameterFile.prm` | Parameter definitions (populated when params detected) |
| `collection_data.dat` | Parameter CSV data |
| `ScriptUploadMetadata.xml` | LRE upload metadata |
| `data/` | Data folder |

**DevWeb ZIP (15+ files):**

| File | Description |
|---|---|
| `main.js` | Generated DevWeb script |
| `rts.yml` | Complete runtime settings (includes grpc, vts, encryption, flow, openTelemetry sections) |
| `scenario.yml` | Scenario configuration |
| `parameters.yml` | Parameter definitions |
| `tsconfig.json` | TypeScript configuration |
| `collection_data.csv` | Parameter CSV data |
| `[name].usr` | VuGen project file |
| `default.cfg` | Default run configuration |
| `default.usp` | User script properties |
| `ScriptUploadMetadata.xml` | LRE upload metadata |
| `Action.c` | Stub (required by VuGen project structure) |
| `vuser_init.c` | Stub |
| `vuser_end.c` | Stub |
| `Bookmarks.xml` | VuGen bookmarks file |
| `Breakpoints.xml` | VuGen breakpoints file |
| `UserTasks.xml` | VuGen user tasks file |
| `DevWebSdk.d.ts` | TypeScript SDK definitions (116KB, embedded directly in HTML) |

---

## VuGen-Script-Studio.html

### Studio Architecture

All application state lives in a single object `S`:

```
S = {
  har1, har2,           // raw parsed HAR objects
  entries1, entries2,   // normalized entry arrays
  txns,                 // detected transactions
  correlations,         // correlation results from two-HAR diff
  candidates,           // candidate values identified for correlation
  params,               // detected parameter substitutions
  harWarning,           // warning message if HAR has issues
  scripts,              // generated script strings
  format,               // 'webhttp' | 'devweb' | 'both'
  mode,                 // 'two-har' | 'single-har'
  tab,                  // active preview tab
  auth,                 // {type, realm, host}
  serverHost,           // {host, proto, prefix}
  isNetLog1, isNetLog2  // true when source is NetLog
}
```

### Studio Input Handling

- Drop zone accepts two HAR files simultaneously, or one file for single-HAR mode
- Also accepts Chrome NetLog `.json` files (same detection as Recorder)
- UI progresses through three phases:
  1. **Upload** — drop zone visible
  2. **Processing** — spinner shown while analysis runs
  3. **Results** — preview panels and download button

### Studio Processing Pipeline

```
analyze()
  → detectAuth()
  → detectCorporateAuth()
  → detectServerHost()
  → applyFilters()
  → twoHarCorrelate(entries1, entries2)  (if two HARs provided)
    OR singleHarCorrelate(entries1)       (if one HAR provided)
  → detectParams(entries, correlations)
  → buildScripts()
  → SSO warning check
```

### Studio Authentication Detection

All detection from the Recorder applies, plus the **4-signal Windows-auth detection** (used after SSO is found):

| Signal | Description |
|---|---|
| 1 | `Negotiate` or `NTLM` headers present in HAR request/response headers |
| 2 | URL contains `/adfs/` or `/wsfed` path |
| 3 | Corporate non-public TLD in hostname (`.mde`, `.local`, `.corp`, `.internal`) → `corporateInternalSso=true` |
| 4 | Azure AD hostnames (`login.microsoftonline.com`, etc.) |

Any one signal sets `S.auth = {type:'negotiate'}`, which triggers `web_set_user()` + RTS entries + auth params in both generators. This catches corporate IdPs (PingFederate, Shibboleth) even when Chrome omits Negotiate headers from the HAR.

`AuthDomain` is added to `S.params`, CSV, and `parameters.yml` when Windows auth is detected.

### Studio Two-HAR Correlation Engine

**`twoHarCorrelate(e1, e2)`** — deterministic diff engine:

1. Pairs requests from HAR1 and HAR2 by matching on `method + normalizedURL`
2. For each pair, diffs all locations where dynamic values can appear
3. Back-traces each changed value to find where it originated in a previous response
4. Generates extractors and substitution tokens

**`isDynamic(value)`** — detects whether a value is dynamic (not safe to hardcode):

| Pattern | Description |
|---|---|
| JWT | `eyJ...` base64 header |
| UUID | `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx` |
| hex32 / hex64 / hex16 | Hexadecimal strings of 16, 32, or 64 chars |
| longToken | Any opaque token ≥32 chars |
| midToken | Tokens 12–35 chars (session-like) |
| numericId | Pure numeric strings ≥15 digits |
| compound | Multiple dynamic segments joined by delimiters |

**Diff locations checked:**

| Location key | What is diffed |
|---|---|
| `url_path` | Full URL path |
| `url_path_seg` | Individual path segments |
| `query` | Query string parameter values |
| `header` | Request header values |
| `cookie` | Cookie values |
| `body_form` | URL-encoded form field values |
| `body_json` | JSON body field values |
| `body_xml` | XML body element/attribute values |

**`findValueInResponse(entry, value)`** — searches a response for a given value in this priority order:

1. `Set-Cookie` response headers (cookie value extraction)
2. Other response headers including `Location` header for 3xx responses (tight LB/RB extraction, path-segment and query-param aware)
3. JSON body (`$.path` detection)
4. XML body (element and attribute values)
5. HTML hidden input (`<input type="hidden" name="x" value="...">`)
6. Vue/React prop assignments
7. `data-*` attributes
8. JavaScript variable assignments
9. `<meta>` tag content attributes
10. Boundary fallback (surrounding chars as LB/RB)

### Studio Extractor Type Selection

Extractor type is chosen based on where the value was found in the response. Both protocol outputs are shown.

| Extractor type | Trigger condition | DevWeb | Web HTTP/HTML (C) |
|---|---|---|---|
| `cookie` | Found in `Set-Cookie` header | `CookieExtractor` | `web_reg_save_param(Search=Headers, LB=name=, RB=;)` |
| `boundary_header` | Found in other response headers | `BoundaryExtractor({scope:Headers})` | `web_reg_save_param(Search=Headers, LB=..., RB=...)` |
| `jsonpath` | Found in JSON response body | `JsonPathExtractor("name","$.path")` | `web_reg_save_param_json(QueryString=$.path)` |
| `html` | Found in HTML hidden input, meta, data-* | `HtmlExtractor("name","selector","attr")` | `web_reg_save_param(LB=name="x" value=", RB=")` |
| `boundary` | Found in body (non-JSON) | `BoundaryExtractor({scope:Body})` | `web_reg_save_param(LB=..., RB=...)` |
| `generate` | Value is client-generated (UUID, random) | `load.utils.uuid()` / `load.utils.randomString(n,{hex:true})` | `lr_param_sprintf` |

**Critical rule:** `BoundaryExtractor` ALWAYS uses the options object form with explicit `scope`. The 3-argument positional form is never used because it scans both headers and body ambiguously.

```javascript
// CORRECT
new load.BoundaryExtractor("name", { leftBoundary: "LB=", rightBoundary: "=RB", scope: load.ExtractorScope.Body })

// WRONG — never use this
new load.BoundaryExtractor("name", "LB=", "=RB")
```

**HtmlExtractor LB/RB derivation:**

| Selector type | LB | RB |
|---|---|---|
| `input[name='x']` | `name="x" value="` | `"` |
| `meta[name='x']` | `name="x" content="` | `"` |
| `[data-x]` | `data-x="` | `"` |

### Studio Single-HAR Correlation

**`singleHarCorrelate(entries)`** — used when only one HAR is provided:

- **Pattern detection**: scans for JWT tokens, CSRF headers, session cookies by known naming patterns
- **Redirect chain detection**: follows 3xx → Location → next URL to detect dynamic path segments passed through redirect chains
- **SSO/OAuth chain**: `findValueInResponse()` extracts values from `Location` headers in 3xx responses using tight LB/RB boundary detection

### Studio SSO/OAuth Detection

**`SSO_URL_PATTERN`** — regex that matches SSO-related URL patterns:

- `oauth2`, `oidc`, `/as/` (PingFederate authorization server)
- `/sso/`, `/saml/`
- Okta hostnames
- ADFS endpoints
- Keycloak endpoints

Detection is applied to both request URLs and `Location` header values in responses.

When SSO is detected:
- A warning banner is shown in the Results UI
- The 4-signal Windows-auth check is applied
- Appropriate `web_set_user()` or `load.setUserCredentials()` calls are generated

### Studio Parameterization

**`detectParams(entries, correlations)`** — scans POST body fields for known parameter patterns.

**Detected parameter keys (`PARAM_KEYS_MAP`):**

| Category | Keys |
|---|---|
| Credentials | `username`, `password` |
| Payment | `CardType`, `CardNumber`, `ExpiryDate` |
| Personal | `FirstName`, `LastName` |
| Address | `BillingAddress1`, `BillingAddress2`, `City`, `State`, `ZipCode`, `Country` |
| Travel | `from`, `to`, `date`, `returnDate` |

Each detected param produces an entry in `S.params[]` with: `csvKey`, `value`, `usages[]`.

Detected params are written to `collection_data.csv` / `collection_data.dat` and `parameters.yml` / `ParameterFile.prm` in the ZIP.

### Studio Generated Output — DevWeb (main.js)

All rules from [Recorder Generated Output — DevWeb (main.js)](#recorder-generated-output--devweb-mainjs) apply, plus the following Studio-specific additions:

**Correlation globals declared at module level:**
```javascript
load.global.TOKEN_NAME = "";
```

**Extractors on the request that RETURNS the value:**
```javascript
await new load.WebRequest({
  url: `${SERVER_HOST}/api/login`,
  method: "POST",
  // ...
  extractors: [
    new load.JsonPathExtractor("TOKEN_NAME", "$.sessionToken")
  ]
}).send();
```

**Correlation token usage in subsequent requests:**
```javascript
await new load.WebRequest({
  url: `${SERVER_HOST}/api/data`,
  headers: {
    "Authorization": `Bearer ${load.global.TOKEN_NAME}`
  }
  // ...
}).send();
```

**Windows auth:**
```javascript
load.setUserCredentials({
  username: load.params.AuthUsername,
  password: load.params.AuthPassword,
  domain: load.params.AuthDomain,
  type: load.CredentialType.kerberos
});
```

Note: Exception handling with `try/catch` per action and `_activeTxn` tracking was removed in the cleanup phase. The generated output is clean with no boilerplate error-trapping scaffolding.

### Studio Generated Output — Web HTTP/HTML (Action.c)

All rules from [Recorder Generated Output — Web HTTP/HTML (Action.c)](#recorder-generated-output--web-httphtml-actionc) apply, plus the following Studio-specific additions:

**Correlation functions — placement rule:** All `web_reg_save_param*` calls must be placed **BEFORE** the request that returns the value (not before the request that uses it).

**Correlation function selection:**

| Source | Function | Notes |
|---|---|---|
| Cookie | `web_reg_save_param("Name","LB=name=","RB=;","Search=Headers",LAST)` | No `web_reg_save_param_cookie` — it does not exist |
| Response header | `web_reg_save_param("Name","LB=...","RB=...","Search=Headers",LAST)` | |
| JSON body | `web_reg_save_param_json("Name","QueryString=$.path","Ord=1",LAST)` | Use `QueryString=` not `QueryParam=` |
| HTML body | `web_reg_save_param("Name","LB=...","RB=...",LAST)` | NEVER `web_reg_save_param_ex` for body/HTML |
| Body boundary | `web_reg_save_param("Name","LB=...","RB=...",LAST)` | Default search is body |

**Correlation token usage:**
```c
web_url("MyRequest",
  "URL=http://{ServerHost}/api/data?token={TOKEN_NAME}",
  LAST);
```

**Form POST bodies** — use `web_submit_data` with `ITEMDATA`:
```c
web_submit_data("form",
  "Action=http://{ServerHost}/login",
  "Method=POST",
  ITEMDATA,
  "Name=username", "Value={AuthUsername}", ENDITEM,
  "Name=password", "Value={AuthPassword}", ENDITEM,
  LAST);
```

**JSON/XML/PUT/DELETE/PATCH** — use `web_custom_request` with `BodyBinary=`.

Hostname substitution is applied to decoded form field values before emission.

### Studio Correlation Placement Rules (C)

```
Request A returns token  ← web_reg_save_param goes HERE (before request A)
Request B uses token     ← {TOKEN_NAME} substituted in URL/body/headers
```

- All `web_reg_save_param*` calls are placed immediately before the request that **returns** the value
- Never place them before the request that **uses** the value
- Correlation values are referenced as `{ParamName}` tokens in all subsequent requests

### Studio Parameter File Rules

These rules are critical — VuGen will reject the script if they are violated.

| Rule | Correct | Wrong |
|---|---|---|
| Section header format | `[parameter:CsvKey]` — no leading spaces | `  [parameter:CsvKey]` |
| Type attribute | `Type="Table"` | `Type="File"` — VuGen rejects this |
| Column name | `ColumnName="CsvKey"` — must match CSV header exactly | Mismatched column name |
| .usr reference | `ParameterFile=ParameterFile.prm` (not empty when params exist) | Empty value when params exist |

### Studio ZIP Download Contents

Same as Recorder ZIP contents with the addition of correlated extractors, parameter substitutions, and populated CSV data files. See [Recorder ZIP Download Contents](#recorder-zip-download-contents) for the full file list.

---

## Shared Rules Reference

### Transaction Naming Convention

Applies to both tools and both output protocols.

**Transformation steps (applied in order):**

1. Extract the raw name from the bookmarklet marker
2. Convert hyphens to underscores
3. Strip leading `T01_` or `t01-` prefix (case-insensitive)
4. Prepend `SC01_XX_` where `XX` is the zero-padded transaction sequence number (01, 02, 03, ...)
5. Uppercase the name portion

**Examples:**

| Raw marker name | Generated transaction name |
|---|---|
| `T01_login` | `SC01_01_LOGIN` |
| `t01-SearchFlights` | `SC01_01_SEARCHFLIGHTS` |
| `checkout` | `SC01_01_CHECKOUT` |
| `T01_view-cart` | `SC01_01_VIEW_CART` |

### Header Skip Lists

Headers in these lists are never emitted by either generator:

**`SKIP_HDR_AC` (Web HTTP/HTML C generator):**
`referer`, `content-length`, `host`, `connection`, `keep-alive`, `accept-encoding`, `expect`, `priority`, `sec-*` (all sec- prefixed headers), `cache-control`, `cookie`, `cookie2`, `if-*` (all if- prefixed headers)

**`SKIP_HDRS` (DevWeb JS generator):**
Same list as above.

### Protocol API Quick Reference

**Web HTTP/HTML — Critical rules:**

| Rule | Detail |
|---|---|
| GET/HEAD requests | `web_url()` |
| All other methods | `web_custom_request()` |
| Body in custom request | Always `BodyBinary=` — never `Body=` |
| JSON extraction | `web_reg_save_param_json("Name","QueryString=$.path","Ord=1",LAST)` |
| Cookie extraction | `web_reg_save_param` with `Search=Headers`, `LB=name=`, `RB=;` |
| Body/HTML extraction | `web_reg_save_param("Name","LB=...","RB=...",LAST)` — never `web_reg_save_param_ex` |
| Non-existent function | `web_reg_save_param_cookie` — does NOT exist |
| Transaction result | `LR_AUTO` — never `LR_PASS` or `LR_FAIL` |
| Think time placement | After `lr_end_transaction()` only |

**DevWeb — Critical rules:**

| Rule | Detail |
|---|---|
| `load` object | Global — no import statement needed |
| Entry file | `main.js` only — no separate init/finalize files |
| Request sending | `await new load.WebRequest({...}).send()` — never `sendSync()` |
| Read response body | Requires `returnBody: true` on the request |
| Exit types | Lowercase: `load.ExitType.stop`, `load.ExitType.iteration` |
| Think time | `load.sleep(n)` or `load.thinkTime(n)` — both work |
| Stop iteration | `return false` from the action function |
| BoundaryExtractor | Always use options object form with explicit `scope` |
| Log level | `load.LogLevel.debug` — never `load.LogLevel.info` |
