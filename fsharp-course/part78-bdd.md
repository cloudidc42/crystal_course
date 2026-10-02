# Part 78 - Behavior-Driven Development (BDD)

## บทนำ (Introduction)

Behavior-Driven Development (BDD) เป็นการพัฒนาซอฟต์แวร์ที่เน้นการสื่อสารระหว่างนักพัฒนา, testers, และ stakeholders ผ่าน "scenarios" ที่เขียนเป็นภาษาธรรมชาติ

**BDD ต่างจาก TDD อย่างไร:**
- TDD: Developer-centric, เน้น implementation details
- BDD: Business-centric, เน้น business behavior
- BDD ใช้ Gherkin syntax: Given/When/Then

---

## 1. BDD Concepts

```
Feature: User authentication
  As a user
  I want to log into the system
  So that I can access my account

  Scenario: Successful login
    Given I have a valid account
    When I enter correct credentials
    Then I should be logged in

  Scenario: Failed login
    Given I have a valid account
    When I enter incorrect password
    Then I should see an error message
```

**คำสำคัญ:**
- **Feature** - ฟีเจอร์ที่กำลังทดสอบ
- **Scenario** - สถานการณ์เฉพาะ
- **Given** - สถานการณ์เริ่มต้น (Context)
- **When** - เหตุการณ์ที่เกิดขึ้น (Action)
- **Then** - ผลลัพธ์ที่คาดหวัง (Outcome)

---

## 2. TickSpec สำหรับ F#

### การติดตั้ง

```xml
<!-- BDD.Tests.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.3">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <!-- TickSpec for BDD -->
    <PackageReference Include="TickSpec" Version="2.1.0" />
  </ItemGroup>

  <ItemGroup>
    <!-- Feature files -->
    <EmbeddedResource Include="Features/**/*.feature" />
    
    <Compile Include="Steps/BankingSteps.fs" />
    <Compile Include="Steps/ShoppingSteps.fs" />
    <Compile Include="Steps/AuthSteps.fs" />
    <Compile Include="BDDRunner.fs" />
  </ItemGroup>
</Project>
```

### โครงสร้าง Project

```
BDD.Tests/
├── Features/
│   ├── Banking.feature
│   ├── Shopping.feature
│   └── Authentication.feature
├── Steps/
│   ├── BankingSteps.fs
│   ├── ShoppingSteps.fs
│   └── AuthSteps.fs
└── BDDRunner.fs
```

---

## 3. Feature Files

### Banking.feature

```gherkin
Feature: Banking Operations
  As a bank customer
  I want to manage my account
  So that I can save and spend money

  Background:
    Given I have a bank account with balance 1000

  Scenario: Successful deposit
    When I deposit 500
    Then my balance should be 1500

  Scenario: Successful withdrawal
    When I withdraw 300
    Then my balance should be 700

  Scenario: Withdrawal with insufficient funds
    When I try to withdraw 2000
    Then I should get an error "Insufficient funds"
    And my balance should remain 1000

  Scenario: Transfer between accounts
    Given I have a second account with balance 500
    When I transfer 200 from my first account to my second account
    Then my first account balance should be 800
    And my second account balance should be 700

  Scenario Outline: Multiple deposits
    When I deposit <amount>
    Then my balance should be <expected_balance>

    Examples:
      | amount | expected_balance |
      | 100    | 1100             |
      | 500    | 1500             |
      | 1000   | 2000             |
```

### Shopping.feature

