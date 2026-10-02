# Part 43 - การเขียนโปรแกรมแบบขนาน (Parallel Programming)

## บทนำ

Parallel Programming คือการรันโค้ดหลายส่วนพร้อมกันบน CPU หลาย core เพื่อเพิ่มประสิทธิภาพ .NET มี Task Parallel Library (TPL) ที่ทรงพลังสำหรับงานนี้ และ F# มี wrapper ที่ใช้งานง่าย

## 1. Array.Parallel.map

```fsharp
// Array.Parallel.map - ประมวลผลแต่ละ element แบบ parallel
let basicParallelMap () =
    let data = [|1..100|]
    
    // Sequential
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let seqResult = data |> Array.map (fun x -> 
        System.Threading.Thread.Sleep(5)
        x * x)
    sw.Stop()
    printfn "Sequential: %d ms" sw.ElapsedMilliseconds
    
    // Parallel
    sw.Restart()
    let parResult = data |> Array.Parallel.map (fun x ->
        System.Threading.Thread.Sleep(5)
        x * x)
    sw.Stop()
    printfn "Parallel: %d ms" sw.ElapsedMilliseconds
    
    printfn "ผลลัพธ์เหมือนกัน: %b" (seqResult = parResult)

basicParallelMap ()
```

```fsharp
// Array.Parallel.map สำหรับการประมวลผลหนัก
let heavyProcessing () =
    let data = [|1..20|]
    
    let cpuIntensive (n: int) =
        // จำลองงาน CPU-intensive
        let mutable sum = 0.0
        for i in 1..1000000 do
            sum <- sum + System.Math.Sqrt(float i * float n)
        sum
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let results = data |> Array.Parallel.map cpuIntensive
    sw.Stop()
    
    printfn "Parallel heavy processing: %d ms" sw.ElapsedMilliseconds
    printfn "ผลลัพธ์แรก: %.2f" results[0]

heavyProcessing ()
```

```fsharp
// Array.Parallel.iter - ทำ side effect แบบ parallel
let parallelIter () =
    let data = [|1..10|]
    
    let mutable processed = 0
    let lockObj = obj()
    
    data |> Array.Parallel.iter (fun x ->
        System.Threading.Thread.Sleep(50)
        lock lockObj (fun () ->
            processed <- processed + 1
            printfn "ประมวลผล %d (รวม: %d)" x processed)
    )

parallelIter ()
```

## 2. Seq.Parallel ผ่าน PLINQ (AsParallel)

```fsharp
open System.Linq

// ใช้ PLINQ ผ่าน AsParallel()
let plinqExample () =
    let data = seq { 1..1000 }
    
    // AsParallel() เปิดใช้ PLINQ
    let result = 
        data.AsParallel()
            .Where(fun x -> x % 2 = 0)
            .Select(fun x -> x * x)
            .ToArray()
    
    printfn "ผลลัพธ์ %d items" result.Length
    printfn "ผลรวม: %d" (result |> Array.sum)
```

```fsharp
// PLINQ กับ WithDegreeOfParallelism
let plinqWithDegree () =
    let data = [|1..100|]
    
    let result = 
        data.AsParallel()
            .WithDegreeOfParallelism(4)  // จำกัด 4 threads
            .Select(fun x ->
                System.Threading.Thread.Sleep(10)
                x * x)
            .ToArray()
    
    printfn "ผลลัพธ์: %d items" result.Length
```

```fsharp
// PLINQ กับ AsOrdered (รักษาลำดับ)
let plinqOrdered () =
    let data = [|1..20|]
    
    // AsOrdered() รักษาลำดับ input
    let result = 
        data.AsParallel()
            .AsOrdered()
            .Select(fun x -> x * 2)
            .ToArray()
    
    printfn "ผลลัพธ์มีลำดับ: %b" (result = data |> Array.map (fun x -> x * 2))
```

```fsharp
// PLINQ กับ Aggregation
let plinqAggregate () =
    let data = [|1..1000000|]
    
    // Parallel Sum
    let sum = data.AsParallel().Sum(fun x -> int64 x)
    printfn "Parallel sum: %d" sum
    
    // Parallel Count
    let evenCount = data.AsParallel().Count(fun x -> x % 2 = 0)
    printfn "จำนวนเลขคู่: %d" evenCount
```

