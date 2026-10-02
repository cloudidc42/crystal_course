# Part 57 - การออกแบบ REST API

## บทนำ

REST (Representational State Transfer) เป็น architectural style สำหรับการออกแบบ web APIs การออกแบบ REST API ที่ดีทำให้ API ใช้งานง่าย บำรุงรักษาง่าย และสามารถ scale ได้

---

## 1. REST Principles

REST มีหลักการ 6 ข้อ:

1. **Client-Server**: แยก client และ server ออกจากกัน
2. **Stateless**: แต่ละ request ต้องมีข้อมูลครบ ไม่อาศัย server state
3. **Cacheable**: response ควร cacheable เมื่อเป็นไปได้
4. **Uniform Interface**: interface สม่ำเสมอทั้ง API
5. **Layered System**: client ไม่รู้ว่าคุยกับ server โดยตรงหรือผ่าน proxy
6. **Code on Demand** (optional): server ส่ง code ให้ client execute ได้

```fsharp
// F# type definitions ที่สะท้อน REST principles

// Resource representation
type Resource<'T> = {
    Data: 'T
    Links: Map<string, string>  // HATEOAS links
    Meta: Map<string, obj>
}

// Stateless request context (ข้อมูลใน token เท่านั้น)
type RequestContext = {
    UserId: int
    Roles: string list
    RequestId: string
    Timestamp: System.DateTime
}
```

---

## 2. Resource Naming Conventions

```fsharp
// Good resource naming

// รูปพหูพจน์สำหรับ collections
// GET /api/users         -> list users
// GET /api/users/123     -> get user 123
// POST /api/users        -> create user
// PUT /api/users/123     -> replace user 123
// PATCH /api/users/123   -> partial update user 123
// DELETE /api/users/123  -> delete user 123

// Nested resources
// GET /api/users/123/orders           -> user's orders
// GET /api/users/123/orders/456       -> specific order
// POST /api/users/123/orders          -> create order for user
// DELETE /api/users/123/orders/456    -> delete specific order

// ใช้ lowercase และ hyphens
// GET /api/product-categories    ✓
// GET /api/productCategories     ✗ (camelCase)
// GET /api/product_categories    ✗ (underscore)

// อย่าใช้ verbs ใน URI
// POST /api/users           ✓ (create user)
// POST /api/createUser      ✗
// GET /api/users/123        ✓ (get user)
// GET /api/getUser/123      ✗

// Actions (กรณีพิเศษที่จำเป็น)
// POST /api/users/123/activate    (action on resource)
// POST /api/orders/456/cancel
// POST /api/payments/789/refund

// Version ใน URI
// /api/v1/users
// /api/v2/users

open Giraffe

// Router ที่ follow naming conventions
let usersRouter : HttpHandler =
    subRoute "/users" (
        choose [
            GET >=> route "" >=> listUsersHandler
            POST >=> route "" >=> createUserHandler
            
            GET >=> routef "/%i" getUserHandler
            PUT >=> routef "/%i" replaceUserHandler
            PATCH >=> routef "/%i" updateUserHandler
            DELETE >=> routef "/%i" deleteUserHandler
            
            // Nested resources
            GET >=> routef "/%i/orders" getUserOrdersHandler
            POST >=> routef "/%i/orders" createUserOrderHandler
            
            // Actions
            POST >=> routef "/%i/activate" activateUserHandler
            POST >=> routef "/%i/deactivate" deactivateUserHandler
        ]
    )
```

---

## 3. HTTP Methods ที่ถูกต้อง

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// GET - ดึงข้อมูล (ไม่มีผลข้างเคียง, cacheable)
let getHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            // Safe and idempotent
            match findById id with
            | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) next ctx
            | Some item -> return! json item next ctx
        }

// POST - สร้างทรัพยากรใหม่ (not idempotent)
let createHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<CreateDto>()
            let created = create dto
            
            // Return 201 Created with Location header
            ctx.Response.Headers["Location"] <- $"/api/items/{created.Id}"
            return! (setStatusCode 201 >=> json created) next ctx
        }

// PUT - แทนที่ทรัพยากรทั้งหมด (idempotent)
let replaceHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<ReplaceDto>()
            
            // PUT จะสร้างถ้าไม่มี หรือแทนที่ถ้ามีอยู่แล้ว
            match replace id dto with
            | Created item -> return! (setStatusCode 201 >=> json item) next ctx
            | Updated item -> return! json item next ctx
        }

// PATCH - อัปเดตบางส่วน (not necessarily idempotent)
let patchHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            let! patch = ctx.BindJsonAsync<PatchDto>()
            
            match applyPatch id patch with
            | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) next ctx
            | Some updated -> return! json updated next ctx
        }

