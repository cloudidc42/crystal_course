# Part 4 - ฟังก์ชัน (Functions)

## บทนำ

Functions เป็นหัวใจของการเขียนโปรแกรมแบบ functional ใน F# functions เป็น first-class citizens หมายความว่า function สามารถ:
- ถูก assign ให้กับตัวแปร
- ส่งเป็น argument ให้กับ function อื่น
- Return เป็นผลลัพธ์จาก function
- เก็บใน data structures

ในบทนี้เราจะเรียนรู้การใช้ functions ใน F# อย่างครบถ้วน

---

## 4.1 การนิยาม Functions (Defining Functions)

### 4.1.1 Basic Function Definition

```fsharp
// syntax: let functionName param1 param2 ... = body

// Function ที่ไม่มี parameter (unit function)
let sayHello () =
    printfn "สวัสดี!"

// Function ที่มี 1 parameter
let double x = x * 2

// Function ที่มีหลาย parameters
let add x y = x + y
let multiply x y z = x * y * z

// เรียกใช้
sayHello ()
printfn "%d" (double 5)
printfn "%d" (add 3 4)
printfn "%d" (multiply 2 3 4)
```

### 4.1.2 Function Type Signatures

```fsharp
// F# แสดง function types แบบ: paramType -> returnType
// multi-param: p1 -> p2 -> p3 -> returnType

// ดูใน FSI:
// > let add x y = x + y;;
// val add : x:int -> y:int -> int

// With type annotations
let addFloat (x: float) (y: float) : float = x + y
let formatName (first: string) (last: string) : string =
    $"{first} {last}"

// Type signatures สำหรับ higher-order functions
let applyTwice (f: int -> int) (x: int) : int = f (f x)
let result = applyTwice (fun x -> x + 1) 5  // 7
printfn "applyTwice: %d" result
```

### 4.1.3 Return Values

```fsharp
// F# return ค่าสุดท้ายของ function body โดยอัตโนมัติ
// ไม่ต้องใช้ return keyword

let square x = x * x  // return x * x

let absolute x =
    if x < 0 then -x
    else x

let classify n =
    if n > 0 then "positive"
    elif n < 0 then "negative"
    else "zero"

printfn "square 5 = %d" (square 5)
printfn "absolute -3 = %d" (absolute -3)
printfn "classify 0 = %s" (classify 0)

// Function ที่ return unit (เหมือน void)
let printSquare x =
    let s = square x
    printfn "%d^2 = %d" x s
    // ไม่มี return value - implicit return ()

printSquare 7
```

---

## 4.2 Recursive Functions

### 4.2.1 rec Keyword

```fsharp
// F# ต้องใช้ rec keyword สำหรับ recursive functions
// เพราะ F# ไม่อนุญาตให้ reference function ก่อน define

let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

let rec fibonacci n =
    match n with
    | 0 -> 0
    | 1 -> 1
    | n -> fibonacci (n - 1) + fibonacci (n - 2)

printfn "5! = %d" (factorial 5)      // 120
printfn "10! = %d" (factorial 10)    // 3628800

printfn "fib(0) = %d" (fibonacci 0)  // 0
printfn "fib(10) = %d" (fibonacci 10) // 55
```

### 4.2.2 Tail Recursion

```fsharp
// Tail recursion - recursive call เป็น last operation
// F# optimize tail calls (tail call optimization)

// Non-tail-recursive factorial (stack overflow กับ n ใหญ่)
let rec factNaive n =
    if n <= 1 then 1
    else n * factNaive (n - 1)  // ต้องรอผล multiply หลัง recursive call

// Tail-recursive factorial (ปลอดภัยกับ n ขนาดใหญ่)
let factTailRec n =
    let rec loop acc remaining =
        if remaining <= 1 then acc
        else loop (acc * remaining) (remaining - 1)  // tail call!
    loop 1 n

printfn "factNaive 10 = %d" (factNaive 10)
printfn "factTailRec 10 = %d" (factTailRec 10)
printfn "factTailRec 20 = %d" (factTailRec 20)

// Tail-recursive sum
let sumToN n =
    let rec loop acc i =
        if i > n then acc
        else loop (acc + i) (i + 1)
    loop 0 1

printfn "Sum 1..100 = %d" (sumToN 100)
printfn "Sum 1..1000000 = %d" (sumToN 1000000)  // ไม่ stack overflow!

// Tail-recursive fibonacci (ด้วย accumulator)
let fibTailRec n =
    let rec loop a b count =
        if count = 0 then a
        else loop b (a + b) (count - 1)
    loop 0 1 n

printfn "fib(50) = %d" (fibTailRec 50)
```

