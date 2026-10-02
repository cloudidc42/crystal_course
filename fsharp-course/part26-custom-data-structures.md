# Part 26 - โครงสร้างข้อมูลที่กำหนดเอง (Custom Data Structures)

## บทนำ (Introduction)

การสร้างโครงสร้างข้อมูลเองให้เหมาะสมกับปัญหาเฉพาะทางเป็นทักษะสำคัญ บท นี้จะสอนการสร้าง custom data structures ใน F# ตั้งแต่ linked lists ไปจนถึง complex structures

---

## 26.1 Custom Linked List

```fsharp
// ============ Singly Linked List ============
type LinkedList<'T> =
    | Empty
    | Node of value: 'T * next: LinkedList<'T>

module LinkedList =
    // สร้าง list จาก list F#
    let ofList lst = List.foldBack (fun x acc -> Node(x, acc)) lst Empty
    
    let toList lst =
        let rec aux lst acc =
            match lst with
            | Empty -> List.rev acc
            | Node(v, next) -> aux next (v :: acc)
        aux lst []
    
    let length lst =
        let rec aux lst n =
            match lst with
            | Empty -> n
            | Node(_, next) -> aux next (n + 1)
        aux lst 0
    
    let prepend value lst = Node(value, lst)
    
    let head = function
        | Empty -> failwith "Empty list"
        | Node(v, _) -> v
    
    let tail = function
        | Empty -> failwith "Empty list"
        | Node(_, next) -> next
    
    let isEmpty = function Empty -> true | _ -> false
    
    let rec append v = function
        | Empty -> Node(v, Empty)
        | Node(head, next) -> Node(head, append v next)
    
    let rec map f = function
        | Empty -> Empty
        | Node(v, next) -> Node(f v, map f next)
    
    let rec filter pred = function
        | Empty -> Empty
        | Node(v, next) ->
            if pred v then Node(v, filter pred next)
            else filter pred next
    
    let rec fold f acc = function
        | Empty -> acc
        | Node(v, next) -> fold f (f acc v) next
    
    let rec reverse = function
        | Empty -> Empty
        | Node(v, next) -> append v (reverse next)
    
    // tail-recursive reverse
    let reverseFast lst =
        let rec aux lst acc =
            match lst with
            | Empty -> acc
            | Node(v, next) -> aux next (Node(v, acc))
        aux lst Empty
    
    let rec find pred = function
        | Empty -> None
        | Node(v, next) -> if pred v then Some v else find pred next
    
    let rec exists pred = function
        | Empty -> false
        | Node(v, next) -> pred v || exists pred next
    
    let rec nth lst n =
        match lst, n with
        | Empty, _ -> failwith "Index out of range"
        | Node(v, _), 0 -> v
        | Node(_, next), n -> nth next (n - 1)

// ============ ใช้งาน ============
let ll = LinkedList.ofList [1; 2; 3; 4; 5]
printfn "Length: %d" (LinkedList.length ll)     // 5
printfn "Head: %d" (LinkedList.head ll)          // 1
printfn "Tail: %A" (LinkedList.toList (LinkedList.tail ll))  // [2;3;4;5]

let doubled = LinkedList.map ((*) 2) ll
printfn "Doubled: %A" (LinkedList.toList doubled)

let evens = LinkedList.filter (fun x -> x % 2 = 0) ll
printfn "Evens: %A" (LinkedList.toList evens)

let sum = LinkedList.fold (+) 0 ll
printfn "Sum: %d" sum

let reversed = LinkedList.reverseFast ll
printfn "Reversed: %A" (LinkedList.toList reversed)

printfn "3rd element: %d" (LinkedList.nth ll 2)  // 3
```

---

## 26.2 Doubly Linked List

