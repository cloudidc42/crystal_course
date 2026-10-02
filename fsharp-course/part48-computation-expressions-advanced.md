# Part 48 - Computation Expressions ขั้นสูง

## บทนำ

Computation Expressions (CE) ใน F# เป็น mechanism ทรงพลังที่ช่วยให้เขียน domain-specific syntax สำหรับ monadic operations บทนี้จะอธิบายวิธีสร้าง CE builder เองอย่างละเอียด

## 1. Full Builder Interface

```fsharp
// Builder class คือ object ที่มี methods พิเศษ
// F# compiler แปลง CE syntax เป็น method calls

type BasicBuilder() =
    // Bind: ใช้สำหรับ let! 
    member _.Bind(m: 'T option, f: 'T -> 'R option) : 'R option =
        match m with
        | Some v -> f v
        | None -> None
    
    // Return: ใช้สำหรับ return
    member _.Return(v: 'T) : 'T option = Some v
    
    // ReturnFrom: ใช้สำหรับ return!
    member _.ReturnFrom(m: 'T option) : 'T option = m
    
    // Zero: ค่า default เมื่อไม่มี return
    member _.Zero() : unit option = Some ()

let basic = BasicBuilder()

// ใช้งาน
let result = basic {
    let! x = Some 10
    let! y = Some 20
    return x + y
}

printfn "ผลลัพธ์: %A" result  // Some 30

let failed = basic {
    let! x = Some 10
    let! y = None  // จะทำให้ทั้งหมดเป็น None
    return x + y
}

printfn "ผลลัพธ์: %A" failed  // None
```

## 2. Return, ReturnFrom, Zero, Combine

```fsharp
// ตัวอย่างครบถ้วนทุก methods
type MaybeBuilder() =
    member _.Bind(m, f) =
        match m with
        | Some v -> f v
        | None -> None
    
    // return value -> wraps ใน context
    member _.Return(v) = Some v
    
    // return! m -> ส่งต่อ context โดยตรง
    member _.ReturnFrom(m) = m
    
    // Zero: ค่า default สำหรับ CE ที่ไม่มี return หรือเป็น unit
    member _.Zero() = Some ()
    
    // Combine: รวม 2 CEs ที่อยู่ต่อกัน
    member _.Combine(m1: 'T option, m2: 'T option) =
        match m1 with
        | Some _ -> m2
        | None -> None
    
    // Delay: ห่อ expression ด้วย function เพื่อ lazy evaluation
    member _.Delay(f: unit -> 'T option) = f
    
    // Run: รัน delayed computation
    member _.Run(f: unit -> 'T option) = f ()

let maybe = MaybeBuilder()

// ทดสอบ Combine
let combined = maybe {
    let! x = Some 10
    printfn "ได้ x = %d" x
    let! y = Some 20  // Combine ถูกเรียกตรงนี้
    return x + y
}
printfn "Combined: %A" combined
```

## 3. Delay และ Run

```fsharp
// Delay ป้องกัน side effects จนกว่าจะต้องการ
type LazyBuilder() =
    member _.Bind(m: Lazy<'T>, f: 'T -> Lazy<'R>) : Lazy<'R> =
        lazy (f (m.Force())).Force()
    
    member _.Return(v: 'T) : Lazy<'T> = lazy v
    
    member _.ReturnFrom(m: Lazy<'T>) : Lazy<'T> = m
    
    member _.Zero() : Lazy<unit> = lazy ()
    
    // Delay: ห่อด้วย Lazy
    member _.Delay(f: unit -> Lazy<'T>) : Lazy<'T> =
        lazy (f ()).Force()
    
    // Run: Force evaluation
    member _.Run(m: Lazy<'T>) : 'T = m.Force()

let lazyBuild = LazyBuilder()

let lazyComputation = lazyBuild {
    printfn "กำลังคำนวณ..."
    let! x = lazy (printfn "Forcing x"; 10)
    let! y = lazy (printfn "Forcing y"; 20)
    return x + y
}

printfn "ยังไม่คำนวณ"
printfn "ผลลัพธ์: %d" lazyComputation  // คำนวณตอนนี้
```

