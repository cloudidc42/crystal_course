# Part 29 - ตัวให้บริการประเภท (Type Providers Introduction)

## บทนำ (Introduction)

Type Providers คือ F# feature ที่สร้าง types อัตโนมัติจากแหล่งข้อมูลภายนอก เช่น CSV, JSON, XML, Database schemas Type Providers ทำให้ได้ type safety แบบ compile-time โดยไม่ต้องเขียน boilerplate code

---

## 29.1 What Are Type Providers?

```fsharp
// ============ Type Providers คืออะไร? ============

// ปัญหาที่ Type Providers แก้ไข:
// 1. ต้องเขียน model classes เองจาก JSON/XML schemas
// 2. ไม่มี type safety เมื่ออ่านข้อมูลจากไฟล์
// 3. Database schema เปลี่ยนแล้วต้อง update code ด้วยมือ

// Type Provider ทำงานอย่างไร:
// 1. คอมไพเลอร์รัน Type Provider ในเวลา compile
// 2. Type Provider อ่าน schema (จาก CSV, JSON, DB, etc.)
// 3. สร้าง .NET types อัตโนมัติ
// 4. เราใช้ types เหล่านั้นได้ใน IDE พร้อม IntelliSense

// ============ ประเภทของ Type Providers ============
(*
    1. Erasing Type Providers:
       - Types ถูก "erased" ที่ runtime
       - เปลี่ยนเป็น object/base types
       - เช่น FSharp.Data providers
    
    2. Generative Type Providers:
       - Types ถูก compile เป็น .NET assemblies จริง
       - ยังคงอยู่ที่ runtime
       - เช่น SqlProvider
*)

// ============ ติดตั้ง NuGet packages ============
(*
    # ใน .fsproj เพิ่ม:
    <PackageReference Include="FSharp.Data" Version="6.*" />
    <PackageReference Include="SQLProvider" Version="1.*" />
    <PackageReference Include="SwaggerProvider" Version="*" />
    
    # หรือรันคำสั่ง:
    dotnet add package FSharp.Data
    dotnet add package SQLProvider
*)
```

---

## 29.2 FSharp.Data.CsvProvider

```fsharp
#r "nuget: FSharp.Data"
open FSharp.Data

// ============ CsvProvider ============
// สร้าง types จาก CSV file

// ตัวอย่าง CSV (sales.csv):
// Date,Product,Quantity,Price,Region
// 2024-01-01,Widget A,100,9.99,North
// 2024-01-02,Widget B,50,19.99,South
// 2024-01-03,Widget A,75,9.99,East

// หมายเหตุ: ในการใช้งานจริง ใส่ path ไฟล์จริง
// นี่คือตัวอย่างการใช้ type inference จาก schema

(*
    // การใช้งาน CsvProvider
    type SalesData = CsvProvider<"sales.csv">
    
    // โหลดข้อมูล
    let sales = SalesData.Load("sales.csv")
    
    // เข้าถึง rows (strongly typed!)
    for row in sales.Rows do
        printfn "Date: %A, Product: %s, Qty: %d, Price: %f" 
            row.Date row.Product row.Quantity row.Price
    
    // คำนวณ
    let totalRevenue = 
        sales.Rows 
        |> Seq.sumBy (fun row -> float row.Quantity * row.Price)
    
    printfn "Total Revenue: $%.2f" totalRevenue
    
    // Filter
    let northSales = sales.Rows |> Seq.filter (fun r -> r.Region = "North")
    
    // Group by product
    let byProduct = 
        sales.Rows
        |> Seq.groupBy (fun r -> r.Product)
        |> Seq.map (fun (product, rows) ->
            (product, rows |> Seq.sumBy (fun r -> r.Quantity)))
*)

// ============ CsvProvider กับ options ============
(*
    // ระบุ separator ต่างๆ
    type TabSeparated = CsvProvider<"data.tsv", Separator = "\t">
    type SemiColonSep = CsvProvider<"data.csv", Separator = ";">
    
    // ระบุ schema เอง (ไม่ต้องมีไฟล์จริง)
    type MyData = CsvProvider<"Name,Age,Score\nAlice,30,95", Schema = "Name,Age(int),Score(float)">
    
    // รองรับ missing values
    type WithMissing = CsvProvider<"data.csv", MissingValues = "N/A,NA,null">
    
    // กำหนด types เอง
    type Typed = CsvProvider<"data.csv", Schema = "Date (date), Amount (decimal), Name (string)">
    
    // ใช้กับ URL
    type WebData = CsvProvider<"https://raw.githubusercontent.com/datasets/population/master/data/population.csv">
*)

// ============ Manual CSV parsing example ============
// สำหรับเมื่อไม่มี CsvProvider
let parseCsv (csvContent: string) =
    let lines = csvContent.Split('\n') |> Array.filter (fun s -> s.Trim() <> "")
    let headers = lines.[0].Split(',') |> Array.map (fun s -> s.Trim())
    let rows = 
        lines.[1..] 
        |> Array.map (fun line ->
            line.Split(',')
            |> Array.mapi (fun i cell -> (headers.[i], cell.Trim()))
            |> Map.ofArray)
    (headers, rows)

let csvContent = """Date,Product,Quantity,Price
2024-01-01,Widget A,100,9.99
2024-01-02,Widget B,50,19.99
2024-01-03,Widget A,75,9.99"""

let (headers, rows) = parseCsv csvContent
printfn "Headers: %A" headers
printfn "Row 0: %A" rows.[0]
printfn "Product in row 1: %s" rows.[1].["Product"]

// Calculate total manually
let totalRevenue = 
    rows 
    |> Array.sumBy (fun r -> float (int r.["Quantity"]) * float r.["Price"])
printfn "Total Revenue: $%.2f" totalRevenue
```

