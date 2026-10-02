# Part 80 - Code Coverage และ Quality

## บทนำ (Introduction)

Code Coverage และ Code Quality เป็นส่วนสำคัญของ software development process ที่ช่วยให้มั่นใจว่า code มีคุณภาพสูงและมีการทดสอบที่เพียงพอ

**เครื่องมือที่จะเรียนรู้:**
- **Coverlet** - Code coverage collection
- **SonarQube** - Static analysis
- **FSharpLint** - F# linting
- **Fantomas** - Code formatting
- **Stryker** - Mutation testing

---

## 1. Coverlet สำหรับ Code Coverage

### การติดตั้ง

```xml
<!-- TestProject.fsproj -->
<ItemGroup>
  <!-- Coverlet via collector -->
  <PackageReference Include="coverlet.collector" Version="6.0.0">
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
    <PrivateAssets>all</PrivateAssets>
  </PackageReference>
  
  <!-- Or coverlet.msbuild for more control -->
  <PackageReference Include="coverlet.msbuild" Version="6.0.0">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

### รัน Tests กับ Coverage

```bash
# Basic coverage collection
dotnet test --collect:"XPlat Code Coverage"

# กำหนด output format
dotnet test --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=cobertura

# Generate report กับ ReportGenerator
dotnet tool install --global dotnet-reportgenerator-globaltool

reportgenerator \
  -reports:"TestResults/*/coverage.cobertura.xml" \
  -targetdir:"coverage-report" \
  -reporttypes:Html

# เปิด report
open coverage-report/index.html
```

---

## 2. dotnet test --collect

```bash
# Coverage options

# Format: lcov, opencover, cobertura, json
dotnet test --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Format=opencover

# Exclude namespaces/classes
dotnet test --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Exclude="[*]*.Migrations.*,[*]*.Generated.*"

# Include only specific
dotnet test --collect:"XPlat Code Coverage" \
  -- DataCollectionRunSettings.DataCollectors.DataCollector.Configuration.Include="[MyProject.*]*"

# หรือ กำหนดใน runsettings file
dotnet test --settings coverage.runsettings
```

```xml
<!-- coverage.runsettings -->
<?xml version="1.0" encoding="utf-8" ?>
<RunSettings>
  <DataCollectionRunSettings>
    <DataCollectors>
      <DataCollector friendlyName="XPlat code coverage">
        <Configuration>
          <Format>cobertura</Format>
          <Exclude>
            [*]*.Migrations.*
            [*]*Generated*
            [*]*Program
          </Exclude>
          <ExcludeByAttribute>
            ExcludeFromCodeCoverage,
            GeneratedCode,
            CompilerGenerated
          </ExcludeByAttribute>
          <IncludeTestAssembly>false</IncludeTestAssembly>
        </Configuration>
      </DataCollector>
    </DataCollectors>
  </DataCollectionRunSettings>
</RunSettings>
```

---

## 3. Coverage Reports

```bash
# สร้าง HTML report
reportgenerator \
  -reports:"**/coverage.cobertura.xml" \
  -targetdir:"coverage-report" \
  -reporttypes:"Html;Badges;TextSummary"

# Badge สำหรับ README
# coverage-report/badge_linecoverage.svg

# Text summary
cat coverage-report/Summary.txt
```

```fsharp
// ExcludeFromCoverage.fs
// บาง code ไม่ต้องการ coverage

open System.Diagnostics.CodeAnalysis

// Exclude specific function
[<ExcludeFromCodeCoverage>]
let debugHelper x =
    printfn "Debug: %A" x
    x

// Exclude entire module
[<ExcludeFromCodeCoverage>]
module DebugHelpers =
    let printState state =
        printfn "State: %A" state
    
    let logError message =
        eprintfn "Error: %s" message

// Coverage-friendly code

// Code ที่ covers ได้ดี
module BusinessLogic =
    let validate (input: string) =
        if System.String.IsNullOrWhiteSpace(input) then
            Error "Input cannot be empty"
        elif input.Length < 3 then
            Error "Input too short"
        elif input.Length > 100 then
            Error "Input too long"
        else
            Ok input
    
    let process value =
        match value with
        | n when n < 0 -> Error "Negative not allowed"
        | 0 -> Ok 0
        | n -> Ok (n * 2)
