# Part 30 - การทำงานร่วมกับ .NET (Interop with .NET)

## บทนำ (Introduction)

F# ทำงานบน .NET runtime เดียวกันกับ C# และ VB.NET ทำให้สามารถใช้ library ทั้งหมดของ .NET ได้ รวมทั้งเรียก F# จาก C# ได้ด้วย

---

## 30.1 Using C# Libraries from F#

```fsharp
// ============ ใช้ C# Libraries ใน F# ============

// .NET Base Class Library (BCL) ใช้ได้โดยตรง
open System
open System.Collections.Generic
open System.IO
open System.Text
open System.Net.Http
open System.Threading.Tasks

// ============ StringBuilder (C# class) ============
let sb = StringBuilder()
sb.Append("Hello") |> ignore
sb.Append(", ") |> ignore
sb.Append("World") |> ignore
sb.AppendLine("!") |> ignore
sb.AppendFormat("F# Version: {0}", "8.0") |> ignore

printfn "%s" (sb.ToString())

// ============ Collections (C# generic) ============
let dict = Dictionary<string, int>()
dict.["one"] <- 1
dict.["two"] <- 2
dict.["three"] <- 3

for KeyValue(k, v) in dict do
    printfn "%s = %d" k v

let list = List<string>()
list.Add("apple")
list.Add("banana")
list.Add("cherry")
list.Sort()
printfn "Sorted: %A" (list |> Seq.toList)

// ============ Stack<T>, Queue<T> ============
let stack = Stack<int>()
stack.Push(1)
stack.Push(2)
stack.Push(3)
while stack.Count > 0 do
    printf "%d " (stack.Pop())
printfn ""

// ============ Predicate and Func ============
// F# lambda works as C# delegates
let numbers = List<int>([1; 2; 3; 4; 5; 6; 7; 8; 9; 10])
let evens = numbers.FindAll(Predicate<int>(fun x -> x % 2 = 0))
printfn "Evens: %A" (evens |> Seq.toList)

// ============ Events (C# style) ============
open System.ComponentModel

type DataService() =
    let propertyChanged = Event<PropertyChangedEventHandler, PropertyChangedEventArgs>()
    let mutable _value = ""
    
    interface INotifyPropertyChanged with
        [<CLIEvent>]
        member _.PropertyChanged = propertyChanged.Publish
    
    member _.Value
        with get() = _value
        and set(v) =
            _value <- v
            propertyChanged.Trigger(null, PropertyChangedEventArgs("Value"))

let service = DataService()
let handler = PropertyChangedEventHandler(fun _ args ->
    printfn "Property changed: %s" args.PropertyName)

(service :> INotifyPropertyChanged).PropertyChanged.AddHandler(handler)
service.Value <- "Hello"
```

---

## 30.2 NuGet Packages

```fsharp
// ============ ใช้ NuGet Packages ============

// ใน .fsproj เพิ่ม:
// <PackageReference Include="Newtonsoft.Json" Version="13.*" />
// <PackageReference Include="Serilog" Version="3.*" />

// #r directive สำหรับ scripting:
// #r "nuget: Newtonsoft.Json"
// #r "nuget: Serilog"

// ============ Newtonsoft.Json ============
(*
    open Newtonsoft.Json
    
    type Product = {
        [<JsonProperty("product_name")>]
        Name: string
        [<JsonProperty("price")>]
        Price: decimal
        [<JsonProperty("in_stock")>]
        InStock: bool
    }
    
    // Serialize
    let product = { Name = "Widget"; Price = 9.99m; InStock = true }
    let json = JsonConvert.SerializeObject(product, Formatting.Indented)
    printfn "JSON:\n%s" json
    
    // Deserialize
    let decoded = JsonConvert.DeserializeObject<Product>(json)
    printfn "Name: %s, Price: %m" decoded.Name decoded.Price
*)

// ============ ใช้ System.Text.Json (built-in) ============
open System.Text.Json
open System.Text.Json.Serialization

type Product = {
    [<JsonPropertyName("product_name")>]
    Name: string
    [<JsonPropertyName("price")>]
    Price: decimal
    [<JsonPropertyName("in_stock")>]
    InStock: bool
    [<JsonPropertyName("tags")>]
    Tags: string list
}

let options = JsonSerializerOptions(WriteIndented = true)
options.Converters.Add(JsonFSharpConverter())  // F# union support

let product = { 
    Name = "Widget A"
    Price = 9.99m
    InStock = true
    Tags = ["electronics"; "affordable"] 
}

let json = JsonSerializer.Serialize(product, options)
printfn "JSON:\n%s" json

let decoded = JsonSerializer.Deserialize<Product>(json, options)
printfn "Decoded: %s, $%.2f" decoded.Name decoded.Price

// ============ Serilog (popular logging) ============
(*
    open Serilog
    
    let log = LoggerConfiguration()
                .WriteTo.Console()
                .WriteTo.File("app.log")
                .MinimumLevel.Debug()
                .CreateLogger()
    
    log.Information("Application started")
    log.Debug("Debug message: {Value}", 42)
    log.Warning("Warning: {Message}", "Something might be wrong")
    log.Error("Error occurred: {Error}", "Division by zero")
    
    Log.CloseAndFlush()
*)
```

