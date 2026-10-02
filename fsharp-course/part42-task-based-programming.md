# Part 42 - การเขียนโปรแกรมแบบ Task (Task-Based Programming)

## บทนำ

ใน .NET มี 2 วิธีหลักในการเขียนโปรแกรมแบบอะซิงโครนัส คือ `Async<T>` ของ F# และ `Task<T>` ของ .NET Task-Based Asynchronous Pattern (TAP) บทนี้จะอธิบายความแตกต่าง วิธีใช้ และการแปลงระหว่างกัน

## 1. Task<T> vs Async<T>

```fsharp
open System.Threading.Tasks

// Async<T> - F# native
let fsharpAsync: Async<int> = async {
    return 42
}

// Task<T> - .NET native (ใช้กันใน C# เป็นหลัก)
let dotnetTask: Task<int> = Task.FromResult(42)

// ความแตกต่างหลัก:
// 1. Async<T> เป็น "cold" - ยังไม่ทำงานจนกว่าจะ run
// 2. Task<T> เป็น "hot" - เริ่มทำงานทันทีเมื่อสร้าง

// เปรียบเทียบ
printfn "Async<int> ยังไม่ทำงาน"
printfn "Task<int> เริ่มทำงานแล้ว: %b" (dotnetTask.IsCompleted)
```

```fsharp
// ตารางเปรียบเทียบ Async vs Task
(*
| คุณสมบัติ       | Async<T>              | Task<T>               |
|-----------------|-----------------------|-----------------------|
| เริ่มทำงาน      | เมื่อ run             | ทันที (hot)           |
| Cancel          | CancellationToken     | CancellationToken     |
| Exception       | ในตัว                 | AggregateException    |
| .NET compat     | ต้องแปลง              | native                |
| F# idiomatic    | ใช่                   | ผ่านได้               |
| Performance     | ดี                    | ดีมาก (F# 6+)         |
*)
```

## 2. task { } Computation Expression (F# 6+)

F# 6 เพิ่ม `task { }` computation expression ที่ทำงานโดยตรงกับ Task<T>

```fsharp
// task { } คล้ายกับ async { } แต่ทำงานกับ Task<T>
let basicTask () : Task<int> = task {
    return 42
}

// รัน task
let t = basicTask ()
printfn "ผลลัพธ์: %d" t.Result
```

```fsharp
// task { } พร้อม let!
let taskWithAwait () : Task<string> = task {
    let! num = Task.FromResult(42)
    let! str = Task.FromResult("Hello")
    return sprintf "%s: %d" str num
}

let result = taskWithAwait().Result
printfn "%s" result
```

```fsharp
// task { } พร้อม async I/O
open System.IO

let readFileTask (path: string) : Task<string> = task {
    let! content = File.ReadAllTextAsync(path)
    return content.ToUpper()
}

// เทียบกับ async version
let readFileAsync (path: string) : Async<string> = async {
    let! content = File.ReadAllTextAsync(path) |> Async.AwaitTask
    return content.ToUpper()
}
```

```fsharp
// task { } กับ do!
let taskWithDoAwait () : Task<unit> = task {
    do! Task.Delay(100)
    printfn "Task หลัง delay"
    do! Task.Delay(100)
    printfn "Task เสร็จสิ้น"
}

taskWithDoAwait().Wait()
```

## 3. let! กับ Task

```fsharp
// ใช้ let! ใน task { } เพื่อรอ Task
let chainedTasks () : Task<int> = task {
    let! a = Task.FromResult(10)
    let! b = Task.FromResult(20)
    let! c = task {
        do! Task.Delay(50)
        return 30
    }
    return a + b + c
}

printfn "ผลรวม: %d" (chainedTasks().Result)
```

```fsharp
// let! กับ Task ที่มี side effects
let sideEffectTask () : Task<int list> = task {
    let results = ResizeArray<int>()
    
    for i in 1..5 do
        let! value = task {
            do! Task.Delay(20)
            return i * i
        }
        results.Add(value)
    
    return results |> Seq.toList
}

let results = sideEffectTask().Result
printfn "ผลลัพธ์: %A" results
```

## 4. Converting Task to Async

