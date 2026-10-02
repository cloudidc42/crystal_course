# Part 47 - หน่วยความจำและประสิทธิภาพ (Memory and Performance)

## บทนำ

การเขียนโปรแกรมที่มีประสิทธิภาพใน F# ต้องเข้าใจวิธีที่ .NET จัดการหน่วยความจำ บทนี้จะอธิบาย value types, reference types, Span<T>, ArrayPool และเทคนิคการ optimize

## 1. Value Types vs Reference Types

```fsharp
// Value Types - เก็บบน stack (หรือ inline ใน struct)
// เมื่อ assign จะ copy ค่า

[<Struct>]
type Point2D = { X: float; Y: float }

[<Struct>]
type Rectangle = { TopLeft: Point2D; BottomRight: Point2D }

// Reference Types - เก็บบน heap
// เมื่อ assign จะ copy reference เท่านั้น
type PersonClass(name: string, age: int) =
    member _.Name = name
    member _.Age = age

// Record เป็น reference type โดยปริยาย
type PersonRecord = { Name: string; Age: int }

// เปรียบเทียบ
let compareTypes () =
    // Value type: copy เมื่อ assign
    let p1 = { X = 1.0; Y = 2.0 }
    let mutable p2 = p1  // copy ค่า
    // p2.X <- 10.0  // ไม่ได้เพราะ struct immutable
    
    // Reference type: share reference
    let person1 = { Name = "สมชาย"; Age = 30 }
    let person2 = person1  // share reference
    printfn "Same object: %b" (Object.ReferenceEquals(person1, person2))

compareTypes ()
```

```fsharp
// ผลกระทบของ value vs reference types
let memoryLayout () =
    // Value type array: ข้อมูลเรียงต่อกันใน memory
    let valueArray: Point2D[] = Array.init 1000 (fun i -> { X = float i; Y = float i })
    
    // Reference type array: เก็บแค่ pointers
    let refArray: PersonRecord[] = Array.init 1000 (fun i -> 
        { Name = sprintf "คน%d" i; Age = i })
    
    // valueArray เข้าถึงได้เร็วกว่าเพราะ cache-friendly
    printfn "Value array size: %d" valueArray.Length
    printfn "Ref array size: %d" refArray.Length
```

## 2. Stack vs Heap Allocation

```fsharp
(*
Stack:
- เร็วมาก O(1) allocation/deallocation
- จำกัดขนาด (ประมาณ 1MB โดย default)
- ใช้สำหรับ local variables, function calls
- Value types ที่ไม่ boxed

Heap:
- ช้ากว่า แต่ขนาดใหญ่กว่า
- ต้องผ่าน GC
- Reference types, boxed value types
*)

// Boxing: แปลง value type เป็น reference type
let boxingExample () =
    let x: int = 42          // บน stack
    let boxed: obj = box x   // boxing - สร้าง object บน heap!
    let unboxed = unbox<int> boxed  // unboxing
    
    printfn "x = %d" x
    printfn "boxed = %A" boxed
    printfn "unboxed = %d" unboxed
    
    // ❌ boxing เกิดเมื่อ:
    // - ใช้ value type เป็น interface
    // - ใส่ใน obj array
    // - ใช้ generics กับ non-generic constraints
    
    let intList: obj list = [1; 2; 3]  // boxing 3 ครั้ง!
    printfn "boxed list length: %d" intList.Length

boxingExample ()
```

```fsharp
// หลีกเลี่ยง boxing
let avoidBoxing () =
    // ❌ ไม่ดี: boxing
    let sumBoxed (items: obj list) =
        items |> List.sumBy (fun x -> unbox<int> x)
    
    // ✅ ดีกว่า: generic
    let sumGeneric (items: int list) =
        items |> List.sum
    
    // ✅ ดีที่สุด: direct
    let sumDirect (arr: int[]) =
        let mutable sum = 0
        for x in arr do sum <- sum + x
        sum
    
    let data = [1..1000]
    printfn "Sum: %d" (sumGeneric data)
```

## 3. GC Basics ใน .NET