### 4.2.3 Mutually Recursive Functions

```fsharp
// and keyword สำหรับ mutually recursive functions
let rec isEven n =
    if n = 0 then true
    else isOdd (n - 1)
and isOdd n =
    if n = 0 then false
    else isEven (n - 1)

printfn "isEven 10 = %b" (isEven 10)
printfn "isOdd 7 = %b" (isOdd 7)

// ตัวอย่างที่ซับซ้อนกว่า
type Tree<'a> =
    | Leaf of 'a
    | Branch of Tree<'a> list

// Mutually recursive tree functions
let rec countLeaves tree =
    match tree with
    | Leaf _ -> 1
    | Branch children -> sumLeaves children
and sumLeaves children =
    children |> List.sumBy countLeaves

let myTree = Branch [
    Leaf 1
    Branch [Leaf 2; Leaf 3]
    Leaf 4
    Branch [Leaf 5; Branch [Leaf 6; Leaf 7]]
]

printfn "Leaves: %d" (countLeaves myTree)  // 7
```

---

## 4.3 Nested Functions

```fsharp
// Functions สามารถ define ภายใน function อื่นได้
let processNumbers numbers =
    // Helper functions ที่ใช้เฉพาะภายใน processNumbers
    let isValidNumber n = n > 0 && n < 1000
    let normalize n = float n / 1000.0
    let format n = sprintf "%.3f" n
    
    numbers
    |> List.filter isValidNumber
    |> List.map normalize
    |> List.map format

let result = processNumbers [-1; 500; 1500; 250; 0; 999]
printfn "Processed: %A" result

// Nested function กับ closure (capture variables)
let makeCounter startValue =
    let mutable count = startValue
    
    let increment () =
        count <- count + 1
        count
    
    let decrement () =
        count <- count - 1
        count
    
    let reset () =
        count <- startValue
    
    let getCount () = count
    
    (increment, decrement, reset, getCount)

let (inc, dec, reset, get) = makeCounter 0
printfn "Initial: %d" (get ())
printfn "After inc: %d" (inc ())
printfn "After inc: %d" (inc ())
printfn "After inc: %d" (inc ())
printfn "After dec: %d" (dec ())
reset ()
printfn "After reset: %d" (get ())
```

---

## 4.4 Lambda Expressions (Anonymous Functions)

### 4.4.1 fun Keyword

```fsharp
// Lambda syntax: fun param -> body
// fun param1 param2 -> body

// Basic lambda
let double = fun x -> x * 2
let add = fun x y -> x + y
let greet = fun name -> $"Hello, {name}!"

printfn "%d" (double 5)
printfn "%d" (add 3 4)
printfn "%s" (greet "Alice")

// Lambda ใน higher-order functions
let numbers = [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]

let evens = numbers |> List.filter (fun x -> x % 2 = 0)
let doubled = numbers |> List.map (fun x -> x * 2)
let sum = numbers |> List.fold (fun acc x -> acc + x) 0

printfn "Evens: %A" evens
printfn "Doubled: %A" doubled
printfn "Sum: %d" sum
```

### 4.4.2 Lambda กับ Multiple Parameters

```fsharp
// Lambda หลาย parameter
let addThree = fun a b c -> a + b + c
printfn "%d" (addThree 1 2 3)

// Lambda ที่ complex
let processItem = fun name price qty ->
    let total = price * float qty
    sprintf "%s: %.2f x %d = %.2f" name price qty total

printfn "%s" (processItem "Widget" 29.99 5)

// Lambda ด้วย pattern matching
let describe = fun (x, y) -> sprintf "(%d, %d)" x y
printfn "%s" (describe (3, 4))

// Lambda ใน pipeline
let result =
    [1..10]
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.fold (fun acc x -> acc + x) 0

printfn "Sum of squares of evens 1-10: %d" result
```

---

## 4.5 Currying และ Partial Application

### 4.5.1 Currying

