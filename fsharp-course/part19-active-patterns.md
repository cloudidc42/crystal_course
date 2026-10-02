# Part 19 - แอคทีฟแพทเทิร์น (Active Patterns)

## บทนำ (Introduction)

Active Patterns คือ F# feature ที่ให้เราสร้าง pattern เองใน pattern matching ได้ ทำให้ code อ่านง่ายขึ้นและสามารถ abstract ความซับซ้อนออกไปได้

---

## 1. What Are Active Patterns?

```fsharp
// Active Pattern คือฟังก์ชันที่ใช้ใน pattern matching ได้
// ใช้วงเล็บพิเศษ (| ... |) ในการประกาศ

// Active Pattern พื้นฐาน: Even/Odd
let (|Even|Odd|) n =
    if n % 2 = 0 then Even else Odd

// ใช้ใน pattern matching
let classify n =
    match n with
    | Even -> sprintf "%d is even" n
    | Odd -> sprintf "%d is odd" n

// ทดสอบ
[1..10] |> List.iter (fun n -> printfn "%s" (classify n))
```

---

## 2. Single-Case Active Pattern

```fsharp
// Single-case: แปลงค่าก่อน match

// แปลง string เป็น lowercase
let (|Lower|) (s: string) = s.ToLower()

// ใช้งาน
let greet name =
    match name with
    | Lower n -> sprintf "Hello, %s!" n  // n เป็น lowercase เสมอ

printfn "%s" (greet "ALICE")  // "Hello, alice!"
printfn "%s" (greet "Bob")    // "Hello, bob!"
```

```fsharp
// Single-case ที่ซับซ้อนขึ้น

// Trim whitespace
let (|Trimmed|) (s: string) = s.Trim()

// แยก email ออกเป็น username และ domain
let (|EmailParts|) (email: string) =
    let parts = email.Split('@')
    if parts.Length = 2 then (parts.[0], parts.[1])
    else ("", "")

let analyzeEmail email =
    match email with
    | EmailParts (user, domain) ->
        printfn "User: %s, Domain: %s" user domain

analyzeEmail "alice@example.com"   // User: alice, Domain: example.com
analyzeEmail "bob@gmail.com"       // User: bob, Domain: gmail.com
```

---

## 3. Multi-Case Active Pattern

```fsharp
// Multi-case: จัดกลุ่มค่าหลายๆ แบบ

// จัดกลุ่มตัวเลข
let (|Negative|Zero|Positive|) n =
    if n < 0 then Negative
    elif n = 0 then Zero
    else Positive

let describeNumber n =
    match n with
    | Negative -> sprintf "%d is negative" n
    | Zero -> "zero"
    | Positive -> sprintf "%d is positive" n

[-3; 0; 5; -1; 2] |> List.iter (fun n -> printfn "%s" (describeNumber n))
```

```fsharp
// จัดกลุ่ม day of week
let (|Weekend|Weekday|) (day: System.DayOfWeek) =
    match day with
    | System.DayOfWeek.Saturday | System.DayOfWeek.Sunday -> Weekend
    | _ -> Weekday

let today = System.DateTime.Today.DayOfWeek
match today with
| Weekend -> printfn "It's the weekend!"
| Weekday -> printfn "It's a weekday."
```

```fsharp
// Multi-case สำหรับ HTTP status codes
let (|Success|Redirect|ClientError|ServerError|Unknown|) (code: int) =
    if code >= 200 && code < 300 then Success
    elif code >= 300 && code < 400 then Redirect
    elif code >= 400 && code < 500 then ClientError
    elif code >= 500 && code < 600 then ServerError
    else Unknown

let describeStatus code =
    match code with
    | Success -> sprintf "%d: OK" code
    | Redirect -> sprintf "%d: Redirect" code
    | ClientError -> sprintf "%d: Client Error" code
    | ServerError -> sprintf "%d: Server Error" code
    | Unknown -> sprintf "%d: Unknown" code

[200; 301; 404; 500; 999] |> List.iter (fun c -> printfn "%s" (describeStatus c))
```

---

## 4. Partial Active Patterns