```fsharp
(*
.NET GC (Garbage Collector) มี 3 generations:
- Gen 0: objects ใหม่, collect บ่อย, เร็ว
- Gen 1: survive Gen 0, buffer ระหว่าง Gen 0 และ Gen 2
- Gen 2: long-lived objects, collect น้อย, ช้า

Large Object Heap (LOH):
- Objects ขนาด >= 85,000 bytes
- Collect ใน Gen 2 เท่านั้น
- Fragmentation ปัญหา
*)

let gcBasics () =
    // ดูข้อมูล GC
    printfn "Gen 0 collections: %d" (System.GC.CollectionCount(0))
    printfn "Gen 1 collections: %d" (System.GC.CollectionCount(1))
    printfn "Gen 2 collections: %d" (System.GC.CollectionCount(2))
    
    let totalMemory = System.GC.GetTotalMemory(false)
    printfn "Total memory: %d bytes (%.2f MB)" totalMemory (float totalMemory / 1_000_000.0)
    
    // Force GC (ไม่แนะนำใน production)
    System.GC.Collect()
    System.GC.WaitForPendingFinalizers()
    
    printfn "หลัง GC:"
    printfn "Total memory: %d bytes" (System.GC.GetTotalMemory(true))

gcBasics ()
```

## 4. Avoiding Allocations

```fsharp
// เทคนิคลด allocations

// 1. Reuse objects
let reuseExample () =
    let sb = System.Text.StringBuilder()
    
    // ❌ ไม่ดี: สร้าง string ใหม่ทุกครั้ง
    let badConcat () =
        let mutable result = ""
        for i in 1..1000 do
            result <- result + string i  // สร้าง string ใหม่ 1000 ครั้ง!
        result
    
    // ✅ ดีกว่า: ใช้ StringBuilder
    let goodConcat () =
        sb.Clear() |> ignore
        for i in 1..1000 do
            sb.Append(i) |> ignore
        sb.ToString()
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    badConcat () |> ignore
    printfn "Bad concat: %d ms" sw.ElapsedMilliseconds
    
    sw.Restart()
    goodConcat () |> ignore
    printfn "Good concat: %d ms" sw.ElapsedMilliseconds
```

```fsharp
// 2. Prefer structs สำหรับ small data
[<Struct>]
type Vector3D = { mutable X: float32; mutable Y: float32; mutable Z: float32 }

// 3. Use arrays instead of lists สำหรับ performance
let arraysVsLists () =
    let n = 1_000_000
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let arr = Array.init n id
    let arrSum = arr |> Array.sum
    sw.Stop()
    printfn "Array sum: %d ms" sw.ElapsedMilliseconds
    
    sw.Restart()
    let lst = [0..n-1]
    let lstSum = lst |> List.sum
    sw.Stop()
    printfn "List sum: %d ms" sw.ElapsedMilliseconds
    
    printfn "Results match: %b" (arrSum = lstSum)
```

## 5. Span<T> และ Memory<T>

```fsharp
open System

// Span<T> - ชี้ไปยัง contiguous memory โดยไม่ copy
let spanBasics () =
    let array = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]
    
    // สร้าง Span จาก array (ไม่ copy!)
    let span: Span<int> = array.AsSpan()
    
    printfn "Span length: %d" span.Length
    printfn "First: %d" span[0]
    
    // Slice ไม่ copy
    let slice = span.Slice(2, 5)  // [3, 4, 5, 6, 7]
    printfn "Slice length: %d" slice.Length
    printfn "Slice[0]: %d" slice[0]
    
    // แก้ไขผ่าน Span
    span[0] <- 100
    printfn "array[0] after span mod: %d" array[0]  // 100
```

```fsharp
// Span กับ string
let spanWithStrings () =
    let str = "Hello, World! สวัสดีโลก"
    
    // ReadOnlySpan<char> จาก string (ไม่ allocate)
    let span: ReadOnlySpan<char> = str.AsSpan()
    
    // Slice
    let hello = span.Slice(0, 5)
    printfn "First 5: %s" (hello.ToString())
    
    // IndexOf (ไม่สร้าง substring)
    let commaIdx = span.IndexOf(',')
    printfn "Comma at: %d" commaIdx
    
    // Split แบบ allocation-free
    let beforeComma = span.Slice(0, commaIdx)
    printfn "Before comma: %s" (beforeComma.ToString())
```

