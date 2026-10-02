# Part 40 - การจัดการข้อยกเว้น (Exception Handling)

## บทนำ (Introduction)

Exception handling ใน F# ให้เครื่องมือสำหรับจัดการ errors ในขณะ runtime ทั้งผ่าน traditional try/with pattern และผ่าน functional approach ด้วย Result type F# ส่งเสริมให้ใช้ Result สำหรับ expected errors และ exceptions สำหรับ truly exceptional situations

---

## 1. try/with in F#

```fsharp
// try/with syntax

// Basic try/with
let safeDivide a b =
    try
        a / b
    with
    | :? System.DivideByZeroException ->
        printfn "Division by zero!"
        0

printfn "10/2 = %d" (safeDivide 10 2)     // 5
printfn "10/0 = %d" (safeDivide 10 0)     // 0 (after error message)

// Multiple exception handlers
let parseAndCompute (input: string) =
    try
        let n = System.Int32.Parse(input)
        100 / n
    with
    | :? System.FormatException ->
        printfn "Input '%s' is not a number" input
        -1
    | :? System.DivideByZeroException ->
        printfn "Cannot divide by zero"
        -2
    | ex ->
        printfn "Unexpected error: %s" ex.Message
        -99

printfn "\nparseAndCompute results:"
printfn "  '5' -> %d" (parseAndCompute "5")
printfn "  'abc' -> %d" (parseAndCompute "abc")
printfn "  '0' -> %d" (parseAndCompute "0")

// try/with เป็น expression (return value)
let result =
    try
        let x = int "42"
        sprintf "Parsed: %d" x
    with
    | ex -> sprintf "Error: %s" ex.Message

printfn "\nResult: %s" result
```

---

## 2. try/finally

```fsharp
// try/finally - รัน cleanup code เสมอไม่ว่าจะเกิด exception หรือไม่

// Resource cleanup pattern
let readFile (filename: string) =
    let reader = new System.IO.StreamReader(filename)
    try
        let content = reader.ReadToEnd()
        content
    finally
        reader.Dispose()
        printfn "File reader closed"

// Database connection pattern
type SimulatedDB() =
    let mutable isOpen = false
    
    member this.Open() =
        isOpen <- true
        printfn "DB connection opened"
    
    member this.Close() =
        isOpen <- false
        printfn "DB connection closed"
    
    member this.Query(sql: string) =
        if not isOpen then failwith "DB not open"
        printfn "Executing: %s" sql
        sprintf "Results of: %s" sql

let executeWithDB (connectionString: string) (query: string) =
    let db = SimulatedDB()
    db.Open()
    try
        let result = db.Query(query)
        printfn "Query result: %s" result
        result
    finally
        db.Close()

let result = executeWithDB "Server=localhost" "SELECT * FROM users"
printfn "Got: %s\n" result

// Nested try/with/finally
let complexOperation () =
    printfn "Starting operation..."
    try
        try
            printfn "  Attempting risky work..."
            failwith "Something went wrong"
        with
        | ex ->
            printfn "  Inner handler caught: %s" ex.Message
            printfn "  Attempting recovery..."
            "recovered"
    finally
        printfn "  Cleanup always runs"

let r = complexOperation()
printfn "Operation result: %s" r

// try/with + try/finally combined
let safeOperation (input: string) =
    let resources = System.Collections.Generic.List<string>()
    try
        resources.Add("Resource1")
        let n = System.Int32.Parse(input)
        resources.Add("Resource2")
        sprintf "Result: %d" (100 / n)
    with
    | :? System.FormatException -> sprintf "Parse error for: %s" input
    | :? System.DivideByZeroException -> "Division by zero"
    |> fun result ->
        resources |> Seq.iter (fun r -> printfn "Releasing: %s" r)
        result

printfn "\nSafe operation:"
printfn "  '5': %s" (safeOperation "5")
printfn "  '0': %s" (safeOperation "0")
printfn "  'x': %s" (safeOperation "x")
```

---

## 3. raise and reraise

