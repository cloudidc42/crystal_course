# Part 54 - Falco Framework

## บทนำ

Falco เป็น F# web framework ที่มุ่งเน้นความเรียบง่าย (simplicity) และ performance บน ASP.NET Core แตกต่างจาก Giraffe และ Saturn ตรงที่ Falco มี API ที่ minimal และใช้ functional programming อย่างแท้จริง

---

## 1. What is Falco?

Falco มีคุณสมบัติหลัก:
- **Route handlers**: simple functions ที่รับ HttpContext
- **Composable**: สามารถ pipe handlers ได้
- **Request binding**: type-safe binding จาก request
- **Response helpers**: สะดวกและ expressive
- **Zero magic**: ไม่มี hidden behavior

### ติดตั้ง

```bash
mkdir MyFalcoApp
cd MyFalcoApp
dotnet new console -lang F# -n MyFalcoApp

dotnet add package Falco
dotnet add package Falco.Routing
```

### .fsproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="Program.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Falco" Version="4.0.0" />
  </ItemGroup>
</Project>
```

### Hello World ด้วย Falco

```fsharp
// Program.fs
open Falco
open Falco.Routing
open Falco.HostBuilder

[<EntryPoint>]
let main args =
    webHost args {
        endpoints [
            get "/" (Response.ofPlainText "Hello, Falco!")
        ]
    }
```

---

## 2. Route Handlers

Falco handler มี type: `HttpHandler = HttpContext -> Task`

```fsharp
open Falco
open Falco.Routing
open Falco.HostBuilder

// Simple text handler
let helloHandler : HttpHandler =
    Response.ofPlainText "Hello, World!"

// JSON handler
let jsonHandler : HttpHandler =
    Response.ofJson {| message = "Hello JSON"; status = "ok" |}

// HTML handler  
let htmlHandler : HttpHandler =
    Response.ofHtml (fun html ->
        html.Doctype()
        html.El("html", [], fun html ->
            html.El("head", [], fun html ->
                html.El("title", [], fun html -> html.Raw "Falco App"))
            html.El("body", [], fun html ->
                html.El("h1", [], fun html -> html.Raw "Hello Falco!"))))

// Handler with route parameter
let greetHandler : HttpHandler =
    fun ctx ->
        let name = Route.get ctx "name"
        Response.ofPlainText $"Hello, {name}!" ctx

// Async handler
let asyncHandler : HttpHandler =
    fun ctx ->
        task {
            let! data = fetchDataAsync ()
            return! Response.ofJson data ctx
        }

// Endpoint registration
[<EntryPoint>]
let main args =
    webHost args {
        endpoints [
            get "/"           helloHandler
            get "/json"       jsonHandler
            get "/html"       htmlHandler
            get "/hello/{name}" greetHandler
            get "/async"      asyncHandler
        ]
    }
```

### Route Patterns

```fsharp
open Falco
open Falco.Routing

// String route param
let getByName : HttpHandler =
    fun ctx ->
        let name = Route.get ctx "name"
        Response.ofPlainText $"Name: {name}" ctx

// Int route param
let getById : HttpHandler =
    fun ctx ->
        let id = Route.get ctx "id" |> int
        Response.ofJson {| id = id |} ctx

// Multiple params
let getOrderItem : HttpHandler =
    fun ctx ->
        let orderId = Route.get ctx "orderId" |> int
        let itemId = Route.get ctx "itemId" |> int
        Response.ofJson {| orderId = orderId; itemId = itemId |} ctx

// Strongly typed route values
let typedRoute : HttpHandler =
    Request.mapRoute
        (fun route ->
            {|
                UserId = route.GetInt "userId" |> Option.defaultValue 0
                Category = route.GetString "category" |> Option.defaultValue "all"
            |})
        (fun data -> Response.ofJson data)

let endpoints = [
    get "/users/{name}"                    getByName
    get "/items/{id}"                      getById
    get "/orders/{orderId}/items/{itemId}" getOrderItem
    get "/users/{userId}/category/{category}" typedRoute
]
```

---

## 3. Request Data Binding

Falco มี `Request` module สำหรับ binding request data:

```fsharp
open Falco
open Falco.Routing

