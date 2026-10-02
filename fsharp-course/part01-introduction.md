# Part 1 - แนะนำ F# และการติดตั้ง (Introduction to F# and Setup)

## บทนำ (Introduction)

F# (อ่านว่า "เอฟ ชาร์ป") เป็นภาษาโปรแกรมมิ่งที่พัฒนาโดย Microsoft Research และ Don Syme เปิดตัวครั้งแรกในปี 2005 และกลายเป็น open source ในปี 2010 F# เป็นภาษาที่ทำงานบน .NET platform ซึ่งหมายความว่าสามารถทำงานร่วมกับ C#, VB.NET และภาษาอื่นๆ ใน .NET ecosystem ได้อย่างสมบูรณ์

---

## 1.1 F# คืออะไร? (What is F#?)

F# เป็น **functional-first, strongly typed, multi-paradigm programming language** ที่:

- **Functional-first**: ออกแบบมาเพื่อการเขียนโปรแกรมแบบ functional เป็นหลัก แต่ก็รองรับ object-oriented และ imperative programming ด้วย
- **Strongly typed**: ระบบ type ที่แข็งแกร่งพร้อม type inference ที่ทรงพลัง ช่วยจับ bug ได้ตั้งแต่ compile time
- **Concise**: โค้ดกระชับ อ่านง่าย ไม่มี boilerplate มากเกินไป
- **Safe**: Immutable by default ลด side effects และ bugs
- **Interoperable**: ทำงานร่วมกับ .NET ecosystem ได้ทั้งหมด

### คุณสมบัติเด่นของ F#

```fsharp
// 1. Type Inference - ไม่ต้องระบุ type เสมอไป
let name = "Alice"        // F# รู้ว่าเป็น string
let age = 30              // F# รู้ว่าเป็น int
let pi = 3.14159          // F# รู้ว่าเป็น float

// 2. Immutable by default
let x = 5
// x <- 10  // Error! ไม่สามารถเปลี่ยนค่าได้ (ต้องใช้ mutable)

// 3. Functional style
let numbers = [1; 2; 3; 4; 5]
let doubled = numbers |> List.map (fun n -> n * 2)
// doubled = [2; 4; 6; 8; 10]

// 4. Pattern matching ที่ทรงพลัง
let describe n =
    match n with
    | 0 -> "zero"
    | 1 -> "one"
    | n when n < 0 -> "negative"
    | _ -> "positive"

// 5. Algebraic types
type Shape =
    | Circle of radius: float
    | Rectangle of width: float * height: float
    | Triangle of base': float * height: float

let area shape =
    match shape with
    | Circle r -> System.Math.PI * r * r
    | Rectangle(w, h) -> w * h
    | Triangle(b, h) -> 0.5 * b * h
```

---

## 1.2 ทำไมต้องใช้ F#? (Why Use F#?)

### 1.2.1 ประโยชน์หลัก

**1. Correctness (ความถูกต้อง)**
```fsharp
// F# บังคับให้จัดการทุก case
type PaymentStatus =
    | Pending
    | Approved of amount: decimal
    | Rejected of reason: string
    | Refunded of originalAmount: decimal * refundAmount: decimal

let processPayment status =
    match status with
    | Pending -> printfn "กำลังรอการชำระเงิน..."
    | Approved amount -> printfn "ชำระเงินสำเร็จ: %M บาท" amount
    | Rejected reason -> printfn "ปฏิเสธการชำระเงิน: %s" reason
    | Refunded(orig, refund) -> printfn "คืนเงิน %M จาก %M บาท" refund orig
// ถ้าขาด case ใดไป compiler จะเตือน!
```

**2. Conciseness (ความกระชับ)**
```fsharp
// C# style (verbose)
// var result = new List<int>();
// foreach (var n in numbers) {
//     if (n % 2 == 0) result.Add(n * n);
// }

// F# style (concise)
let result = numbers |> List.filter (fun n -> n % 2 = 0) |> List.map (fun n -> n * n)
```

