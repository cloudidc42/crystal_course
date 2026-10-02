# Part 23 - คิวและสแตก (Queues and Stacks)

## บทนำ (Introduction)

คิว (Queue) และสแตก (Stack) เป็นโครงสร้างข้อมูลพื้นฐานที่สำคัญ:
- **Stack** - Last In, First Out (LIFO) - เหมือนกองจาน
- **Queue** - First In, First Out (FIFO) - เหมือนแถวคน

ใน F# เราสามารถใช้ทั้ง functional immutable versions และ .NET mutable versions

---

## 23.1 Functional Queue Implementation

```fsharp
// Queue แบบ functional ใช้ two-list technique
// - inbox: รับของใหม่ (prepend เร็ว)
// - outbox: ส่งของออก (pop เร็ว)
// เมื่อ outbox ว่าง จะ reverse inbox มาใส่ outbox

type Queue<'T> = {
    Inbox: 'T list
    Outbox: 'T list
}

module Queue =
    let empty : Queue<'T> = { Inbox = []; Outbox = [] }
    
    let isEmpty q = q.Inbox = [] && q.Outbox = []
    
    let enqueue item q = { q with Inbox = item :: q.Inbox }
    
    let dequeue q =
        match q.Outbox with
        | head :: tail -> (head, { q with Outbox = tail })
        | [] ->
            match List.rev q.Inbox with
            | [] -> failwith "Queue is empty"
            | head :: tail -> (head, { Inbox = []; Outbox = tail })
    
    let tryDequeue q =
        match q.Outbox with
        | head :: tail -> Some (head, { q with Outbox = tail })
        | [] ->
            match List.rev q.Inbox with
            | [] -> None
            | head :: tail -> Some (head, { Inbox = []; Outbox = tail })
    
    let peek q =
        match q.Outbox with
        | head :: _ -> head
        | [] ->
            match List.rev q.Inbox with
            | [] -> failwith "Queue is empty"
            | head :: _ -> head
    
    let length q = List.length q.Inbox + List.length q.Outbox
    
    let toList q = q.Outbox @ List.rev q.Inbox
    
    let ofList lst = 
        lst |> List.fold (fun q item -> enqueue item q) empty

// ============ ใช้งาน Functional Queue ============
let q0 = Queue.empty
let q1 = Queue.enqueue 1 q0
let q2 = Queue.enqueue 2 q1
let q3 = Queue.enqueue 3 q2

printfn "Queue: %A" (Queue.toList q3)   // [1; 2; 3]
printfn "Length: %d" (Queue.length q3)   // 3

let (item1, q4) = Queue.dequeue q3
printfn "Dequeued: %d" item1   // 1
printfn "Remaining: %A" (Queue.toList q4)   // [2; 3]

let (item2, q5) = Queue.dequeue q4
printfn "Dequeued: %d" item2   // 2

// ============ Amortized O(1) analysis ============
// enqueue: O(1) always
// dequeue: O(1) amortized (O(n) when reversing inbox, but each item reversed once)
// peek: O(1) amortized
// length: O(1)

// ============ tryDequeue ============
let q6 = Queue.ofList [10; 20; 30]
match Queue.tryDequeue q6 with
| Some (item, rest) -> printfn "tryDequeue: %d, remaining: %A" item (Queue.toList rest)
| None -> printfn "Queue empty"

// ============ BFS using functional queue ============
let bfsWithFunctionalQueue (graph: Map<int, int list>) (start: int) =
    let rec loop queue visited order =
        if Queue.isEmpty queue then List.rev order
        else
            let (node, rest) = Queue.dequeue queue
            if Set.contains node visited then
                loop rest visited order
            else
                let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
                let newQueue = neighbors |> List.fold (fun q n -> Queue.enqueue n q) rest
                loop newQueue (Set.add node visited) (node :: order)
    
    loop (Queue.enqueue start Queue.empty) Set.empty []

let testGraph = Map.ofList [
    (0, [1; 2])
    (1, [0; 3; 4])
    (2, [0; 5])
    (3, [1])
    (4, [1])
    (5, [2])
]

printfn "BFS: %A" (bfsWithFunctionalQueue testGraph 0)
```

---

## 23.2 Priority Queue Implementation

