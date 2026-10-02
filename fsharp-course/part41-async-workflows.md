# Part 41 - งานแบบอะซิงโครนัส (Async Workflows)

## บทนำ

Async Workflows เป็นหนึ่งในคุณสมบัติที่ทรงพลังที่สุดของ F# สำหรับการจัดการงานแบบอะซิงโครนัส (asynchronous) ซึ่งช่วยให้เขียนโค้ดที่รอผลลัพธ์จากการดำเนินการที่ใช้เวลานาน เช่น การเรียก API, การอ่าน/เขียนไฟล์ ได้อย่างมีประสิทธิภาพโดยไม่บล็อก thread หลัก

## 1. async { } Computation Expression

`async { }` เป็น computation expression พิเศษที่ F# มีให้ใช้สำหรับเขียนโค้ดแบบอะซิงโครนัส

```fsharp
// โครงสร้างพื้นฐาน
let basicAsync = async {
    printfn "เริ่มต้นงาน async"
    return 42
}

// การรันงาน async แบบ synchronous (บล็อก thread จนกว่าจะเสร็จ)
let result = Async.RunSynchronously basicAsync
printfn "ได้ผลลัพธ์: %d" result
```

```fsharp
// ตัวอย่างการสร้าง async workflow หลายรูปแบบ
open System

// Async ที่คืนค่า
let asyncWithReturn: Async<int> = async {
    return 100
}

// Async ที่ไม่คืนค่า (unit)
let asyncUnit: Async<unit> = async {
    printfn "ทำงานโดยไม่คืนค่า"
}

// Async ที่มีการคำนวณ
let asyncCompute (n: int): Async<int> = async {
    let result = n * n
    return result
}

// รันตัวอย่าง
let r1 = Async.RunSynchronously asyncWithReturn
let r3 = Async.RunSynchronously (asyncCompute 5)
printfn "r1 = %d, r3 = %d" r1 r3
```

## 2. let! สำหรับการรอผลลัพธ์ (Awaiting)

`let!` ใช้สำหรับรอผลลัพธ์จาก async workflow อื่นโดยไม่บล็อก thread

```fsharp
// let! ช่วยให้รอผลลัพธ์จาก async อื่น
let fetchData () = async {
    // จำลองการดึงข้อมูล
    do! Async.Sleep 100
    return "ข้อมูลที่ดึงมา"
}

let processData () = async {
    printfn "กำลังดึงข้อมูล..."
    let! data = fetchData ()          // รอผลลัพธ์
    printfn "ได้ข้อมูล: %s" data
    
    let processed = data.ToUpper()    // ประมวลผล
    return processed
}

let processed = Async.RunSynchronously (processData ())
printfn "ผลลัพธ์สุดท้าย: %s" processed
```

```fsharp
// let! หลายครั้งใน async เดียวกัน
let multipleAwait () = async {
    let! val1 = async { return 10 }
    let! val2 = async { return 20 }
    let! val3 = async { return 30 }
    return val1 + val2 + val3
}

let sum = Async.RunSynchronously (multipleAwait ())
printfn "ผลรวม: %d" sum  // 60
```

```fsharp
// ใช้ let! กับฟังก์ชันที่รับพารามิเตอร์
let delay (ms: int) (value: 'a) = async {
    do! Async.Sleep ms
    return value
}

let pipeline () = async {
    let! step1 = delay 50 "Hello"
    let! step2 = delay 50 (step1 + " World")
    let! step3 = delay 50 (step2 + "!")
    return step3
}

let final = Async.RunSynchronously (pipeline ())
printfn "Pipeline: %s" final
```

## 3. do! สำหรับการรอการดำเนินการแบบ unit

`do!` คล้ายกับ `let!` แต่ใช้เมื่อ async ที่รอนั้นคืนค่า unit

```fsharp
// do! สำหรับรอ async unit
let logMessage (msg: string) = async {
    printfn "[LOG] %s" msg
    // ในความเป็นจริงอาจเขียนไปยัง file หรือ database
}

let mainWorkflow () = async {
    do! logMessage "เริ่มต้นการทำงาน"
    do! Async.Sleep 100
    do! logMessage "กำลังดำเนินการ..."
    do! Async.Sleep 100
    do! logMessage "เสร็จสิ้นการทำงาน"
    return "สำเร็จ"
}

let result = Async.RunSynchronously (mainWorkflow ())
printfn "สถานะ: %s" result
```

```fsharp
// เปรียบเทียบ let! กับ do!
let doVsLet () = async {
    // let! - เมื่อต้องการค่าที่ return กลับมา
    let! value = async { return 42 }
    printfn "ค่าที่ได้: %d" value
    
    // do! - เมื่อไม่ต้องการค่า (unit)
    do! async { printfn "ดำเนินการเสร็จแล้ว" }
    
    // ถ้าใช้ let! กับ unit ก็ได้ แต่ไม่สื่อความหมาย
    let! () = async { printfn "อีกวิธี" }
    
    return ()
}

Async.RunSynchronously (doVsLet ())
```

## 4. return ใน async

