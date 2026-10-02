# Part 3 - ประเภทข้อมูลและการอนุมานประเภท (Types and Type Inference)

## บทนำ

F# มีระบบ type ที่แข็งแกร่งและ type inference ที่ทรงพลัง ทำให้เราไม่ต้องระบุ type บ่อยครั้ง แต่โปรแกรมยังคงถูกตรวจสอบ type อย่างเข้มงวดในระหว่าง compile ในบทนี้เราจะเรียนรู้ primitive types ทั้งหมด การแปลง type และ type inference ของ F#

---

## 3.1 Primitive Types พื้นฐาน

### 3.1.1 Integer Types

```fsharp
// int (32-bit signed integer) - ใช้บ่อยที่สุด
let n: int = 42
let negN: int = -17
let minInt = System.Int32.MinValue    // -2,147,483,648
let maxInt = System.Int32.MaxValue    // 2,147,483,647

printfn "int range: %d to %d" minInt maxInt

// int8 (8-bit signed: -128 to 127)
let tiny: int8 = 127y   // suffix y
let tinyMin = System.SByte.MinValue   // -128

// int16 (16-bit signed: -32768 to 32767)
let small: int16 = 1000s   // suffix s
let smallMax = System.Int16.MaxValue  // 32767

// int32 (same as int)
let medium: int32 = 2000000l  // suffix l (optional for int32)

// int64 (64-bit signed)
let large: int64 = 9000000000L   // suffix L
let veryLarge = System.Int64.MaxValue  // 9,223,372,036,854,775,807

printfn "int64 max: %d" veryLarge
printfn "Type of large: %s" (large.GetType().Name)

// nativeint (pointer-sized integer)
let ptr: nativeint = 0n   // suffix n
```

### 3.1.2 Unsigned Integer Types

```fsharp
// uint (32-bit unsigned: 0 to 4,294,967,295)
let u: uint = 4000000000u   // suffix u
printfn "uint: %u" u

// uint8 / byte (0 to 255)
let b: uint8 = 255uy   // suffix uy
let byteVal: byte = 200uy  // byte เป็น alias ของ uint8

// uint16 (0 to 65535)
let u16: uint16 = 60000us   // suffix us

// uint32
let u32: uint32 = 4000000000ul   // suffix ul

// uint64 (0 to 18,446,744,073,709,551,615)
let u64: uint64 = 18000000000000000000UL   // suffix UL

printfn "Unsigned types:"
printfn "  byte: %d" b
printfn "  uint: %u" u
printfn "  uint64: %u" u64
```

### 3.1.3 Floating Point Types

```fsharp
// float / float64 (64-bit double precision) - ใช้บ่อยที่สุด
let f: float = 3.14159265358979
let fExplicit: double = 2.71828  // double เป็น alias
let negF = -1.5
let bigF = 1.23e10    // scientific notation: 12,300,000,000

printfn "float: %f" f
printfn "Scientific: %e" bigF
printfn "Auto format: %g" bigF

// float32 (32-bit single precision)
let f32: float32 = 3.14f   // suffix f
let f32b: single = 2.71f   // single เป็น alias

printfn "float32: %f" f32
printfn "Type: %s" (f32.GetType().Name)

// decimal (128-bit precise decimal) - ใช้สำหรับการเงิน
let price: decimal = 1234.56M   // suffix M
let tax = 0.07M
let total = price * (1.0M + tax)

printfn "Price: %M" price
printfn "Total with tax: %M" total
printfn "Type: %s" (price.GetType().Name)
```

### 3.1.4 bool

```fsharp
// bool - true หรือ false เท่านั้น
let isTrue: bool = true
let isFalse: bool = false

printfn "bool: %b" isTrue
printfn "Type: %s" (isTrue.GetType().Name)
printfn "Size: %d bytes" sizeof<bool>

// bool operations
let and1 = true && false    // false
let or1 = true || false     // true
let not1 = not true          // false
let xor1 = true <> false     // true (XOR using inequality)
```

### 3.1.5 char

