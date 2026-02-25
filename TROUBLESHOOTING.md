# VuGen HAR Script Generator — Troubleshooting Guide

---

## Bookmarklet Issues

---

### ❓ I can't drag the bookmarklet buttons to my Bookmarks Bar

**Check that your Bookmarks Bar is visible first.**

- **Chrome/Edge:** Press `Ctrl+Shift+B` to show it
- **Firefox:** Press `Ctrl+Shift+B` to show it

If the bar is now visible, try dragging again. If dragging still doesn't work:

1. Right-click the **▶ START Transaction** button in the tool
2. Select **"Bookmark this link"** (Chrome/Edge) or **"Bookmark This Link"** (Firefox)
3. In the dialog that appears, choose **Bookmarks Bar** as the folder → click Save
4. Repeat for the **■ END Transaction** button

---

### ❓ I clicked the bookmarklet button on the VuGen tool page and it just showed an alert

**That is intentional.** The buttons on the VuGen-Recorder.html page are for *dragging to your Bookmarks Bar only*. Once you have dragged them to the bar, they appear there permanently and can be clicked during your recording sessions in any other browser tab.

---

### ❓ I clicked the START or END bookmark during recording but nothing visible happened

**That is correct behaviour.** The bookmarklet fires a hidden background request (you stay on your application page). The request is captured silently in the Network tab. You will not see any popup, page change, or error — that is intentional.

**To verify it worked:** Look in the Network tab. You should see a new row with `START-T01_Name.invalid` (status 0 or failed) — that confirms the marker was recorded.

---

### ❓ The bookmarklet prompt appeared but the Network tab shows no marker entry

**Causes and fixes:**

| Cause | Fix |
|---|---|
| Developer Tools was not open when you clicked | Always press F12 and open the Network tab *before* clicking any bookmarklet |
| You clicked Cancel in the prompt | Click the bookmark again and type a name before clicking OK |
| Browser blocked the fetch request | Try in Chrome or Edge (most compatible) |

---

### ❓ My organisation's group policy blocks `javascript:` bookmarks

**Use the Address Bar Method instead.** See the collapsible "Address Bar Method" section at the bottom of the VuGen-Recorder.html welcome page for step-by-step instructions.

In short: type `https://START-T01-Login.invalid` in the address bar and press Enter (error page is expected), then navigate back to your application and continue.

---

## Recording Issues

---

### ❓ I typed the START/END marker URL but the error page looks different

**This is fine.** The browser error page appearance depends on your browser version and organisation settings. As long as you pressed Enter and the browser attempted to load the URL, the marker was recorded. Check the Network tab — you should see the `.invalid` request listed (even though it failed).

---

### ❓ My Network tab is empty when I open it

**Cause:** The Network tab only records requests that happen while it is open.

**Fix:**
1. Close Developer Tools (F12)
2. Go back to your application's starting page
3. Press F12 again to open Developer Tools
4. Click the Network tab
5. Click 🚫 to clear entries
6. Now start your recording from the beginning

---

### ❓ I closed the Developer Tools accidentally during recording

**What this means:** All recordings in that session are lost.

**Fix:** You need to start over.
1. Press F12 to reopen Developer Tools
2. Click 🚫 to clear entries
3. Begin your recording again from the START marker/bookmarklet

---

### ❓ The Network tab shows thousands of requests — is that normal?

**Yes, this is normal.** Modern websites make many requests in the background. Our tool automatically filters out the noise (images, fonts, analytics, ads). When you load your `.har` file into the tool, most of these will be hidden automatically.

---

### ❓ I forgot to put markers for one of my transactions

**Option 1 — Re-record:** Start the recording again (usually takes only a few minutes)

**Option 2 — Use Select Mode in the tool:**
1. Load the HAR file into the tool
2. Click "☑ Select Mode"
3. Click each request row that belongs to the missing transaction
4. Click "+ Create Transaction"
5. Enter the transaction name
6. The transaction is created manually

---

### ❓ I need to navigate away from the app to place a marker but I don't know the next URL

**Use the bookmarklets.** This is exactly the problem they solve.

When you click the **■ END Transaction** or **▶ START Transaction** bookmark, the browser fires the marker request in the background and you **stay on your current page**. You never need to leave your application or know any URL.

If you are using the address bar method and ended up on an error page: type your application's main URL in the address bar (e.g. `https://yourapp.com`) to get back in, then continue from there.

---

### ❓ My start/end markers are not being detected by the tool

**Check your marker URL format.** The tool looks for this exact pattern:

```
✅  https://START-T01-Login.invalid       ← Correct
✅  https://END-T01-Login.invalid         ← Correct
✅  https://START-T01_Login.invalid       ← Also works (underscore)

❌  http://START-T01-Login.invalid        ← Wrong (must be https)
❌  https://START T01 Login.invalid       ← Wrong (spaces not allowed)
❌  https://STARTТ01Login.invalid         ← Wrong (missing hyphen after START)
❌  https://START-T01-Login.com           ← Wrong (must be .invalid)
```

> **Note:** Bookmarklets automatically format the name correctly — this issue mainly affects the manual address bar method.

**If your markers still aren't detected**, use **Select Mode** to group requests manually (see previous answer).

---

### ❓ I accidentally included requests I didn't want

**The filters** at the top of the tool handle most noise automatically. If something specific slipped through:
- The request will be in the generated script
- You can manually delete it from the downloaded file using any text editor (Notepad, Notepad++)
- Ask your LRE admin for help editing the script if needed