---

## 29.3 FSharp.Data.JsonProvider

```fsharp
// ============ JsonProvider ============
(*
    // สร้าง types จาก JSON sample
    type WeatherAPI = JsonProvider<"""
    {
        "city": "Bangkok",
        "temperature": 32.5,
        "humidity": 75,
        "conditions": "Partly Cloudy",
        "forecast": [
            {"day": "Mon", "high": 35, "low": 28},
            {"day": "Tue", "high": 33, "low": 27}
        ]
    }
    """>
    
    // ใช้งาน
    let weather = WeatherAPI.Parse(jsonString)
    printfn "City: %s" weather.City
    printfn "Temperature: %.1f" weather.Temperature
    
    for day in weather.Forecast do
        printfn "%s: High=%d Low=%d" day.Day day.High day.Low
    
    // โหลดจาก URL
    let liveWeather = WeatherAPI.Load("https://api.weather.com/...")
*)

// ============ ตัวอย่าง manual JSON parsing ============
// ใช้เมื่อไม่มี JsonProvider

open System.Text.Json

// Type สำหรับ JSON data
type WeatherForecast = {
    Day: string
    High: int
    Low: int
}

type WeatherData = {
    City: string
    Temperature: float
    Humidity: int
    Conditions: string
    Forecast: WeatherForecast list
}

let parseWeatherJson (json: string) : WeatherData =
    use doc = JsonDocument.Parse(json)
    let root = doc.RootElement
    
    let forecast = 
        root.GetProperty("forecast").EnumerateArray()
        |> Seq.map (fun d -> {
            Day = d.GetProperty("day").GetString()
            High = d.GetProperty("high").GetInt32()
            Low = d.GetProperty("low").GetInt32()
        })
        |> Seq.toList
    
    {
        City = root.GetProperty("city").GetString()
        Temperature = root.GetProperty("temperature").GetDouble()
        Humidity = root.GetProperty("humidity").GetInt32()
        Conditions = root.GetProperty("conditions").GetString()
        Forecast = forecast
    }

let weatherJson = """
{
    "city": "Bangkok",
    "temperature": 32.5,
    "humidity": 75,
    "conditions": "Partly Cloudy",
    "forecast": [
        {"day": "Mon", "high": 35, "low": 28},
        {"day": "Tue", "high": 33, "low": 27}
    ]
}"""

let weather = parseWeatherJson weatherJson
printfn "City: %s, Temp: %.1f°C, Humidity: %d%%" weather.City weather.Temperature weather.Humidity

for day in weather.Forecast do
    printfn "  %s: High=%d°C Low=%d°C" day.Day day.High day.Low

// ============ JSON Serialization ============
let serializeToJson data =
    let options = JsonSerializerOptions(WriteIndented = true)
    JsonSerializer.Serialize(data, options)

type Person = { Name: string; Age: int; Hobbies: string list }

let alice = { Name = "Alice"; Age = 30; Hobbies = ["coding"; "reading"; "hiking"] }
let json = serializeToJson alice
printfn "\nSerialized:\n%s" json

// Deserialize
let decoded = JsonSerializer.Deserialize<Person>(json)
printfn "Deserialized: %A" decoded
```

