# Part 55 - การจัดการ JSON (JSON Handling)

## บทนำ

JSON (JavaScript Object Notation) เป็น data format ที่ใช้กันแพร่หลายในการสื่อสารระหว่าง services ใน F# มีหลายวิธีในการจัดการ JSON แต่ละวิธีมีข้อดีข้อเสียต่างกัน บทนี้จะครอบคลุมทุกวิธีหลักๆ

---

## 1. System.Text.Json กับ F#

System.Text.Json เป็น built-in JSON library ของ .NET มีประสิทธิภาพสูงแต่ต้องการ setup เพิ่มเติมสำหรับ F# types

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Basic types ทำงานได้ทันที
type SimpleRecord = {
    Name: string
    Age: int
    Score: float
}

let basic () =
    let record = { Name = "สมชาย"; Age = 30; Score = 95.5 }
    
    // Serialize
    let json = JsonSerializer.Serialize(record)
    printfn "JSON: %s" json
    // Output: {"Name":"สมชาย","Age":30,"Score":95.5}
    
    // Deserialize
    let back = JsonSerializer.Deserialize<SimpleRecord>(json)
    printfn "Back: %A" back
```

---

## 2. JsonSerializer.Serialize / Deserialize

```fsharp
open System.Text.Json

// Serialize ประเภทต่างๆ
let serializeExamples () =
    // Primitive types
    let str = JsonSerializer.Serialize("Hello")         // "Hello"
    let num = JsonSerializer.Serialize(42)              // 42
    let f = JsonSerializer.Serialize(3.14)              // 3.14
    let b = JsonSerializer.Serialize(true)              // true
    let n = JsonSerializer.Serialize(null: obj)         // null
    
    // Collections
    let arr = JsonSerializer.Serialize([1; 2; 3])       // [1,2,3]
    let lst = JsonSerializer.Serialize(["a"; "b"])      // ["a","b"]
    let map = JsonSerializer.Serialize(Map.ofList [("key", "value")])
    
    // Anonymous records (F# anonymous types)
    let anon = JsonSerializer.Serialize({| name = "Test"; value = 42 |})
    printfn "Anonymous: %s" anon
    // Output: {"name":"Test","value":42}
    
    // Nested
    let nested = JsonSerializer.Serialize({|
        user = {| id = 1; name = "Test" |}
        data = [1; 2; 3]
        meta = {| total = 3 |}
    |})
    printfn "Nested: %s" nested

// Deserialize
let deserializeExamples () =
    // Basic
    let str = JsonSerializer.Deserialize<string>("\"Hello\"")
    let num = JsonSerializer.Deserialize<int>("42")
    
    // Record
    let json = """{"name":"Test","age":25}"""
    
    // ต้องใช้ case-sensitive property names
    // หรือตั้ง options ให้ case-insensitive
    
    // Array
    let arr = JsonSerializer.Deserialize<int[]>("[1,2,3]")
    let lst = JsonSerializer.Deserialize<int list>("[1,2,3]")
    
    // Nullable
    let nullable = JsonSerializer.Deserialize<System.Nullable<int>>("null")
    
    printfn "String: %s" str
    printfn "Array: %A" arr

// Error handling
let safeDeserialize<'T> (json: string) =
    try
        let result = JsonSerializer.Deserialize<'T>(json)
        Ok result
    with
    | :? JsonException as ex ->
        Error $"JSON parsing error: {ex.Message}"
    | ex ->
        Error $"Unexpected error: {ex.Message}"
