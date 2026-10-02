# Part 87 - Message Queues กับ F#

## บทนำ (Introduction)

Message Queues ช่วยให้ services สื่อสารกันแบบ asynchronous โดย MassTransit เป็น library ที่ popular มากสำหรับ .NET ที่รองรับ RabbitMQ, Azure Service Bus, Amazon SQS และอื่นๆ

## 1. MassTransit Setup

```fsharp
// ===== NuGet Packages =====
// MassTransit
// MassTransit.RabbitMQ
// MassTransit.Azure.ServiceBus.Core
// MassTransit.Amazon.SQS

// ===== Message Contracts =====
// แนะนำให้แยก contracts ไว้ใน shared project

namespace Contracts

open System

// Commands (ขอให้ทำอะไรบางอย่าง - มี 1 consumer)
type SubmitOrder = {
    OrderId: string
    CustomerId: string
    Items: OrderItem list
    TotalAmount: decimal
    SubmittedAt: DateTime
}

and OrderItem = {
    ProductId: string
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

type ProcessPayment = {
    PaymentId: string
    OrderId: string
    Amount: decimal
    Currency: string
    PaymentMethod: string
    CustomerId: string
}

type ShipOrder = {
    ShipmentId: string
    OrderId: string
    ShippingAddress: Address
    Items: OrderItem list
}

and Address = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

// Events (บอกว่าอะไรเกิดขึ้นแล้ว - อาจมีหลาย consumers)
type OrderSubmitted = {
    OrderId: string
    CustomerId: string
    TotalAmount: decimal
    SubmittedAt: DateTime
}

type PaymentProcessed = {
    PaymentId: string
    OrderId: string
    Amount: decimal
    Success: bool
    FailureReason: string option
    ProcessedAt: DateTime
}

type OrderShipped = {
    OrderId: string
    ShipmentId: string
    TrackingNumber: string
    EstimatedDelivery: DateTime
    ShippedAt: DateTime
}

type OrderCancelled = {
    OrderId: string
    Reason: string
    CancelledAt: DateTime
}
```

## 2. Consumers (Message Handlers)

```fsharp
// ===== Consumers =====
namespace OrderService.Consumers

open MassTransit
open Contracts
open System

// Order Submission Consumer
type SubmitOrderConsumer(orderRepository: IOrderRepository) =
    interface IConsumer<SubmitOrder> with
        member _.Consume(context: ConsumeContext<SubmitOrder>) =
            task {
                let message = context.Message
                
                printfn "[SubmitOrderConsumer] Processing order %s" message.OrderId
                
                // Business logic
                let order = {
                    Id = message.OrderId
                    CustomerId = message.CustomerId
                    Items = message.Items
                    TotalAmount = message.TotalAmount
                    Status = "Submitted"
                    CreatedAt = message.SubmittedAt
                }
                
                // Save order
                do! orderRepository.Save order
                
                // Publish event (notify others that order was submitted)
                do! context.Publish<OrderSubmitted>({
                    OrderId = message.OrderId
                    CustomerId = message.CustomerId
                    TotalAmount = message.TotalAmount
                    SubmittedAt = message.SubmittedAt
                })
                
                printfn "[SubmitOrderConsumer] Order %s processed, event published" message.OrderId
            }

and IOrderRepository =
    abstract member Save: Order -> Async<unit>
    abstract member FindById: string -> Async<Order option>
    abstract member UpdateStatus: string -> string -> Async<unit>

and Order = {
    Id: string
    CustomerId: string
    Items: OrderItem list
    TotalAmount: decimal
    Status: string
    CreatedAt: DateTime
}

// Payment Consumer
type ProcessPaymentConsumer(paymentGateway: IPaymentGateway) =
    interface IConsumer<ProcessPayment> with
        member _.Consume(context: ConsumeContext<ProcessPayment>) =
            task {
                let message = context.Message
                
                printfn "[ProcessPaymentConsumer] Processing payment %s for order %s" 
                        message.PaymentId message.OrderId
                
                // Process payment
                let! result = paymentGateway.Charge message.Amount message.PaymentMethod
                
                let event = {
                    PaymentId = message.PaymentId
                    OrderId = message.OrderId
                    Amount = message.Amount
                    Success = Result.isOk result
                    FailureReason = match result with Error e -> Some e | Ok _ -> None
                    ProcessedAt = DateTime.UtcNow
                }
                
                do! context.Publish<PaymentProcessed>(event)
                
                printfn "[ProcessPaymentConsumer] Payment %s: %s" 
                        message.PaymentId 
                        (if event.Success then "Success" else "Failed")
            }

and IPaymentGateway =
    abstract member Charge: amount: decimal -> method: string -> Async<Result<string, string>>

// Notification Consumer (subscribes to events)
type OrderSubmittedNotificationConsumer(emailService: IEmailService) =
    interface IConsumer<OrderSubmitted> with
        member _.Consume(context: ConsumeContext<OrderSubmitted>) =
            task {
                let event = context.Message
                
                printfn "[NotificationConsumer] Sending confirmation for order %s" event.OrderId
                
                do! emailService.SendOrderConfirmation 
                        event.CustomerId 
                        event.OrderId 
                        event.TotalAmount
                
                printfn "[NotificationConsumer] Email sent for order %s" event.OrderId
            }

and IEmailService =
    abstract member SendOrderConfirmation: customerId: string -> orderId: string -> amount: decimal -> Async<unit>
```

