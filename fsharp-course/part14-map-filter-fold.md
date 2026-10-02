# Part 14 - Map, Filter, Fold เชิงลึก (Deep Dive)

## บทนำ (Introduction)

Map, Filter, Fold เป็น 3 operations พื้นฐานที่สุดของ functional programming ที่ช่วยให้เราประมวลผล collections โดยไม่ต้องใช้ loop

---

## 1. List.map Deep Dive

```fsharp
// List.map: แปลงทุก element ด้วยฟังก์ชัน f
// signature: ('a -> 'b) -> 'a list -> 'b list

// พื้นฐาน
let numbers = [1; 2; 3; 4; 5]

let doubled = List.map (fun x -> x * 2) numbers
let squared = List.map (fun x -> x * x) numbers
let asStrings = List.map string numbers
let asFloats = List.map float numbers

printfn "doubled: %A" doubled     // [2; 4; 6; 8; 10]
printfn "squared: %A" squared     // [1; 4; 9; 16; 25]
printfn "asStrings: %A" asStrings // ["1"; "2"; "3"; "4"; "5"]
printfn "asFloats: %A" asFloats   // [1.0; 2.0; 3.0; 4.0; 5.0]
```

```fsharp
// map รักษาจำนวน elements
// ถ้า list มี n elements, ผลลัพธ์ก็มี n elements เสมอ

let original = [1..5]
let mapped = List.map (fun x -> x * 3) original

printfn "original length: %d" (List.length original)  // 5
printfn "mapped length: %d" (List.length mapped)      // 5
```

```fsharp
// map กับ record type
type Circle = { Radius: float }
type Square = { Side: float }

let circles = [
    { Radius = 1.0 }
    { Radius = 2.5 }
    { Radius = 3.0 }
]

let areas = List.map (fun c -> System.Math.PI * c.Radius * c.Radius) circles
let circumferences = List.map (fun c -> 2.0 * System.Math.PI * c.Radius) circles

printfn "areas: %A" areas
printfn "circumferences: %A" circumferences
```

```fsharp
// map กับ Option type
let maybeNumbers = [Some 1; None; Some 3; None; Some 5]

// map ใน option จะทำงานเฉพาะกับ Some
let doubled = List.map (Option.map (fun x -> x * 2)) maybeNumbers

printfn "doubled options: %A" doubled
// [Some 2; None; Some 6; None; Some 10]
```

```fsharp
// List.mapi: map พร้อม index
let words = ["hello"; "world"; "from"; "fsharp"]

let indexed = List.mapi (fun i w -> sprintf "%d: %s" i w) words

printfn "indexed: %A" indexed
// ["0: hello"; "1: world"; "2: from"; "3: fsharp"]
```

```fsharp
// List.map2: map กับ 2 lists พร้อมกัน
let list1 = [1; 2; 3; 4; 5]
let list2 = [10; 20; 30; 40; 50]

let sums = List.map2 (+) list1 list2
let products = List.map2 (*) list1 list2
let pairs = List.map2 (fun x y -> (x, y)) list1 list2

printfn "sums: %A" sums       // [11; 22; 33; 44; 55]
printfn "products: %A" products  // [10; 40; 90; 160; 250]
printfn "pairs: %A" pairs
```

```fsharp
// List.collect: map then flatten
// เหมือน flatMap ใน language อื่น
let expand x = [x; x*2; x*3]

let expanded = List.collect expand [1; 2; 3]
printfn "expanded: %A" expanded  // [1; 2; 3; 2; 4; 6; 3; 6; 9]

// เปรียบเทียบกับ map + concat
let expanded2 = [1; 2; 3] |> List.map expand |> List.concat
printfn "expanded2: %A" expanded2  // same result
```

---

## 2. List.filter Deep Dive

```fsharp
// List.filter: กรอง elements ที่ตรงเงื่อนไข
// signature: ('a -> bool) -> 'a list -> 'a list

let numbers = [1..20]

let evens = List.filter (fun x -> x % 2 = 0) numbers
let odds = List.filter (fun x -> x % 2 <> 0) numbers
let primes = List.filter (fun n ->
    n >= 2 && [2..int(sqrt(float n))] |> List.forall (fun d -> n % d <> 0)) numbers

printfn "evens: %A" evens
printfn "odds: %A" odds
printfn "primes: %A" primes
```

