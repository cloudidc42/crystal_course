# Part 82 - Clean Architecture กับ F#

## บทนำ (Introduction)

Clean Architecture คือแนวทางการออกแบบซอฟต์แวร์ที่แยก concerns ออกเป็น layers โดยมีกฎสำคัญคือ **Dependency Rule** - dependencies ต้องชี้เข้าหา center เสมอ

```
         ┌─────────────────────────────┐
         │    Frameworks & Drivers      │  ← External world
         │  ┌───────────────────────┐   │
         │  │    Interface Adapters  │   │
         │  │  ┌─────────────────┐  │   │
         │  │  │  Application    │  │   │
         │  │  │  ┌───────────┐  │  │   │
         │  │  │  │  Domain   │  │  │   │
         │  │  │  │ (Entities)│  │  │   │
         │  │  │  └───────────┘  │  │   │
         │  │  └─────────────────┘  │   │
         │  └───────────────────────┘   │
         └─────────────────────────────┘
```

## 1. โครงสร้าง Project

```
MyApp/
├── MyApp.Domain/               # Domain layer - core business logic
│   ├── Entities/
│   ├── ValueObjects/
│   ├── Aggregates/
│   ├── Events/
│   ├── Repositories/           # Interfaces only
│   └── Services/               # Domain services
│
├── MyApp.Application/          # Application layer - use cases
│   ├── Commands/
│   ├── Queries/
│   ├── Handlers/
│   ├── Ports/                  # Interfaces for infrastructure
│   └── DTOs/
│
├── MyApp.Infrastructure/       # Infrastructure layer
│   ├── Persistence/            # Database implementations
│   ├── Messaging/              # Message queue implementations
│   ├── ExternalServices/       # External API implementations
│   └── DependencyInjection/
│
├── MyApp.Web/                  # Presentation layer
│   ├── Controllers/
│   ├── Middleware/
│   └── Models/
│
└── MyApp.Tests/               # Tests
    ├── UnitTests/
    ├── IntegrationTests/
    └── E2ETests/
```

## 2. Domain Layer

```fsharp
// ===== MyApp.Domain/ValueObjects.fs =====
namespace MyApp.Domain

open System

// Value Objects - ไม่มี dependencies กับ layers อื่น
type UserId = UserId of Guid
type ProductId = ProductId of Guid
type OrderId = OrderId of Guid

module UserId =
    let create () = UserId (Guid.NewGuid())
    let parse (s: string) =
        match Guid.TryParse(s) with
        | true, g -> Ok (UserId g)
        | _ -> Error "Invalid user ID"
    let value (UserId id) = id

type Email = private Email of string

module Email =
    let create (s: string) =
        if String.IsNullOrWhiteSpace(s) then Error "Email required"
        elif not (s.Contains("@")) then Error "Invalid email"
        else Ok (Email (s.ToLowerInvariant().Trim()))
    let value (Email e) = e

type Username = private Username of string

module Username =
    let create (s: string) =
        if String.IsNullOrWhiteSpace(s) then Error "Username required"
        elif s.Length < 3 || s.Length > 50 then Error "Username must be 3-50 chars"
        elif s.Contains(" ") then Error "Username cannot contain spaces"
        else Ok (Username (s.Trim()))
    let value (Username u) = u

type Password = private Password of string  // hashed password

module Password =
    let fromHash (hash: string) = Password hash
    let value (Password p) = p

// Domain Entities
type UserRole = Admin | Regular | Moderator

type User = {
    Id: UserId
    Username: Username
    Email: Email
    PasswordHash: Password
    Role: UserRole
    IsActive: bool
    CreatedAt: DateTime
    UpdatedAt: DateTime
}

module User =
    let create username email passwordHash =
        let now = DateTime.UtcNow
        {
            Id = UserId.create()
            Username = username
            Email = email
            PasswordHash = passwordHash
            Role = Regular
            IsActive = true
            CreatedAt = now
            UpdatedAt = now
        }
    
    let activate user = { user with IsActive = true; UpdatedAt = DateTime.UtcNow }
    let deactivate user = { user with IsActive = false; UpdatedAt = DateTime.UtcNow }
    let promoteToAdmin user = { user with Role = Admin; UpdatedAt = DateTime.UtcNow }
    let changeEmail user newEmail = { user with Email = newEmail; UpdatedAt = DateTime.UtcNow }

// Domain Exceptions / Errors
type DomainError =
    | NotFound of entity: string * id: string
    | AlreadyExists of entity: string * field: string * value: string
    | ValidationError of field: string * message: string
    | BusinessRuleViolation of rule: string * message: string
    | Unauthorized of message: string

// Domain Events
type UserDomainEvent =
    | UserRegistered of {| UserId: UserId; Email: string; Username: string; OccurredAt: DateTime |}
    | UserActivated of {| UserId: UserId; OccurredAt: DateTime |}
    | UserDeactivated of {| UserId: UserId; Reason: string; OccurredAt: DateTime |}
    | PasswordChanged of {| UserId: UserId; OccurredAt: DateTime |}
    | EmailChanged of {| UserId: UserId; NewEmail: string; OccurredAt: DateTime |}

// Repository Interfaces (in Domain, not Infrastructure)
type IUserRepository =
    abstract member FindById: UserId -> Async<User option>
    abstract member FindByEmail: Email -> Async<User option>
    abstract member FindByUsername: Username -> Async<User option>
    abstract member Save: User -> Async<unit>
    abstract member Delete: UserId -> Async<bool>
    abstract member Exists: Email -> Async<bool>
    abstract member GetAll: unit -> Async<User list>
```

