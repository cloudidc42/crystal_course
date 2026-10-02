# Part 74 - Property-Based Testing กับ FsCheck

## บทนำ (Introduction)

Property-Based Testing (PBT) เป็นวิธีการทดสอบที่แทนที่จะเขียน test cases แบบ specific เราเขียน **properties** (คุณสมบัติ) ที่ควรเป็นจริงเสมอสำหรับทุก input

**ตัวอย่างเปรียบเทียบ:**

Example-based testing:
```fsharp
// ทดสอบเพียงไม่กี่ cases
Assert.Equal([3;1;2] |> List.sort, [1;2;3])
Assert.Equal([5;4] |> List.sort, [4;5])
```

Property-based testing:
```fsharp
// ทดสอบ property ที่ต้องเป็นจริงสำหรับ ทุก list
let ``sort produces sorted list`` (lst: int list) =
    let sorted = List.sort lst
    // Property: ทุก element ต้องไม่ > element ถัดไป
    sorted |> List.pairwise |> List.forall (fun (a, b) -> a <= b)
```

FsCheck จะสร้าง input แบบ random หลายร้อยครั้งเพื่อหา counter-example

---

## 1. FsCheck Basics

### การติดตั้ง

```xml
<!-- TestProject.fsproj -->
<ItemGroup>
  <PackageReference Include="FsCheck" Version="2.16.6" />
  <!-- สำหรับ xUnit integration -->
  <PackageReference Include="FsCheck.Xunit" Version="2.16.6" />
  <!-- สำหรับ NUnit integration -->
  <!-- <PackageReference Include="FsCheck.NUnit" Version="2.16.6" /> -->
  <!-- สำหรับ Expecto integration -->
  <PackageReference Include="Expecto.FsCheck" Version="10.1.0" />
</ItemGroup>
```

### การใช้งาน FsCheck โดยตรง

```fsharp
// BasicFsCheck.fs
module Tests.BasicFsCheck

open FsCheck

// Property แบบง่ายที่สุด
let ``reverse twice gives original`` (lst: int list) =
    List.rev (List.rev lst) = lst

// รัน property check
Check.Quick ``reverse twice gives original``
// Output: Ok, passed 100 tests.

// รันแบบ verbose
Check.Verbose ``reverse twice gives original``
// Output: 
// 0: []
// 1: [1]
// 2: [2; 1]
// ...
// Ok, passed 100 tests.

// กำหนดจำนวน tests
let config = { Config.Default with MaxTest = 1000 }
Check.One(config, ``reverse twice gives original``)
```

---

## 2. Arbitrary<T>

```fsharp
// ArbitraryTests.fs
module Tests.ArbitraryTests

open FsCheck
open FsCheck.Xunit

// Arbitrary คือ generator + shrinker สำหรับ type หนึ่ง
// FsCheck มี built-in Arbitrary สำหรับ primitive types

// ดู Arbitrary สำหรับ int
let intArb = Arb.from<int>
let stringArb = Arb.from<string>
let listArb = Arb.from<int list>

// สร้าง Arbitrary เอง
type PositiveInt = PositiveInt of int with
    static member Arbitrary() =
        Arb.from<int>
        |> Arb.filter (fun n -> n > 0)
        |> Arb.convert PositiveInt (fun (PositiveInt n) -> n)

type NonEmptyString = NonEmptyString of string with
    static member Arbitrary() =
        Arb.from<string>
        |> Arb.filter (fun s -> s <> null && s.Length > 0)
        |> Arb.convert NonEmptyString (fun (NonEmptyString s) -> s)

// ใช้ custom Arbitrary
[<Property>]
let ``positive int is always positive`` (PositiveInt n) =
    n > 0

[<Property>]
let ``non-empty string has length > 0`` (NonEmptyString s) =
    s.Length > 0

// Arbitrary.generate - สร้าง values
let sampleInts = Gen.sample 10 5 (Arb.generate<int>)
printfn "Sample ints: %A" sampleInts
```

---

