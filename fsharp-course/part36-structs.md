# Part 36 - สตรัคต์ (Structs)

## บทนำ (Introduction)

Structs เป็น value types ที่ allocate บน stack (หรือ inline ใน heap) แทนที่จะเป็น reference types ที่ allocate บน heap เสมอ การใช้ structs อย่างเหมาะสมช่วยลด garbage collection pressure และเพิ่มประสิทธิภาพ

Structs are value types that are allocated on the stack (or inline in the heap) instead of always being heap-allocated like reference types. Using structs appropriately reduces GC pressure and improves performance.

---

## 1. [<Struct>] Attribute

```fsharp
// การสร้าง struct ใน F#

// Struct record
[<Struct>]
type Point2D = {
    X: float
    Y: float
}

// Struct discriminated union
[<Struct>]
type Color =
    | RGB of r: byte * g: byte * b: byte
    | Grayscale of gray: byte

// การใช้งาน
let p1 = { X = 3.0; Y = 4.0 }
let p2 = { X = 6.0; Y = 8.0 }

let distance (a: Point2D) (b: Point2D) =
    let dx = a.X - b.X
    let dy = a.Y - b.Y
    sqrt (dx * dx + dy * dy)

printfn "Distance: %.2f" (distance p1 p2)

// ตรวจสอบว่าเป็น value type
printfn "Point2D is struct: %b" typeof<Point2D>.IsValueType

let red = RGB(255uy, 0uy, 0uy)
let gray = Grayscale(128uy)

match red with
| RGB(r, g, b) -> printfn "Red: r=%d, g=%d, b=%d" r g b
| Grayscale(v) -> printfn "Gray: %d" v
```

---

## 2. Value Types vs Reference Types

```fsharp
// Value types (struct) vs Reference types (class)

// Reference type - allocated on heap
type PointRef(x: float, y: float) =
    member this.X = x
    member this.Y = y

// Value type - allocated on stack (or inline)
[<Struct>]
type PointVal = {
    X: float
    Y: float
}

// ความแตกต่าง: Value semantics
let v1 = { X = 1.0; Y = 2.0 }
let v2 = v1  // Copy! v2 is independent of v1

// Reference semantics
let r1 = PointRef(1.0, 2.0)
let r2 = r1  // r2 points to same object as r1

printfn "Value type copy semantics:"
printfn "v1.X = %.1f, v2.X = %.1f" v1.X v2.X

// Memory layout test
let values = Array.init 5 (fun i -> { X = float i; Y = float i * 2.0 })
printfn "\nValue array:"
for v in values do
    printf "(%.0f,%.0f) " v.X v.Y
printfn ""

// Performance comparison
open System.Diagnostics

let N = 1_000_000

// Reference type array
let sw1 = Stopwatch.StartNew()
let refs = Array.init N (fun i -> PointRef(float i, float i))
let mutable sum1 = 0.0
for r in refs do
    sum1 <- sum1 + r.X + r.Y
sw1.Stop()
printfn "\nReference: %dms, sum=%.0f" sw1.ElapsedMilliseconds sum1

// Value type array
let sw2 = Stopwatch.StartNew()
let vals = Array.init N (fun i -> { X = float i; Y = float i })
let mutable sum2 = 0.0
for v in vals do
    sum2 <- sum2 + v.X + v.Y
sw2.Stop()
printfn "Value: %dms, sum=%.0f" sw2.ElapsedMilliseconds sum2
```

---

## 3. Struct Records

