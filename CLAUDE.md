# CLAUDE.md — Reno Board

## What this project is

**Reno Board** is a personal home-renovation tracker PWA. It tracks renovation
projects per room (status, cost/time estimates, scope line items), shows
before/after photo comparisons with a drag-to-reveal slider (the "after" will
eventually be an AI-generated render the owner produces separately and uploads),
and rolls everything up into whole-house progress and budget views.

## ⚠️ CURRENT PHASE: DESIGN MOCKUP ONLY — NO BACKEND

**Read this before doing anything else.** This project is intentionally in a
front-end-only mockup phase. The goal right now is to nail the look, layout, and
interaction feel on a real phone *before* any backend work begins.

That means:

- **Do NOT add** a server, database, API layer, framework migration, build
  step, bundler, or npm project. Not even "just a small one."
- **Do NOT add** real persistence. All data is hardcoded placeholder data in a
  `PROJECTS` array and lives in memory. State resets on refresh — that is
  expected and fine for this phase.
- **Do NOT wire up** real photo upload, camera access, or AI image generation.
  The "Before / After (AI)" tiles in the add sheet are visual placeholders.
- **Do NOT add** real authentication. "You" and "Sam" are mock household
  members used purely to demonstrate attribution UI.
- All work in this phase is **visual and interaction design**: layout, flows,
  components, transitions, placeholder data shapes.

If a task seems to require a backend, the correct move is to *fake it
convincingly in the UI* (in-memory state + plausible placeholder data), and
note what the real implementation would need later.

## Tech constraints

- **One app file:** `index.html`. Plain HTML/CSS/vanilla JS — no build step,
  no bundler, no npm. The only sibling files are PWA plumbing
  (`manifest.webmanifest`, `sw.js`, `icon-{180,192,512}.png`).
  All app code stays in `index.html`.
- **Mobile-first, phone-only usage.** Target viewport ~390×844. There is a
  desktop "phone frame" preview at ≥480px, but every decision is made for
  touch: thumb-reachable controls, 44px+ tap targets, bottom nav, sheets
  instead of modals.
- **Hosted on GitHub Pages** (repo `wildurbear/HR`, `main` branch, root) at
  <https://wildurbear.github.io/HR/> — that's how it gets onto the phone.
  Pushing to `main` deploys.
- **PWA via Add to Home Screen.** Real `manifest.webmanifest` + PNG icons +
  a minimal network-first service worker (`sw.js`, same-origin GETs only,
  cache fallback when offline). SW registration is skipped on `file://`.
  If a stale version ever sticks on the phone, bump the `CACHE` name in
  `sw.js`.
- Fonts come from Google Fonts (Space Grotesk / Inter / JetBrains Mono). This
  is the only external dependency.

## Architecture of the file

One HTML file with three sections:

1. **CSS** — design tokens in `:root`, then components in source order:
   shell, top bar, scroll area, home/hero, before-after slider (`.ba`),
   project cards, budget, bottom nav + FAB, detail overlay, add sheet.
2. **Markup** — app shell: fixed `#app` viewport box → top bar → `.scroll`
   content area (3 `.screen` sections: home / projects / budget) → FAB +
   bottom nav → detail `.overlay` → add `.sheet` + `.scrim`.
3. **JS** — in order: `PEOPLE` (mock users), `ROOMS` + `buildRoom()` (procedural
   SVG room art, "before"/"after" variants per room), `PROJECTS` (placeholder
   data), render functions (`renderHome` / `renderProjects` / `renderBudget`),
   `wireSlider()` (before/after drag), `openDetail()` (project detail overlay),
   nav, add sheet, service-worker registration.

### Data shapes (these will become the backend schema later — keep them clean)

```js
PROJECTS = [{
  id: 1,
  room: 'kitchen',          // key into ROOMS
  title: 'Cabinet reface & new island',
  status: 'todo' | 'doing' | 'done',
  statusBy: 'you' | 'sam' | null,   // attribution: who set current status
  statusAt: 'Jun 2 · 9:12 AM',      // display string (mock); real app: ISO timestamp
  cost: 8600,               // estimated cost, USD
  hours: 64,                // estimated labor hours
  notes: '...',
  scope: [                  // line items; drives progress %
    { t: 'Strip & prime', c: 900, done: true, by: 'you', at: 'Jun 3 · 2:14 PM' },
    { t: 'Counter',       c: 2100, done: false }   // by/at absent until checked
  ]
}]
```

`projectDoneRatio()`: done = 1, todo = 0, doing = checked/total scope items.
The home hero % is cost-weighted across all projects.

### Hard-won invariants — do not regress these

- **`#app` is a fixed viewport box pinned by `position:fixed; inset:0`**
  on phones, **plus a JS `visualViewport` height sync** (bottom of the
  script). The sync works around an iOS 26 WebKit bug (rdar 158055568)
  where standalone PWAs compute the layout viewport too short — without
  it there's a dead strip under the nav and the UI drifts after
  keyboard/scroll interactions. CSS units (`100dvh`/`100vh`/`inset:0`)
  all resolve against the same broken layout viewport, so they can't fix
  it — and on affected devices `visualViewport.height` reports the same
  short value, so in standalone mode the sync snaps to `screen.height`
  (unless the shortfall is keyboard-sized). Don't remove the JS sync
  even if the CSS looks sufficient. Only
  `.scroll` and `.ov-scroll` scroll. This is what keeps the bottom nav
  permanently pinned. Never let page-level scrolling come back.
