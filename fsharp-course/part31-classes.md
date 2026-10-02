# Part 31 - คลาสใน F# (Classes in F#)

## บทนำ (Introduction)

คลาสใน F# เป็นส่วนหนึ่งของการเขียนโปรแกรมเชิงวัตถุ (Object-Oriented Programming) ที่ F# รองรับเต็มรูปแบบ แม้ว่า F# จะเน้น Functional Programming เป็นหลัก แต่ก็มีความสามารถในการสร้างคลาสที่สมบูรณ์เต็มรูปแบบ

Classes in F# are part of object-oriented programming that F# fully supports. Although F# emphasizes functional programming, it also has complete capabilities for creating classes.

---

## 1. Class Definition Syntax (ไวยากรณ์การนิยามคลาส)

### Basic Class (คลาสพื้นฐาน)

```fsharp
// การนิยามคลาสพื้นฐาน
type Person(name: string, age: int) =
    // Primary constructor parameters: name, age
    member this.Name = name
    member this.Age = age
    
    override this.ToString() =
        sprintf "Person(%s, %d)" name age

// การสร้าง instance
let person = Person("สมชาย", 30)
printfn "%s" (person.ToString())   // Person(สมชาย, 30)
printfn "ชื่อ: %s" person.Name     // ชื่อ: สมชาย
printfn "อายุ: %d" person.Age      // อายุ: 30
```

### Class with Mutable State (คลาสที่มีสถานะที่เปลี่ยนแปลงได้)

```fsharp
type Counter(initialValue: int) =
    let mutable count = initialValue
    
    member this.Count = count
    
    member this.Increment() =
        count <- count + 1
    
    member this.Decrement() =
        count <- count - 1
    
    member this.Reset() =
        count <- initialValue
    
    override this.ToString() =
        sprintf "Counter(count=%d)" count

let counter = Counter(0)
counter.Increment()
counter.Increment()
counter.Increment()
printfn "%s" (counter.ToString())  // Counter(count=3)
counter.Decrement()
printfn "Count: %d" counter.Count  // Count: 2
counter.Reset()
printfn "After reset: %d" counter.Count  // After reset: 0
```

---

## 2. Primary Constructor (Constructor หลัก)

Primary constructor ใน F# คือพารามิเตอร์ที่อยู่หลังชื่อคลาสในการนิยาม

```fsharp
// Primary constructor มีพารามิเตอร์ x และ y
type Point(x: float, y: float) =
    member this.X = x
    member this.Y = y
    
    member this.Distance(other: Point) =
        let dx = x - other.X
        let dy = y - other.Y
        sqrt (dx * dx + dy * dy)
    
    member this.Translate(dx: float, dy: float) =
        Point(x + dx, y + dy)
    
    override this.ToString() =
        sprintf "(%.2f, %.2f)" x y

let origin = Point(0.0, 0.0)
let p1 = Point(3.0, 4.0)
let p2 = Point(6.0, 8.0)

printfn "p1 = %s" (p1.ToString())          // p1 = (3.00, 4.00)
printfn "p2 = %s" (p2.ToString())          // p2 = (6.00, 8.00)
printfn "Distance p1->p2: %.2f" (p1.Distance(p2))   // Distance p1->p2: 5.00
printfn "Distance origin->p1: %.2f" (origin.Distance(p1))  // 5.00

let p3 = p1.Translate(2.0, 1.0)
printfn "p3 = %s" (p3.ToString())          // p3 = (5.00, 5.00)
```

### Primary Constructor with Validation (Primary Constructor พร้อมการตรวจสอบ)

```fsharp
type PositiveNumber(value: float) =
    do
        if value <= 0.0 then
            invalidArg "value" (sprintf "ค่าต้องเป็นบวก แต่ได้รับ %g" value)
    
    member this.Value = value
    
    member this.Add(other: PositiveNumber) =
        PositiveNumber(value + other.Value)
    
    member this.Multiply(other: PositiveNumber) =
        PositiveNumber(value * other.Value)
    
    override this.ToString() =
        sprintf "PositiveNumber(%g)" value

let num1 = PositiveNumber(5.0)
let num2 = PositiveNumber(3.0)
let sum = num1.Add(num2)
printfn "Sum: %s" (sum.ToString())  // Sum: PositiveNumber(8)

try
    let invalid = PositiveNumber(-1.0)
    ()
with
| :? System.ArgumentException as ex ->
    printfn "Error: %s" ex.Message
```

---

## 3. Additional Constructors (Constructor เพิ่มเติม)

```fsharp
type Rectangle(width: float, height: float) =
    // Primary constructor
    
    // Additional constructor - สร้างสี่เหลี่ยมจัตุรัส
    new(side: float) = Rectangle(side, side)
    
    // Additional constructor - จาก string "WxH"
    new(dimensions: string) =
        let parts = dimensions.Split('x')
        Rectangle(float parts.[0], float parts.[1])
    
    member this.Width = width
    member this.Height = height
    member this.Area = width * height
    member this.Perimeter = 2.0 * (width + height)
    member this.IsSquare = width = height
    
    override this.ToString() =
        sprintf "Rectangle(%.2f x %.2f)" width height

let rect1 = Rectangle(4.0, 3.0)
let square = Rectangle(5.0)
let rect2 = Rectangle("6x4")

printfn "%s - Area: %.2f" (rect1.ToString()) rect1.Area   // Rectangle(4.00 x 3.00) - Area: 12.00
printfn "%s - IsSquare: %b" (square.ToString()) square.IsSquare  // Rectangle(5.00 x 5.00) - IsSquare: true
printfn "%s - Perimeter: %.2f" (rect2.ToString()) rect2.Perimeter  // Rectangle(6.00 x 4.00) - Perimeter: 20.00
```

