# Part 102 - Elmish Architecture

## บทนำ (Introduction)

Elmish เป็น implementation ของ Elm Architecture (MVU - Model-View-Update) สำหรับ F# และ Fable มันเป็น pattern สำหรับจัดการ state ใน web applications โดย:

- **Model**: State ทั้งหมดของ application
- **View**: Function ที่ render UI จาก Model
- **Update**: Function ที่ handle messages และสร้าง Model ใหม่

---

## 1. MVU Pattern ในเชิงลึก

### แนวคิดหลัก

```
                  ┌──────────────┐
                  │              │
           ┌──────▼──────┐  ┌───┴──────────┐
   Msg ────►   Update    │  │    Model     │
           └──────┬──────┘  └───▲──────────┘
                  │             │
                  └──New Model──┘
                  
                  ┌──────────────┐
       Model ────►     View     ├──── HTML
                  └──────┬──────┘
                         │
                    User Actions
                         │
                       Msg
```

### ทำไมต้องใช้ MVU?

```fsharp
// ปัญหาของ Mutable State ทั่วไป
let mutable counter = 0
let mutable isLoading = false

// ไม่รู้ว่า state เปลี่ยนเมื่อไหร่ หรือเปลี่ยนจากที่ไหน
// ยากต่อการ debug และ test

// ===== MVU แก้ปัญหานี้ด้วย =====

// State ทั้งหมดอยู่ใน record เดียว
type Model = {
    Counter: int
    IsLoading: bool
}

// State เปลี่ยนได้แค่ผ่าน messages ที่กำหนดไว้
type Msg =
    | Increment
    | Decrement
    | SetLoading of bool

// ทุก state change ผ่าน update function เดียว → predictable, testable
let update msg model =
    match msg with
    | Increment -> { model with Counter = model.Counter + 1 }, Cmd.none
    | Decrement -> { model with Counter = model.Counter - 1 }, Cmd.none
    | SetLoading b -> { model with IsLoading = b }, Cmd.none
```

---

## 2. Model - State Management

### Simple Model

```fsharp
// Model ง่ายๆ
type Model = {
    Count: int
    Name: string
}

let init () =
    { Count = 0; Name = "Elmish" }, Cmd.none
```

### Complex Model

```fsharp
open System

// Types สำหรับ domain
type UserId = UserId of Guid
type ProductId = ProductId of Guid

type User = {
    Id: UserId
    Email: string
    DisplayName: string
    Role: UserRole
    CreatedAt: DateTime
}

and UserRole =
    | Admin
    | Manager
    | User

type Product = {
    Id: ProductId
    Name: string
    Price: decimal
    Stock: int
    Category: Category
}

and Category = {
    Id: int
    Name: string
}

type Order = {
    Id: Guid
    UserId: UserId
    Items: OrderItem list
    Status: OrderStatus
    CreatedAt: DateTime
    Total: decimal
}

and OrderItem = {
    ProductId: ProductId
    Quantity: int
    UnitPrice: decimal
}

and OrderStatus =
    | Pending
    | Processing
    | Shipped
    | Delivered
    | Cancelled

// Async state wrapper
type Deferred<'T> =
    | NotStarted
    | InProgress
    | Resolved of 'T

type AsyncResult<'T, 'E> = Deferred<Result<'T, 'E>>

// Complex Model
type DashboardModel = {
    // User state
    CurrentUser: User option
    AuthStatus: Deferred<Result<User, string>>
    
    // Data
    Products: AsyncResult<Product list, string>
    Orders: AsyncResult<Order list, string>
    
    // UI State
    SelectedOrder: Order option
    FilterText: string
    SortBy: SortField
    SortDirection: SortDirection
    CurrentPage: int
    PageSize: int
    
    // Modal state
    IsDeleteModalOpen: bool
    ItemToDelete: ProductId option
    
    // Notifications
    Notifications: Notification list
}

and SortField = ByName | ByPrice | ByDate | ByStatus
and SortDirection = Ascending | Descending

and Notification = {
    Id: Guid
    Message: string
    Type: NotificationType
    ExpiresAt: DateTime option
}

and NotificationType = Success | Warning | Error | Info

let initDashboard () =
    {
        CurrentUser = None
        AuthStatus = NotStarted
        Products = NotStarted
        Orders = NotStarted
        SelectedOrder = None
        FilterText = ""
        SortBy = ByDate
        SortDirection = Descending
        CurrentPage = 1
        PageSize = 20
        IsDeleteModalOpen = false
        ItemToDelete = None
        Notifications = []
    }, Cmd.batch [
        Cmd.ofMsg CheckAuth
        Cmd.ofMsg LoadProducts
        Cmd.ofMsg LoadOrders
    ]
```

---

## 3. Messages

### กำหนด Messages อย่างชัดเจน

```fsharp
type Msg =
    // Auth messages
    | CheckAuth
    | AuthChecked of Result<User, string>
    | Login of email: string * password: string
    | LoginResult of Result<User, string>
    | Logout
    
    // Product messages
    | LoadProducts
    | ProductsLoaded of Result<Product list, string>
    | SelectProduct of ProductId
    | OpenDeleteModal of ProductId
    | CloseDeleteModal
    | ConfirmDelete
    | DeleteProduct of ProductId
    | ProductDeleted of Result<unit, string>
    
    // Order messages
    | LoadOrders
    | OrdersLoaded of Result<Order list, string>
    | SelectOrder of Order
    | UpdateOrderStatus of Order * OrderStatus
    | OrderStatusUpdated of Result<Order, string>
    
    // UI messages
    | SetFilter of string
    | SetSort of SortField
    | SetPage of int
    | DismissNotification of Guid
    | AddNotification of Notification
    
    // Navigation
    | NavigateTo of Page

// Message helpers
let successNotification msg = 
    AddNotification {
        Id = System.Guid.NewGuid()
        Message = msg
        Type = Success
        ExpiresAt = Some (System.DateTime.Now.AddSeconds 5.0)
    }

let errorNotification msg =
    AddNotification {
        Id = System.Guid.NewGuid()
        Message = msg
        Type = Error
        ExpiresAt = None
    }
```

