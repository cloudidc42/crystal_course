# Part 103 - Feliz - React DSL for F#

## บทนำ (Introduction)

Feliz คือ React DSL (Domain-Specific Language) สำหรับ F# ที่ทำให้การเขียน React components ด้วย F# เป็นเรื่องง่ายและมี type safety สูง

---

## 1. Feliz Setup

### ติดตั้ง Feliz

```xml
<!-- App.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net6.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <Compile Include="App.fs" />
  </ItemGroup>
  <ItemGroup>
    <PackageReference Include="Fable.Core" Version="4.*" />
    <PackageReference Include="Feliz" Version="2.*" />
    <PackageReference Include="Feliz.Router" Version="4.*" />
    <PackageReference Include="Feliz.UseDeferred" Version="2.*" />
  </ItemGroup>
</Project>
```

### package.json

```json
{
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0"
  },
  "devDependencies": {
    "fable-compiler": "^4.3.0",
    "fable-loader": "^2.1.8",
    "webpack": "^5.80.0",
    "webpack-dev-server": "^4.15.0"
  }
}
```

### index.html

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Feliz App</title>
</head>
<body>
    <div id="app"></div>
    <script src="bundle.js"></script>
</body>
</html>
```

### App.fs entry point

```fsharp
module App

open Feliz
open Browser.Dom

let root = ReactDOM.createRoot(document.getElementById("app"))
root.render(Html.div "Hello from Feliz!")
```

---

## 2. HTML Elements as F# Functions

### Basic HTML Elements

```fsharp
open Feliz

// ===== Block Elements =====

let basicElements = 
    Html.div [
        // Headings
        Html.h1 "หัวข้อใหญ่"
        Html.h2 "หัวข้อรอง"
        Html.h3 "หัวข้อย่อย"
        Html.h4 "หัวข้อ 4"
        Html.h5 "หัวข้อ 5"
        Html.h6 "หัวข้อ 6"
        
        // Paragraphs
        Html.p "นี่คือ paragraph"
        
        // Division
        Html.div [
            Html.span "ข้อความใน div"
        ]
        
        // Semantic elements
        Html.section [
            Html.article [
                Html.header "Header ของ article"
                Html.main "เนื้อหาหลัก"
                Html.footer "Footer ของ article"
            ]
        ]
        
        Html.aside "เนื้อหาข้างเคียง"
        Html.nav "Navigation"
        
        // Lists
        Html.ul [
            Html.li "Item 1"
            Html.li "Item 2"
            Html.li "Item 3"
        ]
        
        Html.ol [
            Html.li "อันดับ 1"
            Html.li "อันดับ 2"
            Html.li "อันดับ 3"
        ]
        
        Html.dl [
            Html.dt "คำศัพท์"
            Html.dd "คำนิยาม"
        ]
        
        // Table
        Html.table [
            Html.thead [
                Html.tr [
                    Html.th "ชื่อ"
                    Html.th "อายุ"
                    Html.th "เมือง"
                ]
            ]
            Html.tbody [
                Html.tr [
                    Html.td "สมชาย"
                    Html.td "30"
                    Html.td "กรุงเทพ"
                ]
                Html.tr [
                    Html.td "สมหญิง"
                    Html.td "25"
                    Html.td "เชียงใหม่"
                ]
            ]
            Html.tfoot [
                Html.tr [
                    Html.td [ prop.colSpan 3; prop.text "Footer" ]
                ]
            ]
        ]
        
        // Pre-formatted text
        Html.pre """
let hello () =
    printfn "Hello, World!"
"""
        
        // Code
        Html.code "let x = 42"
        
        // Quote
        Html.blockquote "ความรู้คืออำนาจ"
        
        // Horizontal rule
        Html.hr []
        
        // Line break
        Html.br []
    ]

// ===== Inline Elements =====

let inlineElements =
    Html.p [
        Html.text "ข้อความ "
        Html.strong "หนา"
        Html.text " และ "
        Html.em "ตัวเอียง"
        Html.text " และ "
        Html.code "code"
        Html.text " และ "
        Html.a [
            prop.href "https://fsharp.org"
            prop.target "_blank"
            prop.text "link"
        ]
        Html.text " และ "
        Html.abbr [
            prop.title "HyperText Markup Language"
            prop.text "HTML"
        ]
    ]

// ===== Media Elements =====

let mediaElements =
    Html.div [
        Html.img [
            prop.src "/images/logo.png"
            prop.alt "Logo"
            prop.width 200
            prop.height 100
        ]
        
        Html.video [
            prop.src "/videos/demo.mp4"
            prop.controls true
            prop.autoPlay false
            prop.width 640
            prop.height 480
        ]
        
        Html.audio [
            prop.src "/audio/music.mp3"
            prop.controls true
        ]
        
        Html.iframe [
            prop.src "https://example.com"
            prop.width 800
            prop.height 600
        ]
    ]
```

---

## 3. Props and Attributes

### HTML Properties

```fsharp
open Feliz

// ===== Common Props =====

let propsExample =
    Html.div [
        // ID and Class
        prop.id "my-div"
        prop.className "container active"
        prop.classes ["container"; "active"; "special"]
        
        // Data attributes
        prop.custom("data-id", "123")
        prop.custom("data-name", "test")
        
        // ARIA
        prop.ariaLabel "Description"
        prop.ariaHidden true
        prop.role "button"
        
        // Children
        prop.children [
            Html.span "Child 1"
            Html.span "Child 2"
        ]
        
        // Single child
        prop.child (Html.p "Single child")
        
        // Text content
        prop.text "Text content"
        
        // Style
        prop.style [
            style.backgroundColor "blue"
            style.color "white"
        ]
    ]

// ===== Input Props =====

