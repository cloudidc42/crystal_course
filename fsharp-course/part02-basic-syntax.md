# Part 2 - ไวยากรณ์พื้นฐาน (Basic Syntax)

## บทนำ

ในบทนี้เราจะเรียนรู้ไวยากรณ์พื้นฐานของ F# อย่างละเอียด ตั้งแต่การประกาศตัวแปร การใช้ operators ไปจนถึงการรับ input และแสดง output F# มีไวยากรณ์ที่กระชับแต่ทรงพลัง ซึ่งช่วยให้เขียนโค้ดได้อย่างมีประสิทธิภาพ

---

## 2.1 let Bindings (การกำหนดค่า)

### 2.1.1 Basic let Binding

```fsharp
// let binding กำหนดค่าให้กับชื่อ (binding ไม่ใช่ assignment)
let x = 42
let name = "Alice"
let pi = 3.14159265358979
let isReady = true

// แสดงค่า
printfn "x = %d" x
printfn "name = %s" name
printfn "pi = %f" pi
printfn "isReady = %b" isReady
```

### 2.1.2 ความแตกต่างระหว่าง Binding และ Assignment

```fsharp
// let binding ไม่ใช่ assignment!
// ค่าที่ bind แล้วไม่สามารถเปลี่ยนได้ (immutable)

let score = 100
// score <- 200  // Error! ไม่สามารถทำได้

// แต่สามารถ shadow ได้ (สร้าง binding ใหม่ด้วยชื่อเดิม)
let result = "first"
let result = "second"   // Shadow - result ใหม่แทนที่ result เดิม
printfn "%s" result     // "second"

// การ shadowing มีประโยชน์สำหรับ intermediate calculations
let price = 100.0
let price = price * 1.07  // รวม VAT 7%
let price = price * 0.9   // ลด 10%
printfn "ราคาสุดท้าย: %.2f" price
```

### 2.1.3 let สำหรับ Functions

```fsharp
// let ใช้สำหรับ bind ทั้ง values และ functions
let addOne x = x + 1
let multiply a b = a * b
let greet name = printfn "สวัสดี, %s!" name

// เรียกใช้
printfn "%d" (addOne 5)        // 6
printfn "%d" (multiply 3 4)    // 12
greet "World"
```

### 2.1.4 let ใน Local Scope

```fsharp
// let สามารถใช้ภายใน function เพื่อสร้าง local values
let calculateArea radius =
    let pi = 3.14159265358979  // local binding
    let radiusSquared = radius * radius  // local binding
    pi * radiusSquared  // return value (ไม่ต้องมี return keyword)

let area = calculateArea 5.0
printfn "Area: %.4f" area
```

---

## 2.2 Mutable Variables (ตัวแปรที่เปลี่ยนได้)

```fsharp
// ต้องใช้ keyword mutable เพื่อให้เปลี่ยนค่าได้
let mutable counter = 0
printfn "Initial counter: %d" counter

counter <- 1   // ใช้ <- สำหรับการ assign ค่าใหม่
counter <- counter + 1
printfn "Counter after increment: %d" counter

// mutable ใน loop
let mutable sum = 0
for i in 1..10 do
    sum <- sum + i
printfn "Sum 1 to 10: %d" sum

// ตัวอย่างจริง: running total
let mutable total = 0.0
let prices = [29.99; 49.99; 15.50; 89.00]
for price in prices do
    total <- total + price
printfn "Total: %.2f" total
```

### 2.2.1 เมื่อไหร่ควรใช้ mutable

```fsharp
// ควรใช้ mutable เมื่อ:
// 1. Performance-critical loops
// 2. Interop กับ .NET APIs
// 3. Implementation ของ algorithms ที่ต้องใช้ state

// แต่โดยทั่วไป ควรหลีกเลี่ยงและใช้ functional approach แทน

// แทนที่จะใช้ mutable:
let mutable total2 = 0.0
for p in prices do
    total2 <- total2 + p

// ควรใช้:
let total3 = prices |> List.sum
printfn "Total (functional): %.2f" total3
```

