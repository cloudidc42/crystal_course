# Part 53 - Saturn Framework

## บทนำ

Saturn เป็น F# web framework ที่สร้างอยู่บน Giraffe และ ASP.NET Core โดยได้รับแรงบันดาลใจจาก Phoenix Framework (Elixir) Saturn เพิ่ม Computation Expressions (CE) ที่ทำให้การเขียน web app มีโครงสร้างชัดเจนและอ่านง่ายขึ้น

---

## 1. What is Saturn?

Saturn มีคุณสมบัติหลัก:
- **application { }** CE สำหรับ configure แอป
- **router { }** CE สำหรับ define routes
- **controller { }** CE สำหรับ CRUD controllers
- **pipeline { }** CE สำหรับ middleware
- **channel { }** CE สำหรับ WebSocket
- Built-in authentication/authorization

### ติดตั้ง

```bash
# สร้าง project
dotnet new console -lang F# -n MySaturnApp
cd MySaturnApp

# เพิ่ม packages
dotnet add package Saturn
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

### .fsproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="Models.fs" />
    <Compile Include="Database.fs" />
    <Compile Include="Controllers.fs" />
    <Compile Include="Router.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Saturn" Version="0.16.0" />
    <PackageReference Include="Giraffe" Version="7.0.0" />
  </ItemGroup>
</Project>
```

---

## 2. application { } Computation Expression

```fsharp
// Program.fs
module Program

open Saturn

let app = application {
    // URL binding
    url "http://localhost:5000"
    
    // Routes
    use_router Router.appRouter
    
    // Static files
    use_static "wwwroot"
    
    // Gzip compression
    use_gzip
    
    // Development mode error pages
    use_developer_exceptions
    
    // Memory cache
    use_memory_cache
    
    // Custom configuration
    service_config (fun services ->
        services
            .AddSingleton<MyService>()
        |> ignore)
    
    // Custom middleware
    app_config (fun app ->
        app.UseResponseCaching() |> ignore)
    
    // Logging
    logging (fun logging ->
        logging.AddConsole() |> ignore)
}

run app
```

### application CE Options ทั้งหมด

```fsharp
let fullApp = application {
    // Network
    url "http://localhost:5000"
    url "https://localhost:5001"    // Multiple URLs
    
    // Routing
    use_router mainRouter
    
    // Static files
    use_static "wwwroot"            // Serve from wwwroot folder
    
    // Error handling
    use_developer_exceptions        // Dev mode error pages
    
    // Performance
    use_gzip                        // GZip compression
    use_response_caching            // HTTP response caching
    use_memory_cache               // In-memory cache
    
    // Security
    use_hsts                        // HTTP Strict Transport Security
    use_https_redirection           // Redirect HTTP to HTTPS
    
    // Authentication
    use_jwt_authentication          // JWT Bearer authentication
    use_cookie_authentication "my-cookie-name"  // Cookie auth
    
    // Services (DI)
    service_config (fun services -> ...)
    
    // App config (middleware pipeline)
    app_config (fun app -> ...)
    
    // Host config
    host_config (fun host -> ...)
    
    // Logging
    logging (fun logging -> ...)
    
    // Environment-specific
    use_iis                         // IIS integration
    no_router                       // When you don't want default router behavior
}
```

---

## 3. router { } Computation Expression

```fsharp
// Router.fs
module Router

open Saturn
open Giraffe

// Basic router
let basicRouter = router {
    get "/" (text "Home Page")
    get "/about" (text "About Page")
    get "/health" (json {| status = "healthy" |})
    post "/submit" (text "Submitted!")
}

// Router with parameters
let parameterRouter = router {
    getf "/users/%i" (fun id -> json {| id = id |})
    getf "/articles/%s" (fun slug -> json {| slug = slug |})
    getf "/orders/%i/items/%i" (fun (orderId, itemId) ->
        json {| orderId = orderId; itemId = itemId |})
}

// Router with pipeline (middleware)
let apiRouter = router {
    pipe_through (pipeline {
        set_header "Content-Type" "application/json"
        set_header "X-API-Version" "1.0"
    })
    
    get "/" (json {| name = "My API"; version = "1.0" |})
    get "/health" (json {| status = "ok" |})
}

// Nested routers with forward
let mainRouter = router {
    forward "/api" apiRouter
    forward "/admin" adminRouter
    forward "" pageRouter
}
```