```fsharp
// raise - throw an exception
// reraise - re-throw the current exception preserving stack trace

// raise
let validateAge (age: int) =
    if age < 0 then
        raise (System.ArgumentException("Age cannot be negative", nameof age))
    if age > 150 then
        raise (System.ArgumentOutOfRangeException(nameof age, "Age seems unrealistic"))
    age

try
    validateAge -5 |> ignore
with
| :? System.ArgumentException as ex ->
    printfn "ArgumentException: %s" ex.Message

// reraise - preserves original stack trace
let processWithLogging (operation: unit -> 'T) =
    try
        operation()
    with
    | ex ->
        printfn "LOGGED: %s at %s" ex.Message ex.StackTrace.[..100]
        reraise()  // Re-throw preserving stack trace

// Contrast with: raise ex (creates new exception, loses stack trace)
let processWithNewRaise (operation: unit -> 'T) =
    try
        operation()
    with
    | ex ->
        printfn "LOGGED: %s" ex.Message
        raise ex  // BAD: loses original stack trace!

try
    processWithLogging (fun () ->
        failwith "Original error"
    )
with
| ex -> printfn "Caught: %s" ex.Message

// Custom re-throw patterns
let tryExecute (action: unit -> 'T) =
    try
        Ok (action())
    with
    | :? System.TimeoutException as ex ->
        Error (sprintf "Timeout: %s" ex.Message)
    | ex ->
        printfn "Unexpected exception - re-raising: %s" ex.GetType().Name
        reraise()

let result1 = tryExecute (fun () -> 42)
let result2 = tryExecute (fun () -> raise (System.TimeoutException("Connection timed out")))

printfn "\ntryExecute results:"
printfn "  Success: %A" result1
printfn "  Timeout: %A" result2
```

---

## 4. failwith, failwithf

```fsharp
// failwith และ failwithf - สร้าง System.Exception ด้วย message

// failwith - basic failure
let divide a b =
    if b = 0 then failwith "Cannot divide by zero"
    else a / b

// failwithf - formatted message
let validateInput (name: string) (value: int) (min: int) (max: int) =
    if value < min || value > max then
        failwithf "Parameter '%s' must be between %d and %d, but got %d" name min max value

// ใช้ failwith ใน domain validation
type Age = private Age of int

module Age =
    let create value =
        if value < 0 then failwithf "Age cannot be negative: %d" value
        elif value > 150 then failwithf "Age too large: %d" value
        else Age value
    
    let value (Age v) = v

type Name = private Name of string

module Name =
    let create (s: string) =
        if System.String.IsNullOrWhiteSpace(s) then 
            failwith "Name cannot be empty"
        elif s.Length > 100 then 
            failwithf "Name too long (max 100 chars): %d chars given" s.Length
        else Name s
    
    let value (Name v) = v

// ทดสอบ
try
    let age = Age.create 30
    printfn "Age: %d" (Age.value age)
    
    let badAge = Age.create -5
    ()
with
| :? System.Exception as ex ->
    printfn "Error: %s" ex.Message

// failwith ใน exhaustive matching
type Shape3 = Circle3 of float | Square3 of float | Triangle3 of float

let area3 shape =
    match shape with
    | Circle3 r -> System.Math.PI * r * r
    | Square3 s -> s * s
    | Triangle3 b -> failwithf "Triangle area requires height - only base %.2f given" b

try
    let a = area3 (Triangle3 5.0)
    ()
with
| ex -> printfn "Error: %s" ex.Message
```

---

## 5. invalidArg, invalidOp, nullArg

```fsharp
// Helper functions สำหรับสร้าง specific exception types

// invalidArg - ArgumentException
let clampValue (paramName: string) (value: int) (min: int) (max: int) =
    if value < min || value > max then
        invalidArg paramName (sprintf "Must be between %d and %d" min max)
    value

// invalidOp - InvalidOperationException
type Counter4(initialValue: int) =
    let mutable count = initialValue
    let mutable locked = false
    
    member this.Increment() =
        if locked then
            invalidOp "Cannot increment a locked counter"
        count <- count + 1
    
    member this.Lock() = locked <- true
    member this.Value = count

// nullArg - ArgumentNullException
let processString (paramName: string) (s: string) =
    if s = null then nullArg paramName
    s.Trim().ToUpper()

// ทดสอบ
try
    let v = clampValue "temperature" 105 0 100
    ()
with
| :? System.ArgumentException as ex ->
    printfn "ArgumentException: Param=%s, Msg=%s" ex.ParamName ex.Message

try
    let c = Counter4(0)
    c.Increment()  // OK
    c.Lock()
    c.Increment()  // Error!
with
| :? System.InvalidOperationException as ex ->
    printfn "InvalidOperationException: %s" ex.Message

try
    processString "text" null |> ignore
with
| :? System.ArgumentNullException as ex ->
    printfn "ArgumentNullException: %s" ex.ParamName

// Practical validation
let validateOrder (orderId: int) (amount: decimal) (customerId: string) =
    if orderId <= 0 then invalidArg "orderId" "Order ID must be positive"
    if amount <= 0M then invalidArg "amount" "Amount must be positive"
    if customerId = null then nullArg "customerId"
    if System.String.IsNullOrWhiteSpace(customerId) then
        invalidArg "customerId" "Customer ID cannot be empty"
    
    {| OrderId = orderId; Amount = amount; CustomerId = customerId |}

let validOrder = validateOrder 1001 500.0M "CUST001"
printfn "\nValid order: #%d, %.2M, %s" validOrder.OrderId validOrder.Amount validOrder.CustomerId

try
    let badOrder = validateOrder 0 500.0M "CUST001"
    ()
with
| :? System.ArgumentException as ex ->
    printfn "Error: %s (%s)" ex.Message ex.ParamName
```

