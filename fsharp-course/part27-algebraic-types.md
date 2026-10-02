# Part 27 - ประเภทพีชคณิต (Algebraic Types in Depth)

## บทนำ (Introduction)

Algebraic Types (ประเภทพีชคณิต) เป็นแนวคิดพื้นฐานของ Type Theory ใน functional programming ประกอบด้วย:
- **Sum Types** (ผลรวมของประเภท): A + B
- **Product Types** (ผลคูณของประเภท): A × B

---

## 27.1 Sum Types (Discriminated Unions)

```fsharp
// ============ Sum Types ============
// Sum type หรือ Tagged Union เป็นประเภทที่มีได้หลาย variant
// ค่าของ Sum Type = หนึ่งใน variants เท่านั้น

// Bool = True | False  (2 values)
type MyBool = MyTrue | MyFalse

// Option<'T> = None | Some 'T
type MyOption<'T> = 
    | MyNone
    | MySome of 'T

// Result<'T,'E> = Ok 'T | Err 'E
type MyResult<'T, 'E> = 
    | MyOk of 'T
    | MyErr of 'E

// ============ Cardinality ============
// |Bool| = 2
// |Option<bool>| = 1 + 2 = 3 (None, Some True, Some False)
// |Option<unit>| = 1 + 1 = 2 (None, Some ())

// ============ DU กับข้อมูล ============
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base_: float * height: float

let area = function
    | Circle r -> System.Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(b, h) -> 0.5 * b * h

// ============ Recursive Sum Types ============
type List<'T> =
    | Nil
    | Cons of head: 'T * tail: List<'T>

let rec myList = Cons(1, Cons(2, Cons(3, Nil)))

let rec listLength = function
    | Nil -> 0
    | Cons(_, tail) -> 1 + listLength tail

printfn "Length: %d" (listLength myList)  // 3

// ============ Mutual Recursion ============
type Expr =
    | Lit of int
    | Add of Expr * Expr
    | If of BoolExpr * Expr * Expr
and BoolExpr =
    | BTrue
    | BFalse
    | And of BoolExpr * BoolExpr
    | Not of BoolExpr
    | Eq of Expr * Expr

let rec evalExpr = function
    | Lit n -> n
    | Add(e1, e2) -> evalExpr e1 + evalExpr e2
    | If(cond, t, f) -> if evalBool cond then evalExpr t else evalExpr f
and evalBool = function
    | BTrue -> true
    | BFalse -> false
    | And(b1, b2) -> evalBool b1 && evalBool b2
    | Not b -> not (evalBool b)
    | Eq(e1, e2) -> evalExpr e1 = evalExpr e2

let expr = If(Eq(Lit 1, Lit 1), Lit 42, Lit 0)
printfn "If (1=1) then 42 else 0 = %d" (evalExpr expr)  // 42
```

---

## 27.2 Product Types (Records and Tuples)

```fsharp
// ============ Product Types ============
// Product type เป็นประเภทที่มีทุก fields พร้อมกัน
// |A × B| = |A| × |B|

// ============ Tuples ============
// Tuple = anonymous product type
let pair : int * string = (42, "hello")
let triple : int * bool * float = (1, true, 3.14)

// Pattern matching
let (x, y) = pair
let (n, b, f) = triple
printfn "x=%d, y=%s" x y

// Unit type เป็น "empty product"
// |unit| = 1 (มีแค่ค่าเดียวคือ ())
let u : unit = ()

// ============ Records ============
// Record = named product type
type Point = { X: float; Y: float }
type RGB = { R: byte; G: byte; B: byte }
type Config = { 
    Host: string
    Port: int
    Timeout: float
    Debug: bool 
}

// ความสัมพันธ์: |Config| = |string| × |int| × |float| × |bool|
// (infinite × infinite × infinite × 2 = infinite)

let p = { X = 3.0; Y = 4.0 }
let color = { R = 255uy; G = 128uy; B = 0uy }

// ============ Nested Products ============
type Circle = { Center: Point; Radius: float }
type Line = { Start: Point; End: Point }

let circle = { Center = { X = 0.0; Y = 0.0 }; Radius = 5.0 }
printfn "Circle: %A" circle

// ============ Record Update Syntax ============
let movedCircle = { circle with Center = { X = 1.0; Y = 2.0 } }
printfn "Moved: %A" movedCircle

// ============ เปรียบเทียบ Sum vs Product ============
// Sum: A | B (either/or) - ขนาด |A| + |B|
// Product: A × B (both) - ขนาด |A| × |B|

// ตัวอย่าง: Payment method
type CreditCard = { Number: string; Expiry: string; CVV: string }
type BankAccount = { AccountId: string; RoutingNumber: string }

// Sum type: payment ต้องเลือกอย่างใดอย่างหนึ่ง
type Payment =
    | ByCard of CreditCard
    | ByBank of BankAccount
    | ByCash of amount: float

// Product type: order มีทุก fields
type Order = {
    Id: string
    Items: string list
    Total: float
    Payment: Payment  // embedded sum type
}
```

