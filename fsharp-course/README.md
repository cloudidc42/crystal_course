# F# หลักสูตรครบวงจร - จากพื้นฐานถึงระดับโลก

## ภาพรวมหลักสูตร

หลักสูตรนี้ครอบคลุม F# programming ตั้งแต่ระดับเริ่มต้นจนถึงระดับ production-ready ประกอบด้วย 100 บทเรียนที่มีตัวอย่าง code ที่ใช้งานได้จริง คำอธิบายภาษาไทย และ English สำหรับ technical terms

---

## หลักสูตรนี้เหมาะสำหรับใคร

- นักพัฒนา .NET/C# ที่ต้องการเรียนรู้ Functional Programming
- นักพัฒนา Python/JavaScript ที่สนใจ strongly-typed functional language
- นักศึกษา Computer Science ที่ต้องการ practical experience
- นักพัฒนาที่ต้องการสร้าง production-grade F# applications
- ผู้ที่สนใจ Domain-Driven Design, Event Sourcing, CQRS
- Data Scientists ที่ต้องการ type-safe ML workflows

---

## Prerequisites

**ระดับ Beginner** (Part 1-30):
- ความรู้พื้นฐาน programming (variables, loops, functions)
- ไม่จำเป็นต้องรู้ F# หรือ Functional Programming มาก่อน
- รู้จัก concept ของ OOP เบื้องต้นจะช่วยได้

**ระดับ Intermediate** (Part 31-60):
- เข้าใจ Part 1-30 หรือมีความรู้ F# พื้นฐาน
- มีประสบการณ์สร้าง web applications
- รู้จัก REST APIs, databases เบื้องต้น

**ระดับ Advanced** (Part 61-100):
- เข้าใจ Part 1-60
- มีประสบการณ์ production development
- เข้าใจ distributed systems เบื้องต้น

---

## โครงสร้างหลักสูตรทั้ง 100 บท

### หมวดที่ 1: F# Fundamentals (Part 1-10)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 01 | Introduction to F# | ประวัติ, ทำไมต้อง F#, setup, Hello World |
| 02 | Values and Types | let bindings, basic types, type inference |
| 03 | Functions | function syntax, currying, partial application |
| 04 | Pattern Matching | match expressions, patterns, guards |
| 05 | Records and Tuples | value types, record syntax, destructuring |
| 06 | Discriminated Unions | sum types, option type, null safety |
| 07 | Lists and Collections | List, Array, Seq, Map, Set |
| 08 | Modules and Namespaces | code organization, open statements |
| 09 | Immutability | why immutability, mutable when needed |
| 10 | Recursion | tail recursion, accumulator pattern |

### หมวดที่ 2: Functional Programming Core (Part 11-20)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 11 | Higher-Order Functions | map, filter, fold, function composition |
| 12 | Computation Expressions | async, seq, custom builders |
| 13 | Error Handling | Result type, Railway-Oriented Programming |
| 14 | Async Programming | async workflows, Task integration |
| 15 | Option Type Deep Dive | option combinators, defaultArg |
| 16 | Function Composition | >>, <<, |>, piping style |
| 17 | Sequences and Lazy | seq, yield, infinite sequences |
| 18 | Active Patterns | complete/partial/parameterized |
| 19 | Units of Measure | dimensional analysis, type safety |
| 20 | Generic Types | generics, constraints, type parameters |

### หมวดที่ 3: Type System (Part 21-30)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 21 | Advanced Pattern Matching | nested patterns, when guards |
| 22 | Interfaces and OOP | implementing interfaces, inheritance |
| 23 | Type Extensions | augmentation, extension methods |
| 24 | Anonymous Records | lightweight records, structural typing |
| 25 | Struct Types | value types, performance |
| 26 | Computation Expression Advanced | custom builders, custom operations |
| 27 | Quotations Introduction | code as data, basic quotations |
| 28 | Reflection in F# | type information at runtime |
| 29 | Type Providers Intro | JSON, CSV, XML providers |
| 30 | Operator Overloading | custom operators, operator precedence |

### หมวดที่ 4: .NET Integration (Part 31-40)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 31 | .NET Interop | working with C# libraries |
| 32 | Collections Deep Dive | IEnumerable, LINQ, performance |
| 33 | String Processing | parsing, formatting, regex |
| 34 | DateTime and TimeZones | DateTime, DateTimeOffset, NodaTime |
| 35 | File I/O | reading, writing, streaming |
| 36 | JSON Serialization | System.Text.Json, Newtonsoft.Json |
| 37 | HTTP Client | HttpClient, HttpClientFactory |
| 38 | Dependency Injection | .NET DI container, lifetime |
| 39 | Configuration | appsettings, environment variables |
| 40 | Logging | Serilog, Microsoft.Extensions.Logging |