```gherkin
Feature: Shopping Cart
  As a customer
  I want to manage my shopping cart
  So that I can purchase products

  Background:
    Given an empty shopping cart

  Scenario: Add single item
    When I add 1 "Widget" at price 9.99
    Then the cart should contain 1 item
    And the cart total should be 9.99

  Scenario: Add multiple items
    When I add 2 "Widget" at price 9.99
    And I add 1 "Gadget" at price 24.99
    Then the cart should contain 2 items
    And the cart total should be 44.97

  Scenario: Remove item from cart
    Given I have added 1 "Widget" at price 9.99
    And I have added 1 "Gadget" at price 24.99
    When I remove "Widget" from the cart
    Then the cart should contain 1 item
    And the cart total should be 24.99

  Scenario: Apply discount
    Given I have added 1 "Widget" at price 100.00
    When I apply a 10% discount
    Then the cart total should be 90.00

  Scenario: Empty cart
    When I clear the cart
    Then the cart should be empty
    And the cart total should be 0.00
```

### Authentication.feature

```gherkin
Feature: User Authentication
  As a user
  I want to authenticate with the system
  So that I can access my account

  Scenario: Successful login
    Given a user "alice@example.com" with password "SecurePass123"
    When I login with "alice@example.com" and "SecurePass123"
    Then I should be logged in
    And I should receive an access token

  Scenario: Login with wrong password
    Given a user "alice@example.com" with password "SecurePass123"
    When I login with "alice@example.com" and "WrongPassword"
    Then I should not be logged in
    And I should see error "Invalid credentials"

  Scenario: Login with unknown email
    When I login with "unknown@example.com" and "anypassword"
    Then I should not be logged in
    And I should see error "User not found"

  Scenario: Password must meet complexity requirements
    When I try to register with password "weak"
    Then registration should fail
    And I should see error "Password does not meet requirements"

  Scenario Outline: Password complexity validation
    When I try to register with password "<password>"
    Then registration result should be "<result>"

    Examples:
      | password    | result  |
      | weak        | failed  |
      | LongEnough1 | success |
      | NoDigits!!! | failed  |
      | Short1!     | failed  |
```

---

## 4. Step Definitions ใน F#

```fsharp
// Steps/BankingSteps.fs
module BDD.Tests.BankingSteps

open TickSpec
open Xunit
open System.Collections.Generic

// Domain types
type Account = { mutable Balance: float; Id: string }

// State ที่แบ่งปันระหว่าง steps
type BankingContext() =
    member val FirstAccount: Account = { Balance = 0.0; Id = "ACC-001" } with get, set
    member val SecondAccount: Account option = None with get, set
    member val LastError: string option = None with get, set

// Context provider สำหรับ TickSpec
let context = BankingContext()

// =====================
// GIVEN steps
// =====================

[<Given("I have a bank account with balance (.*)")]>]
let ``given account with balance`` (balance: float) =
    context.FirstAccount <- { Balance = balance; Id = "ACC-001" }
    context.LastError <- None

[<Given("I have a second account with balance (.*)")]>]
let ``given second account with balance`` (balance: float) =
    context.SecondAccount <- Some { Balance = balance; Id = "ACC-002" }

// =====================
// WHEN steps
// =====================

[<When("I deposit (.*)")]>]
let ``when deposit`` (amount: float) =
    context.FirstAccount <- { context.FirstAccount with Balance = context.FirstAccount.Balance + amount }

[<When("I withdraw (.*)")]>]
let ``when withdraw`` (amount: float) =
    if context.FirstAccount.Balance >= amount then
        context.FirstAccount <- { context.FirstAccount with Balance = context.FirstAccount.Balance - amount }
    else
        context.LastError <- Some "Insufficient funds"

[<When("I try to withdraw (.*)")]>]
let ``when try to withdraw`` (amount: float) =
    if context.FirstAccount.Balance < amount then
        context.LastError <- Some "Insufficient funds"

[<When("I transfer (.*) from my first account to my second account")]>]
let ``when transfer`` (amount: float) =
    match context.SecondAccount with
    | Some second ->
        if context.FirstAccount.Balance >= amount then
            context.FirstAccount <- { context.FirstAccount with Balance = context.FirstAccount.Balance - amount }
            context.SecondAccount <- Some { second with Balance = second.Balance + amount }
        else
            context.LastError <- Some "Insufficient funds for transfer"
    | None ->
        context.LastError <- Some "Second account not found"

// =====================
// THEN steps
// =====================

[<Then("my balance should be (.*)")]>]
let ``then balance should be`` (expected: float) =
    Assert.Equal(expected, context.FirstAccount.Balance, 2)

[<Then("my balance should remain (.*)")]>]
let ``then balance should remain`` (expected: float) =
    Assert.Equal(expected, context.FirstAccount.Balance, 2)

[<Then("my first account balance should be (.*)")]>]
let ``then first account balance`` (expected: float) =
    Assert.Equal(expected, context.FirstAccount.Balance, 2)

[<Then("my second account balance should be (.*)")]>]
let ``then second account balance`` (expected: float) =
    match context.SecondAccount with
    | Some second -> Assert.Equal(expected, second.Balance, 2)
    | None -> Assert.Fail("Second account does not exist")

[<Then("I should get an error \"(.*)\"")]>]
let ``then error message`` (message: string) =
    match context.LastError with
    | Some error -> Assert.Equal(message, error)
    | None -> Assert.Fail("Expected an error but none occurred")
```

