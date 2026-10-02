# Part 108 - Testing Patterns ขั้นสูง

## บทนำ (Introduction)

Advanced testing patterns ช่วยให้ tests มีคุณภาพสูง maintainable และสามารถ verify behavior ของ system ได้ครอบคลุม

---

## 1. Test Data Builders

### Builder Pattern สำหรับ Test Data

```fsharp
// TestBuilders.fs

// ===== Domain Types =====

type Address = {
    Street: string
    City: string
    PostalCode: string
    Country: string
}

type Customer = {
    Id: System.Guid
    Name: string
    Email: string
    Age: int
    Address: Address
    IsActive: bool
    Tags: string list
    CreatedAt: System.DateTime
}

type OrderItem = {
    ProductId: int
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

type Order = {
    Id: System.Guid
    CustomerId: System.Guid
    Items: OrderItem list
    Status: OrderStatus
    CreatedAt: System.DateTime
    TotalAmount: decimal
}

and OrderStatus = 
    | Pending
    | Confirmed
    | Shipped
    | Delivered
    | Cancelled

// ===== Builder Types =====

type AddressBuilder = {
    Street: string
    City: string
    PostalCode: string
    Country: string
}

module AddressBuilder =
    let create () = {
        Street = "123 Main Street"
        City = "Bangkok"
        PostalCode = "10110"
        Country = "Thailand"
    }
    
    let withStreet street b = { b with Street = street }
    let withCity city b = { b with City = city }
    let withPostalCode code b = { b with PostalCode = code }
    let withCountry country b = { b with Country = country }
    
    let build (b: AddressBuilder) : Address = {
        Street = b.Street
        City = b.City
        PostalCode = b.PostalCode
        Country = b.Country
    }

type CustomerBuilder = {
    Id: System.Guid
    Name: string
    Email: string
    Age: int
    Address: AddressBuilder
    IsActive: bool
    Tags: string list
    CreatedAt: System.DateTime
}

module CustomerBuilder =
    let create () = {
        Id = System.Guid.NewGuid()
        Name = "Test Customer"
        Email = "test@example.com"
        Age = 30
        Address = AddressBuilder.create()
        IsActive = true
        Tags = []
        CreatedAt = System.DateTime(2024, 1, 1)
    }
    
    let withId id b = { b with Id = id }
    let withName name b = { b with Name = name }
    let withEmail email b = { b with Email = email }
    let withAge age b = { b with Age = age }
    let withAddress addressFn b = { b with Address = addressFn b.Address }
    let withTags tags b = { b with Tags = tags }
    let addTag tag b = { b with Tags = b.Tags @ [tag] }
    let inactive b = { b with IsActive = false }
    let createdAt dt b = { b with CreatedAt = dt }
    
    let build (b: CustomerBuilder) : Customer = {
        Id = b.Id
        Name = b.Name
        Email = b.Email
        Age = b.Age
        Address = AddressBuilder.build b.Address
        IsActive = b.IsActive
        Tags = b.Tags
        CreatedAt = b.CreatedAt
    }

// ===== Usage in Tests =====

open Xunit

[<Fact>]
let ``Customer builder creates valid customer`` () =
    let customer =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "สมชาย ใจดี"
        |> CustomerBuilder.withEmail "somchai@example.com"
        |> CustomerBuilder.withAge 25
        |> CustomerBuilder.addTag "vip"
        |> CustomerBuilder.addTag "loyal"
        |> CustomerBuilder.withAddress (
            AddressBuilder.withCity "เชียงใหม่"
            >> AddressBuilder.withCountry "Thailand")
        |> CustomerBuilder.build
    
    Assert.Equal("สมชาย ใจดี", customer.Name)
    Assert.Equal(25, customer.Age)
    Assert.Equal("เชียงใหม่", customer.Address.City)
    Assert.Contains("vip", customer.Tags)
    Assert.True(customer.IsActive)

[<Fact>]
let ``Inactive customer builder`` () =
    let customer =
        CustomerBuilder.create()
        |> CustomerBuilder.inactive
        |> CustomerBuilder.build
    
    Assert.False(customer.IsActive)

// ===== Order Builder =====

type OrderItemBuilder = {
    ProductId: int
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

module OrderItemBuilder =
    let create () = {
        ProductId = 1
        ProductName = "Test Product"
        Quantity = 1
        UnitPrice = 100.0m
    }
    
    let withProduct id name b = { b with ProductId = id; ProductName = name }
    let withQuantity qty b = { b with Quantity = qty }
    let withPrice price b = { b with UnitPrice = price }
    
    let build (b: OrderItemBuilder) : OrderItem = {
        ProductId = b.ProductId
        ProductName = b.ProductName
        Quantity = b.Quantity
        UnitPrice = b.UnitPrice
    }

type OrderBuilder = {
    CustomerId: System.Guid
    Items: OrderItemBuilder list
    Status: OrderStatus
}

module OrderBuilder =
    let create customerId = {
        CustomerId = customerId
        Items = [OrderItemBuilder.create()]
        Status = Pending
    }
    
    let withItem itemFn b = { b with Items = b.Items @ [itemFn (OrderItemBuilder.create())] }
    let clearItems b = { b with Items = [] }
    let addItem item b = { b with Items = b.Items @ [item] }
    let withStatus status b = { b with Status = status }
    
    let build (b: OrderBuilder) : Order =
        let items = b.Items |> List.map OrderItemBuilder.build
        let total = items |> List.sumBy (fun i -> i.UnitPrice * decimal i.Quantity)
        {
            Id = System.Guid.NewGuid()
            CustomerId = b.CustomerId
            Items = items
            Status = b.Status
            CreatedAt = System.DateTime.Now
            TotalAmount = total
        }
```

