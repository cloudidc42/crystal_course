# Part 61 - Dapper กับ F# (Dapper with F#)

## บทนำ (Introduction)

Dapper เป็น micro-ORM ที่พัฒนาโดยทีม Stack Overflow มันเป็น extension methods บน `IDbConnection` ที่ช่วยให้การเขียน SQL queries ง่ายขึ้นโดยไม่ต้องเขียน boilerplate code มากมาย

Dapper มีข้อดีดังนี้:
- **เร็ว**: ใกล้เคียงกับ raw ADO.NET
- **ง่าย**: เรียนรู้ได้เร็ว
- **ยืดหยุ่น**: ควบคุม SQL ได้เต็มที่
- **F# friendly**: ทำงานกับ F# records ได้ดี

---

## 1. การติดตั้ง (Setup)

```xml
<!-- fsharp-dapper.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Dapper" Version="2.1.28" />
    <PackageReference Include="Microsoft.Data.Sqlite" Version="8.0.0" />
    <PackageReference Include="Npgsql" Version="8.0.0" />
    <PackageReference Include="Microsoft.Data.SqlClient" Version="5.1.5" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="TypeHandlers.fs" />
    <Compile Include="Models.fs" />
    <Compile Include="Repository.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## 2. Models (โมเดลข้อมูล)

```fsharp
// Models.fs
module Models

open System

// F# record สำหรับ User
type User = {
    Id: int
    Name: string
    Email: string
    CreatedAt: DateTime
    IsActive: bool
}

// F# record สำหรับ Product
type Product = {
    Id: int
    Name: string
    Price: decimal
    CategoryId: int
    Stock: int
    Description: string option  // nullable field
}

// F# record สำหรับ Order
type Order = {
    Id: int
    UserId: int
    TotalAmount: decimal
    Status: string
    CreatedAt: DateTime
}

// F# record สำหรับ OrderItem
type OrderItem = {
    Id: int
    OrderId: int
    ProductId: int
    Quantity: int
    UnitPrice: decimal
}

// Discriminated Union สำหรับ Status
type OrderStatus =
    | Pending
    | Processing
    | Shipped
    | Delivered
    | Cancelled

// Record พร้อม DU
type OrderWithStatus = {
    Id: int
    UserId: int
    TotalAmount: decimal
    Status: OrderStatus
    CreatedAt: DateTime
}

// DTO สำหรับสร้าง User ใหม่
type CreateUserDto = {
    Name: string
    Email: string
}

// DTO สำหรับอัปเดต User
type UpdateUserDto = {
    Id: int
    Name: string
    Email: string
    IsActive: bool
}

// Result type สำหรับ paginated query
type PagedResult<'T> = {
    Items: 'T list
    TotalCount: int
    Page: int
    PageSize: int
    TotalPages: int
}
```

---

## 3. การสร้าง Connection (Creating Connections)

```fsharp
// ConnectionFactory.fs
module ConnectionFactory

open System.Data
open Microsoft.Data.Sqlite
open Npgsql

// SQLite connection
let createSqliteConnection connectionString =
    let conn = new SqliteConnection(connectionString)
    conn :> IDbConnection

// PostgreSQL connection
let createPostgresConnection connectionString =
    let conn = new NpgsqlConnection(connectionString)
    conn :> IDbConnection

// In-memory SQLite สำหรับ testing
let createInMemoryConnection () =
    let conn = new SqliteConnection("Data Source=:memory:")
    conn.Open()
    conn :> IDbConnection

// Connection string builder สำหรับ SQLite
let buildSqliteConnectionString databasePath =
    $"Data Source={databasePath};Mode=ReadWriteCreate;"

// Connection string builder สำหรับ PostgreSQL
let buildPostgresConnectionString host port database username password =
    $"Host={host};Port={port};Database={database};Username={username};Password={password}"
```

---

## 4. Type Handlers (ตัวจัดการประเภทข้อมูล)

```fsharp
// TypeHandlers.fs
module TypeHandlers

open System
open System.Data
open Dapper