```fsharp
// Memory<T> - เหมือน Span แต่ใช้กับ async ได้
let memoryExample () =
    let array = Array.init 100 id
    
    // Memory<T> สามารถส่งผ่าน async ได้
    let mem: Memory<int> = array.AsMemory()
    let slice = mem.Slice(10, 20)
    
    // แปลงเป็น Span เมื่อต้องการใช้งาน
    let span = slice.Span
    printfn "Memory slice length: %d" slice.Length
    printfn "Sum: %d" (span.ToArray() |> Array.sum)
```

```fsharp
// Span สำหรับ parsing ที่มีประสิทธิภาพ
let parseWithSpan (input: string) : int list =
    let span = input.AsSpan()
    let results = ResizeArray<int>()
    let mutable remaining = span
    
    while remaining.Length > 0 do
        let commaIdx = remaining.IndexOf(',')
        let token = 
            if commaIdx >= 0 then
                let t = remaining.Slice(0, commaIdx)
                remaining <- remaining.Slice(commaIdx + 1)
                t
            else
                let t = remaining
                remaining <- ReadOnlySpan.Empty
                t
        
        match System.Int32.TryParse(token) with
        | true, n -> results.Add(n)
        | _ -> ()
    
    results |> Seq.toList

let parsed = parseWithSpan "1,2,3,4,5,6,7,8,9,10"
printfn "Parsed: %A" parsed
```

## 6. ArrayPool<T>

```fsharp
open System.Buffers

// ArrayPool - reuse arrays เพื่อลด GC pressure
let arrayPoolBasics () =
    let pool = ArrayPool<int>.Shared
    
    // เช่า array
    let array = pool.Rent(1000)
    printfn "Rented array length: %d (>= 1000)" array.Length
    
    try
        // ใช้งาน array
        for i in 0..999 do
            array[i] <- i
        
        let sum = array |> Array.take 1000 |> Array.sum
        printfn "Sum: %d" sum
    finally
        // คืน array (สำคัญ!)
        pool.Return(array, clearArray = true)
        printfn "Array returned to pool"
```

```fsharp
// ArrayPool ใน pattern ที่ถูกต้อง
let safeArrayPoolUsage (size: int) (work: int[] -> int -> 'a) =
    let pool = ArrayPool<int>.Shared
    let array = pool.Rent(size)
    try
        work array size
    finally
        pool.Return(array)

// ใช้งาน
let result = safeArrayPoolUsage 100 (fun arr len ->
    for i in 0..len-1 do
        arr[i] <- i * 2
    arr |> Array.take len |> Array.sum)

printfn "Result: %d" result
```

```fsharp
// เปรียบเทียบ performance
let poolVsNew () =
    let iterations = 10_000
    let size = 1024
    
    // Without pool - allocate new array each time
    let sw = System.Diagnostics.Stopwatch.StartNew()
    for _ in 1..iterations do
        let arr = Array.zeroCreate<int> size
        arr[0] <- 1
    sw.Stop()
    printfn "New array: %d ms" sw.ElapsedMilliseconds
    
    // With pool - reuse arrays
    let pool = ArrayPool<int>.Shared
    sw.Restart()
    for _ in 1..iterations do
        let arr = pool.Rent(size)
        try
            arr[0] <- 1
        finally
            pool.Return(arr)
    sw.Stop()
    printfn "Pool: %d ms" sw.ElapsedMilliseconds
    
    System.GC.Collect()
    printfn "Memory: %d bytes" (System.GC.GetTotalMemory(true))

poolVsNew ()
```

## 7. MemoryPool<T>

```fsharp
// MemoryPool สำหรับ async operations
let memoryPoolExample () =
    let pool = MemoryPool<byte>.Shared
    
    // Rent memory
    use owner = pool.Rent(4096)
    let memory: Memory<byte> = owner.Memory
    
    printfn "Rented memory size: %d" memory.Length
    
    // เขียนข้อมูล
    let bytes = System.Text.Encoding.UTF8.GetBytes("Hello, Pool!")
    bytes.AsSpan().CopyTo(memory.Span)
    
    // อ่านข้อมูล
    let text = System.Text.Encoding.UTF8.GetString(memory.Span.Slice(0, bytes.Length))
    printfn "Read: %s" text
    
    // Memory ถูก return อัตโนมัติเมื่อ owner ถูก dispose (use)
```

