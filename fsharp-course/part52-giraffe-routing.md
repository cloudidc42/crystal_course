# Part 52 - Giraffe Routing เชิงลึก

## บทนำ

Routing ใน Giraffe ใช้แนวคิด functional composition ในการ match HTTP requests กับ handlers ต่างๆ บทนี้จะเจาะลึกทุกแง่มุมของ routing ใน Giraffe

---

## 1. route Function

`route` เป็นฟังก์ชันพื้นฐานที่ match URL path แบบ exact match:

```fsharp
open Giraffe

// Exact path matching
let webApp : HttpHandler =
    choose [
        route "/"            >=> text "Home"
        route "/about"       >=> text "About"
        route "/contact"     >=> text "Contact"
        route "/api/health"  >=> json {| status = "ok" |}
        setStatusCode 404    >=> text "Not Found"
    ]

// route เป็น case-insensitive โดย default
// /About และ /about จะ match route "/about" ทั้งคู่

// routeCi - case insensitive ชัดเจน
let webAppCi : HttpHandler =
    choose [
        routeCi "/about" >=> text "About Page"
    ]

// routeStartsWith - match ถ้า path เริ่มด้วย prefix
let webAppPrefix : HttpHandler =
    choose [
        routeStartsWith "/api" >=> text "API endpoint"
        routeStartsWith "/admin" >=> text "Admin area"
    ]

// routeStartsWithCi - case insensitive version
let webAppPrefixCi : HttpHandler =
    choose [
        routeStartsWithCi "/api" >=> text "API endpoint"
    ]
```

### route vs routeStartsWith

```fsharp
// route "/api" - match เฉพาะ /api เท่านั้น
// routeStartsWith "/api" - match /api, /api/users, /api/products, etc.

let example : HttpHandler =
    choose [
        // นี้ match เฉพาะ /api
        route "/api" >=> text "API root"
        
        // นี้ match /api/users, /api/users/1, etc.
        routeStartsWith "/api/users" >=> usersRouter
        
        // นี้ match ทุก /api/** ที่เหลือ
        routeStartsWith "/api" >=> text "Other API"
    ]
```

---

## 2. routef with Format Strings

`routef` ใช้สำหรับ route ที่มี parameters โดยใช้ printf-style format strings:

```fsharp
open Giraffe

// Integer parameter (%i)
let getUser (id: int) : HttpHandler =
    json {| id = id; name = $"User {id}" |}

// String parameter (%s)
let getBySlug (slug: string) : HttpHandler =
    json {| slug = slug; title = $"Article: {slug}" |}

// Float parameter (%f)
let getByPrice (price: float) : HttpHandler =
    json {| price = price |}

// Guid parameter (%O หรือ %o)
let getByGuid (id: System.Guid) : HttpHandler =
    json {| id = id |}

// Multiple parameters
let getOrderItem (orderId: int) (itemId: int) : HttpHandler =
    json {| orderId = orderId; itemId = itemId |}

let webApp : HttpHandler =
    choose [
        routef "/users/%i" getUser                     // /users/123
        routef "/articles/%s" getBySlug                // /articles/my-article
        routef "/products/%f" getByPrice               // /products/29.99
        routef "/items/%O" getByGuid                   // /items/a1b2c3d4-...
        routef "/orders/%i/items/%i" getOrderItem      // /orders/1/items/2
    ]

// routef format types:
// %b - bool
// %c - char
// %s - string (stops at /)
// %i - int
// %d - int64
// %f - float
// %O - any type with IConvertible
// %u - uint64
```

### ตัวอย่าง routef ที่ซับซ้อน

```fsharp
// Version-prefixed API routes
let getApiV2User (version: string) (userId: int) : HttpHandler =
    json {| version = version; userId = userId |}

// Category and product
let getProductInCategory (category: string) (productId: int) : HttpHandler =
    fun next ctx ->
        task {
            let product = {|
                category = category
                productId = productId
                name = $"{category} Product {productId}"
            |}
            return! json product next ctx
        }

let webApp : HttpHandler =
    choose [
        routef "/api/%s/users/%i" getApiV2User
        routef "/categories/%s/products/%i" getProductInCategory
    ]
```

---

## 3. routeStartsWith