---

## 30.3 DateTime, TimeSpan

```fsharp
open System

// ============ DateTime ============
let now = DateTime.Now
let utcNow = DateTime.UtcNow
let today = DateTime.Today

printfn "Now: %A" now
printfn "UTC: %A" utcNow
printfn "Today: %A" today

// สร้าง DateTime
let birthday = DateTime(1990, 5, 15, 10, 30, 0)
printfn "Birthday: %A" birthday

// Properties
printfn "Year: %d, Month: %d, Day: %d" birthday.Year birthday.Month birthday.Day
printfn "Hour: %d, Minute: %d" birthday.Hour birthday.Minute
printfn "DayOfWeek: %A" birthday.DayOfWeek
printfn "DayOfYear: %d" birthday.DayOfYear

// Add/Subtract
let nextWeek = now.AddDays(7.0)
let lastYear = now.AddYears(-1)
let twoHoursLater = now.AddHours(2.0)

printfn "Next week: %A" nextWeek
printfn "Two hours later: %A" twoHoursLater

// ============ TimeSpan ============
let age = now - birthday
printfn "Age: %.1f years" (age.TotalDays / 365.25)
printfn "Age in days: %.0f" age.TotalDays

let duration = TimeSpan(1, 30, 45)   // 1 hour, 30 min, 45 sec
printfn "Duration: %A" duration
printfn "Total minutes: %.2f" duration.TotalMinutes

let ts1 = TimeSpan.FromHours(2.5)
let ts2 = TimeSpan.FromMinutes(90.0)
printfn "ts1 + ts2 = %A" (ts1 + ts2)

// ============ DateTimeOffset ============
let dtOffset = DateTimeOffset.Now
printfn "DateTimeOffset: %A" dtOffset
printfn "Offset: %A" dtOffset.Offset

// UTC conversion
let utcTime = DateTimeOffset.UtcNow
let bangkokTime = utcTime.ToOffset(TimeSpan.FromHours(7.0))
printfn "Bangkok time: %A" bangkokTime

// ============ DateOnly, TimeOnly (.NET 6+) ============
let dateOnly = DateOnly(2024, 3, 15)
let timeOnly = TimeOnly(14, 30, 0)
printfn "Date: %A" dateOnly
printfn "Time: %A" timeOnly

// ============ Formatting ============
printfn "Formatted: %s" (now.ToString("yyyy-MM-dd HH:mm:ss"))
printfn "Short date: %s" (now.ToString("d"))
printfn "Long date: %s" (now.ToString("D"))
printfn "ISO 8601: %s" (now.ToString("o"))

// Parsing
let parsed = DateTime.Parse("2024-03-15 14:30:00")
let parsed2 = DateTime.ParseExact("15/03/2024", "dd/MM/yyyy", null)
printfn "Parsed: %A" parsed
printfn "Parsed2: %A" parsed2

// ============ Working with dates ============
let isWeekend (dt: DateTime) = 
    dt.DayOfWeek = DayOfWeek.Saturday || dt.DayOfWeek = DayOfWeek.Sunday

let nextWorkday (dt: DateTime) =
    let mutable d = dt.AddDays(1.0)
    while isWeekend d do
        d <- d.AddDays(1.0)
    d

printfn "Is today weekend: %b" (isWeekend today)
printfn "Next workday: %A" (nextWorkday today)
```

---

## 30.4 Guid

```fsharp
open System

// ============ Guid ============
// Globally Unique Identifier

// สร้าง new Guid
let id1 = Guid.NewGuid()
let id2 = Guid.NewGuid()

printfn "Guid 1: %A" id1
printfn "Guid 2: %A" id2
printfn "Equal: %b" (id1 = id2)  // false (almost certainly)

// Parse from string
let guidStr = "550e8400-e29b-41d4-a716-446655440000"
let parsed = Guid.Parse(guidStr)
printfn "Parsed: %A" parsed

// TryParse
match Guid.TryParse("not-a-guid") with
| true, g -> printfn "Valid: %A" g
| false, _ -> printfn "Invalid GUID"

// Empty GUID
let empty = Guid.Empty
printfn "Empty: %A" empty
printfn "IsEmpty: %b" (empty = Guid.Empty)

// Formatting
printfn "D format: %s" (id1.ToString("D"))   // xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
printfn "N format: %s" (id1.ToString("N"))   // 32 hex digits no dashes
printfn "B format: %s" (id1.ToString("B"))   // {xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx}
printfn "P format: %s" (id1.ToString("P"))   // (xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx)

// ============ ตัวอย่างใช้งาน ============
type Entity = {
    Id: Guid
    Name: string
    CreatedAt: DateTime
}

let createEntity name = {
    Id = Guid.NewGuid()
    Name = name
    CreatedAt = DateTime.UtcNow
}

let entity1 = createEntity "Product A"
let entity2 = createEntity "Product B"
printfn "Entity 1 ID: %s" (entity1.Id.ToString("D"))
printfn "Entity 2 ID: %s" (entity2.Id.ToString("D"))
```

