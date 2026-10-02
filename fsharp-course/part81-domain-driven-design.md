# Part 81 - Domain-Driven Design (DDD) กับ F#

## บทนำ (Introduction)

Domain-Driven Design (DDD) คือแนวทางการพัฒนาซอฟต์แวร์ที่เน้นการสร้างโมเดลจาก domain ของธุรกิจเป็นหลัก F# เป็นภาษาที่เหมาะสมอย่างยิ่งสำหรับ DDD เพราะ:

- **Algebraic Data Types** ทำให้สร้าง domain model ได้แม่นยำ
- **Immutability by default** ทำให้ value objects เป็นธรรมชาติ
- **Pattern matching** ทำให้จัดการ domain logic ได้ชัดเจน
- **Discriminated Unions** ทำให้ illegal states เป็นไปไม่ได้

## 1. DDD Core Concepts

### 1.1 Ubiquitous Language (ภาษากลาง)

```fsharp
// ภาษากลางคือภาษาที่ developer และ domain expert ใช้ร่วมกัน
// ทุก concept ใน code ควรสะท้อน business vocabulary

// ❌ ไม่ดี - ใช้ technical jargon
type UserRecord = {
    UserId: int
    DataField1: string
    DataField2: string
    Flag1: bool
}

// ✅ ดี - ใช้ ubiquitous language
type Customer = {
    CustomerId: CustomerId
    FullName: PersonName
    EmailAddress: EmailAddress
    IsVerified: bool
}
```

### 1.2 Bounded Contexts (ขอบเขตของ Context)

```fsharp
// แต่ละ bounded context มี model ของตัวเอง
// ชื่อเดียวกันอาจมีความหมายต่างกันใน context ต่างกัน

// ===== Sales Context =====
module SalesContext =
    type Customer = {
        CustomerId: string
        Name: string
        CreditLimit: decimal
    }
    
    type Order = {
        OrderId: string
        CustomerId: string
        Items: OrderItem list
        TotalAmount: decimal
    }
    
    and OrderItem = {
        ProductId: string
        Quantity: int
        UnitPrice: decimal
    }

// ===== Shipping Context =====
module ShippingContext =
    type Customer = {
        CustomerId: string
        DeliveryAddress: Address
        ContactPhone: string
    }
    
    type Shipment = {
        ShipmentId: string
        CustomerId: string
        Items: ShipmentItem list
        Status: ShipmentStatus
    }
    
    and ShipmentItem = {
        ProductId: string
        Quantity: int
        Weight: decimal
    }
    
    and ShipmentStatus =
        | Pending
        | Processing
        | Shipped
        | Delivered
        | Returned
    
    and Address = {
        Street: string
        City: string
        Country: string
        PostalCode: string
    }
```

## 2. Building Blocks ของ DDD

### 2.1 Value Objects

Value objects คือ objects ที่ไม่มี identity - ค่าเหมือนกันหมายความว่าเป็นสิ่งเดียวกัน

```fsharp
// ===== Email Value Object =====
type EmailAddress = private EmailAddress of string

module EmailAddress =
    let create (email: string) =
        if System.String.IsNullOrWhiteSpace(email) then
            Error "Email cannot be empty"
        elif not (email.Contains("@")) then
            Error "Invalid email format"
        elif email.Length > 254 then
            Error "Email too long"
        else
            Ok (EmailAddress (email.ToLower().Trim()))
    
    let value (EmailAddress email) = email
    
    let toString (EmailAddress email) = email

// ===== Money Value Object =====
type Currency = USD | THB | EUR | JPY

type Money = private {
    Amount: decimal
    Currency: Currency
}

module Money =
    let create amount currency =
        if amount < 0m then
            Error "Amount cannot be negative"
        else
            Ok { Amount = amount; Currency = currency }
    
    let zero currency = { Amount = 0m; Currency = currency }
    
    let add m1 m2 =
        if m1.Currency <> m2.Currency then
            Error "Cannot add different currencies"
        else
            Ok { Amount = m1.Amount + m2.Amount; Currency = m1.Currency }
    
    let multiply (money: Money) (factor: decimal) =
        if factor < 0m then
            Error "Factor cannot be negative"
        else
            Ok { money with Amount = money.Amount * factor }
    
    let value (m: Money) = m.Amount
    let currency (m: Money) = m.Currency
    
    let format (m: Money) =
        match m.Currency with
        | USD -> sprintf "$%.2f" m.Amount
        | THB -> sprintf "฿%.2f" m.Amount
        | EUR -> sprintf "€%.2f" m.Amount
        | JPY -> sprintf "¥%.0f" m.Amount

// ===== PersonName Value Object =====
type PersonName = private {
    FirstName: string
    LastName: string
}

module PersonName =
    let create firstName lastName =
        let validateName name fieldName =
            if System.String.IsNullOrWhiteSpace(name) then
                Error (sprintf "%s cannot be empty" fieldName)
            elif name.Length > 100 then
                Error (sprintf "%s too long" fieldName)
            else
                Ok (name.Trim())
        
        match validateName firstName "FirstName", validateName lastName "LastName" with
        | Ok fn, Ok ln -> Ok { FirstName = fn; LastName = ln }
        | Error e, _ -> Error e
        | _, Error e -> Error e
    
    let fullName name = sprintf "%s %s" name.FirstName name.LastName
    let firstName name = name.FirstName
    let lastName name = name.LastName

// ===== Quantity Value Object =====
type Quantity = private Quantity of int

module Quantity =
    let create qty =
        if qty <= 0 then
            Error "Quantity must be positive"
        else
            Ok (Quantity qty)
    
    let value (Quantity qty) = qty
    
    let add (Quantity q1) (Quantity q2) = Quantity (q1 + q2)
    
    let subtract (Quantity q1) (Quantity q2) =
        let result = q1 - q2
        if result < 0 then
            Error "Insufficient quantity"
        else
            Ok (Quantity result)

// ===== Address Value Object =====
type Address = {
    Street: string
    City: string
    State: string
    Country: string
    PostalCode: string
}

module Address =
    let create street city state country postalCode =
        let validate field fieldName maxLen =
            if System.String.IsNullOrWhiteSpace(field) then
                Error (sprintf "%s cannot be empty" fieldName)
            elif field.Length > maxLen then
                Error (sprintf "%s too long (max %d)" fieldName maxLen)
            else
                Ok (field.Trim())
        
        match validate street "Street" 200,
              validate city "City" 100,
              validate country "Country" 100 with
        | Ok s, Ok c, Ok co ->
            Ok {
                Street = s
                City = c
                State = state.Trim()
                Country = co
                PostalCode = postalCode.Trim()
            }
        | Error e, _, _ -> Error e
        | _, Error e, _ -> Error e
        | _, _, Error e -> Error e
    
    let format (addr: Address) =
        sprintf "%s, %s, %s, %s %s" 
            addr.Street addr.City addr.State addr.Country addr.PostalCode
```

