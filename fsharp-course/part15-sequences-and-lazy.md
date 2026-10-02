# Part 15 - ลำดับและการประมวลผลแบบขี้เกียจ (Sequences and Lazy Evaluation)

## บทนำ (Introduction)

**Sequences** ใน F# คือ collection ที่ประเมินค่าแบบ lazy (เมื่อต้องการใช้จริง) ซึ่งต่างจาก List ที่ประเมินทันที (eager)

**Lazy Evaluation** คือการเลื่อนการคำนวณออกไปจนกว่าจะต้องการผลลัพธ์จริงๆ

---

## 1. seq { } Computation Expression

```fsharp
// seq { } ใช้สำหรับสร้าง sequence
let mySeq = seq {
    yield 1
    yield 2
    yield 3
}

// หรือสั้นกว่า
let mySeq2 = seq { 1..5 }

// วนลูปใน seq
let evenNumbers = seq {
    for i in 0..2..20 do
        yield i
}

printfn "mySeq: %A" (Seq.toList mySeq)
printfn "mySeq2: %A" (Seq.toList mySeq2)
printfn "evenNumbers: %A" (Seq.toList evenNumbers)
```

```fsharp
// seq ด้วย if/else
let classifiedNumbers = seq {
    for i in 1..10 do
        if i % 2 = 0 then
            yield sprintf "%d is even" i
        else
            yield sprintf "%d is odd" i
}

classifiedNumbers |> Seq.iter (printfn "%s")
```

```fsharp
// seq ซ้อนกัน
let matrix = seq {
    for i in 1..3 do
        for j in 1..3 do
            yield (i, j, i * j)
}

matrix |> Seq.iter (fun (i, j, p) -> printfn "%d x %d = %d" i j p)
```

```fsharp
// seq กับ mutable state (สามารถใช้ได้ใน seq)
let countdown = seq {
    let mutable n = 10
    while n > 0 do
        yield n
        n <- n - 1
    yield 0
    yield! seq { "Blast off!" |> ignore; () |> ignore }
}

countdown |> Seq.iter (printfn "%d")
```

---

## 2. Infinite Sequences

```fsharp
// Infinite sequences ทำได้เพราะ seq เป็น lazy!

// ตัวเลขธรรมชาติ (ไม่มีสิ้นสุด)
let naturalNumbers = seq {
    let mutable n = 0
    while true do
        yield n
        n <- n + 1
}

// ดึง 10 ตัวแรก
let first10 = naturalNumbers |> Seq.take 10 |> Seq.toList
printfn "First 10: %A" first10
```

```fsharp
// Infinite sequence ด้วย Seq.initInfinite
let naturals = Seq.initInfinite id           // 0, 1, 2, 3, ...
let positives = Seq.initInfinite (fun n -> n + 1)  // 1, 2, 3, 4, ...
let evens = Seq.initInfinite (fun n -> n * 2)       // 0, 2, 4, 6, ...
let odds = Seq.initInfinite (fun n -> n * 2 + 1)    // 1, 3, 5, 7, ...

printfn "First 5 naturals: %A" (naturals |> Seq.take 5 |> Seq.toList)
printfn "First 5 positives: %A" (positives |> Seq.take 5 |> Seq.toList)
printfn "First 5 evens: %A" (evens |> Seq.take 5 |> Seq.toList)
printfn "First 5 odds: %A" (odds |> Seq.take 5 |> Seq.toList)
```

```fsharp
// Infinite powers of 2
let powersOf2 = Seq.unfold (fun n -> Some (n, n * 2)) 1

let first8 = powersOf2 |> Seq.take 8 |> Seq.toList
printfn "Powers of 2: %A" first8  // [1; 2; 4; 8; 16; 32; 64; 128]
```

---

## 3. Seq.initInfinite

```fsharp
// Seq.initInfinite: สร้าง infinite sequence จาก function ที่รับ index
// signature: (int -> 'a) -> seq<'a>

let squares = Seq.initInfinite (fun n -> n * n)
let cubes = Seq.initInfinite (fun n -> n * n * n)
let factorials = Seq.initInfinite (fun n ->
    [1..n] |> List.fold (*) 1)

printfn "First 6 squares: %A" (squares |> Seq.take 6 |> Seq.toList)
printfn "First 6 cubes: %A" (cubes |> Seq.take 6 |> Seq.toList)
printfn "First 7 factorials: %A" (factorials |> Seq.take 7 |> Seq.toList)
```

