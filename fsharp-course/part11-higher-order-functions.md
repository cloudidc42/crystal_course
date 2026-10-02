# Part 11 - ฟังก์ชันอันดับสูง (Higher-Order Functions)

## บทนำ (Introduction)

ฟังก์ชันอันดับสูง (Higher-Order Functions หรือ HOF) คือฟังก์ชันที่:
1. รับฟังก์ชันอื่นเป็น argument (parameter)
2. ส่งคืนฟังก์ชันเป็นผลลัพธ์
3. หรือทั้งสองอย่าง

ใน F# ฟังก์ชันเป็น first-class values หมายความว่าสามารถเก็บในตัวแปร ส่งผ่านเป็น argument และส่งคืนจากฟังก์ชันได้เหมือนกับค่าปกติทุกชนิด

---

## 1. Functions as First-Class Values (ฟังก์ชันในฐานะค่าชั้นหนึ่ง)

```fsharp
// ฟังก์ชันธรรมดา
let add x y = x + y

// เก็บฟังก์ชันในตัวแปร
let myAdd = add

// เรียกใช้ผ่านตัวแปร
let result = myAdd 3 4   // 7

// ฟังก์ชันในรายการ
let operations = [add; fun x y -> x - y; fun x y -> x * y]

// เรียกใช้ฟังก์ชันจากรายการ
let firstOp = List.head operations
let r = firstOp 10 5   // 15

printfn "result = %d" result
printfn "firstOp 10 5 = %d" r
```

```fsharp
// ฟังก์ชันในทูเพิล
let funcPair = (fun x -> x + 1, fun x -> x * 2)

let incrementFunc, doubleFunc = funcPair

printfn "increment 5 = %d" (incrementFunc 5)  // 6
printfn "double 5 = %d" (doubleFunc 5)          // 10
```

```fsharp
// ฟังก์ชันใน record
type MathOps = {
    Add: int -> int -> int
    Multiply: int -> int -> int
    Negate: int -> int
}

let ops = {
    Add = fun x y -> x + y
    Multiply = fun x y -> x * y
    Negate = fun x -> -x
}

printfn "ops.Add 3 4 = %d" (ops.Add 3 4)         // 7
printfn "ops.Multiply 3 4 = %d" (ops.Multiply 3 4) // 12
printfn "ops.Negate 5 = %d" (ops.Negate 5)         // -5
```

---

## 2. Passing Functions as Parameters (การส่งฟังก์ชันเป็น argument)

```fsharp
// ฟังก์ชันที่รับฟังก์ชันเป็น argument
let applyToFive f = f 5

// ส่ง lambda
let r1 = applyToFive (fun x -> x * 2)   // 10
let r2 = applyToFive (fun x -> x + 100) // 105

// ส่งฟังก์ชันที่มีชื่อ
let square x = x * x
let r3 = applyToFive square  // 25

printfn "%d %d %d" r1 r2 r3
```

```fsharp
// ฟังก์ชัน applyTwice: ใช้ฟังก์ชัน f กับค่า x สองครั้ง
let applyTwice f x = f (f x)

let increment x = x + 1
let double x = x * 2

let r1 = applyTwice increment 5  // (5+1)+1 = 7
let r2 = applyTwice double 3     // (3*2)*2 = 12
let r3 = applyTwice square 2     // (2*2)^2 = 16 (where square = x*x)

printfn "applyTwice increment 5 = %d" r1
printfn "applyTwice double 3 = %d" r2
printfn "applyTwice square 2 = %d" r3
```

```fsharp
// ฟังก์ชันที่รับ predicate (เงื่อนไข)
let filterList predicate lst =
    List.filter predicate lst

let numbers = [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]

let evens = filterList (fun x -> x % 2 = 0) numbers
let odds = filterList (fun x -> x % 2 <> 0) numbers
let bigOnes = filterList (fun x -> x > 5) numbers

printfn "evens: %A" evens
printfn "odds: %A" odds
printfn "bigOnes: %A" bigOnes
```

---

## 3. Returning Functions from Functions (การส่งคืนฟังก์ชัน)

