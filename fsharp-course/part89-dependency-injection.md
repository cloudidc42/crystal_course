# Part 89 - Dependency Injection กับ F#

## บทนำ (Introduction)

Dependency Injection (DI) คือ technique สำหรับ inversion of control (IoC) ที่ช่วยให้ dependencies ถูก provide จากภายนอกแทนที่จะสร้างเอง ใน F# มีหลายวิธี:

1. **Microsoft.Extensions.DependencyInjection** - IoC container
2. **Function-based DI** - Partial application
3. **Reader Monad** - Functional DI pattern
4. **Record of functions** - Dependency bundles

## 1. Microsoft.Extensions.DependencyInjection

```fsharp
// ===== Setup =====
// NuGet: Microsoft.Extensions.DependencyInjection

open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open System

// ===== Service Interfaces =====
type IEmailService =
    abstract member Send: to': string -> subject: string -> body: string -> Async<unit>

type ISmsService =
    abstract member Send: to': string -> message: string -> Async<unit>

type IUserRepository =
    abstract member FindById: string -> Async<User option>
    abstract member Save: User -> Async<unit>
    abstract member Exists: email: string -> Async<bool>

type IPasswordHasher =
    abstract member Hash: string -> string
    abstract member Verify: plaintext: string -> hash: string -> bool

type ILogger =
    abstract member Info: string -> unit
    abstract member Error: string -> exn option -> unit
    abstract member Debug: string -> unit

and User = {
    Id: string
    Email: string
    Name: string
    PasswordHash: string
    IsActive: bool
    CreatedAt: DateTime
}

// ===== Concrete Implementations =====

// Email Service - SMTP implementation
type SmtpEmailService(smtpServer: string, port: int, sender: string) =
    interface IEmailService with
        member _.Send to' subject body =
            async {
                printfn "[SMTP] Sending email to %s: %s" to' subject
                // Real implementation would use MailKit or similar
                // use client = new MailKit.Net.Smtp.SmtpClient()
                // do! client.ConnectAsync(smtpServer, port, false)
                // ...
            }

// SendGrid Email Service
type SendGridEmailService(apiKey: string) =
    interface IEmailService with
        member _.Send to' subject body =
            async {
                printfn "[SendGrid] Sending email to %s via SendGrid" to'
                // Real: use SendGrid.NET SDK
            }

// Twilio SMS Service
type TwilioSmsService(accountSid: string, authToken: string, fromNumber: string) =
    interface ISmsService with
        member _.Send to' message =
            async {
                printfn "[Twilio] Sending SMS to %s: %s" to' message
                // Real: use Twilio SDK
            }

// BCrypt Password Hasher
type BCryptPasswordHasher() =
    interface IPasswordHasher with
        member _.Hash plaintext =
            sprintf "bcrypt$12$%s" (plaintext.GetHashCode().ToString("X"))  // Simplified
        member _.Verify plaintext hash =
            hash = sprintf "bcrypt$12$%s" (plaintext.GetHashCode().ToString("X"))

// Console Logger
type ConsoleLogger(serviceName: string) =
    interface ILogger with
        member _.Info message = printfn "[INFO][%s] %s" serviceName message
        member _.Error message exnOpt =
            printfn "[ERROR][%s] %s" serviceName message
            exnOpt |> Option.iter (fun ex -> printfn "  Exception: %s" ex.Message)
        member _.Debug message = printfn "[DEBUG][%s] %s" serviceName message

// In-memory User Repository
type InMemoryUserRepository(logger: ILogger) =
    let storage = System.Collections.Generic.Dictionary<string, User>()
    
    interface IUserRepository with
        member _.FindById id =
            async {
                logger.Debug (sprintf "FindById: %s" id)
                match storage.TryGetValue(id) with
                | true, user -> return Some user
                | _ -> return None
            }
        
        member _.Save user =
            async {
                storage.[user.Id] <- user
                logger.Info (sprintf "Saved user: %s" user.Id)
            }
        
        member _.Exists email =
            async {
                return storage.Values |> Seq.exists (fun u -> u.Email = email)
            }
```

## 2. Service Lifetimes