## 3. Application Layer

```fsharp
// ===== MyApp.Application/Ports.fs =====
namespace MyApp.Application

open System
open MyApp.Domain

// Application Ports (interfaces for infrastructure)
type IPasswordHasher =
    abstract member Hash: plaintext: string -> string
    abstract member Verify: plaintext: string -> hash: string -> bool

type IEmailService =
    abstract member SendWelcomeEmail: email: string -> username: string -> Async<unit>
    abstract member SendPasswordResetEmail: email: string -> token: string -> Async<unit>

type ITokenService =
    abstract member GenerateToken: userId: string -> role: string -> string
    abstract member ValidateToken: token: string -> Result<{| UserId: string; Role: string |}, string>

type IEventBus =
    abstract member PublishAsync: 'T -> Async<unit>

type IUnitOfWork =
    abstract member Users: IUserRepository
    abstract member CommitAsync: unit -> Async<unit>
    abstract member RollbackAsync: unit -> Async<unit>

// ===== MyApp.Application/DTOs.fs =====
// Commands (Input)
type RegisterUserCommand = {
    Username: string
    Email: string
    Password: string
}

type LoginCommand = {
    Email: string
    Password: string
}

type UpdateUserCommand = {
    UserId: string
    Email: string option
    Username: string option
}

type ChangePasswordCommand = {
    UserId: string
    CurrentPassword: string
    NewPassword: string
}

// Queries (Input)
type GetUserQuery = { UserId: string }
type GetUserByEmailQuery = { Email: string }
type ListUsersQuery = { 
    Page: int
    PageSize: int
    RoleFilter: string option
}

// Results (Output)
type UserDto = {
    Id: string
    Username: string
    Email: string
    Role: string
    IsActive: bool
    CreatedAt: DateTime
}

type AuthResult = {
    Token: string
    User: UserDto
    ExpiresAt: DateTime
}

type PagedResult<'T> = {
    Items: 'T list
    TotalCount: int
    Page: int
    PageSize: int
    TotalPages: int
}

module UserDto =
    let fromDomain (user: User) : UserDto = {
        Id = UserId.value user.Id |> string
        Username = Username.value user.Username
        Email = Email.value user.Email
        Role = sprintf "%A" user.Role
        IsActive = user.IsActive
        CreatedAt = user.CreatedAt
    }

// ===== MyApp.Application/UserService.fs =====
type UserApplicationService(
    uow: IUnitOfWork,
    passwordHasher: IPasswordHasher,
    tokenService: ITokenService,
    emailService: IEmailService,
    eventBus: IEventBus) =
    
    member _.RegisterUser (command: RegisterUserCommand) : Async<Result<UserDto, DomainError>> =
        async {
            // Validate inputs
            match Username.create command.Username,
                  Email.create command.Email with
            | Error e, _ -> return Error (ValidationError ("Username", e))
            | _, Error e -> return Error (ValidationError ("Email", e))
            | Ok username, Ok email ->
            
            if String.IsNullOrWhiteSpace(command.Password) || command.Password.Length < 8 then
                return Error (ValidationError ("Password", "Password must be at least 8 characters"))
            else
            
            // Check uniqueness
            let! emailExists = uow.Users.Exists email
            if emailExists then
                return Error (AlreadyExists ("User", "email", command.Email))
            else
            
            let! usernameUser = uow.Users.FindByUsername username
            if usernameUser.IsSome then
                return Error (AlreadyExists ("User", "username", command.Username))
            else
            
            // Create user
            let hash = passwordHasher.Hash command.Password
            let user = User.create username email (Password.fromHash hash)
            
            // Save
            do! uow.Users.Save user
            do! uow.CommitAsync()
            
            // Publish event
            let now = DateTime.UtcNow
            do! eventBus.PublishAsync(UserRegistered {|
                UserId = user.Id
                Email = command.Email
                Username = command.Username
                OccurredAt = now |})
            
            // Send welcome email (fire and forget style)
            Async.Start (emailService.SendWelcomeEmail command.Email command.Username)
            
            return Ok (UserDto.fromDomain user)
        }
    
    member _.Login (command: LoginCommand) : Async<Result<AuthResult, DomainError>> =
        async {
            match Email.create command.Email with
            | Error e -> return Error (ValidationError ("Email", e))
            | Ok email ->
            
            let! userOpt = uow.Users.FindByEmail email
            match userOpt with
            | None -> return Error (Unauthorized "Invalid credentials")
            | Some user ->
            
            if not user.IsActive then
                return Error (Unauthorized "Account is deactivated")
            else
            
            let isValid = passwordHasher.Verify command.Password (Password.value user.PasswordHash)
            if not isValid then
                return Error (Unauthorized "Invalid credentials")
            else
            
            let userId = UserId.value user.Id |> string
            let role = sprintf "%A" user.Role
            let token = tokenService.GenerateToken userId role
            let expiresAt = DateTime.UtcNow.AddHours(24.0)
            
            return Ok {
                Token = token
                User = UserDto.fromDomain user
                ExpiresAt = expiresAt
            }
        }
    
    member _.GetUser (query: GetUserQuery) : Async<Result<UserDto, DomainError>> =
        async {
            match UserId.parse query.UserId with
            | Error _ -> return Error (ValidationError ("UserId", "Invalid user ID format"))
            | Ok userId ->
            
            let! userOpt = uow.Users.FindById userId
            match userOpt with
            | None -> return Error (NotFound ("User", query.UserId))
            | Some user -> return Ok (UserDto.fromDomain user)
        }
    
    member _.ChangePassword (command: ChangePasswordCommand) : Async<Result<unit, DomainError>> =
        async {
            match UserId.parse command.UserId with
            | Error _ -> return Error (ValidationError ("UserId", "Invalid user ID"))
            | Ok userId ->
            
            let! userOpt = uow.Users.FindById userId
            match userOpt with
            | None -> return Error (NotFound ("User", command.UserId))
            | Some user ->
            
            // Verify current password
            let isValid = passwordHasher.Verify command.CurrentPassword (Password.value user.PasswordHash)
            if not isValid then
                return Error (Unauthorized "Current password is incorrect")
            else
            
            // Validate new password
            if command.NewPassword.Length < 8 then
                return Error (ValidationError ("NewPassword", "Password must be at least 8 characters"))
            else
            
            let hash = passwordHasher.Hash command.NewPassword
            let updatedUser = { user with 
                                    PasswordHash = Password.fromHash hash
                                    UpdatedAt = DateTime.UtcNow }
            
            do! uow.Users.Save updatedUser
            do! uow.CommitAsync()
            
            do! eventBus.PublishAsync(PasswordChanged {|
                UserId = user.Id
                OccurredAt = DateTime.UtcNow |})
            
            return Ok ()
        }
```

