# Part 88 - Design Patterns ใน F#

## บทนำ (Introduction)

Design patterns ส่วนใหญ่เกิดขึ้นเพื่อแก้ปัญหาของ OOP ที่ขาด first-class functions ใน F# หลาย patterns มีวิธี functional ที่ elegant และสั้นกว่ามาก

## 1. Strategy Pattern → Function Parameters

```fsharp
// ===== OOP Strategy Pattern (verbose) =====
type ISortStrategy =
    abstract member Sort: int list -> int list

type BubbleSortStrategy() =
    interface ISortStrategy with
        member _.Sort items = 
            items |> List.sort  // Simplified

type QuickSortStrategy() =
    interface ISortStrategy with
        member _.Sort items =
            items |> List.sortBy id  // Simplified

type Sorter(strategy: ISortStrategy) =
    member _.Sort items = strategy.Sort items

// ===== Functional Strategy Pattern (simple) =====
// Strategy เป็น function ที่ pass เข้ามา

let sortWith (sorter: int list -> int list) (items: int list) =
    sorter items

// Different strategies as functions
let bubbleSort items = List.sort items
let insertionSort items = List.sortWith compare items
let randomSort items = 
    let rng = System.Random()
    items |> List.sortBy (fun _ -> rng.Next())

// Usage
let numbers = [5; 2; 8; 1; 9; 3]
let sorted1 = sortWith bubbleSort numbers
let sorted2 = sortWith insertionSort numbers
let sorted3 = sortWith (List.sortByDescending id) numbers

printfn "Ascending: %A" sorted1
printfn "Descending: %A" sorted3

// Real-world example: Payment processing strategies
type PaymentDetails = {
    Amount: decimal
    Currency: string
    CustomerInfo: string
}

type PaymentResult = { TransactionId: string; Success: bool }

// Strategies as function types
type PaymentProcessor = PaymentDetails -> Async<Result<PaymentResult, string>>

let stripeProcessor : PaymentProcessor = fun payment ->
    async {
        printfn "[Stripe] Processing %.2f %s" payment.Amount payment.Currency
        return Ok { TransactionId = sprintf "stripe_%s" (System.Guid.NewGuid().ToString("N")[..7]); Success = true }
    }

let paypalProcessor : PaymentProcessor = fun payment ->
    async {
        printfn "[PayPal] Processing %.2f %s" payment.Amount payment.Currency
        return Ok { TransactionId = sprintf "paypal_%s" (System.Guid.NewGuid().ToString("N")[..7]); Success = true }
    }

let codProcessor : PaymentProcessor = fun payment ->
    async {
        printfn "[COD] Scheduling cash collection for %.2f %s" payment.Amount payment.Currency
        return Ok { TransactionId = sprintf "cod_%s" (System.Guid.NewGuid().ToString("N")[..7]); Success = true }
    }

// Context that uses strategy
let processPayment (processor: PaymentProcessor) (payment: PaymentDetails) =
    async {
        printfn "Processing payment of %.2f %s" payment.Amount payment.Currency
        let! result = processor payment
        match result with
        | Ok r -> printfn "Success! Transaction: %s" r.TransactionId
        | Error e -> printfn "Failed: %s" e
        return result
    }

// Choose strategy at runtime
let getProcessor (method: string) : PaymentProcessor =
    match method.ToLower() with
    | "stripe" -> stripeProcessor
    | "paypal" -> paypalProcessor
    | "cod" -> codProcessor
    | _ -> fun _ -> async { return Error (sprintf "Unknown payment method: %s" method) }
```

## 2. Observer Pattern → Events/Observables

