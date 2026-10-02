# Part 99 - โปรเจคจริง (Real-World Complete Project)

## บทนำ

ในบทนี้เราจะสร้าง E-Commerce API ที่สมบูรณ์ โดยใช้ Domain-Driven Design (DDD), CQRS + Event Sourcing, PostgreSQL, Redis, Giraffe, JWT Authentication, และ Background Jobs

---

## โครงสร้างโปรเจค

```
ECommerce/
├── src/
│   ├── ECommerce.Domain/          # Domain Model (DDD)
│   │   ├── Product.fs
│   │   ├── Order.fs
│   │   ├── Customer.fs
│   │   └── Events.fs
│   ├── ECommerce.Application/     # Application Layer (CQRS)
│   │   ├── Commands/
│   │   ├── Queries/
│   │   └── Handlers/
│   ├── ECommerce.Infrastructure/  # Infrastructure
│   │   ├── Database/
│   │   ├── Cache/
│   │   └── Messaging/
│   └── ECommerce.API/             # API Layer (Giraffe)
│       ├── Controllers/
│       ├── Middleware/
│       └── Program.fs
├── tests/
│   ├── ECommerce.Domain.Tests/
│   ├── ECommerce.Application.Tests/
│   └── ECommerce.API.Tests/
└── docker-compose.yml
```

---

## 1. Domain Model (DDD)

### Domain Types

```fsharp
// ECommerce.Domain/Types.fs
namespace ECommerce.Domain

open System

// Value Objects
[<Struct>]
type Money = {
    Amount: decimal
    Currency: string
}
with
    static member Create amount currency =
        if amount < 0M then Error "Amount cannot be negative"
        elif String.IsNullOrWhiteSpace currency then Error "Currency required"
        else Ok { Amount = amount; Currency = currency }
    
    static member USD amount = { Amount = amount; Currency = "USD" }
    static member THB amount = { Amount = amount; Currency = "THB" }
    
    static member (+) (a: Money, b: Money) =
        if a.Currency <> b.Currency then failwith "Cannot add different currencies"
        { a with Amount = a.Amount + b.Amount }
    
    static member (*) (m: Money, factor: decimal) =
        { m with Amount = m.Amount * factor }
    
    override this.ToString() = $"{this.Amount} {this.Currency}"

[<Struct>]
type Quantity = private Quantity of int with
    static member Create n =
        if n <= 0 then Error "Quantity must be positive"
        else Ok (Quantity n)
    member this.Value = let (Quantity n) = this in n

type ProductId = ProductId of Guid
type CustomerId = CustomerId of Guid
type OrderId = OrderId of Guid
type CategoryId = CategoryId of Guid

// Email value object
type Email = private Email of string with
    static member Create email =
        let pattern = @"^[^@\s]+@[^@\s]+\.[^@\s]+$"
        if System.Text.RegularExpressions.Regex.IsMatch(email, pattern) then
            Ok (Email (email.ToLowerInvariant()))
        else
            Error "Invalid email format"
    member this.Value = let (Email e) = this in e
    override this.ToString() = this.Value

// Phone number value object
type PhoneNumber = private PhoneNumber of string with
    static member Create phone =
        let cleaned = System.Text.RegularExpressions.Regex.Replace(phone, @"[\s\-\(\)]", "")
        if cleaned.Length >= 10 then Ok (PhoneNumber cleaned)
        else Error "Invalid phone number"
    member this.Value = let (PhoneNumber p) = this in p
```

### Product Aggregate

```fsharp
// ECommerce.Domain/Product.fs
namespace ECommerce.Domain

open System

type ProductStatus = Active | Inactive | Discontinued

type Product = {
    Id: ProductId
    Name: string
    Description: string
    Price: Money
    Stock: int
    CategoryId: CategoryId
    Status: ProductStatus
    CreatedAt: DateTime
    UpdatedAt: DateTime
}

type ProductError =
    | ProductNotFound of ProductId
    | InsufficientStock of available: int * requested: int
    | InvalidPrice of string
    | ProductAlreadyExists of string

module Product =
    let create id name description price stock categoryId =
        if String.IsNullOrWhiteSpace name then Error "Name is required"
        elif price.Amount <= 0M then Error "Price must be positive"
        elif stock < 0 then Error "Stock cannot be negative"
        else Ok {
            Id = id
            Name = name.Trim()
            Description = description
            Price = price
            Stock = stock
            CategoryId = categoryId
            Status = Active
            CreatedAt = DateTime.UtcNow
            UpdatedAt = DateTime.UtcNow
        }
    
    let updatePrice product newPrice =
        if newPrice.Amount <= 0M then Error (InvalidPrice "Price must be positive")
        else Ok { product with Price = newPrice; UpdatedAt = DateTime.UtcNow }
    
    let updateStock product quantity =
        let newStock = product.Stock + quantity
        if newStock < 0 then Error (InsufficientStock(product.Stock, -quantity))
        else Ok { product with Stock = newStock; UpdatedAt = DateTime.UtcNow }
    
    let reserveStock product (qty: Quantity) =
        let quantity = qty.Value
        if product.Stock < quantity then
            Error (InsufficientStock(product.Stock, quantity))
        elif product.Status <> Active then
            Error (ProductNotFound product.Id)
        else
            Ok { product with Stock = product.Stock - quantity; UpdatedAt = DateTime.UtcNow }
    
    let deactivate product =
        { product with Status = Inactive; UpdatedAt = DateTime.UtcNow }
    
    let discontinue product =
        { product with Status = Discontinued; UpdatedAt = DateTime.UtcNow }
```

### Order Aggregate

