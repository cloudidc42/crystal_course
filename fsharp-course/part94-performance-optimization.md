# Part 94 - การเพิ่มประสิทธิภาพ (Performance Optimization)

## บทนำ

การเพิ่มประสิทธิภาพ (Performance Optimization) เป็นทักษะสำคัญสำหรับนักพัฒนา F# ที่ต้องการสร้าง applications ที่รวดเร็วและมีประสิทธิภาพ บทนี้จะครอบคลุมเทคนิคต่างๆ ตั้งแต่การ profiling ไปจนถึง low-level optimizations

---

## 1. Profiling Tools

```fsharp
// Profiling ใน F# ทำได้หลายวิธี:
// 1. BenchmarkDotNet - สำหรับ microbenchmarks
// 2. dotnet-trace / dotnet-counters - สำหรับ production profiling
// 3. Visual Studio Profiler
// 4. JetBrains dotMemory / dotTrace

// ติดตั้ง BenchmarkDotNet
// <PackageReference Include="BenchmarkDotNet" Version="0.13.0" />

open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Running

[<MemoryDiagnoser>]
[<RankColumn>]
type StringConcatBenchmarks() =
    
    [<Params(10, 100, 1000)>]
    member val N = 0 with get, set
    
    // Method 1: + operator (สร้าง allocations มาก)
    [<Benchmark(Baseline = true)>]
    member this.StringConcat() =
        let mutable result = ""
        for i in 1..this.N do
            result <- result + string i
        result
    
    // Method 2: String.concat
    [<Benchmark>]
    member this.StringJoin() =
        [1..this.N] 
        |> List.map string 
        |> String.concat ""
    
    // Method 3: StringBuilder (เร็วที่สุดสำหรับ many concatenations)
    [<Benchmark>]
    member this.StringBuilder() =
        let sb = System.Text.StringBuilder()
        for i in 1..this.N do
            sb.Append(string i) |> ignore
        sb.ToString()
    
    // Method 4: sprintf ใน list
    [<Benchmark>]
    member this.SprintfList() =
        Array.init this.N (fun i -> sprintf "%d" i)
        |> Array.reduce (+)

// เรียกใช้ benchmark
// [<EntryPoint>]
// let main _ =
//     BenchmarkRunner.Run<StringConcatBenchmarks>() |> ignore
//     0

// Simple timing สำหรับ quick tests
let time label f =
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let result = f()
    sw.Stop()
    printfn "%s: %d ms" label sw.ElapsedMilliseconds
    result

// ใช้งาน
let _ = time "sorting 1M items" (fun () ->
    let arr = Array.init 1_000_000 (fun _ -> System.Random().Next())
    Array.sort arr
    arr)
```

---

## 2. Hot Path Optimization

```fsharp
// Hot path = code ที่ถูกเรียกบ่อยที่สุด
// ต้องระวังเป็นพิเศษกับ allocations และ branches

// Bad: allocations ใน hot path
let badHotPath (data: int[]) =
    data
    |> Array.map (fun x -> x * 2)  // creates new array
    |> Array.filter (fun x -> x > 10)  // creates new array
    |> Array.sum  // final computation

// Good: ใช้ imperative loop เมื่อ performance critical
let goodHotPath (data: int[]) =
    let mutable sum = 0
    for i in 0..data.Length-1 do
        let doubled = data.[i] * 2
        if doubled > 10 then
            sum <- sum + doubled
    sum

// ตัวอย่าง: Optimized inner loop
let processLargeDataset (data: float[]) =
    // ใช้ Array.Parallel สำหรับ CPU-bound work
    let chunks = 
        data
        |> Array.chunkBySize (data.Length / System.Environment.ProcessorCount)
    
    chunks
    |> Array.Parallel.map (fun chunk ->
        let mutable sum = 0.0
        let mutable sumSq = 0.0
        let mutable count = 0
        
        for x in chunk do
            sum <- sum + x
            sumSq <- sumSq + x * x
            count <- count + 1
        
        (sum, sumSq, count))
    |> Array.reduce (fun (s1, sq1, c1) (s2, sq2, c2) ->
        (s1 + s2, sq1 + sq2, c1 + c2))
    |> fun (sum, sumSq, count) ->
        let mean = sum / float count
        let variance = sumSq / float count - mean * mean
        {| Mean = mean; StdDev = sqrt variance; Count = count |}

// Tail recursion optimization
let rec sumTailRec acc = function
    | [] -> acc
    | x :: xs -> sumTailRec (acc + x) xs  // tail call

// Non-tail recursive (stack overflow for large lists)
let rec sumNonTail = function
    | [] -> 0
    | x :: xs -> x + sumNonTail xs  // NOT tail call - x + ... prevents TCO

// Inline สำหรับ small frequently-called functions
[<Inline>]
let inline square x = x * x

[<Inline>]
let inline clamp min max value =
    if value < min then min
    elif value > max then max
    else value
```

