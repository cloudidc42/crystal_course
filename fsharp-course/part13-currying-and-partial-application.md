# Part 13 - เคอร์รี่และการประยุกต์ใช้บางส่วน (Currying and Partial Application)

## บทนำ (Introduction)

**Currying** คือการแปลงฟังก์ชันที่รับ argument หลายตัวพร้อมกัน ให้เป็นฟังก์ชันที่รับ argument ทีละตัว
**Partial Application** คือการส่ง argument บางส่วนให้กับฟังก์ชัน เพื่อสร้างฟังก์ชันใหม่ที่รับ argument ที่เหลือ

ใน F# ฟังก์ชันทุกตัวถูก curry โดยอัตโนมัติ!

---

## 1. What is Currying? (Currying คืออะไร)

```fsharp
// ใน math: f(x, y) = x + y
// แบบ curried: f(x) = g ที่ g(y) = x + y

// ใน F# ฟังก์ชันทุกตัวเป็น curried โดยอัตโนมัติ
let add x y = x + y

// Type ของ add คือ: int -> int -> int
// อ่านว่า: รับ int แล้วส่งคืน (int -> int)

// ดังนั้น add 3 คือฟังก์ชัน int -> int ที่ "จดจำ" ค่า 3
let add3 = add 3   // Type: int -> int

printfn "add 3 4 = %d" (add 3 4)    // 7
printfn "add3 4 = %d" (add3 4)      // 7
printfn "add3 10 = %d" (add3 10)    // 13
```

```fsharp
// แสดงให้เห็นว่า currying ทำงานอย่างไร

// ฟังก์ชัน 3 argument
let addThree x y z = x + y + z

// Type: int -> int -> int -> int

// ใช้ทีละขั้น
let step1 = addThree 1       // int -> int -> int
let step2 = step1 2          // int -> int
let step3 = step2 3          // int

printfn "step3 = %d" step3  // 6

// หรือเต็มรูปแบบ
let result = addThree 1 2 3
printfn "result = %d" result  // 6
```

---

## 2. F# Automatic Currying (F# Curry อัตโนมัติ)

```fsharp
// F# curry ฟังก์ชันทุกตัวโดยอัตโนมัติ
// ทุกการเรียกใช้ฟังก์ชันเป็น partial application ได้

let multiply x y = x * y
// Type: int -> int -> int

// Partial application
let double = multiply 2   // int -> int
let triple = multiply 3   // int -> int
let times10 = multiply 10 // int -> int

let numbers = [1; 2; 3; 4; 5]

printfn "doubles: %A" (List.map double numbers)  // [2; 4; 6; 8; 10]
printfn "triples: %A" (List.map triple numbers)  // [3; 6; 9; 12; 15]
printfn "times10: %A" (List.map times10 numbers) // [10; 20; 30; 40; 50]
```

```fsharp
// ตรวจสอบ type ของ curried functions
// ใช้ type annotation เพื่อเข้าใจ

let add : int -> int -> int = fun x y -> x + y
let add' : int -> (int -> int) = fun x -> (fun y -> x + y)

// ทั้งสองเหมือนกัน!
printfn "%d" (add 3 4)   // 7
printfn "%d" (add' 3 4)  // 7
```

```fsharp
// F# ทำ automatic currying ให้เองเสมอ
// ไม่ต้องใช้ nested lambdas

let addXYZ x y z = x + y + z

// เหมือนกับ:
let addXYZ' = fun x -> fun y -> fun z -> x + y + z

printfn "%d" (addXYZ 1 2 3)   // 6
printfn "%d" (addXYZ' 1 2 3)  // 6
```

---

## 3. Partial Application Examples

```fsharp
// Partial application: ส่ง argument บางส่วน

// String operations
let contains (substr: string) (s: string) = s.Contains(substr)
let startsWith (prefix: string) (s: string) = s.StartsWith(prefix)
let endsWith (suffix: string) (s: string) = s.EndsWith(suffix)

// Partial application สร้างฟังก์ชันเฉพาะ
let containsHello = contains "Hello"
let startsWithhttp = startsWith "http"
let endsWithcs = endsWith ".cs"

let strings = ["Hello World"; "http://example.com"; "Program.cs"; "HelloWorld.cs"]

let greetings = List.filter containsHello strings
let urls = List.filter startsWithhttp strings
let csharpFiles = List.filter endsWithcs strings

printfn "greetings: %A" greetings
printfn "urls: %A" urls
printfn "csharpFiles: %A" csharpFiles
```