---

## 29.4 FSharp.Data.HtmlProvider

```fsharp
// ============ HtmlProvider ============
(*
    // สร้าง types จาก HTML table
    type WikiTable = HtmlProvider<"https://en.wikipedia.org/wiki/List_of_countries_by_population">
    
    let page = WikiTable.Load("https://...")
    
    // เข้าถึง tables
    for table in page.Tables do
        printfn "Table: %s" table.Name
    
    // เข้าถึง specific table
    let popTable = page.Tables.["Countries"]
    for row in popTable.Rows do
        printfn "%s: %d" row.Country row.Population
    
    // เข้าถึง lists
    for list in page.Lists do
        printfn "List: %s" list.Name
*)

// ============ Manual HTML parsing example ============
open System.Net.Http

let fetchPage (url: string) =
    async {
        use client = new HttpClient()
        return! client.GetStringAsync(url) |> Async.AwaitTask
    }

// HTML Table parsing (simplified)
let parseHtmlTable (html: string) =
    // ตัวอย่างง่ายๆ ไม่ใช่ production-ready
    let rows = 
        html.Split("<tr>") 
        |> Array.skip 1
        |> Array.map (fun row ->
            row.Split("<td>")
            |> Array.skip 1
            |> Array.map (fun cell ->
                let text = 
                    cell.Split("</td>").[0]
                    |> fun s -> System.Text.RegularExpressions.Regex.Replace(s, "<[^>]+>", "")
                text.Trim()))
    rows

// ============ Web scraping example (concept) ============
type CountryData = {
    Name: string
    Population: int64
    Area: float
    Capital: string
}

// สร้าง sample data แทนการ fetch จริง
let countries = [
    { Name = "China"; Population = 1_439_323_776L; Area = 9596960.0; Capital = "Beijing" }
    { Name = "India"; Population = 1_380_004_385L; Area = 3287263.0; Capital = "New Delhi" }
    { Name = "USA"; Population = 331_002_651L; Area = 9833517.0; Capital = "Washington DC" }
    { Name = "Indonesia"; Population = 273_523_615L; Area = 1904569.0; Capital = "Jakarta" }
    { Name = "Thailand"; Population = 69_799_978L; Area = 513120.0; Capital = "Bangkok" }
]

printfn "Top 5 countries by population:"
countries 
|> List.sortByDescending (fun c -> c.Population)
|> List.iter (fun c -> 
    printfn "  %-15s %15d" c.Name c.Population)

let densities = 
    countries 
    |> List.map (fun c -> 
        (c.Name, float c.Population / c.Area))
    |> List.sortByDescending snd

printfn "\nPopulation density (people/km²):"
densities |> List.iter (fun (name, density) -> 
    printfn "  %-15s %8.1f" name density)
```

---

## 29.5 FSharp.Data.XmlProvider