```fsharp
// Struct records - immutable value types

[<Struct>]
type Vector3 = {
    X: float32
    Y: float32
    Z: float32
}

// Operations on struct records
let add (a: Vector3) (b: Vector3) =
    { X = a.X + b.X; Y = a.Y + b.Y; Z = a.Z + b.Z }

let scale (factor: float32) (v: Vector3) =
    { X = v.X * factor; Y = v.Y * factor; Z = v.Z * factor }

let dot (a: Vector3) (b: Vector3) =
    a.X * b.X + a.Y * b.Y + a.Z * b.Z

let magnitude (v: Vector3) =
    sqrt (v.X * v.X + v.Y * v.Y + v.Z * v.Z)

let normalize (v: Vector3) =
    let mag = magnitude v
    { X = v.X / mag; Y = v.Y / mag; Z = v.Z / mag }

let cross (a: Vector3) (b: Vector3) =
    { X = a.Y * b.Z - a.Z * b.Y
      Y = a.Z * b.X - a.X * b.Z
      Z = a.X * b.Y - a.Y * b.X }

let v1 = { X = 1.0f; Y = 0.0f; Z = 0.0f }
let v2 = { X = 0.0f; Y = 1.0f; Z = 0.0f }

printfn "v1 = (%.1f, %.1f, %.1f)" v1.X v1.Y v1.Z
printfn "v2 = (%.1f, %.1f, %.1f)" v2.X v2.Y v2.Z
printfn "v1 + v2 = %A" (add v1 v2)
printfn "dot(v1, v2) = %.1f" (dot v1 v2)  // 0 (perpendicular)
printfn "cross(v1, v2) = %A" (cross v1 v2)  // (0, 0, 1)

// Struct record with methods via member
[<Struct>]
type Rectangle = {
    X: float
    Y: float
    Width: float
    Height: float
}

// Module สำหรับ operations (idiomatic F#)
module Rectangle =
    let area (r: Rectangle) = r.Width * r.Height
    let perimeter (r: Rectangle) = 2.0 * (r.Width + r.Height)
    let contains (point: Point2D) (r: Rectangle) =
        point.X >= r.X && point.X <= r.X + r.Width &&
        point.Y >= r.Y && point.Y <= r.Y + r.Height
    let translate dx dy (r: Rectangle) =
        { r with X = r.X + dx; Y = r.Y + dy }

let rect = { X = 0.0; Y = 0.0; Width = 10.0; Height = 5.0 }
printfn "\nRectangle area: %.1f" (Rectangle.area rect)
printfn "Rectangle perimeter: %.1f" (Rectangle.perimeter rect)
```

---

## 4. Struct Discriminated Unions

```fsharp
// Struct DU - เหมาะสำหรับ small discriminated unions
[<Struct>]
type Result2<'T> =
    | Ok2 of value: 'T
    | Error2 of error: string

[<Struct>]
type Maybe<'T> =
    | Just of item: 'T
    | Nothing

// Struct DU สำหรับ token types
[<Struct>]
type Token =
    | Number of num: float
    | Operator of op: char
    | LeftParen
    | RightParen
    | Identifier of name: string

// ใช้งาน
let tokens = [
    Number 3.14
    Operator '+'
    Number 2.0
    Operator '*'
    LeftParen
    Number 5.0
    Operator '-'
    Number 1.0
    RightParen
]

for token in tokens do
    match token with
    | Number n -> printf "NUM(%.2f) " n
    | Operator op -> printf "OP(%c) " op
    | LeftParen -> printf "( "
    | RightParen -> printf ") "
    | Identifier name -> printf "ID(%s) " name
printfn ""

// Struct DU สำหรับ option type
let divide a b =
    if b = 0 then Nothing
    else Just (a / b)

match divide 10 3 with
| Just result -> printfn "Result: %d" result
| Nothing -> printfn "Division by zero"

// Compute statistics ด้วย struct DU
[<Struct>]
type StatResult =
    | Computed of mean: float * stddev: float
    | InsufficientData

let computeStats (data: float array) =
    if data.Length < 2 then InsufficientData
    else
        let mean = Array.average data
        let variance = data |> Array.averageBy (fun x -> (x - mean) ** 2.0)
        Computed(mean, sqrt variance)

let data = [| 2.0; 4.0; 4.0; 4.0; 5.0; 5.0; 7.0; 9.0 |]
match computeStats data with
| Computed(mean, stddev) -> printfn "Mean=%.2f, StdDev=%.2f" mean stddev
| InsufficientData -> printfn "Not enough data"
```

