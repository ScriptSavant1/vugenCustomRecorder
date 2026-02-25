# VuGen HAR Script Generator — User Guide

> **For:** LRE Users who need to record performance test scripts
> **Replaces:** VuGen Recorder (which is blocked by antivirus/org policy)
> **Skill level required:** Basic browser usage — no technical knowledge needed

---

## What Is This Tool?

This tool helps you create **performance test scripts** for LoadRunner Enterprise (LRE) without needing VuGen's built-in recorder. It works in three simple stages:

```
STAGE 1              STAGE 2                    STAGE 3
─────────            ──────────────────         ──────────────────────
Record your      →   Import the recording   →   Download ready-to-use
actions in           into our tool               scripts for LRE
your browser         and group into              (2 formats generated
using F12            transactions                automatically)
```

**Output:** Two script files are created automatically:
- **`Action.c`** — for the Web HTTP/HTML protocol in VuGen/LRE
- **`main.js`** — for the DevWeb protocol in VuGen/LRE

---

## Before You Start — Checklist

- [ ] You have access to the application you want to test
- [ ] You have the file: **`VuGen-Recorder.html`** (from your LRE admin team)
- [ ] You know the names you want to give your transactions (e.g., Login, Search, Checkout)
- [ ] You are using **Google Chrome**, **Microsoft Edge**, or **Firefox**

> **Note:** No installation is required. No admin rights needed. The tool is a single HTML file — just open it in your browser.

---

## Part 1: Planning Your Transactions

Before recording, decide how to split your user journey into **transactions**.

A **transaction** = one logical user action that you want to measure separately.

**Example — An e-commerce test scenario:**

| Transaction Name | What the user does |
|---|---|
| `T01_Homepage` | Open the website homepage |
| `T02_Login` | Enter username/password and log in |
| `T03_Search` | Search for a product |
| `T04_AddToCart` | Add item to shopping cart |
| `T05_Checkout` | Complete the purchase |

> **Naming tip:** Always use the format `T01_Name`, `T02_Name`, etc.
> - Use only letters, numbers, and underscores
> - No spaces allowed
> - Examples: `T01_Login`, `T02_Product_Search`, `T03_Add_To_Cart`

---

## Part 2: One-Time Bookmarklet Setup

The easiest way to mark transactions is with **browser bookmarklets** — two special bookmarks that send a hidden signal to your recording without taking you away from your application page.

**You only need to do this setup once.**

### Step 1 — Show Your Bookmarks Bar

Your browser's Bookmarks Bar is the strip of bookmarks just below the address bar. If it is hidden:

- **Chrome/Edge:** Press `Ctrl+Shift+B`
- **Firefox:** Press `Ctrl+Shift+B`

The bar will appear with any existing bookmarks.

---

### Step 2 — Open the Tool and Drag the Bookmarklets

1. Open **`VuGen-Recorder.html`** in your browser
2. You will see two coloured buttons in the green **"Set Up Bookmarklets"** box:
   - A **green "▶ START Transaction"** button
   - A **red "■ END Transaction"** button
3. **Drag** each button from the tool onto your Bookmarks Bar

```
   ┌─────────────────────────────────────────────────────────────────┐
   │  Bookmarks Bar:  [Bookmarks] [Other Sites]                      │
   │  ↑ Drag the buttons here ↑                                      │
   ├─────────────────────────────────────────────────────────────────┤
   │  VuGen HAR Script Generator                                     │
   │                                                                 │
   │  🔖 Set Up Bookmarklets                                         │
   │  ┌────────────────────┐  ┌─────────────────────┐               │
   │  │ ▶ START Transaction│  │ ■ END Transaction   │               │
   │  └────────────────────┘  └─────────────────────┘               │
   │           ↑                       ↑                             │
   │      Drag this up            Drag this up                       │
   └─────────────────────────────────────────────────────────────────┘
```

4. After dragging, both bookmarks will appear in your Bookmarks Bar — they are now ready to use.

> **If the buttons don't drag:** Some browsers require you to right-click each link → **"Bookmark this link"** to add it manually.

---

## Part 3: Recording Your Application

### Step 3 — Open Developer Tools