---

## 2.3 Significant Whitespace (การเยื้องบรรทัด)

F# ใช้การเยื้องบรรทัด (indentation) เพื่อกำหนดขอบเขตของโค้ด คล้ายกับ Python

```fsharp
// การเยื้องบรรทัดกำหนด scope
let example value =
    if value > 0 then
        printfn "Positive"     // ต้องเยื้อง
        let doubled = value * 2
        printfn "Doubled: %d" doubled
    else
        printfn "Non-positive" // ต้องเยื้อง
    
    // โค้ดส่วนนี้อยู่นอก if/else
    printfn "Done"

example 5
example -3
```

### 2.3.1 กฎการเยื้องบรรทัด

```fsharp
// กฎ: code ที่เป็น continuation ต้องเยื้องมากกว่า parent

let multilineFunction a b c =
    // แต่ละบรรทัดต้องเยื้องเท่ากัน
    let x = a + b
    let y = b + c
    let z = a + c
    x + y + z

// Multiline expression - ต้อง align หรือเยื้องกว่า
let longCalculation =
    1 + 2 +
    3 + 4 +   // ต้องเยื้อง
    5 + 6

// Function application - ต้องเยื้อง arguments
let result =
    List.map
        (fun x -> x * 2)  // argument ต้องเยื้อง
        [1; 2; 3]
```

### 2.3.2 Common Indentation Mistakes

```fsharp
// WRONG:
// let badFunction x =
// printfn "%d" x   // Error: ต้องเยื้อง

// CORRECT:
let goodFunction x =
    printfn "%d" x  // เยื้อง 4 spaces (หรือ 2, หรือ tab ก็ได้)

// ขนาดการเยื้องไม่สำคัญ แต่ต้องสม่ำเสมอภายใน block เดียวกัน
let consistentIndent a b =
  let x = a + b   // เยื้อง 2 spaces
  let y = a * b   // เยื้อง 2 spaces
  x + y           // เยื้อง 2 spaces - ทั้งหมดเท่ากัน = OK
```

---

## 2.4 Comments (คอมเมนต์)

```fsharp
// นี่คือ single-line comment
// ใช้ // สำหรับ comment บรรทัดเดียว

(* 
   นี่คือ multi-line comment
   ใช้ (* ... *) สำหรับ comment หลายบรรทัด
   สามารถ nest ได้:
   (* nested comment *)
*)

let x = 42 (* inline comment *)

/// XML documentation comment
/// ใช้ /// สำหรับ documentation (แสดงใน IntelliSense)
/// <summary>คำนวณพื้นที่วงกลม</summary>
/// <param name="radius">รัศมีของวงกลม</param>
/// <returns>พื้นที่วงกลม</returns>
let circleArea radius =
    System.Math.PI * radius * radius

// คอมเมนต์ที่ดีอธิบาย "ทำไม" ไม่ใช่ "อะไร"
// GOOD: คำนวณ total พร้อม VAT 7% ตามกฎหมายภาษีไทย
let totalWithVAT amount = amount * 1.07

// BAD: คูณ amount ด้วย 1.07
// let totalWithVAT amount = amount * 1.07
```

---

## 2.5 Strings (สตริง)

### 2.5.1 Basic Strings

```fsharp
// String literals ใช้ double quotes
let str1 = "Hello, World!"
let str2 = "สวัสดีโลก"
let empty = ""

// String concatenation ด้วย + หรือ sprintf
let firstName = "John"
let lastName = "Doe"
let fullName = firstName + " " + lastName
printfn "%s" fullName

// String length
printfn "Length: %d" str1.Length
printfn "Length: %d" (String.length str1)

// Access characters
printfn "First char: %c" str1.[0]
printfn "Last char: %c" str1.[str1.Length - 1]
```

