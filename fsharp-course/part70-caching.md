# Part 70 - การแคช (Caching)

## บทนำ (Introduction)

Caching คือการจัดเก็บข้อมูลชั่วคราวเพื่อให้การเข้าถึงครั้งต่อไปเร็วขึ้น ลด latency และลด load บน database

**ทำไมต้องใช้ Cache?**
- ลด database queries ที่ซ้ำกัน
- เพิ่ม response time ของ API
- ลด load บน downstream services
- รองรับ traffic spike ได้ดีขึ้น

---

## 1. In-Memory Caching

```fsharp
// InMemoryCache.fs
module InMemoryCache

open System
open System.Collections.Concurrent

// ========================================
// 1.1 Simple dictionary-based cache
// ========================================

type CacheEntry<'T> = {
    Value: 'T
    ExpiresAt: DateTime option
}

type SimpleCache<'TKey, 'TValue when 'TKey: equality>() =
    let store = ConcurrentDictionary<'TKey, CacheEntry<'TValue>>()

    member _.Set (key: 'TKey) (value: 'TValue) (ttl: TimeSpan option) =
        let entry = {
            Value = value
            ExpiresAt = ttl |> Option.map (fun t -> DateTime.UtcNow.Add(t))
        }
        store.[key] <- entry

    member _.Get (key: 'TKey) : 'TValue option =
        match store.TryGetValue(key) with
        | false, _ -> None
        | true, entry ->
            match entry.ExpiresAt with
            | Some expiresAt when DateTime.UtcNow > expiresAt ->
                store.TryRemove(key) |> ignore
                None
            | _ ->
                Some entry.Value

    member this.GetOrSet (key: 'TKey) (factory: unit -> 'TValue) (ttl: TimeSpan option) =
        match this.Get key with
        | Some value -> value
        | None ->
            let value = factory()
            this.Set key value ttl
            value

    member _.Remove (key: 'TKey) =
        store.TryRemove(key) |> ignore

    member _.Clear () =
        store.Clear()

    member _.Count = store.Count

// ========================================
// 1.2 Generic F# memoization
// ========================================

// Basic memoize
let memoize (f: 'a -> 'b) : 'a -> 'b =
    let cache = ConcurrentDictionary<'a, 'b>()
    fun arg ->
        cache.GetOrAdd(arg, fun k -> f k)

// Memoize with TTL
let memoizeWithTtl (ttl: TimeSpan) (f: 'a -> 'b) : 'a -> 'b =
    let cache = ConcurrentDictionary<'a, CacheEntry<'b>>()
    fun arg ->
        let now = DateTime.UtcNow
        match cache.TryGetValue(arg) with
        | true, entry when entry.ExpiresAt |> Option.forall (fun exp -> now < exp) ->
            entry.Value
        | _ ->
            let value = f arg
            cache.[arg] <- { Value = value; ExpiresAt = Some (now.Add(ttl)) }
            value

// Async memoize
let memoizeAsync (f: 'a -> Async<'b>) : 'a -> Async<'b> =
    let cache = ConcurrentDictionary<'a, 'b>()
    fun arg ->
        async {
            match cache.TryGetValue(arg) with
            | true, value -> return value
            | _ ->
                let! value = f arg
                cache.TryAdd(arg, value) |> ignore
                return value
        }

// ========================================
// 1.3 ตัวอย่างการใช้งาน memoize
// ========================================

// Fibonacci with memoization
let fibonacci =
    let rec fib n =
        match n with
        | 0 | 1 -> n
        | n -> memoizedFib (n-1) + memoizedFib (n-2)
    and memoizedFib = memoize fib
    memoizedFib

// Expensive computation
let expensiveComputation =
    memoizeWithTtl (TimeSpan.FromMinutes 5.0) (fun (userId: int) ->
        printfn "Computing for user %d (expensive operation)..." userId
        System.Threading.Thread.Sleep(100)  // simulate expensive work
        { UserId = userId; Score = userId * 42 }
    )

type UserScore = { UserId: int; Score: int }
```

---

## 2. IMemoryCache (Microsoft.Extensions.Caching.Memory)