```fsharp
// ECommerce.Domain/Order.fs
namespace ECommerce.Domain

open System

type OrderStatus =
    | Pending
    | Confirmed
    | Processing
    | Shipped
    | Delivered
    | Cancelled
    | Refunded

type OrderLine = {
    ProductId: ProductId
    ProductName: string
    Quantity: Quantity
    UnitPrice: Money
    TotalPrice: Money
}

type ShippingAddress = {
    Street: string
    City: string
    State: string
    ZipCode: string
    Country: string
}

type Order = {
    Id: OrderId
    CustomerId: CustomerId
    Lines: OrderLine list
    Status: OrderStatus
    SubTotal: Money
    ShippingFee: Money
    TaxAmount: Money
    Total: Money
    ShippingAddress: ShippingAddress
    Notes: string option
    CreatedAt: DateTime
    UpdatedAt: DateTime
    ShippedAt: DateTime option
    DeliveredAt: DateTime option
    CancelledAt: DateTime option
}

type OrderError =
    | OrderNotFound of OrderId
    | InvalidTransition of from: OrderStatus * to': OrderStatus
    | EmptyOrder
    | OrderAlreadyCancelled

module Order =
    let private calculateTotals (lines: OrderLine list) =
        let subTotal = 
            lines |> List.fold (fun acc line -> acc + line.TotalPrice) (Money.USD 0M)
        subTotal
    
    let create id customerId lines address notes =
        if List.isEmpty lines then Error EmptyOrder
        else
            let subTotal = calculateTotals lines
            let shippingFee = 
                if subTotal.Amount > 1000M then Money.USD 0M
                else Money.USD 10M
            let taxAmount = subTotal * 0.07M
            let total = subTotal + shippingFee + taxAmount
            
            Ok {
                Id = id
                CustomerId = customerId
                Lines = lines
                Status = Pending
                SubTotal = subTotal
                ShippingFee = shippingFee
                TaxAmount = taxAmount
                Total = total
                ShippingAddress = address
                Notes = notes
                CreatedAt = DateTime.UtcNow
                UpdatedAt = DateTime.UtcNow
                ShippedAt = None
                DeliveredAt = None
                CancelledAt = None
            }
    
    let confirm order =
        match order.Status with
        | Pending -> Ok { order with Status = Confirmed; UpdatedAt = DateTime.UtcNow }
        | status -> Error (InvalidTransition(status, Confirmed))
    
    let startProcessing order =
        match order.Status with
        | Confirmed -> Ok { order with Status = Processing; UpdatedAt = DateTime.UtcNow }
        | status -> Error (InvalidTransition(status, Processing))
    
    let ship trackingNumber order =
        match order.Status with
        | Processing ->
            Ok { order with 
                Status = Shipped
                ShippedAt = Some DateTime.UtcNow
                UpdatedAt = DateTime.UtcNow }
        | status -> Error (InvalidTransition(status, Shipped))
    
    let deliver order =
        match order.Status with
        | Shipped ->
            Ok { order with
                Status = Delivered
                DeliveredAt = Some DateTime.UtcNow
                UpdatedAt = DateTime.UtcNow }
        | status -> Error (InvalidTransition(status, Delivered))
    
    let cancel reason order =
        match order.Status with
        | Cancelled -> Error OrderAlreadyCancelled
        | Delivered -> Error (InvalidTransition(Delivered, Cancelled))
        | _ ->
            Ok { order with
                Status = Cancelled
                CancelledAt = Some DateTime.UtcNow
                Notes = Some (defaultArg order.Notes "" + $"\nCancelled: {reason}")
                UpdatedAt = DateTime.UtcNow }
```

---

## 2. Domain Events

```fsharp
// ECommerce.Domain/Events.fs
namespace ECommerce.Domain

open System

// Domain Events
type DomainEvent =
    // Order events
    | OrderCreated of {| OrderId: OrderId; CustomerId: CustomerId; Total: Money; CreatedAt: DateTime |}
    | OrderConfirmed of {| OrderId: OrderId; ConfirmedAt: DateTime |}
    | OrderShipped of {| OrderId: OrderId; TrackingNumber: string; ShippedAt: DateTime |}
    | OrderDelivered of {| OrderId: OrderId; DeliveredAt: DateTime |}
    | OrderCancelled of {| OrderId: OrderId; Reason: string; CancelledAt: DateTime |}
    
    // Product events
    | ProductCreated of {| ProductId: ProductId; Name: string; Price: Money; CreatedAt: DateTime |}
    | ProductPriceUpdated of {| ProductId: ProductId; OldPrice: Money; NewPrice: Money; UpdatedAt: DateTime |}
    | ProductStockUpdated of {| ProductId: ProductId; OldStock: int; NewStock: int; UpdatedAt: DateTime |}
    | ProductDiscontinued of {| ProductId: ProductId; DiscontinuedAt: DateTime |}
    
    // Customer events
    | CustomerRegistered of {| CustomerId: CustomerId; Email: Email; RegisteredAt: DateTime |}
    | CustomerEmailVerified of {| CustomerId: CustomerId; VerifiedAt: DateTime |}
    | CustomerPasswordChanged of {| CustomerId: CustomerId; ChangedAt: DateTime |}

// Event envelope for event store
type EventEnvelope = {
    EventId: Guid
    AggregateId: Guid
    AggregateType: string
    EventType: string
    Event: DomainEvent
    OccurredAt: DateTime
    Version: int64
    Metadata: Map<string, string>
}

// Event store interface
type IEventStore =
    abstract Append: aggregateId: Guid -> events: DomainEvent list -> expectedVersion: int64 -> Async<Result<int64, string>>
    abstract ReadStream: aggregateId: Guid -> ?fromVersion: int64 -> Async<EventEnvelope list>
    abstract ReadAll: ?fromPosition: int64 -> Async<EventEnvelope list>

// Event bus interface
type IEventBus =
    abstract Publish: DomainEvent -> Async<unit>
    abstract Subscribe: eventType: string -> handler: (DomainEvent -> Async<unit>) -> unit
```

