# Part 59 - Minimal API with F#

## บทนำ

ASP.NET Core Minimal API เป็น lightweight approach สำหรับสร้าง HTTP endpoints โดยไม่ต้องมี controller classes ใช้ lambda functions โดยตรง เหมาะสำหรับ microservices และ simple APIs

---

## 1. ASP.NET Core Minimal API

```fsharp
// Program.fs - Hello World with Minimal API
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    
    // Add services
    builder.Services.AddEndpointsApiExplorer() |> ignore
    builder.Services.AddSwaggerGen() |> ignore
    
    let app = builder.Build()
    
    // Configure pipeline
    if app.Environment.IsDevelopment() then
        app.UseSwagger() |> ignore
        app.UseSwaggerUI() |> ignore
    
    // Define endpoints
    app.MapGet("/", fun () -> "Hello, Minimal API!") |> ignore
    
    app.Run()
    0
```

### .fsproj สำหรับ Minimal API

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Swashbuckle.AspNetCore" Version="6.5.0" />
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="8.0.0" />
    <PackageReference Include="FluentValidation" Version="11.9.0" />
  </ItemGroup>
</Project>
```

---

## 2. app.MapGet, MapPost, etc.

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open System.Threading.Tasks

let builder = WebApplication.CreateBuilder([||])
let app = builder.Build()

// MapGet - retrieve data
app.MapGet("/", fun () -> "Hello World!") |> ignore

app.MapGet("/health", fun () ->
    {| status = "healthy"; timestamp = System.DateTime.UtcNow |}) |> ignore

app.MapGet("/users", fun () ->
    [
        {| id = 1; name = "สมชาย"; email = "somchai@example.com" |}
        {| id = 2; name = "สมหญิง"; email = "somying@example.com" |}
    ]) |> ignore

// MapPost - create data
app.MapPost("/users", fun (user: CreateUserDto) ->
    // Process and return result
    {|
        id = System.Guid.NewGuid()
        name = user.Name
        email = user.Email
        createdAt = System.DateTime.UtcNow
    |}) |> ignore

// MapPut - replace data
app.MapPut("/users/{id}", fun (id: int) (user: UpdateUserDto) ->
    {| id = id; name = user.Name; updatedAt = System.DateTime.UtcNow |}) |> ignore

// MapPatch - partial update
app.MapPatch("/users/{id}", fun (id: int) (patch: PatchUserDto) ->
    {| id = id; patched = true |}) |> ignore

// MapDelete - remove data
app.MapDelete("/users/{id}", fun (id: int) ->
    Results.NoContent()) |> ignore

// MapMethods - custom HTTP methods
app.MapMethods("/users/{id}", [| "HEAD" |], fun (id: int) (ctx: HttpContext) ->
    ctx.Response.Headers["X-User-Exists"] <- "true"
    Results.Ok()) |> ignore

app.Run()
```

### Async Handlers

```fsharp
open Microsoft.AspNetCore.Builder
open System.Threading.Tasks

let app = WebApplication.Create([||])

// Async GET
app.MapGet("/async-data", fun () ->
    task {
        let! data = fetchDataFromDatabaseAsync()
        return data
    }) |> ignore

// Async POST
app.MapPost("/process", fun (request: ProcessRequest) ->
    task {
        let! result = processAsync request
        return {| success = true; result = result |}
    }) |> ignore

// ValueTask (more efficient for hot paths)
app.MapGet("/fast", fun () ->
    ValueTask.FromResult({| message = "Fast response" |})) |> ignore
```

---

## 3. Route Parameters

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http

let app = WebApplication.Create([||])

// Basic route parameter
app.MapGet("/users/{id}", fun (id: int) ->
    {| id = id; name = $"User {id}" |}) |> ignore

// String parameter
app.MapGet("/products/{slug}", fun (slug: string) ->
    {| slug = slug; title = $"Product: {slug}" |}) |> ignore

// GUID parameter
app.MapGet("/orders/{orderId}", fun (orderId: System.Guid) ->
    {| orderId = orderId |}) |> ignore

// Multiple parameters
app.MapGet("/users/{userId}/orders/{orderId}", fun (userId: int) (orderId: int) ->
    {| userId = userId; orderId = orderId |}) |> ignore

// Optional parameter (ต้องใช้ ? หรือ nullable)
app.MapGet("/items/{id?}", fun (id: int?) ->
    match id with
    | null -> {| result = "All items" |}
    | id -> {| result = $"Item {id.Value}" |}) |> ignore

