# Part 75 - Integration Testing

## บทนำ (Introduction)

Integration Testing ทดสอบการทำงานร่วมกันของหลาย components เช่น database, external APIs, และ web services

ต่างจาก Unit Testing ที่ทดสอบ component เดียวแบบ isolated, Integration Testing ทดสอบ:
- Web API endpoints
- Database operations
- External service integrations
- Full request-response cycle

---

## 1. Integration Test Setup

### โครงสร้าง Project

```
MyProject/
├── src/
│   ├── MyProject.Api/         # Web API
│   │   ├── Controllers/
│   │   ├── Program.fs
│   │   └── MyProject.Api.fsproj
│   └── MyProject.Domain/      # Domain logic
│       └── MyProject.Domain.fsproj
└── tests/
    └── MyProject.IntegrationTests/
        ├── ApiTests.fs
        ├── DatabaseTests.fs
        └── MyProject.IntegrationTests.fsproj
```

### .fsproj สำหรับ Integration Tests

```xml
<!-- MyProject.IntegrationTests.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.8.0" />
    <PackageReference Include="xunit" Version="2.6.2" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.5.3">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    
    <!-- ASP.NET Core test server -->
    <PackageReference Include="Microsoft.AspNetCore.Mvc.Testing" Version="8.0.0" />
    
    <!-- Testcontainers for Docker -->
    <PackageReference Include="Testcontainers" Version="3.6.0" />
    <PackageReference Include="Testcontainers.PostgreSql" Version="3.6.0" />
    <PackageReference Include="Testcontainers.Redis" Version="3.6.0" />
    
    <!-- Database -->
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="8.0.0" />
    
    <!-- HTTP client utilities -->
    <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <ProjectReference Include="../../src/MyProject.Api/MyProject.Api.fsproj" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="TestDataBuilders.fs" />
    <Compile Include="ApiTests.fs" />
    <Compile Include="DatabaseTests.fs" />
    <Compile Include="ContainerTests.fs" />
  </ItemGroup>
</Project>
```

---

## 2. WebApplicationFactory สำหรับ API Testing

```fsharp
// WebApiTests.fs
module MyProject.IntegrationTests.WebApiTests

open System
open System.Net
open System.Net.Http
open System.Net.Http.Json
open Microsoft.AspNetCore.Mvc.Testing
open Microsoft.Extensions.DependencyInjection
open Xunit

// สมมติว่า Program.fs ของ API มีลักษณะนี้
// open Microsoft.AspNetCore.Builder
// open Microsoft.Extensions.DependencyInjection
// let builder = WebApplication.CreateBuilder(args)
// // ... configuration
// let app = builder.Build()
// // ... middleware setup
// app.Run()

// Custom WebApplicationFactory สำหรับ integration tests
type TestWebAppFactory() =
    inherit WebApplicationFactory<MyProject.Api.Program>()
    
    override _.ConfigureWebHost(builder) =
        builder.ConfigureServices(fun services ->
            // Override services for testing
            // เช่น ใช้ in-memory database แทน real database
            services.AddSingleton<IConfiguration>(
                ConfigurationBuilder()
                    .AddInMemoryCollection([
                        KeyValuePair("ConnectionStrings:Default", "Data Source=:memory:")
                    ])
                    .Build()
            ) |> ignore
        ) |> ignore

// Test class
type ProductApiTests(factory: TestWebAppFactory) =
    interface IClassFixture<TestWebAppFactory>
    
    let client = factory.CreateClient()
    
    [<Fact>]
    member _.``GET /api/products returns 200`` () = task {
        let! response = client.GetAsync("/api/products")
        Assert.Equal(HttpStatusCode.OK, response.StatusCode)
    }
    
    [<Fact>]
    member _.``GET /api/products returns list`` () = task {
        let! response = client.GetAsync("/api/products")
        let! products = response.Content.ReadFromJsonAsync<{| Id: int; Name: string |}[]>()
        Assert.NotNull(products)
    }
    
    [<Fact>]
    member _.``POST /api/products creates product`` () = task {
        let newProduct = {| Name = "Test Product"; Price = 9.99 |}
        let! response = client.PostAsJsonAsync("/api/products", newProduct)
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode)
        
        let! created = response.Content.ReadFromJsonAsync<{| Id: int; Name: string; Price: float |}>()
        Assert.Equal("Test Product", created.Name)
        Assert.True(created.Id > 0)
    }
    
    [<Fact>]
    member _.``GET /api/products/:id returns 404 for missing`` () = task {
        let! response = client.GetAsync("/api/products/99999")
        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode)
    }
    
    [<Fact>]
    member _.``PUT /api/products/:id updates product`` () = task {
        // First create
        let newProduct = {| Name = "Original Name"; Price = 10.0 |}
        let! createResponse = client.PostAsJsonAsync("/api/products", newProduct)
        let! created = createResponse.Content.ReadFromJsonAsync<{| Id: int; Name: string |}>()
        
        // Then update
        let updateData = {| Name = "Updated Name"; Price = 20.0 |}
        let! updateResponse = client.PutAsJsonAsync($"/api/products/{created.Id}", updateData)
        
        Assert.Equal(HttpStatusCode.OK, updateResponse.StatusCode)
    }
    
    [<Fact>]
    member _.``DELETE /api/products/:id deletes product`` () = task {
        // First create
        let newProduct = {| Name = "To Delete"; Price = 5.0 |}
        let! createResponse = client.PostAsJsonAsync("/api/products", newProduct)
        let! created = createResponse.Content.ReadFromJsonAsync<{| Id: int |}>()
        
        // Then delete
        let! deleteResponse = client.DeleteAsync($"/api/products/{created.Id}")
        Assert.Equal(HttpStatusCode.NoContent, deleteResponse.StatusCode)
        
        // Verify deleted
        let! getResponse = client.GetAsync($"/api/products/{created.Id}")
        Assert.Equal(HttpStatusCode.NotFound, getResponse.StatusCode)
    }

// Custom HttpClient options
type CustomClientTests(factory: TestWebAppFactory) =
    interface IClassFixture<TestWebAppFactory>
    
    let client = 
        factory.CreateClient(
            WebApplicationFactoryClientOptions(
                BaseAddress = Uri("http://localhost"),
                AllowAutoRedirect = false
            )
        )
    
    [<Fact>]
    member _.``api returns correct content type`` () = task {
        let! response = client.GetAsync("/api/products")
        Assert.Equal("application/json", response.Content.Headers.ContentType.MediaType)
    }
```