---

## 2. Object Mother Pattern

```fsharp
// ObjectMother.fs
// Object Mother สร้าง pre-configured test objects

module TestCustomers =
    
    let regularCustomer () =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "Regular Customer"
        |> CustomerBuilder.withEmail "regular@example.com"
        |> CustomerBuilder.withAge 30
        |> CustomerBuilder.build
    
    let vipCustomer () =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "VIP Customer"
        |> CustomerBuilder.withEmail "vip@example.com"
        |> CustomerBuilder.withAge 45
        |> CustomerBuilder.addTag "vip"
        |> CustomerBuilder.addTag "premium"
        |> CustomerBuilder.build
    
    let youngCustomer () =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "Young Customer"
        |> CustomerBuilder.withAge 18
        |> CustomerBuilder.build
    
    let minorCustomer () =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "Minor Customer"
        |> CustomerBuilder.withAge 15
        |> CustomerBuilder.build
    
    let inactiveCustomer () =
        CustomerBuilder.create()
        |> CustomerBuilder.withName "Inactive Customer"
        |> CustomerBuilder.inactive
        |> CustomerBuilder.build
    
    let customerWithManyOrders customerId count =
        let customer = 
            CustomerBuilder.create()
            |> CustomerBuilder.withId customerId
            |> CustomerBuilder.build
        
        let orders = 
            List.init count (fun i ->
                OrderBuilder.create customerId
                |> OrderBuilder.withStatus Delivered
                |> OrderBuilder.build)
        
        customer, orders

module TestOrders =
    
    let pendingOrder customerId =
        OrderBuilder.create customerId
        |> OrderBuilder.withStatus Pending
        |> OrderBuilder.build
    
    let confirmedOrder customerId =
        OrderBuilder.create customerId
        |> OrderBuilder.withStatus Confirmed
        |> OrderBuilder.build
    
    let largeOrder customerId =
        OrderBuilder.create customerId
        |> OrderBuilder.clearItems
        |> OrderBuilder.withItem (
            OrderItemBuilder.withProduct 1 "Laptop"
            >> OrderItemBuilder.withPrice 50000m
            >> OrderItemBuilder.withQuantity 2)
        |> OrderBuilder.withItem (
            OrderItemBuilder.withProduct 2 "Mouse"
            >> OrderItemBuilder.withPrice 500m
            >> OrderItemBuilder.withQuantity 3)
        |> OrderBuilder.build

// ===== Usage =====

[<Fact>]
let ``VIP customer gets discount`` () =
    let customer = TestCustomers.vipCustomer()
    let order = TestOrders.largeOrder customer.Id
    
    let discount = calculateDiscount customer order
    
    Assert.True(discount > 0.0m)

[<Fact>]
let ``Minor customer cannot place order`` () =
    let customer = TestCustomers.minorCustomer()
    let order = TestOrders.pendingOrder customer.Id
    
    let result = placeOrder customer order
    
    Assert.Equal(Error "ต้องมีอายุ 18 ปีขึ้นไป", result)
```

