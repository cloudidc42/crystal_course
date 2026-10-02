# Part 12 - การประกอบฟังก์ชัน (Function Composition)

## บทนำ (Introduction)

Function composition คือการรวมฟังก์ชันหลายตัวเข้าด้วยกัน เพื่อสร้างฟังก์ชันใหม่ที่ทำงานซับซ้อนขึ้น ใน F# มี operators หลายตัวสำหรับ composition และ piping

---

## 1. >> (Forward Composition Operator)

```fsharp
// >> คือ forward composition
// f >> g หมายถึง: รับ input -> ส่งให้ f -> ส่งผลลัพธ์ให้ g -> ได้ output

// ตัวอย่างพื้นฐาน
let add1 x = x + 1
let double x = x * 2
let square x = x * x

let add1ThenDouble = add1 >> double    // x -> double(add1(x))
let doubleAndSquare = double >> square  // x -> square(double(x))

printfn "add1ThenDouble 3 = %d" (add1ThenDouble 3)   // double(4) = 8
printfn "doubleAndSquare 3 = %d" (doubleAndSquare 3) // square(6) = 36
```

```fsharp
// ตรวจสอบ type ของ composed function
// add1: int -> int
// double: int -> int
// add1 >> double: int -> int

// ตัวอย่างกับ type ที่ต่างกัน
let toInt (s: string) = int s
let toHex (n: int) = sprintf "0x%X" n

let stringToHex = toInt >> toHex
// stringToHex: string -> string

printfn "%s" (stringToHex "255")  // "0xFF"
printfn "%s" (stringToHex "16")   // "0x10"
printfn "%s" (stringToHex "0")    // "0x0"
```

```fsharp
// ซ้อน composition หลายชั้น
let trim (s: string) = s.Trim()
let toLower (s: string) = s.ToLower()
let removeSpaces (s: string) = s.Replace(" ", "_")
let addPrefix (s: string) = "user_" + s

let normalize = trim >> toLower >> removeSpaces >> addPrefix

printfn "%s" (normalize "  Alice Smith  ") // "user_alice_smith"
printfn "%s" (normalize " Bob Jones ")     // "user_bob_jones"
```

```fsharp
// composition กับ List functions
let filterPositive = List.filter (fun x -> x > 0)
let squareAll = List.map (fun x -> x * x)
let sumAll = List.sum

let sumOfSquaredPositives = filterPositive >> squareAll >> sumAll

let data = [-3; 1; -2; 4; -5; 2; 3]
let result = sumOfSquaredPositives data
printfn "sumOfSquaredPositives: %d" result  // 1+16+4+9 = 30
```

---

## 2. << (Backward Composition Operator)

```fsharp
// << คือ backward composition
// f << g หมายถึง: รับ input -> ส่งให้ g -> ส่งผลลัพธ์ให้ f -> ได้ output
// (f << g) x = f(g(x))

let add1 x = x + 1
let double x = x * 2

let doubleFirst = double << add1  // x -> double(add1(x))

// เหมือนกันกับ add1 >> double
let forwardComp = add1 >> double
let backwardComp = double << add1

printfn "forward 3 = %d" (forwardComp 3)   // 8
printfn "backward 3 = %d" (backwardComp 3)  // 8
```

```fsharp
// << มีประโยชน์เมื่ออยากอ่านเหมือน math: f ∘ g
// ใน math: (f ∘ g)(x) = f(g(x))

let negate x = -x
let abs x = abs x

// |-x| = abs(negate(x))
let absNeg = abs << negate

printfn "absNeg 5 = %d" (absNeg 5)   // 5
printfn "absNeg -3 = %d" (absNeg -3) // 3
```

```fsharp
// เลือกใช้ >> หรือ << ตามความชัดเจนของโค้ด

// แบบ pipeline (>> ชัดเจนกว่า)
let process1 = 
    List.filter (fun x -> x > 0)
    >> List.map float
    >> List.average

// แบบ mathematical composition (<< อ่านได้ดีในบางกรณี)
let average = List.average << List.map float << List.filter (fun x -> x > 0)

let data = [-1; 2; 3; -4; 5]
printfn "process1: %f" (process1 data)  // avg of [2.0; 3.0; 5.0] = 3.333...
printfn "average: %f" (average data)    // same
```

---

## 3. |> (Forward Pipe Operator)

