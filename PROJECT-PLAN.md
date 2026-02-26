# VuGen Toolset — Complete Project Plan

> **Give this file to Claude at the start of every session.**
> It contains all architectural decisions, confirmed API syntax, file structures, and build plan.

---

## 1. Project Context

| Item | Detail |
|---|---|
| **Goal** | Build tools that replace VuGen's blocked recorder for LRE performance testing |
| **Root cause** | VuGen installs WinPcap/NPCAP kernel drivers — blocked by enterprise AV/org policy |
| **Users** | LRE users with no admin rights, no browser extensions, no Python |
| **Constraint** | Must work as a standalone `.html` file — no server, no installation |
| **Reference project** | `C:\Workspace\bruno-devweb-converter` — fully working Node.js tool (NOT this project) |
| **Workspace** | `C:\Workspace\Vugen-Recorder\` |

---

## 2. Tools Being Built

### Tool 1 — VuGen-Recorder.html (EXISTS — already deployed to GitHub)
**Status:** Complete and working. Deployed at `git@github.com:ScriptSavant1/vugenCustomRecorder.git`

**What it does:**
- User opens browser F12 → records HAR file
- Bookmarklets mark transaction START/END silently without leaving app page
- User drops HAR into tool → sees transaction table with domain filters
- Downloads generated scripts for both protocols

**Current output (Phase 1):** 5 loose files
- `Action.c`, `vuser_init.c`, `vuser_end.c`, `globals.h` (Web HTTP/HTML)
- `main.js` (DevWeb)

**Planned upgrade:** Replace loose files with complete ZIP project (see Phase 2 below)

---

### Tool 2 — VuGen-Script-Studio.html (TO BE BUILT)
**Status:** Not started

**What it does:**
- Takes 1 or 2 HAR files
- Runs correlation engine (two-HAR diff algorithm)
- Adds parameterization, response validation, exception handling
- Downloads complete VuGen project ZIP (ready to open in VuGen, no manual steps)

---

## 3. Confirmed VuGen Project File Structures

### Web HTTP/HTML Complete Project
```
ScriptName/
├── Action.c                          ← Main script (iterated)
├── vuser_init.c                      ← One-time init per vuser
├── vuser_end.c                       ← One-time cleanup per vuser
├── globals.h                         ← Standard includes
├── ScriptName.usr                    ← VuGen project metadata (INI format)
├── default.cfg                       ← Runtime settings (INI format)
├── default.usp                       ← Run logic profile (INI format)
├── ParameterFile.prm                 ← Parameter definitions (INI format)
├── collection_data.dat               ← CSV parameter values
├── ScriptUploadMetadata.xml          ← LRE upload manifest (XML)
└── data/
    ├── HostNames.dat                 ← Domain list
    ├── cross_correlation_parameters.txt  ← Contains: "Delimiter:="
    └── t1.inf, t2.inf, ...           ← Snapshot files (optional, can be empty)
```

### DevWeb Complete Project
```
ScriptName/
├── main.js                           ← SINGLE entry point (confirmed)
├── rts.yml                           ← Runtime settings (YAML)
├── scenario.yml                      ← Scenario config (YAML)
├── parameters.yml                    ← Parameter definitions (YAML)
├── collection_data.csv               ← CSV parameter values
├── tsconfig.json                     ← TypeScript compiler config
├── package.json                      ← Node package metadata
└── DevWebSdk.d.ts                    ← SDK type definitions (embed as string)
```

> **DevWeb note:** There is NO init.js / finalize.js. Everything lives in `main.js`.
> `load.initialize()`, `load.action()`, `load.finalize()` are ALL inside `main.js`.

---

## 4. Confirmed API Syntax — Web HTTP/HTML (C Language)

```c
// --- SCRIPT STRUCTURE ---
// vuser_init.c
vuser_init() { return 0; }

// globals.h
#ifndef _GLOBALS_H
#define _GLOBALS_H
#include "lrun.h"
#include "web_api.h"
#include "lrw_custom_body.h"
#endif

// Action.c
Action() {
    web_set_sockets_option("SSL_VERSION", "AUTO");
    // ... requests ...
    return 0;
}