### 2.2 Entities (นิติบุคคลที่มี Identity)

```fsharp
// Entities มี identity ที่ unique - สิ่งสำคัญคือ ID ไม่ใช่ค่าของ fields

// ===== Product Id =====
type ProductId = ProductId of System.Guid

module ProductId =
    let create () = ProductId (System.Guid.NewGuid())
    let fromString (s: string) =
        match System.Guid.TryParse(s) with
        | true, guid -> Ok (ProductId guid)
        | _ -> Error "Invalid product ID"
    let value (ProductId id) = id
    let toString (ProductId id) = id.ToString()

// ===== Product Entity =====
type ProductCategory =
    | Electronics
    | Clothing
    | Books
    | Food
    | Other of string

type ProductStatus =
    | Active
    | Discontinued
    | OutOfStock

type Product = {
    Id: ProductId
    Name: string
    Description: string
    Price: Money
    Category: ProductCategory
    StockQuantity: int
    Status: ProductStatus
    CreatedAt: System.DateTime
    UpdatedAt: System.DateTime
}

module Product =
    let create name description price category initialStock =
        if System.String.IsNullOrWhiteSpace(name) then
            Error "Product name cannot be empty"
        elif initialStock < 0 then
            Error "Initial stock cannot be negative"
        else
            let now = System.DateTime.UtcNow
            Ok {
                Id = ProductId.create()
                Name = name.Trim()
                Description = description
                Price = price
                Category = category
                StockQuantity = initialStock
                Status = Active
                CreatedAt = now
                UpdatedAt = now
            }
    
    let updatePrice product newPrice =
        { product with Price = newPrice; UpdatedAt = System.DateTime.UtcNow }
    
    let addStock product quantity =
        if quantity <= 0 then
            Error "Quantity must be positive"
        else
            Ok { product with 
                    StockQuantity = product.StockQuantity + quantity
                    UpdatedAt = System.DateTime.UtcNow }
    
    let removeStock product quantity =
        if quantity <= 0 then
            Error "Quantity must be positive"
        elif product.StockQuantity < quantity then
            Error (sprintf "Insufficient stock. Available: %d, Requested: %d" 
                           product.StockQuantity quantity)
        else
            let newQty = product.StockQuantity - quantity
            let newStatus = if newQty = 0 then OutOfStock else product.Status
            Ok { product with 
                    StockQuantity = newQty
                    Status = newStatus
                    UpdatedAt = System.DateTime.UtcNow }
    
    let discontinue product =
        { product with Status = Discontinued; UpdatedAt = System.DateTime.UtcNow }

// ===== Customer Entity =====
type CustomerId = CustomerId of System.Guid

module CustomerId =
    let create () = CustomerId (System.Guid.NewGuid())
    let value (CustomerId id) = id

type CustomerTier = Bronze | Silver | Gold | Platinum

type Customer = {
    Id: CustomerId
    Name: PersonName
    Email: EmailAddress
    ShippingAddresses: Address list
    DefaultAddressIndex: int option
    Tier: CustomerTier
    TotalPurchases: Money
    CreatedAt: System.DateTime
}

module Customer =
    let create name email =
        let now = System.DateTime.UtcNow
        {
            Id = CustomerId.create()
            Name = name
            Email = email
            ShippingAddresses = []
            DefaultAddressIndex = None
            Tier = Bronze
            TotalPurchases = Money.zero USD
            CreatedAt = now
        }
    
    let addAddress customer address =
        let newAddresses = customer.ShippingAddresses @ [address]
        let defaultIndex = 
            match customer.DefaultAddressIndex with
            | None -> Some 0
            | some -> some
        { customer with 
            ShippingAddresses = newAddresses
            DefaultAddressIndex = defaultIndex }
    
    let getDefaultAddress customer =
        match customer.DefaultAddressIndex with
        | None -> None
        | Some idx ->
            if idx < customer.ShippingAddresses.Length then
                Some customer.ShippingAddresses.[idx]
            else
                None
    
    let updateTier customer =
        let amount = Money.value customer.TotalPurchases
        let tier =
            if amount >= 100000m then Platinum
            elif amount >= 50000m then Gold
            elif amount >= 10000m then Silver
            else Bronze
        { customer with Tier = tier }
    
    let recordPurchase customer amount =
        match Money.add customer.TotalPurchases amount with
        | Ok newTotal ->
            let updated = { customer with TotalPurchases = newTotal }
            Ok (updateTier updated)
        | Error e -> Error e
```

### 2.3 Aggregates (กลุ่ม Entities และ Value Objects)

