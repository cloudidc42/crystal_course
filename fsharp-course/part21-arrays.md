# Part 21 - อาร์เรย์ (Arrays)

## บทนำ (Introduction)

อาร์เรย์ใน F# เป็นโครงสร้างข้อมูลที่เก็บข้อมูลในหน่วยความจำแบบต่อเนื่อง (contiguous memory) ทำให้การเข้าถึงข้อมูลด้วย index มีประสิทธิภาพสูง O(1) อาร์เรย์ใน F# เป็น **mutable** ต่างจาก List ที่เป็น immutable

```fsharp
// อาร์เรย์ใน F# ใช้ bracket [| |]
let numbers = [| 1; 2; 3; 4; 5 |]
printfn "Array: %A" numbers
// Output: Array: [|1; 2; 3; 4; 5|]

// เปรียบเทียบกับ List ที่ใช้ [ ]
let list = [ 1; 2; 3; 4; 5 ]
printfn "List: %A" list
// Output: List: [1; 2; 3; 4; 5]
```

---

## 21.1 Array Literals (การสร้างอาร์เรย์แบบ Literal)

```fsharp
// อาร์เรย์ว่าง
let emptyArray : int[] = [||]
printfn "Empty: %A, Length: %d" emptyArray emptyArray.Length

// อาร์เรย์ของ int
let intArray = [| 10; 20; 30; 40; 50 |]

// อาร์เรย์ของ string
let strArray = [| "apple"; "banana"; "cherry" |]

// อาร์เรย์ของ float
let floatArray = [| 1.1; 2.2; 3.3; 4.4 |]

// อาร์เรย์ของ bool
let boolArray = [| true; false; true; true; false |]

// อาร์เรย์ของ tuple
let tupleArray = [| (1, "one"); (2, "two"); (3, "three") |]

// อาร์เรย์ของ record
type Point = { X: float; Y: float }
let points = [| { X = 0.0; Y = 0.0 }; { X = 1.0; Y = 1.0 }; { X = 2.0; Y = 4.0 } |]

// อาร์เรย์ใช้ range expression
let rangeArray = [| 1..10 |]
printfn "Range: %A" rangeArray
// Output: Range: [|1; 2; 3; 4; 5; 6; 7; 8; 9; 10|]

// อาร์เรย์ใช้ step
let stepArray = [| 0..2..20 |]
printfn "Step 2: %A" stepArray
// Output: Step 2: [|0; 2; 4; 6; 8; 10; 12; 14; 16; 18; 20|]

// อาร์เรย์ลดลง
let downArray = [| 10..-1..1 |]
printfn "Down: %A" downArray
// Output: Down: [|10; 9; 8; 7; 6; 5; 4; 3; 2; 1|]

// อาร์เรย์ใช้ comprehension
let squares = [| for i in 1..10 -> i * i |]
printfn "Squares: %A" squares

// อาร์เรย์ใช้ comprehension กับ condition
let evenSquares = [| for i in 1..20 do if i % 2 = 0 then yield i * i |]
printfn "Even Squares: %A" evenSquares

// อาร์เรย์ nested comprehension
let matrix = [| for i in 1..3 do for j in 1..3 do yield (i, j) |]
printfn "Matrix pairs: %A" matrix
```

---

## 21.2 Array Creation Functions (ฟังก์ชันสร้างอาร์เรย์)

```fsharp
// Array.create - สร้างอาร์เรย์ขนาด n ด้วยค่าเริ่มต้น
let arr1 = Array.create 5 0        // [|0; 0; 0; 0; 0|]
let arr2 = Array.create 3 "hello"  // [|"hello"; "hello"; "hello"|]
let arr3 = Array.create 4 true     // [|true; true; true; true|]

printfn "create 5 zeros: %A" arr1
printfn "create 3 hellos: %A" arr2

// Array.init - สร้างอาร์เรย์ขนาด n ด้วยฟังก์ชัน
let arr4 = Array.init 5 (fun i -> i)           // [|0; 1; 2; 3; 4|]
let arr5 = Array.init 5 (fun i -> i * i)       // [|0; 1; 4; 9; 16|]
let arr6 = Array.init 5 (fun i -> float i / 2.0) // [|0.0; 0.5; 1.0; 1.5; 2.0|]
let arr7 = Array.init 5 (fun i -> sprintf "item%d" i) // [|"item0"; ...|]

printfn "init i: %A" arr4
printfn "init i*i: %A" arr5

// Array.zeroCreate - สร้างอาร์เรย์ขนาด n ด้วยค่า default (0 สำหรับตัวเลข)
let zeros = Array.zeroCreate<int> 5        // [|0; 0; 0; 0; 0|]
let floatZeros = Array.zeroCreate<float> 4  // [|0.0; 0.0; 0.0; 0.0|]
let strNulls = Array.zeroCreate<string> 3   // [|null; null; null|]

printfn "zeroCreate int: %A" zeros
printfn "zeroCreate float: %A" floatZeros

// สร้างอาร์เรย์จาก seq
let seqArr = Array.ofSeq (seq { 1..5 })
printfn "ofSeq: %A" seqArr

// สร้างอาร์เรย์จาก list
let listArr = Array.ofList [1; 2; 3; 4; 5]
printfn "ofList: %A" listArr

// สร้างอาร์เรย์ซ้ำ
let repeated = Array.replicate 4 "abc"
printfn "replicate: %A" repeated
// Output: replicate: [|"abc"; "abc"; "abc"; "abc"|]

// การสร้างอาร์เรย์ด้วย unfold
let fibonacci = 
    Array.unfold 
        (fun (a, b) -> if a > 100 then None else Some(a, (b, a + b))) 
        (0, 1)
printfn "Fibonacci: %A" fibonacci
```

---

## 21.3 Array Access (การเข้าถึงอาร์เรย์)