```fsharp
// ฟังก์ชันที่ส่งคืนฟังก์ชัน
let makeAdder n = fun x -> x + n

let add5 = makeAdder 5
let add10 = makeAdder 10

printfn "add5 3 = %d" (add5 3)   // 8
printfn "add10 3 = %d" (add10 3) // 13
```

```fsharp
// ฟังก์ชัน makeMultiplier
let makeMultiplier factor = fun x -> x * factor

let triple = makeMultiplier 3
let times10 = makeMultiplier 10

printfn "triple 7 = %d" (triple 7)   // 21
printfn "times10 4 = %d" (times10 4) // 40
```

```fsharp
// ฟังก์ชัน makeGreeter
let makeGreeter greeting =
    fun name -> sprintf "%s, %s!" greeting name

let hello = makeGreeter "Hello"
let hi = makeGreeter "Hi"
let sawasdee = makeGreeter "สวัสดี"

printfn "%s" (hello "Alice")    // Hello, Alice!
printfn "%s" (hi "Bob")         // Hi, Bob!
printfn "%s" (sawasdee "สมชาย") // สวัสดี, สมชาย!
```

```fsharp
// ฟังก์ชัน makeCounter
let makeCounter () =
    let mutable count = 0
    fun () ->
        count <- count + 1
        count

let counter = makeCounter ()
printfn "%d" (counter ())  // 1
printfn "%d" (counter ())  // 2
printfn "%d" (counter ())  // 3

let counter2 = makeCounter ()
printfn "%d" (counter2 ()) // 1 (counter อิสระ)
```

---

## 4. Common HOF: map, filter, fold, reduce

### 4.1 List.map

```fsharp
// List.map: แปลงทุกองค์ประกอบในรายการ
// signature: ('a -> 'b) -> 'a list -> 'b list

let numbers = [1; 2; 3; 4; 5]

// แปลงเป็นกำลังสอง
let squares = List.map (fun x -> x * x) numbers
printfn "squares: %A" squares  // [1; 4; 9; 16; 25]

// แปลงเป็น string
let strs = List.map string numbers
printfn "strs: %A" strs  // ["1"; "2"; "3"; "4"; "5"]

// แปลงเป็น double
let doubles = List.map float numbers
printfn "doubles: %A" doubles  // [1.0; 2.0; 3.0; 4.0; 5.0]
```

```fsharp
// map กับชนิดข้อมูลที่ซับซ้อน
type Person = { Name: string; Age: int }

let people = [
    { Name = "Alice"; Age = 30 }
    { Name = "Bob"; Age = 25 }
    { Name = "Charlie"; Age = 35 }
]

// ดึงเฉพาะชื่อ
let names = List.map (fun p -> p.Name) people
printfn "names: %A" names

// เพิ่มอายุ 1 ปี
let olderPeople = List.map (fun p -> { p with Age = p.Age + 1 }) people
printfn "olderPeople: %A" olderPeople

// แปลงเป็น string
let descriptions = List.map (fun p -> sprintf "%s is %d years old" p.Name p.Age) people
printfn "descriptions: %A" descriptions
```

### 4.2 List.filter

```fsharp
// List.filter: กรององค์ประกอบที่ตรงเงื่อนไข
// signature: ('a -> bool) -> 'a list -> 'a list

let numbers = [1..20]

let evens = List.filter (fun x -> x % 2 = 0) numbers
let odds = List.filter (fun x -> x % 2 <> 0) numbers
let divisibleBy3 = List.filter (fun x -> x % 3 = 0) numbers
let greaterThan10 = List.filter (fun x -> x > 10) numbers

printfn "evens: %A" evens
printfn "odds: %A" odds
printfn "divisibleBy3: %A" divisibleBy3
printfn "greaterThan10: %A" greaterThan10
```

```fsharp
// filter กับ string
let words = ["apple"; "banana"; "cherry"; "date"; "elderberry"; "fig"]

let longWords = List.filter (fun w -> w.Length > 5) words
let startsWithA = List.filter (fun w -> w.StartsWith("a")) words
let hasE = List.filter (fun w -> w.Contains("e")) words

printfn "longWords: %A" longWords
printfn "startsWithA: %A" startsWithA
printfn "hasE: %A" hasE
```

### 4.3 List.fold