---

## 3. TestServer

```fsharp
// TestServerTests.fs
module MyProject.IntegrationTests.TestServerTests

open System.Net.Http
open Microsoft.AspNetCore.TestHost
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting
open Xunit

// สร้าง TestServer โดยตรง
let createTestServer () =
    let builder = WebHostBuilder()
                    .ConfigureServices(fun services ->
                        services.AddRouting() |> ignore
                        services.AddControllers() |> ignore
                    )
                    .Configure(fun app ->
                        app.UseRouting() |> ignore
                        app.UseEndpoints(fun endpoints ->
                            endpoints.MapControllers() |> ignore
                        ) |> ignore
                    )
    new TestServer(builder)

// Integration tests กับ TestServer
type TestServerTests() =
    let server = createTestServer()
    let client = server.CreateClient()
    
    interface System.IDisposable with
        member _.Dispose() =
            client.Dispose()
            server.Dispose()
    
    [<Fact>]
    member _.``health endpoint returns ok`` () = task {
        let! response = client.GetAsync("/health")
        response.EnsureSuccessStatusCode() |> ignore
    }

// Minimal API test server
let createMinimalApiServer () =
    let builder = WebApplication.CreateBuilder([||])
    
    builder.Services.AddEndpointsApiExplorer() |> ignore
    
    let app = builder.Build()
    
    // Define endpoints
    app.MapGet("/echo/{message}", fun (message: string) ->
        {| Echo = message |}
    ) |> ignore
    
    app.MapGet("/sum", fun (a: int) (b: int) ->
        {| Result = a + b |}
    ) |> ignore
    
    app

// Test สำหรับ minimal API
type MinimalApiTests() =
    // Note: In real code, you'd use WebApplicationFactory
    
    [<Fact>]
    let ``echo endpoint returns message`` () = task {
        // This is conceptual - real test would use WebApplicationFactory
        let message = "hello-world"
        // Test would verify: GET /echo/hello-world returns { Echo: "hello-world" }
        Assert.True(true, "Echo endpoint test")
    }
```

---

## 4. Database Integration Tests