```fsharp
// char - single Unicode character
let c: char = 'A'
let thai: char = 'ก'
let space: char = ' '
let newline: char = '\n'
let tab: char = '\t'
let quote: char = '\''
let backslash: char = '\\'
let unicode: char = 'A'  // Unicode escape = 'A'

printfn "char: %c" c
printfn "Thai: %c" thai
printfn "Unicode value: %d" (int c)

// char operations
printfn "Is letter: %b" (System.Char.IsLetter('A'))
printfn "Is digit: %b" (System.Char.IsDigit('5'))
printfn "Is whitespace: %b" (System.Char.IsWhiteSpace(' '))
printfn "To upper: %c" (System.Char.ToUpper('a'))
printfn "To lower: %c" (System.Char.ToLower('Z'))
```

### 3.1.6 string

```fsharp
// string - immutable sequence of chars
let s: string = "Hello, F#!"
let empty: string = ""
let multiWord = "This is F# programming language"

// String properties
printfn "Length: %d" s.Length
printfn "Is empty: %b" (s = "")
printfn "Is null or empty: %b" (System.String.IsNullOrEmpty(s))
printfn "Is null or whitespace: %b" (System.String.IsNullOrWhiteSpace("   "))

// F# string functions
let upper = s.ToUpper()
let lower = s.ToLower()
let trimmed = "  spaced  ".Trim()
let replaced = s.Replace("Hello", "สวัสดี")
let split = multiWord.Split(' ')
let joined = System.String.Join("-", split)

printfn "Upper: %s" upper
printfn "Lower: %s" lower
printfn "Trimmed: '%s'" trimmed
printfn "Replaced: %s" replaced
printfn "Split count: %d" split.Length
printfn "Joined: %s" joined
```

### 3.1.7 unit type

```fsharp
// unit - เหมือน void ใน C#/C
// แสดงว่าไม่มี return value ที่มีความหมาย

// Function ที่ return unit
let printHello () : unit =
    printfn "Hello!"

// () คือ unit value เดียว
let u: unit = ()

// Functions ที่ return unit โดยปริยาย
let doSomething () =
    printfn "Doing something..."
    // implicit return ()

// unit ใน higher-order functions
let performAction (action: unit -> unit) =
    printfn "Before action"
    action ()
    printfn "After action"

performAction (fun () -> printfn "Custom action!")

// unit เป็น tuple member ได้ (แต่ไม่ค่อยมีประโยชน์)
let unusualTuple = (42, (), "hello")
```

---

## 3.2 Type Inference (การอนุมานประเภท)

### 3.2.1 พื้นฐาน Type Inference

```fsharp
// F# compiler สามารถ infer type ได้จาก context

// จาก literal values
let i = 42           // inferred: int
let f = 3.14         // inferred: float
let s = "hello"      // inferred: string
let b = true         // inferred: bool
let c = 'A'          // inferred: char

// แสดง inferred types ใน FSI:
// > let x = 42;;
// val x : int = 42

// จาก operations
let sum = 1 + 2          // inferred: int
let product = 3.0 * 4.0  // inferred: float
let text = "a" + "b"     // inferred: string

// จาก function return
let double x = x * 2     // inferred: int -> int
let doubleF x = x * 2.0  // inferred: float -> float
```

### 3.2.2 Type Inference ใน Functions

```fsharp
// Inference จาก function body
let isPositive x = x > 0      // inferred: int -> bool
let addTen x = x + 10         // inferred: int -> int
let greet name = "Hello " + name  // inferred: string -> string

// Inference จาก usage
let apply f x = f x           // inferred: ('a -> 'b) -> 'a -> 'b (generic)

// Inference จาก constraints
let sumList lst = lst |> List.sum  // inferred: works with summable types

// เมื่อ inference ไม่สามารถ determine type ได้ต้องระบุเอง
let parseNumber (s: string) = int s   // ต้องระบุ string
// let parseNumber s = int s  // Error: type ของ s ไม่ชัดเจน
```

### 3.2.3 Generics และ Type Inference

```fsharp
// F# infer generic types อัตโนมัติ
let identity x = x        // 'a -> 'a (fully generic)
let pair x y = (x, y)    // 'a -> 'b -> 'a * 'b
let first (x, _) = x     // 'a * 'b -> 'a
let second (_, y) = y    // 'a * 'b -> 'b

// ใช้งาน
let intId = identity 42         // int
let strId = identity "hello"    // string
let intPair = pair 1 "two"      // int * string

printfn "%d" (identity 42)
printfn "%s" (identity "hello")
printfn "%A" (pair 1 "two")
```

### 3.2.4 ข้อจำกัดของ Type Inference