```

---

## 3. JsonSerializerOptions

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Default options
let defaultOptions = JsonSerializerOptions.Default

// Camel case properties
let camelCaseOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts

// Snake case properties
let snakeCaseOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.SnakeCaseLower
    opts

// Readable (indented)
let readableOptions =
    let opts = JsonSerializerOptions()
    opts.WriteIndented <- true
    opts

// Case-insensitive deserialization
let caseInsensitiveOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNameCaseInsensitive <- true
    opts

// Full production options
let productionOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts.PropertyNameCaseInsensitive <- true
    opts.WriteIndented <- false
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    opts.AllowTrailingCommas <- true
    opts.ReadCommentHandling <- JsonCommentHandling.Skip
    opts.NumberHandling <- JsonNumberHandling.AllowReadingFromString
    opts.Encoder <- System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping
    opts

// ตัวอย่างการใช้
let optionsExample () =
    type Product = {
        ProductId: int
        ProductName: string
        UnitPrice: decimal
    }
    
    let product = { ProductId = 1; ProductName = "Widget"; UnitPrice = 9.99m }
    
    // Default (PascalCase)
    let json1 = JsonSerializer.Serialize(product)
    printfn "Default: %s" json1
    // {"ProductId":1,"ProductName":"Widget","UnitPrice":9.99}
    
    // CamelCase
    let json2 = JsonSerializer.Serialize(product, camelCaseOptions)
    printfn "CamelCase: %s" json2
    // {"productId":1,"productName":"Widget","unitPrice":9.99}
    
    // Snake_case
    let json3 = JsonSerializer.Serialize(product, snakeCaseOptions)
    printfn "Snake: %s" json3
    // {"product_id":1,"product_name":"Widget","unit_price":9.99}
```

---

## 4. [<JsonPropertyName>] Attribute

```fsharp
open System.Text.Json.Serialization

// กำหนดชื่อ property ใน JSON ด้วย attribute
type UserProfile = {
    [<JsonPropertyName("user_id")>]
    UserId: int
    
    [<JsonPropertyName("first_name")>]
    FirstName: string
    
    [<JsonPropertyName("last_name")>]
    LastName: string
    
    [<JsonPropertyName("email_address")>]
    Email: string
    
    // ซ่อน property นี้ใน JSON
    [<JsonIgnore>]
    PasswordHash: string
    
    // Include เฉพาะตอน serialize (ไม่อ่านค่าจาก JSON)
    [<JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingDefault)>]
    LastLoginAt: System.DateTime
}

let attributeExample () =
    let profile = {
        UserId = 1
        FirstName = "สมชาย"
        LastName = "ใจดี"
        Email = "somchai@example.com"
        PasswordHash = "hashed_password_here"
        LastLoginAt = System.DateTime.UtcNow
    }
    
    let json = System.Text.Json.JsonSerializer.Serialize(profile)
    printfn "JSON: %s" json
    // {"user_id":1,"first_name":"สมชาย","last_name":"ใจดี","email_address":"somchai@example.com"}
    // Note: PasswordHash ถูกซ่อน

// [<JsonRequired>] - property ต้องมีค่าใน JSON
type RequiredFields = {
    [<JsonRequired>]
    Name: string
    
    [<JsonRequired>]
    Email: string
    
    Description: string // Optional
}

// [<JsonExtensionData>] - เก็บ extra properties
open System.Collections.Generic

type FlexibleRecord = {
    Name: string
    
    [<JsonExtensionData>]
    ExtraProperties: Dictionary<string, System.Text.Json.JsonElement>
}

let extensionDataExample () =
    let json = """{"name": "Test", "customField1": "value1", "customField2": 42}"""
    
    let opts = System.Text.Json.JsonSerializerOptions()
    opts.PropertyNameCaseInsensitive <- true
    
    let record = System.Text.Json.JsonSerializer.Deserialize<FlexibleRecord>(json, opts)
    printfn "Name: %s" record.Name
    printfn "Extra: %A" record.ExtraProperties
```

---

## 5. Custom Converters สำหรับ Discriminated Unions

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Discriminated Union ที่ต้องการ custom converter
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