// --- TRANSACTIONS ---
lr_start_transaction("T01_Login");
lr_end_transaction("T01_Login", LR_AUTO);    // LR_AUTO | LR_PASS | LR_FAIL | LR_STOP

// --- GET REQUEST ---
web_url("stepname",
    "URL=https://example.com/api/data",
    "Resource=0",
    "RecContentType=application/json",
    "Referer=",
    "Snapshot=t1.inf",
    "Mode=HTML",
    LAST);

// --- POST/PUT/DELETE REQUEST ---
web_custom_request("stepname",
    "URL=https://example.com/api/login",
    "Method=POST",
    "Resource=0",
    "RecContentType=application/json",
    "Referer=",
    "Snapshot=t2.inf",
    "Mode=HTML",
    "EncType=application/json",
    "Body={\"username\":\"{Username}\",\"password\":\"{Password}\"}",
    LAST);

// --- CORRELATION (register BEFORE the request that returns the value) ---
web_reg_save_param_json("AuthToken",
    "QueryParam=$.access_token",
    "NotFound=error",
    LAST);

web_reg_save_param_ex("AuthToken",
    "ParamName=AuthToken",
    "LB=\"access_token\":\"",
    "RB=\"",
    "Ord=1",
    "Search=Body",
    "NotFound=error",
    LAST);

web_reg_save_param_cookie("SessionID",
    "CookieName=JSESSIONID",
    LAST);

// Use extracted value in next request:
"Body={\"token\":\"{AuthToken}\"}"
"URL=https://example.com/api/{AuthToken}/profile"
web_add_header("Authorization", "Bearer {AuthToken}");

// --- RESPONSE VALIDATION (register BEFORE the request) ---
web_reg_find("Text=Welcome", "Search=Body", LAST);
web_reg_find("Text=Error", "Search=Body", "Fail=Found", LAST);

// --- OTHER FUNCTIONS ---
lr_think_time(3);
web_add_cookie("name=value; DOMAIN=example.com");
web_add_header("X-Header", "value");          // next request only
web_add_auto_header("X-Header", "value");     // all subsequent requests
lr_log_message("message");
lr_error_message("message %s", lr_eval_string("{ParamName}"));
lr_abort();
```

---

## 5. Confirmed API Syntax — DevWeb (JavaScript, main.js)
> **Source:** Official vendor SDK docs + 45 vendor-provided example scripts

```javascript
// NO import statement needed — load is a GLOBAL object

// --- SCRIPT STRUCTURE ---
load.initialize("Initialize", async function () {
    // Set global defaults once
    load.WebRequest.defaults.returnBody = false;
    load.WebRequest.defaults.headers = {
        "User-Agent": "Mozilla/5.0 ...",
        "Accept-Encoding": "gzip, deflate"
    };
    // Login and store session values
    load.global.authToken = "...";  // stored for all action iterations
});

load.action("Action", async function () {
    // Main workload — runs per iteration
    // Multiple load.action() blocks allowed, run in order
});

load.finalize("Finalize", async function () {
    // Logout, cleanup
});

// --- TRANSACTIONS ---
let T01 = new load.Transaction("T01_Login");
T01.start();
// ... requests ...
T01.stop();                                       // Pass (default)
T01.stop(load.TransactionStatus.Passed);          // Explicit pass
T01.stop(load.TransactionStatus.Failed);          // Fail

// --- HTTP REQUESTS (confirmed patterns from examples) ---

// GET (basic)
new load.WebRequest({
    id: 1,
    url: "https://example.com/",
    method: "GET",
    headers: { "Accept": "text/html" }
}).sendSync();

// GET with extraction
const response = new load.WebRequest({
    id: 2,
    url: "https://example.com/api/data",
    returnBody: true,
    extractors: [
        new load.JsonPathExtractor("AuthToken", "$.access_token"),
        new load.BoundaryExtractor("userId", "<userId>", "</userId>"),
        new load.TextCheckExtractor("loginCheck", { text: "Welcome", failOn: false })
    ]
}).sendSync();
load.global.authToken = response.extractors.AuthToken;  // store for later

