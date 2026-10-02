# Part 10 - Option และ Result (Handling Absence and Errors)

## บทนำ

การจัดการกับ "ไม่มีค่า" และ "ข้อผิดพลาด" เป็นส่วนสำคัญของการเขียนโปรแกรม F# มีวิธีการที่ type-safe และ elegant สำหรับปัญหาเหล่านี้:

- **Option<'T>**: สำหรับค่าที่อาจไม่มี (`Some value` หรือ `None`)
- **Result<'T, 'E>**: สำหรับ operations ที่อาจ succeed หรือ fail (`Ok value` หรือ `Error error`)

แนวทางเหล่านี้ดีกว่าการใช้ null หรือ exception เพราะ compiler บังคับให้เราจัดการทุก case

---

## 10.1 Option<'T> Deep Dive

### 10.1.1 Option Type Definition

```fsharp
// Option type (built-in ใน F#)
// type Option<'T> =
//     | Some of 'T
//     | None

// การใช้งาน
let someValue: int option = Some 42
let noValue: int option = None
let someString: string option = Some "hello"
let noString: string option = None

printfn "someValue: %A" someValue   // Some 42
printfn "noValue: %A" noValue       // None

// Option type ใน pattern matching
let describeOption opt =
    match opt with
    | Some value -> $"Has value: {value}"
    | None -> "No value"

printfn "%s" (describeOption someValue)
printfn "%s" (describeOption noValue)

// Option ใน function return
let safeDivide a b =
    if b = 0 then None
    else Some (a / b)

printfn "10/2: %A" (safeDivide 10 2)
printfn "10/0: %A" (safeDivide 10 0)
```

### 10.1.2 Some และ None ใน Pattern Matching

```fsharp
// เสมอต้อง handle ทั้ง Some และ None
let processAge ageOpt =
    match ageOpt with
    | Some age when age < 0 ->
        printfn "Invalid age: %d" age
    | Some age when age < 18 ->
        printfn "Minor: %d years" age
    | Some age ->
        printfn "Adult: %d years" age
    | None ->
        printfn "Age unknown"

processAge (Some 25)
processAge (Some 15)
processAge (Some -5)
processAge None

// ใน function parameters
let formatName (firstName: string) (lastName: string option) =
    match lastName with
    | Some last -> $"{firstName} {last}"
    | None -> firstName

printfn "%s" (formatName "John" (Some "Doe"))
printfn "%s" (formatName "Alice" None)

// Nested Some
let findNested (map: Map<string, int option>) key =
    match Map.tryFind key map with
    | Some (Some value) -> $"Found: {value}"
    | Some None -> "Key exists but no value"
    | None -> "Key not found"

let data = Map.ofList [("a", Some 1); ("b", None); ("c", Some 3)]
printfn "%s" (findNested data "a")
printfn "%s" (findNested data "b")
printfn "%s" (findNested data "d")
```

---

## 10.2 Option Module Functions

### 10.2.1 Option.map

```fsharp
// Option.map: แปลงค่าใน Some ถ้ามี, ถ้าเป็น None ก็คืน None

let double x = x * 2
let toString x = string x

let opt1 = Some 5
let opt2: int option = None

// map
let doubled1 = opt1 |> Option.map double   // Some 10
let doubled2 = opt2 |> Option.map double   // None
let strOpt = opt1 |> Option.map toString   // Some "5"

printfn "doubled1: %A" doubled1
printfn "doubled2: %A" doubled2
printfn "strOpt: %A" strOpt

// Real examples
type User = { Name: string; Email: string }

let currentUser: User option = Some { Name = "Alice"; Email = "alice@example.com" }
let noUser: User option = None

let userName = currentUser |> Option.map (fun u -> u.Name)
let noName = noUser |> Option.map (fun u -> u.Name)

printfn "userName: %A" userName   // Some "Alice"
printfn "noName: %A" noName       // None

// map กับ arithmetic
let tryGetAge () : int option = Some 25
let nextBirthday = tryGetAge () |> Option.map (fun age -> age + 1)
printfn "Next birthday age: %A" nextBirthday
```

### 10.2.2 Option.bind