```fsharp
open Giraffe

// Group routes by prefix
let apiRoutes : HttpHandler =
    choose [
        routeStartsWith "/api/v1" >=> v1Routes
        routeStartsWith "/api/v2" >=> v2Routes
        routeStartsWith "/api"    >=> json {| error = "Unknown API version" |}
    ]

// ใช้กับ middleware
let requireApiKey : HttpHandler =
    fun next ctx ->
        task {
            let apiKey = ctx.Request.Headers["X-API-Key"].ToString()
            if apiKey = "valid-key" then
                return! next ctx
            else
                return! (setStatusCode 401 >=> text "Invalid API key") next ctx
        }

let secureApiRoutes : HttpHandler =
    routeStartsWith "/api/secure" >=>
    requireApiKey >=>
    choose [
        GET >=> route "/api/secure/data" >=> text "Secret data"
        GET >=> route "/api/secure/users" >=> text "Secret users"
    ]
```

---

## 4. subRoute

`subRoute` ตัด prefix ออกก่อนส่งไป nested handler:

```fsharp
open Giraffe

// subRoute ตัด "/api" ออก แล้วส่ง request ที่เหลือให้ inner handler
let apiRouter : HttpHandler =
    subRoute "/api" (
        choose [
            // ภายใน subRoute จะ match กับ path ที่ตัด /api ออกแล้ว
            route "/users"    >=> text "Users"      // match /api/users
            route "/products" >=> text "Products"   // match /api/products
            route "/orders"   >=> text "Orders"     // match /api/orders
        ]
    )

// Nested subRoutes
let nestedRouter : HttpHandler =
    subRoute "/api" (
        choose [
            subRoute "/v1" (
                choose [
                    route "/users" >=> text "API v1 users"
                    route "/items" >=> text "API v1 items"
                ]
            )
            subRoute "/v2" (
                choose [
                    route "/users" >=> text "API v2 users"
                    route "/items" >=> text "API v2 items"
                ]
            )
        ]
    )

// subRoute with middleware
let adminRouter : HttpHandler =
    subRoute "/admin" (
        requireAuthentication >=>  // Apply auth to all admin routes
        choose [
            route "/dashboard" >=> text "Admin Dashboard"
            route "/users"     >=> text "Manage Users"
            route "/settings"  >=> text "Settings"
        ]
    )
```

### subRoute vs routeStartsWith

```fsharp
// routeStartsWith - ไม่ตัด prefix
// handlers ข้างในต้องใช้ path เต็ม
let withRouteStartsWith : HttpHandler =
    routeStartsWith "/api" >=>
    choose [
        route "/api/users"    >=> text "Users"    // ต้องใช้ path เต็ม
        route "/api/products" >=> text "Products"
    ]

// subRoute - ตัด prefix ออก
// handlers ข้างในใช้ path ที่เหลือ
let withSubRoute : HttpHandler =
    subRoute "/api" (
        choose [
            route "/users"    >=> text "Users"    // ใช้ path ที่ตัดแล้ว
            route "/products" >=> text "Products"
        ]
    )
```

---

## 5. choose for Multiple Routes

```fsharp
open Giraffe

// choose จะลอง match ทีละตัวจนกว่าจะพบ handler ที่ match
let webApp : HttpHandler =
    choose [
        // Static routes ก่อน (เร็วกว่า)
        route "/"       >=> indexHandler
        route "/about"  >=> aboutHandler
        route "/contact" >=> contactHandler
        
        // API routes
        subRoute "/api" apiRoutes
        
        // Admin routes
        subRoute "/admin" adminRoutes
        
        // Catch-all 404
        setStatusCode 404 >=> notFoundHandler
    ]

// choose กับ HTTP methods
let userRoutes : HttpHandler =
    choose [
        GET    >=> route "/users"      >=> getUsersHandler
        GET    >=> routef "/users/%i"  getUserHandler
        POST   >=> route "/users"      >=> createUserHandler
        PUT    >=> routef "/users/%i"  updateUserHandler
        PATCH  >=> routef "/users/%i"  patchUserHandler
        DELETE >=> routef "/users/%i"  deleteUserHandler
    ]

// Nested choose
let webApp2 : HttpHandler =
    choose [
        GET >=> choose [
            route "/users"        >=> getUsersHandler
            routef "/users/%i"    getUser
            route "/products"     >=> getProductsHandler
            routef "/products/%i" getProduct
        ]
        POST >=> choose [
            route "/users"        >=> createUser
            route "/products"     >=> createProduct
        ]
        setStatusCode 404 >=> json {| error = "Not found" |}
    ]
```

