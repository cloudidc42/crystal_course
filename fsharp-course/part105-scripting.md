# Part 105 - F# Scripting และ Automation

## บทนำ (Introduction)

F# script files (.fsx) ทำให้เราสามารถเขียน F# code แบบ scripting โดยไม่ต้องสร้าง full project มันเหมาะสำหรับ:

- Automation scripts
- Data processing
- Build scripts
- Quick prototyping
- DevOps tasks

---

## 1. F# Script Files (.fsx)

### Script พื้นฐาน

```fsharp
// hello.fsx
printfn "Hello from F# script!"

let greet name = printfn "สวัสดี, %s!" name

greet "สมชาย"
greet "สมหญิง"

// รัน: dotnet fsi hello.fsx
```

### Script กับ Arguments

```fsharp
// args.fsx
let args = fsi.CommandLineArgs

printfn "Script: %s" args.[0]

if args.Length > 1 then
    for i in 1 .. args.Length - 1 do
        printfn "Arg %d: %s" i args.[i]
else
    printfn "ไม่มี arguments"

// รัน: dotnet fsi args.fsx arg1 arg2 arg3
```

### Interactive Scripting

```fsharp
// interactive.fsx

// ใน F# Interactive (FSI) สามารถรัน code ทีละบรรทัดได้
// เปิด: dotnet fsi

// หรือรัน script ทั้งไฟล์: dotnet fsi myscript.fsx
```

---

## 2. #r สำหรับ NuGet Packages

### ติดตั้ง Package จาก NuGet

```fsharp
// packages.fsx

// ติดตั้ง package โดยตรงใน script
#r "nuget: Newtonsoft.Json, 13.0.3"
#r "nuget: FSharp.Data, 6.3.0"
#r "nuget: Dapper, 2.0.151"

open Newtonsoft.Json
open FSharp.Data

// ใช้ Newtonsoft.Json
type Person = { Name: string; Age: int }

let json = JsonConvert.SerializeObject({ Name = "สมชาย"; Age = 30 })
printfn "JSON: %s" json

let person = JsonConvert.DeserializeObject<Person>(json)
printfn "Name: %s, Age: %d" person.Name person.Age

// ใช้ FSharp.Data สำหรับ CSV
type Stocks = CsvProvider<"https://raw.githubusercontent.com/dotnet/machinelearning/main/test/data/MSN.csv">
let stocks = Stocks.Load("https://raw.githubusercontent.com/dotnet/machinelearning/main/test/data/MSN.csv")

printfn "จำนวน rows: %d" (stocks.Rows |> Seq.length)
for row in stocks.Rows |> Seq.take 5 do
    printfn "Date: %s, Close: %.2f" (row.Date.ToString("yyyy-MM-dd")) row.Close
```

### ระบุ Version และ Pre-release

```fsharp
// version-control.fsx

// ระบุ version ชัดเจน
#r "nuget: Newtonsoft.Json, 13.0.3"

// ระบุ version range
#r "nuget: FSharp.Data, >= 6.0.0"

// Pre-release
#r "nuget: MyPackage, *-*"

// Force restore (คล้าย --force ใน dotnet restore)
#r "nuget: SomePackage, 1.0.0"
```

---

## 3. #load สำหรับ .fsx Files

### โหลด Script Files อื่น

```fsharp
// utils.fsx - Helper functions
module Utils

let formatDate (dt: System.DateTime) =
    dt.ToString("dd/MM/yyyy HH:mm:ss")

let truncate (maxLen: int) (str: string) =
    if str.Length <= maxLen then str
    else str.[..maxLen-4] + "..."

let parseIntSafe (str: string) =
    match System.Int32.TryParse(str) with
    | true, n -> Some n
    | false, _ -> None
```

```fsharp
// config.fsx - Configuration
module Config

let DatabaseUrl = 
    match System.Environment.GetEnvironmentVariable("DATABASE_URL") with
    | null | "" -> "postgresql://localhost/mydb"
    | url -> url

let ApiKey =
    System.Environment.GetEnvironmentVariable("API_KEY")
    |> Option.ofObj
    |> Option.defaultValue "default-key"

let IsProduction =
    System.Environment.GetEnvironmentVariable("ENVIRONMENT") = "production"
```