let inputProps =
    Html.div [
        // Text input
        Html.input [
            prop.type' "text"
            prop.id "name"
            prop.name "name"
            prop.value "Default value"
            prop.placeholder "Enter text"
            prop.required true
            prop.readOnly false
            prop.disabled false
            prop.maxLength 100
            prop.autoComplete "name"
            prop.autoFocus true
            prop.spellCheck true
        ]
        
        // Number input
        Html.input [
            prop.type' "number"
            prop.min 0
            prop.max 100
            prop.step 1
            prop.value 50
        ]
        
        // Checkbox
        Html.input [
            prop.type' "checkbox"
            prop.id "agree"
            prop.checked' true
            prop.defaultChecked false
        ]
        
        // Radio
        Html.input [
            prop.type' "radio"
            prop.name "option"
            prop.value "option1"
        ]
        
        // File
        Html.input [
            prop.type' "file"
            prop.accept ".jpg,.png,.pdf"
            prop.multiple true
        ]
        
        // Select
        Html.select [
            prop.value "option2"
            prop.multiple false
            prop.size 3
            prop.children [
                Html.option [prop.value "option1"; prop.text "ตัวเลือก 1"]
                Html.option [prop.value "option2"; prop.text "ตัวเลือก 2"]
                Html.option [prop.value "option3"; prop.text "ตัวเลือก 3"; prop.disabled true]
            ]
        ]
        
        // Textarea
        Html.textarea [
            prop.rows 4
            prop.cols 50
            prop.value "เนื้อหา"
            prop.placeholder "กรอกข้อความ"
            prop.resize.none
        ]
        
        // Button
        Html.button [
            prop.type' "submit"
            prop.text "Submit"
            prop.disabled false
        ]
    ]

// ===== Link Props =====

let linkProps =
    Html.div [
        Html.a [
            prop.href "https://example.com"
            prop.target "_blank"
            prop.rel "noopener noreferrer"
            prop.download "file.pdf"
            prop.text "Download"
        ]
    ]
```

---

## 4. CSS Styling

### Inline Styles

```fsharp
open Feliz

// ===== Basic Styles =====

let basicStyles =
    Html.div [
        prop.style [
            // Colors
            style.color "#333"
            style.backgroundColor "#f0f0f0"
            style.borderColor "#ccc"
            
            // Typography
            style.fontSize 16
            style.fontWeight.bold
            style.fontFamily "Arial, sans-serif"
            style.fontStyle.italic
            style.textDecoration.underline
            style.textAlign.center
            style.lineHeight 1.6
            style.letterSpacing 1
            style.textTransform.uppercase
            style.whiteSpace.nowrap
            
            // Box Model
            style.width 300
            style.height 200
            style.minWidth 100
            style.maxWidth 500
            style.padding 20
            style.paddingTop 10
            style.paddingRight 15
            style.paddingBottom 10
            style.paddingLeft 15
            style.margin 10
            style.marginTop 5
            style.marginRight (length.auto)
            style.marginBottom 5
            style.marginLeft (length.auto)
            
            // Border
            style.border "1px solid #ddd"
            style.borderRadius 8
            style.borderTopLeftRadius 4
            style.borderStyle.dashed
            style.borderWidth 2
            style.outline "none"
            
            // Display
            style.display.flex
            style.display.grid
            style.display.block
            style.display.none
            style.display.inlineFlex
            style.visibility.hidden
            style.opacity 0.5
            
            // Flexbox
            style.flexDirection.row
            style.flexWrap.wrap
            style.justifyContent.center
            style.alignItems.center
            style.alignContent.spaceBetween
            style.flexGrow 1
            style.flexShrink 0
            style.flexBasis 100
            style.gap 16
            
            // Grid
            style.gridTemplateColumns "repeat(3, 1fr)"
            style.gridTemplateRows "auto"
            style.gridColumn "1 / 3"
            style.gridRow "1 / 2"
            style.gridGap 16
            
            // Position
            style.position.relative
            style.position.absolute
            style.position.fixed
            style.position.sticky
            style.top 0
            style.right 0
            style.bottom 0
            style.left 0
            style.zIndex 100
            
            // Shadow & Effects
            style.boxShadow "0 2px 8px rgba(0,0,0,0.1)"
            style.textShadow "1px 1px 2px rgba(0,0,0,0.3)"
            style.filter "blur(4px)"
            style.transform "rotate(45deg)"
            style.transition "all 0.3s ease"
            
            // Overflow
            style.overflow.hidden
            style.overflowX.auto
            style.overflowY.scroll
            
            // Cursor
            style.cursor "pointer"
            style.cursor.grab
            
            // Object fit (for images)
            style.objectFit.cover
            style.objectFit.contain
        ]
    ]

// ===== Responsive Styles =====

// ใช้ inline styles สำหรับ responsive design
let responsiveComponent =
    React.functionComponent(fun () ->
        let windowWidth, setWindowWidth = React.useState(Browser.Dom.window.innerWidth)
        
        React.useEffect(fun () ->
            let handler _ = setWindowWidth Browser.Dom.window.innerWidth
            Browser.Dom.window.addEventListener("resize", handler)
            React.createDisposable(fun () ->
                Browser.Dom.window.removeEventListener("resize", handler))
        , [||])
        
        let isMobile = windowWidth < 768
        let isTablet = windowWidth >= 768 && windowWidth < 1024
        
        Html.div [
            prop.style [
                style.display.grid
                style.gridTemplateColumns (
                    if isMobile then "1fr"
                    elif isTablet then "repeat(2, 1fr)"
                    else "repeat(3, 1fr)"
                )
                style.gap 16
                style.padding (if isMobile then 8 else 24)
            ]
            prop.children [
                for i in 1..6 do
                    Html.div [
                        prop.key i
                        prop.style [
                            style.backgroundColor "#f0f0f0"
                            style.padding 16
                            style.borderRadius 8
                        ]
                        prop.text (sprintf "Card %d" i)
                    ]
            ]
        ]
    )
```

---

## 5. Event Handlers

### Event Types

```fsharp
open Feliz
open Browser.Types

