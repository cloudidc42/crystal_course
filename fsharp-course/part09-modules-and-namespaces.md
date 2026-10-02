# Part 9 - โมดูลและเนมสเปซ (Modules and Namespaces)

## บทนำ

การจัดระเบียบโค้ดเป็นสิ่งสำคัญมากในโปรเจกต์ขนาดใหญ่ F# ใช้ Modules และ Namespaces ในการจัดกลุ่มโค้ดที่เกี่ยวข้องกัน ทำให้โค้ดอ่านง่าย maintainable และ reusable บทนี้จะครอบคลุมทุกด้านของการจัดระเบียบโค้ดใน F#

---

## 9.1 module Keyword

### 9.1.1 Basic Module

```fsharp
// module กำหนด scope สำหรับ functions, types, values
module MathUtils

// ทุกอย่างใน module นี้เป็นส่วนหนึ่งของ MathUtils
let pi = System.Math.PI
let e = System.Math.E

let square x = x * x
let cube x = x * x * x
let power base' exp' = System.Math.Pow(base', exp')

let circleArea radius = pi * radius * radius
let sphereVolume radius = (4.0 / 3.0) * pi * power radius 3.0

// ใช้งาน
printfn "pi = %f" MathUtils.pi
printfn "circle area(5) = %f" (MathUtils.circleArea 5.0)
```

### 9.1.2 Module ใน File

```fsharp
// เมื่อไม่ได้ระบุ module name, ใช้ชื่อไฟล์เป็น module name
// ไฟล์ MyModule.fs จะสร้าง module MyModule โดยอัตโนมัติ

// แต่สามารถระบุ module name ชัดเจนได้
module Calculator

type Operator = Add | Sub | Mul | Div

let calculate op a b =
    match op with
    | Add -> a + b
    | Sub -> a - b
    | Mul -> a * b
    | Div ->
        if b = 0.0 then failwith "Division by zero"
        else a / b

let formatResult (a: float) op (b: float) =
    let opSymbol = match op with
                   | Add -> "+" | Sub -> "-" | Mul -> "×" | Div -> "÷"
    let result = calculate op a b
    sprintf "%.2f %s %.2f = %.2f" a opSymbol b result
```

### 9.1.3 Module กับ let rec

```fsharp
module Sequences

let rec factorial n =
    if n <= 1 then 1
    else n * factorial (n - 1)

let rec fibonacci n =
    match n with
    | 0 -> 0
    | 1 -> 1
    | n -> fibonacci (n - 1) + fibonacci (n - 2)

// เรียกใช้
printfn "5! = %d" (Sequences.factorial 5)
printfn "fib(10) = %d" (Sequences.fibonacci 10)
```

---

## 9.2 namespace Keyword

### 9.2.1 Basic Namespace

```fsharp
// namespace กำหนด hierarchical grouping
// แตกต่างจาก module ตรงที่:
// - namespace ไม่สามารถมี let bindings โดยตรง
// - namespace ใช้สำหรับ types และ modules

namespace MyApp.Domain

type ProductId = ProductId of int
type CustomerId = CustomerId of int

type Product = {
    Id: ProductId
    Name: string
    Price: decimal
}

type Customer = {
    Id: CustomerId
    Name: string
    Email: string
}
```

### 9.2.2 Namespace กับ Module

```fsharp
// ใช้ namespace เพื่อจัด modules ในลำดับชั้น
namespace MyApp.Services

module ProductService =
    
    let getById id =
        printfn "Getting product %d" id
        None  // placeholder

    let create name price =
        printfn "Creating product: %s @ %.2f" name price
        Ok ()

    let delete id =
        printfn "Deleting product %d" id
        Ok ()

module CustomerService =
    
    let getById id =
        printfn "Getting customer %d" id
        None

    let create name email =
        printfn "Creating customer: %s (%s)" name email
        Ok ()
```

---

## 9.3 Module vs Namespace