**3. Composability (การประกอบกัน)**
```fsharp
// Pipe operator |> ทำให้โค้ดอ่านเป็น pipeline ที่ชัดเจน
let processData data =
    data
    |> List.filter (fun x -> x > 0)      // กรองค่าบวก
    |> List.map (fun x -> x * 2)          // คูณ 2
    |> List.sort                           // เรียงลำดับ
    |> List.distinct                       // เอาค่าไม่ซ้ำ
    |> List.take 5                         // เอา 5 ตัวแรก
```

**4. Parallelism (การประมวลผลพร้อมกัน)**
```fsharp
open System.Threading.Tasks

// Immutable data ทำให้ parallel programming ปลอดภัยขึ้น
let numbers = [| 1 .. 1000000 |]
let result = 
    numbers 
    |> Array.Parallel.map (fun x -> x * x)
    |> Array.sum
```

---

## 1.3 F# เทียบกับภาษาอื่นๆ (F# vs Other Languages)

### F# vs C#

| Feature | F# | C# |
|---------|----|----|
| Paradigm | Functional-first | OO-first |
| Mutability | Immutable by default | Mutable by default |
| Null handling | Option type | Nullable (null refs) |
| Pattern matching | Comprehensive | Limited (improving) |
| Type inference | More powerful | Limited |
| Verbosity | Low | Higher |
| Interop | Full .NET | Full .NET |

```fsharp
// F# - กระชับ, functional
type Person = { Name: string; Age: int }

let greet person =
    match person.Age with
    | age when age < 18 -> sprintf "สวัสดี %s น้องหนู!" person.Name
    | age when age < 60 -> sprintf "สวัสดีครับ/ค่ะ คุณ%s" person.Name
    | _ -> sprintf "สวัสดีครับ/ค่ะ ท่าน%s" person.Name

let people = [
    { Name = "Alice"; Age = 25 }
    { Name = "Bob"; Age = 15 }
    { Name = "Charlie"; Age = 65 }
]

people |> List.iter (greet >> printfn "%s")
```

```csharp
// C# equivalent - verbose, OO style
public record Person(string Name, int Age);

public static string Greet(Person person)
{
    if (person.Age < 18)
        return $"สวัสดี {person.Name} น้องหนู!";
    else if (person.Age < 60)
        return $"สวัสดีครับ/ค่ะ คุณ{person.Name}";
    else
        return $"สวัสดีครับ/ค่ะ ท่าน{person.Name}";
}
```

### F# vs Python

| Feature | F# | Python |
|---------|----|----|
| Type system | Static, strong | Dynamic |
| Performance | Near C# | Slower |
| Immutability | Default | Optional |
| Concurrency | Strong | GIL limitation |
| Syntax | Significant whitespace | Significant whitespace |
| Learning curve | Medium | Low |

```fsharp
// F# - statically typed, type inference
let sumOfSquares nums =
    nums
    |> List.filter (fun x -> x % 2 = 0)
    |> List.map (fun x -> x * x)
    |> List.sum

let result = sumOfSquares [1..10]
printfn "Sum of squares of evens: %d" result
```

```python
# Python equivalent
def sum_of_squares(nums):
    return sum(x**2 for x in nums if x % 2 == 0)

result = sum_of_squares(range(1, 11))
print(f"Sum of squares of evens: {result}")
```

### F# vs Haskell

| Feature | F# | Haskell |
|---------|----|----|
| Paradigm | Functional-first (multi) | Pure functional |
| Side effects | Allowed | Monadic (IO) |
| Laziness | Strict (opt-in lazy) | Lazy by default |
| Type classes | Limited | Powerful |
| .NET interop | Full | None |
| Learning curve | Medium | High |
| Industry adoption | Growing | Academic/Niche |

```fsharp
// F# - functional แต่ยังมี imperative ได้
let fibonacci n =
    let rec fib a b count =
        if count = 0 then a
        else fib b (a + b) (count - 1)
    fib 0 1 n

[0..10] |> List.map fibonacci |> printfn "Fibonacci: %A"
```

---

## 1.4 การติดตั้ง (Installation)

### 1.4.1 ติดตั้ง .NET SDK

**Windows:**
1. ไปที่ https://dotnet.microsoft.com/download
2. ดาวน์โหลด .NET 8 SDK (หรือเวอร์ชันล่าสุด)
3. รันไฟล์ installer
4. ตรวจสอบด้วย: `dotnet --version`

