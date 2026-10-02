# Part 104 - Bolero - Blazor for F#

## บทนำ (Introduction)

Bolero คือ framework สำหรับพัฒนา web applications ด้วย F# บน Blazor WebAssembly ทำให้เราสามารถรัน F# code บน browser ได้โดยตรงผ่าน WebAssembly โดยไม่ต้องพึ่ง JavaScript

---

## 1. What is Bolero?

```
F# Code → .NET WASM Runtime → Browser WebAssembly
                                        ↑
                              ไม่ต้องใช้ JavaScript!
```

### ข้อดีของ Bolero

- **F# on Browser**: รัน F# โดยตรงบน browser
- **Type Safety**: Full .NET type system
- **Elmish Architecture**: MVU pattern
- **Blazor Integration**: ใช้ Blazor components ได้
- **Server-side Option**: รันแบบ Server-side Blazor ได้ด้วย

### Bolero vs Fable

| Feature | Bolero | Fable |
|---------|--------|-------|
| Runtime | WebAssembly | JavaScript |
| Compilation | .NET → WASM | F# → JS |
| Bundle Size | ใหญ่กว่า (initial) | เล็กกว่า |
| Performance | ดีกว่าสำหรับ compute-heavy | ดีกว่าสำหรับ DOM |
| Ecosystem | .NET | npm |
| Startup Time | ช้ากว่า (WASM init) | เร็วกว่า |

---

## 2. การติดตั้ง Bolero (Setup)

### สร้าง Project ใหม่

```bash
# ติดตั้ง Bolero template
dotnet new install Bolero.Templates

# สร้าง client-only app
dotnet new bolero-app -n MyBoleroApp --hosted=false

# สร้าง hosted app (Client + Server)
dotnet new bolero-app -n MyBoleroApp --hosted=true

cd MyBoleroApp
dotnet run
```

### โครงสร้าง Project

```
MyBoleroApp/
├── src/
│   ├── Client/
│   │   ├── Main.fs          # Entry point
│   │   ├── Pages/
│   │   │   ├── Home.fs
│   │   │   ├── Counter.fs
│   │   │   └── FetchData.fs
│   │   ├── wwwroot/
│   │   │   ├── index.html
│   │   │   └── css/
│   │   └── Client.fsproj
│   └── Server/              # สำหรับ hosted mode
│       ├── Startup.fs
│       └── Server.fsproj
└── MyBoleroApp.sln
```

### Client.fsproj

```xml
<Project Sdk="Microsoft.NET.Sdk.BlazorWebAssembly">
  <PropertyGroup>
    <TargetFramework>net7.0</TargetFramework>
    <LangVersion>preview</LangVersion>
    <RootNamespace>MyBoleroApp.Client</RootNamespace>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="Pages/Counter.fs" />
    <Compile Include="Pages/FetchData.fs" />
    <Compile Include="Main.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Bolero" Version="0.22.*" />
    <PackageReference Include="Bolero.HotReload" Version="0.22.*" />
  </ItemGroup>
</Project>
```

---

## 3. Elmish ใน Bolero

### Basic Elmish App

```fsharp
// Main.fs
module MyBoleroApp.Client.Main

open Elmish
open Bolero
open Bolero.Html

// ===== Model =====

type Model = {
    Page: Page
    Counter: int
    Message: string
}

and Page =
    | Home
    | Counter
    | About

let initModel = {
    Page = Home
    Counter = 0
    Message = ""
}

// ===== Messages =====

type Msg =
    | SetPage of Page
    | Increment
    | Decrement
    | Reset
    | SetMessage of string

// ===== Update =====

let update msg model =
    match msg with
    | SetPage page -> { model with Page = page }, Cmd.none
    | Increment -> { model with Counter = model.Counter + 1 }, Cmd.none
    | Decrement -> { model with Counter = model.Counter - 1 }, Cmd.none
    | Reset -> { model with Counter = 0 }, Cmd.none
    | SetMessage msg -> { model with Message = msg }, Cmd.none

// ===== View =====

let navMenu model dispatch =
    nav {
        ul {
            li {
                a {
                    on.click (fun _ -> dispatch (SetPage Home))
                    "Home"
                }
            }
            li {
                a {
                    on.click (fun _ -> dispatch (SetPage Counter))
                    "Counter"
                }
            }
            li {
                a {
                    on.click (fun _ -> dispatch (SetPage About))
                    "About"
                }
            }
        }
    }

let counterPage model dispatch =
    div {
        h1 { "Counter" }
        p {
            attr.class' "count"
            $"Count: {model.Counter}"
        }
        button {
            on.click (fun _ -> dispatch Increment)
            "+"
        }
        button {
            on.click (fun _ -> dispatch Decrement)
            "-"
        }
        button {
            on.click (fun _ -> dispatch Reset)
            "Reset"
        }
    }

let view model dispatch =
    div {
        navMenu model dispatch
        
        main {
            match model.Page with
            | Home ->
                div {
                    h1 { "Home" }
                    p { "ยินดีต้อนรับสู่ Bolero App!" }
                }
            | Counter ->
                counterPage model dispatch
            | About ->
                div {
                    h1 { "About" }
                    p { "Bolero App ที่เขียนด้วย F#" }
                }
        }
    }

// ===== Program =====

type MyApp() =
    inherit ProgramComponent<Model, Msg>()
    
    override _.Program =
        Program.mkProgram (fun _ -> initModel, Cmd.none) update view
```

---

## 4. HTML Templates

### Bolero HTML Templates

