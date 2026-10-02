# Part 83 - Functional Architecture Patterns

## บทนำ (Introduction)

Functional Architecture เป็นแนวทางการออกแบบซอฟต์แวร์ที่ใช้หลักการ functional programming เป็นแกนหลัก แทนที่จะใช้ OOP patterns แบบเดิม

## 1. Functional Core, Imperative Shell

```fsharp
// หลักการ: แยก pure functions (core) ออกจาก side effects (shell)
// Pure core: ไม่มี I/O, ไม่มี side effects - testable ง่าย
// Imperative shell: จัดการ I/O, database, HTTP calls

// ===== Functional Core =====
module Core =
    
    // Pure domain logic - ไม่มี dependencies ภายนอก
    type Cart = {
        Items: Map<string, int>  // productId -> quantity
        Discount: decimal
    }
    
    type CartCommand =
        | AddItem of productId: string * quantity: int
        | RemoveItem of productId: string
        | ApplyDiscount of percent: decimal
        | ClearCart
    
    type CartEvent =
        | ItemAdded of productId: string * quantity: int
        | ItemRemoved of productId: string
        | DiscountApplied of percent: decimal
        | CartCleared
    
    // Pure state transition
    let evolve (cart: Cart) (event: CartEvent) : Cart =
        match event with
        | ItemAdded (pid, qty) ->
            let currentQty = cart.Items |> Map.tryFind pid |> Option.defaultValue 0
            { cart with Items = cart.Items |> Map.add pid (currentQty + qty) }
        
        | ItemRemoved pid ->
            { cart with Items = cart.Items |> Map.remove pid }
        
        | DiscountApplied pct ->
            { cart with Discount = pct }
        
        | CartCleared ->
            { Items = Map.empty; Discount = 0m }
    
    // Pure command -> event decision
    let decide (cart: Cart) (command: CartCommand) : Result<CartEvent list, string> =
        match command with
        | AddItem (pid, qty) ->
            if qty <= 0 then Error "Quantity must be positive"
            elif System.String.IsNullOrWhiteSpace(pid) then Error "Product ID required"
            else Ok [ItemAdded (pid, qty)]
        
        | RemoveItem pid ->
            if not (cart.Items |> Map.containsKey pid) then
                Error (sprintf "Item %s not in cart" pid)
            else
                Ok [ItemRemoved pid]
        
        | ApplyDiscount pct ->
            if pct < 0m || pct > 50m then
                Error "Discount must be 0-50%"
            else
                Ok [DiscountApplied pct]
        
        | ClearCart ->
            Ok [CartCleared]
    
    // Pure calculation
    let calculateTotal (prices: Map<string, decimal>) (cart: Cart) : decimal =
        let subtotal =
            cart.Items
            |> Map.toSeq
            |> Seq.sumBy (fun (pid, qty) ->
                match prices |> Map.tryFind pid with
                | Some price -> price * decimal qty
                | None -> 0m)
        
        subtotal * (1m - cart.Discount / 100m)
    
    let itemCount (cart: Cart) =
        cart.Items |> Map.values |> Seq.sum

// ===== Imperative Shell =====
module Shell =
    open Core
    
    // Types for external dependencies
    type LoadCart = string -> Async<Cart option>
    type SaveCart = string -> Cart -> Async<unit>
    type GetPrices = string list -> Async<Map<string, decimal>>
    type PublishEvent = CartEvent -> Async<unit>
    
    // Shell orchestrates: load -> decide -> evolve -> save -> publish
    let handleCommand 
        (loadCart: LoadCart)
        (saveCart: SaveCart)
        (getPrices: GetPrices)
        (publishEvent: PublishEvent)
        (cartId: string)
        (command: CartCommand) =
        async {
            // Load state from storage (side effect)
            let! cartOpt = loadCart cartId
            let cart = cartOpt |> Option.defaultValue { Items = Map.empty; Discount = 0m }
            
            // Decide (pure - no side effects)
            match Core.decide cart command with
            | Error e -> return Error e
            | Ok events ->
            
            // Evolve (pure - no side effects)
            let newCart = events |> List.fold Core.evolve cart
            
            // Save (side effect)
            do! saveCart cartId newCart
            
            // Publish events (side effect)
            for event in events do
                do! publishEvent event
            
            // Get prices for total calculation (side effect)
            let productIds = newCart.Items |> Map.keys |> Seq.toList
            let! prices = getPrices productIds
            
            // Calculate total (pure)
            let total = Core.calculateTotal prices newCart
            
            return Ok {| Cart = newCart; Total = total; Events = events |}
        }
```