```fsharp
// IMemoryCacheExample.fs
module IMemoryCacheExample

open System
open Microsoft.Extensions.Caching.Memory

// ========================================
// 2.1 Basic IMemoryCache usage
// ========================================

let demonstrateMemoryCache () =
    let options = MemoryCacheOptions()
    options.SizeLimit <- Nullable(1000L)  // max 1000 items

    use cache = new MemoryCache(options)

    // Set with absolute expiration
    let cacheKey = "my-key"
    let value = "Hello, Cache!"

    let entryOptions = MemoryCacheEntryOptions()
    entryOptions.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 10.0)
    entryOptions.Size <- Nullable(1L)

    cache.Set(cacheKey, value, entryOptions) |> ignore

    // Get
    match cache.TryGetValue<string>(cacheKey) with
    | true, cachedValue -> printfn "Cache hit: %s" cachedValue
    | false, _ -> printfn "Cache miss"

// ========================================
// 2.2 GetOrCreate pattern
// ========================================

let getOrCreateExample (cache: IMemoryCache) (key: string) =
    cache.GetOrCreate(key, fun entry ->
        entry.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 5.0)
        entry.SlidingExpiration <- Nullable(TimeSpan.FromMinutes 2.0)
        entry.Priority <- CacheItemPriority.Normal

        // Register callback ตอน evict
        entry.RegisterPostEvictionCallback(fun evictedKey evictedValue reason _ ->
            printfn "Cache evicted: key=%O, reason=%A" evictedKey reason
        )

        // Return the value to cache
        "computed value for " + key
    )

// ========================================
// 2.3 GetOrCreateAsync
// ========================================

let getOrCreateAsync (cache: IMemoryCache) (userId: int) =
    task {
        let key = $"user:{userId}"
        let! result = cache.GetOrCreateAsync(key, fun entry ->
            task {
                entry.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 5.0)
                // Simulate DB call
                return { Id = userId; Name = $"User {userId}"; Email = $"user{userId}@example.com" }
            })
        return result
    }

type UserDto = { Id: int; Name: string; Email: string }

// ========================================
// 2.4 Cache with tags/dependencies
// ========================================

// MemoryCache ใน .NET ไม่มี built-in tag support
// แต่เราทำเองได้ด้วย CancellationTokenSource

type TaggedMemoryCache(cache: IMemoryCache) =
    let tagTokens = System.Collections.Concurrent.ConcurrentDictionary<string, System.Threading.CancellationTokenSource>()

    let getTagToken (tag: string) =
        tagTokens.GetOrAdd(tag, fun _ -> new System.Threading.CancellationTokenSource())

    member _.Set<'T> (key: string) (value: 'T) (tags: string list) (ttl: TimeSpan) =
        let entry = cache.CreateEntry(key)
        entry.Value <- value :> obj
        entry.AbsoluteExpirationRelativeToNow <- Nullable(ttl)

        // Link to each tag's cancellation token
        for tag in tags do
            let cts = getTagToken tag
            entry.AddExpirationToken(
                Microsoft.Extensions.Primitives.CancellationChangeToken(cts.Token)
            )

        entry.Dispose()

    member _.Get<'T> (key: string) : 'T option =
        match cache.TryGetValue<'T>(key) with
        | true, value -> Some value
        | _ -> None

    member _.InvalidateTag (tag: string) =
        match tagTokens.TryGetValue(tag) with
        | true, cts ->
            cts.Cancel()
            tagTokens.TryRemove(tag) |> ignore
            cts.Dispose()
        | _ -> ()
```

---

## 3. IDistributedCache

```fsharp
// IDistributedCacheExample.fs
module IDistributedCacheExample

open System
open System.Text.Json
open Microsoft.Extensions.Caching.Distributed

// ========================================
// 3.1 IDistributedCache basics
// ========================================

let demonstrateDistributedCache (cache: IDistributedCache) =
    task {
        let key = "distributed-key"
        let value = "Hello from distributed cache!"

        // Set
        let options = DistributedCacheEntryOptions()
        options.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 10.0)
        options.SlidingExpiration <- Nullable(TimeSpan.FromMinutes 3.0)

        do! cache.SetStringAsync(key, value, options)

        // Get
        let! result = cache.GetStringAsync(key)
        match result with
        | null -> printfn "Cache miss"
        | v -> printfn "Cache hit: %s" v

        // Remove
        do! cache.RemoveAsync(key)
    }

// ========================================
// 3.2 Serialize/Deserialize complex types
// ========================================

type CacheExtensions =
    static member SetJsonAsync<'T> (cache: IDistributedCache) (key: string) (value: 'T) (options: DistributedCacheEntryOptions) =
        task {
            let json = JsonSerializer.Serialize(value)
            do! cache.SetStringAsync(key, json, options)
        }

    static member GetJsonAsync<'T> (cache: IDistributedCache) (key: string) =
        task {
            let! json = cache.GetStringAsync(key)
            match json with
            | null -> return None
            | j ->
                try
                    return Some (JsonSerializer.Deserialize<'T>(j))
                with _ ->
                    return None
        }

// ========================================
// 3.3 Cache service wrapper
// ========================================

type ProductCacheService(cache: IDistributedCache) =
    let productKey (id: int) = $"product:{id}"
    let listKey = "products:list"

    let defaultOptions () =
        let opts = DistributedCacheEntryOptions()
        opts.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 15.0)
        opts

    member _.GetProduct (id: int) =
        task {
            let key = productKey id
            return! CacheExtensions.GetJsonAsync<Product> cache key
        }

    member _.SetProduct (product: Product) =
        task {
            let key = productKey product.Id
            do! CacheExtensions.SetJsonAsync cache key product (defaultOptions())
        }

    member _.InvalidateProduct (id: int) =
        task {
            do! cache.RemoveAsync(productKey id)
        }

    member _.GetProductList () =
        task {
            return! CacheExtensions.GetJsonAsync<Product list> cache listKey
        }

    member _.SetProductList (products: Product list) =
        task {
            let opts = DistributedCacheEntryOptions()
            opts.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromMinutes 5.0)
            do! CacheExtensions.SetJsonAsync cache listKey products opts
        }

    member _.InvalidateAll () =
        task {
            do! cache.RemoveAsync(listKey)
            // Note: IDistributedCache ไม่มี bulk remove
            // ต้องใช้ Redis SCAN หรือ pattern matching แทน
        }

type Product = { Id: int; Name: string; Price: decimal }
```

---

## 4. Redis as Distributed Cache

