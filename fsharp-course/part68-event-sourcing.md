# Part 68 - Event Sourcing

## บทนำ (Introduction)

Event Sourcing เป็น architectural pattern ที่เก็บ state ของ application เป็น sequence ของ events แทนที่จะเก็บ current state โดยตรง

**หลักการ**:
- แทนที่จะ store "Alice has $100" → store "Alice received $50, Alice spent $30, Alice received $80"
- State ปัจจุบันได้จากการ "replay" events ทั้งหมด
- Events เป็น immutable (ลบไม่ได้)
- ทุกการเปลี่ยนแปลงถูกบันทึกเป็น fact

**ข้อดี**:
- Audit log ครบถ้วน 100%
- สามารถ time travel ดู state ในอดีต
- Debug ง่าย (รู้ว่าเกิดอะไรขึ้น)
- รองรับ CQRS ได้ดี

---

## 1. Core Event Types

```fsharp
// EventTypes.fs
module EventTypes

open System

// ========================================
// 1.1 Base event type
// ========================================

type EventMetadata = {
    EventId: Guid
    EventType: string
    OccurredAt: DateTime
    StreamId: string
    Version: int64
    UserId: string option
    CorrelationId: Guid option
}

type EventEnvelope<'TEvent> = {
    Metadata: EventMetadata
    Data: 'TEvent
}

// สร้าง metadata helper
let createMetadata streamId version eventType =
    {
        EventId = Guid.NewGuid()
        EventType = eventType
        OccurredAt = DateTime.UtcNow
        StreamId = streamId
        Version = version
        UserId = None
        CorrelationId = None
    }

// ========================================
// 1.2 Bank Account Events
// ========================================

[<RequireQualifiedAccess>]
type BankAccountEvent =
    | AccountOpened of AccountOpenedData
    | MoneyDeposited of MoneyDepositedData
    | MoneyWithdrawn of MoneyWithdrawnData
    | AccountClosed of AccountClosedData
    | InterestApplied of InterestAppliedData
    | AccountFrozen of AccountFrozenData
    | AccountUnfrozen of AccountUnfrozenData

and AccountOpenedData = {
    AccountId: string
    OwnerId: string
    OwnerName: string
    InitialBalance: decimal
    AccountType: string
    OpenedAt: DateTime
}

and MoneyDepositedData = {
    AccountId: string
    Amount: decimal
    Description: string
    DepositedAt: DateTime
}

and MoneyWithdrawnData = {
    AccountId: string
    Amount: decimal
    Description: string
    WithdrawnAt: DateTime
}

and AccountClosedData = {
    AccountId: string
    Reason: string
    FinalBalance: decimal
    ClosedAt: DateTime
}

and InterestAppliedData = {
    AccountId: string
    InterestRate: decimal
    InterestAmount: decimal
    AppliedAt: DateTime
}

and AccountFrozenData = {
    AccountId: string
    Reason: string
    FrozenAt: DateTime
}

and AccountUnfrozenData = {
    AccountId: string
    UnfrozenAt: DateTime
}

// ========================================
// 1.3 Order Events
// ========================================

[<RequireQualifiedAccess>]
type OrderEvent =
    | OrderCreated of OrderCreatedData
    | ItemAdded of ItemAddedData
    | ItemRemoved of ItemRemovedData
    | OrderConfirmed of OrderConfirmedData
    | PaymentReceived of PaymentReceivedData
    | OrderShipped of OrderShippedData
    | OrderDelivered of OrderDeliveredData
    | OrderCancelled of OrderCancelledData

and OrderCreatedData = {
    OrderId: string
    CustomerId: string
    CreatedAt: DateTime
}

and ItemAddedData = {
    OrderId: string
    ProductId: string
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

and ItemRemovedData = {
    OrderId: string
    ProductId: string
    Quantity: int
}

and OrderConfirmedData = {
    OrderId: string
    ConfirmedAt: DateTime
}

and PaymentReceivedData = {
    OrderId: string
    Amount: decimal
    PaymentMethod: string
    TransactionId: string
    ReceivedAt: DateTime
}

and OrderShippedData = {
    OrderId: string
    TrackingNumber: string
    Carrier: string
    ShippedAt: DateTime
}

and OrderDeliveredData = {
    OrderId: string
    DeliveredAt: DateTime
}

and OrderCancelledData = {
    OrderId: string
    Reason: string
    CancelledAt: DateTime
}
```

---

## 2. Aggregate State (State จาก Events)