---

## 3. CQRS - Commands

```fsharp
// ECommerce.Application/Commands.fs
namespace ECommerce.Application

open ECommerce.Domain

// Commands
type CreateOrderCommand = {
    CustomerId: CustomerId
    Items: {| ProductId: ProductId; Quantity: int |} list
    ShippingAddress: ShippingAddress
    Notes: string option
}

type ConfirmOrderCommand = { OrderId: OrderId }
type ShipOrderCommand = { OrderId: OrderId; TrackingNumber: string }
type CancelOrderCommand = { OrderId: OrderId; Reason: string }

type CreateProductCommand = {
    Name: string
    Description: string
    Price: Money
    InitialStock: int
    CategoryId: CategoryId
}

type UpdateProductPriceCommand = {
    ProductId: ProductId
    NewPrice: Money
}

type RegisterCustomerCommand = {
    Email: string
    Password: string
    FirstName: string
    LastName: string
    PhoneNumber: string option
}

// Command results
type CommandResult<'T> = Async<Result<'T, string>>

// Command handlers
module OrderCommandHandlers =
    open System
    
    let handleCreateOrder
        (findProduct: ProductId -> Async<Product option>)
        (reserveStock: ProductId -> Quantity -> Async<Result<unit, ProductError>>)
        (saveOrder: Order -> Async<unit>)
        (publishEvent: DomainEvent -> Async<unit>)
        (cmd: CreateOrderCommand) : CommandResult<OrderId> =
        
        async {
            // Validate and fetch products
            let! productResults = 
                cmd.Items
                |> List.map (fun item ->
                    async {
                        let! productOpt = findProduct item.ProductId
                        return
                            match productOpt with
                            | None -> Error $"Product {item.ProductId} not found"
                            | Some product ->
                                match Quantity.Create item.Quantity with
                                | Error e -> Error e
                                | Ok qty ->
                                    Ok (product, qty)
                    })
                |> Async.Parallel
            
            // Check for errors
            let errors = productResults |> Array.choose (function Error e -> Some e | _ -> None)
            if errors.Length > 0 then
                return Error (String.concat "; " errors)
            else
            
            let productQtyPairs = productResults |> Array.choose (function Ok v -> Some v | _ -> None)
            
            // Build order lines
            let orderLines = 
                productQtyPairs
                |> Array.map (fun (product, qty) ->
                    {
                        ProductId = product.Id
                        ProductName = product.Name
                        Quantity = qty
                        UnitPrice = product.Price
                        TotalPrice = product.Price * decimal qty.Value
                    })
                |> Array.toList
            
            let orderId = OrderId (Guid.NewGuid())
            
            match Order.create orderId cmd.CustomerId orderLines cmd.ShippingAddress cmd.Notes with
            | Error OrderError.EmptyOrder -> return Error "Order must have at least one item"
            | Error e -> return Error (string e)
            | Ok order ->
                // Reserve stock
                let! stockResults =
                    productQtyPairs
                    |> Array.map (fun (product, qty) -> reserveStock product.Id qty)
                    |> Async.Parallel
                
                let stockErrors = stockResults |> Array.choose (function Error e -> Some e | _ -> None)
                if stockErrors.Length > 0 then
                    let errMsg = stockErrors |> Array.map string |> String.concat "; "
                    return Error errMsg
                else
                
                do! saveOrder order
                
                do! publishEvent (OrderCreated {|
                    OrderId = order.Id
                    CustomerId = order.CustomerId
                    Total = order.Total
                    CreatedAt = order.CreatedAt
                |})
                
                return Ok orderId
        }
    
    let handleConfirmOrder
        (findOrder: OrderId -> Async<Order option>)
        (saveOrder: Order -> Async<unit>)
        (publishEvent: DomainEvent -> Async<unit>)
        (cmd: ConfirmOrderCommand) : CommandResult<unit> =
        
        async {
            let! orderOpt = findOrder cmd.OrderId
            match orderOpt with
            | None -> return Error "Order not found"
            | Some order ->
                match Order.confirm order with
                | Error e -> return Error (string e)
                | Ok confirmedOrder ->
                    do! saveOrder confirmedOrder
                    do! publishEvent (OrderConfirmed {|
                        OrderId = confirmedOrder.Id
                        ConfirmedAt = DateTime.UtcNow
                    |})
                    return Ok ()
        }
```

---

## 4. CQRS - Queries