// POST with JSON body
new load.WebRequest({
    id: 3,
    url: "https://example.com/api/login",
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ username: load.params.username }),
    returnBody: true
}).sendSync();

// POST with template literal (dynamic values inline)
new load.WebRequest({
    id: 4,
    url: "https://example.com/api/logout",
    method: "POST",
    body: `<Request><userId>${load.global.userId}</userId></Request>`
}).sendSync();

// GET with dynamic URL
new load.WebRequest({
    id: 5,
    url: `https://example.com/api/users/${load.global.userId}/cart`,
    headers: { "Authorization": `Basic ${load.global.authKey}` }
}).sendSync();

// --- CONDITIONAL LOGIC (confirmed from AdvantageOnlineShopping example) ---
if (response.extractors.isLoginSuccess) {
    T01.stop(load.TransactionStatus.Passed);
} else {
    T01.stop(load.TransactionStatus.Failed);
    return false;    // stops current iteration gracefully
}

// --- EXCEPTION HANDLING ---
try {
    const r = new load.WebRequest({ url: "...", returnBody: true }).sendSync();
    if (r.status !== 200) throw new Error(`Status: ${r.status}`);
    T01.stop(load.TransactionStatus.Passed);
} catch (e) {
    T01.stop(load.TransactionStatus.Failed);
    load.log(`T01 failed: ${e.message}`, load.LogLevel.error);
    load.exit(load.ExitType.stop, "Critical failure");   // ← lowercase 'stop'
}

// --- ERROR HANDLING on requests ---
new load.WebRequest({
    url: "...",
    handleHTTPError: (webResponse) => {
        if (webResponse.status === 404) return false;  // suppress, continue
        // return nothing = fail for other status codes
    }
}).sendSync();

// --- PARAMETERS ---
load.params.username           // read from parameters.yml + CSV
load.params.password

// --- GLOBAL VALUES (persist across all iterations within one vuser run) ---
load.global.authToken = "value";   // set in initialize, read in action

// --- THINK TIME ---
load.sleep(3);                     // sync pause (seconds)
load.thinkTime(2.5);               // also valid (alias)
load.thinkTime(2.5 + Math.random()); // with random variance

// --- LOGGING ---
load.log("message");
load.log(`User: [${load.params.userID}]`, load.LogLevel.info);
load.log("error detail", load.LogLevel.error);

// --- EXIT ---
load.exit(load.ExitType.stop, "reason");        // stop vuser
load.exit(load.ExitType.iteration, "reason");   // stop iteration only
return false;                                    // also stops iteration

// --- COOKIES ---
load.addCookies(new load.Cookie({ name: "sid", value: "abc123", domain: "example.com" }));
load.clearCookies();

// --- CONCURRENT REQUESTS ---
const p1 = new load.WebRequest({ url: "https://cdn.example.com/a.css" }).send();  // .send() = async
const p2 = new load.WebRequest({ url: "https://cdn.example.com/b.js" }).send();
await Promise.all([p1, p2]);   // run both simultaneously

// --- RENDEZVOUS ---
load.rendezvous("SyncPoint");  // no spaces in name
```

---

## 6. Reusable Code from bruno-devweb-converter

`C:\Workspace\bruno-devweb-converter` is a proven, working Node.js tool. The LOGIC can be ported to browser JavaScript.

| Component | Source File | What to Port | Notes |
|---|---|---|---|
| C string escaping | `webHttpScriptGenerator.js` | `escapeCString()`, `escapeCBodyString()` | Critical for POST bodies |
| Body chunking | `webHttpScriptGenerator.js` | String split for 1KB chunks | Long bodies must be split in C |
| URL encoding | `webHttpScriptGenerator.js` | `vuGenEncodeValue()` | Preserve `{varName}` placeholders |
| C script generation | `webHttpScriptGenerator.js` | `generateRequestBlock()` | web_url vs web_custom_request logic |
| DevWeb script generation | `advancedScriptGenerator.js` | `generateRequestCode()` | load.WebRequest structure |
| DevWeb variable handling | `advancedScriptGenerator.js` | `load.global` vs `load.params` classification | |
| Web HTTP/HTML config files | `webHttpMandatoryFilesGenerator.js` | .usr, default.cfg, default.usp, ParameterFile.prm, ScriptUploadMetadata.xml | Port exact INI/XML format |
| DevWeb config files | `mandatoryFilesGenerator.js` | rts.yml, scenario.yml, tsconfig.json | Port directly |

---

## 7. Two-HAR Correlation Algorithm (The Core of Script Studio)

### Why Two HARs?
Record the same scenario twice. Values that differ between HAR1 and HAR2 are **dynamic** (session tokens, IDs, CSRF tokens) — they must be correlated. Values that are the same are **static** — hardcode them.

### Step-by-Step Algorithm

```
STEP 1: PARSE BOTH HARS
  Filter noise from both (static assets, analytics, OPTIONS)
  Result: clean entry lists for HAR1 and HAR2