## 2. Onion Architecture กับ Functions

```fsharp
// Onion Architecture: ทุกอย่างเป็น functions และ types
// ไม่ต้องใช้ interfaces หรือ classes เลย

// Layer 1: Core Domain Types
module Domain =
    type ProductId = string
    type UserId = string
    
    type Product = {
        Id: ProductId
        Name: string
        Price: decimal
        Stock: int
    }
    
    type OrderLine = {
        ProductId: ProductId
        Quantity: int
        Price: decimal
    }
    
    type Order = {
        Id: string
        UserId: UserId
        Lines: OrderLine list
        Status: string
        CreatedAt: System.DateTime
    }
    
    // Pure domain functions
    let orderTotal (order: Order) =
        order.Lines |> List.sumBy (fun l -> l.Price * decimal l.Quantity)
    
    let canFulfill (products: Map<ProductId, Product>) (order: Order) =
        order.Lines |> List.forall (fun line ->
            match products |> Map.tryFind line.ProductId with
            | Some p -> p.Stock >= line.Quantity
            | None -> false)

// Layer 2: Application Use Cases (functions that compose domain)
module UseCases =
    open Domain
    
    // Port types (function types instead of interfaces)
    type FindProduct = ProductId -> Async<Product option>
    type FindOrder = string -> Async<Order option>
    type SaveOrder = Order -> Async<unit>
    type FindUserOrders = UserId -> Async<Order list>
    type NotifyUser = UserId -> string -> Async<unit>
    
    // Use case: Place an order
    let placeOrder 
        (findProduct: FindProduct)
        (saveOrder: SaveOrder)
        (notifyUser: NotifyUser)
        (userId: UserId)
        (items: (ProductId * int) list) =
        async {
            // Validate products
            let! productResults =
                items
                |> List.map (fun (pid, _) -> findProduct pid)
                |> Async.Parallel
            
            let products = productResults |> Array.choose id
            
            if products.Length <> items.Length then
                return Error "Some products not found"
            else
            
            // Create order lines
            let lines = 
                List.zip items (products |> Array.toList)
                |> List.map (fun ((pid, qty), product) ->
                    { ProductId = pid; Quantity = qty; Price = product.Price })
            
            let order = {
                Id = System.Guid.NewGuid().ToString()
                UserId = userId
                Lines = lines
                Status = "Pending"
                CreatedAt = System.DateTime.UtcNow
            }
            
            do! saveOrder order
            
            let total = orderTotal order
            do! notifyUser userId (sprintf "Order placed! Total: %.2f" total)
            
            return Ok order
        }
    
    // Use case: Get order summary
    let getOrderSummary
        (findOrder: FindOrder)
        (orderId: string) =
        async {
            let! orderOpt = findOrder orderId
            match orderOpt with
            | None -> return Error "Order not found"
            | Some order ->
                return Ok {|
                    OrderId = order.Id
                    Total = orderTotal order
                    ItemCount = order.Lines |> List.sumBy (fun l -> l.Quantity)
                    Status = order.Status
                |}
        }

// Layer 3: Infrastructure (concrete implementations)
module Infrastructure =
    open UseCases
    open Domain
    
    // In-memory storage
    let private productStorage = System.Collections.Generic.Dictionary<string, Product>()
    let private orderStorage = System.Collections.Generic.Dictionary<string, Order>()
    
    // Concrete implementations of port functions
    let findProductInMemory : FindProduct = fun productId ->
        async {
            match productStorage.TryGetValue(productId) with
            | true, p -> return Some p
            | _ -> return None
        }
    
    let saveOrderInMemory : SaveOrder = fun order ->
        async { orderStorage.[order.Id] <- order }
    
    let findOrderInMemory : FindOrder = fun orderId ->
        async {
            match orderStorage.TryGetValue(orderId) with
            | true, o -> return Some o
            | _ -> return None
        }
    
    let consoleNotifyUser : NotifyUser = fun userId message ->
        async { printfn "NOTIFY [%s]: %s" userId message }
    
    // Seed data
    let seedProducts () =
        let products = [
            { Id = "P001"; Name = "Widget A"; Price = 100m; Stock = 50 }
            { Id = "P002"; Name = "Widget B"; Price = 200m; Stock = 30 }
            { Id = "P003"; Name = "Gadget X"; Price = 500m; Stock = 10 }
        ]
        for p in products do
            productStorage.[p.Id] <- p

// Layer 4: Composition Root
module CompositionRoot =
    open UseCases
    open Infrastructure
    
    // Wire up everything using partial application
    let placeOrder' : string -> (string * int) list -> Async<Result<Domain.Order, string>> =
        placeOrder
            findProductInMemory
            saveOrderInMemory
            consoleNotifyUser
    
    let getOrderSummary' : string -> Async<Result<{| OrderId: string; Total: decimal; ItemCount: int; Status: string |}, string>> =
        getOrderSummary findOrderInMemory
```