**macOS:**
```bash
# ใช้ Homebrew
brew install dotnet

# หรือดาวน์โหลดจากเว็บ Microsoft
# https://dotnet.microsoft.com/download

dotnet --version
```

**Linux (Ubuntu/Debian):**
```bash
# เพิ่ม Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
sudo apt-get update

# ติดตั้ง .NET SDK
sudo apt-get install -y dotnet-sdk-8.0

# ตรวจสอบ
dotnet --version
```

### 1.4.2 ติดตั้ง Visual Studio Code + Ionide

1. ดาวน์โหลด VS Code จาก https://code.visualstudio.com/
2. เปิด VS Code
3. กด `Ctrl+Shift+X` (Windows/Linux) หรือ `Cmd+Shift+X` (Mac)
4. ค้นหา "Ionide-fsharp"
5. ติดตั้ง extension ที่ชื่อ **Ionide-fsharp** โดย Ionide

**Extension ที่แนะนำเพิ่มเติม:**
- `Ionide-fsharp` - F# language support (สำคัญที่สุด)
- `C# Dev Kit` - .NET tools
- `GitLens` - Git integration
- `Bracket Pair Colorizer` - อ่านโค้ดง่ายขึ้น

### 1.4.3 ทางเลือกอื่น (Other Options)

**Visual Studio (Windows):**
- Visual Studio Community (ฟรี) รองรับ F# ได้ดีมาก
- ดาวน์โหลดจาก https://visualstudio.microsoft.com/

**JetBrains Rider:**
- IDE เชิงพาณิชย์ที่รองรับ F# ได้ดีเยี่ยม
- มีทั้ง paid และ free community edition

**Online:**
- https://try.fsharp.org/ - ทดลองใช้ออนไลน์
- https://dotnetfiddle.net/ - .NET fiddle

---

## 1.5 สร้างโปรเจกต์แรก (Creating Your First Project)

### 1.5.1 Console Application

```bash
# สร้าง directory และ cd เข้าไป
mkdir my-first-fsharp
cd my-first-fsharp

# สร้าง F# console project
dotnet new console -lang F#

# หรือระบุชื่อโปรเจกต์
dotnet new console -lang F# -n MyFirstProject
cd MyFirstProject
```

### 1.5.2 โครงสร้างโปรเจกต์ (Project Structure)

```
my-first-fsharp/
├── Program.fs          # ไฟล์โค้ดหลัก
├── my-first-fsharp.fsproj  # ไฟล์ project configuration
└── obj/               # Build artifacts (auto-generated)
```

### 1.5.3 ไฟล์ .fsproj

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <!-- Target framework: net8.0 = .NET 8 -->
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <!-- ใช้ Nullable reference types -->
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- ไฟล์ F# ต้องระบุลำดับที่นี่! (สำคัญมาก) -->
    <Compile Include="Program.fs" />
  </ItemGroup>

</Project>
```

**หมายเหตุสำคัญ**: ใน F# ลำดับของไฟล์ใน .fsproj มีความสำคัญมาก! ไฟล์ที่อยู่ด้านบนจะถูก compile ก่อน และสามารถใช้เฉพาะสิ่งที่ defined ในไฟล์ก่อนหน้าเท่านั้น

---

## 1.6 Hello World

### 1.6.1 Hello World แบบง่าย

เปิดไฟล์ `Program.fs` และแก้ไขเป็น:

```fsharp
// Program.fs
printfn "Hello, World!"
printfn "สวัสดีโลก!"
```

### 1.6.2 รันโปรแกรม

```bash
# รันโปรแกรม
dotnet run

# Output:
# Hello, World!
# สวัสดีโลก!
```

### 1.6.3 Hello World แบบสมบูรณ์

```fsharp
// Program.fs - Hello World แบบสมบูรณ์

// -----------------------------------------------
// ส่วน 1: Basic output
// -----------------------------------------------

// printfn = print with newline (เหมือน println ในภาษาอื่น)
printfn "Hello, World!"

// printf = print without newline
printf "Hello "
printf "World"
printfn ""  // newline

// -----------------------------------------------
// ส่วน 2: Variables and types
// -----------------------------------------------

