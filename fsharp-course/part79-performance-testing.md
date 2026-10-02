# Part 79 - Performance Testing

## บทนำ (Introduction)

Performance Testing วัดว่า code ทำงานได้เร็วแค่ไหน และใช้ทรัพยากรเท่าไหร่ ใน F#/.NET เราใช้:

- **BenchmarkDotNet** - Micro-benchmarking
- **dotnet-trace** - CPU profiling
- **dotnet-counters** - Runtime metrics
- **NBomber** - Load testing

---

## 1. BenchmarkDotNet Setup

### การติดตั้ง

```xml
<!-- Benchmarks.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <!-- สำคัญ: Release mode เท่านั้น -->
    <Configuration>Release</Configuration>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BenchmarkDotNet" Version="0.13.10" />
    <PackageReference Include="BenchmarkDotNet.Diagnostics.Windows" Version="0.13.10" Condition="'$(OS)' == 'Windows_NT'" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="StringBenchmarks.fs" />
    <Compile Include="SortingBenchmarks.fs" />
    <Compile Include="CollectionBenchmarks.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

### Program.fs

```fsharp
// Program.fs
module Benchmarks.Program

open BenchmarkDotNet.Running

[<EntryPoint>]
let main argv =
    // รัน benchmark เฉพาะ
    BenchmarkRunner.Run<StringBenchmarks.StringBenchmarks>() |> ignore
    BenchmarkRunner.Run<SortingBenchmarks.SortingBenchmarks>() |> ignore
    0

// หรือรัน ทั้งหมดใน assembly
// BenchmarkSwitcher.FromAssembly(System.Reflection.Assembly.GetExecutingAssembly()).Run(argv) |> ignore
```

### รัน Benchmarks

```bash
# รัน benchmarks (ต้องเป็น Release mode!)
dotnet run -c Release

# รัน specific benchmark
dotnet run -c Release -- --filter "*StringBenchmarks*"

# Export results
dotnet run -c Release -- --exporters html csv json

# รัน quick benchmark (ลดจำนวน iterations)
dotnet run -c Release -- --job short
```

---

## 2. [<Benchmark>] Attribute

```fsharp
// StringBenchmarks.fs
module Benchmarks.StringBenchmarks

open BenchmarkDotNet.Attributes
open System.Text

// Basic [<Benchmark>] usage
[<MemoryDiagnoser>]
type StringBenchmarks() =
    
    [<Params(10, 100, 1000)>]
    member val N = 0 with get, set
    
    // Benchmark 1: String concatenation กับ +
    [<Benchmark(Baseline = true)>]
    member this.StringConcat() =
        let mutable result = ""
        for i in 1..this.N do
            result <- result + string i
        result
    
    // Benchmark 2: StringBuilder
    [<Benchmark>]
    member this.StringBuilder() =
        let sb = StringBuilder()
        for i in 1..this.N do
            sb.Append(string i) |> ignore
        sb.ToString()
    
    // Benchmark 3: String.Join
    [<Benchmark>]
    member this.StringJoin() =
        [1..this.N]
        |> List.map string
        |> String.concat ""
    
    // Benchmark 4: Array.join
    [<Benchmark>]
    member this.ArrayJoin() =
        [|1..this.N|]
        |> Array.map string
        |> String.concat ""
    
    // Benchmark 5: sprintf
    [<Benchmark>]
    member this.SprintfConcat() =
        let mutable result = ""
        for i in 1..this.N do
            result <- sprintf "%s%d" result i
        result

