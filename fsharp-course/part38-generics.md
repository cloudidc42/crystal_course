# Part 38 - Generics เชิงลึก (Generics in Depth)

## บทนำ (Introduction)

Generics ใน F# ช่วยให้เขียน code ที่ทำงานได้กับหลาย types โดยไม่ต้องเขียนซ้ำ F# มี type inference ที่ทรงพลังทำให้ไม่ต้องระบุ type parameters ส่วนใหญ่ นอกจากนี้ F# ยังมี SRTP (Statically Resolved Type Parameters) ที่ทำให้สร้าง generic code ที่มีประสิทธิภาพสูงได้

---

## 1. Generic Functions

```fsharp
// Generic function พื้นฐาน
let identity<'T> (x: 'T) : 'T = x

// Type inference - ไม่ต้องระบุ type parameter
let doubled x = x, x  // Automatically generic

// Generic function ที่ทำงานกับ lists
let first (lst: 'T list) : 'T option =
    match lst with
    | [] -> None
    | x :: _ -> Some x

let last (lst: 'T list) : 'T option =
    match lst with
    | [] -> None
    | _ -> Some (List.last lst)

let swap (a: 'T, b: 'U) : 'U * 'T = (b, a)

// ทดสอบ
printfn "identity 42: %d" (identity 42)
printfn "identity 'hello': %s" (identity "hello")
printfn "doubled 5: %A" (doubled 5)
printfn "first [1;2;3]: %A" (first [1;2;3])
printfn "last [1;2;3]: %A" (last [1;2;3])
printfn "swap (1, 'a'): %A" (swap (1, 'a'))

// Generic higher-order functions
let applyTwice (f: 'T -> 'T) (x: 'T) = f (f x)
let compose (f: 'B -> 'C) (g: 'A -> 'B) (x: 'A) = f (g x)

let double = (*) 2
let addOne = (+) 1

printfn "applyTwice double 3: %d" (applyTwice double 3)  // 12
printfn "compose double addOne 5: %d" (compose double addOne 5)  // 12

// Generic fold
let foldCustom (folder: 'State -> 'T -> 'State) (state: 'State) (list: 'T list) =
    let mutable acc = state
    for item in list do
        acc <- folder acc item
    acc

let sum = foldCustom (+) 0 [1;2;3;4;5]
let product = foldCustom (*) 1 [1;2;3;4;5]
printfn "Sum: %d, Product: %d" sum product
```

---

## 2. Generic Types

```fsharp
// Generic type definitions

// Generic pair
type Pair<'T, 'U> = {
    First: 'T
    Second: 'U
}

module Pair =
    let create first second = { First = first; Second = second }
    let map f g pair = { First = f pair.First; Second = g pair.Second }
    let swap pair = { First = pair.Second; Second = pair.First }
    let toTuple pair = (pair.First, pair.Second)

let pair1 = Pair.create 1 "hello"
let pair2 = Pair.create 3.14 true

printfn "pair1: (%d, %s)" pair1.First pair1.Second
printfn "mapped: %A" (Pair.map ((*) 2) String.length pair1)

// Generic Either type (Left or Right)
type Either<'L, 'R> =
    | Left of 'L
    | Right of 'R

module Either =
    let map f = function
        | Left l -> Left l
        | Right r -> Right (f r)
    
    let mapLeft f = function
        | Left l -> Left (f l)
        | Right r -> Right r
    
    let bind f = function
        | Left l -> Left l
        | Right r -> f r
    
    let either fLeft fRight = function
        | Left l -> fLeft l
        | Right r -> fRight r

let result1: Either<string, int> = Right 42
let result2: Either<string, int> = Left "error"

let doubled1 = Either.map ((*) 2) result1
let doubled2 = Either.map ((*) 2) result2

printfn "doubled1: %A" doubled1  // Right 84
printfn "doubled2: %A" doubled2  // Left "error"

// Generic Binary Tree
type Tree<'T> =
    | Leaf
    | Node of value: 'T * left: Tree<'T> * right: Tree<'T>

module Tree =
    let leaf = Leaf
    let node value left right = Node(value, left, right)
    
    let rec insert (value: 'T when 'T : comparison) tree =
        match tree with
        | Leaf -> Node(value, Leaf, Leaf)
        | Node(v, left, right) ->
            if value < v then Node(v, insert value left, right)
            elif value > v then Node(v, left, insert value right)
            else tree
    
    let rec contains (value: 'T when 'T : comparison) tree =
        match tree with
        | Leaf -> false
        | Node(v, left, right) ->
            if value = v then true
            elif value < v then contains value left
            else contains value right
    
    let rec toList tree =
        match tree with
        | Leaf -> []
        | Node(v, left, right) ->
            toList left @ [v] @ toList right
    
    let rec height tree =
        match tree with
        | Leaf -> 0
        | Node(_, left, right) -> 1 + max (height left) (height right)

let bst = 
    [5; 3; 8; 1; 4; 7; 9; 2; 6]
    |> List.fold (fun tree x -> Tree.insert x tree) Leaf

printfn "\nBST in-order: %A" (Tree.toList bst)
printfn "Height: %d" (Tree.height bst)
printfn "Contains 4: %b" (Tree.contains 4 bst)
printfn "Contains 10: %b" (Tree.contains 10 bst)
```

