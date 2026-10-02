# Part 73 - Expecto Testing Framework

## บทนำ (Introduction)

Expecto เป็น F#-native testing framework ที่ออกแบบมาเพื่อ F# โดยเฉพาะ ต่างจาก xUnit ที่เป็น OOP-based, Expecto ใช้ functional approach ในการเขียน tests

**คุณสมบัติหลักของ Expecto:**
- Tests เป็น values (ไม่ใช่ methods ที่มี attributes)
- รองรับ parallel execution
- มี built-in logging
- รองรับ async tests แบบ native
- Performance testing built-in
- Stress testing

---

## 1. การติดตั้ง Expecto (Installation)

### .fsproj Setup

```xml
<!-- Tests.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>

  <ItemGroup>
    <!-- Expecto core -->
    <PackageReference Include="Expecto" Version="10.1.0" />
    <!-- Optional: Better logging -->
    <PackageReference Include="Expecto.Logging" Version="10.1.0" />
    <!-- Optional: FsCheck integration -->
    <PackageReference Include="Expecto.FsCheck" Version="10.1.0" />
    <!-- Optional: BenchmarkDotNet integration -->
    <PackageReference Include="Expecto.BenchmarkDotNet" Version="10.1.0" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Domain.fs" />
    <Compile Include="BasicTests.fs" />
    <Compile Include="AsyncTests.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

### Program.fs - Entry Point

```fsharp
// Program.fs
module Tests.Program

open Expecto

[<EntryPoint>]
let main argv =
    runTestsInAssemblyWithCLIArgs [] argv
```

### การรัน Tests

```bash
# รัน tests ทั้งหมด
dotnet run

# รันพร้อม filter
dotnet run -- --filter "basic"

# รันแบบ sequential
dotnet run -- --sequenced

# รันพร้อม verbose output
dotnet run -- --verbosity verbose

# รัน stress test
dotnet run -- --stress 30  # รัน 30 วินาที

# แสดง test list
dotnet run -- --list-tests
```

---

## 2. Test Definitions: test, testCase

```fsharp
// BasicTests.fs
module Tests.BasicTests

open Expecto

// testCase - สร้าง test case พื้นฐาน
let basicTest1 = testCase "one plus one equals two" <| fun () ->
    Expect.equal (1 + 1) 2 "1 + 1 should equal 2"

// test - alias สำหรับ testCase
let basicTest2 = test "two times three equals six" {
    Expect.equal (2 * 3) 6 "2 * 3 should equal 6"
}

// testCase กับ named function
let addTest () =
    testCase "add function works correctly" <| fun () ->
        let add x y = x + y
        Expect.equal (add 3 4) 7 "3 + 4 should be 7"

// สร้าง tests แบบ functional
let arithmeticTests = testList "Arithmetic" [
    testCase "addition" <| fun () ->
        Expect.equal (1 + 2) 3 "addition failed"
    
    testCase "subtraction" <| fun () ->
        Expect.equal (5 - 3) 2 "subtraction failed"
    
    testCase "multiplication" <| fun () ->
        Expect.equal (4 * 5) 20 "multiplication failed"
    
    testCase "division" <| fun () ->
        Expect.equal (10 / 2) 5 "division failed"
]

// เรียกใช้ใน Program.fs
// runTests defaultConfig arithmeticTests
```

---

## 3. testAsync สำหรับ Async Tests

```fsharp
// AsyncTests.fs
module Tests.AsyncTests

open Expecto
open System.Threading.Tasks

// Async function ที่จะทดสอบ
let fetchDataAsync (id: int) = async {
    do! Async.Sleep 10
    return sprintf "Data for id %d" id
}

let processAsync items = async {
    let! results = 
        items
        |> List.map (fun x -> async { return x * 2 })
        |> Async.Parallel
    return results |> Array.toList
}

let validateAsync (value: int) = async {
    do! Async.Sleep 5
    if value < 0 then return Error "Value must be non-negative"
    else return Ok value
}

