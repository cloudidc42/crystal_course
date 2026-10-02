# Part 20 - นิพจน์การคำนวณ (Computation Expressions)

## บทนำ (Introduction)

Computation Expressions (CE) คือ F# mechanism ที่ช่วยให้เราเขียนโค้ดที่มี "effects" (เช่น Optional values, Error handling, Async) ในรูปแบบที่อ่านง่ายเหมือน imperative code

---

## 1. What Are Computation Expressions?

```fsharp
// Computation Expression คือ syntax sugar ที่แปลงโค้ดแบบ "เป็นขั้นตอน"
// ให้เป็นการเรียกฟังก์ชัน

// ตัวอย่างที่คุ้นเคย: seq
let numbers = seq {
    yield 1
    yield 2
    yield 3
}

// async
let asyncTask = async {
    let! result = Async.Sleep 1000
    return "done"
}

// Option (maybe)
// result (either)
// list
// state
// ...
```

```fsharp
// Desugaring: F# แปลง CE เป็น method calls

// สิ่งที่เขียน:
let result = seq {
    let x = 5
    yield x
    yield x + 1
}

// ถูกแปลงเป็น:
let result' = 
    SeqBuilder().Bind(5, fun x ->
        SeqBuilder().Yield(x) |> Seq.append (SeqBuilder().Yield(x + 1)))

// (simplified - actual desugaring ซับซ้อนกว่า)
```

---

## 2. Builder Class Pattern

```fsharp
// ทุก computation expression ต้องมี Builder class

// Builder class ต้องมี methods ที่เหมาะสม
// ขึ้นอยู่กับ operations ที่ต้องการ:
// - Bind: สำหรับ let!
// - Return: สำหรับ return
// - ReturnFrom: สำหรับ return!
// - Zero: สำหรับ empty CE
// - Yield/YieldFrom: สำหรับ yield/yield!
// - Combine: สำหรับ combining multiple expressions
// - Delay/Run: สำหรับ lazy evaluation

// ตัวอย่าง Builder ที่ง่ายที่สุด
type SimpleBuilder() =
    member _.Return(x) = x
    member _.Bind(x, f) = f x

let simple = SimpleBuilder()

let result = simple {
    let! x = 5
    let! y = 10
    return x + y
}

printfn "result = %d" result  // 15
```

---

## 3. let! (Bind)

```fsharp
// let! เรียก Bind method ของ builder

// สิ่งที่เขียน:
// let! x = someValue
// เทียบเท่ากับ:
// builder.Bind(someValue, fun x -> ...)

// ตัวอย่าง: Maybe builder ที่จัดการ None
type MaybeBuilder() =
    member _.Bind(x, f) =
        match x with
        | None -> None
        | Some v -> f v
    
    member _.Return(x) = Some x
    member _.ReturnFrom(x) = x

let maybe = MaybeBuilder()

// ใช้ let! เพื่อ unwrap Option values
let safeDivide x y =
    maybe {
        let! dividend = if x = 0 then None else Some x
        let! divisor = if y = 0 then None else Some y
        return dividend / divisor
    }

printfn "%A" (safeDivide 10 2)  // Some 5
printfn "%A" (safeDivide 0 2)   // None
printfn "%A" (safeDivide 10 0)  // None
```

```fsharp
// let! กับ Option - ตัวอย่างจริง

let tryGetUser (id: int) =
    if id = 1 then Some { Name = "Alice"; Age = 30 }
    elif id = 2 then Some { Name = "Bob"; Age = 25 }
    else None

and { Name = string; Age = int } = { Name = ""; Age = 0 }

type User = { Name: string; Age: int }

let tryGetUser' (id: int) =
    match id with
    | 1 -> Some { Name = "Alice"; Age = 30 }
    | 2 -> Some { Name = "Bob"; Age = 25 }
    | _ -> None

let getUserGreeting userId =
    maybe {
        let! user = tryGetUser' userId
        let greeting = sprintf "Hello, %s! You are %d years old." user.Name user.Age
        return greeting
    }

printfn "%A" (getUserGreeting 1)   // Some "Hello, Alice!..."
printfn "%A" (getUserGreeting 99)  // None
```