```fsharp
// Option.bind: chain option operations (flatMap)
// ใช้เมื่อ function ที่ต้องการ apply ก็ return Option

let safeDivide a b =
    if b = 0 then None
    else Some (a / b)

let safeIndex (lst: 'a list) (idx: int) =
    if idx < 0 || idx >= lst.Length then None
    else Some lst.[idx]

// bind: ถ้า Some แล้วส่งค่าเข้า function, ถ้า None ก็คืน None
let result1 =
    Some 100
    |> Option.bind (safeDivide 10)  // Some 10

let result2 =
    Some 0
    |> Option.bind (safeDivide 10)  // None (division by zero)

let result3: int option =
    None
    |> Option.bind (safeDivide 10)  // None (already None)

printfn "bind result1: %A" result1
printfn "bind result2: %A" result2
printfn "bind result3: %A" result3

// Chaining ด้วย bind
let parseAndDouble (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None
    |> Option.bind (fun n ->
        if n > 0 then Some (n * 2) else None)

printfn "\nparseAndDouble '5': %A" (parseAndDouble "5")
printfn "parseAndDouble '-3': %A" (parseAndDouble "-3")
printfn "parseAndDouble 'abc': %A" (parseAndDouble "abc")

// Real example: nested data lookup
type Department = { Name: string; ManagerId: int option }
type Employee = { Id: int; Name: string; DepartmentId: int option }

let departments = Map.ofList [
    (1, { Name = "Engineering"; ManagerId = Some 101 })
    (2, { Name = "Marketing"; ManagerId = None })
]

let employees = Map.ofList [
    (101, { Id = 101; Name = "Alice"; DepartmentId = Some 1 })
    (102, { Id = 102; Name = "Bob"; DepartmentId = Some 2 })
]

let getManagerOf (empId: int) =
    Map.tryFind empId employees
    |> Option.bind (fun emp -> emp.DepartmentId)
    |> Option.bind (fun deptId -> Map.tryFind deptId departments)
    |> Option.bind (fun dept -> dept.ManagerId)
    |> Option.bind (fun managerId -> Map.tryFind managerId employees)
    |> Option.map (fun manager -> manager.Name)

printfn "\nManager of 102 (Bob): %A" (getManagerOf 102)
printfn "Manager of 101 (Alice): %A" (getManagerOf 101)
```

### 10.2.3 Option.defaultValue, Option.defaultWith

```fsharp
// Option.defaultValue: return ค่า default ถ้าเป็น None

let opt1 = Some 42
let opt2: int option = None

let val1 = opt1 |> Option.defaultValue 0    // 42
let val2 = opt2 |> Option.defaultValue 0    // 0

printfn "defaultValue: %d, %d" val1 val2

// ใช้ใน code
let getUserName (userOpt: User option) =
    userOpt
    |> Option.map (fun u -> u.Name)
    |> Option.defaultValue "Anonymous"

let user = { Name = "Alice"; Email = "alice@example.com" }
printfn "User: %s" (getUserName (Some user))
printfn "No user: %s" (getUserName None)

// Option.defaultWith: lazy evaluation ของ default value
let expensiveDefault () =
    printfn "Computing expensive default..."
    99

let opt3 = Some 5
let opt4: int option = None

// defaultWith ไม่ compute default value ถ้าไม่จำเป็น
let v3 = opt3 |> Option.defaultWith expensiveDefault  // ไม่ print (มีค่าอยู่แล้ว)
let v4 = opt4 |> Option.defaultWith expensiveDefault  // print (ต้องใช้ default)

printfn "v3: %d, v4: %d" v3 v4
```

### 10.2.4 Option.filter

```fsharp
// Option.filter: กรองค่าใน Option
// ถ้า Some แล้ว predicate ไม่ผ่าน จะ return None

let opt1 = Some 10
let opt2 = Some (-5)
let opt3: int option = None

let positive1 = opt1 |> Option.filter (fun x -> x > 0)  // Some 10
let positive2 = opt2 |> Option.filter (fun x -> x > 0)  // None (negative)
let positive3 = opt3 |> Option.filter (fun x -> x > 0)  // None

printfn "filter positive: %A, %A, %A" positive1 positive2 positive3

// Real example: validate and filter
let parseAge (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

let validAge = 
    parseAge "25"
    |> Option.filter (fun age -> age >= 0 && age <= 150)

let invalidAge =
    parseAge "200"
    |> Option.filter (fun age -> age >= 0 && age <= 150)

printfn "Valid age: %A" validAge    // Some 25
printfn "Invalid age: %A" invalidAge // None

// filter ใน pipeline
let processScore (s: string) =
    parseAge s
    |> Option.filter (fun n -> n >= 0 && n <= 100)
    |> Option.map (fun score ->
        if score >= 80 then "A"
        elif score >= 70 then "B"
        elif score >= 60 then "C"
        else "F")
    |> Option.defaultValue "Invalid score"

printfn "%s" (processScore "85")
printfn "%s" (processScore "110")
printfn "%s" (processScore "abc")
```

### 10.2.5 Option.orElse, Option.orElseWith

