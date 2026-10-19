<div dir="rtl">

# פתרון aguda_forms

הפתרון כולל את הרכיבים הבאים:

| רכיב | סוג | תיאור | קובץ |
|---|---|---|---|
| **Activate StudentsV3** | זרימת ענן | מלווה משתמשים בהפעלת החשבון ובמעבר ל־MFA עם Authenticator (סעיפים 1–7) | [ActivateStudentsV3-….json](src/Workflows/ActivateStudentsV3-FCF4A396-A6CB-F111-A05A-6045BD99F2BC.json) |
| **AddUserToGroup** | זרימת Desktop | מוסיפה משתמש לקבוצת MFA ב־AD המקומי. נקראת מ־Activate StudentsV3 (סעיף 3.4) | [AddUserToGroup-….json](src/Workflows/AddUserToGroup-3853C697-378D-461C-BED5-D7773CDC8C0B.json) |
| **AgudaForm** | Canvas App | טופס אישור חברות באגודת הסטודנטים עם חתימה דיגיטלית (סעיף 8) | [src/CanvasApps/](src/CanvasApps/) |

---

# Activate StudentsV3 – אוטומציה להפעלת חשבונות

זרימת Power Automate שמלווה משתמשים במכללת סמינר הקיבוצים בהפעלת החשבון ובמעבר ל־MFA עם Microsoft Authenticator. היא מטפלת גם במשתמשים שהחשבון שלהם סומן כחשוד.

קובץ הזרימה: [src/Workflows/ActivateStudentsV3-FCF4A396-A6CB-F111-A05A-6045BD99F2BC.json](src/Workflows/ActivateStudentsV3-FCF4A396-A6CB-F111-A05A-6045BD99F2BC.json)

---

## 1. רשימת SharePoint

כל פריט חדש ברשימה מפעיל את הזרימה, וכל ההתקדמות נרשמת בה.

| | |
|---|---|
| אתר | `https://smkb365.sharepoint.com/sites/Forms` |
| מזהה הרשימה | `df0265a3-9454-413f-be10-7013bf73f65c` |
| הגדרות הרשימה | https://smkb365.sharepoint.com/sites/Forms/_layouts/15/listedit.aspx?List=%7Bdf0265a3-9454-413f-be10-7013bf73f65c%7D |

### עמודות שהזרימה משתמשת בהן

| עמודה | תפקיד |
|---|---|
| `Title` | שם מלא |
| `email` | מייל פרטי. לכאן נשלחים מיילי ההפעלה |
| `SeminarMail` | מייל מכללתי (UPN). לכאן נשלחים מיילי Authenticator |
| `Phone` | טלפון לשליחת SMS (בלי 0 בהתחלה, הזרימה מוסיפה אותו) |
| `Para` | **קובע איזה מסלול ירוץ** (ראו סעיף 3) |
| `Job` | `Student` / `Teacher` / `Employee` |
| `Log` | **השלב האחרון שהמשתמש הגיע אליו**. הזרימה כותבת לכאן |
| `CreatedFrom`, `Uniq` | ממולאים כשהזרימה יוצרת בעצמה פריט המשך |

---

## 2. טריגר ואתחול

1. **טריגר:** *When an item is created* ברשימה. הזרימה בודקת פריטים חדשים כל דקה.
2. **אתחול משתנים:** תבניות HTML של המיילים (עם לוגו) ונוסח ה־SMS למשתמש חשוד.
3. **Switch** לפי הערך של `Para`.

---

## 3. מסלולים

### 3.1 `riskyuser` – משתמש חשוד

החשבון נפרץ או שזוהתה בו פעילות חשודה. מטרת המסלול היא להודיע למשתמש שהסיסמה אופסה ושהכניסה עברה ל־MFA.

- אם `email` מסתיים ב־`smkb.ac.il`, נשלח **SMS** דרך Soprano למספר שב־`Phone`. ה־SMS כולל קישור לאיפוס סיסמה (`pass.smkb.ac.il`) ופרטי מוקד התמיכה.
- אחרת נשלח **מייל** "אימות פרטי משתמש" לכתובת שב־`email`.
- המסלול **לא כותב ל־`Log`**.

### 3.2 `newuser` – משתמש חדש