```fsharp
// |> ส่งค่าไปเป็น argument สุดท้ายของฟังก์ชัน
// x |> f  เหมือนกับ f x

// แบบไม่ใช้ pipe
let result1 = List.sum (List.map (fun x -> x * 2) (List.filter (fun x -> x > 3) [1; 2; 3; 4; 5]))

// แบบใช้ pipe (อ่านง่ายกว่ามาก)
let result2 =
    [1; 2; 3; 4; 5]
    |> List.filter (fun x -> x > 3)
    |> List.map (fun x -> x * 2)
    |> List.sum

printfn "result1 = %d" result1  // 18
printfn "result2 = %d" result2  // 18
```

```fsharp
// |> กับฟังก์ชันหลาย argument
// ค่าจะถูกส่งเป็น argument สุดท้าย

let divide x y = x / y  // ลำดับ argument สำคัญ!

// 10 |> divide 2 หมายถึง divide 2 10 = 2/10 ไม่ใช่ 10/2!
// ระวัง: |> ส่งค่าเป็น argument ตัวสุดท้าย

let result = 10 |> divide 2  // divide 2 10 = 0 (integer division 2/10)
printfn "result = %d" result   // 0

// ถ้าต้องการ 10/2 = 5
let divideBy divisor dividend = dividend / divisor
let result2 = 10 |> divideBy 2  // divideBy 2 10 = 10/2 = 5
printfn "result2 = %d" result2
```

```fsharp
// ตัวอย่างการใช้งานจริง
type Person = { Name: string; Age: int; Score: float }

let people = [
    { Name = "Alice"; Age = 28; Score = 95.0 }
    { Name = "Bob"; Age = 35; Score = 72.5 }
    { Name = "Charlie"; Age = 22; Score = 88.0 }
    { Name = "Diana"; Age = 30; Score = 91.5 }
    { Name = "Eve"; Age = 25; Score = 65.0 }
]

let result =
    people
    |> List.filter (fun p -> p.Age >= 25)
    |> List.sortByDescending (fun p -> p.Score)
    |> List.take 3
    |> List.map (fun p -> sprintf "%s: %.1f" p.Name p.Score)

result |> List.iter (printfn "%s")
```

---

## 4. <| (Backward Pipe Operator)

```fsharp
// <| ส่งค่าไปเป็น argument ของฟังก์ชันทางซ้าย
// f <| x  เหมือนกับ f x

// ลดวงเล็บในบางกรณี
let print = printfn "%A"

// แบบปกติ
print (List.map (fun x -> x * 2) [1; 2; 3])

// แบบใช้ <|
print <| List.map (fun x -> x * 2) [1; 2; 3]
```

```fsharp
// <| มีประโยชน์เมื่อ argument ยาวหรือซับซ้อน
let assertsTrue msg condition =
    if not condition then failwith msg

// แบบไม่ใช้ <|
assertsTrue "Sum should be 15" (List.sum [1..5] = 15)

// แบบใช้ <|
assertsTrue "Sum should be 15" <| (List.sum [1..5] = 15)
```

```fsharp
// เปรียบเทียบ |> และ <|
let isPositive x = x > 0

// |> ส่งค่าไปทางขวา
let r1 = 5 |> isPositive    // isPositive 5 = true

// <| ส่งค่าไปทางซ้าย
let r2 = isPositive <| 5    // isPositive 5 = true

printfn "%b %b" r1 r2  // true true
```

---

## 5. ||> (Two-Argument Pipe)

```fsharp
// ||> ส่ง tuple 2 ค่าเป็น argument ของฟังก์ชัน
// (x, y) ||> f  เหมือนกับ f x y

let add x y = x + y
let subtract x y = x - y

let r1 = (3, 4) ||> add       // add 3 4 = 7
let r2 = (10, 3) ||> subtract  // subtract 10 3 = 7

printfn "r1 = %d" r1
printfn "r2 = %d" r2
```

```fsharp
// ||> ใช้งานจริง
let divide (x: float) y = x / y
let power x y = System.Math.Pow(x, y)

let r1 = (10.0, 2.0) ||> divide   // 5.0
let r2 = (2.0, 8.0) ||> power     // 256.0

printfn "10.0 / 2.0 = %f" r1
printfn "2.0 ^ 8.0 = %f" r2

// ใช้กับ List functions
let list1 = [1; 2; 3]
let list2 = [4; 5; 6]

let result = (list1, list2) ||> List.zip
printfn "zip: %A" result  // [(1,4); (2,5); (3,6)]

let result2 = (list1, list2) ||> List.map2 (+)
printfn "map2 (+): %A" result2  // [5; 7; 9]
```

