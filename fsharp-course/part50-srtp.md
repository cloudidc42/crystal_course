# Part 50 - Statically Resolved Type Parameters (SRTP)

## บทนำ

Statically Resolved Type Parameters (SRTP) เป็น feature พิเศษของ F# ที่ช่วยให้เขียน generic code ที่ทำงานกับ "duck typing" ได้ โดย compiler จะตรวจสอบ constraints ในเวลา compile แทนที่จะใช้ polymorphism แบบ runtime

## 1. What are SRTPs

```fsharp
// SRTP ต่างจาก normal type parameters ตรงที่:
// 1. ใช้ ^ แทน '
// 2. ต้องอยู่ใน inline function
// 3. Compiler แทนที่ type parameter ในเวลา compile
// 4. ไม่มี runtime overhead จาก virtual dispatch

// ตัวอย่างง่าย: Normal generic (runtime polymorphism)
let normalAdd<'T> (x: 'T) (y: 'T) : 'T = 
    failwith "ทำงานไม่ได้เพราะไม่รู้ว่า 'T มี + operator"

// SRTP: Statically resolved (compile-time)
let inline srtpAdd (x: ^T) (y: ^T) : ^T =
    x + y  // + operator ถูก resolve ในเวลา compile

// ใช้งานกับหลาย types
let intResult = srtpAdd 10 20          // int + int
let floatResult = srtpAdd 1.5 2.5      // float + float
let strResult = srtpAdd "Hello" " World"  // string concatenation

printfn "int: %d" intResult
printfn "float: %.1f" floatResult
printfn "string: %s" strResult
```

## 2. ^ Syntax for Type Variables

```fsharp
// ^ ใช้แทน ' สำหรับ SRTP type parameters
// ต้องใช้ร่วมกับ inline keyword

// Single SRTP
let inline double (x: ^T) = x + x

// Multiple SRTPs
let inline multiply (x: ^T) (y: ^U) = x  // simplified

// ^ ใน where clause
let inline negate (x: ^T when ^T : (static member (~-) : ^T -> ^T)) =
    -x

// ทดสอบ
printfn "double 5: %d" (double 5)
printfn "double 3.14: %.2f" (double 3.14)
printfn "double 'A': %c" (double 'A')  // สำหรับ char concatenation จะไม่ทำงาน
```

## 3. inline Requirement

```fsharp
// SRTP ต้องใช้กับ inline function เสมอ
// เพราะ compiler ต้อง "expand" function ณ จุดที่ถูกเรียก

// ✅ ถูกต้อง: inline
let inline addTwo (x: ^T) (y: ^T) : ^T = x + y

// ❌ ไม่ถูกต้อง: ไม่มี inline
// let addTwoNoInline (x: ^T) (y: ^T) : ^T = x + y  // compile error!

// เหตุผล: โดยไม่มี inline, compiler ไม่รู้ว่า operator + คืออะไร
// ใน IL (intermediate language) จะไม่มี operator + แบบ generic

// inline function ถูก "inlined" ณ แต่ละ call site
let sum1 = addTwo 1 2       // แทนที่ด้วย 1 + 2
let sum2 = addTwo 1.0 2.0   // แทนที่ด้วย 1.0 + 2.0
let sum3 = addTwo "a" "b"   // แทนที่ด้วย "a" + "b"

printfn "%d, %.1f, %s" sum1 sum2 sum3
```

## 4. Structural Typing กับ SRTPs

```fsharp
// SRTP ทำงานแบบ structural typing (duck typing)
// ไม่ต้องมี interface หรือ inheritance

// ต้องการ member ชื่อ Length
let inline getLength (x: ^T when ^T : (member Length : int)) =
    (^T : (member Length : int) x)

// ทดสอบกับ types ต่างๆ ที่มี Length
let strLen = getLength "Hello"
let arrLen = getLength [|1;2;3;4;5|]
// listLen ไม่ได้เพราะ list ใช้ .Length property ในแบบต่าง

printfn "String length: %d" strLen
printfn "Array length: %d" arrLen
```

