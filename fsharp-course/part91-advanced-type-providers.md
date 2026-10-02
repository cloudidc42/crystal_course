# Part 91 - Type Providers ขั้นสูง (Advanced Type Providers)

## บทนำ

Type Providers เป็นหนึ่งในฟีเจอร์ที่ทรงพลังที่สุดของ F# ซึ่งช่วยให้เราสามารถสร้าง types จากแหล่งข้อมูลภายนอกโดยอัตโนมัติ ในบทนี้เราจะเจาะลึกถึงการใช้งาน Type Providers ขั้นสูง รวมถึงการสร้าง Custom Type Provider ของตัวเอง

---

## 1. SQLProvider - Type-safe Database Access

### การติดตั้ง

```xml
<!-- .fsproj -->
<ItemGroup>
  <PackageReference Include="SQLProvider" Version="1.3.37" />
  <PackageReference Include="System.Data.SQLite" Version="1.0.118" />
  <PackageReference Include="Npgsql" Version="7.0.0" />
  <PackageReference Include="MySqlConnector" Version="2.3.0" />
</ItemGroup>
```

### การใช้งาน SQLProvider กับ SQLite

```fsharp
// SQLite Type Provider
open FSharp.Data.Sql

// กำหนด connection string และ provider
[<Literal>]
let connectionString = 
    "Data Source=" + __SOURCE_DIRECTORY__ + "/shop.db;Version=3"

[<Literal>]
let dbVendor = Common.DatabaseProviderTypes.SQLITE

// สร้าง type จาก database schema
type Sql = SqlDataProvider<
    DatabaseVendor = dbVendor,
    ConnectionString = connectionString,
    ResolutionPath = __SOURCE_DIRECTORY__ + "/lib",
    IndividualsAmount = 1000,
    UseOptionTypes = true>

// ใช้งาน type-safe queries
let getProducts () =
    use ctx = Sql.GetDataContext()
    query {
        for product in ctx.Main.Products do
        where (product.Price > 100.0M)
        orderBy product.Name
        select {|
            Id = product.Id
            Name = product.Name
            Price = product.Price
        |}
    }
    |> Seq.toList

// Insert ข้อมูล
let addProduct name price =
    use ctx = Sql.GetDataContext()
    let product = ctx.Main.Products.Create()
    product.Name <- name
    product.Price <- price
    product.CreatedAt <- System.DateTime.Now
    ctx.SubmitUpdates()

// Update ข้อมูล
let updatePrice productId newPrice =
    use ctx = Sql.GetDataContext()
    let product = 
        query {
            for p in ctx.Main.Products do
            where (p.Id = productId)
            exactlyOne
        }
    product.Price <- newPrice
    ctx.SubmitUpdates()

// Delete ข้อมูล
let deleteProduct productId =
    use ctx = Sql.GetDataContext()
    let product = 
        query {
            for p in ctx.Main.Products do
            where (p.Id = productId)
            exactlyOne
        }
    product.Delete()
    ctx.SubmitUpdates()
```

### PostgreSQL Type Provider

```fsharp
// PostgreSQL
[<Literal>]
let pgConnectionString = 
    "Host=localhost;Port=5432;Database=shopdb;Username=postgres;Password=secret"

type PgSql = SqlDataProvider<
    DatabaseVendor = Common.DatabaseProviderTypes.POSTGRESQL,
    ConnectionString = pgConnectionString,
    UseOptionTypes = true>

// Complex queries with joins
let getOrdersWithItems () =
    use ctx = PgSql.GetDataContext()
    query {
        for order in ctx.Public.Orders do
        join orderItem in ctx.Public.OrderItems on (order.Id = orderItem.OrderId)
        join product in ctx.Public.Products on (orderItem.ProductId = product.Id)
        where (order.Status = "completed")
        select {|
            OrderId = order.Id
            CustomerName = order.CustomerName
            ProductName = product.Name
            Quantity = orderItem.Quantity
            Price = orderItem.Price
        |}
    }
    |> Seq.toList

// Aggregation queries
let getSalesSummary () =
    use ctx = PgSql.GetDataContext()
    query {
        for order in ctx.Public.Orders do
        where (order.Status = "completed")
        groupBy order.CustomerId into g
        select {|
            CustomerId = g.Key
            TotalOrders = g.Count()
            TotalAmount = g.Sum(fun o -> o.TotalAmount)
        |}
    }
    |> Seq.toList
```