---

## 3. Type Constraints

```fsharp
// Type constraints จำกัด type parameters ให้มีคุณสมบัติที่กำหนด

// Constraint: equality
let findFirst<'T when 'T : equality> (value: 'T) (lst: 'T list) =
    lst |> List.tryFind (fun x -> x = value)

// Constraint: comparison
let maximum<'T when 'T : comparison> (lst: 'T list) =
    match lst with
    | [] -> failwith "Empty list"
    | x :: rest -> List.fold max x rest

let minimum<'T when 'T : comparison> (lst: 'T list) =
    match lst with
    | [] -> failwith "Empty list"
    | x :: rest -> List.fold min x rest

// ทดสอบ
printfn "Find 3 in [1;2;3;4]: %A" (findFirst 3 [1;2;3;4])
printfn "Max of [3;1;4;1;5;9]: %d" (maximum [3;1;4;1;5;9])
printfn "Min of [3;1;4;1;5;9]: %d" (minimum [3;1;4;1;5;9])

// Multiple constraints
let printAndSort<'T when 'T : comparison and 'T : (override ToString : unit -> string)> 
    (lst: 'T list) =
    let sorted = List.sort lst
    sorted |> List.iter (fun x -> printfn "  %s" (x.ToString()))
    sorted

printfn "\nSorted strings:"
printAndSort ["banana"; "apple"; "cherry"; "date"] |> ignore
```

---

## 4. where 'T : equality

```fsharp
// equality constraint - 'T รองรับ = operator

let contains<'T when 'T : equality> (item: 'T) (collection: 'T list) =
    List.contains item collection

let distinct<'T when 'T : equality> (lst: 'T list) =
    lst |> List.fold (fun acc x ->
        if contains x acc then acc
        else acc @ [x]
    ) []

let groupBy<'T, 'K when 'K : equality> (keyFn: 'T -> 'K) (lst: 'T list) =
    lst |> List.fold (fun groups item ->
        let key = keyFn item
        match groups |> List.tryFind (fun (k, _) -> k = key) with
        | Some (_, items) ->
            groups |> List.map (fun (k, v) ->
                if k = key then (k, v @ [item]) else (k, v))
        | None -> groups @ [(key, [item])]
    ) []

printfn "Distinct [1;2;3;2;1;4]: %A" (distinct [1;2;3;2;1;4])
printfn "Contains 3 in [1;2;3]: %b" (contains 3 [1;2;3])

let words = ["apple"; "banana"; "avocado"; "berry"; "cherry"; "apricot"]
let grouped = groupBy (fun (s: string) -> s.[0]) words
printfn "\nGrouped by first letter:"
for (letter, words) in grouped do
    printfn "  %c: %A" letter words
```

---

## 5. where 'T : comparison

