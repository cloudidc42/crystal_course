# Part 34 - นิพจน์วัตถุ (Object Expressions)

## บทนำ (Introduction)

Object expressions เป็นฟีเจอร์ที่ทรงพลังใน F# ที่ให้เราสร้าง object ที่ implement interface หรือ abstract class ได้แบบ inline โดยไม่ต้องนิยาม class ใหม่ เป็นวิธีที่กระชับและสะดวกมาก

Object expressions in F# allow you to create objects that implement interfaces or abstract classes inline, without defining a new class. They are concise and very convenient.

---

## 1. Object Expression Syntax

```fsharp
// ไวยากรณ์พื้นฐาน
// { new InterfaceOrAbstractClass with
//     member this.Method(...) = ...
//     member this.Property = ... }

type IGreeter =
    abstract member Greet: name: string -> string

// สร้าง object ด้วย object expression
let englishGreeter =
    { new IGreeter with
        member this.Greet(name) = sprintf "Hello, %s!" name }

let thaiGreeter =
    { new IGreeter with
        member this.Greet(name) = sprintf "สวัสดี, %s!" name }

let formalGreeter =
    { new IGreeter with
        member this.Greet(name) = sprintf "Good day, %s. How do you do?" name }

// ใช้ object expressions
let greeters = [englishGreeter; thaiGreeter; formalGreeter]
for greeter in greeters do
    printfn "%s" (greeter.Greet("World"))

// Object expression ที่มี closure
let makeGreeter (prefix: string) (suffix: string) =
    { new IGreeter with
        member this.Greet(name) = sprintf "%s %s %s" prefix name suffix }

let casualGreeter = makeGreeter "Hey," "!"
let politeGreeter = makeGreeter "Dear" ", welcome."
printfn "%s" (casualGreeter.Greet("Alice"))  // Hey, Alice !
printfn "%s" (politeGreeter.Greet("Bob"))    // Dear Bob , welcome.
```

---

## 2. Implementing Interfaces Inline

```fsharp
type IShape =
    abstract member Area: float
    abstract member Perimeter: float
    abstract member Name: string
    abstract member Draw: unit -> string

// สร้าง shapes ด้วย object expressions
let makeCircle (radius: float) =
    let pi = System.Math.PI
    { new IShape with
        member this.Area = pi * radius * radius
        member this.Perimeter = 2.0 * pi * radius
        member this.Name = sprintf "Circle(r=%.2f)" radius
        member this.Draw() = 
            sprintf "○ Circle with radius %.2f" radius }

let makeRectangle (w: float) (h: float) =
    { new IShape with
        member this.Area = w * h
        member this.Perimeter = 2.0 * (w + h)
        member this.Name = sprintf "Rectangle(%.2f×%.2f)" w h
        member this.Draw() = 
            sprintf "▬ Rectangle %.2f × %.2f" w h }

let makeTriangle (a: float) (b: float) (c: float) =
    let s = (a + b + c) / 2.0
    let area = sqrt (s * (s-a) * (s-b) * (s-c))
    { new IShape with
        member this.Area = area
        member this.Perimeter = a + b + c
        member this.Name = sprintf "Triangle(%.2f,%.2f,%.2f)" a b c
        member this.Draw() =
            sprintf "△ Triangle with sides %.2f, %.2f, %.2f" a b c }

// ใช้ shapes
let shapes = [
    makeCircle 5.0
    makeRectangle 4.0 3.0
    makeTriangle 3.0 4.0 5.0
]

printfn "Shapes:"
for shape in shapes do
    printfn "  %s" (shape.Draw())
    printfn "    Area: %.2f, Perimeter: %.2f" shape.Area shape.Perimeter

let totalArea = shapes |> List.sumBy (fun s -> s.Area)
printfn "Total area: %.2f" totalArea
```

---

## 3. Implementing Abstract Base Classes

