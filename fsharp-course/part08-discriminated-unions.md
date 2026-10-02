# Part 8 - ยูเนียนแบบแยกแยะ (Discriminated Unions)

## บทนำ

Discriminated Unions (DUs) เป็นหนึ่งใน features ที่ทรงพลังที่สุดของ F# DU ช่วยให้เราสร้าง type ที่แสดงค่าที่เป็น "หนึ่งในหลายรูปแบบ" พร้อม data ที่เหมาะสมสำหรับแต่ละรูปแบบ เป็น type-safe กว่า enums มาก และทำงานได้ดีมากกับ pattern matching

---

## 8.1 DU พื้นฐาน

### 8.1.1 Simple DU

```fsharp
// Discriminated Union: type ที่มีหลาย cases
type Color =
    | Red
    | Green
    | Blue

// ใช้งาน
let myColor = Red
let anotherColor = Blue

// Pattern matching
let colorName color =
    match color with
    | Red -> "แดง"
    | Green -> "เขียว"
    | Blue -> "น้ำเงิน"

printfn "Red = %s" (colorName Red)
printfn "Green = %s" (colorName Green)
printfn "Blue = %s" (colorName Blue)

// DU ใน list
let colors = [Red; Green; Blue; Red; Blue]
let reds = colors |> List.filter (fun c -> c = Red)
printfn "Red count: %d" reds.Length
```

### 8.1.2 DU กับ Data

```fsharp
// DU cases สามารถ carry data ได้
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

// สร้าง shapes
let circle = Circle 5.0
let rect = Rectangle(4.0, 3.0)
let tri = Triangle(6.0, 4.0)

// Pattern matching กับ data
let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(b, h) -> 0.5 * b * h

let perimeter shape =
    match shape with
    | Circle r -> 2.0 * System.Math.PI * r
    | Rectangle(w, h) -> 2.0 * (w + h)
    | Triangle(b, h) ->
        let hyp = sqrt (b * b + h * h)
        b + h + hyp

printfn "Circle area: %.4f" (area circle)
printfn "Rectangle area: %.2f" (area rect)
printfn "Triangle area: %.2f" (area tri)
printfn "Circle perimeter: %.4f" (perimeter circle)
```

---

## 8.2 DU กับ Record Data

```fsharp
// DU cases สามารถ carry records ได้

type Person = { Name: string; Age: int }

type UserRole =
    | Guest
    | RegisteredUser of person: Person
    | Admin of person: Person * permissions: string list
    | SuperAdmin

let describeUser role =
    match role with
    | Guest ->
        "Guest user - limited access"
    | RegisteredUser person ->
        sprintf "User: %s (age: %d)" person.Name person.Age
    | Admin(person, perms) ->
        sprintf "Admin: %s - permissions: %s" 
            person.Name 
            (String.concat ", " perms)
    | SuperAdmin ->
        "Super Administrator - full access"

let users = [
    Guest
    RegisteredUser { Name = "Alice"; Age = 30 }
    Admin ({ Name = "Bob"; Age = 35 }, ["read"; "write"; "delete"])
    SuperAdmin
]

users |> List.iter (fun u -> printfn "%s" (describeUser u))

// Check permissions
let canWrite role =
    match role with
    | Admin(_, perms) -> List.contains "write" perms
    | SuperAdmin -> true
    | _ -> false

users |> List.iter (fun u -> 
    printfn "%A can write: %b" u (canWrite u))
```

---

## 8.3 DU กับ Multiple Fields

```fsharp
// แต่ละ case สามารถมี fields ไม่เหมือนกัน

type PaymentMethod =
    | Cash
    | CreditCard of cardNumber: string * expiryDate: string * cvv: string
    | DebitCard of cardNumber: string * bankName: string
    | BankTransfer of accountNumber: string * bankCode: string * reference: string
    | PromptPay of phoneOrId: string
    | Cryptocurrency of walletAddress: string * currency: string

let maskString (s: string) =
    if s.Length <= 4 then String.replicate s.Length "*"
    else String.replicate (s.Length - 4) "*" + s.[s.Length-4..]

let describePayment method' =
    match method' with
    | Cash -> "Cash payment"
    | CreditCard(num, expiry, _) ->
        $"Credit card ending {maskString num}, expires {expiry}"
    | DebitCard(num, bank) ->
        $"Debit card from {bank}, ending {maskString num}"
    | BankTransfer(acct, bank, ref) ->
        $"Bank transfer to {maskString acct} ({bank}), ref: {ref}"
    | PromptPay id ->
        $"PromptPay to {maskString id}"
    | Cryptocurrency(wallet, currency) ->
        $"{currency} to {wallet.[..7]}..."

let payments = [
    Cash
    CreditCard("4532015112830366", "12/26", "123")
    PromptPay("0891234567")
    BankTransfer("1234567890", "SCB", "REF-2024-001")
    Cryptocurrency("0x742d35Cc6634C0532925a3b844Bc454e4438f44e", "ETH")
]

printfn "Payment methods:"
payments |> List.iter (fun p -> printfn "  %s" (describePayment p))
```