```fsharp
// RedisCache.fs
module RedisCache

open System
open System.Text.Json
open StackExchange.Redis

// ========================================
// 4.1 Setup Redis with IDistributedCache
// ========================================

(*
NuGet: Microsoft.Extensions.Caching.StackExchangeRedis

// ใน Program.fs:
builder.Services.AddStackExchangeRedisCache(fun options ->
    options.Configuration <- "localhost:6379"
    options.InstanceName <- "MyApp:"
)
*)

// ========================================
// 4.2 Direct Redis operations (more features)
// ========================================

type RedisCache(connectionString: string) =
    let redis = ConnectionMultiplexer.Connect(connectionString)
    let db = redis.GetDatabase()

    // String operations
    member _.GetString (key: string) =
        task {
            let! value = db.StringGetAsync(key)
            if value.IsNull then return None
            else return Some (string value)
        }

    member _.SetString (key: string) (value: string) (ttl: TimeSpan option) =
        task {
            match ttl with
            | Some t -> do! db.StringSetAsync(key, value, Nullable(t)) |> Async.AwaitTask |> Async.Ignore
            | None -> do! db.StringSetAsync(key, value) |> Async.AwaitTask |> Async.Ignore
        }

    // JSON object caching
    member this.Get<'T> (key: string) =
        task {
            let! json = this.GetString key
            return json |> Option.map JsonSerializer.Deserialize<'T>
        }

    member this.Set<'T> (key: string) (value: 'T) (ttl: TimeSpan option) =
        task {
            let json = JsonSerializer.Serialize(value)
            do! this.SetString key json ttl
        }

    member _.Remove (key: string) =
        task {
            do! db.KeyDeleteAsync(key) |> Async.AwaitTask |> Async.Ignore
        }

    member _.Exists (key: string) =
        task {
            return! db.KeyExistsAsync(key)
        }

    member _.GetTtl (key: string) =
        task {
            return! db.KeyTimeToLiveAsync(key)
        }

    // Hash operations for partial updates
    member _.HashGet<'T> (hashKey: string) (field: string) =
        task {
            let! value = db.HashGetAsync(hashKey, field)
            if value.IsNull then return None
            else return Some (JsonSerializer.Deserialize<'T>(string value))
        }

    member _.HashSet (hashKey: string) (field: string) (value: obj) =
        task {
            let json = JsonSerializer.Serialize(value)
            do! db.HashSetAsync(hashKey, field, json) |> Async.AwaitTask |> Async.Ignore
        }

    member _.HashGetAll<'T> (hashKey: string) =
        task {
            let! entries = db.HashGetAllAsync(hashKey)
            return entries
                |> Array.choose (fun entry ->
                    try Some (string entry.Name, JsonSerializer.Deserialize<'T>(string entry.Value))
                    with _ -> None
                )
                |> Map.ofArray
        }

    interface IDisposable with
        member _.Dispose() = redis.Dispose()
```

---

## 5. Cache-Aside Pattern

```fsharp
// CacheAside.fs
module CacheAside

open System
open System.Text.Json

// ========================================
// 5.1 Cache-Aside (Lazy Loading)
// ========================================

(*
Cache-Aside Pattern:
1. App ตรวจสอบ cache ก่อน
2. ถ้า cache miss -> ดึงจาก DB -> เก็บใน cache -> return
3. ถ้า cache hit -> return จาก cache

เหมาะกับ: read-heavy workloads
ข้อเสีย: cold start ช้า, possible inconsistency
*)

type IUserRepository =
    abstract GetById : int -> System.Threading.Tasks.Task<User option>
    abstract GetAll : unit -> System.Threading.Tasks.Task<User list>
    abstract Save : User -> System.Threading.Tasks.Task<unit>
    abstract Delete : int -> System.Threading.Tasks.Task<unit>

type User = {
    Id: int
    Name: string
    Email: string
    UpdatedAt: DateTime
}

type CachedUserRepository(inner: IUserRepository, cache: StackExchange.Redis.IDatabase) =
    let userKey (id: int) = $"user:{id}"
    let allUsersKey = "users:all"
    let ttl = TimeSpan.FromMinutes 10.0

    let serialize v = JsonSerializer.Serialize(v)
    let deserialize<'T> (s: string) = JsonSerializer.Deserialize<'T>(s)

    interface IUserRepository with
        member _.GetById id =
            task {
                let key = userKey id
                let! cached = cache.StringGetAsync(key)
                if cached.IsNull then
                    // Cache miss - load from DB
                    let! user = inner.GetById id
                    match user with
                    | Some u ->
                        do! cache.StringSetAsync(key, serialize u, Nullable(ttl))
                            |> System.Threading.Tasks.Task.FromResult
                            |> fun _ -> System.Threading.Tasks.Task.CompletedTask
                        return Some u
                    | None ->
                        // Cache negative result too (to prevent DB hammering)
                        do! cache.StringSetAsync(key, "null", Nullable(TimeSpan.FromMinutes 1.0))
                            |> System.Threading.Tasks.Task.FromResult
                            |> fun _ -> System.Threading.Tasks.Task.CompletedTask
                        return None
                else
                    let json = string cached
                    if json = "null" then return None
                    else return Some (deserialize<User> json)
            }

        member _.GetAll () =
            task {
                let! cached = cache.StringGetAsync(allUsersKey)
                if cached.IsNull then
                    let! users = inner.GetAll()
                    do! cache.StringSetAsync(allUsersKey, serialize users, Nullable(TimeSpan.FromMinutes 5.0))
                        |> System.Threading.Tasks.Task.FromResult
                        |> fun _ -> System.Threading.Tasks.Task.CompletedTask
                    return users
                else
                    return deserialize<User list> (string cached)
            }

        member _.Save user =
            task {
                do! inner.Save user
                // Invalidate cache
                do! cache.KeyDeleteAsync(userKey user.Id)
                    |> System.Threading.Tasks.Task.FromResult
                    |> fun _ -> System.Threading.Tasks.Task.CompletedTask
                do! cache.KeyDeleteAsync(allUsersKey)
                    |> System.Threading.Tasks.Task.FromResult
                    |> fun _ -> System.Threading.Tasks.Task.CompletedTask
            }

        member _.Delete id =
            task {
                do! inner.Delete id
                // Invalidate cache
                do! cache.KeyDeleteAsync(userKey id)
                    |> System.Threading.Tasks.Task.FromResult
                    |> fun _ -> System.Threading.Tasks.Task.CompletedTask
                do! cache.KeyDeleteAsync(allUsersKey)
                    |> System.Threading.Tasks.Task.FromResult
                    |> fun _ -> System.Threading.Tasks.Task.CompletedTask
            }
```

