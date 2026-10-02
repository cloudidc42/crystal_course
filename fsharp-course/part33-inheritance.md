# Part 33 - การสืบทอด (Inheritance)

## บทนำ (Introduction)

การสืบทอด (Inheritance) เป็นกลไกใน OOP ที่ให้คลาสหนึ่งสืบทอดคุณสมบัติและพฤติกรรมจากคลาสแม่ ใน F# การสืบทอดทำด้วย `inherit` keyword แต่ F# ส่งเสริมให้ใช้ Composition มากกว่า Inheritance

---

## 1. Base Class Definition (การนิยาม Base Class)

```fsharp
// Base class ที่มี virtual members
type Animal(name: string) =
    // Protected-like field (ใช้ let binding)
    let _name = name
    
    member this.Name = _name
    
    // Virtual method - subclass สามารถ override ได้
    abstract member Speak: unit -> string
    default this.Speak() = "..."
    
    abstract member Move: unit -> string
    default this.Move() = sprintf "%s moves" _name
    
    member this.Introduce() =
        sprintf "I am %s. %s" _name (this.Speak())
    
    override this.ToString() =
        sprintf "Animal(%s)" _name

// Concrete class ที่สืบทอด Animal
type Dog(name: string) =
    inherit Animal(name)
    
    // Override virtual method
    override this.Speak() = "Woof!"
    override this.Move() = sprintf "%s runs on four legs" this.Name
    
    // New member เฉพาะ Dog
    member this.Fetch() =
        sprintf "%s fetches the ball!" this.Name

type Cat(name: string) =
    inherit Animal(name)
    
    override this.Speak() = "Meow!"
    override this.Move() = sprintf "%s slinks gracefully" this.Name
    
    member this.Purr() = sprintf "%s purrs... *purrr*" this.Name

type Bird(name: string, canFly: bool) =
    inherit Animal(name)
    
    override this.Speak() = "Tweet!"
    override this.Move() =
        if canFly then sprintf "%s soars through the sky" this.Name
        else sprintf "%s waddles" this.Name
    
    member this.CanFly = canFly

// ทดสอบ
let dog = Dog("Rex")
let cat = Cat("Whiskers")
let eagle = Bird("Eagle", true)
let penguin = Bird("Penguin", false)

let animals: Animal list = [dog; cat; eagle; penguin]

for animal in animals do
    printfn "%s" (animal.Introduce())
    printfn "  Move: %s" (animal.Move())

// Type-specific operations
printfn "\n%s" (dog.Fetch())
printfn "%s" (cat.Purr())
printfn "Eagle can fly: %b" eagle.CanFly
```

---

## 2. inherit Keyword

```fsharp
// inherit ใช้เรียก base class constructor
type Vehicle(make: string, model: string, year: int) =
    member this.Make = make
    member this.Model = model
    member this.Year = year
    
    abstract member Fuel: string
    default this.Fuel = "Unknown"
    
    abstract member MaxSpeed: float
    default this.MaxSpeed = 0.0
    
    abstract member Describe: unit -> string
    default this.Describe() =
        sprintf "%d %s %s (%s, max %.0f km/h)" year make model this.Fuel this.MaxSpeed

// inherit เรียก base constructor ด้วย Vehicle(make, model, year)
type ElectricCar(make: string, model: string, year: int, range: float) =
    inherit Vehicle(make, model, year)
    
    override this.Fuel = "Electric"
    override this.MaxSpeed = 200.0
    
    member this.Range = range
    
    override this.Describe() =
        sprintf "%s, Range: %.0f km" (base.Describe()) range

type GasCar(make: string, model: string, year: int, engineCC: int) =
    inherit Vehicle(make, model, year)
    
    override this.Fuel = "Gasoline"
    override this.MaxSpeed = 180.0
    
    member this.EngineCC = engineCC
    
    override this.Describe() =
        sprintf "%s, Engine: %d cc" (base.Describe()) engineCC

type Truck(make: string, model: string, year: int, payload: float) =
    inherit Vehicle(make, model, year)
    
    override this.Fuel = "Diesel"
    override this.MaxSpeed = 120.0
    
    member this.Payload = payload
    
    override this.Describe() =
        sprintf "%s, Payload: %.1f tons" (base.Describe()) payload

let tesla = ElectricCar("Tesla", "Model 3", 2023, 500.0)
let toyota = GasCar("Toyota", "Camry", 2022, 2500)
let volvo = Truck("Volvo", "FH16", 2021, 25.0)

let vehicles: Vehicle list = [tesla; toyota; volvo]
for v in vehicles do
    printfn "%s" (v.Describe())
```

