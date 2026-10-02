# Part 71 - Unit Testing กับ xUnit

## บทนำ (Introduction)

Unit Testing เป็นส่วนสำคัญของการพัฒนาซอฟต์แวร์คุณภาพสูง ใน F# เราสามารถใช้ xUnit.net ซึ่งเป็น testing framework ยอดนิยมสำหรับ .NET ecosystem

xUnit.net มีคุณสมบัติเด่น:
- เรียบง่ายและทรงพลัง
- รองรับ parallel testing
- มี attribute-based test definition
- รองรับ async tests
- Integration กับ Visual Studio, VS Code, JetBrains Rider

---

## 1. การติดตั้ง xUnit กับ F# (Setup)

### สร้าง Test Project

```bash
# สร้าง test project ใหม่
dotnet new xunit -lang F# -n MyProject.Tests

# หรือเพิ่มลงใน solution ที่มีอยู่
dotnet new xunit -lang F# -o tests/MyProject.Tests

# เพิ่ม reference ไปยัง project หลัก
cd tests/MyProject.Tests
dotnet add reference ../../src/MyProject/MyProject.fsproj
```

### โครงสร้าง .fsproj สำหรับ Test Project

```xml
<!-- MyProject.Tests.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <!-- xUnit packages -->
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.3">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="coverlet.collector" Version="6.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>

  <ItemGroup>
    <Compile Include="BasicTests.fs" />
    <Compile Include="OptionTests.fs" />
    <Compile Include="ResultTests.fs" />
  </ItemGroup>

</Project>
```

### รัน Tests

```bash
# รัน tests ทั้งหมด
dotnet test

# รันพร้อม verbose output
dotnet test --verbosity normal

# รัน test เฉพาะที่ชื่อตรงกัน
dotnet test --filter "FullyQualifiedName~AddNumbers"

# รัน tests ใน category
dotnet test --filter "Category=Unit"
```

---

## 2. [<Fact>] Attribute - การสร้าง Test เบื้องต้น

```fsharp
// BasicTests.fs
module MyProject.Tests.BasicTests

open Xunit

// Test พื้นฐานที่สุด - ฟังก์ชันที่ไม่รับ parameter
[<Fact>]
let ``simple truth test`` () =
    Assert.True(true)

[<Fact>]
let ``one plus one equals two`` () =
    let result = 1 + 1
    Assert.Equal(2, result)

// ฟังก์ชันบวกเลขที่เราจะทดสอบ
let add x y = x + y

[<Fact>]
let ``add two positive numbers`` () =
    let result = add 3 4
    Assert.Equal(7, result)

[<Fact>]
let ``add negative numbers`` () =
    let result = add (-3) (-4)
    Assert.Equal(-7, result)

[<Fact>]
let ``add zero to number`` () =
    let result = add 5 0
    Assert.Equal(5, result)

// Testing string operations
let greet name = sprintf "Hello, %s!" name

[<Fact>]
let ``greet returns correct greeting`` () =
    let result = greet "World"
    Assert.Equal("Hello, World!", result)

[<Fact>]
let ``greet with empty string`` () =
    let result = greet ""
    Assert.Equal("Hello, !", result)

// Testing list operations
let sumList lst = List.sum lst

[<Fact>]
let ``sum of empty list is zero`` () =
    let result = sumList []
    Assert.Equal(0, result)

[<Fact>]
let ``sum of single element list`` () =
    let result = sumList [42]
    Assert.Equal(42, result)

[<Fact>]
let ``sum of multiple elements`` () =
    let result = sumList [1; 2; 3; 4; 5]
    Assert.Equal(15, result)
```

---

## 3. [<Theory>] กับ [<InlineData>] - Parameterized Tests

