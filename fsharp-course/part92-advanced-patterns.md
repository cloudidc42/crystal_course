# Part 92 - Advanced Functional Patterns

## บทนำ

ในบทนี้เราจะเรียนรู้ patterns ขั้นสูงของ Functional Programming ซึ่งเป็นพื้นฐานของการเขียน F# code ที่ composable, testable และ maintainable

---

## 1. Functor - แนวคิดและ Laws

Functor คือ type class ที่ define วิธีการ map ค่าภายใน context โดยไม่เปลี่ยน structure ของ context

### Functor Laws
```
1. Identity:     map id = id
2. Composition:  map (f >> g) = map f >> map g
```

```fsharp
// Functor สำหรับ Option
module OptionFunctor =
    let map (f: 'a -> 'b) (opt: 'a option) : 'b option =
        match opt with
        | None -> None
        | Some x -> Some (f x)
    
    // ตรวจสอบ Identity Law
    let testIdentityLaw () =
        let x = Some 42
        let result = map id x  // ควรได้ Some 42
        assert (result = x)    // identity: map id = id
    
    // ตรวจสอบ Composition Law
    let testCompositionLaw () =
        let x = Some 5
        let f = (+) 1  // +1
        let g = (*) 2  // *2
        
        let result1 = map (f >> g) x          // map (f >> g) x
        let result2 = (map f >> map g) x      // map f >> map g
        
        assert (result1 = result2)

// Functor สำหรับ Result
module ResultFunctor =
    let map (f: 'a -> 'b) (result: Result<'a, 'e>) : Result<'b, 'e> =
        match result with
        | Error e -> Error e
        | Ok x -> Ok (f x)
    
    // map สำหรับ error path
    let mapError (f: 'e1 -> 'e2) (result: Result<'a, 'e1>) : Result<'a, 'e2> =
        match result with
        | Ok x -> Ok x
        | Error e -> Error (f e)
    
    // bimap - map ทั้ง Ok และ Error
    let bimap (f: 'a -> 'b) (g: 'e1 -> 'e2) (result: Result<'a, 'e1>) : Result<'b, 'e2> =
        match result with
        | Ok x -> Ok (f x)
        | Error e -> Error (g e)

// Functor สำหรับ List
module ListFunctor =
    // List.map คือ functor สำหรับ List
    let map = List.map
    
    // Verify identity law
    let verifyIdentity lst =
        map id lst = lst  // true

// Generic Functor interface
type Functor<'f> =
    abstract member Map: ('a -> 'b) -> 'f -> obj  // simplified

// สำหรับ F# ใช้ inline functions แทน
let inline fmap (f: 'a -> 'b) (x: ^F) = 
    (^F: (static member Map: ('a -> 'b) -> ^F -> obj) (f, x))
```

---

## 2. Applicative Functor - apply และ Laws

Applicative Functor ขยาย Functor โดยเพิ่มความสามารถในการ apply function ที่อยู่ใน context

### Applicative Laws
```
1. Identity:     pure id <*> v = v
2. Composition:  pure (.) <*> u <*> v <*> w = u <*> (v <*> w)
3. Homomorphism: pure f <*> pure x = pure (f x)
4. Interchange:  u <*> pure y = pure ($ y) <*> u
```