---

## 5. Struct Tuples

```fsharp
// Struct tuples - ไม่สร้าง heap allocation

// F# struct tuple ใช้ struct keyword
let structTuple = struct (1, "hello", 3.14)

// หรือใช้ ValueTuple ของ .NET
let vt1 = System.ValueTuple.Create(1, "hello")
let vt2 = System.ValueTuple.Create(1, 2, 3)

// Struct tuple ใน function
let divRem (x: int) (y: int) =
    struct (x / y, x % y)

let struct (quotient, remainder) = divRem 17 5
printfn "17 / 5 = %d remainder %d" quotient remainder

// Comparison: Reference tuple vs Struct tuple
let refTuple = (1, 2, 3)  // Reference tuple
let strTuple = struct (1, 2, 3)  // Struct tuple

printfn "Ref tuple is value type: %b" (refTuple.GetType().IsValueType)  // false
printfn "Struct tuple is value type: %b" (strTuple.GetType().IsValueType)  // true

// High-performance code ที่ใช้ struct tuples
let computeMinMax (data: float array) =
    if data.Length = 0 then failwith "Empty array"
    let mutable min = data.[0]
    let mutable max = data.[0]
    for x in data do
        if x < min then min <- x
        if x > max then max <- x
    struct (min, max)

let data2 = [| 3.0; 1.0; 4.0; 1.0; 5.0; 9.0; 2.0; 6.0 |]
let struct (minimum, maximum) = computeMinMax data2
printfn "Min: %.1f, Max: %.1f" minimum maximum
```

---

## 6. When to Use Structs

```fsharp
// ใช้ struct เมื่อ:
// 1. ขนาดเล็ก (≤ 16 bytes แนะนำ)
// 2. Immutable โดยทั่วไป
// 3. ไม่ต้องการ reference semantics
// 4. Short-lived หรือ embedded ใน arrays

// Good use cases:
[<Struct>] type Coordinate = { Lat: float; Lon: float }  // 16 bytes - good
[<Struct>] type Color4 = { R: byte; G: byte; B: byte; A: byte }  // 4 bytes - great
[<Struct>] type DateRange = { Start: System.DateTime; End: System.DateTime }  // 16 bytes - ok

// Avoid struct when:
// 1. ขนาดใหญ่ (> 16 bytes มักจะ slow เพราะ copy cost)
// 2. ต้องการ polymorphism/inheritance
// 3. ถูก boxed บ่อย (ใน collection ที่เป็น obj)

// Performance test: small struct vs class
[<Struct>]
type SmallPoint = { SX: float32; SY: float32 }  // 8 bytes

type LargeStruct = {
    mutable F1: int64; mutable F2: int64; mutable F3: int64; mutable F4: int64
    mutable F5: int64; mutable F6: int64; mutable F7: int64; mutable F8: int64
}  // 64 bytes - too large for struct benefit

// Demonstrate: struct in array = no boxing, contiguous memory
let coordinates = Array.init 10 (fun i ->
    { Lat = float i * 10.0; Lon = float i * 20.0 })

printfn "Coordinates:"
for coord in coordinates do
    printfn "  (%.1f°N, %.1f°E)" coord.Lat coord.Lon

// Struct appropriate for particles in game/simulation
[<Struct>]
type Particle = {
    X: float32
    Y: float32
    VX: float32
    VY: float32
}

let mutable particles = Array.init 1000 (fun i ->
    { X = float32 i; Y = 0.0f; VX = float32 (i % 5); VY = 1.0f })

// Update particles - no boxing, cache friendly
let dt = 0.016f  // 60fps delta time
for i in 0..particles.Length - 1 do
    particles.[i] <- {
        particles.[i] with
            X = particles.[i].X + particles.[i].VX * dt
            Y = particles.[i].Y + particles.[i].VY * dt
    }

printfn "Updated first particle: (%.3f, %.3f)" particles.[0].X particles.[0].Y
```