```fsharp
// Duck typing ขั้นสูง: ต้องการ method เฉพาะ
let inline callToString (x: ^T when ^T : (member ToString : unit -> string)) =
    (^T : (member ToString : unit -> string) x)

// ทุก .NET object มี ToString()
let s1 = callToString 42
let s2 = callToString 3.14
let s3 = callToString true
let s4 = callToString "already a string"

printfn "int.ToString(): %s" s1
printfn "float.ToString(): %s" s2
printfn "bool.ToString(): %s" s3
printfn "string.ToString(): %s" s4
```

## 5. Numeric Abstractions

```fsharp
// สร้าง abstraction สำหรับ numeric operations

// Generic zero
let inline zero<'T when 'T : (static member Zero : 'T)> () : 'T =
    LanguagePrimitives.GenericZero<'T>

// Generic one
let inline one<'T when 'T : (static member One : 'T)> () : 'T =
    LanguagePrimitives.GenericOne<'T>

// Generic sum
let inline genericSum (items: ^T list) : ^T =
    items |> List.fold (+) LanguagePrimitives.GenericZero

// ทดสอบ
printfn "zero int: %d" (zero<int> ())
printfn "zero float: %.1f" (zero<float> ())
printfn "one int: %d" (one<int> ())
printfn "sum ints: %d" (genericSum [1; 2; 3; 4; 5])
printfn "sum floats: %.1f" (genericSum [1.5; 2.5; 3.0])
```

```fsharp
// Numeric algorithms ที่ทำงานกับทุก numeric types
let inline factorial (n: ^T) : ^T =
    let zero = LanguagePrimitives.GenericZero<^T>
    let one = LanguagePrimitives.GenericOne<^T>
    let mutable result = one
    let mutable i = one
    while i <= n do
        result <- result * i
        i <- i + one
    result

printfn "5! = %d" (factorial 5)
printfn "5! (float) = %.0f" (factorial 5.0)
printfn "10! (int64) = %d" (factorial 10L)
```

## 6. Generic Math Operations

```fsharp
// สร้าง generic math library
module GenericMath =
    
    let inline abs (x: ^T when (^T : (static member Abs : ^T -> ^T) or ^T : comparison)) : ^T =
        if x < LanguagePrimitives.GenericZero then
            LanguagePrimitives.GenericZero - x
        else x
    
    let inline min (a: ^T) (b: ^T) : ^T = if a < b then a else b
    let inline max (a: ^T) (b: ^T) : ^T = if a > b then a else b
    
    let inline clamp (low: ^T) (high: ^T) (value: ^T) : ^T =
        min high (max low value)
    
    let inline lerp (a: ^T) (b: ^T) (t: ^T) : ^T =
        a + t * (b - a)
    
    let inline sum (items: ^T seq) : ^T =
        Seq.fold (+) LanguagePrimitives.GenericZero items
    
    let inline average (items: ^T list) : ^T when ^T : (static member op_Division : ^T * ^T -> ^T) =
        let s = sum items
        let count = float items.Length
        // ต้องแปลง count เป็น ^T
        s  // Simplified

// ใช้งาน
printfn "abs(-5) = %d" (GenericMath.abs -5)
printfn "abs(-3.14) = %.2f" (GenericMath.abs -3.14)
printfn "min(3, 7) = %d" (GenericMath.min 3 7)
printfn "max(3, 7) = %d" (GenericMath.max 3 7)
printfn "clamp(0, 100, 150) = %d" (GenericMath.clamp 0 100 150)
printfn "lerp(0.0, 10.0, 0.5) = %.1f" (GenericMath.lerp 0.0 10.0 0.5)
```

## 7. Custom Type กับ (+), (-), (*)