```fsharp
// Aggregate คือกลุ่มของ entities และ value objects ที่มี consistency boundary
// Aggregate Root เป็น entry point เดียวสำหรับการเข้าถึง aggregate

// ===== Order Aggregate =====
type OrderId = OrderId of System.Guid

module OrderId =
    let create () = OrderId (System.Guid.NewGuid())
    let value (OrderId id) = id
    let toString (OrderId id) = id.ToString()

type OrderLineId = OrderLineId of System.Guid

type OrderLine = {
    Id: OrderLineId
    ProductId: ProductId
    ProductName: string
    Quantity: Quantity
    UnitPrice: Money
    Discount: decimal // percentage 0-100
}

module OrderLine =
    let create productId productName quantity unitPrice discount =
        if discount < 0m || discount > 100m then
            Error "Discount must be between 0 and 100"
        else
            Ok {
                Id = OrderLineId (System.Guid.NewGuid())
                ProductId = productId
                ProductName = productName
                Quantity = quantity
                UnitPrice = unitPrice
                Discount = discount
            }
    
    let subtotal (line: OrderLine) =
        let qty = decimal (Quantity.value line.Quantity)
        let price = Money.value line.UnitPrice
        let discountFactor = 1m - (line.Discount / 100m)
        { Amount = qty * price * discountFactor
          Currency = Money.currency line.UnitPrice }
        // Note: Using internal structure for simplicity

type OrderStatus =
    | Draft
    | Confirmed
    | Processing
    | Shipped
    | Delivered
    | Cancelled
    | Refunded

type PaymentMethod =
    | CreditCard of last4: string
    | BankTransfer of bankCode: string
    | CashOnDelivery
    | DigitalWallet of walletType: string

type PaymentStatus =
    | Pending
    | Paid
    | Failed
    | Refunded

// Domain Events ที่ Order สร้าง
type OrderDomainEvent =
    | OrderCreated of orderId: OrderId * customerId: CustomerId
    | ItemAdded of orderId: OrderId * productId: ProductId * quantity: int
    | ItemRemoved of orderId: OrderId * productId: ProductId
    | OrderConfirmed of orderId: OrderId * totalAmount: decimal
    | OrderCancelled of orderId: OrderId * reason: string
    | OrderShipped of orderId: OrderId * trackingNumber: string
    | PaymentReceived of orderId: OrderId * amount: decimal

// Order Aggregate Root
type Order = {
    Id: OrderId
    CustomerId: CustomerId
    Lines: OrderLine list
    Status: OrderStatus
    PaymentMethod: PaymentMethod option
    PaymentStatus: PaymentStatus
    ShippingAddress: Address option
    Notes: string
    CreatedAt: System.DateTime
    UpdatedAt: System.DateTime
    DomainEvents: OrderDomainEvent list
}

module Order =
    let create customerId =
        let now = System.DateTime.UtcNow
        let orderId = OrderId.create()
        let order = {
            Id = orderId
            CustomerId = customerId
            Lines = []
            Status = Draft
            PaymentMethod = None
            PaymentStatus = Pending
            ShippingAddress = None
            Notes = ""
            CreatedAt = now
            UpdatedAt = now
            DomainEvents = [OrderCreated (orderId, customerId)]
        }
        order
    
    let addItem order productId productName quantity unitPrice discount =
        match order.Status with
        | Draft ->
            match OrderLine.create productId productName quantity unitPrice discount with
            | Error e -> Error e
            | Ok line ->
                let newOrder = {
                    order with
                        Lines = order.Lines @ [line]
                        UpdatedAt = System.DateTime.UtcNow
                        DomainEvents = 
                            order.DomainEvents @ 
                            [ItemAdded (order.Id, productId, Quantity.value quantity)]
                }
                Ok newOrder
        | _ -> Error (sprintf "Cannot add items to order in %A status" order.Status)
    
    let removeItem order productId =
        match order.Status with
        | Draft ->
            let newLines = order.Lines |> List.filter (fun l -> l.ProductId <> productId)
            if newLines.Length = order.Lines.Length then
                Error "Item not found in order"
            else
                Ok { order with 
                        Lines = newLines
                        UpdatedAt = System.DateTime.UtcNow
                        DomainEvents = order.DomainEvents @ [ItemRemoved (order.Id, productId)] }
        | _ -> Error "Cannot remove items from confirmed order"
    
    let calculateTotal order =
        order.Lines
        |> List.sumBy (fun line ->
            let qty = decimal (Quantity.value line.Quantity)
            let price = Money.value line.UnitPrice
            let discountFactor = 1m - (line.Discount / 100m)
            qty * price * discountFactor)
    
    let confirm order shippingAddress paymentMethod =
        match order.Status with
        | Draft ->
            if order.Lines.IsEmpty then
                Error "Cannot confirm empty order"
            else
                let total = calculateTotal order
                let confirmed = {
                    order with
                        Status = Confirmed
                        ShippingAddress = Some shippingAddress
                        PaymentMethod = Some paymentMethod
                        UpdatedAt = System.DateTime.UtcNow
                        DomainEvents = 
                            order.DomainEvents @ [OrderConfirmed (order.Id, total)]
                }
                Ok confirmed
        | _ -> Error "Order is not in draft status"
    
    let cancel order reason =
        match order.Status with
        | Draft | Confirmed | Processing ->
            Ok { order with
                    Status = Cancelled
                    UpdatedAt = System.DateTime.UtcNow
                    DomainEvents = order.DomainEvents @ [OrderCancelled (order.Id, reason)] }
        | Shipped | Delivered ->
            Error "Cannot cancel shipped or delivered order"
        | Cancelled -> Error "Order is already cancelled"
        | Refunded -> Error "Order is already refunded"
    
    let ship order trackingNumber =
        match order.Status with
        | Processing ->
            if System.String.IsNullOrWhiteSpace(trackingNumber) then
                Error "Tracking number is required"
            else
                Ok { order with
                        Status = Shipped
                        UpdatedAt = System.DateTime.UtcNow
                        DomainEvents = order.DomainEvents @ [OrderShipped (order.Id, trackingNumber)] }
        | _ -> Error (sprintf "Cannot ship order in %A status" order.Status)
    
    let markPaymentReceived order amount =
        Ok { order with
                PaymentStatus = PaymentStatus.Paid
                UpdatedAt = System.DateTime.UtcNow
                DomainEvents = order.DomainEvents @ [PaymentReceived (order.Id, amount)] }
    
    let clearEvents order =
        { order with DomainEvents = [] }
```