---

## 30.5 Uri

```fsharp
open System

// ============ Uri ============
let uri1 = Uri("https://api.example.com/v1/users?page=1&limit=20")

printfn "Scheme: %s" uri1.Scheme      // https
printfn "Host: %s" uri1.Host          // api.example.com
printfn "Path: %s" uri1.AbsolutePath  // /v1/users
printfn "Query: %s" uri1.Query        // ?page=1&limit=20
printfn "Port: %d" uri1.Port          // 443

// UriBuilder
let builder = UriBuilder("https", "api.example.com")
builder.Path <- "/v1/products"
builder.Query <- "category=electronics&inStock=true"
let builtUri = builder.Uri
printfn "Built URI: %A" builtUri

// Parse query string
let parseQueryString (query: string) =
    query.TrimStart('?')
    |> (fun s -> s.Split('&'))
    |> Array.choose (fun pair ->
        match pair.Split('=') with
        | [| key; value |] -> Some (Uri.UnescapeDataString(key), Uri.UnescapeDataString(value))
        | _ -> None)
    |> Map.ofArray

let queryParams = parseQueryString uri1.Query
printfn "page: %s" queryParams.["page"]
printfn "limit: %s" queryParams.["limit"]

// Relative URIs
let baseUri = Uri("https://example.com/api/")
let relativeUri = Uri("users/123", UriKind.Relative)
let absoluteUri = Uri(baseUri, relativeUri)
printfn "Absolute: %A" absoluteUri

// Encode/Decode
let encoded = Uri.EscapeDataString("hello world & more!")
let decoded = Uri.UnescapeDataString(encoded)
printfn "Encoded: %s" encoded
printfn "Decoded: %s" decoded

// ============ HTTP Client ============
open System.Net.Http
open System.Threading

let fetchAsync (url: string) =
    async {
        use client = new HttpClient()
        let! response = client.GetAsync(url) |> Async.AwaitTask
        response.EnsureSuccessStatusCode() |> ignore
        return! response.Content.ReadAsStringAsync() |> Async.AwaitTask
    }

// Simulated (won't actually run in basic example)
(*
let result = fetchAsync "https://api.github.com/repos/dotnet/fsharp" |> Async.RunSynchronously
printfn "Response length: %d" result.Length
*)
```

---

## 30.6 File and Directory Operations

```fsharp
open System.IO

// ============ File Operations ============
let tempPath = Path.GetTempPath()
let testFile = Path.Combine(tempPath, "fsharp_test.txt")

// Write file
File.WriteAllText(testFile, "Hello, F#!\nLine 2\nLine 3")
printfn "File written: %s" testFile

// Read file
let content = File.ReadAllText(testFile)
printfn "Content: %s" content

// Read lines
let lines = File.ReadAllLines(testFile)
printfn "Lines count: %d" lines.Length
lines |> Array.iteri (fun i line -> printfn "Line %d: %s" (i+1) line)

// Append
File.AppendAllText(testFile, "\nAppended line")
printfn "After append: %s" (File.ReadAllText(testFile))

// Copy/Move
let copyPath = testFile.Replace(".txt", "_copy.txt")
File.Copy(testFile, copyPath, true)
printfn "Copied to: %s" copyPath

// File info
let fi = FileInfo(testFile)
printfn "Size: %d bytes" fi.Length
printfn "Created: %A" fi.CreationTime
printfn "Modified: %A" fi.LastWriteTime

// Delete
File.Delete(copyPath)
printfn "Deleted copy"

// Exists
printfn "Exists: %b" (File.Exists(testFile))
printfn "Copy exists: %b" (File.Exists(copyPath))

// ============ Directory Operations ============
let testDir = Path.Combine(tempPath, "fsharp_test_dir")

// Create
if not (Directory.Exists(testDir)) then
    Directory.CreateDirectory(testDir) |> ignore
    printfn "Directory created: %s" testDir

// List files
let files = Directory.GetFiles(testDir, "*.*", SearchOption.AllDirectories)
printfn "Files in dir: %d" files.Length

// List directories
let dirs = Directory.GetDirectories(testDir)
printfn "Subdirs: %d" dirs.Length

// Directory info
let di = DirectoryInfo(testDir)
printfn "Dir name: %s" di.Name
printfn "Full path: %s" di.FullName

// ============ Path operations ============
let filePath = "/home/user/documents/report.pdf"
printfn "Extension: %s" (Path.GetExtension(filePath))      // .pdf
printfn "Filename: %s" (Path.GetFileName(filePath))        // report.pdf
printfn "Without ext: %s" (Path.GetFileNameWithoutExtension(filePath))  // report
printfn "Directory: %s" (Path.GetDirectoryName(filePath))  // /home/user/documents

// Combine paths (cross-platform)
let combinedPath = Path.Combine("home", "user", "documents", "file.txt")
printfn "Combined: %s" combinedPath

// ============ StreamReader/Writer ============
let streamFile = Path.Combine(tempPath, "stream_test.txt")

// Write with StreamWriter
use writer = new StreamWriter(streamFile)
for i in 1..5 do
    writer.WriteLine(sprintf "Row %d, Value %d" i (i * i))
writer.Flush()

// Read with StreamReader
use reader = new StreamReader(streamFile)
let mutable line = reader.ReadLine()
while line <> null do
    printfn "Read: %s" line
    line <- reader.ReadLine()

// Cleanup
File.Delete(testFile)
File.Delete(streamFile)
try Directory.Delete(testDir, true) with _ -> ()
```