```fsharp
// ใช้ HTML template
open Bolero.Html
open Bolero.Templating.Client

// สร้าง template จาก wwwroot/templates/main.html
type MainTemplate = Template<"wwwroot/templates/main.html">

// ===== main.html =====
(*
<!DOCTYPE html>
<html>
<body>
    <div id="main">
        <nav>
            <div class="nav-brand">${NavBrand}</div>
            <div class="nav-items">
                <div class="nav-item" onclick="${NavItemClick}">
                    ${NavItemLabel}
                </div>
            </div>
        </nav>
        <main class="content">
            <div ref="Content"></div>
        </main>
    </div>
</body>
</html>
*)

// ใช้ template ใน F#
let view model dispatch =
    MainTemplate()
        .NavBrand("My App")
        .NavItems(
            concat {
                MainTemplate.NavItem()
                    .NavItemLabel("Home")
                    .NavItemClick(fun _ -> dispatch (SetPage Home))
                    .Elt()
                MainTemplate.NavItem()
                    .NavItemLabel("Counter")
                    .NavItemClick(fun _ -> dispatch (SetPage Counter))
                    .Elt()
            }
        )
        .Content(
            match model.Page with
            | Home -> text "Home page content"
            | Counter -> counterPage model dispatch
            | _ -> empty()
        )
        .Elt()
```

### HTML Computation Expressions

```fsharp
open Bolero.Html

// ===== Basic Elements =====

let basicHtml =
    div {
        attr.id "main"
        attr.class' "container"
        
        h1 { "Hello, Bolero!" }
        
        p {
            attr.class' "intro"
            "นี่คือ Bolero HTML"
        }
        
        ul {
            li { "Item 1" }
            li { "Item 2" }
            li { "Item 3" }
        }
        
        table {
            thead {
                tr {
                    th { "Name" }
                    th { "Value" }
                }
            }
            tbody {
                tr {
                    td { "Counter" }
                    td { "42" }
                }
            }
        }
    }

// ===== Conditional Rendering =====

let conditionalHtml (isLoggedIn: bool) userName =
    div {
        if isLoggedIn then
            p { $"Hello, {userName}!" }
            button { "Logout" }
        else
            p { "Please log in" }
            button { "Login" }
    }

// ===== Lists =====

let listHtml (items: string list) =
    ul {
        for item in items do
            li { item }
    }

// ===== Forms =====

let formHtml dispatch =
    form {
        on.submit (fun e -> dispatch Submit)
        
        div {
            attr.class' "form-group"
            label {
                attr.for' "name"
                "ชื่อ"
            }
            input {
                attr.id "name"
                attr.type' "text"
                attr.placeholder "กรอกชื่อ"
                on.change (fun e -> dispatch (SetName e.Value))
            }
        }
        
        button {
            attr.type' "submit"
            "บันทึก"
        }
    }
```

---

## 5. Routing

### Client-side Routing

```fsharp
open Bolero
open Bolero.Html
open Bolero.Routing

// ===== Define Routes =====

type Page =
    | [<EndPoint "/">] Home
    | [<EndPoint "/counter">] Counter
    | [<EndPoint "/users">] Users
    | [<EndPoint "/users/{id}">] UserDetail of id: int
    | [<EndPoint "/blog/{year}/{month}/{slug}">] BlogPost of year: int * month: int * slug: string

// ===== Router =====

let router = Router.infer SetPage (fun m -> m.Page)

// ===== Program with Router =====

type MyApp() =
    inherit ProgramComponent<Model, Msg>()
    
    override _.Program =
        let update msg model =
            match msg with
            | SetPage page ->
                { model with Page = page }, Cmd.none
            | _ -> model, Cmd.none
        
        Program.mkProgram 
            (fun _ -> initModel, Cmd.none) 
            update 
            view
        |> Program.withRouter router
```

---

## 6. HTTP Client

### HTTP Calls ใน Bolero