## 4. For Loop Support

```fsharp
// For loop ใน CE ต้องการ For และ Zero methods
type CollectBuilder() =
    member _.Bind(m: 'T list, f: 'T -> 'R list) : 'R list =
        m |> List.collect f
    
    member _.Return(v: 'T) : 'T list = [v]
    
    member _.ReturnFrom(m: 'T list) : 'T list = m
    
    member _.Zero() : 'T list = []
    
    // Yield ใช้สำหรับ yield ใน sequence-like CEs
    member _.Yield(v: 'T) : 'T list = [v]
    
    member _.YieldFrom(m: 'T list) : 'T list = m
    
    // Combine: รวม 2 lists
    member _.Combine(m1: 'T list, m2: 'T list) : 'T list = m1 @ m2
    
    member _.Delay(f: unit -> 'T list) = f
    member _.Run(f: unit -> 'T list) = f ()
    
    // For: iterate over sequence
    member _.For(sequence: 'T seq, f: 'T -> 'R list) : 'R list =
        sequence |> Seq.collect f |> Seq.toList

let collect = CollectBuilder()

// ใช้ for loop ใน CE
let numbers = collect {
    for i in 1..5 do
        yield i
        yield i * 10
}

printfn "Numbers: %A" numbers
```

```fsharp
// CE ที่รองรับ for loop สำหรับ list comprehension
let listComp = collect {
    for x in 1..3 do
        for y in 1..3 do
            yield x * y
}

printfn "List comp: %A" listComp
```

## 5. While Loop Support

```fsharp
// While loop ต้องการ While method
type StateBuilder() =
    member _.Bind(m: 'S -> 'T * 'S, f: 'T -> 'S -> 'R * 'S) : 'S -> 'R * 'S =
        fun s ->
            let (v, s') = m s
            f v s'
    
    member _.Return(v: 'T) : 'S -> 'T * 'S =
        fun s -> (v, s)
    
    member _.ReturnFrom(m: 'S -> 'T * 'S) = m
    
    member _.Zero() : 'S -> unit * 'S =
        fun s -> ((), s)
    
    member _.Combine(m1: 'S -> unit * 'S, m2: 'S -> 'T * 'S) : 'S -> 'T * 'S =
        fun s ->
            let ((), s') = m1 s
            m2 s'
    
    member _.Delay(f: unit -> 'S -> 'T * 'S) = fun s -> f () s
    
    // While: รัน body ตราบใดที่ guard เป็น true
    member _.While(guard: unit -> bool, body: 'S -> unit * 'S) : 'S -> unit * 'S =
        fun s ->
            let mutable state = s
            while guard () do
                let ((), newState) = body state
                state <- newState
            ((), state)

let state = StateBuilder()
```

## 6. TryWith และ TryFinally

```fsharp
// TryWith สำหรับ exception handling ใน CE
type ResultBuilder() =
    member _.Bind(m: Result<'T, 'E>, f: 'T -> Result<'R, 'E>) : Result<'R, 'E> =
        match m with
        | Ok v -> f v
        | Error e -> Error e
    
    member _.Return(v: 'T) : Result<'T, 'E> = Ok v
    
    member _.ReturnFrom(m: Result<'T, 'E>) : Result<'T, 'E> = m
    
    member _.Zero() : Result<unit, 'E> = Ok ()
    
    member _.Delay(f: unit -> Result<'T, 'E>) = f
    member _.Run(f: unit -> Result<'T, 'E>) = f ()
    
    // TryWith: จัดการ exceptions
    member _.TryWith(f: unit -> Result<'T, 'E>, handler: exn -> Result<'T, 'E>) : Result<'T, 'E> =
        try f ()
        with ex -> handler ex
    
    // TryFinally: cleanup เสมอ
    member _.TryFinally(f: unit -> Result<'T, 'E>, finalizer: unit -> unit) : Result<'T, 'E> =
        try f ()
        finally finalizer ()

let result = ResultBuilder()

// ใช้ try/with ใน CE
let safeOperation = result {
    try
        let! x = Ok 10
        if x > 5 then failwith "ค่ามากเกินไป!"
        return x
    with
    | ex -> return! Error ex.Message
}

printfn "Safe: %A" safeOperation
```

