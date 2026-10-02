# Part 16 - ความไม่เปลี่ยนแปลง (Immutability and Pure Functions)

## บทนำ (Introduction)

ใน F# ค่าทุกอย่างเป็น **immutable** (ไม่เปลี่ยนแปลงได้) โดย default นี่คือหนึ่งในหลักการสำคัญของ functional programming ที่ทำให้โค้ดปลอดภัย คาดเดาได้ และทดสอบง่าย

---

## 1. Immutability by Default ใน F#

```fsharp
// ค่าใน F# ไม่เปลี่ยนแปลงได้ by default
let x = 5
// x <- 10  // ERROR! ไม่สามารถเปลี่ยนค่าได้

// ต้องการ mutable ต้องประกาศชัดเจน
let mutable y = 5
y <- 10  // ได้!
printfn "y = %d" y
```

```fsharp
// Records ก็ immutable by default
type Point = { X: float; Y: float }

let p1 = { X = 1.0; Y = 2.0 }
// p1.X <- 5.0  // ERROR!

// สร้าง copy ด้วย { with }
let p2 = { p1 with X = 5.0 }  // p1 ไม่เปลี่ยน

printfn "p1: %A" p1  // { X = 1.0; Y = 2.0 }
printfn "p2: %A" p2  // { X = 5.0; Y = 2.0 }
```

```fsharp
// Lists ก็ immutable
let list1 = [1; 2; 3]
let list2 = 0 :: list1  // สร้าง list ใหม่ list1 ไม่เปลี่ยน

printfn "list1: %A" list1  // [1; 2; 3]
printfn "list2: %A" list2  // [0; 1; 2; 3]
```

---

## 2. Benefits of Immutability (ประโยชน์)

### Thread Safety

```fsharp
// ปัญหาของ mutable state ใน concurrent code

// แบบ MUTABLE (อันตราย!)
let mutable counter = 0

let incrementUnsafe () =
    // Race condition! หลาย thread อาจอ่านและเขียนพร้อมกัน
    let current = counter
    System.Threading.Thread.Sleep(1)  // simulate latency
    counter <- current + 1

// แบบ IMMUTABLE (ปลอดภัย!)
// ใช้ Interlocked หรือ functional approach
let safeIncrement (current: int) = current + 1  // pure function

// ไม่มี shared mutable state = ไม่มี race condition
```

```fsharp
// Parallel computation ด้วย immutable data

let numbers = [1..100]

// ปลอดภัยใช้ parallel เพราะ immutable
let sumOfSquares =
    numbers
    |> List.toArray
    |> Array.Parallel.map (fun x -> x * x)
    |> Array.sum

printfn "Sum of squares: %d" sumOfSquares
```

### Predictability

```fsharp
// Immutable values ทำให้โค้ดคาดเดาได้

let double x = x * 2  // pure function: ผลลัพธ์คาดเดาได้เสมอ

let x = 5
let y = double x   // y = 10 เสมอ
let z = double x   // z = 10 เสมอ (ไม่ขึ้นกับ state)

// ฟังก์ชันนี้ทดสอบง่าย:
// double 5 = 10  เสมอ ไม่ว่าจะเรียกครั้งไหน
// double 0 = 0   เสมอ
// double -3 = -6 เสมอ
```

### Testability

```fsharp
// Pure functions ทดสอบง่ายมาก

// ฟังก์ชันบวก 2 ตัวเลข (pure)
let add x y = x + y

// ทดสอบง่าย: ไม่ต้องใช้ mock, fixture หรือ setup
assert (add 1 2 = 3)
assert (add 0 0 = 0)
assert (add -1 1 = 0)
assert (add 100 200 = 300)
printfn "All add tests passed!"
```

---

## 3. Pure Functions (ฟังก์ชันบริสุทธิ์)