### router CE Methods

```fsharp
let exampleRouter = router {
    // HTTP Methods
    get     "/path" handler
    post    "/path" handler
    put     "/path" handler
    patch   "/path" handler
    delete  "/path" handler
    head    "/path" handler
    options "/path" handler
    
    // Parameterized routes
    getf    "/users/%i" (fun id -> handler)
    postf   "/users/%i" (fun id -> handler)
    putf    "/users/%i/items/%s" (fun (id, name) -> handler)
    deletef "/users/%i" (fun id -> handler)
    
    // Forward to sub-router (strips prefix)
    forward "/api/users" userRouter
    forward "/api/products" productRouter
    
    // Apply pipeline to all routes in this router
    pipe_through authPipeline
    pipe_through loggingPipeline
    
    // Not Found handler
    not_found_handler (setStatusCode 404 >=> json {| error = "Not Found" |})
}
```

---

## 4. controller { } Computation Expression

Saturn's `controller` CE ช่วยสร้าง RESTful controller ได้ง่ายมาก:

```fsharp
// Models.fs
module Models

type Book = {
    Id: int
    Title: string
    Author: string
    Year: int
    ISBN: string
}

type CreateBookDto = {
    Title: string
    Author: string
    Year: int
    ISBN: string
}
```

### Basic Controller

```fsharp
// BookController.fs
module BookController

open Saturn
open Giraffe
open Models

// In-memory store
let mutable books = [
    { Id = 1; Title = "F# for Fun and Profit"; Author = "Scott Wlaschin"; Year = 2023; ISBN = "978-0-000-00000-1" }
    { Id = 2; Title = "Domain Modeling Made Functional"; Author = "Scott Wlaschin"; Year = 2018; ISBN = "978-1-680-50254-1" }
]

// Action handlers
let index (ctx: Microsoft.AspNetCore.Http.HttpContext) =
    task {
        return! json books ctx
    }

let show (ctx: Microsoft.AspNetCore.Http.HttpContext) (id: int) =
    task {
        match books |> List.tryFind (fun b -> b.Id = id) with
        | None -> return! (setStatusCode 404 >=> json {| error = "Book not found" |}) earlyReturn ctx
        | Some book -> return! json book ctx
    }

let create (ctx: Microsoft.AspNetCore.Http.HttpContext) =
    task {
        let! dto = ctx.BindJsonAsync<CreateBookDto>()
        let newId = if books.IsEmpty then 1 else (books |> List.map (fun b -> b.Id) |> List.max) + 1
        let newBook = { Id = newId; Title = dto.Title; Author = dto.Author; Year = dto.Year; ISBN = dto.ISBN }
        books <- books @ [newBook]
        return! (setStatusCode 201 >=> json newBook) earlyReturn ctx
    }

let update (ctx: Microsoft.AspNetCore.Http.HttpContext) (id: int) =
    task {
        match books |> List.tryFindIndex (fun b -> b.Id = id) with
        | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) earlyReturn ctx
        | Some idx ->
            let! dto = ctx.BindJsonAsync<CreateBookDto>()
            let updated = { Id = id; Title = dto.Title; Author = dto.Author; Year = dto.Year; ISBN = dto.ISBN }
            books <- books |> List.mapi (fun i b -> if i = idx then updated else b)
            return! json updated ctx
    }

let delete (ctx: Microsoft.AspNetCore.Http.HttpContext) (id: int) =
    task {
        books <- books |> List.filter (fun b -> b.Id <> id)
        return! setStatusCode 204 earlyReturn ctx
    }

// Define controller using CE
let bookController = controller {
    index index          // GET /
    show show            // GET /:id
    create create        // POST /
    update update        // PUT /:id
    delete delete        // DELETE /:id
}
```

### Controller CE Options

```fsharp
let fullController = controller {
    // Standard REST actions
    index   indexAction    // GET /
    show    showAction     // GET /:id
    add     addAction      // GET /add (form page)
    create  createAction   // POST /
    edit    editAction     // GET /:id/edit (form page)
    update  updateAction   // PUT /:id (or PATCH /:id)
    delete  deleteAction   // DELETE /:id
    
    // Specify ID type
    id_type RouteConstraints.intConstraint
    
    // Sub-controllers
    subController "/comments" commentController
    
    // Plugs (middleware for this controller only)
    plug [requireAuth; logRequests]
    
    // Custom version
    version 1
}
```