## 4. Infrastructure Layer

```fsharp
// ===== MyApp.Infrastructure/Persistence/UserRepository.fs =====
namespace MyApp.Infrastructure.Persistence

open System
open System.Collections.Generic
open MyApp.Domain
open MyApp.Application

// In-memory implementation (for testing/development)
type InMemoryUserRepository() =
    let storage = Dictionary<Guid, User>()
    
    interface IUserRepository with
        member _.FindById userId =
            async {
                let guid = UserId.value userId
                match storage.TryGetValue(guid) with
                | true, user -> return Some user
                | _ -> return None
            }
        
        member _.FindByEmail email =
            async {
                let emailStr = Email.value email
                return storage.Values 
                       |> Seq.tryFind (fun u -> Email.value u.Email = emailStr)
            }
        
        member _.FindByUsername username =
            async {
                let usernameStr = Username.value username
                return storage.Values 
                       |> Seq.tryFind (fun u -> Username.value u.Username = usernameStr)
            }
        
        member _.Save user =
            async {
                let guid = UserId.value user.Id
                storage.[guid] <- user
            }
        
        member _.Delete userId =
            async {
                let guid = UserId.value userId
                return storage.Remove(guid)
            }
        
        member _.Exists email =
            async {
                let emailStr = Email.value email
                return storage.Values |> Seq.exists (fun u -> Email.value u.Email = emailStr)
            }
        
        member _.GetAll () =
            async {
                return storage.Values |> Seq.toList
            }

// SQL Server implementation (using Dapper)
type SqlUserRepository(connectionString: string) =
    
    let getConnection () =
        new System.Data.SqlClient.SqlConnection(connectionString)
    
    interface IUserRepository with
        member _.FindById userId =
            async {
                use conn = getConnection()
                let sql = """
                    SELECT Id, Username, Email, PasswordHash, Role, IsActive, CreatedAt, UpdatedAt
                    FROM Users
                    WHERE Id = @Id
                """
                // Simplified - real implementation would use Dapper
                let guid = UserId.value userId
                // let result = await conn.QueryFirstOrDefaultAsync<UserRecord>(sql, {| Id = guid |})
                // Convert to domain User
                return None // Placeholder
            }
        
        member _.FindByEmail email =
            async {
                // Implementation
                return None // Placeholder
            }
        
        member _.FindByUsername username =
            async {
                return None // Placeholder
            }
        
        member _.Save user =
            async {
                // Upsert user
                ()
            }
        
        member _.Delete userId =
            async {
                return false
            }
        
        member _.Exists email =
            async {
                return false
            }
        
        member _.GetAll () =
            async {
                return []
            }

// ===== MyApp.Infrastructure/Security/PasswordHasher.fs =====
type BcryptPasswordHasher() =
    interface IPasswordHasher with
        member _.Hash plaintext =
            // In real code: BCrypt.Net.BCrypt.HashPassword(plaintext)
            sprintf "bcrypt:%s" (plaintext.GetHashCode().ToString())
        
        member _.Verify plaintext hash =
            // In real code: BCrypt.Net.BCrypt.Verify(plaintext, hash)
            sprintf "bcrypt:%s" (plaintext.GetHashCode().ToString()) = hash

// ===== MyApp.Infrastructure/Security/JwtTokenService.fs =====
type JwtTokenService(secretKey: string, issuer: string) =
    interface ITokenService with
        member _.GenerateToken userId role =
            // Real implementation using System.IdentityModel.Tokens.Jwt
            sprintf "jwt.%s.%s.%s" userId role (DateTime.UtcNow.Ticks.ToString())
        
        member _.ValidateToken token =
            // Real implementation
            let parts = token.Split('.')
            if parts.Length >= 3 then
                Ok {| UserId = parts.[1]; Role = parts.[2] |}
            else
                Error "Invalid token"

// ===== MyApp.Infrastructure/Email/SmtpEmailService.fs =====
type SmtpEmailService(smtpServer: string, from: string) =
    interface IEmailService with
        member _.SendWelcomeEmail email username =
            async {
                printfn "Sending welcome email to %s (%s)" username email
                // Real implementation using MailKit
            }
        
        member _.SendPasswordResetEmail email token =
            async {
                printfn "Sending password reset to %s with token %s" email token
            }

// ===== MyApp.Infrastructure/Messaging/InMemoryEventBus.fs =====
type InMemoryEventBus() =
    let handlers = Collections.Generic.Dictionary<Type, ResizeArray<obj>>()
    
    interface IEventBus with
        member _.PublishAsync<'T> (event: 'T) =
            async {
                let key = typeof<'T>
                if handlers.ContainsKey(key) then
                    for handler in handlers.[key] do
                        let typedHandler = handler :?> ('T -> Async<unit>)
                        do! typedHandler event
            }
    
    member _.Subscribe<'T>(handler: 'T -> Async<unit>) =
        let key = typeof<'T>
        if not (handlers.ContainsKey(key)) then
            handlers.[key] <- ResizeArray()
        handlers.[key].Add(handler :> obj)

// ===== MyApp.Infrastructure/UnitOfWork.fs =====
type InMemoryUnitOfWork() =
    let userRepo = InMemoryUserRepository() :> IUserRepository
    
    interface IUnitOfWork with
        member _.Users = userRepo
        
        member _.CommitAsync() =
            async {
                // In-memory: nothing to commit
                ()
            }
        
        member _.RollbackAsync() =
            async {
                // In-memory: nothing to rollback
                ()
            }
```