```fsharp
// DatabaseTests.fs
module MyProject.IntegrationTests.DatabaseTests

open Microsoft.EntityFrameworkCore
open Xunit
open System

// Domain models
[<CLIMutable>]
type User = {
    mutable Id: int
    mutable Name: string
    mutable Email: string
    mutable CreatedAt: DateTime
}

[<CLIMutable>]
type Order = {
    mutable Id: int
    mutable UserId: int
    mutable Total: float
    mutable Status: string
    mutable CreatedAt: DateTime
}

// DbContext
type AppDbContext(options: DbContextOptions<AppDbContext>) =
    inherit DbContext(options)
    
    [<DefaultValue>] 
    val mutable private users: DbSet<User>
    member this.Users with get() = this.users and set v = this.users <- v
    
    [<DefaultValue>]
    val mutable private orders: DbSet<Order>
    member this.Orders with get() = this.orders and set v = this.orders <- v
    
    override _.OnModelCreating(modelBuilder) =
        modelBuilder.Entity<User>()
            .HasKey(fun u -> u.Id :> obj)
        |> ignore

// In-memory database tests
type InMemoryDatabaseTests() =
    let createContext () =
        let options = 
            DbContextOptionsBuilder<AppDbContext>()
                .UseInMemoryDatabase(Guid.NewGuid().ToString())
                .Options
        new AppDbContext(options)
    
    [<Fact>]
    member _.``can add and retrieve user`` () = task {
        use ctx = createContext()
        do! ctx.Database.EnsureCreatedAsync() :> System.Threading.Tasks.Task
        
        let user = {
            Id = 0
            Name = "Alice"
            Email = "alice@example.com"
            CreatedAt = DateTime.UtcNow
        }
        
        ctx.Users.Add(user) |> ignore
        let! _ = ctx.SaveChangesAsync()
        
        let! found = ctx.Users.FirstOrDefaultAsync(fun u -> u.Email = "alice@example.com")
        
        Assert.NotNull(found)
        Assert.Equal("Alice", found.Name)
    }
    
    [<Fact>]
    member _.``can update user`` () = task {
        use ctx = createContext()
        
        let user = {
            Id = 0
            Name = "Bob"
            Email = "bob@example.com"
            CreatedAt = DateTime.UtcNow
        }
        
        ctx.Users.Add(user) |> ignore
        let! _ = ctx.SaveChangesAsync()
        
        let! found = ctx.Users.FirstAsync(fun u -> u.Email = "bob@example.com")
        found.Name <- "Robert"
        let! _ = ctx.SaveChangesAsync()
        
        let! updated = ctx.Users.FirstAsync(fun u -> u.Email = "bob@example.com")
        Assert.Equal("Robert", updated.Name)
    }
    
    [<Fact>]
    member _.``can delete user`` () = task {
        use ctx = createContext()
        
        let user = {
            Id = 0
            Name = "Charlie"
            Email = "charlie@example.com"
            CreatedAt = DateTime.UtcNow
        }
        
        ctx.Users.Add(user) |> ignore
        let! _ = ctx.SaveChangesAsync()
        
        let! toDelete = ctx.Users.FirstAsync(fun u -> u.Email = "charlie@example.com")
        ctx.Users.Remove(toDelete) |> ignore
        let! _ = ctx.SaveChangesAsync()
        
        let! count = ctx.Users.CountAsync(fun u -> u.Email = "charlie@example.com")
        Assert.Equal(0, count)
    }
    
    [<Fact>]
    member _.``can query users`` () = task {
        use ctx = createContext()
        
        let users = [
            { Id = 0; Name = "User1"; Email = "user1@example.com"; CreatedAt = DateTime.UtcNow }
            { Id = 0; Name = "User2"; Email = "user2@example.com"; CreatedAt = DateTime.UtcNow }
            { Id = 0; Name = "User3"; Email = "user3@example.com"; CreatedAt = DateTime.UtcNow }
        ]
        
        users |> List.iter (fun u -> ctx.Users.Add(u) |> ignore)
        let! _ = ctx.SaveChangesAsync()
        
        let! count = ctx.Users.CountAsync()
        Assert.Equal(3, count)
    }
```

---

## 5. Testcontainers - PostgreSQL

