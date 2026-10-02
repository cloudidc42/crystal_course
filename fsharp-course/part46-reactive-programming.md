# Part 46 - Reactive Programming

## บทนำ

Reactive Programming เป็น programming paradigm ที่เน้นการทำงานกับ asynchronous data streams F# มี built-in support ผ่าน `IObservable<T>` และ `IObserver<T>` interfaces รวมถึง `Observable` module และ F# Events

## 1. IObservable<T> และ IObserver<T>

```fsharp
open System

// IObservable<T> - แหล่งข้อมูลที่สามารถ subscribe ได้
// IObserver<T> - ผู้รับข้อมูล

// สร้าง custom observable อย่างง่าย
let createSimpleObservable (values: 'a list) : IObservable<'a> =
    { new IObservable<'a> with
        member _.Subscribe(observer: IObserver<'a>) =
            // ส่งค่าทั้งหมด
            for value in values do
                observer.OnNext(value)
            observer.OnCompleted()
            
            // คืน IDisposable สำหรับ unsubscribe
            { new IDisposable with
                member _.Dispose() = ()
            }
    }

// สร้าง observer
let myObserver = 
    { new IObserver<int> with
        member _.OnNext(value) = printfn "รับค่า: %d" value
        member _.OnError(exn) = printfn "ข้อผิดพลาด: %s" exn.Message
        member _.OnCompleted() = printfn "เสร็จสิ้น!"
    }

// Subscribe
let obs = createSimpleObservable [1; 2; 3; 4; 5]
let subscription = obs.Subscribe(myObserver)
subscription.Dispose()
```

```fsharp
// Observer ที่มี action functions
let createObserver (onNext: 'a -> unit) (onError: exn -> unit) (onCompleted: unit -> unit) =
    { new IObserver<'a> with
        member _.OnNext(value) = onNext value
        member _.OnError(exn) = onError exn
        member _.OnCompleted() = onCompleted ()
    }

// สร้างง่ายขึ้น
let simpleObserver<'a> (action: 'a -> unit) =
    createObserver action (fun e -> printfn "Error: %s" e.Message) (fun () -> ())
```

## 2. Observable Module ใน F#

```fsharp
// F# มี Observable module built-in
let observableModuleExample () =
    // สร้าง event
    let event = Event<int>()
    
    // Observable จาก event
    let obs = event.Publish
    
    // Subscribe ด้วย Observable module
    let sub1 = obs |> Observable.subscribe (fun n -> 
        printfn "Observer 1: %d" n)
    
    let sub2 = obs |> Observable.subscribe (fun n -> 
        printfn "Observer 2: %d" n)
    
    // Trigger events
    event.Trigger(1)
    event.Trigger(2)
    event.Trigger(3)
    
    // Unsubscribe
    sub1.Dispose()
    
    event.Trigger(4)  // Observer 2 เท่านั้นที่จะรับ
    
    sub2.Dispose()

observableModuleExample ()
```

```fsharp
// Observable.map - แปลงค่า
let observableMap () =
    let event = Event<int>()
    let obs = event.Publish
    
    // Map: แปลง int -> string
    let mapped = obs |> Observable.map (fun n -> sprintf "ค่า: %d" n)
    
    use _ = mapped |> Observable.subscribe (fun s -> printfn "%s" s)
    
    event.Trigger(10)
    event.Trigger(20)
    event.Trigger(30)
```

```fsharp
// Observable.filter - กรองค่า
let observableFilter () =
    let event = Event<int>()
    let obs = event.Publish
    
    // Filter: เฉพาะเลขคู่
    let evens = obs |> Observable.filter (fun n -> n % 2 = 0)
    
    use _ = evens |> Observable.subscribe (fun n -> 
        printfn "เลขคู่: %d" n)
    
    for i in 1..10 do
        event.Trigger(i)
```

## 3. Event.add, Event.map, Event.filter