```fsharp
// Option.orElse: ถ้าเป็น None ใช้ alternative

let primary: int option = None
let secondary = Some 42

let result = primary |> Option.orElse secondary  // Some 42
printfn "orElse: %A" result

let both = Some 1 |> Option.orElse (Some 2)  // Some 1 (primary wins)
printfn "both Some: %A" both

// ใช้ใน cascading lookups
let tryGetFromCache key = None   // simulate cache miss
let tryGetFromDb key = Some 42   // simulate DB hit
let tryGetFromFile key = Some 0  // backup

let getValue key =
    tryGetFromCache key
    |> Option.orElse (tryGetFromDb key)
    |> Option.orElse (tryGetFromFile key)
    |> Option.defaultValue -1

printfn "getValue: %d" (getValue "someKey")

// orElseWith: lazy alternative
let expensiveOperation () =
    printfn "Expensive fallback operation..."
    Some 99

let opt = None
let result2 = opt |> Option.orElseWith expensiveOperation
printfn "orElseWith: %A" result2

let opt2 = Some 5
let result3 = opt2 |> Option.orElseWith expensiveOperation  // ไม่เรียก expensive
printfn "orElseWith (has value): %A" result3
```

### 10.2.6 Option.toList, Option.toArray, Option.iter

```fsharp
// Option.toList: แปลง Option เป็น list
let opt1 = Some 42
let opt2: int option = None

let list1 = Option.toList opt1    // [42]
let list2 = Option.toList opt2    // []

printfn "toList: %A, %A" list1 list2

// ใช้ประโยชน์: flatten list of options
let results = [Some 1; None; Some 3; None; Some 5]
let values = results |> List.collect Option.toList
printfn "flatten: %A" values  // [1; 3; 5]

// Option.toArray
let arr1 = Option.toArray opt1    // [|42|]
let arr2 = Option.toArray opt2    // [||]

// Option.iter: perform side effect if Some
opt1 |> Option.iter (fun v -> printfn "Value: %d" v)
opt2 |> Option.iter (fun v -> printfn "Won't print: %d" v)

// Real example: conditional operations
let maybeUser: User option = Some { Name = "Alice"; Email = "alice@example.com" }
let noUser: User option = None

maybeUser |> Option.iter (fun u -> printfn "Welcome back, %s!" u.Name)
noUser |> Option.iter (fun u -> printfn "Won't print: %s" u.Name)
```

---

## 10.3 Chaining Options

```fsharp
// Elegant option chaining

type Config = {
    DatabaseUrl: string option
    ApiKey: string option
    MaxConnections: int option
}

let loadConfig () = {
    DatabaseUrl = Some "postgresql://localhost:5432/mydb"
    ApiKey = None
    MaxConnections = Some 10
}

let config = loadConfig ()

// Chain multiple optional values
let connectionInfo =
    match config.DatabaseUrl, config.MaxConnections with
    | Some url, Some max -> Some $"Connecting to {url} with max {max} connections"
    | Some url, None -> Some $"Connecting to {url} (default connections)"
    | None, _ -> None

match connectionInfo with
| Some info -> printfn "%s" info
| None -> printfn "No database URL configured!"

// Using Option module for chaining
let processConfig (cfg: Config) =
    cfg.DatabaseUrl
    |> Option.map (fun url -> url.Replace("postgresql://", ""))
    |> Option.bind (fun url ->
        if url.Contains(":") then Some (url.Split(':').[0]) else None)
    |> Option.map (fun host -> $"Database host: {host}")
    |> Option.defaultValue "Unable to parse database host"

printfn "%s" (processConfig config)
```

---

## 10.4 Result<'T, 'E> Type

### 10.4.1 Result Type พื้นฐาน

```fsharp
// Result type (built-in ใน F#)
// type Result<'T, 'E> =
//     | Ok of 'T
//     | Error of 'E

// การใช้งาน
let success: Result<int, string> = Ok 42
let failure: Result<int, string> = Error "Something went wrong"

printfn "success: %A" success
printfn "failure: %A" failure

// Pattern matching กับ Result
let processResult result =
    match result with
    | Ok value -> printfn "Success! Value = %d" value
    | Error msg -> printfn "Error: %s" msg

processResult success
processResult failure

// Function ที่ return Result
let safeDivide (a: float) (b: float) : Result<float, string> =
    if b = 0.0 then Error "Division by zero"
    else Ok (a / b)

let safeSqrt (x: float) : Result<float, string> =
    if x < 0.0 then Error "Cannot take sqrt of negative"
    else Ok (sqrt x)

printfn "\n%A" (safeDivide 10.0 3.0)
printfn "%A" (safeDivide 10.0 0.0)
printfn "%A" (safeSqrt 16.0)
printfn "%A" (safeSqrt -4.0)
```

### 10.4.2 Result.map, Result.mapError

