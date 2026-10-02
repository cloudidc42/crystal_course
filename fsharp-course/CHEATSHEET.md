# F# Cheat Sheet — Quick Reference
# F# ชีทสรุป — อ้างอิงด่วน

> อ้างอิงด่วนสำหรับ syntax, patterns, และ idioms ของ F#
>
> Quick reference for F# syntax, patterns, and idioms

---

## สารบัญ (Table of Contents)

1. [Basic Syntax](#1-basic-syntax)
2. [Type Definitions](#2-type-definitions)
3. [Pattern Matching](#3-pattern-matching)
4. [Collections](#4-collections)
5. [Async/Task Patterns](#5-asynctask-patterns)
6. [Operators](#6-operators)
7. [Computation Expressions](#7-computation-expressions)
8. [Modules and Namespaces](#8-modules-and-namespaces)
9. [Standard Library Functions](#9-standard-library-functions)
10. [Object-Oriented Features](#10-object-oriented-features)
11. [Error Handling](#11-error-handling)
12. [Common NuGet Packages](#12-common-nuget-packages)
13. [F# vs C# Syntax Comparison](#13-f-vs-c-syntax-comparison)

---

## 1. Basic Syntax

### Let Bindings

```fsharp
// Immutable binding (ค่าไม่เปลี่ยน)
let x = 42
let name = "Alice"
let pi = 3.14159
let isActive = true

// Mutable binding (ค่าเปลี่ยนได้)
let mutable count = 0
count <- count + 1

// Type annotation (explicit)
let length: int = 10
let message: string = "Hello"
let values: int list = [1; 2; 3]

// Unit value (เหมือน void)
let doNothing: unit = ()
```

### Functions

```fsharp
// Function definition
let add x y = x + y
let square x = x * x
let isEven n = n % 2 = 0

// Function with type annotations
let multiply (x: int) (y: int): int = x * y

// Lambda (anonymous function)
let double = fun x -> x * 2
let add3 = fun x y z -> x + y + z

// Recursive function (ต้องใช้ rec keyword)
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

// Mutually recursive functions
let rec isOdd n = if n = 0 then false else isEven (n - 1)
and isEven n = if n = 0 then true else isOdd (n - 1)

// Function with unit parameter
let greet () = printfn "Hello!"
greet ()    // ต้องส่ง () เสมอ

// Nested functions (local functions)
let outerFunction x =
    let innerHelper y = y * 2    // เฉพาะใน outerFunction เท่านั้น
    innerHelper x + innerHelper (x + 1)
```

### Conditionals

```fsharp
// if/then/else เป็น expression
let abs x =
    if x >= 0 then x
    else -x

// if/then/elif/else
let classify n =
    if n > 0 then "positive"
    elif n < 0 then "negative"
    else "zero"

// if/then ไม่มี else (ต้อง return unit)
let printIfPositive n =
    if n > 0 then
        printfn "%d is positive" n

// if/then/else แบบ inline
let result = if condition then valueA else valueB
```

### Loops

```fsharp
// for loop (imperative style)
for i in 1..10 do
    printfn "%d" i

// for loop แบบ step
for i in 0..2..20 do   // 0, 2, 4, ..., 20
    printfn "%d" i

// for loop แบบ ย้อนกลับ
for i in 10..-1..1 do  // 10, 9, ..., 1
    printfn "%d" i

// for in (iterate collection)
for item in ["a"; "b"; "c"] do
    printfn "%s" item

// while loop
let mutable i = 0
while i < 10 do
    printfn "%d" i
    i <- i + 1
```

### String Operations

```fsharp
// String creation
let s1 = "Hello, World!"
let s2 = "Line 1\nLine 2"     // Escaped string
let s3 = @"C:\Users\Alice"    // Verbatim string
let s4 = """Multi
line
string"""                      // Triple-quoted string

// String interpolation (F# 5+)
let name = "Alice"
let age = 30
let msg = $"Name: {name}, Age: {age}"
let pi = $"Pi = {System.Math.PI:F4}"    // Format specifier

// String functions
let upper = "hello".ToUpper()           // "HELLO"
let lower = "HELLO".ToLower()           // "hello"
let trimmed = "  hello  ".Trim()        // "hello"
let split = "a,b,c".Split(',')         // [|"a"; "b"; "c"|]
let joined = String.concat ", " ["a"; "b"; "c"]  // "a, b, c"
let contains = "hello".Contains("ell") // true
let starts = "hello".StartsWith("hel") // true
let replaced = "hello".Replace("l", "r") // "herro"
let len = "hello".Length               // 5
let sub = "hello".[1..3]              // "ell"
let char = "hello".[0]               // 'h'

// sprintf (type-safe string formatting)
let formatted = sprintf "Value: %d, Float: %.2f" 42 3.14
// "Value: 42, Float: 3.14"

// printf/printfn
printfn "Hello, %s! You are %d years old." name age
printf "No newline "
eprintfn "Error: %s" "something went wrong"  // stderr
```

### Numeric Types

```fsharp
// Integer types
let i8: int8 = 127y              // sbyte
let u8: uint8 = 255uy            // byte
let i16: int16 = 32767s          // short
let u16: uint16 = 65535us        // ushort
let i32: int32 = 2147483647      // int (default)
let i32b: int = 2147483647
let u32: uint32 = 4294967295u    // uint
let i64: int64 = 9223372036854775807L   // long
let u64: uint64 = 18446744073709551615UL

// Float types
let f32: float32 = 3.14f         // float (32-bit)
let f64: float = 3.14159265      // double (default)
let dec: decimal = 3.14159m      // decimal

// Conversions
let intVal = int 3.7             // 3 (truncates)
let floatVal = float 42          // 42.0
let decVal = decimal 3.14        // 3.14M
let stringVal = string 42        // "42"

// Numeric operations
open System
let pi = Math.PI
let e = Math.E
let sqrt2 = sqrt 2.0
let power = pown 2 10            // 1024 (int power)
let powerf = 2.0 ** 10.0        // 1024.0 (float power)
let abs = abs -42                // 42
```

---

## 2. Type Definitions

### Aliases

```fsharp
// Type alias (transparent — same underlying type)
type Name = string
type Age = int
type Coordinate = float * float

// Single-case discriminated union (opaque wrapper)
type CustomerId = CustomerId of int
type Email = Email of string

// Unwrap single-case DU
let (CustomerId id) = CustomerId 42    // id = 42
let customerId = CustomerId 42
let (CustomerId rawId) = customerId
```

### Tuples

```fsharp
// Tuple creation
let pair = (1, "hello")
let triple = (1, 2.0, "three")

// Destructuring
let (a, b) = pair
let (x, y, z) = triple

// Access
let first = fst pair    // 1
let second = snd pair   // "hello"

// Ignore elements
let (_, second') = pair

// Struct tuples (เร็วกว่า reference tuples)
let structTuple = struct (1, "hello")
let struct (sa, sb) = structTuple
```

### Records

```fsharp
// Record definition
type Person = {
    Name: string
    Age: int
    Email: string option
}

// Record creation
let alice = { Name = "Alice"; Age = 30; Email = Some "alice@example.com" }

// Record access
let name = alice.Name    // "Alice"
let age = alice.Age      // 30

// Record update (สร้าง copy ใหม่)
let older = { alice with Age = 31 }
let withEmail = { alice with Email = None }

// Pattern matching
let describe { Name = n; Age = a } =
    printfn "%s is %d years old" n a

// Mutable record fields
type Counter = {
    mutable Count: int
    Label: string
}
let c = { Count = 0; Label = "hits" }
c.Count <- c.Count + 1

// Struct records (เร็วกว่าสำหรับ small records)
[<Struct>]
type Point = { X: float; Y: float }
```

### Discriminated Unions

```fsharp
// Simple DU
type Direction = North | South | East | West

// DU with data
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

// Nested DU
type Tree<'a> =
    | Leaf
    | Node of value: 'a * left: Tree<'a> * right: Tree<'a>

// Option type (built-in DU)
type Option<'a> = None | Some of 'a

// Result type (built-in DU)
type Result<'T, 'TError> = Ok of 'T | Error of 'TError

// DU with record data
type Command =
    | CreateUser of {| Name: string; Email: string |}
    | UpdateEmail of UserId: int * NewEmail: string
    | DeleteUser of UserId: int
```

### Interfaces

```fsharp
// Interface definition
type IAnimal =
    abstract member Name: string
    abstract member Sound: unit -> string
    abstract member Move: distance: float -> unit

// Default members (F# 5+)
type ILogger =
    abstract member Log: string -> unit
    abstract member LogError: string -> unit
    default this.LogError msg = this.Log $"ERROR: {msg}"
```

### Enums

```fsharp
// Enum (different from DU — has underlying integer type)
type Color =
    | Red = 0
    | Green = 1
    | Blue = 2

// Use enum
let c = Color.Red
let i = int Color.Green    // 1
let e = enum<Color> 2      // Color.Blue

// [<Flags>] enum
[<System.Flags>]
type Permission =
    | None = 0
    | Read = 1
    | Write = 2
    | Execute = 4
    | All = 7
```

### Generic Types

```fsharp
// Generic function
let identity<'T> (x: 'T): 'T = x

// Generic record
type Pair<'A, 'B> = { First: 'A; Second: 'B }

// Generic DU
type Maybe<'T> = Nothing | Just of 'T

// Constraints
let inline add (x: ^T) (y: ^T): ^T =
    x + y  // requires static member (+)

// Type constraints
let printList (items: 'T list when 'T :> System.IComparable) =
    items |> List.sort |> List.iter (printfn "%A")
```

---

## 3. Pattern Matching

### Match Expression

```fsharp
// Basic match
let describe x =
    match x with
    | 0 -> "zero"
    | 1 -> "one"
    | 2 | 3 -> "two or three"         // OR patterns
    | n when n < 0 -> "negative"      // Guard
    | _ -> "other"                    // Wildcard

// Match DU
let area shape =
    match shape with
    | Circle radius -> Math.PI * radius * radius
    | Rectangle (w, h) -> w * h
    | Triangle (b, h) -> 0.5 * b * h

// Match tuples
let describePoint (x, y) =
    match x, y with
    | 0, 0 -> "origin"
    | x, 0 -> $"on x-axis at {x}"
    | 0, y -> $"on y-axis at {y}"
    | x, y -> $"at ({x}, {y})"

// Match lists
let describeList lst =
    match lst with
    | [] -> "empty"
    | [x] -> $"singleton: {x}"
    | [x; y] -> $"pair: {x}, {y}"
    | x :: rest -> $"head: {x}, tail length: {List.length rest}"

// Match Option
let getOrDefault defaultVal opt =
    match opt with
    | Some v -> v
    | None -> defaultVal

// Nested patterns
type Order = { Status: string; Items: string list }

let processOrder order =
    match order with
    | { Status = "pending"; Items = [] } -> "Empty pending order"
    | { Status = "pending"; Items = items } -> $"Pending with {List.length items} items"
    | { Status = "shipped" } -> "Order shipped"
    | _ -> "Unknown status"
```

### Active Patterns

```fsharp
// Single-case active pattern
let (|Even|Odd|) n = if n % 2 = 0 then Even else Odd

match 42 with
| Even -> printfn "Even"
| Odd -> printfn "Odd"

// Partial active pattern
let (|Positive|_|) n = if n > 0 then Some n else None

match -5 with
| Positive n -> printfn "Positive: %d" n
| _ -> printfn "Not positive"

// Active pattern with parameters
let (|Between|_|) low high n =
    if n >= low && n <= high then Some n else None

match 42 with
| Between 1 100 n -> printfn "In range: %d" n
| _ -> printfn "Out of range"

// Multicase active pattern
let (|Vowel|Consonant|Other|) c =
    match System.Char.ToLower c with
    | 'a' | 'e' | 'i' | 'o' | 'u' -> Vowel
    | c when System.Char.IsLetter c -> Consonant
    | _ -> Other
```

### Destructuring in Let

```fsharp
// Tuple destructuring
let (x, y) = (10, 20)

// Record destructuring
let { Name = name; Age = age } = alice

// List destructuring (head :: tail)
let first :: rest = [1; 2; 3; 4; 5]

// Array destructuring
let [| a; b; _ |] = [| 1; 2; 3 |]

// DU destructuring
let (Circle radius) = Circle 5.0

// Destructure in function parameters
let addPair (x, y) = x + y
let greetPerson { Name = n; Age = a } = $"Hi {n}, age {a}"
```

---

## 4. Collections

### List

```fsharp
// Creation
let empty: int list = []
let nums = [1; 2; 3; 4; 5]
let range = [1..10]
let rangeStep = [0..2..20]
let fromArray = Array.toList [|1;2;3|]

// List comprehension
let squares = [for i in 1..10 -> i * i]
let evens = [for i in 1..20 do if i % 2 = 0 then yield i]

// Head/Tail
let head = List.head [1;2;3]         // 1
let tail = List.tail [1;2;3]         // [2;3]
let h :: t = [1;2;3]                 // h=1, t=[2;3]

// Prepend (fast O(1))
let newList = 0 :: [1;2;3]           // [0;1;2;3]

// Append (O(n))
let combined = [1;2;3] @ [4;5;6]    // [1;2;3;4;5;6]

// Transform
List.map (fun x -> x * 2) [1;2;3]                    // [2;4;6]
List.filter (fun x -> x > 2) [1;2;3;4;5]             // [3;4;5]
List.fold (fun acc x -> acc + x) 0 [1;2;3;4;5]       // 15
List.foldBack (fun x acc -> x :: acc) [1;2;3] []     // [1;2;3]
List.reduce (+) [1;2;3;4;5]                           // 15
List.scan (+) 0 [1;2;3]                               // [0;1;3;6]

// Query
List.length [1;2;3]                  // 3
List.isEmpty []                      // true
List.exists (fun x -> x > 3) [1;2;3;4]  // true
List.forall (fun x -> x > 0) [1;2;3]    // true
List.find (fun x -> x > 3) [1;2;3;4]   // 4
List.tryFind (fun x -> x > 10) [1;2;3] // None
List.contains 3 [1;2;3;4]           // true

// Sort
List.sort [3;1;4;1;5;9]             // [1;1;3;4;5;9]
List.sortBy (fun x -> -x) [1;2;3]   // [3;2;1]
List.sortDescending [1;2;3]          // [3;2;1]

// Group/Partition
List.groupBy (fun x -> x % 2) [1;2;3;4;5]
// [(1,[1;3;5]); (0,[2;4])]
List.partition (fun x -> x % 2 = 0) [1;2;3;4]
// ([2;4],[1;3])
List.chunkBySize 2 [1;2;3;4;5]     // [[1;2];[3;4];[5]]
List.splitAt 3 [1;2;3;4;5]         // ([1;2;3],[4;5])

// Combine
List.zip [1;2;3] ["a";"b";"c"]     // [(1,"a");(2,"b");(3,"c")]
List.unzip [(1,"a");(2,"b")]        // ([1;2],["a";"b"])
List.collect (fun x -> [x;x*2]) [1;2;3]  // [1;2;2;4;3;6]
List.concat [[1;2];[3;4];[5]]       // [1;2;3;4;5]

// Index
List.item 2 [1;2;3;4]              // 3
List.nth [1;2;3;4] 2               // 3 (deprecated, use item)
List.indexed [10;20;30]            // [(0,10);(1,20);(2,30)]

// Window/Slide
List.pairwise [1;2;3;4]            // [(1,2);(2,3);(3,4)]
List.windowed 3 [1;2;3;4;5]        // [[1;2;3];[2;3;4];[3;4;5]]

// Other
List.rev [1;2;3]                    // [3;2;1]
List.distinct [1;2;1;3;2]           // [1;2;3]
List.distinctBy fst [(1,"a");(1,"b");(2,"c")]  // [(1,"a");(2,"c")]
List.take 3 [1;2;3;4;5]            // [1;2;3]
List.skip 2 [1;2;3;4;5]            // [3;4;5]
List.truncate 3 [1;2]              // [1;2] (safe)
List.sum [1;2;3;4;5]               // 15
List.sumBy (fun x -> x * x) [1;2;3]  // 14
List.average [1.0;2.0;3.0]         // 2.0
List.min [3;1;4;1]                  // 1
List.max [3;1;4;1]                  // 4
List.minBy snd [(1,5);(2,3);(3,7)] // (2,3)
List.maxBy snd [(1,5);(2,3);(3,7)] // (3,7)
List.choose (fun x -> if x > 2 then Some (x*10) else None) [1;2;3;4]
// [30;40]
```

### Array

```fsharp
// Creation
let arr = [|1; 2; 3; 4; 5|]
let zeros = Array.create 5 0         // [|0;0;0;0;0|]
let init = Array.init 5 id           // [|0;1;2;3;4|]
let copy = Array.copy arr

// Access (mutable!)
arr.[0]                              // 1
arr.[0] <- 10                        // mutate

// 2D arrays
let mat = Array2D.create 3 3 0
mat.[1, 2] <- 5
let rows = Array2D.length1 mat      // 3
let cols = Array2D.length2 mat      // 3

// Most List functions have Array counterparts
Array.map (fun x -> x * 2) arr
Array.filter (fun x -> x > 2) arr
Array.fold (+) 0 arr
Array.sort arr                       // in-place sort
Array.sortInPlace arr               // explicit in-place

// Span (high-performance)
let span = arr.AsSpan()
let slice = arr.AsSpan(1, 3)        // arr[1..3]
```

### Seq (Lazy)

```fsharp
// Creation
let s1 = seq { 1; 2; 3; 4; 5 }
let s2 = Seq.init 10 id              // 0,1,2,...,9
let infinite = Seq.initInfinite id   // 0,1,2,3,... (lazy!)

// Seq comprehension
let squares = seq { for i in 1..10 -> i*i }

// Key: Seq is lazy — computation happens only when consumed
let pipeline =
    Seq.initInfinite id
    |> Seq.map (fun x -> x * x)
    |> Seq.filter (fun x -> x % 2 = 0)
    |> Seq.take 5
    |> Seq.toList                    // [0; 4; 16; 36; 64]

// Most List functions work on Seq too
Seq.map, Seq.filter, Seq.fold, Seq.collect, etc.
```

### Map (Immutable)

```fsharp
// Creation
let empty = Map.empty<string, int>
let m = Map.ofList [("a", 1); ("b", 2); ("c", 3)]
let m2 = [("a", 1); ("b", 2)] |> Map.ofList

// Access
Map.find "a" m                       // 1
Map.tryFind "z" m                    // None
m.["a"]                              // 1 (may throw)
m |> Map.tryFind "a"                 // Some 1

// Modify (creates new map)
let m3 = Map.add "d" 4 m
let m4 = Map.remove "a" m
let m5 = m |> Map.change "a" (Option.map ((*) 2))

// Query
Map.containsKey "a" m               // true
Map.count m                          // 3
Map.isEmpty empty                    // true

// Transform
Map.map (fun k v -> v * 10) m       // Map with doubled values
Map.filter (fun k v -> v > 1) m
Map.toList m                         // [("a",1);("b",2);("c",3)]
Map.keys m |> Seq.toList            // ["a";"b";"c"]
Map.values m |> Seq.toList          // [1;2;3]
```

### Set (Immutable)

```fsharp
// Creation
let empty = Set.empty<int>
let s = Set.ofList [3; 1; 4; 1; 5; 9; 2; 6]  // {1;2;3;4;5;6;9}

// Operations
Set.add 7 s
Set.remove 3 s
Set.contains 4 s                     // true
Set.count s                          // 7

// Set operations
let s1 = Set.ofList [1;2;3;4]
let s2 = Set.ofList [3;4;5;6]
Set.union s1 s2                      // {1;2;3;4;5;6}
Set.intersect s1 s2                  // {3;4}
Set.difference s1 s2                 // {1;2}
Set.isSubset s2 s1                   // false
Set.isProperSubset s2 s1             // false
```

---

## 5. Async/Task Patterns

### Async Workflows

```fsharp
open System
open System.Net.Http

// Async computation
let fetchUrl (url: string) : Async<string> = async {
    use client = new HttpClient()
    let! response = client.GetStringAsync(url) |> Async.AwaitTask
    return response
}

// Run async
let result = fetchUrl "https://example.com" |> Async.RunSynchronously

// Start without waiting
fetchUrl "https://example.com" |> Async.Start

// Start with callback
Async.StartWithContinuations(
    fetchUrl "https://example.com",
    (fun result -> printfn "Success: %d chars" result.Length),
    (fun ex -> printfn "Error: %s" ex.Message),
    (fun _ -> printfn "Cancelled")
)

// Parallel async
let fetchAll urls = async {
    let! results = urls |> List.map fetchUrl |> Async.Parallel
    return Array.toList results
}

// Async with cancellation
let fetchWithTimeout url timeoutMs = async {
    use cts = new System.Threading.CancellationTokenSource(timeoutMs)
    return! Async.StartImmediateAsTask(fetchUrl url, cts.Token)
           |> Async.AwaitTask
}

// Error handling in async
let safeAsync comp = async {
    try
        let! result = comp
        return Ok result
    with ex ->
        return Error ex.Message
}

// Async.Catch
let result' = fetchUrl "url" |> Async.Catch |> Async.RunSynchronously
match result' with
| Choice1Of2 content -> printfn "Got: %d chars" content.Length
| Choice2Of2 ex -> printfn "Error: %s" ex.Message
```

### Task Workflows (F# 6+)

```fsharp
open System.Threading.Tasks

// task {} computation expression
let fetchUrlTask (url: string) : Task<string> = task {
    use client = new HttpClient()
    return! client.GetStringAsync(url)
}

// Async conversion
let fromTask (t: Task<'T>) : Async<'T> = Async.AwaitTask t
let toTask (a: Async<'T>) : Task<'T> = Async.StartAsTask a

// ValueTask
let getValueTask () : ValueTask<int> = ValueTask(42)

// Background task
let backgroundWork () = backgroundTask {
    do! Task.Delay(1000)
    return 42
}

// WhenAll
let runAll tasks = task {
    let! results = Task.WhenAll(tasks)
    return Array.toList results
}

// WhenAny
let runFirst tasks = task {
    let! first = Task.WhenAny(tasks)
    return! first
}
```

### Mailbox Processor

```fsharp
// Message types
type Message =
    | Increment
    | Decrement
    | Get of AsyncReplyChannel<int>
    | Reset

// Create agent
let counter =
    MailboxProcessor.Start(fun inbox ->
        let rec loop count = async {
            let! msg = inbox.Receive()
            match msg with
            | Increment -> return! loop (count + 1)
            | Decrement -> return! loop (count - 1)
            | Get reply ->
                reply.Reply count
                return! loop count
            | Reset -> return! loop 0
        }
        loop 0
    )

// Use agent
counter.Post Increment
counter.Post Increment
counter.Post Increment
let value = counter.PostAndReply Get     // 3
```

---

## 6. Operators

### Built-in Operators

```fsharp
// Pipe operators
x |> f              // f x   (forward pipe)
f <| x              // f x   (backward pipe)
x ||> f             // f x1 x2 (two-arg forward pipe)
x |||> f            // f x1 x2 x3 (three-arg)

// Composition operators
f >> g              // fun x -> g (f x)   (forward compose)
f << g              // fun x -> f (g x)   (backward compose)

// Arithmetic
+, -, *, /          // standard
%                   // modulo
**                  // float power (2.0 ** 3.0)
pown 2 10           // integer power (1024)

// Comparison
=, <>, <, >, <=, >= // structural equality
==, !=              // NOT F# — use = and <>
obj.Equals(obj2)    // reference comparison sometimes needed

// Logical
&&, ||, not
&&&, |||, ^^^, ~~~  // bitwise AND, OR, XOR, NOT
<<<, >>>            // bitwise shift left/right

// String
+                   // string concatenation (prefer sprintf)
sprintf "%s %s" s1 s2

// Type operators
:?                  // type test  (obj :? string)
:?>                 // downcast   (obj :?> string)
:>                  // upcast     (derived :> base)
box                 // box to obj
unbox<'T>           // unbox from obj
```

### Custom Operators

```fsharp
// Infix operator (must start with !, %, &, *, +, -, ., /, <, =, >, ?, @, ^, |, ~)
let (|+|) x y = x + y + 1
let result = 3 |+| 4    // 8

// Prefix operator (must start with !)
let (!!) x = not x
let isOdd = !! (5 % 2 = 0)   // true

// Result binding operator (common pattern)
let (>>=) result f =
    match result with
    | Ok v -> f v
    | Error e -> Error e

let validateAge age =
    if age >= 0 && age <= 150 then Ok age
    else Error "Invalid age"

let validateName name =
    if String.length name > 0 then Ok name
    else Error "Name cannot be empty"

let result =
    Ok { Age = 25; Name = "" }
    >>= (fun p -> validateAge p.Age |> Result.map (fun a -> { p with Age = a }))
    >>= (fun p -> validateName p.Name |> Result.map (fun n -> { p with Name = n }))
```

### Common Pipeline Patterns

```fsharp
// Data transformation pipeline
let processData data =
    data
    |> List.filter (fun x -> x > 0)
    |> List.map (fun x -> x * 2)
    |> List.sortDescending
    |> List.take 5
    |> List.sum

// Function composition pipeline
let processString =
    String.trim >> String.toLower >> (fun s -> s.Replace(" ", "-"))

// Point-free style
let sumOfSquares = List.map (fun x -> x * x) >> List.sum

// With tap (for debugging in pipelines)
let tap f x = f x; x    // side effect then return value
let debug label x = tap (printfn "[%s] %A" label) x

let result =
    [1..10]
    |> debug "input"
    |> List.map (fun x -> x * x)
    |> debug "squared"
    |> List.filter (fun x -> x > 20)
    |> debug "filtered"
    |> List.sum
```

---

## 7. Computation Expressions

### Sequence Expressions

```fsharp
// seq computation
let fibs = seq {
    let mutable a, b = 0, 1
    while true do
        yield a
        let c = a + b
        a <- b
        b <- c
}

let first10Fibs = fibs |> Seq.take 10 |> Seq.toList
// [0; 1; 1; 2; 3; 5; 8; 13; 21; 34]

// Sequence with for
let evenSquares = seq {
    for i in 1..100 do
        if i % 2 = 0 then
            yield i * i
}
```

### Async Expressions

```fsharp
let workflow = async {
    let! value1 = someAsyncOp1()     // await
    let! value2 = someAsyncOp2()     // await
    do! Async.Sleep 1000             // await unit
    return value1 + value2
}
```

### Result/Option Expressions (Custom CE)

```fsharp
// Result computation expression
type ResultBuilder() =
    member _.Bind(result, f) =
        match result with
        | Ok v -> f v
        | Error e -> Error e
    member _.Return(v) = Ok v
    member _.ReturnFrom(r) = r

let result = ResultBuilder()

let validate input = result {
    let! trimmed = if String.length input > 0 then Ok input else Error "Empty"
    let! upper = if trimmed.Length < 100 then Ok (trimmed.ToUpper()) else Error "Too long"
    return upper
}

// Option computation expression (using option {} from FSharp.Core)
let safeDiv a b = if b = 0 then None else Some (a / b)

let compute a b c = option {
    let! x = safeDiv a b
    let! y = safeDiv x c
    return x + y
}
```

### List/Seq Expressions (Monad)

```fsharp
// List monad (non-determinism)
let pairs = [
    let xs = [1;2;3]
    let ys = [10;20;30]
    for x in xs do
        for y in ys do
            yield (x, y)
]
// [(1,10);(1,20);(1,30);(2,10);...]
```

### Custom Computation Expression

```fsharp
// Minimal CE builder
type MaybeBuilder() =
    member _.Bind(x, f) =
        match x with
        | None -> None
        | Some v -> f v
    member _.Return(v) = Some v
    member _.ReturnFrom(x) = x
    member _.Zero() = None
    member _.Combine(a, b) = if Option.isSome a then a else b
    member _.Delay(f) = f
    member _.Run(f) = f()

let maybe = MaybeBuilder()

let safeLookup (map: Map<'K,'V>) key = Map.tryFind key map

let result = maybe {
    let! user = safeLookup users userId
    let! email = safeLookup emails user.EmailId
    return email.Address
}
```

---

## 8. Modules and Namespaces

### Module Syntax

```fsharp
// Simple module
module MathUtils =
    let add x y = x + y
    let multiply x y = x * y
    let square x = x * x

// Use module
let result = MathUtils.add 3 4
open MathUtils      // import all
let result2 = add 3 4

// Nested modules
module Outer =
    module Inner =
        let hello() = "Hello from Inner"
    let greet() = Inner.hello()

// [<AutoOpen>] — auto-opened when parent is opened
[<AutoOpen>]
module Helpers =
    let inline flip f x y = f y x
    let inline curry f x y = f (x, y)
    let inline uncurry f (x, y) = f x y

// Recursive module (can use types mutually recursive)
module rec Domain =
    type Tree =
        | Leaf of int
        | Branch of Tree * Tree

    let sum = function
        | Leaf n -> n
        | Branch (l, r) -> sum l + sum r
```

### Namespace Syntax

```fsharp
// Namespace (no code, only type/module declarations)
namespace MyCompany.MyApp.Domain

type Customer = { Id: int; Name: string }

module CustomerOps =
    let create id name = { Id = id; Name = name }

// Namespace with module
namespace MyCompany.MyApp

module Config =
    let defaultTimeout = 30
    let maxRetries = 3
```

### Access Modifiers

```fsharp
// public (default)
let publicValue = 42

// internal (same assembly)
let internal internalValue = 42

// private (same module/type)
let private privateValue = 42

// On types
type public PublicRecord = { Value: int }
type internal InternalRecord = { Value: int }
type private PrivateRecord = { Value: int }

// On members
type MyClass() =
    member private this.secret = 42
    member internal this.internal' = 43
    member public this.Public = 44
```

### Common open Statements

```fsharp
// Standard opens
open System
open System.IO
open System.Collections.Generic
open System.Text
open System.Threading
open System.Threading.Tasks
open System.Linq               // LINQ extension methods

// F# specific
open Microsoft.FSharp.Core
open Microsoft.FSharp.Collections
open Microsoft.FSharp.Control    // Async, MailboxProcessor

// Common libraries (after NuGet install)
open Newtonsoft.Json
open FSharp.Data
open Giraffe
open Saturn
open Dapper
```

---

## 9. Standard Library Functions

### Core Functions

```fsharp
// I/O
printfn "format %s %d" str n     // stdout with newline
printf "format %s" str            // stdout without newline
eprintfn "error %s" msg           // stderr
sprintf "format %A" value         // format to string

// Type conversions
int "42"           // parse string to int (throws on failure)
int 3.7            // float to int (truncates)
float 42           // int to float
float "3.14"       // parse string to float
string 42          // to string
string true        // "true"
char 65            // 'A'
int 'A'            // 65
bool "true"        // true
decimal "3.14"     // 3.14M

// Option functions
Option.map f opt              // apply f if Some
Option.bind f opt             // f returns option (flatMap)
Option.filter pred opt        // None if pred fails
Option.defaultValue def opt   // unwrap or use default
Option.defaultWith f opt      // defaultValue with lazy default
Option.orElse alt opt         // opt or alt if None
Option.orElseWith f opt       // lazy orElse
Option.isSome opt             // true if Some
Option.isNone opt             // true if None
Option.get opt                // unwrap (throws if None!)
Option.toList opt             // [] or [v]
Option.toArray opt            // [||] or [|v|]

// Result functions
Result.map f result           // apply f to Ok value
Result.mapError f result      // apply f to Error value
Result.bind f result          // f returns Result (flatMap)
Result.isOk result            // true if Ok
Result.isError result         // true if Error

// Ignore
ignore value                  // discard value, return unit

// Comparison
compare x y                   // -1, 0, or 1
min x y                       // smaller of two
max x y                       // larger of two

// Lazy
let lazyValue = lazy (expensiveComputation())
let result = lazyValue.Value  // computed only on first access
lazyValue.IsValueCreated     // false until accessed
```

### String Module

```fsharp
open System

// Common string operations
String.length "hello"            // 5
String.concat ", " ["a";"b";"c"] // "a, b, c"
String.split [|','|] "a,b,c"    // F# String.split

// System.String methods (use as extension)
"hello".Length                   // 5
"hello".ToUpper()                // "HELLO"
"  hello  ".Trim()               // "hello"
"hello world".Split(' ')        // [|"hello";"world"|]
"hello".StartsWith("hel")       // true
"hello".EndsWith("llo")         // true
"hello".Contains("ell")         // true
"hello".IndexOf('l')            // 2
"hello".Replace("l", "r")       // "herro"
"hello".[1..3]                   // "ell"
String.IsNullOrEmpty ""          // true
String.IsNullOrWhiteSpace "  "   // true
String.Join(", ", ["a";"b";"c"]) // "a, b, c"
```

### Math

```fsharp
open System

Math.PI                   // 3.14159...
Math.E                    // 2.71828...
Math.Abs -5               // 5
Math.Abs -5.0             // 5.0
Math.Ceiling 3.2          // 4.0
Math.Floor 3.8            // 3.0
Math.Round 3.5            // 4.0 (banker's rounding)
Math.Round(3.567, 2)      // 3.57
Math.Sqrt 16.0            // 4.0
Math.Pow(2.0, 10.0)       // 1024.0
Math.Log 1.0              // 0.0 (natural log)
Math.Log10 100.0          // 2.0
Math.Sin, Math.Cos, Math.Tan
Math.Min(3, 5)            // 3
Math.Max(3, 5)            // 5
Math.Clamp(10, 0, 5)      // 5 (clamp to range)
Math.Sign -5              // -1 (sign function)
Math.Truncate 3.9         // 3.0

// F# math functions
abs, sqrt, log, log10, exp
sin, cos, tan, asin, acos, atan, atan2
ceil, floor, round
infinity                  // System.Double.PositiveInfinity
nan                       // System.Double.NaN
isNan, isInfinity
```

### DateTime

```fsharp
open System

// Create
let now = DateTime.Now
let utcNow = DateTime.UtcNow
let today = DateTime.Today
let specific = DateTime(2024, 1, 15, 10, 30, 0)
let fromString = DateTime.Parse("2024-01-15")
let ok, dt = DateTime.TryParse("2024-01-15")

// Properties
now.Year, now.Month, now.Day
now.Hour, now.Minute, now.Second
now.DayOfWeek, now.DayOfYear

// Operations
let tomorrow = today.AddDays(1.0)
let nextMonth = today.AddMonths(1)
let diff = now - DateTime(2000, 1, 1)  // TimeSpan
diff.Days, diff.Hours, diff.TotalHours

// Format
now.ToString("yyyy-MM-dd")             // "2024-01-15"
now.ToString("dd/MM/yyyy HH:mm:ss")    // "15/01/2024 10:30:00"

// DateTimeOffset (timezone-aware)
let dto = DateTimeOffset.UtcNow
let local = dto.ToLocalTime()

// TimeSpan
let ts = TimeSpan.FromHours 1.5       // 1:30:00
let ts2 = TimeSpan(days=1, hours=2, minutes=30, seconds=0, milliseconds=0)
```

### Environment and IO

```fsharp
open System
open System.IO

// Environment
Environment.MachineName
Environment.UserName
Environment.OSVersion
Environment.ProcessorCount
Environment.GetEnvironmentVariable("PATH")
Environment.GetCommandLineArgs()
Environment.CurrentDirectory

// File I/O
File.ReadAllText "file.txt"
File.ReadAllLines "file.txt"
File.WriteAllText("file.txt", "content")
File.WriteAllLines("file.txt", ["line1"; "line2"])
File.AppendAllText("file.txt", "appended")
File.Exists "file.txt"
File.Delete "file.txt"
File.Copy("src.txt", "dst.txt")
File.Move("src.txt", "dst.txt")

// Directory
Directory.CreateDirectory "path/to/dir"
Directory.Exists "path"
Directory.GetFiles("path", "*.fs")
Directory.GetDirectories "path"
Directory.Delete("path", recursive=true)

// Path
Path.Combine("dir", "subdir", "file.txt")
Path.GetFileName "/path/to/file.txt"     // "file.txt"
Path.GetFileNameWithoutExtension "file.txt"  // "file"
Path.GetExtension "file.txt"             // ".txt"
Path.GetDirectoryName "/path/to/file.txt"  // "/path/to"
Path.GetFullPath "relative/path"
Path.GetTempPath()
Path.GetTempFileName()

// StreamReader/Writer
use reader = new StreamReader("file.txt")
let content = reader.ReadToEnd()

use writer = new StreamWriter("file.txt")
writer.WriteLine("Hello")
```

---

## 10. Object-Oriented Features

### Classes

```fsharp
// Basic class
type Person(name: string, age: int) =        // primary constructor
    // Fields
    let mutable _age = age                    // private field

    // Properties
    member this.Name = name                   // auto-property (read-only)
    member this.Age
        with get() = _age
        and set(value) =
            if value >= 0 then _age <- value

    // Methods
    member this.Greet() =
        printfn "Hi, I'm %s!" this.Name

    member this.Birthday() =
        _age <- _age + 1

    // Static members
    static member Create(n, a) = Person(n, a)

    // Override
    override this.ToString() =
        $"Person({this.Name}, {this.Age})"

// Additional constructors
type Rectangle(w: float, h: float) =
    new(side: float) = Rectangle(side, side)  // square constructor
    member _.Width = w
    member _.Height = h
    member this.Area = this.Width * this.Height
```

### Inheritance

```fsharp
// Base class
type Animal(name: string) =
    member _.Name = name
    abstract member Sound: unit -> string
    default _.Sound() = "..."

// Derived class
type Dog(name: string) =
    inherit Animal(name)
    override _.Sound() = "Woof"
    member this.Fetch() = printfn "%s fetches the ball!" this.Name

// Type test and cast
let makeNoise (animal: Animal) =
    match animal with
    | :? Dog as dog -> dog.Fetch(); dog.Sound()
    | _ -> animal.Sound()
```

### Interfaces

```fsharp
type IShape =
    abstract member Area: float
    abstract member Perimeter: float

// Implement interface
type Circle(radius: float) =
    interface IShape with
        member _.Area = Math.PI * radius * radius
        member _.Perimeter = 2.0 * Math.PI * radius

// Object expression (inline implementation)
let makeCircle r =
    { new IShape with
        member _.Area = Math.PI * r * r
        member _.Perimeter = 2.0 * Math.PI * r }
```

---

## 11. Error Handling

### Option Type

```fsharp
// Creating Options
let some = Some 42
let none: int option = None

// Using Options
let doubled = Option.map ((*) 2) (Some 5)      // Some 10
let chained = Some 10 |> Option.bind (fun x -> if x > 5 then Some x else None)

// Practical patterns
let safeDivide a b = if b = 0 then None else Some (a / b)
let safeHead list = List.tryHead list
let safeFind pred list = List.tryFind pred list

// Collecting Options
let results = [Some 1; None; Some 3; None; Some 5]
let values = List.choose id results   // [1; 3; 5]

// Default values
let value = Option.defaultValue 0 (Some 42)     // 42
let value2 = Option.defaultValue 0 None          // 0
```

### Result Type

```fsharp
// Creating Results
let success: Result<int, string> = Ok 42
let failure: Result<int, string> = Error "something went wrong"

// Railway-oriented programming
let validatePositive x =
    if x > 0 then Ok x else Error $"{x} is not positive"

let validateMax limit x =
    if x <= limit then Ok x else Error $"{x} exceeds limit {limit}"

let processValue input =
    input
    |> validatePositive
    |> Result.bind (validateMax 100)
    |> Result.map (fun x -> x * 2)

// Collecting Results
let allOk results =
    results |> List.fold (fun acc r ->
        match acc, r with
        | Ok xs, Ok x -> Ok (xs @ [x])
        | Error e, _ | _, Error e -> Error e
    ) (Ok [])
```

### Exception Handling

```fsharp
// try/with
let safeRead filename =
    try
        File.ReadAllText filename |> Ok
    with
    | :? FileNotFoundException -> Error "File not found"
    | :? UnauthorizedAccessException -> Error "Access denied"
    | ex -> Error $"Unexpected: {ex.Message}"

// try/finally
let withResource () =
    let resource = acquireResource()
    try
        useResource resource
    finally
        releaseResource resource

// using / use (IDisposable)
let readFile path = 
    use reader = new StreamReader(path)
    reader.ReadToEnd()

// Custom exceptions
exception DatabaseError of string
exception ValidationError of string * string   // field, message

// Raise
raise (DatabaseError "Connection failed")
raise (ValidationError ("email", "Invalid format"))
```

---

## 12. Common NuGet Packages

### Core F# Libraries

| Package | Purpose | Install |
|---------|---------|---------|
| `FSharp.Core` | Core F# runtime | Built-in |
| `FSharp.Data` | Type providers for CSV, JSON, XML, HTML, SQL | `dotnet add package FSharp.Data` |
| `FSharpPlus` | Extended functional programming | `dotnet add package FSharpPlus` |
| `Chessie` | Railway-oriented programming | `dotnet add package Chessie` |
| `FsToolkit.ErrorHandling` | Result/Async/Option CE | `dotnet add package FsToolkit.ErrorHandling` |

### Web Frameworks

| Package | Purpose | Install |
|---------|---------|---------|
| `Giraffe` | Functional web framework on ASP.NET Core | `dotnet add package Giraffe` |
| `Saturn` | MVC-style framework on Giraffe | `dotnet add package Saturn` |
| `Fable.Compiler` | F# to JavaScript compiler | `dotnet add package Fable.Compiler` |
| `Elmish` | The Elm Architecture for F# | `dotnet add package Fable.Elmish` |
| `Falco` | Lightweight functional web framework | `dotnet add package Falco` |

### Data Access

| Package | Purpose | Install |
|---------|---------|---------|
| `Dapper` | Micro-ORM for SQL | `dotnet add package Dapper` |
| `Microsoft.EntityFrameworkCore` | Full ORM | `dotnet add package Microsoft.EntityFrameworkCore` |
| `SQLProvider` | Type provider for databases | `dotnet add package SQLProvider` |
| `Npgsql` | PostgreSQL driver | `dotnet add package Npgsql` |
| `MongoDB.Driver` | MongoDB driver | `dotnet add package MongoDB.Driver` |
| `StackExchange.Redis` | Redis client | `dotnet add package StackExchange.Redis` |

### JSON/Serialization

| Package | Purpose | Install |
|---------|---------|---------|
| `Newtonsoft.Json` | JSON serialization | `dotnet add package Newtonsoft.Json` |
| `System.Text.Json` | Built-in JSON | Built-in (.NET 5+) |
| `FSharp.SystemTextJson` | F# DU/record support for STJ | `dotnet add package FSharp.SystemTextJson` |
| `Thoth.Json` | F#-native JSON | `dotnet add package Thoth.Json.Net` |

### Testing

| Package | Purpose | Install |
|---------|---------|---------|
| `xunit` | Unit testing framework | `dotnet add package xunit` |
| `Expecto` | F#-native testing | `dotnet add package Expecto` |
| `FsUnit` | Fluent assertions for F# | `dotnet add package FsUnit.xUnit` |
| `FsCheck` | Property-based testing | `dotnet add package FsCheck` |
| `NSubstitute` | Mocking framework | `dotnet add package NSubstitute` |
| `Shouldly` | Better assertions | `dotnet add package Shouldly` |

### Logging and Observability

| Package | Purpose | Install |
|---------|---------|---------|
| `Serilog` | Structured logging | `dotnet add package Serilog` |
| `Serilog.AspNetCore` | ASP.NET Core sink | `dotnet add package Serilog.AspNetCore` |
| `Microsoft.Extensions.Logging` | Standard logging | Built-in |
| `OpenTelemetry` | Distributed tracing | `dotnet add package OpenTelemetry` |

### Build and Tooling

| Package | Purpose | Install |
|---------|---------|---------|
| `FAKE` | F# build tool | `dotnet tool install fake-cli` |
| `fantomas` | Code formatter | `dotnet tool install fantomas` |
| `paket` | Dependency manager | `dotnet tool install Paket` |
| `dotnet-script` | F# scripting tool | `dotnet tool install dotnet-script` |

### Reactive/Concurrent

| Package | Purpose | Install |
|---------|---------|---------|
| `System.Reactive` | Rx.NET | `dotnet add package System.Reactive` |
| `FSharp.Control.Reactive` | F# wrappers for Rx | `dotnet add package FSharp.Control.Reactive` |
| `Akka.FSharp` | Actor model | `dotnet add package Akka.FSharp` |

---

## 13. F# vs C# Syntax Comparison

### Basic Syntax

| Concept | F# | C# |
|---------|----|----|
| Variable | `let x = 42` | `var x = 42;` |
| Mutable | `let mutable x = 42; x <- 43` | `int x = 42; x = 43;` |
| Function | `let add x y = x + y` | `int Add(int x, int y) => x + y;` |
| Lambda | `fun x -> x * 2` | `x => x * 2` |
| Pipe | `x \|> f \|> g` | `g(f(x))` |
| Tuple | `(1, "hello")` | `(1, "hello")` (C# 7+) |
| Record | `{ Name = "Alice" }` | `new { Name = "Alice" }` (anon) |
| Immutable | default | requires `const`/`readonly` |
| Null | Option type | nullable reference types |
| Semicolons | no | required |
| Braces | no (indentation) | required |

### Type Definitions

| Concept | F# | C# |
|---------|----|----|
| Record | `type R = { X: int }` | `record R(int X);` (C# 9+) |
| Discriminated Union | `type T = A \| B of int` | no equivalent |
| Interface | `type I = abstract M: unit -> int` | `interface I { int M(); }` |
| Class | `type C(x) = member _.X = x` | `class C { int X { get; } C(int x) { X = x; } }` |
| Enum | `type E = A = 1 \| B = 2` | `enum E { A = 1, B = 2 }` |

### Null Handling

| Concept | F# | C# |
|---------|----|----|
| Maybe value | `Option<'T>` (None/Some) | `T?` (nullable) |
| Check null | `match opt with \| Some v -> ...` | `if (x != null) ...` |
| Default | `Option.defaultValue def opt` | `x ?? defaultValue` |
| Chain | `Option.bind f opt` | `x?.Property?.Method()` |

### Error Handling

| Concept | F# | C# |
|---------|----|----|
| Success/Failure | `Result<'T, 'E>` (Ok/Error) | no built-in equivalent |
| Exception | `try ... with \| :? Ex as e -> ...` | `try { } catch (Ex e) { }` |
| Finally | `try ... finally ...` | `try { } finally { }` |
| Using | `use r = new Resource()` | `using var r = new Resource();` |

### Async

| Concept | F# | C# |
|---------|----|----|
| Async computation | `async { let! x = ...; return x }` | `async Task<T> { var x = await ...; return x; }` |
| Await | `let! result = asyncOp` | `var result = await asyncOp;` |
| Run | `Async.RunSynchronously asyncOp` | `.GetAwaiter().GetResult()` |
| Task | `task { let! x = ...; return x }` | Same as C# async |

---

## ตัวอย่างโปรแกรมสมบูรณ์ (Complete Example Programs)

### FizzBuzz

```fsharp
let fizzBuzz n =
    match n % 3, n % 5 with
    | 0, 0 -> "FizzBuzz"
    | 0, _ -> "Fizz"
    | _, 0 -> "Buzz"
    | _ -> string n

[1..100] |> List.map fizzBuzz |> List.iter printfn
```

### Caesar Cipher

```fsharp
let caesarEncrypt shift (text: string) =
    text
    |> Seq.map (fun c ->
        if System.Char.IsLetter c then
            let base' = if System.Char.IsUpper c then int 'A' else int 'a'
            char ((int c - base' + shift) % 26 + base')
        else c
    )
    |> System.String.Concat

let caesarDecrypt shift = caesarEncrypt (26 - shift)

let encrypted = caesarEncrypt 13 "Hello, World!"  // "Uryyb, Jbeyq!"
let decrypted = caesarDecrypt 13 encrypted          // "Hello, World!"
```

### Binary Search Tree

```fsharp
type BST<'a> =
    | Empty
    | Node of 'a * BST<'a> * BST<'a>

let rec insert x = function
    | Empty -> Node (x, Empty, Empty)
    | Node (v, l, r) when x < v -> Node (v, insert x l, r)
    | Node (v, l, r) when x > v -> Node (v, l, insert x r)
    | tree -> tree  // duplicate

let rec contains x = function
    | Empty -> false
    | Node (v, l, _) when x < v -> contains x l
    | Node (v, _, r) when x > v -> contains x r
    | _ -> true

let rec inOrder = function
    | Empty -> []
    | Node (v, l, r) -> inOrder l @ [v] @ inOrder r

let tree =
    [5; 3; 7; 1; 4; 6; 8]
    |> List.fold (fun t x -> insert x t) Empty

inOrder tree  // [1; 3; 4; 5; 6; 7; 8]
```

---

*F# Cheat Sheet — ปรับปรุงสำหรับ F# 8 และ .NET 8 — ตุลาคม 2026*