---

## 3. Overriding Virtual Methods

```fsharp
// Multi-level inheritance
type Shape() =
    abstract member Area: float
    default this.Area = 0.0
    
    abstract member Perimeter: float
    default this.Perimeter = 0.0
    
    abstract member Draw: unit -> string
    default this.Draw() = "Drawing shape"
    
    member this.Describe() =
        sprintf "Area=%.2f, Perimeter=%.2f" this.Area this.Perimeter

type Polygon(sides: float list) =
    inherit Shape()
    
    let _sides = sides
    
    override this.Perimeter =
        List.sum _sides
    
    override this.Area = 0.0  // Override in subclasses
    
    override this.Draw() =
        sprintf "Polygon with %d sides" _sides.Length
    
    member this.Sides = _sides

type RegularPolygon(n: int, sideLength: float) =
    inherit Polygon(List.replicate n sideLength)
    
    let _n = n
    let _sideLength = sideLength
    
    override this.Area =
        let pi = System.Math.PI
        (float _n * _sideLength * _sideLength) / (4.0 * System.Math.Tan(pi / float _n))
    
    override this.Draw() =
        sprintf "Regular %d-gon, side=%.2f" _n _sideLength

// ทดสอบ
let pentagon = RegularPolygon(5, 4.0)
let hexagon = RegularPolygon(6, 3.0)

printfn "Pentagon: %s, %s" (pentagon.Draw()) (pentagon.Describe())
printfn "Hexagon: %s, %s" (hexagon.Draw()) (hexagon.Describe())
```

---

## 4. Abstract Members

```fsharp
// Abstract class - ไม่สามารถสร้าง instance ได้โดยตรง
[<AbstractClass>]
type DatabaseAdapter() =
    // Abstract members ต้อง implement ใน subclass
    abstract member Connect: connectionString: string -> unit
    abstract member Disconnect: unit -> unit
    abstract member ExecuteQuery: sql: string -> string[][]
    abstract member ExecuteNonQuery: sql: string -> int
    
    // Non-abstract members ที่ subclass ใช้ได้
    member val IsConnected = false with get, set
    
    // Template method pattern
    member this.WithConnection(connectionString: string)(action: unit -> 'T) =
        this.Connect(connectionString)
        try
            let result = action()
            result
        finally
            this.Disconnect()

[<AbstractClass>]
type RelationalDatabase() =
    inherit DatabaseAdapter()
    
    // เพิ่ม abstract members เฉพาะ relational databases
    abstract member BeginTransaction: unit -> unit
    abstract member CommitTransaction: unit -> unit
    abstract member RollbackTransaction: unit -> unit
    
    member this.WithTransaction(action: unit -> 'T) =
        this.BeginTransaction()
        try
            let result = action()
            this.CommitTransaction()
            result
        with ex ->
            this.RollbackTransaction()
            reraise()

// Concrete implementation
type SqliteAdapter() =
    inherit RelationalDatabase()
    
    let mutable connection: string option = None
    let mutable inTransaction = false
    
    override this.Connect(cs) =
        connection <- Some cs
        this.IsConnected <- true
        printfn "[SQLite] Connected to: %s" cs
    
    override this.Disconnect() =
        connection <- None
        this.IsConnected <- false
        printfn "[SQLite] Disconnected"
    
    override this.ExecuteQuery(sql) =
        printfn "[SQLite] Query: %s" sql
        [| [| "row1_col1"; "row1_col2" |]; [| "row2_col1"; "row2_col2" |] |]
    
    override this.ExecuteNonQuery(sql) =
        printfn "[SQLite] NonQuery: %s" sql
        1  // rows affected
    
    override this.BeginTransaction() =
        inTransaction <- true
        printfn "[SQLite] BEGIN TRANSACTION"
    
    override this.CommitTransaction() =
        inTransaction <- false
        printfn "[SQLite] COMMIT"
    
    override this.RollbackTransaction() =
        inTransaction <- false
        printfn "[SQLite] ROLLBACK"

let db = SqliteAdapter()
let result = db.WithConnection("test.db") (fun () ->
    let rows = db.ExecuteQuery("SELECT * FROM users")
    rows
)

printfn "Rows returned: %d" result.Length
```