## 5. Presentation Layer (ASP.NET Core)

```fsharp
// ===== MyApp.Web/Controllers/UserController.fs =====
namespace MyApp.Web.Controllers

open Microsoft.AspNetCore.Mvc
open Microsoft.AspNetCore.Authorization
open MyApp.Application

[<ApiController>]
[<Route("api/[controller]")>]
type UsersController(userService: UserApplicationService) =
    inherit ControllerBase()
    
    [<HttpPost("register")>]
    member this.Register([<FromBody>] command: RegisterUserCommand) =
        async {
            match! userService.RegisterUser command with
            | Ok user ->
                return this.CreatedAtAction("GetUser", {| id = user.Id |}, user) :> IActionResult
            | Error (ValidationError (field, msg)) ->
                return this.BadRequest({| field = field; message = msg |}) :> IActionResult
            | Error (AlreadyExists (_, field, value)) ->
                return this.Conflict({| message = sprintf "%s '%s' already exists" field value |}) :> IActionResult
            | Error e ->
                return this.BadRequest({| message = sprintf "%A" e |}) :> IActionResult
        } |> Async.StartAsTask
    
    [<HttpPost("login")>]
    member this.Login([<FromBody>] command: LoginCommand) =
        async {
            match! userService.Login command with
            | Ok result ->
                return this.Ok(result) :> IActionResult
            | Error (Unauthorized msg) ->
                return this.Unauthorized({| message = msg |}) :> IActionResult
            | Error e ->
                return this.BadRequest({| error = sprintf "%A" e |}) :> IActionResult
        } |> Async.StartAsTask
    
    [<HttpGet("{id}")>]
    [<Authorize>]
    member this.GetUser(id: string) =
        async {
            match! userService.GetUser { UserId = id } with
            | Ok user ->
                return this.Ok(user) :> IActionResult
            | Error (NotFound _) ->
                return this.NotFound() :> IActionResult
            | Error e ->
                return this.BadRequest({| error = sprintf "%A" e |}) :> IActionResult
        } |> Async.StartAsTask
    
    [<HttpPut("{id}/password")>]
    [<Authorize>]
    member this.ChangePassword(id: string, [<FromBody>] command: ChangePasswordCommand) =
        async {
            let command = { command with UserId = id }
            match! userService.ChangePassword command with
            | Ok () ->
                return this.NoContent() :> IActionResult
            | Error (Unauthorized msg) ->
                return this.Unauthorized({| message = msg |}) :> IActionResult
            | Error e ->
                return this.BadRequest({| error = sprintf "%A" e |}) :> IActionResult
        } |> Async.StartAsTask

// ===== MyApp.Web/Middleware/ErrorHandlingMiddleware.fs =====
open Microsoft.AspNetCore.Http
open System.Text.Json

type ErrorHandlingMiddleware(next: RequestDelegate) =
    
    member _.InvokeAsync(ctx: HttpContext) =
        async {
            try
                do! next.Invoke(ctx) |> Async.AwaitTask
            with
            | :? System.OperationCanceledException ->
                ctx.Response.StatusCode <- 499
            | ex ->
                ctx.Response.StatusCode <- 500
                ctx.Response.ContentType <- "application/json"
                let error = {| message = "An unexpected error occurred"; traceId = ctx.TraceIdentifier |}
                let json = JsonSerializer.Serialize(error)
                do! ctx.Response.WriteAsync(json) |> Async.AwaitTask
        } |> Async.StartAsTask :> System.Threading.Tasks.Task
```