## 3. Property Definition

```fsharp
// PropertyDefinitions.fs
module Tests.PropertyDefinitions

open FsCheck
open FsCheck.Xunit

// === Property as boolean function ===

// Property: reverse of reverse = original
[<Property>]
let ``list reverse is involutory`` (lst: int list) =
    List.rev (List.rev lst) = lst

// Property: sorted list has all elements in order
[<Property>]
let ``sort produces ordered list`` (lst: int list) =
    let sorted = List.sort lst
    sorted
    |> List.pairwise
    |> List.forall (fun (a, b) -> a <= b)

// Property: sorted list has same elements
[<Property>]
let ``sort preserves elements`` (lst: int list) =
    let sorted = List.sort lst
    sorted |> List.sort = List.sort sorted

// === Property กับ preconditions ===

// ใช้ ==> (implies) operator สำหรับ preconditions
[<Property>]
let ``division inverse of multiplication when divisor nonzero`` (a: int) (b: int) =
    b <> 0 ==> lazy (a * b / b = a)

// Property: absolute value is non-negative
[<Property>]
let ``abs is always non-negative`` (n: int) =
    abs n >= 0

// Property: max returns value >= both inputs
[<Property>]
let ``max returns at least each input`` (a: int) (b: int) =
    let m = max a b
    m >= a && m >= b

// Property: string concat length
[<Property>]
let ``string concat length is sum of lengths`` (s1: string) (s2: string) =
    let s1 = if s1 = null then "" else s1
    let s2 = if s2 = null then "" else s2
    (s1 + s2).Length = s1.Length + s2.Length

// Property: list append
[<Property>]
let ``list append length is sum`` (lst1: int list) (lst2: int list) =
    (lst1 @ lst2).Length = lst1.Length + lst2.Length

// === Property กับ labeling ===

[<Property>]
let ``add commutativity with label`` (a: int) (b: int) =
    let result = a + b = b + a
    result |@ sprintf "a=%d, b=%d: a+b=%d, b+a=%d" a b (a+b) (b+a)
```

---

## 4. Generators: Gen.choose, Gen.elements

```fsharp
// GeneratorTests.fs
module Tests.GeneratorTests

open FsCheck
open FsCheck.Xunit

// Gen.choose - สุ่มตัวเลขในช่วง
let rangeGen = Gen.choose (1, 100)

[<Property>]
let ``choose generates values in range`` () =
    let value = Gen.sample 1 1 rangeGen |> List.head
    value >= 1 && value <= 100

// Gen.elements - สุ่มจาก collection
let colorGen = Gen.elements ["red"; "green"; "blue"; "yellow"]
let diceGen = Gen.elements [1..6]

// สร้าง Arbitrary จาก Gen
let rangeArb = rangeGen |> Arb.fromGen
let colorArb = colorGen |> Arb.fromGen
let diceArb = diceGen |> Arb.fromGen

// ใช้กับ forAll
[<Property>]
let ``color is always valid`` () =
    let validColors = Set.ofList ["red"; "green"; "blue"; "yellow"]
    Prop.forAll colorArb (fun color ->
        validColors |> Set.contains color
    )

// Gen.constant - generator ที่ return ค่าเดิมเสมอ
let alwaysFive = Gen.constant 5
let alwaysHello = Gen.constant "hello"

// Gen.oneof - สุ่มเลือก generator
let mixedGen = Gen.oneof [
    Gen.constant 0
    Gen.choose (1, 100)
    Gen.choose (-100, -1)
]

// Gen.frequency - สุ่มตาม weight
let weightedGen = Gen.frequency [
    (70, Gen.choose (1, 10))    // 70% chance: 1-10
    (20, Gen.choose (11, 100))  // 20% chance: 11-100
    (10, Gen.constant 0)        // 10% chance: 0
]

// Gen.listOf - สร้าง list จาก generator
let listOf5to10 = 
    Gen.sized (fun size ->
        let size' = min size 10 |> max 5
        Gen.listOfLength size' (Gen.choose (1, 100))
    )

// Gen.map - transform generated values
let doubledGen = Gen.choose (1, 50) |> Gen.map ((*) 2)

[<Property>]
let ``doubled gen produces even numbers`` () =
    Prop.forAll (doubledGen |> Arb.fromGen) (fun n ->
        n % 2 = 0
    )

// Gen.zip - combine two generators
let pairGen = Gen.zip (Gen.choose (1, 10)) (Gen.choose (1, 10))

[<Property>]
let ``pair gen produces valid pairs`` () =
    Prop.forAll (pairGen |> Arb.fromGen) (fun (a, b) ->
        a >= 1 && a <= 10 && b >= 1 && b <= 10
    )

// Gen.where - filter values
let evenGen = Gen.choose (1, 100) |> Gen.where (fun n -> n % 2 = 0)

[<Property>]
let ``even gen produces only even numbers`` () =
    Prop.forAll (evenGen |> Arb.fromGen) (fun n ->
        n % 2 = 0
    )
```

