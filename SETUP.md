# Swear Jar — Setup & Deployment Guide

## Quick Start (Local Test)

The app is a single `index.html` file. You can open it directly from a local server:

```bash
cd ~/.openclaw/workspace/projects/swear-jar
python3 -m http.server 8080
# Open http://localhost:8080 in browser
```

Or deploy to GitHub Pages (recommended for phone access):

---

## GitHub Pages Deployment (Recommended)

### 1. Create a GitHub repo

```bash
cd ~/.openclaw/workspace/projects/swear-jar
git init
git add .
git commit -m "Initial Swear Jar PWA"
gh repo create swear-jar --public --source=. --push
```

### 2. Enable GitHub Pages

Go to the repo on GitHub → **Settings** → **Pages** → **Source: main branch / root** → Save.

Your URL will be: `https://YOUR-USERNAME.github.io/swear-jar/`

### 3. Install as PWA on Android (Pixel 10 Pro)

1. Open Chrome → navigate to your GitHub Pages URL
2. Tap the three-dot menu → **Add to Home screen**
3. Tap **Install** — it will appear as a full-screen app icon

### 4. Install as PWA on iPhone

1. Open Safari → navigate to the URL
2. Tap the **Share** icon (square with arrow) → **Add to Home Screen**
3. Tap **Add**

> **Note:** Safari on iOS requires HTTPS for PWA features. GitHub Pages provides this automatically.

---

## Google Sheets Backend (Optional Sync)

This lets both phones see the same log and totals.

### Step 1: Create the Google Sheet

1. Go to [sheets.google.com](https://sheets.google.com) and create a new sheet named **Swear Jar**
2. Share the sheet with both Robert and Tingting (so they can view it)
3. Set up two sheets/tabs:
   - **`Log`** — will store hit events (auto-created by script)
   - **`Totals`** — will store current totals (auto-created by script)

### Step 2: Deploy the Apps Script

1. In the Google Sheet, go to **Extensions → Apps Script**
2. Delete any existing code, paste the following:

```javascript
const SPREADSHEET_ID = SpreadsheetApp.getActiveSpreadsheet().getId();

function doPost(e) {
  try {
    const data = JSON.parse(e.postData.contents);
    const ss = SpreadsheetApp.openById(SPREADSHEET_ID);

    if (data.action === 'logHit') {
      // Append to Log sheet
      let logSheet = ss.getSheetByName('Log');
      if (!logSheet) {
        logSheet = ss.insertSheet('Log');
        logSheet.appendRow(['Timestamp', 'Person', 'Amount', 'Total Robert', 'Total Tingting']);
      }
      logSheet.appendRow([
        data.ts,
        data.person,
        data.amount,
        data.totalRobert,
        data.totalTingting
      ]);

      // Update Totals sheet
      let totSheet = ss.getSheetByName('Totals');
      if (!totSheet) {
        totSheet = ss.insertSheet('Totals');
        totSheet.appendRow(['Person', 'Total', 'Last Updated']);
      }

      // Find or create rows for each person
      const totData = totSheet.getDataRange().getValues();
      let robertRow = -1, tingtingRow = -1;
      for (let i = 1; i < totData.length; i++) {
        if (totData[i][0] === 'Robert')  robertRow = i + 1;
        if (totData[i][0] === 'Tingting') tingtingRow = i + 1;
      }
      const now = new Date().toISOString();
      if (robertRow === -1)  { totSheet.appendRow(['Robert', data.totalRobert, now]); }
      else { totSheet.getRange(robertRow, 2, 1, 2).setValues([[data.totalRobert, now]]); }
      if (tingtingRow === -1) { totSheet.appendRow(['Tingting', data.totalTingting, now]); }
      else { totSheet.getRange(tingtingRow, 2, 1, 2).setValues([[data.totalTingting, now]]); }
    }

    return ContentService
      .createTextOutput(JSON.stringify({ status: 'ok' }))
      .setMimeType(ContentService.MimeType.JSON);
  } catch(err) {
    return ContentService
      .createTextOutput(JSON.stringify({ status: 'error', message: err.toString() }))
      .setMimeType(ContentService.MimeType.JSON);
  }
}

function doGet(e) {
  // Health check / ping
  return ContentService
    .createTextOutput(JSON.stringify({ status: 'ok', message: 'Swear Jar backend running' }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

3. Click **Deploy → New deployment**
4. Select type: **Web app**
5. Description: `Swear Jar v1`
6. Execute as: **Me**
7. Who has access: **Anyone** (required for the app to post from phones)
8. Click **Deploy** → copy the **Web App URL**

### Step 3: Enter the URL in the App

1. Open the Swear Jar app on your phone
2. Tap ⚙️ (Settings) → paste the Web App URL in the **Apps Script Web App URL** field
3. Tap **Save & Test Sheets**

> **Important:** Both phones need to enter the same URL in settings. Settings are stored per device in localStorage, so you'll need to configure this on each phone separately.

---

## How the 24-Hour Free Hit Works

- Each person gets **one FREE hit per 24-hour rolling window**
- The 24-hour clock starts from their **first hit of the day**, not midnight
- Example: Robert hits at 3pm Tuesday → next free hit available at 3pm Wednesday
- Subsequent hits within the same 24h window are **+$10 each**
- The button shows "FREE" (green) or "$10" (red) based on current status
- A countdown shows how long until the free hit resets

---

## PWA Icons

The manifest references `icon-192.png` and `icon-512.png`. To add real icons:

1. Create a 512×512 PNG with your jar icon
2. Resize to 192×192 and 512×512
3. Place both in the `swear-jar/` folder

Without these, the app still installs — it just uses the browser's default icon.

Quick way to generate icons using the built-in emoji:

```bash
# If you have ImageMagick:
convert -background '#1a1a2e' -fill white -gravity Center \
  -font "Apple Color Emoji" -pointsize 400 \
  label:'💰' -resize 512x512 icon-512.png
convert icon-512.png -resize 192x192 icon-192.png
```

---

## Architecture Notes

- **State storage:** `localStorage` — persists across browser sessions on the same device
- **Sync:** POST to Google Apps Script on every hit; no-cors mode (opaque response)
- **Offline:** Service worker caches `index.html`, app works fully offline
- **Sound:** Web Audio API — no external files; synthesized cha-ching on paid hits
- **Two-device sync:** Both phones write to the same Sheet but read state from their own localStorage (keep them roughly in sync; the Sheet is the source of truth log)

---

## Updating the App

To push updates:

```bash
cd ~/.openclaw/workspace/projects/swear-jar
git add . && git commit -m "Update" && git push
```

GitHub Pages auto-deploys on push. The service worker will update on next visit.