---

## 5. CRUD Controller ตัวอย่างสมบูรณ์

```fsharp
// Models.fs
module Models

open System

type Priority = Low | Medium | High | Critical

type Task = {
    Id: Guid
    Title: string
    Description: string
    Priority: Priority
    IsCompleted: bool
    CreatedAt: DateTime
    DueDate: DateTime option
}

type CreateTaskDto = {
    Title: string
    Description: string
    Priority: string
    DueDate: DateTime option
}

type UpdateTaskDto = {
    Title: string option
    Description: string option
    Priority: string option
    IsCompleted: bool option
    DueDate: DateTime option
}

// Repository
module TaskRepo =
    let mutable private tasks : Task list = []
    
    let getAll () = tasks
    
    let getById id = tasks |> List.tryFind (fun t -> t.Id = id)
    
    let create (dto: CreateTaskDto) =
        let priority =
            match dto.Priority.ToLower() with
            | "low" -> Low
            | "high" -> High
            | "critical" -> Critical
            | _ -> Medium
        
        let task = {
            Id = Guid.NewGuid()
            Title = dto.Title
            Description = dto.Description
            Priority = priority
            IsCompleted = false
            CreatedAt = DateTime.UtcNow
            DueDate = dto.DueDate
        }
        tasks <- tasks @ [task]
        task
    
    let update id (dto: UpdateTaskDto) =
        match tasks |> List.tryFindIndex (fun t -> t.Id = id) with
        | None -> None
        | Some idx ->
            let existing = tasks.[idx]
            let priority =
                match dto.Priority with
                | Some p ->
                    match p.ToLower() with
                    | "low" -> Low
                    | "high" -> High
                    | "critical" -> Critical
                    | _ -> existing.Priority
                | None -> existing.Priority
            
            let updated = {
                existing with
                    Title = dto.Title |> Option.defaultValue existing.Title
                    Description = dto.Description |> Option.defaultValue existing.Description
                    Priority = priority
                    IsCompleted = dto.IsCompleted |> Option.defaultValue existing.IsCompleted
                    DueDate = 
                        match dto.DueDate with
                        | Some d -> Some d
                        | None -> existing.DueDate
            }
            tasks <- tasks |> List.mapi (fun i t -> if i = idx then updated else t)
            Some updated
    
    let delete id =
        let existed = tasks |> List.exists (fun t -> t.Id = id)
        tasks <- tasks |> List.filter (fun t -> t.Id <> id)
        existed

// Controller
module TaskController =
    open Saturn
    open Giraffe
    open Microsoft.AspNetCore.Http
    open Models
    
    let index (ctx: HttpContext) =
        task {
            let tasks = TaskRepo.getAll()
            return! json tasks ctx
        }
    
    let show (ctx: HttpContext) (id: string) =
        task {
            match System.Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID" |}) earlyReturn ctx
            | true, guid ->
                match TaskRepo.getById guid with
                | None ->
                    return! (setStatusCode 404 >=> json {| error = "Task not found" |}) earlyReturn ctx
                | Some task ->
                    return! json task ctx
        }
    
    let create (ctx: HttpContext) =
        task {
            let! dto = ctx.BindJsonAsync<CreateTaskDto>()
            
            if System.String.IsNullOrWhiteSpace(dto.Title) then
                return! (setStatusCode 400 >=> json {| errors = ["Title is required"] |}) earlyReturn ctx
            else
                let task = TaskRepo.create dto
                ctx.Response.Headers["Location"] <- $"/api/tasks/{task.Id}"
                return! (setStatusCode 201 >=> json task) earlyReturn ctx
        }
    
    let update (ctx: HttpContext) (id: string) =
        task {
            match System.Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID" |}) earlyReturn ctx
            | true, guid ->
                let! dto = ctx.BindJsonAsync<UpdateTaskDto>()
                match TaskRepo.update guid dto with
                | None ->
                    return! (setStatusCode 404 >=> json {| error = "Task not found" |}) earlyReturn ctx
                | Some updated ->
                    return! json updated ctx
        }
    
    let delete (ctx: HttpContext) (id: string) =
        task {
            match System.Guid.TryParse(id) with
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID" |}) earlyReturn ctx
            | true, guid ->
                if TaskRepo.delete guid then
                    return! setStatusCode 204 earlyReturn ctx
                else
                    return! (setStatusCode 404 >=> json {| error = "Task not found" |}) earlyReturn ctx
        }
    
    let taskController = controller {
        index index
        show show
        create create
        update update
        delete delete
    }
```

