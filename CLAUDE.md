# Ambassadors — Eggy Tour Map & Crew Planner

A single-file web app that maps the band **Eggy's** tour dates and lets a small
crew coordinate attendance, lodging, and logistics. Fan project, tiny audience
(~a handful of friends). Hosted free on **GitHub Pages**, backend on
**Supabase free tier**.

Live URL: https://wooklord.github.io/ambassadortracker/
Repo: wooklord/ambassadortracker

---

## Prime directive: keep it simple

The entire app is ONE file: `index.html` (~1700 lines, HTML + CSS + JS inline).
This is deliberate and is the app's main strength — deployment is "copy one file
to GitHub Pages," there is no build step, and there is almost nothing to break.

**Do not add a build system, bundler, framework, or npm dependency chain** unless
explicitly asked. If splitting the file, the ONLY sanctioned split is extracting
CSS → `styles.css` and JS → `app.js` alongside `index.html`, still with no build
step (plain `<link>` and `<script src>`). Anything heavier makes this app worse.

---

## The BUILD version ritual (important)

Near the bottom of the `<script>` there is:
```js
const BUILD = "vNN";
```
It appends "· vNN" to the header subtitle and logs to console. **Every time you
change `index.html`, bump this number** (v44 → v45 → …). It exists because mobile
browsers cache aggressively and this is how the user confirms a deploy actually
landed. Current version: **v44**. Never skip the bump.

---

## Deploy model

- Editing `index.html` (or `shows.json`) and pushing to `main` auto-deploys via
  GitHub Pages. No CI, no build.
- After a push, Pages can take 1–10 min; mobile may need a hard refresh. The
  BUILD marker is the source of truth for "did it deploy."
- The user historically deployed by uploading files in the GitHub web UI; with
  Claude Code, prefer normal git add/commit/push.

---

## External dependencies (all CDN, all keyless except noted)

- **Leaflet 1.9.4** (map) — loaded from cdnjs, with an **unpkg fallback** if
  cdnjs fails. Critical: the whole app is guarded by `const HAS_MAP = typeof L
  !== "undefined"`. If Leaflet fails to load, the app must still render the list,
  weather, and crew features — never white-screen. Preserve this guarding when
  touching map code. (This was a real production bug; see history.)
- **supabase-js v2** — from jsDelivr. Guarded by `if (window.supabase && ...)`.
- **CARTO tiles** — voyager (light) / dark_all (dark), with automatic fallback to
  openstreetmap.org tiles after 3 tile errors.
- **Open-Meteo** — free/keyless forecast API (Weather tab).
- **RainViewer** — free/keyless animated radar tiles (Weather tab).
- **OSRM** (router.project-osrm.org) — free driving routes between shows.
- **Waze embed** — iframe, live traffic on the Show tab (cropped via CSS to hide
  its chrome).
- **Nominatim** — only used server-side historically; not in the client.

## Secrets / config
- Supabase URL + **publishable key** are hardcoded near the top of the script
  (`SUPA_URL`, `SUPA_KEY`). The publishable key is SAFE to be public (that's its
  design). **Never put a Supabase secret/service-role key in this file.**

---

## Data: shows.json