---

## 6. Custom Exception Types

```fsharp
// Custom exception hierarchy

// Base custom exception
type AppException(message: string, ?innerEx: exn) =
    inherit System.Exception(message, defaultArg innerEx null)
    
    member this.ErrorCode = "APP_ERROR"
    member this.Timestamp = System.DateTime.UtcNow

// Domain-specific exceptions
type ValidationException(message: string, field: string, ?value: obj) =
    inherit AppException(message)
    
    member this.Field = field
    member this.Value = value
    
    override this.Message =
        match value with
        | Some v -> sprintf "Validation failed on field '%s': %s (value: %A)" field message v
        | None -> sprintf "Validation failed on field '%s': %s" field message

type DatabaseException(message: string, query: string, ?innerEx: exn) =
    inherit AppException(message, ?innerEx = innerEx)
    
    member this.Query = query
    
    override this.Message =
        sprintf "Database error: %s\nQuery: %s" message query

type AuthenticationException(message: string, userId: string option) =
    inherit AppException(message)
    
    member this.UserId = userId
    
    override this.Message =
        match userId with
        | Some uid -> sprintf "Authentication failed for user '%s': %s" uid message
        | None -> sprintf "Authentication failed: %s" message

type NetworkException(message: string, url: string, statusCode: int) =
    inherit AppException(message)
    
    member this.Url = url
    member this.StatusCode = statusCode

// Exception handling with custom types
let handleRequest (userId: string option) (query: string) =
    match userId with
    | None ->
        raise (AuthenticationException("No user ID provided", None))
    | Some uid when uid = "banned" ->
        raise (AuthenticationException("User is banned", Some uid))
    | Some _ ->
        if System.String.IsNullOrEmpty(query) then
            raise (ValidationException("Query cannot be empty", "query"))
        
        sprintf "Result for query: %s" query

let testRequests = [
    None, "SELECT * FROM users"
    Some "banned", "SELECT * FROM orders"
    Some "user1", ""
    Some "user1", "SELECT * FROM products"
]

for (userId, query) in testRequests do
    try
        let result = handleRequest userId query
        printfn "Success: %s" result
    with
    | :? AuthenticationException as ex ->
        printfn "Auth Error: %s (User: %A)" ex.Message ex.UserId
    | :? ValidationException as ex ->
        printfn "Validation Error: %s" ex.Message
    | :? AppException as ex ->
        printfn "App Error: %s" ex.Message
```

---

## 7. Exception Filters (when clause)

```fsharp
// when clause ใน with handlers

let processHttpResponse (statusCode: int) (body: string) =
    try
        match statusCode with
        | code when code >= 200 && code < 300 ->
            sprintf "Success: %s" body
        | code when code >= 400 && code < 500 ->
            raise (System.Exception(sprintf "Client error: %d - %s" code body))
        | code when code >= 500 ->
            raise (System.Exception(sprintf "Server error: %d - %s" code body))
        | _ ->
            sprintf "Unknown: %d" statusCode
    with
    | :? System.Exception as ex when ex.Message.StartsWith("Client") ->
        printfn "Handling client error: %s" ex.Message
        "client_error"
    | :? System.Exception as ex when ex.Message.StartsWith("Server") ->
        printfn "Handling server error (will retry): %s" ex.Message
        "server_error_retry"

let responses = [(200, "OK"); (404, "Not Found"); (500, "Internal Error"); (200, "Data")]
for (code, body) in responses do
    let result = processHttpResponse code body
    printfn "Response %d: %s" code result

// when clause กับ custom conditions
type OrderException(message: string, amount: decimal) =
    inherit System.Exception(message)
    member this.Amount = amount

let processPayment (amount: decimal) =
    if amount > 10000M then
        raise (OrderException("Amount exceeds limit", amount))
    elif amount <= 0M then
        raise (OrderException("Invalid amount", amount))
    else
        sprintf "Payment of %.2M processed" amount

let payments = [500.0M; -10.0M; 15000.0M; 1000.0M]
for amount in payments do
    try
        let result = processPayment amount
        printfn "Success: %s" result
    with
    | :? OrderException as ex when ex.Amount > 10000M ->
        printfn "High-value transaction alert: %.2M - %s" ex.Amount ex.Message
    | :? OrderException as ex when ex.Amount <= 0M ->
        printfn "Invalid amount: %.2M - %s" ex.Amount ex.Message
    | :? OrderException as ex ->
        printfn "Order error: %.2M - %s" ex.Amount ex.Message
```