```fsharp
open System.Net.Http
open System.Net.Http.Json
open System.Text.Json

// ===== Types =====

type User = {
    Id: int
    Name: string
    Email: string
}

// ===== HTTP Service =====

type HttpService(http: HttpClient) =
    
    member _.GetUsers() : Async<User list> =
        async {
            let! users = 
                http.GetFromJsonAsync<User[]>("/api/users")
                |> Async.AwaitTask
            return users |> Array.toList
        }
    
    member _.GetUser (id: int) : Async<User option> =
        async {
            try
                let! user = 
                    http.GetFromJsonAsync<User>(sprintf "/api/users/%d" id)
                    |> Async.AwaitTask
                return Some user
            with :? HttpRequestException ->
                return None
        }
    
    member _.CreateUser (user: User) : Async<User> =
        async {
            let! response = 
                http.PostAsJsonAsync("/api/users", user)
                |> Async.AwaitTask
            let! created = 
                response.Content.ReadFromJsonAsync<User>()
                |> Async.AwaitTask
            return created
        }
    
    member _.DeleteUser (id: int) : Async<bool> =
        async {
            let! response = 
                http.DeleteAsync(sprintf "/api/users/%d" id)
                |> Async.AwaitTask
            return response.IsSuccessStatusCode
        }

// ===== Elmish Model with HTTP =====

type UsersModel = {
    Users: User list
    Loading: bool
    Error: string option
    SelectedUser: User option
}

type UsersMsg =
    | LoadUsers
    | UsersLoaded of User list
    | LoadFailed of string
    | SelectUser of User
    | DeleteUser of int
    | UserDeleted of int

let usersInit () =
    { Users = []; Loading = false; Error = None; SelectedUser = None },
    Cmd.ofMsg LoadUsers

let usersUpdate (http: HttpService) msg model =
    match msg with
    | LoadUsers ->
        { model with Loading = true; Error = None },
        Cmd.ofAsync
            (fun () -> http.GetUsers())
            ()
            UsersLoaded
            (fun e -> LoadFailed e.Message)
    
    | UsersLoaded users ->
        { model with Users = users; Loading = false }, Cmd.none
    
    | LoadFailed err ->
        { model with Loading = false; Error = Some err }, Cmd.none
    
    | SelectUser user ->
        { model with SelectedUser = Some user }, Cmd.none
    
    | DeleteUser id ->
        model,
        Cmd.ofAsync
            (fun () -> http.DeleteUser id)
            ()
            (fun _ -> UserDeleted id)
            (fun e -> LoadFailed e.Message)
    
    | UserDeleted id ->
        { model with Users = model.Users |> List.filter (fun u -> u.Id <> id) },
        Cmd.none

let usersView model dispatch =
    div {
        h1 { "Users" }
        
        if model.Loading then
            p { "กำลังโหลด..." }
        
        match model.Error with
        | Some err -> p { attr.class' "error"; err }
        | None -> ()
        
        ul {
            for user in model.Users do
                li {
                    attr.key (string user.Id)
                    span { user.Name }
                    span { $" ({user.Email})" }
                    button {
                        on.click (fun _ -> dispatch (DeleteUser user.Id))
                        "ลบ"
                    }
                }
        }
    }

// ===== Dependency Injection =====

// Program.fs
open Microsoft.AspNetCore.Components.WebAssembly.Hosting
open Microsoft.Extensions.DependencyInjection

[<EntryPoint>]
let main args =
    let builder = WebAssemblyHostBuilder.CreateDefault(args)
    builder.RootComponents.Add<App>("#app")
    
    // Register HttpClient
    builder.Services
        .AddScoped(fun sp ->
            new HttpClient(
                BaseAddress = System.Uri(builder.HostEnvironment.BaseAddress)))
        .AddScoped<HttpService>()
    |> ignore
    
    builder.Build().RunAsync() |> Async.AwaitTask |> Async.RunSynchronously
    0
```

---

## 7. JavaScript Interop

### เรียกใช้ JavaScript จาก F#

```fsharp
open Microsoft.JSInterop
open Bolero

// ===== IJSRuntime =====

// Component ที่ inject IJSRuntime
type JsInteropComponent() =
    inherit ProgramComponent<Model, Msg>()
    
    [<Inject>]
    member val JSRuntime = Unchecked.defaultof<IJSRuntime> with get, set
    
    override this.Program =
        let update msg model =
            match msg with
            | ShowAlert text ->
                model,
                Cmd.ofTask
                    (fun () -> 
                        this.JSRuntime.InvokeVoidAsync("alert", text).AsTask())
                    (fun _ -> AlertShown)
                    (fun e -> Error e.Message)
            | _ -> model, Cmd.none
        
        Program.mkProgram init update view

// ===== JavaScript Functions =====

// Call JavaScript function
let callJsFunction (js: IJSRuntime) (functionName: string) (args: obj[]) =
    js.InvokeAsync<obj>(functionName, args).AsTask()
    |> Async.AwaitTask

// Get value from JavaScript
let getJsValue (js: IJSRuntime) (expression: string) =
    js.InvokeAsync<string>("eval", [| expression :> obj |]).AsTask()
    |> Async.AwaitTask

// ===== LocalStorage via JSInterop =====

type LocalStorageService(js: IJSRuntime) =
    
    member _.GetItem(key: string) : Async<string option> =
        async {
            let! value = 
                js.InvokeAsync<string>("localStorage.getItem", key).AsTask()
                |> Async.AwaitTask
            return if isNull value then None else Some value
        }
    
    member _.SetItem(key: string) (value: string) : Async<unit> =
        js.InvokeVoidAsync("localStorage.setItem", key, value).AsTask()
        |> Async.AwaitTask
    
    member _.RemoveItem(key: string) : Async<unit> =
        js.InvokeVoidAsync("localStorage.removeItem", key).AsTask()
        |> Async.AwaitTask

// ===== Charts via JavaScript =====

// interop.js
(*
window.createChart = function(canvasId, data) {
    const ctx = document.getElementById(canvasId).getContext('2d');
    return new Chart(ctx, {
        type: 'bar',
        data: data,
        options: { responsive: true }
    });
};
*)

let createChart (js: IJSRuntime) canvasId data =
    js.InvokeVoidAsync("createChart", canvasId, data).AsTask()
    |> Async.AwaitTask
```

---

## 8. Forms and Validation