```fsharp
// TheoryTests.fs
module MyProject.Tests.TheoryTests

open Xunit

// ฟังก์ชันที่จะทดสอบ
let multiply x y = x * y
let isPrime n =
    if n < 2 then false
    else
        let limit = int (sqrt (float n))
        [2..limit] |> List.forall (fun i -> n % i <> 0)

// Theory test พื้นฐาน
[<Theory>]
[<InlineData(2, 3, 6)>]
[<InlineData(0, 5, 0)>]
[<InlineData(-2, 3, -6)>]
[<InlineData(10, 10, 100)>]
let ``multiply produces correct result`` (x: int) (y: int) (expected: int) =
    let result = multiply x y
    Assert.Equal(expected, result)

// Theory สำหรับ prime numbers
[<Theory>]
[<InlineData(2, true)>]
[<InlineData(3, true)>]
[<InlineData(5, true)>]
[<InlineData(7, true)>]
[<InlineData(11, true)>]
[<InlineData(13, true)>]
[<InlineData(1, false)>]
[<InlineData(4, false)>]
[<InlineData(6, false)>]
[<InlineData(9, false)>]
let ``isPrime returns correct result`` (n: int) (expected: bool) =
    let result = isPrime n
    Assert.Equal(expected, result)

// Theory กับ string data
let capitalize (s: string) =
    if s = null || s.Length = 0 then s
    else System.Char.ToUpper(s.[0]) |> string |> fun first -> first + s.[1..]

[<Theory>]
[<InlineData("hello", "Hello")>]
[<InlineData("world", "World")>]
[<InlineData("fsharp", "Fsharp")>]
[<InlineData("", "")>]
let ``capitalize returns correct result`` (input: string) (expected: string) =
    let result = capitalize input
    Assert.Equal(expected, result)

// Theory กับ MemberData - ใช้สำหรับ complex data
type FibonacciData() =
    static member Data : obj[][] =
        [|
            [| 0; 0 |]
            [| 1; 1 |]
            [| 2; 1 |]
            [| 3; 2 |]
            [| 4; 3 |]
            [| 5; 5 |]
            [| 6; 8 |]
            [| 7; 13 |]
        |]

let rec fibonacci n =
    match n with
    | 0 -> 0
    | 1 -> 1
    | n -> fibonacci (n-1) + fibonacci (n-2)

[<Theory>]
[<MemberData(nameof FibonacciData.Data, MemberType = typeof<FibonacciData>)>]
let ``fibonacci returns correct result`` (n: int) (expected: int) =
    let result = fibonacci n
    Assert.Equal(expected, result)

// ClassData สำหรับ data ที่ซับซ้อน
type StringOperationData() =
    interface System.Collections.Generic.IEnumerable<obj[]> with
        member _.GetEnumerator() =
            let data = [
                [| "hello world"; 11 |]
                [| ""; 0 |]
                [| "a"; 1 |]
                [| "fsharp is great"; 15 |]
            ]
            (data |> Seq.map (fun x -> x :> obj[]) :> System.Collections.Generic.IEnumerable<obj[]>).GetEnumerator()
    interface System.Collections.IEnumerable with
        member this.GetEnumerator() =
            (this :> System.Collections.Generic.IEnumerable<obj[]>).GetEnumerator() :> System.Collections.IEnumerator

[<Theory>]
[<ClassData(typeof<StringOperationData>)>]
let ``string length is correct`` (input: string) (expected: int) =
    Assert.Equal(expected, input.Length)
```

---

## 4. Assert Methods - วิธีการ Assert ต่างๆ

```fsharp
// AssertTests.fs
module MyProject.Tests.AssertTests

open Xunit
open System.Collections.Generic

// === Boolean Assertions ===

[<Fact>]
let ``assert true and false`` () =
    Assert.True(1 = 1)
    Assert.False(1 = 2)
    Assert.True(true, "This should be true")
    Assert.False(false, "This should be false")

// === Equality Assertions ===

[<Fact>]
let ``assert equal with different types`` () =
    // Int equality
    Assert.Equal(42, 42)
    
    // String equality
    Assert.Equal("hello", "hello")
    
    // Float equality with precision
    Assert.Equal(3.14, 3.14, 2)  // 2 decimal places
    
    // Approximate float comparison
    Assert.Equal(3.14159, System.Math.PI, 3)  // 3 decimal places

[<Fact>]
let ``assert not equal`` () =
    Assert.NotEqual(1, 2)
    Assert.NotEqual("hello", "world")

// === Null Assertions ===

[<Fact>]
let ``assert null and not null`` () =
    let nullString: string = null
    let nonNullString = "hello"
    
    Assert.Null(nullString)
    Assert.NotNull(nonNullString)

// === Collection Assertions ===

[<Fact>]
let ``assert collection equality`` () =
    let expected = [1; 2; 3; 4; 5]
    let actual = [1; 2; 3; 4; 5]
    
    Assert.Equal<int list>(expected, actual)

[<Fact>]
let ``assert collection contains`` () =
    let collection = [1; 2; 3; 4; 5]
    
    Assert.Contains(3, collection)
    Assert.DoesNotContain(6, collection)

[<Fact>]
let ``assert collection with predicate`` () =
    let numbers = [1; 2; 3; 4; 5]
    
    // Assert that collection contains element matching predicate
    Assert.Contains(numbers, fun x -> x > 4)
    Assert.DoesNotContain(numbers, fun x -> x > 10)

[<Fact>]
let ``assert empty collection`` () =
    let empty: int list = []
    let nonEmpty = [1; 2; 3]
    
    Assert.Empty(empty)
    Assert.NotEmpty(nonEmpty)

[<Fact>]
let ``assert collection count`` () =
    let items = ["a"; "b"; "c"]
    Assert.Equal(3, items.Length)

// === Type Assertions ===

[<Fact>]
let ``assert is type`` () =
    let obj: obj = 42 :> obj
    let result = Assert.IsType<int>(obj)
    Assert.Equal(42, result)

[<Fact>]
let ``assert is assignable from type`` () =
    let list = [1; 2; 3]
    Assert.IsAssignableFrom<System.Collections.Generic.IEnumerable<int>>(list)

// === String Assertions ===

[<Fact>]
let ``assert string operations`` () =
    let message = "Hello, World!"
    
    Assert.Contains("World", message)
    Assert.DoesNotContain("Python", message)
    Assert.StartsWith("Hello", message)
    Assert.EndsWith("!", message)
    Assert.Matches(@"Hello,\s\w+!", message)  // Regex match
    Assert.DoesNotMatch(@"\d+", message)       // Regex doesn't match

// === Range Assertions ===

[<Fact>]
let ``assert in range`` () =
    let value = 5
    Assert.InRange(value, 1, 10)
    Assert.NotInRange(value, 11, 20)
```