---

## 7. ValueOption<'T>

```fsharp
// ValueOption - struct version of Option
// F# 5+ มี ValueOption built-in

// ใช้ ValueOption แทน Option สำหรับ performance-critical code
let tryParseInt (s: string) : ValueOption<int> =
    match System.Int32.TryParse(s) with
    | true, v -> ValueSome v
    | _ -> ValueNone

let parseNumbers (input: string list) =
    input
    |> List.choose (fun s ->
        match tryParseInt s with
        | ValueSome v -> Some v
        | ValueNone -> None
    )

let inputs = ["1"; "abc"; "2"; "xyz"; "3"; "4"; "five"]
let numbers = parseNumbers inputs
printfn "Parsed numbers: %A" numbers

// Custom ValueOption operations
let valueOptionMap (f: 'T -> 'U) (opt: ValueOption<'T>) : ValueOption<'U> =
    match opt with
    | ValueSome x -> ValueSome (f x)
    | ValueNone -> ValueNone

let valueOptionBind (f: 'T -> ValueOption<'U>) (opt: ValueOption<'T>) : ValueOption<'U> =
    match opt with
    | ValueSome x -> f x
    | ValueNone -> ValueNone

let safeHead (lst: 'T list) =
    match lst with
    | [] -> ValueNone
    | x :: _ -> ValueSome x

let safeTail (lst: 'T list) =
    match lst with
    | [] -> ValueNone
    | _ :: rest -> ValueSome rest

// ใช้ ValueOption ใน chain
let firstEven (nums: int list) =
    nums |> List.tryFind (fun n -> n % 2 = 0) |> Option.map ValueSome |> Option.defaultValue ValueNone

let result = firstEven [1; 3; 4; 6; 7]
match result with
| ValueSome n -> printfn "First even: %d" n
| ValueNone -> printfn "No even number"
```

---

## 8. Memory Layout

```fsharp
open System.Runtime.InteropServices

// Memory layout ของ structs
[<Struct>]
type Layout1 = {
    ByteField: byte     // 1 byte
    IntField: int       // 4 bytes
    LongField: int64    // 8 bytes
}  // Total with padding: ~16 bytes

[<Struct; StructLayout(LayoutKind.Sequential)>]
type PackedLayout = {
    ByteField: byte
    IntField: int
    LongField: int64
}

// ตรวจสอบขนาด
printfn "Layout1 size: %d bytes" (Marshal.SizeOf<Layout1>())
printfn "PackedLayout size: %d bytes" (Marshal.SizeOf<PackedLayout>())

// Fixed-size arrays in structs (สำหรับ P/Invoke)
[<Struct>]
type FixedBuffer =
    val Buffer: int64  // Only for demonstration

// Array of structs vs struct of arrays
// Array of Structs (AoS) - แต่ละ element มี data ครบ
[<Struct>]
type AoSParticle = {
    X: float32; Y: float32
    VX: float32; VY: float32
    Mass: float32
}

// Struct of Arrays (SoA) - แยก arrays ตาม field
type SoAParticles = {
    X: float32 array
    Y: float32 array
    VX: float32 array
    VY: float32 array
    Mass: float32 array
}

let N = 10_000

// AoS approach
let sw_aos = Stopwatch.StartNew()
let aosParticles = Array.init N (fun i ->
    { X = float32 i; Y = 0.0f; VX = 1.0f; VY = 0.0f; Mass = 1.0f })

let mutable totalKE_aos = 0.0f
for p in aosParticles do
    totalKE_aos <- totalKE_aos + 0.5f * p.Mass * (p.VX * p.VX + p.VY * p.VY)
sw_aos.Stop()
printfn "\nAoS time: %dms, KE=%.0f" sw_aos.ElapsedMilliseconds totalKE_aos

// SoA approach
let sw_soa = Stopwatch.StartNew()
let soaParticles = {
    X = Array.init N float32
    Y = Array.zeroCreate N
    VX = Array.create N 1.0f
    VY = Array.zeroCreate N
    Mass = Array.create N 1.0f
}

let mutable totalKE_soa = 0.0f
for i in 0..N-1 do
    totalKE_soa <- totalKE_soa + 0.5f * soaParticles.Mass.[i] * 
                   (soaParticles.VX.[i] * soaParticles.VX.[i] + soaParticles.VY.[i] * soaParticles.VY.[i])
sw_soa.Stop()
printfn "SoA time: %dms, KE=%.0f" sw_soa.ElapsedMilliseconds totalKE_soa
```

