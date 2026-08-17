# XOV — App Store Publishing Handoff

**Repo:** `liav-chanoch/portfolio`  
**Branch:** `claude/tic-tac-toe-variant-game-lh72jn`  
**Date:** 2026-08-17

---

## Status

| Item | Status |
|---|---|
| PWA / Service Worker | ✅ Done |
| Capacitor 6 scaffolded | ✅ Done |
| Icons generated (Android + iOS + PWA) | ✅ Done |
| Firebase Auth — native WebView fix | ✅ Done |
| Firebase Auth — anonymous guest fix | ✅ Done |
| Realtime Database rules secured | ✅ Done |
| Cloud Firestore rules secured | ✅ Done |
| Anonymous auth provider enabled | ✅ Done |
| Android Studio build (Play Store) | ⚠️ Pending |
| Xcode / iOS build (App Store) | ⚠️ Pending |
| Firebase authorized domain for Capacitor | ⚠️ Pending |
| Confirm bundle ID before store registration | ⚠️ Pending |

---

## What Was Done

### 1. PWA — Service Worker
- Added `sw.js`: cache-first strategy for `game.html`, icons, manifest, sw.js itself
- Registered inline in `game.html <head>`
- Old cache versions cleaned on activate

### 2. Capacitor 6 — Native App Shell
- Added `package.json` with `@capacitor/core`, `cli`, `android`, `ios` v6
- `capacitor.config.json`: **appId `com.liav.xov`**, `webDir: "www"`, `androidScheme: https`
- Build script copies `game.html → www/index.html` + icons/manifest/sw before every sync
- `android/` and `ios/` are **gitignored** — regenerate locally with the workflow below

### 3. Icons & Splash Screens
- Source icon: `assets/icon.png` (1024×1024)
- Run `npm run assets` locally to generate:
  - Android: 74 assets (adaptive icons + splash, all densities)
  - iOS: 7 assets (AppIcon + splash set)
  - PWA: 7 webp icons into `icons/`

### 4. Firebase Auth — Native WebView Fix
- `signInWithPopup` is blocked in WKWebView / Android WebView
- `signInGoogle` now uses `signInWithRedirect` inside Capacitor, `signInWithPopup` everywhere else
- `getRedirectResult(auth)` called on startup to complete the redirect flow
- Imported: `signInWithRedirect`, `getRedirectResult`, `signInAnonymously`

### 5. Firebase Auth — Guest Fix
- Guests previously had no Firebase UID (completely unauthenticated)
- `playAsGuest` now calls `signInAnonymously()` — gives guests a real UID
- `onAuthStateChanged` treats anonymous users as guests (no profile, no stats loaded)
- Required so DB write rules (`auth != null`) don't break guest play

### 6. Firebase Security Rules
Both databases were in 30-day test mode and about to expire.

**Realtime Database** (`games/{roomCode}` — live game state):
```json
{
  "rules": {
    "games": {
      "$gameId": {
        ".read": true,
        ".write": "auth != null"
      }
    }
  }
}
```

**Cloud Firestore** (user profiles + h2h records):
```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /users/{userId} {
      allow read: if request.auth != null;
      allow write: if request.auth != null && request.auth.uid == userId;

      match /h2h/{opponentId} {
        allow read, write: if request.auth != null && request.auth.uid == userId;
      }
    }
  }
}
```

---

## Files Changed / Added

| File | Change |
|---|---|
| `sw.js` | New — cache-first service worker |
| `game.html` | SW registration, signInWithRedirect, getRedirectResult, signInAnonymously for guests, onAuthStateChanged anonymous branch |
| `capacitor.config.json` | New — appId, webDir, androidScheme |
| `package.json` | New — Capacitor deps + build/sync/assets/open scripts |
| `assets/icon.png` | New — 1024×1024 source icon |
| `.gitignore` | New — node_modules, android, ios, www |
| `manifest.json` | Already correct, no changes needed |

---

## npm Scripts Reference

| Script | What it does |
|---|---|
| `npm run build` | Copies game.html → www/index.html + icons + manifest + sw.js |
| `npm run sync` | Runs build then `npx cap sync` (copies www into android + ios) |
| `npm run assets` | Runs build then generates all icon/splash sizes from assets/icon.png |
| `npm run open:android` | Opens android/ in Android Studio |
| `npm run open:ios` | Opens ios/ in Xcode |

---

## Workflow — After Cloning / Pulling

`android/` and `ios/` are gitignored. Run this on the Mac after every fresh clone or pull:

```bash
cd ~/portfolio
npm install
npm install -D @capacitor/assets   # first time only
npx cap add android                # first time only
npx cap add ios                    # first time only — needs Xcode installed
npm run assets                     # generate icons/splash
npm run sync                       # build www/ and copy to native projects
```

---

## Remaining Steps

### Android — Play Store
1. Download [Android Studio](https://developer.android.com/studio)
2. Run `npm run open:android`
3. Build → Generate Signed Bundle/APK
4. Create a keystore (keep it safe — you need it for every future update)

### iOS — App Store (Mac only)
1. Install Xcode from the Mac App Store (~8GB)
2. Run:
   ```bash
   sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
   sudo gem install cocoapods
   npm run sync
   npm run open:ios
   ```
3. In Xcode: Archive → Distribute App → App Store Connect

### Firebase — Authorized Domain
- Firebase Console → Authentication → Settings → Authorized domains
- Add the Capacitor custom scheme URL
- The exact value appears when the first native sign-in attempt fails with `auth/unauthorized-domain`

### Bundle ID
> ⚠️ The bundle ID `com.liav.xov` in `capacitor.config.json` is **permanent** once registered on the stores. Confirm it before submitting anything to App Store Connect or Play Console.

### Splash Image (optional improvement)
Add `assets/splash.png` (2732×2732, logo centered on `#F4E7C9` background) then re-run `npm run assets`. Without it the generator uses the icon on a solid background — functional but not ideal.