---

## 27.3 Unit Type as Identity

```fsharp
// ============ Unit Type ============
// unit เป็น identity element สำหรับ product
// A × unit ≅ A (isomorphic)

// unit ใช้สำหรับ side effects (functions ที่ return nothing meaningful)
let printAndReturn (x: 'T) : 'T =
    printfn "%A" x
    x

// Async unit = "fire and forget"
let fireAndForget () : Async<unit> =
    async { printfn "Doing something..." }

// ============ unit กับ Higher-Kinded patterns ============
// Thunk = unit -> 'T (delayed computation)
type Thunk<'T> = unit -> 'T

let lazyValue : Thunk<int> = fun () -> 
    printfn "Computing..."
    42

// Call only when needed
let forceThunk (thunk: Thunk<'T>) = thunk()
printfn "Value: %d" (forceThunk lazyValue)

// ============ unit กับ Type Algebra ============
// Option<unit> ≅ bool
// None = false
// Some () = true

let optionToBool : unit option -> bool = function
    | None -> false
    | Some () -> true

let boolToOption : bool -> unit option = function
    | false -> None
    | true -> Some ()

// ============ Void Type (F# ไม่มี Void แต่มี Never ผ่าน exception) ============
// ใน F# เราใช้ exn หรือ type ที่ไม่มี instance

// ============ Currying and unit ============
// f: unit -> 'T  ≅  'T  (value vs thunk)
// ใน F# functions เป็น values ด้วย

let constant x = fun () -> x
let alwaysOne = constant 1
let alwaysHello = constant "hello"

printfn "alwaysOne(): %d" (alwaysOne())     // 1
printfn "alwaysHello(): %s" (alwaysHello()) // hello
```

---

## 27.4 Type Algebra

```fsharp
// ============ Type Algebra ============

// ============ A × B (Product) ============
// Functions ที่รับ product type สามารถ curry ได้

// uncurried: A × B -> C
let addUncurried (x: int, y: int) = x + y

// curried: A -> B -> C  (เหมือนกัน แต่ flexible กว่า)
let addCurried (x: int) (y: int) = x + y

// curry / uncurry isomorphism
let curry f x y = f (x, y)
let uncurry f (x, y) = f x y

let addCurried2 = curry addUncurried
let addUncurried2 = uncurry addCurried

printfn "curry: %d" (addCurried2 3 4)        // 7
printfn "uncurry: %d" (addUncurried2 (3, 4)) // 7

// ============ A + B (Sum) ============
// Choice type = Either A B

type Either<'A, 'B> = Left of 'A | Right of 'B

let mapLeft f = function
    | Left a -> Left (f a)
    | Right b -> Right b

let mapRight f = function
    | Left a -> Left a
    | Right b -> Right (f b)

let bimap f g = function
    | Left a -> Left (f a)
    | Right b -> Right (g b)

let fold f g = function
    | Left a -> f a
    | Right b -> g b

// ============ Distributivity ============
// A × (B + C) ≅ (A × B) + (A × C)

// Distribute
let distribute : 'A * Either<'B, 'C> -> Either<'A * 'B, 'A * 'C> = function
    | (a, Left b) -> Left (a, b)
    | (a, Right c) -> Right (a, c)

// Factorize
let factorize : Either<'A * 'B, 'A * 'C> -> 'A * Either<'B, 'C> = function
    | Left (a, b) -> (a, Left b)
    | Right (a, c) -> (a, Right c)

// ============ Exponential Types (Functions) ============
// A -> B = B^A  (A ใน domain, B ใน codomain)
// |A -> B| = |B|^|A|

// bool -> bool: 2^2 = 4 functions
let f1 : bool -> bool = id
let f2 : bool -> bool = not
let f3 : bool -> bool = fun _ -> true
let f4 : bool -> bool = fun _ -> false

// ============ Newton's Law in Types ============
// |unit| = 1
// |A × unit| = |A| × 1 = |A|  (unit is identity for ×)
// |A + Empty| = |A| + 0 = |A|  (empty/never is identity for +)

// ============ Algebra laws ============
// Commutativity: A × B ≅ B × A
let swap (a, b) = (b, a)

// Associativity: (A × B) × C ≅ A × (B × C)  
let assoc1 ((a, b), c) = (a, (b, c))
let assoc2 (a, (b, c)) = ((a, b), c)
```