### Multiple Additional Constructors

```fsharp
type Color(r: int, g: int, b: int) =
    do
        let validate name v =
            if v < 0 || v > 255 then
                invalidArg name (sprintf "%s ต้องอยู่ระหว่าง 0-255 แต่ได้รับ %d" name v)
        validate "r" r
        validate "g" g
        validate "b" b
    
    // Constructor จาก hex string "#RRGGBB"
    new(hex: string) =
        let hex = hex.TrimStart('#')
        let r = System.Convert.ToInt32(hex.[0..1], 16)
        let g = System.Convert.ToInt32(hex.[2..3], 16)
        let b = System.Convert.ToInt32(hex.[4..5], 16)
        Color(r, g, b)
    
    // Constructor จาก grayscale
    new(gray: int) = Color(gray, gray, gray)
    
    member this.R = r
    member this.G = g
    member this.B = b
    
    member this.ToHex() =
        sprintf "#%02X%02X%02X" r g b
    
    member this.IsGrayscale = r = g && g = b
    
    member this.Lighten(amount: float) =
        let lighten c = min 255 (c + int (float c * amount))
        Color(lighten r, lighten g, lighten b)
    
    override this.ToString() =
        sprintf "Color(r=%d, g=%d, b=%d)" r g b

let red = Color(255, 0, 0)
let white = Color("#FFFFFF")
let gray = Color(128)
let lightRed = red.Lighten(0.5)

printfn "%s -> %s" (red.ToString()) (red.ToHex())     // Color(r=255, g=0, b=0) -> #FF0000
printfn "%s -> %s" (white.ToString()) (white.ToHex()) // Color(r=255, g=255, b=255) -> #FFFFFF
printfn "Is grayscale: %b" gray.IsGrayscale           // Is grayscale: true
printfn "Light red: %s" (lightRed.ToString())
```

---

## 4. Instance Members (สมาชิก Instance)

```fsharp
type Circle(radius: float) =
    do
        if radius <= 0.0 then
            invalidArg "radius" "รัศมีต้องมากกว่าศูนย์"
    
    let pi = System.Math.PI
    
    // Properties
    member this.Radius = radius
    member this.Diameter = radius * 2.0
    member this.Area = pi * radius * radius
    member this.Circumference = 2.0 * pi * radius
    
    // Methods
    member this.Scale(factor: float) =
        Circle(radius * factor)
    
    member this.Contains(x: float, y: float) =
        let distSq = x * x + y * y
        distSq <= radius * radius
    
    member this.Intersects(other: Circle, centerX: float, centerY: float) =
        let dist = sqrt (centerX * centerX + centerY * centerY)
        dist < radius + other.Radius
    
    override this.ToString() =
        sprintf "Circle(r=%.2f, area=%.2f)" radius this.Area

let c1 = Circle(5.0)
let c2 = c1.Scale(2.0)

printfn "%s" (c1.ToString())
printfn "%s" (c2.ToString())
printfn "Contains (3, 4): %b" (c1.Contains(3.0, 4.0))   // true (3^2 + 4^2 = 25 = 5^2)
printfn "Contains (4, 4): %b" (c1.Contains(4.0, 4.0))   // false (4^2 + 4^2 = 32 > 25)
```

---

## 5. Properties with Get/Set (Properties แบบ Get/Set)

```fsharp
type Temperature(celsius: float) =
    let mutable _celsius = celsius
    
    // Read-only property
    member this.Celsius = _celsius
    
    // Computed property
    member this.Fahrenheit = _celsius * 9.0 / 5.0 + 32.0
    member this.Kelvin = _celsius + 273.15
    
    // Property with getter and setter
    member this.CelsiusRW
        with get() = _celsius
        and set(value) =
            if value < -273.15 then
                failwith "อุณหภูมิต่ำกว่า absolute zero!"
            _celsius <- value
    
    member this.FahrenheitRW
        with get() = this.Fahrenheit
        and set(value) =
            this.CelsiusRW <- (value - 32.0) * 5.0 / 9.0
    
    override this.ToString() =
        sprintf "%.2f°C = %.2f°F = %.2fK" _celsius this.Fahrenheit this.Kelvin

let temp = Temperature(100.0)
printfn "%s" (temp.ToString())  // 100.00°C = 212.00°F = 373.15K

temp.CelsiusRW <- 0.0
printfn "%s" (temp.ToString())  // 0.00°C = 32.00°F = 273.15K

temp.FahrenheitRW <- 98.6
printfn "Body temp: %s" (temp.ToString())  // 37.00°C

try
    temp.CelsiusRW <- -300.0
with
| Failure msg -> printfn "Error: %s" msg
```

---

## 6. Auto-Properties

```fsharp
type Product(id: int, name: string, price: decimal) =
    // Auto-properties with getters only (immutable)
    member val Id = id
    member val Name = name
    
    // Auto-property with getter and setter (mutable)
    member val Price = price with get, set
    member val InStock = true with get, set
    member val Quantity = 0 with get, set
    
    // Computed property
    member this.TotalValue =
        if this.InStock then
            this.Price * decimal this.Quantity
        else
            0M
    
    override this.ToString() =
        sprintf "Product(id=%d, name=%s, price=%.2M, qty=%d)" 
            id name this.Price this.Quantity

let laptop = Product(1, "Laptop", 25000M)
printfn "%s" (laptop.ToString())

laptop.Price <- 23000M  // ลดราคา
laptop.InStock <- true
laptop.Quantity <- 5

printfn "%s" (laptop.ToString())
printfn "Total value: %.2M" laptop.TotalValue
```