```fsharp
// return ใช้สำหรับส่งค่ากลับจาก async workflow
let simpleReturn () = async {
    return 42
}

// return! ใช้สำหรับส่งต่อ async อื่น (เหมือน await แล้ว return)
let chainedReturn () = async {
    return! simpleReturn ()  // รอแล้วส่งค่ากลับต่อ
}

// เปรียบเทียบ return กับ return!
let returnExample () = async {
    // วิธีที่ 1: ใช้ let! แล้ว return
    let! val = simpleReturn ()
    return val
}

let returnBangExample () = async {
    // วิธีที่ 2: ใช้ return! โดยตรง (กระชับกว่า)
    return! simpleReturn ()
}

let r1 = Async.RunSynchronously (returnExample ())
let r2 = Async.RunSynchronously (returnBangExample ())
printfn "r1 = %d, r2 = %d" r1 r2
```

## 5. Async.RunSynchronously

`Async.RunSynchronously` บล็อก thread ปัจจุบันจนกว่า async workflow จะเสร็จสิ้น

```fsharp
open System

// พื้นฐาน
let syncRun () =
    let result = Async.RunSynchronously (async { return "Hello" })
    printfn "ผลลัพธ์: %s" result

// กับ timeout
let runWithTimeout () =
    let longTask = async {
        do! Async.Sleep 5000
        return "เสร็จแล้ว"
    }
    
    try
        // timeout 1 วินาที
        let result = Async.RunSynchronously(longTask, timeout = 1000)
        printfn "ผลลัพธ์: %s" result
    with
    | :? TimeoutException ->
        printfn "หมดเวลา!"

runWithTimeout ()

// กับ CancellationToken
let runWithCancellation () =
    use cts = new System.Threading.CancellationTokenSource()
    
    let task = async {
        do! Async.Sleep 100
        return "สำเร็จ"
    }
    
    let result = Async.RunSynchronously(task, cancellationToken = cts.Token)
    printfn "ผลลัพธ์: %s" result

runWithCancellation ()
```

## 6. Async.StartImmediate

`Async.StartImmediate` เริ่ม async workflow บน thread ปัจจุบัน โดยไม่รอผลลัพธ์

```fsharp
// StartImmediate เริ่มทันทีบน thread ปัจจุบัน
let immediateExample () =
    printfn "ก่อน StartImmediate"
    
    Async.StartImmediate (async {
        printfn "ใน async (ก่อน sleep)"
        do! Async.Sleep 100
        printfn "ใน async (หลัง sleep)"
    })
    
    printfn "หลัง StartImmediate"
    System.Threading.Thread.Sleep(200)

immediateExample ()
```

```fsharp
// StartImmediate กับ CancellationToken
let startImmediateWithCancel () =
    use cts = new System.Threading.CancellationTokenSource()
    
    Async.StartImmediate(
        async {
            try
                while true do
                    printfn "กำลังทำงาน..."
                    do! Async.Sleep 100
            with
            | :? System.OperationCanceledException ->
                printfn "ถูกยกเลิก!"
        },
        cancellationToken = cts.Token
    )
    
    System.Threading.Thread.Sleep(250)
    cts.Cancel()
    System.Threading.Thread.Sleep(100)

startImmediateWithCancel ()
```

## 7. Async.Start

`Async.Start` เริ่ม async workflow บน thread pool โดยไม่รอผลลัพธ์

```fsharp
// Async.Start เริ่มงานแบบ "fire and forget"
let fireAndForget () =
    printfn "เริ่มต้น"
    
    Async.Start (async {
        do! Async.Sleep 200
        printfn "งานพื้นหลังเสร็จแล้ว"
    })
    
    printfn "โปรแกรมหลักยังคงทำงานต่อ"
    System.Threading.Thread.Sleep(400)

fireAndForget ()
```

```fsharp
// Start กับ CancellationToken
let startWithToken () =
    use cts = new System.Threading.CancellationTokenSource()
    
    Async.Start(
        async {
            let mutable count = 0
            try
                while count < 10 do
                    count <- count + 1
                    printfn "รอบที่ %d" count
                    do! Async.Sleep 100
            with
            | :? System.OperationCanceledException ->
                printfn "ถูกยกเลิกที่รอบ %d" count
        },
        cancellationToken = cts.Token
    )
    
    System.Threading.Thread.Sleep(350)
    cts.Cancel()
    System.Threading.Thread.Sleep(100)

startWithToken ()
```

## 8. Async.StartAsTask

`Async.StartAsTask` แปลง Async เป็น Task สำหรับใช้กับโค้ด C# หรือ .NET API

```fsharp
open System.Threading.Tasks

// แปลง Async เป็น Task
let asyncToTask () =
    let asyncWork = async {
        do! Async.Sleep 100
        return 42
    }
    
    // แปลงเป็น Task<int>
    let task: Task<int> = Async.StartAsTask(asyncWork)
    
    // รอผลลัพธ์จาก Task
    let result = task.Result
    printfn "ผลลัพธ์จาก Task: %d" result

asyncToTask ()
```

```fsharp
// StartAsTask กับ CancellationToken
let startAsTaskWithCancel () =
    use cts = new System.Threading.CancellationTokenSource()
    
    let asyncWork = async {
        do! Async.Sleep 2000
        return "เสร็จ"
    }
    
    let task = Async.StartAsTask(asyncWork, cancellationToken = cts.Token)
    
    System.Threading.Thread.Sleep(500)
    cts.Cancel()
    
    try
        task.Wait()
    with
    | :? AggregateException as ae ->
        printfn "ถูกยกเลิก: %s" ae.InnerException.Message

startAsTaskWithCancel ()
```

