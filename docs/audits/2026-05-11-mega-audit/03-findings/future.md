# Future Roadmap — 2026-05-11

> מפת דרכים בשלושה אופקים. כל פריט מקושר לפיצ'ר מ-`features.md` (לדוגמה `[#3]`). המטרה: לסיים פיילוט אישי איכותי תוך 6-12 חודשים שיכול לעלות ליחידה רחבה יותר.

---

## Horizon 1: 2 weeks (ship now, S effort)

מוקד: לסיים חוב טכני ולתת ערך מיידי ללא שינויי ארכיטקטורה.

1. **סיום `COORDS-TODO`** — לבקש מהמשתמש את הקובץ הממולא, להריץ replace חד-פעמי על כל המערכים (`KEY_VILLAGES`, `P1_LOCATIONS`, `P2_LOCATIONS`, `ISRAELI_LOCATIONS`). נקודות מודיעין/תכנון לא יעוגנו על קואורדינטות שגויות עוד.
2. **Briefing print mode** `[#2]` — `@media print` ייעודי שמסתיר sidebar/legend/header, מציג מפה גדולה + פירוט שלבים בעמוד 2. שינוי CSS בלבד, אפס סיכון.
3. **Cmd/Ctrl+K command palette** `[#4]` — modal פשוט עם רשימת actions (`switchView`, `toggleArrestPlanner`, `setActivityType`, חיפוש ישוב). ~100 שורות.
4. **Plan templates v1** `[#1]` — שדה `isTemplate:true` ב-arrest-plan, כפתור "שמור כתבנית" ו"חדש מתבנית". שימוש מחדש של ה-schema הקיים.
5. **Weather + sunrise/sunset widget** `[#10]` — צ׳יפ קטן בכותרת view-operations: `🌅 06:42 | 🌇 19:18 | 🌙 73% | 💨 12km/h NW`. OpenWeatherMap + SunCalc.
6. **תיקון UX קטנים** — feedback בעת כשל OSRM, אישור מחיקה לתוכנית שמורה (היום מוחק ישירות?), Esc לסגירת `arrestSavedList` הפתוח.
7. **דוקומנטציה פנימית** — `CLAUDE.md` קצר בפרויקט (מה זה §5ד, מה הפלואו של תכנון, מה המוסכמות לקואורדינטות) — שיהיה context ל-LLM הבא ולקצין הבא שנכנס.

**Ship target:** סוף השבוע של 25.05.

---

## Horizon 2: 2-3 months (M-L effort, feature work)

מוקד: הפיכת הדשבורד מ"כלי תכנון אישי" ל"כלי עבודה משותף לכוח קטן" + שדרוג איכות נתונים.

### Tranche A — Field-ready (חודש 1)
1. **PWA + offline tiles** `[#5]` — manifest, service-worker, אריחי Leaflet ב-IndexedDB. install-to-home-screen באנדרואיד/iOS. בלי PWA אין mobile/field רציני.
2. **GPS-tagged photos** `[#7]` — `<input capture>` + Firebase Storage + נקודות על המפה.
3. **Sensitive-area markers / ROE buffers** `[#3]` — שאיבה חד-פעמית מ-Overpass-Turbo: `amenity=hospital|school|place_of_worship|kindergarten`, `landuse=cemetery`. שכבת Leaflet, buffer 75m, אזהרה כשציר עובר.
4. **Big-button field mode** — מצב מובייל מצומצם עם 3 כפתורים: "התחלתי שלב", "סיימתי שלב", "מגע".

### Tranche B — Collaboration (חודש 2)
5. **Multi-user live sync v1** `[#6]` — Firebase Realtime presence + sync לתוכנית הפעילה. שני קצינים על אותה תוכנית רואים שינויים בזמן אמת. אין עדיין handoff/comments — רק sync.
6. **Role-based views (4 תפקידים)** — `commander/officer/dispatcher/field`. UI hiding **+** Firebase Rules התואמות.
7. **Approval workflow** — סטטוסי תוכנית, `commander` חייב לאשר לפני "active".
8. **Audit log חתום** — append-only `audit/{planId}/{ts}` עם hash-chain. דרישה משפטית, לא להמתין.