---

## 8. Exception Hierarchies

```fsharp
// Exception hierarchy สำหรับ application

// Base
exception AppError of message: string

// Subtypes ด้วย F# exception declarations
exception ValidationError of field: string * message: string
exception NotFoundError of resource: string * id: string
exception ConflictError of resource: string * conflictDetails: string
exception UnauthorizedError of message: string

// Custom exceptions ด้วย class
type BusinessRuleException(rule: string, description: string) =
    inherit System.Exception(sprintf "Business rule '%s' violated: %s" rule description)
    member this.Rule = rule
    member this.Description = description

// F# exception declarations ใน pattern matching
let handleAppException (ex: exn) =
    match ex with
    | :? System.Exception as sysEx ->
        match sysEx with
        | _ when sysEx.GetType() = typeof<System.ArgumentNullException> ->
            printfn "Null argument: %s" (sysEx :?> System.ArgumentNullException).ParamName
        | _ when sysEx.GetType() = typeof<System.ArgumentOutOfRangeException> ->
            printfn "Out of range: %s" (sysEx :?> System.ArgumentOutOfRangeException).ParamName
        | _ ->
            printfn "System exception: %s" sysEx.Message
    | _ ->
        printfn "Unknown exception"

// F# exception types
try
    raise (ValidationError("email", "Invalid format"))
with
| ValidationError(field, msg) ->
    printfn "Validation - Field: %s, Message: %s" field msg

try
    raise (NotFoundError("User", "123"))
with
| NotFoundError(resource, id) ->
    printfn "Not found: %s with id=%s" resource id

try
    raise (BusinessRuleException("MIN_ORDER", "Order must be at least $10"))
with
| :? BusinessRuleException as ex ->
    printfn "Business rule violation: %s - %s" ex.Rule ex.Description

// Exception hierarchy in practice
type IOrderService =
    abstract member CreateOrder: customerId: string -> amount: decimal -> Result<int, string>

type OrderService() =
    interface IOrderService with
        member this.CreateOrder(customerId)(amount) =
            try
                if System.String.IsNullOrEmpty(customerId) then
                    raise (ValidationError("customerId", "Cannot be empty"))
                if amount <= 0M then
                    raise (ValidationError("amount", "Must be positive"))
                
                let orderId = System.Random.Shared.Next(1000, 9999)
                printfn "Created order #%d for customer %s" orderId customerId
                Ok orderId
            with
            | ValidationError(field, msg) ->
                Error (sprintf "Validation failed on %s: %s" field msg)
            | ex ->
                Error (sprintf "Unexpected error: %s" ex.Message)

let orderService: IOrderService = OrderService() :> IOrderService

match orderService.CreateOrder "CUST001" 500.0M with
| Ok orderId -> printfn "Order created: #%d" orderId
| Error msg -> printfn "Error: %s" msg

match orderService.CreateOrder "" 500.0M with
| Ok orderId -> printfn "Order created: #%d" orderId
| Error msg -> printfn "Error: %s" msg
```

---

## 9. Result vs Exception Choice