```fsharp
// List.fold: รวมรายการให้เป็นค่าเดียว
// signature: ('state -> 'a -> 'state) -> 'state -> 'a list -> 'state

let numbers = [1; 2; 3; 4; 5]

// หาผลรวม
let sum = List.fold (fun acc x -> acc + x) 0 numbers
printfn "sum = %d" sum  // 15

// หาผลคูณ
let product = List.fold (fun acc x -> acc * x) 1 numbers
printfn "product = %d" product  // 120

// หาค่าสูงสุด
let maxVal = List.fold (fun acc x -> if x > acc then x else acc) System.Int32.MinValue numbers
printfn "maxVal = %d" maxVal  // 5
```

```fsharp
// fold เพื่อสร้าง list ใหม่
let numbers = [1..10]

// สร้าง list ของกำลังสอง (เหมือน map)
let squares = List.fold (fun acc x -> acc @ [x * x]) [] numbers
printfn "squares: %A" squares

// นับองค์ประกอบที่ตรงเงื่อนไข (เหมือน filter + length)
let countEvens = List.fold (fun acc x -> if x % 2 = 0 then acc + 1 else acc) 0 numbers
printfn "countEvens = %d" countEvens  // 5
```

```fsharp
// fold เพื่อสร้าง string
let words = ["Hello"; "World"; "From"; "F#"]

let sentence = List.fold (fun acc w -> if acc = "" then w else acc + " " + w) "" words
printfn "sentence: %s" sentence  // "Hello World From F#"

// ย้อนกลับรายการ (reverse)
let reversed = List.fold (fun acc x -> x :: acc) [] [1; 2; 3; 4; 5]
printfn "reversed: %A" reversed  // [5; 4; 3; 2; 1]
```

### 4.4 List.foldBack

```fsharp
// List.foldBack: เหมือน fold แต่วนจากขวาไปซ้าย
// signature: ('a -> 'state -> 'state) -> 'a list -> 'state -> 'state

let numbers = [1; 2; 3; 4; 5]

// foldBack เพื่อรักษาลำดับ
let copy = List.foldBack (fun x acc -> x :: acc) numbers []
printfn "copy: %A" copy  // [1; 2; 3; 4; 5]

// เปรียบเทียบ fold vs foldBack
let foldResult = List.fold (fun acc x -> acc + sprintf "%d," x) "" numbers
let foldBackResult = List.foldBack (fun x acc -> sprintf "%d," x + acc) numbers ""
printfn "fold: %s" foldResult      // "1,2,3,4,5,"
printfn "foldBack: %s" foldBackResult // "1,2,3,4,5,"
```

### 4.5 List.reduce

```fsharp
// List.reduce: เหมือน fold แต่ใช้องค์ประกอบแรกเป็น initial value
// ต้องมีรายการที่ไม่ว่าง!
// signature: ('a -> 'a -> 'a) -> 'a list -> 'a

let numbers = [1; 2; 3; 4; 5]

let sum = List.reduce (+) numbers          // 15
let product = List.reduce (*) numbers      // 120
let maxVal = List.reduce max numbers       // 5
let minVal = List.reduce min numbers       // 1

printfn "sum = %d" sum
printfn "product = %d" product
printfn "maxVal = %d" maxVal
printfn "minVal = %d" minVal

// ระวัง: reduce กับ list ว่าง จะ throw exception
// List.reduce (+) []  // Exception!

// ทางออก: ใช้ fold
let safeSum = List.fold (+) 0  // ปลอดภัย แม้กับ list ว่าง
```

---

## 5. Custom HOF Implementation (การสร้าง HOF เอง)

```fsharp
// สร้าง HOF ของตัวเอง

// 1. myMap: เหมือน List.map
let rec myMap f lst =
    match lst with
    | [] -> []
    | head :: tail -> f head :: myMap f tail

let result = myMap (fun x -> x * 2) [1; 2; 3; 4; 5]
printfn "myMap: %A" result  // [2; 4; 6; 8; 10]
```

```fsharp
// 2. myFilter: เหมือน List.filter
let rec myFilter predicate lst =
    match lst with
    | [] -> []
    | head :: tail ->
        if predicate head
        then head :: myFilter predicate tail
        else myFilter predicate tail

let evens = myFilter (fun x -> x % 2 = 0) [1..10]
printfn "myFilter evens: %A" evens
```

