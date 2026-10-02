# Part 76 - Mocking กับ F#

## บทนำ (Introduction)

Mocking ใน F# มีความซับซ้อนกว่าใน C# เนื่องจาก F# เน้นการใช้ immutable data และ pure functions แต่ยังมีกรณีที่ต้องใช้ mocking เช่น การทดสอบ code ที่มี side effects หรือ external dependencies

**Test doubles ประเภทต่างๆ:**
- **Stub** - คืนค่าที่กำหนดไว้ ไม่สนใจว่าถูกเรียกอย่างไร
- **Mock** - ตรวจสอบว่าถูกเรียกอย่างไร (verification)
- **Fake** - implementation จริงแต่ simplified สำหรับ tests
- **Spy** - records calls แต่ delegate ไปยัง real implementation

---

## 1. ทำไม Mocking ใน F# ถึงซับซ้อน

```fsharp
// WhyMockingIsComplex.fs
module Tests.WhyMockingIsComplex

// ปัญหา: F# types ส่วนใหญ่เป็น sealed (final)
// Cannot mock: int, string, list, option, Result, DU, records

// ปัญหา: F# functions ไม่ใช่ interfaces
let processData (data: string list) =
    data |> List.map (fun s -> s.ToUpper())

// ไม่สามารถ mock function โดยตรงใน Moq

// วิธีแก้: ใช้ functions เป็น dependencies
let processDataWithDep (transform: string -> string) (data: string list) =
    data |> List.map transform

// ตอนนี้ mock ได้ง่ายๆ
let testTransform = fun s -> s + "_MOCKED"
let result = processDataWithDep testTransform ["hello"; "world"]
// result = ["hello_MOCKED"; "world_MOCKED"]

// ปัญหา: Interfaces ต้องเป็น abstract สำหรับ mocking
// ข้อดีของ F#: Object expressions ทำให้ implement interface ได้ง่าย
```

---

## 2. Object Expressions เป็น Mocks

