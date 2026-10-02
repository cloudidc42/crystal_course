# Part 32 - อินเทอร์เฟซ (Interfaces)

## บทนำ (Introduction)

Interface ใน F# คือสัญญา (contract) ที่กำหนดว่าคลาสต้องมี members อะไรบ้าง โดยไม่ระบุการ implement รายละเอียด Interface เป็นเครื่องมือสำคัญในการเขียน code ที่ยืดหยุ่นและทดสอบได้

Interfaces in F# are contracts that define what members a class must have, without specifying implementation details. They are essential tools for writing flexible and testable code.

---

## 1. Interface Definition (การนิยาม Interface)

```fsharp
// Interface พื้นฐาน
type IShape =
    abstract member Area: float
    abstract member Perimeter: float
    abstract member Name: string

// Interface ที่มี methods
type IMovable =
    abstract member Move: dx: float * dy: float -> unit
    abstract member Position: float * float

// Interface ที่มี generic type
type IRepository<'T> =
    abstract member GetById: id: int -> 'T option
    abstract member GetAll: unit -> 'T list
    abstract member Add: item: 'T -> unit
    abstract member Update: item: 'T -> bool
    abstract member Delete: id: int -> bool
    abstract member Count: int
```

---

## 2. Implementing Interfaces (การ Implement Interface)

```fsharp
type IShape =
    abstract member Area: float
    abstract member Perimeter: float
    abstract member Name: string

// Implement IShape ด้วย Circle
type Circle(radius: float) =
    interface IShape with
        member this.Area = System.Math.PI * radius * radius
        member this.Perimeter = 2.0 * System.Math.PI * radius
        member this.Name = "Circle"
    
    member this.Radius = radius
    
    override this.ToString() =
        sprintf "Circle(r=%.2f)" radius

// Implement IShape ด้วย Rectangle
type Rectangle(width: float, height: float) =
    interface IShape with
        member this.Area = width * height
        member this.Perimeter = 2.0 * (width + height)
        member this.Name = "Rectangle"
    
    member this.Width = width
    member this.Height = height
    
    override this.ToString() =
        sprintf "Rectangle(%.2f x %.2f)" width height

// Implement IShape ด้วย Triangle
type Triangle(a: float, b: float, c: float) =
    do
        if a + b <= c || b + c <= a || a + c <= b then
            failwith "Triangle inequality violated"
    
    interface IShape with
        member this.Area =
            let s = (a + b + c) / 2.0
            sqrt (s * (s-a) * (s-b) * (s-c))  // Heron's formula
        member this.Perimeter = a + b + c
        member this.Name = "Triangle"
    
    override this.ToString() =
        sprintf "Triangle(%.2f, %.2f, %.2f)" a b c

// ใช้ Interface
let shapes: IShape list = [
    Circle(5.0) :> IShape
    Rectangle(4.0, 3.0) :> IShape
    Triangle(3.0, 4.0, 5.0) :> IShape
]

printfn "Shapes:"
for shape in shapes do
    printfn "  %s: Area=%.2f, Perimeter=%.2f" shape.Name shape.Area shape.Perimeter

let totalArea = shapes |> List.sumBy (fun s -> s.Area)
printfn "Total area: %.2f" totalArea
```

---

## 3. Multiple Interface Implementation

