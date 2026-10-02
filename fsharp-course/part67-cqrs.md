# Part 67 - CQRS Pattern

## บทนำ (Introduction)

CQRS (Command Query Responsibility Segregation) เป็น architectural pattern ที่แยก operations ที่เปลี่ยนแปลงข้อมูล (Commands) ออกจาก operations ที่อ่านข้อมูล (Queries)

**หลักการพื้นฐาน**:
- **Command**: เปลี่ยนแปลง state ของระบบ (Write side)
- **Query**: อ่านข้อมูลโดยไม่เปลี่ยน state (Read side)

CQRS ทำให้:
- Scale read และ write แยกกันได้
- Optimize query model สำหรับ read-heavy operations
- ชัดเจนในการแบ่ง responsibilities

---

## 1. Core Concepts

```fsharp
// CoreTypes.fs
module CoreTypes

open System

// ========================================
// 1.1 Command และ Query marker interfaces
// ========================================

// Command: ทำให้เกิดการเปลี่ยนแปลง
[<Interface>]
type ICommand = interface end

// Query: อ่านข้อมูล ไม่เปลี่ยนแปลง
[<Interface>]
type IQuery<'TResult> = interface end

// Command Result
type CommandResult<'T> =
    | Success of 'T
    | Failure of string list  // validation errors

// ========================================
// 1.2 Domain Events
// ========================================

[<Interface>]
type IDomainEvent =
    abstract member OccurredAt: DateTime
    abstract member EventId: Guid

// Base domain event
type DomainEventBase() =
    interface IDomainEvent with
        member _.OccurredAt = DateTime.UtcNow
        member _.EventId = Guid.NewGuid()

// ========================================
// 1.3 Handler interfaces
// ========================================

[<Interface>]
type ICommandHandler<'TCommand, 'TResult when 'TCommand :> ICommand> =
    abstract member HandleAsync: 'TCommand -> System.Threading.Tasks.Task<CommandResult<'TResult>>

[<Interface>]
type IQueryHandler<'TQuery, 'TResult when 'TQuery :> IQuery<'TResult>> =
    abstract member HandleAsync: 'TQuery -> System.Threading.Tasks.Task<'TResult>
```

---

## 2. Write Models (Commands)

```fsharp
// Commands.fs
module Commands

open System
open CoreTypes

// ========================================
// 2.1 User commands
// ========================================

type RegisterUserCommand = {
    Name: string
    Email: string
    Password: string
}
with interface ICommand

type UpdateUserProfileCommand = {
    UserId: int
    Name: string
    Bio: string option
}
with interface ICommand

type DeactivateUserCommand = {
    UserId: int
    Reason: string
}
with interface ICommand

// ========================================
// 2.2 Product commands
// ========================================

type CreateProductCommand = {
    Name: string
    Price: decimal
    Stock: int
    CategoryId: int
    Description: string option
}
with interface ICommand

type UpdateProductPriceCommand = {
    ProductId: int
    NewPrice: decimal
    Reason: string
}
with interface ICommand

type RestockProductCommand = {
    ProductId: int
    Quantity: int
    Supplier: string option
}
with interface ICommand

// ========================================
// 2.3 Order commands
// ========================================

type PlaceOrderCommand = {
    CustomerId: int
    Items: OrderItem list
    ShippingAddress: Address
}
with interface ICommand

and OrderItem = {
    ProductId: int
    Quantity: int
}

and Address = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

type CancelOrderCommand = {
    OrderId: int
    Reason: string
    RequestedBy: int  // userId
}
with interface ICommand

type ShipOrderCommand = {
    OrderId: int
    TrackingNumber: string
    Carrier: string
}
with interface ICommand

// ========================================
// 2.4 Write Model (ใช้ใน Command handlers)
// ========================================

type WriteUser = {
    Id: int
    Name: string
    Email: string
    PasswordHash: string
    IsActive: bool
    CreatedAt: DateTime
}

type WriteProduct = {
    Id: int
    Name: string
    Price: decimal
    Stock: int
    CategoryId: int
    Description: string option
    IsActive: bool
}

type WriteOrder = {
    Id: int
    CustomerId: int
    Items: WriteOrderItem list
    TotalAmount: decimal
    Status: string
    TrackingNumber: string option
    CreatedAt: DateTime
}

and WriteOrderItem = {
    ProductId: int
    Quantity: int
    UnitPrice: decimal
}

// ========================================
// 2.5 Domain Events (ผล output จาก commands)
// ========================================

type UserRegisteredEvent = {
    UserId: int
    Name: string
    Email: string
    RegisteredAt: DateTime
}
with interface IDomainEvent with
    member _.OccurredAt = DateTime.UtcNow
    member _.EventId = Guid.NewGuid()

type OrderPlacedEvent = {
    OrderId: int
    CustomerId: int
    TotalAmount: decimal
    ItemCount: int
    PlacedAt: DateTime
}
with interface IDomainEvent with
    member _.OccurredAt = DateTime.UtcNow
    member _.EventId = Guid.NewGuid()

type ProductPriceChangedEvent = {
    ProductId: int
    OldPrice: decimal
    NewPrice: decimal
    Reason: string
    ChangedAt: DateTime
}
with interface IDomainEvent with
    member _.OccurredAt = DateTime.UtcNow
    member _.EventId = Guid.NewGuid()

type StockUpdatedEvent = {
    ProductId: int
    OldStock: int
    NewStock: int
    UpdatedAt: DateTime
}
with interface IDomainEvent with
    member _.OccurredAt = DateTime.UtcNow
    member _.EventId = Guid.NewGuid()
```