```fsharp
// ObjectExpressionMocks.fs
module Tests.ObjectExpressionMocks

open Xunit

// Interface ที่จะ mock
type IEmailService =
    abstract member SendEmail: to': string -> subject: string -> body: string -> bool
    abstract member IsAvailable: unit -> bool

type ILogger =
    abstract member Log: level: string -> message: string -> unit
    abstract member LogError: exn -> message: string -> unit

type IUserRepository =
    abstract member FindById: int -> Result<{| Id: int; Name: string; Email: string |}, string>
    abstract member Save: {| Id: int; Name: string; Email: string |} -> bool
    abstract member GetAll: unit -> {| Id: int; Name: string; Email: string |} list

// Object expression mock
let createEmailServiceMock (shouldSucceed: bool) =
    { new IEmailService with
        member _.SendEmail _ _ _ = shouldSucceed
        member _.IsAvailable() = shouldSucceed }

let createLoggerMock () =
    let mutable logs: (string * string) list = []
    
    { new ILogger with
        member _.Log level message = 
            logs <- (level, message) :: logs
        member _.LogError _ message = 
            logs <- ("ERROR", message) :: logs },
    fun () -> logs  // Function to retrieve logs

let createUserRepositoryMock (users: Map<int, {| Id: int; Name: string; Email: string |}>) =
    let mutable savedUsers = users
    
    { new IUserRepository with
        member _.FindById id =
            match savedUsers |> Map.tryFind id with
            | Some user -> Ok user
            | None -> Error (sprintf "User %d not found" id)
        
        member _.Save user =
            savedUsers <- savedUsers |> Map.add user.Id user
            true
        
        member _.GetAll() =
            savedUsers |> Map.toList |> List.map snd }

// Service ที่จะทดสอบ
type UserService(emailSvc: IEmailService, logger: ILogger, repo: IUserRepository) =
    
    member _.RegisterUser name email =
        let id = System.Math.Abs(email.GetHashCode())
        let user = {| Id = id; Name = name; Email = email |}
        
        logger.Log "INFO" (sprintf "Registering user: %s" email)
        
        let saved = repo.Save user
        
        if saved then
            let emailSent = emailSvc.SendEmail email "Welcome!" (sprintf "Welcome, %s!" name)
            if emailSent then
                logger.Log "INFO" (sprintf "Welcome email sent to %s" email)
                Ok user
            else
                logger.Log "WARN" (sprintf "Failed to send email to %s" email)
                Ok user  // Still success, just email failed
        else
            logger.Log "ERROR" (sprintf "Failed to save user %s" email)
            Error "Failed to save user"

// Tests กับ object expression mocks
[<Fact>]
let ``registerUser saves user and sends email`` () =
    let emailMock = createEmailServiceMock true
    let (loggerMock, getLogs) = createLoggerMock()
    let repoMock = createUserRepositoryMock Map.empty
    
    let svc = UserService(emailMock, loggerMock, repoMock)
    let result = svc.RegisterUser "Alice" "alice@example.com"
    
    match result with
    | Ok user ->
        Assert.Equal("Alice", user.Name)
        Assert.Equal("alice@example.com", user.Email)
    | Error msg ->
        Assert.Fail(sprintf "Expected Ok but got Error: %s" msg)
    
    // Verify logs were created
    let logs = getLogs()
    Assert.NotEmpty(logs)

[<Fact>]
let ``registerUser logs appropriate messages`` () =
    let emailMock = createEmailServiceMock true
    let (loggerMock, getLogs) = createLoggerMock()
    let repoMock = createUserRepositoryMock Map.empty
    
    let svc = UserService(emailMock, loggerMock, repoMock)
    let _ = svc.RegisterUser "Bob" "bob@example.com"
    
    let logs = getLogs()
    
    // Check that registration was logged
    let hasRegistrationLog = logs |> List.exists (fun (_, msg) -> msg.Contains("bob@example.com"))
    Assert.True(hasRegistrationLog)

[<Fact>]
let ``registerUser handles email failure gracefully`` () =
    let emailMock = createEmailServiceMock false  // Email fails
    let (loggerMock, getLogs) = createLoggerMock()
    let repoMock = createUserRepositoryMock Map.empty
    
    let svc = UserService(emailMock, loggerMock, repoMock)
    let result = svc.RegisterUser "Charlie" "charlie@example.com"
    
    // Should still succeed even if email fails
    Assert.True(Result.isOk result)
    
    // Warning should be logged
    let logs = getLogs()
    let hasWarning = logs |> List.exists (fun (level, _) -> level = "WARN")
    Assert.True(hasWarning)
```

---

## 3. FakeItEasy Library