```fsharp
// Result.map: แปลงค่าใน Ok ถ้า success

let double x = x * 2

let ok = Ok 5
let err = Error "error"

let mapped1 = ok |> Result.map double    // Ok 10
let mapped2 = err |> Result.map double   // Error "error" (unchanged)

printfn "mapped1: %A" mapped1
printfn "mapped2: %A" mapped2

// Result.mapError: แปลง error value
let mapErr = Error 404 |> Result.mapError (fun code -> $"HTTP Error {code}")
printfn "mapError: %A" mapErr  // Error "HTTP Error 404"

// Real example
type DatabaseError = ConnectionFailed | QueryFailed of string | Timeout

let queryDatabase () : Result<int list, DatabaseError> =
    // Simulate success
    Ok [1; 2; 3; 4; 5]

let processData () =
    queryDatabase ()
    |> Result.map (fun data -> data |> List.map (fun x -> x * x))  // square
    |> Result.map (fun data -> data |> List.sum)                     // sum
    |> Result.mapError (fun err ->
        match err with
        | ConnectionFailed -> "Could not connect to database"
        | QueryFailed msg -> $"Query failed: {msg}"
        | Timeout -> "Database timeout")

match processData () with
| Ok total -> printfn "Total: %d" total
| Error msg -> printfn "Error: %s" msg
```

### 10.4.3 Result.bind

```fsharp
// Result.bind: chain result operations

let parseFloat (s: string) : Result<float, string> =
    match System.Double.TryParse(s) with
    | true, f -> Ok f
    | false, _ -> Error $"Cannot parse '{s}' as float"

let ensurePositive (x: float) : Result<float, string> =
    if x > 0.0 then Ok x
    else Error $"Value must be positive, got {x}"

let computeSqrt (x: float) : Result<float, string> =
    Ok (sqrt x)

// Chain operations
let safeComputation input =
    parseFloat input
    |> Result.bind ensurePositive
    |> Result.bind computeSqrt

let inputs = ["16.0"; "-4.0"; "abc"; "25.0"]
for input in inputs do
    match safeComputation input with
    | Ok result -> printfn "'%s' -> sqrt = %.4f" input result
    | Error msg -> printfn "'%s' -> Error: %s" input msg
```

### 10.4.4 Result.isOk, Result.isError

```fsharp
// Check result type without matching

let ok = Ok 42
let err = Error "oops"

printfn "ok isOk: %b" (Result.isOk ok)     // true
printfn "ok isError: %b" (Result.isError ok) // false
printfn "err isOk: %b" (Result.isOk err)   // false
printfn "err isError: %b" (Result.isError err) // true

// ใช้ใน filtering
let results = [Ok 1; Error "a"; Ok 3; Error "b"; Ok 5]
let successes = results |> List.filter Result.isOk
let failures = results |> List.filter Result.isError

printfn "Successes: %d" successes.Length
printfn "Failures: %d" failures.Length

// Get value (unsafe)
let successValues = 
    results 
    |> List.choose (function Ok v -> Some v | Error _ -> None)
printfn "Success values: %A" successValues
```

---

## 10.5 Railway-Oriented Programming

```fsharp
// Railway-Oriented Programming: composing Result operations

// ฟังก์ชันสำหรับ railway operations
let (>>=) result f = Result.bind f result
let (>>|) result f = Result.map f result

// Validation pipeline
type User = {
    Username: string
    Email: string
    Age: int
    Password: string
}

type ValidationError =
    | EmptyField of string
    | TooShort of string * int
    | TooLong of string * int
    | InvalidFormat of string * string
    | OutOfRange of string * int * int

let validateUsername (s: string) : Result<string, ValidationError> =
    if s = "" then Error (EmptyField "username")
    elif s.Length < 3 then Error (TooShort("username", 3))
    elif s.Length > 20 then Error (TooLong("username", 20))
    else Ok s

let validateEmail (s: string) : Result<string, ValidationError> =
    if s = "" then Error (EmptyField "email")
    elif not (s.Contains("@")) then Error (InvalidFormat("email", "must contain @"))
    elif not (s.Contains(".")) then Error (InvalidFormat("email", "must contain ."))
    else Ok s

let validateAge (n: int) : Result<int, ValidationError> =
    if n < 0 || n > 150 then Error (OutOfRange("age", 0, 150))
    else Ok n

let validatePassword (s: string) : Result<string, ValidationError> =
    if s.Length < 8 then Error (TooShort("password", 8))
    elif not (s |> Seq.exists System.Char.IsUpper) then
        Error (InvalidFormat("password", "must contain uppercase"))
    elif not (s |> Seq.exists System.Char.IsDigit) then
        Error (InvalidFormat("password", "must contain digit"))
    else Ok s

// Create user with validation
let createUser username email age password : Result<User, ValidationError> =
    validateUsername username
    >>= (fun _ -> validateEmail email)
    >>= (fun _ -> validateAge age)
    >>= (fun _ -> validatePassword password)
    >>| (fun _ -> { Username = username; Email = email; Age = age; Password = password })

let formatError = function
    | EmptyField f -> $"{f} is required"
    | TooShort(f, min) -> $"{f} must be at least {min} characters"
    | TooLong(f, max) -> $"{f} must be at most {max} characters"
    | InvalidFormat(f, msg) -> $"{f}: {msg}"
    | OutOfRange(f, min, max) -> $"{f} must be between {min} and {max}"

// Test
let tests = [
    ("alice_99", "alice@example.com", 25, "Password1")
    ("ab", "alice@example.com", 25, "Password1")
    ("alice", "invalid-email", 25, "Password1")
    ("alice", "alice@example.com", 200, "Password1")
    ("alice", "alice@example.com", 25, "weak")
]

printfn "User creation tests:"
for (username, email, age, pass) in tests do
    match createUser username email age pass with
    | Ok user -> printfn "✓ Created user: %s" user.Username
    | Error err -> printfn "✗ Failed: %s" (formatError err)
```