---

## 30.7 Stream Operations

```fsharp
open System.IO
open System.IO.Compression

// ============ Memory Stream ============
let ms = new MemoryStream()
let writer = new StreamWriter(ms)
writer.Write("Hello, Stream!")
writer.Flush()

ms.Position <- 0L
let reader = new StreamReader(ms)
let text = reader.ReadToEnd()
printfn "From MemoryStream: %s" text

// ============ MemoryStream as byte array ============
let data = System.Text.Encoding.UTF8.GetBytes("Binary data here")
let ms2 = new MemoryStream(data)
printfn "Stream length: %d" ms2.Length

let bytes = ms2.ToArray()
printfn "Bytes: %A" (bytes |> Array.take 6)

// ============ Async Stream Operations ============
let readFileAsync (path: string) =
    async {
        use fs = new FileStream(path, FileMode.Open, FileAccess.Read, FileShare.Read, 4096, true)
        use reader = new StreamReader(fs)
        return! reader.ReadToEndAsync() |> Async.AwaitTask
    }

let writeFileAsync (path: string) (content: string) =
    async {
        use fs = new FileStream(path, FileMode.Create, FileAccess.Write, FileShare.None, 4096, true)
        use writer = new StreamWriter(fs)
        do! writer.WriteAsync(content) |> Async.AwaitTask
    }

// ============ Compression ============
let compressData (data: byte[]) : byte[] =
    use ms = new MemoryStream()
    use gz = new GZipStream(ms, CompressionMode.Compress)
    gz.Write(data, 0, data.Length)
    gz.Close()
    ms.ToArray()

let decompressData (compressed: byte[]) : byte[] =
    use ms = new MemoryStream(compressed)
    use gz = new GZipStream(ms, CompressionMode.Decompress)
    use output = new MemoryStream()
    gz.CopyTo(output)
    output.ToArray()

let original = System.Text.Encoding.UTF8.GetBytes(String.replicate 100 "Hello, World! ")
let compressed = compressData original
let decompressed = decompressData compressed

printfn "Original size: %d bytes" original.Length
printfn "Compressed size: %d bytes" compressed.Length
printfn "Compression ratio: %.1f%%" (float compressed.Length / float original.Length * 100.0)
printfn "Decompressed matches: %b" (original = decompressed)

// ============ BinaryReader/Writer ============
let binaryFile = Path.GetTempFileName()
use bw = new BinaryWriter(File.Open(binaryFile, FileMode.Create))
bw.Write(42)           // int
bw.Write(3.14)         // double
bw.Write("hello")      // string
bw.Write(true)         // bool
bw.Close()

use br = new BinaryReader(File.Open(binaryFile, FileMode.Open))
let i = br.ReadInt32()
let d = br.ReadDouble()
let s = br.ReadString()
let b = br.ReadBoolean()
br.Close()

printfn "Read: %d, %f, %s, %b" i d s b
File.Delete(binaryFile)
```

---

## 30.8 LINQ from F#