```fsharp
// เมื่อไหรควรใช้ Result vs Exception

// ใช้ Exception สำหรับ:
// - Truly exceptional situations (programming errors, infrastructure failures)
// - Situations that should never happen in correct code
// - .NET interop (framework expectations)

// ใช้ Result สำหรับ:
// - Expected failure cases (validation, not found, etc.)
// - Domain errors that are part of normal flow
// - API design where caller must handle both cases

// Exception approach (ไม่ดีสำหรับ expected errors)
let parseIntExceptionBased (s: string) =
    System.Int32.Parse(s)  // Throws FormatException on invalid input

// Result approach (ดีกว่าสำหรับ expected errors)
let parseIntResult (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Ok n
    | _ -> Error (sprintf "Cannot parse '%s' as integer" s)

// Domain model ที่ใช้ Result
type UserError =
    | UserNotFound of int
    | InvalidEmail of string
    | DuplicateEmail of string
    | PermissionDenied of string

type UserRepository() =
    let users = System.Collections.Generic.Dictionary<int, {| Id: int; Name: string; Email: string |}>()
    
    do
        users.[1] <- {| Id = 1; Name = "Alice"; Email = "alice@example.com" |}
        users.[2] <- {| Id = 2; Name = "Bob"; Email = "bob@example.com" |}
    
    member this.FindById(id: int) : Result<{| Id: int; Name: string; Email: string |}, UserError> =
        match users.TryGetValue(id) with
        | true, user -> Ok user
        | _ -> Error (UserNotFound id)
    
    member this.UpdateEmail(id: int) (email: string) : Result<unit, UserError> =
        if not (email.Contains("@")) then
            Error (InvalidEmail email)
        elif users.Values |> Seq.exists (fun u -> u.Id <> id && u.Email = email) then
            Error (DuplicateEmail email)
        else
            match users.TryGetValue(id) with
            | true, user ->
                users.[id] <- {| user with Email = email |}
                Ok ()
            | _ ->
                Error (UserNotFound id)

let repo = UserRepository()

let tests = [
    1, "newalice@example.com"
    99, "ghost@example.com"
    2, "not-an-email"
    1, "bob@example.com"  // Duplicate
]

for (id, email) in tests do
    match repo.UpdateEmail id email with
    | Ok () -> printfn "Updated user %d email to: %s" id email
    | Error (UserNotFound uid) -> printfn "User not found: %d" uid
    | Error (InvalidEmail e) -> printfn "Invalid email: %s" e
    | Error (DuplicateEmail e) -> printfn "Duplicate email: %s" e
    | Error (PermissionDenied msg) -> printfn "Permission denied: %s" msg
```

---

## 10. Functional Error Handling

```fsharp
// Functional patterns สำหรับ error handling

// Result computation expression (Railway-oriented programming)
type ResultBuilder() =
    member this.Return(x) = Ok x
    member this.ReturnFrom(m) = m
    member this.Bind(m, f) = Result.bind f m
    member this.Zero() = Ok ()

let result = ResultBuilder()

// Chaining operations ด้วย bind
let validateAndProcess (input: string) =
    result {
        // Step 1: Parse
        let! n = 
            match System.Int32.TryParse(input) with
            | true, v -> Ok v
            | _ -> Error "Not a number"
        
        // Step 2: Validate range
        let! validated =
            if n < 0 then Error "Number must be positive"
            elif n > 100 then Error "Number too large (max 100)"
            else Ok n
        
        // Step 3: Transform
        return validated * 2
    }

let inputs = ["42"; "-5"; "abc"; "150"; "25"]
for input in inputs do
    match validateAndProcess input with
    | Ok result -> printfn "'%s' -> %d" input result
    | Error msg -> printfn "'%s' -> Error: %s" input msg

// Error accumulation (applicative style)
type ValidationResult<'T> =
    | Valid of 'T
    | Invalid of string list

module ValidationResult =
    let pure' x = Valid x
    
    let map f = function
        | Valid v -> Valid (f v)
        | Invalid errors -> Invalid errors
    
    let apply vf va =
        match vf, va with
        | Valid f, Valid a -> Valid (f a)
        | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
        | Invalid e, Valid _ | Valid _, Invalid e -> Invalid e
    
    let (<*>) = apply
    
    let bind f = function
        | Valid v -> f v
        | Invalid errors -> Invalid errors

// Validate all fields and collect all errors
let validateName (name: string) =
    if System.String.IsNullOrWhiteSpace(name) then
        ValidationResult.Invalid ["Name cannot be empty"]
    elif name.Length < 2 then
        ValidationResult.Invalid ["Name too short (min 2 chars)"]
    elif name.Length > 50 then
        ValidationResult.Invalid ["Name too long (max 50 chars)"]
    else
        ValidationResult.Valid name

let validateAge2 (age: int) =
    if age < 0 then ValidationResult.Invalid ["Age cannot be negative"]
    elif age > 150 then ValidationResult.Invalid ["Age too large"]
    else ValidationResult.Valid age

let validateEmail3 (email: string) =
    if System.String.IsNullOrEmpty(email) then
        ValidationResult.Invalid ["Email required"]
    elif not (email.Contains("@")) then
        ValidationResult.Invalid ["Email missing @"]
    elif not (email.Contains(".")) then
        ValidationResult.Invalid ["Email missing domain"]
    else
        ValidationResult.Valid email

// Validate person with all errors collected
let validatePerson name age email =
    match validateName name, validateAge2 age, validateEmail3 email with
    | ValidationResult.Valid n, ValidationResult.Valid a, ValidationResult.Valid e ->
        ValidationResult.Valid {| Name = n; Age = a; Email = e |}
    | nameResult, ageResult, emailResult ->
        let errors = 
            [nameResult; ageResult; emailResult]
            |> List.collect (function
                | ValidationResult.Invalid errs -> errs
                | ValidationResult.Valid _ -> []
            )
        ValidationResult.Invalid errors

match validatePerson "" -5 "invalid" with
| ValidationResult.Valid p -> printfn "Valid person: %s" p.Name
| ValidationResult.Invalid errors ->
    printfn "\nAll validation errors:"
    errors |> List.iter (printfn "  - %s")

match validatePerson "Alice" 30 "alice@example.com" with
| ValidationResult.Valid p -> printfn "Valid: %s, %d, %s" p.Name p.Age p.Email
| ValidationResult.Invalid errors -> printfn "Errors: %A" errors
```