```fsharp
// บางครั้งต้องช่วย compiler ด้วย type annotation

// กรณี 1: มีหลาย overload
let parseIntExplicit (s: string) : int = System.Int32.Parse(s)

// กรณี 2: Generic ที่ต้องการ constraint
let printAny (x: 'a) = printfn "%A" x  // ต้องระบุ 'a

// กรณี 3: Flexible types
let addToList (lst: 'a list) (item: 'a) = item :: lst

// กรณี 4: เมื่อมี ambiguity ระหว่าง int และ float
let divide (a: float) (b: float) = a / b   // ต้องระบุ float
// let divide a b = a / b  // Error: ambiguous (int? float?)
```

---

## 3.3 Numeric Literals

### 3.3.1 Integer Literals

```fsharp
// Decimal (ปกติ)
let dec = 1234567

// Hexadecimal (0x prefix)
let hex = 0xFF        // 255
let hex2 = 0x1A2B3C   // 1713980

// Octal (0o prefix)
let oct = 0o777       // 511

// Binary (0b prefix)
let bin = 0b1010      // 10
let bin2 = 0b11111111 // 255

// Underscore separators (readable)
let million = 1_000_000
let hexFormatted = 0xFF_AB_CD
let binFormatted = 0b1111_0000

printfn "Hex 0xFF = %d" hex
printfn "Oct 0o777 = %d" oct
printfn "Bin 0b1010 = %d" bin
printfn "Million: %d" million
```

### 3.3.2 Type Suffixes

```fsharp
// Suffixes กำหนด type ของ literal
let intLit = 42          // int
let int8Lit = 42y        // int8 (sbyte)
let int16Lit = 42s       // int16
let int32Lit = 42l       // int32 (same as int, l is optional)
let int64Lit = 42L       // int64
let uint8Lit = 42uy      // uint8 (byte)
let uint16Lit = 42us     // uint16
let uint32Lit = 42u      // uint32
let uint64Lit = 42UL     // uint64
let float32Lit = 42.0f   // float32 (single)
let float64Lit = 42.0    // float (double)
let decimalLit = 42.0M   // decimal
let nativeintLit = 42n   // nativeint
let unativeintLit = 42un // unativeint

printfn "Types of literals:"
printfn "  int: %s" (intLit.GetType().Name)
printfn "  int64: %s" (int64Lit.GetType().Name)
printfn "  float32: %s" (float32Lit.GetType().Name)
printfn "  decimal: %s" (decimalLit.GetType().Name)
```

---

## 3.4 Type Conversion

### 3.4.1 Explicit Conversions

```fsharp
// F# ไม่มี implicit type conversion - ต้องแปลงชัดเจน

// To int
let fromFloat = int 3.9     // 3 (truncates, not rounds)
let fromStr = int "42"
let fromBool = int true      // 1
let fromChar = int 'A'       // 65

printfn "float to int: %d" fromFloat
printfn "string to int: %d" fromStr
printfn "bool to int: %d" fromBool
printfn "char to int: %d" fromChar

// To float
let intToFloat = float 42     // 42.0
let strToFloat = float "3.14"
let decToFloat = float 99.99M

printfn "int to float: %f" intToFloat
printfn "string to float: %f" strToFloat

// To string
let intStr = string 42        // "42"
let floatStr = string 3.14    // "3.14"
let boolStr = string true     // "True"
let charStr = string 'A'      // "A"

printfn "int to string: %s" intStr

// To bool
let intToBool = bool 1   // Hmm, ใน F# ไม่ work อย่างนี้
// ใช้ comparison แทน:
let isNonZero n = n <> 0

// To decimal
let intToDec = decimal 42     // 42M
let floatToDec = decimal 3.14 // 3.14M (approximation)

printfn "int to decimal: %M" intToDec
```

### 3.4.2 Conversion Functions

