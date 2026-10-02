# Part 45 - Channels

## บทนำ

`System.Threading.Channels` เป็น namespace ใน .NET ที่ให้ channel สำหรับการสื่อสารระหว่าง producers และ consumers อย่างมีประสิทธิภาพ Channels เหมาะสำหรับ producer-consumer pattern และ data pipeline ที่ต้องการ backpressure

## 1. System.Threading.Channels Namespace

```fsharp
open System.Threading.Channels
open System.Threading.Tasks
open System

// Channel พื้นฐาน
let channelBasics () =
    // Channel คือ pipe ที่ข้อมูลไหลจาก writer ไปยัง reader
    // Channel.CreateUnbounded<T>() - ขนาดไม่จำกัด
    // Channel.CreateBounded<T>(capacity) - มีขนาดจำกัด
    
    let channel = Channel.CreateUnbounded<int>()
    
    // Writer ส่งข้อมูล
    channel.Writer.TryWrite(42) |> ignore
    
    // Reader รับข้อมูล
    let mutable value = 0
    if channel.Reader.TryRead(&value) then
        printfn "ได้รับ: %d" value

channelBasics ()
```

## 2. Channel.CreateBounded

```fsharp
// Bounded channel - มี capacity จำกัด
let boundedChannelExample () =
    let options = BoundedChannelOptions(10,
        FullMode = BoundedChannelFullMode.Wait)  // รอเมื่อเต็ม
    
    let channel = Channel.CreateBounded<string>(options)
    
    // เขียนข้อมูล
    let writeTask: Task = task {
        for i in 1..15 do
            // เมื่อ channel เต็ม (> 10) จะรอ
            do! channel.Writer.WriteAsync(sprintf "ข้อมูล %d" i)
            printfn "เขียน: ข้อมูล %d" i
        channel.Writer.Complete()
    }
    
    // อ่านข้อมูล (ช้ากว่า writer)
    let readTask: Task = task {
        let reader = channel.Reader
        while not reader.Completion.IsCompleted do
            match! reader.WaitToReadAsync() with
            | true ->
                let mutable item = ""
                while reader.TryRead(&item) do
                    do! Task.Delay(50)  // อ่านช้ากว่าเขียน
                    printfn "อ่าน: %s" item
            | false -> ()
    }
    
    Task.WhenAll([|writeTask; readTask|]).Wait()

boundedChannelExample ()
```

```fsharp
// BoundedChannelFullMode options
let boundedModes () =
    // Wait - รอจน channel ว่าง (backpressure)
    let waitChannel = Channel.CreateBounded<int>(BoundedChannelOptions(5,
        FullMode = BoundedChannelFullMode.Wait))
    
    // DropNewest - ทิ้ง item ที่เพิ่งเขียน
    let dropNewestChannel = Channel.CreateBounded<int>(BoundedChannelOptions(5,
        FullMode = BoundedChannelFullMode.DropNewest))
    
    // DropOldest - ทิ้ง item เก่าที่สุดใน queue
    let dropOldestChannel = Channel.CreateBounded<int>(BoundedChannelOptions(5,
        FullMode = BoundedChannelFullMode.DropOldest))
    
    // DropWrite - ทิ้ง item ที่กำลังเขียน
    let dropWriteChannel = Channel.CreateBounded<int>(BoundedChannelOptions(5,
        FullMode = BoundedChannelFullMode.DropWrite))
    
    // ทดสอบ DropOldest
    for i in 1..8 do
        dropOldestChannel.Writer.TryWrite(i) |> ignore
    
    // อ่านค่า (จะได้ค่าใหม่กว่า)
    let mutable v = 0
    let values = ResizeArray<int>()
    while dropOldestChannel.Reader.TryRead(&v) do
        values.Add(v)
    printfn "DropOldest (เขียน 1-8, capacity 5): %A" (values |> Seq.toList)
```

## 3. Channel.CreateUnbounded

```fsharp
// Unbounded channel - ไม่จำกัด capacity
let unboundedChannelExample () =
    let channel = Channel.CreateUnbounded<int>()
    
    // เขียนข้อมูลโดยไม่มีการบล็อก
    for i in 1..1000 do
        channel.Writer.TryWrite(i) |> ignore
    
    channel.Writer.Complete()
    
    // อ่านทั้งหมด
    let mutable count = 0
    let mutable v = 0
    while channel.Reader.TryRead(&v) do
        count <- count + 1
    
    printfn "อ่านได้ %d items" count
```

