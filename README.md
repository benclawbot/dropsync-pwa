# DropSync PWA

**2-way sync** between a local folder on your Android phone and a Dropbox folder. No server, no account, no monthly fee. Everything runs in your browser.

---

## What it does

- **Pick a Dropbox folder** via the built-in file browser
- **Pick a local folder** on your phone using Android's native folder picker
- **Choose file filter** — all files or Markdown (`.md`) only
- **Start Sync** — performs a full 2-way diff (last-write-wins)
- **Keeps watching** — after the initial sync, monitors for file changes and syncs them immediately
- **Works offline** — the app UI loads offline; sync requires network

---

## Setup Instructions

### Step 1: Host the app (one-time)

DropSync needs to be served over HTTPS with a proper origin (required for Dropbox OAuth and File System Access API). You have options:

**Option A — Use a free static host (recommended):**

1. Create a GitHub Gist or repo with these 3 files: `index.html`, `manifest.json`, `sw.js`
2. Use **Cloudflare Pages** (free) — connect your repo and deploy
3. Or use **Netlify Drop** — drag-and-drop the folder at https://app.netlify.com/drop

**Option B — Local serve (for testing on your computer):**
```bash
cd dropsync-pwa
npx serve .
```

### Step 2: Create a Dropbox App (one-time)

1. Go to https://www.dropbox.com/developers/create
2. Choose **Scoped Access**, **Full Dropbox**
3. Give it any name (e.g. "My DropSync")
4. In the app's **Settings**, find **OAuth2 Redirect URIs** and add:
   ```
   https://benclawbot.github.io/dropsync-pwa/
   ```
   (Replace with your actual deployed URL — no path needed, the root is fine)
5. Copy the **App Key**

### Step 3: Open DropSync on your Android phone

1. Open Chrome on your Android device
2. Navigate to your deployed URL
3. Tap **"Connect"** and enter your Dropbox App Key when prompted
4. Authorize the app in the Dropbox popup
5. Tap **"Select"** on the Local Folder card — pick any folder in internal storage or SD
6. Tap **"Browse"** on the Dropbox card — navigate to your Obsidian vault folder
7. Choose your file filter (All / Markdown only)
8. Tap **Start Sync**

---

## Packaging as an Android APK

Once deployed, package the PWA as an APK using PWABuilder — no sign-in, no build tools needed:

1. Go to **https://www.pwabuilder.com/**
2. Paste your deployed URL (e.g. `https://my-dropsync.pages.dev/`)
3. Click **Start**
4. On the Android tile, click **Package for stores**
5. Click **Download Starter Project** (or the APK directly if available)
6. Unzip and run the signing step:

```bash
# Align the APK (one-time setup, then sign)
zipalign -v -p 4 my-app.apk my-app-aligned.apk

# Create a debug keystore if you don't have one
keytool -genkey -v -keystore my-release.jks -keyalg RSA -keysize 2048 \
  -validity 10000 -alias my-alias

# Sign the APK
apksigner sign --ks my-release.jks --out my-app-signed.apk my-app-aligned.apk
```

For personal use only, you can install the **debug APK directly** without signing.

---

## How the sync works

**Full sync (on Start Sync):**
1. Lists all files in the Dropbox folder via `files/list_folder`
2. Enumerates all files in the local folder via File System Access API
3. For each file in the union:
   - **Dropbox only** → download to local
   - **Local only** → upload to Dropbox
   - **Both modified** → compare timestamps → newer wins
   - **Same timestamp** → compare SHA-256 hash prefix → re-upload if different

**Watch mode (after initial sync):**
- Uses `SyncAccessHandle.createFileWatcher()` if available (Chrome 102+)
- Falls back to re-scanning every **30 seconds** (watching for changes)
- On each change: single-file delta sync

**Conflict resolution:** Last-write-wins. No special conflict handling — the newest file always overwrites the oldest.

---

## Data & Privacy

- **All processing happens in your browser.** No server receives your files.
- Access token stored in `localStorage` — cleared on logout
- Sync manifest stored in IndexedDB locally
- Activity log stored in IndexedDB locally
- No analytics, no tracking, no telemetry

---

## File Structure

```
dropsync-pwa/
├── index.html      # Full PWA app (HTML + CSS + JS)
├── manifest.json   # PWA manifest (icons, theme, display mode)
└── sw.js           # Service worker (offline caching)
```

---

## Known Limitations

- **Chrome on Android only** — File System Access API is not available in Firefox/Safari
- **Requires HTTPS** — the browser APIs won't work on `http://` origins
- **Large vault performance** — enumerating thousands of files may be slow on first sync. Delta sync after the first run is fast
- **File System Access API permission** — the folder permission persists until you revoke it in Android settings

---

## Troubleshooting

**"Connect" button does nothing:**
- The Dropbox App Key is still the placeholder. Tap "Connect" and enter your real App Key.

**"Could not access folder" error:**
- Make sure you're using Chrome on Android and granted the folder permission when prompted

**Sync doesn't start:**
- Both folders must be selected before "Start Sync" enables
- Network connection required for the initial sync

**Files not syncing after app restart:**
- The local folder handle persists in Chrome's storage but may need re-verification after a browser update. Re-select the folder if needed.