## 9. Async.AwaitTask

`Async.AwaitTask` แปลง Task เป็น Async สำหรับใช้ภายใน async workflow

```fsharp
open System.Threading.Tasks

// ใช้ Async.AwaitTask เพื่อรอ Task ภายใน async
let taskToAsync () = async {
    // สร้าง Task (เช่น จาก .NET library)
    let task = Task.FromResult(42)
    
    // แปลงเป็น Async และรอ
    let! result = Async.AwaitTask task
    printfn "ผลลัพธ์: %d" result
}

Async.RunSynchronously (taskToAsync ())
```

```fsharp
// ใช้กับ Task ที่คืน unit
let taskUnitToAsync () = async {
    let task = Task.Delay(100)
    do! Async.AwaitTask task
    printfn "Task เสร็จแล้ว"
}

Async.RunSynchronously (taskUnitToAsync ())
```

```fsharp
// ตัวอย่างการรวม Task API กับ Async
open System.Net.Http

let fetchUrl (url: string) = async {
    use client = new HttpClient()
    
    // HttpClient.GetStringAsync คืน Task<string>
    let! content = client.GetStringAsync(url) |> Async.AwaitTask
    return content.Length
}

// ในบริบทจริง:
// let size = Async.RunSynchronously (fetchUrl "https://example.com")
```

## 10. Async.Parallel

`Async.Parallel` รันหลาย async workflow พร้อมกัน และรอให้ทุกอันเสร็จ

```fsharp
// รันหลายงานพร้อมกัน
let parallelExample () = async {
    let tasks = [
        async { 
            do! Async.Sleep 100
            return 1 
        }
        async { 
            do! Async.Sleep 200
            return 2 
        }
        async { 
            do! Async.Sleep 150
            return 3 
        }
    ]
    
    // รันทุกงานพร้อมกัน
    let! results = Async.Parallel tasks
    printfn "ผลลัพธ์: %A" results  // [|1; 2; 3|]
    return results
}

let results = Async.RunSynchronously (parallelExample ())
```

```fsharp
// Parallel กับงานจำนวนมาก
let massParallel () = async {
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    // สร้างงาน 100 อย่าง แต่ละอย่างใช้เวลา 50ms
    let tasks = 
        [1..100] 
        |> List.map (fun i -> async {
            do! Async.Sleep 50
            return i * i
        })
    
    let! results = Async.Parallel tasks
    sw.Stop()
    
    printfn "เสร็จภายใน %d ms (แทนที่จะเป็น %d ms)" 
        (int sw.ElapsedMilliseconds) 
        (100 * 50)
    
    return results |> Array.sum
}

let sum = Async.RunSynchronously (massParallel ())
printfn "ผลรวม: %d" sum
```

```fsharp
// Parallel กับ throttle (จำกัดจำนวน concurrent)
let parallelWithThrottle (maxConcurrent: int) (tasks: Async<'a> list) = async {
    // ใช้ Async.Parallel แต่แบ่งเป็น batch
    let batches = tasks |> List.chunkBySize maxConcurrent
    
    let results = ResizeArray<'a>()
    for batch in batches do
        let! batchResults = Async.Parallel batch
        results.AddRange(batchResults)
    
    return results |> Seq.toArray
}

// ทดสอบ
let throttledExample () = async {
    let tasks = 
        [1..20] 
        |> List.map (fun i -> async {
            do! Async.Sleep 100
            printfn "เสร็จงาน %d" i
            return i
        })
    
    return! parallelWithThrottle 5 tasks
}

let results2 = Async.RunSynchronously (throttledExample ())
printfn "ผลลัพธ์: %d งาน" results2.Length
```

## 11. Async.Sequential

`Async.Sequential` รัน async workflow ทีละอัน ตามลำดับ

```fsharp
// รันงานทีละอัน
let sequentialExample () = async {
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    let tasks = [
        async { 
            do! Async.Sleep 100
            printfn "งาน 1 เสร็จ"
            return 1 
        }
        async { 
            do! Async.Sleep 100
            printfn "งาน 2 เสร็จ"
            return 2 
        }
        async { 
            do! Async.Sleep 100
            printfn "งาน 3 เสร็จ"
            return 3 
        }
    ]
    
    // รันทีละอัน
    let! results = Async.Sequential tasks
    sw.Stop()
    
    printfn "ใช้เวลา: %d ms" (int sw.ElapsedMilliseconds)
    return results
}

let seqResults = Async.RunSynchronously (sequentialExample ())
printfn "ผลลัพธ์: %A" seqResults
```

```fsharp
// เปรียบเทียบ Sequential vs Parallel
let compareTime () = 
    let tasks = [1..5] |> List.map (fun i -> async {
        do! Async.Sleep 200
        return i
    })
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let seqResult = Async.RunSynchronously(Async.Sequential tasks)
    let seqTime = sw.ElapsedMilliseconds
    
    sw.Restart()
    let parResult = Async.RunSynchronously(Async.Parallel tasks)
    let parTime = sw.ElapsedMilliseconds
    
    printfn "Sequential: %d ms" seqTime
    printfn "Parallel: %d ms" parTime
    printfn "Parallel เร็วกว่า ~%.1f เท่า" (float seqTime / float parTime)

compareTime ()
```