```fsharp
[<AbstractClass>]
type Animal(name: string) =
    abstract member Sound: string
    abstract member Move: unit -> string
    
    member this.Name = name
    
    member this.Describe() =
        sprintf "%s makes sound '%s' and %s" name this.Sound (this.Move())

// สร้าง animal ด้วย object expression (implement abstract class)
let makeDog (name: string) =
    { new Animal(name) with
        override this.Sound = "Woof"
        override this.Move() = sprintf "%s runs" name }

let makeCat (name: string) =
    { new Animal(name) with
        override this.Sound = "Meow"
        override this.Move() = sprintf "%s sneaks" name }

let makeParrot (name: string) (phrase: string) =
    { new Animal(name) with
        override this.Sound = phrase
        override this.Move() = sprintf "%s flies" name }

let dog = makeDog "Rex"
let cat = makeCat "Whiskers"
let parrot = makeParrot "Polly" "Hello!"

let animals: Animal list = [dog; cat; parrot]
for animal in animals do
    printfn "%s" (animal.Describe())
```

---

## 4. Use Cases: Testing Doubles

```fsharp
// Interfaces สำหรับ dependency injection
type IEmailService =
    abstract member Send: to_: string -> subject: string -> body: string -> bool

type IDatabase =
    abstract member GetUser: id: int -> {| Id: int; Name: string; Email: string |} option
    abstract member SaveUser: name: string -> email: string -> int

type ILogger =
    abstract member Info: string -> unit
    abstract member Error: string -> unit

// Service ที่ใช้ dependencies
type UserService(db: IDatabase, email: IEmailService, logger: ILogger) =
    member this.CreateUser(name: string, userEmail: string) =
        try
            logger.Info(sprintf "Creating user: %s" name)
            let userId = db.SaveUser name userEmail
            let sent = email.Send userEmail "Welcome!" (sprintf "Welcome, %s!" name)
            logger.Info(sprintf "User created with id=%d, email sent=%b" userId sent)
            Ok userId
        with ex ->
            logger.Error(sprintf "Failed to create user: %s" ex.Message)
            Error ex.Message
    
    member this.GetUser(id: int) =
        match db.GetUser(id) with
        | Some user -> 
            logger.Info(sprintf "Found user: %s" user.Name)
            Some user
        | None ->
            logger.Info(sprintf "User not found: %d" id)
            None

// สร้าง test doubles ด้วย object expressions
let createMockDb (shouldFail: bool) =
    let users = System.Collections.Generic.Dictionary<int, {| Id: int; Name: string; Email: string |}>()
    let mutable nextId = 1
    
    { new IDatabase with
        member this.GetUser(id) =
            match users.TryGetValue(id) with
            | true, user -> Some user
            | _ -> None
        
        member this.SaveUser(name)(email) =
            if shouldFail then failwith "Database error!"
            let id = nextId
            users.[id] <- {| Id = id; Name = name; Email = email |}
            nextId <- nextId + 1
            id }

let createMockEmail (shouldFail: bool) =
    let sent = System.Collections.Generic.List<string * string * string>()
    { new IEmailService with
        member this.Send(to_)(subject)(body) =
            if shouldFail then false
            else
                sent.Add((to_, subject, body))
                true },
    sent  // return both the interface and the sent list for verification

let createMockLogger () =
    let logs = System.Collections.Generic.List<string>()
    { new ILogger with
        member this.Info(msg) = logs.Add(sprintf "INFO: %s" msg)
        member this.Error(msg) = logs.Add(sprintf "ERROR: %s" msg) },
    logs

// ทดสอบ happy path
printfn "=== Happy Path Test ==="
let db = createMockDb false
let emailService, sentEmails = createMockEmail false
let logger, logs = createMockLogger()

let service = UserService(db, emailService, logger)
match service.CreateUser("สมชาย", "somchai@test.com") with
| Ok id -> printfn "User created with id: %d" id
| Error msg -> printfn "Error: %s" msg

printfn "Emails sent: %d" sentEmails.Count
printfn "Log entries: %d" logs.Count
logs |> Seq.iter (printfn "  %s")

// ทดสอบ failure case
printfn "\n=== Failure Test ==="
let badDb = createMockDb true
let service2 = UserService(badDb, emailService, logger)
match service2.CreateUser("สมหญิง", "somying@test.com") with
| Ok id -> printfn "User created with id: %d" id
| Error msg -> printfn "Error: %s" msg
```