```fsharp
// Applicative สำหรับ Option
module OptionApplicative =
    // pure: ใส่ค่าใน context
    let pure' (x: 'a) : 'a option = Some x
    
    // apply: apply function ที่อยู่ใน option กับ value ใน option
    let apply (f: ('a -> 'b) option) (x: 'a option) : 'b option =
        match f, x with
        | Some f', Some x' -> Some (f' x')
        | _ -> None
    
    // Operator
    let (<*>) = apply
    
    // ตัวอย่างการใช้งาน: validate form data
    type ValidationError = 
        | EmptyName
        | InvalidAge
        | InvalidEmail
    
    type UserForm = {
        Name: string
        Age: int
        Email: string
    }
    
    let validateName name =
        if String.length name > 0 then Some name
        else None
    
    let validateAge age =
        if age >= 18 && age <= 120 then Some age
        else None
    
    let validateEmail email =
        if email.Contains("@") then Some email
        else None
    
    // ใช้ applicative style
    let createUser name age email =
        { Name = name; Age = age; Email = email }
    
    let validateForm name age email =
        pure' createUser
        <*> validateName name
        <*> validateAge age
        <*> validateEmail email
    
    // ตัวอย่าง
    let result1 = validateForm "Alice" 25 "alice@example.com"
    // Some { Name = "Alice"; Age = 25; Email = "alice@example.com" }
    
    let result2 = validateForm "" 25 "alice@example.com"
    // None (เพราะ name ว่าง)

// Applicative สำหรับ Result (สะสม errors)
module ResultApplicative =
    type Validation<'a, 'e> =
        | Success of 'a
        | Failure of 'e list
    
    let pure' (x: 'a) : Validation<'a, 'e> = Success x
    
    // apply ที่สะสม errors ทั้งหมด
    let apply (f: Validation<'a -> 'b, 'e>) (x: Validation<'a, 'e>) : Validation<'b, 'e> =
        match f, x with
        | Success f', Success x' -> Success (f' x')
        | Failure e1, Failure e2 -> Failure (e1 @ e2)
        | Failure e, _ | _, Failure e -> Failure e
    
    let (<*>) = apply
    let (<!>) f x = apply (pure' f) x
    
    // ตัวอย่าง: Validate ทุก field และ collect ทุก error
    let validateNameV name =
        if String.length name > 0 then Success name
        else Failure ["Name cannot be empty"]
    
    let validateAgeV age =
        if age >= 18 then Success age
        else Failure ["Must be at least 18 years old"]
    
    let validateEmailV email =
        if email.Contains("@") then Success email
        else Failure ["Invalid email format"]
    
    let validateUserForm name age email =
        (fun n a e -> {| Name = n; Age = a; Email = e |})
        <!> validateNameV name
        <*> validateAgeV age
        <*> validateEmailV email
    
    // ตัวอย่าง: invalid ทุก field
    let result = validateUserForm "" 15 "notanemail"
    // Failure ["Name cannot be empty"; "Must be at least 18"; "Invalid email format"]

// Applicative สำหรับ List (cartesian product)
module ListApplicative =
    let pure' x = [x]
    
    let apply (fs: ('a -> 'b) list) (xs: 'a list) : 'b list =
        [ for f in fs do
          for x in xs do
          yield f x ]
    
    // ตัวอย่าง
    let result = apply [(+) 1; (*) 2] [10; 20; 30]
    // [11; 21; 31; 20; 40; 60]
```

---

## 3. Monad - bind และ Laws

Monad ขยาย Applicative โดยเพิ่ม bind (>>= หรือ flatMap) ซึ่งช่วยให้ sequence computations ที่ return context ได้

### Monad Laws
```
1. Left Identity:  return a >>= f = f a
2. Right Identity: m >>= return = m
3. Associativity:  (m >>= f) >>= g = m >>= (fun x -> f x >>= g)
```

