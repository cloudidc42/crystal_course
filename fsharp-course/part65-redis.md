# Part 65 - Redis กับ F#

## บทนำ (Introduction)

Redis (Remote Dictionary Server) เป็น in-memory data structure store ที่ใช้เป็น database, cache, message broker, และ streaming engine มีประสิทธิภาพสูงมากเนื่องจากเก็บข้อมูลใน memory

ใน F# เราใช้ StackExchange.Redis ซึ่งเป็น high-performance Redis client สำหรับ .NET

---

## 1. การติดตั้ง (Installation)

```xml
<!-- fsharp-redis.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="StackExchange.Redis" Version="2.7.20" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="System.Text.Json" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Connection.fs" />
    <Compile Include="StringOps.fs" />
    <Compile Include="HashOps.fs" />
    <Compile Include="ListOps.fs" />
    <Compile Include="SetOps.fs" />
    <Compile Include="SortedSetOps.fs" />
    <Compile Include="PubSub.fs" />
    <Compile Include="Transactions.fs" />
    <Compile Include="Scripting.fs" />
    <Compile Include="CachingPatterns.fs" />
    <Compile Include="DistributedLock.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## 2. Connection (การเชื่อมต่อ)

```fsharp
// Connection.fs
module Connection

open StackExchange.Redis
open System

// ========================================
// 2.1 Basic connection
// ========================================

let createConnection (connectionString: string) =
    ConnectionMultiplexer.Connect(connectionString)

let createLocalConnection () =
    ConnectionMultiplexer.Connect("localhost:6379")

// Connection ที่มีตัวเลือกเพิ่มเติม
let createConnectionWithOptions () =
    let options = ConfigurationOptions()
    options.EndPoints.Add("localhost", 6379)
    options.Password <- "redis_password"  // ถ้ามี
    options.ConnectTimeout <- 5000
    options.SyncTimeout <- 10000
    options.AsyncTimeout <- 10000
    options.ReconnectRetryPolicy <- ExponentialRetry(5000)
    options.AbortOnConnectFail <- false
    options.ConnectRetry <- 3
    ConnectionMultiplexer.Connect(options)

// Sentinel connection สำหรับ high availability
let createSentinelConnection (sentinels: (string * int) list) (masterName: string) =
    let options = ConfigurationOptions()
    for (host, port) in sentinels do
        options.EndPoints.Add(host, port)
    options.ServiceName <- masterName
    ConnectionMultiplexer.Connect(options)

// Cluster connection
let createClusterConnection (nodes: (string * int) list) =
    let options = ConfigurationOptions()
    for (host, port) in nodes do
        options.EndPoints.Add(host, port)
    ConnectionMultiplexer.Connect(options)

// ========================================
// 2.2 Database selection
// ========================================

let getDatabase (connection: ConnectionMultiplexer) (dbIndex: int) =
    connection.GetDatabase(dbIndex)

let getDefaultDatabase (connection: ConnectionMultiplexer) =
    connection.GetDatabase()

// ========================================
// 2.3 Server info
// ========================================

let getServerInfo (connection: ConnectionMultiplexer) =
    let server = connection.GetServer(connection.GetEndPoints().[0])
    {|
        Version = server.Version
        IsConnected = server.IsConnected
        ServerType = server.ServerType
    |}

// ========================================
// 2.4 Async connection
// ========================================

let createConnectionAsync (connectionString: string) =
    task {
        return! ConnectionMultiplexer.ConnectAsync(connectionString)
    }
```

---

## 3. String Operations (การจัดการ Strings)

```fsharp
// StringOps.fs
module StringOps

open System
open StackExchange.Redis
open System.Text.Json

// ========================================
// 3.1 Basic Set/Get
// ========================================

/// Set key-value ธรรมดา
let set (db: IDatabase) (key: string) (value: string) =
    db.StringSet(key, value)

/// Set พร้อม expiry
let setWithExpiry (db: IDatabase) (key: string) (value: string) (ttl: TimeSpan) =
    db.StringSet(key, value, ttl)

/// Set ถ้า key ยังไม่มี (NX)
let setIfNotExists (db: IDatabase) (key: string) (value: string) =
    db.StringSet(key, value, when = When.NotExists)

/// Set ถ้า key มีอยู่แล้ว (XX)
let setIfExists (db: IDatabase) (key: string) (value: string) =
    db.StringSet(key, value, when = When.Exists)

/// Get value
let get (db: IDatabase) (key: string) =
    let value = db.StringGet(key)
    if value.IsNullOrEmpty then None
    else Some (string value)

/// Get and delete (GetDel)
let getAndDelete (db: IDatabase) (key: string) =
    let value = db.StringGetDelete(key)
    if value.IsNullOrEmpty then None
    else Some (string value)

/// Get and Set new value (GetSet)
let getAndSet (db: IDatabase) (key: string) (newValue: string) =
    let oldValue = db.StringGetSet(key, newValue)
    if oldValue.IsNullOrEmpty then None
    else Some (string oldValue)

// ========================================
// 3.2 Numeric operations
// ========================================

/// Increment integer
let increment (db: IDatabase) (key: string) =
    db.StringIncrement(key)

let incrementBy (db: IDatabase) (key: string) (amount: int64) =
    db.StringIncrement(key, amount)

let incrementFloat (db: IDatabase) (key: string) (amount: float) =
    db.StringIncrementFloat(key, amount)

/// Decrement
let decrement (db: IDatabase) (key: string) =
    db.StringDecrement(key)

let decrementBy (db: IDatabase) (key: string) (amount: int64) =
    db.StringDecrement(key, amount)

// ========================================
// 3.3 Batch operations
// ========================================

/// Set หลาย keys พร้อมกัน (MSET)
let setMultiple (db: IDatabase) (keyValues: (string * string) list) =
    let pairs =
        keyValues
        |> List.map (fun (k, v) -> KeyValuePair<RedisKey, RedisValue>(k, v))
        |> List.toArray
    db.StringSet(pairs)