---

## 6. Pipeline and Plug

```fsharp
open Saturn
open Giraffe

// Pipeline CE
let jsonPipeline = pipeline {
    // Set response headers
    set_header "Content-Type" "application/json; charset=utf-8"
    set_header "X-Content-Type-Options" "nosniff"
}

let authPipeline = pipeline {
    // Require authenticated user
    requires_authentication (Giraffe.Auth.challenge Microsoft.AspNetCore.Authentication.JwtBearer.JwtBearerDefaults.AuthenticationScheme)
}

let loggingPipeline = pipeline {
    plug (fun next ctx ->
        task {
            let logger = ctx.GetLogger("RequestPipeline")
            logger.LogInformation($"Processing {ctx.Request.Method} {ctx.Request.Path}")
            return! next ctx
        })
}

// Rate limiting pipeline
let rateLimitPipeline = pipeline {
    plug (fun next ctx ->
        task {
            // Simple rate limiting logic
            return! next ctx
        })
}

// Combine pipelines in router
let apiRouter = router {
    pipe_through jsonPipeline
    pipe_through loggingPipeline
    
    get "/" (json {| status = "ok" |})
    
    // Protected routes
    forward "/secure" (router {
        pipe_through authPipeline
        get "/" (text "Protected!")
    })
}

// pipeline { } options
let fullPipeline = pipeline {
    // Add middleware
    plug myMiddleware
    
    // Set headers
    set_header "X-Custom" "value"
    
    // Fetch model
    fetch_model<UserModel> (fun ctx -> task { return Some { Id = 1; Name = "test" } })
    
    // Require authentication
    requires_authentication challenge
    
    // Require authorization
    requires_role "admin" challenge
    
    // Require policy
    requires_policy "CanRead" challenge
}
```

---

## 7. Channels (WebSocket)

```fsharp
// ChatChannel.fs
module ChatChannel

open Saturn.Channels
open Microsoft.AspNetCore.Http

// Channel state
type Message = {
    From: string
    Text: string
    Timestamp: System.DateTime
}

// Join handler - เรียกเมื่อ client เชื่อมต่อ
let joinHandler (ctx: HttpContext) (msg: Message<string>) =
    task {
        printfn $"User joined: {ctx.Connection.Id}"
        // Broadcast join message
        return! SocketHub.broadcast ctx $"User {ctx.Connection.Id} joined the chat"
    }

// Message handler - เรียกเมื่อได้รับ message
let messageHandler (ctx: HttpContext) (msg: Message<string>) =
    task {
        let message = {
            From = ctx.Connection.Id
            Text = msg.Payload
            Timestamp = System.DateTime.UtcNow
        }
        
        printfn $"Message from {message.From}: {message.Text}"
        
        // Broadcast to all clients
        return! SocketHub.broadcast ctx (System.Text.Json.JsonSerializer.Serialize message)
    }

// Leave handler - เรียกเมื่อ client ตัดการเชื่อมต่อ
let leaveHandler (ctx: HttpContext) =
    task {
        printfn $"User left: {ctx.Connection.Id}"
        return! SocketHub.broadcast ctx $"User {ctx.Connection.Id} left the chat"
    }

// Define channel
let chatChannel = channel {
    join joinHandler
    handle "message" messageHandler
    terminate leaveHandler
}

// Register channel in router
let mainRouter = router {
    forward "/socket" (channel chatChannel "/chat")
}
```

---

## 8. Authentication

