# Part 64 - MongoDB กับ F#

## บทนำ (Introduction)

MongoDB เป็น NoSQL database แบบ document-oriented ที่เก็บข้อมูลในรูปแบบ BSON (Binary JSON) มีความยืดหยุ่นสูงเนื่องจากไม่ต้องมี schema ที่ตายตัว เหมาะสำหรับข้อมูลที่มีโครงสร้างแตกต่างกัน

---

## 1. การติดตั้ง (Installation)

```xml
<!-- fsharp-mongodb.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="MongoDB.Driver" Version="2.26.0" />
    <PackageReference Include="MongoDB.Bson" Version="2.26.0" />
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Models.fs" />
    <Compile Include="Connection.fs" />
    <Compile Include="CrudOperations.fs" />
    <Compile Include="Aggregation.fs" />
    <Compile Include="Indexes.fs" />
    <Compile Include="Transactions.fs" />
    <Compile Include="ChangeStreams.fs" />
    <Compile Include="GridFS.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## 2. Models (โมเดลข้อมูล)

```fsharp
// Models.fs
module Models

open System
open MongoDB.Bson
open MongoDB.Bson.Serialization.Attributes

// ========================================
// 2.1 Basic document types
// ========================================

// วิธีที่ 1: ใช้ class กับ BsonDocument attributes
[<BsonIgnoreExtraElements>]
type UserDocument() =
    [<BsonId>]
    [<BsonRepresentation(BsonType.ObjectId)>]
    member val Id: string = ObjectId.GenerateNewId().ToString() with get, set

    [<BsonElement("name")>]
    member val Name: string = "" with get, set

    [<BsonElement("email")>]
    member val Email: string = "" with get, set

    [<BsonElement("age")>]
    member val Age: int = 0 with get, set

    [<BsonElement("isActive")>]
    member val IsActive: bool = true with get, set

    [<BsonElement("createdAt")>]
    member val CreatedAt: DateTime = DateTime.UtcNow with get, set

    [<BsonElement("tags")>]
    member val Tags: string list = [] with get, set

    [<BsonElement("address")>]
    member val Address: AddressDocument = null with get, set

and [<BsonIgnoreExtraElements>] AddressDocument() =
    [<BsonElement("street")>]
    member val Street: string = "" with get, set

    [<BsonElement("city")>]
    member val City: string = "" with get, set

    [<BsonElement("country")>]
    member val Country: string = "" with get, set

    [<BsonElement("postalCode")>]
    member val PostalCode: string = "" with get, set

// ========================================
// 2.2 Record-based approach (F# friendly)
// ========================================

// วิธีที่ 2: ใช้ F# records (ต้องระวัง serialization)
// MongoDB.Driver รองรับ records ใน .NET 5+ แต่ต้องตั้งค่า custom serializer

type Address = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

// Product document ที่ใช้ ObjectId
type ProductDocument = {
    [<BsonId; BsonRepresentation(BsonType.ObjectId)>]
    Id: string
    Name: string
    Price: decimal
    Description: string option
    CategoryId: string
    Tags: string list
    Stock: int
    IsActive: bool
    CreatedAt: DateTime
    UpdatedAt: DateTime option
    Metadata: Map<string, string>
}

// Order document
type OrderDocument = {
    [<BsonId; BsonRepresentation(BsonType.ObjectId)>]
    Id: string
    CustomerId: string
    Items: OrderItemDoc list
    TotalAmount: decimal
    Status: string
    ShippingAddress: Address option
    CreatedAt: DateTime
    UpdatedAt: DateTime option
}