---

## 10.6 Converting Between Option และ Result

```fsharp
// แปลงระหว่าง Option และ Result

// Option -> Result
let optionToResult (errorMsg: 'e) (opt: 'a option) : Result<'a, 'e> =
    match opt with
    | Some v -> Ok v
    | None -> Error errorMsg

// Result -> Option (ทิ้ง error info)
let resultToOption (result: Result<'a, 'e>) : 'a option =
    match result with
    | Ok v -> Some v
    | Error _ -> None

// Built-in helpers
// Option.toResultWith (F# 5+)

// Practical examples
let lookupUser userId =
    Map.tryFind userId (Map.ofList [(1, "Alice"); (2, "Bob")])

let getUser userId =
    lookupUser userId
    |> optionToResult $"User {userId} not found"

match getUser 1 with
| Ok name -> printfn "Found: %s" name
| Error msg -> printfn "Error: %s" msg

match getUser 99 with
| Ok name -> printfn "Found: %s" name
| Error msg -> printfn "Error: %s" msg

// Option.ofObj / Option.toObj (null handling)
let nullString: string = null
let notNull: string = "hello"

let opt1 = Option.ofObj nullString   // None
let opt2 = Option.ofObj notNull      // Some "hello"

printfn "ofObj null: %A" opt1
printfn "ofObj 'hello': %A" opt2

let back1 = Option.toObj opt1       // null
let back2 = Option.toObj opt2       // "hello"
printfn "toObj None: %A" back1
printfn "toObj Some: %A" back2
```

---

## 10.7 tryXxx Patterns

```fsharp
// F# มีแนวทาง tryXxx สำหรับ operations ที่อาจ fail

// Standard tryXxx pattern
let tryParseInt (s: string) : int option =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

let tryParseFloat (s: string) : float option =
    match System.Double.TryParse(s) with
    | true, f -> Some f
    | false, _ -> None

let tryParseDate (s: string) : System.DateTime option =
    match System.DateTime.TryParse(s) with
    | true, d -> Some d
    | false, _ -> None

// ใช้งาน
let inputs = ["42"; "3.14"; "2024-03-15"; "invalid"]

for input in inputs do
    let intResult = tryParseInt input
    let floatResult = tryParseFloat input
    let dateResult = tryParseDate input
    
    match intResult, floatResult, dateResult with
    | Some n, _, _ -> printfn "'%s' -> int: %d" input n
    | _, Some f, _ -> printfn "'%s' -> float: %f" input f
    | _, _, Some d -> printfn "'%s' -> date: %O" input d
    | _ -> printfn "'%s' -> cannot parse" input

// tryXxx สำหรับ collections
let tryHead (lst: 'a list) = 
    match lst with
    | [] -> None
    | h :: _ -> Some h

let tryLast (lst: 'a list) =
    if lst = [] then None
    else Some (List.last lst)

let tryItem index (lst: 'a list) =
    if index < 0 || index >= lst.Length then None
    else Some lst.[index]

let lst = [1; 2; 3; 4; 5]
printfn "\nList operations:"
printfn "tryHead: %A" (tryHead lst)
printfn "tryHead []: %A" (tryHead<int> [])
printfn "tryItem 2: %A" (tryItem 2 lst)
printfn "tryItem 10: %A" (tryItem 10 lst)
```

---

## 10.8 Custom Error Types กับ DUs

```fsharp
// ใช้ DU สำหรับ structured errors

type AppError =
    | NotFound of entityName: string * id: string
    | Unauthorized of userId: string * action: string
    | ValidationFailed of errors: Map<string, string list>
    | DatabaseError of operation: string * message: string
    | NetworkError of url: string * statusCode: int
    | Timeout of operation: string * timeoutMs: int
    | Unexpected of message: string * innerError: exn option

// Format errors เพื่อ display
let formatAppError error =
    match error with
    | NotFound(entity, id) ->
        $"Not Found: {entity} with ID '{id}' does not exist"
    | Unauthorized(user, action) ->
        $"Unauthorized: User '{user}' cannot perform '{action}'"
    | ValidationFailed errors ->
        let errorList = 
            errors 
            |> Map.toList 
            |> List.collect (fun (field, msgs) -> msgs |> List.map (fun m -> $"  - {field}: {m}"))
        "Validation Failed:\n" + String.concat "\n" errorList
    | DatabaseError(op, msg) ->
        $"Database Error during '{op}': {msg}"
    | NetworkError(url, code) ->
        $"Network Error: HTTP {code} from {url}"
    | Timeout(op, ms) ->
        $"Timeout: '{op}' exceeded {ms}ms"
    | Unexpected(msg, Some ex) ->
        $"Unexpected Error: {msg} (caused by: {ex.Message})"
    | Unexpected(msg, None) ->
        $"Unexpected Error: {msg}"

// ใช้งาน
let errors = [
    NotFound("User", "12345")
    Unauthorized("user123", "delete_admin")
    ValidationFailed(Map.ofList [
        ("email", ["Invalid format"; "Already in use"])
        ("password", ["Too short"])
    ])
    DatabaseError("INSERT", "Duplicate key violation")
    NetworkError("https://api.example.com/users", 503)
    Timeout("SendEmail", 5000)
]

printfn "Error messages:"
errors |> List.iter (fun e -> printfn "\n%s" (formatAppError e))
```