```fsharp
// filter กับ complex predicates
type Student = {
    Name: string
    Grade: float
    Passed: bool
    Department: string
}

let students = [
    { Name = "Alice"; Grade = 85.0; Passed = true; Department = "CS" }
    { Name = "Bob"; Grade = 55.0; Passed = false; Department = "Math" }
    { Name = "Charlie"; Grade = 92.0; Passed = true; Department = "CS" }
    { Name = "Diana"; Grade = 70.0; Passed = true; Department = "Physics" }
    { Name = "Eve"; Grade = 45.0; Passed = false; Department = "CS" }
]

let passedCS = students |> List.filter (fun s -> s.Passed && s.Department = "CS")
let highGraders = students |> List.filter (fun s -> s.Grade >= 80.0)
let failingStudents = students |> List.filter (fun s -> not s.Passed)

printfn "Passed CS: %A" (List.map (fun s -> s.Name) passedCS)
printfn "High Graders: %A" (List.map (fun s -> s.Name) highGraders)
printfn "Failing: %A" (List.map (fun s -> s.Name) failingStudents)
```

```fsharp
// List.choose: filter + map ในครั้งเดียว
// กรองและแปลงพร้อมกัน โดยใช้ Option
let tryParse (s: string) =
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None

let input = ["1"; "hello"; "3"; "world"; "5"; "foo"; "7"]

// แบบ filter then map (ทำสองขั้น)
let numbers1 =
    input
    |> List.choose tryParse  // ทำในขั้นเดียว

printfn "parsed numbers: %A" numbers1  // [1; 3; 5; 7]
```

```fsharp
// List.partition: แบ่ง list ออกเป็น 2 ส่วน
let numbers = [1..10]

let evens, odds = List.partition (fun x -> x % 2 = 0) numbers

printfn "evens: %A" evens  // [2; 4; 6; 8; 10]
printfn "odds: %A" odds    // [1; 3; 5; 7; 9]
```

```fsharp
// List.groupBy: จัดกลุ่ม elements
type Product = { Name: string; Category: string; Price: float }

let products = [
    { Name = "Laptop"; Category = "Electronics"; Price = 999.99 }
    { Name = "Book"; Category = "Education"; Price = 15.99 }
    { Name = "Phone"; Category = "Electronics"; Price = 699.99 }
    { Name = "Course"; Category = "Education"; Price = 49.99 }
    { Name = "Tablet"; Category = "Electronics"; Price = 449.99 }
]

let byCategory = List.groupBy (fun p -> p.Category) products

byCategory |> List.iter (fun (cat, items) ->
    printfn "%s: %A" cat (List.map (fun p -> p.Name) items))
```

---

## 3. List.fold Step-by-Step Explanation

```fsharp
// List.fold: วนรวม elements ด้วย accumulator
// signature: ('state -> 'a -> 'state) -> 'state -> 'a list -> 'state

// การทำงานของ fold ทีละขั้น:
// fold f init [a; b; c] =
//   step1: state = f init a
//   step2: state = f state b  
//   step3: state = f state c
//   return: state

let trace label x =
    printfn "  %s: %A" label x
    x

let numbers = [1; 2; 3; 4; 5]

// fold พร้อม tracing
let result = List.fold (fun acc x ->
    let newAcc = acc + x
    printfn "  acc=%d + x=%d = %d" acc x newAcc
    newAcc) 0 numbers

printfn "Final: %d" result
```

```fsharp
// fold สำหรับการคำนวณต่างๆ

let numbers = [1; 2; 3; 4; 5]

// ผลรวม
let sum = List.fold (+) 0 numbers                   // 15

// ผลคูณ
let product = List.fold (*) 1 numbers                // 120

// ค่าสูงสุด
let maxVal = List.fold max System.Int32.MinValue numbers   // 5

// ค่าต่ำสุด
let minVal = List.fold min System.Int32.MaxValue numbers   // 1

// นับ elements
let count = List.fold (fun acc _ -> acc + 1) 0 numbers  // 5

printfn "sum=%d product=%d max=%d min=%d count=%d"
    sum product maxVal minVal count
```