and OrderItemDoc = {
    ProductId: string
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

// ========================================
// 2.3 Helper functions สำหรับสร้าง ObjectId
// ========================================

let newId () = ObjectId.GenerateNewId().ToString()

let isValidObjectId (id: string) =
    ObjectId.TryParse(id, ref (ObjectId())) |> fst

// ========================================
// 2.4 DTOs สำหรับ API responses
// ========================================

type UserDto = {
    Id: string
    Name: string
    Email: string
    IsActive: bool
}

type ProductDto = {
    Id: string
    Name: string
    Price: decimal
    CategoryId: string
    InStock: bool
}

// Converters
let toUserDto (doc: UserDocument) : UserDto = {
    Id = doc.Id
    Name = doc.Name
    Email = doc.Email
    IsActive = doc.IsActive
}
```

---

## 3. Connection (การเชื่อมต่อ)

```fsharp
// Connection.fs
module Connection

open MongoDB.Driver

// ========================================
// 3.1 Basic connection
// ========================================

let createClient (connectionString: string) =
    MongoClient(connectionString)

let createClientWithOptions (host: string) (port: int) (username: string) (password: string) =
    let settings = MongoClientSettings()
    settings.Server <- MongoServerAddress(host, port)
    if not (String.IsNullOrEmpty username) then
        settings.Credential <- MongoCredential.CreateCredential("admin", username, password)
    settings.ConnectTimeout <- System.TimeSpan.FromSeconds(30.0)
    settings.SocketTimeout <- System.TimeSpan.FromSeconds(60.0)
    settings.MaxConnectionPoolSize <- 100
    settings.MinConnectionPoolSize <- 5
    MongoClient(settings)

// ========================================
// 3.2 Database and Collection access
// ========================================

let getDatabase (client: MongoClient) (databaseName: string) =
    client.GetDatabase(databaseName)

let getCollection<'T> (database: IMongoDatabase) (collectionName: string) =
    database.GetCollection<'T>(collectionName)

// ========================================
// 3.3 Connection test
// ========================================

let testConnection (connectionString: string) =
    task {
        try
            let client = MongoClient(connectionString)
            let db = client.GetDatabase("admin")
            let! _ = db.RunCommandAsync<MongoDB.Bson.BsonDocument>(
                MongoDB.Bson.BsonDocument("ping", 1)
            )
            return true
        with ex ->
            printfn "Connection failed: %s" ex.Message
            return false
    }

// ========================================
// 3.4 MongoUrl builder
// ========================================

let buildConnectionString (host: string) (port: int) (database: string) =
    $"mongodb://{host}:{port}/{database}"

let buildAuthConnectionString (username: string) (password: string) (host: string) (port: int) (database: string) =
    $"mongodb://{username}:{password}@{host}:{port}/{database}?authSource=admin"

// Replica set connection
let buildReplicaSetConnectionString (hosts: string list) (replicaSetName: string) (database: string) =
    let hostList = hosts |> String.concat ","
    $"mongodb://{hostList}/{database}?replicaSet={replicaSetName}"
```

---

## 4. CRUD Operations

```fsharp
// CrudOperations.fs
module CrudOperations

open System
open MongoDB.Driver
open MongoDB.Bson
open Models

// ========================================
// 4.1 Insert operations
// ========================================

/// Insert เอกสารเดียว
let insertOne (collection: IMongoCollection<UserDocument>) (user: UserDocument) =
    task {
        do! collection.InsertOneAsync(user)
        return user.Id
    }

/// Insert หลายเอกสาร
let insertMany (collection: IMongoCollection<UserDocument>) (users: UserDocument list) =
    task {
        if not users.IsEmpty then
            do! collection.InsertManyAsync(users)
        return users.Length
    }

/// Insert พร้อม error handling
let tryInsertOne (collection: IMongoCollection<UserDocument>) (user: UserDocument) =
    task {
        try
            do! collection.InsertOneAsync(user)
            return Ok user.Id
        with
        | :? MongoWriteException as ex when ex.WriteError.Category = ServerErrorCategory.DuplicateKey ->
            return Error "Duplicate key error"
        | ex ->
            return Error ex.Message
    }

// ========================================
// 4.2 Find operations
// ========================================

/// ค้นหาด้วย Filter
let findById (collection: IMongoCollection<UserDocument>) (id: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, id)
        return! collection.Find(filter).FirstOrDefaultAsync()
    }

let findByEmail (collection: IMongoCollection<UserDocument>) (email: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Email, email)
        return! collection.Find(filter).FirstOrDefaultAsync()
    }

let findAll (collection: IMongoCollection<UserDocument>) =
    task {
        return! collection.Find(FilterDefinition<UserDocument>.Empty).ToListAsync()
    }

let findActive (collection: IMongoCollection<UserDocument>) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.IsActive, true)
        return!
            collection.Find(filter)
                .SortBy(fun u -> u.Name :> obj)
                .ToListAsync()
    }

/// ค้นหาด้วย complex filter
let findByAgeRange (collection: IMongoCollection<UserDocument>) (minAge: int) (maxAge: int) =
    task {
        let filter = Builders<UserDocument>.Filter.And(
            Builders<UserDocument>.Filter.Gte(fun u -> u.Age, minAge),
            Builders<UserDocument>.Filter.Lte(fun u -> u.Age, maxAge)
        )
        return! collection.Find(filter).ToListAsync()
    }

/// ค้นหาด้วย text search (ต้องมี text index)
let searchByName (collection: IMongoCollection<UserDocument>) (searchTerm: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Regex(
            (fun u -> u.Name :> obj),
            MongoDB.Bson.BsonRegularExpression(searchTerm, "i")
        )
        return! collection.Find(filter).ToListAsync()
    }

/// ค้นหาด้วย array contains
let findByTag (collection: IMongoCollection<UserDocument>) (tag: string) =
    task {
        let filter = Builders<UserDocument>.Filter.AnyEq(
            fun u -> u.Tags :> obj, tag
        )
        return! collection.Find(filter).ToListAsync()
    }

/// Paged query
let findPaged (collection: IMongoCollection<UserDocument>) (page: int) (pageSize: int) =
    task {
        let filter = FilterDefinition<UserDocument>.Empty
        let skip = (page - 1) * pageSize

        let! total = collection.CountDocumentsAsync(filter)
        let! items =
            collection.Find(filter)
                .Skip(skip)
                .Limit(pageSize)
                .ToListAsync()

        return {|
            Items = items |> Seq.toList
            Total = total
            Page = page
            PageSize = pageSize
            TotalPages = int ((total + int64 pageSize - 1L) / int64 pageSize)
        |}
    }

// ========================================
// 4.3 Update operations
// ========================================

/// Update เอกสารเดียว
let updateOne (collection: IMongoCollection<UserDocument>) (id: string) (name: string) (email: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, id)
        let update = Builders<UserDocument>.Update
                        .Set(fun u -> u.Name, name)
                        .Set(fun u -> u.Email, email)
        let! result = collection.UpdateOneAsync(filter, update)
        return result.ModifiedCount > 0L
    }