```fsharp
// Partial application กับ numeric operations
let between min max x = x >= min && x <= max
let clamp min max x = if x < min then min elif x > max then max else x

let isValidAge = between 0 150
let isValidScore = between 0 100
let clampToUByte = clamp 0 255

let ages = [-5; 0; 25; 100; 150; 200]
let validAges = List.filter isValidAge ages

let scores = [-10; 0; 50; 100; 150]
let validScores = List.filter isValidScore scores

let colorValues = [-50; 0; 128; 200; 300]
let clampedColors = List.map clampToUByte colorValues

printfn "validAges: %A" validAges
printfn "validScores: %A" validScores
printfn "clampedColors: %A" clampedColors
```

```fsharp
// Partial application กับ sorting
type Product = { Name: string; Price: float; Category: string }

let products = [
    { Name = "Laptop"; Price = 999.99; Category = "Electronics" }
    { Name = "Book"; Price = 15.99; Category = "Education" }
    { Name = "Headphones"; Price = 199.99; Category = "Electronics" }
    { Name = "Pen"; Price = 2.99; Category = "Stationery" }
]

let sortBy field = List.sortBy field
let sortByDescending field = List.sortByDescending field
let filterByCategory cat = List.filter (fun p -> p.Category = cat)

// สร้าง specialized functions
let sortByPrice = sortBy (fun p -> p.Price)
let sortByName = sortBy (fun p -> p.Name)
let sortByPriceDesc = sortByDescending (fun p -> p.Price)
let getElectronics = filterByCategory "Electronics"

printfn "By price: %A" (sortByPrice products |> List.map (fun p -> p.Name))
printfn "By name: %A" (sortByName products |> List.map (fun p -> p.Name))
printfn "Electronics: %A" (getElectronics products |> List.map (fun p -> p.Name))
```

---

## 4. Curried vs Tupled Functions

```fsharp
// Curried function: รับ argument ทีละตัว (F# default)
let addCurried x y = x + y
// Type: int -> int -> int

// Tupled function: รับ tuple เป็น argument เดียว
let addTupled (x, y) = x + y
// Type: int * int -> int

// ใช้งานต่างกัน
let r1 = addCurried 3 4     // 7
let r2 = addTupled (3, 4)   // 7

// Partial application ทำได้กับ curried แต่ไม่ได้กับ tupled
let add3 = addCurried 3   // ทำได้
// let add3' = addTupled 3  // ERROR: ต้องส่ง tuple
```

```fsharp
// แปลงระหว่าง curried และ tupled

// curry: แปลง tupled เป็น curried
let curry f x y = f (x, y)

// uncurry: แปลง curried เป็น tupled
let uncurry f (x, y) = f x y

// ตัวอย่าง
let addTupled' (x, y) = x + y
let addCurried' = curry addTupled'

let r1 = addTupled' (3, 4)    // 7
let r2 = addCurried' 3 4      // 7

// partial application หลังจาก curry
let add10 = addCurried' 10
let r3 = add10 5   // 15
```

```fsharp
// เมื่อไหรควรใช้ tupled?
// 1. ต้องการส่งผ่าน pair ข้อมูล
// 2. เมื่อ pair มีความหมายเป็นหน่วยเดียวกัน
// 3. Interop กับ .NET methods

// เช่น Point เป็น tuple ที่มีความหมาย
let distance (x1, y1) (x2, y2) =
    let dx = x2 - x1
    let dy = y2 - y1
    sqrt (float (dx*dx + dy*dy))

let p1 = (0, 0)
let p2 = (3, 4)
let d = distance p1 p2
printfn "distance = %f" d  // 5.0
```

---

## 5. Creating Specialized Functions via Partial Application