```fsharp
// Monad สำหรับ Option
module OptionMonad =
    let return' (x: 'a) = Some x
    
    let bind (f: 'a -> 'b option) (m: 'a option) : 'b option =
        match m with
        | None -> None
        | Some x -> f x
    
    let (>>=) m f = bind f m
    let (>=>) f g x = f x >>= g  // Kleisli composition
    
    // ตรวจสอบ Monad Laws
    let verifyLaws () =
        let a = 42
        let f x = if x > 0 then Some (x * 2) else None
        let g x = if x < 100 then Some (x + 1) else None
        let m = Some 5
        
        // Left Identity: return a >>= f = f a
        assert (return' a >>= f = f a)
        
        // Right Identity: m >>= return = m
        assert (m >>= return' = m)
        
        // Associativity: (m >>= f) >>= g = m >>= (fun x -> f x >>= g)
        assert ((m >>= f) >>= g = (m >>= fun x -> f x >>= g))
    
    // ตัวอย่างการใช้งาน: safe operations chain
    let safeDivide x y = 
        if y = 0 then None else Some (x / y)
    
    let safeSquareRoot x = 
        if x < 0.0 then None else Some (sqrt x)
    
    let safeLog x = 
        if x <= 0.0 then None else Some (log x)
    
    let computation x =
        safeDivide 100 x
        |> Option.bind (fun n -> safeSquareRoot (float n))
        |> Option.bind (fun n -> safeLog n)
    
    // หรือใช้ computation expression
    let computation' x = option {
        let! n = safeDivide 100 x
        let! sqrt = safeSquareRoot (float n)
        return! safeLog sqrt
    }

// Monad สำหรับ Result
module ResultMonad =
    let return' x = Ok x
    
    let bind (f: 'a -> Result<'b, 'e>) (m: Result<'a, 'e>) : Result<'b, 'e> =
        match m with
        | Error e -> Error e
        | Ok x -> f x
    
    let (>>=) m f = bind f m
    
    // Railway-oriented programming
    type Error = 
        | NotFound of string
        | ValidationError of string
        | DatabaseError of string
    
    let findUser userId = 
        if userId > 0 then Ok { Id = userId; Name = "User" }
        else Error (NotFound $"User {userId} not found")
    
    let validateUser user =
        if String.length user.Name > 0 then Ok user
        else Error (ValidationError "Name is required")
    
    let saveUser user =
        // simulate save
        Ok { user with Id = 999 }
    
    let updateUser userId =
        findUser userId
        >>= validateUser
        >>= saveUser

// Monad สำหรับ List
module ListMonad =
    let return' x = [x]
    
    let bind (f: 'a -> 'b list) (m: 'a list) : 'b list =
        m |> List.collect f
    
    // List monad = non-determinism
    let (>>=) m f = bind f m
    
    // ตัวอย่าง: generate pairs
    let pairs = 
        [1..3] >>= fun x ->
        [1..3] >>= fun y ->
        if x <> y then [(x, y)] else []
    // [(1,2); (1,3); (2,1); (2,3); (3,1); (3,2)]
    
    // หรือใช้ list comprehension
    let pairs' = [
        for x in [1..3] do
        for y in [1..3] do
        if x <> y then yield (x, y)
    ]
```

---

## 4. Monad Transformers

Monad Transformers ช่วยให้ stack หลาย monads เข้าด้วยกัน

```fsharp
// OptionT - Option เหนือ monad อื่น
type OptionT<'m, 'a> = OptionT of obj  // simplified representation

// ResultT over Async
module AsyncResult =
    type AsyncResult<'a, 'e> = Async<Result<'a, 'e>>
    
    let return' (x: 'a) : AsyncResult<'a, 'e> = 
        async { return Ok x }
    
    let bind (f: 'a -> AsyncResult<'b, 'e>) (m: AsyncResult<'a, 'e>) : AsyncResult<'b, 'e> =
        async {
            let! result = m
            match result with
            | Error e -> return Error e
            | Ok x -> return! f x
        }
    
    let (>>=) m f = bind f m
    
    let map (f: 'a -> 'b) (m: AsyncResult<'a, 'e>) : AsyncResult<'b, 'e> =
        async {
            let! result = m
            return Result.map f result
        }
    
    let mapError (f: 'e1 -> 'e2) (m: AsyncResult<'a, 'e1>) : AsyncResult<'a, 'e2> =
        async {
            let! result = m
            return Result.mapError f result
        }
    
    // Lift async into AsyncResult
    let ofAsync (a: Async<'a>) : AsyncResult<'a, 'e> =
        async {
            let! x = a
            return Ok x
        }
    
    // Lift Result into AsyncResult
    let ofResult (r: Result<'a, 'e>) : AsyncResult<'a, 'e> =
        async { return r }
    
    // Computation Expression
    type AsyncResultBuilder() =
        member _.Return(x) = return' x
        member _.ReturnFrom(m) = m
        member _.Bind(m, f) = bind f m
        member _.Zero() = return' ()
        member _.Delay(f) = f()
        member _.Combine(a, b) = bind (fun _ -> b) a
        
        member _.TryWith(m, handler) =
            async {
                try
                    return! m
                with e ->
                    return! handler e
            }
        
        member _.TryFinally(m, finalizer) =
            async {
                try
                    return! m
                finally
                    finalizer()
            }
    
    let asyncResult = AsyncResultBuilder()
    
    // ตัวอย่าง: chain async operations with error handling
    type AppError =
        | UserNotFound of int
        | DatabaseError of string
        | NetworkError of string
    
    let fetchUserFromDb userId : AsyncResult<{| Id: int; Name: string |}, AppError> =
        asyncResult {
            do! Async.Sleep 10 |> ofAsync
            if userId > 0 then
                return {| Id = userId; Name = "User" |}
            else
                return! Error (UserNotFound userId) |> ofResult
        }
    
    let fetchUserPosts userId : AsyncResult<string list, AppError> =
        asyncResult {
            do! Async.Sleep 5 |> ofAsync
            return ["Post 1"; "Post 2"]
        }
    
    let getUserProfile userId =
        asyncResult {
            let! user = fetchUserFromDb userId
            let! posts = fetchUserPosts user.Id
            return {|
                User = user
                Posts = posts
            |}
        }

// OptionT over Async
module AsyncOption =
    type AsyncOption<'a> = Async<'a option>
    
    let return' (x: 'a) : AsyncOption<'a> = 
        async { return Some x }
    
    let bind (f: 'a -> AsyncOption<'b>) (m: AsyncOption<'a>) : AsyncOption<'b> =
        async {
            let! opt = m
            match opt with
            | None -> return None
            | Some x -> return! f x
        }
    
    type AsyncOptionBuilder() =
        member _.Return(x) = return' x
        member _.ReturnFrom(m) = m
        member _.Bind(m, f) = bind f m
        member _.Zero() = return' ()
    
    let asyncOption = AsyncOptionBuilder()
```