## 12. Async.map (Custom Implementation)

F# ไม่มี `Async.map` built-in แต่เราสามารถสร้างได้

```fsharp
// Async.map - แปลงค่าภายใน Async
module Async =
    let map (f: 'a -> 'b) (a: Async<'a>) : Async<'b> = async {
        let! value = a
        return f value
    }
    
    // หรือเขียนแบบสั้น
    let map' f a = async {
        let! x = a
        return f x
    }
```

```fsharp
// ใช้งาน Async.map
let mapExample () =
    let asyncValue = async { return 42 }
    
    // แปลงค่าโดยไม่ต้อง unwrap
    let doubled = Async.map (fun x -> x * 2) asyncValue
    let asString = Async.map string asyncValue
    let withContext = Async.map (fun x -> sprintf "ค่าคือ %d" x) asyncValue
    
    printfn "doubled: %d" (Async.RunSynchronously doubled)
    printfn "asString: %s" (Async.RunSynchronously asString)
    printfn "withContext: %s" (Async.RunSynchronously withContext)

mapExample ()
```

```fsharp
// Async.map ใช้ใน pipeline
let mapPipeline () =
    async { return "  hello world  " }
    |> Async.map (fun s -> s.Trim())
    |> Async.map (fun s -> s.ToUpper())
    |> Async.map (fun s -> sprintf "[%s]" s)
    |> Async.RunSynchronously
    |> printfn "ผลลัพธ์: %s"

mapPipeline ()
```

## 13. Async.bind (Custom Implementation)

```fsharp
// Async.bind - chain async operations
module AsyncExtra =
    let bind (f: 'a -> Async<'b>) (a: Async<'a>) : Async<'b> = async {
        let! value = a
        return! f value
    }
    
    // Async.apply - สำหรับ applicative style
    let apply (f: Async<'a -> 'b>) (a: Async<'a>) : Async<'b> = async {
        let! func = f
        let! value = a
        return func value
    }
```

```fsharp
// ใช้งาน bind
let bindExample () =
    let asyncInt = async { return 10 }
    
    // bind ให้สามารถ chain async ได้
    let result = 
        asyncInt 
        |> AsyncExtra.bind (fun x -> async { return x * 2 })
        |> AsyncExtra.bind (fun x -> async { return sprintf "ผลลัพธ์: %d" x })
    
    Async.RunSynchronously result |> printfn "%s"

bindExample ()
```

```fsharp
// bind สำหรับ error handling pattern
type AsyncResult<'a, 'e> = Async<Result<'a, 'e>>

module AsyncResult =
    let bind (f: 'a -> AsyncResult<'b, 'e>) (a: AsyncResult<'a, 'e>) : AsyncResult<'b, 'e> = async {
        let! result = a
        match result with
        | Ok value -> return! f value
        | Error e -> return Error e
    }
    
    let map (f: 'a -> 'b) (a: AsyncResult<'a, 'e>) : AsyncResult<'b, 'e> = async {
        let! result = a
        return Result.map f result
    }
    
    let return' value : AsyncResult<'a, 'e> = async {
        return Ok value
    }
```

```fsharp
// ใช้งาน AsyncResult
let asyncResultExample () =
    let step1 : AsyncResult<int, string> = async {
        do! Async.Sleep 50
        return Ok 42
    }
    
    let step2 (x: int) : AsyncResult<string, string> = async {
        if x > 0 then
            return Ok (sprintf "ค่าบวก: %d" x)
        else
            return Error "ค่าต้องเป็นบวก"
    }
    
    let result = 
        step1 
        |> AsyncResult.bind step2
        |> Async.RunSynchronously
    
    match result with
    | Ok msg -> printfn "สำเร็จ: %s" msg
    | Error e -> printfn "ล้มเหลว: %s" e

asyncResultExample ()
```

## 14. CancellationToken Usage

```fsharp
open System.Threading

// การใช้ CancellationToken
let cancellationExample () =
    use cts = new CancellationTokenSource()
    
    let longRunning = async {
        printfn "เริ่มงานยาว"
        for i in 1..10 do
            // ตรวจสอบการยกเลิกเป็นระยะ
            do! Async.Sleep 200
            printfn "ขั้นตอน %d เสร็จ" i
        return "เสร็จสิ้น"
    }
    
    // เริ่มงาน
    Async.Start(longRunning, cancellationToken = cts.Token)
    
    // ยกเลิกหลัง 600ms
    Thread.Sleep(600)
    cts.Cancel()
    
    Thread.Sleep(200)
    printfn "ยกเลิกแล้ว"

cancellationExample ()
```

```fsharp
// การส่ง CancellationToken ผ่าน Async.CancellationToken
let withCancellationToken () = async {
    // อ่าน token ปัจจุบัน
    let! ct = Async.CancellationToken
    
    printfn "ได้รับ CancellationToken"
    
    // ส่งต่อไปยัง .NET API
    // cts.Cancel() จะยกเลิก operation นี้ได้
    use timer = new System.Timers.Timer(100.0)
    timer.AutoReset <- false
    timer.Start()
    
    return ct.IsCancellationRequested
}

let tokenResult = Async.RunSynchronously (withCancellationToken ())
printfn "ถูกยกเลิก: %b" tokenResult
```