## 3. Domain Events

```fsharp
// Domain Events แทน significant things ที่เกิดขึ้นใน domain
// Events ใช้ past tense เสมอ

open System

type DomainEventId = DomainEventId of Guid

type DomainEventEnvelope<'T> = {
    EventId: DomainEventId
    EventType: string
    OccurredAt: DateTime
    Payload: 'T
    CorrelationId: string option
}

module DomainEventEnvelope =
    let create payload eventType =
        {
            EventId = DomainEventId (Guid.NewGuid())
            EventType = eventType
            OccurredAt = DateTime.UtcNow
            Payload = payload
            CorrelationId = None
        }
    
    let withCorrelation correlationId envelope =
        { envelope with CorrelationId = Some correlationId }

// E-commerce Domain Events
type CustomerRegistered = {
    CustomerId: string
    Email: string
    Name: string
    RegistrationDate: DateTime
}

type CustomerVerified = {
    CustomerId: string
    VerificationDate: DateTime
}

type OrderPlaced = {
    OrderId: string
    CustomerId: string
    Items: {| ProductId: string; Quantity: int; Price: decimal |} list
    TotalAmount: decimal
    PlacedAt: DateTime
}

type OrderPaid = {
    OrderId: string
    Amount: decimal
    PaymentMethod: string
    PaidAt: DateTime
}

type OrderFulfilled = {
    OrderId: string
    TrackingNumber: string
    FulfilledAt: DateTime
}

type ProductRestocked = {
    ProductId: string
    AddedQuantity: int
    NewTotalStock: int
    RestockedAt: DateTime
}

// Event Bus (interface pattern)
type IEventBus =
    abstract member Publish: 'T DomainEventEnvelope -> Async<unit>
    abstract member Subscribe: string -> ('T DomainEventEnvelope -> Async<unit>) -> unit

// In-memory Event Bus implementation (for testing)
type InMemoryEventBus() =
    let handlers = System.Collections.Generic.Dictionary<string, obj list>()
    let lockObj = obj()
    
    interface IEventBus with
        member _.Publish<'T>(envelope: 'T DomainEventEnvelope) =
            async {
                let key = typeof<'T>.Name
                let handlerList =
                    lock lockObj (fun () ->
                        if handlers.ContainsKey(key) then handlers.[key]
                        else [])
                for handler in handlerList do
                    let typedHandler = handler :?> ('T DomainEventEnvelope -> Async<unit>)
                    do! typedHandler envelope
            }
        
        member _.Subscribe<'T>(eventType: string) (handler: 'T DomainEventEnvelope -> Async<unit>) =
            lock lockObj (fun () ->
                let key = typeof<'T>.Name
                let existing = 
                    if handlers.ContainsKey(key) then handlers.[key]
                    else []
                handlers.[key] <- existing @ [handler :> obj])
```

## 4. Repositories (Interface ใน Domain)

```fsharp
// Repository interface อยู่ใน domain layer
// Implementation อยู่ใน infrastructure layer

// Repository interfaces
type IProductRepository =
    abstract member FindById: ProductId -> Async<Product option>
    abstract member FindByCategory: ProductCategory -> Async<Product list>
    abstract member FindActive: unit -> Async<Product list>
    abstract member Save: Product -> Async<unit>
    abstract member Delete: ProductId -> Async<bool>

type IOrderRepository =
    abstract member FindById: OrderId -> Async<Order option>
    abstract member FindByCustomer: CustomerId -> Async<Order list>
    abstract member FindByStatus: OrderStatus -> Async<Order list>
    abstract member Save: Order -> Async<unit>
    abstract member NextOrderNumber: unit -> Async<int>

type ICustomerRepository =
    abstract member FindById: CustomerId -> Async<Customer option>
    abstract member FindByEmail: EmailAddress -> Async<Customer option>
    abstract member Save: Customer -> Async<unit>
    abstract member Exists: EmailAddress -> Async<bool>

// Unit of Work pattern
type IUnitOfWork =
    abstract member Products: IProductRepository
    abstract member Orders: IOrderRepository
    abstract member Customers: ICustomerRepository
    abstract member CommitAsync: unit -> Async<unit>
    abstract member RollbackAsync: unit -> Async<unit>
```

## 5. Domain Services

```fsharp
// Domain services ประกอบ domain logic ที่ไม่ fit กับ single entity/aggregate

type OrderingDomainService(productRepo: IProductRepository) =
    
    // ตรวจสอบว่า products มีใน stock ก่อน place order
    member _.ValidateOrderItems (items: (ProductId * int) list) =
        async {
            let! validationResults =
                items
                |> List.map (fun (productId, qty) ->
                    async {
                        let! product = productRepo.FindById productId
                        match product with
                        | None -> 
                            return Error (sprintf "Product %s not found" (ProductId.toString productId))
                        | Some p ->
                            if p.Status <> Active then
                                return Error (sprintf "Product %s is not available" p.Name)
                            elif p.StockQuantity < qty then
                                return Error (sprintf "Insufficient stock for %s. Available: %d, Requested: %d" 
                                                       p.Name p.StockQuantity qty)
                            else
                                return Ok (p, qty)
                    })
                |> Async.Parallel
            
            let errors = validationResults |> Array.choose (function Error e -> Some e | _ -> None)
            if errors.Length > 0 then
                return Error (errors |> Array.toList)
            else
                let products = validationResults |> Array.choose (function Ok p -> Some p | _ -> None)
                return Ok (products |> Array.toList)
        }

// Discount calculation service
type DiscountDomainService() =
    
    member _.CalculateDiscount (customer: Customer) (orderTotal: decimal) =
        let tierDiscount =
            match customer.Tier with
            | Bronze -> 0m
            | Silver -> 5m
            | Gold -> 10m
            | Platinum -> 15m
        
        let volumeDiscount =
            if orderTotal >= 10000m then 5m
            elif orderTotal >= 5000m then 3m
            elif orderTotal >= 1000m then 1m
            else 0m
        
        // ไม่ให้ discount รวมเกิน 20%
        min 20m (tierDiscount + volumeDiscount)

// Pricing service
type PricingDomainService() =
    
    member _.CalculateFinalPrice (basePrice: decimal) (discountPercent: decimal) (taxRate: decimal) =
        let discountedPrice = basePrice * (1m - discountPercent / 100m)
        let tax = discountedPrice * taxRate / 100m
        {|
            BasePrice = basePrice
            Discount = basePrice - discountedPrice
            TaxableAmount = discountedPrice
            Tax = tax
            Total = discountedPrice + tax
        |}
```

