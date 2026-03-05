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


# What is a Deadlock in SQL Server?

A Deadlock is a situation in a database where two or more transactions are waiting for one another to release locks on resources (like rows, pages, or tables). Because each transaction is holding a lock that the other needs, they end up in an infinite waiting state—a circular dependency. Neither transaction can finish.

## The Classic Analogy:

Imagine two people, Alice and Bob, who both need a Pen and a Paper to write a letter.

1. Alice grabs the Pen.
2. Bob grabs the Paper.
3. Alice asks for the Paper (but Bob has it).
4. Bob asks for the Pen (but Alice has it).

Neither will let go of what they currently hold until they get the other item. They are deadlocked.

## How SQL Server Handles Deadlocks

SQL Server has a background process called the **Deadlock Monitor**. It constantly checks for these circular waiting states. When it detects a deadlock, it automatically intervenes to resolve it:

* It chooses one of the transactions to be the **"Deadlock Victim"** (usually the one that is least "expensive" to roll back).
* It kills the victim's transaction, rolls back its changes, and throws **Error 1205**: "Transaction (Process ID X) was deadlocked on lock resources with another process and has been chosen as the deadlock victim. Rerun the transaction."
* The other transaction is then free to acquire the locks it needs and complete successfully.

## A Concrete Example in SQL

Let's say we have two tables: `Orders` and `Inventory`.

### Transaction A (Window 1):

```sql
BEGIN TRAN;
-- Step 1: Locks the Orders table
UPDATE Orders SET Status = 'Processing' WHERE OrderID = 1;

-- Simulating a slight delay
WAITFOR DELAY '00:00:05'; 

-- Step 2: Tries to lock the Inventory table (but Transaction B has it)
UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductID = 100;
COMMIT;
```

### Transaction B (Window 2): 
Run at the exact same time as A.

```sql
BEGIN TRAN;
-- Step 1: Locks the Inventory table
UPDATE Inventory SET Quantity = Quantity - 1 WHERE ProductID = 100;

-- Simulating a slight delay
WAITFOR DELAY '00:00:05';

-- Step 2: Tries to lock the Orders table (but Transaction A has it)
UPDATE Orders SET Status = 'Processing' WHERE OrderID = 1;
COMMIT;
```

**Result:** One of these transactions will succeed, and the other will fail with Error 1205.

---

# How to Solve and Prevent Deadlocks

You cannot 100% eliminate deadlocks in a highly concurrent system, but you can minimize them to a point where they are negligible. Here are the best practices:

## 1. Access Objects in the Same Order (The Golden Rule) ⭐
If all transactions request locks in the exact same sequence, deadlocks cannot happen.

**Fixing the example above:** Make sure both transactions update `Orders` first, then update `Inventory`. Transaction B will simply wait behind Transaction A nicely without causing a circle.

## 2. Keep Transactions Short and Fast
The longer a transaction runs, the longer it holds locks.

* ❌ Do not wait for user input in the middle of a transaction.
* ❌ Do all heavy calculations or data gathering *before* opening the `BEGIN TRAN`.
* ✅ Commit as soon as the database writes are done.

## 3. Add Proper Indexes
If a query doesn't have a good index, SQL Server might have to scan the entire table (**Table Scan**/ **Clustered Index Scan**). To do this, it might lock the whole table or many irrelevant rows. 

**Proper indexing ensures that:**
* Queries find their data instantly
* Only the exact rows needed are locked (**Row Locks** instead of Table Locks)

## 4. Use the Right Isolation Level (RCSI)
If your deadlocks are caused by readers blocking writers (or writers blocking readers), consider turning on **Read Committed Snapshot Isolation (RCSI)**.

| Isolation Level | Behavior |
|----------------|----------|
| `READ COMMITTED` (default) | Uses shared locks for reading → readers can block writers |
| `READ COMMITTED SNAPSHOT` | Uses **Row Versioning** in `tempdb` → readers don't block writers |

**Result:** Readers read the last committed version of the row without locking it, meaning readers don't block writers, and writers don't block readers.

```sql
ALTER DATABASE YourDatabase SET READ_COMMITTED_SNAPSHOT ON;
```

> ⚠️ **Note:** Enabling RCSI increases `tempdb` usage. Monitor your tempdb performance after enabling.

## 5. Catch and Retry in Application Code
Since deadlocks are a concurrency issue and not a logical error, the standard engineering practice is to catch **Error 1205** in your backend code (C#, Java, Python, etc.) and simply wait a few milliseconds and retry the transaction 3-5 times before throwing an error to the user.

### C# Example (Pseudo-code)
```csharp
int retries = 3;
while (retries > 0) {
    try {
        ExecuteDatabaseTransaction();
        break; // Success!
    } 
    catch (SqlException ex) when (ex.Number == 1205) { // 1205 is the Deadlock error code
        retries--;
        Thread.Sleep(100); // Wait a bit before retrying
    }
}
```

### Python Example (Pseudo-code)
```python
import time
import pyodbc

retries = 3
while retries > 0:
    try:
        execute_database_transaction()
        break  # Success!
    except pyodbc.Error as ex:
        if "1205" in str(ex):  # Deadlock error
            retries -= 1
            time.sleep(0.1)  # Wait 100ms before retrying
        else:
            raise  # Re-raise if it's a different error
```

---

## Quick Reference Checklist ✅

| Strategy | Impact | Effort |
|----------|--------|--------|
| Access objects in same order | 🔴 High | 🟢 Low |
| Keep transactions short | 🔴 High | 🟢 Low |
| Add proper indexes | 🟡 Medium | 🟡 Medium |
| Enable RCSI | 🟡 Medium | 🟢 Low |
| Implement retry logic | 🟢 Low (mitigation) | 🟡 Medium |

> 💡 **Pro Tip:** Use SQL Server Profiler or Extended Events to capture deadlock graphs (`-T1222` or `-T1204` trace flags) to analyze exactly which resources and queries are involved. This helps you target your fixes precisely.