```fsharp
// FakeItEasyTests.fs
module Tests.FakeItEasyTests

open Xunit
open FakeItEasy

// Interface to mock
type IProductRepository =
    abstract member GetById: int -> {| Id: int; Name: string; Price: float |} option
    abstract member GetAll: unit -> {| Id: int; Name: string; Price: float |} list
    abstract member Create: name: string -> price: float -> {| Id: int; Name: string; Price: float |}
    abstract member Delete: int -> bool

// Basic FakeItEasy usage
[<Fact>]
let ``create fake returns default values`` () =
    let repo = A.Fake<IProductRepository>()
    
    // Default: returns null/default for reference types
    let result = repo.GetById(1)
    Assert.True(result.IsNone)  // Default for option

[<Fact>]
let ``configure fake to return specific values`` () =
    let repo = A.Fake<IProductRepository>()
    
    let expectedProduct = Some {| Id = 1; Name = "Widget"; Price = 9.99 |}
    
    A.CallTo(fun () -> repo.GetById(1))
        .Returns(expectedProduct)
    |> ignore
    
    let result = repo.GetById(1)
    Assert.Equal(expectedProduct, result)

[<Fact>]
let ``configure fake to return different values for different args`` () =
    let repo = A.Fake<IProductRepository>()
    
    let product1 = Some {| Id = 1; Name = "Widget"; Price = 9.99 |}
    let product2 = Some {| Id = 2; Name = "Gadget"; Price = 19.99 |}
    
    A.CallTo(fun () -> repo.GetById(1)).Returns(product1) |> ignore
    A.CallTo(fun () -> repo.GetById(2)).Returns(product2) |> ignore
    
    Assert.Equal(product1, repo.GetById(1))
    Assert.Equal(product2, repo.GetById(2))
    Assert.Equal(None, repo.GetById(99))  // Not configured = default

[<Fact>]
let ``verify method was called`` () =
    let repo = A.Fake<IProductRepository>()
    
    A.CallTo(fun () -> repo.Delete(A<int>.Ignored))
        .Returns(true)
    |> ignore
    
    let _ = repo.Delete(5)
    
    // Verify Delete was called exactly once
    A.CallTo(fun () -> repo.Delete(A<int>.Ignored))
        .MustHaveHappenedOnceExactly()

[<Fact>]
let ``verify method was called with specific args`` () =
    let repo = A.Fake<IProductRepository>()
    
    A.CallTo(fun () -> repo.Delete(5)).Returns(true) |> ignore
    
    let _ = repo.Delete(5)
    
    // Verify called with specific arg
    A.CallTo(fun () -> repo.Delete(5))
        .MustHaveHappenedOnceExactly()

[<Fact>]
let ``verify method was never called`` () =
    let repo = A.Fake<IProductRepository>()
    
    // Don't call anything
    
    A.CallTo(fun () -> repo.Delete(A<int>.Ignored))
        .MustNotHaveHappened()

[<Fact>]
let ``configure fake to throw exception`` () =
    let repo = A.Fake<IProductRepository>()
    
    A.CallTo(fun () -> repo.GetById(999))
        .Throws<System.InvalidOperationException>("Item not found")
    |> ignore
    
    Assert.Throws<System.InvalidOperationException>(fun () ->
        repo.GetById(999) |> ignore
    )
```

---

## 4. NSubstitute Library

```fsharp
// NSubstituteTests.fs
module Tests.NSubstituteTests

open Xunit
open NSubstitute

type ICalculationService =
    abstract member Calculate: int -> int -> string -> float
    abstract member Validate: float -> bool

[<Fact>]
let ``nsubstitute basic usage`` () =
    let svc = Substitute.For<ICalculationService>()
    
    // Configure
    svc.Calculate(5, 3, "add").Returns(8.0) |> ignore
    svc.Calculate(5, 3, "multiply").Returns(15.0) |> ignore
    
    // Use
    let addResult = svc.Calculate(5, 3, "add")
    let mulResult = svc.Calculate(5, 3, "multiply")
    
    Assert.Equal(8.0, addResult)
    Assert.Equal(15.0, mulResult)

[<Fact>]
let ``nsubstitute received calls verification`` () =
    let svc = Substitute.For<ICalculationService>()
    svc.Validate(Arg.Any<float>()).Returns(true) |> ignore
    
    let _ = svc.Validate(3.14)
    let _ = svc.Validate(2.71)
    
    // Received(2) - called exactly 2 times
    svc.Received(2).Validate(Arg.Any<float>()) |> ignore
    
    // Received with specific arg
    svc.Received(1).Validate(3.14) |> ignore

[<Fact>]
let ``nsubstitute argument matchers`` () =
    let svc = Substitute.For<ICalculationService>()
    
    // Match any positive number
    svc.Calculate(Arg.Is<int>(fun x -> x > 0), Arg.Any<int>(), Arg.Any<string>())
        .Returns(100.0)
    |> ignore
    
    let result = svc.Calculate(5, 3, "any")
    Assert.Equal(100.0, result)

[<Fact>]
let ``nsubstitute sequential returns`` () =
    let svc = Substitute.For<ICalculationService>()
    
    // Return different values on successive calls
    svc.Validate(Arg.Any<float>())
        .Returns(true, false, true)
    |> ignore
    
    Assert.True(svc.Validate(1.0))   // First call: true
    Assert.False(svc.Validate(2.0))  // Second call: false
    Assert.True(svc.Validate(3.0))   // Third call: true

[<Fact>]
let ``nsubstitute callback on call`` () =
    let svc = Substitute.For<ICalculationService>()
    let mutable callLog: string list = []
    
    svc.Calculate(Arg.Any<int>(), Arg.Any<int>(), Arg.Any<string>())
        .Returns(fun callInfo ->
            let op = callInfo.ArgAt<string>(2)
            callLog <- op :: callLog
            0.0
        )
    |> ignore
    
    let _ = svc.Calculate(1, 2, "add")
    let _ = svc.Calculate(3, 4, "subtract")
    
    Assert.Equal(2, callLog.Length)
    Assert.Contains("add", callLog)
    Assert.Contains("subtract", callLog)
```