let name = "F#"           // string
let version = 8           // int
let isAwesome = true      // bool

printfn "Language: %s" name
printfn "Version: %d" version
printfn "Is Awesome: %b" isAwesome

// -----------------------------------------------
// ส่วน 3: String interpolation
// -----------------------------------------------

let greeting = $"Hello from {name} {version}!"
printfn "%s" greeting

// -----------------------------------------------
// ส่วน 4: Function definition
// -----------------------------------------------

let greet personName =
    printfn "สวัสดี, %s!" personName

greet "Alice"
greet "Bob"
greet "ดิฉัน/ผม"

// -----------------------------------------------
// ส่วน 5: Working with lists
// -----------------------------------------------

let languages = ["F#"; "C#"; "Python"; "Haskell"]
printfn "\nภาษาโปรแกรมมิ่งที่ชอบ:"
languages |> List.iter (fun lang -> printfn "  - %s" lang)

// -----------------------------------------------
// ส่วน 6: Simple calculation
// -----------------------------------------------

let add x y = x + y
let multiply x y = x * y

let sum = add 10 20
let product = multiply 5 6

printfn "\n10 + 20 = %d" sum
printfn "5 × 6 = %d" product

printfn "\nโปรแกรมทำงานเสร็จแล้ว! ✓"
```

---

## 1.7 F# Interactive (FSI) - REPL

F# Interactive (fsi) เป็น REPL (Read-Eval-Print Loop) ที่ช่วยให้ทดลองโค้ดได้ทันที

### 1.7.1 เริ่มใช้งาน FSI

```bash
# เปิด F# Interactive
dotnet fsi

# จะเห็น prompt:
# >
```

### 1.7.2 การใช้งาน FSI

```fsharp
// ใน FSI พิมพ์ code แล้วกด Enter สองครั้ง หรือพิมพ์ ;; แล้วกด Enter

// ทดลอง expressions
> 1 + 2;;
val it: int = 3

> "Hello" + " " + "World";;
val it: string = "Hello World"

> let x = 42;;
val x: int = 42

> x * 2;;
val it: int = 84

// Define function
> let square n = n * n;;
val square: n: int -> int

> square 7;;
val it: int = 49

// Use pipe operator
> [1..10] |> List.map (fun x -> x * x) |> List.sum;;
val it: int = 385

// Exit FSI
> #quit;;
```

### 1.7.3 FSI Directives

```fsharp
// โหลดไฟล์ใน FSI
#load "MyFile.fs"

// โหลด NuGet package
#r "nuget: Newtonsoft.Json"

// ดูข้อมูล type
#t let x = 42

// Clear screen (ใน VS Code terminal)
// ใช้ Ctrl+L
```

### 1.7.4 ใช้ FSI ใน VS Code

ใน VS Code กับ Ionide:
1. เปิดไฟล์ `.fs` หรือ `.fsx`
2. เลือกโค้ดที่ต้องการรัน
3. กด `Alt+Enter` เพื่อส่งไปยัง FSI
4. หรือ `Ctrl+Alt+Enter` เพื่อรันทั้งไฟล์ใน FSI

---

## 1.8 F# Script Files (.fsx)

Script files ใช้สำหรับ scripting, prototyping, และ automation โดยไม่ต้องสร้าง project

### 1.8.1 สร้าง Script File

```bash
# สร้างไฟล์ script
touch hello.fsx

# รัน script
dotnet fsi hello.fsx
```

### 1.8.2 ตัวอย่าง Script File

```fsharp
// hello.fsx - F# Script File

// Script files สามารถใช้ #r เพื่อโหลด packages
// #r "nuget: Newtonsoft.Json"

printfn "F# Script is running!"

// Script arguments
let args = fsi.CommandLineArgs
printfn "Arguments: %A" args

// Script ทำงานได้เหมือน regular F# code
let fibonacci n =
    let rec fib a b count =
        if count = 0 then a
        else fib b (a + b) (count - 1)
    fib 0 1 n

printfn "Fibonacci numbers:"
[0..15] 
|> List.map fibonacci
|> List.iteri (fun i fib -> printfn "  fib(%d) = %d" i fib)
```

### 1.8.3 Script ที่ใช้ Arguments

```fsharp
// greet.fsx
let args = fsi.CommandLineArgs