```fsharp
// ===== Singleton, Scoped, Transient =====

// Application Service
type UserService(
    userRepo: IUserRepository,
    passwordHasher: IPasswordHasher,
    emailService: IEmailService,
    logger: ILogger) =
    
    member _.RegisterUser (email: string) (name: string) (password: string) =
        async {
            logger.Info (sprintf "Registering user: %s" email)
            
            // Validate
            if String.IsNullOrWhiteSpace(email) then
                return Error "Email required"
            elif password.Length < 8 then
                return Error "Password too short"
            else
            
            // Check duplicate
            let! exists = userRepo.Exists email
            if exists then
                return Error "Email already registered"
            else
            
            // Create user
            let user = {
                Id = Guid.NewGuid().ToString()
                Email = email.ToLower()
                Name = name
                PasswordHash = passwordHasher.Hash password
                IsActive = true
                CreatedAt = DateTime.UtcNow
            }
            
            do! userRepo.Save user
            
            // Send welcome email (fire and forget)
            Async.Start (emailService.Send user.Email "Welcome!" (sprintf "Hello %s!" user.Name))
            
            logger.Info (sprintf "User registered: %s" user.Id)
            return Ok user
        }
    
    member _.GetUser (id: string) =
        async {
            return! userRepo.FindById id
        }

// Configuration options (from appsettings.json)
type EmailOptions = {
    SmtpServer: string
    Port: int
    SenderEmail: string
}

type AppOptions = {
    Email: EmailOptions
    DatabaseConnectionString: string
    ServiceName: string
}

// ===== DI Container Setup =====
let createServiceCollection () =
    let services = ServiceCollection()
    
    // Singleton: Created once, shared for entire app lifetime
    services.AddSingleton<ILogger>(fun _ ->
        ConsoleLogger("MyApp") :> ILogger) |> ignore
    
    services.AddSingleton<IPasswordHasher, BCryptPasswordHasher>() |> ignore
    
    // Scoped: Created once per HTTP request (or scope)
    services.AddScoped<IUserRepository>(fun sp ->
        let logger = sp.GetRequiredService<ILogger>()
        InMemoryUserRepository(logger) :> IUserRepository) |> ignore
    
    // Transient: Created new every time requested
    services.AddTransient<IEmailService>(fun _ ->
        SmtpEmailService("smtp.gmail.com", 587, "noreply@myapp.com") :> IEmailService) |> ignore
    
    services.AddTransient<ISmsService>(fun _ ->
        TwilioSmsService("acc-sid", "auth-token", "+1234567890") :> ISmsService) |> ignore
    
    // Application service
    services.AddScoped<UserService>() |> ignore
    
    services

// ===== Using the DI Container =====
let runWithDI () =
    let services = createServiceCollection()
    let serviceProvider = services.BuildServiceProvider()
    
    // Create a scope (simulating HTTP request scope)
    use scope = serviceProvider.CreateScope()
    let userService = scope.ServiceProvider.GetRequiredService<UserService>()
    
    async {
        let! result = userService.RegisterUser "john@example.com" "John Doe" "password123"
        match result with
        | Ok user -> printfn "Registered: %s" user.Id
        | Error e -> printfn "Error: %s" e
    } |> Async.RunSynchronously

runWithDI()
```

## 3. Constructor Injection via Classes

```fsharp
// ===== Constructor Injection =====
// F# classes can use constructor injection just like C#

type ReportService(
    userRepo: IUserRepository,
    emailService: IEmailService,
    logger: ILogger) =
    
    member _.GenerateAndSendReport (userId: string) (recipientEmail: string) =
        async {
            logger.Info (sprintf "Generating report for user %s" userId)
            
            let! userOpt = userRepo.FindById userId
            match userOpt with
            | None ->
                logger.Error (sprintf "User %s not found" userId) None
                return Error "User not found"
            | Some user ->
            
            let report = sprintf """
                Report for: %s
                Email: %s
                Generated: %s
                Status: %s
            """ user.Name user.Email (DateTime.UtcNow.ToString()) (if user.IsActive then "Active" else "Inactive")
            
            do! emailService.Send recipientEmail "Your Report" report
            
            logger.Info (sprintf "Report sent to %s" recipientEmail)
            return Ok "Report sent"
        }

// Registering with ServiceCollection
let registerServices (services: IServiceCollection) =
    services.AddScoped<ReportService>() |> ignore
    services

// Multiple interfaces per class
type NotificationHub(emailService: IEmailService, smsService: ISmsService, logger: ILogger) =
    
    member _.NotifyUser (userId: string) (email: string) (phone: string) (message: string) =
        async {
            logger.Info (sprintf "Notifying user %s" userId)
            
            // Send both email and SMS
            let emailTask = emailService.Send email "Notification" message
            let smsTask = smsService.Send phone message
            
            do! [emailTask; smsTask] |> Async.Parallel |> Async.Ignore
            
            logger.Info "Notifications sent"
        }
```

