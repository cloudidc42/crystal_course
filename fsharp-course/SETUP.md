# F# Development Environment Setup Guide
# คู่มือการติดตั้งสภาพแวดล้อมพัฒนา F#

> คู่มือฉบับสมบูรณ์สำหรับการตั้งค่าสภาพแวดล้อม F# development บน Windows, macOS, และ Linux
>
> Complete setup guide for F# development environment on Windows, macOS, and Linux

---

## สารบัญ (Table of Contents)

1. [ติดตั้ง .NET 8 SDK](#1-ติดตั้ง-net-8-sdk)
2. [VS Code + Ionide](#2-vs-code--ionide)
3. [Visual Studio 2022](#3-visual-studio-2022)
4. [JetBrains Rider](#4-jetbrains-rider)
5. [สร้างโปรเจกต์แรก](#5-สร้างโปรเจกต์แรก)
6. [F# Interactive (fsi)](#6-f-interactive-fsi)
7. [เครื่องมือ F# ที่สำคัญ](#7-เครื่องมือ-f-ที่สำคัญ)
8. [VS Code Extensions ที่แนะนำ](#8-vs-code-extensions-ที่แนะนำ)
9. [Keyboard Shortcuts](#9-keyboard-shortcuts)
10. [การ Debug F# Code](#10-การ-debug-f-code)
11. [dotnet CLI Reference](#11-dotnet-cli-reference)

---

## 1. ติดตั้ง .NET 8 SDK

.NET 8 SDK คือ runtime และเครื่องมือสำหรับพัฒนา F# applications ต้องติดตั้งก่อนทำสิ่งอื่นๆ ทั้งหมด

### Windows

**วิธีที่ 1: ดาวน์โหลดจาก Microsoft (แนะนำ)**

1. ไปที่ https://dot.net/download
2. เลือก **.NET 8** และ **SDK** (ไม่ใช่ Runtime)
3. เลือก Windows x64 (หรือ ARM64 สำหรับ Surface Pro X)
4. รันตัวติดตั้ง `.exe` และทำตามขั้นตอน
5. ตรวจสอบการติดตั้ง:

```powershell
dotnet --version
# ควรแสดง: 8.x.x
```

**วิธีที่ 2: ใช้ winget**

```powershell
winget install Microsoft.DotNet.SDK.8
```

**วิธีที่ 3: ใช้ Chocolatey**

```powershell
choco install dotnet-8.0-sdk
```

**วิธีที่ 4: ใช้ Scoop**

```powershell
scoop install dotnet-sdk
```

### macOS

**วิธีที่ 1: ดาวน์โหลด installer**

1. ไปที่ https://dot.net/download
2. เลือก .NET 8 SDK
3. เลือก macOS x64 (Intel) หรือ macOS Arm64 (Apple Silicon M1/M2/M3)
4. รันไฟล์ `.pkg`

**วิธีที่ 2: ใช้ Homebrew (แนะนำ)**

```bash
brew install --cask dotnet-sdk
# หรือระบุเวอร์ชัน
brew install --cask dotnet-sdk@8
```

**วิธีที่ 3: ใช้ dotnet-install script**

```bash
curl -sSL https://dot.net/v1/dotnet-install.sh | bash /dev/stdin --version latest --channel 8.0
```

**ตั้งค่า PATH สำหรับ macOS:**

```bash
# เพิ่มใน ~/.zshrc หรือ ~/.bash_profile
export DOTNET_ROOT="$HOME/.dotnet"
export PATH="$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools"
```

ตรวจสอบ:

```bash
dotnet --version
dotnet --info
```

### Linux

**Ubuntu/Debian:**

```bash
# เพิ่ม Microsoft package repository
wget https://packages.microsoft.com/config/ubuntu/22.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# ติดตั้ง SDK
sudo apt-get update
sudo apt-get install -y dotnet-sdk-8.0
```

**Fedora/RHEL/CentOS:**

```bash
sudo dnf install dotnet-sdk-8.0
```

**Arch Linux:**

```bash
sudo pacman -S dotnet-sdk
```

**Alpine Linux:**

```bash
apk add dotnet-sdk
```

**ใช้ dotnet-install script (ทำงานได้บนทุก distro):**

```bash
curl -sSL https://dot.net/v1/dotnet-install.sh -o dotnet-install.sh
chmod +x ./dotnet-install.sh
./dotnet-install.sh --version latest --channel 8.0

# เพิ่มใน ~/.bashrc หรือ ~/.zshrc
export DOTNET_ROOT="$HOME/.dotnet"
export PATH="$PATH:$DOTNET_ROOT:$DOTNET_ROOT/tools"
source ~/.bashrc
```

### ตรวจสอบการติดตั้ง

```bash
dotnet --version          # เวอร์ชัน SDK
dotnet --list-sdks        # SDKs ที่ติดตั้งทั้งหมด
dotnet --list-runtimes    # Runtimes ที่ติดตั้งทั้งหมด
dotnet fsi --version      # F# Interactive version
```

---

## 2. VS Code + Ionide

VS Code กับ Ionide extension คือ environment ที่แนะนำที่สุดสำหรับการเรียน F# — ฟรี, เบา, และมี features ครบถ้วน

### ติดตั้ง VS Code

1. ดาวน์โหลดจาก https://code.visualstudio.com/
2. ติดตั้งตามระบบปฏิบัติการ

**macOS — ใช้ Homebrew:**
```bash
brew install --cask visual-studio-code
```

**Linux — Ubuntu/Debian:**
```bash
sudo snap install --classic code
```

**Linux — Flatpak:**
```bash
flatpak install flathub com.visualstudio.code
```

### ติดตั้ง Ionide

Ionide คือ VS Code extension สำหรับ F# ที่ให้:
- Syntax highlighting
- IntelliSense / autocomplete
- Type information on hover
- Go-to-definition
- Find all references
- Rename symbol
- Code lens
- Inline errors
- F# Interactive integration

**ติดตั้ง Ionide-fsharp:**

1. เปิด VS Code
2. กด `Ctrl+Shift+X` (หรือ `Cmd+Shift+X` บน Mac) เปิด Extensions
3. ค้นหา **"Ionide-fsharp"**
4. คลิก Install

หรือจาก command line:
```bash
code --install-extension ionide.ionide-fsharp
```

### ตั้งค่า VS Code สำหรับ F#

เปิด Settings (Ctrl+,) และเพิ่มใน `settings.json`:

```json
{
    "editor.formatOnSave": true,
    "FSharp.enableTreeView": true,
    "FSharp.showExplorerOnStartup": true,
    "FSharp.inlayHints.enabled": true,
    "FSharp.inlayHints.parameterNames": true,
    "FSharp.inlayHints.typeAnnotations": true,
    "FSharp.linter": true,
    "FSharp.fsiExtraParameters": ["--langversion:preview"],
    "editor.tabSize": 4,
    "editor.insertSpaces": true,
    "editor.rulers": [100],
    "[fsharp]": {
        "editor.defaultFormatter": "ionide.ionide-fsharp",
        "editor.formatOnSave": true
    }
}
```

---

## 3. Visual Studio 2022

Visual Studio 2022 เหมาะสำหรับการพัฒนา .NET applications ขนาดใหญ่ มี features ครบถ้วนที่สุดแต่ใช้ทรัพยากรมากกว่า VS Code (Windows เท่านั้น)

### ดาวน์โหลดและติดตั้ง

1. ไปที่ https://visualstudio.microsoft.com/
2. ดาวน์โหลด **Visual Studio 2022 Community** (ฟรี) หรือ Professional/Enterprise
3. รัน installer

### เลือก Workloads

ใน Visual Studio Installer ให้เลือก:
- **ASP.NET and web development** — สำหรับเว็บ applications
- **.NET desktop development** — สำหรับ desktop apps
- **Azure development** — สำหรับ cloud deployment
- **.NET cross-platform development** — สำหรับ Linux/macOS targets

### ติดตั้ง F# Support

F# support มากับ Visual Studio แต่ต้องตรวจสอบ:

1. เปิด Visual Studio Installer
2. คลิก Modify บน VS 2022
3. ไปที่ **Individual components**
4. ค้นหา **F# language support**
5. ตรวจสอบว่าติดตั้งแล้ว

### การตั้งค่า F# ใน Visual Studio

**Tools → Options → F# Tools:**
- Enable formatting on save: On
- Enable linting: On
- Enable type hints: On

---

## 4. JetBrains Rider

Rider คือ cross-platform .NET IDE จาก JetBrains ที่มี F# support ดีเยี่ยม ทำงานได้บน Windows, macOS, และ Linux

### ติดตั้ง Rider

1. ดาวน์โหลดจาก https://www.jetbrains.com/rider/
2. มีทดลองใช้ 30 วัน (หลังจากนั้นต้องมี license)
3. นักเรียน/นักศึกษาได้ฟรีผ่าน https://www.jetbrains.com/student/

**macOS — ใช้ Homebrew:**
```bash
brew install --cask rider
```

### F# Support ใน Rider

Rider รองรับ F# อย่างสมบูรณ์:
- Syntax highlighting และ code completion
- Refactoring tools (rename, extract function, etc.)
- Built-in F# Interactive
- Debugger ที่แสดง F# types ได้ถูกต้อง
- Unit test runner สำหรับ Expecto, xUnit, NUnit

### ReSharper Plugin สำหรับ Visual Studio

หากใช้ Visual Studio ให้ติดตั้ง ReSharper ซึ่งมี F# plugin:
- ไปที่ Extensions → Manage Extensions
- ค้นหา "F# support for ReSharper"

---

## 5. สร้างโปรเจกต์แรก

### F# Console Application

```bash
# สร้างโปรเจกต์ใหม่
dotnet new console -lang F# -n MyFirstFSharp
cd MyFirstFSharp

# รันโปรแกรม
dotnet run
```

ไฟล์ `Program.fs` จะมีเนื้อหา:
```fsharp
// For more information see https://aka.ms/fsharp-console-apps
printfn "Hello from F#"
```

### F# Library

```bash
dotnet new classlib -lang F# -n MyFSharpLib
cd MyFSharpLib
```

### F# Web Application (Giraffe)

```bash
# ติดตั้ง template ก่อน
dotnet new install "Giraffe.Template"

# สร้าง Giraffe web app
dotnet new giraffe -n MyWebApp
cd MyWebApp
dotnet run
```

### Solution พร้อม multiple projects

```bash
# สร้าง solution
dotnet new sln -n MySolution
mkdir src tests

# สร้าง library project
dotnet new classlib -lang F# -n MyLib -o src/MyLib

# สร้าง console project
dotnet new console -lang F# -n MyApp -o src/MyApp

# สร้าง test project
dotnet new xunit -lang F# -n MyTests -o tests/MyTests

# เพิ่มโปรเจกต์ใน solution
dotnet sln add src/MyLib/MyLib.fsproj
dotnet sln add src/MyApp/MyApp.fsproj
dotnet sln add tests/MyTests/MyTests.fsproj

# เพิ่ม project reference
dotnet add src/MyApp/MyApp.fsproj reference src/MyLib/MyLib.fsproj
dotnet add tests/MyTests/MyTests.fsproj reference src/MyLib/MyLib.fsproj
```

### โครงสร้างโปรเจกต์ที่แนะนำ

```
MySolution/
├── MySolution.sln
├── src/
│   ├── Domain/           # Domain models, business logic
│   │   ├── Domain.fsproj
│   │   └── ...
│   ├── Infrastructure/   # Database, external services
│   │   ├── Infrastructure.fsproj
│   │   └── ...
│   └── Api/              # Web API, entry point
│       ├── Api.fsproj
│       └── ...
├── tests/
│   ├── Domain.Tests/
│   │   ├── Domain.Tests.fsproj
│   │   └── ...
│   └── Integration.Tests/
│       └── ...
└── build/                # FAKE build scripts
    └── build.fsx
```

---

## 6. F# Interactive (fsi)

F# Interactive (fsi) เป็น REPL (Read-Eval-Print Loop) สำหรับทดลองและ explore F# code ทันที

### เปิด fsi

```bash
# เปิด F# Interactive
dotnet fsi

# รัน script ไฟล์
dotnet fsi my-script.fsx

# รัน script พร้อม arguments
dotnet fsi my-script.fsx -- arg1 arg2
```

### คำสั่ง fsi พื้นฐาน

```fsharp
// ใน fsi ให้ตามด้วย ;; เพื่อ evaluate
let x = 42;;
// val x : int = 42

let greet name = printfn "Hello, %s!" name;;
greet "World";;
// Hello, World!

// ดู type ของ expression
let nums = [1..10];;
// val nums : int list = [1; 2; 3; 4; 5; 6; 7; 8; 9; 10]

// Load script file
#load "MyModule.fsx";;

// เพิ่ม NuGet package
#r "nuget: Newtonsoft.Json";;

// ออกจาก fsi
#quit;;
// หรือกด Ctrl+D (Linux/Mac) / Ctrl+Z Enter (Windows)
```

### fsi ใน VS Code (Ionide)

1. เปิดไฟล์ `.fs` หรือ `.fsx`
2. เลือก code ที่ต้องการ evaluate
3. กด `Alt+Enter` เพื่อส่ง selection ไป fsi
4. กด `Ctrl+Alt+Enter` เพื่อส่ง ทั้ง file

### .fsx Script Files

สร้างไฟล์ `script.fsx`:

```fsharp
#!/usr/bin/env dotnet-script
// F# script example

#r "nuget: FSharp.Data, 6.3.0"

open FSharp.Data

// Download and parse CSV
let data = CsvFile.Load("https://example.com/data.csv")
for row in data.Rows do
    printfn "%s" row.[0]
```

รันด้วย:
```bash
dotnet fsi script.fsx
# หรือถ้าใช้ dotnet-script tool
dotnet script script.fsx
```

---

## 7. เครื่องมือ F# ที่สำคัญ

### Fantomas — Code Formatter

Fantomas คือ official F# code formatter ที่ทำให้ code มี style สม่ำเสมอ

**ติดตั้ง Fantomas:**

```bash
# ติดตั้งเป็น global tool
dotnet tool install -g fantomas

# หรือเป็น local tool (แนะนำสำหรับโปรเจกต์)
dotnet new tool-manifest    # สร้าง .config/dotnet-tools.json
dotnet tool install fantomas
```

**ใช้ Fantomas:**

```bash
# Format ไฟล์เดียว
fantomas MyFile.fs

# Format ทั้งโฟลเดอร์
fantomas src/

# ตรวจสอบโดยไม่แก้ไข (สำหรับ CI)
fantomas --check src/

# Dry run — แสดงการเปลี่ยนแปลงโดยไม่บันทึก
fantomas --dry-run src/
```

**ตั้งค่า Fantomas ใน `.editorconfig`:**

```ini
[*.fs]
indent_size = 4
max_line_length = 100
fsharp_multiline_block_brackets_on_same_column = true
fsharp_newline_before_multiline_computation_expression = true
```

**ตั้งค่า Fantomas ใน `fantomas-config.json`:**

```json
{
  "IndentSize": 4,
  "MaxLineLength": 100,
  "MultilineBlockBracketsOnSameColumn": true,
  "NewlineBeforeMultilineComputationExpression": true,
  "SpaceBeforeColon": false
}
```

### FSharpLint — Code Linter

FSharpLint ตรวจสอบ code quality และ style issues

**ติดตั้ง FSharpLint:**

```bash
dotnet tool install -g dotnet-fsharplint
```

**ใช้ FSharpLint:**

```bash
# Lint โปรเจกต์
dotnet fsharplint lint MyProject.fsproj

# Lint solution
dotnet fsharplint lint MySolution.sln
```

**ตั้งค่าใน `fsharplint.json`:**

```json
{
    "ignoreFiles": ["**/obj/**", "**/bin/**"],
    "analysers": {
        "Hints": { "enabled": true },
        "Naming": { "enabled": true },
        "Typography": { "enabled": true }
    }
}
```

### Paket — Dependency Manager

Paket เป็น alternative ต่อ NuGet ที่มีความสามารถมากกว่า

**ติดตั้ง Paket:**

```bash
dotnet tool install -g Paket

# หรือ local
dotnet new tool-manifest
dotnet tool install Paket
```

**เริ่มใช้ Paket ในโปรเจกต์:**

```bash
# Initialize
paket init

# เพิ่ม dependency
paket add Newtonsoft.Json

# ติดตั้ง dependencies
paket install

# อัปเดต dependencies
paket update
```

**`paket.dependencies` ตัวอย่าง:**

```
source https://api.nuget.org/v3/index.json

nuget FSharp.Core ~> 8
nuget Newtonsoft.Json >= 13.0
nuget Giraffe >= 7.0
nuget Dapper >= 2.1
```

### FAKE — Build Tool

FAKE (F# Make) เป็น DSL สำหรับเขียน build scripts ใน F#

**ติดตั้ง FAKE:**

```bash
dotnet tool install -g fake-cli

# หรือ local
dotnet new tool-manifest
dotnet tool install fake-cli
```

**สร้าง build script `build.fsx`:**

```fsharp
#r "nuget: Fake.Core.Target"
#r "nuget: Fake.DotNet.Cli"
#r "nuget: Fake.IO.FileSystem"

open Fake.Core
open Fake.DotNet
open Fake.IO

Target.create "Clean" (fun _ ->
    Shell.cleanDirs ["bin"; "obj"; "dist"]
)

Target.create "Build" (fun _ ->
    DotNet.build (fun opts -> { opts with Configuration = DotNet.BuildConfiguration.Release }) "."
)

Target.create "Test" (fun _ ->
    DotNet.test (fun opts -> { opts with NoBuild = true }) "."
)

Target.create "Publish" (fun _ ->
    DotNet.publish (fun opts ->
        { opts with
            Configuration = DotNet.BuildConfiguration.Release
            OutputPath = Some "dist" }
    ) "src/Api/Api.fsproj"
)

open Fake.Core.TargetOperators

"Clean" ==> "Build" ==> "Test" ==> "Publish"

Target.runOrDefault "Test"
```

**รัน FAKE build:**

```bash
fake build
fake build -t Clean
fake build -t Test
```

### dotnet-script

dotnet-script ทำให้รัน F# scripts ง่ายขึ้นพร้อม NuGet references

```bash
dotnet tool install -g dotnet-script
```

**ตัวอย่าง script:**

```fsharp
#!/usr/bin/env dotnet-script
#r "nuget: Spectre.Console, 0.49.1"

open Spectre.Console

AnsiConsole.Write(
    new FigletText("Hello, F#!")
        .Color(Color.Green))
```

---

## 8. VS Code Extensions ที่แนะนำ

### Extensions ที่จำเป็น

| Extension | Publisher | วัตถุประสงค์ |
|-----------|-----------|------------|
| Ionide-fsharp | ionide | F# language support (IntelliSense, linting, formatting) |
| C# Dev Kit | Microsoft | .NET project support |
| .NET Install Tool | Microsoft | จัดการ .NET SDK versions |

**ติดตั้งทั้งหมดพร้อมกัน:**
```bash
code --install-extension ionide.ionide-fsharp
code --install-extension ms-dotnettools.csdevkit
code --install-extension ms-dotnettools.vscode-dotnet-runtime
```

### Extensions ที่แนะนำ

| Extension | วัตถุประสงค์ |
|-----------|------------|
| Error Lens | แสดง errors inline ในไฟล์ |
| GitLens | Git integration ขั้นสูง |
| GitHub Copilot | AI code completion |
| REST Client | ทดสอบ HTTP APIs จาก VS Code |
| Thunder Client | GUI สำหรับ REST API testing |
| Docker | Docker container management |
| Markdown All in One | เขียน documentation |
| Todo Tree | ติดตาม TODO comments |
| Bracket Pair Colorizer | สีสำหรับ brackets |
| indent-rainbow | สีสำหรับ indentation |

```bash
code --install-extension usernamehw.errorlens
code --install-extension eamodio.gitlens
code --install-extension humao.rest-client
code --install-extension ms-azuretools.vscode-docker
```

### การตั้งค่า Extensions

เพิ่มใน `settings.json`:

```json
{
    "errorLens.enabledDiagnosticLevels": ["error", "warning", "info"],
    "gitlens.currentLine.enabled": true,
    "gitlens.codeLens.enabled": true,
    "editor.inlineSuggest.enabled": true
}
```

---

## 9. Keyboard Shortcuts

### VS Code + Ionide (Windows/Linux / macOS)

| การกระทำ | Windows/Linux | macOS |
|--------|--------------|-------|
| Send selection to F# Interactive | `Alt+Enter` | `Option+Enter` |
| Send file to F# Interactive | `Ctrl+Alt+Enter` | `Cmd+Option+Enter` |
| Format document | `Shift+Alt+F` | `Shift+Option+F` |
| Go to definition | `F12` | `F12` |
| Peek definition | `Alt+F12` | `Option+F12` |
| Find all references | `Shift+F12` | `Shift+F12` |
| Rename symbol | `F2` | `F2` |
| Quick fix / Code actions | `Ctrl+.` | `Cmd+.` |
| Toggle comments | `Ctrl+/` | `Cmd+/` |
| Show hover info | `Ctrl+K Ctrl+I` | `Cmd+K Cmd+I` |
| Open Symbols | `Ctrl+Shift+O` | `Cmd+Shift+O` |
| Go to line | `Ctrl+G` | `Cmd+G` |
| Toggle terminal | `` Ctrl+` `` | `` Cmd+` `` |
| Open command palette | `Ctrl+Shift+P` | `Cmd+Shift+P` |
| Open settings | `Ctrl+,` | `Cmd+,` |

### Visual Studio 2022

| การกระทำ | Shortcut |
|--------|---------|
| Build solution | `Ctrl+Shift+B` |
| Run with debugger | `F5` |
| Run without debugger | `Ctrl+F5` |
| Stop debugging | `Shift+F5` |
| Toggle breakpoint | `F9` |
| Step over | `F10` |
| Step into | `F11` |
| Go to definition | `F12` |
| Find all references | `Shift+F12` |
| Rename | `Ctrl+R Ctrl+R` |
| Format document | `Ctrl+K Ctrl+D` |
| Comment selection | `Ctrl+K Ctrl+C` |
| Uncomment selection | `Ctrl+K Ctrl+U` |
| Open F# Interactive | `Alt+F8` |

### JetBrains Rider

| การกระทำ | Windows/Linux | macOS |
|--------|--------------|-------|
| Find action | `Ctrl+Shift+A` | `Cmd+Shift+A` |
| Go to definition | `Ctrl+B` | `Cmd+B` |
| Find usages | `Alt+F7` | `Option+F7` |
| Rename refactor | `Shift+F6` | `Shift+F6` |
| Format code | `Ctrl+Alt+L` | `Cmd+Option+L` |
| Build solution | `Ctrl+F9` | `Cmd+F9` |
| Debug | `Shift+F9` | `Shift+F9` |
| Run | `Shift+F10` | `Shift+F10` |
| Toggle breakpoint | `Ctrl+F8` | `Cmd+F8` |

---

## 10. การ Debug F# Code

### VS Code Debugging

**1. เพิ่ม launch configuration ใน `.vscode/launch.json`:**

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "F# Launch",
            "type": "coreclr",
            "request": "launch",
            "preLaunchTask": "dotnet: build",
            "program": "${workspaceFolder}/bin/Debug/net8.0/MyApp.dll",
            "args": [],
            "cwd": "${workspaceFolder}",
            "stopAtEntry": false,
            "console": "internalConsole"
        },
        {
            "name": "Attach to Process",
            "type": "coreclr",
            "request": "attach"
        }
    ]
}
```

**2. เพิ่ม tasks ใน `.vscode/tasks.json`:**

```json
{
    "version": "2.0.0",
    "tasks": [
        {
            "label": "dotnet: build",
            "command": "dotnet",
            "type": "process",
            "args": ["build", "${workspaceFolder}"],
            "problemMatcher": "$msCompile"
        }
    ]
}
```

**3. วาง Breakpoints:**
- คลิกที่ gutter (ซ้ายของ line number) เพื่อวาง breakpoint
- หรือกด `F9`

**4. เริ่ม Debug:**
- กด `F5` หรือไปที่ Run → Start Debugging

### Debugging Tips สำหรับ F#

**ใช้ `printfn` สำหรับ quick debugging:**

```fsharp
let debugValue label value =
    printfn "[DEBUG] %s = %A" label value
    value    // ส่งค่าคืนเพื่อใช้ใน pipeline

// ใน pipeline
let result =
    [1..10]
    |> List.map (fun x -> x * 2)
    |> debugValue "after map"
    |> List.filter (fun x -> x > 5)
    |> debugValue "after filter"
```

**ใช้ `Debug.Assert`:**

```fsharp
open System.Diagnostics

let divide a b =
    Debug.Assert(b <> 0, "Divisor cannot be zero")
    a / b
```

**ใช้ `Debugger.Break()`:**

```fsharp
open System.Diagnostics

let suspiciousFunction x =
    if x < 0 then
        Debugger.Break()  // หยุดที่นี่เมื่อเชื่อมต่อ debugger
    x * 2
```

### Interactive Debugging ด้วย fsi

```fsharp
// ใน fsi สามารถทดสอบ functions ทีละขั้นตอน
let myList = [1; 2; 3; 4; 5];;
let doubled = List.map (fun x -> x * 2) myList;;
doubled;;
// val doubled : int list = [2; 4; 6; 8; 10]
```

---

## 11. dotnet CLI Reference

### Project Management

```bash
# ดู templates ที่มี
dotnet new list
dotnet new list --type project
dotnet new list --tag fsharp

# สร้างโปรเจกต์
dotnet new console -lang F# -n ProjectName
dotnet new classlib -lang F# -n LibraryName
dotnet new xunit -lang F# -n TestProject
dotnet new webapi -lang F# -n WebApiProject
dotnet new mvc -lang F# -n MvcProject

# สร้าง solution
dotnet new sln -n SolutionName

# เพิ่ม/ลบ project ใน solution
dotnet sln add MyProject/MyProject.fsproj
dotnet sln remove MyProject/MyProject.fsproj
dotnet sln list
```

### Build Commands

```bash
# Build
dotnet build                              # Build default
dotnet build -c Release                   # Release build
dotnet build -c Debug                     # Debug build (default)
dotnet build --no-restore                 # Skip restore
dotnet build -v detailed                  # Verbose output
dotnet build --framework net8.0           # Target specific framework

# Clean
dotnet clean
dotnet clean -c Release
```

### Run Commands

```bash
# Run
dotnet run
dotnet run -c Release
dotnet run -- arg1 arg2                   # Pass arguments to app
dotnet run --project src/MyApp/MyApp.fsproj

# Watch (hot reload)
dotnet watch run
dotnet watch test
dotnet watch build
```

### Test Commands

```bash
# Test
dotnet test
dotnet test -c Release
dotnet test --no-build
dotnet test --filter "ClassName=MyTests"
dotnet test --filter "Method=TestName"
dotnet test -v normal                     # Verbose
dotnet test --collect:"XPlat Code Coverage"  # Coverage
dotnet test --logger "html;logfilename=testresults.html"
```

### NuGet Commands

```bash
# เพิ่ม package
dotnet add package PackageName
dotnet add package PackageName --version 1.2.3
dotnet add package PackageName --prerelease

# ลบ package
dotnet remove package PackageName

# แสดง packages
dotnet list package
dotnet list package --outdated
dotnet list package --vulnerable

# Restore packages
dotnet restore
dotnet restore --no-cache

# NuGet sources
dotnet nuget list source
dotnet nuget add source https://custom-feed.com/index.json -n "CustomFeed"
```

### Publish Commands

```bash
# Publish
dotnet publish
dotnet publish -c Release -o ./dist
dotnet publish -c Release --self-contained
dotnet publish -c Release -r linux-x64 --self-contained
dotnet publish -c Release -r win-x64 /p:PublishSingleFile=true

# Runtime Identifiers (RID)
# Windows: win-x64, win-x86, win-arm64
# Linux: linux-x64, linux-arm, linux-arm64
# macOS: osx-x64, osx-arm64
```

### Tool Management

```bash
# Global tools
dotnet tool install -g fantomas
dotnet tool install -g fake-cli
dotnet tool install -g paket
dotnet tool install -g dotnet-script

# Local tools
dotnet new tool-manifest
dotnet tool install fantomas
dotnet tool restore                       # Restore all local tools

# อัปเดต tools
dotnet tool update -g fantomas
dotnet tool update fantomas

# ลบ tools
dotnet tool uninstall -g fantomas

# แสดง tools ที่ติดตั้ง
dotnet tool list
dotnet tool list -g                       # Global tools
```

### Environment Info

```bash
dotnet --version                          # SDK version
dotnet --list-sdks                        # ดู SDKs ที่ติดตั้ง
dotnet --list-runtimes                    # ดู Runtimes ที่ติดตั้ง
dotnet --info                             # ข้อมูลระบบทั้งหมด
dotnet sdk check                          # ตรวจสอบ updates
```

### global.json — Pin SDK Version

สร้าง `global.json` ใน root ของโปรเจกต์เพื่อระบุ SDK version:

```json
{
    "sdk": {
        "version": "8.0.100",
        "rollForward": "latestFeature"
    }
}
```

`rollForward` options:
- `patch` — ใช้ patch version ที่ใหม่กว่า
- `feature` — ใช้ feature version ที่ใหม่กว่า
- `minor` — ใช้ minor version ที่ใหม่กว่า
- `major` — ใช้ major version ที่ใหม่กว่า
- `latestPatch`, `latestFeature`, `latestMinor`, `latestMajor` — ใช้ latest ในแต่ละ level
- `disable` — ใช้แค่เวอร์ชันที่ระบุเท่านั้น

---

## Tips สำหรับผู้เริ่มต้น

### ปัญหาที่พบบ่อย

**1. "Could not find F# compiler"**
```bash
# ติดตั้ง F# tools
dotnet tool install -g fsharp
# หรือตรวจสอบว่า .NET SDK ติดตั้งครบ
dotnet --info
```

**2. Ionide ไม่ทำงาน**
```bash
# ล้าง FSAutoComplete cache
# ลบโฟลเดอร์ .ionide ใน home directory
# Reload VS Code window: Ctrl+Shift+P → "Developer: Reload Window"
```

**3. ไฟล์ .fs ไม่ compile**
ใน F# ลำดับไฟล์ใน `.fsproj` มีความสำคัญ ไฟล์ที่ถูก include ต้องเรียงตาม dependency:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
  </PropertyGroup>
  <ItemGroup>
    <!-- ลำดับสำคัญมาก! Types ต้องมาก่อน functions ที่ใช้มัน -->
    <Compile Include="Domain.fs" />      <!-- ← ต้องมาก่อน -->
    <Compile Include="Services.fs" />   <!-- ← ใช้ Domain.fs -->
    <Compile Include="Program.fs" />    <!-- ← entry point สุดท้าย -->
  </ItemGroup>
</Project>
```

**4. "The namespace or module 'X' is not defined"**
```fsharp
// ต้องใช้ open ก่อน
open System
open System.IO
open System.Collections.Generic
```

---

*อัปเดตสำหรับ .NET 8 SDK — ตุลาคม 2026*
