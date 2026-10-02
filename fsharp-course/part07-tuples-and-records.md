# Part 7 - ทูเพิลและเรคคอร์ด (Tuples and Records)

## บทนำ

ใน F# เรามีสองวิธีหลักในการรวมข้อมูลหลายชิ้นเข้าด้วยกัน: Tuples (กลุ่มข้อมูลที่ไม่มีชื่อ) และ Records (กลุ่มข้อมูลที่มีชื่อ field) ทั้งสองเป็น immutable โดยค่าเริ่มต้นและมีประโยชน์ในบริบทที่ต่างกัน

---

## 7.1 Tuples พื้นฐาน

### 7.1.1 การสร้าง Tuple

```fsharp
// Tuple: grouping ข้อมูลหลายชนิดเข้าด้วยกัน
// ใช้ parentheses และ comma

// 2-tuple (pair)
let point = (3, 4)
let nameAge = ("Alice", 30)
let priceQty = (29.99, 100)

// 3-tuple (triple)
let rgb = (255, 128, 0)
let nameAgeCity = ("Bob", 25, "Bangkok")

// 4-tuple
let quad = (1, "two", 3.0, true)

// Type annotations
let typedPair: int * string = (42, "hello")
let typedTriple: string * int * bool = ("test", 5, true)

printfn "point: %A" point
printfn "nameAge: %A" nameAge
printfn "rgb: %A" rgb
printfn "quad: %A" quad

// Nested tuples
let nested = ((1, 2), (3, 4))
printfn "nested: %A" nested

// Tuple in list
let points = [(0, 0); (1, 2); (3, 4); (5, 6)]
printfn "points: %A" points
```

### 7.1.2 fst และ snd Functions

```fsharp
// fst: เอา first element จาก pair
// snd: เอา second element จาก pair

let pair = ("hello", 42)

let first = fst pair    // "hello"
let second = snd pair   // 42

printfn "fst: %s" first
printfn "snd: %d" second

// ใช้ใน list operations
let pairs = [("Alice", 90); ("Bob", 85); ("Charlie", 92)]

let names = pairs |> List.map fst
let scores = pairs |> List.map snd

printfn "Names: %A" names
printfn "Scores: %A" scores

// Sorting by second element
let sortedByScore = pairs |> List.sortByDescending snd
printfn "Sorted by score: %A" sortedByScore
```

### 7.1.3 Tuple Decomposition

```fsharp
// Destructuring / decomposition ด้วย let

let (x, y) = (10, 20)
printfn "x=%d, y=%d" x y

let (name, age, city) = ("Alice", 30, "Bangkok")
printfn "Name: %s, Age: %d, City: %s" name age city

// ละเว้น elements ที่ไม่ต้องการ
let (_, b, _) = (1, 2, 3)
printfn "Middle: %d" b

// ใน function parameters
let addPoint (x1, y1) (x2, y2) = (x1 + x2, y1 + y2)
let p1 = (1, 2)
let p2 = (3, 4)
let p3 = addPoint p1 p2
printfn "Sum of points: %A" p3

// Decomposition ใน match
let classifyPoint point =
    match point with
    | (0, 0) -> "origin"
    | (x, 0) -> sprintf "on x-axis at %d" x
    | (0, y) -> sprintf "on y-axis at %d" y
    | (x, y) when x = y -> sprintf "on diagonal at (%d,%d)" x y
    | (x, y) -> sprintf "at (%d,%d)" x y

let testPoints = [(0,0); (5,0); (0,3); (4,4); (2,7)]
testPoints |> List.iter (fun p -> printfn "%A -> %s" p (classifyPoint p))

// Swap function
let swap (a, b) = (b, a)
printfn "swap (1, 'a'): %A" (swap (1, "a"))
printfn "swap ('hello', 42): %A" (swap ("hello", 42))
```

---

## 7.2 Tuples ใน Pattern Matching