```fsharp
let arr = [| 10; 20; 30; 40; 50 |]

// การเข้าถึงด้วย index (0-based)
let first = arr.[0]    // 10
let second = arr.[1]   // 20
let last = arr.[4]     // 50

printfn "First: %d" first
printfn "Second: %d" second
printfn "Last: %d" last

// ใน F# 6+ สามารถใช้แบบนี้ได้
let first2 = arr[0]    // ไม่ต้องมี dot
let last2 = arr[arr.Length - 1]

// ใช้ Array.get
let third = Array.get arr 2
printfn "Third (Array.get): %d" third

// ใช้ index ลบ (F# 8+)
// let lastItem = arr[^1]   // ตัวสุดท้าย

// Length property
printfn "Length: %d" arr.Length

// Array.length function
printfn "Array.length: %d" (Array.length arr)

// การตรวจสอบก่อนเข้าถึง
let safeGet (arr: 'a[]) (index: int) =
    if index >= 0 && index < arr.Length then
        Some arr.[index]
    else
        None

printfn "Safe get 2: %A" (safeGet arr 2)    // Some 30
printfn "Safe get 10: %A" (safeGet arr 10)  // None

// Array.tryItem - ปลอดภัยกว่า
let item2 = Array.tryItem 2 arr    // Some 30
let item10 = Array.tryItem 10 arr  // None
printfn "tryItem 2: %A" item2
printfn "tryItem 10: %A" item10

// Array.item - เหมือน .[i] แต่อาจ throw exception
let item0 = Array.item 0 arr   // 10

// การค้นหาด้วย head/tail concept
let arrayHead = Array.head arr      // 10
let arrayTail = Array.tail arr      // [|20; 30; 40; 50|]
let arrayLast = Array.last arr      // 50

printfn "Head: %d" arrayHead
printfn "Tail: %A" arrayTail
printfn "Last: %d" arrayLast
```

---

## 21.4 Array Mutation (การเปลี่ยนแปลงอาร์เรย์)

```fsharp
// อาร์เรย์ใน F# เป็น mutable
let mutable arr = [| 1; 2; 3; 4; 5 |]

// การแก้ไขด้วย assignment
arr.[0] <- 10
arr.[2] <- 30
arr.[4] <- 50

printfn "After mutation: %A" arr
// Output: After mutation: [|10; 2; 30; 4; 50|]

// Array.set - ฟังก์ชันสำหรับ set ค่า
Array.set arr 1 20
Array.set arr 3 40
printfn "After Array.set: %A" arr
// Output: After Array.set: [|10; 20; 30; 40; 50|]

// การแก้ไขทั้งหมดในลูป
let fillArray = Array.create 5 0
for i in 0..fillArray.Length - 1 do
    fillArray.[i] <- i * 2

printfn "Filled: %A" fillArray
// Output: Filled: [|0; 2; 4; 6; 8|]

// Array.fill - เติมค่าในช่วง
let arr2 = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]
Array.fill arr2 2 4 0   // เริ่มที่ index 2, จำนวน 4 ตัว, ใส่ค่า 0
printfn "After fill: %A" arr2
// Output: After fill: [|1; 2; 0; 0; 0; 0; 7; 8; 9; 10|]

// การแก้ไขด้วย map แบบ in-place (สร้างใหม่)
let original = [| 1; 2; 3; 4; 5 |]
let doubled = Array.map (fun x -> x * 2) original
printfn "Original: %A" original    // ไม่เปลี่ยน
printfn "Doubled: %A" doubled

// การแก้ไขด้วย iteri
let mutable arr3 = [| 1; 2; 3; 4; 5 |]
Array.iteri (fun i x -> arr3.[i] <- x * 3) arr3
printfn "After iteri x3: %A" arr3

// Swap elements
let swap (arr: 'a[]) i j =
    let temp = arr.[i]
    arr.[i] <- arr.[j]
    arr.[j] <- temp

let swapArr = [| 1; 2; 3; 4; 5 |]
swap swapArr 0 4
printfn "After swap 0 4: %A" swapArr
// Output: After swap 0 4: [|5; 2; 3; 4; 1|]
```

---

## 21.5 Array Slicing (การตัดอาร์เรย์)

```fsharp
let arr = [| 0; 1; 2; 3; 4; 5; 6; 7; 8; 9 |]

// Slicing ด้วย range
let slice1 = arr.[1..3]    // [|1; 2; 3|]
let slice2 = arr.[..4]     // [|0; 1; 2; 3; 4|] (ตั้งแต่ต้น)
let slice3 = arr.[5..]     // [|5; 6; 7; 8; 9|] (จนถึงท้าย)
let slice4 = arr.[2..7]    // [|2; 3; 4; 5; 6; 7|]

printfn "arr.[1..3]: %A" slice1
printfn "arr.[..4]: %A" slice2
printfn "arr.[5..]: %A" slice3
printfn "arr.[2..7]: %A" slice4

// Slice เป็น copy (ไม่ใช่ reference เดิม)
let sliceCopy = arr.[1..3]
sliceCopy.[0] <- 100
printfn "Original unchanged: %A" arr.[1..3]  // ยังเป็น [|1; 2; 3|]
printfn "Slice copy: %A" sliceCopy           // [|100; 2; 3|]

// Array.sub - สร้าง sub-array
let sub = Array.sub arr 2 4  // เริ่มที่ index 2, จำนวน 4 ตัว
printfn "sub: %A" sub        // [|2; 3; 4; 5|]

// การ slice แบบ window
let windowSize = 3
let windows = 
    [| for i in 0..(arr.Length - windowSize) -> arr.[i..i + windowSize - 1] |]
printfn "Windows of size 3:"
windows |> Array.iter (fun w -> printfn "  %A" w)

// ตัวอย่างการใช้ slicing จริง: Moving average
let data = [| 2.0; 4.0; 6.0; 8.0; 10.0; 12.0; 14.0 |]
let movingAverage (data: float[]) (windowSize: int) =
    [| for i in 0..(data.Length - windowSize) ->
        let window = data.[i..i + windowSize - 1]
        Array.average window |]

let ma3 = movingAverage data 3
printfn "Moving average (size 3): %A" ma3
// Output: Moving average (size 3): [|4.0; 6.0; 8.0; 10.0; 12.0|]
```