## 3. Ports as Function Types

```fsharp
// แทนที่จะใช้ interfaces - ใช้ function types โดยตรง

// ===== Database Ports =====
type Db<'T> = Async<'T>

type FindById<'Id, 'T> = 'Id -> Db<'T option>
type FindAll<'T> = unit -> Db<'T list>
type FindWhere<'Filter, 'T> = 'Filter -> Db<'T list>
type Save<'T> = 'T -> Db<unit>
type Delete<'Id> = 'Id -> Db<bool>

// User repository as function records
type UserRepository = {
    FindById: FindById<string, User>
    FindByEmail: string -> Db<User option>
    Save: Save<User>
    Delete: Delete<string>
    Exists: string -> Db<bool>
}

and User = {
    Id: string
    Email: string
    Name: string
    CreatedAt: System.DateTime
}

// Create in-memory repository
let createInMemoryUserRepo () : UserRepository =
    let storage = System.Collections.Generic.Dictionary<string, User>()
    {
        FindById = fun id -> async {
            match storage.TryGetValue(id) with
            | true, u -> return Some u
            | _ -> return None }
        
        FindByEmail = fun email -> async {
            return storage.Values |> Seq.tryFind (fun u -> u.Email = email) }
        
        Save = fun user -> async { storage.[user.Id] <- user }
        
        Delete = fun id -> async { return storage.Remove(id) }
        
        Exists = fun email -> async {
            return storage.Values |> Seq.exists (fun u -> u.Email = email) }
    }

// ===== External Service Ports =====
type EmailPort = {
    SendEmail: to': string -> subject: string -> body: string -> Async<Result<unit, string>>
}

type StoragePort = {
    Upload: filename: string -> data: byte[] -> Async<Result<string, string>>
    Download: url: string -> Async<Result<byte[], string>>
}

type CachePort = {
    Get: key: string -> Async<string option>
    Set: key: string -> value: string -> ttlSeconds: int -> Async<unit>
    Delete: key: string -> Async<unit>
}

// Using ports in application logic
let sendWelcomeEmailWorkflow 
    (userRepo: UserRepository)
    (email: EmailPort)
    (userId: string) =
    async {
        let! userOpt = userRepo.FindById userId
        match userOpt with
        | None -> return Error "User not found"
        | Some user ->
            let! result = email.SendEmail user.Email "Welcome!" (sprintf "Hello %s!" user.Name)
            return result
    }
```

## 4. Dependency Injection โดยไม่ใช้ IoC Container