// testAsync tests
let asyncTests = testList "Async Tests" [
    
    testAsync "fetch data returns correct result" {
        let! result = fetchDataAsync 42
        Expect.equal result "Data for id 42" "Unexpected data"
    }
    
    testAsync "process items doubles each element" {
        let! result = processAsync [1; 2; 3; 4; 5]
        Expect.equal result [2; 4; 6; 8; 10] "Processing failed"
    }
    
    testAsync "validate positive number returns Ok" {
        let! result = validateAsync 5
        match result with
        | Ok value -> Expect.equal value 5 "Wrong value"
        | Error msg -> failtest (sprintf "Unexpected error: %s" msg)
    }
    
    testAsync "validate negative number returns Error" {
        let! result = validateAsync (-1)
        match result with
        | Error msg -> Expect.isTrue (msg.Length > 0) "Error message should not be empty"
        | Ok _ -> failtest "Expected Error but got Ok"
    }
    
    // Task-based async
    testCaseAsync "task based test" <| fun () ->
        task {
            let! result = Task.FromResult(42)
            Expect.equal result 42 "Task result should be 42"
        } |> Async.AwaitTask
    
    // Multiple async operations
    testAsync "parallel operations complete correctly" {
        let operations = 
            [1..5] |> List.map (fun i -> async {
                do! Async.Sleep (i * 10)
                return i * i
            })
        
        let! results = Async.Parallel operations
        Expect.equal (Array.sum results) 55 "Sum of squares 1-5 should be 55"
    }
]
```

---

## 4. testList สำหรับ Test Grouping

```fsharp
// GroupingTests.fs
module Tests.GroupingTests

open Expecto

// Domain model
type Product = {
    Id: int
    Name: string
    Price: float
    Category: string
    InStock: bool
}

let createProduct id name price category inStock =
    { Id = id; Name = name; Price = price; Category = category; InStock = inStock }

// Nested testList สำหรับ organization
let productTests = testList "Product" [
    
    testList "Creation" [
        testCase "create product with all fields" <| fun () ->
            let p = createProduct 1 "Widget" 9.99 "Electronics" true
            Expect.equal p.Name "Widget" "Name mismatch"
            Expect.equal p.Price 9.99 "Price mismatch"
        
        testCase "product is in stock by default" <| fun () ->
            let p = createProduct 1 "Widget" 9.99 "Electronics" true
            Expect.isTrue p.InStock "Should be in stock"
    ]
    
    testList "Pricing" [
        testCase "price is positive" <| fun () ->
            let p = createProduct 1 "Widget" 9.99 "Electronics" true
            Expect.isTrue (p.Price > 0.0) "Price should be positive"
        
        testCase "discounted price is less" <| fun () ->
            let p = createProduct 1 "Widget" 100.0 "Electronics" true
            let discounted = { p with Price = p.Price * 0.9 }
            Expect.isTrue (discounted.Price < p.Price) "Discounted price should be less"
        
        testCase "bulk pricing" <| fun () ->
            let unitPrice = 10.0
            let bulkPrice = unitPrice * 0.85
            let total = bulkPrice * 10.0
            Expect.equal total 85.0 "Bulk total should be 85"
    ]
    
    testList "Inventory" [
        testCase "out of stock product" <| fun () ->
            let p = createProduct 1 "Widget" 9.99 "Electronics" false
            Expect.isFalse p.InStock "Should be out of stock"
        
        testCase "can change stock status" <| fun () ->
            let p = createProduct 1 "Widget" 9.99 "Electronics" true
            let outOfStock = { p with InStock = false }
            Expect.isFalse outOfStock.InStock "Should be out of stock after update"
    ]
]

// Deep nesting
let shoppingTests = testList "Shopping" [
    testList "Cart" [
        testList "Add Items" [
            testCase "add single item" <| fun () ->
                let items = ["Widget"]
                Expect.hasLength items 1 "Should have 1 item"
            
            testCase "add multiple items" <| fun () ->
                let items = ["Widget"; "Gadget"; "Gizmo"]
                Expect.hasLength items 3 "Should have 3 items"
        ]
        
        testList "Remove Items" [
            testCase "remove item from list" <| fun () ->
                let items = ["Widget"; "Gadget"]
                let updated = items |> List.filter (fun x -> x <> "Widget")
                Expect.hasLength updated 1 "Should have 1 item after removal"
        ]
        
        testList "Calculate Total" [
            testCase "empty cart total is zero" <| fun () ->
                let prices: float list = []
                let total = List.sum prices
                Expect.equal total 0.0 "Empty cart should be zero"
        ]
    ]
    
    testList "Checkout" [
        testCase "checkout creates order" <| fun () ->
            let items = ["Widget"]
            let order = { | Items = items; Status = "pending" |}  // Record expression
            Expect.equal order.Status "pending" "New order should be pending"
    ]
]
```

---

## 5. testSequenced สำหรับ Sequential Tests

```fsharp
// SequencedTests.fs
module Tests.SequencedTests