---

## 8.4 Nested DUs

```fsharp
// DUs สามารถ contain DUs อื่นได้

type Suit = Hearts | Diamonds | Clubs | Spades

type Rank =
    | Number of int  // 2-10
    | Jack
    | Queen
    | King
    | Ace

type Card =
    | FaceCard of suit: Suit * rank: Rank
    | Joker of color: string

let cardName card =
    let rankName = function
        | Number n -> string n
        | Jack -> "J"
        | Queen -> "Q"
        | King -> "K"
        | Ace -> "A"
    
    let suitSymbol = function
        | Hearts -> "♥"
        | Diamonds -> "♦"
        | Clubs -> "♣"
        | Spades -> "♠"
    
    match card with
    | FaceCard(suit, rank) ->
        rankName rank + suitSymbol suit
    | Joker color ->
        $"{color} Joker"

let cardValue card =
    match card with
    | FaceCard(_, Number n) -> n
    | FaceCard(_, Jack) | FaceCard(_, Queen) | FaceCard(_, King) -> 10
    | FaceCard(_, Ace) -> 11  // simplified
    | Joker _ -> 0

let hand = [
    FaceCard(Hearts, Ace)
    FaceCard(Spades, King)
    FaceCard(Diamonds, Number 7)
    Joker "Red"
]

printfn "Hand:"
hand |> List.iter (fun c -> 
    printfn "  %s (value: %d)" (cardName c) (cardValue c))

let handValue = hand |> List.sumBy cardValue
printfn "Hand total value: %d" handValue
```

---

## 8.5 Recursive DUs

### 8.5.1 Binary Tree

```fsharp
// Recursive DU สำหรับ tree structures

type BinaryTree<'a> =
    | Leaf
    | Node of value: 'a * left: BinaryTree<'a> * right: BinaryTree<'a>

// สร้าง tree
let empty = Leaf
let singleNode = Node(42, Leaf, Leaf)

// Simple BST:
//        5
//       / \
//      3   8
//     / \   \
//    1   4   9
let bst =
    Node(5,
        Node(3, Node(1, Leaf, Leaf), Node(4, Leaf, Leaf)),
        Node(8, Leaf, Node(9, Leaf, Leaf)))

// Tree operations
let rec insert value tree =
    match tree with
    | Leaf -> Node(value, Leaf, Leaf)
    | Node(v, left, right) ->
        if value < v then Node(v, insert value left, right)
        elif value > v then Node(v, left, insert value right)
        else tree  // duplicate, no change

let rec contains value tree =
    match tree with
    | Leaf -> false
    | Node(v, left, right) ->
        if value = v then true
        elif value < v then contains value left
        else contains value right

let rec inorder tree =
    match tree with
    | Leaf -> []
    | Node(v, left, right) -> inorder left @ [v] @ inorder right

let rec height tree =
    match tree with
    | Leaf -> 0
    | Node(_, left, right) -> 1 + max (height left) (height right)

let rec count tree =
    match tree with
    | Leaf -> 0
    | Node(_, left, right) -> 1 + count left + count right

// Test
let tree = [5; 3; 8; 1; 4; 9; 2; 7; 6] 
           |> List.fold (fun acc v -> insert v acc) Leaf

printfn "Inorder: %A" (inorder tree)
printfn "Contains 4: %b" (contains 4 tree)
printfn "Contains 10: %b" (contains 10 tree)
printfn "Height: %d" (height tree)
printfn "Count: %d" (count tree)
```

### 8.5.2 Expression Tree