---

## 4. return (Return)

```fsharp
// return เรียก Return method

// Maybe builder (ต่อจากด้านบน)
type MaybeBuilder2() =
    member _.Bind(x, f) = Option.bind f x
    member _.Return(x) = Some x  // wrap value ใน context
    member _.ReturnFrom(x) = x   // ใช้ค่าโดยตรง (ไม่ wrap)
    member _.Zero() = None

let maybe2 = MaybeBuilder2()

// return: ส่งคืน wrapped value
let r1 = maybe2 { return 42 }           // Some 42
let r2 = maybe2 { return "hello" }      // Some "hello"

printfn "r1 = %A" r1  // Some 42
printfn "r2 = %A" r2  // Some "hello"
```

---

## 5. return! (ReturnFrom)

```fsharp
// return! เรียก ReturnFrom method
// ใช้เมื่อต้องการส่งคืน value ที่ already wrapped

let tryParse (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | _ -> None

let parseAndDouble s =
    maybe {
        let! n = tryParse s
        return n * 2  // ส่งคืน int, return จะ wrap เป็น Some
    }

let parseAndValidate s =
    maybe {
        let! n = tryParse s
        if n > 0 then
            return! Some n  // return! ใช้ Some n โดยตรง
        else
            return! None    // return! None = ส่งคืน None
    }

printfn "%A" (parseAndDouble "21")    // Some 42
printfn "%A" (parseAndDouble "abc")   // None
printfn "%A" (parseAndValidate "5")   // Some 5
printfn "%A" (parseAndValidate "-3")  // None
```

---

## 6. do! for Unit Operations

```fsharp
// do! สำหรับ operation ที่ return unit ใน context

// ตัวอย่าง: Result builder
type ResultBuilder() =
    member _.Bind(x, f) = Result.bind f x
    member _.Return(x) = Ok x
    member _.ReturnFrom(x) = x
    member _.Zero() = Ok ()

let result = ResultBuilder()

let log message : Result<unit, string> =
    printfn "[LOG] %s" message
    Ok ()

let validateAge age : Result<int, string> =
    if age >= 0 && age <= 150 then Ok age
    else Error (sprintf "Invalid age: %d" age)

let processAge age =
    result {
        do! log (sprintf "Processing age: %d" age)  // do! สำหรับ unit operation
        let! validAge = validateAge age
        do! log (sprintf "Valid age: %d" validAge)
        return validAge * 2  // arbitrary processing
    }

printfn "%A" (processAge 25)   // logs + Ok 50
printfn "%A" (processAge 200)  // logs + Error "Invalid age: 200"
```

---

## 7. yield and yield!

```fsharp
// yield ใช้ใน sequence-like builders
// yield! แทรก iterable ทั้งหมด

// Custom list builder
type ListBuilder() =
    member _.Yield(x) = [x]
    member _.YieldFrom(xs) = xs
    member _.Combine(a, b) = a @ b
    member _.Delay(f) = f ()
    member _.Zero() = []
    member _.For(xs, f) = xs |> List.collect f
    member _.Return(x) = [x]

let myList = ListBuilder()

let numbers = myList {
    yield 1
    yield 2
    yield 3
}
printfn "numbers: %A" numbers  // [1; 2; 3]

let mixed = myList {
    yield 0
    yield! [1; 2; 3]
    yield 4
    yield! [5; 6]
}
printfn "mixed: %A" mixed  // [0; 1; 2; 3; 4; 5; 6]
```

---

## 8. for loops in CE

```fsharp
// for loops ใน computation expressions

// seq กับ for loop
let evenSquares = seq {
    for i in 1..10 do
        if i % 2 = 0 then
            yield i * i
}

printfn "even squares: %A" (Seq.toList evenSquares)
// [4; 16; 36; 64; 100]
```

