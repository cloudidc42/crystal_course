# Part 62 - Entity Framework Core กับ F#

## บทนำ (Introduction)

Entity Framework Core (EF Core) เป็น ORM (Object-Relational Mapper) ที่ทรงพลังที่สุดในระบบนิเวศ .NET มันช่วยให้เราทำงานกับฐานข้อมูลโดยใช้ .NET objects และ LINQ queries แทนการเขียน SQL โดยตรง

แม้ EF Core ถูกออกแบบมาเพื่อ C# เป็นหลัก แต่มันก็ทำงานกับ F# ได้ดี โดยต้องปรับแต่งบางอย่าง

---

## 1. การติดตั้ง (Setup)

```xml
<!-- fsharp-efcore.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.EntityFrameworkCore" Version="8.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="8.0.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="8.0.0">
      <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Entities.fs" />
    <Compile Include="AppDbContext.fs" />
    <Compile Include="Migrations.fs" />
    <Compile Include="Queries.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## 2. Entity Types (นิยาม Entities)

```fsharp
// Entities.fs
module Entities

open System
open System.Collections.Generic

// ========================================
// 2.1 Simple Entity (F# class เพราะ EF ต้องการ mutable)
// ========================================

// NOTE: EF Core ต้องการ mutable properties และ parameterless constructor
// ต้องใช้ class แทน record สำหรับ entities

[<AllowNullLiteral>]
type Category() =
    member val Id = 0 with get, set
    member val Name = "" with get, set
    member val Description = "" with get, set
    member val Products = new List<Product>() with get, set

and [<AllowNullLiteral>] Product() =
    member val Id = 0 with get, set
    member val Name = "" with get, set
    member val Price = 0m with get, set
    member val Stock = 0 with get, set
    member val Description: string = null with get, set  // nullable
    member val CategoryId = 0 with get, set
    member val Category: Category = null with get, set
    member val CreatedAt = DateTime.UtcNow with get, set
    member val UpdatedAt: Nullable<DateTime> = Nullable() with get, set
    member val IsActive = true with get, set
    member val OrderItems = new List<OrderItem>() with get, set

and [<AllowNullLiteral>] User() =
    member val Id = 0 with get, set
    member val Name = "" with get, set
    member val Email = "" with get, set
    member val PasswordHash = "" with get, set
    member val CreatedAt = DateTime.UtcNow with get, set
    member val IsActive = true with get, set
    member val Orders = new List<Order>() with get, set
    member val Profile: UserProfile = null with get, set  // one-to-one

and [<AllowNullLiteral>] UserProfile() =
    member val Id = 0 with get, set
    member val UserId = 0 with get, set
    member val Bio = "" with get, set
    member val AvatarUrl: string = null with get, set
    member val PhoneNumber: string = null with get, set
    member val User: User = null with get, set  // navigation property

and [<AllowNullLiteral>] Order() =
    member val Id = 0 with get, set
    member val UserId = 0 with get, set
    member val TotalAmount = 0m with get, set
    member val Status = "Pending" with get, set
    member val CreatedAt = DateTime.UtcNow with get, set
    member val User: User = null with get, set
    member val OrderItems = new List<OrderItem>() with get, set

and [<AllowNullLiteral>] OrderItem() =
    member val Id = 0 with get, set
    member val OrderId = 0 with get, set
    member val ProductId = 0 with get, set
    member val Quantity = 0 with get, set
    member val UnitPrice = 0m with get, set
    member val Order: Order = null with get, set
    member val Product: Product = null with get, set

// Many-to-Many example: Tags and Products
and [<AllowNullLiteral>] Tag() =
    member val Id = 0 with get, set
    member val Name = "" with get, set
    member val Products = new List<Product>() with get, set

// ========================================
// 2.2 Value Objects (Owned Entities)
// ========================================

// Address เป็น value object ที่ถูก owned โดย User
[<AllowNullLiteral>]
type Address() =
    member val Street = "" with get, set
    member val City = "" with get, set
    member val State = "" with get, set
    member val Country = "" with get, set
    member val PostalCode = "" with get, set

// ========================================
// 2.3 F# Records สำหรับ Query Results (ไม่ใช่ Entity)
// ========================================

// DTOs ใช้ F# records ได้ปกติ (ไม่ใช่ entities)
type UserDto = {
    Id: int
    Name: string
    Email: string
    OrderCount: int
}

type ProductSummary = {
    Id: int
    Name: string
    Price: decimal
    CategoryName: string
    InStock: bool
}

type OrderSummary = {
    OrderId: int
    CustomerName: string
    ItemCount: int
    TotalAmount: decimal
    Status: string
}
```

---

## 3. DbContext (การกำหนด Context)

```fsharp
// AppDbContext.fs
module AppDbContext

open System
open Microsoft.EntityFrameworkCore
open Microsoft.EntityFrameworkCore.Design
open Entities

// ========================================
// 3.1 DbContext definition
// ========================================