```fsharp
// comparison constraint - 'T รองรับ < > <= >= compare

let sortUnique<'T when 'T : comparison> (lst: 'T list) =
    lst |> List.distinct |> List.sort

let range<'T when 'T : comparison> (lst: 'T list) =
    match lst with
    | [] -> None
    | _ -> Some (List.min lst, List.max lst)

let median<'T when 'T : comparison> (lst: 'T list) =
    let sorted = List.sort lst
    let n = sorted.Length
    if n = 0 then None
    elif n % 2 = 1 then Some sorted.[n / 2]
    else Some sorted.[(n / 2) - 1]  // return lower median

let binarySearch<'T when 'T : comparison> (target: 'T) (arr: 'T array) =
    let mutable lo = 0
    let mutable hi = arr.Length - 1
    let mutable result = -1
    
    while lo <= hi && result = -1 do
        let mid = lo + (hi - lo) / 2
        let cmp = compare arr.[mid] target
        if cmp = 0 then result <- mid
        elif cmp < 0 then lo <- mid + 1
        else hi <- mid - 1
    
    result

let numbers = [5; 3; 8; 1; 9; 3; 2; 7; 1; 4]
printfn "Sort unique: %A" (sortUnique numbers)
printfn "Range: %A" (range numbers)
printfn "Median: %A" (median numbers)

let sorted = numbers |> sortUnique |> List.toArray
printfn "Binary search 5: index %d" (binarySearch 5 sorted)
printfn "Binary search 6: index %d" (binarySearch 6 sorted)  // -1
```

---

## 6. where 'T : struct

```fsharp
// struct constraint - 'T ต้องเป็น value type

// เหมาะสำหรับ generic code ที่ต้องการ avoid null

let nullableToOption<'T when 'T : struct> (n: System.Nullable<'T>) =
    if n.HasValue then Some n.Value
    else None

let optionToNullable<'T when 'T : struct> (opt: 'T option) =
    match opt with
    | Some v -> System.Nullable<'T>(v)
    | None -> System.Nullable<'T>()

let intNullable: System.Nullable<int> = System.Nullable(42)
let noInt: System.Nullable<int> = System.Nullable()

printfn "Nullable 42 -> %A" (nullableToOption intNullable)  // Some 42
printfn "Nullable null -> %A" (nullableToOption noInt)       // None

let someInt = Some 100
let noInt2 = None

printfn "Some 100 -> HasValue: %b, Value: %A" 
    (optionToNullable someInt).HasValue 
    (optionToNullable someInt).Value  // True, 100

// Generic memory pool สำหรับ structs
type StructPool<'T when 'T : struct>(size: int) =
    let pool = Array.zeroCreate<'T> size
    let mutable nextIndex = 0
    
    member this.Get() =
        if nextIndex >= size then failwith "Pool exhausted"
        let item = pool.[nextIndex]
        nextIndex <- nextIndex + 1
        item
    
    member this.Reset() = nextIndex <- 0
    member this.Available = size - nextIndex

[<Struct>]
type Particle = { X: float32; Y: float32; Active: bool }

let particlePool = StructPool<Particle>(100)
printfn "\nPool available: %d" particlePool.Available
```

---

## 7. where 'T : (new : unit -> 'T)

```fsharp
// default constructor constraint - 'T มี parameterless constructor

let createDefault<'T when 'T : (new : unit -> 'T)>() =
    new 'T()

let createArray<'T when 'T : (new : unit -> 'T)>(count: int) =
    Array.init count (fun _ -> new 'T())

// ทดสอบกับ classes ที่มี default constructor
type DefaultClass() =
    member val Value = 0 with get, set
    override this.ToString() = sprintf "DefaultClass(%d)" this.Value

let instance = createDefault<DefaultClass>()
printfn "Default: %s" (instance.ToString())

let instances = createArray<DefaultClass>(3)
for i, inst in List.indexed (Array.toList instances) do
    inst.Value <- i * 10
    printfn "  instances[%d]: %s" i (inst.ToString())

// Generic factory
let makeMany<'T when 'T : (new : unit -> 'T)> (count: int) (init: 'T -> unit) =
    let items = createArray<'T>(count)
    items |> Array.iter init
    items

type Counter3() =
    static let mutable nextId = 0
    let id = 
        nextId <- nextId + 1
        nextId
    member this.Id = id
    override this.ToString() = sprintf "Counter(%d)" id

let counters = makeMany<Counter3> 5 (fun _ -> ())  // init does nothing
printfn "\nCounters:"
counters |> Array.iter (fun c -> printfn "  %s" (c.ToString()))
```