// Custom converter สำหรับ Shape
type ShapeConverter() =
    inherit JsonConverter<Shape>()
    
    override _.Read(reader, typeToConvert, options) =
        let doc = JsonDocument.ParseValue(&reader)
        let root = doc.RootElement
        let kind = root.GetProperty("kind").GetString()
        
        match kind with
        | "circle" ->
            let radius = root.GetProperty("radius").GetDouble()
            Circle radius
        | "rectangle" ->
            let width = root.GetProperty("width").GetDouble()
            let height = root.GetProperty("height").GetDouble()
            Rectangle(width, height)
        | "triangle" ->
            let base' = root.GetProperty("base").GetDouble()
            let height = root.GetProperty("height").GetDouble()
            Triangle(base', height)
        | k -> failwith $"Unknown shape kind: {k}"
    
    override _.Write(writer, value, options) =
        writer.WriteStartObject()
        match value with
        | Circle radius ->
            writer.WriteString("kind", "circle")
            writer.WriteNumber("radius", radius)
        | Rectangle(w, h) ->
            writer.WriteString("kind", "rectangle")
            writer.WriteNumber("width", w)
            writer.WriteNumber("height", h)
        | Triangle(b, h) ->
            writer.WriteString("kind", "triangle")
            writer.WriteNumber("base", b)
            writer.WriteNumber("height", h)
        writer.WriteEndObject()

// Use converter
let shapeConverterExample () =
    let opts = JsonSerializerOptions()
    opts.Converters.Add(ShapeConverter())
    
    let shapes = [Circle 5.0; Rectangle(3.0, 4.0); Triangle(6.0, 8.0)]
    
    let json = JsonSerializer.Serialize(shapes, opts)
    printfn "Shapes JSON: %s" json
    
    let back = JsonSerializer.Deserialize<Shape list>(json, opts)
    printfn "Shapes back: %A" back

// Option<T> converter
type OptionConverter<'T>() =
    inherit JsonConverter<'T option>()
    
    override _.Read(reader, typeToConvert, options) =
        if reader.TokenType = JsonTokenType.Null then
            None
        else
            let value = JsonSerializer.Deserialize<'T>(&reader, options)
            Some value
    
    override _.Write(writer, value, options) =
        match value with
        | None -> writer.WriteNullValue()
        | Some v -> JsonSerializer.Serialize(writer, v, options)

// Generic OptionConverter factory
type OptionConverterFactory() =
    inherit JsonConverterFactory()
    
    override _.CanConvert(typeToConvert) =
        typeToConvert.IsGenericType &&
        typeToConvert.GetGenericTypeDefinition() = typedefof<_ option>
    
    override _.CreateConverter(typeToConvert, options) =
        let innerType = typeToConvert.GetGenericArguments().[0]
        let converterType = typedefof<OptionConverter<_>>.MakeGenericType(innerType)
        System.Activator.CreateInstance(converterType) :?> JsonConverter
```

---

## 6. Thoth.Json Library

Thoth.Json ให้ encoder/decoder pattern ที่ elegant สำหรับ F#

```bash
dotnet add package Thoth.Json.Net
```

```fsharp
open Thoth.Json.Net

// Basic types
let encodeExamples () =
    // Encode primitives
    let strJson = Encode.string "Hello"
    let intJson = Encode.int 42
    let floatJson = Encode.float 3.14
    let boolJson = Encode.bool true
    let nullJson = Encode.nil
    
    // Encode collections
    let arrayJson = Encode.array [|Encode.int 1; Encode.int 2; Encode.int 3|]
    let listJson = Encode.list [Encode.string "a"; Encode.string "b"]
    
    // Encode object
    let objJson = Encode.object [
        "name", Encode.string "สมชาย"
        "age", Encode.int 30
        "active", Encode.bool true
    ]
    
    printfn "%s" (Encode.toString 2 objJson)

// Decode
let decodeExamples () =
    // Decode primitives
    let strResult = Decode.fromString Decode.string "\"Hello\""
    let intResult = Decode.fromString Decode.int "42"
    
    match intResult with
    | Ok n -> printfn "Got int: %d" n
    | Error e -> printfn "Error: %s" e
    
    // Decode object
    let personDecoder =
        Decode.object (fun get ->
            {|
                name = get.Required.Field "name" Decode.string
                age = get.Required.Field "age" Decode.int
                email = get.Optional.Field "email" Decode.string
            |})
    
    let json = """{"name": "สมชาย", "age": 30}"""
    match Decode.fromString personDecoder json with
    | Ok person -> printfn "Person: %A" person
    | Error e -> printfn "Decode error: %s" e
```

---

## 7. Encode and Decode

```fsharp
open Thoth.Json.Net

// Full encode/decode example

type Address = {
    Street: string
    City: string
    ZipCode: string
    Country: string
}

type Employee = {
    Id: int
    FullName: string
    Email: string
    Department: string
    Salary: decimal
    Address: Address option
    Skills: string list
    StartDate: System.DateTime
}

// Encoder
module EmployeeEncoder =
    let encodeAddress (addr: Address) =
        Encode.object [
            "street", Encode.string addr.Street
            "city", Encode.string addr.City
            "zipCode", Encode.string addr.ZipCode
            "country", Encode.string addr.Country
        ]
    
    let encode (emp: Employee) =
        Encode.object [
            "id", Encode.int emp.Id
            "fullName", Encode.string emp.FullName
            "email", Encode.string emp.Email
            "department", Encode.string emp.Department
            "salary", Encode.decimal emp.Salary
            "address", emp.Address |> Option.map encodeAddress |> Option.defaultValue Encode.nil
            "skills", emp.Skills |> List.map Encode.string |> Encode.list
            "startDate", Encode.datetime emp.StartDate
        ]
    
    let toJson (emp: Employee) =
        encode emp |> Encode.toString 2

// Decoder
module EmployeeDecoder =
    let decodeAddress =
        Decode.object (fun get ->
            {
                Street = get.Required.Field "street" Decode.string
                City = get.Required.Field "city" Decode.string
                ZipCode = get.Required.Field "zipCode" Decode.string
                Country = get.Required.Field "country" Decode.string
            })
    
    let decode =
        Decode.object (fun get ->
            {
                Id = get.Required.Field "id" Decode.int
                FullName = get.Required.Field "fullName" Decode.string
                Email = get.Required.Field "email" Decode.string
                Department = get.Required.Field "department" Decode.string
                Salary = get.Required.Field "salary" Decode.decimal
                Address = get.Optional.Field "address" decodeAddress
                Skills = get.Required.Field "skills" (Decode.list Decode.string)
                StartDate = get.Required.Field "startDate" Decode.datetime
            })
    
    let fromJson json =
        Decode.fromString decode json

// ใช้งาน
let thothExample () =
    let emp = {
        Id = 1
        FullName = "สมชาย ใจดี"
        Email = "somchai@company.com"
        Department = "Engineering"
        Salary = 85000m
        Address = Some {
            Street = "123 ถนนสุขุมวิท"
            City = "กรุงเทพ"
            ZipCode = "10110"
            Country = "Thailand"
        }
        Skills = ["F#"; "C#"; ".NET"; "SQL"]
        StartDate = System.DateTime(2020, 1, 15)
    }
    
    let json = EmployeeEncoder.toJson emp
    printfn "JSON:\n%s" json
    
    match EmployeeDecoder.fromJson json with
    | Ok decoded ->
        printfn "\nDecoded successfully:"
        printfn "Name: %s" decoded.FullName
        printfn "Skills: %A" decoded.Skills
    | Error e ->
        printfn "Error: %s" e
```

---

## 8. Manual Codec Writing

```fsharp
open Thoth.Json.Net

// Complex DU codec
type PaymentMethod =
    | Cash
    | CreditCard of number: string * expiry: string
    | BankTransfer of bankCode: string * accountNumber: string
    | DigitalWallet of provider: string * walletId: string

let encodePaymentMethod (method: PaymentMethod) =
    match method with
    | Cash ->
        Encode.object ["type", Encode.string "cash"]
    | CreditCard(number, expiry) ->
        Encode.object [
            "type", Encode.string "creditCard"
            "number", Encode.string number
            "expiry", Encode.string expiry
        ]
    | BankTransfer(bankCode, accountNumber) ->
        Encode.object [
            "type", Encode.string "bankTransfer"
            "bankCode", Encode.string bankCode
            "accountNumber", Encode.string accountNumber
        ]
    | DigitalWallet(provider, walletId) ->
        Encode.object [
            "type", Encode.string "digitalWallet"
            "provider", Encode.string provider
            "walletId", Encode.string walletId
        ]

let decodePaymentMethod : Decoder<PaymentMethod> =
    Decode.field "type" Decode.string
    |> Decode.andThen (fun kind ->
        match kind with
        | "cash" -> Decode.succeed Cash
        | "creditCard" ->
            Decode.object (fun get ->
                CreditCard(
                    get.Required.Field "number" Decode.string,
                    get.Required.Field "expiry" Decode.string
                ))
        | "bankTransfer" ->
            Decode.object (fun get ->
                BankTransfer(
                    get.Required.Field "bankCode" Decode.string,
                    get.Required.Field "accountNumber" Decode.string
                ))
        | "digitalWallet" ->
            Decode.object (fun get ->
                DigitalWallet(
                    get.Required.Field "provider" Decode.string,
                    get.Required.Field "walletId" Decode.string
                ))
        | unknown ->
            Decode.fail $"Unknown payment method type: {unknown}")

// Transaction codec
type Transaction = {
    Id: System.Guid
    Amount: decimal
    Currency: string
    PaymentMethod: PaymentMethod
    Status: string
    CreatedAt: System.DateTime
}

let encodeTransaction (t: Transaction) =
    Encode.object [
        "id", Encode.string (t.Id.ToString())
        "amount", Encode.decimal t.Amount
        "currency", Encode.string t.Currency
        "paymentMethod", encodePaymentMethod t.PaymentMethod
        "status", Encode.string t.Status
        "createdAt", Encode.datetime t.CreatedAt
    ]

let decodeTransaction : Decoder<Transaction> =
    Decode.object (fun get ->
        {
            Id = get.Required.Field "id" (Decode.string |> Decode.map System.Guid.Parse)
            Amount = get.Required.Field "amount" Decode.decimal
            Currency = get.Required.Field "currency" Decode.string
            PaymentMethod = get.Required.Field "paymentMethod" decodePaymentMethod
            Status = get.Required.Field "status" Decode.string
            CreatedAt = get.Required.Field "createdAt" Decode.datetime
        })
```

---

## 9. Automatic Codecs with Thoth.Json.Auto

```fsharp
open Thoth.Json.Net

// Auto encoding/decoding สำหรับ simple types
type SimpleProduct = {
    Id: int
    Name: string
    Price: float
    InStock: bool
}

let autoExample () =
    let product = { Id = 1; Name = "Widget"; Price = 9.99; InStock = true }
    
    // Auto encode
    let json = Encode.Auto.toString 2 product
    printfn "Auto encoded:\n%s" json
    
    // Auto decode
    match Decode.Auto.fromString<SimpleProduct> json with
    | Ok p -> printfn "Auto decoded: %A" p
    | Error e -> printfn "Error: %s" e

// Auto with extra coders
let autoWithExtra () =
    // Extra coders สำหรับ types พิเศษ
    let extraCoders = Extra.empty
    
    type OrderWithGuid = {
        OrderId: System.Guid
        Products: string list
        Total: decimal
    }
    
    let order = {
        OrderId = System.Guid.NewGuid()
        Products = ["Widget"; "Gadget"]
        Total = 49.99m
    }
    
    let json = Encode.Auto.toString<OrderWithGuid> 2 order
    printfn "%s" json

// Auto decode ด้วย camelCase
let autoDecodeFromCamelCase () =
    let json = """
    {
        "productId": 1,
        "productName": "Test",
        "unitPrice": 9.99
    }
    """
    
    type ProductDto = {
        ProductId: int
        ProductName: string
        UnitPrice: float
    }
    
    // ใช้ CamelCase option
    match Decode.Auto.fromString<ProductDto>(json, isCamelCase = true) with
    | Ok dto -> printfn "Got: %A" dto
    | Error e -> printfn "Error: %s" e
```

---

## 10. Newtonsoft.Json กับ F#

```bash
dotnet add package Newtonsoft.Json
dotnet add package Newtonsoft.Json.FSharp  # F# type support
```

```fsharp
open Newtonsoft.Json
open Newtonsoft.Json.Serialization

// Basic usage
let newtonsoftBasic () =
    let obj = {| name = "สมชาย"; age = 30; active = true |}
    
    // Serialize
    let json = JsonConvert.SerializeObject(obj)
    printfn "JSON: %s" json
    
    // With settings
    let settings = JsonSerializerSettings()
    settings.Formatting <- Formatting.Indented
    settings.NullValueHandling <- NullValueHandling.Ignore
    settings.ContractResolver <- CamelCasePropertyNamesContractResolver()
    
    let prettyJson = JsonConvert.SerializeObject(obj, settings)
    printfn "Pretty:\n%s" prettyJson
    
    // Deserialize
    let back = JsonConvert.DeserializeObject<{| name: string; age: int |}>(json)
    printfn "Back: %A" back

// Newtonsoft เหมาะกับ complex scenarios
let newtonsoftAdvanced () =
    // Custom converter สำหรับ DU
    type Status = Active | Inactive | Suspended of reason: string
    
    // ใช้ StringEnumConverter สำหรับ simple DU
    let settings = JsonSerializerSettings()
    settings.Converters.Add(Newtonsoft.Json.Converters.StringEnumConverter())
    
    // JsonProperty attribute
    open Newtonsoft.Json
    
    type ApiResponse<'T> = {
        [<JsonProperty("success")>]
        Success: bool
        
        [<JsonProperty("data")>]
        Data: 'T option
        
        [<JsonProperty("error")>]
        Error: string option
        
        [<JsonProperty("timestamp")>]
        Timestamp: System.DateTime
    }
    
    let response = {
        Success = true
        Data = Some {| id = 1; name = "Test" |}
        Error = None
        Timestamp = System.DateTime.UtcNow
    }
    
    let json = JsonConvert.SerializeObject(response, Formatting.Indented)
    printfn "%s" json