// ===== Mouse Events =====

let mouseEvents =
    Html.div [
        prop.onClick (fun (e: MouseEvent) ->
            printfn "Clicked at %d, %d" e.clientX e.clientY)
        
        prop.onDoubleClick (fun e ->
            printfn "Double clicked")
        
        prop.onMouseDown (fun e ->
            printfn "Mouse button %d pressed" e.button)
        
        prop.onMouseUp (fun e -> 
            printfn "Mouse released")
        
        prop.onMouseMove (fun e ->
            printfn "Mouse at %d, %d" e.clientX e.clientY)
        
        prop.onMouseEnter (fun _ -> printfn "Mouse entered")
        
        prop.onMouseLeave (fun _ -> printfn "Mouse left")
        
        prop.onContextMenu (fun e ->
            e.preventDefault()
            printfn "Right clicked")
    ]

// ===== Keyboard Events =====

let keyboardEvents =
    Html.input [
        prop.onKeyDown (fun (e: KeyboardEvent) ->
            match e.key with
            | "Enter" -> printfn "Enter pressed"
            | "Escape" -> printfn "Escape pressed"
            | "Tab" -> printfn "Tab pressed"
            | key -> printfn "Key: %s" key
            
            if e.ctrlKey then printfn "Ctrl is held"
            if e.shiftKey then printfn "Shift is held"
            if e.altKey then printfn "Alt is held"
        )
        
        prop.onKeyUp (fun e -> printfn "Key up: %s" e.key)
        
        prop.onKeyPress (fun e -> printfn "Key press: %s" e.key)
    ]

// ===== Form Events =====

let formEvents =
    Html.form [
        prop.onSubmit (fun e ->
            e.preventDefault()
            printfn "Form submitted")
        
        prop.children [
            Html.input [
                prop.onChange (fun (e: Event) ->
                    let input = e.target :?> Browser.Types.HTMLInputElement
                    printfn "Value: %s" input.value)
                
                prop.onInput (fun e ->
                    printfn "Input event")
                
                prop.onFocus (fun _ -> printfn "Focused")
                
                prop.onBlur (fun _ -> printfn "Blurred")
            ]
            
            Html.select [
                prop.onChange (fun e ->
                    let select = e.target :?> Browser.Types.HTMLSelectElement
                    printfn "Selected: %s" select.value)
                prop.children [
                    Html.option [prop.value "a"; prop.text "A"]
                    Html.option [prop.value "b"; prop.text "B"]
                ]
            ]
        ]
    ]

// ===== Clipboard Events =====

let clipboardEvents =
    Html.div [
        prop.onCopy (fun e -> 
            e.preventDefault()
            printfn "Copy prevented")
        
        prop.onPaste (fun e ->
            e.preventDefault()
            let data = e.clipboardData?getData("text")
            printfn "Pasted: %A" data)
        
        prop.onCut (fun _ -> printfn "Cut")
    ]

// ===== Drag Events =====

let dragEvents =
    Html.div [
        prop.draggable true
        
        prop.onDragStart (fun e ->
            e.dataTransfer?setData("text/plain", "dragged data"))
        
        prop.onDragEnd (fun _ -> printfn "Drag ended")
        
        prop.onDragOver (fun e -> e.preventDefault())
        
        prop.onDrop (fun e ->
            e.preventDefault()
            let data = e.dataTransfer?getData("text/plain")
            printfn "Dropped: %A" data)
    ]

// ===== Scroll Events =====

let scrollEvents =
    Html.div [
        prop.style [style.height 400; style.overflow.auto]
        prop.onScroll (fun e ->
            let el = e.target :?> Browser.Types.HTMLElement
            printfn "Scroll: %d" el.scrollTop)
        prop.children [
            for i in 1..50 do
                Html.p (sprintf "Line %d" i)
        ]
    ]
```

---

## 6. State with React Hooks

### useState

```fsharp
open Feliz

// ===== Basic useState =====

let counter = React.functionComponent(fun () ->
    let count, setCount = React.useState(0)
    
    Html.div [
        Html.p (sprintf "Count: %d" count)
        Html.button [
            prop.text "+"
            prop.onClick (fun _ -> setCount (count + 1))
        ]
        Html.button [
            prop.text "-"
            prop.onClick (fun _ -> setCount (count - 1))
        ]
    ]
)

// ===== useState กับ Record =====

type FormState = {
    Name: string
    Email: string
    Message: string
    IsSubmitting: bool
}

let contactForm = React.functionComponent(fun () ->
    let form, setForm = React.useState({
        Name = ""
        Email = ""
        Message = ""
        IsSubmitting = false
    })
    
    let handleSubmit (e: Browser.Types.Event) =
        e.preventDefault()
        setForm { form with IsSubmitting = true }
        // ส่งข้อมูล...
    
    Html.form [
        prop.onSubmit handleSubmit
        prop.children [
            Html.input [
                prop.type' "text"
                prop.placeholder "ชื่อ"
                prop.value form.Name
                prop.onChange (fun e -> setForm { form with Name = e.target.value })
            ]
            Html.input [
                prop.type' "email"
                prop.placeholder "Email"
                prop.value form.Email
                prop.onChange (fun e -> setForm { form with Email = e.target.value })
            ]
            Html.textarea [
                prop.placeholder "ข้อความ"
                prop.value form.Message
                prop.onChange (fun e -> setForm { form with Message = e.target.value })
            ]
            Html.button [
                prop.type' "submit"
                prop.text (if form.IsSubmitting then "กำลังส่ง..." else "ส่ง")
                prop.disabled form.IsSubmitting
            ]
        ]
    ]
)

// ===== useState กับ List =====

