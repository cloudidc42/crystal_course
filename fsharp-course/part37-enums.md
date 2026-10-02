# Part 37 - Enum และ Flags (Enums and Flags)

## บทนำ (Introduction)

Enum ใน F# คือ type ที่มีค่าจำกัดซึ่งเป็น integral type (int, byte, int64 เป็นต้น) Enum แตกต่างจาก Discriminated Union ตรงที่ค่าแต่ละตัวเป็นเพียงตัวเลข ไม่มี associated data F# DU ทรงพลังกว่าแต่ Enum ใช้ง่ายกว่าในบางสถานการณ์

---

## 1. Enum Type Definition

```fsharp
// การนิยาม Enum พื้นฐาน
type DayOfWeek =
    | Monday = 0
    | Tuesday = 1
    | Wednesday = 2
    | Thursday = 3
    | Friday = 4
    | Saturday = 5
    | Sunday = 6

type Month =
    | January = 1
    | February = 2
    | March = 3
    | April = 4
    | May = 5
    | June = 6
    | July = 7
    | August = 8
    | September = 9
    | October = 10
    | November = 11
    | December = 12

// ใช้ enum
let today = DayOfWeek.Wednesday
printfn "Today: %A" today

let birthMonth = Month.July
printfn "Birth month: %A" birthMonth

// Enum values
printfn "Monday = %d" (int DayOfWeek.Monday)
printfn "Friday = %d" (int DayOfWeek.Friday)
printfn "July = %d" (int Month.July)
```

---

## 2. Enum Values

```fsharp
// Enum ที่มีค่าต่างๆ

// HTTP Status codes
type HttpStatus =
    | Ok = 200
    | Created = 201
    | Accepted = 202
    | NoContent = 204
    | BadRequest = 400
    | Unauthorized = 401
    | Forbidden = 403
    | NotFound = 404
    | InternalServerError = 500
    | BadGateway = 502
    | ServiceUnavailable = 503

// Priority levels
type Priority =
    | Low = 1
    | Medium = 5
    | High = 10
    | Critical = 100

// Color enum
type ConsoleColor2 =
    | Black = 0
    | DarkBlue = 1
    | DarkGreen = 2
    | DarkCyan = 3
    | DarkRed = 4
    | DarkMagenta = 5
    | DarkYellow = 6
    | Gray = 7
    | DarkGray = 8
    | Blue = 9
    | Green = 10
    | Cyan = 11
    | Red = 12
    | Magenta = 13
    | Yellow = 14
    | White = 15

// การใช้ enum values
let status = HttpStatus.Ok
printfn "Status: %A (%d)" status (int status)

let priority = Priority.High
printfn "Priority: %A (%d)" priority (int priority)

// ดู enum ที่มีค่าสูงกว่า
let criticalLevel = int Priority.Critical > int Priority.High
printfn "Critical > High: %b" criticalLevel

// Get all enum values
let allStatuses = System.Enum.GetValues<HttpStatus>()
printfn "\nAll HTTP Statuses:"
for status in allStatuses do
    printfn "  %A = %d" status (int status)
```

---

## 3. Enum with Explicit Values

```fsharp
// Enum ที่กำหนดค่าเองทั้งหมดหรือบางส่วน

// Network protocol types (IANA-assigned)
type IPProtocol =
    | ICMP = 1
    | TCP = 6
    | UDP = 17
    | IPv6 = 41
    | GRE = 47
    | ESP = 50
    | AH = 51
    | ICMPv6 = 58
    | OSPF = 89
    | SCTP = 132

// HTTP Methods
type HttpMethod =
    | GET = 0
    | POST = 1
    | PUT = 2
    | DELETE = 3
    | PATCH = 4
    | HEAD = 5
    | OPTIONS = 6
    | TRACE = 7

// Log levels ที่ compatible กับ syslog severity
type LogLevel =
    | Emergency = 0
    | Alert = 1
    | Critical = 2
    | Error = 3
    | Warning = 4
    | Notice = 5
    | Info = 6
    | Debug = 7

// ใช้ enum
let protocol = IPProtocol.TCP
let method = HttpMethod.GET
let level = LogLevel.Warning

printfn "Protocol: %A (%d)" protocol (int protocol)
printfn "Method: %A (%d)" method (int method)
printfn "Log level: %A (%d)" level (int level)

// Enum ที่ไม่ได้ต่อเนื่อง (สำหรับ compatibility)
type ErrorCode =
    | Success = 0
    | NotFound = 404
    | ServerError = 500
    | Timeout = 408
    | Conflict = 409
    | GatewayTimeout = 504

let describeError (code: ErrorCode) =
    match code with
    | ErrorCode.Success -> "Operation completed successfully"
    | ErrorCode.NotFound -> "Resource not found"
    | ErrorCode.ServerError -> "Internal server error"
    | ErrorCode.Timeout -> "Request timeout"
    | ErrorCode.Conflict -> "Resource conflict"
    | ErrorCode.GatewayTimeout -> "Gateway timeout"
    | _ -> sprintf "Unknown error: %d" (int code)

printfn "%s" (describeError ErrorCode.NotFound)
printfn "%s" (describeError ErrorCode.Success)
```