1. קריאת HTTP לזרימה חיצונית שמוסיפה את המשתמש ל־gategoing.
2. ההמשך מתפצל לפי `Job`:

**Student / Teacher**

| שלב | מייל למשתמש | כפתורים | אם לחץ "לשלב הבא" | אם ביקש עזרה |
|---|---|---|---|---|
| 1 | איפוס סיסמה | `לשלב הבא` / `לא מצליח/ה לאפס סיסמא` | `Log = Step1` וממשיכים לשלב 2 | מייל לתמיכה, `Log = Step1 with problem` |
| 2 | כניסה למייל Microsoft 365 | `לשלב הבא` / `לא מצליח/ה להיכנס לחשבון` | `Log = Step2` ומייל סיכום | מייל לתמיכה, `Log = Step2 with problem` |

**Employee**

אותו מבנה. המייל הראשון הוא "ברכות לקבלתך לעבודה", ומייל התמיכה בשלב 2 עוסק בטופס פרטים אישיים לעובד חדש. ראו באג 1 בסעיף 7.

3. **במקביל לשלבים:** הזרימה ממתינה **7 ימים**. אחר כך היא יוצרת פריט חדש ברשימה עם:
   - `Para = MSAuthenticator`
   - `CreatedFrom = Activate StudentsV3 new user case`
   - `Uniq = <SeminarMail>MSAuthenticator`
   - העתק של `Title`, `SeminarMail`, `email`, `Job`, `Phone`

   הפריט החדש מפעיל את הזרימה מחדש במסלול 3.4. הריצה הנוכחית מסתיימת ב־**Cancelled**, וזה צפוי.

### 3.3 `backuser`

המסלול ריק ולא מבצע כלום.

### 3.4 `MSAuthenticator` – מעבר מ־SMS ל־Authenticator

1. שליפת פרופיל המשתמש מ־Office 365.
2. מייל ל־`SeminarMail` בנושא "הודעה חשובה! ביטול שימוש בסמס ומעבר ל־Authenticator", בשני שלבים:

| שלב | כפתורים | הצלחה | בקשת עזרה |
|---|---|---|---|
| 1 | `אנא שלחו לי הודעה בדוא"ל לשלב הבא` / `אבקש סיוע טכני` | `Log = Step 1 of 2` ומייל שלב 2 | מייל לתמיכה, `Log = Step 1 with problem` |
| 2 | `סיום` / `אבקש סיוע טכני` | `Log = Step 2 of 2` והוספה לקבוצת MFA (ראו בהמשך) | מייל לתמיכה, `Log = Step 2 with problem` |

ראו באג 2 בסעיף 7.

3. **הוספה לקבוצת MFA:**
   - שליפת `onPremisesDistinguishedName` של המשתמש מ־Microsoft Graph.
   - הרצת זרימת Power Automate Desktop `AddUserToGroup` (`3853c697-378d-461c-bed5-d7773cdc8c0b`, unattended, [בקובץ](src/Workflows/AddUserToGroup-3853C697-378D-461C-BED5-D7773CDC8C0B.json)), שמוסיפה את המשתמש לקבוצה `CN=G-MFA-Enabled,OU=Groups,OU=Cloud,DC=smkb,DC=ac,DC=il` ב־AD המקומי.
4. **במקביל:** הזרימה ממתינה **4 ימים**, ואז מוסיפה את המשתמש לקבוצת MFA **בכל מקרה**, גם אם לא סיים. הריצה מסתיימת ב־**Succeeded**.

### ערך אחר ב־`Para`

הריצה מסתיימת ב־**Failed** (`Terminate_Should_not_came_here`).

---

## 4. ערכי `Log`

| מסלול | ערך | משמעות |
|---|---|---|
| `newuser` | (ריק) | עוד לא לחץ על אף כפתור |
| | `Step1` | איפס סיסמה, עוד לא אישר כניסה למייל |
| | `Step2` | ✅ סיים בהצלחה |
| | `Step1 with problem` | ❌ ביקש עזרה באיפוס סיסמה |
| | `Step2 with problem` | ❌ ביקש עזרה בכניסה למייל |
| `MSAuthenticator` | (ריק) | עוד לא לחץ על אף כפתור |
| | `Step 1 of 2` | באמצע ההתקנה |
| | `Step 2 of 2` | ✅ סיים והתווסף לקבוצת MFA |
| | `Step 1 with problem` / `Step 2 with problem` | ❌ ביקש עזרה בהתקנת Authenticator |
| `riskyuser` | – | המסלול לא כותב ל־Log |

