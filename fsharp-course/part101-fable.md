# Part 101 - Fable - F# to JavaScript

## บทนำ (Introduction)

Fable คือ F# compiler ที่แปลง F# code ให้เป็น JavaScript ทำให้เราสามารถเขียน frontend web applications ด้วย F# ได้ โดยได้ประโยชน์จาก type safety และ functional programming ของ F# ในขณะที่ยังสามารถทำงานร่วมกับ JavaScript ecosystem ได้

---

## 1. What is Fable?

```
Fable = F# + Babel

F# Source Code → Fable Compiler → JavaScript/TypeScript → Browser/Node.js
```

### ข้อดีของ Fable

- **Type Safety**: ใช้ F# type system ป้องกัน runtime errors
- **Functional Programming**: Pattern matching, immutable data, pure functions
- **JavaScript Interop**: ใช้ JavaScript libraries ได้
- **React Integration**: ใช้ React ผ่าน Feliz
- **Elmish Architecture**: MVU pattern สำหรับ state management

---

## 2. การติดตั้ง Fable (Setup and Installation)

### Prerequisites

```bash
# ติดตั้ง .NET SDK
dotnet --version  # ต้องการ .NET 6+

# ติดตั้ง Node.js
node --version  # ต้องการ Node.js 14+
npm --version
```

### สร้าง Fable Project ใหม่

```bash
# ติดตั้ง Fable template
dotnet new install Fable.Template

# สร้าง project ใหม่
dotnet new fable -n MyFableApp
cd MyFableApp

# ติดตั้ง Node dependencies
npm install

# รัน development server
npm start
```

### โครงสร้าง Project

```
MyFableApp/
├── src/
│   ├── App.fs          # Main F# file
│   └── App.fsproj      # F# project file
├── public/
│   ├── index.html      # HTML entry point
│   └── favicon.ico
├── package.json        # Node.js dependencies
├── webpack.config.js   # Webpack configuration
└── .gitignore
```

### App.fsproj

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
    <LangVersion>preview</LangVersion>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="App.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Fable.Core" Version="4.*" />
    <PackageReference Include="Fable.Browser.Dom" Version="2.*" />
    <PackageReference Include="Feliz" Version="2.*" />
    <PackageReference Include="Feliz.Router" Version="4.*" />
    <PackageReference Include="Fable.Elmish" Version="4.*" />
    <PackageReference Include="Fable.Elmish.React" Version="4.*" />
  </ItemGroup>
</Project>
```

### package.json

```json
{
  "private": true,
  "scripts": {
    "start": "webpack-dev-server",
    "build": "webpack --mode production"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "@babel/core": "^7.21.0",
    "fable-compiler": "^4.3.0",
    "fable-loader": "^2.1.8",
    "webpack": "^5.80.0",
    "webpack-cli": "^5.0.2",
    "webpack-dev-server": "^4.15.0"
  }
}
```

---

## 3. F# to JavaScript Compilation

### Basic F# Code ที่ Fable แปลงได้

```fsharp
// App.fs
module App

// ฟังก์ชันพื้นฐาน - แปลงเป็น JavaScript function
let add x y = x + y

// แปลงเป็น JavaScript arrow function:
// const add = (x, y) => x + y;

// Lists ใน F# → Arrays ใน JavaScript
let numbers = [1; 2; 3; 4; 5]
// แปลงเป็น: const numbers = [1, 2, 3, 4, 5];

// Records ใน F# → Plain Objects ใน JavaScript
type Person = {
    Name: string
    Age: int
}

let person = { Name = "สมชาย"; Age = 30 }
// แปลงเป็น: const person = { Name: "สมชาย", Age: 30 };

// Discriminated Unions → Tagged unions ใน JavaScript
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base: float * height: float

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rectangle (w, h) -> w * h
    | Triangle (b, h) -> 0.5 * b * h

// Pattern matching แปลงเป็น switch/if statements
```

### DOM Manipulation

```fsharp
open Browser.Dom
open Browser.Types

// เข้าถึง DOM elements
let myDiv = document.getElementById("my-div")

// สร้าง element ใหม่
let newParagraph = document.createElement("p")
newParagraph.textContent <- "Hello from F#!"
document.body.appendChild(newParagraph) |> ignore

// เพิ่ม event listener
let button = document.getElementById("my-button") :?> HTMLButtonElement
button.addEventListener("click", fun e ->
    window.alert("Button clicked!")
)

// Fetch API
open Fable.Core
open Fable.Core.JsInterop

let fetchData () =
    promise {
        let! response = fetch "https://api.example.com/data" []
        let! json = response.json()
        return json
    }
```

### JavaScript Interop

```fsharp
open Fable.Core
open Fable.Core.JsInterop

// Import JavaScript function
[<ImportDefault("./myModule.js")>]
let myJsModule : obj = jsNative

// เรียกใช้ JavaScript function
let result = myJsModule?someFunction("arg1", "arg2")

// Emit raw JavaScript
[<Emit("console.log($0)")>]
let consoleLog (msg: string) : unit = jsNative

// ใช้ dynamic typing
let dynamicObj : obj = createEmpty
dynamicObj?name <- "test"
dynamicObj?value <- 42

// Type cast
let typedValue = !!dynamicObj?value : int

// JavaScript Promise
open Fable.Core.JS

let myPromise = 
    Promise.create(fun resolve reject ->
        // ทำงาน async...
        resolve "result"
    )
```

---

## 4. React กับ Fable (Feliz)

### การติดตั้ง Feliz

```xml
<!-- เพิ่มใน .fsproj -->
<PackageReference Include="Feliz" Version="2.*" />
```

### Component พื้นฐาน

```fsharp
open Feliz