### หมวดที่ 5: Web Development (Part 41-50)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 41 | ASP.NET Core Basics | middleware, routing, controllers |
| 42 | Giraffe Framework | functional web framework |
| 43 | REST API Design | CRUD, versioning, documentation |
| 44 | Authentication | JWT, cookies, OAuth2 |
| 45 | Authorization | roles, policies, claims |
| 46 | Validation | FluentValidation, custom validators |
| 47 | Swagger/OpenAPI | API documentation |
| 48 | CORS and Security Headers | web security basics |
| 49 | Rate Limiting | protecting APIs |
| 50 | WebSockets | real-time with SignalR |

### หมวดที่ 6: Data Access (Part 51-60)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 51 | ADO.NET | raw database access |
| 52 | Dapper | micro-ORM for F# |
| 53 | Entity Framework Core | ORM with F# |
| 54 | PostgreSQL | Npgsql, F# specific patterns |
| 55 | SQLite | lightweight database |
| 56 | MongoDB | document database |
| 57 | Redis | caching, pub/sub |
| 58 | Migrations | database versioning |
| 59 | Repository Pattern | data access abstraction |
| 60 | Unit of Work | transaction management |

### หมวดที่ 7: Testing (Part 61-70)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 61 | Unit Testing | xUnit, NUnit, Expecto |
| 62 | FsUnit | F#-idiomatic assertions |
| 63 | FsCheck | property-based testing |
| 64 | Mocking | Moq, NSubstitute |
| 65 | Integration Testing | WebApplicationFactory |
| 66 | Test Doubles | stubs, fakes, mocks |
| 67 | BDD Testing | Gherkin, SpecFlow |
| 68 | Performance Testing | BenchmarkDotNet, k6 |
| 69 | Coverage and Quality | code coverage, analyzers |
| 70 | Test Architecture | test pyramid, strategies |

### หมวดที่ 8: Architecture Patterns (Part 71-80)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 71 | Domain-Driven Design | bounded contexts, aggregates |
| 72 | Event Sourcing | events as truth, replaying |
| 73 | CQRS | separating reads and writes |
| 74 | Clean Architecture | layers, dependencies |
| 75 | Microservices | service boundaries, communication |
| 76 | Event-Driven Architecture | events, reactions |
| 77 | Saga Pattern | distributed transactions |
| 78 | Hexagonal Architecture | ports and adapters |
| 79 | Functional Architecture | pure core, impure shell |
| 80 | Architecture Decision Records | documenting decisions |

### หมวดที่ 9: Advanced .NET (Part 81-90)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 81 | gRPC with F# | Protocol Buffers, streaming |
| 82 | Message Queues | RabbitMQ, Azure Service Bus |
| 83 | Background Services | IHostedService, workers |
| 84 | Caching Strategies | memory, distributed, CDN |
| 85 | Health Checks | liveness, readiness |
| 86 | Observability | metrics, tracing, logging |
| 87 | SignalR Real-time | websockets, groups |
| 88 | GraphQL | HotChocolate with F# |
| 89 | OData | REST with querying |
| 90 | gRPC Streaming | server/client/bi-directional |

### หมวดที่ 10: Expert Level (Part 91-100)

| Part | หัวข้อ | รายละเอียด |
|------|--------|-----------|
| 91 | Advanced Type Providers | SQLProvider, custom providers, ProvidedTypes SDK |
| 92 | Advanced Functional Patterns | Functor, Applicative, Monad, Free Monad |
| 93 | DSL Design | computation expressions, operator overloading |
| 94 | Performance Optimization | profiling, SIMD, unsafe code, Span<T> |
| 95 | Security | OWASP, cryptography, JWT, rate limiting |
| 96 | Deployment & DevOps | Docker, Kubernetes, GitHub Actions, Helm |
| 97 | Cloud Services | Azure SDK, AWS SDK, cloud patterns |
| 98 | Machine Learning | ML.NET, DiffSharp, TorchSharp |
| 99 | Real-World Project | Complete E-Commerce API |
| 100 | Advanced Topics | Effect systems, metaprogramming, next steps |

---

## วิธีใช้หลักสูตรนี้

### Beginner Path (3-6 เดือน)
```
Part 1-10 → Part 11-20 → Part 21-30 → Part 31-40 → Part 41-50
```
เน้น: อ่านทุก example, run code ทุกอัน, ทำแบบฝึกหัดที่ให้

### Web Developer Path (2-4 เดือน)
```
Part 1-20 → Part 41-50 → Part 51-60 → Part 71-75 → Part 95-96
```
เน้น: สร้าง web API ขนาดเล็ก, เพิ่ม features ทีละขั้น

### Data/ML Path (2-3 เดือน)
```
Part 1-20 → Part 29 → Part 91 → Part 98 → Part 100
```
เน้น: Type Providers, ML.NET, data pipelines