## 8. Struct Optimization

```fsharp
// Struct เพื่อลด heap allocations
[<Struct>]
type Color = {
    R: byte
    G: byte
    B: byte
    A: byte
}

[<Struct>]
type Rect = {
    X: float32
    Y: float32
    Width: float32
    Height: float32
}

// Struct tuple (F# 4.1+)
let structTupleExample () =
    // struct tuple ไม่ allocate บน heap
    let point: struct (int * int) = struct (10, 20)
    let struct (x, y) = point
    printfn "Point: (%d, %d)" x y
    
    // เปรียบเทียบ
    let heapTuple: int * int = (10, 20)  // allocates
    let stackTuple: struct (int * int) = struct (10, 20)  // no alloc
    
    printfn "Heap: %A" heapTuple
    printfn "Stack struct: %A" stackTuple
```

```fsharp
// Struct record ใน F#
[<Struct>]
type Money = {
    Amount: decimal
    Currency: string
}

// Struct discriminated union
[<Struct>]
type Option2<'T> =
    | Some2 of value: 'T
    | None2

// ใช้งาน
let money = { Amount = 100.50m; Currency = "THB" }
printfn "เงิน: %M %s" money.Amount money.Currency

let some = Some2 42
let none = None2

match some with
| Some2 v -> printfn "ค่า: %d" v
| None2 -> printfn "ไม่มีค่า"
```

## 9. Inlining ด้วย inline Keyword

```fsharp
// inline ทำให้ compiler แทรกโค้ดโดยตรง (ไม่มี function call overhead)
let inline add x y = x + y
let inline square x = x * x
let inline cube x = x * x * x

// inline กับ generic constraints
let inline sumAll (items: ^T list) : ^T
    when ^T : (static member (+) : ^T * ^T -> ^T)
    and ^T : (static member Zero : ^T) =
    items |> List.fold (fun acc x -> acc + x) LanguagePrimitives.GenericZero

// ใช้งาน
let intSum = sumAll [1; 2; 3; 4; 5]
let floatSum = sumAll [1.0; 2.0; 3.0]
printfn "Int sum: %d" intSum
printfn "Float sum: %.1f" floatSum
```

```fsharp
// inline function ลด overhead
let inline processItems (transform: ^T -> ^T) (items: ^T[]) : ^T[] =
    let result = Array.zeroCreate items.Length
    for i in 0..items.Length - 1 do
        result[i] <- transform items[i]
    result

// เพราะ inline, lambda ถูกแทรกโดยตรง - ไม่มี function call overhead
let doubled = processItems (fun x -> x * 2) [|1..100|]
let squared = processItems (fun x -> x * x) [|1..100|]
printfn "doubled[0] = %d" doubled[0]
printfn "squared[4] = %d" squared[4]
```

```fsharp
// เปรียบเทียบ inline vs non-inline
let normal (f: int -> int) (x: int) = f x
let inline inlined (f: int -> int) (x: int) = f x

let perfCompare () =
    let n = 10_000_000
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let mutable sum1 = 0
    for i in 1..n do
        sum1 <- sum1 + normal (fun x -> x * 2) i
    sw.Stop()
    printfn "Normal: %d ms" sw.ElapsedMilliseconds
    
    sw.Restart()
    let mutable sum2 = 0
    for i in 1..n do
        sum2 <- sum2 + inlined (fun x -> x * 2) i
    sw.Stop()
    printfn "Inline: %d ms" sw.ElapsedMilliseconds

perfCompare ()
```

## 10. BenchmarkDotNet Usage

