# Part 56 - การยืนยันตัวตนและการอนุญาต (Authentication and Authorization)

## บทนำ

Authentication (การยืนยันตัวตน) และ Authorization (การอนุญาต) เป็นส่วนสำคัญของทุก web application ใน F# สามารถใช้ ASP.NET Core authentication/authorization infrastructure ได้เต็มที่

---

## 1. JWT Authentication Setup

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
dotnet add package System.IdentityModel.Tokens.Jwt
```

### Configuration

```fsharp
// Program.fs
open Microsoft.AspNetCore.Authentication.JwtBearer
open Microsoft.IdentityModel.Tokens
open System.Text
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Configuration

let configureJwtAuth (services: IServiceCollection) (config: IConfiguration) =
    let jwtSection = config.GetSection("Jwt")
    let secret = jwtSection["Secret"]
    let issuer = jwtSection["Issuer"]
    let audience = jwtSection["Audience"]
    
    let key = SymmetricSecurityKey(Encoding.UTF8.GetBytes(secret))
    
    services
        .AddAuthentication(fun opts ->
            opts.DefaultAuthenticateScheme <- JwtBearerDefaults.AuthenticationScheme
            opts.DefaultChallengeScheme <- JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(fun opts ->
            opts.TokenValidationParameters <- TokenValidationParameters(
                ValidateIssuer = true,
                ValidIssuer = issuer,
                ValidateAudience = true,
                ValidAudience = audience,
                ValidateLifetime = true,
                ValidateIssuerSigningKey = true,
                IssuerSigningKey = key,
                ClockSkew = System.TimeSpan.Zero  // No tolerance for expired tokens
            )
            
            // Events for debugging
            opts.Events <- JwtBearerEvents(
                OnAuthenticationFailed = (fun ctx ->
                    let logger = ctx.HttpContext.GetService<Microsoft.Extensions.Logging.ILogger>()
                    logger.LogWarning($"JWT auth failed: {ctx.Exception.Message}")
                    System.Threading.Tasks.Task.CompletedTask),
                OnTokenValidated = (fun ctx ->
                    // Called when token is successfully validated
                    System.Threading.Tasks.Task.CompletedTask)
            ))
    |> ignore

// appsettings.json
(*
{
  "Jwt": {
    "Secret": "your-256-bit-secret-key-here-must-be-at-least-32-chars",
    "Issuer": "my-api",
    "Audience": "my-api-users",
    "ExpirationHours": 8
  }
}
*)
```

---

## 2. Creating JWT Tokens

```fsharp
// JwtService.fs
module JwtService

open System
open System.Security.Claims
open Microsoft.IdentityModel.Tokens
open System.IdentityModel.Tokens.Jwt
open System.Text
open Microsoft.Extensions.Configuration

// JWT Configuration
type JwtConfig = {
    Secret: string
    Issuer: string
    Audience: string
    ExpirationHours: int
}

// Token payload
type TokenPayload = {
    UserId: int
    Username: string
    Email: string
    Roles: string list
    Claims: (string * string) list
}

// Token response
type TokenResult = {
    AccessToken: string
    RefreshToken: string
    ExpiresAt: DateTime
    TokenType: string
}

// Create access token
let createAccessToken (config: JwtConfig) (payload: TokenPayload) =
    let key = SymmetricSecurityKey(Encoding.UTF8.GetBytes(config.Secret))
    let credentials = SigningCredentials(key, SecurityAlgorithms.HmacSha256)
    let expiresAt = DateTime.UtcNow.AddHours(float config.ExpirationHours)
    
    let claims = [
        yield Claim(JwtRegisteredClaimNames.Sub, string payload.UserId)
        yield Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString())
        yield Claim(JwtRegisteredClaimNames.Iat, DateTimeOffset.UtcNow.ToUnixTimeSeconds() |> string)
        yield Claim(ClaimTypes.Name, payload.Username)
        yield Claim(ClaimTypes.Email, payload.Email)
        yield! payload.Roles |> List.map (fun r -> Claim(ClaimTypes.Role, r))
        yield! payload.Claims |> List.map (fun (k, v) -> Claim(k, v))
    ]
    
    let token = JwtSecurityToken(
        issuer = config.Issuer,
        audience = config.Audience,
        claims = claims,
        notBefore = DateTime.UtcNow,
        expires = expiresAt,
        signingCredentials = credentials
    )
    
    JwtSecurityTokenHandler().WriteToken(token), expiresAt