// Collection benchmarks
[<MemoryDiagnoser>]
[<RankColumn>]  // แสดง rank ของแต่ละ benchmark
type CollectionBenchmarks() =
    
    let data = Array.init 10000 id
    
    // Benchmark: Map vs Array.map
    [<Benchmark(Baseline = true)>]
    member _.ArrayMap() =
        data |> Array.map (fun x -> x * 2)
    
    [<Benchmark>]
    member _.ListMap() =
        data |> Array.toList |> List.map (fun x -> x * 2)
    
    [<Benchmark>]
    member _.SeqMap() =
        data |> Seq.map (fun x -> x * 2) |> Seq.toArray
    
    // Benchmark: Filter approaches
    [<Benchmark>]
    member _.ArrayFilter() =
        data |> Array.filter (fun x -> x % 2 = 0)
    
    [<Benchmark>]
    member _.ArrayChoose() =
        data |> Array.choose (fun x -> if x % 2 = 0 then Some x else None)
    
    // Benchmark: Sum approaches
    [<Benchmark>]
    member _.ArraySum() =
        Array.sum data
    
    [<Benchmark>]
    member _.ArrayFold() =
        data |> Array.fold (+) 0
    
    [<Benchmark>]
    member _.ArraySumBy() =
        data |> Array.sumBy id
```

---

## 3. Memory Diagnostics

```fsharp
// MemoryBenchmarks.fs
module Benchmarks.MemoryBenchmarks

open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Diagnosers

// [<MemoryDiagnoser>] วัด:
// - Gen0 collections
// - Gen1 collections  
// - Gen2 collections
// - Allocated bytes per operation

[<MemoryDiagnoser>]
[<DisassemblyDiagnoser(maxDepth = 3)>]  // Optional: แสดง Assembly
type AllocationBenchmarks() =
    
    let size = 1000
    
    // High allocation: สร้าง list ใหม่ทุกครั้ง
    [<Benchmark(Baseline = true)>]
    member _.ListCreation() =
        [for i in 1..size -> i]
    
    // Lower allocation: ใช้ array
    [<Benchmark>]
    member _.ArrayCreation() =
        [|for i in 1..size -> i|]
    
    // Lowest allocation: span/stackalloc (ไม่ allocate heap)
    [<Benchmark>]
    member _.SpanUsage() =
        let arr = System.ArrayPool<int>.Shared.Rent(size)
        try
            for i in 0..size-1 do
                arr.[i] <- i
            arr |> Array.sum  // Simplified
        finally
            System.ArrayPool<int>.Shared.Return(arr)
    
    // String allocation comparison
    [<Benchmark>]
    member _.StringInterpolation() =
        [for i in 1..100 -> sprintf "Item %d" i]
    
    [<Benchmark>]
    member _.StringFormat() =
        [for i in 1..100 -> System.String.Format("Item {0}", i)]
    
    [<Benchmark>]
    member _.StringInterpolationNew() =
        [for i in 1..100 -> $"Item {i}"]

// Record vs Class allocation
type MyRecord = { X: int; Y: int; Z: int }
type MyClass(x: int, y: int, z: int) =
    member _.X = x
    member _.Y = y
    member _.Z = z

[<MemoryDiagnoser>]
type RecordVsClassBenchmarks() =
    
    let n = 10000
    
    [<Benchmark(Baseline = true)>]
    member _.CreateRecords() =
        [|for i in 1..n -> { X = i; Y = i*2; Z = i*3 }|]
    
    [<Benchmark>]
    member _.CreateClasses() =
        [|for i in 1..n -> MyClass(i, i*2, i*3)|]
    
    [<Benchmark>]
    member _.CreateTuples() =
        [|for i in 1..n -> (i, i*2, i*3)|]
    
    [<Benchmark>]
    member _.CreateValueTuples() =
        [|for i in 1..n -> struct (i, i*2, i*3)|]
```

---

## 4. GC Pressure

```fsharp
// GCBenchmarks.fs
module Benchmarks.GCBenchmarks

open BenchmarkDotNet.Attributes

// วิธีลด GC pressure