---

## Tool Issues

---

### ❓ The tool doesn't open when I double-click the HTML file

**Common causes and fixes:**

| Cause | Fix |
|---|---|
| No default browser set | Right-click the file → Open with → Chrome or Edge |
| File is on a network drive with restrictions | Copy the file to your Desktop and open it from there |
| Browser security blocked it | In Chrome: click the shield icon in address bar → Allow |

---

### ❓ I dropped the HAR file but nothing happened

**Check these things:**
1. Is the file actually a `.har` file? (File name should end in `.har`)
2. Try the **Browse** button instead of drag-and-drop
3. Make sure the file is not open in another program
4. Try in a different browser (Chrome, Edge, or Firefox)

---

### ❓ The tool loaded the file but shows "No entries"

**Cause:** The HAR file might be empty, corrupted, or from an unsupported source.

**Fix:**
1. Re-record your session
2. Make sure you right-clicked in the Network tab (not the Console or Sources tab) when saving
3. Make sure you selected **"Save all as HAR with content"** (not just "Save as HAR")

---

### ❓ All filters are checked but I still see too many requests

You can manually exclude domains:
1. **Uncheck** filters that are hiding too much
2. Look at the URL column — identify the domain causing the noise
3. Use **Select Mode** to select only the requests you actually want
4. Create transactions from those selected requests

---

### ❓ The generated script is empty / only shows the function wrapper

**Cause:** All requests were filtered out.

**Fix:**
1. Try **unchecking** all three filter checkboxes (Static Assets, Analytics/Ads, OPTIONS)
2. If requests appear now, gradually turn filters back on one at a time
3. If still empty, the HAR file may not contain actual HTTP requests (rare)

---

### ❓ My transaction name has a hyphen in it and it changed to underscore

**This is intentional.** The VuGen C language and DevWeb JavaScript both do not allow hyphens in names. The tool automatically converts hyphens to underscores. The bookmarklet does the same conversion automatically.

```
You typed:   T01-Login
Tool shows:  T01_Login   ← automatically fixed
```

---

## Download Issues

---

### ❓ When I click Download, nothing happens

**Try these fixes in order:**
1. Check your browser's Downloads folder — the file may have downloaded silently
2. Check if your browser is blocking downloads — look for a blocked download notification near the address bar
3. Try a different browser (Chrome usually works best for downloads)
4. Click on the script preview tab first (Action.c), then click Download

---

### ❓ The downloaded file opens in Notepad but looks garbled

**This is normal for `.c` files on Windows.** The file is correct. When you copy it to your LRE project, VuGen/LRE will read it correctly. The content only looks strange in Notepad because of how it handles line endings.

> **Tip:** Use **Notepad++** (free download) if you need to view or edit these files — it handles them much better than regular Notepad.

---

### ❓ I can't find the downloaded files

On Windows, browser downloads typically go to:
```
C:\Users\[YourName]\Downloads\
```

Or check:
- Look at the bottom of your browser window for a download notification
- Click the **⬇** downloads icon in your browser toolbar
- Press **Ctrl + J** in Chrome/Edge to open the downloads page

---

## Script Issues

---

### ❓ LRE says the script has errors when I upload it

**Most common causes:**

1. **Body text contains special characters** — The request body may contain characters that break the C string format. Ask your LRE admin to review the `Action.c` file.

2. **File encoding issue** — Make sure you're copying the file as-is, not copy-pasting the content manually.

3. **Missing files** — Make sure you copied ALL four Web HTTP/HTML files: `Action.c`, `vuser_init.c`, `vuser_end.c`, and `globals.h`.

---

### ❓ The script runs in LRE but transactions show wrong timings

**Cause:** The `lr_think_time(3)` added between transactions may be too short or too long for your application.

**Fix:** Open `Action.c` in Notepad++ and change the number `3` in each `lr_think_time(3)` line to the appropriate think time in seconds for your application.

---

### ❓ The script only has GET requests — my POST requests are missing

**Cause:** POST request bodies may have been filtered or the body content was binary/encoded.

**Check:**
1. Uncheck all filters and reload
2. Look in the table for your POST request — if it shows as dimmed, it was filtered
3. If it appears, check if the body shows in the script preview

---

### ❓ My session cookies / authentication tokens are hardcoded in the script

**This is expected for the first version of the script.** The recorded script captures the actual values used during recording. For load testing with multiple users, these need to be parameterised.

**Next step:** Ask your LRE admin to help you:
- Replace hardcoded tokens with parameters using `web_reg_save_param` (Web HTTP/HTML)
- Replace hardcoded tokens with extractors (DevWeb)

---

## Getting More Help

| Who to contact | When |
|---|---|
| **LRE Admin Team** | Questions about uploading scripts, project folders, LRE settings |
| **Team Lead** | Questions about test scenario design, transaction planning |
| **IT Helpdesk** | Browser issues, file access issues, network restrictions |

---

## Quick Self-Check Before Contacting Support

Before reaching out, please check:
- [ ] Are the ▶ START and ■ END bookmarks in your Bookmarks Bar?
- [ ] Was Developer Tools (F12) open for the entire recording?
- [ ] Did you save as "HAR with content" (not just "HAR")?
- [ ] Did you download all 5 files (not just Action.c)?
- [ ] Did you copy ALL downloaded files to the project folder?

---

*VuGen HAR Script Generator v1.0 — LRE Admin Tool*