```fsharp
type IDrawable =
    abstract member Draw: unit -> string

type IResizable =
    abstract member Resize: factor: float -> unit

type ISaveable =
    abstract member Save: filename: string -> unit

// Implement หลาย Interface พร้อมกัน
type Canvas(width: float, height: float) =
    let mutable _width = width
    let mutable _height = height
    let mutable _elements: string list = []
    
    interface IDrawable with
        member this.Draw() =
            let content = _elements |> String.concat "\n  "
            sprintf "Canvas(%.0fx%.0f):\n  %s" _width _height content
    
    interface IResizable with
        member this.Resize(factor) =
            _width <- _width * factor
            _height <- _height * factor
            printfn "Resized to %.0fx%.0f" _width _height
    
    interface ISaveable with
        member this.Save(filename) =
            printfn "Saving canvas to %s..." filename
    
    member this.AddElement(element: string) =
        _elements <- element :: _elements
    
    member this.Width = _width
    member this.Height = _height

let canvas = Canvas(800.0, 600.0)
canvas.AddElement("Circle at (100, 100)")
canvas.AddElement("Rectangle at (200, 200)")
canvas.AddElement("Text: Hello World")

// ใช้ผ่าน interface
let drawable = canvas :> IDrawable
printfn "%s" (drawable.Draw())

let resizable = canvas :> IResizable
resizable.Resize(1.5)

let saveable = canvas :> ISaveable
saveable.Save("artwork.png")

// Pattern matching กับ interfaces
let processObject (obj: obj) =
    match obj with
    | :? IDrawable as d -> printfn "Drawing: %s" (d.Draw())
    | :? ISaveable as s -> s.Save("output.dat")
    | _ -> printfn "Unknown object type"
```

---

## 4. Explicit Interface Implementation

```fsharp
// ใน F# ทุก interface implementation เป็น explicit โดยค่าเริ่มต้น
// ต้อง upcast เพื่อใช้ interface members

type IAnimal =
    abstract member Speak: unit -> string
    abstract member Name: string

type IPet =
    abstract member Name: string  // ambiguous กับ IAnimal.Name
    abstract member Owner: string

type Dog(name: string, owner: string) =
    interface IAnimal with
        member this.Speak() = "Woof!"
        member this.Name = name
    
    interface IPet with
        member this.Name = name
        member this.Owner = owner
    
    // Own member ไม่ใช่ interface member
    member this.Name = name
    member this.Fetch() = printfn "%s fetches!" name

let dog = Dog("Rex", "John")

// ต้อง cast เพื่อใช้ interface members
printfn "IAnimal.Name: %s" ((dog :> IAnimal).Name)
printfn "IPet.Name: %s" ((dog :> IPet).Name)
printfn "IPet.Owner: %s" ((dog :> IPet).Owner)
printfn "IAnimal.Speak: %s" ((dog :> IAnimal).Speak())

// Direct member access ไม่ต้อง cast
dog.Fetch()
printfn "Dog.Name: %s" dog.Name
```

---

## 5. Default Interface Methods (F# 5.0+)

```fsharp
// F# รองรับ default interface methods จาก .NET 8+
// แต่โดยทั่วไปใช้ abstract class หรือ function แทน

type ILogger =
    abstract member Log: level: string -> message: string -> unit
    
    // Default implementation ด้วย member ที่เรียก abstract member
    abstract member LogInfo: message: string -> unit
    abstract member LogWarning: message: string -> unit
    abstract member LogError: message: string -> unit

type ConsoleLogger(prefix: string) =
    interface ILogger with
        member this.Log(level) message =
            let timestamp = System.DateTime.Now.ToString("HH:mm:ss")
            printfn "[%s][%s] %s: %s" timestamp level prefix message
        
        member this.LogInfo(msg) = (this :> ILogger).Log "INFO" msg
        member this.LogWarning(msg) = (this :> ILogger).Log "WARN" msg
        member this.LogError(msg) = (this :> ILogger).Log "ERROR" msg

type FileLogger(filename: string) =
    interface ILogger with
        member this.Log(level) message =
            let line = sprintf "[%s] %s: %s\n" level (System.DateTime.Now.ToString()) message
            System.IO.File.AppendAllText(filename, line)
        
        member this.LogInfo(msg) = (this :> ILogger).Log "INFO" msg
        member this.LogWarning(msg) = (this :> ILogger).Log "WARN" msg
        member this.LogError(msg) = (this :> ILogger).Log "ERROR" msg

let logger: ILogger = ConsoleLogger("APP") :> ILogger
logger.LogInfo "Application started"
logger.LogWarning "Low memory"
logger.LogError "Connection failed"
```