```fsharp
// Aggregates.fs
module Aggregates

open System
open EventTypes

// ========================================
// 2.1 Bank Account State
// ========================================

type BankAccountStatus =
    | Active
    | Frozen
    | Closed

type BankAccount = {
    AccountId: string
    OwnerId: string
    OwnerName: string
    Balance: decimal
    Status: BankAccountStatus
    AccountType: string
    OpenedAt: DateTime
    ClosedAt: DateTime option
    Version: int64
}

/// Initial empty state
let emptyBankAccount = {
    AccountId = ""
    OwnerId = ""
    OwnerName = ""
    Balance = 0m
    Status = Active
    AccountType = ""
    OpenedAt = DateTime.MinValue
    ClosedAt = None
    Version = 0L
}

/// Apply event to state (pure function)
let applyBankAccountEvent (state: BankAccount) (event: BankAccountEvent) : BankAccount =
    match event with
    | BankAccountEvent.AccountOpened data ->
        { state with
            AccountId = data.AccountId
            OwnerId = data.OwnerId
            OwnerName = data.OwnerName
            Balance = data.InitialBalance
            Status = Active
            AccountType = data.AccountType
            OpenedAt = data.OpenedAt
        }

    | BankAccountEvent.MoneyDeposited data ->
        { state with Balance = state.Balance + data.Amount }

    | BankAccountEvent.MoneyWithdrawn data ->
        { state with Balance = state.Balance - data.Amount }

    | BankAccountEvent.AccountClosed data ->
        { state with
            Status = Closed
            Balance = data.FinalBalance
            ClosedAt = Some data.ClosedAt
        }

    | BankAccountEvent.InterestApplied data ->
        { state with Balance = state.Balance + data.InterestAmount }

    | BankAccountEvent.AccountFrozen _ ->
        { state with Status = Frozen }

    | BankAccountEvent.AccountUnfrozen _ ->
        { state with Status = Active }

/// Reconstruct state จาก events
let replayBankAccount (events: BankAccountEvent list) : BankAccount =
    events |> List.fold applyBankAccountEvent emptyBankAccount

/// ==========================================
/// 2.2 Order State
/// ==========================================

type OrderStatus =
    | Draft
    | Confirmed
    | Paid
    | Shipped
    | Delivered
    | Cancelled

type OrderLine = {
    ProductId: string
    ProductName: string
    Quantity: int
    UnitPrice: decimal
}

type OrderState = {
    OrderId: string
    CustomerId: string
    Lines: Map<string, OrderLine>  // productId -> line
    TotalAmount: decimal
    Status: OrderStatus
    TrackingNumber: string option
    CreatedAt: DateTime
    Version: int64
}

let emptyOrder = {
    OrderId = ""
    CustomerId = ""
    Lines = Map.empty
    TotalAmount = 0m
    Status = Draft
    TrackingNumber = None
    CreatedAt = DateTime.MinValue
    Version = 0L
}

let applyOrderEvent (state: OrderState) (event: OrderEvent) : OrderState =
    match event with
    | OrderEvent.OrderCreated data ->
        { state with
            OrderId = data.OrderId
            CustomerId = data.CustomerId
            CreatedAt = data.CreatedAt
            Status = Draft
        }

    | OrderEvent.ItemAdded data ->
        let line =
            match Map.tryFind data.ProductId state.Lines with
            | Some existing ->
                { existing with Quantity = existing.Quantity + data.Quantity }
            | None ->
                { ProductId = data.ProductId; ProductName = data.ProductName; Quantity = data.Quantity; UnitPrice = data.UnitPrice }
        let newLines = Map.add data.ProductId line state.Lines
        let total = newLines |> Map.values |> Seq.sumBy (fun l -> decimal l.Quantity * l.UnitPrice)
        { state with Lines = newLines; TotalAmount = total }

    | OrderEvent.ItemRemoved data ->
        let newLines =
            match Map.tryFind data.ProductId state.Lines with
            | Some line when line.Quantity <= data.Quantity ->
                Map.remove data.ProductId state.Lines
            | Some line ->
                Map.add data.ProductId { line with Quantity = line.Quantity - data.Quantity } state.Lines
            | None ->
                state.Lines
        let total = newLines |> Map.values |> Seq.sumBy (fun l -> decimal l.Quantity * l.UnitPrice)
        { state with Lines = newLines; TotalAmount = total }

    | OrderEvent.OrderConfirmed _ ->
        { state with Status = Confirmed }

    | OrderEvent.PaymentReceived _ ->
        { state with Status = Paid }

    | OrderEvent.OrderShipped data ->
        { state with Status = Shipped; TrackingNumber = Some data.TrackingNumber }

    | OrderEvent.OrderDelivered _ ->
        { state with Status = Delivered }

    | OrderEvent.OrderCancelled _ ->
        { state with Status = Cancelled }

let replayOrder (events: OrderEvent list) : OrderState =
    events |> List.fold applyOrderEvent emptyOrder
```

---

## 3. Event Store (การเก็บ Events)