---

## 6. |||> (Three-Argument Pipe)

```fsharp
// |||> ส่ง tuple 3 ค่าเป็น argument ของฟังก์ชัน
// (x, y, z) |||> f  เหมือนกับ f x y z

let clamp minVal maxVal value =
    if value < minVal then minVal
    elif value > maxVal then maxVal
    else value

let r1 = (0, 10, 5) |||> clamp   // 5 (within range)
let r2 = (0, 10, -3) |||> clamp  // 0 (below min)
let r3 = (0, 10, 15) |||> clamp  // 10 (above max)

printfn "clamp [0,10] 5 = %d" r1
printfn "clamp [0,10] -3 = %d" r2
printfn "clamp [0,10] 15 = %d" r3
```

```fsharp
// |||> กับ List functions
let fold3 =
    (0, (+), [1; 2; 3; 4; 5]) |||> List.fold

// คือ List.fold (+) 0 [1; 2; 3; 4; 5] = 15
printfn "fold3 = %d" fold3
```

---

## 7. Building Pipelines (การสร้าง Pipeline)

```fsharp
// Pipeline for data processing

// Step functions แต่ละขั้น
let parseNumber (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

let filterPositive (n: int option) =
    match n with
    | Some n when n > 0 -> Some n
    | _ -> None

let doubleNumber (n: int option) =
    Option.map (fun x -> x * 2) n

let formatResult (n: int option) =
    match n with
    | Some n -> sprintf "Result: %d" n
    | None -> "Invalid input"

// Compose ทุก step เป็น pipeline เดียว
let processInput = parseNumber >> filterPositive >> doubleNumber >> formatResult

// Test
let inputs = ["42"; "-5"; "abc"; "10"; "0"; "7"]
inputs |> List.map processInput |> List.iter (printfn "%s")
```

```fsharp
// Pipeline สำหรับ text processing
let splitWords (s: string) = s.Split([|' '; '\t'; '\n'|], System.StringSplitOptions.RemoveEmptyEntries)
let filterLongWords minLen (words: string[]) = words |> Array.filter (fun w -> w.Length >= minLen)
let countWords (words: string[]) = words.Length
let formatCount count = sprintf "Word count: %d" count

// ใช้ partial application เพื่อ configure
let wordCounter minWordLength =
    splitWords
    >> filterLongWords minWordLength
    >> countWords
    >> formatCount

let countLongWords = wordCounter 5

let text = "The quick brown fox jumps over the lazy dog"
printfn "%s" (countLongWords text)  // Words with 5+ chars
```

```fsharp
// Pipeline สำหรับ validation
type ValidationResult<'a> =
    | Valid of 'a
    | Invalid of string list

let validateNotEmpty (s: string) =
    if System.String.IsNullOrWhiteSpace(s) then
        Invalid ["Cannot be empty"]
    else
        Valid s

let validateEmail (result: ValidationResult<string>) =
    match result with
    | Invalid _ -> result
    | Valid s ->
        if s.Contains("@") && s.Contains(".")
        then Valid s
        else Invalid ["Must be a valid email address"]

let validateLength min max (result: ValidationResult<string>) =
    match result with
    | Invalid _ -> result
    | Valid s ->
        if s.Length < min then Invalid [sprintf "Must be at least %d characters" min]
        elif s.Length > max then Invalid [sprintf "Must be at most %d characters" max]
        else Valid s

let validateEmailInput =
    validateNotEmpty
    >> validateLength 6 100
    >> validateEmail

let testEmails = [""; "ab"; "notanemail"; "test@example.com"; "user@domain.co.th"]
testEmails
|> List.map (fun e -> sprintf "%s -> %A" e (validateEmailInput e))
|> List.iter (printfn "%s")
```

---

## 8. Point-Free Style Programming

```fsharp
// Point-free style: เขียนฟังก์ชันโดยไม่ระบุ argument โดยตรง
// ใช้ composition แทน

// แบบ pointful (ระบุ argument)
let sumSquaresPointful numbers =
    numbers
    |> List.map (fun x -> x * x)
    |> List.sum

// แบบ point-free
let sumSquaresPointFree =
    List.map (fun x -> x * x)
    >> List.sum

let data = [1; 2; 3; 4; 5]
printfn "pointful: %d" (sumSquaresPointful data)    // 55
printfn "point-free: %d" (sumSquaresPointFree data)  // 55
```