---

## 2. FSharp.Data Type Providers

### JSON Type Provider

```fsharp
open FSharp.Data

// JSON Type Provider - ดึง schema จาก sample JSON
type WeatherData = JsonProvider<"""
{
    "city": "Bangkok",
    "temperature": 35.5,
    "humidity": 80,
    "forecast": [
        {
            "day": "Monday",
            "high": 36.0,
            "low": 28.5,
            "condition": "Sunny"
        }
    ]
}
""">

// ใช้งาน
let parseWeather jsonString =
    let data = WeatherData.Parse(jsonString)
    printfn "เมือง: %s" data.City
    printfn "อุณหภูมิ: %.1f°C" data.Temperature
    
    for day in data.Forecast do
        printfn "%s: สูง %.1f / ต่ำ %.1f - %s" 
            day.Day day.High day.Low day.Condition

// ดึงข้อมูลจาก URL
type GithubUser = JsonProvider<"https://api.github.com/users/dotnet">

let getGithubUser username =
    let user = GithubUser.Load($"https://api.github.com/users/{username}")
    {|
        Login = user.Login
        Name = user.Name
        PublicRepos = user.PublicRepos
        Followers = user.Followers
    |}
```

### CSV Type Provider

```fsharp
// CSV Type Provider
type SalesData = CsvProvider<"sales.csv", HasHeaders = true>

// sales.csv:
// Date,Product,Quantity,Price,Total
// 2024-01-01,Widget A,10,25.50,255.00

let analyzeSales () =
    let data = SalesData.Load("sales.csv")
    
    // คำนวณสถิติ
    let totalRevenue = 
        data.Rows 
        |> Seq.sumBy (fun row -> row.Total)
    
    let avgOrderValue = 
        data.Rows 
        |> Seq.averageBy (fun row -> row.Total)
    
    let topProducts = 
        data.Rows
        |> Seq.groupBy (fun row -> row.Product)
        |> Seq.map (fun (product, rows) -> 
            product, rows |> Seq.sumBy (fun r -> r.Total))
        |> Seq.sortByDescending snd
        |> Seq.take 5
        |> Seq.toList
    
    {|
        TotalRevenue = totalRevenue
        AverageOrderValue = avgOrderValue
        TopProducts = topProducts
    |}
```

### XML Type Provider

```fsharp
// XML Type Provider
type Config = XmlProvider<"""
<configuration>
  <database>
    <server>localhost</server>
    <port>5432</port>
    <name>mydb</name>
  </database>
  <cache>
    <host>redis://localhost:6379</host>
    <ttl>3600</ttl>
  </cache>
</configuration>
""">

let loadConfig path =
    let config = Config.Load(path)
    {|
        DbServer = config.Database.Server
        DbPort = config.Database.Port
        DbName = config.Database.Name
        CacheHost = config.Cache.Host
        CacheTtl = config.Cache.Ttl
    |}
```

---

## 3. Runtime vs Erased Type Providers

### Erased Type Provider (ที่ใช้กันทั่วไป)

Type ถูกสร้างขึ้นใน compile time แต่ถูก "ลบออก" ใน runtime โดยแทนที่ด้วย base type

```fsharp
// Erased type - มีอยู่แค่ใน compile time
// ใน runtime จะถูกแทนด้วย base class
type ErasedJson = JsonProvider<"sample.json">  // Erased

// ข้อดี:
// - ไม่สร้าง overhead ใน runtime
// - Code ที่ generate ออกมาสะอาด
// - เร็วกว่าใน compile time

// ข้อเสีย:
// - ไม่สามารถใช้ reflection ดู type ได้ใน runtime
// - ไม่สามารถ inherit หรือ implement interface ได้
```

### Generated (Non-Erased) Type Provider

```fsharp
// Generated types - มีอยู่จริงใน runtime
// ใช้สำหรับกรณีที่ต้องการ reflection หรือ serialization

// ตัวอย่าง: SqlDataProvider ใช้ generated types
// type ProductRow มีอยู่จริงใน runtime
// สามารถใช้ JsonSerializer กับมันได้

// ข้อดี:
// - ใช้ reflection ได้
// - Serialize/Deserialize ได้
// - Inherit ได้
// ข้อเสีย:
// - มี overhead มากกว่า
// - Compile time ช้ากว่า
```