```fsharp
// UnboundedChannelOptions
let unboundedWithOptions () =
    let options = UnboundedChannelOptions(
        SingleReader = true,   // มี reader คนเดียว (optimize)
        SingleWriter = true    // มี writer คนเดียว (optimize)
    )
    
    let channel = Channel.CreateUnbounded<string>(options)
    
    channel.Writer.TryWrite("ข้อมูล 1") |> ignore
    channel.Writer.TryWrite("ข้อมูล 2") |> ignore
    
    let mutable item = ""
    while channel.Reader.TryRead(&item) do
        printfn "อ่าน: %s" item
```

## 4. Writing to Channels

```fsharp
// วิธีการเขียนต่างๆ
let writingMethods () = task {
    let channel = Channel.CreateBounded<int>(100)
    let writer = channel.Writer
    
    // TryWrite - ส่งคืน bool ว่าสำเร็จไหม (synchronous)
    let success1 = writer.TryWrite(1)
    printfn "TryWrite สำเร็จ: %b" success1
    
    // WriteAsync - รอถ้า channel เต็ม (asynchronous)
    do! writer.WriteAsync(2)
    
    // WaitToWriteAsync - รอจนกว่าจะเขียนได้
    let! canWrite = writer.WaitToWriteAsync()
    if canWrite then
        writer.TryWrite(3) |> ignore
    
    // Complete - บอกว่าจะไม่เขียนอีกแล้ว
    writer.Complete()
    
    // TryComplete - ปิดด้วย exception (ถ้ามี error)
    // writer.TryComplete(Some (Exception("เกิดข้อผิดพลาด")))
    
    printfn "เขียน 3 items เสร็จแล้ว"
}

writingMethods().Wait()
```

## 5. Reading from Channels

```fsharp
// วิธีการอ่านต่างๆ
let readingMethods () = task {
    let channel = Channel.CreateUnbounded<int>()
    let reader = channel.Reader
    
    // เตรียมข้อมูล
    for i in 1..5 do
        channel.Writer.TryWrite(i) |> ignore
    channel.Writer.Complete()
    
    // TryRead - อ่านทันที ถ้าไม่มีคืน false (synchronous)
    let mutable v = 0
    if reader.TryRead(&v) then
        printfn "TryRead: %d" v
    
    // ReadAsync - รอถ้าไม่มีข้อมูล (asynchronous)
    let! item = reader.ReadAsync()
    printfn "ReadAsync: %d" item
    
    // WaitToReadAsync - รอจนกว่าจะมีข้อมูลหรือ complete
    while! reader.WaitToReadAsync() do
        while reader.TryRead(&v) do
            printfn "WaitToRead: %d" v
    
    printfn "อ่านข้อมูลทั้งหมดแล้ว"
}

readingMethods().Wait()
```

## 6. Async Channel Operations

```fsharp
// Channel operations แบบ async
let asyncChannelOps () = async {
    let channel = Channel.CreateBounded<string>(5)
    
    // Writer async workflow
    let writer = async {
        for i in 1..10 do
            let! canWrite = channel.Writer.WaitToWriteAsync() |> Async.AwaitTask
            if canWrite then
                channel.Writer.TryWrite(sprintf "item-%d" i) |> ignore
                printfn "เขียน item-%d" i
        channel.Writer.Complete()
    }
    
    // Reader async workflow
    let reader = async {
        let mutable continueReading = true
        while continueReading do
            let! hasData = channel.Reader.WaitToReadAsync() |> Async.AwaitTask
            if hasData then
                let mutable item = ""
                while channel.Reader.TryRead(&item) do
                    do! Async.Sleep 50
                    printfn "อ่าน: %s" item
            else
                continueReading <- false
    }
    
    // รันพร้อมกัน
    do! [writer; reader] |> Async.Parallel |> Async.Ignore
}

Async.RunSynchronously (asyncChannelOps ())
```

## 7. Pipeline with Channels