The tour schedule. Loaded first from the repo (`shows.json`), then falls back to
a live Bandsintown attempt, then to a small hardcoded `FALLBACK` array. The repo
file is the real source. Refreshed manually ("the Claude method"): when tour
dates change, regenerate `shows.json` by hand — there is no automation (a GitHub
Action was attempted but every concert data source blocks GitHub's IPs).

Each show object:
```json
{
  "date": "2026-07-11",          // YYYY-MM-DD, required
  "endDate": "2026-10-25",       // optional, festivals only (multi-day)
  "seq": 1,                      // optional, ONLY for same-day ordering (see below)
  "venue": "Levitt Pavilion",
  "city": "Westport", "state": "CT",
  "lat": 41.1415, "lng": -73.3579,  // approximate (city/venue level) — fine for map
  "tickets": "https://...",      // optional
  "festival": true,              // optional, drives "Festival Set" tag + hatch timing
  "with": ["Tom Hamilton"]       // optional co-bill / support acts
}
```

### seq (same-day ordering — admin override)
Shows sort by `date`, then by `seq` (default 50) as a tiebreaker. This ONLY
matters when two shows share a date. Eggy is one band, so same-day shows are
sequential (e.g. afternoon festival set → night club show), and the drive
between them should render correctly. `seq` is the manual override: set it when
two same-day shows need a specific order. It is edited by hand in shows.json (not
in-app — the app can't write to the repo). Coordinates/times must NOT affect order.

### Travel-gap rule (subtle, don't regress)
The gap between consecutive shows is measured from the current show's **start
date** (`show.date`), NOT `endDate`. Reason: at a multi-day festival Eggy plays
one set and leaves for the next show, so measuring from endDate produced negative
gaps and phantom drives. A leg is "drivable" when gap is 0–2 days AND the venues
differ AND neither show is past. Same-venue back-to-backs (two Boston nights)
render nothing between them.

---

## The egg lifecycle (the app's signature feature)

Map pins are hand-drawn SVG eggs (`eggSVG(tier, blucifer, going)`), rendered by
`eggIcon(show, hot)`. Tiers are computed by day-count from today (LOCAL date, via
`todayStr()` / `localYMD` — never UTC, or shows hatch a night early on the East
coast):

- **tier 3** — >90 days out: smooth, muted shell
- **tier 2** — ≤90 days: hairline crack
- **tier 1** — ≤30 days: bigger zigzag crack
- **tier 0** — ≤7 days: big dramatic crack; **pulses** starting at ≤3 days;
  **double-time pulse** on show day
- **tier 4** — played (past): a **hatched chick** SVG, stays on the map for 30
  days after the show's end date, then drops off. Route lines to a show are
  removed at show completion (not at the 30-day mark).

### Blucifer (the Easter egg — keep it undocumented in the UI)
A played show hatches a **blue horse head with a glowing red eye** (Blucifer,
the Denver airport horse) instead of the chick when ANY of:
1. It's on the CURSED list — currently hardcoded: Westport 2026-07-11 (Blucifer's
   one-time debut; stays a horse until it ages off the map).
2. It was a Halloween show (Oct 31 within its date range) — cursed forever.
3. It wins a deterministic 1-in-50 hash lottery (`hashStr(date+venue) % 50`),
   so it's the same cursed shows for everyone on every device, no flicker.
Never add Blucifer to the legend. Secrets don't get legends.

Legend shows only the 5 normal tiers + "Played", plus a note that eggs pulse and
hatched pins roost 30 days.

### "Going" marker
Shows the signed-in user has RSVP'd "going" to get a small yolk dot in the egg
and a yolk-colored left border on the list card. Updates live.

---

## Supabase backend

Project: `zvwsstgnjhlybnbtvibg`. Auth = **email OTP codes** (6–8 digit; NOT magic
links — links broke in the PWA because the email app opens a different browser).
Custom SMTP via **Resend**, sending from `noreply@mail.wooklord.net`. Session
persists per device. Realtime subscriptions keep RSVPs, chat, times, and going-
marks live.

Security model: identity = verified email from the JWT. A `crew` allowlist table
gates everything via RLS; non-crew can authenticate but get nothing. Writes are
pinned to the user's own email (can't post/RSVP as someone else). All user text
is escaped via `esc()` — keep doing this on anything user-generated.

### Tables (see the phase*.sql files for exact DDL + RLS)
- **crew** — allowlist. Columns: `email` (pk, lowercase), `nickname`, `is_admin`,
  `added_at`. Nicknames are what the app displays everywhere. Only the dashboard
  or an admin can add/edit/remove members.