```fsharp
// 3. myFold: เหมือน List.fold
let rec myFold f acc lst =
    match lst with
    | [] -> acc
    | head :: tail -> myFold f (f acc head) tail

let sum = myFold (+) 0 [1..10]
printfn "myFold sum: %d" sum  // 55
```

```fsharp
// 4. myZipWith: รวม 2 รายการด้วยฟังก์ชัน
let rec myZipWith f lst1 lst2 =
    match lst1, lst2 with
    | [], _ | _, [] -> []
    | h1 :: t1, h2 :: t2 -> f h1 h2 :: myZipWith f t1 t2

let sums = myZipWith (+) [1; 2; 3] [10; 20; 30]
printfn "myZipWith (+): %A" sums  // [11; 22; 33]

let products = myZipWith (*) [1; 2; 3] [10; 20; 30]
printfn "myZipWith (*): %A" products  // [10; 40; 90]
```

```fsharp
// 5. myForEach: ทำ side effect กับทุกองค์ประกอบ
let rec myForEach f lst =
    match lst with
    | [] -> ()
    | head :: tail ->
        f head
        myForEach f tail

myForEach (printfn "Item: %d") [1; 2; 3; 4; 5]
```

```fsharp
// 6. myAll: ตรวจสอบว่าทุกองค์ประกอบตรงเงื่อนไข
let rec myAll predicate lst =
    match lst with
    | [] -> true
    | head :: tail -> predicate head && myAll predicate tail

// 7. myAny: ตรวจสอบว่ามีองค์ประกอบใดตรงเงื่อนไข
let rec myAny predicate lst =
    match lst with
    | [] -> false
    | head :: tail -> predicate head || myAny predicate tail

let allPositive = myAll (fun x -> x > 0) [1; 2; 3; 4; 5]
let anyNegative = myAny (fun x -> x < 0) [1; -2; 3; 4; 5]

printfn "allPositive: %b" allPositive  // true
printfn "anyNegative: %b" anyNegative  // true
```

---

## 6. Function Composition with >> and <<

```fsharp
// >> คือ forward composition: f >> g หมายถึง g(f(x))
// << คือ backward composition: f << g หมายถึง f(g(x))

let add1 x = x + 1
let double x = x * 2
let square x = x * x

// forward composition
let add1ThenDouble = add1 >> double  // x -> double(add1(x))
let r1 = add1ThenDouble 5  // double(add1(5)) = double(6) = 12

// backward composition
let doubleAfterAdd1 = double << add1  // เหมือนกัน
let r2 = doubleAfterAdd1 5  // 12

printfn "add1ThenDouble 5 = %d" r1
printfn "doubleAfterAdd1 5 = %d" r2
```

```fsharp
// ซ้อนหลาย composition
let pipeline = add1 >> double >> square >> string

let result = pipeline 3
// 3 -> add1 -> 4 -> double -> 8 -> square -> 64 -> string -> "64"
printfn "pipeline 3 = %s" result  // "64"
```

```fsharp
// composition กับ string functions
let trim (s: string) = s.Trim()
let toLower (s: string) = s.ToLower()
let removeSpaces (s: string) = s.Replace(" ", "_")

let normalize = trim >> toLower >> removeSpaces

let result1 = normalize "  Hello World  "
let result2 = normalize " F# Programming "

printfn "%s" result1  // "hello_world"
printfn "%s" result2  // "f#_programming"
```

```fsharp
// สร้าง composed functions สำหรับ pipeline
let numbers = [1..10]

// ก่อนใช้ composition
let result1 = numbers
              |> List.filter (fun x -> x % 2 = 0)
              |> List.map (fun x -> x * x)
              |> List.sum

// ใช้ composition
let processEvenSquares =
    List.filter (fun x -> x % 2 = 0)
    >> List.map (fun x -> x * x)
    >> List.sum

let result2 = processEvenSquares numbers

printfn "result1 = %d" result1  // 220
printfn "result2 = %d" result2  // 220
```

---

## 7. flip Function