---

## 9. Avoiding Boxing

```fsharp
// Boxing เกิดขึ้นเมื่อ value type ถูก treat as object

// Boxing situations to avoid:
// 1. Storing value types in object collections
let boxingBad = System.Collections.ArrayList()
boxingBad.Add(42)  // Boxing! 42 becomes object

// Better: use generic collections
let boxingGood = System.Collections.Generic.List<int>()
boxingGood.Add(42)  // No boxing!

// 2. Interface dispatch with struct
[<Struct>]
type Counter2 = { Count: int }

// This causes boxing when calling interface methods on struct
// type IIncrement =
//     abstract member Increment: unit -> IIncrement

// Better pattern for struct mutations - return new value
let increment (c: Counter2) = { Count = c.Count + 1 }

let mutable c = { Count = 0 }
for _ in 1..5 do
    c <- increment c
printfn "Counter: %d" c.Count

// 3. Avoid boxing in hot paths
let processInts (data: int array) =
    // Bad: Array.map uses object boxing internally if not careful
    // data |> Array.map (fun x -> x * 2)  // might box
    
    // Good: explicit array allocation
    let result = Array.zeroCreate data.Length
    for i in 0..data.Length - 1 do
        result.[i] <- data.[i] * 2
    result

// 4. struct constraints prevent boxing
let sumValues<'T when 'T : struct and 'T :> System.IComparable<'T>> 
    (values: 'T array) (zero: 'T) =
    // Generic function that works on structs without boxing
    values.Length

// Generic arithmetic with structs (using SRTP)
let inline sum (xs: ^T array) : ^T =
    Array.fold (fun acc x -> acc + x) LanguagePrimitives.GenericZero xs

let intSum = sum [| 1; 2; 3; 4; 5 |]
let floatSum = sum [| 1.0; 2.0; 3.0 |]
printfn "Int sum: %d" intSum
printfn "Float sum: %.1f" floatSum
```

---

## 10. Interop with P/Invoke