---

## 5. Free Monad

Free Monad ช่วยให้แยก definition จาก interpretation ของ program

```fsharp
// Free Monad implementation
module FreeMonad =
    // Functor F
    type IOF<'a> =
        | ReadLine of (string -> 'a)
        | WriteLine of string * 'a
        | GetEnv of string * (string option -> 'a)
    
    // Map สำหรับ IOF
    let mapF (f: 'a -> 'b) (io: IOF<'a>) : IOF<'b> =
        match io with
        | ReadLine k -> ReadLine (k >> f)
        | WriteLine(s, next) -> WriteLine(s, f next)
        | GetEnv(key, k) -> GetEnv(key, k >> f)
    
    // Free Monad
    type Free<'f, 'a> =
        | Pure of 'a
        | Free of 'f  // simplified - normally Free of (Free<'f,'a> 'f -> ...)
    
    // ใช้ Continuation-Passing Style แทน
    type IO<'a> =
        | Pure of 'a
        | Bind of IOStep * ('a -> IO<'a>)
    
    and IOStep =
        | ReadLineStep
        | WriteLineStep of string
        | GetEnvStep of string
    
    // Smart constructors (DSL)
    let readLine () : Async<string> = async { return System.Console.ReadLine() }
    let writeLine s : Async<unit> = async { System.Console.WriteLine(s) }
    let getEnv key : Async<string option> = 
        async { return System.Environment.GetEnvironmentVariable(key) |> Option.ofObj }
    
    // Interpreter
    let rec interpret (program: Async<'a>) : Async<'a> = program
    
    // Production interpreter (real IO)
    let runIO = interpret
    
    // Test interpreter (pure/mock)
    let testIO (inputs: string list) =
        let mutable remaining = inputs
        let mockReadLine () = 
            async {
                match remaining with
                | [] -> return ""
                | h :: t ->
                    remaining <- t
                    return h
            }
        mockReadLine

// Practical Free Monad with DSL
module ProgramDSL =
    // Program algebra
    type ProgramInstruction<'a> =
        | Log of string * 'a
        | Ask of string * (string -> 'a)
        | HttpGet of string * (string -> 'a)
        | Fail of string
    
    type Program<'a> =
        | Done of 'a
        | Step of ProgramInstruction<Program<'a>>
    
    // Smart constructors
    let log msg = Step(Log(msg, Done()))
    let ask prompt = Step(Ask(prompt, Done))
    let httpGet url = Step(HttpGet(url, Done))
    let fail msg = Step(Fail msg)
    
    // Bind (flatMap)
    let rec bind (f: 'a -> Program<'b>) (program: Program<'a>) : Program<'b> =
        match program with
        | Done x -> f x
        | Step(Log(msg, next)) -> Step(Log(msg, bind f next))
        | Step(Ask(prompt, k)) -> Step(Ask(prompt, fun s -> bind f (k s)))
        | Step(HttpGet(url, k)) -> Step(HttpGet(url, fun s -> bind f (k s)))
        | Step(Fail msg) -> Step(Fail msg)
    
    let (>>=) program f = bind f program
    
    // Program computation expression
    type ProgramBuilder() =
        member _.Return(x) = Done x
        member _.ReturnFrom(m) = m
        member _.Bind(m, f) = bind f m
        member _.Zero() = Done ()
    
    let program = ProgramBuilder()
    
    // ตัวอย่าง program
    let greetUser = program {
        let! name = ask "What is your name? "
        do! log $"User said their name is: {name}"
        let! data = httpGet $"https://api.example.com/user/{name}"
        return $"Hello, {name}! Data: {data}"
    }
    
    // Real interpreter
    let rec runReal<'a> (prog: Program<'a>) : Async<'a> =
        async {
            match prog with
            | Done x -> return x
            | Step(Log(msg, next)) ->
                printfn "[LOG] %s" msg
                return! runReal next
            | Step(Ask(prompt, k)) ->
                printf "%s" prompt
                let input = System.Console.ReadLine()
                return! runReal (k input)
            | Step(HttpGet(url, k)) ->
                use client = new System.Net.Http.HttpClient()
                let! data = client.GetStringAsync(url) |> Async.AwaitTask
                return! runReal (k data)
            | Step(Fail msg) ->
                return failwith msg
        }
    
    // Test interpreter
    let rec runTest (inputs: string list) (prog: Program<'a>) : string list * 'a =
        match prog with
        | Done x -> [], x
        | Step(Log(msg, next)) ->
            let (logs, result) = runTest inputs next
            (msg :: logs, result)
        | Step(Ask(_, k)) ->
            match inputs with
            | [] -> failwith "No more inputs"
            | h :: t -> runTest t (k h)
        | Step(HttpGet(url, k)) ->
            runTest inputs (k $"Mock response for {url}")
        | Step(Fail msg) -> failwith msg
```