### 2.5.2 String Interpolation

```fsharp
// F# 5+ รองรับ string interpolation ด้วย $"..."
let name = "Alice"
let age = 30
let city = "Bangkok"

// Basic interpolation
let greeting = $"Hello, {name}!"
printfn "%s" greeting

// With expressions
let info = $"{name} อายุ {age} ปี อาศัยอยู่ที่ {city}"
printfn "%s" info

// Formatting within interpolation
let price = 1234.5678
let formattedPrice = $"ราคา: {price:F2} บาท"
printfn "%s" formattedPrice

// Multiple values
let x = 10
let y = 20
printfn $"{x} + {y} = {x + y}"

// Expression dalam interpolation
let numbers = [1; 2; 3; 4; 5]
printfn $"Sum: {numbers |> List.sum}"
printfn $"Count: {numbers.Length}"

// Nested interpolation (F# 6+)
let items = ["apple"; "banana"; "cherry"]
let listStr = $"Items: {String.concat \", \" items}"
printfn "%s" listStr
```

### 2.5.3 Verbatim Strings

```fsharp
// Verbatim strings ด้วย @"..." - ไม่ต้อง escape backslash
let path = @"C:\Users\Alice\Documents\file.txt"
let normalPath = "C:\\Users\\Alice\\Documents\\file.txt"  // เหมือนกัน

printfn "%s" path
printfn "%s" normalPath

// Multi-line verbatim
let multiLine = @"บรรทัดที่ 1
บรรทัดที่ 2
บรรทัดที่ 3"
printfn "%s" multiLine

// Verbatim string กับ double quotes (ใช้ "" แทน \")
let withQuotes = @"He said ""Hello""!"
printfn "%s" withQuotes
```

### 2.5.4 Triple-quoted Strings

```fsharp
// Triple-quoted strings สำหรับ raw strings
let json = """
{
    "name": "Alice",
    "age": 30,
    "city": "Bangkok"
}
"""
printfn "%s" json

// ใช้งานได้กับ SQL, HTML, etc.
let sql = """
    SELECT name, age
    FROM users
    WHERE city = 'Bangkok'
    ORDER BY name
"""
printfn "%s" sql
```

### 2.5.5 String Operations

```fsharp
open System

let text = "Hello, F# World!"

// Methods ทั่วไป
printfn "Upper: %s" (text.ToUpper())
printfn "Lower: %s" (text.ToLower())
printfn "Trim: '%s'" ("  spaces  ".Trim())
printfn "TrimStart: '%s'" ("  spaces  ".TrimStart())
printfn "TrimEnd: '%s'" ("  spaces  ".TrimEnd())

// Contains, StartsWith, EndsWith
printfn "Contains 'F#': %b" (text.Contains("F#"))
printfn "Starts with 'Hello': %b" (text.StartsWith("Hello"))
printfn "Ends with 'World!': %b" (text.EndsWith("World!"))

// Replace
let replaced = text.Replace("World", "Thailand")
printfn "%s" replaced

// Split
let sentence = "one,two,three,four"
let words = sentence.Split(',')
printfn "Words: %A" words

// Substring
printfn "Substring: %s" (text.Substring(7, 2))  // "F#"

// IndexOf
printfn "Index of 'F#': %d" (text.IndexOf("F#"))

// Join
let parts = ["สวัสดี"; "โลก"; "F#"]
let joined = String.concat " " parts
printfn "%s" joined
```

---

## 2.6 Numbers และ Operators

### 2.6.1 Arithmetic Operators