// Bind query string
let queryHandler : HttpHandler =
    Request.mapQuery
        (fun query ->
            {|
                Name = query.GetString "name" |> Option.defaultValue ""
                Page = query.GetInt "page" |> Option.defaultValue 1
                Limit = query.GetInt "limit" |> Option.defaultValue 20
            |})
        (fun params ->
            Response.ofJson {|
                name = params.Name
                page = params.Page
                limit = params.Limit
            |})

// Bind form data
let formHandler : HttpHandler =
    Request.mapForm
        (fun form ->
            {|
                Username = form.GetString "username" |> Option.defaultValue ""
                Password = form.GetString "password" |> Option.defaultValue ""
                Remember = form.GetBool "remember" |> Option.defaultValue false
            |})
        (fun data ->
            if data.Username = "admin" && data.Password = "password" then
                Response.ofJson {| success = true; message = "Logged in" |}
            else
                Response.withStatusCode 401
                >> Response.ofJson {| success = false; message = "Invalid credentials" |})

// Bind JSON body
type CreateUserDto = {
    Name: string
    Email: string
    Age: int
}

let createUserHandler : HttpHandler =
    Request.mapJson<CreateUserDto>
        (fun dto ->
            if System.String.IsNullOrWhiteSpace(dto.Name) then
                Response.withStatusCode 400
                >> Response.ofJson {| errors = ["Name is required"] |}
            else
                let user = {| id = 1; name = dto.Name; email = dto.Email; age = dto.Age |}
                Response.withStatusCode 201
                >> Response.ofJson user)

// Bind with validation
let bindWithValidation : HttpHandler =
    fun ctx ->
        task {
            let! result = Request.tryBindJson<CreateUserDto> ctx
            
            match result with
            | Error msg ->
                return! (Response.withStatusCode 400 >> Response.ofJson {| error = msg |}) ctx
            | Ok dto ->
                // Process dto
                return! Response.ofJson {| received = dto |} ctx
        }

// Route + Query combination
let searchHandler : HttpHandler =
    fun ctx ->
        let category = Route.get ctx "category"
        let query = ctx.Request.Query
        let search = query["q"].ToString()
        let page = query["page"].ToString() |> fun s -> if s = "" then 1 else int s
        
        let result = {|
            category = category
            search = search
            page = page
        |}
        
        Response.ofJson result ctx
```

---

## 4. Response Writing

Falco มี `Response` module สำหรับ write responses:

```fsharp
open Falco

// Text responses
let textHandler = Response.ofPlainText "Hello World"
let htmlHandler = Response.ofHtmlString "<h1>Hello</h1>"

// JSON responses
let jsonHandler = Response.ofJson {| message = "Hello" |}

// Status codes
let notFoundHandler =
    Response.withStatusCode 404
    >> Response.ofJson {| error = "Not Found" |}

let createdHandler data =
    Response.withStatusCode 201
    >> Response.ofJson data

// Headers
let withHeadersHandler =
    Response.withHeaders [
        "X-Custom-Header", "value"
        "X-Request-Id", System.Guid.NewGuid().ToString()
    ]
    >> Response.ofJson {| status = "ok" |}

// Cookies
let setCookieHandler =
    Response.withCookie "session" "abc123"
    >> Response.ofJson {| message = "Cookie set" |}

// Redirect
let redirectHandler =
    Response.redirectTo "/new-location"

let permanentRedirectHandler =
    Response.redirectPermanentTo "/permanent-new-location"

// File response
let fileHandler (filename: string) : HttpHandler =
    fun ctx ->
        task {
            ctx.Response.Headers["Content-Disposition"] <- $"attachment; filename=\"{filename}\""
            ctx.Response.Headers["Content-Type"] <- "application/octet-stream"
            // Write file content
            do! ctx.Response.WriteAsync("file content here")
        }

// Stream response
let streamHandler : HttpHandler =
    fun ctx ->
        task {
            ctx.Response.ContentType <- "text/event-stream"
            ctx.Response.Headers["Cache-Control"] <- "no-cache"
            ctx.Response.Headers["Connection"] <- "keep-alive"
            
            for i in 1..10 do
                do! ctx.Response.WriteAsync($"data: Event {i}\n\n")
                do! System.Threading.Tasks.Task.Delay(1000)
        }
