# الدليل الشامل لتصميم وبناء APIs متقدمة (بواسطة C# و ASP.NET Core)

هذا الملف يحتوي على ملخص لأهم المفاهيم المتقدمة في بناء الـ APIs مع أمثلة عملية قابلة للاستخدام في بيئة العمل (Production-Ready).

---

## 1. استراتيجيات تقسيم البيانات (API Pagination)
الهدف هو إرجاع البيانات على أجزاء لتحسين أداء النظام وقاعدة البيانات.

* **Offset-based (`Skip` & `Take`):** * **المميزات:** سهلة التنفيذ، وتسمح بالانتقال لصفحة معينة.
  * **العيوب:** أداؤها يقل جداً مع البيانات الكبيرة لأن قاعدة البيانات تقرأ كل الصفوف التي يتم تخطيها.
* **Keyset-based:** * **المميزات:** سريعة جداً؛ تعتمد على عمود فريد في قاعدة البيانات مثل الـ `Id` (مثال: `Where(x => x.Id > lastId)`).
  * **العيوب:** لا تدعم الانتقال لصفحة عشوائية، وتتطلب عموداً مرتباً.
* **Cursor-based:** * **المميزات:** هي الأسرع والأكثر أماناً. نفس فكرة الـ Keyset، لكن يتم تشفير القيمة (مثلاً إلى Base64) لإخفاء تفاصيل قاعدة البيانات عن الواجهة الأمامية (Frontend)، مما يعطي مرونة لتغيير طريقة الترتيب مستقبلاً دون كسر الـ Client.

### مثال عملي: Cursor-Based Pagination بـ Entity Framework Core

```csharp
using System;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using System.Collections.Generic;
using Microsoft.EntityFrameworkCore;

// نموذج الرد (Response Wrapper)
public class CursorPagedResponse<T>
{
    public IEnumerable<T> Data { get; set; }
    public string? NextCursor { get; set; }
}

public async Task<CursorPagedResponse<Product>> GetProductsAsync(string? cursor, int limit = 10)
{
    int lastSeenId = 0;

    // 1. فك تشفير الـ Cursor لو موجود
    if (!string.IsNullOrWhiteSpace(cursor))
    {
        try
        {
            var base64Bytes = Convert.FromBase64String(cursor);
            var decodedString = Encoding.UTF8.GetString(base64Bytes);
            int.TryParse(decodedString, out lastSeenId);
        }
        catch (FormatException) { lastSeenId = 0; }
    }

    // 2. جلب البيانات بناءً على الـ Index (سريع جداً O(1) Seek)
    var products = await _context.Products
        .AsNoTracking()
        .Where(p => p.Id > lastSeenId)
        .OrderBy(p => p.Id)
        .Take(limit)
        .ToListAsync();

    // 3. إنشاء الـ Cursor للصفحة التالية
    string? nextCursor = null;
    if (products.Count == limit)
    {
        var lastItem = products.Last();
        var plainTextBytes = Encoding.UTF8.GetBytes(lastItem.Id.ToString());
        nextCursor = Convert.ToBase64String(plainTextBytes);
    }

    return new CursorPagedResponse<Product>
    {
        Data = products,
        NextCursor = nextCursor
    };
}

```

## 2. الطلبات الشرطية (HTTP Conditional Requests)
تُستخدم لتقليل الحمل على السيرفر وتوفير استهلاك الباندويث بإخبار المتصفح (أو الـ Client) باستخدام النسخة المخزنة عنده (Cache) إذا لم تتغير البيانات.

* تعتمد على إرسال هيدر `ETag` (بصمة محتوى الملف) أو `Last-Modified` (تاريخ التعديل).
* في الطلب التالي، يرسل العميل هيدر `If-None-Match`. إذا تطابق الـ ETag، يرد السيرفر بكود `304 Not Modified` بدون إرسال جسم البيانات (Body).

### مثال عملي: ETag باستخدام ActionFilter في ASP.NET Core

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Net.Http.Headers;
using System.Security.Cryptography;
using System.Text;
using System.Text.Json;