---

## 6. Tagless Final

Tagless Final เป็น alternative ของ Free Monad ที่ใช้ type classes แทน data types

```fsharp
// Tagless Final style
module TaglessFinal =
    // Define algebra as interface
    type Console<'f> =
        abstract ReadLine: unit -> 'f  // 'f = Async<string>
        abstract WriteLine: string -> 'f  // 'f = Async<unit>
    
    // ใช้ higher-kinded types ผ่าน SRTP
    // ปัญหา: F# ไม่ support HKT โดยตรง
    // แก้ไขโดยใช้ concrete type wrapper
    
    // Approach 1: Interface-based
    type ILogger =
        abstract Log: string -> Async<unit>
    
    type IUserRepository =
        abstract GetById: int -> Async<{| Id: int; Name: string |} option>
        abstract Save: {| Id: int; Name: string |} -> Async<unit>
    
    // Program ที่ไม่ผูกกับ implementation
    let getUserProfile (logger: ILogger) (repo: IUserRepository) userId =
        async {
            do! logger.Log $"Fetching user {userId}"
            let! user = repo.GetById userId
            match user with
            | None ->
                do! logger.Log $"User {userId} not found"
                return None
            | Some u ->
                do! logger.Log $"Found user: {u.Name}"
                return Some u
        }
    
    // Production implementation
    let consoleLogger : ILogger =
        { new ILogger with
            member _.Log msg = async { printfn "[INFO] %s" msg } }
    
    let dbUserRepo (connectionString: string) : IUserRepository =
        { new IUserRepository with
            member _.GetById id = 
                async {
                    // real DB query
                    return Some {| Id = id; Name = "User" |}
                }
            member _.Save user = 
                async {
                    // real DB save
                    return ()
                } }
    
    // Test implementation
    let testLogger () =
        let logs = System.Collections.Generic.List<string>()
        let logger = 
            { new ILogger with
                member _.Log msg = async { logs.Add(msg) } }
        logs, logger
    
    let testUserRepo users : IUserRepository =
        { new IUserRepository with
            member _.GetById id = 
                async { return Map.tryFind id users }
            member _.Save _ = async { return () } }
    
    // Approach 2: Reader Monad style
    type Env = {
        Logger: ILogger
        UserRepo: IUserRepository
    }
    
    type App<'a> = Env -> Async<'a>
    
    let log msg : App<unit> = fun env -> env.Logger.Log msg
    let getUser id : App<{| Id: int; Name: string |} option> = 
        fun env -> env.UserRepo.GetById id
    
    let runApp (env: Env) (app: App<'a>) = app env
    
    // Computation expression for App
    type AppBuilder() =
        member _.Return(x) : App<'a> = fun _ -> async { return x }
        member _.ReturnFrom(m: App<'a>) = m
        member _.Bind(m: App<'a>, f: 'a -> App<'b>) : App<'b> =
            fun env -> async {
                let! x = m env
                return! f x env
            }
    
    let app = AppBuilder()
    
    let getUserProfileApp userId : App<{| Id: int; Name: string |} option> =
        app {
            do! log $"Fetching user {userId}"
            let! user = getUser userId
            match user with
            | None ->
                do! log "User not found"
                return None
            | Some u ->
                return Some u
        }
```