---

## 5. Custom Generators

```fsharp
// CustomGenerators.fs
module Tests.CustomGenerators

open FsCheck
open FsCheck.Xunit

// Domain types
type EmailAddress = EmailAddress of string
type PhoneNumber = PhoneNumber of string
type Age = Age of int

type Person = {
    Name: string
    Email: EmailAddress
    Phone: PhoneNumber
    Age: Age
}

// Custom generators สำหรับ domain types
let emailGen =
    gen {
        let! username = Gen.choose (3, 12) |> Gen.bind (fun len ->
            Gen.listOfLength len (Gen.elements ['a'..'z'])
            |> Gen.map (fun chars -> System.String(Array.ofList chars))
        )
        let! domain = Gen.elements ["gmail.com"; "yahoo.com"; "hotmail.com"; "example.com"]
        return EmailAddress (sprintf "%s@%s" username domain)
    }

let phoneGen =
    gen {
        let! areaCode = Gen.choose (100, 999)
        let! prefix = Gen.choose (100, 999)
        let! line = Gen.choose (1000, 9999)
        return PhoneNumber (sprintf "+1-%d-%d-%d" areaCode prefix line)
    }

let ageGen =
    Gen.choose (18, 100) |> Gen.map Age

let nameGen =
    gen {
        let! firstName = Gen.elements [
            "Alice"; "Bob"; "Charlie"; "Diana"; "Eve"
            "Frank"; "Grace"; "Henry"; "Iris"; "Jack"
        ]
        let! lastName = Gen.elements [
            "Smith"; "Johnson"; "Williams"; "Brown"; "Jones"
            "Garcia"; "Miller"; "Davis"; "Wilson"; "Moore"
        ]
        return sprintf "%s %s" firstName lastName
    }

let personGen =
    gen {
        let! name = nameGen
        let! email = emailGen
        let! phone = phoneGen
        let! age = ageGen
        return { Name = name; Email = email; Phone = phone; Age = age }
    }

// Arbitrary สำหรับ Person
type PersonArbitraries =
    static member Person() = Arb.fromGen personGen

// ใช้ custom generator
[<Property(Arbitrary = [| typeof<PersonArbitraries> |])>]
let ``person has valid name`` (p: Person) =
    p.Name.Length > 0

[<Property(Arbitrary = [| typeof<PersonArbitraries> |])>]
let ``person has valid email`` (p: Person) =
    let (EmailAddress email) = p.Email
    email.Contains("@") && email.Contains(".")

[<Property(Arbitrary = [| typeof<PersonArbitraries> |])>]
let ``person age is in valid range`` (p: Person) =
    let (Age age) = p.Age
    age >= 18 && age <= 100

// Complex business rules
let isValidPerson (p: Person) =
    let (EmailAddress email) = p.Email
    let (Age age) = p.Age
    p.Name.Length > 0 
    && email.Contains("@") 
    && age >= 18

[<Property(Arbitrary = [| typeof<PersonArbitraries> |])>]
let ``generated persons are always valid`` (p: Person) =
    isValidPerson p

// Generator สำหรับ financial data
type Money = { Amount: decimal; Currency: string }

let moneyGen =
    gen {
        let! amount = 
            Gen.choose (1, 100000) 
            |> Gen.map (fun n -> decimal n / 100m)
        let! currency = Gen.elements ["USD"; "EUR"; "GBP"; "JPY"; "THB"]
        return { Amount = amount; Currency = currency }
    }

let validTransactionGen =
    gen {
        let! from = moneyGen
        let! to' = { from with Amount = Gen.sample 1 1 (Gen.choose (1, int from.Amount) |> Gen.map decimal) |> List.head }
        return (from, to')
    }
```