```fsharp
// ===== Functional Observer Pattern =====

// Simple event system using functions
type Observable<'T> = {
    Subscribe: ('T -> unit) -> (unit -> unit)  // returns unsubscribe function
}

module Observable =
    let create<'T> () =
        let handlers = ResizeArray<'T -> unit>()
        
        let observable = {
            Subscribe = fun handler ->
                handlers.Add(handler)
                fun () -> handlers.Remove(handler) |> ignore
        }
        
        let publish = fun (value: 'T) ->
            for handler in handlers |> Seq.toList do
                handler value
        
        observable, publish
    
    let map (f: 'A -> 'B) (obs: Observable<'A>) : Observable<'B> = {
        Subscribe = fun handler ->
            obs.Subscribe (fun a -> handler (f a))
    }
    
    let filter (predicate: 'T -> bool) (obs: Observable<'T>) : Observable<'T> = {
        Subscribe = fun handler ->
            obs.Subscribe (fun value ->
                if predicate value then handler value)
    }
    
    let merge (obs1: Observable<'T>) (obs2: Observable<'T>) : Observable<'T> = {
        Subscribe = fun handler ->
            let unsub1 = obs1.Subscribe handler
            let unsub2 = obs2.Subscribe handler
            fun () -> unsub1(); unsub2()
    }

// Usage: Stock price monitor
type StockPrice = { Symbol: string; Price: decimal; Timestamp: System.DateTime }

let priceObservable, publishPrice = Observable.create<StockPrice>()

// Filter for expensive stocks
let expensiveStocks = 
    priceObservable
    |> Observable.filter (fun p -> p.Price > 1000m)

// Transform to just the symbol and price
let priceAlerts =
    priceObservable
    |> Observable.map (fun p -> sprintf "%s: %.2f" p.Symbol p.Price)

// Subscribe
let unsub1 = priceObservable.Subscribe (fun p ->
    printfn "[Monitor] %s = %.2f" p.Symbol p.Price)

let unsub2 = expensiveStocks.Subscribe (fun p ->
    printfn "[Alert] HIGH PRICE: %s = %.2f" p.Symbol p.Price)

// Publish some prices
publishPrice { Symbol = "AAPL"; Price = 150m; Timestamp = System.DateTime.UtcNow }
publishPrice { Symbol = "GOOG"; Price = 2800m; Timestamp = System.DateTime.UtcNow }
publishPrice { Symbol = "MSFT"; Price = 380m; Timestamp = System.DateTime.UtcNow }

// Unsubscribe
unsub1()
unsub2()

// Using .NET Events (F# style)
type PriceChangedEventArgs(symbol: string, price: decimal) =
    inherit System.EventArgs()
    member _.Symbol = symbol
    member _.Price = price

type StockMonitor() =
    let priceChanged = Event<PriceChangedEventArgs>()
    let priceDropped = Event<PriceChangedEventArgs>()
    
    member _.PriceChanged = priceChanged.Publish
    member _.PriceDropped = priceDropped.Publish
    
    member _.UpdatePrice symbol price previousPrice =
        priceChanged.Trigger(PriceChangedEventArgs(symbol, price))
        if price < previousPrice then
            priceDropped.Trigger(PriceChangedEventArgs(symbol, price))
```

## 3. Command Pattern → Discriminated Unions

```fsharp
// ===== Command Pattern กับ DUs =====

// Commands as discriminated union
type TextEditorCommand =
    | InsertText of position: int * text: string
    | DeleteText of position: int * length: int
    | Bold of start: int * length: int
    | Italic of start: int * length: int
    | Undo
    | Redo
    | Save of filename: string

type EditorState = {
    Content: string
    History: TextEditorCommand list
    Future: TextEditorCommand list
}

module TextEditor =
    let empty = { Content = ""; History = []; Future = [] }
    
    let execute (state: EditorState) (command: TextEditorCommand) =
        match command with
        | InsertText (pos, text) ->
            let newContent = state.Content.[..pos-1] + text + state.Content.[pos..]
            { state with Content = newContent; History = command :: state.History; Future = [] }
        
        | DeleteText (pos, len) ->
            let newContent = state.Content.[..pos-1] + state.Content.[pos+len..]
            { state with Content = newContent; History = command :: state.History; Future = [] }
        
        | Undo ->
            match state.History with
            | [] -> state  // Nothing to undo
            | lastCmd :: rest ->
                // Reverse the last command (simplified)
                printfn "Undoing: %A" lastCmd
                { state with History = rest; Future = lastCmd :: state.Future }
        
        | Redo ->
            match state.Future with
            | [] -> state  // Nothing to redo
            | nextCmd :: rest ->
                { state with History = nextCmd :: state.History; Future = rest }
        
        | Save filename ->
            printfn "Saving to %s" filename
            state
        
        | Bold (start, len) ->
            let marked = sprintf "<b>%s</b>" state.Content.[start..start+len-1]
            let newContent = state.Content.[..start-1] + marked + state.Content.[start+len..]
            { state with Content = newContent; History = command :: state.History }
        
        | Italic (start, len) ->
            let marked = sprintf "<i>%s</i>" state.Content.[start..start+len-1]
            let newContent = state.Content.[..start-1] + marked + state.Content.[start+len..]
            { state with Content = newContent; History = command :: state.History }
    
    let executeAll state commands =
        commands |> List.fold execute state

// Demo
let demo () =
    let commands = [
        InsertText (0, "Hello, World!")
        InsertText (13, " How are you?")
        DeleteText (13, 14)
        Save "document.txt"
    ]
    
    let finalState = TextEditor.executeAll TextEditor.empty commands
    printfn "Final content: %s" finalState.Content
    printfn "History count: %d" finalState.History.Length
```