match args with
| [| _; name |] -> 
    printfn "สวัสดี, %s!" name
| [| _; name; lang |] when lang = "en" ->
    printfn "Hello, %s!" name
| [| _ |] ->
    printfn "Usage: dotnet fsi greet.fsx <name>"
| _ ->
    printfn "สวัสดี, World!"
```

```bash
# รัน script พร้อม arguments
dotnet fsi greet.fsx Alice
dotnet fsi greet.fsx Bob en
```

---

## 1.9 โครงสร้างโปรเจกต์ที่ใหญ่ขึ้น (Larger Project Structure)

### 1.9.1 Multi-file Project

```
MyProject/
├── MyProject.fsproj
├── Domain/
│   ├── Types.fs         # Type definitions
│   └── Logic.fs         # Business logic
├── Infrastructure/
│   ├── Database.fs      # DB access
│   └── Http.fs          # HTTP client
├── Api/
│   └── Handlers.fs      # Request handlers
└── Program.fs           # Entry point
```

### 1.9.2 .fsproj สำหรับ Multi-file

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <AssemblyName>MyProject</AssemblyName>
    <RootNamespace>MyProject</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <!-- ลำดับสำคัญมาก! ไฟล์ด้านบนถูก compile ก่อน -->
    <Compile Include="Domain/Types.fs" />
    <Compile Include="Domain/Logic.fs" />
    <Compile Include="Infrastructure/Database.fs" />
    <Compile Include="Infrastructure/Http.fs" />
    <Compile Include="Api/Handlers.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>

  <ItemGroup>
    <!-- NuGet packages -->
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
  </ItemGroup>

</Project>
```

### 1.9.3 ตัวอย่าง Multi-file Project

```fsharp
// Domain/Types.fs
module Domain.Types

type Product = {
    Id: int
    Name: string
    Price: decimal
    Stock: int
}

type Order = {
    Id: int
    Products: Product list
    Total: decimal
}
```

```fsharp
// Domain/Logic.fs
module Domain.Logic

open Domain.Types

let calculateTotal products =
    products |> List.sumBy (fun p -> p.Price)

let createOrder id products =
    { Id = id
      Products = products
      Total = calculateTotal products }

let isInStock product =
    product.Stock > 0
```

```fsharp
// Program.fs
open Domain.Types
open Domain.Logic

let products = [
    { Id = 1; Name = "Widget A"; Price = 29.99M; Stock = 10 }
    { Id = 2; Name = "Widget B"; Price = 49.99M; Stock = 0 }
    { Id = 3; Name = "Widget C"; Price = 19.99M; Stock = 5 }
]

let inStockProducts = products |> List.filter isInStock
printfn "Products in stock: %d" (List.length inStockProducts)

let order = createOrder 1 inStockProducts
printfn "Order total: %M บาท" order.Total
```

---

## 1.10 การรันโปรแกรม (Running Programs)

### 1.10.1 คำสั่ง dotnet พื้นฐาน

```bash
# สร้างโปรเจกต์ใหม่
dotnet new console -lang F# -n MyApp

# รันโปรแกรม
dotnet run

# รันพร้อม arguments
dotnet run -- arg1 arg2

# Build โปรแกรม
dotnet build

# Build แบบ release
dotnet build -c Release

# Publish เพื่อ deploy
dotnet publish -c Release -o ./publish

# รันโปรแกรมที่ publish แล้ว
./publish/MyApp

# เพิ่ม NuGet package
dotnet add package Newtonsoft.Json

# ดู packages ที่ติดตั้ง
dotnet list package

# รัน tests
dotnet test

# Restore dependencies
dotnet restore
```

### 1.10.2 Hot Reload

```bash
# รันพร้อม hot reload (สำหรับ development)
dotnet watch run
```

---

## 1.11 F# Philosophy - หลักการสำคัญ

### 1.11.1 Immutability by Default

