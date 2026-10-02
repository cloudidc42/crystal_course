# Part 72 - FsUnit - F# Fluent Testing

## บทนำ (Introduction)

FsUnit เป็น library ที่ช่วยให้การเขียน test ใน F# อ่านง่ายขึ้นด้วย fluent syntax แบบ natural language โดยใช้งานร่วมกับ xUnit, NUnit, หรือ MSTest

FsUnit ช่วยแปลง assertions จาก:
```fsharp
Assert.Equal(expected, actual)
```
ให้เป็น:
```fsharp
actual |> should equal expected
```

ซึ่งอ่านได้เป็นธรรมชาติมากกว่า

---

## 1. การติดตั้ง FsUnit (Installation)

### กับ xUnit

```xml
<!-- TestProject.fsproj -->
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
    <!-- FsUnit with xUnit -->
    <PackageReference Include="FsUnit.xUnit" Version="6.0.1" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="BasicFsUnitTests.fs" />
  </ItemGroup>
</Project>
```

### กับ NUnit

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
  <PackageReference Include="NUnit" Version="3.14.0" />
  <PackageReference Include="NUnit3TestAdapter" Version="4.5.0">
    <PrivateAssets>all</PrivateAssets>
  </PackageReference>
  <!-- FsUnit with NUnit -->
  <PackageReference Include="FsUnit" Version="6.0.1" />
</ItemGroup>
```

---

## 2. should equal และ should not equal

```fsharp
// EqualityTests.fs
module MyProject.Tests.EqualityTests

open Xunit
open FsUnit.Xunit

// Basic equality
[<Fact>]
let ``number equality`` () =
    42 |> should equal 42
    10 |> should not' (equal 20)

[<Fact>]
let ``string equality`` () =
    "hello" |> should equal "hello"
    "hello" |> should not' (equal "world")

[<Fact>]
let ``float equality`` () =
    3.14 |> should (equalWithin 0.01) 3.14159

[<Fact>]
let ``list equality`` () =
    [1; 2; 3] |> should equal [1; 2; 3]
    [1; 2; 3] |> should not' (equal [1; 2; 4])

// Testing domain objects
type Product = { Name: string; Price: float; InStock: bool }

let createProduct name price inStock =
    { Name = name; Price = price; InStock = inStock }

[<Fact>]
let ``products are equal when all fields match`` () =
    let p1 = createProduct "Widget" 9.99 true
    let p2 = createProduct "Widget" 9.99 true
    
    p1 |> should equal p2

[<Fact>]
let ``products differ when price changes`` () =
    let p1 = createProduct "Widget" 9.99 true
    let p2 = createProduct "Widget" 19.99 true
    
    p1 |> should not' (equal p2)

// Computed values
let add x y = x + y
let multiply x y = x * y

[<Fact>]
let ``add produces correct result`` () =
    add 3 4 |> should equal 7
    add 0 0 |> should equal 0
    add (-5) 5 |> should equal 0

[<Fact>]
let ``multiply produces correct result`` () =
    multiply 3 4 |> should equal 12
    multiply 0 100 |> should equal 0
    multiply (-2) 3 |> should equal -6
```

---

## 3. should be greaterThan, lessThan

```fsharp
// ComparisonTests.fs
module MyProject.Tests.ComparisonTests

open Xunit
open FsUnit.Xunit

// Comparison assertions
[<Fact>]
let ``number comparisons`` () =
    10 |> should be (greaterThan 5)
    3 |> should be (lessThan 10)
    5 |> should be (greaterThanOrEqualTo 5)
    5 |> should be (lessThanOrEqualTo 5)
    10 |> should be (greaterThanOrEqualTo 5)
    3 |> should be (lessThanOrEqualTo 10)

[<Fact>]
let ``float comparisons`` () =
    3.14 |> should be (greaterThan 3.0)
    2.71 |> should be (lessThan 3.0)
    1.0 |> should be (greaterThanOrEqualTo 1.0)

