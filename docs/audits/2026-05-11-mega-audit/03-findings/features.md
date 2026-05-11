# Feature Recommendations — 2026-05-11

> **סקירה:** דשבורד מבט 443 הוא כיום קובץ HTML יחיד (~9,700 שורות), Firebase Auth + RTDB, Leaflet עם GeoJSON של גדר/חסימות, ארבעה סוגי פעילות (מעצר/מחסום/תו״ס/מארב), אשף 5 שלבים לתכנון מעצר עם שיטת סגירת פינות (§5ד), ארכיון תוכניות, יצוא Gmaps/WhatsApp/JSON/PDF.
> ההמלצות מתמקדות בערך מבצעי אמיתי למפקד גדוד / קצין מבצעים, לא בקישוטים.

---

## Top 10 (highest impact / lowest effort first)

| # | Feature | Why it helps | Effort | Risk |
|---|---|---|---|---|
| 1 | **Plan templates / תבניות תוכנית** — שמירת תוכנית כ-template ("פשיטה לילית בית ליקיא", "מחסום נע 443") עם משתנים (תאריך, כוח, יעד) | מבצעים חוזרים מתוכננים מאפס בכל פעם — חיסכון 5-10 דק׳ לתוכנית | S | low |
| 2 | **Briefing/Print mode ייעודי** — תבנית A4: עמ׳ 1 מפה+ציר, עמ׳ 2 לוח״ז שלבים, עמ׳ 3 כוח+תדרים | היום `window.print()` מדפיס את כל הדשבורד. מפקד צריך תדריך נקי. | S | low |
| 3 | **Sensitive-area markers (ROE)** — שכבת מסגדים/בתי״ס/בתי״ח/בתי קברות עם buffer 50-100m + אזהרה כשציר עובר | קריטי לכללי פתיחה באש ולמניעת תקרית הסברתית. נתוני OSM זמינים (Overpass) | S-M | low |
| 4 | **Hotkeys + Cmd/Ctrl+K command palette** — מעבר תצוגות, התחל תוכנית, חיפוש ישוב, פתח יומן | חיסכון של 5-10 שניות לכל פעולה. בחדר מבצעים זה משמעותי | S | low |
| 5 | **Offline-first PWA + cache טיילי מפה** — `service-worker` + cache אריחי Leaflet zoom 13-17 לגזרה (~50-80MB) | רכב סיור באזורי גזרה ללא קליטה. היום הדף לא נטען בלי אינטרנט | M | low |
| 6 | **Real-time multi-user sync** דרך RTDB — תוכנית פעילה משותפת בין דיספאצ׳ר/מפקד/שטח (cursor חי + סמן על המפה) | היום שני קצינים פותחים אותה תוכנית → דריסה. ה-RTDB כבר שם | M | med |
| 7 | **GPS-tagged photos + field notes** במצב מובייל — `<input type=file capture=environment>` + `navigator.geolocation` → Firebase Storage | תיעוד שטח, מעקב שינויים (גרפיטי, מחסומים אזרחיים חדשים), חומר לתחקיר | M | low |
| 8 | **Route-frequency heatmap** על המפה מהיומן | מזהה דפוסים: "פשטנו 7× החודש על אותו ציר מצפון לבית ליקיא" → סיכון לחיזוי. קלט לתכנון אנטי-routine | S-M | low |
| 9 | **Civilian-friction feed** — סקרייפר טלגרם/X של ערוצי שבאב לפי כפר (כבר יש לך `history.threats` ב-`KEY_VILLAGES`) | התראה מוקדמת על "יום זעם", הלוויית שהיד, הפגנה — שיכולים לשבש מבצע | M | med |
| 10 | **Weather + light-cycle widget** — זריחה/שקיעה, פאזת ירח, רוח, גשם — לכל תוכנית | משפיע על תזמון מעצר לילי, הפעלת רחפן, שדה ראייה, חיכוך תרמלי | S | low |

---

## By category

### Operational
- **#1 תבניות תוכנית** — duplicate-from-archive עם החלפת תאריך/כוח/יעד. הוסף שדה `templateName` ו-toggle "שמור כתבנית".
- **#2 Briefing print** — דפי A4 נקיים בלי sidebar/legend/header. שימוש ב-`@media print` כבר חלקי, צריך view מסודר.
- **Mission timer + live checklist** — בזמן ביצוע: לחץ "התחל" → "20:47 — שלב 3 (סגירת פינה צפון-מערב) — 7 דק׳ ב-phase". מאפשר תיעוד time-on-target אמיתי במקום משוער.
- **Casualty / contact-on-target form** — כפתור אדום קטן "מגע!" שפותח טופס מהיר: נק׳ מגע, סוג איום, פתחנו אש?, פציעות. נשמר לארכיון לתחקיר.
- **Quick-add חסימה זמנית** — לחיצה ארוכה על המפה → "צור חסימה זמנית" שמופיעה לכולם בזמן אמת (לתאם פעולה רב-כוחית).