---

## 6. Shrinking

```fsharp
// ShrinkingDemo.fs
module Tests.ShrinkingDemo

open FsCheck
open FsCheck.Xunit

// เมื่อ FsCheck พบ counter-example มันจะ shrink ให้เล็กที่สุด

// Property ที่จะ fail สำหรับ numbers > 100
let ``property that fails for large numbers`` (n: int) =
    n < 100  // จะ fail สำหรับ n >= 100

// FsCheck จะ:
// 1. หา counter-example (เช่น n = 523)
// 2. Shrink: ลอง 262, ลอง 131, ลอง 100, ลอง 100
// 3. Report smallest: n = 100

// Custom shrinker
type EvenInt = EvenInt of int

type EvenArbitraries =
    static member EvenInt() =
        // Generator: สร้าง even numbers
        let gen = Gen.choose (-50, 50) |> Gen.map (fun n -> EvenInt (n * 2))
        
        // Shrinker: shrink even number
        let shrink (EvenInt n) =
            seq {
                if n > 0 then yield EvenInt (n - 2)
                if n < 0 then yield EvenInt (n + 2)
                if n <> 0 then yield EvenInt 0
            }
        
        Arb.fromGenShrink (gen, shrink)

[<Property(Arbitrary = [| typeof<EvenArbitraries> |])>]
let ``even numbers are always even`` (EvenInt n) =
    n % 2 = 0

// ตัวอย่างที่ shrinking มีประโยชน์
[<Property>]
let ``sorted list property`` (lst: int list) =
    // ถ้า property นี้ fail FsCheck จะ shrink list ให้เล็กที่สุดที่ fail
    let sorted = List.sort lst
    List.length sorted = List.length lst  // Should always be true
```

---

## 7. Properties as Invariants

```fsharp
// InvariantTests.fs
module Tests.InvariantTests

open FsCheck
open FsCheck.Xunit

// Stack invariants
type Stack<'a> = Stack of 'a list

let empty<'a> : Stack<'a> = Stack []
let push x (Stack xs) = Stack (x :: xs)
let pop (Stack xs) =
    match xs with
    | [] -> None
    | x :: rest -> Some (x, Stack rest)
let peek (Stack xs) = List.tryHead xs
let size (Stack xs) = List.length xs
let isEmpty (Stack xs) = List.isEmpty xs

// Invariant properties
[<Property>]
let ``push then pop gives same element`` (x: int) =
    let s = push x empty
    match pop s with
    | Some (top, _) -> top = x
    | None -> false

[<Property>]
let ``push increases size by 1`` (x: int) (Stack xs as stack) =
    let before = size stack
    let after = size (push x stack)
    after = before + 1

[<Property>]
let ``pop decreases size by 1`` (x: int) (stack: int list) =
    let s = Stack stack
    if isEmpty s then true  // Precondition: non-empty
    else
        match pop s with
        | Some (_, rest) -> size rest = size s - 1
        | None -> false

[<Property>]
let ``empty stack has size zero`` () =
    size empty<int> = 0

[<Property>]
let ``peek returns same as push`` (x: int) =
    let s = push x empty
    peek s = Some x

// Queue invariants
type Queue<'a> = Queue of 'a list * 'a list

let emptyQ<'a> : Queue<'a> = Queue ([], [])
let enqueue x (Queue (front, back)) = Queue (front, x :: back)
let dequeue (Queue (front, back)) =
    match front with
    | x :: rest -> Some (x, Queue (rest, back))
    | [] ->
        match List.rev back with
        | [] -> None
        | x :: rest -> Some (x, Queue (rest, []))
let queueSize (Queue (f, b)) = List.length f + List.length b

[<Property>]
let ``queue dequeue returns first enqueued`` (x: int) (y: int) =
    let q = enqueue y (enqueue x emptyQ)
    match dequeue q with
    | Some (first, _) -> first = x
    | None -> false

[<Property>]
let ``queue size increases on enqueue`` (x: int) (front: int list) (back: int list) =
    let q = Queue (front, back)
    queueSize (enqueue x q) = queueSize q + 1
```