```fsharp
// F# functions เป็น curried โดยอัตโนมัติ
// add: int -> int -> int
// หมายถึง: function ที่รับ int แล้ว return function ที่รับ int แล้ว return int

let add x y = x + y

// Partial application: ให้ argument บางส่วน
let addFive = add 5      // addFive: int -> int
let addTen = add 10      // addTen: int -> int

printfn "addFive 3 = %d" (addFive 3)   // 8
printfn "addTen 7 = %d" (addTen 7)     // 17

// More examples
let multiply x y = x * y
let double = multiply 2      // double: int -> int
let triple = multiply 3      // triple: int -> int

let numbers = [1; 2; 3; 4; 5]
let doubled = numbers |> List.map double
let tripled = numbers |> List.map triple

printfn "Doubled: %A" doubled
printfn "Tripled: %A" tripled
```

### 4.5.2 Partial Application

```fsharp
// Partial application สร้าง specialized functions จาก general ones

// General: เพิ่ม prefix ให้ string
let addPrefix prefix str = prefix + str

// Specialized versions
let addMr = addPrefix "Mr. "
let addMs = addPrefix "Ms. "
let addDr = addPrefix "Dr. "

printfn "%s" (addMr "Smith")
printfn "%s" (addMs "Jones")
printfn "%s" (addDr "Brown")

// Apply to list
let names = ["Alice"; "Bob"; "Charlie"]
let formalNames = names |> List.map addMs
printfn "Formal names: %A" formalNames

// Practical example: logging
let log level message = printfn "[%s] %s" level message
let logInfo = log "INFO"
let logWarning = log "WARNING"
let logError = log "ERROR"

logInfo "Application started"
logWarning "Low memory"
logError "Database connection failed"

// Filter with partial application
let isGreaterThan threshold n = n > threshold
let isPositive = isGreaterThan 0
let isAbove100 = isGreaterThan 100

let nums = [-5; 0; 50; 150; 200]
printfn "Positive: %A" (nums |> List.filter isPositive)
printfn "Above 100: %A" (nums |> List.filter isAbove100)
```

---

## 4.6 First-class Functions

### 4.6.1 Functions as Values

```fsharp
// Functions สามารถ assign ให้กับ values ได้
let myFunction = fun x -> x * 2
let anotherName = myFunction   // alias

printfn "%d" (myFunction 5)
printfn "%d" (anotherName 5)

// Store functions ใน list
let operations = [
    (fun x -> x + 1)
    (fun x -> x * 2)
    (fun x -> x * x)
]

let applyAll value ops =
    ops |> List.map (fun f -> f value)

let results = applyAll 5 operations
printfn "Apply all to 5: %A" results  // [6; 10; 25]
```

### 4.6.2 Passing Functions as Arguments

```fsharp
// Higher-order functions รับ function เป็น argument

// Custom apply
let apply f x = f x

printfn "%d" (apply (fun x -> x + 1) 5)  // 6
printfn "%s" (apply String.length "hello" |> string)  // Error... let's fix

let applyAndShow (f: int -> int) (x: int) (label: string) =
    let result = f x
    printfn "%s(%d) = %d" label x result
    result

applyAndShow (fun x -> x * x) 7 "square" |> ignore
applyAndShow (fun x -> x + 10) 5 "addTen" |> ignore

// Implement our own map
let myMap (f: 'a -> 'b) (lst: 'a list) : 'b list =
    let rec loop acc remaining =
        match remaining with
        | [] -> List.rev acc
        | head :: tail -> loop (f head :: acc) tail
    loop [] lst

let squared = myMap (fun x -> x * x) [1; 2; 3; 4; 5]
printfn "myMap squared: %A" squared

// Implement our own filter
let myFilter (predicate: 'a -> bool) (lst: 'a list) : 'a list =
    let rec loop acc remaining =
        match remaining with
        | [] -> List.rev acc
        | head :: tail ->
            if predicate head then loop (head :: acc) tail
            else loop acc tail
    loop [] lst

let evens = myFilter (fun x -> x % 2 = 0) [1..10]
printfn "myFilter evens: %A" evens
```

### 4.6.3 Returning Functions

```fsharp
// Functions สามารถ return functions

// Adder factory
let makeAdder n = fun x -> x + n

let add5 = makeAdder 5
let add100 = makeAdder 100

printfn "add5 3 = %d" (add5 3)
printfn "add100 42 = %d" (add100 42)

// Multiplier factory
let makeMultiplier n = fun x -> x * n
let double = makeMultiplier 2
let triple = makeMultiplier 3

// Memoization
let memoize f =
    let cache = System.Collections.Generic.Dictionary<_, _>()
    fun x ->
        if cache.ContainsKey(x) then
            cache.[x]
        else
            let result = f x
            cache.[x] <- result
            result

let expensiveCalc x =
    printfn "Computing for %d..." x
    System.Threading.Thread.Sleep(100)
    x * x

let memoizedCalc = memoize expensiveCalc

printfn "First call:"
let r1 = memoizedCalc 5
printfn "Second call (cached):"
let r2 = memoizedCalc 5  // ไม่ print "Computing for..."
printfn "Results: %d, %d" r1 r2
```