---

## 3. Read Models (Queries)

```fsharp
// Queries.fs
module Queries

open CoreTypes
open System

// ========================================
// 3.1 Query types
// ========================================

type GetUserByIdQuery = {
    UserId: int
}
with interface IQuery<UserReadModel option>

type GetUsersQuery = {
    Page: int
    PageSize: int
    SearchTerm: string option
    ActiveOnly: bool
}
with interface IQuery<PagedResult<UserReadModel>>

type GetProductByIdQuery = {
    ProductId: int
}
with interface IQuery<ProductReadModel option>

type GetProductsQuery = {
    CategoryId: int option
    MinPrice: decimal option
    MaxPrice: decimal option
    SearchTerm: string option
    InStockOnly: bool
    SortBy: string
    SortDescending: bool
    Page: int
    PageSize: int
}
with interface IQuery<PagedResult<ProductReadModel>>

type GetOrderByIdQuery = {
    OrderId: int
}
with interface IQuery<OrderReadModel option>

type GetOrdersByCustomerQuery = {
    CustomerId: int
    Page: int
    PageSize: int
}
with interface IQuery<PagedResult<OrderReadModel>>

type GetDashboardQuery = {
    UserId: int option  // None = admin dashboard
}
with interface IQuery<DashboardReadModel>

// ========================================
// 3.2 Read Models (optimized for reading)
// ========================================

and UserReadModel = {
    Id: int
    Name: string
    Email: string
    IsActive: bool
    CreatedAt: DateTime
    OrderCount: int
    TotalSpent: decimal
}

and ProductReadModel = {
    Id: int
    Name: string
    Price: decimal
    OriginalPrice: decimal option
    DiscountPercent: float option
    Stock: int
    IsAvailable: bool
    CategoryName: string
    Tags: string list
    RatingAverage: float option
    ReviewCount: int
}

and OrderReadModel = {
    Id: int
    CustomerName: string
    CustomerEmail: string
    Items: OrderItemReadModel list
    TotalAmount: decimal
    Status: string
    TrackingNumber: string option
    ShippingAddress: string
    CreatedAt: DateTime
    EstimatedDelivery: DateTime option
}

and OrderItemReadModel = {
    ProductName: string
    Quantity: int
    UnitPrice: decimal
    TotalPrice: decimal
    ImageUrl: string option
}

and DashboardReadModel = {
    TotalUsers: int
    ActiveUsers: int
    TotalOrders: int
    PendingOrders: int
    TotalRevenue: decimal
    RevenueThisMonth: decimal
    TopProducts: TopProductModel list
    RecentOrders: RecentOrderModel list
    SalesByCategory: CategorySalesModel list
}

and TopProductModel = {
    ProductId: int
    ProductName: string
    TotalSold: int
    Revenue: decimal
}

and RecentOrderModel = {
    OrderId: int
    CustomerName: string
    Amount: decimal
    Status: string
    CreatedAt: DateTime
}

and CategorySalesModel = {
    CategoryName: string
    TotalOrders: int
    Revenue: decimal
}

and PagedResult<'T> = {
    Items: 'T list
    TotalCount: int
    Page: int
    PageSize: int
    TotalPages: int
}
```

---

## 4. Command Handlers (ตัวจัดการ Commands)