## 6. Dependency Injection และ Composition Root

```fsharp
// ===== MyApp.Web/Program.fs =====
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open MyApp.Application
open MyApp.Infrastructure.Persistence
open MyApp.Infrastructure

let builder = WebApplication.CreateBuilder()

// Register services
let services = builder.Services

// Domain/Application - no registration needed (pure functions)

// Infrastructure
services.AddSingleton<IUserRepository>(fun _ -> 
    InMemoryUserRepository() :> IUserRepository) |> ignore

services.AddSingleton<IPasswordHasher>(fun _ ->
    BcryptPasswordHasher() :> IPasswordHasher) |> ignore

services.AddSingleton<ITokenService>(fun _ ->
    JwtTokenService("secret-key", "my-app") :> ITokenService) |> ignore

services.AddSingleton<IEmailService>(fun _ ->
    SmtpEmailService("localhost", "noreply@myapp.com") :> IEmailService) |> ignore

services.AddSingleton<IEventBus>(fun _ ->
    InMemoryEventBus() :> IEventBus) |> ignore

services.AddScoped<IUnitOfWork>(fun sp ->
    InMemoryUnitOfWork() :> IUnitOfWork) |> ignore

// Application Service
services.AddScoped<UserApplicationService>(fun sp ->
    UserApplicationService(
        sp.GetRequiredService<IUnitOfWork>(),
        sp.GetRequiredService<IPasswordHasher>(),
        sp.GetRequiredService<ITokenService>(),
        sp.GetRequiredService<IEmailService>(),
        sp.GetRequiredService<IEventBus>()
    )) |> ignore

// ASP.NET Core
services.AddControllers() |> ignore
services.AddEndpointsApiExplorer() |> ignore
services.AddSwaggerGen() |> ignore

let app = builder.Build()

if app.Environment.IsDevelopment() then
    app.UseSwagger() |> ignore
    app.UseSwaggerUI() |> ignore

app.UseHttpsRedirection() |> ignore
app.UseAuthentication() |> ignore
app.UseAuthorization() |> ignore
app.MapControllers() |> ignore

app.Run()
```

