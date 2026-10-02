# Part 18 - ท่อและตัวดำเนินการ (Pipes and Custom Operators)

## บทนำ (Introduction)

Pipe operators ใน F# ทำให้เขียนโค้ดแบบ "data flows" ได้ และ Custom operators ให้เราสร้าง syntax ใหม่เพื่อทำให้โค้ดอ่านง่ายขึ้น

---

## 1. |> Pipe Operator in Depth

```fsharp
// |> คือ forward pipe: ส่งค่าเป็น argument สุดท้ายของฟังก์ชัน
// x |> f เหมือนกับ f x

// พื้นฐาน
let result = 5 |> (fun x -> x * 2)  // 10

// Pipeline: data ไหลจากซ้ายไปขวา
let result2 =
    "  Hello World  "
    |> fun s -> s.Trim()
    |> fun s -> s.ToLower()
    |> fun s -> s.Replace(" ", "-")

printfn "%s" result2  // "hello-world"
```

```fsharp
// |> กับ List operations (มีประโยชน์มาก!)
let numbers = [1..20]

// แบบไม่ใช้ pipe (อ่านยาก)
let r1 = List.take 5 (List.sortDescending (List.filter (fun x -> x % 2 = 0) numbers))

// แบบใช้ pipe (อ่านง่าย)
let r2 =
    numbers
    |> List.filter (fun x -> x % 2 = 0)
    |> List.sortDescending
    |> List.take 5

printfn "r1: %A" r1
printfn "r2: %A" r2
```

```fsharp
// |> กับ curried functions
// ค่าจะถูก apply เป็น argument สุดท้าย

// ฟังก์ชัน 2 argument: argument แรกถูก partial apply ก่อน
let multiplyBy factor x = x * factor

let result = 5 |> multiplyBy 3    // multiplyBy 3 5 = 15
let result2 = 5 |> multiplyBy 10  // multiplyBy 10 5 = 50

printfn "result: %d" result
printfn "result2: %d" result2
```

```fsharp
// |> ในการเขียนฟังก์ชัน
// เปรียบเทียบ OOP style vs functional style

// OOP style
let processMoney1 (amount: float) =
    let rounded = System.Math.Round(amount, 2)
    let formatted = sprintf "%.2f" rounded
    sprintf "$%s" formatted

// Functional pipe style
let processMoney2 amount =
    amount
    |> fun a -> System.Math.Round(a, 2)
    |> fun a -> sprintf "%.2f" a
    |> sprintf "$%s"

printfn "%s" (processMoney1 1234.567)
printfn "%s" (processMoney2 1234.567)
```

---

## 2. Pipeline Style vs Nested Calls

```fsharp
// เปรียบเทียบ: nested calls vs pipeline

type Student = { Name: string; Score: float; Passed: bool }

let students = [
    { Name = "Alice"; Score = 85.0; Passed = true }
    { Name = "Bob"; Score = 55.0; Passed = false }
    { Name = "Charlie"; Score = 92.0; Passed = true }
    { Name = "Diana"; Score = 78.0; Passed = true }
    { Name = "Eve"; Score = 45.0; Passed = false }
]

// Nested (อ่านยาก: ต้องอ่านจากในออกนอก)
let topPassers1 =
    List.take 3 (
        List.sortByDescending (fun s -> s.Score) (
            List.filter (fun s -> s.Passed) students))

// Pipeline (อ่านง่าย: data flows ตามลำดับ)
let topPassers2 =
    students
    |> List.filter (fun s -> s.Passed)
    |> List.sortByDescending (fun s -> s.Score)
    |> List.take 3

printfn "topPassers: %A" (List.map (fun s -> s.Name) topPassers2)
```

---

## 3. Multiple Piped Operations

```fsharp
// Pipeline ซับซ้อน

type Order = {
    Id: int
    Customer: string
    Items: (string * float) list
    Discount: float
}

let orders = [
    { Id = 1; Customer = "Alice"; Items = [("Book", 25.0); ("Pen", 3.0)]; Discount = 0.0 }
    { Id = 2; Customer = "Bob"; Items = [("Laptop", 999.0); ("Mouse", 29.0)]; Discount = 0.1 }
    { Id = 3; Customer = "Alice"; Items = [("Chair", 150.0)]; Discount = 0.05 }
    { Id = 4; Customer = "Charlie"; Items = [("Coffee", 12.0); ("Mug", 8.0)]; Discount = 0.0 }
]

let orderTotal order =
    let subtotal = order.Items |> List.sumBy snd
    subtotal * (1.0 - order.Discount)

let report =
    orders
    |> List.map (fun o -> (o.Customer, orderTotal o))
    |> List.groupBy fst
    |> List.map (fun (customer, totals) ->
        (customer, totals |> List.sumBy snd))
    |> List.sortByDescending snd
    |> List.map (fun (customer, total) ->
        sprintf "%-15s $%.2f" customer total)
    |> String.concat "\n"

printfn "Sales by Customer:\n%s" report
```

