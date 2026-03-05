# LRE HAR Script Generator — User Guide

> **For:** Anyone who needs to create LoadRunner Enterprise (LRE) performance test scripts
> **Skill level required:** Basic computer skills — no technical knowledge needed
> **No installation required** — just open the HTML files in your browser

---

## Overview: Two Tools, One Workflow

This toolset has two tools. Most users will use both:

```
┌──────────────────────┐       ┌──────────────────────────────────────────┐
│                      │       │                                          │
│  VuGen-Recorder.html │  →→→  │  VuGen-Script-Studio.html               │
│                      │       │                                          │
│  RECORD your         │       │  CORRELATE & PARAMETERISE your script    │
│  actions in the      │       │  (makes it ready for real load tests     │
│  browser and save    │       │  with multiple concurrent users)         │
│  as a HAR file       │       │                                          │
└──────────────────────┘       └──────────────────────────────────────────┘
     Step 1                              Step 2
```

**If you just want to explore:** Tool 1 alone generates a working script.
**If you want a production-quality script:** Use Tool 2 after Tool 1.

---

# PART A — TOOL 1: VuGen-Recorder.html

## A1. Before You Start

**You will need:**
- [ ] The file `VuGen-Recorder.html` (from your LRE admin team)
- [ ] Google Chrome, Microsoft Edge, or Firefox
- [ ] Access to the application you want to test
- [ ] A list of the transactions you want to measure (see below)

**What is a transaction?**
A transaction is one logical step your user takes — something you want to measure separately.

**Example — Online shopping site:**

| Transaction Name | User action |
|---|---|
| `T01_Homepage` | Open the website |
| `T02_Login` | Enter credentials and sign in |
| `T03_Search` | Search for a product |
| `T04_AddToCart` | Add an item to the cart |
| `T05_Checkout` | Complete the purchase |

**Transaction naming rules:**
```
✅ Correct:    T01_Login        T02_Search       T03_Add_To_Cart
❌ Wrong:      T01 Login        (no spaces)
❌ Wrong:      01_Login         (must start with a letter)
❌ Wrong:      T01-Login        (no hyphens — use underscores)
❌ Wrong:      T01_Login!!      (no special characters)
```

---

## A2. One-Time Setup — Add the Bookmarklets to Your Browser

> You only need to do this once. After this, the bookmarklets stay in your browser permanently.

**What are bookmarklets?** They are two special bookmarks that silently mark the start and end of each transaction while you record — without taking you away from your application page.

### Step 1 — Show Your Browser's Bookmarks Bar

The Bookmarks Bar is the row of saved links just below the address bar.

- **Chrome or Edge:** Press `Ctrl + Shift + B`
- **Firefox:** Press `Ctrl + Shift + B`

The bar will appear. If you already have bookmarks, you will see them there.

---

### Step 2 — Open VuGen-Recorder.html and Drag the Bookmarklets

1. Open `VuGen-Recorder.html` in your browser (double-click the file)
2. You will see a green box at the top labelled **"Set Up Bookmarklets"**
3. Inside it are two buttons:
   - A green **▶ START Transaction** button
   - A red **■ END Transaction** button
4. **Drag each button** from the page onto your Bookmarks Bar

```
   Bookmarks Bar (just below the address bar):
   ┌───────────────────────────────────────────────────────────────┐
   │  ▶ START Transaction   ■ END Transaction   [other bookmarks]  │
   └───────────────────────────────────────────────────────────────┘
            ↑                       ↑
       Drag here               Drag here
       from the green box in the tool
```

5. Both bookmarks now appear in your Bookmarks Bar — setup complete.

> **If dragging doesn't work:** Right-click each button in the tool → select **"Bookmark this link"** → choose **Bookmarks Bar** as the folder → click Save.

---

## A3. Recording Your Application

### Step 3 — Open Developer Tools (F12)

1. Open your browser (Chrome or Edge recommended)
2. Navigate to your application's starting page
3. Press **F12** on your keyboard — a panel will open at the bottom or side of the screen
4. Click the **"Network"** tab inside that panel
5. Enable the **Preserve Log** checkbox in the Network toolbar
6. Enable the **Disable Cache** checkbox in the Network toolbar
7. Click the **Clear** (stop sign) button to remove any old entries
8. Confirm the **red record circle** is active

> **Important:** Keep Developer Tools open for the entire recording. Do not close it.