// Simple React component
let helloWorld = React.functionComponent(fun () ->
    Html.div [
        Html.h1 "Hello, World from F#!"
        Html.p "นี่คือ React component ที่เขียนด้วย F#"
    ]
)

// Component ที่รับ props
type CounterProps = {
    initialCount: int
    label: string
}

let counter = React.functionComponent(fun (props: CounterProps) ->
    let count, setCount = React.useState(props.initialCount)
    
    Html.div [
        prop.className "counter"
        prop.children [
            Html.h2 props.label
            Html.p (sprintf "Count: %d" count)
            Html.button [
                prop.text "เพิ่ม"
                prop.onClick (fun _ -> setCount (count + 1))
            ]
            Html.button [
                prop.text "ลด"
                prop.onClick (fun _ -> setCount (count - 1))
            ]
            Html.button [
                prop.text "Reset"
                prop.onClick (fun _ -> setCount props.initialCount)
            ]
        ]
    ]
)
```

### Hooks ใน Feliz

```fsharp
open Feliz

// useState
let stateExample = React.functionComponent(fun () ->
    let name, setName = React.useState("")
    let email, setEmail = React.useState("")
    
    Html.div [
        Html.input [
            prop.type' "text"
            prop.placeholder "ชื่อ"
            prop.value name
            prop.onChange (fun e -> setName e.target.value)
        ]
        Html.input [
            prop.type' "email"
            prop.placeholder "Email"
            prop.value email
            prop.onChange (fun e -> setEmail e.target.value)
        ]
        Html.p (sprintf "สวัสดี, %s (%s)" name email)
    ]
)

// useEffect
let effectExample = React.functionComponent(fun () ->
    let data, setData = React.useState<string option>(None)
    let loading, setLoading = React.useState(true)
    
    React.useEffect(fun () ->
        async {
            do! Async.Sleep 1000  // จำลอง API call
            setData (Some "ข้อมูลจาก API")
            setLoading false
        } |> Async.StartImmediate
        
        // Cleanup function
        React.createDisposable(fun () -> ())
    , [||])  // empty deps = run once
    
    if loading then
        Html.div "กำลังโหลด..."
    else
        Html.div [
            match data with
            | Some d -> Html.p d
            | None -> Html.p "ไม่มีข้อมูล"
        ]
)

// useReducer
type Action =
    | Increment
    | Decrement
    | Reset
    | SetValue of int

let reducer state action =
    match action with
    | Increment -> state + 1
    | Decrement -> state - 1
    | Reset -> 0
    | SetValue n -> n

let reducerExample = React.functionComponent(fun () ->
    let state, dispatch = React.useReducer(reducer, 0)
    
    Html.div [
        Html.p (sprintf "State: %d" state)
        Html.button [
            prop.text "+"
            prop.onClick (fun _ -> dispatch Increment)
        ]
        Html.button [
            prop.text "-"
            prop.onClick (fun _ -> dispatch Decrement)
        ]
        Html.button [
            prop.text "Reset"
            prop.onClick (fun _ -> dispatch Reset)
        ]
    ]
)
```

---

## 5. Elmish Architecture

### MVU Pattern Overview

```
User Action → Message → Update → New Model → View → User Action
     ↑                                                    |
     └────────────────────────────────────────────────────┘
```

### การติดตั้ง Elmish

```xml
<PackageReference Include="Fable.Elmish" Version="4.*" />
<PackageReference Include="Fable.Elmish.React" Version="4.*" />
<PackageReference Include="Fable.Elmish.Debugger" Version="4.*" />
```

### Elmish App พื้นฐาน

```fsharp
module Counter

open Elmish
open Elmish.React
open Feliz

// MODEL - State ของ application
type Model = {
    Count: int
    Step: int
    History: int list
}

let init () =
    { Count = 0; Step = 1; History = [] }, Cmd.none

// MESSAGES - Events ที่เกิดขึ้น
type Msg =
    | Increment
    | Decrement
    | Reset
    | SetStep of int
    | Undo

// UPDATE - ฟังก์ชันที่อัปเดต state
let update msg model =
    match msg with
    | Increment ->
        let newCount = model.Count + model.Step
        { model with 
            Count = newCount
            History = model.Count :: model.History }
        , Cmd.none
        
    | Decrement ->
        let newCount = model.Count - model.Step
        { model with 
            Count = newCount
            History = model.Count :: model.History }
        , Cmd.none
        
    | Reset ->
        { model with Count = 0; History = [] }, Cmd.none
        
    | SetStep step ->
        { model with Step = step }, Cmd.none
        
    | Undo ->
        match model.History with
        | [] -> model, Cmd.none
        | prev :: rest ->
            { model with Count = prev; History = rest }, Cmd.none