```fsharp
open Bolero
open Bolero.Html

// ===== Form Types =====

type ContactForm = {
    Name: string
    Email: string
    Subject: string
    Message: string
    Category: string
}

type FormError = {
    Field: string
    Message: string
}

// ===== Validation =====

let validateForm (form: ContactForm) : FormError list =
    [
        if System.String.IsNullOrWhiteSpace(form.Name) then
            yield { Field = "Name"; Message = "กรุณากรอกชื่อ" }
        
        if System.String.IsNullOrWhiteSpace(form.Email) then
            yield { Field = "Email"; Message = "กรุณากรอก Email" }
        elif not (form.Email.Contains("@")) then
            yield { Field = "Email"; Message = "Email ไม่ถูกต้อง" }
        
        if System.String.IsNullOrWhiteSpace(form.Message) then
            yield { Field = "Message"; Message = "กรุณากรอกข้อความ" }
        elif form.Message.Length < 10 then
            yield { Field = "Message"; Message = "ข้อความต้องมีอย่างน้อย 10 ตัวอักษร" }
    ]

// ===== Form Model =====

type FormModel = {
    Form: ContactForm
    Errors: FormError list
    IsSubmitting: bool
    IsSuccess: bool
}

type FormMsg =
    | UpdateName of string
    | UpdateEmail of string
    | UpdateSubject of string
    | UpdateMessage of string
    | UpdateCategory of string
    | Submit
    | SubmitResult of bool
    | Reset

let formInit () =
    {
        Form = { Name = ""; Email = ""; Subject = ""; Message = ""; Category = "general" }
        Errors = []
        IsSubmitting = false
        IsSuccess = false
    }, Cmd.none

let formUpdate msg model =
    match msg with
    | UpdateName name ->
        { model with Form = { model.Form with Name = name } }, Cmd.none
    
    | UpdateEmail email ->
        { model with Form = { model.Form with Email = email } }, Cmd.none
    
    | UpdateSubject subject ->
        { model with Form = { model.Form with Subject = subject } }, Cmd.none
    
    | UpdateMessage message ->
        { model with Form = { model.Form with Message = message } }, Cmd.none
    
    | UpdateCategory category ->
        { model with Form = { model.Form with Category = category } }, Cmd.none
    
    | Submit ->
        let errors = validateForm model.Form
        if not (List.isEmpty errors) then
            { model with Errors = errors }, Cmd.none
        else
            { model with IsSubmitting = true; Errors = [] },
            Cmd.ofAsync
                (fun () -> async {
                    do! Async.Sleep 1000  // จำลอง API call
                    return true
                })
                ()
                SubmitResult
                (fun _ -> SubmitResult false)
    
    | SubmitResult success ->
        { model with IsSubmitting = false; IsSuccess = success }, Cmd.none
    
    | Reset ->
        fst (formInit()), Cmd.none

// ===== Form View =====

let errorFor (field: string) (errors: FormError list) =
    match errors |> List.tryFind (fun e -> e.Field = field) with
    | Some error ->
        span {
            attr.class' "field-error"
            attr.style "color: red; font-size: 0.875em;"
            error.Message
        }
    | None -> empty()

let contactFormView model dispatch =
    if model.IsSuccess then
        div {
            attr.class' "success-message"
            h2 { "✓ ส่งข้อความสำเร็จ!" }
            p { "เราจะติดต่อกลับโดยเร็ว" }
            button {
                on.click (fun _ -> dispatch Reset)
                "ส่งอีกครั้ง"
            }
        }
    else
        form {
            on.submit (fun e -> dispatch Submit)
            
            // Name field
            div {
                attr.class' "form-group"
                label { attr.for' "name"; "ชื่อ *" }
                input {
                    attr.id "name"
                    attr.type' "text"
                    attr.value model.Form.Name
                    on.input (fun e -> dispatch (UpdateName e.Value))
                    attr.placeholder "กรอกชื่อของคุณ"
                }
                errorFor "Name" model.Errors
            }
            
            // Email field
            div {
                attr.class' "form-group"
                label { attr.for' "email"; "Email *" }
                input {
                    attr.id "email"
                    attr.type' "email"
                    attr.value model.Form.Email
                    on.input (fun e -> dispatch (UpdateEmail e.Value))
                    attr.placeholder "your@email.com"
                }
                errorFor "Email" model.Errors
            }
            
            // Category
            div {
                attr.class' "form-group"
                label { attr.for' "category"; "หมวดหมู่" }
                select {
                    attr.id "category"
                    attr.value model.Form.Category
                    on.change (fun e -> dispatch (UpdateCategory e.Value))
                    option { attr.value "general"; "ทั่วไป" }
                    option { attr.value "support"; "ขอความช่วยเหลือ" }
                    option { attr.value "billing"; "การชำระเงิน" }
                    option { attr.value "other"; "อื่นๆ" }
                }
            }
            
            // Message
            div {
                attr.class' "form-group"
                label { attr.for' "message"; "ข้อความ *" }
                textarea {
                    attr.id "message"
                    attr.rows "6"
                    attr.value model.Form.Message
                    on.input (fun e -> dispatch (UpdateMessage e.Value))
                    attr.placeholder "กรอกข้อความของคุณ..."
                }
                errorFor "Message" model.Errors
            }
            
            // Submit button
            button {
                attr.type' "submit"
                attr.disabled model.IsSubmitting
                if model.IsSubmitting then "กำลังส่ง..." else "ส่งข้อความ"
            }
        }
```

---

## 9. Authentication