---

## 5. Shopping Cart Steps

```fsharp
// Steps/ShoppingSteps.fs
module BDD.Tests.ShoppingSteps

open TickSpec
open Xunit

// Domain types
type CartItem = { Name: string; Price: float; Quantity: int }
type Cart = { Items: CartItem list }

// Context
type ShoppingContext() =
    member val Cart: Cart = { Items = [] } with get, set
    member val Discount: float = 0.0 with get, set

let context = ShoppingContext()

// Helper
let cartTotal (cart: Cart) =
    cart.Items |> List.sumBy (fun i -> i.Price * float i.Quantity)

// =====================
// BACKGROUND step
// =====================

[<Given("an empty shopping cart")>]
let ``given empty cart`` () =
    context.Cart <- { Items = [] }
    context.Discount <- 0.0

// =====================
// GIVEN steps
// =====================

[<Given("I have added (.*) \"(.*)\" at price (.*)")]>]
let ``given item added`` (qty: int) (name: string) (price: float) =
    let item = { Name = name; Price = price; Quantity = qty }
    context.Cart <- { Items = item :: context.Cart.Items }

// =====================
// WHEN steps
// =====================

[<When("I add (.*) \"(.*)\" at price (.*)")]>]
let ``when add item`` (qty: int) (name: string) (price: float) =
    let item = { Name = name; Price = price; Quantity = qty }
    context.Cart <- { Items = item :: context.Cart.Items }

[<When("I remove \"(.*)\" from the cart")>]
let ``when remove item`` (name: string) =
    context.Cart <- { Items = context.Cart.Items |> List.filter (fun i -> i.Name <> name) }

[<When("I apply a (.*)% discount")>]
let ``when apply discount`` (discountPercent: float) =
    context.Discount <- discountPercent / 100.0

[<When("I clear the cart")>]
let ``when clear cart`` () =
    context.Cart <- { Items = [] }
    context.Discount <- 0.0

// =====================
// THEN steps
// =====================

[<Then("the cart should contain (.*) item(s?)")>]
let ``then cart item count`` (count: int) _ =
    Assert.Equal(count, context.Cart.Items.Length)

[<Then("the cart total should be (.*)")]>]
let ``then cart total`` (expected: float) =
    let total = cartTotal context.Cart
    let discountedTotal = total * (1.0 - context.Discount)
    Assert.Equal(expected, discountedTotal, 2)

[<Then("the cart should be empty")>]
let ``then cart is empty`` () =
    Assert.Empty(context.Cart.Items)
```

---

## 6. Authentication Steps