```fsharp
// CancellationTokenSource.CancelAfter
let autoCancel () =
    use cts = new CancellationTokenSource()
    cts.CancelAfter(300)  // ยกเลิกอัตโนมัติหลัง 300ms
    
    let work = async {
        try
            let mutable i = 0
            while true do
                i <- i + 1
                printfn "ทำงานรอบ %d" i
                do! Async.Sleep 100
        with
        | :? OperationCanceledException ->
            printfn "ถูกยกเลิกอัตโนมัติ!"
    }
    
    try
        Async.RunSynchronously(work, cancellationToken = cts.Token)
    with
    | :? OperationCanceledException -> ()

autoCancel ()
```

## 15. Async.TryCancelled

```fsharp
// Async.TryCancelled - จัดการเมื่อถูกยกเลิก
let tryCancelledExample () =
    use cts = new CancellationTokenSource()
    
    let work = async {
        do! Async.Sleep 1000
        return "เสร็จ"
    }
    
    // ห่อด้วย TryCancelled เพื่อจัดการการยกเลิก
    let handled = Async.TryCancelled(
        work,
        fun cancelEx ->
            printfn "งานถูกยกเลิก: %s" cancelEx.Message
    )
    
    Async.Start(handled, cancellationToken = cts.Token)
    Thread.Sleep(200)
    cts.Cancel()
    Thread.Sleep(100)

tryCancelledExample ()
```

```fsharp
// TryCancelled กับการล้างทรัพยากร
let cleanupOnCancel () =
    use cts = new CancellationTokenSource()
    
    let resourceWork = async {
        printfn "เปิดทรัพยากร"
        try
            do! Async.Sleep 2000
            printfn "งานเสร็จ"
        finally
            printfn "ปิดทรัพยากร (cleanup)"
    }
    
    let handled = Async.TryCancelled(
        resourceWork,
        fun _ -> printfn "ได้รับสัญญาณยกเลิก"
    )
    
    Async.Start(handled, cancellationToken = cts.Token)
    Thread.Sleep(300)
    cts.Cancel()
    Thread.Sleep(300)

cleanupOnCancel ()
```

## 16. Timeout Pattern

```fsharp
// Pattern: timeout ด้วย Async.Choice
let withTimeout (timeout: int) (work: Async<'a>) : Async<'a option> = async {
    let! result = Async.Choice [
        async {
            let! value = work
            return Some value
        }
        async {
            do! Async.Sleep timeout
            return Some None  // timeout
        }
    ]
    return result |> Option.defaultValue None
}
```

```fsharp
// ใช้งาน timeout pattern
let timeoutExample () = async {
    let slowTask = async {
        do! Async.Sleep 500
        return "เสร็จช้า"
    }
    
    let fastTask = async {
        do! Async.Sleep 100
        return "เสร็จเร็ว"
    }
    
    // ทดสอบ timeout
    let! result1 = withTimeout 200 slowTask  // timeout -> None
    let! result2 = withTimeout 200 fastTask  // สำเร็จ -> Some "เสร็จเร็ว"
    
    printfn "งานช้า: %A" result1   // None
    printfn "งานเร็ว: %A" result2  // Some "เสร็จเร็ว"
}

Async.RunSynchronously (timeoutExample ())
```

```fsharp
// Timeout pattern ด้วย CancellationTokenSource
let withTimeoutV2 (timeoutMs: int) (work: Async<'a>) : Async<Result<'a, string>> = async {
    use cts = new CancellationTokenSource(timeoutMs)
    
    try
        let! result = Async.StartChild(work, timeoutMs)
        let! value = result
        return Ok value
    with
    | :? System.TimeoutException ->
        return Error "หมดเวลา"
    | :? OperationCanceledException ->
        return Error "ถูกยกเลิก"
}

// ทดสอบ
let testTimeout () = async {
    let longTask = async {
        do! Async.Sleep 1000
        return 42
    }
    
    let! result = withTimeoutV2 200 longTask
    match result with
    | Ok v -> printfn "สำเร็จ: %d" v
    | Error msg -> printfn "ล้มเหลว: %s" msg
}

Async.RunSynchronously (testTimeout ())
```

## 17. Error Handling ใน Async

```fsharp
// try/with ใน async
let asyncWithErrorHandling () = async {
    try
        let! result = async {
            // จำลองข้อผิดพลาด
            if true then
                raise (System.Exception("เกิดข้อผิดพลาด!"))
            return 42
        }
        return Ok result
    with
    | ex ->
        printfn "จับข้อผิดพลาด: %s" ex.Message
        return Error ex.Message
}

let handled = Async.RunSynchronously (asyncWithErrorHandling ())
printfn "ผลลัพธ์: %A" handled
```

```fsharp
// try/finally ใน async
let asyncWithFinally () = async {
    let resource = "resource"
    try
        printfn "ใช้ %s" resource
        do! Async.Sleep 100
        // อาจเกิด exception
        return "เสร็จ"
    finally
        // ทำงานเสมอ ไม่ว่าจะสำเร็จหรือล้มเหลว
        printfn "ปล่อย %s" resource
}

let r = Async.RunSynchronously (asyncWithFinally ())
printfn "ผลลัพธ์: %s" r
```

