# Part 77 - Test-Driven Development (TDD)

## บทนำ (Introduction)

Test-Driven Development (TDD) คือกระบวนการพัฒนาที่เขียน test ก่อน code ทำงาน ช่วยให้:
- มั่นใจว่า code ทำงานถูกต้อง
- Design code ที่ testable
- Documentation ที่มีชีวิต (living documentation)
- Refactoring ที่ปลอดภัย

---

## 1. TDD Cycle: Red-Green-Refactor

```
Red   → เขียน test ที่ fail (code ยังไม่มี)
Green → เขียน code ให้ test ผ่าน (minimal implementation)
Refactor → ทำให้ code ดีขึ้นโดย tests ยังผ่าน
```

```fsharp
// TDDCycleDemo.fs
module Tests.TDDCycleDemo

open Xunit

// === CYCLE 1: Simple Calculator ===

// RED: เขียน test ก่อน (จะ fail เพราะยังไม่มี add function)
[<Fact>]
let ``RED: add two numbers`` () =
    // ยังไม่มี Calculator module
    // Assert.Equal(5, Calculator.add 2 3)  // Compile error!
    ()

// GREEN: เขียน minimal implementation
module Calculator =
    let add x y = x + y  // Simplest possible implementation

[<Fact>]
let ``GREEN: add two numbers`` () =
    Assert.Equal(5, Calculator.add 2 3)
    Assert.Equal(0, Calculator.add 0 0)
    Assert.Equal(-5, Calculator.add -2 -3)

// REFACTOR: ปรับปรุง code (ในกรณีนี้ code เรียบร้อยอยู่แล้ว)
// Module Calculator ยังอยู่เหมือนเดิม แต่อาจ add documentation, type annotations, etc.

// === CYCLE 2: More Features ===

// RED: เพิ่ม test สำหรับ subtract
[<Fact>]
let ``RED: subtract two numbers`` () =
    // Calculator.subtract ยังไม่มี
    // Assert.Equal(2, Calculator.subtract 5 3)
    ()

// GREEN: เพิ่ม subtract
module Calculator with
    let subtract x y = x - y

[<Fact>]
let ``GREEN: subtract two numbers`` () =
    Assert.Equal(2, Calculator.subtract 5 3)
    Assert.Equal(-1, Calculator.subtract 0 1)
```

---

## 2. Writing Tests First

```fsharp
// TestFirstApproach.fs
module Tests.TestFirstApproach

open Xunit

// === สร้าง Shopping Cart โดย TDD ===

// Step 1: เขียน test สำหรับ empty cart
[<Fact>]
let ``new cart is empty`` () =
    let cart = ShoppingCart.empty
    Assert.Empty(cart.Items)
    Assert.Equal(0.0, cart.Total)

// Step 2: เขียน minimal implementation
module ShoppingCart =
    type Item = { Name: string; Price: float; Qty: int }
    type Cart = { Items: Item list }
    
    let empty = { Items = [] }
    let total cart = cart.Items |> List.sumBy (fun i -> i.Price * float i.Qty)
    
    // Extension via type augmentation
    type Cart with
        member this.Total = total this

// Step 3: เพิ่ม test สำหรับ add item
[<Fact>]
let ``add item to cart`` () =
    let cart = ShoppingCart.empty
    let item = { ShoppingCart.Name = "Widget"; ShoppingCart.Price = 9.99; ShoppingCart.Qty = 1 }
    let updatedCart = ShoppingCart.addItem item cart
    
    Assert.Equal(1, updatedCart.Items.Length)
    Assert.Contains(item, updatedCart.Items)

// Step 4: เพิ่ม addItem
module ShoppingCart with
    let addItem item cart = { cart with Items = item :: cart.Items }

// Step 5: Test total calculation
[<Fact>]
let ``total reflects all items`` () =
    let cart = 
        ShoppingCart.empty
        |> ShoppingCart.addItem { ShoppingCart.Name = "A"; ShoppingCart.Price = 10.0; ShoppingCart.Qty = 2 }
        |> ShoppingCart.addItem { ShoppingCart.Name = "B"; ShoppingCart.Price = 5.0; ShoppingCart.Qty = 3 }
    
    Assert.Equal(35.0, cart.Total, 2)  // 10*2 + 5*3 = 35

// Step 6: Test remove item
[<Fact>]
let ``remove item from cart`` () =
    let item1 = { ShoppingCart.Name = "Widget"; ShoppingCart.Price = 9.99; ShoppingCart.Qty = 1 }
    let item2 = { ShoppingCart.Name = "Gadget"; ShoppingCart.Price = 19.99; ShoppingCart.Qty = 2 }
    
    let cart = 
        ShoppingCart.empty
        |> ShoppingCart.addItem item1
        |> ShoppingCart.addItem item2
        |> ShoppingCart.removeItem "Widget"
    
    Assert.Equal(1, cart.Items.Length)
    Assert.DoesNotContain(item1, cart.Items)

// Step 7: Implement removeItem
module ShoppingCart with
    let removeItem name cart = 
        { cart with Items = cart.Items |> List.filter (fun i -> i.Name <> name) }
```