```fsharp
open System.Threading.Tasks

// วิธีแปลง Task เป็น Async
let taskToAsyncExamples () = 
    
    // วิธีที่ 1: Async.AwaitTask
    let task1: Task<int> = Task.FromResult(42)
    let async1: Async<int> = Async.AwaitTask task1
    
    // วิธีที่ 2: สร้าง wrapper
    let wrapTask (t: Task<'a>) : Async<'a> = 
        Async.AwaitTask t
    
    // วิธีที่ 3: ใน async { }
    let combined = async {
        let task = Task.FromResult(100)
        let! result = Async.AwaitTask task
        return result * 2
    }
    
    printfn "Async ผลลัพธ์: %d" (Async.RunSynchronously combined)
```

```fsharp
// แปลง Task<unit> เป็น Async<unit>
let taskUnitToAsync () = async {
    let delayTask: Task = Task.Delay(100)  // Task (ไม่ใช่ Task<T>)
    
    // ต้องแปลงก่อน
    do! delayTask |> Async.AwaitTask
    printfn "Delay เสร็จแล้ว"
}

Async.RunSynchronously (taskUnitToAsync ())
```

## 5. Async.AwaitTask และ Async.StartAsTask

```fsharp
// Async.AwaitTask - แปลง Task เป็น Async
let awaitTaskExample () = async {
    // รับ Task จาก .NET library
    let task = Task.FromResult("ข้อมูลจาก .NET")
    
    // แปลงและรอ
    let! result = Async.AwaitTask task
    printfn "ได้รับ: %s" result
    return result
}

Async.RunSynchronously (awaitTaskExample ()) |> ignore
```

```fsharp
// Async.StartAsTask - แปลง Async เป็น Task
let startAsTaskExample () =
    let asyncWork: Async<string> = async {
        do! Async.Sleep 100
        return "ข้อมูลจาก F# Async"
    }
    
    // แปลงเป็น Task (สำหรับส่งให้ C# library)
    let task: Task<string> = Async.StartAsTask asyncWork
    
    // รอผลลัพธ์
    let result = task.Result
    printfn "ได้รับ: %s" result

startAsTaskExample ()
```

```fsharp
// การทำงานร่วมกันระหว่าง Async และ Task
let interopExample () =
    // .NET API ที่คืน Task
    let dotNetOperation (): Task<int> = task {
        do! Task.Delay(50)
        return 42
    }
    
    // F# code ที่ใช้ .NET API
    let fsharpWrapper () = async {
        let! result = dotNetOperation() |> Async.AwaitTask
        return result * 2
    }
    
    // C# code ที่ใช้ F# async
    let forCSharp: Task<int> = Async.StartAsTask (fsharpWrapper ())
    
    printfn "ผลลัพธ์: %d" forCSharp.Result

interopExample ()
```

## 6. ValueTask<T>

```fsharp
open System.Threading.Tasks

// ValueTask<T> เป็น value type ที่มี overhead น้อยกว่า Task<T>
// ใช้เมื่อผลลัพธ์มักจะพร้อมทันที (synchronous path)

let getValueTaskExample () =
    // สร้าง ValueTask ที่เสร็จทันที
    let syncValueTask: ValueTask<int> = ValueTask<int>(42)
    printfn "Sync ValueTask: %d" syncValueTask.Result
    
    // สร้าง ValueTask จาก Task
    let asyncValueTask: ValueTask<int> = 
        ValueTask<int>(Task.FromResult(100))
    printfn "Async ValueTask: %d" asyncValueTask.Result

getValueTaskExample ()
```

```fsharp
// ValueTask ใน F# task { }
let valueTaskInTask () : Task<int> = task {
    // สามารถใช้ let! กับ ValueTask ได้
    let vt = ValueTask<int>(42)
    let! result = vt
    return result * 2
}

printfn "ValueTask ผ่าน task: %d" (valueTaskInTask().Result)
```

```fsharp
// เปรียบเทียบ Task กับ ValueTask
let compareTaskAndValueTask () =
    let iterations = 1_000_000
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    // Task
    for _ in 1..iterations do
        let t = Task.FromResult(42)
        t.Result |> ignore
    
    sw.Stop()
    let taskTime = sw.ElapsedMilliseconds
    
    sw.Restart()
    
    // ValueTask (น้อย allocations กว่า)
    for _ in 1..iterations do
        let vt = ValueTask<int>(42)
        vt.Result |> ignore
    
    sw.Stop()
    let valueTaskTime = sw.ElapsedMilliseconds
    
    printfn "Task: %d ms" taskTime
    printfn "ValueTask: %d ms" valueTaskTime
```

## 7. Task.WhenAll