---

## 8. where 'T :> SomeType

```fsharp
// Interface/base class constraint

type ISerializable =
    abstract member Serialize: unit -> string

type IProcessable =
    abstract member Process: unit -> unit

// Constraint: 'T ต้อง implement ISerializable
let serializeAll<'T when 'T :> ISerializable> (items: 'T list) =
    items |> List.map (fun item -> item.Serialize())

// Multiple interface constraints
let processAndSerialize<'T when 'T :> ISerializable and 'T :> IProcessable> (item: 'T) =
    item.Process()
    item.Serialize()

// Implementation
type Document(title: string, content: string) =
    interface ISerializable with
        member this.Serialize() = sprintf """{"title":"%s","content":"%s"}""" title content
    
    interface IProcessable with
        member this.Process() = printfn "Processing document: %s" title
    
    member this.Title = title

type Image(filename: string, width: int, height: int) =
    interface ISerializable with
        member this.Serialize() = sprintf """{"file":"%s","w":%d,"h":%d}""" filename width height

let docs = [Document("Doc 1", "Content 1"); Document("Doc 2", "Content 2")]
let serialized = serializeAll docs
printfn "Serialized documents:"
serialized |> List.iter (printfn "  %s")

let doc = Document("Test", "Hello World")
let result = processAndSerialize doc
printfn "Result: %s" result

// Base class constraint
[<AbstractClass>]
type Animal2(name: string) =
    abstract member Sound: string
    member this.Name = name

let makeSpeak<'T when 'T :> Animal2> (animal: 'T) =
    printfn "%s says %s" animal.Name (animal :> Animal2).Sound

type Dog2(name: string) =
    inherit Animal2(name)
    override this.Sound = "Woof"

type Cat2(name: string) =
    inherit Animal2(name)
    override this.Sound = "Meow"

makeSpeak (Dog2("Rex"))
makeSpeak (Cat2("Whiskers"))
```

---

## 9. Flexible Type Constraint #Type

```fsharp
// #Type constraint (flexible type) - 'T :> Type

// ใช้ # สำหรับ subtype constraint ใน parameters
let printShape (shape: #IShape) =
    printfn "Shape: area=%.2f, perimeter=%.2f" shape.Area shape.Perimeter

and IShape =
    abstract member Area: float
    abstract member Perimeter: float

// ตัวอย่าง sequence functions ที่ใช้ flexible types
let sumAreas (shapes: #IShape seq) =
    shapes |> Seq.sumBy (fun s -> s.Area)

let largestShape (shapes: #IShape list) =
    shapes |> List.maxBy (fun s -> s.Area)

// Implementations
type FlexCircle(r: float) =
    interface IShape with
        member this.Area = System.Math.PI * r * r
        member this.Perimeter = 2.0 * System.Math.PI * r

type FlexRect(w: float, h: float) =
    interface IShape with
        member this.Area = w * h
        member this.Perimeter = 2.0 * (w + h)

let shapes: IShape list = [
    FlexCircle(5.0) :> IShape
    FlexRect(4.0, 3.0) :> IShape
    FlexCircle(2.0) :> IShape
]

printfn "Total area: %.2f" (sumAreas shapes)
```

---

## 10. SRTP (Statically Resolved Type Parameters)