```fsharp
// Program.fs with Authentication
module Program

open Saturn
open System
open Microsoft.AspNetCore.Authentication.JwtBearer
open Microsoft.IdentityModel.Tokens
open System.Text

let jwtSecret = "my-super-secret-key-that-is-at-least-32-characters"

let app = application {
    url "http://localhost:5000"
    use_router Router.mainRouter
    
    // Configure JWT Authentication
    use_jwt_authentication_with_config (fun opts ->
        opts.TokenValidationParameters <- TokenValidationParameters(
            ValidateIssuer = true,
            ValidIssuer = "my-app",
            ValidateAudience = true,
            ValidAudience = "my-users",
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = SymmetricSecurityKey(Encoding.UTF8.GetBytes(jwtSecret))
        ))
    
    service_config (fun services ->
        services.AddAuthorization(fun opts ->
            opts.AddPolicy("AdminOnly", fun policy ->
                policy.RequireRole("admin") |> ignore)
            opts.AddPolicy("ReadAccess", fun policy ->
                policy.RequireClaim("permission", "read") |> ignore))
        |> ignore)
}
```

### สร้าง JWT Token

```fsharp
// Auth.fs
module Auth

open System
open System.Security.Claims
open Microsoft.IdentityModel.Tokens
open System.IdentityModel.Tokens.Jwt
open System.Text

let private secret = "my-super-secret-key-that-is-at-least-32-characters"
let private issuer = "my-app"
let private audience = "my-users"

type LoginDto = {
    Username: string
    Password: string
}

type TokenResponse = {
    AccessToken: string
    ExpiresAt: DateTime
    TokenType: string
}

let createToken (username: string) (roles: string list) =
    let key = SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret))
    let credentials = SigningCredentials(key, SecurityAlgorithms.HmacSha256)
    let expiresAt = DateTime.UtcNow.AddHours(8.0)
    
    let claims = [
        yield Claim(ClaimTypes.Name, username)
        yield Claim(JwtRegisteredClaimNames.Sub, username)
        yield Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        yield! roles |> List.map (fun r -> Claim(ClaimTypes.Role, r))
    ]
    
    let token = JwtSecurityToken(
        issuer = issuer,
        audience = audience,
        claims = claims,
        expires = expiresAt,
        signingCredentials = credentials
    )
    
    {
        AccessToken = JwtSecurityTokenHandler().WriteToken(token)
        ExpiresAt = expiresAt
        TokenType = "Bearer"
    }

// Login handler
let loginHandler : Giraffe.HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<LoginDto>()
            
            // ตรวจสอบ credentials (production ควรตรวจกับ database)
            match dto.Username, dto.Password with
            | "admin", "password123" ->
                let token = createToken "admin" ["admin"; "user"]
                return! (Giraffe.json token) next ctx
            | "user", "password" ->
                let token = createToken "user" ["user"]
                return! (Giraffe.json token) next ctx
            | _ ->
                return! (Giraffe.setStatusCode 401 >=> Giraffe.json {| error = "Invalid credentials" |}) next ctx
        }
```

---

## 9. Authorization

```fsharp
open Saturn
open Giraffe
open Microsoft.AspNetCore.Authorization

// Role-based authorization
let requireRole (role: string) : HttpHandler =
    authorizeByPolicy $"Role:{role}" (
        setStatusCode 403 >=> json {| error = "Insufficient permissions" |}
    )

// Policy-based authorization
let requirePolicy (policyName: string) : HttpHandler =
    authorizeByPolicy policyName (
        setStatusCode 403 >=> json {| error = "Access denied" |}
    )

// Custom authorization check
let requireOwnership (getUserId: HttpContext -> int) : HttpHandler =
    fun next ctx ->
        task {
            let userId = getUserId ctx
            let resourceId = ctx.GetRouteValue("id") |> int
            
            // ตรวจสอบว่า user เป็นเจ้าของ resource
            if userId = resourceId then
                return! next ctx
            else
                return! (setStatusCode 403 >=> json {| error = "You don't own this resource" |}) next ctx
        }

// Protected routes
let protectedRouter = router {
    pipe_through authPipeline
    
    get "/profile" (fun next ctx ->
        task {
            let username = ctx.User.Identity.Name
            return! (json {| username = username; authenticated = true |}) next ctx
        })
    
    // Admin only
    forward "/admin" (router {
        pipe_through (pipeline { requires_role "admin" (Giraffe.Auth.challenge JwtBearerDefaults.AuthenticationScheme) })
        get "/" (text "Admin Panel")
    })
}
```