---

## 7. Static Members (สมาชิก Static)

```fsharp
type MathHelper() =
    // Static methods
    static member Square(x: float) = x * x
    static member Cube(x: float) = x * x * x
    static member Abs(x: float) = if x < 0.0 then -x else x
    
    static member Max(x: float, y: float) = if x > y then x else y
    static member Min(x: float, y: float) = if x < y then x else y
    
    static member Clamp(value: float, min: float, max: float) =
        MathHelper.Max(min, MathHelper.Min(max, value))
    
    static member Lerp(a: float, b: float, t: float) =
        a + (b - a) * t

// เรียกใช้โดยไม่ต้องสร้าง instance
printfn "Square(5) = %.0f" (MathHelper.Square(5.0))    // 25
printfn "Cube(3) = %.0f" (MathHelper.Cube(3.0))        // 27
printfn "Max(3, 7) = %.0f" (MathHelper.Max(3.0, 7.0))  // 7
printfn "Clamp(15, 0, 10) = %.0f" (MathHelper.Clamp(15.0, 0.0, 10.0))  // 10
printfn "Lerp(0, 100, 0.3) = %.0f" (MathHelper.Lerp(0.0, 100.0, 0.3))  // 30
```

---

## 8. Static Fields (Static Fields)

```fsharp
type Configuration() =
    // Static mutable fields (ใช้ร่วมกันทุก instance)
    static let mutable _debugMode = false
    static let mutable _logLevel = "INFO"
    static let mutable _maxConnections = 10
    
    // Static properties
    static member DebugMode
        with get() = _debugMode
        and set(value) = _debugMode <- value
    
    static member LogLevel
        with get() = _logLevel
        and set(value) = 
            match value with
            | "DEBUG" | "INFO" | "WARN" | "ERROR" -> _logLevel <- value
            | _ -> failwith (sprintf "Log level ไม่ถูกต้อง: %s" value)
    
    static member MaxConnections
        with get() = _maxConnections
        and set(value) = 
            if value <= 0 then failwith "MaxConnections ต้องมากกว่า 0"
            _maxConnections <- value
    
    // Static method
    static member Reset() =
        _debugMode <- false
        _logLevel <- "INFO"
        _maxConnections <- 10
    
    static member Summary() =
        sprintf "Debug=%b, LogLevel=%s, MaxConn=%d" 
            _debugMode _logLevel _maxConnections

printfn "%s" (Configuration.Summary())

Configuration.DebugMode <- true
Configuration.LogLevel <- "DEBUG"
Configuration.MaxConnections <- 50

printfn "%s" (Configuration.Summary())

Configuration.Reset()
printfn "After reset: %s" (Configuration.Summary())
```

### Static Counter Pattern

```fsharp
type Employee(name: string, department: string) =
    static let mutable totalEmployees = 0
    static let mutable nextId = 1
    
    let id = nextId
    
    do
        totalEmployees <- totalEmployees + 1
        nextId <- nextId + 1
    
    member this.Id = id
    member this.Name = name
    member this.Department = department
    
    static member TotalEmployees = totalEmployees
    static member NextId = nextId
    
    override this.ToString() =
        sprintf "Employee(id=%d, name=%s, dept=%s)" id name department

let emp1 = Employee("สมชาย", "IT")
let emp2 = Employee("สมหญิง", "HR")
let emp3 = Employee("สมศักดิ์", "Finance")

printfn "%s" (emp1.ToString())  // id=1
printfn "%s" (emp2.ToString())  // id=2
printfn "%s" (emp3.ToString())  // id=3
printfn "Total employees: %d" Employee.TotalEmployees  // 3
```

---

## 9. Self-Identifier (this)

```fsharp
type Builder() =
    let mutable _parts: string list = []
    
    // this เป็น self-identifier (สามารถตั้งชื่ออื่นได้)
    member this.Add(part: string) =
        _parts <- part :: _parts
        this  // return this เพื่อให้ chain ได้
    
    member this.AddMany(parts: string list) =
        for part in parts do
            _parts <- part :: _parts
        this
    
    member this.Build() =
        _parts |> List.rev |> String.concat ", "
    
    member this.Clear() =
        _parts <- []
        this

let result = 
    Builder()
        .Add("Hello")
        .Add("World")
        .AddMany(["from"; "F#"])
        .Build()

printfn "%s" result  // Hello, World, from, F#
```

### Using Different Self-Identifier Names

```fsharp
type LinkedList<'T>() =
    let mutable head: ('T * LinkedList<'T>) option = None
    
    // ใช้ 'self' แทน 'this'
    member self.Push(value: 'T) =
        head <- Some (value, self)
        self
    
    // ใช้ 'me' แทน 'this'
    member me.Pop() =
        match head with
        | None -> failwith "List is empty"
        | Some (value, _) ->
            head <- None
            value
    
    // ใช้ 'x' แทน 'this'
    member x.Peek() =
        match head with
        | None -> None
        | Some (value, _) -> Some value
    
    member _.IsEmpty =
        head.IsNone
```

---

## 10. Member Visibility (การมองเห็นของสมาชิก)