// Create refresh token
let createRefreshToken () =
    let bytes = Array.zeroCreate<byte> 64
    use rng = System.Security.Cryptography.RandomNumberGenerator.Create()
    rng.GetBytes(bytes)
    Convert.ToBase64String(bytes)

// Create full token response
let createTokens (config: JwtConfig) (payload: TokenPayload) =
    let accessToken, expiresAt = createAccessToken config payload
    let refreshToken = createRefreshToken()
    
    {
        AccessToken = accessToken
        RefreshToken = refreshToken
        ExpiresAt = expiresAt
        TokenType = "Bearer"
    }

// Decode token without validation (for reading claims)
let readTokenClaims (token: string) =
    try
        let handler = JwtSecurityTokenHandler()
        let jwtToken = handler.ReadJwtToken(token)
        Ok jwtToken.Claims
    with ex ->
        Error $"Invalid token: {ex.Message}"
```

---

## 3. Validating JWT Tokens

```fsharp
// TokenValidator.fs
module TokenValidator

open System
open System.Security.Claims
open Microsoft.IdentityModel.Tokens
open System.IdentityModel.Tokens.Jwt
open System.Text

type ValidationResult =
    | Valid of ClaimsPrincipal
    | Expired
    | Invalid of string

let validateToken (config: JwtService.JwtConfig) (token: string) =
    try
        let handler = JwtSecurityTokenHandler()
        let key = SymmetricSecurityKey(Encoding.UTF8.GetBytes(config.Secret))
        
        let validationParams = TokenValidationParameters(
            ValidateIssuer = true,
            ValidIssuer = config.Issuer,
            ValidateAudience = true,
            ValidAudience = config.Audience,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            IssuerSigningKey = key,
            ClockSkew = TimeSpan.Zero
        )
        
        let mutable securityToken: SecurityToken = null
        let principal = handler.ValidateToken(token, validationParams, &securityToken)
        Valid principal
    with
    | :? SecurityTokenExpiredException ->
        Expired
    | :? SecurityTokenException as ex ->
        Invalid ex.Message
    | ex ->
        Invalid $"Unexpected error: {ex.Message}"

// Extract specific claims
let getUserId (principal: ClaimsPrincipal) =
    principal.FindFirst(ClaimTypes.NameIdentifier)
    |> Option.ofObj
    |> Option.map (fun c -> int c.Value)

let getUsername (principal: ClaimsPrincipal) =
    principal.FindFirst(ClaimTypes.Name)
    |> Option.ofObj
    |> Option.map (fun c -> c.Value)

let getRoles (principal: ClaimsPrincipal) =
    principal.FindAll(ClaimTypes.Role)
    |> Seq.map (fun c -> c.Value)
    |> Seq.toList

let hasRole (role: string) (principal: ClaimsPrincipal) =
    principal.IsInRole(role)
```

---

## 4. ASP.NET Core Authentication Middleware

```fsharp
// Giraffe authentication handlers
open Giraffe
open Microsoft.AspNetCore.Http
open Microsoft.AspNetCore.Authentication
open Microsoft.AspNetCore.Authentication.JwtBearer

// Require authentication
let requireAuthentication : HttpHandler =
    requiresAuthentication (
        setStatusCode 401 >=> json {| error = "Authentication required" |}
    )

// Challenge authentication (redirect to login)
let challengeHandler scheme : HttpHandler =
    challenge scheme

// Sign in
let signInHandler (principal: System.Security.Claims.ClaimsPrincipal) : HttpHandler =
    fun next ctx ->
        task {
            do! ctx.SignInAsync(
                Microsoft.AspNetCore.Authentication.Cookies.CookieAuthenticationDefaults.AuthenticationScheme,
                principal)
            return! next ctx
        }

// Sign out
let signOutHandler : HttpHandler =
    fun next ctx ->
        task {
            do! ctx.SignOutAsync()
            return! redirectTo false "/login" next ctx
        }