```fsharp
// fold สำหรับสร้าง data structures ใหม่

let numbers = [1; 2; 3; 4; 5]

// สร้าง list ใหม่ (เหมือน map)
let doubled = List.fold (fun acc x -> acc @ [x * 2]) [] numbers
// หมายเหตุ: @ ช้ากว่า :: ถ้า list ยาว ควรใช้แล้ว reverse

// สร้าง string
let asString = List.fold (fun acc x ->
    if acc = "" then string x
    else acc + ", " + string x) "" numbers

// สร้าง Map
let indexMap = List.fold (fun (acc, i) x ->
    (Map.add i x acc, i + 1)) (Map.empty, 0) numbers
    |> fst

printfn "doubled: %A" doubled
printfn "asString: %s" asString
printfn "indexMap: %A" indexMap
```

```fsharp
// fold สำหรับ statistics
let data = [85.0; 92.0; 78.0; 95.0; 88.0; 73.0]

let stats = List.fold (fun (count, sum, minV, maxV) x ->
    let newCount = count + 1
    let newSum = sum + x
    let newMin = min minV x
    let newMax = max maxV x
    (newCount, newSum, newMin, newMax))
    (0, 0.0, System.Double.MaxValue, System.Double.MinValue)
    data

let count, sum, minV, maxV = stats
let avg = sum / float count

printfn "Count: %d" count
printfn "Sum: %.1f" sum
printfn "Average: %.2f" avg
printfn "Min: %.1f" minV
printfn "Max: %.1f" maxV
```

---

## 4. List.foldBack

```fsharp
// List.foldBack: เหมือน fold แต่วนจากขวาไปซ้าย
// signature: ('a -> 'state -> 'state) -> 'a list -> 'state -> 'state

// สังเกต: ลำดับ argument ของ f กลับกัน!
// fold:     f acc element
// foldBack: f element acc

let numbers = [1; 2; 3; 4; 5]

// foldBack รักษาลำดับ list
let copy = List.foldBack (fun x acc -> x :: acc) numbers []
printfn "copy: %A" copy  // [1; 2; 3; 4; 5]

// fold จะ reverse
let reversed = List.fold (fun acc x -> x :: acc) [] numbers
printfn "reversed: %A" reversed  // [5; 4; 3; 2; 1]
```

```fsharp
// เปรียบเทียบ fold vs foldBack

let numbers = [1; 2; 3; 4; 5]

// fold: ซ้ายไปขวา ((((0 - 1) - 2) - 3) - 4) - 5 = -15
let foldResult = List.fold (-) 0 numbers
printfn "fold result: %d" foldResult  // -15

// foldBack: ขวาไปซ้าย 1 - (2 - (3 - (4 - (5 - 0)))) = 3
let foldBackResult = List.foldBack (-) numbers 0
printfn "foldBack result: %d" foldBackResult  // 3
```

```fsharp
// foldBack สำหรับสร้าง tree structure
type Tree<'a> =
    | Leaf
    | Node of 'a * Tree<'a> * Tree<'a>

// สร้าง BST จาก list ด้วย foldBack
let insertBST x tree =
    let rec insert = function
        | Leaf -> Node(x, Leaf, Leaf)
        | Node(v, l, r) ->
            if x < v then Node(v, insert l, r)
            elif x > v then Node(v, l, insert r)
            else Node(v, l, r)
    insert tree

let bst = List.foldBack insertBST [5; 3; 7; 1; 4; 6; 8] Leaf
printfn "BST: %A" bst
```

---

## 5. List.reduce

```fsharp
// List.reduce: fold ที่ใช้ element แรกเป็น initial value
// ต้องมีอย่างน้อย 1 element!
// signature: ('a -> 'a -> 'a) -> 'a list -> 'a

let numbers = [3; 1; 4; 1; 5; 9; 2; 6]

let sum = List.reduce (+) numbers
let max' = List.reduce max numbers
let min' = List.reduce min numbers

printfn "sum: %d" sum   // 31
printfn "max: %d" max'  // 9
printfn "min: %d" min'  // 1
```