---

## 3. Avoiding Allocations

```fsharp
// Heap allocations ช้ากว่า stack allocations มาก
// เป้าหมาย: ลด allocations ใน hot path

// 1. ใช้ structs แทน classes
[<Struct>]
type Point2D = { X: float32; Y: float32 }

[<Struct>]
type Color = { R: byte; G: byte; B: byte; A: byte }

// 2. ใช้ ValueOption แทน Option ใน hot path
let findInArrayOpt (arr: int[]) target : int voption =
    let mutable i = 0
    let mutable found = ValueNone
    while i < arr.Length && found = ValueNone do
        if arr.[i] = target then
            found <- ValueSome i
        i <- i + 1
    found

// 3. ใช้ byref parameters เพื่อหลีกเลี่ยง boxing
let inline addToSum (sum: byref<int>) value =
    sum <- sum + value

// ใช้งาน
let computeSum (arr: int[]) =
    let mutable total = 0
    for x in arr do
        addToSum &total x
    total

// 4. Avoid closure allocations ใน loops
// Bad: สร้าง closure ใน loop
let badLoopClosures (data: int[]) =
    data
    |> Array.mapi (fun i x -> (i, x))  // creates (int * int) tuples = allocations
    |> Array.filter (fun (_, x) -> x > 0)  // more allocations

// Good: ใช้ mutable state
let goodLoopNoAlloc (data: int[]) =
    let mutable count = 0
    let mutable sum = 0
    for x in data do
        if x > 0 then
            count <- count + 1
            sum <- sum + x
    (count, sum)

// 5. Reuse buffers สำหรับ frequent operations
module BufferPool =
    open System.Buffers
    
    // ใช้ ArrayPool เพื่อ reuse arrays
    let processWithPool (data: byte[]) =
        let buffer = ArrayPool<byte>.Shared.Rent(data.Length)
        try
            System.Array.Copy(data, buffer, data.Length)
            // process buffer...
            let result = buffer |> Array.sum
            result
        finally
            ArrayPool<byte>.Shared.Return(buffer, clearArray = true)

// 6. String interning สำหรับ repeated strings
let inline intern (s: string) = System.String.Intern(s)

// สำหรับ parsing: ใช้ ReadOnlySpan<char>
let parseIntFromSpan (span: System.ReadOnlySpan<char>) =
    let mutable result = 0
    let mutable i = 0
    while i < span.Length do
        result <- result * 10 + (int span.[i] - int '0')
        i <- i + 1
    result
```

---

## 4. Struct Optimization

