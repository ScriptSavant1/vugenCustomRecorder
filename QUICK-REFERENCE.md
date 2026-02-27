# LRE HAR Script Generator — Quick Reference Card

> Print this page and keep it on your desk while recording.
> **No software to install — just open the HTML files in your browser.**

---

## The Two Tools at a Glance

| Tool | File | When to use |
|---|---|---|
| **VuGen-Recorder** | `VuGen-Recorder.html` | Quick script from one HAR recording |
| **Script Studio** | `VuGen-Script-Studio.html` | Production script with auto-correlation (needs two HAR files) |

---

## TOOL 1 — VuGen-Recorder.html

### One-Time Bookmarklet Setup

```
1  Open VuGen-Recorder.html in your browser
2  Show Bookmarks Bar:  Ctrl+Shift+B  (Chrome / Edge / Firefox)
3  Drag  ▶ START Transaction  from the green box → onto the Bookmarks Bar
4  Drag  ■ END Transaction    from the green box → onto the Bookmarks Bar
   Done — stays permanently in your browser
```

---

### Transaction Naming Rules

```
✅  T01_Login              ← CORRECT (letter + number + underscore)
✅  T02_Product_Search     ← CORRECT
✅  T03_Add_To_Cart        ← CORRECT

❌  T01 Login              ← WRONG  (no spaces)
❌  01_Login               ← WRONG  (must start with a letter)
❌  T01-Login              ← WRONG  (use underscores, not hyphens)
```

---

### Recording Checklist

```
BEFORE STARTING:
  □  Plan transaction names (T01, T02, T03...)
  □  Open browser → press F12
  □  Click the "Network" tab
  □  Click 🚫 to clear existing entries

FOR EACH TRANSACTION:
  □  Click  ▶ START Transaction  → type name → OK
  □  Perform the steps in your application (click, fill forms, submit)
  □  Click  ■ END Transaction    → type same name → OK
  □  Immediately start next transaction (click ▶ START again)

AFTER ALL TRANSACTIONS:
  □  Network tab → right-click any row
  □  Click "Save all as HAR with content"
  □  Save the file (e.g. MyApp_Recording_1.har)
```

> ⚠️ **Keep Developer Tools (F12) open the entire time. Closing it loses all data.**

---

### Generate Script (Tool 1)

```
1  Open VuGen-Recorder.html
2  Drag your .har file onto the grey drop area (or click Browse)
3  Choose format: Web HTTP/HTML  or  DevWeb  or  Both
4  Review the request table
5  Click "⬇ Download Script"
6  Unzip and copy files to your LRE project folder
```

---

## TOOL 2 — VuGen-Script-Studio.html

### What You Need

- **HAR Recording 1** — your first recording (`MyApp_Recording_1.har`)
- **HAR Recording 2** — a second identical recording in a fresh browser session (`MyApp_Recording_2.har`)

```
For Recording 2:
  □  Log out of the application completely (or use Incognito window)
  □  Open browser → F12 → Network → Clear
  □  Repeat the EXACT same steps as Recording 1
  □  Save as  MyApp_Recording_2.har
```

---

### Generate Correlated Script (Tool 2)

```
1  Open VuGen-Script-Studio.html
2  Drop  MyApp_Recording_1.har  onto the LEFT drop zone
3  Drop  MyApp_Recording_2.har  onto the RIGHT drop zone
4  Choose format: Web HTTP/HTML  or  DevWeb
5  Click "Generate Script"
6  Review the Correlations panel (right side)
7  Click "⬇ Download ZIP"
8  Unzip and copy files to your LRE project folder
```

> One HAR only? Drop it on the LEFT zone only — Studio still detects common patterns.

---

## What Each Downloaded File Is For

### Web HTTP/HTML Output
| File | Purpose |
|---|---|
| `Action.c` | Main script — all your requests and transactions |
| `vuser_init.c` | Runs once before the test starts |
| `vuser_end.c` | Runs once after the test ends |
| `globals.h` | Required project header |
| `default.cfg` | Runtime settings |
| `ParameterFile.prm` | Parameter definitions (username, password, etc.) |
| `collection_data.dat` | Data rows — one per virtual user iteration |

### DevWeb Output
| File | Purpose |
|---|---|
| `main.js` | Complete script — all requests, correlations, parameters |
| `parameters.yml` | Parameter definitions |
| `collection_data.csv` | Data rows — one per virtual user iteration |

---

## Bookmarklet Explained

| Bookmark | What happens when you click it |
|---|---|
| ▶ START Transaction | Asks for a name → fires `START-T01_Login.invalid` silently → you stay on page |
| ■ END Transaction | Asks for a name → fires `END-T01_Login.invalid` silently → you stay on page |

> The requests show as "failed" in the Network tab — **that is intentional and correct**.

---

## Address Bar Fallback (if bookmarklets are blocked)

| What you want | Type in the address bar |
|---|---|
| Start transaction 1 | `https://START-T01-Login.invalid` |
| End transaction 1 | `https://END-T01-Login.invalid` |
| Start transaction 2 | `https://START-T02-Search.invalid` |

> Press Enter → ignore the error page → press the back button → continue in your app.

---

## Filters (Tool 1 only)

| Checkbox | What it hides |
|---|---|
| Static Assets | Images, fonts, CSS, JavaScript files |
| Analytics/Ads | Google Analytics, Facebook trackers, ad pixels |
| OPTIONS | Browser pre-flight requests (not needed in script) |

> Uncheck to include something. The script preview updates instantly.

**Always auto-filtered (cannot be included):**
CSP violation reports, WebSocket connections (`wss://`/`ws://`).
These are excluded because VuGen cannot replay them — they would cause 400 errors.

---

## Collapsing Transactions (Tool 1)

| Action | How |
|---|---|
| Collapse one transaction | Click the coloured ▼ START row |
| Expand one transaction | Click the coloured ▶ START row |
| Collapse all at once | Click "⊖ Collapse All" in the toolbar |
| Expand all at once | Click "⊕ Expand All" in the toolbar |

---

## Status Colours in the Request Table

| Colour | Code | Meaning |
|---|---|---|
| 🟩 Green | 200–299 | Success |
| 🔵 Blue | 300–399 | Redirect |
| 🟧 Orange | 400–499 | Client error — check if expected |
| 🟥 Red | 500–599 | Server error |
| ⬜ Grey | 0 | Failed or blocked (markers appear as 0 — correct) |

---

## Manual Transactions (forgot bookmarklets?)

```
TOOL 1:
  1  Click "☑ Select Mode" → turns blue
  2  Click each request row you want to include
  3  Click "+ Create Transaction" → type name → Create
  Repeat for each transaction

TOOL 2:
  Not needed — Script Studio detects transactions automatically
  from the markers already in the HAR file
```

---

*LRE Admin Tool — Quick Reference v2.1*