---

## 4. Parsing Enums

```fsharp
// การ parse enum จาก string และ integer

// Parse จาก string
let parseHttpStatus (s: string) =
    match System.Enum.TryParse<HttpStatus>(s) with
    | true, status -> Some status
    | _ -> None

// Parse จาก int
let statusFromInt (code: int) =
    if System.Enum.IsDefined(typeof<HttpStatus>, code) then
        Some (enum<HttpStatus> code)
    else
        None

// Parse case-insensitive
let parseStatusCI (s: string) =
    match System.Enum.TryParse<HttpStatus>(s, ignoreCase = true) with
    | true, status -> Some status
    | _ -> None

// ทดสอบ parsing
let testParsing () =
    // From name
    printfn "From 'Ok': %A" (parseHttpStatus "Ok")
    printfn "From 'ok': %A" (parseStatusCI "ok")
    printfn "From 'invalid': %A" (parseHttpStatus "invalid")
    
    // From int
    printfn "From 200: %A" (statusFromInt 200)
    printfn "From 404: %A" (statusFromInt 404)
    printfn "From 999: %A" (statusFromInt 999)

testParsing()

// Safe parse helper
let tryParseEnum<'T when 'T : struct and 'T :> System.Enum> (s: string) : 'T option =
    match System.Enum.TryParse<'T>(s, ignoreCase = true) with
    | true, v -> Some v
    | _ -> None

let day = tryParseEnum<DayOfWeek> "friday"
printfn "\nDay: %A" day

let priority2 = tryParseEnum<Priority> "HIGH"
printfn "Priority: %A" priority2

// Batch parsing
let statusStrings = ["Ok"; "NotFound"; "InternalServerError"; "Invalid"; "404"]
let parsedStatuses = 
    statusStrings 
    |> List.choose (fun s ->
        match parseHttpStatus s with
        | Some status -> Some (s, status)
        | None ->
            match System.Int32.TryParse(s) with
            | true, n -> statusFromInt n |> Option.map (fun status -> (s, status))
            | _ -> None
    )

printfn "\nParsed statuses:"
for (input, status) in parsedStatuses do
    printfn "  '%s' -> %A (%d)" input status (int status)
```

---

## 5. Enum Arithmetic