## 4. Function-based DI (Partial Application)

```fsharp
// ===== Function-based DI =====
// ไม่ต้องใช้ interface หรือ class เลย

// Define service types as function types
type FindUser = string -> Async<User option>
type SaveUser = User -> Async<unit>
type HashPassword = string -> string
type VerifyPassword = string -> string -> bool
type SendEmail = string -> string -> string -> Async<unit>
type LogInfo = string -> unit

// Pure domain logic - no dependencies
let validateEmail (email: string) =
    if String.IsNullOrWhiteSpace(email) then Error "Email required"
    elif not (email.Contains("@")) then Error "Invalid email"
    else Ok (email.ToLower().Trim())

let validatePassword (password: string) =
    if password.Length < 8 then Error "Password too short"
    elif not (password |> Seq.exists System.Char.IsDigit) then Error "Password must contain a digit"
    else Ok password

// Application function with dependencies injected
let registerUser
    (findUser: FindUser)
    (saveUser: SaveUser)
    (hashPassword: HashPassword)
    (sendWelcomeEmail: SendEmail)
    (logInfo: LogInfo)
    (email: string)
    (name: string)
    (password: string) =
    async {
        logInfo (sprintf "Registering %s" email)
        
        // Validate
        match validateEmail email, validatePassword password with
        | Error e, _ -> return Error e
        | _, Error e -> return Error e
        | Ok validEmail, Ok validPassword ->
        
        // Check existing
        let! existing = findUser validEmail
        if existing.IsSome then
            return Error "Email already in use"
        else
        
        let user = {
            Id = Guid.NewGuid().ToString()
            Email = validEmail
            Name = name
            PasswordHash = hashPassword validPassword
            IsActive = true
            CreatedAt = DateTime.UtcNow
        }
        
        do! saveUser user
        do! sendWelcomeEmail user.Email "Welcome!" (sprintf "Hello %s!" user.Name)
        
        logInfo (sprintf "User %s registered" user.Id)
        return Ok user
    }

// Concrete implementations as simple functions
let findUserInDb (connectionString: string) = fun (email: string) ->
    async {
        // Real: query database
        return None
    }

let saveUserToDb (connectionString: string) = fun (user: User) ->
    async {
        printfn "Saving user %s to DB" user.Id
    }

let bcryptHash = fun (plaintext: string) ->
    sprintf "hash:%s" (plaintext.GetHashCode().ToString("X"))

let sendSmtpEmail (smtpServer: string) = fun (to': string) (subject: string) (body: string) ->
    async { printfn "Sending email to %s: %s" to' subject }

let consoleLog prefix = fun (message: string) ->
    printfn "[%s] %s" prefix message

// Composition Root - wire everything together
let createRegisterUserFunction connectionString smtpServer =
    registerUser
        (findUserInDb connectionString)
        (saveUserToDb connectionString)
        bcryptHash
        (sendSmtpEmail smtpServer)
        (consoleLog "USER-SERVICE")

// Use it
let registerUserFn = createRegisterUserFunction "Server=localhost;Database=myapp" "smtp.gmail.com"

async {
    let! result = registerUserFn "test@example.com" "Test User" "password123"
    match result with
    | Ok user -> printfn "Registered: %s" user.Id
    | Error e -> printfn "Error: %s" e
} |> Async.RunSynchronously
```

## 5. Reader Monad Approach