```fsharp
// PostgreSqlContainerTests.fs
module MyProject.IntegrationTests.PostgreSqlTests

open Xunit
open Testcontainers.PostgreSql
open Npgsql
open System

// PostgreSQL container fixture
type PostgreSqlFixture() =
    let container =
        PostgreSqlBuilder()
            .WithDatabase("testdb")
            .WithUsername("testuser")
            .WithPassword("testpass")
            .Build()
    
    member _.Container = container
    
    member _.ConnectionString = container.GetConnectionString()
    
    interface System.IAsyncLifetime with
        member this.InitializeAsync() = task {
            do! container.StartAsync()
            
            // Create tables
            use conn = new NpgsqlConnection(this.ConnectionString)
            do! conn.OpenAsync()
            
            use cmd = conn.CreateCommand()
            cmd.CommandText <- """
                CREATE TABLE IF NOT EXISTS products (
                    id SERIAL PRIMARY KEY,
                    name VARCHAR(100) NOT NULL,
                    price DECIMAL(10,2) NOT NULL,
                    created_at TIMESTAMP DEFAULT NOW()
                );
                
                CREATE TABLE IF NOT EXISTS categories (
                    id SERIAL PRIMARY KEY,
                    name VARCHAR(50) NOT NULL UNIQUE
                );
                
                ALTER TABLE products 
                ADD COLUMN IF NOT EXISTS category_id INTEGER REFERENCES categories(id);
            """
            do! cmd.ExecuteNonQueryAsync() :> System.Threading.Tasks.Task
        }
        
        member this.DisposeAsync() = task {
            do! container.StopAsync()
            do! container.DisposeAsync()
        }

// Tests ที่ใช้ PostgreSQL container
type PostgreSqlTests(fixture: PostgreSqlFixture) =
    interface IClassFixture<PostgreSqlFixture>
    
    let getConnection () = 
        let conn = new NpgsqlConnection(fixture.ConnectionString)
        conn.Open()
        conn
    
    [<Fact>]
    member _.``can insert and retrieve product`` () = task {
        use conn = new NpgsqlConnection(fixture.ConnectionString)
        do! conn.OpenAsync()
        
        // Insert
        use insertCmd = conn.CreateCommand()
        insertCmd.CommandText <- "INSERT INTO products (name, price) VALUES (@name, @price) RETURNING id"
        insertCmd.Parameters.AddWithValue("@name", "Test Product") |> ignore
        insertCmd.Parameters.AddWithValue("@price", 9.99) |> ignore
        let! id = insertCmd.ExecuteScalarAsync()
        
        Assert.True(Convert.ToInt32(id) > 0)
        
        // Retrieve
        use selectCmd = conn.CreateCommand()
        selectCmd.CommandText <- "SELECT name, price FROM products WHERE id = @id"
        selectCmd.Parameters.AddWithValue("@id", id) |> ignore
        
        use! reader = selectCmd.ExecuteReaderAsync()
        let! hasRow = reader.ReadAsync()
        
        Assert.True(hasRow)
        Assert.Equal("Test Product", reader.GetString(0))
        Assert.Equal(9.99, reader.GetDouble(1), 2)
    }
    
    [<Fact>]
    member _.``can query multiple products`` () = task {
        use conn = new NpgsqlConnection(fixture.ConnectionString)
        do! conn.OpenAsync()
        
        // Insert multiple
        for i in 1..5 do
            use cmd = conn.CreateCommand()
            cmd.CommandText <- "INSERT INTO products (name, price) VALUES (@name, @price)"
            cmd.Parameters.AddWithValue("@name", sprintf "Product %d" i) |> ignore
            cmd.Parameters.AddWithValue("@price", float i * 10.0) |> ignore
            do! cmd.ExecuteNonQueryAsync() :> System.Threading.Tasks.Task
        
        // Count
        use countCmd = conn.CreateCommand()
        countCmd.CommandText <- "SELECT COUNT(*) FROM products"
        let! count = countCmd.ExecuteScalarAsync()
        
        Assert.True(Convert.ToInt64(count) >= 5L)
    }
    
    [<Fact>]
    member _.``can perform transaction`` () = task {
        use conn = new NpgsqlConnection(fixture.ConnectionString)
        do! conn.OpenAsync()
        
        use! tx = conn.BeginTransactionAsync()
        
        try
            use cmd1 = conn.CreateCommand()
            cmd1.Transaction <- tx
            cmd1.CommandText <- "INSERT INTO products (name, price) VALUES ('TxProduct1', 1.0)"
            do! cmd1.ExecuteNonQueryAsync() :> System.Threading.Tasks.Task
            
            use cmd2 = conn.CreateCommand()
            cmd2.Transaction <- tx
            cmd2.CommandText <- "INSERT INTO products (name, price) VALUES ('TxProduct2', 2.0)"
            do! cmd2.ExecuteNonQueryAsync() :> System.Threading.Tasks.Task
            
            do! tx.CommitAsync()
            
            // Verify both inserted
            use checkCmd = conn.CreateCommand()
            checkCmd.CommandText <- "SELECT COUNT(*) FROM products WHERE name LIKE 'TxProduct%'"
            let! count = checkCmd.ExecuteScalarAsync()
            Assert.True(Convert.ToInt64(count) >= 2L)
            
        with ex ->
            do! tx.RollbackAsync()
            raise ex
    }
```