```fsharp
// TryFinally ใน CE
let withCleanup = result {
    let resource = "resource"
    try
        let! value = Ok 42
        printfn "ใช้ %s" resource
        return value * 2
    finally
        printfn "ล้างทรัพยากร: %s" resource
}

printfn "Result: %A" withCleanup
```

## 7. Using (IDisposable)

```fsharp
// Using method สำหรับ use keyword ใน CE
type IOBuilder() =
    member _.Bind(m: Result<'T, exn>, f: 'T -> Result<'R, exn>) =
        match m with
        | Ok v -> f v
        | Error e -> Error e
    
    member _.Return(v) = Ok v
    member _.ReturnFrom(m) = m
    member _.Zero() = Ok ()
    member _.Delay(f) = f
    member _.Run(f: unit -> _) = f ()
    
    member _.TryWith(f, h) =
        try f ()
        with ex -> h ex
    
    member _.TryFinally(f, fin) =
        try f ()
        finally fin ()
    
    // Using: จัดการ IDisposable อัตโนมัติ
    member this.Using(resource: 'T :> System.IDisposable, f: 'T -> Result<'R, exn>) =
        this.TryFinally(
            (fun () -> f resource),
            (fun () -> resource.Dispose())
        )

let io = IOBuilder()

// ใช้ use ใน CE
let fileOperation = io {
    use stream = new System.IO.MemoryStream()
    let bytes = System.Text.Encoding.UTF8.GetBytes("Hello, CE!")
    stream.Write(bytes, 0, bytes.Length)
    
    stream.Position <- 0L
    use reader = new System.IO.StreamReader(stream)
    let! content = Ok (reader.ReadToEnd())
    return content
}

printfn "File: %A" fileOperation
```

## 8. Custom Async Builder

```fsharp
// สร้าง Async builder เอง (simplified)
type MyAsync<'T> = { Run: unit -> 'T }

type MyAsyncBuilder() =
    member _.Bind(m: MyAsync<'T>, f: 'T -> MyAsync<'R>) : MyAsync<'R> =
        { Run = fun () -> (f (m.Run())).Run() }
    
    member _.Return(v: 'T) : MyAsync<'T> =
        { Run = fun () -> v }
    
    member _.ReturnFrom(m: MyAsync<'T>) = m
    
    member _.Zero() : MyAsync<unit> =
        { Run = fun () -> () }
    
    member _.Delay(f: unit -> MyAsync<'T>) : MyAsync<'T> =
        { Run = fun () -> (f ()).Run() }
    
    member _.Combine(m1: MyAsync<unit>, m2: MyAsync<'T>) : MyAsync<'T> =
        { Run = fun () -> m1.Run(); m2.Run() }
    
    member _.TryFinally(m: MyAsync<'T>, fin: unit -> unit) : MyAsync<'T> =
        { Run = fun () ->
            try m.Run()
            finally fin () }

let myAsync = MyAsyncBuilder()

// ใช้งาน
let computation = myAsync {
    printfn "ขั้นตอน 1"
    let! x = myAsync { return 10 }
    printfn "ขั้นตอน 2 (x = %d)" x
    let! y = myAsync { return x * 2 }
    return y + 5
}

printfn "ผลลัพธ์: %d" (computation.Run())
```

## 9. State Monad Builder

