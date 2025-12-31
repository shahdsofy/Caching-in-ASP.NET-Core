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

✔ Ultra-fast
❌ Not shared between servers

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

✔ Shared across servers
✔ Scalable

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

✔ Prevents stale data

---

## ⏱️ Cache Expiration Strategies

### Absolute Expiration

```csharp
_cache.Set("rates", data, TimeSpan.FromMinutes(10));
```

### Sliding Expiration

```csharp
_cache.Set(
    "session",
    data,
    new MemoryCacheEntryOptions
    {
        SlidingExpiration = TimeSpan.FromMinutes(20)
    });
```

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

> **This section explains these two concepts in very simple words.**

---

## ❓ The Problem (Why We Need Them)

Imagine:

* Cache is empty ❌
* 100 requests arrive at the same time
* All requests ask for the SAME data (`product_1`)

❌ Without protection:

```
Request 1 → Database
Request 2 → Database
Request 3 → Database
Request 4 → Database
...
```

💥 Database overload

---

## 🛡️ What is Cache Stampede?

**Cache Stampede** happens when:

* Cache expires or is empty
* Many requests try to load the same data
* All hit the database at once

---

## 🔒 What is SemaphoreSlim?

Think of `SemaphoreSlim` as a **door with rules** 🚪

```csharp
new SemaphoreSlim(1, 1);
```

### What does this mean?

* Only **ONE request** can enter
* Other requests **wait in line**
* When the first finishes, the next enters

### Why SemaphoreSlim?

* Async-safe (works with `async/await`)
* Does NOT block threads
* Perfect for ASP.NET Core

---

## 🔑 What is Per-Key Async Lock?

Per-key async lock means:

> **Each cache key has its own lock**

### Example:

* `product_1` → has its own lock
* `product_2` → has a different lock

✔ Requests for different data run in parallel
✔ Only SAME data is locked

---

## 🧠 Visual Explanation 

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

---

## 🧩 How It Works in Code

### Step 1: Lock Manager

```csharp
public static class CacheLocks
{
    private static readonly ConcurrentDictionary<string, SemaphoreSlim> _locks = new();

    public static SemaphoreSlim Get(string key)
    {
        return _locks.GetOrAdd(key, _ => new SemaphoreSlim(1, 1));
    }
}
```

✔ One semaphore per cache key
✔ Thread-safe

---

### Step 2: Using the Lock

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

---

## 🔁 Why Double Check Cache After Lock?

Because:

* Another request may have already filled the cache
* So database call is no longer needed

✔ Prevents duplicate DB calls

---


> We prevent cache stampede using per-key async locks implemented with SemaphoreSlim. This ensures only one request fetches data from the database per cache key, while others wait asynchronously until the cache is populated.

---

## ⚠️ Important Notes

* SemaphoreSlim is per-server
* Redis handles cross-server sharing
* Never use `lock` keyword with async code
* Never use a global lock for all cache keys

---

```csharp
new SemaphoreSlim(1,1);
```

* Allows only one request at a time
* Async-safe
* Does NOT block threads



---



---

## 🧪 Diagram – Cache Stampede Protection

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

