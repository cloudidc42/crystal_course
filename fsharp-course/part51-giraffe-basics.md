# Part 51 - Giraffe Framework พื้นฐาน

## บทนำ

Giraffe เป็น F# web framework ที่สร้างบน ASP.NET Core โดยใช้ประโยชน์จาก functional programming paradigm ของ F# เต็มที่ แทนที่จะใช้ MVC pattern แบบ C# Giraffe ใช้ HttpHandler ที่เป็น function-based approach ทำให้โค้ดสะอาดและ composable

---

## 1. What is Giraffe?

Giraffe คือ lightweight functional web framework สำหรับ F# ที่รันบน ASP.NET Core มีคุณสมบัติหลักดังนี้:

- **HttpHandler**: type alias สำหรับ `HttpContext -> Task<HttpContext option>` 
- **Composability**: สามารถ compose handlers ได้ง่ายด้วย `>=>` operator
- **ASP.NET Core Integration**: ใช้ middleware, DI, และ hosting ของ ASP.NET Core ได้ทั้งหมด
- **Performance**: High performance เพราะสร้างบน ASP.NET Core Kestrel

### เปรียบเทียบกับ ASP.NET Core MVC

```
ASP.NET Core MVC (C#):                    Giraffe (F#):
Controller class                    →     HttpHandler function
Action methods                      →     Handler composition
Routing attributes                  →     route/routef functions
Model binding                       →     bindJson/bindForm functions
IActionResult                       →     HttpHandler (text, json, html)
```

---

## 2. Creating ASP.NET Core Project with Giraffe

### ขั้นตอนการสร้างโปรเจค

```bash
# สร้าง directory
mkdir MyGiraffeApp
cd MyGiraffeApp

# สร้าง F# project
dotnet new console -lang F# -n MyGiraffeApp

# เพิ่ม packages ที่จำเป็น
dotnet add package Giraffe
dotnet add package Microsoft.AspNetCore.App
```

### .fsproj File

```xml
<!-- MyGiraffeApp.fsproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <RootNamespace>MyGiraffeApp</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <Compile Include="Models.fs" />
    <Compile Include="Handlers.fs" />
    <Compile Include="Router.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>

  <ItemGroup>
    <PackageReference Include="Giraffe" Version="7.0.0" />
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="8.0.0" />
  </ItemGroup>

</Project>
```

### Program.fs พื้นฐาน

```fsharp
// Program.fs
module Program

open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Hosting
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Giraffe

// HttpHandler หลักของแอป
let webApp : HttpHandler =
    choose [
        GET >=> route "/" >=> text "Hello, Giraffe!"
        GET >=> route "/ping" >=> text "pong"
        setStatusCode 404 >=> text "Not Found"
    ]

// Configure DI services
let configureServices (services: IServiceCollection) =
    services.AddGiraffe() |> ignore

// Configure ASP.NET Core pipeline
let configureApp (app: IApplicationBuilder) =
    app.UseGiraffe webApp

[<EntryPoint>]
let main _ =
    Host.CreateDefaultBuilder()
        .ConfigureWebHostDefaults(fun webHost ->
            webHost
                .Configure(configureApp)
                .ConfigureServices(configureServices)
            |> ignore)
        .Build()
        .Run()
    0
```

---

## 3. HttpHandler Type

HttpHandler เป็น type หลักของ Giraffe:

```fsharp
// HttpHandler definition
type HttpHandler = HttpFunc -> HttpContext -> HttpFuncResult

// โดยที่:
// HttpFunc    = HttpContext -> HttpFuncResult
// HttpFuncResult = Task<HttpContext option>
```

### ตัวอย่าง HttpHandler ง่ายๆ

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// Handler ที่ return text
let helloHandler : HttpHandler =
    fun (next: HttpFunc) (ctx: HttpContext) ->
        task {
            ctx.Response.ContentType <- "text/plain"
            do! ctx.Response.WriteAsync("Hello World!")
            return! next ctx
        }

// ใช้ fun keyword แบบสั้น
let greetHandler name : HttpHandler =
    fun next ctx ->
        task {
            return! text $"Hello, {name}!" next ctx
        }

// Handler ที่ตรวจสอบ condition
let authCheckHandler : HttpHandler =
    fun next ctx ->
        task {
            let isAuthenticated = ctx.User.Identity.IsAuthenticated
            if isAuthenticated then
                return! next ctx
            else
                return! (setStatusCode 401 >=> text "Unauthorized") next ctx
        }
```

### การ Compose Handlers ด้วย >=>

```fsharp
// >=> คือ Kleisli composition operator
// (>=>) : HttpHandler -> HttpHandler -> HttpHandler

let setJsonHeader : HttpHandler =
    setHttpHeader "Content-Type" "application/json"

