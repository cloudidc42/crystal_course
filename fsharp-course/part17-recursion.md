# Part 17 - การเรียกซ้ำ (Recursion and Tail Recursion)

## บทนำ (Introduction)

Recursion คือเทคนิคที่ฟังก์ชันเรียกตัวเองเพื่อแก้ปัญหา ใน F# recursion เป็นวิธีหลักแทน loops สำหรับ functional programming

---

## 1. Recursive Functions with rec

```fsharp
// ใน F# ต้องใช้ rec keyword เพื่อระบุว่า function เป็น recursive

// Factorial
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

printfn "5! = %d" (factorial 5)   // 120
printfn "10! = %d" (factorial 10)  // 3628800

// Pattern matching version
let rec factorial' = function
    | n when n <= 1 -> 1
    | n -> n * factorial' (n - 1)

printfn "7! = %d" (factorial' 7)  // 5040
```

```fsharp
// Sum of list
let rec sumList = function
    | [] -> 0
    | head :: tail -> head + sumList tail

printfn "sum [1..5] = %d" (sumList [1..5])  // 15

// Length of list
let rec myLength = function
    | [] -> 0
    | _ :: tail -> 1 + myLength tail

printfn "length [1..10] = %d" (myLength [1..10])  // 10
```

---

## 2. Factorial, Fibonacci Examples

```fsharp
// Factorial variations

// Simple recursive
let rec fact1 n =
    if n = 0 then 1
    else n * fact1 (n - 1)

// Match expression
let rec fact2 = function
    | 0 -> 1
    | n -> n * fact2 (n - 1)

// With bigint for large numbers
let rec factBig (n: bigint) =
    if n = 0I then 1I
    else n * factBig (n - 1I)

printfn "20! = %A" (factBig 20I)  // 2432902008176640000
printfn "50! = %A" (factBig 50I)  // big number!
```

```fsharp
// Fibonacci variations

// Naive recursive (exponential time!)
let rec fibNaive n =
    if n <= 1 then n
    else fibNaive (n - 1) + fibNaive (n - 2)

// Test: fibNaive 30 เริ่มช้าแล้ว
let sw = System.Diagnostics.Stopwatch.StartNew()
let r = fibNaive 30
sw.Stop()
printfn "fibNaive 30 = %d (%.3fs)" r sw.Elapsed.TotalSeconds

// With memoization
let memo = System.Collections.Generic.Dictionary<int, int64>()
let rec fibMemo n =
    if n <= 1 then int64 n
    else
        match memo.TryGetValue(n) with
        | true, v -> v
        | false, _ ->
            let v = fibMemo (n-1) + fibMemo (n-2)
            memo.[n] <- v
            v

sw.Restart()
let r2 = fibMemo 100
sw.Stop()
printfn "fibMemo 100 = %d (%.6fs)" r2 sw.Elapsed.TotalSeconds
```

---

## 3. Stack Overflow Problem

```fsharp
// ปัญหา: Recursive function ที่ deep มากอาจ overflow stack

// ทดสอบ stack overflow
let rec infiniteSum n =
    n + infiniteSum (n + 1)  // ไม่มี base case -> stack overflow

// ฟังก์ชันที่ทำให้ stack overflow ถ้า n ใหญ่เกินไป
let rec naiveSum n =
    if n = 0 then 0
    else n + naiveSum (n - 1)

// naiveSum 100000 อาจ overflow ขึ้นอยู่กับ stack size

// ทดสอบอย่างปลอดภัย
try
    let result = naiveSum 100000
    printfn "naiveSum 100000 = %d" result
with
| :? System.StackOverflowException ->
    printfn "Stack overflow!"
| ex ->
    printfn "Error: %s" ex.Message
```

---

## 4. Tail Recursion Concept

```fsharp
// Tail recursion: recursive call เป็น operation สุดท้าย
// ไม่ต้องเก็บ stack frame ก่อนหน้า

// NOT tail recursive: ต้องรอ recursive call แล้วคูณ n
let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)  // ยังต้องทำ *n หลังจาก recursive call

// Tail recursive: recursive call เป็น operation สุดท้ายจริงๆ
let rec factTail acc n =
    if n <= 1 then acc
    else factTail (acc * n) (n - 1)  // recursive call เป็น last operation

let factorial' n = factTail 1 n

printfn "10! = %d" (factorial' 10)  // 3628800
```

---