// Catch-all parameter
app.MapGet("/files/{*path}", fun (path: string) ->
    {| path = path; exists = System.IO.File.Exists(path) |}) |> ignore

// Route constraints
app.MapGet("/alpha/{name:alpha}", fun (name: string) ->
    {| name = name |}) |> ignore

app.MapGet("/number/{id:int:min(1):max(999)}", fun (id: int) ->
    {| id = id |}) |> ignore

app.MapGet("/date/{date:datetime}", fun (date: System.DateTime) ->
    {| date = date |}) |> ignore

app.Run()
```

---

## 4. Request Body Binding

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open System.Text.Json

// Automatic JSON body binding
type CreateProductRequest = {
    Name: string
    Price: decimal
    Category: string
    InStock: bool
}

let app = WebApplication.Create([||])

// Automatic binding from JSON body
app.MapPost("/products", fun (request: CreateProductRequest) ->
    {|
        id = System.Guid.NewGuid()
        name = request.Name
        price = request.Price
        category = request.Category
        inStock = request.InStock
        createdAt = System.DateTime.UtcNow
    |}) |> ignore

// Manual binding from HttpContext
app.MapPost("/manual-binding", fun (ctx: HttpContext) ->
    task {
        let! request = ctx.Request.ReadFromJsonAsync<CreateProductRequest>()
        
        return {|
            received = request
            timestamp = System.DateTime.UtcNow
        |}
    }) |> ignore

// Form binding
app.MapPost("/form-submit", fun (ctx: HttpContext) ->
    task {
        let! form = ctx.Request.ReadFormAsync()
        let name = form["name"].ToString()
        let email = form["email"].ToString()
        
        return {| name = name; email = email |}
    }) |> ignore

// Multipart form (file upload)
app.MapPost("/upload", fun (ctx: HttpContext) ->
    task {
        let! form = ctx.Request.ReadFormAsync()
        let file = form.Files["file"]
        
        if file = null then
            ctx.Response.StatusCode <- 400
            return! ctx.Response.WriteAsJsonAsync({| error = "No file uploaded" |})
        else
            use stream = System.IO.File.Create($"uploads/{file.FileName}")
            do! file.CopyToAsync(stream)
            return! ctx.Response.WriteAsJsonAsync({| filename = file.FileName; size = file.Length |})
    }) |> ignore

// Custom model binding
app.MapGet("/search", fun ([<Microsoft.AspNetCore.Http.AsParameters>] query: SearchQuery) ->
    {|
        query = query.Q
        page = query.Page
        pageSize = query.PageSize
    |}) |> ignore

app.Run()
```

---

## 5. Response Helpers

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http

let app = WebApplication.Create([||])

// Results class - typed response helpers
app.MapGet("/ok", fun () -> Results.Ok({| message = "OK" |})) |> ignore

app.MapGet("/created", fun () ->
    let item = {| id = 1; name = "New Item" |}
    Results.Created($"/items/{1}", item)) |> ignore

app.MapGet("/no-content", fun () -> Results.NoContent()) |> ignore

app.MapGet("/bad-request", fun () ->
    Results.BadRequest({| error = "Invalid input" |})) |> ignore

app.MapGet("/unauthorized", fun () ->
    Results.Unauthorized()) |> ignore

app.MapGet("/forbidden", fun () ->
    Results.Forbid()) |> ignore

app.MapGet("/not-found", fun () ->
    Results.NotFound({| error = "Resource not found" |})) |> ignore

app.MapGet("/conflict", fun () ->
    Results.Conflict({| error = "Resource already exists" |})) |> ignore

app.MapGet("/redirect", fun () ->
    Results.Redirect("/new-url")) |> ignore

app.MapGet("/permanent-redirect", fun () ->
    Results.RedirectToRoute("new-route", true)) |> ignore

// Custom status code
app.MapGet("/custom-status", fun (ctx: HttpContext) ->
    task {
        ctx.Response.StatusCode <- 418  // I'm a teapot
        return! ctx.Response.WriteAsJsonAsync({| message = "I'm a teapot" |})
    }) |> ignore

// File result
app.MapGet("/download", fun () ->
    Results.File(
        System.IO.File.ReadAllBytes("document.pdf"),
        "application/pdf",
        "document.pdf")) |> ignore