/// Update หลายเอกสาร
let deactivateUsers (collection: IMongoCollection<UserDocument>) (ids: string list) =
    task {
        let filter = Builders<UserDocument>.Filter.In(
            fun u -> u.Id :> obj, ids
        )
        let update = Builders<UserDocument>.Update.Set(fun u -> u.IsActive, false)
        let! result = collection.UpdateManyAsync(filter, update)
        return result.ModifiedCount
    }

/// Upsert (Insert or Update)
let upsertUser (collection: IMongoCollection<UserDocument>) (user: UserDocument) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Email, user.Email)
        let options = ReplaceOptions(IsUpsert = true)
        let! result = collection.ReplaceOneAsync(filter, user, options)
        return
            if result.UpsertedId <> null then
                result.UpsertedId.ToString()
            else
                user.Id
    }

/// Push element to array
let addTagToUser (collection: IMongoCollection<UserDocument>) (userId: string) (tag: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, userId)
        let update = Builders<UserDocument>.Update.AddToSet(fun u -> u.Tags :> obj, tag)
        let! result = collection.UpdateOneAsync(filter, update)
        return result.ModifiedCount > 0L
    }

/// Pull element from array
let removeTagFromUser (collection: IMongoCollection<UserDocument>) (userId: string) (tag: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, userId)
        let update = Builders<UserDocument>.Update.Pull(fun u -> u.Tags :> obj, tag)
        let! result = collection.UpdateOneAsync(filter, update)
        return result.ModifiedCount > 0L
    }

/// Increment a numeric field
let incrementAge (collection: IMongoCollection<UserDocument>) (userId: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, userId)
        let update = Builders<UserDocument>.Update.Inc(fun u -> u.Age, 1)
        let! result = collection.UpdateOneAsync(filter, update)
        return result.ModifiedCount > 0L
    }

// ========================================
// 4.4 Delete operations
// ========================================

/// Delete เอกสารเดียว
let deleteOne (collection: IMongoCollection<UserDocument>) (id: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, id)
        let! result = collection.DeleteOneAsync(filter)
        return result.DeletedCount > 0L
    }

/// Delete หลายเอกสาร
let deleteInactive (collection: IMongoCollection<UserDocument>) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.IsActive, false)
        let! result = collection.DeleteManyAsync(filter)
        return result.DeletedCount
    }

// ========================================
// 4.5 FindAndModify (atomic)
// ========================================

/// Find and update atomically (return updated document)
let findAndUpdate (collection: IMongoCollection<UserDocument>) (id: string) (newName: string) =
    task {
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.Id, id)
        let update = Builders<UserDocument>.Update.Set(fun u -> u.Name, newName)
        let options = FindOneAndUpdateOptions<UserDocument>()
        options.ReturnDocument <- ReturnDocument.After
        return! collection.FindOneAndUpdateAsync(filter, update, options)
    }
```

---

## 5. Filter Builders (การสร้าง Filters)

```fsharp
// FilterBuilders.fs
module FilterBuilders

open MongoDB.Driver
open MongoDB.Bson
open Models

let fb = Builders<UserDocument>.Filter

// Comparison
let equalsFilter = fb.Eq(fun u -> u.Name, "Alice")
let notEqualsFilter = fb.Ne(fun u -> u.IsActive, false)
let greaterThanFilter = fb.Gt(fun u -> u.Age, 18)
let greaterOrEqualFilter = fb.Gte(fun u -> u.Age, 18)
let lessThanFilter = fb.Lt(fun u -> u.Age, 65)
let lessOrEqualFilter = fb.Lte(fun u -> u.Age, 65)

// Logical
let andFilter = fb.And(
    fb.Gte(fun u -> u.Age, 18),
    fb.Lte(fun u -> u.Age, 65),
    fb.Eq(fun u -> u.IsActive, true)
)

let orFilter = fb.Or(
    fb.Eq(fun u -> u.Name, "Alice"),
    fb.Eq(fun u -> u.Name, "Bob")
)

let notFilter = fb.Not(fb.Eq(fun u -> u.IsActive, false))

// Array
let inFilter = fb.In(fun u -> u.Name :> obj, ["Alice"; "Bob"; "Charlie"])
let notInFilter = fb.Nin(fun u -> u.Name :> obj, ["Dave"; "Eve"])
let anyTagFilter = fb.AnyEq(fun u -> u.Tags :> obj, "premium")

// Text
let regexFilter = fb.Regex(fun u -> u.Name :> obj, BsonRegularExpression("^A", "i"))

// Element
let existsFilter = fb.Exists("address", true)
let typeFilter = fb.Type(fun u -> u.Name :> obj, BsonType.String)

// Null check
let notNullFilter = fb.Ne(fun u -> u.Address, null)

// Combining complex filters
let complexSearch (searchTerm: string) (minAge: int) (tags: string list) =
    let conditions = [
        fb.Or(
            fb.Regex(fun u -> u.Name :> obj, BsonRegularExpression(searchTerm, "i")),
            fb.Regex(fun u -> u.Email :> obj, BsonRegularExpression(searchTerm, "i"))
        )
        fb.Gte(fun u -> u.Age, minAge)
        fb.Eq(fun u -> u.IsActive, true)
    ]
    let tagConditions =
        if tags.IsEmpty then []
        else [fb.AnyIn(fun u -> u.Tags :> obj, tags)]

    fb.And(conditions @ tagConditions)
```

---

## 6. Aggregation Pipeline

```fsharp
// Aggregation.fs
module Aggregation