---

## 6. IDisposable Pattern

```fsharp
// IDisposable - สำหรับจัดการ unmanaged resources
type ResourceHandle(name: string) =
    let mutable disposed = false
    
    do printfn "Acquiring resource: %s" name
    
    interface System.IDisposable with
        member this.Dispose() =
            if not disposed then
                printfn "Releasing resource: %s" name
                disposed <- true
    
    member this.Use() =
        if disposed then failwith "Resource has been disposed"
        printfn "Using resource: %s" name

// DatabaseConnection ที่ Implement IDisposable
type DatabaseConnection(connectionString: string) =
    let mutable isOpen = false
    let mutable disposed = false
    
    let checkDisposed () =
        if disposed then
            raise (System.ObjectDisposedException("DatabaseConnection"))
    
    do
        printfn "Opening connection to: %s" connectionString
        isOpen <- true
    
    interface System.IDisposable with
        member this.Dispose() =
            if not disposed then
                if isOpen then
                    printfn "Closing connection"
                    isOpen <- false
                disposed <- true
    
    member this.ExecuteQuery(sql: string) =
        checkDisposed()
        if not isOpen then failwith "Connection is not open"
        printfn "Executing: %s" sql
        sprintf "Results from: %s" sql
    
    member this.IsOpen = isOpen

// ใช้ use keyword (เทียบเท่า using ใน C#)
printfn "=== Using use keyword ==="
use conn = new DatabaseConnection("Server=localhost;DB=test")
let result = conn.ExecuteQuery("SELECT * FROM users")
printfn "Result: %s" result
// conn.Dispose() จะถูกเรียกอัตโนมัติเมื่อออกจาก scope

printfn "\nAfter use block"

// Manual disposal
printfn "\n=== Manual disposal ==="
let resource = new ResourceHandle("FileHandle")
try
    resource.Use()
    resource.Use()
finally
    (resource :> System.IDisposable).Dispose()
```

---

## 7. use Keyword for IDisposable

```fsharp
// use keyword ทำงานเหมือน using statement ใน C#
let processFile (filename: string) =
    use stream = System.IO.File.OpenRead(filename)
    use reader = new System.IO.StreamReader(stream)
    
    let content = reader.ReadToEnd()
    printfn "File has %d characters" content.Length
    content

// use ใน computation expressions
let readLines (filename: string) = seq {
    use reader = System.IO.File.OpenText(filename)
    let mutable line = reader.ReadLine()
    while not (isNull line) do
        yield line
        line <- reader.ReadLine()
}

// Nested use
type TempFile() =
    let filename = System.IO.Path.GetTempFileName()
    
    do printfn "Created temp file: %s" filename
    
    interface System.IDisposable with
        member this.Dispose() =
            if System.IO.File.Exists(filename) then
                System.IO.File.Delete(filename)
                printfn "Deleted temp file: %s" filename
    
    member this.Filename = filename
    member this.Write(content: string) =
        System.IO.File.WriteAllText(filename, content)
    member this.Read() =
        System.IO.File.ReadAllText(filename)

let useTempFile () =
    use tempFile = new TempFile()
    tempFile.Write("Hello, temporary world!")
    let content = tempFile.Read()
    printfn "Content: %s" content
    // TempFile.Dispose() called here

useTempFile()
printfn "Temp file has been cleaned up"
```

---

## 8. IEnumerable<T> Implementation