---

## 3. TDD กับ Pure Functions

```fsharp
// PureFunctionTDD.fs
module Tests.PureFunctionTDD

open Xunit

// === สร้าง String Parser โดย TDD ===

// Test 1: Parse empty string
[<Fact>]
let ``parse empty string returns empty list`` () =
    let result = CsvParser.parse ""
    Assert.Empty(result)

// Test 2: Parse single value
[<Fact>]
let ``parse single value returns list with one item`` () =
    let result = CsvParser.parse "hello"
    Assert.Equal(["hello"], result)

// Test 3: Parse comma separated values
[<Fact>]
let ``parse comma separated values`` () =
    let result = CsvParser.parse "a,b,c"
    Assert.Equal(["a"; "b"; "c"], result)

// Test 4: Handle whitespace
[<Fact>]
let ``parse trims whitespace from values`` () =
    let result = CsvParser.parse " a , b , c "
    Assert.Equal(["a"; "b"; "c"], result)

// Test 5: Handle quoted values
[<Fact>]
let ``parse handles quoted values with commas`` () =
    let result = CsvParser.parse "\"hello, world\",b,c"
    Assert.Equal(["hello, world"; "b"; "c"], result)

// Implementation (after all tests are written)
module CsvParser =
    let parse (input: string) =
        if System.String.IsNullOrWhiteSpace(input) then []
        else
            // Simple implementation that handles basic cases
            let parts = input.Split(',')
            parts |> Array.map (fun s -> s.Trim().Trim('"')) |> Array.toList

// === สร้าง Expression Evaluator โดย TDD ===

type Expr =
    | Number of float
    | Add of Expr * Expr
    | Subtract of Expr * Expr
    | Multiply of Expr * Expr
    | Divide of Expr * Expr

[<Fact>]
let ``evaluate number returns value`` () =
    let result = Evaluator.eval (Number 42.0)
    Assert.Equal(Ok 42.0, result)

[<Fact>]
let ``evaluate add returns sum`` () =
    let result = Evaluator.eval (Add(Number 3.0, Number 4.0))
    Assert.Equal(Ok 7.0, result)

[<Fact>]
let ``evaluate divide by zero returns error`` () =
    let result = Evaluator.eval (Divide(Number 10.0, Number 0.0))
    Assert.True(Result.isError result)

module Evaluator =
    let rec eval = function
        | Number n -> Ok n
        | Add (a, b) ->
            Result.map2 (+) (eval a) (eval b)
        | Subtract (a, b) ->
            Result.map2 (-) (eval a) (eval b)
        | Multiply (a, b) ->
            Result.map2 (*) (eval a) (eval b)
        | Divide (a, b) ->
            eval b |> Result.bind (fun bVal ->
                if bVal = 0.0 then Error "Division by zero"
                else eval a |> Result.map (fun aVal -> aVal / bVal)
            )
```

---

## 4. TDD สำหรับ Domain Logic