---

## 6. Cache Stampede Prevention

```fsharp
// CacheStampede.fs
module CacheStampede

open System
open System.Threading
open System.Collections.Concurrent

// ========================================
// 6.1 Cache stampede problem
// ========================================

(*
Cache Stampede (Thundering Herd):
เกิดขึ้นเมื่อ:
1. Cache expires
2. หลาย requests พร้อมกัน -> ทุกอันเห็น cache miss
3. ทุกอัน query DB พร้อมกัน -> DB overload

วิธีแก้:
1. Mutex/Lock (single flight)
2. Early expiration with background refresh
3. Probabilistic early expiration (XFetch)
*)

// ========================================
// 6.2 Single flight pattern (ป้องกัน stampede)
// ========================================

type SingleFlight<'TKey, 'TValue when 'TKey: equality>() =
    let inFlight = ConcurrentDictionary<'TKey, TaskCompletionSource<'TValue>>()

    member _.GetOrCompute (key: 'TKey) (compute: unit -> System.Threading.Tasks.Task<'TValue>) =
        task {
            let tcs = TaskCompletionSource<'TValue>()
            let existing = inFlight.GetOrAdd(key, tcs)

            if obj.ReferenceEquals(existing, tcs) then
                // We are the leader - do the work
                try
                    let! result = compute()
                    tcs.SetResult(result)
                    inFlight.TryRemove(key) |> ignore
                    return result
                with ex ->
                    tcs.SetException(ex)
                    inFlight.TryRemove(key) |> ignore
                    raise ex
            else
                // We are a follower - wait for leader
                return! existing.Task
        }

// ========================================
// 6.3 Using single flight with cache
// ========================================

type StampedeProtectedCache<'TKey, 'TValue when 'TKey: equality>(cache: SimpleCache<'TKey, 'TValue>) =
    let singleFlight = SingleFlight<'TKey, 'TValue>()

    member _.GetOrLoad (key: 'TKey) (load: unit -> System.Threading.Tasks.Task<'TValue>) (ttl: TimeSpan) =
        task {
            match cache.Get key with
            | Some value -> return value
            | None ->
                let! value = singleFlight.GetOrCompute key (fun () ->
                    task {
                        let! v = load()
                        cache.Set key v (Some ttl)
                        return v
                    }
                )
                return value
        }

// ========================================
// 6.4 Probabilistic early expiration (XFetch algorithm)
// ========================================

(*
XFetch: Recompute cache probabilistically before expiration
Formula: currentTime - (timeToCompute * beta * log(random))

ถ้า expiration ใกล้มาก และ computation ใช้เวลานาน
มีโอกาสสูงที่จะ refresh ก่อน expire
*)

type XFetchCache<'TKey, 'TValue when 'TKey: equality>() =
    let random = Random()
    let store = ConcurrentDictionary<'TKey, {| Value: 'TValue; ExpiresAt: DateTime; Delta: float |}>()

    member _.Set (key: 'TKey) (value: 'TValue) (ttl: TimeSpan) (computeTime: TimeSpan) =
        store.[key] <- {|
            Value = value
            ExpiresAt = DateTime.UtcNow.Add(ttl)
            Delta = computeTime.TotalSeconds
        |}

    member _.ShouldRecompute (key: 'TKey) (beta: float) =
        match store.TryGetValue(key) with
        | false, _ -> true  // not in cache
        | true, entry ->
            let now = DateTime.UtcNow
            let timeToExpiry = (entry.ExpiresAt - now).TotalSeconds
            let earlyExpiry = -entry.Delta * beta * Math.Log(random.NextDouble())
            timeToExpiry - earlyExpiry <= 0.0

    member _.Get (key: 'TKey) =
        match store.TryGetValue(key) with
        | false, _ -> None
        | true, entry when DateTime.UtcNow > entry.ExpiresAt ->
            store.TryRemove(key) |> ignore
            None
        | true, entry -> Some entry.Value

// ========================================
// 6.5 Background refresh pattern
// ========================================

type BackgroundRefreshCache<'TKey, 'TValue when 'TKey: equality>() =
    let store = ConcurrentDictionary<'TKey, {| Value: 'TValue; ExpiresAt: DateTime; RefreshAt: DateTime |}>()
    let refreshLocks = ConcurrentDictionary<'TKey, int>()  // 0 = not refreshing, 1 = refreshing

    member _.Set (key: 'TKey) (value: 'TValue) (ttl: TimeSpan) =
        let now = DateTime.UtcNow
        store.[key] <- {|
            Value = value
            ExpiresAt = now.Add(ttl)
            RefreshAt = now.Add(ttl * 0.75)  // refresh at 75% of TTL
        |}

    member this.Get (key: 'TKey) (loader: unit -> System.Threading.Tasks.Task<'TValue>) =
        task {
            match store.TryGetValue(key) with
            | false, _ ->
                // Cold miss
                let! value = loader()
                this.Set key value (TimeSpan.FromMinutes 5.0)
                return value
            | true, entry ->
                let now = DateTime.UtcNow
                if now > entry.ExpiresAt then
                    // Hard expired
                    let! value = loader()
                    this.Set key value (TimeSpan.FromMinutes 5.0)
                    return value
                elif now > entry.RefreshAt then
                    // Soft expired - return stale, refresh in background
                    if refreshLocks.TryAdd(key, 1) then
                        System.Threading.Tasks.Task.Run(fun () ->
                            task {
                                try
                                    let! value = loader()
                                    this.Set key value (TimeSpan.FromMinutes 5.0)
                                finally
                                    refreshLocks.TryRemove(key) |> ignore
                            }
                        ) |> ignore
                    return entry.Value  // return stale immediately
                else
                    return entry.Value  // fresh hit
        }
```