---

## 4. Debugging Pipelines (tee function)

```fsharp
// tee function: inspect ค่าใน pipeline โดยไม่เปลี่ยนแปลงมัน

let tee f x =
    f x   // เรียก side effect function
    x     // ส่งคืนค่าเดิม

// ใช้ใน pipeline
let result =
    [1..10]
    |> tee (printfn "Before filter: %A")
    |> List.filter (fun x -> x % 2 = 0)
    |> tee (printfn "After filter: %A")
    |> List.map (fun x -> x * x)
    |> tee (printfn "After map: %A")
    |> List.sum
    |> tee (printfn "Sum: %d")

printfn "Final result: %d" result
```

```fsharp
// Debug pipeline ที่ดีกว่า
let debugLog label value =
    printfn "[DEBUG] %s: %A" label value
    value

let result2 =
    [1..5]
    |> debugLog "Input"
    |> List.filter (fun x -> x > 2)
    |> debugLog "After filter"
    |> List.map (fun x -> x * 10)
    |> debugLog "After multiply"
    |> List.sum
    |> debugLog "Sum"
```

```fsharp
// Conditional debug (เปิด/ปิด debug ได้)
let debugMode = true

let debugIf condition label value =
    if condition then
        printfn "[%s] %A" label value
    value

let result3 =
    [1..5]
    |> debugIf debugMode "Start"
    |> List.map ((*) 2)
    |> debugIf debugMode "Doubled"
    |> List.sum
    |> debugIf debugMode "Sum"
```

---

## 5. Custom Operators in F#

```fsharp
// F# ให้สร้าง custom operators ได้

// Operator ต้องใช้ตัวอักษร: ! $ % & * + - . / : < = > ? @ ^ | ~
// และต้องไม่ซ้อนกับ keyword

// สร้าง operator
let (|?|) x defaultVal =
    match x with
    | Some v -> v
    | None -> defaultVal

// ใช้งาน
let someValue = Some 42
let noneValue : int option = None

let v1 = someValue |?| 0  // 42
let v2 = noneValue |?| 99 // 99

printfn "v1 = %d" v1
printfn "v2 = %d" v2
```

```fsharp
// Operator สำหรับ Result type
let (>>=) (result: Result<'a, 'e>) (f: 'a -> Result<'b, 'e>) =
    Result.bind f result

let (>=>) (f: 'a -> Result<'b, 'e>) (g: 'b -> Result<'c, 'e>) =
    fun x -> f x >>= g

// ใช้งาน
let divide x y =
    if y = 0 then Error "Division by zero"
    else Ok (x / y)

let sqrt' x =
    if x < 0 then Error "Cannot take sqrt of negative"
    else Ok (sqrt (float x))

// Chain operations
let r1 = Ok 16 >>= divide 100 // Ok(6) เพราะ 100/16 = 6
let r2 = Ok 0 >>= divide 10   // Error "Division by zero"

printfn "r1: %A" r1
printfn "r2: %A" r2
```

---

## 6. let (|*|) Operator Definition

```fsharp
// syntax ของ custom operator

// Infix operator (ใช้ระหว่าง operands)
let (++) a b = a + b + 1  // ทำให้ += ด้วย

printfn "3 ++ 4 = %d" (3 ++ 4)  // 8

// Unary operator (ใช้กับ operand เดียว)
let (!!) b = not b

printfn "!!true = %b" (!!true)   // false
printfn "!!false = %b" (!!false) // true
```

```fsharp
// Vector/Matrix operators
type Vec2 = { X: float; Y: float }

let (+.) a b = { X = a.X + b.X; Y = a.Y + b.Y }
let (-.) a b = { X = a.X - b.X; Y = a.Y - b.Y }
let ( *.) s v = { X = s * v.X; Y = s * v.Y }  // scalar multiply
let dot a b = a.X * b.X + a.Y * b.Y
let (|.|) a b = dot a b  // custom dot product operator

let v1 = { X = 1.0; Y = 2.0 }
let v2 = { X = 3.0; Y = 4.0 }

let sum = v1 +. v2
let diff = v2 -. v1
let scaled = 2.0 *. v1
let dotProduct = v1 |.| v2

printfn "v1 + v2 = %A" sum
printfn "v2 - v1 = %A" diff
printfn "2 * v1 = %A" scaled
printfn "v1 . v2 = %f" dotProduct
```

