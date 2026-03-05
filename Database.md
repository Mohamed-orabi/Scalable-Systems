# مستويات العزل في SQL Server والمشاكل التي تمنعها (SQL Isolation Levels & Anomalies)

تتناول هذه الوثيقة موضوع "مستويات العزل" (SQL Isolation Levels) والمشاكل أو التضاربات (Anomalies) التي تحدث عند تنفيذ عدة معاملات (Transactions) في نفس الوقت، وكيف تقوم هذه المستويات بحل تلك المشاكل.

---

## أولاً: المشاكل الناتجة عن التزامن (Concurrency Anomalies)

عندما تقوم قاعدة البيانات بتنفيذ أكثر من معاملة في وقت واحد، قد تحدث بعض التضاربات في قراءة البيانات، وتشمل:

1. **القراءة القذرة (Dirty Read):**
   - تحدث عندما تقوم المعاملة (A) بقراءة بيانات قامت معاملة أخرى (B) بتعديلها ولكنها **لم تقم بتأكيدها بعد (Uncommitted)**.
   - **المشكلة:** إذا فشلت المعاملة (B) وتم التراجع عنها (Rollback)، فإن المعاملة (A) تكون قد قرأت بيانات وهمية لم تعد موجودة فعلياً في قاعدة البيانات.

2. **القراءة غير القابلة للتكرار (Non-Repeatable Read):**
   - تحدث عندما تقوم المعاملة (A) بقراءة نفس الصف من البيانات مرتين متتاليتين، ولكنها تحصل على قيم مختلفة في كل مرة.
   - **السبب:** يحدث هذا لأن هناك معاملة أخرى (B) قامت بتعديل هذه البيانات وتأكيدها (Committed) في الوقت الفاصل بين القراءتين.

3. **القراءة الوهمية (Phantom Read):**
   - تحدث عندما تقوم المعاملة (A) بتنفيذ استعلام يُرجع مجموعة من الصفوف، ثم تعيد نفس الاستعلام وتجد أن هناك **صفوفاً جديدة قد ظهرت** (أو اختفت).
   - **السبب:** يعود ذلك إلى وجود معاملة أخرى (B) قامت بإدخال (Insert) أو حذف (Delete) صفوف تتطابق مع شرط البحث، وتم تأكيدها بين القراءتين.

---

## ثانياً: التجهيز للأمثلة العملية (Setup)

لتجربة هذه الأمثلة في **SQL Server**، افتح **نافذتين للاستعلام (Query Windows)** لمحاكاة التزامن بين معاملتين (المعاملة A و المعاملة B).

أولاً، قم بتنفيذ هذا الكود لتجهيز جدول الحسابات البنكية:

```sql
CREATE TABLE BankAccounts (
    ID INT PRIMARY KEY,
    AccountName VARCHAR(50),
    Balance DECIMAL(10,2)
);

INSERT INTO BankAccounts (ID, AccountName, Balance)
VALUES (1, 'Alice', 1000.00), (2, 'Bob', 500.00);
```

## ثالثاً: مستويات العزل والأمثلة العملية (Isolation Levels)
1. **القراءة غير المعتمدة (READ UNCOMMITTED)**
يسمح هذا المستوى بـ القراءة القذرة (Dirty Read).

المعاملة A (النافذة 1): تقوم بتحديث الرصيد ولا تؤكد العملية وتنتظر.
```sql
BEGIN TRAN;
UPDATE BankAccounts SET Balance = 9999.00 WHERE ID = 1;
-- محاكاة معالجة طويلة لمدة 10 ثوانٍ
WAITFOR DELAY '00:00:10';
ROLLBACK; -- تراجع عن التعديل
```
المعاملة B (النافذة 2): قم بتشغيل هذا الكود فوراً بعد تشغيل المعاملة A.
```
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
-- ستقرأ الرصيد الوهمي (9999) رغم أنه سيتم التراجع عنه لاحقاً!
SELECT * FROM BankAccounts WHERE ID = 1;
```