---

## 27.5 Isomorphisms Between Types

```fsharp
// ============ Type Isomorphism ============
// สองประเภทเป็น isomorphic เมื่อมี bijection ระหว่างกัน
// มี to และ from ที่ to >> from = id และ from >> to = id

type Iso<'A, 'B> = {
    To: 'A -> 'B
    From: 'B -> 'A
}

// ============ bool ≅ unit + unit ============
let boolToEither : bool -> Either<unit, unit> = function
    | false -> Left ()
    | true -> Right ()

let eitherToBool : Either<unit, unit> -> bool = function
    | Left () -> false
    | Right () -> true

let boolIso : Iso<bool, Either<unit, unit>> = {
    To = boolToEither
    From = eitherToBool
}

// ============ Option<'T> ≅ unit + 'T ============
let optionToEither : 'T option -> Either<unit, 'T> = function
    | None -> Left ()
    | Some x -> Right x

let eitherToOption : Either<unit, 'T> -> 'T option = function
    | Left () -> None
    | Right x -> Some x

// ============ List<'T> and Stream ============
// List ≅ 1 + T × List (either empty or head×tail)

// ============ A × (B + C) ≅ (A × B) + (A × C) ============
let distribute2 : 'A * ('B option) -> ('A * 'B) option = function
    | (a, Some b) -> Some (a, b)
    | (_, None) -> None

// ============ ตัวอย่างการใช้งาน: Validation ============
// Validated<'T,'E> ≅ 'T | 'E list (but collects errors)

type Validated<'T, 'E> =
    | Valid of 'T
    | Invalid of 'E list

let validateAge (age: int) : Validated<int, string> =
    if age < 0 then Invalid ["Age cannot be negative"]
    elif age > 150 then Invalid ["Age is unrealistically high"]
    else Valid age

let validateName (name: string) : Validated<string, string> =
    if String.length name < 2 then Invalid ["Name too short"]
    elif String.length name > 50 then Invalid ["Name too long"]
    else Valid name

// Applicative-style validation (collect all errors)
let (<*|>) (vf: Validated<'A -> 'B, 'E>) (va: Validated<'A, 'E>) : Validated<'B, 'E> =
    match vf, va with
    | Valid f, Valid a -> Valid (f a)
    | Invalid e1, Invalid e2 -> Invalid (e1 @ e2)
    | Invalid e, _ -> Invalid e
    | _, Invalid e -> Invalid e

let (<|>) (f: 'A -> 'B) (va: Validated<'A, 'E>) : Validated<'B, 'E> =
    match va with
    | Valid a -> Valid (f a)
    | Invalid e -> Invalid e

type UserData = { Name: string; Age: int }

let validateUser name age =
    let createUser n a = { Name = n; Age = a }
    createUser
    <|> validateName name
    <*|> validateAge age

printfn "Valid user: %A" (validateUser "Alice" 30)
printfn "Invalid user: %A" (validateUser "A" -5)  // Collects both errors
```

---

## 27.6 Phantom Types

