# Part 6 - รายการและคอลเลกชัน (Lists and Collections)

## บทนำ

Collections เป็น core feature ของการเขียนโปรแกรมแบบ functional ใน F# เราจะเรียนรู้ List ซึ่งเป็น immutable linked list ที่ใช้บ่อยที่สุด รวมถึง Sequence, Array, และ ResizeArray F# มี module Library ที่ครอบคลุมสำหรับ collection operations

---

## 6.1 F# List พื้นฐาน

### 6.1.1 การสร้าง List

```fsharp
// List literal ใช้ [ ] และคั่นด้วย ;
let empty = []
let numbers = [1; 2; 3; 4; 5]
let strings = ["hello"; "world"; "F#"]
let mixed = [1; 2; 3]  // type ต้องเหมือนกันทั้งหมด

// Computed list
let squares = [1; 4; 9; 16; 25]

printfn "empty: %A" empty
printfn "numbers: %A" numbers
printfn "strings: %A" strings

// Type of list
printfn "Type: %s" (numbers.GetType().Name)  // Microsoft.FSharp.Collections.FSharpList`1[System.Int32]
```

### 6.1.2 List Cons Operator (::)

```fsharp
// :: เพิ่ม element ที่ head ของ list (efficient O(1))
let original = [2; 3; 4; 5]
let withOne = 1 :: original    // [1; 2; 3; 4; 5]
let withZero = 0 :: withOne    // [0; 1; 2; 3; 4; 5]

printfn "original: %A" original
printfn "withOne: %A" withOne
printfn "withZero: %A" withZero

// Building list with ::
let buildList () =
    let mutable result = []
    for i in 5..-1..1 do
        result <- i :: result  // prepend
    result

printfn "built: %A" (buildList ())  // [1; 2; 3; 4; 5]

// Cons ใน pattern matching
let myHead = function
    | [] -> failwith "empty list"
    | head :: _ -> head

let myTail = function
    | [] -> failwith "empty list"
    | _ :: tail -> tail

printfn "head of [1;2;3]: %d" (myHead [1; 2; 3])
printfn "tail of [1;2;3]: %A" (myTail [1; 2; 3])
```

---

## 6.2 List Basic Operations

### 6.2.1 Length, Head, Tail, Last

```fsharp
let lst = [10; 20; 30; 40; 50]

// Length
printfn "Length: %d" (List.length lst)          // 5
printfn "Length: %d" lst.Length                  // 5

// Head (first element)
printfn "Head: %d" (List.head lst)               // 10

// Tail (all except first)
printfn "Tail: %A" (List.tail lst)               // [20; 30; 40; 50]

// Last (last element) - O(n)
printfn "Last: %d" (List.last lst)               // 50

// Item at index
printfn "Item 2: %d" (List.item 2 lst)           // 30
printfn "Item 2: %d" lst.[2]                      // 30 (same)

// Safe versions
printfn "tryHead: %A" (List.tryHead lst)          // Some 10
printfn "tryHead []: %A" (List.tryHead [])         // None
printfn "tryLast: %A" (List.tryLast lst)          // Some 50
printfn "tryItem 10: %A" (List.tryItem 10 lst)    // None
```

### 6.2.2 Append และ Concat

```fsharp
// Append ด้วย @ operator
let a = [1; 2; 3]
let b = [4; 5; 6]
let c = [7; 8; 9]

let appended = a @ b        // [1; 2; 3; 4; 5; 6]
let three = a @ b @ c       // [1..9]

printfn "appended: %A" appended
printfn "three: %A" three

// List.append (เหมือน @)
let appended2 = List.append a b
printfn "appended2: %A" appended2

// List.concat - combine list of lists
let listOfLists = [[1; 2]; [3; 4]; [5; 6]]
let flattened = List.concat listOfLists   // [1; 2; 3; 4; 5; 6]
printfn "concat: %A" flattened

// ระวัง: @ เป็น O(n) ไม่ใช่ O(1)
// สำหรับ performance ใช้ :: แล้ว List.rev ทีหลัง
```

---

## 6.3 List.map

```fsharp
// List.map: แปลงทุก element ด้วย function

let numbers = [1; 2; 3; 4; 5]

// Double all numbers
let doubled = List.map (fun x -> x * 2) numbers
printfn "doubled: %A" doubled  // [2; 4; 6; 8; 10]