```fsharp
// Priority Queue ใช้ Binary Heap
// Min-heap: element ที่มีค่าน้อยสุดอยู่ด้านบน

type PriorityQueue<'T when 'T : comparison>(items: ('T * int) list) =
    // Store as list of (item, priority)
    // priority น้อย = สำคัญกว่า (min-heap)
    let mutable heap : ('T * int) array = 
        items |> List.toArray |> Array.sortBy snd
    let mutable size = items.Length
    
    let swap i j =
        let temp = heap.[i]
        heap.[i] <- heap.[j]
        heap.[j] <- temp
    
    let rec siftUp i =
        if i > 0 then
            let parent = (i - 1) / 2
            if snd heap.[i] < snd heap.[parent] then
                swap i parent
                siftUp parent
    
    let rec siftDown i =
        let left = 2 * i + 1
        let right = 2 * i + 2
        let mutable smallest = i
        
        if left < size && snd heap.[left] < snd heap.[smallest] then
            smallest <- left
        if right < size && snd heap.[right] < snd heap.[smallest] then
            smallest <- right
        
        if smallest <> i then
            swap i smallest
            siftDown smallest
    
    member _.IsEmpty = size = 0
    member _.Count = size
    
    member this.Enqueue(item: 'T, priority: int) =
        if size >= heap.Length then
            let newHeap = Array.create (heap.Length * 2 + 1) (Unchecked.defaultof<'T>, 0)
            Array.blit heap 0 newHeap 0 size
            heap <- newHeap
        heap.[size] <- (item, priority)
        siftUp size
        size <- size + 1
    
    member this.Dequeue() =
        if size = 0 then failwith "Priority queue is empty"
        let (item, pri) = heap.[0]
        size <- size - 1
        if size > 0 then
            heap.[0] <- heap.[size]
            siftDown 0
        (item, pri)
    
    member this.Peek() =
        if size = 0 then failwith "Priority queue is empty"
        heap.[0]

// ============ ใช้งาน Priority Queue ============
let pq = PriorityQueue<string>([])
pq.Enqueue("low priority task", 10)
pq.Enqueue("urgent task", 1)
pq.Enqueue("normal task", 5)
pq.Enqueue("critical task", 0)
pq.Enqueue("medium task", 3)

printfn "Processing in priority order:"
while not pq.IsEmpty do
    let (task, priority) = pq.Dequeue()
    printfn "  Priority %d: %s" priority task

// ============ Dijkstra using Priority Queue ============
let dijkstra (graph: Map<int, (int * int) list>) (start: int) (target: int) =
    let pq = PriorityQueue<int>([])
    pq.Enqueue(start, 0)
    
    let mutable dist = Map.ofList [(start, 0)]
    let mutable prev = Map.empty<int, int>
    
    let rec loop () =
        if pq.IsEmpty then ()
        else
            let (node, d) = pq.Dequeue()
            
            if node = target then ()
            else
                let currentDist = dist |> Map.tryFind node |> Option.defaultValue System.Int32.MaxValue
                if d <= currentDist then
                    let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
                    for (neighbor, weight) in neighbors do
                        let newDist = currentDist + weight
                        let neighborDist = dist |> Map.tryFind neighbor |> Option.defaultValue System.Int32.MaxValue
                        if newDist < neighborDist then
                            dist <- Map.add neighbor newDist dist
                            prev <- Map.add neighbor node prev
                            pq.Enqueue(neighbor, newDist)
                loop ()
    
    loop ()
    
    // Reconstruct path
    let rec buildPath node path =
        match Map.tryFind node prev with
        | Some p -> buildPath p (node :: path)
        | None -> node :: path
    
    (dist |> Map.tryFind target, buildPath target [])

let weightedGraph = Map.ofList [
    (0, [(1, 4); (2, 1)])
    (1, [(3, 1)])
    (2, [(1, 2); (3, 5)])
    (3, [])
]

match dijkstra weightedGraph 0 3 with
| (Some dist, path) -> printfn "Shortest path 0->3: distance=%d, path=%A" dist path
| (None, _) -> printfn "No path found"
```

---

## 23.3 Stack Using Lists

```fsharp
// Stack แบบ functional ใช้ list
// head ของ list = top ของ stack
// push/pop: O(1)

type Stack<'T> = 'T list

module Stack =
    let empty : Stack<'T> = []
    
    let isEmpty (s: Stack<'T>) = s = []
    
    let push item (s: Stack<'T>) : Stack<'T> = item :: s
    
    let pop (s: Stack<'T>) =
        match s with
        | [] -> failwith "Stack is empty"
        | top :: rest -> (top, rest)
    
    let tryPop (s: Stack<'T>) =
        match s with
        | [] -> None
        | top :: rest -> Some (top, rest)
    
    let peek (s: Stack<'T>) =
        match s with
        | [] -> failwith "Stack is empty"
        | top :: _ -> top
    
    let size (s: Stack<'T>) = List.length s
    
    let toList (s: Stack<'T>) = s
    
    let ofList (lst: 'T list) : Stack<'T> = List.rev lst

// ============ ใช้งาน Functional Stack ============
let s0 = Stack.empty
let s1 = Stack.push 1 s0
let s2 = Stack.push 2 s1
let s3 = Stack.push 3 s2

printfn "Stack: %A" (Stack.toList s3)   // [3; 2; 1]
printfn "Peek: %d" (Stack.peek s3)       // 3

let (top1, s4) = Stack.pop s3
printfn "Popped: %d" top1               // 3
printfn "Remaining: %A" (Stack.toList s4)  // [2; 1]

// ============ DFS using functional stack ============
let dfsWithStack (graph: Map<int, int list>) (start: int) =
    let rec loop stack visited order =
        match Stack.tryPop stack with
        | None -> List.rev order
        | Some (node, rest) ->
            if Set.contains node visited then
                loop rest visited order
            else
                let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
                let newStack = neighbors |> List.fold (fun s n -> Stack.push n s) rest
                loop newStack (Set.add node visited) (node :: order)
    
    loop (Stack.push start Stack.empty) Set.empty []

let graph = Map.ofList [
    (0, [1; 2])
    (1, [0; 3; 4])
    (2, [0; 5])
    (3, [1])
    (4, [1])
    (5, [2])
]

printfn "DFS: %A" (dfsWithStack graph 0)
// Output (may vary): DFS: [0; 2; 5; 1; 4; 3]
```