```fsharp
// Custom builder ที่ support for loop
type CollectBuilder() =
    member _.Yield(x) = [x]
    member _.YieldFrom(xs) = List.ofSeq xs
    member _.Combine(a, b) = a @ b
    member _.Delay(f) = f ()
    member _.Zero() = []
    member _.For(xs, f) = xs |> List.collect f

let collect = CollectBuilder()

let results = collect {
    for i in 1..5 do
        for j in 1..3 do
            yield i * j
}

printfn "results: %A" results
// [1; 2; 3; 2; 4; 6; 3; 6; 9; 4; 8; 12; 5; 10; 15]
```

---

## 9. while loops in CE

```fsharp
// while loops ใน computation expressions

// seq กับ while loop
let countdown = seq {
    let mutable n = 10
    while n > 0 do
        yield n
        n <- n - 1
}

printfn "countdown: %A" (countdown |> Seq.toList)
```

```fsharp
// State builder ที่ support while
type StateBuilder<'s>() =
    member _.Bind((state: 's), f: 's -> 'a * 's) = f state
    member _.Return(x) = fun state -> (x, state)
    member _.ReturnFrom(x) = x
    member _.Zero() = fun state -> ((), state)

// สร้าง stateful computations ง่ายๆ
type State<'s, 'a> = 's -> ('a * 's)

let getState : State<int, int> = fun s -> (s, s)
let putState s : State<int, unit> = fun _ -> ((), s)
let modifyState f : State<int, unit> = fun s -> ((), f s)
let runState (m: State<'s, 'a>) initialState = m initialState
```

---

## 10. if/then in CE

```fsharp
// if/then ใน computation expressions ใช้ได้ตามปกติ

// maybe builder
let maybe = MaybeBuilder()

let checkAndProcess value =
    maybe {
        let! v = value
        if v > 0 then
            return v * 2
        else
            return! None  // ถ้า <= 0 ส่งคืน None
    }

printfn "%A" (checkAndProcess (Some 5))   // Some 10
printfn "%A" (checkAndProcess (Some -3))  // None
printfn "%A" (checkAndProcess None)        // None
```

```fsharp
// if/then ใน sequence CE
let classifiedNumbers = seq {
    for i in 1..10 do
        if i % 3 = 0 && i % 5 = 0 then
            yield (i, "FizzBuzz")
        elif i % 3 = 0 then
            yield (i, "Fizz")
        elif i % 5 = 0 then
            yield (i, "Buzz")
        else
            yield (i, string i)
}

classifiedNumbers |> Seq.iter (fun (n, s) -> printfn "%d: %s" n s)
```

---

## 11. Custom Builder: maybe { }

```fsharp
// สร้าง Maybe builder ที่สมบูรณ์

type MaybeBuilder() =
    // let! x = expr
    member _.Bind(x: 'a option, f: 'a -> 'b option) : 'b option =
        Option.bind f x
    
    // return value
    member _.Return(x: 'a) : 'a option = Some x
    
    // return! optionValue
    member _.ReturnFrom(x: 'a option) : 'a option = x
    
    // empty CE block
    member _.Zero() : unit option = Some ()
    
    // combine multiple expressions
    member _.Combine(x: unit option, y: unit option) : unit option =
        Option.bind (fun () -> y) x
    
    // delay evaluation
    member _.Delay(f: unit -> 'a option) : unit -> 'a option = f
    
    // run the computation
    member _.Run(f: unit -> 'a option) : 'a option = f ()
    
    // try/finally
    member _.TryFinally(f: unit -> 'a option, cleanup: unit -> unit) =
        try f () finally cleanup ()
    
    // use statement
    member _.Using(resource: #System.IDisposable, f: #System.IDisposable -> 'a option) =
        try f resource finally resource.Dispose()

let maybe = MaybeBuilder()
```

```fsharp
// ตัวอย่างการใช้ Maybe builder

type User = { Id: int; Name: string; DeptId: int }
type Dept = { Id: int; Name: string; ManagerId: int }

let users = Map.ofList [(1, { Id = 1; Name = "Alice"; DeptId = 10 }); (2, { Id = 2; Name = "Bob"; DeptId = 20 })]
let depts = Map.ofList [(10, { Id = 10; Name = "Engineering"; ManagerId = 1 })]

let findUser id = Map.tryFind id users
let findDept id = Map.tryFind id depts

let getUserDeptName userId =
    maybe {
        let! user = findUser userId
        let! dept = findDept user.DeptId
        return dept.Name
    }

printfn "User 1's dept: %A" (getUserDeptName 1)  // Some "Engineering"
printfn "User 2's dept: %A" (getUserDeptName 2)  // None (DeptId 20 not found)
printfn "User 99's dept: %A" (getUserDeptName 99) // None (user not found)
```

