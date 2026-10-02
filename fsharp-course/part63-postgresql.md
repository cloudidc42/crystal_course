# Part 63 - PostgreSQL กับ F#

## บทนำ (Introduction)

PostgreSQL เป็นระบบฐานข้อมูลเชิงสัมพันธ์แบบ open-source ที่ทรงพลังที่สุดตัวหนึ่ง รองรับฟีเจอร์ขั้นสูงมากมาย เช่น JSON/JSONB, Arrays, Full-text search, Geographic data (PostGIS), และอื่นๆ

ใน F# เราสามารถใช้ Npgsql ซึ่งเป็น .NET driver อย่างเป็นทางการของ PostgreSQL

---

## 1. การติดตั้ง (Installation)

```xml
<!-- fsharp-postgres.fsproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <!-- Core Npgsql driver -->
    <PackageReference Include="Npgsql" Version="8.0.0" />
    <!-- Dapper integration -->
    <PackageReference Include="Dapper" Version="2.1.28" />
    <!-- EF Core integration -->
    <PackageReference Include="Npgsql.EntityFrameworkCore.PostgreSQL" Version="8.0.0" />
    <!-- JSON support -->
    <PackageReference Include="Npgsql.Json.NET" Version="8.0.0" />
    <!-- NodaTime support -->
    <PackageReference Include="Npgsql.NodaTime" Version="8.0.0" />
    <!-- JSON serialization -->
    <PackageReference Include="Newtonsoft.Json" Version="13.0.3" />
    <PackageReference Include="System.Text.Json" Version="8.0.0" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Connection.fs" />
    <Compile Include="Models.fs" />
    <Compile Include="BasicQueries.fs" />
    <Compile Include="JsonOperations.fs" />
    <Compile Include="ArrayOperations.fs" />
    <Compile Include="AsyncOperations.fs" />
    <Compile Include="BulkCopy.fs" />
    <Compile Include="ListenNotify.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

---

## 2. Connection (การเชื่อมต่อ)

```fsharp
// Connection.fs
module Connection

open System
open Npgsql

// ========================================
// 2.1 Connection String formats
// ========================================

// Format 1: Key-value pairs
let connectionString1 =
    "Host=localhost;Port=5432;Database=mydb;Username=postgres;Password=secret"

// Format 2: URI format
let connectionString2 =
    "postgresql://postgres:secret@localhost:5432/mydb"

// Format 3: NpgsqlConnectionStringBuilder
let buildConnectionString (host: string) (port: int) (database: string) (username: string) (password: string) =
    let builder = NpgsqlConnectionStringBuilder()
    builder.Host <- host
    builder.Port <- port
    builder.Database <- database
    builder.Username <- username
    builder.Password <- password
    builder.MaxPoolSize <- 100
    builder.MinPoolSize <- 5
    builder.ConnectionIdleLifetime <- 300
    builder.CommandTimeout <- 30
    builder.ConnectionString

// Format 4: สำหรับ SSL connection
let buildSecureConnectionString host port database username password =
    let builder = NpgsqlConnectionStringBuilder()
    builder.Host <- host
    builder.Port <- port
    builder.Database <- database
    builder.Username <- username
    builder.Password <- password
    builder.SslMode <- SslMode.Require
    builder.TrustServerCertificate <- true
    builder.ConnectionString

// ========================================
// 2.2 Connection pooling
// ========================================

// NpgsqlDataSource เป็น recommended way ใน Npgsql 7+
let createDataSource connectionString =
    NpgsqlDataSource.Create(connectionString)

// DataSource พร้อม options
let createDataSourceWithOptions connectionString =
    let builder = NpgsqlDataSourceBuilder(connectionString)
    // Enable JSON support
    builder.UseNewtonsoftJson() |> ignore
    // Enable geometric types
    // builder.UseNetTopologySuite() |> ignore
    builder.Build()

// สร้าง connection จาก DataSource
let openConnection (dataSource: NpgsqlDataSource) =
    dataSource.OpenConnection()

let openConnectionAsync (dataSource: NpgsqlDataSource) =
    dataSource.OpenConnectionAsync()

// ========================================
// 2.3 Connection lifecycle
// ========================================