```fsharp
// Expression evaluator ด้วย recursive DU

type Expr =
    | Lit of float
    | Var of string
    | Add of Expr * Expr
    | Sub of Expr * Expr
    | Mul of Expr * Expr
    | Div of Expr * Expr
    | Pow of Expr * Expr
    | Neg of Expr
    | Abs of Expr
    | Sqrt of Expr

type Env = Map<string, float>

let rec eval (env: Env) expr =
    match expr with
    | Lit n -> n
    | Var name ->
        match Map.tryFind name env with
        | Some v -> v
        | None -> failwith $"Undefined variable: {name}"
    | Add(a, b) -> eval env a + eval env b
    | Sub(a, b) -> eval env a - eval env b
    | Mul(a, b) -> eval env a * eval env b
    | Div(a, b) ->
        let divisor = eval env b
        if divisor = 0.0 then failwith "Division by zero"
        else eval env a / divisor
    | Pow(base', exp') -> (eval env base') ** (eval env exp')
    | Neg e -> -(eval env e)
    | Abs e -> abs (eval env e)
    | Sqrt e ->
        let v = eval env e
        if v < 0.0 then failwith "Cannot sqrt negative"
        else sqrt v

let rec prettyPrint expr =
    match expr with
    | Lit n -> string n
    | Var name -> name
    | Add(a, b) -> $"({prettyPrint a} + {prettyPrint b})"
    | Sub(a, b) -> $"({prettyPrint a} - {prettyPrint b})"
    | Mul(a, b) -> $"({prettyPrint a} * {prettyPrint b})"
    | Div(a, b) -> $"({prettyPrint a} / {prettyPrint b})"
    | Pow(a, b) -> $"({prettyPrint a}^{prettyPrint b})"
    | Neg e -> $"(-{prettyPrint e})"
    | Abs e -> $"|{prettyPrint e}|"
    | Sqrt e -> $"sqrt({prettyPrint e})"

// Test expressions
let env = Map.ofList [("x", 3.0); ("y", 4.0); ("z", -2.0)]

// (x^2 + y^2)
let hypotenuse = Sqrt(Add(Pow(Var "x", Lit 2.0), Pow(Var "y", Lit 2.0)))
printfn "%s = %.4f" (prettyPrint hypotenuse) (eval env hypotenuse)

// |z| * (x + y)
let expr2 = Mul(Abs(Var "z"), Add(Var "x", Var "y"))
printfn "%s = %.4f" (prettyPrint expr2) (eval env expr2)
```

---

## 8.6 Pattern Matching กับ DUs

```fsharp
// DUs สร้าง pattern ที่ exhaustive และ type-safe

type Result<'T, 'E> =
    | Ok of value: 'T
    | Error of error: 'E

// ทำงานกับ result
let divide (a: float) (b: float) =
    if b = 0.0 then Error "Division by zero"
    else Ok (a / b)

let processResult result =
    match result with
    | Ok value -> printfn "Success: %f" value
    | Error msg -> printfn "Error: %s" msg

processResult (divide 10.0 3.0)
processResult (divide 10.0 0.0)

// Chain results
let sqrt' x =
    if x < 0.0 then Error "Cannot sqrt negative"
    else Ok (sqrt x)

// Calculate sqrt(a/b)
let safeSqrtDivide a b =
    match divide a b with
    | Error e -> Error e
    | Ok quotient ->
        match sqrt' quotient with
        | Error e -> Error e
        | Ok result -> Ok result

printfn "\nsqrt(16/4) = %A" (safeSqrtDivide 16.0 4.0)
printfn "sqrt(16/0) = %A" (safeSqrtDivide 16.0 0.0)
printfn "sqrt(-16/4) = %A" (safeSqrtDivide -16.0 4.0)
```

---

## 8.7 DU เป็น State Machine

