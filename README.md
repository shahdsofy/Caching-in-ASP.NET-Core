# 🚀 Caching in ASP.NET Core 

This README explains **everything related to caching** in ASP.NET Core with **clear concepts, diagrams, and real code examples**.

---

## 📌 What is Caching?

Caching is the process of **storing frequently accessed data in fast storage (memory)** to avoid repeated expensive operations such as database calls.

### 🎯 Why Caching?

* Improve performance ⚡
* Reduce database load
* Improve scalability
* Lower response time

---

## 🧠 Types of Caching in ASP.NET Core

### 1️⃣ In-Memory Cache (IMemoryCache)

* Stored in application RAM
* Fastest caching option
* Per-server (NOT shared)

### 2️⃣ Distributed Cache (Redis)

* Shared across multiple servers
* Network-based
* Slightly slower than memory

### 3️⃣ Hybrid Cache (Memory + Redis)

* Combines speed + scalability
* Best practice for production

---

## ⚡ IMemoryCache Example

### Program.cs

```csharp
builder.Services.AddMemoryCache();
```

### Usage

```csharp
public class ProductService
{
    private readonly IMemoryCache _cache;

    public ProductService(IMemoryCache cache)
    {
        _cache = cache;
    }

    public async Task<Product> GetProductAsync(int id)
    {
        return await _cache.GetOrCreateAsync($"product_{id}", async entry =>
        {
            entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
            return await GetProductFromDatabase(id);
        });
    }
}
```

> If the data is not found in the in-memory cache, the application fetches it from the database and automatically stores it in RAM with an expiration time. Subsequent requests are served directly from memory.

---

## 🌐 Distributed Cache (Redis)

### Program.cs

```csharp
builder.Services.AddStackExchangeRedisCache(options =>
{
    options.Configuration = "localhost:6379";
    options.InstanceName = "App:";
});
```

### Usage

```csharp
public async Task<Product> GetProductAsync(int id)
{
    var key = $"product_{id}";
    var cached = await _cache.GetStringAsync(key);

    if (cached != null)
        return JsonSerializer.Deserialize<Product>(cached);

    var product = await GetProductFromDatabase(id);

    await _cache.SetStringAsync(
        key,
        JsonSerializer.Serialize(product),
        new DistributedCacheEntryOptions
        {
            AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(10)
        });

    return product;
}
```

---

## 🔹 Hybrid Caching 

### Packages to Install

```bash
dotnet add package Microsoft.Extensions.Caching.Hybird

```

### Example Hybrid Code

```csharp
public async Task<Product> GetProductAsync(int id)
{
    string memKey = $"product_{id}";
    string redisKey = $"product_{id}";

    if (_memoryCache.TryGetValue(memKey, out Product product))
        return product;

    var redisData = await _redisCache.GetStringAsync(redisKey);
    if (redisData != null)
    {
        product = JsonSerializer.Deserialize<Product>(redisData);
        _memoryCache.Set(memKey, product, TimeSpan.FromMinutes(1));
        return product;
    }

    product = await GetProductFromDatabase(id);

    _memoryCache.Set(memKey, product, TimeSpan.FromMinutes(1));
    await _redisCache.SetStringAsync(redisKey, JsonSerializer.Serialize(product));

    return product;
}
```


## 🔹 Two-Level Caching (L1 / L2)

* **L1:** Fast in-memory cache (IMemoryCache)
* **L2:** Distributed cache (Redis, SQL Server, etc.)

```
Request
 ↓
L1: Memory Cache ✅ → Hit → return
 ↓
L2: Redis Cache ❌ → Miss
 ↓
Database
 ↓
Store in L2 & L1
 ↓
Return Data
```



## 🔹 Tag-Based Cache Invalidation

```csharp
_cache.Set("product_1", product, new MemoryCacheEntryOptions
{
    Tags = new[] { "Products" },
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
});

// مسح كل منتجات مرة واحدة
_cache.RemoveByTag("Products");
```



## 🔹 Configurable Serialization

```csharp
var data = JsonSerializer.Serialize(product);
await _redisCache.SetStringAsync("product_1", data);
```






---

## 🔁 Cache-Aside Pattern (Most Common)

### الفكرة الأساسية

Cache-Aside هو نمط يتم فيه تحميل البيانات إلى الكاش عند الطلب فقط. خطواته:

1. **Check Cache** → إذا موجودة ترجع مباشرة
2. **Cache Miss** → Load from Database
3. **Store in Cache** → حفظ الداتا في الكاش
4. **Return Data** → إرجاع النتيجة للـ Request

### الرسم الذهني

```
Request
   ↓
Memory Cache
   ↓
✔ Hit → return
❌ Miss
   ↓
Database
   ↓
Cache
   ↓
Return Data
```

### مثال كود

```csharp
public async Task<Product> GetProductAsync(int id)
{
    if (!_cache.TryGetValue($"product_{id}", out Product product))
    {
        product = await GetProductFromDatabase(id);
        _cache.Set($"product_{id}", product, TimeSpan.FromMinutes(5));
    }
    return product;
}
```

### مميزات

* بسيط وسهل للتطبيق
* تحسين الأداء للطلبات التالية

### عيوب

* يحتاج Expiration Strategy واضحة
* إذا عدة طلبات جت في نفس الوقت → ممكن يحصل **Cache Stampede**

---

## ⏱️ Cache Expiration Strategies

### 1️⃣ Absolute Expiration

