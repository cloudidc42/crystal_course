# Part 35 - แอตทริบิวต์และการสะท้อน (Attributes and Reflection)

## บทนำ (Introduction)

Attributes เป็น metadata ที่เพิ่มเข้าไปใน code เพื่อให้ข้อมูลเพิ่มเติมแก่ compiler, runtime, หรือ framework ส่วน Reflection ช่วยให้เราตรวจสอบ type information และเรียกใช้ code แบบ dynamic ในขณะ runtime

---

## 1. Common .NET Attributes

```fsharp
// Common .NET attributes ที่ใช้บ่อยใน F#

// [<Obsolete>] - เตือนว่า API นี้ล้าสมัยแล้ว
[<Obsolete("Use NewFunction instead")>]
let oldFunction x = x + 1

let newFunction x = x + 1

// เรียกใช้ oldFunction จะได้ warning
// let result = oldFunction 5  // Warning: obsolete

// [<Obsolete("msg", true)>] - ทำให้ error ไม่ใช่แค่ warning
[<Obsolete("This method is removed, use alternative", true)>]
let removedFunction x = x

// [<System.Obsolete>]  // เทียบเท่า
```

---

## 2. [<Obsolete>] Attribute

```fsharp
module LegacyApi =
    [<Obsolete("Use calculateTax2024 instead. Will be removed in v3.0")>]
    let calculateTax (income: decimal) =
        income * 0.2M
    
    let calculateTax2024 (income: decimal) (brackets: (decimal * decimal) list) =
        let mutable tax = 0M
        let mutable remaining = income
        
        for (limit, rate) in brackets do
            if remaining > 0M then
                let taxable = min remaining limit
                tax <- tax + taxable * rate
                remaining <- remaining - taxable
        
        tax

module Api =
    // [<Obsolete>] กับ parameter
    type [<Obsolete("Use PersonRecord instead")>] OldPerson(name: string, age: int) =
        member this.Name = name
        member this.Age = age

type PersonRecord = { Name: string; Age: int }

// การใช้ Obsolete ใน module
[<Obsolete("Use V2 module instead")>]
module V1 =
    let process x = x * 2

module V2 =
    let process x = x * 2 + 1

// ตรวจสอบ ObsoleteAttribute ด้วย reflection
let checkObsolete (t: System.Type) =
    let obsoleteAttr = t.GetCustomAttributes(typeof<System.ObsoleteAttribute>, false)
    if obsoleteAttr.Length > 0 then
        let attr = obsoleteAttr.[0] :?> System.ObsoleteAttribute
        printfn "Type '%s' is obsolete: %s (Error: %b)" t.Name attr.Message attr.IsError
    else
        printfn "Type '%s' is not obsolete" t.Name
```

---

## 3. [<Serializable>]

```fsharp
open System.Runtime.Serialization
open System.IO
open System.Runtime.Serialization.Formatters.Binary

[<Serializable>]
type SerializablePoint(x: float, y: float) =
    member this.X = x
    member this.Y = y
    override this.ToString() = sprintf "(%.2f, %.2f)" x y

[<Serializable>]
type SerializableConfig = {
    Host: string
    Port: int
    MaxConnections: int
    Timeout: int
}

// ใช้ System.Text.Json สำหรับ modern serialization
open System.Text.Json

let config = {
    Host = "localhost"
    Port = 8080
    MaxConnections = 100
    Timeout = 30
}

let json = JsonSerializer.Serialize(config)
printfn "JSON: %s" json

let deserialized = JsonSerializer.Deserialize<SerializableConfig>(json)
printfn "Deserialized: %s:%d" deserialized.Host deserialized.Port

// DataContract serialization
[<DataContract>]
type Person = {
    [<DataMember(Name = "full_name")>]
    Name: string
    [<DataMember(Name = "age_years")>]
    Age: int
    [<DataMember(IsRequired = false)>]
    Email: string option
}

let person = { Name = "สมชาย"; Age = 30; Email = Some "somchai@example.com" }
printfn "Person: %A" person
```

---

## 4. [<StructLayout>]