/// Get หลาย keys พร้อมกัน (MGET)
let getMultiple (db: IDatabase) (keys: string list) =
    let redisKeys = keys |> List.map (fun k -> RedisKey(k)) |> List.toArray
    let values = db.StringGet(redisKeys)
    List.zip keys (values |> Array.toList)
    |> List.map (fun (k, v) ->
        k, if v.IsNullOrEmpty then None else Some (string v)
    )

// ========================================
// 3.4 JSON storage
// ========================================

type UserCache = {
    Id: int
    Name: string
    Email: string
    LastLogin: DateTime
}

/// Store object เป็น JSON string
let setObject<'T> (db: IDatabase) (key: string) (obj: 'T) (ttl: TimeSpan option) =
    let json = JsonSerializer.Serialize(obj)
    match ttl with
    | Some t -> db.StringSet(key, json, t)
    | None -> db.StringSet(key, json)

/// Get object จาก JSON string
let getObject<'T> (db: IDatabase) (key: string) =
    let value = db.StringGet(key)
    if value.IsNullOrEmpty then None
    else
        try
            Some (JsonSerializer.Deserialize<'T>(string value))
        with _ ->
            None

// ========================================
// 3.5 String operations
// ========================================

let append (db: IDatabase) (key: string) (value: string) =
    db.StringAppend(key, value)

let getLength (db: IDatabase) (key: string) =
    db.StringLength(key)

let getRange (db: IDatabase) (key: string) (start: int64) (end_: int64) =
    string (db.StringGetRange(key, start, end_))

// ========================================
// 3.6 Async operations
// ========================================

let setAsync (db: IDatabase) (key: string) (value: string) (ttl: TimeSpan option) =
    task {
        return!
            match ttl with
            | Some t -> db.StringSetAsync(key, value, t)
            | None -> db.StringSetAsync(key, value)
    }

let getAsync (db: IDatabase) (key: string) =
    task {
        let! value = db.StringGetAsync(key)
        return if value.IsNullOrEmpty then None else Some (string value)
    }
```

---

## 4. Hash Operations

```fsharp
// HashOps.fs
module HashOps

open StackExchange.Redis

// ========================================
// 4.1 Hash operations
// ========================================

/// Set field ใน hash
let hashSet (db: IDatabase) (key: string) (field: string) (value: string) =
    db.HashSet(key, field, value)

/// Set หลาย fields พร้อมกัน
let hashSetMultiple (db: IDatabase) (key: string) (fields: (string * string) list) =
    let entries =
        fields
        |> List.map (fun (f, v) -> HashEntry(f, v))
        |> List.toArray
    db.HashSet(key, entries)

/// Get field จาก hash
let hashGet (db: IDatabase) (key: string) (field: string) =
    let value = db.HashGet(key, field)
    if value.IsNullOrEmpty then None else Some (string value)

/// Get หลาย fields
let hashGetMultiple (db: IDatabase) (key: string) (fields: string list) =
    let redisFields = fields |> List.map (fun f -> RedisValue(f)) |> List.toArray
    let values = db.HashGet(key, redisFields)
    List.zip fields (values |> Array.toList)
    |> List.map (fun (f, v) -> f, if v.IsNullOrEmpty then None else Some (string v))

/// Get ทั้ง hash
let hashGetAll (db: IDatabase) (key: string) =
    db.HashGetAll(key)
    |> Array.map (fun e -> string e.Name, string e.Value)
    |> Array.toList

/// Delete field
let hashDelete (db: IDatabase) (key: string) (field: string) =
    db.HashDelete(key, field)

/// Check field exists
let hashExists (db: IDatabase) (key: string) (field: string) =
    db.HashExists(key, field)

/// Get all field names
let hashKeys (db: IDatabase) (key: string) =
    db.HashKeys(key) |> Array.map string |> Array.toList

/// Get all values
let hashValues (db: IDatabase) (key: string) =
    db.HashValues(key) |> Array.map string |> Array.toList

/// Count fields
let hashLength (db: IDatabase) (key: string) =
    db.HashLength(key)

/// Increment hash field
let hashIncrement (db: IDatabase) (key: string) (field: string) (amount: int64) =
    db.HashIncrement(key, field, amount)

// ========================================
// 4.2 Store objects as hashes
// ========================================

type SessionData = {
    UserId: int
    Username: string
    Role: string
    LoginTime: System.DateTime
    LastActivity: System.DateTime
}

let storeSession (db: IDatabase) (sessionId: string) (session: SessionData) (ttl: System.TimeSpan) =
    let key = $"session:{sessionId}"
    let fields = [
        "userId", string session.UserId
        "username", session.Username
        "role", session.Role
        "loginTime", session.LoginTime.ToString("o")
        "lastActivity", session.LastActivity.ToString("o")
    ]
    hashSetMultiple db key fields
    db.KeyExpire(key, ttl) |> ignore

let getSession (db: IDatabase) (sessionId: string) =
    let key = $"session:{sessionId}"
    let fields = hashGetAll db key
    if fields.IsEmpty then None
    else
        let fieldMap = fields |> Map.ofList
        let tryGet f = fieldMap |> Map.tryFind f |> Option.defaultValue ""
        try
            Some {
                UserId = int (tryGet "userId")
                Username = tryGet "username"
                Role = tryGet "role"
                LoginTime = System.DateTime.Parse(tryGet "loginTime")
                LastActivity = System.DateTime.Parse(tryGet "lastActivity")
            }
        with _ -> None

let updateSessionActivity (db: IDatabase) (sessionId: string) =
    let key = $"session:{sessionId}"
    hashSet db key "lastActivity" (System.DateTime.UtcNow.ToString("o")) |> ignore
    db.KeyExpire(key, System.TimeSpan.FromHours(2.0)) |> ignore
```

---

## 5. List Operations

```fsharp
// ListOps.fs
module ListOps

open StackExchange.Redis

// ========================================
// 5.1 List operations
// ========================================

/// Push element ซ้าย (head)
let leftPush (db: IDatabase) (key: string) (value: string) =
    db.ListLeftPush(key, value)

/// Push element ขวา (tail)
let rightPush (db: IDatabase) (key: string) (value: string) =
    db.ListRightPush(key, value)

/// Push หลาย elements
let leftPushMany (db: IDatabase) (key: string) (values: string list) =
    let redisValues = values |> List.map (fun v -> RedisValue(v)) |> List.toArray
    db.ListLeftPush(key, redisValues)

let rightPushMany (db: IDatabase) (key: string) (values: string list) =
    let redisValues = values |> List.map (fun v -> RedisValue(v)) |> List.toArray
    db.ListRightPush(key, redisValues)

/// Pop จาก head
let leftPop (db: IDatabase) (key: string) =
    let value = db.ListLeftPop(key)
    if value.IsNullOrEmpty then None else Some (string value)

/// Pop จาก tail
let rightPop (db: IDatabase) (key: string) =
    let value = db.ListRightPop(key)
    if value.IsNullOrEmpty then None else Some (string value)

/// Get element ที่ index
let getAt (db: IDatabase) (key: string) (index: int64) =
    let value = db.ListGetByIndex(key, index)
    if value.IsNullOrEmpty then None else Some (string value)

/// Get range
let getRange (db: IDatabase) (key: string) (start: int64) (stop: int64) =
    db.ListRange(key, start, stop)
    |> Array.map string
    |> Array.toList

/// Get all
let getAll (db: IDatabase) (key: string) =
    getRange db key 0L -1L

/// Length ของ list
let length (db: IDatabase) (key: string) =
    db.ListLength(key)

/// Remove elements
let remove (db: IDatabase) (key: string) (value: string) (count: int64) =
    db.ListRemove(key, value, count)

/// Trim list
let trim (db: IDatabase) (key: string) (start: int64) (stop: int64) =
    db.ListTrim(key, start, stop)

// ========================================
// 5.2 Queue pattern (FIFO) ด้วย Lists
// ========================================

type JobMessage = {
    Id: string
    Type: string
    Payload: string
    CreatedAt: System.DateTime
}

let enqueueJob (db: IDatabase) (queueName: string) (job: JobMessage) =
    let json = System.Text.Json.JsonSerializer.Serialize(job)
    db.ListRightPush(queueName, json) |> ignore

let dequeueJob (db: IDatabase) (queueName: string) =
    let value = db.ListLeftPop(queueName)
    if value.IsNullOrEmpty then None
    else
        try
            Some (System.Text.Json.JsonSerializer.Deserialize<JobMessage>(string value))
        with _ -> None

/// Blocking dequeue (รอจนกว่าจะมี item)
let blockingDequeue (db: IDatabase) (queueName: string) (timeout: System.TimeSpan) =
    let result = db.ListLeftPop(queueName, 1)  // Non-blocking version
    result
    |> Array.tryHead
    |> Option.bind (fun v ->
        if v.IsNullOrEmpty then None
        else
            try Some (System.Text.Json.JsonSerializer.Deserialize<JobMessage>(string v))
            with _ -> None
    )

// ========================================
// 5.3 Stack pattern (LIFO)
// ========================================

let push (db: IDatabase) (stackKey: string) (value: string) =
    db.ListLeftPush(stackKey, value) |> ignore

let pop (db: IDatabase) (stackKey: string) =
    leftPop db stackKey

let peek (db: IDatabase) (stackKey: string) =
    getAt db stackKey 0L
```

---

## 6. Set Operations

```fsharp
// SetOps.fs
module SetOps

open StackExchange.Redis

// ========================================
// 6.1 Basic set operations
// ========================================

let add (db: IDatabase) (key: string) (value: string) =
    db.SetAdd(key, value)

let addMany (db: IDatabase) (key: string) (values: string list) =
    let redisValues = values |> List.map (fun v -> RedisValue(v)) |> List.toArray
    db.SetAdd(key, redisValues)

let remove (db: IDatabase) (key: string) (value: string) =
    db.SetRemove(key, value)

let contains (db: IDatabase) (key: string) (value: string) =
    db.SetContains(key, value)

let getAll (db: IDatabase) (key: string) =
    db.SetMembers(key) |> Array.map string |> Array.toList

let count (db: IDatabase) (key: string) =
    db.SetLength(key)

let popRandom (db: IDatabase) (key: string) =
    let value = db.SetPop(key)
    if value.IsNullOrEmpty then None else Some (string value)

let getRandomMember (db: IDatabase) (key: string) =
    let value = db.SetRandomMember(key)
    if value.IsNullOrEmpty then None else Some (string value)

// ========================================
// 6.2 Set operations
// ========================================

/// Union ของ 2 sets
let union (db: IDatabase) (key1: string) (key2: string) =
    db.SetCombine(SetOperation.Union, [| key1; key2 |])
    |> Array.map string
    |> Array.toList

/// Intersection
let intersect (db: IDatabase) (key1: string) (key2: string) =
    db.SetCombine(SetOperation.Intersect, [| key1; key2 |])
    |> Array.map string
    |> Array.toList

/// Difference
let difference (db: IDatabase) (key1: string) (key2: string) =
    db.SetCombine(SetOperation.Difference, [| key1; key2 |])
    |> Array.map string
    |> Array.toList

/// Store union result
let unionStore (db: IDatabase) (destKey: string) (sourceKeys: string list) =
    let keys = sourceKeys |> List.map (fun k -> RedisKey(k)) |> List.toArray
    db.SetCombineAndStore(SetOperation.Union, destKey, keys)

// ========================================
// 6.3 Practical: Online users tracking
// ========================================

let markUserOnline (db: IDatabase) (userId: int) =
    let now = System.DateTime.UtcNow
    let minuteKey = $"online:{now:yyyyMMddHHmm}"
    add db minuteKey (string userId) |> ignore
    db.KeyExpire(minuteKey, System.TimeSpan.FromMinutes(5.0)) |> ignore

let getOnlineUsers (db: IDatabase) =
    let now = System.DateTime.UtcNow
    let keys = [
        for i in 0..4 do
            let time = now.AddMinutes(-float i)
            yield $"online:{time:yyyyMMddHHmm}"
    ]
    if keys.IsEmpty then []
    else
        let redisKeys = keys |> List.map (fun k -> RedisKey(k)) |> List.toArray
        db.SetCombine(SetOperation.Union, redisKeys)
        |> Array.map (fun v -> int (string v))
        |> Array.toList

let getOnlineCount (db: IDatabase) =
    getOnlineUsers db |> List.length
```

---

## 7. Sorted Set Operations

```fsharp
// SortedSetOps.fs
module SortedSetOps

open StackExchange.Redis

// ========================================
// 7.1 Basic sorted set operations
// ========================================

let add (db: IDatabase) (key: string) (member_: string) (score: float) =
    db.SortedSetAdd(key, member_, score)

let addMany (db: IDatabase) (key: string) (membersWithScores: (string * float) list) =
    let entries =
        membersWithScores
        |> List.map (fun (m, s) -> SortedSetEntry(m, s))
        |> List.toArray
    db.SortedSetAdd(key, entries)

let remove (db: IDatabase) (key: string) (member_: string) =
    db.SortedSetRemove(key, member_)

let getScore (db: IDatabase) (key: string) (member_: string) =
    db.SortedSetScore(key, member_) |> Option.ofNullable

let getRank (db: IDatabase) (key: string) (member_: string) =
    // rank เริ่มจาก 0 (lowest score = rank 0)
    db.SortedSetRank(key, member_) |> Option.ofNullable

/// Get members sorted by score ascending
let getRange (db: IDatabase) (key: string) (start: int64) (stop: int64) =
    db.SortedSetRangeByRank(key, start, stop)
    |> Array.map string
    |> Array.toList

/// Get members with scores
let getRangeWithScores (db: IDatabase) (key: string) (start: int64) (stop: int64) =
    db.SortedSetRangeByRankWithScores(key, start, stop)
    |> Array.map (fun e -> string e.Element, e.Score)
    |> Array.toList

/// Get top N (highest score)
let getTop (db: IDatabase) (key: string) (n: int64) =
    db.SortedSetRangeByRankWithScores(key, -n, -1L, Order.Descending)
    |> Array.map (fun e -> string e.Element, e.Score)
    |> Array.toList

/// Get by score range
let getByScoreRange (db: IDatabase) (key: string) (minScore: float) (maxScore: float) =
    db.SortedSetRangeByScoreWithScores(key, minScore, maxScore)
    |> Array.map (fun e -> string e.Element, e.Score)
    |> Array.toList

let count (db: IDatabase) (key: string) =
    db.SortedSetLength(key)

let increment (db: IDatabase) (key: string) (member_: string) (amount: float) =
    db.SortedSetIncrement(key, member_, amount)

// ========================================
// 7.2 Leaderboard pattern
// ========================================

let addScore (db: IDatabase) (leaderboardKey: string) (userId: int) (score: float) =
    add db leaderboardKey (string userId) score |> ignore

let updateScore (db: IDatabase) (leaderboardKey: string) (userId: int) (additionalScore: float) =
    increment db leaderboardKey (string userId) additionalScore |> ignore

let getLeaderboard (db: IDatabase) (leaderboardKey: string) (topN: int64) =
    db.SortedSetRangeByRankWithScores(leaderboardKey, 0L, topN - 1L, Order.Descending)
    |> Array.mapi (fun i e ->
        {|
            Rank = i + 1
            UserId = int (string e.Element)
            Score = e.Score
        |}
    )
    |> Array.toList

let getUserRank (db: IDatabase) (leaderboardKey: string) (userId: int) =
    let rank = db.SortedSetRank(leaderboardKey, string userId, Order.Descending)
    rank |> Option.ofNullable |> Option.map (fun r -> int r + 1)

let getUserScore (db: IDatabase) (leaderboardKey: string) (userId: int) =
    getScore db leaderboardKey (string userId)

// ========================================
// 7.3 Rate limiting with sorted sets
// ========================================

let checkRateLimit (db: IDatabase) (userId: int) (maxRequests: int) (windowSeconds: int) =
    let key = $"ratelimit:user:{userId}"
    let now = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
    let windowStart = now - int64 windowSeconds * 1000L

    // Remove old entries
    db.SortedSetRemoveRangeByScore(key, -infinity, float windowStart) |> ignore

    // Count current requests
    let requestCount = db.SortedSetLength(key, float windowStart, float now)

    if requestCount >= int64 maxRequests then
        false  // Rate limited
    else
        // Add this request
        db.SortedSetAdd(key, string now, float now) |> ignore
        db.KeyExpire(key, System.TimeSpan.FromSeconds(float windowSeconds * 2.0)) |> ignore
        true  // Allowed
```

---

## 8. Pub/Sub

```fsharp
// PubSub.fs
module PubSub

open StackExchange.Redis
open System

// ========================================
// 8.1 Basic Pub/Sub
// ========================================

type MessageHandler = string -> string -> unit

/// Subscribe to a channel
let subscribe (connection: ConnectionMultiplexer) (channel: string) (handler: MessageHandler) =
    let subscriber = connection.GetSubscriber()
    subscriber.Subscribe(
        RedisChannel.Literal(channel),
        fun ch msg -> handler (string ch) (string msg)
    )

/// Publish message
let publish (connection: ConnectionMultiplexer) (channel: string) (message: string) =
    let subscriber = connection.GetSubscriber()
    subscriber.Publish(RedisChannel.Literal(channel), message)

/// Unsubscribe
let unsubscribe (connection: ConnectionMultiplexer) (channel: string) =
    let subscriber = connection.GetSubscriber()
    subscriber.Unsubscribe(RedisChannel.Literal(channel))

// ========================================
// 8.2 Pattern subscribe
// ========================================

/// Subscribe ด้วย pattern (เช่น "order:*")
let subscribePattern (connection: ConnectionMultiplexer) (pattern: string) (handler: MessageHandler) =
    let subscriber = connection.GetSubscriber()
    subscriber.Subscribe(
        RedisChannel.Pattern(pattern),
        fun ch msg -> handler (string ch) (string msg)
    )

// ========================================
// 8.3 Typed messages
// ========================================

type OrderEvent = {
    OrderId: int
    Status: string
    UpdatedAt: DateTime
}

type EventBus(connection: ConnectionMultiplexer) =
    let subscriber = connection.GetSubscriber()

    member _.PublishOrderEvent (event: OrderEvent) =
        let json = System.Text.Json.JsonSerializer.Serialize(event)
        subscriber.Publish(RedisChannel.Literal("order:events"), json) |> ignore

    member _.SubscribeToOrderEvents (handler: OrderEvent -> unit) =
        subscriber.Subscribe(
            RedisChannel.Literal("order:events"),
            fun _ msg ->
                try
                    let event = System.Text.Json.JsonSerializer.Deserialize<OrderEvent>(string msg)
                    handler event
                with ex ->
                    printfn "Error handling event: %s" ex.Message
        )

// ========================================
// 8.4 Async Pub/Sub
// ========================================

let publishAsync (connection: ConnectionMultiplexer) (channel: string) (message: string) =
    task {
        let subscriber = connection.GetSubscriber()
        return! subscriber.PublishAsync(RedisChannel.Literal(channel), message)
    }

let subscribeAsync (connection: ConnectionMultiplexer) (channel: string) (handler: MessageHandler) =
    task {
        let subscriber = connection.GetSubscriber()
        do! subscriber.SubscribeAsync(
            RedisChannel.Literal(channel),
            fun ch msg -> handler (string ch) (string msg)
        )
    }
```

---

## 9. Transactions และ Pipelining

```fsharp
// Transactions.fs
module Transactions

open StackExchange.Redis

// ========================================
// 9.1 Transactions
// ========================================

/// Atomic transaction
let executeTransaction (db: IDatabase) (operations: ITransaction -> unit) =
    let transaction = db.CreateTransaction()
    operations transaction
    transaction.Execute()

/// Transaction ที่มี conditional watch
let updateWithWatch (db: IDatabase) (key: string) (updateFunc: string -> string) =
    let mutable success = false
    while not success do
        // Watch key สำหรับ optimistic locking
        db.Multiplexer.GetDatabase().Execute("WATCH", key) |> ignore
        let currentValue = db.StringGet(key) |> string
        let newValue = updateFunc currentValue

        let transaction = db.CreateTransaction()
        // Condition: ค่าต้องยังเป็น currentValue
        transaction.AddCondition(Condition.StringEqual(key, currentValue)) |> ignore
        transaction.StringSetAsync(key, newValue) |> ignore

        success <- transaction.Execute()

// ========================================
// 9.2 Pipelining
// ========================================

/// Execute หลาย commands พร้อมกันด้วย pipeline
let executePipeline (db: IDatabase) =
    let batch = db.CreateBatch()

    // Queue commands
    let task1 = batch.StringSetAsync("key1", "value1")
    let task2 = batch.StringSetAsync("key2", "value2")
    let task3 = batch.StringGetAsync("key1")
    let task4 = batch.StringGetAsync("key2")

    // Execute all at once (reduces round trips)
    batch.Execute()

    // Await results
    System.Threading.Tasks.Task.WhenAll(task1, task2) |> Async.AwaitTask |> Async.RunSynchronously |> ignore
    let v1 = task3.Result
    let v2 = task4.Result
    (string v1, string v2)

/// Bulk operations ด้วย pipeline
let bulkSet (db: IDatabase) (keyValues: (string * string) list) =
    let batch = db.CreateBatch()
    let tasks = keyValues |> List.map (fun (k, v) -> batch.StringSetAsync(k, v))
    batch.Execute()
    tasks |> List.map (fun t -> t.Result) |> List.forall id

let bulkGet (db: IDatabase) (keys: string list) =
    let batch = db.CreateBatch()
    let tasks = keys |> List.map (fun k -> k, batch.StringGetAsync(k))
    batch.Execute()
    tasks |> List.map (fun (k, t) -> k, if t.Result.IsNullOrEmpty then None else Some (string t.Result))
```

---

## 10. Lua Scripts

```fsharp
// Scripting.fs
module Scripting

open StackExchange.Redis

// ========================================
// 10.1 Lua scripts
// ========================================

/// Atomic increment กับ max limit
let limitedIncrement (db: IDatabase) (key: string) (maxValue: int) =
    let script = """
        local current = tonumber(redis.call('GET', KEYS[1])) or 0
        if current >= tonumber(ARGV[1]) then
            return -1
        end
        return redis.call('INCR', KEYS[1])
    """
    let result = db.ScriptEvaluate(script, [| RedisKey(key) |], [| RedisValue(maxValue) |])
    int result

/// Check and set (atomic)
let checkAndSet (db: IDatabase) (key: string) (expectedValue: string) (newValue: string) =
    let script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            redis.call('SET', KEYS[1], ARGV[2])
            return 1
        end
        return 0
    """
    let result = db.ScriptEvaluate(
        script,
        [| RedisKey(key) |],
        [| RedisValue(expectedValue); RedisValue(newValue) |]
    )
    int result = 1

/// Cached Lua script (pre-loaded)
let createCachedScript (db: IDatabase) (script: string) =
    LuaScript.Prepare(script)

let executeCachedScript (db: IDatabase) (prepared: LoadedLuaScript) (keys: string[]) (args: string[]) =
    let redisKeys = keys |> Array.map (fun k -> RedisKey(k))
    let redisArgs = args |> Array.map (fun a -> RedisValue(a))
    prepared.Evaluate(db, redisKeys, redisArgs)

// ========================================
// 10.2 Common script patterns
// ========================================

/// Atomic rate limiter
let rateLimitScript = """
    local key = KEYS[1]
    local max = tonumber(ARGV[1])
    local window = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window * 1000)
    local count = redis.call('ZCARD', key)

    if count >= max then
        return 0
    end

    redis.call('ZADD', key, now, now)
    redis.call('PEXPIRE', key, window * 1000)
    return 1