---

## 5. Testing Pure Functions

```fsharp
// PureFunctionTests.fs
module MyProject.Tests.PureFunctionTests

open Xunit

// Domain types
type Temperature =
    | Celsius of float
    | Fahrenheit of float
    | Kelvin of float

// Pure functions ที่จะทดสอบ
module Temperature =
    let toCelsius = function
        | Celsius c -> c
        | Fahrenheit f -> (f - 32.0) * 5.0 / 9.0
        | Kelvin k -> k - 273.15
    
    let toFahrenheit = function
        | Celsius c -> c * 9.0 / 5.0 + 32.0
        | Fahrenheit f -> f
        | Kelvin k -> (k - 273.15) * 9.0 / 5.0 + 32.0
    
    let toKelvin = function
        | Celsius c -> c + 273.15
        | Fahrenheit f -> (f - 32.0) * 5.0 / 9.0 + 273.15
        | Kelvin k -> k

// Tests สำหรับ temperature conversions
[<Fact>]
let ``celsius to fahrenheit freezing point`` () =
    let result = Temperature.toFahrenheit (Celsius 0.0)
    Assert.Equal(32.0, result, 2)

[<Fact>]
let ``celsius to fahrenheit boiling point`` () =
    let result = Temperature.toFahrenheit (Celsius 100.0)
    Assert.Equal(212.0, result, 2)

[<Fact>]
let ``fahrenheit to celsius body temperature`` () =
    let result = Temperature.toCelsius (Fahrenheit 98.6)
    Assert.Equal(37.0, result, 1)

[<Fact>]
let ``kelvin to celsius absolute zero`` () =
    let result = Temperature.toCelsius (Kelvin 0.0)
    Assert.Equal(-273.15, result, 2)

// Testing list processing functions
module ListProcessing =
    let partition pred lst =
        List.foldBack (fun x (trues, falses) ->
            if pred x then (x :: trues, falses)
            else (trues, x :: falses)
        ) lst ([], [])
    
    let chunksOf n lst =
        let rec go acc current remaining =
            match remaining with
            | [] ->
                if List.isEmpty current then List.rev acc
                else List.rev (List.rev current :: acc)
            | x :: rest ->
                if List.length current = n then
                    go (List.rev current :: acc) [x] rest
                else
                    go acc (x :: current) rest
        go [] [] lst

[<Fact>]
let ``partition separates evens and odds`` () =
    let numbers = [1..10]
    let (evens, odds) = ListProcessing.partition (fun x -> x % 2 = 0) numbers
    
    Assert.Equal<int list>([2; 4; 6; 8; 10], evens)
    Assert.Equal<int list>([1; 3; 5; 7; 9], odds)

[<Fact>]
let ``chunksOf creates correct sized chunks`` () =
    let numbers = [1..10]
    let chunks = ListProcessing.chunksOf 3 numbers
    
    Assert.Equal(4, List.length chunks)
    Assert.Equal<int list>([1; 2; 3], List.item 0 chunks)
    Assert.Equal<int list>([4; 5; 6], List.item 1 chunks)
    Assert.Equal<int list>([7; 8; 9], List.item 2 chunks)
    Assert.Equal<int list>([10], List.item 3 chunks)

[<Fact>]
let ``chunksOf with empty list returns empty`` () =
    let result = ListProcessing.chunksOf 3 []
    Assert.Empty(result)

// Testing mathematical functions
module Math =
    let clamp min max value =
        if value < min then min
        elif value > max then max
        else value
    
    let lerp t a b = a + t * (b - a)
    
    let roundToNearest step value =
        step * System.Math.Round(value / step)

[<Fact>]
let ``clamp keeps value in range`` () =
    Assert.Equal(5, Math.clamp 0 10 5)
    Assert.Equal(0, Math.clamp 0 10 -5)
    Assert.Equal(10, Math.clamp 0 10 15)

[<Theory>]
[<InlineData(0.0, 0.0, 10.0, 0.0)>]
[<InlineData(0.5, 0.0, 10.0, 5.0)>]
[<InlineData(1.0, 0.0, 10.0, 10.0)>]
[<InlineData(0.25, 4.0, 8.0, 5.0)>]
let ``lerp interpolates correctly`` (t: float) (a: float) (b: float) (expected: float) =
    let result = Math.lerp t a b
    Assert.Equal(expected, result, 5)
```

---

## 6. Testing กับ Option Type