type AppDbContext(options: DbContextOptions<AppDbContext>) =
    inherit DbContext(options)

    // DbSets
    [<DefaultValue>]
    val mutable private _users: DbSet<User>
    member this.Users with get() = this._users and set(v) = this._users <- v

    [<DefaultValue>]
    val mutable private _categories: DbSet<Category>
    member this.Categories with get() = this._categories and set(v) = this._categories <- v

    [<DefaultValue>]
    val mutable private _products: DbSet<Product>
    member this.Products with get() = this._products and set(v) = this._products <- v

    [<DefaultValue>]
    val mutable private _orders: DbSet<Order>
    member this.Orders with get() = this._orders and set(v) = this._orders <- v

    [<DefaultValue>]
    val mutable private _orderItems: DbSet<OrderItem>
    member this.OrderItems with get() = this._orderItems and set(v) = this._orderItems <- v

    [<DefaultValue>]
    val mutable private _tags: DbSet<Tag>
    member this.Tags with get() = this._tags and set(v) = this._tags <- v

    // OnModelCreating - Fluent API configuration
    override _.OnModelCreating(modelBuilder: ModelBuilder) =
        // ========================================
        // User configuration
        // ========================================
        modelBuilder.Entity<User>(fun entity ->
            entity.ToTable("Users") |> ignore
            entity.HasKey(fun u -> u.Id :> obj) |> ignore
            entity.Property(fun u -> u.Name).IsRequired().HasMaxLength(200) |> ignore
            entity.Property(fun u -> u.Email).IsRequired().HasMaxLength(300) |> ignore
            entity.HasIndex(fun u -> u.Email :> obj).IsUnique() |> ignore
            entity.Property(fun u -> u.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP") |> ignore

            // One-to-One: User -> UserProfile
            entity.HasOne(fun u -> u.Profile)
                .WithOne(fun p -> p.User)
                .HasForeignKey<UserProfile>(fun p -> p.UserId :> obj)
                |> ignore

            // Owned entity: Address
            entity.OwnsOne<Address>(
                "ShippingAddress",
                (fun addr ->
                    addr.Property(fun a -> a.Street).HasColumnName("ShippingStreet").HasMaxLength(500) |> ignore
                    addr.Property(fun a -> a.City).HasColumnName("ShippingCity").HasMaxLength(200) |> ignore
                    addr.Property(fun a -> a.Country).HasColumnName("ShippingCountry").HasMaxLength(100) |> ignore
                    addr.Property(fun a -> a.PostalCode).HasColumnName("ShippingPostalCode").HasMaxLength(20) |> ignore
                )
            ) |> ignore
        ) |> ignore

        // ========================================
        // Category configuration
        // ========================================
        modelBuilder.Entity<Category>(fun entity ->
            entity.ToTable("Categories") |> ignore
            entity.HasKey(fun c -> c.Id :> obj) |> ignore
            entity.Property(fun c -> c.Name).IsRequired().HasMaxLength(200) |> ignore
        ) |> ignore

        // ========================================
        // Product configuration
        // ========================================
        modelBuilder.Entity<Product>(fun entity ->
            entity.ToTable("Products") |> ignore
            entity.HasKey(fun p -> p.Id :> obj) |> ignore
            entity.Property(fun p -> p.Name).IsRequired().HasMaxLength(300) |> ignore
            entity.Property(fun p -> p.Price).HasPrecision(18, 2) |> ignore
            entity.Property(fun p -> p.Description).HasMaxLength(2000) |> ignore
            entity.Property(fun p -> p.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP") |> ignore

            // One-to-Many: Category -> Products
            entity.HasOne(fun p -> p.Category)
                .WithMany(fun c -> c.Products :> _)
                .HasForeignKey(fun p -> p.CategoryId :> obj)
                .OnDelete(DeleteBehavior.Restrict)
                |> ignore

            // Shadow property สำหรับ soft delete
            entity.Property<DateTime?>("DeletedAt") |> ignore
            entity.HasQueryFilter(fun p ->
                Microsoft.EntityFrameworkCore.EF.Property<DateTime?>(p, "DeletedAt") = null
            ) |> ignore
        ) |> ignore

        // ========================================
        // Order configuration
        // ========================================
        modelBuilder.Entity<Order>(fun entity ->
            entity.ToTable("Orders") |> ignore
            entity.HasKey(fun o -> o.Id :> obj) |> ignore
            entity.Property(fun o -> o.TotalAmount).HasPrecision(18, 2) |> ignore
            entity.Property(fun o -> o.Status)
                .IsRequired()
                .HasDefaultValue("Pending")
                .HasConversion(
                    (fun s -> s),
                    (fun s -> s)
                ) |> ignore

            entity.HasOne(fun o -> o.User)
                .WithMany(fun u -> u.Orders :> _)
                .HasForeignKey(fun o -> o.UserId :> obj)
                .OnDelete(DeleteBehavior.Cascade)
                |> ignore
        ) |> ignore

        // ========================================
        // OrderItem configuration
        // ========================================
        modelBuilder.Entity<OrderItem>(fun entity ->
            entity.ToTable("OrderItems") |> ignore
            entity.HasKey(fun oi -> oi.Id :> obj) |> ignore
            entity.Property(fun oi -> oi.UnitPrice).HasPrecision(18, 2) |> ignore

            entity.HasOne(fun oi -> oi.Order)
                .WithMany(fun o -> o.OrderItems :> _)
                .HasForeignKey(fun oi -> oi.OrderId :> obj)
                .OnDelete(DeleteBehavior.Cascade)
                |> ignore

            entity.HasOne(fun oi -> oi.Product)
                .WithMany(fun p -> p.OrderItems :> _)
                .HasForeignKey(fun oi -> oi.ProductId :> obj)
                .OnDelete(DeleteBehavior.Restrict)
                |> ignore
        ) |> ignore

        // ========================================
        // Many-to-Many: Products <-> Tags
        // ========================================
        modelBuilder.Entity<Product>(fun entity ->
            entity.HasMany(fun p -> p.OrderItems :> _)
                .WithOne(fun oi -> oi.Product)
                |> ignore
        ) |> ignore

// ========================================
// 3.2 DbContext Factory สำหรับ Migrations
// ========================================

type AppDbContextFactory() =
    interface IDesignTimeDbContextFactory<AppDbContext> with
        member _.CreateDbContext(_args: string[]) =
            let optionsBuilder = DbContextOptionsBuilder<AppDbContext>()
            optionsBuilder.UseSqlite("Data Source=app.db") |> ignore
            new AppDbContext(optionsBuilder.Options)

// ========================================
// 3.3 Helper สำหรับสร้าง Context
// ========================================

let createSqliteContext (connectionString: string) =
    let optionsBuilder = DbContextOptionsBuilder<AppDbContext>()
    optionsBuilder.UseSqlite(connectionString)
        .EnableSensitiveDataLogging()
        .EnableDetailedErrors()
        |> ignore
    new AppDbContext(optionsBuilder.Options)

let createInMemoryContext (dbName: string) =
    let optionsBuilder = DbContextOptionsBuilder<AppDbContext>()
    optionsBuilder.UseInMemoryDatabase(dbName) |> ignore
    new AppDbContext(optionsBuilder.Options)
```

---

## 4. Migrations (การย้ายข้อมูล)

```fsharp
// Migrations.fs
module MigrationExamples

open Microsoft.EntityFrameworkCore
open AppDbContext

// ========================================
// 4.1 สร้าง Database และ apply migrations
// ========================================

let ensureCreated (context: AppDbContext) =
    // ใช้สำหรับ development/testing เท่านั้น
    context.Database.EnsureCreated() |> ignore
    printfn "Database created"

let migrateDatabase (context: AppDbContext) =
    // Apply pending migrations
    context.Database.Migrate()
    printfn "Migrations applied"

let ensureDeleted (context: AppDbContext) =
    // ใช้สำหรับ testing เท่านั้น!
    context.Database.EnsureDeleted() |> ignore
    printfn "Database deleted"

// ========================================
// 4.2 Migration commands (CLI)
// ========================================

(*
dotnet ef migrations add InitialCreate
dotnet ef migrations add AddUserProfile
dotnet ef database update
dotnet ef database update InitialCreate  -- rollback to specific migration
dotnet ef migrations list
dotnet ef migrations remove  -- remove last migration
dotnet ef database drop
*)

// ========================================
// 4.3 Seed data
// ========================================

let seedDatabase (context: AppDbContext) =
    task {
        // ตรวจสอบว่ามี data อยู่แล้วหรือไม่
        let! hasCategories = context.Categories.AnyAsync()
        if not hasCategories then
            let electronics = Entities.Category()
            electronics.Name <- "Electronics"
            electronics.Description <- "Electronic devices and accessories"

            let accessories = Entities.Category()
            accessories.Name <- "Accessories"
            accessories.Description <- "Computer accessories"

            context.Categories.AddRange(electronics, accessories) |> ignore
            let! _ = context.SaveChangesAsync()

            let laptop = Entities.Product()
            laptop.Name <- "Laptop Pro 15"
            laptop.Price <- 49999m
            laptop.Stock <- 50
            laptop.CategoryId <- electronics.Id

            let mouse = Entities.Product()
            mouse.Name <- "Wireless Mouse"
            mouse.Price <- 899m
            mouse.Stock <- 200
            mouse.CategoryId <- accessories.Id

            context.Products.AddRange(laptop, mouse) |> ignore
            let! _ = context.SaveChangesAsync()

            printfn "Seed data inserted: %s (ID=%d), %s (ID=%d)"
                electronics.Name electronics.Id
                accessories.Name accessories.Id
        else
            printfn "Database already has data, skipping seed"
    }
```

---

## 5. LINQ Queries (การ Query ด้วย LINQ)

```fsharp
// Queries.fs
module QueryExamples

open System.Linq
open Microsoft.EntityFrameworkCore
open AppDbContext
open Entities

// ========================================
// 5.1 Basic queries
// ========================================

/// ดึง Products ทั้งหมด
let getAllProducts (ctx: AppDbContext) =
    task {
        return! ctx.Products.ToListAsync()
    }

/// ดึง Product ด้วย ID
let getProductById (ctx: AppDbContext) (id: int) =
    task {
        let! product = ctx.Products.FindAsync(id)
        return Option.ofObj product
    }

/// ดึง Products ที่ active และมี stock
let getActiveProductsInStock (ctx: AppDbContext) =
    task {
        return!
            ctx.Products
                .Where(fun p -> p.IsActive && p.Stock > 0)
                .OrderBy(fun p -> p.Name)
                .ToListAsync()
    }

/// ดึง Products ตาม Category
let getProductsByCategory (ctx: AppDbContext) (categoryId: int) =
    task {
        return!
            ctx.Products
                .Where(fun p -> p.CategoryId = categoryId)
                .Include(fun p -> p.Category)
                .ToListAsync()
    }

// ========================================
// 5.2 Filtering and sorting
// ========================================

/// ค้นหา Products ตามชื่อ
let searchProducts (ctx: AppDbContext) (searchTerm: string) =
    task {
        return!
            ctx.Products
                .Where(fun p -> p.Name.Contains(searchTerm))
                .OrderByDescending(fun p -> p.Price)
                .ToListAsync()
    }

/// ดึง Products ตาม price range
let getProductsByPriceRange (ctx: AppDbContext) (minPrice: decimal) (maxPrice: decimal) =
    task {
        return!
            ctx.Products
                .Where(fun p -> p.Price >= minPrice && p.Price <= maxPrice)
                .OrderBy(fun p -> p.Price)
                .ToListAsync()
    }

// ========================================
// 5.3 Projections (Select)
// ========================================

/// ดึงเฉพาะ Product summary
let getProductSummaries (ctx: AppDbContext) =
    task {
        return!
            ctx.Products
                .Include(fun p -> p.Category)
                .Select(fun p ->
                    {
                        Id = p.Id
                        Name = p.Name
                        Price = p.Price
                        CategoryName = if p.Category <> null then p.Category.Name else "Unknown"
                        InStock = p.Stock > 0
                    } : ProductSummary)
                .ToListAsync()
    }

/// ดึง User DTOs
let getUserDtos (ctx: AppDbContext) =
    task {
        return!
            ctx.Users
                .Select(fun u ->
                    {
                        Id = u.Id
                        Name = u.Name
                        Email = u.Email
                        OrderCount = u.Orders.Count
                    } : UserDto)
                .ToListAsync()
    }

// ========================================
// 5.4 Aggregation
// ========================================

/// นับจำนวน Products ต่อ Category
let getProductCountByCategory (ctx: AppDbContext) =
    task {
        return!
            ctx.Products
                .GroupBy(fun p -> p.CategoryId)
                .Select(fun g ->
                    {|
                        CategoryId = g.Key
                        Count = g.Count()
                        AvgPrice = g.Average(fun p -> p.Price)
                        MaxPrice = g.Max(fun p -> p.Price)
                    |})
                .ToListAsync()
    }

/// หา Total Revenue
let getTotalRevenue (ctx: AppDbContext) =
    task {
        return! ctx.Orders
            .Where(fun o -> o.Status = "Delivered")
            .SumAsync(fun o -> o.TotalAmount)
    }

// ========================================
// 5.5 Joins
// ========================================

/// ดึง Order summary พร้อมข้อมูล Customer
let getOrderSummaries (ctx: AppDbContext) =
    task {
        return!
            ctx.Orders
                .Include(fun o -> o.User)
                .Include(fun o -> o.OrderItems)
                .Select(fun o ->
                    {
                        OrderId = o.Id
                        CustomerName = if o.User <> null then o.User.Name else "Unknown"
                        ItemCount = o.OrderItems.Count
                        TotalAmount = o.TotalAmount
                        Status = o.Status
                    } : OrderSummary)
                .OrderByDescending(fun o -> o.OrderId)
                .ToListAsync()
    }

// ========================================
// 5.6 Include / ThenInclude (Eager loading)
// ========================================

/// ดึง Order พร้อมข้อมูลทั้งหมด
let getFullOrder (ctx: AppDbContext) (orderId: int) =
    task {
        return!
            ctx.Orders
                .Include(fun o -> o.User)
                    .ThenInclude(fun u -> u.Profile)
                .Include(fun o -> o.OrderItems)
                    .ThenInclude(fun oi -> oi.Product)
                        .ThenInclude(fun p -> p.Category)
                .FirstOrDefaultAsync(fun o -> o.Id = orderId)
            |> System.Threading.Tasks.Task.FromResult
    }

// ========================================
// 5.7 Paged queries
// ========================================

let getPagedProducts (ctx: AppDbContext) (page: int) (pageSize: int) =
    task {
        let query = ctx.Products.Where(fun p -> p.IsActive)
        let! totalCount = query.CountAsync()
        let! items =
            query
                .OrderBy(fun p -> p.Name)
                .Skip((page - 1) * pageSize)
                .Take(pageSize)
                .ToListAsync()

        return {|
            Items = items
            TotalCount = totalCount
            Page = page
            PageSize = pageSize
            TotalPages = (totalCount + pageSize - 1) / pageSize
        |}
    }

// ========================================
// 5.8 Raw SQL with EF Core
// ========================================

/// ใช้ Raw SQL
let getProductsWithRawSql (ctx: AppDbContext) (minStock: int) =
    task {
        return!
            ctx.Products
                .FromSqlRaw("SELECT * FROM Products WHERE Stock >= {0}", minStock)
                .ToListAsync()
    }

/// Execute Raw SQL command
let deleteInactiveProducts (ctx: AppDbContext) =
    task {
        return! ctx.Database.ExecuteSqlRawAsync(
            "DELETE FROM Products WHERE IsActive = 0 AND Stock = 0"
        )
    }

/// Raw SQL ด้วย parameters
let getProductsByCategory2 (ctx: AppDbContext) (categoryName: string) =
    task {
        return!
            ctx.Products
                .FromSqlInterpolated($"SELECT p.* FROM Products p INNER JOIN Categories c ON p.CategoryId = c.Id WHERE c.Name = {categoryName}")
                .ToListAsync()
    }
```

---

## 6. CRUD Operations (การเพิ่ม แก้ไข ลบ)

```fsharp
// CrudOperations.fs
module CrudOperations

open Microsoft.EntityFrameworkCore
open AppDbContext
open Entities

// ========================================
// 6.1 Create
// ========================================

let createUser (ctx: AppDbContext) (name: string) (email: string) (passwordHash: string) =
    task {
        let user = User()
        user.Name <- name
        user.Email <- email
        user.PasswordHash <- passwordHash
        user.CreatedAt <- System.DateTime.UtcNow

        ctx.Users.Add(user) |> ignore
        let! _ = ctx.SaveChangesAsync()
        return user.Id
    }

let createProductWithCategory (ctx: AppDbContext) (productName: string) (price: decimal) (categoryName: string) =
    task {
        // หรือสร้าง Category ใหม่ถ้าไม่มี
        let! category =
            ctx.Categories.FirstOrDefaultAsync(fun c -> c.Name = categoryName)

        let cat =
            if category = null then
                let newCat = Category()
                newCat.Name <- categoryName
                ctx.Categories.Add(newCat) |> ignore
                newCat
            else
                category

        let product = Product()
        product.Name <- productName
        product.Price <- price
        product.Category <- cat
        product.Stock <- 0

        ctx.Products.Add(product) |> ignore
        let! _ = ctx.SaveChangesAsync()
        return product.Id
    }

// ========================================
// 6.2 Update
// ========================================

let updateProduct (ctx: AppDbContext) (id: int) (name: string) (price: decimal) (stock: int) =
    task {
        let! product = ctx.Products.FindAsync(id)
        if product = null then
            return false
        else
            product.Name <- name
            product.Price <- price
            product.Stock <- stock
            product.UpdatedAt <- System.Nullable(System.DateTime.UtcNow)
            let! _ = ctx.SaveChangesAsync()
            return true
    }

/// Update ด้วย Attach (ไม่ต้อง load ก่อน)
let updateProductPrice (ctx: AppDbContext) (id: int) (newPrice: decimal) =
    task {
        let product = Product()
        product.Id <- id
        product.Price <- newPrice
        product.UpdatedAt <- System.Nullable(System.DateTime.UtcNow)

        let entry = ctx.Attach(product)
        entry.Property(fun p -> p.Price).IsModified <- true
        entry.Property(fun p -> p.UpdatedAt).IsModified <- true

        let! _ = ctx.SaveChangesAsync()
        return ()
    }

// ========================================
// 6.3 Delete
// ========================================

let deleteProduct (ctx: AppDbContext) (id: int) =
    task {
        let! product = ctx.Products.FindAsync(id)
        if product = null then
            return false
        else
            ctx.Products.Remove(product) |> ignore
            let! _ = ctx.SaveChangesAsync()
            return true
    }

/// Soft delete ด้วย shadow property
let softDeleteProduct (ctx: AppDbContext) (id: int) =
    task {
        let! product = ctx.Products.FindAsync(id)
        if product = null then
            return false
        else
            ctx.Entry(product).Property("DeletedAt").CurrentValue <- System.DateTime.UtcNow
            let! _ = ctx.SaveChangesAsync()
            return true
    }

// ========================================
// 6.4 Concurrency handling
// ========================================

// Entity ที่มี concurrency token
[<AllowNullLiteral>]
type InventoryItem() =
    member val Id = 0 with get, set
    member val ProductId = 0 with get, set
    member val Quantity = 0 with get, set
    // Concurrency token - EF Core จะตรวจสอบค่านี้เมื่อ update
    [<System.ComponentModel.DataAnnotations.ConcurrencyCheck>]
    member val RowVersion = [||] : byte[] with get, set

let updateInventoryWithConcurrencyCheck (ctx: AppDbContext) (itemId: int) (newQuantity: int) =
    task {
        let mutable retries = 3
        let mutable success = false
        let mutable result = Error "Failed"

        while retries > 0 && not success do
            try
                let! item = ctx.Set<InventoryItem>().FindAsync(itemId)
                if item = null then
                    result <- Error "Item not found"
                    retries <- 0
                else
                    item.Quantity <- newQuantity
                    let! _ = ctx.SaveChangesAsync()
                    success <- true
                    result <- Ok item.Id
            with
            | :? DbUpdateConcurrencyException ->
                retries <- retries - 1
                ctx.ChangeTracker.Clear()  // Reset tracked entities
                if retries = 0 then
                    result <- Error "Concurrency conflict - please retry"

        return result
    }
```

---

## 7. Relationships (ความสัมพันธ์ระหว่าง Entities)

```fsharp
// Relationships.fs
module RelationshipExamples

open Microsoft.EntityFrameworkCore
open AppDbContext
open Entities

// ========================================
// 7.1 One-to-One
// ========================================

let createUserWithProfile (ctx: AppDbContext) (name: string) (email: string) (bio: string) =
    task {
        let user = User()
        user.Name <- name
        user.Email <- email
        user.PasswordHash <- "hashed_password"

        let profile = UserProfile()
        profile.Bio <- bio
        profile.User <- user

        ctx.Users.Add(user) |> ignore
        ctx.Set<UserProfile>().Add(profile) |> ignore

        let! _ = ctx.SaveChangesAsync()
        return user.Id
    }

let getUserWithProfile (ctx: AppDbContext) (userId: int) =
    task {
        return!
            ctx.Users
                .Include(fun u -> u.Profile)
                .FirstOrDefaultAsync(fun u -> u.Id = userId)
    }

// ========================================
// 7.2 One-to-Many
// ========================================

let getUserOrders (ctx: AppDbContext) (userId: int) =
    task {
        return!
            ctx.Orders
                .Where(fun o -> o.UserId = userId)
                .Include(fun o -> o.OrderItems)
                .OrderByDescending(fun o -> o.CreatedAt)
                .ToListAsync()
    }

let createOrderForUser (ctx: AppDbContext) (userId: int) (items: (int * int) list) =
    // items = (productId, quantity) list
    task {
        let! user = ctx.Users.FindAsync(userId)
        if user = null then
            return Error "User not found"
        else
            let order = Order()
            order.UserId <- userId
            order.Status <- "Pending"
            order.CreatedAt <- System.DateTime.UtcNow
            order.TotalAmount <- 0m

            let mutable total = 0m
            for (productId, quantity) in items do
                let! product = ctx.Products.FindAsync(productId)
                if product <> null then
                    let item = OrderItem()
                    item.ProductId <- productId
                    item.Quantity <- quantity
                    item.UnitPrice <- product.Price
                    order.OrderItems.Add(item)
                    total <- total + (product.Price * decimal quantity)

            order.TotalAmount <- total
            ctx.Orders.Add(order) |> ignore
            let! _ = ctx.SaveChangesAsync()
            return Ok order.Id
    }

// ========================================
// 7.3 Many-to-Many (Products <-> Tags)
// ========================================

// ตัวอย่าง Many-to-Many configuration ใน OnModelCreating:
(*
modelBuilder.Entity<Product>()
    .HasMany(p => p.Tags)
    .WithMany(t => t.Products)
    .UsingEntity(j => j.ToTable("ProductTags"));
*)

// Query ด้วย Many-to-Many
let getProductsByTag (ctx: AppDbContext) (tagName: string) =
    task {
        return!
            ctx.Products
                .Where(fun p -> p.OrderItems.Any())  // example filter
                .Include(fun p -> p.Category)
                .ToListAsync()
    }
```

---

## 8. Transactions (การจัดการ Transactions)

```fsharp
// Transactions.fs
module EFTransactions

open Microsoft.EntityFrameworkCore
open AppDbContext
open Entities

// ========================================
// 8.1 Implicit Transactions (SaveChanges)
// ========================================

/// SaveChanges ใช้ transaction โดยอัตโนมัติ
let createMultipleEntities (ctx: AppDbContext) =
    task {
        // การเปลี่ยนแปลงทั้งหมดถูก commit พร้อมกัน
        let cat = Category()
        cat.Name <- "New Category"

        let prod = Product()
        prod.Name <- "New Product"
        prod.Price <- 999m
        prod.Category <- cat

        ctx.Categories.Add(cat) |> ignore
        ctx.Products.Add(prod) |> ignore

        // ถ้า save ล้มเหลว ทั้งหมดจะถูก rollback
        return! ctx.SaveChangesAsync()
    }

// ========================================
// 8.2 Explicit Transactions
// ========================================

let transferOrderToUser (ctx: AppDbContext) (orderId: int) (newUserId: int) =
    task {
        use! transaction = ctx.Database.BeginTransactionAsync()
        try
            let! order = ctx.Orders.FindAsync(orderId)
            let! newUser = ctx.Users.FindAsync(newUserId)

            if order = null then
                do! transaction.RollbackAsync()
                return Error "Order not found"
            elif newUser = null then
                do! transaction.RollbackAsync()
                return Error "User not found"
            else
                order.UserId <- newUserId
                let! _ = ctx.SaveChangesAsync()

                // Log the transfer
                let logEntry = {|
                    OrderId = orderId
                    NewUserId = newUserId
                    TransferredAt = System.DateTime.UtcNow
                |}
                // ctx.TransferLogs.Add(logEntry) |> ignore
                // let! _ = ctx.SaveChangesAsync()

                do! transaction.CommitAsync()
                return Ok "Transfer successful"
        with ex ->
            do! transaction.RollbackAsync()
            return Error ex.Message
    }

// ========================================
// 8.3 Savepoints
// ========================================

let complexOperation (ctx: AppDbContext) =
    task {
        use! transaction = ctx.Database.BeginTransactionAsync()
        try
            // Phase 1
            let user1 = User()
            user1.Name <- "User 1"
            user1.Email <- "user1@test.com"
            ctx.Users.Add(user1) |> ignore
            let! _ = ctx.SaveChangesAsync()

            // Savepoint หลัง phase 1
            do! transaction.CreateSavepointAsync("AfterUser1")

            try
                // Phase 2 (อาจ fail)
                let user2 = User()
                user2.Name <- "User 2"
                user2.Email <- "duplicate@test.com"  // อาจ duplicate
                ctx.Users.Add(user2) |> ignore
                let! _ = ctx.SaveChangesAsync()
            with _ ->
                // Rollback เฉพาะ phase 2
                do! transaction.RollbackToSavepointAsync("AfterUser1")
                ctx.ChangeTracker.Clear()

            do! transaction.CommitAsync()
            return Ok "Done"
        with ex ->
            do! transaction.RollbackAsync()
            return Error ex.Message
    }
```

---

## 9. Value Conversions สำหรับ DUs (Value Conversions for Discriminated Unions)

```fsharp
// ValueConversions.fs
module ValueConversionExamples

open Microsoft.EntityFrameworkCore
open Microsoft.EntityFrameworkCore.Storage.ValueConversion

// ========================================
// 9.1 DU ที่จะใช้กับ EF
// ========================================

type OrderStatus =
    | Pending
    | Processing
    | Shipped
    | Delivered
    | Cancelled

type PaymentMethod =
    | CreditCard
    | DebitCard
    | BankTransfer
    | PromptPay

// ========================================
// 9.2 Custom converters
// ========================================

let orderStatusConverter =
    ValueConverter<OrderStatus, string>(
        (fun status ->
            match status with
            | Pending -> "Pending"
            | Processing -> "Processing"
            | Shipped -> "Shipped"
            | Delivered -> "Delivered"
            | Cancelled -> "Cancelled"),
        (fun str ->
            match str with
            | "Pending" -> Pending
            | "Processing" -> Processing
            | "Shipped" -> Shipped
            | "Delivered" -> Delivered
            | "Cancelled" -> Cancelled
            | other -> failwith $"Unknown status: {other}")
    )

let paymentMethodConverter =
    ValueConverter<PaymentMethod, int>(
        (fun method ->
            match method with
            | CreditCard -> 1
            | DebitCard -> 2
            | BankTransfer -> 3
            | PromptPay -> 4),
        (fun i ->
            match i with
            | 1 -> CreditCard
            | 2 -> DebitCard
            | 3 -> BankTransfer
            | 4 -> PromptPay
            | other -> failwith $"Unknown payment method: {other}")
    )

// ========================================
// 9.3 ลงทะเบียน converters ใน OnModelCreating
// ========================================

(*
// ใน AppDbContext.OnModelCreating:
modelBuilder.Entity<Order>(fun entity ->
    entity.Property(fun o -> o.Status)
        .HasConversion(orderStatusConverter)
        |> ignore
)

// หรือ global conversion
modelBuilder.UseValueConverterForType<OrderStatus>(orderStatusConverter)
*)

// ========================================
// 9.4 Option<T> converter
// ========================================

let optionStringConverter =
    ValueConverter<string option, string>(
        (fun opt -> opt |> Option.defaultValue null),
        (fun str -> if str = null then None else Some str)
    )

let optionIntConverter =
    ValueConverter<int option, int>(
        (fun opt -> opt |> Option.defaultValue 0),
        (fun i -> if i = 0 then None else Some i)
    )
```

---

## 10. Shadow Properties (คุณสมบัติที่ซ่อน)

```fsharp
// ShadowProperties.fs
module ShadowPropertyExamples

open System
open Microsoft.EntityFrameworkCore
open AppDbContext

// ========================================
// 10.1 Shadow properties สำหรับ audit
// ========================================

// ใน OnModelCreating เพิ่ม shadow properties สำหรับทุก entity
let addAuditProperties (modelBuilder: ModelBuilder) =
    for entityType in modelBuilder.Model.GetEntityTypes() do
        modelBuilder.Entity(entityType.Name, fun entity ->
            entity.Property<DateTime>("CreatedAt")
                .HasDefaultValueSql("CURRENT_TIMESTAMP")
                |> ignore
            entity.Property<DateTime?>("UpdatedAt") |> ignore
            entity.Property<string>("CreatedBy").HasMaxLength(200) |> ignore
            entity.Property<string>("UpdatedBy").HasMaxLength(200) |> ignore
        ) |> ignore

// ========================================
// 10.2 การ Set shadow properties
// ========================================

let setAuditProperties (ctx: AppDbContext) (currentUser: string) =
    let now = DateTime.UtcNow
    for entry in ctx.ChangeTracker.Entries() do
        match entry.State with
        | EntityState.Added ->
            entry.Property("CreatedAt").CurrentValue <- box now
            entry.Property("CreatedBy").CurrentValue <- box currentUser
        | EntityState.Modified ->
            entry.Property("UpdatedAt").CurrentValue <- box now
            entry.Property("UpdatedBy").CurrentValue <- box currentUser
        | _ -> ()

// ========================================
// 10.3 Override SaveChanges เพื่อ set audit
// ========================================

// ใน AppDbContext:
(*
override this.SaveChangesAsync(?cancellationToken) =
    setAuditProperties this "system"
    base.SaveChangesAsync(?cancellationToken)
*)

// ========================================
// 10.4 Query โดยใช้ shadow properties
// ========================================

let getRecentlyUpdated (ctx: AppDbContext) =
    task {
        let yesterday = DateTime.UtcNow.AddDays(-1.0)
        return!
            ctx.Products
                .Where(fun p ->
                    EF.Property<DateTime?>(p, "UpdatedAt") > Nullable yesterday
                )
                .ToListAsync()
    }
```

---

## 11. Complete Example (ตัวอย่างครบวงจร)

```fsharp
// Program.fs
module Program

open System
open System.Linq
open Microsoft.EntityFrameworkCore
open AppDbContext
open Entities

[<EntryPoint>]
let main _ =
    task {
        printfn "=== EF Core F# Demo ==="
        printfn "========================"

        // สร้าง in-memory database สำหรับ demo
        use ctx = createInMemoryContext "DemoDb"
        do! MigrationExamples.seedDatabase ctx

        // ========================================
        // CREATE
        // ========================================
        printfn "\n--- Creating entities ---"

        let! cat1Id = CrudOperations.createProductWithCategory ctx "Laptop X1" 35999m "Computers"
        let! cat2Id = CrudOperations.createProductWithCategory ctx "Gaming Mouse" 1299m "Peripherals"
        printfn "Created products: %d, %d" cat1Id cat2Id

        let! userId = CrudOperations.createUser ctx "สมชาย ใจดี" "somchai@test.com" "hash123"
        printfn "Created user ID: %d" userId

        // ========================================
        // READ
        // ========================================
        printfn "\n--- Reading data ---"

        let! products = QueryExamples.getAllProducts ctx
        printfn "Total products: %d" (products |> Seq.length)
        for p in products do
            printfn "  [%d] %s - ฿%.2f (Stock: %d)" p.Id p.Name p.Price p.Stock

        let! summaries = QueryExamples.getProductSummaries ctx
        printfn "\nProduct Summaries:"
        for s in summaries do
            printfn "  %s (Category: %s) - ฿%.2f - In Stock: %b" s.Name s.CategoryName s.Price s.InStock

        // ========================================
        // AGGREGATION
        // ========================================
        printfn "\n--- Aggregation ---"

        let! categoryCounts = QueryExamples.getProductCountByCategory ctx
        for cc in categoryCounts do
            printfn "  CategoryId %d: %d products, avg price ฿%.2f" cc.CategoryId cc.Count cc.AvgPrice

        // ========================================
        // UPDATE
        // ========================================
        printfn "\n--- Updating ---"

        let! updated = CrudOperations.updateProduct ctx 1 "Laptop X1 Pro" 38999m 45
        printfn "Product updated: %b" updated

        let! product = QueryExamples.getProductById ctx 1
        match product with
        | Some p -> printfn "Updated product: %s - ฿%.2f" p.Name p.Price
        | None -> printfn "Product not found"

        // ========================================
        // QUERY with filtering
        // ========================================
        printfn "\n--- Filtered queries ---"

        let! expensive = QueryExamples.getProductsByPriceRange ctx 10000m 100000m
        printfn "Expensive products (>10000):"
        for p in expensive do
            printfn "  %s - ฿%.2f" p.Name p.Price

        // ========================================
        // Transaction
        // ========================================
        printfn "\n--- Transaction ---"

        let items = [(1, 1); (2, 2)]  // (productId, quantity)
        let! orderResult = RelationshipExamples.createOrderForUser ctx userId items
        match orderResult with
        | Ok orderId -> printfn "Created order ID: %d" orderId
        | Error msg -> printfn "Order failed: %s" msg

        // ========================================
        // PAGING
        // ========================================
        printfn "\n--- Paged results ---"

        let! paged = QueryExamples.getPagedProducts ctx 1 3
        printfn "Page 1 of %d (total: %d products)" paged.TotalPages paged.TotalCount
        for p in paged.Items do
            printfn "  %s" p.Name

        // ========================================
        // DELETE
        // ========================================
        printfn "\n--- Deleting ---"

        let! deleted = CrudOperations.deleteProduct ctx 2
        printfn "Product deleted: %b" deleted

        let! remaining = QueryExamples.getAllProducts ctx
        printfn "Remaining products: %d" (remaining |> Seq.length)

        printfn "\n=== Demo Complete ==="
        return 0
    } |> Async.AwaitTask |> Async.RunSynchronously
```

---

## สรุป (Summary)

EF Core กับ F# มีข้อควรรู้ดังนี้:

1. **Entities ต้องเป็น class**: EF Core ต้องการ mutable properties และ parameterless constructor
2. **DTOs ใช้ records ได้**: สำหรับ query results ที่ไม่ต้อง track
3. **Value Conversions**: ใช้แปลง DUs เป็น string/int สำหรับ storage
4. **Shadow Properties**: ใช้สำหรับ audit fields ที่ไม่ต้องการใน entity class
5. **Navigation Properties**: ต้องระวัง null ใน F#

```fsharp
// Key EF Core operations ใน F#
// Query: ctx.Products.Where(...).Include(...).ToListAsync()
// Create: ctx.Entity.Add(obj); ctx.SaveChangesAsync()
// Update: load -> modify -> SaveChangesAsync()
// Delete: ctx.Entity.Remove(obj); ctx.SaveChangesAsync()
// Raw SQL: ctx.Entity.FromSqlRaw("SELECT ...")
// Transaction: use! tx = ctx.Database.BeginTransactionAsync()
```