---

## 21.6 Array.map, Array.filter, Array.fold

```fsharp
// ============ Array.map ============
let numbers = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]

// map - แปลงทุก element
let doubled = Array.map (fun x -> x * 2) numbers
printfn "map double: %A" doubled

// map กับ string
let words = [| "hello"; "world"; "foo"; "bar" |]
let upper = Array.map (fun s -> s.ToUpper()) words
let lengths = Array.map (fun (s: string) -> s.Length) words
printfn "upper: %A" upper
printfn "lengths: %A" lengths

// Array.mapi - map พร้อม index
let indexed = Array.mapi (fun i x -> sprintf "[%d]=%d" i x) numbers
printfn "mapi: %A" indexed

// Array.map2 - map สองอาร์เรย์พร้อมกัน
let a = [| 1; 2; 3; 4; 5 |]
let b = [| 10; 20; 30; 40; 50 |]
let sums = Array.map2 (fun x y -> x + y) a b
printfn "map2 sum: %A" sums
// Output: map2 sum: [|11; 22; 33; 44; 55|]

// ============ Array.filter ============
// filter - กรองตามเงื่อนไข
let evens = Array.filter (fun x -> x % 2 = 0) numbers
let odds = Array.filter (fun x -> x % 2 <> 0) numbers
printfn "evens: %A" evens
printfn "odds: %A" odds

// filter กับ string
let longWords = Array.filter (fun (s: string) -> s.Length > 3) words
printfn "longWords: %A" longWords

// Array.choose - filter + map ในครั้งเดียว
let data = [| -2; -1; 0; 1; 2; 3; 4; 5 |]
let positiveDoubled = Array.choose (fun x -> if x > 0 then Some (x * 2) else None) data
printfn "choose positive doubled: %A" positiveDoubled
// Output: choose positive doubled: [|2; 4; 6; 8; 10|]

// ============ Array.fold ============
// fold - สะสมค่า (left fold)
let sum = Array.fold (fun acc x -> acc + x) 0 numbers
printfn "fold sum: %d" sum   // 55

let product = Array.fold (fun acc x -> acc * x) 1 [| 1; 2; 3; 4; 5 |]
printfn "fold product: %d" product   // 120

// fold string concatenation
let sentence = Array.fold (fun acc w -> acc + " " + w) "" words
printfn "fold concat: '%s'" (sentence.Trim())

// Array.foldBack - fold จากขวา
let rightFold = Array.foldBack (fun x acc -> x :: acc) [| 1; 2; 3 |] []
printfn "foldBack: %A" rightFold   // [1; 2; 3]

// Array.reduce - เหมือน fold แต่ใช้ element แรกเป็น accumulator
let maxVal = Array.reduce max numbers
let minVal = Array.reduce min numbers
printfn "reduce max: %d" maxVal   // 10
printfn "reduce min: %d" minVal   // 1

// Array.scan - เหมือน fold แต่ return ค่าทุก step
let cumulativeSum = Array.scan (fun acc x -> acc + x) 0 numbers
printfn "scan (cumulative sum): %A" cumulativeSum
// Output: scan (cumulative sum): [|0; 1; 3; 6; 10; 15; 21; 28; 36; 45; 55|]

// ============ combinations ============
// Pipeline: filter -> map -> fold
let result = 
    numbers
    |> Array.filter (fun x -> x % 2 = 0)  // [|2;4;6;8;10|]
    |> Array.map (fun x -> x * x)           // [|4;16;36;64;100|]
    |> Array.fold (fun acc x -> acc + x) 0  // 220
printfn "filter -> map -> fold: %d" result
```

---

## 21.7 Array.sort, Array.sortInPlace

```fsharp
// ============ Array.sort ============
// sort - สร้าง array ใหม่ที่ sorted
let unsorted = [| 5; 3; 8; 1; 9; 2; 7; 4; 6 |]
let sorted = Array.sort unsorted
printfn "sort: %A" sorted
printfn "original unchanged: %A" unsorted

// sort string
let words = [| "banana"; "apple"; "cherry"; "date" |]
let sortedWords = Array.sort words
printfn "sorted words: %A" sortedWords

// ============ Array.sortInPlace ============
// sortInPlace - sort ใน array เดิม (mutate)
let arr = [| 5; 3; 8; 1; 9; 2; 7; 4; 6 |]
Array.sortInPlace arr
printfn "sortInPlace: %A" arr

// ============ Array.sortBy ============
// sortBy - sort โดยใช้ key function
type Person = { Name: string; Age: int }
let people = [| 
    { Name = "Charlie"; Age = 30 }
    { Name = "Alice"; Age = 25 }
    { Name = "Bob"; Age = 35 }
    { Name = "Diana"; Age = 28 }
|]

let sortedByAge = Array.sortBy (fun p -> p.Age) people
let sortedByName = Array.sortBy (fun p -> p.Name) people
printfn "sorted by age: %A" (Array.map (fun p -> p.Name) sortedByAge)
printfn "sorted by name: %A" (Array.map (fun p -> p.Name) sortedByName)

// Array.sortByDescending - sort จากมากไปน้อย
let sortedByAgeDesc = Array.sortByDescending (fun p -> p.Age) people
printfn "sorted by age desc: %A" (Array.map (fun p -> p.Name) sortedByAgeDesc)

// ============ Array.sortWith ============
// sortWith - sort โดยใช้ custom comparer
let sortedCustom = 
    Array.sortWith (fun p1 p2 -> 
        let cmp = compare p1.Age p2.Age
        if cmp <> 0 then cmp
        else compare p1.Name p2.Name
    ) people
printfn "sortWith: %A" (Array.map (fun p -> p.Name) sortedCustom)

// ============ Array.sortInPlaceBy ============
let arr2 = [| 5; -3; 8; -1; 9; -2 |]
Array.sortInPlaceBy abs arr2   // sort by absolute value
printfn "sortInPlaceBy abs: %A" arr2

// Array.sortInPlaceWith
let arr3 = [| "banana"; "APPLE"; "Cherry" |]
Array.sortInPlaceWith (fun a b -> String.Compare(a, b, true)) arr3
printfn "sortInPlaceWith case-insensitive: %A" arr3

// ============ Reverse ============
let ascending = [| 1; 2; 3; 4; 5 |]
let descending = Array.rev ascending
printfn "rev: %A" descending
// Output: rev: [|5; 4; 3; 2; 1|]

// Array.sortDescending
let sortedDesc = Array.sortDescending [| 5; 3; 8; 1; 9 |]
printfn "sortDescending: %A" sortedDesc
```