## 3. Parallel.For และ Parallel.ForEach

```fsharp
open System.Threading.Tasks

// Parallel.For - รัน for loop แบบ parallel
let parallelForExample () =
    let results = Array.zeroCreate 10
    
    Parallel.For(0, 10, fun i ->
        System.Threading.Thread.Sleep(100)
        results[i] <- i * i
    ) |> ignore
    
    printfn "ผลลัพธ์: %A" results
```

```fsharp
// Parallel.For กับ ParallelOptions
let parallelForWithOptions () =
    use cts = new System.Threading.CancellationTokenSource()
    
    let options = ParallelOptions(
        MaxDegreeOfParallelism = 4,
        CancellationToken = cts.Token
    )
    
    let mutable processed = 0
    let lockObj = obj()
    
    try
        Parallel.For(0, 100, options, fun i ->
            if i = 50 then
                cts.Cancel()
                cts.Token.ThrowIfCancellationRequested()
            
            lock lockObj (fun () -> processed <- processed + 1)
        ) |> ignore
    with
    | :? OperationCanceledException ->
        printfn "ยกเลิกหลังประมวลผล %d items" processed
```

```fsharp
// Parallel.ForEach
let parallelForEachExample () =
    let data = [1..20] |> List.toArray
    
    let mutable sum = 0
    let lockObj = obj()
    
    Parallel.ForEach(data, fun item ->
        System.Threading.Thread.Sleep(20)
        lock lockObj (fun () -> sum <- sum + item)
    ) |> ignore
    
    printfn "ผลรวม: %d (คาดหวัง: %d)" sum (Array.sum data)
```

```fsharp
// Parallel.ForEach กับ local state (ป้องกัน lock)
let parallelForEachLocalState () =
    let data = [|1..1000|]
    
    // ใช้ local state เพื่อหลีกเลี่ยง contention
    let totalSum = 
        Parallel.ForEach(
            data,
            fun () -> 0L,           // Initialize local state
            fun item state localSum -> // Body
                localSum + int64 item,
            fun localSum ->          // Finalize
                System.Threading.Interlocked.Add(ref 0L, localSum) |> ignore
        )
    
    // วิธีอื่น: ใช้ PLINQ
    let sum = data.AsParallel().Sum(int64)
    printfn "ผลรวม: %d" sum
```

## 4. Task Parallel Library (TPL)

```fsharp
open System.Threading.Tasks

// Task.Run สำหรับ CPU-bound work
let tplExample () =
    let tasks = [|
        Task.Run(fun () ->
            System.Threading.Thread.Sleep(100)
            printfn "Task 1 on thread %d" System.Threading.Thread.CurrentThread.ManagedThreadId
            1)
        Task.Run(fun () ->
            System.Threading.Thread.Sleep(200)
            printfn "Task 2 on thread %d" System.Threading.Thread.CurrentThread.ManagedThreadId
            2)
        Task.Run(fun () ->
            System.Threading.Thread.Sleep(150)
            printfn "Task 3 on thread %d" System.Threading.Thread.CurrentThread.ManagedThreadId
            3)
    |]
    
    Task.WhenAll(tasks).Wait()
    printfn "ทุก task เสร็จ"
```

```fsharp
// Task Continuations
let taskContinuations () =
    let firstTask = Task.Run(fun () ->
        System.Threading.Thread.Sleep(100)
        printfn "Task แรก"
        42)
    
    // ทำงานต่อเมื่อ task แรกเสร็จ
    let continuation = firstTask.ContinueWith(fun (t: Task<int>) ->
        printfn "ค่าจาก task แรก: %d" t.Result
        t.Result * 2)
    
    continuation.Wait()
    printfn "ผลลัพธ์สุดท้าย: %d" continuation.Result
```