```fsharp
// Functional DI ใช้ partial application แทน constructor injection

// ===== Step 1: Define pure domain logic =====
module PureDomain =
    type Product = { Id: string; Name: string; Price: decimal; Stock: int }
    type Order = { Id: string; ProductId: string; Quantity: int }
    
    // Pure functions - no dependencies
    let calculateDiscount (customerTier: string) (amount: decimal) =
        match customerTier with
        | "Gold" -> amount * 0.9m
        | "Silver" -> amount * 0.95m
        | _ -> amount
    
    let validateOrder (stock: int) (requestedQty: int) =
        if requestedQty <= 0 then Error "Quantity must be positive"
        elif requestedQty > stock then Error "Insufficient stock"
        else Ok ()

// ===== Step 2: Define capability types =====
type FindProduct = string -> Async<PureDomain.Product option>
type SaveOrder = PureDomain.Order -> Async<unit>
type SendNotification = string -> string -> Async<unit>
type GetCustomerTier = string -> Async<string>

// ===== Step 3: Create use cases via partial application =====
let createOrderWorkflow
    (findProduct: FindProduct)
    (saveOrder: SaveOrder)
    (sendNotification: SendNotification)
    (getCustomerTier: GetCustomerTier) =
    // Returns a function with all dependencies baked in
    fun (customerId: string) (productId: string) (quantity: int) ->
        async {
            let! productOpt = findProduct productId
            match productOpt with
            | None -> return Error "Product not found"
            | Some product ->
            
            match PureDomain.validateOrder product.Stock quantity with
            | Error e -> return Error e
            | Ok () ->
            
            let! tier = getCustomerTier customerId
            let total = PureDomain.calculateDiscount tier (product.Price * decimal quantity)
            
            let order = {
                PureDomain.Id = System.Guid.NewGuid().ToString()
                PureDomain.ProductId = productId
                PureDomain.Quantity = quantity
            }
            
            do! saveOrder order
            do! sendNotification customerId (sprintf "Order created! Total: %.2f" total)
            
            return Ok {| OrderId = order.Id; Total = total |}
        }

// ===== Step 4: Wire up at composition root =====
let wireUpProduction () =
    // Production implementations
    let findProductDb (id: string) = async {
        // Real DB call
        return None
    }
    
    let saveOrderDb (order: PureDomain.Order) = async {
        printfn "Saving order %s" order.Id
    }
    
    let sendEmailNotification (userId: string) (msg: string) = async {
        printfn "Email to %s: %s" userId msg
    }
    
    let getCustomerTierDb (userId: string) = async {
        return "Regular"
    }
    
    // Create the workflow with all dependencies
    createOrderWorkflow findProductDb saveOrderDb sendEmailNotification getCustomerTierDb

// Use it
let createOrder = wireUpProduction()
// Now createOrder: string -> string -> int -> Async<Result<...>>
```

## 5. Reader Monad for Dependencies

```fsharp
// Reader monad เป็น elegant way ในการ pass dependencies

// ===== Reader type =====
type Reader<'Env, 'T> = Reader of ('Env -> 'T)

module Reader =
    let run (Reader f) env = f env
    
    let ask = Reader id  // ดึง environment
    
    let asks f = Reader f  // ดึง part of environment
    
    let return' x = Reader (fun _ -> x)
    
    let map f (Reader g) = Reader (fun env -> f (g env))
    
    let bind (Reader g) f =
        Reader (fun env ->
            let a = g env
            run (f a) env)
    
    let local (f: 'E1 -> 'E2) (Reader g) =
        Reader (fun env -> g (f env))

// ===== Async Reader (ReaderT over Async) =====
type AsyncReader<'Env, 'T> = AsyncReader of ('Env -> Async<'T>)

module AsyncReader =
    let run (AsyncReader f) env = f env
    
    let ask = AsyncReader (fun env -> async { return env })
    
    let asks f = AsyncReader (fun env -> async { return f env })
    
    let return' x = AsyncReader (fun _ -> async { return x })
    
    let map f (AsyncReader g) =
        AsyncReader (fun env -> async {
            let! a = g env
            return f a })
    
    let bind (AsyncReader g) f =
        AsyncReader (fun env -> async {
            let! a = g env
            let (AsyncReader h) = f a
            return! h env })
    
    let liftAsync (operation: Async<'T>) =
        AsyncReader (fun _ -> operation)
    
    let fromReader (Reader f) =
        AsyncReader (fun env -> async { return f env })

// ===== Environment type =====
type AppEnvironment = {
    FindUser: string -> Async<{| Id: string; Email: string; Name: string |} option>
    FindProduct: string -> Async<{| Id: string; Name: string; Price: decimal |} option>
    SaveOrder: {| UserId: string; ProductId: string; Total: decimal |} -> Async<string>
    SendEmail: string -> string -> Async<unit>
    Logger: string -> unit
}

// ===== Business logic using Reader =====
let getUserOrFail (userId: string) : AsyncReader<AppEnvironment, Result<{| Id: string; Email: string; Name: string |}, string>> =
    AsyncReader (fun env -> async {
        let! userOpt = env.FindUser userId
        match userOpt with
        | None -> return Error (sprintf "User %s not found" userId)
        | Some user -> return Ok user
    })

let getProductOrFail (productId: string) : AsyncReader<AppEnvironment, Result<{| Id: string; Name: string; Price: decimal |}, string>> =
    AsyncReader (fun env -> async {
        let! productOpt = env.FindProduct productId
        match productOpt with
        | None -> return Error (sprintf "Product %s not found" productId)
        | Some product -> return Ok product
    })

let logMessage (message: string) : AsyncReader<AppEnvironment, unit> =
    AsyncReader (fun env -> async {
        env.Logger message
    })

// Compose business operations
let placeOrderWithReader (userId: string) (productId: string) (quantity: int) =
    AsyncReader (fun env -> async {
        // Log
        env.Logger (sprintf "Placing order for user %s" userId)
        
        // Get user
        let! userOpt = env.FindUser userId
        match userOpt with
        | None -> return Error "User not found"
        | Some user ->
        
        // Get product
        let! productOpt = env.FindProduct productId
        match productOpt with
        | None -> return Error "Product not found"
        | Some product ->
        
        // Calculate total
        let total = product.Price * decimal quantity
        
        // Save order
        let! orderId = env.SaveOrder {| UserId = userId; ProductId = productId; Total = total |}
        
        // Send email
        do! env.SendEmail user.Email (sprintf "Order %s confirmed! Total: %.2f" orderId total)
        
        env.Logger (sprintf "Order %s placed successfully" orderId)
        
        return Ok {| OrderId = orderId; Total = total; ProductName = product.Name |}
    })

// Create environment and run
let createTestEnvironment () =
    let products = Map.ofList [
        ("P001", {| Id = "P001"; Name = "Widget"; Price = 99.99m |})
    ]
    let users = Map.ofList [
        ("U001", {| Id = "U001"; Email = "john@example.com"; Name = "John" |})
    ]
    
    {
        FindUser = fun id -> async { return users |> Map.tryFind id }
        FindProduct = fun id -> async { return products |> Map.tryFind id }
        SaveOrder = fun _ -> async { return System.Guid.NewGuid().ToString() }
        SendEmail = fun email msg -> async { printfn "Email to %s: %s" email msg }
        Logger = fun msg -> printfn "[LOG] %s" msg
    }

let runExample () =
    let env = createTestEnvironment()
    let workflow = placeOrderWithReader "U001" "P001" 3
    let result = AsyncReader.run workflow env |> Async.RunSynchronously
    printfn "Result: %A" result
```