---

## 6. Route Parameters

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// อ่าน route value จาก context
let getRouteValue (ctx: HttpContext) (key: string) =
    ctx.GetRouteValue(key) |> Option.ofObj |> Option.map string

// Route values จาก ASP.NET Core routing
let handler : HttpHandler =
    fun next ctx ->
        task {
            // อ่านจาก route data
            let id = ctx.GetRouteValue("id") :?> string
            return! text $"ID: {id}" next ctx
        }

// ใช้กับ routef - strongly typed
let typedRouteHandler (userId: int) (resourceId: string) : HttpHandler =
    fun next ctx ->
        task {
            // userId และ resourceId ได้ type ที่ถูกต้องแล้ว
            return! json {| userId = userId; resourceId = resourceId |} next ctx
        }

let webApp : HttpHandler =
    routef "/users/%i/resources/%s" typedRouteHandler

// Optional route segments (ต้องใช้ choose)
let optionalSegmentRoutes : HttpHandler =
    choose [
        routef "/posts/%i/%s" (fun (id, slug) -> json {| id = id; slug = slug |})
        routef "/posts/%i"    (fun id -> json {| id = id |})
        route "/posts"        >=> json {| message = "all posts" |}
    ]
```

---

## 7. Query String Binding

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// อ่าน query parameter ทีละตัว
let searchHandler : HttpHandler =
    fun next ctx ->
        task {
            // TryGetQueryStringValue return Option<string>
            let q = ctx.TryGetQueryStringValue "q"
            let page = ctx.TryGetQueryStringValue "page" |> Option.map int |> Option.defaultValue 1
            let limit = ctx.TryGetQueryStringValue "limit" |> Option.map int |> Option.defaultValue 20
            let sortBy = ctx.TryGetQueryStringValue "sortBy" |> Option.defaultValue "createdAt"
            let order = ctx.TryGetQueryStringValue "order" |> Option.defaultValue "desc"
            
            let result = {|
                query = q
                page = page
                limit = limit
                sortBy = sortBy
                order = order
            |}
            
            return! json result next ctx
        }

// BindQueryString - bind ทั้ง object
type SearchParams = {
    Q: string
    Page: int
    Limit: int
    SortBy: string
    Tags: string list
}

// กำหนด default values
type SearchParams2 = {
    Q: string
    Page: int
    Limit: int
}
with
    static member Default = { Q = ""; Page = 1; Limit = 20 }

let bindQueryHandler : HttpHandler =
    fun next ctx ->
        task {
            let! params' = ctx.BindQueryStringAsync<SearchParams2>()
            
            // Validate and sanitize
            let validatedParams = {
                Q = params'.Q |> Option.ofObj |> Option.defaultValue ""
                Page = max 1 params'.Page
                Limit = min 100 (max 1 params'.Limit)
            }
            
            return! json validatedParams next ctx
        }

// อ่าน multiple values สำหรับ key เดียวกัน
let multiValueQueryHandler : HttpHandler =
    fun next ctx ->
        task {
            // ?tags=f#&tags=dotnet&tags=web
            let tags = 
                ctx.Request.Query["tags"]
                |> Seq.toList
            
            return! json {| tags = tags |} next ctx
        }
```

---

## 8. Header Binding

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open Microsoft.Extensions.Primitives

// อ่าน single header
let readHeaderHandler : HttpHandler =
    fun next ctx ->
        task {
            let contentType = ctx.Request.Headers["Content-Type"].ToString()
            let authorization = ctx.Request.Headers["Authorization"].ToString()
            let userAgent = ctx.Request.Headers["User-Agent"].ToString()
            
            return! json {|
                contentType = contentType
                hasAuth = not (System.String.IsNullOrEmpty(authorization))
                userAgent = userAgent
            |} next ctx
        }

// Custom header middleware
let requireCustomHeader (headerName: string) (value: string) : HttpHandler =
    fun next ctx ->
        task {
            let headerValue = ctx.Request.Headers[headerName].ToString()
            if headerValue = value then
                return! next ctx
            else
                return! (setStatusCode 400 >=> json {| error = $"Missing or invalid {headerName} header" |}) next ctx
        }