```fsharp
// ECommerce.Application/Queries.fs
namespace ECommerce.Application

open ECommerce.Domain
open System

// Query models (read models)
type ProductSummary = {
    Id: ProductId
    Name: string
    Price: Money
    Stock: int
    Status: ProductStatus
    CategoryName: string
}

type OrderSummary = {
    Id: OrderId
    CustomerId: CustomerId
    CustomerName: string
    ItemCount: int
    Total: Money
    Status: OrderStatus
    CreatedAt: DateTime
}

type OrderDetails = {
    Id: OrderId
    Customer: {| Id: CustomerId; Name: string; Email: Email |}
    Lines: {| ProductName: string; Quantity: int; UnitPrice: Money; TotalPrice: Money |} list
    SubTotal: Money
    ShippingFee: Money
    TaxAmount: Money
    Total: Money
    Status: OrderStatus
    ShippingAddress: ShippingAddress
    Notes: string option
    CreatedAt: DateTime
    ShippedAt: DateTime option
    DeliveredAt: DateTime option
}

// Query types
type GetProductsQuery = {
    CategoryId: CategoryId option
    Status: ProductStatus option
    MinPrice: decimal option
    MaxPrice: decimal option
    SearchTerm: string option
    Page: int
    PageSize: int
    SortBy: string
    SortDescending: bool
}

type GetOrdersQuery = {
    CustomerId: CustomerId option
    Status: OrderStatus option
    FromDate: DateTime option
    ToDate: DateTime option
    Page: int
    PageSize: int
}

type PagedResult<'T> = {
    Items: 'T list
    TotalCount: int
    Page: int
    PageSize: int
    TotalPages: int
}

// Query handlers (use read models/projections)
module OrderQueryHandlers =
    
    let handleGetOrderDetails
        (getOrderById: OrderId -> Async<OrderDetails option>)
        (getFromCache: string -> Async<OrderDetails option>)
        (setCache: string -> OrderDetails -> Async<unit>)
        orderId =
        
        async {
            let cacheKey = $"order:{orderId}"
            
            let! cached = getFromCache cacheKey
            match cached with
            | Some details -> return Some details
            | None ->
                let! details = getOrderById orderId
                match details with
                | None -> return None
                | Some d ->
                    do! setCache cacheKey d
                    return Some d
        }
    
    let handleGetOrders
        (getOrders: GetOrdersQuery -> Async<PagedResult<OrderSummary>>)
        query =
        
        async {
            return! getOrders query
        }
```

---

## 5. Infrastructure - Database

```fsharp
// ECommerce.Infrastructure/Database/PostgresEventStore.fs
namespace ECommerce.Infrastructure.Database

open ECommerce.Domain
open Npgsql
open Npgsql.FSharp
open System

// Event store using PostgreSQL
type PostgresEventStore(connectionString: string) =
    
    let serialize event = System.Text.Json.JsonSerializer.Serialize(event)
    let deserialize<'T> json = System.Text.Json.JsonSerializer.Deserialize<'T>(json)
    
    interface IEventStore with
        member _.Append aggregateId events expectedVersion = async {
            try
                let! result =
                    connectionString
                    |> Sql.connect
                    |> Sql.executeTransactionAsync [
                        for i, event in events |> List.indexed do
                            let version = expectedVersion + int64 i + 1L
                            let eventType = event.GetType().Name
                            
                            yield """
                                INSERT INTO domain_events 
                                    (event_id, aggregate_id, aggregate_type, event_type, event_data, version, occurred_at)
                                VALUES 
                                    (@eventId, @aggregateId, @aggregateType, @eventType, @eventData::jsonb, @version, @occurredAt)
                            """, [
                                "@eventId", Sql.uuid (Guid.NewGuid())
                                "@aggregateId", Sql.uuid aggregateId
                                "@aggregateType", Sql.string "Order"  // simplified
                                "@eventType", Sql.string eventType
                                "@eventData", Sql.string (serialize event)
                                "@version", Sql.int64 version
                                "@occurredAt", Sql.timestamptz DateTime.UtcNow
                            ]
                    ]
                
                return Ok expectedVersion
            with ex ->
                return Error ex.Message
        }
        
        member _.ReadStream aggregateId ?fromVersion = async {
            let minVersion = defaultArg fromVersion 0L
            
            let! events =
                connectionString
                |> Sql.connect
                |> Sql.query """
                    SELECT event_id, aggregate_id, aggregate_type, event_type, 
                           event_data, version, occurred_at
                    FROM domain_events
                    WHERE aggregate_id = @aggregateId AND version >= @fromVersion
                    ORDER BY version ASC
                """
                |> Sql.parameters [
                    "@aggregateId", Sql.uuid aggregateId
                    "@fromVersion", Sql.int64 minVersion
                ]
                |> Sql.executeAsync (fun read -> {
                    EventId = read.uuid "event_id"
                    AggregateId = read.uuid "aggregate_id"
                    AggregateType = read.string "aggregate_type"
                    EventType = read.string "event_type"
                    Event = deserialize<DomainEvent> (read.string "event_data")
                    OccurredAt = read.dateTime "occurred_at"
                    Version = read.int64 "version"
                    Metadata = Map.empty
                })
            
            return events
        }
        
        member _.ReadAll ?fromPosition = async {
            let minPosition = defaultArg fromPosition 0L
            
            return! 
                connectionString
                |> Sql.connect
                |> Sql.query """
                    SELECT * FROM domain_events
                    WHERE global_position >= @fromPosition
                    ORDER BY global_position ASC
                    LIMIT 1000
                """
                |> Sql.parameters ["@fromPosition", Sql.int64 minPosition]
                |> Sql.executeAsync (fun read -> {
                    EventId = read.uuid "event_id"
                    AggregateId = read.uuid "aggregate_id"
                    AggregateType = read.string "aggregate_type"
                    EventType = read.string "event_type"
                    Event = deserialize<DomainEvent> (read.string "event_data")
                    OccurredAt = read.dateTime "occurred_at"
                    Version = read.int64 "version"
                    Metadata = Map.empty
                })
        }

// Product repository
type PostgresProductRepository(connectionString: string) =
    
    member _.FindById (ProductId id) = async {
        let! results =
            connectionString
            |> Sql.connect
            |> Sql.query "SELECT * FROM products WHERE id = @id"
            |> Sql.parameters ["@id", Sql.uuid id]
            |> Sql.executeAsync (fun read -> {
                Id = ProductId (read.uuid "id")
                Name = read.string "name"
                Description = read.string "description"
                Price = { Amount = read.decimal "price"; Currency = read.string "currency" }
                Stock = read.int "stock"
                CategoryId = CategoryId (read.uuid "category_id")
                Status = 
                    match read.string "status" with
                    | "Active" -> Active
                    | "Inactive" -> Inactive
                    | _ -> Discontinued
                CreatedAt = read.dateTime "created_at"
                UpdatedAt = read.dateTime "updated_at"
            })
        
        return results |> List.tryHead
    }
    
    member _.Save (product: Product) = async {
        let (ProductId id) = product.Id
        let (CategoryId catId) = product.CategoryId
        
        let! _ =
            connectionString
            |> Sql.connect
            |> Sql.query """
                INSERT INTO products (id, name, description, price, currency, stock, category_id, status, created_at, updated_at)
                VALUES (@id, @name, @description, @price, @currency, @stock, @categoryId, @status, @createdAt, @updatedAt)
                ON CONFLICT (id) DO UPDATE SET
                    name = EXCLUDED.name,
                    description = EXCLUDED.description,
                    price = EXCLUDED.price,
                    stock = EXCLUDED.stock,
                    status = EXCLUDED.status,
                    updated_at = EXCLUDED.updated_at
            """
            |> Sql.parameters [
                "@id", Sql.uuid id
                "@name", Sql.string product.Name
                "@description", Sql.string product.Description
                "@price", Sql.decimal product.Price.Amount
                "@currency", Sql.string product.Price.Currency
                "@stock", Sql.int product.Stock
                "@categoryId", Sql.uuid catId
                "@status", Sql.string (string product.Status)
                "@createdAt", Sql.timestamptz product.CreatedAt
                "@updatedAt", Sql.timestamptz product.UpdatedAt
            ]
            |> Sql.executeNonQueryAsync
        
        return ()
    }
```