```fsharp
// Implement IEnumerable<T> สำหรับ custom collection
type NumberRange(start: int, finish: int, step: int) =
    interface System.Collections.Generic.IEnumerable<int> with
        member this.GetEnumerator() =
            let mutable current = start - step
            { new System.Collections.Generic.IEnumerator<int> with
                member this.Current = current
                member this.MoveNext() =
                    current <- current + step
                    current <= finish
                member this.Dispose() = ()
            interface System.Collections.IEnumerator with
                member this.Current = box current
                member this.MoveNext() = (this :> System.Collections.Generic.IEnumerator<int>).MoveNext()
                member this.Reset() = current <- start - step
            }
    
    interface System.Collections.IEnumerable with
        member this.GetEnumerator() =
            (this :> System.Collections.Generic.IEnumerable<int>).GetEnumerator() 
            :> System.Collections.IEnumerator

// InfiniteSequence
type FibonacciSequence() =
    interface System.Collections.Generic.IEnumerable<int64> with
        member this.GetEnumerator() =
            let mutable a = 0L
            let mutable b = 1L
            { new System.Collections.Generic.IEnumerator<int64> with
                member this.Current = a
                member this.MoveNext() =
                    let temp = a + b
                    a <- b
                    b <- temp
                    true
                member this.Dispose() = ()
            interface System.Collections.IEnumerator with
                member this.Current = box a
                member this.MoveNext() = (this :> System.Collections.Generic.IEnumerator<int64>).MoveNext()
                member this.Reset() =
                    a <- 0L
                    b <- 1L
            }
    
    interface System.Collections.IEnumerable with
        member this.GetEnumerator() =
            (this :> System.Collections.Generic.IEnumerable<int64>).GetEnumerator()
            :> System.Collections.IEnumerator

// ทดสอบ NumberRange
let range = NumberRange(1, 20, 3)
printfn "Range 1 to 20 step 3:"
for n in range do
    printf "%d " n
printfn ""

// ใช้ LINQ/Seq operations
let evens = range |> Seq.filter (fun n -> n % 2 = 0) |> Seq.toList
printfn "Evens in range: %A" evens

// ทดสอบ Fibonacci
let fib = FibonacciSequence()
let first10 = fib |> Seq.take 10 |> Seq.toList
printfn "First 10 Fibonacci: %A" first10
```

---

## 9. IComparable<T> and IEquatable<T>

```fsharp
// การ implement IComparable<T> และ IEquatable<T>
type Priority = Low | Medium | High | Critical

type Task(title: string, priority: Priority, dueDate: System.DateTime) =
    interface System.IComparable<Task> with
        member this.CompareTo(other) =
            // Sort by priority desc, then by due date asc
            let priorityOrder = function
                | Critical -> 0 | High -> 1 | Medium -> 2 | Low -> 3
            
            let cmp = compare (priorityOrder priority) (priorityOrder other.Priority)
            if cmp <> 0 then cmp
            else compare dueDate other.DueDate
    
    interface System.IComparable with
        member this.CompareTo(obj) =
            match obj with
            | :? Task as other -> (this :> System.IComparable<Task>).CompareTo(other)
            | _ -> failwith "Cannot compare"
    
    interface System.IEquatable<Task> with
        member this.Equals(other) =
            title = other.Title && priority = other.Priority && dueDate = other.DueDate
    
    override this.Equals(obj) =
        match obj with
        | :? Task as other -> (this :> System.IEquatable<Task>).Equals(other)
        | _ -> false
    
    override this.GetHashCode() =
        hash (title, priority, dueDate)
    
    member this.Title = title
    member this.Priority = priority
    member this.DueDate = dueDate
    
    override this.ToString() =
        sprintf "[%A] %s (due: %s)" priority title (dueDate.ToString("dd/MM/yyyy"))

let today = System.DateTime.Today
let tasks = [
    Task("Fix critical bug", Critical, today)
    Task("Review PR", High, today.AddDays(1.0))
    Task("Update docs", Low, today.AddDays(7.0))
    Task("Deploy to staging", High, today)
    Task("Team meeting", Medium, today.AddDays(2.0))
]

let sorted = tasks |> List.sortWith (fun a b -> (a :> System.IComparable<Task>).CompareTo(b))

printfn "Tasks by priority:"
sorted |> List.iter (fun t -> printfn "  %s" (t.ToString()))
```