```fsharp
// EventStore.fs
module EventStore

open System
open System.Collections.Generic
open EventTypes

// ========================================
// 3.1 Event store interface
// ========================================

type StoredEvent = {
    StreamId: string
    EventId: Guid
    EventType: string
    EventData: string  // JSON serialized
    Metadata: string   // JSON serialized metadata
    Version: int64
    Timestamp: DateTime
}

type AppendResult =
    | Appended of int64  // new version
    | WrongExpectedVersion of int64 * int64  // expected, actual

[<Interface>]
type IEventStore =
    abstract member AppendAsync: string -> int64 -> string list -> System.Threading.Tasks.Task<AppendResult>
    abstract member LoadAsync: string -> System.Threading.Tasks.Task<StoredEvent list>
    abstract member LoadFromAsync: string -> int64 -> System.Threading.Tasks.Task<StoredEvent list>
    abstract member LoadToAsync: string -> int64 -> System.Threading.Tasks.Task<StoredEvent list>

// ========================================
// 3.2 In-memory Event Store
// ========================================

type InMemoryEventStore() =
    let streams = Dictionary<string, ResizeArray<StoredEvent>>()
    let mutable globalPosition = 0L

    let getOrCreateStream streamId =
        if not (streams.ContainsKey(streamId)) then
            streams.[streamId] <- ResizeArray()
        streams.[streamId]

    interface IEventStore with
        member _.AppendAsync streamId expectedVersion eventJsonList = task {
            let stream = getOrCreateStream streamId
            let currentVersion = if stream.Count = 0 then -1L else stream.[stream.Count - 1].Version

            if expectedVersion <> -2L && currentVersion <> expectedVersion then
                // -2L = any version (no conflict check)
                return WrongExpectedVersion(expectedVersion, currentVersion)
            else
                let mutable version = currentVersion
                for eventJson in eventJsonList do
                    version <- version + 1L
                    globalPosition <- globalPosition + 1L
                    stream.Add({
                        StreamId = streamId
                        EventId = Guid.NewGuid()
                        EventType = "Unknown"  // จะเอา type จาก JSON จริงๆ
                        EventData = eventJson
                        Metadata = "{}"
                        Version = version
                        Timestamp = DateTime.UtcNow
                    })
                return Appended version
        }

        member _.LoadAsync streamId = task {
            let stream = getOrCreateStream streamId
            return stream |> Seq.toList
        }

        member _.LoadFromAsync streamId fromVersion = task {
            let stream = getOrCreateStream streamId
            return stream |> Seq.filter (fun e -> e.Version >= fromVersion) |> Seq.toList
        }

        member _.LoadToAsync streamId toVersion = task {
            let stream = getOrCreateStream streamId
            return stream |> Seq.filter (fun e -> e.Version <= toVersion) |> Seq.toList
        }

    member _.GetAllStreams() = streams.Keys |> Seq.toList
    member _.GetEventCount(streamId: string) =
        if streams.ContainsKey(streamId) then streams.[streamId].Count
        else 0

// ========================================
// 3.3 Typed Event Store (with serialization)
// ========================================

open System.Text.Json

type TypedEventStore<'TEvent>(innerStore: IEventStore, streamPrefix: string) =
    let serializeEvent (event: 'TEvent) =
        let wrapper = {|
            Type = event.GetType().Name
            Data = JsonSerializer.Serialize(event)
        |}
        JsonSerializer.Serialize(wrapper)

    let deserializeEvent (json: string) =
        try
            JsonSerializer.Deserialize<'TEvent>(json) |> Some
        with _ ->
            None

    member _.AppendAsync (streamId: string) (expectedVersion: int64) (events: 'TEvent list) =
        task {
            let fullStreamId = $"{streamPrefix}-{streamId}"
            let jsonEvents = events |> List.map serializeEvent
            return! innerStore.AppendAsync fullStreamId expectedVersion jsonEvents
        }

    member _.LoadAsync (streamId: string) =
        task {
            let fullStreamId = $"{streamPrefix}-{streamId}"
            let! stored = innerStore.LoadAsync fullStreamId
            return
                stored
                |> List.choose (fun e -> deserializeEvent e.EventData)
        }

    member _.LoadFromVersionAsync (streamId: string) (fromVersion: int64) =
        task {
            let fullStreamId = $"{streamPrefix}-{streamId}"
            let! stored = innerStore.LoadFromAsync fullStreamId fromVersion
            return
                stored
                |> List.choose (fun e -> deserializeEvent e.EventData)
        }
```

---

## 4. Aggregates with Event Store