```fsharp
// Partial active patterns: match อาจสำเร็จหรือล้มเหลว
// ใช้ (|Name|_|) format: ส่งคืน option

// ParseInt: แปลง string เป็น int ถ้าทำได้
let (|ParseInt|_|) (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

// ParseFloat: แปลง string เป็น float
let (|ParseFloat|_|) (s: string) =
    match System.Double.TryParse(s) with
    | true, f -> Some f
    | false, _ -> None

// ใช้ใน pattern matching
let parseInput input =
    match input with
    | ParseInt n -> sprintf "Integer: %d" n
    | ParseFloat f -> sprintf "Float: %f" f
    | s -> sprintf "String: %s" s

let inputs = ["42"; "3.14"; "hello"; "-100"; "2.718"]
inputs |> List.iter (fun s -> printfn "%s" (parseInput s))
```

```fsharp
// Partial pattern สำหรับ validation

// Email validation
let (|ValidEmail|_|) (s: string) =
    if s.Contains("@") && s.Contains(".") && s.Length > 5 then
        Some s
    else None

// URL validation
let (|ValidUrl|_|) (s: string) =
    if s.StartsWith("http://") || s.StartsWith("https://") then
        Some s
    else None

// Phone number
let (|ValidPhone|_|) (s: string) =
    if s.Length = 10 && s |> Seq.forall System.Char.IsDigit then
        Some s
    else None

let validateInput input =
    match input with
    | ValidEmail e -> sprintf "Valid email: %s" e
    | ValidUrl u -> sprintf "Valid URL: %s" u
    | ValidPhone p -> sprintf "Valid phone: %s" p
    | _ -> sprintf "Invalid input: %s" input

let inputs = ["alice@example.com"; "https://example.com"; "0812345678"; "invalid"]
inputs |> List.iter (fun s -> printfn "%s" (validateInput s))
```

---

## 5. Parameterized Active Patterns

```fsharp
// Active patterns ที่รับ parameter เพิ่มเติม

// StartsWith: ตรวจสอบว่า string เริ่มต้นด้วย prefix
let (|StartsWith|_|) (prefix: string) (s: string) =
    if s.StartsWith(prefix) then Some (s.Substring(prefix.Length))
    else None

// EndsWith: ตรวจสอบว่า string ลงท้ายด้วย suffix
let (|EndsWith|_|) (suffix: string) (s: string) =
    if s.EndsWith(suffix) then Some (s.Substring(0, s.Length - suffix.Length))
    else None

// ใช้งาน
let classifyFile filename =
    match filename with
    | EndsWith ".fsx" name -> sprintf "F# Script: %s" name
    | EndsWith ".fs" name -> sprintf "F# Source: %s" name
    | EndsWith ".md" name -> sprintf "Markdown: %s" name
    | EndsWith ".json" name -> sprintf "JSON: %s" name
    | _ -> sprintf "Unknown: %s" filename

["main.fsx"; "Program.fs"; "README.md"; "config.json"; "image.png"]
|> List.iter (fun f -> printfn "%s" (classifyFile f))
```

```fsharp
// Divisible: ตรวจสอบว่า หารด้วย n ลงตัว
let (|Divisible|_|) n x =
    if x % n = 0 then Some (x / n) else None

// FizzBuzz ด้วย active patterns!
let fizzBuzz n =
    match n with
    | Divisible 15 _ -> "FizzBuzz"
    | Divisible 3 _ -> "Fizz"
    | Divisible 5 _ -> "Buzz"
    | n -> string n

[1..20] |> List.iter (fun n -> printfn "%s" (fizzBuzz n))
```

```fsharp
// Between: ตรวจสอบว่าค่าอยู่ในช่วง
let (|Between|_|) (lo, hi) x =
    if x >= lo && x <= hi then Some x else None

let gradeToLetter score =
    match score with
    | Between (90, 100) _ -> "A"
    | Between (80, 89) _ -> "B"
    | Between (70, 79) _ -> "C"
    | Between (60, 69) _ -> "D"
    | _ -> "F"

[95; 85; 75; 65; 55] |> List.iter (fun s ->
    printfn "%d -> %s" s (gradeToLetter s))
```

---

## 6. (|Even|Odd|) Example