---

## 6. Redis Container Tests

```fsharp
// RedisContainerTests.fs
module MyProject.IntegrationTests.RedisTests

open Xunit
open Testcontainers.Redis
open StackExchange.Redis
open System

// Redis container fixture
type RedisFixture() =
    let container =
        RedisBuilder()
            .WithImage("redis:7-alpine")
            .Build()
    
    member _.Container = container
    
    let mutable connectionMultiplexer: ConnectionMultiplexer = null
    
    member _.GetDatabase() = connectionMultiplexer.GetDatabase()
    
    interface System.IAsyncLifetime with
        member _.InitializeAsync() = task {
            do! container.StartAsync()
            connectionMultiplexer <- 
                ConnectionMultiplexer.Connect(container.GetConnectionString())
        }
        
        member _.DisposeAsync() = task {
            if connectionMultiplexer <> null then
                do! connectionMultiplexer.CloseAsync()
                connectionMultiplexer.Dispose()
            do! container.StopAsync()
            do! container.DisposeAsync()
        }

// Redis tests
type RedisTests(fixture: RedisFixture) =
    interface IClassFixture<RedisFixture>
    
    [<Fact>]
    member _.``can set and get string value`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:string:%s" (Guid.NewGuid().ToString("N"))
        
        do! db.StringSetAsync(key, "hello redis") :> System.Threading.Tasks.Task
        let! value = db.StringGetAsync(key)
        
        Assert.Equal("hello redis", value.ToString())
    }
    
    [<Fact>]
    member _.``can set value with expiry`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:expiry:%s" (Guid.NewGuid().ToString("N"))
        
        do! db.StringSetAsync(key, "expires soon", TimeSpan.FromSeconds(1.0)) :> System.Threading.Tasks.Task
        
        let! valueBefore = db.StringGetAsync(key)
        Assert.True(valueBefore.HasValue)
        
        // In real test, we'd wait but for brevity just check TTL
        let! ttl = db.KeyTimeToLiveAsync(key)
        Assert.True(ttl.HasValue && ttl.Value.TotalSeconds <= 1.0)
    }
    
    [<Fact>]
    member _.``can increment counter`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:counter:%s" (Guid.NewGuid().ToString("N"))
        
        let! v1 = db.StringIncrementAsync(key)
        let! v2 = db.StringIncrementAsync(key)
        let! v3 = db.StringIncrementAsync(key)
        
        Assert.Equal(1L, v1)
        Assert.Equal(2L, v2)
        Assert.Equal(3L, v3)
    }
    
    [<Fact>]
    member _.``can work with hash`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:hash:%s" (Guid.NewGuid().ToString("N"))
        
        do! db.HashSetAsync(key, [|
            HashEntry("name", "Alice")
            HashEntry("age", "30")
            HashEntry("email", "alice@example.com")
        |]) :> System.Threading.Tasks.Task
        
        let! name = db.HashGetAsync(key, "name")
        let! age = db.HashGetAsync(key, "age")
        
        Assert.Equal("Alice", name.ToString())
        Assert.Equal("30", age.ToString())
    }
    
    [<Fact>]
    member _.``can work with list`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:list:%s" (Guid.NewGuid().ToString("N"))
        
        do! db.ListRightPushAsync(key, "item1") :> System.Threading.Tasks.Task
        do! db.ListRightPushAsync(key, "item2") :> System.Threading.Tasks.Task
        do! db.ListRightPushAsync(key, "item3") :> System.Threading.Tasks.Task
        
        let! length = db.ListLengthAsync(key)
        Assert.Equal(3L, length)
        
        let! first = db.ListLeftPopAsync(key)
        Assert.Equal("item1", first.ToString())
    }
    
    [<Fact>]
    member _.``can work with set`` () = task {
        let db = fixture.GetDatabase()
        let key = sprintf "test:set:%s" (Guid.NewGuid().ToString("N"))
        
        do! db.SetAddAsync(key, "a") :> System.Threading.Tasks.Task
        do! db.SetAddAsync(key, "b") :> System.Threading.Tasks.Task
        do! db.SetAddAsync(key, "c") :> System.Threading.Tasks.Task
        do! db.SetAddAsync(key, "a") :> System.Threading.Tasks.Task  // Duplicate
        
        let! count = db.SetLengthAsync(key)
        Assert.Equal(3L, count)  // Should be 3, not 4
    }