let corsHeader : HttpHandler =
    setHttpHeader "Access-Control-Allow-Origin" "*"

// Compose หลาย handlers เข้าด้วยกัน
let combinedHandler : HttpHandler =
    setJsonHeader >=> corsHeader >=> text "{\"status\": \"ok\"}"

// ใน route
let webApp : HttpHandler =
    GET >=> route "/api/status" >=> combinedHandler
```

---

## 4. text, json, html Handlers

Giraffe มี built-in handlers สำหรับ response types ที่ใช้บ่อย:

### text Handler

```fsharp
open Giraffe

// ส่ง plain text response
let textExample : HttpHandler =
    text "Hello, World!"

// ส่ง text พร้อม status code
let textWithStatus : HttpHandler =
    setStatusCode 201 >=> text "Created successfully"

// Dynamic text
let dynamicText (name: string) : HttpHandler =
    text $"Welcome, {name}!"
```

### json Handler

```fsharp
open Giraffe

// Model สำหรับ serialize
type Person = {
    Id: int
    Name: string
    Age: int
    Email: string
}

// ส่ง JSON response
let personHandler : HttpHandler =
    let person = { Id = 1; Name = "สมชาย"; Age = 30; Email = "somchai@example.com" }
    json person

// ส่ง list เป็น JSON
let peopleHandler : HttpHandler =
    let people = [
        { Id = 1; Name = "สมชาย"; Age = 30; Email = "somchai@example.com" }
        { Id = 2; Name = "สมหญิง"; Age = 25; Email = "somying@example.com" }
    ]
    json people

// JSON พร้อม status code
let createdPersonHandler : HttpHandler =
    let person = { Id = 3; Name = "ใหม่"; Age = 22; Email = "new@example.com" }
    setStatusCode 201 >=> json person
```

### html Handler

```fsharp
open Giraffe

// ส่ง HTML response
let htmlPageHandler : HttpHandler =
    html """
    <!DOCTYPE html>
    <html>
    <head><title>Giraffe App</title></head>
    <body>
        <h1>Hello from Giraffe!</h1>
        <p>This is an HTML response</p>
    </body>
    </html>
    """

// HTML ด้วย GiraffeViewEngine (strongly typed HTML)
open Giraffe.ViewEngine

let indexView =
    html [] [
        head [] [
            title [] [ str "My Giraffe App" ]
            link [ rel "stylesheet"; href "/css/app.css" ]
        ]
        body [] [
            div [ _class "container" ] [
                h1 [] [ str "Welcome to Giraffe!" ]
                p [] [ str "Built with F# and ASP.NET Core" ]
                ul [] [
                    li [] [ a [ _href "/api/users" ] [ str "Users API" ] ]
                    li [] [ a [ _href "/api/products" ] [ str "Products API" ] ]
                ]
            ]
        ]
    ]

let htmlViewHandler : HttpHandler =
    htmlView indexView
```

### redirectTo Handler

```fsharp
// Redirect
let redirectHandler : HttpHandler =
    redirectTo false "/new-location"

// Permanent redirect (301)
let permanentRedirectHandler : HttpHandler =
    redirectTo true "/permanent-new-location"
```

---

## 5. Route Matching

### route Function

```fsharp
open Giraffe

let webApp : HttpHandler =
    choose [
        // Exact match
        route "/" >=> text "Home Page"
        route "/about" >=> text "About Page"
        route "/api/health" >=> json {| status = "healthy"; timestamp = System.DateTime.UtcNow |}
        
        // 404 fallback
        setStatusCode 404 >=> text "Page not found"
    ]
```

### routef - Format String Matching

```fsharp
open Giraffe

// Route พร้อม parameter
let getUser (id: int) : HttpHandler =
    fun next ctx ->
        task {
            return! json {| id = id; name = "User " + string id |} next ctx
        }

let getUserByName (name: string) : HttpHandler =
    json {| name = name; greeting = $"Hello, {name}!" |}

// Mixed parameters
let getOrderItem (orderId: int) (itemId: int) : HttpHandler =
    json {| orderId = orderId; itemId = itemId |}

let webApp : HttpHandler =
    choose [
        // Integer parameter
        routef "/api/users/%i" getUser
        
        // String parameter
        routef "/api/greet/%s" getUserByName
        
        // Multiple parameters
        routef "/api/orders/%i/items/%i" getOrderItem
        
        setStatusCode 404 >=> text "Not found"
    ]
```

### choose Function

```fsharp
open Giraffe