---

## 11. Using Result Instead of Exceptions

```fsharp
// Result<'T, 'E> pattern

// Helper functions
let tryOperation (f: unit -> 'T) : Result<'T, string> =
    try Ok (f())
    with ex -> Error ex.Message

let tryOperationAsync (f: unit -> Async<'T>) : Async<Result<'T, string>> =
    async {
        try
            let! result = f()
            return Ok result
        with ex ->
            return Error ex.Message
    }

// File operations with Result
let tryReadFile (path: string) : Result<string, string> =
    try
        Ok (System.IO.File.ReadAllText(path))
    with
    | :? System.IO.FileNotFoundException ->
        Error (sprintf "File not found: %s" path)
    | :? System.UnauthorizedAccessException ->
        Error (sprintf "Access denied: %s" path)
    | ex ->
        Error (sprintf "IO error: %s" ex.Message)

let tryWriteFile (path: string) (content: string) : Result<unit, string> =
    try
        System.IO.File.WriteAllText(path, content)
        Ok ()
    with
    | :? System.UnauthorizedAccessException ->
        Error (sprintf "Cannot write to: %s" path)
    | ex ->
        Error (sprintf "Write error: %s" ex.Message)

// Result pipeline
let processFile inputPath outputPath =
    inputPath
    |> tryReadFile
    |> Result.map (fun content ->
        content.ToUpper()  // Transform
    )
    |> Result.bind (fun transformed ->
        tryWriteFile outputPath transformed
    )

match processFile "nonexistent.txt" "output.txt" with
| Ok () -> printfn "File processed successfully"
| Error msg -> printfn "Error: %s" msg

// HTTP-like operations with Result
type HttpError =
    | NotFound of string
    | ServerError of string
    | NetworkError of string
    | ParseError of string

let fetchUser (id: int) : Result<{| Id: int; Name: string |}, HttpError> =
    match id with
    | 1 -> Ok {| Id = 1; Name = "Alice" |}
    | 2 -> Ok {| Id = 2; Name = "Bob" |}
    | id when id < 0 -> Error (ServerError "Invalid ID format")
    | _ -> Error (NotFound (sprintf "User %d not found" id))

let getUserName (id: int) =
    fetchUser id
    |> Result.map (fun user -> user.Name)
    |> Result.mapError (function
        | NotFound msg -> sprintf "404: %s" msg
        | ServerError msg -> sprintf "500: %s" msg
        | NetworkError msg -> sprintf "Network: %s" msg
        | ParseError msg -> sprintf "Parse: %s" msg
    )

for id in [1; 2; 99; -1] do
    match getUserName id with
    | Ok name -> printfn "User: %s" name
    | Error msg -> printfn "Error: %s" msg
```

---

## 12. When to Use Exceptions