```fsharp
// Tuple pattern matching scenarios

// Multiple return values
let divmod a b =
    if b = 0 then None
    else Some (a / b, a % b)

match divmod 17 5 with
| Some (quotient, remainder) ->
    printfn "17 / 5 = %d remainder %d" quotient remainder
| None ->
    printfn "Division by zero!"

// Comparing coordinates
let direction (x1, y1) (x2, y2) =
    match (x2 - x1, y2 - y1) with
    | (dx, 0) when dx > 0 -> "East"
    | (dx, 0) when dx < 0 -> "West"
    | (0, dy) when dy > 0 -> "North"
    | (0, dy) when dy < 0 -> "South"
    | (dx, dy) when dx > 0 && dy > 0 -> "Northeast"
    | (dx, dy) when dx < 0 && dy > 0 -> "Northwest"
    | (dx, dy) when dx > 0 && dy < 0 -> "Southeast"
    | (dx, dy) when dx < 0 && dy < 0 -> "Southwest"
    | _ -> "Same location"

printfn "%s" (direction (0, 0) (5, 0))
printfn "%s" (direction (0, 0) (0, 3))
printfn "%s" (direction (0, 0) (4, 3))
printfn "%s" (direction (0, 0) (-2, -2))

// Validate ranges
let validateCoordinate (lat, lon) =
    match (lat, lon) with
    | (lat, _) when lat < -90.0 || lat > 90.0 ->
        Error $"Invalid latitude: {lat}"
    | (_, lon) when lon < -180.0 || lon > 180.0 ->
        Error $"Invalid longitude: {lon}"
    | coord ->
        Ok coord

match validateCoordinate (13.756, 100.502) with
| Ok (lat, lon) -> printfn "Valid: lat=%.3f, lon=%.3f" lat lon
| Error msg -> printfn "%s" msg

match validateCoordinate (100.0, 0.0) with
| Ok _ -> printfn "Valid"
| Error msg -> printfn "%s" msg
```

---

## 7.3 เมื่อไหร่ใช้ Tuple vs Record

```fsharp
// Tuples เหมาะสำหรับ:
// 1. Return multiple values จาก function
// 2. Temporary grouping
// 3. Known structure (ไม่กี่ elements)
// 4. Standard pair/triple

// Records เหมาะสำหรับ:
// 1. Domain objects ที่มีชื่อ fields
// 2. Configuration/settings
// 3. Complex structures
// 4. ต้องการ readability

// Tuple example: simple pairing
let coordinates = (13.756, 100.502)  // (latitude, longitude)
let minMax = (1, 100)                // (min, max)

// Record example: complex domain object
type Location = {
    Latitude: float
    Longitude: float
    Name: string
    Country: string
}

let bangkok = {
    Latitude = 13.756
    Longitude = 100.502
    Name = "Bangkok"
    Country = "Thailand"
}

// Tuple สำหรับ multiple returns
let parseDate (dateStr: string) =
    let parts = dateStr.Split('-')
    match parts with
    | [| y; m; d |] -> 
        try (int y, int m, int d) |> Some
        with _ -> None
    | _ -> None

match parseDate "2024-03-15" with
| Some (year, month, day) ->
    printfn "Date: %04d-%02d-%02d" year month day
| None ->
    printfn "Invalid date"
```

---

## 7.4 Record Type Definitions

### 7.4.1 Basic Record

```fsharp
// Define record type
type Person = {
    Name: string
    Age: int
    Email: string
}

// Create record
let alice = { Name = "Alice"; Age = 30; Email = "alice@example.com" }
let bob = { Name = "Bob"; Age = 25; Email = "bob@example.com" }

printfn "Alice: %A" alice
printfn "Bob: %A" bob

// Access fields
printfn "Name: %s" alice.Name
printfn "Age: %d" alice.Age

// Records เป็น immutable โดยค่าเริ่มต้น
// alice.Age <- 31  // Error!
```