---

## 7. Sliding vs Absolute Expiration

```fsharp
// ExpirationStrategies.fs
module ExpirationStrategies

open System
open Microsoft.Extensions.Caching.Memory

// ========================================
// 7.1 Absolute Expiration
// ========================================

(*
Absolute Expiration:
- หมดอายุตามเวลาที่กำหนดตายตัว
- ไม่ขึ้นกับว่ามีการใช้งานหรือไม่
- เหมาะกับ: ข้อมูลที่ต้องอัพเดทตาม schedule (เช่น daily price)
*)

let setWithAbsoluteExpiration (cache: IMemoryCache) (key: string) (value: obj) =
    let options = MemoryCacheEntryOptions()
    // หมดอายุเวลา midnight วันพรุ่งนี้
    let tomorrow = DateTime.Today.AddDays(1.0)
    options.AbsoluteExpiration <- Nullable(DateTimeOffset(tomorrow))
    cache.Set(key, value, options) |> ignore

// ========================================
// 7.2 Sliding Expiration
// ========================================

(*
Sliding Expiration:
- Reset expiration ทุกครั้งที่มีการ access
- ถ้าไม่มีการใช้งาน X นาที -> expire
- เหมาะกับ: session data, user preferences
*)

let setWithSlidingExpiration (cache: IMemoryCache) (key: string) (value: obj) =
    let options = MemoryCacheEntryOptions()
    options.SlidingExpiration <- Nullable(TimeSpan.FromMinutes 20.0)
    cache.Set(key, value, options) |> ignore

// ========================================
// 7.3 Combined: Sliding + Absolute Maximum
// ========================================

(*
Combined Strategy:
- Sliding: reset ทุก access
- Absolute maximum: ไม่เกิน N ชั่วโมงไม่ว่าจะมีการใช้แค่ไหน
- เหมาะกับ: authentication tokens, shopping cart
*)

let setWithCombinedExpiration (cache: IMemoryCache) (key: string) (value: obj) =
    let options = MemoryCacheEntryOptions()
    options.SlidingExpiration <- Nullable(TimeSpan.FromMinutes 30.0)
    options.AbsoluteExpirationRelativeToNow <- Nullable(TimeSpan.FromHours 8.0)
    cache.Set(key, value, options) |> ignore

// ========================================
// 7.4 Expiration strategies comparison
// ========================================

let demonstrateExpirationStrategies () =
    printfn "=== Expiration Strategies ==="
    printfn ""
    printfn "1. Absolute Expiration"
    printfn "   - หมดอายุเวลาตายตัว เช่น 14:00:00"
    printfn "   - ใช้สำหรับ: daily data, scheduled refreshes"
    printfn ""
    printfn "2. AbsoluteExpirationRelativeToNow"
    printfn "   - หมดอายุหลังจาก X เวลานับจากสร้าง"
    printfn "   - ใช้สำหรับ: API responses, computed values"
    printfn ""
    printfn "3. Sliding Expiration"
    printfn "   - Reset ทุกครั้งที่ access"
    printfn "   - ใช้สำหรับ: sessions, user preferences"
    printfn ""
    printfn "4. Combined (Sliding + Absolute Max)"
    printfn "   - Best of both worlds"
    printfn "   - ใช้สำหรับ: auth tokens, shopping carts"
```

---

## 8. F# Functional Caching Patterns

