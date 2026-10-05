# 19. Why Guaranteed Prevention of Secret Exposure Is Not Possible

**Project:** AI Scan Repos
**Document Version:** 1.0
**Date:** 2026-08-02
**Status:** Current
**Subject:** the limits of preventing a credential that exists *inside a
reviewed repository* from appearing in a report, a log, or a model provider's
request

> 🇬🇧 **English** — below · 🇮🇱 **עברית** — [לחצו כאן](#חלק-ב--עברית)
>
> Related: [06-threat-model.md](06-threat-model.md) (rows *"Detected secret is
> reproduced in report/AI"* and *"Model response reflects secret or unsafe
> link"*) · [03-high-level-design/09-ai-analysis-engine.md](03-high-level-design/09-ai-analysis-engine.md)
> (Redaction and Token Budget) · [17-review-api.md](17-review-api.md) ·
> [15-mcp-server-documentation.md](15-mcp-server-documentation.md)

---

# Part A — English

## 1. The question

> If an API key or credential is stored in the repository being reviewed, can
> it appear in the report or in the logs — or is it impossible for the AI to
> pass that information on?

**Short answer: it is not impossible, and it cannot be made impossible.**

What *can* be done is reduce the likelihood, shrink the blast radius, and
detect it when it happens. This document explains why the stronger guarantee
is unreachable, so that the limitation is a documented, accepted property of
the system rather than an oversight.

### First, a distinction that matters

Two different assets are easy to confuse:

| Asset | Status |
|---|---|
| **The clone token** — the PAT the server uses to *reach* a private repository | **Protected.** Environment-only, never a tool argument, delivered to git via `GIT_ASKPASS`, redacted at every message-building site, absent from argv, URLs, logs and reports |
| **Credentials committed *inside* the repository under review** | **The subject of this document.** None of the machinery above applies to them |

The first is a secret the system *owns* and can therefore control absolutely.
The second is a secret the system *is being asked to look at*. That difference
is the root of everything below.

---

## 2. Six structural reasons

### 2.1 The goal contradicts the product's purpose

Finding hardcoded secrets is a **required feature**. A report that says
*"a hardcoded Django SECRET_KEY is present at `settings.py:22`"* has already
disclosed information about the secret: that it exists, what kind it is, and
exactly where to find it. Anyone holding that report and any read access to
the repository now has the value.

Perfect non-disclosure would mean not reporting the finding at all — which is
the tool failing at its job. There is no configuration that gives both. The
only real decision is **where on that spectrum to sit**, and the sane position
is: report the location and impact, never the value.

### 2.2 The model must read the plaintext

AI review requires the source code in the prompt. There is no practical way
to have a language model reason about code it cannot see — no blind or
homomorphic equivalent exists for this task.

The consequence is timing-based and irreversible: **by the time any report is
written, the secret has already left the process.** With an on-prem provider
it reached another process on your network. With a cloud provider
(`google_ai_studio`, `openai`, `groq`, `azure_openai`) it reached a third
party, together with whatever retention, logging and abuse-monitoring policy
that provider applies. Sanitising the report afterwards does not undo the
request.

This is why the provider registry labels every endpoint `on_prem` or `cloud`
and surfaces that label in `/api/v1/providers` and in the demo console. The
label exists precisely because this step cannot be removed — only made
visible and chosen deliberately.

### 2.3 You cannot redact what you cannot detect

Any redaction filter is itself a secret detector, and secret detection has no
complete solution. Secrets have no universal syntax:

- **High-entropy strings** — catchable, but so are commit hashes, UUIDs,
  minified assets, base64 test fixtures and checksums.
- **Low-entropy secrets** — `password = "letmein"`, or a passphrase in plain
  English, carry no statistical signal at all.
- **Structural variation** — a private key split across lines, a token
  assembled by concatenation, a value hidden in a comment or a docstring, a
  credential inside a connection string or a URL.
- **Custom formats** — every internal system invents its own token shape.

Every filter therefore produces both error types, and both are harmful:

| Error | Consequence |
|---|---|
| **False negative** (missed secret) | The value passes through. The guarantee is broken by a single miss |
| **False positive** (over-redaction) | Legitimate findings are mangled into unreadable `[REDACTED]` soup, and the report loses its value |

The project's own threat model states this limitation directly
([06-threat-model.md:132](06-threat-model.md)):

> Secret detection and redaction may not identify novel credential formats.

A guarantee cannot be built on a best-effort detector. This is the single
strongest reason the answer is "not possible" rather than "not yet
implemented".

### 2.4 Model output is unbounded natural language

A generative model cannot be *constrained* to never emit a particular
substring. It can only be **instructed** not to, and its output **filtered**
afterwards — and that filter inherits every limitation in §2.3.

Worse, exact-match filtering is defeated by ordinary paraphrase, with no
malice required:

- splitting the value across a sentence or a code block;
- describing it (*"the key begins `django-insecure-` followed by 50
  characters"*) — which is often enough to reconstruct or to confirm a guess;
- reproducing it inside a *"here is the fixed code"* suggestion;
- encoding or reformatting it.

An output scrubber catches the literal copy. It does not catch a
transformation, and enumerating all transformations is not a finite task.

### 2.5 There are more output channels than the report

Even a perfect report scrubber would not control every path out:

| Channel | Controlled by us? |
|---|---|
| `reports/<id>.json` and `.md` | Yes |
| `<id>.partial.json` crash checkpoints | Yes |
| `<report>.notes.md`, written by the model through `append_report_notes` | Partly — the content is model-authored free text |
| The **MCP tool response**, which enters the host model's context and the host's chat transcript (e.g. LM Studio's stored conversation) | **No** |
| The **cloud provider's** own request logs and retention | **No** |
| `ingest_repository_only(output_path=…)`, which writes the **entire repository source** verbatim to `reports/` | By policy only — it is the tool's documented purpose |
| `findings_file` intake — descriptions from an upstream scanner are attached **verbatim** and deliberately cannot be altered by the model | By upstream scanner configuration |

Two of these are outside the software's reach entirely. A guarantee that
depends on components you do not own is not a guarantee.

### 2.6 Repository content is attacker-controlled

The code under review is **untrusted input**. A repository can contain text
crafted to manipulate the reviewing model — for example, instructing it to
include literal values, or to ignore prior instructions.

The project already treats content as untrusted and says so in the prompt
(`_UNTRUSTED_DATA_INSTRUCTION` in `prompts.py`, and the equivalent line in
`llm.py`). That is the correct mitigation, and it is not a guarantee:
instruction-following models comply with system prompts *probabilistically*,
not deterministically. Prompt injection remains an open research problem.

So for a hostile repository the question is not merely *"might a secret leak
by accident?"* but *"can the repository actively try to make it leak?"* — and
the honest answer to the second is yes, it can try.

---

## 3. What the current implementation actually does

Assessed by reading the code and by inspecting real reports produced by
`google/gemma-4-26b-a4b-qat` against a repository containing a real hardcoded
`SECRET_KEY` and plaintext database credentials in comments.

### 3.1 Verified safe

| Component | Finding |
|---|---|
| **Quick-mode scanner** (`surface.py`) | Structurally cannot leak a value. `surface_scan` records only `severity`, `line`, and a **fixed** title string — the matched text is never captured |
| **Logging** | No file content, prompt, or model response is logged anywhere in either component. Only git commands (already token-redacted) and startup lines |
| **JSON failure path** | Python's `JSONDecodeError` string omits the document, so a malformed model response containing a secret does not reach `failures[].error` |
| **Report schema** | Carries file and line references, not source snippets. Checkpoints store model output, not the packed source |

### 3.2 Guaranteed exposure, with no model involved

1. **`ingest_repository_only(output_path=…)`** writes every file's full
   content to `reports/<name>.json` in plaintext. This is the tool's stated
   purpose, but it is the largest single surface, and the host model can
   invoke it.
2. **`findings_file` descriptions** are copied verbatim (300-character cap)
   into `static_findings[].description`. That path is *deliberately*
   identity-preserving so that the model cannot delete or downgrade a
   deterministic finding — which also means nothing downstream will strip a
   secret the upstream scanner placed there.

### 3.3 Model-dependent, and currently unenforced

Nothing in the code constrains what the model writes into `title`,
`description` or `suggestion`.

**Observed behaviour was good.** Across three real deep reviews of the same
file, the model wrote *"The Django SECRET_KEY is hardcoded in the source
code"* and described the impact — and never reproduced the value. **Three out
of three.**

That is encouraging, and it is **not a control**. It is one model, one file,
three samples. Note also that the `secrets` persona actively invites the
opposite behaviour:

> *"Report every credential, API key, token, private key or connection string
> committed to the code, and say what an attacker could do with each."*

### 3.4 An aggravating factor

`reports/` persists on the host and is shared by both front ends, and
`GET /api/v1/reviews/{id}/report` is **unauthenticated by default**
(`api_key: ""`). Anything that reaches a report is readable by anyone who can
reach the port.

---

## 4. What is achievable instead

"Not guaranteeable" must not be read as "nothing can be done". The realistic
objective is **defence in depth plus detection**, which is exactly what the
threat model prescribes.

| Control | Effect | Honest limitation |
|---|---|---|
| **Prompt rule**: never reproduce a credential value; cite file and line | Removes the most common accidental case | Probabilistic; defeated by injection |
| **Output scrubber** over model text and findings descriptions | Catches literal copies of detected patterns | Bounded by §2.3; blind to paraphrase |
| **Seeded canary-secret test** | Proves the property holds for known formats and prevents regressions | Only tests formats you thought of |
| **Prefer `on_prem` providers for sensitive repositories** | Keeps source inside the network | Does not help the report channel |
| **Set `api_key`, keep the bind on localhost** | Limits who can read reports | Does not stop creation |
| **Treat `reports/` as sensitive** — same handling as source | Correct classification | Operational discipline |

### 4.1 The control that actually resolves it

> **Any credential found by a code review must be treated as already
> compromised, and rotated.**

This is the only measure that makes the disclosure question moot, and it holds
regardless of how well the pipeline behaves. The reasoning is simple: the
credential was committed to a repository. It is in the git history, on every
clone, in every developer's working copy, and possibly in CI logs and
backups — **long before this tool ever read it**. The review did not create
the exposure; it discovered it.

A team that rotates on discovery is safe even if a report quotes the value in
full. A team that does not rotate is already exposed even if the report is
perfectly redacted. **The pipeline's behaviour is the smaller risk.**

---

## 5. Conclusion

Guaranteed prevention is unreachable because it requires, simultaneously:

1. reporting secrets while not disclosing them (§2.1);
2. reasoning over plaintext the model must receive (§2.2);
3. a complete secret detector, which does not exist (§2.3);
4. deterministic control over generative output (§2.4);
5. authority over channels we do not own (§2.5);
6. immunity to attacker-controlled input (§2.6).

Each is independently sufficient to break the guarantee. Together they make it
a **structural property of AI-assisted code review**, not a defect of this
implementation.

The correct posture is therefore: state the limitation openly, implement the
defence-in-depth controls in §4, choose provider location deliberately, treat
review artifacts as sensitive — and **rotate anything a review finds**.

---
---

<a id="חלק-ב--עברית"></a>

# חלק ב׳ — עברית

## 1. השאלה

> אם מפתח API או פרטי גישה שמורים בתוך המאגר שנסרק, האם הם עלולים להופיע
> בדוח או בלוגים — או שבלתי אפשרי שה־AI יעביר את המידע הזה הלאה?

**תשובה קצרה: זה אפשרי, ולא ניתן למנוע זאת באופן מוחלט.**

מה שכן אפשר לעשות הוא להקטין את הסבירות, לצמצם את היקף הנזק, ולזהות מקרים
כאלה כשהם קורים. המסמך הזה מסביר מדוע ההבטחה החזקה יותר אינה ברת־השגה, כדי
שהמגבלה תהיה תכונה מתועדת ומקובלת של המערכת — ולא פספוס.

### קודם כול, הבחנה חשובה

קל לבלבל בין שני נכסים שונים לחלוטין:

| הנכס | המצב |
|---|---|
| **טוקן השכפול (clone token)** — ה־PAT שהשרת משתמש בו כדי *להגיע* למאגר פרטי | **מוגן.** נקרא ממשתני סביבה בלבד, לעולם לא כארגומנט של כלי, מועבר ל־git דרך `GIT_ASKPASS`, מוסתר בכל מקום שבו נבנית הודעה, ואינו מופיע ב־argv, בכתובות, בלוגים או בדוחות |
| **פרטי גישה שנשמרו *בתוך* המאגר הנסרק** | **הנושא של המסמך הזה.** אף אחד מהמנגנונים שלמעלה לא חל עליהם |

הראשון הוא סוד שהמערכת **מחזיקה**, ולכן יכולה לשלוט בו באופן מוחלט. השני הוא
סוד שהמערכת **מתבקשת להסתכל עליו**. ההבדל הזה הוא השורש של כל מה שלהלן.

---

## 2. שש סיבות מבניות

### 2.1 המטרה סותרת את ייעוד המוצר

איתור סודות שנשמרו בקוד הוא **דרישה מהמוצר**. דוח שאומר *"קיים
`SECRET_KEY` קשיח ב־`settings.py:22`"* כבר חשף מידע על הסוד: שהוא קיים, מאיזה
סוג הוא, ואיפה בדיוק למצוא אותו. כל מי שמחזיק בדוח הזה ויש לו גישת קריאה
כלשהי למאגר — מחזיק למעשה בערך עצמו.

אי־חשיפה מושלמת פירושה לא לדווח על הממצא בכלל — כלומר שהכלי נכשל בתפקידו. אין
הגדרה שנותנת את שניהם. ההחלטה האמיתית היחידה היא **היכן על הרצף למקם את
עצמנו**, והעמדה ההגיונית היא: לדווח על המיקום ועל ההשלכה, לעולם לא על הערך.

### 2.2 המודל חייב לקרוא את הטקסט הגלוי

סקירת קוד באמצעות AI מחייבת שקוד המקור יגיע לפרומפט. אין דרך מעשית שמודל שפה
ינתח קוד שהוא לא רואה — לא קיים מקבילה "עיוורת" או הצפנה הומומורפית למשימה הזו.

ההשלכה נוגעת לתזמון והיא בלתי הפיכה: **עד שנכתב דוח כלשהו, הסוד כבר יצא
מהתהליך.** עם ספק מקומי (`on_prem`) הוא הגיע לתהליך אחר ברשת שלכם. עם ספק
ענן (`google_ai_studio`, `openai`, `groq`, `azure_openai`) הוא הגיע לצד
שלישי — יחד עם כל מדיניות השמירה, הלוגים וניטור השימוש לרעה של אותו ספק.
ניקוי הדוח בדיעבד לא מבטל את הבקשה שכבר נשלחה.

זו בדיוק הסיבה שרישום הספקים מסמן כל נקודת קצה כ־`on_prem` או `cloud`, ומציג
את הסימון הזה ב־`/api/v1/providers` ובקונסולת ההדגמה. הסימון קיים משום שאי
אפשר לבטל את השלב הזה — אפשר רק להפוך אותו לגלוי ולבחירה מודעת.

### 2.3 אי אפשר להסתיר את מה שאי אפשר לזהות

כל מסנן הסתרה הוא בעצמו מזהה סודות, ולזיהוי סודות אין פתרון שלם. לסודות אין
תחביר אחיד:

- **מחרוזות בעלות אנטרופיה גבוהה** — ניתנות לזיהוי, אבל כך גם חתימות commit,
  מזהי UUID, קבצים ממוזערים, נתוני בדיקה ב־base64 וסכומי ביקורת.
- **סודות בעלי אנטרופיה נמוכה** — `password = "letmein"`, או משפט־סיסמה
  באנגלית פשוטה, אינם נושאים שום סימן סטטיסטי.
- **שוני מבני** — מפתח פרטי שמפוצל לשורות, טוקן שמורכב משרשור מחרוזות, ערך
  שמוסתר בהערה או ב־docstring, או פרטי גישה בתוך מחרוזת חיבור או כתובת URL.
- **פורמטים ייחודיים** — כל מערכת פנימית ממציאה לעצמה מבנה טוקן משלה.

לכן כל מסנן מייצר את שני סוגי השגיאות, ושניהם מזיקים:

| סוג השגיאה | ההשלכה |
|---|---|
| **שלילי שגוי** (סוד שלא זוהה) | הערך עובר החוצה. החמצה אחת מספיקה כדי לשבור את ההבטחה |
| **חיובי שגוי** (הסתרת יתר) | ממצאים לגיטימיים הופכים לגיבוב בלתי קריא של `[REDACTED]`, והדוח מאבד את ערכו |

מודל האיומים של הפרויקט עצמו מציין את המגבלה הזו במפורש
([06-threat-model.md:132](06-threat-model.md)):

> זיהוי והסתרה של סודות עלולים שלא לזהות פורמטים חדשים של פרטי גישה.

אי אפשר לבנות הבטחה על גבי מזהה שפועל במאמץ מיטבי בלבד. זו הסיבה החזקה ביותר
לכך שהתשובה היא "לא אפשרי" ולא "עדיין לא מומש".

### 2.4 פלט המודל הוא שפה טבעית בלתי מוגבלת

אי אפשר **לאלץ** מודל גנרטיבי לעולם לא לפלוט מחרוזת מסוימת. אפשר רק
**להנחות** אותו שלא לעשות זאת, ו**לסנן** את הפלט בדיעבד — והמסנן הזה יורש את
כל המגבלות שבסעיף 2.3.

גרוע מכך, סינון לפי התאמה מדויקת נשבר על ידי ניסוח מחדש רגיל, בלי שום כוונת
זדון:

- פיצול הערך על פני משפט או בלוק קוד;
- תיאור שלו (*"המפתח מתחיל ב־`django-insecure-` ואחריו 50 תווים"*) — שלעיתים
  קרובות מספיק כדי לשחזר אותו או לאשש ניחוש;
- שכפולו בתוך הצעה מסוג *"הנה הקוד המתוקן"*;
- קידוד או עיצוב מחדש שלו.

מסנן פלט תופס העתקה מילולית. הוא לא תופס טרנספורמציה, ומניית כל
הטרנספורמציות האפשריות אינה משימה סופית.

### 2.5 יש יותר ערוצי פלט מאשר הדוח

אפילו מסנן דוחות מושלם לא היה שולט בכל נתיב היציאה:

| הערוץ | בשליטתנו? |
|---|---|
| `reports/<id>.json` ו־`.md` | כן |
| נקודות שמירה `<id>.partial.json` | כן |
| `<report>.notes.md`, שנכתב על ידי המודל דרך `append_report_notes` | חלקית — התוכן הוא טקסט חופשי שהמודל מחבר |
| **תשובת כלי ה־MCP**, שנכנסת להקשר של מודל המארח ולתמלול השיחה שלו (למשל השיחה השמורה ב־LM Studio) | **לא** |
| הלוגים ומדיניות השמירה של **ספק הענן** | **לא** |
| `ingest_repository_only(output_path=…)`, שכותב את **כל קוד המקור של המאגר** כלשונו אל `reports/` | במדיניות בלבד — זהו ייעודו המתועד של הכלי |
| קליטת `findings_file` — תיאורים מסורק חיצוני מצורפים **כלשונם** ובמכוון אינם ניתנים לשינוי על ידי המודל | בהגדרות הסורק החיצוני |

שניים מאלה נמצאים לחלוטין מחוץ להישג ידה של התוכנה. הבטחה שתלויה ברכיבים
שאינם בבעלותכם אינה הבטחה.

### 2.6 תוכן המאגר נשלט על ידי התוקף

הקוד הנסרק הוא **קלט לא מהימן**. מאגר יכול להכיל טקסט שנבנה במיוחד כדי לתמרן
את המודל הסוקר — למשל להנחות אותו לכלול ערכים מילוליים, או להתעלם מהוראות
קודמות.

הפרויקט כבר מתייחס לתוכן כלא מהימן ואומר זאת בפרומפט
(`_UNTRUSTED_DATA_INSTRUCTION` ב־`prompts.py`, והשורה המקבילה ב־`llm.py`). זהו
אמצעי ההגנה הנכון, והוא אינו הבטחה: מודלים שעוקבים אחר הוראות מצייתים
לפרומפט המערכת **בהסתברות**, לא באופן דטרמיניסטי. הזרקת הנחיות
(prompt injection) נותרה בעיית מחקר פתוחה.

לכן, עבור מאגר עוין, השאלה אינה רק *"האם סוד עלול לדלוף בטעות?"* אלא *"האם
המאגר יכול לנסות באופן פעיל לגרום לדליפה?"* — והתשובה הכנה לשנייה היא כן, הוא
יכול לנסות.

---

## 3. מה המימוש הנוכחי באמת עושה

הבדיקה נעשתה על ידי קריאת הקוד ובחינת דוחות אמיתיים שהופקו על ידי
`google/gemma-4-26b-a4b-qat` מול מאגר שמכיל `SECRET_KEY` קשיח אמיתי ופרטי
גישה למסד נתונים בטקסט גלוי בתוך הערות.

### 3.1 נבדק ונמצא בטוח

| הרכיב | הממצא |
|---|---|
| **סורק מצב quick** (`surface.py`) | מבנית אינו יכול לדלוף ערך. `surface_scan` שומר רק `severity`, `line` ומחרוזת כותרת **קבועה** — הטקסט שהותאם לעולם אינו נשמר |
| **לוגים** | תוכן קבצים, פרומפטים ותשובות מודל אינם נרשמים בשום מקום באף אחד מהרכיבים. רק פקודות git (שכבר מוסתרות מטוקן) ושורות עלייה |
| **נתיב כשל JSON** | מחרוזת ה־`JSONDecodeError` של פייתון אינה כוללת את המסמך, ולכן תשובת מודל פגומה שמכילה סוד אינה מגיעה אל `failures[].error` |
| **סכמת הדוח** | מכילה הפניות לקובץ ולשורה, לא קטעי קוד. נקודות השמירה מאחסנות פלט של המודל, לא את המקור הארוז |

### 3.2 חשיפה מובטחת, ללא מעורבות המודל

1. **`ingest_repository_only(output_path=…)`** כותב את התוכן המלא של כל קובץ
   אל `reports/<name>.json` בטקסט גלוי. זהו ייעודו המוצהר של הכלי, אבל זהו
   משטח החשיפה הגדול ביותר — ומודל המארח יכול להפעיל אותו.
2. **תיאורי `findings_file`** מועתקים כלשונם (עד 300 תווים) אל
   `static_findings[].description`. הנתיב הזה שומר על זהות הממצא **במכוון**,
   כדי שהמודל לא יוכל למחוק או להנמיך ממצא דטרמיניסטי — מה שגם אומר ששום שלב
   בהמשך לא יסיר סוד שהסורק החיצוני שם שם.

### 3.3 תלוי במודל, וכרגע לא נאכף

שום דבר בקוד אינו מגביל את מה שהמודל כותב לתוך `title`, `description` או
`suggestion`.

**ההתנהגות שנצפתה הייתה טובה.** בשלוש סקירות עומק אמיתיות של אותו קובץ, המודל
כתב *"ה־`SECRET_KEY` של Django קשיח בקוד המקור"* ותיאר את ההשלכה — ומעולם לא
שכפל את הערך. **שלוש מתוך שלוש.**

זה מעודד, וזה **לא אמצעי בקרה**. זהו מודל אחד, קובץ אחד, שלוש דגימות. שימו לב
גם שהפרסונה `secrets` דווקא מזמינה את ההתנהגות ההפוכה:

> *"דווח על כל פרטי גישה, מפתח API, טוקן, מפתח פרטי או מחרוזת חיבור שנשמרו
> בקוד, ואמור מה תוקף יכול לעשות עם כל אחד מהם."*

### 3.4 גורם מחמיר

תיקיית `reports/` נשמרת על המארח ומשותפת לשתי החזיתות, ו־
`GET /api/v1/reviews/{id}/report` **אינו מאומת כברירת מחדל** (`api_key: ""`).
כל דבר שמגיע לדוח ניתן לקריאה על ידי כל מי שיכול להגיע לפורט.

---

## 4. מה כן ניתן להשיג

"לא ניתן להבטחה" אסור שייקרא כ"אין מה לעשות". היעד המציאותי הוא **הגנה
לעומק בשילוב זיהוי**, וזה בדיוק מה שמודל האיומים מורה.

| אמצעי הבקרה | ההשפעה | המגבלה הכנה |
|---|---|---|
| **כלל בפרומפט**: לעולם אל תשכפל ערך של פרטי גישה; ציין קובץ ושורה | מסיר את המקרה המקרי הנפוץ ביותר | הסתברותי; נשבר על ידי הזרקה |
| **מסנן פלט** על טקסט המודל ועל תיאורי ממצאים | תופס העתקות מילוליות של תבניות מוכרות | מוגבל לפי 2.3; עיוור לניסוח מחדש |
| **בדיקת סוד־פיתיון (canary) מושתל** | מוכיחה שהתכונה מתקיימת עבור פורמטים ידועים ומונעת נסיגות | בודקת רק פורמטים שחשבתם עליהם |
| **העדפת ספקי `on_prem` למאגרים רגישים** | שומרת את קוד המקור בתוך הרשת | לא עוזרת בערוץ הדוח |
| **הגדרת `api_key`, השארת ההאזנה על localhost** | מגבילה מי יכול לקרוא דוחות | לא מונעת את היצירה |
| **התייחסות ל־`reports/` כרגיש** — אותו טיפול כמו לקוד המקור | סיווג נכון | משמעת תפעולית |

### 4.1 אמצעי הבקרה שבאמת פותר את זה

> **יש להתייחס לכל פרטי גישה שנמצאו בסקירת קוד כאל כאלה שכבר נחשפו — ולהחליף
> אותם.**

זהו האמצעי היחיד שהופך את שאלת החשיפה ללא רלוונטית, והוא תקף ללא קשר לאיכות
התנהגותו של הצינור. הנימוק פשוט: פרטי הגישה הוכנסו למאגר. הם נמצאים בהיסטוריית
ה־git, בכל שכפול, בעותק העבודה של כל מפתח, ואולי גם בלוגים של CI ובגיבויים —
**הרבה לפני שהכלי הזה בכלל קרא אותם**. הסקירה לא יצרה את החשיפה; היא גילתה
אותה.

צוות שמחליף פרטי גישה עם הגילוי מוגן גם אם הדוח מצטט את הערך במלואו. צוות
שאינו מחליף כבר חשוף גם אם הדוח מוסתר בצורה מושלמת. **התנהגות הצינור היא
הסיכון הקטן יותר.**

---

## 5. מסקנה

מניעה מובטחת אינה ברת־השגה משום שהיא דורשת, בו־זמנית:

1. לדווח על סודות מבלי לחשוף אותם (2.1);
2. לנתח טקסט גלוי שהמודל חייב לקבל (2.2);
3. מזהה סודות שלם, שאינו קיים (2.3);
4. שליטה דטרמיניסטית בפלט גנרטיבי (2.4);
5. סמכות על ערוצים שאינם בבעלותנו (2.5);
6. חסינות בפני קלט שנשלט על ידי תוקף (2.6).

כל אחת מהן מספיקה בפני עצמה כדי לשבור את ההבטחה. יחד הן הופכות אותה
ל**תכונה מבנית של סקירת קוד בסיוע AI**, ולא לפגם במימוש הזה.

העמדה הנכונה היא אפוא: להצהיר על המגבלה בגלוי, לממש את אמצעי ההגנה לעומק
שבסעיף 4, לבחור את מיקום הספק במודע, להתייחס לתוצרי הסקירה כרגישים — **ולהחליף
כל דבר שסקירה מוצאת.**