// Testing function outputs with comparisons
let computeDiscount price quantity =
    if quantity >= 10 then price * 0.8
    elif quantity >= 5 then price * 0.9
    else price

[<Fact>]
let ``bulk discount is less than regular price`` () =
    let regularPrice = computeDiscount 100.0 1
    let bulkPrice = computeDiscount 100.0 10
    
    bulkPrice |> should be (lessThan regularPrice)

[<Fact>]
let ``medium discount is between bulk and regular`` () =
    let regularPrice = computeDiscount 100.0 1
    let mediumPrice = computeDiscount 100.0 5
    let bulkPrice = computeDiscount 100.0 10
    
    mediumPrice |> should be (greaterThan bulkPrice)
    mediumPrice |> should be (lessThan regularPrice)

// String length comparisons
[<Fact>]
let ``long name has more characters`` () =
    let shortName = "Al"
    let longName = "Alexander"
    
    longName.Length |> should be (greaterThan shortName.Length)

// Date comparisons
[<Fact>]
let ``future date is greater than past date`` () =
    let past = System.DateTime(2020, 1, 1)
    let future = System.DateTime(2030, 1, 1)
    
    future |> should be (greaterThan past)
    past |> should be (lessThan future)
```

---

## 4. should contain

```fsharp
// ContainTests.fs
module MyProject.Tests.ContainTests

open Xunit
open FsUnit.Xunit

// Collection contains
[<Fact>]
let ``list contains element`` () =
    let numbers = [1; 2; 3; 4; 5]
    
    numbers |> should contain 3
    numbers |> should not' (contain 6)

[<Fact>]
let ``array contains element`` () =
    let fruits = [| "apple"; "banana"; "cherry" |]
    
    fruits |> should contain "banana"
    fruits |> should not' (contain "grape")

[<Fact>]
let ``string contains substring`` () =
    let message = "Hello, World!"
    
    message |> should haveSubstring "World"
    message |> should not' (haveSubstring "Python")

// Contains with complex types
type Order = { Id: int; Product: string; Quantity: int }

let createOrder id product qty = 
    { Id = id; Product = product; Quantity = qty }

[<Fact>]
let ``order list contains specific order`` () =
    let orders = [
        createOrder 1 "Widget" 5
        createOrder 2 "Gadget" 3
        createOrder 3 "Gizmo" 10
    ]
    
    orders |> should contain (createOrder 2 "Gadget" 3)

// Sequence contains
[<Fact>]
let ``sequence contains elements`` () =
    let seq = seq { for i in 1..10 -> i }
    
    seq |> should contain 5
    seq |> should contain 10

// Map contains key
[<Fact>]
let ``map contains key`` () =
    let config = Map.ofList [
        "host", "localhost"
        "port", "5432"
        "database", "mydb"
    ]
    
    config |> Map.containsKey "host" |> should equal true
    config |> Map.containsKey "password" |> should equal false

// Testing Set
[<Fact>]
let ``set contains element`` () =
    let numbers = Set.ofList [1; 2; 3; 4; 5]
    
    numbers |> Set.contains 3 |> should equal true
    numbers |> Set.contains 6 |> should equal false
```

---

## 5. should be null, not null

```fsharp
// NullTests.fs
module MyProject.Tests.NullTests

open Xunit
open FsUnit.Xunit

// Null reference checks
[<Fact>]
let ``null reference assertions`` () =
    let nullString: string = null
    let nonNullString = "hello"
    
    nullString |> should be Null
    nonNullString |> should not' (be Null)

[<Fact>]
let ``null object assertions`` () =
    let nullObj: obj = null
    let nonNullObj: obj = box 42
    
    nullObj |> should be Null
    nonNullObj |> should not' (be Null)

// Testing functions that may return null
let findUserById (id: int) (users: Map<int, string>) : string =
    match users |> Map.tryFind id with
    | Some name -> name
    | None -> null