```

---

## 7. Cleanup Strategies

```fsharp
// CleanupTests.fs
module MyProject.IntegrationTests.CleanupTests

open Xunit
open System

// Strategy 1: Per-test cleanup กับ IDisposable
type PerTestCleanupTests() =
    let mutable createdIds: int list = []
    
    // Record created IDs for cleanup
    let trackId id = 
        createdIds <- id :: createdIds
        id
    
    interface IDisposable with
        member _.Dispose() =
            // Cleanup all created records
            for id in createdIds do
                // deleteRecord id
                printfn "Cleaning up record %d" id
    
    [<Fact>]
    member _.``creates and tracks record for cleanup`` () =
        let id = trackId 1  // Simulate creating a record
        Assert.Equal(1, id)
        Assert.Contains(1, createdIds)

// Strategy 2: Database cleanup กับ transaction rollback
type TransactionRollbackTests() =
    // เปิด transaction ก่อนแต่ละ test
    // Rollback หลังจบ test
    // ทำให้ database กลับสู่สถานะเดิม
    
    [<Fact>]
    member _.``test with transaction rollback`` () =
        // In real implementation:
        // begin transaction
        // ... do operations ...
        // rollback (not commit)
        // Database is back to original state
        Assert.True(true, "Transaction test")

// Strategy 3: Test isolation กับ unique prefixes
type TestIsolationTests() =
    let testId = Guid.NewGuid().ToString("N").[..7]
    let prefix = sprintf "test_%s_" testId
    
    [<Fact>]
    member _.``creates uniquely prefixed resources`` () =
        let resourceName = sprintf "%smy_resource" prefix
        // All resources created with unique prefix
        // Can be safely cleaned up by prefix
        Assert.StartsWith(prefix, resourceName)

// Strategy 4: Docker container cleanup
type ContainerCleanupFixture() =
    // Container starts fresh for each test run
    // Entire container is thrown away after tests
    
    let mutable containerStarted = false
    
    interface System.IAsyncLifetime with
        member _.InitializeAsync() = task {
            containerStarted <- true
            printfn "Container started"
        }
        
        member _.DisposeAsync() = task {
            printfn "Container stopped and removed"
        }
    
    member _.IsStarted = containerStarted
```

---

## 8. Test Data Builders

```fsharp
// TestDataBuilders.fs
module MyProject.IntegrationTests.TestDataBuilders

open System

// Domain types
type Address = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

type Customer = {
    Id: int
    Name: string
    Email: string
    Phone: string option
    Address: Address
    CreatedAt: DateTime
    IsActive: bool
}

type OrderLine = {
    ProductId: int
    ProductName: string
    Quantity: int
    UnitPrice: float
}

type Order = {
    Id: int
    CustomerId: int
    Lines: OrderLine list
    Status: string
    CreatedAt: DateTime
    ShippedAt: DateTime option
}

// Builder สำหรับ Address
type AddressBuilder() =
    let mutable street = "123 Main St"
    let mutable city = "Springfield"
    let mutable country = "US"
    let mutable postalCode = "12345"
    
    member _.WithStreet(s) = street <- s; ()
    member _.WithCity(c) = city <- c; ()
    member _.WithCountry(c) = country <- c; ()
    member _.WithPostalCode(p) = postalCode <- p; ()
    
    member _.Build() = {
        Street = street
        City = city
        Country = country
        PostalCode = postalCode
    }

// Builder สำหรับ Customer
type CustomerBuilder() =
    let mutable id = 1
    let mutable name = "John Doe"
    let mutable email = "john@example.com"
    let mutable phone: string option = None
    let mutable address = AddressBuilder().Build()
    let mutable createdAt = DateTime.UtcNow
    let mutable isActive = true
    
    member this.WithId(i) = id <- i; this
    member this.WithName(n) = name <- n; this
    member this.WithEmail(e) = email <- e; this
    member this.WithPhone(p) = phone <- Some p; this
    member this.WithAddress(a: Address) = address <- a; this
    member this.WithAddress(builder: AddressBuilder) = address <- builder.Build(); this
    member this.WithCreatedAt(dt) = createdAt <- dt; this
    member this.AsInactive() = isActive <- false; this
    
    member _.Build() = {
        Id = id
        Name = name
        Email = email
        Phone = phone
        Address = address
        CreatedAt = createdAt
        IsActive = isActive
    }