## 3. MassTransit Configuration

```fsharp
// ===== Program.fs =====
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open MassTransit
open Contracts

let configureServices (services: IServiceCollection) =
    
    // Register consumers
    services.AddScoped<SubmitOrderConsumer>() |> ignore
    services.AddScoped<ProcessPaymentConsumer>() |> ignore
    services.AddScoped<OrderSubmittedNotificationConsumer>() |> ignore
    
    // Configure MassTransit with RabbitMQ
    services.AddMassTransit(fun x ->
        
        // Add consumers
        x.AddConsumer<SubmitOrderConsumer>()
        x.AddConsumer<ProcessPaymentConsumer>()
        x.AddConsumer<OrderSubmittedNotificationConsumer>()
        
        // Use RabbitMQ
        x.UsingRabbitMq(fun ctx cfg ->
            cfg.Host("localhost", "/", fun h ->
                h.Username("admin")
                h.Password("password"))
            
            // Configure endpoints
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<SubmitOrderConsumer>(ctx)
                
                // Retry policy
                e.UseMessageRetry(fun r ->
                    r.Intervals(
                        System.TimeSpan.FromSeconds(1.0),
                        System.TimeSpan.FromSeconds(5.0),
                        System.TimeSpan.FromSeconds(15.0)
                    ))
                
                // Dead letter queue
                e.DiscardFaultedMessages()
            )
            
            cfg.ReceiveEndpoint("payment-processing", fun e ->
                e.ConfigureConsumer<ProcessPaymentConsumer>(ctx)
                e.UseMessageRetry(fun r -> r.Immediate(3))
            )
            
            cfg.ReceiveEndpoint("order-notifications", fun e ->
                e.ConfigureConsumer<OrderSubmittedNotificationConsumer>(ctx)
            )
        )
    ) |> ignore

// Alternative: Azure Service Bus
let configureAzureServiceBus (services: IServiceCollection) (connectionString: string) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<SubmitOrderConsumer>()
        
        x.UsingAzureServiceBus(fun ctx cfg ->
            cfg.Host(connectionString)
            
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<SubmitOrderConsumer>(ctx)
            )
        )
    ) |> ignore

// Alternative: Amazon SQS
let configureAmazonSqs (services: IServiceCollection) (region: string) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<SubmitOrderConsumer>()
        
        x.UsingAmazonSqs(fun ctx cfg ->
            cfg.Host(region, fun h ->
                h.AccessKey("access-key")
                h.SecretKey("secret-key"))
            
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<SubmitOrderConsumer>(ctx)
            )
        )
    ) |> ignore
```

## 4. Publishing Messages