```fsharp
// main.fsx - Main script
#load "utils.fsx"
#load "config.fsx"

open Utils
open Config

printfn "Database: %s" DatabaseUrl
printfn "Is Production: %b" IsProduction

let now = System.DateTime.Now
printfn "Current time: %s" (formatDate now)
printfn "Truncated: %s" (truncate 20 "This is a very long string that needs truncation")
```

---

## 4. Scripting with dotnet-script

### ติดตั้ง dotnet-script

```bash
# ติดตั้ง dotnet-script tool
dotnet tool install -g dotnet-script

# หรือ local
dotnet tool install dotnet-script

# ตรวจสอบ
dotnet script --version
```

### Script กับ dotnet-script

```fsharp
#!/usr/bin/env dotnet-script
// advanced-script.fsx

#r "nuget: Spectre.Console, 0.47.0"
#r "nuget: FsToolkit.ErrorHandling, 4.3.0"

open Spectre.Console
open FsToolkit.ErrorHandling

// แสดงผลด้วย Spectre Console
let table = Table()
table.AddColumn("[bold]Name[/]") |> ignore
table.AddColumn("[bold]Age[/]") |> ignore
table.AddColumn("[bold]City[/]") |> ignore

[
    "สมชาย", 30, "กรุงเทพ"
    "สมหญิง", 25, "เชียงใหม่"
    "สมศักดิ์", 35, "ภูเก็ต"
]
|> List.iter (fun (name, age, city) ->
    table.AddRow(name, string age, city) |> ignore)

AnsiConsole.Write(table)

// Progress bar
AnsiConsole.Progress().Start(fun ctx ->
    let task = ctx.AddTask("[green]Processing[/]")
    for i in 1..100 do
        task.Increment(1.0)
        System.Threading.Thread.Sleep(10)
)
```

---

## 5. File System Operations

### การทำงานกับ Files

```fsharp
// filesystem.fsx
open System.IO

// ===== Read Files =====

// อ่านไฟล์ทั้งหมด
let content = File.ReadAllText("/path/to/file.txt")

// อ่านทีละบรรทัด
let lines = File.ReadAllLines("/path/to/file.txt")
for line in lines do
    printfn "%s" line

// Streaming (สำหรับไฟล์ใหญ่)
use reader = new StreamReader("/path/to/large-file.txt")
while not reader.EndOfStream do
    let line = reader.ReadLine()
    printfn "%s" line

// ===== Write Files =====

// เขียนไฟล์ใหม่
File.WriteAllText("/path/to/output.txt", "Hello, World!")

// เขียนหลายบรรทัด
let lines = [| "Line 1"; "Line 2"; "Line 3" |]
File.WriteAllLines("/path/to/output.txt", lines)

// Append
File.AppendAllText("/path/to/log.txt", sprintf "%s: New entry\n" (System.DateTime.Now.ToString()))

// ===== Directory Operations =====

// สร้าง directory
Directory.CreateDirectory("/path/to/new-dir") |> ignore

// ลบ directory
Directory.Delete("/path/to/dir", recursive = true)

// List files
let files = Directory.GetFiles("/path/to/dir", "*.txt", SearchOption.AllDirectories)
for file in files do
    printfn "File: %s" file

// ===== Path Operations =====

let dir = Path.GetDirectoryName("/path/to/file.txt")
let filename = Path.GetFileName("/path/to/file.txt")
let ext = Path.GetExtension("/path/to/file.txt")
let nameWithoutExt = Path.GetFileNameWithoutExtension("/path/to/file.txt")
let combined = Path.Combine("/path", "to", "file.txt")

// ===== File Info =====

let info = FileInfo("/path/to/file.txt")
printfn "Size: %d bytes" info.Length
printfn "Created: %s" (info.CreationTime.ToString())
printfn "Modified: %s" (info.LastWriteTime.ToString())
printfn "Exists: %b" info.Exists

// ===== Recursive File Processing =====

let processFiles (directory: string) (pattern: string) (processor: string -> unit) =
    Directory.GetFiles(directory, pattern, SearchOption.AllDirectories)
    |> Array.iter processor

// Example: นับ lines ใน F# files
let countLinesInFSharpFiles () =
    let mutable totalLines = 0
    processFiles "." "*.fs" (fun file ->
        let count = File.ReadAllLines(file).Length
        printfn "%s: %d lines" (Path.GetFileName(file)) count
        totalLines <- totalLines + count)
    printfn "Total: %d lines" totalLines

countLinesInFSharpFiles()
```