```fsharp
// ===== Reader Monad for DI =====
// Thread environment through computation without explicit passing

// Environment (all dependencies bundled)
type AppEnv = {
    UserRepo: IUserRepository
    EmailService: IEmailService
    PasswordHasher: IPasswordHasher
    Logger: ILogger
}

// Reader type
type Reader<'Env, 'T> = Reader of ('Env -> Async<'T>)

module Reader =
    let run (Reader f) env = f env
    
    let ask = Reader (fun env -> async { return env })
    
    let asks (f: 'Env -> 'T) = Reader (fun env -> async { return f env })
    
    let return' x = Reader (fun _ -> async { return x })
    
    let bind (Reader f) (k: 'T -> Reader<'Env, 'U>) =
        Reader (fun env -> async {
            let! a = f env
            let (Reader g) = k a
            return! g env
        })
    
    let map (f: 'T -> 'U) (Reader g) =
        Reader (fun env -> async {
            let! a = g env
            return f a
        })
    
    let liftAsync (op: Async<'T>) : Reader<'Env, 'T> =
        Reader (fun _ -> op)

type ReaderBuilder() =
    member _.Return x = Reader.return' x
    member _.ReturnFrom r = r
    member _.Bind(m, f) = Reader.bind m f
    member _.Zero() = Reader.return' ()

let reader = ReaderBuilder()

// ===== Application Logic using Reader =====
let getLogger = Reader.asks (fun (env: AppEnv) -> env.Logger)
let getUserRepo = Reader.asks (fun env -> env.UserRepo)
let getEmailService = Reader.asks (fun env -> env.EmailService)
let getPasswordHasher = Reader.asks (fun env -> env.PasswordHasher)

let registerUserReader (email: string) (name: string) (password: string) =
    reader {
        let! logger = getLogger
        let! userRepo = getUserRepo
        let! emailService = getEmailService
        let! hasher = getPasswordHasher
        
        logger.Info (sprintf "Registering user: %s" email)
        
        // Validate (inline for simplicity)
        if String.IsNullOrWhiteSpace(email) then
            return Error "Email required"
        else
        
        let! exists = Reader.liftAsync (userRepo.Exists email)
        if exists then
            return Error "Email already registered"
        else
        
        let user = {
            Id = Guid.NewGuid().ToString()
            Email = email.ToLower()
            Name = name
            PasswordHash = hasher.Hash password
            IsActive = true
            CreatedAt = DateTime.UtcNow
        }
        
        do! Reader.liftAsync (userRepo.Save user)
        do! Reader.liftAsync (emailService.Send user.Email "Welcome!" (sprintf "Hello %s!" user.Name))
        
        logger.Info (sprintf "User %s registered" user.Id)
        return Ok user
    }

// Create and run with environment
let runReaderExample () =
    let services = createServiceCollection()
    let provider = services.BuildServiceProvider()
    use scope = provider.CreateScope()
    
    let env = {
        UserRepo = scope.ServiceProvider.GetRequiredService<IUserRepository>()
        EmailService = scope.ServiceProvider.GetRequiredService<IEmailService>()
        PasswordHasher = scope.ServiceProvider.GetRequiredService<IPasswordHasher>()
        Logger = scope.ServiceProvider.GetRequiredService<ILogger>()
    }
    
    let registration = registerUserReader "alice@example.com" "Alice" "password123"
    Reader.run registration env |> Async.RunSynchronously
```

## 6. Record of Functions (Dependency Bundles)

