# Part 39 - Delegates และ Events

## บทนำ (Introduction)

Delegates ใน .NET เป็น type-safe function pointers ที่ใช้ส่งต่อ functions ระหว่าง components Events ใน F# เป็นกลไกสำหรับ publish/subscribe pattern ซึ่ง F# มี module พิเศษ Event และ Observable ที่ช่วยทำงานกับ events แบบ functional

---

## 1. .NET Delegates in F#

```fsharp
// Delegate ใน .NET
// ใน F# ปกติใช้ function values แทน delegates
// แต่ต้องเข้าใจ delegates เพื่อ interop กับ .NET

// สร้าง delegate type
type IntBinaryOp = delegate of int * int -> int
type StringTransform = delegate of string -> string
type VoidCallback = delegate of unit -> unit

// สร้าง delegate instances
let add = IntBinaryOp(fun a b -> a + b)
let multiply = IntBinaryOp(fun a b -> a * b)
let toUpper = StringTransform(fun s -> s.ToUpper())

// เรียกใช้ delegate
printfn "add(3, 4) = %d" (add.Invoke(3, 4))
printfn "multiply(3, 4) = %d" (multiply.Invoke(3, 4))
printfn "toUpper('hello') = %s" (toUpper.Invoke("hello"))

// Delegate combination (multicast)
let printResult = VoidCallback(fun () -> printfn "First callback")
let printDone = VoidCallback(fun () -> printfn "Second callback")

let combined = System.Delegate.Combine(printResult, printDone) :?> VoidCallback
combined.Invoke()  // Calls both!

// F# function to delegate conversion
let fsharpAdd (a: int) (b: int) = a + b
let delegateFromFSharp = IntBinaryOp(fsharpAdd)
printfn "delegateFromFSharp(5, 6) = %d" (delegateFromFSharp.Invoke(5, 6))
```

---

## 2. Action<T>, Func<T,R>

```fsharp
// .NET built-in delegates

// Action delegates (no return value)
let printInt: System.Action<int> = System.Action<int>(fun n -> printfn "Number: %d" n)
let printString: System.Action<string> = System.Action<string>(fun s -> printfn "String: %s" s)
let printPair: System.Action<string, int> = System.Action<string, int>(fun s n -> printfn "%s=%d" s n)

printInt.Invoke(42)
printString.Invoke("hello")
printPair.Invoke("count", 100)

// Func delegates (with return value)
let square: System.Func<int, int> = System.Func<int, int>(fun x -> x * x)
let add2: System.Func<int, int, int> = System.Func<int, int, int>(fun a b -> a + b)
let greet: System.Func<string, string> = System.Func<string, string>(fun name -> sprintf "Hello, %s!" name)

printfn "\nsquare(5) = %d" (square.Invoke(5))
printfn "add2(3, 4) = %d" (add2.Invoke(3, 4))
printfn "greet('World') = %s" (greet.Invoke("World"))

// Using with .NET LINQ-like operations
let numbers = [1..10]
let filtered = numbers |> List.filter (fun n -> n % 2 = 0)
let doubled = filtered |> List.map (fun n -> n * 2)
printfn "\nEven doubled: %A" doubled

// Predicate
let isPositive: System.Predicate<int> = System.Predicate<int>(fun n -> n > 0)
let positives = [|1; -2; 3; -4; 5|] |> Array.filter (fun n -> isPositive.Invoke(n))
printfn "Positives: %A" positives

// Comparison
let compareByLength: System.Comparison<string> = 
    System.Comparison<string>(fun a b -> compare a.Length b.Length)

let words = [|"banana"; "apple"; "kiwi"; "cherry"|]
System.Array.Sort(words, compareByLength)
printfn "Sorted by length: %A" words
```

---

## 3. Creating Delegates