---

## 21.8 Array.copy, Array.append, Array.concat

```fsharp
// ============ Array.copy ============
// copy - สร้าง shallow copy
let original = [| 1; 2; 3; 4; 5 |]
let copy = Array.copy original

// แก้ไข copy ไม่กระทบ original
copy.[0] <- 100
printfn "original: %A" original   // [|1; 2; 3; 4; 5|]
printfn "copy: %A" copy           // [|100; 2; 3; 4; 5|]

// Shallow copy กับ reference types
type Box = { mutable Value: int }
let boxes = [| { Value = 1 }; { Value = 2 }; { Value = 3 } |]
let boxesCopy = Array.copy boxes

// การเปลี่ยน reference กัน ไม่กระทบ
boxesCopy.[0] <- { Value = 100 }
printfn "boxes.[0]: %A" boxes.[0]       // { Value = 1 }

// แต่การแก้ไข field ของ mutable record จะกระทบ
boxesCopy.[1].Value <- 200
printfn "boxes.[1].Value: %d" boxes.[1].Value   // 200! (shared reference)

// ============ Array.append ============
// append - รวมสอง array
let a = [| 1; 2; 3 |]
let b = [| 4; 5; 6 |]
let combined = Array.append a b
printfn "append: %A" combined
// Output: append: [|1; 2; 3; 4; 5; 6|]

// append กับ empty
let withEmpty = Array.append a [||]
printfn "append empty: %A" withEmpty   // [|1; 2; 3|]

// ============ Array.concat ============
// concat - รวมหลาย array
let arrays = [| [| 1; 2 |]; [| 3; 4 |]; [| 5; 6 |] |]
let flat = Array.concat arrays
printfn "concat: %A" flat
// Output: concat: [|1; 2; 3; 4; 5; 6|]

// concat sequence of arrays
let manyArrays = [| [| 1 |]; [| 2; 3 |]; [| 4; 5; 6 |]; [| 7; 8; 9; 10 |] |]
let allTogether = Array.concat manyArrays
printfn "concat many: %A" allTogether

// ============ Array.collect ============
// collect - flatMap (map then flatten)
let nested = [| 1; 2; 3; 4; 5 |]
let expanded = Array.collect (fun x -> [| x; x * 10 |]) nested
printfn "collect: %A" expanded
// Output: collect: [|1; 10; 2; 20; 3; 30; 4; 40; 5; 50|]

// ============ Array.zip / Array.unzip ============
let xs = [| 1; 2; 3; 4; 5 |]
let ys = [| "a"; "b"; "c"; "d"; "e" |]
let zipped = Array.zip xs ys
printfn "zip: %A" zipped
// Output: zip: [|(1, "a"); (2, "b"); ...|]

let (unzippedXs, unzippedYs) = Array.unzip zipped
printfn "unzip xs: %A" unzippedXs
printfn "unzip ys: %A" unzippedYs
```

---

## 21.9 2D Arrays (อาร์เรย์ 2 มิติ)

```fsharp
open System

// ============ array2D ============
// สร้าง 2D array จาก sequence of sequences
let matrix = array2D [ [1; 2; 3]; [4; 5; 6]; [7; 8; 9] ]

// เข้าถึงด้วย [i, j]
printfn "matrix.[0,0]: %d" matrix.[0, 0]   // 1
printfn "matrix.[1,2]: %d" matrix.[1, 2]   // 6
printfn "matrix.[2,2]: %d" matrix.[2, 2]   // 9

// Dimensions
printfn "rows: %d" (Array2D.length1 matrix)   // 3
printfn "cols: %d" (Array2D.length2 matrix)   // 3

// ============ Array2D module ============
// Array2D.create
let zeroMatrix = Array2D.create 3 4 0
printfn "zeroMatrix dimensions: %dx%d" (Array2D.length1 zeroMatrix) (Array2D.length2 zeroMatrix)

// Array2D.init
let multiplicationTable = Array2D.init 5 5 (fun i j -> (i + 1) * (j + 1))
printfn "Multiplication table:"
for i in 0..4 do
    for j in 0..4 do
        printf "%4d" multiplicationTable.[i, j]
    printfn ""

// Array2D.map
let doubledMatrix = Array2D.map (fun x -> x * 2) multiplicationTable
printfn "Doubled [0,0]: %d" doubledMatrix.[0, 0]   // 2

// Array2D.mapi
let labeledMatrix = Array2D.mapi (fun i j x -> sprintf "(%d,%d)=%d" i j x) matrix
printfn "labeled [0,0]: %s" labeledMatrix.[0, 0]

// Array2D.iter
printfn "All elements:"
Array2D.iter (fun x -> printf "%d " x) matrix
printfn ""

// Array2D.iteri
Array2D.iteri (fun i j x -> 
    if i = j then printfn "Diagonal [%d,%d] = %d" i j x
) matrix

// Array2D mutation
let mutableMatrix = Array2D.create 3 3 0
for i in 0..2 do
    for j in 0..2 do
        mutableMatrix.[i, j] <- i * 3 + j
printfn "Mutable matrix [1,2]: %d" mutableMatrix.[1, 2]   // 5

// ============ 2D Slicing ============
// ตัดแถว
let row0 = matrix.[0, *]     // row 0: [|1; 2; 3|]
let col1 = matrix.[*, 1]     // col 1: [|2; 5; 8|]
printfn "row 0: %A" row0
printfn "col 1: %A" col1

// ============ เมทริกซ์คูณ ============
let matMul (a: int[,]) (b: int[,]) =
    let rows = Array2D.length1 a
    let cols = Array2D.length2 b
    let inner = Array2D.length2 a
    let result = Array2D.create rows cols 0
    for i in 0..rows - 1 do
        for j in 0..cols - 1 do
            let mutable sum = 0
            for k in 0..inner - 1 do
                sum <- sum + a.[i, k] * b.[k, j]
            result.[i, j] <- sum
    result

let mat1 = array2D [ [1; 2]; [3; 4] ]
let mat2 = array2D [ [5; 6]; [7; 8] ]
let product = matMul mat1 mat2

printfn "Matrix multiplication:"
for i in 0..1 do
    for j in 0..1 do
        printf "%4d" product.[i, j]
    printfn ""
// [19, 22]
// [43, 50]
```

