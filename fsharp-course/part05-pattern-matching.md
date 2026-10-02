# Part 5 - การจับคู่รูปแบบ (Pattern Matching)

## บทนำ

Pattern matching เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ F# มันช่วยให้เขียนโค้ดที่ express intent ได้ชัดเจนและปลอดภัย เพราะ compiler ตรวจสอบว่าเราจัดการทุก case ครบถ้วน ต่างจาก if/else ที่ง่ายต่อการลืม case

---

## 5.1 match Expression พื้นฐาน

### 5.1.1 Basic Match

```fsharp
// syntax:
// match expression with
// | pattern1 -> result1
// | pattern2 -> result2
// | _ -> default

let describe n =
    match n with
    | 0 -> "zero"
    | 1 -> "one"
    | 2 -> "two"
    | _ -> "many"

printfn "%s" (describe 0)    // zero
printfn "%s" (describe 1)    // one
printfn "%s" (describe 5)    // many

// Match กับ string
let greetLanguage lang =
    match lang with
    | "th" -> "สวัสดี"
    | "en" -> "Hello"
    | "jp" -> "こんにちは"
    | "cn" -> "你好"
    | other -> $"Hello in {other}"

printfn "%s" (greetLanguage "th")
printfn "%s" (greetLanguage "en")
printfn "%s" (greetLanguage "ko")
```

### 5.1.2 Match เป็น Expression

```fsharp
// match return ค่า ทำให้ใช้เป็น expression ได้

let dayType day =
    match day with
    | "Saturday" | "Sunday" -> "Weekend"
    | "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday" -> "Weekday"
    | _ -> "Unknown"

// ใช้ใน let binding
let today = "Monday"
let typeOfDay = match today with
                | "Saturday" | "Sunday" -> "Weekend"
                | _ -> "Weekday"
printfn "%s is a %s" today typeOfDay

// ใช้ใน function argument
printfn "%s" (match 42 with | n when n > 0 -> "positive" | _ -> "non-positive")
```

---

## 5.2 Constant Patterns

```fsharp
// Match กับค่าคงที่ (integers, strings, booleans, etc.)

let httpStatus code =
    match code with
    | 200 -> "OK"
    | 201 -> "Created"
    | 204 -> "No Content"
    | 301 -> "Moved Permanently"
    | 302 -> "Found"
    | 400 -> "Bad Request"
    | 401 -> "Unauthorized"
    | 403 -> "Forbidden"
    | 404 -> "Not Found"
    | 500 -> "Internal Server Error"
    | 503 -> "Service Unavailable"
    | code -> $"Unknown status code: {code}"

for code in [200; 404; 500; 999] do
    printfn "%d: %s" code (httpStatus code)

// Match กับ bool
let boolToThai b =
    match b with
    | true -> "ใช่"
    | false -> "ไม่ใช่"

printfn "true = %s" (boolToThai true)
printfn "false = %s" (boolToThai false)

// Match กับ char
let charCategory c =
    match c with
    | 'A' | 'E' | 'I' | 'O' | 'U' -> "uppercase vowel"
    | 'a' | 'e' | 'i' | 'o' | 'u' -> "lowercase vowel"
    | c when System.Char.IsUpper(c) -> "uppercase consonant"
    | c when System.Char.IsLower(c) -> "lowercase consonant"
    | c when System.Char.IsDigit(c) -> "digit"
    | _ -> "other"

for c in ['A'; 'e'; 'Z'; 'g'; '5'; '@'] do
    printfn "'%c': %s" c (charCategory c)
```

---

## 5.3 Variable Patterns