### Tranche C — Intelligence (חודש 3)
9. **Route-frequency heatmap** `[#8]` — חישוב client-side מהיומן (כל ה-plans), grid 50m, color ramp. אזהרה לתוכנית חדשה ב-cell חם.
10. **Wanted/HVT list (אם מאושר רגולטורית)** `[#Intelligence]` — שולחן RTDB מאובטח, "צור תוכנית מעצר ליעד".

**Ship target:** ~31.07. הפעלה מוגבלת ביחידה אחת.

---

## Horizon 3: 6-12 months (architecture & scale)

מוקד: עזיבת ה-monolithic HTML, הכנה ל-scale-out אם הפיילוט מצליח.

### Decision points (חייבים החלטה לפני כן)
- **single-file HTML → component framework?** `index.html` ב-~9,700 שורות מתחיל לשבור גרירה (חיפוש איטי, conflicts בעריכה, אין hot reload). מועמדים:
  - **Vite + vanilla JS modules** — minimal migration, שומר על "no build" באתר הסטטי. **מומלץ.**
  - Vite + Svelte/Solid — אם רוצים reactive proper.
  - לא React — overhead מיותר לפרויקט הזה.
- **Backend?** היום RTDB + auth client-side בלבד. אם מוסיפים Wanted list, audit log חתום, או scraping — צריך:
  - **Cloudflare Workers** (free tier, edge, פשוט) — מומלץ ל-scrapers ולשרת audit-log signing.
  - לא Node/Express שרת — overhead תפעולי.
- **Database growth** — RTDB נוח אבל לא טוב לחיפוש/אגרגציה. אם plans>10k או רוצים analytics → migrate ל-**Firestore** (אותו אקוסיסטם, אותו auth, queries הרבה יותר טובים). לעשות זאת לפני שיש יותר מ-100 משתמשים, אחר כך כואב.
- **Hosting** — GitHub Pages מספיק לפיילוט אבל אסור לדלוף לזה דבר מסווג. אם המערכת הופכת לחצי-מסווגת → לעבור ל-hosting פנימי (S3 פרטי / Azure Israel / on-prem).

### Tranche D — Architecture (חודשים 4-6)
11. **Split של index.html ל-modules** — Vite + ES modules. מודולים: `map.js`, `arrest-planner.js`, `auth.js`, `data-tabs.js`, `history.js`. אותו פלט HTML סטטי, build step מוסיף bundling.
12. **Backend ראשון: Cloudflare Worker** — endpoint יחיד `/api/audit-sign` שעוטף שינוי בתוכנית עם hash-chain חתום (HMAC-SHA256 עם secret ב-Worker env).
13. **Civilian-friction feed** `[#9]` — Worker שני שסורק טלגרם פעם ב-15 דק׳, שומר ל-KV, חושף `/api/friction/{village}`.
14. **Migration ל-Firestore** — אם כמות התוכניות גדלה, או אם נדרשים queries מורכבים (e.g., "כל התוכניות שעברו בכפר X ב-90 יום").

### Tranche E — Hardening (חודשים 7-12)
15. **End-to-end encryption לתוכניות רגישות** — תוכנית נשמרת מוצפנת בצד הלקוח עם מפתח לפי `userKey`, ה-backend לא רואה plaintext. רלוונטי במיוחד ל-HVT list.
16. **Penetration test** ע"י צד שלישי — לפני שזה הופך לכלי עבודה רשמי, לא אחרי.
17. **Disaster recovery + backup policy** — RTDB snapshot יומי ל-bucket נפרד, retention 90 יום.
18. **Localization מלא** — RTL מובנה כבר, אבל הפרדה של מחרוזות מהקוד. תומך עברית/ערבית/אנגלית.
19. **Tutorial / onboarding** — guided tour לחיילים חדשים, וידאו קצר על §5ד, FAQ.

---

## Architecture decisions to make soon (במיוחד לפני Horizon 3)