```fsharp
// Async.Catch - แปลง exception เป็น Choice
let asyncCatchExample () = async {
    let riskyWork = async {
        raise (System.Exception("บัง!"))
        return 42
    }
    
    // Async.Catch จะดักจับ exception
    let! result = Async.Catch riskyWork
    
    match result with
    | Choice1Of2 value ->
        printfn "สำเร็จ: %d" value
    | Choice2Of2 ex ->
        printfn "ล้มเหลว: %s" ex.Message
}

Async.RunSynchronously (asyncCatchExample ())
```

## 18. Async.StartChild

```fsharp
// StartChild เพื่อรัน async แบบ concurrent ใน async
let startChildExample () = async {
    // เริ่มงานลูก
    let! child1 = Async.StartChild (async {
        do! Async.Sleep 200
        return "งานลูก 1"
    })
    
    let! child2 = Async.StartChild (async {
        do! Async.Sleep 100
        return "งานลูก 2"
    })
    
    // ทำงานอื่นระหว่างรอ
    printfn "ทำงานหลักระหว่างรองาน..."
    do! Async.Sleep 50
    
    // รอผลลัพธ์
    let! result1 = child1
    let! result2 = child2
    
    printfn "%s เสร็จ" result1
    printfn "%s เสร็จ" result2
    
    return (result1, result2)
}

let (r1, r2) = Async.RunSynchronously (startChildExample ())
printfn "ผลลัพธ์: %s, %s" r1 r2
```

## 19. ตัวอย่างจริง: HTTP Calls

```fsharp
open System.Net.Http
open System.Text.Json

// การเรียก HTTP แบบ async
module HttpExample =
    
    // Download ข้อมูลจาก URL
    let downloadString (url: string) = async {
        use client = new HttpClient()
        client.DefaultRequestHeaders.Add("User-Agent", "FSharp-Async-Example/1.0")
        
        printfn "กำลังดาวน์โหลด: %s" url
        let! response = client.GetStringAsync(url) |> Async.AwaitTask
        printfn "ดาวน์โหลดเสร็จ (%d ตัวอักษร)" response.Length
        return response
    }
    
    // ดาวน์โหลดหลาย URL พร้อมกัน
    let downloadAll (urls: string list) = async {
        let tasks = urls |> List.map downloadString
        return! Async.Parallel tasks
    }
    
    // ดาวน์โหลดพร้อม retry
    let rec downloadWithRetry (maxRetries: int) (url: string) = async {
        try
            return! downloadString url
        with
        | ex when maxRetries > 0 ->
            printfn "ล้มเหลว จะลองใหม่ (%d ครั้งที่เหลือ): %s" maxRetries ex.Message
            do! Async.Sleep 1000
            return! downloadWithRetry (maxRetries - 1) url
        | ex ->
            return raise ex
    }
```

```fsharp
// POST request
module HttpPost =
    open System.Text
    
    let postJson (url: string) (json: string) = async {
        use client = new HttpClient()
        use content = new StringContent(json, Encoding.UTF8, "application/json")
        
        let! response = client.PostAsync(url, content) |> Async.AwaitTask
        response.EnsureSuccessStatusCode() |> ignore
        
        let! responseBody = response.Content.ReadAsStringAsync() |> Async.AwaitTask
        return responseBody
    }
    
    // สร้าง request แบบ type-safe
    type CreateUserRequest = {
        Name: string
        Email: string
    }
    
    let createUser (baseUrl: string) (request: CreateUserRequest) = async {
        let json = System.Text.Json.JsonSerializer.Serialize(request)
        return! postJson (baseUrl + "/users") json
    }
```

## 20. ตัวอย่างจริง: File I/O

```fsharp
open System.IO

// การอ่านเขียนไฟล์แบบ async
module FileIO =
    
    // อ่านไฟล์
    let readFileAsync (path: string) = async {
        let! content = File.ReadAllTextAsync(path) |> Async.AwaitTask
        return content
    }
    
    // เขียนไฟล์
    let writeFileAsync (path: string) (content: string) = async {
        do! File.WriteAllTextAsync(path, content) |> Async.AwaitTask
        printfn "เขียนไฟล์เสร็จ: %s" path
    }
    
    // อ่านทีละบรรทัด (เหมาะกับไฟล์ขนาดใหญ่)
    let readLinesAsync (path: string) = async {
        let! lines = File.ReadAllLinesAsync(path) |> Async.AwaitTask
        return lines
    }
    
    // คัดลอกไฟล์
    let copyFileAsync (source: string) (dest: string) = async {
        use sourceStream = File.OpenRead(source)
        use destStream = File.Create(dest)
        do! sourceStream.CopyToAsync(destStream) |> Async.AwaitTask
        printfn "คัดลอกไฟล์เสร็จ: %s -> %s" source dest
    }
```