```fsharp
// Variable pattern: จับค่าใส่ตัวแปร

let showNumber n =
    match n with
    | 0 -> printfn "Zero"
    | n -> printfn "Number: %d" n  // n จับค่าที่ match

showNumber 0
showNumber 42

// ใช้ variable ใน guard
let classifyAge age =
    match age with
    | a when a < 0 -> "Invalid age"
    | a when a < 13 -> $"Child ({a} years)"
    | a when a < 18 -> $"Teenager ({a} years)"
    | a when a < 65 -> $"Adult ({a} years)"
    | a -> $"Senior ({a} years)"

for age in [-1; 5; 15; 30; 70] do
    printfn "%s" (classifyAge age)

// Variable ใน tuple pattern
let describePair (pair: int * int) =
    match pair with
    | (0, 0) -> "origin"
    | (x, 0) -> $"x-axis at {x}"
    | (0, y) -> $"y-axis at {y}"
    | (x, y) when x = y -> $"diagonal at ({x}, {y})"
    | (x, y) -> $"point at ({x}, {y})"

printfn "%s" (describePair (0, 0))
printfn "%s" (describePair (5, 0))
printfn "%s" (describePair (3, 3))
printfn "%s" (describePair (2, 7))
```

---

## 5.4 Wildcard Pattern (_)

```fsharp
// _ matches ทุกค่า แต่ไม่ bind ให้กับตัวแปร

let justGrade score =
    match score with
    | s when s >= 90 -> "A"
    | s when s >= 80 -> "B"
    | s when s >= 70 -> "C"
    | _ -> "Below C"  // ไม่สนใจค่า

// _ ใน tuple: ละเว้น elements ที่ไม่สนใจ
let getX (x, _, _) = x
let getY (_, y, _) = y
let getZ (_, _, z) = z

let point3D = (1, 2, 3)
printfn "X: %d, Y: %d, Z: %d" (getX point3D) (getY point3D) (getZ point3D)

// _ ใน nested patterns
let processData data =
    match data with
    | (name, _, age) when age >= 18 ->
        printfn "%s is an adult" name
    | (name, _, _) ->
        printfn "%s is not an adult" name

processData ("Alice", "alice@email.com", 25)
processData ("Bob", "bob@email.com", 15)
```

---

## 5.5 Tuple Patterns

```fsharp
// Pattern matching กับ tuples

// 2-tuple
let addOrMultiply operation a b =
    match operation with
    | "add" -> a + b
    | "multiply" -> a * b
    | _ -> failwith "Unknown operation"

// Match tuple โดยตรง
let describePoint point =
    match point with
    | (0, 0) -> "origin"
    | (x, 0) -> $"on x-axis ({x})"
    | (0, y) -> $"on y-axis ({y})"
    | (x, y) when x > 0 && y > 0 -> $"quadrant I ({x}, {y})"
    | (x, y) when x < 0 && y > 0 -> $"quadrant II ({x}, {y})"
    | (x, y) when x < 0 && y < 0 -> $"quadrant III ({x}, {y})"
    | (x, y) -> $"quadrant IV ({x}, {y})"

let points = [(0,0); (3,4); (-2,5); (-1,-3); (4,-2)]
points |> List.iter (fun p -> printfn "%A -> %s" p (describePoint p))

// 3-tuple
let describeRGB (r, g, b) =
    match (r, g, b) with
    | (255, 0, 0) -> "Red"
    | (0, 255, 0) -> "Green"
    | (0, 0, 255) -> "Blue"
    | (255, 255, 0) -> "Yellow"
    | (0, 255, 255) -> "Cyan"
    | (255, 0, 255) -> "Magenta"
    | (255, 255, 255) -> "White"
    | (0, 0, 0) -> "Black"
    | (r, g, b) when r = g && g = b -> $"Gray (value={r})"
    | (r, g, b) -> $"Custom RGB({r}, {g}, {b})"

for color in [(255,0,0); (128,128,128); (100,50,200)] do
    printfn "%A -> %s" color (describeRGB color)
```

---

## 5.6 List Patterns

