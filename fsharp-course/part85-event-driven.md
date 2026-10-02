# Part 85 - Event-Driven Architecture

## บทนำ (Introduction)

Event-Driven Architecture (EDA) คือรูปแบบการออกแบบที่ services สื่อสารกันผ่าน events แทนที่จะ call กันโดยตรง ทำให้ระบบ loosely coupled และ scalable

## 1. Domain Events vs Integration Events

```fsharp
// ===== Domain Events =====
// เกิดขึ้นภายใน domain - ใช้ภายใน bounded context เดียวกัน

type OrderDomainEvent =
    | OrderCreated of {| OrderId: string; CustomerId: string; Items: string list; At: System.DateTime |}
    | ItemAdded of {| OrderId: string; ProductId: string; Quantity: int; At: System.DateTime |}
    | OrderConfirmed of {| OrderId: string; TotalAmount: decimal; At: System.DateTime |}
    | OrderCancelled of {| OrderId: string; Reason: string; At: System.DateTime |}
    | PaymentReceived of {| OrderId: string; Amount: decimal; TransactionId: string; At: System.DateTime |}

// ===== Integration Events =====
// ส่งข้าม bounded contexts - ต้องเป็น stable, versioned

[<System.Serializable>]
type OrderPlacedIntegrationEvent = {
    EventId: string
    EventVersion: string  // "v1"
    OccurredAt: System.DateTime
    OrderId: string
    CustomerId: string
    Items: OrderItemDto list
    TotalAmount: decimal
    Currency: string
    ShippingAddress: AddressDto
}

and OrderItemDto = {
    ProductId: string
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

and AddressDto = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

[<System.Serializable>]
type OrderShippedIntegrationEvent = {
    EventId: string
    EventVersion: string
    OccurredAt: System.DateTime
    OrderId: string
    TrackingNumber: string
    EstimatedDelivery: System.DateTime
}

[<System.Serializable>]
type PaymentProcessedIntegrationEvent = {
    EventId: string
    EventVersion: string
    OccurredAt: System.DateTime
    OrderId: string
    PaymentId: string
    Amount: decimal
    Success: bool
    FailureReason: string option
}

// Event factory
let createIntegrationEvent<'T> (payload: 'T) version =
    // แยก payload ออกจาก envelope
    payload
```

## 2. Event Bus Implementation

```fsharp
open System
open System.Collections.Generic
open System.Text.Json

// ===== Event Envelope =====
type EventEnvelope = {
    EventId: string
    EventType: string
    EventVersion: string
    OccurredAt: DateTime
    CorrelationId: string option
    CausationId: string option  // ID ของ event ที่ทำให้เกิด event นี้
    Source: string              // ชื่อ service ที่สร้าง event
    Payload: string             // JSON payload
}

module EventEnvelope =
    let create<'T> (payload: 'T) (eventType: string) (source: string) = {
        EventId = Guid.NewGuid().ToString()
        EventType = eventType
        EventVersion = "v1"
        OccurredAt = DateTime.UtcNow
        CorrelationId = None
        CausationId = None
        Source = source
        Payload = JsonSerializer.Serialize(payload)
    }
    
    let withCorrelation correlationId envelope =
        { envelope with CorrelationId = Some correlationId }
    
    let withCausation causationId envelope =
        { envelope with CausationId = Some causationId }
    
    let deserializePayload<'T> (envelope: EventEnvelope) : 'T =
        JsonSerializer.Deserialize<'T>(envelope.Payload)

// ===== Event Bus Interface =====
type EventHandler = EventEnvelope -> Async<unit>

type IEventBus =
    abstract member PublishAsync: EventEnvelope -> Async<unit>
    abstract member Subscribe: eventType: string -> EventHandler -> unit
    abstract member Unsubscribe: eventType: string -> unit

// ===== In-Memory Event Bus (for testing) =====
type InMemoryEventBus() =
    let handlers = Dictionary<string, ResizeArray<EventHandler>>()
    let publishedEvents = ResizeArray<EventEnvelope>()
    let lockObj = obj()
    
    interface IEventBus with
        member _.PublishAsync envelope =
            async {
                lock lockObj (fun () -> publishedEvents.Add(envelope))
                
                let handlerList =
                    lock lockObj (fun () ->
                        if handlers.ContainsKey(envelope.EventType) then
                            handlers.[envelope.EventType] |> Seq.toList
                        else [])
                
                for handler in handlerList do
                    do! handler envelope
            }
        
        member _.Subscribe eventType handler =
            lock lockObj (fun () ->
                if not (handlers.ContainsKey(eventType)) then
                    handlers.[eventType] <- ResizeArray()
                handlers.[eventType].Add(handler))
        
        member _.Unsubscribe eventType =
            lock lockObj (fun () ->
                handlers.Remove(eventType) |> ignore)
    
    member _.PublishedEvents = lock lockObj (fun () -> publishedEvents |> Seq.toList)
    member _.ClearEvents () = lock lockObj (fun () -> publishedEvents.Clear())
```