```fsharp
// Classic Even/Odd example ใน depth

let (|Even|Odd|) n =
    if n % 2 = 0 then Even else Odd

// ใช้ใน pattern matching ซับซ้อน
let describeList lst =
    match lst with
    | [] -> "empty list"
    | [Even x] -> sprintf "single even: %d" x
    | [Odd x] -> sprintf "single odd: %d" x
    | Even x :: rest -> sprintf "starts with even %d, has %d more" x (List.length rest)
    | Odd x :: rest -> sprintf "starts with odd %d, has %d more" x (List.length rest)

printfn "%s" (describeList [])
printfn "%s" (describeList [4])
printfn "%s" (describeList [3])
printfn "%s" (describeList [2; 3; 4])
printfn "%s" (describeList [1; 2; 3])
```

```fsharp
// Even/Odd ใน recursive function
let rec sumEvens = function
    | [] -> 0
    | Even x :: rest -> x + sumEvens rest
    | Odd _ :: rest -> sumEvens rest

let rec sumOdds = function
    | [] -> 0
    | Odd x :: rest -> x + sumOdds rest
    | Even _ :: rest -> sumOdds rest

let numbers = [1..10]
printfn "Sum of evens: %d" (sumEvens numbers)  // 2+4+6+8+10 = 30
printfn "Sum of odds: %d" (sumOdds numbers)    // 1+3+5+7+9 = 25
```

---

## 7. (|ParseInt|_|) Example

```fsharp
// ParseInt pattern: แปลง string เป็น int

let (|ParseInt|_|) (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

// Command interpreter
type Command =
    | Add of int * int
    | Multiply of int * int
    | Unknown of string

let parseCommand (parts: string list) =
    match parts with
    | ["add"; ParseInt x; ParseInt y] -> Add(x, y)
    | ["mul"; ParseInt x; ParseInt y] -> Multiply(x, y)
    | _ -> Unknown (String.concat " " parts)

let executeCommand = function
    | Add(x, y) -> printfn "%d + %d = %d" x y (x + y)
    | Multiply(x, y) -> printfn "%d × %d = %d" x y (x * y)
    | Unknown cmd -> printfn "Unknown command: %s" cmd

["add 3 4"; "mul 5 6"; "add 10 abc"; "divide 8 2"]
|> List.map (fun s -> s.Split(' ') |> Array.toList)
|> List.map parseCommand
|> List.iter executeCommand
```

---

## 8. Active Patterns for Parsing

```fsharp
// Active patterns สำหรับ parsing text

// Integer literal
let (|IntLiteral|_|) (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some (IntLiteral n)
    | _ -> None

// String literal (เริ่มและจบด้วย ")
let (|StringLiteral|_|) (s: string) =
    if s.StartsWith("\"") && s.EndsWith("\"") && s.Length >= 2 then
        Some (StringLiteral (s.[1..s.Length-2]))
    else None

// Boolean literal
let (|BoolLiteral|_|) = function
    | "true" -> Some (BoolLiteral true)
    | "false" -> Some (BoolLiteral false)
    | _ -> None

// Null literal
let (|NullLiteral|_|) = function
    | "null" -> Some ()
    | _ -> None

type Value =
    | VInt of int
    | VString of string
    | VBool of bool
    | VNull

let parseValue s =
    match s with
    | IntLiteral n -> VInt n
    | StringLiteral s -> VString s
    | BoolLiteral b -> VBool b
    | NullLiteral -> VNull
    | _ -> VString s  // treat as string

let testValues = ["42"; "\"hello\""; "true"; "false"; "null"; "someString"]
testValues |> List.iter (fun s -> printfn "%s -> %A" s (parseValue s))
```

```fsharp
// Parser สำหรับ key=value pairs
let (|KeyValue|_|) (s: string) =
    let parts = s.Split('=')
    if parts.Length = 2 then
        Some (parts.[0].Trim(), parts.[1].Trim())
    else None

// Config file parser
let parseConfig lines =
    lines
    |> List.choose (fun line ->
        match line with
        | KeyValue (k, v) -> Some (k, v)
        | _ -> None)
    |> Map.ofList

let configLines = [
    "host = localhost"
    "port = 8080"
    "# this is a comment"
    "timeout = 30"
    "invalid line"
]

let config = parseConfig configLines
printfn "Config: %A" config
```