---

## 7. Profunctor Optics

Profunctor Optics เป็นวิธีการเข้าถึงและแก้ไข nested data structures

```fsharp
// Profunctor Optics in F#
module Optics =
    // Lens: เข้าถึง field ใน record
    type Lens<'s, 'a> = {
        Get: 's -> 'a
        Set: 'a -> 's -> 's
    }
    
    // Smart constructor
    let lens get set = { Get = get; Set = set }
    
    // Compose lenses
    let composeLens (outer: Lens<'s, 'a>) (inner: Lens<'a, 'b>) : Lens<'s, 'b> =
        {
            Get = outer.Get >> inner.Get
            Set = fun b s -> outer.Set (inner.Set b (outer.Get s)) s
        }
    
    let (>>.) = composeLens
    
    // Modify function
    let modify (l: Lens<'s, 'a>) (f: 'a -> 'a) (s: 's) : 's =
        l.Set (f (l.Get s)) s
    
    // ตัวอย่าง
    type Address = { Street: string; City: string; Country: string }
    type Person = { Name: string; Age: int; Address: Address }
    type Company = { Name: string; Ceo: Person }
    
    // Define lenses
    let nameL = lens (fun p -> p.Name) (fun n p -> { p with Name = n })
    let ageL = lens (fun p -> p.Age) (fun a p -> { p with Age = a })
    let addressL = lens (fun p -> p.Address) (fun a p -> { p with Address = a })
    let cityL = lens (fun a -> a.City) (fun c a -> { a with City = c })
    let ceoL = lens (fun c -> c.Ceo) (fun ceo c -> { c with Ceo = ceo })
    
    // Compose
    let ceoNameL = ceoL >>. nameL
    let ceoCityL = ceoL >>. addressL >>. cityL
    
    // ใช้งาน
    let company = {
        Name = "Acme Corp"
        Ceo = {
            Name = "John Doe"
            Age = 45
            Address = { Street = "123 Main St"; City = "New York"; Country = "USA" }
        }
    }
    
    let ceoName = ceoNameL.Get company  // "John Doe"
    let updatedCompany = ceoNameL.Set "Jane Doe" company
    let movedCeo = ceoCityL.Set "Bangkok" company
    let olderCeo = modify (ceoL >>. ageL) ((+) 1) company

// Prism: เข้าถึง discriminated union cases
module Prisms =
    type Prism<'s, 'a> = {
        Preview: 's -> 'a option
        Review: 'a -> 's
    }
    
    let prism preview review = { Preview = preview; Review = review }
    
    // ตัวอย่าง: Prism สำหรับ Result
    let okPrism<'a, 'e> : Prism<Result<'a, 'e>, 'a> = 
        prism
            (function Ok x -> Some x | Error _ -> None)
            Ok
    
    let errorPrism<'a, 'e> : Prism<Result<'a, 'e>, 'e> =
        prism
            (function Error e -> Some e | Ok _ -> None)
            Error
    
    // Prism สำหรับ union types
    type Shape =
        | Circle of float
        | Rectangle of float * float
        | Triangle of float * float * float
    
    let circlePrism : Prism<Shape, float> =
        prism
            (function Circle r -> Some r | _ -> None)
            Circle
    
    let rectanglePrism : Prism<Shape, float * float> =
        prism
            (function Rectangle(w, h) -> Some (w, h) | _ -> None)
            (fun (w, h) -> Rectangle(w, h))
    
    // Modify through prism (only if matches)
    let modifyPrism (p: Prism<'s, 'a>) (f: 'a -> 'a) (s: 's) : 's =
        match p.Preview s with
        | None -> s
        | Some x -> p.Review (f x)
    
    // ตัวอย่างการใช้งาน
    let shapes = [Circle 5.0; Rectangle(3.0, 4.0); Circle 2.0; Triangle(3.0, 4.0, 5.0)]
    
    // Get all circles
    let circles = shapes |> List.choose circlePrism.Preview
    
    // Double all circle radii
    let doubledCircles = shapes |> List.map (modifyPrism circlePrism ((*) 2.0))

// Traversal: เข้าถึง elements ทั้งหมดใน structure
module Traversals =
    type Traversal<'s, 'a> = {
        ToList: 's -> 'a list
        SetAll: 'a list -> 's -> 's
    }
    
    // ตัวอย่าง: traverse list elements
    let listTraversal<'a>() : Traversal<'a list, 'a> = {
        ToList = id
        SetAll = fun xs _ -> xs
    }
    
    // Traverse result elements
    let resultTraversal<'a, 'e>() : Traversal<Result<'a, 'e>, 'a> = {
        ToList = function Ok x -> [x] | Error _ -> []
        SetAll = fun xs orig -> 
            match xs, orig with
            | [x], Ok _ -> Ok x
            | _, Error e -> Error e
            | _ -> orig
    }
```