---

## 3. Test Fixtures

```fsharp
// TestFixtures.fs

// ===== Class-based Fixture =====

type DatabaseFixture() =
    let connectionString = "Host=localhost;Database=test_db;Username=test;Password=test"
    
    do
        // Setup: สร้าง test database
        use conn = new Npgsql.NpgsqlConnection(connectionString)
        conn.Open()
        conn.Execute("CREATE TABLE IF NOT EXISTS customers (...)") |> ignore
    
    member _.ConnectionString = connectionString
    
    interface System.IDisposable with
        member _.Dispose() =
            // Teardown: ลบ test data
            use conn = new Npgsql.NpgsqlConnection(connectionString)
            conn.Open()
            conn.Execute("TRUNCATE TABLE customers CASCADE") |> ignore

type CustomerTests(fixture: DatabaseFixture) =
    interface IClassFixture<DatabaseFixture>
    
    [<Fact>]
    member _.``Can create customer`` () =
        let repo = CustomerRepository(fixture.ConnectionString)
        let customer = TestCustomers.regularCustomer()
        
        let result = repo.Create(customer)
        
        Assert.Equal(Ok customer.Id, result)
    
    [<Fact>]
    member _.``Can find customer by email`` () =
        let repo = CustomerRepository(fixture.ConnectionString)
        let customer = TestCustomers.regularCustomer()
        repo.Create(customer) |> ignore
        
        let found = repo.FindByEmail(customer.Email)
        
        Assert.Equal(Some customer, found)

// ===== Collection Fixture =====

[<CollectionDefinition("Database collection")>]
type DatabaseCollection() =
    interface ICollectionFixture<DatabaseFixture>

[<Collection("Database collection")>]
type OrderTests(fixture: DatabaseFixture) =
    [<Fact>]
    member _.``Can create order`` () =
        // ...
        ()
```

---

## 4. Snapshot Testing

```fsharp
// SnapshotTests.fs

// Snapshot testing ตรวจสอบว่า output ตรงกับ "snapshot" ที่บันทึกไว้

module SnapshotTesting =
    open System.IO
    open System.Text.Json
    
    let snapshotDir = "./snapshots"
    
    let private getSnapshotPath testName =
        Path.Combine(snapshotDir, sprintf "%s.snap" testName)
    
    let save (testName: string) (value: obj) =
        Directory.CreateDirectory(snapshotDir) |> ignore
        let json = JsonSerializer.Serialize(value, JsonSerializerOptions(WriteIndented = true))
        File.WriteAllText(getSnapshotPath testName, json)
    
    let verify<'T> (testName: string) (actual: 'T) =
        let snapshotPath = getSnapshotPath testName
        
        if not (File.Exists(snapshotPath)) then
            // ครั้งแรก: บันทึก snapshot
            save testName actual
            printfn "Created snapshot: %s" snapshotPath
            true
        else
            let saved = File.ReadAllText(snapshotPath)
            let actualJson = JsonSerializer.Serialize(actual, JsonSerializerOptions(WriteIndented = true))
            
            if saved = actualJson then true
            else
                printfn "Snapshot mismatch for: %s" testName
                printfn "Expected:\n%s" saved
                printfn "Actual:\n%s" actualJson
                false
    
    let update (testName: string) (actual: obj) =
        save testName actual
        printfn "Updated snapshot: %s" testName

// ===== Usage =====

[<Fact>]
let ``Report format matches snapshot`` () =
    let orders = [
        TestOrders.largeOrder (System.Guid.NewGuid())
        TestOrders.confirmedOrder (System.Guid.NewGuid())
    ]
    
    let report = generateReport orders
    
    let matches = SnapshotTesting.verify "order-report" report
    Assert.True(matches, "Report format changed! Update snapshot if intentional.")

// ===== String Snapshot =====

[<Fact>]
let ``Email template matches snapshot`` () =
    let customer = TestCustomers.vipCustomer()
    let order = TestOrders.largeOrder customer.Id
    
    let emailBody = generateEmailBody customer order
    
    // Simple text comparison
    let snapshotPath = "./snapshots/vip-email.txt"
    
    if not (System.IO.File.Exists(snapshotPath)) then
        System.IO.Directory.CreateDirectory("./snapshots") |> ignore
        System.IO.File.WriteAllText(snapshotPath, emailBody)
        printfn "Created email snapshot"
    else
        let expected = System.IO.File.ReadAllText(snapshotPath)
        Assert.Equal(expected, emailBody)
```