```fsharp
// DomainTDD.fs
module Tests.DomainTDD

open Xunit

// === Banking Domain โดย TDD ===

// Test 1: Open account
[<Fact>]
let ``open account with initial balance`` () =
    let account = BankAccount.open' "ACC-001" 1000.0
    Assert.Equal("ACC-001", account.Number)
    Assert.Equal(1000.0, account.Balance)
    Assert.True(account.IsActive)

// Test 2: Deposit
[<Fact>]
let ``deposit increases balance`` () =
    let account = BankAccount.open' "ACC-001" 1000.0
    match BankAccount.deposit 500.0 account with
    | Ok updated -> Assert.Equal(1500.0, updated.Balance)
    | Error msg -> Assert.Fail(msg)

// Test 3: Invalid deposit
[<Fact>]
let ``deposit negative amount fails`` () =
    let account = BankAccount.open' "ACC-001" 1000.0
    let result = BankAccount.deposit (-100.0) account
    Assert.True(Result.isError result)

// Test 4: Withdraw
[<Fact>]
let ``withdraw decreases balance`` () =
    let account = BankAccount.open' "ACC-001" 1000.0
    match BankAccount.withdraw 300.0 account with
    | Ok updated -> Assert.Equal(700.0, updated.Balance)
    | Error msg -> Assert.Fail(msg)

// Test 5: Overdraft
[<Fact>]
let ``withdraw more than balance fails`` () =
    let account = BankAccount.open' "ACC-001" 100.0
    let result = BankAccount.withdraw 200.0 account
    Assert.True(Result.isError result)

// Test 6: Close account
[<Fact>]
let ``close account sets inactive`` () =
    let account = BankAccount.open' "ACC-001" 0.0
    let closed = BankAccount.close account
    Assert.False(closed.IsActive)

// Test 7: Cannot operate on closed account
[<Fact>]
let ``cannot deposit to closed account`` () =
    let account = BankAccount.open' "ACC-001" 0.0 |> BankAccount.close
    let result = BankAccount.deposit 100.0 account
    Assert.True(Result.isError result)

// Implementation
module BankAccount =
    type Account = {
        Number: string
        Balance: float
        IsActive: bool
    }
    
    let open' number initialBalance = {
        Number = number
        Balance = initialBalance
        IsActive = true
    }
    
    let close account = { account with IsActive = false }
    
    let deposit amount account =
        if not account.IsActive then Error "Account is closed"
        elif amount <= 0.0 then Error "Deposit amount must be positive"
        else Ok { account with Balance = account.Balance + amount }
    
    let withdraw amount account =
        if not account.IsActive then Error "Account is closed"
        elif amount <= 0.0 then Error "Withdrawal amount must be positive"
        elif account.Balance < amount then Error "Insufficient funds"
        else Ok { account with Balance = account.Balance - amount }

// Test 8: Transfer between accounts
[<Fact>]
let ``transfer moves money between accounts`` () =
    let from = BankAccount.open' "ACC-001" 1000.0
    let to' = BankAccount.open' "ACC-002" 500.0
    
    match BankAccount.transfer 200.0 from to' with
    | Ok (updatedFrom, updatedTo) ->
        Assert.Equal(800.0, updatedFrom.Balance)
        Assert.Equal(700.0, updatedTo.Balance)
    | Error msg ->
        Assert.Fail(msg)

// Test 9: Transfer fails with insufficient funds
[<Fact>]
let ``transfer fails with insufficient funds`` () =
    let from = BankAccount.open' "ACC-001" 100.0
    let to' = BankAccount.open' "ACC-002" 500.0
    
    let result = BankAccount.transfer 200.0 from to'
    Assert.True(Result.isError result)

// Implementation additions
module BankAccount with
    let transfer amount fromAccount toAccount =
        fromAccount
        |> withdraw amount
        |> Result.bind (fun updatedFrom ->
            toAccount
            |> deposit amount
            |> Result.map (fun updatedTo -> (updatedFrom, updatedTo))
        )
```

---

## 5. TDD Kata - Bowling Game