// Stream result
app.MapGet("/stream", fun () ->
    let stream = new System.IO.MemoryStream(System.Text.Encoding.UTF8.GetBytes("Stream data"))
    Results.Stream(stream, "text/plain")) |> ignore

// TypedResults (compile-time safe)
app.MapGet("/typed", fun () ->
    TypedResults.Ok({| message = "Typed OK" |})) |> ignore

app.Run()
```

---

## 6. Middleware

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.DependencyInjection

let builder = WebApplication.CreateBuilder([||])
let app = builder.Build()

// Built-in middleware
app.UseHttpsRedirection() |> ignore
app.UseStaticFiles() |> ignore
app.UseAuthentication() |> ignore
app.UseAuthorization() |> ignore

// Custom middleware with Use
app.Use(fun (ctx: HttpContext) (next: RequestDelegate) ->
    task {
        ctx.Response.Headers["X-Request-Id"] <- System.Guid.NewGuid().ToString()
        do! next.Invoke(ctx)
    } :> System.Threading.Tasks.Task) |> ignore

// Logging middleware
app.Use(fun (ctx: HttpContext) (next: RequestDelegate) ->
    task {
        let start = System.DateTime.UtcNow
        printfn $"[{start}] {ctx.Request.Method} {ctx.Request.Path}"
        do! next.Invoke(ctx)
        let duration = (System.DateTime.UtcNow - start).TotalMilliseconds
        printfn $"[{System.DateTime.UtcNow}] {ctx.Response.StatusCode} - {duration:F1}ms"
    } :> System.Threading.Tasks.Task) |> ignore

// Error handling middleware
app.UseExceptionHandler(fun errApp ->
    errApp.Run(fun ctx ->
        task {
            ctx.Response.StatusCode <- 500
            ctx.Response.ContentType <- "application/problem+json"
            let ex = ctx.Features.Get<Microsoft.AspNetCore.Diagnostics.IExceptionHandlerFeature>()
            if ex <> null then
                do! ctx.Response.WriteAsJsonAsync({|
                    type' = "https://api.example.com/errors/server-error"
                    title = "Internal Server Error"
                    status = 500
                    detail = ex.Error.Message
                |})
        } :> System.Threading.Tasks.Task)) |> ignore

// Middleware in groups (Route groups)
let apiGroup = app.MapGroup("/api")

apiGroup.Use(fun (ctx: HttpContext) (next: RequestDelegate) ->
    task {
        ctx.Response.Headers["X-API-Version"] <- "1.0"
        do! next.Invoke(ctx)
    } :> System.Threading.Tasks.Task) |> ignore

apiGroup.MapGet("/status", fun () -> {| status = "api ok" |}) |> ignore

app.Run()
```

---

## 7. Filters

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open System.Threading.Tasks

// IEndpointFilter - reusable filter
type LoggingFilter() =
    interface IEndpointFilter with
        member _.InvokeAsync(context: EndpointFilterInvocationContext, next: EndpointFilterDelegate) =
            task {
                let sw = System.Diagnostics.Stopwatch.StartNew()
                let path = context.HttpContext.Request.Path
                printfn $"[Filter] Start: {path}"
                
                let! result = next.Invoke(context)
                
                sw.Stop()
                printfn $"[Filter] End: {path} - {sw.ElapsedMilliseconds}ms"
                
                return result
            }

// Validation filter
type ValidationFilter<'T when 'T : not struct>() =
    interface IEndpointFilter with
        member _.InvokeAsync(context: EndpointFilterInvocationContext, next: EndpointFilterDelegate) =
            task {
                let model = context.Arguments |> Seq.tryPick (fun a -> 
                    match a with
                    | :? 'T as m -> Some m
                    | _ -> None)
                
                match model with
                | None -> return! next.Invoke(context)
                | Some m ->
                    let errors = validate m  // Custom validation function
                    if errors.IsEmpty then
                        return! next.Invoke(context)
                    else
                        return Results.BadRequest({| errors = errors |}) :> obj
            }

// Auth filter
type RequireAuthFilter() =
    interface IEndpointFilter with
        member _.InvokeAsync(context: EndpointFilterInvocationContext, next: EndpointFilterDelegate) =
            task {
                let ctx = context.HttpContext
                if ctx.User.Identity.IsAuthenticated then
                    return! next.Invoke(context)
                else
                    return Results.Unauthorized() :> obj
            }

// Use filters
let app = WebApplication.Create([||])