// VIEW - Render ส่วน UI
let view model dispatch =
    Html.div [
        prop.className "app"
        prop.children [
            Html.h1 "Elmish Counter"
            
            Html.div [
                prop.className "counter-display"
                prop.children [
                    Html.span [
                        prop.className "count"
                        prop.text (string model.Count)
                    ]
                ]
            ]
            
            Html.div [
                prop.className "controls"
                prop.children [
                    Html.button [
                        prop.text "+"
                        prop.onClick (fun _ -> dispatch Increment)
                    ]
                    Html.button [
                        prop.text "-"
                        prop.onClick (fun _ -> dispatch Decrement)
                    ]
                    Html.button [
                        prop.text "Reset"
                        prop.onClick (fun _ -> dispatch Reset)
                        prop.disabled (model.Count = 0)
                    ]
                    Html.button [
                        prop.text "Undo"
                        prop.onClick (fun _ -> dispatch Undo)
                        prop.disabled (List.isEmpty model.History)
                    ]
                ]
            ]
            
            Html.div [
                prop.className "step-control"
                prop.children [
                    Html.label "Step: "
                    Html.input [
                        prop.type' "range"
                        prop.min 1
                        prop.max 10
                        prop.value model.Step
                        prop.onChange (fun e -> 
                            dispatch (SetStep (int e.target.value)))
                    ]
                    Html.span (string model.Step)
                ]
            ]
            
            if not (List.isEmpty model.History) then
                Html.div [
                    prop.className "history"
                    prop.children [
                        Html.h3 "History:"
                        Html.ul [
                            for h in model.History do
                                Html.li (string h)
                        ]
                    ]
                ]
        ]
    ]

// PROGRAM - เริ่มต้น Elmish app
Program.mkProgram init update view
|> Program.withReactSynchronous "app"
|> Program.run
```

---

## 6. Commands (Cmd)

### Commands คืออะไร

Commands เป็น side effects ที่จะถูก execute หลังจาก update function return

```fsharp
open Elmish

// Cmd.none - ไม่มี side effect
let update1 msg model =
    model, Cmd.none

// Cmd.ofMsg - dispatch message อีกอันทันที
let update2 msg model =
    match msg with
    | SomeAction ->
        model, Cmd.ofMsg AnotherAction

// Cmd.ofAsync - รัน async computation
type Msg =
    | LoadData
    | DataLoaded of string
    | LoadFailed of exn

let loadData () =
    async {
        let! response = Http.get "https://api.example.com/data"
        return response.Body
    }

let update msg model =
    match msg with
    | LoadData ->
        { model with Loading = true },
        Cmd.ofAsync loadData () DataLoaded LoadFailed
        
    | DataLoaded data ->
        { model with Data = Some data; Loading = false }, Cmd.none
        
    | LoadFailed ex ->
        { model with Error = Some ex.Message; Loading = false }, Cmd.none

// Cmd.batch - รัน commands หลายอัน
let update3 msg model =
    match msg with
    | Initialize ->
        model,
        Cmd.batch [
            Cmd.ofMsg LoadUsers
            Cmd.ofMsg LoadPosts
            Cmd.ofMsg LoadComments
        ]

// Cmd.ofPromise - รัน JavaScript Promise
open Fable.Core.JS

let fetchUsers () : JS.Promise<User list> =
    fetch "https://api.example.com/users" []
    |> Promise.bind (fun r -> r.json())

let update4 msg model =
    match msg with
    | LoadUsers ->
        model,
        Cmd.OfPromise.either fetchUsers () UsersLoaded LoadFailed
```

---

## 7. Routing ใน Fable Apps

### การติดตั้ง Feliz.Router

```xml
<PackageReference Include="Feliz.Router" Version="4.*" />
```

### Basic Routing

```fsharp
open Feliz
open Feliz.Router

// กำหนด routes
type Page =
    | Home
    | About
    | Users
    | UserDetail of int
    | NotFound

// Parse URL เป็น Page
let parsePage (segments: string list) =
    match segments with
    | [] -> Home
    | [ "about" ] -> About
    | [ "users" ] -> Users
    | [ "users"; Route.Int id ] -> UserDetail id
    | _ -> NotFound

// Model ที่มี current page
type Model = {
    CurrentPage: Page
    // ... other state
}

// Messages
type Msg =
    | NavigateTo of Page
    | UrlChanged of string list

// Update
let update msg model =
    match msg with
    | NavigateTo page ->
        let url = 
            match page with
            | Home -> "/"
            | About -> "/about"
            | Users -> "/users"
            | UserDetail id -> sprintf "/users/%d" id
            | NotFound -> "/404"
        model, Cmd.navigate url
        
    | UrlChanged segments ->
        { model with CurrentPage = parsePage segments }, Cmd.none

// View
let view model dispatch =
    React.router [
        router.onUrlChanged (fun segments -> dispatch (UrlChanged segments))
        router.children [
            // Navigation
            Html.nav [
                Html.a [
                    prop.href "/"
                    prop.text "Home"
                ]
                Html.a [
                    prop.href "/about"
                    prop.text "About"
                ]
                Html.a [
                    prop.href "/users"
                    prop.text "Users"
                ]
            ]
            
            // Page content
            match model.CurrentPage with
            | Home -> homePage model dispatch
            | About -> aboutPage model dispatch
            | Users -> usersPage model dispatch
            | UserDetail id -> userDetailPage id model dispatch
            | NotFound -> Html.div "404 - Page not found"
        ]
    ]
```

---

## 8. HTTP Calls จาก Browser

### ใช้ Fable.Fetch

```xml
<PackageReference Include="Fable.Fetch" Version="2.*" />
<PackageReference Include="Thoth.Json" Version="11.*" />
```

### HTTP GET

```fsharp
open Fetch
open Thoth.Json

type User = {
    Id: int
    Name: string
    Email: string
}

module User =
    let decoder : Decoder<User> =
        Decode.object (fun get -> {
            Id = get.Required.Field "id" Decode.int
            Name = get.Required.Field "name" Decode.string
            Email = get.Required.Field "email" Decode.string
        })

// GET request
let getUser (id: int) : Async<Result<User, string>> =
    async {
        try
            let! response = 
                fetch (sprintf "https://jsonplaceholder.typicode.com/users/%d" id) []
                |> Async.AwaitPromise
            
            if response.Ok then
                let! text = response.text() |> Async.AwaitPromise
                return Decode.fromString User.decoder text
            else
                return Error (sprintf "HTTP Error: %d" response.Status)
        with
        | ex -> return Error ex.Message
    }