```fsharp
// OptionTests.fs
module MyProject.Tests.OptionTests

open Xunit

// Functions ที่ return Option
let safeDivide x y =
    if y = 0 then None
    else Some (x / y)

let findFirst pred lst =
    List.tryFind pred lst

let tryParseInt (s: string) =
    match System.Int32.TryParse(s) with
    | true, value -> Some value
    | false, _ -> None

let getFirstElement lst =
    List.tryHead lst

// Tests สำหรับ Option
[<Fact>]
let ``safe divide returns Some for valid division`` () =
    let result = safeDivide 10 2
    Assert.Equal(Some 5, result)

[<Fact>]
let ``safe divide returns None for division by zero`` () =
    let result = safeDivide 10 0
    Assert.Equal(None, result)

// Helper สำหรับ unwrap Option
let unwrapSome opt =
    match opt with
    | Some v -> v
    | None -> failwith "Expected Some but got None"

[<Fact>]
let ``safe divide result is accessible`` () =
    let result = safeDivide 15 3
    let value = unwrapSome result
    Assert.Equal(5, value)

[<Fact>]
let ``findFirst returns Some when found`` () =
    let numbers = [1; 2; 3; 4; 5]
    let result = findFirst (fun x -> x > 3) numbers
    
    Assert.True(result.IsSome)
    Assert.Equal(4, result.Value)

[<Fact>]
let ``findFirst returns None when not found`` () =
    let numbers = [1; 2; 3]
    let result = findFirst (fun x -> x > 10) numbers
    
    Assert.True(result.IsNone)

// Testing Option chaining
[<Fact>]
let ``option map transforms value`` () =
    let result = 
        Some 5
        |> Option.map (fun x -> x * 2)
        |> Option.map (fun x -> x + 1)
    
    Assert.Equal(Some 11, result)

[<Fact>]
let ``option map on None returns None`` () =
    let result = 
        None
        |> Option.map (fun x -> x * 2)
    
    Assert.Equal(None, result)

[<Fact>]
let ``option bind chains optional operations`` () =
    let divide x y = if y = 0 then None else Some (x / y)
    
    let result = 
        Some 100
        |> Option.bind (fun x -> divide x 5)
        |> Option.bind (fun x -> divide x 4)
    
    Assert.Equal(Some 5, result)

[<Fact>]
let ``option bind short circuits on None`` () =
    let mutable callCount = 0
    
    let increment () =
        callCount <- callCount + 1
        None
    
    let result = 
        None
        |> Option.bind (fun _ -> increment())
        |> Option.bind (fun _ -> increment())
    
    Assert.Equal(None, result)
    Assert.Equal(0, callCount)

[<Fact>]
let ``tryParseInt parses valid numbers`` () =
    let cases = [
        ("42", Some 42)
        ("0", Some 0)
        ("-100", Some -100)
        ("not a number", None)
        ("", None)
    ]
    
    for (input, expected) in cases do
        Assert.Equal(expected, tryParseInt input)

[<Fact>]
let ``getFirstElement returns head or None`` () =
    Assert.Equal(None, getFirstElement ([] : int list))
    Assert.Equal(Some 1, getFirstElement [1; 2; 3])
    Assert.Equal(Some "hello", getFirstElement ["hello"; "world"])

// Testing Option.defaultValue
[<Fact>]
let ``option defaultValue provides fallback`` () =
    let someValue = Some 42
    let noneValue: int option = None
    
    Assert.Equal(42, Option.defaultValue 0 someValue)
    Assert.Equal(0, Option.defaultValue 0 noneValue)

// Testing Option.orElse
[<Fact>]
let ``option orElse falls back to alternative`` () =
    let primary: int option = None
    let fallback = Some 99
    
    let result = primary |> Option.orElse fallback
    Assert.Equal(Some 99, result)
```

---

## 7. Testing กับ Result Type