```fsharp
// Steps/AuthSteps.fs
module BDD.Tests.AuthSteps

open TickSpec
open Xunit
open System.Collections.Generic

// Domain types
type User = { Email: string; PasswordHash: string }
type AuthResult = 
    | LoggedIn of token: string
    | Failed of reason: string

// Simple hash (not for production!)
let hashPassword (password: string) = 
    System.Convert.ToBase64String(
        System.Text.Encoding.UTF8.GetBytes(password)
    )

// Context
type AuthContext() =
    member val Users: Dictionary<string, User> = Dictionary<string, User>() with get
    member val LastResult: AuthResult option = None with get, set
    member val LastRegistrationError: string option = None with get, set

let context = AuthContext()

// Password validation
let validatePassword (password: string) =
    let hasMinLength = password.Length >= 8
    let hasUppercase = password |> Seq.exists System.Char.IsUpper
    let hasDigit = password |> Seq.exists System.Char.IsDigit
    hasMinLength && hasUppercase && hasDigit

// =====================
// GIVEN steps
// =====================

[<Given("a user \"(.*)\" with password \"(.*)\"")>]
let ``given user with password`` (email: string) (password: string) =
    let user = { Email = email; PasswordHash = hashPassword password }
    context.Users.[email] <- user

// =====================
// WHEN steps
// =====================

[<When("I login with \"(.*)\" and \"(.*)\"")>]
let ``when login`` (email: string) (password: string) =
    match context.Users.TryGetValue(email) with
    | true, user ->
        if user.PasswordHash = hashPassword password then
            context.LastResult <- Some (LoggedIn (sprintf "token-%s" (System.Guid.NewGuid().ToString("N").[..7])))
        else
            context.LastResult <- Some (Failed "Invalid credentials")
    | false, _ ->
        context.LastResult <- Some (Failed "User not found")

[<When("I try to register with password \"(.*)\"")>]
let ``when try register with password`` (password: string) =
    if validatePassword password then
        context.LastRegistrationError <- None
    else
        context.LastRegistrationError <- Some "Password does not meet requirements"

// =====================
// THEN steps
// =====================

[<Then("I should be logged in")>]
let ``then logged in`` () =
    match context.LastResult with
    | Some (LoggedIn _) -> ()
    | Some (Failed reason) -> Assert.Fail(sprintf "Expected logged in but got: %s" reason)
    | None -> Assert.Fail("No auth result")

[<Then("I should not be logged in")>]
let ``then not logged in`` () =
    match context.LastResult with
    | Some (Failed _) -> ()
    | Some (LoggedIn _) -> Assert.Fail("Expected failed but was logged in")
    | None -> Assert.Fail("No auth result")

[<Then("I should receive an access token")>]
let ``then has access token`` () =
    match context.LastResult with
    | Some (LoggedIn token) -> Assert.NotEmpty(token)
    | _ -> Assert.Fail("Expected access token")

[<Then("I should see error \"(.*)\"")>]
let ``then error message`` (expectedError: string) =
    match context.LastResult with
    | Some (Failed reason) -> Assert.Equal(expectedError, reason)
    | _ -> Assert.Fail("Expected error message")

[<Then("registration should fail")>]
let ``then registration failed`` () =
    Assert.True(context.LastRegistrationError.IsSome)

[<Then("registration result should be \"(.*)\"")>]
let ``then registration result`` (expected: string) =
    match expected with
    | "success" -> Assert.True(context.LastRegistrationError.IsNone)
    | "failed" -> Assert.True(context.LastRegistrationError.IsSome)
    | _ -> Assert.Fail(sprintf "Unknown expected result: %s" expected)
```

---

## 7. BDD Runner