```fsharp
// ค่าใน F# เป็น immutable โดยค่าเริ่มต้น
let x = 5
// x <- 10  // Error: ไม่สามารถทำได้

// ต้องการ mutable ต้องระบุชัดเจน
let mutable y = 5
y <- 10  // OK
printfn "y = %d" y

// ข้อดีของ immutability:
// 1. ง่ายต่อการ debug
// 2. ปลอดภัยสำหรับ concurrent programming
// 3. Referential transparency
// 4. ง่ายต่อการทดสอบ
```

### 1.11.2 Expressions vs Statements

```fsharp
// ใน F# เกือบทุกอย่างเป็น expression (มีค่า return)

// if เป็น expression
let description = if true then "yes" else "no"
printfn "%s" description

// match เป็น expression
let grade score =
    match score with
    | s when s >= 90 -> "A"
    | s when s >= 80 -> "B"
    | s when s >= 70 -> "C"
    | s when s >= 60 -> "D"
    | _ -> "F"

printfn "Grade: %s" (grade 85)

// ทุก function return ค่าสุดท้ายโดยอัตโนมัติ
let add a b = a + b  // ไม่ต้องมี return keyword
```

### 1.11.3 Function Composition

```fsharp
// Functions เป็น first-class citizens
let double x = x * 2
let addOne x = x + 1
let square x = x * x

// Compose ด้วย >> (forward composition)
let doubleAndAddOne = double >> addOne
let doubleThenSquare = double >> square

printfn "%d" (doubleAndAddOne 5)   // (5*2)+1 = 11
printfn "%d" (doubleThenSquare 3)  // (3*2)^2 = 36

// Pipe operator |>
let result = 
    5
    |> double      // 10
    |> addOne      // 11
    |> square      // 121

printfn "Result: %d" result
```

### 1.11.4 Type Safety

```fsharp
// F# ใช้ type system เพื่อป้องกัน bug
// Single-case DU สำหรับ type safety
type CustomerId = CustomerId of int
type ProductId = ProductId of int

let getCustomer (CustomerId id) =
    printfn "Getting customer %d" id

let getProduct (ProductId id) =
    printfn "Getting product %d" id

let custId = CustomerId 42
let prodId = ProductId 100

getCustomer custId   // OK
getProduct prodId    // OK
// getCustomer prodId  // Error! Type mismatch - ป้องกัน bug!
```

### 1.11.5 Algebraic Type System

```fsharp
// Option type - แทนที่ null
type DatabaseResult =
    | Found of string
    | NotFound
    | Error of string

let lookupUser userId =
    match userId with
    | 1 -> Found "Alice"
    | 2 -> Found "Bob"
    | n when n < 0 -> Error "Invalid ID"
    | _ -> NotFound

let printUser userId =
    match lookupUser userId with
    | Found name -> printfn "User: %s" name
    | NotFound -> printfn "User not found"
    | Error msg -> printfn "Error: %s" msg

printUser 1
printUser 5
printUser -1
```

---

## 1.12 ตัวอย่างโปรแกรมสมบูรณ์ (Complete Example Programs)

### 1.12.1 โปรแกรมคำนวณ BMI

```fsharp
// bmi.fsx - โปรแกรมคำนวณ BMI
open System

let calculateBMI weight height =
    weight / (height * height)

let classifyBMI bmi =
    if bmi < 18.5 then "น้ำหนักน้อยกว่าเกณฑ์"
    elif bmi < 25.0 then "น้ำหนักปกติ"
    elif bmi < 30.0 then "น้ำหนักเกิน"
    else "โรคอ้วน"

let printBMIResult name weight height =
    let bmi = calculateBMI weight height
    let classification = classifyBMI bmi
    printfn "=== ผลลัพธ์สำหรับ %s ===" name
    printfn "น้ำหนัก: %.1f กก." weight
    printfn "ส่วนสูง: %.2f เมตร" height
    printfn "BMI: %.2f" bmi
    printfn "สถานะ: %s" classification
    printfn ""

// ทดสอบ
printBMIResult "คนที่ 1" 55.0 1.65
printBMIResult "คนที่ 2" 85.0 1.70
printBMIResult "คนที่ 3" 45.0 1.60
```

### 1.12.2 เครื่องคิดเลขอย่างง่าย