---

## 7. Operator Precedence

```fsharp
// F# operator precedence (สูงสุดไปต่ำสุด)
// 1. !, ~, ++ (unary prefix)
// 2. **                     (right assoc)
// 3. *, /, %               (left assoc)
// 4. +, -                  (left assoc)
// 5. :, ::                 (right assoc)
// 6. @, ^                  (right assoc)
// 7. =, <, >, |, &, $, %  (left assoc)
// 8. &, &&                 (left assoc)
// 9. ||                    (left assoc)
// 10. ,                    (left assoc)
// 11. :=                   (right assoc)
// 12. if, fun, let, match  (special)

// Custom operators ได้ precedence จาก first character
// ! # $ % & * + - . / : < = > ? @ ^ | ~

// ตัวอย่าง
printfn "%d" (2 + 3 * 4)   // 14 (ไม่ใช่ 20)
printfn "%d" (2 ** 3 + 1)  // 9 (ไม่ใช่ 16: 2^3=8, +1=9)
```

```fsharp
// Custom operator ที่มี precedence ตาม first character

// เริ่มด้วย * จะมี precedence เหมือน *
let ( *+* ) a b = a * a + b * b   // first char *

// เริ่มด้วย + จะมี precedence เหมือน +
let ( +++ ) a b = a + b + b

// ทดสอบ precedence
let r1 = 2 *+* 3 + 1    // (2 *+* 3) + 1 = (4+9) + 1 = 14
let r2 = 1 + 2 *+* 3    // 1 + (2 *+* 3) = 1 + 13 = 14

printfn "2 *+* 3 + 1 = %d" r1  // 14
printfn "1 + 2 *+* 3 = %d" r2  // 14
```

---

## 8. Infix vs Prefix Operators

```fsharp
// Infix: ใช้ระหว่าง operands (most common)
let ( ++ ) a b = a + b + 1
let result = 3 ++ 4  // 8

// Prefix (unary): ใช้นำหน้า operand
// ต้องเริ่มด้วย ! หรือ ~
let (!) x = not x
let (!!) x = not (not x)

printfn "!true = %b" (not true)
printfn "!!true = %b" (!!true)
```

```fsharp
// Backtick notation: ทำให้ฟังก์ชันธรรมดาใช้เป็น infix ได้
let isMultipleOf divisor n = n % divisor = 0

// แบบปกติ
let r1 = isMultipleOf 3 9  // true

// แบบ backtick infix
let r2 = 9 `isMultipleOf` 3  // F# ไม่ support รูปแบบนี้โดยตรง
// ต้องใช้ operator แทน

let ( |%%| ) n d = n % d = 0

let r3 = 9 |%%| 3  // true
let r4 = 10 |%%| 3 // false

printfn "9 |%%| 3 = %b" r3
printfn "10 |%%| 3 = %b" r4
```

---

## 9. Numeric Operators

```fsharp
// Operators สำหรับ numeric types

// พื้นฐาน
let (+) = (+)  // integer add
let (*) = (*)  // integer multiply

// Custom arithmetic operators
let ( **. ) (x: float) (y: float) = System.Math.Pow(x, y)  // float power
let ( /. ) (x: float) (y: float) = x / y

// Integer division and modulo
let (%%) x y = x - y * (x / y)  // floor division

printfn "2.0 **. 10.0 = %f" (2.0 **. 10.0)  // 1024.0
printfn "10.0 /. 3.0 = %f" (10.0 /. 3.0)    // 3.333...
```

```fsharp
// Safe arithmetic operators (return Result)
let safeDiv x y =
    if y = 0 then Error "Division by zero"
    else Ok (x / y)

let safeSqrt x =
    if x < 0.0 then Error "Cannot sqrt negative"
    else Ok (sqrt x)

let ( /? ) x y = safeDiv x y
let ( ?= ) = Result.defaultValue

let r1 = (10 /? 2) ?= 0  // 5
let r2 = (10 /? 0) ?= -1 // -1 (default)

printfn "10 /? 2 = %d" r1
printfn "10 /? 0 = %d" r2
```

---

## 10. Comparison Operators