### 7.4.2 Record Field Access

```fsharp
type Product = {
    Id: int
    Name: string
    Price: float
    Category: string
    InStock: bool
}

let widget = {
    Id = 101
    Name = "Super Widget"
    Price = 49.99
    Category = "Electronics"
    InStock = true
}

// Field access
printfn "Name: %s" widget.Name
printfn "Price: %.2f" widget.Price
printfn "In Stock: %b" widget.InStock

// ใช้ใน expressions
let discountedPrice = widget.Price * 0.9
let summary = $"{widget.Name}: ${discountedPrice:.2f}"
printfn "%s" summary

// Function ที่รับ record
let formatProduct (p: Product) =
    sprintf "[%d] %s (%.2f) - %s%s" 
        p.Id p.Name p.Price p.Category
        (if p.InStock then "" else " [OUT OF STOCK]")

printfn "%s" (formatProduct widget)
```

---

## 7.5 Record Update Syntax

```fsharp
// Copy record พร้อมเปลี่ยนบาง fields ด้วย { r with field = value }

type Config = {
    Host: string
    Port: int
    UseSSL: bool
    Timeout: int
    MaxConnections: int
}

let defaultConfig = {
    Host = "localhost"
    Port = 5432
    UseSSL = false
    Timeout = 30
    MaxConnections = 10
}

// สร้าง config ใหม่โดย copy และเปลี่ยนบาง fields
let productionConfig = { defaultConfig with
    Host = "db.production.com"
    Port = 5433
    UseSSL = true
    MaxConnections = 100 }

let testConfig = { defaultConfig with
    Host = "db.test.com"
    Timeout = 10 }

printfn "Default: %A" defaultConfig
printfn "Production: %A" productionConfig
printfn "Test: %A" testConfig

// Update ใน pipeline
let applyDiscount discount (product: Product) =
    { product with Price = product.Price * (1.0 - discount) }

let markAsOutOfStock (product: Product) =
    { product with InStock = false }

let processProduct product =
    product
    |> applyDiscount 0.1
    |> markAsOutOfStock

let processed = processProduct widget
printfn "\nProcessed: %A" processed
```

---

## 7.6 Recursive Records

```fsharp
// Records สามารถมี optional fields สำหรับ recursion

type Node = {
    Value: int
    Left: Node option
    Right: Node option
}

// สร้าง binary tree
let leaf value = { Value = value; Left = None; Right = None }

let node value left right = {
    Value = value
    Left = Some left
    Right = Some right
}

// Simple tree:
//       5
//      / \
//     3   8
//    / \
//   1   4

let tree = 
    node 5 
        (node 3 (leaf 1) (leaf 4))
        (leaf 8)

// Traverse tree
let rec sumTree node =
    let leftSum = node.Left |> Option.map sumTree |> Option.defaultValue 0
    let rightSum = node.Right |> Option.map sumTree |> Option.defaultValue 0
    node.Value + leftSum + rightSum

printfn "Tree sum: %d" (sumTree tree)  // 1+3+4+5+8 = 21

// Count nodes
let rec countNodes node =
    let leftCount = node.Left |> Option.map countNodes |> Option.defaultValue 0
    let rightCount = node.Right |> Option.map countNodes |> Option.defaultValue 0
    1 + leftCount + rightCount

printfn "Node count: %d" (countNodes tree)  // 5

// Inorder traversal
let rec inorder node =
    let left = node.Left |> Option.map inorder |> Option.defaultValue []
    let right = node.Right |> Option.map inorder |> Option.defaultValue []
    left @ [node.Value] @ right

printfn "Inorder: %A" (inorder tree)  // [1; 3; 4; 5; 8]
```

---

## 7.7 Anonymous Records