1. Open **Google Chrome** or **Microsoft Edge**
2. Press **`F12`** on your keyboard
3. Click on the **"Network"** tab
4. Click the **🚫 (Clear)** button to remove any existing entries
5. Make sure the **red circle ⏺** (Record button) is active

> **Important:** Keep the Developer Tools panel open for the entire recording. Do not close it.

---

### Step 4 — Mark the Start of Your First Transaction

When you are ready to begin recording a transaction:

1. Navigate to the starting point of your transaction in the application
2. Click the **▶ START Transaction** bookmark in your Bookmarks Bar
3. A small popup appears — type your transaction name (e.g. `T01_Login`) and click **OK**

```
   ┌──────────────────────────────────────────┐
   │  Transaction name (e.g. T01_Login):      │
   │  ┌──────────────────────────────────┐    │
   │  │ T01_Login                        │    │
   │  └──────────────────────────────────┘    │
   │                    [Cancel]  [OK]         │
   └──────────────────────────────────────────┘
```

4. The popup closes and you stay on your application page — the marker is recorded silently in the background

> **You will not see any visible change on your screen — that is normal.** The bookmarklet fires a silent background request that gets captured in the Network tab.

---

### Step 5 — Perform Your Transaction Steps

Now perform all the browser steps that belong to this transaction.

**You can navigate completely naturally:**
- Click links and buttons in the application
- Fill in forms and submit them
- Wait for pages to load

There is no restriction on how you navigate. You do not need to type URLs in the address bar.

---

### Step 6 — Mark the End of Your Transaction

When you have finished all steps for this transaction:

1. Click the **■ END Transaction** bookmark in your Bookmarks Bar
2. Type the **same transaction name** you used for START (e.g. `T01_Login`) → click **OK**
3. You stay on your current page — continue directly with the next transaction

---

### Step 7 — Continue to the Next Transaction

Immediately after clicking END:

1. Click **▶ START Transaction** again
2. Type the next transaction name (e.g. `T02_Search`) → click **OK**
3. Perform the steps for that transaction
4. Click **■ END Transaction** → type `T02_Search` → click **OK**
5. Repeat for all transactions

```
What this looks like in the Network tab:
──────────────────────────────────────────────────────────
  START-T01_Login.invalid          ← Silent marker (failed)
  GET   /login                200
  POST  /api/authenticate     200
  GET   /api/user/profile     200
  END-T01_Login.invalid            ← Silent marker (failed)
  START-T02_Search.invalid         ← Silent marker (failed)
  GET   /search?q=laptop      200
  GET   /api/products         200
  END-T02_Search.invalid           ← Silent marker (failed)
──────────────────────────────────────────────────────────
```

| ✅ Allowed during recording | ❌ Never do during recording |
|---|---|
| Click links and buttons in the app | Close/reopen Developer Tools (F12) |
| Fill forms and submit them | Refresh the page (F5 / Ctrl+R) |
| Navigate anywhere by clicking | Use bookmarklets on the VuGen-Recorder.html page itself |

---

### Step 8 — Export the HAR File

Once you have completed all your transactions:

1. In the **Network tab**, right-click on any row in the request list
2. Click **"Save all as HAR with content"**

```
   ┌──────────────────────────────────┐
   │  Copy link address               │
   │  Open in new tab                 │
   │  Clear                           │
   │  ─────────────────────────────── │
   │  Save all as HAR with content ← │
   │  ─────────────────────────────── │
   │  Block request URL               │
   └──────────────────────────────────┘
```

3. A **Save file dialog** appears
4. Choose a folder you can easily find (e.g., Desktop or Downloads)
5. Give it a meaningful name: e.g., `MyApp_Recording.har`
6. Click **Save**

> **Firefox users:** Click the ⬇ download icon in the Network toolbar, then "Save All As HAR"

---

## Part 4: Using the Script Generator Tool

### Step 9 — Load Your HAR File

1. Open **`VuGen-Recorder.html`** in your browser
2. **Drag and drop** your `.har` file onto the grey drop area, or click **"Browse HAR File"** to find it

The tool will automatically:
- ✅ Read all the recorded requests
- ✅ Detect your transaction start/end markers
- ✅ Filter out noise (images, fonts, analytics trackers)
- ✅ Generate both script files instantly