[<MemoryDiagnoser>]
type GCPressureBenchmarks() =
    
    // High GC pressure: สร้าง objects มาก
    [<Benchmark(Baseline = true)>]
    member _.HighGCPressure() =
        let mutable sum = 0
        for _ in 1..10000 do
            let box = box 42  // Boxing = heap allocation
            sum <- sum + (unbox box : int)
        sum
    
    // Low GC pressure: ไม่ boxing
    [<Benchmark>]
    member _.LowGCPressure() =
        let mutable sum = 0
        for _ in 1..10000 do
            sum <- sum + 42
        sum
    
    // สาธิต: Seq.unfold vs List.init
    [<Benchmark>]
    member _.SeqUnfold() =
        Seq.unfold (fun state -> if state > 1000 then None else Some(state, state + 1)) 1
        |> Seq.sum
    
    [<Benchmark>]
    member _.ListInit() =
        List.init 1001 id |> List.sum
    
    // Object pool pattern
    let pool = System.Collections.Concurrent.ConcurrentBag<System.Text.StringBuilder>()
    
    let rentStringBuilder () =
        match pool.TryTake() with
        | true, sb ->
            sb.Clear() |> ignore
            sb
        | false, _ ->
            System.Text.StringBuilder()
    
    let returnStringBuilder (sb: System.Text.StringBuilder) =
        if pool.Count < 10 then
            pool.Add(sb)
    
    [<Benchmark>]
    member _.WithObjectPool() =
        let sb = rentStringBuilder()
        for i in 1..100 do
            sb.Append(string i) |> ignore
        let result = sb.ToString()
        returnStringBuilder sb
        result
    
    [<Benchmark>]
    member _.WithoutObjectPool() =
        let sb = System.Text.StringBuilder()
        for i in 1..100 do
            sb.Append(string i) |> ignore
        sb.ToString()
```

---

## 5. Comparing Implementations

```fsharp
// SortingBenchmarks.fs
module Benchmarks.SortingBenchmarks

open BenchmarkDotNet.Attributes
open System

// เปรียบเทียบ sorting algorithms
[<MemoryDiagnoser>]
[<RankColumn>]
type SortingBenchmarks() =
    
    let rng = Random(42)
    
    [<Params(100, 1000, 10000)>]
    member val Size = 0 with get, set
    
    member private this.Data() =
        Array.init this.Size (fun _ -> rng.Next())
    
    [<Benchmark(Baseline = true)>]
    member this.BuiltInSort() =
        let data = this.Data()
        Array.sort data
        data
    
    [<Benchmark>]
    member this.ListSort() =
        let data = this.Data() |> Array.toList
        List.sort data
    
    [<Benchmark>]
    member this.LinqOrderBy() =
        this.Data()
        |> Seq.sortBy id
        |> Seq.toArray
    
    [<Benchmark>]
    member this.ManualQuickSort() =
        let data = this.Data()
        
        let rec quickSort (arr: int[]) low high =
            if low < high then
                let pivot = arr.[high]
                let mutable i = low - 1
                
                for j in low..high-1 do
                    if arr.[j] <= pivot then
                        i <- i + 1
                        let tmp = arr.[i]
                        arr.[i] <- arr.[j]
                        arr.[j] <- tmp
                
                let tmp = arr.[i+1]
                arr.[i+1] <- arr.[high]
                arr.[high] <- tmp
                let pi = i + 1
                
                quickSort arr low (pi - 1)
                quickSort arr (pi + 1) high
        
        quickSort data 0 (data.Length - 1)
        data
    
    // Parallel sorting
    [<Benchmark>]
    member this.ParallelSort() =
        let data = this.Data()
        System.Array.Sort(data)  // Uses intro sort
        data

// F# specific comparisons
[<MemoryDiagnoser>]
type FSharpCollectionBenchmarks() =
    
    let size = 10000
    let data = Array.init size id
    
    // Fold vs sum
    [<Benchmark(Baseline = true)>]
    member _.ArrayFold() =
        data |> Array.fold (+) 0
    
    [<Benchmark>]
    member _.ArraySum() =
        Array.sum data
    
    [<Benchmark>]
    member _.ArraySumBy() =
        data |> Array.sumBy id
    
    // Filter + Map vs Choose
    [<Benchmark>]
    member _.FilterThenMap() =
        data
        |> Array.filter (fun x -> x % 2 = 0)
        |> Array.map (fun x -> x * x)
    
    [<Benchmark>]
    member _.Choose() =
        data
        |> Array.choose (fun x -> 
            if x % 2 = 0 then Some (x * x)
            else None)
    
    // List vs Array vs Seq
    [<Benchmark>]
    member _.SumWithList() =
        data
        |> Array.toList
        |> List.map (fun x -> x * 2)
        |> List.sum
    
    [<Benchmark>]
    member _.SumWithArray() =
        data
        |> Array.map (fun x -> x * 2)
        |> Array.sum
    
    [<Benchmark>]
    member _.SumWithSeq() =
        data
        |> Seq.map (fun x -> x * 2)
        |> Seq.sum