### การ Monitor ไฟล์

```fsharp
// file-watcher.fsx
open System.IO

let watchDirectory (path: string) =
    use watcher = new FileSystemWatcher(path)
    watcher.NotifyFilter <- 
        NotifyFilters.LastWrite ||| 
        NotifyFilters.FileName |||
        NotifyFilters.DirectoryName
    
    watcher.Filter <- "*.*"
    watcher.IncludeSubdirectories <- true
    
    watcher.Changed.Add(fun e ->
        printfn "[CHANGED] %s" e.FullPath)
    
    watcher.Created.Add(fun e ->
        printfn "[CREATED] %s" e.FullPath)
    
    watcher.Deleted.Add(fun e ->
        printfn "[DELETED] %s" e.FullPath)
    
    watcher.Renamed.Add(fun e ->
        printfn "[RENAMED] %s -> %s" e.OldFullPath e.FullPath)
    
    watcher.EnableRaisingEvents <- true
    
    printfn "Watching: %s" path
    printfn "Press Enter to stop..."
    System.Console.ReadLine() |> ignore

watchDirectory "."
```

---

## 6. Process Execution

### รัน External Processes

```fsharp
// process.fsx
open System.Diagnostics

// ===== Simple Process Execution =====

let runProcess (command: string) (args: string) =
    let psi = ProcessStartInfo(command, args)
    psi.RedirectStandardOutput <- true
    psi.RedirectStandardError <- true
    psi.UseShellExecute <- false
    
    use proc = Process.Start(psi)
    let output = proc.StandardOutput.ReadToEnd()
    let error = proc.StandardError.ReadToEnd()
    proc.WaitForExit()
    
    (proc.ExitCode, output, error)

// รัน git commands
let (code, output, error) = runProcess "git" "status"
printfn "Exit code: %d" code
printfn "Output:\n%s" output
if error <> "" then printfn "Error:\n%s" error

// รัน shell commands
let runShell command =
    runProcess "bash" (sprintf "-c \"%s\"" command)

let (_, ls, _) = runShell "ls -la"
printfn "%s" ls

// ===== Process กับ Streaming Output =====

let runWithStreaming (command: string) (args: string) =
    let psi = ProcessStartInfo(command, args)
    psi.RedirectStandardOutput <- true
    psi.UseShellExecute <- false
    
    use proc = Process.Start(psi)
    
    while not proc.StandardOutput.EndOfStream do
        let line = proc.StandardOutput.ReadLine()
        printfn ">> %s" line
    
    proc.WaitForExit()
    proc.ExitCode

// ===== Async Process =====

let runAsync (command: string) (args: string) =
    async {
        let psi = ProcessStartInfo(command, args)
        psi.RedirectStandardOutput <- true
        psi.RedirectStandardError <- true
        psi.UseShellExecute <- false
        
        use proc = Process.Start(psi)
        
        let! output = 
            proc.StandardOutput.ReadToEndAsync()
            |> Async.AwaitTask
        
        let! error =
            proc.StandardError.ReadToEndAsync()
            |> Async.AwaitTask
        
        proc.WaitForExit()
        return proc.ExitCode, output, error
    }

// รัน async
async {
    let! (code, out, err) = runAsync "dotnet" "--version"
    printfn ".NET version: %s" out.Trim()
} |> Async.RunSynchronously

// ===== Running Multiple Processes =====

let runParallel commands =
    commands
    |> List.map (fun (cmd, args) -> runAsync cmd args)
    |> Async.Parallel
    |> Async.RunSynchronously

let results = 
    runParallel [
        "git", "status"
        "dotnet", "--version"
        "node", "--version"
    ]

for (code, out, err) in results do
    printfn "Code: %d, Output: %s" code (out.Trim())
```

---

## 7. HTTP Calls ใน Scripts