// Type handler สำหรับ F# option types
type OptionHandler<'T>() =
    inherit SqlMapper.TypeHandler<'T option>()

    override _.SetValue(parameter, value) =
        match value with
        | Some v ->
            parameter.Value <- v
        | None ->
            parameter.Value <- DBNull.Value

    override _.Parse(value) =
        if value = null || value = box DBNull.Value then
            None
        else
            Some (value :?> 'T)

// Type handler สำหรับ DateTimeOffset
type DateTimeOffsetHandler() =
    inherit SqlMapper.TypeHandler<DateTimeOffset>()

    override _.SetValue(parameter, value) =
        parameter.Value <- value.ToString("o")  // ISO 8601

    override _.Parse(value) =
        DateTimeOffset.Parse(value.ToString())

// Type handler สำหรับ Uri
type UriHandler() =
    inherit SqlMapper.TypeHandler<Uri>()

    override _.SetValue(parameter, value) =
        parameter.Value <- value.ToString()

    override _.Parse(value) =
        Uri(value.ToString())

// Type handler สำหรับ OrderStatus DU
type OrderStatusHandler() =
    inherit SqlMapper.TypeHandler<OrderStatus>()

    override _.SetValue(parameter, value) =
        parameter.Value <-
            match value with
            | Models.Pending -> "Pending"
            | Models.Processing -> "Processing"
            | Models.Shipped -> "Shipped"
            | Models.Delivered -> "Delivered"
            | Models.Cancelled -> "Cancelled"

    override _.Parse(value) =
        match value.ToString() with
        | "Pending" -> Models.Pending
        | "Processing" -> Models.Processing
        | "Shipped" -> Models.Shipped
        | "Delivered" -> Models.Delivered
        | "Cancelled" -> Models.Cancelled
        | other -> failwith $"Unknown OrderStatus: {other}"

// ลงทะเบียน type handlers ทั้งหมด
let registerAll () =
    SqlMapper.AddTypeHandler(OptionHandler<string>())
    SqlMapper.AddTypeHandler(OptionHandler<int>())
    SqlMapper.AddTypeHandler(OptionHandler<decimal>())
    SqlMapper.AddTypeHandler(OptionHandler<DateTime>())
    SqlMapper.AddTypeHandler(OptionHandler<bool>())
    SqlMapper.AddTypeHandler(DateTimeOffsetHandler())
    SqlMapper.AddTypeHandler(UriHandler())
    SqlMapper.AddTypeHandler(OrderStatusHandler())
```

---

## 5. การ Execute SQL (Executing SQL)

```fsharp
// DapperExamples.fs
module DapperExamples

open System
open System.Data
open Dapper
open Models

// ========================================
// 5.1 Execute (DDL และ DML ที่ไม่ return rows)
// ========================================

/// สร้างตาราง (Create Table)
let createTables (conn: IDbConnection) =
    let sql = """
        CREATE TABLE IF NOT EXISTS Users (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            Name TEXT NOT NULL,
            Email TEXT NOT NULL UNIQUE,
            CreatedAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            IsActive BOOLEAN NOT NULL DEFAULT 1
        );

        CREATE TABLE IF NOT EXISTS Products (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            Name TEXT NOT NULL,
            Price DECIMAL(10,2) NOT NULL,
            CategoryId INTEGER NOT NULL,
            Stock INTEGER NOT NULL DEFAULT 0,
            Description TEXT
        );

        CREATE TABLE IF NOT EXISTS Orders (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            UserId INTEGER NOT NULL,
            TotalAmount DECIMAL(10,2) NOT NULL,
            Status TEXT NOT NULL DEFAULT 'Pending',
            CreatedAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (UserId) REFERENCES Users(Id)
        );

        CREATE TABLE IF NOT EXISTS OrderItems (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            OrderId INTEGER NOT NULL,
            ProductId INTEGER NOT NULL,
            Quantity INTEGER NOT NULL,
            UnitPrice DECIMAL(10,2) NOT NULL,
            FOREIGN KEY (OrderId) REFERENCES Orders(Id),
            FOREIGN KEY (ProductId) REFERENCES Products(Id)
        );
    """
    conn.Execute(sql) |> ignore

/// ลบตาราง (Drop Table)
let dropTables (conn: IDbConnection) =
    let sql = """
        DROP TABLE IF EXISTS OrderItems;
        DROP TABLE IF EXISTS Orders;
        DROP TABLE IF EXISTS Products;
        DROP TABLE IF EXISTS Users;
    """
    conn.Execute(sql) |> ignore

// ========================================
// 5.2 Insert (การเพิ่มข้อมูล)
// ========================================

/// เพิ่ม User คนเดียว
let insertUser (conn: IDbConnection) (dto: CreateUserDto) =
    let sql = """
        INSERT INTO Users (Name, Email, CreatedAt, IsActive)
        VALUES (@Name, @Email, @CreatedAt, @IsActive);
        SELECT last_insert_rowid();
    """
    let parameters = {|
        Name = dto.Name
        Email = dto.Email
        CreatedAt = DateTime.UtcNow
        IsActive = true
    |}
    conn.QuerySingle<int>(sql, parameters)

/// เพิ่ม Product
let insertProduct (conn: IDbConnection) (name: string) (price: decimal) (categoryId: int) (stock: int) (description: string option) =
    let sql = """
        INSERT INTO Products (Name, Price, CategoryId, Stock, Description)
        VALUES (@Name, @Price, @CategoryId, @Stock, @Description);
        SELECT last_insert_rowid();
    """
    let parameters = {|
        Name = name
        Price = price
        CategoryId = categoryId
        Stock = stock
        Description = description |> Option.toObj
    |}
    conn.QuerySingle<int>(sql, parameters)

// ========================================
// 5.3 Query<T> สำหรับ single result
// ========================================

/// ค้นหา User ด้วย ID
let getUserById (conn: IDbConnection) (id: int) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users WHERE Id = @Id"
    conn.QuerySingleOrDefault<User>(sql, {| Id = id |})
    |> Option.ofObj

/// ค้นหา User ด้วย Email
let getUserByEmail (conn: IDbConnection) (email: string) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users WHERE Email = @Email"
    conn.QueryFirstOrDefault<User>(sql, {| Email = email |})
    |> Option.ofObj

// ========================================
// 5.4 Query<T> สำหรับ multiple results
// ========================================

/// ดึง User ทั้งหมด
let getAllUsers (conn: IDbConnection) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users ORDER BY Id"
    conn.Query<User>(sql) |> Seq.toList

/// ดึง Active Users
let getActiveUsers (conn: IDbConnection) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users WHERE IsActive = 1"
    conn.Query<User>(sql) |> Seq.toList

/// ค้นหา Users ด้วยชื่อ (partial match)
let searchUsersByName (conn: IDbConnection) (searchTerm: string) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users WHERE Name LIKE @SearchTerm"
    conn.Query<User>(sql, {| SearchTerm = $"%%{searchTerm}%%" |}) |> Seq.toList

// ========================================
// 5.5 QueryFirst, QueryFirstOrDefault
// ========================================

/// ดึง User แรกที่สร้าง
let getFirstCreatedUser (conn: IDbConnection) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users ORDER BY CreatedAt ASC"
    try
        conn.QueryFirst<User>(sql) |> Some
    with _ -> None  // QueryFirst throws if no rows

/// ดึง User ล่าสุด (หรือ None ถ้าไม่มี)
let getLatestUser (conn: IDbConnection) =
    let sql = "SELECT Id, Name, Email, CreatedAt, IsActive FROM Users ORDER BY CreatedAt DESC"
    conn.QueryFirstOrDefault<User>(sql) |> Option.ofObj

// ========================================
// 5.6 Update
// ========================================

/// อัปเดต User
let updateUser (conn: IDbConnection) (dto: UpdateUserDto) =
    let sql = """
        UPDATE Users
        SET Name = @Name, Email = @Email, IsActive = @IsActive
        WHERE Id = @Id
    """
    let rowsAffected = conn.Execute(sql, dto)
    rowsAffected > 0

/// เปิด/ปิดใช้งาน User
let setUserActive (conn: IDbConnection) (id: int) (isActive: bool) =
    let sql = "UPDATE Users SET IsActive = @IsActive WHERE Id = @Id"
    conn.Execute(sql, {| Id = id; IsActive = isActive |}) > 0

// ========================================
// 5.7 Delete
// ========================================

/// ลบ User ด้วย ID
let deleteUser (conn: IDbConnection) (id: int) =
    let sql = "DELETE FROM Users WHERE Id = @Id"
    conn.Execute(sql, {| Id = id |}) > 0

/// ลบ Users ที่ไม่ active (soft delete ด้วย IsActive)
let deleteInactiveUsers (conn: IDbConnection) =
    let sql = "DELETE FROM Users WHERE IsActive = 0"
    conn.Execute(sql)
```

---

## 6. Parameters (การส่ง Parameters)

```fsharp
// Parameters.fs
module ParameterExamples

open System.Data
open Dapper
open Models

// ========================================
// 6.1 Anonymous types (แบบ F# anonymous records)
// ========================================

let queryWithAnonymousParams (conn: IDbConnection) =
    let sql = "SELECT * FROM Users WHERE Id = @Id AND IsActive = @IsActive"

    // ใช้ anonymous record ใน F#
    let parameters = {| Id = 1; IsActive = true |}
    conn.Query<User>(sql, parameters) |> Seq.toList

// ========================================
// 6.2 DynamicParameters
// ========================================

let queryWithDynamicParams (conn: IDbConnection) (minPrice: decimal) (maxPrice: decimal option) =
    let sql =
        match maxPrice with
        | Some _ -> "SELECT * FROM Products WHERE Price >= @MinPrice AND Price <= @MaxPrice"
        | None -> "SELECT * FROM Products WHERE Price >= @MinPrice"

    let parameters = DynamicParameters()
    parameters.Add("MinPrice", minPrice)
    maxPrice |> Option.iter (fun v -> parameters.Add("MaxPrice", v))

    conn.Query<Product>(sql, parameters) |> Seq.toList

// ========================================
// 6.3 List parameters (IN clause)
// ========================================

let getUsersByIds (conn: IDbConnection) (ids: int list) =
    if ids.IsEmpty then []
    else
        let sql = "SELECT * FROM Users WHERE Id IN @Ids"
        conn.Query<User>(sql, {| Ids = ids |}) |> Seq.toList

let getProductsByCategories (conn: IDbConnection) (categoryIds: int list) =
    if categoryIds.IsEmpty then []
    else
        let sql = "SELECT * FROM Products WHERE CategoryId IN @CategoryIds ORDER BY Name"
        conn.Query<Product>(sql, {| CategoryIds = categoryIds |}) |> Seq.toList

// ========================================
// 6.4 Paged queries
// ========================================

let getPagedUsers (conn: IDbConnection) (page: int) (pageSize: int) =
    let offset = (page - 1) * pageSize
    let sql = """
        SELECT Id, Name, Email, CreatedAt, IsActive
        FROM Users
        ORDER BY Id
        LIMIT @PageSize OFFSET @Offset
    """
    let countSql = "SELECT COUNT(*) FROM Users"

    let users = conn.Query<User>(sql, {| PageSize = pageSize; Offset = offset |}) |> Seq.toList
    let totalCount = conn.QuerySingle<int>(countSql)

    {
        Items = users
        TotalCount = totalCount
        Page = page
        PageSize = pageSize
        TotalPages = (totalCount + pageSize - 1) / pageSize
    }

// ========================================
// 6.5 Output parameters
// ========================================

let insertUserWithOutputParam (conn: IDbConnection) (name: string) (email: string) =
    // SQLite ไม่รองรับ OUTPUT parameters โดยตรง แต่ใช้ RETURNING (PostgreSQL)
    // หรือ last_insert_rowid() (SQLite)
    let sql = """
        INSERT INTO Users (Name, Email, CreatedAt, IsActive)
        VALUES (@Name, @Email, datetime('now'), 1);
        SELECT last_insert_rowid() AS NewId;
    """
    conn.QuerySingle<int>(sql, {| Name = name; Email = email |})
```

---

## 7. Stored Procedures (Stored Procedures)

```fsharp
// StoredProcedures.fs
module StoredProcedureExamples

open System.Data
open Dapper
open Models

// ========================================
// 7.1 การเรียก Stored Procedure
// ========================================

/// เรียก stored procedure สำหรับดึง Users ตาม status
let getUsersByStatusSP (conn: IDbConnection) (isActive: bool) =
    // สร้าง stored procedure ก่อน (SQLite ไม่รองรับ SP, ใช้ SQL Server/PostgreSQL)
    let parameters = DynamicParameters()
    parameters.Add("IsActive", isActive)

    conn.Query<User>(
        "sp_GetUsersByStatus",
        parameters,
        commandType = CommandType.StoredProcedure
    ) |> Seq.toList

/// เรียก stored procedure พร้อม output parameters
let createOrderSP (conn: IDbConnection) (userId: int) (totalAmount: decimal) =
    let parameters = DynamicParameters()
    parameters.Add("UserId", userId)
    parameters.Add("TotalAmount", totalAmount)
    parameters.Add("OrderId", dbType = DbType.Int32, direction = ParameterDirection.Output)
    parameters.Add("ErrorMessage", dbType = DbType.String, size = 500, direction = ParameterDirection.Output)

    conn.Execute(
        "sp_CreateOrder",
        parameters,
        commandType = CommandType.StoredProcedure
    ) |> ignore

    let orderId = parameters.Get<int>("OrderId")
    let errorMsg = parameters.Get<string>("ErrorMessage")

    if String.isNullOrEmpty errorMsg then
        Ok orderId
    else
        Error errorMsg

// ========================================
// 7.2 Table-valued function
// ========================================

let getProductsByPriceRange (conn: IDbConnection) (minPrice: decimal) (maxPrice: decimal) =
    // ตัวอย่างสำหรับ SQL Server
    let sql = "SELECT * FROM fn_GetProductsByPriceRange(@MinPrice, @MaxPrice)"
    conn.Query<Product>(sql, {| MinPrice = minPrice; MaxPrice = maxPrice |}) |> Seq.toList
```

---

## 8. Transactions (การจัดการ Transactions)

```fsharp
// Transactions.fs
module TransactionExamples

open System.Data
open Dapper
open Models

// ========================================
// 8.1 Basic Transaction
// ========================================

/// โอนเงินระหว่าง accounts (ตัวอย่าง transaction)
let transferFunds (conn: IDbConnection) (fromUserId: int) (toUserId: int) (amount: decimal) =
    use transaction = conn.BeginTransaction()
    try
        // ตรวจสอบ balance ก่อน
        let balanceSql = "SELECT Balance FROM Accounts WHERE UserId = @UserId"
        let fromBalance = conn.QuerySingle<decimal>(balanceSql, {| UserId = fromUserId |}, transaction)

        if fromBalance < amount then
            transaction.Rollback()
            Error "Insufficient funds"
        else
            // หักเงินจาก sender
            let debitSql = "UPDATE Accounts SET Balance = Balance - @Amount WHERE UserId = @UserId"
            conn.Execute(debitSql, {| Amount = amount; UserId = fromUserId |}, transaction) |> ignore

            // เพิ่มเงินให้ receiver
            let creditSql = "UPDATE Accounts SET Balance = Balance + @Amount WHERE UserId = @UserId"
            conn.Execute(creditSql, {| Amount = amount; UserId = toUserId |}, transaction) |> ignore

            // บันทึก transaction log
            let logSql = """
                INSERT INTO TransactionLog (FromUserId, ToUserId, Amount, CreatedAt)
                VALUES (@FromUserId, @ToUserId, @Amount, @CreatedAt)
            """
            conn.Execute(logSql, {|
                FromUserId = fromUserId
                ToUserId = toUserId
                Amount = amount
                CreatedAt = System.DateTime.UtcNow
            |}, transaction) |> ignore

            transaction.Commit()
            Ok "Transfer successful"
    with ex ->
        transaction.Rollback()
        Error ex.Message

// ========================================
// 8.2 Transaction กับ multiple operations
// ========================================

/// สร้าง Order พร้อม OrderItems
let createOrder (conn: IDbConnection) (userId: int) (items: (int * int * decimal) list) =
    // items = (productId, quantity, unitPrice) list
    use transaction = conn.BeginTransaction()
    try
        // สร้าง Order
        let orderSql = """
            INSERT INTO Orders (UserId, TotalAmount, Status, CreatedAt)
            VALUES (@UserId, @TotalAmount, 'Pending', @CreatedAt);
            SELECT last_insert_rowid();
        """
        let totalAmount = items |> List.sumBy (fun (_, qty, price) -> decimal qty * price)
        let orderId = conn.QuerySingle<int>(orderSql, {|
            UserId = userId
            TotalAmount = totalAmount
            CreatedAt = System.DateTime.UtcNow
        |}, transaction)

        // สร้าง OrderItems
        let itemSql = """
            INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice)
            VALUES (@OrderId, @ProductId, @Quantity, @UnitPrice)
        """
        for (productId, quantity, unitPrice) in items do
            conn.Execute(itemSql, {|
                OrderId = orderId
                ProductId = productId
                Quantity = quantity
                UnitPrice = unitPrice
            |}, transaction) |> ignore

            // อัปเดต stock
            let updateStockSql = "UPDATE Products SET Stock = Stock - @Quantity WHERE Id = @ProductId AND Stock >= @Quantity"
            let affected = conn.Execute(updateStockSql, {|
                Quantity = quantity
                ProductId = productId
            |}, transaction)

            if affected = 0 then
                failwith $"Insufficient stock for product {productId}"

        transaction.Commit()
        Ok orderId
    with ex ->
        transaction.Rollback()
        Error ex.Message

// ========================================
// 8.3 SavePoints
// ========================================

let complexTransaction (conn: IDbConnection) =
    // Note: SavePoints ขึ้นอยู่กับ database ที่ใช้
    use transaction = conn.BeginTransaction()
    try
        // Phase 1
        conn.Execute("UPDATE Users SET IsActive = 1 WHERE Id = 1", transaction = transaction) |> ignore

        // Phase 2 - ถ้าส่วนนี้ fail, rollback เฉพาะ phase 2
        try
            conn.Execute("UPDATE Users SET Name = 'Updated' WHERE Id = 999", transaction = transaction) |> ignore
        with _ ->
            () // ignore phase 2 errors

        transaction.Commit()
        Ok "Done"
    with ex ->
        transaction.Rollback()
        Error ex.Message
```

---

## 9. Multi-mapping (การ Map ข้อมูลจากหลายตาราง)

```fsharp
// MultiMapping.fs
module MultiMappingExamples

open System.Data
open Dapper
open Models

// ========================================
// 9.1 One-to-One mapping
// ========================================

type UserWithProfile = {
    User: User
    Profile: UserProfile option
}

and UserProfile = {
    Id: int
    UserId: int
    Bio: string
    AvatarUrl: string option
}

let getUserWithProfile (conn: IDbConnection) (userId: int) =
    let sql = """
        SELECT u.Id, u.Name, u.Email, u.CreatedAt, u.IsActive,
               p.Id, p.UserId, p.Bio, p.AvatarUrl
        FROM Users u
        LEFT JOIN UserProfiles p ON u.Id = p.UserId
        WHERE u.Id = @UserId
    """
    conn.Query<User, UserProfile, UserWithProfile>(
        sql,
        fun user profile ->
            { User = user; Profile = if profile = Unchecked.defaultof<UserProfile> then None else Some profile },
        {| UserId = userId |},
        splitOn = "Id"
    ) |> Seq.tryHead

// ========================================
// 9.2 One-to-Many mapping
// ========================================

type OrderWithItems = {
    Order: Order
    Items: OrderItem list
}

let getOrderWithItems (conn: IDbConnection) (orderId: int) =
    let sql = """
        SELECT o.Id, o.UserId, o.TotalAmount, o.Status, o.CreatedAt,
               oi.Id, oi.OrderId, oi.ProductId, oi.Quantity, oi.UnitPrice
        FROM Orders o
        LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
        WHERE o.Id = @OrderId
    """
    let orderDict = System.Collections.Generic.Dictionary<int, OrderWithItems>()

    conn.Query<Order, OrderItem, Order>(
        sql,
        fun order item ->
            if not (orderDict.ContainsKey(order.Id)) then
                orderDict.[order.Id] <- { Order = order; Items = [] }

            if item <> Unchecked.defaultof<OrderItem> then
                let existing = orderDict.[order.Id]
                orderDict.[order.Id] <- { existing with Items = item :: existing.Items }

            order,
        {| OrderId = orderId |},
        splitOn = "Id"
    ) |> ignore

    orderDict.Values |> Seq.tryHead
    |> Option.map (fun owm -> { owm with Items = List.rev owm.Items })

// ========================================
// 9.3 Triple mapping
// ========================================

type FullOrderInfo = {
    Order: Order
    User: User
    Items: OrderItem list
}

let getFullOrderInfo (conn: IDbConnection) (orderId: int) =
    let sql = """
        SELECT o.Id, o.UserId, o.TotalAmount, o.Status, o.CreatedAt,
               u.Id, u.Name, u.Email, u.CreatedAt, u.IsActive,
               oi.Id, oi.OrderId, oi.ProductId, oi.Quantity, oi.UnitPrice
        FROM Orders o
        INNER JOIN Users u ON o.UserId = u.Id
        LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
        WHERE o.Id = @OrderId
    """
    let orderDict = System.Collections.Generic.Dictionary<int, FullOrderInfo>()

    conn.Query<Order, User, OrderItem, Order>(
        sql,
        (fun order user item ->
            if not (orderDict.ContainsKey(order.Id)) then
                orderDict.[order.Id] <- { Order = order; User = user; Items = [] }

            if item <> Unchecked.defaultof<OrderItem> then
                let existing = orderDict.[order.Id]
                orderDict.[order.Id] <- { existing with Items = item :: existing.Items }

            order),
        {| OrderId = orderId |},
        splitOn = "Id,Id"
    ) |> ignore

    orderDict.Values |> Seq.tryHead
    |> Option.map (fun info -> { info with Items = List.rev info.Items })
```

---

## 10. Multi-result Sets (ผลลัพธ์หลายชุด)

```fsharp
// MultiResults.fs
module MultiResultExamples

open System.Data
open Dapper
open Models

// ========================================
// 10.1 QueryMultiple
// ========================================

type DashboardData = {
    TotalUsers: int
    ActiveUsers: int
    RecentOrders: Order list
    TopProducts: Product list
}

let getDashboardData (conn: IDbConnection) =
    let sql = """
        SELECT COUNT(*) FROM Users;
        SELECT COUNT(*) FROM Users WHERE IsActive = 1;
        SELECT * FROM Orders ORDER BY CreatedAt DESC LIMIT 5;
        SELECT p.*, SUM(oi.Quantity) as TotalSold
        FROM Products p
        LEFT JOIN OrderItems oi ON p.Id = oi.ProductId
        GROUP BY p.Id
        ORDER BY TotalSold DESC
        LIMIT 5;
    """
    use multi = conn.QueryMultiple(sql)

    {
        TotalUsers = multi.ReadSingle<int>()
        ActiveUsers = multi.ReadSingle<int>()
        RecentOrders = multi.Read<Order>() |> Seq.toList
        TopProducts = multi.Read<Product>() |> Seq.toList
    }

/// ดึงข้อมูล User พร้อมสถิติ
let getUserDashboard (conn: IDbConnection) (userId: int) =
    let sql = """
        SELECT * FROM Users WHERE Id = @UserId;
        SELECT COUNT(*) FROM Orders WHERE UserId = @UserId;
        SELECT SUM(TotalAmount) FROM Orders WHERE UserId = @UserId AND Status = 'Delivered';
        SELECT * FROM Orders WHERE UserId = @UserId ORDER BY CreatedAt DESC LIMIT 10;
    """
    use multi = conn.QueryMultiple(sql, {| UserId = userId |})

    let user = multi.ReadSingleOrDefault<User>()
    let orderCount = multi.ReadSingle<int>()
    let totalSpent = multi.ReadSingle<decimal option>()
    let recentOrders = multi.Read<Order>() |> Seq.toList

    if user = Unchecked.defaultof<User> then None
    else
        Some {|
            User = user
            OrderCount = orderCount
            TotalSpent = totalSpent |> Option.defaultValue 0m
            RecentOrders = recentOrders
        |}
```

---

## 11. Bulk Operations (การ Insert/Update จำนวนมาก)

```fsharp
// BulkOperations.fs
module BulkOperations

open System.Data
open Dapper
open Models

// ========================================
// 11.1 Bulk Insert
// ========================================

/// Insert Users จำนวนมากด้วย transaction
let bulkInsertUsers (conn: IDbConnection) (users: CreateUserDto list) =
    if users.IsEmpty then 0
    else
        let sql = """
            INSERT INTO Users (Name, Email, CreatedAt, IsActive)
            VALUES (@Name, @Email, @CreatedAt, @IsActive)
        """
        use transaction = conn.BeginTransaction()
        try
            let parameters =
                users |> List.map (fun u -> {|
                    Name = u.Name
                    Email = u.Email
                    CreatedAt = System.DateTime.UtcNow
                    IsActive = true
                |})

            let rowsAffected = conn.Execute(sql, parameters, transaction)
            transaction.Commit()
            rowsAffected
        with ex ->
            transaction.Rollback()
            raise ex

/// Bulk insert ด้วย chunking (สำหรับข้อมูลจำนวนมาก)
let bulkInsertInChunks (conn: IDbConnection) (users: CreateUserDto list) (chunkSize: int) =
    let chunks = users |> List.chunkBySize chunkSize
    let mutable totalInserted = 0

    for chunk in chunks do
        let inserted = bulkInsertUsers conn chunk
        totalInserted <- totalInserted + inserted

    totalInserted

// ========================================
// 11.2 Bulk Update
// ========================================

let bulkUpdateUserStatus (conn: IDbConnection) (updates: (int * bool) list) =
    let sql = "UPDATE Users SET IsActive = @IsActive WHERE Id = @Id"
    use transaction = conn.BeginTransaction()
    try
        let parameters =
            updates |> List.map (fun (id, isActive) -> {| Id = id; IsActive = isActive |})
        let rowsAffected = conn.Execute(sql, parameters, transaction)
        transaction.Commit()
        rowsAffected
    with ex ->
        transaction.Rollback()
        raise ex

// ========================================
// 11.3 Upsert (Insert or Update)
// ========================================

/// SQLite UPSERT ด้วย INSERT OR REPLACE
let upsertUser (conn: IDbConnection) (user: User) =
    let sql = """
        INSERT OR REPLACE INTO Users (Id, Name, Email, CreatedAt, IsActive)
        VALUES (@Id, @Name, @Email, @CreatedAt, @IsActive)
    """
    conn.Execute(sql, user) > 0

/// PostgreSQL UPSERT ด้วย ON CONFLICT
let upsertUserPostgres (conn: IDbConnection) (user: User) =
    let sql = """
        INSERT INTO Users (Id, Name, Email, CreatedAt, IsActive)
        VALUES (@Id, @Name, @Email, @CreatedAt, @IsActive)
        ON CONFLICT (Email)
        DO UPDATE SET
            Name = EXCLUDED.Name,
            IsActive = EXCLUDED.IsActive
    """
    conn.Execute(sql, user) > 0
```

---

## 12. Async Operations (การทำงานแบบ Async)

```fsharp
// AsyncDapper.fs
module AsyncDapper

open System.Data
open System.Threading.Tasks
open Dapper
open Models

// ========================================
// 12.1 Async Query methods
// ========================================

/// ดึง User ทั้งหมดแบบ async
let getAllUsersAsync (conn: IDbConnection) =
    task {
        let sql = "SELECT * FROM Users ORDER BY Id"
        let! result = conn.QueryAsync<User>(sql)
        return result |> Seq.toList
    }

/// ค้นหา User ด้วย ID แบบ async
let getUserByIdAsync (conn: IDbConnection) (id: int) =
    task {
        let sql = "SELECT * FROM Users WHERE Id = @Id"
        let! result = conn.QueryFirstOrDefaultAsync<User>(sql, {| Id = id |})
        return result |> Option.ofObj
    }

/// Insert User แบบ async
let insertUserAsync (conn: IDbConnection) (dto: CreateUserDto) =
    task {
        let sql = """
            INSERT INTO Users (Name, Email, CreatedAt, IsActive)
            VALUES (@Name, @Email, @CreatedAt, @IsActive);
            SELECT last_insert_rowid();
        """
        let parameters = {|
            Name = dto.Name
            Email = dto.Email
            CreatedAt = System.DateTime.UtcNow
            IsActive = true
        |}
        return! conn.QuerySingleAsync<int>(sql, parameters)
    }

/// Async transaction
let createOrderAsync (conn: IDbConnection) (userId: int) (items: (int * int * decimal) list) =
    task {
        use transaction = conn.BeginTransaction()
        try
            let totalAmount = items |> List.sumBy (fun (_, qty, price) -> decimal qty * price)
            let orderSql = """
                INSERT INTO Orders (UserId, TotalAmount, Status, CreatedAt)
                VALUES (@UserId, @TotalAmount, 'Pending', @CreatedAt);
                SELECT last_insert_rowid();
            """
            let! orderId = conn.QuerySingleAsync<int>(orderSql, {|
                UserId = userId
                TotalAmount = totalAmount
                CreatedAt = System.DateTime.UtcNow
            |}, transaction)

            let itemSql = """
                INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice)
                VALUES (@OrderId, @ProductId, @Quantity, @UnitPrice)
            """
            for (productId, quantity, unitPrice) in items do
                let! _ = conn.ExecuteAsync(itemSql, {|
                    OrderId = orderId
                    ProductId = productId
                    Quantity = quantity
                    UnitPrice = unitPrice
                |}, transaction)
                ()

            transaction.Commit()
            return Ok orderId
        with ex ->
            transaction.Rollback()
            return Error ex.Message
    }

// ========================================
// 12.2 Parallel queries
// ========================================

let getMultipleDataParallel (conn: IDbConnection) =
    task {
        // Note: ต้องใช้ connection แยกสำหรับ parallel queries
        let usersTask = conn.QueryAsync<User>("SELECT * FROM Users")
        let productsTask = conn.QueryAsync<Product>("SELECT * FROM Products")
        let ordersTask = conn.QueryAsync<Order>("SELECT * FROM Orders")

        let! users = usersTask
        let! products = productsTask
        let! orders = ordersTask

        return {|
            Users = users |> Seq.toList
            Products = products |> Seq.toList
            Orders = orders |> Seq.toList
        |}
    }
```

---

## 13. Repository Pattern (ตัวอย่าง Repository)

```fsharp
// UserRepository.fs
module UserRepository

open System.Data
open Dapper
open Models

// Interface สำหรับ User Repository
type IUserRepository =
    abstract member GetByIdAsync: int -> System.Threading.Tasks.Task<User option>
    abstract member GetAllAsync: unit -> System.Threading.Tasks.Task<User list>
    abstract member CreateAsync: CreateUserDto -> System.Threading.Tasks.Task<int>
    abstract member UpdateAsync: UpdateUserDto -> System.Threading.Tasks.Task<bool>
    abstract member DeleteAsync: int -> System.Threading.Tasks.Task<bool>
    abstract member SearchAsync: string -> System.Threading.Tasks.Task<User list>

// Implementation
type DapperUserRepository(conn: IDbConnection) =
    interface IUserRepository with
        member _.GetByIdAsync(id) = task {
            let sql = "SELECT * FROM Users WHERE Id = @Id"
            let! result = conn.QueryFirstOrDefaultAsync<User>(sql, {| Id = id |})
            return result |> Option.ofObj
        }

        member _.GetAllAsync() = task {
            let sql = "SELECT * FROM Users ORDER BY Name"
            let! result = conn.QueryAsync<User>(sql)
            return result |> Seq.toList
        }

        member _.CreateAsync(dto) = task {
            let sql = """
                INSERT INTO Users (Name, Email, CreatedAt, IsActive)
                VALUES (@Name, @Email, @CreatedAt, 1);
                SELECT last_insert_rowid();
            """
            return! conn.QuerySingleAsync<int>(sql, {|
                Name = dto.Name
                Email = dto.Email
                CreatedAt = System.DateTime.UtcNow
            |})
        }

        member _.UpdateAsync(dto) = task {
            let sql = """
                UPDATE Users
                SET Name = @Name, Email = @Email, IsActive = @IsActive
                WHERE Id = @Id
            """
            let! rows = conn.ExecuteAsync(sql, dto)
            return rows > 0
        }

        member _.DeleteAsync(id) = task {
            let sql = "DELETE FROM Users WHERE Id = @Id"
            let! rows = conn.ExecuteAsync(sql, {| Id = id |})
            return rows > 0
        }

        member _.SearchAsync(term) = task {
            let sql = "SELECT * FROM Users WHERE Name LIKE @Term OR Email LIKE @Term"
            let! result = conn.QueryAsync<User>(sql, {| Term = $"%%{term}%%" |})
            return result |> Seq.toList
        }
```

---

## 14. Complete CRUD Example (ตัวอย่างครบวงจร)

```fsharp
// Program.fs
module Program

open System
open System.Data
open Microsoft.Data.Sqlite
open Dapper
open Models

let createConnection () =
    let conn = new SqliteConnection("Data Source=:memory:")
    conn.Open()
    conn :> IDbConnection

let setupDatabase (conn: IDbConnection) =
    let sql = """
        CREATE TABLE Users (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            Name TEXT NOT NULL,
            Email TEXT NOT NULL UNIQUE,
            CreatedAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
            IsActive BOOLEAN NOT NULL DEFAULT 1
        );

        CREATE TABLE Products (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            Name TEXT NOT NULL,
            Price DECIMAL(10,2) NOT NULL,
            CategoryId INTEGER NOT NULL DEFAULT 1,
            Stock INTEGER NOT NULL DEFAULT 0,
            Description TEXT
        );

        CREATE TABLE Orders (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            UserId INTEGER NOT NULL,
            TotalAmount DECIMAL(10,2) NOT NULL,
            Status TEXT NOT NULL DEFAULT 'Pending',
            CreatedAt DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
        );

        CREATE TABLE OrderItems (
            Id INTEGER PRIMARY KEY AUTOINCREMENT,
            OrderId INTEGER NOT NULL,
            ProductId INTEGER NOT NULL,
            Quantity INTEGER NOT NULL,
            UnitPrice DECIMAL(10,2) NOT NULL
        );
    """
    conn.Execute(sql) |> ignore
    printfn "Database setup complete"

let seedData (conn: IDbConnection) =
    // Insert Users
    let usersSql = """
        INSERT INTO Users (Name, Email, CreatedAt, IsActive) VALUES
        ('Alice Johnson', 'alice@example.com', datetime('now', '-30 days'), 1),
        ('Bob Smith', 'bob@example.com', datetime('now', '-20 days'), 1),
        ('Charlie Brown', 'charlie@example.com', datetime('now', '-10 days'), 0),
        ('Diana Prince', 'diana@example.com', datetime('now', '-5 days'), 1),
        ('Eve Wilson', 'eve@example.com', datetime('now'), 1)
    """
    conn.Execute(usersSql) |> ignore

    // Insert Products
    let productsSql = """
        INSERT INTO Products (Name, Price, CategoryId, Stock, Description) VALUES
        ('Laptop Pro', 49999.00, 1, 50, 'High-performance laptop'),
        ('Wireless Mouse', 899.00, 2, 200, 'Ergonomic wireless mouse'),
        ('Mechanical Keyboard', 2499.00, 2, 150, 'RGB mechanical keyboard'),
        ('Monitor 27"', 12999.00, 1, 30, '4K IPS monitor'),
        ('USB-C Hub', 1299.00, 2, 300, '7-in-1 USB-C hub'),
        ('Webcam HD', 1999.00, 3, 100, '1080p webcam'),
        ('Headphones', 3499.00, 3, 80, 'Noise cancelling headphones')
    """
    conn.Execute(productsSql) |> ignore

    // Insert Orders
    let ordersSql = """
        INSERT INTO Orders (UserId, TotalAmount, Status, CreatedAt) VALUES
        (1, 53397.00, 'Delivered', datetime('now', '-25 days')),
        (2, 4698.00, 'Shipped', datetime('now', '-15 days')),
        (4, 14298.00, 'Processing', datetime('now', '-3 days')),
        (5, 3499.00, 'Pending', datetime('now', '-1 days'))
    """
    conn.Execute(ordersSql) |> ignore

    // Insert OrderItems
    let itemsSql = """
        INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice) VALUES
        (1, 1, 1, 49999.00),
        (1, 2, 1, 899.00),
        (1, 5, 2, 1299.00),  -- 2x USB-C Hub = 2598.00, total = 53396 (close)
        (2, 3, 1, 2499.00),
        (2, 2, 2, 899.00),   -- 2x Mouse = 1798, but let's say 2 items
        (3, 4, 1, 12999.00),
        (3, 5, 1, 1299.00),
        (4, 7, 1, 3499.00)
    """
    conn.Execute(itemsSql) |> ignore

    printfn "Seed data inserted"

[<EntryPoint>]
let main _ =
    // ลงทะเบียน type handlers
    TypeHandlers.registerAll()

    use conn = createConnection()
    setupDatabase conn
    seedData conn

    printfn "\n=== DAPPER CRUD DEMO ==="
    printfn "================================"

    // READ - ดึงข้อมูลทั้งหมด
    printfn "\n--- All Users ---"
    let allUsers = conn.Query<User>("SELECT * FROM Users ORDER BY Id") |> Seq.toList
    for user in allUsers do
        printfn "  [%d] %s <%s> - Active: %b" user.Id user.Name user.Email user.IsActive

    // READ - ค้นหา User ด้วย ID
    printfn "\n--- User by ID (2) ---"
    let user2 = conn.QueryFirstOrDefault<User>("SELECT * FROM Users WHERE Id = @Id", {| Id = 2 |})
    if user2 <> Unchecked.defaultof<User> then
        printfn "  Found: %s <%s>" user2.Name user2.Email
    else
        printfn "  Not found"

    // CREATE - เพิ่ม User ใหม่
    printfn "\n--- Create New User ---"
    let newUserSql = """
        INSERT INTO Users (Name, Email, CreatedAt, IsActive)
        VALUES (@Name, @Email, @CreatedAt, @IsActive);
        SELECT last_insert_rowid();
    """
    let newUserId = conn.QuerySingle<int>(newUserSql, {|
        Name = "Frank Castle"
        Email = "frank@example.com"
        CreatedAt = DateTime.UtcNow
        IsActive = true
    |})
    printfn "  Created user with ID: %d" newUserId

    // UPDATE - แก้ไขข้อมูล
    printfn "\n--- Update User ---"
    let updateSql = "UPDATE Users SET Name = @Name WHERE Id = @Id"
    let rowsAffected = conn.Execute(updateSql, {| Name = "Frank Castle Jr."; Id = newUserId |})
    printfn "  Updated %d row(s)" rowsAffected

    // READ - ยืนยันการแก้ไข
    let updatedUser = conn.QueryFirstOrDefault<User>("SELECT * FROM Users WHERE Id = @Id", {| Id = newUserId |})
    if updatedUser <> Unchecked.defaultof<User> then
        printfn "  Updated name: %s" updatedUser.Name

    // Paged query
    printfn "\n--- Paged Users (Page 1, Size 3) ---"
    let pagedSql = "SELECT * FROM Users ORDER BY Id LIMIT @PageSize OFFSET @Offset"
    let pagedUsers = conn.Query<User>(pagedSql, {| PageSize = 3; Offset = 0 |}) |> Seq.toList
    for u in pagedUsers do
        printfn "  [%d] %s" u.Id u.Name

    // Products
    printfn "\n--- All Products ---"
    let products = conn.Query<Product>("SELECT * FROM Products ORDER BY Price DESC") |> Seq.toList
    for p in products do
        printfn "  [%d] %s - ฿%.2f (Stock: %d)" p.Id p.Name p.Price p.Stock

    // Orders with items count
    printfn "\n--- Orders Summary ---"
    let ordersSql = """
        SELECT o.Id, o.UserId, u.Name as UserName, o.TotalAmount, o.Status,
               COUNT(oi.Id) as ItemCount
        FROM Orders o
        INNER JOIN Users u ON o.UserId = u.Id
        LEFT JOIN OrderItems oi ON o.Id = oi.OrderId
        GROUP BY o.Id, o.UserId, u.Name, o.TotalAmount, o.Status
        ORDER BY o.Id
    """
    let orderSummaries = conn.Query(ordersSql) |> Seq.toList
    for o in orderSummaries do
        let dict = o :> System.Collections.Generic.IDictionary<string, obj>
        printfn "  Order #%A - %A (฿%A) - Status: %A - Items: %A"
            dict.["Id"] dict.["UserName"] dict.["TotalAmount"] dict.["Status"] dict.["ItemCount"]

    // Transaction example
    printfn "\n--- Transaction: Create New Order ---"
    use transaction = conn.BeginTransaction()
    try
        let createOrderSql = """
            INSERT INTO Orders (UserId, TotalAmount, Status, CreatedAt)
            VALUES (@UserId, @TotalAmount, 'Pending', @CreatedAt);
            SELECT last_insert_rowid();
        """
        let newOrderId = conn.QuerySingle<int>(createOrderSql, {|
            UserId = 1
            TotalAmount = 6997.00m
            CreatedAt = DateTime.UtcNow
        |}, transaction)

        let addItemSql = """
            INSERT INTO OrderItems (OrderId, ProductId, Quantity, UnitPrice)
            VALUES (@OrderId, @ProductId, @Quantity, @UnitPrice)
        """
        conn.Execute(addItemSql, {| OrderId = newOrderId; ProductId = 2; Quantity = 1; UnitPrice = 899.00m |}, transaction) |> ignore
        conn.Execute(addItemSql, {| OrderId = newOrderId; ProductId = 6; Quantity = 3; UnitPrice = 1999.00m |}, transaction) |> ignore  // 3x webcam = 5997

        transaction.Commit()
        printfn "  Created order #%d successfully" newOrderId
    with ex ->
        transaction.Rollback()
        printfn "  Transaction failed: %s" ex.Message

    // QueryMultiple
    printfn "\n--- Dashboard Data (QueryMultiple) ---"
    let dashSql = """
        SELECT COUNT(*) FROM Users;
        SELECT COUNT(*) FROM Users WHERE IsActive = 1;
        SELECT COUNT(*) FROM Orders;
        SELECT SUM(TotalAmount) FROM Orders WHERE Status = 'Delivered';
    """
    use multi = conn.QueryMultiple(dashSql)
    let totalUsers = multi.ReadSingle<int>()
    let activeUsers = multi.ReadSingle<int>()
    let totalOrders = multi.ReadSingle<int>()
    let revenue = multi.ReadSingle<decimal option>()

    printfn "  Total Users: %d" totalUsers
    printfn "  Active Users: %d" activeUsers
    printfn "  Total Orders: %d" totalOrders
    printfn "  Total Revenue: ฿%.2f" (revenue |> Option.defaultValue 0m)

    // DELETE
    printfn "\n--- Delete User ---"
    let deleteSql = "DELETE FROM Users WHERE Id = @Id"
    let deleted = conn.Execute(deleteSql, {| Id = newUserId |}) > 0
    printfn "  User deleted: %b" deleted

    printfn "\n=== Demo Complete ==="
    0
```

---

## 15. Option Types กับ Dapper (Option Types with Dapper)

```fsharp
// OptionTypes.fs
module OptionTypeExamples

open System
open System.Data
open Dapper

// ========================================
// การใช้ Option types กับ Dapper
// ========================================

// Record ที่มี nullable fields
type ProductWithOptions = {
    Id: int
    Name: string
    Price: decimal
    Description: string option   // nullable
    DiscountPercent: float option // nullable
    ExpiryDate: DateTime option   // nullable
}

// วิธีที่ 1: ใช้ Type Handlers (แนะนำ)
type OptionTypeHandler<'T>() =
    inherit SqlMapper.TypeHandler<'T option>()

    override _.SetValue(parameter, value) =
        match value with
        | Some v -> parameter.Value <- box v
        | None   -> parameter.Value <- DBNull.Value

    override _.Parse(value) =
        if isNull value || value = box DBNull.Value then None
        else Some (value :?> 'T)

// Register handlers
let registerOptionHandlers() =
    SqlMapper.AddTypeHandler(OptionTypeHandler<string>())
    SqlMapper.AddTypeHandler(OptionTypeHandler<int>())
    SqlMapper.AddTypeHandler(OptionTypeHandler<float>())
    SqlMapper.AddTypeHandler(OptionTypeHandler<decimal>())
    SqlMapper.AddTypeHandler(OptionTypeHandler<DateTime>())

// วิธีที่ 2: Map manually หลัง Query
let getProductManual (conn: IDbConnection) (id: int) =
    let sql = """
        SELECT Id, Name, Price, Description, DiscountPercent, ExpiryDate
        FROM Products WHERE Id = @Id
    """
    let rows = conn.Query(sql, {| Id = id |})
    rows
    |> Seq.tryHead
    |> Option.map (fun row ->
        let dict = row :> System.Collections.Generic.IDictionary<string, obj>
        let getOpt key =
            match dict.TryGetValue(key) with
            | true, v when v <> null && v <> box DBNull.Value -> Some v
            | _ -> None

        {
            Id = dict.["Id"] :?> int
            Name = dict.["Name"] :?> string
            Price = dict.["Price"] :?> decimal
            Description = getOpt "Description" |> Option.map string
            DiscountPercent = getOpt "DiscountPercent" |> Option.map (fun v -> v :?> float)
            ExpiryDate = getOpt "ExpiryDate" |> Option.map (fun v -> v :?> DateTime)
        }
    )

// Insert ด้วย Option types
let insertProductWithOptions (conn: IDbConnection) (product: ProductWithOptions) =
    let sql = """
        INSERT INTO Products (Name, Price, Description, DiscountPercent, ExpiryDate)
        VALUES (@Name, @Price, @Description, @DiscountPercent, @ExpiryDate);
        SELECT last_insert_rowid();
    """
    let parameters = DynamicParameters()
    parameters.Add("Name", product.Name)
    parameters.Add("Price", product.Price)
    parameters.Add("Description",
        match product.Description with
        | Some d -> box d
        | None -> box DBNull.Value)
    parameters.Add("DiscountPercent",
        match product.DiscountPercent with
        | Some d -> box d
        | None -> box DBNull.Value)
    parameters.Add("ExpiryDate",
        match product.ExpiryDate with
        | Some d -> box d
        | None -> box DBNull.Value)

    conn.QuerySingle<int>(sql, parameters)
```

---

## สรุป (Summary)

Dapper เป็นเครื่องมือที่ยอดเยี่ยมสำหรับ F# developer ที่ต้องการ:

1. **ประสิทธิภาพสูง**: ใกล้เคียงกับ raw ADO.NET
2. **ควบคุม SQL**: เขียน SQL เองได้เต็มที่
3. **ง่ายต่อการเรียนรู้**: API ไม่ซับซ้อน
4. **F# friendly**: ทำงานร่วมกับ F# records และ Option types ได้ดี

ข้อควรระวัง:
- ต้องลงทะเบียน TypeHandlers สำหรับ F# specific types
- Option types ต้องการ handler พิเศษ
- Discriminated Unions ต้องแปลงเป็น string/int ก่อน store

```fsharp
// Quick reference
// Query multiple rows: conn.Query<T>(sql, parameters)
// Query single row: conn.QuerySingle<T>(sql) / QueryFirstOrDefault<T>(sql)
// Execute DML: conn.Execute(sql, parameters)
// Multi-result: conn.QueryMultiple(sql)
// Async: conn.QueryAsync<T>(sql) / ExecuteAsync(sql)
// Transactions: use transaction = conn.BeginTransaction()
// Bulk: conn.Execute(sql, IEnumerable<parameters>)
```