```fsharp
type BankAccountPrivate(initialBalance: decimal) =
    let mutable balance = initialBalance
    let mutable transactionHistory: string list = []
    
    // Private method - เรียกใช้ได้ภายในคลาสเท่านั้น
    let logTransaction (description: string) =
        let entry = sprintf "[%s] %s" (System.DateTime.Now.ToString("HH:mm:ss")) description
        transactionHistory <- entry :: transactionHistory
    
    // Private member method
    member private this.ValidateAmount(amount: decimal) =
        if amount <= 0M then
            invalidArg "amount" "จำนวนเงินต้องมากกว่าศูนย์"
    
    // Internal member - มองเห็นได้ภายใน assembly เดียวกัน
    member internal this.GetRawBalance() = balance
    
    // Public members
    member this.Balance = balance
    
    member this.Deposit(amount: decimal) =
        this.ValidateAmount(amount)
        balance <- balance + amount
        logTransaction (sprintf "Deposit: +%.2M (Balance: %.2M)" amount balance)
    
    member this.Withdraw(amount: decimal) =
        this.ValidateAmount(amount)
        if amount > balance then
            failwith "ยอดเงินไม่เพียงพอ"
        balance <- balance - amount
        logTransaction (sprintf "Withdraw: -%.2M (Balance: %.2M)" amount balance)
    
    member this.GetHistory() =
        transactionHistory |> List.rev

let account = BankAccountPrivate(1000M)
account.Deposit(500M)
account.Withdraw(200M)

printfn "Balance: %.2M" account.Balance
printfn "\nTransaction history:"
account.GetHistory() |> List.iter (printfn "  %s")
```

---

## 11. Overriding ToString()

```fsharp
type Matrix(rows: int, cols: int, data: float[,]) =
    do
        if Array2D.length1 data <> rows || Array2D.length2 data <> cols then
            failwith "ขนาด data ไม่ตรงกับ rows/cols"
    
    member this.Rows = rows
    member this.Cols = cols
    member this.Item(i, j) = data.[i, j]
    
    static member Zero(rows: int, cols: int) =
        Matrix(rows, cols, Array2D.zeroCreate rows cols)
    
    static member Identity(n: int) =
        let data = Array2D.init n n (fun i j -> if i = j then 1.0 else 0.0)
        Matrix(n, n, data)
    
    override this.ToString() =
        let sb = System.Text.StringBuilder()
        sb.AppendLine(sprintf "Matrix %dx%d:" rows cols) |> ignore
        for i in 0 .. rows - 1 do
            sb.Append("  [") |> ignore
            for j in 0 .. cols - 1 do
                if j > 0 then sb.Append(", ") |> ignore
                sb.Append(sprintf "%6.2f" data.[i, j]) |> ignore
            sb.AppendLine("]") |> ignore
        sb.ToString()

let identity = Matrix.Identity(3)
printfn "%s" (identity.ToString())
// Matrix 3x3:
//   [  1.00,   0.00,   0.00]
//   [  0.00,   1.00,   0.00]
//   [  0.00,   0.00,   1.00]
```

---

## 12. Overriding Equals() and GetHashCode()

```fsharp
type Fraction(numerator: int, denominator: int) =
    do
        if denominator = 0 then
            invalidArg "denominator" "ตัวหารต้องไม่เป็นศูนย์"
    
    // ลด fraction ให้เป็น simplest form
    let gcd a b =
        let rec gcd' a b = if b = 0 then a else gcd' b (a % b)
        gcd' (abs a) (abs b)
    
    let sign = if (numerator < 0) <> (denominator < 0) then -1 else 1
    let g = gcd (abs numerator) (abs denominator)
    let num = sign * abs numerator / g
    let den = abs denominator / g
    
    member this.Numerator = num
    member this.Denominator = den
    
    member this.ToFloat() = float num / float den
    
    member this.Add(other: Fraction) =
        Fraction(num * other.Denominator + other.Numerator * den, den * other.Denominator)
    
    member this.Multiply(other: Fraction) =
        Fraction(num * other.Numerator, den * other.Denominator)
    
    override this.Equals(obj) =
        match obj with
        | :? Fraction as other -> num = other.Numerator && den = other.Denominator
        | _ -> false
    
    override this.GetHashCode() =
        // ต้อง consistent กับ Equals
        hash (num, den)
    
    override this.ToString() =
        if den = 1 then sprintf "%d" num
        else sprintf "%d/%d" num den

let f1 = Fraction(1, 2)
let f2 = Fraction(2, 4)  // เท่ากับ 1/2
let f3 = Fraction(3, 4)

printfn "f1 = %s" (f1.ToString())   // 1/2
printfn "f2 = %s" (f2.ToString())   // 1/2
printfn "f3 = %s" (f3.ToString())   // 3/4

printfn "f1 = f2: %b" (f1.Equals(f2))    // true
printfn "f1 = f3: %b" (f1.Equals(f3))    // false
printfn "f1.GetHashCode() = f2.GetHashCode(): %b" (f1.GetHashCode() = f2.GetHashCode())  // true

let sum = f1.Add(f3)
printfn "1/2 + 3/4 = %s" (sum.ToString())  // 5/4

// ใช้ใน Dictionary
let fractionMap = System.Collections.Generic.Dictionary<Fraction, string>()
fractionMap.[f1] <- "หนึ่งส่วนสอง"
printfn "f2 in map: %s" fractionMap.[f2]  // หนึ่งส่วนสอง (เพราะ f1 = f2)
```

---

## 13. Implementing IComparable