```fsharp
// CommandHandlers.fs
module CommandHandlers

open System
open System.Collections.Generic
open CoreTypes
open Commands

// ========================================
// 4.1 In-memory write store
// ========================================

let private users = Dictionary<int, WriteUser>()
let private products = Dictionary<int, WriteProduct>()
let private orders = Dictionary<int, WriteOrder>()
let private eventLog = ResizeArray<IDomainEvent>()
let mutable private nextUserId = 1
let mutable private nextProductId = 1
let mutable private nextOrderId = 1

let publishEvent (event: IDomainEvent) =
    eventLog.Add(event)
    printfn "[Event] %s at %A" (event.GetType().Name) (event.OccurredAt)

// ========================================
// 4.2 User command handlers
// ========================================

let handleRegisterUser (cmd: RegisterUserCommand) =
    task {
        // Validation
        let errors = ResizeArray<string>()
        if String.IsNullOrWhiteSpace cmd.Name then errors.Add("Name is required")
        if String.IsNullOrWhiteSpace cmd.Email then errors.Add("Email is required")
        elif not (cmd.Email.Contains("@")) then errors.Add("Invalid email format")
        if cmd.Password.Length < 8 then errors.Add("Password must be at least 8 characters")

        // Check duplicate email
        let emailExists = users.Values |> Seq.exists (fun u -> u.Email = cmd.Email)
        if emailExists then errors.Add($"Email {cmd.Email} already registered")

        if errors.Count > 0 then
            return Failure (errors |> Seq.toList)
        else
            let userId = nextUserId
            nextUserId <- nextUserId + 1

            let user = {
                Id = userId
                Name = cmd.Name
                Email = cmd.Email
                PasswordHash = $"hashed:{cmd.Password}"  // ในงานจริงใช้ BCrypt
                IsActive = true
                CreatedAt = DateTime.UtcNow
            }
            users.[userId] <- user

            // Publish event
            publishEvent {
                UserId = userId
                Name = cmd.Name
                Email = cmd.Email
                RegisteredAt = DateTime.UtcNow
            }

            return Success userId
    }

let handleUpdateUserProfile (cmd: UpdateUserProfileCommand) =
    task {
        match users.TryGetValue(cmd.UserId) with
        | false, _ ->
            return Failure [$"User {cmd.UserId} not found"]
        | true, user ->
            if String.IsNullOrWhiteSpace cmd.Name then
                return Failure ["Name is required"]
            else
                users.[cmd.UserId] <- { user with Name = cmd.Name }
                return Success ()
    }

let handleDeactivateUser (cmd: DeactivateUserCommand) =
    task {
        match users.TryGetValue(cmd.UserId) with
        | false, _ ->
            return Failure [$"User {cmd.UserId} not found"]
        | true, user ->
            users.[cmd.UserId] <- { user with IsActive = false }
            return Success ()
    }

// ========================================
// 4.3 Product command handlers
// ========================================

let handleCreateProduct (cmd: CreateProductCommand) =
    task {
        let errors = ResizeArray<string>()
        if String.IsNullOrWhiteSpace cmd.Name then errors.Add("Name is required")
        if cmd.Price <= 0m then errors.Add("Price must be positive")
        if cmd.Stock < 0 then errors.Add("Stock cannot be negative")

        if errors.Count > 0 then
            return Failure (errors |> Seq.toList)
        else
            let productId = nextProductId
            nextProductId <- nextProductId + 1

            let product = {
                Id = productId
                Name = cmd.Name
                Price = cmd.Price
                Stock = cmd.Stock
                CategoryId = cmd.CategoryId
                Description = cmd.Description
                IsActive = true
            }
            products.[productId] <- product
            return Success productId
    }

let handleUpdateProductPrice (cmd: UpdateProductPriceCommand) =
    task {
        match products.TryGetValue(cmd.ProductId) with
        | false, _ ->
            return Failure [$"Product {cmd.ProductId} not found"]
        | true, product ->
            if cmd.NewPrice <= 0m then
                return Failure ["Price must be positive"]
            else
                let oldPrice = product.Price
                products.[cmd.ProductId] <- { product with Price = cmd.NewPrice }

                publishEvent {
                    ProductId = cmd.ProductId
                    OldPrice = oldPrice
                    NewPrice = cmd.NewPrice
                    Reason = cmd.Reason
                    ChangedAt = DateTime.UtcNow
                }

                return Success ()
    }

let handleRestockProduct (cmd: RestockProductCommand) =
    task {
        match products.TryGetValue(cmd.ProductId) with
        | false, _ ->
            return Failure [$"Product {cmd.ProductId} not found"]
        | true, product ->
            if cmd.Quantity <= 0 then
                return Failure ["Restock quantity must be positive"]
            else
                let oldStock = product.Stock
                let newStock = product.Stock + cmd.Quantity
                products.[cmd.ProductId] <- { product with Stock = newStock }

                publishEvent {
                    ProductId = cmd.ProductId
                    OldStock = oldStock
                    NewStock = newStock
                    UpdatedAt = DateTime.UtcNow
                }

                return Success newStock
    }

// ========================================
// 4.4 Order command handlers
// ========================================

let handlePlaceOrder (cmd: PlaceOrderCommand) =
    task {
        // Validate customer
        match users.TryGetValue(cmd.CustomerId) with
        | false, _ ->
            return Failure [$"Customer {cmd.CustomerId} not found"]
        | true, customer when not customer.IsActive ->
            return Failure ["Customer account is not active"]
        | true, _ ->
            let errors = ResizeArray<string>()
            if cmd.Items.IsEmpty then errors.Add("Order must have at least one item")

            let orderItems = ResizeArray<WriteOrderItem>()
            let mutable totalAmount = 0m

            for item in cmd.Items do
                match products.TryGetValue(item.ProductId) with
                | false, _ ->
                    errors.Add($"Product {item.ProductId} not found")
                | true, product when not product.IsActive ->
                    errors.Add($"Product {product.Name} is not available")
                | true, product when product.Stock < item.Quantity ->
                    errors.Add($"Insufficient stock for {product.Name} (available: {product.Stock})")
                | true, product ->
                    let lineTotal = product.Price * decimal item.Quantity
                    totalAmount <- totalAmount + lineTotal
                    orderItems.Add({
                        ProductId = item.ProductId
                        Quantity = item.Quantity
                        UnitPrice = product.Price
                    })

            if errors.Count > 0 then
                return Failure (errors |> Seq.toList)
            else
                // Deduct stock
                for item in cmd.Items do
                    let product = products.[item.ProductId]
                    products.[item.ProductId] <- { product with Stock = product.Stock - item.Quantity }

                let orderId = nextOrderId
                nextOrderId <- nextOrderId + 1

                let order = {
                    Id = orderId
                    CustomerId = cmd.CustomerId
                    Items = orderItems |> Seq.toList
                    TotalAmount = totalAmount
                    Status = "Pending"
                    TrackingNumber = None
                    CreatedAt = DateTime.UtcNow
                }
                orders.[orderId] <- order

                publishEvent {
                    OrderId = orderId
                    CustomerId = cmd.CustomerId
                    TotalAmount = totalAmount
                    ItemCount = cmd.Items.Length
                    PlacedAt = DateTime.UtcNow
                }

                return Success orderId
    }

let handleCancelOrder (cmd: CancelOrderCommand) =
    task {
        match orders.TryGetValue(cmd.OrderId) with
        | false, _ ->
            return Failure [$"Order {cmd.OrderId} not found"]
        | true, order when order.Status = "Shipped" || order.Status = "Delivered" ->
            return Failure [$"Cannot cancel order in {order.Status} status"]
        | true, order ->
            // Restore stock
            for item in order.Items do
                match products.TryGetValue(item.ProductId) with
                | true, product ->
                    products.[item.ProductId] <- { product with Stock = product.Stock + item.Quantity }
                | _ -> ()

            orders.[cmd.OrderId] <- { order with Status = "Cancelled" }
            return Success ()
    }

let handleShipOrder (cmd: ShipOrderCommand) =
    task {
        match orders.TryGetValue(cmd.OrderId) with
        | false, _ ->
            return Failure [$"Order {cmd.OrderId} not found"]
        | true, order when order.Status <> "Processing" ->
            return Failure [$"Order must be in Processing status to ship (current: {order.Status})"]
        | true, order ->
            orders.[cmd.OrderId] <- {
                order with
                    Status = "Shipped"
                    TrackingNumber = Some cmd.TrackingNumber
            }
            return Success ()
    }

// ========================================
// 4.5 Expose state for query handlers
// ========================================

let getUsers () = users
let getProducts () = products
let getOrders () = orders
let getEvents () = eventLog |> Seq.toList
```