```fsharp
// Pure function: 
// 1. ผลลัพธ์ขึ้นอยู่กับ input เท่านั้น (deterministic)
// 2. ไม่มี side effects

// PURE functions
let square x = x * x                    // no side effects
let add x y = x + y                     // deterministic
let length (lst: 'a list) = lst.Length  // no external state

// IMPURE functions
let mutable state = 0
let impureGet () = state  // depends on external state
let impureSet v = state <- v  // side effect: mutation

let printSomething x =
    printfn "%d" x  // side effect: I/O
    x

let getTime () = System.DateTime.Now  // non-deterministic
```

```fsharp
// ตัวอย่าง pure vs impure

// IMPURE: อ่าน/เขียน global state
let mutable globalConfig = Map.empty<string, string>

let getConfig key =
    Map.tryFind key globalConfig  // depends on global state

let setConfig key value =
    globalConfig <- Map.add key value globalConfig  // side effect

// PURE: รับ config เป็น parameter
let getConfigPure (config: Map<string, string>) key =
    Map.tryFind key config

let setConfigPure (config: Map<string, string>) key value =
    Map.add key value config  // returns new Map

// ใช้งาน
let config1 = Map.empty
let config2 = setConfigPure config1 "host" "localhost"
let config3 = setConfigPure config2 "port" "8080"

printfn "host: %A" (getConfigPure config3 "host")
printfn "port: %A" (getConfigPure config3 "port")
```

---

## 4. Side Effects and When They're Needed

```fsharp
// Side effects จำเป็นสำหรับ:
// 1. I/O (อ่าน/เขียนไฟล์, network, database)
// 2. Logging
// 3. User interface
// 4. Random numbers
// 5. Time

// Best practice: isolate side effects ที่ boundaries

// Core logic (pure)
let calculateTotal (items: (string * float * int) list) =
    items
    |> List.sumBy (fun (_, price, qty) -> price * float qty)

let applyDiscount (rate: float) total =
    total * (1.0 - rate)

let formatReceipt items total discounted =
    let lines = items |> List.map (fun (name, price, qty) ->
        sprintf "  %-20s %3d × $%6.2f = $%7.2f" name qty price (price * float qty))
    let header = "=== Receipt ==="
    let footer = sprintf "  Subtotal: $%.2f\n  Discount: $%.2f\n  Total:    $%.2f" total (total - discounted) discounted
    header :: lines @ [footer] |> String.concat "\n"

// Side effects (at the boundary)
let processOrder items discountRate =
    let total = calculateTotal items
    let discounted = applyDiscount discountRate total
    let receipt = formatReceipt items total discounted
    
    // Side effects here (I/O)
    printfn "%s" receipt
    
    // Return pure value
    discounted

let items = [
    ("Coffee", 4.50, 2)
    ("Muffin", 3.25, 1)
    ("Orange Juice", 5.00, 1)
]

let finalTotal = processOrder items 0.10
printfn "Payment: $%.2f" finalTotal
```

---

## 5. Referential Transparency

```fsharp
// Referential transparency: สามารถแทนที่ expression ด้วยค่าของมันได้เสมอ

// TRANSPARENT: add 1 2 สามารถแทนที่ด้วย 3 ได้เสมอ
let add x y = x + y
let r1 = add 1 2 + add 3 4  // เหมือนกับ 3 + 7 = 10
let r2 = 3 + 7               // 10

printfn "%d = %d" r1 r2  // true

// NOT TRANSPARENT: ผลลัพธ์อาจต่างกันในการเรียกแต่ละครั้ง
let rand = System.Random()
let randomNum () = rand.Next()  // ไม่ใช่ referentially transparent

let n1 = randomNum ()  // อาจได้ 42
let n2 = randomNum ()  // อาจได้ 87 (ต่างกัน!)
```

```fsharp
// ประโยชน์ของ referential transparency:
// 1. Easier to reason about code
// 2. Safe to refactor
// 3. Can be cached/memoized
// 4. Can be reordered

// ตัวอย่าง: refactoring ที่ปลอดภัย

// ก่อน
let result1 = (1 + 2) * (1 + 2)

// หลัง (refactored, แต่ทำงานเหมือนกัน)
let sum = 1 + 2
let result2 = sum * sum

printfn "%d = %d" result1 result2  // true
```