---

## 4. Update Function

### Pattern Matching ใน Update

```fsharp
let update msg model =
    match msg with
    // ========= Auth =========
    | CheckAuth ->
        { model with AuthStatus = InProgress },
        Cmd.ofAsync 
            AuthService.getCurrentUser 
            () 
            (AuthChecked << Ok)
            (AuthChecked << Error << fun e -> e.Message)
    
    | AuthChecked (Ok user) ->
        { model with 
            CurrentUser = Some user
            AuthStatus = Resolved (Ok user) }
        , Cmd.none
    
    | AuthChecked (Error msg) ->
        { model with 
            CurrentUser = None
            AuthStatus = Resolved (Error msg) }
        , Cmd.none
    
    | Login (email, password) ->
        model,
        Cmd.ofAsync
            (fun () -> AuthService.login email password)
            ()
            LoginResult
            (LoginResult << Error << fun e -> e.Message)
    
    | LoginResult (Ok user) ->
        { model with CurrentUser = Some user },
        Cmd.ofMsg (successNotification (sprintf "ยินดีต้อนรับ, %s!" user.DisplayName))
    
    | LoginResult (Error err) ->
        model,
        Cmd.ofMsg (errorNotification (sprintf "Login ล้มเหลว: %s" err))
    
    | Logout ->
        { model with CurrentUser = None },
        Cmd.batch [
            Cmd.ofAsync AuthService.logout () (fun _ -> NavigateTo LoginPage) ignore
        ]
    
    // ========= Products =========
    | LoadProducts ->
        { model with Products = InProgress },
        Cmd.ofAsync
            ProductService.getAll
            ()
            (ProductsLoaded << Ok)
            (ProductsLoaded << Error << fun e -> e.Message)
    
    | ProductsLoaded result ->
        { model with Products = Resolved result }, Cmd.none
    
    | OpenDeleteModal productId ->
        { model with 
            IsDeleteModalOpen = true
            ItemToDelete = Some productId }
        , Cmd.none
    
    | CloseDeleteModal ->
        { model with 
            IsDeleteModalOpen = false
            ItemToDelete = None }
        , Cmd.none
    
    | ConfirmDelete ->
        match model.ItemToDelete with
        | Some productId ->
            model,
            Cmd.batch [
                Cmd.ofMsg CloseDeleteModal
                Cmd.ofMsg (DeleteProduct productId)
            ]
        | None -> model, Cmd.none
    
    | DeleteProduct productId ->
        model,
        Cmd.ofAsync
            (fun () -> ProductService.delete productId)
            ()
            (ProductDeleted << Ok)
            (ProductDeleted << Error << fun e -> e.Message)
    
    | ProductDeleted (Ok ()) ->
        let updatedProducts =
            match model.Products with
            | Resolved (Ok products) ->
                Resolved (Ok (products |> List.filter (fun p -> Some p.Id <> model.ItemToDelete)))
            | other -> other
        
        { model with Products = updatedProducts },
        Cmd.ofMsg (successNotification "ลบสินค้าสำเร็จ")
    
    | ProductDeleted (Error err) ->
        model,
        Cmd.ofMsg (errorNotification (sprintf "ลบไม่ได้: %s" err))
    
    // ========= UI =========
    | SetFilter text ->
        { model with FilterText = text; CurrentPage = 1 }, Cmd.none
    
    | SetSort field ->
        let direction =
            if model.SortBy = field then
                match model.SortDirection with
                | Ascending -> Descending
                | Descending -> Ascending
            else
                Ascending
        { model with SortBy = field; SortDirection = direction }, Cmd.none
    
    | SetPage page ->
        { model with CurrentPage = page }, Cmd.none
    
    | DismissNotification id ->
        { model with 
            Notifications = model.Notifications |> List.filter (fun n -> n.Id <> id) }
        , Cmd.none
    
    | AddNotification notification ->
        let newModel = 
            { model with Notifications = model.Notifications @ [notification] }
        
        match notification.ExpiresAt with
        | Some expiresAt ->
            let delay = max 0.0 (expiresAt - System.DateTime.Now).TotalMilliseconds
            newModel,
            Cmd.ofAsync
                (fun () -> async { do! Async.Sleep (int delay) })
                ()
                (fun _ -> DismissNotification notification.Id)
                ignore
        | None ->
            newModel, Cmd.none
    
    | _ -> model, Cmd.none
```

---

## 5. Commands (Cmd)

### Command Types