```fsharp
// Event module
let eventModuleExample () =
    let event = new Event<string>()
    
    // Event.add - เพิ่ม handler
    event.Publish |> Event.add (fun msg -> 
        printfn "Handler 1: %s" msg)
    
    // Event.map - แปลงก่อน subscribe
    event.Publish 
    |> Event.map (fun msg -> msg.ToUpper())
    |> Event.add (fun msg -> printfn "Handler 2 (upper): %s" msg)
    
    // Event.filter - กรองก่อน subscribe
    event.Publish 
    |> Event.filter (fun msg -> msg.StartsWith("A"))
    |> Event.add (fun msg -> printfn "Handler 3 (A only): %s" msg)
    
    // Trigger
    event.Trigger("Apple")
    event.Trigger("Banana")
    event.Trigger("Avocado")
    event.Trigger("Cherry")

eventModuleExample ()
```

```fsharp
// Event.choose - filter และ map ในขั้นตอนเดียว
let eventChoose () =
    let event = new Event<string>()
    
    event.Publish
    |> Event.choose (fun s ->
        match System.Int32.TryParse(s) with
        | true, n -> Some (n * 2)
        | false, _ -> None)
    |> Event.add (fun n -> printfn "จำนวนคูณ 2: %d" n)
    
    event.Trigger("5")
    event.Trigger("hello")
    event.Trigger("10")
    event.Trigger("world")
    event.Trigger("3")
```

```fsharp
// Event.scan - เก็บ state สะสม
let eventScan () =
    let event = new Event<int>()
    
    // สะสม sum
    event.Publish
    |> Event.scan (fun sum n -> sum + n) 0
    |> Event.add (fun sum -> printfn "ผลรวมสะสม: %d" sum)
    
    for i in 1..5 do
        event.Trigger(i)
```

## 4. Observable.subscribe

```fsharp
// Observable.subscribe variants
let subscribeVariants () =
    let event = new Event<int>()
    let obs = event.Publish
    
    // Subscribe ด้วย onNext เท่านั้น
    let sub1 = obs |> Observable.subscribe (fun n -> printfn "Simple: %d" n)
    
    // Subscribe ด้วย full observer
    let sub2 = obs.Subscribe(
        (fun n -> printfn "OnNext: %d" n),
        (fun e -> printfn "OnError: %s" e.Message),
        (fun () -> printfn "OnCompleted")
    )
    
    event.Trigger(1)
    event.Trigger(2)
    
    sub1.Dispose()
    sub2.Dispose()
```

## 5. Observable.map และ Observable.filter

```fsharp
// Observable operators chain
let observableChain () =
    let event = new Event<int>()
    
    event.Publish
    |> Observable.filter (fun n -> n > 0)
    |> Observable.map (fun n -> n * n)
    |> Observable.map (fun n -> sprintf "กำลังสอง: %d" n)
    |> Observable.subscribe (fun s -> printfn "%s" s)
    |> ignore
    
    for i in -2..5 do
        event.Trigger(i)
```

```fsharp
// Observable.pairwise - รับคู่ค่า current และ previous
let observablePairwise () =
    let event = new Event<int>()
    
    event.Publish
    |> Observable.pairwise
    |> Observable.map (fun (prev, curr) -> curr - prev)
    |> Observable.subscribe (fun diff -> 
        printfn "การเปลี่ยนแปลง: %d" diff)
    |> ignore
    
    [10; 15; 12; 20; 18] |> List.iter event.Trigger
```

```fsharp
// Observable.scan - stateful transformation
let observableScan () =
    let event = new Event<float>()
    
    // คำนวณ moving average
    event.Publish
    |> Observable.scan (fun (values: float list) n -> 
        let recent = (n :: values) |> List.truncate 3
        recent) []
    |> Observable.map (fun values -> 
        List.average values)
    |> Observable.subscribe (fun avg -> 
        printfn "Moving avg: %.2f" avg)
    |> ignore
    
    [10.0; 20.0; 15.0; 25.0; 18.0; 22.0] |> List.iter event.Trigger
```

## 6. Observable.merge

```fsharp
// Observable.merge - รวม 2 observables
let observableMerge () =
    let event1 = new Event<string>()
    let event2 = new Event<string>()
    
    let merged = Observable.merge event1.Publish event2.Publish
    
    use _ = merged |> Observable.subscribe (fun msg ->
        printfn "Merged: %s" msg)
    
    event1.Trigger("จาก Event 1")
    event2.Trigger("จาก Event 2")
    event1.Trigger("Event 1 อีกครั้ง")
    event2.Trigger("Event 2 อีกครั้ง")
```