```fsharp
// ตัวอย่าง point-free หลากหลาย

// แปลงเป็น bool ว่า list ว่างไหม
let isNonEmpty = List.isEmpty >> not

// หาค่าสูงสุดในรายการ
let listMax = List.reduce max

// นับจำนวนที่ตรงเงื่อนไข
let countWhere predicate = List.filter predicate >> List.length

// หา first element ที่ตรงเงื่อนไข
let findFirst predicate = List.filter predicate >> List.tryHead

// ตัวอย่างการใช้งาน
let nums = [3; 1; 4; 1; 5; 9; 2; 6]

printfn "isNonEmpty: %b" (isNonEmpty nums)
printfn "listMax: %d" (listMax nums)
printfn "countEvens: %d" (countWhere (fun x -> x % 2 = 0) nums)
printfn "firstBig: %A" (findFirst (fun x -> x > 7) nums)
```

```fsharp
// ระวัง: point-free อาจอ่านยากเมื่อซับซ้อนเกินไป

// อ่านง่าย
let processData =
    List.filter (fun x -> x > 0)
    >> List.map (fun x -> x * 2)
    >> List.sum

// อ่านยาก (ซับซ้อนเกินไป)
let hardToRead =
    List.filter ((<) 0)   // ((<) 0) x = 0 < x = x > 0
    >> List.map ((*) 2)   // ((*) 2) x = 2 * x
    >> List.sum

// ทั้งสองทำงานเหมือนกัน แต่อันแรกอ่านง่ายกว่า
let data = [-1; 2; -3; 4; 5; -6]
printfn "processData: %d" (processData data)  // 22
printfn "hardToRead: %d" (hardToRead data)    // 22
```

---

## 9. Practical Pipeline Examples

### 9.1 CSV Data Processing

```fsharp
// ประมวลผล CSV data

type SalesRecord = {
    Product: string
    Quantity: int
    Price: float
    Region: string
}

let parseLine (line: string) =
    let parts = line.Split(',')
    if parts.Length = 4 then
        try
            Some {
                Product = parts.[0].Trim()
                Quantity = int parts.[1]
                Price = float parts.[2]
                Region = parts.[3].Trim()
            }
        with _ -> None
    else None

let sampleData = [
    "Laptop,5,999.99,North"
    "Mouse,50,29.99,South"
    "Keyboard,30,79.99,North"
    "Monitor,10,299.99,East"
    "Headset,20,149.99,South"
    "invalid line"
]

// Pipeline สำหรับ analysis
let totalRevenue =
    List.choose parseLine
    >> List.map (fun r -> float r.Quantity * r.Price)
    >> List.sum

let byRegion region =
    List.choose parseLine
    >> List.filter (fun r -> r.Region = region)
    >> List.sumBy (fun r -> float r.Quantity * r.Price)

let revenue = totalRevenue sampleData
let northRevenue = byRegion "North" sampleData
let southRevenue = byRegion "South" sampleData

printfn "Total Revenue: $%.2f" revenue
printfn "North Revenue: $%.2f" northRevenue
printfn "South Revenue: $%.2f" southRevenue
```

### 9.2 Log Processing Pipeline

```fsharp
// ประมวลผล log files

type LogLevel = Debug | Info | Warning | Error

type LogEntry = {
    Timestamp: System.DateTime
    Level: LogLevel
    Message: string
}

let parseLogLevel (s: string) =
    match s.ToUpper() with
    | "DEBUG" -> Some Debug
    | "INFO" -> Some Info
    | "WARNING" | "WARN" -> Some Warning
    | "ERROR" -> Some Error
    | _ -> None

let parseLogLine (line: string) =
    let parts = line.Split([|' '|], 3)
    if parts.Length = 3 then
        match System.DateTime.TryParse(parts.[0]), parseLogLevel parts.[1] with
        | (true, ts), Some level ->
            Some { Timestamp = ts; Level = level; Message = parts.[2] }
        | _ -> None
    else None

let sampleLogs = [
    "2024-01-01 INFO Application started"
    "2024-01-01 DEBUG Loading configuration"
    "2024-01-01 WARNING Disk space low"
    "2024-01-01 ERROR Database connection failed"
    "2024-01-01 INFO User logged in"
    "invalid log line"
    "2024-01-01 ERROR Null reference exception"
]

// Pipeline สำหรับวิเคราะห์ log
let getErrors =
    List.choose parseLogLine
    >> List.filter (fun e -> e.Level = Error)
    >> List.map (fun e -> e.Message)

let countByLevel =
    List.choose parseLogLine
    >> List.groupBy (fun e -> e.Level)
    >> List.map (fun (level, entries) -> (level, List.length entries))

let errors = getErrors sampleLogs
let counts = countByLevel sampleLogs

printfn "Errors:"
errors |> List.iter (printfn "  - %s")
printfn "Counts: %A" counts
```