```fsharp
// ============ XmlProvider ============
(*
    // สร้าง types จาก XML
    type BookCatalog = XmlProvider<"""
    <catalog>
        <book id="b001">
            <author>Alice Smith</author>
            <title>F# Programming</title>
            <price>29.99</price>
            <year>2024</year>
        </book>
        <book id="b002">
            <author>Bob Jones</author>
            <title>Functional Design</title>
            <price>34.99</price>
            <year>2023</year>
        </book>
    </catalog>
    """>
    
    let catalog = BookCatalog.Parse(xmlString)
    
    for book in catalog.Books do
        printfn "ID: %s, Title: %s, Price: $%.2f" book.Id book.Title book.Price
    
    // นับหนังสือที่ราคา > 30
    let expensive = catalog.Books |> Array.filter (fun b -> b.Price > 30m)
    printfn "Expensive books: %d" expensive.Length
*)

// ============ Manual XML parsing ============
open System.Xml.Linq

let xmlContent = """
<catalog>
    <book id="b001">
        <author>Alice Smith</author>
        <title>F# Programming</title>
        <price>29.99</price>
        <year>2024</year>
    </book>
    <book id="b002">
        <author>Bob Jones</author>
        <title>Functional Design</title>
        <price>34.99</price>
        <year>2023</year>
    </book>
    <book id="b003">
        <author>Alice Smith</author>
        <title>Advanced F#</title>
        <price>39.99</price>
        <year>2024</year>
    </book>
</catalog>"""

type Book = {
    Id: string
    Author: string
    Title: string
    Price: decimal
    Year: int
}

let parseBooks (xml: string) : Book list =
    let doc = XDocument.Parse(xml)
    doc.Root.Elements("book")
    |> Seq.map (fun book ->
        {
            Id = book.Attribute(XName.Get("id")).Value
            Author = book.Element(XName.Get("author")).Value
            Title = book.Element(XName.Get("title")).Value
            Price = decimal book.Element(XName.Get("price")).Value
            Year = int book.Element(XName.Get("year")).Value
        })
    |> Seq.toList

let books = parseBooks xmlContent

printfn "Books:"
books |> List.iter (fun b ->
    printfn "  [%s] %s by %s ($%.2f, %d)" b.Id b.Title b.Author b.Price b.Year)

// Queries
let byAlice = books |> List.filter (fun b -> b.Author = "Alice Smith")
printfn "\nBooks by Alice Smith:"
byAlice |> List.iter (fun b -> printfn "  %s" b.Title)

let avgPrice = books |> List.averageBy (fun b -> float b.Price)
printfn "\nAverage price: $%.2f" avgPrice

// ============ Generate XML ============
let generateBookXml (books: Book list) =
    let catalog = XElement(XName.Get("catalog"))
    for book in books do
        let elem = XElement(XName.Get("book"),
            XAttribute(XName.Get("id"), book.Id),
            XElement(XName.Get("author"), book.Author),
            XElement(XName.Get("title"), book.Title),
            XElement(XName.Get("price"), book.Price),
            XElement(XName.Get("year"), book.Year))
        catalog.Add(elem)
    catalog.ToString()

let newBook = { Id = "b004"; Author = "Carol Lee"; Title = "Type Theory"; Price = 44.99m; Year = 2024 }
let updatedXml = generateBookXml (books @ [newBook])
printfn "\nUpdated XML (first 200 chars):"
printfn "%s..." (updatedXml.[..200])
```

---

## 29.6 SqlProvider