// GET list
let getUsers () : Async<Result<User list, string>> =
    async {
        try
            let! response = 
                fetch "https://jsonplaceholder.typicode.com/users" []
                |> Async.AwaitPromise
            
            let! text = response.text() |> Async.AwaitPromise
            return Decode.fromString (Decode.list User.decoder) text
        with
        | ex -> return Error ex.Message
    }
```

### HTTP POST/PUT/DELETE

```fsharp
open Fetch
open Thoth.Json

type CreateUserRequest = {
    Name: string
    Email: string
}

module CreateUserRequest =
    let encoder (req: CreateUserRequest) =
        Encode.object [
            "name", Encode.string req.Name
            "email", Encode.string req.Email
        ]

// POST request
let createUser (req: CreateUserRequest) : Async<Result<User, string>> =
    async {
        try
            let body = 
                CreateUserRequest.encoder req
                |> Encode.toString 0
            
            let options = 
                requestProps [
                    Method HttpMethod.POST
                    Body (body |> unbox)
                    requestHeaders [
                        ContentType "application/json"
                    ]
                ]
            
            let! response = 
                fetch "https://jsonplaceholder.typicode.com/users" options
                |> Async.AwaitPromise
            
            let! text = response.text() |> Async.AwaitPromise
            return Decode.fromString User.decoder text
        with
        | ex -> return Error ex.Message
    }

// PUT request
let updateUser (id: int) (name: string) : Async<Result<User, string>> =
    async {
        try
            let body = sprintf """{"name": "%s"}""" name
            
            let options =
                requestProps [
                    Method HttpMethod.PUT
                    Body (body |> unbox)
                    requestHeaders [ContentType "application/json"]
                ]
            
            let! response =
                fetch (sprintf "https://jsonplaceholder.typicode.com/users/%d" id) options
                |> Async.AwaitPromise
            
            let! text = response.text() |> Async.AwaitPromise
            return Decode.fromString User.decoder text
        with
        | ex -> return Error ex.Message
    }

// DELETE request
let deleteUser (id: int) : Async<Result<unit, string>> =
    async {
        try
            let options = requestProps [Method HttpMethod.DELETE]
            
            let! response =
                fetch (sprintf "https://jsonplaceholder.typicode.com/users/%d" id) options
                |> Async.AwaitPromise
            
            if response.Ok then return Ok ()
            else return Error (sprintf "Delete failed: %d" response.Status)
        with
        | ex -> return Error ex.Message
    }
```

---

## 9. LocalStorage

```fsharp
open Browser.WebStorage
open Thoth.Json

// บันทึก data ลง LocalStorage
let saveToStorage<'T> (key: string) (encoder: Encoder<'T>) (data: 'T) : unit =
    let json = Encode.toString 0 (encoder data)
    localStorage.setItem(key, json)

// โหลด data จาก LocalStorage
let loadFromStorage<'T> (key: string) (decoder: Decoder<'T>) : Result<'T, string> =
    let json = localStorage.getItem(key)
    if isNull json then
        Error "No data found"
    else
        Decode.fromString decoder json

// ลบ data จาก LocalStorage
let removeFromStorage (key: string) : unit =
    localStorage.removeItem(key)

// Example: บันทึก user preferences
type Preferences = {
    Theme: string
    Language: string
    FontSize: int
}

module Preferences =
    let encoder (pref: Preferences) =
        Encode.object [
            "theme", Encode.string pref.Theme
            "language", Encode.string pref.Language
            "fontSize", Encode.int pref.FontSize
        ]
    
    let decoder : Decoder<Preferences> =
        Decode.object (fun get -> {
            Theme = get.Required.Field "theme" Decode.string
            Language = get.Required.Field "language" Decode.string
            FontSize = get.Required.Field "fontSize" Decode.int
        })
    
    let defaultPreferences = {
        Theme = "light"
        Language = "th"
        FontSize = 16
    }
    
    let save (pref: Preferences) =
        saveToStorage "preferences" encoder pref
    
    let load () =
        match loadFromStorage "preferences" decoder with
        | Ok pref -> pref
        | Error _ -> defaultPreferences

// ใช้ใน Elmish
type Msg =
    | SavePreferences of Preferences
    | LoadPreferences

let update msg model =
    match msg with
    | SavePreferences pref ->
        Preferences.save pref
        { model with Preferences = pref }, Cmd.none
    
    | LoadPreferences ->
        let pref = Preferences.load()
        { model with Preferences = pref }, Cmd.none
```

---

## 10. CSS-in-F#

### ใช้ Feliz สำหรับ Inline Styles

```fsharp
open Feliz

let styledComponent = React.functionComponent(fun () ->
    Html.div [
        prop.style [
            style.backgroundColor "#f0f0f0"
            style.padding 20
            style.borderRadius 8
            style.boxShadow "0 2px 4px rgba(0,0,0,0.1)"
        ]
        prop.children [
            Html.h2 [
                prop.style [
                    style.color "#333"
                    style.fontFamily "Arial, sans-serif"
                    style.fontSize 24
                    style.marginBottom 10
                ]
                prop.text "Styled Header"
            ]
            Html.p [
                prop.style [
                    style.color "#666"
                    style.lineHeight 1.6
                ]
                prop.text "นี่คือ styled paragraph"
            ]
        ]
    ]
)
```

### CSS Modules กับ Fable

```fsharp
open Fable.Core
open Fable.Core.JsInterop

// Import CSS module
[<ImportDefault("./App.module.css")>]
let styles : obj = jsNative