```fsharp
// ============ Phantom Types ============
// Type parameters ที่ไม่ปรากฏใน runtime แต่ให้ type safety

// ============ Type-safe state machine ============
type Locked = Locked
type Unlocked = Unlocked

type Safe<'State> = private Safe of contents: string list

module Safe =
    let create () : Safe<Locked> = Safe []
    
    let unlock (password: string) (Safe contents : Safe<Locked>) : Safe<Unlocked> option =
        if password = "secret" then Some (Safe contents)
        else None
    
    let lock (Safe contents : Safe<Unlocked>) : Safe<Locked> = Safe contents
    
    let addItem (item: string) (Safe contents : Safe<Unlocked>) : Safe<Unlocked> =
        Safe (item :: contents)
    
    let removeItem (item: string) (Safe contents : Safe<Unlocked>) : Safe<Unlocked> =
        Safe (List.filter ((<>) item) contents)
    
    let viewContents (Safe contents : Safe<Unlocked>) : string list = contents

// Cannot call unlocked operations on locked safe - COMPILE ERROR!
let mySafe = Safe.create()

match Safe.unlock "secret" mySafe with
| Some unlocked ->
    let withItem = Safe.addItem "gold" unlocked
    let withMore = Safe.addItem "diamonds" withItem
    printfn "Contents: %A" (Safe.viewContents withMore)
    let locked = Safe.lock withMore
    // Safe.viewContents locked  // COMPILE ERROR! Type mismatch
    printfn "Safe locked"
| None ->
    printfn "Wrong password"

// ============ Type-safe Units ============
// ป้องกันการผสม units ที่ไม่ถูกต้อง

type Meter = Meter
type Kilogram = Kilogram
type Second = Second

type Quantity<'Unit>(value: float) =
    member _.Value = value
    member _.Add(other: Quantity<'Unit>) = Quantity<'Unit>(value + other.Value)
    member _.Subtract(other: Quantity<'Unit>) = Quantity<'Unit>(value - other.Value)
    member _.Scale(factor: float) = Quantity<'Unit>(value * factor)
    override _.ToString() = sprintf "%.2f" value

let meters (n: float) = Quantity<Meter>(n)
let kilograms (n: float) = Quantity<Kilogram>(n)
let seconds (n: float) = Quantity<Second>(n)

let height = meters 1.8
let weight = kilograms 75.0
let time = seconds 60.0

let doubleHeight = height.Scale(2.0)
let totalWeight = weight.Add(kilograms 5.0)
// height.Add(weight)  // COMPILE ERROR! Cannot add meters and kilograms

printfn "Double height: %A m" doubleHeight
printfn "Total weight: %A kg" totalWeight

// ============ Type-safe Database IDs ============
type UserId = UserId
type ProductId = ProductId
type OrderId = OrderId

type Id<'T>(value: int) =
    member _.Value = value
    override _.ToString() = sprintf "Id(%d)" value

let userId (n: int) = Id<UserId>(n)
let productId (n: int) = Id<ProductId>(n)

let getUserById (id: Id<UserId>) = sprintf "User %d" id.Value
let getProductById (id: Id<ProductId>) = sprintf "Product %d" id.Value

let uid = userId 42
let pid = productId 100

printfn "%s" (getUserById uid)
printfn "%s" (getProductById pid)
// getUserById pid  // COMPILE ERROR! Product ID used where User ID expected
```

---

## 27.7 Existential Types Concept

```fsharp
// ============ Existential Types ============
// ใช้สำหรับ hiding implementation details

// ============ Existential via interface ============
[<Interface>]
type IAnimal =
    abstract Sound: string
    abstract Move: unit -> string

type Dog() =
    interface IAnimal with
        member _.Sound = "Woof"
        member _.Move() = "Running"

type Bird() =
    interface IAnimal with
        member _.Sound = "Tweet"
        member _.Move() = "Flying"

type Fish() =
    interface IAnimal with
        member _.Sound = "..."
        member _.Move() = "Swimming"

// Existential: มี animal ที่รู้ว่า implement IAnimal แต่ไม่รู้ว่าชนิดใด
let animals : IAnimal list = [Dog(); Bird(); Fish()]

for animal in animals do
    printfn "%s - %s" animal.Sound (animal.Move())

// ============ Existential via Object Expressions ============
let makeCounter (start: int) =
    let mutable n = start
    { new System.Collections.Generic.IEnumerator<int> with
        member _.Current = n
        member _.MoveNext() = n <- n + 1; true
        member _.Reset() = n <- start
        member _.Dispose() = ()
      interface System.Collections.IEnumerator with
        member this.Current = box (this :> System.Collections.Generic.IEnumerator<int>).Current
        member this.MoveNext() = (this :> System.Collections.Generic.IEnumerator<int>).MoveNext()
        member this.Reset() = (this :> System.Collections.Generic.IEnumerator<int>).Reset() }

// ============ Existential via closures ============
// Hide internal state

type Counter = {
    Increment: unit -> unit
    Decrement: unit -> unit
    Value: unit -> int
    Reset: unit -> unit
}

let makeCounterClosures (initial: int) =
    let mutable value = initial
    {
        Increment = fun () -> value <- value + 1
        Decrement = fun () -> value <- value - 1
        Value = fun () -> value
        Reset = fun () -> value <- initial
    }

let counter = makeCounterClosures 0
counter.Increment()
counter.Increment()
counter.Increment()
counter.Decrement()
printfn "Counter value: %d" (counter.Value())  // 2
counter.Reset()
printfn "After reset: %d" (counter.Value())   // 0
```