---

## 10.9 Error Aggregation

```fsharp
// รวบรวม errors หลาย errors แทนที่จะ fail ที่ first error

type ValidationResult<'T> =
    | Valid of 'T
    | Invalid of errors: string list

// Combine validation results
let andAlso result1 result2 =
    match result1, result2 with
    | Valid v1, Valid v2 -> Valid (v1, v2)
    | Invalid e1, Valid _ -> Invalid e1
    | Valid _, Invalid e2 -> Invalid e2
    | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)

// Validation functions
let validateNotEmpty (fieldName: string) (value: string) =
    if System.String.IsNullOrWhiteSpace(value)
    then Invalid [$"{fieldName} is required"]
    else Valid value

let validateMinLength (fieldName: string) (minLen: int) (value: string) =
    if value.Length < minLen
    then Invalid [$"{fieldName} must be at least {minLen} characters"]
    else Valid value

let validateMaxLength (fieldName: string) (maxLen: int) (value: string) =
    if value.Length > maxLen
    then Invalid [$"{fieldName} must be at most {maxLen} characters"]
    else Valid value

// Validate all fields and collect all errors
let validateRegistration username email password =
    let usernameValidation =
        [validateNotEmpty "Username" username
         validateMinLength "Username" 3 username
         validateMaxLength "Username" 20 username]
        |> List.fold (fun acc v ->
            match acc, v with
            | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
            | Invalid e, Valid _ -> Invalid e
            | Valid _, Invalid e -> Invalid e
            | Valid _, Valid _ -> Valid username) (Valid username)
    
    let emailValidation =
        if not (email.Contains("@")) then
            Invalid ["Email must contain @"]
        else Valid email
    
    let passwordValidation =
        [validateMinLength "Password" 8 password]
        |> List.fold (fun acc v ->
            match acc, v with
            | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
            | Invalid e, Valid _ -> Invalid e
            | Valid _, Invalid e -> Invalid e
            | Valid _, Valid _ -> Valid password) (Valid password)
    
    match usernameValidation, emailValidation, passwordValidation with
    | Valid u, Valid e, Valid p -> Valid {| Username = u; Email = e; Password = p |}
    | u, e, p ->
        let errors = [
            match u with Invalid errs -> yield! errs | _ -> ()
            match e with Invalid errs -> yield! errs | _ -> ()
            match p with Invalid errs -> yield! errs | _ -> ()
        ]
        Invalid errors

// Test
let testCases = [
    ("alice", "alice@example.com", "password123")
    ("ab", "invalid-email", "short")
    ("", "", "")
]

printfn "Registration validation:"
for (username, email, password) in testCases do
    printfn "\nUsername='%s', Email='%s', Password='%s'" username email password
    match validateRegistration username email password with
    | Valid user -> printfn "✓ Valid! Username: %s" user.Username
    | Invalid errors ->
        printfn "✗ Errors:"
        errors |> List.iter (fun e -> printfn "  - %s" e)
```

---

## 10.10 ตัวอย่างโปรแกรมสมบูรณ์

### 10.10.1 Form Validation System