open System
open MongoDB.Driver
open MongoDB.Bson
open Models

// ========================================
// 6.1 Basic aggregation
// ========================================

/// นับจำนวน users ต่อ status
let countUsersByStatus (collection: IMongoCollection<UserDocument>) =
    task {
        let pipeline = [
            BsonDocument("$group", BsonDocument(
                dict [
                    "_id", BsonString("$isActive") :> BsonValue
                    "count", BsonDocument("$sum", 1) :> BsonValue
                ]
            ))
            BsonDocument("$sort", BsonDocument("_id", -1))
        ]
        return! collection.Aggregate<BsonDocument>(pipeline).ToListAsync()
    }

// ========================================
// 6.2 Aggregation with typed pipeline builder
// ========================================

type UserStats = {
    AgeGroup: string
    Count: int
    AverageAge: float
}

let getUserAgeStats (collection: IMongoCollection<UserDocument>) =
    task {
        let pipeline =
            collection.Aggregate()
                .Match(Builders<UserDocument>.Filter.Eq(fun u -> u.IsActive, true))
                .Group(
                    fun u ->
                        if u.Age < 30 then "Young"
                        elif u.Age < 50 then "Middle"
                        else "Senior",
                    fun g ->
                        {|
                            AgeGroup = g.Key
                            Count = g.Count()
                            AvgAge = g.Average(fun u -> float u.Age)
                        |}
                )
                .SortByDescending(fun g -> g.Count :> obj)

        return! pipeline.ToListAsync()
    }

// ========================================
// 6.3 Lookup (join)
// ========================================

/// Join orders กับ customer info
let getOrdersWithCustomers (db: IMongoDatabase) =
    task {
        let orders = db.GetCollection<BsonDocument>("orders")

        let pipeline = [
            BsonDocument("$lookup", BsonDocument(
                dict [
                    "from", BsonString("users") :> BsonValue
                    "localField", BsonString("customerId") :> BsonValue
                    "foreignField", BsonString("_id") :> BsonValue
                    "as", BsonString("customer") :> BsonValue
                ]
            ))
            BsonDocument("$unwind", BsonDocument(
                dict [
                    "path", BsonString("$customer") :> BsonValue
                    "preserveNullAndEmptyArrays", BsonBoolean(false) :> BsonValue
                ]
            ))
            BsonDocument("$project", BsonDocument(
                dict [
                    "orderId", BsonString("$_id") :> BsonValue
                    "customerName", BsonString("$customer.name") :> BsonValue
                    "totalAmount", BsonInt32(1) :> BsonValue
                    "status", BsonInt32(1) :> BsonValue
                ]
            ))
            BsonDocument("$sort", BsonDocument("totalAmount", -1))
            BsonDocument("$limit", BsonInt32(10))
        ]

        return! orders.Aggregate<BsonDocument>(pipeline).ToListAsync()
    }

// ========================================
// 6.4 Unwind, Group, Project
// ========================================

/// Top 10 tags
let getTopTags (collection: IMongoCollection<UserDocument>) (limit: int) =
    task {
        let pipeline = [
            BsonDocument("$unwind", BsonString("$tags"))
            BsonDocument("$group", BsonDocument(
                dict [
                    "_id", BsonString("$tags") :> BsonValue
                    "count", BsonDocument("$sum", 1) :> BsonValue
                ]
            ))
            BsonDocument("$sort", BsonDocument("count", -1))
            BsonDocument("$limit", BsonInt32(limit))
            BsonDocument("$project", BsonDocument(
                dict [
                    "_id", BsonInt32(0) :> BsonValue
                    "tag", BsonString("$_id") :> BsonValue
                    "count", BsonInt32(1) :> BsonValue
                ]
            ))
        ]
        return! collection.Aggregate<BsonDocument>(pipeline).ToListAsync()
    }

// ========================================
// 6.5 $facet - multiple aggregations at once
// ========================================

/// ดึงสถิติหลายอย่างพร้อมกัน
let getComprehensiveStats (collection: IMongoCollection<UserDocument>) =
    task {
        let pipeline = [
            BsonDocument("$facet", BsonDocument(
                dict [
                    "byAge", BsonArray([
                        BsonDocument("$group", BsonDocument(
                            dict [
                                "_id", BsonDocument("$cond", BsonArray([
                                    BsonDocument("$lt", BsonArray([BsonString("$age"); BsonInt32(30)]))
                                    BsonString("young")
                                    BsonString("adult")
                                ])) :> BsonValue
                                "count", BsonDocument("$sum", 1) :> BsonValue
                            ]
                        ))
                    ]) :> BsonValue
                    "totalActive", BsonArray([
                        BsonDocument("$match", BsonDocument("isActive", true))
                        BsonDocument("$count", BsonString("total"))
                    ]) :> BsonValue
                    "avgAge", BsonArray([
                        BsonDocument("$group", BsonDocument(
                            dict [
                                "_id", BsonNull.Value :> BsonValue
                                "avg", BsonDocument("$avg", BsonString("$age")) :> BsonValue
                            ]
                        ))
                    ]) :> BsonValue
                ]
            ))
        ]
        return! collection.Aggregate<BsonDocument>(pipeline).FirstOrDefaultAsync()
    }
```

---

## 7. Indexes (การสร้าง Index)

```fsharp
// Indexes.fs
module Indexes