```fsharp
// ResultTests.fs
module MyProject.Tests.ResultTests

open Xunit

// Domain errors
type ValidationError =
    | EmptyName
    | NameTooLong of int
    | InvalidAge of int
    | InvalidEmail of string

type Person = { Name: string; Age: int; Email: string }

// Validation functions ที่ return Result
let validateName (name: string) =
    if System.String.IsNullOrWhiteSpace(name) then Error EmptyName
    elif name.Length > 50 then Error (NameTooLong name.Length)
    else Ok name

let validateAge age =
    if age < 0 || age > 150 then Error (InvalidAge age)
    else Ok age

let validateEmail (email: string) =
    if System.String.IsNullOrWhiteSpace(email) then Error (InvalidEmail "empty")
    elif not (email.Contains("@")) then Error (InvalidEmail email)
    else Ok email

// Tests สำหรับ Result
[<Fact>]
let ``validateName returns Ok for valid name`` () =
    let result = validateName "John Doe"
    Assert.Equal(Ok "John Doe", result)

[<Fact>]
let ``validateName returns error for empty name`` () =
    let result = validateName ""
    Assert.Equal(Error EmptyName, result)

[<Fact>]
let ``validateName returns error for whitespace name`` () =
    let result = validateName "   "
    Assert.Equal(Error EmptyName, result)

[<Fact>]
let ``validateName returns error for long name`` () =
    let longName = System.String.replicate 51 "a"
    let result = validateName longName
    match result with
    | Error (NameTooLong length) -> Assert.Equal(51, length)
    | _ -> Assert.Fail("Expected NameTooLong error")

[<Fact>]
let ``validateAge returns Ok for valid age`` () =
    let result = validateAge 25
    Assert.Equal(Ok 25, result)

[<Theory>]
[<InlineData(-1)>]
[<InlineData(151)>]
[<InlineData(-100)>]
let ``validateAge returns error for invalid ages`` (age: int) =
    let result = validateAge age
    match result with
    | Error (InvalidAge n) -> Assert.Equal(age, n)
    | _ -> Assert.Fail($"Expected InvalidAge error for {age}")

[<Fact>]
let ``validateEmail returns Ok for valid email`` () =
    let result = validateEmail "test@example.com"
    Assert.Equal(Ok "test@example.com", result)

[<Fact>]
let ``validateEmail returns error for missing at sign`` () =
    let result = validateEmail "notanemail"
    Assert.Equal(Error (InvalidEmail "notanemail"), result)

// Testing Result chaining with bind
[<Fact>]
let ``result bind chains validations`` () =
    let validatePositive n =
        if n > 0 then Ok n
        else Error "Not positive"
    
    let validateEven n =
        if n % 2 = 0 then Ok n
        else Error "Not even"
    
    let result =
        Ok 4
        |> Result.bind validatePositive
        |> Result.bind validateEven
    
    Assert.Equal(Ok 4, result)

[<Fact>]
let ``result bind short circuits on error`` () =
    let mutable callCount = 0
    
    let validatePositive n =
        callCount <- callCount + 1
        if n > 0 then Ok n
        else Error "Not positive"
    
    let result =
        Ok (-1)
        |> Result.bind validatePositive  // This sets error
        |> Result.bind validatePositive  // This should NOT be called
    
    Assert.Equal(1, callCount)
    Assert.Equal(Error "Not positive", result)

// Testing Result.map
[<Fact>]
let ``result map transforms Ok value`` () =
    let result =
        Ok 5
        |> Result.map (fun x -> x * 2)
        |> Result.map (fun x -> x.ToString())
    
    Assert.Equal(Ok "10", result)

[<Fact>]
let ``result map does not affect Error`` () =
    let result =
        Error "original error"
        |> Result.map (fun x -> x + " modified")
    
    Assert.Equal(Error "original error", result)

// Testing Result.mapError
[<Fact>]
let ``result mapError transforms error value`` () =
    let result =
        Error "error"
        |> Result.mapError (fun e -> e.ToUpper())
    
    Assert.Equal(Error "ERROR", result)

// Testing isOk and isError
[<Fact>]
let ``result isOk and isError helpers`` () =
    let ok = Ok 42
    let error = Error "failure"
    
    Assert.True(Result.isOk ok)
    Assert.False(Result.isOk error)
    Assert.True(Result.isError error)
    Assert.False(Result.isError ok)
```

---

## 8. Testing Exceptions (Assert.Throws)

```fsharp
// ExceptionTests.fs
module MyProject.Tests.ExceptionTests

open Xunit
open System

// Functions ที่ throw exceptions
let divide x y =
    if y = 0 then raise (DivideByZeroException("Cannot divide by zero"))
    else x / y

let getElement (arr: 'a array) index =
    if index < 0 || index >= arr.Length then
        raise (IndexOutOfRangeException(sprintf "Index %d is out of range [0, %d)" index arr.Length))
    arr.[index]

let parsePositiveInt (s: string) =
    let n = int s
    if n <= 0 then invalidArg "s" "Value must be positive"
    n

// Tests สำหรับ exceptions
[<Fact>]
let ``divide by zero throws DivideByZeroException`` () =
    Assert.Throws<DivideByZeroException>(fun () -> 
        divide 10 0 |> ignore
    )

[<Fact>]
let ``divide by zero exception has correct message`` () =
    let ex = Assert.Throws<DivideByZeroException>(fun () -> 
        divide 10 0 |> ignore
    )
    Assert.Equal("Cannot divide by zero", ex.Message)

[<Fact>]
let ``array access out of range throws exception`` () =
    let arr = [|1; 2; 3|]
    Assert.Throws<IndexOutOfRangeException>(fun () -> 
        getElement arr 5 |> ignore
    )

[<Fact>]
let ``negative index throws IndexOutOfRangeException`` () =
    let arr = [|1; 2; 3|]
    let ex = Assert.Throws<IndexOutOfRangeException>(fun () -> 
        getElement arr (-1) |> ignore
    )
    Assert.Contains("-1", ex.Message)

[<Fact>]
let ``valid array access does not throw`` () =
    let arr = [|10; 20; 30|]
    let result = getElement arr 1
    Assert.Equal(20, result)

[<Fact>]
let ``parsePositiveInt throws for non-positive`` () =
    Assert.Throws<ArgumentException>(fun () -> 
        parsePositiveInt "0" |> ignore
    )
    Assert.Throws<ArgumentException>(fun () -> 
        parsePositiveInt "-5" |> ignore
    )

[<Fact>]
let ``parsePositiveInt throws for invalid format`` () =
    Assert.Throws<FormatException>(fun () -> 
        parsePositiveInt "not a number" |> ignore
    )

// Testing กับ custom exceptions
type BusinessException(message: string, code: int) =
    inherit Exception(message)
    member _.Code = code

let processOrder quantity =
    if quantity <= 0 then
        raise (BusinessException("Quantity must be positive", 1001))
    elif quantity > 1000 then
        raise (BusinessException("Quantity exceeds maximum", 1002))
    else
        quantity * 10  // total price

[<Fact>]
let ``processOrder throws BusinessException for zero quantity`` () =
    let ex = Assert.Throws<BusinessException>(fun () -> 
        processOrder 0 |> ignore
    )
    Assert.Equal(1001, ex.Code)
    Assert.Equal("Quantity must be positive", ex.Message)

[<Fact>]
let ``processOrder throws BusinessException for excessive quantity`` () =
    let ex = Assert.Throws<BusinessException>(fun () -> 
        processOrder 1001 |> ignore
    )
    Assert.Equal(1002, ex.Code)

[<Fact>]
let ``processOrder calculates correct total`` () =
    let result = processOrder 5
    Assert.Equal(50, result)
```