```fsharp
// http.fsx
#r "nuget: FSharp.Data, 6.3.0"
#r "nuget: Newtonsoft.Json, 13.0.3"

open System.Net.Http
open Newtonsoft.Json

// ===== Simple HTTP GET =====

let httpGet (url: string) =
    async {
        use client = new HttpClient()
        let! response = client.GetStringAsync(url) |> Async.AwaitTask
        return response
    }

let json = 
    httpGet "https://jsonplaceholder.typicode.com/posts/1"
    |> Async.RunSynchronously

printfn "Response: %s" (json.[..200])

// ===== HTTP กับ Type Mapping =====

type Post = {
    id: int
    title: string
    body: string
    userId: int
}

let getPost id =
    async {
        use client = new HttpClient()
        let url = sprintf "https://jsonplaceholder.typicode.com/posts/%d" id
        let! json = client.GetStringAsync(url) |> Async.AwaitTask
        return JsonConvert.DeserializeObject<Post>(json)
    }

let post = getPost 1 |> Async.RunSynchronously
printfn "Title: %s" post.title

// ===== HTTP POST =====

let createPost (title: string) (body: string) =
    async {
        use client = new HttpClient()
        let content = {| title = title; body = body; userId = 1 |}
        let json = JsonConvert.SerializeObject(content)
        use body = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
        
        let! response = 
            client.PostAsync("https://jsonplaceholder.typicode.com/posts", body)
            |> Async.AwaitTask
        
        let! responseBody = response.Content.ReadAsStringAsync() |> Async.AwaitTask
        return JsonConvert.DeserializeObject<Post>(responseBody)
    }

let newPost = createPost "F# Script" "Hello from F# script" |> Async.RunSynchronously
printfn "Created post ID: %d" newPost.id

// ===== HTTP กับ Authentication =====

let authenticatedGet (url: string) (token: string) =
    async {
        use client = new HttpClient()
        client.DefaultRequestHeaders.Authorization <-
            System.Net.Http.Headers.AuthenticationHeaderValue("Bearer", token)
        
        let! response = client.GetAsync(url) |> Async.AwaitTask
        
        if response.IsSuccessStatusCode then
            let! content = response.Content.ReadAsStringAsync() |> Async.AwaitTask
            return Ok content
        else
            return Error (sprintf "HTTP %d: %s" (int response.StatusCode) response.ReasonPhrase)
    }

// ===== Download File =====

let downloadFile (url: string) (outputPath: string) =
    async {
        use client = new HttpClient()
        let! bytes = client.GetByteArrayAsync(url) |> Async.AwaitTask
        System.IO.File.WriteAllBytes(outputPath, bytes)
        printfn "Downloaded %d bytes to %s" bytes.Length outputPath
    }

// ดาวน์โหลดไฟล์
// downloadFile "https://example.com/file.zip" "/tmp/downloaded.zip" |> Async.RunSynchronously
```

---

## 8. JSON Processing ใน Scripts