```fsharp
// BDDRunner.fs
module BDD.Tests.BDDRunner

open System
open System.IO
open System.Reflection
open Xunit
open TickSpec

// TickSpec runner สำหรับ xUnit
type BDDTests() =
    
    // รัน feature files ทั้งหมดที่ embedded
    static member private GetFeatures() =
        let assembly = Assembly.GetExecutingAssembly()
        
        assembly.GetManifestResourceNames()
        |> Array.filter (fun name -> name.EndsWith(".feature"))
        |> Array.map (fun name ->
            let stream = assembly.GetManifestResourceStream(name)
            use reader = new StreamReader(stream)
            let content = reader.ReadToEnd()
            (name, content)
        )
    
    // สร้าง test data จาก features
    static member FeatureData() : obj[][] =
        BDDTests.GetFeatures()
        |> Array.map (fun (name, content) -> [| name :> obj; content :> obj |])
    
    [<Theory>]
    [<MemberData("FeatureData")>]
    member _.``BDD Feature Tests`` (featureName: string) (featureContent: string) =
        let assembly = Assembly.GetExecutingAssembly()
        
        let feature = TickSpec.FeatureParser.parseFeature featureName (featureContent.Split('\n'))
        
        // Run scenarios
        let stepDefinitions = 
            StepDefinitions(
                [| assembly |],
                (fun (scenarioName, tags) -> 
                    printfn "Running: %s" scenarioName
                )
            )
        
        // Each scenario
        for scenario in feature.Scenarios do
            try
                stepDefinitions.Execute(scenario)
                printfn "PASSED: %s" scenario.Name
            with ex ->
                printfn "FAILED: %s - %s" scenario.Name ex.Message
                raise ex
```

---

## 8. Tables ใน Gherkin

```gherkin
# Features/TableTests.feature
Feature: Order Processing with Tables
  As a store manager
  I want to process orders
  So that customers receive their products

  Scenario: Process order with multiple items
    Given the following products are available:
      | Name    | Price | Stock |
      | Widget  | 9.99  | 100   |
      | Gadget  | 24.99 | 50    |
      | Gizmo   | 4.99  | 200   |
    When a customer orders:
      | Product | Quantity |
      | Widget  | 2        |
      | Gadget  | 1        |
    Then the order total should be 44.97
    And the stock should be updated:
      | Name    | Remaining Stock |
      | Widget  | 98              |
      | Gadget  | 49              |
      | Gizmo   | 200             |
```

```fsharp
// Steps/TableSteps.fs
module BDD.Tests.TableSteps

open TickSpec
open Xunit

type Product = { Name: string; Price: float; Stock: int }

type TableContext() =
    member val Products: Map<string, Product> = Map.empty with get, set
    member val OrderTotal: float = 0.0 with get, set

let context = TableContext()

// Table step
[<Given("the following products are available:")>]
let ``given products available`` (table: Table) =
    let products = 
        table.Rows
        |> Array.map (fun row ->
            let name = row.["Name"]
            let price = float row.["Price"]
            let stock = int row.["Stock"]
            name, { Name = name; Price = price; Stock = stock }
        )
        |> Map.ofArray
    
    context.Products <- products

[<When("a customer orders:")>]
let ``when customer orders`` (table: Table) =
    let mutable total = 0.0
    let mutable updatedProducts = context.Products
    
    for row in table.Rows do
        let productName = row.["Product"]
        let quantity = int row.["Quantity"]
        
        match updatedProducts |> Map.tryFind productName with
        | Some product ->
            total <- total + product.Price * float quantity
            updatedProducts <- updatedProducts |> Map.add productName 
                { product with Stock = product.Stock - quantity }
        | None ->
            failwithf "Product '%s' not found" productName
    
    context.OrderTotal <- total
    context.Products <- updatedProducts

[<Then("the order total should be (.*)")>]
let ``then order total`` (expected: float) =
    Assert.Equal(expected, context.OrderTotal, 2)

[<Then("the stock should be updated:")>]
let ``then stock updated`` (table: Table) =
    for row in table.Rows do
        let name = row.["Name"]
        let expectedStock = int row.["Remaining Stock"]
        
        match context.Products |> Map.tryFind name with
        | Some product -> Assert.Equal(expectedStock, product.Stock)
        | None -> Assert.Fail(sprintf "Product '%s' not found" name)
```