---

## 21.10 Jagged Arrays (อาร์เรย์ขรุขระ)

```fsharp
// Jagged array = array of arrays (แต่ละแถวมีขนาดต่างกัน)
// ต่างจาก 2D array ที่ทุกแถวมีขนาดเท่ากัน

// สร้าง jagged array
let jagged : int[][] = [|
    [| 1 |]
    [| 1; 2 |]
    [| 1; 2; 3 |]
    [| 1; 2; 3; 4 |]
    [| 1; 2; 3; 4; 5 |]
|]

// เข้าถึง
printfn "jagged.[0]: %A" jagged.[0]         // [|1|]
printfn "jagged.[2]: %A" jagged.[2]         // [|1; 2; 3|]
printfn "jagged.[2].[1]: %d" jagged.[2].[1] // 2

// iterate
for row in jagged do
    printfn "Row length %d: %A" row.Length row

// ตัวอย่าง Pascal's Triangle
let pascalTriangle (n: int) =
    let triangle = Array.init n (fun i -> Array.create (i + 1) 1)
    for i in 2..n - 1 do
        for j in 1..i - 1 do
            triangle.[i].[j] <- triangle.[i-1].[j-1] + triangle.[i-1].[j]
    triangle

let pascal = pascalTriangle 8
printfn "\nPascal's Triangle:"
for row in pascal do
    let padding = String.replicate (8 - row.Length) "  "
    printf "%s" padding
    for x in row do
        printf "%4d" x
    printfn ""

// ตัวอย่าง adjacency list (กราฟ)
let graph : int[][] = [|
    [| 1; 2 |]      // node 0 เชื่อมกับ 1, 2
    [| 0; 3 |]      // node 1 เชื่อมกับ 0, 3
    [| 0; 3; 4 |]   // node 2 เชื่อมกับ 0, 3, 4
    [| 1; 2; 4 |]   // node 3 เชื่อมกับ 1, 2, 4
    [| 2; 3 |]      // node 4 เชื่อมกับ 2, 3
|]

printfn "\nGraph adjacency list:"
Array.iteri (fun i neighbors ->
    printfn "Node %d -> %A" i neighbors
) graph
```

---

## 21.11 Array.blit (การคัดลอก)

```fsharp
// Array.blit - คัดลอกส่วนหนึ่งของ array ไปยังอีก array
// Array.blit source sourceIndex destination destIndex count

let source = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]
let dest = Array.create 10 0

// คัดลอก 5 ตัวจาก source index 2 ไปที่ dest index 3
Array.blit source 2 dest 3 5
printfn "After blit: %A" dest
// Output: After blit: [|0; 0; 0; 3; 4; 5; 6; 7; 0; 0|]

// การใช้ blit สำหรับ circular buffer
let circularInsert (buffer: int[]) (pos: ref<int>) (value: int) =
    buffer.[!pos % buffer.Length] <- value
    pos := !pos + 1

let buf = Array.create 5 0
let pos = ref 0
for i in 1..8 do
    circularInsert buf pos i
printfn "Circular buffer: %A" buf   // ค่าล่าสุด 5 ค่า

// ตัวอย่าง: sliding window ด้วย blit
let slideWindow (data: int[]) (windowSize: int) =
    let result = Array.create data.Length 0.0
    let window = Array.create windowSize 0
    
    // เติม window แรก
    Array.blit data 0 window 0 windowSize
    
    for i in 0..(data.Length - windowSize - 1) do
        result.[i] <- float (Array.sum window) / float windowSize
        // shift window
        Array.blit window 1 window 0 (windowSize - 1)
        window.[windowSize - 1] <- data.[i + windowSize]
    
    result.[data.Length - windowSize] <- 
        float (Array.sum window) / float windowSize
    result

let testData = [| 1; 3; 5; 7; 9; 11; 13; 15 |]
let ma = slideWindow testData 3
printfn "Moving avg (blit): %A" ma
```

---

## 21.12 Array.indexed

