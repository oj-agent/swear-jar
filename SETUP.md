# Swear Jar — Setup & Deployment Guide

## Quick Start (Local Test)

```bash
cd ~/.openclaw/workspace/projects/swear-jar
python3 -m http.server 8080
# Open http://localhost:8080
```

---

## GitHub Pages Deployment

```bash
cd ~/.openclaw/workspace/projects/swear-jar
git add . && git commit -m "Update" && git push
```

GitHub Pages auto-deploys on push. The service worker updates on next visit.

**Live URL:** https://oj-agent.github.io/swear-jar/

### Install as PWA on Android (Pixel 10 Pro)

1. Open Chrome → go to the URL above
2. Tap ⋮ menu → **Add to Home screen** → **Install**

### Install as PWA on iPhone

1. Open Safari → go to the URL above
2. Tap **Share** → **Add to Home Screen** → **Add**

---

## Google Sheets Sync — Full Bidirectional Setup

The app now uses **bidirectional sync**: the Google Sheet is the single source of truth. Both phones push their local hits AND pull the complete log from the Sheet, then merge + recompute totals locally. This means:

- Totals are always consistent across both phones after a sync
- The 24h free-hit timer is also synced (derived from the log, not device clock)
- No duplicate entries (deduplication by timestamp)

### Step 1: Update the Apps Script

The sync now requires a **GET endpoint** in the Apps Script (in addition to the existing POST). You need to replace the old script with the version below.

1. Open your Google Sheet
2. Go to **Extensions → Apps Script**
3. **Delete all existing code**
4. Paste the following:

```javascript
const SPREADSHEET_ID = SpreadsheetApp.getActiveSpreadsheet().getId();

function getOrCreateLogSheet(ss) {
  let sheet = ss.getSheetByName('Log');
  if (!sheet) {
    sheet = ss.insertSheet('Log');
    sheet.appendRow(['Timestamp', 'Person', 'Amount']);
  }
  return sheet;
}

/**
 * POST: Receives an array (or single object) of hit entries.
 * Appends new entries to Log sheet, skipping duplicates by timestamp.
 */
function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const logSheet = getOrCreateLogSheet(ss);

    // Load existing timestamps to avoid duplicates
    const existing = logSheet.getDataRange().getValues();
    const existingTs = new Set(existing.slice(1).map(r => String(r[0])));

    const entries = Array.isArray(data) ? data : [data];
    for (const entry of entries) {
      if (!entry.ts || !entry.person) continue;
      if (!existingTs.has(entry.ts)) {
        logSheet.appendRow([entry.ts, entry.person, Number(entry.amount) || 0]);
        existingTs.add(entry.ts);
      }
    }

    return ContentService
      .createTextOutput(JSON.stringify({ status: 'ok' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'error', message: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

/**
 * GET: Returns all log entries as JSON.
 * Used by the app to pull the full shared log and recompute state.
 */
function doGet(e) {
  try {
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);
    const logSheet = getOrCreateLogSheet(ss);
    const data = logSheet.getDataRange().getValues();
    const rows = data
      .slice(1)                         // skip header row
      .filter(r => r[0])                // skip blank rows
      .map(r => ({
        ts:     String(r[0]),
        person: String(r[1]),
        amount: Number(r[2]) || 0
      }));
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'ok', log: rows }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch (err) {
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'error', message: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}
```

### Step 2: Re-deploy as a New Version

> **Important:** You must deploy a *new version* — editing the code alone doesn't update the live endpoint.

1. Click **Deploy → Manage deployments** (or **New deployment** if this is your first time)
2. Click the ✏️ edit icon next to your existing deployment
3. Under **Version**, select **New version**
4. Click **Deploy**
5. Copy the **Web App URL** (it stays the same URL, just updated)

> If you've never deployed before: Click **Deploy → New deployment** → Type: **Web app** → Execute as: **Me** → Who has access: **Anyone** → Deploy → copy URL.

### Step 3: Enter the URL on Each Phone

1. Open Swear Jar app → tap ⚙️ Settings
2. Paste the Web App URL in **Apps Script Web App URL**
3. Tap **Save & Test Sheets** — should show "✅ Connected! GET sync enabled."

> Both phones need the same URL. Settings are stored per device in localStorage.

---

## How Sync Works (New Logic)

Each time you tap Hit or press ☁️:

1. **Push** — any unsynced local hits are sent to the Sheet (POST, no-cors)
2. **Pull** — all log rows are fetched from the Sheet (GET)
3. **Merge** — local + remote entries are unioned, deduplicated by timestamp
4. **Recompute** — totals and the 24h free-hit timer are computed from the merged log

The app also **auto-syncs on open**, so each device immediately picks up the other's state.

---

## How the 24-Hour Free Hit Works

- Each person gets **one FREE hit per 24-hour rolling window**
- The clock starts from the timestamp of the free hit, not midnight
- Subsequent hits within the same 24h window are +$10 each
- After sync, both phones see the same free-hit timer (derived from the shared log)

---

## Architecture Notes

- **State storage:** `localStorage` — persists across sessions on the same device
- **Source of truth:** Google Sheet Log tab — all devices derive state from it
- **Offline:** Service worker caches app; hits are saved locally and synced when back online
- **Sound:** Web Audio API synthesized cha-ching on paid hits
- **Deduplication:** Entries are matched by ISO timestamp; same hit won't appear twice

---

## Updating the App

```bash
cd ~/.openclaw/workspace/projects/swear-jar
git add . && git commit -m "Update" && git push
```