```fsharp
// สร้าง domain-specific functions จาก general functions

// General logger
let log (level: string) (timestamp: System.DateTime) (message: string) =
    sprintf "[%s] %s: %s" (timestamp.ToString("HH:mm:ss")) level message

// Specialized loggers
let logInfo = log "INFO"
let logWarning = log "WARNING"
let logError = log "ERROR"

let now = System.DateTime.Now
printfn "%s" (logInfo now "Application started")
printfn "%s" (logWarning now "Memory usage high")
printfn "%s" (logError now "Connection failed")
```

```fsharp
// Database query builder (conceptual)
let query table condition limit =
    sprintf "SELECT * FROM %s WHERE %s LIMIT %d" table condition limit

// Partial application เพื่อสร้าง query functions เฉพาะ
let queryUsers = query "users"
let queryProducts = query "products"

let activeUsers = queryUsers "active = 1" 100
let allProducts = queryProducts "1=1" 50
let expensiveProducts = queryProducts "price > 100" 20

printfn "%s" activeUsers
printfn "%s" allProducts
printfn "%s" expensiveProducts
```

```fsharp
// Config-based functions
let sendEmail (host: string) (port: int) (from: string) (to_: string) (subject: string) (body: string) =
    sprintf "Sending email via %s:%d from %s to %s: [%s] %s" host port from to_ subject body

// ใช้ config จาก environment
let mailConfig = sendEmail "smtp.example.com" 587 "noreply@example.com"

// Partial application สำหรับ notification
let sendNotification = mailConfig "admin@example.com" "Notification"
let sendAlert = mailConfig "ops@example.com" "ALERT"

printfn "%s" (sendNotification "System is running normally")
printfn "%s" (sendAlert "Database connection pool exhausted!")
```

---

## 6. Practical Examples

### 6.1 Data Transformation

```fsharp
// Pipeline using partial application

type Employee = {
    Id: int
    Name: string
    Department: string
    Salary: float
    YearsOfService: int
}

let employees = [
    { Id = 1; Name = "Alice"; Department = "Engineering"; Salary = 85000.0; YearsOfService = 5 }
    { Id = 2; Name = "Bob"; Department = "Marketing"; Salary = 65000.0; YearsOfService = 3 }
    { Id = 3; Name = "Charlie"; Department = "Engineering"; Salary = 95000.0; YearsOfService = 8 }
    { Id = 4; Name = "Diana"; Department = "HR"; Salary = 70000.0; YearsOfService = 2 }
    { Id = 5; Name = "Eve"; Department = "Engineering"; Salary = 90000.0; YearsOfService = 6 }
]

// Reusable partial-applied functions
let getByDept dept = List.filter (fun e -> e.Department = dept)
let getAboveSalary sal = List.filter (fun e -> e.Salary > sal)
let getAboveYears years = List.filter (fun e -> e.YearsOfService >= years)
let sortBySalary = List.sortByDescending (fun e -> e.Salary)
let takeSenior = List.filter (fun e -> e.YearsOfService > 4)

// Compose ฟังก์ชันเหล่านี้
let engineeringTeam = employees |> getByDept "Engineering"
let seniorEngineers = engineeringTeam |> takeSenior
let topEarners = employees |> getAboveSalary 80000.0 |> sortBySalary

printfn "Engineering Team:"
engineeringTeam |> List.iter (fun e -> printfn "  %s" e.Name)

printfn "Senior Engineers (5+ years):"
seniorEngineers |> List.iter (fun e -> printfn "  %s (%d years)" e.Name e.YearsOfService)

printfn "Top Earners (>$80k):"
topEarners |> List.iter (fun e -> printfn "  %s: $%.0f" e.Name e.Salary)
```

### 6.2 Configuration Pattern

```fsharp
// Configuration-based partial application

type DatabaseConfig = {
    Host: string
    Port: int
    Database: string
    Username: string
}

let connect (config: DatabaseConfig) (query: string) =
    sprintf "Connecting to %s:%d/%s as %s and running: %s"
        config.Host config.Port config.Database config.Username query

let devConfig = { Host = "localhost"; Port = 5432; Database = "myapp_dev"; Username = "dev_user" }
let prodConfig = { Host = "db.example.com"; Port = 5432; Database = "myapp_prod"; Username = "app_user" }

// Partial application: bind config
let devQuery = connect devConfig
let prodQuery = connect prodConfig

printfn "%s" (devQuery "SELECT * FROM users")
printfn "%s" (prodQuery "SELECT * FROM orders")
```