```fsharp
// สร้าง custom type ที่ทำงานกับ SRTP operators

[<Struct>]
type Vec2 = {
    X: float
    Y: float
} with
    // สร้าง operators เพื่อให้ทำงานกับ SRTP
    static member (+) (a: Vec2, b: Vec2) = { X = a.X + b.X; Y = a.Y + b.Y }
    static member (-) (a: Vec2, b: Vec2) = { X = a.X - b.X; Y = a.Y - b.Y }
    static member (*) (a: Vec2, s: float) = { X = a.X * s; Y = a.Y * s }
    static member (*) (s: float, a: Vec2) = { X = a.X * s; Y = a.Y * s }
    static member Zero = { X = 0.0; Y = 0.0 }
    static member One = { X = 1.0; Y = 1.0 }
    member this.Length = sqrt (this.X * this.X + this.Y * this.Y)
    override this.ToString() = sprintf "Vec2(%.2f, %.2f)" this.X this.Y

// ทดสอบ
let v1 = { X = 3.0; Y = 4.0 }
let v2 = { X = 1.0; Y = 2.0 }

let sum = v1 + v2
let diff = v1 - v2
let scaled = v1 * 2.0

printfn "v1 + v2 = %A" sum
printfn "v1 - v2 = %A" diff
printfn "v1 * 2 = %A" scaled
printfn "v1.Length = %.2f" v1.Length
```

```fsharp
// ใช้ Vec2 กับ generic function
let inline vectorSum (items: ^T list) : ^T =
    items |> List.fold (+) LanguagePrimitives.GenericZero

let vectors = [
    { X = 1.0; Y = 0.0 }
    { X = 0.0; Y = 1.0 }
    { X = 2.0; Y = 3.0 }
]

let totalVec = vectorSum vectors
printfn "Total vector: %A" totalVec
```

```fsharp
// Matrix type
[<Struct>]
type Mat2x2 = {
    A11: float; A12: float
    A21: float; A22: float
} with
    static member (+) (m1: Mat2x2, m2: Mat2x2) = {
        A11 = m1.A11 + m2.A11; A12 = m1.A12 + m2.A12
        A21 = m1.A21 + m2.A21; A22 = m1.A22 + m2.A22
    }
    static member (*) (m1: Mat2x2, m2: Mat2x2) = {
        A11 = m1.A11 * m2.A11 + m1.A12 * m2.A21
        A12 = m1.A11 * m2.A12 + m1.A12 * m2.A22
        A21 = m1.A21 * m2.A11 + m1.A22 * m2.A21
        A22 = m1.A21 * m2.A12 + m1.A22 * m2.A22
    }
    static member Zero = { A11 = 0.0; A12 = 0.0; A21 = 0.0; A22 = 0.0 }
    static member Identity = { A11 = 1.0; A12 = 0.0; A21 = 0.0; A22 = 1.0 }

let m1 = { A11 = 1.0; A12 = 2.0; A21 = 3.0; A22 = 4.0 }
let m2 = { A11 = 5.0; A12 = 6.0; A21 = 7.0; A22 = 8.0 }

printfn "m1 + m2 = %A" (m1 + m2)
printfn "m1 * m2 = %A" (m1 * m2)
```

## 8. PrintfFormat และ printf

```fsharp
// SRTP ใช้ใน printf implementation ของ F#
// printf ใน F# เป็น type-safe โดยใช้ SRTP

// เข้าใจว่า printf ทำงานอย่างไร
let myPrintf (fmt: Printf.StringFormat<'T>) : 'T =
    Printf.sprintf fmt

let result1 = myPrintf "%d + %d = %d" 1 2 3
let result2 = myPrintf "Hello, %s!" "World"
let result3 = myPrintf "Pi = %.4f" System.Math.PI

printfn "%s" result1
printfn "%s" result2
printfn "%s" result3
```

```fsharp
// สร้าง type-safe log function
let inline logInfo (fmt: Printf.TextWriterFormat<'T>) : 'T =
    printf "[INFO] "
    printfn fmt

let inline logError (fmt: Printf.TextWriterFormat<'T>) : 'T =
    printf "[ERROR] "
    printfn fmt

logInfo "Application started"
logInfo "Processing %d items" 42
logError "Failed with code %d: %s" 404 "Not Found"
```