```fsharp
// Custom comparison operators

// >=< (between)
let ( >=< ) x (lo, hi) = x >= lo && x <= hi

let age = 25
let isAdult = age >=< (18, 65)
let isTeen = age >=< (13, 19)

printfn "25 >=< (18,65) = %b" isAdult  // true
printfn "25 >=< (13,19) = %b" isTeen   // false
```

```fsharp
// Approximate equality สำหรับ float
let epsilon = 1e-9
let ( ~=~ ) a b = abs (a - b) < epsilon

let r1 = 0.1 + 0.2 ~=~ 0.3  // true (เพราะ float arithmetic)
let r2 = 1.0 ~=~ 2.0         // false

printfn "0.1+0.2 ~=~ 0.3 = %b" r1  // true
printfn "1.0 ~=~ 2.0 = %b" r2      // false
```

---

## 11. Logical Operators

```fsharp
// Custom logical operators

// Exclusive OR
let ( |^| ) a b = a <> b

printfn "true |^| true = %b" (true |^| true)   // false
printfn "true |^| false = %b" (true |^| false)  // true
printfn "false |^| false = %b" (false |^| false) // false
```

```fsharp
// Short-circuit operators สำหรับ Option
let ( ?&& ) a b =
    match a with
    | None -> None
    | Some _ -> b

let ( ?|| ) a b =
    match a with
    | Some _ -> a
    | None -> b

let s = Some 1
let n : int option = None

printfn "Some 1 ?&& None = %A" (s ?&& n)     // None
printfn "Some 1 ?|| None = %A" (s ?|| n)     // Some 1
printfn "None ?|| Some 2 = %A" (n ?|| Some 2) // Some 2
```

---

## 12. String Operators

```fsharp
// Custom string operators

// Concatenation with separator
let ( +/ ) (a: string) (b: string) = a + "/" + b
let ( +. ) (a: string) (b: string) = a + "." + b
let ( +@ ) (a: string) (b: string) = a + "@" + b

let path = "home" +/ "user" +/ "documents"
let domain = "www" +. "example" +. "com"
let email = "user" +@ "example.com"

printfn "path: %s" path    // "home/user/documents"
printfn "domain: %s" domain // "www.example.com"
printfn "email: %s" email   // "user@example.com"
```

```fsharp
// Pattern matching operators
let ( =~ ) (input: string) (pattern: string) =
    System.Text.RegularExpressions.Regex.IsMatch(input, pattern)

let ( !~ ) (input: string) (pattern: string) =
    not (input =~ pattern)

let isEmail s = s =~ @"^[^@]+@[^@]+\.[^@]+$"
let isPhoneNumber s = s =~ @"^\d{10}$"

printfn "alice@ex.com is email: %b" (isEmail "alice@ex.com")  // true
printfn "notanemail is email: %b" (isEmail "notanemail")        // false
printfn "0812345678 is phone: %b" (isPhoneNumber "0812345678") // true
```

---

## 13. Custom Operators for DSLs

```fsharp
// Domain-Specific Language (DSL) operators

// HTML Builder DSL
type HtmlElement =
    | Text of string
    | Tag of string * (string * string) list * HtmlElement list

let h1 content = Tag("h1", [], [Text content])
let p content = Tag("p", [], [Text content])
let div children = Tag("div", [], children)
let withClass cls elem =
    match elem with
    | Tag(name, attrs, children) -> Tag(name, ("class", cls) :: attrs, children)
    | _ -> elem

let ( <+> ) parent child =
    match parent with
    | Tag(name, attrs, children) -> Tag(name, attrs, children @ [child])
    | _ -> parent

let rec renderHtml = function
    | Text t -> t
    | Tag(name, attrs, children) ->
        let attrStr = attrs |> List.map (fun (k, v) -> sprintf " %s=\"%s\"" k v) |> String.concat ""
        let childStr = children |> List.map renderHtml |> String.concat ""
        sprintf "<%s%s>%s</%s>" name attrStr childStr name

// สร้าง HTML
let page =
    div [
        h1 "Hello, World!" |> withClass "title"
        p "Welcome to F# DSL" |> withClass "intro"
        (div [] |> withClass "content")
            <+> (p "First paragraph")
            <+> (p "Second paragraph")
    ]

printfn "%s" (renderHtml page)
```