```fsharp
open System.Runtime.InteropServices

// ควบคุม layout ของ struct ใน memory

[<Struct; StructLayout(LayoutKind.Sequential)>]
type SequentialStruct =
    val Field1: int
    val Field2: float
    val Field3: byte

[<Struct; StructLayout(LayoutKind.Explicit)>]
type ExplicitStruct =
    [<FieldOffset(0)>] val IntValue: int
    [<FieldOffset(0)>] val ByteValue1: byte  // Same position as IntValue
    [<FieldOffset(1)>] val ByteValue2: byte
    [<FieldOffset(2)>] val ByteValue3: byte
    [<FieldOffset(3)>] val ByteValue4: byte

// Point2D ที่ใช้ Sequential layout
[<Struct; StructLayout(LayoutKind.Sequential, Pack = 4)>]
type Point2D =
    val X: float32
    val Y: float32
    new(x, y) = { X = x; Y = y }

printfn "Size of SequentialStruct: %d" (Marshal.SizeOf<SequentialStruct>())
printfn "Size of Point2D: %d" (Marshal.SizeOf<Point2D>())

let p = Point2D(3.0f, 4.0f)
printfn "Point: (%.1f, %.1f)" p.X p.Y
```

---

## 5. F#-Specific Attributes: [<AutoOpen>]

```fsharp
// [<AutoOpen>] - เปิด module อัตโนมัติเมื่อ parent module/namespace ถูกเปิด

// ใน file อื่น:
// namespace MyLibrary
//
// [<AutoOpen>]
// module Helpers =
//     let helper1 x = x + 1
//     let helper2 x = x * 2
//
// module Main =
//     // helper1 และ helper2 พร้อมใช้โดยอัตโนมัติ (ไม่ต้อง open Helpers)
//     let result = helper1 5

// ตัวอย่างในไฟล์เดียวกัน
module MyLibrary =
    [<AutoOpen>]
    module Internal =
        let formatValue v = sprintf "[%A]" v
        let wrapValue v = (v, System.DateTime.Now)
    
    // formatValue และ wrapValue พร้อมใช้ในทุก module ใน MyLibrary
    module Public =
        let process x = 
            let formatted = formatValue x  // ใช้จาก AutoOpen module
            printfn "Processing: %s" formatted
            fst (wrapValue x)

MyLibrary.Public.process 42
MyLibrary.Public.process "hello"
```

---

## 6. [<RequireQualifiedAccess>]

```fsharp
// [<RequireQualifiedAccess>] - บังคับให้ระบุ module name เสมอ

[<RequireQualifiedAccess>]
module Color =
    type RGB = { R: int; G: int; B: int }
    
    let red = { R = 255; G = 0; B = 0 }
    let green = { R = 0; G = 255; B = 0 }
    let blue = { R = 0; G = 0; B = 255 }
    
    let mix (c1: RGB) (c2: RGB) =
        { R = (c1.R + c2.R) / 2
          G = (c1.G + c2.G) / 2
          B = (c1.B + c2.B) / 2 }
    
    let toHex (c: RGB) =
        sprintf "#%02X%02X%02X" c.R c.G c.B

// ต้องใช้ Color.xxx เสมอ ไม่สามารถ open Color แล้วใช้ชื่อสั้นๆ ได้
let orange = Color.mix Color.red Color.green
printfn "Orange: %s" (Color.toHex orange)

// ตัวอย่างกับ DU
[<RequireQualifiedAccess>]
type Direction =
    | North
    | South
    | East
    | West

// ต้องเขียน Direction.North ไม่ใช่แค่ North
let move direction steps =
    match direction with
    | Direction.North -> printfn "Moving north %d steps" steps
    | Direction.South -> printfn "Moving south %d steps" steps
    | Direction.East -> printfn "Moving east %d steps" steps
    | Direction.West -> printfn "Moving west %d steps" steps

move Direction.North 5
move Direction.East 3
```

---

## 7. [<Struct>]