| החלטה | ברירות | המלצה ראשונית | דחיפות |
|---|---|---|---|
| Single-file HTML או build pipeline? | (a) להישאר monolithic (b) Vite + ES modules (c) Vite + framework | **(b) Vite + ES modules** — שומר על פריסה סטטית, מאפשר tree-shaking | חודש 3-4 |
| Database — RTDB עד מתי? | RTDB / Firestore / Postgres + backend | **Firestore** ברגע שיש >10 משתמשים פעילים או queries מורכבים | חודש 3 |
| Backend או רק client+BaaS? | (a) רק Firebase (b) Cloudflare Workers (c) שרת Node | **(b) Workers** לפיצ'רים שדורשים סודיות (scrapers, signing) | חודש 4 |
| Hosting — GitHub Pages עד מתי? | Pages / Azure Static / on-prem | Pages לפיילוט, **on-prem ברגע שיש מידע מבצעי-של-אמת** | תלוי תוכן |
| Auth — Gmail בלבד? | Gmail / SSO ארגוני / 2FA חובה | להוסיף **2FA חובה** לכל role≠`field` ברגע שיש tier production | חודש 2-3 |
| Mobile — PWA או native? | PWA / React Native / Flutter | **PWA** — אחזקה אחת, הספיק לצרכים. native רק אם דרושים PTT/בלוטות׳ ייעודיים | החלטה: PWA |
| Map — Leaflet או MapLibre/Mapbox GL? | Leaflet / MapLibre GL / Mapbox | **להישאר עם Leaflet** — heatmap+PWA יחזיקו. WebGL אם וכאשר נדרש 3D | החלטה: Leaflet |

---

## Open questions for the user (בקשת קלט)

1. **שותפים פוטנציאליים** — מי עוד אמור להשתמש בדשבורד הזה ב-3 חודשים? מפקד גדוד אחד? פלוגת מילואים? יחידה רחבה? זה משנה דרמטית את העדיפויות (multi-user, roles, audit).
2. **רגולציה** — האם יש סיווג? אם כן, GitHub Pages פתוח **לא יכול** להחזיק. מתי אנחנו חוצים את הקו?
3. **Wanted/HVT list** — האם זה רעיון שיוצא מהדמיון שלי, או צורך מבצעי אמיתי? אם מבצעי — דורש דיון רגולטורי לפני פיתוח.
4. **תקציב / זמן** — האם זה פרויקט צד שמתחזק לבד, או יש tasking רשמי עם זמן ייעודי?
5. **טלגרם / ערוצים** — האם כבר יש איסוף קיים שאפשר לדגום (Open-source intel, מערכת אחרת), או שה-Worker יסרוק מאפס?
6. **תרגול / משחק** — האם המערכת תשמש גם לתרגולי שולחן (TTX) או רק לתכנון אמיתי? אם גם תרגול — צריך mode "תרגיל" שלא מתערבב ב-audit log.
7. **חיבור ל-`battalion-scheduler`** — שני הפרויקטים בעולם דומה. האם אנחנו רוצים auth/users משותפים? cross-link של תוכניות לכוננויות?
8. **קואורדינטות COORDS-TODO** — האם המשתמש מתכוון להחזיר את הקובץ הממולא, או שאני אעשה את העבודה אוטומטית מ-OSM Overpass?

---

## TL;DR מסלול מומלץ

> שבועיים: סגור חוב טכני (COORDS, print mode, Cmd+K, templates).
> חודש 1: PWA + GPS photos + ROE markers — Field-ready.
> חודש 2: Multi-user + roles + audit log חתום — Team-ready.
> חודש 3: Heatmap + intelligence feeds — Analytical depth.
> חודש 4-6: Migration ל-Vite + Workers — Architecture clean.
> חודש 7-12: E2E encryption + pentest + DR — Production-grade.

המנוף הכי גבוה כרגע: **#2 (Briefing print) + #5 (PWA) + #6 (Multi-user)**. שלושת אלה הופכים את הכלי מ"prototype אישי יפה" ל"כלי עבודה אמיתי".