```fsharp
open Elmish

// === Cmd.none ===
// ไม่ทำอะไรหลัง update
let update1 msg model =
    model, Cmd.none

// === Cmd.ofMsg ===
// Dispatch message อื่นทันที
let update2 msg model =
    match msg with
    | TriggerChain ->
        model, Cmd.ofMsg DoNext
    | DoNext ->
        // ...
        model, Cmd.none

// === Cmd.ofAsync ===
// รัน async computation
let loadData () : Async<string> =
    async { return "data" }

let update3 msg model =
    match msg with
    | Load ->
        model, 
        Cmd.ofAsync loadData () DataLoaded (Error >> DataFailed)

// === Cmd.OfAsync.either ===
// รัน async ที่ return Result หรือ choice
let update4 msg model =
    match msg with
    | Load ->
        model,
        Cmd.OfAsync.either
            loadData
            ()
            (Ok >> DataResult)
            (Error >> DataResult)

// === Cmd.batch ===
// รัน commands หลายอัน
let update5 msg model =
    match msg with
    | Initialize ->
        model,
        Cmd.batch [
            Cmd.ofMsg LoadUsers
            Cmd.ofMsg LoadProducts
            Cmd.ofMsg StartTimer
        ]

// === Cmd.map ===
// Transform messages จาก sub-component
let parentUpdate msg model =
    match msg with
    | ChildMsg childMsg ->
        let newChild, childCmd = Child.update childMsg model.Child
        { model with Child = newChild },
        Cmd.map ChildMsg childCmd

// === Custom Command ===
// สร้าง command เอง
let delay (ms: int) (msg: 'msg) : Cmd<'msg> =
    let sub dispatch =
        async {
            do! Async.Sleep ms
            dispatch msg
        } |> Async.StartImmediate
    [ sub ]

// ใช้งาน
let update6 msg model =
    match msg with
    | ShowThenHide ->
        { model with Visible = true },
        delay 3000 Hide
```

---

## 6. Subscriptions

### External Events และ Subscriptions

```fsharp
open Elmish
open Browser.Dom
open Browser.Types

// Subscription สำหรับ keyboard events
let keyboardSubscription dispatch =
    let handler (e: KeyboardEvent) =
        match e.key with
        | "ArrowUp" -> dispatch (Move Up)
        | "ArrowDown" -> dispatch (Move Down)
        | "ArrowLeft" -> dispatch (Move Left)
        | "ArrowRight" -> dispatch (Move Right)
        | "Escape" -> dispatch Cancel
        | _ -> ()
    
    document.addEventListener("keydown", handler)
    
    // Return disposable
    { new System.IDisposable with
        member _.Dispose() =
            document.removeEventListener("keydown", handler) }

// Subscription สำหรับ window resize
let resizeSubscription dispatch =
    let handler _ =
        dispatch (WindowResized (window.innerWidth, window.innerHeight))
    
    window.addEventListener("resize", handler)
    
    { new System.IDisposable with
        member _.Dispose() =
            window.removeEventListener("resize", handler) }

// Timer subscription
let timerSubscription (intervalMs: int) dispatch =
    let timerId = 
        window.setInterval(
            (fun _ -> dispatch Tick),
            intervalMs
        )
    
    { new System.IDisposable with
        member _.Dispose() =
            window.clearInterval(timerId) }

// WebSocket subscription
let websocketSubscription (url: string) dispatch =
    let ws = Browser.WebSocket.WebSocket.Create(url)
    
    ws.onmessage <- fun event ->
        dispatch (MessageReceived (string event.data))
    
    ws.onerror <- fun _ ->
        dispatch WebSocketError
    
    ws.onclose <- fun _ ->
        dispatch WebSocketClosed
    
    { new System.IDisposable with
        member _.Dispose() ->
            ws.close() }

// รวม subscriptions
let subscribe model =
    [
        if model.KeyboardEnabled then
            yield keyboardSubscription
        
        if model.TrackWindowSize then
            yield resizeSubscription
        
        if model.TimerRunning then
            yield timerSubscription 1000
        
        match model.WebSocketUrl with
        | Some url -> yield websocketSubscription url
        | None -> ()
    ]

// ใช้ใน Program
Program.mkProgram init update view
|> Program.withSubscription subscribe
|> Program.withReactSynchronous "app"
|> Program.run
```

---

## 7. Side Effects

### จัดการ Side Effects อย่างถูกต้อง

```fsharp
// ===== Side Effects ผ่าน Commands =====

// HTTP calls
module ApiService =
    open Fetch
    
    let get<'T> (url: string) (decoder: Decoder<'T>) : Async<Result<'T, string>> =
        async {
            try
                let! response = fetch url [] |> Async.AwaitPromise
                if response.Ok then
                    let! text = response.text() |> Async.AwaitPromise
                    return Decode.fromString decoder text
                else
                    return Error (sprintf "HTTP %d" response.Status)
            with ex ->
                return Error ex.Message
        }

// Logging side effect
let logCommand (msg: Msg) : Cmd<Msg> =
    let sub dispatch =
        Browser.Dom.console.log(sprintf "[MSG] %A" msg)
    [ sub ]

// Analytics tracking
let trackEvent (eventName: string) (properties: obj) : Cmd<Msg> =
    let sub _ =
        // ส่ง event ไปยัง analytics service
        Browser.Dom.window?gtag("event", eventName, properties)
    [ sub ]

// Local storage side effect
let saveToStorage (key: string) (value: string) : Cmd<Msg> =
    let sub _ =
        Browser.WebStorage.localStorage.setItem(key, value)
    [ sub ]

// Combining side effects
let update msg model =
    match msg with
    | UserLoggedIn user ->
        let newModel = { model with CurrentUser = Some user }
        newModel,
        Cmd.batch [
            trackEvent "user_login" {| userId = user.Id; email = user.Email |}
            saveToStorage "auth_token" user.Token
            Cmd.ofMsg (LoadUserProfile user.Id)
            logCommand msg
        ]
```

---

## 8. Navigation

### Setup Routing กับ Elmish