```fsharp
// [<Struct>] - ทำให้ discriminated union หรือ record เป็น value type

// Struct Record
[<Struct>]
type Point3D = {
    X: float
    Y: float
    Z: float
}

// Struct Discriminated Union
[<Struct>]
type Result2<'T, 'E> =
    | Ok2 of value: 'T
    | Error2 of error: 'E

// Struct Union
[<Struct>]
type Option2<'T> =
    | Some2 of item: 'T
    | None2

// เปรียบเทียบขนาด
let normalPoint = { X = 1.0; Y = 2.0; Z = 3.0 }
printfn "Point3D is struct: %b" (typeof<Point3D>.IsValueType)

// Performance test
let mutable sum = 0.0
let count = 1_000_000

// Using struct points (stack allocated, better cache performance)
let sw = System.Diagnostics.Stopwatch.StartNew()
for i in 1..count do
    let p = { X = float i; Y = float i; Z = float i }
    sum <- sum + p.X
sw.Stop()
printfn "Struct Point3D time: %dms, sum=%.0f" sw.ElapsedMilliseconds sum

// Struct in collections
let points = Array.init 100 (fun i -> 
    { X = float i; Y = float i * 2.0; Z = float i * 3.0 })

let totalDistance = 
    points 
    |> Array.sumBy (fun p -> 
        sqrt (p.X * p.X + p.Y * p.Y + p.Z * p.Z))

printfn "Total distance: %.2f" totalDistance
```

---

## 8. [<Measure>]

```fsharp
// [<Measure>] - Units of measure สำหรับ type safety

[<Measure>] type kg
[<Measure>] type m
[<Measure>] type s
[<Measure>] type km
[<Measure>] type hr
[<Measure>] type celsius
[<Measure>] type fahrenheit

// ฟังก์ชันที่มี unit safety
let weight1 = 70.0<kg>
let distance1 = 100.0<m>
let time1 = 10.0<s>

let speed = distance1 / time1  // ชนิดข้อมูล: float<m/s>
printfn "Speed: %.1f m/s" speed

// Conversion functions
let kmToM (km: float<km>) : float<m> = km * 1000.0<m/km>
let mToKm (m: float<m>) : float<km> = m / 1000.0<m/km>
let hrToS (hr: float<hr>) : float<s> = hr * 3600.0<s/hr>

let kmh = 90.0<km/hr>
let ms = kmToM (1.0<km>) / hrToS (1.0<hr>)  // m/s per km/hr
printfn "%.1f km/h = %.2f m/s" (float kmh) (float kmh * float ms)

// Temperature conversion
let celsiusToFahrenheit (c: float<celsius>) : float<fahrenheit> =
    (float c * 9.0 / 5.0 + 32.0) * 1.0<fahrenheit>

let body = 37.0<celsius>
let bodyF = celsiusToFahrenheit body
printfn "Body temperature: %.1f°C = %.1f°F" (float body) (float bodyF)

// Physics calculations
[<Measure>] type N   // Newton
[<Measure>] type J   // Joule

let mass = 5.0<kg>
let gravity = 9.81<m/s^2>
let force = mass * gravity  // N = kg * m/s^2

let height = 10.0<m>
let potentialEnergy = mass * gravity * height  // J = kg * (m/s^2) * m

printfn "Force: %.2f N" (float force)
printfn "Potential Energy: %.2f J" (float potentialEnergy)
```

---

## 9. [<Literal>]

```fsharp
// [<Literal>] - สร้าง compile-time constants

[<Literal>]
let MaxRetries = 3

[<Literal>]
let DefaultTimeout = 30_000  // 30 seconds in ms

[<Literal>]
let ApiVersion = "v2.1"

[<Literal>]
let DefaultHost = "localhost"

[<Literal>]
let Pi = 3.14159265358979323846

// ใช้ใน match expression
let describeRetries (attempts: int) =
    match attempts with
    | 0 -> "No attempts yet"
    | MaxRetries -> "Maximum retries reached"  // ใช้ literal ใน pattern matching
    | n when n < MaxRetries -> sprintf "%d/%d attempts" n MaxRetries
    | _ -> "Exceeded maximum retries"

printfn "%s" (describeRetries 0)
printfn "%s" (describeRetries 2)
printfn "%s" (describeRetries 3)
printfn "%s" (describeRetries 5)

// Literal ใน attributes
[<Literal>]
let TableName = "Users"

// Database query ที่ใช้ literal
let buildQuery (tableName: string) condition =
    sprintf "SELECT * FROM %s WHERE %s" tableName condition

// Literal สำหรับ regex patterns
[<Literal>]
let EmailPattern = @"^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$"

[<Literal>]
let PhonePattern = @"^\d{3}-\d{3}-\d{4}$"

let validateEmail email =
    System.Text.RegularExpressions.Regex.IsMatch(email, EmailPattern)

let validatePhone phone =
    System.Text.RegularExpressions.Regex.IsMatch(phone, PhonePattern)

printfn "Email valid: %b" (validateEmail "user@example.com")
printfn "Phone valid: %b" (validatePhone "123-456-7890")
```