```fsharp
// Arithmetic ด้วย enum values

// Enum + int = เปลี่ยนไปยัง enum อื่น
let nextDay (day: DayOfWeek) =
    let nextValue = (int day + 1) % 7
    enum<DayOfWeek> nextValue

let previousDay (day: DayOfWeek) =
    let prevValue = (int day + 6) % 7  // +6 แทน -1 เพื่อหลีกเลี่ยง negative modulo
    enum<DayOfWeek> prevValue

let addDays (day: DayOfWeek) (days: int) =
    enum<DayOfWeek> ((int day + days) % 7)

printfn "Tomorrow after Wednesday: %A" (nextDay DayOfWeek.Wednesday)
printfn "Yesterday of Monday: %A" (previousDay DayOfWeek.Monday)
printfn "Wednesday + 3 days: %A" (addDays DayOfWeek.Wednesday 3)

// Days until next occurrence
let daysUntil (from: DayOfWeek) (target: DayOfWeek) =
    let diff = (int target - int from + 7) % 7
    if diff = 0 then 7  // next week
    else diff

printfn "\nDays until Saturday from Wednesday: %d" (daysUntil DayOfWeek.Wednesday DayOfWeek.Saturday)
printfn "Days until Monday from Friday: %d" (daysUntil DayOfWeek.Friday DayOfWeek.Monday)

// Enum comparison
let isWeekend (day: DayOfWeek) =
    day = DayOfWeek.Saturday || day = DayOfWeek.Sunday

let isWeekday (day: DayOfWeek) = not (isWeekend day)

let allDays = System.Enum.GetValues<DayOfWeek>() |> Array.toList
let weekdays = allDays |> List.filter isWeekday
let weekendDays = allDays |> List.filter isWeekend

printfn "\nWeekdays: %A" (weekdays |> List.map string)
printfn "Weekend: %A" (weekendDays |> List.map string)

// Month arithmetic
let nextMonth (month: Month) =
    let next = (int month % 12) + 1
    enum<Month> next

let daysInMonth (month: Month) (year: int) =
    System.DateTime.DaysInMonth(year, int month)

printfn "\nDays in February 2024: %d" (daysInMonth Month.February 2024)
printfn "Days in February 2023: %d" (daysInMonth Month.February 2023)
printfn "Next month after December: %A" (nextMonth Month.December)
```

---

## 6. [<Flags>] Attribute for Bitfield Enums

```fsharp
// [<Flags>] สำหรับ bitwise combination

[<System.Flags>]
type FilePermission =
    | None = 0
    | Read = 1
    | Write = 2
    | Execute = 4
    | ReadWrite = 3   // Read ||| Write = 1 ||| 2 = 3
    | ReadExecute = 5 // Read ||| Execute = 1 ||| 4 = 5
    | All = 7         // Read ||| Write ||| Execute

[<System.Flags>]
type NotificationChannel =
    | None = 0
    | Email = 1
    | SMS = 2
    | Push = 4
    | InApp = 8
    | All = 15  // Email ||| SMS ||| Push ||| InApp

[<System.Flags>]
type DayFlags =
    | None = 0
    | Monday = 1
    | Tuesday = 2
    | Wednesday = 4
    | Thursday = 8
    | Friday = 16
    | Saturday = 32
    | Sunday = 64
    | Weekday = 31  // Mon+Tue+Wed+Thu+Fri
    | Weekend = 96  // Sat+Sun
    | All = 127     // All days

// ใช้ Flags enum
let userPerms = FilePermission.ReadWrite  // = FilePermission.Read ||| FilePermission.Write
printfn "User permissions: %A (%d)" userPerms (int userPerms)
printfn "Has read: %b" (userPerms &&& FilePermission.Read = FilePermission.Read)
printfn "Has write: %b" (userPerms &&& FilePermission.Write = FilePermission.Write)
printfn "Has execute: %b" (userPerms &&& FilePermission.Execute = FilePermission.Execute)

// Check if flag is set
let hasFlag (flags: 'T when 'T : (static member op_BitwiseAnd: 'T * 'T -> 'T)) flag =
    ()  // complex with generics

let hasPermission (perms: FilePermission) (flag: FilePermission) =
    perms &&& flag = flag

let userPermissions = FilePermission.Read ||| FilePermission.Write
printfn "\nUser can read: %b" (hasPermission userPermissions FilePermission.Read)
printfn "User can write: %b" (hasPermission userPermissions FilePermission.Write)
printfn "User can execute: %b" (hasPermission userPermissions FilePermission.Execute)
```

---

## 7. Bitwise Operations

