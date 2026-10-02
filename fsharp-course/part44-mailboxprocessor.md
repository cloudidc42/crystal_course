# Part 44 - MailboxProcessor (Actor Model)

## บทนำ

`MailboxProcessor<T>` ของ F# เป็นการนำ Actor Model มาใช้ใน .NET Actor Model เป็น pattern การเขียนโปรแกรม concurrent โดยใช้ "actors" ที่สื่อสารกันผ่านการส่ง message แทนการแชร์ memory โดยตรง ช่วยหลีกเลี่ยงปัญหา race condition และ deadlock ได้

## 1. Actor Model Concept

```fsharp
(*
Actor Model หลักการ:
- แต่ละ Actor มี mailbox (inbox) ของตัวเอง
- Actors สื่อสารกันผ่านการส่ง messages
- Actor ประมวลผล message ทีละอัน (sequential)
- ไม่มี shared mutable state ระหว่าง actors
- Actors สามารถสร้าง actor ลูกได้

ประโยชน์:
- ไม่ต้องใช้ locks
- ไม่มี race conditions
- Scale ได้ดี
- ง่ายต่อการ reason about
*)
```

## 2. MailboxProcessor พื้นฐาน

```fsharp
// สร้าง MailboxProcessor อย่างง่าย
let basicMailbox () =
    // สร้าง agent ที่รับ string messages
    let agent = MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()  // รอ message
            printfn "ได้รับ: %s" msg
            return! loop ()  // วนซ้ำ
        }
        loop ()
    )
    
    // ส่ง messages
    agent.Post("สวัสดี")
    agent.Post("F# Actor")
    agent.Post("Model")
    
    System.Threading.Thread.Sleep(100)

basicMailbox ()
```

```fsharp
// MailboxProcessor กับ discriminated union messages
type CounterMessage =
    | Increment
    | Decrement
    | Reset
    | GetValue of replyChannel: AsyncReplyChannel<int>

let counterAgent () =
    let agent = MailboxProcessor<CounterMessage>.Start(fun inbox ->
        let rec loop (count: int) = async {
            let! msg = inbox.Receive()
            match msg with
            | Increment -> return! loop (count + 1)
            | Decrement -> return! loop (count - 1)
            | Reset -> return! loop 0
            | GetValue replyChannel ->
                replyChannel.Reply count
                return! loop count
        }
        loop 0
    )
    agent

let counter = counterAgent ()

// ส่ง messages
counter.Post Increment
counter.Post Increment
counter.Post Increment
counter.Post Decrement

// ขอค่าปัจจุบัน (synchronous call)
let value = counter.PostAndReply (fun rc -> GetValue rc)
printfn "ค่าปัจจุบัน: %d" value  // 2
```

## 3. Posting Messages

```fsharp
// วิธีการส่ง messages ต่างๆ
let messageSendingExample () =
    let agent = MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            printfn "[Agent] %s" msg
            return! loop ()
        }
        loop ()
    )
    
    // Post - fire and forget (ไม่รอ)
    agent.Post "ข้อความ 1"
    
    // PostAndReply - รอการตอบกลับ (synchronous)
    // (ต้องการ reply channel - ดูตัวอย่างด้านล่าง)
    
    // TryPost - ส่งโดยไม่รอถ้า mailbox เต็ม
    let success = agent.TryPost "ข้อความ 2"
    printfn "TryPost สำเร็จ: %b" success
    
    System.Threading.Thread.Sleep(100)

messageSendingExample ()
```

```fsharp
// PostAndReply vs PostAndAsyncReply
type CalcMessage =
    | Add of int * int * AsyncReplyChannel<int>
    | Multiply of int * int * AsyncReplyChannel<int>

let calcAgent = MailboxProcessor<CalcMessage>.Start(fun inbox ->
    let rec loop () = async {
        let! msg = inbox.Receive()
        match msg with
        | Add (a, b, rc) ->
            rc.Reply (a + b)
        | Multiply (a, b, rc) ->
            rc.Reply (a * b)
        return! loop ()
    }
    loop ()
)

// PostAndReply - รอผลลัพธ์ (synchronous, บล็อก thread)
let sum = calcAgent.PostAndReply (fun rc -> Add(10, 20, rc))
printfn "10 + 20 = %d" sum

// PostAndAsyncReply - รอแบบ async
let product = 
    async {
        return! calcAgent.PostAndAsyncReply (fun rc -> Multiply(5, 6, rc))
    } |> Async.RunSynchronously

printfn "5 * 6 = %d" product
```