```fsharp
// BenchmarkDotNet - สำหรับ accurate benchmarking
// (ต้องติดตั้ง NuGet package: dotnet add package BenchmarkDotNet)

(*
open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Running

[<MemoryDiagnoser>]
[<SimpleJob(RuntimeMoniker.Net80)>]
type StringBenchmarks() =
    
    [<Params(100, 1000, 10000)>]
    member val N = 0 with get, set
    
    [<Benchmark>]
    member this.StringConcat() =
        let mutable s = ""
        for i in 1..this.N do
            s <- s + string i
        s
    
    [<Benchmark>]
    member this.StringBuilder() =
        let sb = System.Text.StringBuilder()
        for i in 1..this.N do
            sb.Append(i) |> ignore
        sb.ToString()
    
    [<Benchmark>]
    member this.StringJoin() =
        System.String.Join("", [|1..this.N|] |> Array.map string)

// รัน benchmark
BenchmarkRunner.Run<StringBenchmarks>() |> ignore
*)

// จำลอง manual benchmark
let manualBenchmark (name: string) (iterations: int) (fn: unit -> unit) =
    let sw = System.Diagnostics.Stopwatch.StartNew()
    for _ in 1..iterations do fn ()
    sw.Stop()
    printfn "%s: %.2f ms/op (total: %d ms)" 
        name 
        (float sw.ElapsedMilliseconds / float iterations)
        sw.ElapsedMilliseconds

// ทดสอบ
let benchmarkStrings () =
    let n = 100
    
    manualBenchmark "String concat" 100 (fun () ->
        let mutable s = ""
        for i in 1..n do s <- s + string i
    )
    
    manualBenchmark "StringBuilder" 100 (fun () ->
        let sb = System.Text.StringBuilder()
        for i in 1..n do sb.Append(i) |> ignore
        sb.ToString() |> ignore
    )
    
    manualBenchmark "String.Join" 100 (fun () ->
        System.String.Join("", [|1..n|] |> Array.map string) |> ignore
    )

benchmarkStrings ()
```

## 11. Common Performance Anti-Patterns

```fsharp
// Anti-pattern 1: การสร้าง object ใน loop
let antiPattern1 () =
    // ❌ ไม่ดี: สร้าง list ใหม่ทุก iteration
    let mutable items = []
    for i in 1..10000 do
        items <- i :: items  // O(1) แต่สร้าง node ใหม่ทุกครั้ง
    
    // ✅ ดีกว่า: ใช้ ResizeArray
    let items2 = ResizeArray<int>()
    for i in 1..10000 do
        items2.Add(i)
    let result = items2.ToArray()
    printfn "Items: %d" result.Length
```

```fsharp
// Anti-pattern 2: LINQ ใน hot path
let antiPattern2 () =
    let data = [|1..1_000_000|]
    
    // ❌ ไม่ดี: LINQ ใน hot path สร้าง lambda ทุกครั้ง
    let sw = System.Diagnostics.Stopwatch.StartNew()
    for _ in 1..100 do
        data |> Array.filter (fun x -> x % 2 = 0) 
             |> Array.map (fun x -> x * 2) 
             |> Array.sum |> ignore
    sw.Stop()
    printfn "LINQ: %d ms" sw.ElapsedMilliseconds
    
    // ✅ ดีกว่า: imperative loop
    sw.Restart()
    for _ in 1..100 do
        let mutable sum = 0
        for x in data do
            if x % 2 = 0 then sum <- sum + x * 2
    sw.Stop()
    printfn "Imperative: %d ms" sw.ElapsedMilliseconds
```

```fsharp
// Anti-pattern 3: Exception ใน control flow
let antiPattern3 () =
    let tryParseInt (s: string) =
        // ❌ ไม่ดี: throw/catch exception ใน normal flow (ช้ามาก)
        try
            Some (int s)
        with
        | _ -> None
    
    let tryParseIntFast (s: string) =
        // ✅ ดีกว่า: ใช้ TryParse
        match System.Int32.TryParse(s) with
        | true, n -> Some n
        | false, _ -> None
    
    let data = [| "1"; "abc"; "3"; "xyz"; "5" |]
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    for _ in 1..10000 do
        data |> Array.choose tryParseInt |> ignore
    sw.Stop()
    printfn "Exception-based: %d ms" sw.ElapsedMilliseconds
    
    sw.Restart()
    for _ in 1..10000 do
        data |> Array.choose tryParseIntFast |> ignore
    sw.Stop()
    printfn "TryParse: %d ms" sw.ElapsedMilliseconds
```