let webApp : HttpHandler =
    choose [
        // GET routes
        GET >=> choose [
            route "/api/users" >=> getUsersHandler
            routef "/api/users/%i" getUser
            route "/api/products" >=> getProductsHandler
        ]
        
        // POST routes
        POST >=> choose [
            route "/api/users" >=> createUserHandler
            route "/api/products" >=> createProductHandler
        ]
        
        // PUT routes
        PUT >=> choose [
            routef "/api/users/%i" updateUserHandler
        ]
        
        // DELETE routes
        DELETE >=> choose [
            routef "/api/users/%i" deleteUserHandler
        ]
        
        // Fallback
        setStatusCode 404 >=> text "Not Found"
    ]
```

---

## 6. GET, POST, PUT, DELETE Handlers

### GET Handler

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// In-memory data store (ตัวอย่าง)
let mutable users = [
    { Id = 1; Name = "สมชาย"; Age = 30; Email = "somchai@example.com" }
    { Id = 2; Name = "สมหญิง"; Age = 25; Email = "somying@example.com" }
]

// GET all users
let getUsersHandler : HttpHandler =
    fun next ctx ->
        task {
            return! json users next ctx
        }

// GET user by ID
let getUserHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match users |> List.tryFind (fun u -> u.Id = id) with
            | Some user -> return! json user next ctx
            | None -> return! (setStatusCode 404 >=> json {| error = "User not found" |}) next ctx
        }
```

### POST Handler

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open System.Text.Json

// DTO สำหรับ create
type CreateUserDto = {
    Name: string
    Age: int
    Email: string
}

// POST - Create user
let createUserHandler : HttpHandler =
    fun next ctx ->
        task {
            // อ่าน request body เป็น JSON
            let! dto = ctx.BindJsonAsync<CreateUserDto>()
            
            // สร้าง new user
            let newId = users |> List.map (fun u -> u.Id) |> List.max |> (+) 1
            let newUser = {
                Id = newId
                Name = dto.Name
                Age = dto.Age
                Email = dto.Email
            }
            
            // เพิ่มเข้า list
            users <- users @ [newUser]
            
            // Return 201 Created พร้อม location header
            ctx.Response.Headers.Add("Location", $"/api/users/{newUser.Id}")
            return! (setStatusCode 201 >=> json newUser) next ctx
        }
```

### PUT Handler

```fsharp
// DTO สำหรับ update
type UpdateUserDto = {
    Name: string option
    Age: int option
    Email: string option
}

// PUT - Update user
let updateUserHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match users |> List.tryFindIndex (fun u -> u.Id = id) with
            | None ->
                return! (setStatusCode 404 >=> json {| error = "User not found" |}) next ctx
            | Some idx ->
                let! dto = ctx.BindJsonAsync<UpdateUserDto>()
                let existing = users.[idx]
                let updated = {
                    existing with
                        Name = dto.Name |> Option.defaultValue existing.Name
                        Age = dto.Age |> Option.defaultValue existing.Age
                        Email = dto.Email |> Option.defaultValue existing.Email
                }
                users <- users |> List.mapi (fun i u -> if i = idx then updated else u)
                return! json updated next ctx
        }
```

### DELETE Handler

```fsharp
// DELETE - Remove user
let deleteUserHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match users |> List.tryFind (fun u -> u.Id = id) with
            | None ->
                return! (setStatusCode 404 >=> json {| error = "User not found" |}) next ctx
            | Some _ ->
                users <- users |> List.filter (fun u -> u.Id <> id)
                return! setStatusCode 204 next ctx
        }
```

---

## 7. Reading Query Parameters

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// อ่าน query string ทีละ parameter
let searchUsersHandler : HttpHandler =
    fun next ctx ->
        task {
            // อ่านค่า query parameters
            let nameFilter = ctx.TryGetQueryStringValue "name"
            let ageFilter = ctx.TryGetQueryStringValue "age" |> Option.map int
            let pageStr = ctx.TryGetQueryStringValue "page" |> Option.defaultValue "1"
            let pageSizeStr = ctx.TryGetQueryStringValue "pageSize" |> Option.defaultValue "10"
            
            let page = int pageStr
            let pageSize = int pageSizeStr
            
            // Filter users
            let filtered =
                users
                |> List.filter (fun u ->
                    match nameFilter with
                    | Some name -> u.Name.Contains(name)
                    | None -> true)
                |> List.filter (fun u ->
                    match ageFilter with
                    | Some age -> u.Age = age
                    | None -> true)
            
            // Paginate
            let total = filtered |> List.length
            let paged =
                filtered
                |> List.skip ((page - 1) * pageSize)
                |> List.truncate pageSize
            
            let result = {|
                data = paged
                page = page
                pageSize = pageSize
                total = total
                totalPages = (total + pageSize - 1) / pageSize
            |}
            
            return! json result next ctx
        }

// BindQueryString - bind ทั้ง object จาก query string
type UserSearchQuery = {
    Name: string option
    MinAge: int option
    MaxAge: int option
    SortBy: string option
    Page: int
    PageSize: int
}

let searchWithBindHandler : HttpHandler =
    fun next ctx ->
        task {
            // BindQueryString จะ map query params ไปยัง type
            let query = ctx.BindQueryString<UserSearchQuery>()
            
            let page = max 1 query.Page
            let pageSize = min 100 (max 1 query.PageSize)
            
            let result = {|
                query = query
                page = page
                pageSize = pageSize
            |}
            
            return! json result next ctx
        }
```