```fsharp
// Module:
// - มี let bindings ได้
// - สามารถ nested ได้
// - ใช้เป็น code organization unit หลัก
// - เป็น first-class (สามารถ open ได้)

// Namespace:
// - ไม่มี let bindings โดยตรง
// - ใช้สำหรับ hierarchical type organization
// - ไม่สามารถ open ได้ (เฉพาะ modules ใน namespace ต่างหาก)

// ตัวอย่างเปรียบเทียบ:

// Module - มี values และ functions
module StringUtils =
    let isEmpty (s: string) = s.Length = 0
    let isNotEmpty (s: string) = not (isEmpty s)
    let trim (s: string) = s.Trim()
    let toUpper (s: string) = s.ToUpper()
    let toLower (s: string) = s.ToLower()
    let contains (substring: string) (s: string) = s.Contains(substring)
    
    // Nested module
    module Validation =
        let isValidEmail (email: string) = 
            email.Contains("@") && email.Contains(".")
        let isValidPhone (phone: string) =
            phone |> Seq.forall (fun c -> System.Char.IsDigit(c) || c = '-')

// ใช้งาน module
let name = "  Hello World  "
printfn "Trimmed: '%s'" (StringUtils.trim name)
printfn "Upper: %s" (StringUtils.toUpper name)
printfn "Is valid email: %b" (StringUtils.Validation.isValidEmail "test@example.com")
```

---

## 9.4 Nested Modules

```fsharp
// Modules สามารถ nest ได้หลายชั้น

module MyApp

    module Infrastructure
        
        module Database
            let connect connectionString =
                printfn "Connecting to: %s" connectionString
                true
            
            let query sql =
                printfn "Executing: %s" sql
                []
        
        module Cache
            let mutable private storage = Map.empty<string, obj>
            
            let get key =
                Map.tryFind key storage
            
            let set key value =
                storage <- Map.add key value storage
            
            let invalidate key =
                storage <- Map.remove key storage
        
        module Http
            let get url =
                printfn "GET %s" url
                ""
            
            let post url body =
                printfn "POST %s with body: %s" url body
                ""
    
    module Domain
        
        type UserId = UserId of int
        type Email = Email of string
        
        module User
            type User = {
                Id: UserId
                Email: Email
                Name: string
            }
            
            let create id email name =
                { Id = UserId id; Email = Email email; Name = name }
            
            let formatUser user =
                let (UserId id) = user.Id
                let (Email email) = user.Email
                $"User[{id}]: {user.Name} <{email}>"

// ใช้งาน
let _ = MyApp.Infrastructure.Database.connect "localhost:5432"
let user = MyApp.Domain.User.create 1 "alice@example.com" "Alice"
printfn "%s" (MyApp.Domain.User.formatUser user)
```

---

## 9.5 open Statement

```fsharp
// open นำ module หรือ namespace มาใช้โดยไม่ต้อง qualify

module MathFunctions =
    let add x y = x + y
    let subtract x y = x - y
    let multiply x y = x * y

// โดยไม่มี open
let result1 = MathFunctions.add 3 4

// หลังจาก open
open MathFunctions

let result2 = add 3 4    // ไม่ต้อง qualify
let result3 = multiply 5 6

printfn "%d, %d, %d" result1 result2 result3

// open system modules
open System
open System.Collections.Generic

let list = List<int>()
let now = DateTime.Now
printfn "Now: %O" now

// open สามารถอยู่ใน local scope ได้
let formatDate (dt: DateTime) =
    open System.Globalization
    dt.ToString("dd MMMM yyyy", CultureInfo("th-TH"))

// ระวัง: open อาจทำให้เกิด name conflicts
// ถ้ามีหลาย modules ที่มีชื่อเหมือนกัน จะใช้ module ที่ open ล่าสุด
```

---

## 9.6 Module Aliases

```fsharp
// Module aliases ช่วยลด typing

// Long module name
module VeryLongModuleName =
    let doSomething x = x * 2
    let doAnotherThing x = x + 1

// Alias
module VLM = VeryLongModuleName

let r1 = VLM.doSomething 5
let r2 = VLM.doAnotherThing 5

printfn "%d, %d" r1 r2

// Common aliases ใน F# ecosystem
// module L = Microsoft.FSharp.Collections.List
// module A = Microsoft.FSharp.Collections.Array
// module S = Microsoft.FSharp.Collections.Seq

// Practical: collection module aliases
module List = Microsoft.FSharp.Collections.List

let numbers = [1; 2; 3; 4; 5]
let doubled = List.map (fun x -> x * 2) numbers
printfn "doubled: %A" doubled

// Domain-specific aliases
module Db =
    let query (sql: string) = printfn "Query: %s" sql
    let execute (sql: string) = printfn "Execute: %s" sql
    let transaction (actions: unit -> unit) = actions ()

// ใช้งาน
Db.query "SELECT * FROM users"
Db.execute "UPDATE users SET active = 1"
```