---

## 4.7 Function Composition

### 4.7.1 Pipe Operator (|>)

```fsharp
// |> ส่งค่าทางซ้ายเป็น argument สุดท้ายของ function ทางขวา
// x |> f = f x

let result = 5 |> (fun x -> x * 2)   // 10
printfn "%d" result

// Pipeline ยาว
let processData =
    [1..20]
    |> List.filter (fun x -> x % 2 = 0)   // even numbers
    |> List.map (fun x -> x * x)            // square
    |> List.filter (fun x -> x > 50)        // > 50
    |> List.sum

printfn "Process result: %d" processData

// ทำให้โค้ดอ่านง่ายจาก left to right
let words = "the quick brown fox jumps over the lazy dog"
let result2 =
    words
    |> fun s -> s.Split(' ')
    |> Array.toList
    |> List.distinct
    |> List.sort
    |> List.length

printfn "Unique words: %d" result2
```

### 4.7.2 Composition Operator (>>)

```fsharp
// >> compose functions: (f >> g) x = g (f x)
// f คำนวณก่อน แล้วส่งผลให้ g

let addOne = fun x -> x + 1
let double = fun x -> x * 2
let square = fun x -> x * x

// Compose
let addOneThenDouble = addOne >> double     // double(addOne x)
let doubleThenSquare = double >> square    // square(double x)
let addOneThenDoubleSquare = addOne >> double >> square

printfn "addOneThenDouble 5 = %d" (addOneThenDouble 5)   // (5+1)*2 = 12
printfn "doubleThenSquare 3 = %d" (doubleThenSquare 3)   // (3*2)^2 = 36

// << compose backward: (f << g) x = f (g x)
let squareThenDouble = double << square    // double(square x)
printfn "squareThenDouble 3 = %d" (squareThenDouble 3)   // (3^2)*2 = 18

// Real-world example: string processing pipeline
let trim (s: string) = s.Trim()
let toLower (s: string) = s.ToLower()
let removeSpaces (s: string) = s.Replace(" ", "")

let normalize = trim >> toLower >> removeSpaces

printfn "Normalized: '%s'" (normalize "  Hello World  ")

// Apply to list
let inputs = ["  Alice  "; " BOB "; "  Charlie   "]
let normalized = inputs |> List.map normalize
printfn "All normalized: %A" normalized
```

---

## 4.8 Generic Functions

```fsharp
// Generic functions ทำงานกับ types หลากหลาย

// Identity function
let identity<'T> (x: 'T) : 'T = x

printfn "identity int: %d" (identity 42)
printfn "identity string: %s" (identity "hello")
printfn "identity bool: %b" (identity true)

// Swap tuple elements
let swap<'a, 'b> (x: 'a, y: 'b) : 'b * 'a = (y, x)

printfn "swap (1, \"a\"): %A" (swap (1, "a"))
printfn "swap (true, 3.14): %A" (swap (true, 3.14))

// Generic pair
let makePair<'a, 'b> (x: 'a) (y: 'b) : 'a * 'b = (x, y)

// Generic list operations
let safeHead<'T> (lst: 'T list) : 'T option =
    match lst with
    | [] -> None
    | head :: _ -> Some head

let safeLast<'T> (lst: 'T list) : 'T option =
    match lst with
    | [] -> None
    | _ -> Some (List.last lst)

printfn "safeHead [1;2;3]: %A" (safeHead [1; 2; 3])
printfn "safeHead []: %A" (safeHead<int> [])
printfn "safeLast [1;2;3]: %A" (safeLast [1; 2; 3])

// Generic apply twice
let applyTwice (f: 'a -> 'a) (x: 'a) : 'a = f (f x)

printfn "applyTwice double 3 = %d" (applyTwice double 3)
printfn "applyTwice 'abc' reverse = %s" (applyTwice (fun (s: string) -> new string(s |> Seq.rev |> Seq.toArray)) "abcd")
```

---