### Intelligence & Data
- **#3 Sensitive-area markers** — Overpass-Turbo חד-פעמי לכל ה-amenities של הגזרה, cache ב-localStorage. שכבה נדלקת/כבית.
- **#9 Civilian-friction feed** — Cloudflare Worker קל שסורק ערוצי טלגרם של "מוקדים", מסמן anomaly (קפיצה 3× ב-keyword frequency של "אלאקצא"/"שהיד"/"חמאס").
- **#10 Weather + light** — OpenWeatherMap free + `SunCalc.js`. נקודה אחת (SECTOR_CENTER) מספיקה לרזולוציה הזו.
- **Wanted/HVT list** — טבלה פנימית ב-RTDB של יעדי מעצר בגזרה: שם, כפר, צילום, תיק. כפתור "צור תוכנית מעצר ליעד זה" → ממלא target+ישוב. **רגיש — Firebase Rules הדוקות + encryption client-side.**
- **שכבת תקריות 30 יום אחרונים** מ-`history` — נקודות על המפה עם opacity לפי זמן (חדש=מלא, ישן=שקוף).
- **Israeli-side situation awareness** — לא רק "מה הם עושים" אלא גם תנועת אזרחים ישראלים (הפקקים על 443? אירוע ביטחוני סמוך?). מועיל לדיספאצ׳ר.

### Collaboration
- **#6 Multi-user live sync** — avatars בפינה (מי משתמש), על איזו תוכנית הם עובדים, סמן עכבר חי על המפה (פסטלי).
- **Plan handoff** — "העבר תוכנית ל..." → user picker → התוכנית מופיעה אצל הקצין השני עם notification badge.
- **Comments per plan** — chat thread קצר (לא chat מלא — רק הערות עוקבות, רס"ן/סמ"פ).
- **Role-based views**:
  - `commander` — רואה הכל, מאשר תוכניות.
  - `officer` — בונה תוכניות, רואה רק שלו + שותפו.
  - `dispatcher` — read-only על המפה החיה + יומן.
  - `field` — מובייל בלבד, רואה את התוכנית הפעילה + מעלה דיווחים.
- **Approval workflow** — `draft → pending → approved → active → completed/aborted`. הפעלת תוכנית דורשת אישור `commander`.

### Mobile/Field
- **#5 Offline PWA** — install-to-home-screen, manifest, service worker, cache אריחי מפה לכל הגזרה.
- **#7 GPS photos** — צילום מהפיתוח עם מיקום, נקודה על המפה עם thumbnail.
- **Live position on shared map** — קצין שטח עם GPS opt-in מעדכן מיקומו לתוכנית הפעילה. **רגיש לפרטיות — opt-in מפורש בלבד, נמחק אחרי 24h.**
- **Big-button field mode** — UI מצומצם עם 3 כפתורים ענקיים: "התחלתי שלב", "סיימתי שלב", "מגע!". מתאים לכפפות/חשיכה.
- **Push-to-talk piggyback** — לא לבנות PTT, אבל כפתור "פתח Mosajef/Walkie" שמשגר deep link לאפליקציית התקשורת המאושרת.

### Analytics
- **#8 Route-frequency heatmap** — pixel-grid 50m על הגזרה, color ramp לפי כמה ציר עבר שם ב-90 יום. אזהרה אם תוכנית חדשה שוחזרת ל-heat cell חם.
- **Sector activity timeline** — ציר זמן אופקי עם dots לכל פעולה (מעצר/מחסום/תו״ס/מארב) ב-30 יום, צבע לפי סוג, hover מציג סיכום.
- **Response-time tracking** — מהדיווח בשטח ועד "תוכנית הופעלה" — מדידה אובייקטיבית של זמני תגובה. KPI אמיתי.
- **Per-village threat score** — אגרגציה של דיווחים+keywords+מבצעים → ציון 0-100 לכל כפר, מתעדכן יומית. תצוגה כ-heat overlay.
- **Plan outcome tagging** — אחרי תוכנית: success / partial / aborted / no-contact. לימוד מה עובד.

### QoL
- **#4 Cmd+K command palette** — `Esc` כבר תופס סגירת overlays; הוסף `Cmd/Ctrl+K`. רשימת actions נחפשת מהירה.
- **Recent plans drawer** — מהפינה — 5 תוכניות אחרונות, לחיצה פותחת.
- **Plan diff** — "מה שונה בין התוכנית הזו לקודמת?" — diff על שלבים+ציר. עוזר כש"מעדכנים תוכנית בעקבות שינוי" ולא בונים מאפס.
- **Hebrew/Arabic name toggle** — שמות הכפרים בעברית בלבד. toggle לערבית (بيت لقيا, بدّو) — שימושי לקצינים דוברי ערבית או מתאם פעולות אזרחי.
- **Inline OSRM error feedback** — היום אם OSRM נופל המשתמש לא מבין למה "התאם לכביש" לא עובד. הצג ✓/✗ + הודעה ספציפית.
- **Search-anywhere** — כפר/תוכנית/מילת מפתח/דיווח — תיבת חיפוש אחת.
- **Coordinate paste** — הדבק `31.870, 35.120` מ-OSM/Gmaps → קופץ למיקום + אופציה לסמן.
- **תיקון COORDS-TODO** — יש 40+ קואורדינטות ידועות-לא-מדויקות ברשימה. סקריפט one-shot שמקבל את הקובץ עם המספרים האמיתיים, מעדכן את כל המערכים בקובץ.