// DELETE - ลบทรัพยากร (idempotent)
let deleteHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            // ลบแล้วก็ลบแล้ว - การลบซ้ำควร return 204 หรือ 404
            delete id
            return! setStatusCode 204 next ctx
        }

// HEAD - เหมือน GET แต่ไม่มี body
let headHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match findById id with
            | None ->
                return! setStatusCode 404 next ctx
            | Some item ->
                ctx.Response.Headers["ETag"] <- $"\"{item.GetHashCode()}\""
                ctx.Response.Headers["Last-Modified"] <- item.UpdatedAt.ToString("R")
                return! setStatusCode 200 next ctx
        }

// OPTIONS - return supported methods
let optionsHandler : HttpHandler =
    fun next ctx ->
        task {
            ctx.Response.Headers["Allow"] <- "GET, POST, PUT, PATCH, DELETE, HEAD, OPTIONS"
            ctx.Response.Headers["Accept"] <- "application/json"
            return! setStatusCode 200 next ctx
        }
```

---

## 4. Status Codes ที่ถูกต้อง

```fsharp
// 2xx Success
// 200 OK - GET, PUT, PATCH requests ที่สำเร็จ
// 201 Created - POST ที่สร้าง resource ใหม่
// 202 Accepted - async operations
// 204 No Content - DELETE สำเร็จ, หรือ PUT/PATCH ที่ไม่ return body

// 3xx Redirection  
// 301 Moved Permanently - permanent redirect
// 302 Found - temporary redirect
// 304 Not Modified - conditional GET

// 4xx Client Errors
// 400 Bad Request - invalid request
// 401 Unauthorized - not authenticated
// 403 Forbidden - authenticated but not authorized
// 404 Not Found - resource not found
// 405 Method Not Allowed - HTTP method not supported
// 406 Not Acceptable - Accept header not supported
// 409 Conflict - resource conflict
// 410 Gone - resource permanently deleted
// 415 Unsupported Media Type - Content-Type not supported
// 422 Unprocessable Entity - validation errors
// 429 Too Many Requests - rate limit exceeded

// 5xx Server Errors
// 500 Internal Server Error - unexpected error
// 502 Bad Gateway - upstream service error
// 503 Service Unavailable - server overloaded/maintenance
// 504 Gateway Timeout - upstream timeout

open Giraffe

// Status code helpers
module Http =
    let ok = setStatusCode 200
    let created = setStatusCode 201
    let accepted = setStatusCode 202
    let noContent = setStatusCode 204
    let movedPermanently = setStatusCode 301
    let found = setStatusCode 302
    let notModified = setStatusCode 304
    let badRequest = setStatusCode 400
    let unauthorized = setStatusCode 401
    let forbidden = setStatusCode 403
    let notFound = setStatusCode 404
    let methodNotAllowed = setStatusCode 405
    let conflict = setStatusCode 409
    let gone = setStatusCode 410
    let unprocessableEntity = setStatusCode 422
    let tooManyRequests = setStatusCode 429
    let internalError = setStatusCode 500
    let serviceUnavailable = setStatusCode 503
```

---

## 5. Request/Response DTOs

```fsharp
open System
open System.Text.Json.Serialization

// Input DTOs (what clients send)
type CreateUserRequest = {
    Username: string
    Email: string
    Password: string
    FirstName: string
    LastName: string
    DateOfBirth: DateOnly option
    Phone: string option
}

type UpdateUserRequest = {
    FirstName: string option
    LastName: string option
    Phone: string option
    Bio: string option
}

// Output DTOs (what server returns)
// ไม่ return internal fields เช่น PasswordHash
type UserResponse = {
    Id: int
    Username: string
    Email: string
    FirstName: string
    LastName: string
    FullName: string
    DateOfBirth: DateOnly option
    Phone: string option
    CreatedAt: DateTime
    UpdatedAt: DateTime
}

// List response with pagination
type PagedResponse<'T> = {
    Data: 'T list
    Pagination: PaginationInfo
}

and PaginationInfo = {
    Page: int
    PageSize: int
    TotalItems: int
    TotalPages: int
    HasPreviousPage: bool
    HasNextPage: bool
}

// DTO mapping
let toUserResponse (user: Domain.User) : UserResponse =
    {
        Id = user.Id
        Username = user.Username
        Email = user.Email
        FirstName = user.FirstName
        LastName = user.LastName
        FullName = $"{user.FirstName} {user.LastName}"
        DateOfBirth = user.DateOfBirth
        Phone = user.Phone
        CreatedAt = user.CreatedAt
        UpdatedAt = user.UpdatedAt
    }