---

## 8. Reading Request Body

```fsharp
open Giraffe
open System.IO
open System.Text
open System.Text.Json

// อ่าน body เป็น string
let readBodyHandler : HttpHandler =
    fun next ctx ->
        task {
            ctx.Request.EnableBuffering()
            use reader = new StreamReader(ctx.Request.Body, Encoding.UTF8, leaveOpen = true)
            let! body = reader.ReadToEndAsync()
            ctx.Request.Body.Position <- 0L
            
            return! text $"Received body: {body}" next ctx
        }

// BindJsonAsync - แปลง JSON body เป็น type
type CreateProductDto = {
    Name: string
    Price: decimal
    Category: string
    InStock: bool
}

let createProductHandler : HttpHandler =
    fun next ctx ->
        task {
            try
                let! dto = ctx.BindJsonAsync<CreateProductDto>()
                
                // Validation
                if System.String.IsNullOrWhiteSpace(dto.Name) then
                    return! (setStatusCode 400 >=> json {| error = "Name is required" |}) next ctx
                elif dto.Price <= 0m then
                    return! (setStatusCode 400 >=> json {| error = "Price must be positive" |}) next ctx
                else
                    let product = {|
                        id = System.Guid.NewGuid()
                        name = dto.Name
                        price = dto.Price
                        category = dto.Category
                        inStock = dto.InStock
                        createdAt = System.DateTime.UtcNow
                    |}
                    return! (setStatusCode 201 >=> json product) next ctx
            with ex ->
                return! (setStatusCode 400 >=> json {| error = "Invalid JSON body" |}) next ctx
        }

// BindFormAsync - อ่านข้อมูลจาก form
let handleFormHandler : HttpHandler =
    fun next ctx ->
        task {
            let! form = ctx.Request.ReadFormAsync()
            
            let name = form["name"].ToString()
            let email = form["email"].ToString()
            
            return! json {| name = name; email = email |} next ctx
        }
```

---

## 9. JSON Serialization with System.Text.Json

```fsharp
open System.Text.Json
open System.Text.Json.Serialization
open Giraffe

// Custom JsonSerializerOptions
let jsonOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts.WriteIndented <- true
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    // เพิ่ม converter สำหรับ F# types
    opts.Converters.Add(JsonFSharpConverter())
    opts

// Configure Giraffe ให้ใช้ custom options
let configureServices (services: IServiceCollection) =
    services
        .AddGiraffe()
        .AddSingleton<Json.ISerializer>(
            SystemTextJson.Serializer(jsonOptions))
    |> ignore

// Type ที่ใช้ attribute
type ProductResponse = {
    [<JsonPropertyName("product_id")>]
    ProductId: int
    
    [<JsonPropertyName("product_name")>]
    ProductName: string
    
    [<JsonPropertyName("unit_price")>]
    UnitPrice: decimal
    
    [<JsonPropertyName("in_stock")>]
    InStock: bool
    
    [<JsonIgnore>]
    InternalNotes: string
}

// เพิ่ม JsonFSharp converter เพื่อรองรับ DU และ Option
open System.Text.Json.Serialization

type Status = 
    | Active
    | Inactive
    | Pending of string

type UserRecord = {
    Id: int
    Name: string
    Status: Status
    Email: string option
}

// ตั้งค่า JsonFSharpConverter
let fsharpJsonOptions =
    let opts = JsonSerializerOptions()
    opts.Converters.Add(JsonFSharpConverter(
        JsonUnionEncoding.FSharpLuLike
    ))
    opts

// serialize/deserialize
let serializationExample () =
    let user = {
        Id = 1
        Name = "สมชาย"
        Status = Active
        Email = Some "somchai@example.com"
    }
    
    let json = JsonSerializer.Serialize(user, fsharpJsonOptions)
    printfn "Serialized: %s" json
    
    let deserialized = JsonSerializer.Deserialize<UserRecord>(json, fsharpJsonOptions)
    printfn "Deserialized: %A" deserialized
```

---

## 10. Error Handling

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open System

// Custom Error type
type ApiError = {
    Code: string
    Message: string
    Details: string option
}