```fsharp
// Task.WhenAll - รอให้ทุก task เสร็จ
let whenAllExample () : Task = task {
    let task1 = task {
        do! Task.Delay(100)
        printfn "Task 1 เสร็จ"
    }
    
    let task2 = task {
        do! Task.Delay(200)
        printfn "Task 2 เสร็จ"
    }
    
    let task3 = task {
        do! Task.Delay(150)
        printfn "Task 3 เสร็จ"
    }
    
    // รอให้ทุก task เสร็จ
    do! Task.WhenAll([task1; task2; task3])
    printfn "ทุก task เสร็จแล้ว"
}

whenAllExample().Wait()
```

```fsharp
// Task.WhenAll กับ Task<T>
let whenAllWithResults () : Task<int[]> = task {
    let tasks = 
        [1..5] 
        |> List.map (fun i -> task {
            do! Task.Delay(i * 50)
            return i * i
        })
    
    // รอและรับผลลัพธ์ทั้งหมด
    let! results = Task.WhenAll(tasks)
    return results
}

let results = whenAllExample().ContinueWith(fun _ -> whenAllWithResults().Result).Result
printfn "WhenAll ผลลัพธ์: %A" results
```

```fsharp
// WhenAll กับ error handling
let whenAllWithError () : Task<string list> = task {
    let failingTask: Task<string> = task {
        do! Task.Delay(50)
        raise (System.Exception("Task ล้มเหลว!"))
        return "ไม่มีทางถึงตรงนี้"
    }
    
    let okTask: Task<string> = task {
        do! Task.Delay(100)
        return "Task สำเร็จ"
    }
    
    try
        let! results = Task.WhenAll([failingTask; okTask])
        return results |> Array.toList
    with
    | :? AggregateException as ae ->
        for ex in ae.InnerExceptions do
            printfn "ข้อผิดพลาด: %s" ex.Message
        return []
}

let wae = whenAllWithError ()
wae.Wait()
printfn "ผลลัพธ์: %A" wae.Result
```

## 8. Task.WhenAny

```fsharp
// Task.WhenAny - รอ task แรกที่เสร็จ
let whenAnyExample () : Task = task {
    let slow = task {
        do! Task.Delay(1000)
        return "ช้า"
    }
    
    let fast = task {
        do! Task.Delay(100)
        return "เร็ว"
    }
    
    let medium = task {
        do! Task.Delay(500)
        return "ปานกลาง"
    }
    
    // รอ task แรกที่เสร็จ
    let! winner = Task.WhenAny([slow; fast; medium])
    printfn "Task แรกที่เสร็จ: %s" winner.Result
}

whenAnyExample().Wait()
```

```fsharp
// WhenAny สำหรับ timeout pattern
let withTaskTimeout (timeoutMs: int) (work: Task<'a>) : Task<'a option> = task {
    let timeoutTask: Task<'a option> = task {
        do! Task.Delay(timeoutMs)
        return None
    }
    
    let workWithSome: Task<'a option> = task {
        let! result = work
        return Some result
    }
    
    let! winner = Task.WhenAny([timeoutTask; workWithSome])
    return! winner
}

// ทดสอบ timeout
let timeoutTest () = task {
    let slowWork = task {
        do! Task.Delay(500)
        return "เสร็จแล้ว"
    }
    
    let! result = withTaskTimeout 200 slowWork
    printfn "ผลลัพธ์: %A" result  // None (timeout)
}

timeoutTest().Wait()
```

## 9. Task.Delay

```fsharp
// Task.Delay สำหรับการรอ
let delayExamples () = task {
    printfn "เริ่มต้น"
    
    // รอ 100ms
    do! Task.Delay(100)
    printfn "หลัง 100ms"
    
    // รอ TimeSpan
    do! Task.Delay(System.TimeSpan.FromSeconds(0.1))
    printfn "หลัง 100ms (TimeSpan)"
    
    // รอพร้อม CancellationToken
    use cts = new System.Threading.CancellationTokenSource()
    do! Task.Delay(100, cts.Token)
    printfn "หลัง 100ms (cancellable)"
}

delayExamples().Wait()
```