```fsharp
// State Monad: computation ที่มี implicit state
type State<'S, 'T> = State of ('S -> 'T * 'S)

module State =
    let run s (State f) = f s
    let get = State (fun s -> s, s)
    let put s = State (fun _ -> (), s)
    let modify f = State (fun s -> (), f s)

type StateBuilder() =
    member _.Return(v) = State (fun s -> v, s)
    
    member _.ReturnFrom(m) = m
    
    member _.Bind(State m, f) =
        State (fun s ->
            let (v, s') = m s
            State.run s' (f v))
    
    member _.Zero() = State (fun s -> (), s)
    
    member _.Delay(f) = f ()
    
    member _.Combine(m1, m2) =
        State (fun s ->
            let ((), s') = State.run s m1
            State.run s' m2)

let state = StateBuilder()

// ตัวอย่าง: Stack ด้วย State Monad
type Stack<'T> = 'T list

let push (v: 'T) : State<Stack<'T>, unit> =
    State (fun stack -> (), v :: stack)

let pop<'T> () : State<Stack<'T>, 'T option> =
    State (fun stack ->
        match stack with
        | [] -> None, []
        | h :: t -> Some h, t)

let peek<'T> () : State<Stack<'T>, 'T option> =
    State (fun stack ->
        match stack with
        | [] -> None, stack
        | h :: _ -> Some h, stack)

// ใช้งาน
let stackProgram = state {
    do! push 1
    do! push 2
    do! push 3
    
    let! top = peek ()
    printfn "Top: %A" top
    
    let! popped = pop ()
    printfn "Popped: %A" popped
    
    let! secondTop = peek ()
    printfn "New top: %A" secondTop
    
    return! State.get
}

let (finalStack, _) = State.run [] stackProgram
printfn "Final stack: %A" finalStack
```

## 10. Writer Monad Builder

```fsharp
// Writer Monad: computation ที่รวบรวม log
type Writer<'T, 'W> = Writer of 'T * 'W list

type WriterBuilder() =
    member _.Return(v: 'T) : Writer<'T, 'W> = Writer(v, [])
    
    member _.ReturnFrom(m: Writer<'T, 'W>) = m
    
    member _.Bind(Writer(v, log1), f: 'T -> Writer<'R, 'W>) : Writer<'R, 'W> =
        let Writer(result, log2) = f v
        Writer(result, log1 @ log2)
    
    member _.Zero() = Writer((), [])

// Helper: write to log
let tell (msg: 'W) : Writer<unit, 'W> = Writer((), [msg])

let writer = WriterBuilder()

// ตัวอย่าง: logging computation
let computeWithLog = writer {
    do! tell "เริ่มต้นคำนวณ"
    
    let x = 10
    do! tell (sprintf "ตั้งค่า x = %d" x)
    
    let y = 20
    do! tell (sprintf "ตั้งค่า y = %d" y)
    
    let sum = x + y
    do! tell (sprintf "คำนวณ sum = %d + %d = %d" x y sum)
    
    do! tell "เสร็จสิ้น"
    return sum
}

let Writer(result, logs) = computeWithLog
printfn "ผลลัพธ์: %d" result
printfn "Log:"
for log in logs do
    printfn "  %s" log
```

## 11. Reader Monad Builder

```fsharp
// Reader Monad: computation ที่ depends on environment
type Reader<'E, 'T> = Reader of ('E -> 'T)

module Reader =
    let run env (Reader f) = f env
    let ask = Reader id
    let asks f = Reader f
    let local f (Reader m) = Reader (fun env -> m (f env))

type ReaderBuilder() =
    member _.Return(v: 'T) : Reader<'E, 'T> = Reader (fun _ -> v)
    
    member _.ReturnFrom(m: Reader<'E, 'T>) = m
    
    member _.Bind(Reader m, f: 'T -> Reader<'E, 'R>) : Reader<'E, 'R> =
        Reader (fun env ->
            let v = m env
            Reader.run env (f v))
    
    member _.Zero() = Reader (fun _ -> ())
    member _.Delay(f) = Reader (fun env -> Reader.run env (f ()))

let reader = ReaderBuilder()

// ตัวอย่าง: configuration-based computation
type Config = {
    DatabaseUrl: string
    MaxRetries: int
    Timeout: int
}

let getDbUrl = Reader.asks (fun cfg -> cfg.DatabaseUrl)
let getMaxRetries = Reader.asks (fun cfg -> cfg.MaxRetries)
let getTimeout = Reader.asks (fun cfg -> cfg.Timeout)

let connectToDb = reader {
    let! dbUrl = getDbUrl
    let! retries = getMaxRetries
    let! timeout = getTimeout
    
    printfn "เชื่อมต่อ %s (retry: %d, timeout: %ds)" dbUrl retries timeout
    return sprintf "Connection to %s" dbUrl
}

let config = {
    DatabaseUrl = "postgresql://localhost:5432/mydb"
    MaxRetries = 3
    Timeout = 30
}

let connection = Reader.run config connectToDb
printfn "Connection: %s" connection
```