```fsharp
// Data pipeline ด้วย channels
let channelPipeline () = task {
    // Stage 1: Generate numbers
    let stage1 = Channel.CreateBounded<int>(10)
    // Stage 2: Filter evens
    let stage2 = Channel.CreateBounded<int>(10)
    // Stage 3: Square
    let stage3 = Channel.CreateBounded<int>(10)
    
    // Producer
    let produce = task {
        for i in 1..20 do
            do! stage1.Writer.WriteAsync(i)
            printfn "Generate: %d" i
        stage1.Writer.Complete()
    }
    
    // Filter stage
    let filter = task {
        let reader = stage1.Reader
        while! reader.WaitToReadAsync() do
            let mutable v = 0
            while reader.TryRead(&v) do
                if v % 2 = 0 then
                    do! stage2.Writer.WriteAsync(v)
                    printfn "Filter: %d (เลขคู่)" v
        stage2.Writer.Complete()
    }
    
    // Transform stage
    let transform = task {
        let reader = stage2.Reader
        while! reader.WaitToReadAsync() do
            let mutable v = 0
            while reader.TryRead(&v) do
                let squared = v * v
                do! stage3.Writer.WriteAsync(squared)
                printfn "Transform: %d^2 = %d" v squared
        stage3.Writer.Complete()
    }
    
    // Consumer
    let consume = task {
        let results = ResizeArray<int>()
        let reader = stage3.Reader
        while! reader.WaitToReadAsync() do
            let mutable v = 0
            while reader.TryRead(&v) do
                results.Add(v)
                printfn "Consume: %d" v
        printfn "\nผลลัพธ์สุดท้าย: %A" (results |> Seq.toList)
    }
    
    // รันทุก stages พร้อมกัน
    do! Task.WhenAll([|produce; filter; transform; consume|])
}

channelPipeline().Wait()
```

## 8. Fan-Out Pattern

```fsharp
// Fan-out: กระจายงานจาก 1 source ไปหลาย workers
let fanOutPattern () = task {
    let source = Channel.CreateUnbounded<int>()
    let workerCount = 4
    
    // สร้าง worker channels
    let workerChannels = 
        Array.init workerCount (fun _ -> Channel.CreateUnbounded<int>())
    
    // Dispatcher - กระจายงานแบบ round-robin
    let dispatch = task {
        let mutable workerIdx = 0
        while! source.Reader.WaitToReadAsync() do
            let mutable item = 0
            while source.Reader.TryRead(&item) do
                do! workerChannels[workerIdx].Writer.WriteAsync(item)
                workerIdx <- (workerIdx + 1) % workerCount
        
        // ปิดทุก worker channels
        for wc in workerChannels do
            wc.Writer.Complete()
    }
    
    // Worker tasks
    let workers = 
        workerChannels 
        |> Array.mapi (fun i ch -> task {
            while! ch.Reader.WaitToReadAsync() do
                let mutable item = 0
                while ch.Reader.TryRead(&item) do
                    do! Task.Delay(50)  // จำลองการทำงาน
                    printfn "Worker %d ประมวลผล: %d" i item
        })
    
    // ส่งข้อมูล
    let produce = task {
        for i in 1..20 do
            do! source.Writer.WriteAsync(i)
        source.Writer.Complete()
    }
    
    // รันทั้งหมด
    do! Task.WhenAll(Array.append [|produce; dispatch|] workers)
    printfn "Fan-out เสร็จสิ้น"
}

fanOutPattern().Wait()
```

## 9. Fan-In Pattern

```fsharp
// Fan-in: รวมผลลัพธ์จากหลาย sources มาที่ 1 output
let fanInPattern () = task {
    let output = Channel.CreateUnbounded<string>()
    
    // สร้าง multiple sources
    let createSource (id: int) : Task = task {
        for i in 1..5 do
            do! Task.Delay(id * 20 + i * 10)
            let message = sprintf "Source %d: ข้อมูล %d" id i
            do! output.Writer.WriteAsync(message)
        printfn "Source %d เสร็จ" id
    }
    
    let sources = [|1..4|] |> Array.map createSource
    
    // รอทุก sources แล้วปิด output
    let closeWhenDone = task {
        do! Task.WhenAll(sources)
        output.Writer.Complete()
        printfn "ทุก sources เสร็จ - ปิด output"
    }
    
    // Consumer
    let consume = task {
        let results = ResizeArray<string>()
        while! output.Reader.WaitToReadAsync() do
            let mutable item = ""
            while output.Reader.TryRead(&item) do
                results.Add(item)
                printfn "Received: %s" item
        printfn "\nรวม %d items จาก fan-in" results.Count
    }
    
    do! Task.WhenAll([|closeWhenDone; consume|])
}

fanInPattern().Wait()
```

## 10. Backpressure