```fsharp
// Pattern matching กับ lists

// Empty list
let describeList lst =
    match lst with
    | [] -> "empty"
    | [x] -> $"singleton: {x}"
    | [x; y] -> $"pair: {x}, {y}"
    | [x; y; z] -> $"triple: {x}, {y}, {z}"
    | x :: rest -> $"head={x}, tail length={List.length rest}"

printfn "%s" (describeList [])
printfn "%s" (describeList [1])
printfn "%s" (describeList [1; 2])
printfn "%s" (describeList [1; 2; 3])
printfn "%s" (describeList [1; 2; 3; 4; 5])

// Cons pattern (::)
let rec sumList lst =
    match lst with
    | [] -> 0
    | head :: tail -> head + sumList tail

printfn "Sum [1..5]: %d" (sumList [1; 2; 3; 4; 5])

// Pattern ที่ซับซ้อน
let rec containsDuplicate lst =
    match lst with
    | [] | [_] -> false
    | x :: y :: _ when x = y -> true
    | _ :: rest -> containsDuplicate rest

printfn "Has dup [1;2;3;3;4]: %b" (containsDuplicate [1;2;3;3;4])
printfn "Has dup [1;2;3;4;5]: %b" (containsDuplicate [1;2;3;4;5])

// First and last
let firstAndLast lst =
    match lst with
    | [] -> None
    | [x] -> Some (x, x)
    | x :: rest -> Some (x, List.last rest)

printfn "First/Last []: %A" (firstAndLast<int> [])
printfn "First/Last [1]: %A" (firstAndLast [1])
printfn "First/Last [1..5]: %A" (firstAndLast [1..5])

// Decode list operations
let rec interleave xs ys =
    match xs, ys with
    | [], ys -> ys
    | xs, [] -> xs
    | x :: xs', y :: ys' -> x :: y :: interleave xs' ys'

printfn "Interleave: %A" (interleave [1;3;5] [2;4;6])
```

---

## 5.7 Array Patterns

```fsharp
// Pattern matching กับ arrays

let describeArray arr =
    match arr with
    | [||] -> "empty array"
    | [| x |] -> $"single element: {x}"
    | [| x; y |] -> $"two elements: {x}, {y}"
    | arr -> $"array with {arr.Length} elements, first={arr.[0]}"

printfn "%s" (describeArray [||])
printfn "%s" (describeArray [| 42 |])
printfn "%s" (describeArray [| 1; 2 |])
printfn "%s" (describeArray [| 1; 2; 3; 4 |])

// Array pattern ใน function
let processArgs (args: string[]) =
    match args with
    | [||] ->
        printfn "No arguments"
    | [| cmd |] ->
        printfn "Command: %s" cmd
    | [| cmd; arg |] ->
        printfn "Command: %s, Argument: %s" cmd arg
    | args ->
        printfn "Command: %s, %d arguments" args.[0] (args.Length - 1)

processArgs [||]
processArgs [| "run" |]
processArgs [| "copy"; "file.txt" |]
processArgs [| "convert"; "input.txt"; "output.pdf"; "--verbose" |]
```

---

## 5.8 Record Patterns

```fsharp
// Pattern matching กับ record types

type Person = {
    Name: string
    Age: int
    Email: string option
}

let describePerson person =
    match person with
    | { Name = "Admin"; Age = _ } ->
        "Administrator account"
    | { Name = name; Age = age } when age < 18 ->
        $"{name} is a minor"
    | { Name = name; Email = Some email } ->
        $"{name} (email: {email})"
    | { Name = name; Email = None } ->
        $"{name} (no email)"

let people = [
    { Name = "Admin"; Age = 99; Email = Some "admin@system.com" }
    { Name = "Alice"; Age = 15; Email = None }
    { Name = "Bob"; Age = 30; Email = Some "bob@email.com" }
    { Name = "Charlie"; Age = 25; Email = None }
]

people |> List.iter (fun p -> printfn "%s" (describePerson p))

// Destructuring in let
let { Name = personName; Age = personAge } = people.[2]
printfn "Name: %s, Age: %d" personName personAge

// Record pattern กับ nested records
type Address = { City: string; Country: string }
type Employee = { Name: string; Address: Address; Salary: float }

let classifyEmployee emp =
    match emp with
    | { Address = { Country = "Thailand" }; Salary = s } when s > 100000.0 ->
        "Thai high earner"
    | { Address = { Country = "Thailand" } } ->
        "Thai employee"
    | { Address = { Country = country } } ->
        $"International employee from {country}"

let employees = [
    { Name = "สมชาย"; Address = { City = "Bangkok"; Country = "Thailand" }; Salary = 120000.0 }
    { Name = "สมหญิง"; Address = { City = "Chiang Mai"; Country = "Thailand" }; Salary = 80000.0 }
    { Name = "John"; Address = { City = "New York"; Country = "USA" }; Salary = 150000.0 }
]

employees |> List.iter (fun e -> printfn "%s: %s" e.Name (classifyEmployee e))
```