---

## 9. Setup และ Teardown กับ IDisposable

```fsharp
// SetupTeardownTests.fs
module MyProject.Tests.SetupTeardownTests

open Xunit
open System
open System.IO

// Test class ที่ใช้ IDisposable สำหรับ cleanup
type FileTests() =
    // สร้าง temp file ใน constructor (Setup)
    let tempFile = Path.GetTempFileName()
    
    do
        // เขียนข้อมูลเริ่มต้น
        File.WriteAllText(tempFile, "initial content")
    
    // Cleanup ใน Dispose (Teardown)
    interface IDisposable with
        member _.Dispose() =
            if File.Exists(tempFile) then
                File.Delete(tempFile)
    
    [<Fact>]
    member _.``file exists after creation`` () =
        Assert.True(File.Exists(tempFile))
    
    [<Fact>]
    member _.``file has correct initial content`` () =
        let content = File.ReadAllText(tempFile)
        Assert.Equal("initial content", content)
    
    [<Fact>]
    member _.``can write and read from file`` () =
        File.WriteAllText(tempFile, "new content")
        let content = File.ReadAllText(tempFile)
        Assert.Equal("new content", content)
    
    [<Fact>]
    member _.``can append to file`` () =
        File.AppendAllText(tempFile, " appended")
        let content = File.ReadAllText(tempFile)
        Assert.Equal("initial content appended", content)

// Test class สำหรับ database-like state
type CounterTests() =
    let mutable counter = 0
    
    // Reset counter ก่อนแต่ละ test
    member _.Reset() =
        counter <- 0
    
    interface IDisposable with
        member this.Dispose() =
            this.Reset()
    
    [<Fact>]
    member _.``counter starts at zero`` () =
        Assert.Equal(0, counter)
    
    [<Fact>]
    member _.``increment increases counter`` () =
        counter <- counter + 1
        Assert.Equal(1, counter)
    
    [<Fact>]
    member _.``multiple increments accumulate`` () =
        counter <- counter + 1
        counter <- counter + 1
        counter <- counter + 1
        Assert.Equal(3, counter)

// Setup ผ่าน constructor injection
type DatabaseSimulation = {
    mutable Records: Map<int, string>
    mutable IsConnected: bool
}

let createDatabase () = {
    Records = Map.empty
    IsConnected = true
}

type DatabaseTests() =
    let db = createDatabase()
    
    interface IDisposable with
        member _.Dispose() =
            // Simulate disconnect
            ()
    
    [<Fact>]
    member _.``database starts empty`` () =
        Assert.Empty(db.Records)
    
    [<Fact>]
    member _.``can add record`` () =
        let updated = { db with Records = db.Records |> Map.add 1 "Record 1" }
        Assert.Equal(1, updated.Records.Count)
        Assert.Equal("Record 1", updated.Records.[1])
    
    [<Fact>]
    member _.``database is connected on creation`` () =
        Assert.True(db.IsConnected)
```

---

## 10. Test Fixtures

```fsharp
// FixtureTests.fs
module MyProject.Tests.FixtureTests

open Xunit
open System

// Shared fixture สำหรับ expensive setup
type DatabaseFixture() =
    let connectionString = "Server=localhost;Database=TestDB;..."
    
    // Simulate expensive database setup
    do
        printfn "Setting up database connection..."
    
    member _.ConnectionString = connectionString
    
    member _.ExecuteQuery (sql: string) =
        // Simulate query execution
        sprintf "Result of: %s" sql
    
    interface IDisposable with
        member _.Dispose() =
            printfn "Tearing down database connection..."

// Collection definition - แบ่งปัน fixture ระหว่าง test classes
[<CollectionDefinition("Database collection")>]
type DatabaseCollection() =
    interface ICollectionFixture<DatabaseFixture>

[<Collection("Database collection")>]
type DatabaseReadTests(fixture: DatabaseFixture) =
    [<Fact>]
    member _.``can execute select query`` () =
        let result = fixture.ExecuteQuery "SELECT * FROM users"
        Assert.Contains("SELECT * FROM users", result)
    
    [<Fact>]
    member _.``connection string is available`` () =
        Assert.NotNull(fixture.ConnectionString)
        Assert.NotEmpty(fixture.ConnectionString)

[<Collection("Database collection")>]
type DatabaseWriteTests(fixture: DatabaseFixture) =
    [<Fact>]
    member _.``can execute insert query`` () =
        let result = fixture.ExecuteQuery "INSERT INTO users VALUES (...)"
        Assert.Contains("INSERT", result)
    
    [<Fact>]
    member _.``can execute update query`` () =
        let result = fixture.ExecuteQuery "UPDATE users SET name='John' WHERE id=1"
        Assert.Contains("UPDATE", result)
```