```fsharp
// BankAccountAggregate.fs
module BankAccountAggregate

open System
open EventTypes
open Aggregates
open EventStore

// ========================================
// 4.1 Commands
// ========================================

type OpenAccountCommand = {
    AccountId: string
    OwnerId: string
    OwnerName: string
    InitialBalance: decimal
    AccountType: string
}

type DepositCommand = {
    AccountId: string
    Amount: decimal
    Description: string
}

type WithdrawCommand = {
    AccountId: string
    Amount: decimal
    Description: string
}

type CloseAccountCommand = {
    AccountId: string
    Reason: string
}

type FreezeAccountCommand = {
    AccountId: string
    Reason: string
}

// ========================================
// 4.2 Business logic (Command -> Events)
// ========================================

type CommandError =
    | AccountNotFound of string
    | InsufficientFunds of decimal * decimal
    | AccountAlreadyClosed
    | AccountFrozenError
    | ValidationError of string

let openAccount (cmd: OpenAccountCommand) (currentState: BankAccount) : Result<BankAccountEvent list, CommandError> =
    if cmd.InitialBalance < 0m then
        Error (ValidationError "Initial balance cannot be negative")
    elif currentState.AccountId <> "" then
        Error (ValidationError "Account already exists")
    else
        Ok [
            BankAccountEvent.AccountOpened {
                AccountId = cmd.AccountId
                OwnerId = cmd.OwnerId
                OwnerName = cmd.OwnerName
                InitialBalance = cmd.InitialBalance
                AccountType = cmd.AccountType
                OpenedAt = DateTime.UtcNow
            }
        ]

let deposit (cmd: DepositCommand) (currentState: BankAccount) : Result<BankAccountEvent list, CommandError> =
    if cmd.Amount <= 0m then
        Error (ValidationError "Deposit amount must be positive")
    elif currentState.Status = Closed then
        Error AccountAlreadyClosed
    elif currentState.Status = Frozen then
        Error AccountFrozenError
    else
        Ok [
            BankAccountEvent.MoneyDeposited {
                AccountId = cmd.AccountId
                Amount = cmd.Amount
                Description = cmd.Description
                DepositedAt = DateTime.UtcNow
            }
        ]

let withdraw (cmd: WithdrawCommand) (currentState: BankAccount) : Result<BankAccountEvent list, CommandError> =
    if cmd.Amount <= 0m then
        Error (ValidationError "Withdrawal amount must be positive")
    elif currentState.Status = Closed then
        Error AccountAlreadyClosed
    elif currentState.Status = Frozen then
        Error AccountFrozenError
    elif currentState.Balance < cmd.Amount then
        Error (InsufficientFunds (currentState.Balance, cmd.Amount))
    else
        Ok [
            BankAccountEvent.MoneyWithdrawn {
                AccountId = cmd.AccountId
                Amount = cmd.Amount
                Description = cmd.Description
                WithdrawnAt = DateTime.UtcNow
            }
        ]

let closeAccount (cmd: CloseAccountCommand) (currentState: BankAccount) : Result<BankAccountEvent list, CommandError> =
    if currentState.Status = Closed then
        Error AccountAlreadyClosed
    else
        Ok [
            BankAccountEvent.AccountClosed {
                AccountId = cmd.AccountId
                Reason = cmd.Reason
                FinalBalance = currentState.Balance
                ClosedAt = DateTime.UtcNow
            }
        ]

// ========================================
// 4.3 Aggregate service (Load, Execute, Save)
// ========================================

type BankAccountService(eventStore: TypedEventStore<BankAccountEvent>) =
    /// Load current state จาก event store
    member _.LoadAsync accountId = task {
        let! events = eventStore.LoadAsync accountId
        return replayBankAccount events
    }

    /// Execute command
    member this.ExecuteAsync<'TCommand> accountId (execute: BankAccount -> Result<BankAccountEvent list, CommandError>) = task {
        let! currentState = this.LoadAsync accountId
        match execute currentState with
        | Error err ->
            return Error err
        | Ok newEvents ->
            let! result = eventStore.AppendAsync accountId -2L newEvents  // -2L = no version check
            match result with
            | Appended _ ->
                return Ok (replayBankAccount (List.append [] newEvents))  // Simplified
            | WrongExpectedVersion(expected, actual) ->
                return Error (ValidationError $"Concurrency conflict: expected version {expected}, got {actual}")
    }

    member this.OpenAccount (cmd: OpenAccountCommand) = task {
        return! this.ExecuteAsync cmd.AccountId (openAccount cmd)
    }

    member this.Deposit (cmd: DepositCommand) = task {
        return! this.ExecuteAsync cmd.AccountId (deposit cmd)
    }

    member this.Withdraw (cmd: WithdrawCommand) = task {
        return! this.ExecuteAsync cmd.AccountId (withdraw cmd)
    }

    member this.CloseAccount (cmd: CloseAccountCommand) = task {
        return! this.ExecuteAsync cmd.AccountId (closeAccount cmd)
    }
```

---

## 5. Projections (การสร้าง Read Models)