```fsharp
// form_validation.fsx - Complete form validation

open System
open System.Text.RegularExpressions

// Error types
type FieldError =
    | Required
    | MinLength of int
    | MaxLength of int
    | InvalidEmail
    | InvalidPhone
    | WeakPassword
    | MustMatch of string

type FormErrors = Map<string, FieldError list>

// Validation builder
type Validator<'T> = {
    Value: 'T
    FieldName: string
    Errors: FieldError list
}

let validate fieldName value = { Value = value; FieldName = fieldName; Errors = [] }

let check (rule: 'T -> bool) (error: FieldError) (v: Validator<'T>) =
    if rule v.Value then v
    else { v with Errors = v.Errors @ [error] }

let required (v: Validator<string>) =
    check (fun s -> not (String.IsNullOrWhiteSpace s)) Required v

let minLength min (v: Validator<string>) =
    check (fun s -> s.Length >= min) (MinLength min) v

let maxLength max (v: Validator<string>) =
    check (fun s -> s.Length <= max) (MaxLength max) v

let isEmail (v: Validator<string>) =
    check (fun s -> Regex.IsMatch(s, @"^[^@\s]+@[^@\s]+\.[^@\s]+$")) InvalidEmail v

let isPhone (v: Validator<string>) =
    check (fun s -> Regex.IsMatch(s, @"^[0-9\-+\s()]{7,15}$")) InvalidPhone v

let isStrongPassword (v: Validator<string>) =
    check (fun s ->
        s.Length >= 8 &&
        s |> Seq.exists Char.IsUpper &&
        s |> Seq.exists Char.IsLower &&
        s |> Seq.exists Char.IsDigit) WeakPassword v

let matches (other: string) otherName (v: Validator<string>) =
    check (fun s -> s = other) (MustMatch otherName) v

let result (v: Validator<'T>) : Result<'T, (string * FieldError list)> =
    if v.Errors = [] then Ok v.Value
    else Error (v.FieldName, v.Errors)

// Registration form
type RegistrationForm = {
    FirstName: string
    LastName: string
    Email: string
    Phone: string
    Password: string
    ConfirmPassword: string
}

let validateForm (form: RegistrationForm) : Result<RegistrationForm, FormErrors> =
    let validations = [
        validate "First Name" form.FirstName
        |> required |> minLength 2 |> maxLength 50
        |> result |> Result.map ignore
        
        validate "Last Name" form.LastName
        |> required |> minLength 2 |> maxLength 50
        |> result |> Result.map ignore
        
        validate "Email" form.Email
        |> required |> isEmail
        |> result |> Result.map ignore
        
        validate "Phone" form.Phone
        |> required |> isPhone
        |> result |> Result.map ignore
        
        validate "Password" form.Password
        |> required |> isStrongPassword
        |> result |> Result.map ignore
        
        validate "Confirm Password" form.ConfirmPassword
        |> required |> matches form.Password "Password"
        |> result |> Result.map ignore
    ]
    
    let errors =
        validations
        |> List.choose (function
            | Error (field, errs) -> Some (field, errs)
            | Ok _ -> None)
    
    if errors = [] then Ok form
    else Error (Map.ofList errors)

let formatFieldError = function
    | Required -> "This field is required"
    | MinLength n -> $"Must be at least {n} characters"
    | MaxLength n -> $"Must be at most {n} characters"
    | InvalidEmail -> "Invalid email address"
    | InvalidPhone -> "Invalid phone number"
    | WeakPassword -> "Password must be 8+ chars with upper, lower, and digit"
    | MustMatch field -> $"Must match {field}"

let printFormErrors errors =
    errors |> Map.iter (fun field errs ->
        errs |> List.iter (fun err ->
            printfn "  [%s] %s" field (formatFieldError err)))

// Test forms
let validForm = {
    FirstName = "สมชาย"
    LastName = "ใจดี"
    Email = "somchai@example.com"
    Phone = "081-234-5678"
    Password = "P@ssword1"
    ConfirmPassword = "P@ssword1"
}

let invalidForm = {
    FirstName = "A"
    LastName = ""
    Email = "not-an-email"
    Phone = "abc"
    Password = "weak"
    ConfirmPassword = "different"
}

printfn "=== Form Validation ==="
printfn "\nValid form:"
match validateForm validForm with
| Ok _ -> printfn "✓ Registration successful!"
| Error errors ->
    printfn "✗ Validation errors:"
    printFormErrors errors

printfn "\nInvalid form:"
match validateForm invalidForm with
| Ok _ -> printfn "✓ Registration successful!"
| Error errors ->
    printfn "✗ Validation errors:"
    printFormErrors errors
```

### 10.10.2 Data Pipeline กับ Error Handling