## 4. Receive Loop

```fsharp
// Receive loop patterns
let receiveLoopPatterns () =
    // Pattern 1: Simple loop
    let simpleAgent = MailboxProcessor<int>.Start(fun inbox ->
        let rec loop () = async {
            let! n = inbox.Receive()
            printfn "ได้รับ: %d" n
            return! loop ()
        }
        loop ()
    )
    
    // Pattern 2: Loop with state
    let statefulAgent = MailboxProcessor<int>.Start(fun inbox ->
        let rec loop (sum: int) (count: int) = async {
            let! n = inbox.Receive()
            let newSum = sum + n
            let newCount = count + 1
            printfn "ผลรวม: %d, จำนวน: %d, เฉลี่ย: %.2f" 
                newSum newCount (float newSum / float newCount)
            return! loop newSum newCount
        }
        loop 0 0
    )
    
    // Pattern 3: Loop with timeout
    let timeoutAgent = MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.TryReceive(timeout = 500)  // รอ 500ms
            match msg with
            | Some m ->
                printfn "ได้รับ: %s" m
                return! loop ()
            | None ->
                printfn "หมดเวลา! ไม่มี message"
                return! loop ()
        }
        loop ()
    )
    
    // ส่ง messages
    for i in 1..5 do
        simpleAgent.Post i
        statefulAgent.Post (i * 10)
    
    System.Threading.Thread.Sleep(200)
    timeoutAgent.Post "test"
    System.Threading.Thread.Sleep(700)  // รอ timeout
```

## 5. Reply Channels

```fsharp
// Reply Channels สำหรับ request-response pattern
type DatabaseMessage =
    | Insert of key:string * value:string
    | Get of key:string * AsyncReplyChannel<string option>
    | Delete of key:string
    | GetAll of AsyncReplyChannel<Map<string, string>>
    | Clear

let databaseAgent () =
    MailboxProcessor<DatabaseMessage>.Start(fun inbox ->
        let rec loop (db: Map<string, string>) = async {
            let! msg = inbox.Receive()
            match msg with
            | Insert (key, value) ->
                return! loop (db |> Map.add key value)
            
            | Get (key, rc) ->
                rc.Reply (db |> Map.tryFind key)
                return! loop db
            
            | Delete key ->
                return! loop (db |> Map.remove key)
            
            | GetAll rc ->
                rc.Reply db
                return! loop db
            
            | Clear ->
                return! loop Map.empty
        }
        loop Map.empty
    )

// ใช้งาน
let db = databaseAgent ()

db.Post (Insert ("user:1", "สมชาย"))
db.Post (Insert ("user:2", "สุดา"))
db.Post (Insert ("user:3", "มนัส"))

let user1 = db.PostAndReply (fun rc -> Get("user:1", rc))
printfn "user:1 = %A" user1

let allUsers = db.PostAndReply (fun rc -> GetAll rc)
printfn "ผู้ใช้ทั้งหมด: %A" allUsers

db.Post (Delete "user:2")
let afterDelete = db.PostAndReply (fun rc -> GetAll rc)
printfn "หลังลบ: %A" afterDelete
```

## 6. Error Handling ใน Actors

```fsharp
// Error handling ใน MailboxProcessor
let errorHandlingAgent () =
    let agent = MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            try
                if msg = "error" then
                    raise (System.Exception("เกิดข้อผิดพลาด!"))
                printfn "ประมวลผล: %s" msg
            with
            | ex ->
                printfn "ข้อผิดพลาดใน message '%s': %s" msg ex.Message
            return! loop ()
        }
        loop ()
    )
    
    // จัดการ agent-level errors
    agent.Error.Add(fun ex ->
        printfn "Agent หยุดทำงาน: %s" ex.Message
    )
    
    agent

let errorAgent = errorHandlingAgent ()
errorAgent.Post "ข้อความปกติ"
errorAgent.Post "error"
errorAgent.Post "ข้อความปกติอีกครั้ง"
System.Threading.Thread.Sleep(200)
```