---

## 9. Active Patterns for Validation

```fsharp
// Active patterns สำหรับ validation

// Non-empty string
let (|NonEmpty|_|) (s: string) =
    if System.String.IsNullOrWhiteSpace(s) then None
    else Some (s.Trim())

// Valid age
let (|ValidAge|_|) age =
    if age >= 0 && age <= 150 then Some age
    else None

// Positive number
let (|Positive|_|) n =
    if n > 0 then Some n else None

// Strong password (8+ chars, has uppercase, has digit)
let (|StrongPassword|_|) (s: string) =
    if s.Length >= 8 
       && s |> Seq.exists System.Char.IsUpper
       && s |> Seq.exists System.Char.IsDigit
    then Some s
    else None

// Validation pipeline
type UserInput = {
    Name: string
    Age: string
    Password: string
}

type ValidUser = {
    Name: string
    Age: int
    Password: string
}

let validateUser input =
    match input.Name, input.Age, input.Password with
    | NonEmpty name, ParseInt (ValidAge age), StrongPassword pwd ->
        Ok { Name = name; Age = age; Password = pwd }
    | _, _, _ when System.String.IsNullOrWhiteSpace(input.Name) ->
        Error "Name cannot be empty"
    | _, ParseInt (ValidAge _), _ | _, ParseInt _, _ ->
        Error "Invalid age"
    | _ ->
        Error "Password must be 8+ chars with uppercase and digit"

let testInputs = [
    { Name = "Alice"; Age = "25"; Password = "Secure1234" }
    { Name = ""; Age = "25"; Password = "Secure1234" }
    { Name = "Bob"; Age = "200"; Password = "Secure1234" }
    { Name = "Charlie"; Age = "30"; Password = "weakpwd" }
]

testInputs |> List.iter (fun input ->
    match validateUser input with
    | Ok user -> printfn "Valid: %s (age %d)" user.Name user.Age
    | Error msg -> printfn "Invalid: %s" msg)
```

---

## 10. Active Patterns for Categorization

```fsharp
// ใช้ active patterns เพื่อจัดหมวดหมู่ข้อมูล

// Categorize prices
let (|Cheap|Medium|Expensive|) price =
    if price < 100.0 then Cheap
    elif price < 1000.0 then Medium
    else Expensive

type Product = { Name: string; Price: float }

let products = [
    { Name = "Coffee"; Price = 45.0 }
    { Name = "Book"; Price = 250.0 }
    { Name = "Laptop"; Price = 35000.0 }
    { Name = "Pen"; Price = 15.0 }
    { Name = "Monitor"; Price = 8500.0 }
]

let categorizeProduct product =
    match product.Price with
    | Cheap -> sprintf "%s (Cheap: $%.0f)" product.Name product.Price
    | Medium -> sprintf "%s (Medium: $%.0f)" product.Name product.Price
    | Expensive -> sprintf "%s (Expensive: $%.0f)" product.Name product.Price

products |> List.iter (fun p -> printfn "%s" (categorizeProduct p))
```

```fsharp
// Categorize by type
type Animal =
    | Dog of string
    | Cat of string
    | Bird of string
    | Fish of string

let (|Mammal|NonMammal|) = function
    | Dog name -> Mammal name
    | Cat name -> Mammal name
    | Bird name -> NonMammal ("bird:" + name)
    | Fish name -> NonMammal ("fish:" + name)

let (|Vocal|Silent|) = function
    | Dog _ -> Vocal "woof"
    | Cat _ -> Vocal "meow"
    | Bird _ -> Vocal "tweet"
    | Fish _ -> Silent

let animals = [Dog "Rex"; Cat "Whiskers"; Bird "Tweety"; Fish "Nemo"]

animals |> List.iter (fun animal ->
    let sound =
        match animal with
        | Vocal s -> sprintf "says %s" s
        | Silent -> "is silent"
    
    let kind =
        match animal with
        | Mammal name -> sprintf "Mammal (%s)" name
        | NonMammal desc -> sprintf "Non-mammal (%s)" desc
    
    printfn "%s - %s" kind sound)
```