## 3. Event Handlers

```fsharp
// ===== Event Handlers =====
// แต่ละ service subscribe ต่อ events ที่สนใจ

// Notification Service Handler
type NotificationEventHandler(emailService: IEmailService, smsService: ISmsService) =
    
    let handleOrderPlaced (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<OrderPlacedIntegrationEvent> envelope
            
            printfn "[NotificationHandler] Order placed: %s" event.OrderId
            
            // Send confirmation email
            do! emailService.Send {|
                To = event.CustomerId  // In real code, look up email
                Subject = sprintf "Order Confirmed: #%s" event.OrderId.[..7]
                Body = sprintf """
                    <h1>Order Confirmed!</h1>
                    <p>Your order #%s has been placed successfully.</p>
                    <p>Total: %s %.2f</p>
                    <p>We'll notify you when it ships.</p>
                """ event.OrderId event.Currency event.TotalAmount
                IsHtml = true
            |}
            
            printfn "[NotificationHandler] Confirmation email sent for order %s" event.OrderId
        }
    
    let handleOrderShipped (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<OrderShippedIntegrationEvent> envelope
            
            // Send shipping notification
            do! smsService.Send {|
                To = "+66812345678"  // Look up from customer
                Message = sprintf "Your order %s has shipped! Track: %s" event.OrderId.[..7] event.TrackingNumber
            |}
            
            printfn "[NotificationHandler] Shipping SMS sent for order %s" event.OrderId
        }
    
    member this.RegisterHandlers (bus: IEventBus) =
        bus.Subscribe "OrderPlacedIntegrationEvent" handleOrderPlaced
        bus.Subscribe "OrderShippedIntegrationEvent" handleOrderShipped
        printfn "[NotificationHandler] Registered for order events"

and IEmailService =
    abstract member Send: {| To: string; Subject: string; Body: string; IsHtml: bool |} -> Async<unit>

and ISmsService =
    abstract member Send: {| To: string; Message: string |} -> Async<unit>

// Inventory Handler
type InventoryEventHandler() =
    
    let stockReservations = Dictionary<string, Map<string, int>>()  // orderId -> (productId -> qty)
    
    let handleOrderPlaced (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<OrderPlacedIntegrationEvent> envelope
            
            printfn "[InventoryHandler] Reserving stock for order %s" event.OrderId
            
            let reservations =
                event.Items
                |> List.map (fun item -> item.ProductId, item.Quantity)
                |> Map.ofList
            
            stockReservations.[event.OrderId] <- reservations
            
            printfn "[InventoryHandler] Reserved %d items for order %s" 
                    event.Items.Length event.OrderId
        }
    
    let handleOrderCancelled (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<{| OrderId: string; Reason: string |}> envelope
            
            if stockReservations.ContainsKey(event.OrderId) then
                stockReservations.Remove(event.OrderId) |> ignore
                printfn "[InventoryHandler] Released reservation for cancelled order %s" event.OrderId
        }
    
    member _.RegisterHandlers (bus: IEventBus) =
        bus.Subscribe "OrderPlacedIntegrationEvent" handleOrderPlaced
        bus.Subscribe "OrderCancelledIntegrationEvent" handleOrderCancelled

// Analytics Handler
type AnalyticsEventHandler() =
    let orderStats = Dictionary<string, decimal>()  // date -> revenue
    
    let handleOrderPlaced (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<OrderPlacedIntegrationEvent> envelope
            let date = event.OccurredAt.ToString("yyyy-MM-dd")
            let current = 
                if orderStats.ContainsKey(date) then orderStats.[date]
                else 0m
            orderStats.[date] <- current + event.TotalAmount
            printfn "[Analytics] Daily revenue for %s: %.2f" date (orderStats.[date])
        }
    
    member _.RegisterHandlers (bus: IEventBus) =
        bus.Subscribe "OrderPlacedIntegrationEvent" handleOrderPlaced
    
    member _.GetDailyRevenue date =
        if orderStats.ContainsKey(date) then orderStats.[date]
        else 0m
```