---

## 23.4 Queue<T> (.NET Mutable)

```fsharp
open System.Collections.Generic

// ============ Queue<T> ============
let q = Queue<int>()

// Enqueue - เพิ่มท้ายคิว
q.Enqueue(1)
q.Enqueue(2)
q.Enqueue(3)
q.Enqueue(4)
q.Enqueue(5)

printfn "Count: %d" q.Count   // 5

// Peek - ดูหัวคิวโดยไม่ลบ
printfn "Peek: %d" (q.Peek())   // 1

// Dequeue - เอาออกจากหัวคิว
let first = q.Dequeue()
printfn "Dequeued: %d" first   // 1
printfn "Count after dequeue: %d" q.Count   // 4

// TryDequeue (F# 8 / .NET 5+)
let mutable value = 0
if q.TryDequeue(&value) then
    printfn "TryDequeued: %d" value   // 2
else
    printfn "Queue empty"

// TryPeek
let mutable peekVal = 0
if q.TryPeek(&peekVal) then
    printfn "TryPeek: %d" peekVal

// Contains
printfn "Contains 3: %b" (q.Contains(3))
printfn "Contains 10: %b" (q.Contains(10))

// Clear
let q2 = Queue<string>()
q2.Enqueue("a")
q2.Enqueue("b")
q2.Enqueue("c")
q2.Clear()
printfn "After clear: %d" q2.Count   // 0

// สร้างจาก collection
let q3 = Queue<int>(seq { 1..10 })
printfn "Queue from seq count: %d" q3.Count

// iterate (ไม่ dequeue)
for item in q do
    printf "%d " item
printfn ""

// CopyTo
let arr = Array.create q.Count 0
q.CopyTo(arr, 0)
printfn "CopyTo: %A" arr

// ToArray
let arrFromQ = q.ToArray()
printfn "ToArray: %A" arrFromQ

// ============ ตัวอย่าง: Task Queue ============
type Task = { Id: int; Name: string; Priority: int }

let taskQueue = Queue<Task>()
taskQueue.Enqueue({ Id = 1; Name = "Build"; Priority = 1 })
taskQueue.Enqueue({ Id = 2; Name = "Test"; Priority = 2 })
taskQueue.Enqueue({ Id = 3; Name = "Deploy"; Priority = 3 })

printfn "\nProcessing tasks:"
while taskQueue.Count > 0 do
    let task = taskQueue.Dequeue()
    printfn "  Processing: %s (id=%d)" task.Name task.Id
```

---

## 23.5 Stack<T> (.NET Mutable)

```fsharp
open System.Collections.Generic

// ============ Stack<T> ============
let s = Stack<int>()

// Push - เพิ่มบน stack
s.Push(1)
s.Push(2)
s.Push(3)
s.Push(4)
s.Push(5)

printfn "Count: %d" s.Count   // 5

// Peek - ดู top โดยไม่ลบ
printfn "Peek: %d" (s.Peek())   // 5

// Pop - เอาออกจาก top
let top = s.Pop()
printfn "Popped: %d" top   // 5
printfn "Count after pop: %d" s.Count   // 4

// TryPop (F#/.NET modern)
let mutable popVal = 0
if s.TryPop(&popVal) then
    printfn "TryPopped: %d" popVal   // 4

// TryPeek
let mutable peekVal = 0
if s.TryPeek(&peekVal) then
    printfn "TryPeek: %d" peekVal

// Contains
printfn "Contains 2: %b" (s.Contains(2))

// Clear
let s2 = Stack<string>()
s2.Push("a")
s2.Push("b")
s2.Clear()
printfn "After clear: %d" s2.Count

// สร้างจาก collection
let s3 = Stack<int>(seq { 1..5 })
printfn "Stack from seq: %A" (s3.ToArray())

// iterate (top to bottom)
for item in s do
    printf "%d " item
printfn ""

// ============ ตัวอย่าง: Undo/Redo ============
type Command = 
    | SetValue of int
    | AddValue of int
    | MultiplyValue of int

type Editor = {
    mutable Value: int
    UndoStack: Stack<Command>
    RedoStack: Stack<Command>
}

let createEditor () = {
    Value = 0
    UndoStack = Stack<Command>()
    RedoStack = Stack<Command>()
}

let applyCommand (editor: Editor) (cmd: Command) =
    editor.UndoStack.Push(cmd)
    editor.RedoStack.Clear()
    match cmd with
    | SetValue v -> editor.Value <- v
    | AddValue v -> editor.Value <- editor.Value + v
    | MultiplyValue v -> editor.Value <- editor.Value * v

let undo (editor: Editor) =
    if editor.UndoStack.Count > 0 then
        let cmd = editor.UndoStack.Pop()
        editor.RedoStack.Push(cmd)
        // Recompute from scratch (simplified)
        let initialValue = 0
        editor.Value <- 
            editor.UndoStack.ToArray()
            |> Array.rev
            |> Array.fold (fun v cmd ->
                match cmd with
                | SetValue x -> x
                | AddValue x -> v + x
                | MultiplyValue x -> v * x
            ) initialValue
        printfn "Undone: %A, Value: %d" cmd editor.Value
    else
        printfn "Nothing to undo"

let editor = createEditor()
applyCommand editor (SetValue 10)
applyCommand editor (AddValue 5)
applyCommand editor (MultiplyValue 2)
printfn "Value after commands: %d" editor.Value   // (10+5)*2 = 30

undo editor   // undo MultiplyValue 2 -> 15
undo editor   // undo AddValue 5 -> 10
```