---

## 6. Functional Updates (Record Update Syntax)

```fsharp
// Record update syntax: { existing with field = newValue }

type Person = {
    Name: string
    Age: int
    Email: string
    Active: bool
}

let alice = { Name = "Alice"; Age = 30; Email = "alice@example.com"; Active = true }

// อัพเดท field เดียว
let alice2 = { alice with Age = 31 }

// อัพเดทหลาย fields
let alice3 = { alice with Age = 31; Email = "alice.new@example.com" }

// alice ไม่เปลี่ยน
printfn "alice: %A" alice
printfn "alice2: %A" alice2
printfn "alice3: %A" alice3
```

```fsharp
// Nested records
type Address = { Street: string; City: string; Country: string }

type Employee = {
    Name: string
    Age: int
    Address: Address
}

let emp = {
    Name = "Bob"
    Age = 25
    Address = { Street = "123 Main St"; City = "Bangkok"; Country = "Thailand" }
}

// อัพเดท nested record
let emp2 = { emp with Address = { emp.Address with City = "Chiang Mai" } }

printfn "Original city: %s" emp.Address.City
printfn "Updated city: %s" emp2.Address.City
```

```fsharp
// Functional update pattern สำหรับ history/undo

type AppState = {
    Count: int
    Message: string
    History: AppState list
}

let increment state =
    { state with
        Count = state.Count + 1
        History = state :: state.History }

let setMessage msg state =
    { state with
        Message = msg
        History = state :: state.History }

let undo state =
    match state.History with
    | [] -> state
    | prev :: _ -> prev

let initialState = { Count = 0; Message = ""; History = [] }
let state1 = increment initialState      // Count = 1
let state2 = increment state1            // Count = 2
let state3 = setMessage "Hello" state2   // Message = "Hello"
let state4 = undo state3                 // ย้อนกลับ

printfn "state1.Count = %d" state1.Count
printfn "state2.Count = %d" state2.Count
printfn "state3.Message = %s" state3.Message
printfn "state4.Count = %d" state4.Count  // 2 (ย้อนกลับ)
```

---

## 7. Persistent Data Structures Concept

```fsharp
// Persistent data structures: เมื่อ "เปลี่ยน" จะได้ version ใหม่ ไม่ทำลาย version เดิม

// List เป็น persistent data structure
let list1 = [1; 2; 3]
let list2 = 0 :: list1      // [0; 1; 2; 3] - สร้างจาก list1 ไม่ copy ทั้งหมด
let list3 = 99 :: list1     // [99; 1; 2; 3] - share [1;2;3] กับ list1

// Structural sharing: ทั้ง list1, list2, list3 share [1;2;3] ใน memory
// efficient! ไม่ต้อง copy ทั้งหมด

printfn "list1: %A" list1
printfn "list2: %A" list2
printfn "list3: %A" list3
```

```fsharp
// Map ก็ persistent
let map1 = Map.ofList [("a", 1); ("b", 2)]
let map2 = Map.add "c" 3 map1    // map1 ไม่เปลี่ยน
let map3 = Map.remove "a" map1   // map1 ไม่เปลี่ยน

printfn "map1: %A" map1
printfn "map2: %A" map2
printfn "map3: %A" map3
```

```fsharp
// Set ก็ persistent
let set1 = Set.ofList [1; 2; 3; 4; 5]
let set2 = Set.add 6 set1
let set3 = Set.remove 1 set1

printfn "set1: %A" set1
printfn "set2: %A" set2
printfn "set3: %A" set3
```

---

## 8. When to Use mutable