## 4.9 Inline Functions

```fsharp
// inline ให้ compiler copy function body ที่ call site
// ช่วย performance สำหรับ small, frequently called functions

let inline square x = x * x
let inline add x y = x + y
let inline isEven x = x % 2 = 0

// inline กับ generic constraints
let inline sum (a: ^T) (b: ^T) = a + b  // ^ สำหรับ statically resolved type parameters

printfn "sum int: %d" (sum 3 4)
printfn "sum float: %f" (sum 3.0 4.0)
printfn "sum string (concat): %s" (sum "hello" " world")

// inline ที่ใช้บ่อย: numeric operations
let inline clamp minVal maxVal x =
    if x < minVal then minVal
    elif x > maxVal then maxVal
    else x

printfn "clamp 0 10 5 = %d" (clamp 0 10 5)
printfn "clamp 0 10 -3 = %d" (clamp 0 10 -3)
printfn "clamp 0 10 15 = %d" (clamp 0 10 15)

// float version (same code works!)
printfn "clamp 0.0 1.0 0.5 = %f" (clamp 0.0 1.0 0.5)
printfn "clamp 0.0 1.0 -0.1 = %f" (clamp 0.0 1.0 -0.1)
```

---

## 4.10 Optional Parameters และ Default Values

```fsharp
// F# ไม่มี default parameters แบบ Python/C# โดยตรง
// แต่ใช้ Option type แทนได้

// Pattern 1: Option type
let greet (name: string) (title: string option) =
    match title with
    | Some t -> $"สวัสดีคุณ {t} {name}"
    | None -> $"สวัสดี {name}"

printfn "%s" (greet "Smith" (Some "Dr."))
printfn "%s" (greet "Alice" None)

// Pattern 2: Optional parameter ด้วย ?
let greetOpt (name: string) (?title: string) =
    match title with
    | Some t -> $"สวัสดีคุณ {t} {name}"
    | None -> $"สวัสดี {name}"

// เรียกใช้
printfn "%s" (greetOpt "Smith" "Mr.")   // Some "Mr." ถูก pass
printfn "%s" (greetOpt "Alice")          // None

// Pattern 3: defaultArg
let createMessage (subject: string) (body: string) (?signature: string) =
    let sig = defaultArg signature "Best regards"
    sprintf "Subject: %s\n\n%s\n\n%s" subject body sig

let msg1 = createMessage "Hello" "How are you?"
let msg2 = createMessage "Hello" "How are you?" "Cheers"
printfn "%s\n" msg1
printfn "%s\n" msg2
```

---

## 4.11 Function Patterns

### 4.11.1 Point-free Style

```fsharp
// Point-free: ไม่ระบุ argument ชัดเจน แต่ compose functions

// Normal style
let normalDouble lst = lst |> List.map (fun x -> x * 2)

// Point-free style
let double = (*) 2  // partial application
let pointfreeDouble = List.map double

let numbers = [1; 2; 3; 4; 5]
printfn "Normal: %A" (normalDouble numbers)
printfn "Point-free: %A" (pointfreeDouble numbers)

// More examples
let isPositive = (<) 0    // 0 < x = x > 0 -- Hmm, ระวัง order!
// Actually: (<) 0 x = 0 < x = true when x > 0
let isPositiveFixed = fun x -> x > 0

// sum of list
let sumList = List.fold (+) 0

printfn "Sum: %d" (sumList [1..10])

// String operations point-free
let wordsOf (s: string) = s.Split(' ') |> Array.toList
let countWords = wordsOf >> List.length

printfn "Word count: %d" (countWords "hello world foo bar")
```

### 4.11.2 Memoization

```fsharp
open System.Collections.Generic

// Generic memoize function
let memoize (f: 'a -> 'b) =
    let cache = Dictionary<'a, 'b>()
    fun x ->
        match cache.TryGetValue(x) with
        | true, result -> result
        | false, _ ->
            let result = f x
            cache.[x] <- result
            result

// Fibonacci ที่ช้ามาก (exponential)
let rec fibSlow n =
    if n <= 1 then n
    else fibSlow (n - 1) + fibSlow (n - 2)

// Fibonacci ที่เร็ว (memoized)
let rec fibMemo =
    memoize (fun n ->
        if n <= 1 then n
        else fibMemo (n - 1) + fibMemo (n - 2))

// เปรียบเทียบความเร็ว
let sw = System.Diagnostics.Stopwatch.StartNew()
let r1 = fibSlow 35
sw.Stop()
printfn "fibSlow(35) = %d, Time: %dms" r1 sw.ElapsedMilliseconds

sw.Restart()
let r2 = fibMemo 35
sw.Stop()
printfn "fibMemo(35) = %d, Time: %dms" r2 sw.ElapsedMilliseconds
```