---

## 5. Mocking with Object Expressions

```fsharp
// Configurable mock สำหรับ testing scenarios
type MockBuilder<'T>() =
    let mutable calls: string list = []
    let mutable behaviors: System.Collections.Generic.Dictionary<string, obj> = 
        System.Collections.Generic.Dictionary<string, obj>()
    
    member this.Setup(methodName: string, returnValue: obj) =
        behaviors.[methodName] <- returnValue
        this
    
    member this.GetReturn<'R>(methodName: string, defaultValue: 'R) =
        match behaviors.TryGetValue(methodName) with
        | true, value -> value :?> 'R
        | _ -> defaultValue
    
    member this.TrackCall(methodName: string) =
        calls <- methodName :: calls
    
    member this.VerifyCall(methodName: string) =
        calls |> List.contains methodName
    
    member this.CallCount(methodName: string) =
        calls |> List.filter ((=) methodName) |> List.length

// Fluent mock builder
type IProductRepository =
    abstract member FindById: int -> {| Id: int; Name: string; Price: decimal |} option
    abstract member FindAll: unit -> {| Id: int; Name: string; Price: decimal |} list
    abstract member Save: {| Id: int; Name: string; Price: decimal |} -> bool
    abstract member Delete: int -> bool

let createProductRepoMock () =
    let callLog = System.Collections.Generic.List<string * obj list>()
    let products = System.Collections.Generic.Dictionary<int, {| Id: int; Name: string; Price: decimal |}>()
    
    // Seed data
    products.[1] <- {| Id = 1; Name = "Laptop"; Price = 25000M |}
    products.[2] <- {| Id = 2; Name = "Mouse"; Price = 500M |}
    products.[3] <- {| Id = 3; Name = "Keyboard"; Price = 1500M |}
    
    let mock = 
        { new IProductRepository with
            member this.FindById(id) =
                callLog.Add("FindById", [box id])
                match products.TryGetValue(id) with
                | true, p -> Some p
                | _ -> None
            
            member this.FindAll() =
                callLog.Add("FindAll", [])
                products.Values |> Seq.toList
            
            member this.Save(product) =
                callLog.Add("Save", [box product])
                products.[product.Id] <- product
                true
            
            member this.Delete(id) =
                callLog.Add("Delete", [box id])
                products.Remove(id) }
    
    mock, callLog

// Product service
type ProductService(repo: IProductRepository) =
    member this.GetProduct(id) = repo.FindById(id)
    
    member this.GetAllProducts() = repo.FindAll()
    
    member this.UpdatePrice(id, newPrice: decimal) =
        match repo.FindById(id) with
        | Some product ->
            let updated = {| product with Price = newPrice |}
            repo.Save(updated) |> ignore
            Some updated
        | None -> None
    
    member this.DeleteProduct(id) = repo.Delete(id)

// ทดสอบ
let (repoMock, callLog) = createProductRepoMock()
let productService = ProductService(repoMock)

printfn "=== Product Service Tests ==="
let allProducts = productService.GetAllProducts()
printfn "All products: %d" allProducts.Length

let laptop = productService.GetProduct(1)
match laptop with
| Some p -> printfn "Found: %s at %.2M" p.Name p.Price
| None -> printfn "Not found"

let updated = productService.UpdatePrice(1, 22000M)
match updated with
| Some p -> printfn "Updated price to %.2M" p.Price
| None -> printfn "Update failed"

printfn "\nCall log:"
for (method, args) in callLog do
    printfn "  %s(%A)" method args
```

---

## 6. IDisposable Object Expression