```

---

## 4. Coverage Thresholds

```xml
<!-- Global threshold ใน .fsproj -->
<PropertyGroup>
  <!-- Fail build if coverage below threshold -->
  <CoverletThreshold>80</CoverletThreshold>
  <CoverletThresholdType>line</CoverletThresholdType>  <!-- line, branch, method -->
  <CoverletThresholdStat>minimum</CoverletThresholdStat>
</PropertyGroup>
```

```bash
# Threshold ด้วย command line
dotnet test \
  /p:CollectCoverage=true \
  /p:CoverletOutputFormat=cobertura \
  /p:Threshold=80 \
  /p:ThresholdType=line \
  /p:ThresholdStat=minimum

# Multiple thresholds
dotnet test \
  /p:CollectCoverage=true \
  /p:CoverletOutputFormat=cobertura \
  /p:Threshold="80,60,90" \
  /p:ThresholdType="line,branch,method"
```

```yaml
# GitHub Actions CI กับ threshold
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Run tests with coverage
      run: |
        dotnet test \
          /p:CollectCoverage=true \
          /p:CoverletOutputFormat=cobertura \
          /p:CoverletOutput=./coverage/ \
          /p:Threshold=80 \
          /p:ThresholdType=line
    
    - name: Generate coverage report
      run: |
        dotnet tool install --global dotnet-reportgenerator-globaltool
        reportgenerator \
          -reports:"coverage/coverage.cobertura.xml" \
          -targetdir:"coverage-html" \
          -reporttypes:Html
    
    - name: Upload coverage report
      uses: actions/upload-artifact@v3
      with:
        name: coverage-report
        path: coverage-html/
```

---

## 5. SonarQube Integration

```yaml
# SonarQube CI Pipeline
name: SonarQube Analysis

on: [push]

jobs:
  sonar:
    runs-on: ubuntu-latest
    steps:
    - uses: actions/checkout@v3
      with:
        fetch-depth: 0  # SonarQube needs full history
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    - name: Install SonarScanner
      run: dotnet tool install --global dotnet-sonarscanner
    
    - name: Begin analysis
      run: |
        dotnet sonarscanner begin \
          /k:"my-project" \
          /d:sonar.host.url="https://sonarqube.mycompany.com" \
          /d:sonar.login="${{ secrets.SONAR_TOKEN }}" \
          /d:sonar.cs.opencover.reportsPaths="**/coverage.opencover.xml"
    
    - name: Build and test
      run: |
        dotnet build
        dotnet test \
          /p:CollectCoverage=true \
          /p:CoverletOutputFormat=opencover \
          /p:CoverletOutput=coverage/
    
    - name: End analysis
      run: |
        dotnet sonarscanner end \
          /d:sonar.login="${{ secrets.SONAR_TOKEN }}"
```

```xml
<!-- sonar-project.properties -->
sonar.projectKey=my-fsharp-project
sonar.projectName=My F# Project
sonar.projectVersion=1.0

sonar.sources=src
sonar.tests=tests
sonar.language=cs