STEP 2: MATCH REQUEST PAIRS
  For each request in HAR1, find its counterpart in HAR2
  Match key: HTTP method + URL template
  URL template: replace /users/12345/ with /users/{id}/
               replace UUID-formatted values with {uuid}
  Skip unmatched requests (mark as "no correlation possible")

STEP 3: DIFF REQUEST VALUES
  For each matched pair (HAR1_req, HAR2_req), compare:
  a) URL query parameters: ?token=abc123 vs ?token=xyz789 → token is dynamic
  b) URL path segments: /users/12345/ vs /users/67890/ → 12345 is dynamic
  c) Request headers: Authorization, Cookie, X-CSRF-Token, X-Request-ID
  d) Request body (JSON): compare field by field
  e) Request body (form-encoded): compare field by field

  Collect: { fieldName, har1Value, har2Value, location: 'body'|'header'|'url' }

STEP 4: BACK-TRACE TO SOURCE
  For each dynamic value (har1Value):
  Search backwards through ALL previous responses in HAR1:
  - Response JSON: JSON.parse body, search recursively for value
  - Response HTML: search for value in body string
  - Response headers: check Set-Cookie, Location, X-* headers

  Result: { value, sourceRequestIndex, sourceLocation: 'json'|'body'|'cookie'|'header', jsonPath }

STEP 5: CONFIDENCE SCORING
  HIGH (auto-apply):
    - Value length >= 16 chars AND looks random (UUID, JWT, hex token)
    - Found in response body/header with clear source
    - Appears in multiple subsequent requests

  MEDIUM (show user, default checked):
    - Numeric ID in URL path (found in response)
    - Value in auth header

  LOW (show user, default unchecked):
    - Short numeric values (could be counts)
    - Timestamps / dates
    - Values NOT found in any response (may be client-generated)

STEP 6: GENERATE EXTRACTORS
  JSON response source → web_reg_save_param_json() / JsonPathExtractor
  HTML/text source    → web_reg_save_param_ex() (LB/RB) / BoundaryExtractor
  Cookie              → web_reg_save_param_cookie() / CookieExtractor
  Header              → web_reg_save_param_ex(Search=Headers) / BoundaryExtractor(scope=Headers)

STEP 7: REPLACE IN SCRIPT
  All occurrences of dynamic value in requests after extraction point:
  Web HTTP/HTML: {ParamName} in URL, Body strings, header values
  DevWeb: load.global.ParamName in url, body, headers
```

### Single-HAR Mode (no second HAR)
When only one HAR is provided:
- Pattern-based detection only (no diff)
- HIGH confidence: JWT tokens (eyJ...), UUID format values in headers/cookies
- HIGH confidence: session cookies (JSESSIONID, PHPSESSID, ASP.NET_SessionId)
- MEDIUM confidence: CSRF tokens (X-CSRF-Token header, csrf/xsrf in body)
- MEDIUM confidence: Authorization Bearer token
- LOW confidence: everything else

---

## 8. Parameterization Strategy

### What to Parameterize (from HAR)
| Pattern | How to Detect | Parameter Type |
|---|---|---|
| Server hostname | Extract base URL from all requests | Config (`nextValue: once`) |
| Login username | POST body field named `username`, `user`, `email`, `login` | Test data (per iteration) |
| Login password | POST body field named `password`, `pass`, `pwd` | Test data (per iteration) |
| Search term | URL query param `q`, `search`, `query` | Test data (per iteration) |
| Date/time values | ISO date format, unix timestamp | Generated (per iteration) |
| Environment ID | Repeated numeric/string in URL path across all requests | Config (`nextValue: once`) |

### Parameter File Formats

**Web HTTP/HTML — ParameterFile.prm:**
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
Type="File"
OriginalValue="testuser"
```