```fsharp
// สร้าง IDisposable objects ด้วย object expression

// Simple disposable wrapper
let makeDisposable (cleanup: unit -> unit) =
    { new System.IDisposable with
        member this.Dispose() = cleanup() }

// ทดสอบ
printfn "=== IDisposable Object Expressions ==="
use resource1 = makeDisposable (fun () -> printfn "Resource 1 released")
use resource2 = makeDisposable (fun () -> printfn "Resource 2 released")
printfn "Using resources..."
// Resources released when leaving scope

// Disposable ที่ track state
let makeTrackedDisposable (name: string) =
    let mutable isDisposed = false
    { new System.IDisposable with
        member this.Dispose() =
            if not isDisposed then
                printfn "Disposing: %s" name
                isDisposed <- true }

// Resource pool simulation
type ResourcePool<'T>(factory: unit -> 'T, cleanup: 'T -> unit, maxSize: int) =
    let available = System.Collections.Generic.Queue<'T>()
    let mutable created = 0
    
    member this.Acquire() =
        let resource =
            if available.Count > 0 then
                available.Dequeue()
            elif created < maxSize then
                created <- created + 1
                let r = factory()
                printfn "Created resource #%d" created
                r
            else
                failwith "Pool exhausted"
        
        // Return a disposable that returns resource to pool
        { new System.IDisposable with
            member this.Dispose() =
                cleanup resource
                available.Enqueue(resource)
                printfn "Resource returned to pool" },
        resource
    
    member this.Available = available.Count
    member this.Created = created

// Simulated connection pool
let connectionPool = ResourcePool<string>(
    factory = (fun () -> sprintf "Connection_%d" (System.Random.Shared.Next(1000, 9999))),
    cleanup = (fun conn -> printfn "Cleaned connection: %s" conn),
    maxSize = 3
)

// ใช้ pool
let (disposable1, conn1) = connectionPool.Acquire()
printfn "Got: %s" conn1

let (disposable2, conn2) = connectionPool.Acquire()
printfn "Got: %s" conn2

printfn "Pool: %d available, %d created" connectionPool.Available connectionPool.Created

disposable1.Dispose()
printfn "After release: %d available" connectionPool.Available

let (disposable3, conn3) = connectionPool.Acquire()  // Reuses released connection
printfn "Reused: %s" conn3
disposable2.Dispose()
disposable3.Dispose()
```

---

## 7. Factory Patterns with Object Expressions

```fsharp
// Factory pattern ด้วย object expressions

type ISerializer =
    abstract member Serialize: obj -> string
    abstract member Deserialize<'T> : string -> 'T

type SerializerFactory() =
    static member CreateJson() =
        { new ISerializer with
            member this.Serialize(obj) =
                // Simple JSON-like serialization
                sprintf """{"type": "%s", "value": "%A"}""" (obj.GetType().Name) obj
            
            member this.Deserialize<'T>(json) =
                // Simplified - just return default for demo
                Unchecked.defaultof<'T> }
    
    static member CreateXml() =
        { new ISerializer with
            member this.Serialize(obj) =
                sprintf "<Value type=\"%s\">%A</Value>" (obj.GetType().Name) obj
            
            member this.Deserialize<'T>(xml) =
                Unchecked.defaultof<'T> }
    
    static member CreateCsv() =
        { new ISerializer with
            member this.Serialize(obj) =
                sprintf "%A" obj
            
            member this.Deserialize<'T>(csv) =
                Unchecked.defaultof<'T> }

// Plugin system using object expressions
type IPlugin =
    abstract member Name: string
    abstract member Version: string
    abstract member Initialize: unit -> unit
    abstract member Execute: string -> string
    abstract member Cleanup: unit -> unit

let createLoggingPlugin (logLevel: string) =
    { new IPlugin with
        member this.Name = "Logging"
        member this.Version = "1.0.0"
        member this.Initialize() = printfn "[%s Plugin] Initialized with level: %s" this.Name logLevel
        member this.Execute(input) =
            printfn "[%s] %s: %s" logLevel this.Name input
            sprintf "logged: %s" input
        member this.Cleanup() = printfn "[%s Plugin] Cleanup complete" this.Name }

let createCachingPlugin (maxSize: int) =
    let cache = System.Collections.Generic.Dictionary<string, string>()
    { new IPlugin with
        member this.Name = "Caching"
        member this.Version = "1.0.0"
        member this.Initialize() = 
            printfn "[Cache Plugin] Initialized with max size: %d" maxSize
        member this.Execute(input) =
            match cache.TryGetValue(input) with
            | true, cached ->
                printfn "[Cache] Hit: %s" input
                cached
            | _ ->
                let result = sprintf "processed: %s" input
                if cache.Count < maxSize then
                    cache.[input] <- result
                    printfn "[Cache] Miss: %s (cached)" input
                else
                    printfn "[Cache] Miss: %s (cache full)" input
                result
        member this.Cleanup() = 
            cache.Clear()
            printfn "[Cache Plugin] Cache cleared" }

// Plugin manager
type PluginManager() =
    let mutable plugins: IPlugin list = []
    
    member this.Register(plugin: IPlugin) =
        plugin.Initialize()
        plugins <- plugin :: plugins
    
    member this.Execute(input: string) =
        plugins |> List.rev |> List.fold (fun acc plugin ->
            plugin.Execute(acc)
        ) input
    
    member this.Shutdown() =
        for plugin in plugins do
            plugin.Cleanup()
        plugins <- []

let manager = PluginManager()
manager.Register(createLoggingPlugin "INFO")
manager.Register(createCachingPlugin 100)

printfn "\n=== Plugin System ==="
let result1 = manager.Execute("Hello World")
let result2 = manager.Execute("Hello World")  // Should hit cache
printfn "Result: %s" result2

manager.Shutdown()
```