## 12. Monad Stack

```fsharp
// รวม monads หลายชั้น
// Reader + Writer + State = RWS Monad

type RWS<'R, 'W, 'S, 'T> = 
    RWS of ('R -> 'S -> 'T * 'S * 'W list)

module RWS =
    let run env state (RWS f) = f env state
    let ask = RWS (fun env s -> env, s, [])
    let tell w = RWS (fun _ s -> (), s, [w])
    let get = RWS (fun _ s -> s, s, [])
    let put s = RWS (fun _ _ -> (), s, [])

type RWSBuilder() =
    member _.Return(v) = RWS (fun _ s -> v, s, [])
    
    member _.Bind(RWS m, f) =
        RWS (fun env s ->
            let (v, s', log1) = m env s
            let RWS m2 = f v
            let (result, s'', log2) = m2 env s'
            (result, s'', log1 @ log2))
    
    member _.Zero() = RWS (fun _ s -> (), s, [])

let rws = RWSBuilder()

// ตัวอย่าง
type AppConfig = { AppName: string }
type AppState = { RequestCount: int }

let handleRequest (requestName: string) = rws {
    let! config = RWS.ask
    let! state = RWS.get
    do! RWS.tell (sprintf "[%s] Request: %s (count: %d)" config.AppName requestName state.RequestCount)
    do! RWS.put { state with RequestCount = state.RequestCount + 1 }
    return sprintf "Handled: %s" requestName
}

let program = rws {
    let! r1 = handleRequest "GET /users"
    let! r2 = handleRequest "POST /users"
    let! r3 = handleRequest "GET /products"
    let! state = RWS.get
    do! RWS.tell (sprintf "รวม %d requests" state.RequestCount)
    return [r1; r2; r3]
}

let config2 = { AppName = "MyApp" }
let initState = { RequestCount = 0 }
let (results, finalState, logs) = RWS.run config2 initState program

printfn "Results: %A" results
printfn "Final count: %d" finalState.RequestCount
printfn "Logs:"
for log in logs do printfn "  %s" log
```

## 13. Applicative Style

