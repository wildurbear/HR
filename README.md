# Reno Board — Home Renovation PWA

A personal home-renovation tracker PWA: projects per room (status, cost/time,
scope line items), before/after photo comparisons with a drag-to-reveal slider,
and whole-house progress + budget roll-ups.

**Live:** <https://wildurbear.github.io/HR/> (also a custom domain configured in
Pages settings). Open on a phone and use *Add to Home Screen* to install.

## Current phase: design mockup only — no backend

Front-end only. All data is hardcoded placeholder data in a `PROJECTS` array
and lives in memory — **state resets on refresh, by design.** No server, DB,
build step, auth, real photo upload, or AI generation yet. The goal of this
phase is to nail the look and interaction feel on a real phone first. See
[CLAUDE.md](CLAUDE.md) for the full brief, data shapes, and hard-won invariants
(read it before changing anything).

## Files

| File | Purpose |
|------|---------|
| `index.html` | The entire app — HTML + CSS + vanilla JS, no build step. |
| `manifest.webmanifest` | PWA manifest. |
| `sw.js` | Minimal network-first service worker. Bump `CACHE` to force-update installed PWAs. |
| `icon-{180,192,512}.png` | PWA / home-screen icons. |
| `.github/workflows/deploy-pages.yml` | Deploys repo root to GitHub Pages on push to `main`. |

## Develop / verify

No build, no tests. Serve over http (the manifest + service worker need http,
not `file://`):

```bash
python3 -m http.server 8080
```

Then test in mobile dev-tools **and** on a real phone (push to `main` and open
the live URL). After JS edits, syntax-check by extracting the `<script>` block
and running `node --check`. On a real phone, **triple-tap the avatar** to toggle
a viewport-measurement debug overlay (a temporary dev tool for the iOS-26
standalone viewport bug — see CLAUDE.md; remove when the backend phase starts).

## Next: backend phase (not started)

Add real persistence. Leading plan: SQLite (single file, no server to run — right
size for a 2-person household app) rather than Postgres; the data shapes in
CLAUDE.md port to either. A backend means moving off GitHub Pages (static-only)
to a small host. Then: real photo upload, AI "after" render generation, and
multi-user sync/auth.