// Get current user info
let meHandler : HttpHandler =
    requireAuthentication >=>
    fun next ctx ->
        task {
            let userId = ctx.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value
            let username = ctx.User.Identity?.Name
            let roles = ctx.User.FindAll(System.Security.Claims.ClaimTypes.Role) |> Seq.map (fun c -> c.Value) |> Seq.toList
            
            return! json {|
                userId = userId
                username = username
                roles = roles
                isAuthenticated = ctx.User.Identity.IsAuthenticated
            |} next ctx
        }
```

---

## 5. Claims-based Identity

```fsharp
open System.Security.Claims

// สร้าง ClaimsIdentity
let createIdentity (userId: int) (username: string) (email: string) (roles: string list) =
    let claims = [
        yield Claim(ClaimTypes.NameIdentifier, string userId)
        yield Claim(ClaimTypes.Name, username)
        yield Claim(ClaimTypes.Email, email)
        yield! roles |> List.map (fun r -> Claim(ClaimTypes.Role, r))
        // Custom claims
        yield Claim("subscription", "premium")
        yield Claim("locale", "th-TH")
    ]
    
    ClaimsIdentity(claims, "jwt")

// สร้าง ClaimsPrincipal
let createPrincipal identity =
    ClaimsPrincipal(identity)

// Extension methods สำหรับอ่าน claims ง่ายๆ
type ClaimsPrincipal with
    member this.UserId =
        this.FindFirst(ClaimTypes.NameIdentifier)
        |> Option.ofObj
        |> Option.map (fun c -> int c.Value)
    
    member this.Username =
        this.FindFirst(ClaimTypes.Name)
        |> Option.ofObj
        |> Option.map (fun c -> c.Value)
    
    member this.Email =
        this.FindFirst(ClaimTypes.Email)
        |> Option.ofObj
        |> Option.map (fun c -> c.Value)
    
    member this.Roles =
        this.FindAll(ClaimTypes.Role)
        |> Seq.map (fun c -> c.Value)
        |> Seq.toList
    
    member this.GetClaim(claimType: string) =
        this.FindFirst(claimType)
        |> Option.ofObj
        |> Option.map (fun c -> c.Value)

// ใช้งานใน handler
let profileHandler : HttpHandler =
    fun next ctx ->
        task {
            let user = ctx.User
            let profile = {|
                userId = user.UserId
                username = user.Username
                email = user.Email
                roles = user.Roles
                subscription = user.GetClaim("subscription")
            |}
            return! json profile next ctx
        }
```

---

## 6. Role-based Authorization

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// Role-based handlers
let requireRole (role: string) : HttpHandler =
    requiresRole role (
        setStatusCode 403 >=> json {| error = $"Role '{role}' required" |}
    )

let requireAnyRole (roles: string list) : HttpHandler =
    fun next ctx ->
        task {
            let hasRole = roles |> List.exists (fun r -> ctx.User.IsInRole(r))
            if hasRole then
                return! next ctx
            else
                return! (setStatusCode 403 >=> json {| 
                    error = "Insufficient permissions"
                    required = roles
                |}) next ctx
        }

let requireAllRoles (roles: string list) : HttpHandler =
    fun next ctx ->
        task {
            let missingRoles = roles |> List.filter (fun r -> not (ctx.User.IsInRole(r)))
            if missingRoles.IsEmpty then
                return! next ctx
            else
                return! (setStatusCode 403 >=> json {|
                    error = "Insufficient permissions"
                    missingRoles = missingRoles
                |}) next ctx
        }

// ใช้งาน
let webApp : HttpHandler =
    choose [
        // Public
        GET >=> route "/api/products" >=> getProducts
        
        // Require login
        requireAuthentication >=> choose [
            GET >=> route "/api/profile" >=> getProfile
            
            // Require admin role
            requireRole "admin" >=> subRoute "/api/admin" (
                choose [
                    GET >=> route "/users" >=> adminGetUsers
                    DELETE >=> routef "/users/%i" adminDeleteUser
                ]
            )
            
            // Require any of these roles
            requireAnyRole ["manager"; "admin"] >=> choose [
                GET >=> route "/api/reports" >=> getReports
            ]
        ]
    ]
```