```fsharp
open System.Collections.Generic

// ============ Doubly Linked List (mutable) ============
type DLLNode<'T>(value: 'T) =
    member val Value = value with get, set
    member val Prev: DLLNode<'T> option = None with get, set
    member val Next: DLLNode<'T> option = None with get, set

type DoublyLinkedList<'T>() =
    let mutable head: DLLNode<'T> option = None
    let mutable tail: DLLNode<'T> option = None
    let mutable count = 0
    
    member _.Count = count
    member _.Head = head
    member _.Tail = tail
    
    member _.AddFirst(value: 'T) =
        let node = DLLNode(value)
        match head with
        | None ->
            head <- Some node
            tail <- Some node
        | Some h ->
            node.Next <- Some h
            h.Prev <- Some node
            head <- Some node
        count <- count + 1
        node
    
    member _.AddLast(value: 'T) =
        let node = DLLNode(value)
        match tail with
        | None ->
            head <- Some node
            tail <- Some node
        | Some t ->
            t.Next <- Some node
            node.Prev <- Some t
            tail <- Some node
        count <- count + 1
        node
    
    member _.AddBefore(refNode: DLLNode<'T>, value: 'T) =
        let node = DLLNode(value)
        node.Next <- Some refNode
        node.Prev <- refNode.Prev
        match refNode.Prev with
        | Some prev -> prev.Next <- Some node
        | None -> head <- Some node
        refNode.Prev <- Some node
        count <- count + 1
        node
    
    member _.RemoveFirst() =
        match head with
        | None -> failwith "List is empty"
        | Some h ->
            match h.Next with
            | None -> head <- None; tail <- None
            | Some next -> next.Prev <- None; head <- Some next
            count <- count - 1
            h.Value
    
    member _.RemoveLast() =
        match tail with
        | None -> failwith "List is empty"
        | Some t ->
            match t.Prev with
            | None -> head <- None; tail <- None
            | Some prev -> prev.Next <- None; tail <- Some prev
            count <- count - 1
            t.Value
    
    member _.Remove(node: DLLNode<'T>) =
        match node.Prev with
        | Some prev -> prev.Next <- node.Next
        | None -> head <- node.Next
        match node.Next with
        | Some next -> next.Prev <- node.Prev
        | None -> tail <- node.Prev
        count <- count - 1
    
    member _.ToList() =
        let result = ResizeArray<'T>()
        let mutable current = head
        while current.IsSome do
            result.Add(current.Value.Value)
            current <- current.Value.Next
        result |> Seq.toList
    
    member _.ToListReverse() =
        let result = ResizeArray<'T>()
        let mutable current = tail
        while current.IsSome do
            result.Add(current.Value.Value)
            current <- current.Value.Prev
        result |> Seq.toList

// ============ ใช้งาน ============
let dll = DoublyLinkedList<int>()
dll.AddLast(1) |> ignore
dll.AddLast(2) |> ignore
dll.AddLast(3) |> ignore
dll.AddFirst(0) |> ignore

printfn "DLL forward: %A" (dll.ToList())    // [0; 1; 2; 3]
printfn "DLL reverse: %A" (dll.ToListReverse())  // [3; 2; 1; 0]

let removed = dll.RemoveFirst()
printfn "Removed first: %d" removed   // 0
printfn "After remove: %A" (dll.ToList())   // [1; 2; 3]

let removedLast = dll.RemoveLast()
printfn "Removed last: %d" removedLast  // 3
```

---

## 26.3 Circular Buffer

```fsharp
// ============ Circular Buffer (immutable functional) ============
type CircularBuffer<'T> private (data: 'T[], head: int, count: int) =
    let capacity = data.Length
    
    static member Create(capacity: int) =
        CircularBuffer<'T>(Array.create capacity Unchecked.defaultof<'T>, 0, 0)
    
    member _.Capacity = capacity
    member _.Count = count
    member _.IsEmpty = count = 0
    member _.IsFull = count = capacity
    
    member _.Enqueue(value: 'T) : CircularBuffer<'T> =
        let newData = Array.copy data
        let tail = (head + count) % capacity
        newData.[tail] <- value
        if count < capacity then
            CircularBuffer(newData, head, count + 1)
        else
            // Overwrite oldest
            CircularBuffer(newData, (head + 1) % capacity, count)
    
    member _.Dequeue() : 'T * CircularBuffer<'T> =
        if count = 0 then failwith "Buffer empty"
        let value = data.[head]
        (value, CircularBuffer(data, (head + 1) % capacity, count - 1))
    
    member _.Peek() =
        if count = 0 then failwith "Buffer empty"
        data.[head]
    
    member _.ToArray() =
        [| for i in 0..count - 1 -> data.[(head + i) % capacity] |]
    
    member _.ToList() = 
        [ for i in 0..count - 1 -> data.[(head + i) % capacity] ]

// ============ ใช้งาน ============
let cb = CircularBuffer<int>.Create(5)

// เพิ่ม elements
let cb1 = cb.Enqueue(1).Enqueue(2).Enqueue(3).Enqueue(4).Enqueue(5)
printfn "Full buffer: %A" (cb1.ToList())  // [1; 2; 3; 4; 5]

// เพิ่มเมื่อเต็ม (เขียนทับเก่า)
let cb2 = cb1.Enqueue(6).Enqueue(7)
printfn "After overflow: %A" (cb2.ToList())  // [3; 4; 5; 6; 7]

// Dequeue
let (v, cb3) = cb2.Dequeue()
printfn "Dequeued: %d, remaining: %A" v (cb3.ToList())

// ============ Fixed-size Log Buffer ============
type LogBuffer(capacity: int) =
    let mutable buffer = CircularBuffer<string>.Create(capacity)
    
    member _.Log(message: string) =
        buffer <- buffer.Enqueue(message)
    
    member _.Recent(n: int) =
        buffer.ToList() |> List.rev |> List.take (min n buffer.Count)
    
    member _.All() = buffer.ToList()

let log = LogBuffer(5)
for i in 1..8 do
    log.Log($"Event {i}")

printfn "Recent 3 events: %A" (log.Recent(3))
printfn "All stored events: %A" (log.All())
```