```fsharp
// SRTP - compile-time duck typing
// ใช้ ^ แทน ' สำหรับ SRTP, ต้องใช้ inline function

// Generic arithmetic ที่ใช้ SRTP
let inline add (a: ^T) (b: ^T) : ^T =
    a + b  // Works because ^T has + operator

let inline multiply (a: ^T) (b: ^T) : ^T =
    a * b

let inline negate (a: ^T) : ^T =
    -a

// ทดสอบ
printfn "add 1 2 = %d" (add 1 2)
printfn "add 1.5 2.5 = %.1f" (add 1.5 2.5)
printfn "add 'hello' ' world' = %s" (add "hello" " world")

// SRTP ที่ complex กว่า
let inline sumArray (arr: ^T array) : ^T =
    Array.fold (fun acc x -> acc + x) LanguagePrimitives.GenericZero arr

let intSum = sumArray [| 1; 2; 3; 4; 5 |]
let floatSum = sumArray [| 1.0; 2.0; 3.0 |]
printfn "Int sum: %d" intSum
printfn "Float sum: %.1f" floatSum

// SRTP member constraints
let inline getLength (x: ^T when ^T : (member Length: int)) =
    (^T : (member Length: int) x)

printfn "\nLength of 'hello': %d" (getLength "hello")
printfn "Length of [1;2;3]: %d" (getLength [1;2;3])
printfn "Length of [|1;2;3|]: %d" (getLength [|1;2;3|])

// SRTP for numeric operations
let inline square x = x * x
let inline cube x = x * x * x
let inline power (x: ^T) (n: int) : ^T =
    let mutable result = LanguagePrimitives.GenericOne
    for _ in 1..n do
        result <- result * x
    result

printfn "\n2^10 = %d" (power 2 10)
printfn "3.0^3 = %.1f" (power 3.0 3)
```

---

## 11. inline Functions with SRTP

```fsharp
// inline functions เป็นสิ่งจำเป็นสำหรับ SRTP

// Generic dot product
let inline dotProduct (a: ^T array) (b: ^T array) : ^T =
    Array.map2 (fun x y -> x * y) a b
    |> Array.sum

let intDot = dotProduct [|1;2;3|] [|4;5;6|]
let floatDot = dotProduct [|1.0;2.0;3.0|] [|4.0;5.0;6.0|]
printfn "Int dot: %d" intDot      // 32
printfn "Float dot: %.1f" floatDot  // 32.0

// Generic norm
let inline norm (v: ^T array) : float =
    v |> Array.sumBy (fun x -> float (x * x)) |> sqrt

printfn "Norm [3,4]: %.1f" (norm [| 3.0; 4.0 |])   // 5.0
printfn "Norm [1,1,1]: %.3f" (norm [| 1.0; 1.0; 1.0 |])

// Generic matrix operations
let inline matMul (a: ^T[,]) (b: ^T[,]) : ^T[,] =
    let rows = Array2D.length1 a
    let cols = Array2D.length2 b
    let inner = Array2D.length2 a
    Array2D.init rows cols (fun i j ->
        [0..inner-1] 
        |> List.sumBy (fun k -> a.[i,k] * b.[k,j])
        |> id
    )

let matA = array2D [[1.0; 2.0]; [3.0; 4.0]]
let matB = array2D [[5.0; 6.0]; [7.0; 8.0]]
let matC = matMul matA matB

printfn "\nMatrix multiplication:"
printfn "  [[1,2],[3,4]] * [[5,6],[7,8]]"
printfn "  = [[%.0f,%.0f],[%.0f,%.0f]]" matC.[0,0] matC.[0,1] matC.[1,0] matC.[1,1]

// SRTP for comparison
let inline clamp (value: ^T) (minVal: ^T) (maxVal: ^T) : ^T =
    if value < minVal then minVal
    elif value > maxVal then maxVal
    else value

printfn "\nclamp 15 0 10 = %d" (clamp 15 0 10)
printfn "clamp -5 0 10 = %d" (clamp -5 0 10)
printfn "clamp 5 0 10 = %d" (clamp 5 0 10)
printfn "clamp 3.14 0.0 2.0 = %.2f" (clamp 3.14 0.0 2.0)
```

---

## 12. Generic Interfaces