---

## 7. Policy-based Authorization

```fsharp
open Microsoft.AspNetCore.Authorization
open Microsoft.Extensions.DependencyInjection

// Define policies
let configureAuthorization (services: IServiceCollection) =
    services.AddAuthorization(fun opts ->
        // Simple role policy
        opts.AddPolicy("AdminOnly", fun policy ->
            policy.RequireRole("admin") |> ignore)
        
        // Claim-based policy
        opts.AddPolicy("PremiumUser", fun policy ->
            policy.RequireClaim("subscription", "premium") |> ignore)
        
        // Combined policy
        opts.AddPolicy("SeniorEditor", fun policy ->
            policy.RequireRole("editor")
                  .RequireClaim("experience", "senior")
            |> ignore)
        
        // Custom requirement policy
        opts.AddPolicy("MinAge18", fun policy ->
            policy.Requirements.Add(MinAgeRequirement(18))
            |> ignore)
        
        // Multiple requirements (AND logic)
        opts.AddPolicy("SuperAdmin", fun policy ->
            policy.RequireRole("admin")
                  .RequireClaim("mfa_verified", "true")
            |> ignore))
    |> ignore

// Custom requirement
and MinAgeRequirement(minAge: int) =
    interface IAuthorizationRequirement

// Custom handler
type MinAgeHandler() =
    inherit AuthorizationHandler<MinAgeRequirement>()
    
    override _.HandleRequirementAsync(context, requirement) =
        task {
            let birthDateClaim = context.User.FindFirst("birth_date")
            if birthDateClaim <> null then
                let birthDate = System.DateTime.Parse(birthDateClaim.Value)
                let age = System.DateTime.Today.Year - birthDate.Year
                if age >= requirement.MinAge then
                    context.Succeed(requirement)
        }

// Giraffe policy handler
open Giraffe

let requirePolicy (policyName: string) : HttpHandler =
    authorizeByPolicy policyName (
        setStatusCode 403 >=> json {| error = $"Policy '{policyName}' not satisfied" |}
    )

// ใช้งาน
let webApp : HttpHandler =
    choose [
        requireAuthentication >=> choose [
            requirePolicy "AdminOnly" >=> route "/admin" >=> adminHandler
            requirePolicy "PremiumUser" >=> route "/premium" >=> premiumHandler
        ]
    ]
```

---

## 8. Cookie Authentication

```fsharp
open Microsoft.AspNetCore.Authentication.Cookies
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Http

// Configure cookie auth
let configureCookieAuth (services: IServiceCollection) =
    services
        .AddAuthentication(CookieAuthenticationDefaults.AuthenticationScheme)
        .AddCookie(fun opts ->
            opts.LoginPath <- PathString "/login"
            opts.LogoutPath <- PathString "/logout"
            opts.ExpireTimeSpan <- System.TimeSpan.FromDays(7)
            opts.SlidingExpiration <- true
            opts.Cookie.HttpOnly <- true
            opts.Cookie.SecurePolicy <- CookieSecurePolicy.Always
            opts.Cookie.SameSite <- SameSiteMode.Strict
            opts.Cookie.Name <- "my_app_auth"
            
            opts.Events <- CookieAuthenticationEvents(
                OnRedirectToLogin = (fun ctx ->
                    // เปลี่ยนจาก redirect เป็น 401 สำหรับ API
                    if ctx.Request.Path.StartsWithSegments("/api") then
                        ctx.Response.StatusCode <- 401
                        System.Threading.Tasks.Task.CompletedTask
                    else
                        ctx.Response.Redirect(ctx.RedirectUri)
                        System.Threading.Tasks.Task.CompletedTask)))
    |> ignore

// Login handler ด้วย cookie
open Giraffe
open System.Security.Claims

type LoginDto = {
    Username: string
    Password: string
    RememberMe: bool
}

let loginHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<LoginDto>()
            
            // Validate credentials
            let isValid = dto.Username = "admin" && dto.Password = "password"
            
            if not isValid then
                return! (setStatusCode 401 >=> json {| error = "Invalid credentials" |}) next ctx
            else
                let claims = [
                    Claim(ClaimTypes.Name, dto.Username)
                    Claim(ClaimTypes.NameIdentifier, "1")
                    Claim(ClaimTypes.Role, "admin")
                ]
                let identity = ClaimsIdentity(claims, CookieAuthenticationDefaults.AuthenticationScheme)
                let principal = ClaimsPrincipal(identity)
                
                let authProps = Microsoft.AspNetCore.Authentication.AuthenticationProperties()
                authProps.IsPersistent <- dto.RememberMe
                authProps.ExpiresUtc <- 
                    if dto.RememberMe then
                        System.DateTimeOffset.UtcNow.AddDays(30) |> System.Nullable
                    else
                        System.DateTimeOffset.UtcNow.AddHours(8) |> System.Nullable
                
                do! ctx.SignInAsync(
                    CookieAuthenticationDefaults.AuthenticationScheme,
                    principal,
                    authProps)
                
                return! json {|
                    success = true
                    username = dto.Username
                    message = "Logged in successfully"
                |} next ctx
        }

// Logout handler
let logoutHandler : HttpHandler =
    fun next ctx ->
        task {
            do! ctx.SignOutAsync(CookieAuthenticationDefaults.AuthenticationScheme)
            return! (setStatusCode 200 >=> json {| message = "Logged out" |}) next ctx
        }
```