```

---

## 5. Error Handling

```fsharp
open Falco
open Falco.HostBuilder

// Custom error handler
let errorHandler (ex: System.Exception) (ctx: Microsoft.AspNetCore.Http.HttpContext) =
    let logger = ctx.GetLogger("ErrorHandler")
    logger.LogError(ex, "Unhandled exception")
    
    (Response.withStatusCode 500
     >> Response.ofJson {|
         error = "Internal Server Error"
         message = if ctx.Request.Host.Host = "localhost" then ex.Message else "An error occurred"
     |}) ctx

// Not found handler
let notFoundHandler : HttpHandler =
    Response.withStatusCode 404
    >> Response.ofJson {| error = "The requested resource was not found" |}

// Handler with try/catch
let safeHandler (handler: HttpHandler) : HttpHandler =
    fun ctx ->
        task {
            try
                return! handler ctx
            with
            | :? System.ArgumentException as ex ->
                return! (Response.withStatusCode 400 >> Response.ofJson {| error = ex.Message |}) ctx
            | :? System.UnauthorizedAccessException ->
                return! (Response.withStatusCode 401 >> Response.ofJson {| error = "Unauthorized" |}) ctx
            | ex ->
                return! (Response.withStatusCode 500 >> Response.ofJson {| error = "Internal error" |}) ctx
        }

// Domain error handling pattern
type AppError =
    | NotFound of string
    | ValidationError of string list
    | Unauthorized
    | InternalError of exn

let handleError (error: AppError) : HttpHandler =
    match error with
    | NotFound msg ->
        Response.withStatusCode 404
        >> Response.ofJson {| error = msg |}
    | ValidationError errors ->
        Response.withStatusCode 400
        >> Response.ofJson {| errors = errors |}
    | Unauthorized ->
        Response.withStatusCode 401
        >> Response.ofJson {| error = "Authentication required" |}
    | InternalError ex ->
        Response.withStatusCode 500
        >> Response.ofJson {| error = "Internal server error" |}

// Use Result type with error handling
let resultHandler : HttpHandler =
    fun ctx ->
        task {
            let result = Ok {| message = "Success" |}  // or Error (NotFound "item")
            
            match result with
            | Ok data -> return! Response.ofJson data ctx
            | Error error -> return! handleError error ctx
        }

// Register error handlers in host
let app =
    webHost [||] {
        not_found_handler notFoundHandler
        use_error_handler errorHandler
        endpoints [
            get "/" (Response.ofPlainText "Home")
        ]
    }
```

---

## 6. Middleware

```fsharp
open Falco
open Falco.HostBuilder
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Http

// ASP.NET Core middleware
let loggingMiddleware (next: RequestDelegate) (ctx: HttpContext) =
    task {
        let logger = ctx.GetLogger("Middleware")
        logger.LogInformation($"Request: {ctx.Request.Method} {ctx.Request.Path}")
        do! next.Invoke(ctx)
        logger.LogInformation($"Response: {ctx.Response.StatusCode}")
    }

// Add middleware to Falco app
let appWithMiddleware =
    webHost [||] {
        // Add ASP.NET Core middleware
        use_middleware (fun app ->
            app.Use(fun ctx next -> loggingMiddleware next ctx :> System.Threading.Tasks.Task)
            |> ignore)
        
        // Add static files
        use_static_files
        
        // Add CORS
        use_cors (fun policy ->
            policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader() |> ignore)
        
        endpoints [
            get "/" (Response.ofPlainText "With Middleware")
        ]
    }

// Falco handler as middleware
let authMiddleware (next: HttpHandler) : HttpHandler =
    fun ctx ->
        task {
            let token = ctx.Request.Headers["Authorization"].ToString()
            
            if token.StartsWith("Bearer ") then
                return! next ctx
            else
                return! (Response.withStatusCode 401 >> Response.ofJson {| error = "Unauthorized" |}) ctx
        }