[<Fact>]
let ``found user is not null`` () =
    let users = Map.ofList [(1, "Alice"); (2, "Bob")]
    let user = findUserById 1 users
    
    user |> should not' (be Null)
    user |> should equal "Alice"

[<Fact>]
let ``not found user is null`` () =
    let users = Map.ofList [(1, "Alice")]
    let user = findUserById 999 users
    
    user |> should be Null

// Testing Option.toObj
[<Fact>]
let ``option none converts to null`` () =
    let opt: string option = None
    let result = opt |> Option.toObj
    
    result |> should be Null

[<Fact>]
let ``option some converts to non-null`` () =
    let opt = Some "value"
    let result = opt |> Option.toObj
    
    result |> should not' (be Null)
    result |> should equal "value"
```

---

## 6. shouldFail

```fsharp
// ShouldFailTests.fs
module MyProject.Tests.ShouldFailTests

open Xunit
open FsUnit.Xunit

// Functions that should throw
let divideUnsafe x y =
    if y = 0 then failwith "Division by zero"
    x / y

let parseAge (s: string) =
    let age = int s
    if age < 0 || age > 150 then
        failwithf "Invalid age: %d" age
    age

// shouldFail tests
[<Fact>]
let ``division by zero should fail`` () =
    (fun () -> divideUnsafe 10 0 |> ignore) |> should throw typeof<System.Exception>

[<Fact>]
let ``invalid age should fail`` () =
    (fun () -> parseAge "-1" |> ignore) |> should throw typeof<System.Exception>

[<Fact>]
let ``valid division should not fail`` () =
    let result = divideUnsafe 10 2
    result |> should equal 5

// Testing with specific exception types
let connectToDatabase (connectionString: string) =
    if System.String.IsNullOrEmpty(connectionString) then
        raise (System.ArgumentException("Connection string cannot be empty"))
    "Connected"

[<Fact>]
let ``empty connection string throws ArgumentException`` () =
    (fun () -> connectToDatabase "" |> ignore) 
    |> should throw typeof<System.ArgumentException>

[<Fact>]
let ``valid connection string does not throw`` () =
    let result = connectToDatabase "Server=localhost"
    result |> should equal "Connected"
```

---

## 7. Custom Matchers

```fsharp
// CustomMatcherTests.fs
module MyProject.Tests.CustomMatcherTests

open Xunit
open FsUnit.Xunit
open NHamcrest

// สร้าง custom matcher
let bePositive = 
    CustomMatcher<int>("positive", fun x -> x > 0)

let beEven = 
    CustomMatcher<int>("even", fun x -> x % 2 = 0)

let startWithCapital = 
    CustomMatcher<string>("start with capital", fun s ->
        s.Length > 0 && System.Char.IsUpper(s.[0])
    )

let beValidEmail =
    CustomMatcher<string>("valid email", fun s ->
        s.Contains("@") && s.Contains(".") && s.Length > 5
    )

// Tests กับ custom matchers
[<Fact>]
let ``number is positive`` () =
    42 |> should be bePositive
    -5 |> should not' (be bePositive)
    0 |> should not' (be bePositive)

[<Fact>]
let ``number is even`` () =
    4 |> should be beEven
    3 |> should not' (be beEven)
    0 |> should be beEven

[<Fact>]
let ``string starts with capital`` () =
    "Hello" |> should be startWithCapital
    "hello" |> should not' (be startWithCapital)
    "" |> should not' (be startWithCapital)

[<Fact>]
let ``email address is valid`` () =
    "test@example.com" |> should be beValidEmail
    "invalid" |> should not' (be beValidEmail)
    "no-at-sign.com" |> should not' (be beValidEmail)

// Complex custom matcher สำหรับ domain objects
type BankAccount = {
    AccountNumber: string
    Balance: float
    IsActive: bool
    Transactions: float list
}