```fsharp
// Generic interface definitions

type IMapper<'TInput, 'TOutput> =
    abstract member Map: 'TInput -> 'TOutput

type IFilter<'T> =
    abstract member Filter: 'T -> bool

type IPipeline<'TInput, 'TOutput> =
    abstract member Execute: 'TInput -> 'TOutput

// Implementations
type StringToIntMapper() =
    interface IMapper<string, int> with
        member this.Map(s) = 
            match System.Int32.TryParse(s) with
            | true, n -> n
            | _ -> 0

type PositiveFilter() =
    interface IFilter<int> with
        member this.Filter(n) = n > 0

// Generic pipeline
type Pipeline<'TInput, 'TOutput>(stages: (obj -> obj) list) =
    interface IPipeline<'TInput, 'TOutput> with
        member this.Execute(input) =
            stages |> List.fold (fun acc stage -> stage acc) (box input) :?> 'TOutput

// Fluent pipeline builder
type PipelineBuilder<'T>() =
    let mutable stages: ('T -> 'T) list = []
    
    member this.AddStage(stage: 'T -> 'T) =
        stages <- stages @ [stage]
        this
    
    member this.Build() : 'T -> 'T =
        fun input -> stages |> List.fold (fun acc stage -> stage acc) input

// ทดสอบ
let mapper: IMapper<string, int> = StringToIntMapper() :> IMapper<string, int>
let filter: IFilter<int> = PositiveFilter() :> IFilter<int>

let nums = ["1"; "-2"; "3"; "-4"; "5"; "abc"]
let processed = 
    nums
    |> List.map (fun s -> mapper.Map(s))
    |> List.filter (fun n -> filter.Filter(n))

printfn "Processed: %A" processed

// Generic event system
type IEventSource<'T> =
    abstract member Subscribe: ('T -> unit) -> System.IDisposable
    abstract member Publish: 'T -> unit

type SimpleEventSource<'T>() =
    let handlers = System.Collections.Generic.List<'T -> unit>()
    
    interface IEventSource<'T> with
        member this.Subscribe(handler) =
            handlers.Add(handler)
            { new System.IDisposable with
                member this.Dispose() = handlers.Remove(handler) |> ignore }
        
        member this.Publish(event) =
            for handler in handlers do
                handler event

let intEvents: IEventSource<int> = SimpleEventSource<int>() :> IEventSource<int>
use sub1 = intEvents.Subscribe(fun n -> printfn "[Handler1] Got: %d" n)
use sub2 = intEvents.Subscribe(fun n -> printfn "[Handler2] Got*2: %d" (n*2))

intEvents.Publish(42)
intEvents.Publish(100)
```

---

## 13. Generic Algorithms

```fsharp
// Generic algorithms ที่ reusable

// Generic merge sort
let rec mergeSort<'T when 'T : comparison> (lst: 'T list) =
    let merge left right =
        let rec merge' left right acc =
            match left, right with
            | [], _ -> List.rev acc @ right
            | _, [] -> List.rev acc @ left
            | lh :: lt, rh :: rt ->
                if lh <= rh then merge' lt right (lh :: acc)
                else merge' left rt (rh :: acc)
        merge' left right []
    
    let rec split lst =
        let rec split' lst left right toggle =
            match lst with
            | [] -> left, right
            | x :: rest ->
                if toggle then split' rest (x :: left) right false
                else split' rest left (x :: right) true
        split' lst [] [] true
    
    match lst with
    | [] | [_] -> lst
    | _ ->
        let (left, right) = split lst
        merge (mergeSort left) (mergeSort right)

printfn "Merge sort [3;1;4;1;5;9;2;6;5]: %A" 
    (mergeSort [3;1;4;1;5;9;2;6;5])

// Generic topological sort
let topologicalSort<'T when 'T : equality> 
    (nodes: 'T list) 
    (edges: ('T * 'T) list) =
    
    let inDegree = 
        nodes 
        |> List.map (fun n ->
            let degree = edges |> List.filter (fun (_, b) -> b = n) |> List.length
            n, degree)
        |> Map.ofList
    
    let rec sort remaining inDegree result =
        let ready = remaining |> List.filter (fun n -> Map.find n inDegree = 0)
        match ready with
        | [] -> 
            if List.isEmpty remaining then List.rev result
            else failwith "Circular dependency detected"
        | next :: _ ->
            let newRemaining = remaining |> List.filter (fun n -> n <> next)
            let newInDegree = 
                edges 
                |> List.filter (fun (a, _) -> a = next)
                |> List.fold (fun deg (_, b) ->
                    Map.add b (Map.find b deg - 1) deg
                ) inDegree
            sort newRemaining newInDegree (next :: result)
    
    sort nodes inDegree []

let nodes = ["E"; "A"; "D"; "B"; "C"]
let edges = ["A","B"; "A","C"; "B","D"; "C","D"; "D","E"]
let sorted = topologicalSort nodes edges
printfn "Topological sort: %A" sorted

// Generic graph DFS
let dfs<'T when 'T : equality> 
    (start: 'T) 
    (neighbors: 'T -> 'T list) 
    (visit: 'T -> unit) =
    
    let visited = System.Collections.Generic.HashSet<'T>()
    
    let rec dfs' node =
        if not (visited.Contains(node)) then
            visited.Add(node) |> ignore
            visit node
            for neighbor in neighbors node do
                dfs' neighbor
    
    dfs' start

// ทดสอบ DFS
let graph = Map.ofList [
    "A", ["B"; "C"]
    "B", ["D"; "E"]
    "C", ["F"]
    "D", []
    "E", []
    "F", []
]

printfn "\nDFS from A:"
dfs "A" (fun n -> Map.tryFind n graph |> Option.defaultValue []) (printfn "  Visit: %s")
```