```fsharp
// Integer arithmetic
let a = 10
let b = 3

printfn "a + b = %d" (a + b)    // 13
printfn "a - b = %d" (a - b)    // 7
printfn "a * b = %d" (a * b)    // 30
printfn "a / b = %d" (a / b)    // 3 (integer division!)
printfn "a %% b = %d" (a % b)   // 1 (modulo)

// Float arithmetic
let x = 10.0
let y = 3.0

printfn "x + y = %f" (x + y)    // 13.0
printfn "x - y = %f" (x - y)    // 7.0
printfn "x * y = %f" (x * y)    // 30.0
printfn "x / y = %f" (x / y)    // 3.333...
printfn "x %% y = %f" (x % y)   // 1.0

// Power (exponentiation)
let power = 2.0 ** 10.0   // 1024.0
printfn "2^10 = %f" power

// Negative numbers
let neg = -5
let negFloat = -3.14
printfn "Negatives: %d, %f" neg negFloat
```

### 2.6.2 Integer Division และ Modulo

```fsharp
// Integer division truncates toward zero
printfn "7 / 2 = %d" (7 / 2)      // 3
printfn "-7 / 2 = %d" (-7 / 2)    // -3
printfn "7 / -2 = %d" (7 / -2)    // -3

// Modulo
printfn "7 %% 3 = %d" (7 % 3)     // 1
printfn "-7 %% 3 = %d" (-7 % 3)   // -1 (sign of dividend)
printfn "7 %% -3 = %d" (7 % -3)   // 1

// Practical use: check even/odd
let isEven n = n % 2 = 0
let isOdd n = n % 2 <> 0

printfn "10 is even: %b" (isEven 10)
printfn "7 is odd: %b" (isOdd 7)

// Check divisibility
let isDivisibleBy divisor n = n % divisor = 0
let isMultipleOf5 = isDivisibleBy 5

printfn "15 is multiple of 5: %b" (isMultipleOf5 15)
printfn "13 is multiple of 5: %b" (isMultipleOf5 13)
```

### 2.6.3 Comparison Operators

```fsharp
// Comparison operators
printfn "5 = 5: %b" (5 = 5)      // true  (= not ==)
printfn "5 <> 6: %b" (5 <> 6)    // true  (<> not !=)
printfn "5 < 6: %b" (5 < 6)      // true
printfn "5 > 4: %b" (5 > 4)      // true
printfn "5 <= 5: %b" (5 <= 5)    // true
printfn "5 >= 6: %b" (5 >= 6)    // false

// Comparing strings
printfn "\"abc\" < \"abd\": %b" ("abc" < "abd")  // true (lexicographic)
printfn "\"ABC\" = \"abc\": %b" ("ABC" = "abc")  // false (case-sensitive)

// Structural equality สำหรับ lists, tuples, records
let list1 = [1; 2; 3]
let list2 = [1; 2; 3]
let list3 = [1; 2; 4]
printfn "list1 = list2: %b" (list1 = list2)  // true
printfn "list1 = list3: %b" (list1 = list3)  // false

// Tuple comparison
let t1 = (1, "a")
let t2 = (1, "a")
let t3 = (2, "b")
printfn "t1 = t2: %b" (t1 = t2)  // true
printfn "t1 < t3: %b" (t1 < t3)  // true (compares element by element)
```

### 2.6.4 Boolean Operators

```fsharp
// Boolean values
let trueVal = true
let falseVal = false

// AND - ทั้งสองต้องเป็น true
printfn "true && true = %b" (true && true)   // true
printfn "true && false = %b" (true && false) // false
printfn "false && true = %b" (false && true) // false

// OR - อย่างน้อยหนึ่งต้องเป็น true
printfn "true || false = %b" (true || false) // true
printfn "false || false = %b" (false || false) // false

// NOT
printfn "not true = %b" (not true)   // false
printfn "not false = %b" (not false) // true

// Short-circuit evaluation
let checkAge age =
    printfn "Checking age %d..." age
    age >= 18

let checkId hasId =
    printfn "Checking ID: %b" hasId
    hasId

// && short-circuits: ถ้าตัวแรก false ไม่ evaluate ตัวที่สอง
let canEnter1 = checkAge 15 && checkId true
// จะ print "Checking age 15..." แต่ไม่ print "Checking ID..." เพราะ short-circuit

printfn "---"

// || short-circuits: ถ้าตัวแรก true ไม่ evaluate ตัวที่สอง  
let canEnter2 = checkAge 20 || checkId false
// จะ print "Checking age 20..." แต่ไม่ print "Checking ID..."
```