---

## 5. Moq Library

```fsharp
// MoqTests.fs
module Tests.MoqTests

open Xunit
open Moq

type INotificationService =
    abstract member Notify: userId: int -> message: string -> bool
    abstract member NotifyAll: message: string -> int  // Returns count sent

[<Fact>]
let ``moq basic setup and verify`` () =
    let mock = Mock<INotificationService>()
    
    mock.Setup(fun svc -> svc.Notify(It.IsAny<int>(), It.IsAny<string>()))
        .Returns(true)
    |> ignore
    
    let svc = mock.Object
    let result = svc.Notify(1, "Hello")
    
    Assert.True(result)
    
    mock.Verify(fun svc -> svc.Notify(1, "Hello"), Times.Once())

[<Fact>]
let ``moq with specific argument`` () =
    let mock = Mock<INotificationService>()
    
    mock.Setup(fun svc -> svc.Notify(42, It.IsAny<string>()))
        .Returns(true)
    |> ignore
    
    mock.Setup(fun svc -> svc.Notify(It.Is<int>(fun x -> x <> 42), It.IsAny<string>()))
        .Returns(false)
    |> ignore
    
    let svc = mock.Object
    
    Assert.True(svc.Notify(42, "test"))
    Assert.False(svc.Notify(1, "test"))

[<Fact>]
let ``moq throws exception`` () =
    let mock = Mock<INotificationService>()
    
    mock.Setup(fun svc -> svc.NotifyAll(It.IsAny<string>()))
        .Throws<System.Exception>("Service unavailable")
    |> ignore
    
    let svc = mock.Object
    
    Assert.Throws<System.Exception>(fun () ->
        svc.NotifyAll("message") |> ignore
    )

[<Fact>]
let ``moq verifies call count`` () =
    let mock = Mock<INotificationService>()
    mock.Setup(fun svc -> svc.Notify(It.IsAny<int>(), It.IsAny<string>()))
        .Returns(true)
    |> ignore
    
    let svc = mock.Object
    
    for i in 1..3 do
        svc.Notify(i, "message") |> ignore
    
    mock.Verify(
        fun svc -> svc.Notify(It.IsAny<int>(), It.IsAny<string>()),
        Times.Exactly(3)
    )

[<Fact>]
let ``moq strict behavior`` () =
    // Strict mock - throws on unexpected calls
    let mock = Mock<INotificationService>(MockBehavior.Strict)
    
    mock.Setup(fun svc -> svc.Notify(1, "Hello"))
        .Returns(true)
    |> ignore
    
    let svc = mock.Object
    
    // This call is configured - works fine
    let result = svc.Notify(1, "Hello")
    Assert.True(result)
    
    // Unconfigured call would throw MockException
```

---

## 6. สร้าง Test Doubles เอง