"""

let checkRateLimit (db: IDatabase) (userId: int) (maxReq: int) (windowSec: int) =
    let key = $"ratelimit:{userId}"
    let now = System.DateTimeOffset.UtcNow.ToUnixTimeMilliseconds()
    let result = db.ScriptEvaluate(
        rateLimitScript,
        [| RedisKey(key) |],
        [| RedisValue(maxReq); RedisValue(windowSec); RedisValue(now) |]
    )
    int result = 1
```

---

## 11. Key Expiration และ TTL

```fsharp
// KeyExpiration.fs
module KeyExpiration

open StackExchange.Redis
open System

// ========================================
// 11.1 Key expiration
// ========================================

let setExpiry (db: IDatabase) (key: string) (ttl: TimeSpan) =
    db.KeyExpire(key, ttl)

let setExpiryAt (db: IDatabase) (key: string) (expiryDate: DateTime) =
    db.KeyExpireAsync(key, expiryDate) |> Async.AwaitTask |> Async.RunSynchronously

let getTimeToLive (db: IDatabase) (key: string) =
    db.KeyTimeToLive(key) |> Option.ofNullable

let removeExpiry (db: IDatabase) (key: string) =
    db.KeyPersist(key)

// ========================================
// 11.2 Key management
// ========================================

let exists (db: IDatabase) (key: string) =
    db.KeyExists(key)

let delete (db: IDatabase) (key: string) =
    db.KeyDelete(key)

let deleteMany (db: IDatabase) (keys: string list) =
    let redisKeys = keys |> List.map (fun k -> RedisKey(k)) |> List.toArray
    db.KeyDelete(redisKeys)

let keyType (db: IDatabase) (key: string) =
    db.KeyType(key)

let renameKey (db: IDatabase) (oldKey: string) (newKey: string) =
    db.KeyRename(oldKey, newKey)

/// Scan keys ด้วย pattern
let scanKeys (db: IDatabase) (server: IServer) (pattern: string) =
    server.Keys(pattern = pattern) |> Seq.map string |> Seq.toList

// ========================================
// 11.3 TTL patterns
// ========================================

/// Sliding expiration (รีเซ็ต TTL ทุกครั้งที่ access)
let getWithSlidingExpiry (db: IDatabase) (key: string) (slidingTtl: TimeSpan) =
    let value = db.StringGet(key)
    if not value.IsNullOrEmpty then
        db.KeyExpire(key, slidingTtl) |> ignore
    if value.IsNullOrEmpty then None else Some (string value)

/// Absolute expiration
let setWithAbsoluteExpiry (db: IDatabase) (key: string) (value: string) (expiryDate: DateTime) =
    db.StringSet(key, value) |> ignore
    db.KeyExpire(key, expiryDate) |> ignore
```

---

## 12. Distributed Lock

```fsharp
// DistributedLock.fs
module DistributedLock

open StackExchange.Redis
open System

// ========================================
// 12.1 Simple distributed lock
// ========================================

type LockResult =
    | Acquired of string  // lock value
    | NotAcquired

/// เข้า lock (Redis SETNX)
let acquireLock (db: IDatabase) (lockKey: string) (ttl: TimeSpan) =
    let lockValue = Guid.NewGuid().ToString("N")
    let acquired = db.StringSet(lockKey, lockValue, ttl, When.NotExists)
    if acquired then Acquired lockValue
    else NotAcquired

/// ปล่อย lock (ต้องตรวจสอบ value ก่อนเพื่อความปลอดภัย)
let releaseLock (db: IDatabase) (lockKey: string) (lockValue: string) =
    let script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('DEL', KEYS[1])
        end
        return 0
    """
    let result = db.ScriptEvaluate(
        script,
        [| RedisKey(lockKey) |],
        [| RedisValue(lockValue) |]
    )
    int result = 1