```fsharp
open Feliz.Router
open Elmish

// Define pages
type Page =
    | HomePage
    | ProductsPage
    | ProductDetailPage of id: int
    | CartPage
    | CheckoutPage
    | OrdersPage
    | ProfilePage
    | LoginPage
    | NotFoundPage

// Parse URL
let parsePage = function
    | [] -> HomePage
    | [ "products" ] -> ProductsPage
    | [ "products"; Route.Int id ] -> ProductDetailPage id
    | [ "cart" ] -> CartPage
    | [ "checkout" ] -> CheckoutPage
    | [ "orders" ] -> OrdersPage
    | [ "profile" ] -> ProfilePage
    | [ "login" ] -> LoginPage
    | _ -> NotFoundPage

// Serialize Page เป็น URL
let pageToUrl = function
    | HomePage -> "/"
    | ProductsPage -> "/products"
    | ProductDetailPage id -> sprintf "/products/%d" id
    | CartPage -> "/cart"
    | CheckoutPage -> "/checkout"
    | OrdersPage -> "/orders"
    | ProfilePage -> "/profile"
    | LoginPage -> "/login"
    | NotFoundPage -> "/404"

// Model
type Model = {
    CurrentPage: Page
    PreviousPage: Page option
    NavHistory: Page list
    // ... other state
}

// Messages
type Msg =
    | NavigateTo of Page
    | NavigateBack
    | UrlChanged of string list
    // ... other messages

// Update
let update msg model =
    match msg with
    | NavigateTo page ->
        let history = model.CurrentPage :: model.NavHistory |> List.truncate 10
        { model with 
            PreviousPage = Some model.CurrentPage
            NavHistory = history }
        , Router.navigate (pageToUrl page)
    
    | NavigateBack ->
        match model.PreviousPage with
        | Some prev ->
            { model with 
                CurrentPage = prev
                PreviousPage = None }
            , Router.navigate (pageToUrl prev)
        | None ->
            model, Cmd.none
    
    | UrlChanged segments ->
        let page = parsePage segments
        { model with CurrentPage = page }, Cmd.none

// View
let view model dispatch =
    React.router [
        router.onUrlChanged (UrlChanged >> dispatch)
        router.children [
            Html.div [
                // Navigation bar
                navBar model dispatch
                
                // Page content
                match model.CurrentPage with
                | HomePage -> homePage model dispatch
                | ProductsPage -> productsPage model dispatch
                | ProductDetailPage id -> productDetailPage id model dispatch
                | CartPage -> cartPage model dispatch
                | CheckoutPage -> 
                    match model.CurrentUser with
                    | Some _ -> checkoutPage model dispatch
                    | None -> 
                        Html.div []  // redirect to login
                | OrdersPage -> ordersPage model dispatch
                | ProfilePage -> profilePage model dispatch
                | LoginPage -> loginPage model dispatch
                | NotFoundPage -> notFoundPage model dispatch
            ]
        ]
    ]
```

---

## 9. Nested Models

### Component Decomposition

```fsharp
// ===== Child Components =====

// Counter sub-component
module Counter =
    type Model = { Count: int; Step: int }
    type Msg = | Increment | Decrement | SetStep of int
    
    let init step = { Count = 0; Step = step }
    
    let update msg model =
        match msg with
        | Increment -> { model with Count = model.Count + model.Step }
        | Decrement -> { model with Count = model.Count - model.Step }
        | SetStep s -> { model with Step = s }
    
    let view model dispatch =
        Html.div [
            Html.button [
                prop.text "-"
                prop.onClick (fun _ -> dispatch Decrement)
            ]
            Html.span (string model.Count)
            Html.button [
                prop.text "+"
                prop.onClick (fun _ -> dispatch Increment)
            ]
        ]

// Search sub-component
module Search =
    type Model = {
        Query: string
        Suggestions: string list
        IsOpen: bool
    }
    
    type Msg =
        | UpdateQuery of string
        | SelectSuggestion of string
        | OpenDropdown
        | CloseDropdown
        | FetchSuggestions of string
        | SuggestionsLoaded of string list
    
    let init () = { Query = ""; Suggestions = []; IsOpen = false }
    
    let update msg model =
        match msg with
        | UpdateQuery q ->
            { model with Query = q }, Cmd.ofMsg (FetchSuggestions q)
        | SelectSuggestion s ->
            { model with Query = s; IsOpen = false }, Cmd.none
        | OpenDropdown -> { model with IsOpen = true }, Cmd.none
        | CloseDropdown -> { model with IsOpen = false }, Cmd.none
        | FetchSuggestions q ->
            model, Cmd.ofAsync
                (fun () -> SearchService.getSuggestions q)
                ()
                SuggestionsLoaded
                (fun _ -> SuggestionsLoaded [])
        | SuggestionsLoaded s ->
            { model with Suggestions = s; IsOpen = s.Length > 0 }, Cmd.none
    
    let view model dispatch =
        Html.div [
            Html.input [
                prop.value model.Query
                prop.onChange (fun e -> dispatch (UpdateQuery e.target.value))
                prop.onFocus (fun _ -> dispatch OpenDropdown)
            ]
            if model.IsOpen then
                Html.ul [
                    for s in model.Suggestions do
                        Html.li [
                            prop.text s
                            prop.onClick (fun _ -> dispatch (SelectSuggestion s))
                        ]
                ]
        ]

// ===== Parent Component =====

module App =
    type Model = {
        Counter: Counter.Model
        Search: Search.Model
        // ...
    }
    
    type Msg =
        | CounterMsg of Counter.Msg
        | SearchMsg of Search.Msg
        // ...
    
    let init () =
        {
            Counter = Counter.init 1
            Search = Search.init()
        }, Cmd.none
    
    let update msg model =
        match msg with
        | CounterMsg counterMsg ->
            let newCounter = Counter.update counterMsg model.Counter
            { model with Counter = newCounter }, Cmd.none
        
        | SearchMsg searchMsg ->
            let newSearch, searchCmd = Search.update searchMsg model.Search
            { model with Search = newSearch },
            Cmd.map SearchMsg searchCmd
    
    let view model dispatch =
        Html.div [
            Counter.view model.Counter (CounterMsg >> dispatch)
            Search.view model.Search (SearchMsg >> dispatch)
        ]
```