```fsharp
// หลายวิธีในการสร้าง delegates

// Method 1: Lambda
let lambdaDelegate = System.Action<string>(fun s -> printfn "%s" s)

// Method 2: Named function
let myFunction (s: string) = printfn "Function: %s" s
let funcDelegate = System.Action<string>(myFunction)

// Method 3: Method reference
type Calculator() =
    member this.Double(x: int) = x * 2
    member this.Triple(x: int) = x * 3

let calc = Calculator()
let doubleDelegate = System.Func<int, int>(calc.Double)
let tripleDelegate = System.Func<int, int>(calc.Triple)

printfn "\nDouble(5) = %d" (doubleDelegate.Invoke(5))
printfn "Triple(5) = %d" (tripleDelegate.Invoke(5))

// Delegate chaining
type EventHandler2 = delegate of string -> unit

let handler1 = EventHandler2(fun msg -> printfn "[H1] %s" msg)
let handler2 = EventHandler2(fun msg -> printfn "[H2] %s" msg)
let handler3 = EventHandler2(fun msg -> printfn "[H3] %s" msg)

let multicast = 
    System.Delegate.Combine(handler1, handler2) 
    |> fun d -> System.Delegate.Combine(d, handler3)
    |> (fun d -> d :?> EventHandler2)

printfn "\nMulticast invoke:"
multicast.Invoke("Hello from multicast!")

// Remove handler
let afterRemove = System.Delegate.Remove(multicast, handler2) :?> EventHandler2
printfn "\nAfter removing H2:"
afterRemove.Invoke("Hello again!")

// GetInvocationList - get all registered handlers
printfn "\nHandlers in multicast: %d" (multicast.GetInvocationList().Length)
```

---

## 4. Event Handling

```fsharp
// Events ใน F# ใช้ Event<'T> และ IEvent<'T>

type Button(label: string) =
    let clicked = new Event<unit>()
    let doubleClicked = new Event<unit>()
    let labelChanged = new Event<string>()
    
    let mutable _label = label
    
    member this.Label
        with get() = _label
        and set(value) =
            _label <- value
            labelChanged.Trigger(value)
    
    member this.Click() =
        printfn "Button '%s' clicked" _label
        clicked.Trigger(())
    
    member this.DoubleClick() =
        printfn "Button '%s' double-clicked" _label
        doubleClicked.Trigger(())
    
    // Expose events as IEvent
    member this.Clicked = clicked.Publish
    member this.DoubleClicked = doubleClicked.Publish
    member this.LabelChanged = labelChanged.Publish

// Subscribe to events
let btn = Button("Click Me!")

// Subscribe ด้วย add
btn.Clicked.Add(fun () -> printfn "  [Handler 1] Button was clicked!")
btn.Clicked.Add(fun () -> printfn "  [Handler 2] Another click handler!")
btn.DoubleClicked.Add(fun () -> printfn "  [DC Handler] Double click!")
btn.LabelChanged.Add(fun newLabel -> printfn "  [Label] Changed to: %s" newLabel)

// Trigger events
btn.Click()
btn.Click()
btn.DoubleClick()
btn.Label <- "New Label"
btn.Click()
```

---

## 5. Custom Events

```fsharp
// Custom event args
type OrderEventArgs(orderId: int, status: string, amount: decimal) =
    inherit System.EventArgs()
    member this.OrderId = orderId
    member this.Status = status
    member this.Amount = amount

type OrderUpdatedEventArgs(orderId: int, oldStatus: string, newStatus: string) =
    inherit System.EventArgs()
    member this.OrderId = orderId
    member this.OldStatus = oldStatus
    member this.NewStatus = newStatus

type Order(orderId: int, amount: decimal) =
    let orderCreated = new Event<System.EventHandler<OrderEventArgs>, OrderEventArgs>()
    let orderUpdated = new Event<System.EventHandler<OrderUpdatedEventArgs>, OrderUpdatedEventArgs>()
    let orderCancelled = new Event<System.EventHandler<OrderEventArgs>, OrderEventArgs>()
    
    let mutable status = "Pending"
    
    member this.OrderId = orderId
    member this.Amount = amount
    member this.Status = status
    
    member this.OrderCreated = orderCreated.Publish
    member this.OrderUpdated = orderUpdated.Publish
    member this.OrderCancelled = orderCancelled.Publish
    
    member this.Create() =
        orderCreated.Trigger(this, OrderEventArgs(orderId, status, amount))
    
    member this.UpdateStatus(newStatus: string) =
        let oldStatus = status
        status <- newStatus
        orderUpdated.Trigger(this, OrderUpdatedEventArgs(orderId, oldStatus, newStatus))
    
    member this.Cancel() =
        status <- "Cancelled"
        orderCancelled.Trigger(this, OrderEventArgs(orderId, status, amount))

// Subscribe ด้วย System.EventHandler
let order = Order(1001, 500.0M)

order.OrderCreated.Add(fun args ->
    printfn "Order created: #%d, Status=%s, Amount=%.2M" args.OrderId args.Status args.Amount
)

order.OrderUpdated.Add(fun args ->
    printfn "Order updated: #%d, %s -> %s" args.OrderId args.OldStatus args.NewStatus
)

order.OrderCancelled.Add(fun args ->
    printfn "Order cancelled: #%d" args.OrderId
)

order.Create()
order.UpdateStatus("Processing")
order.UpdateStatus("Shipped")
order.Cancel()
```