---

## 5. Property-Based Testing

```fsharp
// PropertyTests.fs
#r "nuget: FsCheck, 2.16.6"
#r "nuget: FsCheck.Xunit, 2.16.6"

open FsCheck
open FsCheck.Xunit

// ===== Basic Property Tests =====

[<Property>]
let ``Sorting is idempotent`` (list: int list) =
    let sorted = List.sort list
    let sortedTwice = List.sort sorted
    sorted = sortedTwice

[<Property>]
let ``Reverse of reverse is identity`` (list: int list) =
    List.rev (List.rev list) = list

[<Property>]
let ``Concatenation is associative`` (a: string) (b: string) (c: string) =
    (a + b) + c = a + (b + c)

// ===== Custom Generators =====

// Generator สำหรับ valid email
let emailGen =
    gen {
        let! user = Gen.elements ["alice"; "bob"; "charlie"; "david"; "eve"]
        let! domain = Gen.elements ["example.com"; "test.org"; "sample.net"]
        return sprintf "%s@%s" user domain
    }

let validAgeGen = Gen.choose (18, 100)
let invalidAgeGen = Gen.choose (-100, 17)

// Generator สำหรับ Customer
let customerGen =
    gen {
        let! name = Gen.elements ["สมชาย"; "สมหญิง"; "สมศักดิ์"; "Alice"; "Bob"]
        let! email = emailGen
        let! age = validAgeGen
        return 
            CustomerBuilder.create()
            |> CustomerBuilder.withName name
            |> CustomerBuilder.withEmail email
            |> CustomerBuilder.withAge age
            |> CustomerBuilder.build
    }

type Generators =
    static member Customer() = Arb.fromGen customerGen

[<Property(Arbitrary = [| typeof<Generators> |])>]
let ``Valid customers can always place orders`` (customer: Customer) =
    let order = TestOrders.pendingOrder customer.Id
    let result = placeOrder customer order
    
    match result with
    | Ok _ -> true
    | Error _ -> false

// ===== Model-Based Testing =====

// State machine model
type CartState =
    | Empty
    | HasItems of items: (int * int) list  // (productId, quantity)

type CartAction =
    | AddItem of productId: int * quantity: int
    | RemoveItem of productId: int
    | Clear

let cartTransition state action =
    match state, action with
    | _, Clear -> Empty
    | Empty, AddItem (id, qty) when qty > 0 -> HasItems [(id, qty)]
    | HasItems items, AddItem (id, qty) when qty > 0 ->
        let existing = items |> List.tryFindIndex (fun (pid, _) -> pid = id)
        match existing with
        | Some i ->
            let (_, oldQty) = items.[i]
            HasItems (items |> List.mapi (fun j item -> if j = i then (id, oldQty + qty) else item))
        | None -> HasItems (items @ [(id, qty)])
    | HasItems items, RemoveItem id ->
        let newItems = items |> List.filter (fun (pid, _) -> pid <> id)
        if List.isEmpty newItems then Empty else HasItems newItems
    | other, _ -> other

// Property: Cart item count is always non-negative
[<Property>]
let ``Cart always has non-negative items`` (actions: CartAction list) =
    let finalState = 
        actions |> List.fold cartTransition Empty
    
    match finalState with
    | Empty -> true
    | HasItems items ->
        items |> List.forall (fun (_, qty) -> qty > 0)
```

---

## 6. Contract Testing with Pact