## 4. Outbox Pattern (Reliability)

```fsharp
// ===== Outbox Pattern =====
// ป้องกัน lost events เมื่อ service crash ระหว่าง publish

type OutboxMessage = {
    Id: string
    EventType: string
    Payload: string
    CreatedAt: DateTime
    ProcessedAt: DateTime option
    RetryCount: int
    Error: string option
}

type IOutboxRepository =
    abstract member Save: OutboxMessage -> Async<unit>
    abstract member GetPending: limit: int -> Async<OutboxMessage list>
    abstract member MarkProcessed: id: string -> Async<unit>
    abstract member MarkFailed: id: string -> error: string -> Async<unit>
    abstract member IncrementRetry: id: string -> Async<unit>

// In-memory outbox for demo
type InMemoryOutboxRepository() =
    let messages = Dictionary<string, OutboxMessage>()
    let lockObj = obj()
    
    interface IOutboxRepository with
        member _.Save message =
            async { lock lockObj (fun () -> messages.[message.Id] <- message) }
        
        member _.GetPending limit =
            async {
                return lock lockObj (fun () ->
                    messages.Values
                    |> Seq.filter (fun m -> m.ProcessedAt.IsNone && m.RetryCount < 3)
                    |> Seq.sortBy (fun m -> m.CreatedAt)
                    |> Seq.truncate limit
                    |> Seq.toList)
            }
        
        member _.MarkProcessed id =
            async {
                lock lockObj (fun () ->
                    if messages.ContainsKey(id) then
                        messages.[id] <- { messages.[id] with ProcessedAt = Some DateTime.UtcNow })
            }
        
        member _.MarkFailed id error =
            async {
                lock lockObj (fun () ->
                    if messages.ContainsKey(id) then
                        messages.[id] <- { messages.[id] with Error = Some error })
            }
        
        member _.IncrementRetry id =
            async {
                lock lockObj (fun () ->
                    if messages.ContainsKey(id) then
                        messages.[id] <- { messages.[id] with RetryCount = messages.[id].RetryCount + 1 })
            }

// Transactional outbox service
type OutboxService(outboxRepo: IOutboxRepository, eventBus: IEventBus) =
    
    // Save event to outbox (transactionally with business operation)
    member _.SaveEvent<'T> (event: 'T) (eventType: string) =
        async {
            let message = {
                Id = Guid.NewGuid().ToString()
                EventType = eventType
                Payload = JsonSerializer.Serialize(event)
                CreatedAt = DateTime.UtcNow
                ProcessedAt = None
                RetryCount = 0
                Error = None
            }
            do! outboxRepo.Save message
            printfn "[Outbox] Event saved: %s (%s)" message.Id eventType
        }
    
    // Background processor - reads outbox and publishes
    member _.ProcessOutbox () =
        async {
            while true do
                try
                    let! pending = outboxRepo.GetPending 10
                    
                    for message in pending do
                        try
                            let envelope = {
                                EventId = message.Id
                                EventType = message.EventType
                                EventVersion = "v1"
                                OccurredAt = message.CreatedAt
                                CorrelationId = None
                                CausationId = None
                                Source = "outbox-processor"
                                Payload = message.Payload
                            }
                            
                            do! eventBus.PublishAsync envelope
                            do! outboxRepo.MarkProcessed message.Id
                            printfn "[Outbox] Published: %s" message.Id
                        with ex ->
                            do! outboxRepo.IncrementRetry message.Id
                            do! outboxRepo.MarkFailed message.Id ex.Message
                            printfn "[Outbox] Failed to publish %s: %s" message.Id ex.Message
                    
                    do! Async.Sleep 1000  // Process every second
                with ex ->
                    printfn "[Outbox] Error: %s" ex.Message
                    do! Async.Sleep 5000
        }
```