```fsharp
// calculator.fsx - เครื่องคิดเลข

type Operator = Add | Subtract | Multiply | Divide

let calculate op a b =
    match op with
    | Add -> Some (a + b)
    | Subtract -> Some (a - b)
    | Multiply -> Some (a * b)
    | Divide ->
        if b = 0.0 then None
        else Some (a / b)

let printResult op a b =
    let symbol =
        match op with
        | Add -> "+"
        | Subtract -> "-"
        | Multiply -> "×"
        | Divide -> "÷"
    
    match calculate op a b with
    | Some result -> 
        printfn "%.2f %s %.2f = %.2f" a symbol b result
    | None ->
        printfn "%.2f %s %.2f = ไม่สามารถหารด้วยศูนย์ได้!" a symbol b

printResult Add 10.0 5.0
printResult Subtract 20.0 8.0
printResult Multiply 4.0 7.0
printResult Divide 15.0 3.0
printResult Divide 10.0 0.0
```

### 1.12.3 โปรแกรมจัดการรายชื่อ

```fsharp
// contacts.fsx - โปรแกรมจัดการรายชื่อ

type Contact = {
    Name: string
    Phone: string
    Email: string
}

let contacts = [
    { Name = "สมชาย ใจดี"; Phone = "081-234-5678"; Email = "somchai@email.com" }
    { Name = "สมหญิง รักดี"; Phone = "082-345-6789"; Email = "somying@email.com" }
    { Name = "มานะ พยายาม"; Phone = "083-456-7890"; Email = "mana@email.com" }
    { Name = "มานี รักเรียน"; Phone = "084-567-8901"; Email = "manee@email.com" }
    { Name = "ปิติ ยินดี"; Phone = "085-678-9012"; Email = "piti@email.com" }
]

let printContact c =
    printfn "ชื่อ: %s | โทร: %s | อีเมล: %s" c.Name c.Phone c.Email

let searchByName name contacts =
    contacts |> List.filter (fun c -> c.Name.Contains(name))

let sortByName contacts =
    contacts |> List.sortBy (fun c -> c.Name)

printfn "=== รายชื่อทั้งหมด ==="
contacts |> List.iter printContact

printfn "\n=== ค้นหา 'มา' ==="
contacts 
|> searchByName "มา"
|> List.iter printContact

printfn "\n=== เรียงตามชื่อ ==="
contacts
|> sortByName
|> List.iter printContact

printfn "\nจำนวนรายชื่อทั้งหมด: %d คน" (List.length contacts)
```

---

## 1.13 ทรัพยากรการเรียนรู้ (Learning Resources)

### เว็บไซต์และเอกสาร
- **F# Official Docs**: https://learn.microsoft.com/en-us/dotnet/fsharp/
- **F# for Fun and Profit**: https://fsharpforfunandprofit.com/ (แนะนำอย่างยิ่ง!)
- **F# Foundation**: https://fsharp.org/
- **Try F#**: https://try.fsharp.org/
- **F# Cheatsheet**: https://fsprojects.github.io/fsharp-cheatsheet/

### หนังสือ
- "Get Programming with F#" by Isaac Abraham
- "Domain Modeling Made Functional" by Scott Wlaschin
- "Stylish F#" by Kit Eason

### Community
- **F# Slack**: https://fsharp.slack.com/
- **F# Discord**: ค้นหาใน Discord
- **Stack Overflow**: แท็ก `f#`

---

## สรุป Part 1

ในบทนี้เราได้เรียนรู้:
- ✅ F# คืออะไรและทำไมต้องใช้
- ✅ เปรียบเทียบ F# กับ C#, Python, Haskell
- ✅ การติดตั้ง .NET SDK และ VS Code + Ionide
- ✅ การสร้างโปรเจกต์ F# แรก
- ✅ Hello World program
- ✅ F# Interactive (fsi) - REPL
- ✅ Script files (.fsx)
- ✅ โครงสร้างโปรเจกต์
- ✅ หลักการ F#: functional-first, immutable, type-safe

**ใน Part 2** เราจะเรียนรู้ไวยากรณ์พื้นฐานของ F# อย่างละเอียด!

```fsharp
// preview ของ Part 2
let message = "พบกันใน Part 2!"
printfn "%s" message
```