```fsharp
// TaskFactory สำหรับ configuration ขั้นสูง
let taskFactoryExample () =
    let factory = TaskFactory(
        System.Threading.CancellationToken.None,
        TaskCreationOptions.LongRunning,
        TaskContinuationOptions.None,
        TaskScheduler.Default
    )
    
    let longRunningTask = factory.StartNew(fun () ->
        // LongRunning hint ทำให้ .NET ใช้ dedicated thread
        System.Threading.Thread.Sleep(200)
        printfn "Long running task บน thread %d" System.Threading.Thread.CurrentThread.ManagedThreadId
        "เสร็จแล้ว"
    )
    
    longRunningTask.Wait()
    printfn "ผลลัพธ์: %s" longRunningTask.Result
```

## 5. Data Parallelism

```fsharp
// Data parallelism: ประมวลผลชุดข้อมูลแบบ parallel
let dataParallelism () =
    let data = [|1..1_000_000|]
    
    // Sequential
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let seqSum = data |> Array.sumBy (fun x -> 
        System.Math.Sqrt(float x))
    sw.Stop()
    printfn "Sequential: %d ms, sum = %.2f" sw.ElapsedMilliseconds seqSum
    
    // Parallel with chunk processing
    sw.Restart()
    let cpuCount = System.Environment.ProcessorCount
    let chunkSize = data.Length / cpuCount
    
    let partialSums =
        [|0..cpuCount-1|]
        |> Array.Parallel.map (fun i ->
            let start = i * chunkSize
            let end_ = if i = cpuCount - 1 then data.Length else (i + 1) * chunkSize
            let mutable sum = 0.0
            for j in start..end_-1 do
                sum <- sum + System.Math.Sqrt(float data[j])
            sum)
    
    let parSum = partialSums |> Array.sum
    sw.Stop()
    printfn "Parallel: %d ms, sum = %.2f" sw.ElapsedMilliseconds parSum
```

```fsharp
// Map-Reduce pattern
let mapReduce (mapper: 'a -> 'b) (reducer: 'b -> 'b -> 'b) (identity: 'b) (data: 'a[]) =
    data
    |> Array.Parallel.map mapper
    |> Array.fold reducer identity

// ตัวอย่าง word count
let wordCount (texts: string[]) =
    let countWords (text: string) =
        text.Split(' ', System.StringSplitOptions.RemoveEmptyEntries).Length
    
    mapReduce countWords (+) 0 texts

let texts = [|
    "สวัสดี โลก F Sharp"
    "การเขียนโปรแกรม เชิงฟังก์ชัน"
    "ขนาน และ อะซิงโครนัส"
|]

printfn "จำนวนคำทั้งหมด: %d" (wordCount texts)
```

## 6. Task Parallelism

```fsharp
// Task parallelism: รันงานต่างๆ พร้อมกัน
let taskParallelism () =
    // งานอิสระที่รันพร้อมกัน
    let fetchData () = async {
        do! Async.Sleep 200
        return [| 1; 2; 3 |]
    }
    
    let processConfig () = async {
        do! Async.Sleep 100
        return {| DatabaseUrl = "localhost"; MaxConnections = 10 |}
    }
    
    let loadCache () = async {
        do! Async.Sleep 150
        return Map.ofList [ ("key1", "value1"); ("key2", "value2") ]
    }
    
    // รัน 3 งานพร้อมกัน
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    let data, config, cache =
        let d = Async.StartAsTask(fetchData ())
        let c = Async.StartAsTask(processConfig ())
        let l = Async.StartAsTask(loadCache ())
        Task.WhenAll([d :> Task; c :> Task; l :> Task]).Wait()
        d.Result, c.Result, l.Result
    
    sw.Stop()
    printfn "โหลด 3 ทรัพยากรใน %d ms (แทน ~450 ms)" sw.ElapsedMilliseconds
    printfn "Data: %A" data
    printfn "Config: %A" config
    printfn "Cache entries: %d" cache.Count
```

## 7. Thread Safety Issues

```fsharp
open System.Threading

// ปัญหา race condition
let raceConditionDemo () =
    let mutable counter = 0
    let iterations = 100_000
    
    // เพิ่ม counter จาก 2 threads พร้อมกัน
    let thread1 = Thread(fun () ->
        for _ in 1..iterations do
            counter <- counter + 1  // ไม่ thread-safe!
    )
    
    let thread2 = Thread(fun () ->
        for _ in 1..iterations do
            counter <- counter + 1  // ไม่ thread-safe!
    )
    
    thread1.Start()
    thread2.Start()
    thread1.Join()
    thread2.Join()
    
    printfn "คาดหวัง: %d, ได้จริง: %d" (iterations * 2) counter
    printfn "หาย: %d" (iterations * 2 - counter)
```