---

## 9. Background Steps

```gherkin
# Features/Background.feature
Feature: Database operations with shared setup
  
  Background:
    Given the database is clean
    And the following users exist:
      | Name    | Email               | Role  |
      | Alice   | alice@example.com   | admin |
      | Bob     | bob@example.com     | user  |
      | Charlie | charlie@example.com | user  |

  Scenario: Admin can list all users
    Given I am logged in as "alice@example.com"
    When I request the user list
    Then I should see 3 users

  Scenario: Regular user cannot list all users
    Given I am logged in as "bob@example.com"
    When I request the user list
    Then I should be denied access

  Scenario: User can update their own profile
    Given I am logged in as "bob@example.com"
    When I update my name to "Robert"
    Then my profile should show "Robert"
```

```fsharp
// Steps/BackgroundSteps.fs
module BDD.Tests.BackgroundSteps

open TickSpec
open Xunit

type UserEntry = { Name: string; Email: string; Role: string }

type BackgroundContext() =
    member val Users: UserEntry list = [] with get, set
    member val CurrentUser: UserEntry option = None with get, set
    member val LastResponse: Result<obj, string> = Error "No response" with get, set

let context = BackgroundContext()

// Background steps
[<Given("the database is clean")>]
let ``given database clean`` () =
    context.Users <- []
    context.CurrentUser <- None
    context.LastResponse <- Error "No response"

[<Given("the following users exist:")>]
let ``given users exist`` (table: Table) =
    let users = 
        table.Rows
        |> Array.map (fun row ->
            { 
                Name = row.["Name"]
                Email = row.["Email"]
                Role = row.["Role"]
            }
        )
        |> Array.toList
    
    context.Users <- users

// Scenario steps
[<Given("I am logged in as \"(.*)\"")>]
let ``given logged in as`` (email: string) =
    match context.Users |> List.tryFind (fun u -> u.Email = email) with
    | Some user -> context.CurrentUser <- Some user
    | None -> failwithf "User '%s' not found" email

[<When("I request the user list")>]
let ``when request user list`` () =
    match context.CurrentUser with
    | Some user when user.Role = "admin" ->
        context.LastResponse <- Ok (box context.Users)
    | Some _ ->
        context.LastResponse <- Error "Access denied"
    | None ->
        context.LastResponse <- Error "Not logged in"

[<Then("I should see (.*) users")>]
let ``then see users count`` (count: int) =
    match context.LastResponse with
    | Ok users ->
        let userList = users :?> UserEntry list
        Assert.Equal(count, userList.Length)
    | Error msg ->
        Assert.Fail(sprintf "Expected user list but got: %s" msg)

[<Then("I should be denied access")>]
let ``then denied access`` () =
    match context.LastResponse with
    | Error "Access denied" -> ()
    | Error msg -> Assert.Fail(sprintf "Expected 'Access denied' but got: %s" msg)
    | Ok _ -> Assert.Fail("Expected access denial but got success")
```

---

## 10. Scenario Outlines

```gherkin
# Features/ScenarioOutline.feature
Feature: Tax calculation
  As an accountant
  I want to calculate taxes
  So that I can process invoices correctly

  Scenario Outline: Calculate VAT for different countries
    Given the product price is <price>
    And the country is "<country>"
    When I calculate the tax
    Then the tax amount should be <tax>
    And the total should be <total>

    Examples:
      | price  | country | tax   | total  |
      | 100.00 | UK      | 20.00 | 120.00 |
      | 100.00 | DE      | 19.00 | 119.00 |
      | 100.00 | US      | 10.00 | 110.00 |
      | 100.00 | TH      | 7.00  | 107.00 |
      | 0.00   | UK      | 0.00  | 0.00   |
```

