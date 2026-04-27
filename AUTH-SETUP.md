# הגדרת התחברות עם Gmail — תכנון מבצעי

## שלב 1 — Firebase Console (חד־פעמי, 3 דקות)

1. היכנס ל-https://console.firebase.google.com → בחר את הפרויקט `project-8874611670473210078`
2. **הפעלת Google Sign-In:**
   - Authentication → Sign-in method → Google → **Enable**
   - הגדר Project support email → שמור
3. **קבלת Web API Key:**
   - Project Settings (⚙️) → General → Your apps → Web app (אם לא קיים: צור חדש)
   - העתק מהסעיף `firebaseConfig`:
     - `apiKey`
     - `authDomain` (בד״כ `<project-id>.firebaseapp.com`)
4. **הגדרת Authorized domains:**
   - Authentication → Settings → Authorized domains → הוסף את הדומיין שהדשבורד רץ ממנו
   - אם רץ מקומית: `localhost` כבר מורשה

## שלב 2 — הטמעה ב-index.html

ערוך את `index.html` ומצא את השורות (סביב שורה 8632):

```js
const FIREBASE_AUTH_CONFIG = {
    apiKey: window.__FIREBASE_API_KEY__ || 'REPLACE_WITH_WEB_API_KEY',
    authDomain: window.__FIREBASE_AUTH_DOMAIN__ || 'project-8874611670473210078.firebaseapp.com',
    projectId: window.__FIREBASE_PROJECT_ID__ || 'project-8874611670473210078',
    ...
};
```

החלף `REPLACE_WITH_WEB_API_KEY` במפתח מ-Firebase Console.

> **אבטחה:** ה-Web API Key הוא ציבורי (בסדר לחשוף), אבל האכיפה האמיתית היא דרך Firebase Rules. וודא שהפעלת את `npm run rules`.

## שלב 3 — אישור משתמשים

מתוך תיקיית `mabat-443/mabat-443/`:

```bash
# אישור משתמש חדש
npm run approve approve elchaifinn@gmail.com

# רשימת מאושרים
npm run approve list

# ביטול אישור
npm run approve revoke some@email.com
```

## שלב 4 — עדכון Firebase Rules

```bash
npm run rules
```

מפעיל את הכללים החדשים: `approved-users` ציבורי, `user-plans/{email}` נגיש **רק** לבעל האימייל.

## איך זה עובד

1. משתמש נכנס לדשבורד → רואה כפתור "התחבר עם Google" בתחתית הסרגל
2. מתחבר → המערכת בודקת אם האימייל ב-`approved-users/`
3. **אם מאושר:** הטאב "תכנון מבצעי" נגלה, תוכניות נשמרות ב-Firebase תחת `user-plans/<email-key>/arrest-plans/`
4. **אם לא מאושר:** הטאב מוסתר, ניסיון גישה חוזר לדשבורד הראשי

## גיבוי תוכניות קיימות

תוכניות שכבר שמורות ב-localStorage נשארות זמינות כ-fallback. בכניסה ראשונה עם Gmail, ה-Firebase של המשתמש יהיה ריק — אפשר לייצא ידנית מה-DevTools:

```js
// בקונסול הדפדפן
copy(localStorage.getItem('mabat443-arrest-plans'));
```

ואז לייבא:

```js
await fetch(`${FIREBASE_URL}/user-plans/${Auth.userKey()}/arrest-plans.json`, {
    method: 'PUT',
    body: localStorage.getItem('mabat443-arrest-plans')
});
```

## מצב Fail-Open זמני

עד שמגדירים את ה-`apiKey`, המערכת פועלת במצב "גישה פתוחה" (כמו קודם) כדי לא לחסום עבודה. ברגע שהמפתח מוגדר — הגישה לתכנון מבצעי נסגרת למאושרים בלבד.