```fsharp
// เมื่อไหรใช้ reduce vs fold?

// ใช้ fold เมื่อ:
// 1. list อาจว่าง
// 2. ต้องการ initial value ที่ต่างจาก type
// 3. ต้องการ state ที่ต่างจาก element type

// ใช้ reduce เมื่อ:
// 1. รับรองว่า list ไม่ว่าง
// 2. operation มี identity element ชัดเจน (แต่ไม่สะดวกเขียน)
// 3. type ของ result เหมือนกับ element

let words = ["Hello"; "World"; "From"; "F#"]
let sentence = List.reduce (fun acc w -> acc + " " + w) words
printfn "sentence: %s" sentence
```

```fsharp
// List.reduceBack: เหมือน reduce แต่จากขวาไปซ้าย

let words = ["Hello"; "World"; "From"; "F#"]

let left = List.reduce (fun acc w -> sprintf "(%s %s)" acc w) words
let right = List.reduceBack (fun w acc -> sprintf "(%s %s)" w acc) words

printfn "reduceLeft: %s" left   // (((Hello World) From) F#)
printfn "reduceBack: %s" right  // (Hello (World (From F#)))
```

---

## 6. Building map using fold

```fsharp
// สร้าง map โดยใช้ fold
let myMap f lst =
    List.foldBack (fun x acc -> f x :: acc) lst []

// ทดสอบ
let doubled = myMap (fun x -> x * 2) [1; 2; 3; 4; 5]
printfn "myMap doubled: %A" doubled  // [2; 4; 6; 8; 10]

// เปรียบเทียบกับ List.map
let doubled2 = List.map (fun x -> x * 2) [1; 2; 3; 4; 5]
printfn "List.map doubled: %A" doubled2  // same
```

```fsharp
// ทำไม foldBack ดีกว่า fold สำหรับ map?
// เพราะ foldBack รักษาลำดับ

let myMapFold f lst =
    List.fold (fun acc x -> acc @ [f x]) [] lst  // ช้า! O(n^2)

let myMapFoldBack f lst =
    List.foldBack (fun x acc -> f x :: acc) lst []  // เร็วกว่า O(n)

let data = [1..5]
printfn "fold map: %A" (myMapFold ((*) 2) data)
printfn "foldBack map: %A" (myMapFoldBack ((*) 2) data)
```

---

## 7. Building filter using fold

```fsharp
// สร้าง filter โดยใช้ fold
let myFilter predicate lst =
    List.foldBack (fun x acc ->
        if predicate x then x :: acc
        else acc) lst []

// ทดสอบ
let evens = myFilter (fun x -> x % 2 = 0) [1..10]
printfn "myFilter evens: %A" evens  // [2; 4; 6; 8; 10]

// สร้าง filterMap (filter then map)
let myFilterMap f lst =
    List.foldBack (fun x acc ->
        match f x with
        | Some v -> v :: acc
        | None -> acc) lst []

let input = ["1"; "hello"; "3"; "world"; "5"]
let parsed = myFilterMap (fun s ->
    match System.Int32.TryParse(s) with
    | true, n -> Some n
    | false, _ -> None) input

printfn "parsed: %A" parsed  // [1; 3; 5]
```

---

## 8. scan and scanBack

```fsharp
// List.scan: เหมือน fold แต่เก็บทุก intermediate state
// signature: ('state -> 'a -> 'state) -> 'state -> 'a list -> 'state list

let numbers = [1; 2; 3; 4; 5]

// running sum
let runningSums = List.scan (+) 0 numbers
printfn "running sums: %A" runningSums
// [0; 1; 3; 6; 10; 15]  (เริ่มจาก initial state)

// running product
let runningProducts = List.scan (*) 1 numbers
printfn "running products: %A" runningProducts
// [1; 1; 2; 6; 24; 120]
```

```fsharp
// scan สำหรับ animation/visualization
let velocities = [0; 10; 20; 15; 5; 0; -5; -10]

// คำนวณตำแหน่ง (integral of velocity)
let positions = List.scan (+) 0 velocities
printfn "positions: %A" positions

// คำนวณ cumulative max
let cumulativeMax = List.scan max System.Int32.MinValue numbers
printfn "cumulative max: %A" cumulativeMax
```

```fsharp
// List.scanBack: เหมือน scan แต่จากขวาไปซ้าย

let numbers = [1; 2; 3; 4; 5]

let rightSums = List.scanBack (+) numbers 0
printfn "right sums: %A" rightSums
// [15; 14; 12; 9; 5; 0]  (suffix sums)
```