## 7. Ports and Adapters Pattern

```fsharp
// Ports = Interfaces defined in domain/application
// Adapters = Implementations in infrastructure

// ===== Port (Application Layer) =====
type IStorageAdapter =
    abstract member UploadFile: filename: string -> content: byte[] -> Async<string>  // returns URL
    abstract member DeleteFile: url: string -> Async<bool>
    abstract member GetFileUrl: filename: string -> string

type IPaymentGateway =
    abstract member Charge: amount: decimal -> currency: string -> token: string -> Async<Result<string, string>>
    abstract member Refund: transactionId: string -> amount: decimal -> Async<Result<unit, string>>

type INotificationService =
    abstract member SendPushNotification: userId: string -> message: string -> Async<unit>
    abstract member SendSms: phone: string -> message: string -> Async<unit>

// ===== Adapters (Infrastructure Layer) =====

// S3 Adapter
type S3StorageAdapter(bucketName: string, region: string) =
    interface IStorageAdapter with
        member _.UploadFile filename content =
            async {
                // Real: use AWSSDK.S3
                printfn "Uploading %s to S3 bucket %s" filename bucketName
                return sprintf "https://%s.s3.%s.amazonaws.com/%s" bucketName region filename
            }
        
        member _.DeleteFile url =
            async {
                printfn "Deleting %s from S3" url
                return true
            }
        
        member _.GetFileUrl filename =
            sprintf "https://%s.s3.%s.amazonaws.com/%s" bucketName region filename

// Azure Blob Storage Adapter
type AzureBlobStorageAdapter(connectionString: string, containerName: string) =
    interface IStorageAdapter with
        member _.UploadFile filename content =
            async {
                printfn "Uploading %s to Azure Blob %s" filename containerName
                return sprintf "https://storage.blob.core.windows.net/%s/%s" containerName filename
            }
        
        member _.DeleteFile url =
            async { return true }
        
        member _.GetFileUrl filename =
            sprintf "https://storage.blob.core.windows.net/%s/%s" containerName filename

// Stripe Payment Adapter
type StripePaymentAdapter(apiKey: string) =
    interface IPaymentGateway with
        member _.Charge amount currency token =
            async {
                printfn "Charging %.2f %s via Stripe" amount currency
                // Real: use Stripe.net
                return Ok (sprintf "stripe-txn-%s" (System.Guid.NewGuid().ToString("N")[..7]))
            }
        
        member _.Refund transactionId amount =
            async {
                printfn "Refunding %.2f from transaction %s" amount transactionId
                return Ok ()
            }
```