// Apply to single endpoint
app.MapPost("/users", fun (user: CreateUserDto) -> 
    {| created = true; user = user |})
    .AddEndpointFilter<LoggingFilter>()
    .AddEndpointFilter<ValidationFilter<CreateUserDto>>()
|> ignore

// Apply to group
let adminGroup = app.MapGroup("/admin")
adminGroup.AddEndpointFilter<RequireAuthFilter>() |> ignore
adminGroup.MapGet("/stats", fun () -> {| visitors = 1000 |}) |> ignore

// Inline filter
app.MapGet("/filtered", fun () -> {| data = "filtered result" |})
    .AddEndpointFilter(fun (context: EndpointFilterInvocationContext) (next: EndpointFilterDelegate) ->
        task {
            printfn "Before handler"
            let! result = next.Invoke(context)
            printfn "After handler"
            return result
        })
|> ignore

app.Run()
```

---

## 8. Validation

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.DependencyInjection
open System.ComponentModel.DataAnnotations
open System.Collections.Generic

// Data Annotations validation
type CreateUserDto() =
    [<Required>]
    [<StringLength(100, MinimumLength = 3)>]
    member val Name: string = "" with get, set
    
    [<Required>]
    [<EmailAddress>]
    member val Email: string = "" with get, set
    
    [<Range(18, 150)>]
    member val Age: int = 0 with get, set

// Manual validation
let validateUser (dto: CreateUserDto) =
    let context = ValidationContext(dto)
    let results = List<ValidationResult>()
    
    if Validator.TryValidateObject(dto, context, results, validateAllProperties = true) then
        Ok dto
    else
        let errors = 
            results
            |> Seq.map (fun r -> {| 
                field = r.MemberNames |> Seq.tryHead |> Option.defaultValue "unknown"
                message = r.ErrorMessage 
            |})
            |> Seq.toList
        Error errors

// FluentValidation
// dotnet add package FluentValidation
open FluentValidation

type CreateProductDto = {
    Name: string
    Price: decimal
    Category: string
    Stock: int
}

type CreateProductValidator() =
    inherit AbstractValidator<CreateProductDto>()
    
    do
        base.RuleFor(fun p -> p.Name)
            .NotEmpty().WithMessage("Name is required")
            .MinimumLength(3).WithMessage("Name must be at least 3 characters")
            .MaximumLength(200).WithMessage("Name must be at most 200 characters")
        
        base.RuleFor(fun p -> p.Price)
            .GreaterThan(0m).WithMessage("Price must be positive")
            .LessThanOrEqualTo(999999m).WithMessage("Price is too high")
        
        base.RuleFor(fun p -> p.Category)
            .NotEmpty().WithMessage("Category is required")
            .Must(fun c -> ["Electronics"; "Clothing"; "Food"] |> List.contains c)
            .WithMessage("Invalid category")
        
        base.RuleFor(fun p -> p.Stock)
            .GreaterThanOrEqualTo(0).WithMessage("Stock cannot be negative")

// Register FluentValidation
let builder = WebApplication.CreateBuilder([||])
builder.Services.AddSingleton<IValidator<CreateProductDto>, CreateProductValidator>() |> ignore

let app = builder.Build()

// Use in endpoint
app.MapPost("/products", fun (dto: CreateProductDto) (validator: IValidator<CreateProductDto>) ->
    task {
        let! validationResult = validator.ValidateAsync(dto)
        
        if not validationResult.IsValid then
            let errors = 
                validationResult.Errors
                |> Seq.map (fun e -> {| 
                    field = e.PropertyName
                    code = e.ErrorCode
                    message = e.ErrorMessage
                |})
                |> Seq.toList
            return Results.UnprocessableEntity({| errors = errors |})
        else
            let product = {|
                id = System.Guid.NewGuid()
                name = dto.Name
                price = dto.Price
                category = dto.Category
                stock = dto.Stock
            |}
            return Results.Created($"/products/{product.id}", product)
    }) |> ignore

app.Run()
```

---

## 9. Swagger Integration