```fsharp
// Fibonacci ด้วย Seq.initInfinite
// ต้องใช้ memoization หรือ unfold สำหรับ fibonacci ที่มีประสิทธิภาพ

let fibonacci = 
    Seq.unfold (fun (a, b) -> Some (a, (b, a + b))) (0, 1)

let first15 = fibonacci |> Seq.take 15 |> Seq.toList
printfn "Fibonacci: %A" first15
// [0; 1; 1; 2; 3; 5; 8; 13; 21; 34; 55; 89; 144; 233; 377]
```

---

## 4. Seq.take, Seq.skip

```fsharp
// Seq.take: ดึง n elements แรก
let numbers = Seq.initInfinite id

let first5 = numbers |> Seq.take 5 |> Seq.toList
let next5 = numbers |> Seq.skip 5 |> Seq.take 5 |> Seq.toList

printfn "first5: %A" first5  // [0; 1; 2; 3; 4]
printfn "next5: %A" next5    // [5; 6; 7; 8; 9]
```

```fsharp
// Seq.takeWhile: ดึง elements จนกว่าเงื่อนไขจะเป็น false
let numbers = Seq.initInfinite id

let lessThan10 = numbers |> Seq.takeWhile (fun x -> x < 10) |> Seq.toList
printfn "lessThan10: %A" lessThan10  // [0; 1; 2; 3; 4; 5; 6; 7; 8; 9]
```

```fsharp
// Seq.skipWhile: ข้าม elements จนกว่าเงื่อนไขจะเป็น false
let numbers = seq { 1..10 }

let afterFirstEvens = numbers |> Seq.skipWhile (fun x -> x % 2 <> 0) |> Seq.toList
printfn "afterFirstEvens: %A" afterFirstEvens  // [2; 3; 4; 5; 6; 7; 8; 9; 10]
```

```fsharp
// ใช้ take และ skip ร่วมกัน: pagination

let allItems = Seq.initInfinite (fun n -> sprintf "Item %d" n)

let getPage pageSize pageNum =
    allItems
    |> Seq.skip (pageSize * pageNum)
    |> Seq.take pageSize
    |> Seq.toList

let page0 = getPage 5 0
let page1 = getPage 5 1
let page2 = getPage 5 2

printfn "Page 0: %A" page0
printfn "Page 1: %A" page1
printfn "Page 2: %A" page2
```

---

## 5. Seq.map, Seq.filter

```fsharp
// Seq.map และ Seq.filter ทำงานแบบ lazy

// ไม่คำนวณทันที
let bigSeq =
    Seq.initInfinite id
    |> Seq.map (fun x -> x * x)      // ยังไม่คำนวณ
    |> Seq.filter (fun x -> x % 2 = 0)  // ยังไม่คำนวณ
    |> Seq.take 5                    // ยังไม่คำนวณ

// คำนวณเมื่อ iterate เท่านั้น
let result = bigSeq |> Seq.toList
printfn "result: %A" result  // [0; 4; 16; 36; 64]
```

```fsharp
// Composing seq operations
let primes =
    seq {
        let sieve = System.Collections.Generic.HashSet<int>()
        let mutable n = 2
        while true do
            if not (sieve.Contains(n)) then
                yield n
                let mutable multiple = n * n
                while multiple < 1000000 do
                    sieve.Add(multiple) |> ignore
                    multiple <- multiple + n
            n <- n + 1
    }

let first20Primes = primes |> Seq.take 20 |> Seq.toList
printfn "First 20 primes: %A" first20Primes
```

---

## 6. Seq.unfold for Generating Sequences

```fsharp
// Seq.unfold: สร้าง sequence จาก state
// signature: ('state -> ('output * 'state) option) -> 'state -> seq<'output>

// Range
let range start stop step =
    Seq.unfold (fun current ->
        if current > stop then None
        else Some (current, current + step)) start

let r1 = range 0 10 1 |> Seq.toList
let r2 = range 0 10 2 |> Seq.toList
let r3 = range 1 100 7 |> Seq.toList

printfn "range 0..10: %A" r1
printfn "range 0..10 step 2: %A" r2
printfn "range 1..100 step 7: %A" r3
```

```fsharp
// unfold สำหรับ Collatz sequence
let collatz n =
    Seq.unfold (fun x ->
        if x = 1 then None
        elif x % 2 = 0 then Some (x, x / 2)
        else Some (x, x * 3 + 1)) n

let collatz27 = collatz 27 |> Seq.toList
printfn "Collatz 27 length: %d" (List.length collatz27)
```