---

## 14. Covariance and Contravariance in .NET

```fsharp
// Covariance และ Contravariance ใน .NET interfaces

// Covariant: IEnumerable<out T> - producer
// ถ้า Dog :> Animal, แล้ว IEnumerable<Dog> :> IEnumerable<Animal>

let printAnimals (animals: seq<obj>) =
    for a in animals do
        printfn "  Animal: %A" a

// Covariant ทำงาน
let dogs: seq<string> = ["Rex"; "Buddy"; "Max"] :> seq<string>
// dogs :> seq<obj>  // This would fail in F# due to value types

// IReadOnlyList<out T> - covariant
let intList: System.Collections.Generic.IReadOnlyList<int> = [|1;2;3|] :> System.Collections.Generic.IReadOnlyList<int>

// Contravariant: Action<in T> - consumer
// ถ้า Dog :> Animal, แล้ว Action<Animal> :> Action<Dog>
let animalAction: System.Action<obj> = System.Action<obj>(fun a -> printfn "Animal: %A" a)

// F# functions ไม่ direct covariant/contravariant
// แต่ใช้ generic functions แทน

// Generic wrapper ที่ safe
type Producer<'T>(produce: unit -> 'T) =
    member this.Get() = produce()

type Consumer<'T>(consume: 'T -> unit) =
    member this.Accept(item: 'T) = consume item

// Variance ผ่าน generic functions
let mapProducer<'T, 'U> (f: 'T -> 'U) (producer: Producer<'T>) =
    Producer<'U>(fun () -> f (producer.Get()))

let mapConsumer<'T, 'U> (f: 'U -> 'T) (consumer: Consumer<'T>) =
    Consumer<'U>(fun u -> consumer.Accept(f u))

let intProducer = Producer<int>(fun () -> 42)
let strProducer = mapProducer string intProducer
printfn "Produced: %s" (strProducer.Get())

let printConsumer = Consumer<string>(printfn "Consuming: %s")
let intConsumer = mapConsumer string printConsumer
intConsumer.Accept(100)
```

---

## 15. Advanced Generic Patterns