```fsharp
// FunctionalCaching.fs
module FunctionalCaching

open System
open System.Collections.Concurrent

// ========================================
// 8.1 Recursive memoization
// ========================================

// Y-combinator style memoization สำหรับ recursive functions
let memoizeRec (f: ('a -> 'b) -> 'a -> 'b) : 'a -> 'b =
    let cache = ConcurrentDictionary<'a, 'b>()
    let rec memoized x =
        cache.GetOrAdd(x, fun k -> f memoized k)
    memoized

// Fibonacci ด้วย memoizeRec
let fibonacci =
    memoizeRec (fun fib n ->
        if n <= 1 then n
        else fib (n-1) + fib (n-2)
    )

// ========================================
// 8.2 Computation Expression for caching
// ========================================

type CacheBuilder(cache: ConcurrentDictionary<string, obj>) =
    member _.Return(x) = x
    member _.ReturnFrom(x) = x

    member _.Bind(m: string * (unit -> 'a), f: 'a -> 'b) : 'b =
        let key, compute = m
        let value =
            cache.GetOrAdd(key, fun _ -> compute() :> obj) :?> 'a
        f value

    member _.Zero() = ()

let cacheStore = ConcurrentDictionary<string, obj>()
let cached = CacheBuilder(cacheStore)

// ========================================
// 8.3 Cache monad for composable caching
// ========================================

type CacheResult<'T> =
    | Hit of 'T
    | Miss of (unit -> 'T)

let withCache (cache: ConcurrentDictionary<string, obj>) (key: string) (f: unit -> 'T) =
    match cache.TryGetValue(key) with
    | true, v -> Hit (v :?> 'T)
    | false, _ ->
        Miss (fun () ->
            let v = f()
            cache.TryAdd(key, v :> obj) |> ignore
            v
        )

let execute = function
    | Hit v -> v
    | Miss f -> f()

// ========================================
// 8.4 Async cache with Result type
// ========================================

type AsyncCache<'TKey, 'TValue when 'TKey: equality>(ttl: TimeSpan) =
    let store = ConcurrentDictionary<'TKey, {| Value: 'TValue; ExpiresAt: DateTime |}>()

    member _.GetOrComputeAsync (key: 'TKey) (compute: unit -> Async<Result<'TValue, string>>) =
        async {
            match store.TryGetValue(key) with
            | true, entry when DateTime.UtcNow < entry.ExpiresAt ->
                return Ok entry.Value
            | _ ->
                let! result = compute()
                match result with
                | Ok value ->
                    store.[key] <- {| Value = value; ExpiresAt = DateTime.UtcNow.Add(ttl) |}
                    return Ok value
                | Error e ->
                    return Error e
        }

// ========================================
// 8.5 Layered cache (L1=memory, L2=redis)
// ========================================

type LayeredCache<'TKey, 'TValue when 'TKey: equality>(
    l1: SimpleCache<'TKey, 'TValue>,
    l2: AsyncCache<'TKey, 'TValue>) =

    member _.Get (key: 'TKey) (load: unit -> Async<Result<'TValue, string>>) =
        async {
            // Check L1 (fast, in-memory)
            match l1.Get key with
            | Some v ->
                return Ok v
            | None ->
                // Check L2 (slower, distributed)
                let! result = l2.GetOrComputeAsync key load
                match result with
                | Ok v ->
                    // Populate L1
                    l1.Set key v (Some (TimeSpan.FromMinutes 1.0))
                    return Ok v
                | Error e ->
                    return Error e
        }

type SimpleCache<'TKey, 'TValue when 'TKey: equality>() =
    let store = ConcurrentDictionary<'TKey, {| Value: 'TValue; ExpiresAt: DateTime option |}>()

    member _.Get (key: 'TKey) =
        match store.TryGetValue(key) with
        | false, _ -> None
        | true, entry ->
            match entry.ExpiresAt with
            | Some exp when DateTime.UtcNow > exp ->
                store.TryRemove(key) |> ignore
                None
            | _ -> Some entry.Value

    member _.Set (key: 'TKey) (value: 'TValue) (ttl: TimeSpan option) =
        store.[key] <- {|
            Value = value
            ExpiresAt = ttl |> Option.map (fun t -> DateTime.UtcNow.Add(t))
        |}
```

---

## 9. Complete Demo