---

## 6. IEvent<T>

```fsharp
// IEvent<'T> interface
// Event<'T> implements IEvent<'T>

// Creating observable sequences
type Timer(intervalMs: int) =
    let tick = new Event<int>()
    let mutable count = 0
    let mutable running = false
    
    member this.Tick = tick.Publish
    
    member this.Start() =
        running <- true
        async {
            while running do
                do! Async.Sleep(intervalMs)
                if running then
                    count <- count + 1
                    tick.Trigger(count)
        } |> Async.Start
    
    member this.Stop() = running <- false
    member this.Count = count

// ใช้ IEvent<T>
let timer = Timer(100)
let sub = timer.Tick.Subscribe(fun n -> 
    if n <= 5 then printfn "Tick: %d" n)

timer.Start()
System.Threading.Thread.Sleep(600)
timer.Stop()
(sub :> System.IDisposable).Dispose()
printfn "Total ticks: %d" timer.Count

// IEvent operations
type MessageBus() =
    let messageReceived = new Event<string>()
    
    member this.MessageReceived: IEvent<string> = messageReceived.Publish
    
    member this.Send(message: string) =
        messageReceived.Trigger(message)

let bus = MessageBus()

// Subscribe ด้วย IEvent
let subscription = bus.MessageReceived.Subscribe(fun msg -> printfn "Received: %s" msg)
bus.Send("Hello!")
bus.Send("World!")
subscription.Dispose()  // Unsubscribe
bus.Send("This won't be received")
printfn "Unsubscribed successfully"
```

---

## 7. Event.map, Event.filter, Event.merge

```fsharp
// F# Event module สำหรับ functional event handling

type NumberSource() =
    let generated = new Event<int>()
    
    member this.Generated = generated.Publish
    
    member this.GenerateNumbers() =
        for n in [1..20] do
            generated.Trigger(n)

let source = NumberSource()

// Event.map - transform events
let doubledEvents = source.Generated |> Event.map (fun n -> n * 2)
let squared = source.Generated |> Event.map (fun n -> n * n)

// Event.filter - filter events
let evenEvents = source.Generated |> Event.filter (fun n -> n % 2 = 0)
let primes = 
    source.Generated 
    |> Event.filter (fun n -> 
        if n < 2 then false
        else [2..(int (sqrt (float n)))] |> List.forall (fun d -> n % d <> 0)
    )

// Subscribe to filtered/mapped events
evenEvents.Add(fun n -> printfn "Even: %d" n)
primes.Add(fun n -> printfn "Prime: %d" n)

printfn "=== Generating numbers ==="
source.GenerateNumbers()

// Event.merge - merge multiple event sources
type Button2() =
    let clicked = new Event<string>()
    member this.Name = "Button"
    member this.Clicked = clicked.Publish
    member this.Click() = clicked.Trigger(this.Name)

let btn1 = Button2()
let btn2 = Button2()
let btn3 = Button2()

// Merge all button clicks into one stream
let allClicks = 
    Event.merge btn1.Clicked 
        (Event.merge btn2.Clicked btn3.Clicked)

allClicks.Add(fun source -> printfn "Click from: %s" source)

printfn "\n=== Button clicks ==="
btn1.Click()
btn2.Click()
btn3.Click()
btn1.Click()

// Event.pairwise - pairs of consecutive events
type SensorData() =
    let reading = new Event<float>()
    member this.Reading = reading.Publish
    member this.Record(value: float) = reading.Trigger(value)

let sensor = SensorData()
let changes = sensor.Reading |> Event.pairwise
changes.Add(fun (prev, curr) -> 
    printfn "Change: %.2f -> %.2f (delta=%.2f)" prev curr (curr - prev))

sensor.Record(10.0)
sensor.Record(12.5)
sensor.Record(11.0)
sensor.Record(15.0)
```

---

## 8. Observable Module