```fsharp
// Merge หลาย observables
let mergeMany (observables: IObservable<'a> list) : IObservable<'a> =
    match observables with
    | [] -> 
        { new IObservable<'a> with
            member _.Subscribe(obs) =
                obs.OnCompleted()
                { new IDisposable with member _.Dispose() = () }
        }
    | [single] -> single
    | first :: rest -> 
        rest |> List.fold Observable.merge first

// ตัวอย่าง
let mergeMany_test () =
    let events = [for _ in 1..4 -> new Event<int>()]
    let observables = events |> List.map (fun e -> e.Publish :> IObservable<int>)
    
    use _ = mergeMany observables |> Observable.subscribe (fun n ->
        printfn "ได้รับ: %d" n)
    
    events[0].Trigger(1)
    events[1].Trigger(2)
    events[2].Trigger(3)
    events[3].Trigger(4)

mergeMany_test ()
```

## 7. Observable.combineLatest Concept

```fsharp
// combineLatest - รวมค่าล่าสุดจาก 2 observables
// F# ไม่มี built-in แต่เราสร้างได้

let combineLatest (obs1: IObservable<'a>) (obs2: IObservable<'b>) : IObservable<'a * 'b> =
    { new IObservable<'a * 'b> with
        member _.Subscribe(observer) =
            let mutable latest1: 'a option = None
            let mutable latest2: 'b option = None
            
            let emit () =
                match latest1, latest2 with
                | Some v1, Some v2 -> observer.OnNext(v1, v2)
                | _ -> ()
            
            let sub1 = obs1.Subscribe(
                (fun v -> latest1 <- Some v; emit ()),
                (fun e -> observer.OnError e),
                (fun () -> ())
            )
            
            let sub2 = obs2.Subscribe(
                (fun v -> latest2 <- Some v; emit ()),
                (fun e -> observer.OnError e),
                (fun () -> 
                    if latest1.IsSome && latest2.IsSome then
                        observer.OnCompleted())
            )
            
            { new IDisposable with
                member _.Dispose() =
                    sub1.Dispose()
                    sub2.Dispose()
            }
    }

// ทดสอบ
let combineLatestTest () =
    let event1 = new Event<string>()
    let event2 = new Event<int>()
    
    use _ = combineLatest event1.Publish event2.Publish
    |> Observable.subscribe (fun (s, n) ->
        printfn "Combined: %s + %d" s n)
    
    event1.Trigger("A")
    event2.Trigger(1)     // A + 1
    event2.Trigger(2)     // A + 2
    event1.Trigger("B")   // B + 2
    event1.Trigger("C")   // C + 2
    event2.Trigger(3)     // C + 3

combineLatestTest ()
```

## 8. Subject<T>

```fsharp
// Subject คือ Observable ที่เป็น Observer ด้วย (hot observable)
// F# ไม่มี built-in แต่สามารถสร้างได้

type Subject<'T>() =
    let observers = System.Collections.Generic.List<IObserver<'T>>()
    let lockObj = obj()
    let mutable completed = false
    
    interface IObservable<'T> with
        member _.Subscribe(observer) =
            lock lockObj (fun () ->
                if not completed then
                    observers.Add(observer)
            )
            { new IDisposable with
                member _.Dispose() =
                    lock lockObj (fun () ->
                        observers.Remove(observer) |> ignore
                    )
            }
    
    interface IObserver<'T> with
        member _.OnNext(value) =
            lock lockObj (fun () ->
                for obs in observers |> Seq.toList do
                    obs.OnNext(value)
            )
        
        member _.OnError(exn) =
            lock lockObj (fun () ->
                for obs in observers |> Seq.toList do
                    obs.OnError(exn)
            )
        
        member _.OnCompleted() =
            lock lockObj (fun () ->
                completed <- true
                for obs in observers |> Seq.toList do
                    obs.OnCompleted()
            )
    
    // Helper methods
    member this.OnNext(value) = (this :> IObserver<'T>).OnNext(value)
    member this.OnCompleted() = (this :> IObserver<'T>).OnCompleted()
    member this.AsObservable() = this :> IObservable<'T>

// ทดสอบ Subject
let subjectTest () =
    let subject = Subject<string>()
    
    // Subscribe
    use sub1 = subject.AsObservable() |> Observable.subscribe (fun s ->
        printfn "Observer 1: %s" s)
    
    use sub2 = subject.AsObservable() |> Observable.subscribe (fun s ->
        printfn "Observer 2: %s" s)
    
    // Publish values
    subject.OnNext("ข้อความ 1")
    subject.OnNext("ข้อความ 2")
    
    sub1.Dispose()  // Observer 1 หยุดรับ
    
    subject.OnNext("ข้อความ 3")  // เฉพาะ Observer 2
    subject.OnCompleted()

subjectTest ()
```