---

## 23.6 LinkedList<T>

```fsharp
open System.Collections.Generic

// ============ LinkedList<T> ============
// Doubly linked list
let ll = LinkedList<int>()

// AddLast - เพิ่มท้าย
ll.AddLast(1) |> ignore
ll.AddLast(2) |> ignore
ll.AddLast(3) |> ignore

// AddFirst - เพิ่มหัว
ll.AddFirst(0) |> ignore

printfn "LinkedList: %A" (ll |> Seq.toList)   // [0; 1; 2; 3]

// First / Last
printfn "First: %d" ll.First.Value   // 0
printfn "Last: %d" ll.Last.Value     // 3

// Find a node
let node2 = ll.Find(2)
if node2 <> null then
    printfn "Found 2"
    
    // AddBefore/AddAfter
    ll.AddBefore(node2, 15) |> ignore
    ll.AddAfter(node2, 25) |> ignore
    printfn "After insert: %A" (ll |> Seq.toList)
    // [0; 1; 15; 2; 25; 3]

// Remove
ll.Remove(15) |> ignore
printfn "After remove 15: %A" (ll |> Seq.toList)

// RemoveFirst / RemoveLast
ll.RemoveFirst()
ll.RemoveLast()
printfn "After remove first/last: %A" (ll |> Seq.toList)

// Navigate with Next/Previous
let mutable current = ll.First
while current <> null do
    printf "%d " current.Value
    current <- current.Next
printfn ""

// ============ ตัวอย่าง: LRU Cache ============
type LRUCache<'K, 'V>(capacity: int) =
    let dict = Dictionary<'K, LinkedListNode<'K * 'V>>()
    let lruList = LinkedList<'K * 'V>()
    
    member _.Get(key: 'K) : 'V option =
        match dict.TryGetValue(key) with
        | true, node ->
            lruList.Remove(node)
            lruList.AddFirst(node)
            Some (snd node.Value)
        | false, _ -> None
    
    member _.Put(key: 'K, value: 'V) =
        match dict.TryGetValue(key) with
        | true, node ->
            lruList.Remove(node)
            lruList.AddFirst(node)
            // Update value (create new node since LinkedListNode is immutable in value)
            dict.[key] <- lruList.First
        | false, _ ->
            if lruList.Count >= capacity then
                let lruNode = lruList.Last
                lruList.RemoveLast()
                dict.Remove(fst lruNode.Value) |> ignore
            
            let newNode = lruList.AddFirst((key, value))
            dict.[key] <- newNode

let lru = LRUCache<int, string>(3)
lru.Put(1, "one")
lru.Put(2, "two")
lru.Put(3, "three")
printfn "Get 1: %A" (lru.Get(1))    // Some "one"
lru.Put(4, "four")   // evicts 2 (least recently used)
printfn "Get 2: %A" (lru.Get(2))    // None (evicted)
printfn "Get 3: %A" (lru.Get(3))    // Some "three"
```

---

## 23.7 Deque Implementation

