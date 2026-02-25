# VuGen HAR Script Generator — Quick Reference Card

> Print this page and keep it on your desk while recording.

---

## One-Time Setup — Add Bookmarklets

```
1  Open VuGen-Recorder.html in your browser
2  Show your Bookmarks Bar:  Ctrl+Shift+B  (Chrome / Edge / Firefox)
3  Drag  ▶ START Transaction  from the green box → onto the Bookmarks Bar
4  Drag  ■ END Transaction    from the green box → onto the Bookmarks Bar
Done — bookmarklets stay permanently in your browser
```

---

## Transaction Name Rules

```
✅  T01_Login              ← CORRECT
✅  T02_Product_Search     ← CORRECT
✅  T03_Add_To_Cart        ← CORRECT

❌  T01 Login              ← WRONG (no spaces)
❌  01_Login               ← WRONG (must start with a letter)
❌  T01-Login              ← WRONG (hyphens not allowed)
❌  T01_Login!!            ← WRONG (no special characters)
```

---

## Recording Checklist

```
Before Recording:
  □  Plan your transaction names (T01, T02, T03...)
  □  Open browser → press F12
  □  Click "Network" tab
  □  Click 🚫 to clear existing entries
  □  Confirm ▶ START and ■ END bookmarks are in your Bookmarks Bar

For EACH Transaction:
  □  Click  ▶ START Transaction  bookmark → type name → OK
  □  Browse your application normally (click, fill forms, submit)
  □  Click  ■ END Transaction    bookmark → type same name → OK
  □  Immediately click ▶ START for the next transaction

After All Transactions:
  □  Network tab → right-click any row
  □  Click "Save all as HAR with content"
  □  Save the file somewhere you can find it
```

> ⚠️ **Keep Developer Tools (F12) open for the ENTIRE recording. Closing it loses all data.**

---

## What the Bookmarklets Do

| Bookmark | What happens when you click it |
|---|---|
| ▶ START Transaction | Asks for a name → silently fires `START-TXX.invalid` in the background → you stay on your page |
| ■ END Transaction | Asks for a name → silently fires `END-TXX.invalid` in the background → you stay on your page |

> The requests fail (`.invalid` domain doesn't exist) — **that is intentional and expected**.

---

## Marker URL Examples (for reference / address bar fallback)

| What you want | Type this in address bar |
|---|---|
| Start transaction 1 (Login) | `https://START-T01-Login.invalid` |
| End transaction 1 (Login) | `https://END-T01-Login.invalid` |
| Start transaction 2 (Search) | `https://START-T02-Search.invalid` |
| End transaction 2 (Search) | `https://END-T02-Search.invalid` |

> Use the address bar method only if bookmarklets are blocked by your organisation.

---

## Tool — Step by Step

```
1  Open VuGen-Recorder.html in your browser
2  Drop your .har file onto the grey area (or click Browse)
3  Review the coloured transaction banners in the table
4  Click "⬇ Download All Files" at the bottom
5  Copy the downloaded files to your LRE project folder
```

---

## What Each Downloaded File Is For

| File | Where it goes | Protocol |
|---|---|---|
| `Action.c` | Your VuGen project folder | Web HTTP/HTML |
| `vuser_init.c` | Your VuGen project folder | Web HTTP/HTML |
| `vuser_end.c` | Your VuGen project folder | Web HTTP/HTML |
| `globals.h` | Your VuGen project folder | Web HTTP/HTML |
| `main.js` | Your DevWeb script folder | DevWeb |

---

## Toolbar Filters

| Checkbox | What it hides |
|---|---|
| Static Assets | Images, fonts, CSS, JS files |
| Analytics/Ads | Google Analytics, Facebook trackers |
| OPTIONS | Browser pre-flight requests |

> Uncheck to include something in the script. Script updates instantly.

---

## Manual Transaction (forgot bookmarklets/markers?)

```
1  Click "☑ Select Mode" button → turns blue
2  Click each request row that belongs to the transaction
3  Click "+ Create Transaction"
4  Type transaction name → click Create
5  Repeat for all transactions
```

---

## Status Code Guide

| Colour | Code | Meaning |
|---|---|---|
| 🟩 Green | 200–299 | Success — include in script |
| 🔵 Blue | 300–399 | Redirect — usually fine |
| 🟧 Orange | 400–499 | Client error — check if expected |
| 🟥 Red | 500–599 | Server error — may need investigation |
| ⬜ Grey | 0 | Failed/blocked — markers show as 0 |

---

*VuGen HAR Script Generator v1.0 — LRE Admin Tool*