```fsharp
open System

// ใช้ Convert class
let s1 = Convert.ToString(42)
let i1 = Convert.ToInt32("42")
let f1 = Convert.ToDouble("3.14")
let b1 = Convert.ToBoolean(1)

printfn "Convert.ToString: %s" s1
printfn "Convert.ToInt32: %d" i1
printfn "Convert.ToDouble: %f" f1
printfn "Convert.ToBoolean: %b" b1

// Parse methods
let parsed1 = Int32.Parse("42")
let parsed2 = Double.Parse("3.14")
let parsed3 = Boolean.Parse("true")

// TryParse (safe, no exception)
let success1, val1 = Int32.TryParse("42")
let success2, val2 = Int32.TryParse("not a number")

printfn "TryParse '42': %b, %d" success1 val1
printfn "TryParse 'not a number': %b, %d" success2 val2

// F# way ด้วย option
let tryParseInt (s: string) =
    match Int32.TryParse(s) with
    | true, v -> Some v
    | false, _ -> None

let result1 = tryParseInt "100"
let result2 = tryParseInt "abc"

printfn "tryParseInt '100': %A" result1
printfn "tryParseInt 'abc': %A" result2
```

### 3.4.3 Numeric Conversions

```fsharp
// Widening conversions (ปลอดภัย)
let byteVal: byte = 200uy
let intFromByte = int byteVal     // 200
let int64FromInt = int64 42       // 42L
let floatFromInt = float 42       // 42.0

// Narrowing conversions (อาจสูญเสียข้อมูล)
let largeInt64: int64 = 9999999999L
let intFromLarge = int largeInt64  // อาจ overflow!

printfn "Large int64: %d" largeInt64
printfn "int from large: %d" intFromLarge  // overflow!

// Checked conversions (throw exception on overflow)
try
    let checkedResult = Checked.int largeInt64
    printfn "Checked result: %d" checkedResult
with
| :? OverflowException as ex ->
    printfn "Overflow! %s" ex.Message
```

---

## 3.5 bigint (Arbitrary Precision Integer)

```fsharp
open System.Numerics

// bigint - จำนวนเต็มที่ใหญ่แค่ไหนก็ได้
let big1: bigint = 99999999999999999999999999I  // suffix I
let big2 = bigint 1000000000000L

// Operations
let bigSum = big1 + big2
let bigProduct = big1 * 2I
let bigPower = BigInteger.Pow(2I, 100)  // 2^100

printfn "big1: %A" big1
printfn "bigSum: %A" bigSum
printfn "2^100 = %A" bigPower

// Factorial example (overflow with int64 after n=20)
let rec factorial n =
    if n <= 1I then 1I
    else n * factorial (n - 1I)

let fac20 = factorial 20I
let fac100 = factorial 100I

printfn "20! = %A" fac20
printfn "100! = %A" fac100
```

---

## 3.6 Checked Arithmetic

```fsharp
open Checked

// Checked operations ตรวจสอบ overflow
try
    let x = Int32.MaxValue
    let overflow = Checked.(+) x 1  // throw OverflowException
    printfn "Should not reach: %d" overflow
with
| :? OverflowException ->
    printfn "Caught overflow!"

// Unchecked (ค่าเริ่มต้น) - เงียบๆ overflow
let uncheckedResult = System.Int32.MaxValue + 1  // -2147483648 (wraps around)
printfn "Unchecked overflow: %d" uncheckedResult

// Explicit unchecked
let unchecked2 = Unchecked.defaultof<int>  // default value for int = 0
printfn "Default int: %d" unchecked2
```

---

## 3.7 Infinity และ NaN

```fsharp
// Infinity
let posInf = infinity           // +∞
let negInf = -infinity          // -∞
let posInf2 = 1.0 / 0.0         // +∞ (ไม่ throw exception!)
let negInf2 = -1.0 / 0.0        // -∞

printfn "Positive infinity: %f" posInf
printfn "Negative infinity: %f" negInf
printfn "Is infinity: %b" (Double.IsInfinity(posInf))
printfn "Is positive infinity: %b" (Double.IsPositiveInfinity(posInf))
printfn "Is negative infinity: %b" (Double.IsNegativeInfinity(negInf))

// Arithmetic with infinity
printfn "inf + 1 = %f" (posInf + 1.0)   // inf
printfn "inf * 2 = %f" (posInf * 2.0)   // inf
printfn "inf - inf = %f" (posInf - posInf)  // NaN!
printfn "1/inf = %f" (1.0 / posInf)     // 0.0
printfn "inf > 1e308 = %b" (posInf > 1e308)  // true

// NaN (Not a Number)
let nan = nan                   // NaN
let nan2 = 0.0 / 0.0            // NaN
let nan3 = posInf - posInf      // NaN
let nan4 = sqrt (-1.0)          // NaN

printfn "NaN: %f" nan
printfn "Is NaN: %b" (Double.IsNaN(nan))
printfn "NaN = NaN: %b" (nan = nan)    // false! NaN ไม่เท่ากับตัวเอง
printfn "NaN <> NaN: %b" (nan <> nan)  // true

// ตรวจสอบ NaN ต้องใช้ IsNaN
let safeCalculate x =
    let result = sqrt x
    if Double.IsNaN(result) then
        printfn "Cannot take sqrt of negative number"
        0.0
    else
        result

printfn "sqrt(4): %f" (safeCalculate 4.0)
printfn "sqrt(-1): %f" (safeCalculate -1.0)
```