## 5. Inbox Pattern (Idempotency)

```fsharp
// ===== Inbox Pattern =====
// ป้องกัน duplicate processing เมื่อ message ถูก deliver มากกว่าหนึ่งครั้ง

type InboxMessage = {
    EventId: string
    EventType: string
    ProcessedAt: DateTime
}

type IInboxRepository =
    abstract member IsProcessed: eventId: string -> Async<bool>
    abstract member MarkProcessed: eventId: string -> eventType: string -> Async<unit>

type InMemoryInboxRepository() =
    let processed = Collections.Concurrent.ConcurrentDictionary<string, InboxMessage>()
    
    interface IInboxRepository with
        member _.IsProcessed eventId =
            async { return processed.ContainsKey(eventId) }
        
        member _.MarkProcessed eventId eventType =
            async {
                processed.[eventId] <- {
                    EventId = eventId
                    EventType = eventType
                    ProcessedAt = DateTime.UtcNow
                }
            }

// Idempotent event handler wrapper
let makeIdempotent (inboxRepo: IInboxRepository) (handler: EventEnvelope -> Async<unit>) =
    fun (envelope: EventEnvelope) ->
        async {
            let! alreadyProcessed = inboxRepo.IsProcessed envelope.EventId
            
            if alreadyProcessed then
                printfn "[Inbox] Skipping duplicate event: %s" envelope.EventId
            else
                do! handler envelope
                do! inboxRepo.MarkProcessed envelope.EventId envelope.EventType
                printfn "[Inbox] Processed event: %s" envelope.EventId
        }
```

## 6. Saga Orchestration