```fsharp
// ===== Publishing Messages =====
open MassTransit

// Using IPublishEndpoint (for events - fan-out)
type OrderApplicationService(publishEndpoint: IPublishEndpoint, sendEndpoint: ISendEndpointProvider) =
    
    // Publish event - sends to all subscribers
    member _.PublishOrderCreated (orderId: string) (customerId: string) (amount: decimal) =
        async {
            do! publishEndpoint.Publish<OrderSubmitted>({
                OrderId = orderId
                CustomerId = customerId
                TotalAmount = amount
                SubmittedAt = System.DateTime.UtcNow
            }) |> Async.AwaitTask
            
            printfn "Published OrderSubmitted event for order %s" orderId
        }
    
    // Send command - sends to specific queue (1 consumer)
    member _.SendToPaymentService (orderId: string) (amount: decimal) =
        async {
            let! endpoint = sendEndpoint.GetSendEndpoint(
                System.Uri("queue:payment-processing")) |> Async.AwaitTask
            
            do! endpoint.Send<ProcessPayment>({
                PaymentId = System.Guid.NewGuid().ToString()
                OrderId = orderId
                Amount = amount
                Currency = "THB"
                PaymentMethod = "CreditCard"
                CustomerId = "CUST-001"
            }) |> Async.AwaitTask
            
            printfn "Sent ProcessPayment command for order %s" orderId
        }

// Using IBus directly
type DirectPublisher(bus: IBus) =
    
    member _.SendOrder (order: SubmitOrder) =
        async {
            // Send to specific endpoint
            do! bus.Send<SubmitOrder>(order) |> Async.AwaitTask
        }
    
    member _.PublishEvent<'T> (event: 'T) =
        async {
            do! bus.Publish<'T>(event) |> Async.AwaitTask
        }
```

## 5. Request/Response Pattern

```fsharp
// ===== Request/Response (like RPC but async) =====

// Define request/response contracts
type GetProductInfo = {
    ProductId: string
}

type ProductInfo = {
    ProductId: string
    Name: string
    Price: decimal
    InStock: bool
}

// Consumer that responds
type GetProductInfoConsumer() =
    interface IConsumer<GetProductInfo> with
        member _.Consume(context: ConsumeContext<GetProductInfo>) =
            task {
                let productId = context.Message.ProductId
                
                // Simulate product lookup
                let product = {
                    ProductId = productId
                    Name = sprintf "Product %s" productId
                    Price = 99.99m
                    InStock = true
                }
                
                // Respond to the request
                do! context.RespondAsync<ProductInfo>(product)
                
                printfn "[GetProductInfoConsumer] Responded to request for %s" productId
            }

// Client making request
type ProductQueryService(requestClient: IRequestClient<GetProductInfo>) =
    
    member _.GetProductInfo (productId: string) =
        async {
            try
                let! response = requestClient.GetResponse<ProductInfo>({
                    ProductId = productId
                }, timeout = System.TimeSpan.FromSeconds(10.0)) |> Async.AwaitTask
                
                return Ok response.Message
            with
            | :? RequestTimeoutException ->
                return Error "Request timed out"
            | ex ->
                return Error (sprintf "Error: %s" ex.Message)
        }

// Register request client
let configureRequestClient (services: IServiceCollection) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<GetProductInfoConsumer>()
        x.AddRequestClient<GetProductInfo>()  // Register as client
        
        x.UsingRabbitMq(fun ctx cfg ->
            cfg.Host("localhost", "/", fun h ->
                h.Username("admin")
                h.Password("password"))
            cfg.ConfigureEndpoints(ctx)
        )
    ) |> ignore
```

## 6. Saga กับ MassTransit