```

---

## 11. Handling null in JSON

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Null handling ใน System.Text.Json
type UserWithNulls = {
    Id: int
    Name: string
    Nickname: string  // อาจเป็น null ได้ใน JSON
    Bio: string option  // F# way ของ optional
}

let nullHandlingExample () =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    
    // JSON มี null
    let json = """{"id":1,"name":"Test","nickname":null,"bio":null}"""
    
    // ถ้าใช้ string ปกติ null จะเป็น null (ไม่ใช่ None)
    // ถ้าใช้ string option null จะเป็น None (ต้องใช้ JsonFSharpConverter)
    
    // Default behavior: null -> null
    let defaultOpts = JsonSerializerOptions()
    defaultOpts.PropertyNameCaseInsensitive <- true
    
    try
        let user = JsonSerializer.Deserialize<UserWithNulls>(json, defaultOpts)
        printfn "Name: %s, Nickname: %A, Bio: %A" user.Name user.Nickname user.Bio
    with ex ->
        printfn "Error: %s" ex.Message

// Ignore null ใน output
let ignoreNullExample () =
    let opts = JsonSerializerOptions()
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    
    let user = { Id = 1; Name = "Test"; Nickname = null; Bio = None }
    let json = JsonSerializer.Serialize(user, opts)
    printfn "Without nulls: %s" json

// Always include null
let alwaysNullExample () =
    let opts = JsonSerializerOptions()
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.Never
    
    let user = { Id = 1; Name = "Test"; Nickname = null; Bio = None }
    let json = JsonSerializer.Serialize(user, opts)
    printfn "With nulls: %s" json
```