```fsharp
// Deque (Double-ended Queue) - เพิ่ม/ลบได้ทั้งสองด้าน

type Deque<'T> = {
    Front: 'T list
    Back: 'T list
}

module Deque =
    let empty = { Front = []; Back = [] }
    
    let isEmpty d = d.Front = [] && d.Back = []
    
    let length d = List.length d.Front + List.length d.Back
    
    let pushFront item d = { d with Front = item :: d.Front }
    
    let pushBack item d = { d with Back = item :: d.Back }
    
    let popFront d =
        match d.Front with
        | head :: tail -> (head, { d with Front = tail })
        | [] ->
            match List.rev d.Back with
            | [] -> failwith "Deque is empty"
            | head :: tail -> (head, { Front = tail; Back = [] })
    
    let popBack d =
        match d.Back with
        | head :: tail -> (head, { d with Back = tail })
        | [] ->
            match List.rev d.Front with
            | [] -> failwith "Deque is empty"
            | head :: tail -> (head, { Front = []; Back = tail })
    
    let peekFront d =
        match d.Front with
        | head :: _ -> head
        | [] ->
            match List.rev d.Back with
            | [] -> failwith "Deque is empty"
            | head :: _ -> head
    
    let peekBack d =
        match d.Back with
        | head :: _ -> head
        | [] ->
            match List.rev d.Front with
            | [] -> failwith "Deque is empty"
            | head :: _ -> head
    
    let toList d = d.Front @ List.rev d.Back

// ============ ใช้งาน Deque ============
let d0 = Deque.empty
let d1 = d0 |> Deque.pushBack 1 |> Deque.pushBack 2 |> Deque.pushBack 3
let d2 = d1 |> Deque.pushFront 0
let d3 = d2 |> Deque.pushFront (-1)

printfn "Deque: %A" (Deque.toList d3)   // [-1; 0; 1; 2; 3]

let (frontItem, d4) = Deque.popFront d3
printfn "PopFront: %d" frontItem   // -1

let (backItem, d5) = Deque.popBack d4
printfn "PopBack: %d" backItem    // 3

printfn "Remaining: %A" (Deque.toList d5)   // [0; 1; 2]

// ============ ตัวอย่าง: Sliding Window Maximum ============
let slidingWindowMax (arr: int[]) (k: int) =
    if arr.Length = 0 || k = 0 then [||]
    else
        let deque = System.Collections.Generic.LinkedList<int>()  // store indices
        let result = System.Collections.Generic.List<int>()
        
        for i in 0..arr.Length - 1 do
            // ลบ elements ที่อยู่นอก window
            while deque.Count > 0 && deque.First.Value < i - k + 1 do
                deque.RemoveFirst()
            
            // ลบ elements ที่น้อยกว่า arr.[i]
            while deque.Count > 0 && arr.[deque.Last.Value] < arr.[i] do
                deque.RemoveLast()
            
            deque.AddLast(i) |> ignore
            
            // เพิ่ม maximum ของ window ปัจจุบัน
            if i >= k - 1 then
                result.Add(arr.[deque.First.Value])
        
        result.ToArray()

let arr = [| 1; 3; -1; -3; 5; 3; 6; 7 |]
let maxK3 = slidingWindowMax arr 3
printfn "Sliding window max (k=3): %A" maxK3
// Output: [3; 3; 5; 5; 6; 7]
```

---

## 23.8 Ring Buffer

```fsharp
// Ring Buffer (Circular Buffer)
// Fixed-size buffer ที่เมื่อเต็มจะเขียนทับข้อมูลเก่า

type RingBuffer<'T>(capacity: int) =
    let data = Array.create capacity Unchecked.defaultof<'T>
    let mutable head = 0    // read pointer
    let mutable tail = 0    // write pointer
    let mutable count = 0
    
    member _.Capacity = capacity
    member _.Count = count
    member _.IsEmpty = count = 0
    member _.IsFull = count = capacity
    
    member _.Write(item: 'T) =
        data.[tail] <- item
        tail <- (tail + 1) % capacity
        if count < capacity then
            count <- count + 1
        else
            // Overwrite: move head forward
            head <- (head + 1) % capacity
    
    member _.Read() =
        if count = 0 then failwith "Buffer empty"
        let item = data.[head]
        head <- (head + 1) % capacity
        count <- count - 1
        item
    
    member _.TryRead() =
        if count = 0 then None
        else
            let item = data.[head]
            head <- (head + 1) % capacity
            count <- count - 1
            Some item
    
    member _.Peek() =
        if count = 0 then failwith "Buffer empty"
        data.[head]
    
    member _.ToArray() =
        [| for i in 0..count - 1 -> data.[(head + i) % capacity] |]

// ============ ใช้งาน Ring Buffer ============
let rb = RingBuffer<int>(5)

// เติม buffer
for i in 1..5 do
    rb.Write(i)
    
printfn "Full buffer: %A" (rb.ToArray())   // [|1; 2; 3; 4; 5|]
printfn "IsFull: %b" rb.IsFull            // true

// เขียนเพิ่ม (เขียนทับเก่า)
rb.Write(6)
rb.Write(7)
printfn "After overwrite: %A" (rb.ToArray())   // [|3; 4; 5; 6; 7|]

// อ่าน
let v1 = rb.Read()
let v2 = rb.Read()
printfn "Read: %d, %d" v1 v2   // 3, 4
printfn "After reads: %A" (rb.ToArray())   // [|5; 6; 7|]

// ============ ตัวอย่าง: Moving Average ============
type MovingAverage(windowSize: int) =
    let buffer = RingBuffer<float>(windowSize)
    let mutable sum = 0.0
    let mutable filledCount = 0
    
    member _.AddSample(value: float) =
        if buffer.IsFull then
            let oldest = buffer.Peek()
            sum <- sum - oldest
        else
            filledCount <- filledCount + 1
        
        buffer.Write(value)
        sum <- sum + value
    
    member _.Average =
        if filledCount = 0 then 0.0
        else sum / float (min filledCount windowSize)

let ma = MovingAverage(3)
let samples = [| 2.0; 4.0; 6.0; 8.0; 10.0; 12.0 |]

printfn "\nMoving Average (window=3):"
for s in samples do
    ma.AddSample(s)
    printfn "  Added %.1f, MA = %.2f" s ma.Average

// ============ Lock-free Ring Buffer concept ============
(*
    ใน production code ควรใช้ System.Collections.Concurrent.ConcurrentQueue
    หรือ System.Threading.Channels สำหรับ thread-safe ring buffer
    
    open System.Threading.Channels
    let channel = Channel.CreateBounded<int>(100)
    // channel.Writer.TryWrite(item)
    // channel.Reader.TryRead(&item)
*)
```