```fsharp
// สาเหตุของ race condition
(*
Thread 1: อ่าน counter = 5
Thread 2: อ่าน counter = 5  (ก่อน Thread 1 เขียนกลับ)
Thread 1: เขียน counter = 6
Thread 2: เขียน counter = 6  (ทับ Thread 1!)
ผลลัพธ์: counter = 6 แทนที่จะเป็น 7
*)
```

## 8. Race Conditions

```fsharp
// ตัวอย่าง race condition ใน collection
let collectionRaceCondition () =
    let list = System.Collections.Generic.List<int>()
    
    let tasks = 
        [|1..10|] 
        |> Array.map (fun i ->
            Task.Run(fun () ->
                for _ in 1..100 do
                    list.Add(i)  // ไม่ thread-safe!
            ))
    
    Task.WhenAll(tasks).Wait()
    
    printfn "คาดหวัง: %d, ได้จริง: %d" 1000 list.Count
    // อาจน้อยกว่า 1000 เพราะ race condition!
```

```fsharp
// วิธีแก้ race condition
let fixedCounter () =
    let mutable counter = 0
    let iterations = 100_000
    
    let thread1 = Thread(fun () ->
        for _ in 1..iterations do
            Interlocked.Increment(&counter) |> ignore  // Thread-safe!
    )
    
    let thread2 = Thread(fun () ->
        for _ in 1..iterations do
            Interlocked.Increment(&counter) |> ignore  // Thread-safe!
    )
    
    thread1.Start()
    thread2.Start()
    thread1.Join()
    thread2.Join()
    
    printfn "คาดหวัง: %d, ได้จริง: %d" (iterations * 2) counter

fixedCounter ()
```

## 9. Locks และ Monitor

```fsharp
// lock statement (syntactic sugar สำหรับ Monitor)
let lockExample () =
    let lockObj = obj()
    let mutable counter = 0
    
    let increment () =
        lock lockObj (fun () ->
            counter <- counter + 1
        )
    
    let tasks = [|1..100|] |> Array.map (fun _ ->
        Task.Run(fun () ->
            for _ in 1..100 do
                increment ()
        ))
    
    Task.WhenAll(tasks).Wait()
    printfn "Counter: %d (คาดหวัง: 10000)" counter
```

```fsharp
// Monitor สำหรับ granular control
let monitorExample () =
    let lockObj = obj()
    let mutable resource = "เริ่มต้น"
    
    let updateResource (newValue: string) =
        Monitor.Enter(lockObj)
        try
            printfn "เริ่ม update (thread %d)" Thread.CurrentThread.ManagedThreadId
            System.Threading.Thread.Sleep(100)
            resource <- newValue
            printfn "Update เสร็จ: %s" newValue
        finally
            Monitor.Exit(lockObj)
    
    // รันพร้อมกัน แต่จะทำทีละคน
    let tasks = 
        [| "ค่าที่ 1"; "ค่าที่ 2"; "ค่าที่ 3" |]
        |> Array.map (fun v -> Task.Run(fun () -> updateResource v))
    
    Task.WhenAll(tasks).Wait()
    printfn "ค่าสุดท้าย: %s" resource
```

```fsharp
// ReaderWriterLockSlim - ให้ readers หลายคนอ่านพร้อมกัน
let readerWriterExample () =
    let rwLock = new ReaderWriterLockSlim()
    let mutable data = Map.empty<string, int>
    
    let read (key: string) =
        rwLock.EnterReadLock()
        try
            data |> Map.tryFind key
        finally
            rwLock.ExitReadLock()
    
    let write (key: string) (value: int) =
        rwLock.EnterWriteLock()
        try
            data <- data |> Map.add key value
        finally
            rwLock.ExitWriteLock()
    
    // หลาย readers พร้อมกัน
    let readers = [|1..5|] |> Array.map (fun _ ->
        Task.Run(fun () ->
            for _ in 1..100 do
                read "key1" |> ignore
        ))
    
    // Writer ทีละคน
    let writer = Task.Run(fun () ->
        for i in 1..100 do
            write "key1" i
    )
    
    Task.WhenAll(Array.append readers [|writer|]).Wait()
    printfn "ค่าสุดท้าย: %A" (read "key1")
```

