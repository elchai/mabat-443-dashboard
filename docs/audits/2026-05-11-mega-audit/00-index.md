# Mega-Audit — mabat-443-dashboard — 2026-05-11

> סבב #1. מסגרת: Static HTML+JS יחיד (~441KB, 9715 שורות) + GeoJSON + Firebase Auth/RTDB + Leaflet. אירוח: GitHub Pages.
> סה"כ ממצאים פעולה: **78** (Security 24, Bugs 27, UI/UX 17, Mobile 10) + 10 פיצ'רים מומלצים + מפת דרכים 3-אופקים.

## תקציר מנהלים

| חומרה | כמות | תקציר |
|---|---:|---|
| **P0 (קריטי)** | **17** | Auth fail-open ל-true • Firebase REST ללא idToken (writes פתוחים) • Mapillary token חשוף • saved-plans crash (`Invalid Date` + TypeError על target=null) • Silent data-loss בשמירה • Mobile inputs <16px → iOS auto-zoom • `100vh` חוסם תוכן מתחת ל-URL bar • אין SRI על Firebase/Leaflet |
| P1 (משמעותי) | 29 | אין CSP/Referrer-Policy • `escapeHtml` לא מקודד `"`/`'` (XSS דרך attributes) • email-key collisions • race conditions ב-RTDB writes • zero `aria-label` ב-9715 שורות • zero `prefers-reduced-motion` • OSRM/Overpass מדליפים waypoints/AOI • `--cyan-dim` < 3:1 contrast |
| P2 (חשוב) | 22 | יציאה לא מנקה sessionStorage • `reportId` collisions • Google Maps tiles ToS-violation • Z-index war (8000-100000) • iPhone tap targets <44px • `Promise.all` נופל אם node אחד איטי |
| P3 (Polish) | 10 | swallowed errors • dead code (`arrestRestoreSession`) • localStorage לא מוצפן • mismatch "500KB" |

---

## ממצאי P0 — Critical (לטפל קודם)