## 4. Decorator Pattern → Function Composition

```fsharp
// ===== Decorator Pattern กับ Function Composition =====

// Base function type
type Operation<'T> = 'T -> 'T

// Decorators as higher-order functions
let withLogging (logger: string -> unit) (name: string) (operation: Operation<'T>) : Operation<'T> =
    fun input ->
        logger (sprintf "Calling %s" name)
        let result = operation input
        logger (sprintf "%s completed" name)
        result

let withTiming (operation: Operation<'T>) : Operation<'T> =
    fun input ->
        let sw = System.Diagnostics.Stopwatch.StartNew()
        let result = operation input
        sw.Stop()
        printfn "Execution time: %dms" sw.ElapsedMilliseconds
        result

let withRetry (maxRetries: int) (operation: 'T -> Result<'U, string>) : 'T -> Result<'U, string> =
    fun input ->
        let mutable attempt = 0
        let mutable result = Error "Not started"
        while attempt < maxRetries && Result.isError result do
            attempt <- attempt + 1
            try
                result <- operation input
            with ex ->
                result <- Error ex.Message
                if attempt < maxRetries then
                    printfn "Retry %d/%d after error: %s" attempt maxRetries ex.Message
        result

let withCaching (cache: System.Collections.Generic.Dictionary<'K, 'V>) 
               (keyFn: 'K -> string) 
               (operation: 'K -> 'V) : 'K -> 'V =
    fun key ->
        let keyStr = keyFn key
        match cache.TryGetValue(keyStr |> hash |> string |> int |> ignore; key) with
        | _ ->
            // Simplified cache check
            if cache.ContainsKey(keyStr |> fun _ -> key) then
                cache.[keyStr |> fun _ -> key]
            else
                let result = operation key
                // cache.[key] <- result
                result

// Apply decorators using >> (function composition)
let processData (data: string) =
    data.ToUpper().Trim()

let decoratedProcess =
    processData
    |> withLogging printfn "processData"
    |> withTiming

printfn "%s" (decoratedProcess "  hello world  ")

// HTTP client with decorators
type HttpClient = string -> Async<Result<string, string>>

let withHttpLogging (logger: string -> unit) (client: HttpClient) : HttpClient =
    fun url ->
        async {
            logger (sprintf "GET %s" url)
            let! result = client url
            match result with
            | Ok _ -> logger "200 OK"
            | Error e -> logger (sprintf "Error: %s" e)
            return result
        }

let withHttpRetry (maxRetries: int) (client: HttpClient) : HttpClient =
    fun url ->
        async {
            let mutable attempt = 0
            let mutable result = Error "Not started"
            while attempt < maxRetries && Result.isError result do
                attempt <- attempt + 1
                let! r = client url
                result <- r
                match result with
                | Error _ when attempt < maxRetries ->
                    do! Async.Sleep (1000 * attempt)
                | _ -> ()
            return result
        }

let withHttpTimeout (seconds: int) (client: HttpClient) : HttpClient =
    fun url ->
        async {
            let! result = 
                Async.StartChild(client url, millisecondsTimeout = seconds * 1000)
                |> Async.bind id
            return result
        }
```

## 5. Factory Pattern → Constructors/Modules