```fsharp
// Structs มี value semantics และ stack allocation
// เหมาะสำหรับ small, frequently-created objects

// กฎ: ใช้ struct เมื่อ
// 1. Size <= 16 bytes (rule of thumb)
// 2. Short-lived (ไม่ boxed บ่อย)
// 3. Immutable หรือ rarely mutated
// 4. ไม่ implement interfaces มาก

[<Struct>]
type Complex = 
    val Real: float
    val Imag: float
    
    new(r, i) = { Real = r; Imag = i }
    
    static member (+) (a: Complex, b: Complex) = 
        Complex(a.Real + b.Real, a.Imag + b.Imag)
    
    static member (*) (a: Complex, b: Complex) =
        Complex(
            a.Real * b.Real - a.Imag * b.Imag,
            a.Real * b.Imag + a.Imag * b.Real)
    
    member this.Magnitude = 
        sqrt (this.Real * this.Real + this.Imag * this.Imag)

// Struct with mutable fields (careful with copying!)
[<Struct>]
type MutablePoint =
    val mutable X: float
    val mutable Y: float
    
    new(x, y) = { X = x; Y = y }
    
    member this.Move(dx, dy) =
        // สร้าง new struct (value semantics)
        MutablePoint(this.X + dx, this.Y + dy)

// ข้อระวัง: struct copying
let testStructCopy () =
    let p1 = MutablePoint(1.0, 2.0)
    let mutable p2 = p1  // copy!
    p2.X <- 10.0
    
    printfn "p1.X = %f" p1.X  // 1.0 (ไม่ถูก modify)
    printfn "p2.X = %f" p2.X  // 10.0

// StructuralEquality สำหรับ structs
[<Struct; CustomEquality; CustomComparison>]
type Temperature =
    val Celsius: float
    new(c) = { Celsius = c }
    
    member this.Fahrenheit = this.Celsius * 9.0 / 5.0 + 32.0
    
    interface System.IEquatable<Temperature> with
        member this.Equals(other) = this.Celsius = other.Celsius
    
    interface System.IComparable<Temperature> with
        member this.CompareTo(other) = compare this.Celsius other.Celsius
    
    override this.Equals(obj) =
        match obj with
        | :? Temperature as other -> this.Celsius = other.Celsius
        | _ -> false
    
    override this.GetHashCode() = hash this.Celsius

// Array of structs vs Array of references
let benchmarkStructVsClass () =
    let n = 1_000_000
    
    // Struct array: elements laid out contiguously in memory
    let structArray = Array.init n (fun i -> Complex(float i, float i * 2.0))
    
    // Better cache performance for struct arrays
    let sumMagnitudes = structArray |> Array.sumBy (fun c -> c.Magnitude)
    sumMagnitudes
```

---

## 5. Span<T> สำหรับ Zero-Copy Operations

```fsharp
open System
open System.Runtime.InteropServices

// Span<T> ช่วยให้ทำงานกับ memory blocks โดยไม่ต้องสร้าง copies

// ตัวอย่าง: parsing ข้อมูลจาก byte array
let parseHeader (data: ReadOnlySpan<byte>) =
    if data.Length < 8 then Error "Too short"
    else
        let magic = data.Slice(0, 4)
        let version = data.Slice(4, 2)
        let flags = data.Slice(6, 2)
        
        let magicStr = System.Text.Encoding.ASCII.GetString(magic)
        let versionNum = MemoryMarshal.Read<uint16>(version)
        let flagsNum = MemoryMarshal.Read<uint16>(flags)
        
        Ok {| Magic = magicStr; Version = versionNum; Flags = flagsNum |}

// String slicing with Span (no allocations!)
let countWords (text: ReadOnlySpan<char>) =
    let mutable count = 0
    let mutable inWord = false
    
    for i in 0..text.Length-1 do
        let c = text.[i]
        if Char.IsWhiteSpace(c) then
            if inWord then
                count <- count + 1
                inWord <- false
        else
            inWord <- true
    
    if inWord then count + 1 else count

let countWordsStr (s: string) =
    countWords (s.AsSpan())

// Memory<T> สำหรับ async contexts (Span ใช้ใน async ไม่ได้)
let processChunksAsync (data: Memory<byte>) = async {
    let mutable offset = 0
    let chunkSize = 1024
    
    while offset < data.Length do
        let length = min chunkSize (data.Length - offset)
        let chunk = data.Slice(offset, length)
        
        // Process chunk (ใช้ Span ใน sync context)
        let span = chunk.Span
        // ... process span
        
        offset <- offset + length
}

// StackAlloc equivalent (unsafe)
// ใน F# ใช้ stackalloc ผ่าน NativePtr
open Microsoft.FSharp.NativeInterop

let computeHashFast (data: byte[]) =
    let bufferSize = 32
    // allocate on stack (no GC pressure)
    let buffer : nativeptr<byte> = NativePtr.stackalloc bufferSize
    
    // ใช้ buffer...
    let mutable hash = 0UL
    for b in data do
        hash <- hash ^^^ (uint64 b)
    hash

// Span สำหรับ efficient CSV parsing
let parseCsvLine (line: ReadOnlySpan<char>) =
    let results = System.Collections.Generic.List<string>()
    let mutable start = 0
    let mutable i = 0
    
    while i < line.Length do
        if line.[i] = ',' then
            results.Add(string (line.Slice(start, i - start)))
            start <- i + 1
        i <- i + 1
    
    // Add last field
    results.Add(string (line.Slice(start, line.Length - start)))
    results.ToArray()
```