---

## 4. Custom Type Provider Creation

### ติดตั้ง ProvidedTypes SDK

```xml
<PackageReference Include="FSharp.TypeProviders.SDK" Version="6.0.0" />
```

### สร้าง Simple String Type Provider

```fsharp
namespace MyTypeProviders

open System
open System.Reflection
open FSharp.Core.CompilerServices
open ProviderImplementation.ProvidedTypes

// ITypeProvider interface implementation
[<TypeProvider>]
type SimpleStringProvider(config: TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "MyTypeProviders"
    let asm = Assembly.GetExecutingAssembly()
    
    // สร้าง root type
    let myType = ProvidedTypeDefinition(asm, ns, "StringUtils", Some typeof<obj>)
    
    do
        // เพิ่ม static method
        let reverseMethod = 
            ProvidedMethod(
                methodName = "Reverse",
                parameters = [ProvidedParameter("input", typeof<string>)],
                returnType = typeof<string>,
                isStatic = true,
                invokeCode = fun args ->
                    <@@ 
                        let s = (%%args.[0]: string)
                        new string(Array.rev (s.ToCharArray()))
                    @@>)
        
        reverseMethod.AddXmlDoc("Reverses a string")
        myType.AddMember(reverseMethod)
        
        // เพิ่ม property
        let versionProp =
            ProvidedProperty(
                propertyName = "Version",
                propertyType = typeof<string>,
                isStatic = true,
                getterCode = fun _ -> <@@ "1.0.0" @@>)
        
        myType.AddMember(versionProp)
        
        this.AddNamespace(ns, [myType])
```

### Type Provider สำหรับ CSV Schema

```fsharp
namespace CsvTypeProvider

open System
open System.IO
open System.Reflection
open FSharp.Core.CompilerServices
open ProviderImplementation.ProvidedTypes
open Microsoft.FSharp.Quotations

[<TypeProvider>]
type CsvProvider(config: TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "CsvTypeProvider"
    let asm = Assembly.GetExecutingAssembly()
    
    // Helper: อ่าน headers จาก CSV file
    let getHeaders (filePath: string) =
        if File.Exists(filePath) then
            let firstLine = File.ReadLines(filePath) |> Seq.head
            firstLine.Split(',') 
            |> Array.map (fun h -> h.Trim())
        else
            [||]
    
    // Helper: ตรวจจับ column type จาก sample data
    let detectType (values: string[]) =
        let mutable isInt = true
        let mutable isFloat = true
        let mutable isDate = true
        
        for v in values do
            let v = v.Trim()
            if isInt then
                match System.Int32.TryParse(v) with
                | false, _ -> isInt <- false
                | _ -> ()
            if isFloat then
                match System.Double.TryParse(v) with
                | false, _ -> isFloat <- false
                | _ -> ()
            if isDate then
                match System.DateTime.TryParse(v) with
                | false, _ -> isDate <- false
                | _ -> ()
        
        if isInt then typeof<int>
        elif isFloat then typeof<float>
        elif isDate then typeof<DateTime>
        else typeof<string>
    
    // สร้าง row type
    let makeRowType (typeDef: ProvidedTypeDefinition) (headers: string[]) =
        let rowType = ProvidedTypeDefinition("Row", Some typeof<string[]>)
        
        headers |> Array.iteri (fun i header ->
            let idx = i // capture for closure
            let prop = 
                ProvidedProperty(
                    propertyName = header,
                    propertyType = typeof<string>,
                    getterCode = fun args ->
                        <@@ (%%args.[0]: string[]).[idx] @@>)
            prop.AddXmlDoc(sprintf "Column: %s (index %d)" header idx)
            rowType.AddMember(prop))
        
        typeDef.AddMember(rowType)
        rowType
    
    let buildTypes (typeName: string) (filePath: string) =
        let myType = ProvidedTypeDefinition(asm, ns, typeName, Some typeof<obj>)
        
        let headers = getHeaders filePath
        let rowType = makeRowType myType headers
        
        // Load method
        let loadMethod =
            ProvidedMethod(
                methodName = "Load",
                parameters = [ProvidedParameter("path", typeof<string>)],
                returnType = typedefof<seq<_>>.MakeGenericType(rowType),
                isStatic = true,
                invokeCode = fun args ->
                    <@@
                        let path = (%%args.[0]: string)
                        File.ReadLines(path)
                        |> Seq.skip 1
                        |> Seq.map (fun line -> line.Split(',') :> obj)
                    @@>)
        
        myType.AddMember(loadMethod)
        myType
    
    // IProvidedNamespace: register provider
    let provider = 
        ProvidedTypeDefinition(asm, ns, "CsvFile", Some typeof<obj>)
    
    do
        provider.DefineStaticParameters(
            [ProvidedStaticParameter("FilePath", typeof<string>)],
            fun typeName args ->
                let filePath = args.[0] :?> string
                buildTypes typeName filePath)
        
        this.AddNamespace(ns, [provider])
```