```fsharp
// Guidelines: when to use exceptions vs Result

// ควรใช้ Exceptions:
// 1. Programming errors (bugs that shouldn't happen)
//    - NullReferenceException, IndexOutOfRangeException
//    - These indicate bugs, not expected conditions

// 2. Infrastructure failures
//    - Network is down
//    - Disk full
//    - Out of memory

// 3. .NET framework/library expectations
//    - Some APIs expect you to throw, not return Result

// 4. Truly unrecoverable situations

// ควรใช้ Result:
// 1. Business rule violations
// 2. User input validation
// 3. Expected "not found" scenarios
// 4. When caller MUST handle the error

// Pattern: ใช้ exceptions ใน low-level, convert ใน higher level
type IRepository<'T> =
    abstract member FindById: int -> 'T option
    abstract member Save: 'T -> unit

type SafeRepository<'T>(inner: IRepository<'T>) =
    member this.TryFindById(id: int) : Result<'T, string> =
        try
            match inner.FindById(id) with
            | Some item -> Ok item
            | None -> Error (sprintf "Item %d not found" id)
        with ex ->
            Error (sprintf "Repository error: %s" ex.Message)
    
    member this.TrySave(item: 'T) : Result<unit, string> =
        try
            inner.Save(item)
            Ok ()
        with ex ->
            Error (sprintf "Save failed: %s" ex.Message)

// Application layer
let runApplication () =
    printfn "=== Exception Handling Best Practices ==="
    printfn ""
    printfn "Use exceptions for:"
    printfn "  - Programming bugs (null refs, bounds violations)"
    printfn "  - Infrastructure failures (IO, network)"
    printfn "  - Truly exceptional situations"
    printfn ""
    printfn "Use Result<T,E> for:"
    printfn "  - Expected failure modes"
    printfn "  - Business rule violations"
    printfn "  - Validation errors"
    printfn "  - When caller must handle both cases"
    printfn ""
    printfn "Conversion pattern:"
    printfn "  try ... with ex -> Error ex.Message"

runApplication()
```

---

## 13. Exception Safety (Cleanup)

```fsharp
// Exception safety - ensure cleanup happens

// RAII pattern ด้วย use keyword
type Resource(name: string) =
    do printfn "Acquiring: %s" name
    
    interface System.IDisposable with
        member this.Dispose() =
            printfn "Releasing: %s" name

// use keyword - automatic disposal
let safeOperation2 () =
    use r1 = new Resource("Resource1")
    use r2 = new Resource("Resource2")
    printfn "Using resources..."
    // Resources released automatically (even if exception)

safeOperation2()

// Exception in the middle - still cleans up
let operationWithException () =
    try
        use r1 = new Resource("A")
        use r2 = new Resource("B")
        printfn "Before error"
        failwith "Oops!"
        printfn "After error (never reached)"
    with
    | ex -> printfn "Caught: %s" ex.Message

printfn "\nOperation with exception:"
operationWithException()

// Nested scopes
let nestedOperation () =
    use outer = new Resource("Outer")
    printfn "In outer scope"
    
    do
        use inner = new Resource("Inner")
        printfn "In inner scope"
        // Inner released here
    
    printfn "Back in outer scope"
    // Outer released here

printfn "\nNested scopes:"
nestedOperation()

// Atomic operations - rollback on failure
type TransactionManager() =
    let mutable operations: (unit -> unit) list = []
    let mutable rollbacks: (unit -> unit) list = []
    
    member this.Do(operation: unit -> unit, rollback: unit -> unit) =
        operation()
        rollbacks <- rollback :: rollbacks
    
    member this.Commit() =
        operations <- []
        rollbacks <- []
        printfn "Transaction committed"
    
    member this.Rollback() =
        for rollback in rollbacks do
            try rollback()
            with ex -> printfn "Rollback error: %s" ex.Message
        rollbacks <- []
        printfn "Transaction rolled back"

let mutable inventory = Map.ofList [("Apple", 10); ("Banana", 5)]

let executeTransaction () =
    let tx = TransactionManager()
    try
        // Add item
        tx.Do(
            (fun () -> inventory <- inventory |> Map.add "Cherry" 15),
            (fun () -> inventory <- inventory |> Map.remove "Cherry")
        )
        
        // Update stock
        let currentApples = inventory.["Apple"]
        tx.Do(
            (fun () -> inventory <- inventory |> Map.add "Apple" (currentApples - 3)),
            (fun () -> inventory <- inventory |> Map.add "Apple" currentApples)
        )
        
        // Simulate error
        failwith "Database unavailable!"
        
        tx.Commit()
    with
    | ex ->
        printfn "Error: %s" ex.Message
        tx.Rollback()

printfn "\nInventory before: %A" inventory
executeTransaction()
printfn "Inventory after rollback: %A" inventory
```