## 10. Mutex

```fsharp
// Mutex สำหรับ synchronization ข้าม process
let mutexExample () =
    use mutex = new Mutex(false, "MyAppMutex")
    
    // ลองได้รับ mutex
    if mutex.WaitOne(1000) then  // รอสูงสุด 1 วินาที
        try
            printfn "ได้รับ mutex - ทำงาน exclusive"
            Thread.Sleep(200)
            printfn "งาน exclusive เสร็จ"
        finally
            mutex.ReleaseMutex()
    else
        printfn "ไม่ได้รับ mutex - process อื่นกำลังใช้งาน"

mutexExample ()
```

## 11. Semaphore

```fsharp
// SemaphoreSlim - จำกัดจำนวน concurrent operations
let semaphoreExample () =
    use semaphore = new SemaphoreSlim(3)  // สูงสุด 3 concurrent
    
    let doWork (id: int) = async {
        do! semaphore.WaitAsync() |> Async.AwaitTask
        try
            printfn "เริ่มงาน %d (current: %d)" id (3 - semaphore.CurrentCount)
            do! Async.Sleep 200
            printfn "งาน %d เสร็จ" id
        finally
            semaphore.Release() |> ignore
    }
    
    let tasks = [1..10] |> List.map doWork
    Async.RunSynchronously (Async.Parallel tasks) |> ignore

semaphoreExample ()
```

```fsharp
// Semaphore สำหรับ connection pool
let connectionPoolExample () =
    let maxConnections = 5
    use semaphore = new SemaphoreSlim(maxConnections)
    
    let executeQuery (queryId: int) = async {
        printfn "Query %d รอ connection..." queryId
        do! semaphore.WaitAsync() |> Async.AwaitTask
        printfn "Query %d ได้ connection" queryId
        
        try
            // จำลองการ execute query
            do! Async.Sleep (100 + queryId * 10)
            printfn "Query %d เสร็จ" queryId
            return sprintf "ผลลัพธ์ query %d" queryId
        finally
            semaphore.Release() |> ignore
            printfn "Query %d คืน connection" queryId
    }
    
    // 15 queries แต่มีแค่ 5 connections
    let queries = [1..15] |> List.map executeQuery
    let results = Async.RunSynchronously (Async.Parallel queries)
    printfn "เสร็จทั้งหมด %d queries" results.Length
```

## 12. ThreadLocal<T>

```fsharp
// ThreadLocal<T> - ข้อมูลที่แยกต่อ thread
let threadLocalExample () =
    // แต่ละ thread มีค่าของตัวเอง
    let threadLocal = new ThreadLocal<int>(fun () -> Thread.CurrentThread.ManagedThreadId)
    
    let printThreadValue () =
        printfn "Thread %d มีค่า: %d" 
            Thread.CurrentThread.ManagedThreadId 
            threadLocal.Value
    
    let tasks = [|1..5|] |> Array.map (fun _ ->
        Task.Run(fun () ->
            // แต่ละ Task อาจรันบน thread ต่างกัน
            printThreadValue ()
            Thread.Sleep(100)
            printThreadValue ()
        ))
    
    Task.WhenAll(tasks).Wait()
    threadLocal.Dispose()
```

```fsharp
// ThreadLocal สำหรับ Random (thread-safe)
let threadSafeRandom () =
    let threadLocalRandom = new ThreadLocal<System.Random>(
        fun () -> System.Random(Thread.CurrentThread.ManagedThreadId)
    )
    
    let generateNumbers (count: int) =
        [|for _ in 1..count -> threadLocalRandom.Value.Next(1, 100)|]
    
    let results = 
        [|1..5|] 
        |> Array.Parallel.map (fun _ -> generateNumbers 10)
    
    printfn "สร้างตัวเลขสุ่ม: %d ชุด" results.Length
    threadLocalRandom.Dispose()
```