## 5. Tail Call Optimization in F#

```fsharp
// F# ทำ tail call optimization (TCO) สำหรับ tail recursive functions
// ทำให้ไม่เกิด stack overflow แม้กับ deep recursion

// Tail recursive sum (จะไม่ overflow)
let rec sumTail acc = function
    | [] -> acc
    | head :: tail -> sumTail (acc + head) tail

let sumList = sumTail 0

// ทดสอบกับ large list
let bigList = [1..1000000]
let bigSum = sumList bigList
printfn "Sum of 1..1000000 = %d" bigSum  // 500000500000

// ถ้าไม่ใช่ tail recursive จะ overflow
// let rec sumNotTail = function
//     | [] -> 0
//     | h :: t -> h + sumNotTail t  // ไม่ใช่ tail call!
// sumNotTail bigList  // StackOverflowException!
```

```fsharp
// ตรวจสอบ tail recursion ด้วย <TailCall> attribute (F# 8+)
// [<TailCall>]
let rec isTailRecursive n acc =
    if n <= 0 then acc
    else isTailRecursive (n - 1) (acc + n)

printfn "%d" (isTailRecursive 1000000 0)
```

---

## 6. Converting to Tail-Recursive

```fsharp
// แปลง non-tail recursive เป็น tail recursive

// Non-tail recursive Fibonacci (exponential)
let rec fibSlow n =
    if n <= 1 then n
    else fibSlow (n - 1) + fibSlow (n - 2)

// Tail recursive Fibonacci (linear)
let rec fibFast n a b =
    if n = 0 then a
    else fibFast (n - 1) b (a + b)

let fib n = fibFast n 0 1

printfn "fib 50 = %d" (fib 50)   // fast!
printfn "fib 100 = %d" (fib 100) // fast!
```

```fsharp
// แปลง map เป็น tail recursive

// Non-tail recursive
let rec mapNonTail f = function
    | [] -> []
    | h :: t -> f h :: mapNonTail f t  // cons หลัง recursive call

// Tail recursive (ด้วย accumulator + reverse)
let rec mapTail f acc = function
    | [] -> List.rev acc
    | h :: t -> mapTail f (f h :: acc) t

let myMap f lst = mapTail f [] lst

printfn "%A" (myMap ((*) 2) [1..5])  // [2; 4; 6; 8; 10]
```

```fsharp
// แปลง filter เป็น tail recursive

let rec filterTail pred acc = function
    | [] -> List.rev acc
    | h :: t ->
        if pred h then filterTail pred (h :: acc) t
        else filterTail pred acc t

let myFilter pred lst = filterTail pred [] lst

printfn "%A" (myFilter (fun x -> x > 3) [1..6])  // [4; 5; 6]
```

---

## 7. Accumulator Pattern

```fsharp
// Accumulator pattern: ส่งค่าสะสมเป็น argument แทนการส่งคืนค่าสะสม

// ตัวอย่าง: reverse list
let rec reverseNonTail = function
    | [] -> []
    | h :: t -> reverseNonTail t @ [h]  // O(n^2)!

let rec reverseTail acc = function
    | [] -> acc
    | h :: t -> reverseTail (h :: acc) t  // O(n)

let myReverse lst = reverseTail [] lst

printfn "%A" (myReverse [1..5])  // [5; 4; 3; 2; 1]
```

```fsharp
// Accumulator pattern สำหรับ flatten
let rec flattenTail acc = function
    | [] -> List.rev acc
    | [] :: rest -> flattenTail acc rest
    | (h :: t) :: rest -> flattenTail (h :: acc) (t :: rest)

let myFlatten lst = flattenTail [] lst

let nested = [[1; 2]; [3; 4]; [5]]
printfn "%A" (myFlatten nested)  // [1; 2; 3; 4; 5]
```

```fsharp
// Accumulator pattern สำหรับ counting
let rec countWhere pred acc = function
    | [] -> acc
    | h :: t ->
        if pred h then countWhere pred (acc + 1) t
        else countWhere pred acc t

let myCount pred lst = countWhere pred 0 lst

let evens = myCount (fun x -> x % 2 = 0) [1..20]
printfn "evens count: %d" evens  // 10
```

---

## 8. CPS (Continuation-Passing Style)