```fsharp
// ใช้ Task.Delay สำหรับ polling
let pollUntilReady (checkFn: unit -> Task<bool>) (intervalMs: int) (maxAttempts: int) = task {
    let mutable attempts = 0
    let mutable ready = false
    
    while not ready && attempts < maxAttempts do
        let! isReady = checkFn ()
        if isReady then
            ready <- true
        else
            attempts <- attempts + 1
            do! Task.Delay(intervalMs)
    
    return ready
}

// จำลองการ poll
let pollExample () = task {
    let mutable callCount = 0
    
    let check () = task {
        callCount <- callCount + 1
        printfn "ตรวจสอบครั้งที่ %d" callCount
        return callCount >= 3  // พร้อมหลัง 3 ครั้ง
    }
    
    let! ready = pollUntilReady check 100 10
    printfn "พร้อม: %b (ตรวจ %d ครั้ง)" ready callCount
}

pollExample().Wait()
```

## 10. Cancellation กับ CancellationTokenSource

```fsharp
open System.Threading

// CancellationTokenSource พื้นฐาน
let cancellationBasics () =
    use cts = new CancellationTokenSource()
    
    let work: Task = task {
        try
            for i in 1..10 do
                cts.Token.ThrowIfCancellationRequested()
                printfn "ทำงานรอบ %d" i
                do! Task.Delay(100, cts.Token)
        with
        | :? OperationCanceledException ->
            printfn "ถูกยกเลิก!"
    }
    
    // ยกเลิกหลัง 350ms
    Task.Delay(350).ContinueWith(fun _ -> cts.Cancel()) |> ignore
    
    work.Wait()

cancellationBasics ()
```

```fsharp
// CancelAfter สำหรับ timeout อัตโนมัติ
let autoCancelTask () =
    use cts = new CancellationTokenSource()
    cts.CancelAfter(300)  // ยกเลิกอัตโนมัติหลัง 300ms
    
    let work: Task<string> = task {
        do! Task.Delay(500, cts.Token)
        return "เสร็จแล้ว"
    }
    
    try
        work.Wait()
        printfn "ผลลัพธ์: %s" work.Result
    with
    | :? AggregateException as ae when 
        ae.InnerException :? OperationCanceledException ->
        printfn "หมดเวลาและถูกยกเลิก!"

autoCancelTask ()
```

```fsharp
// Linked CancellationTokenSource
let linkedTokens () =
    use cts1 = new CancellationTokenSource()  // user cancel
    use cts2 = new CancellationTokenSource(2000)  // timeout
    
    // รวม tokens ทั้งสอง
    use linkedCts = CancellationTokenSource.CreateLinkedTokenSource(cts1.Token, cts2.Token)
    
    let work: Task = task {
        try
            let mutable i = 0
            while true do
                i <- i + 1
                printfn "ทำงาน %d" i
                do! Task.Delay(300, linkedCts.Token)
        with
        | :? OperationCanceledException ->
            if cts1.IsCancellationRequested then
                printfn "ยกเลิกโดยผู้ใช้"
            elif cts2.IsCancellationRequested then
                printfn "หมดเวลา"
    }
    
    // ยกเลิกโดยผู้ใช้หลัง 700ms
    Task.Delay(700).ContinueWith(fun _ -> cts1.Cancel()) |> ignore
    work.Wait()

linkedTokens ()
```

## 11. Error Handling ใน Tasks

```fsharp
// จัดการ exception ใน task
let errorHandlingInTask () = task {
    try
        let! result = task {
            raise (System.Exception("บัง!"))
            return 0
        }
        printfn "สำเร็จ: %d" result
    with
    | ex ->
        printfn "ข้อผิดพลาด: %s" ex.Message
}

errorHandlingInTask().Wait()
```

```fsharp
// Task<Result<T,E>> pattern
let safeTask (work: Task<'a>) : Task<Result<'a, string>> = task {
    try
        let! result = work
        return Ok result
    with
    | ex ->
        return Error ex.Message
}

// ใช้งาน
let safeExample () = task {
    let riskyWork: Task<int> = task {
        raise (System.Exception("ข้อผิดพลาดทดสอบ"))
        return 42
    }
    
    let! result = safeTask riskyWork
    
    match result with
    | Ok v -> printfn "สำเร็จ: %d" v
    | Error msg -> printfn "ล้มเหลว: %s" msg
}

safeExample().Wait()
```