```fsharp
// ContractTests.fs
// Consumer-Driven Contract Testing

// ===== Consumer side (API client) =====

module UserApiClient =
    open System.Net.Http
    open System.Text.Json
    
    type User = {
        id: int
        name: string
        email: string
    }
    
    let getUser (http: HttpClient) (id: int) : Async<User option> =
        async {
            try
                let! response = 
                    http.GetAsync(sprintf "/api/users/%d" id)
                    |> Async.AwaitTask
                
                if response.IsSuccessStatusCode then
                    let! body = response.Content.ReadAsStringAsync() |> Async.AwaitTask
                    let user = JsonSerializer.Deserialize<User>(body)
                    return Some user
                else
                    return None
            with _ ->
                return None
        }
    
    let createUser (http: HttpClient) (name: string) (email: string) : Async<User option> =
        async {
            try
                let body = JsonSerializer.Serialize({| name = name; email = email |})
                use content = new StringContent(body, System.Text.Encoding.UTF8, "application/json")
                
                let! response =
                    http.PostAsync("/api/users", content)
                    |> Async.AwaitTask
                
                if response.IsSuccessStatusCode then
                    let! responseBody = response.Content.ReadAsStringAsync() |> Async.AwaitTask
                    let user = JsonSerializer.Deserialize<User>(responseBody)
                    return Some user
                else
                    return None
            with _ ->
                return None
        }

// ===== Pact Consumer Tests =====

// (ต้องการ PactNet library ซึ่งต้องการ setup เพิ่มเติม)
(*
open PactNet
open PactNet.Mocks.MockHttpService

type UserApiConsumerTests() =
    let mockServiceUri = Uri "http://localhost:9222"
    
    [<Fact>]
    member _.``Get user by id`` () =
        use pactBuilder = PactBuilder(new PactConfig {
            SpecificationVersion = "2.0.0"
            PactDir = "./pacts"
            LogDir = "./logs"
        })
        
        let pact = pactBuilder.ServiceConsumer("UserClient").HasPactWith("UserService")
        let mockService = pact.MockService(9222)
        
        mockService
            .Given("User 1 exists")
            .UponReceiving("a request for user 1")
            .With(new ProviderServiceRequest {
                Method = HttpVerb.Get
                Path = "/api/users/1"
            })
            .WillRespondWith(new ProviderServiceResponse {
                Status = 200
                Headers = dict ["Content-Type", "application/json"]
                Body = {| id = 1; name = "Alice"; email = "alice@example.com" |}
            })
        
        let http = new HttpClient { BaseAddress = mockServiceUri }
        let user = UserApiClient.getUser http 1 |> Async.RunSynchronously
        
        Assert.True(user.IsSome)
        Assert.Equal(1, user.Value.id)
        Assert.Equal("Alice", user.Value.name)
        
        mockService.VerifyInteractions()
*)
```

---

## 7. Mutation Testing

```fsharp
// MutationTestingExample.fs
// Mutation testing ตรวจสอบว่า tests ของเรา "แข็งแกร่ง" พอไหม
// โดยทำการ "mutate" code แล้วดูว่า tests fail หรือไม่

// ===== Code ที่จะ test =====

module Calculator =
    
    let add x y = x + y        // Mutation: + → -
    let subtract x y = x - y   // Mutation: - → +
    let multiply x y = x * y   // Mutation: * → /
    let divide x y =            // Mutation: y = 0 condition
        if y = 0 then failwith "Division by zero"
        else x / y
    
    let isPositive x = x > 0   // Mutation: > → >=
    let isEven x = x % 2 = 0   // Mutation: = 0 → <> 0

// ===== Tests ที่ "อ่อนแอ" (ไม่ detect mutations) =====

[<Fact>]
let ``Weak test - might not catch mutations`` () =
    // นี่เป็น test ที่อ่อนแอ เพราะตรวจสอบแค่ว่า function return บางอย่าง
    let result = Calculator.add 2 3
    Assert.NotEqual(0, result)  // ถ้า mutation เปลี่ยน + → -, result = -1 ซึ่ง <> 0 ยังผ่าน!

// ===== Tests ที่ "แข็งแกร่ง" (detect mutations) =====

[<Fact>]
let ``Strong test - detects mutations`` () =
    Assert.Equal(5, Calculator.add 2 3)         // ถ้า + → -, result = -1 ≠ 5 → FAIL
    Assert.Equal(-1, Calculator.subtract 2 3)    // ถ้า - → +, result = 5 ≠ -1 → FAIL
    Assert.Equal(6, Calculator.multiply 2 3)     // ถ้า * → /, result = 0 ≠ 6 → FAIL

[<Fact>]
let ``Division by zero test`` () =
    Assert.Throws<System.Exception>(fun () ->
        Calculator.divide 10 0 |> ignore)

[<Fact>]
let ``isPositive boundary tests`` () =
    Assert.True(Calculator.isPositive 1)    // ถ้า > → >=, still true
    Assert.False(Calculator.isPositive 0)   // ถ้า > → >=, would be true! → FAIL
    Assert.False(Calculator.isPositive -1)

[<Theory>]
[<InlineData(2, true)>]
[<InlineData(3, false)>]
[<InlineData(0, true)>]
[<InlineData(-2, true)>]
[<InlineData(-3, false)>]
let ``isEven parametric tests`` (n: int) (expected: bool) =
    Assert.Equal(expected, Calculator.isEven n)
```

