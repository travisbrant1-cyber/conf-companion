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

### Data model (per device id = `cc` + 10 chars)
```
{ name, linkedin, landing, landingLabel, companyUrl, companySummary,
  scheduleUrl, sessions:[{id,time,title,speaker,desc,day}], favs:{}, rev }
```

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