```fsharp
// json-processing.fsx
#r "nuget: Newtonsoft.Json, 13.0.3"
#r "nuget: FSharp.SystemTextJson, 1.2.42"

open System.Text.Json

// ===== System.Text.Json =====

type Address = {
    Street: string
    City: string
    Country: string
}

type Employee = {
    Id: int
    Name: string
    Email: string
    Department: string
    Salary: decimal
    Address: Address
    Skills: string list
}

// Serialize
let employee = {
    Id = 1
    Name = "สมชาย ใจดี"
    Email = "somchai@example.com"
    Department = "Engineering"
    Salary = 75000m
    Address = { Street = "123 Main St"; City = "กรุงเทพ"; Country = "Thailand" }
    Skills = ["F#"; "C#"; "TypeScript"]
}

let options = JsonSerializerOptions(WriteIndented = true)
let json = JsonSerializer.Serialize(employee, options)
printfn "JSON:\n%s" json

// Deserialize
let deserialized = JsonSerializer.Deserialize<Employee>(json)
printfn "Name: %s, Dept: %s" deserialized.Name deserialized.Department

// ===== Process JSON Data =====

let processJsonFile (inputPath: string) (outputPath: string) =
    // อ่าน JSON
    let json = System.IO.File.ReadAllText(inputPath)
    let employees = JsonSerializer.Deserialize<Employee[]>(json)
    
    // Process
    let result = 
        employees
        |> Array.filter (fun e -> e.Department = "Engineering")
        |> Array.sortBy (fun e -> e.Salary)
        |> Array.map (fun e -> {| Name = e.Name; Salary = e.Salary; TopSkill = e.Skills |> List.tryHead |})
    
    // เขียนผลลัพธ์
    let outputJson = JsonSerializer.Serialize(result, options)
    System.IO.File.WriteAllText(outputPath, outputJson)
    printfn "Processed %d engineers, saved to %s" result.Length outputPath

// ===== JsonDocument สำหรับ dynamic JSON =====

let parseUnknownJson (json: string) =
    use doc = JsonDocument.Parse(json)
    let root = doc.RootElement
    
    match root.ValueKind with
    | JsonValueKind.Object ->
        printfn "Object with keys:"
        for prop in root.EnumerateObject() do
            printfn "  %s: %s" prop.Name (prop.Value.ToString())
    | JsonValueKind.Array ->
        printfn "Array with %d items" (root.GetArrayLength())
        for item in root.EnumerateArray() do
            printfn "  - %s" (item.ToString())
    | kind ->
        printfn "Value: %A" kind

parseUnknownJson """{"name": "test", "value": 42, "active": true}"""
```

---

## 9. CSV Processing

```fsharp
// csv-processing.fsx
#r "nuget: CsvHelper, 33.0.1"
#r "nuget: FSharp.Data, 6.3.0"

open System.IO
open CsvHelper
open CsvHelper.Configuration

// ===== Simple CSV Reading with FSharp.Data =====

open FSharp.Data

type SalesData = CsvProvider<"date,product,quantity,price
2024-01-01,Apple,100,25.50
2024-01-02,Banana,150,15.00", HasHeaders=true, Schema="date=date, product=string, quantity=int, price=float">

let salesFile = SalesData.Load("sales.csv")
for row in salesFile.Rows do
    printfn "Date: %s, Product: %s, Revenue: %.2f" 
        (row.Date.ToString("yyyy-MM-dd"))
        row.Product
        (float row.Quantity * row.Price)

// ===== CsvHelper =====

type SaleRecord = {
    Date: System.DateTime
    Product: string
    Quantity: int
    Price: decimal
    Revenue: decimal
}

let readCsv (filePath: string) =
    let config = CsvConfiguration(System.Globalization.CultureInfo.InvariantCulture)
    use reader = new StreamReader(filePath)
    use csv = new CsvReader(reader, config)
    
    csv.GetRecords<SaleRecord>()
    |> Seq.toList

let writeCsv (filePath: string) (records: SaleRecord list) =
    use writer = new StreamWriter(filePath)
    use csv = new CsvWriter(writer, System.Globalization.CultureInfo.InvariantCulture)
    
    csv.WriteRecords(records)

// ===== Data Analysis Script =====

let analyzeSales (filePath: string) =
    let data = readCsv filePath
    
    printfn "===== Sales Analysis ======"
    printfn "Total Records: %d" data.Length
    
    let totalRevenue = data |> List.sumBy (fun r -> r.Revenue)
    printfn "Total Revenue: ฿%.2f" totalRevenue
    
    let byProduct = 
        data
        |> List.groupBy (fun r -> r.Product)
        |> List.map (fun (product, rows) -> 
            product, rows |> List.sumBy (fun r -> r.Revenue))
        |> List.sortByDescending snd
    
    printfn "\nRevenue by Product:"
    for product, revenue in byProduct do
        let percentage = revenue / totalRevenue * 100m
        printfn "  %s: ฿%.2f (%.1f%%)" product revenue percentage
    
    let byMonth =
        data
        |> List.groupBy (fun r -> r.Date.ToString("yyyy-MM"))
        |> List.map (fun (month, rows) ->
            month, rows |> List.sumBy (fun r -> r.Revenue))
        |> List.sortBy fst
    
    printfn "\nRevenue by Month:"
    for month, revenue in byMonth do
        printfn "  %s: ฿%.2f" month revenue

// ===== CSV Transformation =====

let transformCsv (inputPath: string) (outputPath: string) =
    // อ่าน CSV
    let config = CsvConfiguration(System.Globalization.CultureInfo.InvariantCulture)
    use reader = new StreamReader(inputPath)
    use csv = new CsvReader(reader, config)
    
    // Transform
    let rows = 
        csv.GetRecords<{| Name: string; Value: int |}>()
        |> Seq.map (fun row -> 
            {| row with Value = row.Value * 2; Name = row.Name.ToUpper() |})
        |> Seq.toList
    
    // เขียน CSV ใหม่
    use writer = new StreamWriter(outputPath)
    use csvWriter = new CsvWriter(writer, System.Globalization.CultureInfo.InvariantCulture)
    csvWriter.WriteRecords(rows)
    
    printfn "Transformed %d rows" rows.Length
```