## 13. Concurrent Collections

```fsharp
open System.Collections.Concurrent

// ConcurrentDictionary - thread-safe dictionary
let concurrentDictionaryExample () =
    let dict = ConcurrentDictionary<string, int>()
    
    // เพิ่มข้อมูล
    dict["key1"] <- 1
    dict.TryAdd("key2", 2) |> ignore
    
    // AddOrUpdate - thread-safe
    let newValue = dict.AddOrUpdate("key1", 10, fun _ oldVal -> oldVal + 10)
    printfn "key1 = %d" newValue
    
    // GetOrAdd
    let val3 = dict.GetOrAdd("key3", 30)
    printfn "key3 = %d" val3
    
    // Thread-safe updates
    let tasks = [|1..100|] |> Array.map (fun _ ->
        Task.Run(fun () ->
            dict.AddOrUpdate("counter", 1, fun _ old -> old + 1) |> ignore
        ))
    
    Task.WhenAll(tasks).Wait()
    printfn "counter = %d (คาดหวัง: 100)" dict["counter"]
```

## 14. ConcurrentQueue

```fsharp
// ConcurrentQueue - thread-safe FIFO queue
let concurrentQueueExample () =
    let queue = ConcurrentQueue<int>()
    
    // Producers
    let producers = [|1..3|] |> Array.map (fun producerId ->
        Task.Run(fun () ->
            for i in 1..10 do
                let value = producerId * 100 + i
                queue.Enqueue(value)
                printfn "Producer %d: enqueue %d" producerId value
                Thread.Sleep(10)
        ))
    
    // Consumer
    let mutable consumed = 0
    let consumer = Task.Run(fun () ->
        let mutable running = true
        while running || not queue.IsEmpty do
            match queue.TryDequeue() with
            | true, value ->
                consumed <- consumed + 1
                printfn "Consumer: dequeue %d (รวม: %d)" value consumed
            | false, _ ->
                if not running then
                    ()
                else
                    Thread.Sleep(5)
        
        printfn "Consumer หยุดแล้ว"
    )
    
    Task.WhenAll(producers).Wait()
    printfn "Producers เสร็จทั้งหมด"
    Thread.Sleep(100)  // ให้ consumer ล้าง queue
    
    printfn "รวม consumed: %d" consumed
```

## 15. ConcurrentStack

```fsharp
// ConcurrentStack - thread-safe LIFO stack
let concurrentStackExample () =
    let stack = ConcurrentStack<string>()
    
    // Push items
    stack.Push("item1")
    stack.Push("item2")
    stack.Push("item3")
    
    // Push range
    stack.PushRange([|"a"; "b"; "c"|])
    
    // Pop items
    let mutable item = ""
    while stack.TryPop(&item) do
        printfn "Pop: %s" item
    
    printfn "Stack empty: %b" stack.IsEmpty
```

```fsharp
// Producer-Consumer ด้วย ConcurrentStack
let stackProducerConsumer () =
    let stack = ConcurrentStack<int>()
    use cts = new System.Threading.CancellationTokenSource()
    
    let producer = Task.Run(fun () ->
        for i in 1..20 do
            stack.Push(i)
            Thread.Sleep(50)
        printfn "Producer เสร็จ"
    )
    
    let consumer = Task.Run(fun () ->
        let mutable count = 0
        while count < 20 do
            let mutable value = 0
            if stack.TryPop(&value) then
                count <- count + 1
                printfn "Consumed: %d (%d/20)" value count
            else
                Thread.Sleep(10)
        printfn "Consumer เสร็จ"
    )
    
    Task.WhenAll([|producer; consumer|]).Wait()
```

## 16. BlockingCollection