---

## 5. Advanced Type Provider: JSON Schema Provider

```fsharp
namespace JsonSchemaProvider

open System
open System.Reflection
open System.Text.Json
open FSharp.Core.CompilerServices
open ProviderImplementation.ProvidedTypes

// Parse JSON schema และสร้าง types
module SchemaParser =
    type JsonSchemaType =
        | JString
        | JNumber
        | JBoolean
        | JArray of JsonSchemaType
        | JObject of Map<string, JsonSchemaType>
        | JNullable of JsonSchemaType
    
    let rec parseSchema (element: JsonElement) : JsonSchemaType =
        match element.GetProperty("type").GetString() with
        | "string" -> JString
        | "number" | "integer" -> JNumber
        | "boolean" -> JBoolean
        | "array" ->
            let itemType = parseSchema (element.GetProperty("items"))
            JArray itemType
        | "object" ->
            let props = element.GetProperty("properties")
            let required = 
                if element.TryGetProperty("required") |> fst then
                    element.GetProperty("required").EnumerateArray()
                    |> Seq.map (fun e -> e.GetString())
                    |> Set.ofSeq
                else
                    Set.empty
            let fields =
                props.EnumerateObject()
                |> Seq.map (fun prop ->
                    let fieldType = parseSchema prop.Value
                    let finalType = 
                        if Set.contains prop.Name required then fieldType
                        else JNullable fieldType
                    prop.Name, finalType)
                |> Map.ofSeq
            JObject fields
        | t -> failwith $"Unknown type: {t}"
    
    let rec toNetType (t: JsonSchemaType) =
        match t with
        | JString -> typeof<string>
        | JNumber -> typeof<float>
        | JBoolean -> typeof<bool>
        | JArray inner -> typedefof<seq<_>>.MakeGenericType(toNetType inner)
        | JNullable inner -> 
            typedefof<option<_>>.MakeGenericType(toNetType inner)
        | JObject _ -> typeof<obj> // simplified

[<TypeProvider>]
type JsonSchemaProvider(config: TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "JsonSchemaProvider"
    let asm = Assembly.GetExecutingAssembly()
    
    let buildFromSchema (typeName: string) (schemaJson: string) =
        let schema = JsonDocument.Parse(schemaJson).RootElement
        let schemaType = SchemaParser.parseSchema schema
        
        let typeDef = ProvidedTypeDefinition(asm, ns, typeName, Some typeof<obj>)
        
        match schemaType with
        | SchemaParser.JObject fields ->
            for KeyValue(name, fieldType) in fields do
                let netType = SchemaParser.toNetType fieldType
                let prop = 
                    ProvidedProperty(
                        propertyName = name,
                        propertyType = netType,
                        getterCode = fun _ -> <@@ Unchecked.defaultof<obj> @@>)
                typeDef.AddMember(prop)
        | _ -> ()
        
        typeDef
    
    let provider = ProvidedTypeDefinition(asm, ns, "Schema", Some typeof<obj>)
    
    do
        provider.DefineStaticParameters(
            [ProvidedStaticParameter("Schema", typeof<string>)],
            fun typeName args ->
                let schema = args.[0] :?> string
                buildFromSchema typeName schema)
        
        this.AddNamespace(ns, [provider])
```

---

## 6. ITypeProvider Interface ในเชิงลึก