# F# specific settings
sonar.fsharp.enable=true
sonar.exclusions=**/obj/**,**/bin/**,**/.git/**

# Coverage settings
sonar.cs.opencover.reportsPaths=**/coverage.opencover.xml
sonar.coverage.exclusions=**Tests*.fs,**/Migrations/**

# Quality gates
sonar.qualitygate.wait=true
```

---

## 6. F# Lint (FSharpLint)

### การติดตั้ง

```bash
# Global tool
dotnet tool install --global dotnet-fsharplint

# รัน lint
dotnet fsharplint lint MyProject.fsproj

# หรือ ทั้ง solution
dotnet fsharplint lint MyProject.sln
```

### Configuration

```json
// .fsharplint.json
{
  "UseDefaultConfiguration": true,
  "Analysers": {
    "Conventions": {
      "Naming": {
        "Enabled": true,
        "Rules": {
          "InterfaceNames": {
            "Enabled": true,
            "NamingUnderscores": "None",
            "Prefix": "I",
            "Suffix": ""
          },
          "ExceptionNames": {
            "Enabled": true,
            "Suffix": "Exception"
          }
        }
      }
    },
    "Hints": {
      "Enabled": true,
      "Hints": [
        {
          "Hint": "not (a = b) ===> a <> b",
          "ParseHints": true
        },
        {
          "Hint": "List.head (List.sort x) ===> List.min x",
          "ParseHints": true
        }
      ]
    }
  }
}
```

```fsharp
// Code ที่ lint จะตรวจสอบ

// Bad: ใช้ not (a = b) แทน a <> b
let isNotEqual a b = not (a = b)  // Lint warning

// Good:
let isNotEqual' a b = a <> b  // No warning

// Bad: ชื่อไม่ตรง convention
let myFunction = 42  // Should be let myFunction () = 42 or val myFunction: int

// Good naming conventions
type IUserService =  // Interface with I prefix
    abstract member GetUser: int -> string option

type DatabaseException(message: string) =  // Exception suffix
    inherit exn(message)

// Bad: unnecessary lambda
let doubled = [1..10] |> List.map (fun x -> double x)  // Can simplify

// Good:
let doubled' = [1..10] |> List.map double  // Simpler
```

---

## 7. Fantomas Code Formatter

### การติดตั้ง

```bash
# Global tool
dotnet tool install --global fantomas

# หรือ local tool
dotnet tool install fantomas

# Format single file
fantomas MyFile.fs

# Format directory
fantomas src/

# Check (ไม่เปลี่ยน file - เหมาะสำหรับ CI)
fantomas --check src/

# Format กับ config
fantomas --config .editorconfig src/
```

### Fantomas Configuration

```editorconfig
# .editorconfig
root = true

[*.{fs,fsi,fsx}]
indent_size = 4
indent_style = space
end_of_line = lf
insert_final_newline = true
max_line_length = 120

# Fantomas specific
fsharp_indent_on_try_with = true
fsharp_align_function_signature_to_indentation = false
fsharp_newline_between_type_definition_and_members = true
fsharp_keep_if_then_in_same_line = false
fsharp_max_if_then_else_short_width = 40
fsharp_max_value_binding_width = 60
fsharp_max_function_binding_width = 40
fsharp_max_dot_get_expression_width = 50
fsharp_bar_before_discriminated_union_declaration = true
fsharp_experimental_stroustrup_style = false
fsharp_experimental_keep_indent_in_branch = false
```

```fsharp
// ก่อน format
let add x y=x+y
let greet name= sprintf "Hello %s!" name
type Person={Name:string;Age:int}

// หลัง format (Fantomas จัดให้)
let add x y = x + y
let greet name = sprintf "Hello %s!" name

type Person = { Name: string; Age: int }
```

---

## 8. EditorConfig

```ini
# .editorconfig สำหรับ F# project

root = true

# All files
[*]
indent_style = space
end_of_line = lf
charset = utf-8
trim_trailing_whitespace = true
insert_final_newline = true

# F# files
[*.{fs,fsi,fsx}]
indent_size = 4
max_line_length = 120

# XML project files
[*.{csproj,fsproj,props,targets}]
indent_size = 2

# JSON files
[*.json]
indent_size = 2

# YAML files
[*.{yml,yaml}]
indent_size = 2

# Markdown
[*.md]
max_line_length = off
trim_trailing_whitespace = false
```

---

## 9. CI/CD Quality Gates

```yaml
# .github/workflows/quality.yml
name: Quality Gate

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  quality:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup .NET
      uses: actions/setup-dotnet@v3
      with:
        dotnet-version: '8.0.x'
    
    # 1. Build
    - name: Build
      run: dotnet build --no-restore -c Release
    
    # 2. Test + Coverage
    - name: Test with Coverage
      run: |
        dotnet test \
          --no-build \
          -c Release \
          /p:CollectCoverage=true \
          /p:CoverletOutputFormat=cobertura \
          /p:CoverletOutput=./coverage/ \
          /p:Threshold=80 \
          /p:ThresholdType=line \
          --verbosity normal
    
    # 3. Lint check
    - name: Install FSharpLint
      run: dotnet tool install --global dotnet-fsharplint
    
    - name: Lint
      run: dotnet fsharplint lint MyProject.sln
    
    # 4. Format check
    - name: Install Fantomas
      run: dotnet tool install --global fantomas
    
    - name: Check Formatting
      run: fantomas --check src/
    
    # 5. Upload coverage report
    - name: Upload Coverage
      uses: codecov/codecov-action@v3
      with:
        files: ./coverage/coverage.cobertura.xml
        fail_ci_if_error: true
        
    # 6. Comment coverage on PR
    - name: Coverage Comment
      uses: 5monkeys/cobertura-action@master
      with:
        path: coverage/coverage.cobertura.xml
        repo_token: ${{ secrets.GITHUB_TOKEN }}
        minimum_coverage: 80
```

---

## 10. Mutation Testing กับ Stryker

### การติดตั้ง

```bash
# ติดตั้ง Stryker.NET
dotnet tool install --global dotnet-stryker

# รัน mutation tests
dotnet stryker

# รันพร้อม config
dotnet stryker --config-file stryker-config.json
```

### Configuration

```json
// stryker-config.json
{
  "stryker-config": {
    "project": "MyProject/MyProject.fsproj",
    "test-projects": [
      "MyProject.Tests/MyProject.Tests.fsproj"
    ],
    "mutation-level": "Advanced",
    "reporters": [
      "progress",
      "html",
      "cleartext"
    ],
    "threshold-high": 80,
    "threshold-low": 60,
    "threshold-break": 50,
    "excluded-mutations": [
      "string"
    ],
    "ignore-mutations": [
      "arithmetic.addition"
    ],
    "mutate": [
      "src/**/*.fs",
      "!src/**/Generated/**"
    ]
  }
}
```

### ตัวอย่าง Mutation Testing

```fsharp
// Code ที่จะถูก mutate
module Calculator =
    let add x y = x + y         // Mutant: x - y
    let subtract x y = x - y    // Mutant: x + y
    let multiply x y = x * y    // Mutant: x / y
    let isPositive n = n > 0    // Mutant: n >= 0, n < 0