## 6. Free Monad for Testability

```fsharp
// Free monad ทำให้ test ได้โดยไม่ต้องมี actual implementations

// ===== Step 1: Define operations as discriminated union =====
type DatabaseOp<'Next> =
    | FindUser of id: string * next: ({| Id: string; Name: string |} option -> 'Next)
    | SaveUser of user: {| Id: string; Name: string |} * next: 'Next
    | DeleteUser of id: string * next: bool -> 'Next

type EmailOp<'Next> =
    | SendEmail of to': string * subject: string * body: string * next: 'Next

// Free monad (simplified version)
type AppOp<'Next> =
    | Db of DatabaseOp<AppOp<'Next>>
    | Email of EmailOp<AppOp<'Next>>
    | Pure of 'Next

// ===== Step 2: Smart constructors =====
let findUser id = Db (FindUser (id, Pure))
let saveUser user = Db (SaveUser (user, Pure ()))
let sendEmail to' subject body = Email (SendEmail (to', subject, body, Pure ()))

// ===== Step 3: Interpreter (for production) =====
let rec interpretProd (op: AppOp<'T>) : Async<'T> =
    match op with
    | Pure x -> async { return x }
    
    | Db (FindUser (id, next)) ->
        async {
            // Real database call
            let user = None  // Simulate DB lookup
            return! interpretProd (next user)
        }
    
    | Db (SaveUser (_, next)) ->
        async {
            // Real save to DB
            return! interpretProd next
        }
    
    | Db (DeleteUser (_, next)) ->
        async {
            return! interpretProd (next true)
        }
    
    | Email (SendEmail (to', subject, body, next)) ->
        async {
            printfn "Sending email to %s: %s" to' subject
            return! interpretProd next
        }

// ===== Step 4: Test interpreter (no side effects) =====
type TestState = {
    Users: Map<string, {| Id: string; Name: string |}>
    SentEmails: (string * string * string) list
}

let rec interpretTest (state: TestState) (op: AppOp<'T>) : TestState * 'T =
    match op with
    | Pure x -> state, x
    
    | Db (FindUser (id, next)) ->
        let user = state.Users |> Map.tryFind id
        interpretTest state (next user)
    
    | Db (SaveUser (user, next)) ->
        let newState = { state with Users = state.Users |> Map.add user.Id user }
        interpretTest newState next
    
    | Db (DeleteUser (id, next)) ->
        let exists = state.Users |> Map.containsKey id
        let newState = { state with Users = state.Users |> Map.remove id }
        interpretTest newState (next exists)
    
    | Email (SendEmail (to', subject, body, next)) ->
        let newState = { state with SentEmails = (to', subject, body) :: state.SentEmails }
        interpretTest newState next
```