---

## 26.4 Persistent Stack

```fsharp
// ============ Persistent Stack ============
// Stack ที่เก็บ history ทุก version

type PersistentStack<'T> =
    | PSEmpty
    | PSNode of value: 'T * tail: PersistentStack<'T>

module PersistentStack =
    let empty = PSEmpty
    
    let isEmpty = function PSEmpty -> true | _ -> false
    
    let push value stack = PSNode(value, stack)
    
    let pop = function
        | PSEmpty -> failwith "Empty stack"
        | PSNode(v, tail) -> (v, tail)
    
    let peek = function
        | PSEmpty -> failwith "Empty stack"
        | PSNode(v, _) -> v
    
    let size stack =
        let rec aux stack n =
            match stack with
            | PSEmpty -> n
            | PSNode(_, tail) -> aux tail (n + 1)
        aux stack 0
    
    let toList stack =
        let rec aux stack acc =
            match stack with
            | PSEmpty -> acc
            | PSNode(v, tail) -> aux tail (v :: acc)
        aux stack [] |> List.rev

// Versioned stack ที่เก็บ snapshots
type VersionedStack<'T>() =
    let mutable versions: PersistentStack<'T> list = [PersistentStack.empty]
    let mutable currentVersion = 0
    
    member _.Current = versions.[currentVersion]
    
    member _.Push(value: 'T) =
        let newStack = PersistentStack.push value versions.[currentVersion]
        versions <- versions @ [newStack]
        currentVersion <- versions.Length - 1
    
    member _.Pop() =
        let (v, newStack) = PersistentStack.pop versions.[currentVersion]
        versions <- versions @ [newStack]
        currentVersion <- versions.Length - 1
        v
    
    member _.Undo() =
        if currentVersion > 0 then
            currentVersion <- currentVersion - 1
    
    member _.Redo() =
        if currentVersion < versions.Length - 1 then
            currentVersion <- currentVersion + 1
    
    member _.Snapshot() = currentVersion
    
    member _.RestoreSnapshot(snapshot: int) =
        if snapshot >= 0 && snapshot < versions.Length then
            currentVersion <- snapshot
    
    member _.ToList() = PersistentStack.toList versions.[currentVersion]

// ============ ใช้งาน ============
let vs = VersionedStack<int>()
vs.Push(1)
vs.Push(2)
vs.Push(3)
let snapshot1 = vs.Snapshot()
printfn "After 3 pushes: %A" (vs.ToList())  // [3; 2; 1]

vs.Push(4)
vs.Push(5)
printfn "After 2 more pushes: %A" (vs.ToList())  // [5; 4; 3; 2; 1]

vs.Undo()
vs.Undo()
printfn "After 2 undos: %A" (vs.ToList())  // [3; 2; 1]

vs.Redo()
printfn "After 1 redo: %A" (vs.ToList())  // [4; 3; 2; 1]

vs.RestoreSnapshot(snapshot1)
printfn "After restore snapshot: %A" (vs.ToList())  // [3; 2; 1]
```