```fsharp
open System.Linq
open System.Collections.Generic

// ============ LINQ ใน F# ============
// F# มี Seq, List, Array modules แต่ยังใช้ LINQ ได้

let numbers = [| 1; 2; 3; 4; 5; 6; 7; 8; 9; 10 |]

// LINQ methods
let evenNumbers = numbers.Where(fun x -> x % 2 = 0).ToArray()
let doubled = numbers.Select(fun x -> x * 2).ToList()
let sum = numbers.Sum()
let max = numbers.Max()
let first = numbers.First(fun x -> x > 5)
let firstOrDefault = numbers.FirstOrDefault(fun x -> x > 100)

printfn "Evens: %A" evenNumbers
printfn "Sum: %d, Max: %d" sum max
printfn "First > 5: %d" first
printfn "First > 100: %d" firstOrDefault   // 0 (default)

// ============ F# Query Expression ============
// ใช้ query { } syntax (เหมือน LINQ)

let data = [ 
    ("Alice", 30, "Engineering")
    ("Bob", 25, "Marketing")
    ("Charlie", 35, "Engineering")
    ("Diana", 28, "Finance")
    ("Eve", 32, "Marketing")
]

let engineers = query {
    for (name, age, dept) in data do
    where (dept = "Engineering")
    sortBy age
    select name
} |> Seq.toList

printfn "Engineers: %A" engineers

let avgAgeByDept = query {
    for (name, age, dept) in data do
    groupBy dept into g
    select (g.Key, g.Average(fun (_, a, _) -> float a))
} |> Seq.toList

printfn "Avg age by dept: %A" avgAgeByDept

// ============ IEnumerable extensions ============
// F# lists/seqs work with LINQ
let fsharpList = [1; 2; 3; 4; 5]
let linqResult = fsharpList.Where(fun x -> x > 3).Sum()
printfn "LINQ on F# list: %d" linqResult   // 9

// GroupBy with LINQ
let words = ["apple"; "ant"; "bear"; "bee"; "cat"]
let grouped = words.GroupBy(fun w -> w.[0]).ToList()
for group in grouped do
    printfn "Letter '%c': %A" group.Key (group.ToList())

// Zip (LINQ)
let names = ["Alice"; "Bob"; "Charlie"]
let scores = [95; 87; 92]
let combined = names.Zip(scores, fun n s -> sprintf "%s:%d" n s).ToList()
printfn "Zipped: %A" combined

// ============ Parallel LINQ (PLINQ) ============
let bigArray = Array.init 1_000_000 id

let sw = System.Diagnostics.Stopwatch.StartNew()
let seqResult = bigArray.Where(fun x -> x % 2 = 0).Sum()
sw.Stop()
printfn "Sequential: %d ms" sw.ElapsedMilliseconds

sw.Restart()
let parallelResult = bigArray.AsParallel().Where(fun x -> x % 2 = 0).Sum()
sw.Stop()
printfn "Parallel: %d ms" sw.ElapsedMilliseconds

printfn "Results equal: %b" (seqResult = parallelResult)
```

---

## 30.9 Converting Between F# and C# Types

```fsharp
// ============ Converting F# <-> C# Types ============

// ============ Option <-> Nullable ============
open System

// F# Option to .NET Nullable
let optionToNullable : 'T option -> Nullable<'T> = function
    | Some v -> Nullable(v)
    | None -> Nullable()

// .NET Nullable to F# Option
let nullableToOption (n: Nullable<'T>) : 'T option =
    if n.HasValue then Some n.Value else None

let someValue = Some 42
let noneValue : int option = None

let nullable1 = optionToNullable someValue    // Nullable<int>(42)
let nullable2 = optionToNullable noneValue    // Nullable<int>()

printfn "nullable1 HasValue: %b, Value: %d" nullable1.HasValue nullable1.Value
printfn "nullable2 HasValue: %b" nullable2.HasValue

let option1 = nullableToOption nullable1   // Some 42
let option2 = nullableToOption nullable2   // None
printfn "option1: %A" option1
printfn "option2: %A" option2

// ============ F# List <-> C# List ============
let fsharpList = [1; 2; 3; 4; 5]
let csharpList = System.Collections.Generic.List<int>(fsharpList)
let backToFsharp = csharpList |> Seq.toList

printfn "C# List: %A" (csharpList |> Seq.toList)
printfn "Back to F#: %A" backToFsharp

// ============ F# Array <-> C# Array ============
let fsharpArray = [| 1; 2; 3 |]
let dotnetArray : int[] = fsharpArray   // F# arrays ARE .NET arrays!

// ============ F# Map <-> C# Dictionary ============
let fsharpMap = Map.ofList [("a", 1); ("b", 2); ("c", 3)]
let csharpDict = System.Collections.Generic.Dictionary<string, int>(fsharpMap)
let backToMap = csharpDict |> Seq.map (fun kv -> (kv.Key, kv.Value)) |> Map.ofSeq
printfn "Map back: %A" backToMap

// ============ F# Seq <-> IEnumerable ============
let fsharpSeq = seq { 1..10 }
let ienumerable : System.Collections.Generic.IEnumerable<int> = fsharpSeq  // direct cast
let seqFromEnum = ienumerable |> Seq.filter (fun x -> x % 2 = 0) |> Seq.toList
printfn "From IEnumerable: %A" seqFromEnum

// ============ F# Tuple <-> ValueTuple ============
let fsharpTuple = (1, "hello", true)
let valueTuple = struct (1, "hello", true)   // C# ValueTuple

// ============ F# Function <-> C# Delegate ============
let fsharpFunc : int -> int -> int = fun x y -> x + y

// Convert to C# Func<>
let csharpFunc : System.Func<int, int, int> = 
    System.Func<int, int, int>(fun x y -> fsharpFunc x y)

printfn "Delegate result: %d" (csharpFunc.Invoke(3, 4))

// ============ F# Record <-> C# POCO ============
// [<CLIMutable>] allows C# code to create record with reflection
[<CLIMutable>]
type UserRecord = {
    Id: int
    Name: string
    Email: string
}

// This record can now be created by C# code like:
// var user = new UserRecord { Id = 1, Name = "Alice", Email = "alice@example.com" };
```