---

## 5.9 Discriminated Union Patterns

```fsharp
// Pattern matching กับ Discriminated Unions (DUs)

type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float
    | Point

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(b, h) -> 0.5 * b * h
    | Point -> 0.0

let perimeter shape =
    match shape with
    | Circle r -> 2.0 * System.Math.PI * r
    | Rectangle(w, h) -> 2.0 * (w + h)
    | Triangle(b, h) ->
        let hypotenuse = sqrt (b * b + h * h)
        b + h + hypotenuse
    | Point -> 0.0

let describeShape shape =
    match shape with
    | Circle r -> $"Circle with radius {r:.2f}, area={area shape:.4f}"
    | Rectangle(w, h) -> $"Rectangle {w}x{h}, area={area shape:.2f}"
    | Triangle(b, h) -> $"Triangle base={b}, height={h}, area={area shape:.2f}"
    | Point -> "Point (no area)"

let shapes = [Circle 5.0; Rectangle(4.0, 6.0); Triangle(3.0, 4.0); Point]
shapes |> List.iter (fun s -> printfn "%s" (describeShape s))

// Traffic light DU
type TrafficLight = Red | Yellow | Green

let action light =
    match light with
    | Red -> "Stop"
    | Yellow -> "Slow down"
    | Green -> "Go"

let nextLight light =
    match light with
    | Red -> Green
    | Green -> Yellow
    | Yellow -> Red

let lights = [Red; Green; Yellow]
for light in lights do
    printfn "%A -> %s (next: %A)" light (action light) (nextLight light)
```

---

## 5.10 Type Test Patterns (:?)

```fsharp
// :? สำหรับ type testing

let describeObject (obj: obj) =
    match obj with
    | :? int as i -> $"int: {i}"
    | :? float as f -> $"float: {f}"
    | :? string as s -> $"string: '{s}'"
    | :? bool as b -> $"bool: {b}"
    | :? (int list) as lst -> $"int list with {lst.Length} elements"
    | null -> "null"
    | _ -> $"unknown type: {obj.GetType().Name}"

printfn "%s" (describeObject (42 :> obj))
printfn "%s" (describeObject (3.14 :> obj))
printfn "%s" (describeObject ("hello" :> obj))
printfn "%s" (describeObject (true :> obj))
printfn "%s" (describeObject ([1;2;3] :> obj))
printfn "%s" (describeObject (null :> obj))

// Type test ใน exception handling
let handleException (ex: System.Exception) =
    match ex with
    | :? System.ArgumentNullException as e ->
        $"Null argument: {e.ParamName}"
    | :? System.ArgumentOutOfRangeException as e ->
        $"Out of range: {e.ParamName}"
    | :? System.InvalidOperationException ->
        "Invalid operation"
    | :? System.IO.IOException as e ->
        $"IO error: {e.Message}"
    | e ->
        $"General error: {e.Message}"

let ex1 = System.ArgumentNullException("myParam")
let ex2 = System.InvalidOperationException("Can't do that")
printfn "%s" (handleException ex1)
printfn "%s" (handleException ex2)
```

---

## 5.11 AND Patterns (&)

```fsharp
// & รวม patterns - ค่าต้อง match ทั้งสอง pattern

// ตรวจสอบทั้ง value และ property
let checkEvenPositive n =
    match n with
    | x & _ when x > 0 && x % 2 = 0 -> $"{x} is positive and even"
    | x & _ when x > 0 -> $"{x} is positive but odd"
    | x & _ when x = 0 -> "zero"
    | x -> $"{x} is negative"

for n in [-4; -1; 0; 3; 6; 8] do
    printfn "%s" (checkEvenPositive n)

// AND pattern กับ DU
type Message =
    | Text of string
    | Number of int
    | Empty

let processMessage msg =
    match msg with
    | Text s & _ when s.Length > 10 -> $"Long text: {s}"
    | Text s -> $"Short text: {s}"
    | Number n & _ when n > 100 -> $"Large number: {n}"
    | Number n -> $"Small number: {n}"
    | Empty -> "Empty message"

let messages = [Text "Hi"; Text "This is a long message"; Number 42; Number 200; Empty]
messages |> List.iter (fun m -> printfn "%s" (processMessage m))
```