let todoList = React.functionComponent(fun () ->
    let todos, setTodos = React.useState<string list>([])
    let input, setInput = React.useState("")
    
    let addTodo () =
        if input.Trim() <> "" then
            setTodos (todos @ [input.Trim()])
            setInput ""
    
    let removeTodo index =
        setTodos (todos |> List.mapi (fun i t -> (i, t)) |> List.filter (fun (i, _) -> i <> index) |> List.map snd)
    
    Html.div [
        Html.div [
            prop.style [style.display.flex; style.gap 8]
            prop.children [
                Html.input [
                    prop.value input
                    prop.onChange (fun e -> setInput e.target.value)
                    prop.onKeyDown (fun e -> if e.key = "Enter" then addTodo())
                    prop.placeholder "เพิ่ม todo"
                ]
                Html.button [
                    prop.text "เพิ่ม"
                    prop.onClick (fun _ -> addTodo())
                ]
            ]
        ]
        Html.ul [
            for i, todo in todos |> List.mapi (fun i t -> (i, t)) do
                Html.li [
                    prop.key i
                    prop.style [style.display.flex; style.justifyContent.spaceBetween]
                    prop.children [
                        Html.span todo
                        Html.button [
                            prop.text "×"
                            prop.onClick (fun _ -> removeTodo i)
                        ]
                    ]
                ]
        ]
    ]
)
```

---

## 7. useEffect

### Side Effects กับ useEffect

```fsharp
open Feliz

// ===== Run Once (empty deps) =====

let fetchOnMount = React.functionComponent(fun () ->
    let data, setData = React.useState<string option>(None)
    let error, setError = React.useState<string option>(None)
    
    // รัน once เมื่อ component mount
    React.useEffect(fun () ->
        async {
            try
                let! result = fetchData()
                setData (Some result)
            with ex ->
                setError (Some ex.Message)
        } |> Async.StartImmediate
        
        React.createDisposable(fun () -> ())
    , [||])
    
    match error with
    | Some err -> Html.div [prop.style [style.color "red"]; prop.text err]
    | None ->
        match data with
        | None -> Html.div "Loading..."
        | Some d -> Html.div d
)

// ===== Run on Dependency Change =====

let searchResults = React.functionComponent(fun (props: {| query: string |}) ->
    let results, setResults = React.useState<string list>([])
    let loading, setLoading = React.useState(false)
    
    // รันใหม่ทุกครั้งที่ query เปลี่ยน
    React.useEffect(fun () ->
        if props.query.Length > 2 then
            setLoading true
            async {
                let! found = search props.query
                setResults found
                setLoading false
            } |> Async.StartImmediate
        else
            setResults []
        
        React.createDisposable(fun () -> ())
    , [| props.query |])
    
    Html.div [
        if loading then Html.div "Searching..."
        else
            Html.ul [
                for r in results do Html.li r
            ]
    ]
)

// ===== Cleanup with Disposable =====

let timer = React.functionComponent(fun () ->
    let seconds, setSeconds = React.useState(0)
    let running, setRunning = React.useState(false)
    
    React.useEffect(fun () ->
        if running then
            let timerId = 
                Browser.Dom.window.setInterval(
                    (fun _ -> setSeconds (fun s -> s + 1)),
                    1000
                )
            
            // Cleanup: ยกเลิก interval เมื่อ component unmount หรือ deps เปลี่ยน
            React.createDisposable(fun () ->
                Browser.Dom.window.clearInterval(timerId))
        else
            React.createDisposable(fun () -> ())
    , [| running |])
    
    Html.div [
        Html.p (sprintf "Time: %d seconds" seconds)
        Html.button [
            prop.text (if running then "หยุด" else "เริ่ม")
            prop.onClick (fun _ -> setRunning (not running))
        ]
        Html.button [
            prop.text "Reset"
            prop.onClick (fun _ -> setSeconds 0; setRunning false)
        ]
    ]
)

// ===== useLayoutEffect =====

let measuredComponent = React.functionComponent(fun () ->
    let divRef = React.useRef<Browser.Types.HTMLDivElement option>(None)
    let dimensions, setDimensions = React.useState({| width = 0; height = 0 |})
    
    React.useLayoutEffect(fun () ->
        match divRef.current with
        | Some el ->
            setDimensions {| 
                width = int el.clientWidth
                height = int el.clientHeight 
            |}
        | None -> ()
        
        React.createDisposable(fun () -> ())
    , [||])
    
    Html.div [
        prop.ref divRef
        prop.text (sprintf "Size: %dx%d" dimensions.width dimensions.height)
    ]
)
```

---

## 8. Custom Hooks

### การสร้าง Custom Hooks

```fsharp
open Feliz

// ===== useLocalStorage =====

let useLocalStorage<'T> (key: string) (defaultValue: 'T) =
    let getStoredValue () =
        try
            let item = Browser.WebStorage.localStorage.getItem(key)
            if isNull item then defaultValue
            else Fable.Core.JS.JSON.parse(item) :?> 'T
        with _ -> defaultValue
    
    let value, setValue = React.useState(getStoredValue)
    
    let setStoredValue (newValue: 'T) =
        setValue newValue
        try
            Browser.WebStorage.localStorage.setItem(
                key, 
                Fable.Core.JS.JSON.stringify(newValue)
            )
        with _ -> ()
    
    value, setStoredValue

// ใช้งาน
let settingsComponent = React.functionComponent(fun () ->
    let theme, setTheme = useLocalStorage "theme" "light"
    let language, setLanguage = useLocalStorage "language" "th"
    
    Html.div [
        Html.p (sprintf "Theme: %s" theme)
        Html.button [
            prop.text "Toggle Theme"
            prop.onClick (fun _ -> 
                setTheme (if theme = "light" then "dark" else "light"))
        ]
    ]
)

// ===== useFetch =====

type FetchState<'T> =
    | Loading
    | Success of 'T
    | Failure of string

let useFetch<'T> (url: string) =
    let state, setState = React.useState(Loading)
    
    React.useEffect(fun () ->
        async {
            try
                let! response = Fetch.fetch url [] |> Async.AwaitPromise
                if response.Ok then
                    let! text = response.text() |> Async.AwaitPromise
                    // แปลง JSON เป็น type ที่ต้องการ
                    let data = Fable.Core.JS.JSON.parse(text) :?> 'T
                    setState (Success data)
                else
                    setState (Failure (sprintf "HTTP %d" response.Status))
            with ex ->
                setState (Failure ex.Message)
        } |> Async.StartImmediate
        
        React.createDisposable(fun () -> ())
    , [| url |])
    
    state

// ===== useDebounce =====

let useDebounce<'T> (value: 'T) (delay: int) =
    let debouncedValue, setDebouncedValue = React.useState(value)
    
    React.useEffect(fun () ->
        let timer = 
            Browser.Dom.window.setTimeout(
                (fun () -> setDebouncedValue value),
                delay
            )
        
        React.createDisposable(fun () ->
            Browser.Dom.window.clearTimeout(timer))
    , [| value |])
    
    debouncedValue