### 9.3 Image Processing Pipeline (ตัวอย่าง conceptual)

```fsharp
// Conceptual image processing pipeline

type Color = { R: int; G: int; B: int }

let clampChannel v = max 0 (min 255 v)

let applyBrightness factor color = {
    R = clampChannel (int (float color.R * factor))
    G = clampChannel (int (float color.G * factor))
    B = clampChannel (int (float color.B * factor))
}

let applyContrast factor color =
    let adjust x = clampChannel (int ((float x - 128.0) * factor + 128.0))
    { R = adjust color.R; G = adjust color.G; B = adjust color.B }

let toGrayscale color =
    let gray = (color.R + color.G + color.B) / 3
    { R = gray; G = gray; B = gray }

let invertColor color = {
    R = 255 - color.R
    G = 255 - color.G
    B = 255 - color.B
}

// สร้าง image filter pipeline
let vintageFilter =
    applyBrightness 1.1
    >> applyContrast 0.9
    >> (fun c -> { c with B = clampChannel (c.B - 20) })  // Reduce blue

let dramaticFilter =
    applyContrast 1.5
    >> applyBrightness 0.9

let processImage (filter: Color -> Color) (pixels: Color list) =
    pixels |> List.map filter

// ทดสอบ
let samplePixels = [
    { R = 200; G = 150; B = 100 }
    { R = 50; G = 100; B = 200 }
    { R = 128; G = 128; B = 128 }
]

let vintagePixels = processImage vintageFilter samplePixels
let dramaticPixels = processImage dramaticFilter samplePixels

printfn "Original: %A" samplePixels
printfn "Vintage: %A" vintagePixels
printfn "Dramatic: %A" dramaticPixels
```

---

## 10. Composing with Partial Application

```fsharp
// Partial application ทำให้ composition ง่ายขึ้น

// ฟังก์ชันพื้นฐาน
let between min max x = x >= min && x <= max
let divisibleBy n x = x % n = 0
let startsWith (prefix: string) (s: string) = s.StartsWith(prefix)

// Partial application เพื่อสร้างฟังก์ชันที่เฉพาะเจาะจง
let isTeen = between 13 19
let isThirties = between 30 39
let isDivisibleBy3 = divisibleBy 3
let isDivisibleBy5 = divisibleBy 5
let startsWithA = startsWith "A"

// รวม predicates
let both f g x = f x && g x
let either f g x = f x || g x

let isFizzBuzz = both isDivisibleBy3 isDivisibleBy5
let isFizzOrBuzz = either isDivisibleBy3 isDivisibleBy5

// ใช้กับ pipeline
let numbers = [1..30]

let fizzBuzzNumbers = numbers |> List.filter isFizzBuzz
let teensNumbers = numbers |> List.filter isTeen

printfn "FizzBuzz: %A" fizzBuzzNumbers  // [15; 30]
printfn "Teens: %A" teensNumbers        // [13..19]
```

```fsharp
// Partial application กับ formatting functions
let padLeft (totalWidth: int) (c: char) (s: string) = s.PadLeft(totalWidth, c)
let padRight (totalWidth: int) (c: char) (s: string) = s.PadRight(totalWidth, c)

let padToWidth20 = padLeft 20 ' '
let padRightTo30 = padRight 30 '-'
let centerText width (s: string) =
    let padding = (width - s.Length) / 2
    s.PadLeft(s.Length + padding).PadRight(width)

// Table formatter
let formatTableRow (cols: string list) widths =
    List.map2 (fun w s -> s.PadRight(w)) widths cols
    |> String.concat " | "

let headers = ["Name"; "Age"; "Department"]
let widths = [20; 5; 15]

let formatRow = formatTableRow headers widths
let separator = String.replicate (20+5+15+6) "-"

printfn "%s" (formatTableRow headers widths)
printfn "%s" separator
```

---

## 11. Building Complex Transformations