```fsharp
// AggregateException handling
let aggregateExceptionExample () =
    let task1: Task = task { raise (System.Exception("ข้อผิดพลาด 1")) }
    let task2: Task = task { raise (System.Exception("ข้อผิดพลาด 2")) }
    let task3: Task = Task.Delay(100)
    
    try
        Task.WhenAll([task1; task2; task3]).Wait()
    with
    | :? AggregateException as ae ->
        printfn "พบ %d ข้อผิดพลาด:" ae.InnerExceptions.Count
        for ex in ae.InnerExceptions do
            printfn "  - %s" ex.Message

aggregateExceptionExample ()
```

## 12. Combining Async and Task

```fsharp
// ผสม async และ task ในโปรแกรมเดียวกัน
let mixingAsyncAndTask () = async {
    // เรียก Task API
    let taskResult: Task<int> = task {
        do! Task.Delay(50)
        return 42
    }
    
    // แปลงและรอ
    let! result = Async.AwaitTask taskResult
    
    // ทำงาน async ต่อ
    let doubled = result * 2
    
    // แปลงกลับเป็น task ถ้าจำเป็น
    let backToTask: Task<int> = Async.StartAsTask (async { return doubled })
    let! finalResult = Async.AwaitTask backToTask
    
    return finalResult
}

let final = Async.RunSynchronously (mixingAsyncAndTask ())
printfn "ผลลัพธ์สุดท้าย: %d" final
```

```fsharp
// สร้าง bridge module
module AsyncTaskBridge =
    // Async -> Task
    let toTask (a: Async<'a>) : Task<'a> = 
        Async.StartAsTask a
    
    // Task -> Async
    let toAsync (t: Task<'a>) : Async<'a> = 
        Async.AwaitTask t
    
    // Task (unit) -> Async<unit>
    let toAsyncUnit (t: Task) : Async<unit> = 
        Async.AwaitTask t
    
    // รัน async จาก synchronous context
    let run (a: Async<'a>) : 'a = 
        Async.RunSynchronously a

// ใช้งาน bridge
let bridgeExample () =
    let asyncOp = async { return "จาก Async" }
    let taskOp: Task<string> = task { return "จาก Task" }
    
    let taskFromAsync = AsyncTaskBridge.toTask asyncOp
    let asyncFromTask = AsyncTaskBridge.toAsync taskOp
    
    printfn "%s" taskFromAsync.Result
    printfn "%s" (Async.RunSynchronously asyncFromTask)

bridgeExample ()
```

## 13. Performance Comparison

```fsharp
open System.Diagnostics

// เปรียบเทียบ performance ระหว่าง Async และ Task
let performanceComparison () =
    let iterations = 10_000
    
    // Task performance
    let sw = Stopwatch.StartNew()
    for _ in 1..iterations do
        let t: Task<int> = task { return 42 }
        t.Result |> ignore
    sw.Stop()
    let taskTime = sw.ElapsedMilliseconds
    
    // Async performance
    sw.Restart()
    for _ in 1..iterations do
        let a = async { return 42 }
        Async.RunSynchronously a |> ignore
    sw.Stop()
    let asyncTime = sw.ElapsedMilliseconds
    
    printfn "Task: %d ms สำหรับ %d iterations" taskTime iterations
    printfn "Async: %d ms สำหรับ %d iterations" asyncTime iterations
    printfn "Ratio: %.2f" (float asyncTime / float taskTime)

performanceComparison ()
```

```fsharp
// Benchmark การ await จำนวนมาก
let awaitManyBenchmark () =
    let count = 1000
    
    // Task parallel
    let sw = Stopwatch.StartNew()
    let tasks = [|for _ in 1..count -> Task.FromResult(1)|]
    Task.WhenAll(tasks).Wait()
    sw.Stop()
    let taskParallelTime = sw.ElapsedMilliseconds
    
    // Async parallel
    sw.Restart()
    let asyncs = [for _ in 1..count -> async { return 1 }]
    Async.RunSynchronously(Async.Parallel asyncs) |> ignore
    sw.Stop()
    let asyncParallelTime = sw.ElapsedMilliseconds
    
    printfn "Task.WhenAll(%d): %d ms" count taskParallelTime
    printfn "Async.Parallel(%d): %d ms" count asyncParallelTime
```

## 14. Task Composition Patterns