### 4.11.3 Function Pipelines

```fsharp
// สร้าง data processing pipelines

type Employee = {
    Name: string
    Department: string
    Salary: float
    YearsOfService: int
}

let employees = [
    { Name = "Alice"; Department = "Engineering"; Salary = 80000.0; YearsOfService = 5 }
    { Name = "Bob"; Department = "Marketing"; Salary = 60000.0; YearsOfService = 3 }
    { Name = "Charlie"; Department = "Engineering"; Salary = 95000.0; YearsOfService = 8 }
    { Name = "Diana"; Department = "HR"; Salary = 55000.0; YearsOfService = 2 }
    { Name = "Eve"; Department = "Engineering"; Salary = 75000.0; YearsOfService = 4 }
    { Name = "Frank"; Department = "Marketing"; Salary = 70000.0; YearsOfService = 6 }
]

// Pipeline operations
let engineeringDeptAvgSalary =
    employees
    |> List.filter (fun e -> e.Department = "Engineering")
    |> List.map (fun e -> e.Salary)
    |> List.average

printfn "Engineering avg salary: %.2f" engineeringDeptAvgSalary

// Top earners
let topEarners =
    employees
    |> List.sortByDescending (fun e -> e.Salary)
    |> List.take 3
    |> List.map (fun e -> e.Name)

printfn "Top 3 earners: %A" topEarners

// Salary raise for senior employees
let afterRaise =
    employees
    |> List.map (fun e ->
        if e.YearsOfService >= 5 then
            { e with Salary = e.Salary * 1.10 }
        else e)

printfn "\nSalaries after 10%% raise for 5+ years:"
afterRaise |> List.iter (fun e -> printfn "  %s: %.0f" e.Name e.Salary)
```

---

## 4.12 Advanced Function Patterns

### 4.12.1 Continuation Passing Style (CPS)

```fsharp
// CPS - ส่ง "what to do next" เป็น function argument

// Normal style
let addNormal x y = x + y

// CPS style
let addCPS x y (cont: int -> 'r) = cont (x + y)
let multiplyyCPS x y (cont: int -> 'r) = cont (x * y)

// ใช้งาน CPS
addCPS 3 4 (fun sum ->
    multiplyyCPS sum 2 (fun product ->
        printfn "Sum: %d, Product: %d" sum product))

// CPS สำหรับ early termination
let safeDivideCPS (a: float) (b: float) onSuccess onError =
    if b = 0.0 then
        onError "Division by zero!"
    else
        onSuccess (a / b)

safeDivideCPS 10.0 2.0
    (fun result -> printfn "Result: %f" result)
    (fun err -> printfn "Error: %s" err)

safeDivideCPS 10.0 0.0
    (fun result -> printfn "Result: %f" result)
    (fun err -> printfn "Error: %s" err)
```

### 4.12.2 Strategy Pattern

```fsharp
// Strategy pattern ด้วย function parameters

type SortStrategy<'a> = 'a list -> 'a list

let bubbleSort (lst: int list) =
    // simple bubble sort
    let arr = List.toArray lst
    let n = arr.Length
    for i in 0..n-2 do
        for j in 0..n-i-2 do
            if arr.[j] > arr.[j+1] then
                let temp = arr.[j]
                arr.[j] <- arr.[j+1]
                arr.[j+1] <- temp
    Array.toList arr

let mergeSort (lst: int list) = List.sort lst  // ใช้ built-in แทน

let processWithSort (strategy: SortStrategy<int>) (data: int list) =
    printfn "Input: %A" data
    let sorted = strategy data
    printfn "Sorted: %A" sorted
    sorted

let data = [5; 3; 8; 1; 9; 2; 7; 4; 6]

printfn "Using bubbleSort:"
processWithSort bubbleSort data |> ignore

printfn "\nUsing mergeSort:"
processWithSort mergeSort data |> ignore
```

### 4.12.3 Decorator Pattern