// Tests ที่ KILL mutants
// (Tests ที่ fail เมื่อ code ถูก mutate = good tests!)

[<Fact>]
let ``add kills mutation`` () =
    // add x y = x + y  -> mutant: x - y
    // Test: add 3 4 = 7, mutant: 3 - 4 = -1 (KILLED!)
    Assert.Equal(7, Calculator.add 3 4)
    Assert.Equal(-1, Calculator.add (-3) 2)  // Kills more mutants

[<Fact>]
let ``isPositive kills boundary mutation`` () =
    // isPositive n = n > 0  -> mutant: n >= 0
    // Test: isPositive 0 = false, mutant: 0 >= 0 = true (KILLED!)
    Assert.False(Calculator.isPositive 0)
    Assert.True(Calculator.isPositive 1)
    Assert.False(Calculator.isPositive (-1))

// Tests ที่ SURVIVE mutants
// (Tests ที่ pass แม้ code ถูก mutate = weak tests)
[<Fact>]
let ``weak test may survive mutations`` () =
    // Only tests one case
    Assert.Equal(5, Calculator.add 2 3)
    // Mutant: 2 - 3 = -1 ≠ 5 -> KILLED
    // But mutant: 5 (constant) would SURVIVE this test!
```

---

## 11. ตัวอย่าง Complete Quality Pipeline

```fsharp
// QualityDemo.fs
// ตัวอย่าง code ที่ผ่านทุก quality checks

module Domain.BankAccount

open System

// Types ที่ชัดเจน
type AccountId = AccountId of Guid
type Money = Money of decimal

type AccountError =
    | InsufficientFunds of required: decimal * available: decimal
    | AccountClosed
    | InvalidAmount of decimal
    | TransferToSameAccount

type Account = {
    Id: AccountId
    Balance: Money
    IsActive: bool
    CreatedAt: DateTimeOffset
}

// Pure functions - easy to test, lint, format
let createAccount () = {
    Id = AccountId (Guid.NewGuid())
    Balance = Money 0m
    IsActive = true
    CreatedAt = DateTimeOffset.UtcNow
}