let beHealthyAccount =
    CustomMatcher<BankAccount>("healthy bank account", fun acc ->
        acc.IsActive && acc.Balance >= 0.0 && acc.AccountNumber.Length > 0
    )

let havePositiveBalance =
    CustomMatcher<BankAccount>("positive balance", fun acc ->
        acc.Balance > 0.0
    )

[<Fact>]
let ``active account with positive balance is healthy`` () =
    let account = {
        AccountNumber = "ACC-001"
        Balance = 1000.0
        IsActive = true
        Transactions = [100.0; 200.0; -50.0]
    }
    
    account |> should be beHealthyAccount
    account |> should be havePositiveBalance

[<Fact>]
let ``inactive account is not healthy`` () =
    let account = {
        AccountNumber = "ACC-002"
        Balance = 500.0
        IsActive = false
        Transactions = []
    }
    
    account |> should not' (be beHealthyAccount)
```

---

## 8. FsUnit กับ xUnit - การใช้งานครบถ้วน

```fsharp
// XUnitFsUnitTests.fs
module MyProject.Tests.XUnitFsUnitTests

open Xunit
open FsUnit.Xunit

// Domain model
type CartItem = { ProductId: int; Name: string; Price: float; Quantity: int }
type Cart = { Items: CartItem list; Discount: float }

// Shopping cart operations
let addItem (item: CartItem) (cart: Cart) =
    { cart with Items = item :: cart.Items }

let removeItem productId (cart: Cart) =
    { cart with Items = cart.Items |> List.filter (fun i -> i.ProductId <> productId) }

let calculateSubtotal (cart: Cart) =
    cart.Items |> List.sumBy (fun i -> i.Price * float i.Quantity)

let calculateTotal (cart: Cart) =
    let subtotal = calculateSubtotal cart
    subtotal * (1.0 - cart.Discount)

let emptyCart = { Items = []; Discount = 0.0 }

// Tests
[<Fact>]
let ``empty cart has no items`` () =
    emptyCart.Items |> should be Empty

[<Fact>]
let ``empty cart total is zero`` () =
    calculateTotal emptyCart |> should equal 0.0

[<Fact>]
let ``adding item increases cart size`` () =
    let item = { ProductId = 1; Name = "Widget"; Price = 9.99; Quantity = 1 }
    let cart = addItem item emptyCart
    
    cart.Items |> should haveLength 1
    cart.Items |> should contain item

[<Fact>]
let ``removing item decreases cart size`` () =
    let item1 = { ProductId = 1; Name = "Widget"; Price = 9.99; Quantity = 1 }
    let item2 = { ProductId = 2; Name = "Gadget"; Price = 19.99; Quantity = 2 }
    
    let cart = 
        emptyCart 
        |> addItem item1 
        |> addItem item2
        |> removeItem 1
    
    cart.Items |> should haveLength 1
    cart.Items |> should not' (contain item1)
    cart.Items |> should contain item2

[<Fact>]
let ``subtotal sums all items`` () =
    let items = [
        { ProductId = 1; Name = "Widget"; Price = 10.0; Quantity = 2 }
        { ProductId = 2; Name = "Gadget"; Price = 15.0; Quantity = 1 }
    ]
    
    let cart = { Items = items; Discount = 0.0 }
    calculateSubtotal cart |> should equal 35.0

[<Fact>]
let ``discount reduces total`` () =
    let items = [{ ProductId = 1; Name = "Widget"; Price = 100.0; Quantity = 1 }]
    let cart = { Items = items; Discount = 0.1 }  // 10% discount
    
    let total = calculateTotal cart
    total |> should equal 90.0
    total |> should be (lessThan 100.0)