```fsharp
// ===== Saga Orchestration =====
// Saga coordinator บัญชา flow ของ distributed transaction

type SagaState =
    | Started
    | ValidatingPayment
    | PaymentValidated
    | ReservingInventory
    | InventoryReserved
    | ProcessingShipment
    | Completed
    | Compensating of failedStep: string
    | Failed of reason: string

type SagaStep = {
    Name: string
    Execute: Async<Result<unit, string>>
    Compensate: Async<unit>
}

type OrderSaga = {
    SagaId: string
    OrderId: string
    State: SagaState
    CompletedSteps: string list
    StartedAt: DateTime
    UpdatedAt: DateTime
}

module OrderSaga =
    let create orderId = {
        SagaId = Guid.NewGuid().ToString()
        OrderId = orderId
        State = Started
        CompletedSteps = []
        StartedAt = DateTime.UtcNow
        UpdatedAt = DateTime.UtcNow
    }
    
    let transition saga newState =
        { saga with State = newState; UpdatedAt = DateTime.UtcNow }
    
    let completeStep saga stepName =
        { saga with 
            CompletedSteps = saga.CompletedSteps @ [stepName]
            UpdatedAt = DateTime.UtcNow }

// Saga Orchestrator
type PlaceOrderSagaOrchestrator(
    paymentService: IPaymentService,
    inventoryService: IInventoryService,
    shippingService: IShippingService,
    eventBus: IEventBus) =
    
    let sagaStore = Dictionary<string, OrderSaga>()
    
    member _.Execute (orderId: string) (amount: decimal) (customerId: string) =
        async {
            let saga = OrderSaga.create orderId
            sagaStore.[saga.SagaId] <- saga
            
            printfn "[Saga] Starting order saga %s for order %s" saga.SagaId orderId
            
            // Step 1: Validate Payment
            let saga = OrderSaga.transition saga ValidatingPayment
            sagaStore.[saga.SagaId] <- saga
            
            match! paymentService.Validate customerId amount with
            | Error e ->
                let failed = OrderSaga.transition saga (Failed (sprintf "Payment validation failed: %s" e))
                sagaStore.[failed.SagaId] <- failed
                return Error e
            | Ok paymentId ->
            
            let saga = OrderSaga.completeStep (OrderSaga.transition saga PaymentValidated) "ValidatePayment"
            sagaStore.[saga.SagaId] <- saga
            printfn "[Saga] Payment validated: %s" paymentId
            
            // Step 2: Reserve Inventory
            let saga = OrderSaga.transition saga ReservingInventory
            sagaStore.[saga.SagaId] <- saga
            
            match! inventoryService.Reserve orderId with
            | Error e ->
                // Compensate: cancel payment
                do! paymentService.Cancel paymentId
                let failed = OrderSaga.transition saga (Failed (sprintf "Inventory reservation failed: %s" e))
                sagaStore.[failed.SagaId] <- failed
                return Error e
            | Ok reservationId ->
            
            let saga = OrderSaga.completeStep (OrderSaga.transition saga InventoryReserved) "ReserveInventory"
            sagaStore.[saga.SagaId] <- saga
            printfn "[Saga] Inventory reserved: %s" reservationId
            
            // Step 3: Process Shipment
            let saga = OrderSaga.transition saga ProcessingShipment
            sagaStore.[saga.SagaId] <- saga
            
            match! shippingService.Schedule orderId with
            | Error e ->
                // Compensate: release inventory, cancel payment
                do! inventoryService.Release reservationId
                do! paymentService.Cancel paymentId
                let failed = OrderSaga.transition saga (Failed (sprintf "Shipping failed: %s" e))
                sagaStore.[failed.SagaId] <- failed
                return Error e
            | Ok trackingNumber ->
            
            // All steps completed
            let completed = OrderSaga.completeStep (OrderSaga.transition saga Completed) "ProcessShipment"
            sagaStore.[completed.SagaId] <- completed
            
            // Publish success event
            let envelope = EventEnvelope.create
                                {| OrderId = orderId; TrackingNumber = trackingNumber; SagaId = saga.SagaId |}
                                "OrderSagaCompleted"
                                "order-saga"
            do! eventBus.PublishAsync envelope
            
            printfn "[Saga] Saga %s completed successfully!" saga.SagaId
            return Ok trackingNumber
        }

and IPaymentService =
    abstract member Validate: customerId: string -> amount: decimal -> Async<Result<string, string>>
    abstract member Cancel: paymentId: string -> Async<unit>

and IInventoryService =
    abstract member Reserve: orderId: string -> Async<Result<string, string>>
    abstract member Release: reservationId: string -> Async<unit>

and IShippingService =
    abstract member Schedule: orderId: string -> Async<Result<string, string>>

// Mock implementations
type MockPaymentService() =
    interface IPaymentService with
        member _.Validate customerId amount =
            async {
                printfn "[Payment] Validating %.2f for %s" amount customerId
                do! Async.Sleep 50
                return Ok (sprintf "PAY-%s" (Guid.NewGuid().ToString("N")[..7]))
            }
        member _.Cancel paymentId =
            async { printfn "[Payment] Cancelled %s" paymentId }

type MockInventoryService() =
    interface IInventoryService with
        member _.Reserve orderId =
            async {
                printfn "[Inventory] Reserving for order %s" orderId
                do! Async.Sleep 30
                return Ok (sprintf "RES-%s" (Guid.NewGuid().ToString("N")[..7]))
            }
        member _.Release reservationId =
            async { printfn "[Inventory] Released %s" reservationId }

type MockShippingService() =
    interface IShippingService with
        member _.Schedule orderId =
            async {
                printfn "[Shipping] Scheduling for order %s" orderId
                do! Async.Sleep 40
                return Ok (sprintf "TH%s" (Guid.NewGuid().ToString("N")[..9].ToUpper()))
            }
```