---

## 14. Global Exception Handling

```fsharp
// Global exception handling สำหรับ unhandled exceptions

// Setup global handler
let setupGlobalExceptionHandler () =
    // For .NET applications
    System.AppDomain.CurrentDomain.UnhandledException.Add(fun args ->
        let ex = args.ExceptionObject :?> System.Exception
        printfn "FATAL: Unhandled exception: %s" ex.Message
        printfn "Stack trace: %s" ex.StackTrace
    )
    
    // For async operations
    System.Threading.Tasks.TaskScheduler.UnobservedTaskException.Add(fun args ->
        printfn "Unobserved task exception: %s" args.Exception.Message
        args.SetObserved()  // Prevent process crash
    )

setupGlobalExceptionHandler()

// Structured exception logging
type ExceptionLogger() =
    member this.Log(ex: exn, context: string) =
        let timestamp = System.DateTime.UtcNow.ToString("yyyy-MM-dd HH:mm:ss.fff")
        printfn "[%s] Exception in %s:" timestamp context
        printfn "  Type: %s" (ex.GetType().FullName)
        printfn "  Message: %s" ex.Message
        if ex.InnerException <> null then
            printfn "  Inner: %s" ex.InnerException.Message
        printfn "  Stack: %s" (ex.StackTrace.Split('\n').[0].Trim())

let exLogger = ExceptionLogger()

// Retry pattern
let withRetry (maxRetries: int) (operation: unit -> 'T) =
    let rec retry count =
        try
            operation()
        with ex ->
            if count >= maxRetries then
                printfn "Max retries exceeded"
                reraise()
            else
                printfn "Attempt %d failed: %s, retrying..." count ex.Message
                System.Threading.Thread.Sleep(100 * count)
                retry (count + 1)
    retry 1

let mutable attempt = 0
try
    withRetry 3 (fun () ->
        attempt <- attempt + 1
        if attempt < 3 then failwith (sprintf "Failure attempt %d" attempt)
        else printfn "Success on attempt %d" attempt
    )
with
| ex -> printfn "Failed after retries: %s" ex.Message
```

---

## สรุป (Summary)

```fsharp
printfn "=== Exception Handling Summary ==="
printfn ""
printfn "Syntax:"
printfn "  try"
printfn "    // code"
printfn "  with"
printfn "  | :? SpecificException as ex -> ..."
printfn "  | ex when condition -> ..."
printfn "  | ex -> ..."
printfn ""
printfn "  try ... finally ..."
printfn ""
printfn "Creating exceptions:"
printfn "  failwith 'message'"
printfn "  failwithf 'format' args"
printfn "  raise (MyException 'msg')"
printfn "  invalidArg 'param' 'message'"
printfn "  invalidOp 'message'"
printfn "  nullArg 'param'"
printfn ""
printfn "Re-throwing:"
printfn "  reraise() // preserves stack trace"
printfn "  raise ex  // loses stack trace"
printfn ""
printfn "Functional approach:"
printfn "  Result<T, E> for expected errors"
printfn "  try ... with -> Result conversion"
printfn "  Railway-oriented programming"
printfn ""
printfn "Best practices:"
printfn "  - use keyword for IDisposable"
printfn "  - exceptions for unexpected/unrecoverable"
printfn "  - Result for expected/recoverable errors"
printfn "  - always unsubscribe events"
printfn "  - log exceptions with context"
```

---

## บทสรุป

Exception handling ใน F# มีสองแนวทาง:

1. **Traditional exceptions** - ใช้กับ infrastructure errors, bugs, .NET interop
2. **Result<T,E>** - ใช้กับ expected errors, business rules, validation

สิ่งสำคัญ:
- `try/with` สำหรับจัดการ exceptions
- `try/finally` สำหรับ cleanup
- `use` keyword สำหรับ IDisposable (RAII pattern)
- `reraise()` แทน `raise ex` เพื่อ preserve stack trace
- Custom exceptions สำหรับ domain-specific errors
- Result type สำหรับ functional error handling
- ป้องกัน memory leaks ด้วยการ unsubscribe events