## 7. Functional Domain Events

```fsharp
// Events เป็น immutable records, Handlers เป็น pure functions

// ===== Event Types =====
type DomainEvent =
    | UserCreated of {| Id: string; Email: string; Name: string; At: System.DateTime |}
    | UserUpdated of {| Id: string; Changes: Map<string, string>; At: System.DateTime |}
    | UserDeleted of {| Id: string; At: System.DateTime |}
    | OrderPlaced of {| OrderId: string; UserId: string; Total: decimal; At: System.DateTime |}
    | OrderShipped of {| OrderId: string; TrackingNumber: string; At: System.DateTime |}

// ===== Event Store =====
type EventStore<'Event> = {
    Append: 'Event -> Async<unit>
    GetAll: unit -> Async<'Event list>
    GetAfter: System.DateTime -> Async<'Event list>
}

let createInMemoryEventStore<'T> () =
    let events = ResizeArray<'T>()
    {
        Append = fun e -> async { events.Add(e) }
        GetAll = fun () -> async { return events |> Seq.toList }
        GetAfter = fun _ -> async { return events |> Seq.toList }  // Simplified
    }

// ===== Event Handlers (pure projections) =====
type UserReadModel = {
    Id: string
    Email: string
    Name: string
    OrderCount: int
    TotalSpent: decimal
    LastOrderAt: System.DateTime option
}

let initialUserReadModel id = {
    Id = id
    Email = ""
    Name = ""
    OrderCount = 0
    TotalSpent = 0m
    LastOrderAt = None
}

// Pure event projection
let applyEventToUserModel (model: UserReadModel) (event: DomainEvent) =
    match event with
    | UserCreated data when data.Id = model.Id ->
        { model with Email = data.Email; Name = data.Name }
    
    | UserUpdated data when data.Id = model.Id ->
        let newModel =
            data.Changes 
            |> Map.fold (fun m key value ->
                match key with
                | "Email" -> { m with Email = value }
                | "Name" -> { m with Name = value }
                | _ -> m) model
        newModel
    
    | OrderPlaced data when data.UserId = model.Id ->
        { model with
            OrderCount = model.OrderCount + 1
            TotalSpent = model.TotalSpent + data.Total
            LastOrderAt = Some data.At }
    
    | _ -> model

// Build read model from events (pure function)
let buildUserModel (userId: string) (events: DomainEvent list) : UserReadModel =
    events |> List.fold applyEventToUserModel (initialUserReadModel userId)

// ===== Event-Driven Workflow =====
type EventDrivenWorkflow<'State, 'Command, 'Event> = {
    InitialState: 'State
    Decide: 'State -> 'Command -> Result<'Event list, string>
    Evolve: 'State -> 'Event -> 'State
}

let runWorkflow<'State, 'Command, 'Event>
    (workflow: EventDrivenWorkflow<'State, 'Command, 'Event>)
    (existingEvents: 'Event list)
    (command: 'Command) : Result<'Event list * 'State, string> =
    
    // Rebuild state from events
    let state = existingEvents |> List.fold workflow.Evolve workflow.InitialState
    
    // Decide
    match workflow.Decide state command with
    | Error e -> Error e
    | Ok newEvents ->
        // Apply new events
        let newState = newEvents |> List.fold workflow.Evolve state
        Ok (newEvents, newState)
```

## 8. Complete Functional Architecture Example