```fsharp
// BowlingGameKata.fs
module Tests.BowlingGame

open Xunit

// === Bowling Game Kata ===
// Rules:
// - Game has 10 frames
// - Each frame: 1 or 2 rolls
// - Strike (10): only 1 roll, bonus = next 2 rolls
// - Spare (/): 2 rolls, bonus = next 1 roll
// - 10th frame special: can have up to 3 rolls

// Red: Write failing tests first

[<Fact>]
let ``gutter game scores zero`` () =
    let game = BowlingGame.newGame()
    let game' = [1..20] |> List.fold (fun g _ -> BowlingGame.roll 0 g) game
    Assert.Equal(0, BowlingGame.score game')

[<Fact>]
let ``all ones scores twenty`` () =
    let game = BowlingGame.newGame()
    let game' = [1..20] |> List.fold (fun g _ -> BowlingGame.roll 1 g) game
    Assert.Equal(20, BowlingGame.score game')

[<Fact>]
let ``spare followed by three scores sixteen`` () =
    let game = BowlingGame.newGame()
    let game' = 
        game 
        |> BowlingGame.roll 5  // Frame 1: spare
        |> BowlingGame.roll 5
        |> BowlingGame.roll 3  // Frame 2: 3, followed by...
        |> BowlingGame.roll 0
    let game'' = [1..16] |> List.fold (fun g _ -> BowlingGame.roll 0 g) game'
    Assert.Equal(16, BowlingGame.score game'')

[<Fact>]
let ``strike followed by three and four scores twenty four`` () =
    let game = BowlingGame.newGame()
    let game' =
        game
        |> BowlingGame.roll 10  // Strike
        |> BowlingGame.roll 3
        |> BowlingGame.roll 4
    let game'' = [1..16] |> List.fold (fun g _ -> BowlingGame.roll 0 g) game'
    Assert.Equal(24, BowlingGame.score game'')

[<Fact>]
let ``perfect game scores three hundred`` () =
    let game = BowlingGame.newGame()
    let game' = [1..12] |> List.fold (fun g _ -> BowlingGame.roll 10 g) game
    Assert.Equal(300, BowlingGame.score game')

// Green: Implement bowling game
module BowlingGame =
    type Game = { Rolls: int list }
    
    let newGame () = { Rolls = [] }
    
    let roll pins game = { Rolls = game.Rolls @ [pins] }
    
    let score game =
        let rolls = game.Rolls |> Array.ofList
        let mutable total = 0
        let mutable rollIndex = 0
        
        for frame in 1..10 do
            if rollIndex < rolls.Length then
                if rolls.[rollIndex] = 10 then  // Strike
                    total <- total + 10
                    if rollIndex + 1 < rolls.Length then
                        total <- total + rolls.[rollIndex + 1]
                    if rollIndex + 2 < rolls.Length then
                        total <- total + rolls.[rollIndex + 2]
                    rollIndex <- rollIndex + 1
                elif rollIndex + 1 < rolls.Length && 
                     rolls.[rollIndex] + rolls.[rollIndex + 1] = 10 then  // Spare
                    total <- total + 10
                    if rollIndex + 2 < rolls.Length then
                        total <- total + rolls.[rollIndex + 2]
                    rollIndex <- rollIndex + 2
                else  // Normal
                    total <- total + rolls.[rollIndex]
                    if rollIndex + 1 < rolls.Length then
                        total <- total + rolls.[rollIndex + 1]
                    rollIndex <- rollIndex + 2
        
        total
```

---

## 6. TDD Kata - String Calculator