---

## 9. unfold

```fsharp
// List.unfold: สร้าง list จาก state
// เป็น "ด้านตรงข้าม" ของ fold
// signature: ('state -> ('output * 'state) option) -> 'state -> 'output list

// สร้าง list จาก state
// ถ้า None: หยุด
// ถ้า Some (element, nextState): ผลิต element และดำเนินต่อ

// ตัวเลข 1 ถึง n
let range n = List.unfold (fun i -> if i > n then None else Some (i, i + 1)) 1

printfn "range 5: %A" (range 5)   // [1; 2; 3; 4; 5]
printfn "range 10: %A" (range 10) // [1..10]
```

```fsharp
// unfold สำหรับ Fibonacci
let fibonacci n =
    List.unfold (fun (a, b) ->
        if a > n then None
        else Some (a, (b, a + b))) (0, 1)

printfn "fibonacci 100: %A" (fibonacci 100)
// [0; 1; 1; 2; 3; 5; 8; 13; 21; 34; 55; 89]
```

```fsharp
// unfold สำหรับ string splitting แบบ custom
let splitOn (sep: char) (s: string) =
    let rec split' pos acc =
        if pos >= s.Length then List.rev acc
        else
            match s.IndexOf(sep, pos) with
            | -1 -> List.rev (s.Substring(pos) :: acc)
            | idx -> split' (idx + 1) (s.Substring(pos, idx - pos) :: acc)
    split' 0 []

let result = splitOn ',' "a,b,c,d,e"
printfn "split: %A" result  // ["a"; "b"; "c"; "d"; "e"]
```

```fsharp
// unfold สำหรับ binary representation
let toBinary n =
    if n = 0 then [0]
    else
        List.unfold (fun x ->
            if x = 0 then None
            else Some (x % 2, x / 2)) n
        |> List.rev

printfn "toBinary 10: %A" (toBinary 10)   // [1; 0; 1; 0]
printfn "toBinary 255: %A" (toBinary 255) // [1; 1; 1; 1; 1; 1; 1; 1]
```

---

## 10. Seq.map (Lazy Evaluation)

```fsharp
// Seq.map ทำงานแบบ lazy (ประเมินเมื่อต้องการ)
// เหมาะกับ infinite sequences หรือ large data

// สร้าง infinite sequence
let naturalNumbers = Seq.initInfinite id

// ใช้ Seq.map กับ infinite sequence ได้เพราะ lazy
let squares = Seq.map (fun x -> x * x) naturalNumbers

// ดึงเฉพาะ 10 ตัวแรก
let first10Squares = squares |> Seq.take 10 |> Seq.toList
printfn "first 10 squares: %A" first10Squares
```

```fsharp
// เปรียบเทียบ performance: Seq vs List

open System.Diagnostics

// List.map: สร้างทุก element ทันที
let sw1 = Stopwatch.StartNew()
let listResult =
    [1..1000000]
    |> List.map (fun x -> x * 2)
    |> List.filter (fun x -> x % 3 = 0)
    |> List.take 5
sw1.Stop()
printfn "List took: %dms, result: %A" sw1.ElapsedMilliseconds listResult

// Seq.map: ประเมินเฉพาะที่ต้องการ
let sw2 = Stopwatch.StartNew()
let seqResult =
    Seq.initInfinite (fun x -> x + 1)
    |> Seq.map (fun x -> x * 2)
    |> Seq.filter (fun x -> x % 3 = 0)
    |> Seq.take 5
    |> Seq.toList
sw2.Stop()
printfn "Seq took: %dms, result: %A" sw2.ElapsedMilliseconds seqResult
```

---

## 11. Array.map (Mutable)

```fsharp
// Array.map: เหมือน List.map แต่ทำงานกับ array
// Array ใน .NET เป็น mutable

let arr = [|1; 2; 3; 4; 5|]

let doubled = Array.map (fun x -> x * 2) arr
let squared = Array.map (fun x -> x * x) arr

printfn "doubled: %A" doubled  // [|2; 4; 6; 8; 10|]
printfn "squared: %A" squared  // [|1; 4; 9; 16; 25|]

// Array.map สร้าง array ใหม่ (immutable result)
// arr ไม่เปลี่ยนแปลง
printfn "original: %A" arr  // [|1; 2; 3; 4; 5|]
```