### Civil-affairs / Compliance
- **#3 ROE buffers** — מסגדים/בתי״ס/בתי״ח עם buffer ויזואלי + לוג שהמפקד אישר במודע חציה.
- **Curfew / שעת שגרה awareness** — אם הכפר תחת סגר/שגרת לימודים → indicator על המפה.
- **Audit log חתום** — כל שינוי בתוכנית פעילה נשמר עם user+timestamp+IP, hash-chain (פלט hash של רשומה N תלוי ב-N-1). **קריטי משפטית** לבג"ץ/מצ"ח/תחקיר.
- **Briefing-checklist ROE** — checkbox-list של אישורי כללי לחימה (מקלעים? קליעת ירי-לאחור? פגישה עם אזרחים?). מוצמד לכל תוכנית מעצר.
- **Data retention policy** — auto-purge של תוכניות מעל גיל X (ברירת מחדל 6 חודשים), אלא אם מוקפאות כראייה.

---

## Anti-features (do NOT build — explain why)

| לא לבנות | למה לא |
|---|---|
| **AI chat assistant** ("שאל את הדשבורד") | LLM-ים לא יודעים את הדוקטרינה הספציפית (§5ד), hallucinate על קואורדינטות, ופותחים דליפת מודיעין לספק חיצוני. אם בכל זאת — local model בלבד. |
| **3D / WebGL tactical view** | יפה ב-demo, חסר ערך בגזרה שטוחה יחסית. Leaflet 2D עם topo overlay מספיק. עלות פיתוח גבוהה, baggage מובייל כבד. |
| **Real-time drone feed integration** | חוצה רשתות (אזרחית ↔ צבאית) — אסור. אם צריך, מערכת נפרדת לחלוטין. |
| **Social-network OSINT אוטומטי על פרופילים אישיים** | משפטית רעיל (פרטיות), הסיכון בטעות גדול מהתועלת. מקסימום ערוצי טלגרם ציבוריים של "מוקדים" — לא של אנשים. |
| **Gamification / נקודות / leaderboards למפקדים** | הטרבי-טרבי הזה נוגד את אופי העבודה. ילד פוסל את הכלי. |
| **Auto-suggest "המעצר הבא"** מ-ML | dataset לא קיים בגודל הנכון, יוצר bias-loop (מעצרים → נתונים → המלצות → עוד באותו מקום). לא לפני framework בקרה אנושי מוסדר. |
| **תרגום אוטומטי של דיווחי שטח לאנגלית/ערבית** | סיכון שטעות תרגום תיצור אי-הבנה מבצעית. השאר ידני. |
| **שילוב WhatsApp/Telegram bot לשליחת תוכניות** | תוכניות מבצעיות במסרים אזרחיים = אסון אבטחת מידע. שיתוף רק בתוך המערכת המאומתת. ה-WhatsApp הקיים שולח **תקציר טקסט**, לא קובץ תוכנית מלא — זה גבול נכון. |
| **Wiki-style "Notes על כל כפר" עם עריכה חופשית** | מהר מאוד הופך לרכילות / סטראוטיפים / חוסר רגישות. אם בכל זאת — שדות מובנים בלבד. |
| **Slack/Discord/Teams integration** | פלטפורמות צרכניות, לא אישור ביטחוני. |

---

## הערות מימוש קצרות

- **המעבר ל-PWA (#5) הוא enabler** ל-#7 (GPS photos), ל-Big-button field mode, ול-offline בכלל. כדאי לעשות מוקדם.
- **Multi-user (#6) דורש Firebase Rules איכותיות** — לא לסמוך על UI hiding. כל role-based view צריך rule מקבילה ב-DB.
- **Audit log חתום הוא דרישה משפטית** ברגע שהדשבורד עולה ליחידה מעבר לפיילוט אישי. תעדף לפני scale-out.
- **כל פיצ'ר שנוגע ב-PII** (תצלומי שטח, מיקום משתמש, רשימת HVT) — דורש review של רס"ן מבצעי + סייבר לפני production.
- **Firebase RTDB ב-europe-west1 בסדר רגולטורית** לכרגע (GDPR-aligned region). אם המערכת מתרחבת — לשקול on-prem או Azure Israel Central.