// Wrapper สำหรับ single resource
type ResourceResponse<'T> = {
    Data: 'T
    [<JsonIgnore(Condition = JsonIgnoreCondition.WhenWritingNull)>]
    Links: Map<string, string> option
}

// Helper สร้าง response
let okResponse data =
    { Data = data; Links = None }

let okResponseWithLinks data links =
    { Data = data; Links = Some links }
```

---

## 6. Validation

```fsharp
open System
open System.Text.RegularExpressions

// Validation result type
type ValidationError = {
    Field: string
    Code: string
    Message: string
}

type ValidationResult<'T> =
    | Valid of 'T
    | Invalid of ValidationError list

// Validation functions
module Validate =
    let required (field: string) (value: string) =
        if String.IsNullOrWhiteSpace(value) then
            Some { Field = field; Code = "REQUIRED"; Message = $"{field} is required" }
        else None
    
    let minLength (field: string) (min: int) (value: string) =
        if value.Length < min then
            Some { Field = field; Code = "MIN_LENGTH"; Message = $"{field} must be at least {min} characters" }
        else None
    
    let maxLength (field: string) (max: int) (value: string) =
        if value.Length > max then
            Some { Field = field; Code = "MAX_LENGTH"; Message = $"{field} must be at most {max} characters" }
        else None
    
    let email (field: string) (value: string) =
        let emailRegex = Regex(@"^[^@\s]+@[^@\s]+\.[^@\s]+$")
        if not (emailRegex.IsMatch(value)) then
            Some { Field = field; Code = "INVALID_EMAIL"; Message = $"{field} must be a valid email address" }
        else None
    
    let minValue (field: string) (min: 'T) (value: 'T) (compare: 'T -> 'T -> int) =
        if compare value min < 0 then
            Some { Field = field; Code = "MIN_VALUE"; Message = $"{field} must be at least {min}" }
        else None
    
    let maxValue (field: string) (max: 'T) (value: 'T) (compare: 'T -> 'T -> int) =
        if compare value max > 0 then
            Some { Field = field; Code = "MAX_VALUE"; Message = $"{field} must be at most {max}" }
        else None
    
    let combine (validators: 'T option list) =
        validators |> List.choose id

// User validation
let validateCreateUser (dto: CreateUserRequest) =
    let errors =
        Validate.combine [
            Validate.required "username" dto.Username
            Validate.minLength "username" 3 dto.Username
            Validate.maxLength "username" 50 dto.Username
            Validate.required "email" dto.Email
            Validate.email "email" dto.Email
            Validate.required "password" dto.Password
            Validate.minLength "password" 8 dto.Password
            Validate.required "firstName" dto.FirstName
            Validate.required "lastName" dto.LastName
        ]
    
    if errors.IsEmpty then Valid dto
    else Invalid errors

// Handler with validation
open Giraffe

let createUserHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<CreateUserRequest>()
            
            match validateCreateUser dto with
            | Invalid errors ->
                return! (setStatusCode 422 >=> json {|
                    error = "Validation failed"
                    errors = errors
                |}) next ctx
            | Valid validDto ->
                // Process...
                let user = createUser validDto
                return! (setStatusCode 201 >=> json (toUserResponse user)) next ctx
        }