// ใช้งาน
let searchInput = React.functionComponent(fun () ->
    let query, setQuery = React.useState("")
    let debouncedQuery = useDebounce query 300
    
    React.useEffect(fun () ->
        if debouncedQuery <> "" then
            printfn "Searching for: %s" debouncedQuery
        React.createDisposable(fun () -> ())
    , [| debouncedQuery |])
    
    Html.input [
        prop.value query
        prop.onChange (fun e -> setQuery e.target.value)
        prop.placeholder "ค้นหา..."
    ]
)

// ===== useClickOutside =====

let useClickOutside (callback: unit -> unit) =
    let ref = React.useRef<Browser.Types.HTMLElement option>(None)
    
    React.useEffect(fun () ->
        let handler (e: Browser.Types.MouseEvent) =
            match ref.current with
            | Some el when not (el.contains(e.target :?> Browser.Types.Node)) ->
                callback()
            | _ -> ()
        
        Browser.Dom.document.addEventListener("mousedown", handler)
        
        React.createDisposable(fun () ->
            Browser.Dom.document.removeEventListener("mousedown", handler))
    , [||])
    
    ref

// ===== useMediaQuery =====

let useMediaQuery (query: string) =
    let matches, setMatches = 
        React.useState(
            Browser.Dom.window.matchMedia(query).matches
        )
    
    React.useEffect(fun () ->
        let mql = Browser.Dom.window.matchMedia(query)
        let handler _ = setMatches mql.matches
        mql.addEventListener("change", handler)
        
        React.createDisposable(fun () ->
            mql.removeEventListener("change", handler))
    , [| query |])
    
    matches

// ===== usePrevious =====

let usePrevious<'T> (value: 'T) =
    let ref = React.useRef<'T option>(None)
    
    React.useEffect(fun () ->
        ref.current <- Some value
        React.createDisposable(fun () -> ())
    , [| value |])
    
    ref.current
```

---

## 9. Component Composition

### Composing Components

```fsharp
open Feliz

// ===== Higher Order Components =====

// withAuth HOC
let withAuth (requiredRole: string) (Component: unit -> ReactElement) =
    React.functionComponent(fun () ->
        let isAuthenticated = true  // check auth
        let userRole = "admin"      // get from context/store
        
        if not isAuthenticated then
            Html.div "กรุณาเข้าสู่ระบบ"
        elif requiredRole <> "" && userRole <> requiredRole then
            Html.div "ไม่มีสิทธิ์เข้าถึง"
        else
            Component()
    )

// withLoading HOC
let withLoading (isLoading: bool) (Component: unit -> ReactElement) =
    React.functionComponent(fun () ->
        if isLoading then
            Html.div [
                prop.style [
                    style.display.flex
                    style.justifyContent.center
                    style.alignItems.center
                    style.padding 48
                ]
                prop.children [
                    Html.div [
                        prop.className "spinner"
                        prop.style [
                            style.width 40
                            style.height 40
                            style.borderRadius (length.percent 50)
                            style.border "4px solid #f0f0f0"
                            style.borderTop "4px solid #007bff"
                            style.animation "spin 1s linear infinite"
                        ]
                    ]
                ]
            ]
        else
            Component()
    )

// ===== Render Props =====

type DataProviderProps<'T> = {
    data: 'T
    render: 'T -> ReactElement
}

let dataProvider<'T> = React.functionComponent(fun (props: DataProviderProps<'T>) ->
    props.render props.data
)

// ===== Slots Pattern =====

type CardProps = {
    header: ReactElement option
    body: ReactElement
    footer: ReactElement option
    variant: string
}

let card = React.functionComponent(fun (props: CardProps) ->
    Html.div [
        prop.className (sprintf "card card-%s" props.variant)
        prop.style [
            style.border "1px solid #ddd"
            style.borderRadius 8
            style.overflow.hidden
        ]
        prop.children [
            match props.header with
            | Some h ->
                Html.div [
                    prop.className "card-header"
                    prop.style [
                        style.padding 16
                        style.backgroundColor "#f8f9fa"
                        style.borderBottom "1px solid #ddd"
                    ]
                    prop.child h
                ]
            | None -> Html.none
            
            Html.div [
                prop.className "card-body"
                prop.style [style.padding 16]
                prop.child props.body
            ]
            
            match props.footer with
            | Some f ->
                Html.div [
                    prop.className "card-footer"
                    prop.style [
                        style.padding 16
                        style.backgroundColor "#f8f9fa"
                        style.borderTop "1px solid #ddd"
                    ]
                    prop.child f
                ]
            | None -> Html.none
        ]
    ]
)

// ===== Context Pattern =====

open Fable.Core
open Feliz

type ThemeContext = {
    primaryColor: string
    backgroundColor: string
    textColor: string
    isDark: bool
    toggle: unit -> unit
}