// Square all numbers
let squared = numbers |> List.map (fun x -> x * x)
printfn "squared: %A" squared  // [1; 4; 9; 16; 25]

// Convert to strings
let strings = numbers |> List.map string
printfn "strings: %A" strings  // ["1"; "2"; "3"; "4"; "5"]

// Map with records
type Product = { Name: string; Price: float }

let products = [
    { Name = "Widget A"; Price = 10.0 }
    { Name = "Widget B"; Price = 20.0 }
    { Name = "Widget C"; Price = 15.0 }
]

// Apply 10% discount
let discounted = products |> List.map (fun p -> { p with Price = p.Price * 0.9 })
discounted |> List.iter (fun p -> printfn "%s: %.2f" p.Name p.Price)

// Extract field
let prices = products |> List.map (fun p -> p.Price)
printfn "Prices: %A" prices

// Map with index
let withIndex = List.mapi (fun i x -> (i + 1, x)) numbers
printfn "With index: %A" withIndex  // [(1, 1); (2, 2); (3, 3); (4, 4); (5, 5)]
```

---

## 6.4 List.filter

```fsharp
// List.filter: เลือก elements ที่ผ่าน predicate

let numbers = [1..20]

// Filter even numbers
let evens = List.filter (fun x -> x % 2 = 0) numbers
printfn "evens: %A" evens

// Filter with pipe
let largeOdds =
    numbers
    |> List.filter (fun x -> x % 2 <> 0)
    |> List.filter (fun x -> x > 10)
printfn "large odds: %A" largeOdds

// Filter with records
type Student = { Name: string; GPA: float; Graduated: bool }

let students = [
    { Name = "Alice"; GPA = 3.8; Graduated = false }
    { Name = "Bob"; GPA = 2.5; Graduated = true }
    { Name = "Charlie"; GPA = 3.5; Graduated = false }
    { Name = "Diana"; GPA = 3.9; Graduated = true }
    { Name = "Eve"; GPA = 2.8; Graduated = false }
]

let honorsStudents = 
    students 
    |> List.filter (fun s -> s.GPA >= 3.5 && not s.Graduated)

printfn "\nHonors students (not graduated):"
honorsStudents |> List.iter (fun s -> printfn "  %s (GPA: %.1f)" s.Name s.GPA)

// List.partition - split into two lists
let (passing, failing) = 
    students |> List.partition (fun s -> s.GPA >= 3.0)

printfn "\nPassing students: %d" passing.Length
printfn "Failing students: %d" failing.Length
```

---

## 6.5 List.fold และ List.foldBack

```fsharp
// List.fold: สะสมค่าจาก left to right
// fold: (state -> element -> state) -> initial state -> list -> final state

let numbers = [1; 2; 3; 4; 5]

// Sum ด้วย fold
let sum = List.fold (fun acc x -> acc + x) 0 numbers
printfn "sum: %d" sum  // 15

// Product
let product = List.fold (fun acc x -> acc * x) 1 numbers
printfn "product: %d" product  // 120

// Max
let maxVal = List.fold max System.Int32.MinValue numbers
printfn "max: %d" maxVal  // 5

// Build list in reverse (using fold)
let reversed = List.fold (fun acc x -> x :: acc) [] numbers
printfn "reversed: %A" reversed  // [5; 4; 3; 2; 1]

// Implement filter using fold
let myFilter predicate lst =
    List.foldBack (fun x acc -> if predicate x then x :: acc else acc) lst []

let evens = myFilter (fun x -> x % 2 = 0) [1..10]
printfn "myFilter evens: %A" evens

// String concatenation
let words = ["Hello"; " "; "World"; "!"]
let sentence = List.fold (+) "" words
printfn "sentence: %s" sentence

// Complex fold: compute statistics
type Stats = { Count: int; Sum: float; Min: float; Max: float }

let computeStats numbers =
    let initial = { Count = 0; Sum = 0.0; Min = infinity; Max = -infinity }
    List.fold 
        (fun stats n -> 
            { Count = stats.Count + 1
              Sum = stats.Sum + n
              Min = min stats.Min n
              Max = max stats.Max n })
        initial
        numbers

let data = [5.5; 3.2; 8.8; 1.1; 9.9; 4.4]
let stats = computeStats data
printfn "\nStats:"
printfn "  Count: %d" stats.Count
printfn "  Sum: %.1f" stats.Sum
printfn "  Min: %.1f" stats.Min
printfn "  Max: %.1f" stats.Max
printfn "  Average: %.2f" (stats.Sum / float stats.Count)
```

---

## 6.6 List.reduce

```fsharp
// List.reduce: เหมือน fold แต่ใช้ element แรกเป็น initial state