```fsharp
// unfold สำหรับ iterate กับ function
let iterate f x =
    Seq.unfold (fun state -> Some (state, f state)) x

let doubleSeq = iterate ((*) 2) 1 |> Seq.take 10 |> Seq.toList
let addPiSeq = iterate ((+) System.Math.PI) 0.0 |> Seq.take 5 |> Seq.toList

printfn "doubleSeq: %A" doubleSeq
printfn "addPiSeq: %A" addPiSeq
```

---

## 7. Lazy<'T> Type

```fsharp
// Lazy<'T>: เก็บการคำนวณที่ยังไม่ได้ทำ
// จะทำเมื่อเรียก .Value ครั้งแรก และ cache ผลลัพธ์

let expensiveComputation () =
    printfn "Computing..."
    System.Threading.Thread.Sleep(1000)
    42

// สร้าง lazy value
let lazyValue = lazy expensiveComputation ()

printfn "Before accessing value"
// "Computing..." ยังไม่ถูกพิมพ์

printfn "Accessing value..."
let result1 = lazyValue.Value  // คำนวณครั้งนี้
printfn "Result: %d" result1

let result2 = lazyValue.Value  // ใช้ cached value, ไม่คำนวณซ้ำ
printfn "Result again: %d" result2
```

```fsharp
// Lazy<'T> กับ IsValueCreated
let lazyNum = lazy (printfn "Computing!"; 100)

printfn "IsValueCreated: %b" lazyNum.IsValueCreated  // false
let v = lazyNum.Value
printfn "IsValueCreated: %b" lazyNum.IsValueCreated  // true
printfn "Value: %d" v
```

---

## 8. lazy keyword

```fsharp
// lazy keyword: short form สำหรับ Lazy<'T>

// แบบ Lazy constructor
let lazyVal1 = Lazy<int>(fun () -> 42)

// แบบ lazy keyword (สั้นกว่า ใช้บ่อยกว่า)
let lazyVal2 = lazy 42

// แบบ lazy กับ expression ซับซ้อน
let lazyList = lazy ([1..1000000] |> List.sum)

printfn "Before: %b" lazyList.IsValueCreated  // false
let sum = lazyList.Value
printfn "After: %b" lazyList.IsValueCreated   // true
printfn "Sum: %d" sum
```

```fsharp
// ใช้ lazy สำหรับ expensive initialization

type ExpensiveService() =
    do printfn "ExpensiveService created!"
    member _.DoWork() = printfn "Working..."

// สร้าง service เมื่อต้องการครั้งแรก
let lazyService = lazy ExpensiveService()

printfn "Service not created yet"

// เรียกใช้เมื่อต้องการจริงๆ
lazyService.Value.DoWork()

// เรียกซ้ำ: ใช้ instance เดิม
lazyService.Value.DoWork()
```

---

## 9. Force Evaluation

```fsharp
// Lazy.force: เหมือน .Value แต่เป็น function form
let lazyValue = lazy (42 + 58)

let result = Lazy.force lazyValue
printfn "result: %d" result  // 100
```

```fsharp
// ใช้ |> กับ Lazy.force
let computations = [
    lazy (1 + 1)
    lazy (2 * 3)
    lazy (10 - 4)
]

let results = computations |> List.map Lazy.force
printfn "results: %A" results  // [2; 6; 6]
```

---

## 10. Sequence vs List Performance

```fsharp
open System.Diagnostics

// List: สร้างทั้งหมดก่อน แล้วค่อย process
let measureList () =
    let sw = Stopwatch.StartNew()
    let result =
        [1..10000000]
        |> List.filter (fun x -> x % 2 = 0)
        |> List.map (fun x -> x * x)
        |> List.head
    sw.Stop()
    printfn "List: %d in %dms" result sw.ElapsedMilliseconds

// Seq: lazy, ประมวลผลทีละตัวจนได้ผลลัพธ์
let measureSeq () =
    let sw = Stopwatch.StartNew()
    let result =
        Seq.initInfinite (fun x -> x + 1)
        |> Seq.filter (fun x -> x % 2 = 0)
        |> Seq.map (fun x -> x * x)
        |> Seq.head
    sw.Stop()
    printfn "Seq: %d in %dms" result sw.ElapsedMilliseconds

measureList ()
measureSeq ()
```