---

## 8. Adapter Pattern

```fsharp
// Adapter pattern ด้วย object expressions

// Legacy interface ที่ไม่สามารถแก้ไขได้
type LegacyPrinter =
    abstract member PrintLine: string -> unit
    abstract member PrintHeader: string -> unit
    abstract member PrintFooter: string -> unit

// New interface ที่เราต้องการใช้
type IModernPrinter =
    abstract member Print: content: string -> unit
    abstract member PrintSection: title: string -> content: string -> unit
    abstract member Flush: unit -> unit

// Adapter ที่ convert LegacyPrinter เป็น IModernPrinter
let createModernPrinterAdapter (legacy: LegacyPrinter) =
    let buffer = System.Text.StringBuilder()
    
    { new IModernPrinter with
        member this.Print(content) =
            buffer.AppendLine(content) |> ignore
        
        member this.PrintSection(title)(content) =
            legacy.PrintHeader(title)
            for line in content.Split('\n') do
                legacy.PrintLine(line)
            legacy.PrintFooter("---")
        
        member this.Flush() =
            let lines = buffer.ToString().Split('\n')
            for line in lines do
                if not (System.String.IsNullOrEmpty(line)) then
                    legacy.PrintLine(line)
            buffer.Clear() |> ignore }

// Legacy printer implementation
let consoleLegacyPrinter = 
    { new LegacyPrinter with
        member this.PrintLine(text) = printfn "  %s" text
        member this.PrintHeader(title) = printfn "\n=== %s ===" title
        member this.PrintFooter(footer) = printfn "%s" footer }

// ใช้ adapter
let modernPrinter = createModernPrinterAdapter consoleLegacyPrinter

modernPrinter.Print("Line 1")
modernPrinter.Print("Line 2")
modernPrinter.Print("Line 3")
modernPrinter.Flush()

modernPrinter.PrintSection "Report" "Revenue: $1000\nCosts: $600\nProfit: $400"

// Third-party library adapter
type ThirdPartyLogger(tag: string) =
    member this.WriteLog(msg: string) =
        printfn "[3rdParty][%s] %s" tag msg

// Adapt third-party logger to our ILogger interface
let adaptThirdPartyLogger (thirdParty: ThirdPartyLogger) =
    { new ILogger with
        member this.Info(msg) = thirdParty.WriteLog(sprintf "INFO: %s" msg)
        member this.Error(msg) = thirdParty.WriteLog(sprintf "ERROR: %s" msg) }
    
and ILogger =
    abstract member Info: string -> unit
    abstract member Error: string -> unit

let thirdParty = ThirdPartyLogger("MyApp")
let adaptedLogger = adaptThirdPartyLogger thirdParty

adaptedLogger.Info "Application started"
adaptedLogger.Error "Something went wrong"
```