let numbers = [1; 2; 3; 4; 5]

// Sum ด้วย reduce
let sum = List.reduce (+) numbers          // 15
let product = List.reduce (*) numbers      // 120
let maxVal = List.reduce max numbers       // 5
let minVal = List.reduce min numbers       // 1

printfn "sum: %d" sum
printfn "product: %d" product
printfn "max: %d" maxVal
printfn "min: %d" minVal

// Reduce strings
let words = ["one"; "two"; "three"]
let combined = List.reduce (fun a b -> a + ", " + b) words
printfn "combined: %s" combined  // "one, two, three"

// ระวัง: reduce ใช้ไม่ได้กับ empty list! (ต้องใช้ fold แทน)
try
    let empty = []
    List.reduce (+) empty |> ignore
with
| :? System.ArgumentException as ex ->
    printfn "Error: %s" ex.Message

// reduceBack (จากขวาไปซ้าย)
let subtracted = List.reduceBack (-) [10; 3; 2; 1]
printfn "reduceBack (-): %d" subtracted  // 10 - (3 - (2 - 1)) = 10 - (3 - 1) = 10 - 2 = 8
```

---

## 6.7 List.iter และ List.iteri

```fsharp
// List.iter: ทำ action กับแต่ละ element

let fruits = ["apple"; "banana"; "cherry"; "date"]

// Basic iter
List.iter (fun f -> printfn "Fruit: %s" f) fruits

// iter กับ pipe
fruits |> List.iter (fun f -> printfn "  - %s" f)

// iteri: iter พร้อม index
List.iteri (fun i f -> printfn "%d. %s" (i+1) f) fruits

// iteri กับ pipe
fruits |> List.iteri (fun i f -> printfn "[%d] %s" i f)

// ตัวอย่างจริง: print table
type Item = { Name: string; Price: float; Qty: int }
let inventory = [
    { Name = "Widget A"; Price = 29.99; Qty = 100 }
    { Name = "Widget B"; Price = 49.99; Qty = 50 }
    { Name = "Gadget C"; Price = 99.99; Qty = 25 }
]

printfn "\n%-5s %-15s %10s %6s %12s" "No." "Name" "Price" "Qty" "Total"
printfn "%s" (String.replicate 52 "-")
inventory |> List.iteri (fun i item ->
    let total = item.Price * float item.Qty
    printfn "%-5d %-15s %10.2f %6d %12.2f" (i+1) item.Name item.Price item.Qty total)
```

---

## 6.8 List.collect (flatMap)

```fsharp
// List.collect: map แล้ว flatten (= flatMap ในภาษาอื่น)

// Basic collect
let numbers = [1; 2; 3]
let expanded = numbers |> List.collect (fun n -> [n; n * 10; n * 100])
printfn "expanded: %A" expanded  // [1; 10; 100; 2; 20; 200; 3; 30; 300]

// Flatten nested lists
let nested = [[1; 2; 3]; [4; 5]; [6; 7; 8; 9]]
let flat = nested |> List.collect id
printfn "flat: %A" flat  // [1; 2; 3; 4; 5; 6; 7; 8; 9]

// Real example: expand tags
type Article = { Title: string; Tags: string list }
let articles = [
    { Title = "F# Introduction"; Tags = ["fsharp"; "programming"; "functional"] }
    { Title = "C# vs F#"; Tags = ["fsharp"; "csharp"; "comparison"] }
    { Title = "Functional Programming"; Tags = ["functional"; "fp"; "programming"] }
]

let allTags = articles |> List.collect (fun a -> a.Tags)
let uniqueTags = allTags |> List.distinct |> List.sort
printfn "All unique tags: %A" uniqueTags

// Orders and items
type Order = { OrderId: int; Items: string list }
let orders = [
    { OrderId = 1; Items = ["apple"; "banana"] }
    { OrderId = 2; Items = ["cherry"] }
    { OrderId = 3; Items = ["date"; "elderberry"; "fig"] }
]

let allItems = orders |> List.collect (fun o -> o.Items)
printfn "All items: %A" allItems
```

---

## 6.9 List.choose

```fsharp
// List.choose: map + filter ในครั้งเดียว
// เหมือน filterMap - เลือก Some values และ unwrap