```fsharp
// ===== Factory Pattern =====

// Simple factory using discriminated unions
type DatabaseType = SQLite | PostgreSQL | SqlServer | InMemory

type IDatabase =
    abstract member Query: string -> Async<string list>
    abstract member Execute: string -> Async<int>
    abstract member Dispose: unit -> unit

// Concrete implementations
type SqliteDatabase(path: string) =
    interface IDatabase with
        member _.Query sql = async { return [sprintf "SQLite result for: %s" sql] }
        member _.Execute sql = async { return 1 }
        member _.Dispose () = ()

type PostgreSQLDatabase(connectionString: string) =
    interface IDatabase with
        member _.Query sql = async { return [sprintf "PostgreSQL result for: %s" sql] }
        member _.Execute sql = async { return 1 }
        member _.Dispose () = ()

type InMemoryDatabase() =
    let storage = System.Collections.Generic.Dictionary<string, string>()
    interface IDatabase with
        member _.Query _ = async { return storage.Values |> Seq.toList }
        member _.Execute _ = async { return 1 }
        member _.Dispose () = storage.Clear()

// Factory function
let createDatabase (dbType: DatabaseType) (config: Map<string, string>) : IDatabase =
    match dbType with
    | SQLite ->
        let path = config |> Map.tryFind "path" |> Option.defaultValue ":memory:"
        SqliteDatabase(path) :> IDatabase
    
    | PostgreSQL ->
        let conn = config |> Map.tryFind "connectionString" |> Option.defaultValue ""
        PostgreSQLDatabase(conn) :> IDatabase
    
    | InMemory ->
        InMemoryDatabase() :> IDatabase
    
    | SqlServer ->
        // Fallback
        InMemoryDatabase() :> IDatabase

// Abstract factory using records of functions
type DatabaseFactory = {
    CreateConnection: string -> IDatabase
    CreateMigrator: IDatabase -> (unit -> Async<unit>)
    CreateSeeder: IDatabase -> (unit -> Async<unit>)
}

let sqliteFactory : DatabaseFactory = {
    CreateConnection = fun path -> SqliteDatabase(path) :> IDatabase
    CreateMigrator = fun db -> fun () -> async { printfn "Running SQLite migrations" }
    CreateSeeder = fun db -> fun () -> async { printfn "Seeding SQLite" }
}

let testFactory : DatabaseFactory = {
    CreateConnection = fun _ -> InMemoryDatabase() :> IDatabase
    CreateMigrator = fun _ -> fun () -> async { () }
    CreateSeeder = fun _ -> fun () -> async { () }
}
```

## 6. Builder Pattern → Computation Expressions