```fsharp
// StringCalculatorKata.fs
module Tests.StringCalculator

open Xunit

// === String Calculator Kata ===
// 1. Empty string returns 0
// 2. Single number returns that number
// 3. Two numbers with comma delimiter: "1,2" -> 3
// 4. Unknown number of numbers: "1,2,3,4,5" -> 15
// 5. Allow newline as delimiter: "1\n2,3" -> 6
// 6. Custom delimiters: "//;\n1;2" -> 3
// 7. Negative numbers throw exception
// 8. Numbers > 1000 are ignored

// RED: Test 1
[<Fact>]
let ``empty string returns 0`` () =
    Assert.Equal(0, StringCalculator.add "")

// GREEN: Simplest implementation
module StringCalculator =
    let add (input: string) = 0  // Red passes, now make it work

// RED: Test 2
[<Fact>]
let ``single number returns value`` () =
    Assert.Equal(5, StringCalculator.add "5")
    Assert.Equal(42, StringCalculator.add "42")

// GREEN: Update implementation
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        else int input

// RED: Test 3
[<Fact>]
let ``two numbers with comma`` () =
    Assert.Equal(3, StringCalculator.add "1,2")
    Assert.Equal(10, StringCalculator.add "4,6")

// GREEN: Handle comma delimiter
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        else
            input.Split(',')
            |> Array.sumBy int

// RED: Test 4 - multiple numbers
[<Fact>]
let ``multiple numbers with comma`` () =
    Assert.Equal(15, StringCalculator.add "1,2,3,4,5")

// Already works! No change needed

// RED: Test 5 - newline delimiter
[<Fact>]
let ``newline as delimiter`` () =
    Assert.Equal(6, StringCalculator.add "1\n2,3")

// GREEN: Handle newline
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        else
            input.Split([|','; '\n'|])
            |> Array.sumBy int

// RED: Test 6 - custom delimiter
[<Fact>]
let ``custom delimiter`` () =
    Assert.Equal(3, StringCalculator.add "//;\n1;2")
    Assert.Equal(10, StringCalculator.add "//|\n4|6")

// GREEN: Handle custom delimiter
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        elif input.StartsWith("//") then
            let delimLine = input.IndexOf('\n')
            let delimiter = input.[2..delimLine-1]
            let numbers = input.[delimLine+1..]
            numbers.Split([|delimiter|], System.StringSplitOptions.None)
            |> Array.sumBy int
        else
            input.Split([|','; '\n'|])
            |> Array.sumBy int

// RED: Test 7 - negative numbers
[<Fact>]
let ``negative numbers throw exception`` () =
    let ex = Assert.Throws<System.Exception>(fun () -> 
        StringCalculator.add "1,-2,3" |> ignore
    )
    Assert.Contains("-2", ex.Message)

// GREEN: Handle negatives
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        else
            let numbers =
                if input.StartsWith("//") then
                    let delimLine = input.IndexOf('\n')
                    let delimiter = input.[2..delimLine-1]
                    let numberStr = input.[delimLine+1..]
                    numberStr.Split([|delimiter|], System.StringSplitOptions.None)
                    |> Array.map int
                else
                    input.Split([|','; '\n'|])
                    |> Array.map int
            
            let negatives = numbers |> Array.filter (fun n -> n < 0)
            if negatives.Length > 0 then
                let negStr = negatives |> Array.map string |> String.concat ","
                raise (System.Exception(sprintf "negatives not allowed: %s" negStr))
            
            Array.sum numbers

// RED: Test 8 - ignore > 1000
[<Fact>]
let ``numbers greater than 1000 are ignored`` () =
    Assert.Equal(2, StringCalculator.add "2,1001")
    Assert.Equal(1001, StringCalculator.add "1,1000,1001")

// GREEN: Filter > 1000
module StringCalculator with
    let add (input: string) =
        if input = "" then 0
        else
            let numbers =
                if input.StartsWith("//") then
                    let delimLine = input.IndexOf('\n')
                    let delimiter = input.[2..delimLine-1]
                    let numberStr = input.[delimLine+1..]
                    numberStr.Split([|delimiter|], System.StringSplitOptions.None)
                    |> Array.map int
                else
                    input.Split([|','; '\n'|])
                    |> Array.map int
            
            let negatives = numbers |> Array.filter (fun n -> n < 0)
            if negatives.Length > 0 then
                let negStr = negatives |> Array.map string |> String.concat ","
                raise (System.Exception(sprintf "negatives not allowed: %s" negStr))
            
            numbers 
            |> Array.filter (fun n -> n <= 1000)
            |> Array.sum

// REFACTOR: ทำให้ code สะอาดขึ้น
module StringCalculatorRefactored =
    let private parseNumbers (input: string) =
        if input.StartsWith("//") then
            let delimLine = input.IndexOf('\n')
            let delimiter = input.[2..delimLine-1]
            let numberStr = input.[delimLine+1..]
            numberStr.Split([|delimiter|], System.StringSplitOptions.None)
            |> Array.map int
        else
            input.Split([|','; '\n'|])
            |> Array.map int
    
    let private validateNoNegatives numbers =
        let negatives = numbers |> Array.filter (fun n -> n < 0)
        if negatives.Length > 0 then
            let negStr = negatives |> Array.map string |> String.concat ","
            Error (sprintf "negatives not allowed: %s" negStr)
        else
            Ok numbers
    
    let private filterLarge numbers =
        numbers |> Array.filter (fun n -> n <= 1000)
    
    let add (input: string) =
        if input = "" then 0
        else
            let numbers = parseNumbers input
            match validateNoNegatives numbers with
            | Error msg -> raise (System.Exception(msg))
            | Ok valid -> valid |> filterLarge |> Array.sum
```

---

## 7. Acceptance TDD