---

## 10. Custom Interfaces for Domain Modeling

```fsharp
// Domain-specific interfaces
type IValidator<'T> =
    abstract member Validate: value: 'T -> Result<'T, string list>

type ISerializer =
    abstract member Serialize: obj -> string
    abstract member Deserialize<'T> : string -> 'T

type IEventHandler<'TEvent> =
    abstract member Handle: event: 'TEvent -> Async<unit>

// Email validator
type EmailValidator() =
    interface IValidator<string> with
        member this.Validate(email) =
            let errors = System.Collections.Generic.List<string>()
            
            if System.String.IsNullOrWhiteSpace(email) then
                errors.Add("Email cannot be empty")
            elif not (email.Contains("@")) then
                errors.Add("Email must contain @")
            elif not (email.Contains(".")) then
                errors.Add("Email must contain .")
            elif email.Length > 255 then
                errors.Add("Email too long")
            
            if errors.Count = 0 then Ok email
            else Error (errors |> Seq.toList)

// Password validator
type PasswordValidator(minLength: int) =
    interface IValidator<string> with
        member this.Validate(password) =
            let errors = System.Collections.Generic.List<string>()
            
            if password.Length < minLength then
                errors.Add(sprintf "Password must be at least %d characters" minLength)
            if not (password |> Seq.exists System.Char.IsUpper) then
                errors.Add("Password must contain uppercase letter")
            if not (password |> Seq.exists System.Char.IsDigit) then
                errors.Add("Password must contain digit")
            
            if errors.Count = 0 then Ok password
            else Error (errors |> Seq.toList)

// Composite validator
type CompositeValidator<'T>(validators: IValidator<'T> list) =
    interface IValidator<'T> with
        member this.Validate(value) =
            let errors = 
                validators
                |> List.collect (fun v ->
                    match v.Validate(value) with
                    | Ok _ -> []
                    | Error errs -> errs
                )
            if errors.IsEmpty then Ok value
            else Error errors

// ทดสอบ
let emailValidator: IValidator<string> = EmailValidator() :> IValidator<string>
let passwordValidator: IValidator<string> = PasswordValidator(8) :> IValidator<string>

let testEmail email =
    match emailValidator.Validate(email) with
    | Ok v -> printfn "Valid email: %s" v
    | Error errors -> printfn "Invalid email '%s': %A" email errors

testEmail "user@example.com"
testEmail "invalid-email"
testEmail ""

let testPassword pass =
    match passwordValidator.Validate(pass) with
    | Ok v -> printfn "Valid password"
    | Error errors -> printfn "Invalid password: %A" errors

testPassword "MyPass123"
testPassword "weak"
testPassword "nouppercase1"
```

---

## 11. Interface Segregation

```fsharp
// Bad: Fat interface ที่บีบให้ implement ทุกอย่าง
type IBadDocument =
    abstract member Read: unit -> string
    abstract member Write: content: string -> unit
    abstract member Print: unit -> unit
    abstract member Fax: number: string -> unit
    abstract member Scan: unit -> byte[]
    abstract member Delete: unit -> unit

// Good: Interface Segregation Principle
type IReadable =
    abstract member Read: unit -> string

type IWritable =
    abstract member Write: content: string -> unit

type IPrintable =
    abstract member Print: unit -> unit

type IDeletable =
    abstract member Delete: unit -> unit

// ReadOnly document
type ReadOnlyDocument(content: string) =
    interface IReadable with
        member this.Read() = content
    
    interface IPrintable with
        member this.Print() =
            printfn "=== Document ==="
            printfn "%s" content
            printfn "==============="

// Full document
type Document(filename: string) =
    let mutable content = ""
    
    do
        if System.IO.File.Exists(filename) then
            content <- System.IO.File.ReadAllText(filename)
    
    interface IReadable with
        member this.Read() = content
    
    interface IWritable with
        member this.Write(newContent) =
            content <- newContent
            System.IO.File.WriteAllText(filename, content)
    
    interface IPrintable with
        member this.Print() =
            printfn "File: %s" filename
            printfn "%s" content
    
    interface IDeletable with
        member this.Delete() =
            if System.IO.File.Exists(filename) then
                System.IO.File.Delete(filename)
                printfn "Deleted: %s" filename

// Function ที่รับเฉพาะ interface ที่ต้องการ
let printContent (doc: IReadable & IPrintable) =
    printfn "Content: %s" (doc.Read())
    doc.Print()

// เฉพาะบาง interface
let processReadable (doc: IReadable) =
    let content = doc.Read()
    printfn "Processing: %d chars" content.Length
    content

let readOnlyDoc = ReadOnlyDocument("This is a read-only document content")
processReadable (readOnlyDoc :> IReadable)
(readOnlyDoc :> IPrintable).Print()
```