// ใช้ CSS class names
let app = React.functionComponent(fun () ->
    Html.div [
        prop.className !!styles?container
        prop.children [
            Html.h1 [
                prop.className !!styles?title
                prop.text "Hello"
            ]
            Html.button [
                prop.className !!styles?button
                prop.text "Click me"
            ]
        ]
    ]
)
```

### Emotion CSS-in-JS กับ Fable

```fsharp
open Fable.Core
open Fable.Core.JsInterop

// Import emotion
[<Import("css", "@emotion/css")>]
let css (styles: obj) : string = jsNative

// สร้าง style
let containerStyle = 
    css {|
        backgroundColor = "#fff"
        padding = "20px"
        borderRadius = "8px"
        maxWidth = "1200px"
        margin = "0 auto"
    |}

let buttonStyle =
    css {|
        backgroundColor = "#007bff"
        color = "#fff"
        border = "none"
        padding = "10px 20px"
        borderRadius = "4px"
        cursor = "pointer"
        ``&:hover`` = {| backgroundColor = "#0056b3" |}
    |}

// ใช้ใน component
let styledApp = React.functionComponent(fun () ->
    Html.div [
        prop.className containerStyle
        prop.children [
            Html.button [
                prop.className buttonStyle
                prop.text "Styled Button"
            ]
        ]
    ]
)
```

---

## 11. Fable Bindings

### สร้าง Bindings สำหรับ JavaScript Library

```fsharp
open Fable.Core
open Fable.Core.JsInterop

// Binding สำหรับ moment.js
[<Import("default", "moment")>]
let moment : obj = jsNative

// Type-safe binding
type IMoment =
    abstract format: string -> string
    abstract fromNow: unit -> string
    abstract add: int * string -> IMoment
    abstract subtract: int * string -> IMoment

[<Import("default", "moment")>]
let createMoment : obj -> IMoment = jsNative

let now = createMoment (System.DateTime.Now)
let formatted = now.format("YYYY-MM-DD")
let relative = now.fromNow()

// Binding สำหรับ Chart.js
type ChartConfig = {
    ``type``: string
    data: obj
    options: obj
}

[<Import("Chart", "chart.js")>]
type Chart(ctx: obj, config: obj) =
    member _.update() : unit = jsNative
    member _.destroy() : unit = jsNative

// ใช้งาน
let createChart (canvasId: string) =
    let canvas = Browser.Dom.document.getElementById(canvasId)
    let config = {|
        ``type`` = "bar"
        data = {|
            labels = [| "Jan"; "Feb"; "Mar" |]
            datasets = [|
                {|
                    label = "Sales"
                    data = [| 100; 200; 150 |]
                    backgroundColor = "rgba(75, 192, 192, 0.2)"
                |}
            |]
        |}
    |}
    Chart(canvas, config)
```

---

## 12. Complete Fable Web App Example

### Todo Application

```fsharp
// Todo.fs - Complete Todo App with Fable + Elmish + Feliz
module Todo

open Elmish
open Elmish.React
open Feliz
open Browser.WebStorage
open Thoth.Json

// --- Domain Types ---

type TodoId = int

type Priority =
    | Low
    | Medium
    | High

type TodoItem = {
    Id: TodoId
    Title: string
    Completed: bool
    Priority: Priority
    CreatedAt: System.DateTime
    Tags: string list
}

type Filter =
    | All
    | Active
    | Completed
    | ByPriority of Priority

// --- Model ---

type Model = {
    Todos: TodoItem list
    NewTodoText: string
    NewTodoPriority: Priority
    Filter: Filter
    EditingId: TodoId option
    EditingText: string
    SearchQuery: string
    NextId: TodoId
    DarkMode: bool
}

let init () =
    let savedTodos = 
        try
            let json = localStorage.getItem("todos")
            if isNull json then []
            else
                match Decode.fromString (Decode.list todoDecoder) json with
                | Ok todos -> todos
                | Error _ -> []
        with _ -> []
    
    {
        Todos = savedTodos
        NewTodoText = ""
        NewTodoPriority = Medium
        Filter = All
        EditingId = None
        EditingText = ""
        SearchQuery = ""
        NextId = (savedTodos |> List.map (fun t -> t.Id) |> List.fold max 0) + 1
        DarkMode = false
    }, Cmd.none

// --- Messages ---

type Msg =
    | UpdateNewTodoText of string
    | UpdateNewTodoPriority of Priority
    | AddTodo
    | ToggleTodo of TodoId
    | DeleteTodo of TodoId
    | StartEditing of TodoId
    | UpdateEditingText of string
    | SaveEditing
    | CancelEditing
    | SetFilter of Filter
    | UpdateSearchQuery of string
    | ClearCompleted
    | ToggleDarkMode
    | AddTag of TodoId * string
    | RemoveTag of TodoId * string

// --- JSON Encoding/Decoding ---

and todoDecoder : Decoder<TodoItem> =
    Decode.object (fun get -> {
        Id = get.Required.Field "id" Decode.int
        Title = get.Required.Field "title" Decode.string
        Completed = get.Required.Field "completed" Decode.bool
        Priority = 
            match get.Required.Field "priority" Decode.string with
            | "high" -> High
            | "low" -> Low
            | _ -> Medium
        CreatedAt = get.Required.Field "createdAt" Decode.datetime
        Tags = get.Optional.Field "tags" (Decode.list Decode.string) |> Option.defaultValue []
    })