```fsharp
// ตัวอย่างการใช้งาน File I/O
let fileIOExample () = async {
    let tempPath = Path.GetTempPath()
    let testFile = Path.Combine(tempPath, "test_fsharp_async.txt")
    
    // เขียนไฟล์
    do! FileIO.writeFileAsync testFile "Hello, F# Async!\nบรรทัดสอง\nบรรทัดสาม"
    
    // อ่านไฟล์
    let! content = FileIO.readFileAsync testFile
    printfn "เนื้อหาไฟล์:\n%s" content
    
    // อ่านทีละบรรทัด
    let! lines = FileIO.readLinesAsync testFile
    printfn "จำนวนบรรทัด: %d" lines.Length
    for i, line in lines |> Array.indexed do
        printfn "  บรรทัด %d: %s" (i + 1) line
    
    // ลบไฟล์ทดสอบ
    File.Delete(testFile)
    printfn "ลบไฟล์ทดสอบแล้ว"
}

Async.RunSynchronously (fileIOExample ())
```

## 21. Async.Choice

`Async.Choice` รัน async หลายอันพร้อมกัน คืนค่าแรกที่เสร็จก่อน (และไม่ใช่ None)

```fsharp
// Async.Choice - คืนค่าแรกที่เสร็จ
let choiceExample () = async {
    let fast = async {
        do! Async.Sleep 100
        return Some "เร็ว"
    }
    
    let slow = async {
        do! Async.Sleep 1000
        return Some "ช้า"
    }
    
    // จะได้ค่าจาก fast
    let! winner = Async.Choice [fast; slow]
    printfn "ผู้ชนะ: %A" winner
}

Async.RunSynchronously (choiceExample ())
```

```fsharp
// ใช้ Choice สำหรับ fallback pattern
let withFallback (primary: Async<'a option>) (fallback: Async<'a option>) = async {
    return! Async.Choice [primary; fallback]
}

// ตัวอย่าง: ลองหลาย endpoints
let tryEndpoints () = async {
    let endpoint1 = async {
        do! Async.Sleep 500  // ช้า
        return Some "endpoint1"
    }
    
    let endpoint2 = async {
        do! Async.Sleep 100  // เร็ว
        return Some "endpoint2"
    }
    
    let! result = Async.Choice [endpoint1; endpoint2]
    printfn "ใช้ endpoint: %A" result
}

Async.RunSynchronously (tryEndpoints ())
```

## 22. Async Workflow กับ Mutable State

```fsharp
// ระวัง: mutable state ใน async
let mutableStateExample () =
    let mutable counter = 0
    
    let increment () = async {
        // ไม่ thread-safe!
        let current = counter
        do! Async.Sleep 10
        counter <- current + 1
    }
    
    // รัน 100 increments พร้อมกัน
    let tasks = [1..100] |> List.map (fun _ -> increment ())
    Async.RunSynchronously (Async.Parallel tasks) |> ignore
    
    printfn "คาดหวัง: 100, ได้จริง: %d (อาจไม่ถูกต้อง)" counter
```

```fsharp
// แก้ด้วย Interlocked
open System.Threading

let threadSafeCounter () =
    let mutable counter = 0
    
    let increment () = async {
        Interlocked.Increment(&counter) |> ignore
    }
    
    let tasks = [1..100] |> List.map (fun _ -> increment ())
    Async.RunSynchronously (Async.Parallel tasks) |> ignore
    
    printfn "ผลลัพธ์ถูกต้อง: %d" counter  // 100

threadSafeCounter ()
```

## 23. ตัวอย่างครบ: Async Data Processing Pipeline

```fsharp
open System
open System.IO

// Pipeline สำหรับประมวลผลข้อมูลแบบ async
module DataPipeline =
    
    type Record = {
        Id: int
        Value: float
        Category: string
    }
    
    // จำลองการดึงข้อมูลจาก database
    let fetchRecords (count: int) = async {
        do! Async.Sleep 100  // จำลอง I/O delay
        return [1..count] |> List.map (fun i -> {
            Id = i
            Value = float i * 1.5
            Category = if i % 2 = 0 then "A" else "B"
        })
    }
    
    // ประมวลผลแต่ละ record
    let processRecord (record: Record) = async {
        do! Async.Sleep 10  // จำลองการประมวลผล
        return { record with Value = record.Value * 2.0 }
    }
    
    // กรองตาม category
    let filterByCategory (cat: string) (records: Record list) = async {
        return records |> List.filter (fun r -> r.Category = cat)
    }
    
    // บันทึกผลลัพธ์
    let saveResults (records: Record list) = async {
        let path = Path.Combine(Path.GetTempPath(), "pipeline_result.txt")
        let content = 
            records 
            |> List.map (fun r -> sprintf "Id=%d, Value=%.2f, Cat=%s" r.Id r.Value r.Category)
            |> String.concat "\n"
        do! File.WriteAllTextAsync(path, content) |> Async.AwaitTask
        printfn "บันทึก %d records ไปที่ %s" records.Length path
        return path
    }
    
    // Pipeline หลัก
    let run (count: int) = async {
        printfn "เริ่ม pipeline..."
        
        // ขั้นที่ 1: ดึงข้อมูล
        let! records = fetchRecords count
        printfn "ดึงข้อมูลแล้ว: %d records" records.Length
        
        // ขั้นที่ 2: ประมวลผลแบบ parallel
        let! processed = 
            records 
            |> List.map processRecord 
            |> Async.Parallel
        printfn "ประมวลผลแล้ว: %d records" processed.Length
        
        // ขั้นที่ 3: กรองและแยก
        let! catA = filterByCategory "A" (processed |> Array.toList)
        let! catB = filterByCategory "B" (processed |> Array.toList)
        printfn "Category A: %d, B: %d" catA.Length catB.Length
        
        // ขั้นที่ 4: บันทึกผลลัพธ์
        let! path = saveResults (catA @ catB)
        
        return {|
            Total = processed.Length
            CategoryA = catA.Length
            CategoryB = catB.Length
            OutputPath = path
        |}
    }
```