```fsharp
type Version(major: int, minor: int, patch: int) =
    interface System.IComparable with
        member this.CompareTo(obj) =
            match obj with
            | :? Version as other ->
                let cmp = compare major other.Major
                if cmp <> 0 then cmp
                else
                    let cmp = compare minor other.Minor
                    if cmp <> 0 then cmp
                    else compare patch other.Patch
            | _ -> failwith "Cannot compare with non-Version object"
    
    interface System.IComparable<Version> with
        member this.CompareTo(other) =
            let cmp = compare major other.Major
            if cmp <> 0 then cmp
            else
                let cmp = compare minor other.Minor
                if cmp <> 0 then cmp
                else compare patch other.Patch
    
    member this.Major = major
    member this.Minor = minor
    member this.Patch = patch
    
    override this.Equals(obj) =
        match obj with
        | :? Version as other ->
            major = other.Major && minor = other.Minor && patch = other.Patch
        | _ -> false
    
    override this.GetHashCode() = hash (major, minor, patch)
    
    override this.ToString() =
        sprintf "%d.%d.%d" major minor patch

let v1 = Version(1, 0, 0)
let v2 = Version(1, 2, 0)
let v3 = Version(2, 0, 0)
let v4 = Version(1, 0, 5)

let versions = [v3; v1; v4; v2]
let sorted = versions |> List.sortWith (fun a b -> (a :> System.IComparable<Version>).CompareTo(b))

printfn "Sorted versions:"
sorted |> List.iter (fun v -> printfn "  %s" (v.ToString()))
// 1.0.0, 1.0.5, 1.2.0, 2.0.0
```

---

## 14. Readonly and Mutable Fields

```fsharp
type ImmutablePoint(x: float, y: float) =
    // Readonly - ค่าถูก set ใน constructor และไม่เปลี่ยนแปลง
    let _x = x  // private readonly
    let _y = y  // private readonly
    
    member this.X = _x
    member this.Y = _y
    
    // สร้าง point ใหม่แทนการแก้ไข
    member this.WithX(newX: float) = ImmutablePoint(newX, _y)
    member this.WithY(newY: float) = ImmutablePoint(_x, newY)
    member this.Translate(dx, dy) = ImmutablePoint(_x + dx, _y + dy)
    
    override this.ToString() = sprintf "(%.2f, %.2f)" _x _y

type MutablePoint(x: float, y: float) =
    let mutable _x = x
    let mutable _y = y
    
    member this.X
        with get() = _x
        and set(value) = _x <- value
    
    member this.Y
        with get() = _y
        and set(value) = _y <- value
    
    member this.Translate(dx: float, dy: float) =
        _x <- _x + dx
        _y <- _y + dy
    
    override this.ToString() = sprintf "(%.2f, %.2f)" _x _y

// Immutable
let ip1 = ImmutablePoint(1.0, 2.0)
let ip2 = ip1.Translate(3.0, 4.0)
printfn "ip1: %s (unchanged)" (ip1.ToString())  // (1.00, 2.00)
printfn "ip2: %s (new point)" (ip2.ToString())  // (4.00, 6.00)

// Mutable
let mp = MutablePoint(1.0, 2.0)
printfn "before: %s" (mp.ToString())
mp.Translate(3.0, 4.0)
printfn "after: %s" (mp.ToString())   // (4.00, 6.00) - same object modified
```

---

## 15. let and do Bindings in Class Body

```fsharp
type DatabaseConnection(connectionString: string) =
    // let bindings - private fields/functions
    let mutable isOpen = false
    let mutable queryCount = 0
    let startTime = System.DateTime.Now
    
    // Private helper function ด้วย let
    let validateConnectionString (cs: string) =
        if System.String.IsNullOrEmpty(cs) then
            failwith "Connection string ต้องไม่ว่าง"
        if not (cs.Contains("Server=")) then
            failwith "Connection string ต้องมี Server="
    
    let formatQuery (sql: string) =
        sql.Trim().ToUpper()
    
    // do binding - code ที่รันเมื่อสร้าง instance
    do
        printfn "Initializing connection..."
        validateConnectionString connectionString
        printfn "Connection validated: %s" (connectionString.Split(';').[0])
    
    // Another do binding
    do
        printfn "Connection created at %s" (startTime.ToString("HH:mm:ss"))
    
    member this.ConnectionString = connectionString
    member this.IsOpen = isOpen
    member this.QueryCount = queryCount
    member this.Uptime = System.DateTime.Now - startTime
    
    member this.Open() =
        if isOpen then failwith "Connection already open"
        isOpen <- true
        printfn "Connection opened"
    
    member this.Close() =
        if not isOpen then failwith "Connection is not open"
        isOpen <- false
        printfn "Connection closed"
    
    member this.ExecuteQuery(sql: string) =
        if not isOpen then failwith "Connection is not open"
        let formatted = formatQuery sql
        queryCount <- queryCount + 1
        printfn "Query #%d: %s" queryCount formatted
        sprintf "Results for: %s" formatted

// ทดสอบ
let conn = DatabaseConnection("Server=localhost;Database=mydb;")
conn.Open()
let result = conn.ExecuteQuery("SELECT * FROM users")
printfn "Result: %s" result
printfn "Queries executed: %d" conn.QueryCount
conn.Close()
```

---

## 16. Generic Classes