```

---

## 6. Micro-benchmarking Pitfalls

```fsharp
// BenchmarkPitfalls.fs
module Benchmarks.BenchmarkPitfalls

open BenchmarkDotNet.Attributes

// Pitfall 1: Dead code elimination
// JIT อาจลบ code ที่ไม่ใช้ผลลัพธ์

// BAD: ผลลัพธ์ถูก discard - JIT อาจ optimize away
[<Benchmark>]
let badBenchmark () =
    let _ = List.sort [3;1;2]  // Result not used, JIT may skip this!
    ()

// GOOD: Return ผลลัพธ์
[<Benchmark>]
let goodBenchmark () =
    List.sort [3;1;2]  // Return value prevents dead code elimination

// Pitfall 2: Setup ใน benchmark body
// BAD: การสร้าง data ใน benchmark body เพิ่ม overhead
[<MemoryDiagnoser>]
type BadBenchmarks() =
    [<Benchmark>]
    member _.BadSetup() =
        // Data creation is part of benchmark!
        let data = Array.init 1000 id  // This is measured too
        Array.sum data

// GOOD: Setup ใน GlobalSetup/IterationSetup
type GoodBenchmarks() =
    let mutable data: int[] = [||]
    
    [<GlobalSetup>]
    member _.Setup() =
        data <- Array.init 1000 id  // Runs before benchmarking
    
    [<Benchmark>]
    member _.GoodBenchmark() =
        Array.sum data  // Only this is measured

// Pitfall 3: Benchmarking JIT compilation
// First run includes JIT compilation overhead
// BenchmarkDotNet handles this with warmup runs

// Pitfall 4: Benchmark ที่ซับซ้อนเกินไป
// BAD: ทดสอบหลายสิ่งในครั้งเดียว
type BadComplexBenchmark() =
    [<Benchmark>]
    member _.TooManyOperations() =
        // Sorting + filtering + mapping - ไม่รู้ว่าอะไรช้า
        [1..1000]
        |> List.sort
        |> List.filter (fun x -> x % 2 = 0)
        |> List.map (fun x -> x * x)
        |> List.sum

// GOOD: แยก benchmark ตามการทำงาน
type GoodSeparatedBenchmarks() =
    let data = [1..1000]
    
    [<Benchmark>]
    member _.SortOnly() = List.sort data
    
    [<Benchmark>]
    member _.FilterOnly() = data |> List.filter (fun x -> x % 2 = 0)
    
    [<Benchmark>]
    member _.MapOnly() = data |> List.map (fun x -> x * x)

// Pitfall 5: Randomness ใน benchmarks
// ใช้ seed ที่กำหนดไว้เพื่อ reproducibility
type ReproducibleBenchmarks() =
    let rng = System.Random(42)  // Fixed seed!
    
    member _.GetData() = 
        Array.init 1000 (fun _ -> rng.Next())
    
    [<Benchmark>]
    member this.SortRandom() =
        let data = this.GetData()
        Array.sort data
        data
```

---

## 7. Profiling กับ dotnet-trace

```bash
# ติดตั้ง dotnet-trace
dotnet tool install --global dotnet-trace

# Collect CPU trace
dotnet-trace collect --process-id <pid> --duration 00:00:30

# Collect from startup
dotnet-trace collect -- dotnet run --project MyApp.fsproj

# ดู available providers
dotnet-trace list-profiles