## 9. Duck Typing Patterns

```fsharp
// Pattern: ต้องการ specific methods โดยไม่มี interface

// ต้องการ method Parse
let inline parse (s: string) : ^T =
    (^T : (static member Parse : string -> ^T) s)

// ทดสอบ
let intParsed: int = parse "42"
let floatParsed: float = parse "3.14"
let boolParsed: bool = parse "true"

printfn "int: %d" intParsed
printfn "float: %.2f" floatParsed
printfn "bool: %b" boolParsed
```

```fsharp
// Pattern: ต้องการ static factory method
let inline create<'T when 'T : (static member Create : unit -> 'T)> () : 'T =
    (^T : (static member Create : unit -> 'T) ())

// ต้องการ property
let inline isEmpty<'T when 'T : (member IsEmpty : bool)> (x: ^T) : bool =
    (^T : (member IsEmpty : bool) x)

// ทดสอบ
let emptyList: int list = []
// printfn "list isEmpty: %b" (isEmpty emptyList)  // List ไม่มี .IsEmpty property โดยตรง

// Custom type ที่มี IsEmpty
type Stack<'T> = Stack of 'T list with
    member this.IsEmpty = match this with Stack [] -> true | _ -> false
    member this.Push(v) = match this with Stack lst -> Stack (v :: lst)
    member this.Pop() = match this with Stack (h :: t) -> Some h, Stack t | _ -> None, this

let stack = Stack []
let pushed = stack.Push(42)
printfn "empty stack isEmpty: %b" (isEmpty stack)
printfn "pushed stack isEmpty: %b" (isEmpty pushed)
```

## 10. Practical Generic Algorithms

```fsharp
// Practical algorithms ที่ใช้ SRTP

// Generic power function
let inline pow (base': ^T) (exp: int) : ^T =
    let one = LanguagePrimitives.GenericOne<^T>
    let mutable result = one
    for _ in 1..exp do
        result <- result * base'
    result

// ทดสอบ
printfn "2^10 = %d" (pow 2 10)
printfn "2.0^10 = %.0f" (pow 2.0 10)
printfn "3^3 = %d" (pow 3 3)
```

```fsharp
// Generic statistics
module Statistics =
    
    let inline mean (items: ^T list) : ^T =
        let sum = items |> List.fold (+) LanguagePrimitives.GenericZero
        // ต้องหาร - simplified
        sum
    
    let inline sum (items: ^T seq) : ^T =
        Seq.fold (+) LanguagePrimitives.GenericZero items
    
    let inline product (items: ^T seq) : ^T =
        Seq.fold (*) LanguagePrimitives.GenericOne items

// ใช้งาน
let intData = [1; 2; 3; 4; 5]
let floatData = [1.5; 2.5; 3.0; 4.5; 5.5]

printfn "int sum: %d" (Statistics.sum intData)
printfn "float sum: %.1f" (Statistics.sum floatData)
printfn "int product: %d" (Statistics.product [1..5])
printfn "float product: %.1f" (Statistics.product [1.0; 2.0; 3.0])
```

```fsharp
// Generic sorting predicate
let inline sortByKey (getKey: ^T -> ^K) (items: ^T list) =
    items |> List.sortBy getKey

// Generic max/min ด้วย SRTP
let inline genericMax (items: ^T seq) : ^T option when ^T : comparison =
    items |> Seq.tryReduce (fun a b -> if a > b then a else b)

let inline genericMin (items: ^T seq) : ^T option when ^T : comparison =
    items |> Seq.tryReduce (fun a b -> if a < b then a else b)

printfn "max int: %A" (genericMax [3; 1; 4; 1; 5; 9; 2; 6])
printfn "min float: %A" (genericMin [3.1; 1.4; 2.7; 0.5])
printfn "max string: %A" (genericMax ["banana"; "apple"; "cherry"])
```