---

## 12. Option Type ใน JSON

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// F# Option type ต้องการ custom converter
// ใช้ FSharp.SystemTextJson package
// dotnet add package FSharp.SystemTextJson

open System.Text.Json.Serialization

let fsharpJsonOptions () =
    let opts = JsonSerializerOptions()
    opts.Converters.Add(JsonFSharpConverter())
    opts

// กับ JsonFSharpConverter
// None -> null ใน JSON
// Some "value" -> "value" ใน JSON (ไม่ใช่ {"Some": "value"})

type Profile = {
    UserId: int
    DisplayName: string
    Bio: string option          // null หรือ string
    Website: string option      // null หรือ string
    Avatar: string option
}

let optionExample () =
    let opts = fsharpJsonOptions()
    
    let profile = {
        UserId = 1
        DisplayName = "สมชาย"
        Bio = Some "นักพัฒนา F#"
        Website = None
        Avatar = Some "https://example.com/avatar.jpg"
    }
    
    let json = JsonSerializer.Serialize(profile, opts)
    printfn "JSON: %s" json
    // {"userId":1,"displayName":"สมชาย","bio":"นักพัฒนา F#","website":null,"avatar":"..."}
    
    // Deserialize
    let json2 = """{"userId":2,"displayName":"Test","bio":null,"website":"https://test.com"}"""
    let profile2 = JsonSerializer.Deserialize<Profile>(json2, opts)
    printfn "Profile: %A" profile2
    // Bio = None, Website = Some "https://test.com"