---

## 5.12 OR Patterns (|)

```fsharp
// | ใน patterns - ค่า match ถ้าตรงกับ pattern ใด pattern หนึ่ง

// Match หลายค่า
let isWeekend day =
    match day with
    | "Saturday" | "Sunday" -> true
    | _ -> false

// หลาย cases ทำสิ่งเดียวกัน
let classify n =
    match n with
    | 0 | 1 | 2 -> "small"
    | 3 | 4 | 5 -> "medium"
    | 6 | 7 | 8 | 9 -> "large"
    | _ -> "very large"

for n in [0; 1; 4; 7; 15] do
    printfn "%d: %s" n (classify n)

// OR pattern กับ DU
type Fruit = Apple | Orange | Banana | Grape | Mango

let isTropical fruit =
    match fruit with
    | Mango | Banana -> true   // OR pattern
    | Apple | Orange | Grape -> false

let fruits = [Apple; Banana; Mango; Orange; Grape]
fruits |> List.iter (fun f -> printfn "%A is tropical: %b" f (isTropical f))
```

---

## 5.13 Guards (when Clause)

```fsharp
// when clause เพิ่มเงื่อนไขใน pattern

let fizzbuzz n =
    match n with
    | n when n % 15 = 0 -> "FizzBuzz"
    | n when n % 3 = 0 -> "Fizz"
    | n when n % 5 = 0 -> "Buzz"
    | n -> string n

[1..20] |> List.iter (fun n -> printf "%s " (fizzbuzz n))
printfn ""

// Guard กับ multiple conditions
let classifyTemperature temp =
    match temp with
    | t when t < -20.0 -> "Extremely cold"
    | t when t < 0.0 -> "Freezing"
    | t when t < 10.0 -> "Cold"
    | t when t < 20.0 -> "Cool"
    | t when t < 30.0 -> "Warm"
    | t when t < 40.0 -> "Hot"
    | _ -> "Extremely hot"

for temp in [-30.0; -5.0; 5.0; 15.0; 25.0; 35.0; 45.0] do
    printfn "%.0f°C: %s" temp (classifyTemperature temp)

// Guard กับ complex conditions
type Student = { Name: string; Grade: float; Attendance: float }

let getScholarship student =
    match student with
    | { Grade = g; Attendance = a } when g >= 3.5 && a >= 90.0 ->
        "Full scholarship"
    | { Grade = g } when g >= 3.0 ->
        "Partial scholarship"
    | { Attendance = a } when a >= 95.0 ->
        "Attendance award"
    | _ ->
        "No scholarship"

let students = [
    { Name = "Alice"; Grade = 3.8; Attendance = 95.0 }
    { Name = "Bob"; Grade = 3.2; Attendance = 85.0 }
    { Name = "Charlie"; Grade = 2.5; Attendance = 98.0 }
    { Name = "Diana"; Grade = 2.0; Attendance = 70.0 }
]

students |> List.iter (fun s -> printfn "%s: %s" s.Name (getScholarship s))
```

---

## 5.14 Nested Patterns