---

## 9.7 AutoOpen Attribute

```fsharp
// [<AutoOpen>] ทำให้ module ถูก open โดยอัตโนมัติ
// เมื่อ outer module/namespace ถูก open

[<AutoOpen>]
module CommonOperators

// Operators เหล่านี้จะพร้อมใช้เมื่อ open ไฟล์นี้
let (|>) x f = f x  // pipeline (already in F#)
let (>>?) f g = fun x ->
    match f x with
    | None -> None
    | Some y -> g y

// Option composition
let safeDiv a b = if b = 0 then None else Some (a / b)
let safeSqrt x = if x < 0.0 then None else Some (sqrt x)

let chainedOp =
    Some 16.0
    |> Option.bind (safeSqrt)

printfn "AutoOpen result: %A" chainedOp

// ใน real project:
// [<AutoOpen>]
// module Prelude =
//     // Common helpers ที่ใช้ทั่วทั้ง project
//     let inline (|>) x f = f x
//     let inline (<|) f x = f x
//     let tap f x = f x; x
```

---

## 9.8 RequireQualifiedAccess Attribute

```fsharp
// [<RequireQualifiedAccess>] บังคับให้ต้อง qualify เสมอ
// ป้องกัน name collision

[<RequireQualifiedAccess>]
module Direction

let North = "North"
let South = "South"
let East = "East"
let West = "West"

// ใช้งาน - ต้อง qualify เสมอ
printfn "Going %s" Direction.North
// open Direction  // Error! ไม่สามารถ open module ที่มี RequireQualifiedAccess

// ประโยชน์: ป้องกัน ambiguity
// ตัวอย่างที่ดีคือ List, Array, Seq modules ใน F#
// [<RequireQualifiedAccess>]
// module List =
//     let map f lst = ...  // ต้องใช้ List.map ไม่ใช่แค่ map

// Custom enum-like type ด้วย RequireQualifiedAccess
[<RequireQualifiedAccess>]
type Status =
    | Active
    | Inactive
    | Pending
    | Suspended

let checkStatus status =
    match status with
    | Status.Active -> "Active account"
    | Status.Inactive -> "Inactive account"
    | Status.Pending -> "Pending approval"
    | Status.Suspended -> "Account suspended"

printfn "%s" (checkStatus Status.Active)
```

---

## 9.9 Module Functions vs Type Methods

```fsharp
// สองแนวทางในการ organize functions:
// 1. Module functions (functional style)
// 2. Type methods (OOP style)

// Module function approach
module PersonModule =
    type Person = { Name: string; Age: int }
    
    let create name age = { Name = name; Age = age }
    let greet person = sprintf "Hello, %s!" person.Name
    let isAdult person = person.Age >= 18
    let birthday person = { person with Age = person.Age + 1 }

// Type method approach
type PersonWithMethods = {
    Name: string
    Age: int
} with
    member this.Greet() = sprintf "Hello, %s!" this.Name
    member this.IsAdult = this.Age >= 18
    member this.Birthday() = { this with Age = this.Age + 1 }
    
    static member Create(name, age) = { Name = name; Age = age }

// Module functions (preferred in F#)
let person1 = PersonModule.create "Alice" 30
printfn "%s" (PersonModule.greet person1)
printfn "Is adult: %b" (PersonModule.isAdult person1)

let olderPerson = PersonModule.birthday person1
printfn "After birthday: %d" olderPerson.Age

// Type methods
let person2 = PersonWithMethods.Create("Bob", 25)
printfn "%s" (person2.Greet())
printfn "Is adult: %b" person2.IsAdult

let olderPerson2 = person2.Birthday()
printfn "After birthday: %d" olderPerson2.Age

// ทั้งสองแบบสามารถผสมกันได้
// โดยทั่วไป: ใช้ module functions สำหรับ core logic
// และ methods สำหรับ convenience หรือ .NET interop
```

---

## 9.10 File Ordering ใน .fsproj