---

## 10. Saturn vs Giraffe Comparison

### Giraffe Style

```fsharp
// Giraffe - functional, composable
let webApp =
    choose [
        GET >=> route "/users" >=> getUsersHandler
        POST >=> route "/users" >=> createUserHandler
        GET >=> routef "/users/%i" getUser
        PUT >=> routef "/users/%i" updateUser
        DELETE >=> routef "/users/%i" deleteUser
        setStatusCode 404 >=> text "Not Found"
    ]
```

### Saturn Style

```fsharp
// Saturn - structured CE syntax
let userController = controller {
    index   listUsers
    show    getUser
    create  createUser
    update  updateUser
    delete  deleteUser
}

let mainRouter = router {
    forward "/users" userController
}

let app = application {
    url "http://localhost:5000"
    use_router mainRouter
    use_gzip
    use_developer_exceptions
}

run app
```

### เปรียบเทียบคุณสมบัติ

| คุณสมบัติ | Giraffe | Saturn |
|-----------|---------|--------|
| Style | Functional composition | CE-based |
| Boilerplate | ต่ำ | ต่ำมาก |
| Convention | Manual | Convention over Configuration |
| Flexibility | สูงมาก | สูง |
| Learning curve | ปานกลาง | ต่ำ |
| Authentication | Manual | Built-in helpers |
| WebSocket | ต้องตั้งเอง | channel { } CE |
| Use case | Custom APIs | Standard web apps |

---

## 11. Complete Example Application

ตัวอย่างแอป Blog สมบูรณ์ด้วย Saturn:

```fsharp
// Models.fs
module Models

open System

type Tag = { Id: int; Name: string }

type Post = {
    Id: int
    Title: string
    Slug: string
    Content: string
    Summary: string
    AuthorId: int
    Tags: string list
    IsPublished: bool
    PublishedAt: DateTime option
    CreatedAt: DateTime
    UpdatedAt: DateTime
}

type User = {
    Id: int
    Username: string
    Email: string
    Role: string
}

type CreatePostDto = {
    Title: string
    Content: string
    Summary: string
    Tags: string list
}

// Data store
module Store =
    let mutable users = [
        { Id = 1; Username = "admin"; Email = "admin@blog.com"; Role = "admin" }
        { Id = 2; Username = "writer"; Email = "writer@blog.com"; Role = "writer" }
    ]
    
    let mutable posts = [
        {
            Id = 1
            Title = "Getting Started with F#"
            Slug = "getting-started-with-fsharp"
            Content = "F# is a functional-first language..."
            Summary = "Introduction to F# programming"
            AuthorId = 1
            Tags = ["fsharp"; "programming"; "functional"]
            IsPublished = true
            PublishedAt = Some DateTime.UtcNow
            CreatedAt = DateTime.UtcNow.AddDays(-7)
            UpdatedAt = DateTime.UtcNow.AddDays(-1)
        }
    ]
    
    let createSlug (title: string) =
        title.ToLower().Replace(" ", "-").Replace("'", "").Replace(",", "")

// Post Controller
module PostController =
    open Saturn
    open Giraffe
    open Microsoft.AspNetCore.Http
    open Models
    
    let index (ctx: HttpContext) =
        task {
            let published = Store.posts |> List.filter (fun p -> p.IsPublished)
            return! json published ctx
        }
    
    let show (ctx: HttpContext) (id: int) =
        task {
            match Store.posts |> List.tryFind (fun p -> p.Id = id) with
            | None -> return! (setStatusCode 404 >=> json {| error = "Post not found" |}) earlyReturn ctx
            | Some post -> return! json post ctx
        }
    
    let create (ctx: HttpContext) =
        task {
            // Get current user from JWT claims
            let authorId = 
                ctx.User.Claims 
                |> Seq.tryFind (fun c -> c.Type = System.Security.Claims.ClaimTypes.NameIdentifier)
                |> Option.map (fun c -> int c.Value)
                |> Option.defaultValue 1
            
            let! dto = ctx.BindJsonAsync<CreatePostDto>()
            
            if System.String.IsNullOrWhiteSpace(dto.Title) then
                return! (setStatusCode 400 >=> json {| errors = ["Title is required"] |}) earlyReturn ctx
            else
                let newId = if Store.posts.IsEmpty then 1 else (Store.posts |> List.map (fun p -> p.Id) |> List.max) + 1
                let now = DateTime.UtcNow
                let post = {
                    Id = newId
                    Title = dto.Title
                    Slug = Store.createSlug dto.Title
                    Content = dto.Content
                    Summary = dto.Summary
                    AuthorId = authorId
                    Tags = dto.Tags
                    IsPublished = false
                    PublishedAt = None
                    CreatedAt = now
                    UpdatedAt = now
                }
                Store.posts <- Store.posts @ [post]
                ctx.Response.Headers["Location"] <- $"/api/posts/{post.Id}"
                return! (setStatusCode 201 >=> json post) earlyReturn ctx
        }
    
    let update (ctx: HttpContext) (id: int) =
        task {
            match Store.posts |> List.tryFindIndex (fun p -> p.Id = id) with
            | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) earlyReturn ctx
            | Some idx ->
                let! dto = ctx.BindJsonAsync<CreatePostDto>()
                let existing = Store.posts.[idx]
                let updated = {
                    existing with
                        Title = if System.String.IsNullOrWhiteSpace(dto.Title) then existing.Title else dto.Title
                        Content = if System.String.IsNullOrWhiteSpace(dto.Content) then existing.Content else dto.Content
                        Summary = if System.String.IsNullOrWhiteSpace(dto.Summary) then existing.Summary else dto.Summary
                        Tags = if dto.Tags.IsEmpty then existing.Tags else dto.Tags
                        UpdatedAt = DateTime.UtcNow
                }
                Store.posts <- Store.posts |> List.mapi (fun i p -> if i = idx then updated else p)
                return! json updated ctx
        }
    
    let delete (ctx: HttpContext) (id: int) =
        task {
            Store.posts <- Store.posts |> List.filter (fun p -> p.Id <> id)
            return! setStatusCode 204 earlyReturn ctx
        }
    
    let postController = controller {
        index index
        show show
        create create
        update update
        delete delete
    }

// Router.fs
module Router =
    open Saturn
    open Giraffe
    
    let authPipeline = pipeline {
        plug (fun next ctx ->
            task {
                if ctx.User.Identity.IsAuthenticated then
                    return! next ctx
                else
                    return! (setStatusCode 401 >=> json {| error = "Unauthorized" |}) earlyReturn ctx
            })
    }
    
    let apiRouter = router {
        forward "/posts" PostController.postController
    }
    
    let mainRouter = router {
        // Public routes
        get "/" (json {| name = "Blog API"; version = "1.0" |})
        get "/health" (json {| status = "healthy" |})
        
        // API routes
        forward "/api" apiRouter
        
        // Auth routes
        post "/auth/login" Auth.loginHandler
        
        // Not found
        not_found_handler (setStatusCode 404 >=> json {| error = "Not Found" |})
    }

// Program.fs
module Program =
    open Saturn
    
    let app = application {
        url "http://localhost:5000"
        use_router Router.mainRouter
        use_gzip
        use_developer_exceptions
        
        service_config (fun services ->
            services.AddCors(fun opts ->
                opts.AddDefaultPolicy(fun policy ->
                    policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader() |> ignore))
            |> ignore)
        
        app_config (fun app ->
            app.UseCors() |> ignore)
    }
    
    run app
```

---

## สรุป

Saturn ทำให้การสร้าง web apps ด้วย F# ง่ายขึ้นมากด้วย CE syntax ที่ชัดเจน:

- **`application { }`** - กำหนดการตั้งค่าทั้งหมดของแอป
- **`router { }`** - กำหนด routing
- **`controller { }`** - สร้าง RESTful controller อย่างรวดเร็ว
- **`pipeline { }`** - สร้าง middleware chain
- **`channel { }`** - สร้าง WebSocket endpoint

Saturn เหมาะสำหรับ:
- Web APIs ที่ต้องการ convention-over-configuration
- แอปที่มี standard CRUD operations
- ทีมที่ต้องการ structure ชัดเจน

ใช้ Giraffe เมื่อ:
- ต้องการ flexibility สูง
- Custom routing logic ซับซ้อน
- Minimal setup