---

## 11. Async Tests

```fsharp
// AsyncTests.fs
module MyProject.Tests.AsyncTests

open Xunit
open System.Threading.Tasks
open System.Threading

// Async functions ที่จะทดสอบ
let fetchDataAsync (delay: int) = async {
    do! Async.Sleep delay
    return "data fetched"
}

let calculateAsync x y = async {
    do! Async.Sleep 10
    return x + y
}

let processItemsAsync items = async {
    let! results = 
        items 
        |> List.map (fun x -> async { return x * 2 })
        |> Async.Parallel
    return results |> Array.toList
}

// Async Tests กับ xUnit
[<Fact>]
let ``async fetch returns data`` () =
    let result = 
        fetchDataAsync 50
        |> Async.RunSynchronously
    Assert.Equal("data fetched", result)

[<Fact>]
let ``async calculation returns correct result`` () =
    let result = 
        calculateAsync 3 4
        |> Async.RunSynchronously
    Assert.Equal(7, result)

// Task-based async tests
[<Fact>]
let ``task based async test`` () : Task =
    task {
        let! result = Task.FromResult(42)
        Assert.Equal(42, result)
    }

[<Fact>]
let ``task with delay`` () : Task =
    task {
        do! Task.Delay(50)
        let result = "completed"
        Assert.Equal("completed", result)
    }

// Async with cancellation
[<Fact>]
let ``async supports cancellation`` () =
    let cts = new CancellationTokenSource()
    cts.CancelAfter(100)
    
    let longRunning = async {
        do! Async.Sleep 50
        return "completed"
    }
    
    let result = Async.RunSynchronously(longRunning, cancellationToken = cts.Token)
    Assert.Equal("completed", result)

// Testing async exceptions
[<Fact>]
let ``async throws exception`` () =
    let failingAsync = async {
        do! Async.Sleep 10
        failwith "async error"
        return "never"
    }
    
    Assert.Throws<System.Exception>(fun () ->
        Async.RunSynchronously(failingAsync) |> ignore
    )

// Parallel async operations
[<Fact>]
let ``parallel async operations complete correctly`` () =
    let result = 
        processItemsAsync [1; 2; 3; 4; 5]
        |> Async.RunSynchronously
    
    Assert.Equal(5, result.Length)
    Assert.Contains(2, result)
    Assert.Contains(10, result)
```

---

## 12. Skip Attribute

```fsharp
// SkipTests.fs
module MyProject.Tests.SkipTests

open Xunit

// Skip test ที่ยังไม่พร้อม
[<Fact(Skip = "Feature not yet implemented")>]
let ``this test is skipped`` () =
    Assert.True(false)  // Would fail if run

// Skip test ที่ต้องใช้ external resource
[<Fact(Skip = "Requires database connection")>]
let ``database integration test`` () =
    // This would need a real database
    Assert.True(false)

// Skip theory
[<Theory(Skip = "Performance test - run manually")>]
[<InlineData(1000)>]
[<InlineData(10000)>]
let ``performance test`` (iterations: int) =
    let mutable sum = 0
    for _ in 1..iterations do
        sum <- sum + 1
    Assert.Equal(iterations, sum)

// Conditional skip using custom approach
[<Fact>]
let ``test that may be skipped conditionally`` () =
    let isCI = System.Environment.GetEnvironmentVariable("CI") = "true"
    if isCI then
        ()  // Skip in CI
    else
        // Run test logic
        Assert.True(true)
```

---

## 13. Test Naming Conventions

```fsharp
// NamingConventionTests.fs
module MyProject.Tests.NamingConventionTests

open Xunit

// Pattern 1: "method_state_expected"
module MethodStateExpected =
    let divide x y =
        if y = 0 then None
        else Some (x / y)
    
    [<Fact>]
    let divide_ByZero_ReturnsNone () =
        Assert.Equal(None, divide 10 0)
    
    [<Fact>]
    let divide_ByPositiveNumber_ReturnsSome () =
        Assert.Equal(Some 5, divide 10 2)

// Pattern 2: "Given_When_Then" (BDD style)
module GivenWhenThen =
    let calculateDiscount price quantity =
        if quantity >= 10 then price * 0.9
        elif quantity >= 5 then price * 0.95
        else price
    
    [<Fact>]
    let ``Given quantity is 10 When calculating discount Then 10 percent discount applies`` () =
        let result = calculateDiscount 100.0 10
        Assert.Equal(90.0, result, 2)
    
    [<Fact>]
    let ``Given quantity is 3 When calculating discount Then no discount applies`` () =
        let result = calculateDiscount 100.0 3
        Assert.Equal(100.0, result, 2)

// Pattern 3: Descriptive F# backtick names
module DescriptiveNames =
    let isAdult age = age >= 18
    
    [<Fact>]
    let ``18 year old is considered adult`` () =
        Assert.True(isAdult 18)
    
    [<Fact>]
    let ``17 year old is not considered adult`` () =
        Assert.False(isAdult 17)
    
    [<Fact>]
    let ``age 0 is not adult`` () =
        Assert.False(isAdult 0)
```