```fsharp
// ===== Builder Pattern กับ CE =====

// HTTP Request Builder
type RequestBuilder = {
    Url: string
    Method: string
    Headers: Map<string, string>
    Body: string option
    Timeout: int
    RetryCount: int
}

module RequestBuilder =
    let empty = {
        Url = ""
        Method = "GET"
        Headers = Map.empty
        Body = None
        Timeout = 30
        RetryCount = 0
    }
    
    let withUrl url builder = { builder with Url = url }
    let withMethod method builder = { builder with Method = method }
    let withHeader key value builder = 
        { builder with Headers = builder.Headers |> Map.add key value }
    let withBody body builder = { builder with Body = Some body }
    let withTimeout seconds builder = { builder with Timeout = seconds }
    let withRetry count builder = { builder with RetryCount = count }
    let withJson body builder = 
        { builder with 
            Body = Some body
            Headers = builder.Headers |> Map.add "Content-Type" "application/json" }
    let withAuth token builder =
        builder |> withHeader "Authorization" (sprintf "Bearer %s" token)
    
    let build builder = builder  // Returns the request

// Method chaining style (F# pipe style)
let request =
    RequestBuilder.empty
    |> RequestBuilder.withUrl "https://api.example.com/orders"
    |> RequestBuilder.withMethod "POST"
    |> RequestBuilder.withAuth "my-token"
    |> RequestBuilder.withJson """{"orderId": "123"}"""
    |> RequestBuilder.withTimeout 60
    |> RequestBuilder.withRetry 3
    |> RequestBuilder.build

printfn "Request: %A" request

// ===== Computation Expression Builder =====
type QueryBuilder<'T>() =
    
    let mutable filters: ('T -> bool) list = []
    let mutable sorts: ('T -> 'T -> int) list = []
    let mutable limit: int option = None
    let mutable offset: int = 0
    
    member _.Where (predicate: 'T -> bool) =
        filters <- predicate :: filters
        this
    
    member _.OrderBy (comparer: 'T -> 'T -> int) =
        sorts <- comparer :: sorts
        this
    
    member _.Take n =
        limit <- Some n
        this
    
    member _.Skip n =
        offset <- n
        this
    
    member _.Execute (source: 'T list) : 'T list =
        let filtered = 
            filters
            |> List.fold (fun items f -> items |> List.filter f) source
        
        let sorted =
            sorts
            |> List.fold (fun items s -> items |> List.sortWith s) filtered
        
        let paged =
            sorted
            |> List.skip (min offset sorted.Length)
        
        match limit with
        | Some n -> paged |> List.truncate n
        | None -> paged

// Usage
type Product = { Id: int; Name: string; Price: decimal; Category: string; InStock: bool }

let products = [
    { Id = 1; Name = "Widget A"; Price = 100m; Category = "Electronics"; InStock = true }
    { Id = 2; Name = "Widget B"; Price = 200m; Category = "Electronics"; InStock = false }
    { Id = 3; Name = "Gadget X"; Price = 500m; Category = "Electronics"; InStock = true }
    { Id = 4; Name = "Book Y"; Price = 50m; Category = "Books"; InStock = true }
    { Id = 5; Name = "Shirt Z"; Price = 80m; Category = "Clothing"; InStock = true }
]

let query = QueryBuilder<Product>()
let results = 
    query
        .Where(fun p -> p.Category = "Electronics")
        .Where(fun p -> p.InStock)
        .OrderBy(fun a b -> compare a.Price b.Price)
        .Take(2)
        .Execute(products)

printfn "Query results: %A" (results |> List.map (fun p -> p.Name))
```

## 7. Template Method → Higher-Order Functions

```fsharp
// ===== Template Method Pattern =====

// Template: Define algorithm skeleton, let subclasses fill in steps
// Functional version: Pass the steps as functions

// Data processing template
let processData 
    (loadData: unit -> Async<string list>)
    (validateItem: string -> Result<string, string>)
    (transformItem: string -> string)
    (saveResults: string list -> Async<unit>) =
    async {
        // Step 1: Load
        let! rawData = loadData()
        printfn "Loaded %d items" rawData.Length
        
        // Step 2: Validate
        let validItems, errors = 
            rawData 
            |> List.partitionMap (fun item ->
                match validateItem item with
                | Ok v -> Choice1Of2 v
                | Error e -> Choice2Of2 (sprintf "Error on '%s': %s" item e))
        
        if errors.Length > 0 then
            printfn "Validation errors: %d" errors.Length
            for e in errors do printfn "  %s" e
        
        // Step 3: Transform
        let transformed = validItems |> List.map transformItem
        
        // Step 4: Save
        do! saveResults transformed
        
        return {| Processed = transformed.Length; Errors = errors.Length |}
    }

// Concrete implementations
let csvLoader () = async {
    return ["alice,30"; "bob,25"; "charlie,invalid_age"; "diana,35"]
}

let validateCsvRow (row: string) =
    let parts = row.Split(',')
    if parts.Length <> 2 then Error "Invalid format"
    elif System.String.IsNullOrWhiteSpace(parts.[0]) then Error "Name is empty"
    else
        match System.Int32.TryParse(parts.[1]) with
        | false, _ -> Error "Age is not a number"
        | true, age when age < 0 || age > 150 -> Error "Age out of range"
        | true, _ -> Ok row

let transformCsvRow (row: string) =
    let parts = row.Split(',')
    sprintf "Name: %s, Age: %s years old" parts.[0] parts.[1]

let consoleSaver results = async {
    printfn "Results:"
    for r in results do printfn "  %s" r
}

// Run the template
let run () =
    processData csvLoader validateCsvRow transformCsvRow consoleSaver
    |> Async.RunSynchronously
    |> fun r -> printfn "Processed %d, Errors %d" r.Processed r.Errors

run()
```

## 8. Visitor Pattern → Pattern Matching