---

## 12. Testing with Interfaces (Mocking)

```fsharp
// Interface สำหรับ testing
type IEmailService =
    abstract member SendEmail: to_: string -> subject: string -> body: string -> bool

type IDatabase =
    abstract member GetUser: id: int -> {| Id: int; Name: string; Email: string |} option
    abstract member SaveUser: user: {| Id: int; Name: string; Email: string |} -> bool

// Service ที่ใช้ dependency injection
type UserService(db: IDatabase, emailService: IEmailService) =
    member this.RegisterUser(name: string, email: string) =
        let newUser = {| Id = System.Random.Shared.Next(1000, 9999); Name = name; Email = email |}
        
        if db.SaveUser(newUser) then
            let sent = emailService.SendEmail email "Welcome!" (sprintf "Hello %s, welcome!" name)
            if sent then
                printfn "User registered and email sent: %s" email
                Ok newUser
            else
                printfn "User registered but email failed: %s" email
                Ok newUser
        else
            printfn "Failed to save user"
            Error "Database error"

// Mock implementations สำหรับ testing
type MockDatabase() =
    let mutable users: {| Id: int; Name: string; Email: string |} list = []
    
    interface IDatabase with
        member this.GetUser(id) =
            users |> List.tryFind (fun u -> u.Id = id)
        
        member this.SaveUser(user) =
            users <- user :: users
            printfn "[MockDB] Saved user: %s" user.Name
            true
    
    member this.Users = users

type MockEmailService() =
    let mutable sentEmails: (string * string * string) list = []
    
    interface IEmailService with
        member this.SendEmail(to_)(subject)(body) =
            sentEmails <- (to_, subject, body) :: sentEmails
            printfn "[MockEmail] Sent to: %s, Subject: %s" to_ subject
            true
    
    member this.SentEmails = sentEmails
    member this.SentCount = sentEmails.Length

// ทดสอบ
printfn "=== Testing with Mocks ==="
let mockDb = MockDatabase()
let mockEmail = MockEmailService()
let userService = UserService(mockDb :> IDatabase, mockEmail :> IEmailService)

let result = userService.RegisterUser("สมชาย", "somchai@example.com")
match result with
| Ok user -> printfn "Success: User %s created" user.Name
| Error msg -> printfn "Failed: %s" msg

printfn "Total users in DB: %d" mockDb.Users.Length
printfn "Total emails sent: %d" mockEmail.SentCount
```

---

## 13. Object Expressions for Interfaces

Object expressions ช่วยให้เราสามารถ implement interface ได้แบบ inline โดยไม่ต้องสร้าง class ใหม่