```fsharp
// Restart pattern เมื่อ agent ล้มเหลว
let resilientAgent () =
    let mutable agent: MailboxProcessor<string> option = None
    
    let createAgent () =
        let a = MailboxProcessor<string>.Start(fun inbox ->
            let rec loop () = async {
                let! msg = inbox.Receive()
                if msg = "crash" then
                    failwith "Agent crash!"
                printfn "ประมวลผล: %s" msg
                return! loop ()
            }
            loop ()
        )
        
        a.Error.Add(fun ex ->
            printfn "Agent ล้มเหลว: %s - กำลัง restart..." ex.Message
            System.Threading.Thread.Sleep(100)
            agent <- Some (createAgent ())
        )
        a
    
    agent <- Some (createAgent ())
    agent

let resilient = resilientAgent ()
// สร้าง wrapper สำหรับ post
let post msg = resilient.Value.Value.Post msg

post "ข้อความ 1"
post "crash"  // agent จะ restart
System.Threading.Thread.Sleep(300)
post "ข้อความ 2"
System.Threading.Thread.Sleep(100)
```

## 7. Stateful Actors

```fsharp
// Stateful actor ที่ซับซ้อนขึ้น
type CartMessage =
    | AddItem of name:string * price:float
    | RemoveItem of name:string
    | GetTotal of AsyncReplyChannel<float>
    | GetItems of AsyncReplyChannel<(string * float) list>
    | Checkout of AsyncReplyChannel<{| Total: float; ItemCount: int |}>

type CartState = {
    Items: Map<string, float>
}

let shoppingCartAgent () =
    MailboxProcessor<CartMessage>.Start(fun inbox ->
        let rec loop (state: CartState) = async {
            let! msg = inbox.Receive()
            match msg with
            | AddItem (name, price) ->
                let newState = { state with Items = state.Items |> Map.add name price }
                printfn "เพิ่ม %s (%.2f)" name price
                return! loop newState
            
            | RemoveItem name ->
                let newState = { state with Items = state.Items |> Map.remove name }
                printfn "ลบ %s" name
                return! loop newState
            
            | GetTotal rc ->
                let total = state.Items |> Map.toSeq |> Seq.sumBy snd
                rc.Reply total
                return! loop state
            
            | GetItems rc ->
                rc.Reply (state.Items |> Map.toList)
                return! loop state
            
            | Checkout rc ->
                let total = state.Items |> Map.toSeq |> Seq.sumBy snd
                let count = state.Items.Count
                rc.Reply {| Total = total; ItemCount = count |}
                // ล้าง cart หลัง checkout
                return! loop { Items = Map.empty }
        }
        loop { Items = Map.empty }
    )

// ใช้งาน
let cart = shoppingCartAgent ()

cart.Post (AddItem("ข้าวสาร", 35.0))
cart.Post (AddItem("น้ำมันพืช", 65.0))
cart.Post (AddItem("ผัก", 20.0))

let items = cart.PostAndReply (fun rc -> GetItems rc)
printfn "\nรายการในตะกร้า:"
for name, price in items do
    printfn "  %s: %.2f บาท" name price

let total = cart.PostAndReply (fun rc -> GetTotal rc)
printfn "รวม: %.2f บาท" total

cart.Post (RemoveItem "ผัก")

let summary = cart.PostAndReply (fun rc -> Checkout rc)
printfn "\nCheckout:"
printfn "  จำนวนสินค้า: %d" summary.ItemCount
printfn "  ยอดรวม: %.2f บาท" summary.Total
```

## 8. Actor-Based Counter