open MongoDB.Driver
open MongoDB.Bson
open Models

// ========================================
// 7.1 สร้าง indexes
// ========================================

let createIndexes (collection: IMongoCollection<UserDocument>) =
    task {
        let indexModels = [
            // Unique index บน email
            CreateIndexModel<UserDocument>(
                Builders<UserDocument>.IndexKeys.Ascending(fun u -> u.Email :> obj),
                CreateIndexOptions(Unique = true, Name = "idx_email_unique")
            )
            // Compound index สำหรับ active users sorted by name
            CreateIndexModel<UserDocument>(
                Builders<UserDocument>.IndexKeys
                    .Ascending(fun u -> u.IsActive :> obj)
                    .Ascending(fun u -> u.Name :> obj),
                CreateIndexOptions(Name = "idx_active_name")
            )
            // Text index สำหรับ full-text search
            CreateIndexModel<UserDocument>(
                Builders<UserDocument>.IndexKeys.Text(fun u -> u.Name :> obj),
                CreateIndexOptions(Name = "idx_name_text")
            )
            // TTL index (auto-delete documents after X seconds)
            CreateIndexModel<UserDocument>(
                Builders<UserDocument>.IndexKeys.Ascending(fun u -> u.CreatedAt :> obj),
                CreateIndexOptions(
                    Name = "idx_created_ttl",
                    ExpireAfter = System.TimeSpan.FromDays(365.0)  // Auto-delete after 1 year
                )
            )
        ]

        let! result = collection.Indexes.CreateManyAsync(indexModels)
        printfn "Created indexes: %A" (result |> Seq.toList)
    }

/// List existing indexes
let listIndexes (collection: IMongoCollection<UserDocument>) =
    task {
        let! cursor = collection.Indexes.ListAsync()
        return! cursor.ToListAsync()
    }

/// Drop specific index
let dropIndex (collection: IMongoCollection<UserDocument>) (indexName: string) =
    task {
        do! collection.Indexes.DropOneAsync(indexName)
        printfn "Dropped index: %s" indexName
    }

// ========================================
// 7.2 Partial indexes
// ========================================

let createPartialIndex (collection: IMongoCollection<UserDocument>) =
    task {
        // Index เฉพาะ active users เท่านั้น
        let filter = Builders<UserDocument>.Filter.Eq(fun u -> u.IsActive, true)
        let indexModel = CreateIndexModel<UserDocument>(
            Builders<UserDocument>.IndexKeys.Ascending(fun u -> u.Email :> obj),
            CreateIndexOptions(
                Name = "idx_active_email",
                PartialFilterExpression = filter
            )
        )
        let! result = collection.Indexes.CreateOneAsync(indexModel)
        printfn "Created partial index: %s" result
    }

// ========================================
// 7.3 Wildcard indexes
// ========================================

let createWildcardIndex (collection: IMongoCollection<BsonDocument>) =
    task {
        // Index ทุก field ใน metadata subdocument
        let indexModel = CreateIndexModel<BsonDocument>(
            IndexKeysDefinition<BsonDocument>.Parse("{\"metadata.$**\": 1}"),
            CreateIndexOptions(Name = "idx_metadata_wildcard")
        )
        let! result = collection.Indexes.CreateOneAsync(indexModel)
        printfn "Created wildcard index: %s" result
    }
```

---

## 8. Transactions (การจัดการ Transactions)

```fsharp
// Transactions.fs
module Transactions

open MongoDB.Driver
open Models

// ========================================
// 8.1 Multi-document transactions
// ========================================

/// โอนเงินระหว่าง accounts (multi-document transaction)
let transferFunds
    (client: MongoClient)
    (fromUserId: string)
    (toUserId: string)
    (amount: decimal) =
    task {
        let db = client.GetDatabase("shopdb")
        let accounts = db.GetCollection<BsonDocument>("accounts")

        use session = client.StartSession()
        try
            session.StartTransaction()

            // ตรวจสอบ balance
            let fromFilter = Builders<BsonDocument>.Filter.Eq("_id", fromUserId)
            let! fromAccount = accounts.Find(session, fromFilter).FirstOrDefaultAsync()

            if fromAccount = null then
                session.AbortTransaction()
                return Error "From account not found"
            else
                let balance = fromAccount.["balance"].AsDecimal
                if balance < amount then
                    session.AbortTransaction()
                    return Error "Insufficient funds"
                else
                    // หักเงิน
                    let debitUpdate = Builders<BsonDocument>.Update.Inc("balance", -amount)
                    let! _ = accounts.UpdateOneAsync(session, fromFilter, debitUpdate)

                    // เพิ่มเงิน
                    let toFilter = Builders<BsonDocument>.Filter.Eq("_id", toUserId)
                    let creditUpdate = Builders<BsonDocument>.Update.Inc("balance", amount)
                    let! _ = accounts.UpdateOneAsync(session, toFilter, creditUpdate)

                    session.CommitTransaction()
                    return Ok $"Transferred {amount} successfully"
        with ex ->
            session.AbortTransaction()
            return Error ex.Message
    }

// ========================================
// 8.2 Async transactions
// ========================================