```fsharp
// ManualTestDoubles.fs
module Tests.ManualTestDoubles

open Xunit

// Interface
type IEmailSender =
    abstract member Send: to': string -> subject: string -> body: string -> unit
    abstract member SendBulk: recipients: string list -> subject: string -> body: string -> int

// Stub - คืนค่าที่กำหนดไว้ เรียบง่าย
let emailSenderStub = {
    new IEmailSender with
        member _.Send _ _ _ = ()  // Does nothing
        member _.SendBulk recipients _ _ = recipients.Length  // Returns count
}

// Spy - บันทึก calls
type EmailSenderSpy() =
    let mutable sentEmails: (string * string * string) list = []
    let mutable bulkSends: (string list * string * string) list = []
    
    interface IEmailSender with
        member _.Send to' subject body = 
            sentEmails <- (to', subject, body) :: sentEmails
        
        member _.SendBulk recipients subject body =
            bulkSends <- (recipients, subject, body) :: bulkSends
            recipients.Length
    
    member _.SentEmails = sentEmails
    member _.BulkSends = bulkSends
    member _.TotalEmailsSent = 
        sentEmails.Length + (bulkSends |> List.sumBy (fun (r, _, _) -> r.Length))

// Fake - ทำงานจริงแต่ simplified
type FakeEmailSender() =
    let inbox = System.Collections.Generic.Dictionary<string, (string * string) list>()
    
    interface IEmailSender with
        member _.Send to' subject body =
            let current = 
                match inbox.TryGetValue(to') with
                | true, msgs -> msgs
                | false, _ -> []
            inbox.[to'] <- (subject, body) :: current
        
        member _.SendBulk recipients subject body =
            for r in recipients do
                (this :> IEmailSender).Send r subject body
            recipients.Length
    
    member _.GetInbox (email: string) =
        match inbox.TryGetValue(email) with
        | true, msgs -> msgs
        | false, _ -> []
    
    member _.TotalMessages =
        inbox.Values |> Seq.sumBy List.length

// Service that uses email
type WelcomeService(emailSender: IEmailSender) =
    member _.WelcomeUser name email =
        emailSender.Send email "Welcome!" (sprintf "Welcome to the platform, %s!" name)
    
    member _.WelcomeAll users =
        let recipients = users |> List.map snd
        emailSender.SendBulk recipients "Welcome to our platform!" "We're glad to have you!"

// Tests กับ manual doubles
[<Fact>]
let ``welcomeUser sends email to correct recipient`` () =
    let spy = EmailSenderSpy()
    let svc = WelcomeService(spy)
    
    svc.WelcomeUser "Alice" "alice@example.com"
    
    Assert.Equal(1, spy.TotalEmailsSent)
    
    match spy.SentEmails with
    | [(to', subject, _)] ->
        Assert.Equal("alice@example.com", to')
        Assert.Equal("Welcome!", subject)
    | _ ->
        Assert.Fail("Expected exactly one email sent")

[<Fact>]
let ``welcomeAll sends to all users`` () =
    let fake = FakeEmailSender()
    let svc = WelcomeService(fake)
    
    let users = [
        ("Alice", "alice@example.com")
        ("Bob", "bob@example.com")
        ("Charlie", "charlie@example.com")
    ]
    
    svc.WelcomeAll users
    
    Assert.Equal(3, fake.TotalMessages)
    
    // Verify each user got an email
    for (_, email) in users do
        let inbox = fake.GetInbox email
        Assert.NotEmpty(inbox)

[<Fact>]
let ``stub ignores all calls`` () =
    let svc = WelcomeService(emailSenderStub)
    
    // Should not throw
    svc.WelcomeUser "Test" "test@example.com"
    Assert.True(true, "Completed without error")
```

---

## 7. Testing กับ Functional Style (ไม่ต้องใช้ Mocking)