// Apply middleware to specific routes
let protectedHandler : HttpHandler =
    authMiddleware (fun ctx ->
        task {
            return! Response.ofJson {| message = "Protected data" |} ctx
        })

// CORS middleware handler
let corsHandler : HttpHandler =
    fun ctx ->
        task {
            ctx.Response.Headers["Access-Control-Allow-Origin"] <- "*"
            ctx.Response.Headers["Access-Control-Allow-Methods"] <- "GET, POST, PUT, DELETE"
            ctx.Response.Headers["Access-Control-Allow-Headers"] <- "Content-Type, Authorization"
            
            if ctx.Request.Method = "OPTIONS" then
                ctx.Response.StatusCode <- 204
            else
                return! Response.ofPlainText "Next handler" ctx
        }
```

---

## 7. JSON Handling

```fsharp
open Falco
open System.Text.Json
open System.Text.Json.Serialization

// Custom JSON options
let jsonOptions =
    let opts = JsonSerializerOptions()
    opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
    opts.WriteIndented <- false
    opts.DefaultIgnoreCondition <- JsonIgnoreCondition.WhenWritingNull
    opts.Converters.Add(JsonFSharpConverter())
    opts

// Types with F# features
type OrderStatus =
    | Pending
    | Processing
    | Shipped
    | Delivered
    | Cancelled

type OrderItem = {
    ProductId: int
    ProductName: string
    Quantity: int
    UnitPrice: decimal
    TotalPrice: decimal
}

type Order = {
    Id: int
    CustomerId: int
    Status: OrderStatus
    Items: OrderItem list
    Total: decimal
    CreatedAt: System.DateTime
    Notes: string option
}

// Serialize with custom options
let serializeOrder (order: Order) =
    JsonSerializer.Serialize(order, jsonOptions)

// Deserialize
let deserializeOrder (json: string) =
    JsonSerializer.Deserialize<Order>(json, jsonOptions)

// Handler ที่ serialize/deserialize manually
let orderHandler : HttpHandler =
    fun ctx ->
        task {
            // Read body
            use reader = new System.IO.StreamReader(ctx.Request.Body)
            let! json = reader.ReadToEndAsync()
            
            try
                let order = deserializeOrder json
                
                // Process order...
                let response = {|
                    orderId = order.Id
                    status = "created"
                    total = order.Total
                |}
                
                // Write JSON response manually
                ctx.Response.ContentType <- "application/json"
                ctx.Response.StatusCode <- 201
                do! ctx.Response.WriteAsync(JsonSerializer.Serialize(response, jsonOptions))
            with ex ->
                ctx.Response.StatusCode <- 400
                do! ctx.Response.WriteAsync($"{{\"error\": \"{ex.Message}\"}}")
        }

// Configure Falco to use custom JSON serializer
let appWithCustomJson =
    webHost [||] {
        add_service (fun services ->
            services.AddSingleton<Falco.Json.ISerializer>(
                Falco.SystemTextJson.Serializer(jsonOptions))
            |> ignore)
        
        endpoints [
            post "/orders" orderHandler
        ]
    }
```

---

## 8. Form Handling

```fsharp
open Falco
open Falco.Routing

// HTML form definition
let loginFormHtml = """
<!DOCTYPE html>
<html>
<head><title>Login</title></head>
<body>
    <form method="POST" action="/login">
        <input type="text" name="username" placeholder="Username" />
        <input type="password" name="password" placeholder="Password" />
        <input type="checkbox" name="remember" value="true" /> Remember me
        <button type="submit">Login</button>
    </form>
</body>
</html>
"""

// Show login form
let showLoginForm : HttpHandler =
    Response.ofHtmlString loginFormHtml

// Process form submission
let processLogin : HttpHandler =
    Request.mapForm
        (fun form ->
            {|
                Username = form.GetString "username" |> Option.defaultValue ""
                Password = form.GetString "password" |> Option.defaultValue ""
                Remember = form.GetBool "remember" |> Option.defaultValue false
            |})
        (fun data ->
            // Validate credentials
            if data.Username = "admin" && data.Password = "secret" then
                Response.withCookie "auth_token" "valid-token-here"
                >> Response.redirectTo "/dashboard"
            else
                Response.withStatusCode 401
                >> Response.ofHtmlString """
                    <p>Invalid credentials</p>
                    <a href="/login">Try again</a>
                """)