```

---

## 7. Pagination

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// Pagination parameters
type PaginationParams = {
    Page: int
    PageSize: int
}
with
    static member Default = { Page = 1; PageSize = 20 }
    static member MaxPageSize = 100

let parsePaginationParams (ctx: HttpContext) =
    let page = 
        ctx.TryGetQueryStringValue "page" 
        |> Option.bind (fun s -> 
            match System.Int32.TryParse(s) with 
            | true, n -> Some n 
            | _ -> None)
        |> Option.defaultValue 1
    
    let pageSize = 
        ctx.TryGetQueryStringValue "pageSize" 
        |> Option.bind (fun s -> 
            match System.Int32.TryParse(s) with 
            | true, n -> Some n 
            | _ -> None)
        |> Option.defaultValue 20
    
    {
        Page = max 1 page
        PageSize = min PaginationParams.MaxPageSize (max 1 pageSize)
    }

// Paginate a list
let paginate (params: PaginationParams) (items: 'T list) =
    let total = items.Length
    let skip = (params.Page - 1) * params.PageSize
    let data = items |> List.skip (min skip total) |> List.truncate params.PageSize
    let totalPages = (total + params.PageSize - 1) / params.PageSize
    
    {
        Data = data
        Pagination = {
            Page = params.Page
            PageSize = params.PageSize
            TotalItems = total
            TotalPages = totalPages
            HasPreviousPage = params.Page > 1
            HasNextPage = params.Page < totalPages
        }
    }

// Cursor-based pagination (สำหรับ large datasets)
type CursorPaginationParams = {
    Cursor: string option  // Last seen ID
    Limit: int
    Direction: string  // "next" or "prev"
}

type CursorPagedResult<'T> = {
    Data: 'T list
    NextCursor: string option
    PreviousCursor: string option
    HasMore: bool
}

// Link header pagination (RFC 5988)
let addPaginationHeaders (ctx: HttpContext) (result: PagedResponse<'T>) =
    let baseUrl = $"{ctx.Request.Scheme}://{ctx.Request.Host}{ctx.Request.Path}"
    let buildUrl page = $"{baseUrl}?page={page}&pageSize={result.Pagination.PageSize}"
    
    let links = ResizeArray<string>()
    links.Add($"<{buildUrl 1}>; rel=\"first\"")
    links.Add($"<{buildUrl result.Pagination.TotalPages}>; rel=\"last\"")
    
    if result.Pagination.HasPreviousPage then
        links.Add($"<{buildUrl (result.Pagination.Page - 1)}>; rel=\"prev\"")
    
    if result.Pagination.HasNextPage then
        links.Add($"<{buildUrl (result.Pagination.Page + 1)}>; rel=\"next\"")
    
    ctx.Response.Headers["Link"] <- String.concat ", " links
    ctx.Response.Headers["X-Total-Count"] <- string result.Pagination.TotalItems

// Paginated handler
let listUsersHandler : HttpHandler =
    fun next ctx ->
        task {
            let params' = parsePaginationParams ctx
            let allUsers = getAllUsers()
            let result = paginate params' allUsers
            
            addPaginationHeaders ctx result
            
            return! json result next ctx
        }
```

---

## 8. Filtering and Sorting

```fsharp
open System
open Microsoft.AspNetCore.Http

// Filter parameters
type UserFilter = {
    Search: string option      // Full-text search
    Status: string option      // active, inactive
    Role: string option        // admin, user, etc.
    CreatedAfter: DateTime option
    CreatedBefore: DateTime option
    MinAge: int option
    MaxAge: int option
}

let parseUserFilter (ctx: HttpContext) =
    {
        Search = ctx.TryGetQueryStringValue "q"
        Status = ctx.TryGetQueryStringValue "status"
        Role = ctx.TryGetQueryStringValue "role"
        CreatedAfter = 
            ctx.TryGetQueryStringValue "createdAfter"
            |> Option.bind (fun s -> 
                match DateTime.TryParse(s) with
                | true, d -> Some d
                | _ -> None)
        CreatedBefore = 
            ctx.TryGetQueryStringValue "createdBefore"
            |> Option.bind (fun s -> 
                match DateTime.TryParse(s) with
                | true, d -> Some d
                | _ -> None)
        MinAge = ctx.TryGetQueryStringValue "minAge" |> Option.bind (fun s -> 
            match Int32.TryParse(s) with true, n -> Some n | _ -> None)
        MaxAge = ctx.TryGetQueryStringValue "maxAge" |> Option.bind (fun s -> 
            match Int32.TryParse(s) with true, n -> Some n | _ -> None)
    }

// Apply filter
let applyUserFilter (filter: UserFilter) (users: User list) =
    users
    |> List.filter (fun u ->
        match filter.Search with
        | Some q -> 
            u.Username.Contains(q, StringComparison.OrdinalIgnoreCase) ||
            u.Email.Contains(q, StringComparison.OrdinalIgnoreCase) ||
            u.FirstName.Contains(q, StringComparison.OrdinalIgnoreCase)
        | None -> true)
    |> List.filter (fun u ->
        match filter.Status with
        | Some "active" -> u.IsActive
        | Some "inactive" -> not u.IsActive
        | _ -> true)
    |> List.filter (fun u ->
        match filter.CreatedAfter with
        | Some d -> u.CreatedAt >= d
        | None -> true)
    |> List.filter (fun u ->
        match filter.CreatedBefore with
        | Some d -> u.CreatedAt <= d
        | None -> true)

// Sort parameters
type SortParams = {
    SortBy: string
    Order: string  // "asc" or "desc"
}

let parseSortParams (ctx: HttpContext) (defaultField: string) =
    {
        SortBy = ctx.TryGetQueryStringValue "sortBy" |> Option.defaultValue defaultField
        Order = ctx.TryGetQueryStringValue "order" |> Option.defaultValue "asc"
    }

// Apply sort
let applyUserSort (sort: SortParams) (users: User list) =
    let sorted =
        match sort.SortBy.ToLower() with
        | "username" -> users |> List.sortBy (fun u -> u.Username)
        | "email" -> users |> List.sortBy (fun u -> u.Email)
        | "createdat" -> users |> List.sortBy (fun u -> u.CreatedAt)
        | "firstname" -> users |> List.sortBy (fun u -> u.FirstName)
        | _ -> users |> List.sortBy (fun u -> u.Id)
    
    if sort.Order.ToLower() = "desc" then
        sorted |> List.rev
    else
        sorted

// Full-featured list handler
let listUsersFullHandler : HttpHandler =
    fun next ctx ->
        task {
            let filter = parseUserFilter ctx
            let sort = parseSortParams ctx "id"
            let pagination = parsePaginationParams ctx
            
            let result =
                getAllUsers()
                |> applyUserFilter filter
                |> applyUserSort sort
                |> paginate pagination
            
            addPaginationHeaders ctx result
            
            return! json result next ctx
        }
```