---

## 2.7 Type Annotations

```fsharp
// F# สามารถ infer types ได้ แต่บางครั้งต้องระบุชัดเจน

// ไม่ต้องระบุ type (inferred)
let x = 42            // int
let y = 3.14          // float
let z = "hello"       // string

// ระบุ type ชัดเจน
let a: int = 42
let b: float = 3.14
let c: string = "hello"
let d: bool = true

// Function type annotations
let add (x: int) (y: int) : int = x + y
let concat (s1: string) (s2: string) : string = s1 + s2

// Type annotations ช่วยเมื่อ type inference ไม่ทำงาน
let divide (a: float) (b: float) = a / b

// Generic type annotation
let identity<'T> (x: 'T) : 'T = x

// ตัวอย่างที่ต้องการ type annotation
let emptyList: int list = []
let emptyArray: string array = [||]
let optionValue: int option = Some 42
let noneValue: string option = None
```

---

## 2.8 Shadowing (การบังเงา)

```fsharp
// Shadowing: กำหนด binding ใหม่ด้วยชื่อเดิม
let value = 10
printfn "Original: %d" value

let value = value + 5    // Shadow: value ใหม่ = 15
printfn "After shadow: %d" value

let value = value * 2    // Shadow again: value ใหม่ = 30
printfn "After second shadow: %d" value

// Shadowing ใน nested scopes
let outerValue = 100

let exampleFunction () =
    let outerValue = 200   // Shadow ภายใน function
    printfn "Inner: %d" outerValue  // 200
    
    let result =
        let outerValue = 300   // Shadow ภายใน let
        outerValue * 2          // 600
    
    printfn "Result: %d" result  // 600
    printfn "After inner scope: %d" outerValue  // 200 (ยังคงเป็น 200)

exampleFunction ()
printfn "Outer: %d" outerValue  // 100 (ยังคงเป็น 100)
```

---

## 2.9 Underscore _ (การละเว้นค่า)

```fsharp
// _ ใช้สำหรับ ignore values ที่ไม่ต้องการ

// ใน pattern matching
let tuple = (1, 2, 3)
let (first, _, third) = tuple   // ละเว้นค่าที่ 2
printfn "First: %d, Third: %d" first third

// ใน function parameters
let printFirst (x, _) = printfn "First: %d" x
printFirst (42, "ignored")

// ใน loop
for _ in 1..5 do
    printfn "Repeating..."  // ไม่สนใจค่า index

// ใน match
let classify n =
    match n with
    | 0 -> "zero"
    | 1 | 2 | 3 -> "small"
    | _ -> "large"   // wildcard - match ทุกค่าที่เหลือ

// Return unit แสดงว่าไม่สนใจ return value
let _ = System.Console.ReadLine()   // อ่านแต่ไม่เก็บ

// ใน tuple deconstruction
let getCoordinates () = (10.5, 20.3, 100.0)
let (lat, long, _) = getCoordinates ()  // ละเว้น altitude
printfn "Lat: %f, Long: %f" lat long
```

---

## 2.10 printfn, printf, sprintf

### 2.10.1 Format Specifiers

```fsharp
// %d - integer (decimal)
printfn "Integer: %d" 42
printfn "Negative: %d" -17

// %i - integer (เหมือน %d)
printfn "Integer: %i" 42

// %f - float
printfn "Float: %f" 3.14159

// %e - scientific notation
printfn "Scientific: %e" 123456.789

// %g - ใช้ %f หรือ %e อัตโนมัติ (whichever is shorter)
printfn "Auto: %g" 0.0000001
printfn "Auto: %g" 1234.5

// %s - string
printfn "String: %s" "hello"

// %b - boolean
printfn "Bool: %b" true

// %c - character
printfn "Char: %c" 'A'

// %A - any value (uses F# pretty printing)
printfn "Any: %A" [1; 2; 3]
printfn "Any: %A" (Some 42)
printfn "Any tuple: %A" (1, "two", 3.0)

// %O - uses .ToString()
printfn "Object: %O" System.DateTime.Now
```