```fsharp
// Program.fs - Complete Caching Demo
module Program

open System
open System.Collections.Concurrent

// ========================================
// Types
// ========================================

type Product = {
    Id: int
    Name: string
    Price: decimal
    Category: string
}

type CacheEntry<'T> = {
    Value: 'T
    CreatedAt: DateTime
    ExpiresAt: DateTime option
    HitCount: int
}

// ========================================
// In-memory cache implementation
// ========================================

type InMemoryCache<'TKey, 'TValue when 'TKey: equality>(name: string) =
    let store = ConcurrentDictionary<'TKey, CacheEntry<'TValue>>()
    let mutable hits = 0
    let mutable misses = 0

    member _.Name = name

    member _.Set (key: 'TKey) (value: 'TValue) (ttl: TimeSpan option) =
        let entry = {
            Value = value
            CreatedAt = DateTime.UtcNow
            ExpiresAt = ttl |> Option.map (fun t -> DateTime.UtcNow.Add(t))
            HitCount = 0
        }
        store.[key] <- entry

    member _.Get (key: 'TKey) : 'TValue option =
        match store.TryGetValue(key) with
        | false, _ ->
            System.Threading.Interlocked.Increment(&misses) |> ignore
            None
        | true, entry ->
            match entry.ExpiresAt with
            | Some exp when DateTime.UtcNow > exp ->
                store.TryRemove(key) |> ignore
                System.Threading.Interlocked.Increment(&misses) |> ignore
                None
            | _ ->
                // Update hit count
                let updated = { entry with HitCount = entry.HitCount + 1 }
                store.[key] <- updated
                System.Threading.Interlocked.Increment(&hits) |> ignore
                Some entry.Value

    member this.GetOrSet (key: 'TKey) (factory: unit -> 'TValue) (ttl: TimeSpan option) =
        match this.Get key with
        | Some v -> v
        | None ->
            let v = factory()
            this.Set key v ttl
            v

    member _.Stats () =
        let total = hits + misses
        let hitRate = if total = 0 then 0.0 else float hits / float total * 100.0
        {|
            Name = name
            TotalKeys = store.Count
            Hits = hits
            Misses = misses
            HitRate = hitRate
        |}

    member _.Evict () =
        let now = DateTime.UtcNow
        let expired = store |> Seq.filter (fun kv ->
            match kv.Value.ExpiresAt with
            | Some exp -> now > exp
            | None -> false
        ) |> Seq.map (fun kv -> kv.Key) |> Seq.toList

        for key in expired do
            store.TryRemove(key) |> ignore

        expired.Length

// ========================================
// Memoization utilities
// ========================================

let memoize (f: 'a -> 'b) =
    let cache = ConcurrentDictionary<'a, 'b>()
    fun x -> cache.GetOrAdd(x, fun k -> f k)

let memoizeRec (f: ('a -> 'b) -> 'a -> 'b) =
    let cache = ConcurrentDictionary<'a, 'b>()
    let rec memo x = cache.GetOrAdd(x, fun k -> f memo k)
    memo

// ========================================
// Demo
// ========================================

[<EntryPoint>]
let main _ =
    printfn "=== F# Caching Demo ==="
    printfn "======================="

    // ========================================
    // Demo 1: Basic memoization
    // ========================================
    printfn "\n--- Demo 1: Memoization ---"

    let mutable callCount = 0
    let expensiveOp =
        memoize (fun (n: int) ->
            callCount <- callCount + 1
            printfn "  Computing for %d..." n
            n * n
        )

    printfn "First calls (will compute):"
    printfn "  5^2 = %d" (expensiveOp 5)
    printfn "  7^2 = %d" (expensiveOp 7)
    printfn "  5^2 = %d (cached)" (expensiveOp 5)
    printfn "  7^2 = %d (cached)" (expensiveOp 7)
    printfn "Total computations: %d (expected 2)" callCount

    // ========================================
    // Demo 2: Recursive memoization
    // ========================================
    printfn "\n--- Demo 2: Recursive Memoization (Fibonacci) ---"

    let fib =
        memoizeRec (fun fib n ->
            if n <= 1 then n
            else fib (n-1) + fib (n-2)
        )

    printfn "Fibonacci sequence:"
    for i in 0..10 do
        printf "%d " (fib i)
    printfn ""
    printfn "fib(30) = %d" (fib 30)
    printfn "fib(40) = %d" (fib 40)

    // ========================================
    // Demo 3: In-memory cache with TTL
    // ========================================
    printfn "\n--- Demo 3: Cache with TTL ---"

    let productCache = InMemoryCache<int, Product>("ProductCache")

    let products = [
        { Id = 1; Name = "Laptop Pro"; Price = 45999m; Category = "Electronics" }
        { Id = 2; Name = "Wireless Mouse"; Price = 799m; Category = "Accessories" }
        { Id = 3; Name = "USB Hub"; Price = 599m; Category = "Accessories" }
    ]

    // Load into cache
    printfn "Loading products into cache..."
    for p in products do
        productCache.Set p.Id p (Some (TimeSpan.FromSeconds 5.0))  // 5 second TTL

    // Access cache
    printfn "\nAccessing products:"
    for i in 1..4 do
        match productCache.Get i with
        | Some p -> printfn "  Cache HIT  - Product %d: %s (฿%.2f)" i p.Name p.Price
        | None    -> printfn "  Cache MISS - Product %d not found" i

    // ========================================
    // Demo 4: GetOrSet pattern
    // ========================================
    printfn "\n--- Demo 4: GetOrSet (Cache-Aside) ---"

    let dbCallCount = ref 0
    let userCache = InMemoryCache<int, string>("UserCache")

    let loadUser (id: int) =
        // Simulate DB call
        dbCallCount.Value <- dbCallCount.Value + 1
        printfn "  [DB] Loading user %d..." id
        $"User_{id}"

    printfn "Getting users (DB should be called only once per user):"
    for _ in 1..3 do
        for userId in [1; 2; 3] do
            let user = userCache.GetOrSet userId (fun () -> loadUser userId) (Some (TimeSpan.FromMinutes 1.0))
            ignore user

    printfn "DB calls made: %d (expected 3 for 3 unique users)" dbCallCount.Value

    // ========================================
    // Demo 5: Cache eviction
    // ========================================
    printfn "\n--- Demo 5: Cache Eviction ---"

    let shortLivedCache = InMemoryCache<string, int>("ShortLivedCache")
    shortLivedCache.Set "a" 1 (Some (TimeSpan.FromMilliseconds 100.0))
    shortLivedCache.Set "b" 2 (Some (TimeSpan.FromSeconds 60.0))
    shortLivedCache.Set "c" 3 None  // no expiration

    printfn "Before sleep - count: %d" shortLivedCache.Stats().TotalKeys

    System.Threading.Thread.Sleep(200)  // Wait for "a" to expire

    let evicted = shortLivedCache.Evict()
    printfn "After 200ms - evicted %d entries" evicted
    printfn "After eviction - count: %d" shortLivedCache.Stats().TotalKeys

    match shortLivedCache.Get "a" with
    | Some _ -> printfn "  'a': still in cache (unexpected)"
    | None   -> printfn "  'a': expired and evicted (expected)"

    match shortLivedCache.Get "b" with
    | Some v -> printfn "  'b': in cache, value=%d (expected)" v
    | None   -> printfn "  'b': not in cache (unexpected)"

    // ========================================
    // Demo 6: Cache statistics
    // ========================================
    printfn "\n--- Demo 6: Cache Statistics ---"

    let stats = productCache.Stats()
    printfn "Cache: %s" stats.Name
    printfn "  Total keys: %d" stats.TotalKeys
    printfn "  Cache hits: %d" stats.Hits
    printfn "  Cache misses: %d" stats.Misses
    printfn "  Hit rate: %.1f%%" stats.HitRate

    // ========================================
    // Demo 7: Layered caching concept
    // ========================================
    printfn "\n--- Demo 7: Layered Cache (L1/L2) ---"

    let l1Cache = InMemoryCache<string, string>("L1-Memory")  // fast, small, short TTL
    let l2Cache = InMemoryCache<string, string>("L2-Extended")  // slower, bigger, long TTL

    let getWithLayeredCache (key: string) (loader: unit -> string) =
        // Check L1 first
        match l1Cache.Get key with
        | Some v ->
            printfn "  L1 HIT for '%s'" key
            v
        | None ->
            // Check L2
            match l2Cache.Get key with
            | Some v ->
                printfn "  L2 HIT for '%s' (populating L1)" key
                l1Cache.Set key v (Some (TimeSpan.FromSeconds 30.0))
                v
            | None ->
                // Load from source
                printfn "  MISS for '%s' (loading from source)" key
                let v = loader()
                l1Cache.Set key v (Some (TimeSpan.FromSeconds 30.0))
                l2Cache.Set key v (Some (TimeSpan.FromMinutes 10.0))
                v

    // Simulate access pattern
    let keys = ["user:1"; "product:5"; "user:1"; "product:5"; "user:2"]
    printfn "Access pattern:"
    for key in keys do
        let value = getWithLayeredCache key (fun () -> $"data-for-{key}")
        ignore value

    printfn "\nL1 Stats: hits=%d, misses=%d" (l1Cache.Stats().Hits) (l1Cache.Stats().Misses)
    printfn "L2 Stats: hits=%d, misses=%d" (l2Cache.Stats().Hits) (l2Cache.Stats().Misses)

    // ========================================
    // Summary
    // ========================================
    printfn "\n=== Caching Best Practices ==="
    let practices = [
        "1. เลือก cache strategy ให้เหมาะกับ use case"
        "2. Absolute expiration สำหรับ data ที่มี schedule"
        "3. Sliding expiration สำหรับ session/user data"
        "4. Combined expiration ป้องกัน cache ค้างนานเกินไป"
        "5. ใช้ cache-aside pattern สำหรับ read-heavy workloads"
        "6. ป้องกัน cache stampede ด้วย single-flight/locking"
        "7. Background refresh ป้องกัน latency spike ตอน expire"
        "8. Layered cache (L1/L2) สำหรับ optimal performance"
        "9. Monitor cache hit rate และ eviction rate"
        "10. Cache invalidation ต้องทำทันทีตอน update data"
    ]

    for p in practices do
        printfn "  %s" p

    printfn "\n=== Demo Complete ==="
    0
```