**DevWeb — parameters.yml:**
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

## 9. Response Validation Strategy

Auto-detect good validation points:
1. **Login response**: Check for user-specific field in response (`"userId"`, `"email"`, `"name"`)
2. **Page title**: Check for expected page title text
3. **Status field**: Check JSON field `"status": "success"` or `"result": "ok"`
4. **Negative check**: If response contains `"error"`, `"unauthorized"`, `"forbidden"` → fail

**Web HTTP/HTML (register BEFORE request):**
```c
web_reg_find("Text=Welcome", "Search=Body", LAST);
web_reg_find("Text=error", "Search=Body", "Fail=Found", LAST);
```

**DevWeb (check AFTER response):**
```javascript
if (!response.body.includes("Welcome")) {
    throw new Error("Login validation failed");
}
```

---

## 10. Exception Handling Strategy

**Web HTTP/HTML pattern:**
```c
lr_start_transaction("T01_Login");
web_reg_find("Text=access_token", "Search=Body", LAST);  // validation
web_custom_request("login", "URL=...", "Method=POST", "Body=...", LAST);
lr_end_transaction("T01_Login", LR_AUTO);
```

**DevWeb pattern:**
```javascript
const txn = new load.Transaction("T01_Login");
txn.start();
try {
    const response = new load.WebRequest({
        url: "https://example.com/api/login",
        method: "POST",
        body: JSON.stringify({ username: load.params.username }),
        returnBody: true
    }).sendSync();
    if (response.status !== 200) throw new Error(`Status: ${response.status}`);
    if (!response.body.includes("access_token")) throw new Error("Login validation failed");
    load.global.authToken = response.extractors.AuthToken;
    txn.stop(load.TransactionStatus.Passed);
} catch (e) {
    txn.stop(load.TransactionStatus.Failed);
    load.log(`T01_Login failed: ${e.message}`, load.LogLevel.error);
    load.exit(load.ExitType.Iteration, "Critical transaction failed");
}
```

---

## 11. Build Plan

### Phase 1 — Upgrade VuGen-Recorder.html (Quick Win)
**What:** Instead of downloading 5 loose files, download a complete ZIP project.

**Changes to VuGen-Recorder.html:**
- Add JSZip library (via CDN or inline)
- Add script name input field (user types e.g. "T01_Login_Script")
- Replace individual download buttons with "Download Web HTTP/HTML Project" / "Download DevWeb Project"
- Port generator logic from bruno-devweb-converter for config files
- ZIP structure: `ScriptName/Action.c`, `ScriptName/globals.h`, etc.

**Effort:** 1-2 days

---

### Phase 2 — VuGen-Script-Studio.html (New Tool)
**What:** Advanced tool with correlation, parameterization, validation, exception handling.