```fsharp
// Backpressure: ชะลอ producer เมื่อ consumer ช้า
let backpressureExample () = task {
    // Channel ขนาดเล็กทำให้มี backpressure
    let channel = Channel.CreateBounded<int>(BoundedChannelOptions(5,
        FullMode = BoundedChannelFullMode.Wait))
    
    let sw = System.Diagnostics.Stopwatch()
    
    // Fast producer
    let produce = task {
        for i in 1..20 do
            let start = sw.ElapsedMilliseconds
            do! channel.Writer.WriteAsync(i)
            let writeTime = sw.ElapsedMilliseconds - start
            printfn "Producer: เขียน %d (%d ms รอ)" i writeTime
        channel.Writer.Complete()
    }
    
    // Slow consumer
    let consume = task {
        while! channel.Reader.WaitToReadAsync() do
            let mutable v = 0
            while channel.Reader.TryRead(&v) do
                do! Task.Delay(100)  // ช้ากว่า producer
                printfn "Consumer: อ่าน %d" v
    }
    
    sw.Start()
    do! Task.WhenAll([|produce; consume|])
    printfn "\nBackpressure demo เสร็จสิ้น"
}

backpressureExample().Wait()
```

## 11. Channel Completion

```fsharp
// การจัดการ channel completion
let channelCompletionExample () = task {
    let channel = Channel.CreateUnbounded<int>()
    
    // เขียนข้อมูล
    for i in 1..5 do
        channel.Writer.TryWrite(i) |> ignore
    
    // Complete channel (normal)
    channel.Writer.Complete()
    
    // ตรวจสอบ completion
    printfn "Writer completed: %b" channel.Writer.TryComplete()
    
    // รอ completion ของ reader
    try
        do! channel.Reader.Completion
        printfn "Reader completed (normal)"
    with
    | ex ->
        printfn "Reader completed with error: %s" ex.Message
}

channelCompletionExample().Wait()
```

```fsharp
// Complete ด้วย error
let completionWithError () = task {
    let channel = Channel.CreateUnbounded<int>()
    
    // ส่งข้อมูล
    channel.Writer.TryWrite(1) |> ignore
    channel.Writer.TryWrite(2) |> ignore
    
    // Complete ด้วย error
    channel.Writer.TryComplete(Exception("เกิดข้อผิดพลาด!")) |> ignore
    
    try
        // อ่านข้อมูล
        while! channel.Reader.WaitToReadAsync() do
            let mutable v = 0
            while channel.Reader.TryRead(&v) do
                printfn "อ่าน: %d" v
        
        // รอ completion - จะ throw exception
        do! channel.Reader.Completion
    with
    | ex ->
        printfn "ข้อผิดพลาด: %s" ex.Message
}

completionWithError().Wait()
```

## 12. Error Handling

```fsharp
// Error handling ใน channel operations
let errorHandling () = task {
    let channel = Channel.CreateBounded<int>(10)
    
    // Writer ที่อาจล้มเหลว
    let write = task {
        try
            for i in 1..5 do
                do! channel.Writer.WriteAsync(i)
            channel.Writer.Complete()
        with
        | ex ->
            printfn "Writer error: %s" ex.Message
            channel.Writer.TryComplete(ex) |> ignore
    }
    
    // Reader ที่จัดการ errors
    let read = task {
        try
            while! channel.Reader.WaitToReadAsync() do
                let mutable v = 0
                while channel.Reader.TryRead(&v) do
                    printfn "อ่าน: %d" v
            
            // รอ completion
            do! channel.Reader.Completion
            printfn "อ่านเสร็จ (ปกติ)"
        with
        | :? ChannelClosedException ->
            printfn "Channel ถูกปิดแบบ normal"
        | ex ->
            printfn "Reader error: %s" ex.Message
    }
    
    do! Task.WhenAll([|write; read|])
}

errorHandling().Wait()
```

## 13. Producer-Consumer Pattern