## 6. Application Services

```fsharp
// Application services orchestrate domain objects
// ไม่มี business logic เอง - delegate ไปให้ domain

// DTOs (Data Transfer Objects)
type PlaceOrderCommand = {
    CustomerId: string
    Items: {| ProductId: string; Quantity: int |} list
    ShippingAddress: {| Street: string; City: string; State: string; Country: string; PostalCode: string |}
    PaymentMethod: string
    Notes: string
}

type OrderResult = {
    OrderId: string
    TotalAmount: decimal
    Status: string
    Message: string
}

type RegisterCustomerCommand = {
    FirstName: string
    LastName: string
    Email: string
}

type CustomerResult = {
    CustomerId: string
    FullName: string
    Email: string
    Message: string
}

// Application Service
type OrderApplicationService(
    uow: IUnitOfWork,
    orderingService: OrderingDomainService,
    discountService: DiscountDomainService,
    eventBus: IEventBus) =
    
    member _.PlaceOrder (command: PlaceOrderCommand) =
        async {
            // 1. Parse inputs
            let! customerResult = uow.Customers.FindById (CustomerId (System.Guid.Parse(command.CustomerId)))
            match customerResult with
            | None -> return Error "Customer not found"
            | Some customer ->
            
            // 2. Validate product availability
            let productIds = 
                command.Items 
                |> List.map (fun i -> ProductId (System.Guid.Parse(i.ProductId)), i.Quantity)
            
            let! validationResult = orderingService.ValidateOrderItems productIds
            match validationResult with
            | Error errors -> return Error (String.concat ", " errors)
            | Ok products ->
            
            // 3. Create order
            let customerId = customer.Id
            let mutable order = Order.create customerId
            
            // 4. Add items
            let mutable addError = None
            for (product, qty) in products do
                if addError.IsNone then
                    match Quantity.create qty with
                    | Error e -> addError <- Some e
                    | Ok quantity ->
                        let discount = discountService.CalculateDiscount customer (Order.calculateTotal order)
                        match Order.addItem order product.Id product.Name quantity product.Price discount with
                        | Error e -> addError <- Some e
                        | Ok updatedOrder -> order <- updatedOrder
            
            match addError with
            | Some e -> return Error e
            | None ->
            
            // 5. Build shipping address
            let addr = command.ShippingAddress
            match Address.create addr.Street addr.City addr.State addr.Country addr.PostalCode with
            | Error e -> return Error e
            | Ok address ->
            
            // 6. Determine payment method
            let paymentMethod =
                match command.PaymentMethod.ToLower() with
                | "cod" -> CashOnDelivery
                | "bank" -> BankTransfer "KBANK"
                | method -> DigitalWallet method
            
            // 7. Confirm order
            match Order.confirm order address paymentMethod with
            | Error e -> return Error e
            | Ok confirmedOrder ->
            
            // 8. Save order
            do! uow.Orders.Save confirmedOrder
            do! uow.CommitAsync()
            
            // 9. Publish events
            let total = Order.calculateTotal confirmedOrder
            let event = DomainEventEnvelope.create
                            { OrderId = OrderId.toString confirmedOrder.Id
                              CustomerId = command.CustomerId
                              Items = command.Items |> List.map (fun i -> {| ProductId = i.ProductId; Quantity = i.Quantity; Price = 0m |})
                              TotalAmount = total
                              PlacedAt = System.DateTime.UtcNow }
                            "OrderPlaced"
            do! eventBus.Publish event
            
            return Ok {
                OrderId = OrderId.toString confirmedOrder.Id
                TotalAmount = total
                Status = "Confirmed"
                Message = "Order placed successfully"
            }
        }

type CustomerApplicationService(
    uow: IUnitOfWork,
    eventBus: IEventBus) =
    
    member _.RegisterCustomer (command: RegisterCustomerCommand) =
        async {
            // 1. Validate email
            match EmailAddress.create command.Email with
            | Error e -> return Error e
            | Ok email ->
            
            // 2. Check for duplicate
            let! exists = uow.Customers.Exists email
            if exists then
                return Error "Customer with this email already exists"
            else
            
            // 3. Create name
            match PersonName.create command.FirstName command.LastName with
            | Error e -> return Error e
            | Ok name ->
            
            // 4. Create customer
            let customer = Customer.create name email
            
            // 5. Save
            do! uow.Customers.Save customer
            do! uow.CommitAsync()
            
            // 6. Publish event
            let event = DomainEventEnvelope.create
                            { CustomerId = (CustomerId.value customer.Id).ToString()
                              Email = command.Email
                              Name = PersonName.fullName name
                              RegistrationDate = System.DateTime.UtcNow }
                            "CustomerRegistered"
            do! eventBus.Publish event
            
            return Ok {
                CustomerId = (CustomerId.value customer.Id).ToString()
                FullName = PersonName.fullName customer.Name
                Email = EmailAddress.toString customer.Email
                Message = "Customer registered successfully"
            }
        }
```

## 7. Making Illegal States Unrepresentable