> **Why Preserve Log?** Keeps all network requests visible across page navigations and redirects. Without this, requests made during login redirects disappear from the list.

> **Why Disable Cache?** Forces fresh responses from the server instead of the browser cache. Without this, response bodies may be empty in the saved HAR — making correlation extraction in Script Studio impossible.

---

### Step 4 — Record Your Transactions

Repeat these three steps for each transaction:

**a) Mark the START of the transaction**
1. Click the **▶ START Transaction** bookmark in your Bookmarks Bar
2. A small popup appears — type your transaction name (e.g. `T01_Login`) and click **OK**
3. You stay on your current page — the marker fires silently in the background

**b) Perform the transaction steps**
- Click links and buttons in your application
- Fill in forms, submit them, wait for pages to load
- Use the application exactly as a real user would

**c) Mark the END of the transaction**
1. Click the **■ END Transaction** bookmark
2. Type the **same name** you used for START (e.g. `T01_Login`) → click **OK**
3. Immediately click **▶ START Transaction** for your next transaction

**Example of what you will see in the Network tab:**
```
  START-T01_Login.invalid     ← Marker (status: failed — this is correct)
  GET   /login          200
  POST  /authenticate   200
  GET   /dashboard      200
  END-T01_Login.invalid       ← Marker (status: failed — this is correct)
  START-T02_Search.invalid    ← Marker
  GET   /search?q=...   200
  END-T02_Search.invalid      ← Marker
```

> The `.invalid` markers show as "failed" in the Network tab — **this is intentional and correct**.

---

### Step 5 — Save the HAR File

After completing all transactions:

1. In the Network tab, **right-click** on any request row
2. Click **"Save all as HAR with content"**
3. A Save dialog appears — choose a folder (Desktop or Downloads)
4. Name the file something meaningful: e.g., `MyApp_Recording_1.har`
5. Click **Save**

> **Firefox users:** Click the download arrow icon (⬇) in the Network tab toolbar → "Save All As HAR"

---

### ⚠️ What if my application opens a popup window or new tab?

Developer Tools (F12) only captures traffic from the tab it is attached to. If your user journey involves a popup window or new browser tab, use **`chrome://net-export/`** — it captures all tabs simultaneously and both tools read NetLog `.json` files directly (no conversion needed).

**Using chrome://net-export/:**
1. Open a new tab and navigate to `chrome://net-export/`
2. Click **"Start Logging to Disk"** → save the file
3. Open your application in **another new tab** (the net-export tab stays running in the background)
4. Use the **▶ START** / **■ END** bookmarklets as normal — they are captured from any tab
5. Complete your entire journey including all popups and new tabs
6. Return to `chrome://net-export/` and click **"Stop Logging"**
7. Drop the `.json` file into the tool — it works the same as a `.har` file

> **Note:** NetLog does not include POST request bodies. The generated script will show `// TODO: POST body not available in NetLog` for those requests — add the body content manually using your application's API documentation or by checking DevTools in a parallel session.

---

## A4. Generating Your Script

### Step 6 — Load the HAR File into the Tool

1. Open `VuGen-Recorder.html` in your browser
2. **Drag and drop** your `.har` file onto the grey drop area
   — OR click the **"Browse HAR File"** button and find the file
3. A dialog appears asking which script format you want:

| Format | Choose this if... |
|---|---|
| **Web HTTP/HTML** | Your LRE project uses Web HTTP/HTML protocol |
| **DevWeb** | Your LRE project uses DevWeb protocol |
| **Both** | You want both formats generated at once |

4. Select the format and click **OK**

The tool will instantly:
- Detect your transaction markers
- Filter out noise (images, fonts, analytics trackers)
- Generate your script(s)

---

### Step 7 — Review and Download

**Review the table:** You will see your requests grouped into colour-coded transaction bands. Check that the groupings look correct.

**Collapse/expand transactions:** If you have many transactions, click any transaction header row (the coloured ▼ START row) to collapse that group. Use the **"⊖ Collapse All"** button in the toolbar to minimise everything at once, then expand just the transactions you want to inspect.

**Adjust filters if needed:**
- The left panel shows domains — uncheck any you do not want in the script
- The top checkboxes filter static assets, analytics, and pre-flight requests
- The script preview updates live as you change filters

> **Note:** Some requests are always excluded automatically: browser pre-flights (OPTIONS), analytics trackers, CSP violation reports, and WebSocket connections (`wss://`). These cannot be replayed by VuGen and would cause errors if included.