```fsharp
// ===== Record of Functions Pattern =====
// Bundle related dependencies together

type OrderDependencies = {
    FindProduct: string -> Async<{| Id: string; Name: string; Price: decimal; Stock: int |} option>
    SaveOrder: {| Id: string; CustomerId: string; Total: decimal; Items: string list |} -> Async<unit>
    CheckInventory: string -> int -> Async<bool>
    ChargePayment: string -> decimal -> Async<Result<string, string>>
    SendConfirmation: string -> string -> Async<unit>
    Log: string -> unit
}

// Use case function
let placeOrder (deps: OrderDependencies) (customerId: string) (items: (string * int) list) =
    async {
        deps.Log (sprintf "Placing order for customer %s" customerId)
        
        // Check all products exist
        let! productResults =
            items |> List.map (fun (pid, qty) ->
                async {
                    let! product = deps.FindProduct pid
                    return (pid, qty, product)
                }) |> Async.Parallel
        
        let missing = productResults |> Array.choose (fun (pid, _, p) -> if p.IsNone then Some pid else None)
        if missing.Length > 0 then
            return Error (sprintf "Products not found: %s" (String.concat ", " missing))
        else
        
        // Check inventory
        let! stockChecks =
            productResults
            |> Array.choose (fun (pid, qty, p) -> p |> Option.map (fun _ -> (pid, qty)))
            |> Array.map (fun (pid, qty) ->
                async {
                    let! available = deps.CheckInventory pid qty
                    return (pid, qty, available)
                }) |> Async.Parallel
        
        let unavailable = stockChecks |> Array.filter (fun (_, _, avail) -> not avail)
        if unavailable.Length > 0 then
            return Error "Some items are out of stock"
        else
        
        // Calculate total
        let total =
            productResults
            |> Array.sumBy (fun (_, qty, p) ->
                match p with
                | Some prod -> prod.Price * decimal qty
                | None -> 0m)
        
        // Charge payment
        let! paymentResult = deps.ChargePayment customerId total
        match paymentResult with
        | Error e -> return Error (sprintf "Payment failed: %s" e)
        | Ok transactionId ->
        
        // Save order
        let orderId = Guid.NewGuid().ToString()
        let itemDescriptions = productResults |> Array.choose (fun (_, qty, p) -> p |> Option.map (fun prod -> sprintf "%s x%d" prod.Name qty)) |> Array.toList
        do! deps.SaveOrder {| Id = orderId; CustomerId = customerId; Total = total; Items = itemDescriptions |}
        
        // Send confirmation
        do! deps.SendConfirmation customerId (sprintf "Order %s placed! Total: %.2f" orderId total)
        
        deps.Log (sprintf "Order %s placed successfully" orderId)
        return Ok orderId
    }

// Create production dependencies
let createProductionOrderDeps (dbConnection: string) (stripeKey: string) =
    {
        FindProduct = fun productId -> async {
            // Real DB call
            return None  // Placeholder
        }
        
        SaveOrder = fun order -> async {
            printfn "Saving order %s" order.Id
        }
        
        CheckInventory = fun productId qty -> async {
            return true  // Placeholder
        }
        
        ChargePayment = fun customerId amount -> async {
            printfn "Charging %.2f to %s via Stripe" amount customerId
            return Ok (sprintf "stripe-%s" (Guid.NewGuid().ToString("N")[..7]))
        }
        
        SendConfirmation = fun userId message -> async {
            printfn "Email to %s: %s" userId message
        }
        
        Log = fun msg -> printfn "[ORDER] %s" msg
    }

// Create test dependencies
let createTestOrderDeps () =
    let products = Map.ofList [
        ("P001", {| Id = "P001"; Name = "Widget"; Price = 100m; Stock = 50 |})
        ("P002", {| Id = "P002"; Name = "Gadget"; Price = 250m; Stock = 10 |})
    ]
    let savedOrders = ResizeArray()
    
    {
        FindProduct = fun id -> async { return products |> Map.tryFind id }
        
        SaveOrder = fun order -> async { savedOrders.Add(order) }
        
        CheckInventory = fun productId qty -> async {
            return products |> Map.tryFind productId 
                   |> Option.map (fun p -> p.Stock >= qty)
                   |> Option.defaultValue false
        }
        
        ChargePayment = fun _ amount -> async {
            return Ok (sprintf "test-txn-%s" (Guid.NewGuid().ToString("N")[..7]))
        }
        
        SendConfirmation = fun userId msg -> async {
            printfn "[TEST EMAIL] To %s: %s" userId msg
        }
        
        Log = fun msg -> printfn "[TEST LOG] %s" msg
    }, savedOrders
```

## 7. Scrutor สำหรับ Assembly Scanning