// Multi-part form with file
let uploadFormHandler : HttpHandler =
    fun ctx ->
        task {
            let form = ctx.Request.Form
            let file = form.Files["file"]
            
            if file = null then
                return! (Response.withStatusCode 400 >> Response.ofJson {| error = "No file uploaded" |}) ctx
            else
                let filename = System.IO.Path.GetFileName(file.FileName)
                let size = file.Length
                
                // Save file
                let savePath = System.IO.Path.Combine("uploads", filename)
                use stream = System.IO.File.Create(savePath)
                do! file.CopyToAsync(stream)
                
                return! Response.ofJson {|
                    filename = filename
                    size = size
                    saved = true
                |} ctx
        }

// Registration form with validation
type RegisterForm = {
    Username: string
    Email: string
    Password: string
    ConfirmPassword: string
    Age: int
}

let validateRegisterForm (form: RegisterForm) =
    let errors = ResizeArray<string>()
    
    if System.String.IsNullOrWhiteSpace(form.Username) then
        errors.Add("Username is required")
    elif form.Username.Length < 3 then
        errors.Add("Username must be at least 3 characters")
    
    if System.String.IsNullOrWhiteSpace(form.Email) then
        errors.Add("Email is required")
    elif not (form.Email.Contains("@")) then
        errors.Add("Invalid email format")
    
    if form.Password.Length < 8 then
        errors.Add("Password must be at least 8 characters")
    
    if form.Password <> form.ConfirmPassword then
        errors.Add("Passwords do not match")
    
    if form.Age < 18 then
        errors.Add("Must be 18 or older")
    
    errors |> Seq.toList

let registerHandler : HttpHandler =
    Request.mapForm
        (fun form ->
            {
                Username = form.GetString "username" |> Option.defaultValue ""
                Email = form.GetString "email" |> Option.defaultValue ""
                Password = form.GetString "password" |> Option.defaultValue ""
                ConfirmPassword = form.GetString "confirmPassword" |> Option.defaultValue ""
                Age = form.GetInt "age" |> Option.defaultValue 0
            })
        (fun data ->
            let errors = validateRegisterForm data
            if not errors.IsEmpty then
                Response.withStatusCode 400
                >> Response.ofJson {| errors = errors |}
            else
                Response.withStatusCode 201
                >> Response.ofJson {| message = "Registration successful"; username = data.Username |})
```

---

## 9. File Uploads

```fsharp
open Falco
open Microsoft.AspNetCore.Http
open System.IO

// Maximum file size (10MB)
let maxFileSize = 10L * 1024L * 1024L

// Allowed file types
let allowedExtensions = [".jpg"; ".jpeg"; ".png"; ".gif"; ".pdf"]

// Validate uploaded file
let validateFile (file: IFormFile) =
    let errors = ResizeArray<string>()
    
    if file.Length = 0L then
        errors.Add("File is empty")
    elif file.Length > maxFileSize then
        errors.Add($"File size exceeds maximum limit of {maxFileSize / 1024L / 1024L}MB")
    
    let ext = Path.GetExtension(file.FileName).ToLower()
    if not (allowedExtensions |> List.contains ext) then
        errors.Add($"File type not allowed. Allowed types: {String.concat ", " allowedExtensions}")
    
    errors |> Seq.toList

// Single file upload
let singleUploadHandler : HttpHandler =
    fun ctx ->
        task {
            let form = ctx.Request.Form
            
            if form.Files.Count = 0 then
                return! (Response.withStatusCode 400 >> Response.ofJson {| error = "No file provided" |}) ctx
            else
                let file = form.Files.[0]
                let errors = validateFile file
                
                if not errors.IsEmpty then
                    return! (Response.withStatusCode 400 >> Response.ofJson {| errors = errors |}) ctx
                else
                    // Generate safe filename
                    let ext = Path.GetExtension(file.FileName)
                    let safeFilename = $"{System.Guid.NewGuid()}{ext}"
                    let uploadPath = Path.Combine("uploads", safeFilename)
                    
                    // Ensure directory exists
                    Directory.CreateDirectory("uploads") |> ignore
                    
                    // Save file
                    use stream = File.Create(uploadPath)
                    do! file.CopyToAsync(stream)
                    
                    return! Response.ofJson {|
                        filename = safeFilename
                        originalName = file.FileName
                        size = file.Length
                        contentType = file.ContentType
                        path = $"/uploads/{safeFilename}"
                    |} ctx
        }