---

## 8. Commutativity, Associativity Properties

```fsharp
// AlgebraicProperties.fs
module Tests.AlgebraicProperties

open FsCheck
open FsCheck.Xunit

// Commutativity: a op b = b op a
[<Property>]
let ``addition is commutative`` (a: int) (b: int) =
    a + b = b + a

[<Property>]
let ``multiplication is commutative`` (a: int) (b: int) =
    a * b = b * a

[<Property>]
let ``set union is commutative`` (s1: int list) (s2: int list) =
    let set1 = Set.ofList s1
    let set2 = Set.ofList s2
    Set.union set1 set2 = Set.union set2 set1

// Associativity: (a op b) op c = a op (b op c)
[<Property>]
let ``addition is associative`` (a: int) (b: int) (c: int) =
    (a + b) + c = a + (b + c)

[<Property>]
let ``string concat is associative`` (a: string) (b: string) (c: string) =
    let a = if a = null then "" else a
    let b = if b = null then "" else b
    let c = if c = null then "" else c
    (a + b) + c = a + (b + c)

[<Property>]
let ``list append is associative`` (l1: int list) (l2: int list) (l3: int list) =
    (l1 @ l2) @ l3 = l1 @ (l2 @ l3)

// Identity element: a op identity = identity op a = a
[<Property>]
let ``zero is identity for addition`` (n: int) =
    n + 0 = n && 0 + n = n

[<Property>]
let ``one is identity for multiplication`` (n: int) =
    n * 1 = n && 1 * n = n

[<Property>]
let ``empty list is identity for append`` (lst: int list) =
    [] @ lst = lst && lst @ [] = lst

// Distributivity: a * (b + c) = a*b + a*c
[<Property>]
let ``multiplication distributes over addition`` (a: int) (b: int) (c: int) =
    a * (b + c) = a * b + a * c

// Idempotency: a op a = a
[<Property>]
let ``set union is idempotent`` (lst: int list) =
    let s = Set.ofList lst
    Set.union s s = s

[<Property>]
let ``min is idempotent`` (n: int) =
    min n n = n

// Monotonicity
[<Property>]
let ``sort is monotone`` (lst: int list) =
    let sorted = List.sort lst
    sorted 
    |> List.pairwise 
    |> List.forall (fun (a, b) -> a <= b)
```

---

## 9. Round-Trip Properties