---

## 5. Query Handlers (ตัวจัดการ Queries)

```fsharp
// QueryHandlers.fs
module QueryHandlers

open System
open CoreTypes
open Queries
open CommandHandlers  // Access write store

// ========================================
// 5.1 Category lookup (mock data)
// ========================================

let private categories =
    dict [
        1, "Electronics"
        2, "Accessories"
        3, "Software"
        4, "Gaming"
    ]

let getCategoryName catId =
    match categories.TryGetValue(catId) with
    | true, name -> name
    | _ -> "Unknown"

// ========================================
// 5.2 User query handlers
// ========================================

let handleGetUserById (query: GetUserByIdQuery) =
    task {
        let users = getUsers()
        return
            match users.TryGetValue(query.UserId) with
            | false, _ -> None
            | true, u ->
                let orders = getOrders()
                let userOrders = orders.Values |> Seq.filter (fun o -> o.CustomerId = u.Id) |> Seq.toList
                let totalSpent = userOrders |> List.sumBy (fun o -> o.TotalAmount)
                Some {
                    Id = u.Id
                    Name = u.Name
                    Email = u.Email
                    IsActive = u.IsActive
                    CreatedAt = u.CreatedAt
                    OrderCount = userOrders.Length
                    TotalSpent = totalSpent
                }
    }

let handleGetUsers (query: GetUsersQuery) =
    task {
        let users = getUsers()
        let orders = getOrders()

        let filtered =
            users.Values
            |> Seq.filter (fun u ->
                let activeMatch = not query.ActiveOnly || u.IsActive
                let searchMatch =
                    query.SearchTerm
                    |> Option.map (fun s ->
                        let term = s.ToLower()
                        u.Name.ToLower().Contains(term) || u.Email.ToLower().Contains(term)
                    )
                    |> Option.defaultValue true
                activeMatch && searchMatch
            )
            |> Seq.toList

        let total = filtered.Length
        let items =
            filtered
            |> List.skip ((query.Page - 1) * query.PageSize)
            |> List.truncate query.PageSize
            |> List.map (fun u ->
                let userOrders = orders.Values |> Seq.filter (fun o -> o.CustomerId = u.Id) |> Seq.toList
                {
                    Id = u.Id
                    Name = u.Name
                    Email = u.Email
                    IsActive = u.IsActive
                    CreatedAt = u.CreatedAt
                    OrderCount = userOrders.Length
                    TotalSpent = userOrders |> List.sumBy (fun o -> o.TotalAmount)
                }
            )

        return {
            Items = items
            TotalCount = total
            Page = query.Page
            PageSize = query.PageSize
            TotalPages = (total + query.PageSize - 1) / query.PageSize
        }
    }

// ========================================
// 5.3 Product query handlers
// ========================================

let handleGetProductById (query: GetProductByIdQuery) =
    task {
        let products = getProducts()
        return
            match products.TryGetValue(query.ProductId) with
            | false, _ -> None
            | true, p ->
                Some {
                    Id = p.Id
                    Name = p.Name
                    Price = p.Price
                    OriginalPrice = None
                    DiscountPercent = None
                    Stock = p.Stock
                    IsAvailable = p.IsActive && p.Stock > 0
                    CategoryName = getCategoryName p.CategoryId
                    Tags = []
                    RatingAverage = None
                    ReviewCount = 0
                }
    }

let handleGetProducts (query: GetProductsQuery) =
    task {
        let products = getProducts()
        let filtered =
            products.Values
            |> Seq.filter (fun p ->
                let catMatch = query.CategoryId |> Option.map (fun c -> p.CategoryId = c) |> Option.defaultValue true
                let minMatch = query.MinPrice |> Option.map (fun m -> p.Price >= m) |> Option.defaultValue true
                let maxMatch = query.MaxPrice |> Option.map (fun m -> p.Price <= m) |> Option.defaultValue true
                let searchMatch =
                    query.SearchTerm
                    |> Option.map (fun s -> p.Name.ToLower().Contains(s.ToLower()))
                    |> Option.defaultValue true
                let stockMatch = not query.InStockOnly || p.Stock > 0
                catMatch && minMatch && maxMatch && searchMatch && stockMatch && p.IsActive
            )
            |> Seq.toList

        let sorted =
            match query.SortBy with
            | "price" when query.SortDescending -> filtered |> List.sortByDescending (fun p -> p.Price)
            | "price" -> filtered |> List.sortBy (fun p -> p.Price)
            | "name" when query.SortDescending -> filtered |> List.sortByDescending (fun p -> p.Name)
            | _ -> filtered |> List.sortBy (fun p -> p.Name)

        let total = sorted.Length
        let items =
            sorted
            |> List.skip ((query.Page - 1) * query.PageSize)
            |> List.truncate query.PageSize
            |> List.map (fun p ->
                {
                    Id = p.Id
                    Name = p.Name
                    Price = p.Price
                    OriginalPrice = None
                    DiscountPercent = None
                    Stock = p.Stock
                    IsAvailable = p.IsActive && p.Stock > 0
                    CategoryName = getCategoryName p.CategoryId
                    Tags = []
                    RatingAverage = None
                    ReviewCount = 0
                }
            )

        return {
            Items = items
            TotalCount = total
            Page = query.Page
            PageSize = query.PageSize
            TotalPages = (total + query.PageSize - 1) / query.PageSize
        }
    }

// ========================================
// 5.4 Dashboard query handler
// ========================================

let handleGetDashboard (query: GetDashboardQuery) =
    task {
        let users = getUsers()
        let products = getProducts()
        let orders = getOrders()

        let totalRevenue = orders.Values |> Seq.sumBy (fun o -> o.TotalAmount)
        let thisMonth = DateTime.UtcNow.AddDays(-30.0)
        let revenueThisMonth =
            orders.Values
            |> Seq.filter (fun o -> o.CreatedAt >= thisMonth)
            |> Seq.sumBy (fun o -> o.TotalAmount)

        let topProducts =
            orders.Values
            |> Seq.collect (fun o -> o.Items)
            |> Seq.groupBy (fun i -> i.ProductId)
            |> Seq.map (fun (productId, items) ->
                let totalSold = items |> Seq.sumBy (fun i -> i.Quantity)
                let revenue = items |> Seq.sumBy (fun i -> decimal i.Quantity * i.UnitPrice)
                let productName =
                    match products.TryGetValue(productId) with
                    | true, p -> p.Name
                    | _ -> "Unknown"
                { ProductId = productId; ProductName = productName; TotalSold = totalSold; Revenue = revenue }
            )
            |> Seq.sortByDescending (fun p -> p.Revenue)
            |> Seq.truncate 5
            |> Seq.toList

        let recentOrders =
            orders.Values
            |> Seq.sortByDescending (fun o -> o.CreatedAt)
            |> Seq.truncate 5
            |> Seq.map (fun o ->
                let customerName =
                    match users.TryGetValue(o.CustomerId) with
                    | true, u -> u.Name
                    | _ -> "Unknown"
                {
                    OrderId = o.Id
                    CustomerName = customerName
                    Amount = o.TotalAmount
                    Status = o.Status
                    CreatedAt = o.CreatedAt
                }
            )
            |> Seq.toList

        return {
            TotalUsers = users.Count
            ActiveUsers = users.Values |> Seq.filter (fun u -> u.IsActive) |> Seq.length
            TotalOrders = orders.Count
            PendingOrders = orders.Values |> Seq.filter (fun o -> o.Status = "Pending") |> Seq.length
            TotalRevenue = totalRevenue
            RevenueThisMonth = revenueThisMonth
            TopProducts = topProducts
            RecentOrders = recentOrders
            SalesByCategory = []  // simplified
        }
    }
```