---

## 30.10 Nullable Types

```fsharp
open System

// ============ Nullable<T> ============
let n1 = Nullable<int>(42)
let n2 = Nullable<int>()  // null

// Check and access
if n1.HasValue then
    printfn "Value: %d" n1.Value
else
    printfn "No value"

// GetValueOrDefault
let v1 = n1.GetValueOrDefault()    // 42
let v2 = n2.GetValueOrDefault()    // 0 (default)
let v3 = n2.GetValueOrDefault(99)  // 99 (custom default)

printfn "v1=%d, v2=%d, v3=%d" v1 v2 v3

// ============ Nullable operators in F# ============
// ใช้ ?= ?<> ?< ?> etc.
let x : Nullable<int> = Nullable(5)
let y : Nullable<int> = Nullable(10)
let z : Nullable<int> = Nullable()

// Arithmetic with Nullable
let sum = 
    if x.HasValue && y.HasValue then Nullable(x.Value + y.Value)
    else Nullable<int>()

printfn "sum: %A" sum

// ============ FSharp.Core Nullable helpers ============
module Nullable =
    let orDefault defaultValue (n: Nullable<'T>) =
        if n.HasValue then n.Value else defaultValue
    
    let map f (n: Nullable<'T>) : Nullable<'U> =
        if n.HasValue then Nullable(f n.Value) else Nullable()
    
    let bind f (n: Nullable<'T>) : Nullable<'U> =
        if n.HasValue then f n.Value else Nullable()
    
    let toOption (n: Nullable<'T>) =
        if n.HasValue then Some n.Value else None
    
    let ofOption = function
        | Some v -> Nullable(v)
        | None -> Nullable()

let doubled = Nullable.map ((*) 2) x
printfn "Doubled nullable: %A" doubled

let stringed = Nullable.map (fun i -> sprintf "Value: %d" i) x
printfn "To string: %s" (Nullable.orDefault "No value" stringed)

// ============ null vs Nullable ============
// null - สำหรับ reference types
// Nullable<T> - สำหรับ value types

let nullString : string = null   // OK for reference type
// let nullInt : int = null       // ERROR! int is value type

let nullableInt : int? = 5       // int? = Nullable<int>
let nullableEmpty : int? = null  // OK with int?

// ============ Database patterns ============
type DbUser = {
    Id: int
    Name: string
    Age: Nullable<int>        // nullable column
    Email: Nullable<string>   // nullable reference
}

let dbUsers = [
    { Id = 1; Name = "Alice"; Age = Nullable(30); Email = Nullable("alice@co.com") }
    { Id = 2; Name = "Bob"; Age = Nullable<int>(); Email = Nullable("bob@co.com") }
    { Id = 3; Name = "Charlie"; Age = Nullable(25); Email = Nullable<string>() }
]

for user in dbUsers do
    let age = Nullable.orDefault -1 user.Age
    let email = if user.Email.HasValue then user.Email.Value else "N/A"
    printfn "  %s: age=%d, email=%s" user.Name age email
```

---

## 30.11 Calling F# from C#