---

## 12. Custom Builder: result { }

```fsharp
// Result builder สำหรับ error handling

type ResultBuilder() =
    member _.Bind(x: Result<'a, 'e>, f: 'a -> Result<'b, 'e>) : Result<'b, 'e> =
        Result.bind f x
    
    member _.Return(x: 'a) : Result<'a, 'e> = Ok x
    
    member _.ReturnFrom(x: Result<'a, 'e>) : Result<'a, 'e> = x
    
    member _.Zero() : Result<unit, 'e> = Ok ()
    
    member _.Combine(x: Result<unit, 'e>, y: Result<unit, 'e>) =
        Result.bind (fun () -> y) x
    
    member _.Delay(f) = f
    member _.Run(f) = f ()

let result = ResultBuilder()
```

```fsharp
// ใช้ result builder สำหรับ validation pipeline

type ValidationError =
    | EmptyName
    | InvalidAge of string
    | InvalidEmail
    | WeakPassword

let validateName (name: string) =
    if System.String.IsNullOrWhiteSpace(name) then Error EmptyName
    else Ok (name.Trim())

let validateAge (ageStr: string) =
    match System.Int32.TryParse(ageStr) with
    | true, age when age >= 18 && age <= 100 -> Ok age
    | true, age -> Error (InvalidAge (sprintf "Age %d is out of range (18-100)" age))
    | false, _ -> Error (InvalidAge (sprintf "'%s' is not a valid age" ageStr))

let validateEmail (email: string) =
    if email.Contains("@") && email.Contains(".") then Ok email
    else Error InvalidEmail

let validatePassword (password: string) =
    if password.Length >= 8 
       && password |> Seq.exists System.Char.IsUpper
       && password |> Seq.exists System.Char.IsDigit
    then Ok password
    else Error WeakPassword

type UserRegistration = {
    Name: string
    Age: string
    Email: string
    Password: string
}

type ValidUser = {
    Name: string
    Age: int
    Email: string
}

let registerUser (input: UserRegistration) : Result<ValidUser, ValidationError> =
    result {
        let! name = validateName input.Name
        let! age = validateAge input.Age
        let! email = validateEmail input.Email
        let! _ = validatePassword input.Password
        return { Name = name; Age = age; Email = email }
    }

// ทดสอบ
let validInput = { Name = "Alice"; Age = "25"; Email = "alice@example.com"; Password = "Secure1!" }
let emptyName = { validInput with Name = "" }
let badAge = { validInput with Age = "10" }
let badEmail = { validInput with Email = "notanemail" }

[validInput; emptyName; badAge; badEmail]
|> List.iter (fun input ->
    match registerUser input with
    | Ok user -> printfn "Registered: %s (age %d)" user.Name user.Age
    | Error err -> printfn "Error: %A" err)
```

---

## 13. Custom Builder: list { }

```fsharp
// List builder (Monad สำหรับ non-determinism)

type ListBuilder() =
    member _.Bind(xs: 'a list, f: 'a -> 'b list) : 'b list =
        List.collect f xs
    
    member _.Return(x: 'a) : 'a list = [x]
    
    member _.ReturnFrom(xs: 'a list) : 'a list = xs
    
    member _.Zero() : 'a list = []
    
    member _.Yield(x) = [x]
    
    member _.YieldFrom(xs) = xs
    
    member _.Combine(a, b) = a @ b
    
    member _.Delay(f) = f ()
    
    member _.For(xs, f) = List.collect f xs

let listBuilder = ListBuilder()

// List monad: สร้าง combinations ทั้งหมด
let combinations =
    listBuilder {
        let! x = [1; 2; 3]
        let! y = ["a"; "b"]
        return sprintf "%d%s" x y
    }

printfn "combinations: %A" combinations
// ["1a"; "1b"; "2a"; "2b"; "3a"; "3b"]
```