```fsharp
// ITypeProvider interface มี methods หลักๆ:
// - GetNamespaces(): IProvidedNamespace[]
// - GetStaticParameters(typeWithoutArguments): ParameterInfo[]
// - ApplyStaticArguments(typeWithoutArguments, typeNameWithArguments, staticArguments): Type
// - GetInvokerExpression(syntheticMethodBase, parameters): Expr
// - Dispose()

// IProvidedNamespace:
// - NamespaceName: string
// - GetNestedNamespaces(): IProvidedNamespace[]
// - GetTypes(): Type[]
// - ResolveTypeName(typeName): Type

// ProvidedTypeDefinition สำคัญ:
type ExampleProvider(config) as this =
    inherit TypeProviderForNamespaces(config)
    
    let makeType () =
        let t = ProvidedTypeDefinition("MyType", Some typeof<obj>)
        
        // Constructor
        let ctor = 
            ProvidedConstructor(
                parameters = [ProvidedParameter("value", typeof<int>)],
                invokeCode = fun args ->
                    <@@ box (%%args.[0]: int) @@>)
        t.AddMember(ctor)
        
        // Instance method
        let method1 =
            ProvidedMethod(
                methodName = "Double",
                parameters = [],
                returnType = typeof<int>,
                invokeCode = fun args ->
                    <@@ 
                        let v = %%args.[0]: obj
                        (v :?> int) * 2 
                    @@>)
        t.AddMember(method1)
        
        // Static method
        let staticMethod =
            ProvidedMethod(
                methodName = "Create",
                parameters = [ProvidedParameter("n", typeof<int>)],
                returnType = t,
                isStatic = true,
                invokeCode = fun args ->
                    <@@ box (%%args.[0]: int) @@>)
        t.AddMember(staticMethod)
        
        // Nested type
        let nestedType = ProvidedTypeDefinition("Nested", Some typeof<obj>)
        t.AddMember(nestedType)
        
        t
    
    do
        this.AddNamespace("Example", [makeType()])
```

---

## 7. Erasure กับ Non-Erased Types

```fsharp
// Erased Type Provider
// Type ที่ generate จะถูกแทนด้วย base type ใน IL

[<TypeProvider>]
type ErasedProvider(config) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "ErasedExample"
    let asm = Assembly.GetExecutingAssembly()
    
    let makeErasedType () =
        // IsErased = true (default)
        let t = ProvidedTypeDefinition(asm, ns, "ErasedType", 
                    baseType = Some typeof<obj>, 
                    isErased = true)  // erased!
        
        // Property ที่ถูก erase
        let prop =
            ProvidedProperty("Value", typeof<string>,
                getterCode = fun args ->
                    // args.[0] จะเป็น base type ใน runtime
                    <@@ (%%args.[0]: obj).ToString() @@>)
        t.AddMember(prop)
        t
    
    do this.AddNamespace(ns, [makeErasedType()])

// Generated Type Provider
[<TypeProvider>]
type GeneratedProvider(config) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "GeneratedExample"
    let asm = Assembly.GetExecutingAssembly()
    
    let makeGeneratedType () =
        // IsErased = false -> generated type จะมีอยู่จริงใน IL
        let t = ProvidedTypeDefinition(asm, ns, "GeneratedType", 
                    baseType = Some typeof<obj>, 
                    isErased = false)  // not erased!
        
        // สามารถ inherit, implement interface ได้
        // สามารถใช้ reflection ได้
        let prop =
            ProvidedProperty("Id", typeof<Guid>,
                isStatic = true,
                getterCode = fun _ -> <@@ Guid.NewGuid() @@>)
        t.AddMember(prop)
        t
    
    do this.AddNamespace(ns, [makeGeneratedType()])
```

---

## 8. Complete Custom Type Provider: RegexProvider