```fsharp
// FunctionalTestingWithoutMocks.fs
module Tests.FunctionalTestingWithoutMocks

open Xunit

// ข้อดีของ functional style:
// Pure functions ไม่ต้องการ mocking เลย!

// Bad: stateful approach (ต้องใช้ mock)
// type UserService(db: IDatabase, email: IEmailService) =
//     member _.Register name email = ...

// Good: functional approach (ไม่ต้องใช้ mock)
module UserRegistration =
    type RegistrationError =
        | InvalidEmail of string
        | DuplicateEmail of string
        | SaveFailed of string
    
    type User = { Name: string; Email: string }
    
    // Pure validation
    let validateEmail (email: string) =
        if email.Contains("@") && email.Contains(".")
        then Ok email
        else Error (InvalidEmail email)
    
    // Pure business logic
    let createUser name email existingEmails =
        validateEmail email
        |> Result.bind (fun validEmail ->
            if existingEmails |> List.contains validEmail then
                Error (DuplicateEmail validEmail)
            else
                Ok { Name = name; Email = validEmail }
        )
    
    // Pure email content generation
    let createWelcomeEmail user =
        {| 
            To = user.Email
            Subject = "Welcome!"
            Body = sprintf "Welcome, %s!" user.Name 
        |}
    
    // Orchestration: compose pure functions with effects at the edges
    let registerUser 
        (getExistingEmails: unit -> string list)  // Effect
        (saveUser: User -> Result<User, string>)   // Effect
        (sendEmail: {| To: string; Subject: string; Body: string |} -> bool)  // Effect
        name email =
        
        // Get existing emails (effect)
        let existing = getExistingEmails()
        
        // Pure: validate and create user
        createUser name email existing
        |> Result.bind (fun user ->
            // Effect: save user
            saveUser user
            |> Result.mapError SaveFailed
        )
        |> Result.map (fun savedUser ->
            // Pure: create email content
            let email = createWelcomeEmail savedUser
            // Effect: send email (don't fail registration if email fails)
            sendEmail email |> ignore
            savedUser
        )

// Tests สำหรับ pure functions - ไม่ต้องใช้ mocking เลย!
[<Fact>]
let ``validateEmail accepts valid email`` () =
    let result = UserRegistration.validateEmail "test@example.com"
    Assert.Equal(Ok "test@example.com", result)

[<Fact>]
let ``validateEmail rejects invalid email`` () =
    let result = UserRegistration.validateEmail "not-an-email"
    match result with
    | Error (UserRegistration.InvalidEmail _) -> ()
    | _ -> Assert.Fail("Expected InvalidEmail error")

[<Fact>]
let ``createUser succeeds with valid unique email`` () =
    let existingEmails = ["existing@example.com"]
    let result = UserRegistration.createUser "Alice" "alice@example.com" existingEmails
    
    match result with
    | Ok user -> Assert.Equal("Alice", user.Name)
    | Error e -> Assert.Fail(sprintf "Expected Ok but got Error: %A" e)

[<Fact>]
let ``createUser fails with duplicate email`` () =
    let existingEmails = ["alice@example.com"]
    let result = UserRegistration.createUser "Alice" "alice@example.com" existingEmails
    
    match result with
    | Error (UserRegistration.DuplicateEmail email) -> 
        Assert.Equal("alice@example.com", email)
    | _ -> Assert.Fail("Expected DuplicateEmail error")

[<Fact>]
let ``createWelcomeEmail has correct content`` () =
    let user = { UserRegistration.Name = "Bob"; UserRegistration.Email = "bob@example.com" }
    let email = UserRegistration.createWelcomeEmail user
    
    Assert.Equal("bob@example.com", email.To)
    Assert.Equal("Welcome!", email.Subject)
    Assert.Contains("Bob", email.Body)

// Tests สำหรับ orchestration - minimal mocking
[<Fact>]
let ``registerUser orchestrates correctly`` () =
    // Simple in-memory implementations (fakes, not mocks)
    let existingEmails = ["existing@example.com"]
    let savedUsers = System.Collections.Generic.List<UserRegistration.User>()
    let sentEmails = System.Collections.Generic.List<{| To: string; Subject: string; Body: string |}>()
    
    let getExisting = fun () -> existingEmails
    let saveUser = fun user ->
        savedUsers.Add(user)
        Ok user
    let sendEmail = fun email ->
        sentEmails.Add(email)
        true
    
    let result = UserRegistration.registerUser getExisting saveUser sendEmail "Alice" "alice@example.com"
    
    match result with
    | Ok user ->
        Assert.Equal("Alice", user.Name)
        Assert.Equal(1, savedUsers.Count)
        Assert.Equal(1, sentEmails.Count)
    | Error e ->
        Assert.Fail(sprintf "Expected Ok but got Error: %A" e)
```

---

## 8. Dependency Injection สำหรับ Testability