```fsharp
// Memory usage comparison (conceptual)

// List: O(n) memory - สร้าง list ทั้งหมด
// [1; 2; 3; ... ; 1000000]

// Seq: O(1) memory - ผลิตทีละตัว
// ในขณะ iterate มีแค่ element ปัจจุบัน

// ตัวอย่าง: หา first prime > 1000000
let isNotComposite n = n >= 2 && [2..int(sqrt(float n))] |> List.forall (fun d -> n % d <> 0)

// แบบ Seq: efficient
let firstBigPrime =
    Seq.initInfinite (fun n -> n + 1000000)
    |> Seq.filter isNotComposite
    |> Seq.head

printfn "First prime > 1000000: %d" firstBigPrime
```

---

## 11. Fibonacci with Sequences

```fsharp
// Fibonacci sequences หลายวิธี

// วิธีที่ 1: Seq.unfold (clean and efficient)
let fibSeq1 = Seq.unfold (fun (a, b) -> Some (a, (b, a + b))) (0, 1)

// วิธีที่ 2: seq computation expression
let fibSeq2 = seq {
    let mutable a = 0
    let mutable b = 1
    while true do
        yield a
        let temp = a + b
        a <- b
        b <- temp
}

// วิธีที่ 3: recursive function
let rec fibN n =
    if n <= 1 then n
    else fibN (n - 1) + fibN (n - 2)

let fibSeq3 = Seq.initInfinite fibN

// ทดสอบ
printfn "Fib (method 1): %A" (fibSeq1 |> Seq.take 10 |> Seq.toList)
printfn "Fib (method 2): %A" (fibSeq2 |> Seq.take 10 |> Seq.toList)
printfn "Fib (method 3): %A" (fibSeq3 |> Seq.take 10 |> Seq.toList)
```

```fsharp
// หา Fibonacci ที่มากกว่า 1000 ตัวแรก
let firstFibOver1000 =
    fibSeq1
    |> Seq.filter (fun n -> n > 1000)
    |> Seq.head

printfn "First Fibonacci > 1000: %d" firstFibOver1000  // 1597

// หา Fibonacci ที่เป็นเลขคู่และน้อยกว่า 1000000
let evenFibs =
    fibSeq1
    |> Seq.takeWhile (fun n -> n < 1000000)
    |> Seq.filter (fun n -> n % 2 = 0)
    |> Seq.toList

printfn "Even Fibs < 1M: %A" evenFibs
printfn "Sum: %d" (List.sum evenFibs)
```

---

## 12. Sieve of Eratosthenes

```fsharp
// Sieve of Eratosthenes: หาจำนวนเฉพาะ

// วิธีที่ 1: แบบ simple (ด้วย HashSet)
let primeSieve limit =
    let isComposite = System.Collections.Generic.HashSet<int>()
    seq {
        for n in 2..limit do
            if not (isComposite.Contains(n)) then
                yield n
                let mutable multiple = n * 2
                while multiple <= limit do
                    isComposite.Add(multiple) |> ignore
                    multiple <- multiple + n
    }

let primes100 = primeSieve 100 |> Seq.toList
printfn "Primes to 100: %A" primes100
printfn "Count: %d" (List.length primes100)  // 25
```

```fsharp
// Sieve แบบ functional (lazy)
let lazyPrimes =
    let rec sieve (s: seq<int>) = seq {
        let head = Seq.head s
        yield head
        yield! sieve (Seq.filter (fun n -> n % head <> 0) (Seq.tail s))
    }
    sieve (Seq.initInfinite (fun n -> n + 2))

// ระวัง: เวอร์ชัน functional นี้ช้ากว่ามากสำหรับตัวเลขใหญ่
let first15Primes = lazyPrimes |> Seq.take 15 |> Seq.toList
printfn "First 15 primes: %A" first15Primes
```

```fsharp
// หาจำนวนเฉพาะด้วย isPrime
let isPrime n =
    if n < 2 then false
    elif n = 2 then true
    elif n % 2 = 0 then false
    else
        let sqrtN = int (sqrt (float n))
        [3..2..sqrtN] |> List.forall (fun d -> n % d <> 0)

// Infinite sequence of primes
let infinitePrimes =
    Seq.initInfinite (fun n -> n + 2)
    |> Seq.filter isPrime

let primes = infinitePrimes |> Seq.take 100 |> Seq.toList
printfn "100th prime: %d" (List.last primes)  // 541
```

---

## 13. File Reading as Sequence

