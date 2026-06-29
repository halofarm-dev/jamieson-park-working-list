# Handoff: Host the Jamieson Park "Working List" with light shared sync

## Overview
`Working List` is a **finished, working** single-page web app for planning a thoroughbred stud's
matings: stallions as lanes, mares as draggable cards, status circles (review/pending/good),
per-mare comments, an Undo stack, a live "Changes" log, and a "What's new since you last opened"
banner. It currently runs entirely in the browser and **saves to that browser's `localStorage`**.

**The job is NOT to rebuild the UI.** The UI is done and approved. The job is to:
1. **Host** the app at a URL the owner and manager can open on any device.
2. **Swap the storage layer** from per-browser `localStorage` to **one shared cloud store**, so
   everyone sees the same board ("light" sync: load on open, save on change, refresh to see others').
3. Add **two roles** — *editor* (read + write) and *viewer* (read-only).
4. Keep the existing **"What's new"** feature working.

This is a small, well-contained backend-integration task — roughly "replace ~5 storage calls with
network calls, add a read-only mode, and deploy a static file."

## About the design files
- `Working List.dc.html` — the **source** of the app. It is an Om "Design Component": an HTML file
  with an inline template and a `class Component extends DCLogic` logic class. All app logic lives in
  that class. This is the file to read to understand behavior and data.
- `Jamieson Park Working List — for James.html` — the **same app, fully self-contained** (all runtime
  inlined). Open it in any browser to see/run the finished product. It is generated output — don't
  hand-edit it; treat it as a reference build.

Fidelity: **high** — this is the real, final app, not a mock. Preserve its look and behavior exactly.

## Current architecture (what you're integrating with)
- Pure client-side. No server today.
- All state is one JSON object called `board`, shape:
  ```
  board = {
    lanes: [ { id, type, name, country, mareIds: [string] } ],   // type: 'stallion'|'dry'|'late'|'new'
    mares: { [id]: { id, n, loc, if, sv, due, est, p, comment, status, manual } },
    log:   [ { t:Number(ms), text:String, c:String(hexColor) } ]  // newest pushed to end
  }
  ```
  Whole-document size is ~10–20 KB. Treat it as **one document per season** (one row / one key).
- Persistence touchpoints (search the logic class for `localStorage` — there are only a few):
  - `KEY = 'jp-working-list-2026-v4'` → the board JSON. Written by `persist(board)`; read in
    `componentDidMount`. **This is the one to move to the cloud.**
  - `KEY + '-view'` → which view ('board'|'overview'). Fine to leave per-device.
  - `KEY + '-seen'` → timestamp of last open, drives the "What's new" banner. Leave per-device
    (it's correctly a per-viewer marker), or tie it to the logged-in user if you add auth.
- `Save file` / `Open file` buttons export/import the board as JSON. Keep them as a manual backup.

## What to build

### 1. Shared store (light sync)
- **On load:** fetch the single board document; if none exists yet, seed it from the app's built-in
  `derive()` default (the original proposed matings) and write it back.
- **On change:** the app calls `persist(board)` after every edit. Make `persist` (debounced ~400–700ms)
  write the whole board document to the cloud. Last-write-wins on the whole document is acceptable for
  2–3 light users; note the small risk of one person overwriting another's simultaneous edit.
- **Seeing others' changes:** "refresh to see latest" is acceptable (owner's stated preference).
  Optional niceties: poll every ~15s, or use the provider's realtime channel to auto-refresh.
- Keep `Save file`/`Open file` as-is for backups.

### 2. Roles: editor vs viewer
- Simplest approach that needs no login UI: **two links** — an *edit link* and a *view-only link*
  (e.g. distinguished by a URL token/param, enforced by the provider's access rules).
- Add a `readOnly` flag to the app. When true: disable drag, the +/− buttons, status-circle taps,
  comment editing, Add mare/stallion, Reset, and writes (still allow viewing, the Changes log, and
  "What's new"). Most edit controls already carry the `jp-noprint` class and are easy to find; gate
  them on `readOnly`.

### 3. Keep "What's new"
- Already implemented: on open it compares `board.log` entries' timestamps against `KEY+'-seen'` and
  shows "N changes since you last opened" + per-entry **NEW** tags; "Mark seen" advances the marker.
- With a shared log, this now naturally shows what *other* people did since the viewer last opened —
  no change needed beyond keeping the `-seen` marker per viewer.

## Recommended stack (owner wants the simplest, cheapest path; free tiers are fine)
Any of these work; pick one:
- **Supabase (recommended):** one table `boards(id text primary key, data jsonb, updated_at timestamptz)`.
  Read with the anon key; gate writes with Row-Level Security so only the editor role/token can update.
  Host the static HTML on Supabase Storage, **or** on Netlify / Cloudflare Pages / GitHub Pages.
  Optional: Supabase Realtime to auto-refresh viewers.
- **Firebase Firestore:** one document holding the board; realtime sync is built in; Firebase Hosting
  for the static file. Slightly more config, nicer live updates out of the box.
- **Any tiny key-value/REST store + a static host** if the team already uses one.

Whichever: the app only needs **GET one document** and **PUT one document**, plus a way to tell
editor from viewer. Don't over-build it.

## What the (non-technical) owner will do
1. Create **one** free account on the chosen provider.
2. Send the developer the two values it generates (project URL + public key).
3. Receive back an **edit link** and a **view-only link**; share them with the manager and anyone
   who should just look.

## Deploying the static file
- The included self-contained HTML runs without a build step. Cleanest integration: keep the app as-is
  and only replace the storage calls (load in `componentDidMount`, `persist`) with cloud calls, plus
  the `readOnly` gate. You can do this in the `Working List.dc.html` logic class and re-bundle, or lift
  the logic into your own host page — your call.

## Files in this bundle
- `Working List.dc.html` — source app (template + logic). Start here.
- `Jamieson Park Working List — for James.html` — runnable, self-contained reference build.
- `README.md` — this document.

## Notes
- No external brand assets; fonts are Google Fonts (Spectral + IBM Plex Mono) loaded by URL.
- Data is the stud's own breeding info — keep the store private (don't expose write access publicly).