```fsharp
// Array.indexed - เพิ่ม index ให้กับแต่ละ element
let fruits = [| "apple"; "banana"; "cherry"; "date" |]
let indexed = Array.indexed fruits
printfn "indexed: %A" indexed
// Output: indexed: [|(0, "apple"); (1, "banana"); (2, "cherry"); (3, "date")|]

// ใช้กับ pattern matching
for (i, fruit) in Array.indexed fruits do
    printfn "Position %d: %s" i fruit

// ใช้ค้นหา index ของ element ที่ตรงเงื่อนไข
let fruits2 = [| "apple"; "banana"; "cherry"; "date"; "elderberry" |]
let longFruits = 
    fruits2
    |> Array.indexed
    |> Array.filter (fun (_, f) -> f.Length > 5)
    |> Array.map (fun (i, f) -> sprintf "Index %d: %s" i f)

printfn "Long fruits:"
Array.iter (printfn "  %s") longFruits

// ตัวอย่าง: หา index ทั้งหมดที่ตรงเงื่อนไข
let findAllIndices (predicate: 'a -> bool) (arr: 'a[]) =
    arr
    |> Array.indexed
    |> Array.choose (fun (i, x) -> if predicate x then Some i else None)

let data = [| 1; 5; 2; 8; 3; 9; 4; 7; 6 |]
let greaterThan5Indices = findAllIndices (fun x -> x > 5) data
printfn "Indices > 5: %A" greaterThan5Indices
// Output: Indices > 5: [|1; 3; 5; 7|]

// Array.tryFindIndex, Array.findIndex
let firstEvenIndex = Array.tryFindIndex (fun x -> x % 2 = 0) data
printfn "First even index: %A" firstEvenIndex   // Some 2

let lastEvenIndex = Array.tryFindIndexBack (fun x -> x % 2 = 0) data
printfn "Last even index: %A" lastEvenIndex    // Some 8

// Array.findIndex (throws if not found)
let firstEvenIdx = Array.findIndex (fun x -> x % 2 = 0) data
printfn "findIndex even: %d" firstEvenIdx
```

---

## 21.13 ArraySegment

```fsharp
open System

// ArraySegment - แทนส่วนหนึ่งของ array โดยไม่ต้อง copy
let arr = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]

// สร้าง ArraySegment
let segment = ArraySegment<int>(arr, 2, 5)  // เริ่มที่ index 2, ความยาว 5

printfn "Segment Count: %d" segment.Count     // 5
printfn "Segment Offset: %d" segment.Offset   // 2
printfn "Segment Array Length: %d" segment.Array.Length  // 10

// เข้าถึง elements
printfn "Segment[0]: %d" segment.[0]   // 3
printfn "Segment[1]: %d" segment.[1]   // 4

// iterate
for x in segment do
    printf "%d " x
printfn ""   // 3 4 5 6 7

// แปลงเป็น array
let segArray = segment.ToArray()
printfn "Segment to array: %A" segArray

// การใช้งานจริง: buffer processing
let processChunk (segment: ArraySegment<byte>) =
    let mutable checksum = 0
    for b in segment do
        checksum <- checksum ^^^ int b
    checksum

let buffer = [| 1uy; 2uy; 3uy; 4uy; 5uy; 6uy; 7uy; 8uy |]
let chunk1 = ArraySegment<byte>(buffer, 0, 4)
let chunk2 = ArraySegment<byte>(buffer, 4, 4)
printfn "Chunk 1 XOR: %d" (processChunk chunk1)
printfn "Chunk 2 XOR: %d" (processChunk chunk2)

// Memory<T> และ Span<T> (สมัยใหม่กว่า)
// ใช้สำหรับ high-performance scenarios
let memory = Memory<int>(arr)
let span = memory.Span
printfn "Memory Length: %d" memory.Length

// Slice memory
let slicedMemory = memory.Slice(2, 5)
printfn "Sliced memory Length: %d" slicedMemory.Length
```

---

## 21.14 Performance Comparison: Array vs List

```fsharp
open System
open System.Diagnostics

// ============ การเปรียบเทียบประสิทธิภาพ ============

// Random access: Array O(1) vs List O(n)
let n = 100_000
let testArray = Array.init n id
let testList = List.init n id

let sw = Stopwatch()

// Array random access
sw.Restart()
let mutable sum1 = 0
for _ in 1..1000 do
    sum1 <- sum1 + testArray.[n / 2]
sw.Stop()
printfn "Array random access: %d ms" sw.ElapsedMilliseconds

// List random access
sw.Restart()
let mutable sum2 = 0
for _ in 1..1000 do
    sum2 <- sum2 + testList.[n / 2]
sw.Stop()
printfn "List random access: %d ms" sw.ElapsedMilliseconds

// Sequential access (iteration)
sw.Restart()
let mutable total1 = 0
for x in testArray do
    total1 <- total1 + x
sw.Stop()
printfn "Array sequential: %d ms" sw.ElapsedMilliseconds

sw.Restart()
let mutable total2 = 0
for x in testList do
    total2 <- total2 + x
sw.Stop()
printfn "List sequential: %d ms" sw.ElapsedMilliseconds

// Prepend: Array O(n) copy vs List O(1)
sw.Restart()
let mutable arr2 = testArray
for i in 1..100 do
    arr2 <- Array.append [| i |] arr2
sw.Stop()
printfn "Array prepend x100: %d ms" sw.ElapsedMilliseconds

sw.Restart()
let mutable lst = testList
for i in 1..100 do
    lst <- i :: lst
sw.Stop()
printfn "List prepend x100: %d ms" sw.ElapsedMilliseconds

// Memory usage comparison (approximate)
printfn "\nMemory layout:"
printfn "Array: contiguous memory, cache-friendly"
printfn "List: linked nodes, pointer per element"
```

---

## 21.15 When to Use Arrays vs Lists