// Multiple file upload
let multiUploadHandler : HttpHandler =
    fun ctx ->
        task {
            let form = ctx.Request.Form
            
            if form.Files.Count = 0 then
                return! (Response.withStatusCode 400 >> Response.ofJson {| error = "No files provided" |}) ctx
            else
                let results = ResizeArray<{| filename: string; size: int64; success: bool; error: string option |}>()
                
                for file in form.Files do
                    let errors = validateFile file
                    
                    if not errors.IsEmpty then
                        results.Add({|
                            filename = file.FileName
                            size = file.Length
                            success = false
                            error = Some (String.concat "; " errors)
                        |})
                    else
                        try
                            let ext = Path.GetExtension(file.FileName)
                            let safeFilename = $"{System.Guid.NewGuid()}{ext}"
                            let uploadPath = Path.Combine("uploads", safeFilename)
                            
                            Directory.CreateDirectory("uploads") |> ignore
                            
                            use stream = File.Create(uploadPath)
                            do! file.CopyToAsync(stream)
                            
                            results.Add({|
                                filename = safeFilename
                                size = file.Length
                                success = true
                                error = None
                            |})
                        with ex ->
                            results.Add({|
                                filename = file.FileName
                                size = file.Length
                                success = false
                                error = Some ex.Message
                            |})
                
                let successCount = results |> Seq.filter (fun r -> r.success) |> Seq.length
                let failCount = results |> Seq.filter (fun r -> not r.success) |> Seq.length
                
                return! Response.ofJson {|
                    total = form.Files.Count
                    successful = successCount
                    failed = failCount
                    files = results |> Seq.toList
                |} ctx
        }
```

---

## 10. Security Headers

```fsharp
open Falco
open Microsoft.AspNetCore.Http

// Security headers middleware
let securityHeadersMiddleware (next: RequestDelegate) (ctx: HttpContext) =
    task {
        // Prevent clickjacking
        ctx.Response.Headers["X-Frame-Options"] <- "DENY"
        
        // Prevent MIME sniffing
        ctx.Response.Headers["X-Content-Type-Options"] <- "nosniff"
        
        // XSS Protection (legacy, but still useful)
        ctx.Response.Headers["X-XSS-Protection"] <- "1; mode=block"
        
        // Content Security Policy
        ctx.Response.Headers["Content-Security-Policy"] <- 
            "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'"
        
        // Referrer Policy
        ctx.Response.Headers["Referrer-Policy"] <- "strict-origin-when-cross-origin"
        
        // Permissions Policy
        ctx.Response.Headers["Permissions-Policy"] <- "geolocation=(), microphone=(), camera=()"
        
        // Remove server header
        ctx.Response.Headers.Remove("Server") |> ignore
        
        do! next.Invoke(ctx)
    }

// HSTS header for HTTPS
let hstsMiddleware (next: RequestDelegate) (ctx: HttpContext) =
    task {
        if ctx.Request.IsHttps then
            ctx.Response.Headers["Strict-Transport-Security"] <- "max-age=31536000; includeSubDomains; preload"
        do! next.Invoke(ctx)
    }