```fsharp
open Bolero
open Bolero.Html
open Microsoft.AspNetCore.Components.Authorization

// ===== Auth Types =====

type AuthState =
    | NotAuthenticated
    | Authenticating
    | Authenticated of username: string * role: string
    | AuthError of string

type LoginForm = {
    Username: string
    Password: string
}

type AuthMsg =
    | SetUsername of string
    | SetPassword of string
    | Login
    | LoginSuccess of username: string * role: string
    | LoginFailed of string
    | Logout

// ===== Auth Service =====

type AuthService(http: System.Net.Http.HttpClient) =
    
    member _.Login (username: string) (password: string) : Async<Result<string * string, string>> =
        async {
            try
                let body = {| username = username; password = password |}
                let! response = 
                    http.PostAsJsonAsync("/api/auth/login", body)
                    |> Async.AwaitTask
                
                if response.IsSuccessStatusCode then
                    let! result = 
                        response.Content.ReadFromJsonAsync<{| username: string; role: string |}>()
                        |> Async.AwaitTask
                    return Ok (result.username, result.role)
                else
                    return Error "Invalid credentials"
            with ex ->
                return Error ex.Message
        }
    
    member _.Logout () : Async<unit> =
        async {
            do! http.PostAsync("/api/auth/logout", null) |> Async.AwaitTask |> Async.Ignore
        }

// ===== Protected Component =====

type ProtectedComponent() =
    inherit ProgramComponent<Model, Msg>()
    
    [<Inject>]
    member val AuthState = Unchecked.defaultof<AuthenticationStateProvider> with get, set
    
    override this.Program =
        let init _ =
            let model = { (* ... *) }
            model,
            Cmd.ofAsync
                (fun () -> async {
                    let! authState = 
                        this.AuthState.GetAuthenticationStateAsync()
                        |> Async.AwaitTask
                    return authState.User.Identity.IsAuthenticated
                })
                ()
                CheckedAuth
                (fun _ -> CheckedAuth false)
        
        Program.mkProgram init update view

// ===== Login Page =====

type LoginModel = {
    Form: LoginForm
    AuthState: AuthState
}

let loginView model dispatch =
    div {
        attr.class' "login-container"
        
        match model.AuthState with
        | Authenticated (username, role) ->
            div {
                p { $"สวัสดี, {username}! (Role: {role})" }
                button {
                    on.click (fun _ -> dispatch Logout)
                    "ออกจากระบบ"
                }
            }
        | AuthError err ->
            div {
                attr.class' "error"
                p { $"Error: {err}" }
            }
        | _ -> ()
        
        if model.AuthState <> Authenticated ("", "") then
            form {
                on.submit (fun e -> dispatch Login)
                
                h2 { "เข้าสู่ระบบ" }
                
                div {
                    label { "Username" }
                    input {
                        attr.type' "text"
                        attr.value model.Form.Username
                        on.input (fun e -> dispatch (SetUsername e.Value))
                    }
                }
                
                div {
                    label { "Password" }
                    input {
                        attr.type' "password"
                        attr.value model.Form.Password
                        on.input (fun e -> dispatch (SetPassword e.Value))
                    }
                }
                
                button {
                    attr.type' "submit"
                    attr.disabled (model.AuthState = Authenticating)
                    if model.AuthState = Authenticating then "กำลังเข้าสู่ระบบ..." else "เข้าสู่ระบบ"
                }
            }
    }
```

---

## 10. Complete Bolero App

### Full Task Manager App