```fsharp
// mutable ควรใช้เมื่อ:
// 1. Performance critical code (inner loops)
// 2. Interop กับ .NET code
// 3. Algorithms ที่ต้องการ mutable state โดยเนื้อแท้

// ตัวอย่างที่ mutable สมเหตุสมผล: in-place sort
let sortInPlace (arr: int[]) =
    let mutable changed = true
    while changed do
        changed <- false
        for i in 0..arr.Length-2 do
            if arr.[i] > arr.[i+1] then
                let temp = arr.[i]
                arr.[i] <- arr.[i+1]
                arr.[i+1] <- temp
                changed <- true
    arr

let arr = [|5; 3; 1; 4; 2|]
sortInPlace arr |> ignore
printfn "Sorted: %A" arr  // [|1; 2; 3; 4; 5|]
```

```fsharp
// Performance: mutable ภายใน function (local scope)
// เป็นที่ยอมรับได้ถ้า external interface ยังคง functional

let sumWithMutable lst =
    let mutable sum = 0
    for x in lst do
        sum <- sum + x
    sum  // ส่งคืน immutable value

// เหมือนกับ List.sum แต่เร็วกว่าในบางกรณี
printfn "sum: %d" (sumWithMutable [1..100])  // 5050
```

```fsharp
// Encapsulate mutation ใน class
type Counter() =
    let mutable value = 0
    
    member _.Increment() = value <- value + 1
    member _.Decrement() = value <- value - 1
    member _.Reset() = value <- 0
    member _.Value = value

let c = Counter()
c.Increment()
c.Increment()
c.Increment()
c.Decrement()
printfn "Counter value: %d" c.Value  // 2
```

---

## 9. mutable keyword

```fsharp
// การใช้ mutable

// mutable ใน let binding
let mutable x = 0
x <- x + 1
x <- x + 1
printfn "x = %d" x  // 2

// mutable ใน loop
let mutable total = 0.0
let prices = [10.5; 20.0; 15.75; 8.25]
for p in prices do
    total <- total + p
printfn "total = %.2f" total  // 54.50
```

```fsharp
// mutable ใน record (ระวัง: ทำให้ record ไม่ pure)
type MutablePoint = {
    mutable X: float
    mutable Y: float
}

let p = { X = 1.0; Y = 2.0 }
p.X <- 5.0  // OK เพราะ X เป็น mutable
p.Y <- 10.0

printfn "p = %A" p  // { X = 5.0; Y = 10.0 }
```

```fsharp
// ใช้ mutable เพื่อสะสม results ที่ต้องการ performance
let buildString (parts: string list) =
    let sb = System.Text.StringBuilder()
    for part in parts do
        sb.Append(part) |> ignore
    sb.ToString()

let result = buildString ["Hello"; " "; "World"; "!"]
printfn "%s" result  // "Hello World!"

// เปรียบเทียบ: functional แต่ช้ากว่า
let buildString2 = String.concat ""
let result2 = buildString2 ["Hello"; " "; "World"; "!"]
```

---

## 10. ref cells

```fsharp
// ref cells: wraps mutable value ใน reference

let counter = ref 0

counter := !counter + 1  // := เขียนค่า, ! อ่านค่า
counter := !counter + 1
printfn "counter = %d" !counter  // 2

// ref cell ทำให้ส่ง mutable value เป็น argument ได้
let incrementBy n cell =
    cell := !cell + n

let myCell = ref 10
incrementBy 5 myCell
printfn "myCell = %d" !myCell  // 15
```

```fsharp
// ใช้ ref cell สำหรับ shared state ระหว่าง closures
let makeAccumulator () =
    let total = ref 0
    let add n = total := !total + n
    let get () = !total
    (add, get)

let (add, get) = makeAccumulator ()
add 10
add 20
add 5
printfn "Total: %d" (get ())  // 35
```

---

## 11. Mutable Fields in Records