```fsharp
// อ่านไฟล์เป็น sequence (lazy)

// อ่านทีละบรรทัด
let readLines (path: string) : seq<string> = seq {
    use reader = System.IO.File.OpenText(path)
    while not reader.EndOfStream do
        yield reader.ReadLine()
}

// อ่านทีละ chunk
let readChunks (chunkSize: int) (path: string) : seq<string> = seq {
    use reader = System.IO.File.OpenText(path)
    let buffer = Array.create chunkSize ' '
    let mutable bytesRead = reader.Read(buffer, 0, chunkSize)
    while bytesRead > 0 do
        yield System.String(buffer, 0, bytesRead)
        bytesRead <- reader.Read(buffer, 0, chunkSize)
}
```

```fsharp
// ตัวอย่างการใช้งาน (สมมติว่ามีไฟล์)
let processLogFile (path: string) =
    if System.IO.File.Exists(path) then
        readLines path
        |> Seq.filter (fun line -> line.Contains("ERROR"))
        |> Seq.map (fun line -> line.Trim())
        |> Seq.take 100  // จำกัดการประมวลผล
        |> Seq.toList
    else
        []

// สร้างไฟล์ทดสอบ
let testFile = System.IO.Path.GetTempFileName()
System.IO.File.WriteAllLines(testFile, [
    "2024-01-01 INFO Application started"
    "2024-01-01 ERROR Database connection failed"
    "2024-01-01 INFO User logged in"
    "2024-01-01 ERROR Null reference exception"
    "2024-01-01 WARNING Low memory"
])

let errors = processLogFile testFile
printfn "Errors: %A" errors
System.IO.File.Delete(testFile)
```

```fsharp
// อ่าน CSV แบบ lazy streaming
let readCsv (path: string) =
    if System.IO.File.Exists(path) then
        readLines path
        |> Seq.skip 1  // skip header
        |> Seq.map (fun line -> line.Split(','))
        |> Seq.filter (fun parts -> parts.Length >= 3)
    else
        Seq.empty
```

---

## 14. Seq.cache

```fsharp
// Seq.cache: cache ผลลัพธ์ของ sequence เพื่อไม่ต้องคำนวณซ้ำ

let mutable computeCount = 0

let expensiveSeq = seq {
    for i in 1..5 do
        computeCount <- computeCount + 1
        printfn "  Computing element %d" i
        yield i * i
}

printfn "Without cache:"
printfn "Sum 1: %d" (Seq.sum expensiveSeq)     // คำนวณทั้งหมด
printfn "Sum 2: %d" (Seq.sum expensiveSeq)     // คำนวณซ้ำ!
printfn "Total computes: %d" computeCount  // 10

computeCount <- 0
printfn "\nWith cache:"
let cachedSeq = Seq.cache expensiveSeq

printfn "Sum 1: %d" (Seq.sum cachedSeq)    // คำนวณ
printfn "Sum 2: %d" (Seq.sum cachedSeq)    // ใช้ cache
printfn "Total computes: %d" computeCount  // 5 (ไม่ใช่ 10)
```

```fsharp
// เมื่อไหรใช้ Seq.cache?
// - เมื่อ iterate sequence หลายครั้ง
// - เมื่อ computation ใน sequence มีค่าใช้จ่ายสูง
// - เมื่อต้องการ share sequence ระหว่าง operations

// ระวัง: cache เก็บ elements ทั้งหมดใน memory
// ถ้า infinite sequence ไม่ควรใช้ cache ทั้งหมด
```

---

## 15. yield and yield!

```fsharp
// yield: ผลิต element เดียว
// yield!: ผลิต elements จาก iterable อื่น (flatten)

// yield ธรรมดา
let simple = seq {
    yield 1
    yield 2
    yield 3
}

// yield! แทรก sequence อื่น
let combined = seq {
    yield 0
    yield! [1; 2; 3]
    yield 4
    yield! seq { 5..8 }
    yield 9
}

printfn "combined: %A" (Seq.toList combined)
// [0; 1; 2; 3; 4; 5; 6; 7; 8; 9]
```

```fsharp
// yield! สำหรับ recursive sequences

// Tree traversal
type Tree<'a> = Leaf | Node of 'a * Tree<'a> * Tree<'a>

let rec inorder = function
    | Leaf -> Seq.empty
    | Node(v, l, r) -> seq {
        yield! inorder l
        yield v
        yield! inorder r
      }

let rec preorder = function
    | Leaf -> Seq.empty
    | Node(v, l, r) -> seq {
        yield v
        yield! preorder l
        yield! preorder r
      }

let tree = Node(4,
    Node(2, Node(1, Leaf, Leaf), Node(3, Leaf, Leaf)),
    Node(6, Node(5, Leaf, Leaf), Node(7, Leaf, Leaf)))

printfn "Inorder: %A" (inorder tree |> Seq.toList)   // [1;2;3;4;5;6;7]
printfn "Preorder: %A" (preorder tree |> Seq.toList) // [4;2;1;3;6;5;7]
```