## 9. Hot vs Cold Observables

```fsharp
(*
Cold Observable:
- เริ่มส่งข้อมูลเมื่อมี subscriber
- แต่ละ subscriber รับข้อมูลตั้งแต่ต้น
- ตัวอย่าง: F# Async, file reading

Hot Observable:
- ส่งข้อมูลโดยไม่สนว่ามี subscriber หรือไม่
- Subscribers รับเฉพาะข้อมูลที่เกิดขึ้นหลัง subscribe
- ตัวอย่าง: UI events, Subject, sensors
*)

// Cold observable - แต่ละ subscriber รับข้อมูลตั้งแต่ต้น
let coldObservable = 
    { new IObservable<int> with
        member _.Subscribe(obs) =
            // เริ่มส่งเมื่อมี subscriber
            for i in 1..3 do
                obs.OnNext(i)
            obs.OnCompleted()
            { new IDisposable with member _.Dispose() = () }
    }

printfn "Cold Observable:"
coldObservable.Subscribe(fun n -> printfn "Sub1: %d" n) |> ignore
coldObservable.Subscribe(fun n -> printfn "Sub2: %d" n) |> ignore

// Hot observable - ส่งข้อมูลโดยไม่สนใจ subscriber
let subject = Subject<int>()
let hotObservable = subject.AsObservable()

printfn "\nHot Observable:"
use sub1 = hotObservable.Subscribe(fun n -> printfn "Sub1: %d" n)
subject.OnNext(1)  // Sub1 รับ

use sub2 = hotObservable.Subscribe(fun n -> printfn "Sub2: %d" n)
subject.OnNext(2)  // Sub1 และ Sub2 รับ

sub1.Dispose()
subject.OnNext(3)  // Sub2 รับเท่านั้น
```

## 10. ตัวอย่าง: UI Events (จำลอง)

```fsharp
// จำลอง UI events
type ButtonClick = { ButtonId: string; Timestamp: DateTime }
type TextInput = { InputId: string; Text: string }
type UIEvent =
    | Click of ButtonClick
    | TextChanged of TextInput

// Event bus สำหรับ UI
let uiEventBus = Subject<UIEvent>()

// Handlers แยกตามประเภท
let clickEvents = 
    uiEventBus.AsObservable()
    |> Observable.choose (function
        | Click c -> Some c
        | _ -> None)

let textEvents =
    uiEventBus.AsObservable()
    |> Observable.choose (function
        | TextChanged t -> Some t
        | _ -> None)

// Subscribe
use _ = clickEvents |> Observable.subscribe (fun c ->
    printfn "Click: %s at %s" c.ButtonId (c.Timestamp.ToString("HH:mm:ss")))

use _ = textEvents |> Observable.subscribe (fun t ->
    printfn "Text '%s': %s" t.InputId t.Text)

// จำลองการใช้งาน
uiEventBus.OnNext(Click { ButtonId = "btn-login"; Timestamp = DateTime.UtcNow })
uiEventBus.OnNext(TextChanged { InputId = "txt-username"; Text = "john" })
uiEventBus.OnNext(TextChanged { InputId = "txt-password"; Text = "***" })
uiEventBus.OnNext(Click { ButtonId = "btn-submit"; Timestamp = DateTime.UtcNow })
```

## 11. ตัวอย่าง: Real-time Data Stream

