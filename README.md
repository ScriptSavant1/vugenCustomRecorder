# VuGen HAR Script Generator

A single-file browser tool that converts browser HAR recordings into ready-to-use LoadRunner Enterprise (LRE) performance test scripts — no installation, no admin rights, no browser extension required.

## Why This Tool Exists

VuGen's built-in recorder installs kernel-level network drivers (WinPcap/NPCAP) which are blocked by enterprise antivirus and org policy. This tool provides a fully browser-based alternative using the browser's built-in DevTools HAR export.

## How It Works

```
Step 1 — Record         Step 2 — Import          Step 3 — Download
──────────────          ────────────────          ─────────────────
Open browser F12   →    Drop .har file       →    Scripts generated
Use bookmarklets        into the tool             automatically
to mark START/END       Group into                for both protocols
of transactions         transactions
```

## Features

- **Zero dependencies** — single `.html` file, works offline, no npm, no server
- **Bookmarklet-based transaction marking** — click a bookmark to start/end a transaction without leaving your app
- **Dual format output** — generates both Web HTTP/HTML (C) and DevWeb (JavaScript) scripts simultaneously
- **Format selector** — choose which protocol format you need before generating
- **Domain filter panel** — filter requests by domain (like VuGen's Recording Report Hosts tab)
- **Smart noise filtering** — auto-removes static assets, analytics trackers, OPTIONS preflight requests
- **Select Mode** — manually group requests into transactions if you forgot to use bookmarklets
- **Resizable panels** — drag the dividers between domain list, request table, and script preview
- **Live script preview** — see generated scripts update instantly as you change filters

## Output Files

| File | Protocol | Purpose |
|---|---|---|
| `Action.c` | Web HTTP/HTML | Main script with transactions and requests |
| `vuser_init.c` | Web HTTP/HTML | Initialization stub |
| `vuser_end.c` | Web HTTP/HTML | Cleanup stub |
| `globals.h` | Web HTTP/HTML | Required header file |
| `main.js` | DevWeb | Complete DevWeb protocol script |

## Usage

1. **One-time setup:** Open `VuGen-Recorder.html` → drag the **▶ START Transaction** and **■ END Transaction** bookmarklets to your browser's Bookmarks Bar
2. **Record:** Open your app → F12 → Network tab → click bookmarks around each transaction → save as HAR
3. **Generate:** Drop the HAR file into the tool → select format → review → download

## Documentation

| File | Contents |
|---|---|
| `USER-GUIDE.md` | Full step-by-step guide for end users |
| `QUICK-REFERENCE.md` | Printable one-page cheat sheet |
| `TROUBLESHOOTING.md` | Common issues and fixes |

## Supported Browsers

Chrome, Microsoft Edge, Firefox

## Requirements

- No installation
- No admin rights
- No internet connection required (fully offline)
- Works on any machine that can open an HTML file in a browser

---

*LRE Admin Tool — VuGen HAR Script Generator v1.0*