```fsharp
open System.Runtime.InteropServices

// P/Invoke ใช้ structs สำหรับ interop กับ native code

// Native struct (matching C struct layout)
[<Struct; StructLayout(LayoutKind.Sequential)>]
type POINT =
    val X: int32
    val Y: int32
    new(x, y) = { X = x; Y = y }

[<Struct; StructLayout(LayoutKind.Sequential)>]
type RECT =
    val Left: int32
    val Top: int32
    val Right: int32
    val Bottom: int32
    new(l, t, r, b) = { Left = l; Top = t; Right = r; Bottom = b }

// Windows API calls (โดยทั่วไปใช้บน Windows)
// ตัวอย่างไม่ run จริงเพื่อ cross-platform compatibility
// [<DllImport("user32.dll")>]
// extern bool GetCursorPos(POINT* lpPoint)

// [<DllImport("user32.dll")>]
// extern bool GetWindowRect(nativeint hWnd, RECT* lpRect)

// Simulated P/Invoke usage
let convertToNativePoint (x: float, y: float) =
    POINT(int x, int y)

let convertFromNativePoint (pt: POINT) =
    (float pt.X, float pt.Y)

let p = convertToNativePoint (3.7, 4.2)
printfn "Native POINT: (%d, %d)" p.X p.Y

let (fx, fy) = convertFromNativePoint p
printfn "Back to float: (%.1f, %.1f)" fx fy

// Buffer struct สำหรับ binary data
[<Struct; StructLayout(LayoutKind.Sequential, Pack = 1)>]
type NetworkPacket =
    val Version: byte
    val Type: byte
    val Length: uint16
    val Checksum: uint32

let createPacket (packetType: byte) (payloadLength: int) =
    let mutable packet = NetworkPacket()
    // Can't easily initialize struct fields in F# this way
    // Using explicit approach
    printfn "Packet size: %d bytes" (Marshal.SizeOf<NetworkPacket>())
    packet

// Marshal.PtrToStructure and StructureToPtr for binary serialization
let serializeToBytes<'T when 'T : struct>(value: 'T) =
    let size = Marshal.SizeOf<'T>()
    let bytes = Array.zeroCreate<byte> size
    let ptr = Marshal.AllocHGlobal(size)
    try
        Marshal.StructureToPtr(value, ptr, false)
        Marshal.Copy(ptr, bytes, 0, size)
        bytes
    finally
        Marshal.FreeHGlobal(ptr)

let deserializeFromBytes<'T when 'T : struct>(bytes: byte array) =
    let size = Marshal.SizeOf<'T>()
    let ptr = Marshal.AllocHGlobal(size)
    try
        Marshal.Copy(bytes, 0, ptr, size)
        Marshal.PtrToStructure<'T>(ptr)
    finally
        Marshal.FreeHGlobal(ptr)

let pt = POINT(42, 100)
let bytes = serializeToBytes pt
printfn "Serialized POINT to %d bytes: %A" bytes.Length bytes

let deserializedPt = deserializeFromBytes<POINT> bytes
printfn "Deserialized: (%d, %d)" deserializedPt.X deserializedPt.Y
```

---

## 11. Span<T> and Memory<T>

```fsharp
open System

// Span<T> - slice ของ memory (stack-only)
// Memory<T> - heap-safe version ของ Span

// การใช้ Span<T>
let useSpan () =
    let data = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]
    let span = Span<int>(data)
    
    // Slice without copying
    let slice = span.Slice(2, 5)  // elements 3..7
    
    printfn "Original length: %d" data.Length
    printfn "Span length: %d" span.Length
    printfn "Slice length: %d" slice.Length
    
    // Access elements
    for i in 0..slice.Length - 1 do
        printf "%d " slice.[i]
    printfn ""
    
    // Modify through span
    slice.[0] <- 99
    printfn "After modification: %A" data

useSpan()

// Memory<T> - heap-safe, can be stored in fields
type Buffer<'T>(size: int) =
    let data = Array.zeroCreate<'T> size
    
    member this.Memory: Memory<'T> = Memory<'T>(data)
    member this.Length = size
    
    member this.Slice(start: int, length: int) =
        Memory<'T>(data, start, length)

let buffer = Buffer<byte>(1024)
let slice1 = buffer.Slice(0, 512)
let slice2 = buffer.Slice(512, 512)

printfn "Buffer: %d bytes" buffer.Length
printfn "Slice1: %d bytes" slice1.Length
printfn "Slice2: %d bytes" slice2.Length

// String as Span<char>
let processString (input: string) =
    let span = input.AsSpan()
    
    // Find first space without allocation
    let spaceIndex = span.IndexOf(' ')
    
    if spaceIndex >= 0 then
        let firstName = span.Slice(0, spaceIndex).ToString()
        let lastName = span.Slice(spaceIndex + 1).ToString()
        printfn "First: %s, Last: %s" firstName lastName
    else
        printfn "Single word: %s" input

processString "John Doe"
processString "Alice"

// ReadOnlySpan<char> for parsing
let parseInt (input: ReadOnlySpan<char>) =
    let mutable result = 0
    for c in input do
        if System.Char.IsDigit(c) then
            result <- result * 10 + (int c - int '0')
    result

let numStr = "12345".AsSpan()
let num = parseInt numStr
printfn "Parsed: %d" num
```