```fsharp
// Observable module - Reactive programming

open System

// Observable ทำงานคล้ายกับ Event แต่เป็น cold observable
type TemperatureSensor(name: string) =
    let obs = Event<float>()
    member this.Name = name
    member this.AsObservable = obs.Publish :> IObservable<float>
    member this.ReadTemperature(temp: float) = obs.Trigger(temp)

let sensor1 = TemperatureSensor("Sensor A")
let sensor2 = TemperatureSensor("Sensor B")

// Observable operations
let avgTemperature = sensor1.AsObservable |> Observable.pairwise
// |> Observable.map (fun (a, b) -> (a + b) / 2.0)

// Observable.filter
let highTemperature = 
    sensor1.AsObservable 
    |> Observable.filter (fun t -> t > 35.0)

// Observable.map  
let fahrenheit =
    sensor1.AsObservable
    |> Observable.map (fun celsius -> celsius * 9.0 / 5.0 + 32.0)

// Subscribe
let sub1 = fahrenheit.Subscribe(fun f -> printfn "[F] %.1f°F" f)
let sub2 = highTemperature.Subscribe(fun t -> printfn "[HIGH] %.1f°C - ALERT!" t)

printfn "=== Temperature readings ==="
[25.0; 30.0; 36.5; 28.0; 38.0; 40.0; 22.0]
|> List.iter sensor1.ReadTemperature

sub1.Dispose()
sub2.Dispose()

// Create Observable from sequence
let fromList (items: 'T list) =
    { new IObservable<'T> with
        member this.Subscribe(observer) =
            try
                for item in items do
                    observer.OnNext(item)
                observer.OnCompleted()
            with ex ->
                observer.OnError(ex)
            
            { new IDisposable with
                member this.Dispose() = () } }

let numbers = fromList [1..5]
let sub3 = numbers.Subscribe(
    { new IObserver<int> with
        member this.OnNext(n) = printf "%d " n
        member this.OnError(ex) = printfn "Error: %s" ex.Message
        member this.OnCompleted() = printfn "\nCompleted!" }
)
```

---

## 9. Publishing Events

```fsharp
// Publishing events ในรูปแบบต่างๆ

// Publisher-Subscriber pattern
type EventType =
    | UserLogin of userId: int * username: string
    | UserLogout of userId: int
    | OrderPlaced of orderId: int * amount: decimal
    | PaymentReceived of orderId: int * amount: decimal

type EventBus() =
    let handlers = System.Collections.Generic.Dictionary<System.Type, System.Collections.Generic.List<obj -> unit>>()
    
    let getHandlers (t: System.Type) =
        match handlers.TryGetValue(t) with
        | true, list -> list
        | _ ->
            let list = System.Collections.Generic.List<obj -> unit>()
            handlers.[t] <- list
            list
    
    member this.Subscribe<'T>(handler: 'T -> unit) =
        let wrappedHandler (o: obj) = handler (o :?> 'T)
        getHandlers typeof<'T> |> fun list -> list.Add(wrappedHandler)
        { new IDisposable with
            member this.Dispose() =
                let list = getHandlers typeof<'T>
                list.Remove(wrappedHandler) |> ignore }
    
    member this.Publish<'T>(event: 'T) =
        match handlers.TryGetValue(typeof<'T>) with
        | true, list ->
            for handler in list do
                handler (box event)
        | _ -> ()

let bus = EventBus()

// Subscribe to different event types
let sub1 = bus.Subscribe<UserLogin>(fun login ->
    printfn "[Auth] User '%s' (id=%d) logged in" login.username login.userId
)

let sub2 = bus.Subscribe<OrderPlaced>(fun order ->
    printfn "[Order] New order #%d for %.2M" order.orderId order.amount
)

let sub3 = bus.Subscribe<PaymentReceived>(fun payment ->
    printfn "[Payment] Received %.2M for order #%d" payment.amount payment.orderId
)

// Publish events
printfn "=== Event Stream ==="
bus.Publish(UserLogin(1, "สมชาย"))
bus.Publish(OrderPlaced(1001, 1500.0M))
bus.Publish(PaymentReceived(1001, 1500.0M))
bus.Publish(UserLogout(1))

sub1.Dispose()
bus.Publish(UserLogin(2, "สมหญิง"))  // No auth handler now
bus.Publish(OrderPlaced(1002, 800.0M))  // Order handler still active
```

---

## 10. Subscribing to Events