```fsharp
// ============ SqlProvider ============
// Type-safe database access

(*
    // ติดตั้ง: dotnet add package SQLProvider
    #r "nuget: SQLProvider"
    open FSharp.Data.Sql
    
    // PostgreSQL example
    [<Literal>]
    let connectionString = "Host=localhost;Database=mydb;Username=user;Password=pass"
    
    type Db = SqlDataProvider<
        Common.DatabaseProviderTypes.POSTGRESQL,
        connectionString,
        ResolutionPath = ".",
        IndividualsAmount = 1000,
        UseOptionTypes = true>
    
    // Query ด้วย LINQ-like syntax
    let ctx = Db.GetDataContext()
    
    // Select all users
    let users = 
        query {
            for u in ctx.Public.Users do
            select u
        } |> Seq.toList
    
    for user in users do
        printfn "User: %s, Email: %s" user.Name user.Email
    
    // Filtered query
    let activeUsers = 
        query {
            for u in ctx.Public.Users do
            where (u.IsActive = true)
            select u
        }
    
    // Join
    let usersWithOrders = 
        query {
            for u in ctx.Public.Users do
            join o in ctx.Public.Orders on (u.Id = o.UserId)
            select (u.Name, o.Total)
        }
    
    // Insert
    let newUser = ctx.Public.Users.Create()
    newUser.Name <- "Charlie"
    newUser.Email <- "charlie@example.com"
    newUser.IsActive <- true
    ctx.SubmitUpdates()
    
    // Update
    let user = 
        query {
            for u in ctx.Public.Users do
            where (u.Id = 1)
            exactlyOne
        }
    user.Email <- "newemail@example.com"
    ctx.SubmitUpdates()
*)

// ============ Database simulation ============
// สาธิตการ query pattern โดยไม่ต้องมี DB จริง

type User = {
    Id: int
    Name: string
    Email: string
    IsActive: bool
    DepartmentId: int
}

type Department = {
    Id: int
    Name: string
    Budget: decimal
}

type Order = {
    Id: int
    UserId: int
    Total: decimal
    Status: string
}

// Mock database
let users = [
    { Id = 1; Name = "Alice"; Email = "alice@co.com"; IsActive = true; DepartmentId = 1 }
    { Id = 2; Name = "Bob"; Email = "bob@co.com"; IsActive = false; DepartmentId = 2 }
    { Id = 3; Name = "Charlie"; Email = "charlie@co.com"; IsActive = true; DepartmentId = 1 }
    { Id = 4; Name = "Diana"; Email = "diana@co.com"; IsActive = true; DepartmentId = 3 }
]

let departments = [
    { Id = 1; Name = "Engineering"; Budget = 500000m }
    { Id = 2; Name = "Marketing"; Budget = 200000m }
    { Id = 3; Name = "Finance"; Budget = 300000m }
]

let orders = [
    { Id = 1; UserId = 1; Total = 150.0m; Status = "completed" }
    { Id = 2; UserId = 1; Total = 75.0m; Status = "pending" }
    { Id = 3; UserId = 3; Total = 200.0m; Status = "completed" }
    { Id = 4; UserId = 4; Total = 350.0m; Status = "completed" }
]

// LINQ-style queries using F# query expressions
let activeUsers = query {
    for u in users do
    where (u.IsActive = true)
    select u
} |> Seq.toList

printfn "Active users: %A" (activeUsers |> List.map (fun u -> u.Name))

// Join
let usersWithDepts = query {
    for u in users do
    join d in departments on (u.DepartmentId = d.Id)
    where (u.IsActive)
    select (u.Name, d.Name)
} |> Seq.toList

printfn "\nActive users with departments:"
usersWithDepts |> List.iter (fun (name, dept) -> printfn "  %s - %s" name dept)

// Aggregation
let orderStats = query {
    for o in orders do
    where (o.Status = "completed")
    groupBy o.UserId into g
    select (g.Key, g.Average(fun x -> x.Total), g.Count())
} |> Seq.toList

printfn "\nOrder stats per user:"
orderStats |> List.iter (fun (userId, avg, count) ->
    printfn "  User %d: %d orders, avg $%.2f" userId count avg)
```

---

## 29.7 Type Erasure vs Generative

```fsharp
// ============ Type Erasure vs Generative ============

(*
    Erasing Type Providers:
    - Types ถูก "erase" ที่ runtime
    - ใช้ obj/base types ที่ runtime
    - Fast compilation
    - Cannot be used as generic type arguments
    - ตัวอย่าง: FSharp.Data providers (CSV, JSON, XML, HTML)
    
    Generative Type Providers:
    - สร้าง .NET types จริงๆ
    - Types คงอยู่ที่ runtime
    - Slower compilation
    - Can be used as generic type arguments
    - ตัวอย่าง: SqlProvider
    
    ตัวอย่างของ Erasing:
*)

// ============ Simulate erasing behavior ============
// CsvProvider สร้าง type ที่มีชื่อเหมือน columns
// แต่จริงๆ underlying เป็น object array

type RowAccessor(data: obj[]) =
    member _.Item(index: int) = data.[index]
    member _.AsString(index: int) = string data.[index]
    member _.AsInt(index: int) = int (string data.[index])
    member _.AsFloat(index: int) = float (string data.[index])

type TypedRow(rawRow: obj[]) =
    let accessor = RowAccessor(rawRow)
    
    // These look like typed properties but are backed by obj[]
    member _.Name = accessor.AsString(0)
    member _.Age = accessor.AsInt(1)
    member _.Score = accessor.AsFloat(2)

// Simulate what CsvProvider generates
let simulatedRows = 
    [| "Alice,30,95.5"; "Bob,25,87.0"; "Charlie,35,92.3" |]
    |> Array.map (fun line ->
        line.Split(',') |> Array.map box |> TypedRow)

printfn "=== Simulated Type Provider ==="
for row in simulatedRows do
    printfn "Name: %s, Age: %d, Score: %.1f" row.Name row.Age row.Score

// ============ Benefits of each type ============
(*
    เลือก Erasing เมื่อ:
    - ต้องการ fast compilation
    - Data shape ซับซ้อนมาก
    - Data เป็น schema-less
    
    เลือก Generative เมื่อ:
    - ต้องการใช้ types เป็น generic arguments
    - ต้องการ reflection-based operations
    - ต้องการ binary compatibility
*)
```

