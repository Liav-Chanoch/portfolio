# XOV — Project Handoff

> Written 2026-10-14 for a fresh session with no memory of the earlier work.
> Everything here was checked against the repo on this date unless marked **(unverified)**.
> This file replaces the shorter `HANDOFF.md` from earlier in the session.

---

## 1. TL;DR

**XOV** is a three-mark tic-tac-toe game (X, O and V rotate every turn). It is one file, `game.html`, that runs as:

1. a **web app** at `liav-chanoch.com/game.html` (Firebase Hosting, auto-deployed from `main`),
2. an **installable PWA** (manifest + service worker),
3. a **native iOS/Android app** via Capacitor 6, bundle ID `com.liav.xov`.

This session took XOV from "web game" to "ready to be wrapped for the stores", and fixed two expiring Firebase databases. The **web and PWA parts are done and live. The native apps have never been built or run on a device.** Several problems below will probably show up the first time they are.

**Do these first:**

1. On the Mac: `git pull`, then `npm run sync` (the Mac clone predates the anonymous-auth change — see §6).
2. Read §9 "Known issues". The two most likely to block a native release are the **iOS in-app-browser check swallowing Google sign-in** and **Google redirect auth inside a WebView**.
3. Decide the bundle ID is final before anything is registered with Apple or Google (§10).

---

## 2. People, machines and working style

| Thing | Detail |
|---|---|
| Owner | Liav Chanoch (GitHub `liav-chanoch`, `liavch1@gmail.com`). Use their name or "they"; don't guess pronouns. |
| Collaborator | A "Claude Cowork" session/colleague asked the original setup questions. Liav relays between sessions. |
| Liav's Mac | MacBook Air, user `liavchanoch`, zsh, Node **v24.15.0**, repo cloned at `~/portfolio`. **No Xcode, no CocoaPods, Android Studio not confirmed installed.** |
| Cloud container | This Claude Code session: `/home/user/portfolio`. Ephemeral. Cannot build native apps; `sharp`/libvips and Xcode don't work here. |
| Language | Liav sometimes writes in Hebrew. Reply in the language they use. |
| Style that worked | Short, exact, copy-pasteable terminal steps; ask Liav to paste output back; one step at a time. Liav is comfortable following instructions but is not a native-app developer. |

---

## 3. Repository and branches

Remote: `github.com/Liav-Chanoch/portfolio`.

| Branch | Role |
|---|---|
| `main` | Production. **Every push deploys to Firebase Hosting** (§5). Also carries the portfolio-site redesign and `/dj/` page done in other sessions. |
| `claude/tic-tac-toe-variant-game-lh72jn` | The branch this session was told to develop on. Contains the XOV work below. |

**The two branches have unrelated histories** (`git merge-base` finds nothing; `main` has ~50 commits the feature branch lacks and vice versa). Content-wise, the XOV files (`game.html`, `sw.js`, `manifest.json`, `package.json`, `capacitor.config.json`, `assets/icon.png`, `icons/`) are **byte-identical** on both. The differences are the portfolio site (`index.html`, `v2/`, `v3/`, `dj/`, `cv/`, `assets/shots`, etc.) which exist only on `main`, and this `HANDOFF.md` which exists only on the feature branch.

Consequences for you:
- Don't try to merge the feature branch into `main`. Branch from `origin/main` for new work and port anything you need.
- During the session the XOV files were pushed to **both** branches by copying files onto `main` (`git checkout <branch> -- <files>`) and pushing. Each of those pushes triggered a live deploy. Don't push to `main` without asking Liav.
- `firebase.json` hosts `"public": "."` — the **entire repo root is served publicly**, with only `firebase.json`, dotfiles, `node_modules` and `.github` excluded. A root-level `HANDOFF.md`, `package.json` etc. would be downloadable at `liav-chanoch.com/…` if they reach `main`. This file contains no secrets, but consider adding docs to the `ignore` list or keeping them under an ignored folder.

### Repo map (feature-branch view)