// Custom Option converter ถ้าไม่ต้องการ FSharp.SystemTextJson
type OptionJsonConverter<'T>() =
    inherit JsonConverter<'T option>()
    
    override _.Read(reader, _, options) =
        if reader.TokenType = JsonTokenType.Null then
            reader.Read() |> ignore
            None
        else
            Some (JsonSerializer.Deserialize<'T>(&reader, options))
    
    override _.Write(writer, value, options) =
        match value with
        | None -> writer.WriteNullValue()
        | Some v -> JsonSerializer.Serialize(writer, v, options)
```

---

## 13. DU Serialization Strategies

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Strategy 1: Type discriminator
type AnimalV1 =
    | Dog of name: string * breed: string
    | Cat of name: string * indoor: bool
    | Bird of name: string * canFly: bool

// Custom converter สำหรับ strategy นี้
type AnimalConverter() =
    inherit JsonConverter<AnimalV1>()
    
    override _.Read(reader, _, options) =
        use doc = JsonDocument.ParseValue(&reader)
        let root = doc.RootElement
        let type' = root.GetProperty("type").GetString()
        match type' with
        | "dog" ->
            Dog(
                root.GetProperty("name").GetString(),
                root.GetProperty("breed").GetString()
            )
        | "cat" ->
            Cat(
                root.GetProperty("name").GetString(),
                root.GetProperty("indoor").GetBoolean()
            )
        | "bird" ->
            Bird(
                root.GetProperty("name").GetString(),
                root.GetProperty("canFly").GetBoolean()
            )
        | t -> failwith $"Unknown animal type: {t}"
    
    override _.Write(writer, value, options) =
        writer.WriteStartObject()
        match value with
        | Dog(name, breed) ->
            writer.WriteString("type", "dog")
            writer.WriteString("name", name)
            writer.WriteString("breed", breed)
        | Cat(name, indoor) ->
            writer.WriteString("type", "cat")
            writer.WriteString("name", name)
            writer.WriteBoolean("indoor", indoor)
        | Bird(name, canFly) ->
            writer.WriteString("type", "bird")
            writer.WriteString("name", name)
            writer.WriteBoolean("canFly", canFly)
        writer.WriteEndObject()

// Strategy 2: FSharp.SystemTextJson JsonUnionEncoding
// ต้องการ package FSharp.SystemTextJson

let fsharpLuLikeOpts () =
    let opts = JsonSerializerOptions()
    // FSharpLuLike: Case name เป็น field แรก
    opts.Converters.Add(JsonFSharpConverter(JsonUnionEncoding.FSharpLuLike))
    opts

let adjacentTagOpts () =
    let opts = JsonSerializerOptions()
    // AdjacentTag: {"Case": "Dog", "Fields": {...}}
    opts.Converters.Add(JsonFSharpConverter(JsonUnionEncoding.AdjacentTag))
    opts

let internalTagOpts () =
    let opts = JsonSerializerOptions()
    // InternalTag: embed tag inside object
    opts.Converters.Add(JsonFSharpConverter(JsonUnionEncoding.InternalTag))
    opts

let externalTagOpts () =
    let opts = JsonSerializerOptions()
    // ExternalTag: {"Dog": {...}}
    opts.Converters.Add(JsonFSharpConverter(JsonUnionEncoding.ExternalTag))
    opts

// Test different encodings
let testEncodings () =
    type Vehicle =
        | Car of make: string * model: string * year: int
        | Truck of capacity: float
        | Motorcycle of cc: int
    
    let car = Car("Toyota", "Camry", 2023)
    
    // FSharpLuLike: {"car": {"make": "Toyota", "model": "Camry", "year": 2023}}
    let json1 = JsonSerializer.Serialize(car, fsharpLuLikeOpts())
    printfn "FSharpLuLike: %s" json1
    
    // AdjacentTag: {"Case": "Car", "Fields": {"make": "Toyota", ...}}
    let json2 = JsonSerializer.Serialize(car, adjacentTagOpts())
    printfn "AdjacentTag: %s" json2
```

---

## 14. Real-world Serialization Patterns

```fsharp
open System.Text.Json
open System.Text.Json.Serialization

// Pattern 1: API Response wrapper
type ApiResult<'T> =
    | Success of data: 'T * meta: {| total: int option; page: int option |}
    | Failure of errors: string list * code: int

type ApiResponse<'T> = {
    Success: bool
    Data: 'T option
    Errors: string list option
    Meta: {| total: int option; page: int option |} option
    Timestamp: System.DateTime
}

let toApiResponse (result: ApiResult<'T>) =
    match result with
    | Success(data, meta) ->
        {
            Success = true
            Data = Some data
            Errors = None
            Meta = Some meta
            Timestamp = System.DateTime.UtcNow
        }
    | Failure(errors, _) ->
        {
            Success = false
            Data = None
            Errors = Some errors
            Meta = None
            Timestamp = System.DateTime.UtcNow
        }

// Pattern 2: Versioned DTO
type UserV1 = {
    Id: int
    Name: string
    Email: string
}

type UserV2 = {
    Id: int
    FirstName: string
    LastName: string
    Email: string
    Phone: string option
}

// Migration function
let migrateUserV1ToV2 (v1: UserV1) : UserV2 =
    let names = v1.Name.Split(' ')
    {
        Id = v1.Id
        FirstName = if names.Length > 0 then names.[0] else v1.Name
        LastName = if names.Length > 1 then names.[1] else ""
        Email = v1.Email
        Phone = None
    }

// Pattern 3: Polymorphic deserialization
type EventBase = {
    EventId: System.Guid
    EventType: string
    OccurredAt: System.DateTime
}

type UserCreatedEvent = {
    EventId: System.Guid
    EventType: string
    OccurredAt: System.DateTime
    UserId: int
    Email: string
}

type OrderPlacedEvent = {
    EventId: System.Guid
    EventType: string
    OccurredAt: System.DateTime
    OrderId: int
    Total: decimal
}

type DomainEvent =
    | UserCreated of UserCreatedEvent
    | OrderPlaced of OrderPlacedEvent
    | Unknown of EventBase

let deserializeDomainEvent (json: string) =
    let opts = JsonSerializerOptions()
    opts.PropertyNameCaseInsensitive <- true
    
    // อ่าน base event ก่อนเพื่อดู type
    let baseEvent = JsonSerializer.Deserialize<EventBase>(json, opts)
    
    match baseEvent.EventType with
    | "UserCreated" ->
        let e = JsonSerializer.Deserialize<UserCreatedEvent>(json, opts)
        UserCreated e
    | "OrderPlaced" ->
        let e = JsonSerializer.Deserialize<OrderPlacedEvent>(json, opts)
        OrderPlaced e
    | _ ->
        Unknown baseEvent

// Pattern 4: Partial updates (JSON Merge Patch)
type UpdateFields = {
    Name: JsonElement option  // ใช้ JsonElement สำหรับ true optional
    Email: JsonElement option
    Age: JsonElement option
}

let applyPatch (current: UserV1) (patch: UpdateFields) =
    {
        current with
            Name = 
                match patch.Name with
                | Some elem when elem.ValueKind <> JsonValueKind.Null ->
                    elem.GetString()
                | Some _ -> null  // explicit null = clear
                | None -> current.Name  // missing = keep
            Email =
                match patch.Email with
                | Some elem when elem.ValueKind <> JsonValueKind.Null ->
                    elem.GetString()
                | Some _ -> null
                | None -> current.Email
    }
```

---

## สรุป

การจัดการ JSON ใน F# มีหลายทางเลือก:

1. **System.Text.Json** - Built-in, fast, ต้องการ setup สำหรับ F# types
2. **FSharp.SystemTextJson** - Adds F# type support ให้ System.Text.Json
3. **Thoth.Json** - Pure F# encoder/decoder pattern, explicit, safe
4. **Newtonsoft.Json** - Mature, feature-rich, สำหรับ complex scenarios

แนะนำ:
- **Production API**: System.Text.Json + FSharp.SystemTextJson
- **Compile-time safety**: Thoth.Json
- **Legacy projects**: Newtonsoft.Json
- **F# DU serialization**: ใช้ JsonFSharpConverter พร้อม JsonUnionEncoding ที่เหมาะสม