---

## 23.9 BFS with Queue, DFS with Stack

```fsharp
// ============ BFS (Breadth-First Search) ============
// ใช้ Queue เพื่อ explore ระดับต่อระดับ

open System.Collections.Generic

type Graph = Map<string, string list>

let buildGraph edges =
    edges |> List.fold (fun g (from, to_) ->
        let neighbors = g |> Map.tryFind from |> Option.defaultValue []
        Map.add from (to_ :: neighbors) g
    ) Map.empty

let bfs (graph: Graph) (start: string) =
    let queue = Queue<string>()
    let visited = HashSet<string>()
    let order = ResizeArray<string>()
    
    queue.Enqueue(start)
    visited.Add(start) |> ignore
    
    while queue.Count > 0 do
        let node = queue.Dequeue()
        order.Add(node)
        
        let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
        for neighbor in neighbors do
            if not (visited.Contains(neighbor)) then
                visited.Add(neighbor) |> ignore
                queue.Enqueue(neighbor)
    
    order |> Seq.toList

// BFS shortest path
let bfsShortestPath (graph: Graph) (start: string) (target: string) =
    let queue = Queue<string * string list>()
    let visited = HashSet<string>()
    
    queue.Enqueue((start, [start]))
    visited.Add(start) |> ignore
    
    let mutable result = None
    
    while queue.Count > 0 && result.IsNone do
        let (node, path) = queue.Dequeue()
        
        if node = target then
            result <- Some path
        else
            let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
            for neighbor in neighbors do
                if not (visited.Contains(neighbor)) then
                    visited.Add(neighbor) |> ignore
                    queue.Enqueue((neighbor, path @ [neighbor]))
    
    result

let cityGraph = Map.ofList [
    ("Bangkok", ["Chiang Mai"; "Phuket"; "Pattaya"])
    ("Chiang Mai", ["Bangkok"; "Pai"])
    ("Phuket", ["Bangkok"; "Krabi"])
    ("Pattaya", ["Bangkok"])
    ("Pai", ["Chiang Mai"])
    ("Krabi", ["Phuket"])
]

printfn "BFS from Bangkok: %A" (bfs cityGraph "Bangkok")
printfn "Shortest path Bangkok->Pai: %A" (bfsShortestPath cityGraph "Bangkok" "Pai")

// ============ DFS (Depth-First Search) ============
// ใช้ Stack เพื่อ explore ลึกสุดก่อน

let dfs (graph: Graph) (start: string) =
    let stack = Stack<string>()
    let visited = HashSet<string>()
    let order = ResizeArray<string>()
    
    stack.Push(start)
    
    while stack.Count > 0 do
        let node = stack.Pop()
        
        if not (visited.Contains(node)) then
            visited.Add(node) |> ignore
            order.Add(node)
            
            let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
            for neighbor in List.rev neighbors do   // reverse เพื่อให้ visit ตามลำดับ
                if not (visited.Contains(neighbor)) then
                    stack.Push(neighbor)
    
    order |> Seq.toList

// DFS recursive (เทียบกับ iterative)
let dfsRecursive (graph: Graph) (start: string) =
    let visited = HashSet<string>()
    let order = ResizeArray<string>()
    
    let rec visit node =
        if not (visited.Contains(node)) then
            visited.Add(node) |> ignore
            order.Add(node)
            let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
            for neighbor in neighbors do
                visit neighbor
    
    visit start
    order |> Seq.toList

printfn "\nDFS (iterative): %A" (dfs cityGraph "Bangkok")
printfn "DFS (recursive): %A" (dfsRecursive cityGraph "Bangkok")
```