```fsharp
// Array.mapi: map พร้อม index
let arr = [|"a"; "b"; "c"; "d"|]

let indexed = Array.mapi (fun i s -> sprintf "%d:%s" i s) arr
printfn "indexed: %A" indexed  // [|"0:a"; "1:b"; "2:c"; "3:d"|]
```

```fsharp
// Array.map2: map กับ 2 arrays
let a = [|1; 2; 3|]
let b = [|10; 20; 30|]

let sums = Array.map2 (+) a b
printfn "sums: %A" sums  // [|11; 22; 33|]
```

```fsharp
// เมื่อไหรใช้ Array vs List?
// Array:
//   - ต้องการ random access O(1)
//   - ใช้กับ .NET interop
//   - ต้องการ performance สูงสุด
//   - data ขนาดใหญ่ที่ต้องการ mutable

// List:
//   - functional style (immutable by default)
//   - prepend O(1)
//   - pattern matching
//   - ส่วนใหญ่ใช้ List ใน F# idiomatic code
```

---

## 12. Map.map (Dictionary-like)

```fsharp
// Map ใน F# คือ persistent ordered dictionary (immutable)
// Map.map: แปลง values ใน Map

let scores = Map.ofList [("Alice", 85); ("Bob", 72); ("Charlie", 91)]

// เพิ่ม 10 คะแนนทุกคน
let bonusScores = Map.map (fun name score -> score + 10) scores

printfn "original: %A" scores
printfn "with bonus: %A" bonusScores
```

```fsharp
// Map operations
let inventory = Map.ofList [
    ("apple", 50)
    ("banana", 30)
    ("cherry", 10)
]

// map ทุก value
let doubled = Map.map (fun _ qty -> qty * 2) inventory

// filter ตาม value
let inStock = Map.filter (fun _ qty -> qty > 0) inventory
let lowStock = Map.filter (fun _ qty -> qty < 20) inventory

// fold เพื่อคำนวณ total
let total = Map.fold (fun acc _ qty -> acc + qty) 0 inventory

printfn "inventory: %A" inventory
printfn "doubled: %A" doubled
printfn "inStock: %A" inStock
printfn "lowStock: %A" lowStock
printfn "total: %d" total
```

```fsharp
// Map.map เพื่อ transform keys ด้วย
let priceMap = Map.ofList [
    ("item1", 10.0)
    ("item2", 20.0)
    ("item3", 30.0)
]

// เพิ่ม 10% ราคา
let withTax = priceMap |> Map.map (fun _ price -> price * 1.1)

// สร้าง Map ใหม่จาก transform ทั้ง key และ value
let formatted = priceMap |> Map.toList |> List.map (fun (k, v) -> (k.ToUpper(), sprintf "$%.2f" v)) |> Map.ofList

printfn "withTax: %A" withTax
printfn "formatted: %A" formatted
```

---

## 13. Custom Fold-Based Algorithms

```fsharp
// ใช้ fold สำหรับ algorithm ต่างๆ

// 1. Run-Length Encoding
let rleEncode lst =
    List.fold (fun acc x ->
        match acc with
        | [] -> [(1, x)]
        | (count, v) :: rest when v = x -> (count + 1, v) :: rest
        | _ -> (1, x) :: acc) [] lst
    |> List.rev

let input = ['a'; 'a'; 'b'; 'b'; 'b'; 'c'; 'a'; 'a']
let encoded = rleEncode input
printfn "RLE: %A" encoded  // [(2,'a'); (3,'b'); (1,'c'); (2,'a')]
```