---

## 27.8 GADTs Concept

```fsharp
// ============ Generalized Algebraic Data Types (GADTs) ============
// F# ไม่มี native GADT support แต่สามารถ simulate ได้

// ============ ตัวอย่าง: Type-safe expression ============
// ใน Haskell: data Expr a where
//   Lit  :: Int -> Expr Int
//   Bool :: Bool -> Expr Bool
//   Add  :: Expr Int -> Expr Int -> Expr Int
//   If   :: Expr Bool -> Expr a -> Expr a -> Expr a

// F# simulation using phantom types and interface
[<Interface>]
type IExpr<'T> =
    abstract Eval: unit -> 'T

type LitExpr(n: int) =
    interface IExpr<int> with
        member _.Eval() = n

type BoolLitExpr(b: bool) =
    interface IExpr<bool> with
        member _.Eval() = b

type AddExpr(left: IExpr<int>, right: IExpr<int>) =
    interface IExpr<int> with
        member _.Eval() = left.Eval() + right.Eval()

type IfExpr<'T>(cond: IExpr<bool>, ifTrue: IExpr<'T>, ifFalse: IExpr<'T>) =
    interface IExpr<'T> with
        member _.Eval() = if cond.Eval() then ifTrue.Eval() else ifFalse.Eval()

// ============ ใช้งาน ============
let expr1 : IExpr<int> = 
    IfExpr(
        BoolLitExpr(true),
        AddExpr(LitExpr(1), LitExpr(2)),
        LitExpr(0))

printfn "if true then 1+2 else 0 = %d" (expr1.Eval())  // 3

// Type safety: IfExpr requires both branches to have same type
// IfExpr(BoolLitExpr(true), LitExpr(1), BoolLitExpr(false))  // COMPILE ERROR

// ============ ตัวอย่าง: Type-safe stack ============
// Stack ที่รู้จักขนาดตัวเองในระดับ type

type Zero = Zero
type Succ<'N> = Succ

type Stack<'T, 'Size> = private Stack of 'T list

module TypedStack =
    let empty : Stack<'T, Zero> = Stack []
    
    let push (x: 'T) (Stack items : Stack<'T, 'N>) : Stack<'T, Succ<'N>> =
        Stack (x :: items)
    
    let pop (Stack items : Stack<'T, Succ<'N>>) : 'T * Stack<'T, 'N> =
        match items with
        | [] -> failwith "Impossible - type guarantees non-empty"
        | x :: rest -> (x, Stack rest)
    
    let toList (Stack items) = items

// ============ ใช้งาน ============
let s0 = TypedStack.empty
let s1 = TypedStack.push 1 s0
let s2 = TypedStack.push 2 s1
let s3 = TypedStack.push 3 s2

let (v1, s4) = TypedStack.pop s3
let (v2, s5) = TypedStack.pop s4
let (v3, s6) = TypedStack.pop s5
// TypedStack.pop s6  // COMPILE ERROR! Cannot pop empty stack

printfn "Popped: %d, %d, %d" v1 v2 v3
```

---

## 27.9 Type-Level Programming Basics