---

## 5. sealed Keyword

```fsharp
// sealed class ไม่สามารถสืบทอดได้
[<Sealed>]
type ImmutableVector3(x: float, y: float, z: float) =
    member this.X = x
    member this.Y = y
    member this.Z = z
    
    member this.Length =
        sqrt (x * x + y * y + z * z)
    
    member this.Normalize() =
        let len = this.Length
        if len = 0.0 then ImmutableVector3(0.0, 0.0, 0.0)
        else ImmutableVector3(x / len, y / len, z / len)
    
    member this.Add(other: ImmutableVector3) =
        ImmutableVector3(x + other.X, y + other.Y, z + other.Z)
    
    member this.Scale(factor: float) =
        ImmutableVector3(x * factor, y * factor, z * factor)
    
    member this.Dot(other: ImmutableVector3) =
        x * other.X + y * other.Y + z * other.Z
    
    static member Zero = ImmutableVector3(0.0, 0.0, 0.0)
    static member UnitX = ImmutableVector3(1.0, 0.0, 0.0)
    static member UnitY = ImmutableVector3(0.0, 1.0, 0.0)
    static member UnitZ = ImmutableVector3(0.0, 0.0, 1.0)
    
    override this.ToString() =
        sprintf "(%.3f, %.3f, %.3f)" x y z

let v1 = ImmutableVector3(1.0, 2.0, 3.0)
let v2 = ImmutableVector3(4.0, 5.0, 6.0)
printfn "v1 = %s" (v1.ToString())
printfn "v2 = %s" (v2.ToString())
printfn "v1 + v2 = %s" ((v1.Add(v2)).ToString())
printfn "v1.Length = %.3f" v1.Length
printfn "v1.Dot(v2) = %.1f" (v1.Dot(v2))

// ต่อไปนี้จะ compile error:
// type ExtendedVector(x, y, z) =
//     inherit ImmutableVector3(x, y, z)  // Error! sealed class
```

---

## 6. Calling Base Class Methods

```fsharp
type Logger() =
    abstract member Log: string -> unit
    default this.Log(message) =
        printfn "[LOG] %s" message
    
    abstract member LogError: string -> unit
    default this.LogError(message) =
        this.Log(sprintf "ERROR: %s" message)

type TimestampLogger() =
    inherit Logger()
    
    override this.Log(message) =
        let ts = System.DateTime.Now.ToString("HH:mm:ss.fff")
        base.Log(sprintf "[%s] %s" ts message)  // เรียก base method

type PrefixLogger(prefix: string) =
    inherit Logger()
    
    override this.Log(message) =
        base.Log(sprintf "[%s] %s" prefix message)
    
    override this.LogError(message) =
        // เรียก base.LogError ซึ่งเรียก this.Log (polymorphic)
        base.LogError(message)

// ทดสอบ
let logger = TimestampLogger()
logger.Log "Application started"
logger.LogError "Connection failed"

let prefixLogger = PrefixLogger("MYAPP")
prefixLogger.Log "Processing request"
prefixLogger.LogError "Something went wrong"
```

---