```fsharp
// ===== Complete example: Blog system =====
module Blog =
    
    // Domain types
    type PostId = string
    type AuthorId = string
    
    type PostStatus = Draft | Published | Archived
    
    type Post = {
        Id: PostId
        Title: string
        Content: string
        AuthorId: AuthorId
        Status: PostStatus
        Tags: string list
        CreatedAt: System.DateTime
        UpdatedAt: System.DateTime
        PublishedAt: System.DateTime option
    }
    
    // Domain events
    type PostEvent =
        | PostDrafted of {| Id: PostId; Title: string; Content: string; AuthorId: AuthorId; At: System.DateTime |}
        | PostPublished of {| Id: PostId; At: System.DateTime |}
        | PostUpdated of {| Id: PostId; Title: string option; Content: string option; At: System.DateTime |}
        | PostArchived of {| Id: PostId; At: System.DateTime |}
        | TagAdded of {| Id: PostId; Tag: string; At: System.DateTime |}
        | TagRemoved of {| Id: PostId; Tag: string; At: System.DateTime |}
    
    // Commands
    type PostCommand =
        | DraftPost of title: string * content: string * authorId: AuthorId
        | PublishPost of PostId
        | UpdatePost of PostId * title: string option * content: string option
        | ArchivePost of PostId
        | AddTag of PostId * string
        | RemoveTag of PostId * string
    
    // Initial state
    let emptyPost id = {
        Id = id
        Title = ""
        Content = ""
        AuthorId = ""
        Status = Draft
        Tags = []
        CreatedAt = System.DateTime.MinValue
        UpdatedAt = System.DateTime.MinValue
        PublishedAt = None
    }
    
    // Pure: Evolve state from event
    let evolve (post: Post) (event: PostEvent) : Post =
        match event with
        | PostDrafted data ->
            { post with
                Id = data.Id
                Title = data.Title
                Content = data.Content
                AuthorId = data.AuthorId
                Status = Draft
                CreatedAt = data.At
                UpdatedAt = data.At }
        
        | PostPublished data ->
            { post with
                Status = Published
                PublishedAt = Some data.At
                UpdatedAt = data.At }
        
        | PostUpdated data ->
            { post with
                Title = data.Title |> Option.defaultValue post.Title
                Content = data.Content |> Option.defaultValue post.Content
                UpdatedAt = data.At }
        
        | PostArchived data ->
            { post with Status = Archived; UpdatedAt = data.At }
        
        | TagAdded data ->
            if post.Tags |> List.contains data.Tag then post
            else { post with Tags = post.Tags @ [data.Tag] }
        
        | TagRemoved data ->
            { post with Tags = post.Tags |> List.filter ((<>) data.Tag) }
    
    // Pure: Decide what events to produce
    let decide (post: Post) (command: PostCommand) : Result<PostEvent list, string> =
        let now = System.DateTime.UtcNow
        
        match command with
        | DraftPost (title, content, authorId) ->
            if System.String.IsNullOrWhiteSpace(title) then
                Error "Title is required"
            elif content.Length < 10 then
                Error "Content too short (min 10 chars)"
            else
                let id = System.Guid.NewGuid().ToString()
                Ok [PostDrafted {| Id = id; Title = title; Content = content; AuthorId = authorId; At = now |}]
        
        | PublishPost postId ->
            if post.Id <> postId then Error "Post ID mismatch"
            elif post.Status = Published then Error "Post already published"
            elif post.Status = Archived then Error "Cannot publish archived post"
            elif System.String.IsNullOrWhiteSpace(post.Title) then Error "Cannot publish without title"
            else Ok [PostPublished {| Id = postId; At = now |}]
        
        | UpdatePost (postId, title, content) ->
            if post.Id <> postId then Error "Post ID mismatch"
            elif post.Status = Archived then Error "Cannot update archived post"
            else
                let hasChanges = title.IsSome || content.IsSome
                if not hasChanges then Error "No changes provided"
                else Ok [PostUpdated {| Id = postId; Title = title; Content = content; At = now |}]
        
        | ArchivePost postId ->
            if post.Id <> postId then Error "Post ID mismatch"
            elif post.Status = Archived then Error "Already archived"
            else Ok [PostArchived {| Id = postId; At = now |}]
        
        | AddTag (postId, tag) ->
            if System.String.IsNullOrWhiteSpace(tag) then Error "Tag cannot be empty"
            elif post.Tags.Length >= 10 then Error "Maximum 10 tags allowed"
            elif post.Tags |> List.contains tag then Error (sprintf "Tag '%s' already exists" tag)
            else Ok [TagAdded {| Id = postId; Tag = tag; At = now |}]
        
        | RemoveTag (postId, tag) ->
            if not (post.Tags |> List.contains tag) then
                Error (sprintf "Tag '%s' not found" tag)
            else
                Ok [TagRemoved {| Id = postId; Tag = tag; At = now |}]
    
    // ===== Impure Shell =====
    type BlogPorts = {
        LoadPost: PostId -> Async<PostEvent list>
        AppendEvents: PostId -> PostEvent list -> Async<unit>
        PublishToFeed: Post -> Async<unit>
    }
    
    let handleCommand (ports: BlogPorts) (postId: PostId) (command: PostCommand) =
        async {
            // Load existing events
            let! events = ports.LoadPost postId
            
            // Rebuild current state
            let currentPost = events |> List.fold evolve (emptyPost postId)
            
            // Decide what to do (pure)
            match decide currentPost command with
            | Error e -> return Error e
            | Ok newEvents ->
            
            // Append to store
            do! ports.AppendEvents postId newEvents
            
            // Build new state
            let newPost = newEvents |> List.fold evolve currentPost
            
            // Side effects based on events
            for event in newEvents do
                match event with
                | PostPublished _ -> do! ports.PublishToFeed newPost
                | _ -> ()
            
            return Ok newPost
        }
    
    // In-memory event store for testing
    let createTestPorts () =
        let eventStore = System.Collections.Generic.Dictionary<string, ResizeArray<PostEvent>>()
        let publishedPosts = ResizeArray<Post>()
        
        {
            LoadPost = fun postId ->
                async {
                    match eventStore.TryGetValue(postId) with
                    | true, events -> return events |> Seq.toList
                    | _ -> return []
                }
            
            AppendEvents = fun postId events ->
                async {
                    if not (eventStore.ContainsKey(postId)) then
                        eventStore.[postId] <- ResizeArray()
                    for e in events do
                        eventStore.[postId].Add(e)
                }
            
            PublishToFeed = fun post ->
                async {
                    publishedPosts.Add(post)
                    printfn "Published to feed: %s" post.Title
                }
        }, publishedPosts
    
    // Demo
    let demo () =
        async {
            let ports, published = createTestPorts()
            let handle = handleCommand ports
            
            printfn "=== Blog System Demo ==="
            
            // Create a post
            let newId = "post-001"
            let! result1 = handle newId (DraftPost ("My First Post", "This is the content of my first post!", "author-001"))
            match result1 with
            | Ok post -> printfn "✓ Post drafted: %s (ID: %s)" post.Title post.Id
            | Error e -> printfn "✗ Error: %s" e
            
            // Add tags
            let! _ = handle newId (AddTag (newId, "fsharp"))
            let! _ = handle newId (AddTag (newId, "functional"))
            printfn "✓ Tags added"
            
            // Publish
            let! result2 = handle newId (PublishPost newId)
            match result2 with
            | Ok post -> printfn "✓ Post published! Status: %A, Published at: %A" post.Status post.PublishedAt
            | Error e -> printfn "✗ Error: %s" e
            
            printfn "✓ Posts in feed: %d" published.Count
        }
    
    demo() |> Async.RunSynchronously
```