open Expecto

// State ที่ต้องการ sequence
let mutable globalCounter = 0

// Tests ที่ต้อง run ตามลำดับ
let sequencedTests = testSequenced <| testList "Sequential Counter" [
    testCase "counter starts at zero" <| fun () ->
        Expect.equal globalCounter 0 "Counter should start at 0"
    
    testCase "increment counter" <| fun () ->
        globalCounter <- globalCounter + 1
        Expect.equal globalCounter 1 "Counter should be 1"
    
    testCase "increment counter again" <| fun () ->
        globalCounter <- globalCounter + 1
        Expect.equal globalCounter 2 "Counter should be 2"
    
    testCase "reset counter" <| fun () ->
        globalCounter <- 0
        Expect.equal globalCounter 0 "Counter should be back to 0"
]

// Database operations ที่ต้อง run ตามลำดับ
type SimpleDB = {
    mutable Records: Map<int, string>
}

let db = { Records = Map.empty }

let dbTests = testSequenced <| testList "Database Operations" [
    testCase "database starts empty" <| fun () ->
        Expect.isEmpty db.Records "DB should be empty"
    
    testCase "insert record" <| fun () ->
        db.Records <- db.Records |> Map.add 1 "First Record"
        Expect.equal db.Records.Count 1 "Should have 1 record"
    
    testCase "insert another record" <| fun () ->
        db.Records <- db.Records |> Map.add 2 "Second Record"
        Expect.equal db.Records.Count 2 "Should have 2 records"
    
    testCase "delete record" <| fun () ->
        db.Records <- db.Records |> Map.remove 1
        Expect.equal db.Records.Count 1 "Should have 1 record after delete"
    
    testCase "update record" <| fun () ->
        db.Records <- db.Records |> Map.add 2 "Updated Record"
        Expect.equal db.Records.[2] "Updated Record" "Record should be updated"
]
```

---

## 6. Expect Module - การใช้งาน

```fsharp
// ExpectTests.fs
module Tests.ExpectTests

open Expecto

let expectTests = testList "Expect Module" [
    
    // === Equality ===
    testCase "equal" <| fun () ->
        Expect.equal 42 42 "Should be equal"
        Expect.equal "hello" "hello" "Strings should match"
        Expect.equal [1;2;3] [1;2;3] "Lists should match"
    
    testCase "notEqual" <| fun () ->
        Expect.notEqual 1 2 "Should not be equal"
        Expect.notEqual "a" "b" "Strings should differ"
    
    // === Boolean ===
    testCase "isTrue" <| fun () ->
        Expect.isTrue true "Should be true"
        Expect.isTrue (1 < 2) "1 is less than 2"
        Expect.isTrue ([1;2;3] |> List.contains 2) "List should contain 2"
    
    testCase "isFalse" <| fun () ->
        Expect.isFalse false "Should be false"
        Expect.isFalse (1 > 2) "1 is not greater than 2"
    
    // === Comparisons ===
    testCase "isGreaterThan" <| fun () ->
        Expect.isGreaterThan 10 5 "10 > 5"
        Expect.isGreaterThan 3.14 2.71 "Pi > e"
    
    testCase "isLessThan" <| fun () ->
        Expect.isLessThan 3 10 "3 < 10"
        Expect.isLessThan -1 0 "-1 < 0"
    
    testCase "isGreaterThanOrEqual" <| fun () ->
        Expect.isGreaterThanOrEqual 5 5 "5 >= 5"
        Expect.isGreaterThanOrEqual 6 5 "6 >= 5"
    
    testCase "isLessThanOrEqual" <| fun () ->
        Expect.isLessThanOrEqual 5 5 "5 <= 5"
        Expect.isLessThanOrEqual 4 5 "4 <= 5"
    
    // === Float ===
    testCase "floatClose" <| fun () ->
        Expect.floatClose Accuracy.medium 3.14159 System.Math.PI "Pi approximation"
        Expect.floatClose Accuracy.low 2.71 System.Math.E "E approximation"
    
    // === Strings ===
    testCase "stringContains" <| fun () ->
        Expect.stringContains "Hello, World!" "World" "Should contain World"
        Expect.stringStarts "Hello" "He" "Should start with He"
        Expect.stringEnds "Hello" "lo" "Should end with lo"
    
    // === Collections ===
    testCase "isEmpty" <| fun () ->
        Expect.isEmpty [] "Empty list"
        Expect.isEmpty "" "Empty string"
        Expect.isEmpty [||] "Empty array"
    
    testCase "isNonEmpty" <| fun () ->
        Expect.isNonEmpty [1;2;3] "Non-empty list"
        Expect.isNonEmpty "hello" "Non-empty string"
    
    testCase "hasLength" <| fun () ->
        Expect.hasLength [1;2;3] 3 "Should have 3 elements"
        Expect.hasLength "hello" 5 "Should have 5 chars"
    
    testCase "contains" <| fun () ->
        Expect.contains [1;2;3;4;5] 3 "Should contain 3"
    
    testCase "all" <| fun () ->
        let numbers = [2;4;6;8;10]
        Expect.all numbers (fun x -> x % 2 = 0) "All should be even"
    
    testCase "exists" <| fun () ->
        let numbers = [1;2;3;4;5]
        Expect.exists numbers (fun x -> x > 4) "Should have element > 4"
    
    // === Option ===
    testCase "isSome" <| fun () ->
        Expect.isSome (Some 42) "Should be Some"
    
    testCase "isNone" <| fun () ->
        Expect.isNone None "Should be None"
    
    testCase "wantSome" <| fun () ->
        let value = Expect.wantSome (Some 42) "Should unwrap to 42"
        Expect.equal value 42 "Unwrapped value should be 42"
    
    // === Result ===
    testCase "isOk" <| fun () ->
        Expect.isOk (Ok 42) "Should be Ok"
    
    testCase "isError" <| fun () ->
        Expect.isError (Error "oops") "Should be Error"
    
    testCase "wantOk" <| fun () ->
        let value = Expect.wantOk (Ok "success") "Should unwrap Ok"
        Expect.equal value "success" "Should be 'success'"
    
    testCase "wantError" <| fun () ->
        let error = Expect.wantError (Error "failure") "Should unwrap Error"
        Expect.equal error "failure" "Should be 'failure'"
]
```

---

## 7. expect equal, notEqual

```fsharp
// EqualityTests.fs
module Tests.EqualityTests