---

## 10. Performance Optimization

### Avoiding Unnecessary Re-renders

```fsharp
open Feliz

// ใช้ React.memo เพื่อ prevent unnecessary re-renders
let expensiveComponent = React.memo(fun (props: {| data: Data list |}) ->
    // Component นี้จะ re-render เฉพาะเมื่อ props เปลี่ยน
    Html.div [
        for item in props.data do
            Html.div [
                prop.key item.Id
                prop.text item.Name
            ]
    ]
, (fun prev next -> prev.data = next.data))

// ใช้ useCallback สำหรับ memoized event handlers
let optimizedList = React.functionComponent(fun (props: {| items: Item list; onSelect: Item -> unit |}) ->
    let handleSelect = React.useCallback(
        (fun item -> props.onSelect item),
        [| props.onSelect |]
    )
    
    Html.ul [
        for item in props.items do
            Html.li [
                prop.key item.Id
                prop.onClick (fun _ -> handleSelect item)
                prop.text item.Name
            ]
    ]
)

// ใช้ useMemo สำหรับ expensive computations
let computeStats = React.functionComponent(fun (props: {| orders: Order list |}) ->
    let stats = React.useMemo(
        (fun () ->
            {|
                total = props.orders |> List.sumBy (fun o -> o.Total)
                count = props.orders.Length
                average = 
                    if props.orders.IsEmpty then 0m
                    else (props.orders |> List.sumBy (fun o -> o.Total)) / decimal props.orders.Length
            |}),
        [| props.orders |]
    )
    
    Html.div [
        Html.p (sprintf "Total: %.2f" stats.total)
        Html.p (sprintf "Count: %d" stats.count)
        Html.p (sprintf "Average: %.2f" stats.average)
    ]
)

// Virtual scrolling สำหรับ large lists
let virtualList = React.functionComponent(fun (props: {| items: Item list |}) ->
    let containerRef = React.useRef(None)
    let scrollTop, setScrollTop = React.useState(0)
    let itemHeight = 50
    let containerHeight = 500
    
    let visibleStart = scrollTop / itemHeight
    let visibleEnd = (scrollTop + containerHeight) / itemHeight + 1
    let visibleItems = 
        props.items 
        |> List.skip (min visibleStart props.items.Length)
        |> List.truncate (visibleEnd - visibleStart)
    
    Html.div [
        prop.ref containerRef
        prop.style [
            style.height containerHeight
            style.overflow.auto
        ]
        prop.onScroll (fun e -> setScrollTop (int e.currentTarget?scrollTop))
        prop.children [
            // Spacer for items above
            Html.div [
                prop.style [style.height (visibleStart * itemHeight)]
            ]
            
            // Visible items
            for item in visibleItems do
                Html.div [
                    prop.key item.Id
                    prop.style [style.height itemHeight]
                    prop.text item.Name
                ]
            
            // Spacer for items below
            Html.div [
                prop.style [
                    style.height ((props.items.Length - visibleEnd) * itemHeight)
                ]
            ]
        ]
    ]
)
```

---

## 11. Testing Elmish Components

### Unit Testing Models และ Update Functions

```fsharp
// Tests.fs
module Tests

open Xunit
open FsUnit.Xunit

// ทดสอบ init function
[<Fact>]
let ``init returns empty model`` () =
    let model, cmd = Counter.init()
    model.Count |> should equal 0
    model.Step |> should equal 1

// ทดสอบ update function
[<Fact>]
let ``Increment increases count by step`` () =
    let model = { Count = 5; Step = 2 }
    let newModel, _ = Counter.update Counter.Increment model
    newModel.Count |> should equal 7

[<Fact>]
let ``Decrement decreases count by step`` () =
    let model = { Count = 10; Step = 3 }
    let newModel, _ = Counter.update Counter.Decrement model
    newModel.Count |> should equal 7

// ทดสอบ commands
[<Fact>]
let ``LoadUsers command dispatches correct messages`` () =
    let dispatchedMessages = System.Collections.Generic.List<Msg>()
    
    let model = { Users = NotStarted }
    let newModel, cmd = update LoadUsers model
    
    // Execute command
    cmd |> List.iter (fun sub -> sub (fun msg -> dispatchedMessages.Add(msg)))
    
    newModel.Users |> should equal InProgress

// Property-based testing
open FsCheck
open FsCheck.Xunit

[<Property>]
let ``Count never goes below zero with non-negative steps`` (steps: PositiveInt) =
    let model = { Count = 0; Step = steps.Get }
    let newModel, _ = Counter.update Counter.Decrement model
    newModel.Count >= 0  // ถ้า logic ป้องกัน negative

// Testing with Elmish testing helpers
open Elmish.Testing

[<Fact>]
let ``Full flow test`` () =
    let program = 
        Program.mkProgram Counter.init Counter.update Counter.view
        |> Program.toModel
    
    let initialModel = program.Model
    
    let afterIncrement = 
        program |> Program.dispatch Counter.Increment |> _.Model
    
    afterIncrement.Count |> should equal (initialModel.Count + 1)
```

---

## 12. Real Application Example

### Full E-commerce App