## 11. SRTP vs Interfaces

```fsharp
(*
เปรียบเทียบ SRTP กับ Interface:

| คุณสมบัติ       | SRTP                    | Interface               |
|----------------|-------------------------|-------------------------|
| Type safety    | Compile-time            | Compile-time            |
| Performance    | Zero overhead (inlined) | Virtual dispatch        |
| Flexibility    | Structural typing       | Nominal typing          |
| Existing types | ทำงานได้               | ต้อง implement          |
| Constraint     | ใน where clause         | : IInterface            |
| Debugging      | ยากกว่า                | ง่ายกว่า                |
| IDE support    | Limited                 | Full                    |
*)

// Interface approach
type IAddable<'T> =
    abstract member Add: 'T -> 'T

type AddableInt(value: int) =
    interface IAddable<AddableInt> with
        member _.Add(other) = AddableInt(value + match other with :? AddableInt as a -> a.Value | _ -> 0)
    member _.Value = value

// SRTP approach - ทำงานกับ existing types โดยตรง
let inline addSrtp (a: ^T) (b: ^T) : ^T when ^T : (static member (+) : ^T * ^T -> ^T) =
    a + b

// SRTP ทำงานกับ int, float, string โดยไม่ต้อง wrap
printfn "SRTP int: %d" (addSrtp 1 2)
printfn "SRTP float: %.1f" (addSrtp 1.5 2.5)
printfn "SRTP string: %s" (addSrtp "Hello" " World")
```

## 12. SRTP Member Constraints

```fsharp
// ชนิดของ constraints ที่ SRTP รองรับ

// 1. Static member constraint
let inline hasStaticMember (x: ^T when ^T : (static member Zero : ^T)) = 
    (^T : (static member Zero : ^T))

// 2. Instance member constraint
let inline hasInstanceMember (x: ^T when ^T : (member Length : int)) =
    (^T : (member Length : int) x)

// 3. Method constraint
let inline hasMethod (x: ^T when ^T : (member ToString : unit -> string)) =
    (^T : (member ToString : unit -> string) x)

// 4. Operator constraint
let inline addOp (a: ^T) (b: ^T) when ^T : (static member (+) : ^T * ^T -> ^T) =
    a + b

// 5. Comparison constraint
let inline compareValues (a: ^T) (b: ^T) when ^T : comparison =
    compare a b

// 6. Equality constraint
let inline areEqual (a: ^T) (b: ^T) when ^T : equality =
    a = b

// 7. Null constraint
let inline acceptNull (x: ^T when ^T : null) =
    x = null

// ทดสอบ
printfn "int Zero: %d" (hasStaticMember 42 : int)
printfn "string Length: %d" (hasInstanceMember "Hello")
printfn "int ToString: %s" (hasMethod 42)
printfn "add: %d" (addOp 5 3)
printfn "compare: %d" (compareValues "abc" "abd")
printfn "equal: %b" (areEqual 42 42)
```

## 13. ตัวอย่างจริง: Generic Number Operations