```fsharp
// ใน F#, ลำดับไฟล์ใน .fsproj มีความสำคัญมาก!
// ไฟล์สามารถใช้เฉพาะ types/functions ที่ defined ใน files ก่อนหน้า

// ตัวอย่าง .fsproj:
// <Compile Include="Domain/Types.fs" />       <!-- 1st: base types -->
// <Compile Include="Domain/Validation.fs" />  <!-- 2nd: uses Types -->
// <Compile Include="Services/UserService.fs" /> <!-- 3rd: uses both -->
// <Compile Include="Api/Handlers.fs" />       <!-- 4th: uses all above -->
// <Compile Include="Program.fs" />            <!-- Last: entry point -->

// Example: Domain/Types.fs
module Domain.Types

type UserId = UserId of int
type ProductId = ProductId of int

type User = {
    Id: UserId
    Name: string
    Email: string
}

// Example: Domain/Validation.fs
module Domain.Validation
// สามารถใช้ Domain.Types ได้เพราะอยู่ก่อน

let validateEmail (email: string) =
    if email.Contains("@") && email.Contains(".")
    then Ok email
    else Error "Invalid email format"

let validateName (name: string) =
    if name.Length >= 2 && name.Length <= 50
    then Ok name
    else Error "Name must be 2-50 characters"
```

---

## 9.11 Internal และ Private Functions

```fsharp
// F# ใช้ private สำหรับ implementation details

module BankAccount =
    // Private state
    let private mutable balance = 0.0
    
    // Private helper
    let private logTransaction transType amount =
        printfn "[LOG] %s: %.2f" transType amount
    
    // Private validation
    let private validateAmount amount =
        if amount <= 0.0 then Error "Amount must be positive"
        else Ok amount
    
    // Public interface
    let deposit amount =
        match validateAmount amount with
        | Error e -> Error e
        | Ok a ->
            balance <- balance + a
            logTransaction "DEPOSIT" a
            Ok balance
    
    let withdraw amount =
        match validateAmount amount with
        | Error e -> Error e
        | Ok a ->
            if a > balance then Error "Insufficient funds"
            else
                balance <- balance - a
                logTransaction "WITHDRAWAL" a
                Ok balance
    
    let getBalance () = balance

// ใช้งาน
BankAccount.deposit 1000.0 |> ignore
BankAccount.deposit 500.0 |> ignore
BankAccount.withdraw 200.0 |> ignore

printfn "Balance: %.2f" (BankAccount.getBalance ())

// BankAccount.logTransaction  // Error: private
// BankAccount.balance         // Error: private
```

---

## 9.12 Module Signatures (.fsi Files)

```fsharp
// .fsi files กำหนด public interface ของ module
// ช่วย encapsulate implementation details

// MyModule.fsi (signature file)
// module MyModule
// 
// val add : int -> int -> int
// val multiply : int -> int -> int
// // ไม่มี helper ที่ไม่ต้องการ expose

// MyModule.fs (implementation)
// module MyModule
// 
// let private helper x = x + 1  // ไม่อยู่ใน .fsi = private
// let add x y = x + y
// let multiply x y = x * y

// ตัวอย่างสมบูรณ์
module PublicAPI =
    // Public types
    type User = { Name: string; Email: string }
    type Error = NotFound | Unauthorized | ValidationError of string
    
    // Public functions
    let createUser name email : Result<User, Error> =
        if name = "" then Error (ValidationError "Name required")
        elif not (email.Contains("@")) then Error (ValidationError "Invalid email")
        else Ok { Name = name; Email = email }
    
    let getUserName (user: User) = user.Name
    let getUserEmail (user: User) = user.Email

// ใช้งาน
match PublicAPI.createUser "Alice" "alice@example.com" with
| Ok user -> printfn "Created: %s" (PublicAPI.getUserName user)
| Error (PublicAPI.ValidationError msg) -> printfn "Error: %s" msg
| Error _ -> printfn "Unknown error"
```

---

## 9.13 Module Patterns

### 9.13.1 Builder Pattern

```fsharp
// Builder pattern ด้วย module functions

type QueryBuilder = {
    Table: string
    Conditions: string list
    OrderBy: string option
    Limit: int option
    Offset: int option
}

module Query =
    let from table = {
        Table = table
        Conditions = []
        OrderBy = None
        Limit = None
        Offset = None
    }
    
    let where condition query =
        { query with Conditions = condition :: query.Conditions }
    
    let orderBy column query =
        { query with OrderBy = Some column }
    
    let limit n query =
        { query with Limit = Some n }
    
    let offset n query =
        { query with Offset = Some n }
    
    let build query =
        let conditions = 
            if query.Conditions = [] then ""
            else " WHERE " + String.concat " AND " (List.rev query.Conditions)
        
        let orderBy = 
            match query.OrderBy with
            | Some col -> $" ORDER BY {col}"
            | None -> ""
        
        let limit =
            match query.Limit with
            | Some n -> $" LIMIT {n}"
            | None -> ""
        
        let offset =
            match query.Offset with
            | Some n -> $" OFFSET {n}"
            | None -> ""
        
        $"SELECT * FROM {query.Table}{conditions}{orderBy}{limit}{offset}"

// ใช้งาน
let sql =
    Query.from "users"
    |> Query.where "age > 18"
    |> Query.where "active = 1"
    |> Query.orderBy "name"
    |> Query.limit 20
    |> Query.offset 40
    |> Query.build

printfn "SQL: %s" sql
```