```fsharp
// ============ F# ที่เรียกจาก C# ============

// ============ [<CompiledName>] ============
// เปลี่ยนชื่อที่ compile เป็น .NET assembly

module MathOperations =
    [<CompiledName("Add")>]
    let add x y = x + y
    
    [<CompiledName("Multiply")>]
    let multiply x y = x * y
    
    [<CompiledName("Factorial")>]
    let rec factorial n =
        if n <= 1 then 1
        else n * factorial (n - 1)

// C# สามารถเรียก:
// MathOperations.Add(3, 4)        // ไม่ใช่ MathOperations.add(3, 4)
// MathOperations.Multiply(2, 5)
// MathOperations.Factorial(5)

// ============ [<CLIMutable>] ============
// ทำให้ record สามารถสร้างด้วย reflection (C# object initializer)

[<CLIMutable>]
type PersonDto = {
    FirstName: string
    LastName: string
    Age: int
    Email: string
}

// C# สามารถใช้:
// var person = new PersonDto { FirstName = "Alice", LastName = "Smith", Age = 30 };
// หรือ: var person = Activator.CreateInstance<PersonDto>();

// ============ Using C# Interfaces ============
[<Interface>]
type ICalculator =
    abstract Add: int -> int -> int
    abstract Subtract: int -> int -> int
    abstract Multiply: int -> int -> int

// Implement C# interface ใน F#
type Calculator() =
    interface ICalculator with
        member _.Add x y = x + y
        member _.Subtract x y = x - y
        member _.Multiply x y = x * y
    
    // Extra method not in interface
    member _.Factorial n =
        let rec fact n = if n <= 1 then 1 else n * fact (n - 1)
        fact n

let calc : ICalculator = Calculator()
printfn "Add: %d" (calc.Add 5 3)
printfn "Subtract: %d" (calc.Subtract 10 4)

// ============ AbstractClass ============
[<AbstractClass>]
type Animal(name: string) =
    member _.Name = name
    abstract member Speak: unit -> string
    
    member this.ToString() = sprintf "%s says %s" this.Name (this.Speak())

type Dog(name: string) =
    inherit Animal(name)
    override _.Speak() = "Woof!"

type Cat(name: string) =
    inherit Animal(name)
    override _.Speak() = "Meow!"

let animals : Animal list = [Dog("Rex"); Cat("Whiskers"); Dog("Buddy")]
animals |> List.iter (fun a -> printfn "%s" (a.ToString()))

// ============ F# Unions from C# ============
// C# ต้องใช้ pattern matching API

type Result<'T> =
    | Success of value: 'T
    | Failure of message: string

// C# ใช้ได้แต่ต้องรู้ว่า DU ถูก compile เป็น class hierarchy
(*
    // C# code:
    if (result is Result.Success<int> success)
    {
        Console.WriteLine(success.value);
    }
    else if (result is Result.Failure<int> failure)
    {
        Console.WriteLine(failure.message);
    }
*)

// ============ Extension Methods ============
[<System.Runtime.CompilerServices.Extension>]
module StringExtensions =
    [<System.Runtime.CompilerServices.Extension>]
    let ToPascalCase (s: string) =
        if String.IsNullOrEmpty(s) then s
        else
            s.Split([|' '; '_'; '-'|], StringSplitOptions.RemoveEmptyEntries)
            |> Array.map (fun word -> 
                if word.Length > 0 then
                    string (Char.ToUpper(word.[0])) + word.[1..].ToLower()
                else word)
            |> String.concat ""

let result2 = StringExtensions.ToPascalCase("hello world")
printfn "PascalCase: %s" result2   // HelloWorld

// C# extension method syntax:
// using FSharpModule;
// var result = "hello world".ToPascalCase();
```

---

## 30.12 F# Module Patterns for Interop