// CORS handler
let corsHandler (allowedOrigins: string list) : HttpHandler =
    fun ctx ->
        task {
            let origin = ctx.Request.Headers["Origin"].ToString()
            
            if allowedOrigins |> List.contains origin then
                ctx.Response.Headers["Access-Control-Allow-Origin"] <- origin
            elif allowedOrigins |> List.contains "*" then
                ctx.Response.Headers["Access-Control-Allow-Origin"] <- "*"
            
            ctx.Response.Headers["Access-Control-Allow-Methods"] <- "GET, POST, PUT, PATCH, DELETE, OPTIONS"
            ctx.Response.Headers["Access-Control-Allow-Headers"] <- "Content-Type, Authorization, X-Request-Id"
            ctx.Response.Headers["Access-Control-Max-Age"] <- "86400"
            
            if ctx.Request.Method = "OPTIONS" then
                ctx.Response.StatusCode <- 204
            else
                return! Response.ofPlainText "" ctx
        }

// Rate limiting handler
let mutable private requestLog = System.Collections.Generic.Dictionary<string, (int * System.DateTime)>()

let rateLimitHandler (maxRequests: int) (windowSeconds: int) : HttpHandler =
    fun next ctx ->
        task {
            let ip = ctx.Connection.RemoteIpAddress.ToString()
            let now = System.DateTime.UtcNow
            let windowStart = now.AddSeconds(-float windowSeconds)
            
            let (count, lastReset) =
                match requestLog.TryGetValue(ip) with
                | true, (c, t) when t > windowStart -> (c, t)
                | _ -> (0, now)
            
            if count >= maxRequests then
                ctx.Response.Headers["Retry-After"] <- string windowSeconds
                return! (Response.withStatusCode 429 >> Response.ofJson {|
                    error = "Too Many Requests"
                    retryAfter = windowSeconds
                |}) ctx
            else
                requestLog.[ip] <- (count + 1, now)
                return! next ctx
        }

// Apply security in app
let secureApp =
    webHost [||] {
        use_middleware (fun app ->
            app
                .Use(fun ctx next -> securityHeadersMiddleware next ctx :> System.Threading.Tasks.Task)
                .Use(fun ctx next -> hstsMiddleware next ctx :> System.Threading.Tasks.Task)
            |> ignore)
        
        endpoints [
            get "/" (Response.ofPlainText "Secure Hello!")
        ]
    }
```

---

## 11. Complete API Example

```fsharp
// Complete Falco REST API

// Models
type Customer = {
    Id: System.Guid
    Name: string
    Email: string
    Phone: string option
    CreatedAt: System.DateTime
}

type CreateCustomerDto = {
    Name: string
    Email: string
    Phone: string option
}

type UpdateCustomerDto = {
    Name: string option
    Email: string option
    Phone: string option
}

// Store
module CustomerStore =
    open System
    let mutable private customers: Customer list = [
        {
            Id = Guid.Parse("11111111-1111-1111-1111-111111111111")
            Name = "สมชาย"
            Email = "somchai@example.com"
            Phone = Some "081-234-5678"
            CreatedAt = DateTime.UtcNow.AddDays(-30)
        }
    ]
    
    let getAll () = customers
    let getById id = customers |> List.tryFind (fun c -> c.Id = id)
    let getByEmail email = customers |> List.tryFind (fun c -> c.Email = email)
    
    let create (dto: CreateCustomerDto) =
        let customer = {
            Id = Guid.NewGuid()
            Name = dto.Name
            Email = dto.Email
            Phone = dto.Phone
            CreatedAt = DateTime.UtcNow
        }
        customers <- customers @ [customer]
        customer
    
    let update id (dto: UpdateCustomerDto) =
        match customers |> List.tryFindIndex (fun c -> c.Id = id) with
        | None -> None
        | Some idx ->
            let existing = customers.[idx]
            let updated = {
                existing with
                    Name = dto.Name |> Option.defaultValue existing.Name
                    Email = dto.Email |> Option.defaultValue existing.Email
                    Phone = match dto.Phone with Some p -> Some p | None -> existing.Phone
            }
            customers <- customers |> List.mapi (fun i c -> if i = idx then updated else c)
            Some updated
    
    let delete id =
        let existed = customers |> List.exists (fun c -> c.Id = id)
        customers <- customers |> List.filter (fun c -> c.Id <> id)
        existed