```fsharp
// Bitwise operations กับ Flags enum

[<System.Flags>]
type Permission =
    | None = 0b00000000  // 0
    | Read = 0b00000001  // 1
    | Write = 0b00000010 // 2
    | Execute = 0b00000100 // 4
    | Admin = 0b10000000 // 128

// Set flag
let addPermission (perms: Permission) (flag: Permission) =
    perms ||| flag

// Remove flag
let removePermission (perms: Permission) (flag: Permission) =
    perms &&& (~~~flag)

// Toggle flag
let togglePermission (perms: Permission) (flag: Permission) =
    perms ^^^ flag

// Check flag
let hasPermission2 (perms: Permission) (flag: Permission) =
    perms &&& flag = flag

// Grant/Revoke permissions
let mutable userPerms = Permission.None
printfn "Initial: %A (%d)" userPerms (int userPerms)

userPerms <- addPermission userPerms Permission.Read
printfn "After adding Read: %A (%d)" userPerms (int userPerms)

userPerms <- addPermission userPerms Permission.Write
printfn "After adding Write: %A (%d)" userPerms (int userPerms)

userPerms <- removePermission userPerms Permission.Write
printfn "After removing Write: %A (%d)" userPerms (int userPerms)

// All permission flags
let describePermissions (perms: Permission) =
    let flags = [
        Permission.Read, "Read"
        Permission.Write, "Write"
        Permission.Execute, "Execute"
        Permission.Admin, "Admin"
    ]
    flags
    |> List.filter (fun (flag, _) -> hasPermission2 perms flag)
    |> List.map snd
    |> String.concat ", "

let adminPerms = Permission.Read ||| Permission.Write ||| Permission.Execute ||| Permission.Admin
printfn "\nAdmin permissions: %s" (describePermissions adminPerms)
printfn "User permissions: %s" (describePermissions userPerms)

// Bitwise operations ด้วย int
let setBit (n: int) (bit: int) = n ||| (1 <<< bit)
let clearBit (n: int) (bit: int) = n &&& ~~~(1 <<< bit)
let toggleBit (n: int) (bit: int) = n ^^^ (1 <<< bit)
let hasBit (n: int) (bit: int) = n &&& (1 <<< bit) <> 0

let mutable flags2 = 0b00000000
printfn "\nBitwise operations:"
flags2 <- setBit flags2 0  // Set bit 0
flags2 <- setBit flags2 3  // Set bit 3
printfn "After setting bits 0 and 3: %08b (%d)" flags2 flags2
flags2 <- clearBit flags2 0  // Clear bit 0
printfn "After clearing bit 0: %08b (%d)" flags2 flags2
printfn "Has bit 3: %b" (hasBit flags2 3)
```

---

## 8. Comparing Enums

```fsharp
// Enum comparison

let comparePriorities (p1: Priority) (p2: Priority) =
    compare (int p1) (int p2)

let priorities = [Priority.High; Priority.Low; Priority.Critical; Priority.Medium]
let sortedPriorities = priorities |> List.sortWith comparePriorities
printfn "Sorted priorities: %A" sortedPriorities

// Group by comparison
let isHighOrCritical (p: Priority) = p >= Priority.High
let isLowPriority (p: Priority) = p = Priority.Low

let tasks = [
    "Fix bug", Priority.Critical
    "Write docs", Priority.Low
    "Code review", Priority.High
    "Refactor", Priority.Medium
    "Update README", Priority.Low
]

printfn "\nHigh priority tasks:"
tasks 
|> List.filter (fun (_, p) -> isHighOrCritical p)
|> List.iter (fun (task, p) -> printfn "  [%A] %s" p task)

printfn "\nLow priority tasks:"
tasks
|> List.filter (fun (_, p) -> isLowPriority p)
|> List.iter (fun (task, p) -> printfn "  [%A] %s" p task)

// Min/Max
let highestPriority = tasks |> List.maxBy (fun (_, p) -> int p) |> snd
let lowestPriority = tasks |> List.minBy (fun (_, p) -> int p) |> snd
printfn "\nHighest: %A, Lowest: %A" highestPriority lowestPriority

// Enum ordering
let statusOrder = [
    HttpStatus.Ok, 0
    HttpStatus.Created, 1
    HttpStatus.Accepted, 2
    HttpStatus.NoContent, 3
    HttpStatus.BadRequest, 100
    HttpStatus.Unauthorized, 101
    HttpStatus.Forbidden, 102
    HttpStatus.NotFound, 103
    HttpStatus.InternalServerError, 200
] |> Map.ofList

let compareStatuses s1 s2 =
    match statusOrder.TryFind s1, statusOrder.TryFind s2 with
    | Some o1, Some o2 -> compare o1 o2
    | _ -> compare (int s1) (int s2)
```

---

## 9. Enum in Pattern Matching