## 7. Choreography (Event-Based)

```fsharp
// ===== Choreography =====
// Services react to events ไม่มี central coordinator

// แต่ละ service สร้าง event และ react ต่อ events ของ service อื่น

module OrderServiceChoreography =
    
    type IOrderRepository =
        abstract member FindById: string -> Async<Order option>
        abstract member Save: Order -> Async<unit>
    
    and Order = {
        Id: string
        CustomerId: string
        Status: string
        TrackingNumber: string option
        PaymentId: string option
    }
    
    // React to payment confirmation
    let handlePaymentConfirmed (repo: IOrderRepository) (bus: IEventBus) (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<PaymentProcessedIntegrationEvent> envelope
            
            if event.Success then
                let! orderOpt = repo.FindById event.OrderId
                match orderOpt with
                | None -> printfn "[OrderService] Order not found: %s" event.OrderId
                | Some order ->
                    let updated = { order with 
                                        Status = "PaymentConfirmed"
                                        PaymentId = Some event.PaymentId }
                    do! repo.Save updated
                    
                    // Emit next event for inventory to pick up
                    let nextEvent = EventEnvelope.create
                                        {| OrderId = event.OrderId; PaymentId = event.PaymentId |}
                                        "OrderReadyForFulfillment"
                                        "order-service"
                    do! bus.PublishAsync nextEvent
                    printfn "[OrderService] Order %s ready for fulfillment" event.OrderId
        }

module InventoryServiceChoreography =
    
    // React to order ready for fulfillment
    let handleOrderReadyForFulfillment (bus: IEventBus) (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<{| OrderId: string; PaymentId: string |}> envelope
            
            printfn "[InventoryService] Processing fulfillment for order %s" event.OrderId
            
            // Check and reserve stock
            do! Async.Sleep 20  // Simulate
            
            // Emit fulfillment event
            let fulfillmentEvent = EventEnvelope.create
                                        {| OrderId = event.OrderId; ReservationId = "RES-001" |}
                                        "InventoryFulfilled"
                                        "inventory-service"
            do! bus.PublishAsync fulfillmentEvent
            printfn "[InventoryService] Inventory fulfilled for order %s" event.OrderId
        }

module ShippingServiceChoreography =
    
    // React to inventory fulfillment
    let handleInventoryFulfilled (bus: IEventBus) (envelope: EventEnvelope) =
        async {
            let event = EventEnvelope.deserializePayload<{| OrderId: string; ReservationId: string |}> envelope
            
            printfn "[ShippingService] Scheduling shipment for order %s" event.OrderId
            
            let trackingNumber = sprintf "TH%s" (Guid.NewGuid().ToString("N")[..9].ToUpper())
            
            // Emit shipped event
            let shippedEvent = EventEnvelope.create
                                    {| 
                                        OrderId = event.OrderId
                                        TrackingNumber = trackingNumber
                                        EstimatedDelivery = DateTime.UtcNow.AddDays(3.0)
                                    |}
                                    "OrderShippedIntegrationEvent"
                                    "shipping-service"
            do! bus.PublishAsync shippedEvent
            printfn "[ShippingService] Order %s shipped: %s" event.OrderId trackingNumber
        }
```

## 8. Dead Letter Queue (DLQ)