```fsharp
// yield! กับ Lists, Arrays, Seqs
let mixed = seq {
    yield! [1; 2; 3]
    yield! [|4; 5; 6|]
    yield! seq { 7..9 }
}

printfn "mixed: %A" (Seq.toList mixed)
// [1; 2; 3; 4; 5; 6; 7; 8; 9]
```

```fsharp
// Flatten nested sequences
let nested = [[1; 2]; [3; 4]; [5; 6]]

let flattened = seq {
    for inner in nested do
        yield! inner
}

printfn "flattened: %A" (Seq.toList flattened)
// [1; 2; 3; 4; 5; 6]

// หรือใช้ Seq.concat
let flattened2 = Seq.concat nested |> Seq.toList
printfn "flattened2: %A" flattened2
```

---

## 16. Advanced Sequence Patterns

```fsharp
// Sliding window
let window n (s: seq<'a>) = seq {
    let buffer = System.Collections.Generic.Queue<'a>()
    for item in s do
        buffer.Enqueue(item)
        if buffer.Count = n then
            yield buffer |> Seq.toList
            buffer.Dequeue() |> ignore
}

let data = seq { 1..10 }
let windows3 = window 3 data |> Seq.toList

printfn "Sliding windows of 3: %A" windows3
// [[1;2;3]; [2;3;4]; [3;4;5]; [4;5;6]; [5;6;7]; [6;7;8]; [7;8;9]; [8;9;10]]
```

```fsharp
// Moving average
let movingAverage n s =
    window n s
    |> Seq.map (fun w -> List.averageBy float w)

let prices = seq { 10; 12; 8; 15; 11; 13; 9; 14 }
let ma3 = movingAverage 3 prices |> Seq.toList

printfn "Prices: %A" (Seq.toList prices)
printfn "Moving Avg (3): %A" ma3
```

```fsharp
// Interleave sequences
let interleave (s1: seq<'a>) (s2: seq<'a>) = seq {
    use e1 = s1.GetEnumerator()
    use e2 = s2.GetEnumerator()
    let mutable c1 = e1.MoveNext()
    let mutable c2 = e2.MoveNext()
    while c1 || c2 do
        if c1 then
            yield e1.Current
            c1 <- e1.MoveNext()
        if c2 then
            yield e2.Current
            c2 <- e2.MoveNext()
}

let odds = seq { 1..2..10 }
let evens = seq { 2..2..10 }
let merged = interleave odds evens |> Seq.toList

printfn "interleaved: %A" merged  // [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]
```

---

## สรุป (Summary)

```fsharp
// สรุป Sequences and Lazy Evaluation

// 1. seq { } computation expression
let basic = seq { 1..5 }

// 2. Infinite sequences
let infinite = Seq.initInfinite id

// 3. Seq.unfold
let countdown = Seq.unfold (fun n -> if n = 0 then None else Some(n, n-1)) 5

// 4. Lazy<'T>
let expensive = lazy (printfn "Computing!"; 42)

// 5. Take what you need
let first5 = infinite |> Seq.take 5 |> Seq.toList

// 6. yield and yield!
let combined = seq {
    yield! first5
    yield! countdown
}

// 7. Cache for reuse
let cached = Seq.cache combined

printfn "countdown: %A" (countdown |> Seq.toList)
printfn "combined: %A" (combined |> Seq.toList)
printfn "cached twice: %d %d" (Seq.sum cached) (Seq.sum cached)
```

Sequences เหมาะสำหรับ:
- **Large data**: ไม่โหลดทั้งหมดเข้า memory
- **Infinite data**: เช่น streams, events, random data
- **Lazy evaluation**: คำนวณเมื่อต้องการจริงๆ
- **Pipelines**: ประมวลผลแบบ streaming

ข้อดีหลัก:
- **Memory efficient**: O(1) memory สำหรับ infinite sequences
- **Early termination**: หยุดเมื่อได้ผลลัพธ์ที่ต้องการ
- **Composable**: รวม operations ได้ง่าย