---

## 26.5 Difference Lists

```fsharp
// ============ Difference Lists ============
// Efficient list concatenation: O(1) append, O(n) toList
// ใช้แทน list concatenation ที่ O(n) ปกติ

// A difference list is a function: list -> list
type DList<'T> = DList of ('T list -> 'T list)

module DList =
    let empty = DList id
    
    let singleton x = DList (fun tail -> x :: tail)
    
    let ofList lst = DList (fun tail -> lst @ tail)
    
    let toList (DList f) = f []
    
    // O(1) append!
    let append (DList f) (DList g) = DList (fun tail -> f (g tail))
    
    let cons x (DList f) = DList (fun tail -> x :: f tail)
    
    let snoc (DList f) x = DList (fun tail -> f (x :: tail))
    
    let length dl = List.length (toList dl)
    
    // map
    let map f (DList g) =
        DList (fun tail -> List.map f (g []) @ tail)

// ============ ใช้งาน ============
let dl1 = DList.ofList [1; 2; 3]
let dl2 = DList.ofList [4; 5; 6]
let dl3 = DList.ofList [7; 8; 9]

// O(1) concatenation
let combined = dl1 |> DList.append dl2 |> DList.append dl3
printfn "Combined: %A" (DList.toList combined)  // [1;2;3;4;5;6;7;8;9]

// Performance comparison: Building a string
let buildWithDList (n: int) =
    let dl = List.init n (fun i -> DList.singleton i)
    dl |> List.fold DList.append DList.empty |> DList.toList

let buildWithList (n: int) =
    List.init n id |> List.rev

let sw = System.Diagnostics.Stopwatch.StartNew()
let _ = buildWithDList 1000
sw.Stop()
printfn "DList build 1000: %d ms" sw.ElapsedMilliseconds

// ============ ตัวอย่าง: Pretty Printer ============
type Doc = DList<string>

let text s = DList.singleton s
let line = DList.singleton "\n"
let space = DList.singleton " "
let (<+>) a b = DList.append a (DList.append space b)
let (<.>) a b = DList.append a (DList.append line b)

let render doc = DList.toList doc |> String.concat ""

let sampleDoc = 
    text "Hello" <+> text "World" <.>
    text "This" <+> text "is" <+> text "F#" <.>
    text "Programming"

printfn "Rendered:\n%s" (render sampleDoc)
```

---

## 26.6 Association Lists

```fsharp
// ============ Association Lists ============
// Simpler alternative to Map for small collections
// List of key-value pairs

type AssocList<'K, 'V> = ('K * 'V) list

module AssocList =
    let empty : AssocList<'K, 'V> = []
    
    let add key value (lst: AssocList<'K, 'V>) : AssocList<'K, 'V> =
        (key, value) :: (lst |> List.filter (fun (k, _) -> k <> key))
    
    let remove key (lst: AssocList<'K, 'V>) : AssocList<'K, 'V> =
        lst |> List.filter (fun (k, _) -> k <> key)
    
    let find key (lst: AssocList<'K, 'V>) : 'V =
        lst |> List.find (fun (k, _) -> k = key) |> snd
    
    let tryFind key (lst: AssocList<'K, 'V>) : 'V option =
        lst |> List.tryFind (fun (k, _) -> k = key) |> Option.map snd
    
    let containsKey key (lst: AssocList<'K, 'V>) : bool =
        lst |> List.exists (fun (k, _) -> k = key)
    
    let keys (lst: AssocList<'K, 'V>) = lst |> List.map fst
    
    let values (lst: AssocList<'K, 'V>) = lst |> List.map snd
    
    let map f (lst: AssocList<'K, 'V>) : AssocList<'K, 'U> =
        lst |> List.map (fun (k, v) -> (k, f k v))
    
    let filter pred (lst: AssocList<'K, 'V>) : AssocList<'K, 'V> =
        lst |> List.filter (fun (k, v) -> pred k v)
    
    let merge (lst1: AssocList<'K, 'V>) (lst2: AssocList<'K, 'V>) =
        lst2 |> List.fold (fun acc (k, v) -> add k v acc) lst1
    
    let size = List.length
    
    let toMap (lst: AssocList<'K, 'V>) = Map.ofList lst
    
    let ofMap (m: Map<'K, 'V>) = Map.toList m

// ============ ใช้งาน ============
let al = 
    AssocList.empty
    |> AssocList.add "name" "Alice"
    |> AssocList.add "age" "30"
    |> AssocList.add "city" "Bangkok"

printfn "name: %A" (AssocList.tryFind "name" al)  // Some "Alice"
printfn "email: %A" (AssocList.tryFind "email" al)  // None

let al2 = AssocList.add "name" "Bob" al   // update
printfn "Updated name: %s" (AssocList.find "name" al2)

let al3 = AssocList.remove "age" al
printfn "After remove age: %A" al3

// ============ ตัวอย่าง: HTTP Headers ============
type Headers = AssocList<string, string>

let parseHeaders (lines: string list) : Headers =
    lines 
    |> List.choose (fun line ->
        match line.Split([|':'|], 2) with
        | [| key; value |] -> Some (key.Trim(), value.Trim())
        | _ -> None)

let responseHeaders = parseHeaders [
    "Content-Type: application/json"
    "Content-Length: 256"
    "Cache-Control: no-cache"
    "X-Request-Id: abc123"
]

printfn "Content-Type: %A" (AssocList.tryFind "Content-Type" responseHeaders)
printfn "All headers: %A" responseHeaders
```