```
game.html              the whole game: HTML + CSS + JS in one file (~2,400 lines)
index.html             older copy of the portfolio (main has the redesigned one)
sw.js                  service worker
manifest.json          PWA manifest (start_url /game.html, theme #F4E7C9)
capacitor.config.json  appId com.liav.xov, webDir www, androidScheme https
package.json           Capacitor 6 deps + build/sync/assets/open scripts
assets/icon.png        1024×1024 source for icon/splash generation
icons/                 favicons, apple-touch-icon, maskable, ios/ sizes, master-paper-1024.png
.gitignore             node_modules/ android/ ios/ www/ dist/ .DS_Store
HANDOFF.md             this file
```

On `main` only: `.github/workflows/firebase-deploy.yml`, `firebase.json`, `.firebaserc` (project `xov---game`), `CNAME` (`liav-chanoch.com`), `README.md`, `robots.txt`, `sitemap.xml`, `cv/`, `dj/`, `v2/`, `v3/`.

---

## 4. How XOV works (so you can change it safely)

### Game
- Board is a string of `.`/`0`/`1`/`2` characters (`emptyBoard`, `parseBoard`, `boardToStr`). Symbols are `X`,`O`,`V`; the mark placed is decided by the **turn counter**, not by player.
- Grid is 4×4 or 5×5; win length 3 or 4 (chosen with the drum pickers). Old records use `gameType: '4x4' | '5x5'` — `setGameType()` keeps them working.
- Modes: **online** (Firebase), **vs AI** (easy/medium/hard; hard uses alpha-beta `mmAI`), **local pass-and-play** (pass overlay between turns). AI/local never touch the database.
- Matches are first-to-N style with per-game scores (`matchP1Score`, `matchP2Score`, `matchGame`, `matchWinner`).

### Realtime Database — live games
Path `games/{4-letter room code}`:

```
board, turn, status ('waiting' | 'playing' | 'finished'), winner, winCells,
gridCols, winLen,
p1Name, p1Photo, p1Uid,  p2Name, p2Photo, p2Uid,
matchP1Score, matchP2Score, matchGame, matchWinner
```

Flow: `createGame` → `set()` the room as p1 (`status:'waiting'`) → other player `joinGame` reads the room, `update()`s `status:'playing'` and p2 fields → both `onValue` listen (`listen()` / `render()`) → `makeMove` writes via `update()`. Page refresh restores the room from `sessionStorage` (`xov-room`, `xov-player`).

### Firestore — accounts and stats
- `users/{uid}`: `displayName, photoURL, wins, losses, draws, createdAt`
- `users/{uid}/h2h/{opponentUid}`: `wins/losses/draws` counters + `opponentName`
- Created by `ensureUserProfile`, incremented by `updateScore` (only when `currentUser && !isGuest`, online games only for h2h).

### Auth (Firebase Auth, SDK 10.14.1 loaded from gstatic)
Enabled providers: **Email/Password, Google, Anonymous**.

- **Google:** `signInGoogle()` → `signInWithRedirect` when `window.Capacitor?.isNativePlatform()`, otherwise `signInWithPopup`. `getRedirectResult(auth)` runs once at startup. It first bails out if `isInAppBrowser()` is true (see §9, issue 1).
- **Email/password:** `handleEmailAuth` / `toggleAuthMode`, with `friendlyAuthError` messages and a nickname modal when `displayName` is empty.
- **Guest (changed this session):** "Play as guest" now calls `signInAnonymously()`. `onAuthStateChanged` treats `user.isAnonymous` as a guest: `isGuest = true`, no Firestore profile, no stats, but a real UID exists so database rules can require `auth != null`. If anonymous sign-in fails it falls back to the old unauthenticated guest — which can no longer write to the database.
- `signOutUser` returns to the login screen. `renderNav` (avatar, nickname, Sign out) is only drawn for non-anonymous users.