```fsharp
// Anonymous records: ไม่ต้อง define type ล่วงหน้า

// สร้าง anonymous record ด้วย {| ... |}
let anon = {| Name = "Alice"; Age = 30 |}
printfn "Anonymous: %A" anon
printfn "Name: %s" anon.Name
printfn "Age: %d" anon.Age

// ใช้ใน function return
let getUserInfo userId =
    {| Id = userId; Name = "User" + string userId; Active = true |}

let user = getUserInfo 42
printfn "User: %A" user

// Anonymous record ใน list
let items = [
    {| Title = "F# Book"; Price = 29.99 |}
    {| Title = "Haskell Book"; Price = 39.99 |}
    {| Title = "Rust Book"; Price = 34.99 |}
]

items |> List.iter (fun item -> 
    printfn "  %s: $%.2f" item.Title item.Price)

// ประโยชน์: ไม่ต้อง define type สำหรับ one-time use
let transformData data =
    data 
    |> List.map (fun (name, score) -> 
        {| Name = name
           Score = score
           Grade = if score >= 90 then "A" 
                   elif score >= 80 then "B" 
                   else "C" |})

let results = transformData [("Alice", 95.0); ("Bob", 82.0); ("Charlie", 75.0)]
results |> List.iter (fun r ->
    printfn "%s: %.1f (%s)" r.Name r.Score r.Grade)
```

---

## 7.8 Struct Records

```fsharp
// Struct records: value type (stack allocated) แทน reference type
// ใช้ [<Struct>] attribute

[<Struct>]
type Vector3D = {
    X: float
    Y: float
    Z: float
}

let v1 = { X = 1.0; Y = 2.0; Z = 3.0 }
let v2 = { X = 4.0; Y = 5.0; Z = 6.0 }

// Operations
let add v1 v2 = { X = v1.X + v2.X; Y = v1.Y + v2.Y; Z = v1.Z + v2.Z }
let scale s v = { X = v.X * s; Y = v.Y * s; Z = v.Z * s }
let dot v1 v2 = v1.X * v2.X + v1.Y * v2.Y + v1.Z * v2.Z
let magnitude v = sqrt (v.X*v.X + v.Y*v.Y + v.Z*v.Z)
let normalize v =
    let m = magnitude v
    { X = v.X/m; Y = v.Y/m; Z = v.Z/m }

printfn "v1: %A" v1
printfn "v2: %A" v2
printfn "v1 + v2: %A" (add v1 v2)
printfn "v1 * 2: %A" (scale 2.0 v1)
printfn "v1 . v2: %f" (dot v1 v2)
printfn "|v1|: %f" (magnitude v1)
printfn "normalize v1: %A" (normalize v1)

// ข้อดีของ struct:
// - Faster allocation (stack vs heap)
// - No GC pressure
// - Cache-friendly (value semantics)

// ข้อเสีย:
// - Copying overhead สำหรับ large structs
// - No inheritance
// - Mutable ต้องระวัง
```

---

## 7.9 Record Equality และ Comparison

```fsharp
// Records มี structural equality โดยอัตโนมัติ

type Point = { X: int; Y: int }

let p1 = { X = 1; Y = 2 }
let p2 = { X = 1; Y = 2 }
let p3 = { X = 3; Y = 4 }

// Equality
printfn "p1 = p2: %b" (p1 = p2)   // true (structural equality)
printfn "p1 = p3: %b" (p1 = p3)   // false

// Comparison (ถ้า fields comparable)
printfn "p1 < p3: %b" (p1 < p3)   // true (compares field by field)
printfn "p3 > p1: %b" (p3 > p1)   // true

// Sorting list of records
let points = [{ X = 3; Y = 1 }; { X = 1; Y = 4 }; { X = 2; Y = 2 }]
let sorted = List.sort points
printfn "Sorted: %A" sorted

// Custom equality สำหรับ records กับ mutable/function fields
[<CustomEquality; CustomComparison>]
type Temperature = {
    Celsius: float
} with
    override this.Equals(other) =
        match other with
        | :? Temperature as t -> abs(this.Celsius - t.Celsius) < 0.001
        | _ -> false
    
    override this.GetHashCode() = hash (round this.Celsius)
    
    interface System.IComparable with
        member this.CompareTo(other) =
            match other with
            | :? Temperature as t -> this.Celsius.CompareTo(t.Celsius)
            | _ -> 0

let temp1 = { Celsius = 25.0 }
let temp2 = { Celsius = 25.0005 }
let temp3 = { Celsius = 30.0 }

printfn "temp1 = temp2: %b" (temp1 = temp2)  // true (within 0.001)
printfn "temp1 < temp3: %b" (temp1 < temp3)   // true
```