```fsharp
// Pattern matching กับ enum

let describeDay (day: DayOfWeek) =
    match day with
    | DayOfWeek.Monday -> "เริ่มสัปดาห์ใหม่!"
    | DayOfWeek.Tuesday -> "เริ่มเข้าสู่กลางสัปดาห์"
    | DayOfWeek.Wednesday -> "ครึ่งสัปดาห์แล้ว"
    | DayOfWeek.Thursday -> "ใกล้ถึงวันหยุดแล้ว"
    | DayOfWeek.Friday -> "วันสุดท้ายของสัปดาห์"
    | DayOfWeek.Saturday | DayOfWeek.Sunday -> "วันหยุด!"
    | _ -> "วันไม่ทราบ"

let allDays = System.Enum.GetValues<DayOfWeek>() |> Array.toList
for day in allDays do
    printfn "%A: %s" day (describeDay day)

// Pattern matching กับ HTTP status
let handleResponse (status: HttpStatus) (body: string) =
    match status with
    | HttpStatus.Ok | HttpStatus.Created | HttpStatus.Accepted ->
        printfn "Success (%A): %s" status body
    | HttpStatus.NoContent ->
        printfn "No content"
    | HttpStatus.BadRequest ->
        printfn "Bad request: %s" body
    | HttpStatus.Unauthorized | HttpStatus.Forbidden ->
        printfn "Access denied (%A)" status
    | HttpStatus.NotFound ->
        printfn "Resource not found"
    | s when int s >= 500 ->
        printfn "Server error (%A): %s" s body
    | _ ->
        printfn "Unknown status: %A" status

handleResponse HttpStatus.Ok "Data retrieved successfully"
handleResponse HttpStatus.NotFound ""
handleResponse HttpStatus.InternalServerError "Database connection failed"

// Nested pattern matching
let classifyDay (day: DayOfWeek) (hour: int) =
    match day, hour with
    | (DayOfWeek.Saturday | DayOfWeek.Sunday), _ -> "Weekend!"
    | _, h when h < 9 -> "Early morning"
    | _, h when h < 17 -> "Work hours"
    | _, h when h < 21 -> "Evening"
    | _ -> "Late night"

printfn "\nDay/time classification:"
printfn "Mon 8am: %s" (classifyDay DayOfWeek.Monday 8)
printfn "Wed 2pm: %s" (classifyDay DayOfWeek.Wednesday 14)
printfn "Sat 3pm: %s" (classifyDay DayOfWeek.Saturday 15)
```

---

## 10. Converting Enum to/from int

```fsharp
// การแปลงระหว่าง enum และ int

// Enum to int
let httpCode = int HttpStatus.NotFound  // 404
printfn "NotFound = %d" httpCode

// Int to enum
let statusFromCode (code: int) : HttpStatus =
    enum<HttpStatus> code

let status404 = statusFromCode 404
printfn "404 = %A" status404

// Safe conversion (check if valid)
let safeStatusFromCode (code: int) =
    if System.Enum.IsDefined(typeof<HttpStatus>, code) then
        Some (enum<HttpStatus> code)
    else
        None

printfn "%A" (safeStatusFromCode 200)  // Some Ok
printfn "%A" (safeStatusFromCode 999)  // None

// Enum to string and back
let statusToString (s: HttpStatus) = string s
let statusFromString (s: string) =
    System.Enum.TryParse<HttpStatus>(s) |> function
    | true, v -> Some v
    | _ -> None

printfn "'%s'" (statusToString HttpStatus.NotFound)  // "NotFound"
printfn "%A" (statusFromString "InternalServerError")  // Some InternalServerError

// All values and names
let enumValues<'T when 'T : struct and 'T :> System.Enum>() =
    System.Enum.GetValues<'T>() |> Array.toList

let enumNames<'T when 'T : struct and 'T :> System.Enum>() =
    System.Enum.GetNames(typeof<'T>) |> Array.toList

let statusValues = enumValues<HttpStatus>()
let statusNames = enumNames<HttpStatus>()

printfn "\nHTTP Status codes:"
List.zip statusNames statusValues
|> List.iter (fun (name, value) -> printfn "  %s = %d" name (int value))
```

---

## 11. F# DU vs Enum