---

## 12. ByRef Types

```fsharp
// byref<'T>, inref<'T>, outref<'T>

// byref - mutable reference (in or out)
let increment (x: byref<int>) =
    x <- x + 1

let mutable value = 5
increment &value
printfn "After increment: %d" value  // 6

// inref<'T> - readonly reference (optimization)
let readValue (x: inref<int>) =
    printfn "Reading: %d" x

readValue &value  // No copy!

// outref<'T> - write-only reference
let computeResult (input: int) (result: outref<int>) (error: outref<string>) =
    if input < 0 then
        result <- 0
        error <- "Negative input"
    else
        result <- input * input
        error <- null

let mutable result = 0
let mutable error = ""
computeResult 5 &result &error
printfn "Result: %d, Error: %s" result (if isNull error then "none" else error)

computeResult -3 &result &error
printfn "Result: %d, Error: %s" result (if isNull error then "none" else error)

// Struct methods with byref
[<Struct>]
type MutablePoint3D = {
    mutable X: float
    mutable Y: float
    mutable Z: float
}

let translate (point: byref<MutablePoint3D>) (dx: float) (dy: float) (dz: float) =
    point <- { X = point.X + dx; Y = point.Y + dy; Z = point.Z + dz }

let mutable mp = { X = 1.0; Y = 2.0; Z = 3.0 }
printfn "Before: (%.1f, %.1f, %.1f)" mp.X mp.Y mp.Z
translate &mp 5.0 3.0 1.0
printfn "After: (%.1f, %.1f, %.1f)" mp.X mp.Y mp.Z

// NativeInterop ด้วย byref
// [<DllImport("some.dll")>]
// extern void setValueByRef(int& value, int newValue)
```

---

## 13. inref and outref

```fsharp
// inref - F# 4.5+ read-only by-reference

// inref สำหรับ large structs (avoid copy)
[<Struct>]
type BigStruct = {
    A: float; B: float; C: float; D: float
    E: float; F: float; G: float; H: float
}  // 64 bytes

// ไม่มี inref - copy big struct
let processNormal (s: BigStruct) =
    s.A + s.B + s.C + s.D + s.E + s.F + s.G + s.H

// มี inref - no copy!
let processWithInRef (s: inref<BigStruct>) =
    s.A + s.B + s.C + s.D + s.E + s.F + s.G + s.H

let bigStruct = { A = 1.0; B = 2.0; C = 3.0; D = 4.0; E = 5.0; F = 6.0; G = 7.0; H = 8.0 }
let sum1 = processNormal bigStruct  // Copies 64 bytes
let sum2 = processWithInRef &bigStruct  // No copy
printfn "Sum (normal): %.1f" sum1
printfn "Sum (inref): %.1f" sum2

// outref pattern
let divmod (dividend: int) (divisor: int) (quotient: outref<int>) (remainder: outref<int>) =
    quotient <- dividend / divisor
    remainder <- dividend % divisor

let mutable q = 0
let mutable r = 0
divmod 17 5 &q &r
printfn "17 / 5 = %d remainder %d" q r

// TryParse pattern with outref
let tryParseInt2 (s: string) (result: outref<int>) =
    match System.Int32.TryParse(s) with
    | true, v ->
        result <- v
        true
    | _ ->
        result <- 0
        false

let mutable parsed = 0
if tryParseInt2 "42" &parsed then
    printfn "Parsed: %d" parsed
else
    printfn "Failed to parse"
```