---

### Step 10 — Review the Request Table

After loading, you will see a table of all your recorded requests:

```
┌────────────────────────────────────────────────────────────────┐
│  Filters: ☑ Static Assets  ☑ Analytics/Ads  ☑ OPTIONS        │
├────┬────────┬────────────────────────┬────────┬──────┬─────────┤
│  # │ Method │ URL                    │ Status │ Size │ Time    │
├────┴────────┴────────────────────────┴────────┴──────┴─────────┤
│  ▶ START: T01_Login   ← Blue banner marks transaction start   │
├────┬────────┬────────────────────────┬────────┬──────┬─────────┤
│  3 │ GET    │ /login                 │  200   │ 12KB │ 1200ms  │
│  4 │ POST   │ /api/authenticate      │  200   │ 1KB  │  450ms  │
│  5 │ GET    │ /api/user/profile      │  200   │ 2KB  │  380ms  │
├────┴────────┴────────────────────────┴────────┴──────┴─────────┤
│  ■ END: T01_Login     ← Blue banner marks transaction end     │
└────────────────────────────────────────────────────────────────┘
```

---

### Step 11 — Download Your Scripts

Click **"⬇ Download All Files"** at the bottom of the screen.

**Files you will get:**

| File | Protocol | Purpose |
|---|---|---|
| `Action.c` | Web HTTP/HTML | Main script with all your requests and transactions |
| `vuser_init.c` | Web HTTP/HTML | Initialization file |
| `vuser_end.c` | Web HTTP/HTML | Cleanup file |
| `globals.h` | Web HTTP/HTML | Required header file |
| `main.js` | DevWeb | Complete DevWeb protocol script |

---

### Step 12 — Copy Scripts to Your LRE Project

**For Web HTTP/HTML protocol:**
1. Go to your VuGen/LRE project folder
2. Copy `Action.c`, `vuser_init.c`, `vuser_end.c`, and `globals.h` into it

**For DevWeb protocol:**
1. Go to your DevWeb script folder
2. Copy `main.js` into it

> **Ask your LRE admin** if you are unsure where your project folder is located.

---

## Part 5: Filters Explained

The tool automatically removes "noise" from your recording.

| Filter | What it removes |
|---|---|
| **Static Assets** | Images, fonts, CSS, JavaScript files |
| **Analytics/Ads** | Google Analytics, DoubleClick, Facebook trackers |
| **OPTIONS** | Browser pre-flight check requests |

**To include something that was filtered out:** Uncheck the relevant filter — the table and scripts update instantly.

---

## Part 6: If You Forgot to Use Bookmarklets

Don't worry — you can still group requests manually using **Select Mode**.

1. Load your HAR file into the tool
2. Click the **"☑ Select Mode"** button in the toolbar (it turns blue)
3. Click on each request row that belongs to your first transaction
4. Click **"+ Create Transaction"**
5. Type your transaction name (e.g. `T01_Login`) → click **Create**
6. Repeat for each transaction

---

## Summary — The Complete Workflow

```
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│  ONE-TIME SETUP                                                          │
│  Open VuGen-Recorder.html → drag ▶ START and ■ END to Bookmarks Bar    │
│                                                                          │
│  FOR EACH RECORDING SESSION                                              │
│  1. Open browser → F12 → Network tab → clear entries                    │
│  2. Navigate to the start of your application                            │
│  3. For each transaction:                                                │
│     a. Click ▶ START Transaction → type name → OK                       │
│     b. Browse application naturally (click buttons, fill forms, etc.)   │
│     c. Click ■ END Transaction → type same name → OK                    │
│  4. Network tab → Right-click → Save all as HAR with content            │
│  5. Open VuGen-Recorder.html → Drop the HAR file                        │
│  6. Review the request table                                             │
│  7. Click "⬇ Download All Files"                                        │
│  8. Copy files to your LRE project folder                               │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## Need Help?

Contact your **LRE Admin Team** for:
- Getting the `VuGen-Recorder.html` file
- Understanding where to copy your script files
- Uploading scripts to LRE

---

*VuGen HAR Script Generator v1.0 — LRE Admin Tool*