// Theory กับ FsUnit
[<Theory>]
[<InlineData(0.0, 100.0, 100.0)>]
[<InlineData(0.1, 100.0, 90.0)>]
[<InlineData(0.25, 100.0, 75.0)>]
[<InlineData(0.5, 100.0, 50.0)>]
let ``discount calculation is correct`` (discount: float) (price: float) (expected: float) =
    let items = [{ ProductId = 1; Name = "Item"; Price = price; Quantity = 1 }]
    let cart = { Items = items; Discount = discount }
    
    calculateTotal cart |> should (equalWithin 0.001) expected
```

---

## 9. FsUnit กับ NUnit

```fsharp
// NUnitFsUnitTests.fs
// ใช้กับ NUnit แทน xUnit
module MyProject.Tests.NUnitFsUnitTests

open NUnit.Framework
open FsUnit

// Domain types
type ValidationResult =
    | Valid
    | Invalid of string list

let validatePassword (password: string) =
    let errors = [
        if password.Length < 8 then yield "Password must be at least 8 characters"
        if not (password |> Seq.exists System.Char.IsUpper) then yield "Password must contain uppercase"
        if not (password |> Seq.exists System.Char.IsDigit) then yield "Password must contain a digit"
    ]
    if errors.IsEmpty then Valid
    else Invalid errors

// NUnit tests กับ FsUnit
[<Test>]
let ``valid password passes all checks`` () =
    let result = validatePassword "SecureP4ss"
    result |> should equal Valid

[<Test>]
let ``short password fails validation`` () =
    let result = validatePassword "Short1"
    match result with
    | Invalid errors ->
        errors |> should not' (be Empty)
        errors |> should contain "Password must be at least 8 characters"
    | Valid ->
        Assert.Fail("Expected Invalid result")

[<Test>]
let ``password without uppercase fails`` () =
    let result = validatePassword "lowercase123"
    match result with
    | Invalid errors ->
        errors |> should contain "Password must contain uppercase"
    | Valid ->
        Assert.Fail("Expected Invalid result")

[<Test>]
let ``weak password has multiple errors`` () =
    let result = validatePassword "weak"
    match result with
    | Invalid errors ->
        errors.Length |> should be (greaterThan 1)
    | Valid ->
        Assert.Fail("Expected Invalid result")

// TestCase attribute ใน NUnit (เทียบเท่า Theory ใน xUnit)
[<TestCase("Password123", true)>]
[<TestCase("short1", false)>]
[<TestCase("nouppercase123", false)>]
[<TestCase("NoDigits!!", false)>]
let ``password validation results`` (password: string) (expectedValid: bool) =
    let result = validatePassword password
    let isValid = result = Valid
    isValid |> should equal expectedValid
```

---

## 10. เปรียบเทียบ FsUnit กับ xUnit Assertions

```fsharp
// ComparisonTests.fs
module MyProject.Tests.ComparisonTests

open Xunit
open FsUnit.Xunit

// ฟังก์ชันที่ทดสอบ
let divide x y =
    if y = 0.0 then None
    else Some (x / y)

// === xUnit style ===
module XUnitStyle =
    [<Fact>]
    let ``xUnit style assertions`` () =
        // Basic equality
        Assert.Equal(42, 42)
        
        // Negation
        Assert.NotEqual(1, 2)
        
        // Boolean
        Assert.True(1 = 1)
        Assert.False(1 = 2)
        
        // Collection
        Assert.Contains(3, [1; 2; 3])
        Assert.Empty([])
        
        // Null
        Assert.Null(null: string)
        Assert.NotNull("hello")

// === FsUnit style ===
module FsUnitStyle =
    [<Fact>]
    let ``FsUnit style assertions`` () =
        // Basic equality
        42 |> should equal 42
        
        // Negation
        1 |> should not' (equal 2)
        
        // Boolean
        (1 = 1) |> should be True
        (1 = 2) |> should be False
        
        // Collection
        [1; 2; 3] |> should contain 3
        [] |> should be Empty
        
        // Null
        (null: string) |> should be Null
        "hello" |> should not' (be Null)

