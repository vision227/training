# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Single-file Progressive Web App (PWA) for tracking workout sessions. Vanilla JS/HTML/CSS, no frameworks, no build step, no backend. All data stored in browser localStorage.

## Deployment

```bash
vercel --prod          # Deploy to Vercel (serves workout-pwa/ directory)
```

No build, lint, or test commands — the app is a single `index.html` with inline CSS and JS.

**After any deploy**, bump the cache name in `sw.js` (e.g. `rpt-tracker-v5` → `rpt-tracker-v6`) so the service worker update mechanism triggers a reload for existing users.

## Architecture

### File Layout

- `workout-pwa/index.html` — Entire app (~2300 lines): styles, HTML, and JS all inline
- `workout-pwa/sw.js` — Service worker: network-first for HTML, cache-first for assets
- `workout-pwa/manifest.json` — PWA manifest (app name: "Train")
- `vercel.json` — Vercel config with no-cache headers for `sw.js`

### Data Model (localStorage)

| Key | Helper | Contents |
|-----|--------|----------|
| `wh` | `db.hist()` / `db.saveHist()` | Array of completed workouts (date, wid, duration, bw, exercises with sets) |
| `ws` | `db.sess()` / `db.saveSess()` | Current in-progress session (null when idle) |
| `wc` | `db.sched()` / `db.saveSched()` | Array of scheduled workouts ({date, wid}) |
| `bw` | `getBw()` / `saveBw()` | Current bodyweight (number) |

### Key Data Structures in JS

- **`PROGRAMS`** → array of programs, each with `workouts` array containing exercise templates (name, sets with target rep ranges and rest times)
- **`WORKOUTS`** → flat list derived from `PROGRAMS.flatMap(p => p.workouts)`
- **`ALTS`** → object mapping exercise names to arrays of substitute exercises
- **`STEP`** → weight increment per exercise (used for +/- buttons)
- **`BENCH`** → strength benchmark standards (bodyweight multipliers)
- **`session`** → global state for active workout; has `exercises[]` with `origName` for swap tracking

### Navigation

Four screens: `home`, `workout`, `stats`, `calendar` — toggled via `show(id)`. When a session exists, `show('home')` renders the workout screen instead.

### Service Worker Update Flow

1. `sw.js` uses `skipWaiting()` + `clients.claim()` on install/activate
2. On activate, if old caches exist (version upgrade), posts `SW_UPDATED` message to all clients
3. `index.html` listens for both `controllerchange` and `SW_UPDATED` message → reloads
4. First-install doesn't trigger reload (checks `hasController` / old cache existence)
5. Vercel serves `sw.js` with `no-cache` headers so browsers always fetch latest

### Exercise Swap System

Exercises track `origName` (from template) and `name` (current). ALTS lookups use `origName` so the swap button persists after switching. Swap options include the original exercise for switching back.

## Conventions

- All UI elements use `$('id')` shorthand (defined as `document.getElementById`)
- Exercise/workout data changes always call `db.saveSess(session)` immediately
- `renderW()` must be called after modifying session to update the workout UI
- The app is iOS-PWA-first: uses `env(safe-area-inset-*)`, `-webkit-tap-highlight-color`, standalone detection
- CSS variables defined on `:root` — `--warm` (gold accent), `--cool` (blue accent), `--bg` (near-black)