---

## 6. SIMD กับ Vector<T>

```fsharp
open System.Numerics
open System.Runtime.Intrinsics
open System.Runtime.Intrinsics.X86

// SIMD (Single Instruction Multiple Data) ช่วยให้ทำงานกับข้อมูลหลายๆ ตัวพร้อมกัน

// ตัวอย่าง: Vector addition
let addArraysSIMD (a: float32[]) (b: float32[]) =
    let result = Array.zeroCreate a.Length
    let vecSize = Vector<float32>.Count
    let mutable i = 0
    
    // Process in SIMD chunks
    while i <= a.Length - vecSize do
        let va = Vector<float32>(a, i)
        let vb = Vector<float32>(b, i)
        let vc = va + vb
        vc.CopyTo(result, i)
        i <- i + vecSize
    
    // Handle remaining elements
    while i < a.Length do
        result.[i] <- a.[i] + b.[i]
        i <- i + 1
    
    result

// Dot product with SIMD
let dotProductSIMD (a: float32[]) (b: float32[]) =
    if a.Length <> b.Length then failwith "Arrays must have same length"
    
    let vecSize = Vector<float32>.Count
    let mutable sumVec = Vector<float32>.Zero
    let mutable i = 0
    
    while i <= a.Length - vecSize do
        let va = Vector<float32>(a, i)
        let vb = Vector<float32>(b, i)
        sumVec <- sumVec + (va * vb)
        i <- i + vecSize
    
    let mutable sum = Vector.Dot(sumVec, Vector<float32>.One)
    
    while i < a.Length do
        sum <- sum + a.[i] * b.[i]
        i <- i + 1
    
    sum

// Matrix multiplication with SIMD
let matMulSIMD (a: float32[,]) (b: float32[,]) =
    let m = Array2D.length1 a
    let n = Array2D.length2 b
    let k = Array2D.length2 a
    
    let result = Array2D.zeroCreate m n
    
    for i in 0..m-1 do
        for j in 0..n-1 do
            let mutable sum = 0.0f
            for l in 0..k-1 do
                sum <- sum + a.[i, l] * b.[l, j]
            result.[i, j] <- sum
    
    result

// ตัวอย่างการใช้ Hardware Intrinsics
let checkSIMDSupport () =
    printfn "SSE2: %b" Sse2.IsSupported
    printfn "AVX: %b" Avx.IsSupported
    printfn "AVX2: %b" Avx2.IsSupported
    printfn "Vector<float> size: %d elements" Vector<float>.Count
    printfn "Vector<float32> size: %d elements" Vector<float32>.Count
    printfn "Vector<int> size: %d elements" Vector<int>.Count
```

---

## 7. Unsafe Code ใน F#