---

## 10. [<ThreadStatic>]

```fsharp
// [<ThreadStatic>] - ทำให้ static field มีค่าแยกกันในแต่ละ thread

// Note: [<ThreadStatic>] ต้องใช้กับ static field เท่านั้น
[<ThreadStatic; DefaultValue>]
val mutable private _threadLocalValue: int

// F# ThreadLocal alternative (more idiomatic)
let threadLocal = new System.Threading.ThreadLocal<int>(fun () -> 0)

// ทดสอบ ThreadLocal
let runInThread (threadId: int) =
    async {
        threadLocal.Value <- threadId * 100
        do! Async.Sleep(100)  // simulate work
        printfn "Thread %d value: %d" threadId threadLocal.Value
    }

let tasks = [1..5] |> List.map runInThread
Async.RunSynchronously (Async.Parallel tasks |> Async.Ignore)

// Request context ใน web application
type RequestContext = {
    RequestId: string
    UserId: int option
    StartTime: System.DateTime
}

let currentRequest = new System.Threading.ThreadLocal<RequestContext option>(fun () -> None)

let processRequest (requestId: string) (userId: int option) =
    currentRequest.Value <- Some {
        RequestId = requestId
        UserId = userId
        StartTime = System.DateTime.Now
    }
    
    printfn "Processing request: %s" requestId
    
    // Access context anywhere in call stack
    match currentRequest.Value with
    | Some ctx -> printfn "  User: %A, Started: %s" ctx.UserId (ctx.StartTime.ToString("HH:mm:ss"))
    | None -> printfn "  No context"

processRequest "REQ-001" (Some 42)
processRequest "REQ-002" None
```

---

## 11. Custom Attribute Definition

```fsharp
// สร้าง custom attribute

// ValidateAttribute สำหรับ validation
[<System.AttributeUsage(System.AttributeTargets.Property, AllowMultiple = true)>]
type ValidateAttribute(validationType: string) =
    inherit System.Attribute()
    
    let mutable errorMessage = ""
    let mutable parameters: obj array = [||]
    
    member this.ValidationType = validationType
    member this.ErrorMessage
        with get() = errorMessage
        and set(v) = errorMessage <- v
    member this.Parameters
        with get() = parameters
        and set(v) = parameters <- v

// RouteAttribute สำหรับ routing
[<System.AttributeUsage(System.AttributeTargets.Method)>]
type RouteAttribute(path: string, method: string) =
    inherit System.Attribute()
    member this.Path = path
    member this.Method = method

[<System.AttributeUsage(System.AttributeTargets.Method)>]
type GetAttribute(path: string) =
    inherit RouteAttribute(path, "GET")

[<System.AttributeUsage(System.AttributeTargets.Method)>]
type PostAttribute(path: string) =
    inherit RouteAttribute(path, "POST")

// AuditAttribute สำหรับ audit trail
[<System.AttributeUsage(System.AttributeTargets.Method)>]
type AuditAttribute(action: string) =
    inherit System.Attribute()
    member this.Action = action
    member val LogRequest = true with get, set
    member val LogResponse = false with get, set

// Class ที่ใช้ custom attributes
type UserController() =
    [<Get("/users")>]
    [<Audit("GetAllUsers", LogRequest = false)>]
    member this.GetAll() = "All users"
    
    [<Get("/users/{id}")>]
    [<Audit("GetUser")>]
    member this.GetById(id: int) = sprintf "User %d" id
    
    [<Post("/users")>]
    [<Audit("CreateUser", LogRequest = true, LogResponse = true)>]
    member this.Create(name: string) = sprintf "Created user: %s" name

// ดึง route information ด้วย reflection
let discoverRoutes (controllerType: System.Type) =
    controllerType.GetMethods()
    |> Array.choose (fun method ->
        let routeAttr = method.GetCustomAttributes(typeof<RouteAttribute>, false)
        if routeAttr.Length > 0 then
            let attr = routeAttr.[0] :?> RouteAttribute
            Some {| Method = attr.Method; Path = attr.Path; Action = method.Name |}
        else
            None
    )

let routes = discoverRoutes typeof<UserController>
printfn "Discovered routes:"
for route in routes do
    printfn "  %s %s -> %s" route.Method route.Path route.Action
```