```fsharp
// ===== Visitor Pattern กับ Pattern Matching =====

// Expression tree
type Expr =
    | Number of decimal
    | Variable of string
    | Add of Expr * Expr
    | Subtract of Expr * Expr
    | Multiply of Expr * Expr
    | Divide of Expr * Expr
    | Negate of Expr
    | IfPositive of condition: Expr * thenExpr: Expr * elseExpr: Expr

// "Visitors" as functions using pattern matching
let rec evaluate (vars: Map<string, decimal>) (expr: Expr) : Result<decimal, string> =
    match expr with
    | Number n -> Ok n
    
    | Variable name ->
        match vars |> Map.tryFind name with
        | Some v -> Ok v
        | None -> Error (sprintf "Undefined variable: %s" name)
    
    | Add (left, right) ->
        match evaluate vars left, evaluate vars right with
        | Ok l, Ok r -> Ok (l + r)
        | Error e, _ | _, Error e -> Error e
    
    | Subtract (left, right) ->
        match evaluate vars left, evaluate vars right with
        | Ok l, Ok r -> Ok (l - r)
        | Error e, _ | _, Error e -> Error e
    
    | Multiply (left, right) ->
        match evaluate vars left, evaluate vars right with
        | Ok l, Ok r -> Ok (l * r)
        | Error e, _ | _, Error e -> Error e
    
    | Divide (left, right) ->
        match evaluate vars left, evaluate vars right with
        | Ok l, Ok r ->
            if r = 0m then Error "Division by zero"
            else Ok (l / r)
        | Error e, _ | _, Error e -> Error e
    
    | Negate expr ->
        evaluate vars expr |> Result.map (fun v -> -v)
    
    | IfPositive (cond, thenExpr, elseExpr) ->
        match evaluate vars cond with
        | Error e -> Error e
        | Ok condVal ->
            if condVal > 0m then evaluate vars thenExpr
            else evaluate vars elseExpr

// Another visitor: Pretty printer
let rec prettyPrint (expr: Expr) : string =
    match expr with
    | Number n -> string n
    | Variable name -> name
    | Add (l, r) -> sprintf "(%s + %s)" (prettyPrint l) (prettyPrint r)
    | Subtract (l, r) -> sprintf "(%s - %s)" (prettyPrint l) (prettyPrint r)
    | Multiply (l, r) -> sprintf "(%s * %s)" (prettyPrint l) (prettyPrint r)
    | Divide (l, r) -> sprintf "(%s / %s)" (prettyPrint l) (prettyPrint r)
    | Negate e -> sprintf "-(${%s})" (prettyPrint e)
    | IfPositive (c, t, e) -> sprintf "if (%s > 0) then %s else %s" (prettyPrint c) (prettyPrint t) (prettyPrint e)

// Type checker visitor
let rec typeCheck (expr: Expr) : Result<unit, string list> =
    match expr with
    | Number _ -> Ok ()
    | Variable _ -> Ok ()
    | Add (l, r) | Subtract (l, r) | Multiply (l, r) | Divide (l, r) ->
        match typeCheck l, typeCheck r with
        | Ok (), Ok () -> Ok ()
        | Error e1, Error e2 -> Error (e1 @ e2)
        | Error e, _ | _, Error e -> Error e
    | Negate e -> typeCheck e
    | IfPositive (c, t, e) ->
        [typeCheck c; typeCheck t; typeCheck e]
        |> List.collect (function Error e -> e | Ok () -> [])
        |> function [] -> Ok () | errors -> Error errors

// Demo
let expr = 
    Add(
        Multiply(Variable "x", Number 2m),
        IfPositive(
            Variable "x",
            Number 10m,
            Negate(Number 5m)
        )
    )

let vars = Map.ofList [("x", 3m)]

printfn "Expression: %s" (prettyPrint expr)
match evaluate vars expr with
| Ok v -> printfn "Result: %M" v
| Error e -> printfn "Error: %s" e
```

## 9. State Machine กับ Discriminated Unions