open Expecto

// Domain
type Money = { Amount: float; Currency: string }

let createMoney amount currency = { Amount = amount; Currency = currency }

let addMoney m1 m2 =
    if m1.Currency <> m2.Currency then
        Error (sprintf "Cannot add %s and %s" m1.Currency m2.Currency)
    else
        Ok { Amount = m1.Amount + m2.Amount; Currency = m1.Currency }

let equalityTests = testList "Equality Tests" [
    
    // Basic values
    testCase "integers are equal" <| fun () ->
        Expect.equal 42 42 "Integers should be equal"
    
    testCase "integers are not equal" <| fun () ->
        Expect.notEqual 1 2 "1 and 2 should differ"
    
    // Records
    testCase "money records are equal" <| fun () ->
        let m1 = createMoney 100.0 "USD"
        let m2 = createMoney 100.0 "USD"
        Expect.equal m1 m2 "Same money values should be equal"
    
    testCase "money records with different amounts are not equal" <| fun () ->
        let m1 = createMoney 100.0 "USD"
        let m2 = createMoney 200.0 "USD"
        Expect.notEqual m1 m2 "Different amounts should not be equal"
    
    // Lists
    testCase "lists are equal" <| fun () ->
        Expect.equal [1;2;3] [1;2;3] "Lists should be equal"
    
    testCase "lists with different order are not equal" <| fun () ->
        Expect.notEqual [1;2;3] [3;2;1] "Different order should not be equal"
    
    // Discriminated unions
    testCase "discriminated unions are equal" <| fun () ->
        Expect.equal (Some 42) (Some 42) "Options should be equal"
        Expect.equal None (None: int option) "None values should be equal"
        Expect.notEqual (Some 1) (Some 2) "Different Some values differ"
    
    // Adding money
    testCase "adding same currency works" <| fun () ->
        let m1 = createMoney 100.0 "USD"
        let m2 = createMoney 50.0 "USD"
        let result = addMoney m1 m2
        Expect.isOk result "Should succeed"
        let sum = Expect.wantOk result "Sum"
        Expect.equal sum.Amount 150.0 "Sum should be 150"
        Expect.equal sum.Currency "USD" "Currency should be USD"
    
    testCase "adding different currencies fails" <| fun () ->
        let m1 = createMoney 100.0 "USD"
        let m2 = createMoney 50.0 "EUR"
        let result = addMoney m1 m2
        Expect.isError result "Should fail for different currencies"
]
```

---

## 8. expect isTrue, isFalse

```fsharp
// BooleanTests.fs
module Tests.BooleanTests