```fsharp
// เปรียบเทียบ F# Discriminated Union กับ Enum

// Enum - simple, integral values
type DirectionEnum =
    | North = 0
    | South = 1
    | East = 2
    | West = 3

// F# DU - ทรงพลังกว่า
type Direction =
    | North
    | South
    | East
    | West
    | NE  // Northeast
    | NW  // Northwest
    | SE  // Southeast
    | SW  // Southwest

// DU สามารถมี data
type Command =
    | Move of direction: Direction * distance: float
    | Turn of degrees: float
    | Stop
    | SetSpeed of mph: float

// Enum ทำได้แค่นี้
type CommandEnum =
    | Move = 0
    | Turn = 1
    | Stop = 2
    | SetSpeed = 3
    // ไม่สามารถมี associated data

// DU ใน pattern matching ทรงพลังกว่า
let executeCommand cmd =
    match cmd with
    | Move(North, d) -> printfn "Moving north %.1f units" d
    | Move(South, d) -> printfn "Moving south %.1f units" d
    | Move(dir, d) -> printfn "Moving %A %.1f units" dir d
    | Turn degrees -> printfn "Turning %.1f degrees" degrees
    | Stop -> printfn "Stopping"
    | SetSpeed mph -> printfn "Speed: %.1f mph" mph

executeCommand (Move(North, 10.0))
executeCommand (Turn 90.0)
executeCommand Stop
executeCommand (SetSpeed 60.0)

// DU with methods
type Shape2 =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base_: float * height: float

    member this.Area =
        match this with
        | Circle r -> System.Math.PI * r * r
        | Rectangle(w, h) -> w * h
        | Triangle(b, h) -> 0.5 * b * h

// Enum ทำแบบนี้ไม่ได้
let shape = Circle 5.0
printfn "Area: %.2f" shape.Area
```

---

## 12. When to Use Each

```fsharp
// ควรใช้ Enum เมื่อ:
// 1. Interop กับ .NET libraries หรือ P/Invoke
// 2. ต้องการ flags/bitwise operations
// 3. ค่าต้องเป็น integer ที่กำหนดไว้ล่วงหน้า
// 4. Serialization/deserialization ที่ต้องการ integer values
// 5. Mapping กับ database enum columns

// ควรใช้ DU เมื่อ:
// 1. Cases มี different shapes (associated data)
// 2. ต้องการ exhaustive pattern matching
// 3. Type safety มีความสำคัญมาก
// 4. Modeling complex domain types

// ตัวอย่าง: Database status column ใช้ enum
type OrderStatus =
    | Pending = 0
    | Processing = 1
    | Shipped = 2
    | Delivered = 3
    | Cancelled = 4
    | Refunded = 5

// ตัวอย่าง: Domain model ใช้ DU
type OrderState =
    | Pending
    | Processing of processorId: int
    | Shipped of trackingNumber: string * carrier: string
    | Delivered of deliveredAt: System.DateTime
    | Cancelled of reason: string
    | Refunded of refundId: string * amount: decimal

// Convert ระหว่าง DU และ Enum (สำหรับ persistence)
let orderStateToStatus (state: OrderState) =
    match state with
    | Pending -> OrderStatus.Pending
    | Processing _ -> OrderStatus.Processing
    | Shipped _ -> OrderStatus.Shipped
    | Delivered _ -> OrderStatus.Delivered
    | Cancelled _ -> OrderStatus.Cancelled
    | Refunded _ -> OrderStatus.Refunded

let orderStatusToState (status: OrderStatus) =
    match status with
    | OrderStatus.Pending -> Some Pending
    | OrderStatus.Processing -> None  // Need more data for full DU
    | OrderStatus.Shipped -> None     // Need tracking number, carrier
    | OrderStatus.Delivered -> None   // Need delivery date
    | OrderStatus.Cancelled -> None   // Need reason
    | OrderStatus.Refunded -> None    // Need refund details
    | _ -> None

// ทดสอบ
let states = [
    Pending
    Processing 42
    Shipped("TH123456789", "DHL")
    Delivered(System.DateTime.Today)
]

printfn "Order states with DB status:"
for state in states do
    printfn "  %A -> %A" state (orderStateToStatus state)
```

---

## 13. Advanced Enum Patterns