```fsharp
// Patterns สามารถ nest ได้หลายชั้น

type Address = { City: string; PostCode: string }
type Customer = {
    Name: string
    Address: Address option
    Orders: int list
}

let describeCustomer customer =
    match customer with
    | { Name = name; Address = Some { City = "Bangkok"; PostCode = pc }; Orders = [] } ->
        $"{name} from Bangkok ({pc}) - no orders yet"
    | { Name = name; Address = Some { City = city }; Orders = orders } ->
        $"{name} from {city} - {orders.Length} orders"
    | { Name = name; Address = None; Orders = orders } when orders.Length > 5 ->
        $"{name} (no address) - loyal customer with {orders.Length} orders"
    | { Name = name; Address = None } ->
        $"{name} - no address on file"

let customers = [
    { Name = "สมชาย"; Address = Some { City = "Bangkok"; PostCode = "10100" }; Orders = [] }
    { Name = "สมหญิง"; Address = Some { City = "Chiang Mai"; PostCode = "50000" }; Orders = [1;2;3] }
    { Name = "มานะ"; Address = None; Orders = [1;2;3;4;5;6;7] }
    { Name = "มานี"; Address = None; Orders = [1;2] }
]

customers |> List.iter (fun c -> printfn "%s" (describeCustomer c))

// Deeply nested list patterns
let rec processTree tree =
    match tree with
    | [] -> "empty"
    | [x] -> $"leaf: {x}"
    | [[x; y]] -> $"nested pair: {x}, {y}"
    | head :: tail ->
        $"node {head}, subtrees: {tail.Length}"

// Complex nested
let parseCommand (parts: string list) =
    match parts with
    | [] -> "Empty command"
    | ["help"] -> "Show help"
    | ["exit"] | ["quit"] -> "Exit program"
    | ["set"; key; value] -> $"Set {key} = {value}"
    | ["get"; key] -> $"Get {key}"
    | ["list"; category] when category = "all" -> "List all items"
    | ["list"; category] -> $"List {category}"
    | cmd :: _ -> $"Unknown command: {cmd}"

let commands = [
    []
    ["help"]
    ["set"; "name"; "Alice"]
    ["get"; "age"]
    ["list"; "all"]
    ["list"; "products"]
    ["unknown"; "stuff"]
]

commands |> List.iter (fun cmd -> printfn "%A -> %s" cmd (parseCommand cmd))
```

---

## 5.15 Exhaustive Pattern Matching

```fsharp
// F# compiler ตรวจสอบว่า pattern matching ครอบคลุมทุก case

type Day = Monday | Tuesday | Wednesday | Thursday | Friday | Saturday | Sunday

// Exhaustive - ครอบคลุมทุก case
let isWorkDay day =
    match day with
    | Monday | Tuesday | Wednesday | Thursday | Friday -> true
    | Saturday | Sunday -> false

// ถ้าขาด case จะได้ warning:
// Warning: Incomplete pattern matches on this expression. 
// For example, the value 'Sunday' will not be matched.

// ใช้ _ เพื่อ catch remaining cases (แต่ระวังอาจ miss bug)
let scheduleMeeting day =
    match day with
    | Monday -> "Monday stand-up"
    | Wednesday -> "Wednesday review"
    | Friday -> "Friday demo"
    | _ -> "No scheduled meeting"

// Option type ต้อง handle ทั้ง Some และ None
let safeDivide a b =
    if b = 0 then None
    else Some (a / b)

let showResult result =
    match result with
    | Some value -> printfn "Result: %d" value
    | None -> printfn "Cannot divide by zero"

showResult (safeDivide 10 2)
showResult (safeDivide 10 0)
```

---

## 5.16 Pattern Matching ใน Let Bindings

```fsharp
// Pattern matching ใช้ใน let binding ได้

// Tuple decomposition
let (x, y) = (10, 20)
printfn "x=%d, y=%d" x y

let (a, b, c) = (1, "two", 3.0)
printfn "a=%d, b=%s, c=%f" a b c

// List decomposition
let [first; second; third] = [1; 2; 3]  // Warning: incomplete!
printfn "first=%d" first

// Record decomposition
type Point = { X: int; Y: int }
let { X = px; Y = py } = { X = 5; Y = 10 }
printfn "Point: (%d, %d)" px py

// ใน function parameters
let addPoints { X = x1; Y = y1 } { X = x2; Y = y2 } =
    { X = x1 + x2; Y = y1 + y2 }

let p1 = { X = 1; Y = 2 }
let p2 = { X = 3; Y = 4 }
let p3 = addPoints p1 p2
printfn "Sum: (%d, %d)" p3.X p3.Y

// DU decomposition (risky if not exhaustive)
type Maybe<'T> = Just of 'T | Nothing

let (Just value) = Just 42  // Warning: incomplete - จะ throw กับ Nothing
printfn "Value: %d" value
```

---