let executeWithConnection (connectionString: string) (action: NpgsqlConnection -> 'T) =
    use conn = new NpgsqlConnection(connectionString)
    conn.Open()
    action conn

let executeWithConnectionAsync (connectionString: string) (action: NpgsqlConnection -> System.Threading.Tasks.Task<'T>) =
    task {
        use conn = new NpgsqlConnection(connectionString)
        do! conn.OpenAsync()
        return! action conn
    }

// ========================================
// 2.4 Connection health check
// ========================================

let testConnection (connectionString: string) =
    try
        use conn = new NpgsqlConnection(connectionString)
        conn.Open()
        use cmd = conn.CreateCommand()
        cmd.CommandText <- "SELECT 1"
        let result = cmd.ExecuteScalar() :?> int
        result = 1
    with ex ->
        printfn "Connection failed: %s" ex.Message
        false
```

---

## 3. Basic Queries (การ Query พื้นฐาน)

```fsharp
// BasicQueries.fs
module BasicQueries

open System
open Npgsql
open Dapper

// ========================================
// 3.1 Models
// ========================================

type Customer = {
    Id: int
    Name: string
    Email: string
    CreatedAt: DateTime
    IsActive: bool
}

type Product = {
    Id: int
    Name: string
    Price: decimal
    Description: string option
    CategoryId: int
    Tags: string[]  // PostgreSQL array
    Metadata: string  // JSON stored as text
}

// ========================================
// 3.2 Setup tables
// ========================================

let createTables (conn: NpgsqlConnection) =
    let sql = """
        CREATE TABLE IF NOT EXISTS customers (
            id SERIAL PRIMARY KEY,
            name VARCHAR(200) NOT NULL,
            email VARCHAR(300) UNIQUE NOT NULL,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
            is_active BOOLEAN NOT NULL DEFAULT TRUE
        );

        CREATE TABLE IF NOT EXISTS categories (
            id SERIAL PRIMARY KEY,
            name VARCHAR(200) NOT NULL,
            description TEXT
        );

        CREATE TABLE IF NOT EXISTS products (
            id SERIAL PRIMARY KEY,
            name VARCHAR(300) NOT NULL,
            price NUMERIC(10,2) NOT NULL,
            description TEXT,
            category_id INTEGER REFERENCES categories(id),
            tags TEXT[] NOT NULL DEFAULT '{}',
            metadata JSONB,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
            updated_at TIMESTAMPTZ,
            is_active BOOLEAN NOT NULL DEFAULT TRUE
        );

        CREATE TABLE IF NOT EXISTS orders (
            id SERIAL PRIMARY KEY,
            customer_id INTEGER NOT NULL REFERENCES customers(id),
            total_amount NUMERIC(10,2) NOT NULL,
            status VARCHAR(50) NOT NULL DEFAULT 'pending',
            shipping_address JSONB,
            created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
        );

        CREATE TABLE IF NOT EXISTS order_items (
            id SERIAL PRIMARY KEY,
            order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
            product_id INTEGER NOT NULL REFERENCES products(id),
            quantity INTEGER NOT NULL,
            unit_price NUMERIC(10,2) NOT NULL
        );

        -- Index สำหรับ performance
        CREATE INDEX IF NOT EXISTS idx_products_category ON products(category_id);
        CREATE INDEX IF NOT EXISTS idx_orders_customer ON orders(customer_id);
        CREATE INDEX IF NOT EXISTS idx_products_tags ON products USING gin(tags);
        CREATE INDEX IF NOT EXISTS idx_products_metadata ON products USING gin(metadata);
    """
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.ExecuteNonQuery() |> ignore
    printfn "Tables created"

// ========================================
// 3.3 Basic CRUD
// ========================================

/// Insert customer และ return ID ด้วย RETURNING clause
let insertCustomer (conn: NpgsqlConnection) (name: string) (email: string) =
    let sql = """
        INSERT INTO customers (name, email)
        VALUES (@name, @email)
        RETURNING id
    """
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("name", name) |> ignore
    cmd.Parameters.AddWithValue("email", email) |> ignore
    cmd.ExecuteScalar() :?> int

/// Insert ด้วย Dapper
let insertCustomerDapper (conn: NpgsqlConnection) (name: string) (email: string) =
    let sql = """
        INSERT INTO customers (name, email)
        VALUES (@Name, @Email)
        RETURNING id
    """
    conn.QuerySingle<int>(sql, {| Name = name; Email = email |})

/// Query ทั้งหมด
let getAllCustomers (conn: NpgsqlConnection) =
    let sql = "SELECT id, name, email, created_at, is_active FROM customers ORDER BY id"
    conn.Query<Customer>(sql) |> Seq.toList

/// Query ด้วย filter
let getActiveCustomers (conn: NpgsqlConnection) =
    let sql = "SELECT id, name, email, created_at, is_active FROM customers WHERE is_active = TRUE ORDER BY name"
    conn.Query<Customer>(sql) |> Seq.toList

/// Query ด้วย parameter
let getCustomerById (conn: NpgsqlConnection) (id: int) =
    let sql = "SELECT id, name, email, created_at, is_active FROM customers WHERE id = @Id"
    conn.QueryFirstOrDefault<Customer>(sql, {| Id = id |}) |> Option.ofObj

// ========================================
// 3.4 Prepared Statements
// ========================================

/// Prepared statement ช่วยเพิ่ม performance สำหรับ queries ที่รันบ่อยๆ
let preparedInsert (conn: NpgsqlConnection) =
    let cmd = new NpgsqlCommand("INSERT INTO customers (name, email) VALUES ($1, $2) RETURNING id", conn)
    cmd.Parameters.Add(NpgsqlParameter()) |> ignore
    cmd.Parameters.Add(NpgsqlParameter()) |> ignore
    cmd.Prepare()  // เตรียม execution plan ไว้ล่วงหน้า

    // ใช้ prepared statement หลายครั้ง
    let insertMany (names: (string * string) list) =
        [ for (name, email) in names do
            cmd.Parameters.[0].Value <- name
            cmd.Parameters.[1].Value <- email
            yield cmd.ExecuteScalar() :?> int ]

    insertMany

// ========================================
// 3.5 Upsert (INSERT ON CONFLICT)
// ========================================

let upsertCustomer (conn: NpgsqlConnection) (name: string) (email: string) =
    let sql = """
        INSERT INTO customers (name, email)
        VALUES (@Name, @Email)
        ON CONFLICT (email)
        DO UPDATE SET
            name = EXCLUDED.name,
            is_active = TRUE
        RETURNING id, (xmax = 0) AS is_insert
    """
    conn.QuerySingle(sql, {| Name = name; Email = email |})

// ========================================
// 3.6 Batch operations
// ========================================

let insertManyCustomers (conn: NpgsqlConnection) (customers: (string * string) list) =
    use transaction = conn.BeginTransaction()
    try
        let sql = "INSERT INTO customers (name, email) VALUES (@Name, @Email)"
        let parameters =
            customers |> List.map (fun (name, email) -> {| Name = name; Email = email |})
        let rowsAffected = conn.Execute(sql, parameters, transaction)
        transaction.Commit()
        rowsAffected
    with ex ->
        transaction.Rollback()
        raise ex
```

---

## 4. JSON Operations (การทำงานกับ JSON)

```fsharp
// JsonOperations.fs
module JsonOperations

open System
open System.Text.Json
open Npgsql
open Dapper

// ========================================
// 4.1 JSONB operations
// ========================================

type Address = {
    Street: string
    City: string
    Country: string
    PostalCode: string
}

type ProductMetadata = {
    Weight: float
    Dimensions: {| Width: float; Height: float; Depth: float |}
    Brand: string
    Tags: string list
}

/// Insert ด้วย JSONB
let insertProductWithMetadata (conn: NpgsqlConnection) (name: string) (price: decimal) (metadata: ProductMetadata) =
    let sql = """
        INSERT INTO products (name, price, metadata, category_id)
        VALUES (@Name, @Price, @Metadata::jsonb, 1)
        RETURNING id
    """
    let metadataJson = JsonSerializer.Serialize(metadata)
    conn.QuerySingle<int>(sql, {| Name = name; Price = price; Metadata = metadataJson |})

/// Query ด้วย JSONB operators
let getProductsByBrand (conn: NpgsqlConnection) (brand: string) =
    let sql = """
        SELECT id, name, price, metadata::text as metadata_text
        FROM products
        WHERE metadata->>'Brand' = @Brand
        AND is_active = TRUE
    """
    conn.Query(sql, {| Brand = brand |}) |> Seq.toList

/// Query nested JSON
let getHeavyProducts (conn: NpgsqlConnection) (minWeight: float) =
    let sql = """
        SELECT id, name, price,
               (metadata->>'Weight')::float as weight
        FROM products
        WHERE (metadata->>'Weight')::float > @MinWeight
        ORDER BY weight DESC
    """
    conn.Query(sql, {| MinWeight = minWeight |}) |> Seq.toList

/// Update JSONB field
let updateProductMetadataField (conn: NpgsqlConnection) (productId: int) (fieldName: string) (value: string) =
    let sql = """
        UPDATE products
        SET metadata = jsonb_set(metadata, @Path, @Value::jsonb)
        WHERE id = @Id
    """
    let path = $"{{{fieldName}}}"
    conn.Execute(sql, {|
        Id = productId
        Path = path
        Value = $"\"{value}\""
    |})

/// Aggregate JSON data
let getMetadataSummary (conn: NpgsqlConnection) =
    let sql = """
        SELECT
            metadata->>'Brand' as brand,
            COUNT(*) as product_count,
            AVG((metadata->>'Weight')::float) as avg_weight,
            MIN(price) as min_price,
            MAX(price) as max_price
        FROM products
        WHERE metadata IS NOT NULL
        GROUP BY metadata->>'Brand'
        ORDER BY product_count DESC
    """
    conn.Query(sql) |> Seq.toList

// ========================================
// 4.2 JSON aggregation
// ========================================

/// สร้าง JSON array จาก rows
let getOrdersAsJson (conn: NpgsqlConnection) (customerId: int) =
    let sql = """
        SELECT json_agg(
            json_build_object(
                'id', o.id,
                'total', o.total_amount,
                'status', o.status,
                'items', (
                    SELECT json_agg(
                        json_build_object(
                            'product', p.name,
                            'qty', oi.quantity,
                            'price', oi.unit_price
                        )
                    )
                    FROM order_items oi
                    JOIN products p ON oi.product_id = p.id
                    WHERE oi.order_id = o.id
                )
            )
            ORDER BY o.created_at DESC
        ) as orders_json
        FROM orders o
        WHERE o.customer_id = @CustomerId
    """
    conn.QuerySingle<string>(sql, {| CustomerId = customerId |})

// ========================================
// 4.3 JSONB with Npgsql native mapping
// ========================================

let insertOrderWithAddress (conn: NpgsqlConnection) (customerId: int) (address: Address) (total: decimal) =
    let sql = """
        INSERT INTO orders (customer_id, total_amount, shipping_address)
        VALUES (@CustomerId, @TotalAmount, @Address::jsonb)
        RETURNING id
    """
    let addressJson = JsonSerializer.Serialize(address)
    conn.QuerySingle<int>(sql, {|
        CustomerId = customerId
        TotalAmount = total
        Address = addressJson
    |})

let getOrderAddress (conn: NpgsqlConnection) (orderId: int) =
    let sql = "SELECT shipping_address::text FROM orders WHERE id = @Id"
    let json = conn.QueryFirstOrDefault<string>(sql, {| Id = orderId |})
    if json = null then None
    else
        try
            Some (JsonSerializer.Deserialize<Address>(json))
        with _ ->
            None
```

---

## 5. Arrays in PostgreSQL

```fsharp
// ArrayOperations.fs
module ArrayOperations

open System
open Npgsql
open Dapper

// ========================================
// 5.1 Working with PostgreSQL arrays
// ========================================

/// Insert product ด้วย tags array
let insertProductWithTags (conn: NpgsqlConnection) (name: string) (price: decimal) (tags: string[]) =
    let sql = """
        INSERT INTO products (name, price, tags, category_id)
        VALUES (@Name, @Price, @Tags, 1)
        RETURNING id
    """
    // Npgsql แปลง F# array เป็น PostgreSQL array อัตโนมัติ
    let cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("Name", name) |> ignore
    cmd.Parameters.AddWithValue("Price", price) |> ignore
    cmd.Parameters.AddWithValue("Tags", tags) |> ignore
    cmd.ExecuteScalar() :?> int

/// ค้นหาด้วย array contains
let getProductsByTag (conn: NpgsqlConnection) (tag: string) =
    let sql = """
        SELECT id, name, price, tags
        FROM products
        WHERE @Tag = ANY(tags)
        ORDER BY name
    """
    conn.Query(sql, {| Tag = tag |}) |> Seq.toList

/// ค้นหาด้วย array overlap (@&)
let getProductsByAnyTag (conn: NpgsqlConnection) (tags: string[]) =
    let sql = """
        SELECT id, name, price, tags
        FROM products
        WHERE tags && @Tags
        ORDER BY name
    """
    let cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("Tags", tags) |> ignore

    use reader = cmd.ExecuteReader()
    [ while reader.Read() do
        yield {|
            Id = reader.GetInt32(0)
            Name = reader.GetString(1)
            Price = reader.GetDecimal(2)
            Tags = reader.GetFieldValue<string[]>(3)
        |}
    ]

/// ค้นหาด้วย array contains all (@>)
let getProductsWithAllTags (conn: NpgsqlConnection) (requiredTags: string[]) =
    let sql = """
        SELECT id, name, tags
        FROM products
        WHERE tags @> @RequiredTags
    """
    let cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("RequiredTags", requiredTags) |> ignore

    use reader = cmd.ExecuteReader()
    [ while reader.Read() do
        yield {|
            Id = reader.GetInt32(0)
            Name = reader.GetString(1)
            Tags = reader.GetFieldValue<string[]>(2)
        |}
    ]

/// Update array (append element)
let addTagToProduct (conn: NpgsqlConnection) (productId: int) (newTag: string) =
    let sql = """
        UPDATE products
        SET tags = array_append(tags, @Tag)
        WHERE id = @Id
        AND NOT (@Tag = ANY(tags))
    """
    conn.Execute(sql, {| Id = productId; Tag = newTag |})

/// Remove from array
let removeTagFromProduct (conn: NpgsqlConnection) (productId: int) (tag: string) =
    let sql = """
        UPDATE products
        SET tags = array_remove(tags, @Tag)
        WHERE id = @Id
    """
    conn.Execute(sql, {| Id = productId; Tag = tag |})

// ========================================
// 5.2 Integer arrays
// ========================================

/// Get users ที่มี IDs ใน list
let getUsersByIds (conn: NpgsqlConnection) (ids: int[]) =
    let sql = "SELECT id, name, email FROM customers WHERE id = ANY(@Ids)"
    let cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("Ids", ids) |> ignore

    use reader = cmd.ExecuteReader()
    [ while reader.Read() do
        yield {|
            Id = reader.GetInt32(0)
            Name = reader.GetString(1)
            Email = reader.GetString(2)
        |}
    ]

// ========================================
// 5.3 Array functions
// ========================================

let getTagStatistics (conn: NpgsqlConnection) =
    let sql = """
        SELECT
            unnest(tags) as tag,
            COUNT(*) as product_count
        FROM products
        WHERE is_active = TRUE
        GROUP BY tag
        ORDER BY product_count DESC, tag
        LIMIT 20
    """
    conn.Query(sql) |> Seq.toList
```

---

## 6. Transactions (การจัดการ Transactions)

```fsharp
// Transactions.fs
module TransactionExamples

open System
open Npgsql
open Dapper

// ========================================
// 6.1 Basic transaction
// ========================================

let createOrderWithItems (conn: NpgsqlConnection) (customerId: int) (items: (int * int) list) =
    // items = (productId, quantity) list
    use transaction = conn.BeginTransaction()
    try
        // คำนวณ total
        let productIds = items |> List.map fst
        let pricesSql = "SELECT id, price FROM products WHERE id = ANY(@Ids)"
        let cmd = new NpgsqlCommand(pricesSql, conn, transaction)
        cmd.Parameters.AddWithValue("Ids", productIds |> List.toArray) |> ignore
        use reader = cmd.ExecuteReader()
        let priceMap =
            [ while reader.Read() do
                yield reader.GetInt32(0), reader.GetDecimal(1) ]
            |> Map.ofList
        reader.Close()

        let total =
            items
            |> List.sumBy (fun (productId, qty) ->
                match Map.tryFind productId priceMap with
                | Some price -> price * decimal qty
                | None -> failwith $"Product {productId} not found"
            )

        // สร้าง order
        let orderSql = """
            INSERT INTO orders (customer_id, total_amount, status)
            VALUES (@CustomerId, @TotalAmount, 'pending')
            RETURNING id
        """
        let orderId = conn.QuerySingle<int>(orderSql, {|
            CustomerId = customerId
            TotalAmount = total
        |}, transaction)

        // สร้าง order items
        for (productId, quantity) in items do
            let unitPrice = Map.find productId priceMap
            let itemSql = """
                INSERT INTO order_items (order_id, product_id, quantity, unit_price)
                VALUES (@OrderId, @ProductId, @Quantity, @UnitPrice)
            """
            conn.Execute(itemSql, {|
                OrderId = orderId
                ProductId = productId
                Quantity = quantity
                UnitPrice = unitPrice
            |}, transaction) |> ignore

        transaction.Commit()
        Ok orderId
    with ex ->
        transaction.Rollback()
        Error ex.Message

// ========================================
// 6.2 Async transaction
// ========================================

let createOrderAsync (conn: NpgsqlConnection) (customerId: int) (total: decimal) =
    task {
        use! transaction = conn.BeginTransactionAsync()
        try
            let sql = """
                INSERT INTO orders (customer_id, total_amount)
                VALUES (@CustomerId, @TotalAmount)
                RETURNING id
            """
            let! orderId = conn.QuerySingleAsync<int>(sql, {|
                CustomerId = customerId
                TotalAmount = total
            |}, transaction)

            do! transaction.CommitAsync()
            return Ok orderId
        with ex ->
            do! transaction.RollbackAsync()
            return Error ex.Message
    }

// ========================================
// 6.3 Advisory locks
// ========================================

/// PostgreSQL advisory lock สำหรับ distributed locking
let withAdvisoryLock (conn: NpgsqlConnection) (lockId: int64) (action: unit -> 'T) =
    let acquired =
        conn.QuerySingle<bool>("SELECT pg_try_advisory_lock(@LockId)", {| LockId = lockId |})
    if not acquired then
        failwith "Could not acquire lock"
    try
        let result = action()
        conn.Execute("SELECT pg_advisory_unlock(@LockId)", {| LockId = lockId |}) |> ignore
        result
    with ex ->
        conn.Execute("SELECT pg_advisory_unlock(@LockId)", {| LockId = lockId |}) |> ignore
        raise ex
```

---

## 7. Async Operations

```fsharp
// AsyncOperations.fs
module AsyncOperations

open System
open Npgsql
open Dapper

// ========================================
// 7.1 Async queries
// ========================================

let getAllCustomersAsync (conn: NpgsqlConnection) =
    task {
        let sql = "SELECT id, name, email, created_at, is_active FROM customers ORDER BY name"
        let! result = conn.QueryAsync(sql)
        return result |> Seq.toList
    }

let getCustomerByIdAsync (conn: NpgsqlConnection) (id: int) =
    task {
        let sql = "SELECT id, name, email, created_at, is_active FROM customers WHERE id = @Id"
        let! result = conn.QueryFirstOrDefaultAsync(sql, {| Id = id |})
        return result |> Option.ofObj
    }

let createCustomerAsync (conn: NpgsqlConnection) (name: string) (email: string) =
    task {
        let sql = """
            INSERT INTO customers (name, email)
            VALUES (@Name, @Email)
            RETURNING id
        """
        return! conn.QuerySingleAsync<int>(sql, {| Name = name; Email = email |})
    }

// ========================================
// 7.2 Async with cancellation
// ========================================

let getLargeDatasetAsync (conn: NpgsqlConnection) (cancellationToken: System.Threading.CancellationToken) =
    task {
        let sql = "SELECT * FROM products ORDER BY id"
        let! result = conn.QueryAsync(sql, cancellationToken = cancellationToken)
        return result |> Seq.toList
    }

// ========================================
// 7.3 Streaming large results
// ========================================

/// Stream ข้อมูลขนาดใหญ่ (ไม่ load ทั้งหมดใน memory)
let streamLargeResults (conn: NpgsqlConnection) (batchSize: int) (processFunc: obj list -> unit) =
    task {
        let sql = "SELECT id, name, price FROM products ORDER BY id"
        use cmd = new NpgsqlCommand(sql, conn)
        cmd.CommandTimeout <- 300

        use! reader = cmd.ExecuteReaderAsync()
        let mutable batch = []
        let mutable count = 0

        while! reader.ReadAsync() do
            let row = {|
                Id = reader.GetInt32(0)
                Name = reader.GetString(1)
                Price = reader.GetDecimal(2)
            |}
            batch <- (box row) :: batch
            count <- count + 1

            if count % batchSize = 0 then
                processFunc (List.rev batch)
                batch <- []

        if not batch.IsEmpty then
            processFunc (List.rev batch)
    }

// ========================================
// 7.4 Parallel queries
// ========================================

let getMultipleDataSetsAsync (dataSource: NpgsqlDataSource) =
    task {
        // ต้องใช้ connections แยกกันสำหรับ parallel queries
        let customersTask =
            task {
                use! conn = dataSource.OpenConnectionAsync()
                return! conn.QueryAsync("SELECT * FROM customers LIMIT 100")
            }

        let productsTask =
            task {
                use! conn = dataSource.OpenConnectionAsync()
                return! conn.QueryAsync("SELECT * FROM products LIMIT 100")
            }

        let ordersTask =
            task {
                use! conn = dataSource.OpenConnectionAsync()
                return! conn.QueryAsync("SELECT * FROM orders LIMIT 100")
            }

        let! results = System.Threading.Tasks.Task.WhenAll(customersTask, productsTask, ordersTask)

        return {|
            Customers = results.[0] |> Seq.toList
            Products = results.[1] |> Seq.toList
            Orders = results.[2] |> Seq.toList
        |}
    }
```

---

## 8. Bulk Copy (การ Copy ข้อมูลจำนวนมาก)

```fsharp
// BulkCopy.fs
module BulkCopy

open System
open Npgsql

// ========================================
// 8.1 COPY command สำหรับ bulk insert
// ========================================

type CustomerRecord = {
    Name: string
    Email: string
    IsActive: bool
}

/// Bulk insert ด้วย COPY (เร็วมากสำหรับข้อมูลจำนวนมาก)
let bulkInsertCustomers (conn: NpgsqlConnection) (customers: CustomerRecord list) =
    use writer = conn.BeginBinaryImport("""
        COPY customers (name, email, is_active)
        FROM STDIN (FORMAT BINARY)
    """)

    for customer in customers do
        writer.StartRow()
        writer.Write(customer.Name, NpgsqlTypes.NpgsqlDbType.Varchar)
        writer.Write(customer.Email, NpgsqlTypes.NpgsqlDbType.Varchar)
        writer.Write(customer.IsActive, NpgsqlTypes.NpgsqlDbType.Boolean)

    writer.Complete()

type ProductRecord = {
    Name: string
    Price: decimal
    CategoryId: int
    Stock: int
    Tags: string[]
}

/// Bulk insert Products
let bulkInsertProducts (conn: NpgsqlConnection) (products: ProductRecord list) =
    use writer = conn.BeginBinaryImport("""
        COPY products (name, price, category_id, stock, tags)
        FROM STDIN (FORMAT BINARY)
    """)

    for p in products do
        writer.StartRow()
        writer.Write(p.Name, NpgsqlTypes.NpgsqlDbType.Varchar)
        writer.Write(p.Price, NpgsqlTypes.NpgsqlDbType.Numeric)
        writer.Write(p.CategoryId, NpgsqlTypes.NpgsqlDbType.Integer)
        writer.Write(p.Stock, NpgsqlTypes.NpgsqlDbType.Integer)
        writer.Write(p.Tags, NpgsqlTypes.NpgsqlDbType.Array ||| NpgsqlTypes.NpgsqlDbType.Text)

    writer.Complete()

// ========================================
// 8.2 COPY TO (export)
// ========================================

/// Export ข้อมูลเป็น CSV format
let exportCustomersToCSV (conn: NpgsqlConnection) (filePath: string) =
    use reader = conn.BeginTextExport("COPY customers TO STDOUT (FORMAT CSV, HEADER TRUE)")
    use file = System.IO.File.CreateText(filePath)
    let mutable line = reader.ReadLine()
    while line <> null do
        file.WriteLine(line)
        line <- reader.ReadLine()
    printfn "Exported to %s" filePath

/// Export แบบ binary สำหรับ re-import
let exportCustomersBinary (conn: NpgsqlConnection) =
    use reader = conn.BeginBinaryExport("COPY customers TO STDOUT (FORMAT BINARY)")
    let buffer = Array.zeroCreate<byte> 65536
    let mutable bytesRead = 0
    let ms = new System.IO.MemoryStream()
    bytesRead <- reader.Read(buffer, 0, buffer.Length)
    while bytesRead > 0 do
        ms.Write(buffer, 0, bytesRead)
        bytesRead <- reader.Read(buffer, 0, buffer.Length)
    ms.ToArray()

// ========================================
// 8.3 Async bulk copy
// ========================================

let bulkInsertAsync (conn: NpgsqlConnection) (customers: CustomerRecord list) =
    task {
        let writer = conn.BeginBinaryImport("""
            COPY customers (name, email, is_active)
            FROM STDIN (FORMAT BINARY)
        """)

        for customer in customers do
            writer.StartRow()
            writer.Write(customer.Name, NpgsqlTypes.NpgsqlDbType.Varchar)
            writer.Write(customer.Email, NpgsqlTypes.NpgsqlDbType.Varchar)
            writer.Write(customer.IsActive, NpgsqlTypes.NpgsqlDbType.Boolean)

        do! writer.CompleteAsync()
        printfn "Bulk insert complete: %d rows" customers.Length
    }
```

---

## 9. LISTEN/NOTIFY สำหรับ Real-time

```fsharp
// ListenNotify.fs
module ListenNotify

open System
open Npgsql

// ========================================
// 9.1 LISTEN สำหรับรับ notifications
// ========================================

let startListening (connectionString: string) (channel: string) (handler: NpgsqlNotificationEventArgs -> unit) =
    let conn = new NpgsqlConnection(connectionString)
    conn.Open()

    // Subscribe to channel
    conn.Notification.Add(fun args ->
        if args.Channel = channel then
            handler args
    )

    use cmd = new NpgsqlCommand($"LISTEN {channel}", conn)
    cmd.ExecuteNonQuery() |> ignore

    printfn "Listening on channel: %s" channel
    conn  // Return connection (caller owns it)

// ========================================
// 9.2 NOTIFY สำหรับส่ง notification
// ========================================

let sendNotification (conn: NpgsqlConnection) (channel: string) (payload: string) =
    let sql = "SELECT pg_notify(@Channel, @Payload)"
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("Channel", channel) |> ignore
    cmd.Parameters.AddWithValue("Payload", payload) |> ignore
    cmd.ExecuteNonQuery() |> ignore

// ========================================
// 9.3 Practical example: Real-time order updates
// ========================================

type OrderNotification = {
    OrderId: int
    Status: string
    UpdatedAt: DateTime
}

/// Trigger ที่ส่ง notification เมื่อ order status เปลี่ยน
let createOrderNotificationTrigger (conn: NpgsqlConnection) =
    let sql = """
        CREATE OR REPLACE FUNCTION notify_order_status_change()
        RETURNS TRIGGER AS $$
        BEGIN
            PERFORM pg_notify(
                'order_updates',
                json_build_object(
                    'orderId', NEW.id,
                    'status', NEW.status,
                    'updatedAt', NOW()
                )::text
            );
            RETURN NEW;
        END;
        $$ LANGUAGE plpgsql;

        DROP TRIGGER IF EXISTS order_status_change ON orders;

        CREATE TRIGGER order_status_change
        AFTER UPDATE OF status ON orders
        FOR EACH ROW
        EXECUTE FUNCTION notify_order_status_change();
    """
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.ExecuteNonQuery() |> ignore
    printfn "Order notification trigger created"

/// Listen สำหรับ order updates
let listenForOrderUpdates (connectionString: string) (onUpdate: OrderNotification -> unit) =
    task {
        use conn = new NpgsqlConnection(connectionString)
        do! conn.OpenAsync()

        conn.Notification.Add(fun args ->
            try
                let notification = System.Text.Json.JsonSerializer.Deserialize<OrderNotification>(args.Payload)
                onUpdate notification
            with ex ->
                printfn "Error parsing notification: %s" ex.Message
        )

        use cmd = new NpgsqlCommand("LISTEN order_updates", conn)
        do! cmd.ExecuteNonQueryAsync() |> Async.AwaitTask |> Async.Ignore

        printfn "Listening for order updates..."

        // Keep connection alive
        while true do
            do! conn.WaitAsync() |> Async.AwaitTask |> Async.Ignore
    }
```

---

## 10. PostgreSQL-specific Types (ประเภทข้อมูลพิเศษ)

```fsharp
// PostgresTypes.fs
module PostgresTypes

open System
open Npgsql
open NpgsqlTypes

// ========================================
// 10.1 Date/Time types
// ========================================

let workWithDateTypes (conn: NpgsqlConnection) =
    let sql = """
        SELECT
            NOW()::TIMESTAMPTZ as now_tz,
            NOW()::TIMESTAMP as now_no_tz,
            CURRENT_DATE as today,
            CURRENT_TIME as time_now,
            INTERVAL '3 hours 30 minutes' as interval_val,
            TSTZRANGE(NOW(), NOW() + INTERVAL '1 day') as date_range
    """
    use cmd = new NpgsqlCommand(sql, conn)
    use reader = cmd.ExecuteReader()
    if reader.Read() then
        printfn "Now (with TZ): %A" (reader.GetFieldValue<DateTimeOffset>(0))
        printfn "Now (no TZ): %A" (reader.GetFieldValue<DateTime>(1))
        printfn "Today: %A" (reader.GetFieldValue<DateOnly>(2))
        printfn "Time: %A" (reader.GetFieldValue<TimeOnly>(3))

// ========================================
// 10.2 Network types
// ========================================

let workWithNetworkTypes (conn: NpgsqlConnection) =
    let sql = """
        SELECT
            '192.168.1.0/24'::CIDR as network,
            '192.168.1.100'::INET as host,
            '08:00:2b:01:02:03'::MACADDR as mac
    """
    use cmd = new NpgsqlCommand(sql, conn)
    use reader = cmd.ExecuteReader()
    if reader.Read() then
        printfn "Network: %A" (reader[0])
        printfn "Host: %A" (reader[1])
        printfn "MAC: %A" (reader[2])

// ========================================
// 10.3 UUID
// ========================================

let workWithUuids (conn: NpgsqlConnection) =
    let sql = """
        SELECT
            gen_random_uuid() as uuid1,
            '550e8400-e29b-41d4-a716-446655440000'::UUID as fixed_uuid
    """
    use cmd = new NpgsqlCommand(sql, conn)
    use reader = cmd.ExecuteReader()
    if reader.Read() then
        printfn "Random UUID: %A" (reader.GetGuid(0))
        printfn "Fixed UUID: %A" (reader.GetGuid(1))

// ========================================
// 10.4 Range types
// ========================================

let workWithRangeTypes (conn: NpgsqlConnection) =
    let sql = """
        CREATE TEMP TABLE IF NOT EXISTS price_ranges (
            product_id INT,
            valid_price NUMRANGE
        );

        INSERT INTO price_ranges VALUES
            (1, '[100, 500]'),
            (2, '[200, 800)'),
            (3, '(0, 1000]');

        SELECT * FROM price_ranges
        WHERE valid_price @> 250::NUMERIC;
    """
    use cmd = new NpgsqlCommand(sql, conn)
    use reader = cmd.ExecuteReader()
    while reader.Read() do
        printfn "Product %d: range %A" (reader.GetInt32(0)) (reader[1])

// ========================================
// 10.5 Full-text search
// ========================================

let setupFullTextSearch (conn: NpgsqlConnection) =
    let sql = """
        ALTER TABLE products
        ADD COLUMN IF NOT EXISTS search_vector TSVECTOR
            GENERATED ALWAYS AS (
                to_tsvector('english',
                    coalesce(name, '') || ' ' ||
                    coalesce(description, '')
                )
            ) STORED;

        CREATE INDEX IF NOT EXISTS idx_products_fts
        ON products USING GIN(search_vector);
    """
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.ExecuteNonQuery() |> ignore

let searchProducts (conn: NpgsqlConnection) (query: string) =
    let sql = """
        SELECT id, name, price,
               ts_rank(search_vector, plainto_tsquery('english', @Query)) as rank
        FROM products
        WHERE search_vector @@ plainto_tsquery('english', @Query)
        ORDER BY rank DESC
        LIMIT 20
    """
    use cmd = new NpgsqlCommand(sql, conn)
    cmd.Parameters.AddWithValue("Query", query) |> ignore

    use reader = cmd.ExecuteReader()
    [ while reader.Read() do
        yield {|
            Id = reader.GetInt32(0)
            Name = reader.GetString(1)
            Price = reader.GetDecimal(2)
            Rank = reader.GetFloat(3)
        |}
    ]
```

---

## 11. Complete Example (ตัวอย่างครบวงจร)

```fsharp
// Program.fs
module Program

open System
open Npgsql
open Dapper

[<EntryPoint>]
let main _ =
    // สำหรับ demo นี้ ใช้ SQLite แทน PostgreSQL เพื่อไม่ต้องติดตั้ง server
    // ในงานจริงให้ใช้ NpgsqlConnection แทน
    printfn "=== PostgreSQL F# Demo ==="
    printfn "Note: Using demonstration mode with mock data"
    printfn ""

    // ตัวอย่างการใช้งาน NpgsqlConnectionStringBuilder
    let connStrBuilder = NpgsqlConnectionStringBuilder()
    connStrBuilder.Host <- "localhost"
    connStrBuilder.Port <- 5432
    connStrBuilder.Database <- "shopdb"
    connStrBuilder.Username <- "postgres"
    connStrBuilder.Password <- "secret"
    connStrBuilder.MaxPoolSize <- 50
    connStrBuilder.MinPoolSize <- 5

    printfn "Connection string: %s" (connStrBuilder.ConnectionString.Replace("Password=secret", "Password=***"))

    // ตัวอย่าง SQL queries ที่ใช้ PostgreSQL features
    printfn "\n--- PostgreSQL-specific SQL examples ---"

    let exampleQueries = [
        "JSONB query", "SELECT * FROM products WHERE metadata->>'brand' = 'Apple'"
        "Array contains", "SELECT * FROM products WHERE 'sale' = ANY(tags)"
        "Array overlap", "SELECT * FROM products WHERE tags && ARRAY['sale', 'new']"
        "Full-text search", "SELECT * FROM products WHERE search_vector @@ plainto_tsquery('laptop')"
        "Window function", "SELECT *, ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY price DESC) FROM products"
        "CTE", "WITH top_products AS (SELECT * FROM products ORDER BY price DESC LIMIT 5) SELECT * FROM top_products"
        "UPSERT", "INSERT INTO customers (email, name) VALUES ($1, $2) ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name"
        "RETURNING", "UPDATE products SET price = price * 1.1 WHERE category_id = 1 RETURNING id, name, price"
        "Range type", "SELECT * FROM price_ranges WHERE valid_price @> 250.00"
        "Lateral join", "SELECT c.name, o.* FROM customers c, LATERAL (SELECT * FROM orders WHERE customer_id = c.id LIMIT 3) o"
        "Generate series", "SELECT generate_series(1, 10) as n"
        "LISTEN/NOTIFY", "LISTEN order_updates; NOTIFY order_updates, '{\"status\": \"shipped\"}'"
    ]

    for (name, sql) in exampleQueries do
        printfn "  [%s]" name
        printfn "    %s" sql
        printfn ""

    // Npgsql type mapping examples
    printfn "--- Npgsql Type Mappings ---"
    let typeMappings = [
        ("INTEGER", "int")
        ("BIGINT", "int64")
        ("DECIMAL/NUMERIC", "decimal")
        ("VARCHAR/TEXT", "string")
        ("BOOLEAN", "bool")
        ("TIMESTAMP", "DateTime")
        ("TIMESTAMPTZ", "DateTimeOffset")
        ("DATE", "DateOnly")
        ("TIME", "TimeOnly")
        ("UUID", "Guid")
        ("JSONB/JSON", "string (or custom type with NpgsqlJsonHandler)")
        ("TEXT[]", "string[]")
        ("INTEGER[]", "int[]")
        ("BYTEA", "byte[]")
        ("INTERVAL", "TimeSpan")
        ("INET", "NpgsqlInet")
        ("CIDR", "NpgsqlCidr")
        ("MACADDR", "PhysicalAddress")
    ]

    for (pgType, fsType) in typeMappings do
        printfn "  PostgreSQL %-20s -> F# %s" pgType fsType

    // Connection pooling best practices
    printfn "\n--- Connection Pooling Best Practices ---"
    printfn "1. ใช้ NpgsqlDataSource (Npgsql 7+) แทน NpgsqlConnection โดยตรง"
    printfn "2. ตั้งค่า MaxPoolSize ตาม workload"
    printfn "3. ใช้ 'use' keyword เพื่อ return connection กลับ pool"
    printfn "4. อย่า hold connection นานเกินจำเป็น"
    printfn "5. ใช้ connection timeout ที่เหมาะสม"

    // Performance tips
    printfn "\n--- Performance Tips ---"
    printfn "1. ใช้ Prepared Statements สำหรับ queries ที่รันบ่อย"
    printfn "2. ใช้ COPY สำหรับ bulk insert (เร็วกว่า INSERT 10-100x)"
    printfn "3. ใช้ Indexes ที่เหมาะสม (B-tree, GIN สำหรับ arrays/JSONB)"
    printfn "4. ใช้ EXPLAIN ANALYZE เพื่อตรวจสอบ query plan"
    printfn "5. ใช้ Connection Pooling (pgBouncer, PgCat)"
    printfn "6. Partition tables สำหรับข้อมูลขนาดใหญ่"

    printfn "\n=== Demo Complete ==="
    0
```

---

## สรุป (Summary)

PostgreSQL กับ F# ผ่าน Npgsql มีข้อดีมากมาย:

1. **Type safety**: Npgsql แมป .NET types กับ PostgreSQL types ได้อย่างถูกต้อง
2. **Feature-rich**: รองรับ JSONB, Arrays, Full-text search, Geographic data
3. **Performance**: Connection pooling, Prepared statements, Bulk copy
4. **Real-time**: LISTEN/NOTIFY สำหรับ push notifications
5. **Dapper integration**: ใช้ Dapper ร่วมกับ Npgsql ได้ทันที

```fsharp
// Key patterns:
// Connection: NpgsqlDataSource.Create(connStr)
// Query: conn.QueryAsync<T>(sql, params)
// JSONB: SELECT data->>'field' FROM table WHERE data @> '{"key": "val"}'::jsonb
// Arrays: WHERE 'tag' = ANY(tags) / WHERE tags && ARRAY['a','b']
// UPSERT: INSERT ... ON CONFLICT ... DO UPDATE SET ...
// Bulk: conn.BeginBinaryImport("COPY table FROM STDIN (FORMAT BINARY)")
// Listen: conn.Notification.Add(handler); LISTEN channel;
```