public class ETagFilterAttribute : ActionFilterAttribute
{
    public override void OnActionExecuted(ActionExecutedContext context)
    {
        var request = context.HttpContext.Request;
        var response = context.HttpContext.Response;

        // التأكد من أن الطلب GET وناجح ويحتوي على بيانات
        if (request.Method == HttpMethods.Get && 
            response.StatusCode == 200 && 
            context.Result is ObjectResult objResult)
        {
            // إنشاء بصمة (Hash) لمحتوى البيانات
            var content = JsonSerializer.Serialize(objResult.Value);
            var eTag = $"\"{GenerateHash(content)}\""; 

            // التحقق من هيدر المتصفح
            if (request.Headers.TryGetValue(HeaderNames.IfNoneMatch, out var clientETag) && clientETag == eTag)
            {
                // الداتا ماتغيرتش، ارجع 304
                context.Result = new StatusCodeResult(StatusCodes.Status304NotModified);
            }
            else
            {
                // إرسال الداتا مع البصمة الجديدة
                response.Headers[HeaderNames.ETag] = eTag;
            }
        }
    }

    private string GenerateHash(string input)
    {
        using var md5 = MD5.Create(); // MD5 سريع ومناسب للـ ETags
        var hash = md5.ComputeHash(Encoding.UTF8.GetBytes(input));
        return Convert.ToHexString(hash);
    }
}

// طريقة الاستخدام:
// [HttpGet]
// [ETagFilter] 
// public IActionResult GetData() { ... }
```
## 4. الحماية من التكرار (Idempotency Keys)
آلية حيوية لمنع تكرار العمليات الحساسة (مثل عمليات الدفع - Payments) إذا قام المستخدم بالضغط على الزر مرتين أو حدثت مشكلة بالشبكة (Timeout).

* **الفكرة:** يرسل العميل كوداً فريداً `Idempotency-Key` مع الطلب.
* **التنفيذ:** السيرفر يفحص الكود؛ إذا كان جديداً ينفذ العملية ويحفظ النتيجة. إذا كان موجوداً مسبقاً، يرجع النتيجة المحفوظة فوراً دون إعادة تنفيذ العملية (دون خصم الرصيد مرة أخرى).

### مثال عملي: Idempotency باستخدام IDistributedCache في ASP.NET Core

```csharp
using Microsoft.AspNetCore.Mvc;
using Microsoft.AspNetCore.Mvc.Filters;
using Microsoft.Extensions.Caching.Distributed;
using System;
using System.Text.Json;
using System.Threading.Tasks;

[AttributeUsage(AttributeTargets.Method)]
public class IdempotentAttribute : Attribute, IAsyncActionFilter
{
    private const string HeaderKeyName = "Idempotency-Key";
    private readonly int _expireHours;

    public IdempotentAttribute(int expireHours = 24)
    {
        _expireHours = expireHours;
    }

    public async Task OnActionExecutionAsync(ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var request = context.HttpContext.Request;

        // 1. التأكد من وجود المفتاح في الهيدر
        if (!request.Headers.TryGetValue(HeaderKeyName, out var idempotencyKey))
        {
            context.Result = new BadRequestObjectResult($"Missing {HeaderKeyName} header.");
            return;
        }

        var cache = context.HttpContext.RequestServices.GetRequiredService<IDistributedCache>();
        var cacheKey = $"Idempotency_{idempotencyKey}"; 

        // 2. التحقق مما إذا كان الطلب قد تم تنفيذه مسبقاً
        var cachedData = await cache.GetStringAsync(cacheKey);
        if (!string.IsNullOrEmpty(cachedData))
        {
            // الطلب مكرر: إرجاع نفس النتيجة المحفوظة
            var savedResponse = JsonSerializer.Deserialize<CachedResponse>(cachedData);
            context.Result = new ObjectResult(savedResponse.Value) 
            { 
                StatusCode = savedResponse.StatusCode 
            };
            return;
        }

        // 3. طلب جديد: تنفيذ العملية (Controller Action)
        var executedContext = await next();

        // 4. حفظ النتيجة في الكاش للطلبات المستقبلية المتطابقة
        if (executedContext.Exception == null && executedContext.Result is ObjectResult objResult)
        {
            var responseToCache = new CachedResponse
            {
                StatusCode = objResult.StatusCode ?? 200,
                Value = objResult.Value
            };

            var cacheOptions = new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = TimeSpan.FromHours(_expireHours)
            };

            await cache.SetStringAsync(cacheKey, JsonSerializer.Serialize(responseToCache), cacheOptions);
        }
    }

    private class CachedResponse
    {
        public int StatusCode { get; set; }
        public object Value { get; set; }
    }
}

// طريقة الاستخدام في الـ Controller:
// [HttpPost("charge")]
// [Idempotent(expireHours: 24)]
// public async Task<IActionResult> ProcessPayment([FromBody] PaymentRequest request) 
// { 
//     // يتم تنفيذ كود الدفع هنا
//     return Ok(new { Status = "Success", Message = "Payment Processed" });
// }