```fsharp
// RoundTripTests.fs
module Tests.RoundTripTests

open FsCheck
open FsCheck.Xunit
open System

// Serialization round-trips
[<Property>]
let ``int serializes and deserializes correctly`` (n: int) =
    let serialized = string n
    let deserialized = int serialized
    deserialized = n

[<Property>]
let ``float serializes and deserializes approximately`` (f: float) =
    not (Double.IsNaN f || Double.IsInfinity f) ==>
    lazy (
        let serialized = sprintf "%.10f" f
        let deserialized = float serialized
        abs (deserialized - f) < 0.0000001
    )

// Encoding round-trips
let encode (s: string) = 
    s |> System.Text.Encoding.UTF8.GetBytes |> Convert.ToBase64String

let decode (s: string) = 
    s |> Convert.FromBase64String |> System.Text.Encoding.UTF8.GetString

[<Property>]
let ``base64 encode decode roundtrip`` (s: string) =
    let s = if s = null then "" else s
    decode (encode s) = s

// Parsing round-trips
type Color = { R: byte; G: byte; B: byte }

let colorToHex (c: Color) = sprintf "#%02X%02X%02X" c.R c.G c.B

let hexToColor (hex: string) =
    let hex = hex.TrimStart('#')
    if hex.Length = 6 then
        let r = Convert.ToByte(hex.[0..1], 16)
        let g = Convert.ToByte(hex.[2..3], 16)
        let b = Convert.ToByte(hex.[4..5], 16)
        Some { R = r; G = g; B = b }
    else None

[<Property>]
let ``color hex roundtrip`` (r: byte) (g: byte) (b: byte) =
    let color = { R = r; G = g; B = b }
    let hex = colorToHex color
    match hexToColor hex with
    | Some decoded -> decoded = color
    | None -> false

// Compression round-trip (conceptual)
let compress (data: byte[]) =
    use ms = new System.IO.MemoryStream()
    use gz = new System.IO.Compression.GZipStream(ms, System.IO.Compression.CompressionMode.Compress)
    gz.Write(data, 0, data.Length)
    gz.Close()
    ms.ToArray()

let decompress (data: byte[]) =
    use input = new System.IO.MemoryStream(data)
    use gz = new System.IO.Compression.GZipStream(input, System.IO.Compression.CompressionMode.Decompress)
    use output = new System.IO.MemoryStream()
    gz.CopyTo(output)
    output.ToArray()

[<Property>]
let ``compression roundtrip preserves data`` (data: byte[]) =
    let compressed = compress data
    let decompressed = decompress compressed
    decompressed = data

// JSON round-trip (with Newtonsoft.Json)
// [<Property>]
// let ``json serialization roundtrip`` (p: Person) =
//     let json = JsonConvert.SerializeObject(p)
//     let deserialized = JsonConvert.DeserializeObject<Person>(json)
//     deserialized = p
```

---

## 10. Integration กับ xUnit

```fsharp
// XUnitIntegrationTests.fs
module Tests.XUnitIntegrationTests

open FsCheck
open FsCheck.Xunit
open Xunit

// [<Property>] attribute จาก FsCheck.Xunit
[<Property>]
let ``sorting preserves length`` (lst: int list) =
    List.sort lst |> List.length = List.length lst

// กำหนด test count
[<Property(MaxTest = 500)>]
let ``addition is always commutative`` (a: int) (b: int) =
    a + b = b + a

// Quiet mode - ไม่แสดง test cases
[<Property(Quiet = true)>]
let ``string has non-negative length`` (s: string) =
    let s = if s = null then "" else s
    s.Length >= 0

// Verbose - แสดงทุก test case
[<Property(Verbose = true)>]
let ``abs is non-negative`` (n: int) =
    abs n >= 0

// StartSize และ EndSize
[<Property(StartSize = 0, EndSize = 100)>]
let ``list length with bounded size`` (lst: int list) =
    lst.Length <= 100

// ผสมกับ xUnit Facts
[<Fact>]
let ``specific example still works`` () =
    // Example-based test
    let result = List.sort [3; 1; 2]
    Assert.Equal([1; 2; 3], result)

[<Property>]
let ``property test complements example test`` (lst: int list) =
    // Property test
    let sorted = List.sort lst
    sorted |> List.pairwise |> List.forall (fun (a, b) -> a <= b)

// Custom Arbitrary ใน xUnit integration
type PositiveIntArb =
    static member PositiveInt() =
        Arb.from<int> |> Arb.filter (fun n -> n > 0)

[<Property(Arbitrary = [| typeof<PositiveIntArb> |])>]
let ``sqrt of positive is positive`` (n: int) =
    sqrt (float n) > 0.0
```

---

## 11. Integration กับ Expecto