```fsharp
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Microsoft.OpenApi.Models

let builder = WebApplication.CreateBuilder([||])

// Configure Swagger
builder.Services.AddEndpointsApiExplorer() |> ignore
builder.Services.AddSwaggerGen(fun opts ->
    opts.SwaggerDoc("v1", OpenApiInfo(
        Title = "My Minimal API",
        Version = "v1",
        Description = "F# Minimal API with Swagger",
        Contact = OpenApiContact(
            Name = "Dev Team",
            Email = "dev@example.com"
        )
    ))
    
    // Add JWT authentication
    opts.AddSecurityDefinition("Bearer", OpenApiSecurityScheme(
        Name = "Authorization",
        In = ParameterLocation.Header,
        Type = SecuritySchemeType.ApiKey,
        Scheme = "Bearer",
        Description = "Enter 'Bearer {token}'"
    ))
    
    let requirement = OpenApiSecurityRequirement()
    requirement.Add(
        OpenApiSecurityScheme(
            Reference = OpenApiReference(Type = ReferenceType.SecurityScheme, Id = "Bearer")),
        [||])
    opts.AddSecurityRequirement(requirement)) |> ignore

let app = builder.Build()

// Enable Swagger UI
if app.Environment.IsDevelopment() then
    app.UseSwagger() |> ignore
    app.UseSwaggerUI(fun opts ->
        opts.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1")
        opts.RoutePrefix <- "swagger") |> ignore

// Endpoints with OpenAPI metadata
app.MapGet("/users", fun () -> [{| id = 1; name = "Test" |}])
    .WithName("GetUsers")
    .WithTags("Users")
    .WithSummary("Get all users")
    .WithDescription("Returns a list of all users in the system")
    .Produces<{| id: int; name: string |} list>(200)
    .ProducesProblem(500)
|> ignore

app.MapPost("/users", fun (dto: CreateUserDto) -> {| id = 1 |})
    .WithName("CreateUser")
    .WithTags("Users")
    .WithSummary("Create a new user")
    .Accepts<CreateUserDto>("application/json")
    .Produces<{| id: int |}>(201)
    .ProducesValidationProblem()
|> ignore

app.MapGet("/users/{id:int}", fun (id: int) -> {| id = id |})
    .WithName("GetUserById")
    .WithTags("Users")
    .ExcludeFromDescription()  // ซ่อนจาก Swagger
|> ignore

// Add OpenAPI metadata globally
app.MapGet("/status", fun () -> {| status = "ok" |})
    .WithOpenApi(fun op ->
        op.Summary <- "Health check endpoint"
        op.Description <- "Returns the current status of the API"
        op) |> ignore

app.Run()
```

---

## 10. Comparing with Giraffe

```fsharp
// Giraffe Style
open Giraffe

let getUserHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match users |> List.tryFind (fun u -> u.Id = id) with
            | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) next ctx
            | Some user -> return! json user next ctx
        }

let giraffeApp : HttpHandler =
    choose [
        GET >=> route "/users" >=> getUsersHandler
        GET >=> routef "/users/%i" getUserHandler
        POST >=> route "/users" >=> createUserHandler
    ]

// Minimal API Style
let app = WebApplication.Create([||])

app.MapGet("/users", fun () -> getUsers()) |> ignore

app.MapGet("/users/{id:int}", fun (id: int) ->
    match users |> List.tryFind (fun u -> u.Id = id) with
    | None -> Results.NotFound({| error = "Not found" |})
    | Some user -> Results.Ok(user)) |> ignore

app.MapPost("/users", fun (dto: CreateUserDto) ->
    let user = createUser dto
    Results.Created($"/users/{user.Id}", user)) |> ignore
```

### เปรียบเทียบ

| คุณสมบัติ | Giraffe | Minimal API |
|-----------|---------|-------------|
| Style | Functional composition | Lambda/delegate |
| Type safety | F# type system | C#-style typing |
| Routing | route/routef/choose | MapGet/MapPost etc. |
| Middleware | HttpHandler chain | IEndpointFilter |
| OpenAPI | manual/Swashbuckle | Built-in metadata |
| Performance | High | Very high |
| Learning curve | F# focused | .NET focused |
| Interop | F#-first | C# compatible |

---

## 11. When to Choose Minimal API

```fsharp
// เลือก Minimal API เมื่อ:
// 1. ต้องการ startup รวดเร็ว
// 2. Simple CRUD APIs
// 3. Microservices ขนาดเล็ก
// 4. ต้องการ interop กับ C# ecosystem ง่าย
// 5. OpenAPI documentation เป็น priority

// เลือก Giraffe เมื่อ:
// 1. ต้องการ functional composition เต็มรูปแบบ
// 2. Complex routing logic
// 3. F# idiomatic code
// 4. ต้องการ compose handlers แบบ pipeline

// เลือก Saturn เมื่อ:
// 1. ต้องการ convention over configuration
// 2. Standard CRUD applications
// 3. ทีมที่ต้องการ structure ชัดเจน
```