# Collect with specific profile
dotnet-trace collect --process-id <pid> --profile cpu-sampling

# Convert trace to other formats
dotnet-trace convert trace.nettrace --format speedscope
dotnet-trace convert trace.nettrace --format chromium

# View in Visual Studio or PerfView
# Windows: PerfView
# Cross-platform: Speedscope (https://www.speedscope.app/)
```

```fsharp
// ProfilingExample.fs
module MyApp.Profiling

// Code ที่อาจต้องการ profiling

// วิธีเพิ่ม custom events สำหรับ tracing
open System.Diagnostics.Tracing

[<EventSource(Name = "MyApp-Events")>]
type MyEventSource() =
    inherit EventSource()
    
    static let instance = new MyEventSource()
    static member Instance = instance
    
    [<Event(1, Message = "Processing started: {0}", Level = EventLevel.Informational)>]
    member this.ProcessingStarted(name: string) =
        if this.IsEnabled() then
            this.WriteEvent(1, name)
    
    [<Event(2, Message = "Processing completed in {0}ms", Level = EventLevel.Informational)>]
    member this.ProcessingCompleted(durationMs: int64) =
        if this.IsEnabled() then
            this.WriteEvent(2, durationMs)

// ใช้งาน events
let processData data =
    MyEventSource.Instance.ProcessingStarted("processData")
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    let result = data |> List.map (fun x -> x * 2)
    
    sw.Stop()
    MyEventSource.Instance.ProcessingCompleted(sw.ElapsedMilliseconds)
    result
```

---

## 8. dotnet-counters

```bash
# ติดตั้ง dotnet-counters
dotnet tool install --global dotnet-counters

# Monitor running process
dotnet-counters monitor --process-id <pid>

# Monitor specific counters
dotnet-counters monitor --process-id <pid> \
  --counters System.Runtime[cpu-usage,working-set,gc-heap-size]

# Monitor with refresh interval
dotnet-counters monitor --process-id <pid> --refresh-interval 1

# Collect to file
dotnet-counters collect --process-id <pid> --output metrics.json

# Available counter providers:
# - System.Runtime (CPU, memory, GC, etc.)
# - Microsoft.AspNetCore.Hosting (requests/sec, etc.)
# - Grpc.AspNetCore.Server
# - Custom providers
```

```fsharp
// CustomCounters.fs
module MyApp.CustomCounters

open System.Diagnostics.Metrics

// สร้าง custom metrics
let meter = new Meter("MyApp.Metrics", "1.0.0")

// Counter: counts events
let requestCounter = meter.CreateCounter<int>("requests.total")

// Histogram: tracks distributions
let requestDuration = meter.CreateHistogram<double>("request.duration", "ms")

// Gauge: tracks current value
let activeConnections = meter.CreateObservableGauge("connections.active", fun () ->
    // Return current active connections
    seq { Measurement<int>(42) }
)

// ใช้งาน metrics
let processRequest () =
    requestCounter.Add(1)
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    // Process...
    System.Threading.Thread.Sleep(10)
    
    sw.Stop()
    requestDuration.Record(sw.Elapsed.TotalMilliseconds)
```

---

## 9. Load Testing กับ NBomber

### การติดตั้ง

```xml
<PackageReference Include="NBomber" Version="5.2.0" />
<PackageReference Include="NBomber.Http" Version="5.2.0" />
```

### Basic Load Test

```fsharp
// LoadTests.fs
module Tests.LoadTests

open NBomber
open NBomber.Contracts
open NBomber.FSharp
open NBomber.Http.FSharp
open System.Net.Http