```fsharp
// Projections.fs
module Projections

open System
open System.Collections.Generic
open EventTypes
open Aggregates

// ========================================
// 5.1 Account Summary Projection
// ========================================

type AccountSummary = {
    AccountId: string
    OwnerName: string
    Balance: decimal
    Status: string
    TransactionCount: int
    LastTransactionAt: DateTime option
}

type AccountSummaryProjection() =
    let summaries = Dictionary<string, AccountSummary>()

    member _.HandleEvent (event: BankAccountEvent) =
        match event with
        | BankAccountEvent.AccountOpened data ->
            summaries.[data.AccountId] <- {
                AccountId = data.AccountId
                OwnerName = data.OwnerName
                Balance = data.InitialBalance
                Status = "Active"
                TransactionCount = 0
                LastTransactionAt = None
            }

        | BankAccountEvent.MoneyDeposited data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- {
                    s with
                        Balance = s.Balance + data.Amount
                        TransactionCount = s.TransactionCount + 1
                        LastTransactionAt = Some data.DepositedAt
                }
            | _ -> ()

        | BankAccountEvent.MoneyWithdrawn data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- {
                    s with
                        Balance = s.Balance - data.Amount
                        TransactionCount = s.TransactionCount + 1
                        LastTransactionAt = Some data.WithdrawnAt
                }
            | _ -> ()

        | BankAccountEvent.AccountClosed data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- { s with Status = "Closed"; Balance = data.FinalBalance }
            | _ -> ()

        | BankAccountEvent.AccountFrozen data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- { s with Status = "Frozen" }
            | _ -> ()

        | BankAccountEvent.AccountUnfrozen data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- { s with Status = "Active" }
            | _ -> ()

        | BankAccountEvent.InterestApplied data ->
            match summaries.TryGetValue(data.AccountId) with
            | true, s ->
                summaries.[data.AccountId] <- { s with Balance = s.Balance + data.InterestAmount }
            | _ -> ()

    member _.GetAll() = summaries.Values |> Seq.toList
    member _.GetById id =
        match summaries.TryGetValue(id) with
        | true, s -> Some s
        | _ -> None

// ========================================
// 5.2 Transaction History Projection
// ========================================

type TransactionType = Deposit | Withdrawal | Interest | OpeningBalance

type Transaction = {
    AccountId: string
    Type: TransactionType
    Amount: decimal
    Description: string
    OccurredAt: DateTime
    BalanceAfter: decimal
}

type TransactionHistoryProjection() =
    let histories = Dictionary<string, ResizeArray<Transaction>>()
    let runningBalances = Dictionary<string, decimal>()

    let getOrCreate (accountId: string) =
        if not (histories.ContainsKey(accountId)) then
            histories.[accountId] <- ResizeArray()
        histories.[accountId]

    member _.HandleEvent (event: BankAccountEvent) =
        match event with
        | BankAccountEvent.AccountOpened data ->
            runningBalances.[data.AccountId] <- data.InitialBalance
            let history = getOrCreate data.AccountId
            history.Add({
                AccountId = data.AccountId
                Type = OpeningBalance
                Amount = data.InitialBalance
                Description = "Account opened"
                OccurredAt = data.OpenedAt
                BalanceAfter = data.InitialBalance
            })

        | BankAccountEvent.MoneyDeposited data ->
            let balance = runningBalances.GetValueOrDefault(data.AccountId, 0m)
            let newBalance = balance + data.Amount
            runningBalances.[data.AccountId] <- newBalance
            let history = getOrCreate data.AccountId
            history.Add({
                AccountId = data.AccountId
                Type = Deposit
                Amount = data.Amount
                Description = data.Description
                OccurredAt = data.DepositedAt
                BalanceAfter = newBalance
            })

        | BankAccountEvent.MoneyWithdrawn data ->
            let balance = runningBalances.GetValueOrDefault(data.AccountId, 0m)
            let newBalance = balance - data.Amount
            runningBalances.[data.AccountId] <- newBalance
            let history = getOrCreate data.AccountId
            history.Add({
                AccountId = data.AccountId
                Type = Withdrawal
                Amount = -data.Amount
                Description = data.Description
                OccurredAt = data.WithdrawnAt
                BalanceAfter = newBalance
            })

        | BankAccountEvent.InterestApplied data ->
            let balance = runningBalances.GetValueOrDefault(data.AccountId, 0m)
            let newBalance = balance + data.InterestAmount
            runningBalances.[data.AccountId] <- newBalance
            let history = getOrCreate data.AccountId
            history.Add({
                AccountId = data.AccountId
                Type = Interest
                Amount = data.InterestAmount
                Description = $"Interest at {data.InterestRate}%%"
                OccurredAt = data.AppliedAt
                BalanceAfter = newBalance
            })

        | _ -> ()  // Ignore other events

    member _.GetHistory accountId =
        match histories.TryGetValue(accountId) with
        | true, h -> h |> Seq.toList
        | _ -> []

// ========================================
// 5.3 Statistics Projection
// ========================================

type BankStats = {
    mutable TotalAccounts: int
    mutable ActiveAccounts: int
    mutable TotalDeposits: decimal
    mutable TotalWithdrawals: decimal
    mutable TotalInterestPaid: decimal
}

type StatisticsProjection() =
    let stats = {
        TotalAccounts = 0
        ActiveAccounts = 0
        TotalDeposits = 0m
        TotalWithdrawals = 0m
        TotalInterestPaid = 0m
    }

    member _.HandleEvent (event: BankAccountEvent) =
        match event with
        | BankAccountEvent.AccountOpened _ ->
            stats.TotalAccounts <- stats.TotalAccounts + 1
            stats.ActiveAccounts <- stats.ActiveAccounts + 1
            stats.TotalDeposits <- stats.TotalDeposits  // Opening balance not counted as deposit

        | BankAccountEvent.MoneyDeposited data ->
            stats.TotalDeposits <- stats.TotalDeposits + data.Amount

        | BankAccountEvent.MoneyWithdrawn data ->
            stats.TotalWithdrawals <- stats.TotalWithdrawals + data.Amount

        | BankAccountEvent.InterestApplied data ->
            stats.TotalInterestPaid <- stats.TotalInterestPaid + data.InterestAmount

        | BankAccountEvent.AccountClosed _ ->
            stats.ActiveAccounts <- stats.ActiveAccounts - 1

        | _ -> ()

    member _.GetStats() = {|
        TotalAccounts = stats.TotalAccounts
        ActiveAccounts = stats.ActiveAccounts
        TotalDeposits = stats.TotalDeposits
        TotalWithdrawals = stats.TotalWithdrawals
        TotalInterestPaid = stats.TotalInterestPaid
        NetFlow = stats.TotalDeposits - stats.TotalWithdrawals
    |}
```