// API version from header
let apiVersionHandler : HttpHandler =
    fun next ctx ->
        task {
            let version = 
                ctx.Request.Headers["Api-Version"]
                |> StringValues.op_Implicit
                |> Option.ofObj
                |> Option.defaultValue "v1"
            
            let response = {|
                version = version
                message = $"Using API version {version}"
            |}
            
            return! json response next ctx
        }

// BindHeader - strongly typed header binding
type RequestHeaders = {
    Authorization: string
    ContentType: string
    AcceptLanguage: string
}

let bindHeadersHandler : HttpHandler =
    fun next ctx ->
        task {
            let headers = {
                Authorization = ctx.Request.Headers["Authorization"].ToString()
                ContentType = ctx.Request.ContentType
                AcceptLanguage = ctx.Request.Headers["Accept-Language"].ToString()
            }
            return! json headers next ctx
        }

// Set response headers
let setCustomHeadersHandler : HttpHandler =
    fun next ctx ->
        task {
            ctx.Response.Headers["X-Powered-By"] <- "Giraffe/F#"
            ctx.Response.Headers["X-Request-Id"] <- System.Guid.NewGuid().ToString()
            ctx.Response.Headers["Cache-Control"] <- "no-cache"
            return! next ctx
        }
```

---

## 9. Custom Route Constraints

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open System.Text.RegularExpressions

// Custom constraint - validate UUID format
let requireGuid (next: HttpHandler) : HttpHandler =
    fun next' ctx ->
        task {
            let id = ctx.GetRouteValue("id") |> string
            match System.Guid.TryParse(id) with
            | true, _ -> return! next next' ctx
            | false, _ ->
                return! (setStatusCode 400 >=> json {| error = "Invalid ID format, must be a valid GUID" |}) next' ctx
        }

// Custom constraint - validate slug format
let slugConstraint (slug: string) =
    Regex.IsMatch(slug, "^[a-z0-9-]+$")

let routeWithSlugConstraint (handler: string -> HttpHandler) : HttpHandler =
    routef "/articles/%s" (fun slug ->
        if slugConstraint slug then
            handler slug
        else
            setStatusCode 400 >=> json {| error = "Invalid slug format" |}
    )

// Constraint ตรวจสอบ range
let routeWithRangeConstraint (min: int) (max: int) (handler: int -> HttpHandler) : HttpHandler =
    fun next ctx ->
        task {
            match ctx.TryGetRouteValue "id" with
            | None -> return! (setStatusCode 400 >=> text "Missing id") next ctx
            | Some id ->
                match System.Int32.TryParse(id.ToString()) with
                | false, _ -> return! (setStatusCode 400 >=> text "Invalid id") next ctx
                | true, n when n < min || n > max ->
                    return! (setStatusCode 400 >=> json {| error = $"ID must be between {min} and {max}" |}) next ctx
                | true, n ->
                    return! handler n next ctx
        }

// Regex route matching
let routeByRegex (pattern: string) (handler: HttpHandler) : HttpHandler =
    fun next ctx ->
        task {
            let path = ctx.Request.Path.Value
            if Regex.IsMatch(path, pattern) then
                return! handler next ctx
            else
                return! next ctx
        }

// ตัวอย่างการใช้งาน custom constraints
let webApp : HttpHandler =
    choose [
        // Route ที่ต้องการ slug format
        routeWithSlugConstraint (fun slug ->
            json {| slug = slug; title = $"Article: {slug}" |}
        )
        
        // Route พร้อม range constraint
        routeWithRangeConstraint 1 9999 (fun id ->
            json {| id = id; message = $"Valid ID: {id}" |}
        )
        
        // Regex route
        routeByRegex "^/files/[A-Za-z0-9_-]+\\.pdf$" (
            fun next ctx ->
                task {
                    let filename = ctx.Request.Path.Value.TrimStart('/')
                    return! text $"Serving file: {filename}" next ctx
                }
        )
    ]
```

---

## 10. Middleware Integration

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection

// ASP.NET Core Middleware ทำงานก่อน Giraffe
let configureApp (app: IApplicationBuilder) =
    app
        // Standard middleware
        .UseStaticFiles()
        .UseAuthentication()
        .UseAuthorization()
        .UseResponseCompression()
        
        // Custom middleware
        .Use(fun ctx (next: System.Func<System.Threading.Tasks.Task>) ->
            task {
                // Before request
                ctx.Response.Headers["X-Request-Id"] <- System.Guid.NewGuid().ToString()
                
                do! next.Invoke()
                
                // After request
                // (ทำงานหลังจาก response headers ถูก sent แล้ว)
            } :> System.Threading.Tasks.Task)
        
        // Giraffe handler
        .UseGiraffe(webApp)
    |> ignore