```fsharp
// RegexProvider - สร้าง type-safe regex matching
namespace RegexTypeProvider

open System
open System.Reflection
open System.Text.RegularExpressions
open FSharp.Core.CompilerServices
open ProviderImplementation.ProvidedTypes
open Microsoft.FSharp.Quotations

[<TypeProvider>]
type RegexProvider(config: TypeProviderConfig) as this =
    inherit TypeProviderForNamespaces(config)
    
    let ns = "RegexTypeProvider"
    let asm = Assembly.GetExecutingAssembly()
    
    // Helper: ดึง group names จาก regex pattern
    let getGroupNames (pattern: string) =
        let regex = Regex(pattern)
        regex.GetGroupNames()
        |> Array.filter (fun n -> 
            // กรอง numeric groups ออก
            match Int32.TryParse(n) with
            | true, _ -> false
            | _ -> true)
    
    // สร้าง MatchResult type
    let makeMatchType (typeName: string) (pattern: string) =
        let matchType = ProvidedTypeDefinition(asm, ns, typeName, Some typeof<Match>)
        
        // IsMatch property
        let isMatchProp =
            ProvidedProperty(
                "IsMatch", typeof<bool>,
                getterCode = fun args ->
                    <@@ (%%args.[0]: Match).Success @@>)
        isMatchProp.AddXmlDoc("Whether the regex matched")
        matchType.AddMember(isMatchProp)
        
        // Value property (full match)
        let valueProp =
            ProvidedProperty(
                "Value", typeof<string>,
                getterCode = fun args ->
                    <@@ (%%args.[0]: Match).Value @@>)
        valueProp.AddXmlDoc("The full matched string")
        matchType.AddMember(valueProp)
        
        // สร้าง property สำหรับแต่ละ named group
        let groupNames = getGroupNames pattern
        for groupName in groupNames do
            let name = groupName // capture
            let groupProp =
                ProvidedProperty(
                    name, typeof<string option>,
                    getterCode = fun args ->
                        <@@
                            let m = (%%args.[0]: Match)
                            let g = m.Groups.[name]
                            if g.Success then Some g.Value
                            else None
                        @@>)
            groupProp.AddXmlDoc(sprintf "Named group: %s" name)
            matchType.AddMember(groupProp)
        
        matchType
    
    // สร้าง main regex type
    let makeRegexType (typeName: string) (pattern: string) =
        let regexType = ProvidedTypeDefinition(asm, ns, typeName, Some typeof<obj>)
        
        let matchType = makeMatchType (typeName + "Match") pattern
        regexType.AddMember(matchType)
        
        // IsMatch static method
        let isMatchMethod =
            ProvidedMethod(
                "IsMatch",
                [ProvidedParameter("input", typeof<string>)],
                typeof<bool>,
                isStatic = true,
                invokeCode = fun args ->
                    let pat = pattern
                    <@@
                        Regex.IsMatch((%%args.[0]: string), pat)
                    @@>)
        isMatchMethod.AddXmlDoc("Tests if the input matches the pattern")
        regexType.AddMember(isMatchMethod)
        
        // Match static method
        let matchMethod =
            ProvidedMethod(
                "Match",
                [ProvidedParameter("input", typeof<string>)],
                matchType,
                isStatic = true,
                invokeCode = fun args ->
                    let pat = pattern
                    <@@
                        Regex.Match((%%args.[0]: string), pat) :> obj
                    @@>)
        matchMethod.AddXmlDoc("Matches the input against the pattern")
        regexType.AddMember(matchMethod)
        
        // Matches static method (all matches)
        let matchesMethod =
            ProvidedMethod(
                "Matches",
                [ProvidedParameter("input", typeof<string>)],
                typeof<MatchCollection>,
                isStatic = true,
                invokeCode = fun args ->
                    let pat = pattern
                    <@@
                        Regex.Matches((%%args.[0]: string), pat)
                    @@>)
        matchesMethod.AddXmlDoc("Returns all matches")
        regexType.AddMember(matchesMethod)
        
        // Pattern property
        let patternProp =
            ProvidedProperty(
                "Pattern", typeof<string>,
                isStatic = true,
                getterCode = fun _ ->
                    let pat = pattern
                    <@@ pat @@>)
        patternProp.AddXmlDoc("The regex pattern")
        regexType.AddMember(patternProp)
        
        regexType
    
    let provider = ProvidedTypeDefinition(asm, ns, "Regex", Some typeof<obj>)
    
    do
        provider.DefineStaticParameters(
            [ProvidedStaticParameter("Pattern", typeof<string>)],
            fun typeName args ->
                let pattern = args.[0] :?> string
                
                // Validate pattern at compile time!
                try
                    Regex(pattern) |> ignore
                with
                | :? ArgumentException as e ->
                    failwith $"Invalid regex pattern: {e.Message}"
                
                makeRegexType typeName pattern)
        
        this.AddNamespace(ns, [provider])

// การใช้งาน RegexProvider
// open RegexTypeProvider
// 
// type EmailRegex = Regex<@"^(?<user>[^@]+)@(?<domain>[^@]+)$">
// type PhoneRegex = Regex<@"^(?<country>\+\d{1,3})?\s*(?<number>\d{10})$">
//
// let validateEmail email =
//     let m = EmailRegex.Match(email)
//     if m.IsMatch then
//         printfn "User: %A, Domain: %A" m.User m.Domain
//     else
//         printfn "Invalid email"
```