| # | ממד | file:line | בעיה | תיקון | מאמץ |
|---|---|---|---|---|---|
| 1 | Security | [index.html:9474](../../../index.html#L9474) | **Fail-open**: כשה-apiKey לא מוגדר, `isApproved=true` נותן גישה מלאה למבצעים. רוטציית מפתח שגויה = פתיחה אילמת. | Fail closed: `isApproved=false` + banner. | S |
| 2 | Security | [index.html:9657,9504,4810,8850](../../../index.html#L9657) | כל ה-Firebase RTDB writes הולכים REST ללא `?auth=<idToken>`. אם ה-Rules לא הדוקות → כל אנונימי משכתב `user-plans`, `auto-news`, profiles. | (a) `?auth=` בכל PUT/DELETE (b) ודא ב-Firebase Console שכל הענפים סגורים לאנונימי. | M |
| 3 | Security | [index.html:9500](../../../index.html#L9500) | `_checkApproval` קורא `approved-users/<key>` ללא token — רשימת קצינים מאושרים enumerable פומבית. | Rule: `.read: "auth!=null && auth.token.email == <key>"`. | M |
| 4 | Security | [index.html:5582,9579](../../../index.html#L5582) | Operations gate **קוסמטי** — `switchView('operations')` ניתן להפעלה ישירה מ-DevTools. | מודלינג איום: לתעד שזה UX hint בלבד; אכיפה אמיתית ב-Rules בלבד. | M |
| 5 | Security | [index.html:5303](../../../index.html#L5303) | Mapillary client token (`MLY\|26636963...`) קומיט פומבי ב-GitHub. | רוטציה מיידית + injection ב-runtime. | S |
| 6 | Security | [index.html:7004](../../../index.html#L7004) | כל arrest plan נשלח בקלר ל-`router.project-osrm.org` (demo public). כל waypoint נכנס ל-access log חיצוני. | Self-host OSRM או proxy דרך Cloudflare Worker. | L |
| 7 | Security | [index.html:5089,5117](../../../index.html#L5089) | Overpass mới נשלח לכל page load עם bbox הגזרה → signaling חיצוני. | Cache לוקלי קיים (fence-inline.js) — בטל את ה-fetch ה-fallback. | M |
| 8 | Security | [index.html:1-15](../../../index.html#L1) | אין CSP, אין SRI על Firebase + Leaflet (3 sources). CDN compromise = כל הסשנים. | הוסף `<meta http-equiv="Content-Security-Policy">` + `integrity=` ל-script tags. | M |
| 9 | Bug | [index.html:7290,7107](../../../index.html#L7290) | `savedAt` נכתב, `p.ts` נקרא → **כל saved-plan card מציג "Invalid Date"**. | אחד מהשניים: `savedAt: Date.now()` בכתיבה או `p.savedAt` בקריאה. | S |
| 10 | Bug | [index.html:7142,7111](../../../index.html#L7142) | `arrestLoadPlan` ו-renderer לא מגנים על `target=null` — תוכנית observation/checkpoint שנשמרה בלי target → TypeError, list crashes. | `if (!plan.target) fallback` או אכוף non-null בשמירה. | S |
| 11 | Bug | [index.html:9668](../../../index.html#L9668) | Read-modify-write race ב-`UserPlans.add`: שתי לחיצות → save השנייה דורסת את הראשונה. | `push()` במקום `PUT` של מערך מלא, או טרנזקציה. | M |
| 12 | Bug | [index.html:7325](../../../index.html#L7325) | `UserPlans.add(payload).catch(...)` לא ב-await; שורה אחרי `arrestClear(true)`. כשל רשת = toast "נשמר" + איבוד מידע. | `await` בתוך try/catch; toast "נשמר מקומית בלבד" בכשל. | S |
| 13 | Bug | [index.html:5758](../../../index.html#L5758) | `submitFieldReportInline` מנקה inputs בלי await — כשל רשת = משתמש מקליד מחדש מאפס. | `await` + ניקוי רק בהצלחה. | S |
| 14 | UI/UX | [index.html:1](../../../index.html#L1) | **0 `aria-label`/`role`/`tabindex` ב-9715 שורות.** Icon-only buttons לא נגישים. | הוסף aria-label בעיקר ל: sidebar toggle, close, מפה, planner, 8 export buttons. | M |
| 15 | UI/UX + Mobile | [index.html:888,1297,1547,2436,2474,4167](../../../index.html#L888) | כל ה-inputs ב-11-12px → **iOS auto-zoom שובר את ה-layout באופן קבוע**. | `@media (max-width:768px){input,textarea,select{font-size:16px!important}}`. | S |
| 16 | Mobile | [index.html:61,185,194,3614,3680](../../../index.html#L61) | 9 שימושים ב-`100vh` → תחתית התוכן חבויה מאחורי URL bar ב-iOS. | החלף ב-`100dvh` (עם fallback). | S |
| 17 | Integrations | [index.html:11,13-14,4453](../../../index.html#L11) | אין SRI על Firebase + Leaflet (CDN). CDN poisoning = full takeover. | הוסף `integrity="sha384-..." crossorigin="anonymous"`. | S |

---

## ממצאי P1 — Significant (29) — Top 10 בולטים

| # | ממד | file:line | בעיה | תיקון |
|---|---|---|---|---|
| 1 | Security | index.html:8700 | `escapeHtml` לא מקודד `"` ו-`'` → XSS דרך attributes (item.source, photo, videoUrl). | קידוד מלא + protocol allowlist. |
| 2 | Security | index.html:8041,5441,5489 | Video modal + Mapillary modal interpolate `${url}` ישירות ל-`href`/`src`. | `setAttribute` + allowlist. |
| 3 | Security | index.html:9513 | email-key collision: `a.b@x.com` ↔ `a_b@x.com` חוזרים על אותו key → impersonation וplans-namespace שיתופי. | Firebase Auth UID במקום email-key. |
| 4 | Security | index.html:9543 | `signOut()` לא מנקה `arrestState`/sessionStorage — מחשב משותף חושף תוכנית קודמת. | Clear state + force redirect. |
| 5 | UI/UX | index.html:78-507 | 29 animations בו-זמנית, אין `prefers-reduced-motion`. | בלוק אחד @media reduce → `animation-duration: 0.01ms`. |
| 6 | UI/UX | index.html:611,1093,2135 | `--cyan-dim #00d4ff44` (27% opacity) = 2.1:1 contrast — נכשל WCAG AA 3:1 לאלמנטים פונקציונליים. | החלף ל-`--cyan` מלא בכל state-conveying borders. |
| 7 | UI/UX | index.html:5717 | Mobile drawer ללא Escape/focus-trap/`aria-expanded`/scroll-lock. | `document.body.style.overflow='hidden'` + ESC + focus initial item. |
| 8 | Bug | index.html:8723 | `loadPublisherProfiles` עלול ל-prototype-pollute דרך RTDB (anonymous writable). | `Object.create(null)` + strip `__proto__`. |
| 9 | Bug | index.html:7693 | `Promise.all` של 7 fetch — node אחד איטי = dashboard crash. | `Promise.allSettled` או `.catch` בכל fetch. |
| 10 | Integrations | index.html:7004,5089,7433 | OSRM + Overpass + Google Maps internal endpoint = 3 ערוצים שמדליפים AOI/route. | Self-host או Mapbox keyed. |

> רשימה מלאה ב-[03-findings/security.md](03-findings/security.md), [bugs.md](03-findings/bugs.md), [ui-ux.md](03-findings/ui-ux.md), [mobile.md](03-findings/mobile.md), [integrations.md](03-findings/integrations.md).

---

## ממצאים לפי ממד

- [Security](03-findings/security.md) — **24 ממצאים** (8 P0)
- [Bugs](03-findings/bugs.md) — **27 ממצאים** (5 P0)
- [UI/UX](03-findings/ui-ux.md) — **17 ממצאים** (2 P0)
- [Mobile](03-findings/mobile.md) — **10 ממצאים** (2 P0)
- [Integrations](03-findings/integrations.md) — **17 שירותים אקטיביים, 10 בעיות, 9 חדשים מומלצים**
- [Features](03-findings/features.md) — **10 פיצ'רים top + 40+ רעיונות לפי קטגוריה**
- [Future Roadmap](03-findings/future.md) — **3 אופקים** (2 שבועות / 2-3 חודשים / 6-12 חודשים)

---

## סדר ביצוע מומלץ (Top 10 לשבועיים הקרובים)

1. **[Sec P0 #1] Fail-closed auth** — שורה אחת ב-`Auth._checkApproval`. ~10 דק׳.
2. **[Bug P0 #9] תיקון "Invalid Date" ב-saved plans** — בחר `savedAt` או `p.ts`. ~5 דק׳.
3. **[Bug P0 #10] guard על `plan.target=null`** — `arrestLoadPlan` + renderer. ~15 דק׳.
4. **[Bug P0 #12] await על UserPlans.add** + toast מבדל מקומי/ענן. ~20 דק׳.
5. **[Mobile P0] `100vh` → `100dvh`** ב-9 מקומות + `input{font-size:16px}` ב-@media. ~10 דק׳.
6. **[UI/UX P1] `prefers-reduced-motion`** — 4 שורות CSS. ~3 דק׳.
7. **[Integrations P0] SRI על Firebase + Leaflet** — 4 hashes, sub-resource integrity. ~20 דק׳.
8. **[Sec P0 #5] רוטציה של Mapillary token** + injection runtime. ~30 דק׳.
9. **[Sec P0 #2] `?auth=<idToken>` בכל REST writes** — ~30 חזרות בקובץ אחד. ~45 דק׳.
10. **[Sec P0 #8] CSP meta tag** + `integrity=` על external scripts. ~20 דק׳.

**סה"כ שבועיים = ~4 שעות עבודה ממוקדת** מסירות 9 מתוך 17 ממצאי P0.

---

## פיצ'רים top שווי-עלות גבוה

מתוך [features.md](03-findings/features.md) — 10 הראשונים, ממוינים לפי impact/effort:

1. **Plan templates** (S, low) — חיסכון 5-10 דק׳ לתוכנית חוזרת.
2. **Briefing/Print A4 mode** (S, low) — תדריך נקי במקום כל הדשבורד.
3. **ROE sensitive-area markers** (S-M, low) — מסגדים/בתי״ס/בתי״ח buffer 75m.
4. **Cmd+K command palette** (S, low) — חיסכון 5-10 שניות לפעולה בחדר מבצעים.
5. **Offline-first PWA + cached tiles** (M, low) — שימוש ברכב סיור בלי קליטה.
6. **Real-time multi-user sync דרך RTDB** (M, med) — שני קצינים על אותה תוכנית.
7. **GPS-tagged field photos** (M, low) — תיעוד שטח עם מיקום.
8. **Route-frequency heatmap** (S-M, low) — אנטי-routine, מזהה דפוסים.
9. **Civilian-friction feed** (M, med) — סקרייפר ערוצי שבאב לפי כפר.
10. **Weather + light-cycle widget** (S, low) — זריחה/שקיעה/ירח/רוח לתזמון לילה.

---

## ממצאים חיוביים (מה עובד טוב)

- **אין `eval`, אין `Function()`, אין `document.write`** — baseline נקי.
- `escapeHtml` נקרא עקבית על שדות מ-RTDB ברנדרים העיקריים (השדות הבעייתיים הם בעיקר ב-`onclick` attributes ו-URL fields).
- `AbortSignal.timeout` בכל fetch חיצוני — מונע hung tabs.
- `popup-closed-by-user` / `popup-blocked` טיפול מבודל ב-Firebase Auth.
- Recent commit **`2222d8b` (low-contrast fix)** ניכר — text vars עכשיו עומדים AAA.
- **`tactical-modal`** מחליף `confirm`/`alert` נטיב — properly Esc/Enter bindable.
- Stepper UI ל-5-step wizard ויזואלית חד-משמעי (`step` / `step.active` / `step.done`).
- Map reparenting בין dashboard ↔ operations view — chip optimization חכם.
- 50-plan cap ב-`UserPlans.add` + 20-plan ב-sessionStorage — bounded growth.
- Mobile drawer ב-RTL נכון (sidebar slides from visual-right, `order: -1` על toggle).
- מובייל media queries יסודיים (3119-3327 + 3713-3738).
- Two-pass route trim past target ב-OSRM snap — robust handling.
- Firebase popup auth (commit `67507cc`) — בחירה נכונה ל-GitHub Pages.

---

## הקשר וקבצים נלווים

- צילומי מסך: **לא בוצעו בסבב זה** (אין dev server נדרש; פתיחה ישירה של HTML; דחיתי לסבב #2).
- פלט גולמי: [99-raw-agent-outputs/](99-raw-agent-outputs/) (4 סוכני משנה שרצו במקביל).
- baseline להשוואה בסבב הבא: **אין** — זה סבב #1.
- Commits עיקריים בחודש האחרון שנכללו ב-context: `841710f` (corner-closing), `8c7141b` (5-step wizard), `2222d8b` (low-contrast), `67507cc` (popup auth).

---

## הערה לסבב הבא

כשתפעיל מחדש (`mega-audit --baseline docs/audits/2026-05-11-mega-audit/00-index.md`), הסקיל ישווה אילו מ-17 ה-P0 תוקנו, אילו עדיין קיימים, ומה חדש. עד אז — שווה לפתוח את 10 ה-Top במסלול שעות. Auth + saved-plans crash + Mobile inputs הם הניצחונות המהירים ביותר.