---

## 10. Automation Examples

### Git Automation

```fsharp
// git-automation.fsx
open System.Diagnostics
open System.IO

let runGit args =
    let psi = ProcessStartInfo("git", args)
    psi.RedirectStandardOutput <- true
    psi.RedirectStandardError <- true
    psi.UseShellExecute <- false
    
    use proc = Process.Start(psi)
    let output = proc.StandardOutput.ReadToEnd()
    let error = proc.StandardError.ReadToEnd()
    proc.WaitForExit()
    
    (proc.ExitCode, output.Trim(), error.Trim())

// Status report
let gitStatus () =
    let (_, output, _) = runGit "status --porcelain"
    let lines = output.Split('\n') |> Array.filter (fun s -> s <> "")
    
    printfn "Git Status:"
    let modified = lines |> Array.filter (fun l -> l.StartsWith(" M") || l.StartsWith("M "))
    let added = lines |> Array.filter (fun l -> l.StartsWith("A ") || l.StartsWith("?? "))
    let deleted = lines |> Array.filter (fun l -> l.StartsWith(" D") || l.StartsWith("D "))
    
    printfn "  Modified: %d files" modified.Length
    printfn "  Added: %d files" added.Length
    printfn "  Deleted: %d files" deleted.Length

// Commit with auto-message
let autoCommit () =
    let timestamp = System.DateTime.Now.ToString("yyyy-MM-dd HH:mm")
    let (_, status, _) = runGit "status --porcelain"
    
    if status = "" then
        printfn "Nothing to commit"
    else
        runGit "add -A" |> ignore
        let (code, _, error) = runGit (sprintf "commit -m \"Auto commit: %s\"" timestamp)
        if code = 0 then printfn "Committed successfully"
        else printfn "Commit failed: %s" error

// List recent commits
let recentCommits (n: int) =
    let (_, output, _) = runGit (sprintf "log --oneline -%d" n)
    printfn "Recent %d commits:" n
    for line in output.Split('\n') do
        if line <> "" then printfn "  %s" line

gitStatus()
recentCommits 10
```

### Database Script

```fsharp
// database.fsx
#r "nuget: Dapper, 2.1.28"
#r "nuget: Npgsql, 8.0.2"

open Dapper
open Npgsql

let connectionString = 
    match System.Environment.GetEnvironmentVariable("DATABASE_URL") with
    | null -> "Host=localhost;Database=mydb;Username=postgres;Password=password"
    | url -> url

type User = {
    Id: int
    Name: string
    Email: string
    CreatedAt: System.DateTime
}

let getConnection () = new NpgsqlConnection(connectionString)

let getUsers () =
    use conn = getConnection()
    conn.Query<User>("SELECT * FROM users ORDER BY created_at DESC") |> Seq.toList

let createUser (name: string) (email: string) =
    use conn = getConnection()
    let sql = "INSERT INTO users (name, email, created_at) VALUES (@name, @email, NOW()) RETURNING id"
    conn.ExecuteScalar<int>(sql, {| name = name; email = email |})

let deleteOldUsers (daysOld: int) =
    use conn = getConnection()
    let sql = "DELETE FROM users WHERE created_at < NOW() - INTERVAL '1 day' * @days"
    let count = conn.Execute(sql, {| days = daysOld |})
    printfn "Deleted %d old users" count

// รัน migration
let runMigration (sql: string) =
    use conn = getConnection()
    conn.Execute(sql) |> ignore
    printfn "Migration ran successfully"

let createUsersTable = """
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
)
"""

runMigration createUsersTable

let users = getUsers()
printfn "Found %d users" users.Length
```