```fsharp
// Generic monad-like operations

// Reader monad (ผ่าน generic functions)
type Reader<'Env, 'T> = Reader of ('Env -> 'T)

module Reader =
    let run (Reader f) env = f env
    let pure' value = Reader (fun _ -> value)
    let ask = Reader id
    let asks f = Reader f
    let map f (Reader r) = Reader (fun env -> f (r env))
    let bind f (Reader r) = Reader (fun env -> run (f (r env)) env)

// ทดสอบ Reader
type AppConfig = { DbConnectionString: string; MaxRetries: int }

let getDbString = Reader.asks (fun (c: AppConfig) -> c.DbConnectionString)
let getMaxRetries = Reader.asks (fun (c: AppConfig) -> c.MaxRetries)

let buildQuery tableName =
    Reader.asks (fun (config: AppConfig) ->
        sprintf "SELECT * FROM %s [MaxRetries=%d]" tableName config.MaxRetries
    )

let config = { DbConnectionString = "Server=localhost"; MaxRetries = 3 }
printfn "DB: %s" (Reader.run getDbString config)
printfn "Retries: %d" (Reader.run getMaxRetries config)
printfn "Query: %s" (Reader.run (buildQuery "users") config)

// Generic validation
type Validation<'T> =
    | Valid of 'T
    | Invalid of string list

module Validation =
    let pure' value = Valid value
    
    let map f = function
        | Valid v -> Valid (f v)
        | Invalid errors -> Invalid errors
    
    let apply vf va =
        match vf, va with
        | Valid f, Valid a -> Valid (f a)
        | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
        | Invalid e, _ | _, Invalid e -> Invalid e
    
    let bind f = function
        | Valid v -> f v
        | Invalid errors -> Invalid errors

// Form validation
type RegistrationForm = { Username: string; Email: string; Age: int }

let validateUsername name =
    if System.String.IsNullOrEmpty(name) then Invalid ["Username cannot be empty"]
    elif name.Length < 3 then Invalid ["Username must be at least 3 characters"]
    else Valid name

let validateEmail2 email =
    if System.String.IsNullOrEmpty(email) then Invalid ["Email cannot be empty"]
    elif not (email.Contains("@")) then Invalid ["Email must contain @"]
    else Valid email

let validateAge age =
    if age < 0 then Invalid ["Age cannot be negative"]
    elif age > 150 then Invalid ["Age seems unrealistic"]
    else Valid age

let validateForm username email age =
    let usernameResult = validateUsername username
    let emailResult = validateEmail2 email
    let ageResult = validateAge age
    
    match usernameResult, emailResult, ageResult with
    | Valid u, Valid e, Valid a -> Valid { Username = u; Email = e; Age = a }
    | _ ->
        let errors = 
            [usernameResult; emailResult; ageResult]
            |> List.collect (function
                | Invalid e -> e
                | Valid _ -> []
            )
        Invalid errors

let form1 = validateForm "john" "john@example.com" 25
let form2 = validateForm "" "invalid" -5

match form1 with
| Valid f -> printfn "\nValid form: %s, %s, %d" f.Username f.Email f.Age
| Invalid errors -> printfn "Invalid: %A" errors

match form2 with
| Valid f -> printfn "Valid form: %s" f.Username
| Invalid errors -> 
    printfn "Invalid form errors:"
    errors |> List.iter (printfn "  - %s")
```

---

## สรุป (Summary)

```fsharp
printfn "=== Generics Summary ==="
printfn ""
printfn "Generic function:"
printfn "  let f<'T> (x: 'T) = ..."
printfn "  let f x = ... // often inferred"
printfn ""
printfn "Type constraints:"
printfn "  'T : equality  -> can use ="
printfn "  'T : comparison -> can use < >"
printfn "  'T : struct -> value type"
printfn "  'T : (new : unit -> 'T) -> default ctor"
printfn "  'T :> SomeType -> subtype"
printfn ""
printfn "SRTP (Statically Resolved Type Parameters):"
printfn "  let inline f (x: ^T) = x + x  // requires inline"
printfn "  Works with operator constraints (+, -, *, /)"
printfn ""
printfn "Generic types:"
printfn "  type Container<'T> = ..."
printfn "  type Tree<'T> = Leaf | Node of 'T * Tree<'T> * Tree<'T>"
```

---

## บทสรุป

Generics ใน F# ช่วยให้:
1. **Code reuse** - เขียน algorithm ครั้งเดียว ใช้กับหลาย types
2. **Type safety** - compile-time type checking
3. **Performance** - SRTP ไม่มี boxing overhead
4. **Abstraction** - type constraints กำหนด capabilities
5. **Functional patterns** - generic monads, functors, applicatives