**Download your script:**
Click **"⬇ Download Script"** → a ZIP file downloads containing your complete VuGen project.

---

### Step 8 — Copy to Your LRE Project

**Unzip the downloaded file.** It contains:

| File | Protocol | Purpose |
|---|---|---|
| `Action.c` | Web HTTP/HTML | Main script with all requests and transactions |
| `vuser_init.c` | Web HTTP/HTML | Initialization (runs once at start) |
| `vuser_end.c` | Web HTTP/HTML | Cleanup (runs once at end) |
| `globals.h` | Web HTTP/HTML | Required header file |
| `default.cfg` | Web HTTP/HTML | Runtime settings |
| `ScriptUploadMetadata.xml` | Web HTTP/HTML | Required project metadata |
| `main.js` | DevWeb | Complete DevWeb script |

Copy all files into your VuGen/LRE project folder. Ask your LRE admin if you are unsure where that is.

---

# PART B — TOOL 2: VuGen-Script-Studio.html

## Why Use Tool 2?

A basic recording captures the exact values that were used during your recording session — for example, your session ID, CSRF token, and hidden form fields. These values change with every new session. If you run the basic script under load, every virtual user will fail because they will all try to use the same session ID from your recording.

**Tool 2 solves this** by comparing two recordings and automatically detecting every value that changes between sessions. It then generates `web_reg_save_param` rules to extract those values dynamically at runtime.

---

## B1. Record a Second HAR File

You need to record the **same user journey twice**. Each recording must be a completely separate browser session (log out fully, then log in again as a fresh user).

**Recording 1:** Already done in Part A — this is your `MyApp_Recording_1.har`

**Recording 2:**
1. Close your browser completely (or open a fresh Incognito window: `Ctrl+Shift+N`)
2. Repeat the exact same steps as Recording 1:
   - Press F12 → Network tab → enable **Preserve Log** and **Disable Cache** → Clear entries
   - Click ▶ START for each transaction → do the steps → click ■ END
   - Save as HAR → name it `MyApp_Recording_2.har`

> **Important:** Perform exactly the same actions in both recordings. The tool compares them step by step.

---

## B2. Open VuGen-Script-Studio.html

1. Double-click `VuGen-Script-Studio.html` to open it in your browser
2. You will see the upload area with two drop zones

---

## B3. Load Your HAR Files

**With two HAR files (recommended):**
1. Drag `MyApp_Recording_1.har` onto the **left drop zone** (labelled "HAR Recording 1")
2. Drag `MyApp_Recording_2.har` onto the **right drop zone** (labelled "HAR Recording 2")
3. Click **"Generate Script"**

**With one HAR file only:**
1. Drag your HAR file onto the **left drop zone** only
2. Click **"Generate Script"**
   (The tool will still detect common patterns like JWT tokens, CSRF headers, and session cookies)

---

## B4. What the Tool Does Automatically

While processing, the tool performs several steps automatically:

```
1. Match requests    → Pairs up each request in Recording 1 with its
                       equivalent in Recording 2 (same URL + method)

2. Detect changes    → Finds every value that is different between the
                       two sessions (session IDs, tokens, form fields,
                       CSRF headers, UUID path segments, cookies)

3. Back-trace source → For each changed value, finds the source:
                         a) Response body / header / Set-Cookie extractor
                         b) Double-submit cookie (CSRF headers) — even
                            when Set-Cookie is absent from HAR
                         c) Client-generated token (UUID/hex format) →
                            generates a helper function called only before
                            the specific requests that need the header

4. Generate rules    → Creates web_reg_save_param() / CookieExtractor /
                       helper function rules placed correctly.
                       CSRF/XSRF helper: only the N requests that send the
                       header call the function — others are untouched.

5. Parameterise      → Detects common user data fields (username, password,
                       card number, dates) and replaces them with parameters

6. Build project     → Assembles a complete, ready-to-use VuGen project ZIP
```

---

## B5. Review the Results

After processing, you will see:

**Script preview tabs:**
- **Action.c** — Web HTTP/HTML script with all correlations and parameters
- **main.js** — DevWeb script with all correlations and parameters
- **ParameterFile.prm** — Parameter file listing all detected data parameters
- **collection_data.dat** / **collection_data.csv** — Data files with your recorded values