```fsharp
// Classic Producer-Consumer ด้วย Channel
module ProducerConsumer =
    
    type WorkItem = {
        Id: int
        Data: string
        Priority: int
    }
    
    // Producer
    let produce (channel: ChannelWriter<WorkItem>) (count: int) = async {
        for i in 1..count do
            let item = {
                Id = i
                Data = sprintf "งาน-%d" i
                Priority = i % 3 + 1
            }
            do! channel.WriteAsync(item) |> Async.AwaitTask
            printfn "Producer: ส่งงาน %d (priority: %d)" i item.Priority
        
        channel.Complete()
        printfn "Producer: เสร็จสิ้น"
    }
    
    // Consumer
    let consume (id: int) (channel: ChannelReader<WorkItem>) = async {
        while! (channel.WaitToReadAsync() |> Async.AwaitTask) do
            let mutable item = Unchecked.defaultof<WorkItem>
            while channel.TryRead(&item) do
                do! Async.Sleep (item.Priority * 30)
                printfn "Consumer %d: ประมวลผลงาน %d (%s)" id item.Id item.Data
        
        printfn "Consumer %d: หยุดแล้ว" id
    }

// ทดสอบ
let producerConsumerTest () = async {
    let channel = Channel.CreateBounded<ProducerConsumer.WorkItem>(10)
    
    // รัน 1 producer และ 3 consumers พร้อมกัน
    let producer = ProducerConsumer.produce channel.Writer 15
    let consumers = 
        [1..3] 
        |> List.map (fun i -> ProducerConsumer.consume i channel.Reader)
    
    do! [producer] @ consumers |> Async.Parallel |> Async.Ignore
    printfn "ทดสอบเสร็จ"
}

Async.RunSynchronously (producerConsumerTest ())
```

## 14. Channel กับ IAsyncEnumerable

```fsharp
// อ่าน channel ด้วย IAsyncEnumerable (ReadAllAsync)
let asyncEnumerableExample () = task {
    let channel = Channel.CreateUnbounded<int>()
    
    // Producer
    let produce = task {
        for i in 1..10 do
            channel.Writer.TryWrite(i) |> ignore
            do! Task.Delay(50)
        channel.Writer.Complete()
    }
    
    // Consumer ด้วย ReadAllAsync (elegant!)
    let consume = task {
        let results = ResizeArray<int>()
        
        // ReadAllAsync คืน IAsyncEnumerable<T>
        for item in channel.Reader.ReadAllAsync() do
            results.Add(item)
            printfn "อ่าน: %d" item
        
        printfn "รวม: %d items" results.Count
    }
    
    do! Task.WhenAll([|produce; consume|])
}

asyncEnumerableExample().Wait()
```

## 15. ตัวอย่างจริง: Message Processing System

```fsharp
// ระบบประมวลผล messages แบบ real-world
module MessageSystem =
    
    type Priority = High | Normal | Low
    
    type Message = {
        Id: Guid
        Content: string
        Priority: Priority
        Timestamp: DateTime
    }
    
    type ProcessResult = {
        MessageId: Guid
        Success: bool
        ProcessingTime: int64
    }
    
    // Priority queue ด้วย 3 channels
    let createPriorityQueue () =
        let highChannel = Channel.CreateUnbounded<Message>()
        let normalChannel = Channel.CreateBounded<Message>(100)
        let lowChannel = Channel.CreateBounded<Message>(50)
        
        highChannel, normalChannel, lowChannel
    
    // Router ที่ส่ง message ไปยัง channel ที่ถูกต้อง
    let route (high: ChannelWriter<Message>) 
              (normal: ChannelWriter<Message>) 
              (low: ChannelWriter<Message>) 
              (msg: Message) = async {
        match msg.Priority with
        | High -> do! high.WriteAsync(msg) |> Async.AwaitTask
        | Normal -> do! normal.WriteAsync(msg) |> Async.AwaitTask
        | Low -> do! low.WriteAsync(msg) |> Async.AwaitTask
    }
    
    // Worker ที่ประมวลผล messages
    let worker (id: int) (reader: ChannelReader<Message>) 
               (results: ChannelWriter<ProcessResult>) = async {
        while! (reader.WaitToReadAsync() |> Async.AwaitTask) do
            let mutable msg = Unchecked.defaultof<Message>
            while reader.TryRead(&msg) do
                let sw = System.Diagnostics.Stopwatch.StartNew()
                
                // ประมวลผล
                do! Async.Sleep 100
                printfn "Worker %d: ประมวลผล [%s] %s" id (msg.Priority |> string) msg.Content
                
                sw.Stop()
                let result = {
                    MessageId = msg.Id
                    Success = true
                    ProcessingTime = sw.ElapsedMilliseconds
                }
                do! results.WriteAsync(result) |> Async.AwaitTask
    }

// ทดสอบ
let messageSystemTest () = async {
    let high, normal, low = MessageSystem.createPriorityQueue ()
    let resultsChannel = Channel.CreateUnbounded<MessageSystem.ProcessResult>()
    
    // สร้าง messages
    let messages = [
        { Id = Guid.NewGuid(); Content = "งานด่วน!"; Priority = MessageSystem.High; Timestamp = DateTime.UtcNow }
        { Id = Guid.NewGuid(); Content = "งานปกติ 1"; Priority = MessageSystem.Normal; Timestamp = DateTime.UtcNow }
        { Id = Guid.NewGuid(); Content = "งานต่ำ 1"; Priority = MessageSystem.Low; Timestamp = DateTime.UtcNow }
        { Id = Guid.NewGuid(); Content = "งานด่วนอีก!"; Priority = MessageSystem.High; Timestamp = DateTime.UtcNow }
        { Id = Guid.NewGuid(); Content = "งานปกติ 2"; Priority = MessageSystem.Normal; Timestamp = DateTime.UtcNow }
    ]
    
    // ส่ง messages
    for msg in messages do
        do! MessageSystem.route high.Writer normal.Writer low.Writer msg
    
    high.Writer.Complete()
    normal.Writer.Complete()
    low.Writer.Complete()
    
    // รัน workers
    let highWorker = MessageSystem.worker 1 high.Reader resultsChannel.Writer
    let normalWorker = MessageSystem.worker 2 normal.Reader resultsChannel.Writer
    let lowWorker = MessageSystem.worker 3 low.Reader resultsChannel.Writer
    
    do! [highWorker; normalWorker; lowWorker] |> Async.Parallel |> Async.Ignore
    
    resultsChannel.Writer.Complete()
    
    // อ่านผลลัพธ์
    let mutable totalTime = 0L
    let mutable count = 0
    let mutable result = Unchecked.defaultof<MessageSystem.ProcessResult>
    while resultsChannel.Reader.TryRead(&result) do
        totalTime <- totalTime + result.ProcessingTime
        count <- count + 1
    
    printfn "\nสรุป: ประมวลผล %d messages, เฉลี่ย %d ms" count (totalTime / int64 count)
}

Async.RunSynchronously (messageSystemTest ())
```