```fsharp
// ============ เมื่อไหรควรใช้ Array ============

// 1. เมื่อต้องการ random access บ่อย
let lookupTable = [| 0; 1; 1; 2; 3; 5; 8; 13; 21; 34 |]  // Fibonacci
let fib10 = lookupTable.[9]   // O(1) access
printfn "Fib(10) = %d" fib10

// 2. เมื่อต้องการ mutation
let scores = Array.create 5 0
scores.[0] <- 100
scores.[1] <- 95
scores.[2] <- 87
scores.[3] <- 92
scores.[4] <- 78
printfn "Scores: %A" scores

// 3. เมื่อทำงานกับ data จำนวนมากและต้องการ performance
let bigData = Array.init 1_000_000 (fun i -> float i)
let sum = Array.sum bigData   // ประมวลผลได้เร็วมาก
printfn "Sum of 1M numbers: %.0f" sum

// 4. เมื่อต้องการ interop กับ .NET libraries
// byte arrays สำหรับ I/O
let bytes = [| 72uy; 101uy; 108uy; 108uy; 111uy |]
let text = System.Text.Encoding.ASCII.GetString(bytes)
printfn "Decoded: %s" text

// 5. เมื่อต้องการ 2D/multi-dimensional data
let grid = Array2D.create 10 10 0
Array2D.iteri (fun i j _ -> grid.[i, j] <- i * 10 + j) grid

// ============ เมื่อไหรควรใช้ List ============

// 1. เมื่อ prepend บ่อย (functional programming pattern)
let buildList () =
    [1..100]
    |> List.map (fun x -> x * x)
    |> List.filter (fun x -> x % 2 = 0)

// 2. เมื่อ pattern match กับโครงสร้าง
let rec sumList = function
    | [] -> 0
    | x :: xs -> x + sumList xs

printfn "sumList: %d" (sumList [1..10])

// 3. เมื่อต้องการ immutability
let original = [1; 2; 3; 4; 5]
let added = 0 :: original  // original ไม่เปลี่ยน
printfn "original: %A" original
printfn "added: %A" added

// ============ สรุปแนวทาง ============
(*
    ใช้ Array เมื่อ:
    - ต้องการ O(1) random access
    - ต้องการ mutation
    - ทำงานกับ .NET APIs ที่ต้องการ array
    - ทำงานกับ 2D/multi-dimensional data
    - ต้องการ performance สูงสุด (cache locality)
    
    ใช้ List เมื่อ:
    - ต้องการ immutability
    - ต้อง prepend บ่อย (cons operation)
    - pattern matching กับโครงสร้าง
    - functional programming style
    - ขนาดข้อมูลน้อยและ random access ไม่จำเป็น
    
    ใช้ Seq เมื่อ:
    - ต้องการ lazy evaluation
    - ข้อมูลมาจาก stream/infinite source
    - ต้องการ pipeline processing โดยไม่สร้าง intermediate collections
*)
```

---

## 21.16 Advanced Array Operations

```fsharp
// ============ Array.partition ============
let nums = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]
let (evens, odds) = Array.partition (fun x -> x % 2 = 0) nums
printfn "evens: %A" evens
printfn "odds: %A" odds

// ============ Array.groupBy ============
let words = [| "apple"; "ant"; "bear"; "bee"; "cat"; "cow"; "dog" |]
let grouped = Array.groupBy (fun (s: string) -> s.[0]) words
for (key, values) in grouped do
    printfn "Letter '%c': %A" key values

// ============ Array.distinct / Array.distinctBy ============
let dupes = [| 1; 2; 3; 2; 4; 1; 5; 3; 6 |]
let unique = Array.distinct dupes
printfn "distinct: %A" unique

type Item = { Id: int; Name: string }
let items = [|
    { Id = 1; Name = "A" }
    { Id = 2; Name = "B" }
    { Id = 1; Name = "A duplicate" }  // dup Id
    { Id = 3; Name = "C" }
|]
let uniqueById = Array.distinctBy (fun item -> item.Id) items
printfn "distinctBy Id: %A" (Array.map (fun i -> i.Id) uniqueById)

// ============ Array.pairwise ============
let consecutive = [| 1; 3; 6; 10; 15; 21 |]
let pairs = Array.pairwise consecutive
printfn "pairwise: %A" pairs
// Output: pairwise: [|(1, 3); (3, 6); (6, 10); (10, 15); (15, 21)|]

// ใช้คำนวณ differences
let diffs = Array.map (fun (a, b) -> b - a) (Array.pairwise consecutive)
printfn "differences: %A" diffs
// Output: differences: [|2; 3; 4; 5; 6|]

// ============ Array.windowed ============
let nums2 = [| 1; 2; 3; 4; 5; 6; 7 |]
let windows = Array.windowed 3 nums2
printfn "windowed 3: %A" windows
// Output: windowed 3: [|[|1;2;3|]; [|2;3;4|]; [|3;4;5|]; [|4;5;6|]; [|5;6;7|]|]

// ============ Array.chunkBySize ============
let allNums = [| 1..15 |]
let chunks = Array.chunkBySize 4 allNums
printfn "chunkBySize 4: %A" chunks

// ============ Array.splitAt ============
let (left, right) = Array.splitAt 5 allNums
printfn "splitAt 5 - left: %A" left
printfn "splitAt 5 - right: %A" right

// ============ Array.take / Array.skip ============
let taken = Array.take 5 allNums
let skipped = Array.skip 10 allNums
printfn "take 5: %A" taken
printfn "skip 10: %A" skipped

// Array.takeWhile / Array.skipWhile
let takeWhile = Array.takeWhile (fun x -> x < 7) allNums
let skipWhile = Array.skipWhile (fun x -> x < 7) allNums
printfn "takeWhile < 7: %A" takeWhile
printfn "skipWhile < 7: %A" skipWhile

// ============ Array.forall / Array.exists ============
let allPositive = Array.forall (fun x -> x > 0) [| 1; 2; 3; 4; 5 |]
let anyNegative = Array.exists (fun x -> x < 0) [| 1; -2; 3; 4; 5 |]
printfn "allPositive: %b" allPositive   // true
printfn "anyNegative: %b" anyNegative   // true

// Array.contains
let hasFive = Array.contains 5 [| 1; 2; 3; 4; 5 |]
printfn "contains 5: %b" hasFive   // true

// ============ Array.sum, Array.average, Array.min, Array.max ============
let floats = [| 1.0; 2.5; 3.7; 4.2; 5.1 |]
printfn "sum: %f" (Array.sum floats)
printfn "average: %f" (Array.average floats)
printfn "min: %f" (Array.min floats)
printfn "max: %f" (Array.max floats)

// Array.sumBy, Array.averageBy, Array.minBy, Array.maxBy
type Student = { Name: string; Score: float }
let students = [|
    { Name = "Alice"; Score = 90.0 }
    { Name = "Bob"; Score = 75.5 }
    { Name = "Charlie"; Score = 88.0 }
|]
let avgScore = Array.averageBy (fun s -> s.Score) students
let topStudent = Array.maxBy (fun s -> s.Score) students
printfn "avgScore: %f" avgScore
printfn "topStudent: %s" topStudent.Name

// ============ Array.truncate ============
let truncated = Array.truncate 5 [| 1..100 |]
printfn "truncate 5: %A" truncated
```