```fsharp
// DependencyInjectionTests.fs
module Tests.DependencyInjectionTests

open Xunit
open Microsoft.Extensions.DependencyInjection

// Interfaces
type ITimeProvider =
    abstract member Now: unit -> System.DateTime
    abstract member UtcNow: unit -> System.DateTime

type IIdGenerator =
    abstract member NewId: unit -> System.Guid
    abstract member NewIntId: unit -> int

type ICache<'k, 'v when 'k : comparison> =
    abstract member Get: 'k -> 'v option
    abstract member Set: 'k -> 'v -> unit
    abstract member Remove: 'k -> unit

// Real implementations
type SystemTimeProvider() =
    interface ITimeProvider with
        member _.Now() = System.DateTime.Now
        member _.UtcNow() = System.DateTime.UtcNow

type GuidIdGenerator() =
    interface IIdGenerator with
        member _.NewId() = System.Guid.NewGuid()
        member _.NewIntId() = abs (System.Guid.NewGuid().GetHashCode())

type InMemoryCache<'k, 'v when 'k : comparison>() =
    let mutable storage = Map.empty<'k, 'v>
    
    interface ICache<'k, 'v> with
        member _.Get key = Map.tryFind key storage
        member _.Set key value = storage <- Map.add key value storage
        member _.Remove key = storage <- Map.remove key storage

// Service using DI
type OrderProcessor(
    timeProvider: ITimeProvider,
    idGenerator: IIdGenerator) =
    
    member _.CreateOrder customerId items =
        {| 
            Id = idGenerator.NewIntId()
            CustomerId = customerId
            Items = items
            CreatedAt = timeProvider.UtcNow()
            Status = "pending"
        |}

// Test doubles using object expressions
let fixedTime = System.DateTime(2024, 1, 15, 10, 0, 0)
let fixedTimeProvider = {
    new ITimeProvider with
        member _.Now() = fixedTime
        member _.UtcNow() = fixedTime
}

let mutable nextId = 1000
let sequentialIdGenerator = {
    new IIdGenerator with
        member _.NewId() = System.Guid.NewGuid()
        member _.NewIntId() = 
            let id = nextId
            nextId <- nextId + 1
            id
}

// DI container setup for tests
let createTestServiceProvider () =
    let services = ServiceCollection()
    
    services.AddSingleton<ITimeProvider>(fixedTimeProvider) |> ignore
    services.AddSingleton<IIdGenerator>(sequentialIdGenerator) |> ignore
    services.AddSingleton<OrderProcessor>() |> ignore
    
    services.BuildServiceProvider()

// Tests
[<Fact>]
let ``createOrder uses fixed time in tests`` () =
    let processor = OrderProcessor(fixedTimeProvider, sequentialIdGenerator)
    let order = processor.CreateOrder 1 ["item1"; "item2"]
    
    Assert.Equal(fixedTime, order.CreatedAt)

[<Fact>]
let ``createOrder generates sequential ids`` () =
    nextId <- 100  // Reset
    let processor = OrderProcessor(fixedTimeProvider, sequentialIdGenerator)
    
    let order1 = processor.CreateOrder 1 ["item1"]
    let order2 = processor.CreateOrder 2 ["item2"]
    
    Assert.Equal(100, order1.Id)
    Assert.Equal(101, order2.Id)

[<Fact>]
let ``di container resolves dependencies`` () =
    let sp = createTestServiceProvider()
    let processor = sp.GetRequiredService<OrderProcessor>()
    
    let order = processor.CreateOrder 1 ["item1"]
    
    Assert.Equal(fixedTime, order.CreatedAt)
    Assert.Equal("pending", order.Status)
```

---

## 9. Functions เป็น Dependencies