בכל בקשת עזרה נשלח גם מייל ל־`Support.Srv@smkb.ac.il`.

---

## 5. איך יודעים אם הריצה הסתיימה בסדר

### סטטוס הריצה (רק ב־Run history של Power Automate, לא ברשימה)

| מסלול | סטטוס תקין |
|---|---|
| `riskyuser` | Succeeded |
| `newuser` | **Cancelled** (תמיד, גם כשהכול עבר בסדר) |
| `MSAuthenticator` | Succeeded |
| `backuser` | Succeeded |
| `Para` לא מוכר | Failed |

- **Running:** הזרימה ממתינה ללחיצה על כפתור או לסוף טיימר. זה תקין בימים הראשונים.
- **Failed** בכל מסלול אחר: פעולה נכשלה. פותחים את הריצה ומחפשים את הפעולה המסומנת באדום.
- `CallAddUserToGroup` במצב **Skipped**: זה צפוי (ראו באג 3).

### סיכום האינדיקטורים

| מצב | איך מזהים |
|---|---|
| ✅ הצליח | `Log` = `Step2` או `Step 2 of 2` |
| ⏳ באמצע | `Log` ריק, `Step1` או `Step 1 of 2`, והפריט עדיין בחלון הזמן (7 ימים ל־newuser, 4 ימים ל־MSAuthenticator) |
| ⚠️ תקוע | אותם ערכים, אבל **חלון הזמן כבר עבר**. הריצה הסתיימה, והפריט לא יתקדם לבד |
| ❌ ביקש עזרה | `Log` מסתיים ב־`with problem` |
| 🔴 כשל טכני | Failed ב־Run history, או ש־newuser לא יצר פריט `MSAuthenticator` אחרי 7 ימים |

**דחוף במיוחד:** פריט `MSAuthenticator` שתקוע אחרי 4 ימים. המשתמש כבר נמצא בקבוצת MFA בלי ש־Authenticator מותקן אצלו, והוא עלול להינעל מחוץ לחשבון.

**הערה:** מהרשימה אי אפשר להבדיל בין משתמש שלא הגיב לבין ריצה שנכשלה. בשני המקרים ה־Log נשאר תקוע. כדי לדעת מה קרה צריך להיכנס ל־Run history.

---

## 6. תצוגות מומלצות ברשימה

### סטודנטים שלא השלימו הפעלת חשבון
- `Para` = `newuser`
- `Job` = `Student`
- `Log` ≠ `Step2`

### סטודנטים שלא השלימו את המעבר ל־Authenticator
- `Para` = `MSAuthenticator`
- `Job` = `Student`
- `Log` ≠ `Step 2 of 2`

### פריטים תקועים
- `Log` ≠ `Step2` וגם `Log` ≠ `Step 2 of 2`
- `Modified` **is less than** `[Today]-7` (אפשר לשנות למספר ימים אחר)

עמודת `Modified` מתעדכנת בכל כתיבה ל־`Log`. אם מישהו עורך את הפריט ידנית, הפריט ייראה "טרי".

---

## 7. בעיות ידועות