// Giraffe-level middleware (HttpHandler)
let requestLoggingMiddleware : HttpHandler =
    fun next ctx ->
        task {
            let start = System.DateTime.UtcNow
            let logger = ctx.GetLogger("RequestLogging")
            
            logger.LogInformation($"[{ctx.TraceIdentifier}] {ctx.Request.Method} {ctx.Request.Path}")
            
            let! result = next ctx
            
            let duration = (System.DateTime.UtcNow - start).TotalMilliseconds
            logger.LogInformation($"[{ctx.TraceIdentifier}] Completed in {duration:F2}ms")
            
            return result
        }

// Inject middleware into route tree
let webApp : HttpHandler =
    requestLoggingMiddleware >=>
    choose [
        GET >=> route "/" >=> text "Home"
        subRoute "/api" apiRoutes
    ]

// Middleware กับ DI
let dbMiddleware : HttpHandler =
    fun next ctx ->
        task {
            // ดึง service จาก DI container
            let db = ctx.GetService<IDbContext>()
            
            // เพิ่ม item ลงใน request context
            ctx.Items.["db"] <- db
            
            return! next ctx
        }

// อ่าน service จาก context ใน handler
let handlerUsingDb : HttpHandler =
    fun next ctx ->
        task {
            let db = ctx.Items.["db"] :?> IDbContext
            // ใช้ db...
            return! text "OK" next ctx
        }
```

---

## 11. Routing Best Practices

```fsharp
open Giraffe

// 1. จัดเรียง routes จาก specific ไป general
let goodRouteOrder : HttpHandler =
    choose [
        // Specific first
        GET >=> route "/api/users/me"        >=> getMeHandler         // ต้องมาก่อน
        GET >=> routef "/api/users/%i"       getUser                  // ไม่งั้น "me" จะถูก match เป็น string
        GET >=> route "/api/users"           >=> getUsersHandler
    ]

// 2. Group related routes ด้วย subRoute
let wellOrganizedRoutes : HttpHandler =
    choose [
        subRoute "/api" (
            choose [
                subRoute "/users" userRoutes
                subRoute "/products" productRoutes
                subRoute "/orders" orderRoutes
            ]
        )
        subRoute "/admin" adminRoutes
        route "/" >=> indexHandler
    ]

// 3. แยก route definitions ออกเป็น modules
// Users.fs
module Users =
    let routes : HttpHandler =
        choose [
            GET    >=> route ""      >=> listUsers
            POST   >=> route ""      >=> createUser
            GET    >=> routef "/%i"  getUser
            PUT    >=> routef "/%i"  updateUser
            DELETE >=> routef "/%i"  deleteUser
        ]

// Products.fs
module Products =
    let routes : HttpHandler =
        choose [
            GET    >=> route ""      >=> listProducts
            POST   >=> route ""      >=> createProduct
            GET    >=> routef "/%i"  getProduct
        ]

// Router.fs - combine all routes
let apiRoutes : HttpHandler =
    subRoute "/api" (
        choose [
            subRoute "/users"    Users.routes
            subRoute "/products" Products.routes
        ]
    )

// 4. ใช้ HTTP method combinators อย่างถูกต้อง
let httpMethodHandlers : HttpHandler =
    choose [
        GET    >=> route "/resource" >=> getHandler
        POST   >=> route "/resource" >=> createHandler
        PUT    >=> route "/resource" >=> replaceHandler
        PATCH  >=> route "/resource" >=> updateHandler
        DELETE >=> route "/resource" >=> deleteHandler
        HEAD   >=> route "/resource" >=> headHandler    // Same as GET but no body
        OPTIONS >=> route "/resource" >=> optionsHandler
    ]

// 5. Return 405 Method Not Allowed เมื่อ path match แต่ method ไม่ match
let resourceHandler : HttpHandler =
    route "/resource" >=>
    choose [
        GET    >=> getHandler
        POST   >=> createHandler
        // ถ้า method อื่น match path แต่ไม่ match method
        fun next ctx ->
            task {
                ctx.Response.Headers["Allow"] <- "GET, POST"
                return! (setStatusCode 405 >=> text "Method Not Allowed") next ctx
            }
    ]