```fsharp
// Applicative: apply ฟังก์ชันใน context
// สำหรับ independent operations (parallel-ish)

type Validation<'T, 'E> =
    | Valid of 'T
    | Invalid of 'E list

type ValidationBuilder() =
    // Applicative Apply: f <*> x
    member _.Apply(mf: Validation<'T -> 'R, 'E>, mx: Validation<'T, 'E>) : Validation<'R, 'E> =
        match mf, mx with
        | Valid f, Valid x -> Valid (f x)
        | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
        | Invalid e, _ -> Invalid e
        | _, Invalid e -> Invalid e
    
    // Return ใน applicative context
    member _.Return(v: 'T) : Validation<'T, 'E> = Valid v
    
    // Monad Bind (เมื่อต้องการ sequential)
    member _.Bind(m: Validation<'T, 'E>, f: 'T -> Validation<'R, 'E>) : Validation<'R, 'E> =
        match m with
        | Valid v -> f v
        | Invalid e -> Invalid e
    
    member _.ReturnFrom(m) = m
    member _.Zero() = Valid ()

let validation = ValidationBuilder()

// ตรวจสอบข้อมูล
let validateName (name: string) : Validation<string, string> =
    if name.Length >= 2 then Valid name
    else Invalid ["ชื่อต้องมีอย่างน้อย 2 ตัวอักษร"]

let validateAge (age: int) : Validation<int, string> =
    if age >= 0 && age <= 150 then Valid age
    else Invalid ["อายุต้องอยู่ระหว่าง 0-150"]

let validateEmail (email: string) : Validation<string, string> =
    if email.Contains("@") then Valid email
    else Invalid ["อีเมลไม่ถูกต้อง"]

// Validate และรวม errors ทั้งหมด
type User = { Name: string; Age: int; Email: string }

let validateUser (name: string) (age: int) (email: string) =
    let nameResult = validateName name
    let ageResult = validateAge age
    let emailResult = validateEmail email
    
    // รวม validations (applicative style)
    match nameResult, ageResult, emailResult with
    | Valid n, Valid a, Valid e ->
        Valid { Name = n; Age = a; Email = e }
    | _ ->
        let errors = [
            match nameResult with Invalid e -> yield! e | _ -> ()
            match ageResult with Invalid e -> yield! e | _ -> ()
            match emailResult with Invalid e -> yield! e | _ -> ()
        ]
        Invalid errors

// ทดสอบ
let user1 = validateUser "สมชาย" 30 "somchai@example.com"
printfn "User 1: %A" user1  // Valid

let user2 = validateUser "A" -5 "notvalid"
printfn "User 2: %A" user2  // Invalid with multiple errors
```

## 14. ตัวอย่างครบ: Option Builder ขั้นสูง

```fsharp
// Option builder พร้อม features ครบ
type OptionBuilder() =
    member _.Bind(m: 'T option, f: 'T -> 'R option) = Option.bind f m
    member _.Return(v) = Some v
    member _.ReturnFrom(m) = m
    member _.Zero() = Some ()
    member _.Delay(f) = f
    member _.Run(f: unit -> 'T option) = f ()
    
    member this.Combine(m1: unit option, m2: unit -> 'T option) =
        this.Bind(m1, fun () -> m2 ())
    
    member this.TryWith(f: unit -> 'T option, handler: exn -> 'T option) =
        try f ()
        with ex -> handler ex
    
    member this.TryFinally(f: unit -> 'T option, fin: unit -> unit) =
        try f ()
        finally fin ()
    
    member this.Using(resource: 'T :> System.IDisposable, f: 'T -> 'R option) =
        this.TryFinally(fun () -> f resource, resource.Dispose)
    
    member this.While(guard: unit -> bool, body: unit -> unit option) =
        if not (guard ()) then Some ()
        else
            match body () with
            | Some () -> this.While(guard, body)
            | None -> None
    
    member this.For(sequence: 'T seq, f: 'T -> unit option) =
        let enum = sequence.GetEnumerator()
        let rec loop () =
            if enum.MoveNext() then
                match f enum.Current with
                | Some () -> loop ()
                | None -> None
            else Some ()
        loop ()

let option = OptionBuilder()

// ใช้งาน features ต่างๆ
let complexOption = option {
    // let! binding
    let! x = Some 10
    
    // do! (unit option)
    do! (if x > 0 then Some () else None)
    
    // for loop
    let mutable sum = 0
    for i in [1..5] do
        sum <- sum + i
    
    // try/with
    let! parsed =
        try Some (int "42")
        with _ -> None
    
    return x + sum + parsed
}

printfn "Complex: %A" complexOption
```

## 15. Custom CE สำหรับ Async Result