// Error response helpers
let badRequest msg =
    setStatusCode 400 >=> json { Code = "BAD_REQUEST"; Message = msg; Details = None }

let notFound msg =
    setStatusCode 404 >=> json { Code = "NOT_FOUND"; Message = msg; Details = None }

let internalError (ex: Exception) =
    setStatusCode 500 >=> json { 
        Code = "INTERNAL_ERROR"
        Message = "An unexpected error occurred"
        Details = Some ex.Message 
    }

let unauthorized () =
    setStatusCode 401 >=> json { Code = "UNAUTHORIZED"; Message = "Authentication required"; Details = None }

let forbidden () =
    setStatusCode 403 >=> json { Code = "FORBIDDEN"; Message = "Access denied"; Details = None }

// try-catch ใน handler
let safeHandler (handler: HttpHandler) : HttpHandler =
    fun next ctx ->
        task {
            try
                return! handler next ctx
            with
            | :? ArgumentException as ex ->
                return! (setStatusCode 400 >=> json {| error = ex.Message |}) next ctx
            | :? UnauthorizedAccessException ->
                return! (setStatusCode 401 >=> json {| error = "Unauthorized" |}) next ctx
            | ex ->
                // Log error
                let logger = ctx.GetService<Microsoft.Extensions.Logging.ILogger>()
                logger.LogError(ex, "Unhandled exception")
                return! (setStatusCode 500 >=> json {| error = "Internal server error" |}) next ctx
        }

// Global error handler
let errorHandler (ex: Exception) (logger: Microsoft.Extensions.Logging.ILogger) =
    logger.LogError(ex, "An unhandled exception has occurred")
    clearResponse >=>
    setStatusCode 500 >=>
    json {| error = "Internal Server Error"; message = ex.Message |}

// Configure error handler
let configureApp (app: IApplicationBuilder) =
    let env = app.ApplicationServices.GetService<IWebHostEnvironment>()
    
    if env.IsDevelopment() then
        app.UseDeveloperExceptionPage() |> ignore
    else
        app.UseGiraffeErrorHandler(errorHandler) |> ignore
    
    app.UseGiraffe webApp
```

---

## 11. Status Codes

```fsharp
open Giraffe

// Common status code helpers
let ok = setStatusCode 200
let created = setStatusCode 201
let accepted = setStatusCode 202
let noContent = setStatusCode 204
let badRequest' = setStatusCode 400
let unauthorized' = setStatusCode 401
let forbidden' = setStatusCode 403
let notFound' = setStatusCode 404
let conflict = setStatusCode 409
let unprocessableEntity = setStatusCode 422
let tooManyRequests = setStatusCode 429
let internalServerError = setStatusCode 500
let serviceUnavailable = setStatusCode 503

// ตัวอย่างการใช้งาน
let createResourceHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<CreateUserDto>()
            
            // Validation
            if System.String.IsNullOrEmpty(dto.Name) then
                return! (badRequest' >=> json {| errors = ["Name is required"] |}) next ctx
            else
                // สร้าง resource
                let resource = { Id = 1; Name = dto.Name; Age = dto.Age; Email = dto.Email }
                
                // Set Location header
                ctx.Response.Headers["Location"] <- $"/api/resources/{resource.Id}"
                
                // Return 201 Created
                return! (created >=> json resource) next ctx
        }

// Conditional responses
let conditionalHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match findById id with
            | None -> 
                return! (notFound' >=> json {| message = $"Resource {id} not found" |}) next ctx
            | Some resource ->
                // Check ETag
                let etag = $"\"{resource.GetHashCode()}\""
                let requestEtag = ctx.Request.Headers["If-None-Match"].ToString()
                
                if requestEtag = etag then
                    return! setStatusCode 304 next ctx
                else
                    ctx.Response.Headers["ETag"] <- etag
                    return! json resource next ctx
        }
```

---

## 12. Simple CRUD API Example

ตัวอย่าง CRUD API สมบูรณ์สำหรับ Todo List:

### Models.fs

```fsharp
// Models.fs
module Models

open System
open System.Text.Json.Serialization

type TodoStatus = 
    | Pending
    | InProgress  
    | Completed
    | Cancelled

type Todo = {
    Id: Guid
    Title: string
    Description: string option
    Status: TodoStatus
    CreatedAt: DateTime
    UpdatedAt: DateTime
    DueDate: DateTime option
}

type CreateTodoDto = {
    Title: string
    Description: string option
    DueDate: DateTime option
}

type UpdateTodoDto = {
    Title: string option
    Description: string option
    Status: TodoStatus option
    DueDate: DateTime option
}

type PagedResult<'T> = {
    Data: 'T list
    Page: int
    PageSize: int
    Total: int
    TotalPages: int
}
```

### Repository.fs

```fsharp
// Repository.fs
module Repository