// === เปรียบเทียบ readability ===
module ReadabilityComparison =
    type User = { Name: string; Age: int; IsActive: bool }
    
    let users = [
        { Name = "Alice"; Age = 30; IsActive = true }
        { Name = "Bob"; Age = 25; IsActive = false }
        { Name = "Charlie"; Age = 35; IsActive = true }
    ]
    
    // xUnit style - อ่านยากกว่าเล็กน้อย
    [<Fact>]
    let ``xUnit: active users exist`` () =
        let activeUsers = users |> List.filter (fun u -> u.IsActive)
        Assert.NotEmpty(activeUsers)
        Assert.Equal(2, activeUsers.Length)
    
    // FsUnit style - อ่านง่ายกว่า
    [<Fact>]
    let ``FsUnit: active users exist`` () =
        let activeUsers = users |> List.filter (fun u -> u.IsActive)
        
        activeUsers |> should not' (be Empty)
        activeUsers |> should haveLength 2
    
    // Complex assertion - FsUnit ชนะด้านความ readable
    [<Fact>]
    let ``average age is in valid range`` () =
        let avgAge = users |> List.averageBy (fun u -> float u.Age)
        
        // xUnit
        Assert.InRange(avgAge, 20.0, 50.0)
        
        // FsUnit  
        avgAge |> should be (greaterThan 20.0)
        avgAge |> should be (lessThan 50.0)
```

---

## 11. Building Readable Test Descriptions

```fsharp
// ReadableTests.fs
module MyProject.Tests.ReadableTests

open Xunit
open FsUnit.Xunit

// Domain: Banking System
type TransactionType = Deposit | Withdrawal | Transfer

type Transaction = {
    Id: System.Guid
    Type: TransactionType
    Amount: float
    Description: string
    Date: System.DateTime
}

type Account = {
    AccountNumber: string
    Owner: string
    Balance: float
    Transactions: Transaction list
}

// Operations
let deposit amount description (account: Account) =
    let tx = {
        Id = System.Guid.NewGuid()
        Type = Deposit
        Amount = amount
        Description = description
        Date = System.DateTime.Now
    }
    { account with
        Balance = account.Balance + amount
        Transactions = tx :: account.Transactions }

let withdraw amount description (account: Account) =
    if account.Balance < amount then
        Error (sprintf "Insufficient funds. Balance: %.2f, Requested: %.2f" account.Balance amount)
    else
        let tx = {
            Id = System.Guid.NewGuid()
            Type = Withdrawal
            Amount = amount
            Description = description
            Date = System.DateTime.Now
        }
        Ok { account with
                Balance = account.Balance - amount
                Transactions = tx :: account.Transactions }

let createAccount number owner initialBalance = {
    AccountNumber = number
    Owner = owner
    Balance = initialBalance
    Transactions = []
}

// Readable tests using FsUnit
[<Fact>]
let ``new account has correct initial balance`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    
    account.Balance |> should equal 1000.0
    account.Transactions |> should be Empty

[<Fact>]
let ``depositing increases balance by exact amount`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    let updated = deposit 500.0 "Salary" account
    
    updated.Balance |> should equal 1500.0
    updated.Transactions |> should haveLength 1

[<Fact>]
let ``transaction description matches deposit reason`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    let updated = deposit 500.0 "Christmas bonus" account
    
    let latestTx = updated.Transactions.Head
    latestTx.Description |> should equal "Christmas bonus"
    latestTx.Type |> should equal Deposit
    latestTx.Amount |> should equal 500.0

[<Fact>]
let ``withdrawing decreases balance by exact amount`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    
    match withdraw 300.0 "Rent" account with
    | Ok updated ->
        updated.Balance |> should equal 700.0
        updated.Transactions |> should haveLength 1
    | Error msg ->
        Assert.Fail(sprintf "Expected Ok but got Error: %s" msg)