---

## 11. Composing Active Patterns

```fsharp
// รวม active patterns เข้าด้วยกัน

// Base patterns
let (|Positive|_|) n = if n > 0 then Some n else None
let (|Even|_|) n = if n % 2 = 0 then Some n else None
let (|Odd|_|) n = if n % 2 <> 0 then Some n else None
let (|LargerThan|_|) threshold n = if n > threshold then Some n else None

// Composed patterns
let (|PositiveEven|_|) n =
    match n with
    | Positive _ & Even _ -> Some n
    | _ -> None

let (|PositiveOdd|_|) n =
    match n with
    | Positive _ & Odd _ -> Some n
    | _ -> None

let classify n =
    match n with
    | PositiveEven n when n |> (LargerThan 10 |> Option.isSome << Some) -> sprintf "%d: big positive even" n
    | PositiveEven n -> sprintf "%d: positive even" n
    | PositiveOdd n -> sprintf "%d: positive odd" n
    | _ -> sprintf "%d: non-positive" n

[-2; -1; 0; 1; 2; 3; 10; 12; 15] |> List.iter (fun n -> printfn "%s" (classify n))
```

```fsharp
// Pattern composition ด้วย OR patterns
let (|SmallNumber|_|) n = if n >= 0 && n <= 9 then Some n else None
let (|TwoDigit|_|) n = if n >= 10 && n <= 99 then Some n else None
let (|ThreeDigit|_|) n = if n >= 100 && n <= 999 then Some n else None

let describeDigits n =
    match n with
    | SmallNumber d -> sprintf "%d has 1 digit" d
    | TwoDigit d -> sprintf "%d has 2 digits" d
    | ThreeDigit d -> sprintf "%d has 3 digits" d
    | _ -> sprintf "%d has 4+ digits" n

[3; 42; 567; 1234; 0] |> List.iter (fun n -> printfn "%s" (describeDigits n))
```

---

## 12. Performance Considerations

```fsharp
// Active patterns มี overhead บ้างเมื่อเทียบกับ direct pattern matching

// ตัวอย่าง: เปรียบเทียบ performance

open System.Diagnostics

let n = 10000000

// แบบ direct if/else
let sw1 = Stopwatch.StartNew()
let mutable sum1 = 0
for i in 1..n do
    if i % 2 = 0 then sum1 <- sum1 + i
sw1.Stop()

// แบบ active pattern
let (|E|O|) x = if x % 2 = 0 then E else O

let sw2 = Stopwatch.StartNew()
let mutable sum2 = 0
for i in 1..n do
    match i with
    | E -> sum2 <- sum2 + i
    | O -> ()
sw2.Stop()

printfn "Direct: %dms" sw1.ElapsedMilliseconds
printfn "Active pattern: %dms" sw2.ElapsedMilliseconds
printfn "sums equal: %b" (sum1 = sum2)

// ส่วนใหญ่ performance ต่างกันเล็กน้อย
// Active patterns เหมาะกับ code ที่อ่านง่ายกว่า performance-critical code
```

---

## 13. Real-World Examples

### 13.1 Command-Line Parser

```fsharp
// Active patterns สำหรับ parse command-line arguments

let (|Flag|_|) (flag: string) (arg: string) =
    if arg = flag then Some () else None

let (|Param|_|) (prefix: string) (arg: string) =
    if arg.StartsWith(prefix) then
        Some (arg.Substring(prefix.Length))
    else None

type Args = {
    Verbose: bool
    OutputFile: string option
    InputFile: string option
    Format: string option
}

let parseArgs (args: string list) =
    let mutable verbose = false
    let mutable output = None
    let mutable input = None
    let mutable format = None
    
    for arg in args do
        match arg with
        | Flag "--verbose" -> verbose <- true
        | Param "--output=" v -> output <- Some v
        | Param "--input=" v -> input <- Some v
        | Param "--format=" v -> format <- Some v
        | _ -> printfn "Unknown argument: %s" arg
    
    { Verbose = verbose; OutputFile = output; InputFile = input; Format = format }

let args = ["--verbose"; "--input=data.csv"; "--output=result.json"; "--format=json"]
let parsed = parseArgs args
printfn "Args: %A" parsed
```