### Firebase web config
Committed in `game.html` on purpose (project `xov---game`, RTDB in `europe-west1`, Firestore in `europe-west10`). Web API keys are public identifiers; protection comes from the rules in §7 and, ideally, App Check (not enabled — Firebase's console banner suggests it).

---

## 5. Hosting and deploy

- `.github/workflows/firebase-deploy.yml` (on `main`): on every push, `firebase deploy --only hosting --project xov---game --token "$FIREBASE_TOKEN" || true`.
- **The trailing `|| true` hides failed deploys.** A green workflow does not prove the site updated. After any push to `main`, open the live site and check.
- `CNAME` contains `liav-chanoch.com` (a GitHub Pages leftover), while the README says the domain is served from Firebase Hosting. **(unverified)** which actually serves the domain — worth a 30-second check in the Firebase and GitHub settings.
- The live site **needs** the anonymous-auth change, because guests can no longer write without it. Confirm it is deployed: on `liav-chanoch.com/game.html` tap *Play as guest* → *Play online* → create a room. If creation fails, the deploy didn't land (or Anonymous auth got disabled).

---

## 6. Native packaging (Capacitor 6)

### How it fits together
- Capacitor needs a real `webDir`. `"."` is rejected, so the build copies the web files into `www/` (gitignored):
  `npm run build` = `mkdir -p www && cp game.html www/index.html && cp manifest.json www/ && cp sw.js www/ && cp -r icons www/`
- `game.html` becomes `www/index.html`, so inside the native app the game **is** the home page.
- `android/` and `ios/` are generated and gitignored. They are recreated with `npx cap add android|ios`.
- `androidScheme: "https"` serves the Android WebView from `https://localhost`.

### npm scripts

| Script | Does |
|---|---|
| `npm run build` | refresh `www/` |
| `npm run sync` | build, then `npx cap sync` (copy `www/` into `android/` and `ios/`) |
| `npm run assets` | build, then `@capacitor/assets generate` with `#F4E7C9` backgrounds |
| `npm run open:android` / `open:ios` | open the native project in Android Studio / Xcode |

`@capacitor/assets` is **not** in `package.json` — it pulls in `sharp`, which won't install in the cloud container. Liav installed it locally with `npm install -D @capacitor/assets`, which edited their local `package.json`/lockfile (not committed). Re-run that on any new machine.

### State on Liav's Mac (as last reported)
- Cloned, `npm install` done, `android/` and `ios/` added, **`npm run assets` succeeded** (74 Android files, 7 iOS files, 7 PWA `.webp` icons written into `icons/` — those are uncommitted changes in the Mac working tree).
- `npm run sync` updated Android fine. For iOS it failed at `pod install`: **CocoaPods not installed and only Command Line Tools present (no full Xcode).**
- Capacitor telemetry declined.
- **The Mac clone is older than the anonymous-auth commit.** Run `git pull` (resolve against whichever branch they cloned — they cloned the default branch, i.e. `main`), then `npm run sync`, or the native build will ship the old guest flow.
- `npm audit` reports 8 vulnerabilities, all in dev tooling pulled by `@capacitor/assets`. Not shipped in the app. Don't run `npm audit fix --force`.

### Splash/icon
Only `assets/icon.png` exists, so the generator derived splash screens from the icon on `#F4E7C9`. A dedicated `assets/splash.png` (2732×2732, logo centered) would look better; it's optional.

---

## 7. Firebase project state (console, not in git)

Project **XOV - GAME** (`xov---game`), Spark (free) plan.

**Both databases were in 30-day test mode and about to lock everyone out** (Firebase emails: RTDB warning, then a Firestore warning with 1 day left; the Firestore rule expired on 2026-07-23). Liav replaced both and confirmed publishing (the Firestore Rules tab screenshot showed the new rules published "Today 11:43 AM"; the RTDB publish was confirmed in chat).

**Realtime Database rules:**
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

**Firestore rules:**
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

**Authentication → Sign-in method:** Email/Password ✔, Google ✔, **Anonymous ✔ (enabled this session)**.

**These rules are only in the console.** There is no `database.rules.json`, `firestore.rules`, or rules section in `firebase.json`. They are not versioned and the Hosting deploy does not touch them. Treat adding them to the repo as a to-do.

### What these rules do and don't protect (be honest with Liav about this)
They are a **stopgap that keeps the game working and stops unauthenticated bots, not real security.**
- RTDB: any signed-in user — and anyone can mint an anonymous account just by loading the page — can overwrite **any** room, including mid-game. Rooms are 4 letters (~457k codes) and `.read` is public per room, so codes are guessable. There is no validation of the data shape.
- Firestore: any signed-in (including anonymous) user can read every profile (`displayName`, `photoURL`, win/loss counts). A user can write arbitrary numbers into their own `wins/losses/draws`, so stats are self-reported and trivially forged.
- Untested tighter RTDB direction **(proposal — not applied, test with two accounts before publishing)**: require `newData.child('p1Uid').val() === auth.uid` on create; allow later writes only if `auth.uid` equals the stored `p1Uid` or `p2Uid`, plus the single "join" transition (`status` `waiting` → `playing` where the new `p2Uid === auth.uid`); `.read` only for `auth != null`. Remember the AI fallback path writes `p1Uid:''`, and rematch/`playAgain` writes must still pass.

---

## 8. What was done this session (chronological, with commits)

| Commit (feature branch) | What |
|---|---|
| `ae85fcf` | Added `sw.js` and its registration; `package.json`, `capacitor.config.json` (initially `webDir: "."` — wrong), `assets/icon.png`, `.gitignore`; Google sign-in switched to redirect on native. |
| `185c2b2` | Fixed `webDir` → `www`; added the `build` script and made `sync` run it; `www/` gitignored. |
| `36a9ba4` | `assets` script now runs `build` first (generation failed on the Mac because `www/manifest.json` didn't exist yet). |
| `2ccaa47` | Guests use Firebase **anonymous auth**; `onAuthStateChanged` has an anonymous branch. |
| `048ec45` | First, shorter handoff doc. |

Non-code work done by Liav on instruction:
- Cloned the repo to `~/portfolio`, ran `npm install`, `npm install -D @capacitor/assets`, `cap add android/ios`, `npm run assets`, `npm run sync`.
- Enabled Anonymous sign-in; published new RTDB and Firestore rules (§7).

Also produced earlier in the session: an HTML handoff artifact (`claude.ai/code/artifact/a7c19271-1fce-4048-a17f-73fa64cd80a6`). **It is now outdated** — it shows the optional splash item with a green check, and carries the incorrect authorized-domain advice listed in §11. Prefer this file.

---

## 9. Known issues and risks

Severity is my judgement. "Verified" means seen in the code; "likely" means inferred from how WebViews/Firebase behave and **not tested on a device**.

### High — likely to bite the first native build

1. **iOS app will probably refuse Google sign-in.** `isInAppBrowser()` (`game.html` ~L1076) returns true for any iPhone UA that contains `AppleWebKit` but not `Safari/`. A Capacitor WKWebView has exactly that UA, so `signInGoogle()` exits with "Open in Safari or Chrome to sign in with Google" before reaching the Capacitor branch. *(Verified the code; likely the behaviour.)* Fix: skip the check when `window.Capacitor?.isNativePlatform()`.
2. **Google redirect auth in a WebView is unreliable even with that fixed.** Firebase's web `signInWithRedirect` from `capacitor://localhost` (iOS) or `https://localhost` (Android) commonly fails or loses the result (unauthorized origin, storage partitioning). Firebase's own guidance for Capacitor is native sign-in, e.g. the `@capacitor-firebase/authentication` plugin with the Firebase SDK in "skip native auth" mode. *(Likely; test on device before promising Google sign-in in the apps.)* Email/password and anonymous guest should work in a WebView unchanged.
3. **Apple review rules (verify against current guidelines):** an app offering Google login generally must also offer an equivalent privacy-preserving option (Sign in with Apple, or an alternative that meets Apple's criteria); apps that create accounts must offer **in-app account deletion** — XOV has none; both stores require a **privacy policy URL**; Google Play needs the Data safety form and (for new personal accounts) closed-testing requirements. None of this has been started.

### Medium

4. **Service worker is cache-first with a fixed cache name (`xov-v1`) and never revalidates.** After any deploy, installed PWAs and returning browsers will keep serving the old `game.html` until the cache name is bumped or the cache is cleared. For a game that ships fixes, switch `game.html` (and other app shell files) to network-first/stale-while-revalidate, or version `CACHE` on every release. *(Verified in code.)*
5. **SW precache lists `/game.html`, but in the native bundle that file is `index.html`.** `cache.addAll` rejects on a 404, so the SW install fails inside the app. Harmless today (native doesn't need a SW) but noisy; consider not registering the SW when `window.Capacitor?.isNativePlatform()`.
6. **"Portfolio" back-link** in the nav (`<a href="index.html" class="nav-back">`) points at `index.html`, which inside the native app is the game itself. Hide it on native. *(Verified.)*
7. **Anonymous accounts pile up.** Each guest creates a permanent anonymous Auth user. Consider Firebase's auto-cleanup of old anonymous accounts, and note that an anonymous user persists across reloads, so a guest lands straight on the home screen with no way back to the login screen except clearing site data.
8. **Weak database rules** (§7).
9. **Deploys can fail silently** (§5) and **`main`/feature-branch histories are unrelated** (§3).

### Low

10. `ensureUserProfile` runs only for non-anonymous users, so a guest who later signs in with Google/email starts with a fresh profile (no account linking via `linkWithCredential`).
11. h2h records can be written against an opponent's anonymous UID when a signed-in player meets a guest; those rows are never readable again by a meaningful identity.
12. `index.html` on the feature branch is a stale copy of the portfolio; the real one is on `main`.

---

## 10. Remaining work (suggested order)

1. **Mac:** `git pull` → `npm install` → `npm run sync`.
2. **Decide the bundle ID.** `com.liav.xov` was chosen as a placeholder when an interactive question failed; Liav never explicitly confirmed it. It is permanent once registered with App Store Connect / Play Console. Change it in `capacitor.config.json` (and re-add platforms, or edit the native projects) before registering.
3. **Android first** (no Mac-only tooling): install Android Studio → `npm run open:android` → run on an emulator/device → test Guest, email sign-in, create/join room. Then Build → *Generate Signed Bundle / APK*; create a keystore and **back it up** — losing it blocks all future updates.
4. Fix issues 1, 2, 4–6 in §9 **in `game.html` / `sw.js`** (on a branch from `origin/main`), then `npm run sync` again.
5. **iOS:** install full Xcode from the App Store, then
   ```bash
   sudo xcode-select -s /Applications/Xcode.app/Contents/Developer
   sudo gem install cocoapods
   npm run sync && npm run open:ios
   ```
   Set the signing team and bundle ID in Xcode, run on a simulator/device, then Product → Archive → Distribute. Requires an Apple Developer Program membership (paid).
6. Store listings: screenshots, description, privacy policy URL, age rating, support contact; Play Data safety form; account-deletion flow if keeping accounts.
7. Hardening: put rules in the repo and deploy them with `firebase deploy --only database,firestore`; tighten RTDB rules (§7); consider App Check; make the `|| true` in the workflow honest.
8. Optional: `assets/splash.png`, dark-mode splash, versioned SW cache.

---

## 11. Corrections to things said earlier in this session

- **Authorized domains:** Liav was told to add `com.liav.xov` / "the Capacitor custom scheme" under *Authentication → Settings → Authorized domains*. That was wrong. That list takes **hostnames** (e.g. `localhost`, `liav-chanoch.com`); an app scheme like `capacitor://localhost` can't be added. Don't spend time there; see §9 issues 1–2. `localhost` is normally present by default.
- The first `HANDOFF.md` and the HTML artifact were dated 2026-08-17; the work actually ran from about **2026-06-28 to 2026-07-22** (Mac login timestamp, Firebase emails).
- The first handoff described `capacitor.config.json` as `webDir: "."`. The real value is `"www"` (§6).
- The HTML artifact marks the optional splash image as done. It is not.

---

## 12. Working agreements for the next session

- Develop on a **new branch from `origin/main`** (or the branch Liav assigns). Don't push to `main` without asking — it deploys to the live domain.
- Don't open a pull request unless asked.
- Commit trailer used this session:
  `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>` and `Claude-Session: <session URL>`.
- No secrets in git. The Firebase web config is intentionally public; the `FIREBASE_TOKEN` lives in GitHub Actions secrets.
- Anything Liav must run is on their Mac, not here — give exact commands and ask for the output.
- Before claiming something works on iOS/Android, say whether it was tested on a device. So far, **nothing native has been run.**

---

## 13. Quick command reference

```bash
# fresh machine
git clone https://github.com/liav-chanoch/portfolio.git ~/portfolio && cd ~/portfolio
npm install
npm install -D @capacitor/assets      # local only (needs sharp)
npx cap add android                   # once
npx cap add ios                       # once, needs Xcode + CocoaPods
npm run assets                        # icons + splash from assets/icon.png
npm run sync                          # build www/ and copy into native projects

# day to day
npm run sync && npm run open:android
npm run sync && npm run open:ios

# live checks
open https://liav-chanoch.com/game.html      # Play as guest → create a room
```

Firebase console shortcuts: Authentication → Sign-in method (providers) · Realtime Database → Rules · Firestore Database → Rules · Project Overview for usage.