## 7. Downcasting with :? and :?>

```fsharp
type Employee(name: string) =
    member this.Name = name
    
    abstract member Role: string
    default this.Role = "Employee"
    
    abstract member GetDetails: unit -> string
    default this.GetDetails() = sprintf "%s (%s)" name this.Role

type Manager(name: string, department: string) =
    inherit Employee(name)
    
    override this.Role = "Manager"
    
    override this.GetDetails() =
        sprintf "%s, Dept: %s" (base.GetDetails()) department
    
    member this.Department = department
    member this.Approve(request: string) =
        printfn "Manager %s approved: %s" name request

type Developer(name: string, language: string) =
    inherit Employee(name)
    
    override this.Role = "Developer"
    
    override this.GetDetails() =
        sprintf "%s, Lang: %s" (base.GetDetails()) language
    
    member this.Language = language
    member this.WriteCode() =
        printfn "Developer %s writes %s code" name language

// ทดสอบ downcasting
let employees: Employee list = [
    Manager("สมชาย", "IT")
    Developer("สมหญิง", "F#")
    Manager("สมศักดิ์", "Finance")
    Developer("สมปอง", "Python")
]

printfn "=== All Employees ==="
for emp in employees do
    printfn "%s" (emp.GetDetails())

printfn "\n=== Managers ==="
for emp in employees do
    match emp with
    | :? Manager as mgr ->
        printfn "%s manages %s dept" mgr.Name mgr.Department
    | _ -> ()

printfn "\n=== Developers ==="
for emp in employees do
    match emp with
    | :? Developer as dev ->
        printfn "%s uses %s" dev.Name dev.Language
    | _ -> ()

// :?> operator - unsafe downcast
let firstEmployee = employees.[0]
try
    let mgr = firstEmployee :?> Manager
    printfn "\nFirst is manager: %s" mgr.Department
with
| :? System.InvalidCastException ->
    printfn "Not a manager!"

// :? สำหรับ type check
let isManager (emp: Employee) = emp :? Manager
let isDeveloper (emp: Employee) = emp :? Developer

let managers = employees |> List.filter isManager
printfn "\nManager count: %d" managers.Length
```

---

## 8. When to Use Inheritance in F#

```fsharp
// F# ส่งเสริม Composition มากกว่า Inheritance
// แต่ Inheritance มีประโยชน์ใน:
// 1. .NET interop (extend .NET classes)
// 2. Exception hierarchies
// 3. Abstract base class patterns

// Custom exception hierarchy (ดูตัวอย่างดี)
type AppException(message: string, ?innerEx: exn) =
    inherit System.Exception(message, defaultArg innerEx null)
    member this.Code = "APP_ERROR"

type DatabaseException(message: string, query: string) =
    inherit AppException(message)
    member this.Query = query
    override this.Message = sprintf "[DB] %s (Query: %s)" message query

type NetworkException(message: string, url: string, statusCode: int) =
    inherit AppException(message)
    member this.Url = url
    member this.StatusCode = statusCode
    override this.Message = sprintf "[NET] %s (%d: %s)" message statusCode url

type ValidationException(message: string, field: string) =
    inherit AppException(message)
    member this.Field = field
    override this.Message = sprintf "[VAL] %s (field: %s)" message field

// ใช้ exception hierarchy
let processRequest (input: string) =
    if System.String.IsNullOrEmpty(input) then
        raise (ValidationException("Input cannot be empty", "input"))
    
    try
        // Simulate database query
        if input = "bad_query" then
            raise (DatabaseException("Query failed", sprintf "SELECT * FROM %s" input))
        
        // Simulate network call
        if input = "timeout" then
            raise (NetworkException("Request timed out", "http://api.example.com", 408))
        
        sprintf "Processed: %s" input
    with
    | :? ValidationException as ex ->
        printfn "Validation error on field '%s': %s" ex.Field ex.Message
        "validation_error"
    | :? DatabaseException as ex ->
        printfn "Database error: %s" ex.Message
        "db_error"
    | :? NetworkException as ex ->
        printfn "Network error %d: %s" ex.StatusCode ex.Message
        "network_error"
    | :? AppException as ex ->
        printfn "App error: %s" ex.Message
        "app_error"

let results = ["valid_input"; ""; "bad_query"; "timeout"] |> List.map processRequest
printfn "\nResults: %A" results
```