```

---

## 12. Building a RESTful Router

ตัวอย่าง Router ที่สมบูรณ์สำหรับ RESTful API:

```fsharp
// Domain types
type Product = {
    Id: int
    Name: string
    Price: decimal
    Category: string
    InStock: bool
}

type CreateProductDto = {
    Name: string
    Price: decimal
    Category: string
}

type UpdateProductDto = {
    Name: string option
    Price: decimal option
    Category: string option
    InStock: bool option
}

// In-memory store
module ProductStore =
    let mutable private products: Product list = [
        { Id = 1; Name = "Laptop"; Price = 45000m; Category = "Electronics"; InStock = true }
        { Id = 2; Name = "Mouse"; Price = 800m; Category = "Electronics"; InStock = true }
        { Id = 3; Name = "Desk"; Price = 12000m; Category = "Furniture"; InStock = false }
    ]
    
    let getAll () = products
    
    let getById id = products |> List.tryFind (fun p -> p.Id = id)
    
    let getByCategory cat =
        products |> List.filter (fun p -> p.Category.ToLower() = cat.ToLower())
    
    let create (dto: CreateProductDto) =
        let id = if products.IsEmpty then 1 else (products |> List.map (fun p -> p.Id) |> List.max) + 1
        let product = { Id = id; Name = dto.Name; Price = dto.Price; Category = dto.Category; InStock = true }
        products <- products @ [product]
        product
    
    let update id (dto: UpdateProductDto) =
        match products |> List.tryFindIndex (fun p -> p.Id = id) with
        | None -> None
        | Some idx ->
            let existing = products.[idx]
            let updated = {
                existing with
                    Name = dto.Name |> Option.defaultValue existing.Name
                    Price = dto.Price |> Option.defaultValue existing.Price
                    Category = dto.Category |> Option.defaultValue existing.Category
                    InStock = dto.InStock |> Option.defaultValue existing.InStock
            }
            products <- products |> List.mapi (fun i p -> if i = idx then updated else p)
            Some updated
    
    let delete id =
        let existed = products |> List.exists (fun p -> p.Id = id)
        products <- products |> List.filter (fun p -> p.Id <> id)
        existed

// Handlers
module ProductHandlers =
    open Giraffe
    
    let list : HttpHandler =
        fun next ctx ->
            task {
                let category = ctx.TryGetQueryStringValue "category"
                let products =
                    match category with
                    | Some cat -> ProductStore.getByCategory cat
                    | None -> ProductStore.getAll()
                return! json products next ctx
            }
    
    let get (id: int) : HttpHandler =
        fun next ctx ->
            task {
                match ProductStore.getById id with
                | None -> return! (setStatusCode 404 >=> json {| error = $"Product {id} not found" |}) next ctx
                | Some p -> return! json p next ctx
            }
    
    let create : HttpHandler =
        fun next ctx ->
            task {
                let! dto = ctx.BindJsonAsync<CreateProductDto>()
                
                // Validation
                if System.String.IsNullOrWhiteSpace(dto.Name) then
                    return! (setStatusCode 400 >=> json {| errors = ["Name required"] |}) next ctx
                elif dto.Price <= 0m then
                    return! (setStatusCode 400 >=> json {| errors = ["Price must be positive"] |}) next ctx
                else
                    let product = ProductStore.create dto
                    ctx.Response.Headers["Location"] <- $"/api/products/{product.Id}"
                    return! (setStatusCode 201 >=> json product) next ctx
            }
    
    let update (id: int) : HttpHandler =
        fun next ctx ->
            task {
                let! dto = ctx.BindJsonAsync<UpdateProductDto>()
                match ProductStore.update id dto with
                | None -> return! (setStatusCode 404 >=> json {| error = $"Product {id} not found" |}) next ctx
                | Some updated -> return! json updated next ctx
            }
    
    let delete (id: int) : HttpHandler =
        fun next ctx ->
            task {
                if ProductStore.delete id then
                    return! setStatusCode 204 next ctx
                else
                    return! (setStatusCode 404 >=> json {| error = $"Product {id} not found" |}) next ctx
            }