---

## 12. Reflection: Type.GetType()

```fsharp
// Reflection basics
open System.Reflection

// ดึง Type information
let getTypeInfo (obj: obj) =
    let t = obj.GetType()
    printfn "Type: %s" t.FullName
    printfn "Namespace: %s" t.Namespace
    printfn "Is ValueType: %b" t.IsValueType
    printfn "Is Abstract: %b" t.IsAbstract
    printfn "Is Sealed: %b" t.IsSealed
    printfn "Base Type: %A" (Option.ofObj t.BaseType |> Option.map (fun t -> t.Name))
    
    let interfaces = t.GetInterfaces()
    printfn "Interfaces: %A" (interfaces |> Array.map (fun i -> i.Name))

// ตัวอย่าง
printfn "=== Type Info for int ==="
getTypeInfo (42 :> obj)

printfn "\n=== Type Info for string ==="
getTypeInfo ("hello" :> obj)

// สร้าง Type จาก string name
let getTypeByName (typeName: string) =
    match System.Type.GetType(typeName) with
    | null -> 
        // ลอง assembly-qualified name
        System.AppDomain.CurrentDomain.GetAssemblies()
        |> Array.tryPick (fun asm -> 
            match asm.GetType(typeName) with
            | null -> None
            | t -> Some t)
    | t -> Some t

let intType = getTypeByName "System.Int32"
let stringType = getTypeByName "System.String"

printfn "\nInt32 methods: %d" (intType.Value.GetMethods().Length)
printfn "String methods: %d" (stringType.Value.GetMethods().Length)
```

---

## 13. Getting Custom Attributes via Reflection

```fsharp
// ดึง custom attributes ด้วย reflection

type CategoryAttribute(name: string) =
    inherit System.Attribute()
    member this.Name = name

type PriorityAttribute(level: int) =
    inherit System.Attribute()
    member this.Level = level

type [<Category("Math")>][<Priority(1)>] MathFunctions() =
    [<Category("Basic")>]
    member this.Add(a: int, b: int) = a + b
    
    [<Category("Advanced")>]
    [<Priority(5)>]
    member this.Factorial(n: int) =
        if n <= 1 then 1
        else n * this.Factorial(n - 1)

// ดึง attributes จาก type
let inspectType (t: System.Type) =
    printfn "Type: %s" t.Name
    
    let classAttrs = t.GetCustomAttributes(false)
    printfn "  Class attributes:"
    for attr in classAttrs do
        match attr with
        | :? CategoryAttribute as cat -> printfn "    Category: %s" cat.Name
        | :? PriorityAttribute as pri -> printfn "    Priority: %d" pri.Level
        | _ -> ()
    
    printfn "  Methods:"
    for method in t.GetMethods(BindingFlags.Instance ||| BindingFlags.Public ||| BindingFlags.DeclaredOnly) do
        let methodAttrs = method.GetCustomAttributes(false)
        if methodAttrs.Length > 0 then
            printf "    %s: " method.Name
            for attr in methodAttrs do
                match attr with
                | :? CategoryAttribute as cat -> printf "[Cat:%s] " cat.Name
                | :? PriorityAttribute as pri -> printf "[Pri:%d] " pri.Level
                | _ -> ()
            printfn ""

inspectType typeof<MathFunctions>
```

---

## 14. Dynamic Invocation