let createOrderTransaction
    (client: MongoClient)
    (customerId: string)
    (items: (string * int) list) =
    // items = (productId, quantity) list
    task {
        let db = client.GetDatabase("shopdb")
        let orders = db.GetCollection<BsonDocument>("orders")
        let products = db.GetCollection<BsonDocument>("products")

        use! session = client.StartSessionAsync()
        try
            session.StartTransaction()

            // ตรวจสอบและ lock inventory
            let mutable totalAmount = 0m
            let orderItems = ResizeArray()

            for (productId, qty) in items do
                let filter = Builders<BsonDocument>.Filter.And(
                    Builders<BsonDocument>.Filter.Eq("_id", productId),
                    Builders<BsonDocument>.Filter.Gte("stock", qty)
                )
                let update = Builders<BsonDocument>.Update.Inc("stock", -qty)
                let options = FindOneAndUpdateOptions<BsonDocument>()
                options.ReturnDocument <- ReturnDocument.After

                let! product = products.FindOneAndUpdateAsync(session, filter, update, options)
                if product = null then
                    do! session.AbortTransactionAsync()
                    return Error $"Product {productId} not available in required quantity"
                else
                    let price = product.["price"].AsDecimal
                    totalAmount <- totalAmount + price * decimal qty
                    orderItems.Add(BsonDocument(
                        dict [
                            "productId", BsonString(productId) :> BsonValue
                            "quantity", BsonInt32(qty) :> BsonValue
                            "unitPrice", BsonDecimal128(price) :> BsonValue
                        ]
                    ))

            // สร้าง order
            let order = BsonDocument(
                dict [
                    "customerId", BsonString(customerId) :> BsonValue
                    "items", BsonArray(orderItems) :> BsonValue
                    "totalAmount", BsonDecimal128(totalAmount) :> BsonValue
                    "status", BsonString("pending") :> BsonValue
                    "createdAt", BsonDateTime(System.DateTime.UtcNow) :> BsonValue
                ]
            )

            do! orders.InsertOneAsync(session, order)
            do! session.CommitTransactionAsync()

            return Ok (order.["_id"].ToString())
        with ex ->
            do! session.AbortTransactionAsync()
            return Error ex.Message
    }
```

---

## 9. Change Streams (การติดตามการเปลี่ยนแปลง)

```fsharp
// ChangeStreams.fs
module ChangeStreams

open System
open System.Threading
open MongoDB.Driver
open Models

// ========================================
// 9.1 Watch collection changes
// ========================================

let watchCollection (collection: IMongoCollection<UserDocument>) (cancellationToken: CancellationToken) =
    task {
        let pipeline = [
            BsonDocumentPipelineStageDefinition<ChangeStreamDocument<UserDocument>, ChangeStreamDocument<UserDocument>>(
                MongoDB.Bson.BsonDocument("$match",
                    MongoDB.Bson.BsonDocument("operationType",
                        MongoDB.Bson.BsonDocument("$in",
                            MongoDB.Bson.BsonArray(["insert"; "update"; "delete"])
                        )
                    )
                )
            )
        ]

        let options = ChangeStreamOptions()
        options.FullDocument <- ChangeStreamFullDocumentOption.UpdateLookup

        use! cursor = collection.WatchAsync(pipeline, options, cancellationToken)

        printfn "Watching for changes on users collection..."
        while! cursor.MoveNextAsync(cancellationToken) do
            for change in cursor.Current do
                match change.OperationType with
                | ChangeStreamOperationType.Insert ->
                    printfn "New user: %s (%s)" change.FullDocument.Name change.FullDocument.Email
                | ChangeStreamOperationType.Update ->
                    printfn "Updated user: %s" (change.DocumentKey.["_id"].ToString())
                | ChangeStreamOperationType.Delete ->
                    printfn "Deleted user: %s" (change.DocumentKey.["_id"].ToString())
                | _ ->
                    printfn "Other operation: %A" change.OperationType
    }

// ========================================
// 9.2 Resume token (สำหรับ reconnection)
// ========================================

let watchWithResumeToken
    (collection: IMongoCollection<UserDocument>)
    (resumeToken: BsonDocument option)
    (cancellationToken: CancellationToken) =
    task {
        let options = ChangeStreamOptions()
        resumeToken |> Option.iter (fun token ->
            options.ResumeAfter <- token
        )
        options.FullDocument <- ChangeStreamFullDocumentOption.UpdateLookup

        let mutable lastToken: BsonDocument = null

        use! cursor = collection.WatchAsync(options, cancellationToken)
        while! cursor.MoveNextAsync(cancellationToken) do
            for change in cursor.Current do
                lastToken <- change.ResumeToken
                printfn "Change: %A at %A" change.OperationType change.ClusterTime

        return lastToken  // Return last token for next session
    }

// ========================================
// 9.3 Watch entire database
// ========================================

let watchDatabase (db: IMongoDatabase) (cancellationToken: CancellationToken) =
    task {
        use! cursor = db.WatchAsync(cancellationToken = cancellationToken)
        while! cursor.MoveNextAsync(cancellationToken) do
            for change in cursor.Current do
                printfn "DB Change in %s.%s: %A"
                    change.CollectionNamespace.DatabaseNamespace.DatabaseName
                    change.CollectionNamespace.CollectionName
                    change.OperationType
    }
```

---

## 10. GridFS (การจัดเก็บไฟล์ขนาดใหญ่)

```fsharp
// GridFS.fs
module GridFS

open System
open System.IO
open MongoDB.Driver
open MongoDB.Driver.GridFS

// ========================================
// 10.1 Setup GridFS
// ========================================