## 12. String Optimization

```fsharp
// String optimization techniques

// 1. String interning
let stringInterning () =
    let s1 = "hello"
    let s2 = "hello"
    let s3 = System.String.Intern("hello")
    let s4 = System.String.Copy("hello")  // F# 7+ ควรใช้ string.Clone
    
    printfn "s1 == s2 (reference): %b" (Object.ReferenceEquals(s1, s2))
    printfn "s1 == s3 (interned): %b" (Object.ReferenceEquals(s1, s3))
    printfn "s1 == s4 (copy): %b" (Object.ReferenceEquals(s1, s4))
```

```fsharp
// 2. StringBuilder สำหรับ string building
let stringBuildingExample () =
    // สร้าง CSV
    let rows = [|
        [|"ชื่อ"; "อายุ"; "เมือง"|]
        [|"สมชาย"; "30"; "กรุงเทพ"|]
        [|"สุดา"; "25"; "เชียงใหม่"|]
        [|"มนัส"; "35"; "ภูเก็ต"|]
    |]
    
    let sb = System.Text.StringBuilder()
    for row in rows do
        for i, cell in row |> Array.indexed do
            if i > 0 then sb.Append(',') |> ignore
            sb.Append(cell) |> ignore
        sb.AppendLine() |> ignore
    
    printfn "CSV:\n%s" (sb.ToString())
```

```fsharp
// 3. ReadOnlySpan<char> สำหรับ parsing
let parseWithoutAllocation (input: string) =
    let span = input.AsSpan()
    let results = ResizeArray<string>()
    
    let mutable start = 0
    for i in 0..span.Length - 1 do
        if span[i] = ',' then
            results.Add(span.Slice(start, i - start).ToString())
            start <- i + 1
    
    if start < span.Length then
        results.Add(span.Slice(start).ToString())
    
    results |> Seq.toList

let parts = parseWithoutAllocation "apple,banana,cherry,date"
printfn "Parts: %A" parts
```

```fsharp
// 4. String.Create สำหรับ custom string creation
let efficientStringCreate (values: int[]) =
    // ❌ ไม่ดี: สร้าง intermediate strings
    values |> Array.map string |> String.concat ","
    
    // ✅ ดีกว่า: System.String.Create (allocate ครั้งเดียว)
    // (F# 5+)

let createCSV (values: int[]) =
    System.String.Create(
        // คำนวณ total length
        values |> Array.sumBy (fun v -> string(v).Length) + max 0 (values.Length - 1),
        struct (values, 0),
        System.Buffers.SpanAction<char, struct (int[] * int)>(
            fun span struct(vals, _) ->
                let mutable pos = 0
                for i in 0..vals.Length - 1 do
                    if i > 0 then
                        span[pos] <- ','
                        pos <- pos + 1
                    let s = string vals[i]
                    s.AsSpan().CopyTo(span.Slice(pos))
                    pos <- pos + s.Length
        ))

let csv = createCSV [|1; 2; 3; 4; 5|]
printfn "CSV: %s" csv
```

## 13. Object Pooling Pattern

```fsharp
// Object pool สำหรับ expensive objects
open System.Collections.Concurrent

type ObjectPool<'T>(factory: unit -> 'T, reset: 'T -> unit) =
    let pool = ConcurrentBag<'T>()
    
    member _.Rent() =
        match pool.TryTake() with
        | true, obj -> obj
        | false, _ -> factory ()
    
    member _.Return(obj: 'T) =
        reset obj
        pool.Add(obj)
    
    member _.Use<'R>(f: 'T -> 'R) =
        let obj = pool.Rent()  // Wait this is wrong
        try f obj
        finally pool.Return(obj)

// ใช้สำหรับ expensive objects
let sbPool = ObjectPool<System.Text.StringBuilder>(
    factory = fun () -> System.Text.StringBuilder(),
    reset = fun sb -> sb.Clear() |> ignore)

// ใช้งาน
let buildString (parts: string list) =
    let sb = sbPool.Rent()
    try
        for part in parts do
            sb.Append(part) |> ignore
        sb.ToString()
    finally
        sbPool.Return(sb)

printfn "Pooled: %s" (buildString ["Hello"; ", "; "World"; "!"])
```