---

## สรุป (Summary)

```fsharp
// Caching strategies ใน F#:

// 1. Simple memoization
let memoize f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    fun x -> cache.GetOrAdd(x, f)

// 2. Recursive memoization
let memoizeRec f =
    let cache = System.Collections.Concurrent.ConcurrentDictionary()
    let rec memo x = cache.GetOrAdd(x, fun k -> f memo k)
    memo

// 3. Cache-aside pattern
// Check cache -> miss -> load DB -> store in cache -> return

// 4. Cache invalidation strategies:
//    - TTL (absolute/sliding)
//    - Event-driven (invalidate on update)
//    - Tag-based (invalidate group of keys)

// 5. Layered cache
//    L1 (memory, fast, small, short TTL) ->
//    L2 (Redis, slower, large, long TTL) ->
//    DB (source of truth)
```

| Pattern | Use Case | Library |
|---|---|---|
| **Memoize** | Pure functions, computation | Built-in F# |
| **IMemoryCache** | Single-process caching | Microsoft.Extensions.Caching.Memory |
| **IDistributedCache** | Multi-process caching | Microsoft.Extensions.Caching.StackExchangeRedis |
| **Cache-Aside** | Read-heavy, DB-backed | Custom / any cache |
| **Layered** | High performance | Custom combination |
| **Single Flight** | Stampede prevention | Custom |
| **Background Refresh** | No-latency updates | Custom |