```fsharp
// flip: สลับ argument แรกและที่สองของฟังก์ชัน
let flip f x y = f y x

// ตัวอย่าง
let subtract x y = x - y
let subtractFrom = flip subtract

printfn "subtract 10 3 = %d" (subtract 10 3)      // 7
printfn "subtractFrom 3 10 = %d" (subtractFrom 3 10)  // 7
```

```fsharp
// ใช้ flip กับ List functions
let numbers = [1; 2; 3; 4; 5]

// List.map ต้องการ (f, list) แต่ในบาง context เราอยากส่ง list ก่อน
let mapFlipped = flip List.map

let squaredList = mapFlipped numbers (fun x -> x * x)
printfn "squaredList: %A" squaredList  // [1; 4; 9; 16; 25]
```

```fsharp
// flip ใน partial application
let divideBy divisor dividend = dividend / divisor

let divideBy2 = divideBy 2
let halve = flip divideBy 2  // เหมือนกัน

printfn "divideBy2 10 = %d" (divideBy2 10)  // 5
printfn "halve 10 = %d" (halve 10)           // 5

let numbers = [10; 20; 30; 40; 50]
let halved = List.map halve numbers
printfn "halved: %A" halved  // [5; 10; 15; 20; 25]
```

---

## 8. apply Function (|>)

```fsharp
// |> คือ pipe operator: ส่งค่าไปเป็น argument สุดท้ายของฟังก์ชัน
// x |> f  เหมือนกับ f x

let result = 5 |> (fun x -> x * 2)  // 10

// Pipeline
let numbers = [1..10]
let result2 =
    numbers
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.sum

printfn "result2 = %d" result2  // 220
```

```fsharp
// สร้าง custom apply
let apply f x = f x

// ใช้กับรายการฟังก์ชัน
let fns = [(+) 1; (*) 2; (fun x -> x - 3)]

let results = List.map (apply 10) fns
printfn "results: %A" results  // [11; 20; 7]
```

---

## 9. twice, thrice Pattern

```fsharp
// twice: ใช้ฟังก์ชันสองครั้ง
let twice f = f >> f

// thrice: ใช้ฟังก์ชันสามครั้ง
let thrice f = f >> f >> f

// ntimes: ใช้ฟังก์ชัน n ครั้ง
let rec ntimes n f =
    if n <= 0 then id
    elif n = 1 then f
    else f >> ntimes (n - 1) f

let add1 x = x + 1
let double x = x * 2

printfn "twice add1 5 = %d" (twice add1 5)    // 7
printfn "thrice add1 5 = %d" (thrice add1 5)  // 8
printfn "ntimes 5 add1 5 = %d" (ntimes 5 add1 5)  // 10
printfn "twice double 3 = %d" (twice double 3)    // 12
```

```fsharp
// iterate: ใช้ฟังก์ชัน n ครั้งและเก็บผลลัพธ์ทั้งหมด
let iterate n f x =
    let rec go remaining current acc =
        if remaining = 0 then List.rev acc
        else
            let next = f current
            go (remaining - 1) next (next :: acc)
    x :: go n x []

let results = iterate 5 add1 0
printfn "iterate 5 add1 0: %A" results  // [0; 1; 2; 3; 4; 5]

let doubles = iterate 5 double 1
printfn "iterate 5 double 1: %A" doubles  // [1; 2; 4; 8; 16; 32]
```

---

## 10. Memoization

```fsharp
// Memoization: เก็บผลลัพธ์ที่คำนวณแล้ว เพื่อไม่ต้องคำนวณซ้ำ

let memoize f =
    let cache = System.Collections.Generic.Dictionary<_, _>()
    fun x ->
        match cache.TryGetValue(x) with
        | true, v -> v
        | false, _ ->
            let v = f x
            cache.[x] <- v
            v

// Fibonacci แบบช้า (exponential time)
let rec fib n =
    if n <= 1 then n
    else fib (n - 1) + fib (n - 2)

// Fibonacci แบบ memoized
let rec fibMemo =
    memoize (fun n ->
        if n <= 1 then n
        else fibMemo (n - 1) + fibMemo (n - 2))

// ทดสอบความเร็ว
let sw = System.Diagnostics.Stopwatch.StartNew()
let r1 = fib 35
sw.Stop()
printfn "fib 35 = %d (%.3f seconds)" r1 sw.Elapsed.TotalSeconds

sw.Restart()
let r2 = fibMemo 35
sw.Stop()
printfn "fibMemo 35 = %d (%.3f seconds)" r2 sw.Elapsed.TotalSeconds
```