---

## 9. Type Provider Testing

```fsharp
// การ test type providers
module TypeProviderTests

open Xunit
open FsUnit.Xunit

// Test erased type provider
[<Fact>]
let ``JsonProvider parses sample correctly`` () =
    open FSharp.Data
    
    type Sample = JsonProvider<"""{"name":"test","value":42}""">
    
    let data = Sample.Parse("""{"name":"hello","value":100}""")
    data.Name |> should equal "hello"
    data.Value |> should equal 100

// Test SQL provider (requires actual DB)
[<Fact>]
let ``SqlProvider connects and queries`` () =
    use ctx = Sql.GetDataContext()
    let products = 
        query {
            for p in ctx.Main.Products do
            select p.Name
        }
        |> Seq.toList
    
    products |> should not' (be Empty)

// Test custom type provider
[<Fact>]
let ``RegexProvider validates at compile time`` () =
    // This would fail to compile with invalid pattern:
    // type BadRegex = RegexTypeProvider.Regex<"[invalid">
    
    type EmailRegex = RegexTypeProvider.Regex<@"^(?<user>[^@]+)@(?<domain>[^@]+)$">
    
    let m = EmailRegex.Match("user@example.com")
    m.IsMatch |> should equal true
    m.User |> should equal (Some "user")
    m.Domain |> should equal (Some "example.com")
```

---

## 10. Best Practices สำหรับ Type Providers

```fsharp
// 1. ใช้ compile-time validation
// Validate ข้อมูล input ใน DefineStaticParameters
// เพื่อให้เกิด error ใน compile time แทน runtime

// 2. Caching: หลีกเลี่ยงการสร้าง type ซ้ำ
type CachedProvider(config) as this =
    inherit TypeProviderForNamespaces(config)
    
    let cache = System.Collections.Concurrent.ConcurrentDictionary<string, ProvidedTypeDefinition>()
    
    let getOrCreate key factory =
        cache.GetOrAdd(key, fun k -> factory k)
    
    // ใช้ cache ใน provider logic...

// 3. Error messages ที่ชัดเจน
let validateInput (input: string) =
    if String.IsNullOrEmpty(input) then
        failwith "Input cannot be empty"
    elif not (System.IO.File.Exists(input)) then
        failwith $"File not found: {input}\nMake sure the file exists relative to the project directory."
    else
        input

// 4. Documentation บน provided types
let addDocumentation (t: ProvidedTypeDefinition) (doc: string) =
    t.AddXmlDoc(doc)
    t

// 5. Handle optional parameters
let makeType typeName (args: obj[]) =
    let filePath = args.[0] :?> string
    let hasHeaders = if args.Length > 1 then args.[1] :?> bool else true
    let separator = if args.Length > 2 then args.[2] :?> char else ','
    // ...
    ()
```

---

## สรุป

Type Providers เป็นเครื่องมือที่ทรงพลังมากใน F# ecosystem:

1. **Erased Types**: เหมาะสำหรับการเข้าถึงข้อมูลภายนอก (JSON, CSV, XML)
2. **Generated Types**: เหมาะสำหรับกรณีที่ต้องการ runtime reflection
3. **Custom Providers**: ช่วยสร้าง domain-specific abstractions
4. **Compile-time Safety**: ตรวจสอบข้อผิดพลาดก่อน runtime
5. **ProvidedTypes SDK**: ฐานสำหรับสร้าง custom providers

Type Providers ช่วยให้ F# เป็นภาษาที่ "type-safe ถึง edge" ของ application - ตั้งแต่ database schema ไปจนถึง REST APIs และ configuration files

---

*ต่อไป: Part 92 - Advanced Functional Patterns*