```fsharp
// หนึ่งในข้อดีที่สำคัญที่สุดของ F# สำหรับ DDD

// ❌ ปัญหา - สามารถมี state ที่ไม่ valid ได้
type BadEmailVerification = {
    Email: string
    IsVerified: bool
    VerificationCode: string option  // อาจจะ None แม้ IsVerified = false
    VerifiedAt: System.DateTime option
}

// ✅ ดี - illegal states เป็นไปไม่ได้
type EmailVerificationStatus =
    | Unverified of code: string * expiresAt: System.DateTime
    | Verified of verifiedAt: System.DateTime
    | Expired of expiredAt: System.DateTime

type EmailVerification = {
    Email: EmailAddress
    Status: EmailVerificationStatus
}

// ===== Payment State Machine =====
// ❌ ปัญหา
type BadPayment = {
    Amount: decimal
    IsPaid: bool
    PaidAt: System.DateTime option  // อาจจะ Some แม้ IsPaid = false
    IsRefunded: bool
    RefundedAt: System.DateTime option
    RefundAmount: decimal option
}

// ✅ ดี - State machine ที่ชัดเจน
type PaymentState =
    | AwaitingPayment of dueDate: System.DateTime
    | PaymentReceived of {| Amount: decimal; ReceivedAt: System.DateTime; TransactionId: string |}
    | PaymentFailed of {| FailureReason: string; FailedAt: System.DateTime |}
    | PaymentRefunded of {| OriginalAmount: decimal; RefundedAmount: decimal; RefundedAt: System.DateTime |}

type Payment = {
    Id: System.Guid
    OrderId: OrderId
    State: PaymentState
}

module Payment =
    let create orderId dueDate = {
        Id = System.Guid.NewGuid()
        OrderId = orderId
        State = AwaitingPayment dueDate
    }
    
    let receive payment amount transactionId =
        match payment.State with
        | AwaitingPayment _ ->
            Ok { payment with
                    State = PaymentReceived {| 
                        Amount = amount
                        ReceivedAt = System.DateTime.UtcNow
                        TransactionId = transactionId |} }
        | PaymentReceived _ -> Error "Payment already received"
        | PaymentFailed _ -> Error "Payment has failed, cannot receive"
        | PaymentRefunded _ -> Error "Payment already refunded"
    
    let refund payment refundAmount =
        match payment.State with
        | PaymentReceived p when refundAmount <= p.Amount ->
            Ok { payment with
                    State = PaymentRefunded {|
                        OriginalAmount = p.Amount
                        RefundedAmount = refundAmount
                        RefundedAt = System.DateTime.UtcNow |} }
        | PaymentReceived p ->
            Error (sprintf "Refund amount %.2f exceeds payment amount %.2f" refundAmount p.Amount)
        | _ -> Error "Can only refund a received payment"

// ===== Contact Information =====
// Ensuring at least one contact method exists
type ContactInfo =
    | EmailOnly of EmailAddress
    | PhoneOnly of phone: string
    | Both of email: EmailAddress * phone: string

module ContactInfo =
    let email = function
        | EmailOnly e -> Some e
        | Both (e, _) -> Some e
        | PhoneOnly _ -> None
    
    let phone = function
        | PhoneOnly p -> Some p
        | Both (_, p) -> Some p
        | EmailOnly _ -> None
    
    let addEmail (ci: ContactInfo) email =
        match ci with
        | PhoneOnly p -> Both (email, p)
        | EmailOnly _ -> EmailOnly email
        | Both (_, p) -> Both (email, p)

// ===== Non-empty list =====
type NonEmptyList<'T> = {
    Head: 'T
    Tail: 'T list
}

module NonEmptyList =
    let create head tail = { Head = head; Tail = tail }
    let singleton item = { Head = item; Tail = [] }
    let toList nel = nel.Head :: nel.Tail
    let length nel = 1 + nel.Tail.Length
    let map f nel = { Head = f nel.Head; Tail = List.map f nel.Tail }
    let fromList = function
        | [] -> None
        | head :: tail -> Some { Head = head; Tail = tail }
```

## 8. Smart Constructors

```fsharp
// Smart constructors ทำให้ invalid data ไม่สามารถสร้างได้

// ===== Validated Types =====
type ProductCode = private ProductCode of string

module ProductCode =
    let create (code: string) =
        if System.String.IsNullOrWhiteSpace(code) then
            Error "Product code cannot be empty"
        elif code.Length < 3 || code.Length > 20 then
            Error "Product code must be 3-20 characters"
        elif not (System.Text.RegularExpressions.Regex.IsMatch(code, @"^[A-Z0-9\-]+$")) then
            Error "Product code can only contain uppercase letters, numbers, and hyphens"
        else
            Ok (ProductCode (code.Trim().ToUpper()))
    
    let value (ProductCode code) = code

type PhoneNumber = private PhoneNumber of string

module PhoneNumber =
    let private thaiPattern = System.Text.RegularExpressions.Regex(@"^(0[0-9]{8,9}|[+]66[0-9]{8,9})$")
    
    let create (phone: string) =
        let cleaned = System.Text.RegularExpressions.Regex.Replace(phone, @"[\s\-\(\)]", "")
        if System.String.IsNullOrWhiteSpace(cleaned) then
            Error "Phone number cannot be empty"
        elif not (thaiPattern.IsMatch(cleaned)) then
            Error "Invalid Thai phone number format"
        else
            Ok (PhoneNumber cleaned)
    
    let value (PhoneNumber phone) = phone

type Percentage = private Percentage of decimal

module Percentage =
    let create value =
        if value < 0m || value > 100m then
            Error "Percentage must be between 0 and 100"
        else
            Ok (Percentage value)
    
    let value (Percentage p) = p
    let apply (Percentage p) amount = amount * p / 100m

type PositiveInt = private PositiveInt of int

module PositiveInt =
    let create value =
        if value <= 0 then Error "Value must be positive"
        else Ok (PositiveInt value)
    
    let value (PositiveInt v) = v

type NonNegativeDecimal = private NonNegativeDecimal of decimal

module NonNegativeDecimal =
    let create value =
        if value < 0m then Error "Value cannot be negative"
        else Ok (NonNegativeDecimal value)
    
    let value (NonNegativeDecimal v) = v
    let zero = NonNegativeDecimal 0m
```

## 9. Complete E-commerce Domain Model