### 9.13.2 Repository Pattern

```fsharp
// Repository pattern ด้วย module functions

type User = {
    Id: int
    Name: string
    Email: string
    Active: bool
}

// In-memory repository (สำหรับ testing)
module UserRepository =
    let private mutable users = [
        { Id = 1; Name = "Alice"; Email = "alice@example.com"; Active = true }
        { Id = 2; Name = "Bob"; Email = "bob@example.com"; Active = false }
        { Id = 3; Name = "Charlie"; Email = "charlie@example.com"; Active = true }
    ]
    
    let private nextId () = 
        if users = [] then 1
        else (users |> List.maxBy (fun u -> u.Id)).Id + 1
    
    let getAll () = users
    
    let getById id =
        users |> List.tryFind (fun u -> u.Id = id)
    
    let getActive () =
        users |> List.filter (fun u -> u.Active)
    
    let create name email =
        let newUser = { Id = nextId(); Name = name; Email = email; Active = true }
        users <- newUser :: users
        newUser
    
    let update id updateFn =
        users <- users |> List.map (fun u ->
            if u.Id = id then updateFn u else u)
        getById id
    
    let deactivate id =
        update id (fun u -> { u with Active = false })
    
    let delete id =
        users <- users |> List.filter (fun u -> u.Id <> id)

// ใช้งาน
printfn "All users: %d" (UserRepository.getAll () |> List.length)

let newUser = UserRepository.create "Diana" "diana@example.com"
printfn "Created: %A" newUser

match UserRepository.getById 1 with
| Some user -> printfn "Found: %s" user.Name
| None -> printfn "Not found"

UserRepository.deactivate 2 |> ignore
printfn "Active users: %d" (UserRepository.getActive () |> List.length)
```

---

## 9.14 ตัวอย่างโปรแกรมสมบูรณ์

### 9.14.1 Modular Task Management System