```fsharp
// สร้าง transformation complex จากฟังก์ชันง่ายๆ

// Building blocks สำหรับ JSON-like output
let quote s = sprintf "\"%s\"" s
let pair key value = sprintf "%s: %s" (quote key) value
let obj fields = sprintf "{ %s }" (String.concat ", " fields)
let arr items = sprintf "[ %s ]" (String.concat ", " items)
let num n = string n
let bool b = if b then "true" else "false"
let null_ = "null"

// สร้าง transformer สำหรับ Person
type Person = {
    Name: string
    Age: int
    Active: bool
    Email: string option
}

let personToJson (p: Person) =
    let emailField =
        match p.Email with
        | Some e -> pair "email" (quote e)
        | None -> pair "email" null_
    
    obj [
        pair "name" (quote p.Name)
        pair "age" (num p.Age)
        pair "active" (bool p.Active)
        emailField
    ]

let people = [
    { Name = "Alice"; Age = 30; Active = true; Email = Some "alice@example.com" }
    { Name = "Bob"; Age = 25; Active = false; Email = None }
]

let jsonArray =
    people
    |> List.map personToJson
    |> arr

printfn "%s" jsonArray
```

---

## 12. Data Processing Pipelines

```fsharp
// ETL (Extract, Transform, Load) Pipeline

// Extract
let extractFromCsv (csvContent: string) =
    csvContent.Split('\n')
    |> Array.toList
    |> List.tail  // skip header
    |> List.filter (fun line -> line.Trim() <> "")

// Transform
let parseEmployee (line: string) =
    let parts = line.Split(',')
    if parts.Length >= 3 then
        try
            Some {|
                Id = int parts.[0]
                Name = parts.[1].Trim()
                Salary = float parts.[2]
            |}
        with _ -> None
    else None

let enrichWithTax (emp: {| Id: int; Name: string; Salary: float |}) =
    let taxRate = if emp.Salary > 50000.0 then 0.30 else 0.20
    {|
        emp with
            Tax = emp.Salary * taxRate
            NetSalary = emp.Salary * (1.0 - taxRate)
    |}

// Load
let formatReport employees =
    let header = sprintf "%-5s %-20s %10s %10s %10s" "ID" "Name" "Salary" "Tax" "Net"
    let separator = String.replicate 60 "-"
    let rows = employees |> List.map (fun e ->
        sprintf "%-5d %-20s %10.2f %10.2f %10.2f" e.Id e.Name e.Salary e.Tax e.NetSalary)
    [header; separator] @ rows |> String.concat "\n"

// Full ETL pipeline
let csvData = """Id,Name,Salary
1,Alice Smith,75000
2,Bob Jones,45000
3,Charlie Brown,60000
4,Diana Prince,35000"""

let report =
    csvData
    |> extractFromCsv
    |> List.choose parseEmployee
    |> List.map enrichWithTax
    |> List.sortByDescending (fun e -> e.Salary)
    |> formatReport

printfn "%s" report
```

---

## สรุป (Summary)

การ compose ฟังก์ชันใน F# มีหลายวิธี:

| Operator | ความหมาย | ตัวอย่าง |
|----------|-----------|---------|
| `>>` | Forward composition | `f >> g` = `g(f(x))` |
| `<<` | Backward composition | `f << g` = `f(g(x))` |
| `\|>` | Forward pipe | `x \|> f` = `f x` |
| `<\|` | Backward pipe | `f <\| x` = `f x` |
| `\|\|>` | Two-arg pipe | `(x,y) \|\|> f` = `f x y` |
| `\|\|\|>` | Three-arg pipe | `(x,y,z) \|\|\|> f` = `f x y z` |

```fsharp
// ตัวอย่างสรุปใช้ทุก operator
let numbers = [1..10]

// Forward pipe pipeline
let result1 =
    numbers
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.sum

// Forward composition
let sumOfEvenSquares =
    List.filter (fun x -> x % 2 = 0)
    >> List.map (fun x -> x * x)
    >> List.sum

let result2 = sumOfEvenSquares numbers

// Two-arg pipe
let result3 = ([1..5], [6..10]) ||> List.map2 (+)

printfn "result1 = %d" result1  // 220
printfn "result2 = %d" result2  // 220
printfn "result3 = %A" result3  // [7; 9; 11; 13; 15]
```

Function composition ทำให้โค้ดอ่านง่าย maintainable และ reusable โดยแบ่งปัญหาซับซ้อนเป็นฟังก์ชันง่ายๆ แล้วประกอบเข้าด้วยกัน