---

## 9. Composition over Inheritance

```fsharp
// Composition pattern - prefer in F#

// Interface-based components
type ILogger =
    abstract member Log: string -> unit

type IValidator<'T> =
    abstract member IsValid: 'T -> bool
    abstract member GetErrors: 'T -> string list

// Concrete implementations
type ConsoleLogger() =
    interface ILogger with
        member this.Log(msg) = printfn "[LOG] %s" msg

type EmailValidator() =
    interface IValidator<string> with
        member this.IsValid(email) = email.Contains("@") && email.Contains(".")
        member this.GetErrors(email) =
            [ if not (email.Contains("@")) then yield "Missing @"
              if not (email.Contains(".")) then yield "Missing ." ]

// Service ที่ใช้ composition แทน inheritance
type UserRegistrationService(logger: ILogger, emailValidator: IValidator<string>) =
    member this.Register(username: string, email: string) =
        logger.Log(sprintf "Registering user: %s" username)
        
        let emailErrors = emailValidator.GetErrors(email)
        if not emailErrors.IsEmpty then
            logger.Log(sprintf "Validation failed: %A" emailErrors)
            Error emailErrors
        else
            logger.Log(sprintf "User registered: %s (%s)" username email)
            Ok {| Username = username; Email = email |}

// Comparison: Inheritance approach (not preferred)
type BaseService() =
    member this.Log(msg) = printfn "[BASE] %s" msg
    abstract member Execute: string -> string
    default this.Execute(input) = input

type DerivedService() =
    inherit BaseService()
    override this.Execute(input) =
        base.Log(sprintf "Executing: %s" input)
        sprintf "Result: %s" input

// Composition approach (preferred in F#)
let createService (logger: ILogger) =
    fun (input: string) ->
        logger.Log(sprintf "Processing: %s" input)
        sprintf "Result: %s" input

// ทดสอบ
let logger = ConsoleLogger() :> ILogger
let emailValidator = EmailValidator() :> IValidator<string>
let service = UserRegistrationService(logger, emailValidator)

match service.Register("john", "john@example.com") with
| Ok user -> printfn "Success: %s" user.Username
| Error errors -> printfn "Errors: %A" errors

match service.Register("jane", "invalid-email") with
| Ok user -> printfn "Success: %s" user.Username
| Error errors -> printfn "Errors: %A" errors
```

---

## 10. Abstract Classes

```fsharp
// Abstract class pattern
[<AbstractClass>]
type Serializer<'T>() =
    // Abstract methods
    abstract member Serialize: 'T -> string
    abstract member Deserialize: string -> 'T
    
    // Template methods using abstract members
    member this.SerializeToFile(value: 'T, filename: string) =
        let content = this.Serialize(value)
        System.IO.File.WriteAllText(filename, content)
        printfn "Serialized to %s (%d bytes)" filename content.Length
    
    member this.DeserializeFromFile(filename: string) =
        let content = System.IO.File.ReadAllText(filename)
        this.Deserialize(content)

// JSON-like serializer
type JsonSerializer<'T>() =
    inherit Serializer<'T>()
    
    override this.Serialize(value) =
        // Simplified serialization
        sprintf """{"value": %A}""" value
    
    override this.Deserialize(json) =
        // Simplified deserialization
        failwith "Not implemented in this example"

// CSV serializer for records
[<AbstractClass>]
type CsvSerializer<'T>() =
    inherit Serializer<'T>()
    
    abstract member GetHeaders: unit -> string list
    abstract member ToRow: 'T -> string list
    abstract member FromRow: string list -> 'T
    
    override this.Serialize(value) =
        let headers = this.GetHeaders() |> String.concat ","
        let row = this.ToRow(value) |> String.concat ","
        sprintf "%s\n%s" headers row
    
    override this.Deserialize(csv) =
        let lines = csv.Split('\n')
        let values = lines.[1].Split(',') |> Array.toList
        this.FromRow(values)

type PersonCsvSerializer() =
    inherit CsvSerializer<{| Name: string; Age: int |}>()
    
    override this.GetHeaders() = ["Name"; "Age"]
    
    override this.ToRow(person) =
        [person.Name; string person.Age]
    
    override this.FromRow(row) =
        {| Name = row.[0]; Age = int row.[1] |}

// ทดสอบ
let serializer = PersonCsvSerializer()
let person = {| Name = "สมชาย"; Age = 30 |}
let csv = serializer.Serialize(person)
printfn "CSV:\n%s" csv

let deserialized = serializer.Deserialize(csv)
printfn "Deserialized: %s (%d)" deserialized.Name deserialized.Age
```

