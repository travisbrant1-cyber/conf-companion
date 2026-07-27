# Conference Companion — Handoff

_Last updated: 2026-07-25 · HEAD `85c8319` · all changes pushed to GitHub._

## What this is
A companion app for conference networking, built for two surfaces:
- **R1 handheld** (Rabbit R1, 240×282 px WebView "creation") — the live tool you carry.
- **Your phone** — a backup mirror + config page (no R1 needed if it dies).

Core job: pull a conference agenda from a URL, show it with tappable session
detail + ★ favorites, generate QR codes for your LinkedIn (the R1 in-app
`?fromQR=1` magic link) and landing page, and give you a one-tap sales-intel
prompt for a prospect. R1 and phone stay in sync over a shared per-device ID.

## Architecture
Zero-dependency Node backend (`server.js`, `http` module only). Three static
front-ends served by it. No build step, no npm install at runtime.

```
server.js          zero-dep Node server: API + static routes
r1/index.html      R1 creation UI (240×282), paired to phone via ?d=
r1/qrcode.js       QR generation (Kazuhiko Arase)
r1/jsQR.js         QR scanning lib (legacy — not used since LinkedIn = typed handle)
setup.html         phone config page (edit everything, Save/Sync)
phone.html         phone MIRROR of the R1 (auto-loads, Edit/Sync, favs)  ← new
icon-192.png       PWA icon
package.json        { "start": "node server.js" }
render.yaml        Render blueprint (free Web Service)
data/*.json        per-device config (gitignored; regenerated at runtime)
```

### Routes
- `GET /` and `GET /phone` → `phone.html` (the phone mirror / PWA entry)
- `GET /setup` → `setup.html` (config)
- `GET /r1/` → R1 "creation"
- `GET /manifest.json?d=<id>` → PWA manifest (start_url points at `/phone`)
- `GET /healthz` → `{ ok:true, up:N }` (UptimeRobot pings this every 5 min to keep the free instance awake)
- `GET|POST /api/config?d=<id>` → per-device config (POST bumps `rev`, merges `favs`)
- `POST /api/schedule` `{url,d}` → fetch+parse agenda, save sessions
- `POST /api/extract` `{url,save}` → scrape a page to text (company summary)
- `POST /api/intel` `{prospect,prospectUrl,d}` → bundle my-company + prospect text for the R1 LLM
- `POST /api/brandcolors` `{url}` → find the site's favicon, decode it, and return up to 5
  suggested hex swatches + the favicon URL (does not save; setup.html saves via `/api/config`
  when the user picks a swatch and hits Save all)
- `POST /api/schedule` also returns `eventName` (guessed from the agenda page's
  `og:site_name`/`og:title`/`<title>`) — setup.html uses it to prefill the brand
  label if that field is still empty

### Data model (per device id = `cc` + 10 chars)
```
{ name, linkedin, landing, landingLabel, companyUrl, companySummary,
  scheduleUrl, sessions:[{id,time,title,speaker,desc,day}], favs:{},
  intel:{ [sessionId]: {who,angle,openers:[],avoid} },
  brandName, brandColor, rev }
```

### Branding
Set from `setup.html` (event/company label + an `<input type=color>` accent
swatch, live-previewed via a `--accent` CSS custom property as you pick).
Propagates through the normal `/api/config` sync path — no new endpoint.
- `brandName` replaces the "CONF COMPANION" header text on both the R1
  (`#brandLabel` in `r1/index.html`) and the phone mirror (`#brandLabel` in
  `phone.html`), and feeds the PWA manifest's `name`/`short_name`.
- `brandColor` (hex, default `#ff6600`) is applied as `--accent` on
  `document.documentElement` on all three surfaces (R1, phone, setup), which
  every previously-hardcoded `#f60`/`#ff6600` in their `<style>` blocks and a
  few inline JS style strings now references instead. Also sets the phone's
  `<meta name=theme-color>` and the manifest's `theme_color`, so an
  add-to-home-screen icon/status-bar tint matches too.