// Scenario สำหรับ load test
let createHttpScenario () =
    let httpClient = new HttpClient()
    
    // Step: HTTP GET request
    let getProducts = 
        Step.create("get_products", fun context -> task {
            let! response = httpClient.GetAsync("http://localhost:5000/api/products")
            
            return if response.IsSuccessStatusCode then
                       Response.ok(statusCode = int response.StatusCode)
                   else
                       Response.fail(statusCode = int response.StatusCode)
        })
    
    // Step: HTTP POST request
    let createProduct =
        Step.create("create_product", fun context -> task {
            let content = System.Net.Http.StringContent(
                """{"name":"Test Product","price":9.99}""",
                System.Text.Encoding.UTF8,
                "application/json"
            )
            
            let! response = httpClient.PostAsync("http://localhost:5000/api/products", content)
            
            return if response.IsSuccessStatusCode then
                       Response.ok()
                   else
                       Response.fail()
        })
    
    // กำหนด scenario
    Scenario.create("product_api_scenario", [getProducts; createProduct])
    |> Scenario.withWarmUpDuration (System.TimeSpan.FromSeconds(5.0))
    |> Scenario.withLoadSimulations [
        // ค่อยๆ เพิ่ม load
        Simulation.rampingInject(rate = 50, interval = System.TimeSpan.FromSeconds(1.0), during = System.TimeSpan.FromSeconds(30.0))
        // คงที่
        Simulation.inject(rate = 100, interval = System.TimeSpan.FromSeconds(1.0), during = System.TimeSpan.FromSeconds(60.0))
        // ค่อยๆ ลด
        Simulation.rampingInject(rate = 0, interval = System.TimeSpan.FromSeconds(1.0), during = System.TimeSpan.FromSeconds(30.0))
    ]

// Simple Load Test Runner
let runLoadTest () =
    let scenario = createHttpScenario()
    
    NBomberRunner
        .registerScenarios([scenario])
        .run()

// ใช้ใน test
[<Xunit.Fact>]
let ``api handles 100 requests per second`` () =
    let scenario = 
        Scenario.create("simple_load", [
            Step.create("get", fun _ -> task {
                // Simulate work
                do! System.Threading.Tasks.Task.Delay(10)
                return Response.ok()
            })
        ])
        |> Scenario.withLoadSimulations [
            Simulation.inject(rate = 100, interval = System.TimeSpan.FromSeconds(1.0), during = System.TimeSpan.FromSeconds(10.0))
        ]
    
    let stats = 
        NBomberRunner
            .registerScenarios([scenario])
            .run()
    
    // Assert performance criteria
    let stepStats = stats.ScenarioStats.[0].StepStats.[0]
    Xunit.Assert.True(stepStats.Ok.Request.RPS >= 90.0, "Should handle at least 90 RPS")
    Xunit.Assert.True(stepStats.Fail.Request.Count = 0, "Should have no failures")
```

---

## 10. Advanced BenchmarkDotNet

```fsharp
// AdvancedBenchmarks.fs
module Benchmarks.Advanced

open BenchmarkDotNet.Attributes
open BenchmarkDotNet.Configs
open BenchmarkDotNet.Jobs

// Custom configuration
type BenchmarkConfig() =
    inherit ManualConfig()
    
    do
        base.AddJob(Job.ShortRun) |> ignore
        base.AddJob(Job.LongRun) |> ignore

// Multiple runtime targets
[<Config(typeof<BenchmarkConfig>)>]
[<MemoryDiagnoser>]
[<CsvExporter>]
[<HtmlExporter>]
[<RPlotExporter>]
type ComprehensiveBenchmarks() =
    
    [<GlobalSetup>]
    member _.Setup() =
        printfn "Setting up benchmarks..."
    
    [<GlobalCleanup>]
    member _.Cleanup() =
        printfn "Cleaning up..."
    
    [<IterationSetup>]
    member _.IterationSetup() =
        // Called before each iteration
        ()
    
    [<IterationCleanup>]
    member _.IterationCleanup() =
        // Called after each iteration
        ()
    
    [<Benchmark(Description = "LINQ approach")>]
    member _.LinqApproach() =
        [1..1000]
        |> Seq.filter (fun x -> x % 2 = 0)
        |> Seq.map (fun x -> x * x)
        |> Seq.sum
    
    [<Benchmark(Description = "F# native approach", Baseline = true)>]
    member _.FSharpNative() =
        [|1..1000|]
        |> Array.choose (fun x -> if x % 2 = 0 then Some (x * x) else None)
        |> Array.sum