```fsharp
// TaskManager.fs - Complete Bolero App
module TaskManager

open System
open Bolero
open Bolero.Html
open Microsoft.AspNetCore.Components.WebAssembly.Http

// ===== Domain Types =====

type TaskId = int

type Priority = Low | Normal | High | Critical

type Status = Todo | InProgress | Done | Cancelled

type Task = {
    Id: TaskId
    Title: string
    Description: string
    Priority: Priority
    Status: Status
    AssignedTo: string option
    DueDate: DateTime option
    CreatedAt: DateTime
    Tags: string list
}

type Project = {
    Id: int
    Name: string
    Description: string
    Tasks: Task list
    Color: string
}

// ===== UI Types =====

type Page =
    | Dashboard
    | ProjectList
    | ProjectDetail of int
    | TaskDetail of int
    | CreateTask
    | Settings

type ViewMode = ListView | KanbanView | CalendarView

type SortField = ByTitle | ByPriority | ByStatus | ByDueDate | ByCreatedAt

// ===== Model =====

type AppModel = {
    Page: Page
    Projects: Project list
    SelectedProject: Project option
    SelectedTask: Task option
    ViewMode: ViewMode
    SortField: SortField
    SortAscending: bool
    FilterStatus: Status option
    FilterPriority: Priority option
    SearchQuery: string
    IsLoading: bool
    Error: string option
    Notifications: (string * string) list  // (type, message)
    // New task form
    NewTaskTitle: string
    NewTaskDescription: string
    NewTaskPriority: Priority
    NewTaskDueDate: string
}

let initApp () =
    let sampleProject = {
        Id = 1
        Name = "Website Redesign"
        Description = "ปรับปรุง UI/UX ของเว็บไซต์"
        Color = "#007bff"
        Tasks = [
            { Id = 1; Title = "Design mockups"; Description = "สร้าง mockup สำหรับ home page"; Priority = High; Status = Done; AssignedTo = Some "Alice"; DueDate = Some (DateTime.Now.AddDays -2.0); CreatedAt = DateTime.Now.AddDays -10.0; Tags = ["design"; "ui"] }
            { Id = 2; Title = "Implement navigation"; Description = "พัฒนา navigation component"; Priority = Normal; Status = InProgress; AssignedTo = Some "Bob"; DueDate = Some (DateTime.Now.AddDays 3.0); CreatedAt = DateTime.Now.AddDays -5.0; Tags = ["dev"; "frontend"] }
            { Id = 3; Title = "Write API docs"; Description = "เขียน documentation สำหรับ API"; Priority = Low; Status = Todo; AssignedTo = None; DueDate = Some (DateTime.Now.AddDays 7.0); CreatedAt = DateTime.Now.AddDays -1.0; Tags = ["docs"] }
            { Id = 4; Title = "Performance testing"; Description = "ทดสอบ performance"; Priority = Critical; Status = Todo; AssignedTo = Some "Charlie"; DueDate = Some (DateTime.Now.AddDays 10.0); CreatedAt = DateTime.Now; Tags = ["testing"] }
        ]
    }
    
    {
        Page = Dashboard
        Projects = [sampleProject]
        SelectedProject = Some sampleProject
        SelectedTask = None
        ViewMode = KanbanView
        SortField = ByCreatedAt
        SortAscending = false
        FilterStatus = None
        FilterPriority = None
        SearchQuery = ""
        IsLoading = false
        Error = None
        Notifications = []
        NewTaskTitle = ""
        NewTaskDescription = ""
        NewTaskPriority = Normal
        NewTaskDueDate = ""
    }, Cmd.none

// ===== Messages =====

type AppMsg =
    | NavigateTo of Page
    | SelectProject of Project
    | SelectTask of Task
    | ClearSelection
    | SetViewMode of ViewMode
    | SetSort of SortField
    | SetFilter of Status option
    | SetPriorityFilter of Priority option
    | SetSearch of string
    | UpdateNewTaskTitle of string
    | UpdateNewTaskDesc of string
    | UpdateNewTaskPriority of Priority
    | UpdateNewTaskDueDate of string
    | CreateTask
    | UpdateTaskStatus of TaskId * Status
    | DeleteTask of TaskId
    | AddTag of TaskId * string
    | DismissError
    | AddNotification of string * string
    | DismissNotification of int

// ===== Update =====

let appUpdate msg model =
    match msg with
    | NavigateTo page ->
        { model with Page = page }, Cmd.none
    
    | SelectProject project ->
        { model with SelectedProject = Some project; Page = ProjectDetail project.Id }, Cmd.none
    
    | SelectTask task ->
        { model with SelectedTask = Some task }, Cmd.none
    
    | SetViewMode mode ->
        { model with ViewMode = mode }, Cmd.none
    
    | SetSort field ->
        let asc = 
            if model.SortField = field then not model.SortAscending
            else true
        { model with SortField = field; SortAscending = asc }, Cmd.none
    
    | SetFilter status ->
        { model with FilterStatus = status }, Cmd.none
    
    | SetPriorityFilter priority ->
        { model with FilterPriority = priority }, Cmd.none
    
    | SetSearch q ->
        { model with SearchQuery = q }, Cmd.none
    
    | UpdateNewTaskTitle t -> { model with NewTaskTitle = t }, Cmd.none
    | UpdateNewTaskDesc d -> { model with NewTaskDescription = d }, Cmd.none
    | UpdateNewTaskPriority p -> { model with NewTaskPriority = p }, Cmd.none
    | UpdateNewTaskDueDate d -> { model with NewTaskDueDate = d }, Cmd.none
    
    | CreateTask when model.NewTaskTitle.Trim() <> "" ->
        match model.SelectedProject with
        | Some project ->
            let dueDate = 
                if model.NewTaskDueDate <> "" then
                    match DateTime.TryParse(model.NewTaskDueDate) with
                    | true, dt -> Some dt
                    | _ -> None
                else None
            
            let maxId = 
                project.Tasks 
                |> List.map (fun t -> t.Id) 
                |> List.fold max 0
            
            let newTask = {
                Id = maxId + 1
                Title = model.NewTaskTitle.Trim()
                Description = model.NewTaskDescription
                Priority = model.NewTaskPriority
                Status = Todo
                AssignedTo = None
                DueDate = dueDate
                CreatedAt = DateTime.Now
                Tags = []
            }
            
            let updatedProject = { project with Tasks = project.Tasks @ [newTask] }
            let updatedProjects = 
                model.Projects |> List.map (fun p -> 
                    if p.Id = project.Id then updatedProject else p)
            
            { model with
                Projects = updatedProjects
                SelectedProject = Some updatedProject
                NewTaskTitle = ""
                NewTaskDescription = ""
                NewTaskDueDate = "" }
            , Cmd.ofMsg (AddNotification ("success", sprintf "สร้าง task '%s' สำเร็จ" newTask.Title))
        | None -> model, Cmd.none
    
    | CreateTask -> model, Cmd.none
    
    | UpdateTaskStatus (taskId, newStatus) ->
        match model.SelectedProject with
        | Some project ->
            let updatedTasks = 
                project.Tasks |> List.map (fun t ->
                    if t.Id = taskId then { t with Status = newStatus }
                    else t)
            let updatedProject = { project with Tasks = updatedTasks }
            let updatedProjects = 
                model.Projects |> List.map (fun p ->
                    if p.Id = project.Id then updatedProject else p)
            
            { model with
                Projects = updatedProjects
                SelectedProject = Some updatedProject }
            , Cmd.none
        | None -> model, Cmd.none
    
    | DeleteTask taskId ->
        match model.SelectedProject with
        | Some project ->
            let updatedTasks = project.Tasks |> List.filter (fun t -> t.Id <> taskId)
            let updatedProject = { project with Tasks = updatedTasks }
            let updatedProjects =
                model.Projects |> List.map (fun p ->
                    if p.Id = project.Id then updatedProject else p)
            
            { model with
                Projects = updatedProjects
                SelectedProject = Some updatedProject }
            , Cmd.none
        | None -> model, Cmd.none
    
    | AddNotification (notifType, msg) ->
        { model with Notifications = model.Notifications @ [(notifType, msg)] },
        Cmd.ofAsync
            (fun () -> async { do! Async.Sleep 3000 })
            ()
            (fun _ -> DismissNotification (model.Notifications.Length))
            ignore
    
    | DismissNotification i ->
        let notifications =
            model.Notifications |> List.mapi (fun idx n -> (idx, n))
            |> List.filter (fun (idx, _) -> idx <> i)
            |> List.map snd
        { model with Notifications = notifications }, Cmd.none
    
    | _ -> model, Cmd.none

// ===== Views =====

let priorityColor = function
    | Critical -> "#dc3545"
    | High -> "#fd7e14"
    | Normal -> "#007bff"
    | Low -> "#6c757d"

let statusColor = function
    | Todo -> "#6c757d"
    | InProgress -> "#007bff"
    | Done -> "#28a745"
    | Cancelled -> "#dc3545"

let priorityLabel = function
    | Critical -> "วิกฤต"
    | High -> "สูง"
    | Normal -> "ปกติ"
    | Low -> "ต่ำ"

let statusLabel = function
    | Todo -> "รอดำเนินการ"
    | InProgress -> "กำลังทำ"
    | Done -> "เสร็จแล้ว"
    | Cancelled -> "ยกเลิก"

let taskCard (task: Task) dispatch =
    div {
        attr.class' "task-card"
        attr.key (string task.Id)
        attr.style $"border-left: 4px solid {priorityColor task.Priority}; padding: 12px; background: white; border-radius: 6px; margin-bottom: 8px; box-shadow: 0 1px 3px rgba(0,0,0,0.1)"
        
        div {
            attr.style "display: flex; justify-content: space-between; align-items: start"
            
            div {
                p {
                    attr.style "margin: 0 0 4px; font-weight: bold"
                    task.Title
                }
                if task.Description <> "" then
                    p {
                        attr.style "margin: 0 0 8px; color: #666; font-size: 0.875em"
                        task.Description
                    }
            }
            
            button {
                attr.style "background: none; border: none; cursor: pointer; color: #dc3545; font-size: 16px"
                on.click (fun _ -> dispatch (DeleteTask task.Id))
                "×"
            }
        }
        
        // Tags
        if not (List.isEmpty task.Tags) then
            div {
                attr.style "display: flex; gap: 4px; flex-wrap: wrap; margin-bottom: 8px"
                for tag in task.Tags do
                    span {
                        attr.style "background: #e9ecef; padding: 2px 8px; border-radius: 4px; font-size: 0.75em"
                        tag
                    }
            }
        
        // Meta info
        div {
            attr.style "display: flex; justify-content: space-between; align-items: center; font-size: 0.75em; color: #999"
            
            match task.AssignedTo with
            | Some person -> span { $"👤 {person}" }
            | None -> span { "ไม่ได้กำหนด" }
            
            match task.DueDate with
            | Some dt ->
                let isOverdue = dt < DateTime.Now && task.Status <> Done
                span {
                    attr.style $"color: {if isOverdue then "#dc3545" else "#6c757d"}"
                    $"📅 {dt.ToString("dd/MM/yyyy")}"
                }
            | None -> span { "" }
        }
        
        // Status selector
        div {
            attr.style "margin-top: 8px"
            select {
                attr.value (
                    match task.Status with
                    | Todo -> "todo"
                    | InProgress -> "inprogress"
                    | Done -> "done"
                    | Cancelled -> "cancelled"
                )
                on.change (fun e ->
                    let status =
                        match e.Value with
                        | "inprogress" -> InProgress
                        | "done" -> Done
                        | "cancelled" -> Cancelled
                        | _ -> Todo
                    dispatch (UpdateTaskStatus (task.Id, status)))
                attr.style "font-size: 0.875em; padding: 4px; border-radius: 4px; border: 1px solid #ddd"
                option { attr.value "todo"; statusLabel Todo }
                option { attr.value "inprogress"; statusLabel InProgress }
                option { attr.value "done"; statusLabel Done }
                option { attr.value "cancelled"; statusLabel Cancelled }
            }
        }
    }

let kanbanView (tasks: Task list) dispatch =
    let columns = [
        Todo, "รอดำเนินการ", "#6c757d"
        InProgress, "กำลังทำ", "#007bff"
        Done, "เสร็จแล้ว", "#28a745"
        Cancelled, "ยกเลิก", "#dc3545"
    ]
    
    div {
        attr.style "display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px"
        
        for (status, label, color) in columns do
            let columnTasks = tasks |> List.filter (fun t -> t.Status = status)
            
            div {
                attr.style "background: #f8f9fa; border-radius: 8px; padding: 12px"
                
                div {
                    attr.style $"display: flex; align-items: center; margin-bottom: 12px"
                    span {
                        attr.style $"width: 12px; height: 12px; border-radius: 50%; background: {color}; margin-right: 8px"
                        ""
                    }
                    strong { label }
                    span {
                        attr.style "margin-left: auto; background: #e9ecef; border-radius: 12px; padding: 2px 8px; font-size: 0.875em"
                        string columnTasks.Length
                    }
                }
                
                for task in columnTasks do
                    taskCard task dispatch
            }
    }

let appView model dispatch =
    div {
        attr.style "font-family: Arial, sans-serif; min-height: 100vh; background: #f0f2f5"
        
        // Notifications
        div {
            attr.style "position: fixed; top: 16px; right: 16px; z-index: 1000"
            for i, (notifType, msg) in model.Notifications |> List.mapi (fun i n -> (i, n)) do
                div {
                    attr.key (string i)
                    attr.style $"background: {if notifType = \"success\" then \"#d4edda\" else \"#f8d7da\"}; padding: 12px 16px; border-radius: 8px; margin-bottom: 8px; display: flex; justify-content: space-between; min-width: 280px"
                    span { msg }
                    button {
                        attr.style "background: none; border: none; cursor: pointer; margin-left: 16px"
                        on.click (fun _ -> dispatch (DismissNotification i))
                        "×"
                    }
                }
        }
        
        // Header
        header {
            attr.style "background: #1a1a2e; color: white; padding: 16px 24px; display: flex; align-items: center"
            h1 {
                attr.style "margin: 0; font-size: 1.25em"
                "📋 Task Manager (Bolero/F#)"
            }
        }
        
        // Main layout
        div {
            attr.style "display: flex; min-height: calc(100vh - 64px)"
            
            // Sidebar
            nav {
                attr.style "width: 240px; background: white; padding: 16px; border-right: 1px solid #e0e0e0"
                
                p {
                    attr.style "font-size: 0.75em; text-transform: uppercase; color: #999; margin-bottom: 8px"
                    "Projects"
                }
                
                for project in model.Projects do
                    div {
                        attr.style $"padding: 8px 12px; cursor: pointer; border-radius: 6px; display: flex; align-items: center; background: {if model.SelectedProject |> Option.map (fun p -> p.Id = project.Id) |> Option.defaultValue false then \"#e8f4fd\" else \"transparent\"}"
                        on.click (fun _ -> dispatch (SelectProject project))
                        span {
                            attr.style $"width: 10px; height: 10px; border-radius: 50%; background: {project.Color}; margin-right: 8px"
                            ""
                        }
                        span { project.Name }
                        span {
                            attr.style "margin-left: auto; font-size: 0.75em; color: #999"
                            string project.Tasks.Length
                        }
                    }
            }
            
            // Content
            main {
                attr.style "flex: 1; padding: 24px; overflow: auto"
                
                match model.SelectedProject with
                | None ->
                    div {
                        attr.style "text-align: center; padding: 48px; color: #999"
                        "เลือก project จาก sidebar"
                    }
                | Some project ->
                    div {
                        // Project header
                        div {
                            attr.style "display: flex; justify-content: space-between; align-items: center; margin-bottom: 24px"
                            
                            div {
                                h2 {
                                    attr.style "margin: 0 0 4px"
                                    project.Name
                                }
                                p {
                                    attr.style "margin: 0; color: #666"
                                    project.Description
                                }
                            }
                            
                            // View mode buttons
                            div {
                                attr.style "display: flex; gap: 8px"
                                for (mode, label) in [(ListView, "📋 List"); (KanbanView, "🗂️ Kanban")] do
                                    button {
                                        attr.style $"padding: 8px 16px; border-radius: 6px; border: 1px solid #ddd; cursor: pointer; background: {if model.ViewMode = mode then \"#007bff\" else \"white\"}; color: {if model.ViewMode = mode then \"white\" else \"#333\"}"
                                        on.click (fun _ -> dispatch (SetViewMode mode))
                                        label
                                    }
                            }
                        }
                        
                        // Add task form
                        div {
                            attr.style "background: white; padding: 16px; border-radius: 8px; margin-bottom: 24px; box-shadow: 0 1px 3px rgba(0,0,0,0.1)"
                            
                            h3 { attr.style "margin: 0 0 12px"; "เพิ่ม Task ใหม่" }
                            
                            div {
                                attr.style "display: flex; gap: 8px; flex-wrap: wrap"
                                
                                input {
                                    attr.type' "text"
                                    attr.placeholder "ชื่อ Task"
                                    attr.value model.NewTaskTitle
                                    on.input (fun e -> dispatch (UpdateNewTaskTitle e.Value))
                                    attr.style "flex: 1; padding: 8px 12px; border: 1px solid #ddd; border-radius: 6px; min-width: 200px"
                                }
                                
                                select {
                                    attr.value (
                                        match model.NewTaskPriority with
                                        | Critical -> "critical"
                                        | High -> "high"
                                        | Normal -> "normal"
                                        | Low -> "low")
                                    on.change (fun e ->
                                        let p =
                                            match e.Value with
                                            | "critical" -> Critical
                                            | "high" -> High
                                            | "low" -> Low
                                            | _ -> Normal
                                        dispatch (UpdateNewTaskPriority p))
                                    attr.style "padding: 8px 12px; border: 1px solid #ddd; border-radius: 6px"
                                    option { attr.value "low"; "ต่ำ" }
                                    option { attr.value "normal"; "ปกติ" }
                                    option { attr.value "high"; "สูง" }
                                    option { attr.value "critical"; "วิกฤต" }
                                }
                                
                                input {
                                    attr.type' "date"
                                    attr.value model.NewTaskDueDate
                                    on.input (fun e -> dispatch (UpdateNewTaskDueDate e.Value))
                                    attr.style "padding: 8px 12px; border: 1px solid #ddd; border-radius: 6px"
                                }
                                
                                button {
                                    on.click (fun _ -> dispatch CreateTask)
                                    attr.disabled (model.NewTaskTitle.Trim() = "")
                                    attr.style "padding: 8px 16px; background: #007bff; color: white; border: none; border-radius: 6px; cursor: pointer"
                                    "เพิ่ม Task"
                                }
                            }
                        }
                        
                        // Tasks view
                        match model.ViewMode with
                        | KanbanView ->
                            kanbanView project.Tasks dispatch
                        | ListView ->
                            div {
                                for task in project.Tasks do
                                    taskCard task dispatch
                            }
                        | _ ->
                            p { "View mode ไม่รองรับ" }
                    }
            }
        ]
    }

// ===== App Component =====

type App() =
    inherit ProgramComponent<AppModel, AppMsg>()
    
    override _.Program =
        Program.mkProgram initApp appUpdate appView
```

---

## สรุป (Summary)

Bolero ให้ความสามารถในการพัฒนา web apps ด้วย F# โดยรันบน WebAssembly:

1. **Full F# on Browser**: ไม่ต้องแปลงเป็น JavaScript
2. **Elmish Architecture**: MVU pattern ที่ familiar
3. **HTML Templates**: ใช้ HTML templates แบบ type-safe
4. **Routing**: Type-safe routing ด้วย Endpoint attributes
5. **HTTP Client**: .NET HttpClient ที่ทรงพลัง
6. **JS Interop**: ยังสามารถใช้ JavaScript ได้เมื่อจำเป็น

---

*ไปต่อที่ Part 105: F# Scripting และ Automation*