---

## 6. Infrastructure - Redis Cache

```fsharp
// ECommerce.Infrastructure/Cache/RedisCache.fs
namespace ECommerce.Infrastructure.Cache

open StackExchange.Redis
open System.Text.Json

type RedisCache(connectionString: string) =
    let redis = ConnectionMultiplexer.Connect(connectionString)
    let db = redis.GetDatabase()
    
    let serialize<'T> (value: 'T) = JsonSerializer.Serialize(value)
    let deserialize<'T> (json: string) = JsonSerializer.Deserialize<'T>(json)
    
    member _.Get<'T> (key: string) : Async<'T option> = async {
        let! value = db.StringGetAsync(key) |> Async.AwaitTask
        if value.IsNull then return None
        else return Some (deserialize<'T> (string value))
    }
    
    member _.Set<'T> (key: string) (value: 'T) (expiry: System.TimeSpan option) : Async<unit> = async {
        let json = serialize value
        let exp = expiry |> Option.map (fun e -> System.Nullable e) |> Option.defaultValue (System.Nullable())
        let! _ = db.StringSetAsync(key, json, exp) |> Async.AwaitTask
        return ()
    }
    
    member _.Delete (key: string) : Async<unit> = async {
        let! _ = db.KeyDeleteAsync(key) |> Async.AwaitTask
        return ()
    }
    
    member _.Exists (key: string) : Async<bool> = async {
        return! db.KeyExistsAsync(key) |> Async.AwaitTask
    }
    
    member _.Increment (key: string) (by: int64) : Async<int64> = async {
        return! db.StringIncrementAsync(key, by) |> Async.AwaitTask
    }
    
    member _.SetHash (key: string) (field: string) (value: string) : Async<unit> = async {
        let! _ = db.HashSetAsync(key, field, value) |> Async.AwaitTask
        return ()
    }
    
    member _.GetHash (key: string) (field: string) : Async<string option> = async {
        let! value = db.HashGetAsync(key, field) |> Async.AwaitTask
        if value.IsNull then return None
        else return Some (string value)
    }
    
    // Rate limiting using Redis
    member this.CheckRateLimit (key: string) (maxRequests: int) (windowSeconds: int) = async {
        let count = db.StringIncrement(key)
        if count = 1L then
            db.KeyExpire(key, System.TimeSpan.FromSeconds(float windowSeconds)) |> ignore
        return count <= int64 maxRequests
    }
    
    interface System.IDisposable with
        member _.Dispose() = redis.Dispose()
```

---

## 7. API Layer (Giraffe)