open Expecto

// Business logic functions
let isValidEmail (email: string) =
    email.Contains("@") && email.Contains(".") && email.Length >= 5

let isAdult age = age >= 18

let isPalindrome (s: string) =
    let cleaned = s.ToLower() |> Seq.filter System.Char.IsLetterOrDigit |> Seq.toArray
    cleaned = Array.rev cleaned

let isLeapYear year =
    (year % 4 = 0 && year % 100 <> 0) || (year % 400 = 0)

let booleanTests = testList "Boolean Tests" [
    
    testList "Email validation" [
        testCase "valid email is valid" <| fun () ->
            Expect.isTrue (isValidEmail "test@example.com") "Should be valid email"
        
        testCase "email without @ is invalid" <| fun () ->
            Expect.isFalse (isValidEmail "notanemail.com") "Should be invalid"
        
        testCase "empty string is invalid" <| fun () ->
            Expect.isFalse (isValidEmail "") "Empty string is invalid"
    ]
    
    testList "Age validation" [
        testCase "18 is adult" <| fun () ->
            Expect.isTrue (isAdult 18) "18 should be adult"
        
        testCase "17 is not adult" <| fun () ->
            Expect.isFalse (isAdult 17) "17 should not be adult"
        
        testCase "0 is not adult" <| fun () ->
            Expect.isFalse (isAdult 0) "0 should not be adult"
    ]
    
    testList "Palindrome check" [
        testCase "racecar is palindrome" <| fun () ->
            Expect.isTrue (isPalindrome "racecar") "racecar is palindrome"
        
        testCase "hello is not palindrome" <| fun () ->
            Expect.isFalse (isPalindrome "hello") "hello is not palindrome"
        
        testCase "A man a plan a canal Panama is palindrome" <| fun () ->
            Expect.isTrue (isPalindrome "A man a plan a canal Panama") "Should be palindrome"
    ]
    
    testList "Leap year" [
        testCase "2000 is leap year" <| fun () ->
            Expect.isTrue (isLeapYear 2000) "2000 is leap year"
        
        testCase "1900 is not leap year" <| fun () ->
            Expect.isFalse (isLeapYear 1900) "1900 is not leap year"
        
        testCase "2024 is leap year" <| fun () ->
            Expect.isTrue (isLeapYear 2024) "2024 is leap year"
        
        testCase "2023 is not leap year" <| fun () ->
            Expect.isFalse (isLeapYear 2023) "2023 is not leap year"
    ]
]
```

---

## 9. expect throws

```fsharp
// ThrowTests.fs
module Tests.ThrowTests

open Expecto
open System

// Functions that may throw
let unsafeDivide (x: int) (y: int) =
    if y = 0 then raise (DivideByZeroException("Cannot divide by zero"))
    x / y

let getById (id: int) (items: Map<int, string>) =
    match items |> Map.tryFind id with
    | Some item -> item
    | None -> raise (KeyNotFoundException(sprintf "Item with id %d not found" id))

let parsePositive (s: string) =
    let n = int s  // May throw FormatException
    if n <= 0 then raise (ArgumentException("Must be positive"))
    n

let throwTests = testList "Throw Tests" [
    
    testCase "divide by zero throws" <| fun () ->
        Expect.throws (fun () -> unsafeDivide 10 0 |> ignore) "Should throw on division by zero"
    
    testCase "divide by zero throws DivideByZeroException" <| fun () ->
        Expect.throwsT<DivideByZeroException> 
            (fun () -> unsafeDivide 10 0 |> ignore) 
            "Should throw DivideByZeroException"
    
    testCase "missing item throws" <| fun () ->
        let items = Map.ofList [(1, "Item 1")]
        Expect.throws 
            (fun () -> getById 999 items |> ignore) 
            "Should throw for missing item"
    
    testCase "invalid format throws FormatException" <| fun () ->
        Expect.throwsT<FormatException>
            (fun () -> parsePositive "not a number" |> ignore)
            "Should throw FormatException for invalid format"
    
    testCase "negative value throws ArgumentException" <| fun () ->
        Expect.throwsT<ArgumentException>
            (fun () -> parsePositive "-5" |> ignore)
            "Should throw ArgumentException for negative"
    
    // Valid cases don't throw
    testCase "valid division does not throw" <| fun () ->
        let result = unsafeDivide 10 2
        Expect.equal result 5 "Should return 5"
    
    testCase "existing item does not throw" <| fun () ->
        let items = Map.ofList [(1, "Item 1"); (2, "Item 2")]
        let result = getById 1 items
        Expect.equal result "Item 1" "Should return Item 1"
    
    // Async throws
    testCaseAsync "async throws exception" <| fun () -> async {
        let! result = async {
            try
                do! Async.Sleep 10
                failwith "async error"
                return "never"
            with ex ->
                return ex.Message
        }
        Expect.stringContains result "async error" "Should catch async error"
    }
]
```

---

## 10. Running Tests

```fsharp
// Program.fs
module Tests.Program