```fsharp
// ===== State Machine Pattern =====

// Traffic Light State Machine
type TrafficLight =
    | Red
    | Yellow
    | Green

let nextState = function
    | Red -> Green
    | Green -> Yellow
    | Yellow -> Red

let lightDuration = function
    | Red -> 30
    | Green -> 25
    | Yellow -> 5

let lightColor = function
    | Red -> "🔴 STOP"
    | Yellow -> "🟡 CAUTION"
    | Green -> "🟢 GO"

// Simulate traffic light
let simulateLight startState steps =
    (startState, [0..steps-1])
    ||> List.scan (fun state _ -> nextState state)
    |> List.map (fun state -> sprintf "%s (%ds)" (lightColor state) (lightDuration state))

printfn "Traffic light simulation:"
simulateLight Red 6 |> List.iter (printfn "  %s")

// ===== Order State Machine =====
type OrderState =
    | PendingPayment
    | PaymentConfirmed
    | Processing
    | ReadyToShip
    | Shipped
    | Delivered
    | Cancelled
    | Refunded

type OrderTransition =
    | ConfirmPayment
    | StartProcessing
    | MarkReadyToShip
    | MarkShipped of trackingNumber: string
    | MarkDelivered
    | Cancel of reason: string
    | RequestRefund

type StateTransitionResult =
    | Transitioned of OrderState
    | InvalidTransition of from: OrderState * attempted: OrderTransition * reason: string

let transition (state: OrderState) (event: OrderTransition) : StateTransitionResult =
    match state, event with
    | PendingPayment, ConfirmPayment ->
        Transitioned PaymentConfirmed
    
    | PaymentConfirmed, StartProcessing ->
        Transitioned Processing
    
    | Processing, MarkReadyToShip ->
        Transitioned ReadyToShip
    
    | ReadyToShip, MarkShipped trackingNumber ->
        printfn "Shipment created: %s" trackingNumber
        Transitioned Shipped
    
    | Shipped, MarkDelivered ->
        Transitioned Delivered
    
    | PendingPayment, Cancel reason ->
        printfn "Cancelled before payment: %s" reason
        Transitioned Cancelled
    
    | PaymentConfirmed, Cancel reason ->
        printfn "Cancelled after payment, refund needed: %s" reason
        Transitioned Cancelled  // Would also trigger refund
    
    | Delivered, RequestRefund ->
        Transitioned Refunded
    
    | Cancelled, _ ->
        InvalidTransition (state, event, "Cannot process cancelled order")
    
    | Delivered, _ ->
        InvalidTransition (state, event, "Order already delivered")
    
    | state, transition ->
        InvalidTransition (state, transition, sprintf "Cannot %A order in %A state" transition state)

// Demo state machine
let processOrder () =
    let events = [
        ConfirmPayment
        StartProcessing
        MarkReadyToShip
        MarkShipped "TH1234567890"
        MarkDelivered
    ]
    
    let finalState =
        events |> List.fold (fun state event ->
            match transition state event with
            | Transitioned newState ->
                printfn "✓ %A -> %A" state newState
                newState
            | InvalidTransition (from, evt, reason) ->
                printfn "✗ Cannot transition from %A with %A: %s" from evt reason
                from
        ) PendingPayment
    
    printfn "Final state: %A" finalState

processOrder()
```

## 10. Interpreter Pattern

