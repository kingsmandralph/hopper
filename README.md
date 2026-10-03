# Hopper — Android app via Bubblewrap

This folder turns the Hopper game into an installable Android app using
[Bubblewrap](https://github.com/GoogleChromeLabs/bubblewrap), Google's tool for
packaging a web app as a **Trusted Web Activity (TWA)**.

```
hopper-twa/
├── www/                     ← the PWA you host (this IS the app's content)
│   ├── index.html           ← the game (standalone, full offline)
│   ├── manifest.json        ← web app manifest
│   ├── sw.js                ← service worker (offline caching)
│   ├── icons/               ← 192, 512, and maskable 512 PNGs
│   └── .well-known/
│       └── assetlinks.json  ← ownership proof (you fill in one value)
├── twa-manifest.json        ← reference Bubblewrap config
└── README.md                ← you're reading it
```

## How a TWA works (read this first)

A TWA app is a thin Android wrapper that opens your **hosted** web app full-screen,
with no browser address bar. That means **two things are true**:

1. The game must live at a public **HTTPS URL** you control (free options below).
2. You prove you own that URL by hosting `.well-known/assetlinks.json` with your
   signing key's fingerprint. Without it the app still runs, but shows a URL bar.

The offline service worker means that once the app has loaded, it keeps working
with no connection — so it behaves like a normal installed game after first launch.

> Want a 100% offline APK with **no hosting at all**? That's a plain WebView app
> (Android Studio), not Bubblewrap — see the note at the bottom.

---

## Step 1 — Host the `www/` folder (free)

Pick one. All give you HTTPS. Examples below assume the game ends up at a path
like `https://<you>.github.io/hopper/`.

### Option A: GitHub Pages
```bash
# from inside this folder
cd www
git init
git add .
git commit -m "Hopper PWA"
# create an empty repo named "hopper" on github.com first, then:
git branch -M main
git remote add origin https://github.com/<YOUR_USERNAME>/hopper.git
git push -u origin main
```
Then on GitHub: **Settings → Pages → Build from branch → main → /(root) → Save.**
After a minute your game is live at `https://<YOUR_USERNAME>.github.io/hopper/`.

### Option B: Netlify / Cloudflare Pages
Drag the `www` folder onto the Netlify dashboard (or connect the repo in Cloudflare
Pages). You'll get a URL like `https://hopper-xyz.netlify.app/`.

**Confirm it works**: open the URL on your phone's Chrome. You should be able to
play, and Chrome may even offer "Add to Home screen" already.

---

## Step 2 — Install Bubblewrap

Prereqs: **Node.js 18+**. Bubblewrap downloads and manages its own JDK and Android
SDK on first run, so you don't need Android Studio.

```bash
npm install -g @bubblewrap/cli
```

---

## Step 3 — Initialise the project from your live manifest

Point Bubblewrap at the `manifest.json` you just hosted:

```bash
# in a NEW empty folder (not www/)
bubblewrap init --manifest https://<YOUR_USERNAME>.github.io/hopper/manifest.json
```

It asks a series of questions — sensible answers for Hopper:

| Prompt | Answer |
|---|---|
| Domain | `<YOUR_USERNAME>.github.io` |
| URL path | `/hopper/` |
| Application name | `Hopper` |
| Short name | `Hopper` |
| Application ID (package) | `com.ghalahad.hopper` (any reverse-domain string that's yours) |
| Display mode | `fullscreen` |
| Orientation | `default` (or `landscape` if you prefer) |
| Status/nav bar color | `#1a1033` |
| Icon / maskable icon | accept the ones from the manifest |
| Signing key | let it **generate a new keystore** (remember the passwords!) |

> The included `twa-manifest.json` in this folder is a filled-in reference you can
> copy your values from. If you'd rather not answer prompts, drop it into your
> Bubblewrap project folder (edit the `YOUR_USERNAME` placeholders first) and skip
> straight to `bubblewrap build`.

**Keep the generated `android.keystore` file and its passwords safe.** You need the
exact same key to ship any future update of the app.

---

## Step 4 — Build the APK

```bash
bubblewrap build
```

Output in the project folder:
- `app-release-signed.apk` ← install this on a phone
- `app-release-bundle.aab` ← only needed if you ever upload to Google Play

Install it on a USB-connected phone (with USB debugging on):
```bash
adb install app-release-signed.apk
```
Or just copy the `.apk` to your phone, tap it, and allow "install from unknown
sources." Hopper appears in your app drawer with its own icon.

---

## Step 5 — Remove the address bar (Digital Asset Links)

To make it a true full-screen app (no URL bar), link the app to the website:

1. Get your signing key's SHA-256 fingerprint:
   ```bash
   bubblewrap fingerprint list
   ```
   Copy the long `AA:BB:CC:...` SHA-256 value.
2. Open `www/.well-known/assetlinks.json` and replace
   `REPLACE_WITH_YOUR_SHA256_FINGERPRINT` with that value. Make sure
   `package_name` matches the Application ID you chose (`com.ghalahad.hopper`).
3. Re-deploy `www/` so the file is live at
   `https://<YOUR_USERNAME>.github.io/hopper/.well-known/assetlinks.json`.
4. Reinstall the app. The address bar is gone.

> Bubblewrap can also generate this file for you: `bubblewrap fingerprint
> generateAssetLinks` prints the exact JSON to host.

---

## Updating the game later

1. Edit files in `www/`, bump `CACHE` in `sw.js` (e.g. `hopper-v2`) so the service
   worker refreshes, and re-deploy.
2. For the Android app, bump `appVersionCode` (and `appVersionName`) in
   `twa-manifest.json`, then `bubblewrap update && bubblewrap build` with the
   **same keystore**.

Because the APK just loads your hosted site, small content changes often need only
a re-deploy of `www/` — no rebuild at all.

---

## Alternative: fully offline APK, no hosting

Bubblewrap always points at a hosted URL. If you want the HTML bundled *inside* the
APK with zero hosting:

1. Android Studio → New Project → **Empty Views Activity**.
2. Put `www/`'s files in `app/src/main/assets/`.
3. In the activity, load a `WebView` with
   `webView.loadUrl("file:///android_asset/index.html")` and enable JavaScript
   (`webView.settings.javaScriptEnabled = true`) and DOM storage
   (`webView.settings.domStorageEnabled = true`, so the high-score saving works).
4. Build → Build APK.

More setup than Bubblewrap, but the result needs no internet and no website.

---

## Google Play (optional)

If you ever want it on the Play Store: one-time **$25** developer registration,
then upload the `.aab` from Step 4. Play requires the TWA + asset links to be set
up (Steps 1–5) so the app verifies against your domain.