### 13.2 Log Level Parser

```fsharp
// Active patterns สำหรับ parse log entries

type LogLevel = Debug | Info | Warning | Error | Critical

let (|LogLevel|_|) = function
    | "DEBUG" | "debug" -> Some Debug
    | "INFO" | "info" -> Some Info
    | "WARN" | "WARNING" | "warn" -> Some Warning
    | "ERROR" | "error" -> Some Error
    | "CRITICAL" | "FATAL" -> Some Critical
    | _ -> None

let (|LogEntry|_|) (line: string) =
    let parts = line.Split([|' '|], 3)
    if parts.Length = 3 then
        match parts.[1] with
        | LogLevel level ->
            match System.DateTime.TryParse(parts.[0]) with
            | true, ts -> Some (ts, level, parts.[2])
            | _ -> None
        | _ -> None
    else None

let sampleLogs = [
    "2024-01-01 INFO Application started"
    "2024-01-01 DEBUG Loading config"
    "2024-01-01 ERROR Database failed"
    "2024-01-01 WARN Memory low"
    "invalid log entry"
]

sampleLogs |> List.iter (fun line ->
    match line with
    | LogEntry (ts, level, msg) ->
        printfn "[%A] %A: %s" ts level msg
    | _ ->
        printfn "Unparseable: %s" line)
```

### 13.3 Pattern-Based Router

```fsharp
// URL router ด้วย active patterns

let (|Route|_|) (pattern: string) (url: string) =
    let patternParts = pattern.Split('/')
    let urlParts = url.TrimStart('/').Split('/')
    
    if patternParts.Length <> urlParts.Length then None
    else
        let rec matchParts ps us params =
            match ps, us with
            | [], [] -> Some (List.rev params)
            | p :: restP, u :: restU ->
                if p.StartsWith(":") then
                    matchParts restP restU ((p.[1..], u) :: params)
                elif p = u then
                    matchParts restP restU params
                else None
            | _ -> None
        
        matchParts (Array.toList patternParts) (Array.toList urlParts) []

let router url =
    match url with
    | Route "/users/:id" [("id", id)] ->
        sprintf "Get user %s" id
    | Route "/users/:id/posts/:postId" [("id", userId); ("postId", postId)] ->
        sprintf "Get post %s of user %s" postId userId
    | Route "/products" [] ->
        "List all products"
    | Route "/products/:name" [("name", name)] ->
        sprintf "Get product: %s" name
    | _ ->
        "404 Not Found"

let urls = ["/users/42"; "/users/10/posts/5"; "/products"; "/products/laptop"; "/unknown"]
urls |> List.iter (fun url -> printfn "%s -> %s" url (router url))
```

---

## สรุป (Summary)

```fsharp
// สรุป Active Patterns ใน F#

// 1. Single-case: แปลงค่า
let (|Trimmed|) (s: string) = s.Trim()

// 2. Multi-case: จัดกลุ่ม
let (|Pos|Neg|Zero|) n =
    if n > 0 then Pos elif n < 0 then Neg else Zero

// 3. Partial: match หรือไม่ match
let (|Int|_|) (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | _ -> None

// 4. Parameterized
let (|Prefix|_|) (prefix: string) (s: string) =
    if s.StartsWith(prefix) then Some (s.[prefix.Length..]) else None

// ใช้ร่วมกัน
let classify input =
    match input with
    | Int n ->
        match n with
        | Pos -> sprintf "positive: %d" n
        | Neg -> sprintf "negative: %d" n
        | Zero -> "zero"
    | Prefix "http" url -> sprintf "URL: http%s" url
    | Trimmed s -> sprintf "string: '%s'" s

let tests = ["42"; "-5"; "0"; "http://example.com"; "  hello  "]
tests |> List.iter (fun s -> printfn "%s -> %s" s (classify s))
```

Active Patterns ทำให้:
- **Code อ่านง่าย**: abstract ความซับซ้อนในการ match
- **Reusable**: ใช้ pattern เดียวกันได้หลายที่
- **Composable**: รวม patterns เข้าด้วยกันได้
- **Domain-specific**: สร้าง patterns ที่สื่อความหมาย domain