---

## 6. Mediator Pattern (ตัวกลาง)

```fsharp
// Mediator.fs
module Mediator

open System
open CoreTypes
open System.Collections.Generic

// ========================================
// 6.1 Simple Mediator
// ========================================

type IMediator =
    abstract member SendAsync<'TCommand, 'TResult when 'TCommand :> ICommand> :
        'TCommand -> System.Threading.Tasks.Task<CommandResult<'TResult>>
    abstract member QueryAsync<'TQuery, 'TResult when 'TQuery :> IQuery<'TResult>> :
        'TQuery -> System.Threading.Tasks.Task<'TResult>

// ========================================
// 6.2 In-memory Mediator implementation
// ========================================

type InMemoryMediator() =
    // Delegate-based handlers สำหรับ F#
    let commandHandlers = Dictionary<Type, obj>()
    let queryHandlers = Dictionary<Type, obj>()

    member _.RegisterCommandHandler<'TCmd, 'TResult when 'TCmd :> ICommand>
        (handler: 'TCmd -> System.Threading.Tasks.Task<CommandResult<'TResult>>) =
        commandHandlers.[typeof<'TCmd>] <- box handler

    member _.RegisterQueryHandler<'TQuery, 'TResult when 'TQuery :> IQuery<'TResult>>
        (handler: 'TQuery -> System.Threading.Tasks.Task<'TResult>) =
        queryHandlers.[typeof<'TQuery>] <- box handler

    member this.SendAsync<'TCmd, 'TResult when 'TCmd :> ICommand> (cmd: 'TCmd) =
        task {
            match commandHandlers.TryGetValue(typeof<'TCmd>) with
            | false, _ ->
                return Failure [$"No handler registered for {typeof<'TCmd>.Name}"]
            | true, handler ->
                let typedHandler = handler :?> ('TCmd -> System.Threading.Tasks.Task<CommandResult<'TResult>>)
                return! typedHandler cmd
        }

    member this.QueryAsync<'TQuery, 'TResult when 'TQuery :> IQuery<'TResult>> (query: 'TQuery) =
        task {
            match queryHandlers.TryGetValue(typeof<'TQuery>) with
            | false, _ ->
                return failwith $"No handler registered for {typeof<'TQuery>.Name}"
            | true, handler ->
                let typedHandler = handler :?> ('TQuery -> System.Threading.Tasks.Task<'TResult>)
                return! typedHandler query
        }

// ========================================
// 6.3 Setup mediator
// ========================================

let setupMediator () =
    let mediator = InMemoryMediator()

    // Register command handlers
    mediator.RegisterCommandHandler<Commands.RegisterUserCommand, int>(
        CommandHandlers.handleRegisterUser
    )
    mediator.RegisterCommandHandler<Commands.UpdateUserProfileCommand, unit>(
        CommandHandlers.handleUpdateUserProfile
    )
    mediator.RegisterCommandHandler<Commands.CreateProductCommand, int>(
        CommandHandlers.handleCreateProduct
    )
    mediator.RegisterCommandHandler<Commands.UpdateProductPriceCommand, unit>(
        CommandHandlers.handleUpdateProductPrice
    )
    mediator.RegisterCommandHandler<Commands.RestockProductCommand, int>(
        CommandHandlers.handleRestockProduct
    )
    mediator.RegisterCommandHandler<Commands.PlaceOrderCommand, int>(
        CommandHandlers.handlePlaceOrder
    )
    mediator.RegisterCommandHandler<Commands.CancelOrderCommand, unit>(
        CommandHandlers.handleCancelOrder
    )
    mediator.RegisterCommandHandler<Commands.ShipOrderCommand, unit>(
        CommandHandlers.handleShipOrder
    )

    // Register query handlers
    mediator.RegisterQueryHandler<Queries.GetUserByIdQuery, Queries.UserReadModel option>(
        QueryHandlers.handleGetUserById
    )
    mediator.RegisterQueryHandler<Queries.GetUsersQuery, Queries.PagedResult<Queries.UserReadModel>>(
        QueryHandlers.handleGetUsers
    )
    mediator.RegisterQueryHandler<Queries.GetProductByIdQuery, Queries.ProductReadModel option>(
        QueryHandlers.handleGetProductById
    )
    mediator.RegisterQueryHandler<Queries.GetProductsQuery, Queries.PagedResult<Queries.ProductReadModel>>(
        QueryHandlers.handleGetProducts
    )
    mediator.RegisterQueryHandler<Queries.GetDashboardQuery, Queries.DashboardReadModel>(
        QueryHandlers.handleGetDashboard
    )

    mediator
```