open System
open Models

// In-memory storage (production ควรใช้ database)
let mutable private todos: Todo list = [
    {
        Id = Guid.Parse("a1b2c3d4-e5f6-7890-abcd-ef1234567890")
        Title = "เรียน F#"
        Description = Some "เรียน functional programming"
        Status = InProgress
        CreatedAt = DateTime.UtcNow.AddDays(-5)
        UpdatedAt = DateTime.UtcNow.AddDays(-1)
        DueDate = Some (DateTime.UtcNow.AddDays(30))
    }
    {
        Id = Guid.NewGuid()
        Title = "สร้าง web app"
        Description = Some "ใช้ Giraffe framework"
        Status = Pending
        CreatedAt = DateTime.UtcNow.AddDays(-2)
        UpdatedAt = DateTime.UtcNow.AddDays(-2)
        DueDate = None
    }
]

let getAll () = todos

let getById id = todos |> List.tryFind (fun t -> t.Id = id)

let create (dto: CreateTodoDto) =
    let now = DateTime.UtcNow
    let todo = {
        Id = Guid.NewGuid()
        Title = dto.Title
        Description = dto.Description
        Status = Pending
        CreatedAt = now
        UpdatedAt = now
        DueDate = dto.DueDate
    }
    todos <- todos @ [todo]
    todo

let update (id: Guid) (dto: UpdateTodoDto) =
    match todos |> List.tryFindIndex (fun t -> t.Id = id) with
    | None -> None
    | Some idx ->
        let existing = todos.[idx]
        let updated = {
            existing with
                Title = dto.Title |> Option.defaultValue existing.Title
                Description = 
                    match dto.Description with
                    | Some d -> Some d
                    | None -> existing.Description
                Status = dto.Status |> Option.defaultValue existing.Status
                DueDate = 
                    match dto.DueDate with
                    | Some d -> Some d
                    | None -> existing.DueDate
                UpdatedAt = DateTime.UtcNow
        }
        todos <- todos |> List.mapi (fun i t -> if i = idx then updated else t)
        Some updated

let delete id =
    let existed = todos |> List.exists (fun t -> t.Id = id)
    if existed then
        todos <- todos |> List.filter (fun t -> t.Id <> id)
        true
    else
        false

let search (query: string option) (status: TodoStatus option) (page: int) (pageSize: int) =
    let filtered =
        todos
        |> List.filter (fun t ->
            match query with
            | Some q -> t.Title.Contains(q, StringComparison.OrdinalIgnoreCase)
            | None -> true)
        |> List.filter (fun t ->
            match status with
            | Some s -> t.Status = s
            | None -> true)
        |> List.sortByDescending (fun t -> t.CreatedAt)
    
    let total = filtered |> List.length
    let data = 
        filtered
        |> List.skip ((page - 1) * pageSize)
        |> List.truncate pageSize
    
    {
        Data = data
        Page = page
        PageSize = pageSize
        Total = total
        TotalPages = (total + pageSize - 1) / pageSize
    }
```

### Handlers.fs

```fsharp
// Handlers.fs
module Handlers

open System
open Giraffe
open Microsoft.AspNetCore.Http
open Models
open Repository

// Validation helper
let validateCreateDto (dto: CreateTodoDto) =
    let errors = ResizeArray<string>()
    
    if String.IsNullOrWhiteSpace(dto.Title) then
        errors.Add("Title is required")
    elif dto.Title.Length > 200 then
        errors.Add("Title must be 200 characters or less")
    
    match dto.DueDate with
    | Some d when d < DateTime.UtcNow ->
        errors.Add("Due date must be in the future")
    | _ -> ()
    
    errors |> Seq.toList

// GET /api/todos
let getTodosHandler : HttpHandler =
    fun next ctx ->
        task {
            let query = ctx.TryGetQueryStringValue "q"
            let statusStr = ctx.TryGetQueryStringValue "status"
            let page = ctx.TryGetQueryStringValue "page" |> Option.map int |> Option.defaultValue 1
            let pageSize = ctx.TryGetQueryStringValue "pageSize" |> Option.map int |> Option.defaultValue 20
            
            let status =
                match statusStr with
                | Some "pending" -> Some Pending
                | Some "inprogress" -> Some InProgress
                | Some "completed" -> Some Completed
                | Some "cancelled" -> Some Cancelled
                | _ -> None
            
            let result = search query status (max 1 page) (min 100 pageSize)
            return! json result next ctx
        }

// GET /api/todos/:id
let getTodoHandler (id: string) : HttpHandler =
    fun next ctx ->
        task {
            match Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID format" |}) next ctx
            | true, guid ->
                match getById guid with
                | None ->
                    return! (setStatusCode 404 >=> json {| error = "Todo not found" |}) next ctx
                | Some todo ->
                    return! json todo next ctx
        }