---

## 6. Snapshots (การบันทึก State ล่วงหน้า)

```fsharp
// Snapshots.fs
module Snapshots

open System
open EventTypes
open Aggregates

// ========================================
// 6.1 Snapshot types
// ========================================

type Snapshot<'TState> = {
    StreamId: string
    Version: int64
    State: 'TState
    TakenAt: DateTime
}

// ========================================
// 6.2 Snapshot store
// ========================================

type ISnapshotStore<'TState> =
    abstract member SaveAsync: Snapshot<'TState> -> System.Threading.Tasks.Task<unit>
    abstract member LoadAsync: string -> System.Threading.Tasks.Task<Snapshot<'TState> option>

type InMemorySnapshotStore<'TState>() =
    let snapshots = System.Collections.Generic.Dictionary<string, Snapshot<'TState>>()

    interface ISnapshotStore<'TState> with
        member _.SaveAsync snapshot = task {
            snapshots.[snapshot.StreamId] <- snapshot
        }

        member _.LoadAsync streamId = task {
            return
                match snapshots.TryGetValue(streamId) with
                | true, s -> Some s
                | _ -> None
        }

// ========================================
// 6.3 Snapshot-aware aggregate service
// ========================================

let snapshotInterval = 50  // Take snapshot every 50 events

type SnapshotBankAccountService
    (eventStore: EventStore.TypedEventStore<BankAccountEvent>,
     snapshotStore: ISnapshotStore<BankAccount>) =

    member _.LoadAsync accountId = task {
        // 1. Load snapshot ล่าสุด
        let! snapshot = snapshotStore.LoadAsync accountId

        // 2. Load events หลัง snapshot (ถ้ามี)
        let (startVersion, baseState) =
            match snapshot with
            | Some s -> s.Version + 1L, s.State
            | None -> 0L, emptyBankAccount

        let! recentEvents = eventStore.LoadFromVersionAsync accountId startVersion

        // 3. Apply recent events onto snapshot state
        let currentState = recentEvents |> List.fold applyBankAccountEvent baseState

        return currentState, recentEvents.Length
    }

    member this.ExecuteAsync accountId execute = task {
        let! (currentState, recentEventCount) = this.LoadAsync accountId

        match execute currentState with
        | Error err -> return Error err
        | Ok newEvents ->
            let! result = eventStore.AppendAsync accountId -2L newEvents
            match result with
            | EventStore.Appended newVersion ->
                let newState = newEvents |> List.fold applyBankAccountEvent currentState

                // Take snapshot if needed
                if recentEventCount + newEvents.Length >= snapshotInterval then
                    do! snapshotStore.SaveAsync {
                        StreamId = accountId
                        Version = newVersion
                        State = newState
                        TakenAt = DateTime.UtcNow
                    }
                    printfn "[Snapshot] Taken for %s at version %d" accountId newVersion

                return Ok newState
            | EventStore.WrongExpectedVersion(e, a) ->
                return Error (BankAccountAggregate.ValidationError $"Concurrency conflict: expected {e}, got {a}")
    }
```