---

## 26.7 Multimap

```fsharp
// ============ Multimap ============
// Map ที่ key หนึ่งสามารถมีหลาย values

type Multimap<'K, 'V when 'K : comparison> = Map<'K, 'V list>

module Multimap =
    let empty : Multimap<'K, 'V> = Map.empty
    
    let add key value (m: Multimap<'K, 'V>) : Multimap<'K, 'V> =
        let existing = m |> Map.tryFind key |> Option.defaultValue []
        Map.add key (value :: existing) m
    
    let addAll key values (m: Multimap<'K, 'V>) : Multimap<'K, 'V> =
        values |> List.fold (fun acc v -> add key v acc) m
    
    let remove key value (m: Multimap<'K, 'V>) : Multimap<'K, 'V> =
        match Map.tryFind key m with
        | None -> m
        | Some values ->
            let filtered = List.filter ((<>) value) values
            if filtered.IsEmpty then Map.remove key m
            else Map.add key filtered m
    
    let removeAll key (m: Multimap<'K, 'V>) : Multimap<'K, 'V> =
        Map.remove key m
    
    let get key (m: Multimap<'K, 'V>) : 'V list =
        m |> Map.tryFind key |> Option.defaultValue []
    
    let containsKey key = Map.containsKey key
    
    let containsValue key value (m: Multimap<'K, 'V>) =
        m |> Map.tryFind key 
        |> Option.map (List.contains value) 
        |> Option.defaultValue false
    
    let keys (m: Multimap<'K, 'V>) = m |> Map.keys
    
    let allValues (m: Multimap<'K, 'V>) =
        m |> Map.values |> Seq.collect id |> Seq.toList
    
    let count (m: Multimap<'K, 'V>) =
        m |> Map.fold (fun acc _ vs -> acc + List.length vs) 0
    
    let map f (m: Multimap<'K, 'V>) : Multimap<'K, 'U> =
        m |> Map.map (fun _ vs -> List.map f vs)
    
    let filter pred (m: Multimap<'K, 'V>) : Multimap<'K, 'V> =
        m 
        |> Map.map (fun _ vs -> List.filter pred vs)
        |> Map.filter (fun _ vs -> not vs.IsEmpty)
    
    let ofList (pairs: ('K * 'V) list) : Multimap<'K, 'V> =
        pairs |> List.fold (fun m (k, v) -> add k v m) empty
    
    let toList (m: Multimap<'K, 'V>) : ('K * 'V) list =
        m |> Map.toList |> List.collect (fun (k, vs) -> List.map (fun v -> (k, v)) vs)

// ============ ใช้งาน ============
let mm = Multimap.ofList [
    ("fruits", "apple")
    ("fruits", "banana")
    ("fruits", "cherry")
    ("veggies", "carrot")
    ("veggies", "broccoli")
    ("grains", "rice")
]

printfn "fruits: %A" (Multimap.get "fruits" mm)
printfn "total items: %d" (Multimap.count mm)
printfn "all values: %A" (Multimap.allValues mm)

let mm2 = Multimap.remove "fruits" "banana" mm
printfn "after remove banana: %A" (Multimap.get "fruits" mm2)

// ============ ตัวอย่าง: Index ============
type SearchIndex = Multimap<string, string>

let buildIndex (documents: Map<string, string>) : SearchIndex =
    documents |> Map.fold (fun idx docId content ->
        content.ToLower().Split([|' '; ','; '.'; '!'; '?'|], System.StringSplitOptions.RemoveEmptyEntries)
        |> Array.fold (fun idx word -> 
            Multimap.add word docId idx
        ) idx
    ) Multimap.empty

let docs = Map.ofList [
    ("doc1", "F# is a functional programming language")
    ("doc2", "F# supports object-oriented programming too")
    ("doc3", "Functional programming is fun")
]

let index = buildIndex docs

printfn "\nSearch index:"
printfn "  'programming' -> %A" (Multimap.get "programming" index)
printfn "  'functional' -> %A" (Multimap.get "functional" index)
printfn "  'f#' -> %A" (Multimap.get "f#" index)
```