---

## 9. Lightweight Interface Implementations

```fsharp
// Object expressions สำหรับ lightweight implementations

// Comparer
let intDescending =
    { new System.Collections.Generic.IComparer<int> with
        member this.Compare(x, y) = compare y x }

let stringByLength =
    { new System.Collections.Generic.IComparer<string> with
        member this.Compare(x, y) = compare x.Length y.Length }

let numbers = [| 5; 2; 8; 1; 9; 3; 7 |]
System.Array.Sort(numbers, intDescending)
printfn "Sorted descending: %A" numbers

let words = [| "banana"; "apple"; "kiwi"; "cherry" |]
System.Array.Sort(words, stringByLength)
printfn "Sorted by length: %A" words

// EqualityComparer
let caseInsensitiveComparer =
    { new System.Collections.Generic.IEqualityComparer<string> with
        member this.Equals(x, y) = 
            System.String.Compare(x, y, System.StringComparison.OrdinalIgnoreCase) = 0
        member this.GetHashCode(s) = 
            s.ToLower().GetHashCode() }

let dict = System.Collections.Generic.Dictionary<string, int>(caseInsensitiveComparer)
dict.["Hello"] <- 1
dict.["World"] <- 2

printfn "dict['hello'] = %d" dict.["hello"]  // 1 (case insensitive)
printfn "dict['WORLD'] = %d" dict.["WORLD"]  // 2

// Event handler pattern
let createThrottledHandler (intervalMs: int) (handler: string -> unit) =
    let mutable lastFire = System.DateTime.MinValue
    { new System.EventHandler<string> with
        member this.Invoke(sender, e) =
            let now = System.DateTime.Now
            if (now - lastFire).TotalMilliseconds >= float intervalMs then
                lastFire <- now
                handler e }

// Callback adapter
type ICallback =
    abstract member OnSuccess: result: string -> unit
    abstract member OnError: error: string -> unit
    abstract member OnProgress: percent: int -> unit

let createCallback (onSuccess: string -> unit) (onError: string -> unit) =
    { new ICallback with
        member this.OnSuccess(result) = onSuccess result
        member this.OnError(error) = onError error
        member this.OnProgress(percent) = 
            printf "\rProgress: %d%%    " percent }

let callback = createCallback
    (fun result -> printfn "\nSuccess: %s" result)
    (fun error -> printfn "\nError: %s" error)

// Simulate progress
for i in [25; 50; 75; 100] do
    callback.OnProgress(i)
    System.Threading.Thread.Sleep(100)

callback.OnSuccess("Operation completed!")
```

---

## 10. Comparison with C# Anonymous Objects

```fsharp
// F# object expressions vs C# anonymous objects

// C# anonymous object (for reference):
// var person = new { Name = "John", Age = 30 };
// (read-only, no methods, used mainly for projections)

// F# object expressions can implement full interfaces:
type IProcessor =
    abstract member Process: string -> string
    abstract member GetStats: unit -> {| Processed: int; Failed: int |}

// F# creates a full interface implementation inline
let createProcessor (transform: string -> string) =
    let mutable processed = 0
    let mutable failed = 0
    
    { new IProcessor with
        member this.Process(input) =
            try
                let result = transform input
                processed <- processed + 1
                result
            with ex ->
                failed <- failed + 1
                sprintf "Error: %s" ex.Message
        
        member this.GetStats() =
            {| Processed = processed; Failed = failed |} }

let upperProcessor = createProcessor (fun s -> s.ToUpper())
let reverseProcessor = createProcessor (fun s -> 
    s |> Seq.rev |> System.String.Concat)

let inputs = ["hello"; "world"; "F#"]
for input in inputs do
    printfn "'%s' -> '%s'" input (upperProcessor.Process(input))

let stats = upperProcessor.GetStats()
printfn "Stats: %d processed, %d failed" stats.Processed stats.Failed

// F# record types for simple data grouping (more F# idiomatic)
type PersonData = { Name: string; Age: int }
let person = { Name = "John"; Age = 30 }
printfn "Person: %s, %d" person.Name person.Age

// F# anonymous records (similar to C# anonymous objects)
let anonPerson = {| Name = "Jane"; Age = 25 |}
printfn "Anon: %s, %d" anonPerson.Name anonPerson.Age
```