```fsharp
// Real-time data stream (จำลอง sensor data)
type SensorReading = {
    SensorId: string
    Temperature: float
    Humidity: float
    Timestamp: DateTime
}

let createSensorStream (sensorId: string) : IObservable<SensorReading> =
    { new IObservable<SensorReading> with
        member _.Subscribe(observer) =
            let rng = Random()
            let mutable running = true
            
            let thread = System.Threading.Thread(fun () ->
                let mutable temp = 25.0
                let mutable humidity = 60.0
                
                while running do
                    temp <- temp + (rng.NextDouble() - 0.5) * 2.0
                    humidity <- humidity + (rng.NextDouble() - 0.5) * 5.0
                    
                    observer.OnNext({
                        SensorId = sensorId
                        Temperature = System.Math.Round(temp, 1)
                        Humidity = System.Math.Round(humidity, 1)
                        Timestamp = DateTime.UtcNow
                    })
                    
                    System.Threading.Thread.Sleep(100)
            )
            
            thread.IsBackground <- true
            thread.Start()
            
            { new IDisposable with
                member _.Dispose() = running <- false }
    }

// ใช้งาน
let processSensorData () =
    let sensor = createSensorStream "SENSOR-001"
    
    // Alert เมื่ออุณหภูมิสูง
    let tempAlerts = 
        sensor
        |> Observable.filter (fun r -> r.Temperature > 26.0)
        |> Observable.map (fun r -> 
            sprintf "⚠️ อุณหภูมิสูง! %.1f°C ที่ %s" r.Temperature r.SensorId)
    
    // ค่าเฉลี่ยทุก 5 samples
    let avgReadings =
        sensor
        |> Observable.scan (fun acc r -> r :: acc |> List.truncate 5) []
        |> Observable.filter (fun lst -> lst.Length = 5)
        |> Observable.map (fun lst ->
            let avgTemp = lst |> List.averageBy (fun r -> r.Temperature)
            let avgHumidity = lst |> List.averageBy (fun r -> r.Humidity)
            avgTemp, avgHumidity)
    
    use alertSub = tempAlerts |> Observable.subscribe (fun msg ->
        printfn "%s" msg)
    
    use avgSub = avgReadings |> Observable.subscribe (fun (temp, hum) ->
        printfn "เฉลี่ย 5 samples: %.1f°C, %.1f%%" temp hum)
    
    System.Threading.Thread.Sleep(1000)
    printfn "หยุด monitoring"

processSensorData ()
```

## 12. Observable Combinators

```fsharp
// สร้าง observable combinators เพิ่มเติม
module ObservableExt =
    
    // throttle - จำกัดความถี่
    let throttle (intervalMs: int) (source: IObservable<'a>) : IObservable<'a> =
        { new IObservable<'a> with
            member _.Subscribe(observer) =
                let mutable lastEmit = DateTime.MinValue
                
                source.Subscribe(
                    (fun value ->
                        let now = DateTime.UtcNow
                        if (now - lastEmit).TotalMilliseconds >= float intervalMs then
                            lastEmit <- now
                            observer.OnNext(value)),
                    observer.OnError,
                    observer.OnCompleted
                )
        }
    
    // debounce - รอให้ไม่มี event ช่วงหนึ่งก่อน emit
    let debounce (waitMs: int) (source: IObservable<'a>) : IObservable<'a> =
        { new IObservable<'a> with
            member _.Subscribe(observer) =
                let mutable lastValue: 'a option = None
                let mutable timer: System.Threading.Timer option = None
                
                source.Subscribe(
                    (fun value ->
                        lastValue <- Some value
                        timer |> Option.iter (fun t -> t.Dispose())
                        timer <- Some (new System.Threading.Timer(
                            (fun _ ->
                                lastValue |> Option.iter observer.OnNext
                                lastValue <- None),
                            null, waitMs, -1))),
                    observer.OnError,
                    (fun () ->
                        lastValue |> Option.iter observer.OnNext
                        observer.OnCompleted())
                )
        }
    
    // take - รับ n ค่าแล้วหยุด
    let take (count: int) (source: IObservable<'a>) : IObservable<'a> =
        { new IObservable<'a> with
            member _.Subscribe(observer) =
                let mutable received = 0
                let mutable sub: IDisposable = null
                
                sub <- source.Subscribe(
                    (fun value ->
                        if received < count then
                            received <- received + 1
                            observer.OnNext(value)
                            if received = count then
                                observer.OnCompleted()
                                sub.Dispose()),
                    observer.OnError,
                    observer.OnCompleted
                )
                sub
        }
    
    // skip - ข้าม n ค่าแรก
    let skip (count: int) (source: IObservable<'a>) : IObservable<'a> =
        { new IObservable<'a> with
            member _.Subscribe(observer) =
                let mutable skipped = 0
                
                source.Subscribe(
                    (fun value ->
                        if skipped >= count then
                            observer.OnNext(value)
                        else
                            skipped <- skipped + 1),
                    observer.OnError,
                    observer.OnCompleted
                )
        }

// ทดสอบ
let combinatorsTest () =
    let event = new Event<int>()
    
    // Throttle
    event.Publish
    |> ObservableExt.throttle 200
    |> Observable.subscribe (fun n -> printfn "Throttled: %d" n)
    |> ignore
    
    // Take first 3
    event.Publish
    |> ObservableExt.take 3
    |> Observable.subscribe (fun n -> printfn "First 3: %d" n)
    |> ignore
    
    // Skip first 3
    event.Publish
    |> ObservableExt.skip 3
    |> Observable.subscribe (fun n -> printfn "After 3: %d" n)
    |> ignore
    
    for i in 1..7 do
        event.Trigger(i)
        System.Threading.Thread.Sleep(50)

combinatorsTest ()
```