```fsharp
open Microsoft.FSharp.NativeInterop
open System.Runtime.CompilerServices

// Unsafe code ช่วยให้ทำงานกับ memory โดยตรง
// ใช้เมื่อ performance สำคัญมากๆ เท่านั้น

// ยก คำเตือน: unsafe code อาจทำให้ memory corruption และ security issues

// NativePtr operations
let sumArrayUnsafe (arr: int[]) =
    let mutable sum = 0
    use pinned = fixed arr  // Pin array in memory
    let ptr : nativeptr<int> = pinned
    
    for i in 0..arr.Length-1 do
        sum <- sum + NativePtr.get ptr i
    
    sum

// Unsafe bitwise operations
let inline bitsToFloat (bits: uint32) : float32 =
    Unsafe.As<uint32, float32>(&bits)  // reinterpret_cast

let inline floatToBits (value: float32) : uint32 =
    Unsafe.As<float32, uint32>(&value)

// Fast absolute value for float
let inline fastAbs (x: float32) : float32 =
    let bits = floatToBits x
    bitsToFloat (bits &&& 0x7FFFFFFFu)  // clear sign bit

// Fast inverse square root (famous Quake III algorithm)
let inline fastInvSqrt (x: float32) : float32 =
    let halfX = x * 0.5f
    let mutable y = x
    let i = Unsafe.As<float32, int32>(&y)
    let i2 = 0x5f3759df - (i >>> 1)
    y <- Unsafe.As<int32, float32>(&i2)
    y <- y * (1.5f - halfX * y * y)
    y

// Memory-efficient large array processing
let processLargeFile (path: string) =
    use mmf = System.IO.MemoryMappedFiles.MemoryMappedFile.CreateFromFile(path)
    use accessor = mmf.CreateViewAccessor()
    
    let size = accessor.Capacity
    let mutable i = 0L
    let mutable sum = 0L
    
    while i < size - 8L do
        let value = accessor.ReadInt64(i)
        sum <- sum + value
        i <- i + 8L
    
    sum
```

---

## 8. Memory-Mapped Files

```fsharp
open System.IO
open System.IO.MemoryMappedFiles
open System.Runtime.InteropServices

// Memory-mapped files สำหรับ large file processing

// Reading large binary file
let readLargeBinaryFile (path: string) =
    use mmf = MemoryMappedFile.CreateFromFile(path, FileMode.Open)
    use view = mmf.CreateViewStream()
    
    let buffer = Array.zeroCreate<byte> 8192
    let mutable bytesRead = view.Read(buffer, 0, buffer.Length)
    
    let mutable totalSum = 0L
    while bytesRead > 0 do
        for i in 0..bytesRead-1 do
            totalSum <- totalSum + int64 buffer.[i]
        bytesRead <- view.Read(buffer, 0, buffer.Length)
    
    totalSum

// Shared memory between processes
let createSharedMemory name size =
    MemoryMappedFile.CreateOrOpen(name, size)

let writeToSharedMemory (mmf: MemoryMappedFile) offset (data: byte[]) =
    use accessor = mmf.CreateViewAccessor(offset, data.Length)
    accessor.WriteArray(0L, data, 0, data.Length)

let readFromSharedMemory (mmf: MemoryMappedFile) offset size =
    let buffer = Array.zeroCreate<byte> size
    use accessor = mmf.CreateViewAccessor(offset, size)
    accessor.ReadArray(0L, buffer, 0, size)
    buffer

// High-performance log file processing
let countLinesMMF (path: string) =
    use mmf = MemoryMappedFile.CreateFromFile(path, FileMode.Open)
    let length = (new FileInfo(path)).Length
    use accessor = mmf.CreateViewAccessor(0L, length)
    
    let mutable count = 0
    let mutable i = 0L
    
    while i < length do
        if accessor.ReadByte(i) = byte '\n' then
            count <- count + 1
        i <- i + 1L
    
    count
```

---

## 9. Pooling Strategies