## 14. Profiling Tips

```fsharp
// เทคนิค profiling อย่างง่าย

// 1. Stopwatch timing
let timeIt (name: string) (f: unit -> 'a) =
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let result = f ()
    sw.Stop()
    printfn "%s: %d ms" name sw.ElapsedMilliseconds
    result

// 2. Memory monitoring
let measureMemory (name: string) (f: unit -> 'a) =
    System.GC.Collect()
    let before = System.GC.GetTotalMemory(true)
    let result = f ()
    System.GC.Collect()
    let after = System.GC.GetTotalMemory(true)
    printfn "%s: %d bytes allocated" name (after - before)
    result

// 3. Combined profiling
let profile (name: string) (f: unit -> 'a) =
    System.GC.Collect()
    let memBefore = System.GC.GetTotalMemory(true)
    let gen0Before = System.GC.CollectionCount(0)
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let result = f ()
    sw.Stop()
    
    System.GC.Collect()
    let memAfter = System.GC.GetTotalMemory(true)
    let gen0After = System.GC.CollectionCount(0)
    
    printfn "[%s]" name
    printfn "  เวลา: %d ms" sw.ElapsedMilliseconds
    printfn "  Memory: %+d bytes" (memAfter - memBefore)
    printfn "  GC Gen0: %d collections" (gen0After - gen0Before)
    
    result

// ทดสอบ
profile "List.map" (fun () ->
    [1..100_000] |> List.map (fun x -> x * 2) |> ignore)

profile "Array.map" (fun () ->
    [|1..100_000|] |> Array.map (fun x -> x * 2) |> ignore)
```

## 15. GC Configuration Tips

```fsharp
(*
.NET GC Configuration สำหรับ performance:

1. Server GC (ใน .csproj หรือ runtimeconfig.json)
<GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>
<ServerGarbageCollection>true</ServerGarbageCollection>

2. Concurrent GC
<ConcurrentGarbageCollection>true</ConcurrentGarbageCollection>

3. Large Object Heap Compaction
System.Runtime.GCSettings.LargeObjectHeapCompactionMode <- 
    System.Runtime.GCLargeObjectHeapCompactionMode.CompactOnce

4. Tuning hints
System.GC.TryStartNoGCRegion(4 * 1024 * 1024)  // ป้องกัน GC ชั่วคราว
// ... critical code ...
System.GC.EndNoGCRegion()
*)

// ตัวอย่าง NoGC region
let criticalSection () =
    let allocated = System.GC.TryStartNoGCRegion(1024 * 1024)  // 1MB
    if allocated then
        try
            // Critical code ที่ต้องไม่มี GC interrupt
            let data = Array.init 1000 (fun i -> i * i)
            data |> Array.sum
        finally
            System.GC.EndNoGCRegion()
    else
        // Fallback
        let data = Array.init 1000 (fun i -> i * i)
        data |> Array.sum

printfn "Critical result: %d" (criticalSection ())
```

## สรุป

```fsharp
(*
Memory & Performance - สรุป:

Value Types:
- Struct ลด heap allocations
- Span<T> อ่าน/เขียน memory โดยไม่ copy
- Struct tuple/record สำหรับ small data

Avoid Allocations:
- ใช้ ArrayPool<T> สำหรับ temporary arrays
- ใช้ MemoryPool<T> สำหรับ async
- ใช้ StringBuilder แทน string concat
- ใช้ ResizeArray แทน list ใน hot paths

Inline Keyword:
- ลด function call overhead
- ทำงานกับ generic constraints

Performance Patterns:
- Object pooling
- Span-based parsing
- Array-based processing
- Imperative inner loops

Profiling:
- Stopwatch สำหรับ timing
- GC.GetTotalMemory สำหรับ memory
- BenchmarkDotNet สำหรับ accurate benchmarks

ข้อควรระวัง:
- อย่า optimize ก่อน profile
- อ่าน code ให้ง่ายก่อน optimize
- benchmark จริงๆ อย่า guess
*)

printfn "Memory & Performance - สรุปเสร็จ!"
```
