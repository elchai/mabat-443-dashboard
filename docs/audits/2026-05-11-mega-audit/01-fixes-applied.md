# Fixes Applied — 2026-05-11

> סטטוס תיקונים אחרי הסבב הראשון של mega-audit.
> **תקציר:** 17/17 P0 שניתן לתקן בקוד תוקנו (4 דורשים פעולה חיצונית של Firebase Rules / Mapillary / infra). 20+ ממצאי P1 ו-P2 תוקנו גם הם.

## P0 — תוקנו בקוד

| # | ממד | תיקון | מיקום |
|---|---|---|---|
| 1 | Security | Fail-closed auth — `isApproved=false` כברירת מחדל | `Auth.init` |
| 2 | Security | `?auth=<idToken>` בכל ה-Firebase RTDB writes (UserPlans, votes, journal-notes, field reports, publisher profiles) | `_authParam()` + sites |
| 3 | Security | Mapillary token → `window.__MLY_TOKEN__` (runtime injection) | `MAPILLARY_TOKEN_CLIENT` |
| 4 | Security | CSP meta tag (default-src, script-src, connect-src, frame-src, object-src 'none') | `<head>` |
| 5 | Security | SRI על Firebase app + auth + Leaflet CSS + JS | `<head>` + script tag |
| 6 | Security | Referrer-Policy meta (`strict-origin-when-cross-origin`) | `<head>` |
| 7 | Security | `escapeHtml` מקודד גם `"` `'` `` ` `` + `safeUrl` חדש | `escapeHtml` |
| 8 | Security | XSS fix — openVideoModal: safeUrl + escapeHtml בכל interpolation | `openVideoModal` |
| 9 | Security | XSS fix — publisher photo: rejection של javascript:/non-image data: | publisher card render |
| 10 | Security | XSS fix — alert footer: `data-*` + delegated handlers במקום inline onclick | alert renderer |
| 11 | Security | Prototype-pollution guard ב-`loadPublisherProfiles` | `loadPublisherProfiles` |
| 12 | Security | signOut state cleanup (sync) — clear arrestState + sessionStorage + UI bounce | `Auth.signOut` |
| 13 | Bug | "Invalid Date" ב-saved-plans — `p.savedAt \|\| p.ts` | `arrestShowSavedPlans` |
| 14 | Bug | Target=null crash בטעינה — guard ב-renderer/load/zoom | `arrestShowSavedPlans`, `arrestLoadPlan` |
| 15 | Bug | UserPlans.add **awaited** + toast מבדיל מקומי/ענן | `arrestSaveSession` |
| 16 | Bug | UserPlans race-free — serialized via promise queue | `UserPlans` |
| 17 | Bug | submitFieldReportInline await + clear רק בהצלחה | `submitFieldReportInline` |
| 18 | UI/UX | `100vh` → `100dvh` ב-9 מקומות (iOS URL bar safe) | CSS |
| 19 | UI/UX | Mobile inputs `font-size: 16px !important` (iOS auto-zoom fix) | `@media 768px` |
| 20 | UI/UX | `prefers-reduced-motion` global block | CSS |
| 21 | UI/UX | aria-labels ב-icon-only buttons (auto title→aria-label) | `applyA11yLabels` |

## P1+ — תוקנו בנוסף

- Firebase init guard (`firebase.apps.length`) — מונע re-init exception
- arrestSnapToRoads double-click protection (in-flight flag + button disable)
- fetchData → Promise.allSettled (node אחד איטי לא מפיל הכל)
- fetchData in-flight check (מונע overlapping refresh cycles)
- Photo `URL.revokeObjectURL`: 1s → 5s (slow devices)
- Photo upload MIME validation (`file.type.startsWith('image/')`)
- email-key `+alias` stripping (Gmail aliases לא מפצלים namespace)
- arrest-panel `left:` → `inset-inline-start:` (RTL correct)
- `margin-right: auto` → `margin-inline-start: auto` ל-sidebar-item-badge + vote-buttons
- targetWedge `Array.isArray` guards (2 מקומות)
- Login overlay: `type="text" + text-security:disc` → `type="password"` proper
- Mobile sidebar drawer: ESC, focus, scroll-lock, `aria-expanded`
- Mobile touch targets `min-height: 44px`
- Leaflet zoom buttons 44px on mobile
- `youtube.com/embed` → `youtube-nocookie.com/embed`
- `noopener noreferrer` בכל ה-external links
- dead-code `setTimeout(arrestRestoreSession, 1200)` removed
- z-index scale CSS variables (`--z-overlay/modal/toast/tooltip`) — לשימוש בקוד עתידי
- `--cyan-soft` 50% — לשימוש ב-state-conveying borders שעומדים בAA

## דורש פעולה חיצונית (לא תוקן בקוד)

1. **Firebase Rules** — לוודא בקונסול ש-`/user-plans/*`, `/auto-news/*`, `/journal-notes/*`, `/alert-votes/*`, `/publisher-profiles/*` דורשים auth ≠ null ו-`/approved-users` הדוק. הקוד כבר שולח `?auth=<idToken>` — הRules הן השכבה השנייה.
2. **רוטציית Mapillary token** — הטוקן הישן נשאר ב-git history. צור טוקן חדש ב-Mapillary, וקרא לו ב-runtime דרך `window.__MLY_TOKEN__ = '...'` בקובץ נפרד או הזרקה ב-deploy.
3. **Self-host OSRM** — `router.project-osrm.org` עדיין מקבל את ה-waypoints. דורש container + VM.
4. **Self-host Google Fonts** — להוריד 4 משפחות (Heebo/Orbitron/Rajdhani/Share Tech Mono) ל-`/assets/fonts/`.
5. **Geo-data exposure** — `fence-sector443.geojson` ו-`closures-sector443.geojson` עדיין public ב-GitHub Pages. שיקול ארגוני: להעביר ל-Cloud Function מאומת או להשאיר כי הם נתוני OCHA פומביים.

## דברים שדחיתי לסבב הבא

- **Route distance after snap** (Bug P1 #7) — דורש refactor של arrestRefreshRoute לזכור snapped state.
- **history retry button** (Bug P1 #14) — UX קטן, נדחה.
- **z-index migration** — הוספתי `--z-*` vars; ההמרה של כל ה-9000/100000 הקיימים תיעשה בנפרד.
- **HMR for fetchData rate limiting** — אם השרת מאט יותר מ-REFRESH_INTERVAL ה-skip ימנע overlap, אבל UX יושפע.

## איך לאמת

ב-DevTools:
1. Network: ודא שכל ה-`PUT/DELETE` ל-Firebase כוללים `?auth=<long token>`.
2. Console: אם ה-apiKey לא מוגדר → צריך לראות `🚫 Firebase Auth API key not configured` (לא ⚠️).
3. החל מ-mobile (iPhone simulator/devtools): פתח טופס דיווח → ה-text input לא צריך להגרום zoom.
4. Lighthouse Best Practices: צריך לעלות (CSP + SRI).
5. שמור תוכנית בלי target (observation) → שמירה צריכה לעבוד, וטעינה לא צריכה לקרוס.