```fsharp
// CPS: แทนที่จะส่งคืนค่า ให้ส่ง "ต่อจะทำอะไร" (continuation) เป็น argument

// Factorial แบบ CPS
let rec factCPS n cont =
    if n <= 1 then cont 1
    else factCPS (n - 1) (fun result -> cont (n * result))

// เรียกด้วย identity continuation
let factorial n = factCPS n id

printfn "5! = %d" (factorial 5)   // 120
printfn "10! = %d" (factorial 10)  // 3628800
```

```fsharp
// CPS ทำให้ทุก recursive call เป็น tail call
// แต่สร้าง closure chain ใหญ่

// Fibonacci แบบ CPS
let rec fibCPS n cont =
    if n <= 1 then cont n
    else
        fibCPS (n - 1) (fun r1 ->
            fibCPS (n - 2) (fun r2 ->
                cont (r1 + r2)))

let fib n = fibCPS n id

// ระวัง: fibCPS ยังคง exponential time แต่ stack ไม่ overflow
printfn "fib 20 = %d" (fib 20)  // 6765
```

```fsharp
// CPS กับ Tree traversal
type Tree<'a> = Leaf | Node of 'a * Tree<'a> * Tree<'a>

let rec sumCPS tree cont =
    match tree with
    | Leaf -> cont 0
    | Node(v, l, r) ->
        sumCPS l (fun leftSum ->
            sumCPS r (fun rightSum ->
                cont (v + leftSum + rightSum)))

let tree = Node(1, Node(2, Leaf, Leaf), Node(3, Leaf, Leaf))
let treeSum = sumCPS tree id
printfn "Tree sum = %d" treeSum  // 6
```

---

## 9. Mutual Recursion

```fsharp
// Mutual recursion: ฟังก์ชัน A เรียก B และ B เรียก A
// ใช้ and keyword

let rec isEven n =
    if n = 0 then true
    else isOdd (n - 1)
and isOdd n =
    if n = 0 then false
    else isEven (n - 1)

printfn "isEven 4 = %b" (isEven 4)   // true
printfn "isOdd 7 = %b" (isOdd 7)     // true
printfn "isEven 100 = %b" (isEven 100) // true
```

```fsharp
// Mutual recursion: Expression evaluator

type Expr =
    | Num of int
    | Add of Expr * Expr
    | Mul of Expr * Expr
    | Neg of Expr
    | IfPos of Expr * Expr * Expr  // if > 0 then else

let rec eval = function
    | Num n -> n
    | Add(l, r) -> eval l + eval r
    | Mul(l, r) -> eval l * eval r
    | Neg e -> -(eval e)
    | IfPos(cond, t, f) ->
        if eval cond > 0 then eval t
        else eval f

// Test
let expr = Add(Num 3, Mul(Num 2, Num 4))  // 3 + (2 * 4) = 11
printfn "3 + 2*4 = %d" (eval expr)

let expr2 = IfPos(Add(Num 1, Num(-2)), Num 100, Num(-100))
// if (1 + (-2)) > 0 then 100 else -100
// if (-1) > 0 then 100 else -100
// -100
printfn "if (-1>0) 100 -100 = %d" (eval expr2)  // -100
```

```fsharp
// Mutual recursion: Parser (simplified)
let rec parseDigit (s: string) pos =
    if pos >= s.Length then None
    else
        match s.[pos] with
        | c when c >= '0' && c <= '9' -> Some (int c - int '0', pos + 1)
        | _ -> None

and parseNumber (s: string) pos =
    match parseDigit s pos with
    | None -> None
    | Some (d, nextPos) ->
        match parseNumber s nextPos with
        | None -> Some (d, nextPos)
        | Some (rest, finalPos) -> Some (d * (pown 10 (finalPos - nextPos)) + rest, finalPos)

let result = parseNumber "12345abc" 0
printfn "parseNumber: %A" result  // Some(12345, 5)
```

---

## 10. Indirect Recursion

```fsharp
// Indirect recursion ผ่าน higher-order functions

// ส่ง self เป็น argument
let fixpoint f x =
    let rec go x = f go x
    go x

// Factorial ด้วย fixpoint
let factFixpoint = fixpoint (fun self n ->
    if n <= 1 then 1
    else n * self (n - 1))

printfn "5! = %d" (factFixpoint 5)  // 120
```

```fsharp
// Y-combinator (ทางทฤษฎี)
// ใน F# มี rec ทำให้ไม่ต้องใช้ แต่เข้าใจไว้ดี

let Y f =
    let inner x = f (fun v -> x x v)
    inner inner

let factY = Y (fun self n ->
    if n <= 1 then 1
    else n * self (n - 1))

printfn "factY 6 = %d" (factY 6)  // 720
```