```fsharp
// Thread-safe counter ด้วย actor
type CounterMsg =
    | Inc of int
    | Dec of int
    | Get of AsyncReplyChannel<int>
    | Reset

let createCounter (initial: int) =
    MailboxProcessor<CounterMsg>.Start(fun inbox ->
        let rec loop (value: int) = async {
            let! msg = inbox.Receive()
            match msg with
            | Inc n -> return! loop (value + n)
            | Dec n -> return! loop (value - n)
            | Get rc ->
                rc.Reply value
                return! loop value
            | Reset -> return! loop initial
        }
        loop initial
    )

// ทดสอบ thread-safety
let counterStressTest () =
    let counter = createCounter 0
    
    // ส่ง 1000 increments จาก multiple threads
    let tasks = 
        [|1..100|] 
        |> Array.map (fun _ ->
            System.Threading.Tasks.Task.Run(fun () ->
                for _ in 1..10 do
                    counter.Post (Inc 1)
            ))
    
    System.Threading.Tasks.Task.WhenAll(tasks).Wait()
    System.Threading.Thread.Sleep(100)  // รอ process messages
    
    let value = counter.PostAndReply (fun rc -> Get rc)
    printfn "Counter value: %d (คาดหวัง: 1000)" value

counterStressTest ()
```

## 9. Actor-Based Queue

```fsharp
// Queue actor
type QueueMessage<'T> =
    | Enqueue of 'T
    | Dequeue of AsyncReplyChannel<'T option>
    | Peek of AsyncReplyChannel<'T option>
    | Size of AsyncReplyChannel<int>
    | IsEmpty of AsyncReplyChannel<bool>

let createQueueActor<'T> () =
    MailboxProcessor<QueueMessage<'T>>.Start(fun inbox ->
        let rec loop (queue: 'T list) = async {
            let! msg = inbox.Receive()
            match msg with
            | Enqueue item ->
                return! loop (queue @ [item])
            
            | Dequeue rc ->
                match queue with
                | [] ->
                    rc.Reply None
                    return! loop []
                | head :: tail ->
                    rc.Reply (Some head)
                    return! loop tail
            
            | Peek rc ->
                rc.Reply (List.tryHead queue)
                return! loop queue
            
            | Size rc ->
                rc.Reply queue.Length
                return! loop queue
            
            | IsEmpty rc ->
                rc.Reply queue.IsEmpty
                return! loop queue
        }
        loop []
    )

// ใช้งาน
let queue = createQueueActor<string> ()

queue.Post (Enqueue "งานที่ 1")
queue.Post (Enqueue "งานที่ 2")
queue.Post (Enqueue "งานที่ 3")

let size = queue.PostAndReply (fun rc -> Size rc)
printfn "ขนาด queue: %d" size

while not (queue.PostAndReply (fun rc -> IsEmpty rc)) do
    let item = queue.PostAndReply (fun rc -> Dequeue rc)
    printfn "ดึงออก: %A" item
```

## 10. Supervisor Pattern

```fsharp
// Supervisor actor ที่ดูแล worker actors
type WorkerMessage =
    | DoWork of int
    | GetStats of AsyncReplyChannel<{| Processed: int; Errors: int |}>

type SupervisorMessage =
    | StartWorkers of count:int
    | SendWork of data:int
    | GetStatus of AsyncReplyChannel<string>
    | StopAll

let createWorker (id: int) =
    MailboxProcessor<WorkerMessage>.Start(fun inbox ->
        let rec loop (processed: int) (errors: int) = async {
            let! msg = inbox.Receive()
            match msg with
            | DoWork data ->
                try
                    // จำลองการทำงาน
                    if data % 7 = 0 then  // บางงานล้มเหลว
                        raise (System.Exception(sprintf "Worker %d: งาน %d ล้มเหลว" id data))
                    System.Threading.Thread.Sleep(10)
                    printfn "Worker %d: ประมวลผล %d" id data
                    return! loop (processed + 1) errors
                with
                | ex ->
                    printfn "Worker %d: ข้อผิดพลาด - %s" id ex.Message
                    return! loop processed (errors + 1)
            
            | GetStats rc ->
                rc.Reply {| Processed = processed; Errors = errors |}
                return! loop processed errors
        }
        loop 0 0
    )

let createSupervisor () =
    MailboxProcessor<SupervisorMessage>.Start(fun inbox ->
        let rec loop (workers: MailboxProcessor<WorkerMessage> list) (nextWork: int) = async {
            let! msg = inbox.Receive()
            match msg with
            | StartWorkers count ->
                let newWorkers = [1..count] |> List.map createWorker
                printfn "Supervisor: เริ่ม %d workers" count
                return! loop newWorkers nextWork
            
            | SendWork data ->
                if workers.IsEmpty then
                    printfn "Supervisor: ไม่มี workers!"
                else
                    // Round-robin distribution
                    let workerIndex = nextWork % workers.Length
                    workers[workerIndex].Post (DoWork data)
                return! loop workers (nextWork + 1)
            
            | GetStatus rc ->
                let stats = 
                    workers 
                    |> List.mapi (fun i w -> 
                        let s = w.PostAndReply (fun rc -> GetStats rc)
                        sprintf "Worker %d: %d ประมวลผล, %d ข้อผิดพลาด" (i+1) s.Processed s.Errors)
                    |> String.concat "\n"
                rc.Reply stats
                return! loop workers nextWork
            
            | StopAll ->
                printfn "Supervisor: หยุดทุก workers"
                return! loop [] nextWork
        }
        loop [] 0
    )

// ทดสอบ
let supervisor = createSupervisor ()

supervisor.Post (StartWorkers 3)
for i in 1..30 do
    supervisor.Post (SendWork i)

System.Threading.Thread.Sleep(500)

let status = supervisor.PostAndReply (fun rc -> GetStatus rc)
printfn "\nสถานะ:\n%s" status

supervisor.Post StopAll
```