---

## 7.10 Nested Records

```fsharp
// Records สามารถ contain records อื่น

type Address = {
    Street: string
    City: string
    PostCode: string
    Country: string
}

type ContactInfo = {
    Phone: string option
    Email: string
    Website: string option
}

type Company = {
    Name: string
    Address: Address
    Contact: ContactInfo
    Founded: int
    Employees: int
}

let anthropic = {
    Name = "Anthropic"
    Address = {
        Street = "548 Market Street"
        City = "San Francisco"
        PostCode = "94104"
        Country = "USA"
    }
    Contact = {
        Phone = None
        Email = "info@anthropic.com"
        Website = Some "https://anthropic.com"
    }
    Founded = 2021
    Employees = 500
}

// Access nested fields
printfn "Company: %s" anthropic.Name
printfn "City: %s" anthropic.Address.City
printfn "Email: %s" anthropic.Contact.Email

match anthropic.Contact.Website with
| Some url -> printfn "Website: %s" url
| None -> printfn "No website"

// Update nested field
let withNewPhone = { anthropic with
    Contact = { anthropic.Contact with Phone = Some "+1-555-0100" } }

match withNewPhone.Contact.Phone with
| Some phone -> printfn "Phone: %s" phone
| None -> ()

// Nested records ใน function parameters
let formatAddress ({ Street = s; City = c; PostCode = p; Country = co }: Address) =
    sprintf "%s, %s %s, %s" s c p co

printfn "%s" (formatAddress anthropic.Address)
```

---

## 7.11 Records กับ Functions (Methods)

```fsharp
// Records สามารถมี member functions ได้

type Rectangle = {
    Width: float
    Height: float
} with
    member this.Area = this.Width * this.Height
    member this.Perimeter = 2.0 * (this.Width + this.Height)
    member this.IsSquare = this.Width = this.Height
    member this.Scale(factor: float) = 
        { this with Width = this.Width * factor; Height = this.Height * factor }
    
    override this.ToString() =
        sprintf "Rectangle(%.2f x %.2f)" this.Width this.Height

let rect = { Width = 5.0; Height = 3.0 }
printfn "Rect: %O" rect
printfn "Area: %f" rect.Area
printfn "Perimeter: %f" rect.Perimeter
printfn "IsSquare: %b" rect.IsSquare

let scaled = rect.Scale(2.0)
printfn "Scaled: %O" scaled

// Compare ด้วย area
let rects = [
    { Width = 4.0; Height = 5.0 }
    { Width = 2.0; Height = 10.0 }
    { Width = 3.0; Height = 3.0 }
]

let sortedByArea = rects |> List.sortBy (fun r -> r.Area)
sortedByArea |> List.iter (fun r -> printfn "  %O = %.1f" r r.Area)
```

---

## 7.12 Records ในงาน Domain Modeling

### 7.12.1 E-Commerce Domain