```fsharp
// หลายวิธีในการ subscribe

type DataStream() =
    let dataReceived = new Event<int[]>()
    let error = new Event<exn>()
    let completed = new Event<unit>()
    
    member this.DataReceived = dataReceived.Publish
    member this.Error = error.Publish
    member this.Completed = completed.Publish
    
    member this.Simulate() =
        async {
            for i in 1..5 do
                let batch = [| i*10; i*10+1; i*10+2 |]
                dataReceived.Trigger(batch)
                do! Async.Sleep(50)
            completed.Trigger(())
        } |> Async.Start

let stream = DataStream()

// Method 1: .Add()
stream.DataReceived.Add(fun data ->
    printfn "[Add] Received batch: %A" data
)

// Method 2: .Subscribe() with IObserver
let observer = 
    { new System.IObserver<int[]> with
        member this.OnNext(data) = printfn "[Observer] Data: %A" data
        member this.OnError(ex) = printfn "[Observer] Error: %s" ex.Message
        member this.OnCompleted() = printfn "[Observer] Stream completed" }

let sub = stream.DataReceived.Subscribe(observer)

// Method 3: Subscribe to all events
stream.Completed.Add(fun () -> printfn "[Completed] All data received")
stream.Error.Add(fun ex -> printfn "[Error] %s" ex.Message)

stream.Simulate()
System.Threading.Thread.Sleep(400)
sub.Dispose()
```

---

## 11. Unsubscribing

```fsharp
// การ unsubscribe ป้องกัน memory leaks

type EventSource() =
    let event = new Event<string>()
    member this.Event = event.Publish
    member this.Fire(msg) = event.Trigger(msg)

let source = EventSource()

// ใช้ Dispose pattern
let handler1 msg = printfn "[H1] %s" msg
let handler2 msg = printfn "[H2] %s" msg

let sub1 = source.Event.Subscribe(handler1)
let sub2 = source.Event.Subscribe(handler2)

source.Fire("First")  // Both handlers fire

sub1.Dispose()  // Unsubscribe H1
source.Fire("Second")  // Only H2 fires

sub2.Dispose()  // Unsubscribe H2
source.Fire("Third")  // No handlers

// Automatic unsubscribe with use
printfn "\n=== Scoped subscription ==="
let outerSource = EventSource()

do
    use scopedSub = outerSource.Event.Subscribe(fun msg -> printfn "[Scoped] %s" msg)
    outerSource.Fire("Inside scope")  // Handler active
    // scopedSub.Dispose() called automatically at end of scope

outerSource.Fire("Outside scope")  // No handler

// Reference-based unsubscribe pattern
type ManagedSubscription(event: IEvent<string>, handler: string -> unit) =
    let sub = event.Subscribe(handler)
    let mutable disposed = false
    
    interface IDisposable with
        member this.Dispose() =
            if not disposed then
                sub.Dispose()
                disposed <- true
    
    member this.IsActive = not disposed

use managedSub = new ManagedSubscription(source.Event, fun msg -> printfn "[Managed] %s" msg)
source.Fire("Test 1")
printfn "Sub active: %b" managedSub.IsActive
(managedSub :> IDisposable).Dispose()
printfn "Sub active: %b" managedSub.IsActive
source.Fire("Test 2")  // No handler
```

---

## 12. Memory Leaks with Events