---

## 11. Advanced Object Expression Patterns

```fsharp
// Recursive object expressions
type ITree<'T> =
    abstract member Value: 'T
    abstract member Children: ITree<'T> list
    abstract member IsLeaf: bool
    abstract member Depth: int

let rec makeTree (value: 'T) (children: ITree<'T> list) =
    { new ITree<'T> with
        member this.Value = value
        member this.Children = children
        member this.IsLeaf = children.IsEmpty
        member this.Depth =
            if children.IsEmpty then 0
            else 1 + (children |> List.map (fun c -> c.Depth) |> List.max) }

let leaf v = makeTree v []
let node v children = makeTree v children

let tree = 
    node 1 [
        node 2 [leaf 4; leaf 5]
        node 3 [leaf 6]
    ]

printfn "Root: %d, Depth: %d" tree.Value tree.Depth
printfn "Is leaf: %b" tree.IsLeaf
printfn "Children count: %d" tree.Children.Length

// Decorator pattern
type ITextTransformer =
    abstract member Transform: string -> string

let baseTransformer =
    { new ITextTransformer with
        member this.Transform(text) = text }

let addPrefix (prefix: string) (inner: ITextTransformer) =
    { new ITextTransformer with
        member this.Transform(text) =
            prefix + inner.Transform(text) }

let addSuffix (suffix: string) (inner: ITextTransformer) =
    { new ITextTransformer with
        member this.Transform(text) =
            inner.Transform(text) + suffix }

let toUpper (inner: ITextTransformer) =
    { new ITextTransformer with
        member this.Transform(text) =
            inner.Transform(text).ToUpper() }

// Chain decorators
let transformer =
    baseTransformer
    |> addPrefix "["
    |> addSuffix "]"
    |> toUpper

printfn "%s" (transformer.Transform("hello world"))  // [HELLO WORLD]

// Strategy pattern
type ISortStrategy<'T> =
    abstract member Sort: 'T list -> 'T list

let bubbleSort<'T when 'T : comparison> () =
    { new ISortStrategy<'T> with
        member this.Sort(items) =
            let arr = Array.ofList items
            for i in 0 .. arr.Length - 2 do
                for j in 0 .. arr.Length - i - 2 do
                    if arr.[j] > arr.[j+1] then
                        let temp = arr.[j]
                        arr.[j] <- arr.[j+1]
                        arr.[j+1] <- temp
            Array.toList arr }

let insertionSort<'T when 'T : comparison> () =
    { new ISortStrategy<'T> with
        member this.Sort(items) =
            items |> List.sortWith compare }

type Sorter<'T when 'T : comparison>(strategy: ISortStrategy<'T>) =
    member val Strategy = strategy with get, set
    member this.Sort(items: 'T list) = this.Strategy.Sort(items)

let sorter = Sorter<int>(bubbleSort())
let sorted = sorter.Sort([5; 2; 8; 1; 9])
printfn "Bubble sorted: %A" sorted

sorter.Strategy <- insertionSort()
let sorted2 = sorter.Sort([5; 2; 8; 1; 9])
printfn "Insertion sorted: %A" sorted2
```

---

## 12. Practical Example: Event System