```fsharp
// Map สำหรับ Task
let taskMap (f: 'a -> 'b) (t: Task<'a>) : Task<'b> = task {
    let! value = t
    return f value
}

// Bind สำหรับ Task
let taskBind (f: 'a -> Task<'b>) (t: Task<'a>) : Task<'b> = task {
    let! value = t
    return! f value
}

// ตัวอย่าง
let compositionExample () = task {
    let initial: Task<int> = Task.FromResult(10)
    
    let doubled = taskMap (fun x -> x * 2) initial
    let asString = taskMap string doubled
    let withBrackets = taskBind (fun s -> task { return sprintf "[%s]" s }) asString
    
    let! result = withBrackets
    printfn "ผลลัพธ์: %s" result
}

compositionExample().Wait()
```

```fsharp
// Task pipeline operator
let (|>>) (t: Task<'a>) (f: 'a -> 'b) : Task<'b> = taskMap f t
let (>>>) (t: Task<'a>) (f: 'a -> Task<'b>) : Task<'b> = taskBind f t

// ใช้งาน
let pipelineExample () = 
    Task.FromResult(42)
    |>> (fun x -> x * 2)
    |>> string
    |>> (fun s -> sprintf "ผลลัพธ์: %s" s)

printfn "%s" (pipelineExample().Result)
```

## 15. Task กับ IDisposable

```fsharp
// ใช้ use ใน task
let taskWithDisposable () : Task = task {
    // use จะ dispose เมื่อ task เสร็จหรือเกิด exception
    use stream = new System.IO.MemoryStream()
    
    // เขียนข้อมูล
    let bytes = System.Text.Encoding.UTF8.GetBytes("Hello, Task!")
    do! stream.WriteAsync(bytes, 0, bytes.Length)
    
    printfn "เขียน %d bytes" stream.Length
    
    // MemoryStream จะถูก dispose อัตโนมัติ
}

taskWithDisposable().Wait()
```

```fsharp
// IAsyncDisposable
open System

// จำลอง async disposable resource
type AsyncResource() =
    let mutable disposed = false
    
    interface IAsyncDisposable with
        member _.DisposeAsync() = 
            task {
                printfn "กำลัง dispose แบบ async..."
                do! Task.Delay(50)
                disposed <- true
                printfn "Dispose เสร็จแล้ว"
            } |> ValueTask
    
    member _.DoWork() = task {
        if disposed then failwith "ถูก dispose แล้ว"
        do! Task.Delay(100)
        printfn "ทำงานเสร็จ"
    }

let asyncDisposeExample () : Task = task {
    await (asyncResource <- new AsyncResource())
    do! asyncResource.DoWork()
    // AsyncResource จะถูก DisposeAsync เมื่อออกจาก scope
}

// Note: F# task { } รองรับ IAsyncDisposable ผ่าน await using
```

## 16. ตัวอย่างจริง: HTTP Client

```fsharp
open System.Net.Http
open System.Text.Json

// HTTP Client ที่ใช้ Task
module HttpClient =
    
    type ApiResponse<'T> = {
        Data: 'T
        StatusCode: int
        IsSuccess: bool
    }
    
    let get<'T> (url: string) : Task<ApiResponse<'T>> = task {
        use client = new HttpClient()
        
        let! response = client.GetAsync(url)
        let statusCode = int response.StatusCode
        
        if response.IsSuccessStatusCode then
            let! body = response.Content.ReadAsStringAsync()
            let data = JsonSerializer.Deserialize<'T>(body)
            return { Data = data; StatusCode = statusCode; IsSuccess = true }
        else
            return failwithf "HTTP %d: %s" statusCode response.ReasonPhrase
    }
    
    // Retry wrapper
    let rec getWithRetry<'T> (maxRetries: int) (url: string) : Task<ApiResponse<'T>> = task {
        try
            return! get<'T> url
        with
        | ex when maxRetries > 0 ->
            printfn "ลองใหม่ (%d): %s" maxRetries ex.Message
            do! Task.Delay(1000)
            return! getWithRetry (maxRetries - 1) url
    }
```

## 17. ตัวอย่างจริง: Database Operations

