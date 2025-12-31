# 🚀 Caching in ASP.NET Core – Complete Guide

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

## 🔁 Cache-Aside Pattern (Most Common)

### Flow

1. Check cache
2. If miss → read DB
3. Store result in cache
4. Return response

### Update Handling

```csharp
await UpdateProductInDatabase(product);
_cache.Remove($"product_{product.Id}");
```

---

## ⏱️ Cache Expiration Strategies

### 1️⃣ Absolute Expiration 

* الكاش ينتهي بعد مدة محددة مهما حصل.
* بعد المدة → Cache Miss → Database

#### مثال كود

```csharp
_cache.Set(
    "product_1",
    product,
    new MemoryCacheEntryOptions
    {
        AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5)
    });
```

### 2️⃣ Sliding Expiration 

* مدة الكاش تتجدد عند كل استخدام.
* إذا لم يتم الوصول → تنتهي بعد المدة المحددة.

#### مثال كود

```csharp
_cache.Set(
    "user_session_123",
    sessionData,
    new MemoryCacheEntryOptions
    {
        SlidingExpiration = TimeSpan.FromMinutes(20)
    });
```

### الفرق بين Absolute و Sliding

| النقطة    | Absolute Expiration | Sliding Expiration           |
| --------- | ------------------- | ---------------------------- |
| مدة الكاش | ثابتة               | تتجدد عند الاستخدام          |
| مناسب لـ  | بيانات ثابتة        | جلسات أو بيانات متغيرة بكثرة |
| بعد المدة | ينتهي               | ينتهي إذا لم يتم استخدامه    |

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

* AbsoluteExpiration → ينتهي بعد وقت محدد مهما حصل
* SlidingExpiration → ينتهي بعد فترة من آخر استخدام

---

## 🔒 Concurrency & Thread Safety

* IMemoryCache is thread-safe
* Distributed cache handles concurrency internally
* Async code must avoid blocking locks

---

## 💥 What is Cache Stampede?

When:

* Cache expires
* Many requests arrive simultaneously
* All hit the database

❌ Result: Database overload

---

## 🛡️ Cache Stampede Protection (Per-Key Async Lock)

### Concept

* Each cache key has its own lock
* Only ONE request fetches from DB
* Others wait asynchronously

---

## 🔑 SemaphoreSlim & Per-Key Async Locks (Simple Explanation)

> **This section explains these two concepts in very simple wordsز**

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

### Summary

* **Use TryGetValue**: large projects, hybrid cache, Redis, high concurrency, cache stampede protection
* **Use GetOrCreateAsync**: simple projects, single server, small apps

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

## 🔥 Redis vs IMemoryCache Comparison

| Feature     | IMemoryCache | Redis      |
| ----------- | ------------ | ---------- |
| Speed       | Fastest      | Fast       |
| Shared      | ❌ No         | ✅ Yes      |
| Persistence | ❌ No         | ✅ Optional |
| Scalability | Low          | High       |

---

## 🧠 Hybrid Cache Strategy (Best Practice)

```
Request
 ↓
Memory Cache
 ↓
Redis
 ↓
Database
```

✔ Fast
✔ Scalable
✔ Production-ready

---


> We use hybrid caching with IMemoryCache and Redis, apply cache-aside pattern, handle expiration properly, and prevent cache stampede using per-key async locks with SemaphoreSlim to protect the database under high concurrency.

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