---

## 14. Practical: High-Performance Matrix

```fsharp
// High-performance matrix ด้วย structs

[<Struct>]
type Matrix2x2 = {
    M00: float; M01: float
    M10: float; M11: float
}

module Matrix2x2 =
    let zero = { M00 = 0.0; M01 = 0.0; M10 = 0.0; M11 = 0.0 }
    let identity = { M00 = 1.0; M01 = 0.0; M10 = 0.0; M11 = 1.0 }
    
    let multiply (a: Matrix2x2) (b: Matrix2x2) =
        { M00 = a.M00 * b.M00 + a.M01 * b.M10
          M01 = a.M00 * b.M01 + a.M01 * b.M11
          M10 = a.M10 * b.M00 + a.M11 * b.M10
          M11 = a.M10 * b.M01 + a.M11 * b.M11 }
    
    let transform (m: Matrix2x2) (x: float, y: float) =
        (m.M00 * x + m.M01 * y, m.M10 * x + m.M11 * y)
    
    let rotation (theta: float) =
        let cos = System.Math.Cos(theta)
        let sin = System.Math.Sin(theta)
        { M00 = cos; M01 = -sin; M10 = sin; M11 = cos }
    
    let scale (sx: float) (sy: float) =
        { M00 = sx; M01 = 0.0; M10 = 0.0; M11 = sy }
    
    let determinant (m: Matrix2x2) =
        m.M00 * m.M11 - m.M01 * m.M10

let rot45 = Matrix2x2.rotation (System.Math.PI / 4.0)
let scaled = Matrix2x2.scale 2.0 3.0

let combined = Matrix2x2.multiply rot45 scaled
let (tx, ty) = Matrix2x2.transform combined (1.0, 0.0)

printfn "Transform result: (%.3f, %.3f)" tx ty
printfn "Determinant: %.3f" (Matrix2x2.determinant combined)

// Batch transform - no boxing, cache friendly
let points = Array.init 1000 (fun i -> (float i * 0.01, float i * 0.01))
let transformedPoints = Array.map (Matrix2x2.transform rot45) points
printfn "Transformed %d points" transformedPoints.Length
printfn "First: (%.3f, %.3f)" (fst transformedPoints.[0]) (snd transformedPoints.[0])
```

---

## สรุป (Summary)

```fsharp
printfn "=== Structs Summary ==="
printfn ""
printfn "Creating structs:"
printfn "  [<Struct>] type Point = { X: float; Y: float }"
printfn "  [<Struct>] type Option2<'T> = Some2 of 'T | None2"
printfn "  struct (1, 2, 3)  // struct tuple"
printfn ""
printfn "When to use structs:"
printfn "  - Small types (≤ 16 bytes)"
printfn "  - Frequently created/destroyed"
printfn "  - Value semantics needed"
printfn "  - Performance-critical arrays"
printfn "  - P/Invoke interop"
printfn ""
printfn "When to avoid structs:"
printfn "  - Large types (> 16 bytes)"
printfn "  - Polymorphism needed"
printfn "  - Frequent boxing"
printfn "  - Mutable shared state"
printfn ""
printfn "Special types:"
printfn "  - ValueOption<'T> - struct version of Option"
printfn "  - Span<T> - stack-only memory slice"
printfn "  - Memory<T> - heap-safe memory slice"
printfn "  - byref/inref/outref - by-reference parameters"
```

---

## บทสรุป

Structs ใน F# มีประโยชน์ในด้าน:
1. **Performance** - ลด GC pressure, better cache locality
2. **Interop** - matching native struct layouts
3. **Value semantics** - copy-by-value behavior
4. **Memory efficiency** - contiguous arrays
5. **Span<T>/Memory<T>** - zero-copy memory operations

การเลือกใช้ struct vs class ต้องพิจารณา size, usage pattern, และ semantics ที่ต้องการ
