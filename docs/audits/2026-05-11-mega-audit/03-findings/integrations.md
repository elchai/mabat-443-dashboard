# Integrations — 2026-05-11

Audit target: `index.html` (9715 lines) — single-file dashboard, GitHub Pages hosted.
Scope: every external URL the browser hits at load or runtime.

---

## Current integrations

| # | Service | Where (file:line) | Pinned? | SRI? | Risk | Notes |
|---|---|---|---|---|---|---|
| 1 | Firebase JS SDK (app + auth, compat) | `index.html:13-14` — `https://www.gstatic.com/firebasejs/10.7.1/firebase-{app,auth}-compat.js` | Yes (10.7.1) | **No** | Med | Compat build is the legacy bundle; modular SDK is half the size. Loaded from Google CDN, served with strong caching but no SRI. |
| 2 | Firebase Realtime DB (europe-west1) | `index.html:4460` — `FIREBASE_URL = 'https://project-8874611670473210078-default-rtdb.europe-west1.firebasedatabase.app'` | n/a | n/a | **High** | Used by 20+ `fetch()` sites: `alert-votes`, `journal-notes`, `auto-news`, `weather`, `calendar`, `sentiment`, `trends`, `channel-discoveries`, `publisher-profiles`, `system-keywords`, `approved-users`, `user-plans`. All anonymous `fetch` (REST), not the SDK — security rules are the *only* gate. |
| 3 | Firebase Auth (Google popup) | `index.html:9453-9458,9527-9532` — config block, `signInWithPopup` | Yes | n/a | Med | API key + appId embedded in client (expected for web SDK). `apiKey: AIzaSyCwj7jy1GaAdf1nGt3nBeLZgQlyOKWOM4o` is public-ish but should be domain-locked in Google Cloud Console. |
| 4 | Leaflet 1.9.4 CSS | `index.html:11` — `https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.css` | Yes | **No** | Med | Cloudflare CDN. No SRI ⇒ a compromised cdnjs entry could inject CSS-based exfil. |
| 5 | Leaflet 1.9.4 JS | `index.html:4453` — `https://cdnjs.cloudflare.com/ajax/libs/leaflet/1.9.4/leaflet.min.js` | Yes | **No** | **High** | Same CDN — JS injection on a military dashboard is the worst-case scenario. Add SRI hash. |
| 6 | Google Fonts (Heebo, Orbitron, Rajdhani, Share Tech Mono) | `index.html:21` `@import url(https://fonts.googleapis.com/...)` | n/a | n/a | Med (privacy) | Every page load pings Google with the operator's IP + UA. **Operational fingerprint leak.** |
| 7 | OpenStreetMap tiles | `index.html:7442` — `https://{s}.tile.openstreetmap.org/{z}/{x}/{y}.png` | n/a | n/a | Med | Default base layer. OSM logs tile requests with IP + zoom + tile = approximate coords of what the operator is viewing. |
| 8 | ArcGIS World Imagery (satellite) | `index.html:7429` — `https://server.arcgisonline.com/.../World_Imagery/...` | n/a | n/a | Med | Esri sees every tile request. No API key required but heavy use violates ToS. |
| 9 | Google Maps tiles (sat + hybrid) | `index.html:7433,7436` — `https://mt{s}.google.com/vt/lyrs=s` and `lyrs=y` | n/a | n/a | **High** | Undocumented Google internal endpoint, no ToS coverage, no key. **Google logs each tile**. Likely to break without warning. Also = privacy bleed to Google. |
| 10 | OpenTopoMap tiles | `index.html:7439` — `https://{s}.tile.opentopomap.org/{z}/{x}/{y}.png` | n/a | n/a | Low | Volunteer infra, rate-limited, occasional outages. |
| 11 | Carto basemap + labels | `index.html:7445,7451` — `https://{s}.basemaps.cartocdn.com/dark_all/...` and `/voyager_only_labels/...` | n/a | n/a | Low | Carto free tier explicitly excludes military use in ToS. |
| 12 | Overpass API (OSM queries) | `index.html:5089,5117` — `https://overpass-api.de/api/interpreter?data=...` | n/a | n/a | Med | Public Overpass logs each query string — i.e. the operational area-of-interest. Shared instance, also flaky under load. |
| 13 | Mapillary Graph API + embeds | `index.html:5303` token, `:5371,:7501` API, `:5489` `mapillary.com/embed` iframe | Hard-coded token | n/a | **High** | **Client token `MLY|26636963299290715|555a2cc2563d0e534e9b49896c447f93` is committed in source.** Anyone can scrape it. Token can be revoked → silent feature loss. |
| 14 | OSRM public router | `index.html:7004` — `https://router.project-osrm.org` | n/a | n/a | Med | "Demo server, no SLA" per OSRM docs. The fetched URL contains *every arrest-route waypoint as lat/lng query string* — full plan leaks into OSRM access logs. |
| 15 | YouTube embed iframe | `index.html:5429` — `https://www.youtube.com/embed/${vid}` | n/a | n/a | Low | Google tracks viewer. Use `youtube-nocookie.com` instead. |
| 16 | WhatsApp deep links | `index.html:3789,4449,6737` — `https://wa.me/...` | n/a | n/a | Low | Just `<a href>`, no script load — fine. |
| 17 | `news.google.com` + `t.me/...` outbound links | scattered (`:4627,:4891,:5688,:8015,:8759`) | n/a | n/a | Low | Plain `<a>` tags; non-issue. |