```fsharp
// ===== Interpreter Pattern =====

// Domain Specific Language (DSL) for data queries
type QueryExpr =
    | SelectAll
    | SelectFields of string list
    | Where of condition: FilterExpr
    | OrderBy of field: string * descending: bool
    | Limit of int
    | Skip of int
    | Join of table: string * on: FilterExpr

and FilterExpr =
    | Equals of field: string * value: obj
    | NotEquals of field: string * value: obj
    | GreaterThan of field: string * value: decimal
    | LessThan of field: string * value: decimal
    | Contains of field: string * substring: string
    | And of FilterExpr * FilterExpr
    | Or of FilterExpr * FilterExpr
    | Not of FilterExpr

// Query builder DSL
type Query = {
    Table: string
    Expressions: QueryExpr list
}

module Query =
    let from table = { Table = table; Expressions = [] }
    let select fields query = { query with Expressions = query.Expressions @ [SelectFields fields] }
    let selectAll query = { query with Expressions = query.Expressions @ [SelectAll] }
    let where filter query = { query with Expressions = query.Expressions @ [Where filter] }
    let orderBy field desc query = { query with Expressions = query.Expressions @ [OrderBy (field, desc)] }
    let take n query = { query with Expressions = query.Expressions @ [Limit n] }
    let skip n query = { query with Expressions = query.Expressions @ [Skip n] }

// SQL interpreter
let interpretToSQL (query: Query) : string =
    let mutable sql = sprintf "SELECT "
    let mutable whereClause = ""
    let mutable orderClause = ""
    let mutable limitClause = ""
    let mutable offsetClause = ""
    let mutable selectedFields = "*"
    
    let rec filterToSql = function
        | Equals (f, v) -> sprintf "%s = '%A'" f v
        | NotEquals (f, v) -> sprintf "%s != '%A'" f v
        | GreaterThan (f, v) -> sprintf "%s > %M" f v
        | LessThan (f, v) -> sprintf "%s < %M" f v
        | Contains (f, s) -> sprintf "%s LIKE '%%%s%%'" f s
        | And (l, r) -> sprintf "(%s AND %s)" (filterToSql l) (filterToSql r)
        | Or (l, r) -> sprintf "(%s OR %s)" (filterToSql l) (filterToSql r)
        | Not e -> sprintf "NOT (%s)" (filterToSql e)
    
    for expr in query.Expressions do
        match expr with
        | SelectAll -> selectedFields <- "*"
        | SelectFields fields -> selectedFields <- String.concat ", " fields
        | Where filter -> whereClause <- sprintf " WHERE %s" (filterToSql filter)
        | OrderBy (field, desc) ->
            orderClause <- sprintf " ORDER BY %s %s" field (if desc then "DESC" else "ASC")
        | Limit n -> limitClause <- sprintf " LIMIT %d" n
        | Skip n -> offsetClause <- sprintf " OFFSET %d" n
        | Join (table, on) ->
            sql <- sql + sprintf " JOIN %s ON %s" table (filterToSql on)
    
    sprintf "SELECT %s FROM %s%s%s%s%s"
        selectedFields query.Table whereClause orderClause limitClause offsetClause

// Demo DSL
let q = 
    Query.from "products"
    |> Query.select ["id"; "name"; "price"]
    |> Query.where (
        And(
            Equals ("category", "Electronics" :> obj),
            GreaterThan ("price", 100m)
        ))
    |> Query.orderBy "price" false
    |> Query.take 10
    |> Query.skip 20

printfn "Generated SQL:"
printfn "%s" (interpretToSQL q)
```

## 11. สรุป Design Patterns ใน F#

```fsharp
// ===== Pattern Mapping Summary =====

(*
OOP Pattern          → F# Functional Equivalent
─────────────────────────────────────────────────
Strategy             → Function parameters (HOF)
Observer             → Events / IObservable<T>
Command              → Discriminated Unions + fold
Decorator            → Function composition (>>)
Factory              → Module functions / factory functions  
Builder              → Pipe operator + builder records
Template Method      → Higher-order functions with callbacks
Visitor              → Pattern matching on DUs
State Machine        → DUs + transition function
Iterator             → Seq module + computation expressions
Chain of Responsibility → Middleware functions chained
Proxy                → Function wrapping
Memento             → Immutable state + history list
Interpreter         → Recursive pattern matching
Composite           → Recursive DUs
Flyweight           → Memoization
Singleton           → Module-level let bindings

Key Insight:
- Many OOP patterns solve problems that F# solves at language level
- First-class functions replace Strategy, Command, Observer
- Immutability replaces Memento
- DUs replace Visitor, State Machine
- Pattern matching replaces Chain of Responsibility
- Modules replace Singleton, Flyweight

Benefits in F#:
✓ Less boilerplate
✓ More type-safe
✓ Easier to reason about
✓ Composable
*)

printfn "Design Patterns in F# - Complete!"
printfn "Many OOP patterns become simple functions in F#"
```

---

**สรุป**: ใน F# design patterns จาก OOP โลกมักกลายเป็นโครงสร้างภาษาธรรมดา เช่น function parameters (Strategy), discriminated unions (Command/Visitor), และ function composition (Decorator) ทำให้ code สั้นกว่าและ type-safe กว่ามาก