---

## 21.17 Practical Examples (ตัวอย่างการใช้งานจริง)

```fsharp
open System

// ============ ตัวอย่าง 1: Image Processing (Gray scale) ============
type Pixel = { R: byte; G: byte; B: byte }

let toGrayscale (pixel: Pixel) =
    let gray = byte (0.299 * float pixel.R + 0.587 * float pixel.G + 0.114 * float pixel.B)
    { R = gray; G = gray; B = gray }

// สร้าง fake image
let image = Array2D.init 4 4 (fun i j ->
    { R = byte (i * 64); G = byte (j * 64); B = byte ((i + j) * 32) }
)

let grayImage = Array2D.map toGrayscale image
printfn "Gray pixel [0,0]: %A" grayImage.[0, 0]

// ============ ตัวอย่าง 2: Histogram ============
let histogram (data: int[]) (bins: int) =
    let minVal = Array.min data
    let maxVal = Array.max data
    let binSize = float (maxVal - minVal + 1) / float bins
    let hist = Array.create bins 0
    
    for x in data do
        let binIdx = min (bins - 1) (int (float (x - minVal) / binSize))
        hist.[binIdx] <- hist.[binIdx] + 1
    
    hist

let testData = [| for _ in 1..1000 -> Random().Next(1, 101) |]
let hist = histogram testData 10
printfn "Histogram (10 bins): %A" hist

// ============ ตัวอย่าง 3: String to character frequency ============
let charFrequency (text: string) =
    let counts = Array.create 256 0
    for c in text do
        counts.[int c] <- counts.[int c] + 1
    counts
    |> Array.indexed
    |> Array.filter (fun (_, count) -> count > 0)
    |> Array.map (fun (charCode, count) -> (char charCode, count))
    |> Array.sortByDescending snd

let freq = charFrequency "hello world"
printfn "Character frequencies:"
freq |> Array.iter (fun (c, n) -> printfn "  '%c': %d" c n)

// ============ ตัวอย่าง 4: Stack implementation using Array ============
type ArrayStack<'T>(capacity: int) =
    let data = Array.create capacity Unchecked.defaultof<'T>
    let mutable top = -1
    
    member _.Push(value: 'T) =
        if top >= capacity - 1 then
            failwith "Stack overflow"
        top <- top + 1
        data.[top] <- value
    
    member _.Pop() =
        if top < 0 then failwith "Stack underflow"
        let value = data.[top]
        top <- top - 1
        value
    
    member _.Peek() =
        if top < 0 then failwith "Stack empty"
        data.[top]
    
    member _.IsEmpty = top < 0
    member _.Count = top + 1

let stack = ArrayStack<int>(10)
stack.Push(1)
stack.Push(2)
stack.Push(3)
printfn "Peek: %d" (stack.Peek())   // 3
printfn "Pop: %d" (stack.Pop())     // 3
printfn "Pop: %d" (stack.Pop())     // 2
printfn "Count: %d" stack.Count     // 1

// ============ ตัวอย่าง 5: Sieve of Eratosthenes ============
let sieve (limit: int) =
    let isPrime = Array.create (limit + 1) true
    isPrime.[0] <- false
    isPrime.[1] <- false
    
    let mutable i = 2
    while i * i <= limit do
        if isPrime.[i] then
            let mutable j = i * i
            while j <= limit do
                isPrime.[j] <- false
                j <- j + i
        i <- i + 1
    
    [| for i in 2..limit do if isPrime.[i] then yield i |]

let primes = sieve 100
printfn "Primes up to 100: %A" primes

// ============ ตัวอย่าง 6: Merge Sort ============
let rec mergeSort (arr: int[]) =
    if arr.Length <= 1 then arr
    else
        let mid = arr.Length / 2
        let left = mergeSort arr.[..mid - 1]
        let right = mergeSort arr.[mid..]
        
        let merge (a: int[]) (b: int[]) =
            let result = Array.create (a.Length + b.Length) 0
            let mutable i, j, k = 0, 0, 0
            while i < a.Length && j < b.Length do
                if a.[i] <= b.[j] then
                    result.[k] <- a.[i]; i <- i + 1
                else
                    result.[k] <- b.[j]; j <- j + 1
                k <- k + 1
            while i < a.Length do
                result.[k] <- a.[i]; i <- i + 1; k <- k + 1
            while j < b.Length do
                result.[k] <- b.[j]; j <- j + 1; k <- k + 1
            result
        
        merge left right

let unsorted = [| 38; 27; 43; 3; 9; 82; 10 |]
let sortedArr = mergeSort unsorted
printfn "Merge sort: %A" sortedArr
// Output: Merge sort: [|3; 9; 10; 27; 38; 43; 82|]
```

---

## สรุป (Summary)

```
อาร์เรย์ใน F#:
- ใช้ [| |] สำหรับ array literals
- เป็น mutable (แก้ไขได้)
- เข้าถึงด้วย index O(1)
- มี Array module ที่ครบครัน

ฟังก์ชันสำคัญ:
- Array.create, Array.init, Array.zeroCreate - การสร้าง
- Array.map, Array.filter, Array.fold - functional operations
- Array.sort, Array.sortInPlace - การเรียงลำดับ
- Array.copy, Array.append, Array.concat - การรวม/คัดลอก
- Array2D module - สำหรับ 2D arrays
- Array.blit - การคัดลอกแบบ efficient

เมื่อไหรควรใช้ Array:
- Random access บ่อย
- ต้องการ mutation
- Interop กับ .NET
- Performance สำคัญ
```

---

*จบ Part 21 - อาร์เรย์ (Arrays)*