```fsharp
open System.Buffers
open System.Collections.Concurrent

// Object Pooling ช่วยลด GC pressure

// Simple Object Pool
type ObjectPool<'T when 'T : not struct>(factory: unit -> 'T, ?maxSize: int) =
    let pool = ConcurrentBag<'T>()
    let maxSize = defaultArg maxSize 1000
    
    member _.Get() =
        match pool.TryTake() with
        | true, obj -> obj
        | false, _ -> factory()
    
    member _.Return(obj: 'T) =
        if pool.Count < maxSize then
            pool.Add(obj)

// StringBuilder pool
let stringBuilderPool = 
    ObjectPool<System.Text.StringBuilder>(
        factory = fun () -> System.Text.StringBuilder(),
        maxSize = 50)

let buildStringEfficient items =
    let sb = stringBuilderPool.Get()
    try
        sb.Clear() |> ignore
        for item in items do
            sb.Append(item).Append(", ") |> ignore
        if sb.Length > 2 then
            sb.Remove(sb.Length - 2, 2) |> ignore
        sb.ToString()
    finally
        stringBuilderPool.Return(sb)

// ArrayPool for byte buffers
let processChunkWithPool (data: byte[]) =
    let buffer = ArrayPool<byte>.Shared.Rent(8192)
    try
        let mutable position = 0
        let mutable result = 0
        
        while position < data.Length do
            let bytesToCopy = min 8192 (data.Length - position)
            System.Array.Copy(data, position, buffer, 0, bytesToCopy)
            
            // process buffer[0..bytesToCopy-1]
            for i in 0..bytesToCopy-1 do
                result <- result + int buffer.[i]
            
            position <- position + bytesToCopy
        
        result
    finally
        ArrayPool<byte>.Shared.Return(buffer)

// Connection pool (สำหรับ database connections)
type ConnectionPool(connectionString: string, maxSize: int) =
    let pool = ConcurrentQueue<System.Data.SqlClient.SqlConnection>()
    let semaphore = new System.Threading.SemaphoreSlim(maxSize, maxSize)
    let mutable disposed = false
    
    let createConnection () =
        let conn = new System.Data.SqlClient.SqlConnection(connectionString)
        conn.Open()
        conn
    
    member _.GetAsync() = async {
        do! semaphore.WaitAsync() |> Async.AwaitTask
        return
            match pool.TryDequeue() with
            | true, conn ->
                if conn.State = System.Data.ConnectionState.Open then conn
                else createConnection()
            | false, _ -> createConnection()
    }
    
    member _.Return(conn: System.Data.SqlClient.SqlConnection) =
        if not disposed && conn.State = System.Data.ConnectionState.Open then
            pool.Enqueue(conn)
        else
            conn.Dispose()
        semaphore.Release() |> ignore
    
    interface System.IDisposable with
        member _.Dispose() =
            disposed <- true
            while not pool.IsEmpty do
                match pool.TryDequeue() with
                | true, conn -> conn.Dispose()
                | false, _ -> ()
            semaphore.Dispose()
```

---

## 10. Task vs Async Performance