## 16. Channel Metrics และ Monitoring

```fsharp
// Monitoring channel usage
let channelMonitoring () = task {
    let channel = Channel.CreateBounded<int>(20)
    
    let mutable written = 0
    let mutable read = 0
    
    // Monitor ใน background
    let monitor = task {
        while not channel.Reader.Completion.IsCompleted do
            do! Task.Delay(100)
            printfn "สถานะ: เขียน=%d, อ่าน=%d, ใน queue=%d" 
                written read (written - read)
    }
    
    // Producer
    let produce = task {
        for i in 1..50 do
            do! channel.Writer.WriteAsync(i)
            written <- written + 1
        channel.Writer.Complete()
    }
    
    // Consumer
    let consume = task {
        while! channel.Reader.WaitToReadAsync() do
            let mutable v = 0
            while channel.Reader.TryRead(&v) do
                do! Task.Delay(30)
                read <- read + 1
    }
    
    do! Task.WhenAll([|produce; consume|])
    printfn "\nสรุป: เขียน=%d, อ่าน=%d" written read
}

channelMonitoring().Wait()
```

## สรุป

```fsharp
(*
Channels - สรุปจุดสำคัญ:

สร้าง Channel:
- Channel.CreateUnbounded<T>()    - ไม่จำกัดขนาด
- Channel.CreateBounded<T>(n)     - จำกัดขนาด n

เขียนข้อมูล:
- writer.TryWrite(item)           - synchronous, คืน bool
- writer.WriteAsync(item)         - async, รอถ้าเต็ม
- writer.WaitToWriteAsync()       - รอจนกว่าจะเขียนได้
- writer.Complete()               - ปิด channel

อ่านข้อมูล:
- reader.TryRead(&item)           - synchronous, คืน bool
- reader.ReadAsync()              - async, รอถ้าว่าง
- reader.WaitToReadAsync()        - รอจนกว่าจะมีข้อมูล
- reader.ReadAllAsync()           - IAsyncEnumerable
- reader.Completion               - Task ที่ complete เมื่อ channel ปิด

BoundedChannelFullMode:
- Wait        - รอ (backpressure)
- DropNewest  - ทิ้งล่าสุด
- DropOldest  - ทิ้งเก่าสุด
- DropWrite   - ทิ้งที่กำลังเขียน

Patterns:
- Pipeline    - ส่งต่อข้อมูลผ่าน stages
- Fan-out     - กระจายไปหลาย consumers
- Fan-in      - รวมจากหลาย producers
- Backpressure - ชะลอ producer อัตโนมัติ
*)

printfn "Channels - สรุปเสร็จ!"
```