---

## 11. Type Hierarchy in F#

```fsharp
// F# type hierarchy example
// obj (System.Object) -> Animal -> Mammal -> Dog

type Animal(name: string) =
    abstract member Sound: string
    default this.Sound = "..."
    
    member this.Name = name
    
    override this.ToString() = sprintf "Animal(%s)" name

type Mammal(name: string) =
    inherit Animal(name)
    
    abstract member WarmBlooded: bool
    default this.WarmBlooded = true
    
    member this.GiveBirth() = sprintf "%s gives live birth" this.Name

type Dog(name: string, breed: string) =
    inherit Mammal(name)
    
    override this.Sound = "Woof"
    
    member this.Breed = breed
    member this.Fetch() = sprintf "%s fetches!" this.Name
    
    override this.ToString() = sprintf "Dog(%s, %s)" name breed

type Labrador(name: string) =
    inherit Dog(name, "Labrador")
    
    member this.GuideUser() = sprintf "%s guides the user" name
    
    override this.ToString() = sprintf "Labrador(%s)" name

// Type hierarchy operations
let lab = Labrador("Buddy")

// ตรวจสอบ type hierarchy
printfn "Is Labrador: %b" (lab :? Labrador)
printfn "Is Dog: %b" (lab :? Dog)
printfn "Is Mammal: %b" (lab :? Mammal)
printfn "Is Animal: %b" (lab :? Animal)
printfn "Is obj: %b" (box lab :? obj)

// Polymorphism
let animals: Animal list = [
    lab :> Animal
    Dog("Rex", "German Shepherd") :> Animal
    Mammal("Generic Mammal") :> Animal
]

for a in animals do
    printfn "%s says: %s" a.Name a.Sound
    
    match a with
    | :? Labrador as l -> printfn "  %s" (l.GuideUser())
    | :? Dog as d -> printfn "  %s" (d.Fetch())
    | :? Mammal as m -> printfn "  %s" (m.GiveBirth())
    | _ -> ()
```

---

## 12. Practical: UI Component Hierarchy