---

## 7. Event Versioning (การจัดการ Version)

```fsharp
// EventVersioning.fs
module EventVersioning

open System
open System.Text.Json

// ========================================
// 7.1 Event versioning strategy
// ========================================

// Version 1 (original)
type MoneyDepositedV1 = {
    AccountId: string
    Amount: decimal
}

// Version 2 (added Description and Timestamp)
type MoneyDepositedV2 = {
    AccountId: string
    Amount: decimal
    Description: string
    DepositedAt: DateTime
}

// Upcaster: V1 -> V2
let upcaseMoneyDepositedV1ToV2 (v1: MoneyDepositedV1) : MoneyDepositedV2 = {
    AccountId = v1.AccountId
    Amount = v1.Amount
    Description = "Legacy deposit"  // default value
    DepositedAt = DateTime.UtcNow  // approximate time
}

// ========================================
// 7.2 Versioned event serialization
// ========================================

type VersionedEvent = {
    EventType: string
    Version: int
    Data: JsonElement
}

let deserializeWithUpcast (json: string) =
    try
        let versioned = JsonSerializer.Deserialize<VersionedEvent>(json)
        match versioned.EventType, versioned.Version with
        | "MoneyDeposited", 1 ->
            let v1 = JsonSerializer.Deserialize<MoneyDepositedV1>(versioned.Data.GetRawText())
            Some (upcaseMoneyDepositedV1ToV2 v1)
        | "MoneyDeposited", 2 ->
            JsonSerializer.Deserialize<MoneyDepositedV2>(versioned.Data.GetRawText()) |> Some
        | _ ->
            None
    with _ ->
        None

// ========================================
// 7.3 Best practices สำหรับ event versioning
// ========================================

(*
Rules for backwards-compatible changes:
1. Adding new optional fields (with defaults) - SAFE
2. Renaming fields - NEEDS UPCASTER
3. Changing field types - NEEDS UPCASTER
4. Removing fields - NEEDS UPCASTER
5. Adding new event types - SAFE
6. Removing event types - NEEDS MIGRATION

Strategy:
- เก็บ version number ใน event metadata
- สร้าง upcaster สำหรับแต่ละ version
- Chain upcasters: V1 -> V2 -> V3 -> ...
*)
```

---

## 8. Complete Example