```fsharp
// domain_model.fsx - E-commerce domain

open System

type Money = { Amount: decimal; Currency: string }

type Category = Electronics | Clothing | Food | Books | Sports

type Product = {
    Id: int
    Sku: string
    Name: string
    Description: string
    Price: Money
    Category: Category
    StockCount: int
    CreatedAt: DateTime
}

type Address = {
    Line1: string
    Line2: string option
    City: string
    Province: string
    PostCode: string
    Country: string
}

type Customer = {
    Id: int
    Email: string
    Name: string
    ShippingAddress: Address
    BillingAddress: Address option   // Same as shipping if None
}

type OrderItem = {
    Product: Product
    Quantity: int
    UnitPrice: Money
}

type OrderStatus =
    | Draft
    | Pending
    | Confirmed
    | Processing
    | Shipped
    | Delivered
    | Cancelled

type Order = {
    Id: int
    Customer: Customer
    Items: OrderItem list
    Status: OrderStatus
    CreatedAt: DateTime
    UpdatedAt: DateTime
    Notes: string option
}

// Helper functions
let totalAmount (items: OrderItem list) =
    items 
    |> List.sumBy (fun item -> item.UnitPrice.Amount * decimal item.Quantity)

let createOrder id customer items =
    let now = DateTime.Now
    { Id = id
      Customer = customer
      Items = items
      Status = Draft
      CreatedAt = now
      UpdatedAt = now
      Notes = None }

// Sample data
let sampleProduct = {
    Id = 1
    Sku = "WIDGET-001"
    Name = "Super Widget"
    Description = "A fantastic widget for all your needs"
    Price = { Amount = 299.00M; Currency = "THB" }
    Category = Electronics
    StockCount = 50
    CreatedAt = DateTime(2024, 1, 1)
}

let sampleCustomer = {
    Id = 1
    Email = "customer@example.com"
    Name = "John Doe"
    ShippingAddress = {
        Line1 = "123 Main St"
        Line2 = Some "Apt 4B"
        City = "Bangkok"
        Province = "Bangkok"
        PostCode = "10100"
        Country = "Thailand"
    }
    BillingAddress = None  // Same as shipping
}

let sampleOrderItem = {
    Product = sampleProduct
    Quantity = 2
    UnitPrice = sampleProduct.Price
}

let sampleOrder = createOrder 1 sampleCustomer [sampleOrderItem]

// Display
printfn "Order #%d" sampleOrder.Id
printfn "Customer: %s (%s)" sampleOrder.Customer.Name sampleOrder.Customer.Email
printfn "Status: %A" sampleOrder.Status
printfn "Items: %d" sampleOrder.Items.Length
printfn "Total: %.2M %s" (totalAmount sampleOrder.Items) "THB"
```

---

## 7.13 Record Serialization

```fsharp
// Basic record to string/JSON-like serialization

type Person = {
    Name: string
    Age: int
    Email: string option
}

// Custom serialization
let serializePerson (p: Person) =
    let emailStr =
        match p.Email with
        | Some e -> sprintf "\"%s\"" e
        | None -> "null"
    sprintf """{"name": "%s", "age": %d, "email": %s}""" p.Name p.Age emailStr

let deserializePerson (json: string) : Person option =
    // This is very simplified - in real code use Newtonsoft.Json or System.Text.Json
    None  // placeholder

let people = [
    { Name = "Alice"; Age = 30; Email = Some "alice@example.com" }
    { Name = "Bob"; Age = 25; Email = None }
]

printfn "Serialized people:"
people |> List.iter (fun p -> printfn "  %s" (serializePerson p))

// Using System.Text.Json (real serialization)
open System.Text.Json

let serializeToJson<'T> (value: 'T) =
    let options = JsonSerializerOptions(WriteIndented = true)
    JsonSerializer.Serialize(value, options)

// Note: F# records serialize well with System.Text.Json
// but may need custom converters for DUs and Option types
```

---

## 7.14 ตัวอย่างโปรแกรมสมบูรณ์

### 7.14.1 Library Management System