---

## 9. HATEOAS Concept

```fsharp
// HATEOAS - Hypermedia as the Engine of Application State
// เพิ่ม links ใน response เพื่อบอก client ว่าสามารถทำอะไรต่อได้

type Link = {
    Href: string
    Method: string
    Title: string option
}

type HateoasResponse<'T> = {
    Data: 'T
    Links: Map<string, Link>
}

// Helper สร้าง links
let userLinks (userId: int) (isActive: bool) =
    Map.ofList [
        "self", { Href = $"/api/users/{userId}"; Method = "GET"; Title = Some "Get user" }
        "update", { Href = $"/api/users/{userId}"; Method = "PUT"; Title = Some "Update user" }
        "delete", { Href = $"/api/users/{userId}"; Method = "DELETE"; Title = Some "Delete user" }
        "orders", { Href = $"/api/users/{userId}/orders"; Method = "GET"; Title = Some "Get user orders" }
        if isActive then
            "deactivate", { Href = $"/api/users/{userId}/deactivate"; Method = "POST"; Title = Some "Deactivate user" }
        else
            "activate", { Href = $"/api/users/{userId}/activate"; Method = "POST"; Title = Some "Activate user" }
    ]

// HATEOAS handler
let getUserHateoasHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match findUserById id with
            | None -> return! (setStatusCode 404 >=> json {| error = "Not found" |}) next ctx
            | Some user ->
                let response = {
                    Data = toUserResponse user
                    Links = userLinks user.Id user.IsActive
                }
                return! json response next ctx
        }

// Collection HATEOAS
type CollectionResponse<'T> = {
    Data: 'T list
    Links: Map<string, Link>
    Meta: Map<string, obj>
}

let getUsersHateoasHandler : HttpHandler =
    fun next ctx ->
        task {
            let users = getAllUsers()
            let response = {
                Data = users |> List.map toUserResponse
                Links = Map.ofList [
                    "self", { Href = "/api/users"; Method = "GET"; Title = Some "Current page" }
                    "create", { Href = "/api/users"; Method = "POST"; Title = Some "Create user" }
                ]
                Meta = Map.ofList [
                    "total", users.Length :> obj
                ]
            }
            return! json response next ctx
        }
```

---

## 10. API Versioning

```fsharp
open Giraffe

// Strategy 1: URL path versioning (แนะนำ)
let v1Routes : HttpHandler =
    subRoute "/api/v1" (
        choose [
            GET >=> route "/users" >=> V1.getUsersHandler
            POST >=> route "/users" >=> V1.createUserHandler
        ]
    )

let v2Routes : HttpHandler =
    subRoute "/api/v2" (
        choose [
            GET >=> route "/users" >=> V2.getUsersHandler  // Enhanced response
            POST >=> route "/users" >=> V2.createUserHandler  // More fields
        ]
    )

// Strategy 2: Header versioning
let headerVersionRouter : HttpHandler =
    fun next ctx ->
        task {
            let version = 
                ctx.Request.Headers["Api-Version"].ToString()
                |> fun s -> if System.String.IsNullOrEmpty(s) then "1" else s
            
            let handler =
                match version with
                | "2" | "v2" -> v2Handler
                | _ -> v1Handler
            
            return! handler next ctx
        }

// Strategy 3: Query string versioning
let queryVersionRouter : HttpHandler =
    fun next ctx ->
        task {
            let version = ctx.TryGetQueryStringValue "version" |> Option.defaultValue "1"
            let handler = match version with "2" -> v2Handler | _ -> v1Handler
            return! handler next ctx
        }

// Strategy 4: Accept header versioning (Content negotiation)
let contentNegotiationRouter : HttpHandler =
    fun next ctx ->
        task {
            let accept = ctx.Request.Headers["Accept"].ToString()
            let handler =
                if accept.Contains("application/vnd.myapi.v2+json") then v2Handler
                else v1Handler
            return! handler next ctx
        }

// Version deprecation notice
let deprecationMiddleware (deprecatedVersion: string) (sunsetDate: string) : HttpHandler =
    fun next ctx ->
        task {
            ctx.Response.Headers["Deprecation"] <- "true"
            ctx.Response.Headers["Sunset"] <- sunsetDate
            ctx.Response.Headers["Link"] <- "</api/v2/users>; rel=\"successor-version\""
            return! next ctx
        }
```