## 13. Reactive Pattern: Stock Price Monitor

```fsharp
// ตัวอย่างครบ: Stock price monitoring
type StockQuote = {
    Symbol: string
    Price: decimal
    Volume: int
    Timestamp: DateTime
}

type PriceAlert = {
    Symbol: string
    AlertType: string
    Price: decimal
    Message: string
}

module StockMonitor =
    
    // จำลอง stock data stream
    let createStockStream (symbol: string) (initialPrice: decimal) : IObservable<StockQuote> =
        let subject = Subject<StockQuote>()
        let rng = Random()
        
        let thread = System.Threading.Thread(fun () ->
            let mutable price = initialPrice
            for _ in 1..20 do
                let change = decimal (rng.NextDouble() - 0.48) * 2m
                price <- price + change
                subject.OnNext({
                    Symbol = symbol
                    Price = System.Math.Round(price, 2)
                    Volume = rng.Next(100, 10000)
                    Timestamp = DateTime.UtcNow
                })
                System.Threading.Thread.Sleep(100)
            subject.OnCompleted()
        )
        thread.IsBackground <- true
        thread.Start()
        
        subject.AsObservable()
    
    // ตรวจจับการเปลี่ยนแปลงราคา
    let detectPriceChange (threshold: decimal) (stream: IObservable<StockQuote>) =
        stream
        |> Observable.pairwise
        |> Observable.choose (fun (prev, curr) ->
            let change = (curr.Price - prev.Price) / prev.Price * 100m
            if abs change >= threshold then
                Some {
                    Symbol = curr.Symbol
                    AlertType = if change > 0m then "UP" else "DOWN"
                    Price = curr.Price
                    Message = sprintf "%s: %+.2f%% (%M -> %M)" curr.Symbol change prev.Price curr.Price
                }
            else None)
    
    // คำนวณ moving average
    let movingAverage (windowSize: int) (stream: IObservable<StockQuote>) =
        stream
        |> Observable.scan (fun window q ->
            let updated = q :: window |> List.truncate windowSize
            updated) []
        |> Observable.filter (fun w -> w.Length >= windowSize)
        |> Observable.map (fun w ->
            let avg = w |> List.averageBy (fun q -> q.Price)
            w.Head.Symbol, System.Math.Round(avg, 2))

// ใช้งาน
let stockMonitorExample () =
    let aaplStream = StockMonitor.createStockStream "AAPL" 150m
    let googStream = StockMonitor.createStockStream "GOOG" 2800m
    
    // Monitor price alerts
    use _ = aaplStream
    |> StockMonitor.detectPriceChange 0.5m
    |> Observable.subscribe (fun alert ->
        printfn "⚠️ Alert [%s]: %s" alert.AlertType alert.Message)
    
    // Moving average
    use _ = aaplStream
    |> StockMonitor.movingAverage 5
    |> Observable.subscribe (fun (symbol, avg) ->
        printfn "📊 %s MA5: %M" symbol avg)
    
    // Raw price display
    use _ = Observable.merge aaplStream googStream
    |> Observable.subscribe (fun q ->
        printfn "📈 %s: %M" q.Symbol q.Price)
    
    System.Threading.Thread.Sleep(2500)
    printfn "หยุด monitoring"

stockMonitorExample ()
```