---

## 23.10 Expression Evaluation Using Stack

```fsharp
// ============ Evaluate Reverse Polish Notation (RPN) ============
// ตัวอย่าง: "3 4 + 2 * 7 /" = (3+4)*2/7 = 2

let evaluateRPN (expression: string) =
    let stack = Stack<float>()
    let tokens = expression.Split(' ')
    
    for token in tokens do
        match token with
        | "+" -> 
            let b = stack.Pop()
            let a = stack.Pop()
            stack.Push(a + b)
        | "-" -> 
            let b = stack.Pop()
            let a = stack.Pop()
            stack.Push(a - b)
        | "*" -> 
            let b = stack.Pop()
            let a = stack.Pop()
            stack.Push(a * b)
        | "/" -> 
            let b = stack.Pop()
            let a = stack.Pop()
            stack.Push(a / b)
        | num -> stack.Push(float num)
    
    stack.Pop()

printfn "3 4 + 2 * = %f" (evaluateRPN "3 4 + 2 *")    // (3+4)*2 = 14
printfn "5 1 2 + 4 * + 3 - = %f" (evaluateRPN "5 1 2 + 4 * + 3 -")  // 14

// ============ Infix to RPN (Shunting-yard algorithm) ============
let infixToRPN (expression: string) =
    let output = ResizeArray<string>()
    let opStack = Stack<string>()
    
    let precedence = function
        | "+" | "-" -> 1
        | "*" | "/" -> 2
        | _ -> 0
    
    let tokens = expression.Split(' ')
    
    for token in tokens do
        match token with
        | "+" | "-" | "*" | "/" ->
            while opStack.Count > 0 && 
                  opStack.Peek() <> "(" && 
                  precedence (opStack.Peek()) >= precedence token do
                output.Add(opStack.Pop())
            opStack.Push(token)
        | "(" -> opStack.Push(token)
        | ")" ->
            while opStack.Count > 0 && opStack.Peek() <> "(" do
                output.Add(opStack.Pop())
            if opStack.Count > 0 then opStack.Pop() |> ignore
        | num -> output.Add(num)
    
    while opStack.Count > 0 do
        output.Add(opStack.Pop())
    
    output |> Seq.toArray |> String.concat " "

let infix = "3 + 4 * 2"
let rpn = infixToRPN infix
printfn "Infix: %s -> RPN: %s" infix rpn
printfn "Evaluated: %f" (evaluateRPN rpn)   // 3+4*2 = 11

// ============ Balanced Parentheses Check ============
let isBalanced (s: string) =
    let stack = Stack<char>()
    let matching = dict [(')', '('); (']', '['); ('}', '{')]
    
    let mutable valid = true
    for c in s do
        if valid then
            match c with
            | '(' | '[' | '{' -> stack.Push(c)
            | ')' | ']' | '}' ->
                if stack.Count = 0 || stack.Peek() <> matching.[c] then
                    valid <- false
                else
                    stack.Pop() |> ignore
            | _ -> ()
    
    valid && stack.Count = 0

printfn "\nBalance check:"
printfn "({[]}): %b" (isBalanced "({[]})")   // true
printfn "([)]: %b" (isBalanced "([)]")        // false
printfn "{[()]}: %b" (isBalanced "{[()]}")    // true
printfn "((: %b" (isBalanced "((")             // false
```

---

## 23.11 Topological Sort with Stack

```fsharp
open System.Collections.Generic

// Topological Sort ใช้ DFS + Stack
// ใช้สำหรับ task dependency ordering

let topologicalSort (graph: Map<string, string list>) =
    let visited = HashSet<string>()
    let stack = Stack<string>()
    
    let allNodes = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    let rec dfs node =
        if not (visited.Contains(node)) then
            visited.Add(node) |> ignore
            let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
            for neighbor in neighbors do
                dfs neighbor
            stack.Push(node)
    
    for node in allNodes do
        dfs node
    
    stack |> Seq.toList

// ตัวอย่าง: task dependencies
let dependencies = Map.ofList [
    ("deploy", ["build"; "test"])
    ("test", ["build"; "lint"])
    ("build", ["compile"])
    ("lint", ["compile"])
    ("compile", [])
]

let order = topologicalSort dependencies
printfn "Build order: %A" order
// Output should show compile before build before test/lint before deploy

// ============ Kahn's Algorithm (BFS-based topological sort) ============
let kahnSort (graph: Map<string, string list>) =
    let allNodes = 
        graph |> Map.toList 
        |> List.collect (fun (k, vs) -> k :: vs)
        |> List.distinct
    
    // Calculate in-degrees
    let inDegree = Dictionary<string, int>()
    for node in allNodes do
        if not (inDegree.ContainsKey(node)) then
            inDegree.[node] <- 0
    
    for KeyValue(_, neighbors) in graph do
        for n in neighbors do
            inDegree.[n] <- (inDegree |> Seq.tryFind (fun kv -> kv.Key = n) |> Option.map (fun kv -> kv.Value) |> Option.defaultValue 0) + 1
    
    let queue = Queue<string>()
    for node in allNodes do
        if inDegree.[node] = 0 then
            queue.Enqueue(node)
    
    let result = ResizeArray<string>()
    
    while queue.Count > 0 do
        let node = queue.Dequeue()
        result.Add(node)
        
        let neighbors = graph |> Map.tryFind node |> Option.defaultValue []
        for neighbor in neighbors do
            inDegree.[neighbor] <- inDegree.[neighbor] - 1
            if inDegree.[neighbor] = 0 then
                queue.Enqueue(neighbor)
    
    if result.Count = allNodes.Length then
        Some (result |> Seq.toList)
    else
        None   // Cycle detected

match kahnSort dependencies with
| Some order -> printfn "Kahn order: %A" order
| None -> printfn "Cycle detected!"
```