---

## 11. Error Response Format (RFC 7807)

```fsharp
open System.Text.Json.Serialization

// RFC 7807 Problem Details format
type ProblemDetails = {
    [<JsonPropertyName("type")>]
    Type: string  // URI reference

    [<JsonPropertyName("title")>]
    Title: string  // Short description

    [<JsonPropertyName("status")>]
    Status: int  // HTTP status code

    [<JsonPropertyName("detail")>]
    Detail: string option  // Human-readable explanation

    [<JsonPropertyName("instance")>]
    Instance: string option  // URI of specific occurrence

    [<JsonPropertyName("errors")>]
    Errors: Map<string, string list> option  // Validation errors
}

// Create problem details
let createProblem status type' title detail instance =
    {
        Type = type'
        Title = title
        Status = status
        Detail = detail
        Instance = instance
        Errors = None
    }

let notFoundProblem (resourceType: string) (id: obj) =
    createProblem 404
        "https://api.example.com/problems/not-found"
        "Resource Not Found"
        (Some $"The {resourceType} with ID '{id}' was not found")
        None

let validationProblem (errors: Map<string, string list>) =
    {
        Type = "https://api.example.com/problems/validation-error"
        Title = "Validation Error"
        Status = 422
        Detail = Some "One or more validation errors occurred"
        Instance = None
        Errors = Some errors
    }

let unauthorizedProblem () =
    createProblem 401
        "https://api.example.com/problems/unauthorized"
        "Unauthorized"
        (Some "Authentication is required to access this resource")
        None

// Response helpers
open Giraffe

let problemResponse (problem: ProblemDetails) : HttpHandler =
    fun next ctx ->
        task {
            ctx.Response.ContentType <- "application/problem+json"
            return! (setStatusCode problem.Status >=> json problem) next ctx
        }

// Usage
let userHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match findUserById id with
            | None ->
                return! problemResponse (notFoundProblem "user" id) next ctx
            | Some user ->
                return! json (toUserResponse user) next ctx
        }
```

---

## 12. OpenAPI/Swagger Documentation

```bash
dotnet add package Swashbuckle.AspNetCore
```

```fsharp
open Microsoft.Extensions.DependencyInjection
open Microsoft.OpenApi.Models
open Swashbuckle.AspNetCore.SwaggerGen

// Configure Swagger
let configureSwagger (services: IServiceCollection) =
    services.AddEndpointsApiExplorer() |> ignore
    
    services.AddSwaggerGen(fun opts ->
        opts.SwaggerDoc("v1", OpenApiInfo(
            Title = "My API",
            Version = "v1",
            Description = "F# REST API Documentation",
            Contact = OpenApiContact(
                Name = "API Team",
                Email = "api@example.com",
                Url = System.Uri("https://example.com")
            ),
            License = OpenApiLicense(
                Name = "MIT",
                Url = System.Uri("https://opensource.org/licenses/MIT")
            )
        ))
        
        // Add JWT authentication to Swagger
        let securityScheme = OpenApiSecurityScheme(
            Name = "Authorization",
            Description = "Enter 'Bearer {token}'",
            In = ParameterLocation.Header,
            Type = SecuritySchemeType.ApiKey,
            Scheme = "Bearer"
        )
        opts.AddSecurityDefinition("Bearer", securityScheme)
        
        let securityReq = OpenApiSecurityRequirement()
        securityReq.Add(
            OpenApiSecurityScheme(
                Reference = OpenApiReference(
                    Type = ReferenceType.SecurityScheme,
                    Id = "Bearer"
                )
            ),
            [||])
        opts.AddSecurityRequirement(securityReq)
        
        // Include XML comments
        let xmlFile = "MyApi.xml"
        let xmlPath = System.IO.Path.Combine(System.AppContext.BaseDirectory, xmlFile)
        if System.IO.File.Exists(xmlPath) then
            opts.IncludeXmlComments(xmlPath))
    |> ignore

// Add Swagger middleware
let configureSwaggerApp (app: Microsoft.AspNetCore.Builder.IApplicationBuilder) =
    app.UseSwagger() |> ignore
    app.UseSwaggerUI(fun opts ->
        opts.SwaggerEndpoint("/swagger/v1/swagger.json", "My API v1")
        opts.RoutePrefix <- "docs")
    |> ignore
```