```fsharp
open System.Threading.Tasks

// Task<T> vs Async<T>:
// Task: .NET native, zero-copy หากใช้ ValueTask
// Async: F# native, ดีกว่าสำหรับ cancellation/composition
// ValueTask: สำหรับ hot paths ที่ sync บ่อย

// Benchmark: Task vs Async
let benchmarkAsync () =
    // Async - overhead จาก F# async machinery
    let asyncWork () = async {
        let mutable sum = 0
        for i in 1..1000 do
            sum <- sum + i
        return sum
    }
    
    // Task - เร็วกว่าสำหรับ simple cases
    let taskWork () : Task<int> = task {
        let mutable sum = 0
        for i in 1..1000 do
            sum <- sum + i
        return sum
    }
    
    // ValueTask - ดีที่สุดเมื่อ result available synchronously
    let valueTaskWork () : ValueTask<int> =
        let sum = Seq.sum [1..1000]
        ValueTask<int>(sum)  // completed immediately

// ใช้ ConfigureAwait(false) เพื่อหลีกเลี่ยง context switching
let efficientAsync (url: string) = task {
    use client = new System.Net.Http.HttpClient()
    let! response = client.GetAsync(url)  // ConfigureAwait is handled by F# task CE
    let! content = response.Content.ReadAsStringAsync()
    return content
}

// Parallel async operations
let fetchAllParallel (urls: string list) = async {
    let! results =
        urls
        |> List.map (fun url -> async {
            use client = new System.Net.Http.HttpClient()
            return! client.GetStringAsync(url) |> Async.AwaitTask
        })
        |> Async.Parallel
    return results
}

// Sequential vs Parallel performance
let sequentialSum (items: int list) = async {
    let mutable total = 0
    for item in items do
        total <- total + item
    return total
}

let parallelSum (items: int[]) = async {
    let result =
        items
        |> Array.Parallel.map id
        |> Array.sum
    return result
}

// Using Tasks for max performance
let highPerformanceParallel (items: int[]) =
    let tasks = 
        items 
        |> Array.chunkBySize (max 1 (items.Length / System.Environment.ProcessorCount))
        |> Array.map (fun chunk ->
            Task.Run(fun () -> Array.sum chunk))
    
    Task.WhenAll(tasks)
    |> fun t -> t.Result
    |> Array.sum
```

---

## 11. Benchmarking Methodology

```fsharp
// Правильный подход к benchmarking

// 1. Warm up JIT
// 2. Multiple iterations
// 3. Measure median, not just mean
// 4. Avoid optimizations misleading the compiler

open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Running
open BenchmarkDotNet.Configs
open BenchmarkDotNet.Jobs

[<Config(typeof<FastBenchmarkConfig>)>]
[<MemoryDiagnoser>]
type ListVsArrayBenchmarks() =
    
    [<Params(100, 1000, 10000)>]
    member val Size = 0 with get, set
    
    [<Benchmark(Baseline = true)>]
    member this.ListMap() =
        List.init this.Size id
        |> List.map (fun x -> x * 2)
        |> List.sum
    
    [<Benchmark>]
    member this.ArrayMap() =
        Array.init this.Size id
        |> Array.map (fun x -> x * 2)
        |> Array.sum
    
    [<Benchmark>]
    member this.ArrayFold() =
        let arr = Array.init this.Size id
        let mutable sum = 0
        for x in arr do
            sum <- sum + x * 2
        sum
    
    [<Benchmark>]
    member this.SeqMap() =
        Seq.init this.Size id
        |> Seq.map (fun x -> x * 2)
        |> Seq.sum

type FastBenchmarkConfig() =
    inherit ManualConfig()
    do
        base.AddJob(Job.ShortRun) |> ignore

// Manual microbenchmark (when BenchmarkDotNet not available)
let manualBenchmark iterations warmup name f =
    // Warmup
    for _ in 1..warmup do
        f() |> ignore
    
    System.GC.Collect()
    System.GC.WaitForPendingFinalizers()
    System.GC.Collect()
    
    let times = Array.zeroCreate iterations
    for i in 0..iterations-1 do
        let sw = System.Diagnostics.Stopwatch.StartNew()
        f() |> ignore
        sw.Stop()
        times.[i] <- sw.Elapsed.TotalMilliseconds
    
    Array.sort times
    let median = times.[iterations / 2]
    let mean = Array.average times
    let p95 = times.[int (float iterations * 0.95)]
    
    printfn "\n%s" name
    printfn "  Iterations: %d" iterations
    printfn "  Median: %.3f ms" median
    printfn "  Mean: %.3f ms" mean
    printfn "  P95: %.3f ms" p95
    printfn "  Min: %.3f ms" times.[0]
    printfn "  Max: %.3f ms" times.[iterations-1]
```

---

## 12. Common F# Performance Pitfalls