```fsharp
// DUs สำหรับ modeling state machines

type OrderStatus =
    | Pending
    | PaymentReceived
    | Processing
    | Shipped of trackingNumber: string
    | Delivered of deliveredAt: System.DateTime
    | Cancelled of reason: string
    | Refunded of amount: decimal

type OrderEvent =
    | PaymentConfirmed
    | StartProcessing
    | MarkShipped of trackingNumber: string
    | MarkDelivered
    | Cancel of reason: string
    | Refund of amount: decimal

// State transition
let transition (status: OrderStatus) (event: OrderEvent) : Result<OrderStatus, string> =
    match status, event with
    | Pending, PaymentConfirmed ->
        Ok PaymentReceived
    | PaymentReceived, StartProcessing ->
        Ok Processing
    | Processing, MarkShipped trackingNum ->
        Ok (Shipped trackingNum)
    | Shipped _, MarkDelivered ->
        Ok (Delivered System.DateTime.Now)
    | (Pending | PaymentReceived | Processing), Cancel reason ->
        Ok (Cancelled reason)
    | (Delivered _ | Shipped _), Refund amount ->
        Ok (Refunded amount)
    | status, event ->
        Error $"Invalid transition: {status} -> {event}"

// Simulate order lifecycle
let events = [
    PaymentConfirmed
    StartProcessing
    MarkShipped "TH1234567890"
    MarkDelivered
]

let mutable currentStatus = Pending

printfn "Order State Machine:"
printfn "Initial: %A" currentStatus

for event in events do
    match transition currentStatus event with
    | Ok newStatus ->
        printfn "Event: %A -> Status: %A" event newStatus
        currentStatus <- newStatus
    | Error msg ->
        printfn "Error: %s" msg
```

---

## 8.8 Option Type

```fsharp
// Option<'T> เป็น built-in DU ใน F#
// type Option<'T> = Some of 'T | None

// แสดงการใช้งาน Option
let safeDivide a b =
    if b = 0 then None
    else Some (a / b)

let result1 = safeDivide 10 2    // Some 5
let result2 = safeDivide 10 0    // None

match result1 with
| Some v -> printfn "Result: %d" v
| None -> printfn "Error"

// Option.map: แปลงค่าถ้ามี
let doubled = result1 |> Option.map (fun x -> x * 2)
printfn "Doubled: %A" doubled  // Some 10

// Option.bind: chain operations
let result3 = 
    safeDivide 100 5
    |> Option.bind (fun x -> safeDivide x 2)
    |> Option.bind (fun x -> safeDivide x 2)

printfn "Chain: %A" result3  // 100/5/2/2 = Some 12 (เป็น int division)

// Option.defaultValue
let value1 = result1 |> Option.defaultValue 0   // 5
let value2 = result2 |> Option.defaultValue 0   // 0 (default)
printfn "defaultValue: %d, %d" value1 value2

// Option.orElse
let value3 = result2 |> Option.orElse (Some 99)
printfn "orElse: %A" value3  // Some 99

// Option.filter
let bigResult = result1 |> Option.filter (fun x -> x > 3)
let smallResult = result1 |> Option.filter (fun x -> x > 10)
printfn "filter > 3: %A" bigResult    // Some 5
printfn "filter > 10: %A" smallResult // None

// Option.iter (perform side effect if Some)
result1 |> Option.iter (fun v -> printfn "Value exists: %d" v)
result2 |> Option.iter (fun v -> printfn "This won't print: %d" v)
```

---

## 8.9 Result Type

```fsharp
// Result<'T, 'E> เป็น built-in DU
// type Result<'T, 'E> = Ok of 'T | Error of 'E

// Custom error type
type ValidationError =
    | Required of fieldName: string
    | TooShort of fieldName: string * minLength: int
    | TooLong of fieldName: string * maxLength: int
    | InvalidFormat of fieldName: string * expectedFormat: string
    | OutOfRange of fieldName: string * min: int * max: int

type UserInput = {
    Username: string
    Email: string
    Age: string
    Password: string
}

// Validation functions
let validateUsername (name: string) : Result<string, ValidationError> =
    if name = "" then Error (Required "username")
    elif name.Length < 3 then Error (TooShort("username", 3))
    elif name.Length > 20 then Error (TooLong("username", 20))
    elif not (name |> Seq.forall (fun c -> System.Char.IsLetterOrDigit(c) || c = '_')) then
        Error (InvalidFormat("username", "letters, digits, underscore only"))
    else Ok name

let validateEmail (email: string) : Result<string, ValidationError> =
    if email = "" then Error (Required "email")
    elif not (email.Contains("@")) then Error (InvalidFormat("email", "must contain @"))
    elif not (email.Contains(".")) then Error (InvalidFormat("email", "must contain ."))
    else Ok email

let validateAge (ageStr: string) : Result<int, ValidationError> =
    match System.Int32.TryParse(ageStr) with
    | false, _ -> Error (InvalidFormat("age", "must be a number"))
    | true, age when age < 0 || age > 150 -> Error (OutOfRange("age", 0, 150))
    | true, age -> Ok age

let formatError = function
    | Required field -> $"{field} is required"
    | TooShort(field, min) -> $"{field} must be at least {min} characters"
    | TooLong(field, max) -> $"{field} must be at most {max} characters"
    | InvalidFormat(field, fmt) -> $"{field}: {fmt}"
    | OutOfRange(field, min, max) -> $"{field} must be between {min} and {max}"

// Test validation
let inputs = [
    { Username = "alice"; Email = "alice@example.com"; Age = "30"; Password = "pass123" }
    { Username = "ab"; Email = "invalid-email"; Age = "200"; Password = "pass" }
    { Username = ""; Email = "test@test.com"; Age = "25"; Password = "pass" }
]

printfn "Validation results:"
for input in inputs do
    printfn "\nInput: %A" input
    let nameResult = validateUsername input.Username
    let emailResult = validateEmail input.Email
    let ageResult = validateAge input.Age
    
    match nameResult, emailResult, ageResult with
    | Ok _, Ok _, Ok _ ->
        printfn "  ✓ All valid!"
    | r1, r2, r3 ->
        [r1 |> Result.mapError formatError
         r2 |> Result.mapError formatError
         r3 |> Result.mapError formatError]
        |> List.iter (fun r ->
            match r with
            | Error msg -> printfn "  ✗ %s" msg
            | Ok _ -> ())
```