## 11. Multiple Actors Communicating

```fsharp
// Pipeline ของ actors
type PipelineMessage =
    | Process of data:int
    | Stop

let createPipelineStage (name: string) (transform: int -> int) (next: MailboxProcessor<PipelineMessage> option) =
    MailboxProcessor<PipelineMessage>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            match msg with
            | Process data ->
                let result = transform data
                printfn "[%s] %d -> %d" name data result
                
                match next with
                | Some nextActor -> nextActor.Post (Process result)
                | None -> printfn "[%s] ผลสุดท้าย: %d" name result
                
                return! loop ()
            
            | Stop ->
                match next with
                | Some nextActor -> nextActor.Post Stop
                | None -> printfn "Pipeline หยุด"
        }
        loop ()
    )

// สร้าง pipeline: stage1 -> stage2 -> stage3 -> output
let stage3 = createPipelineStage "Stage3" (fun x -> x * x) None
let stage2 = createPipelineStage "Stage2" (fun x -> x + 10) (Some stage3)
let stage1 = createPipelineStage "Stage1" (fun x -> x * 2) (Some stage2)

// ส่งข้อมูลผ่าน pipeline
for i in 1..5 do
    stage1.Post (Process i)

System.Threading.Thread.Sleep(200)
stage1.Post Stop
System.Threading.Thread.Sleep(100)
```

## 12. Actor-Based Pub/Sub

```fsharp
// Publish-Subscribe pattern ด้วย actors
type PubSubMessage<'T> =
    | Subscribe of topic:string * subscriber:MailboxProcessor<'T>
    | Unsubscribe of topic:string * subscriber:MailboxProcessor<'T>
    | Publish of topic:string * message:'T

let createPubSubBus<'T> () =
    MailboxProcessor<PubSubMessage<'T>>.Start(fun inbox ->
        let rec loop (subscriptions: Map<string, MailboxProcessor<'T> list>) = async {
            let! msg = inbox.Receive()
            match msg with
            | Subscribe (topic, subscriber) ->
                let current = subscriptions |> Map.tryFind topic |> Option.defaultValue []
                let updated = subscriptions |> Map.add topic (subscriber :: current)
                return! loop updated
            
            | Unsubscribe (topic, subscriber) ->
                let current = subscriptions |> Map.tryFind topic |> Option.defaultValue []
                let updated = subscriptions |> Map.add topic (current |> List.filter (fun s -> s <> subscriber))
                return! loop updated
            
            | Publish (topic, message) ->
                let subscribers = subscriptions |> Map.tryFind topic |> Option.defaultValue []
                printfn "Publish '%s' ถึง %d subscribers" topic subscribers.Length
                for sub in subscribers do
                    sub.Post message
                return! loop subscriptions
        }
        loop Map.empty
    )

// สร้าง subscribers
let createSubscriber (name: string) =
    MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            printfn "[%s] ได้รับ: %s" name msg
            return! loop ()
        }
        loop ()
    )

// ทดสอบ
let bus = createPubSubBus<string> ()

let sub1 = createSubscriber "Alice"
let sub2 = createSubscriber "Bob"
let sub3 = createSubscriber "Charlie"

// Subscribe
bus.Post (Subscribe ("news", sub1))
bus.Post (Subscribe ("news", sub2))
bus.Post (Subscribe ("sports", sub2))
bus.Post (Subscribe ("sports", sub3))

// Publish
bus.Post (Publish ("news", "ข่าวด่วน: F# 9 ออกแล้ว!"))
bus.Post (Publish ("sports", "ทีมชาติชนะ 3-0!"))
bus.Post (Publish ("news", "ข่าวเศรษฐกิจ: ดีขึ้น"))

System.Threading.Thread.Sleep(200)

// Unsubscribe
bus.Post (Unsubscribe ("news", sub1))
bus.Post (Publish ("news", "ข่าว: Alice ออกแล้ว"))

System.Threading.Thread.Sleep(200)
```