let encodeTodo (todo: TodoItem) =
    let priorityStr =
        match todo.Priority with
        | High -> "high"
        | Medium -> "medium"
        | Low -> "low"
    
    Encode.object [
        "id", Encode.int todo.Id
        "title", Encode.string todo.Title
        "completed", Encode.bool todo.Completed
        "priority", Encode.string priorityStr
        "createdAt", Encode.datetime todo.CreatedAt
        "tags", todo.Tags |> List.map Encode.string |> Encode.list
    ]

let saveTodos (todos: TodoItem list) =
    let json = todos |> List.map encodeTodo |> Encode.list |> Encode.toString 0
    localStorage.setItem("todos", json)

// --- Update ---

let update msg model =
    match msg with
    | UpdateNewTodoText text ->
        { model with NewTodoText = text }, Cmd.none
    
    | UpdateNewTodoPriority priority ->
        { model with NewTodoPriority = priority }, Cmd.none
    
    | AddTodo when model.NewTodoText.Trim() <> "" ->
        let newTodo = {
            Id = model.NextId
            Title = model.NewTodoText.Trim()
            Completed = false
            Priority = model.NewTodoPriority
            CreatedAt = System.DateTime.Now
            Tags = []
        }
        let todos = model.Todos @ [newTodo]
        saveTodos todos
        { model with 
            Todos = todos
            NewTodoText = ""
            NextId = model.NextId + 1 }, Cmd.none
    
    | AddTodo -> model, Cmd.none
    
    | ToggleTodo id ->
        let todos = 
            model.Todos 
            |> List.map (fun t -> 
                if t.Id = id then { t with Completed = not t.Completed }
                else t)
        saveTodos todos
        { model with Todos = todos }, Cmd.none
    
    | DeleteTodo id ->
        let todos = model.Todos |> List.filter (fun t -> t.Id <> id)
        saveTodos todos
        { model with Todos = todos }, Cmd.none
    
    | StartEditing id ->
        let title = 
            model.Todos 
            |> List.tryFind (fun t -> t.Id = id)
            |> Option.map (fun t -> t.Title)
            |> Option.defaultValue ""
        { model with EditingId = Some id; EditingText = title }, Cmd.none
    
    | UpdateEditingText text ->
        { model with EditingText = text }, Cmd.none
    
    | SaveEditing ->
        match model.EditingId with
        | Some id when model.EditingText.Trim() <> "" ->
            let todos =
                model.Todos
                |> List.map (fun t ->
                    if t.Id = id then { t with Title = model.EditingText.Trim() }
                    else t)
            saveTodos todos
            { model with Todos = todos; EditingId = None; EditingText = "" }, Cmd.none
        | _ ->
            { model with EditingId = None; EditingText = "" }, Cmd.none
    
    | CancelEditing ->
        { model with EditingId = None; EditingText = "" }, Cmd.none
    
    | SetFilter filter ->
        { model with Filter = filter }, Cmd.none
    
    | UpdateSearchQuery query ->
        { model with SearchQuery = query }, Cmd.none
    
    | ClearCompleted ->
        let todos = model.Todos |> List.filter (fun t -> not t.Completed)
        saveTodos todos
        { model with Todos = todos }, Cmd.none
    
    | ToggleDarkMode ->
        { model with DarkMode = not model.DarkMode }, Cmd.none
    
    | AddTag (id, tag) ->
        let todos =
            model.Todos
            |> List.map (fun t ->
                if t.Id = id && not (List.contains tag t.Tags) then
                    { t with Tags = t.Tags @ [tag] }
                else t)
        saveTodos todos
        { model with Todos = todos }, Cmd.none
    
    | RemoveTag (id, tag) ->
        let todos =
            model.Todos
            |> List.map (fun t ->
                if t.Id = id then
                    { t with Tags = t.Tags |> List.filter (fun t -> t <> tag) }
                else t)
        saveTodos todos
        { model with Todos = todos }, Cmd.none

// --- Helpers ---

let filteredTodos model =
    let byFilter =
        match model.Filter with
        | All -> model.Todos
        | Active -> model.Todos |> List.filter (fun t -> not t.Completed)
        | Completed -> model.Todos |> List.filter (fun t -> t.Completed)
        | ByPriority p -> model.Todos |> List.filter (fun t -> t.Priority = p)
    
    if model.SearchQuery = "" then byFilter
    else
        byFilter 
        |> List.filter (fun t -> 
            t.Title.ToLower().Contains(model.SearchQuery.ToLower()))

// --- View Components ---

let priorityBadge (priority: Priority) =
    let text, color =
        match priority with
        | High -> "สูง", "#dc3545"
        | Medium -> "กลาง", "#ffc107"
        | Low -> "ต่ำ", "#28a745"
    
    Html.span [
        prop.style [
            style.backgroundColor color
            style.color "white"
            style.padding (2, 6)
            style.borderRadius 4
            style.fontSize 12
            style.marginRight 8
        ]
        prop.text text
    ]