## 8. Testing Clean Architecture

```fsharp
// Testing ง่ายเพราะ dependencies ถูก inject และสามารถ mock ได้

// ===== Test Doubles =====
type MockPasswordHasher() =
    interface IPasswordHasher with
        member _.Hash plaintext = sprintf "hash:%s" plaintext
        member _.Verify plaintext hash = hash = sprintf "hash:%s" plaintext

type MockTokenService() =
    interface ITokenService with
        member _.GenerateToken userId role = sprintf "token:%s:%s" userId role
        member _.ValidateToken token =
            let parts = token.Split(':')
            if parts.Length = 3 && parts.[0] = "token" then
                Ok {| UserId = parts.[1]; Role = parts.[2] |}
            else
                Error "Invalid token"

type MockEmailService() =
    let sentEmails = ResizeArray<string * string>()
    
    interface IEmailService with
        member _.SendWelcomeEmail email username =
            async { sentEmails.Add(email, username) }
        member _.SendPasswordResetEmail email token =
            async { sentEmails.Add(email, token) }
    
    member _.SentEmails = sentEmails |> Seq.toList

type MockEventBus() =
    let publishedEvents = ResizeArray<obj>()
    
    interface IEventBus with
        member _.PublishAsync<'T> (event: 'T) =
            async { publishedEvents.Add(event) }
    
    member _.PublishedEvents = publishedEvents |> Seq.toList

// ===== Integration Test Setup =====
let createTestServices () =
    let userRepo = InMemoryUserRepository() :> IUserRepository
    let uow = { new IUnitOfWork with
                    member _.Users = userRepo
                    member _.CommitAsync() = async { () }
                    member _.RollbackAsync() = async { () } }
    let passwordHasher = MockPasswordHasher() :> IPasswordHasher
    let tokenService = MockTokenService() :> ITokenService
    let emailService = MockEmailService()
    let eventBus = MockEventBus()
    
    let service = UserApplicationService(
        uow, passwordHasher, tokenService, 
        emailService :> IEmailService, 
        eventBus :> IEventBus)
    
    service, emailService, eventBus

// ===== Tests =====
let testUserRegistration () =
    async {
        let service, emailService, eventBus = createTestServices()
        
        let command = {
            Username = "johndoe"
            Email = "john@example.com"
            Password = "password123"
        }
        
        let! result = service.RegisterUser command
        
        match result with
        | Ok user ->
            printfn "✓ User registered: %s" user.Username
            assert (user.Username = "johndoe")
            assert (user.Email = "john@example.com")
            assert (user.Role = "Regular")
        | Error e ->
            failwith (sprintf "Registration failed: %A" e)
        
        // Verify email was sent
        let emails = emailService.SentEmails
        assert (emails.Length = 1)
        printfn "✓ Welcome email sent"
        
        // Verify event was published
        let events = eventBus.PublishedEvents
        assert (events.Length = 1)
        printfn "✓ Event published"
    }

let testDuplicateRegistration () =
    async {
        let service, _, _ = createTestServices()
        
        let command = {
            Username = "johndoe"
            Email = "john@example.com"
            Password = "password123"
        }
        
        let! _ = service.RegisterUser command
        let! result = service.RegisterUser { command with Username = "johndoe2" }
        
        match result with
        | Error (AlreadyExists _) ->
            printfn "✓ Duplicate email rejected"
        | _ ->
            failwith "Should have rejected duplicate email"
    }

let testLoginFlow () =
    async {
        let service, _, _ = createTestServices()
        
        // Register
        let registerCmd = { Username = "testuser"; Email = "test@example.com"; Password = "secret123" }
        let! _ = service.RegisterUser registerCmd
        
        // Login with correct credentials
        let loginCmd = { Email = "test@example.com"; Password = "secret123" }
        let! loginResult = service.Login loginCmd
        
        match loginResult with
        | Ok auth ->
            printfn "✓ Login successful, token: %s..." (auth.Token.[..15])
        | Error e ->
            failwith (sprintf "Login failed: %A" e)
        
        // Login with wrong password
        let wrongCmd = { Email = "test@example.com"; Password = "wrong" }
        let! wrongResult = service.Login wrongCmd
        
        match wrongResult with
        | Error (Unauthorized _) -> printfn "✓ Wrong password rejected"
        | _ -> failwith "Should reject wrong password"
    }

// Run all tests
let runTests () =
    printfn "=== Clean Architecture Tests ==="
    [testUserRegistration(); testDuplicateRegistration(); testLoginFlow()]
    |> Async.Sequential
    |> Async.RunSynchronously
    |> ignore
    printfn "✓ All tests passed!"

runTests()
```