---

## 9. OAuth2/OpenID Connect

```bash
dotnet add package Microsoft.AspNetCore.Authentication.Google
dotnet add package Microsoft.AspNetCore.Authentication.MicrosoftAccount
dotnet add package Microsoft.AspNetCore.Authentication.GitHub
```

```fsharp
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Authentication.Cookies
open Microsoft.AspNetCore.Authentication.Google

// Configure OAuth2
let configureOAuth (services: IServiceCollection) (config: Microsoft.Extensions.Configuration.IConfiguration) =
    services
        .AddAuthentication(fun opts ->
            opts.DefaultScheme <- CookieAuthenticationDefaults.AuthenticationScheme
            opts.DefaultChallengeScheme <- GoogleDefaults.AuthenticationScheme)
        .AddCookie()
        .AddGoogle(fun opts ->
            opts.ClientId <- config["Google:ClientId"]
            opts.ClientSecret <- config["Google:ClientSecret"]
            opts.CallbackPath <- Microsoft.AspNetCore.Http.PathString "/auth/google/callback"
            opts.Scope.Add("profile")
            opts.Scope.Add("email")
            opts.SaveTokens <- true
            
            // Map Google claims to standard claims
            opts.Events.OnCreatingTicket <- (fun ctx ->
                task {
                    // อ่าน user info จาก Google
                    let googleUserId = ctx.Principal.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value
                    let email = ctx.Principal.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value
                    let name = ctx.Principal.FindFirst(System.Security.Claims.ClaimTypes.Name)?.Value
                    
                    printfn $"Google login: {name} ({email})"
                }))
    |> ignore

// OAuth login handler
open Giraffe

let googleLoginHandler : HttpHandler =
    challenge GoogleDefaults.AuthenticationScheme

let googleCallbackHandler : HttpHandler =
    fun next ctx ->
        task {
            // User is now authenticated
            let user = ctx.User
            let email = user.FindFirst(System.Security.Claims.ClaimTypes.Email)?.Value
            let name = user.Identity?.Name
            
            // Create or update user in database...
            
            return! redirectTo false "/dashboard" next ctx
        }

// OpenID Connect
let configureOIDC (services: IServiceCollection) =
    services
        .AddAuthentication()
        .AddOpenIdConnect("oidc", fun opts ->
            opts.Authority <- "https://identity-provider.example.com"
            opts.ClientId <- "my-client-id"
            opts.ClientSecret <- "my-client-secret"
            opts.ResponseType <- "code"
            opts.Scope.Add("openid")
            opts.Scope.Add("profile")
            opts.Scope.Add("email")
            opts.SaveTokens <- true
            opts.GetClaimsFromUserInfoEndpoint <- true)
    |> ignore
```

---

## 10. ASP.NET Core Identity

```bash
dotnet add package Microsoft.AspNetCore.Identity.EntityFrameworkCore
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
```