```fsharp
// List builder สำหรับ non-deterministic computation

// หา pythagorean triples
let pythagorean n =
    listBuilder {
        let! a = [1..n]
        let! b = [a..n]
        let! c = [b..n]
        if a*a + b*b = c*c then
            return (a, b, c)
    }

printfn "Pythagorean triples up to 20:"
pythagorean 20 |> List.iter (printfn "  %A")
```

---

## 14. seq { } Revisited

```fsharp
// seq { } เป็น built-in CE ที่สมบูรณ์มาก

// Comprehension style
let evenSquares = seq {
    for i in 1..100 do
        if i % 2 = 0 then
            yield i * i
}

// Generator style
let fibonacci = seq {
    let mutable (a, b) = 0, 1
    while true do
        yield a
        let next = a + b
        a <- b
        b <- next
}

// Nested
let matrix n = seq {
    for i in 1..n do
        for j in 1..n do
            yield (i, j)
}

// yield! สำหรับ flatten
let flatten xss = seq {
    for xs in xss do
        yield! xs
}

printfn "First 10 even squares: %A" (evenSquares |> Seq.take 10 |> Seq.toList)
printfn "First 10 fibonacci: %A" (fibonacci |> Seq.take 10 |> Seq.toList)
printfn "2x2 matrix: %A" (matrix 2 |> Seq.toList)
printfn "flatten [[1;2];[3;4]]: %A" (flatten [[1;2];[3;4]] |> Seq.toList)
```

---

## 15. async { } Introduction

```fsharp
// async { } เป็น built-in CE สำหรับ asynchronous programming

// async computation
let asyncGreet name = async {
    do! Async.Sleep 100  // รอ 100ms (ไม่ block thread)
    return sprintf "Hello, %s!" name
}

// รัน async computation
let result = asyncGreet "Alice" |> Async.RunSynchronously
printfn "%s" result
```

```fsharp
// async ด้วย multiple operations
let fetchData id = async {
    do! Async.Sleep 50  // simulate network call
    return sprintf "Data_%d" id
}

let processData data = async {
    do! Async.Sleep 20  // simulate processing
    return data.ToUpper()
}

let getAndProcess id = async {
    let! data = fetchData id
    let! processed = processData data
    return processed
}

let result = getAndProcess 42 |> Async.RunSynchronously
printfn "%s" result  // "DATA_42"
```

```fsharp
// Parallel async operations
let tasks = [1..5] |> List.map (fun id -> async {
    do! Async.Sleep (id * 10)
    return sprintf "Task_%d" id
})

let results = tasks |> Async.Parallel |> Async.RunSynchronously
printfn "Results: %A" results
```

---

## 16. Understanding Desugaring

```fsharp
// Desugaring: F# แปลง CE syntax เป็น method calls

// สิ่งที่เขียน (syntax sugar):
let maybeResult = maybe {
    let! x = Some 5
    let! y = Some 10
    return x + y
}

// ถูก desugar เป็น:
let maybeResult' = 
    maybe.Bind(Some 5, fun x ->
        maybe.Bind(Some 10, fun y ->
            maybe.Return(x + y)))

printfn "maybeResult: %A" maybeResult   // Some 15
printfn "maybeResult': %A" maybeResult' // Some 15
```

```fsharp
// Desugar ของ do!
let doExample = maybe {
    do! Some ()
    return 42
}

// เทียบเท่ากับ:
let doExample' = 
    maybe.Bind(Some (), fun () ->
        maybe.Return(42))

printfn "doExample: %A" doExample   // Some 42
printfn "doExample': %A" doExample' // Some 42
```

```fsharp
// Desugar ของ for
let forExample = seq {
    for i in 1..3 do
        yield i * 2
}

// เทียบเท่ากับ (simplified):
let forExample' = 
    seq {
        yield! Seq.map (fun i -> i * 2) (seq {1..3})
    }

printfn "forExample: %A" (Seq.toList forExample)
printfn "forExample': %A" (Seq.toList forExample')
```