### 2.10.2 Formatting Numbers

```fsharp
// Width and alignment
printfn "%10d" 42         // "        42" (right-aligned, width 10)
printfn "%-10d|" 42       // "42        |" (left-aligned, width 10)
printfn "%010d" 42        // "0000000042" (zero-padded, width 10)

// Float precision
printfn "%.2f" 3.14159    // "3.14" (2 decimal places)
printfn "%.4f" 3.14159    // "3.1416" (4 decimal places)
printfn "%8.2f" 3.14159   // "    3.14" (width 8, 2 decimals)

// String width
printfn "%10s" "hi"       // "        hi" (right-aligned)
printfn "%-10s|" "hi"     // "hi        |" (left-aligned)
```

### 2.10.3 sprintf สำหรับ String Formatting

```fsharp
// sprintf return string แทนที่จะ print
let formatted = sprintf "Hello, %s! You are %d years old." "Alice" 30
printfn "%s" formatted

// ใช้สร้าง formatted strings
let formatCurrency amount =
    sprintf "฿%.2f" amount

let formatPercent value =
    sprintf "%.1f%%" (value * 100.0)

printfn "%s" (formatCurrency 1234.56)    // ฿1234.56
printfn "%s" (formatPercent 0.0756)      // 7.6%

// สร้าง table
let printTable data =
    printfn "%-20s %10s %8s" "Name" "Price" "Qty"
    printfn "%s" (String.replicate 40 "-")
    for (name, price, qty) in data do
        printfn "%-20s %10.2f %8d" name price qty

let products = [
    ("Widget A", 29.99, 100)
    ("Super Widget B", 49.99, 50)
    ("Mini Widget", 9.99, 200)
]

printTable products
```

### 2.10.4 eprintfn สำหรับ Error Output

```fsharp
// eprintfn เขียนไปยัง stderr
eprintfn "This is an error message"
eprintfn "Error code: %d" 404

// printfn เขียนไปยัง stdout
printfn "This is normal output"
```

---

## 2.11 Console Input (รับ Input จากผู้ใช้)

```fsharp
// รับ input จาก console
printf "กรุณาใส่ชื่อ: "
let inputName = System.Console.ReadLine()
printfn "สวัสดี, %s!" inputName

// รับตัวเลข
printf "กรุณาใส่อายุ: "
let ageStr = System.Console.ReadLine()
let age = int ageStr   // แปลงเป็น int

if age >= 18 then
    printfn "คุณเป็นผู้ใหญ่"
else
    printfn "คุณยังเป็นเด็ก"

// รับ input อย่างปลอดภัย (กัน exception)
let tryReadInt prompt =
    printf "%s" prompt
    let input = System.Console.ReadLine()
    match System.Int32.TryParse(input) with
    | true, value -> Some value
    | false, _ -> None

match tryReadInt "กรุณาใส่ตัวเลข: " with
| Some n -> printfn "คุณใส่: %d" n
| None -> printfn "ไม่ใช่ตัวเลขที่ถูกต้อง"
```

---

## 2.12 if/elif/else Expressions