### 6.3 Formatting Library

```fsharp
// สร้าง formatting library ด้วย partial application

let formatNumber (decimals: int) (thousandSep: bool) (n: float) =
    let formatted = sprintf "%.*f" decimals n
    if thousandSep then
        let parts = formatted.Split('.')
        let intPart = parts.[0]
        let withCommas =
            intPart
            |> Seq.rev
            |> Seq.chunkBySize 3
            |> Seq.map (fun chunk -> System.String(Array.ofSeq chunk))
            |> String.concat ","
            |> (fun s -> System.String(s.ToCharArray() |> Array.rev))
        if parts.Length > 1 then withCommas + "." + parts.[1]
        else withCommas
    else formatted

let formatCurrency = formatNumber 2 true
let formatPercentage = formatNumber 1 false
let formatInteger n = formatNumber 0 true n

printfn "$%s" (formatCurrency 1234567.89)  // $1,234,567.89
printfn "%s%%" (formatPercentage 0.8765)   // 0.9%
printfn "%s" (formatInteger 42000.0)        // 42,000
```

---

## 7. flip for Reordering Arguments

```fsharp
// flip: สลับ argument แรกและที่สองของ curried function
let flip f x y = f y x

// ตัวอย่างการใช้
let subtract x y = x - y
let divide x y = x / y
let modulo x y = x % y

// แบบปกติ
let sub5from = subtract 5   // subtract 5 y = 5 - y

// แบบ flip
let subtractFrom5 = flip subtract 5  // flip subtract 5 y = subtract y 5 = y - 5

printfn "subtract 10 3 = %d" (subtract 10 3)       // 7
printfn "subtractFrom5 10 = %d" (subtractFrom5 10) // 5
```

```fsharp
// flip ใช้ได้ดีกับ pipeline

// ปัญหา: List.filter ต้องการ (predicate, list) แต่เราอยากส่ง list ก่อน
let isEven x = x % 2 = 0
let numbers = [1..10]

// แบบปกติ
let evens = List.filter isEven numbers

// แบบใช้ flip เพื่อให้ list อยู่ทางซ้าย
let filterFlipped = flip List.filter
let evens2 = filterFlipped numbers isEven

printfn "evens: %A" evens
printfn "evens2: %A" evens2
```

```fsharp
// ตัวอย่าง flip กับ string operations
let replace (oldVal: string) (newVal: string) (s: string) = s.Replace(oldVal, newVal)
let split (sep: char) (s: string) = s.Split(sep) |> Array.toList

let replaceSpaceWithUnderscore = replace " " "_"
let splitByComma = split ','

let sentence = "Hello World foo bar"
let csv = "a,b,c,d,e"

printfn "%s" (replaceSpaceWithUnderscore sentence)  // "Hello_World_foo_bar"
printfn "%A" (splitByComma csv)                      // ["a"; "b"; "c"; "d"; "e"]
```

---

## 8. When to Use Partial Application vs Lambda

```fsharp
// เมื่อไหรควรใช้ partial application?
// 1. เมื่อฟังก์ชันมี argument หลายตัวและต้องการ "fix" บางตัว
// 2. เมื่อต้องการสร้าง specialized function

// เมื่อไหรควรใช้ lambda?
// 1. เมื่อต้องการ logic เพิ่มเติมที่ไม่ใช่แค่ partial application
// 2. เมื่อต้องการ pattern matching ใน argument

// Partial application: ชัดเจน สั้น
let add10 = (+) 10
let numbers = List.map add10 [1..5]  // [11; 12; 13; 14; 15]

// Lambda: จำเป็นเมื่อมี logic เพิ่มเติม
let complexOperation = List.map (fun x -> if x > 3 then x * 2 else x + 10) [1..5]

printfn "add10: %A" numbers
printfn "complexOperation: %A" complexOperation
```