## 14. Error Handling ใน Observables

```fsharp
// Error handling patterns
let errorHandlingObservables () =
    let event = new Event<int>()
    
    // OnError - จัดการ error
    event.Publish
    |> Observable.subscribe (
        (fun n -> 
            if n < 0 then failwith "ค่าลบ!"
            printfn "ค่า: %d" n),
        (fun e -> printfn "Error: %s" e.Message))
    |> ignore
    
    // Catch pattern
    event.Publish
    |> Observable.map (fun n ->
        if n = 0 then raise (DivideByZeroException())
        100 / n)
    |> Observable.subscribe (
        (fun n -> printfn "ผลหาร: %d" n),
        (fun e -> printfn "Caught: %s" e.Message))
    |> ignore
    
    event.Trigger(5)
    event.Trigger(10)
    // event.Trigger(0)  // จะ trigger error
    event.Trigger(4)
```

## 15. Performance และ Memory Management

```fsharp
// สำคัญ: การจัดการ subscriptions
let subscriptionManagement () =
    let event = new Event<int>()
    
    // ❌ Memory leak - ไม่ dispose subscription
    let badSub = event.Publish |> Observable.subscribe (fun n ->
        printfn "Bad: %d" n)
    // badSub ไม่เคยถูก dispose -> memory leak
    
    // ✅ ดีกว่า - ใช้ use หรือ dispose
    use goodSub = event.Publish |> Observable.subscribe (fun n ->
        printfn "Good: %d" n)
    // goodSub จะถูก dispose เมื่อออกจาก scope
    
    event.Trigger(1)
    
    // dispose explicitly
    badSub.Dispose()
    printfn "subscriptions disposed"

subscriptionManagement ()
```

```fsharp
// Composite disposable สำหรับจัดการหลาย subscriptions
type CompositeDisposable() =
    let disposables = System.Collections.Generic.List<IDisposable>()
    
    member _.Add(d: IDisposable) = disposables.Add(d)
    
    interface IDisposable with
        member _.Dispose() =
            for d in disposables do
                d.Dispose()
            disposables.Clear()

let compositeExample () =
    let event1 = new Event<int>()
    let event2 = new Event<string>()
    
    use composite = new CompositeDisposable()
    
    composite.Add(
        event1.Publish |> Observable.subscribe (fun n ->
            printfn "Event1: %d" n))
    
    composite.Add(
        event2.Publish |> Observable.subscribe (fun s ->
            printfn "Event2: %s" s))
    
    event1.Trigger(1)
    event2.Trigger("hello")
    
    // เมื่อ composite ถูก dispose ทุก subscriptions จะถูก dispose ด้วย

compositeExample ()
```

## สรุป

```fsharp
(*
Reactive Programming ใน F# - สรุป:

Built-in:
- IObservable<T>     - interface สำหรับ observable
- IObserver<T>       - interface สำหรับ observer
- Event<T>           - F# event (IObservable)
- Observable module  - operators สำหรับ observables
- Event module       - operators สำหรับ events

Operators หลัก:
- Observable.map        - แปลงค่า
- Observable.filter     - กรองค่า
- Observable.merge      - รวม observables
- Observable.scan       - สะสม state
- Observable.pairwise   - คู่ค่า
- Observable.choose     - filter + map
- Observable.subscribe  - รับค่า
- Event.add             - เพิ่ม event handler

Patterns:
- Hot Observable     - Subject<T>
- Cold Observable    - สร้างจาก factory
- Fan-out            - 1 source -> n consumers
- Fan-in             - n sources -> 1 consumer
- Pipeline           - ประมวลผลเป็น stages
- Error Recovery     - OnError handler

สำหรับ production:
- FSharp.Control.Reactive (wrapper สำหรับ Rx.NET)
- System.Reactive (Rx.NET)
- Elmish (สำหรับ UI)
*)

printfn "Reactive Programming - สรุปเสร็จ!"
```