- **`.after-layer` in the slider must stay `position:absolute; inset:0`** —
  its `clip-path` is what creates the before/after reveal. (Earlier bug: an
  unpositioned layer made before and after look identical.)
- **FAB sits bottom-right above the nav**, never centered over the tab bar.
- **Scroll resets to top** on tab switch, filter change, and opening a detail
  view. Exception: re-renders *within* an open detail (checking an item,
  changing status) preserve scroll position via `openDetail(id, true)`.
- **Attribution stamps**: checking a scope item or changing status stamps
  `CURRENT_USER` + `nowStamp()`. Unchecking clears the stamp. The Activity
  feed in detail view is derived from these stamps (newest first).
- Card thumbnails show the **before** art until status is `done`, then flip
  to **after** — intentional reward moment, keep it.
- Respect `prefers-reduced-motion`; keep visible `:focus-visible` outlines.

### Design system (use the tokens, don't invent colors)

- Concept: contractor's punch list × paint sample board. Every room has a
  paint-chip color (`ROOMS[key].chip`) used consistently on dots, bars, art.
- Palette: plaster `--plaster`/`--card` surfaces, ink `--ink` text, petrol
  `--petrol` primary (done/actions), marigold `--marigold` (in progress).
- Type: Space Grotesk (display) / Inter (body) / JetBrains Mono (all numbers,
  timestamps, eyebrows — anything "spec sheet").
- Room art is procedural SVG from `buildRoom(key, 'before'|'after')`. Befores
  are deliberately dingy (stains, clutter, bare bulbs); afters are fresh
  (paint, plants, warm light). If adding a room, build both states.

## How to verify changes

No build, no tests. Serve `index.html` locally (the service worker and
manifest need http, not `file://`):

```bash
# from the project directory
python3 -m http.server 8080
```

Then check in responsive dev-tools mode at iPhone dimensions **and** on a real
phone — either via LAN IP or by pushing to `main` and opening
<https://wildurbear.github.io/HR/>. Minimum manual pass: all 3 tabs, open a project, drag the
slider both directions, check/uncheck a scope item, change a status, add a
project via FAB, edit a project via the pencil (name/room/cost/hours),
toggle scope Edit and rename/re-cost/add/delete a line item, tap notes and
save an edit, delete a project and hit Undo, confirm the nav never moves
and every view opens at the top.
Syntax check after JS edits: extract the script block and `node --check`.
On-device viewport debugging: triple-tap the avatar to toggle an overlay of
every viewport measurement (inner/visualViewport/screen/safe-area insets).
Dev tool for the iOS standalone viewport bug — remove when the backend
phase starts.

## TODO — next design tasks (in priority order)

- [x] **Edit existing entries.** Done (Jun 2026). How it works:
  - [x] Project name/room/cost/hours: pencil button (`#ov-edit`) in the
        detail overlay's top bar reuses the add sheet in edit mode —
        `openSheet(id)` pre-fills and switches title/button to
        "Edit project" / "Save changes"; `editingId` distinguishes the
        two modes in the submit handler.
  - [x] Scope line items: an Edit/Done `mini-btn` toggle next to the scope
        section header (`scopeEdit` flag). In edit mode rows become
        name + cost inputs with an × delete button, plus a
        "+ Add line item" row. Chosen over swipe-to-delete for
        discoverability in a mockup.
  - [x] Notes: tap the notes card → textarea + "Save notes"
        (`notesEdit` flag).
  - [x] Delete project: "Delete project" button in the sheet (edit mode
        only), two-tap confirm (button arms to "Tap again to delete"),
        then in-memory removal + a 5s Undo toast (`showToast`) that
        restores at the original index.
  - Editing invariants: detail re-renders during editing use
    `openDetail(id, true)` to preserve scroll; `scopeEdit`/`notesEdit`
    reset when a *different* project is opened; cost edits flow into
    budget/progress immediately; input values are escaped via `escAttr`.
- [ ] Replace the placeholder SVG before/after art path with support for real
      images (`<img>` in the `.ba` slider) while keeping SVG as the fallback —
      still no upload backend, just prove the layout works with photo
      aspect ratios.
- [x] Empty states for first run (zero projects) on all three tabs.
      Done (Jun 2026): `renderHome`/`renderBudget` short-circuit with an
      `.empty` block when `PROJECTS` is empty (previously this crashed —
      `feat.room` on undefined / `Math.max()` of nothing); Projects tab
      already had one. Delete-all then re-add is part of the manual pass.
- [ ] Light haptic/visual feedback polish: pressed states, check animation.
- [ ] Optional: per-room grouping toggle on the Projects tab.

## Later phases (explicitly OUT of scope right now — do not start these)

- Real persistence (likely localStorage first, then a proper backend)
- Real photo capture/upload + storage
- AI "after" render generation integration
- Multi-user sync + real auth (decides whether attribution is real or local)