---

## 12. Complete Example

```fsharp
// Complete Minimal API application
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open System.Threading.Tasks

// Models
type Todo = {
    Id: int
    Title: string
    IsCompleted: bool
    CreatedAt: System.DateTime
}

type CreateTodoDto = {
    Title: string
}

type UpdateTodoDto = {
    Title: string option
    IsCompleted: bool option
}

// In-memory store
module Store =
    let mutable todos: Todo list = [
        { Id = 1; Title = "Learn F#"; IsCompleted = true; CreatedAt = System.DateTime.UtcNow.AddDays(-7) }
        { Id = 2; Title = "Build Minimal API"; IsCompleted = false; CreatedAt = System.DateTime.UtcNow }
    ]

// Entry point
[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    
    builder.Services.AddEndpointsApiExplorer() |> ignore
    builder.Services.AddSwaggerGen() |> ignore
    builder.Services.AddCors(fun opts ->
        opts.AddDefaultPolicy(fun policy ->
            policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader() |> ignore)) |> ignore
    
    let app = builder.Build()
    
    app.UseSwagger() |> ignore
    app.UseSwaggerUI() |> ignore
    app.UseCors() |> ignore
    
    // GET all todos
    app.MapGet("/api/todos", fun () -> Store.todos)
        .WithName("GetTodos")
        .WithTags("Todos")
    |> ignore
    
    // GET todo by ID
    app.MapGet("/api/todos/{id:int}", fun (id: int) ->
        match Store.todos |> List.tryFind (fun t -> t.Id = id) with
        | None -> Results.NotFound({| error = $"Todo {id} not found" |})
        | Some todo -> Results.Ok(todo))
        .WithName("GetTodo")
        .WithTags("Todos")
    |> ignore
    
    // POST create todo
    app.MapPost("/api/todos", fun (dto: CreateTodoDto) ->
        if System.String.IsNullOrWhiteSpace(dto.Title) then
            Results.BadRequest({| errors = ["Title is required"] |})
        else
            let newId = 
                if Store.todos.IsEmpty then 1 
                else (Store.todos |> List.map (fun t -> t.Id) |> List.max) + 1
            let todo = { Id = newId; Title = dto.Title; IsCompleted = false; CreatedAt = System.DateTime.UtcNow }
            Store.todos <- Store.todos @ [todo]
            Results.Created($"/api/todos/{todo.Id}", todo))
        .WithName("CreateTodo")
        .WithTags("Todos")
    |> ignore
    
    // PUT update todo
    app.MapPut("/api/todos/{id:int}", fun (id: int) (dto: UpdateTodoDto) ->
        match Store.todos |> List.tryFindIndex (fun t -> t.Id = id) with
        | None -> Results.NotFound({| error = $"Todo {id} not found" |})
        | Some idx ->
            let existing = Store.todos.[idx]
            let updated = {
                existing with
                    Title = dto.Title |> Option.defaultValue existing.Title
                    IsCompleted = dto.IsCompleted |> Option.defaultValue existing.IsCompleted
            }
            Store.todos <- Store.todos |> List.mapi (fun i t -> if i = idx then updated else t)
            Results.Ok(updated))
        .WithName("UpdateTodo")
        .WithTags("Todos")
    |> ignore
    
    // DELETE todo
    app.MapDelete("/api/todos/{id:int}", fun (id: int) ->
        if Store.todos |> List.exists (fun t -> t.Id = id) then
            Store.todos <- Store.todos |> List.filter (fun t -> t.Id <> id)
            Results.NoContent()
        else
            Results.NotFound({| error = $"Todo {id} not found" |}))
        .WithName("DeleteTodo")
        .WithTags("Todos")
    |> ignore
    
    app.Run()
    0
```

---

## สรุป

Minimal API ใน F# มีข้อดี:
1. **Simple syntax**: lambda functions สั้นกระชับ
2. **Built-in OpenAPI**: metadata ง่าย
3. **TypedResults**: compile-time response types
4. **IEndpointFilter**: reusable filter logic
5. **Route groups**: organize related endpoints
6. **Good DI integration**: inject services ได้โดยตรง

เหมาะสำหรับ:
- Microservices
- Simple CRUD APIs
- Prototyping
- Teams ที่คุ้นเคยกับ ASP.NET Core