```fsharp
// Program.fs
module Program

open System
open EventTypes
open EventStore
open Aggregates
open Projections

[<EntryPoint>]
let main _ =
    task {
        printfn "=== Event Sourcing F# Demo ==="
        printfn "=============================="

        // ========================================
        // Setup
        // ========================================
        let innerStore = InMemoryEventStore()
        let accountEventStore = TypedEventStore<BankAccountEvent>(innerStore, "account")
        let snapshotStore = Snapshots.InMemorySnapshotStore<BankAccount>()
        let service = Snapshots.SnapshotBankAccountService(accountEventStore, snapshotStore)

        // Projections
        let summaryProjection = AccountSummaryProjection()
        let historyProjection = TransactionHistoryProjection()
        let statsProjection = StatisticsProjection()

        let applyToProjections event =
            summaryProjection.HandleEvent event
            historyProjection.HandleEvent event
            statsProjection.HandleEvent event

        // ========================================
        // Open accounts
        // ========================================
        printfn "\n--- Opening Accounts ---"

        let accounts = [
            "ACC001", "Alice Johnson", 10000m
            "ACC002", "Bob Smith", 5000m
            "ACC003", "Charlie Brown", 25000m
        ]

        for (accountId, ownerName, initialBalance) in accounts do
            let openEvent = BankAccountEvent.AccountOpened {
                AccountId = accountId
                OwnerId = $"USER_{accountId}"
                OwnerName = ownerName
                InitialBalance = initialBalance
                AccountType = "Savings"
                OpenedAt = DateTime.UtcNow
            }
            let! _ = accountEventStore.AppendAsync accountId -2L [openEvent]
            applyToProjections openEvent
            printfn "  Opened account %s for %s with ฿%.2f" accountId ownerName initialBalance

        // ========================================
        // Transactions
        // ========================================
        printfn "\n--- Transactions ---"

        let transactions = [
            "ACC001", BankAccountEvent.MoneyDeposited { AccountId = "ACC001"; Amount = 5000m; Description = "Salary"; DepositedAt = DateTime.UtcNow }
            "ACC001", BankAccountEvent.MoneyWithdrawn { AccountId = "ACC001"; Amount = 2000m; Description = "Rent"; WithdrawnAt = DateTime.UtcNow }
            "ACC002", BankAccountEvent.MoneyDeposited { AccountId = "ACC002"; Amount = 3000m; Description = "Freelance"; DepositedAt = DateTime.UtcNow }
            "ACC002", BankAccountEvent.MoneyWithdrawn { AccountId = "ACC002"; Amount = 500m; Description = "Food"; WithdrawnAt = DateTime.UtcNow }
            "ACC003", BankAccountEvent.InterestApplied { AccountId = "ACC003"; InterestRate = 2.5m; InterestAmount = 625m; AppliedAt = DateTime.UtcNow }
        ]

        for (accountId, event) in transactions do
            let! _ = accountEventStore.AppendAsync accountId -2L [event]
            applyToProjections event
            match event with
            | BankAccountEvent.MoneyDeposited d -> printfn "  %s: +฿%.2f (%s)" accountId d.Amount d.Description
            | BankAccountEvent.MoneyWithdrawn d -> printfn "  %s: -฿%.2f (%s)" accountId d.Amount d.Description
            | BankAccountEvent.InterestApplied d -> printfn "  %s: Interest +฿%.2f (%.1f%%)" accountId d.InterestAmount (float d.InterestRate)
            | _ -> ()

        // ========================================
        // Read from projections (no need to replay)
        // ========================================
        printfn "\n--- Account Summaries (from Projection) ---"
        for summary in summaryProjection.GetAll() do
            printfn "  [%s] %s: ฿%.2f (%s) - %d transactions"
                summary.AccountId summary.OwnerName summary.Balance summary.Status summary.TransactionCount

        // ========================================
        // Transaction History
        // ========================================
        printfn "\n--- Transaction History for ACC001 ---"
        let history = historyProjection.GetHistory "ACC001"
        for tx in history do
            let sign = if tx.Amount >= 0m then "+" else ""
            printfn "  %s%A | %s฿%.2f | Balance: ฿%.2f"
                (tx.OccurredAt.ToString("HH:mm")) tx.Type sign tx.Amount tx.BalanceAfter

        // ========================================
        // Replay state from events (time travel)
        // ========================================
        printfn "\n--- Time Travel: Replay ACC001 ---"

        let! allEvents = accountEventStore.LoadAsync "ACC001"
        printfn "Total events for ACC001: %d" allEvents.Length

        // State after each event
        let mutable replayState = emptyBankAccount
        for i, event in allEvents |> List.mapi (fun i e -> i, e) do
            replayState <- applyBankAccountEvent replayState event
            printfn "  After event %d: Balance = ฿%.2f" (i + 1) replayState.Balance

        // ========================================
        // Statistics
        // ========================================
        printfn "\n--- Bank Statistics ---"
        let stats = statsProjection.GetStats()
        printfn "  Total Accounts: %d" stats.TotalAccounts
        printfn "  Active Accounts: %d" stats.ActiveAccounts
        printfn "  Total Deposits: ฿%.2f" stats.TotalDeposits
        printfn "  Total Withdrawals: ฿%.2f" stats.TotalWithdrawals
        printfn "  Net Flow: ฿%.2f" stats.NetFlow
        printfn "  Interest Paid: ฿%.2f" stats.TotalInterestPaid

        // ========================================
        // Event sourcing benefits summary
        // ========================================
        printfn "\n--- Event Sourcing Benefits ---"
        let benefits = [
            "Complete audit trail - ทุก transaction ถูกบันทึก"
            "Time travel - ดู state ในอดีตได้"
            "Event replay - สร้าง read models ใหม่จาก events เดิมได้"
            "Debugging - รู้ว่าเกิดอะไรขึ้นทุกขั้นตอน"
            "Scalability - Read/Write แยกกันด้วย CQRS"
            "Snapshots - ลด replay time สำหรับ aggregates ที่มี events เยอะ"
        ]
        for b in benefits do
            printfn "  ✓ %s" b

        printfn "\n=== Demo Complete ==="
        return 0
    } |> Async.AwaitTask |> Async.RunSynchronously
```

---

## สรุป (Summary)

Event Sourcing เป็น pattern ที่ทรงพลังแต่มีความซับซ้อนสูง เหมาะกับ:

1. **Audit requirements**: ระบบที่ต้องการ audit trail 100%
2. **Complex business domains**: Domain ที่มี business logic ซับซ้อน
3. **CQRS**: ใช้ร่วมกันได้ดีมาก
4. **Event-driven**: ระบบที่ต้องการ react ต่อ events

ข้อควรระวัง:
- **Eventual consistency**: Read models อาจ lag หลัง write side
- **Complexity**: Learning curve สูง
- **Storage**: เก็บ events ทั้งหมด → storage มาก (ใช้ snapshots ช่วย)

```fsharp
// Key concepts:
// Event = fact ที่เกิดขึ้นแล้ว (immutable)
// Aggregate = entities ที่ apply events เพื่อได้ current state
// Event Store = append-only log ของ events
// Projection = read model สร้างจาก events
// Snapshot = cached state เพื่อลด replay time
// Upcaster = แปลง old event version เป็น new version
```