---

## 13. Rate Limiting

```fsharp
open System.Collections.Concurrent
open System.Threading
open Microsoft.AspNetCore.Http

// Simple in-memory rate limiter
type RateLimiter(maxRequests: int, windowSeconds: int) =
    let requests = ConcurrentDictionary<string, ConcurrentQueue<System.DateTime>>()
    
    member _.IsAllowed(key: string) =
        let window = System.DateTime.UtcNow.AddSeconds(-float windowSeconds)
        let queue = requests.GetOrAdd(key, fun _ -> ConcurrentQueue<System.DateTime>())
        
        // Remove old requests
        let mutable item = System.DateTime.MinValue
        while not queue.IsEmpty && queue.TryPeek(&item) && item < window do
            queue.TryDequeue(&item) |> ignore
        
        if queue.Count < maxRequests then
            queue.Enqueue(System.DateTime.UtcNow)
            true
        else
            false
    
    member _.RemainingRequests(key: string) =
        let window = System.DateTime.UtcNow.AddSeconds(-float windowSeconds)
        match requests.TryGetValue(key) with
        | true, queue ->
            let recent = queue |> Seq.filter (fun t -> t >= window) |> Seq.length
            max 0 (maxRequests - recent)
        | _ -> maxRequests

// Rate limit middleware
open Giraffe

let createRateLimitMiddleware (limiter: RateLimiter) (keyExtractor: HttpContext -> string) : HttpHandler =
    fun next ctx ->
        task {
            let key = keyExtractor ctx
            
            if limiter.IsAllowed(key) then
                ctx.Response.Headers["X-RateLimit-Limit"] <- string 100
                ctx.Response.Headers["X-RateLimit-Remaining"] <- string (limiter.RemainingRequests(key))
                ctx.Response.Headers["X-RateLimit-Window"] <- "60"
                return! next ctx
            else
                ctx.Response.Headers["Retry-After"] <- "60"
                ctx.Response.Headers["X-RateLimit-Limit"] <- string 100
                ctx.Response.Headers["X-RateLimit-Remaining"] <- "0"
                return! problemResponse {
                    Type = "https://api.example.com/problems/rate-limited"
                    Title = "Rate Limit Exceeded"
                    Status = 429
                    Detail = Some "Too many requests. Please wait before retrying."
                    Instance = None
                    Errors = None
                } next ctx
        }

let limiter = RateLimiter(100, 60)  // 100 requests per 60 seconds

// Rate limit by IP
let ipRateLimiter = createRateLimitMiddleware limiter (fun ctx ->
    ctx.Connection.RemoteIpAddress.ToString())

// Rate limit by user ID
let userRateLimiter = createRateLimitMiddleware limiter (fun ctx ->
    ctx.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value
    |> Option.ofObj
    |> Option.defaultValue (ctx.Connection.RemoteIpAddress.ToString()))
```

---

## 14. CORS Configuration

```fsharp
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Cors.Infrastructure

// Configure CORS
let configureCors (services: IServiceCollection) =
    services.AddCors(fun opts ->
        // Development policy
        opts.AddPolicy("Development", fun policy ->
            policy
                .AllowAnyOrigin()
                .AllowAnyMethod()
                .AllowAnyHeader()
            |> ignore)
        
        // Production policy
        opts.AddPolicy("Production", fun policy ->
            policy
                .WithOrigins(
                    "https://app.example.com",
                    "https://www.example.com")
                .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE")
                .WithHeaders(
                    "Content-Type",
                    "Authorization",
                    "X-Request-Id",
                    "X-API-Version")
                .AllowCredentials()
                .SetPreflightMaxAge(System.TimeSpan.FromHours(1))
            |> ignore))
    |> ignore

// Use CORS in pipeline
let configureCorsApp (app: IApplicationBuilder) (env: IWebHostEnvironment) =
    let policyName = if env.IsDevelopment() then "Development" else "Production"
    app.UseCors(policyName) |> ignore

// Dynamic CORS with allowed origins from config
let configureDynamicCors (services: IServiceCollection) (allowedOrigins: string[]) =
    services.AddCors(fun opts ->
        opts.AddPolicy("Dynamic", fun policy ->
            if allowedOrigins |> Array.contains "*" then
                policy.AllowAnyOrigin() |> ignore
            else
                policy.WithOrigins(allowedOrigins) |> ignore
            
            policy
                .WithMethods("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS")
                .WithHeaders("Content-Type", "Authorization")
                .AllowCredentials()
            |> ignore))
    |> ignore
```

---

## 15. Complete API Example