## 13. Agent Pool

```fsharp
// Pool ของ agents สำหรับ load balancing
type PoolMessage = 
    | Work of data:int * reply:AsyncReplyChannel<int>

let createAgentPool (size: int) (processWork: int -> int) =
    let workers = Array.init size (fun id ->
        MailboxProcessor<PoolMessage>.Start(fun inbox ->
            let rec loop () = async {
                let! Work(data, rc) = inbox.Receive()
                let result = processWork data
                rc.Reply result
                return! loop ()
            }
            loop ()
        )
    )
    
    let mutable nextWorker = 0
    let lockObj = obj()
    
    // Return a function to submit work
    fun (data: int) ->
        let workerIdx = 
            lock lockObj (fun () ->
                let idx = nextWorker
                nextWorker <- (nextWorker + 1) % size
                idx
            )
        workers[workerIdx].PostAndReply (fun rc -> Work(data, rc))

// ใช้งาน
let pool = createAgentPool 4 (fun x -> 
    System.Threading.Thread.Sleep(50)
    x * x)

let sw = System.Diagnostics.Stopwatch.StartNew()
let results = 
    [|1..20|] 
    |> Array.Parallel.map (fun i -> pool i)
sw.Stop()

printfn "ประมวลผล 20 items ด้วย pool ใน %d ms" sw.ElapsedMilliseconds
printfn "ผลลัพธ์: %A" results
```

## 14. Throttled Agent

```fsharp
// Agent ที่จำกัดอัตราการประมวลผล
type ThrottledMessage =
    | Request of data:string * reply:AsyncReplyChannel<string>

let createThrottledAgent (maxPerSecond: int) =
    let delayMs = 1000 / maxPerSecond
    
    MailboxProcessor<ThrottledMessage>.Start(fun inbox ->
        let rec loop (lastProcessed: System.DateTime) = async {
            let! Request(data, rc) = inbox.Receive()
            
            // คำนวณ delay ที่จำเป็น
            let elapsed = (System.DateTime.UtcNow - lastProcessed).TotalMilliseconds
            let remainingDelay = float delayMs - elapsed
            
            if remainingDelay > 0.0 then
                do! Async.Sleep (int remainingDelay)
            
            let result = sprintf "ประมวลผล: %s" data
            rc.Reply result
            
            return! loop System.DateTime.UtcNow
        }
        loop System.DateTime.MinValue
    )

// ทดสอบ throttling (จำกัด 5 ต่อวินาที)
let throttled = createThrottledAgent 5

let sw = System.Diagnostics.Stopwatch.StartNew()
for i in 1..10 do
    let result = throttled.PostAndReply (fun rc -> Request(sprintf "งาน%d" i, rc))
    printfn "%s (เวลา: %.0f ms)" result sw.ElapsedMilliseconds
```

## 15. Comparison with Akka.NET