---

## Issues with current integrations

### P0
1. **Mapillary client token committed to repo** (`index.html:5303`) — public GitHub Pages repo means the token is in the git history forever. Rotate immediately; even though it's a "client" token, it's still rate-limited per token and tied to your account.
2. **No SRI hashes on three external scripts/styles** (`index.html:11,13,14,4453`) — Firebase + Leaflet load with full JS execution privileges. A compromised gstatic or cdnjs entry compromises every signed-in operator. Add `integrity="sha384-..."` + `crossorigin="anonymous"`.
3. **No Content-Security-Policy** (no `<meta http-equiv="Content-Security-Policy">` found in `index.html:1-15`). For a military dashboard, the absence of CSP is the single biggest XSS amplifier — note that the codebase uses heavy `innerHTML` patterns.

### P1
4. **Firebase Realtime DB accessed by raw REST** (`index.html:4678, 4692, 4696, 4704, 4721, 4732, 4810, 5672, 7721-7727, 8412, 8491, 8727, 8850, 8880, 9504, 9640, 9657`) — none of the `fetch(${FIREBASE_URL}/...json)` calls send an `Authorization: Bearer <idToken>` header. So **either** the DB is wide-open at the read/write layer **or** these endpoints silently fail for unauthenticated users. Verify rules file and switch to authenticated REST (`?auth=<idToken>`) or move to the SDK.
5. **Operational route data leaked to OSRM public demo** (`index.html:7004-7016`) — the URL `router.project-osrm.org/route/v1/driving/<all-waypoints>` exposes the exact arrest plan to a public log. Self-host OSRM or use Mapbox Directions with a key.
6. **Overpass queries leak AOI** (`index.html:5089, 5117`) — each query string contains the polygon being investigated.
7. **Google Maps tiles via internal `mt{s}.google.com`** (`index.html:7433,7436`) — unsupported endpoint; violates ToS; could be cut off. Use Mapbox Satellite (paid, with a key) or stay on ArcGIS only.

### P2
8. **Google Fonts as @import** (`index.html:21`) — every page load contacts `fonts.googleapis.com` + `fonts.gstatic.com`. Self-host the 4 woff2 files (~120KB total) under `/assets/fonts/` and drop the @import.
9. **No fallback if cdnjs is down** — Leaflet loads from one origin only. Either add a fallback (`onerror` → second CDN) or vendor it.
10. **Firebase compat SDK** (`index.html:13-14`) — `firebase-app-compat` + `firebase-auth-compat` is ~150KB gzipped; modular v10 is ~70KB. Worth migrating because every user pays the byte cost.

---

## Recommended new integrations

| # | Service | Value | Effort | Risk |
|---|---|---|---|---|
| A | **OSM Nominatim** reverse-geocode (`https://nominatim.openstreetmap.org/reverse?lat=&lon=&format=json`) | Replaces the manual coord→village list (`COORDS-TODO.md`). Auto-fills settlement names on map clicks. | S (one fetch, ~30 LoC) | Low — public usage policy: 1 req/sec, set `User-Agent`. Better: self-host or use a paid provider for military use. |
| B | **Mapbox Satellite + Directions** (paid, ~$5/month for this scale) | Replaces both the unofficial Google sat tiles **and** the OSRM public demo with one keyed, SLA-backed vendor. | M (swap tile URLs + Directions API shape) | Med — Mapbox sees your usage but is contractually bound (DPA available); token should be domain-locked. |
| C | **Service Worker (offline-first)** | Field officers with bad LTE get cached map tiles, last-known journal, last weather. Critical for ops near fence. | M (~150 LoC, manifest + sw.js) | Low — cache invalidation is the only gotcha. |
| D | **Print stylesheet + `window.print()` for arrest-plan page** | Operational briefing → PDF without external service. Already have a 5-step wizard; just needs `@media print` rules. | S | None |
| E | **Web Share API** (`navigator.share({title, text, files})`) | One-tap share of plan summary or screenshot to ops WhatsApp. Native on Android/iOS. | S (10 LoC) | Low — falls back gracefully on desktop. |
| F | **Hebrew Calendar via Hebcal API** (`https://www.hebcal.com/hebcal?cfg=json&...`) | Already showing `/calendar/current.json` from Firebase — but Hebcal direct gives chag-erev/motzaei flags accurate to the day. Useful for ops timing. | S | Low — public API, no key. Consider self-host of the JS lib for privacy. |
| G | **IMS open data (Israel Meteorological Service)** (`https://api.ims.gov.il/v1/envista/...`) | Replaces whatever populates `/weather/current.json`. Free, official, IDF-grade. Wind direction matters for arrest ops. | M (needs free token via gov form) | Low |
| H | **Self-hosted error logger → Firebase RTDB** (`/errors/<ts>`) instead of Sentry | Sentry would exfil stack traces with hostnames/IDs. A 30-line custom `window.onerror` handler that writes to RTDB keeps everything in your tenant. | S | None — keeps data inside Firebase project. |
| I | **SRI hash generator script** (`scripts/sri-update.sh`) committed to repo | Automates rotating SRI hashes when bumping Firebase/Leaflet versions. Prevents the SRI from going stale and getting silently removed. | S | None |