let defaultTheme = {
    primaryColor = "#007bff"
    backgroundColor = "#ffffff"
    textColor = "#333333"
    isDark = false
    toggle = fun () -> ()
}

let themeContext = React.createContext(defaultTheme)

let themeProvider = React.functionComponent(fun (props: {| children: ReactElement list |}) ->
    let isDark, setIsDark = React.useState(false)
    
    let theme = 
        if isDark then {
            primaryColor = "#4da6ff"
            backgroundColor = "#1a1a2e"
            textColor = "#e0e0e0"
            isDark = true
            toggle = fun () -> setIsDark false
        }
        else {
            primaryColor = "#007bff"
            backgroundColor = "#ffffff"
            textColor = "#333333"
            isDark = false
            toggle = fun () -> setIsDark true
        }
    
    React.contextProvider(themeContext, theme, props.children)
)

let themedButton = React.functionComponent(fun (props: {| text: string; onClick: unit -> unit |}) ->
    let theme = React.useContext(themeContext)
    
    Html.button [
        prop.style [
            style.backgroundColor theme.primaryColor
            style.color "white"
            style.border "none"
            style.borderRadius 8
            style.padding (10, 20)
            style.cursor "pointer"
        ]
        prop.text props.text
        prop.onClick (fun _ -> props.onClick())
    ]
)
```

---

## 10. Conditional Rendering

```fsharp
open Feliz

// ===== if/then/else =====

let conditionalExample = React.functionComponent(fun (props: {| isLoggedIn: bool |}) ->
    if props.isLoggedIn then
        Html.div "ยินดีต้อนรับ!"
    else
        Html.div "กรุณาเข้าสู่ระบบ"
)

// ===== Pattern matching =====

type UserStatus = Active | Inactive | Banned | Loading

let userBadge = React.functionComponent(fun (props: {| status: UserStatus |}) ->
    match props.status with
    | Active ->
        Html.span [
            prop.style [style.backgroundColor "#28a745"; style.color "white"; style.padding (2, 8); style.borderRadius 4]
            prop.text "Active"
        ]
    | Inactive ->
        Html.span [
            prop.style [style.backgroundColor "#6c757d"; style.color "white"; style.padding (2, 8); style.borderRadius 4]
            prop.text "Inactive"
        ]
    | Banned ->
        Html.span [
            prop.style [style.backgroundColor "#dc3545"; style.color "white"; style.padding (2, 8); style.borderRadius 4]
            prop.text "Banned"
        ]
    | Loading ->
        Html.span [
            prop.style [style.backgroundColor "#ffc107"; style.color "black"; style.padding (2, 8); style.borderRadius 4]
            prop.text "Loading..."
        ]
)

// ===== Html.none =====

let optionalSection = React.functionComponent(fun (props: {| showExtra: bool; extraContent: string |}) ->
    Html.div [
        Html.p "เนื้อหาหลัก"
        
        if props.showExtra then
            Html.p props.extraContent
        // else = Html.none (implicit)
        
        // Explicit Html.none
        Html.div [
            if true then
                Html.span "แสดง"
            else
                Html.none
        ]
    ]
)

// ===== Conditional class names =====

let buttonWithState = React.functionComponent(fun (props: {| active: bool; disabled: bool; loading: bool |}) ->
    Html.button [
        prop.className [
            "btn"
            if props.active then "btn-active"
            if props.disabled then "btn-disabled"
            if props.loading then "btn-loading"
        ]
        prop.disabled props.disabled
        prop.text (if props.loading then "Loading..." else "Click")
    ]
)
```

---

## 11. Lists and Keys

```fsharp
open Feliz

// ===== Basic list rendering =====

let basicList = React.functionComponent(fun () ->
    let items = ["Apple"; "Banana"; "Cherry"; "Date"]
    
    Html.ul [
        for item in items do
            Html.li [
                prop.key item  // key สำคัญมาก!
                prop.text item
            ]
    ]
)

// ===== List ที่ซับซ้อน =====

type Employee = {
    Id: int
    Name: string
    Department: string
    Salary: decimal
}

let employeeTable = React.functionComponent(fun (props: {| employees: Employee list |}) ->
    let sortField, setSortField = React.useState("name")
    let sortAsc, setSortAsc = React.useState(true)
    
    let sorted =
        let key =
            match sortField with
            | "name" -> (fun e -> e.Name :> obj)
            | "dept" -> (fun e -> e.Department :> obj)
            | "salary" -> (fun e -> e.Salary :> obj)
            | _ -> (fun e -> e.Name :> obj)
        
        if sortAsc then props.employees |> List.sortBy key
        else props.employees |> List.sortByDescending key
    
    let handleSort field =
        if sortField = field then setSortAsc (not sortAsc)
        else setSortField field; setSortAsc true
    
    let sortIcon field =
        if sortField = field then
            if sortAsc then " ▲" else " ▼"
        else ""
    
    Html.table [
        prop.style [style.width (length.percent 100); style.borderCollapse.collapse]
        prop.children [
            Html.thead [
                Html.tr [
                    for (field, label) in [("name", "ชื่อ"); ("dept", "แผนก"); ("salary", "เงินเดือน")] do
                        Html.th [
                            prop.style [
                                style.padding 12
                                style.borderBottom "2px solid #ddd"
                                style.cursor "pointer"
                                style.textAlign.left
                            ]
                            prop.onClick (fun _ -> handleSort field)
                            prop.text (sprintf "%s%s" label (sortIcon field))
                        ]
                ]
            ]
            Html.tbody [
                for employee in sorted do
                    Html.tr [
                        prop.key employee.Id
                        prop.style [style.borderBottom "1px solid #eee"]
                        prop.children [
                            Html.td [prop.style [style.padding 12]; prop.text employee.Name]
                            Html.td [prop.style [style.padding 12]; prop.text employee.Department]
                            Html.td [prop.style [style.padding 12]; prop.text (sprintf "฿%s" (employee.Salary.ToString("N0")))]
                        ]
                    ]
            ]
        ]
    ]
)
```

---

## 12. Forms

### Complex Form Handling

```fsharp
open Feliz