```fsharp
// ===== Dead Letter Queue =====
// จัดการ messages ที่ process ไม่สำเร็จ

type DeadLetter = {
    OriginalEventId: string
    OriginalEventType: string
    Payload: string
    FailureReason: string
    FailedAt: DateTime
    RetryCount: int
    LastRetryAt: DateTime option
}

type IDeadLetterQueue =
    abstract member Send: DeadLetter -> Async<unit>
    abstract member GetPending: limit: int -> Async<DeadLetter list>
    abstract member Retry: originalEventId: string -> Async<unit>
    abstract member Discard: originalEventId: string -> string -> Async<unit>

type InMemoryDeadLetterQueue() =
    let queue = Dictionary<string, DeadLetter>()
    
    interface IDeadLetterQueue with
        member _.Send letter =
            async { queue.[letter.OriginalEventId] <- letter }
        
        member _.GetPending limit =
            async {
                return queue.Values
                       |> Seq.filter (fun l -> l.RetryCount < 5)
                       |> Seq.sortBy (fun l -> l.FailedAt)
                       |> Seq.truncate limit
                       |> Seq.toList
            }
        
        member _.Retry eventId =
            async {
                if queue.ContainsKey(eventId) then
                    queue.[eventId] <- {
                        queue.[eventId] with
                            RetryCount = queue.[eventId].RetryCount + 1
                            LastRetryAt = Some DateTime.UtcNow
                    }
            }
        
        member _.Discard eventId reason =
            async {
                queue.Remove(eventId) |> ignore
                printfn "[DLQ] Discarded %s: %s" eventId reason
            }

// Resilient event handler with DLQ
let makeResilient (dlq: IDeadLetterQueue) (maxRetries: int) (handler: EventEnvelope -> Async<unit>) =
    fun (envelope: EventEnvelope) ->
        async {
            let mutable attempt = 0
            let mutable success = false
            let mutable lastError = ""
            
            while not success && attempt < maxRetries do
                attempt <- attempt + 1
                try
                    do! handler envelope
                    success <- true
                with ex ->
                    lastError <- ex.Message
                    printfn "[Resilient] Attempt %d failed for %s: %s" attempt envelope.EventId ex.Message
                    if attempt < maxRetries then
                        do! Async.Sleep (1000 * attempt)  // Exponential backoff
            
            if not success then
                let deadLetter = {
                    OriginalEventId = envelope.EventId
                    OriginalEventType = envelope.EventType
                    Payload = envelope.Payload
                    FailureReason = lastError
                    FailedAt = DateTime.UtcNow
                    RetryCount = 0
                    LastRetryAt = None
                }
                do! dlq.Send deadLetter
                printfn "[DLQ] Sent to dead letter queue: %s" envelope.EventId
        }
```

## 9. MediatR กับ F# (CQRS Event Handling)