```fsharp
// ECommerce.API/Controllers/OrderController.fs
namespace ECommerce.API

open Giraffe
open Microsoft.AspNetCore.Http
open System.Text.Json
open ECommerce.Application
open ECommerce.Domain

// Request/Response DTOs
type CreateOrderRequest = {
    Items: {| ProductId: string; Quantity: int |} list
    ShippingAddress: {|
        Street: string
        City: string
        State: string
        ZipCode: string
        Country: string
    |}
    Notes: string option
}

type CreateOrderResponse = {
    OrderId: string
    Total: decimal
    Currency: string
}

// Error response
type ErrorResponse = {
    Error: string
    Details: string list
}

// Helper functions
let inline sendJson<'T> (statusCode: int) (body: 'T) : HttpHandler =
    setStatusCode statusCode >=> json body

let sendError statusCode message =
    sendJson statusCode { Error = message; Details = [] }

let sendErrors statusCode message details =
    sendJson statusCode { Error = message; Details = details }

// Order handlers
module OrderHandlers =
    
    let getOrder (orderId: string) : HttpHandler =
        fun next ctx -> task {
            let orderService = ctx.GetService<IOrderQueryService>()
            
            match System.Guid.TryParse(orderId) with
            | false, _ ->
                return! sendError 400 "Invalid order ID" next ctx
            | true, guid ->
                let! orderOpt = orderService.GetOrderDetails (OrderId guid)
                
                match orderOpt with
                | None -> return! sendError 404 "Order not found" next ctx
                | Some order -> return! sendJson 200 order next ctx
        }
    
    let getOrders : HttpHandler =
        fun next ctx -> task {
            let orderService = ctx.GetService<IOrderQueryService>()
            
            let query = {
                CustomerId = None
                Status = None
                FromDate = None
                ToDate = None
                Page = defaultArg (ctx.TryGetQueryStringValue "page" |> Option.bind (fun s -> match System.Int32.TryParse(s) with true, n -> Some n | _ -> None)) 1
                PageSize = min 100 (defaultArg (ctx.TryGetQueryStringValue "pageSize" |> Option.bind (fun s -> match System.Int32.TryParse(s) with true, n -> Some n | _ -> None)) 20)
            }
            
            let! orders = orderService.GetOrders query
            return! sendJson 200 orders next ctx
        }
    
    let createOrder : HttpHandler =
        fun next ctx -> task {
            let! request = ctx.BindJsonAsync<CreateOrderRequest>()
            let orderService = ctx.GetService<IOrderCommandService>()
            
            // Get current user from JWT claims
            let customerId = 
                ctx.User.FindFirst("sub")
                |> Option.ofObj
                |> Option.map (fun c -> CustomerId (System.Guid.Parse(c.Value)))
            
            match customerId with
            | None -> return! sendError 401 "Unauthorized" next ctx
            | Some custId ->
                let address = {
                    Street = request.ShippingAddress.Street
                    City = request.ShippingAddress.City
                    State = request.ShippingAddress.State
                    ZipCode = request.ShippingAddress.ZipCode
                    Country = request.ShippingAddress.Country
                }
                
                let cmd : CreateOrderCommand = {
                    CustomerId = custId
                    Items = request.Items |> List.map (fun i -> {|
                        ProductId = ProductId (System.Guid.Parse(i.ProductId))
                        Quantity = i.Quantity
                    |})
                    ShippingAddress = address
                    Notes = request.Notes
                }
                
                let! result = orderService.CreateOrder cmd
                
                match result with
                | Error msg -> return! sendError 400 msg next ctx
                | Ok (OrderId id) ->
                    return! sendJson 201 {| OrderId = string id |} next ctx
        }
    
    let cancelOrder (orderId: string) : HttpHandler =
        fun next ctx -> task {
            match System.Guid.TryParse(orderId) with
            | false, _ -> return! sendError 400 "Invalid order ID" next ctx
            | true, guid ->
                let! body = ctx.BindJsonAsync<{| Reason: string |}>()
                let orderService = ctx.GetService<IOrderCommandService>()
                
                let cmd = { 
                    OrderId = OrderId guid
                    Reason = body.Reason
                }
                
                let! result = orderService.CancelOrder cmd
                
                match result with
                | Error msg -> return! sendError 400 msg next ctx
                | Ok () -> return! sendJson 200 {| Message = "Order cancelled" |} next ctx
        }

// Product handlers
module ProductHandlers =
    
    let getProducts : HttpHandler =
        fun next ctx -> task {
            let productService = ctx.GetService<IProductQueryService>()
            
            let query = {
                CategoryId = None
                Status = Some Active
                MinPrice = None
                MaxPrice = None
                SearchTerm = ctx.TryGetQueryStringValue "q"
                Page = 1
                PageSize = 20
                SortBy = "name"
                SortDescending = false
            }
            
            let! products = productService.GetProducts query
            return! sendJson 200 products next ctx
        }
    
    let getProduct (productId: string) : HttpHandler =
        fun next ctx -> task {
            let productService = ctx.GetService<IProductQueryService>()
            
            match System.Guid.TryParse(productId) with
            | false, _ -> return! sendError 400 "Invalid product ID" next ctx
            | true, guid ->
                let! productOpt = productService.GetProductById (ProductId guid)
                
                match productOpt with
                | None -> return! sendError 404 "Product not found" next ctx
                | Some p -> return! sendJson 200 p next ctx
        }

// Routes
let webApp : HttpHandler =
    choose [
        GET >=> route "/health" >=> sendJson 200 {| Status = "healthy" |}
        
        subRoute "/api/v1" (
            choose [
                // Orders
                GET >=> routef "/orders/%s" OrderHandlers.getOrder
                GET >=> route "/orders" >=> OrderHandlers.getOrders
                POST >=> route "/orders" >=> OrderHandlers.createOrder
                POST >=> routef "/orders/%s/cancel" OrderHandlers.cancelOrder
                
                // Products
                GET >=> route "/products" >=> ProductHandlers.getProducts
                GET >=> routef "/products/%s" ProductHandlers.getProduct
                
                setStatusCode 404 >=> sendError 404 "Not Found"
            ])
    ]
```

---

## 8. Program.fs - Application Entry Point