```fsharp
// UI Component hierarchy ตัวอย่าง
[<AbstractClass>]
type UIComponent(id: string) =
    let mutable _visible = true
    let mutable _enabled = true
    
    member this.Id = id
    
    member this.IsVisible
        with get() = _visible
        and set(v) = _visible <- v
    
    member this.IsEnabled
        with get() = _enabled
        and set(v) = _enabled <- v
    
    abstract member Render: unit -> string
    abstract member HandleEvent: string -> bool
    
    member this.RenderIfVisible() =
        if _visible then this.Render()
        else sprintf "<!-- %s is hidden -->" id

[<AbstractClass>]
type Container(id: string) =
    inherit UIComponent(id)
    
    let mutable children: UIComponent list = []
    
    member this.AddChild(child: UIComponent) =
        children <- child :: children
    
    member this.Children = List.rev children
    
    abstract member Layout: string
    
    override this.HandleEvent(event) =
        children |> List.exists (fun c -> c.HandleEvent(event))
    
    override this.Render() =
        let childrenHtml = 
            children 
            |> List.rev
            |> List.map (fun c -> "  " + c.Render())
            |> String.concat "\n"
        sprintf """<%s id="%s" layout="%s">
%s
</%s>""" (this.Layout) id this.Layout childrenHtml (this.Layout)

type Button(id: string, label: string, onClick: unit -> unit) =
    inherit UIComponent(id)
    
    member val Label = label with get, set
    
    override this.Render() =
        sprintf """<button id="%s" enabled="%b">%s</button>""" id this.IsEnabled label
    
    override this.HandleEvent(event) =
        if event = sprintf "click:%s" id && this.IsEnabled then
            onClick()
            true
        else false

type TextInput(id: string, placeholder: string) =
    inherit UIComponent(id)
    
    member val Value = "" with get, set
    member val Placeholder = placeholder with get, set
    
    override this.Render() =
        sprintf """<input id="%s" type="text" placeholder="%s" value="%s"/>""" 
            id this.Placeholder this.Value
    
    override this.HandleEvent(event) =
        if event.StartsWith(sprintf "input:%s:" id) then
            this.Value <- event.Substring(sprintf "input:%s:" id |> String.length)
            true
        else false

type Panel(id: string) =
    inherit Container(id)
    override this.Layout = "div"

// ทดสอบ
let panel = Panel("main-panel")
let nameInput = TextInput("name-input", "Enter your name")
let emailInput = TextInput("email-input", "Enter your email")
let submitBtn = Button("submit-btn", "Submit", fun () -> 
    printfn "Form submitted!")
let cancelBtn = Button("cancel-btn", "Cancel", fun () ->
    printfn "Cancelled!")

panel.AddChild(nameInput)
panel.AddChild(emailInput)
panel.AddChild(submitBtn)
panel.AddChild(cancelBtn)

printfn "Rendered UI:"
printfn "%s" (panel.Render())

printfn "\nSimulating events:"
nameInput.HandleEvent("input:name-input:John Doe") |> ignore
emailInput.HandleEvent("input:email-input:john@example.com") |> ignore
panel.HandleEvent("click:submit-btn") |> ignore
```

---

## สรุป (Summary)

```fsharp
printfn "=== Inheritance in F# Summary ==="
printfn ""
printfn "1. inherit keyword:"
printfn "   type Child(args) ="
printfn "       inherit Parent(parentArgs)"
printfn ""
printfn "2. Override virtual members:"
printfn "   abstract member Method: returnType"
printfn "   default this.Method = defaultImpl"
printfn "   // In subclass:"
printfn "   override this.Method = newImpl"
printfn ""
printfn "3. Call base members:"
printfn "   base.Method(args)"
printfn ""
printfn "4. Type checking:"
printfn "   match obj with"
printfn "   | :? SubClass as s -> ..."  
printfn ""
printfn "5. Unsafe downcast:"
printfn "   let s = obj :?> SubClass"
printfn ""
printfn "6. Abstract class:"
printfn "   [<AbstractClass>]"
printfn "   type Abstract() = ..."
printfn ""
printfn "7. Sealed class:"
printfn "   [<Sealed>]"
printfn "   type Final() = ..."
printfn ""
printfn "Best practice: Prefer composition over inheritance in F#"
```

---

## บทสรุป

Inheritance ใน F# มีประโยชน์ในสถานการณ์เหล่านี้:
1. **Exception hierarchies** - สร้างลำดับชั้น exception
2. **Abstract base classes** - กำหนด template สำหรับ subclasses
3. **.NET framework classes** - extend .NET classes
4. **UI frameworks** - component hierarchies

แต่โดยทั่วไป F# ส่งเสริม:
- **Composition** มากกว่า Inheritance
- **Discriminated Unions** สำหรับ data hierarchies
- **Interfaces** สำหรับ polymorphism
- **Function composition** สำหรับ behavior reuse