```fsharp
// Record ที่มี mutable fields
type GameState = {
    mutable Score: int
    mutable Lives: int
    PlayerName: string  // immutable
}

let game = { Score = 0; Lives = 3; PlayerName = "Alice" }

// อัพเดท mutable fields
game.Score <- game.Score + 100
game.Score <- game.Score + 250
game.Lives <- game.Lives - 1

printfn "Player: %s" game.PlayerName  // ไม่เปลี่ยน
printfn "Score: %d" game.Score
printfn "Lives: %d" game.Lives
```

```fsharp
// เมื่อไหรควรใช้ mutable fields vs record copy?

// Mutable fields: เหมาะเมื่อ
// - ต้องการ performance สูง
// - Object มีอายุยาวและ update บ่อย
// - Game state, simulation, accumulator

// Record copy: เหมาะเมื่อ
// - ต้องการ immutability
// - ต้องการ history/undo
// - Thread safety สำคัญ
// - Functional style เป็นที่ต้องการ
```

---

## 12. Porting Mutable Code to Immutable Style

```fsharp
// ตัวอย่าง: แปลง imperative มาเป็น functional

// IMPERATIVE (mutable)
let imperativeFilter (predicate: int -> bool) (lst: int list) : int list =
    let mutable result = []
    for x in lst do
        if predicate x then
            result <- result @ [x]
    result

// FUNCTIONAL (immutable)
let functionalFilter predicate lst =
    List.filter predicate lst

// MANUAL FUNCTIONAL (เพื่อเข้าใจ)
let rec manualFilter predicate = function
    | [] -> []
    | x :: rest ->
        if predicate x then x :: manualFilter predicate rest
        else manualFilter predicate rest

let data = [1..10]
let p = (fun x -> x % 2 = 0)

printfn "imperative: %A" (imperativeFilter p data)
printfn "functional: %A" (functionalFilter p data)
printfn "manual: %A" (manualFilter p data)
```

```fsharp
// แปลง loop-based algorithm เป็น functional

// IMPERATIVE: หา max ด้วย loop
let maxImperative (lst: int list) =
    if List.isEmpty lst then failwith "Empty list"
    let mutable currentMax = List.head lst
    for x in List.tail lst do
        if x > currentMax then
            currentMax <- x
    currentMax

// FUNCTIONAL: หา max ด้วย fold
let maxFunctional =
    List.reduce max

let nums = [3; 1; 4; 1; 5; 9; 2; 6]
printfn "maxImperative: %d" (maxImperative nums)  // 9
printfn "maxFunctional: %d" (maxFunctional nums)  // 9
```

```fsharp
// แปลง mutable accumulator เป็น fold

// IMPERATIVE: คำนวณ statistics
let statsImperative (data: float list) =
    let mutable n = 0
    let mutable sum = 0.0
    let mutable min = System.Double.MaxValue
    let mutable max = System.Double.MinValue
    
    for x in data do
        n <- n + 1
        sum <- sum + x
        if x < min then min <- x
        if x > max then max <- x
    
    (n, sum, min, max, sum / float n)

// FUNCTIONAL: ด้วย fold
let statsFunctional (data: float list) =
    let (n, sum, min, max) =
        data |> List.fold (fun (n, s, mn, mx) x ->
            (n + 1, s + x, System.Math.Min(mn, x), System.Math.Max(mx, x)))
            (0, 0.0, System.Double.MaxValue, System.Double.MinValue)
    (n, sum, min, max, sum / float n)

let data = [1.0; 5.0; 3.0; 2.0; 4.0]
let (n1, s1, mn1, mx1, avg1) = statsImperative data
let (n2, s2, mn2, mx2, avg2) = statsFunctional data

printfn "n=%d sum=%.1f min=%.1f max=%.1f avg=%.2f" n1 s1 mn1 mx1 avg1
printfn "n=%d sum=%.1f min=%.1f max=%.1f avg=%.2f" n2 s2 mn2 mx2 avg2
```

---

## 13. Examples of Refactoring