```fsharp
// Generic Stack
type Stack<'T>() =
    let mutable items: 'T list = []
    
    member this.Push(item: 'T) =
        items <- item :: items
    
    member this.Pop() =
        match items with
        | [] -> failwith "Stack is empty"
        | head :: tail ->
            items <- tail
            head
    
    member this.Peek() =
        match items with
        | [] -> failwith "Stack is empty"
        | head :: _ -> head
    
    member this.IsEmpty = items.IsEmpty
    member this.Count = items.Length
    
    member this.ToList() = List.rev items
    
    override this.ToString() =
        sprintf "Stack[%s]" (items |> List.map (sprintf "%A") |> String.concat ", ")

// Generic Pair
type Pair<'T, 'U>(first: 'T, second: 'U) =
    member this.First = first
    member this.Second = second
    
    member this.Swap() = Pair<'U, 'T>(second, first)
    
    member this.Map(f: 'T -> 'A, g: 'U -> 'B) =
        Pair<'A, 'B>(f first, g second)
    
    override this.ToString() =
        sprintf "(%A, %A)" first second

// Generic Result Container
type Container<'T>(value: 'T) =
    let mutable _value = value
    
    member this.Value = _value
    
    member this.Map(f: 'T -> 'U) =
        Container<'U>(f _value)
    
    member this.Update(f: 'T -> 'T) =
        _value <- f _value
        this

// ทดสอบ Generic Stack
let intStack = Stack<int>()
intStack.Push(1)
intStack.Push(2)
intStack.Push(3)
printfn "Stack: %s" (intStack.ToString())  // Stack[3, 2, 1]
printfn "Pop: %d" (intStack.Pop())         // 3
printfn "Peek: %d" (intStack.Peek())       // 2

let strStack = Stack<string>()
strStack.Push("hello")
strStack.Push("world")
printfn "String stack: %s" (strStack.ToString())

// ทดสอบ Pair
let pair = Pair<string, int>("age", 25)
printfn "Pair: %s" (pair.ToString())
let swapped = pair.Swap()
printfn "Swapped: %s" (swapped.ToString())

// ทดสอบ Container
let c = Container<int>(42)
let c2 = c.Map(fun x -> x * 2)
printfn "Container value: %d" c2.Value
```

---

## 17. Practical Example: BankAccount

```fsharp
type TransactionType =
    | Deposit
    | Withdrawal
    | Transfer

type Transaction = {
    Type: TransactionType
    Amount: decimal
    Description: string
    Timestamp: System.DateTime
    Balance: decimal
}

type BankAccount(accountNumber: string, owner: string, initialBalance: decimal) =
    do
        if System.String.IsNullOrEmpty(accountNumber) then
            invalidArg "accountNumber" "เลขบัญชีต้องไม่ว่าง"
        if System.String.IsNullOrEmpty(owner) then
            invalidArg "owner" "ชื่อเจ้าของต้องไม่ว่าง"
        if initialBalance < 0M then
            invalidArg "initialBalance" "ยอดเงินเริ่มต้นต้องไม่ติดลบ"
    
    let mutable _balance = initialBalance
    let mutable _transactions: Transaction list = []
    let mutable _isActive = true
    
    let addTransaction txType amount desc =
        let tx = {
            Type = txType
            Amount = amount
            Description = desc
            Timestamp = System.DateTime.Now
            Balance = _balance
        }
        _transactions <- tx :: _transactions
    
    let validateActive () =
        if not _isActive then failwith "บัญชีถูกปิดแล้ว"
    
    let validateAmount (amount: decimal) (minAmount: decimal) =
        if amount < minAmount then
            invalidArg "amount" (sprintf "จำนวนเงินต้องมากกว่า %.2M" minAmount)
    
    // Properties
    member this.AccountNumber = accountNumber
    member this.Owner = owner
    member this.Balance = _balance
    member this.IsActive = _isActive
    member this.TransactionCount = _transactions.Length
    
    // Methods
    member this.Deposit(amount: decimal, ?description: string) =
        validateActive()
        validateAmount amount 0.01M
        _balance <- _balance + amount
        let desc = defaultArg description "Deposit"
        addTransaction Deposit amount desc
        printfn "ฝากเงิน %.2M บาท - ยอดคงเหลือ: %.2M บาท" amount _balance
    
    member this.Withdraw(amount: decimal, ?description: string) =
        validateActive()
        validateAmount amount 0.01M
        if amount > _balance then
            failwith (sprintf "ยอดเงินไม่เพียงพอ (ต้องการ %.2M มีอยู่ %.2M)" amount _balance)
        _balance <- _balance - amount
        let desc = defaultArg description "Withdrawal"
        addTransaction Withdrawal amount desc
        printfn "ถอนเงิน %.2M บาท - ยอดคงเหลือ: %.2M บาท" amount _balance
    
    member this.Transfer(amount: decimal, targetAccount: BankAccount) =
        validateActive()
        targetAccount |> (fun ta -> if not ta.IsActive then failwith "บัญชีปลายทางถูกปิดแล้ว")
        this.Withdraw(amount, sprintf "โอนไปบัญชี %s" targetAccount.AccountNumber)
        targetAccount.Deposit(amount, sprintf "รับโอนจากบัญชี %s" accountNumber)
    
    member this.GetStatement() =
        let header = sprintf "=== Statement: %s (%s) ===" accountNumber owner
        let lines = 
            _transactions 
            |> List.rev 
            |> List.map (fun tx ->
                sprintf "%s | %-10s | %10.2M | Balance: %10.2M | %s"
                    (tx.Timestamp.ToString("dd/MM HH:mm"))
                    (sprintf "%A" tx.Type)
                    tx.Amount
                    tx.Balance
                    tx.Description
            )
        header :: lines |> String.concat "\n"
    
    member this.Close() =
        validateActive()
        _isActive <- false
        printfn "ปิดบัญชี %s" accountNumber
    
    override this.ToString() =
        sprintf "BankAccount(%s, owner=%s, balance=%.2M, active=%b)" 
            accountNumber owner _balance _isActive

// ทดสอบ BankAccount
printfn "=== ทดสอบ BankAccount ==="
let acc1 = BankAccount("1234567890", "สมชาย ใจดี", 10000M)
let acc2 = BankAccount("0987654321", "สมหญิง ใจดี", 5000M)

printfn "\n--- Deposit ---"
acc1.Deposit(5000M, "เงินเดือน")
acc1.Deposit(1000M, "โบนัส")

printfn "\n--- Withdraw ---"
acc1.Withdraw(2000M, "ค่าเช่า")

printfn "\n--- Transfer ---"
acc1.Transfer(3000M, acc2)

printfn "\n--- Balance ---"
printfn "acc1: %.2M" acc1.Balance
printfn "acc2: %.2M" acc2.Balance

printfn "\n%s" (acc1.GetStatement())
```