```fsharp
// สร้าง number operations ที่ทำงานกับทุก numeric type

module Numbers =
    
    // Clamp: จำกัดค่าให้อยู่ในช่วง
    let inline clamp (minVal: ^T) (maxVal: ^T) (value: ^T) : ^T =
        if value < minVal then minVal
        elif value > maxVal then maxVal
        else value
    
    // Normalize: แปลงเป็น 0.0-1.0 range
    let inline normalize (min: float) (max: float) (value: float) : float =
        (value - min) / (max - min) |> clamp 0.0 1.0
    
    // Linear interpolation
    let inline lerp (start: ^T) (end': ^T) (t: float) =
        let startF = float (start |> unbox)  // simplified
        let endF = float (end' |> unbox)
        startF + t * (endF - startF)
    
    // Map value from one range to another
    let inline remap (inMin: float) (inMax: float) (outMin: float) (outMax: float) (value: float) =
        let t = normalize inMin inMax value
        outMin + t * (outMax - outMin)
    
    // Check if value is in range
    let inline inRange (min: ^T) (max: ^T) (value: ^T) : bool =
        value >= min && value <= max

// ใช้งาน
printfn "clamp(0, 10, 15) = %d" (Numbers.clamp 0 10 15)
printfn "clamp(0, 10, 5) = %d" (Numbers.clamp 0 10 5)
printfn "clamp(0, 10, -5) = %d" (Numbers.clamp 0 10 -5)
printfn "normalize(0, 100, 50) = %.2f" (Numbers.normalize 0.0 100.0 50.0)
printfn "remap(0, 100, 0, 1, 75) = %.2f" (Numbers.remap 0.0 100.0 0.0 1.0 75.0)
printfn "inRange(1, 10, 5) = %b" (Numbers.inRange 1 10 5)
printfn "inRange(1, 10, 15) = %b" (Numbers.inRange 1 10 15)
```

## 14. Limitations และ Gotchas

```fsharp
(*
ข้อจำกัดและข้อควรระวัง:

1. ต้องเป็น inline เสมอ
   - ไม่สามารถใช้ SRTP กับ non-inline functions
   - ไม่สามารถเก็บ function ใน data structure

2. ไม่ทำงานกับ virtual dispatch
   - SRTP ทำงาน ณ compile-time เท่านั้น
   - ไม่รองรับ polymorphism แบบ runtime

3. ข้อจำกัดใน recursion
   - Recursive inline functions มีข้อจำกัด
   - ต้องใช้ [<TailCall>] หรือ explicit recursion

4. Error messages ซับซ้อน
   - Compile error จาก SRTP อาจอ่านยาก

5. IL ขนาดใหญ่
   - inline function สร้าง duplicate code
   - Binary ขนาดใหญ่กว่า

6. ไม่ทำงานกับ F# lists ที่มี type class
*)

// ตัวอย่างข้อจำกัด
// ❌ ไม่ได้: เก็บ SRTP function ใน list
// let functions = [addSrtp; addSrtp]  // Error!

// ✅ แต่สามารถใช้ interface แทน
type IAddOp<'T> = interface
    abstract Add: 'T -> 'T -> 'T
end

// ❌ ไม่ได้: SRTP กับ first-class functions แบบ heterogeneous
// let heterogeneous: (int -> int) = addSrtp  // Works for specific type

// ✅ SRTP ทำงานกับ specific type
let addIntSpecific = addOp<int>  // Not valid SRTP syntax, but concept
let intAdd = addSrtp 5 3  // Works
```

```fsharp
// ปัญหา: SRTP ใน recursive functions
// ต้องใช้ [<TailCall>] หรือ accumulator pattern

let inline sumRec (items: ^T list) : ^T =
    let rec loop acc = function
        | [] -> acc
        | h :: t -> loop (acc + h) t
    loop LanguagePrimitives.GenericZero items

printfn "sumRec [1..5] = %d" (sumRec [1..5])
printfn "sumRec [1.0..5.0] = %.1f" (sumRec [1.0; 2.0; 3.0; 4.0; 5.0])
```

## 15. Advanced SRTP Patterns

```fsharp
// Pattern: ตรวจสอบ capabilities
let inline supportsArithmetic< ^T
    when ^T : (static member (+) : ^T * ^T -> ^T)
    and ^T : (static member (-) : ^T * ^T -> ^T)
    and ^T : (static member (*) : ^T * ^T -> ^T)
    and ^T : (static member (/) : ^T * ^T -> ^T)
    and ^T : (static member Zero : ^T)
    and ^T : (static member One : ^T)> (v: ^T) =
    
    let z = LanguagePrimitives.GenericZero<^T>
    let o = LanguagePrimitives.GenericOne<^T>
    printfn "Type supports arithmetic: zero=%A, one=%A" z o

supportsArithmetic 42
supportsArithmetic 3.14
```