* ينتهي بعد مدة محددة مهما حصل

```csharp
_cache.Set("product_1", product, new MemoryCacheEntryOptions
{
    AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
});
```

### 2️⃣ Sliding Expiration

* تتجدد عند كل استخدام

```csharp
_cache.Set("user_session_123", sessionData, new MemoryCacheEntryOptions
{
    SlidingExpiration = TimeSpan.FromMinutes(20)
});
```

### الفرق بين Absolute و Sliding

| النقطة    | Absolute Expiration | Sliding Expiration        |
| --------- | ------------------- | ------------------------- |
| مدة الكاش | ثابتة               | تتجدد عند الاستخدام       |
| مناسب لـ  | بيانات ثابتة        | جلسات أو بيانات متغيرة    |
| بعد المدة | ينتهي               | ينتهي إذا لم يتم استخدامه |

### مثال عملي في GetOrCreateAsync

```csharp
return await _cache.GetOrCreateAsync("product_1", entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    // أو
    // entry.SlidingExpiration = TimeSpan.FromMinutes(10);
    return GetProductFromDatabase(1);
});
```

---

## 🔒 Concurrency & Thread Safety

* IMemoryCache is thread-safe
* Distributed cache handles concurrency internally
* Async code must avoid blocking locks

---

## 💥 Cache Stampede Protection

* مشكلة: كثير من الطلبات تجي على Key فاضي → Database overload
* الحل: Per-Key Async Lock + SemaphoreSlim

```csharp
var semaphore = CacheLocks.Get(cacheKey);
await semaphore.WaitAsync();
try
{
    if (!_memoryCache.TryGetValue(cacheKey, out product))
    {
        product = await GetFromRedisOrDatabase(cacheKey);
        _memoryCache.Set(cacheKey, product, TimeSpan.FromMinutes(5));
    }
}
finally
{
    semaphore.Release();
}
```

---

## 🔑 SemaphoreSlim & Per-Key Async Locks (Simple Explanation)

> **This section explains these two concepts in very simple words.**

### The Problem

* Cache empty
* 100 requests for the same key (`product_1`)
* Without protection, all hit the database → overload

### SemaphoreSlim

Think of `SemaphoreSlim` as a **door with rules** 🚪

```csharp
new SemaphoreSlim(1, 1);
```

* Only **ONE request** enters at a time
* Others wait in line
* Async-safe (works with `async/await`)

### Per-Key Async Lock

* Each cache key has its own lock
* `product_1` has one semaphore
* `product_2` has another semaphore
* Requests for different data run in parallel, requests for same data wait

### Visual Explanation

```
Requests for product_1
   ↓
Semaphore (product_1)
   ↓
ONE request enters
   ↓
Database
   ↓
Cache filled
   ↓
All requests read from cache
```

### Code Example

```csharp
var semaphore = CacheLocks.Get(redisKey);
await semaphore.WaitAsync();
try
{
    // double check cache
    // load from database ONLY ONCE
}
finally
{
    semaphore.Release();
}
```

### Double Check

* Prevents duplicate DB calls if another request filled the cache while waiting


> We prevent cache stampede using per-key async locks implemented with SemaphoreSlim. This ensures only one request fetches data from the database per cache key, while others wait asynchronously until the cache is populated.

---

---
## 🔹 TryGetValue vs GetOrCreateAsync

### TryGetValue

* Checks if data exists
* Manual handling if cache miss
* Longer code but full control

```csharp
if (_cache.TryGetValue("key", out Product product)) return product;
product = await GetProductFromDatabase(id);
_cache.Set("key", product, TimeSpan.FromMinutes(5));
```

### GetOrCreateAsync

* Checks cache
* If missing → fetch, cache, return automatically
* Shorter code, less control

```csharp
return await _cache.GetOrCreateAsync("key", async entry =>
{
    entry.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await GetProductFromDatabase(id);
});
```

### Comparison Table

| Feature             | TryGetValue | GetOrCreateAsync |
| ------------------- | ----------- | ---------------- |
| Code                | Longer      | Shorter          |
| Storage             | Manual      | Automatic        |
| Control             | High        | Limited          |
| Stampede Protection | Easy        | Hard             |
| Hybrid Cache        | Easy        | Hard             |
| Simple Projects     | ❌           | ✔                |
| Complex Projects    | ✔           | ❌                |



* TryGetValue: manual, full control, مناسب للمشاريع الكبيرة و Hybrid cache
* GetOrCreateAsync: أوتوماتيك، مناسب للبسيط

---

## 🔹 Redis vs IMemoryCache Comparison

| Feature     | IMemoryCache | Redis      |
| ----------- | ------------ | ---------- |
| Speed       | Fastest      | Fast       |
| Shared      | ❌ No         | ✅ Yes      |
| Persistence | ❌ No         | ✅ Optional |
| Scalability | Low          | High       |


---

## 🧩 Diagram – Cache Stampede Protection

```
Requests
   ↓
Memory Cache ❌
   ↓
Redis ❌
   ↓
Semaphore (per-key)
   ↓
ONE request
   ↓
Database
   ↓
Cache Filled
   ↓
All requests served from cache
```

---
---

## ✅ Best Practices

* Short memory TTL
* Longer Redis TTL
* Invalidate cache on updates
* Avoid global locks
* Monitor cache hit ratio

---

## 📌 Conclusion

Caching is a critical performance optimization technique. A well-designed caching strategy improves scalability, reduces load, and ensures system reliability.

---