---

## 18. Practical Example: Stack Implementation

```fsharp
type StackException(message: string) =
    inherit System.Exception(message)

type Stack2<'T>() =
    let mutable items: 'T[] = Array.zeroCreate 4
    let mutable count = 0
    
    let resize () =
        let newItems = Array.zeroCreate (items.Length * 2)
        Array.blit items 0 newItems 0 count
        items <- newItems
    
    member this.Count = count
    member this.IsEmpty = count = 0
    member this.Capacity = items.Length
    
    member this.Push(item: 'T) =
        if count = items.Length then resize()
        items.[count] <- item
        count <- count + 1
    
    member this.Pop() =
        if count = 0 then raise (StackException("Stack is empty"))
        count <- count - 1
        let item = items.[count]
        items.[count] <- Unchecked.defaultof<'T>  // clear reference
        item
    
    member this.Peek() =
        if count = 0 then raise (StackException("Stack is empty"))
        items.[count - 1]
    
    member this.TryPop() =
        if count = 0 then None
        else Some (this.Pop())
    
    member this.TryPeek() =
        if count = 0 then None
        else Some this.Peek
    
    member this.Clear() =
        for i in 0 .. count - 1 do
            items.[i] <- Unchecked.defaultof<'T>
        count <- 0
    
    member this.Contains(item: 'T) =
        let mutable found = false
        let mutable i = 0
        while not found && i < count do
            if items.[i] = item then found <- true
            i <- i + 1
        found
    
    member this.ToArray() =
        items.[0..count-1] |> Array.rev
    
    override this.ToString() =
        let elements = items.[0..count-1] |> Array.rev |> Array.map (sprintf "%A") |> String.concat ", "
        sprintf "Stack([%s], count=%d, capacity=%d)" elements count items.Length

// ทดสอบ Stack2
let stack = Stack2<int>()
for i in 1..10 do
    stack.Push(i)

printfn "%s" (stack.ToString())
printfn "Count: %d" stack.Count
printfn "Capacity: %d" stack.Capacity  // 16 (resized from 4 -> 8 -> 16)

printfn "Pop: %d" (stack.Pop())  // 10
printfn "Peek: %d" (stack.Peek())  // 9
printfn "Contains 5: %b" (stack.Contains(5))  // true
printfn "Contains 15: %b" (stack.Contains(15))  // false

// ทดสอบ exception handling
try
    let emptyStack = Stack2<int>()
    emptyStack.Pop() |> ignore
with
| :? StackException as ex ->
    printfn "Exception: %s" ex.Message

// Expression evaluator using stack
let evalPostfix (expression: string) =
    let stack = Stack2<float>()
    
    for token in expression.Split(' ') do
        match System.Double.TryParse(token) with
        | true, num -> stack.Push(num)
        | _ ->
            let b = stack.Pop()
            let a = stack.Pop()
            let result = 
                match token with
                | "+" -> a + b
                | "-" -> a - b
                | "*" -> a * b
                | "/" -> a / b
                | _ -> failwith (sprintf "Unknown operator: %s" token)
            stack.Push(result)
    
    stack.Pop()

printfn "\nPostfix evaluation:"
printfn "3 4 + = %.0f" (evalPostfix "3 4 +")         // 7
printfn "5 1 2 + 4 * + 3 - = %.0f" (evalPostfix "5 1 2 + 4 * + 3 -")  // 14
```

---

## 19. Practical Example: Queue