---

## 29.8 Benefits of Type Providers

```fsharp
// ============ ประโยชน์ของ Type Providers ============

(*
    1. Type Safety ที่ Compile Time
       - ไม่มี typos ในชื่อ columns/properties
       - Type mismatches ถูกจับตั้งแต่ compile time
       - ไม่ต้องเขียน casting code
    
    2. IntelliSense / Auto-completion
       - IDE รู้ว่า CSV มี columns อะไร
       - Auto-complete ชื่อ properties
       - ลด time ในการ explore data
    
    3. ลด Boilerplate
       - ไม่ต้องเขียน model classes
       - ไม่ต้องเขียน parsing code ซ้ำๆ
       - Schema เปลี่ยน → compile error (ไม่ใช่ runtime error!)
    
    4. Live Schema
       - Type Provider อ่าน schema จริงในเวลา compile
       - เมื่อ DB schema เปลี่ยน compiler บอกทันที
    
    5. Productivity
       - ลดเวลา dev สำหรับ data access layer
       - Rapid prototyping กับ external data
*)

// ============ Comparison: With vs Without Type Provider ============

// WITHOUT type provider (manual):
module WithoutTypeProvider =
    type WeatherReading = {
        Timestamp: System.DateTime
        Temperature: float
        Humidity: int
        WindSpeed: float
        Pressure: float
    }
    
    let parseRow (line: string) =
        let parts = line.Split(',')
        {
            Timestamp = System.DateTime.Parse(parts.[0])
            Temperature = float parts.[1]
            Humidity = int parts.[2]
            WindSpeed = float parts.[3]
            Pressure = float parts.[4]
        }
    
    let loadData (filePath: string) =
        System.IO.File.ReadLines(filePath)
        |> Seq.skip 1  // skip header
        |> Seq.map parseRow
        |> Seq.toList
    
    // ถ้า schema เปลี่ยน → runtime error
    // ไม่มี IntelliSense สำหรับ column names
    // ต้องดูไฟล์ CSV เพื่อรู้ columns

// WITH type provider (commented since requires actual file):
(*
    // WITH type provider:
    type WeatherData = CsvProvider<"weather.csv">
    
    let loadData () = WeatherData.Load("weather.csv")
    
    // IntelliSense รู้ว่ามี columns อะไร
    // ถ้า schema เปลี่ยน → compile error
    // ไม่ต้องเขียน parsing code
    
    let data = loadData()
    for row in data.Rows do
        // row.Temperature, row.Humidity etc. ถูก suggest โดย IDE
        printfn "%.1f°C" row.Temperature
*)

// ============ ตัวอย่าง: Data Pipeline ============
type DataPipeline<'Source, 'Result> = {
    Source: 'Source
    Transform: 'Source -> 'Result
    Sink: 'Result -> unit
}

let runPipeline pipeline =
    let data = pipeline.Source
    let result = pipeline.Transform data
    pipeline.Sink result

// Simulated pipeline with CSV-like data
let salesData = [
    ("2024-01", "Product A", 100, 9.99)
    ("2024-01", "Product B", 50, 19.99)
    ("2024-02", "Product A", 120, 9.99)
    ("2024-02", "Product C", 30, 29.99)
]

let pipeline = {
    Source = salesData
    Transform = fun data ->
        data
        |> List.groupBy (fun (month, _, _, _) -> month)
        |> List.map (fun (month, rows) ->
            (month, rows |> List.sumBy (fun (_, _, qty, price) -> float qty * price)))
        |> List.sortBy fst
    Sink = fun results ->
        printfn "Monthly Revenue:"
        results |> List.iter (fun (month, rev) ->
            printfn "  %s: $%.2f" month rev)
}

runPipeline pipeline
```