```fsharp
// ===== Scrutor Assembly Scanning =====
// NuGet: Scrutor

open Microsoft.Extensions.DependencyInjection

// สมมุติเรามี marker interface
type ITransient = interface end
type IScoped = interface end
type ISingleton = interface end

// Implementations เพียงแค่ implement marker interface
type ConcreteEmailService() =
    interface IEmailService with
        member _.Send to' subject body = async { printfn "Email: %s" subject }
    interface ITransient  // Mark as transient

type ConcreteUserRepository() =
    interface IUserRepository with
        member _.FindById id = async { return None }
        member _.Save user = async { () }
        member _.Exists email = async { return false }
    interface IScoped  // Mark as scoped

// Register all services automatically
let registerServicesWithScrutor (services: IServiceCollection) =
    // Real Scrutor usage:
    (*
    services.Scan(fun scan ->
        scan.FromAssemblyOf<ConcreteEmailService>()
            .AddClasses(fun classes -> 
                classes.AssignableTo<ITransient>())
            .AsImplementedInterfaces()
            .WithTransientLifetime()
            
            .AddClasses(fun classes ->
                classes.AssignableTo<IScoped>())
            .AsImplementedInterfaces()
            .WithScopedLifetime()
    ) |> ignore
    *)
    
    // Manual equivalent
    services.AddTransient<IEmailService, ConcreteEmailService>() |> ignore
    services.AddScoped<IUserRepository, ConcreteUserRepository>() |> ignore
    services
```

## 8. Testing กับ DI

```fsharp
// ===== Testing with DI =====

// Test doubles
type TestEmailService() =
    let sent = ResizeArray<string * string * string>()
    
    interface IEmailService with
        member _.Send to' subject body =
            async { sent.Add(to', subject, body) }
    
    member _.SentEmails = sent |> Seq.toList

type TestUserRepository() =
    let users = System.Collections.Generic.Dictionary<string, User>()
    
    interface IUserRepository with
        member _.FindById id = async {
            match users.TryGetValue(id) with
            | true, u -> return Some u
            | _ -> return None }
        
        member _.Save user = async { users.[user.Id] <- user }
        
        member _.Exists email = async {
            return users.Values |> Seq.exists (fun u -> u.Email = email) }
    
    member _.AllUsers = users.Values |> Seq.toList

// Build test service provider
let buildTestServiceProvider () =
    let emailService = TestEmailService()
    let userRepo = TestUserRepository()
    
    let services = ServiceCollection()
    services.AddSingleton<IEmailService>(emailService :> IEmailService) |> ignore
    services.AddSingleton<IUserRepository>(userRepo :> IUserRepository) |> ignore
    services.AddSingleton<IPasswordHasher, BCryptPasswordHasher>() |> ignore
    services.AddSingleton<ILogger>(ConsoleLogger("Test") :> ILogger) |> ignore
    services.AddTransient<UserService>() |> ignore
    
    services.BuildServiceProvider(), emailService, userRepo

// Tests
let testUserRegistration () =
    async {
        let provider, emailService, userRepo = buildTestServiceProvider()
        use scope = provider.CreateScope()
        let userService = scope.ServiceProvider.GetRequiredService<UserService>()
        
        let! result = userService.RegisterUser "test@example.com" "Test User" "password123"
        
        match result with
        | Ok user ->
            printfn "✓ Registration successful: %s" user.Id
            
            // Verify in repo
            assert (userRepo.AllUsers.Length = 1)
            printfn "✓ User saved to repository"
            
            // Verify email sent
            assert (emailService.SentEmails.Length = 1)
            printfn "✓ Welcome email sent"
            
        | Error e ->
            failwith (sprintf "Registration failed: %s" e)
    }

let testDuplicateRegistration () =
    async {
        let provider, _, _ = buildTestServiceProvider()
        use scope = provider.CreateScope()
        let userService = scope.ServiceProvider.GetRequiredService<UserService>()
        
        // First registration
        let! _ = userService.RegisterUser "test@example.com" "User 1" "password123"
        
        // Duplicate attempt
        let! result = userService.RegisterUser "test@example.com" "User 2" "password456"
        
        match result with
        | Error "Email already registered" ->
            printfn "✓ Duplicate email rejected"
        | _ ->
            failwith "Should have rejected duplicate"
    }

// Run tests
let runAllTests () =
    printfn "=== DI Tests ==="
    [testUserRegistration(); testDuplicateRegistration()]
    |> Async.Sequential
    |> Async.RunSynchronously
    |> ignore
    printfn "✓ All tests passed!"

runAllTests()
```