```fsharp
(*
เปรียบเทียบ F# MailboxProcessor กับ Akka.NET:

| คุณสมบัติ        | MailboxProcessor    | Akka.NET               |
|-----------------|---------------------|------------------------|
| Overhead        | น้อย               | มาก (JVM-style)        |
| Distributed     | ไม่รองรับ          | รองรับ                 |
| Supervision     | Manual             | Built-in               |
| Persistence     | ไม่มี built-in     | Akka.Persistence       |
| Clustering      | ไม่รองรับ          | Akka.Cluster           |
| Router          | Manual             | Built-in               |
| Performance     | ดีมาก              | ดี (ต้อง tuning)        |
| Learning Curve  | ต่ำ               | สูง                    |
| Use Case        | Local concurrency  | Distributed systems    |

สำหรับระบบในเครื่องเดียว MailboxProcessor มักจะเหมาะกว่า
สำหรับ distributed system ให้พิจารณา Akka.NET หรือ Orleans
*)
```

```fsharp
// Akka.NET style supervision ด้วย MailboxProcessor
type SupervisionStrategy =
    | Restart
    | Stop
    | Escalate

type SupervisedActorMessage<'T> =
    | Message of 'T
    | ChildFailed of exn

let createSupervisedActor (strategy: SupervisionStrategy) (behavior: 'T -> unit) =
    let mutable child: MailboxProcessor<'T> option = None
    
    let createChild (parent: MailboxProcessor<SupervisedActorMessage<'T>>) =
        MailboxProcessor<'T>.Start(fun inbox ->
            let rec loop () = async {
                let! msg = inbox.Receive()
                try
                    behavior msg
                with
                | ex ->
                    parent.Post (ChildFailed ex)
                return! loop ()
            }
            loop ()
        )
    
    let supervisor = MailboxProcessor<SupervisedActorMessage<'T>>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            match msg with
            | Message m ->
                match child with
                | Some c -> c.Post m
                | None -> printfn "Supervisor: ไม่มี child"
            
            | ChildFailed ex ->
                printfn "Supervisor: Child ล้มเหลว (%s)" ex.Message
                match strategy with
                | Restart ->
                    printfn "Supervisor: Restart child"
                    // child จะถูกสร้างใหม่
                | Stop ->
                    printfn "Supervisor: Stop child"
                    child <- None
                | Escalate ->
                    printfn "Supervisor: Escalate"
                    raise ex
            
            return! loop ()
        }
        loop ()
    )
    
    child <- Some (createChild supervisor)
    supervisor
```

## 16. ตัวอย่างครบ: Chat System

```fsharp
// ระบบ chat อย่างง่ายด้วย actors
type ChatMessage =
    | Join of username:string * inbox:MailboxProcessor<string>
    | Leave of username:string
    | Send of from:string * message:string
    | GetUsers of AsyncReplyChannel<string list>

let createChatRoom (name: string) =
    MailboxProcessor<ChatMessage>.Start(fun inbox ->
        let rec loop (users: Map<string, MailboxProcessor<string>>) = async {
            let! msg = inbox.Receive()
            match msg with
            | Join (username, userInbox) ->
                printfn "[%s] %s เข้าร่วม" name username
                // แจ้งทุกคน
                for KeyValue(_, ui) in users do
                    ui.Post (sprintf ">>> %s เข้าร่วมห้อง" username)
                return! loop (users |> Map.add username userInbox)
            
            | Leave username ->
                printfn "[%s] %s ออกจากห้อง" name username
                let newUsers = users |> Map.remove username
                for KeyValue(_, ui) in newUsers do
                    ui.Post (sprintf ">>> %s ออกจากห้อง" username)
                return! loop newUsers
            
            | Send (from, message) ->
                // ส่งให้ทุกคน ยกเว้นผู้ส่ง
                for KeyValue(username, ui) in users do
                    if username <> from then
                        ui.Post (sprintf "[%s]: %s" from message)
                return! loop users
            
            | GetUsers rc ->
                rc.Reply (users |> Map.keys |> Seq.toList)
                return! loop users
        }
        loop Map.empty
    )

// สร้าง user actor
let createUser (username: string) =
    MailboxProcessor<string>.Start(fun inbox ->
        let rec loop () = async {
            let! msg = inbox.Receive()
            printfn "[%s] ได้รับ: %s" username msg
            return! loop ()
        }
        loop ()
    )

// ทดสอบ
let room = createChatRoom "F# Chat"

let alice = createUser "Alice"
let bob = createUser "Bob"
let charlie = createUser "Charlie"

room.Post (Join ("Alice", alice))
room.Post (Join ("Bob", bob))
room.Post (Join ("Charlie", charlie))

System.Threading.Thread.Sleep(50)

room.Post (Send ("Alice", "สวัสดีทุกคน!"))
room.Post (Send ("Bob", "สวัสดี Alice!"))

let users = room.PostAndReply (fun rc -> GetUsers rc)
printfn "\nผู้ใช้ในห้อง: %A" users

room.Post (Leave "Charlie")

System.Threading.Thread.Sleep(100)
room.Post (Send ("Alice", "แล้ว Charlie ไปไหน?"))

System.Threading.Thread.Sleep(100)
```