---

## 11. Tree Traversal (DFS, BFS)

```fsharp
// Tree type
type BTree<'a> = Leaf | Node of 'a * BTree<'a> * BTree<'a>

// สร้าง BST จาก list
let rec insert x = function
    | Leaf -> Node(x, Leaf, Leaf)
    | Node(v, l, r) ->
        if x < v then Node(v, insert x l, r)
        elif x > v then Node(v, l, insert x r)
        else Node(v, l, r)

let bst = List.fold (fun t x -> insert x t) Leaf [5; 3; 7; 1; 4; 6; 8; 2]
```

```fsharp
// DFS traversals
let rec inorder = function
    | Leaf -> []
    | Node(v, l, r) -> inorder l @ [v] @ inorder r

let rec preorder = function
    | Leaf -> []
    | Node(v, l, r) -> [v] @ preorder l @ preorder r

let rec postorder = function
    | Leaf -> []
    | Node(v, l, r) -> postorder l @ postorder r @ [v]

printfn "Inorder: %A" (inorder bst)   // sorted!
printfn "Preorder: %A" (preorder bst)
printfn "Postorder: %A" (postorder bst)
```

```fsharp
// BFS using queue
let bfs tree =
    let rec go queue acc =
        match queue with
        | [] -> List.rev acc
        | Leaf :: rest -> go rest acc
        | Node(v, l, r) :: rest -> go (rest @ [l; r]) (v :: acc)
    go [tree] []

printfn "BFS: %A" (bfs bst)
```

```fsharp
// Tree depth
let rec depth = function
    | Leaf -> 0
    | Node(_, l, r) -> 1 + max (depth l) (depth r)

// Tree size
let rec size = function
    | Leaf -> 0
    | Node(_, l, r) -> 1 + size l + size r

// Sum all nodes
let rec treeSum = function
    | Leaf -> 0
    | Node(v, l, r) -> v + treeSum l + treeSum r

printfn "Depth: %d" (depth bst)
printfn "Size: %d" (size bst)
printfn "Sum: %d" (treeSum bst)
```

```fsharp
// Path finding in tree
let rec pathTo target = function
    | Leaf -> None
    | Node(v, l, r) ->
        if v = target then Some []
        else
            match pathTo target l with
            | Some path -> Some ("left" :: path)
            | None ->
                match pathTo target r with
                | Some path -> Some ("right" :: path)
                | None -> None

let path = pathTo 4 bst
printfn "Path to 4: %A" path
```

---

## 12. Ackermann Function

```fsharp
// Ackermann function: grows faster than any primitive recursive function
// ใช้ทดสอบ stack depth และ recursion depth

let rec ackermann m n =
    if m = 0 then n + 1
    elif n = 0 then ackermann (m - 1) 1
    else ackermann (m - 1) (ackermann m (n - 1))

// ค่าเล็กๆ เท่านั้น! ค่าใหญ่ทำให้ stack overflow หรือใช้เวลามาก
printfn "A(0,0) = %d" (ackermann 0 0)  // 1
printfn "A(1,1) = %d" (ackermann 1 1)  // 3
printfn "A(2,2) = %d" (ackermann 2 2)  // 7
printfn "A(3,3) = %d" (ackermann 3 3)  // 61
// A(4,4) = 2^(2^(2^65536)) - 3  // ใหญ่มาก!
```

```fsharp
// Ackermann แบบ CPS เพื่อหลีกเลี่ยง stack overflow
let ackermannCPS m n cont =
    let rec ack m n cont =
        if m = 0 then cont (n + 1)
        elif n = 0 then ack (m - 1) 1 cont
        else ack m (n - 1) (fun r -> ack (m - 1) r cont)
    ack m n cont

let r = ackermannCPS 3 6 id
printfn "A(3,6) = %d" r  // 509
```

---

## 13. Tower of Hanoi

```fsharp
// Tower of Hanoi: ปัญหาคลาสสิก
// เลื่อน n discs จาก source ไป target โดยใช้ auxiliary

let rec hanoi n source target auxiliary =
    if n = 1 then
        printfn "Move disc 1 from %s to %s" source target
    else
        hanoi (n - 1) source auxiliary target  // เลื่อน n-1 ไป auxiliary
        printfn "Move disc %d from %s to %s" n source target
        hanoi (n - 1) auxiliary target source  // เลื่อน n-1 จาก auxiliary ไป target

printfn "Hanoi 3 discs:"
hanoi 3 "A" "C" "B"
```