```fsharp
// 2. Tokenizer ง่ายๆ
type Token = Word of string | Number of int | Punctuation of char

let tokenize (input: string) =
    let processChar (acc: Token list * string) (c: char) =
        let tokens, current = acc
        match c with
        | c when System.Char.IsLetter(c) -> (tokens, current + string c)
        | c when System.Char.IsDigit(c) -> (tokens, current + string c)
        | ' ' | '\t' ->
            if current = "" then (tokens, "")
            else
                let token =
                    match System.Int32.TryParse(current) with
                    | true, n -> Number n
                    | false, _ -> Word current
                (token :: tokens, "")
        | c ->
            let newTokens =
                if current = "" then Punctuation c :: tokens
                else
                    let token =
                        match System.Int32.TryParse(current) with
                        | true, n -> Number n
                        | false, _ -> Word current
                    Punctuation c :: token :: tokens
            (newTokens, "")
    
    let finalTokens, remaining = input.ToCharArray() |> Array.fold processChar ([], "")
    let allTokens =
        if remaining <> "" then
            let token =
                match System.Int32.TryParse(remaining) with
                | true, n -> Number n
                | false, _ -> Word remaining
            token :: finalTokens
        else finalTokens
    List.rev allTokens

let tokens = tokenize "hello 42 world! foo 123"
printfn "tokens: %A" tokens
```

```fsharp
// 3. Matrix multiplication ด้วย fold
type Matrix = float[,]

let matMul (a: Matrix) (b: Matrix) =
    let rows = Array2D.length1 a
    let cols = Array2D.length2 b
    let k = Array2D.length2 a
    
    Array2D.init rows cols (fun i j ->
        [0..k-1]
        |> List.fold (fun acc n -> acc + a.[i, n] * b.[n, j]) 0.0)

let a = array2D [[1.0; 2.0]; [3.0; 4.0]]
let b = array2D [[5.0; 6.0]; [7.0; 8.0]]
let c = matMul a b

printfn "Matrix result:"
for i in 0..1 do
    for j in 0..1 do
        printf "%.0f " c.[i, j]
    printfn ""
```

```fsharp
// 4. Stack-based calculator
type StackOp = Push of float | Pop | Add | Subtract | Multiply | Divide

let evaluate ops =
    List.fold (fun stack op ->
        match op, stack with
        | Push n, _ -> n :: stack
        | Add, y :: x :: rest -> (x + y) :: rest
        | Subtract, y :: x :: rest -> (x - y) :: rest
        | Multiply, y :: x :: rest -> (x * y) :: rest
        | Divide, y :: x :: rest when y <> 0.0 -> (x / y) :: rest
        | Pop, _ :: rest -> rest
        | _ -> stack) [] ops

// 3 + 4 * 2 = 11
let program = [
    Push 3.0
    Push 4.0
    Push 2.0
    Multiply   // 4 * 2 = 8
    Add        // 3 + 8 = 11
]

let result = evaluate program
printfn "Stack result: %A" result  // [11.0]
```

---

## สรุป (Summary)

```fsharp
// สรุปการใช้งาน Map, Filter, Fold

// map: แปลง elements
let doubled = [1..5] |> List.map ((*) 2)        // [2; 4; 6; 8; 10]

// filter: กรอง elements
let evens = [1..10] |> List.filter (fun x -> x % 2 = 0)  // [2; 4; 6; 8; 10]

// fold: รวมเป็นค่าเดียว
let sum = [1..10] |> List.fold (+) 0             // 55

// scan: รวมพร้อมเก็บ intermediate states
let sums = [1..5] |> List.scan (+) 0             // [0; 1; 3; 6; 10; 15]

// choose: filter + map
let evens' = [1..10] |> List.choose (fun x -> if x % 2 = 0 then Some (x/2) else None)

// unfold: สร้าง list จาก state
let powers2 = List.unfold (fun n -> if n > 100 then None else Some(n, n*2)) 1

printfn "powers of 2: %A" powers2  // [1; 2; 4; 8; 16; 32; 64]

// ใช้ร่วมกัน
let result =
    [1..20]
    |> List.filter (fun x -> x % 3 = 0)   // [3; 6; 9; 12; 15; 18]
    |> List.map (fun x -> x * x)           // [9; 36; 81; 144; 225; 324]
    |> List.fold (+) 0                     // 819

printfn "result: %d" result
```

Map, Filter, Fold เป็นเครื่องมือที่ทรงพลัง ช่วยให้เราเขียนโค้ดที่:
- **Declarative**: บอกว่าต้องการอะไร ไม่ใช่ทำอย่างไร
- **Composable**: รวมกันได้ง่าย
- **Testable**: ทดสอบแต่ละส่วนได้อิสระ
- **Readable**: อ่านเข้าใจง่ายกว่า loop แบบ imperative