```fsharp
// Event system ด้วย object expressions
type IEventHandler<'T> =
    abstract member Handle: 'T -> unit
    abstract member CanHandle: 'T -> bool

type IEventBus =
    abstract member Subscribe<'T> : IEventHandler<'T> -> System.IDisposable
    abstract member Publish<'T> : 'T -> unit

type SimpleEventBus() =
    let handlers = System.Collections.Generic.Dictionary<System.Type, System.Collections.Generic.List<obj>>()
    
    let getHandlers (t: System.Type) =
        match handlers.TryGetValue(t) with
        | true, list -> list
        | _ ->
            let list = System.Collections.Generic.List<obj>()
            handlers.[t] <- list
            list
    
    interface IEventBus with
        member this.Subscribe<'T>(handler: IEventHandler<'T>) =
            let list = getHandlers typeof<'T>
            list.Add(handler)
            
            { new System.IDisposable with
                member this.Dispose() =
                    list.Remove(handler) |> ignore }
        
        member this.Publish<'T>(event: 'T) =
            let list = getHandlers typeof<'T>
            for handlerObj in list do
                let handler = handlerObj :?> IEventHandler<'T>
                if handler.CanHandle(event) then
                    handler.Handle(event)

// Domain events
type UserCreated = { UserId: int; Name: string; Email: string }
type OrderPlaced = { OrderId: int; UserId: int; Total: decimal }

// Event handlers ด้วย object expressions
let createEmailHandler () =
    { new IEventHandler<UserCreated> with
        member this.Handle(event) =
            printfn "[EmailHandler] Sending welcome email to %s" event.Email
        member this.CanHandle(_) = true }

let createAuditHandler () =
    { new IEventHandler<UserCreated> with
        member this.Handle(event) =
            printfn "[AuditHandler] User %d created: %s" event.UserId event.Name
        member this.CanHandle(_) = true }

let createOrderEmailHandler () =
    { new IEventHandler<OrderPlaced> with
        member this.Handle(event) =
            printfn "[EmailHandler] Sending order confirmation #%d (Total: %.2M)" event.OrderId event.Total
        member this.CanHandle(_) = true }

// ใช้ event bus
let bus = SimpleEventBus() :> IEventBus

use sub1 = bus.Subscribe(createEmailHandler())
use sub2 = bus.Subscribe(createAuditHandler())
use sub3 = bus.Subscribe(createOrderEmailHandler())

printfn "=== Event Bus Demo ==="
bus.Publish({ UserId = 1; Name = "สมชาย"; Email = "somchai@example.com" })
bus.Publish({ OrderId = 101; UserId = 1; Total = 1500.50M })
bus.Publish({ UserId = 2; Name = "สมหญิง"; Email = "somying@example.com" })
```

---

## สรุป (Summary)

```fsharp
printfn "=== Object Expression Summary ==="
printfn ""
printfn "Basic syntax:"
printfn "{ new ISomeInterface with"
printfn "    member this.Method(args) = ..." 
printfn "    member this.Property = ... }"
printfn ""
printfn "With abstract class:"
printfn "{ new AbstractBase() with"
printfn "    override this.AbstractMethod() = ..."
printfn "    override this.AbstractProp = ... }"
printfn ""
printfn "Key benefits:"
printfn "1. No need to define a named class"
printfn "2. Closures work naturally"
printfn "3. Perfect for one-off implementations"
printfn "4. Excellent for mocking in tests"
printfn "5. Lightweight adapters and decorators"
printfn ""
printfn "Common use cases:"
printfn "- IDisposable for RAII"
printfn "- IComparer for sorting"
printfn "- Test doubles (mocks/stubs)"
printfn "- Adapter pattern"
printfn "- Strategy pattern"
printfn "- Factory pattern"
```

---

## บทสรุป

Object expressions ใน F# เป็นฟีเจอร์ที่ทรงพลังที่:
1. ให้เราสร้าง objects ที่ implement interfaces ได้แบบ inline
2. รองรับ closures ทำให้เก็บ state ได้สะดวก
3. ลดความจำเป็นในการสร้าง named classes
4. เหมาะสำหรับ testing, adapters, และ one-off implementations
5. ทำให้ code กระชับและอ่านง่ายกว่าการสร้าง class ใหม่