```fsharp
// Decorator pattern ด้วย higher-order functions

// Logger decorator
let withLogging (f: 'a -> 'b) (name: string) =
    fun x ->
        printfn "Calling %s with %A" name x
        let result = f x
        printfn "%s returned %A" name result
        result

// Timer decorator
let withTiming (f: 'a -> 'b) (name: string) =
    fun x ->
        let sw = System.Diagnostics.Stopwatch.StartNew()
        let result = f x
        sw.Stop()
        printfn "%s took %dms" name sw.ElapsedMilliseconds
        result

// Retry decorator
let withRetry (maxRetries: int) (f: 'a -> 'b option) =
    fun x ->
        let rec loop attempts =
            if attempts > maxRetries then
                printfn "All %d attempts failed" maxRetries
                None
            else
                match f x with
                | Some result -> Some result
                | None ->
                    printfn "Attempt %d failed, retrying..." attempts
                    loop (attempts + 1)
        loop 1

// Compose decorators
let myFunction x = x * x
let loggedFunction = withLogging myFunction "square"
let timedFunction = withTiming myFunction "square"

loggedFunction 5 |> ignore
timedFunction 1000 |> ignore

// Retry example
let mutable callCount = 0
let unreliableOperation x =
    callCount <- callCount + 1
    if callCount < 3 then
        printfn "  Simulated failure..."
        None
    else
        Some (x * 2)

let reliableOperation = withRetry 5 unreliableOperation
match reliableOperation 21 with
| Some result -> printfn "Final result: %d" result
| None -> printfn "Gave up!"
```

---

## 4.13 ตัวอย่างโปรแกรมสมบูรณ์

### 4.13.1 Functional Calculator

```fsharp
// functional_calculator.fsx

type BinaryOp = float -> float -> float
type UnaryOp = float -> float

let add: BinaryOp = (+)
let subtract: BinaryOp = (-)
let multiply: BinaryOp = (*)
let divide: BinaryOp = (/)

let absOp: UnaryOp = abs
let negOp: UnaryOp = fun x -> -x
let squareOp: UnaryOp = fun x -> x * x
let sqrtOp: UnaryOp = sqrt

let safeDivide (a: float) (b: float) : float option =
    if b = 0.0 then None
    else Some (a / b)

// Calculator state
type CalcState = {
    CurrentValue: float
    LastOp: string
    History: (string * float) list
}

let initialState = { CurrentValue = 0.0; LastOp = ""; History = [] }

let applyBinaryOp (op: BinaryOp) (opName: string) (state: CalcState) (value: float) =
    let result = op state.CurrentValue value
    { state with
        CurrentValue = result
        LastOp = opName
        History = (sprintf "%s %s %s = %s" (string state.CurrentValue) opName (string value) (string result), result) :: state.History }

let applyUnaryOp (op: UnaryOp) (opName: string) (state: CalcState) =
    let result = op state.CurrentValue
    { state with
        CurrentValue = result
        LastOp = opName
        History = (sprintf "%s(%s) = %s" opName (string state.CurrentValue) (string result), result) :: state.History }

// Demo
let calc =
    { initialState with CurrentValue = 10.0 }
    |> applyBinaryOp add "+" { initialState with CurrentValue = 10.0 } 5.0
    
let state = { initialState with CurrentValue = 10.0 }
let state2 = state |> applyBinaryOp add "+" state 5.0
let state3 = state2 |> applyBinaryOp multiply "*" state2 3.0
let state4 = state3 |> applyUnaryOp sqrt "sqrt"

printfn "Final value: %f" state4.CurrentValue
printfn "History:"
state4.History 
|> List.rev 
|> List.iter (fun (desc, _) -> printfn "  %s" desc)
```

---

## สรุป Part 4

ในบทนี้เราได้เรียนรู้:
- ✅ การนิยาม functions ด้วย let
- ✅ Function parameters และ return values
- ✅ Recursive functions ด้วย rec keyword
- ✅ Tail recursion
- ✅ Mutually recursive functions ด้วย and
- ✅ Nested functions
- ✅ Lambda expressions (fun x -> ...)
- ✅ Currying และ partial application
- ✅ First-class functions
- ✅ Function composition (>> และ |>)
- ✅ Generic functions
- ✅ Inline functions
- ✅ Optional parameters
- ✅ Memoization
- ✅ Higher-order patterns: CPS, Strategy, Decorator

**ใน Part 5** เราจะเรียนรู้ Pattern Matching ซึ่งเป็นหนึ่งใน features ที่ทรงพลังที่สุดของ F#!