```fsharp
// Enum helper functions
module EnumHelper =
    let values<'T when 'T : struct and 'T :> System.Enum> () =
        System.Enum.GetValues<'T>() |> Array.toList
    
    let names<'T when 'T : struct and 'T :> System.Enum> () =
        System.Enum.GetNames(typeof<'T>) |> Array.toList
    
    let pairs<'T when 'T : struct and 'T :> System.Enum> () =
        let vals = values<'T>()
        let names = names<'T>()
        List.zip names vals
    
    let tryParse<'T when 'T : struct and 'T :> System.Enum> (s: string) =
        match System.Enum.TryParse<'T>(s, ignoreCase = true) with
        | true, v -> Some v
        | _ -> None
    
    let isDefined<'T when 'T : struct and 'T :> System.Enum> (value: int) =
        System.Enum.IsDefined(typeof<'T>, value)
    
    let fromInt<'T when 'T : struct and 'T :> System.Enum> (value: int) =
        if isDefined<'T> value then Some (enum<'T> value)
        else None
    
    let description<'T when 'T : struct and 'T :> System.Enum> (value: 'T) =
        let memberInfo = typeof<'T>.GetMember(string value)
        if memberInfo.Length > 0 then
            let descAttr = memberInfo.[0].GetCustomAttributes(typeof<System.ComponentModel.DescriptionAttribute>, false)
            if descAttr.Length > 0 then
                (descAttr.[0] :?> System.ComponentModel.DescriptionAttribute).Description
            else
                string value
        else
            string value

// ใช้ helper
printfn "All days:"
EnumHelper.pairs<DayOfWeek>()
|> List.iter (fun (name, value) -> printfn "  %s = %d" name (int value))

printfn "\nParsing:"
printfn "  'monday' -> %A" (EnumHelper.tryParse<DayOfWeek> "monday")
printfn "  'FRIDAY' -> %A" (EnumHelper.tryParse<DayOfWeek> "FRIDAY")
printfn "  'invalid' -> %A" (EnumHelper.tryParse<DayOfWeek> "invalid")

printfn "\nFrom int:"
printfn "  0 -> %A" (EnumHelper.fromInt<DayOfWeek> 0)
printfn "  6 -> %A" (EnumHelper.fromInt<DayOfWeek> 6)
printfn "  7 -> %A" (EnumHelper.fromInt<DayOfWeek> 7)

// Enum range
let workdaysInMonth (year: int) (month: Month) =
    let daysInMonth = System.DateTime.DaysInMonth(year, int month)
    [1..daysInMonth]
    |> List.filter (fun day ->
        let date = System.DateTime(year, int month, day)
        let dow = enum<DayOfWeek> (int date.DayOfWeek)
        dow <> DayOfWeek.Saturday && dow <> DayOfWeek.Sunday
    )
    |> List.length

printfn "\nWorkdays:"
for month in [Month.January..Month.December] do
    printfn "  %A 2024: %d workdays" month (workdaysInMonth 2024 month)
```

---

## สรุป (Summary)

```fsharp
printfn "=== Enum Summary ==="
printfn ""
printfn "Basic enum:"
printfn "  type Status ="
printfn "      | Active = 0"
printfn "      | Inactive = 1"
printfn ""
printfn "Flags enum:"
printfn "  [<System.Flags>]"
printfn "  type Permission ="
printfn "      | None = 0"
printfn "      | Read = 1"
printfn "      | Write = 2"
printfn ""
printfn "Operations:"
printfn "  int myEnum       -> convert to int"
printfn "  enum<T> intVal   -> convert from int"
printfn "  a ||| b          -> combine flags"
printfn "  a &&& b          -> check flag"
printfn "  a ^^^ b          -> toggle flag"
printfn "  ~~~a             -> invert flags"
printfn ""
printfn "Enum vs DU:"
printfn "  Enum: integral, flags, interop"
printfn "  DU: associated data, exhaustive matching"
```

---

## บทสรุป

Enum ใน F# เป็นเครื่องมือที่มีประโยชน์เมื่อ:
1. **Interop** กับ .NET libraries ที่ใช้ enum
2. **Flags** สำหรับ bitwise combinations
3. **Integer mapping** สำหรับ database หรือ protocol
4. **Simple constants** ที่ไม่ต้องการ associated data

แต่สำหรับ domain modeling ใน F# แบบ idiomatic ควรใช้ **Discriminated Unions** เพราะทรงพลังกว่าและ type-safe กว่า