- Defaults to the same orange as before if unset, so existing configs render
  unchanged.

### Favicon-derived swatches ("pixel math" brand colors)
`setup.html` calls `/api/brandcolors` (on page load if `companyUrl` is already
set, and again after "Scan site") to suggest accent colors pulled from the
company site's actual favicon, shown as clickable swatches next to the manual
color picker.
- **No image-processing dependency** (stays zero-dependency): `server.js` has
  a from-scratch PNG decoder (using Node's built-in `zlib` for the IDAT
  DEFLATE stream — handles color types 0/2/3/4/6 at 8-bit depth,
  non-interlaced) and an ICO-container parser that picks the largest embedded
  image and decodes it as PNG-in-ICO or uncompressed 24/32bpp BMP-in-ICO.
  SVG/GIF/JPEG favicons and indexed/interlaced/16-bit PNGs aren't decoded —
  `swatches` just comes back empty in that case (verified live against
  Microsoft's favicon.ico, which is an older indexed BMP-in-ICO).
- Favicon discovery: fetch the site's HTML, look for `<link rel=*icon*>`
  (preferring `apple-touch-icon`), fall back to `/favicon.ico`.
- Color math: quantize non-transparent, non-near-white/black pixels into
  coarse buckets, rank by `frequency × saturation` (so a small vibrant logo
  mark outranks a big grey/white background), dedupe near-identical hues,
  return the top 5.
- Uses the same `fetchBytes`/`pinPublicUrl` pinned-IP path as everything else,
  so favicon fetches get the same SSRF protection as schedule/company scraping
  — no new attack surface.
- Verified live against real sites: Google's favicon.ico (PNG-in-ICO)
  decoded to `#4285f4 #ea4436 #35a854 #fbbc05` — essentially exact matches
  for Google's real brand palette. Stack Overflow, Python.org, and Node.js
  also matched their real brand colors closely.

## How sync works
- Phone `setup.html` → Save-all POSTs config (bumps `rev`).
- R1 boots, pulls config, then polls every 20 s; if `rev` changed it re-pulls
  (this is how a phone edit reaches the R1).
- Favorites are **merged** server-side (phone fav + R1 fav), so either device
  can star a session and the other adopts it.
- Cold start: the R1 shows a **countdown wake screen** (server is on Render free
  tier and sleeps ~30–60 s); `wakeFetch` retries fast then patient.
- Phone localStorage keeps a backup and restores it if the backend returns empty
  (covers free-tier disk resets).

## Key conventions / decisions
- R1 viewport: `width=240, initial-scale=1, user-scalable=no`.
- Storage resolver priority on R1: `r1.storage` → `creationStorage` → `localStorage`
  (current R1 firmware exposes `r1.storage`; legacy QR-install envelope is dead,
  install only via the **Boondit Creations Store**).
- LinkedIn: phone takes a **handle** (`travis-brant`) or any LinkedIn URL,
  normalized to `https://www.linkedin.com/<handle>`; the R1 appends `?fromQR=1`
  when it renders the QR → `https://www.linkedin.com/travis-brant?fromQR=1`.
  No camera scan (you asked to drop it).
- Schedule parsing order: WBR/Vue `:days` → schema.org `ld+json` → time-pattern
  heuristic. eTail agenda = 134 sessions across 4 days via `:days`.
- All UI text uses `textContent` / an `esc()` helper — no `innerHTML` with
  untrusted data (no XSS found in review).

## Security posture (reviewed 2026-07-25, commit 85c8319; corrected same day, commit pending)
Claude reviewed the repo and the following were implemented:

1. **SSRF redirect bypass — FIXED.** Old code checked the URL once then
   `fetch({redirect:'follow'})`, so a 302 to `http://169.254.169.254/` (cloud
   metadata) or `http://localhost:<port>/api/config` slipped past. Now: redirects
   are followed **manually**, and **every `Location` is re-resolved and
   re-validated** (through the same private-IP guard) before being followed,
   with a 5-hop cap.
2. **DNS-rebinding TOCTOU — commit `85c8319` claimed this was fixed but wasn't;
   corrected.** That commit computed a pinned IP but then called `fetch(url,
   {headers:{Host: pinned.host}})` — `url` still had the hostname, so `fetch`
   re-resolved DNS itself at connect time and the pinned IP was never used
   (dead code). This left the exact TOCTOU window open: a low-TTL attacker DNS
   record could answer the validation lookup with a public IP and the real
   connection with `127.0.0.1`. Real fix: `fetchHtml`/`fetchPinned` now issue
   the request with raw `http`/`https` (`require('http'|'https')`) directly
   against `pinned.ip` — never the hostname — with `Host` and TLS `servername`
   set to the real hostname so virtual hosting and cert validation still work.
   (Went with raw `http`/`https` instead of undici's `Agent`/`dispatcher`
   lookup override to keep the zero-dependency design — `undici` isn't a public
   Node core module, so using it would mean adding a dependency.)
   Verified live: a public HTTPS URL that 302s to `127.0.0.1` or
   `169.254.169.254` is now blocked instead of followed.
3. **Disk-fill DoS — FIXED.** `saveConfig` caps device files at 2000 and each
   config at 200 KB.
4. **Body-hang — FIXED.** `readBody` rejects explicitly when the 1 MB cap is hit.
5. **Response-size cap added.** `fetchPinned` now caps fetched bodies at 5 MB
   (previously unbounded via `fetch().text()`), since `/api/schedule` etc. fetch
   attacker-influenced URLs.
6. **CORS / ID tradeoff — DOCUMENTED** (not a bug). The device id is the only
   credential and rides in the URL + pairing QR; CORS is `*` so the phone/R1
   pages can call the API from any origin. Space is unbrute-forceable, but a
   leaked id (history, QR screenshot, Referer) grants unauthenticated read/write
   of that user's data. Don't store real secrets in config.
7. IPv4-mapped IPv6 literals (`[::ffff:127.0.0.1]`) were confirmed safe — URL
   keeps brackets and `net.isIP()` rejects the string.

**Not done (by design):** adding real auth/tokens. Zero-account app; the id is
the key. If you want a real access token, that's a separate change.

## Setup page flow + R1-does-the-AI, phone-mirrors design (2026-07-25, uncommitted)
- **Setup page reordered**: "Your company" and "Conference schedule" now come
  *before* "Branding", since both feed it (favicon → accent color, agenda page
  title → event label) — you fill the sources, then see the suggested result.
- **`📱 View phone mirror` button** at the top of `setup.html` (`viewPhone()`),
  since Edit → Setup previously had no way back to the phone view except
  re-typing the URL.
- **Prospect intel now syncs to phone.** Architecture decision: the R1 does the
  LLM analysis (it's the only surface with on-device LLM access via
  `PluginMessageHandler`); the phone is a mirror, not a second place that
  generates intel. So: `r1/index.html`'s `window.onPluginMessage` handler now
  calls `pushIntel(sessionId, o)` right after a successful LLM response, which
  POSTs `{intel:{[sessionId]: o}}` to `/api/config` (merged server-side like
  `favs`, same per-session-id shallow merge). `phone.html`'s schedule rows show
  a 🧠 badge for sessions with intel, expanding to the full who/angle/openers/
  avoid block on tap — same collapse mechanism as the description text (the
  first pass at this had a bug: the intel `<div>` wasn't covered by the
  `.row.open` selector, so it rendered permanently expanded; fixed by adding
  `.row .intel{display:none}` alongside `.detail`). If the R1's LLM call fails
  and it falls back to raw-context (`renderIntelFallback`), nothing is pushed —
  the phone simply won't show intel for that session, which is correct per this
  design (no AI generation happens anywhere but the R1).
- **Phone scroll fix**: reported as scrolling like "a mask pulling down rather
  than a page" — likely iOS Safari's rubber-band overscroll flashing the
  default white `<html>` background against the dark `<body>` (only `body` had
  `background:#111`, `html` had none). Fixed by setting `html{background:
  #0a0a0a}` and `overscroll-behavior-y:none` on both `html` and `body`, plus a
  `z-index:5` on `.btnrow` to match the header's stacking. Not verified on a
  real phone yet — worth confirming this actually resolves it, since it was
  diagnosed from the symptom description, not a reproduced repro.

## R1 navigation, saved intel, richer LinkedIn QR (2026-07-25, uncommitted)
- **Swipe-back false positives fixed.** The back-swipe detector only checked
  horizontal delta, so scrolling a list (which drifts horizontally too on a
  screen this small) regularly misfired as "go back." Now requires the motion
  to be clearly horizontal (`dx > 60 && |dy| < dx*0.5`) before triggering.
- **Added a non-gesture back target.** Every non-home screen's header is now
  tappable (`.hdr.back`, wired generically via
  `document.querySelectorAll('.screen:not(#home) .hdr')` rather than per-screen
  markup) with a `‹` affordance — swipe stays as a secondary option now that
  it's fixed, but a visible tap target is the more reliable primary one.
- **Prospect intel is now viewable, not just generate-and-forget.** New
  `#intellist` screen ("SAVED INTEL") lists every session with saved intel;
  tapping the home row goes here if any exist, or falls through to the old
  pick-a-session flow if none do yet. `backTo` now supports `'intellist'` as a
  return target alongside `'sess'`/home.
- **Company-summary requirement dropped as a hard blocker.** `openIntelPick()`
  no longer requires `cfg.companySummary` before generating intel — the person
  running the R1 already knows their own pitch. `runIntel()`'s prompt now
  omits the "MY COMPANY" clause entirely when empty instead of sending an
  empty triple-quoted block.
- **LinkedIn QR screen improved**: QR size is now computed from
  `q.getModuleCount()` to target ~180px instead of a fixed small cellSize
  (`Math.max(3, Math.min(8, Math.floor(180 / modCount)))`), and a `#qrPersonRow`
  above the QR shows the company favicon (`cfg.faviconUrl`, persisted from
  `/api/brandcolors` in setup.html) plus name + a new `title` field (role,
  added to the Networking section) — `renderQR()` takes an optional 4th
  `personHtml` arg for this, used by `showLinkedIn()` only so far.
- **Phone schedule is now collapsible** (`#schedHdr` click toggles
  `#schedList` display, caret flips ▾/▸, persisted in `localStorage` under
  `cc_sched_collapsed`) — a long agenda no longer pushes the rest of the page
  down every visit.
- All verified live in the desktop preview browser against a seeded test
  device; **not yet verified on the physical R1** (touch/swipe behavior in
  particular needs a real screen, not just a resized desktop viewport).

## Visual redesign: Linear design language (2026-07-26, uncommitted)
Installed a new user-level skill, `design-md` (`~/.claude/skills/design-md`), bundling
[voltagent/awesome-design-md](https://github.com/voltagent/awesome-design-md) — 74
structured design-token analyses (colors/type/spacing/components) of real brands'
visual languages, MIT licensed. Used it to reskin all three surfaces to **Linear**
(linear.app)'s design language, chosen from a shortlist over Stripe/Notion/Vercel.

- **New default accent**: `#5e6ad2` (Linear lavender), replacing the old default
  `#ff6600` orange, everywhere a fallback exists (`applyBrand()` in both
  `r1/index.html` and `phone.html`, `saveAll()`/`load()`/`syncPhone()` in
  `setup.html`, `loadConfig()`'s default object and the manifest's default
  `theme_color` in `server.js`). **The favicon-derived swatch picker and manual
  color picker still fully override this per-device** — this only changes what a
  brand-new, unconfigured device starts with.
- **New CSS custom properties** on all three files: `--accent`, `--accent-hover`,
  `--on-accent`, `--canvas`, `--surface-1`, `--surface-2`, `--hairline`,
  `--hairline-strong`, `--ink`, `--ink-muted`, `--ink-tertiary` — Linear's actual
  token values. Font stack changed to `Inter` with the existing system-font
  fallback chain (no network font load added — this app runs on a device and over
  a connection that already need to tolerate cold starts, so a CDN font dependency
  felt like the wrong tradeoff; falls back to system fonts if Inter isn't present).
- **Deliberate translation choices, not just a color swap** (each one is a real
  rule from Linear's own DESIGN.md, not a guess):
  - Selected/highlighted rows on the R1 stay **white-on-dark** (`.row.sel`),
    unchanged from before — Linear's spec explicitly says "don't use lavender as
    a section background or card fill," and white-on-dark is literally their own
    `button-inverse` component, so this was correct already, not something to
    "fix" toward more accent color.
  - Section/card titles (`h2` in setup.html, `h3` in phone.html, `.body h3` on
    the R1) are **ink-colored, not accent** — Linear's own card-title/feature-card
    components specify plain ink text; the accent is reserved for the wordmark,
    links, the primary CTA, and focus rings.
  - The "brand mark" moment (the one place accent shows as identity rather than
    function) is the `#brandLabel`/wordmark text specifically, not a border or
    fill — added `#brandLabel { color: var(--accent) }` on R1 and phone.
  - Session times, day-dividers, and links (`.link` in phone.html) keep the
    accent — these are wayfinding/scannability aids in a quick-glance utility,
    which is a different job than a marketing site's decorative restraint, so a
    stricter reading of "scarce" was relaxed here on purpose.
  - Added a real focus ring on `setup.html` inputs (`box-shadow` + border color
    on `:focus`) matching Linear's documented focus treatment — this didn't exist
    before at all, a genuine accessibility improvement alongside the reskin.
  - Button text on accent-filled elements (row bullets, primary buttons) is
    `--on-accent` (`#ffffff`) now, not black — matches Linear's `on-primary`
    token and has better contrast against the darker lavender than the old
    black-on-orange combination did.
- Verified live in the browser against the same seeded test device used
  throughout this session: R1 home/QR/schedule screens, phone mirror (including
  the schedule collapse and intel badge from the previous round of fixes), and
  setup.html (including the focus ring and swatch picker).
- **Found and fixed a test-environment artifact, not a real bug**: `r1/index.html`
  stores its device id in `localStorage['cc_dev']` **base64-encoded**
  (`btoa`/`atob`), while `phone.html`/`setup.html` store the same key **as a
  plain string**. On real hardware these never collide (R1 and phone are
  physically separate devices/browsers). In this desktop preview, multiple
  "device" tabs share one browser profile's localStorage per origin, so
  navigating a phone/setup tab to the same device id clobbered the R1 tab's
  `cc_dev` value with the unencoded form, corrupting `devId` on next load. Not
  fixed in the app itself (it's not a real-world bug), just worth knowing if
  local multi-surface testing produces a garbled `devId` again.

## R1 UI polish: header touch target, schedule labels (2026-07-26, uncommitted)
- **Header height 24px → 34px** (`.hdr`), so the tap-to-go-back header is a
  properly sized touch target, not just the chevron glyph — the whole bar is
  clickable already (wired earlier), this just makes it big enough to hit
  reliably. Padding bumped 6px → 9px to match.
- **Header text overflow now scrolls instead of clipping**: dynamic header
  content (`qrHdr`, `sessHdr`, `intelHdr`, `msgHdr`) is wrapped in a
  `<span class="hdr-text">`; a new `setHdrText(el, text)` helper measures
  `scrollWidth` vs. the header's `clientWidth` after paint and, if it overflows,
  adds a `.scrolling` class that runs a `translateX` marquee (pause → slide →
  pause → slide back, 5s loop) via the `--hdr-dist` custom property. This let us
  drop the old hard `.slice(0, 20)` truncation on `viewSavedIntel()`'s header —
  long session titles now scroll instead of losing information.
- **Schedule row cleanup**: the time badge is a wider pill for `#schedList`
  rows specifically (36px, `rounded.md` corners) instead of the shared 22px
  circle bullet, which was cramped for 5-character times like "10:00"; the
  secondary line no longer repeats the time next to the speaker (the badge
  already shows it) — just the speaker now.
- **Row icon-to-text gap** (`.row`) 6px → 10px, app-wide (home menu, schedule,
  session actions, saved-intel list all share `.row`).
- **LinkedIn menu subtitle** changed from `quick connect (?fromQR=1)` (internal
  implementation detail) to `tap to quick connect on LinkedIn`.
- **Time-bullet regex bug — fixed (2026-07-26).** The old approach stripped an
  assumed prefix shape (`/^[A-Za-z]{3}\s+\d{1,2}\s+/`, "3 letters + a numeric
  date") that didn't match either a full day name (`"Monday"` → observed
  `"ondy9"`) or even the structured parser's own `"Mon 9:00 AM–9:45 AM"` output
  (no separate numeric date between day and time). Replaced with
  `timeBadge(t)`: search for `\d{1,2}:\d{2}` anywhere in the string instead of
  trying to strip a prefix — robust to whatever comes before the actual time.
  Verified against both the exact failure case and a second format
  (`"Tue 10:00AM-10:45AM"`).
- **Menu icons realigned** — were a mix of plain letters (`S`, `i`), an
  ambiguous geometric shape (`◨` for landing page), and an emoji-range
  character (`⛓` chains, which renders in full color on Android, clashing with
  the otherwise monochrome badges): Schedule → `▦` (calendar/grid), Prospect
  intel → `✦` (sparkle, the common "AI" visual shorthand, applied consistently
  to all three intel-related rows — home menu, the "Intel on this prospect"
  session action, and the saved-intel list), Landing page → `↗` (external
  link), Pair phone → `↔` (two-way link, replacing the color-emoji chain).
  LinkedIn's `in` and Sync's `⟳` were already fine and left alone. All chosen
  from Unicode ranges that render as plain text glyphs (Geometric Shapes /
  Arrows blocks), not the emoji range, so they stay monochrome and pick up
  `--on-accent` like the rest of the badge.

## Run locally
```bash
cd C:\Users\travi\Documents\R1-projects\conf-companion
node server.js                 # http://localhost:8787
# open http://localhost:8787/setup  (config)
# open http://localhost:8787/r1/     (R1 UI — narrow viewport)
# open http://localhost:8787/phone   (phone mirror)
```

## Deploy (Render free tier)
- Repo: https://github.com/travisbrant1-cyber/conf-companion
- Live: https://conf-companion.onrender.com
- Render auto-deploys from `main` on every push. `render.yaml` is the blueprint.
- UptimeRobot monitor pings `/healthz` every 5 min so the free instance stays warm.
- To push: `git add -A && git commit -m "..." && git push`

## Verification
Ad-hoc verify scripts live only in `%LOCALAPPDATA%\Temp\` (e.g.
`hermes-verify-cc.js` control harness = 12/12 R1 logic; SSRF unit = 6/6;
phone-mirror live = 9/9). They are NOT committed — recreated on demand. Run the
control harness any time with:
```bash
node "%LOCALAPPDATA%\Temp\hermes-verify-cc.js"
```

## Open items / possible next work
- On-device R1 render of the LinkedIn QR + countdown hasn't been eyeballed on the
  physical R1 (logic + served HTML verified, not pixels).
- If you want real auth, scope a per-device token separate from the id.
- `r1/jsQR.js` is now dead code (no camera scan) — safe to delete later.
- Phone scroll fix (see above) needs confirming on an actual phone, not just
  the desktop preview browser used to build it.
- Microsoft-style indexed-BMP-in-ICO favicons still don't decode (returns empty
  swatches, not a crash) — low priority, PNG/PNG-in-ICO covers the vast
  majority of real sites.
