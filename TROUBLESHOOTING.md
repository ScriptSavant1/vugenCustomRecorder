# LRE HAR Script Generator — Troubleshooting Guide

---

# SECTION 1 — Bookmarklet Issues (Tool 1)

---

### ❓ I cannot drag the bookmarklet buttons to my Bookmarks Bar

**First — make sure the Bookmarks Bar is visible:**
- **Chrome/Edge:** Press `Ctrl+Shift+B`
- **Firefox:** Press `Ctrl+Shift+B`

If dragging still doesn't work, add them manually:
1. Right-click the **▶ START Transaction** button in the tool
2. Select **"Bookmark this link"** (Chrome/Edge) or **"Bookmark This Link"** (Firefox)
3. Choose **Bookmarks Bar** as the folder → click Save
4. Repeat for the **■ END Transaction** button

---

### ❓ I clicked the START or END bookmark but nothing happened — no prompt appeared at all

**Most likely cause: you are on a New Tab page or a browser-internal page.**

Chrome and Edge silently block `javascript:` bookmarklets on any `chrome://` page (including the New Tab page, Settings, etc.). There is no error — the bookmark just does nothing.

**Fix:**
1. Navigate to your application URL first (e.g. `https://myapp.company.com`)
2. Open F12 **after** you are on your application page
3. Now click the bookmark — the name prompt will appear

> ⚠️ You must be on a real `http://` or `https://` page for bookmarklets to work.

---

### ❓ I clicked the START or END bookmark but nothing visible happened *(after the prompt)*

**This is correct behaviour.** The bookmarklet fires a silent background request. You stay on your application page. No popup, no page change, no visible error after you type the name and click OK.

**To verify it worked:** Look in the Network tab — you should see a new row with `START-T01_Name.invalid` (status 0 or failed).

---

### ❓ The Network tab shows no marker entry after clicking the bookmark