```fsharp
open Microsoft.AspNetCore.Identity
open Microsoft.AspNetCore.Identity.EntityFrameworkCore
open Microsoft.EntityFrameworkCore
open Microsoft.Extensions.DependencyInjection

// Custom user class
type ApplicationUser() =
    inherit IdentityUser<int>()
    
    member val DisplayName: string = "" with get, set
    member val ProfileImage: string = null with get, set
    member val CreatedAt: System.DateTime = System.DateTime.UtcNow with get, set

// DbContext
type AppDbContext(options: DbContextOptions<AppDbContext>) =
    inherit IdentityDbContext<ApplicationUser, IdentityRole<int>, int>(options)

// Configure Identity
let configureIdentity (services: IServiceCollection) =
    services.AddDbContext<AppDbContext>(fun opts ->
        opts.UseSqlite("Data Source=app.db") |> ignore)
    |> ignore
    
    services
        .AddIdentity<ApplicationUser, IdentityRole<int>>(fun opts ->
            // Password requirements
            opts.Password.RequireDigit <- true
            opts.Password.RequireLowercase <- true
            opts.Password.RequireUppercase <- true
            opts.Password.RequireNonAlphanumeric <- false
            opts.Password.MinimumLength <- 8
            
            // Lockout settings
            opts.Lockout.DefaultLockoutTimeSpan <- System.TimeSpan.FromMinutes(15)
            opts.Lockout.MaxFailedAccessAttempts <- 5
            opts.Lockout.AllowedForNewUsers <- true
            
            // User settings
            opts.User.RequireUniqueEmail <- true
            opts.SignIn.RequireConfirmedEmail <- false)
        .AddEntityFrameworkStores<AppDbContext>()
        .AddDefaultTokenProviders()
    |> ignore

// Identity handlers
open Giraffe
open Microsoft.AspNetCore.Http

type RegisterDto = {
    Username: string
    Email: string
    Password: string
    DisplayName: string
}

let registerHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<RegisterDto>()
            let userManager = ctx.GetService<UserManager<ApplicationUser>>()
            
            let user = ApplicationUser(
                UserName = dto.Username,
                Email = dto.Email,
                DisplayName = dto.DisplayName
            )
            
            let! result = userManager.CreateAsync(user, dto.Password)
            
            if result.Succeeded then
                // Add default role
                let! _ = userManager.AddToRoleAsync(user, "user")
                
                return! (setStatusCode 201 >=> json {|
                    message = "Account created successfully"
                    userId = user.Id
                    username = user.UserName
                |}) next ctx
            else
                let errors = result.Errors |> Seq.map (fun e -> e.Description) |> Seq.toList
                return! (setStatusCode 400 >=> json {| errors = errors |}) next ctx
        }

let identityLoginHandler : HttpHandler =
    fun next ctx ->
        task {
            let! dto = ctx.BindJsonAsync<LoginDto>()
            let signInManager = ctx.GetService<SignInManager<ApplicationUser>>()
            let userManager = ctx.GetService<UserManager<ApplicationUser>>()
            
            let! user = userManager.FindByNameAsync(dto.Username)
            
            if user = null then
                return! (setStatusCode 401 >=> json {| error = "Invalid credentials" |}) next ctx
            else
                let! result = signInManager.CheckPasswordSignInAsync(user, dto.Password, lockoutOnFailure = true)
                
                if result.Succeeded then
                    let! roles = userManager.GetRolesAsync(user)
                    let payload = {
                        UserId = user.Id
                        Username = user.UserName
                        Email = user.Email
                        Roles = roles |> Seq.toList
                        Claims = []
                    }
                    let config = ctx.GetService<JwtService.JwtConfig>()
                    let tokens = JwtService.createTokens config payload
                    return! json tokens next ctx
                elif result.IsLockedOut then
                    return! (setStatusCode 423 >=> json {| error = "Account is locked out" |}) next ctx
                else
                    return! (setStatusCode 401 >=> json {| error = "Invalid credentials" |}) next ctx
        }
```

---

## 11. Custom Authorization Handlers