open Expecto

// Import all test modules
open Tests.BasicTests
open Tests.AsyncTests
open Tests.GroupingTests
open Tests.ExpectTests
open Tests.EqualityTests
open Tests.BooleanTests
open Tests.ThrowTests

// รวม tests ทั้งหมด
let allTests = testList "All Tests" [
    arithmeticTests
    asyncTests
    productTests
    expectTests
    equalityTests
    booleanTests
    throwTests
]

[<EntryPoint>]
let main argv =
    // Default config
    runTestsWithCLIArgs [] argv allTests

// Custom config
let customConfig = {
    defaultConfig with
        verbosity = Logging.LogLevel.Info
        failOnFocusedTests = true
        printer = TestPrinters.defaultPrinter
}

// Alternative entry with custom config
// runTests customConfig allTests
```

---

## 11. Test Filtering

```fsharp
// FilteringDemo.fs
module Tests.FilteringDemo

open Expecto

// Tests with labels for filtering
let integrationTests = testList "Integration" [
    testCase "database connection" <| fun () ->
        // Would need real DB
        Expect.isTrue true "Placeholder"
    
    testCase "API call" <| fun () ->
        Expect.isTrue true "Placeholder"
]

let unitTests = testList "Unit" [
    testCase "pure function" <| fun () ->
        let double x = x * 2
        Expect.equal (double 5) 10 "Should double"
    
    testCase "string operation" <| fun () ->
        let upper (s: string) = s.ToUpper()
        Expect.equal (upper "hello") "HELLO" "Should uppercase"
]

let slowTests = testList "Slow" [
    testCaseAsync "simulated slow test" <| fun () -> async {
        do! Async.Sleep 100
        Expect.isTrue true "Completed"
    }
]

// การรัน:
// dotnet run -- --filter "Unit"     # รัน unit tests เท่านั้น
// dotnet run -- --filter "Slow"     # รัน slow tests เท่านั้น  
// dotnet run -- --filter-test-list "Integration" # filter by list name
```

---

## 12. Parallel Test Execution

```fsharp
// ParallelTests.fs
module Tests.ParallelTests

open Expecto
open System.Diagnostics

// Tests ที่ run parallel ได้ (default ใน Expecto)
let parallelTests = testList "Parallel Tests" [
    
    testCase "independent test 1" <| fun () ->
        let result = [1..100] |> List.sum
        Expect.equal result 5050 "Sum 1 to 100"
    
    testCase "independent test 2" <| fun () ->
        let result = [1..10] |> List.map (fun x -> x * x) |> List.sum
        Expect.equal result 385 "Sum of squares"
    
    testCase "independent test 3" <| fun () ->
        let isPrime n =
            if n < 2 then false
            else [2..int (sqrt (float n))] |> List.forall (fun i -> n % i <> 0)
        
        let primes = [2..50] |> List.filter isPrime
        Expect.hasLength primes 15 "Should have 15 primes up to 50"
]

// Tests ที่ต้อง run sequenced
let sequencedTests = testSequenced <| testList "Sequential" [
    testCase "step 1" <| fun () ->
        Expect.isTrue true "Step 1"
    
    testCase "step 2" <| fun () ->
        Expect.isTrue true "Step 2"
    
    testCase "step 3" <| fun () ->
        Expect.isTrue true "Step 3"
]