---

## 8.10 Single-Case DUs สำหรับ Type Safety

```fsharp
// Single-case DU สร้าง distinct types จาก primitives
// ป้องกันการสลับ IDs หรือค่าที่มี type เหมือนกัน

type CustomerId = CustomerId of int
type OrderId = OrderId of int
type ProductId = ProductId of int

// ป้องกัน bug: ส่ง OrderId แทน CustomerId
let getCustomer (CustomerId id) =
    printfn "Getting customer with id: %d" id

let getOrder (OrderId id) =
    printfn "Getting order with id: %d" id

let customerId = CustomerId 42
let orderId = OrderId 42

getCustomer customerId   // OK
getOrder orderId         // OK
// getCustomer orderId   // Error! Type mismatch

// Unwrap value
let (CustomerId customerIdValue) = customerId
printfn "Customer ID: %d" customerIdValue

// More examples: domain-specific string types
type EmailAddress = EmailAddress of string
type PhoneNumber = PhoneNumber of string
type Url = Url of string

let sendEmail (EmailAddress email) message =
    printfn "Sending to %s: %s" email message

let email = EmailAddress "user@example.com"
let phone = PhoneNumber "081-234-5678"

sendEmail email "Hello!"
// sendEmail phone "Hello!"  // Error!

// Type-safe currency
type THB = THB of decimal
type USD = USD of decimal

let thbToUsd rate (THB amount) = USD (amount / rate)
let usdToThb rate (USD amount) = THB (amount * rate)

let myBalance = THB 35000.0M
let usdBalance = thbToUsd 35.0M myBalance
let (USD usdAmount) = usdBalance
printfn "Balance: $%.2f" usdAmount
```

---

## 8.11 DU กับ toString

```fsharp
// Override ToString สำหรับ DU

type Priority =
    | Critical
    | High
    | Medium
    | Low
    | None'
    
    override this.ToString() =
        match this with
        | Critical -> "⚡ Critical"
        | High -> "🔴 High"
        | Medium -> "🟡 Medium"
        | Low -> "🟢 Low"
        | None' -> "⬜ None"

type Task = {
    Title: string
    Priority: Priority
    Completed: bool
}

let tasks = [
    { Title = "Fix production bug"; Priority = Critical; Completed = false }
    { Title = "Write tests"; Priority = High; Completed = false }
    { Title = "Update docs"; Priority = Medium; Completed = true }
    { Title = "Refactor code"; Priority = Low; Completed = false }
]

printfn "Task List:"
tasks 
|> List.sortBy (fun t -> 
    match t.Priority with
    | Critical -> 0 | High -> 1 | Medium -> 2 | Low -> 3 | None' -> 4)
|> List.iter (fun task ->
    let status = if task.Completed then "✓" else "○"
    printfn "  [%s] %s - %O" status task.Title task.Priority)
```

---

## 8.12 DU vs Class Hierarchy