```fsharp
// ===== State Machine Saga =====
// Coordinates multi-step business process

open MassTransit

// Saga state (stored in database)
type OrderProcessingSagaState() =
    inherit SagaStateMachineInstance()
    
    let mutable correlationId = System.Guid.Empty
    let mutable currentState = ""
    let mutable orderId = ""
    let mutable customerId = ""
    let mutable totalAmount = 0m
    let mutable paymentId = ""
    let mutable trackingNumber = ""
    
    interface ISaga with
        member _.CorrelationId with get() = correlationId and set(v) = correlationId <- v
    
    member _.CurrentState with get() = currentState and set(v) = currentState <- v
    member _.OrderId with get() = orderId and set(v) = orderId <- v
    member _.CustomerId with get() = customerId and set(v) = customerId <- v
    member _.TotalAmount with get() = totalAmount and set(v) = totalAmount <- v
    member _.PaymentId with get() = paymentId and set(v) = paymentId <- v
    member _.TrackingNumber with get() = trackingNumber and set(v) = trackingNumber <- v

// Saga events
type OrderStarted = { CorrelationId: System.Guid; OrderId: string; CustomerId: string; Amount: decimal }
type PaymentApproved = { CorrelationId: System.Guid; PaymentId: string }
type PaymentDeclined = { CorrelationId: System.Guid; Reason: string }
type ItemsShipped = { CorrelationId: System.Guid; TrackingNumber: string }
type DeliveryConfirmed = { CorrelationId: System.Guid }

// Saga State Machine
type OrderProcessingStateMachine() =
    inherit MassTransitStateMachine<OrderProcessingSagaState>()
    
    // States
    member val WaitingForPayment = Unchecked.defaultof<State> with get, set
    member val PaymentApproved' = Unchecked.defaultof<State> with get, set
    member val Shipping = Unchecked.defaultof<State> with get, set
    member val Delivered = Unchecked.defaultof<State> with get, set
    member val Failed = Unchecked.defaultof<State> with get, set
    
    // Events
    member val OrderStarted = Unchecked.defaultof<Event<OrderStarted>> with get, set
    member val PaymentApproved = Unchecked.defaultof<Event<PaymentApproved>> with get, set
    member val PaymentDeclined = Unchecked.defaultof<Event<PaymentDeclined>> with get, set
    member val ItemsShipped = Unchecked.defaultof<Event<ItemsShipped>> with get, set
    member val DeliveryConfirmed = Unchecked.defaultof<Event<DeliveryConfirmed>> with get, set
    
    // Note: Real MassTransit saga requires more complex setup
    // This is a simplified representation
```

## 7. Message Serialization

```fsharp
// ===== Custom Serialization =====
open System.Text.Json
open MassTransit

// MassTransit uses System.Text.Json by default in v8+
// Configure custom options
let configureJsonSerialization (services: IServiceCollection) =
    services.AddMassTransit(fun x ->
        x.UsingRabbitMq(fun ctx cfg ->
            cfg.Host("localhost", "/", fun h ->
                h.Username("admin")
                h.Password("password"))
            
            // Configure JSON
            cfg.UseJsonSerializer()
            cfg.ConfigureJsonSerializerOptions(fun opts ->
                opts.PropertyNamingPolicy <- JsonNamingPolicy.CamelCase
                opts.WriteIndented <- false
                opts)
            
            cfg.ConfigureEndpoints(ctx)
        )
    ) |> ignore

// Message headers and metadata
type MessageWithMetadata<'T>(payload: 'T, correlationId: string, causationId: string option) =
    member _.Payload = payload
    member _.CorrelationId = correlationId
    member _.CausationId = causationId
    member _.Timestamp = System.DateTime.UtcNow
    member _.Source = "order-service"
    member _.Version = "1.0"

// Using message filters for adding metadata
type CorrelationIdFilter<'T when 'T : not struct>() =
    interface IFilter<SendContext<'T>> with
        member _.Probe(probe: IProbeContext) = ()
        
        member _.Send(context: SendContext<'T>, next: IPipe<SendContext<'T>>) =
            task {
                if System.String.IsNullOrEmpty(context.CorrelationId.ToString()) then
                    context.CorrelationId <- System.Guid.NewGuid()
                
                do! next.Send(context)
            }
```

## 8. Error Handling and Retry