---

## 8. MTL-style Type Classes ใน F#

```fsharp
// MTL (Monad Transformer Library) style
module MTL =
    // MonadReader
    type Reader<'r, 'a> = Reader of ('r -> 'a)
    
    let ask<'r> : Reader<'r, 'r> = Reader id
    let asks (f: 'r -> 'a) : Reader<'r, 'a> = Reader f
    let local (f: 'r -> 'r) (Reader m) : Reader<'r, 'a> = Reader (f >> m)
    let runReader r (Reader m) = m r
    
    type ReaderBuilder() =
        member _.Return(x) = Reader (fun _ -> x)
        member _.ReturnFrom(m) = m
        member _.Bind(Reader m, f) = 
            Reader (fun r -> let x = m r in let (Reader n) = f x in n r)
    
    let reader = ReaderBuilder()
    
    // MonadState
    type State<'s, 'a> = State of ('s -> 'a * 's)
    
    let get<'s> : State<'s, 's> = State (fun s -> s, s)
    let put (s: 's) : State<'s, unit> = State (fun _ -> (), s)
    let modify (f: 's -> 's) : State<'s, unit> = State (fun s -> (), f s)
    let runState s (State m) = m s
    let evalState s m = fst (runState s m)
    let execState s m = snd (runState s m)
    
    type StateBuilder() =
        member _.Return(x) = State (fun s -> x, s)
        member _.ReturnFrom(m) = m
        member _.Bind(State m, f) =
            State (fun s ->
                let x, s' = m s
                let (State n) = f x
                n s')
    
    let state = StateBuilder()
    
    // MonadWriter
    type Writer<'w, 'a> = Writer of 'a * 'w
    
    let tell (w: 'w) : Writer<'w list, unit> = Writer((), [w])
    let runWriter (Writer(a, w)) = a, w
    
    type WriterBuilder() =
        member _.Return(x) = Writer(x, [])
        member _.ReturnFrom(m) = m
        member _.Bind(Writer(a, w1), f) =
            let (Writer(b, w2)) = f a
            Writer(b, w1 @ w2)
    
    let writer = WriterBuilder()
    
    // ตัวอย่าง: การใช้ State monad
    let counter = state {
        let! count = get
        do! put (count + 1)
        let! newCount = get
        return newCount
    }
    
    let run3Times = state {
        let! a = counter
        let! b = counter
        let! c = counter
        return (a, b, c)
    }
    
    let result, finalState = runState 0 run3Times
    // result = (1, 2, 3), finalState = 3
    
    // ตัวอย่าง: การใช้ Writer monad สำหรับ audit log
    let validateAndLog name age =
        writer {
            do! tell $"Validating user: {name}"
            if String.length name = 0 then
                do! tell "ERROR: Name is empty"
                return Error "Name required"
            elif age < 18 then
                do! tell $"ERROR: Age {age} is under 18"
                return Error "Must be adult"
            else
                do! tell $"OK: User {name} age {age} validated"
                return Ok {| Name = name; Age = age |}
        }
    
    let result2, logs = runWriter (validateAndLog "Alice" 25)
    // result2 = Ok { Name="Alice"; Age=25 }
    // logs = ["Validating user: Alice"; "OK: User Alice age 25 validated"]
```