```fsharp
// Refactoring 1: Shopping cart

// BEFORE: mutable state
type MutableCart() =
    let mutable items: (string * float * int) list = []
    
    member _.AddItem name price qty =
        items <- (name, price, qty) :: items
    
    member _.RemoveItem name =
        items <- List.filter (fun (n, _, _) -> n <> name) items
    
    member _.Total =
        items |> List.sumBy (fun (_, p, q) -> p * float q)

// AFTER: immutable functional style
type CartItem = { Name: string; Price: float; Quantity: int }
type Cart = { Items: CartItem list }

let emptyCart = { Items = [] }

let addItem item cart =
    { cart with Items = item :: cart.Items }

let removeItem name cart =
    { cart with Items = cart.Items |> List.filter (fun i -> i.Name <> name) }

let cartTotal cart =
    cart.Items |> List.sumBy (fun i -> i.Price * float i.Quantity)

let updateQuantity name qty cart =
    { cart with
        Items = cart.Items |> List.map (fun i ->
            if i.Name = name then { i with Quantity = qty }
            else i) }

// ใช้งาน
let cart =
    emptyCart
    |> addItem { Name = "Apple"; Price = 1.50; Quantity = 3 }
    |> addItem { Name = "Bread"; Price = 3.00; Quantity = 1 }
    |> addItem { Name = "Milk"; Price = 2.50; Quantity = 2 }
    |> updateQuantity "Apple" 5
    |> removeItem "Bread"

printfn "Cart items: %A" (List.map (fun i -> i.Name) cart.Items)
printfn "Cart total: $%.2f" (cartTotal cart)
```

```fsharp
// Refactoring 2: Configuration builder

// BEFORE: mutable builder
type Config() =
    let mutable host = "localhost"
    let mutable port = 8080
    let mutable timeout = 30
    
    member _.SetHost h = host <- h
    member _.SetPort p = port <- p
    member _.SetTimeout t = timeout <- t
    member _.Build() = {| Host = host; Port = port; Timeout = timeout |}

// AFTER: functional builder
type ServerConfig = {
    Host: string
    Port: int
    Timeout: int
}

let defaultConfig = { Host = "localhost"; Port = 8080; Timeout = 30 }

let withHost h config = { config with Host = h }
let withPort p config = { config with Port = p }
let withTimeout t config = { config with Timeout = t }

// ใช้งาน
let prodConfig =
    defaultConfig
    |> withHost "api.example.com"
    |> withPort 443
    |> withTimeout 60

printfn "Config: %A" prodConfig
```

---

## สรุป (Summary)

```fsharp
// หลักการ Immutability ใน F#

// 1. Immutable by default
let x = 5  // ไม่เปลี่ยนได้

// 2. Pure functions
let square n = n * n  // ผลลัพธ์ขึ้นกับ input เท่านั้น

// 3. Functional updates
type Config = { Host: string; Port: int }
let config = { Host = "localhost"; Port = 8080 }
let updated = { config with Port = 9090 }  // สร้างใหม่

// 4. mutable เมื่อจำเป็น
let mutable counter = 0
counter <- counter + 1

// 5. ref cells
let refVal = ref 0
refVal := !refVal + 1

// สรุปประโยชน์
// - Thread safety: ไม่มี shared mutable state
// - Predictability: ผลลัพธ์คาดเดาได้
// - Testability: ทดสอบง่าย
// - Referential transparency: แทนที่ expression ด้วยค่าได้
// - Persistent data: ไม่สูญเสีย history

printfn "x = %d" x
printfn "square 5 = %d" (square 5)
printfn "config: %A" config
printfn "updated: %A" updated
printfn "counter = %d" counter
printfn "refVal = %d" !refVal
```

**กฎง่ายๆ:**
1. เริ่มด้วย immutable เสมอ
2. ใช้ mutable เมื่อมีเหตุผลชัดเจน (performance, interop)
3. encapsulate mutation ไว้ในขอบเขตเล็กที่สุดที่เป็นไปได้
4. Pure functions ก่อน, side effects ที่ boundary