## 9. Dependency Rule Summary

```fsharp
(*
Clean Architecture Dependency Rule:
─────────────────────────────────────

Domain Layer (Center):
- Entities, Value Objects, Aggregates
- Domain Events, Domain Services
- Repository Interfaces
- NO dependencies on other layers

Application Layer:
- Use Cases (Application Services)
- Commands, Queries, DTOs
- Port Interfaces (for Infrastructure)
- Depends only on: Domain Layer

Infrastructure Layer:
- Repository Implementations (SQL, MongoDB, etc.)
- External Service Adapters (Email, SMS, Storage)
- Security (JWT, Hashing)
- Depends on: Domain + Application layers

Presentation Layer (Outer):
- Controllers, Middleware
- Request/Response models
- Depends on: Application Layer (not Domain directly)

Key Principle: Dependencies point INWARD
   Outer layers know about inner layers
   Inner layers know NOTHING about outer layers

Benefits:
1. Domain logic is isolated and testable
2. Infrastructure can be swapped without changing business logic
3. Tests can use mock implementations
4. Business rules don't depend on frameworks
*)

printfn "Clean Architecture principles applied!"
printfn "Domain is isolated from infrastructure"
printfn "Dependencies flow inward"
```

---

**สรุป**: Clean Architecture ใน F# ทำให้แต่ละ layer มีความรับผิดชอบที่ชัดเจน โดย Domain layer อยู่ตรงกลางและไม่ขึ้นกับ infrastructure ใดๆ ทำให้ test ง่ายและเปลี่ยน implementation ได้โดยไม่กระทบ business logic