// Builder สำหรับ Order
type OrderLineBuilder() =
    let mutable productId = 1
    let mutable productName = "Test Product"
    let mutable quantity = 1
    let mutable unitPrice = 9.99
    
    member this.WithProduct(id, name) = productId <- id; productName <- name; this
    member this.WithQuantity(q) = quantity <- q; this
    member this.WithPrice(p) = unitPrice <- p; this
    
    member _.Build() = {
        ProductId = productId
        ProductName = productName
        Quantity = quantity
        UnitPrice = unitPrice
    }

type OrderBuilder() =
    let mutable id = 1
    let mutable customerId = 1
    let mutable lines = [OrderLineBuilder().Build()]
    let mutable status = "pending"
    let mutable createdAt = DateTime.UtcNow
    let mutable shippedAt: DateTime option = None
    
    member this.WithId(i) = id <- i; this
    member this.WithCustomer(cId) = customerId <- cId; this
    member this.WithLines(l) = lines <- l; this
    member this.WithStatus(s) = status <- s; this
    member this.AsShipped() =
        status <- "shipped"
        shippedAt <- Some DateTime.UtcNow
        this
    
    member _.Build() = {
        Id = id
        CustomerId = customerId
        Lines = lines
        Status = status
        CreatedAt = createdAt
        ShippedAt = shippedAt
    }

// ใช้งาน builders ใน tests
module BuilderUsageExamples =
    open Xunit
    
    [<Fact>]
    let ``builder creates valid customer`` () =
        let customer = 
            CustomerBuilder()
                .WithName("Alice Smith")
                .WithEmail("alice@example.com")
                .WithPhone("555-1234")
                .Build()
        
        Assert.Equal("Alice Smith", customer.Name)
        Assert.Equal("alice@example.com", customer.Email)
        Assert.True(customer.IsActive)
    
    [<Fact>]
    let ``builder creates inactive customer`` () =
        let customer = 
            CustomerBuilder()
                .AsInactive()
                .Build()
        
        Assert.False(customer.IsActive)
    
    [<Fact>]
    let ``builder creates order with multiple lines`` () =
        let lines = [
            OrderLineBuilder().WithProduct(1, "Widget").WithQuantity(2).WithPrice(10.0).Build()
            OrderLineBuilder().WithProduct(2, "Gadget").WithQuantity(1).WithPrice(25.0).Build()
        ]
        
        let order = 
            OrderBuilder()
                .WithLines(lines)
                .Build()
        
        Assert.Equal(2, order.Lines.Length)
        let total = order.Lines |> List.sumBy (fun l -> float l.Quantity * l.UnitPrice)
        Assert.Equal(45.0, total, 2)
```

---

## 9. Object Mother Pattern

```fsharp
// ObjectMother.fs
module MyProject.IntegrationTests.ObjectMother

open System
open TestDataBuilders

// Object Mother - factory methods สำหรับ test data
module TestCustomers =
    let alice () =
        CustomerBuilder()
            .WithId(1)
            .WithName("Alice Johnson")
            .WithEmail("alice@example.com")
            .WithPhone("555-0101")
            .Build()
    
    let bob () =
        CustomerBuilder()
            .WithId(2)
            .WithName("Bob Smith")
            .WithEmail("bob@example.com")
            .Build()
    
    let inactiveCustomer () =
        CustomerBuilder()
            .WithId(99)
            .WithName("Inactive User")
            .WithEmail("inactive@example.com")
            .AsInactive()
            .Build()
    
    let customerWithAddress city =
        let address = 
            AddressBuilder()
            |> fun b ->
                b.WithCity(city)
                b.WithStreet("456 Oak Ave")
                b
        CustomerBuilder()
            .WithAddress(address.Build())
            .Build()