**UI Layout:**
```
┌─────────────────────────────────────────────────────────────────┐
│  VuGen Script Studio                                            │
├─────────────────────────────────────────────────────────────────┤
│  HAR File 1: [Drop here or Browse]   HAR File 2: [Optional]    │
│  Protocol: ○ Web HTTP/HTML  ○ DevWeb  ○ Both                   │
│  Script Name: [____________]    [⚡ Analyze & Generate]         │
├─────────────────────────────────────────────────────────────────┤
│  ANALYSIS RESULTS                                               │
│  ┌─────────────────┐ ┌──────────────────┐ ┌─────────────────┐  │
│  │ Correlations (4)│ │Parameters (3)    │ │Validations (5)  │  │
│  │ ☑ AuthToken     │ │ ☑ Username       │ │ ☑ T01 Welcome   │  │
│  │   HIGH ████     │ │ ☑ BaseURL        │ │ ☑ T02 dashboard │  │
│  │ ☑ CSRFToken     │ │ ☑ SearchTerm     │ │ ☑ Error check   │  │
│  │   HIGH ████     │ │                  │ │                  │  │
│  │ ☐ timestamp     │ │                  │ │                  │  │
│  │   LOW  ██       │ │                  │ │                  │  │
│  └─────────────────┘ └──────────────────┘ └─────────────────┘  │
├─────────────────────────────────────────────────────────────────┤
│  SCRIPT PREVIEW  [Action.c] [main.js]                           │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ (generated script)                                      │    │
│  └─────────────────────────────────────────────────────────┘    │
├─────────────────────────────────────────────────────────────────┤
│  [⬇ Download Web HTTP/HTML Project]  [⬇ Download DevWeb Project]│
└─────────────────────────────────────────────────────────────────┘
```

**Build Order:**
1. HAR parser (trivial — HAR is standard JSON)
2. Single-HAR pattern-based correlation (sessions, JWTs, cookies)
3. Basic parameterization (base URL, credentials)
4. Config file generators (port from bruno-devweb-converter)
5. ZIP download with complete project structure
6. Two-HAR diff correlation (the advanced algorithm)
7. Response validation generator
8. Exception handling wrapper

**Effort:** 5-7 days

---

## 12. Sample HAR Available for Testing

**File:** `C:\Users\karrir\Downloads\perfmatrix.com.har`
- **Size:** 17MB
- **Entries:** 409 total
- **Pages:** 3 (Home, TPS Calculator, Pacing Calculator)
- **Markers found:**
  - `https://start-t01_home.invalid/`
  - `https://end-t01_home.invalid/`
  - `https://start-t02_tps_calculator.invalid/`
  - `https://end-t02_tps_calculator.invalid/`
  - `https://start-t03_pcing_calculator.invalid/`
  - `https://end-t03_pcing_calculator.invalid/`
- **Bookmarklets working correctly** — markers detected as status 0, contain `.invalid` domain

---

## 13. Key Technical Decisions Made

| Decision | Choice | Reason |
|---|---|---|
| Tool delivery | Standalone `.html` file (no server) | Enterprise users: no admin rights, no Node.js |
| Correlation approach | Two-HAR diff algorithm | 100% deterministic, no AI needed, works offline |
| Script format | DevWeb uses `main.js` (NOT `action.js`) | Confirmed by user |
| DevWeb API | `load.` global object (NOT `lrd.`) | Confirmed from official documentation + 45 examples |
| Exit type syntax | `load.ExitType.stop` (lowercase) | Confirmed from AbortVuser example |
| Think time | `load.sleep()` AND `load.thinkTime()` both work | Confirmed from examples |
| Stop iteration | `return false` from action block | Confirmed from AdvantageOnlineShopping example |
| TextCheckExtractor | `failOn: false` = returns boolean; `failOn: true` = throws if found | Confirmed from examples |
| Global defaults | `load.WebRequest.defaults.headers/returnBody/handleHTTPError` | Confirmed from examples |
| ZIP in browser | JSZip library | Already used in VuGen-Recorder.html |
| Config files | Port logic from bruno-devweb-converter | Already validated, generates correct VuGen-compatible files |
| Body chunking | Split at 1000 chars in C strings | VuGen C compiler limitation |

---

## 14. GitHub Repository

- **Repo:** `git@github.com:ScriptSavant1/vugenCustomRecorder.git`
- **Branch:** `main`
- **Committed files:** VuGen-Recorder.html, README.md, USER-GUIDE.md, QUICK-REFERENCE.md, TROUBLESHOOTING.md, .gitignore
- **Next commit:** Phase 1 ZIP project download upgrade

---

## 15. Documentation Links (to be read when building)

When building DevWeb generators, read the official docs at these locations (provided by user in previous sessions — ask user to re-share if needed).
Key confirmed API points are captured in this document and in `memory/protocols.md`.

---

*Last updated: 2026-02-25*
*Generated by Claude Sonnet 4.6 based on full project analysis*