```fsharp
// ===== Error Handling =====

// Custom exception for business errors (should NOT retry)
exception OrderAlreadyExistsException of orderId: string
exception InsufficientFundsException of accountId: string * amount: decimal

// Consumer with proper error handling
type RobustOrderConsumer() =
    interface IConsumer<SubmitOrder> with
        member _.Consume(context: ConsumeContext<SubmitOrder>) =
            task {
                let message = context.Message
                
                try
                    // Business logic that might throw
                    if message.TotalAmount <= 0m then
                        raise (System.ArgumentException("Amount must be positive"))
                    
                    // Simulate duplicate check
                    if message.OrderId = "duplicate" then
                        raise (OrderAlreadyExistsException message.OrderId)
                    
                    printfn "[RobustOrderConsumer] Processing order %s" message.OrderId
                    
                    // Success - publish event
                    do! context.Publish<OrderSubmitted>({
                        OrderId = message.OrderId
                        CustomerId = message.CustomerId
                        TotalAmount = message.TotalAmount
                        SubmittedAt = message.SubmittedAt
                    })
                    
                with
                | OrderAlreadyExistsException orderId ->
                    // Don't retry - business error
                    printfn "[RobustOrderConsumer] Order %s already exists, skipping" orderId
                    // Optionally: publish error event
                    
                | ex ->
                    // Re-throw for MassTransit retry logic
                    printfn "[RobustOrderConsumer] Error processing order %s: %s" message.OrderId ex.Message
                    reraise()
            }

// Configure retry with exponential backoff
let configureRetry (services: IServiceCollection) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<RobustOrderConsumer>()
        
        x.UsingRabbitMq(fun ctx cfg ->
            cfg.Host("localhost", "/", fun h ->
                h.Username("admin")
                h.Password("password"))
            
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<RobustOrderConsumer>(ctx)
                
                // Retry policy: 3 retries with exponential backoff
                e.UseMessageRetry(fun r ->
                    r.Exponential(
                        retryLimit = 3,
                        minInterval = System.TimeSpan.FromSeconds(1.0),
                        maxInterval = System.TimeSpan.FromSeconds(30.0),
                        intervalDelta = System.TimeSpan.FromSeconds(5.0)
                    )
                    // Don't retry business errors
                    r.Ignore<OrderAlreadyExistsException>()
                    r.Ignore<InsufficientFundsException>()
                )
                
                // Concurrent message limit
                e.PrefetchCount <- 10
                e.ConcurrentMessageLimit <- 5
            )
        )
    ) |> ignore
```

## 9. Dead Letter Queue (DLQ)

```fsharp
// ===== Dead Letter Queue Configuration =====

// Configure DLQ on error
let configureDlq (services: IServiceCollection) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<RobustOrderConsumer>()
        
        x.UsingRabbitMq(fun ctx cfg ->
            cfg.Host("localhost", "/", fun h ->
                h.Username("admin")
                h.Password("password"))
            
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<RobustOrderConsumer>(ctx)
                
                // After max retries, send to DLQ
                e.UseMessageRetry(fun r -> r.Immediate(3))
                
                // DLQ: messages go to "order-submission_error" queue
                // Default behavior in MassTransit
                
                // Or custom error queue:
                e.ConfigureDeadLetterQueueDeadLetterTransport() |> ignore
            )
        )
    ) |> ignore

// Monitor and reprocess DLQ
type DeadLetterProcessor(bus: IBus) =
    
    member _.ReprocessMessage (errorQueue: string) (messageId: string) =
        async {
            // In production: read from DLQ and republish
            printfn "Reprocessing message %s from %s" messageId errorQueue
            
            // Get message from DLQ (implementation depends on transport)
            // Republish to original queue
            // do! bus.Publish<SubmitOrder>(originalMessage)
        }
    
    member _.MonitorDlq (errorQueue: string) =
        async {
            // Check DLQ metrics periodically
            // Alert if messages accumulate
            printfn "Monitoring DLQ: %s" errorQueue
        }
```

## 10. Azure Service Bus

```fsharp
// ===== Azure Service Bus =====
// MassTransit.Azure.ServiceBus.Core

let configureAzureServiceBusDetailed (services: IServiceCollection) =
    let connectionString = "Endpoint=sb://myapp.servicebus.windows.net/;SharedAccessKeyName=..."
    
    services.AddMassTransit(fun x ->
        x.AddConsumer<SubmitOrderConsumer>()
        x.AddConsumer<ProcessPaymentConsumer>()
        
        x.UsingAzureServiceBus(fun ctx cfg ->
            cfg.Host(connectionString)
            
            // Configure topics (for pub/sub)
            cfg.Message<OrderSubmitted>(fun m ->
                m.SetEntityName("order-submitted"))
            
            // Configure subscriptions
            cfg.SubscriptionEndpoint<OrderSubmitted>(
                "notification-service",
                fun e ->
                    e.ConfigureConsumer<OrderSubmittedNotificationConsumer>(ctx)
                    e.UseMessageRetry(fun r -> r.Immediate(3))
            )
            
            // Configure queues
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<SubmitOrderConsumer>(ctx)
                e.RequiresSession <- true  // Session-based ordering
                e.MaxDeliveryCount <- 5    // DLQ after 5 failures
            )
        )
    ) |> ignore
```