```fsharp
// เปรียบเทียบ DU กับ class hierarchy

// Class hierarchy (OOP style)
// Shape: Circle, Rectangle, Triangle
// ข้อเสีย:
// - ต้องใช้ virtual methods หรือ visitor pattern
// - ไม่มี exhaustiveness checking
// - boilerplate code มาก

// DU (FP style)
type Shape2D =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of sideA: float * sideB: float * sideC: float
    | Ellipse of semiMajor: float * semiMinor: float
    | RegularPolygon of sides: int * sideLength: float

// ข้อดีของ DU:
// 1. Exhaustiveness checking
// 2. ง่ายกว่ามาก
// 3. Pattern matching ที่ทรงพลัง
// 4. Immutable by default

let area2D shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(a, b, c) ->
        // Heron's formula
        let s = (a + b + c) / 2.0
        sqrt (s * (s - a) * (s - b) * (s - c))
    | Ellipse(a, b) -> System.Math.PI * a * b
    | RegularPolygon(n, l) ->
        (float n * l * l) / (4.0 * tan (System.Math.PI / float n))

let shapes = [
    Circle 5.0
    Rectangle(4.0, 6.0)
    Triangle(3.0, 4.0, 5.0)
    Ellipse(5.0, 3.0)
    RegularPolygon(6, 4.0)  // regular hexagon
]

printfn "Shape areas:"
shapes |> List.iter (fun s -> printfn "  %A: %.4f" s (area2D s))
```

---

## 8.13 DU สำหรับ Error Handling

```fsharp
// Railway-oriented programming ด้วย Result DU

type AppError =
    | NotFound of resource: string * id: string
    | Unauthorized of message: string
    | ValidationFailed of errors: string list
    | DatabaseError of message: string
    | ExternalServiceError of service: string * message: string

// Result type operations
let map f result =
    match result with
    | Ok v -> Ok (f v)
    | Error e -> Error e

let bind f result =
    match result with
    | Ok v -> f v
    | Error e -> Error e

let mapError f result =
    match result with
    | Ok v -> Ok v
    | Error e -> Error (f e)

// Simulated service functions
let getUserFromDb userId : Result<string, AppError> =
    match userId with
    | 1 -> Ok "Alice"
    | 2 -> Ok "Bob"
    | _ -> Error (NotFound("user", string userId))

let validateAge age : Result<int, AppError> =
    if age >= 18 && age <= 120 then Ok age
    else Error (ValidationFailed [$"Age must be 18-120, got {age}"])

let checkPermission (user: string) (action: string) : Result<unit, AppError> =
    match user with
    | "Admin" -> Ok ()
    | _ -> Error (Unauthorized $"User '{user}' not permitted to '{action}'")

// Railway-oriented pipeline
let processUserAction userId age action =
    getUserFromDb userId
    |> bind (fun user ->
        validateAge age
        |> map (fun _ -> user))
    |> bind (fun user ->
        checkPermission user action
        |> map (fun _ -> user))
    |> map (fun user -> $"Action '{action}' completed by {user}")

let formatAppError error =
    match error with
    | NotFound(resource, id) -> $"Not Found: {resource} with id '{id}'"
    | Unauthorized msg -> $"Unauthorized: {msg}"
    | ValidationFailed errors -> $"Validation Failed: {String.concat '; ' errors}"
    | DatabaseError msg -> $"Database Error: {msg}"
    | ExternalServiceError(svc, msg) -> $"External Service Error ({svc}): {msg}"

// Test
let tests = [
    (1, 25, "read")
    (1, 15, "read")    // age validation fail
    (99, 25, "read")   // user not found
    (2, 30, "admin")   // not authorized
]

printfn "Railway Oriented Programming:"
tests |> List.iter (fun (userId, age, action) ->
    match processUserAction userId age action with
    | Ok msg -> printfn "✓ %s" msg
    | Error err -> printfn "✗ %s" (formatAppError err))
```

---

## 8.14 ตัวอย่างโปรแกรมสมบูรณ์

### 8.14.1 Abstract Syntax Tree สำหรับ Mini Language

