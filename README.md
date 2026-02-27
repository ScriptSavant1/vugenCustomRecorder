# LRE HAR Script Generator — Tool Suite

A set of **zero-installation** browser tools that replace VuGen's blocked network recorder for LoadRunner Enterprise (LRE) performance testing.

> **No installation. No admin rights. No internet connection. No additional files to download.**
> Everything is packaged inside the two HTML files below.

---

## What's In This Folder

| File | What it does |
|---|---|
| **`VuGen-Recorder.html`** | **Step 1 tool** — Records a HAR file and converts it into a basic LRE script |
| **`VuGen-Script-Studio.html`** | **Step 2 tool** — Compares two HAR recordings to automatically detect and correlate dynamic values (session IDs, tokens, form fields) |
| `USER-GUIDE.md` | Full step-by-step guide for both tools |
| `QUICK-REFERENCE.md` | Printable one-page cheat sheet |
| `TROUBLESHOOTING.md` | Solutions to common problems |

---

## Which Tool Do I Use?

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  STARTING OUT?  →  Use  VuGen-Recorder.html  first                  │
│                                                                     │
│  GETTING CORRELATION ERRORS or WANT A PRODUCTION-READY SCRIPT?     │
│                  →  Use  VuGen-Script-Studio.html  afterwards        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Tool 1 — VuGen-Recorder.html (Basic)

Use this when you want to:
- Quickly generate a script to test your recording works
- Get a first-cut script with transactions and request structure

**How it works:**
1. Open your app in a browser, press F12, record your actions using the bookmarklet markers
2. Export the HAR file from DevTools
3. Drop the HAR into this tool → review the request table → download your script

**Output:** A ZIP file containing a complete VuGen project (Action.c for Web HTTP/HTML, or main.js for DevWeb)

**Table features:** Requests are grouped into colour-coded transaction bands. Click a transaction header to collapse/expand its requests. Use "⊖ Collapse All" to minimise all transactions at once — useful when you have many transactions and want an overview.

**Auto-filtered (never included in scripts):** images, fonts, CSS, analytics trackers, OPTIONS pre-flights, CSP violation reports, and WebSocket (`wss://`/`ws://`) upgrade requests.

---

### Tool 2 — VuGen-Script-Studio.html (Advanced — Recommended for production)

Use this when you want to:
- Automatically detect dynamic values (session IDs, CSRF tokens, hidden form fields) and generate correlation rules
- Replace hardcoded values (username, password, card numbers, etc.) with parameterised data
- Generate a script that will actually work under load with multiple concurrent users

**How it works:**
1. Record your application **twice** (two separate HAR files — each recording is a separate login session)
2. Drop **both** HAR files into this tool
3. The tool compares the two recordings, detects every value that changed between sessions, and automatically generates all the `web_reg_save_param` extraction rules
4. Download a complete, production-ready VuGen project ZIP

**Can I use it with only 1 HAR?** Yes — it still detects common patterns (JWT tokens, CSRF headers, session cookies). Two HARs gives far better results.

---

## Distribution — What To Give To Your Team

This entire toolset is just **two HTML files**. No web server, no installation, no dependencies.

To share with your team, copy these files to a shared folder or email them:
```
VuGen-Recorder.html         ← Tool 1 (self-contained, ~170KB)
VuGen-Script-Studio.html    ← Tool 2 (self-contained, ~200KB)
USER-GUIDE.md               ← How-to guide
QUICK-REFERENCE.md          ← Cheat sheet
TROUBLESHOOTING.md          ← Problem fixes
```

> Both HTML files include all JavaScript libraries (JSZip etc.) **built-in**. Nothing else to install or download.

---

## Why This Tool Exists

VuGen's built-in recorder fails on most enterprise machines because it installs kernel-level network drivers (WinPcap/NPCAP) which are blocked by antivirus and organisation security policy. This toolset provides a fully browser-based alternative using the browser's built-in DevTools HAR export — requiring zero installation and zero admin rights.

---

## Supported Browsers

| Browser | Supported |
|---|---|
| Google Chrome | ✅ Recommended |
| Microsoft Edge | ✅ Fully supported |
| Firefox | ✅ Supported |
| Internet Explorer | ❌ Not supported |

---

## Supported Output Formats

| Format | Main File | Use In |
|---|---|---|
| Web HTTP/HTML (C) | `Action.c` | VuGen Web HTTP/HTML protocol projects |
| DevWeb (JavaScript) | `main.js` | VuGen DevWeb protocol projects |

---

*LRE Admin Tool — VuGen HAR Script Generator Suite v2.1*