```fsharp
// library.fsx - ระบบจัดการห้องสมุด

open System

type ISBN = ISBN of string
type Author = { FirstName: string; LastName: string }
type Genre = Fiction | NonFiction | Science | History | Biography | Technology

type Book = {
    ISBN: ISBN
    Title: string
    Authors: Author list
    Genre: Genre
    PublishedYear: int
    TotalCopies: int
    AvailableCopies: int
}

type MemberId = MemberId of int

type Member = {
    Id: MemberId
    Name: string
    Email: string
    JoinedDate: DateTime
    BorrowedBooks: ISBN list
}

type LoanRecord = {
    BookISBN: ISBN
    MemberId: MemberId
    BorrowDate: DateTime
    DueDate: DateTime
    ReturnDate: DateTime option
}

// Helper functions
let formatAuthor (a: Author) = $"{a.FirstName} {a.LastName}"

let formatAuthors authors =
    authors |> List.map formatAuthor |> String.concat ", "

let isAvailable book = book.AvailableCopies > 0

let daysOverdue (loan: LoanRecord) =
    match loan.ReturnDate with
    | None ->
        let overdue = (DateTime.Now - loan.DueDate).Days
        if overdue > 0 then Some overdue else None
    | Some _ -> None

// Sample library
let books = [
    { ISBN = ISBN "978-0-7432-7356-5"
      Title = "The Great Gatsby"
      Authors = [{ FirstName = "F. Scott"; LastName = "Fitzgerald" }]
      Genre = Fiction
      PublishedYear = 1925
      TotalCopies = 5
      AvailableCopies = 3 }
    
    { ISBN = ISBN "978-0-06-112008-4"
      Title = "To Kill a Mockingbird"
      Authors = [{ FirstName = "Harper"; LastName = "Lee" }]
      Genre = Fiction
      PublishedYear = 1960
      TotalCopies = 4
      AvailableCopies = 4 }
    
    { ISBN = ISBN "978-0-13-468599-1"
      Title = "The Pragmatic Programmer"
      Authors = [
          { FirstName = "Andrew"; LastName = "Hunt" }
          { FirstName = "David"; LastName = "Thomas" }
      ]
      Genre = Technology
      PublishedYear = 2019
      TotalCopies = 3
      AvailableCopies = 1 }
]

// Display catalog
printfn "=== Library Catalog ==="
printfn "%-40s %-30s %-12s %s %s" "Title" "Authors" "Genre" "Year" "Available"
printfn "%s" (String.replicate 100 "-")

books |> List.iter (fun book ->
    let (ISBN isbn) = book.ISBN
    let available = if isAvailable book then $"{book.AvailableCopies}/{book.TotalCopies}" else "NONE"
    printfn "%-40s %-30s %-12A %4d %s"
        book.Title
        (formatAuthors book.Authors)
        book.Genre
        book.PublishedYear
        available)

// Find available books by genre
let availableByGenre genre =
    books
    |> List.filter (fun b -> b.Genre = genre && isAvailable b)

printfn "\n=== Available Technology Books ==="
availableByGenre Technology |> List.iter (fun b ->
    printfn "  %s by %s" b.Title (formatAuthors b.Authors))
```

---

## สรุป Part 7

ในบทนี้เราได้เรียนรู้:
- ✅ Tuples: (a, b), (a, b, c) - grouping หลาย values
- ✅ Tuple decomposition
- ✅ fst, snd functions
- ✅ Tuples ใน pattern matching
- ✅ เมื่อไหร่ใช้ tuple vs record
- ✅ Record type definitions
- ✅ Record creation และ field access
- ✅ Record update syntax ({ r with field = value })
- ✅ Recursive records
- ✅ Anonymous records {| ... |}
- ✅ Struct records [<Struct>]
- ✅ Record equality และ comparison
- ✅ Nested records
- ✅ Records กับ member functions

**ใน Part 8** เราจะเรียนรู้ Discriminated Unions ซึ่งเป็นหนึ่งใน features ที่ทรงพลังที่สุดของ F#!