```fsharp
// Hanoi ที่ส่งคืน list ของ moves
let rec hanoiMoves n source target auxiliary =
    if n = 0 then []
    else
        let movesToAux = hanoiMoves (n - 1) source auxiliary target
        let moveDisc = [(source, target)]
        let movesFromAux = hanoiMoves (n - 1) auxiliary target source
        movesToAux @ moveDisc @ movesFromAux

let moves = hanoiMoves 3 'A' 'C' 'B'
printfn "\nHanoi 3 moves:"
moves |> List.iter (fun (f, t) -> printfn "  %c -> %c" f t)
printfn "Total moves: %d (= 2^n - 1 = %d)" (List.length moves) (pown 2 3 - 1)
```

---

## 14. Additional Recursive Patterns

```fsharp
// สร้าง permutations ด้วย recursion
let rec permutations = function
    | [] -> [[]]
    | lst ->
        [for x in lst do
            let rest = List.filter ((<>) x) lst
            for perm in permutations rest do
                yield x :: perm]

let perms = permutations [1; 2; 3]
printfn "Permutations of [1;2;3]:"
perms |> List.iter (printfn "  %A")
printfn "Count: %d" (List.length perms)  // 6 = 3!
```

```fsharp
// สร้าง combinations ด้วย recursion
let rec combinations k lst =
    if k = 0 then [[]]
    else
        match lst with
        | [] -> []
        | h :: t ->
            let withH = combinations (k - 1) t |> List.map (fun c -> h :: c)
            let withoutH = combinations k t
            withH @ withoutH

let combs = combinations 2 [1; 2; 3; 4]
printfn "C(4,2) combinations:"
combs |> List.iter (printfn "  %A")
printfn "Count: %d" (List.length combs)  // 6 = C(4,2)
```

```fsharp
// Merge sort (recursive)
let rec mergeSort = function
    | [] | [_] as lst -> lst
    | lst ->
        let mid = List.length lst / 2
        let left = mergeSort (List.take mid lst)
        let right = mergeSort (List.skip mid lst)
        
        let rec merge l r =
            match l, r with
            | [], x | x, [] -> x
            | lh :: lt, rh :: rt ->
                if lh <= rh then lh :: merge lt r
                else rh :: merge l rt
        
        merge left right

let sorted = mergeSort [5; 2; 8; 1; 9; 3; 7; 4; 6]
printfn "Sorted: %A" sorted
```

---

## สรุป (Summary)

```fsharp
// สรุป Recursion ใน F#

// 1. ใช้ rec keyword
let rec sum = function
    | [] -> 0
    | h :: t -> h + sum t

// 2. Tail recursion (ป้องกัน stack overflow)
let rec sumTail acc = function
    | [] -> acc
    | h :: t -> sumTail (acc + h) t
let sumList lst = sumTail 0 lst

// 3. Accumulator pattern
let rec reverse acc = function
    | [] -> acc
    | h :: t -> reverse (h :: acc) t

// 4. Mutual recursion (and keyword)
let rec even n = if n = 0 then true else odd (n - 1)
and odd n = if n = 0 then false else even (n - 1)

// 5. Tree operations
type Tree = Leaf | Node of int * Tree * Tree
let rec treeHeight = function
    | Leaf -> 0
    | Node(_, l, r) -> 1 + max (treeHeight l) (treeHeight r)

// Test all
printfn "sum [1..10] = %d" (sum [1..10])
printfn "sumTail [1..10] = %d" (sumList [1..10])
printfn "reverse [1..5] = %A" (reverse [] [1..5])
printfn "even 8 = %b" (even 8)
printfn "odd 7 = %b" (odd 7)
```

**หลักการสำคัญ:**
1. **Base case**: ต้องมีเงื่อนไขที่หยุด recursion
2. **Progress**: แต่ละ recursive call ต้องเข้าใกล้ base case
3. **Tail recursion**: ใช้เมื่อ recursion deep มาก
4. **Accumulator**: สะสมผลลัพธ์เพื่อ enable tail recursion
5. **CPS**: ทางเลือกสุดท้ายสำหรับกรณีที่ซับซ้อนมาก