// Router
module ProductRouter =
    open Giraffe
    open ProductHandlers
    
    let routes : HttpHandler =
        subRoute "/products" (
            choose [
                GET    >=> route ""     >=> list
                POST   >=> route ""     >=> create
                GET    >=> routef "/%i" get
                PUT    >=> routef "/%i" update
                DELETE >=> routef "/%i" delete
                
                // 405 Method Not Allowed
                route "" >=> (fun next ctx ->
                    task {
                        ctx.Response.Headers["Allow"] <- "GET, POST"
                        return! (setStatusCode 405 >=> text "Method Not Allowed") next ctx
                    })
                routef "/%i" (fun _ next ctx ->
                    task {
                        ctx.Response.Headers["Allow"] <- "GET, PUT, DELETE"
                        return! (setStatusCode 405 >=> text "Method Not Allowed") next ctx
                    })
            ]
        )

// Main router
let apiRouter : HttpHandler =
    subRoute "/api" (
        choose [
            ProductRouter.routes
            // Other resource routes...
            setStatusCode 404 >=> json {| error = "API endpoint not found" |}
        ]
    )

let webApp : HttpHandler =
    choose [
        GET >=> route "/" >=> json {|
            name = "Products API"
            version = "1.0.0"
            endpoints = ["/api/products"]
        |}
        GET >=> route "/health" >=> json {| status = "healthy" |}
        apiRouter
        setStatusCode 404 >=> json {| error = "Not Found" |}
    ]
```

---

## 13. Advanced Routing Patterns

### Versioned API Routing

```fsharp
open Giraffe

// Version ใน URL path
let v1Handler : HttpHandler =
    choose [
        route "/users" >=> json {| version = "v1"; users = [] |}
    ]

let v2Handler : HttpHandler =
    choose [
        route "/users" >=> json {| version = "v2"; users = []; meta = {| total = 0 |} |}
    ]

let versionedApiRouter : HttpHandler =
    subRoute "/api" (
        choose [
            subRoute "/v1" v1Handler
            subRoute "/v2" v2Handler
            // Default version
            subRoute "/v2" v2Handler  // redirect to latest
        ]
    )

// Version ใน header
let headerVersionRouter : HttpHandler =
    fun next ctx ->
        task {
            let version = ctx.Request.Headers["Api-Version"].ToString()
            let handler =
                match version with
                | "2" | "v2" -> v2Handler
                | _ -> v1Handler
            return! handler next ctx
        }
```

### Content Negotiation

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

let contentNegotiationHandler<'T> (data: 'T) : HttpHandler =
    fun next ctx ->
        task {
            let accept = ctx.Request.Headers["Accept"].ToString()
            
            if accept.Contains("application/json") || accept.Contains("*/*") then
                return! json data next ctx
            elif accept.Contains("text/plain") then
                return! text (sprintf "%A" data) next ctx
            elif accept.Contains("text/html") then
                return! html $"<pre>{sprintf "%A" data}</pre>" next ctx
            else
                return! (setStatusCode 406 >=> text "Not Acceptable") next ctx
        }

let webApp : HttpHandler =
    choose [
        route "/data" >=> contentNegotiationHandler {| message = "Hello"; status = "ok" |}
    ]
```

### Route with Caching

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

let cacheHandler (maxAge: int) (handler: HttpHandler) : HttpHandler =
    fun next ctx ->
        task {
            ctx.Response.Headers["Cache-Control"] <- $"public, max-age={maxAge}"
            ctx.Response.Headers["Vary"] <- "Accept-Encoding"
            return! handler next ctx
        }

let webApp : HttpHandler =
    choose [
        // Cache static data for 1 hour
        GET >=> route "/api/categories" >=> cacheHandler 3600 getCategoriesHandler
        
        // No cache for user-specific data
        GET >=> route "/api/user/profile" >=> (fun next ctx ->
            task {
                ctx.Response.Headers["Cache-Control"] <- "private, no-cache"
                return! getUserProfileHandler next ctx
            })
    ]
```

---

## สรุป

Giraffe Routing มีความยืดหยุ่นสูงและ functional approach ที่ clean:

1. **route** - exact match
2. **routef** - parameterized match with type safety
3. **routeStartsWith** - prefix match
4. **subRoute** - nested routing with prefix stripping
5. **choose** - try each handler in order
6. **HTTP method combinators** - GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS

Best practices:
- จัด route จาก specific ไป general
- ใช้ subRoute เพื่อ group related routes
- แยก route definitions ออกเป็น modules
- Handle 404 และ 405 explicitly
- ใช้ middleware สำหรับ cross-cutting concerns