```fsharp
// 1. Boxing/Unboxing ใน generic functions
// Bad: เกิด boxing
let boxingExample (x: 'a) = 
    let obj : obj = box x  // boxing!
    unbox<'a> obj  // unboxing!

// Good: ใช้ inline และ static constraints
let inline noBoxing (x: ^a when ^a: (member Foo: int)) = 
    (^a: (member Foo: int) x)  // no boxing

// 2. List prepend vs append
// Good: prepend O(1)
let prepend x lst = x :: lst

// Bad: append O(n) 
let append lst x = lst @ [x]  // creates new list!

// 3. Seq ใน hot paths (lazy evaluation overhead)
// Bad: Seq chains have overhead
let badSeqChain (data: int[]) =
    data
    |> Seq.filter (fun x -> x > 0)
    |> Seq.map (fun x -> x * 2)
    |> Seq.take 100
    |> Seq.toArray

// Good: Array operations (eager, cache-friendly)
let goodArrayChain (data: int[]) =
    data
    |> Array.filter (fun x -> x > 0)
    |> Array.map (fun x -> x * 2)
    |> Array.truncate 100

// 4. Closure capture ทำให้เกิด heap allocations
let n = 10
let badClosure = Array.init 100 (fun i -> i + n)  // n captured in closure

// Better: pass explicitly
let goodNoCapture n = Array.init 100 (fun i -> i + n)

// 5. Record updates ใน tight loops
type Stats = { Count: int; Sum: float; Min: float; Max: float }

// Bad: creates new record every iteration
let updateStatsBad stats x =
    { stats with Count = stats.Count + 1; Sum = stats.Sum + x }

// Good: use mutable or struct
[<Struct>]
type MutableStats = 
    val mutable Count: int
    val mutable Sum: float

let updateStatsMutable (stats: byref<MutableStats>) x =
    stats.Count <- stats.Count + 1
    stats.Sum <- stats.Sum + x

// 6. Exception handling ใน hot paths
// Bad: exceptions สำหรับ control flow
let badParse s =
    try
        int s
    with _ -> 0

// Good: TryParse
let goodParse s =
    match System.Int32.TryParse(s) with
    | true, n -> n
    | false, _ -> 0

// 7. String formatting overhead
// Bad: sprintf เร็วกว่า string.Format แต่ยังมี overhead
let badFormat i j = sprintf "Item %d/%d" i j

// Good: interpolation หรือ StringBuilder
let goodFormat i j = $"Item {i}/{j}"
let betterFormat (sb: System.Text.StringBuilder) i j =
    sb.Clear().Append("Item ").Append(i).Append('/').Append(j).ToString()

// 8. Recursive functions ที่ไม่ใช่ tail recursive
// ตัวอย่าง Fibonacci: naive
let rec fibSlow n =
    if n <= 1 then n
    else fibSlow (n-1) + fibSlow (n-2)  // exponential time!

// ดีกว่า: memoized
let fibMemo =
    let cache = System.Collections.Generic.Dictionary<int, int64>()
    let rec fib n =
        match cache.TryGetValue(n) with
        | true, v -> v
        | false, _ ->
            let v = if n <= 1L then int64 n else fib (n-1) + fib (n-2)
            cache.[n] <- v
            v
    fib

// ดีที่สุด: iterative
let fibFast n =
    if n <= 1 then int64 n
    else
        let mutable a, b = 0L, 1L
        for _ in 2..n do
            let c = a + b
            a <- b
            b <- c
        b
```

---

## สรุป

Performance Optimization ใน F# ต้องพิจารณา:

1. **Profiling ก่อน optimize** - อย่า optimize สิ่งที่ไม่ใช่ bottleneck
2. **Reduce allocations** - ใช้ structs, span, pooling
3. **Cache-friendly data structures** - arrays สำหรับ sequential access
4. **Parallel processing** - ใช้ Task.WhenAll, Array.Parallel
5. **SIMD** - สำหรับ numerical computations
6. **Avoid common pitfalls** - boxing, closures, unnecessary copying

กฎ: **"Measure first, optimize second"** - อย่า premature optimize

---

*ต่อไป: Part 95 - Security กับ F#*