let deposit (Money amount) account =
    if amount <= 0m then Error (InvalidAmount amount)
    elif not account.IsActive then Error AccountClosed
    else
        let (Money currentBalance) = account.Balance
        Ok { account with Balance = Money (currentBalance + amount) }

let withdraw (Money amount) account =
    if amount <= 0m then Error (InvalidAmount amount)
    elif not account.IsActive then Error AccountClosed
    else
        let (Money currentBalance) = account.Balance
        if currentBalance < amount then
            Error (InsufficientFunds (amount, currentBalance))
        else
            Ok { account with Balance = Money (currentBalance - amount) }

let transfer (Money amount) fromAccount toAccount =
    if fromAccount.Id = toAccount.Id then
        Error TransferToSameAccount
    else
        fromAccount
        |> withdraw (Money amount)
        |> Result.bind (fun updatedFrom ->
            toAccount
            |> deposit (Money amount)
            |> Result.map (fun updatedTo -> updatedFrom, updatedTo)
        )

// Tests ที่ครอบคลุม
module BankAccountTests =
    open Xunit
    open FsUnit.Xunit
    
    [<Fact>]
    let ``new account has zero balance`` () =
        let account = createAccount()
        account.Balance |> should equal (Money 0m)
    
    [<Fact>]
    let ``deposit increases balance`` () =
        let account = createAccount()
        let result = deposit (Money 100m) account
        
        result |> should be (ofCase <@ Ok @>)
        match result with
        | Ok updated -> updated.Balance |> should equal (Money 100m)
        | Error _ -> failwith "Expected Ok"
    
    [<Theory>]
    [<InlineData(0.0)>]
    [<InlineData(-100.0)>]
    let ``invalid deposit returns error`` (amount: float) =
        let account = createAccount()
        let result = deposit (Money (decimal amount)) account
        result |> should be (ofCase <@ Error @>)
    
    [<Fact>]
    let ``withdraw from empty account fails`` () =
        let account = createAccount()
        let result = withdraw (Money 100m) account
        
        match result with
        | Error (InsufficientFunds (required, available)) ->
            required |> should equal 100m
            available |> should equal 0m
        | _ -> failwith "Expected InsufficientFunds error"
    
    [<Fact>]
    let ``transfer moves money correctly`` () =
        let account1 = { createAccount() with Balance = Money 500m }
        let account2 = createAccount()
        
        match transfer (Money 200m) account1 account2 with
        | Ok (from', to') ->
            from'.Balance |> should equal (Money 300m)
            to'.Balance |> should equal (Money 200m)
        | Error e ->
            failwith (sprintf "Transfer failed: %A" e)
    
    [<Fact>]
    let ``cannot transfer to same account`` () =
        let account = { createAccount() with Balance = Money 500m }
        let result = transfer (Money 100m) account account
        result |> should equal (Error TransferToSameAccount)
```

---

## สรุป (Summary)

Code Quality tools สำหรับ F#:

1. **Coverlet** - รวบรวม code coverage metrics
2. **ReportGenerator** - สร้าง HTML coverage reports
3. **Coverage thresholds** - กำหนด minimum coverage
4. **SonarQube** - Static analysis และ quality gates
5. **FSharpLint** - F# specific linting rules
6. **Fantomas** - Consistent code formatting
7. **EditorConfig** - Editor-agnostic formatting rules
8. **Stryker** - Mutation testing validates test quality

### Quality Checklist

```bash
# รัน quality checks ทั้งหมด
#!/bin/bash

echo "Building..."
dotnet build -c Release

echo "Running tests with coverage..."
dotnet test /p:CollectCoverage=true /p:CoverletOutputFormat=cobertura /p:Threshold=80

echo "Checking formatting..."
fantomas --check src/

echo "Running lint..."
dotnet fsharplint lint src/

echo "Running mutation tests..."
dotnet stryker

echo "Quality check complete!"
```

ด้วยเครื่องมือเหล่านี้ทำให้แน่ใจว่า code มีคุณภาพสูง ทดสอบอย่างเพียงพอ และ consistent ตลอด codebase