Skipped (asked but not recommended):
- **Google Maps JS API satellite layer** — same privacy issue as the unofficial tiles, plus cost. Mapbox is the cleaner answer.
- **Sentry / PostHog** — both phone home to vendor infra. For military data, recommendation H replaces them.
- **Bituach Leumi / IDF API** — confirmed not viable, no public surface.

---

## Recommended removals

1. **Drop `mt{s}.google.com` tile layers** (`index.html:7433,7436`) — unauthorized endpoint; replace with Mapbox Satellite (recommendation B) or just keep ArcGIS.
2. **Drop the `@import` to Google Fonts** (`index.html:21`) — self-host the 4 families. Saves ~2 cross-origin requests per page load, removes Google fingerprinting.
3. **Drop OSRM public demo** (`index.html:7004`) — replace with Mapbox Directions (recommendation B) or self-hosted OSRM. The current setup leaks the *exact arrest-plan route* to a public service.
4. **Drop or rotate the Mapillary token** (`index.html:5303`) — if Mapillary stays, move the token to a build-time injected value, accept that it's still visible in the bundle, and rotate quarterly. If street-level imagery isn't core, drop it entirely (it's the only reason that whole token is exposed).
5. **`youtube.com/embed` → `youtube-nocookie.com/embed`** (`index.html:5429`) — one-word change, kills the YouTube tracking cookie on operator browsers.

---

## Privacy considerations (military context)

The dashboard currently leaks operational metadata to **at least 9 external parties** per session:

| Party | What they see |
|---|---|
| Google (gstatic, fonts, mt{s}) | IP, UA, every map tile coord = areas of operational interest, every font request = page load timing |
| Cloudflare (cdnjs) | IP + UA on every page load |
| Esri / ArcGIS | satellite tile coords = current AOI |
| OpenStreetMap Foundation | tile coords |
| OpenTopoMap | tile coords |
| Carto | basemap + label tile coords |
| Overpass-turbo | full polygon of each OSM query (= AOI boundary) |
| Mapillary (Meta) | bbox of every street-imagery search |
| project-osrm.org | **every waypoint of every arrest plan**, in cleartext URL |

For a military-sector tool, items 7 ("overpass logs my AOI polygon"), 9 ("OSRM logs my arrest route"), and the Google Fonts ping are the most concerning because they tie *operational intent* (not just "user looked at a map") to an IP address. Items 1, 4, 6 are baseline web-app exposure but should still be self-hosted to remove the third-party dependency.

Quick wins (1-day work):
- Self-host fonts + Leaflet + Firebase compat bundles ⇒ removes Google Fonts and cdnjs as parties.
- Switch to `youtube-nocookie.com` ⇒ removes YouTube cookie.
- Add SRI on the three remaining external scripts ⇒ defense-in-depth even if cdnjs/gstatic is compromised.
- Add a minimal CSP (`default-src 'self'; script-src 'self' 'unsafe-inline' https://www.gstatic.com; connect-src 'self' https://*.firebasedatabase.app https://www.googleapis.com; img-src 'self' data: https://{tile-domains}; style-src 'self' 'unsafe-inline'`) ⇒ contains the blast radius of any XSS.

Medium-term (1-2 weeks):
- Move OSRM + Overpass + Mapillary behind a thin proxy you control (Cloudflare Worker on `mabat.fine.dev` or a Firebase Function). Operators hit your proxy; the proxy hits the upstream. Logs land in your tenant only.
- Or self-host OSRM (one Docker image, single VM) since the AOI is small and bounded.

---

DONE — integrations.md written