## 17. Performance Considerations

```fsharp
// ทดสอบ performance ของ MailboxProcessor
let performanceTest () =
    let agent = MailboxProcessor<int>.Start(fun inbox ->
        let rec loop (sum: int64) (count: int64) = async {
            let! n = inbox.Receive()
            if n = -1 then
                printfn "ผลรวม: %d, จำนวน: %d" sum count
            else
                return! loop (sum + int64 n) (count + 1L)
        }
        loop 0L 0L
    )
    
    let sw = System.Diagnostics.Stopwatch.StartNew()
    
    // ส่ง 1 ล้าน messages
    for i in 1..1_000_000 do
        agent.Post i
    
    agent.Post -1  // สัญญาณหยุด
    System.Threading.Thread.Sleep(2000)  // รอผลลัพธ์
    
    sw.Stop()
    printfn "ส่ง 1M messages ใน %d ms" sw.ElapsedMilliseconds

performanceTest ()
```

```fsharp
// Batch processing เพื่อเพิ่มประสิทธิภาพ
type BatchMessage =
    | Single of int
    | Batch of int[]
    | FlushAndReport of AsyncReplyChannel<int>

let batchAgent (batchSize: int) =
    MailboxProcessor<BatchMessage>.Start(fun inbox ->
        let rec loop (buffer: int list) (total: int) = async {
            let! msg = inbox.Receive()
            match msg with
            | Single n ->
                let newBuffer = n :: buffer
                if newBuffer.Length >= batchSize then
                    // ประมวลผล batch
                    let batchResult = newBuffer |> List.sum
                    return! loop [] (total + batchResult)
                else
                    return! loop newBuffer total
            
            | Batch items ->
                let result = items |> Array.sum
                return! loop buffer (total + result)
            
            | FlushAndReport rc ->
                let remaining = buffer |> List.sum
                rc.Reply (total + remaining)
                return! loop [] 0
        }
        loop [] 0
    )

// ทดสอบ
let batcher = batchAgent 100

let sw = System.Diagnostics.Stopwatch.StartNew()
for i in 1..10_000 do
    batcher.Post (Single i)

let total = batcher.PostAndReply (fun rc -> FlushAndReport rc)
sw.Stop()
printfn "ผลรวม: %d, เวลา: %d ms" total sw.ElapsedMilliseconds
```

## สรุป

```fsharp
(*
MailboxProcessor - สรุปจุดสำคัญ:

1. สร้างด้วย MailboxProcessor<T>.Start(body)
2. body คือ function รับ inbox และคืน Async<unit>
3. inbox.Receive() รอ message ถัดไป
4. agent.Post() ส่ง message แบบ fire-and-forget
5. agent.PostAndReply() ส่งและรอ response
6. ใช้ discriminated union สำหรับ messages
7. State ถูกเก็บเป็น parameter ของ recursive loop
8. MailboxProcessor เป็น single-threaded ต่อ actor
9. ไม่ต้องใช้ locks
10. Error ถูก handle ผ่าน agent.Error event

Best Practices:
- ออกแบบ messages ให้ชัดเจนด้วย DU
- ใช้ AsyncReplyChannel สำหรับ request-response
- Keep state immutable
- Handle errors ใน agent ไม่ให้ crash
- ใช้ supervisor สำหรับ restart logic
*)

printfn "MailboxProcessor - สรุปเสร็จ!"
```