```fsharp
// Memoization แบบ thread-safe
open System.Collections.Concurrent

let memoizeSafe f =
    let cache = ConcurrentDictionary<_, _>()
    fun x -> cache.GetOrAdd(x, f)

let expensiveCompute x =
    System.Threading.Thread.Sleep(100)  // จำลองการคำนวณที่ใช้เวลา
    x * x

let fastCompute = memoizeSafe expensiveCompute

let t = System.Diagnostics.Stopwatch.StartNew()
for _ in 1..5 do
    fastCompute 42 |> ignore  // เรียก 5 ครั้ง แต่คำนวณจริงแค่ครั้งแรก
t.Stop()
printfn "5 calls took: %.3f seconds (vs 0.5 if no memo)" t.Elapsed.TotalSeconds
```

---

## 11. Decorator Pattern using HOF

```fsharp
// Decorator pattern: เพิ่ม behavior ให้ฟังก์ชันโดยไม่แก้ไข code เดิม

// Logging decorator
let withLogging functionName f =
    fun args ->
        printfn "Calling %s with args: %A" functionName args
        let result = f args
        printfn "%s returned: %A" functionName result
        result

// Timing decorator
let withTiming f =
    fun args ->
        let sw = System.Diagnostics.Stopwatch.StartNew()
        let result = f args
        sw.Stop()
        printfn "Took %.6f seconds" sw.Elapsed.TotalSeconds
        result

// ฟังก์ชันเดิม
let compute x = x * x + 2 * x + 1

// ฟังก์ชันที่ถูก decorate
let loggedCompute = withLogging "compute" compute

let r = loggedCompute 5
printfn "Result: %d" r
```

```fsharp
// Retry decorator
let withRetry maxAttempts f =
    fun args ->
        let rec attempt n =
            try
                f args
            with ex ->
                if n >= maxAttempts then
                    reraise ()
                else
                    printfn "Attempt %d failed: %s. Retrying..." n ex.Message
                    attempt (n + 1)
        attempt 1

// ฟังก์ชันที่อาจล้มเหลว
let mutable callCount = 0
let unreliableFunction x =
    callCount <- callCount + 1
    if callCount < 3 then
        failwith "Transient error"
    else
        x * 2

let reliableFunction = withRetry 5 unreliableFunction

let result = reliableFunction 21
printfn "Result: %d" result  // 42
```

```fsharp
// Caching decorator
let withCache f =
    let cache = System.Collections.Generic.Dictionary<_, _>()
    fun args ->
        match cache.TryGetValue(args) with
        | true, result ->
            printfn "Cache hit for %A" args
            result
        | false, _ ->
            let result = f args
            cache.[args] <- result
            printfn "Cache miss for %A, computed: %A" args result
            result

let slowSquare x =
    System.Threading.Thread.Sleep(500)
    x * x

let fastSquare = withCache slowSquare

fastSquare 5 |> printfn "First call: %d"   // Cache miss
fastSquare 5 |> printfn "Second call: %d"  // Cache hit
fastSquare 3 |> printfn "Third call: %d"   // Cache miss
fastSquare 3 |> printfn "Fourth call: %d"  // Cache hit
```

---

## 12. Strategy Pattern using HOF

```fsharp
// Strategy pattern: เลือก algorithm ณ runtime โดยใช้ฟังก์ชัน

// Sorting strategies
type SortStrategy<'a> = 'a list -> 'a list

let bubbleSortStrategy : SortStrategy<int> =
    fun lst ->
        let arr = List.toArray lst
        let n = arr.Length
        for i in 0..n-2 do
            for j in 0..n-i-2 do
                if arr.[j] > arr.[j+1] then
                    let temp = arr.[j]
                    arr.[j] <- arr.[j+1]
                    arr.[j+1] <- temp
        Array.toList arr

let insertionSortStrategy : SortStrategy<int> =
    fun lst ->
        let rec insert x = function
            | [] -> [x]
            | h :: t -> if x <= h then x :: h :: t else h :: insert x t
        List.fold (fun acc x -> insert x acc) [] lst

let sortWithStrategy (strategy: SortStrategy<_>) lst =
    strategy lst

let numbers = [5; 2; 8; 1; 9; 3; 7; 4; 6]

let bubbleSorted = sortWithStrategy bubbleSortStrategy numbers
let insertionSorted = sortWithStrategy insertionSortStrategy numbers

printfn "Bubble: %A" bubbleSorted
printfn "Insertion: %A" insertionSorted
```