```fsharp
// ShopApp.fs - Complete E-commerce with Elmish
module ShopApp

open Elmish
open Feliz

// ===== Types =====

type ProductId = int
type CartItemId = int

type Product = {
    Id: ProductId
    Name: string
    Price: decimal
    Description: string
    ImageUrl: string
    Stock: int
    Category: string
    Rating: float
    ReviewCount: int
}

type CartItem = {
    Id: CartItemId
    Product: Product
    Quantity: int
}

type Cart = {
    Items: CartItem list
    NextId: CartItemId
}

module Cart =
    let empty = { Items = []; NextId = 1 }
    
    let addItem (product: Product) (cart: Cart) =
        let existing = cart.Items |> List.tryFind (fun i -> i.Product.Id = product.Id)
        match existing with
        | Some item ->
            let updated = { item with Quantity = item.Quantity + 1 }
            { cart with Items = cart.Items |> List.map (fun i -> if i.Id = item.Id then updated else i) }
        | None ->
            let newItem = { Id = cart.NextId; Product = product; Quantity = 1 }
            { cart with Items = cart.Items @ [newItem]; NextId = cart.NextId + 1 }
    
    let removeItem (itemId: CartItemId) (cart: Cart) =
        { cart with Items = cart.Items |> List.filter (fun i -> i.Id <> itemId) }
    
    let updateQuantity (itemId: CartItemId) (qty: int) (cart: Cart) =
        if qty <= 0 then removeItem itemId cart
        else
            { cart with
                Items = cart.Items |> List.map (fun i ->
                    if i.Id = itemId then { i with Quantity = qty }
                    else i) }
    
    let total (cart: Cart) =
        cart.Items |> List.sumBy (fun i -> i.Product.Price * decimal i.Quantity)
    
    let itemCount (cart: Cart) =
        cart.Items |> List.sumBy (fun i -> i.Quantity)

// ===== Model =====

type ShopPage =
    | ProductListPage
    | ProductDetailPage of ProductId
    | CartPage
    | CheckoutPage
    | OrderConfirmationPage of orderId: string

type ShopModel = {
    Page: ShopPage
    Products: Product list
    LoadingProducts: bool
    SelectedProduct: Product option
    Cart: Cart
    SearchQuery: string
    SelectedCategory: string option
    SortBy: string
    Notification: (string * string) option  // (type, message)
}

let initShop () =
    {
        Page = ProductListPage
        Products = []
        LoadingProducts = true
        SelectedProduct = None
        Cart = Cart.empty
        SearchQuery = ""
        SelectedCategory = None
        SortBy = "name"
        Notification = None
    }, Cmd.ofMsg LoadProducts

// ===== Messages =====

type ShopMsg =
    | LoadProducts
    | ProductsLoaded of Product list
    | LoadFailed of string
    | NavigateToProduct of ProductId
    | NavigateBack
    | NavigateToCart
    | NavigateToCheckout
    | AddToCart of Product
    | RemoveFromCart of CartItemId
    | UpdateCartItemQty of CartItemId * int
    | ClearCart
    | PlaceOrder
    | OrderPlaced of orderId: string
    | SetSearchQuery of string
    | SetCategory of string option
    | SetSort of string
    | DismissNotification

// ===== Mock Data =====

let mockProducts = [
    { Id = 1; Name = "MacBook Pro"; Price = 59900m; Description = "Apple MacBook Pro 14-inch"; ImageUrl = "/images/macbook.jpg"; Stock = 10; Category = "Computers"; Rating = 4.8; ReviewCount = 256 }
    { Id = 2; Name = "iPhone 15"; Price = 32900m; Description = "Apple iPhone 15 Pro Max"; ImageUrl = "/images/iphone.jpg"; Stock = 25; Category = "Phones"; Rating = 4.9; ReviewCount = 512 }
    { Id = 3; Name = "AirPods Pro"; Price = 8900m; Description = "Apple AirPods Pro (2nd gen)"; ImageUrl = "/images/airpods.jpg"; Stock = 50; Category = "Audio"; Rating = 4.7; ReviewCount = 128 }
    { Id = 4; Name = "iPad Air"; Price = 21900m; Description = "Apple iPad Air 5th generation"; ImageUrl = "/images/ipad.jpg"; Stock = 15; Category = "Tablets"; Rating = 4.6; ReviewCount = 89 }
    { Id = 5; Name = "Apple Watch"; Price = 14900m; Description = "Apple Watch Series 9"; ImageUrl = "/images/watch.jpg"; Stock = 30; Category = "Wearables"; Rating = 4.7; ReviewCount = 201 }
]

// ===== Update =====

let shopUpdate msg model =
    match msg with
    | LoadProducts ->
        { model with LoadingProducts = true },
        Cmd.ofAsync
            (fun () -> async {
                do! Async.Sleep 500  // จำลอง network delay
                return mockProducts
            })
            ()
            ProductsLoaded
            (fun e -> LoadFailed e.Message)
    
    | ProductsLoaded products ->
        { model with Products = products; LoadingProducts = false }, Cmd.none
    
    | LoadFailed err ->
        { model with LoadingProducts = false; Notification = Some ("error", err) }, Cmd.none
    
    | NavigateToProduct id ->
        let product = model.Products |> List.tryFind (fun p -> p.Id = id)
        { model with Page = ProductDetailPage id; SelectedProduct = product }, Cmd.none
    
    | NavigateBack ->
        { model with Page = ProductListPage }, Cmd.none
    
    | NavigateToCart ->
        { model with Page = CartPage }, Cmd.none
    
    | NavigateToCheckout ->
        { model with Page = CheckoutPage }, Cmd.none
    
    | AddToCart product ->
        let newCart = Cart.addItem product model.Cart
        { model with 
            Cart = newCart
            Notification = Some ("success", sprintf "เพิ่ม %s ในตะกร้าแล้ว!" product.Name) }
        , Cmd.none
    
    | RemoveFromCart itemId ->
        { model with Cart = Cart.removeItem itemId model.Cart }, Cmd.none
    
    | UpdateCartItemQty (itemId, qty) ->
        { model with Cart = Cart.updateQuantity itemId qty model.Cart }, Cmd.none
    
    | ClearCart ->
        { model with Cart = Cart.empty }, Cmd.none
    
    | PlaceOrder ->
        let orderId = sprintf "ORD-%s" (System.Guid.NewGuid().ToString("N").[..7].ToUpper())
        { model with Cart = Cart.empty; Page = OrderConfirmationPage orderId }, Cmd.none
    
    | OrderPlaced orderId ->
        { model with Cart = Cart.empty; Page = OrderConfirmationPage orderId }, Cmd.none
    
    | SetSearchQuery q ->
        { model with SearchQuery = q }, Cmd.none
    
    | SetCategory cat ->
        { model with SelectedCategory = cat }, Cmd.none
    
    | SetSort sort ->
        { model with SortBy = sort }, Cmd.none
    
    | DismissNotification ->
        { model with Notification = None }, Cmd.none

// ===== Helpers =====

let filteredAndSortedProducts model =
    let filtered =
        model.Products
        |> List.filter (fun p ->
            let matchesSearch = 
                model.SearchQuery = "" ||
                p.Name.ToLower().Contains(model.SearchQuery.ToLower())
            let matchesCategory =
                match model.SelectedCategory with
                | None -> true
                | Some cat -> p.Category = cat
            matchesSearch && matchesCategory)
    
    filtered |> List.sortBy (fun p ->
        match model.SortBy with
        | "price_asc" -> (p.Price |> string)
        | "price_desc" -> (- p.Price |> string)
        | "rating" -> (- p.Rating |> string)
        | _ -> p.Name)

// ===== Views =====

let productCard (product: Product) dispatch =
    Html.div [
        prop.key product.Id
        prop.style [
            style.border "1px solid #e0e0e0"
            style.borderRadius 12
            style.overflow.hidden
            style.cursor "pointer"
            style.transition "box-shadow 0.2s"
        ]
        prop.children [
            Html.img [
                prop.src product.ImageUrl
                prop.style [style.width (length.percent 100); style.height 200; style.objectFit.cover]
            ]
            Html.div [
                prop.style [style.padding 16]
                prop.children [
                    Html.h3 [
                        prop.style [style.margin 0; style.marginBottom 8]
                        prop.text product.Name
                    ]
                    Html.p [
                        prop.style [style.color "#666"; style.fontSize 14]
                        prop.text product.Description
                    ]
                    Html.div [
                        prop.style [
                            style.display.flex
                            style.justifyContent.spaceBetween
                            style.alignItems.center
                            style.marginTop 12
                        ]
                        prop.children [
                            Html.span [
                                prop.style [style.fontSize 20; style.fontWeight.bold; style.color "#007bff"]
                                prop.text (sprintf "฿%s" (product.Price.ToString("N0")))
                            ]
                            Html.div [
                                prop.children [
                                    Html.button [
                                        prop.style [
                                            style.backgroundColor "#007bff"
                                            style.color "white"
                                            style.border "none"
                                            style.borderRadius 8
                                            style.padding (8, 16)
                                            style.cursor "pointer"
                                            style.marginRight 8
                                        ]
                                        prop.text "ดูรายละเอียด"
                                        prop.onClick (fun _ -> dispatch (NavigateToProduct product.Id))
                                    ]
                                    Html.button [
                                        prop.style [
                                            style.backgroundColor "#28a745"
                                            style.color "white"
                                            style.border "none"
                                            style.borderRadius 8
                                            style.padding (8, 16)
                                            style.cursor "pointer"
                                        ]
                                        prop.text "เพิ่มในตะกร้า"
                                        prop.onClick (fun _ -> dispatch (AddToCart product))
                                        prop.disabled (product.Stock = 0)
                                    ]
                                ]
                            ]
                        ]
                    ]
                ]
            ]
        ]
    ]

let cartBadge (count: int) =
    if count > 0 then
        Html.span [
            prop.style [
                style.backgroundColor "#dc3545"
                style.color "white"
                style.borderRadius (length.percent 50)
                style.padding (2, 6)
                style.fontSize 12
                style.marginLeft 4
            ]
            prop.text (string count)
        ]
    else Html.none

let shopView model dispatch =
    Html.div [
        prop.style [style.fontFamily "Arial, sans-serif"; style.minHeight (length.vh 100)]
        prop.children [
            // Header
            Html.header [
                prop.style [
                    style.backgroundColor "#007bff"
                    style.color "white"
                    style.padding (16, 24)
                    style.display.flex
                    style.justifyContent.spaceBetween
                    style.alignItems.center
                ]
                prop.children [
                    Html.h1 [
                        prop.style [style.margin 0; style.fontSize 24]
                        prop.text "🛍️ F# Shop"
                    ]
                    Html.button [
                        prop.style [
                            style.backgroundColor "transparent"
                            style.border "2px solid white"
                            style.color "white"
                            style.borderRadius 8
                            style.padding (8, 16)
                            style.cursor "pointer"
                        ]
                        prop.children [
                            Html.text "ตะกร้า"
                            cartBadge (Cart.itemCount model.Cart)
                        ]
                        prop.onClick (fun _ -> dispatch NavigateToCart)
                    ]
                ]
            ]
            
            // Notification
            match model.Notification with
            | Some (notifType, msg) ->
                Html.div [
                    prop.style [
                        style.backgroundColor (if notifType = "success" then "#d4edda" else "#f8d7da")
                        style.color (if notifType = "success" then "#155724" else "#721c24")
                        style.padding 16
                        style.display.flex
                        style.justifyContent.spaceBetween
                    ]
                    prop.children [
                        Html.span msg
                        Html.button [
                            prop.style [style.backgroundColor "transparent"; style.border "none"; style.cursor "pointer"]
                            prop.text "×"
                            prop.onClick (fun _ -> dispatch DismissNotification)
                        ]
                    ]
                ]
            | None -> Html.none
            
            // Main content
            Html.main [
                prop.style [style.padding 24]
                prop.children [
                    match model.Page with
                    | ProductListPage ->
                        // Search and filters
                        Html.div [
                            prop.style [style.marginBottom 24]
                            prop.children [
                                Html.input [
                                    prop.type' "search"
                                    prop.placeholder "ค้นหาสินค้า..."
                                    prop.value model.SearchQuery
                                    prop.onChange (fun e -> dispatch (SetSearchQuery e.target.value))
                                    prop.style [
                                        style.width (length.percent 100)
                                        style.padding 12
                                        style.borderRadius 8
                                        style.border "1px solid #ddd"
                                        style.fontSize 16
                                        style.boxSizing.borderBox
                                    ]
                                ]
                            ]
                        ]
                        
                        // Product grid
                        if model.LoadingProducts then
                            Html.div [
                                prop.style [style.textAlign.center; style.padding 48]
                                prop.text "กำลังโหลดสินค้า..."
                            ]
                        else
                            let products = filteredAndSortedProducts model
                            Html.div [
                                prop.style [
                                    style.display.grid
                                    style.gridTemplateColumns "repeat(auto-fill, minmax(280px, 1fr))"
                                    style.gap 24
                                ]
                                prop.children [
                                    for product in products do
                                        productCard product dispatch
                                ]
                            ]
                    
                    | CartPage ->
                        Html.div [
                            Html.h2 "ตะกร้าสินค้า"
                            if Cart.itemCount model.Cart = 0 then
                                Html.p "ตะกร้าว่างเปล่า"
                            else
                                Html.div [
                                    for item in model.Cart.Items do
                                        Html.div [
                                            prop.key item.Id
                                            prop.style [
                                                style.display.flex
                                                style.alignItems.center
                                                style.padding 16
                                                style.borderBottom "1px solid #eee"
                                            ]
                                            prop.children [
                                                Html.div [
                                                    prop.style [style.flexGrow 1]
                                                    prop.children [
                                                        Html.h4 item.Product.Name
                                                        Html.p (sprintf "฿%s" (item.Product.Price.ToString("N0")))
                                                    ]
                                                ]
                                                Html.input [
                                                    prop.type' "number"
                                                    prop.value item.Quantity
                                                    prop.min 1
                                                    prop.onChange (fun e -> 
                                                        dispatch (UpdateCartItemQty (item.Id, int e.target.value)))
                                                    prop.style [style.width 60; style.padding 8; style.marginRight 16]
                                                ]
                                                Html.button [
                                                    prop.text "ลบ"
                                                    prop.onClick (fun _ -> dispatch (RemoveFromCart item.Id))
                                                    prop.style [
                                                        style.backgroundColor "#dc3545"
                                                        style.color "white"
                                                        style.border "none"
                                                        style.borderRadius 4
                                                        style.padding (6, 12)
                                                        style.cursor "pointer"
                                                    ]
                                                ]
                                            ]
                                        ]
                                    
                                    Html.div [
                                        prop.style [
                                            style.textAlign.right
                                            style.padding 16
                                            style.fontSize 20
                                            style.fontWeight.bold
                                        ]
                                        prop.text (sprintf "รวม: ฿%s" (Cart.total model.Cart |> _.ToString("N0")))
                                    ]
                                    
                                    Html.button [
                                        prop.text "สั่งซื้อ"
                                        prop.onClick (fun _ -> dispatch NavigateToCheckout)
                                        prop.style [
                                            style.width (length.percent 100)
                                            style.backgroundColor "#28a745"
                                            style.color "white"
                                            style.border "none"
                                            style.borderRadius 8
                                            style.padding 16
                                            style.cursor "pointer"
                                            style.fontSize 18
                                        ]
                                    ]
                                ]
                        ]
                    
                    | OrderConfirmationPage orderId ->
                        Html.div [
                            prop.style [style.textAlign.center; style.padding 48]
                            prop.children [
                                Html.h2 "✓ สั่งซื้อสำเร็จ!"
                                Html.p (sprintf "หมายเลขคำสั่งซื้อ: %s" orderId)
                                Html.button [
                                    prop.text "กลับหน้าหลัก"
                                    prop.onClick (fun _ -> dispatch NavigateBack)
                                    prop.style [
                                        style.backgroundColor "#007bff"
                                        style.color "white"
                                        style.border "none"
                                        style.borderRadius 8
                                        style.padding (12, 24)
                                        style.cursor "pointer"
                                    ]
                                ]
                            ]
                        ]
                    
                    | _ -> Html.div "Page under construction"
                ]
            ]
        ]
    ]

// ===== Program =====

Program.mkProgram initShop shopUpdate shopView
|> Program.withReactSynchronous "app"
|> Program.run
```

---

## สรุป (Summary)

Elmish Architecture ให้:

1. **Predictability**: State เปลี่ยนได้แค่ผ่าน update function
2. **Testability**: Test update function ได้ง่ายเพราะเป็น pure function
3. **Maintainability**: Code structure ชัดเจน แยก concerns ออกจากกัน
4. **Scalability**: Compose models และ messages ได้
5. **Debuggability**: Time-travel debugging ผ่าน Elmish Debugger

---

*ไปต่อที่ Part 103: Feliz - React DSL for F#*