```fsharp
// if เป็น expression ใน F# (มี return value)

// Basic if/else
let isPositive n =
    if n > 0 then "positive"
    else "non-positive"

printfn "%s" (isPositive 5)
printfn "%s" (isPositive -3)

// if/elif/else
let grade score =
    if score >= 90 then "A"
    elif score >= 80 then "B"
    elif score >= 70 then "C"
    elif score >= 60 then "D"
    else "F"

for score in [95; 83; 72; 65; 45] do
    printfn "Score %d -> Grade %s" score (grade score)

// if expression ต้องการ else ถ้าใช้เป็น expression
let message = 
    if true then "yes"
    else "no"    // else จำเป็น

// if ที่ return unit ไม่ต้องการ else
let checkValue n =
    if n > 100 then
        printfn "%d is large" n  // return unit

// Nested if
let classifyTemperature temp =
    if temp < 0.0 then
        "freezing"
    elif temp < 10.0 then
        "cold"
    elif temp < 20.0 then
        "cool"
    elif temp < 30.0 then
        "comfortable"
    elif temp < 40.0 then
        "hot"
    else
        "very hot"

for temp in [-5.0; 5.0; 15.0; 25.0; 35.0; 45.0] do
    printfn "%.0f°C: %s" temp (classifyTemperature temp)
```

---

## 2.13 for และ while Loops

```fsharp
// for..in loop
printfn "\n=== for..in ==="
for i in 1..5 do
    printfn "i = %d" i

// for..in กับ step
printfn "\n=== for..in with step ==="
for i in 0..2..10 do
    printfn "i = %d" i

// Countdown
printfn "\n=== countdown ==="
for i in 5..-1..1 do
    printfn "%d..." i
printfn "Go!"

// for..in กับ list
printfn "\n=== for..in list ==="
let fruits = ["apple"; "banana"; "cherry"]
for fruit in fruits do
    printfn "Fruit: %s" fruit

// while loop
printfn "\n=== while ==="
let mutable count = 0
while count < 5 do
    printfn "Count: %d" count
    count <- count + 1

// do..while equivalent (F# ไม่มี do..while โดยตรง)
printfn "\n=== do..while equivalent ==="
let mutable running = true
let mutable attempts = 0
while running do
    attempts <- attempts + 1
    printfn "Attempt %d" attempts
    if attempts >= 3 then
        running <- false
```

---

## 2.14 ตัวอย่างโปรแกรมสมบูรณ์

### 2.14.1 โปรแกรมคำนวณเกรด

```fsharp
// grade_calculator.fsx

open System

// Type aliases
type Score = float
type Grade = string

// คำนวณเกรด
let getGrade (score: Score) : Grade =
    if score >= 80.0 then "A"
    elif score >= 70.0 then "B"
    elif score >= 60.0 then "C"
    elif score >= 50.0 then "D"
    else "F"

// ข้อมูลนักเรียน
let students = [
    ("สมชาย", 85.5)
    ("สมหญิง", 72.0)
    ("มานะ", 91.0)
    ("มานี", 45.5)
    ("ปิติ", 63.0)
]

// แสดงผลในรูปแบบตาราง
printfn "%-15s %10s %5s" "ชื่อ" "คะแนน" "เกรด"
printfn "%s" (String.replicate 32 "-")

for (name, score) in students do
    let grade = getGrade score
    printfn "%-15s %10.1f %5s" name score grade

// สถิติ
let scores = students |> List.map snd
let average = scores |> List.average
let maxScore = scores |> List.max
let minScore = scores |> List.min
let passing = students |> List.filter (fun (_, s) -> s >= 50.0) |> List.length

printfn "%s" (String.replicate 32 "-")
printfn "คะแนนเฉลี่ย: %.2f" average
printfn "คะแนนสูงสุด: %.1f" maxScore
printfn "คะแนนต่ำสุด: %.1f" minScore
printfn "ผ่าน: %d/%d คน" passing (List.length students)
```

### 2.14.2 โปรแกรมตรวจสอบ Password