// Measuring parallel benefit
let benchmarkParallelism () =
    let sw = Stopwatch.StartNew()
    
    let tests = testList "Timing" [
        for i in 1..5 do
            testCaseAsync (sprintf "async task %d" i) <| fun () -> async {
                do! Async.Sleep 100
                Expect.isTrue true "Completed"
            }
    ]
    
    sw.Stop()
    printfn "Setup time: %dms" sw.ElapsedMilliseconds
    tests
```

---

## 13. Logging in Tests

```fsharp
// LoggingTests.fs
module Tests.LoggingTests

open Expecto
open Expecto.Logging
open Expecto.Logging.Message

// สร้าง logger
let logger = Log.create "Tests"

let loggingTests = testList "Logging Tests" [
    
    testCase "test with debug logging" <| fun () ->
        logger.debug (eventX "Starting test")
        
        let result = 2 + 2
        
        logger.debug (eventX "Computed result {result}" >> setField "result" result)
        
        Expect.equal result 4 "2 + 2 = 4"
    
    testCase "test with info logging" <| fun () ->
        logger.info (eventX "Processing items")
        
        let items = [1..10]
        let sum = List.sum items
        
        logger.info (eventX "Processed {count} items, sum = {sum}" 
                    >> setField "count" items.Length 
                    >> setField "sum" sum)
        
        Expect.equal sum 55 "Sum 1-10 = 55"
    
    testCase "test with warn logging" <| fun () ->
        let value = -5
        
        if value < 0 then
            logger.warn (eventX "Negative value encountered: {value}" 
                        >> setField "value" value)
        
        Expect.isTrue (value < 0) "Value is negative"
    
    testCase "test with error logging" <| fun () ->
        try
            let _ = 1 / 0
            ()
        with ex ->
            logger.error (eventX "Error occurred: {message}" >> setField "message" ex.Message)
        
        Expect.isTrue true "Error was caught and logged"
    
    // Logging timing
    testCaseAsync "test with timing" <| fun () -> async {
        let sw = System.Diagnostics.Stopwatch.StartNew()
        
        do! Async.Sleep 50
        
        sw.Stop()
        logger.info (eventX "Operation took {ms}ms" >> setField "ms" sw.ElapsedMilliseconds)
        
        Expect.isTrue (sw.ElapsedMilliseconds >= 50L) "Should take at least 50ms"
    }
]
```

---

## 14. Performance Tests กับ Expecto

```fsharp
// PerformanceTests.fs
module Tests.PerformanceTests

open Expecto

// Functions to benchmark
let bubbleSort (arr: int array) =
    let a = Array.copy arr
    let n = a.Length
    for i in 0..n-2 do
        for j in 0..n-i-2 do
            if a.[j] > a.[j+1] then
                let tmp = a.[j]
                a.[j] <- a.[j+1]
                a.[j+1] <- tmp
    a

let quickSort arr =
    let rec sort = function
        | [] -> []
        | pivot :: rest ->
            let smaller = rest |> List.filter (fun x -> x <= pivot)
            let larger = rest |> List.filter (fun x -> x > pivot)
            sort smaller @ [pivot] @ sort larger
    sort arr

// Performance tests (ค่า max ที่ยอมรับได้)
let performanceTests = testList "Performance" [
    
    testCase "bubble sort 1000 elements" <| fun () ->
        let arr = Array.init 1000 (fun i -> 1000 - i)
        
        let sw = System.Diagnostics.Stopwatch.StartNew()
        let sorted = bubbleSort arr
        sw.Stop()
        
        // Assert correctness
        Expect.equal sorted.[0] 1 "First element should be 1"
        Expect.equal sorted.[999] 1000 "Last element should be 1000"
        
        // Assert performance (< 1 second for 1000 elements)
        Expect.isTrue (sw.ElapsedMilliseconds < 1000L) 
            (sprintf "Bubble sort took %dms, expected < 1000ms" sw.ElapsedMilliseconds)
    
    testCase "quick sort 1000 elements" <| fun () ->
        let lst = [1000..-1..1]
        
        let sw = System.Diagnostics.Stopwatch.StartNew()
        let sorted = quickSort lst
        sw.Stop()
        
        Expect.equal sorted.[0] 1 "First element should be 1"
        Expect.equal (List.last sorted) 1000 "Last element should be 1000"
        
        // Quick sort should be faster than bubble sort
        Expect.isTrue (sw.ElapsedMilliseconds < 100L) 
            (sprintf "Quick sort took %dms, expected < 100ms" sw.ElapsedMilliseconds)
    
    // Memory performance
    testCase "no excessive allocations" <| fun () ->
        let gcBefore = System.GC.GetTotalMemory(true)
        
        for _ in 1..1000 do
            let _ = [1..100] |> List.sum
            ()
        
        let gcAfter = System.GC.GetTotalMemory(false)
        let allocated = gcAfter - gcBefore
        
        // Allow up to 10MB for 1000 iterations
        Expect.isTrue (allocated < 10_000_000L)
            (sprintf "Allocated %d bytes, expected < 10MB" allocated)
]
```

---

## 15. Stress Testing

```fsharp
// StressTests.fs
module Tests.StressTests