type ValidationResult = 
    | Valid
    | Invalid of string

let validateEmail (email: string) =
    if email.Contains("@") && email.Contains(".") then Valid
    else Invalid "Email ไม่ถูกต้อง"

let validateRequired (fieldName: string) (value: string) =
    if value.Trim() = "" then Invalid (sprintf "%s ต้องไม่ว่าง" fieldName)
    else Valid

let validateMinLength (min: int) (value: string) =
    if value.Length < min then Invalid (sprintf "ต้องมีอย่างน้อย %d ตัวอักษร" min)
    else Valid

type RegisterForm = {
    FirstName: string
    LastName: string
    Email: string
    Password: string
    ConfirmPassword: string
    Age: int
    Gender: string
    AcceptTerms: bool
}

type FormErrors = {
    FirstName: string option
    LastName: string option
    Email: string option
    Password: string option
    ConfirmPassword: string option
    Age: string option
}

let emptyErrors = {
    FirstName = None
    LastName = None
    Email = None
    Password = None
    ConfirmPassword = None
    Age = None
}

let validateForm (form: RegisterForm) =
    let errors = {
        FirstName = 
            match validateRequired "ชื่อ" form.FirstName with
            | Invalid err -> Some err
            | Valid -> None
        LastName =
            match validateRequired "นามสกุล" form.LastName with
            | Invalid err -> Some err
            | Valid -> None
        Email =
            match validateEmail form.Email with
            | Invalid err -> Some err
            | Valid -> None
        Password =
            match validateMinLength 8 form.Password with
            | Invalid err -> Some err
            | Valid -> None
        ConfirmPassword =
            if form.Password <> form.ConfirmPassword then Some "รหัสผ่านไม่ตรงกัน"
            else None
        Age =
            if form.Age < 18 then Some "ต้องมีอายุ 18 ปีขึ้นไป"
            elif form.Age > 120 then Some "อายุไม่ถูกต้อง"
            else None
    }
    errors

let hasErrors errors =
    [errors.FirstName; errors.LastName; errors.Email; errors.Password; errors.ConfirmPassword; errors.Age]
    |> List.exists Option.isSome