## 5.17 Pattern Matching ใน Function Parameters

```fsharp
// Functions สามารถ pattern match ใน parameters

// Tuple parameter decomposition
let addPair (a, b) = a + b
printfn "addPair (3, 4) = %d" (addPair (3, 4))

// Record parameter decomposition
type Rectangle = { Width: float; Height: float }

let area { Width = w; Height = h } = w * h
let perimeter { Width = w; Height = h } = 2.0 * (w + h)

let rect = { Width = 5.0; Height = 3.0 }
printfn "Area: %f" (area rect)
printfn "Perimeter: %f" (perimeter rect)

// Multiple patterns
let processCoord (x, y) =
    if x = 0 && y = 0 then "origin"
    elif x = 0 then "y-axis"
    elif y = 0 then "x-axis"
    else sprintf "(%d,%d)" x y

// List head/tail in parameter (advanced)
let safeHead = function
    | [] -> None
    | head :: _ -> Some head

printfn "safeHead []: %A" (safeHead<int> [])
printfn "safeHead [1;2;3]: %A" (safeHead [1;2;3])

// function keyword เป็น shorthand สำหรับ fun x -> match x with
let classify = function
    | n when n < 0 -> "negative"
    | 0 -> "zero"
    | n when n < 10 -> "small positive"
    | _ -> "large positive"

for n in [-5; 0; 3; 100] do
    printfn "%d: %s" n (classify n)
```

---

## 5.18 Active Patterns

```fsharp
// Active patterns ช่วยสร้าง custom patterns

// Single case active pattern
let (|Even|Odd|) n =
    if n % 2 = 0 then Even else Odd

let describeNum n =
    match n with
    | Even -> $"{n} is even"
    | Odd -> $"{n} is odd"

for n in [1..6] do
    printfn "%s" (describeNum n)

// Multi-case active pattern
let (|Small|Medium|Large|) n =
    if n < 10 then Small
    elif n < 100 then Medium
    else Large

for n in [5; 50; 500] do
    match n with
    | Small -> printfn "%d is small" n
    | Medium -> printfn "%d is medium" n
    | Large -> printfn "%d is large" n

// Parameterized active pattern
let (|DivisibleBy|_|) divisor n =
    if n % divisor = 0 then Some (n / divisor)
    else None

for n in [1..20] do
    match n with
    | DivisibleBy 15 _ -> printf "FizzBuzz "
    | DivisibleBy 3 _ -> printf "Fizz "
    | DivisibleBy 5 _ -> printf "Buzz "
    | _ -> printf "%d " n
printfn ""

// Partial active pattern สำหรับ parsing
open System

let (|Int|_|) (str: string) =
    match Int32.TryParse(str) with
    | true, n -> Some n
    | false, _ -> None

let (|Float|_|) (str: string) =
    match Double.TryParse(str) with
    | true, f -> Some f
    | false, _ -> None

let parseAndDescribe str =
    match str with
    | Int n -> $"Integer: {n}"
    | Float f -> $"Float: {f}"
    | _ -> $"Not a number: {str}"

for s in ["42"; "3.14"; "hello"; "100"] do
    printfn "'%s' -> %s" s (parseAndDescribe s)
```

---

## 5.19 match vs if/elif

```fsharp
// เมื่อไหร่ควรใช้ match vs if/elif

// Use match when:
// 1. Matching against multiple possible values
// 2. Working with DUs, tuples, records
// 3. Want exhaustiveness checking

// Use if/elif when:
// 1. Simple boolean conditions
// 2. Checking ranges

// match เหมาะสำหรับ
let describeDay = function
    | "Mon" | "Tue" | "Wed" | "Thu" | "Fri" -> "Weekday"
    | "Sat" | "Sun" -> "Weekend"
    | _ -> "Unknown"

// if/elif เหมาะสำหรับ
let describeScore score =
    if score >= 90 then "Excellent"
    elif score >= 70 then "Good"
    elif score >= 50 then "Pass"
    else "Fail"

// ทั้งสองรวมกัน
let fullAnalysis (score: int) (day: string) =
    let performance =
        match score with
        | s when s >= 90 -> "outstanding"
        | s when s >= 70 -> "good"
        | _ -> "needs improvement"
    
    let timing = 
        if day = "Friday" then "end of week"
        else "regular day"
    
    $"Performance: {performance} on {timing} ({day})"

printfn "%s" (fullAnalysis 95 "Monday")
printfn "%s" (fullAnalysis 65 "Friday")
```