```fsharp
// FunctionDependencies.fs
module Tests.FunctionDependencies

open Xunit

// แทนที่ interfaces ด้วย functions
// ทำให้ testing ง่ายขึ้นมาก

type Dependencies = {
    GetUser: int -> {| Id: int; Name: string; Email: string |} option
    SaveUser: {| Id: int; Name: string; Email: string |} -> bool
    SendNotification: string -> string -> bool
    Log: string -> string -> unit
    GetCurrentTime: unit -> System.DateTime
}

// Service ที่รับ dependencies เป็น record of functions
module UserService =
    type UpdateResult =
        | Updated of {| Id: int; Name: string; Email: string |}
        | NotFound
        | SaveFailed

    let updateUserName (deps: Dependencies) userId newName =
        deps.Log "INFO" (sprintf "Updating user %d name to %s" userId newName)
        
        match deps.GetUser userId with
        | None ->
            deps.Log "WARN" (sprintf "User %d not found" userId)
            NotFound
        
        | Some user ->
            let updated = {| user with Name = newName |}
            
            if deps.SaveUser updated then
                deps.SendNotification user.Email 
                    (sprintf "Your name has been updated to %s" newName)
                |> ignore
                Updated updated
            else
                deps.Log "ERROR" (sprintf "Failed to save user %d" userId)
                SaveFailed

// Simple test doubles using functions
let createTestDeps () =
    let users = System.Collections.Generic.Dictionary<int, {| Id: int; Name: string; Email: string |}>()
    users.[1] <- {| Id = 1; Name = "Alice"; Email = "alice@example.com" |}
    users.[2] <- {| Id = 2; Name = "Bob"; Email = "bob@example.com" |}
    
    let logs = System.Collections.Generic.List<string * string>()
    let notifications = System.Collections.Generic.List<string * string>()
    
    let deps = {
        GetUser = fun id ->
            match users.TryGetValue(id) with
            | true, user -> Some user
            | false, _ -> None
        
        SaveUser = fun user ->
            users.[user.Id] <- user
            true
        
        SendNotification = fun email msg ->
            notifications.Add((email, msg))
            true
        
        Log = fun level msg ->
            logs.Add((level, msg))
        
        GetCurrentTime = fun () -> System.DateTime.UtcNow
    }
    
    deps, users, logs, notifications

// Tests
[<Fact>]
let ``updateUserName updates existing user`` () =
    let (deps, users, logs, _) = createTestDeps()
    
    let result = UserService.updateUserName deps 1 "Alicia"
    
    match result with
    | UserService.Updated user ->
        Assert.Equal("Alicia", user.Name)
        Assert.Equal(1, user.Id)
    | _ ->
        Assert.Fail("Expected Updated result")
    
    Assert.Equal("Alicia", users.[1].Name)

[<Fact>]
let ``updateUserName returns NotFound for missing user`` () =
    let (deps, _, _, _) = createTestDeps()
    
    let result = UserService.updateUserName deps 999 "New Name"
    
    Assert.Equal(UserService.NotFound, result)

[<Fact>]
let ``updateUserName sends notification on success`` () =
    let (deps, _, _, notifications) = createTestDeps()
    
    let _ = UserService.updateUserName deps 1 "Alicia"
    
    Assert.Equal(1, notifications.Count)
    let (email, msg) = notifications.[0]
    Assert.Equal("alice@example.com", email)
    Assert.Contains("Alicia", msg)

[<Fact>]
let ``updateUserName logs appropriately`` () =
    let (deps, _, logs, _) = createTestDeps()
    
    let _ = UserService.updateUserName deps 1 "New Name"
    
    Assert.NotEmpty(logs)
    let hasInfoLog = logs |> Seq.exists (fun (level, _) -> level = "INFO")
    Assert.True(hasInfoLog)
```

---

## สรุป (Summary)

Mocking ใน F# มีหลายวิธี:

1. **Object expressions** - F#-native, ไม่ต้องใช้ library
2. **FakeItEasy** - readable syntax, ดีสำหรับ .NET interfaces
3. **NSubstitute** - clean API สำหรับ substitutes
4. **Moq** - มาตรฐาน .NET mocking
5. **Manual doubles** - ควบคุมได้มาก, explicit
6. **Functional style** - หลีกเลี่ยง mocking ด้วย pure functions
7. **Function dependencies** - inject functions แทน interfaces

**Best practice สำหรับ F#:**
- ใช้ pure functions ให้มากที่สุด - ไม่ต้องการ mocking
- ใช้ object expressions สำหรับ interfaces
- inject functions แทน service interfaces เมื่อทำได้
- ใช้ mocking library เฉพาะเมื่อจำเป็นจริงๆ