```fsharp
// Dynamic method invocation ด้วย reflection

type Calculator() =
    member this.Add(a: int, b: int) = a + b
    member this.Subtract(a: int, b: int) = a - b
    member this.Multiply(a: int, b: int) = a * b
    member this.Divide(a: int, b: int) = 
        if b = 0 then failwith "Division by zero"
        else a / b

// Dynamic invocation
let invokeMethod (obj: obj) (methodName: string) (args: obj array) =
    let t = obj.GetType()
    let method = t.GetMethod(methodName)
    if method = null then
        failwith (sprintf "Method '%s' not found" methodName)
    method.Invoke(obj, args)

let calc = Calculator()
let operations = [
    ("Add", [| box 10; box 5 |])
    ("Subtract", [| box 10; box 5 |])
    ("Multiply", [| box 4; box 7 |])
    ("Divide", [| box 20; box 4 |])
]

printfn "Dynamic calculation:"
for (methodName, args) in operations do
    let result = invokeMethod calc methodName args
    printfn "  %s(%A, %A) = %A" methodName args.[0] args.[1] result

// Plugin system ด้วย reflection
type IPlugin =
    abstract member Name: string
    abstract member Execute: string -> string

let loadPluginFromType (typeName: string) (assembly: Assembly) =
    let t = assembly.GetType(typeName)
    if t = null then None
    elif not (typeof<IPlugin>.IsAssignableFrom(t)) then None
    else
        try
            Some (System.Activator.CreateInstance(t) :?> IPlugin)
        with ex ->
            printfn "Failed to create plugin '%s': %s" typeName ex.Message
            None

// Generic factory ด้วย reflection
let createInstance<'T> (typeName: string) (args: obj array) =
    let assembly = typeof<'T>.Assembly
    let t = assembly.GetType(typeName)
    System.Activator.CreateInstance(t, args) :?> 'T

// สร้าง object แบบ dynamic
type AnimalFactory() =
    static member Create(animalType: string, name: string) =
        match animalType.ToLower() with
        | "dog" -> printfn "Creating dog: %s" name; {| Type = "Dog"; Name = name; Sound = "Woof" |}
        | "cat" -> printfn "Creating cat: %s" name; {| Type = "Cat"; Name = name; Sound = "Meow" |}
        | _ -> failwith (sprintf "Unknown animal type: %s" animalType)

let animals = ["Dog", "Rex"; "Cat", "Whiskers"; "Dog", "Buddy"]
for (animalType, name) in animals do
    let animal = AnimalFactory.Create(animalType, name)
    printfn "%s: %s" animal.Name animal.Sound
```

---

## 15. Performance Considerations

```fsharp
// Performance considerations สำหรับ Reflection

open System.Diagnostics

// Reflection มี overhead - ควร cache results
let methodCache = System.Collections.Generic.Dictionary<string, System.Reflection.MethodInfo>()

let getCachedMethod (t: System.Type) (methodName: string) =
    let key = sprintf "%s.%s" t.FullName methodName
    match methodCache.TryGetValue(key) with
    | true, method -> method
    | _ ->
        let method = t.GetMethod(methodName)
        if method <> null then
            methodCache.[key] <- method
        method

// Benchmark: Direct call vs Reflection
type MathOps() =
    member this.Double(x: int) = x * 2

let iterations = 100_000
let mathObj = MathOps()

// Direct call
let sw = Stopwatch.StartNew()
let mutable sum1 = 0
for i in 1..iterations do
    sum1 <- sum1 + mathObj.Double(i)
sw.Stop()
printfn "Direct call: %dms (sum=%d)" sw.ElapsedMilliseconds sum1

// Reflection call (uncached)
let doubleMethod = typeof<MathOps>.GetMethod("Double")
let sw2 = Stopwatch.StartNew()
let mutable sum2 = 0
for i in 1..iterations do
    sum2 <- sum2 + (doubleMethod.Invoke(mathObj, [| box i |]) :?> int)
sw2.Stop()
printfn "Reflection call: %dms (sum=%d)" sw2.ElapsedMilliseconds sum2

// Using delegates for better performance
let delegateMethod = 
    System.Delegate.CreateDelegate(typeof<System.Func<int, int>>, mathObj, doubleMethod) :?> System.Func<int, int>

let sw3 = Stopwatch.StartNew()
let mutable sum3 = 0
for i in 1..iterations do
    sum3 <- sum3 + delegateMethod.Invoke(i)
sw3.Stop()
printfn "Delegate call: %dms (sum=%d)" sw3.ElapsedMilliseconds sum3

// Expression tree compilation for fast reflection
open System.Linq.Expressions

let compileGetter<'T, 'R> (propertyName: string) =
    let param = Expression.Parameter(typeof<'T>, "obj")
    let property = Expression.Property(param, propertyName)
    let lambda = Expression.Lambda<System.Func<'T, 'R>>(property, param)
    lambda.Compile()

type DataRecord = { Name: string; Age: int; Score: float }

let nameGetter = compileGetter<DataRecord, string> "Name"
let ageGetter = compileGetter<DataRecord, int> "Age"

let record = { Name = "Alice"; Age = 25; Score = 95.5 }
printfn "\nCompiled getter:"
printfn "Name: %s" (nameGetter.Invoke(record))
printfn "Age: %d" (ageGetter.Invoke(record))
```