```fsharp
// Steps/TaxSteps.fs
module BDD.Tests.TaxSteps

open TickSpec
open Xunit

let vatRates = Map.ofList [
    "UK", 0.20
    "DE", 0.19
    "US", 0.10
    "TH", 0.07
    "FR", 0.20
    "JP", 0.10
]

type TaxContext() =
    member val Price: float = 0.0 with get, set
    member val Country: string = "" with get, set
    member val CalculatedTax: float = 0.0 with get, set

let context = TaxContext()

[<Given("the product price is (.*)")]>]
let ``given product price`` (price: float) =
    context.Price <- price

[<Given("the country is \"(.*)\"")>]
let ``given country`` (country: string) =
    context.Country <- country

[<When("I calculate the tax")>]
let ``when calculate tax`` () =
    let rate = 
        vatRates 
        |> Map.tryFind context.Country 
        |> Option.defaultValue 0.0
    context.CalculatedTax <- context.Price * rate

[<Then("the tax amount should be (.*)")]>]
let ``then tax amount`` (expected: float) =
    Assert.Equal(expected, context.CalculatedTax, 2)

[<Then("the total should be (.*)")]>]
let ``then total`` (expected: float) =
    let total = context.Price + context.CalculatedTax
    Assert.Equal(expected, total, 2)
```

---

## 11. Living Documentation

```fsharp
// LivingDocumentation.fs
// BDD feature files เป็น living documentation
// เมื่อ code เปลี่ยน tests fail -> update features too

// F# domain model ที่ตรงกับ BDD scenarios
module Domain =
    type AccountBalance = AccountBalance of float
    type TransferAmount = TransferAmount of float
    
    type TransferError =
        | InsufficientFunds
        | AccountNotFound
        | InvalidAmount
    
    type Transfer = {
        From: string
        To: string
        Amount: TransferAmount
    }
    
    // Business rules (มาจาก BDD scenarios)
    let processTransfer 
        (getBalance: string -> AccountBalance option)
        (updateBalance: string -> AccountBalance -> unit)
        (transfer: Transfer) =
        
        let (TransferAmount amount) = transfer.Amount
        
        if amount <= 0.0 then
            Error InvalidAmount
        else
            match getBalance transfer.From, getBalance transfer.To with
            | None, _ -> Error AccountNotFound
            | _, None -> Error AccountNotFound
            | Some (AccountBalance fromBalance), Some (AccountBalance toBalance) ->
                if fromBalance < amount then
                    Error InsufficientFunds
                else
                    updateBalance transfer.From (AccountBalance (fromBalance - amount))
                    updateBalance transfer.To (AccountBalance (toBalance + amount))
                    Ok ()

// Feature file สะท้อน domain rules:
// Feature: Money Transfer
//   Scenario: Successful transfer
//     Given account A has balance 1000
//     And account B has balance 500
//     When I transfer 200 from A to B
//     Then account A should have balance 800
//     And account B should have balance 700
//
//   Scenario: Transfer with insufficient funds
//     Given account A has balance 100
//     When I try to transfer 200 from A to B
//     Then I should get "Insufficient funds" error
//     And no balances should change
```

---

## สรุป (Summary)

BDD ใน F# กับ TickSpec:

1. **Feature files** เขียนเป็น Gherkin syntax
2. **Step definitions** เชื่อม Gherkin กับ F# code
3. **Background** setup สำหรับหลาย scenarios
4. **Scenario Outline** ทดสอบหลาย input sets
5. **Tables** สำหรับ structured data

**ประโยชน์ของ BDD:**
- สร้าง shared understanding ระหว่าง dev, QA, business
- Feature files เป็น living documentation
- Tests อ่านง่ายสำหรับ non-technical stakeholders
- ค้นหา gaps ใน requirements ได้ก่อน implement

```bash
# รัน BDD tests
dotnet test

# รัน specific feature
dotnet test --filter "FullyQualifiedName~BankingTests"
```