```fsharp
// AcceptanceTDD.fs
module Tests.AcceptanceTDD

open Xunit

// Acceptance test: ทดสอบจาก perspective ของ user
// เขียนก่อน implementation ทั้งหมด

// User Story: "As a customer, I want to place an order
//             so that I can receive products"

// Acceptance Criteria:
// - Customer can add items to order
// - Order has correct total
// - Order can be submitted
// - Order confirmation is returned

[<Fact>]
let ``customer can place an order and receive confirmation`` () =
    // Given: A customer with items to order
    let customerId = 1
    let items = [
        {| ProductId = 101; Name = "Widget"; Price = 9.99; Qty = 2 |}
        {| ProductId = 102; Name = "Gadget"; Price = 24.99; Qty = 1 |}
    ]
    
    // When: Customer places order
    let orderResult = OrderService.placeOrder customerId items
    
    // Then: Order confirmation is returned with correct details
    match orderResult with
    | Ok confirmation ->
        Assert.True(confirmation.OrderId > 0, "Order ID should be positive")
        Assert.Equal(customerId, confirmation.CustomerId)
        Assert.Equal(44.97, confirmation.Total, 2)  // 2*9.99 + 1*24.99
        Assert.Equal("pending", confirmation.Status)
        Assert.True(confirmation.CreatedAt <= System.DateTime.UtcNow)
    | Error msg ->
        Assert.Fail(sprintf "Order failed: %s" msg)

// Acceptance test สำหรับ refund scenario
[<Fact>]
let ``customer can request refund for recent order`` () =
    // Given: An existing order
    let customerId = 1
    let items = [{| ProductId = 101; Name = "Widget"; Price = 9.99; Qty = 1 |}]
    let orderResult = OrderService.placeOrder customerId items
    
    let orderId = 
        match orderResult with
        | Ok c -> c.OrderId
        | Error _ -> failwith "Setup failed"
    
    // When: Customer requests refund
    let refundResult = OrderService.requestRefund orderId "Product damaged"
    
    // Then: Refund is approved
    match refundResult with
    | Ok refund ->
        Assert.Equal(orderId, refund.OrderId)
        Assert.True(refund.RefundAmount > 0.0)
        Assert.Equal("approved", refund.Status)
    | Error msg ->
        Assert.Fail(sprintf "Refund failed: %s" msg)

// Implementation to make acceptance tests pass
module OrderService =
    type OrderConfirmation = {
        OrderId: int
        CustomerId: int
        Total: float
        Status: string
        CreatedAt: System.DateTime
    }
    
    type RefundConfirmation = {
        OrderId: int
        RefundAmount: float
        Status: string
    }
    
    let mutable private nextOrderId = 1
    let mutable private orders = Map.empty<int, OrderConfirmation>
    
    let placeOrder customerId items =
        let total = items |> List.sumBy (fun i -> i.Price * float i.Qty)
        let orderId = nextOrderId
        nextOrderId <- nextOrderId + 1
        
        let confirmation = {
            OrderId = orderId
            CustomerId = customerId
            Total = total
            Status = "pending"
            CreatedAt = System.DateTime.UtcNow
        }
        
        orders <- Map.add orderId confirmation orders
        Ok confirmation
    
    let requestRefund orderId reason =
        match Map.tryFind orderId orders with
        | None -> Error (sprintf "Order %d not found" orderId)
        | Some order ->
            Ok {
                OrderId = orderId
                RefundAmount = order.Total
                Status = "approved"
            }
```

---

## 8. Outside-In TDD