let createBucket (db: IMongoDatabase) (bucketName: string) =
    let options = GridFSBucketOptions()
    options.BucketName <- bucketName
    options.ChunkSizeBytes <- 1024 * 1024  // 1 MB chunks
    GridFSBucket(db, options)

// ========================================
// 10.2 Upload files
// ========================================

let uploadFile (bucket: GridFSBucket) (fileName: string) (content: byte[]) (contentType: string) =
    task {
        let metadata = MongoDB.Bson.BsonDocument(
            dict [
                "contentType", MongoDB.Bson.BsonString(contentType) :> MongoDB.Bson.BsonValue
                "uploadedAt", MongoDB.Bson.BsonDateTime(DateTime.UtcNow) :> MongoDB.Bson.BsonValue
            ]
        )
        let options = GridFSUploadOptions()
        options.Metadata <- metadata

        use stream = new MemoryStream(content)
        return! bucket.UploadFromStreamAsync(fileName, stream, options)
    }

let uploadLocalFile (bucket: GridFSBucket) (localPath: string) =
    task {
        let fileName = Path.GetFileName(localPath)
        use stream = File.OpenRead(localPath)
        return! bucket.UploadFromStreamAsync(fileName, stream)
    }

// ========================================
// 10.3 Download files
// ========================================

let downloadFile (bucket: GridFSBucket) (fileId: MongoDB.Bson.ObjectId) =
    task {
        use ms = new MemoryStream()
        do! bucket.DownloadToStreamAsync(fileId, ms)
        return ms.ToArray()
    }

let downloadFileByName (bucket: GridFSBucket) (fileName: string) =
    task {
        use ms = new MemoryStream()
        do! bucket.DownloadToStreamByNameAsync(fileName, ms)
        return ms.ToArray()
    }

// ========================================
// 10.4 Find files
// ========================================

let findFiles (bucket: GridFSBucket) (pattern: string) =
    task {
        let filter = Builders<GridFSFileInfo>.Filter.Regex(
            "filename", MongoDB.Bson.BsonRegularExpression(pattern, "i")
        )
        return! bucket.Find(filter).ToListAsync()
    }

let getAllFiles (bucket: GridFSBucket) =
    task {
        return! bucket.Find(FilterDefinition<GridFSFileInfo>.Empty).ToListAsync()
    }

// ========================================
// 10.5 Delete files
// ========================================

let deleteFile (bucket: GridFSBucket) (fileId: MongoDB.Bson.ObjectId) =
    task {
        do! bucket.DeleteAsync(fileId)
        printfn "Deleted file: %A" fileId
    }
```

---

## 11. Complete Example (ตัวอย่างครบวงจร)

```fsharp
// Program.fs
module Program

open System
open MongoDB.Driver
open MongoDB.Bson
open Models