1. **Employee: תנאי שלב 1 תמיד שקרי.** `ConditionStep1_3` משווה `""` ל־`"לשלב הבא"`, ולא את הכפתור שהמשתמש בחר. התוצאה: כל עובד יגיע ל־`Step1 with problem`, וייפתח מייל לתמיכה גם כשהכול בסדר.
2. **MSAuthenticator: תנאי שלב 1 לא תואם את הכפתור.** `Condition_3` בודק אם התשובה מכילה `שלחו לי הודעה לשלב הבא`, אבל הכפתור בפועל הוא `אנא שלחו לי הודעה בדוא"ל לשלב הבא`. אחרי `הודעה` מופיע `בדוא"ל`, אז הטקסט לא נמצא. התוצאה: כל משתמש יגיע ל־`Step 1 with problem` ולא יקבל את מייל שלב 2. ההוספה ל־MFA עדיין תקרה, דרך הטיימר של 4 ימים. (הבדיקה נעשתה על הקובץ המיוצא. כדאי לוודא שגם הגרסה בסביבה זהה.)
3. **`CallAddUserToGroup` / `CallAddUserToGroup_2` לעולם לא רצות.** הן מוגדרות לרוץ אחרי `Terminate`, ואחרי `Terminate` הזרימה לא ממשיכה.
4. **הטיימרים מבטלים ריצות שעדיין ממתינות.** אחרי 7 ימים (newuser) או 4 ימים (MSAuthenticator) הריצה מסתיימת. לחיצה על כפתור במייל אחרי הזמן הזה לא תעשה כלום.
5. **פרטי גישה כתובים בגלוי בזרימה.** שם המשתמש והסיסמה של Soprano נמצאים בכתובת של קריאת ה־SMS, ויש חתימות SAS (`sig=`) בכתובות של הזרימות החיצוניות. מומלץ להעביר אותם למשתני סביבה או ל־Azure Key Vault ולהחליף אותם. **בקובץ שבמאגר הסודות האלה, וגם ה־client secret של Graph, הוחלפו ב־`<REDACTED>`.** לפני ייבוא הפתרון מהמאגר צריך להחזיר אותם.
   - **גם בזרימת `AddUserToGroup`** שם המשתמש והסיסמה (המוצפנת) של חשבון השירות שמתחבר ל־AD הוחלפו ב־`<REDACTED>`, גם ב־`.data.xml` וגם בחבילה המקודדת ב־`.json`. אחרי ייבוא צריך להזין אותם מחדש ב־Power Automate Desktop.
   - **באפליקציית AgudaForm** (בתוך ה־`.msapp`) הוסרו כתובת המייל של משתמש הבדיקה וחתימת ה־SAS (`sig=`) של קישור ה־blob לתמונת הלוגו.
6. **`Get_file_content` במסלול MSAuthenticator** שולפת קובץ מ־OneDrive, אבל אף פעולה לא משתמשת בתוצאה.
7. **`backuser`** מוגדר כמסלול, אבל הוא ריק.

---

# AgudaForm – אפליקציית אישור חברות באגודה

## 8. אפליקציית AgudaForm