```fsharp
// ============ Type-Level Programming ============

// ============ 1. Marker Types ============
// Types ที่ไม่มีค่าแต่ใช้สำหรับ type checking

type ReadOnly = private ReadOnly
type ReadWrite = private ReadWrite

type Permission<'Mode> = private Permission

module Permission =
    let readOnly () : Permission<ReadOnly> = Permission
    let readWrite () : Permission<ReadWrite> = Permission
    
    // Downgrade permission
    let toReadOnly (_ : Permission<ReadWrite>) : Permission<ReadOnly> = Permission

let readData (_ : Permission<ReadOnly>) = "data"
let writeData (_ : Permission<ReadWrite>) = printfn "Writing..."
let readWithAny (_ : Permission<'T>) = "read ok"

let rwPerm = Permission.readWrite()
writeData rwPerm
readData (Permission.toReadOnly rwPerm)

// ============ 2. Newtype Pattern ============
// Wrapper type สำหรับ type safety

type EmailAddress = private EmailAddress of string
type PhoneNumber = private PhoneNumber of string
type NonEmptyString = private NonEmptyString of string

module EmailAddress =
    let create (s: string) : EmailAddress option =
        if s.Contains("@") && s.Contains(".") then Some (EmailAddress s)
        else None
    
    let value (EmailAddress s) = s

module NonEmptyString =
    let create (s: string) : NonEmptyString option =
        if String.length s > 0 then Some (NonEmptyString s)
        else None
    
    let value (NonEmptyString s) = s

let email = EmailAddress.create "alice@example.com"
let badEmail = EmailAddress.create "not-an-email"

printfn "Valid email: %A" (email |> Option.map EmailAddress.value)
printfn "Invalid email: %A" badEmail

// ============ 3. Builder Pattern with Types ============

type Incomplete = Incomplete
type Complete = Complete

type FormBuilder<'State> private (data: Map<string, string>) =
    member _.Data = data
    
    static member Start() : FormBuilder<Incomplete> = FormBuilder(Map.empty)
    
    member _.SetName(name: string) : FormBuilder<Incomplete> =
        FormBuilder(Map.add "name" name data)
    
    member _.SetEmail(email: string) : FormBuilder<Incomplete> =
        FormBuilder(Map.add "email" email data)
    
    // Submit only if both name and email are set
    member _.Submit() : FormBuilder<Complete> option =
        if Map.containsKey "name" data && Map.containsKey "email" data then
            Some (FormBuilder data)
        else
            None

type CompletedForm = { Name: string; Email: string }

let extractForm (form: FormBuilder<Complete>) =
    {
        Name = form.Data.["name"]
        Email = form.Data.["email"]
    }

let result = 
    FormBuilder.Start()
        .SetName("Alice")
        .SetEmail("alice@example.com")
        .Submit()
    |> Option.map extractForm

printfn "Form result: %A" result

// ============ 4. Effect Types ============
// Type ที่ encode side effects

type Pure<'T> = Pure of 'T
type IO<'T> = IO of (unit -> 'T)

module IO =
    let run (IO f) = f()
    let pure_ x = IO (fun () -> x)
    let map f (IO x) = IO (fun () -> f (x()))
    let bind (IO x) f = IO (fun () -> let v = x() in IO.run (f v))

let readLine = IO (fun () -> System.Console.ReadLine())
let writeLine s = IO (fun () -> printfn "%s" s)

let program = 
    IO.bind (writeLine "Enter your name:") (fun () ->
    IO.bind readLine (fun name ->
    writeLine (sprintf "Hello, %s!" name)))

// IO.run program  // ค่อยๆ execute เมื่อ run
printfn "IO program defined (not yet executed)"
```

---

## สรุป (Summary)

```
Algebraic Types ใน F#:

Sum Types (A + B):
- Discriminated Unions
- ค่าเป็น exactly หนึ่ง variant
- pattern matching ครอบคลุมทุก cases

Product Types (A × B):
- Records, Tuples
- ค่ามีทุก fields พร้อมกัน
- ขนาด = ผลคูณของทุก fields

Unit Type:
- Identity สำหรับ product
- ใช้สำหรับ side effects

Type Algebra:
- |A + B| = |A| + |B|
- |A × B| = |A| × |B|
- |A -> B| = |B|^|A|

Phantom Types:
- Type parameters ที่ไม่มีค่า runtime
- ใช้สำหรับ type safety

Existential Types:
- Hide implementation via interface
- OOP-style encapsulation

GADTs:
- F# simulate ด้วย phantom types + interfaces
- Type-safe expressions, stacks

Type-Level Programming:
- Marker types, newtypes
- Builder pattern
- Effect types
```

---

*จบ Part 27 - ประเภทพีชคณิต (Algebraic Types in Depth)*