```fsharp
type IComparerStrategy<'T> =
    abstract member Compare: 'T -> 'T -> int

// สร้าง comparers ด้วย object expressions
let intAscending: IComparerStrategy<int> = 
    { new IComparerStrategy<int> with
        member this.Compare(a)(b) = compare a b }

let intDescending: IComparerStrategy<int> =
    { new IComparerStrategy<int> with
        member this.Compare(a)(b) = compare b a }

let stringByLength: IComparerStrategy<string> =
    { new IComparerStrategy<string> with
        member this.Compare(a)(b) = compare a.Length b.Length }

// Generic sort ที่ใช้ strategy
let sortWith (strategy: IComparerStrategy<'T>) (items: 'T list) =
    items |> List.sortWith (fun a b -> strategy.Compare a b)

let numbers = [5; 2; 8; 1; 9; 3; 7]
printfn "Ascending: %A" (sortWith intAscending numbers)   // [1;2;3;5;7;8;9]
printfn "Descending: %A" (sortWith intDescending numbers) // [9;8;7;5;3;2;1]

let words = ["banana"; "apple"; "cherry"; "kiwi"; "date"]
printfn "By length: %A" (sortWith stringByLength words)

// IDisposable ด้วย object expression
let createTempResource (name: string) =
    printfn "Creating resource: %s" name
    { new System.IDisposable with
        member this.Dispose() =
            printfn "Disposing resource: %s" name }

use resource1 = createTempResource "Resource A"
use resource2 = createTempResource "Resource B"
printfn "Using resources..."
// Resources disposed when leaving scope

// IEnumerable ด้วย object expression
let fromTo (start: int) (finish: int) =
    { new System.Collections.Generic.IEnumerable<int> with
        member this.GetEnumerator() =
            let mutable current = start - 1
            { new System.Collections.Generic.IEnumerator<int> with
                member this.Current = current
                member this.MoveNext() =
                    current <- current + 1
                    current <= finish
                member this.Dispose() = ()
            interface System.Collections.IEnumerator with
                member this.Current = box current
                member this.MoveNext() = (this :> System.Collections.Generic.IEnumerator<int>).MoveNext()
                member this.Reset() = current <- start - 1
            }
    interface System.Collections.IEnumerable with
        member this.GetEnumerator() =
            (this :> System.Collections.Generic.IEnumerable<int>).GetEnumerator()
            :> System.Collections.IEnumerator
    }

let range = fromTo 1 5
for n in range do printf "%d " n
printfn ""
```

---

## 14. Advanced Interface Patterns

```fsharp
// Builder pattern ด้วย interfaces
type IQueryBuilder<'T> =
    abstract member Where: (('T -> bool)) -> IQueryBuilder<'T>
    abstract member OrderBy: (('T -> obj)) -> IQueryBuilder<'T>
    abstract member Take: int -> IQueryBuilder<'T>
    abstract member Execute: unit -> 'T list

type Person = { Name: string; Age: int; City: string }

type PersonQueryBuilder(people: Person list) =
    let mutable filters: (Person -> bool) list = []
    let mutable sorter: (Person -> obj) option = None
    let mutable limit: int option = None
    
    interface IQueryBuilder<Person> with
        member this.Where(predicate) =
            filters <- predicate :: filters
            this :> IQueryBuilder<Person>
        
        member this.OrderBy(keySelector) =
            sorter <- Some keySelector
            this :> IQueryBuilder<Person>
        
        member this.Take(count) =
            limit <- Some count
            this :> IQueryBuilder<Person>
        
        member this.Execute() =
            let mutable result = people
            
            for filter in List.rev filters do
                result <- result |> List.filter filter
            
            match sorter with
            | Some s -> result <- result |> List.sortBy s
            | None -> ()
            
            match limit with
            | Some n -> result <- result |> List.take (min n result.Length)
            | None -> ()
            
            result

let people = [
    { Name = "สมชาย"; Age = 30; City = "Bangkok" }
    { Name = "สมหญิง"; Age = 25; City = "Chiang Mai" }
    { Name = "สมศักดิ์"; Age = 35; City = "Bangkok" }
    { Name = "สมปอง"; Age = 28; City = "Phuket" }
    { Name = "สมใจ"; Age = 22; City = "Bangkok" }
]

let query: IQueryBuilder<Person> = PersonQueryBuilder(people) :> IQueryBuilder<Person>
let results = 
    query
        .Where(fun p -> p.City = "Bangkok")
        .OrderBy(fun p -> box p.Age)
        .Execute()

printfn "Bangkok residents sorted by age:"
results |> List.iter (fun p -> printfn "  %s (%d)" p.Name p.Age)
```