---

## 26.8 Interval Tree

```fsharp
// ============ Interval Tree ============
// ค้นหา intervals ที่ overlap กับ point หรือ interval ที่กำหนด

type Interval = { Low: float; High: float }

let overlaps (a: Interval) (b: Interval) = a.Low <= b.High && b.Low <= a.High

let contains (interval: Interval) (point: float) =
    point >= interval.Low && point <= interval.High

type IntervalNode = {
    Interval: Interval
    Max: float      // maximum high value ใน subtree
    Left: IntervalTree
    Right: IntervalTree
}
and IntervalTree =
    | ITEmpty
    | ITNode of IntervalNode

let itMax = function
    | ITEmpty -> System.Double.NegativeInfinity
    | ITNode n -> n.Max

let itInsert (interval: Interval) (tree: IntervalTree) =
    let rec insert = function
        | ITEmpty ->
            ITNode { 
                Interval = interval
                Max = interval.High
                Left = ITEmpty
                Right = ITEmpty 
            }
        | ITNode node ->
            if interval.Low < node.Interval.Low then
                let newLeft = insert node.Left
                ITNode { node with 
                    Left = newLeft
                    Max = max node.Interval.High (max (itMax newLeft) (itMax node.Right)) }
            else
                let newRight = insert node.Right
                ITNode { node with 
                    Right = newRight
                    Max = max node.Interval.High (max (itMax node.Left) (itMax newRight)) }
    insert tree

// ค้นหา interval ที่ overlap กับ query
let rec itSearch (query: Interval) = function
    | ITEmpty -> []
    | ITNode node ->
        let leftResults =
            match node.Left with
            | ITEmpty -> []
            | ITNode leftNode ->
                if leftNode.Max >= query.Low then itSearch query node.Left
                else []
        
        let current = if overlaps node.Interval query then [node.Interval] else []
        
        leftResults @ current @ itSearch query node.Right

// ============ ใช้งาน ============
let intervals = [
    { Low = 15.0; High = 20.0 }
    { Low = 10.0; High = 30.0 }
    { Low = 17.0; High = 19.0 }
    { Low = 5.0; High = 20.0 }
    { Low = 12.0; High = 15.0 }
    { Low = 30.0; High = 40.0 }
]

let itree = List.fold (fun t i -> itInsert i t) ITEmpty intervals

let query = { Low = 14.0; High = 16.0 }
let overlapping = itSearch query itree

printfn "Intervals overlapping [14, 16]:"
overlapping |> List.iter (fun i -> printfn "  [%.1f, %.1f]" i.Low i.High)

// ============ ตัวอย่าง: Calendar Scheduling ============
type Meeting = { 
    Title: string
    Start: float  // hours from midnight
    End: float 
}

let meetings = [
    { Title = "Standup"; Start = 9.0; End = 9.5 }
    { Title = "Design Review"; Start = 10.0; End = 11.0 }
    { Title = "Lunch"; Start = 12.0; End = 13.0 }
    { Title = "Sprint Planning"; Start = 14.0; End = 16.0 }
    { Title = "1:1"; Start = 10.5; End = 11.5 }
]

let meetingIntervals = meetings |> List.map (fun m -> { Low = m.Start; High = m.End })
let calendarTree = List.fold (fun t i -> itInsert i t) ITEmpty meetingIntervals

let checkConflict (start: float) (end_: float) =
    let query = { Low = start; High = end_ }
    itSearch query calendarTree

let conflicts = checkConflict 10.0 11.5
printfn "\nConflicts with [10:00-11:30]:"
conflicts |> List.iter (fun i -> printfn "  [%.1f, %.1f]" i.Low i.High)
```