---

## 16. Reflection for Framework Building

```fsharp
// ตัวอย่าง mini DI container ที่ใช้ reflection

type IService = interface end

type IDependency = interface end

type ConcreteService(dep: IDependency) =
    interface IService
    member this.DoWork() = 
        printfn "Service doing work with %s" (dep.GetType().Name)

type ConcreteDependency() =
    interface IDependency

// Simple DI Container
type Container() =
    let registrations = System.Collections.Generic.Dictionary<System.Type, System.Type>()
    let singletons = System.Collections.Generic.Dictionary<System.Type, obj>()
    
    member this.Register<'TInterface, 'TImplementation when 'TImplementation :> 'TInterface>() =
        registrations.[typeof<'TInterface>] <- typeof<'TImplementation>
    
    member this.RegisterSingleton<'TInterface, 'TImplementation when 'TImplementation :> 'TInterface>() =
        registrations.[typeof<'TInterface>] <- typeof<'TImplementation>
        // Will be created lazily
    
    member this.Resolve<'T>() : 'T =
        this.Resolve(typeof<'T>) :?> 'T
    
    member this.Resolve(t: System.Type) : obj =
        // Check for singleton
        match singletons.TryGetValue(t) with
        | true, instance -> instance
        | _ ->
            match registrations.TryGetValue(t) with
            | true, implType ->
                // Get constructor with most parameters
                let ctors = implType.GetConstructors()
                let ctor = ctors |> Array.maxBy (fun c -> c.GetParameters().Length)
                
                // Resolve all parameters recursively
                let args = ctor.GetParameters() |> Array.map (fun p -> this.Resolve(p.ParameterType))
                
                let instance = ctor.Invoke(args)
                instance
            | _ ->
                failwith (sprintf "Type %s not registered" t.FullName)

// ทดสอบ DI container
let container = Container()
container.Register<IDependency, ConcreteDependency>()
container.Register<IService, ConcreteService>()

let service = container.Resolve<IService>() :?> ConcreteService
service.DoWork()

printfn "\nDI Container test complete"
```

---

## สรุป (Summary)

```fsharp
printfn "=== Attributes & Reflection Summary ==="
printfn ""
printfn ".NET Attributes:"
printfn "  [<Obsolete>] - mark deprecated code"
printfn "  [<Serializable>] - allow serialization"
printfn "  [<StructLayout>] - control memory layout"
printfn ""
printfn "F# Attributes:"
printfn "  [<AutoOpen>] - auto-open module"
printfn "  [<RequireQualifiedAccess>] - force module prefix"
printfn "  [<Struct>] - make type a value type"
printfn "  [<Measure>] - units of measure"
printfn "  [<Literal>] - compile-time constant"
printfn "  [<ThreadStatic>] - thread-local static"
printfn ""
printfn "Reflection:"
printfn "  Type.GetType() - get type info"
printfn "  GetCustomAttributes() - get attributes"
printfn "  GetMethods() / GetProperties() - inspect members"
printfn "  MethodInfo.Invoke() - dynamic invocation"
printfn ""
printfn "Performance tips:"
printfn "  - Cache reflection results"
printfn "  - Use compiled delegates"
printfn "  - Use expression trees for hot paths"
```

---

## บทสรุป

Attributes และ Reflection ใน F# ช่วยให้:
1. **Attributes** เพิ่ม metadata และควบคุม behavior ของ code
2. **F# specific attributes** ให้ control เพิ่มเติมสำหรับ F# features
3. **Reflection** ให้ inspect และ manipulate types ใน runtime
4. **Dynamic invocation** สร้าง flexible frameworks
5. **Performance** ต้องระวัง overhead ของ reflection - use caching