```fsharp
// ป้องกัน memory leaks จาก events

// Memory leak example (ไม่ unsubscribe)
type LongLivedService() =
    let event = new Event<string>()
    member this.Event = event.Publish
    member this.Raise(msg) = event.Trigger(msg)
    
    // Service ที่อยู่ตลอดชีวิตของ application

type ShortLivedHandler(service: LongLivedService) =
    // Subscribe แต่ไม่ unsubscribe = memory leak!
    do service.Event.Add(fun msg -> printfn "Handling: %s" msg)
    // handler holds reference to ShortLivedHandler, preventing GC

// Safe pattern: WeakReference
type WeakEventManager<'T>() =
    let handlers = System.Collections.Generic.List<System.WeakReference<'T -> unit>>()
    
    member this.Subscribe(handler: 'T -> unit) =
        let weakRef = System.WeakReference<'T -> unit>(handler)
        handlers.Add(weakRef)
        { new IDisposable with
            member this.Dispose() =
                handlers.RemoveAll(fun wr ->
                    match wr.TryGetTarget() with
                    | true, h -> obj.ReferenceEquals(h, handler)
                    | false, _ -> true  // Remove dead refs too
                ) |> ignore }
    
    member this.Invoke(event: 'T) =
        let deadRefs = System.Collections.Generic.List<_>()
        for wr in handlers do
            match wr.TryGetTarget() with
            | true, handler -> handler event
            | false, _ -> deadRefs.Add(wr)
        for dead in deadRefs do
            handlers.Remove(dead) |> ignore

// Subscription management best practices
type SafeSubscriptionManager() =
    let subscriptions = System.Collections.Generic.List<IDisposable>()
    
    member this.Add(sub: IDisposable) =
        subscriptions.Add(sub)
    
    member this.DisposeAll() =
        for sub in subscriptions do
            sub.Dispose()
        subscriptions.Clear()
    
    interface IDisposable with
        member this.Dispose() = this.DisposeAll()

// Pattern: Component ที่ manage subscriptions อย่างถูกต้อง
type UIComponent(dataSource: EventSource) =
    let subscriptions = new SafeSubscriptionManager()
    
    do
        // Subscribe และ track subscription
        subscriptions.Add(
            dataSource.Event.Subscribe(fun msg ->
                printfn "[UI] Update: %s" msg
            )
        )
    
    interface IDisposable with
        member this.Dispose() =
            (subscriptions :> IDisposable).Dispose()
            printfn "UIComponent disposed"

let dataSource = EventSource()
let component = new UIComponent(dataSource)
dataSource.Fire("Data 1")
dataSource.Fire("Data 2")
(component :> IDisposable).Dispose()
dataSource.Fire("Data 3")  // No more handling
```

---

## 13. Reactive Programming Basics

```fsharp
// Reactive programming patterns ด้วย F# events

// Subject - both Observer and Observable
type Subject<'T>() =
    let observers = System.Collections.Generic.List<IObserver<'T>>()
    let mutable completed = false
    
    interface IObservable<'T> with
        member this.Subscribe(observer) =
            if not completed then
                observers.Add(observer)
            { new IDisposable with
                member this.Dispose() = observers.Remove(observer) |> ignore }
    
    interface IObserver<'T> with
        member this.OnNext(value) =
            for obs in List.ofSeq observers do
                obs.OnNext(value)
        member this.OnError(ex) =
            for obs in List.ofSeq observers do
                obs.OnError(ex)
        member this.OnCompleted() =
            completed <- true
            for obs in List.ofSeq observers do
                obs.OnCompleted()
    
    member this.Next(value) = (this :> IObserver<'T>).OnNext(value)
    member this.Complete() = (this :> IObserver<'T>).OnCompleted()
    member this.Error(ex) = (this :> IObserver<'T>).OnError(ex)

// Operator: debounce-like accumulation
let accumulateEvents (source: IEvent<'T>) (windowMs: int) =
    let subject = Subject<'T list>()
    let mutable buffer: 'T list = []
    let mutable timer: System.Threading.Timer option = None
    
    source.Add(fun item ->
        buffer <- item :: buffer
        
        match timer with
        | Some t -> t.Dispose()
        | None -> ()
        
        timer <- Some (new System.Threading.Timer(
            fun _ ->
                let items = List.rev buffer
                buffer <- []
                timer <- None
                subject.Next(items),
            null, windowMs, System.Threading.Timeout.Infinite
        ))
    )
    
    subject :> IObservable<'T list>

// Reactive counter
type ReactiveCounter() =
    let incrementEvent = new Event<unit>()
    let decrementEvent = new Event<unit>()
    let resetEvent = new Event<unit>()
    
    let mutable count = 0
    let changed = Subject<int>()
    
    do
        incrementEvent.Publish.Add(fun () -> 
            count <- count + 1
            changed.Next(count)
        )
        decrementEvent.Publish.Add(fun () -> 
            count <- count - 1
            changed.Next(count)
        )
        resetEvent.Publish.Add(fun () ->
            count <- 0
            changed.Next(count)
        )
    
    member this.Increment() = incrementEvent.Trigger(())
    member this.Decrement() = decrementEvent.Trigger(())
    member this.Reset() = resetEvent.Trigger(())
    member this.Count = count
    member this.Changed: IObservable<int> = changed :> IObservable<int>

let counter = ReactiveCounter()

let sub1 = counter.Changed.Subscribe(fun count ->
    printfn "Count changed to: %d" count
)

let highValues = 
    counter.Changed 
    |> Observable.filter (fun n -> n >= 3)

let sub2 = highValues.Subscribe(fun count ->
    printfn "[HIGH] Count is now high: %d" count
)

counter.Increment()
counter.Increment()
counter.Increment()
counter.Increment()
counter.Decrement()
counter.Reset()

sub1.Dispose()
sub2.Dispose()
```