```fsharp
type Queue<'T>() =
    // ใช้ two-list queue algorithm สำหรับ O(1) amortized operations
    let mutable inbox: 'T list = []   // สำหรับ enqueue
    let mutable outbox: 'T list = []  // สำหรับ dequeue
    let mutable count = 0
    
    let normalize () =
        if outbox.IsEmpty then
            outbox <- List.rev inbox
            inbox <- []
    
    member this.Count = count
    member this.IsEmpty = count = 0
    
    member this.Enqueue(item: 'T) =
        inbox <- item :: inbox
        count <- count + 1
    
    member this.Dequeue() =
        if count = 0 then failwith "Queue is empty"
        normalize()
        match outbox with
        | [] -> failwith "Queue is empty"
        | head :: tail ->
            outbox <- tail
            count <- count - 1
            head
    
    member this.Peek() =
        if count = 0 then failwith "Queue is empty"
        normalize()
        match outbox with
        | [] -> failwith "Queue is empty"
        | head :: _ -> head
    
    member this.TryDequeue() =
        if count = 0 then None
        else Some (this.Dequeue())
    
    member this.Clear() =
        inbox <- []
        outbox <- []
        count <- 0
    
    member this.ToList() =
        List.append (List.rev outbox) (List.rev inbox)
    
    override this.ToString() =
        let items = this.ToList() |> List.map (sprintf "%A") |> String.concat ", "
        sprintf "Queue([%s], count=%d)" items count

// Priority Queue
type PriorityQueue<'T when 'T : comparison>() =
    let mutable heap: 'T[] = Array.zeroCreate 8
    let mutable count = 0
    
    let swap i j =
        let temp = heap.[i]
        heap.[i] <- heap.[j]
        heap.[j] <- temp
    
    let parent i = (i - 1) / 2
    let leftChild i = 2 * i + 1
    let rightChild i = 2 * i + 2
    
    let rec bubbleUp i =
        if i > 0 then
            let p = parent i
            if heap.[i] < heap.[p] then
                swap i p
                bubbleUp p
    
    let rec bubbleDown i =
        let left = leftChild i
        let right = rightChild i
        let mutable smallest = i
        
        if left < count && heap.[left] < heap.[smallest] then
            smallest <- left
        if right < count && heap.[right] < heap.[smallest] then
            smallest <- right
        
        if smallest <> i then
            swap i smallest
            bubbleDown smallest
    
    member this.Count = count
    member this.IsEmpty = count = 0
    
    member this.Enqueue(item: 'T) =
        if count = heap.Length then
            let newHeap = Array.zeroCreate (heap.Length * 2)
            Array.blit heap 0 newHeap 0 count
            heap <- newHeap
        heap.[count] <- item
        count <- count + 1
        bubbleUp (count - 1)
    
    member this.Dequeue() =
        if count = 0 then failwith "Priority Queue is empty"
        let min = heap.[0]
        count <- count - 1
        heap.[0] <- heap.[count]
        heap.[count] <- Unchecked.defaultof<'T>
        if count > 0 then bubbleDown 0
        min
    
    member this.Peek() =
        if count = 0 then failwith "Priority Queue is empty"
        heap.[0]

// ทดสอบ Queue
printfn "\n=== Queue Test ==="
let queue = Queue<string>()
queue.Enqueue("Task 1")
queue.Enqueue("Task 2")
queue.Enqueue("Task 3")
printfn "%s" (queue.ToString())
printfn "Dequeue: %s" (queue.Dequeue())  // Task 1
printfn "Peek: %s" (queue.Peek())         // Task 2
printfn "%s" (queue.ToString())

// ทดสอบ Priority Queue
printfn "\n=== Priority Queue Test ==="
let pq = PriorityQueue<int>()
for x in [5; 3; 8; 1; 9; 2; 7] do
    pq.Enqueue(x)

printfn "Sorted output:"
while not pq.IsEmpty do
    printf "%d " (pq.Dequeue())
printfn ""  // 1 2 3 5 7 8 9
```

---

## 20. สรุป (Summary)

```fsharp
// Summary of F# Classes

// 1. Basic class with primary constructor
type Animal(name: string, sound: string) =
    member this.Name = name
    member this.Sound = sound
    member this.Speak() = printfn "%s says %s!" name sound
    override this.ToString() = sprintf "Animal(%s)" name

// 2. Inheritance (see Part 33 for full details)
type Dog(name: string) =
    inherit Animal(name, "Woof")
    member this.Fetch() = printfn "%s fetches the ball!" name

// 3. Class with interface implementation
type Sortable(value: int) =
    interface System.IComparable with
        member this.CompareTo(other) =
            match other with
            | :? Sortable as s -> compare value s.Value
            | _ -> failwith "Cannot compare"
    member this.Value = value
    override this.ToString() = sprintf "Sortable(%d)" value

// Demo
let animals = [| Animal("Cat", "Meow"); Animal("Cow", "Moo") |]
let dog = Dog("Rex")

for a in animals do a.Speak()
dog.Speak()
dog.Fetch()

let sortables = [| Sortable(5); Sortable(2); Sortable(8); Sortable(1) |]
System.Array.Sort(sortables)
printfn "Sorted: %s" (sortables |> Array.map (fun s -> s.ToString()) |> String.concat ", ")

printfn "\n=== Key Concepts ==="
printfn "1. Primary Constructor: type MyClass(param1, param2) = ..."
printfn "2. Additional Constructors: new(altParams) = MyClass(...)"
printfn "3. Instance Members: member this.Name = ..."
printfn "4. Static Members: static member Name = ..."
printfn "5. Properties: member this.Prop with get() = ... and set(v) = ..."
printfn "6. Auto-properties: member val Prop = value with get, set"
printfn "7. Private fields: let mutable field = value"
printfn "8. Initialization: do ... (runs in constructor)"
```

---

## สรุปบทเรียน

ในบทนี้เราได้เรียนรู้:

1. **Class definition syntax** - การนิยามคลาสด้วย primary constructor
2. **Additional constructors** - การเพิ่ม constructor ทางเลือก
3. **Instance members** - methods และ properties ของ instance
4. **Static members** - สมาชิกที่ใช้ร่วมกันทุก instance
5. **Auto-properties** - properties ที่สร้างได้ง่ายด้วย `member val`
6. **Visibility** - public, private, internal
7. **Overriding** - ToString(), Equals(), GetHashCode()
8. **Generic classes** - คลาสที่รับ type parameter
9. **Practical examples** - BankAccount, Stack, Queue

คลาสใน F# มีความสามารถเทียบเท่ากับ C# และ Java แต่มีไวยากรณ์ที่กระชับกว่าและรวมกับ Functional Programming ได้ดี