---

## 9. Practical Example: E-commerce Domain

```fsharp
// ประยุกต์ใช้ทุก pattern ในตัวอย่างจริง
module EcommerceDomain =
    open AsyncResult
    
    // Domain types
    type ProductId = ProductId of int
    type CustomerId = CustomerId of int
    type OrderId = OrderId of int
    
    type Product = {
        Id: ProductId
        Name: string
        Price: decimal
        Stock: int
    }
    
    type OrderLine = {
        Product: Product
        Quantity: int
    }
    
    type Order = {
        Id: OrderId
        Customer: CustomerId
        Lines: OrderLine list
        Total: decimal
    }
    
    type DomainError =
        | ProductNotFound of ProductId
        | InsufficientStock of ProductId * available: int * requested: int
        | CustomerNotFound of CustomerId
        | OrderNotFound of OrderId
        | InvalidQuantity of int
    
    // Algebras (Tagless Final style)
    type IProductService =
        abstract FindById: ProductId -> AsyncResult<Product, DomainError>
        abstract UpdateStock: ProductId -> int -> AsyncResult<unit, DomainError>
    
    type IOrderService =
        abstract Create: CustomerId -> OrderLine list -> AsyncResult<Order, DomainError>
        abstract FindById: OrderId -> AsyncResult<Order, DomainError>
    
    // Lenses สำหรับ domain
    let productStockL = 
        Optics.lens 
            (fun p -> p.Stock) 
            (fun s p -> { p with Stock = s })
    
    let orderTotalL =
        Optics.lens
            (fun o -> o.Total)
            (fun t o -> { o with Total = t })
    
    // Validation using Applicative
    let validateQuantity qty =
        if qty > 0 then ResultApplicative.Success qty
        else ResultApplicative.Failure ["Quantity must be positive"]
    
    let validateProductExists (product: Product option) =
        match product with
        | Some p -> ResultApplicative.Success p
        | None -> ResultApplicative.Failure ["Product not found"]
    
    // Business logic using Monad
    let processOrder 
        (productService: IProductService) 
        (customerId: CustomerId)
        (items: (ProductId * int) list) =
        asyncResult {
            // Validate all items
            let! orderLines =
                items
                |> List.map (fun (productId, qty) ->
                    asyncResult {
                        let! product = productService.FindById productId
                        if product.Stock < qty then
                            return! Error (InsufficientStock(productId, product.Stock, qty)) 
                                   |> AsyncResult.ofResult
                        else
                            return { Product = product; Quantity = qty }
                    })
                |> Async.Parallel
                |> AsyncResult.ofAsync
                |> AsyncResult.bind (fun results ->
                    results
                    |> Array.toList
                    |> List.map (fun r -> r)
                    |> fun xs -> 
                        xs |> List.fold (fun acc r ->
                            match acc, r with
                            | Ok lines, Ok line -> Ok (line :: lines)
                            | Error e, _ -> Error e
                            | _, Error e -> Error e) (Ok [])
                    |> AsyncResult.ofResult)
            
            let total = 
                orderLines 
                |> List.sumBy (fun l -> l.Product.Price * decimal l.Quantity)
            
            return {
                Id = OrderId 0  // assigned by DB
                Customer = customerId
                Lines = orderLines
                Total = total
            }
        }
```

---

## สรุป

Advanced Functional Patterns ที่เราได้เรียนรู้:

1. **Functor**: map ค่าใน context
2. **Applicative**: apply function ใน context, สะสม errors
3. **Monad**: chain computations, sequence effects
4. **Monad Transformers**: stack หลาย effects
5. **Free Monad**: แยก description จาก execution
6. **Tagless Final**: polymorphic programs ผ่าน type classes
7. **Profunctor Optics**: immutable data manipulation
8. **MTL Style**: composable computational effects

Pattern เหล่านี้ช่วยให้เขียน code ที่:
- **Composable**: ต่อกันได้อย่างง่ายดาย
- **Testable**: แยก logic จาก side effects
- **Type-safe**: compiler ตรวจสอบ correctness ให้
- **Readable**: แสดงเจตนาของ code ชัดเจน

---

*ต่อไป: Part 93 - Domain Specific Languages (DSL) กับ F#*