---

## 11. Build Scripts กับ FAKE

### การติดตั้ง FAKE

```bash
# ติดตั้ง FAKE CLI
dotnet tool install fake-cli -g

# หรือ local
dotnet tool install fake-cli
```

### build.fsx

```fsharp
// build.fsx
#r "nuget: Fake.Core.Target, 6.0.0"
#r "nuget: Fake.DotNet.Cli, 6.0.0"
#r "nuget: Fake.IO.FileSystem, 6.0.0"
#r "nuget: Fake.Core.Environment, 6.0.0"
#r "nuget: Fake.Tools.Git, 6.0.0"

open Fake.Core
open Fake.DotNet
open Fake.IO
open Fake.IO.FileSystemOperators
open Fake.IO.Globbing.Operators
open Fake.Tools.Git

// ===== Setup =====

Context.FakeExecutionContext.Create false "build.fsx" []
|> Context.RuntimeContext.Fake
|> Context.setExecutionContext

// ===== Configuration =====

let buildDir = "./dist"
let testDir = "./test-results"
let version = 
    match Environment.environVarOrNone "BUILD_VERSION" with
    | Some v -> v
    | None -> "1.0.0"

// ===== Targets =====

Target.create "Clean" (fun _ ->
    Shell.cleanDirs [buildDir; testDir]
    printfn "Cleaned build directories"
)

Target.create "Restore" (fun _ ->
    DotNet.restore id "."
)

Target.create "Build" (fun _ ->
    DotNet.build (fun opts ->
        { opts with
            Configuration = DotNet.BuildConfiguration.Release
            OutputPath = Some buildDir
        }) "."
)

Target.create "Test" (fun _ ->
    DotNet.test (fun opts ->
        { opts with
            Configuration = DotNet.BuildConfiguration.Release
            ResultsDirectory = Some testDir
            Logger = Some "trx;LogFileName=results.trx"
        }) "."
)

Target.create "Publish" (fun _ ->
    DotNet.publish (fun opts ->
        { opts with
            Configuration = DotNet.BuildConfiguration.Release
            OutputPath = Some (buildDir </> "published")
            Runtime = Some "linux-x64"
            SelfContained = Some true
        }) "./src/MyApp"
)

Target.create "Docker" (fun _ ->
    let tag = sprintf "myapp:%s" version
    
    let buildResult = 
        Process.shellExec {
            Program = "docker"
            Args = sprintf "build -t %s ." tag
            WorkingDir = "."
            Environment = Map.empty
        }
    
    if buildResult <> 0 then
        failwith "Docker build failed"
    
    printfn "Docker image built: %s" tag
)

Target.create "All" ignore

// ===== Dependencies =====

open Fake.Core.TargetOperators

"Clean"
    ==> "Restore"
    ==> "Build"
    ==> "Test"
    ==> "Publish"
    ==> "Docker"
    ==> "All"

// รัน default target
Target.runOrDefault "All"
```

---

## 12. Data Processing Scripts

### Large Data Processing