let todoItemView (todo: TodoItem) (editing: bool) (dispatch: Msg -> unit) =
    Html.li [
        prop.key todo.Id
        prop.style [
            style.display.flex
            style.alignItems.center
            style.padding 12
            style.marginBottom 8
            style.backgroundColor (if todo.Completed then "#f8f9fa" else "white")
            style.borderRadius 8
            style.boxShadow "0 1px 3px rgba(0,0,0,0.1)"
            style.opacity (if todo.Completed then 0.7 else 1.0)
        ]
        prop.children [
            // Checkbox
            Html.input [
                prop.type' "checkbox"
                prop.checked' todo.Completed
                prop.onChange (fun _ -> dispatch (ToggleTodo todo.Id))
                prop.style [style.marginRight 12; style.cursor "pointer"]
            ]
            
            // Priority badge
            priorityBadge todo.Priority
            
            // Title or edit input
            if editing then
                Html.input [
                    prop.type' "text"
                    prop.value (
                        // This would need model.EditingText in real app
                        todo.Title
                    )
                    prop.autoFocus true
                    prop.style [
                        style.flexGrow 1
                        style.padding (4, 8)
                        style.border "1px solid #007bff"
                        style.borderRadius 4
                    ]
                    prop.onChange (fun e -> dispatch (UpdateEditingText e.target.value))
                    prop.onKeyDown (fun e ->
                        if e.key = "Enter" then dispatch SaveEditing
                        elif e.key = "Escape" then dispatch CancelEditing)
                ]
            else
                Html.span [
                    prop.style [
                        style.flexGrow 1
                        style.textDecoration (if todo.Completed then "line-through" else "none")
                        style.cursor "pointer"
                    ]
                    prop.text todo.Title
                    prop.onDoubleClick (fun _ -> dispatch (StartEditing todo.Id))
                ]
            
            // Tags
            Html.div [
                prop.style [style.display.flex; style.gap 4; style.marginLeft 8]
                prop.children [
                    for tag in todo.Tags do
                        Html.span [
                            prop.style [
                                style.backgroundColor "#e9ecef"
                                style.padding (2, 6)
                                style.borderRadius 4
                                style.fontSize 12
                                style.cursor "pointer"
                            ]
                            prop.text (sprintf "× %s" tag)
                            prop.onClick (fun _ -> dispatch (RemoveTag (todo.Id, tag)))
                        ]
                ]
            ]
            
            // Delete button
            Html.button [
                prop.style [
                    style.marginLeft 8
                    style.backgroundColor "#dc3545"
                    style.color "white"
                    style.border "none"
                    style.borderRadius 4
                    style.padding (4, 8)
                    style.cursor "pointer"
                ]
                prop.text "ลบ"
                prop.onClick (fun _ -> dispatch (DeleteTodo todo.Id))
            ]
        ]
    ]

// --- Main View ---

let view (model: Model) (dispatch: Msg -> unit) =
    Html.div [
        prop.style [
            style.backgroundColor (if model.DarkMode then "#1a1a2e" else "#f0f2f5")
            style.minHeight (length.vh 100)
            style.padding 20
            style.fontFamily "Arial, sans-serif"
        ]
        prop.children [
            Html.div [
                prop.style [
                    style.maxWidth 800
                    style.margin (0, length.auto)
                ]
                prop.children [
                    // Header
                    Html.div [
                        prop.style [
                            style.display.flex
                            style.justifyContent.spaceBetween
                            style.alignItems.center
                            style.marginBottom 20
                        ]
                        prop.children [
                            Html.h1 [
                                prop.style [
                                    style.color (if model.DarkMode then "#e0e0e0" else "#333")
                                    style.margin 0
                                ]
                                prop.text "✓ Todo App (F# + Fable)"
                            ]
                            Html.button [
                                prop.style [
                                    style.backgroundColor "transparent"
                                    style.border "none"
                                    style.fontSize 24
                                    style.cursor "pointer"
                                ]
                                prop.text (if model.DarkMode then "☀️" else "🌙")
                                prop.onClick (fun _ -> dispatch ToggleDarkMode)
                            ]
                        ]
                    ]
                    
                    // Add Todo Form
                    Html.div [
                        prop.style [
                            style.display.flex
                            style.gap 8
                            style.marginBottom 20
                        ]
                        prop.children [
                            Html.input [
                                prop.type' "text"
                                prop.placeholder "เพิ่ม todo ใหม่..."
                                prop.value model.NewTodoText
                                prop.onChange (fun e -> dispatch (UpdateNewTodoText e.target.value))
                                prop.onKeyDown (fun e -> 
                                    if e.key = "Enter" then dispatch AddTodo)
                                prop.style [
                                    style.flexGrow 1
                                    style.padding (10, 15)
                                    style.borderRadius 8
                                    style.border "1px solid #ddd"
                                    style.fontSize 16
                                ]
                            ]
                            Html.select [
                                prop.value (
                                    match model.NewTodoPriority with
                                    | High -> "high"
                                    | Medium -> "medium"
                                    | Low -> "low"
                                )
                                prop.onChange (fun e ->
                                    let priority =
                                        match e.target.value with
                                        | "high" -> High
                                        | "low" -> Low
                                        | _ -> Medium
                                    dispatch (UpdateNewTodoPriority priority))
                                prop.style [
                                    style.padding (10, 15)
                                    style.borderRadius 8
                                    style.border "1px solid #ddd"
                                ]
                                prop.children [
                                    Html.option [prop.value "low"; prop.text "ต่ำ"]
                                    Html.option [prop.value "medium"; prop.text "กลาง"]
                                    Html.option [prop.value "high"; prop.text "สูง"]
                                ]
                            ]
                            Html.button [
                                prop.style [
                                    style.backgroundColor "#007bff"
                                    style.color "white"
                                    style.border "none"
                                    style.borderRadius 8
                                    style.padding (10, 20)
                                    style.cursor "pointer"
                                    style.fontSize 16
                                ]
                                prop.text "เพิ่ม"
                                prop.onClick (fun _ -> dispatch AddTodo)
                            ]
                        ]
                    ]
                    
                    // Stats
                    let totalCount = model.Todos.Length
                    let completedCount = model.Todos |> List.filter (fun t -> t.Completed) |> List.length
                    let activeCount = totalCount - completedCount
                    
                    Html.div [
                        prop.style [
                            style.display.flex
                            style.gap 16
                            style.marginBottom 16
                            style.color "#666"
                        ]
                        prop.children [
                            Html.span (sprintf "ทั้งหมด: %d" totalCount)
                            Html.span (sprintf "ค้างอยู่: %d" activeCount)
                            Html.span (sprintf "เสร็จแล้ว: %d" completedCount)
                        ]
                    ]
                    
                    // Filter buttons
                    Html.div [
                        prop.style [
                            style.display.flex
                            style.gap 8
                            style.marginBottom 16
                            style.flexWrap.wrap
                        ]
                        prop.children [
                            for (filter, label) in [All, "ทั้งหมด"; Active, "ค้างอยู่"; Completed, "เสร็จแล้ว"; ByPriority High, "สำคัญมาก"; ByPriority Medium, "กลาง"; ByPriority Low, "ต่ำ"] do
                                Html.button [
                                    prop.style [
                                        style.padding (6, 12)
                                        style.borderRadius 4
                                        style.border "1px solid #007bff"
                                        style.backgroundColor (if model.Filter = filter then "#007bff" else "white")
                                        style.color (if model.Filter = filter then "white" else "#007bff")
                                        style.cursor "pointer"
                                    ]
                                    prop.text label
                                    prop.onClick (fun _ -> dispatch (SetFilter filter))
                                ]
                        ]
                    ]
                    
                    // Search
                    Html.input [
                        prop.type' "search"
                        prop.placeholder "ค้นหา todos..."
                        prop.value model.SearchQuery
                        prop.onChange (fun e -> dispatch (UpdateSearchQuery e.target.value))
                        prop.style [
                            style.width (length.percent 100)
                            style.padding (10, 15)
                            style.borderRadius 8
                            style.border "1px solid #ddd"
                            style.marginBottom 16
                            style.boxSizing.borderBox
                        ]
                    ]
                    
                    // Todo list
                    let todos = filteredTodos model
                    
                    if List.isEmpty todos then
                        Html.div [
                            prop.style [
                                style.textAlign.center
                                style.padding 40
                                style.color "#999"
                            ]
                            prop.text "ไม่มี todos ที่ตรงกับเงื่อนไข"
                        ]
                    else
                        Html.ul [
                            prop.style [style.listStyleType.none; style.padding 0; style.margin 0]
                            prop.children [
                                for todo in todos do
                                    todoItemView todo (model.EditingId = Some todo.Id) dispatch
                            ]
                        ]
                    
                    // Clear completed button
                    if completedCount > 0 then
                        Html.button [
                            prop.style [
                                style.marginTop 16
                                style.backgroundColor "#6c757d"
                                style.color "white"
                                style.border "none"
                                style.borderRadius 8
                                style.padding (8, 16)
                                style.cursor "pointer"
                            ]
                            prop.text (sprintf "ลบที่เสร็จแล้ว (%d)" completedCount)
                            prop.onClick (fun _ -> dispatch ClearCompleted)
                        ]
                ]
            ]
        ]
    ]