```fsharp
// ===== Complete Domain Model =====
module EcommerceDomain =
    
    // ===== Inventory Aggregate =====
    type StockMovementType =
        | Purchase
        | Sale
        | Return
        | Adjustment
        | Damaged
    
    type StockMovement = {
        Id: System.Guid
        ProductId: ProductId
        Type: StockMovementType
        Quantity: int
        Reference: string
        OccurredAt: System.DateTime
    }
    
    type InventoryItem = {
        ProductId: ProductId
        AvailableQuantity: int
        ReservedQuantity: int
        ReorderPoint: int
        MaxStockLevel: int
        Movements: StockMovement list
    }
    
    module InventoryItem =
        let create productId reorderPoint maxStock = {
            ProductId = productId
            AvailableQuantity = 0
            ReservedQuantity = 0
            ReorderPoint = reorderPoint
            MaxStockLevel = maxStock
            Movements = []
        }
        
        let totalQuantity item = item.AvailableQuantity + item.ReservedQuantity
        
        let needsReorder item = item.AvailableQuantity <= item.ReorderPoint
        
        let addStock item quantity reference =
            if quantity <= 0 then Error "Quantity must be positive"
            else
                let movement = {
                    Id = System.Guid.NewGuid()
                    ProductId = item.ProductId
                    Type = Purchase
                    Quantity = quantity
                    Reference = reference
                    OccurredAt = System.DateTime.UtcNow
                }
                Ok { item with
                        AvailableQuantity = item.AvailableQuantity + quantity
                        Movements = item.Movements @ [movement] }
        
        let reserve item quantity =
            if quantity <= 0 then Error "Quantity must be positive"
            elif item.AvailableQuantity < quantity then
                Error (sprintf "Insufficient stock. Available: %d" item.AvailableQuantity)
            else
                Ok { item with
                        AvailableQuantity = item.AvailableQuantity - quantity
                        ReservedQuantity = item.ReservedQuantity + quantity }
        
        let fulfill item quantity reference =
            if item.ReservedQuantity < quantity then
                Error "Insufficient reserved quantity"
            else
                let movement = {
                    Id = System.Guid.NewGuid()
                    ProductId = item.ProductId
                    Type = Sale
                    Quantity = -quantity
                    Reference = reference
                    OccurredAt = System.DateTime.UtcNow
                }
                Ok { item with
                        ReservedQuantity = item.ReservedQuantity - quantity
                        Movements = item.Movements @ [movement] }
    
    // ===== Catalog Aggregate =====
    type ProductImage = {
        Url: string
        AltText: string
        IsPrimary: bool
        SortOrder: int
    }
    
    type ProductVariant = {
        Id: System.Guid
        SKU: string
        Name: string
        Price: Money
        Attributes: Map<string, string>  // e.g., {"Color": "Red", "Size": "XL"}
    }
    
    type CatalogProduct = {
        Id: ProductId
        Code: ProductCode
        Name: string
        Description: string
        ShortDescription: string
        Category: ProductCategory
        BasePrice: Money
        Images: ProductImage list
        Variants: ProductVariant list
        Tags: string list
        IsPublished: bool
        PublishedAt: System.DateTime option
        CreatedAt: System.DateTime
        UpdatedAt: System.DateTime
    }
    
    module CatalogProduct =
        let create code name description shortDesc category basePrice =
            let now = System.DateTime.UtcNow
            {
                Id = ProductId.create()
                Code = code
                Name = name
                Description = description
                ShortDescription = shortDesc
                Category = category
                BasePrice = basePrice
                Images = []
                Variants = []
                Tags = []
                IsPublished = false
                PublishedAt = None
                CreatedAt = now
                UpdatedAt = now
            }
        
        let addImage product image =
            let processedImage =
                if product.Images.IsEmpty then
                    { image with IsPrimary = true; SortOrder = 0 }
                else
                    { image with 
                        IsPrimary = false
                        SortOrder = product.Images.Length }
            { product with
                Images = product.Images @ [processedImage]
                UpdatedAt = System.DateTime.UtcNow }
        
        let publish product =
            if product.Images.IsEmpty then
                Error "Cannot publish product without images"
            elif System.String.IsNullOrWhiteSpace(product.Name) then
                Error "Product name is required"
            else
                Ok { product with
                        IsPublished = true
                        PublishedAt = Some System.DateTime.UtcNow
                        UpdatedAt = System.DateTime.UtcNow }
        
        let addVariant product variant =
            let exists = product.Variants |> List.exists (fun v -> v.SKU = variant.SKU)
            if exists then
                Error (sprintf "Variant with SKU %s already exists" variant.SKU)
            else
                Ok { product with
                        Variants = product.Variants @ [variant]
                        UpdatedAt = System.DateTime.UtcNow }
    
    // ===== Promotion Aggregate =====
    type DiscountType =
        | PercentageDiscount of Percentage
        | FixedAmountDiscount of Money
        | BuyXGetY of buyQuantity: int * getQuantity: int
    
    type PromotionCondition =
        | MinimumOrderAmount of Money
        | SpecificProducts of ProductId list
        | CustomerTierRequired of CustomerTier
        | FirstOrderOnly
        | MaxUsesPerCustomer of int
    
    type Promotion = {
        Id: System.Guid
        Code: string
        Name: string
        Description: string
        DiscountType: DiscountType
        Conditions: PromotionCondition list
        StartDate: System.DateTime
        EndDate: System.DateTime
        MaxTotalUses: int option
        CurrentUses: int
        IsActive: bool
    }
    
    module Promotion =
        let create code name description discountType startDate endDate maxUses =
            if startDate >= endDate then
                Error "End date must be after start date"
            else
                Ok {
                    Id = System.Guid.NewGuid()
                    Code = code.ToUpper().Trim()
                    Name = name
                    Description = description
                    DiscountType = discountType
                    Conditions = []
                    StartDate = startDate
                    EndDate = endDate
                    MaxTotalUses = maxUses
                    CurrentUses = 0
                    IsActive = true
                }
        
        let isValid promotion now =
            promotion.IsActive &&
            now >= promotion.StartDate &&
            now <= promotion.EndDate &&
            (match promotion.MaxTotalUses with
             | None -> true
             | Some max -> promotion.CurrentUses < max)
        
        let calculateDiscount promotion orderAmount =
            match promotion.DiscountType with
            | PercentageDiscount pct ->
                Percentage.apply pct (Money.value orderAmount)
            | FixedAmountDiscount money ->
                min (Money.value money) (Money.value orderAmount)
            | BuyXGetY _ ->
                0m  // Simplified
        
        let use' promotion =
            { promotion with CurrentUses = promotion.CurrentUses + 1 }
```