```fsharp
// task_system.fsx - ระบบจัดการ tasks แบบ modular

// Domain Types
module TaskDomain =
    type TaskId = TaskId of int
    type UserId = UserId of int
    
    type Priority = Low | Medium | High | Critical
    
    type TaskStatus = 
        | Todo
        | InProgress of startedAt: System.DateTime
        | Done of completedAt: System.DateTime
        | Cancelled of reason: string
    
    type Tag = Tag of string
    
    type Task = {
        Id: TaskId
        Title: string
        Description: string
        Priority: Priority
        Status: TaskStatus
        AssignedTo: UserId option
        Tags: Tag list
        DueDate: System.DateTime option
        CreatedAt: System.DateTime
    }

// Validation
module TaskValidation =
    open TaskDomain
    
    type ValidationError =
        | EmptyTitle
        | TitleTooLong of maxLength: int
        | InvalidDueDate of reason: string
    
    let validateTitle (title: string) =
        if title.Trim() = "" then Error EmptyTitle
        elif title.Length > 200 then Error (TitleTooLong 200)
        else Ok (title.Trim())
    
    let validateDueDate (date: System.DateTime option) =
        match date with
        | None -> Ok None
        | Some d when d < System.DateTime.Now ->
            Error (InvalidDueDate "Due date cannot be in the past")
        | Some d -> Ok (Some d)

// Repository
module TaskRepository =
    open TaskDomain
    
    let private mutable tasks: Task list = []
    let private mutable nextId = 1
    
    let private getNextId () =
        let id = nextId
        nextId <- nextId + 1
        TaskId id
    
    let add (task: Task) =
        let taskWithId = { task with Id = getNextId () }
        tasks <- taskWithId :: tasks
        taskWithId
    
    let getAll () = tasks
    
    let getById (TaskId id) =
        tasks |> List.tryFind (fun t -> t.Id = TaskId id)
    
    let getByStatus status =
        tasks |> List.filter (fun t -> t.Status = status)
    
    let getByPriority priority =
        tasks |> List.filter (fun t -> t.Priority = priority)
    
    let update (TaskId id) updateFn =
        let mutable found = false
        tasks <- tasks |> List.map (fun t ->
            if t.Id = TaskId id then
                found <- true
                updateFn t
            else t)
        if found then Ok ()
        else Error $"Task {id} not found"

// Service
module TaskService =
    open TaskDomain
    open TaskValidation
    open TaskRepository
    
    let createTask title description priority dueDate =
        match validateTitle title, validateDueDate dueDate with
        | Ok validTitle, Ok validDue ->
            let task = {
                Id = TaskId 0  // will be replaced
                Title = validTitle
                Description = description
                Priority = priority
                Status = Todo
                AssignedTo = None
                Tags = []
                DueDate = validDue
                CreatedAt = System.DateTime.Now
            }
            Ok (add task)
        | Error e, _ -> Error (string e)
        | _, Error e -> Error (string e)
    
    let startTask taskId =
        update taskId (fun t ->
            { t with Status = InProgress System.DateTime.Now })
    
    let completeTask taskId =
        update taskId (fun t ->
            { t with Status = Done System.DateTime.Now })
    
    let cancelTask taskId reason =
        update taskId (fun t ->
            { t with Status = Cancelled reason })
    
    let assignTask taskId userId =
        update taskId (fun t ->
            { t with AssignedTo = Some userId })
    
    let addTag taskId tag =
        update taskId (fun t ->
            { t with Tags = Tag tag :: t.Tags })

// Display
module TaskDisplay =
    open TaskDomain
    
    let formatPriority = function
        | Low -> "⬇ Low"
        | Medium -> "➡ Med"
        | High -> "⬆ High"
        | Critical -> "🔥 CRIT"
    
    let formatStatus = function
        | Todo -> "[ ]"
        | InProgress _ -> "[~]"
        | Done _ -> "[✓]"
        | Cancelled _ -> "[✗]"
    
    let printTask (task: Task) =
        let (TaskId id) = task.Id
        printfn "%s #%d: %s [%s]" 
            (formatStatus task.Status) 
            id 
            task.Title 
            (formatPriority task.Priority)
    
    let printTaskList (title: string) (tasks: Task list) =
        printfn "\n=== %s ===" title
        if tasks = [] then printfn "  (empty)"
        else tasks |> List.iter printTask

// Main program
open TaskDomain
open TaskService
open TaskDisplay

// Create tasks
let tasks = [
    createTask "Fix login bug" "Users can't log in" Critical None
    createTask "Write documentation" "Update API docs" Low None
    createTask "Implement search" "Full text search feature" High None
    createTask "Code review" "Review PR #42" Medium None
]

// Process results
let createdTasks = tasks |> List.choose (function Ok t -> Some t | _ -> None)
printfn "Created %d tasks" createdTasks.Length

// Perform operations
match createdTasks with
| [t1; t2; t3; t4] ->
    startTask t1.Id |> ignore
    startTask t3.Id |> ignore
    completeTask t2.Id |> ignore
    cancelTask t4.Id "Not needed" |> ignore
| _ -> ()

// Display
printTaskList "All Tasks" (TaskRepository.getAll ())
printTaskList "Todo Tasks" (TaskRepository.getByStatus Todo)
printTaskList "Critical Tasks" (TaskRepository.getByPriority Critical)
```

---

## สรุป Part 9

ในบทนี้เราได้เรียนรู้:
- ✅ module keyword สำหรับ code organization
- ✅ namespace keyword สำหรับ hierarchical grouping
- ✅ Module vs Namespace ความแตกต่างและเมื่อไหรใช้อะไร
- ✅ Nested modules
- ✅ open statement
- ✅ Module aliases
- ✅ [<AutoOpen>] attribute
- ✅ [<RequireQualifiedAccess>] attribute
- ✅ Module functions vs type methods
- ✅ File ordering ใน .fsproj
- ✅ Internal/private functions
- ✅ Module signatures (.fsi files)
- ✅ Module patterns: Builder, Repository

**ใน Part 10** เราจะเรียนรู้ Option และ Result types อย่างลึกซึ้ง ซึ่งเป็น core ของการจัดการ absence และ errors ใน F#!