// --- Program ---

Program.mkProgram init update view
|> Program.withReactSynchronous "app"
|> Program.run
```

---

## 13. การ Deploy Fable Application

### Build สำหรับ Production

```bash
# Build production bundle
npm run build

# Output จะอยู่ใน /dist folder
ls dist/
# index.html
# bundle.js
# bundle.js.map
```

### webpack.config.js สำหรับ Production

```javascript
const path = require("path");
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin");

module.exports = (env, options) => {
    const isProduction = options.mode === "production";
    
    return {
        entry: "./src/App.fsproj",
        output: {
            path: path.join(__dirname, "dist"),
            filename: isProduction ? "[name].[contenthash].js" : "bundle.js"
        },
        module: {
            rules: [
                {
                    test: /\.fs(x|proj)?$/,
                    use: {
                        loader: "fable-loader",
                        options: {
                            optimize: isProduction
                        }
                    }
                },
                {
                    test: /\.css$/,
                    use: [
                        isProduction ? MiniCssExtractPlugin.loader : "style-loader",
                        "css-loader"
                    ]
                }
            ]
        },
        plugins: [
            new HtmlWebpackPlugin({
                template: "./public/index.html"
            }),
            ...(isProduction ? [new MiniCssExtractPlugin({
                filename: "[name].[contenthash].css"
            })] : [])
        ],
        devServer: {
            port: 8080,
            hot: true,
            historyApiFallback: true
        },
        optimization: isProduction ? {
            splitChunks: {
                chunks: "all"
            }
        } : {}
    };
};
```

### Deploy ไปยัง GitHub Pages

```bash
# เพิ่ม gh-pages package
npm install --save-dev gh-pages

# เพิ่มใน package.json
{
  "scripts": {
    "deploy": "npm run build && gh-pages -d dist"
  }
}

# Deploy
npm run deploy
```

---

## สรุป (Summary)

Fable เป็น powerful tool สำหรับการพัฒนา web applications ด้วย F# โดยมี features หลักๆ คือ:

1. **Type Safety**: ใช้ F# type system ป้องกัน bugs
2. **Functional Programming**: Pattern matching, immutable data
3. **React Integration**: ผ่าน Feliz library
4. **Elmish Architecture**: MVU pattern สำหรับ predictable state management
5. **JavaScript Interop**: สามารถใช้ JavaScript libraries ได้
6. **LocalStorage**: บันทึก state ระหว่าง sessions

การพัฒนา Fable app ทำให้เราได้ประโยชน์ทั้งจาก F# ecosystem และ JavaScript ecosystem พร้อมกัน

---

*ไปต่อที่ Part 102: Elmish Architecture ในเชิงลึก*