---

## 14. Event Streams and Async

```fsharp
// Events กับ Async computation

type AsyncEventProcessor<'T, 'R>() =
    let results = System.Collections.Generic.Queue<'R>()
    let processed = new Event<'R>()
    
    member this.Processed = processed.Publish
    
    member this.Process(item: 'T) (processor: 'T -> Async<'R>) =
        async {
            let! result = processor item
            processed.Trigger(result)
        } |> Async.Start

// Async pipeline
type Pipeline2<'T>() =
    let input = new Event<'T>()
    let output = new Event<string>()
    
    do
        input.Publish.Add(fun item ->
            // Process asynchronously
            async {
                do! Async.Sleep(100)  // Simulate async work
                let result = sprintf "Processed: %A" item
                output.Trigger(result)
            } |> Async.Start
        )
    
    member this.Send(item: 'T) = input.Trigger(item)
    member this.Output = output.Publish

let pipeline = Pipeline2<int>()
pipeline.Output.Add(fun result -> printfn "Pipeline result: %s" result)

for i in [1..5] do
    pipeline.Send(i)

System.Threading.Thread.Sleep(800)

// Event-driven state machine
type TrafficLight() =
    let stateChanged = new Event<string>()
    let mutable currentState = "Red"
    let mutable timer: System.Threading.Timer option = None
    
    let getNextState = function
        | "Red" -> "Green"
        | "Green" -> "Yellow"
        | "Yellow" -> "Red"
        | _ -> "Red"
    
    let getDuration = function
        | "Red" -> 3000
        | "Green" -> 2000
        | "Yellow" -> 1000
        | _ -> 1000
    
    member this.StateChanged = stateChanged.Publish
    member this.CurrentState = currentState
    
    member this.Start() =
        let rec transition () =
            let nextState = getNextState currentState
            let duration = getDuration nextState
            currentState <- nextState
            stateChanged.Trigger(currentState)
            timer <- Some (new System.Threading.Timer(
                fun _ -> transition (),
                null, duration, System.Threading.Timeout.Infinite
            ))
        
        stateChanged.Trigger(currentState)
        timer <- Some (new System.Threading.Timer(
            fun _ -> transition (),
            null, getDuration currentState, System.Threading.Timeout.Infinite
        ))
    
    member this.Stop() =
        match timer with
        | Some t -> t.Dispose()
        | None -> ()

let light = TrafficLight()
light.StateChanged.Add(fun state -> printfn "[Traffic Light] %s" state)
light.Start()
System.Threading.Thread.Sleep(7000)
light.Stop()
printfn "Final state: %s" light.CurrentState
```

---

## สรุป (Summary)

```fsharp
printfn "=== Events Summary ==="
printfn ""
printfn "Creating events:"
printfn "  let myEvent = new Event<'T>()"
printfn "  member this.MyEvent = myEvent.Publish"
printfn ""
printfn "Firing events:"
printfn "  myEvent.Trigger(value)"
printfn ""
printfn "Subscribing:"
printfn "  event.Add(handler)"
printfn "  let sub = event.Subscribe(handler)"
printfn "  sub.Dispose() // unsubscribe"
printfn ""
printfn "Event combinators:"
printfn "  Event.map f event"
printfn "  Event.filter pred event"
printfn "  Event.merge e1 e2"
printfn "  Event.pairwise event"
printfn ""
printfn "Observable:"
printfn "  Observable.filter"
printfn "  Observable.map"
printfn "  Observable.pairwise"
printfn ""
printfn "Key principles:"
printfn "  - Always unsubscribe to prevent memory leaks"
printfn "  - Use IDisposable for subscription management"
printfn "  - Prefer functional Event combinators"
```

---

## บทสรุป

Events และ Delegates ใน F# ให้ความสามารถใน:
1. **Event-driven programming** - react to changes
2. **Publish/Subscribe** - loose coupling between components
3. **Reactive patterns** - compose event streams functionally
4. **Async integration** - async event handling
5. **Memory management** - proper unsubscription prevents leaks