---

## 8. Approval Testing (Golden Master)

```fsharp
// ApprovalTests.fs

module ApprovalTesting =
    open System.IO
    
    let approvalsDir = "./approvals"
    
    let private getApprovedPath name = 
        Path.Combine(approvalsDir, sprintf "%s.approved.txt" name)
    
    let private getReceivedPath name =
        Path.Combine(approvalsDir, sprintf "%s.received.txt" name)
    
    let verify (testName: string) (actual: string) =
        Directory.CreateDirectory(approvalsDir) |> ignore
        let approvedPath = getApprovedPath testName
        let receivedPath = getReceivedPath testName
        
        // เสมอบันทึก received
        File.WriteAllText(receivedPath, actual)
        
        if not (File.Exists(approvedPath)) then
            // ยังไม่มี approved: fail และบอกให้ approve
            failwith (sprintf 
                "No approved file for '%s'. Review received output at %s and copy to %s to approve."
                testName receivedPath approvedPath)
        else
            let approved = File.ReadAllText(approvedPath)
            if approved = actual then
                // ลบ received เมื่อ pass
                if File.Exists(receivedPath) then File.Delete(receivedPath)
            else
                failwith (sprintf 
                    "Approval mismatch for '%s'. Expected:\n%s\n\nActual:\n%s\n\nIf this is correct, copy received to approved."
                    testName approved actual)

// ===== Complex Output Testing =====

type ReportFormatter =
    static member FormatOrderReport (orders: Order list) =
        let sb = System.Text.StringBuilder()
        sb.AppendLine("=== Order Report ===") |> ignore
        sb.AppendLine() |> ignore
        
        for order in orders |> List.sortBy (fun o -> o.CreatedAt) do
            sb.AppendLine(sprintf "Order ID: %s" (order.Id.ToString("N").[..7])) |> ignore
            sb.AppendLine(sprintf "Status: %A" order.Status) |> ignore
            sb.AppendLine(sprintf "Items:") |> ignore
            
            for item in order.Items do
                sb.AppendLine(sprintf "  - %s x%d @ ฿%.2f" item.ProductName item.Quantity item.UnitPrice) |> ignore
            
            sb.AppendLine(sprintf "Total: ฿%.2f" order.TotalAmount) |> ignore
            sb.AppendLine() |> ignore
        
        sb.ToString()

[<Fact>]
let ``Order report format`` () =
    let customerId = System.Guid.Parse("00000000-0000-0000-0000-000000000001")
    let orders = [
        OrderBuilder.create customerId
        |> OrderBuilder.withStatus Delivered
        |> OrderBuilder.build
    ]
    
    let report = ReportFormatter.FormatOrderReport orders
    
    ApprovalTesting.verify "order-report-format" report
```

---

## 9. Fuzz Testing

```fsharp
// FuzzTesting.fs

// ===== Basic Fuzzing =====

let fuzzTest (testFn: string -> unit) (seeds: string list) =
    let rng = System.Random(42)
    
    // รัน seeds
    for seed in seeds do
        try testFn seed
        with ex -> printfn "Seed '%s' failed: %s" seed ex.Message
    
    // Generate random inputs
    for _ in 1..100 do
        let length = rng.Next(1, 100)
        let chars = Array.init length (fun _ -> char (rng.Next(32, 127)))
        let input = System.String(chars)
        try testFn input
        with ex -> printfn "Random input '%s' failed: %s" input ex.Message

// ===== Fuzz Parser =====

let fuzzJsonParser () =
    let seeds = [
        "{}"
        "[]"
        """{"key": "value"}"""
        "null"
        "true"
        "false"
        "123"
        """"string""""
        """{"nested": {"key": "value"}}"""
        "[1, 2, 3]"
    ]
    
    // Parser ควร handle ทุก input โดยไม่ throw unexpected exceptions
    fuzzTest (fun input ->
        try
            let result = parseJson input
            // บอกว่า parsed หรือ error อย่างสง่างาม
            ()
        with 
        | :? System.ArgumentException -> ()  // OK: invalid input
        | ex -> failwith (sprintf "Unexpected exception for input '%s': %s" input ex.Message)
    ) seeds

// ===== Property-Based Fuzzing =====

open FsCheck

[<Property>]
let ``Parser handles any string input`` (input: string) =
    let input = if isNull input then "" else input
    
    try
        // ไม่ควร throw exception ที่ไม่คาดหมาย
        let _ = parseJson input
        true
    with
    | :? System.ArgumentException -> true  // Expected error
    | :? System.FormatException -> true    // Expected error
    | ex ->
        printfn "Unexpected exception for input '%s': %s" input ex.Message
        false
```