```fsharp
// Pattern: เลือก implementation ตาม type
let inline processNumber (x: ^T) =
    // F# เลือก implementation ที่ถูกต้องใน compile-time
    printfn "Processing: %A (type: %s)" x (typeof<^T>.Name)

processNumber 42
processNumber 3.14
processNumber 42L
```

```fsharp
// SRTP สำหรับ serialization (concept)
let inline serialize (x: ^T when ^T : (member ToString: unit -> string)) : string =
    (^T : (member ToString: unit -> string) x)

let inline deserialize< ^T when ^T : (static member Parse: string -> ^T)> (s: string) : ^T =
    (^T : (static member Parse: string -> ^T) s)

// ใช้งาน
let serialized = serialize 42
printfn "Serialized: %s" serialized

let deserialized: int = deserialize "42"
printfn "Deserialized: %d" deserialized
```

## 16. SRTP ใน Library Design

```fsharp
// ออกแบบ library ที่ใช้ SRTP
module MathLib =
    
    // Vector operations (generic)
    let inline dot (v1: ^V) (v2: ^V) : ^V = v1 * v2  // simplified
    
    // Distance ระหว่าง 2 points
    let inline distance (a: ^T) (b: ^T) : float
        when ^T : (static member (-) : ^T * ^T -> ^T) =
        let diff = a - b
        // ต้องการ float conversion
        0.0  // simplified
    
    // Generic accumulator
    let inline accumulate (items: ^T seq) : ^T =
        Seq.fold (+) LanguagePrimitives.GenericZero items

// ตัวอย่าง practical use
let nums = seq { 1..100 }
let total = MathLib.accumulate nums
printfn "Sum 1..100 = %d" total

let floats = seq { 0.1..0.1..1.0 }
let ftotal = MathLib.accumulate floats
printfn "Sum 0.1..1.0 = %.1f" ftotal
```

## สรุป

```fsharp
(*
SRTP (Statically Resolved Type Parameters) - สรุป:

Syntax:
- ^T แทน 'T สำหรับ SRTP type variable
- ต้องใช้กับ inline function เสมอ
- where clause สำหรับ constraints

Constraints ที่รองรับ:
- static member      : static method หรือ property
- member             : instance method หรือ property
- operator (+,-,*,/) : arithmetic operators
- comparison         : >, <, >=, <=
- equality           : =, <>
- null               : null check

ข้อดี:
- Zero runtime overhead (inlined)
- Structural typing (duck typing)
- ทำงานกับ existing types
- Type-safe

ข้อเสีย:
- ต้องเป็น inline เสมอ
- ไม่รองรับ runtime polymorphism
- Error messages ซับซ้อน
- IL ขนาดใหญ่
- จำกัดใน higher-order functions

เมื่อใช้ SRTP:
- Generic math/numeric operations
- Parser/serializer สำหรับหลาย types
- Generic algorithms ที่ต้องการ performance
- Type-safe duck typing

เมื่อใช้ Interface:
- Runtime polymorphism
- ต้องเก็บ function ใน data structure
- เมื่อ error message ต้องชัดเจน
- เมื่อ IDE support สำคัญ
*)

printfn "SRTP - สรุปเสร็จ!"
printfn "\nจบ Part 50 - Statically Resolved Type Parameters"
printfn "จบหลักสูตร F# Parts 41-50!"
```

## ตารางสรุป SRTP Constraints

| Constraint | Syntax | ตัวอย่าง |
|------------|--------|---------|
| Static member | `^T : (static member Name : Type)` | `(^T : (static member Zero : ^T))` |
| Instance member | `^T : (member Name : Type)` | `(^T : (member Length : int) x)` |
| Operator | ต้องเป็น `inline` | `x + y` |
| Comparison | `^T : comparison` | `a < b` |
| Equality | `^T : equality` | `a = b` |
| Null | `^T : null` | `x = null` |
| Default ctor | `^T : (new : unit -> ^T)` | `new ^T()` |