---

## 7. Complete Example

```fsharp
// Program.fs
module Program

open System
open Commands
open Queries

[<EntryPoint>]
let main _ =
    task {
        printfn "=== CQRS Pattern F# Demo ==="
        printfn "==========================="

        let mediator = Mediator.setupMediator()

        // ========================================
        // COMMANDS (Write side)
        // ========================================
        printfn "\n--- Commands ---"

        // Register users
        let! r1 = mediator.SendAsync<RegisterUserCommand, int> {
            Name = "Alice Johnson"
            Email = "alice@example.com"
            Password = "SecurePass123"
        }
        let! r2 = mediator.SendAsync<RegisterUserCommand, int> {
            Name = "Bob Smith"
            Email = "bob@example.com"
            Password = "AnotherPass456"
        }

        let userId1 = match r1 with Success id -> id | _ -> 0
        let userId2 = match r2 with Success id -> id | _ -> 0
        printfn "Registered users: %d, %d" userId1 userId2

        // Create products
        let! pr1 = mediator.SendAsync<CreateProductCommand, int> {
            Name = "Laptop Ultra Pro"
            Price = 49999m
            Stock = 50
            CategoryId = 1
            Description = Some "High performance laptop"
        }
        let! pr2 = mediator.SendAsync<CreateProductCommand, int> {
            Name = "Wireless Keyboard"
            Price = 1599m
            Stock = 200
            CategoryId = 2
            Description = None
        }
        let! pr3 = mediator.SendAsync<CreateProductCommand, int> {
            Name = "Gaming Headset"
            Price = 3299m
            Stock = 75
            CategoryId = 4
            Description = Some "7.1 Surround sound"
        }

        let productId1 = match pr1 with Success id -> id | _ -> 0
        let productId2 = match pr2 with Success id -> id | _ -> 0
        let productId3 = match pr3 with Success id -> id | _ -> 0
        printfn "Created products: %d, %d, %d" productId1 productId2 productId3

        // Place orders
        let! or1 = mediator.SendAsync<PlaceOrderCommand, int> {
            CustomerId = userId1
            Items = [
                { ProductId = productId1; Quantity = 1 }
                { ProductId = productId2; Quantity = 2 }
            ]
            ShippingAddress = {
                Street = "123 ถ.สุขุมวิท"
                City = "กรุงเทพฯ"
                Country = "Thailand"
                PostalCode = "10110"
            }
        }
        match or1 with
        | Success orderId -> printfn "Order placed: #%d" orderId
        | Failure errors -> printfn "Order failed: %A" errors

        // Update product price
        let! priceResult = mediator.SendAsync<UpdateProductPriceCommand, unit> {
            ProductId = productId1
            NewPrice = 47999m
            Reason = "Promotional price"
        }
        printfn "Price update: %A" priceResult

        // Restock
        let! stockResult = mediator.SendAsync<RestockProductCommand, int> {
            ProductId = productId2
            Quantity = 100
            Supplier = Some "Logitech Thailand"
        }
        match stockResult with
        | Success newStock -> printfn "Restocked product %d, new stock: %d" productId2 newStock
        | Failure errors -> printfn "Restock failed: %A" errors

        // ========================================
        // QUERIES (Read side)
        // ========================================
        printfn "\n--- Queries ---"

        // Get user
        let! user = mediator.QueryAsync<GetUserByIdQuery, UserReadModel option> {
            UserId = userId1
        }
        match user with
        | Some u ->
            printfn "User: %s <%s> - Orders: %d, Spent: ฿%.2f" u.Name u.Email u.OrderCount u.TotalSpent
        | None ->
            printfn "User not found"

        // Get products
        let! products = mediator.QueryAsync<GetProductsQuery, PagedResult<ProductReadModel>> {
            CategoryId = None
            MinPrice = None
            MaxPrice = None
            SearchTerm = None
            InStockOnly = true
            SortBy = "price"
            SortDescending = false
            Page = 1
            PageSize = 10
        }
        printfn "\nProducts (sorted by price):"
        for p in products.Items do
            printfn "  [%d] %s - ฿%.2f (Stock: %d, Available: %b)" p.Id p.Name p.Price p.Stock p.IsAvailable

        // Dashboard
        let! dashboard = mediator.QueryAsync<GetDashboardQuery, DashboardReadModel> {
            UserId = None
        }
        printfn "\nDashboard:"
        printfn "  Total Users: %d (Active: %d)" dashboard.TotalUsers dashboard.ActiveUsers
        printfn "  Total Orders: %d (Pending: %d)" dashboard.TotalOrders dashboard.PendingOrders
        printfn "  Total Revenue: ฿%.2f" dashboard.TotalRevenue
        printfn "  Top Products:"
        for p in dashboard.TopProducts do
            printfn "    %s - Sold: %d, Revenue: ฿%.2f" p.ProductName p.TotalSold p.Revenue

        // Events log
        printfn "\n--- Domain Events Published ---"
        let events = CommandHandlers.getEvents()
        printfn "Total events: %d" events.Length
        for event in events do
            printfn "  [%s] at %A" (event.GetType().Name) event.OccurredAt

        printfn "\n=== Demo Complete ==="
        return 0
    } |> Async.AwaitTask |> Async.RunSynchronously
```

---

## สรุป (Summary)

CQRS ใน F# มีข้อดีหลายประการ:

1. **Command = เปลี่ยนแปลง state** → Return `CommandResult<T>` (Success/Failure)
2. **Query = อ่านข้อมูล** → Return ข้อมูลตรงๆ ไม่มี side effects
3. **Mediator** → Decouples callers จาก handlers
4. **Domain Events** → บันทึกสิ่งที่เกิดขึ้น ใช้ audit log และ projections
5. **Read Models** → Optimized สำหรับ UI/API responses

```fsharp
// CQRS Pattern Summary:
// Command: สั่งให้ระบบทำสิ่งที่เปลี่ยนแปลง state
// Query: ถามข้อมูลจากระบบ ไม่เปลี่ยนอะไร
// Handler: ประมวลผล command/query
// Mediator: ส่ง command/query ไปยัง handler ที่ถูกต้อง
// Event: บันทึกสิ่งที่เกิดขึ้น (audit, projections)
// Read Model: View ที่ optimized สำหรับ queries
```