## 9. สรุป Functional Architecture

```fsharp
(*
หลักการสำคัญของ Functional Architecture:

1. Functional Core, Imperative Shell
   - Core: Pure functions, no side effects, easy to test
   - Shell: Side effects at the edges (I/O, DB, Network)

2. Ports as Function Types
   - แทน interfaces ด้วย function types
   - Partial application แทน constructor injection

3. Reader Monad
   - Thread environment ผ่าน computation
   - ทำให้ dependencies explicit

4. Event Sourcing กับ Functional Patterns
   - State = fold of events
   - decide: State -> Command -> Events (pure)
   - evolve: State -> Event -> State (pure)

5. Composition over Inheritance
   - ประกอบ functions แทนที่จะ inherit classes
   
Benefits:
✓ Highly testable - pure functions test ง่าย
✓ Explicit dependencies - ไม่มี hidden state
✓ Composable - สร้าง complex behaviors จาก simple functions
✓ Immutable by default - thread-safe
*)

printfn "Functional Architecture - Complete!"
```

---

**สรุป**: Functional Architecture ใน F# ใช้ประโยชน์จาก immutability และ pure functions เพื่อสร้างระบบที่ test ง่าย ทำความเข้าใจง่าย และ compose ได้ดี โดยแยก pure logic ออกจาก side effects อย่างชัดเจน