open Expecto

// A service ที่ต้องการ stress test
type Counter() =
    let mutable value = 0
    let lockObj = obj()
    
    member _.Increment() =
        lock lockObj (fun () -> value <- value + 1)
    
    member _.Value = value

// Thread-safe collection
let threadSafeAdd (collection: System.Collections.Concurrent.ConcurrentBag<int>) items =
    items |> List.iter collection.Add

let stressTests = testList "Stress Tests" [
    
    // Stress test สำหรับ thread safety
    testCase "counter is thread safe" <| fun () ->
        let counter = Counter()
        let iterations = 1000
        let tasks = 
            Array.init 10 (fun _ ->
                System.Threading.Tasks.Task.Run(fun () ->
                    for _ in 1..iterations do
                        counter.Increment()
                )
            )
        
        System.Threading.Tasks.Task.WaitAll(tasks)
        
        Expect.equal counter.Value (10 * iterations) "Counter should equal total increments"
    
    // Concurrent collection test
    testCase "concurrent bag is thread safe" <| fun () ->
        let bag = System.Collections.Concurrent.ConcurrentBag<int>()
        
        let tasks =
            Array.init 5 (fun i ->
                System.Threading.Tasks.Task.Run(fun () ->
                    for j in 1..100 do
                        bag.Add(i * 100 + j)
                )
            )
        
        System.Threading.Tasks.Task.WaitAll(tasks)
        
        Expect.equal bag.Count 500 "Should have 500 items"
    
    // Memory stress test
    testCase "handles large data without OOM" <| fun () ->
        let largeList = [1..100_000]
        let sum = List.sum largeList
        Expect.equal sum 5_000_050_000 "Sum should be correct"
    
    // Repeated operation test
    testCase "repeated operations are stable" <| fun () ->
        let mutable results = []
        
        for _ in 1..100 do
            let result = [1..1000] |> List.fold (+) 0
            results <- result :: results
        
        // All results should be the same
        let unique = results |> List.distinct
        Expect.hasLength unique 1 "All iterations should produce same result"
        Expect.equal unique.[0] 500500 "Result should be 500500"
]
```

---

## 16. โปรแกรมตัวอย่างสมบูรณ์

```fsharp
// Program.fs - Complete test suite
module Tests.Program

open Expecto

// Import all test modules
let allTests = testList "Complete Test Suite" [
    
    testList "Unit Tests" [
        Tests.BasicTests.arithmeticTests
        Tests.EqualityTests.equalityTests
        Tests.BooleanTests.booleanTests
        Tests.ThrowTests.throwTests
        Tests.ExpectTests.expectTests
    ]
    
    testList "Async Tests" [
        Tests.AsyncTests.asyncTests
    ]
    
    testList "Performance Tests" [
        Tests.PerformanceTests.performanceTests
    ]
    
    testList "Stress Tests" [
        Tests.StressTests.stressTests
    ]
]

[<EntryPoint>]
let main argv =
    runTestsWithCLIArgs [] argv allTests
```

---

## 17. สรุป (Summary)

Expecto เป็น F#-native testing framework ที่ทรงพลัง:

**Core concepts:**
- `testCase` - สร้าง single test
- `testList` - จัดกลุ่ม tests
- `testAsync` - async test
- `testSequenced` - sequential execution
- `Expect.*` - assertion functions

**ข้อดีของ Expecto:**
1. Tests เป็น values - สามารถ compose ได้
2. Parallel by default
3. Built-in logging
4. Native F# style
5. No magic attributes

**Command line options:**
```bash
dotnet run -- --filter "name"     # filter by name
dotnet run -- --sequenced         # force sequential
dotnet run -- --stress 30         # stress test 30s
dotnet run -- --list-tests        # list all tests
dotnet run -- --verbosity verbose # verbose output
```