```fsharp
// ===== MediatR-style implementation in F# =====

// Request/Response types
type IRequest<'TResponse> = interface end
type INotification = interface end

type IRequestHandler<'TRequest, 'TResponse when 'TRequest :> IRequest<'TResponse>> =
    abstract member Handle: 'TRequest -> Async<'TResponse>

type INotificationHandler<'TNotification when 'TNotification :> INotification> =
    abstract member Handle: 'TNotification -> Async<unit>

// Mediator implementation
type Mediator() =
    let requestHandlers = Dictionary<Type, obj>()
    let notificationHandlers = Dictionary<Type, ResizeArray<obj>>()
    
    member _.RegisterHandler<'TReq, 'TRes when 'TReq :> IRequest<'TRes>>
        (handler: IRequestHandler<'TReq, 'TRes>) =
        requestHandlers.[typeof<'TReq>] <- handler :> obj
    
    member _.RegisterNotificationHandler<'T when 'T :> INotification>
        (handler: INotificationHandler<'T>) =
        let key = typeof<'T>
        if not (notificationHandlers.ContainsKey(key)) then
            notificationHandlers.[key] <- ResizeArray()
        notificationHandlers.[key].Add(handler :> obj)
    
    member _.Send<'TReq, 'TRes when 'TReq :> IRequest<'TRes>> (request: 'TReq) : Async<'TRes> =
        async {
            let key = typeof<'TReq>
            if requestHandlers.ContainsKey(key) then
                let handler = requestHandlers.[key] :?> IRequestHandler<'TReq, 'TRes>
                return! handler.Handle request
            else
                return failwith (sprintf "No handler registered for %s" key.Name)
        }
    
    member _.Publish<'T when 'T :> INotification> (notification: 'T) : Async<unit> =
        async {
            let key = typeof<'T>
            if notificationHandlers.ContainsKey(key) then
                let handlers = notificationHandlers.[key]
                for handler in handlers do
                    let typedHandler = handler :?> INotificationHandler<'T>
                    do! typedHandler.Handle notification
        }

// ===== CQRS Commands =====
type CreateOrderCommand = {
    CustomerId: string
    Items: {| ProductId: string; Quantity: int |} list
} with interface IRequest<Result<string, string>>

type GetOrderQuery = {
    OrderId: string
} with interface IRequest<{| Id: string; Status: string; Total: decimal |} option>

// ===== Notifications =====
type OrderCreatedNotification = {
    OrderId: string
    CustomerId: string
    Total: decimal
} with interface INotification

// ===== Handlers =====
type CreateOrderHandler(eventBus: IEventBus) =
    interface IRequestHandler<CreateOrderCommand, Result<string, string>> with
        member _.Handle command =
            async {
                // Business logic
                let orderId = Guid.NewGuid().ToString()
                printfn "[CreateOrderHandler] Creating order %s" orderId
                
                // Publish notification
                return Ok orderId
            }

type OrderCreatedEmailHandler() =
    interface INotificationHandler<OrderCreatedNotification> with
        member _.Handle notification =
            async {
                printfn "[EmailHandler] Sending email for order %s" notification.OrderId
            }

type OrderCreatedAnalyticsHandler() =
    interface INotificationHandler<OrderCreatedNotification> with
        member _.Handle notification =
            async {
                printfn "[AnalyticsHandler] Recording order %s, total: %.2f" 
                        notification.OrderId notification.Total
            }

// ===== Demo =====
let mediatorDemo () =
    async {
        let eventBus = InMemoryEventBus() :> IEventBus
        let mediator = Mediator()
        
        // Register handlers
        mediator.RegisterHandler<CreateOrderCommand, Result<string, string>>(
            CreateOrderHandler(eventBus))
        
        mediator.RegisterNotificationHandler<OrderCreatedNotification>(
            OrderCreatedEmailHandler())
        
        mediator.RegisterNotificationHandler<OrderCreatedNotification>(
            OrderCreatedAnalyticsHandler())
        
        // Send command
        let command = {
            CustomerId = "CUST-001"
            Items = [| {| ProductId = "P001"; Quantity = 2 |} |] |> Array.toList
        }
        
        let! result = mediator.Send<CreateOrderCommand, Result<string, string>> command
        
        match result with
        | Ok orderId ->
            printfn "Order created: %s" orderId
            
            // Publish notification
            do! mediator.Publish<OrderCreatedNotification> {
                OrderId = orderId
                CustomerId = command.CustomerId
                Total = 299.99m
            }
        | Error e ->
            printfn "Error: %s" e
    }

mediatorDemo() |> Async.RunSynchronously
```

## 10. สรุป Event-Driven Architecture

```fsharp
(*
Event-Driven Architecture Key Concepts:

1. Domain Events vs Integration Events
   - Domain: Internal to bounded context
   - Integration: Cross bounded context boundaries

2. Event Bus
   - Publish-subscribe pattern
   - Topics/Queues for routing

3. Outbox Pattern
   - Ensures events are not lost
   - Transactional consistency

4. Inbox Pattern  
   - Idempotent event processing
   - Prevents duplicate processing

5. Saga Pattern
   - Distributed transaction management
   - Orchestration: Central coordinator
   - Choreography: Events drive flow

6. Dead Letter Queue
   - Failed message handling
   - Manual review and retry

7. Event Ordering
   - Use sequence numbers
   - Causation/correlation IDs
   - Partition by aggregate ID

F# Advantages:
✓ Immutable events by default (records)
✓ DUs for event types (exhaustive pattern matching)
✓ Async workflows for async processing
✓ Functional composition for event handlers
*)

printfn "Event-Driven Architecture with F# - Complete!"
```

---

**สรุป**: Event-Driven Architecture ใน F# ใช้ discriminated unions เพื่อแทน events และ immutable records สำหรับ event data ทำให้ type-safe และ reliable ส่วน Outbox/Inbox patterns ช่วยรับประกัน exactly-once processing