```fsharp
// จำลอง database operations ด้วย Task
module Database =
    
    type User = {
        Id: int
        Name: string
        Email: string
    }
    
    // จำลอง async database call
    let getUser (id: int) : Task<User option> = task {
        do! Task.Delay(50)  // จำลอง I/O
        
        if id = 1 then
            return Some { Id = 1; Name = "สมชาย"; Email = "somchai@example.com" }
        else
            return None
    }
    
    let getUsersByIds (ids: int list) : Task<User list> = task {
        // ดึงแบบ parallel
        let! users = Task.WhenAll(ids |> List.map getUser)
        return users |> Array.choose id |> Array.toList
    }
    
    let saveUser (user: User) : Task<bool> = task {
        do! Task.Delay(100)  // จำลองการเขียน
        printfn "บันทึก user: %s" user.Name
        return true
    }
    
    // Transaction-like operation
    let createAndSave (name: string) (email: string) : Task<User> = task {
        let newUser = {
            Id = System.Random.Shared.Next(1000)
            Name = name
            Email = email
        }
        
        let! success = saveUser newUser
        
        if not success then
            failwith "บันทึกล้มเหลว"
        
        return newUser
    }

// ใช้งาน
let dbExample () = task {
    // ดึง user เดี่ยว
    let! user = Database.getUser 1
    printfn "User: %A" user
    
    // ดึงหลาย user พร้อมกัน
    let! users = Database.getUsersByIds [1; 2; 3]
    printfn "Users: %A" users
    
    // สร้าง user ใหม่
    let! newUser = Database.createAndSave "สุดา" "suda@example.com"
    printfn "ผู้ใช้ใหม่: %A" newUser
}

dbExample().Wait()
```

## 18. Task กับ Parallel Processing

```fsharp
// ประมวลผลข้อมูลแบบ parallel ด้วย Task
let parallelProcessing () = task {
    let items = [1..20]
    let maxConcurrent = 5
    
    // ใช้ Semaphore เพื่อจำกัด concurrent tasks
    use semaphore = new System.Threading.SemaphoreSlim(maxConcurrent)
    
    let processItem (i: int) = task {
        do! semaphore.WaitAsync()
        try
            do! Task.Delay(100)  // จำลองการประมวลผล
            return i * i
        finally
            semaphore.Release() |> ignore
    }
    
    let tasks = items |> List.map processItem
    let! results = Task.WhenAll(tasks)
    
    printfn "ประมวลผล %d items" results.Length
    printfn "ผลรวม: %d" (results |> Array.sum)
    
    return results
}

let results = parallelProcessing().Result
```

## 19. Task Scheduling

```fsharp
open System.Threading

// ควบคุม scheduling ของ Task
let schedulingExample () =
    // Task บน ThreadPool
    let threadPoolTask = Task.Run(fun () ->
        printfn "ThreadPool thread: %d" Thread.CurrentThread.ManagedThreadId
    )
    
    // Task บน UI thread (จำลอง)
    let context = SynchronizationContext.Current
    printfn "Current context: %A" context
    
    threadPoolTask.Wait()
```

```fsharp
// ConfigureAwait สำหรับ library code
let libraryTask () : Task<int> = task {
    // ใน library ควรใช้ ConfigureAwait(false)
    // เพื่อหลีกเลี่ยง deadlock ใน UI apps
    do! Task.Delay(100).ConfigureAwait(false)
    
    let! result = Task.FromResult(42).ConfigureAwait(false)
    return result
}

printfn "Library task: %d" (libraryTask().Result)
```

## 20. สรุป

```fsharp
// สรุป: เมื่อใช้ Task vs Async

(*
ใช้ Task<T> เมื่อ:
- ต้องการ interop กับ C# library
- ต้องการ performance สูงสุด (task { } ใน F# 6+)
- ใช้ Task.WhenAll/WhenAny
- ใช้กับ .NET framework APIs

ใช้ Async<T> เมื่อ:
- เขียน F#-only code
- ต้องการ idiomatic F# style
- ใช้ Async.Parallel, Async.Choice
- ต้องการ CancellationToken integration ง่ายๆ

ใน F# 6+ สามารถใช้ task { } แทน async { } ได้
เมื่อต้องการ performance และ interop กับ .NET
*)

printfn "Task-Based Programming - สรุปเสร็จ!"
```

## ตารางสรุปคำสั่ง

| คำสั่ง | ใช้งาน |
|--------|--------|
| `task { }` | สร้าง Task computation (F# 6+) |
| `let! x = task` | รอ Task ใน task/async |
| `do! task` | รอ Task<unit> |
| `Task.WhenAll` | รอทุก task |
| `Task.WhenAny` | รอ task แรกที่เสร็จ |
| `Task.Delay` | รอ (non-blocking) |
| `Async.AwaitTask` | Task -> Async |
| `Async.StartAsTask` | Async -> Task |
| `ValueTask<T>` | ประสิทธิภาพสูงกว่า |
| `CancellationTokenSource` | ยกเลิก task |
