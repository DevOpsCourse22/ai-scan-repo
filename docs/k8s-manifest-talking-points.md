# Kubernetes Manifest — Points Worth Being Able to Explain

Running list of non-obvious decisions made while writing the K8s manifests,
kept in Q&A form so whoever presents this (to the lecturer, to each other)
has a ready answer instead of re-deriving it live. Add to this as we go —
one entry per point, short enough to actually remember.

---

**Q: Why `ReadWriteOnce` on the reports PVC instead of `ReadWriteMany`?**

A: Two independent reasons that happen to point the same way:

1. Our cluster's default storage (k3s's `local-path` provisioner) only
   supports `ReadWriteOnce` — a PVC can be mounted for writing by Pods on
   one node at a time. `ReadWriteMany` needs different storage
   infrastructure (e.g. NFS) that we don't have.
2. Even if we had `ReadWriteMany` available, the platform currently runs
   one review at a time by design — a single local LLM (Elad's machine)
   would just queue concurrent requests against itself anyway (see
   `jobs.py`'s `JobStore` docstring). So `ReadWriteOnce` isn't a
   compromise forced by missing infrastructure; it matches a constraint
   that already exists for an unrelated reason (the AI backend).

**If asked "what would change to allow parallel reviews":** the
deterministic scanner stage doesn't need the shared LLM and could run
concurrently today. Real parallelism for the AI stage would need either
multiple LLM backends (cost/hardware) or a queueing model in front of one
— and only then would `ReadWriteMany` storage actually matter.

---

**Q: Why can't we just rename the namespace later if we pick the wrong name?**

A: Kubernetes Namespaces are immutable — there's no rename operation.
Changing a namespace's name means creating a new Namespace object and
recreating everything inside it (Deployments, Secrets, the PVC and its
data). That's why naming was settled explicitly before anything real was
deployed into it, rather than treated as a detail to fix later.

---

**Q: Why is everything in one namespace? Wouldn't separating the scanner
Jobs from the services be stronger isolation?**

A: It would be, but it is not possible with this design. The Jobs write their
reports to the shared reports PVC, and a PVC can only be mounted by Pods in
its own namespace. So the Jobs must live where the PVC lives. A second
namespace (`ai-scan-repos-jobs`) was defined early to keep the option open;
it was deleted on 2026-09-30 because it contradicts the design. The isolation
it would have bought comes from elsewhere instead: the Jobs get no
ServiceAccount token, run as non-root under Pod Security `restricted`, and
review-api's RBAC cannot read Pods or Job specs. (Elad's earlier `code-review`
namespace was merged into `ai-scan-repos` on the same day.)

---

**Q: Why a headless Service (`clusterIP: None`) for Postgres instead of a
normal one, like gateway's?**

A: A normal Service is a load-balanced virtual IP — fine when any replica
can answer a request, which is true for gateway but not for a database. A
headless Service instead gives each Pod its own stable DNS name
(`postgres-0.postgres.ai-scan-repos.svc.cluster.local`), so a client always
reaches the SAME instance. That only matters once there's more than one
Postgres Pod, but the Service type has to be chosen now — you can't turn a
normal Service headless later without changing how every client connects.

---

**Q: What's `volumeClaimTemplates` for, and why doesn't the gateway/backend
Deployments use it?**

A: It's a StatefulSet-only mechanism that provisions one PVC per Pod
automatically and keeps it matched to that Pod's identity across restarts
(`postgres-data-postgres-0`, and `-postgres-1` if ever scaled). A plain
Deployment has no per-replica identity, so if two replicas both mounted
one PVC they'd race to write the same files — which is exactly why our
Deployments (gateway, backend) don't use PVCs for anything they'd run more
than one replica of.

---

**Q: Why is `PGDATA` set to a subdirectory (`/var/lib/postgresql/data/pgdata`)
instead of the mount path itself?**

A: `initdb` refuses to initialize a non-empty directory, and some CSI
storage drivers write a `lost+found` entry into a freshly-provisioned
volume's root — enough for `initdb` to see the mount root as "non-empty"
and fail on first boot. Pointing `PGDATA` at a subdirectory sidesteps this
regardless of which storage driver the cluster happens to use.

---

**Q: Doesn't the namespace's "restricted" Pod Security Standard conflict
with running Postgres, which traditionally wants root for setup?**

A: Possibly, and this is flagged as unverified in `postgres.yaml` rather
than assumed away. The official `postgres` image is built to support
running as its own non-root `postgres` user (UID 999) against restricted
security contexts, which is what the manifest asks for — but this has not
yet been run against Elad's actual cluster. If it turns out not to work,
the fallback is either a rootless-focused Postgres image variant, or
putting Postgres in a namespace with a less strict Pod Security level than
`restricted` (e.g. `baseline`) — a real trade-off to make consciously if
it comes to that, not silently.

---

## הסבר בעברית — securityContext, שדה-שדה, ולמה כל אחד קיים

לשימוש בהצגה בעל-פה למרצה. שישה שדות, כל אחד סוגר איום ספציפי:

**`runAsNonRoot: true`** — קוברנטיס מסרב להריץ את ה-container אם הוא ינסה
לרוץ כ-UID 0 (root). הסיבה: אם יש חולשה בקוד שמאפשרת לתוקף להריץ קוד,
וה-process רץ כ-root — הקוד הזדוני **גם הוא** רץ כ-root. root בתוך
container לא "מבודד" באמת כמו שנדמה, כי הוא חולק kernel עם המחשב המארח,
ומקל משמעותית על ניסיונות בריחה מה-container. השדה הזה נאכף **בזמן ריצה**
ע"י קוברנטיס עצמו — לא סומך על כך שהמפתח לא שכח לכתוב `USER` ב-Dockerfile.

**`runAsUser: 10001`** — קובע איזה UID *בדיוק*. חייב להתאים למה
שה-Dockerfile באמת יצר (`useradd --uid 10001`), אחרת מקבלים שגיאות הרשאה
(קבצים שנוצרו ע"י UID אחד, מנוסה להיפתח ע"י UID שונה).

**`seccompProfile: RuntimeDefault`** — syscalls הם הדרך שתהליך "מבקש"
מה-kernel לבצע פעולה (לפתוח קובץ, ליצור חיבור רשת וכו'). בלי הגבלה,
container יכול לקרוא לכל syscall שקיים — כולל נדירים ומסוכנים, שדרכם
עברו לא מעט חולשות אבטחה היסטוריות. הפרופיל הזה מגביל לרשימה שאפליקציה
רגילה באמת צריכה — גם אם תוקף מריץ קוד, הוא לא יכול לקרוא ל-syscalls
שמחוץ לרשימה.

**`allowPrivilegeEscalation: false`** — שונה מ-`runAsNonRoot`! זה חוסם
מנגנון ספציפי (setuid binaries — קבצים שרצים בהרשאות ה*בעלים* שלהם, לא
של מי שהריץ אותם — כך `sudo`/`passwd` עובדים). גם תהליך non-root יכול
תיאורטית "לטפס" דרך setuid binary — השדה הזה סוגר את הדלת הזו במפורש.

**`readOnlyRootFilesystem: true`** — כל מערכת הקבצים הופכת לקריאה-בלבד,
חוץ מ-volumes מפורשים. הסיבה: אם תוקף כבר מריץ קוד, השלב הבא הטבעי הוא
לכתוב קובץ (backdoor, persistence). בלי מקום לכתוב אליו — זה נחסם.

**`capabilities: drop: ["ALL"]`** — Linux מפרק את "root" ל-כ-40 הרשאות
עדינות בנפרד (למשל יצירת raw sockets לציתות רשת, פעולות ניהוליות רחבות).
גם container לא-root מקבל כברירת מחדל סט בסיסי מהן. הורדת כולן משאירה
מינימום אמיתי — האפליקציה שלנו לא צריכה אף אחת מהן.

### "אבל אם יש לתוקף גישה ל-`/reports`, הוא לא יכול לכתוב קובץ ולהריץ אותו?"

שאלה שעלתה בדיון ונבדקה בפועל בקוד (לא רק תיאורטית): חיפשנו את כל
הקריאות ל-subprocess בכל הפרויקט (`grep -rn "subprocess\|Popen"`), ואף
אחת מהן לא נוגעת ב-`/reports` — ה-scanner מריץ רק את semgrep/gitleaks/trivy
עצמם (דרך PATH, לא מהתיקייה), ו-`/reports` נכתב אך ורק כטקסט
(`Path.write_text`), לעולם לא מבוצע.

**אבל** — "לכתוב ולהריץ קובץ שני" ממילא לא נותן לתוקף שום דבר חדש, כי
כדי לכתוב את הקובץ מלכתחילה הוא **כבר** מריץ קוד. הסיכון האמיתי הוא לא
הרצה חוזרת, אלא:
1. **Persistence** — ה-Job עצמו אפמרי ונעלם, אבל קובץ על ה-PVC נשאר.
2. **זיהום צולב** — אותו PVC גם מ-mount-ל ה-`review-api` (backend)!
   קובץ זדוני שיושב שם עלול מאוחר יותר להיפגע ע"י רכיב **אחר, מהימן
   יותר**, שיטפל בו בטעות.

**הפתרון שהוספנו:** StorageClass ייעודי (`storageclass-reports.yaml`) עם
`mountOptions: [noexec, nosuid, nodev]` — חוסם הרצת קבצים מה-volume הזה
ברמת ה-kernel, בלי קשר להרשאות הקובץ עצמו. **חשוב לציין למרצה בכנות:**
זה **לא מאומת** עדיין מול ה-`local-path-provisioner` האמיתי של k3s
(כלי פשוט יחסית מבוסס hostPath) — יכול להיות שהוא לא מכבד mountOptions
מותאמים אישית כמו storage driver "מלא". זה מתועד כפער פתוח, לא כפתרון
מוכח.