```fsharp
open Microsoft.AspNetCore.Authorization
open Microsoft.AspNetCore.Http

// Custom requirement: user must own the resource
type ResourceOwnerRequirement(paramName: string) =
    interface IAuthorizationRequirement
    member _.ParamName = paramName

// Custom handler
type ResourceOwnerHandler(httpContextAccessor: IHttpContextAccessor) =
    inherit AuthorizationHandler<ResourceOwnerRequirement>()
    
    override _.HandleRequirementAsync(context, requirement) =
        task {
            let ctx = httpContextAccessor.HttpContext
            
            // Get resource ID from route
            let resourceIdStr = 
                ctx.Request.RouteValues[requirement.ParamName]
                |> Option.ofObj
                |> Option.map string
            
            // Get current user ID
            let userId = 
                context.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)
                |> Option.ofObj
                |> Option.map (fun c -> c.Value)
            
            match resourceIdStr, userId with
            | Some rid, Some uid when rid = uid ->
                context.Succeed(requirement)
            | _ ->
                // Admin can always access
                if context.User.IsInRole("admin") then
                    context.Succeed(requirement)
        }

// Time-based requirement
type BusinessHoursRequirement() =
    interface IAuthorizationRequirement

type BusinessHoursHandler() =
    inherit AuthorizationHandler<BusinessHoursRequirement>()
    
    override _.HandleRequirementAsync(context, requirement) =
        task {
            let now = System.DateTime.Now
            let isBusinessHours = 
                now.DayOfWeek <> System.DayOfWeek.Saturday &&
                now.DayOfWeek <> System.DayOfWeek.Sunday &&
                now.Hour >= 8 && now.Hour < 18
            
            if isBusinessHours || context.User.IsInRole("admin") then
                context.Succeed(requirement)
        }

// IP-based requirement
type AllowedIpRequirement(allowedIps: string list) =
    interface IAuthorizationRequirement
    member _.AllowedIps = allowedIps

type AllowedIpHandler(httpContextAccessor: IHttpContextAccessor) =
    inherit AuthorizationHandler<AllowedIpRequirement>()
    
    override _.HandleRequirementAsync(context, requirement) =
        task {
            let ctx = httpContextAccessor.HttpContext
            let clientIp = ctx.Connection.RemoteIpAddress?.ToString() ?? "unknown"
            
            if requirement.AllowedIps |> List.contains clientIp || 
               requirement.AllowedIps |> List.contains "127.0.0.1" then
                context.Succeed(requirement)
        }

// Register handlers
let configureCustomAuth (services: IServiceCollection) =
    services
        .AddHttpContextAccessor()
        .AddSingleton<IAuthorizationHandler, ResourceOwnerHandler>()
        .AddSingleton<IAuthorizationHandler, BusinessHoursHandler>()
        .AddSingleton<IAuthorizationHandler>(
            AllowedIpHandler(services.BuildServiceProvider().GetService<IHttpContextAccessor>()))
        .AddAuthorization(fun opts ->
            opts.AddPolicy("ResourceOwner", fun policy ->
                policy.Requirements.Add(ResourceOwnerRequirement("id"))
                |> ignore)
            opts.AddPolicy("BusinessHours", fun policy ->
                policy.Requirements.Add(BusinessHoursRequirement())
                |> ignore)
            opts.AddPolicy("OfficeOnly", fun policy ->
                policy.Requirements.Add(AllowedIpRequirement(["192.168.1.0"; "10.0.0.0"]))
                |> ignore))
    |> ignore
```

---

## 12. Securing API Endpoints

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http

// Complete secured API example

// Authentication middleware
let authenticate : HttpHandler =
    fun next ctx ->
        task {
            if ctx.User.Identity.IsAuthenticated then
                return! next ctx
            else
                return! (setStatusCode 401 >=> json {|
                    error = "Unauthorized"
                    message = "Authentication is required"
                |}) next ctx
        }

// Authorization helpers
let authorize (policy: string) : HttpHandler =
    authorizeByPolicy policy (
        setStatusCode 403 >=> json {|
            error = "Forbidden"
            message = $"Policy '{policy}' is required"
        |}
    )

let authorizeRole (role: string) : HttpHandler =
    requiresRole role (
        setStatusCode 403 >=> json {|
            error = "Forbidden"
            message = $"Role '{role}' is required"
        |}
    )