---

## 10. Test Organization

```fsharp
// TestOrganization.fs

// ===== Arrange-Act-Assert Pattern =====

[<Fact>]
let ``Order calculation with discount`` () =
    // Arrange
    let customer = TestCustomers.vipCustomer()
    let items = [
        OrderItemBuilder.create()
        |> OrderItemBuilder.withProduct 1 "Laptop"
        |> OrderItemBuilder.withPrice 50000m
        |> OrderItemBuilder.withQuantity 1
        |> OrderItemBuilder.build
        
        OrderItemBuilder.create()
        |> OrderItemBuilder.withProduct 2 "Mouse"
        |> OrderItemBuilder.withPrice 500m
        |> OrderItemBuilder.withQuantity 2
        |> OrderItemBuilder.build
    ]
    
    // Act
    let order = { 
        Id = System.Guid.NewGuid()
        CustomerId = customer.Id
        Items = items
        Status = Pending
        CreatedAt = System.DateTime.Now
        TotalAmount = 51000m
    }
    
    let discount = calculateVipDiscount customer order
    let finalAmount = order.TotalAmount - discount
    
    // Assert
    Assert.True(discount > 0m, "VIP customer should get discount")
    Assert.Equal(51000m * 0.9m, finalAmount)  // 10% VIP discount

// ===== Behavior Categories =====

module HappyPath =
    [<Fact>]
    let ``Valid order is processed successfully`` () = 
        let customer = TestCustomers.regularCustomer()
        let order = TestOrders.pendingOrder customer.Id
        let result = processOrder customer order
        Assert.Equal(Ok Confirmed, result)

module EdgeCases =
    [<Fact>]
    let ``Empty order cannot be placed`` () =
        let customer = TestCustomers.regularCustomer()
        let emptyOrder = 
            OrderBuilder.create customer.Id
            |> OrderBuilder.clearItems
            |> OrderBuilder.build
        let result = processOrder customer emptyOrder
        Assert.Equal(Error "Order must have at least one item", result)

module ErrorCases =
    [<Fact>]
    let ``Inactive customer cannot place order`` () =
        let customer = TestCustomers.inactiveCustomer()
        let order = TestOrders.pendingOrder customer.Id
        let result = processOrder customer order
        Assert.Equal(Error "Customer account is inactive", result)

// ===== Test Collections =====

[<Collection("Integration Tests")>]
type IntegrationTestCollection() = class end

[<CollectionDefinition("Integration Tests", DisableParallelization = true)>]
type IntegrationTests() =
    interface ICollectionFixture<DatabaseFixture>
```

---

## สรุป (Summary)

Testing Patterns ขั้นสูงที่ครอบคลุมใน Part นี้:

1. **Test Data Builders**: สร้าง test data แบบ fluent, reusable
2. **Object Mother**: Pre-configured test objects สำหรับ scenarios ต่างๆ
3. **Test Fixtures**: Setup/teardown สำหรับ shared resources
4. **Snapshot Testing**: Verify complex output ด้วยการเปรียบเทียบกับ "snapshot"
5. **Property-Based Testing**: ใช้ FsCheck generate test cases จาก properties
6. **Contract Testing**: Consumer-driven contracts กับ Pact
7. **Mutation Testing**: ตรวจสอบความแข็งแกร่งของ test suite
8. **Approval Testing**: Golden master testing สำหรับ complex output
9. **Fuzz Testing**: ทดสอบด้วย random/unexpected inputs

---

*ไปต่อที่ Part 109: Functional Patterns ขั้นสูง*