```fsharp
// Complete F# REST API with all best practices

// Domain types
type ProductCategory = Electronics | Clothing | Food | Books | Sports

type Product = {
    Id: int
    Name: string
    Description: string
    Price: decimal
    Category: ProductCategory
    Stock: int
    IsActive: bool
    CreatedAt: System.DateTime
    UpdatedAt: System.DateTime
}

// DTOs
type CreateProductRequest = {
    Name: string
    Description: string
    Price: decimal
    Category: string
    Stock: int
}

type UpdateProductRequest = {
    Name: string option
    Description: string option
    Price: decimal option
    Stock: int option
}

type ProductResponse = {
    Id: int
    Name: string
    Description: string
    Price: decimal
    Category: string
    Stock: int
    InStock: bool
    Links: Map<string, string>
}

// Mapping
let toProductResponse (product: Product) =
    {
        Id = product.Id
        Name = product.Name
        Description = product.Description
        Price = product.Price
        Category = string product.Category
        Stock = product.Stock
        InStock = product.Stock > 0
        Links = Map.ofList [
            "self", $"/api/v1/products/{product.Id}"
            "update", $"/api/v1/products/{product.Id}"
            "delete", $"/api/v1/products/{product.Id}"
        ]
    }

// Handlers
open Giraffe

let listProductsHandler : HttpHandler =
    fun next ctx ->
        task {
            let filter = parseProductFilter ctx
            let sort = parseSortParams ctx "name"
            let pagination = parsePaginationParams ctx
            
            let result =
                ProductRepo.getAll()
                |> applyFilter filter
                |> applySort sort
                |> List.map toProductResponse
                |> paginate pagination
            
            addPaginationHeaders ctx result
            ctx.Response.Headers["X-Total-Count"] <- string result.Pagination.TotalItems
            
            return! json result next ctx
        }

let getProductHandler (id: int) : HttpHandler =
    fun next ctx ->
        task {
            match ProductRepo.getById id with
            | None ->
                return! problemResponse (notFoundProblem "product" id) next ctx
            | Some product ->
                let etag = $"\"{product.GetHashCode()}\""
                let requestEtag = ctx.Request.Headers["If-None-Match"].ToString()
                
                if requestEtag = etag then
                    return! setStatusCode 304 next ctx
                else
                    ctx.Response.Headers["ETag"] <- etag
                    ctx.Response.Headers["Last-Modified"] <- product.UpdatedAt.ToString("R")
                    ctx.Response.Headers["Cache-Control"] <- "private, max-age=60"
                    return! json (toProductResponse product) next ctx
        }

let createProductHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<CreateProductRequest>()
            
            match validateCreateProduct dto with
            | Invalid errors ->
                let errorMap = errors |> List.groupBy (fun e -> e.Field) |> Map.ofList |> Map.map (fun _ v -> v |> List.map (fun e -> e.Message))
                return! problemResponse (validationProblem errorMap) next ctx
            | Valid _ ->
                let product = ProductRepo.create dto
                ctx.Response.Headers["Location"] <- $"/api/v1/products/{product.Id}"
                return! (setStatusCode 201 >=> json (toProductResponse product)) next ctx
        }

// Router
let productRouter : HttpHandler =
    ipRateLimiter >=>
    subRoute "/api/v1/products" (
        choose [
            GET >=> route "" >=> listProductsHandler
            POST >=> route "" >=> (requireAuthentication >=> createProductHandler)
            GET >=> routef "/%i" getProductHandler
            PUT >=> routef "/%i" (requireAuthentication >=> updateProductHandler)
            PATCH >=> routef "/%i" (requireAuthentication >=> patchProductHandler)
            DELETE >=> routef "/%i" (requireAuthentication >=> requireRole "admin" >=> deleteProductHandler)
        ]
    )
```

---

## สรุป

การออกแบบ REST API ที่ดีต้องคำนึงถึง:

1. **Naming conventions**: plural nouns, lowercase, hyphens
2. **HTTP methods**: ใช้ method ที่ถูกต้องตาม semantics
3. **Status codes**: ส่ง status code ที่สื่อความหมายถูกต้อง
4. **DTOs**: แยก input/output DTOs ออกจาก domain models
5. **Validation**: validate input ก่อน process เสมอ
6. **Pagination**: ทุก collection endpoint ควรมี pagination
7. **Filtering/Sorting**: ใช้ query parameters
8. **Error format**: ใช้ RFC 7807 Problem Details
9. **Versioning**: version ตั้งแต่ต้น
10. **Rate limiting**: ป้องกัน abuse
11. **CORS**: configure ให้เหมาะสมกับ environment
12. **Documentation**: Swagger/OpenAPI