let registrationForm = React.functionComponent(fun () ->
    let form, setForm = React.useState({
        FirstName = ""
        LastName = ""
        Email = ""
        Password = ""
        ConfirmPassword = ""
        Age = 18
        Gender = "other"
        AcceptTerms = false
    })
    
    let errors, setErrors = React.useState(emptyErrors)
    let touched, setTouched = React.useState(Set.empty<string>)
    let submitted, setSubmitted = React.useState(false)
    let submitSuccess, setSubmitSuccess = React.useState(false)
    
    let touch (field: string) =
        setTouched (touched.Add(field))
    
    let showError field error =
        if touched.Contains(field) || submitted then
            match error with
            | Some err -> Html.span [prop.style [style.color "red"; style.fontSize 12]; prop.text err]
            | None -> Html.none
        else Html.none
    
    let handleSubmit (e: Browser.Types.Event) =
        e.preventDefault()
        setSubmitted true
        let errs = validateForm form
        setErrors errs
        if not (hasErrors errs) && form.AcceptTerms then
            // ส่งข้อมูล...
            setSubmitSuccess true
    
    if submitSuccess then
        Html.div [
            prop.style [style.textAlign.center; style.padding 48]
            prop.children [
                Html.h2 "✓ สมัครสมาชิกสำเร็จ!"
                Html.p (sprintf "ยินดีต้อนรับ, %s %s!" form.FirstName form.LastName)
            ]
        ]
    else
        Html.form [
            prop.onSubmit handleSubmit
            prop.style [
                style.maxWidth 480
                style.margin (0, length.auto)
                style.padding 32
                style.backgroundColor "white"
                style.borderRadius 12
                style.boxShadow "0 4px 16px rgba(0,0,0,0.1)"
            ]
            prop.children [
                Html.h2 [
                    prop.style [style.marginBottom 24; style.textAlign.center]
                    prop.text "สมัครสมาชิก"
                ]
                
                // First Name
                Html.div [
                    prop.style [style.marginBottom 16]
                    prop.children [
                        Html.label [prop.for' "firstName"; prop.text "ชื่อ *"]
                        Html.input [
                            prop.id "firstName"
                            prop.type' "text"
                            prop.value form.FirstName
                            prop.onChange (fun e -> 
                                setForm { form with FirstName = e.target.value }
                                setErrors { errors with FirstName = None })
                            prop.onBlur (fun _ -> 
                                touch "firstName"
                                let errs = validateForm form
                                setErrors { errors with FirstName = errs.FirstName })
                            prop.style [
                                style.width (length.percent 100)
                                style.padding 10
                                style.borderRadius 6
                                style.border (if errors.FirstName.IsSome && (touched.Contains("firstName") || submitted) then "1px solid red" else "1px solid #ddd")
                                style.boxSizing.borderBox
                            ]
                        ]
                        showError "firstName" errors.FirstName
                    ]
                ]
                
                // Email
                Html.div [
                    prop.style [style.marginBottom 16]
                    prop.children [
                        Html.label [prop.for' "email"; prop.text "Email *"]
                        Html.input [
                            prop.id "email"
                            prop.type' "email"
                            prop.value form.Email
                            prop.onChange (fun e -> setForm { form with Email = e.target.value })
                            prop.onBlur (fun _ -> touch "email")
                            prop.style [
                                style.width (length.percent 100)
                                style.padding 10
                                style.borderRadius 6
                                style.border "1px solid #ddd"
                                style.boxSizing.borderBox
                            ]
                        ]
                        showError "email" errors.Email
                    ]
                ]
                
                // Accept Terms
                Html.div [
                    prop.style [style.marginBottom 24]
                    prop.children [
                        Html.label [
                            prop.style [style.display.flex; style.alignItems.center; style.gap 8]
                            prop.children [
                                Html.input [
                                    prop.type' "checkbox"
                                    prop.checked' form.AcceptTerms
                                    prop.onChange (fun _ -> setForm { form with AcceptTerms = not form.AcceptTerms })
                                ]
                                Html.text "ยอมรับ "
                                Html.a [prop.href "/terms"; prop.text "ข้อกำหนดการใช้งาน"]
                            ]
                        ]
                    ]
                ]
                
                // Submit
                Html.button [
                    prop.type' "submit"
                    prop.text "สมัครสมาชิก"
                    prop.disabled (not form.AcceptTerms)
                    prop.style [
                        style.width (length.percent 100)
                        style.backgroundColor (if form.AcceptTerms then "#007bff" else "#6c757d")
                        style.color "white"
                        style.border "none"
                        style.borderRadius 8
                        style.padding 14
                        style.cursor (if form.AcceptTerms then "pointer" else "not-allowed")
                        style.fontSize 16
                    ]
                ]
            ]
        ]
)
```

---

## 13. Feliz.Router

```fsharp
open Feliz
open Feliz.Router

// ===== Basic Routing =====

type Page =
    | Home
    | About
    | Contact
    | Blog
    | BlogPost of int
    | UserProfile of string
    | NotFound

let parsePage = function
    | [] -> Home
    | ["about"] -> About
    | ["contact"] -> Contact
    | ["blog"] -> Blog
    | ["blog"; Route.Int id] -> BlogPost id
    | ["user"; username] -> UserProfile username
    | _ -> NotFound

let pageToUrl = function
    | Home -> "/"
    | About -> "/about"
    | Contact -> "/contact"
    | Blog -> "/blog"
    | BlogPost id -> sprintf "/blog/%d" id
    | UserProfile user -> sprintf "/user/%s" user
    | NotFound -> "/404"

let navLink (label: string) (page: Page) (currentPage: Page) dispatch =
    Html.a [
        prop.href (pageToUrl page)
        prop.style [
            style.color (if currentPage = page then "#007bff" else "#333")
            style.fontWeight (if currentPage = page then "bold" else "normal")
            style.textDecoration.none
            style.padding (8, 16)
        ]
        prop.onClick (fun e ->
            e.preventDefault()
            dispatch (NavigateTo page))
        prop.text label
    ]

let routedApp = React.functionComponent(fun () ->
    let currentPage, setCurrentPage = 
        React.useState(parsePage (Router.currentPath()))
    
    let navigate page =
        Router.navigate (pageToUrl page)
        setCurrentPage page
    
    React.router [
        router.onUrlChanged (fun segments -> 
            setCurrentPage (parsePage segments))
        
        router.children [
            Html.div [
                prop.style [style.fontFamily "Arial, sans-serif"]
                prop.children [
                    // Navigation
                    Html.nav [
                        prop.style [
                            style.display.flex
                            style.gap 8
                            style.padding (16, 24)
                            style.backgroundColor "#f8f9fa"
                            style.borderBottom "1px solid #ddd"
                        ]
                        prop.children [
                            navLink "Home" Home currentPage navigate
                            navLink "About" About currentPage navigate
                            navLink "Blog" Blog currentPage navigate
                            navLink "Contact" Contact currentPage navigate
                        ]
                    ]
                    
                    // Content
                    Html.main [
                        prop.style [style.padding 24]
                        prop.children [
                            match currentPage with
                            | Home ->
                                Html.div [
                                    Html.h1 "หน้าแรก"
                                    Html.p "ยินดีต้อนรับสู่ Feliz Router App"
                                ]
                            | About ->
                                Html.div [
                                    Html.h1 "เกี่ยวกับเรา"
                                    Html.p "นี่คือหน้า About"
                                ]
                            | Blog ->
                                Html.div [
                                    Html.h1 "บล็อก"
                                    Html.ul [
                                        for i in 1..5 do
                                            Html.li [
                                                Html.a [
                                                    prop.href (sprintf "/blog/%d" i)
                                                    prop.onClick (fun e ->
                                                        e.preventDefault()
                                                        navigate (BlogPost i))
                                                    prop.text (sprintf "บทความที่ %d" i)
                                                ]
                                            ]
                                    ]
                                ]
                            | BlogPost id ->
                                Html.div [
                                    Html.h1 (sprintf "บทความที่ %d" id)
                                    Html.p "เนื้อหาบทความ..."
                                    Html.button [
                                        prop.text "กลับ"
                                        prop.onClick (fun _ -> navigate Blog)
                                    ]
                                ]
                            | NotFound ->
                                Html.div [
                                    Html.h1 "404 - ไม่พบหน้านี้"
                                    Html.a [
                                        prop.href "/"
                                        prop.onClick (fun e -> e.preventDefault(); navigate Home)
                                        prop.text "กลับหน้าแรก"
                                    ]
                                ]
                            | _ ->
                                Html.div "Page under construction"
                        ]
                    ]
                ]
            ]
        ]
    ]
)

// ===== โปรแกรมหลัก =====

open Browser.Dom

let root = ReactDOM.createRoot(document.getElementById("app"))
root.render(routedApp())
```

---

## สรุป (Summary)

Feliz ทำให้การเขียน React กับ F# เป็นเรื่องสนุกและมี type safety โดยมีคุณสมบัติหลักๆ ดังนี้:

1. **HTML as Functions**: เขียน HTML แบบ type-safe
2. **Props System**: Props ที่ type-checked
3. **Hooks**: useState, useEffect, useRef, useContext
4. **Custom Hooks**: สร้าง reusable logic
5. **Component Composition**: HOC, Render Props, Context
6. **Feliz.Router**: Type-safe client-side routing

---

*ไปต่อที่ Part 104: Bolero - Blazor for F#*