let tryParseInt (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

let inputs = ["1"; "abc"; "3"; "xyz"; "5"; "6"]
let numbers = inputs |> List.choose tryParseInt
printfn "parsed numbers: %A" numbers  // [1; 3; 5; 6]

// choose กับ Option
let maybeValues = [Some 1; None; Some 3; None; Some 5]
let values = maybeValues |> List.choose id
printfn "values: %A" values  // [1; 3; 5]

// Real example: safe division
let safeDivide a b =
    if b = 0 then None
    else Some (float a / float b)

let dividends = [10; 20; 30; 40; 50]
let divisors = [2; 0; 3; 0; 5]

let results =
    List.zip dividends divisors
    |> List.choose (fun (a, b) -> safeDivide a b)

printfn "division results: %A" results  // [5.0; 10.0; 10.0]

// Product search
type Product = { Id: int; Name: string; Price: float option }
let products = [
    { Id = 1; Name = "Widget A"; Price = Some 29.99 }
    { Id = 2; Name = "Widget B"; Price = None }        // discontinued
    { Id = 3; Name = "Widget C"; Price = Some 19.99 }
    { Id = 4; Name = "Widget D"; Price = None }        // discontinued
]

let availableProducts = 
    products 
    |> List.choose (fun p -> 
        p.Price |> Option.map (fun price -> (p.Name, price)))

printfn "\nAvailable products:"
availableProducts |> List.iter (fun (name, price) -> printfn "  %s: %.2f" name price)
```

---

## 6.10 List.find, List.tryFind, List.exists, List.forall

```fsharp
let numbers = [3; 1; 4; 1; 5; 9; 2; 6; 5; 3]

// List.find: หา element แรกที่ match, throw ถ้าไม่เจอ
let firstEven = List.find (fun x -> x % 2 = 0) numbers
printfn "first even: %d" firstEven  // 4

// List.tryFind: หา element แรกที่ match, return Option
let maybeEven = List.tryFind (fun x -> x % 2 = 0) numbers
let maybeLarge = List.tryFind (fun x -> x > 100) numbers
printfn "tryFind even: %A" maybeEven    // Some 4
printfn "tryFind >100: %A" maybeLarge  // None

// List.findIndex, List.tryFindIndex
let evenIdx = List.findIndex (fun x -> x % 2 = 0) numbers
printfn "index of first even: %d" evenIdx  // 2 (value 4 is at index 2)

// List.exists: มี element ที่ match ไหม?
let hasNegative = List.exists (fun x -> x < 0) numbers
let hasLarge = List.exists (fun x -> x > 8) numbers
printfn "has negative: %b" hasNegative  // false
printfn "has > 8: %b" hasLarge          // true (9)

// List.forall: ทุก element match ไหม?
let allPositive = List.forall (fun x -> x > 0) numbers
let allEven = List.forall (fun x -> x % 2 = 0) numbers
printfn "all positive: %b" allPositive  // true
printfn "all even: %b" allEven          // false

// Practical: validate list
let validateScores scores =
    if scores = [] then
        printfn "Error: No scores"
    elif not (List.forall (fun s -> s >= 0 && s <= 100) scores) then
        printfn "Error: Scores must be 0-100"
    elif not (List.exists (fun s -> s >= 50) scores) then
        printfn "Warning: No passing scores"
    else
        printfn "Scores OK: average = %.1f" (List.averageBy float scores)

validateScores [85; 72; 91; 68; 45]
validateScores [110; 85; 72]
validateScores []
```

---

## 6.11 List.sort, List.sortBy, List.groupBy

```fsharp
// Sorting
let numbers = [5; 3; 8; 1; 9; 2; 7; 4; 6]

// Basic sort (ascending)
let sorted = List.sort numbers
printfn "sorted: %A" sorted

// Sort descending
let sortedDesc = List.sortDescending numbers
printfn "sorted desc: %A" sortedDesc

// sortBy: เรียงตาม key
type Person = { Name: string; Age: int; Score: float }
let people = [
    { Name = "Charlie"; Age = 30; Score = 85.0 }
    { Name = "Alice"; Age = 25; Score = 92.0 }
    { Name = "Bob"; Age = 35; Score = 78.0 }
    { Name = "Diana"; Age = 28; Score = 95.0 }
]

let byName = people |> List.sortBy (fun p -> p.Name)
printfn "\nSorted by name:"
byName |> List.iter (fun p -> printfn "  %s" p.Name)

let byAge = people |> List.sortBy (fun p -> p.Age)
printfn "\nSorted by age:"
byAge |> List.iter (fun p -> printfn "  %s (%d)" p.Name p.Age)

let byScoreDesc = people |> List.sortByDescending (fun p -> p.Score)
printfn "\nSorted by score (desc):"
byScoreDesc |> List.iter (fun p -> printfn "  %s: %.1f" p.Name p.Score)

// groupBy: จัดกลุ่ม
let words = ["apple"; "banana"; "avocado"; "blueberry"; "cherry"; "apricot"]
let grouped = words |> List.groupBy (fun w -> w.[0])  // group by first letter

printfn "\nGrouped by first letter:"
grouped |> List.iter (fun (letter, words) ->
    printfn "  %c: %A" letter words)

// Group people by age range
let ageGroups =
    people
    |> List.groupBy (fun p ->
        if p.Age < 30 then "20s"
        elif p.Age < 40 then "30s"
        else "40+")

printfn "\nPeople by age group:"
ageGroups |> List.iter (fun (group, members) ->
    printfn "  %s: %A" group (members |> List.map (fun p -> p.Name)))
```

---

## 6.12 List.zip, List.unzip

```fsharp
// List.zip: รวม 2 lists เป็น list ของ pairs
let names = ["Alice"; "Bob"; "Charlie"]
let ages = [25; 30; 35]

let zipped = List.zip names ages
printfn "zipped: %A" zipped  // [("Alice", 25); ("Bob", 30); ("Charlie", 35)]

// List.unzip: แยก list ของ pairs เป็น 2 lists
let (names2, ages2) = List.unzip zipped
printfn "names: %A" names2
printfn "ages: %A" ages2

// zip3 สำหรับ 3 lists
let scores = [92.0; 78.0; 85.0]
let zipped3 = List.zip3 names ages scores
printfn "zipped3: %A" zipped3

// Practical: combine student data
type Grade = { Student: string; Subject: string; Score: float }

let students = ["Alice"; "Bob"; "Charlie"]
let mathScores = [90.0; 75.0; 88.0]
let scienceScores = [85.0; 92.0; 70.0]

let mathGrades = 
    List.zip students mathScores
    |> List.map (fun (s, score) -> { Student = s; Subject = "Math"; Score = score })

let scienceGrades =
    List.zip students scienceScores
    |> List.map (fun (s, score) -> { Student = s; Subject = "Science"; Score = score })

let allGrades = mathGrades @ scienceGrades
printfn "\nAll grades:"
allGrades |> List.iter (fun g -> printfn "  %s %s: %.0f" g.Student g.Subject g.Score)
```

---

## 6.13 List Comprehensions

```fsharp
// List comprehensions ด้วย [ for x in ... -> ... ]

// Basic range
let oneToTen = [1..10]
let evens = [2..2..20]
let countdown = [10..-1..1]

printfn "1-10: %A" oneToTen
printfn "evens: %A" evens
printfn "countdown: %A" countdown

// Comprehension with transformation
let squares = [for x in 1..10 -> x * x]
printfn "squares: %A" squares

// Comprehension with filter
let evenSquares = [for x in 1..10 do if x % 2 = 0 then yield x * x]
// หรือสั้นกว่า
let evenSquares2 = [for x in 1..10 do if x % 2 = 0 then x * x]
printfn "even squares: %A" evenSquares

// Nested comprehension (Cartesian product)
let pairs = [for x in 1..3 do for y in 1..3 do (x, y)]
printfn "pairs: %A" pairs

// Multiplication table
let multiTable = [
    for i in 1..5 do
        for j in 1..5 do
            yield (i, j, i * j)
]

printfn "\nMultiplication table (5x5):"
let mutable lastI = 0
for (i, j, product) in multiTable do
    if i <> lastI then
        printfn ""
        lastI <- i
    printf "%4d" product
printfn ""

// Conditional yield
let pythagorean n =
    [for a in 1..n do
        for b in a..n do
            let c = sqrt (float (a*a + b*b))
            if c = floor c && int c <= n then
                yield (a, b, int c)]

printfn "Pythagorean triples (n=20): %A" (pythagorean 20)

// Fibonacci sequence (lazy)
let fibList n =
    let rec gen a b count =
        if count = 0 then []
        else a :: gen b (a + b) (count - 1)
    gen 0 1 n

printfn "First 10 fibs: %A" (fibList 10)
```

---

## 6.14 List.take, List.skip

```fsharp
let numbers = [1..20]

// List.take: เอา n elements แรก
let first5 = List.take 5 numbers
printfn "take 5: %A" first5

// List.skip: ข้าม n elements แรก
let skip5 = List.skip 5 numbers
printfn "skip 5: %A" skip5

// List.truncate: เหมือน take แต่ไม่ throw ถ้า list สั้นกว่า
let safeFirst = List.truncate 5 [1; 2; 3]
printfn "truncate: %A" safeFirst  // [1; 2; 3]

// List.takeWhile: เอา elements ตราบเท่าที่ predicate เป็น true
let lessThan10 = numbers |> List.takeWhile (fun x -> x < 10)
printfn "takeWhile < 10: %A" lessThan10

// List.skipWhile: ข้าม elements ตราบเท่าที่ predicate เป็น true
let startFrom10 = numbers |> List.skipWhile (fun x -> x < 10)
printfn "skipWhile < 10: %A" startFrom10

// Pagination example
let page (pageNum: int) (pageSize: int) (lst: 'a list) =
    lst
    |> List.skip ((pageNum - 1) * pageSize)
    |> List.truncate pageSize

let items = [1..50]
printfn "\nPage 1 (size 10): %A" (page 1 10 items)
printfn "Page 3 (size 10): %A" (page 3 10 items)
printfn "Page 5 (size 10): %A" (page 5 10 items)
```

---

## 6.15 List.rev, List.distinct

```fsharp
// List.rev: กลับลำดับ list
let lst = [1; 2; 3; 4; 5]
let reversed = List.rev lst
printfn "reversed: %A" reversed  // [5; 4; 3; 2; 1]

// Palindrome check
let isPalindrome lst = lst = List.rev lst
printfn "isPalindrome [1;2;1]: %b" (isPalindrome [1; 2; 1])
printfn "isPalindrome [1;2;3]: %b" (isPalindrome [1; 2; 3])

// List.distinct: เอา elements ไม่ซ้ำ
let withDups = [1; 2; 3; 2; 1; 4; 3; 5]
let unique = List.distinct withDups
printfn "distinct: %A" unique  // [1; 2; 3; 4; 5] (preserves first occurrence order)

// distinctBy: unique ตาม key
type Person = { Name: string; City: string }
let people = [
    { Name = "Alice"; City = "Bangkok" }
    { Name = "Bob"; City = "Chiang Mai" }
    { Name = "Charlie"; City = "Bangkok" }
    { Name = "Diana"; City = "Phuket" }
]

let uniqueCities = people |> List.distinctBy (fun p -> p.City)
printfn "\nUnique cities (first person):"
uniqueCities |> List.iter (fun p -> printfn "  %s (%s)" p.City p.Name)
```

---

## 6.16 Sequences (seq)

```fsharp
// Sequences เป็น lazy evaluated collections
// ใช้เมื่อ data มีขนาดใหญ่มากหรือ infinite

// Create sequence
let seq1 = seq { 1; 2; 3; 4; 5 }
let seq2 = Seq.ofList [1; 2; 3]
let seq3 = {1..10}

// Infinite sequence!
let naturals = Seq.initInfinite (fun i -> i + 1)

// เอา 10 ตัวแรกจาก infinite sequence
let first10 = naturals |> Seq.take 10 |> Seq.toList
printfn "First 10 naturals: %A" first10

// Fibonacci sequence (lazy)
let fibSeq =
    Seq.unfold 
        (fun (a, b) -> Some(a, (b, a + b))) 
        (0, 1)

let first15Fibs = fibSeq |> Seq.take 15 |> Seq.toList
printfn "First 15 fibs: %A" first15Fibs

// Sequence operations (same as List but lazy)
let evenSquares =
    Seq.initInfinite (fun i -> i + 1)
    |> Seq.filter (fun x -> x % 2 = 0)
    |> Seq.map (fun x -> x * x)
    |> Seq.take 10
    |> Seq.toList

printfn "Even squares: %A" evenSquares

// yield in sequence
let customSeq = seq {
    yield 1
    yield 2
    yield! [3; 4; 5]    // yield multiple
    for i in 6..10 do
        yield i
}

printfn "Custom seq: %A" (Seq.toList customSeq)

// File lines example (lazy)
// let fileLines = 
//     seq {
//         use reader = System.IO.File.OpenText("file.txt")
//         while not reader.EndOfStream do
//             yield reader.ReadLine()
//     }
```

---

## 6.17 Arrays [||]

```fsharp
// Arrays: mutable, fixed-size, random access O(1)

// Create array
let arr1 = [| 1; 2; 3; 4; 5 |]
let arr2 = Array.create 5 0      // [|0; 0; 0; 0; 0|]
let arr3 = Array.init 5 (fun i -> i * 2)  // [|0; 2; 4; 6; 8|]
let arr4 = [| 1..10 |]

printfn "arr1: %A" arr1
printfn "arr2: %A" arr2
printfn "arr3: %A" arr3

// Array access (mutable)
arr1.[0] <- 10     // arr1 = [|10; 2; 3; 4; 5|]
printfn "after mutation: %A" arr1

// Array operations
printfn "Length: %d" arr4.Length
printfn "Head: %d" arr4.[0]
printfn "Last: %d" arr4.[arr4.Length - 1]

// Array operations (similar to List)
let doubled = arr4 |> Array.map (fun x -> x * 2)
let evens = arr4 |> Array.filter (fun x -> x % 2 = 0)
let sum = arr4 |> Array.sum
let sorted = arr4 |> Array.sort  // sorts in-place!

printfn "doubled: %A" doubled
printfn "evens: %A" evens
printfn "sum: %d" sum

// Array slicing
let slice = arr4.[2..5]   // [|3; 4; 5; 6|]
printfn "slice [2..5]: %A" slice

// Parallel operations (ข้อดีหลักของ Array)
let largeArray = [| 1 .. 1000000 |]
let sumParallel = largeArray |> Array.Parallel.map (fun x -> x * x) |> Array.sum
printfn "Parallel sum of squares: %d" sumParallel

// Convert between List, Seq, Array
let fromList = List.toArray [1; 2; 3]
let fromSeq = Seq.toArray {1..5}
let toList = Array.toList [|1; 2; 3|]
let toSeq = Array.toSeq [|1; 2; 3|]
```

---

## 6.18 ResizeArray<T>

```fsharp
// ResizeArray เป็น alias สำหรับ System.Collections.Generic.List<T>
// Mutable, dynamic size

let resizable = ResizeArray<int>()
resizable.Add(1)
resizable.Add(2)
resizable.Add(3)

printfn "ResizeArray: %A" (resizable |> Seq.toList)

// Initialize
let items = ResizeArray<string>(["apple"; "banana"; "cherry"])

// Operations
items.Add("date")
items.Insert(0, "avocado")    // insert at index
items.Remove("banana") |> ignore
items.RemoveAt(2)              // remove at index

printfn "Items: %A" (items |> Seq.toList)

// อ่านค่า
printfn "Count: %d" items.Count
printfn "Contains 'apple': %b" (items.Contains("apple"))
printfn "Index of 'apple': %d" (items.IndexOf("apple"))

// Sorting
items.Sort()
printfn "Sorted: %A" (items |> Seq.toList)

// Convert
let asList = items |> Seq.toList
let asArray = items.ToArray()

// เมื่อไหร่ควรใช้ ResizeArray:
// - ต้องการ mutable list ที่ grow ได้
// - Interop กับ .NET API ที่ expect List<T>
// - Performance-critical ที่ต้องการ random access
```

---

## 6.19 Map, Set (Functional Data Structures)

```fsharp
// Map: immutable key-value dictionary
let emptyMap: Map<string, int> = Map.empty

// สร้าง Map
let scores = Map.ofList [("Alice", 90); ("Bob", 85); ("Charlie", 92)]
printfn "scores: %A" scores

// เพิ่ม/อัพเดต (return new Map)
let updated = scores |> Map.add "Diana" 88
let aliceUpdated = updated |> Map.add "Alice" 95
printfn "updated: %A" aliceUpdated

// Lookup
printfn "Alice's score: %d" scores.["Alice"]
printfn "Alice (tryFind): %A" (Map.tryFind "Alice" scores)
printfn "Zara (tryFind): %A" (Map.tryFind "Zara" scores)

// Remove
let withoutBob = scores |> Map.remove "Bob"
printfn "without Bob: %A" withoutBob

// Map operations
let doubled = scores |> Map.map (fun _ v -> v * 2)
printfn "doubled scores: %A" doubled

// containsKey
printfn "Has 'Charlie': %b" (Map.containsKey "Charlie" scores)

// Set: immutable set of unique values
let set1 = Set.ofList [1; 2; 3; 4; 5]
let set2 = Set.ofList [3; 4; 5; 6; 7]

// Set operations
let union = Set.union set1 set2
let intersection = Set.intersect set1 set2
let difference = Set.difference set1 set2

printfn "\nSet operations:"
printfn "union: %A" union
printfn "intersection: %A" intersection
printfn "difference: %A" difference

printfn "Contains 3: %b" (Set.contains 3 set1)
printfn "Size: %d" set1.Count
```

---

## 6.20 ตัวอย่างโปรแกรมสมบูรณ์

### 6.20.1 Student Grade Analyzer

```fsharp
// grade_analyzer.fsx - วิเคราะห์เกรดนักเรียน

type Student = {
    Name: string
    StudentId: string
    Scores: float list
}

let students = [
    { Name = "สมชาย"; StudentId = "S001"; Scores = [85.0; 90.0; 78.0; 92.0; 88.0] }
    { Name = "สมหญิง"; StudentId = "S002"; Scores = [72.0; 68.0; 75.0; 80.0; 71.0] }
    { Name = "มานะ"; StudentId = "S003"; Scores = [95.0; 98.0; 92.0; 96.0; 94.0] }
    { Name = "มานี"; StudentId = "S004"; Scores = [45.0; 52.0; 48.0; 60.0; 55.0] }
    { Name = "ปิติ"; StudentId = "S005"; Scores = [60.0; 65.0; 70.0; 68.0; 72.0] }
]

let calculateAverage scores =
    if scores = [] then 0.0
    else List.averageBy id scores

let getGrade avg =
    if avg >= 80.0 then "A"
    elif avg >= 70.0 then "B"
    elif avg >= 60.0 then "C"
    elif avg >= 50.0 then "D"
    else "F"

// วิเคราะห์แต่ละคน
let analysis =
    students
    |> List.map (fun s ->
        let avg = calculateAverage s.Scores
        let grade = getGrade avg
        let highest = List.max s.Scores
        let lowest = List.min s.Scores
        (s.Name, s.StudentId, avg, grade, highest, lowest))
    |> List.sortByDescending (fun (_, _, avg, _, _, _) -> avg)

// แสดงผล
printfn "%-15s %-8s %8s %6s %8s %8s" "ชื่อ" "รหัส" "เฉลี่ย" "เกรด" "สูงสุด" "ต่ำสุด"
printfn "%s" (String.replicate 58 "-")

analysis |> List.iteri (fun rank (name, id, avg, grade, high, low) ->
    printfn "%-15s %-8s %8.2f %6s %8.1f %8.1f" name id avg grade high low)

// สถิติรวม
let allAverages = 
    students 
    |> List.map (fun s -> calculateAverage s.Scores)

printfn "\n%s" (String.replicate 58 "=")
printfn "เฉลี่ยรวม: %.2f" (List.average allAverages)
printfn "เกรด A: %d คน" (analysis |> List.filter (fun (_, _, _, g, _, _) -> g = "A") |> List.length)
printfn "เกรด B: %d คน" (analysis |> List.filter (fun (_, _, _, g, _, _) -> g = "B") |> List.length)
printfn "ผ่าน: %d คน" (analysis |> List.filter (fun (_, _, _, g, _, _) -> g <> "F") |> List.length)
```

---

## สรุป Part 6

ในบทนี้เราได้เรียนรู้:
- ✅ F# List: immutable linked list
- ✅ List literals และ cons operator (::)
- ✅ List.map, List.filter, List.fold
- ✅ List.reduce, List.iter, List.iteri
- ✅ List.mapi, List.collect (flatMap)
- ✅ List.choose (filterMap)
- ✅ List.find, List.tryFind, List.exists, List.forall
- ✅ List.sort, List.sortBy, List.groupBy
- ✅ List.partition, List.zip, List.unzip
- ✅ List.take, List.skip, List.rev, List.distinct
- ✅ List comprehensions
- ✅ Sequences (seq) - lazy evaluation
- ✅ Arrays [||] - mutable, random access
- ✅ ResizeArray<T>
- ✅ Map และ Set

**ใน Part 7** เราจะเรียนรู้ Tuples และ Records - การสร้าง compound types!