2. **القراءة المعتمدة (READ COMMITTED)**
هو المستوى الافتراضي في SQL Server. يمنع القراءة القذرة، لكنه يسمح بـ القراءة غير القابلة للتكرار (Non-Repeatable Read).
المعاملة A (النافذة 1): تقرأ الرصيد مرتين وبينهما فاصل زمني.
```
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
BEGIN TRAN;
-- القراءة الأولى (ستكون 1000)
SELECT Balance FROM BankAccounts WHERE ID = 1;

WAITFOR DELAY '00:00:10';

-- القراءة الثانية (ستكون 800! لقد تغيرت القيمة في نفس المعاملة)
SELECT Balance FROM BankAccounts WHERE ID = 1;
COMMIT;
```
المعاملة B (النافذة 2): قم بتشغيله أثناء انتظار المعاملة A.
```
BEGIN TRAN;
-- نقوم بتعديل الرصيد وتأكيده فوراً
UPDATE BankAccounts SET Balance = 800.00 WHERE ID = 1;
COMMIT;
```
3. **القراءة القابلة للتكرار (REPEATABLE READ)**
يمنع القراءة القذرة وغير القابلة للتكرار، ولكنه يسمح بـ القراءة الوهمية (Phantom Read).

المعاملة A (النافذة 1): تقرأ عدد الحسابات مرتين.
```
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
BEGIN TRAN;
-- القراءة الأولى: ترجع صفين (Alice و Bob)
SELECT * FROM BankAccounts WHERE Balance > 0;

WAITFOR DELAY '00:00:10';

-- القراءة الثانية: ترجع ثلاثة صفوف! ظهر حساب "Charlie" الوهمي.
SELECT * FROM BankAccounts WHERE Balance > 0;
COMMIT;
```
المعاملة B (النافذة 2): قم بتشغيله أثناء انتظار المعاملة A.
```
BEGIN TRAN;
-- إضافة حساب جديد يتطابق مع شرط البحث في المعاملة A
INSERT INTO BankAccounts (ID, AccountName, Balance) VALUES (3, 'Charlie', 1500.00);
COMMIT;
```
4. **التسلسل الكامل (SERIALIZABLE)**
أعلى مستوى أمان. يمنع كل المشاكل السابقة بما فيها القراءة الوهمية (Phantom Read)، عن طريق قفل نطاق البيانات (Range Locks).

المعاملة A (النافذة 1):
```
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
BEGIN TRAN;
-- القراءة الأولى: ترجع 3 صفوف
SELECT * FROM BankAccounts WHERE Balance > 0;

WAITFOR DELAY '00:00:10';

-- القراءة الثانية: سترجع 3 صفوف أيضاً! لن يتغير شيء.
SELECT * FROM BankAccounts WHERE Balance > 0;
COMMIT;
```

المعاملة B (النافذة 2): قم بتشغيله أثناء انتظار المعاملة A.

```
BEGIN TRAN;
-- هذا الاستعلام سيتم "حجبه" (Blocked) ولن يُنفذ
-- سيظل يدور في حالة انتظار حتى تنتهي المعاملة A وتُفلت الأقفال
INSERT INTO BankAccounts (ID, AccountName, Balance) VALUES (4, 'Diana', 2000.00);
COMMIT;
```
## جدول مقارنة مستويات العزل والمشاكل التي تمنعها

| مستوى العزل (Isolation Level) | القراءة القذرة (Dirty Read) | القراءة غير القابلة للتكرار (Non-Repeatable Read) | القراءة الوهمية (Phantom Read) | الأداء (Performance) |
| :--- | :---: | :---: | :---: | :---: |
| **Read Uncommitted** | ❌ مسموح | ❌ مسموح | ❌ مسموح | 🚀 الأسرع (أعلى أداء) |
| **Read Committed** | ✅ محمي | ❌ مسموح | ❌ مسموح | ⚡ سريع جداً |
| **Repeatable Read** | ✅ محمي | ✅ محمي | ❌ مسموح | 🐢 أبطأ (بسبب الأقفال) |
| **Serializable** | ✅ محمي | ✅ محمي | ✅ محمي | 🛑 الأبطأ (أقفال صارمة) |

*(ملاحظة: علامة ✅ تعني أن المستوى يمنع المشكلة، وعلامة ❌ تعني أن المشكلة قد تحدث).*