```fsharp
// data_pipeline.fsx - Data processing pipeline with error handling

open System

type DataError =
    | ParseError of string
    | ValidationError of string
    | ProcessingError of string

type RawData = { Line: int; Data: string }
type ParsedData = { Line: int; Value: float; Unit: string }
type ProcessedData = { Line: int; NormalizedValue: float; Category: string }

// Step 1: Parse
let parseData (raw: RawData) : Result<ParsedData, DataError> =
    let parts = raw.Data.Split(' ')
    match parts with
    | [| valueStr; unit |] ->
        match Double.TryParse(valueStr) with
        | true, value -> Ok { Line = raw.Line; Value = value; Unit = unit }
        | false, _ -> Error (ParseError $"Line {raw.Line}: Cannot parse '{valueStr}' as number")
    | _ ->
        Error (ParseError $"Line {raw.Line}: Expected 'value unit' format, got '{raw.Data}'")

// Step 2: Validate
let validateData (parsed: ParsedData) : Result<ParsedData, DataError> =
    if parsed.Value < 0.0 then
        Error (ValidationError $"Line {parsed.Line}: Negative value not allowed")
    elif not (["kg"; "lbs"; "g"] |> List.contains parsed.Unit) then
        Error (ValidationError $"Line {parsed.Line}: Unknown unit '{parsed.Unit}'")
    else
        Ok parsed

// Step 3: Process/normalize to kg
let processData (validated: ParsedData) : Result<ProcessedData, DataError> =
    let toKg value unit =
        match unit with
        | "kg" -> Ok value
        | "lbs" -> Ok (value * 0.453592)
        | "g" -> Ok (value / 1000.0)
        | u -> Error (ProcessingError $"Unknown unit: {u}")
    
    match toKg validated.Value validated.Unit with
    | Error e -> Error e
    | Ok kgValue ->
        let category =
            if kgValue < 10.0 then "Light"
            elif kgValue < 50.0 then "Medium"
            elif kgValue < 200.0 then "Heavy"
            else "Very Heavy"
        
        Ok { Line = validated.Line; NormalizedValue = kgValue; Category = category }

// Pipeline
let processLine line =
    line
    |> parseData
    |> Result.bind validateData
    |> Result.bind processData

// Test data
let rawData = [
    { Line = 1; Data = "5.5 kg" }
    { Line = 2; Data = "100 lbs" }
    { Line = 3; Data = "500 g" }
    { Line = 4; Data = "-10 kg" }
    { Line = 5; Data = "abc kg" }
    { Line = 6; Data = "50 tons" }
    { Line = 7; Data = "invalid format here extra" }
]

printfn "=== Data Processing Pipeline ==="
let (successes, failures) = rawData |> List.partition (fun d ->
    match processLine d with Ok _ -> true | Error _ -> false)

printfn "\n✓ Successful (%d):" successes.Length
successes |> List.iter (fun d ->
    match processLine d with
    | Ok result ->
        printfn "  Line %d: %.3f kg (%s)" result.Line result.NormalizedValue result.Category
    | _ -> ())

printfn "\n✗ Failed (%d):" failures.Length
failures |> List.iter (fun d ->
    match processLine d with
    | Error err ->
        let errMsg = match err with
                     | ParseError m | ValidationError m | ProcessingError m -> m
        printfn "  %s" errMsg
    | _ -> ())

// Statistics
let processed = 
    rawData 
    |> List.choose (fun d ->
        match processLine d with Ok r -> Some r | _ -> None)

if processed <> [] then
    let avgKg = processed |> List.averageBy (fun d -> d.NormalizedValue)
    let maxKg = processed |> List.maxBy (fun d -> d.NormalizedValue)
    printfn "\nStatistics:"
    printfn "  Average: %.3f kg" avgKg
    printfn "  Heaviest: Line %d = %.3f kg" maxKg.Line maxKg.NormalizedValue
```

---

## สรุป Part 10

ในบทนี้เราได้เรียนรู้:
- ✅ Option<'T>: Some และ None สำหรับ optional values
- ✅ Option.map: แปลงค่าใน Some
- ✅ Option.bind: chain option operations
- ✅ Option.defaultValue, Option.defaultWith
- ✅ Option.filter
- ✅ Option.orElse, Option.orElseWith
- ✅ Option.toList, Option.toArray, Option.iter
- ✅ Chaining options
- ✅ Result<'T, 'E>: Ok และ Error
- ✅ Result.map, Result.mapError, Result.bind
- ✅ Result.isOk, Result.isError
- ✅ Railway-Oriented Programming
- ✅ Converting between Option และ Result
- ✅ Working with null (.NET interop)
- ✅ Option.ofObj, Option.toObj
- ✅ tryXxx patterns
- ✅ Custom error types ด้วย DUs
- ✅ Error aggregation
- ✅ Form validation และ data pipeline

## สรุปหลักสูตร F# ทั้งหมด

ตลอด 10 บทที่ผ่านมา เราได้เรียนรู้:

1. **Introduction** - F# คืออะไร, ติดตั้ง, Hello World
2. **Basic Syntax** - let bindings, operators, I/O
3. **Types and Inference** - type system, conversion, inference
4. **Functions** - currying, composition, higher-order, lambda
5. **Pattern Matching** - exhaustive, guards, active patterns
6. **Lists and Collections** - List, Seq, Array, Map, Set
7. **Tuples and Records** - compound types, update syntax
8. **Discriminated Unions** - algebraic types, state machines
9. **Modules and Namespaces** - code organization, visibility
10. **Option and Result** - null-safety, error handling, railway

**F# เป็นภาษาที่ทรงพลังมาก ด้วยการผสมผสาน functional programming กับ .NET ecosystem ทำให้เหมาะสำหรับ:**
- Domain modeling ที่ type-safe
- Data processing pipelines
- Business applications
- Financial systems
- Scientific computing
- Web development (ด้วย Giraffe/Saturn/Fable)