```fsharp
// Validation strategies
type Validator<'a> = 'a -> Result<'a, string>

let notEmpty : Validator<string> =
    fun s ->
        if System.String.IsNullOrWhiteSpace(s) then Error "Cannot be empty"
        else Ok s

let minLength n : Validator<string> =
    fun s ->
        if s.Length < n then Error (sprintf "Must be at least %d characters" n)
        else Ok s

let maxLength n : Validator<string> =
    fun s ->
        if s.Length > n then Error (sprintf "Must be at most %d characters" n)
        else Ok s

// Combine validators
let combineValidators (validators: Validator<'a> list) : Validator<'a> =
    fun value ->
        validators
        |> List.fold (fun acc v ->
            match acc with
            | Error e -> Error e
            | Ok x -> v x) (Ok value)

let validateUsername =
    combineValidators [
        notEmpty
        minLength 3
        maxLength 20
    ]

printfn "%A" (validateUsername "alice")    // Ok "alice"
printfn "%A" (validateUsername "")         // Error "Cannot be empty"
printfn "%A" (validateUsername "ab")       // Error "Must be at least 3 characters"
printfn "%A" (validateUsername (String.replicate 25 "a"))  // Error "Must be at most 20..."
```

---

## 13. Practical Examples with Real Use Cases

### 13.1 Data Processing Pipeline

```fsharp
// ตัวอย่างจริง: ประมวลผลข้อมูลนักศึกษา

type Student = {
    Name: string
    Grade: float
    Department: string
}

let students = [
    { Name = "Alice"; Grade = 85.5; Department = "CS" }
    { Name = "Bob"; Grade = 72.0; Department = "Math" }
    { Name = "Charlie"; Grade = 91.3; Department = "CS" }
    { Name = "Diana"; Grade = 68.5; Department = "Physics" }
    { Name = "Eve"; Grade = 95.0; Department = "CS" }
    { Name = "Frank"; Grade = 55.0; Department = "Math" }
    { Name = "Grace"; Grade = 88.0; Department = "Physics" }
]

// Higher-order functions สำหรับ analysis
let getTopStudents minGrade =
    List.filter (fun s -> s.Grade >= minGrade)

let getByDepartment dept =
    List.filter (fun s -> s.Department = dept)

let averageGrade students =
    if List.isEmpty students then 0.0
    else List.averageBy (fun s -> s.Grade) students

let getRanked =
    List.sortByDescending (fun s -> s.Grade)

// ใช้งาน
let csStudents = students |> getByDepartment "CS"
let topStudents = students |> getTopStudents 80.0
let csAvg = csStudents |> averageGrade
let ranked = students |> getRanked

printfn "CS Students: %A" (List.map (fun s -> s.Name) csStudents)
printfn "Top Students (>=80): %A" (List.map (fun s -> s.Name) topStudents)
printfn "CS Average Grade: %.2f" csAvg
printfn "Ranked: %A" (List.map (fun s -> (s.Name, s.Grade)) ranked)
```

### 13.2 Event Handling System

```fsharp
// ระบบ event handler ที่ใช้ HOF

type EventHandler<'a> = 'a -> unit

type EventEmitter<'a>() =
    let mutable handlers: EventHandler<'a> list = []
    
    member _.Subscribe(handler) =
        handlers <- handler :: handlers
    
    member _.Emit(event) =
        handlers |> List.iter (fun h -> h event)

// ใช้งาน
let emitter = EventEmitter<string>()

emitter.Subscribe(fun msg -> printfn "Logger: %s" msg)
emitter.Subscribe(fun msg -> printfn "Console: [%s]" msg)
emitter.Subscribe(fun msg ->
    if msg.Contains("ERROR") then
        printfn "Alert! Error detected: %s" msg)

emitter.Emit("User logged in")
emitter.Emit("ERROR: Database connection failed")
emitter.Emit("User logged out")
```