```fsharp
// ECommerce.API/Program.fs
module Program

open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Giraffe
open ECommerce.Infrastructure.Database
open ECommerce.Infrastructure.Cache

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    
    // Configuration
    let config = builder.Configuration
    let connectionString = config.GetConnectionString("Database")
    let redisConnectionString = config.GetConnectionString("Redis")
    
    // Services
    builder.Services.AddGiraffe() |> ignore
    
    // Infrastructure
    builder.Services.AddSingleton<PostgresProductRepository>(
        fun _ -> PostgresProductRepository(connectionString))
    |> ignore
    
    builder.Services.AddSingleton<RedisCache>(
        fun _ -> new RedisCache(redisConnectionString))
    |> ignore
    
    // Application services
    builder.Services.AddScoped<IOrderCommandService, OrderCommandService>() |> ignore
    builder.Services.AddScoped<IOrderQueryService, OrderQueryService>() |> ignore
    builder.Services.AddScoped<IProductQueryService, ProductQueryService>() |> ignore
    
    // JWT Authentication
    builder.Services
        .AddAuthentication(Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(fun opts ->
            let key = System.Text.Encoding.UTF8.GetBytes(config["Jwt:Secret"])
            opts.TokenValidationParameters <- Microsoft.IdentityModel.Tokens.TokenValidationParameters(
                ValidateIssuer = true,
                ValidIssuer = config["Jwt:Issuer"],
                ValidateAudience = true,
                ValidAudience = config["Jwt:Audience"],
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = Microsoft.IdentityModel.Tokens.SymmetricSecurityKey(key),
                ClockSkew = System.TimeSpan.Zero))
    |> ignore
    
    // Authorization
    builder.Services.AddAuthorization() |> ignore
    
    // CORS
    builder.Services.AddCors(fun opts ->
        opts.AddPolicy("DefaultPolicy", fun policy ->
            policy.WithOrigins("https://myapp.com", "http://localhost:3000")
                  .AllowAnyMethod()
                  .AllowAnyHeader()
                  .AllowCredentials()
            |> ignore))
    |> ignore
    
    // Health checks
    builder.Services
        .AddHealthChecks()
        .AddNpgsql(connectionString)
        .AddRedis(redisConnectionString)
    |> ignore
    
    // Background jobs with Hangfire
    builder.Services.AddHangfire(fun config ->
        config.UsePostgreSqlStorage(connectionString) |> ignore)
    |> ignore
    builder.Services.AddHangfireServer() |> ignore
    
    // Build app
    let app = builder.Build()
    
    // Middleware pipeline
    app.UseCors("DefaultPolicy") |> ignore
    app.UseAuthentication() |> ignore
    app.UseAuthorization() |> ignore
    app.UseGiraffe(webApp)
    
    // Health check endpoints
    app.MapHealthChecks("/health") |> ignore
    app.MapHealthChecks("/health/ready") |> ignore
    
    // Hangfire Dashboard
    app.UseHangfireDashboard("/jobs") |> ignore
    
    app.Run()
    0
```

---

## 9. Background Jobs (Hangfire)

```fsharp
// ECommerce.Application/BackgroundJobs.fs
namespace ECommerce.Application

open Hangfire

type OrderNotificationJob(emailService: IEmailService, smsService: ISmsService) =
    
    [<AutomaticRetry(Attempts = 3)>]
    member _.SendOrderConfirmation(orderId: string) = task {
        // Fetch order
        // Send email notification
        // Send SMS if phone number available
        printfn "Sending order confirmation for %s" orderId
    }
    
    [<AutomaticRetry(Attempts = 3)>]
    member _.SendShippingNotification(orderId: string, trackingNumber: string) = task {
        printfn "Sending shipping notification for %s, tracking: %s" orderId trackingNumber
    }

type StockReplenishmentJob(productRepo: IProductRepository, notificationService: INotificationService) =
    
    // Run daily at 9 AM
    [<RecurringJob("StockCheck", Cron.Daily, TimeZone = "UTC")>]
    member _.CheckLowStock() = task {
        let! lowStockProducts = productRepo.GetLowStockProducts(threshold = 10)
        
        for product in lowStockProducts do
            do! notificationService.NotifyLowStock(product)
    }

// Register background jobs
let registerJobs () =
    RecurringJob.AddOrUpdate<StockReplenishmentJob>(
        "stock-check",
        fun j -> j.CheckLowStock() :> System.Threading.Tasks.Task,
        Cron.Daily)
```

---

## 10. Testing Strategy

```fsharp
// tests/ECommerce.Domain.Tests/OrderTests.fs
module OrderTests

open Xunit
open FsUnit.Xunit
open ECommerce.Domain
open System

let private createTestProduct () = {
    Id = ProductId (Guid.NewGuid())
    Name = "Test Product"
    Description = "A test product"
    Price = Money.USD 100.0M
    Stock = 10
    CategoryId = CategoryId (Guid.NewGuid())
    Status = Active
    CreatedAt = DateTime.UtcNow
    UpdatedAt = DateTime.UtcNow
}

let private createTestAddress () = {
    Street = "123 Test St"
    City = "Bangkok"
    State = "Bangkok"
    ZipCode = "10100"
    Country = "Thailand"
}

[<Fact>]
let ``Order.create should create order with correct total`` () =
    let product = createTestProduct()
    let qty = Quantity.Create 2 |> Result.defaultValue (Quantity.Create 1 |> Result.defaultValue (failwith ""))
    
    let line = {
        ProductId = product.Id
        ProductName = product.Name
        Quantity = qty
        UnitPrice = product.Price
        TotalPrice = product.Price * decimal qty.Value
    }
    
    let result = Order.create (OrderId (Guid.NewGuid())) (CustomerId (Guid.NewGuid())) [line] (createTestAddress()) None
    
    match result with
    | Error e -> failwith $"Expected Ok but got Error: {e}"
    | Ok order ->
        order.SubTotal.Amount |> should equal 200.0M
        order.Status |> should equal Pending

[<Fact>]
let ``Order.confirm should transition from Pending to Confirmed`` () =
    let product = createTestProduct()
    let qty = Quantity.Create 1 |> Result.defaultValue (failwith "")
    let line = { ProductId = product.Id; ProductName = product.Name; Quantity = qty; UnitPrice = product.Price; TotalPrice = product.Price }
    
    let order = Order.create (OrderId (Guid.NewGuid())) (CustomerId (Guid.NewGuid())) [line] (createTestAddress()) None |> Result.defaultValue (failwith "")
    
    let result = Order.confirm order
    
    result |> should be (ofCase <@ Ok @>)

[<Fact>]
let ``Order.cancel should prevent cancellation of delivered order`` () =
    let product = createTestProduct()
    let qty = Quantity.Create 1 |> Result.defaultValue (failwith "")
    let line = { ProductId = product.Id; ProductName = product.Name; Quantity = qty; UnitPrice = product.Price; TotalPrice = product.Price }
    
    let order = Order.create (OrderId (Guid.NewGuid())) (CustomerId (Guid.NewGuid())) [line] (createTestAddress()) None |> Result.defaultValue (failwith "")
    let deliveredOrder = { order with Status = Delivered }
    
    let result = Order.cancel "test reason" deliveredOrder
    
    match result with
    | Ok _ -> failwith "Expected Error"
    | Error (InvalidTransition(Delivered, Cancelled)) -> ()  // Expected
    | Error e -> failwith $"Unexpected error: {e}"

// Integration tests
[<Fact>]
let ``CreateOrder handler should reserve stock`` () = task {
    // Arrange
    let product = createTestProduct()
    let mutable stockReserved = false
    
    let findProduct id =
        if id = product.Id then async { return Some product }
        else async { return None }
    
    let reserveStock productId qty = async {
        stockReserved <- true
        return Ok ()
    }
    
    let saveOrder _ = async { return () }
    let publishEvent _ = async { return () }
    
    let cmd : CreateOrderCommand = {
        CustomerId = CustomerId (Guid.NewGuid())
        Items = [| {| ProductId = product.Id; Quantity = 1 |} |] |> Array.toList
        ShippingAddress = createTestAddress()
        Notes = None
    }
    
    // Act
    let! result = OrderCommandHandlers.handleCreateOrder findProduct reserveStock saveOrder publishEvent cmd
    
    // Assert
    result |> should be (ofCase <@ Ok @>)
    stockReserved |> should be True
}
```