## 11. Amazon SQS/SNS

```fsharp
// ===== Amazon SQS/SNS =====
// MassTransit.Amazon.SQS

let configureAwsSqs (services: IServiceCollection) (region: string) =
    services.AddMassTransit(fun x ->
        x.AddConsumer<SubmitOrderConsumer>()
        
        x.UsingAmazonSqs(fun ctx cfg ->
            cfg.Host(region, fun h ->
                // Use environment variables or IAM roles in production
                h.AccessKey(System.Environment.GetEnvironmentVariable("AWS_ACCESS_KEY_ID"))
                h.SecretKey(System.Environment.GetEnvironmentVariable("AWS_SECRET_ACCESS_KEY")))
            
            cfg.ReceiveEndpoint("order-submission", fun e ->
                e.ConfigureConsumer<SubmitOrderConsumer>(ctx)
                
                // SQS-specific settings
                e.WaitTimeSeconds <- 20  // Long polling
                e.VisibilityTimeout <- System.TimeSpan.FromMinutes(5.0)
            )
        )
    ) |> ignore
```

## 12. Complete Example: Order Processing System

```fsharp
// ===== Complete Order Processing Demo =====

// Simplified in-memory MassTransit-like system for demo
module OrderProcessingDemo =
    
    open System
    open System.Collections.Generic
    
    // Message broker simulation
    type MessageBroker() =
        let queues = Dictionary<string, Queue<obj>>()
        let subscribers = Dictionary<string, ResizeArray<obj -> Async<unit>>>()
        
        member _.Publish<'T> (message: 'T) =
            async {
                let key = typeof<'T>.Name
                if subscribers.ContainsKey(key) then
                    for handler in subscribers.[key] do
                        do! handler (message :> obj)
            }
        
        member _.Subscribe<'T> (handler: 'T -> Async<unit>) =
            let key = typeof<'T>.Name
            if not (subscribers.ContainsKey(key)) then
                subscribers.[key] <- ResizeArray()
            subscribers.[key].Add(fun o -> handler (o :?> 'T))
    
    // Domain
    type Order = {
        Id: string
        CustomerId: string
        Items: string list
        Total: decimal
        Status: string
    }
    
    // Messages
    type SubmitOrderMsg = { OrderId: string; CustomerId: string; Items: string list; Total: decimal }
    type OrderSubmittedEvent = { OrderId: string; CustomerId: string; Total: decimal }
    type PaymentRequestMsg = { OrderId: string; CustomerId: string; Amount: decimal }
    type PaymentCompletedEvent = { OrderId: string; Success: bool }
    type ShipOrderMsg = { OrderId: string }
    type OrderShippedEvent = { OrderId: string; TrackingNo: string }
    
    // Services
    let orders = Dictionary<string, Order>()
    let broker = MessageBroker()
    
    // Setup handlers
    let setup () =
        // Order Service - handles order submission
        broker.Subscribe<SubmitOrderMsg>(fun msg ->
            async {
                printfn "[OrderService] Processing order %s" msg.OrderId
                
                let order = {
                    Id = msg.OrderId
                    CustomerId = msg.CustomerId
                    Items = msg.Items
                    Total = msg.Total
                    Status = "Submitted"
                }
                orders.[msg.OrderId] <- order
                
                // Publish event
                do! broker.Publish<OrderSubmittedEvent>({
                    OrderId = msg.OrderId
                    CustomerId = msg.CustomerId
                    Total = msg.Total
                })
                
                // Send payment request
                do! broker.Publish<PaymentRequestMsg>({
                    OrderId = msg.OrderId
                    CustomerId = msg.CustomerId
                    Amount = msg.Total
                })
            })
        
        // Payment Service - handles payment
        broker.Subscribe<PaymentRequestMsg>(fun msg ->
            async {
                printfn "[PaymentService] Processing payment for order %s (%.2f)" msg.OrderId msg.Amount
                do! Async.Sleep 100  // Simulate payment processing
                
                // Success (in real life, call payment gateway)
                do! broker.Publish<PaymentCompletedEvent>({
                    OrderId = msg.OrderId
                    Success = true
                })
            })
        
        // Order Service - handles payment completion
        broker.Subscribe<PaymentCompletedEvent>(fun event ->
            async {
                if event.Success then
                    if orders.ContainsKey(event.OrderId) then
                        orders.[event.OrderId] <- { orders.[event.OrderId] with Status = "PaymentConfirmed" }
                    
                    printfn "[OrderService] Payment confirmed for order %s" event.OrderId
                    
                    // Send shipping request
                    do! broker.Publish<ShipOrderMsg>({ OrderId = event.OrderId })
                else
                    if orders.ContainsKey(event.OrderId) then
                        orders.[event.OrderId] <- { orders.[event.OrderId] with Status = "PaymentFailed" }
                    printfn "[OrderService] Payment failed for order %s" event.OrderId
            })
        
        // Shipping Service - handles shipping
        broker.Subscribe<ShipOrderMsg>(fun msg ->
            async {
                printfn "[ShippingService] Shipping order %s" msg.OrderId
                do! Async.Sleep 200  // Simulate shipping
                
                let trackingNo = sprintf "TH%s" (Guid.NewGuid().ToString("N")[..9].ToUpper())
                
                do! broker.Publish<OrderShippedEvent>({
                    OrderId = msg.OrderId
                    TrackingNo = trackingNo
                })
            })
        
        // Notification Service - handles events
        broker.Subscribe<OrderSubmittedEvent>(fun event ->
            async {
                printfn "[NotificationService] Sending confirmation email for order %s" event.OrderId
            })
        
        broker.Subscribe<OrderShippedEvent>(fun event ->
            async {
                if orders.ContainsKey(event.OrderId) then
                    orders.[event.OrderId] <- { orders.[event.OrderId] with Status = "Shipped" }
                printfn "[NotificationService] Order %s shipped! Tracking: %s" event.OrderId event.TrackingNo
            })
    
    // Run demo
    let run () =
        async {
            setup()
            
            printfn "=== Message Queue Demo ==="
            printfn "Processing order through message queue pipeline..."
            printfn ""
            
            // Submit an order
            do! broker.Publish<SubmitOrderMsg>({
                OrderId = "ORD-" + Guid.NewGuid().ToString("N")[..7].ToUpper()
                CustomerId = "CUST-001"
                Items = ["Widget A x2"; "Widget B x1"]
                Total = 349.97m
            })
            
            // Wait for async processing
            do! Async.Sleep 1000
            
            printfn ""
            printfn "Final order statuses:"
            for kvp in orders do
                printfn "  Order %s: %s" kvp.Key kvp.Value.Status
        }

OrderProcessingDemo.run() |> Async.RunSynchronously
```