---

## 26.9 Custom Collection Interface

```fsharp
open System.Collections.Generic

// ============ ICollection Interface ============
[<Interface>]
type IImmutableCollection<'T> =
    abstract Count: int
    abstract Contains: 'T -> bool
    abstract ToSeq: unit -> 'T seq

// ============ Bag (Multiset) ============
// Collection ที่เก็บ elements พร้อมความถี่

type Bag<'T when 'T : comparison>(counts: Map<'T, int>) =
    
    interface IImmutableCollection<'T> with
        member _.Count = counts |> Map.fold (fun acc _ n -> acc + n) 0
        member _.Contains(x) = Map.containsKey x counts
        member _.ToSeq() = 
            counts |> Map.toSeq |> Seq.collect (fun (k, n) -> Seq.replicate n k)
    
    member _.Counts = counts
    
    member _.Add(item: 'T, ?times: int) =
        let n = Option.defaultValue 1 times
        let existing = counts |> Map.tryFind item |> Option.defaultValue 0
        Bag(Map.add item (existing + n) counts)
    
    member _.Remove(item: 'T, ?times: int) =
        let n = Option.defaultValue 1 times
        match Map.tryFind item counts with
        | None -> Bag(counts)
        | Some existing ->
            let newCount = existing - n
            if newCount <= 0 then Bag(Map.remove item counts)
            else Bag(Map.add item newCount counts)
    
    member _.GetCount(item: 'T) =
        counts |> Map.tryFind item |> Option.defaultValue 0
    
    member _.Union(other: Bag<'T>) =
        other.Counts |> Map.fold (fun (b: Bag<'T>) k n -> b.Add(k, n)) (Bag(counts))
    
    member _.Intersection(other: Bag<'T>) =
        let newCounts = 
            counts |> Map.choose (fun k n ->
                match Map.tryFind k other.Counts with
                | Some n2 -> Some (min n n2)
                | None -> None)
        Bag(newCounts)
    
    static member Empty = Bag<'T>(Map.empty)
    static member OfList(lst: 'T list) =
        lst |> List.fold (fun (b: Bag<'T>) x -> b.Add(x)) Bag.Empty

// ============ ใช้งาน ============
let bag = Bag.OfList [1; 2; 3; 2; 1; 3; 3; 4]

printfn "Count of 3: %d" (bag.GetCount(3))   // 3
printfn "Count of 5: %d" (bag.GetCount(5))   // 0
printfn "Total: %d" ((bag :> IImmutableCollection<int>).Count)  // 8

let bag2 = bag.Remove(3)   // remove one 3
printfn "After remove one 3: %d" (bag2.GetCount(3))   // 2

let bag3 = Bag.OfList [2; 3; 5; 6]
let intersection = bag.Intersection(bag3)
printfn "Intersection: %A" (intersection.Counts)
// map [(2, 1); (3, 1)]

// ============ Word frequency as Bag ============
let wordBag = 
    "the quick brown fox jumps over the lazy dog the fox"
    |> (fun s -> s.Split(' '))
    |> Array.toList
    |> Bag.OfList

printfn "\nWord frequencies:"
wordBag.Counts |> Map.toList |> List.sortByDescending snd 
|> List.iter (fun (w, n) -> printfn "  %s: %d" w n)
```

---

## 26.10 Implementing IEnumerable