```fsharp
// data-processing.fsx
#r "nuget: FSharp.Data, 6.3.0"
#r "nuget: Deedle, 3.0.0"
#r "nuget: CsvHelper, 33.0.1"

open System
open System.IO
open Deedle

// ===== Load Large Dataset =====

let loadLargeFile (filePath: string) (batchSize: int) =
    seq {
        use reader = new StreamReader(filePath)
        let mutable line = reader.ReadLine()  // Skip header
        
        let mutable batch = []
        let mutable count = 0
        
        while not reader.EndOfStream do
            line <- reader.ReadLine()
            batch <- line :: batch
            count <- count + 1
            
            if count % batchSize = 0 then
                yield List.rev batch
                batch <- []
        
        if not (List.isEmpty batch) then
            yield List.rev batch
    }

// Process in batches
let processInBatches (filePath: string) =
    let mutable total = 0
    
    for batch in loadLargeFile filePath 1000 do
        let batchResult = 
            batch 
            |> List.map (fun line -> line.Split(','))
            |> List.filter (fun parts -> parts.Length >= 3)
            |> List.length
        total <- total + batchResult
        printf "."  // Progress indicator
    
    printfn "\nProcessed %d records" total

// ===== Deedle Data Frames =====

let analyzeWithDeedle () =
    // สร้าง DataFrame
    let data = 
        Frame.ofRecords [
            {| Date = DateTime(2024, 1, 1); Revenue = 1000.0; Expenses = 700.0 |}
            {| Date = DateTime(2024, 1, 2); Revenue = 1200.0; Expenses = 800.0 |}
            {| Date = DateTime(2024, 1, 3); Revenue = 950.0; Expenses = 650.0 |}
            {| Date = DateTime(2024, 1, 4); Revenue = 1350.0; Expenses = 900.0 |}
            {| Date = DateTime(2024, 1, 5); Revenue = 1100.0; Expenses = 750.0 |}
        ]
    
    printfn "Data shape: %d rows x %d cols" (data |> Frame.countRows) (data |> Frame.countCols)
    
    // คำนวณ Profit
    let profit = data?Revenue - data?Expenses
    let dataWithProfit = data |> Frame.addColumn "Profit" profit
    
    // Statistics
    let revenueStats = data?Revenue |> Stats.mean
    printfn "Average Revenue: %.2f" revenueStats
    
    let maxProfit = profit |> Series.values |> Seq.max
    printfn "Max Profit: %.2f" maxProfit
    
    // Group by month
    printfn "\nProfit Summary:"
    dataWithProfit |> Frame.print

// ===== Parallel Processing =====

let processFilesParallel (directory: string) =
    let files = Directory.GetFiles(directory, "*.csv")
    
    files
    |> Array.Parallel.map (fun file ->
        let content = File.ReadAllLines(file)
        let count = content.Length
        (file, count))
    |> Array.sortByDescending snd
    |> Array.iter (fun (file, count) ->
        printfn "%s: %d lines" (Path.GetFileName(file)) count)

// ===== Pipeline Processing =====

let pipeline =
    // อ่านข้อมูล
    File.ReadAllLines("data.csv")
    // Skip header
    |> Array.skip 1
    // Parse
    |> Array.map (fun line ->
        let parts = line.Split(',')
        {| Date = DateTime.Parse(parts.[0])
           Product = parts.[1]
           Quantity = int parts.[2]
           Price = float parts.[3] |})
    // Filter
    |> Array.filter (fun r -> r.Quantity > 0 && r.Price > 0.0)
    // Compute
    |> Array.map (fun r -> {| r with Revenue = float r.Quantity * r.Price |})
    // Group
    |> Array.groupBy (fun r -> r.Product)
    // Aggregate
    |> Array.map (fun (product, rows) ->
        {| Product = product
           TotalRevenue = rows |> Array.sumBy (fun r -> r.Revenue)
           TotalQuantity = rows |> Array.sumBy (fun r -> r.Quantity)
           AveragePrice = rows |> Array.averageBy (fun r -> r.Price) |})
    // Sort
    |> Array.sortByDescending (fun r -> r.TotalRevenue)

printfn "Product Sales Summary:"
for product in pipeline do
    printfn "  %s: Revenue=%.2f, Qty=%d, Avg Price=%.2f" 
        product.Product 
        product.TotalRevenue 
        product.TotalQuantity 
        product.AveragePrice
```

---

## สรุป (Summary)

F# Scripting มีความสามารถหลักๆ ดังนี้:

1. **Script Files (.fsx)**: เขียน F# แบบ script ไม่ต้อง project
2. **NuGet Integration**: `#r "nuget: ..."` ติดตั้ง packages ตรงใน script
3. **File Loading**: `#load` โหลด script files อื่น
4. **File System**: เขียน/อ่าน files ได้ง่าย
5. **Process Execution**: รัน external commands
6. **HTTP Calls**: เรียก API endpoints
7. **Data Processing**: CSV, JSON processing
8. **FAKE**: Build automation ที่ทรงพลัง

---

*ไปต่อที่ Part 106: Parser Combinators กับ F#*
