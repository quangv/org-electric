# GitHub Login — Options for org-electric

Status: **undecided**. Captured so we can pick later.

Web is currently considered the "main" target. Desktop is Electron Forge + Vite wrapping the same renderer.

## What `../alpha` does today

### `alpha/org-sync-web` (Next.js) — NextAuth v5 OAuth
- Provider: GitHub, scope `repo`.
- Key files:
  - `lib/auth.ts` — NextAuth config.
  - `app/api/auth/[...nextauth]/route.ts` — route handler.
  - `app/page.tsx` — `signIn("github", { redirectTo: "/editor" })`.
  - `app/editor/page.tsx` — `await auth()`; passes `session.accessToken` to the client.
  - `types/next-auth.d.ts` — adds `accessToken` to Session.
  - `lib/github.ts` — Octokit calls using the session token.
- Env (`.env.local.example`):
  ```
  AUTH_GITHUB_ID=...
  AUTH_GITHUB_SECRET=...
  AUTH_SECRET=...            # openssl rand -base64 33
  DEV_GITHUB_TOKEN=ghp_...   # optional dev bypass, skips OAuth
  ```
- Deps: `next-auth@^5.0.0-beta.32`, `@octokit/rest`, `next@16`.

### `alpha/org-sync` (Tauri desktop) — manual PAT
- No OAuth. User pastes a PAT in Settings.
- Stored plaintext in `{appConfigDir}/config.json`.
- Rust commands: `set_github_token`, `authed_url` (injects token into git remote URL `https://TOKEN@github.com/...`).
- Key files: `src-tauri/src/lib.rs:14-100`, `src-tauri/src/git.rs`, `src/App.tsx:412-424`.

## Options for this repo

### A. Mirror alpha exactly
Web = NextAuth OAuth, desktop = paste-a-PAT. Two code paths, two UXes.
- Pro: proven, directly ported.
- Con: this repo is Vite, not Next.js — the web side would require either switching to Next or hand-rolling the OAuth exchange. Two auth models to maintain.

### B. Switch web target to Next.js
Port `org-sync-web` nearly verbatim. Electron loads the deployed URL (or a bundled Next server) later.
- Pro: lowest-friction port of a working OAuth setup, keeps `DEV_GITHUB_TOKEN` DX, "Sign in with GitHub" button for real.
- Con: swaps out the current Vite web scaffold. Electron integration becomes "point at Next" rather than "share a renderer."

### C. Keep Vite, add a tiny OAuth backend
~50 lines of Express/Hono to do the `code → access_token` exchange (needs the client secret, can't be done in-browser). Frontend redirects to `github.com/login/oauth/authorize`, backend sets httpOnly cookie.
- Pro: keep Vite, get real OAuth, works for both web and (via custom protocol handler) desktop.
- Con: hand-rolled plumbing, hosting a tiny backend somewhere.

### D. Unified PAT + `.env` dev bypass on both targets
Token resolution: `VITE_GITHUB_TOKEN` env → saved token (localStorage on web, Electron `safeStorage` on desktop) → login prompt with PAT input.
- Pro: zero OAuth app / backend, one shared `src/github/auth.js`, same UX both sides, matches the `DEV_GITHUB_TOKEN` ergonomics the user liked.
- Con: "paste a token" UX, not "Sign in with GitHub." Weak if web ships to real end-users.

### E. GitHub Device Flow on both targets
User sees a code, visits `github.com/login/device`. No backend, no deep links, no PAT paste. Same code on web and desktop.
- Pro: no secret, works everywhere, nicer than PAT.
- Con: needs polling; UX is a short detour through github.com; still not the single-click "Sign in with GitHub" button.

## Decision matrix (rough)

| Option | Web UX   | Desktop UX | Backend?    | Code to write | Matches alpha? |
|--------|----------|------------|-------------|---------------|----------------|
| A      | OAuth    | PAT        | Yes (Next)  | Medium        | Exact          |
| B      | OAuth    | OAuth (via webview) | Yes (Next) | Low           | Web yes        |
| C      | OAuth    | OAuth      | Yes (tiny)  | Medium-high   | Spirit         |
| D      | PAT      | PAT        | No          | Low           | Desktop only   |
| E      | Device   | Device     | No          | Medium        | Neither        |

## Recommendation when the user picks

- If web is truly the main shipping product: **B** (switch to Next.js, port org-sync-web).
- If web is the dev playground and we want to move today: **D** (unified PAT).
- If we care about "Sign in with GitHub" but refuse to adopt Next: **C** (tiny OAuth backend) or **E** (device flow, no backend).

## Token storage decision (already made)

Desktop → **Electron `safeStorage`** (Keychain on macOS), not plain JSON. Encrypted file in `app.getPath('userData')`.