```fsharp
// เปรียบเทียบ: partial application vs lambda

let data = ["hello"; "world"; "foo"; "bar"; "baz"]

// ด้วย partial application (ชัดเจนกว่า)
let startWithH = List.filter (fun s -> s.StartsWith("h"))
let hasLength3 = List.filter (fun s -> s.Length = 3)

// ด้วย lambda (บางครั้งจำเป็น)
let startsWithAndLength prefix len =
    List.filter (fun s -> s.StartsWith(prefix) && s.Length = len)

let result1 = startWithH data
let result2 = hasLength3 data
let result3 = startsWithAndLength "b" 3 data

printfn "startWithH: %A" result1
printfn "hasLength3: %A" result2
printfn "startsWithB_len3: %A" result3
```

```fsharp
// Guidelines ในการเลือก

// ใช้ partial application เมื่อ:
// - ส่ง argument เดียวกันซ้ำๆ
let processWithSameConfig config = List.map (processItem config)

// ใช้ lambda เมื่อ:
// - ต้องการ pattern matching
let classify = List.map (function
    | x when x > 0 -> "positive"
    | x when x < 0 -> "negative"
    | _ -> "zero")

// ใช้ทั้งสองเมื่อ:
// - ต้องการ partial application แต่ argument ไม่ได้ match กับ order ที่ต้องการ
let filterByMinLength min = List.filter (fun s -> String.length s >= min)
```

---

## 9. Building Domain-Specific Combinators

```fsharp
// Combinator: ฟังก์ชันที่รวม functions เข้าด้วยกัน

// Predicate combinators
let andAlso f g x = f x && g x    // AND
let orElse f g x = f x || g x     // OR
let negate f x = not (f x)         // NOT

let isPositive x = x > 0
let isEven x = x % 2 = 0
let isLarge x = x > 100

let isPositiveEven = andAlso isPositive isEven
let isPositiveOrLarge = orElse isPositive isLarge
let isNegativeOrZero = negate isPositive

let numbers = [-5; -2; 0; 1; 2; 3; 50; 101; 200]

printfn "positiveEven: %A" (List.filter isPositiveEven numbers)
printfn "positiveOrLarge: %A" (List.filter isPositiveOrLarge numbers)
printfn "negativeOrZero: %A" (List.filter isNegativeOrZero numbers)
```

```fsharp
// Transformer combinators

// on: apply f then compare/transform with g
let on (g: 'a -> 'b) (f: 'b -> 'b -> 'c) x y = f (g x) (g y)

// comparing on a property
let compareBy proj x y = compare (proj x) (proj y)
let sortByLength = List.sortWith (compareBy String.length)
let sortByFirstChar = List.sortWith (compareBy (fun (s: string) -> s.[0]))

let words = ["banana"; "apple"; "cherry"; "date"; "elderberry"]

printfn "byLength: %A" (sortByLength words)
printfn "byFirstChar: %A" (sortByFirstChar words)
```

```fsharp
// Option combinators
let bindOption f = Option.bind f
let mapOption f = Option.map f
let defaultOption defaultVal = Option.defaultValue defaultVal

let tryDivide (x: float) y =
    if y = 0.0 then None else Some (x / y)

let trySquareRoot x =
    if x < 0.0 then None else Some (sqrt x)

// Chain operations that might fail
let safeOperation x y =
    tryDivide x y
    |> Option.bind trySquareRoot
    |> Option.map (fun r -> sprintf "Result: %.4f" r)
    |> Option.defaultValue "Error: invalid operation"

printfn "%s" (safeOperation 16.0 4.0)   // "Result: 2.0000"
printfn "%s" (safeOperation 16.0 0.0)   // "Error: invalid operation"
printfn "%s" (safeOperation -16.0 4.0)  // "Error: invalid operation"
```