```fsharp
open System.Collections
open System.Collections.Generic

// ============ Custom Sequence ============
// สร้าง type ที่ implement IEnumerable

type NumberRange(start: int, stop: int, step: int) =
    
    interface IEnumerable<int> with
        member _.GetEnumerator() : IEnumerator<int> =
            let mutable current = start - step
            { new IEnumerator<int> with
                member _.Current = current
                member _.MoveNext() =
                    current <- current + step
                    (step > 0 && current <= stop) || (step < 0 && current >= stop)
                member _.Reset() = current <- start - step
                member _.Dispose() = ()
              interface IEnumerator with
                member this.Current = box (this :> IEnumerator<int>).Current
                member this.MoveNext() = (this :> IEnumerator<int>).MoveNext()
                member this.Reset() = (this :> IEnumerator<int>).Reset() }
    
    interface IEnumerable with
        member this.GetEnumerator() = (this :> IEnumerable<int>).GetEnumerator() :> IEnumerator

// ============ Lazy Tree Traversal ============
type InfiniteTree =
    | Leaf of int
    | Branch of InfiniteTree * int * InfiniteTree

// Generate infinite tree lazily
let rec buildInfiniteTree n = 
    Branch(buildInfiniteTree (2 * n), n, buildInfiniteTree (2 * n + 1))

// Lazy in-order traversal using sequence
let rec lazyInorder tree =
    seq {
        match tree with
        | Leaf n -> yield n
        | Branch(left, n, right) ->
            yield! lazyInorder left
            yield n
            yield! lazyInorder right
    }

// ============ ใช้งาน ============
let range = NumberRange(1, 10, 2)
printfn "NumberRange 1 to 10 step 2:"
for n in range do
    printf "%d " n
printfn ""

// ใช้กับ LINQ/Seq
let doubled = range |> Seq.map ((*) 2) |> Seq.toList
printfn "Doubled: %A" doubled

// ============ Lazy Fibonacci Sequence ============
let fibonacci =
    Seq.unfold (fun (a, b) -> Some(a, (b, a + b))) (0, 1)

printfn "First 15 Fibonacci:"
fibonacci |> Seq.take 15 |> Seq.toList |> printfn "%A"

// ============ Custom Sorted Collection ============
type SortedList<'T when 'T : comparison>(items: 'T list) =
    let sorted = List.sort items
    
    interface IEnumerable<'T> with
        member _.GetEnumerator() = 
            (sorted :> IEnumerable<'T>).GetEnumerator()
    
    interface IEnumerable with
        member this.GetEnumerator() = 
            (this :> IEnumerable<'T>).GetEnumerator() :> IEnumerator
    
    member _.Count = List.length sorted
    member _.Min = List.head sorted
    member _.Max = List.last sorted
    member _.Contains(x) = List.contains x sorted
    
    member _.Add(x: 'T) = SortedList(x :: items)
    
    member _.Remove(x: 'T) = SortedList(List.filter ((<>) x) items)
    
    // Binary search
    member _.BinarySearch(target: 'T) =
        let arr = Array.ofList sorted
        let rec search low high =
            if low > high then None
            else
                let mid = (low + high) / 2
                let cmp = compare arr.[mid] target
                if cmp = 0 then Some mid
                elif cmp < 0 then search (mid + 1) high
                else search low (mid - 1)
        search 0 (arr.Length - 1)

let sl = SortedList([5; 3; 8; 1; 9; 2; 7])
printfn "\nSortedList:"
for x in sl do printf "%d " x
printfn ""
printfn "Min: %d, Max: %d" sl.Min sl.Max
printfn "BinarySearch 7: %A" (sl.BinarySearch(7))   // Some 5
printfn "BinarySearch 4: %A" (sl.BinarySearch(4))   // None
```

---

## สรุป (Summary)

```
Custom Data Structures ใน F#:

Linked List:
- Functional: recursive DU Empty | Node
- Doubly: mutable class with prev/next pointers
- O(1) prepend, O(n) append

Circular Buffer:
- Fixed-size, overwrites oldest
- Ring indexing: (head + i) % capacity
- ใช้สำหรับ streaming data

Persistent Stack:
- Immutable, retains history
- Undo/redo support

Difference Lists:
- O(1) append via function composition
- ใช้สำหรับ building strings/lists

Association Lists:
- Simple key-value list
- ดีสำหรับ small collections

Multimap:
- Map<K, V list>
- One key -> many values
- ใช้สำหรับ indexes, grouping

Interval Tree:
- Query overlapping intervals
- ใช้สำหรับ scheduling, range queries

Custom IEnumerable:
- Implement interface for custom iteration
- Lazy sequences
```

---

*จบ Part 26 - โครงสร้างข้อมูลที่กำหนดเอง (Custom Data Structures)*