### Architect Path (4-6 เดือน)
```
Part 1-30 → Part 71-80 → Part 91-93 → Part 99 → Part 100
```
เน้น: DDD, Event Sourcing, CQRS, architectural patterns

---

## Environment Setup

### ติดตั้ง .NET SDK
```bash
# macOS
brew install dotnet

# Windows
winget install Microsoft.DotNet.SDK.9

# Linux
wget https://dot.net/v1/dotnet-install.sh
bash dotnet-install.sh --version latest
```

### ติดตั้ง Editor
```bash
# VS Code (แนะนำ)
code --install-extension ionide.ionide-fsharp

# JetBrains Rider (powerful IDE)
# ดาวน์โหลดที่ jetbrains.com/rider

# Visual Studio 2022 (Windows)
# รวม F# support อัตโนมัติ
```

### สร้าง F# Project
```bash
# Console application
dotnet new console -lang F# -n MyFirstFSharpApp

# Web API
dotnet new webapi -lang F# -n MyWebApi

# Library
dotnet new classlib -lang F# -n MyLibrary

# Test project
dotnet new xunit -lang F# -n MyTests

# รัน project
cd MyFirstFSharpApp
dotnet run

# สร้าง solution
dotnet new sln -n MySolution
dotnet sln add MyFirstFSharpApp/MyFirstFSharpApp.fsproj
```

### Useful .fsproj configuration
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net9.0</TargetFramework>
    <RootNamespace>MyApp</RootNamespace>
    <Nullable>enable</Nullable>
    <LangVersion>preview</LangVersion>
    <TreatWarningsAsErrors>false</TreatWarningsAsErrors>
    <Optimize>true</Optimize>
  </PropertyGroup>

  <ItemGroup>
    <!-- F# files must be listed in compilation order -->
    <Compile Include="Types.fs" />
    <Compile Include="Domain.fs" />
    <Compile Include="Services.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## Essential NuGet Packages

```xml
<!-- Core utilities -->
<PackageReference Include="FSharpPlus" Version="4.4.0" />
<PackageReference Include="FSharp.Collections.ParallelSeq" Version="1.2.0" />

<!-- Web -->
<PackageReference Include="Giraffe" Version="7.0.0" />
<PackageReference Include="Falco" Version="4.0.0" />

<!-- Database -->
<PackageReference Include="Npgsql.FSharp" Version="5.7.0" />
<PackageReference Include="Dapper.FSharp" Version="2.2.0" />

<!-- JSON -->
<PackageReference Include="FSharp.SystemTextJson" Version="1.3.0" />
<PackageReference Include="Thoth.Json.Net" Version="11.0.0" />

<!-- Testing -->
<PackageReference Include="xunit" Version="2.6.0" />
<PackageReference Include="FsUnit.xUnit" Version="6.0.0" />
<PackageReference Include="FsCheck.Xunit" Version="3.0.0" />
<PackageReference Include="Expecto" Version="10.1.0" />

<!-- Type Providers -->
<PackageReference Include="FSharp.Data" Version="6.4.0" />
<PackageReference Include="SQLProvider" Version="1.3.37" />

<!-- Logging -->
<PackageReference Include="Serilog.AspNetCore" Version="8.0.0" />
<PackageReference Include="Serilog.Sinks.Console" Version="5.0.0" />

<!-- Utilities -->
<PackageReference Include="Polly" Version="8.2.0" />
<PackageReference Include="FluentValidation" Version="11.8.0" />
```

---

## Contributing

หากต้องการ contribute ให้กับหลักสูตรนี้:

1. **แก้ไข typos หรือ errors**: สร้าง Pull Request พร้อม description
2. **เพิ่ม examples**: เพิ่ม code examples ที่ practical และ runnable
3. **แปลเพิ่มเติม**: ช่วย translate technical terms
4. **แจ้ง issues**: หาก code ใดไม่ compile หรือผิดพลาด

### Guidelines สำหรับ contribution:
- Code ทุกอัน ต้องรัน compile ได้จริง
- ใช้ F# 8/9 syntax
- มีคำอธิบายภาษาไทยสำหรับ concepts สำคัญ
- ทำ test examples ให้ชัดเจน
- หลีกเลี่ยง anti-patterns

---

## License

หลักสูตรนี้เผยแพร่ภายใต้ MIT License - ใช้ได้อย่างเสรีสำหรับการศึกษาและการสอน

---

## ผู้จัดทำ

สร้างขึ้นเพื่อ F# Community ภาษาไทยและนักพัฒนาที่ต้องการเรียน Functional Programming ใน .NET ecosystem

---

*"Make illegal states unrepresentable" - Scott Wlaschin*

*"The goal of F# is to let you think about your problem domain, rather than fighting with the language" - Don Syme*