```fsharp
// OutsideInTDD.fs
module Tests.OutsideInTDD

open Xunit

// Outside-In TDD (London School):
// เริ่มจาก high-level test (integration/acceptance)
// แล้วค่อย drill down ไปยัง unit tests

// Layer 1: API/Controller level (outside)
[<Fact>]
let ``POST /api/users creates user and returns 201`` () =
    // High-level acceptance test
    // Test the full flow: HTTP request -> response
    let request = {| Name = "Alice"; Email = "alice@example.com" |}
    
    // สมมติว่าเรามี API client
    let response = ApiClient.post "/api/users" request
    
    Assert.Equal(201, response.StatusCode)
    Assert.True(response.Body.Id > 0)

// Layer 2: Service level (middle)
[<Fact>]
let ``userService.createUser returns created user`` () =
    // Mock repository
    let mockRepo = {
        new IUserRepository with
            member _.Save user = Ok { user with Id = 42 }
            member _.FindById _ = None
    }
    
    let svc = UserService(mockRepo)
    let result = svc.CreateUser "Alice" "alice@example.com"
    
    match result with
    | Ok user ->
        Assert.Equal(42, user.Id)
        Assert.Equal("Alice", user.Name)
    | Error msg ->
        Assert.Fail(msg)

// Layer 3: Repository level (inside)
[<Fact>]
let ``userRepository.save stores user in database`` () =
    use ctx = createTestContext()
    let repo = DbUserRepository(ctx)
    
    let user = { Id = 0; Name = "Alice"; Email = "alice@example.com" }
    let result = repo.Save user
    
    match result with
    | Ok saved -> Assert.True(saved.Id > 0)
    | Error msg -> Assert.Fail(msg)

// Interfaces for outside-in
type IUserRepository =
    abstract member Save: {| Id: int; Name: string; Email: string |} -> Result<{| Id: int; Name: string; Email: string |}, string>
    abstract member FindById: int -> {| Id: int; Name: string; Email: string |} option

// Service implementation (grows as we add tests)
type UserService(repo: IUserRepository) =
    member _.CreateUser name email =
        if System.String.IsNullOrWhiteSpace(name) then
            Error "Name cannot be empty"
        elif not (email.Contains("@")) then
            Error "Invalid email"
        else
            repo.Save {| Id = 0; Name = name; Email = email |}

// Placeholder for context and repo
let createTestContext () = null :> System.IDisposable

type DbUserRepository(ctx: System.IDisposable) =
    interface IUserRepository with
        member _.Save user = Ok {| user with Id = 1 |}
        member _.FindById id = None

// Placeholder for API client
module ApiClient =
    let post url body = {| StatusCode = 201; Body = {| Id = 1 |} |}
```

---

## 9. Walking Skeleton

```fsharp
// WalkingSkeleton.fs
module Tests.WalkingSkeleton

open Xunit

// Walking Skeleton: skeleton ที่ทำงานได้จาก end to end
// แม้จะไม่มี feature ครบก็ตาม

// Thin slice: สร้าง minimal working system ที่ครอบคลุมทุก layer

// End-to-end acceptance test (ก่อน implement)
[<Fact>]
let ``walking skeleton: create and retrieve item`` () =
    // 1. Create an item (API call)
    let createResponse = TodoApi.createItem "Buy milk"
    
    // 2. Retrieve the item
    let getResponse = TodoApi.getItem createResponse.Id
    
    // 3. Verify
    Assert.Equal("Buy milk", getResponse.Title)
    Assert.False(getResponse.Completed)

// Minimal implementation for walking skeleton
module TodoApi =
    type TodoItem = { Id: int; Title: string; Completed: bool }
    
    let mutable private items = Map.empty<int, TodoItem>
    let mutable private nextId = 1
    
    let createItem title =
        let id = nextId
        nextId <- nextId + 1
        let item = { Id = id; Title = title; Completed = false }
        items <- Map.add id item items
        item
    
    let getItem id =
        match Map.tryFind id items with
        | Some item -> item
        | None -> failwithf "Item %d not found" id
    
    let completeItem id =
        match Map.tryFind id items with
        | Some item ->
            let updated = { item with Completed = true }
            items <- Map.add id updated items
            Ok updated
        | None ->
            Error (sprintf "Item %d not found" id)

// Growing the skeleton with more tests
[<Fact>]
let ``complete item marks it as done`` () =
    let item = TodoApi.createItem "Task to complete"
    let result = TodoApi.completeItem item.Id
    
    match result with
    | Ok completed ->
        Assert.True(completed.Completed)
    | Error msg ->
        Assert.Fail(msg)

[<Fact>]
let ``completing nonexistent item returns error`` () =
    let result = TodoApi.completeItem 99999
    Assert.True(Result.isError result)
```

---

## สรุป (Summary)

TDD ใน F# มีข้อดีพิเศษเพราะ:

1. **Pure functions** ทดสอบง่ายมาก
2. **Types** ช่วย document requirements
3. **Pattern matching** ทำให้ test สมบูรณ์ (exhaustive)
4. **Immutability** ลด side effects ที่ต้องจัดการ

```
TDD Cycle:
Red → Green → Refactor
├── เขียน test ที่ fail
├── เขียน minimal code ให้ผ่าน
└── ทำให้ code ดีขึ้น (ไม่เปลี่ยน behavior)
```

**TDD Tips สำหรับ F#:**
- เริ่ม test จาก domain types
- เขียน happy path ก่อน
- เพิ่ม edge cases ทีละ test
- Refactor เมื่อ tests ผ่านแล้ว