```fsharp
// รัน pipeline
let pipelineResult = Async.RunSynchronously (DataPipeline.run 50)
printfn "\nสรุปผล Pipeline:"
printfn "  รวม: %d records" pipelineResult.Total
printfn "  Category A: %d" pipelineResult.CategoryA
printfn "  Category B: %d" pipelineResult.CategoryB
printfn "  ไฟล์ผลลัพธ์: %s" pipelineResult.OutputPath
```

## 24. ตัวอย่างครบ: Async Retry Logic

```fsharp
// Retry logic สำหรับ async operations
module AsyncRetry =
    
    type RetryPolicy = {
        MaxRetries: int
        Delay: int  // milliseconds
        BackoffMultiplier: float
    }
    
    let defaultPolicy = {
        MaxRetries = 3
        Delay = 1000
        BackoffMultiplier = 2.0
    }
    
    // ลองทำงานซ้ำเมื่อล้มเหลว
    let rec retry (policy: RetryPolicy) (attempt: int) (work: Async<'a>) = async {
        try
            return! work
        with
        | ex when attempt < policy.MaxRetries ->
            let delay = int (float policy.Delay * Math.Pow(policy.BackoffMultiplier, float attempt))
            printfn "ครั้งที่ %d ล้มเหลว: %s (รอ %dms ก่อนลองใหม่)" (attempt + 1) ex.Message delay
            do! Async.Sleep delay
            return! retry policy (attempt + 1) work
        | ex ->
            printfn "ล้มเหลวทุกครั้ง (%d ครั้ง)" (attempt + 1)
            return raise ex
    }
    
    let withRetry (policy: RetryPolicy) (work: Async<'a>) =
        retry policy 0 work
```

```fsharp
// ทดสอบ retry
let retryExample () = async {
    let mutable callCount = 0
    
    let unreliableService () = async {
        callCount <- callCount + 1
        if callCount < 3 then
            raise (Exception(sprintf "Service ไม่พร้อม (ครั้งที่ %d)" callCount))
        return "สำเร็จ!"
    }
    
    let policy = { AsyncRetry.defaultPolicy with MaxRetries = 5; Delay = 100 }
    
    try
        let! result = AsyncRetry.withRetry policy (unreliableService ())
        printfn "ผลลัพธ์: %s (เรียก %d ครั้ง)" result callCount
    with
    | ex ->
        printfn "ล้มเหลว: %s" ex.Message
}

Async.RunSynchronously (retryExample ())
```

## 25. สรุปและ Best Practices

```fsharp
// Best Practices สำหรับ Async ใน F#

// 1. ใช้ async { } สำหรับ I/O-bound operations
let goodAsyncUsage () = async {
    // ดี: รอ I/O โดยไม่บล็อก thread
    let! content = File.ReadAllTextAsync("file.txt") |> Async.AwaitTask
    return content.Length
}

// 2. ใช้ Async.Parallel เมื่องานเป็นอิสระต่อกัน
let independentTasks () = async {
    let! results = 
        [1..10] 
        |> List.map (fun i -> async { return i * i })
        |> Async.Parallel
    return results

}

// 3. ใช้ CancellationToken เสมอสำหรับงานที่อาจถูกยกเลิก
let cancellableWork (ct: CancellationToken) = async {
    let! _ = Async.CancellationToken
    // pass token to underlying operations
    return ()
}

// 4. จัดการ exception อย่างเหมาะสม
let properErrorHandling () = async {
    try
        let! result = async { return 42 }
        return Ok result
    with
    | ex -> return Error ex.Message
}

// 5. ไม่ใช้ .Wait() หรือ .Result บน Task (อาจเกิด deadlock)
// ใช้ Async.AwaitTask แทน
let safeTaskUsage () = async {
    let task = Task.FromResult(42)
    let! result = Async.AwaitTask task  // ถูกต้อง
    return result
}

printfn "Async Workflows - สรุปเสร็จ!"
```

## สรุปคำสั่งสำคัญ

| คำสั่ง | ใช้งาน |
|--------|--------|
| `async { }` | สร้าง async workflow |
| `let!` | รอผลลัพธ์จาก async |
| `do!` | รอ async unit |
| `return` | ส่งค่ากลับ |
| `return!` | ส่งต่อ async อื่น |
| `Async.RunSynchronously` | รัน async แบบ sync (บล็อก) |
| `Async.Start` | เริ่มแบบ fire-and-forget |
| `Async.StartAsTask` | แปลงเป็น Task |
| `Async.AwaitTask` | รอ Task ใน async |
| `Async.Parallel` | รันพร้อมกัน |
| `Async.Sequential` | รันตามลำดับ |
| `Async.Choice` | รับผลแรกที่เสร็จ |
| `Async.TryCancelled` | จัดการการยกเลิก |
| `Async.Catch` | ดักจับ exception |