---

## 23.12 Stack Applications: Calculator

```fsharp
open System.Collections.Generic

// ============ Full Expression Calculator ============
// Support: +, -, *, /, (, ), unary minus

type Token =
    | Number of float
    | Plus | Minus | Star | Slash
    | LeftParen | RightParen

let tokenize (s: string) : Token list =
    let mutable i = 0
    let tokens = ResizeArray<Token>()
    
    while i < s.Length do
        match s.[i] with
        | ' ' -> i <- i + 1
        | '+' -> tokens.Add(Plus); i <- i + 1
        | '-' -> tokens.Add(Minus); i <- i + 1
        | '*' -> tokens.Add(Star); i <- i + 1
        | '/' -> tokens.Add(Slash); i <- i + 1
        | '(' -> tokens.Add(LeftParen); i <- i + 1
        | ')' -> tokens.Add(RightParen); i <- i + 1
        | c when System.Char.IsDigit(c) || c = '.' ->
            let mutable j = i
            while j < s.Length && (System.Char.IsDigit(s.[j]) || s.[j] = '.') do
                j <- j + 1
            tokens.Add(Number (float s.[i..j-1]))
            i <- j
        | _ -> i <- i + 1
    
    tokens |> Seq.toList

let evaluateExpression (expr: string) =
    let tokens = tokenize expr
    let values = Stack<float>()
    let ops = Stack<Token>()
    
    let precedence = function
        | Plus | Minus -> 1
        | Star | Slash -> 2
        | _ -> 0
    
    let applyOp () =
        let b = values.Pop()
        let a = values.Pop()
        match ops.Pop() with
        | Plus -> values.Push(a + b)
        | Minus -> values.Push(a - b)
        | Star -> values.Push(a * b)
        | Slash -> values.Push(a / b)
        | _ -> ()
    
    for token in tokens do
        match token with
        | Number n -> values.Push(n)
        | LeftParen -> ops.Push(token)
        | RightParen ->
            while ops.Count > 0 && ops.Peek() <> LeftParen do
                applyOp ()
            if ops.Count > 0 then ops.Pop() |> ignore
        | op ->
            while ops.Count > 0 && 
                  ops.Peek() <> LeftParen &&
                  precedence (ops.Peek()) >= precedence op do
                applyOp ()
            ops.Push(op)
    
    while ops.Count > 0 do
        applyOp ()
    
    if values.Count > 0 then values.Pop()
    else 0.0

printfn "3 + 4 * 2 = %f" (evaluateExpression "3 + 4 * 2")         // 11
printfn "(3 + 4) * 2 = %f" (evaluateExpression "(3 + 4) * 2")      // 14
printfn "10 / 2 + 3 * 4 = %f" (evaluateExpression "10 / 2 + 3 * 4") // 17
printfn "2 * (3 + (4 - 1)) * 2 = %f" (evaluateExpression "2 * (3 + (4 - 1)) * 2") // 24
```

---

## สรุป (Summary)

```
Queue (FIFO):
- Functional: สองลิสต์ (inbox/outbox), amortized O(1)
- Priority Queue: heap-based, O(log n)
- Queue<T> (.NET): mutable, hash-based

Stack (LIFO):
- Functional: list, O(1)
- Stack<T> (.NET): mutable
- Applications: DFS, expression evaluation, undo/redo

Deque:
- Double-ended queue
- Push/pop จากทั้งสองด้าน

Ring Buffer:
- Fixed-size circular buffer
- เขียนทับข้อมูลเก่าเมื่อเต็ม
- ใช้สำหรับ streaming data, moving average

Applications:
- BFS: ใช้ Queue
- DFS: ใช้ Stack
- Expression evaluation: ใช้ Stack
- Topological sort: ใช้ Queue หรือ Stack
```

---

*จบ Part 23 - คิวและสแตก (Queues and Stacks)*