---

## 17. Comprehensive Example

```fsharp
// ตัวอย่างรวม: ระบบ user authentication

// Domain types
type AuthError =
    | UserNotFound of string
    | InvalidPassword
    | AccountLocked
    | TokenExpired

type User = {
    Username: string
    PasswordHash: string
    Locked: bool
    Permissions: string list
}

// Database (mock)
let users = Map.ofList [
    ("alice", { Username = "alice"; PasswordHash = "abc123_hashed"; Locked = false; Permissions = ["read"; "write"] })
    ("admin", { Username = "admin"; PasswordHash = "admin_hashed"; Locked = false; Permissions = ["read"; "write"; "admin"] })
    ("locked", { Username = "locked"; PasswordHash = "locked_hashed"; Locked = true; Permissions = [] })
]

// Auth functions
let findUser username : Result<User, AuthError> =
    match Map.tryFind username users with
    | Some user -> Ok user
    | None -> Error (UserNotFound username)

let checkLocked (user: User) : Result<User, AuthError> =
    if user.Locked then Error AccountLocked
    else Ok user

let verifyPassword (password: string) (user: User) : Result<User, AuthError> =
    let hash = password + "_hashed"  // mock hashing
    if hash = user.PasswordHash then Ok user
    else Error InvalidPassword

let checkPermission (permission: string) (user: User) : Result<User, AuthError> =
    if List.contains permission user.Permissions then Ok user
    else Error (UserNotFound (sprintf "Permission '%s' not found" permission))

// Authentication pipeline ด้วย result builder
let authenticate username password permission =
    result {
        let! user = findUser username
        let! unlockedUser = checkLocked user
        let! authenticatedUser = verifyPassword password unlockedUser
        let! authorizedUser = checkPermission permission authenticatedUser
        return sprintf "Welcome, %s! You have '%s' access." authorizedUser.Username permission
    }

// ทดสอบ
let testCases = [
    ("alice", "abc123", "read")
    ("alice", "wrongpwd", "read")
    ("alice", "abc123", "admin")
    ("nobody", "pass", "read")
    ("locked", "locked", "read")
    ("admin", "admin", "admin")
]

testCases |> List.iter (fun (user, pwd, perm) ->
    let r = authenticate user pwd perm
    printfn "%s/%s/%s -> %A" user pwd perm r)
```

---

## สรุป (Summary)

```fsharp
// สรุป Computation Expressions

// 1. Builder class
type MyBuilder() =
    member _.Bind(x, f) = f x
    member _.Return(x) = x
    member _.ReturnFrom(x) = x
    member _.Zero() = ()
    member _.Delay(f) = f ()

let myBuilder = MyBuilder()

// 2. Basic usage
let result = myBuilder {
    let! x = 5       // Bind
    let! y = 10      // Bind
    return x + y     // Return
}

// 3. let! = Bind
// return = Return  
// return! = ReturnFrom
// do! = Bind with unit
// yield = Yield (for sequence-like)
// yield! = YieldFrom

// 4. Real builders
let maybeResult = maybe { return 42 }          // Option
let resultValue = result { return "hello" }    // Result
let seqValue = seq { yield 1; yield 2 }        // Sequence
// let asyncValue = async { return "async" }   // Async

printfn "result = %d" result
printfn "maybe = %A" maybeResult
```

Computation Expressions ทำให้:
- **Maybe/Option**: เขียน null-safe code โดยไม่ต้อง match ทุกขั้น
- **Result/Either**: error handling แบบ typed ที่อ่านง่าย
- **Async**: asynchronous code ที่เหมือน synchronous
- **State**: stateful computations แบบ functional
- **List/Seq**: non-deterministic computations

สิ่งสำคัญที่ต้องจำ:
1. Builder class เป็น core ของทุก CE
2. `let!` = Bind (extract value from context)
3. `return` = Return (put value in context)
4. `return!` = ReturnFrom (return existing context)
5. `do!` = Bind สำหรับ unit operations
6. CE ถูก desugar เป็น method calls