---

## 11. Database Migrations

```sql
-- migrations/001_initial_schema.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";

-- Products
CREATE TABLE categories (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    description TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE products (
    id UUID PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    price DECIMAL(10, 2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    stock INTEGER NOT NULL DEFAULT 0,
    category_id UUID NOT NULL REFERENCES categories(id),
    status VARCHAR(20) NOT NULL DEFAULT 'Active',
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_status ON products(status);

-- Customers
CREATE TABLE customers (
    id UUID PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    phone_number VARCHAR(20),
    password_hash VARCHAR(255) NOT NULL,
    is_email_verified BOOLEAN NOT NULL DEFAULT FALSE,
    failed_login_attempts INTEGER NOT NULL DEFAULT 0,
    locked_until TIMESTAMPTZ,
    last_login_at TIMESTAMPTZ,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Orders
CREATE TABLE orders (
    id UUID PRIMARY KEY,
    customer_id UUID NOT NULL REFERENCES customers(id),
    status VARCHAR(20) NOT NULL DEFAULT 'Pending',
    subtotal DECIMAL(12, 2) NOT NULL,
    shipping_fee DECIMAL(10, 2) NOT NULL DEFAULT 0,
    tax_amount DECIMAL(10, 2) NOT NULL DEFAULT 0,
    total DECIMAL(12, 2) NOT NULL,
    currency CHAR(3) NOT NULL DEFAULT 'USD',
    shipping_street VARCHAR(255),
    shipping_city VARCHAR(100),
    shipping_state VARCHAR(100),
    shipping_zip_code VARCHAR(20),
    shipping_country VARCHAR(100),
    notes TEXT,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    shipped_at TIMESTAMPTZ,
    delivered_at TIMESTAMPTZ,
    cancelled_at TIMESTAMPTZ
);

CREATE INDEX idx_orders_customer ON orders(customer_id);
CREATE INDEX idx_orders_status ON orders(status);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

CREATE TABLE order_lines (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    order_id UUID NOT NULL REFERENCES orders(id),
    product_id UUID NOT NULL REFERENCES products(id),
    product_name VARCHAR(200) NOT NULL,
    quantity INTEGER NOT NULL,
    unit_price DECIMAL(10, 2) NOT NULL,
    total_price DECIMAL(12, 2) NOT NULL
);

-- Event Store
CREATE TABLE domain_events (
    global_position BIGSERIAL,
    event_id UUID NOT NULL UNIQUE,
    aggregate_id UUID NOT NULL,
    aggregate_type VARCHAR(100) NOT NULL,
    event_type VARCHAR(200) NOT NULL,
    event_data JSONB NOT NULL,
    version BIGINT NOT NULL,
    occurred_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (aggregate_id, version)
);

CREATE INDEX idx_events_aggregate ON domain_events(aggregate_id, version);
CREATE INDEX idx_events_global ON domain_events(global_position);
CREATE INDEX idx_events_type ON domain_events(event_type);
```

---

## สรุป

โปรเจค E-Commerce นี้ใช้หลักการสำคัญ:

1. **DDD**: Domain model ที่ชัดเจน, Aggregates, Value Objects
2. **CQRS**: แยก read/write operations
3. **Event Sourcing**: บันทึก events แทน state
4. **Clean Architecture**: แยก layers ชัดเจน
5. **Type Safety**: F# type system ป้องกัน bugs
6. **Testing**: Domain logic ทดสอบง่าย
7. **Infrastructure**: PostgreSQL, Redis, Hangfire

สิ่งที่ทำให้ F# เหมาะสำหรับ production:
- **Immutability by default**: ลด bugs จาก state mutation
- **Exhaustive pattern matching**: ไม่มี missing cases
- **Railway-oriented programming**: Error handling ที่ชัดเจน
- **Type providers**: Type-safe access to external data

---

*ต่อไป: Part 100 - หัวข้อขั้นสูง (Advanced Topics and Next Steps)*