## 13. สรุป Message Queues กับ F#

```fsharp
(*
Message Queue Key Concepts:

1. MassTransit Abstractions
   - IConsumer<T>: Handle messages
   - IPublishEndpoint: Publish events (fan-out)
   - ISendEndpointProvider: Send commands (point-to-point)
   - IRequestClient<T>: Request/Response pattern

2. Message Types
   - Commands: SubmitOrder, ProcessPayment (one consumer)
   - Events: OrderSubmitted, PaymentProcessed (many consumers)

3. Transport Options
   - RabbitMQ: Open source, flexible, on-premise
   - Azure Service Bus: Managed, sessions, transactions
   - Amazon SQS/SNS: AWS-native, simple

4. Reliability
   - Retry: Automatic with exponential backoff
   - Dead Letter Queue: Failed messages go here
   - Saga: Coordinating multi-step processes
   - Outbox: Transactional publishing

5. F# Advantages
   - Immutable messages (F# records)
   - DUs for message types
   - Async workflows for consumers
   - Type-safe message contracts

Best Practices:
✓ Design messages as immutable value objects
✓ Version your messages (v1, v2)
✓ Use correlation/causation IDs
✓ Handle idempotency
✓ Monitor DLQ
✓ Test with in-memory transport
*)

printfn "Message Queues with F# - Complete!"
```

---

**สรุป**: MassTransit ทำให้การใช้ Message Queues ใน F# ง่ายขึ้นมาก โดย abstract รายละเอียดของ transport (RabbitMQ, Azure Service Bus, SQS) และให้ patterns ที่ดีสำหรับ reliability เช่น retry, DLQ, และ saga