// POST /api/todos
let createTodoHandler : HttpHandler =
    fun next ctx ->
        task {
            try
                let! dto = ctx.BindJsonAsync<CreateTodoDto>()
                
                let errors = validateCreateDto dto
                if not errors.IsEmpty then
                    return! (setStatusCode 400 >=> json {| errors = errors |}) next ctx
                else
                    let todo = create dto
                    ctx.Response.Headers["Location"] <- $"/api/todos/{todo.Id}"
                    return! (setStatusCode 201 >=> json todo) next ctx
            with ex ->
                return! (setStatusCode 400 >=> json {| error = "Invalid request body" |}) next ctx
        }

// PUT /api/todos/:id
let updateTodoHandler (id: string) : HttpHandler =
    fun next ctx ->
        task {
            match Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID format" |}) next ctx
            | true, guid ->
                try
                    let! dto = ctx.BindJsonAsync<UpdateTodoDto>()
                    
                    match update guid dto with
                    | None ->
                        return! (setStatusCode 404 >=> json {| error = "Todo not found" |}) next ctx
                    | Some updated ->
                        return! json updated next ctx
                with ex ->
                    return! (setStatusCode 400 >=> json {| error = "Invalid request body" |}) next ctx
        }

// DELETE /api/todos/:id
let deleteTodoHandler (id: string) : HttpHandler =
    fun next ctx ->
        task {
            match Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID format" |}) next ctx
            | true, guid ->
                if delete guid then
                    return! setStatusCode 204 next ctx
                else
                    return! (setStatusCode 404 >=> json {| error = "Todo not found" |}) next ctx
        }

// PATCH /api/todos/:id/status
let updateStatusHandler (id: string) : HttpHandler =
    fun next ctx ->
        task {
            match Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID format" |}) next ctx
            | true, guid ->
                let! body = ctx.BindJsonAsync<{| status: string |}>()
                
                let newStatus =
                    match body.status.ToLower() with
                    | "pending" -> Some Pending
                    | "inprogress" -> Some InProgress
                    | "completed" -> Some Completed
                    | "cancelled" -> Some Cancelled
                    | _ -> None
                
                match newStatus with
                | None ->
                    return! (setStatusCode 400 >=> json {| error = "Invalid status value" |}) next ctx
                | Some status ->
                    let dto = { Title = None; Description = None; Status = Some status; DueDate = None }
                    match update guid dto with
                    | None ->
                        return! (setStatusCode 404 >=> json {| error = "Todo not found" |}) next ctx
                    | Some updated ->
                        return! json updated next ctx
        }
```

### Router.fs

```fsharp
// Router.fs
module Router

open Giraffe
open Handlers

let todoRoutes : HttpHandler =
    subRoute "/api/todos" (
        choose [
            GET >=> choose [
                route "" >=> getTodosHandler
                routef "/%s" getTodoHandler
            ]
            POST >=> route "" >=> createTodoHandler
            PUT >=> routef "/%s" updateTodoHandler
            PATCH >=> routef "/%s/status" updateStatusHandler
            DELETE >=> routef "/%s" deleteTodoHandler
        ]
    )

let webApp : HttpHandler =
    choose [
        GET >=> route "/" >=> text "Todo API - Powered by Giraffe"
        GET >=> route "/health" >=> json {| status = "healthy"; timestamp = System.DateTime.UtcNow |}
        
        todoRoutes
        
        // 404 handler
        setStatusCode 404 >=> json {| error = "Not Found"; path = "The requested resource was not found" |}
    ]
```

### Program.fs สมบูรณ์

```fsharp
// Program.fs
module Program

open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Hosting
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Microsoft.Extensions.Logging
open System.Text.Json
open System.Text.Json.Serialization
open Giraffe
open Router

// Configure JSON serialization options
let jsonOptions () =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts.WriteIndented <- true
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    opts.Converters.Add(JsonFSharpConverter(JsonUnionEncoding.FSharpLuLike))
    opts

// Error handler
let errorHandler (ex: System.Exception) (logger: ILogger) =
    logger.LogError(ex, "An unhandled exception occurred")
    clearResponse >=>
    setStatusCode 500 >=>
    json {| error = "Internal Server Error" |}

// Configure services
let configureServices (services: IServiceCollection) =
    services
        .AddGiraffe()
        .AddSingleton<Json.ISerializer>(
            SystemTextJson.Serializer(jsonOptions()))
        .AddCors(fun opts ->
            opts.AddDefaultPolicy(fun policy ->
                policy
                    .AllowAnyOrigin()
                    .AllowAnyMethod()
                    .AllowAnyHeader()
                |> ignore))
        .AddResponseCaching()
    |> ignore