---

## 14. F#-Friendly Assertions

```fsharp
// FriendlyAssertions.fs
module MyProject.Tests.FriendlyAssertions

open Xunit

// F# helper functions สำหรับ assertion ที่ readable มากขึ้น
let shouldEqual expected actual =
    Assert.Equal(expected, actual)

let shouldNotEqual unexpected actual =
    Assert.NotEqual(unexpected, actual)

let shouldBeTrue condition =
    Assert.True(condition)

let shouldBeFalse condition =
    Assert.False(condition)

let shouldContain item collection =
    Assert.Contains(item, collection)

let shouldBeEmpty collection =
    Assert.Empty(collection)

let shouldNotBeEmpty collection =
    Assert.NotEmpty(collection)

let shouldBeSome opt =
    match opt with
    | Some _ -> ()
    | None -> Assert.Fail("Expected Some but got None")

let shouldBeNone opt =
    match opt with
    | None -> ()
    | Some v -> Assert.Fail(sprintf "Expected None but got Some %A" v)

let shouldBeOk result =
    match result with
    | Ok _ -> ()
    | Error e -> Assert.Fail(sprintf "Expected Ok but got Error: %A" e)

let shouldBeError result =
    match result with
    | Error _ -> ()
    | Ok v -> Assert.Fail(sprintf "Expected Error but got Ok: %A" v)

// การใช้งาน helper functions
[<Fact>]
let ``using friendly assertion helpers`` () =
    let numbers = [1; 2; 3; 4; 5]
    let result = List.sum numbers
    
    result |> shouldEqual 15
    numbers |> shouldContain 3
    numbers |> shouldNotBeEmpty

[<Fact>]
let ``option assertions`` () =
    let someValue = Some 42
    let noneValue: int option = None
    
    someValue |> shouldBeSome
    noneValue |> shouldBeNone

[<Fact>]
let ``result assertions`` () =
    let okResult: Result<int, string> = Ok 42
    let errorResult: Result<int, string> = Error "failed"
    
    okResult |> shouldBeOk
    errorResult |> shouldBeError

// Assertion กับ discriminated unions
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

let area = function
    | Circle r -> System.Math.PI * r * r
    | Rectangle (w, h) -> w * h
    | Triangle (b, h) -> 0.5 * b * h

[<Fact>]
let ``area of circle is calculated correctly`` () =
    let shape = Circle 5.0
    let expectedArea = System.Math.PI * 25.0
    
    let actualArea = area shape
    Assert.Equal(expectedArea, actualArea, 5)

[<Fact>]
let ``area of rectangle is calculated correctly`` () =
    let shape = Rectangle (4.0, 6.0)
    area shape |> shouldEqual 24.0

[<Fact>]
let ``area of triangle is calculated correctly`` () =
    let shape = Triangle (3.0, 4.0)
    area shape |> shouldEqual 6.0
```

---

## 15. สรุป (Summary)

xUnit.net เป็น testing framework ที่ทรงพลังสำหรับ F# และ .NET ในบทนี้เราได้เรียนรู้:

1. **[<Fact>]** - การสร้าง test พื้นฐาน
2. **[<Theory>] + [<InlineData>]** - Parameterized tests
3. **Assert methods** - วิธีการ assert ต่างๆ
4. **Testing pure functions** - การทดสอบฟังก์ชัน pure
5. **Option testing** - การทดสอบค่า Option
6. **Result testing** - การทดสอบค่า Result
7. **Exception testing** - การทดสอบ exceptions ด้วย Assert.Throws
8. **IDisposable** - Setup และ teardown
9. **Fixtures** - การแบ่งปัน state ระหว่าง tests
10. **Async tests** - การทดสอบ async code
11. **Skip attribute** - การข้าม tests
12. **Test naming** - Convention การตั้งชื่อ test

### Best Practices

```fsharp
// Good: Test ที่ชัดเจนและ isolated
[<Fact>]
let ``validateInput returns error for empty string`` () =
    // Arrange
    let input = ""
    
    // Act
    let result = validateInput input
    
    // Assert
    Assert.Equal(Error "Input cannot be empty", result)

// Good: ใช้ descriptive names
[<Fact>]
let ``shopping cart total includes all items`` () =
    let cart = [
        { Name = "Item 1"; Price = 10.0; Quantity = 2 }
        { Name = "Item 2"; Price = 5.0; Quantity = 3 }
    ]
    let total = calculateTotal cart
    Assert.Equal(35.0, total, 2)

// Good: Test หนึ่ง assertion ต่อ concept
[<Fact>]
let ``registered user has default role`` () =
    let user = registerUser "john@example.com" "password"
    Assert.Equal(Role.User, user.Role)
```

### การรัน Tests

```bash
# รัน tests ทั้งหมด
dotnet test

# รันและดู output
dotnet test --verbosity normal

# รัน test เฉพาะ
dotnet test --filter "FullyQualifiedName~BasicTests"

# รัน tests และ collect coverage
dotnet test --collect:"XPlat Code Coverage"
```