// CSRF protection (for cookie-based auth)
let validateCsrfToken : HttpHandler =
    fun next ctx ->
        task {
            let headerToken = ctx.Request.Headers["X-CSRF-Token"].ToString()
            let cookieToken = ctx.Request.Cookies["csrf_token"]
            
            if headerToken = cookieToken && not (System.String.IsNullOrEmpty(headerToken)) then
                return! next ctx
            else
                return! (setStatusCode 403 >=> json {| error = "CSRF token validation failed" |}) next ctx
        }

// API key authentication
let requireApiKey (validKeys: string list) : HttpHandler =
    fun next ctx ->
        task {
            let apiKey = ctx.Request.Headers["X-API-Key"].ToString()
            
            if validKeys |> List.contains apiKey then
                return! next ctx
            else
                return! (setStatusCode 401 >=> json {| error = "Invalid API key" |}) next ctx
        }

// Rate limit by user
let mutable private userRequests = System.Collections.Concurrent.ConcurrentDictionary<string, int * System.DateTime>()

let userRateLimit (maxRequests: int) (windowMinutes: int) : HttpHandler =
    fun next ctx ->
        task {
            let userId = 
                ctx.User.FindFirst(System.Security.Claims.ClaimTypes.NameIdentifier)?.Value
                |> Option.ofObj
                |> Option.defaultValue (ctx.Connection.RemoteIpAddress.ToString())
            
            let window = System.DateTime.UtcNow.AddMinutes(-float windowMinutes)
            
            let (count, _) = 
                userRequests.GetOrAdd(userId, (0, System.DateTime.UtcNow))
            
            if count >= maxRequests then
                ctx.Response.Headers["Retry-After"] <- string (windowMinutes * 60)
                return! (setStatusCode 429 >=> json {|
                    error = "Rate limit exceeded"
                    limit = maxRequests
                    window = $"{windowMinutes} minutes"
                |}) next ctx
            else
                let newCount = count + 1
                userRequests.[userId] <- (newCount, System.DateTime.UtcNow)
                ctx.Response.Headers["X-RateLimit-Limit"] <- string maxRequests
                ctx.Response.Headers["X-RateLimit-Remaining"] <- string (maxRequests - newCount)
                return! next ctx
        }

// Secured routes
let securedWebApp : HttpHandler =
    choose [
        // Public endpoints
        GET >=> route "/" >=> json {| message = "Welcome to the API" |}
        POST >=> route "/auth/login" >=> loginHandler
        POST >=> route "/auth/register" >=> registerHandler
        
        // JWT authenticated
        authenticate >=> choose [
            // All authenticated users
            GET >=> route "/api/me" >=> meHandler
            
            userRateLimit 100 60 >=> choose [
                GET >=> route "/api/products" >=> getProducts
                GET >=> routef "/api/products/%i" getProduct
            ]
            
            // Admin only
            authorizeRole "admin" >=> subRoute "/api/admin" (
                choose [
                    GET >=> route "/users" >=> adminGetUsers
                    DELETE >=> routef "/users/%i" adminDeleteUser
                    POST >=> route "/announcements" >=> createAnnouncement
                ]
            )
            
            // Premium users
            authorize "PremiumUser" >=> choose [
                GET >=> route "/api/reports" >=> getPremiumReports
                GET >=> route "/api/analytics" >=> getAnalytics
            ]
            
            setStatusCode 404 >=> json {| error = "Endpoint not found" |}
        ]
    ]
```

---

## สรุป

Authentication และ Authorization ใน F# / ASP.NET Core:

1. **JWT**: ใช้สำหรับ stateless APIs, mobile apps
2. **Cookie**: ใช้สำหรับ web applications ที่ต้องการ session
3. **OAuth2/OIDC**: ใช้สำหรับ third-party authentication
4. **ASP.NET Core Identity**: ใช้สำหรับ full user management
5. **Custom handlers**: ใช้สำหรับ business rules ที่ซับซ้อน

Best practices:
- Always use HTTPS
- Store secrets ใน environment variables หรือ secret manager
- ใช้ short-lived access tokens + refresh tokens
- Implement rate limiting
- Log authentication failures
- ใช้ CSRF protection สำหรับ cookie auth