## 10. Testing Domain Model

```fsharp
// Domain model tests - ทดสอบ business rules
module DomainTests =
    
    open System
    
    let private assertOk result =
        match result with
        | Ok v -> v
        | Error e -> failwith (sprintf "Expected Ok but got Error: %s" e)
    
    let private assertError result =
        match result with
        | Error e -> e
        | Ok _ -> failwith "Expected Error but got Ok"
    
    // Test Email validation
    let testEmailValidation() =
        printfn "=== Testing Email Validation ==="
        
        let valid = EmailAddress.create "test@example.com"
        let invalid1 = EmailAddress.create "not-an-email"
        let invalid2 = EmailAddress.create ""
        
        match valid with
        | Ok email -> printfn "✓ Valid email: %s" (EmailAddress.value email)
        | Error e -> printfn "✗ Should be valid: %s" e
        
        match invalid1 with
        | Error e -> printfn "✓ Invalid email rejected: %s" e
        | Ok _ -> printfn "✗ Should be invalid"
        
        match invalid2 with
        | Error e -> printfn "✓ Empty email rejected: %s" e
        | Ok _ -> printfn "✗ Should be invalid"
    
    // Test Order workflow
    let testOrderWorkflow() =
        printfn "\n=== Testing Order Workflow ==="
        
        let customerId = CustomerId.create()
        let order = Order.create customerId
        printfn "✓ Order created: %s" (OrderId.toString order.Id)
        
        // Cannot ship draft order
        let shipResult = Order.ship order "TH123456789"
        match shipResult with
        | Error e -> printfn "✓ Cannot ship draft order: %s" e
        | Ok _ -> printfn "✗ Should not be able to ship draft order"
        
        // Add items
        let productId = ProductId.create()
        let price = { Amount = 1000m; Currency = THB }
        let qty = assertOk (Quantity.create 2)
        let orderWithItems = assertOk (Order.addItem order productId "Test Product" qty price 10m)
        printfn "✓ Item added to order"
        
        // Calculate total
        let total = Order.calculateTotal orderWithItems
        printfn "✓ Order total: %.2f" total
        
        // Confirm order
        let address = assertOk (Address.create "123 Main St" "Bangkok" "BKK" "Thailand" "10110")
        let confirmed = assertOk (Order.confirm orderWithItems address CashOnDelivery)
        printfn "✓ Order confirmed, status: %A" confirmed.Status
        
        // Cannot add items to confirmed order
        let addResult = Order.addItem confirmed productId "Another" qty price 0m
        match addResult with
        | Error e -> printfn "✓ Cannot add to confirmed order: %s" e
        | Ok _ -> printfn "✗ Should not be able to add to confirmed order"
    
    // Test Payment state machine
    let testPaymentStateMachine() =
        printfn "\n=== Testing Payment State Machine ==="
        
        let orderId = OrderId.create()
        let dueDate = DateTime.UtcNow.AddDays(7.0)
        let payment = Payment.create orderId dueDate
        
        // Receive payment
        let received = assertOk (Payment.receive payment 1000m "TXN123")
        printfn "✓ Payment received"
        
        // Cannot receive twice
        let duplicateResult = Payment.receive received 1000m "TXN456"
        match duplicateResult with
        | Error e -> printfn "✓ Duplicate payment rejected: %s" e
        | Ok _ -> printfn "✗ Should not allow duplicate payment"
        
        // Refund
        let refunded = assertOk (Payment.refund received 500m)
        printfn "✓ Partial refund processed"
        
        // Cannot refund more than paid
        let overRefund = Payment.refund received 2000m
        match overRefund with
        | Error e -> printfn "✓ Over-refund rejected: %s" e
        | Ok _ -> printfn "✗ Should reject over-refund"
    
    let runAll() =
        testEmailValidation()
        testOrderWorkflow()
        testPaymentStateMachine()
        printfn "\n✓ All domain tests passed!"

// Run tests
DomainTests.runAll()
```

## 11. สรุป DDD กับ F#

```fsharp
// สรุปข้อดีของการใช้ F# สำหรับ DDD:

(*
1. Value Objects เป็นธรรมชาติ - F# records immutable by default
2. Entities ชัดเจน - สามารถแยก identity จาก value ได้ง่าย
3. Illegal states เป็นไปไม่ได้ - Discriminated Unions ช่วยสร้าง precise types
4. Smart constructors - modules ทำให้ validation เป็นส่วนหนึ่งของ creation
5. Domain events - DUs เหมาะมากสำหรับแทน events
6. Pattern matching - ทำให้ handle cases ครบถ้วนโดย compiler บังคับ
7. Function composition - domain services compose ได้ง่าย
*)

// Key principles:
// 1. Model the domain explicitly in types
// 2. Make invalid states impossible
// 3. Use smart constructors for validation
// 4. Express business rules as types
// 5. Use domain events to communicate between aggregates

printfn "Domain-Driven Design with F# - Complete!"
printfn "F# makes domain modeling expressive and type-safe"
```

---

**สรุป**: F# เป็นภาษาที่เหมาะสมอย่างยิ่งสำหรับ DDD เพราะ type system ที่แข็งแกร่งช่วยให้เราสร้าง domain model ที่ตรงกับความเป็นจริงของธุรกิจ ในขณะที่ compiler ช่วยตรวจสอบว่า business rules ถูกปฏิบัติตามเสมอ