---

## 29.9 Creating Type Providers (Concept)

```fsharp
// ============ สร้าง Custom Type Provider (Concept) ============
(*
    Type Provider ต้องใช้:
    1. ProvidedTypes library
    2. F# Quotations
    3. TypeProviderAssembly attribute
    
    dotnet add package FSharp.TypeProviders.SDK
*)

(*
    // ตัวอย่าง Custom Type Provider อย่างง่าย
    
    open ProviderImplementation.ProvidedTypes
    open Microsoft.FSharp.Core.CompilerServices
    
    [<TypeProvider>]
    type BasicProvider(config: TypeProviderConfig) as this =
        inherit TypeProviderForNamespaces(config)
        
        let ns = "MyProviders"
        let asm = System.Reflection.Assembly.GetExecutingAssembly()
        
        let createTypes () =
            let myType = ProvidedTypeDefinition(asm, ns, "Hello", Some typeof<obj>)
            
            let prop = ProvidedProperty("World", typeof<string>, isStatic = true,
                getterCode = fun _ -> <@@ "Hello, World!" @@>)
            
            myType.AddMember(prop)
            
            let method = ProvidedMethod("Greet", 
                [ProvidedParameter("name", typeof<string>)],
                typeof<string>,
                isStatic = true,
                invokeCode = fun args -> 
                    <@@ sprintf "Hello, %s!" (%%args.[0] : string) @@>)
            
            myType.AddMember(method)
            myType
        
        do this.AddNamespace(ns, [createTypes()])
    
    [<assembly: TypeProviderAssembly>]
    do ()
    
    // ใช้งาน:
    // let hello = MyProviders.Hello.World  // "Hello, World!"
    // let greeting = MyProviders.Hello.Greet("Alice")  // "Hello, Alice!"
*)

// ============ Simulated Custom Provider ============
// สาธิต concept โดยไม่ต้อง compile เป็น provider จริง

type SchemaField = { Name: string; Type: string; Nullable: bool }

type GeneratedType(schema: SchemaField list) =
    let fields = schema
    
    member _.GetValue(instance: Map<string, obj>) (fieldName: string) =
        match fields |> List.tryFind (fun f -> f.Name = fieldName) with
        | Some field ->
            match Map.tryFind fieldName instance with
            | Some value -> value
            | None when field.Nullable -> null
            | None -> failwith $"Required field '{fieldName}' is missing"
        | None -> failwith $"Unknown field '{fieldName}'"
    
    member _.ValidateInstance(instance: Map<string, obj>) =
        fields 
        |> List.choose (fun f ->
            if not f.Nullable && not (Map.containsKey f.Name instance) then
                Some $"Missing required field: {f.Name}"
            else None)

let personSchema = [
    { Name = "Name"; Type = "string"; Nullable = false }
    { Name = "Age"; Type = "int"; Nullable = false }
    { Name = "Email"; Type = "string"; Nullable = true }
]

let personType = GeneratedType(personSchema)

let instance = Map.ofList [("Name", box "Alice"); ("Age", box 30)]
let errors = personType.ValidateInstance(instance)

if errors.IsEmpty then
    printfn "Valid instance: Name=%A" (personType.GetValue instance "Name")
else
    errors |> List.iter (printfn "Error: %s")
```

---

## สรุป (Summary)

```
Type Providers ใน F#:

What they do:
- สร้าง types อัตโนมัติจาก external schemas
- Compile-time type safety
- IntelliSense support
- ลด boilerplate

Types:
- Erasing: erased at runtime (FSharp.Data)
- Generative: real .NET types (SqlProvider)

Popular Type Providers:
- FSharp.Data.CsvProvider: CSV files
- FSharp.Data.JsonProvider: JSON APIs
- FSharp.Data.XmlProvider: XML files
- FSharp.Data.HtmlProvider: HTML tables
- SQLProvider: Database access

Benefits:
- Type safety กับ external data
- Automatic schema detection
- Live schema updates
- Rapid data exploration

Getting Started:
dotnet add package FSharp.Data
dotnet add package SQLProvider
```

---

*จบ Part 29 - ตัวให้บริการประเภท (Type Providers Introduction)*