---

## 15. Observable and Event Interfaces

```fsharp
// IObservable pattern
type IObservable<'T> =
    abstract member Subscribe: observer: IObserver<'T> -> System.IDisposable

type IObserver<'T> =
    abstract member OnNext: value: 'T -> unit
    abstract member OnError: error: System.Exception -> unit
    abstract member OnCompleted: unit -> unit

// Simple observable
type SimpleObservable<'T>(source: seq<'T>) =
    let observers = System.Collections.Generic.List<IObserver<'T>>()
    
    interface IObservable<'T> with
        member this.Subscribe(observer) =
            observers.Add(observer)
            { new System.IDisposable with
                member this.Dispose() =
                    observers.Remove(observer) |> ignore
            }
    
    member this.Publish() =
        try
            for item in source do
                for obs in observers do
                    obs.OnNext(item)
            for obs in observers do
                obs.OnCompleted()
        with ex ->
            for obs in observers do
                obs.OnError(ex)

// Console observer
type ConsoleObserver<'T>(name: string) =
    interface IObserver<'T> with
        member this.OnNext(value) = printfn "[%s] Next: %A" name value
        member this.OnError(ex) = printfn "[%s] Error: %s" name ex.Message
        member this.OnCompleted() = printfn "[%s] Completed" name

let observable: IObservable<int> = SimpleObservable([1..5]) :> IObservable<int>
let observer1: IObserver<int> = ConsoleObserver<int>("Observer1") :> IObserver<int>
let observer2: IObserver<int> = ConsoleObserver<int>("Observer2") :> IObserver<int>

use sub1 = observable.Subscribe(observer1)
use sub2 = observable.Subscribe(observer2)

(observable :?> SimpleObservable<int>).Publish()
```

---

## สรุป (Summary)

```fsharp
printfn "=== Interface Summary ==="
printfn ""
printfn "1. Interface definition:"
printfn "   type IMyInterface ="
printfn "       abstract member Method: param -> returnType"
printfn ""
printfn "2. Implementation:"
printfn "   type MyClass() ="
printfn "       interface IMyInterface with"
printfn "           member this.Method(param) = ..."
printfn ""
printfn "3. Multiple interfaces:"
printfn "   type MyClass() ="
printfn "       interface IFirst with ..."
printfn "       interface ISecond with ..."
printfn ""
printfn "4. Explicit casting needed:"
printfn "   let iface = myObj :> IMyInterface"
printfn "   iface.Method(param)"
printfn ""
printfn "5. Object expressions:"
printfn "   let impl = { new IMyInterface with"
printfn "                   member this.Method(p) = ... }"
printfn ""
printfn "6. IDisposable + use keyword:"
printfn "   use resource = new MyDisposable()"
printfn "   // Dispose called automatically"
```

---

## บทสรุป

Interface ใน F# เป็นเครื่องมือสำคัญสำหรับ:

1. **Abstraction** - ซ่อน implementation details
2. **Polymorphism** - ใช้ objects ต่างชนิดในแบบเดียวกัน
3. **Testability** - inject mock objects ในการ test
4. **Decoupling** - แยก code ให้ independent จากกัน
5. **.NET integration** - ใช้งาน .NET interfaces เช่น IDisposable, IEnumerable

Object expressions ใน F# ทำให้การ implement interface สะดวกยิ่งขึ้นโดยไม่ต้องสร้าง class ใหม่ทุกครั้ง