// Benchmark กับ async
[<MemoryDiagnoser>]
type AsyncBenchmarks() =
    
    [<Benchmark>]
    member _.AsyncWorkflow() =
        async {
            let! result = async { return 42 }
            return result
        }
        |> Async.RunSynchronously
    
    [<Benchmark>]
    member _.TaskBasedAsync() =
        System.Threading.Tasks.Task.Run(fun () -> 42).Result
    
    [<Benchmark>]
    member _.ValueTaskAsync() =
        System.Threading.Tasks.ValueTask<int>(42).Result

// Reporting results
let printResults (stats: BenchmarkDotNet.Reports.Summary) =
    printfn "=== Benchmark Results ==="
    for report in stats.Reports do
        let method = report.BenchmarkCase.Descriptor.WorkloadMethod.Name
        let mean = report.ResultStatistics.Mean
        let stdDev = report.ResultStatistics.StandardDeviation
        printfn "%s: %.2f ns (±%.2f ns)" method mean stdDev
```

---

## 11. Performance Testing Best Practices

```fsharp
// BestPractices.fs
module Benchmarks.BestPractices

open BenchmarkDotNet.Attributes

// 1. Always use Release build
// dotnet run -c Release

// 2. Warm up ก่อน benchmark
// BenchmarkDotNet handles this automatically

// 3. ลด noise
[<MemoryDiagnoser>]
[<DisassemblyDiagnoser>]
type CleanBenchmarks() =
    
    // 4. ใช้ params สำหรับ different sizes
    [<Params(100, 1000, 10000, 100000)>]
    member val N = 0 with get, set
    
    member private this.CreateData() = Array.init this.N id
    
    // 5. ทดสอบ realistic workloads
    [<Benchmark>]
    member this.RealisticWorkload() =
        // Simulates actual use case
        let data = this.CreateData()
        data
        |> Array.filter (fun x -> x % 3 = 0)
        |> Array.map (fun x -> x * x)
        |> Array.take (min 100 (data.Length / 3))
        |> Array.sum
    
    // 6. เปรียบเทียบ alternatives
    [<Benchmark(Baseline = true)>]
    member this.Implementation1() =
        this.CreateData()
        |> Array.choose (fun x ->
            if x % 3 = 0 then Some (x * x)
            else None)
        |> Array.truncate 100
        |> Array.sum
    
    // 7. Document ทำไมถึงเปรียบเทียบ
    /// <summary>
    /// Alternative using PLINQ for parallelism
    /// Expected to be faster for large N
    /// </summary>
    [<Benchmark>]
    member this.ParallelImplementation() =
        let data = this.CreateData()
        data.AsParallel()
        |> Seq.filter (fun x -> x % 3 = 0)
        |> Seq.map (fun x -> x * x)
        |> Seq.take (min 100 (data.Length / 3))
        |> Seq.sum
```

---

## สรุป (Summary)

Performance Testing ใน F#:

1. **BenchmarkDotNet** - Micro-benchmarks ที่เชื่อถือได้
2. **[<MemoryDiagnoser>]** - วัด allocations
3. **[<Params>]** - ทดสอบ different sizes
4. **dotnet-trace** - CPU profiling
5. **dotnet-counters** - Runtime metrics monitoring
6. **NBomber** - Load testing

**Common Performance Tips สำหรับ F#:**
- ใช้ `Array` แทน `List` สำหรับ performance-critical code
- ใช้ `Array.choose` แทน `filter + map`
- หลีกเลี่ยง boxing/unboxing
- ใช้ `ValueTask` แทน `Task` สำหรับ frequent, fast operations
- ใช้ `Span<T>` สำหรับ zero-allocation slicing
- Profile ก่อน optimize: measure, don't guess

```bash
# Quick benchmarking
dotnet run -c Release -- --job short --filter "*MyBenchmarks*"

# Full analysis
dotnet run -c Release -- --exporters html csv --artifacts ./results
```