```fsharp
// mini_lang.fsx - Mini programming language evaluator

type Identifier = string

type Literal =
    | IntLit of int
    | FloatLit of float
    | StringLit of string
    | BoolLit of bool
    | NullLit

type Operator =
    | Plus | Minus | Multiply | Divide | Modulo
    | Equal | NotEqual | Less | LessOrEqual | Greater | GreaterOrEqual
    | And | Or | Not

type Statement =
    | Assign of variable: Identifier * value: Expression
    | If of condition: Expression * thenBranch: Statement list * elseBranch: Statement list
    | While of condition: Expression * body: Statement list
    | Print of Expression
    | Block of Statement list

and Expression =
    | Literal of Literal
    | Variable of Identifier
    | BinaryOp of Operator * Expression * Expression
    | UnaryOp of Operator * Expression
    | Call of functionName: Identifier * args: Expression list

type Environment = Map<Identifier, Literal>

// Evaluate expression
let rec evalExpr (env: Environment) expr =
    match expr with
    | Literal lit -> lit
    | Variable name ->
        match Map.tryFind name env with
        | Some value -> value
        | None -> failwith $"Undefined variable: {name}"
    | BinaryOp(op, left, right) ->
        let l = evalExpr env left
        let r = evalExpr env right
        evalBinaryOp op l r
    | UnaryOp(Not, expr) ->
        match evalExpr env expr with
        | BoolLit b -> BoolLit (not b)
        | _ -> failwith "NOT requires boolean"
    | _ -> failwith "Unsupported expression"

and evalBinaryOp op left right =
    match op, left, right with
    | Plus, IntLit a, IntLit b -> IntLit (a + b)
    | Plus, FloatLit a, FloatLit b -> FloatLit (a + b)
    | Plus, StringLit a, StringLit b -> StringLit (a + b)
    | Minus, IntLit a, IntLit b -> IntLit (a - b)
    | Multiply, IntLit a, IntLit b -> IntLit (a * b)
    | Divide, IntLit a, IntLit b -> 
        if b = 0 then failwith "Division by zero"
        else IntLit (a / b)
    | Equal, a, b -> BoolLit (a = b)
    | NotEqual, a, b -> BoolLit (a <> b)
    | Less, IntLit a, IntLit b -> BoolLit (a < b)
    | Greater, IntLit a, IntLit b -> BoolLit (a > b)
    | And, BoolLit a, BoolLit b -> BoolLit (a && b)
    | Or, BoolLit a, BoolLit b -> BoolLit (a || b)
    | _ -> failwith $"Invalid operation: {op} on {left} and {right}"

// Execute statement
let rec execStatement (env: Environment) stmt =
    match stmt with
    | Assign(name, expr) ->
        let value = evalExpr env expr
        Map.add name value env
        
    | Print expr ->
        let value = evalExpr env expr
        let str = match value with
                  | IntLit n -> string n
                  | FloatLit f -> string f
                  | StringLit s -> s
                  | BoolLit b -> string b
                  | NullLit -> "null"
        printfn "%s" str
        env
        
    | If(cond, thenBranch, elseBranch) ->
        match evalExpr env cond with
        | BoolLit true -> 
            thenBranch |> List.fold execStatement env
        | BoolLit false ->
            elseBranch |> List.fold execStatement env
        | _ -> failwith "If condition must be boolean"
        
    | While(cond, body) ->
        let mutable currentEnv = env
        let mutable continueLoop = true
        while continueLoop do
            match evalExpr currentEnv cond with
            | BoolLit true ->
                currentEnv <- body |> List.fold execStatement currentEnv
            | BoolLit false ->
                continueLoop <- false
            | _ -> failwith "While condition must be boolean"
        currentEnv
        
    | Block stmts ->
        stmts |> List.fold execStatement env

// Test program:
// x = 1
// while x <= 5:
//     print x
//     x = x + 1
let program = [
    Assign("x", Literal (IntLit 1))
    While(
        BinaryOp(LessOrEqual, Variable "x", Literal (IntLit 5)),
        [
            Print (Variable "x")
            Assign("x", BinaryOp(Plus, Variable "x", Literal (IntLit 1)))
        ]
    )
]

printfn "Running mini language program:"
let _ = program |> List.fold execStatement Map.empty
```

---

## สรุป Part 8

ในบทนี้เราได้เรียนรู้:
- ✅ DU พื้นฐาน: enum-like และ cases กับ data
- ✅ DU กับ record data
- ✅ DU กับ multiple fields ที่ต่างกัน
- ✅ Nested DUs
- ✅ Recursive DUs: trees และ lists
- ✅ Pattern matching กับ DUs
- ✅ DU เป็น state machine
- ✅ Result type (Ok/Error)
- ✅ Option type (Some/None)
- ✅ Expression evaluator
- ✅ Single-case DUs สำหรับ type safety
- ✅ DU vs class hierarchy
- ✅ ToString override
- ✅ Railway-oriented error handling

**ใน Part 9** เราจะเรียนรู้ Modules และ Namespaces สำหรับการจัดระเบียบโค้ด!