```fsharp
// Result combinators
let bindResult f = Result.bind f
let mapResult f = Result.map f
let mapError f = Result.mapError f

type ParseError =
    | InvalidFormat of string
    | OutOfRange of string

let parseInt (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Ok n
    | false, _ -> Error (InvalidFormat (sprintf "'%s' is not a valid integer" s))

let validateRange min max n =
    if n >= min && n <= max then Ok n
    else Error (OutOfRange (sprintf "%d is not in range [%d, %d]" n min max))

let parseAndValidateAge s =
    parseInt s
    |> Result.bind (validateRange 0 150)
    |> Result.map (fun age -> sprintf "Valid age: %d" age)

let testInputs = ["25"; "abc"; "-5"; "200"; "42"]
testInputs
|> List.map (fun s -> sprintf "%s -> %A" s (parseAndValidateAge s))
|> List.iter (printfn "%s")
```

---

## 10. Advanced Partial Application Patterns

```fsharp
// การใช้ partial application เพื่อ configure behaviors

// Logger with levels
type LogLevel = Debug | Info | Warning | Error

let createLogger (minLevel: LogLevel) (output: string -> unit) (level: LogLevel) (msg: string) =
    let levels = [Debug; Info; Warning; Error]
    let minIdx = List.findIndex ((=) minLevel) levels
    let currIdx = List.findIndex ((=) level) levels
    if currIdx >= minIdx then
        output (sprintf "[%A] %s" level msg)

// สร้าง loggers ต่างๆ
let consoleLogger = createLogger Info (printfn "%s")
let verboseLogger = createLogger Debug (printfn "%s")
let silentLogger = createLogger Error (printfn "%s")

consoleLogger Debug "This won't show"
consoleLogger Info "Application started"
consoleLogger Warning "Memory usage high"
consoleLogger Error "Critical error!"

printfn "---"
verboseLogger Debug "This will show"
```

```fsharp
// Builder pattern with partial application

// HTTP request builder
let createRequest (method: string) (baseUrl: string) (path: string) (headers: (string * string) list) (body: string option) =
    {|
        Method = method
        Url = baseUrl + path
        Headers = headers
        Body = body
    |}

// Partial application
let getRequest = createRequest "GET"
let postRequest = createRequest "POST"
let apiRequest = getRequest "https://api.example.com"
let apiPost = postRequest "https://api.example.com"

let standardHeaders = [
    ("Content-Type", "application/json")
    ("Authorization", "Bearer token123")
]

let getUserRequest = apiRequest "/users" standardHeaders None
let createUserRequest = apiPost "/users" standardHeaders (Some """{"name":"Alice"}""")

printfn "GET: %A" getUserRequest
printfn "POST: %A" createUserRequest
```

---

## สรุป (Summary)

```fsharp
// สรุป: Currying and Partial Application ใน F#

// 1. F# curry อัตโนมัติ
let multiply x y = x * y          // int -> int -> int
let double = multiply 2            // int -> int (partial application)

// 2. Partial application สร้าง specialized functions
let add5 = (+) 5                   // int -> int
let isEven = (%) 2 >> (=) 0       // int -> bool (ระวัง: ต้องระมัดระวัง argument order)
let isEven' n = n % 2 = 0         // ชัดเจนกว่า

// 3. flip สลับ argument
let flip f x y = f y x
let divideBy2 = flip (/) 2        // int -> int

// 4. curry/uncurry แปลงระหว่าง tupled และ curried
let curry f x y = f (x, y)
let uncurry f (x, y) = f x y

// 5. สร้าง domain combinators
let both f g x = f x && g x
let either f g x = f x || g x

// ตัวอย่างรวม
let numbers = [1..20]
let isValidScore = both (fun n -> n >= 1) (fun n -> n <= 20)
let highScores = both (fun n -> n >= 15) isValidScore

let validNums = List.filter isValidScore numbers
let highNums = List.filter highScores numbers
let doubled = List.map (multiply 2) validNums

printfn "valid: %A" validNums
printfn "high: %A" highNums
printfn "doubled: %A" doubled
```

Currying และ Partial Application เป็นเครื่องมือทรงพลังที่ทำให้:
- **Code reuse**: สร้างฟังก์ชัน specialized จาก general functions
- **Readability**: โค้ดอ่านง่ายขึ้นด้วย named functions
- **Composability**: รวมกับ function composition ได้ดี
- **Testability**: ทดสอบง่ายกว่าเพราะแต่ละ function ทำงานเดียว