[<EntryPoint>]
let main _ =
    printfn "=== MongoDB F# Demo ==="
    printfn "========================"

    // MongoDB connection (เชื่อมต่อกับ MongoDB server)
    // ในที่นี้จะแสดงตัวอย่างการใช้งาน API
    let connectionString = "mongodb://localhost:27017"
    printfn "\nConnection string: %s" connectionString

    // แสดง Document structure
    printfn "\n--- Document Structure Example ---"
    let userDoc = UserDocument()
    userDoc.Name <- "สมชาย ใจดี"
    userDoc.Email <- "somchai@example.com"
    userDoc.Age <- 30
    userDoc.IsActive <- true
    userDoc.Tags <- ["customer"; "premium"; "thailand"]

    let address = AddressDocument()
    address.Street <- "123 ถ.สุขุมวิท"
    address.City <- "กรุงเทพมหานคร"
    address.Country <- "Thailand"
    address.PostalCode <- "10110"
    userDoc.Address <- address

    printfn "User Document:"
    printfn "  ID: %s" userDoc.Id
    printfn "  Name: %s" userDoc.Name
    printfn "  Email: %s" userDoc.Email
    printfn "  Age: %d" userDoc.Age
    printfn "  Tags: %A" userDoc.Tags
    printfn "  Address: %s, %s" address.Street address.City

    // แสดง Filter Builder examples
    printfn "\n--- Filter Builder Examples ---"

    let filters = [
        "Simple equality", "{ 'isActive': true }"
        "Range query", "{ 'age': { '$gte': 18, '$lte': 65 } }"
        "Array contains", "{ 'tags': { '$in': ['premium', 'vip'] } }"
        "Nested field", "{ 'address.city': 'Bangkok' }"
        "Regex", "{ 'name': { '$regex': '^Soma', '$options': 'i' } }"
        "AND", "{ '$and': [{ 'isActive': true }, { 'age': { '$gte': 18 } }] }"
        "OR", "{ '$or': [{ 'name': 'Alice' }, { 'email': 'alice@example.com' }] }"
        "Exists", "{ 'address': { '$exists': true } }"
        "Text search", "{ '$text': { '$search': 'programming' } }"
    ]

    for (name, filter) in filters do
        printfn "  [%s]: %s" name filter

    // แสดง Aggregation Pipeline examples
    printfn "\n--- Aggregation Pipeline Examples ---"

    let pipelines = [
        "$match + $group", """
            [
                { '$match': { 'isActive': true } },
                { '$group': { '_id': '$age', 'count': { '$sum': 1 } } },
                { '$sort': { 'count': -1 } }
            ]"""
        "$lookup (join)", """
            [
                { '$lookup': {
                    'from': 'orders',
                    'localField': '_id',
                    'foreignField': 'customerId',
                    'as': 'orders'
                }},
                { '$addFields': { 'orderCount': { '$size': '$orders' } } }
            ]"""
        "$unwind + $group", """
            [
                { '$unwind': '$tags' },
                { '$group': { '_id': '$tags', 'count': { '$sum': 1 } } },
                { '$sort': { 'count': -1 } },
                { '$limit': 10 }
            ]"""
        "$facet", """
            [
                { '$facet': {
                    'byCity': [
                        { '$group': { '_id': '$address.city', 'count': { '$sum': 1 } } }
                    ],
                    'stats': [
                        { '$group': { '_id': null, 'avgAge': { '$avg': '$age' } } }
                    ]
                }}
            ]"""
    ]

    for (name, pipeline) in pipelines do
        printfn "  [%s]:" name
        printfn "    %s" (pipeline.Trim())
        printfn ""

    // แสดง Update operators
    printfn "--- Update Operators ---"
    let updateOps = [
        "$set", "{ '$set': { 'name': 'New Name' } }"
        "$unset", "{ '$unset': { 'oldField': '' } }"
        "$inc", "{ '$inc': { 'age': 1 } }"
        "$push", "{ '$push': { 'tags': 'newTag' } }"
        "$addToSet", "{ '$addToSet': { 'tags': 'uniqueTag' } }"
        "$pull", "{ '$pull': { 'tags': 'removeThis' } }"
        "$pop", "{ '$pop': { 'tags': 1 } }  // remove last"
        "$currentDate", "{ '$currentDate': { 'updatedAt': true } }"
        "Array filter", """{ '$set': { 'items.$[elem].price': 100 } }
                  arrayFilters: [{ 'elem.productId': 'abc' }]"""
    ]

    for (op, example) in updateOps do
        printfn "  [%s]: %s" op example

    // แสดง Index types
    printfn "\n--- Index Types ---"
    let indexTypes = [
        "Single field", "db.users.createIndex({ email: 1 })"
        "Compound", "db.users.createIndex({ isActive: 1, name: 1 })"
        "Unique", "db.users.createIndex({ email: 1 }, { unique: true })"
        "Text", "db.users.createIndex({ name: 'text', bio: 'text' })"
        "Geospatial", "db.places.createIndex({ location: '2dsphere' })"
        "TTL", "db.sessions.createIndex({ createdAt: 1 }, { expireAfterSeconds: 3600 })"
        "Partial", "db.users.createIndex({ email: 1 }, { partialFilterExpression: { isActive: true } })"
        "Sparse", "db.users.createIndex({ phone: 1 }, { sparse: true })"
        "Wildcard", "db.users.createIndex({ 'metadata.$**': 1 })"
        "Hashed", "db.users.createIndex({ _id: 'hashed' })  // for sharding"
    ]

    for (name, cmd) in indexTypes do
        printfn "  [%s]: %s" name cmd

    // แสดง F# type mapping issues
    printfn "\n--- F# Type Mapping Notes ---"
    let notes = [
        "Records", "ใช้ได้แต่ต้องระวัง - record fields เป็น immutable, MongoDB driver ต้องการ mutable สำหรับ deserialization"
        "Option<T>", "ต้อง register custom serializer หรือใช้ Nullable<T> แทน"
        "DU", "ต้องใช้ BsonSerializer.RegisterSerializer หรือแปลงเป็น string/int ก่อน"
        "List<T>", "แปลง BSON array ได้อัตโนมัติ"
        "Map<K,V>", "แปลงเป็น BsonDocument ได้ แต่ต้องระวัง key types"
        "DateTime", "ควรใช้ DateTime.UtcNow เสมอ, MongoDB เก็บใน UTC"
        "ObjectId", "ใช้ BsonRepresentation(BsonType.ObjectId) attribute กับ string field"
        "Classes", "แนะนำสำหรับ entities ที่ต้อง insert/update (mutable + parameterless ctor)"
    ]

    for (issue, note) in notes do
        printfn "  [%s]: %s" issue note

    printfn "\n=== Demo Complete ==="
    0
```

---

## สรุป (Summary)

MongoDB กับ F# มีสิ่งที่ต้องระวัง:

1. **Class vs Records**: MongoDB.Driver ทำงานได้ดีกับ classes มากกว่า F# records เนื่องจากต้องการ mutable properties
2. **Option types**: ต้องการ custom serializer
3. **Discriminated Unions**: ต้องแปลงเป็น string/int ก่อน serialize
4. **ObjectId**: ใช้ attribute `[BsonRepresentation(BsonType.ObjectId)]` กับ `string` fields
5. **Transactions**: ต้องใช้ replica set (ไม่ทำงานกับ standalone MongoDB)

```fsharp
// Key MongoDB operations:
// Insert: collection.InsertOneAsync(doc)
// Find: collection.Find(filter).ToListAsync()
// Update: collection.UpdateOneAsync(filter, update)
// Delete: collection.DeleteOneAsync(filter)
// Aggregate: collection.Aggregate(pipeline).ToListAsync()
// Transaction: client.StartSession() -> session.StartTransaction() -> session.CommitTransaction()
// Change stream: collection.WatchAsync()
// GridFS: bucket.UploadFromStreamAsync() / bucket.DownloadToStreamAsync()
```