```fsharp
// ExpectoFsCheckTests.fs
module Tests.ExpectoFsCheckTests

open Expecto
open Expecto.FsCheck
open FsCheck

// testProperty สำหรับ Expecto
let fsCheckTests = testList "FsCheck with Expecto" [
    
    testProperty "reverse twice gives original" (fun (lst: int list) ->
        List.rev (List.rev lst) = lst
    )
    
    testProperty "sort preserves length" (fun (lst: int list) ->
        List.sort lst |> List.length = List.length lst
    )
    
    testProperty "abs is non-negative" (fun (n: int) ->
        abs n >= 0
    )
    
    testProperty "list concat length" (fun (l1: int list) (l2: int list) ->
        (l1 @ l2).Length = l1.Length + l2.Length
    )
    
    // กับ custom config
    testPropertyWithConfig { FsCheckConfig.defaultConfig with maxTest = 500 }
        "addition commutative with more tests"
        (fun (a: int) (b: int) -> a + b = b + a)
    
    // Async property
    testPropertyAsync "async property" (fun (n: int) ->
        async {
            do! Async.Sleep 1
            return abs n >= 0
        }
    )
]
```

---

## 12. Stateful Testing

```fsharp
// StatefulTests.fs
module Tests.StatefulTests

open FsCheck
open FsCheck.Xunit

// Model: Simple counter
type CounterState = { Count: int; History: int list }

type CounterCommand =
    | Increment
    | Decrement
    | Reset
    | SetValue of int

// Implementation
type Counter() =
    let mutable count = 0
    let mutable history = []
    
    member _.Increment() =
        count <- count + 1
        history <- count :: history
    
    member _.Decrement() =
        count <- count - 1
        history <- count :: history
    
    member _.Reset() =
        count <- 0
        history <- 0 :: history
    
    member _.SetValue(n: int) =
        count <- n
        history <- n :: history
    
    member _.Count = count
    member _.History = history

// Stateful property test
[<Property>]
let ``counter state is consistent`` (commands: CounterCommand list) =
    let counter = Counter()
    let mutable expectedCount = 0
    
    for cmd in commands do
        match cmd with
        | Increment ->
            counter.Increment()
            expectedCount <- expectedCount + 1
        | Decrement ->
            counter.Decrement()
            expectedCount <- expectedCount - 1
        | Reset ->
            counter.Reset()
            expectedCount <- 0
        | SetValue n ->
            counter.SetValue(n)
            expectedCount <- n
    
    counter.Count = expectedCount

[<Property>]
let ``counter history tracks all changes`` (commands: CounterCommand list) =
    let counter = Counter()
    
    for cmd in commands do
        match cmd with
        | Increment -> counter.Increment()
        | Decrement -> counter.Decrement()
        | Reset -> counter.Reset()
        | SetValue n -> counter.SetValue(n)
    
    counter.History.Length = commands.Length
```

---

## 13. Model-Based Testing

```fsharp
// ModelBasedTests.fs
module Tests.ModelBasedTests

open FsCheck
open FsCheck.Xunit

// Model (specification) vs Implementation

// Model: Simple dictionary (spec)
type DictionaryModel<'k, 'v when 'k : comparison> = Map<'k, 'v>

// Operations on model
let modelAdd k v (model: DictionaryModel<_, _>) = Map.add k v model
let modelGet k (model: DictionaryModel<_, _>) = Map.tryFind k model
let modelRemove k (model: DictionaryModel<_, _>) = Map.remove k model
let modelContains k (model: DictionaryModel<_, _>) = Map.containsKey k model
let modelCount (model: DictionaryModel<_, _>) = Map.count model

// Implementation: Actual Dictionary
open System.Collections.Generic

type DictionaryImpl<'k, 'v when 'k : comparison>() =
    let dict = Dictionary<'k, 'v>()
    
    member _.Add(k, v) = dict.[k] <- v
    member _.Get(k) =
        match dict.TryGetValue(k) with
        | true, v -> Some v
        | _ -> None
    member _.Remove(k) = dict.Remove(k) |> ignore
    member _.Contains(k) = dict.ContainsKey(k)
    member _.Count = dict.Count

// Model-based property
type DictCommand =
    | Add of key: int * value: string
    | Get of key: int
    | Remove of key: int
    | Contains of key: int

[<Property>]
let ``dictionary implementation matches model`` (commands: DictCommand list) =
    let model = ref Map.empty
    let impl = DictionaryImpl<int, string>()
    
    let mutable consistent = true
    
    for cmd in commands do
        match cmd with
        | Add (k, v) ->
            model := modelAdd k v !model
            impl.Add(k, v)
        
        | Get k ->
            let modelResult = modelGet k !model
            let implResult = impl.Get(k)
            consistent <- consistent && (modelResult = implResult)
        
        | Remove k ->
            model := modelRemove k !model
            impl.Remove(k)
        
        | Contains k ->
            let modelResult = modelContains k !model
            let implResult = impl.Contains(k)
            consistent <- consistent && (modelResult = implResult)
    
    // Final state should match
    consistent && (modelCount !model = impl.Count)
```