## 9. Service Locator Anti-Pattern

```fsharp
// ===== Service Locator Anti-Pattern (อย่าทำแบบนี้!) =====

// ❌ Bad: Service Locator - dependencies hidden, not visible in type
type BadService(serviceLocator: IServiceProvider) =
    member _.DoSomething() =
        async {
            // Dependencies hidden - hard to test!
            let emailService = serviceLocator.GetRequiredService<IEmailService>()
            let userRepo = serviceLocator.GetRequiredService<IUserRepository>()
            
            // ...
            ()
        }

// ✅ Good: Constructor injection - dependencies explicit
type GoodService(emailService: IEmailService, userRepo: IUserRepository) =
    member _.DoSomething() =
        async {
            // Dependencies explicit - easy to test!
            // ...
            ()
        }

// ===== Composition Root Anti-Pattern =====
// ❌ Bad: DI configuration spread across application
// ✅ Good: All DI in one place (Program.fs or Startup.fs)

// The composition root should be as close to the entry point as possible
// and contain ALL dependency registration
```

## 10. Keyed Services (ASP.NET Core 8+)

```fsharp
// ===== Keyed Services =====
// New in .NET 8 - inject specific named implementation

// Multiple implementations of same interface
type SmtpEmailServiceV2(server: string) =
    interface IEmailService with
        member _.Send to' subject body = async { printfn "[SMTP] %s" subject }

type SendGridEmailServiceV2(apiKey: string) =
    interface IEmailService with
        member _.Send to' subject body = async { printfn "[SendGrid] %s" subject }

let registerKeyedServices (services: IServiceCollection) =
    // Register with keys
    services.AddKeyedSingleton<IEmailService>("smtp", fun sp _ ->
        SmtpEmailServiceV2("smtp.gmail.com") :> IEmailService) |> ignore
    
    services.AddKeyedSingleton<IEmailService>("sendgrid", fun sp _ ->
        SendGridEmailServiceV2("sg-api-key") :> IEmailService) |> ignore
    
    services

// Inject specific implementation
type MarketingService([<FromKeyedServices("sendgrid")>] emailService: IEmailService) =
    member _.SendCampaign (recipients: string list) (subject: string) (body: string) =
        async {
            for recipient in recipients do
                do! emailService.Send recipient subject body
        }

// Note: FromKeyedServices attribute shown for concept;
// actual attribute is in Microsoft.Extensions.DependencyInjection
```

## 11. สรุป Dependency Injection กับ F#

```fsharp
(*
Dependency Injection Approaches in F#:

1. Microsoft.Extensions.DependencyInjection (IoC Container)
   - Pro: Familiar to .NET developers, integrates with ASP.NET Core
   - Con: Reflection-based, runtime errors possible
   - When: ASP.NET Core applications

2. Function-based DI (Partial Application)
   - Pro: Pure F#, no container needed, compile-time safety
   - Con: Can lead to many parameters
   - When: Small services, functional architecture

3. Reader Monad
   - Pro: Elegant, testable, explicit
   - Con: Learning curve, some ceremony
   - When: Complex applications with many dependencies

4. Record of Functions
   - Pro: Simple, explicit, easy to mock
   - Con: Must define record type
   - When: When related deps belong together

Lifetimes:
- Singleton: Database connections, caches, configuration
- Scoped: DbContext, unit of work, per-request state
- Transient: Lightweight stateless services

Testing:
- Inject test doubles through constructor
- Build test container with mock services
- Don't use Service Locator (anti-pattern)

F# Advantages:
✓ Immutable by default = safer dependencies
✓ Record types = easy dependency bundles
✓ Function types = natural port abstractions
✓ Partial application = built-in DI without containers
*)

printfn "Dependency Injection with F# - Complete!"
```

---

**สรุป**: F# มีหลายแนวทางสำหรับ DI ตั้งแต่ Microsoft.Extensions.DependencyInjection ที่คุ้นเคยสำหรับ .NET developers ไปจนถึง functional approaches อย่าง partial application และ Reader Monad ที่ให้ความ type-safety และ testability ที่สูงกว่า