---

## 3.8 Null ใน F#

```fsharp
// F# ไม่ใช้ null สำหรับ F# types โดยปกติ
// แต่ต้องจัดการ null เมื่อ interop กับ .NET

// F# record/union types ไม่สามารถเป็น null ได้
type Person = { Name: string; Age: int }
// let p: Person = null  // Compile error!

// แต่ .NET types สามารถเป็น null ได้
let nullableString: string = null     // WARNING ใน F# กับ Nullable types
let notNull: string = "hello"

// ตรวจสอบ null
let isNull (s: string) = s = null
let isNotNull (s: string) = s <> null

// ใช้ Option แทน null (recommended)
type SafePerson = {
    Name: string
    Age: int
    Email: string option  // อาจไม่มี email
}

let person1 = { Name = "Alice"; Age = 30; Email = Some "alice@example.com" }
let person2 = { Name = "Bob"; Age = 25; Email = None }

let printEmail p =
    match p.Email with
    | Some email -> printfn "%s's email: %s" p.Name email
    | None -> printfn "%s has no email" p.Name

printEmail person1
printEmail person2

// Null handling สำหรับ .NET interop
open System.IO

let readFile path =
    let content = File.ReadAllText(path)  // อาจ return null หรือ throw
    if content = null then
        None
    else
        Some content

// Option.ofObj - แปลง null เป็น None, non-null เป็น Some
let safeString (s: string) : string option =
    Option.ofObj s

let result1 = safeString "hello"     // Some "hello"
let result2 = safeString null        // None

printfn "safeString result: %A" result1
printfn "null string: %A" result2

// Option.toObj - แปลง Some x เป็น x, None เป็น null
let toNullable (opt: string option) : string =
    Option.defaultValue null opt
    // หรือ Option.toObj opt  (ใน F# 5+)
```

---

## 3.9 sizeof และ Type Information

```fsharp
// sizeof<T> - ขนาดของ type ใน bytes
printfn "sizeof<bool> = %d" sizeof<bool>     // 1
printfn "sizeof<byte> = %d" sizeof<byte>     // 1
printfn "sizeof<int16> = %d" sizeof<int16>   // 2
printfn "sizeof<int> = %d" sizeof<int>       // 4
printfn "sizeof<int64> = %d" sizeof<int64>   // 8
printfn "sizeof<float32> = %d" sizeof<float32> // 4
printfn "sizeof<float> = %d" sizeof<float>   // 8
printfn "sizeof<decimal> = %d" sizeof<decimal> // 16
printfn "sizeof<char> = %d" sizeof<char>     // 2 (UTF-16)

// typedefof<T> - ดู type object
let intType = typedefof<int>
let strType = typedefof<string>
printfn "int type: %s" intType.FullName
printfn "string type: %s" strType.FullName

// typeof<T> - ดู Type
let intTypeOf = typeof<int>
printfn "int is value type: %b" intTypeOf.IsValueType
printfn "string is value type: %b" (typeof<string>.IsValueType)
```

---

## 3.10 Type Aliases

```fsharp
// type aliases ด้วย type keyword
type Age = int
type Name = string
type Price = decimal

// ใช้งาน
let myAge: Age = 30
let myName: Name = "Alice"
let itemPrice: Price = 99.99M

// Type aliases ไม่ได้สร้าง distinct type (เป็นแค่ shorthand)
let ageValue: int = myAge  // OK เพราะ Age = int

// สำหรับ distinct types ต้องใช้ single-case DU
type CustomerId = CustomerId of int
type OrderId = OrderId of int

let custId = CustomerId 42
let orderId = OrderId 42

// custId = orderId  // Error! Type mismatch
```