### 13.3 Functional Middleware

```fsharp
// Middleware pattern (เหมือน ASP.NET middleware แต่ functional style)

type HttpContext = {
    Path: string
    Method: string
    mutable Response: string
}

type Middleware = HttpContext -> (HttpContext -> unit) -> unit

let loggingMiddleware : Middleware =
    fun ctx next ->
        printfn "[LOG] %s %s" ctx.Method ctx.Path
        next ctx
        printfn "[LOG] Response: %s" ctx.Response

let authMiddleware : Middleware =
    fun ctx next ->
        if ctx.Path.StartsWith("/admin") then
            ctx.Response <- "401 Unauthorized"
        else
            next ctx

let handler : HttpContext -> unit =
    fun ctx ->
        ctx.Response <- sprintf "200 OK - %s" ctx.Path

// Compose middleware
let compose (middlewares: Middleware list) (finalHandler: HttpContext -> unit) =
    List.foldBack (fun mw next -> fun ctx -> mw ctx next) middlewares finalHandler

let pipeline = compose [loggingMiddleware; authMiddleware] handler

// Test
let ctx1 = { Path = "/home"; Method = "GET"; Response = "" }
pipeline ctx1
printfn "Response: %s" ctx1.Response

let ctx2 = { Path = "/admin/dashboard"; Method = "GET"; Response = "" }
pipeline ctx2
printfn "Response: %s" ctx2.Response
```

### 13.4 Pipeline Builder Pattern

```fsharp
// สร้าง data transformation pipeline

type Pipeline<'a> =
    | Pipeline of ('a -> 'a)

let createPipeline () = Pipeline id

let addStep (Pipeline current) step =
    Pipeline (current >> step)

let run (Pipeline pipeline) input =
    pipeline input

// ใช้งาน
let textPipeline =
    createPipeline ()
    |> addStep (fun (s: string) -> s.Trim())
    |> addStep (fun s -> s.ToLower())
    |> addStep (fun s -> s.Replace(" ", "-"))
    |> addStep (fun s -> sprintf "https://example.com/%s" s)

let url = run textPipeline "  Hello World  "
printfn "URL: %s" url  // "https://example.com/hello-world"
```

---

## 14. สรุป (Summary)

Higher-Order Functions เป็นหัวใจสำคัญของ functional programming ใน F#:

1. **Functions as first-class values**: ฟังก์ชันสามารถเก็บ ส่งผ่าน และส่งคืนได้เหมือนค่าปกติ
2. **map**: แปลงทุกองค์ประกอบในคอลเลกชัน
3. **filter**: กรององค์ประกอบตามเงื่อนไข
4. **fold**: รวมคอลเลกชันเป็นค่าเดียว
5. **reduce**: คล้าย fold แต่ใช้องค์ประกอบแรกเป็น initial value
6. **>>** และ **<<**: ประกอบฟังก์ชันเข้าด้วยกัน
7. **flip**: สลับ argument ของฟังก์ชัน
8. **Memoization**: cache ผลลัพธ์เพื่อเพิ่มประสิทธิภาพ
9. **Decorator pattern**: เพิ่ม behavior โดยไม่แก้โค้ดเดิม
10. **Strategy pattern**: เลือก algorithm ณ runtime

```fsharp
// ตัวอย่างรวม: ใช้ HOF ทั้งหมดร่วมกัน
let processData =
    List.filter (fun x -> x > 0)          // กรองค่าบวก
    >> List.map (fun x -> x * x)           // ยกกำลังสอง
    >> List.filter (fun x -> x < 100)      // กรองค่าน้อยกว่า 100
    >> List.fold (+) 0                      // รวมทั้งหมด

let data = [-3; 1; -2; 5; 8; -1; 3; 10; 2]
let result = processData data
printfn "Result: %d" result  // 1 + 25 + 9 + 4 = 39
```

Higher-Order Functions ทำให้โค้ดเป็น declarative (บอกว่า "ต้องการอะไร") มากกว่า imperative (บอกว่า "ทำอย่างไร") ทำให้อ่านง่าย ทดสอบง่าย และนำกลับมาใช้ใหม่ได้