[<Fact>]
let ``withdrawal fails when insufficient funds`` () =
    let account = createAccount "ACC-001" "Alice" 100.0
    
    let result = withdraw 500.0 "Large purchase" account
    
    result |> should not' (equal (Ok account))
    match result with
    | Error msg ->
        msg |> should haveSubstring "Insufficient funds"
    | Ok _ ->
        Assert.Fail("Expected Error but got Ok")

[<Fact>]
let ``multiple transactions are tracked correctly`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    
    let finalAccount =
        account
        |> deposit 500.0 "Salary"
        |> deposit 200.0 "Freelance"
        |> fun acc ->
            match withdraw 300.0 "Rent" acc with
            | Ok updated -> updated
            | Error _ -> acc
    
    finalAccount.Balance |> should equal 1400.0
    finalAccount.Transactions |> should haveLength 3

[<Fact>]
let ``account balance never goes negative on valid withdrawals`` () =
    let account = createAccount "ACC-001" "Alice" 1000.0
    
    let withdrawalResult = withdraw 1000.0 "Full withdrawal" account
    
    match withdrawalResult with
    | Ok updated ->
        updated.Balance |> should be (greaterThanOrEqualTo 0.0)
    | Error _ ->
        ()  // Error is also acceptable

// Grouping related tests
module DepositTests =
    [<Fact>]
    let ``deposit with zero amount does not change balance`` () =
        let account = createAccount "ACC-001" "Alice" 500.0
        let updated = deposit 0.0 "Empty deposit" account
        
        updated.Balance |> should equal 500.0

    [<Fact>]
    let ``deposit adds transaction to history`` () =
        let account = createAccount "ACC-001" "Alice" 500.0
        let updated = deposit 100.0 "Test deposit" account
        
        updated.Transactions |> should not' (be Empty)
        updated.Transactions.Head.Type |> should equal Deposit

module WithdrawalTests =
    [<Fact>]
    let ``withdrawal from empty account fails`` () =
        let account = createAccount "ACC-001" "Alice" 0.0
        let result = withdraw 1.0 "Test" account
        
        result |> should not' (equal (Ok account))

    [<Fact>]
    let ``exact balance withdrawal succeeds`` () =
        let account = createAccount "ACC-001" "Alice" 100.0
        let result = withdraw 100.0 "Full withdrawal" account
        
        match result with
        | Ok updated ->
            updated.Balance |> should equal 0.0
        | Error msg ->
            Assert.Fail(sprintf "Unexpected error: %s" msg)
```

---

## 12. สรุป (Summary)

FsUnit ทำให้การเขียน test ใน F# อ่านได้เป็นธรรมชาติมากขึ้น:

```fsharp
// สรุป syntax ที่ใช้บ่อย

// Equality
value |> should equal expected
value |> should not' (equal unexpected)

// Comparisons  
value |> should be (greaterThan minimum)
value |> should be (lessThan maximum)
value |> should be (greaterThanOrEqualTo minimum)
value |> should be (lessThanOrEqualTo maximum)

// Collections
list |> should contain element
list |> should not' (contain element)
list |> should be Empty
list |> should not' (be Empty)
list |> should haveLength n

// Strings
str |> should haveSubstring substring
str |> should startWith prefix
str |> should endWith suffix

// Null
obj |> should be Null
obj |> should not' (be Null)

// Boolean
condition |> should be True
condition |> should be False

// Approximate equality
float |> should (equalWithin delta) expected

// Exceptions
(fun () -> ...) |> should throw typeof<ExceptionType>
```

### เลือกใช้ FsUnit เมื่อไหร่

FsUnit เหมาะกับ:
- Tests ที่ต้องการ readable มากๆ
- Teams ที่ชอบ natural language style
- BDD-like testing
- Tests ที่มีคนอ่านมาก (documentation purposes)

xUnit โดยตรงเหมาะกับ:
- Tests ที่เรียบง่าย
- Performance sensitive contexts
- Teams ที่คุ้นกับ .NET testing