```fsharp
// password_checker.fsx

let checkPassword (password: string) =
    let hasLength = password.Length >= 8
    let hasUpper = password |> Seq.exists System.Char.IsUpper
    let hasLower = password |> Seq.exists System.Char.IsLower
    let hasDigit = password |> Seq.exists System.Char.IsDigit
    let hasSpecial = password |> Seq.exists (fun c -> "!@#$%^&*".Contains(c))
    
    let strength =
        [hasLength; hasUpper; hasLower; hasDigit; hasSpecial]
        |> List.filter id
        |> List.length
    
    let messages = [
        if not hasLength then yield "❌ ต้องมีอย่างน้อย 8 ตัวอักษร"
        if not hasUpper then yield "❌ ต้องมีตัวพิมพ์ใหญ่"
        if not hasLower then yield "❌ ต้องมีตัวพิมพ์เล็ก"
        if not hasDigit then yield "❌ ต้องมีตัวเลข"
        if not hasSpecial then yield "❌ ต้องมีอักขระพิเศษ (!@#$%^&*)"
    ]
    
    let strengthLabel =
        match strength with
        | 5 -> "แข็งแรงมาก 💪"
        | 4 -> "แข็งแรง 👍"
        | 3 -> "ปานกลาง 😐"
        | 2 -> "อ่อนแอ 😕"
        | _ -> "อ่อนแอมาก ❌"
    
    (strengthLabel, messages)

let testPasswords = [
    "abc"
    "password123"
    "Password1"
    "P@ssw0rd"
    "Str0ng!Pass"
]

printfn "=== ตรวจสอบความแข็งแรงของ Password ==="
for pw in testPasswords do
    let (strength, messages) = checkPassword pw
    printfn "\nPassword: %s" pw
    printfn "ความแข็งแรง: %s" strength
    if messages.Length > 0 then
        messages |> List.iter (printfn "  %s")
```

### 2.14.3 โปรแกรม Number Guessing Game

```fsharp
// guessing_game.fsx

open System

let random = Random()

let playGame () =
    let target = random.Next(1, 101)  // 1-100
    let mutable attempts = 0
    let mutable won = false
    
    printfn "=== เกมทายตัวเลข ==="
    printfn "ฉันคิดตัวเลขระหว่าง 1-100"
    printfn "คุณมี 10 ครั้ง!"
    printfn ""
    
    while not won && attempts < 10 do
        attempts <- attempts + 1
        printf "ครั้งที่ %d - ทาย: " attempts
        
        let input = Console.ReadLine()
        match Int32.TryParse(input) with
        | true, guess ->
            if guess = target then
                printfn "🎉 ถูกต้อง! ตัวเลขคือ %d!" target
                printfn "คุณใช้ %d ครั้ง" attempts
                won <- true
            elif guess < target then
                printfn "📈 มากกว่านี้!"
            else
                printfn "📉 น้อยกว่านี้!"
        | _ ->
            printfn "กรุณาใส่ตัวเลขที่ถูกต้อง"
            attempts <- attempts - 1  // ไม่นับครั้งนี้
    
    if not won then
        printfn "\n😔 หมดสิทธิ์แล้ว! ตัวเลขคือ %d" target

// playGame ()  // uncomment เพื่อเล่น
printfn "โปรแกรมพร้อมเล่น! เรียก playGame() เพื่อเริ่ม"
```

---

## สรุป Part 2

ในบทนี้เราได้เรียนรู้:
- ✅ let bindings - การกำหนดค่าแบบ immutable
- ✅ mutable variables - เมื่อต้องการเปลี่ยนค่า
- ✅ Significant whitespace - การเยื้องบรรทัด
- ✅ Comments - single-line, multi-line, XML docs
- ✅ String interpolation และ verbatim strings
- ✅ Arithmetic, comparison, boolean operators
- ✅ Integer division และ modulo
- ✅ Type annotations
- ✅ Shadowing
- ✅ Underscore _ สำหรับ ignored values
- ✅ printfn, printf, sprintf formatting
- ✅ Console input/output
- ✅ if/elif/else expressions
- ✅ for และ while loops

**ใน Part 3** เราจะเรียนรู้เกี่ยวกับ types และ type inference อย่างละเอียด!