// Configure application pipeline
let configureApp (app: IApplicationBuilder) =
    let env = app.ApplicationServices.GetService<IWebHostEnvironment>()
    
    if env.IsDevelopment() then
        app.UseDeveloperExceptionPage() |> ignore
    
    app
        .UseCors()
        .UseResponseCaching()
        .UseGiraffeErrorHandler(errorHandler)
        .UseGiraffe(webApp)
    |> ignore

// Configure logging
let configureLogging (logging: ILoggingBuilder) =
    logging
        .AddConsole()
        .AddDebug()
    |> ignore

[<EntryPoint>]
let main args =
    Host.CreateDefaultBuilder(args)
        .ConfigureWebHostDefaults(fun webHost ->
            webHost
                .Configure(configureApp)
                .ConfigureServices(configureServices)
                .ConfigureLogging(configureLogging)
                .UseUrls("http://localhost:5000")
            |> ignore)
        .Build()
        .Run()
    0
```

---

## 13. Testing the API

```bash
# Get all todos
curl http://localhost:5000/api/todos

# Search todos
curl "http://localhost:5000/api/todos?q=F%23&status=inprogress"

# Get specific todo
curl http://localhost:5000/api/todos/a1b2c3d4-e5f6-7890-abcd-ef1234567890

# Create todo
curl -X POST http://localhost:5000/api/todos \
  -H "Content-Type: application/json" \
  -d '{"title": "เรียน Giraffe", "description": "สร้าง REST API"}'

# Update todo
curl -X PUT http://localhost:5000/api/todos/a1b2c3d4-e5f6-7890-abcd-ef1234567890 \
  -H "Content-Type: application/json" \
  -d '{"title": "เรียน F# และ Giraffe"}'

# Update status
curl -X PATCH http://localhost:5000/api/todos/a1b2c3d4-e5f6-7890-abcd-ef1234567890/status \
  -H "Content-Type: application/json" \
  -d '{"status": "completed"}'

# Delete todo
curl -X DELETE http://localhost:5000/api/todos/a1b2c3d4-e5f6-7890-abcd-ef1234567890
```

---

## 14. Middleware Integration

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// Logging middleware
let loggingMiddleware (next: HttpHandler) : HttpHandler =
    fun (next': HttpFunc) ctx ->
        task {
            let sw = System.Diagnostics.Stopwatch.StartNew()
            let path = ctx.Request.Path
            let method = ctx.Request.Method
            
            let logger = ctx.GetService<Microsoft.Extensions.Logging.ILogger<_>>()
            logger.LogInformation($"START {method} {path}")
            
            let! result = next next' ctx
            
            sw.Stop()
            logger.LogInformation($"END {method} {path} - {sw.ElapsedMilliseconds}ms")
            
            return result
        }

// Rate limiting middleware (ตัวอย่างง่ายๆ)
let mutable private requestCounts = System.Collections.Generic.Dictionary<string, int>()

let rateLimitMiddleware (maxRequests: int) (next: HttpHandler) : HttpHandler =
    fun next' ctx ->
        task {
            let ip = ctx.Connection.RemoteIpAddress.ToString()
            
            let count = 
                match requestCounts.TryGetValue(ip) with
                | true, c -> c
                | false, _ -> 0
            
            if count >= maxRequests then
                return! (setStatusCode 429 >=> text "Too Many Requests") next' ctx
            else
                requestCounts.[ip] <- count + 1
                return! next next' ctx
        }

// Authentication middleware
let requireAuthentication (next: HttpHandler) : HttpHandler =
    fun next' ctx ->
        task {
            let token = ctx.Request.Headers["Authorization"].ToString()
            
            if System.String.IsNullOrEmpty(token) then
                return! (setStatusCode 401 >=> json {| error = "Authentication required" |}) next' ctx
            else
                // validate token...
                return! next next' ctx
        }
```

---

## สรุป

Giraffe เป็น framework ที่ทรงพลังและเหมาะกับ F# developer ที่ต้องการ:
- **Functional approach**: ทุกอย่างเป็น function และ composable
- **Type safety**: รับประกันความถูกต้องตั้งแต่ compile time
- **ASP.NET Core compatibility**: ใช้ ecosystem ของ .NET ได้ทั้งหมด
- **Performance**: ไม่มี overhead พิเศษนอกจาก ASP.NET Core เอง

Key concepts ที่ควรจำ:
1. `HttpHandler = HttpFunc -> HttpContext -> Task<HttpContext option>`
2. `>=>` operator สำหรับ compose handlers
3. `choose` สำหรับ routing
4. `routef` สำหรับ parameterized routes
5. Built-in helpers: `text`, `json`, `html`, `setStatusCode`