```fsharp
// Query DSL
type Query<'a> = {
    Source: 'a list
    Filter: 'a -> bool
    Transform: 'a -> 'a
    Limit: int option
}

let emptyQuery source = {
    Source = source
    Filter = fun _ -> true
    Transform = id
    Limit = None
}

let ( |>> ) query f = { query with Transform = query.Transform >> f }
let ( |?> ) query pred = { query with Filter = fun x -> query.Filter x && pred x }
let ( |!> ) query n = { query with Limit = Some n }

let runQuery q =
    let filtered = q.Source |> List.filter q.Filter
    let transformed = filtered |> List.map q.Transform
    match q.Limit with
    | Some n -> List.take (min n (List.length transformed)) transformed
    | None -> transformed

let data = [1..20]
let result =
    emptyQuery data
    |?> (fun x -> x % 2 = 0)    // filter evens
    |>> (fun x -> x * x)         // square
    |?> (fun x -> x > 10)        // filter > 10
    |!> 5                         // take 5
    |> runQuery

printfn "Query result: %A" result
```

```fsharp
// Configuration DSL
type AppConfig = {
    Host: string
    Port: int
    MaxConnections: int
    Timeout: int
    UseSSL: bool
}

let defaultAppConfig = {
    Host = "localhost"
    Port = 8080
    MaxConnections = 100
    Timeout = 30
    UseSSL = false
}

// Operators for building config
let ( @@ ) config host = { config with Host = host }
let ( @! ) config port = { config with Port = port }
let ( @> ) config maxConn = { config with MaxConnections = maxConn }
let ( @? ) config ssl = { config with UseSSL = ssl }

let productionConfig =
    defaultAppConfig
    @@ "api.example.com"
    @! 443
    @> 500
    @? true

printfn "Config: %A" productionConfig
```

---

## 14. Practical Pipeline Examples

```fsharp
// ตัวอย่างจริง: Data processing pipeline

// Step 1: Define domain
type Transaction = {
    Id: int
    Date: System.DateTime
    Amount: float
    Category: string
    Description: string
}

// Step 2: Sample data
let transactions = [
    { Id = 1; Date = System.DateTime(2024, 1, 1); Amount = 150.0; Category = "Food"; Description = "Restaurant" }
    { Id = 2; Date = System.DateTime(2024, 1, 2); Amount = 30.0; Category = "Transport"; Description = "Bus" }
    { Id = 3; Date = System.DateTime(2024, 1, 3); Amount = 500.0; Category = "Shopping"; Description = "Clothes" }
    { Id = 4; Date = System.DateTime(2024, 1, 4); Amount = 75.0; Category = "Food"; Description = "Groceries" }
    { Id = 5; Date = System.DateTime(2024, 1, 5); Amount = 1200.0; Category = "Bills"; Description = "Electricity" }
]

// Step 3: Analysis pipeline
let monthlyReport transactions =
    transactions
    |> List.groupBy (fun t -> t.Category)
    |> List.map (fun (cat, txns) ->
        {| Category = cat
           Count = List.length txns
           Total = txns |> List.sumBy (fun t -> t.Amount)
           Average = txns |> List.averageBy (fun t -> t.Amount) |})
    |> List.sortByDescending (fun r -> r.Total)

let report = monthlyReport transactions
report |> List.iter (fun r ->
    printfn "%-15s: %3d txns, $%8.2f total, $%6.2f avg"
        r.Category r.Count r.Total r.Average)
```

---

## สรุป (Summary)

```fsharp
// สรุป Pipes and Operators ใน F#

// 1. |> forward pipe
let result1 =
    [1..10]
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.sum
printfn "result1 = %d" result1

// 2. tee for debugging
let tee f x = f x; x
let result2 =
    [1..5]
    |> tee (printfn "Input: %A")
    |> List.map ((*) 2)
    |> List.sum
printfn "result2 = %d" result2

// 3. Custom operators
let ( |?| ) opt def = Option.defaultValue def opt
let ( >>= ) r f = Result.bind f r

let opt = Some 42
let v = opt |?| 0  // 42
printfn "v = %d" v

// 4. Domain operators
let ( >=< ) x (lo, hi) = x >= lo && x <= hi
let inRange = 5 >=< (1, 10)  // true
printfn "5 in [1,10] = %b" inRange
```

Custom operators และ pipe operators ทำให้ F# มีความยืดหยุ่นสูงในการ:
- สร้าง **DSLs** ที่อ่านง่าย
- เขียน **pipelines** ที่ logic ชัดเจน
- ทำให้ **domain concepts** มี syntax ที่เป็นธรรมชาติ
- **Debug** pipelines ได้ง่ายด้วย tee