אפליקציית Canvas (פורמט טלפון, 640×1136) שבה סטודנטים מאשרים חברות באגודת הסטודנטים (דמי רווחה של 250 ש"ח) וחותמים על הצהרה. גרסה נוכחית: **2.6** (מוצגת בפינת מסך הפתיחה).

| | |
|---|---|
| שם תצוגה | `AgudaForm` |
| שם פנימי | `cr10a_agudaform_6de3a` |
| קבצים | [src/CanvasApps/](src/CanvasApps/). קוד האפליקציה נמצא בתוך `cr10a_agudaform_6de3a_DocumentUri.msapp` (קובץ zip; מסכי ה־YAML בתיקייה `Src/` שבתוכו) |
| מקור נתונים | זרימת הענן `AgudaForm-StudentEmailQuery` (`c268bac9-8a8f-4dfe-bee5-ad162a13b0b4`). **הזרימה לא נמצאת בפתרון הזה** |

### 8.1 הזרימה `AgudaForm-StudentEmailQuery`

האפליקציה לא ניגשת לנתונים ישירות. כל קריאה וכתיבה עוברות דרך הזרימה, עם שישה פרמטרים:

| פרמטר | ערכים שהאפליקציה שולחת |
|---|---|
| `userEmail` | `varUserEmail` |
| `para` | `Query` (שליפת סטטוס) / `Result` (שמירת אישור) / `Parking` (מנוי חניון, לא בשימוש) |
| `Status` | `None` / `New` / `Update` |
| `FormID` | `1`, או ה־`FormID` הקיים בעדכון |
| `HtmlText` | טקסט ההצהרה המלא (HTML), כולל החתימה |
| `Signature` | תמונת החתימה (`JSON(..., IncludeBinaryData)`) |

בשאילתה הזרימה מחזירה רשימה עם `StudentName`, `StudentMail`, `StudentStatus`, `StudentApprove`, `FormID`, שנשמרת ב־`colUserData`.

### 8.2 מסכים וזרימת משתמש

1. **Welcome** – בטעינה נשלפים נתוני הסטודנט (`para = Query`). אחרי 3 שניות (טיימר) מופיע הכפתור **"הצטרפות לאגודה"**, שמנתב לפי `StudentApprove`:

| `StudentApprove` | מסך יעד |
|---|---|
| `מאושר` | **ScrApprove** – "יש לך כבר אישור לחברות באגודה", קישור לכרטיס סטודנט בפורטל, וכפתור "סיום" |
| `לא מאושר` | **DeclerationUpdate** – הצהרה עם הודעה "שנה קודמת בחרת לא לשלם" |
| אחר / אין רשומה | **DeclerationNew** – הצהרה לסטודנט חדש |

   אם הסטטוס לא `מאושר` ולא `לא מאושר`, הכפתור קודם שולף את הנתונים מחדש ורק אז מנתב.

2. **DeclerationNew / DeclerationUpdate** – טקסט ההצהרה, תיבת חתימה (`PenInput`) וכפתור **"אישור"**, שמופעל רק אחרי חתימה. הלחיצה שולחת לזרימה `para = Result` עם `Status = New` או `Update`.
3. **Finish** – "תודה על הרישום, ניפגש".

### 8.3 מסכים לא פעילים

- **Parking** – רכישת מנוי מוזל לחניון קמפוס נמיר. כל ה־`Navigate(Parking)` בהערה, והכפתור "מנוי לחניון נמיר" במסך הפתיחה מוסתר (`Visible = false`). לפי הודעת ה־commit של גרסה 2.6: "ביטול מסכי ולחצני חנייה".
- **ScrNotApprove** – הסבר שאינך חבר/ת אגודה וטבלת מחירי חניון. הגישה אליו הייתה רק דרך כפתור החניון המוסתר.

### 8.4 בעיות ידועות באפליקציה

1. **המייל של המשתמש כתוב בקוד.** ב־`Welcome.OnVisible` השורה `Set(varUserEmail, Lower(User().Email))` בהערה, ובמקומה כתובת קבועה של משתמש בדיקה (בקובץ שבמאגר הכתובת הוחלפה ב־`<REDACTED>@smkb.ac.il`). במצב הזה **כל משתמש יראה ויעדכן את הרשומה של משתמש הבדיקה**. חובה להחזיר את `User().Email` לפני פרסום.
2. **DeclerationUpdate לא ממשיך אחרי אישור.** הכפתור שולח את הנתונים לזרימה, אבל `Navigate` בהערה ואין מעבר ל־Finish. המשתמש נשאר במסך ההצהרה ועלול ללחוץ שוב (ולשלוח שוב).
3. **הטקסט ב־Welcome עדיין מזכיר את החניון** ("ו/או לרכישת מנוי מוזל לחניון"), למרות שהאפשרות בוטלה.
4. **שם המשתמש בהצהרה נלקח מ־`First(colUserData)`** ולא מהרשומה שתואמת את `varUserEmail`, בשונה מהניתוב שמשתמש ב־`LookUp`.

---

## 9. תלויות

### Activate StudentsV3

| רכיב | פרטים |
|---|---|
| SharePoint Online | הרשימה בסעיף 1 |
| Office 365 Outlook | שליחת מיילים, כולל מיילים עם כפתורים (Send email with options) |
| Office 365 Users | שליפת פרופיל משתמש |
| OneDrive for Business | `Get_file_content` |
| Power Automate Desktop | זרימה `3853c697-378d-461c-bed5-d7773cdc8c0b`. מוסיפה משתמש לקבוצה ב־AD המקומי. דורשת מכונה זמינה במצב unattended |
| Microsoft Graph | שליפת `onPremisesDistinguishedName` (אפליקציה רשומה ב־Entra ID, client credentials) |
| Soprano | שליחת SMS |
| זרימות חיצוניות (HTTP trigger) | הוספה ל־gategoing, והוספה לקבוצה (`CallAddUserToGroup`) |
| תמיכה | `Support.Srv@smkb.ac.il` |

### AgudaForm

| רכיב | פרטים |
|---|---|
| Logic flows | הזרימה `AgudaForm-StudentEmailQuery` (מחוץ לפתרון) |
| Office 365 Outlook, SharePoint, Office 365 Users | חיבורים שהזרימה `AgudaForm-StudentEmailQuery` משתמשת בהם |

</div>