- **rsvps** — one row per (show_key, email): `status` (going/maybe/out), `note`.
- **messages** — per-show chat thread. Immutable (delete + repost, no edit).
- **show_times** — one shared row per show: `doors`, `showtime` (stored HH:MM,
  displayed 12-hour), `edited_by`. Any crew member can edit; last write wins.
  Native `<input type="time">` wheel; a subtle hover tooltip shows last editor.

`show_key` everywhere = `"YYYY-MM-DD|Venue Name"` (see `showKey(show)`).

Helper SQL functions: `is_crew()` and `is_admin()` (both SECURITY DEFINER).

### SQL files (run order, one-time setup per environment)
1. `phase2-schema.sql` — crew, rsvps, messages, is_crew(), RLS, realtime.
2. `phase2-showtimes.sql` — show_times table.
3. `phase3-admin.sql` — adds is_admin flag + admin RLS; bootstrap yourself by
   editing the final UPDATE to your email.

These are idempotent-ish but written for fresh setup; don't re-run blindly
against the live DB without reading them.

---

## UI structure

- **Left sidebar / bottom sheet (mobile):** header (logo, title, theme button,
  legend, data-source badge) + scrollable show list with drive-leg connectors.
- **Map:** Leaflet, egg pins, dashed orange routes with direction chevrons
  between drivable shows.
- **Drawer (per show), 4 tabs:**
  - **Show** — 2×2 grid (Date | Doors, Venue | Show), co-bill, crew-going pile,
    tickets/calendar/maps/directions buttons, live Waze traffic.
  - **Weather** — Open-Meteo daily cards (show days + 1 "getaway" day, faded);
    for non-festival shows a 3 PM–2 AM hourly grid (6 cols × 2 rows) with temp/
    rain/humidity; live RainViewer animated radar mini-map.
  - **Lodging** — Airbnb/Booking/hotel links pre-filled with the show's dates;
    "pre-show fuel" pizza/restaurants/dispensary map searches.
  - **Crew** — OTP sign-in; RSVP (going/maybe/out) + lodging note; roster list;
    per-show realtime chat.
- **Admin gear (bottom-left, admins only):** modal to add/rename/remove crew and
  toggle admin. Enforced by RLS, not just hidden UI. Can't remove/un-admin self.

### Design tokens / theme
CSS variables in `:root`; dark theme via `[data-theme="dark"]` overrides. Warm
"roasted brown" dark palette (NOT the sibling app's indigo). Three-state theme
button cycles auto → light → dark (🌗/☀️/🌙), persisted in localStorage
(`amb_theme`), auto follows the OS live. Map tiles and the `theme-color` meta
swap with the theme. Fonts: Fraunces (display) + Nunito Sans (body). Weather
numbers use the body font intentionally (serif looked bad at small sizes).

### PWA
`manifest.webmanifest` + `apple-touch-icon.png` + `icon-192/512.png` (512 is
maskable, egg on dark backing). Favicon is an inline SVG data-URI egg.

---

## Conventions & gotchas

- **Bump BUILD on every index.html change.** (Said twice on purpose.)
- **Preserve the `HAS_MAP` guards** — a blocked CDN must not white-screen the app.
- **All dates are LOCAL**, via `todayStr()`/`localYMD`, never `toISOString()` for
  "today" comparisons.
- **Escape all user content** with `esc()`.
- **Never expose the Supabase service key.** Publishable key is fine.
- Coordinates are approximate; Google Maps links use venue NAME (self-correcting),
  while map pins/OSRM use lat/lng (block-level precision doesn't matter there).
- There's a sibling app ("Fantasy Tour" / Fantasy Eggy) — different repo, shares
  aesthetic conventions but not code. Don't conflate them.
- Emojis can't be recolored; the direction chevron is a real inline SVG for that
  reason (a text glyph rendered inconsistently across platforms).

## Sanity check after edits
There's no test suite. Minimum check: the page loads, the header shows the new
BUILD version, and there are no console errors — with Leaflet both present AND
blocked (to confirm graceful degradation). The user reviews visually on mobile.