module TestOrders =
    let pendingOrder customerId =
        OrderBuilder()
            .WithCustomer(customerId)
            .WithStatus("pending")
            .Build()
    
    let shippedOrder customerId =
        OrderBuilder()
            .WithCustomer(customerId)
            .AsShipped()
            .Build()
    
    let orderWithItems customerId items =
        OrderBuilder()
            .WithCustomer(customerId)
            .WithLines(items)
            .Build()
    
    let emptyOrder customerId =
        OrderBuilder()
            .WithCustomer(customerId)
            .WithLines([])
            .Build()

// ใช้ Object Mother ใน tests
module ObjectMotherUsage =
    open Xunit
    
    [<Fact>]
    let ``alice is active customer`` () =
        let alice = TestCustomers.alice()
        Assert.True(alice.IsActive)
        Assert.Equal("Alice Johnson", alice.Name)
    
    [<Fact>]
    let ``inactive customer has flag set`` () =
        let customer = TestCustomers.inactiveCustomer()
        Assert.False(customer.IsActive)
    
    [<Fact>]
    let ``pending order has correct status`` () =
        let order = TestOrders.pendingOrder 1
        Assert.Equal("pending", order.Status)
        Assert.True(order.ShippedAt.IsNone)
    
    [<Fact>]
    let ``shipped order has shipped date`` () =
        let order = TestOrders.shippedOrder 1
        Assert.Equal("shipped", order.Status)
        Assert.True(order.ShippedAt.IsSome)
```

---

## 10. Fixture Sharing

```fsharp
// FixtureSharing.fs
module MyProject.IntegrationTests.FixtureSharing

open Xunit
open System

// Shared expensive fixture
type ExpensiveFixture() =
    let startTime = DateTime.UtcNow
    
    // Simulate expensive initialization
    do
        // e.g., Start a server, warm up cache, etc.
        System.Threading.Thread.Sleep(10)
    
    member _.StartTime = startTime
    
    member _.ExecuteOperation(name: string) =
        sprintf "Operation %s completed at %A" name DateTime.UtcNow
    
    interface IDisposable with
        member _.Dispose() =
            // Cleanup expensive resource
            ()

// Share fixture across multiple test classes
[<CollectionDefinition("Shared Expensive Resource")>]
type SharedExpensiveCollection() =
    interface ICollectionFixture<ExpensiveFixture>

[<Collection("Shared Expensive Resource")>]
type FirstGroupTests(fixture: ExpensiveFixture) =
    [<Fact>]
    member _.``can use shared fixture`` () =
        let result = fixture.ExecuteOperation("test1")
        Assert.Contains("test1", result)
    
    [<Fact>]
    member _.``fixture is initialized once`` () =
        // Same startTime across all tests using this fixture
        Assert.True(fixture.StartTime <= DateTime.UtcNow)

[<Collection("Shared Expensive Resource")>]
type SecondGroupTests(fixture: ExpensiveFixture) =
    [<Fact>]
    member _.``shares same fixture instance`` () =
        let result = fixture.ExecuteOperation("test2")
        Assert.Contains("test2", result)

// Nested fixtures
type DatabaseSchemaFixture() =
    do printfn "Creating schema..."
    
    interface System.IAsyncLifetime with
        member _.InitializeAsync() = task {
            // Create schema once
            printfn "Schema ready"
        }
        member _.DisposeAsync() = task {
            // Drop schema
            printfn "Schema dropped"
        }

[<CollectionDefinition("Database Schema")>]
type DatabaseSchemaCollection() =
    interface ICollectionFixture<DatabaseSchemaFixture>

[<Collection("Database Schema")>]
type SchemaTests(schemaFixture: DatabaseSchemaFixture) =
    [<Fact>]
    member _.``can use schema`` () =
        Assert.NotNull(schemaFixture)
```

---

## สรุป (Summary)

Integration Testing ครอบคลุมหลาย layers:

1. **WebApplicationFactory** - ทดสอบ Web API endpoints แบบครบ
2. **TestServer** - lightweight test server
3. **In-memory databases** - เร็ว ไม่ต้องการ external service
4. **Testcontainers** - real databases/services ใน Docker
5. **Test Data Builders** - สร้าง test data แบบ flexible
6. **Object Mother** - factory methods สำหรับ common test data
7. **Fixture sharing** - ลด overhead ของ expensive setup

```bash
# รัน integration tests
dotnet test --filter "Category=Integration"

# รันพร้อม Docker (Testcontainers)
dotnet test  # Docker must be running

# รัน specific test class
dotnet test --filter "FullyQualifiedName~PostgreSqlTests"
```