```fsharp
// BlockingCollection สำหรับ bounded producer-consumer
let blockingCollectionExample () =
    // จำกัดขนาด buffer ที่ 5
    use collection = new BlockingCollection<int>(5)
    
    let producer = Task.Run(fun () ->
        for i in 1..15 do
            collection.Add(i)  // บล็อกถ้า collection เต็ม
            printfn "Produced: %d (Count: %d)" i collection.Count
        collection.CompleteAdding()  // บอกว่าจะไม่เพิ่มอีกแล้ว
        printfn "Producer เสร็จ"
    )
    
    let consumer = Task.Run(fun () ->
        // รับจนกว่า collection จะ complete และว่าง
        for item in collection.GetConsumingEnumerable() do
            Thread.Sleep(100)  // ช้ากว่า producer
            printfn "Consumed: %d" item
        printfn "Consumer เสร็จ"
    )
    
    Task.WhenAll([|producer; consumer|]).Wait()
```

## 17. Interlocked Operations

```fsharp
// Interlocked สำหรับ atomic operations
let interlockedExample () =
    let mutable counter = 0L
    let mutable flag = 0  // 0 = false, 1 = true
    
    // Increment/Decrement
    Interlocked.Increment(&counter) |> ignore
    Interlocked.Add(&counter, 10L) |> ignore
    printfn "Counter: %d" counter
    
    // CompareExchange - อัปเดตเฉพาะเมื่อค่าตรงกับที่คาดหวัง
    let success = Interlocked.CompareExchange(&flag, 1, 0) = 0
    printfn "Flag set: %b (flag = %d)" success flag
    
    // Exchange - ตั้งค่าและรับค่าเก่า
    let oldFlag = Interlocked.Exchange(&flag, 0)
    printfn "Old flag: %d, new flag: %d" oldFlag flag
```

```fsharp
// Thread-safe counter class
type AtomicCounter() =
    let mutable value = 0L
    
    member _.Increment() = Interlocked.Increment(&value)
    member _.Decrement() = Interlocked.Decrement(&value)
    member _.Add(n: int64) = Interlocked.Add(&value, n)
    member _.Value = Interlocked.Read(&value)
    member _.Reset() = Interlocked.Exchange(&value, 0L) |> ignore

// ทดสอบ
let counterTest () =
    let counter = AtomicCounter()
    
    let tasks = [|1..100|] |> Array.map (fun _ ->
        Task.Run(fun () ->
            for _ in 1..100 do
                counter.Increment() |> ignore
        ))
    
    Task.WhenAll(tasks).Wait()
    printfn "Counter: %d (คาดหวัง: 10000)" counter.Value

counterTest ()
```

## 18. Parallel Algorithms

```fsharp
// Parallel merge sort
let rec parallelMergeSort (threshold: int) (arr: int[]) =
    if arr.Length <= threshold then
        Array.sort arr
        arr
    else
        let mid = arr.Length / 2
        let left = arr[..mid-1]
        let right = arr[mid..]
        
        let sortedLeft, sortedRight =
            if arr.Length > 1000 then
                // รันแบบ parallel สำหรับ array ใหญ่
                let leftTask = Task.Run(fun () -> parallelMergeSort threshold left)
                let rightTask = Task.Run(fun () -> parallelMergeSort threshold right)
                leftTask.Result, rightTask.Result
            else
                // Sequential สำหรับ array เล็ก
                parallelMergeSort threshold left, parallelMergeSort threshold right
        
        // Merge
        let result = Array.zeroCreate arr.Length
        let mutable i, j, k = 0, 0, 0
        while i < sortedLeft.Length && j < sortedRight.Length do
            if sortedLeft[i] <= sortedRight[j] then
                result[k] <- sortedLeft[i]
                i <- i + 1
            else
                result[k] <- sortedRight[j]
                j <- j + 1
            k <- k + 1
        while i < sortedLeft.Length do
            result[k] <- sortedLeft[i]
            i <- i + 1
            k <- k + 1
        while j < sortedRight.Length do
            result[k] <- sortedRight[j]
            j <- j + 1
            k <- k + 1
        result

// ทดสอบ
let sortTest () =
    let rng = System.Random(42)
    let data = Array.init 100_000 (fun _ -> rng.Next(1_000_000))
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let sorted = parallelMergeSort 1000 data
    sw.Stop()
    
    printfn "Sorted %d items ใน %d ms" sorted.Length sw.ElapsedMilliseconds
    
    // ตรวจสอบความถูกต้อง
    let isOrdered = sorted |> Array.pairwise |> Array.forall (fun (a, b) -> a <= b)
    printfn "ผลถูกต้อง: %b" isOrdered

sortTest ()
```