```fsharp
// AsyncResult: Async<Result<T,E>> computation expression
type AsyncResult<'T, 'E> = Async<Result<'T, 'E>>

type AsyncResultBuilder() =
    member _.Return(v: 'T) : AsyncResult<'T, 'E> = async { return Ok v }
    
    member _.ReturnFrom(m: AsyncResult<'T, 'E>) = m
    
    member _.Bind(m: AsyncResult<'T, 'E>, f: 'T -> AsyncResult<'R, 'E>) : AsyncResult<'R, 'E> =
        async {
            let! result = m
            match result with
            | Ok v -> return! f v
            | Error e -> return Error e
        }
    
    member _.Zero() : AsyncResult<unit, 'E> = async { return Ok () }
    
    member _.Delay(f: unit -> AsyncResult<'T, 'E>) = f ()
    
    member _.TryWith(m: AsyncResult<'T, 'E>, handler: exn -> AsyncResult<'T, 'E>) =
        async {
            try return! m
            with ex -> return! handler ex
        }
    
    member _.TryFinally(m: AsyncResult<'T, 'E>, fin: unit -> unit) =
        async {
            try return! m
            finally fin ()
        }
    
    // mapError - เปลี่ยน error type
    member _.Throw(e: 'E) : AsyncResult<'T, 'E> = async { return Error e }

let asyncResult = AsyncResultBuilder()

// ใช้งาน
let safeDivide (a: int) (b: int) : AsyncResult<int, string> = asyncResult {
    if b = 0 then
        return! asyncResult.Throw "หารด้วยศูนย์!"
    return a / b
}

let pipeline = asyncResult {
    let! step1 = safeDivide 100 4
    let! step2 = safeDivide step1 2
    let! step3 = safeDivide step2 1
    return step3
}

let result = Async.RunSynchronously pipeline
printfn "Pipeline: %A" result

let failedPipeline = asyncResult {
    let! step1 = safeDivide 100 4
    let! step2 = safeDivide step1 0  // จะ fail ตรงนี้
    let! step3 = safeDivide step2 1  // ไม่ถูกเรียก
    return step3
}

let failedResult = Async.RunSynchronously failedPipeline
printfn "Failed pipeline: %A" failedResult
```

## 16. ตัวอย่าง: Free Monad

```fsharp
// Free Monad: DSL สำหรับ programs
type Free<'F, 'A> =
    | Pure of 'A
    | Free of 'F * (obj -> Free<'F, 'A>)

// Instructions สำหรับ console DSL
type ConsoleF<'Next> =
    | PrintLine of string * 'Next
    | ReadLine of (string -> 'Next)

// สร้าง smart constructors
let printLine (s: string) : Free<ConsoleF<obj>, unit> =
    Free(PrintLine(s, ()), fun _ -> Pure ())

let readLine () : Free<ConsoleF<obj>, string> =
    Free(ReadLine(id), fun s -> Pure (s :?> string))

// Interpreter (interpreter pattern)
let rec interpret (program: Free<ConsoleF<obj>, 'A>) : 'A =
    match program with
    | Pure a -> a
    | Free(PrintLine(msg, next), k) ->
        printfn "%s" msg
        interpret (k next)
    | Free(ReadLine(f), k) ->
        let input = System.Console.ReadLine() |? "test input"
        interpret (k (f input))

// ตัวอย่างง่ายๆ ของ Free Monad
printfn "Free Monad concept demonstrated above"
```

## สรุป

```fsharp
(*
Computation Expressions - Methods ที่ต้องมี:

| Method          | ใช้สำหรับ         |
|-----------------|-------------------|
| Bind            | let!              |
| Return          | return            |
| ReturnFrom      | return!           |
| Zero            | unit return, if   |
| Combine         | หลาย statements   |
| Delay           | lazy eval         |
| Run             | รัน delayed       |
| For             | for...in          |
| While           | while             |
| TryWith         | try/with          |
| TryFinally      | try/finally       |
| Using           | use               |
| Yield           | yield             |
| YieldFrom       | yield!            |

Monads ที่พบบ่อย:
- Maybe/Option   - computation ที่อาจล้มเหลว
- Result         - computation ที่มี error
- State          - computation ที่มี state
- Writer         - computation ที่รวบรวม log
- Reader         - computation ที่ depends on config
- Async          - asynchronous computation
- List/Seq       - non-deterministic computation
*)

printfn "Computation Expressions ขั้นสูง - สรุปเสร็จ!"
```