// Handlers
module CustomerHandlers =
    open Falco
    
    let list : HttpHandler =
        fun ctx ->
            task {
                let customers = CustomerStore.getAll()
                return! Response.ofJson customers ctx
            }
    
    let get : HttpHandler =
        fun ctx ->
            task {
                let idStr = Route.get ctx "id"
                match System.Guid.TryParse(idStr) with
                | false, _ ->
                    return! (Response.withStatusCode 400 >> Response.ofJson {| error = "Invalid ID format" |}) ctx
                | true, id ->
                    match CustomerStore.getById id with
                    | None ->
                        return! (Response.withStatusCode 404 >> Response.ofJson {| error = "Customer not found" |}) ctx
                    | Some customer ->
                        return! Response.ofJson customer ctx
            }
    
    let create : HttpHandler =
        Request.mapJson<CreateCustomerDto>
            (fun dto ->
                let errors = ResizeArray<string>()
                if System.String.IsNullOrWhiteSpace(dto.Name) then errors.Add("Name is required")
                if System.String.IsNullOrWhiteSpace(dto.Email) then errors.Add("Email is required")
                elif not (dto.Email.Contains("@")) then errors.Add("Invalid email format")
                
                if errors.Count > 0 then
                    Response.withStatusCode 400 >> Response.ofJson {| errors = errors |> Seq.toList |}
                else
                    match CustomerStore.getByEmail dto.Email with
                    | Some _ ->
                        Response.withStatusCode 409 >> Response.ofJson {| error = "Email already exists" |}
                    | None ->
                        let customer = CustomerStore.create dto
                        Response.withStatusCode 201 >> Response.ofJson customer)
    
    let update : HttpHandler =
        fun ctx ->
            task {
                let idStr = Route.get ctx "id"
                match System.Guid.TryParse(idStr) with
                | false, _ ->
                    return! (Response.withStatusCode 400 >> Response.ofJson {| error = "Invalid ID" |}) ctx
                | true, id ->
                    let! dto = ctx.BindJsonAsync<UpdateCustomerDto>()
                    match CustomerStore.update id dto with
                    | None ->
                        return! (Response.withStatusCode 404 >> Response.ofJson {| error = "Customer not found" |}) ctx
                    | Some updated ->
                        return! Response.ofJson updated ctx
            }
    
    let delete : HttpHandler =
        fun ctx ->
            task {
                let idStr = Route.get ctx "id"
                match System.Guid.TryParse(idStr) with
                | false, _ ->
                    return! (Response.withStatusCode 400 >> Response.ofJson {| error = "Invalid ID" |}) ctx
                | true, id ->
                    if CustomerStore.delete id then
                        ctx.Response.StatusCode <- 204
                    else
                        return! (Response.withStatusCode 404 >> Response.ofJson {| error = "Customer not found" |}) ctx
            }

// Program
open Falco
open Falco.Routing
open Falco.HostBuilder
open CustomerHandlers

[<EntryPoint>]
let main args =
    webHost args {
        use_middleware (fun app ->
            app.Use(fun ctx next -> securityHeadersMiddleware next ctx :> System.Threading.Tasks.Task)
            |> ignore)
        
        not_found_handler (Response.withStatusCode 404 >> Response.ofJson {| error = "Not Found" |})
        
        endpoints [
            get    "/"                     (Response.ofJson {| name = "Falco API"; version = "1.0" |})
            get    "/health"               (Response.ofJson {| status = "healthy" |})
            get    "/api/customers"        list
            get    "/api/customers/{id}"   get
            post   "/api/customers"        create
            put    "/api/customers/{id}"   update
            delete "/api/customers/{id}"   delete
        ]
    }
```

---

## สรุป

Falco เป็น framework ที่เรียบง่ายและ minimal สำหรับ F# web development:

1. **Route handlers** เป็น simple functions: `HttpContext -> Task`
2. **Request module** สำหรับ bind request data แบบ type-safe
3. **Response module** สำหรับ write responses อย่าง declarative
4. **Minimal magic** - เข้าใจง่าย debug ง่าย
5. **Full ASP.NET Core access** - ใช้ middleware และ services ได้หมด

Falco เหมาะกับ:
- แอปขนาดเล็กถึงกลาง
- Developer ที่ต้องการ control ทั้งหมด
- Performance-critical applications
- ทีมที่ prefer minimal abstractions