## 19. Parallel Pattern: Fork-Join

```fsharp
// Fork-Join pattern
let forkJoin (tasks: (unit -> 'a) list) : 'a list =
    // Fork: เริ่มทุก task พร้อมกัน
    let runningTasks = tasks |> List.map (fun t -> Task.Run(t))
    
    // Join: รอทุก task เสร็จ
    Task.WhenAll(runningTasks).Wait()
    
    // รวบรวมผลลัพธ์
    runningTasks |> List.map (fun t -> t.Result)

// ตัวอย่าง
let forkJoinExample () =
    let workItems = [
        fun () ->
            Thread.Sleep(100)
            "ผลงาน 1"
        fun () ->
            Thread.Sleep(200)
            "ผลงาน 2"
        fun () ->
            Thread.Sleep(150)
            "ผลงาน 3"
    ]
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let results = forkJoin workItems
    sw.Stop()
    
    printfn "ผลลัพธ์: %A" results
    printfn "ใช้เวลา: %d ms (แทนที่จะเป็น ~450ms)" sw.ElapsedMilliseconds

forkJoinExample ()
```

## 20. Performance Guidelines

```fsharp
// แนวทางการเขียน Parallel code ที่ดี

// 1. ใช้ Parallel สำหรับ CPU-bound, Async สำหรับ I/O-bound
let guideline1 () =
    // CPU-bound: ใช้ Array.Parallel.map หรือ Parallel.For
    let cpuResult = 
        [|1..1000|] 
        |> Array.Parallel.map (fun x -> 
            // การคำนวณหนัก
            System.Math.Pow(float x, 2.0))
    
    // I/O-bound: ใช้ Async.Parallel
    let ioResult = Async.RunSynchronously (
        [1..10] 
        |> List.map (fun i -> async {
            do! Async.Sleep 100
            return i
        })
        |> Async.Parallel
    )
    printfn "CPU: %d, IO: %d" cpuResult.Length ioResult.Length

// 2. หลีกเลี่ยง shared mutable state
let guideline2 () =
    // ไม่ดี: mutable shared state
    let mutable badSum = 0
    // [|1..100|] |> Array.Parallel.iter (fun x -> badSum <- badSum + x)
    
    // ดี: ใช้ aggregation
    let goodSum = [|1..100|] |> Array.Parallel.map id |> Array.sum
    printfn "Good sum: %d" goodSum

// 3. ระวัง overhead ของ parallelism สำหรับงานเล็ก
let guideline3 () =
    let sw = System.Diagnostics.Stopwatch.StartNew()
    let seqResult = [|1..10|] |> Array.map (fun x -> x * x)
    sw.Stop()
    let seqTime = sw.ElapsedMilliseconds
    
    sw.Restart()
    let parResult = [|1..10|] |> Array.Parallel.map (fun x -> x * x)
    sw.Stop()
    let parTime = sw.ElapsedMilliseconds
    
    printfn "Sequential (tiny): %d ms" seqTime
    printfn "Parallel (tiny): %d ms" parTime
    printfn "สำหรับงานเล็ก Parallel อาจช้ากว่า Sequential!"

guideline1 ()
guideline2 ()
guideline3 ()
```

## สรุป

```fsharp
// สรุปคำสั่งสำคัญ
(*
Array.Parallel.map   - map แบบ parallel
Array.Parallel.iter  - iter แบบ parallel
AsParallel()         - PLINQ
Parallel.For         - for loop แบบ parallel
Parallel.ForEach     - foreach แบบ parallel
Task.Run             - รัน CPU-bound task
Task.WhenAll         - รอทุก task
lock                 - mutual exclusion
Interlocked          - atomic operations
SemaphoreSlim        - จำกัด concurrent
ReaderWriterLockSlim - read/write lock
ConcurrentDictionary - thread-safe dictionary
ConcurrentQueue      - thread-safe queue
BlockingCollection   - bounded buffer
ThreadLocal<T>       - per-thread data
*)

printfn "Parallel Programming - สรุปเสร็จ!"
```