---

## 5.20 ตัวอย่างโปรแกรมสมบูรณ์

### 5.20.1 Expression Evaluator

```fsharp
// Simple expression evaluator using pattern matching

type Expr =
    | Num of float
    | Add of Expr * Expr
    | Sub of Expr * Expr
    | Mul of Expr * Expr
    | Div of Expr * Expr
    | Neg of Expr
    | Abs of Expr

let rec eval expr =
    match expr with
    | Num n -> n
    | Add(a, b) -> eval a + eval b
    | Sub(a, b) -> eval a - eval b
    | Mul(a, b) -> eval a * eval b
    | Div(a, b) ->
        let divisor = eval b
        if divisor = 0.0 then
            failwith "Division by zero"
        else
            eval a / divisor
    | Neg e -> -(eval e)
    | Abs e -> abs (eval e)

let rec prettyPrint expr =
    match expr with
    | Num n -> string n
    | Add(a, b) -> $"({prettyPrint a} + {prettyPrint b})"
    | Sub(a, b) -> $"({prettyPrint a} - {prettyPrint b})"
    | Mul(a, b) -> $"({prettyPrint a} * {prettyPrint b})"
    | Div(a, b) -> $"({prettyPrint a} / {prettyPrint b})"
    | Neg e -> $"(-{prettyPrint e})"
    | Abs e -> $"|{prettyPrint e}|"

// (2 + 3) * (10 - 4)
let expr1 = Mul(Add(Num 2.0, Num 3.0), Sub(Num 10.0, Num 4.0))
printfn "%s = %f" (prettyPrint expr1) (eval expr1)

// |(5 - 8)| / 3
let expr2 = Div(Abs(Sub(Num 5.0, Num 8.0)), Num 3.0)
printfn "%s = %f" (prettyPrint expr2) (eval expr2)
```

### 5.20.2 State Machine

```fsharp
// Traffic light state machine with pattern matching

type TrafficState =
    | RedLight of timeRemaining: int
    | YellowLight of timeRemaining: int
    | GreenLight of timeRemaining: int

let tick state =
    match state with
    | RedLight 1 -> GreenLight 30
    | RedLight n -> RedLight (n - 1)
    | YellowLight 1 -> RedLight 20
    | YellowLight n -> YellowLight (n - 1)
    | GreenLight 1 -> YellowLight 5
    | GreenLight n -> GreenLight (n - 1)

let getColor state =
    match state with
    | RedLight _ -> "RED"
    | YellowLight _ -> "YELLOW"
    | GreenLight _ -> "GREEN"

let getTime state =
    match state with
    | RedLight t | YellowLight t | GreenLight t -> t

// Simulate
let mutable current = RedLight 3

printfn "Traffic Light Simulation:"
for _ in 1..20 do
    printfn "%s (%d seconds remaining)" (getColor current) (getTime current)
    current <- tick current
```

---

## สรุป Part 5

ในบทนี้เราได้เรียนรู้:
- ✅ match expression พื้นฐาน
- ✅ Constant patterns
- ✅ Variable patterns
- ✅ Wildcard pattern (_)
- ✅ Tuple patterns
- ✅ List patterns ([], x::xs, [x;y;z])
- ✅ Array patterns
- ✅ Record patterns
- ✅ Discriminated Union patterns
- ✅ Type test patterns (:?)
- ✅ AND patterns (&)
- ✅ OR patterns (|)
- ✅ Guards (when clause)
- ✅ Nested patterns
- ✅ Exhaustive pattern matching
- ✅ Pattern matching ใน let bindings
- ✅ Pattern matching ใน function parameters
- ✅ Active patterns
- ✅ match vs if/elif

**ใน Part 6** เราจะเรียนรู้ Lists และ Collections ที่เป็น backbone ของ functional programming!