**Correlations panel (right side):** Lists every dynamic value that was detected and correlated. For each one, it shows:
- The parameter name (e.g. `JSessionId`, `SourcePage`, `AuthToken`)
- The extraction type (from response body, response header, or cookie)
- Where it is used in the script

**Parameters panel:** Lists data values that were detected and made into parameters (e.g. `Username`, `Password`, `CardNumber`).

---

## B6. Download the Script

Click **"⬇ Download ZIP"** — a complete VuGen project archive downloads.

**What is inside the ZIP:**

For **Web HTTP/HTML:**
```
Action.c                  ← Main script with correlations and parameterisation
vuser_init.c              ← Initialization file
vuser_end.c               ← Cleanup file
globals.h                 ← Required header
default.cfg               ← Runtime settings
ScriptUploadMetadata.xml  ← Project metadata
ParameterFile.prm         ← Parameter definitions (username, password, etc.)
collection_data.dat       ← Data rows (one per virtual user iteration)
```

For **DevWeb:**
```
main.js                   ← Complete DevWeb script
parameters.yml            ← Parameter definitions
collection_data.csv       ← Data rows
rts.yml                   ← Runtime settings
tsconfig.json             ← TypeScript config
DevWebSdk.d.ts            ← SDK type definitions
```

---

## B7. Copy to Your LRE Project

Unzip the downloaded file and copy all files into your LRE project folder. For Web HTTP/HTML, copy into your existing VuGen project folder (replacing existing files). Ask your LRE admin if you need help.

---

# PART C — COMMON QUESTIONS

## Can I use Tool 2 without Tool 1?

Yes. VuGen-Script-Studio.html is completely independent. You can record HAR files directly using browser DevTools (F12) without going through VuGen-Recorder.html first.

## What format should I choose?

Ask your LRE admin. If you are unsure:
- **Web HTTP/HTML** — the more established format, used in most existing LRE environments
- **DevWeb** — newer format, preferred for modern web applications

## What if my username and password are hardcoded in the script?

Tool 2 automatically detects username and password fields and replaces them with `{Username}` and `{Password}` parameters. You then add your test user credentials to the `collection_data.dat` file (one row per virtual user).

## What if the tool shows "unresolved candidates"?

CSRF/XSRF headers and UUID/hex tokens are **always handled automatically** — they will not appear as unresolved.

For all other values, the tool traces back through responses automatically. If a value cannot be traced (e.g., response body was not captured because the browser served from cache), a `// TODO` comment block appears in the script. **No broken code is emitted** — the comment explains what is needed. Your LRE admin can resolve these by re-recording with **"Disable cache"** enabled in DevTools (⚙ → Disable cache), which causes the response body to be captured in the HAR, and then regenerating the script.

---

# PART D — WORKFLOW SUMMARY

```
┌─────────────────────────────────────────────────────────────────────────┐
│                                                                         │
│  STEP 1 — PLAN                                                          │
│  Write down your transaction names (T01_Login, T02_Search, ...)         │
│                                                                         │
│  STEP 2 — SETUP (one-time only)                                         │
│  Open VuGen-Recorder.html → drag ▶ START and ■ END to Bookmarks Bar    │
│                                                                         │
│  STEP 3 — RECORD (do this twice for best results)                       │
│  F12 → Network tab → Preserve Log ON → Disable Cache ON → Clear         │
│  For each transaction:                                                   │
│    Click ▶ START → do the steps → click ■ END                          │
│  Right-click in Network → Save all as HAR with content                  │
│                                                                         │
│  STEP 4 — GENERATE BASIC SCRIPT                                         │
│  Open VuGen-Recorder.html → drop HAR → choose format → download ZIP    │
│                                                                         │
│  STEP 5 — GENERATE PRODUCTION SCRIPT (recommended)                      │
│  Open VuGen-Script-Studio.html → drop BOTH HAR files → Generate        │
│  → Review correlations panel → download ZIP                             │
│                                                                         │
│  STEP 6 — UPLOAD TO LRE                                                 │
│  Copy ZIP contents to your VuGen project folder                         │
│  Ask your LRE admin to upload and run the script                        │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Need Help?

| Who to contact | When |
|---|---|
| **LRE Admin Team** | Getting the tools, understanding output formats, uploading to LRE |
| **Team Lead** | Deciding which transactions to test, test scenario design |
| **IT Helpdesk** | Browser issues, file access, blocked downloads |

---

*LRE Admin Tool — VuGen HAR Script Generator Suite v2.0*