/// ต่ออายุ lock
let renewLock (db: IDatabase) (lockKey: string) (lockValue: string) (ttl: TimeSpan) =
    let script = """
        if redis.call('GET', KEYS[1]) == ARGV[1] then
            return redis.call('PEXPIRE', KEYS[1], ARGV[2])
        end
        return 0
    """
    let result = db.ScriptEvaluate(
        script,
        [| RedisKey(lockKey) |],
        [| RedisValue(lockValue); RedisValue(int ttl.TotalMilliseconds) |]
    )
    int result = 1

// ========================================
// 12.2 Lock with auto-release
// ========================================

type DistributedLockHandle(db: IDatabase, lockKey: string, lockValue: string) =
    let mutable released = false

    interface IDisposable with
        member _.Dispose() =
            if not released then
                released <- true
                releaseLock db lockKey lockValue |> ignore

    member _.IsValid =
        let value = db.StringGet(lockKey)
        not value.IsNullOrEmpty && string value = lockValue

/// ใช้งาน lock แบบ using
let withLock (db: IDatabase) (lockKey: string) (ttl: TimeSpan) (action: unit -> 'T) =
    match acquireLock db lockKey ttl with
    | NotAcquired -> Error "Could not acquire lock"
    | Acquired lockValue ->
        use _lock = new DistributedLockHandle(db, lockKey, lockValue)
        try
            Ok (action())
        with ex ->
            Error ex.Message

// ========================================
// 12.3 Retry with lock
// ========================================

let withLockRetry (db: IDatabase) (lockKey: string) (ttl: TimeSpan) (maxRetries: int) (retryDelay: TimeSpan) (action: unit -> 'T) =
    let mutable retries = 0
    let mutable result = None

    while result.IsNone && retries < maxRetries do
        match acquireLock db lockKey ttl with
        | Acquired lockValue ->
            use _lock = new DistributedLockHandle(db, lockKey, lockValue)
            result <- Some (try Ok (action()) with ex -> Error ex.Message)
        | NotAcquired ->
            retries <- retries + 1
            if retries < maxRetries then
                System.Threading.Thread.Sleep(retryDelay)

    result |> Option.defaultValue (Error "Could not acquire lock after max retries")
```

---

## 13. Caching Patterns (รูปแบบการ Cache)

```fsharp
// CachingPatterns.fs
module CachingPatterns

open StackExchange.Redis
open System
open System.Text.Json

// ========================================
// 13.1 Cache-aside pattern
// ========================================

type CacheAside<'T>(db: IDatabase, keyPrefix: string, defaultTtl: TimeSpan) =
    let makeKey id = $"{keyPrefix}:{id}"

    /// Get from cache or load from source
    member _.GetOrSet (id: string) (loader: string -> 'T option) =
        let key = makeKey id
        let cached = db.StringGet(key)
        if not cached.IsNullOrEmpty then
            try
                JsonSerializer.Deserialize<'T>(string cached) |> Some
            with _ ->
                None
        else
            match loader id with
            | Some value ->
                let json = JsonSerializer.Serialize(value)
                db.StringSet(key, json, defaultTtl) |> ignore
                Some value
            | None ->
                None

    member _.Invalidate (id: string) =
        db.KeyDelete(makeKey id) |> ignore

    member _.Set (id: string) (value: 'T) (ttl: TimeSpan option) =
        let key = makeKey id
        let json = JsonSerializer.Serialize(value)
        match ttl with
        | Some t -> db.StringSet(key, json, t) |> ignore
        | None -> db.StringSet(key, json, defaultTtl) |> ignore

// ========================================
// 13.2 Write-through pattern
// ========================================

let writeThrough (db: IDatabase) (key: string) (value: string) (ttl: TimeSpan) (writeToDb: unit -> unit) =
    // เขียนลง cache และ database พร้อมกัน
    writeToDb()
    db.StringSet(key, value, ttl) |> ignore

// ========================================
// 13.3 Cache stampede prevention
// ========================================

/// ป้องกัน cache stampede ด้วย locking
let getOrSetWithLock<'T> (db: IDatabase) (key: string) (lockKey: string) (ttl: TimeSpan) (loader: unit -> 'T) =
    let cached = db.StringGet(key)
    if not cached.IsNullOrEmpty then
        try Some (JsonSerializer.Deserialize<'T>(string cached))
        with _ -> None
    else
        // ลอง acquire lock
        let lockValue = Guid.NewGuid().ToString("N")
        let lockAcquired = db.StringSet(lockKey, lockValue, TimeSpan.FromSeconds(30.0), When.NotExists)

        if lockAcquired then
            try
                // Double-check หลัง lock
                let cached2 = db.StringGet(key)
                if not cached2.IsNullOrEmpty then
                    try Some (JsonSerializer.Deserialize<'T>(string cached2))
                    with _ -> None
                else
                    let value = loader()
                    let json = JsonSerializer.Serialize(value)
                    db.StringSet(key, json, ttl) |> ignore
                    Some value
            finally
                // Release lock
                let script = "if redis.call('GET', KEYS[1]) == ARGV[1] then return redis.call('DEL', KEYS[1]) end return 0"
                db.ScriptEvaluate(script, [| RedisKey(lockKey) |], [| RedisValue(lockValue) |]) |> ignore
        else
            // รอให้ lock ถูกปล่อย และ get จาก cache
            System.Threading.Thread.Sleep(100)
            let cached3 = db.StringGet(key)
            if not cached3.IsNullOrEmpty then
                try Some (JsonSerializer.Deserialize<'T>(string cached3))
                with _ -> None
            else
                None

// ========================================
// 13.4 Multi-level cache
// ========================================

type MultiLevelCache<'T>(l1Cache: System.Runtime.Caching.MemoryCache, l2Db: IDatabase, keyPrefix: string, l1Ttl: TimeSpan, l2Ttl: TimeSpan) =
    let makeKey id = $"{keyPrefix}:{id}"

    member _.Get (id: string) =
        let key = makeKey id

        // L1: in-process memory cache
        let l1Result = l1Cache.Get(key)
        if l1Result <> null then
            Some (l1Result :?> 'T)
        else
            // L2: Redis
            let l2Result = l2Db.StringGet(key)
            if not l2Result.IsNullOrEmpty then
                try
                    let value = JsonSerializer.Deserialize<'T>(string l2Result)
                    // Populate L1
                    l1Cache.Set(key, value, DateTimeOffset.UtcNow.Add(l1Ttl))
                    Some value
                with _ ->
                    None
            else
                None

    member _.Set (id: string) (value: 'T) =
        let key = makeKey id
        let json = JsonSerializer.Serialize(value)

        // Set in both levels
        l1Cache.Set(key, value, DateTimeOffset.UtcNow.Add(l1Ttl))
        l2Db.StringSet(key, json, l2Ttl) |> ignore

    member _.Invalidate (id: string) =
        let key = makeKey id
        l1Cache.Remove(key) |> ignore
        l2Db.KeyDelete(key) |> ignore
```

---

## 14. Complete Example (ตัวอย่างครบวงจร)

```fsharp
// Program.fs
module Program

open System
open StackExchange.Redis

[<EntryPoint>]
let main _ =
    printfn "=== Redis F# Demo ==="
    printfn "====================="

    // สร้าง mock connection สำหรับ demo
    // ในงานจริง: let connection = Connection.createLocalConnection()
    printfn "\nConnection Setup:"
    printfn "  let connection = ConnectionMultiplexer.Connect(\"localhost:6379\")"
    printfn "  let db = connection.GetDatabase()"

    // แสดง Redis data structures
    printfn "\n--- Redis Data Structures ---"

    let dataStructures = [
        "Strings", "Simple key-value pairs, counters, JSON objects"
        "Hashes", "Field-value maps, perfect for objects/sessions"
        "Lists", "Ordered lists, queues (FIFO), stacks (LIFO)"
        "Sets", "Unique unordered collections, set operations"
        "Sorted Sets", "Scored members, leaderboards, rate limiting"
        "Streams", "Append-only log, message queues (Redis 5.0+)"
        "Bitmaps", "Bit-level operations on strings"
        "HyperLogLog", "Approximate cardinality counting"
        "Geospatial", "Location-based operations"
    ]

    for (name, desc) in dataStructures do
        printfn "  [%s]: %s" name desc

    // แสดง common commands
    printfn "\n--- Common Redis Commands ---"

    let commands = [
        "Strings", [
            "SET key value [EX seconds]"
            "GET key"
            "INCR key / INCRBY key amount"
            "MSET key1 v1 key2 v2 / MGET key1 key2"
            "SETNX key value (set if not exists)"
            "GETSET key newvalue"
        ]
        "Hashes", [
            "HSET key field value"
            "HGET key field"
            "HMSET key f1 v1 f2 v2"
            "HMGET key f1 f2"
            "HGETALL key"
            "HINCRBY key field amount"
        ]
        "Lists", [
            "LPUSH key value / RPUSH key value"
            "LPOP key / RPOP key"
            "LRANGE key start stop"
            "LLEN key"
            "BLPOP key timeout (blocking)"
        ]
        "Sets", [
            "SADD key member1 member2"
            "SMEMBERS key"
            "SISMEMBER key member"
            "SUNION key1 key2"
            "SINTER key1 key2"
        ]
        "Sorted Sets", [
            "ZADD key score member"
            "ZRANGE key start stop [WITHSCORES]"
            "ZREVRANGE key start stop"
            "ZSCORE key member"
            "ZRANK key member"
        ]
    ]

    for (category, cmds) in commands do
        printfn "\n  [%s]:" category
        for cmd in cmds do
            printfn "    %s" cmd

    // แสดง use cases
    printfn "\n--- Common Use Cases ---"
    let useCases = [
        "Session storage", "Hash per session, TTL for auto-expiry"
        "Caching", "String/Hash with TTL, cache-aside pattern"
        "Rate limiting", "Sorted Set with timestamp scores"
        "Leaderboards", "Sorted Set ZADD/ZREVRANGE"
        "Job queues", "List RPUSH/BLPOP"
        "Pub/Sub", "Real-time notifications, chat"
        "Distributed lock", "SETNX with TTL + Lua script release"
        "Counting", "INCR/INCRBY for atomic counters"
        "Online users", "Set with SADD/SREM"
        "Auto-complete", "Sorted Set with lexicographic scores"
    ]

    for (case, implementation) in useCases do
        printfn "  [%s]: %s" case implementation

    // แสดง best practices
    printfn "\n--- Best Practices ---"
    let practices = [
        "1. กำหนด key naming convention เช่น 'resource:id:field'"
        "2. ตั้ง TTL ให้กับ keys ที่ไม่ permanent เสมอ"
        "3. ใช้ Pipelining เพื่อลด round trips"
        "4. ใช้ Lua scripts สำหรับ atomic operations"
        "5. Monitor memory usage และตั้ง maxmemory policy"
        "6. ใช้ connection pooling (ConnectionMultiplexer เป็น singleton)"
        "7. Handle connection failures และ implement retry logic"
        "8. ใช้ Redis Cluster สำหรับ horizontal scaling"
        "9. Enable persistence (AOF/RDB) ถ้าต้องการ durability"
        "10. ใช้ SCAN แทน KEYS ใน production"
    ]

    for practice in practices do
        printfn "  %s" practice

    printfn "\n=== Demo Complete ==="
    0
```

---

## สรุป (Summary)

Redis กับ F# ผ่าน StackExchange.Redis:

1. **ConnectionMultiplexer** เป็น thread-safe และ reusable - ใช้เป็น singleton
2. **Data structures** แต่ละแบบมีจุดแข็งต่างกัน
3. **Pipelining** ช่วยลด latency สำหรับ batch operations
4. **Lua scripts** ช่วยทำ atomic operations ที่ซับซ้อน
5. **TTL** สำคัญมากสำหรับ cache management

```fsharp
// F# helper functions สำหรับ Redis:
let inline (?) (value: RedisValue) (default_: 'T) =
    if value.IsNullOrEmpty then default_
    else value :?> 'T

// Usage patterns:
// Cache: getOrSet db "user:1" (fun () -> loadFromDb 1) (TimeSpan.FromMinutes 5.0)
// Queue: RPUSH + BLPOP
// Lock: SETNX + Lua DELETE
// Leaderboard: ZADD + ZREVRANGE
// Rate limit: ZADD + ZCOUNT + ZREMRANGEBYSCORE
```