| Cause | Fix |
|---|---|
| You were on the New Tab page (chrome://newtab) | Navigate to your application first, then click the bookmark |
| Developer Tools was closed when you clicked | Press F12 first, then click the bookmark |
| You clicked Cancel in the name prompt | Click the bookmark again and type a name before OK |
| Browser blocked the background request | Try Chrome or Edge instead of Firefox |

---

### ❓ My organisation blocks `javascript:` bookmarks

**Use the Address Bar Method instead:**
1. Navigate to the start of your transaction in your application
2. Click the address bar and type: `https://START-T01-Login.invalid` → press Enter
3. An error page appears — this is expected
4. Press the **Back button** to return to your application
5. Perform your transaction steps
6. Click the address bar again and type: `https://END-T01-Login.invalid` → press Enter
7. Press **Back** and continue with the next transaction

---

# SECTION 2 — Recording Issues

---

### ❓ My Network tab is empty when I open it

**Cause:** The Network tab only captures requests made while it is open.

**Fix:**
1. Press F12 → click the **Network** tab
2. Click 🚫 to clear any entries
3. Now start your recording (do NOT reopen/close DevTools)

---

### ❓ I accidentally closed Developer Tools during recording

All captured data is lost. Start over:
1. Press F12 to reopen Developer Tools
2. Click 🚫 to clear entries
3. Begin your recording from the very first START marker

---

### ❓ My application opens a popup window or new tab during the journey — those requests are missing

**Cause:** Developer Tools only captures traffic from the tab it is open in. Requests made in a popup window or new tab are not recorded in the original tab's Network view.

**Recommended — use `chrome://net-export/` (captures ALL tabs simultaneously):**

Both **VuGen HAR Script Generator** and **VuGen Script Studio** can now read Chrome NetLog `.json` files directly — no conversion needed.

1. In Chrome/Edge, open a new tab and navigate to `chrome://net-export/`
2. Click **"Start Logging to Disk"** → choose a file name → click **Save**
3. Open your application in a **separate new tab** (the net-export tab stays running in the background)
4. Use the **▶ START Transaction** and **■ END Transaction** bookmarklets as normal — they are captured by NetLog regardless of which tab fires them
5. Complete your entire user journey (including any popup windows or new tabs the app opens)
6. Switch back to the `chrome://net-export/` tab and click **"Stop Logging"**
7. Drop the saved `.json` NetLog file into the tool just as you would a `.har` file

> **Important:** NetLog does not capture POST request bodies. The generated script will contain `// TODO: POST body not available in NetLog` comments for those requests. Add the body content manually by checking what was sent in the application (use browser DevTools in the same session for reference, or check API documentation).

**Transaction markers in NetLog:** The bookmarklet fires `fetch("https://START-TxnName.invalid")` which operates at the browser network-stack level — captured by NetLog from **any tab**, so transaction names are preserved correctly.

**Alternative — record each tab separately (if net-export is unavailable):**
1. In your **main browser tab**, open F12 → Network tab → clear entries
2. Start your journey in the main tab
3. When the popup opens, press F12 in that window too and clear entries
4. Export HAR from each tab separately and load them into **Script Studio** as Recording 1 and Recording 2

---

### ❓ My start/end markers are not being detected by the tool

Check the marker URL format. The tool requires exactly this pattern:

```
✅  https://START-T01-Login.invalid       (hyphen between START and name)
✅  https://END-T01-Login.invalid
✅  https://START-T01_Login.invalid       (underscore also works)

❌  http://START-T01-Login.invalid        (must be https)
❌  https://START T01 Login.invalid       (no spaces)
❌  https://STARTT01Login.invalid         (missing hyphen after START)
❌  https://START-T01-Login.com           (must end in .invalid)
```

> Bookmarklets format the name automatically — this mostly affects the manual address bar method.

---

### ❓ The Network tab shows thousands of requests — is that normal?

Yes — modern websites make many background requests. The tool automatically filters out images, fonts, CSS, analytics, and other noise. When you import the HAR, most of this will be hidden.

---

### ❓ I forgot to put markers for one of my transactions

**Option 1 — Re-record** (usually takes just a few minutes — quickest option)

**Option 2 — Use Select Mode in Tool 1:**
1. Load the HAR into `VuGen-Recorder.html`
2. Click **"☑ Select Mode"** → it turns blue
3. Click each request row that belongs to the missing transaction
4. Click **"+ Create Transaction"** → type the name → click Create

---

# SECTION 3 — Tool 1 Issues (VuGen-Recorder.html)

---

### ❓ The tool doesn't open when I double-click it

| Cause | Fix |
|---|---|
| No default browser set | Right-click the file → Open with → Chrome or Edge |
| File is on a restricted network drive | Copy the file to your Desktop and open it from there |
| Browser security warning | In Chrome: click the shield icon in the address bar → Allow |

---

### ❓ I dropped the HAR file but nothing happened

1. Confirm the file ends in `.har` (not `.zip`, `.txt`, or anything else)
2. Try the **Browse** button instead of drag-and-drop
3. Confirm the file is not open in another program
4. Try a different browser (Chrome usually works best)

---

### ❓ The tool loaded the file but shows "No entries" or an empty table

**Cause:** The HAR file is empty, corrupted, or contains no valid HTTP requests.

**Fix:**
1. Re-record your session
2. In DevTools Network tab, right-click and select **"Save all as HAR with content"** (not just "Save as HAR")
3. Make sure you clicked inside the **Network tab** (not Console or Sources) when saving

---

### ❓ All filters are on but there are still too many requests

Use the domain panel on the left side — uncheck any domains you do not want included.

---

### ❓ The request table has many rows — it is hard to see the overall structure

Use the **collapse** feature to hide the requests inside each transaction:

1. Click any coloured **▼ START** header row to collapse that transaction — all its requests hide and the row shows **▶ START** instead
2. Click **▶ START** again to expand it
3. Use **"⊖ Collapse All"** in the toolbar to collapse every transaction at once
4. Use **"⊕ Expand All"** to restore everything

The collapse state is remembered while you adjust filters — transactions you collapsed stay collapsed.

---

### ❓ The generated Action.c has no request headers (no Accept, User-Agent, etc.)

**This should no longer happen** — the tool now automatically generates:
- `web_add_auto_header()` calls at the top of `Action()` for headers that appear with the same value across ≥80% of requests, such as `User-Agent`, `Accept`, `Accept-Language`, `X-Requested-With`, and any app-specific constant headers
- `web_add_header()` calls before specific requests for per-request varying headers, such as `Origin` (on cross-origin POST requests) and custom app headers that change per-request

Header names are correctly formatted with hyphens preserved: `x-xsrf-header` → `X-Xsrf-Header` (not `XXsrfHeader`).

If you see a header-less script from an older version, regenerate using the current version of the tool.

> **Intentionally omitted** (VuGen handles these automatically or they are browser-internal):
> - `Referer` → already in the `"Referer=..."` attribute of `web_url` / `web_custom_request`
> - `Content-Type` → handled by `"EncType=..."` in `web_custom_request` and automatically by ITEMDATA in `web_submit_data`
> - `Cookie` → managed by VuGen's cookie jar automatically
> - `Content-Length` → computed by VuGen from the body
> - `Accept-Encoding` → VuGen handles gzip/deflate/br decompression automatically
> - `Priority` → HTTP/2 urgency hint, not applicable to VuGen's protocol layer
> - `Keep-Alive`, `Connection` → connection management handled internally by VuGen
> - Browser client hints: `Sec-CH-UA-*`, `Sec-Fetch-*`, `DNT`, `Cache-Control`, `Pragma`

---

### ❓ My script contains WebSocket requests (wss://) that fail with 400 Bad Request

**This should no longer happen** — the tool automatically filters out all `wss://` and `ws://` URLs. If you are seeing this in a script generated by an older version, regenerate using the current version of the tool.

> VuGen's Web HTTP/HTML protocol cannot replay WebSocket connections. WebSocket traffic captured in a HAR file is only the HTTP upgrade request — the actual WebSocket frames are not in the HAR. Including these produces 400 errors and adds no value to the script.

---

### ❓ My script contains requests with `"blocked-uri":"https://...invalid/"` in the body

**This should no longer happen** — the tool automatically filters out CSP violation reports. These are POST requests that browsers send automatically when a Content Security Policy blocks a URL (such as the `.invalid` bookmarklet marker URLs).

If you see them in a script generated by an older version, regenerate using the current version.

---

### ❓ The generated script is empty or only has the function wrapper

**Cause:** All requests were filtered out (nothing passed the filters).

**Fix:**
1. Uncheck **all** filter checkboxes (Static Assets, Analytics/Ads, OPTIONS)
2. If requests appear, turn filters back on one at a time until you find which one removed your requests
3. If still empty, the HAR may not contain real HTTP requests — try re-recording

---

### ❓ My transaction name changed from a hyphen to an underscore

**This is intentional.** VuGen C code and DevWeb JavaScript do not allow hyphens in names. The tool automatically converts them:

```
You typed:   T01-Login
Tool shows:  T01_Login   ← automatically corrected
```

---

# SECTION 4 — Tool 2 Issues (VuGen-Script-Studio.html)

---

### ❓ The tool shows an error or spins forever after I drop the HAR files

**Check these things:**
1. Both files must end in `.har`
2. Both files should contain the same user journey (same URLs in the same order)
3. The files should not be open in another program
4. Try one HAR file at a time to see which causes the problem

---

### ❓ What types of dynamic values can Script Studio find?

**Two-HAR mode** detects any value that changes between your two recordings and matches one of these patterns:

| Token type | Examples |
|---|---|
| JWT tokens | `eyJhbGciOi...` (Authorization Bearer, API tokens) |
| UUIDs (any case) | `8f14e45f-ceea-467a-a866-051e1f0b36fe` |
| MD5 / SHA-256 hashes | 32-char or 64-char hex strings |
| Long opaque tokens ≥32 chars | OAuth tokens, Base64 session keys |
| Instance / trace / request IDs | `i-0a1b2c3d4e5f67890`, `req-abc123-def456`, `x-instance-id` values (12–35 alphanumeric chars) |
| Snowflake / numeric IDs | 15+ digit pure numeric IDs |
| Compound server keys | `201;437;02/27/2026`, `tok:v1:abc\|xyz` |
| Framework hidden fields | `_sourcePage`, `__fp`, `__VIEWSTATE`, `__RequestVerificationToken`, `authenticity_token` and many more |
| CSRF headers | `X-CSRF-Token`, `X-XSRF-Token`, `X-Request-Token`, and any custom header whose name contains `csrf`, `xsrf`, `antiforg`, or similar — including app-specific names like `x-xsrf-header` |
| Session cookies | JSESSIONID, PHPSESSID, ASP.NET_SessionId, and others |

The correlation is also checked in URL path parameters (`;key=value`), query parameters, request headers, JSON body fields, and form-encoded body fields.

---

### ❓ My CSRF header (e.g. `x-xsrf-header`) is showing as "Unresolved" or not correlated

**This should no longer happen.** Script Studio automatically correlates any request header whose name contains `csrf`, `xsrf`, `antiforg`, or similar — including custom application headers like `x-xsrf-header`, `x-csrf-header`, and `x-anti-forgery-token`.

**How it works:** CSRF tokens follow the double-submit cookie pattern — the server sets a cookie (e.g. `XSRF-TOKEN`) and the browser JavaScript reads it and sends it back as a request header. The tool detects this by matching the header value against the cookies in the same request, then generates an extractor that reads the cookie directly.

This works even when the original `Set-Cookie` response is not captured in the HAR (common when the browser served it from cache).

If you see this in a script generated by an older version, regenerate using the current version of the tool.

---

### ❓ The tool only shows a few correlations — I expected more

**Likely cause:** The two HAR files are from the same browser session (not truly separate sessions). Each recording must be from a fresh login with a different session ID.

> ✅ **Using the same username and password for both recordings is completely fine and expected.** Credentials do not need to change. What matters is that each recording starts a fresh browser session so the application issues new session IDs, CSRF tokens, and other server-generated values. If you logged out (or used Incognito) before Recording 2, your workflow is correct.

**Fix (if you did not start a fresh session):**
- Use an Incognito/Private window (`Ctrl+Shift+N`) for the second recording
- Or log out fully and clear cookies before the second recording
- Or record on a different computer

---

### ❓ I only have one HAR file — can I still use Script Studio?

Yes. Drop only the first HAR onto the **left drop zone** and leave the right zone empty. The tool will switch to single-HAR mode and detect common patterns:
- JWT Bearer tokens
- CSRF headers (`X-CSRF-Token`, `X-XSRF-Token`, and any header name matching the csrf/xsrf/antiforg pattern)
- Session cookies in URL paths (`;jsessionid=xxx`)
- Known hidden form fields (`_sourcePage`, `__fp`, `__VIEWSTATE`, etc.)

Two HARs gives significantly better results for custom application parameters.

---

### ❓ Script Studio says "TODO — manual correlation needed"

Some dynamic values could not be automatically traced to their source. This can happen when:
- The HAR file was saved without response bodies (check you selected "with content")
- The value is generated client-side (JavaScript-calculated, not from a server response)
- The value comes from a redirect chain that wasn't captured

**What to do:** Pass the script to your LRE admin — they can manually add the correlation rule using VuGen's Correlation Studio.

---

### ❓ JSESSIONID is still hardcoded in the script

This means the tool could not find the JSESSIONID in the HAR response headers (common when HAR is recorded with DevTools caching enabled).

**Fix — use two HAR files.** The two-HAR diff will detect JSESSIONID as a changed value and automatically generate a `web_reg_save_param` rule to extract it from the `Set-Cookie` response header.

---

### ❓ The Parameters panel in VuGen shows no parameters

This can happen if the `.usr` file references the wrong parameter file. Make sure you copied **all** files from the ZIP:
- `ParameterFile.prm` — must be present
- `collection_data.dat` — must be present
- Both must be in the same folder as `Action.c`

If VuGen still shows nothing, open `[ScriptName].usr` in Notepad++ and check that it contains:
```
ParameterFile=ParameterFile.prm
```
(Not `ParameterFile=` with nothing after it)

---

# SECTION 5 — Download Issues

---

### ❓ When I click Download, nothing happens

1. Check your browser's Downloads folder — the ZIP may have downloaded silently
2. Look for a blocked download notification near the browser address bar
3. Try a different browser (Chrome usually works best for downloads)

---

### ❓ The downloaded ZIP file won't open

| Cause | Fix |
|---|---|
| File is corrupted (partial download) | Try downloading again |
| No ZIP program installed | Right-click → Open with → Windows Explorer (built-in) or 7-Zip |
| File was renamed to `.zip.crdownload` | Download did not complete — try again |

---

### ❓ I can't find the downloaded file

On Windows, browser downloads go to:
```
C:\Users\[YourName]\Downloads\
```

Or press **Ctrl+J** in Chrome/Edge to open the downloads list.

---

# SECTION 6 — Script Runtime Issues (in VuGen/LRE)

---

### ❓ VuGen shows "Unresolved symbol" error when replaying

**Common causes:**

| Error text | Meaning and fix |
|---|---|
| `Unresolved symbol: web_reg_save_param_cookie` | This function does not exist. The script was generated by an older version of the tool — regenerate using the current version of Script Studio. |
| `Unresolved symbol: LAST` | The `globals.h` file is missing from your project folder. Make sure you copied all files from the ZIP. |

---

### ❓ VuGen shows Error -26396 "argument unrecognised" in web_reg_save_param

**Cause:** The generated `web_reg_save_param` call has incorrect syntax. This is a bug in an older version of the tool.

**Fix:** Regenerate the script using the current version of `VuGen-Script-Studio.html`. The correct syntax is:
```c
web_reg_save_param("ParamName",
    "LB=left boundary text",
    "RB=right boundary text",
    "Ord=1",
    LAST);
```

---

### ❓ The script runs but all transactions fail with "Parameter not found"

**Cause:** `web_reg_save_param` could not find the expected text in the server response at runtime.

**Common reasons:**
- The application response changed since you recorded (different HTML structure)
- The correlation boundaries (LB/RB) are too specific
- The request that was supposed to return the value is being cached

**What to do:** Ask your LRE admin to check the correlation in VuGen's Correlation Studio.

---

### ❓ The script runs but all virtual users use the same credentials

**Cause:** The `collection_data.dat` file only has one row of data.

**Fix:** Open `collection_data.dat` in Notepad++ and add one row per virtual user:
```
"Username","Password"
"user1","pass1"
"user2","pass2"
"user3","pass3"
```

---

### ❓ The script has `// TODO: Unresolved correlations` comment blocks — what do I do?

These comments appear when a dynamic value was detected (it changed between your two recordings) but its source response body was not captured in the HAR — typically because the browser used a cached response.

**The comment tells you:**
- Which parameter name needs to be resolved (e.g. `// • AuthToken — used in: https://app.example.com/api/login`)
- What to do next: re-record with **"Disable cache"** enabled in DevTools (⚙ → Disable cache), then regenerate

**No code is broken** — the TODO is a comment only. The script will compile, but the dynamic value will not be extracted at runtime until you resolve it.

---

### ❓ Think time in the script is wrong

The script adds `lr_think_time(3)` (3 seconds) between transactions by default.

To change it: Open `Action.c` in Notepad++ → find each `lr_think_time(3)` line → change the number to your required think time in seconds.

---

### ❓ The .c file looks garbled when opened in Notepad

This is normal — Notepad doesn't handle the line endings in C files well.

**Use Notepad++** (free download) to view and edit `.c` files. It displays them correctly.

---

# SECTION 7 — Getting More Help

| Who to contact | When |
|---|---|
| **LRE Admin Team** | Script uploads, project folders, LRE settings, manual correlation |
| **Team Lead** | Test scenario design, transaction planning, load model |
| **IT Helpdesk** | Browser issues, file access restrictions, blocked downloads |

---

## Quick Self-Check Before Contacting Support

- [ ] Are the ▶ START and ■ END bookmarks in your Bookmarks Bar?
- [ ] Was Developer Tools (F12) open for the **entire** recording?
- [ ] Did you save as **"HAR with content"** (not just "HAR")?
- [ ] For Script Studio: did you use two separate browser sessions (not the same)?
- [ ] Did you copy **ALL** files from the downloaded ZIP to your project folder?
- [ ] Is the `ParameterFile.prm` file in the same folder as `Action.c`?

---

*LRE Admin Tool — Troubleshooting Guide v2.1*