---

## 3.11 ตัวอย่างการใช้ Types จริง

### 3.11.1 การเงินและสกุลเงิน

```fsharp
// ใช้ decimal สำหรับการเงิน (ไม่ใช้ float!)
type Currency = THB | USD | EUR

type Money = {
    Amount: decimal
    Currency: Currency
}

let createMoney amount currency = { Amount = amount; Currency = currency }

let addMoney m1 m2 =
    if m1.Currency = m2.Currency then
        Some { Amount = m1.Amount + m2.Amount; Currency = m1.Currency }
    else
        None  // ไม่สามารถบวกต่างสกุลได้

let thb = createMoney 1000.00M THB
let usd = createMoney 28.50M USD

match addMoney thb (createMoney 500.00M THB) with
| Some total -> printfn "Total: %M %A" total.Amount total.Currency
| None -> printfn "Currency mismatch!"

// ค่าไม่ถูกต้องจาก float arithmetic
printfn "Float: %f" (0.1 + 0.2)      // 0.300000000000000004
printfn "Decimal: %M" (0.1M + 0.2M)  // 0.3 (exact)
```

### 3.11.2 Scientific Computing

```fsharp
open System

// Constants
let e = Math.E           // 2.718281828459045
let pi = Math.PI         // 3.141592653589793

// Math functions
let sqrtOf2 = sqrt 2.0
let log10_100 = log10 100.0
let logNatural = log (Math.E)
let sin90 = sin (Math.PI / 2.0)
let cos0 = cos 0.0

printfn "e = %.15f" e
printfn "pi = %.15f" pi
printfn "sqrt(2) = %.10f" sqrtOf2
printfn "log10(100) = %f" log10_100
printfn "ln(e) = %f" logNatural
printfn "sin(90°) = %f" sin90
printfn "cos(0°) = %f" cos0

// Geometric calculations
let circleArea r = pi * r * r
let sphereVolume r = (4.0/3.0) * pi * r * r * r
let cylinderVolume r h = pi * r * r * h

printfn "\nGeometry:"
printfn "Circle area (r=5): %.4f" (circleArea 5.0)
printfn "Sphere volume (r=3): %.4f" (sphereVolume 3.0)
printfn "Cylinder volume (r=2, h=10): %.4f" (cylinderVolume 2.0 10.0)
```

### 3.11.3 Working with Bytes

```fsharp
// Byte manipulation
let data: byte[] = [| 0uy; 128uy; 255uy; 42uy |]

printfn "Byte array: %A" data

// Convert bytes to/from int
let byteToInt (b: byte) = int b
let intToByte (i: int) = byte i

// Hex representation
let toHex (b: byte) = sprintf "%02X" b
let hexString = data |> Array.map toHex |> String.concat " "
printfn "Hex: %s" hexString

// Bitwise operations
let a = 0b11001010uy  // 202
let b = 0b10110011uy  // 179

printfn "a = %d (0b%s)" a (System.Convert.ToString(int a, 2).PadLeft(8, '0'))
printfn "b = %d (0b%s)" b (System.Convert.ToString(int b, 2).PadLeft(8, '0'))
printfn "a AND b = %d" (a &&& b)
printfn "a OR b = %d" (a ||| b)
printfn "a XOR b = %d" (a ^^^ b)
printfn "NOT a = %d" (~~~a)
printfn "a << 2 = %d" (a <<< 2)
printfn "a >> 1 = %d" (a >>> 1)
```

---

## สรุป Part 3

ในบทนี้เราได้เรียนรู้:
- ✅ Primitive types: int, float, decimal, string, bool, char, byte, unit
- ✅ Integer variants: int8, int16, int32, int64, uint variants
- ✅ Floating point: float32 vs float64
- ✅ bigint สำหรับตัวเลขขนาดใหญ่
- ✅ Type inference - compiler ช่วย infer types
- ✅ Numeric literals: hex, octal, binary, type suffixes
- ✅ Type conversion functions
- ✅ Checked arithmetic ป้องกัน overflow
- ✅ Infinity และ NaN
- ✅ Null ใน F# และทำไมต้องหลีกเลี่ยง
- ✅ sizeof, type information

**ใน Part 4** เราจะเรียนรู้เกี่ยวกับ Functions ใน F# อย่างลึกซึ้ง!
