# liav-chanoch.com

Personal site and portfolio for Liav Chanoch — Product & Project Manager, Systems Builder.
Hand-written HTML and CSS, no framework, no build step. Deployed to Firebase Hosting on
every push to `main`.

**Live:** https://liav-chanoch.com

## Contents

| File | What it is |
|---|---|
| `index.html` | The portfolio site — experience, the systems I've shipped, skills, and the FDE track. Single file, inline CSS, ~1k lines. |
| `game.html` | **XOV** — a live multiplayer game that also ships as a native app. See below. |
| `sw.js`, `manifest.json` | Service worker and PWA manifest for XOV |
| `.github/workflows/` | Firebase Hosting deploy on push to `main` |

## XOV

Three-mark tic-tac-toe: X, O, and V rotate every turn, so the mark you place is decided by
the turn counter rather than by which player you are. Real-time multiplayer over Firebase
Realtime Database, with Firestore for persistence and anonymous auth for lobbies.

The same `game.html` is wrapped by Capacitor into iOS and Android builds (`com.liav.xov`),
so the web version and the native apps run identical code.

```bash
npm run build        # copy game.html + assets into www/
npm run sync         # build, then npx cap sync
npm run open:ios     # or open:android
npm run assets       # regenerate icon and splash sets
```

## Deploying

Pushing to `main` triggers `.github/workflows/firebase-deploy.yml`, which deploys to
Firebase Hosting using the `FIREBASE_TOKEN` repository secret.

## A note on the Firebase config

The Firebase web config in `game.html` is committed on purpose. Web API keys are public
identifiers, not credentials — they identify the project to Google's servers and are meant
to ship in client code. Access control is enforced by Firebase security rules.