---

## 14. Best Practices สรุป

```fsharp
// BestPractices.fs
module Tests.BestPractices

open FsCheck
open FsCheck.Xunit

// 1. เริ่มจาก properties ง่ายๆ
[<Property>]
let ``easy property: abs never negative`` (n: int) =
    abs n >= 0

// 2. ใช้ preconditions กับ ==>
[<Property>]
let ``with precondition: sqrt is inverse of square for positive`` (n: int) =
    n > 0 ==> lazy (
        let squared = float n * float n
        abs (sqrt squared - float n) < 0.0001
    )

// 3. Label properties สำหรับ debugging
[<Property>]
let ``labeled property`` (a: int) (b: int) =
    let result = a + b
    result >= a |@ sprintf "a=%d, b=%d, result=%d" a b result

// 4. สร้าง custom Arbitrary สำหรับ domain types
type ValidAge = ValidAge of int
type ValidAgeArb =
    static member ValidAge() =
        Arb.from<int> 
        |> Arb.filter (fun n -> n >= 0 && n <= 150)
        |> Arb.convert ValidAge (fun (ValidAge n) -> n)

[<Property(Arbitrary = [| typeof<ValidAgeArb> |])>]
let ``valid age is in range`` (ValidAge age) =
    age >= 0 && age <= 150

// 5. Test algebraic laws
[<Property>]
let ``functor identity law for option`` (opt: int option) =
    Option.map id opt = opt

[<Property>]
let ``functor composition for option`` (opt: int option) =
    let f = fun x -> x + 1
    let g = fun x -> x * 2
    Option.map (f >> g) opt = (Option.map f >> Option.map g) opt

// 6. Test idempotent operations
[<Property>]
let ``sort is idempotent`` (lst: int list) =
    List.sort lst = List.sort (List.sort lst)

[<Property>]
let ``distinct is idempotent`` (lst: int list) =
    List.distinct lst = List.distinct (List.distinct lst)

// 7. Test oracle comparisons
let slowButCorrectSort lst = lst |> List.sort  // Reference implementation
let fastSort lst = lst |> List.sortDescending |> List.rev  // Optimized

[<Property>]
let ``fast sort matches slow sort`` (lst: int list) =
    fastSort lst = slowButCorrectSort lst
```

---

## สรุป (Summary)

FsCheck ช่วยให้เราค้นพบ bugs ที่ example-based tests มักพลาด:

1. **Properties** คือ invariants ที่ต้องเป็นจริงสำหรับทุก input
2. **Generators** สร้าง random test data
3. **Shrinking** หา smallest failing case
4. **Round-trip** ทดสอบ serialization/deserialization
5. **Algebraic** ทดสอบ mathematical laws
6. **Model-based** เปรียบ implementation กับ specification

```bash
# FsCheck generates 100 tests by default
# Can increase with:
[<Property(MaxTest = 1000)>]
let ``more thorough test`` (n: int) = abs n >= 0
```