```fsharp
// ============ Module Patterns ============

// ============ AutoOpen ============
// Automatically open module when assembly is referenced

[<AutoOpen>]
module CoreExtensions =
    let inline (++) (a: 'T seq) (b: 'T seq) = Seq.append a b
    
    type System.String with
        member this.IsNullOrEmpty = String.IsNullOrEmpty(this)
        member this.TrimEnd () = this.TrimEnd()

// ============ Module with explicit namespace ============
namespace MyLibrary

[<AutoOpen>]
module PublicAPI =
    let greet name = sprintf "Hello, %s!" name
    let version = "1.0.0"

namespace MyLibrary.Internal

module PrivateImplementation =
    let helperFunction x = x * 2

// Back to root namespace
namespace global

module Example =
    let useLibrary () =
        printfn "%s" (MyLibrary.PublicAPI.greet "World")
        printfn "Version: %s" MyLibrary.PublicAPI.version

// ============ Interfaces สำหรับ DI ============
[<Interface>]
type IEmailService =
    abstract SendEmail: to_:string -> subject:string -> body:string -> Async<Result<unit, string>>

[<Interface>]
type IUserRepository =
    abstract GetUser: id:int -> Async<PersonDto option>
    abstract SaveUser: user:PersonDto -> Async<Result<int, string>>

// Implementations
type MockEmailService() =
    interface IEmailService with
        member _.SendEmail to_ subject body =
            async {
                printfn "Sending email to %s: %s" to_ subject
                return Ok ()
            }

type InMemoryUserRepo(users: System.Collections.Generic.Dictionary<int, PersonDto>) =
    interface IUserRepository with
        member _.GetUser id =
            async {
                match users.TryGetValue(id) with
                | true, user -> return Some user
                | false, _ -> return None
            }
        
        member _.SaveUser user =
            async {
                let id = users.Count + 1
                users.[id] <- { user with Age = user.Age }  // simplified
                return Ok id
            }

// ============ ตัวอย่าง: Service Layer ============
type UserService(repo: IUserRepository, email: IEmailService) =
    
    member _.RegisterUser (name: string) (userEmail: string) =
        async {
            let user = { FirstName = name; LastName = ""; Age = 0; Email = userEmail }
            match! repo.SaveUser user with
            | Ok id ->
                let! emailResult = email.SendEmail userEmail "Welcome!" (sprintf "Welcome, %s!" name)
                match emailResult with
                | Ok () -> return Ok id
                | Error e -> return Error (sprintf "Saved but email failed: %s" e)
            | Error e ->
                return Error e
        }

let users = System.Collections.Generic.Dictionary<int, PersonDto>()
let svc = UserService(InMemoryUserRepo(users), MockEmailService())

let registrationResult = 
    svc.RegisterUser "Alice" "alice@example.com" 
    |> Async.RunSynchronously

printfn "Registration result: %A" registrationResult
```

---

## 30.13 Async Interop

```fsharp
open System.Threading.Tasks

// ============ Task <-> Async ============

// C# Task -> F# Async
let asyncFromTask (task: Task<'T>) : Async<'T> =
    Async.AwaitTask task

let asyncFromUnitTask (task: Task) : Async<unit> =
    Async.AwaitTask task

// F# Async -> C# Task
let taskFromAsync (comp: Async<'T>) : Task<'T> =
    Async.StartAsTask comp

// ============ Mixed async patterns ============
let fetchData (url: string) : Async<string> =
    async {
        use client = new System.Net.Http.HttpClient()
        // Note: AwaitTask converts C# Task to F# Async
        let! response = client.GetAsync(url) |> Async.AwaitTask
        return! response.Content.ReadAsStringAsync() |> Async.AwaitTask
    }

// Use in Task-based C# code
let fetchDataAsTask url : Task<string> = 
    fetchData url |> Async.StartAsTask

// ============ ValueTask interop ============
let fromValueTask (vt: ValueTask<'T>) : Async<'T> =
    vt.AsTask() |> Async.AwaitTask

// ============ CancellationToken ============
open System.Threading

let cancellableOperation (ct: CancellationToken) =
    async {
        use _ = ct.Register(fun () -> printfn "Cancelled!")
        
        for i in 1..10 do
            ct.ThrowIfCancellationRequested()
            do! Async.Sleep(100)
            printfn "Step %d" i
    }

let cts = new CancellationTokenSource(500)  // Cancel after 500ms
try
    cancellableOperation cts.Token |> Async.RunSynchronously
with
| :? OperationCanceledException ->
    printfn "Operation was cancelled"

// ============ Parallel operations ============
let parallelFetch urls =
    urls
    |> List.map (fun url -> 
        async { return sprintf "Data from %s" url })
    |> Async.Parallel

let results = 
    parallelFetch ["url1"; "url2"; "url3"]
    |> Async.RunSynchronously

printfn "Parallel results: %A" results
```

---

## สรุป (Summary)

```
F# กับ .NET Interop:

ใช้ C# Libraries:
- BCL (System.*) ใช้ได้โดยตรง
- NuGet packages ผ่าน PackageReference
- Delegates/Events รองรับ

DateTime/TimeSpan:
- DateTime.Now, UtcNow, Today
- AddDays, AddHours, AddYears
- Format: ToString("yyyy-MM-dd")

Guid/Uri:
- Guid.NewGuid() สำหรับ unique IDs
- Uri/UriBuilder สำหรับ URL handling

File/Directory:
- System.IO.File, Directory, Path
- StreamReader/Writer
- Compression

LINQ:
- F# sequences work with LINQ
- query { } syntax
- Parallel LINQ (PLINQ)

Type Conversions:
- Option <-> Nullable
- F# List <-> C# List
- F# Map <-> Dictionary

Calling F# from C#:
- [<CompiledName>]: rename for C# consumers
- [<CLIMutable>]: record with C# initializer
- Interface implementations
- Extension methods

Async Interop:
- Task <-> Async
- CancellationToken
- Parallel.Async
```

---

*จบ Part 30 - การทำงานร่วมกับ .NET (Interop with .NET)*
