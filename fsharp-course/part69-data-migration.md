# Part 69 - การย้ายข้อมูล (Data Migration)

## บทนำ (Introduction)

Data Migration คือกระบวนการเปลี่ยนแปลง schema ของฐานข้อมูล หรือย้ายข้อมูลจากรูปแบบหนึ่งไปอีกรูปแบบหนึ่ง อย่างปลอดภัยและเป็นระบบ

**ทำไมต้องมี Migration?**
- เพิ่ม/ลบ/เปลี่ยน columns
- เพิ่ม indexes
- เปลี่ยน data types
- ย้าย data ระหว่าง tables
- Rename ตาราง/คอลัมน์

---

## 1. DbUp Library

```fsharp
// DbUpExample.fs
module DbUpExample

open System
open DbUp  // NuGet: dbup-core, dbup-sqlserver, dbup-postgresql, dbup-sqlite

// ========================================
// 1.1 การติดตั้ง (Installation)
// ========================================

(*
NuGet packages:
- dbup-core: Core library
- dbup-sqlite: SQLite support
- dbup-postgresql: PostgreSQL support
- dbup-sqlserver: SQL Server support
- dbup-mysql: MySQL support
*)

// ========================================
// 1.2 Basic DbUp setup
// ========================================

let runMigrations (connectionString: string) =
    let upgrader =
        DeployChanges
            .To.SQLiteDatabase(connectionString)
            .WithScriptsEmbeddedInAssembly(System.Reflection.Assembly.GetExecutingAssembly())
            .WithTransaction()
            .LogToConsole()
            .Build()

    let result = upgrader.PerformUpgrade()
    if result.Successful then
        printfn "Database migration successful!"
        true
    else
        printfn "Database migration failed: %s" result.Error.Message
        false

// ========================================
// 1.3 DbUp with PostgreSQL
// ========================================

let runPostgresMigrations (connectionString: string) =
    let upgrader =
        DeployChanges
            .To.PostgresqlDatabase(connectionString)
            .WithScriptsEmbeddedInAssembly(
                System.Reflection.Assembly.GetExecutingAssembly(),
                fun name -> name.Contains("Migrations")
            )
            .WithTransaction()
            .LogToConsole()
            .JournalToPostgresqlTable("public", "schema_versions")
            .Build()

    let isUpgradeRequired = upgrader.IsUpgradeRequired()
    printfn "Upgrade required: %b" isUpgradeRequired

    if isUpgradeRequired then
        let result = upgrader.PerformUpgrade()
        result.Successful
    else
        printfn "Database is up to date"
        true

// ========================================
// 1.4 Script naming convention
// ========================================

(*
SQL scripts ที่ DbUp ใช้มักตั้งชื่อตามนี้:
Scripts/
  0001_InitialCreate.sql
  0002_AddUsersTable.sql
  0003_AddProductsTable.sql
  0004_AddIndexes.sql
  0005_SeedData.sql

แต่ละ script จะรันครั้งเดียวและถูกบันทึกใน SchemaVersions table
*)

// ========================================
// 1.5 SQL Migration Scripts
// ========================================

// Example SQL scripts:

let createUsersScript = """
-- 0001_CreateUsersTable.sql
CREATE TABLE IF NOT EXISTS users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(300) NOT NULL UNIQUE,
    password_hash VARCHAR(500) NOT NULL,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_is_active ON users(is_active) WHERE is_active = TRUE;
"""

let addUserProfileScript = """
-- 0002_AddUserProfile.sql
CREATE TABLE IF NOT EXISTS user_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL UNIQUE REFERENCES users(id) ON DELETE CASCADE,
    bio TEXT,
    avatar_url VARCHAR(500),
    phone_number VARCHAR(20),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Backfill: สร้าง profile สำหรับ users ที่มีอยู่แล้ว
INSERT INTO user_profiles (user_id)
SELECT id FROM users
WHERE id NOT IN (SELECT user_id FROM user_profiles);
"""

let addProductsScript = """
-- 0003_AddProductsTable.sql
CREATE TABLE IF NOT EXISTS categories (
    id SERIAL PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    description TEXT,
    parent_id INTEGER REFERENCES categories(id),
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(300) NOT NULL,
    price NUMERIC(10,2) NOT NULL CHECK (price >= 0),
    stock INTEGER NOT NULL DEFAULT 0 CHECK (stock >= 0),
    category_id INTEGER REFERENCES categories(id),
    description TEXT,
    tags TEXT[] DEFAULT '{}',
    metadata JSONB,
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ
);

CREATE INDEX idx_products_category ON products(category_id);
CREATE INDEX idx_products_price ON products(price);
CREATE INDEX idx_products_tags ON products USING gin(tags);
CREATE INDEX idx_products_metadata ON products USING gin(metadata);
CREATE INDEX idx_products_search ON products USING gin(
    to_tsvector('english', name || ' ' || COALESCE(description, ''))
);
"""

// ========================================
// 1.6 ตัวอย่างการใช้งาน F# scripts
// ========================================

let runCustomScript (connectionString: string) (scriptName: string) (sql: string) =
    let upgrader =
        DeployChanges
            .To.SQLiteDatabase(connectionString)
            .WithScript(scriptName, sql)
            .WithTransaction()
            .LogToConsole()
            .Build()

    let result = upgrader.PerformUpgrade()
    result.Successful
```

---

## 2. FluentMigrator

```fsharp
// FluentMigratorExample.fs
module FluentMigratorExample

// ========================================
// 2.1 Migration classes (ต้องใช้ C# style สำหรับ FluentMigrator)
// ========================================

// FluentMigrator ทำงานกับ F# ได้แต่ต้องใช้ class-based approach
// เพราะ attribute inheritance ใน F# มีข้อจำกัด

(*
// ใน F# ต้องทำแบบนี้:

open FluentMigrator

[<Migration(20240101001L)>]
type InitialCreate() =
    inherit Migration()

    override this.Up() =
        this.Create.Table("Users")
            .WithColumn("Id").AsInt32().PrimaryKey().Identity()
            .WithColumn("Name").AsString(200).NotNullable()
            .WithColumn("Email").AsString(300).NotNullable().Unique()
            .WithColumn("CreatedAt").AsDateTime().NotNullable().WithDefaultValue(SystemMethods.CurrentUTCDateTime)
        |> ignore

    override this.Down() =
        this.Delete.Table("Users") |> ignore

[<Migration(20240101002L)>]
type AddUserProfile() =
    inherit Migration()

    override this.Up() =
        this.Create.Table("UserProfiles")
            .WithColumn("Id").AsInt32().PrimaryKey().Identity()
            .WithColumn("UserId").AsInt32().NotNullable().ForeignKey("Users", "Id")
            .WithColumn("Bio").AsString(2000).Nullable()
        |> ignore

        this.Create.Index("IX_UserProfiles_UserId")
            .OnTable("UserProfiles")
            .OnColumn("UserId").Ascending()
        |> ignore

    override this.Down() =
        this.Delete.Table("UserProfiles") |> ignore

[<Migration(20240101003L)>]
type AddProductCategoryIndex() =
    inherit Migration()

    override this.Up() =
        this.Create.Column("CategoryId").OnTable("Products").AsInt32().Nullable() |> ignore
        this.Create.Index("IX_Products_CategoryId")
            .OnTable("Products")
            .OnColumn("CategoryId")
        |> ignore

    override this.Down() =
        this.Delete.Index("IX_Products_CategoryId").OnTable("Products") |> ignore
        this.Delete.Column("CategoryId").FromTable("Products") |> ignore
*)

// ========================================
// 2.2 FluentMigrator runner
// ========================================

let runFluentMigrations (connectionString: string) =
    // ใช้ Microsoft.Extensions.DependencyInjection
    printfn "FluentMigrator Example:"
    printfn "1. Define migrations as classes inheriting from Migration"
    printfn "2. Add [<Migration(version)>] attribute"
    printfn "3. Override Up() and Down() methods"
    printfn "4. Run migrations via MigrationRunner"

    // Code ตัวอย่าง (ต้องมี DI setup):
    (*
    let services = ServiceCollection()
    services.AddFluentMigratorCore()
        .ConfigureRunner(fun rb ->
            rb.AddPostgres()
              .WithGlobalConnectionString(connectionString)
              .ScanIn(Assembly.GetExecutingAssembly()).For.Migrations()
        ) |> ignore

    let serviceProvider = services.BuildServiceProvider()
    let runner = serviceProvider.GetRequiredService<IMigrationRunner>()
    runner.MigrateUp()
    *)
    ()
```

---

## 3. EF Core Migrations

```fsharp
// EFCoreMigrations.fs
module EFCoreMigrations

open Microsoft.EntityFrameworkCore

// ========================================
// 3.1 EF Core migration commands
// ========================================

(*
dotnet ef migrations add InitialCreate --output-dir Migrations
dotnet ef migrations add AddUserProfile
dotnet ef database update
dotnet ef database update 20240101001_InitialCreate  -- rollback to specific
dotnet ef migrations list
dotnet ef migrations remove  -- remove last pending migration
dotnet ef database drop
dotnet ef migrations script  -- generate SQL script
*)

// ========================================
// 3.2 Applying migrations programmatically
// ========================================

let applyMigrationsOnStartup (context: DbContext) =
    task {
        let pendingMigrations = context.Database.GetPendingMigrations()
        if not (Seq.isEmpty pendingMigrations) then
            printfn "Applying %d pending migrations..." (Seq.length pendingMigrations)
            for migration in pendingMigrations do
                printfn "  - %s" migration
            do! context.Database.MigrateAsync()
            printfn "Migrations applied successfully"
        else
            printfn "Database is up to date"
    }

// ========================================
// 3.3 EF Core migration ที่มี data transformation
// ========================================

(*
// ใน Migration class (C#-like):
public override void Up()
{
    // 1. Add new column (nullable)
    migrationBuilder.AddColumn<string>(
        name: "FullName",
        table: "Users",
        nullable: true);

    // 2. Migrate data
    migrationBuilder.Sql(@"
        UPDATE Users
        SET FullName = FirstName || ' ' || LastName
        WHERE FirstName IS NOT NULL AND LastName IS NOT NULL
    ");

    // 3. Make column not nullable
    migrationBuilder.AlterColumn<string>(
        name: "FullName",
        table: "Users",
        nullable: false,
        defaultValue: "");

    // 4. Drop old columns
    migrationBuilder.DropColumn("FirstName", "Users");
    migrationBuilder.DropColumn("LastName", "Users");
}
*)
```

---

## 4. SQL Scripts Approach

```fsharp
// SqlScripts.fs
module SqlScripts

open System
open System.IO

// ========================================
// 4.1 Custom SQL script runner
// ========================================

type MigrationScript = {
    Version: int
    Name: string
    Sql: string
    AppliedAt: DateTime option
}

type ScriptRunner(connectionString: string) =
    // ต้องมี schema_versions table ก่อน
    let ensureVersionTable (conn: Npgsql.NpgsqlConnection) =
        let sql = """
            CREATE TABLE IF NOT EXISTS schema_versions (
                id SERIAL PRIMARY KEY,
                version INTEGER NOT NULL UNIQUE,
                name VARCHAR(500) NOT NULL,
                applied_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
                checksum VARCHAR(64)
            )
        """
        use cmd = new Npgsql.NpgsqlCommand(sql, conn)
        cmd.ExecuteNonQuery() |> ignore

    let getAppliedVersions (conn: Npgsql.NpgsqlConnection) =
        let sql = "SELECT version FROM schema_versions ORDER BY version"
        use cmd = new Npgsql.NpgsqlCommand(sql, conn)
        use reader = cmd.ExecuteReader()
        [ while reader.Read() do yield reader.GetInt32(0) ]

    let markApplied (conn: Npgsql.NpgsqlConnection) (transaction: Npgsql.NpgsqlTransaction) (script: MigrationScript) =
        let sql = """
            INSERT INTO schema_versions (version, name, checksum)
            VALUES (@Version, @Name, @Checksum)
        """
        use cmd = new Npgsql.NpgsqlCommand(sql, conn, transaction)
        cmd.Parameters.AddWithValue("Version", script.Version) |> ignore
        cmd.Parameters.AddWithValue("Name", script.Name) |> ignore
        let checksum = script.Sql |> System.Text.Encoding.UTF8.GetBytes |> System.Security.Cryptography.SHA256.HashData |> Convert.ToHexString
        cmd.Parameters.AddWithValue("Checksum", checksum) |> ignore
        cmd.ExecuteNonQuery() |> ignore

    member _.Run (scripts: MigrationScript list) =
        use conn = new Npgsql.NpgsqlConnection(connectionString)
        conn.Open()
        ensureVersionTable conn

        let appliedVersions = getAppliedVersions conn |> Set.ofList
        let pending = scripts |> List.filter (fun s -> not (Set.contains s.Version appliedVersions))

        if pending.IsEmpty then
            printfn "No pending migrations"
        else
            printfn "Applying %d migration(s)..." pending.Length
            for script in pending |> List.sortBy (fun s -> s.Version) do
                use transaction = conn.BeginTransaction()
                try
                    printfn "  Applying: %d_%s" script.Version script.Name
                    use cmd = new Npgsql.NpgsqlCommand(script.Sql, conn, transaction)
                    cmd.ExecuteNonQuery() |> ignore
                    markApplied conn transaction script
                    transaction.Commit()
                    printfn "  Applied: %d_%s" script.Version script.Name
                with ex ->
                    transaction.Rollback()
                    printfn "  Failed: %s" ex.Message
                    raise ex

// ========================================
// 4.2 Load scripts จากไฟล์
// ========================================

let loadScriptsFromDirectory (directoryPath: string) =
    if not (Directory.Exists directoryPath) then
        []
    else
        Directory.GetFiles(directoryPath, "*.sql")
        |> Array.sortBy id
        |> Array.toList
        |> List.choose (fun filePath ->
            let fileName = Path.GetFileNameWithoutExtension(filePath)
            // Expected format: 0001_MigrationName.sql
            match fileName.Split('_') with
            | [| versionStr; name |] ->
                match Int32.TryParse versionStr with
                | true, version ->
                    Some {
                        Version = version
                        Name = name
                        Sql = File.ReadAllText filePath
                        AppliedAt = None
                    }
                | _ -> None
            | _ -> None
        )

// ========================================
// 4.3 Rollback strategy
// ========================================

type MigrationWithRollback = {
    Version: int
    Name: string
    UpSql: string
    DownSql: string
}

let rollbackMigration (connectionString: string) (version: int) (scripts: MigrationWithRollback list) =
    match scripts |> List.tryFind (fun s -> s.Version = version) with
    | None ->
        printfn "Migration version %d not found" version
        false
    | Some script ->
        use conn = new Npgsql.NpgsqlConnection(connectionString)
        conn.Open()
        use transaction = conn.BeginTransaction()
        try
            printfn "Rolling back: %d_%s" script.Version script.Name
            use cmd = new Npgsql.NpgsqlCommand(script.DownSql, conn, transaction)
            cmd.ExecuteNonQuery() |> ignore

            use deleteCmd = new Npgsql.NpgsqlCommand(
                "DELETE FROM schema_versions WHERE version = @Version",
                conn, transaction
            )
            deleteCmd.Parameters.AddWithValue("Version", version) |> ignore
            deleteCmd.ExecuteNonQuery() |> ignore

            transaction.Commit()
            printfn "Rollback successful"
            true
        with ex ->
            transaction.Rollback()
            printfn "Rollback failed: %s" ex.Message
            false
```

---

## 5. Zero-Downtime Migrations

```fsharp
// ZeroDowntimeMigrations.fs
module ZeroDowntimeMigrations

// ========================================
// 5.1 Zero-downtime migration strategies
// ========================================

(*
Strategy 1: Expand-Contract (Parallel Change)
=============================================
ปัญหา: Rename column users.name -> users.full_name
แต่ไม่อยากหยุด service

Phase 1 - EXPAND (เพิ่ม column ใหม่):
    ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
    UPDATE users SET full_name = name;
    -- Create trigger to sync both columns
    CREATE OR REPLACE FUNCTION sync_user_name()
    RETURNS TRIGGER AS $$
    BEGIN
        IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
            IF NEW.name IS NOT NULL AND NEW.full_name IS NULL THEN
                NEW.full_name := NEW.name;
            ELSIF NEW.full_name IS NOT NULL AND NEW.name IS NULL THEN
                NEW.name := NEW.full_name;
            END IF;
        END IF;
        RETURN NEW;
    END;
    $$ LANGUAGE plpgsql;

    CREATE TRIGGER sync_user_name_trigger
    BEFORE INSERT OR UPDATE ON users
    FOR EACH ROW EXECUTE FUNCTION sync_user_name();

Phase 2 - DEPLOY new code (reads from full_name, writes to both):
    -- Deploy application version that uses full_name
    -- Old code still works because both columns exist

Phase 3 - CONTRACT (remove old column):
    DROP TRIGGER sync_user_name_trigger ON users;
    DROP FUNCTION sync_user_name();
    ALTER TABLE users DROP COLUMN name;
*)

// ========================================
// 5.2 F# implementation ของ expand-contract
// ========================================

type MigrationPhase =
    | Expand      // เพิ่ม new structure, keep old
    | Contract    // ลบ old structure

let expandPhaseScript = """
-- Phase 1: EXPAND
-- เพิ่ม full_name column
ALTER TABLE users ADD COLUMN IF NOT EXISTS full_name VARCHAR(200);

-- Copy data
UPDATE users SET full_name = name WHERE full_name IS NULL;

-- Trigger to sync ทั้ง 2 columns
CREATE OR REPLACE FUNCTION sync_user_name()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'INSERT' OR TG_OP = 'UPDATE' THEN
        IF NEW.name IS NOT NULL AND NEW.full_name IS NULL THEN
            NEW.full_name = NEW.name;
        ELSIF NEW.full_name IS NOT NULL AND NEW.name IS NULL THEN
            NEW.name = NEW.full_name;
        END IF;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER IF NOT EXISTS sync_user_name_trigger
BEFORE INSERT OR UPDATE ON users
FOR EACH ROW EXECUTE FUNCTION sync_user_name();
"""

let contractPhaseScript = """
-- Phase 3: CONTRACT (หลัง deploy code ใหม่แล้ว)
DROP TRIGGER IF EXISTS sync_user_name_trigger ON users;
DROP FUNCTION IF EXISTS sync_user_name();
ALTER TABLE users DROP COLUMN IF EXISTS name;
"""

// ========================================
// 5.3 Adding NOT NULL column safely
// ========================================

(*
ปัญหา: ต้องการเพิ่ม column ที่ NOT NULL แต่มี rows อยู่แล้ว

ห้ามทำ: ALTER TABLE users ADD COLUMN phone VARCHAR(20) NOT NULL;
-- จะ error เพราะ rows เดิมไม่มี value

ทำแบบนี้แทน:
*)

let addNotNullColumnSafely = """
-- Step 1: เพิ่ม column แบบ nullable ก่อน
ALTER TABLE users ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

-- Step 2: Set default value สำหรับ rows เดิม
UPDATE users SET phone = 'N/A' WHERE phone IS NULL;

-- Step 3: Add NOT NULL constraint
ALTER TABLE users ALTER COLUMN phone SET NOT NULL;

-- Step 4: Add DEFAULT ถ้าต้องการ
ALTER TABLE users ALTER COLUMN phone SET DEFAULT 'N/A';
"""

// ========================================
// 5.4 Large table migration (batching)
// ========================================

let batchMigrationScript = """
-- Migrate data in batches เพื่อไม่ให้ lock table นานเกินไป
DO $$
DECLARE
    batch_size INT := 10000;
    last_id INT := 0;
    max_id INT;
    rows_updated INT;
BEGIN
    SELECT MAX(id) INTO max_id FROM orders;

    LOOP
        UPDATE orders
        SET status = 'legacy_' || status
        WHERE id > last_id AND id <= last_id + batch_size
          AND status NOT LIKE 'legacy_%';

        GET DIAGNOSTICS rows_updated = ROW_COUNT;

        last_id := last_id + batch_size;
        EXIT WHEN last_id > max_id OR rows_updated = 0;

        -- เพิ่ม delay เล็กน้อยเพื่อลด load
        PERFORM pg_sleep(0.1);

        RAISE NOTICE 'Processed up to id %', last_id;
    END LOOP;

    RAISE NOTICE 'Migration complete';
END $$;
"""
```

---

## 6. Testing Migrations

```fsharp
// TestingMigrations.fs
module TestingMigrations

open System

// ========================================
// 6.1 Migration testing strategy
// ========================================

(*
วิธีทดสอบ migrations:
1. ใช้ in-memory SQLite สำหรับ unit tests
2. ใช้ Docker PostgreSQL สำหรับ integration tests
3. Test ทั้ง Up และ Down migrations
4. Test data integrity หลัง migration
*)

// ========================================
// 6.2 Migration test helper
// ========================================

type MigrationTestFixture(connectionString: string) =
    member _.RunMigrationsUp () =
        task {
            // Apply all migrations
            let upgrader =
                DbUp.DeployChanges
                    .To.SQLiteDatabase(connectionString)
                    .WithScriptsEmbeddedInAssembly(System.Reflection.Assembly.GetExecutingAssembly())
                    .LogToConsole()
                    .Build()

            let result = upgrader.PerformUpgrade()
            return result.Successful
        }

    member _.VerifyTableExists (tableName: string) =
        task {
            use conn = new Microsoft.Data.Sqlite.SqliteConnection(connectionString)
            conn.Open()
            let sql = "SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name=@TableName"
            use cmd = new Microsoft.Data.Sqlite.SqliteCommand(sql, conn)
            cmd.Parameters.AddWithValue("@TableName", tableName) |> ignore
            let count = cmd.ExecuteScalar() :?> int64
            return count > 0L
        }

    member _.VerifyColumnExists (tableName: string) (columnName: string) =
        task {
            use conn = new Microsoft.Data.Sqlite.SqliteConnection(connectionString)
            conn.Open()
            let sql = $"PRAGMA table_info({tableName})"
            use cmd = new Microsoft.Data.Sqlite.SqliteCommand(sql, conn)
            use reader = cmd.ExecuteReader()
            return
                seq {
                    while reader.Read() do
                        yield reader.GetString(1)  // column name
                }
                |> Seq.exists (fun col -> col = columnName)
        }

// ========================================
// 6.3 Example test scenarios
// ========================================

let runMigrationTests () =
    task {
        let connStr = "Data Source=:memory:"
        let fixture = MigrationTestFixture(connStr)

        printfn "=== Migration Tests ==="

        // Test 1: ตรวจสอบว่า migrations รันสำเร็จ
        let! success = fixture.RunMigrationsUp()
        printfn "Migrations applied: %b" success

        // Test 2: ตรวจสอบว่า tables มีอยู่
        let tables = ["users"; "products"; "orders"; "order_items"]
        for table in tables do
            let! exists = fixture.VerifyTableExists table
            printfn "Table '%s' exists: %b" table exists

        // Test 3: ตรวจสอบ columns
        let! hasEmail = fixture.VerifyColumnExists "users" "email"
        printfn "Column 'email' in 'users': %b" hasEmail

        printfn "=== Tests Complete ==="
    }
```

---

## 7. Complete Migration Example

```fsharp
// Program.fs
module Program

open System
open Microsoft.Data.Sqlite

[<EntryPoint>]
let main _ =
    printfn "=== Data Migration F# Demo ==="
    printfn "=============================="

    // ========================================
    // Setup in-memory SQLite database
    // ========================================
    let connectionString = "Data Source=migration-demo.db;Mode=ReadWriteCreate"
    use conn = new SqliteConnection(connectionString)
    conn.Open()

    // ========================================
    // Manual migration approach (demo)
    // ========================================
    printfn "\n--- Running Migrations ---"

    let migrations = [
        1, "CreateSchemaVersions", """
            CREATE TABLE IF NOT EXISTS schema_versions (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                version INTEGER NOT NULL UNIQUE,
                name TEXT NOT NULL,
                applied_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
            )
        """
        2, "CreateUsers", """
            CREATE TABLE IF NOT EXISTS users (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT NOT NULL UNIQUE,
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                is_active INTEGER NOT NULL DEFAULT 1
            )
        """
        3, "CreateCategories", """
            CREATE TABLE IF NOT EXISTS categories (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                description TEXT
            )
        """
        4, "CreateProducts", """
            CREATE TABLE IF NOT EXISTS products (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                price REAL NOT NULL,
                stock INTEGER NOT NULL DEFAULT 0,
                category_id INTEGER REFERENCES categories(id),
                is_active INTEGER NOT NULL DEFAULT 1,
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
            )
        """
        5, "CreateOrders", """
            CREATE TABLE IF NOT EXISTS orders (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                user_id INTEGER NOT NULL REFERENCES users(id),
                total_amount REAL NOT NULL,
                status TEXT NOT NULL DEFAULT 'pending',
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
            );

            CREATE TABLE IF NOT EXISTS order_items (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                order_id INTEGER NOT NULL REFERENCES orders(id),
                product_id INTEGER NOT NULL REFERENCES products(id),
                quantity INTEGER NOT NULL,
                unit_price REAL NOT NULL
            )
        """
        6, "AddUserProfileColumn", """
            ALTER TABLE users ADD COLUMN bio TEXT;
            ALTER TABLE users ADD COLUMN avatar_url TEXT
        """
        7, "AddProductIndex", """
            CREATE INDEX IF NOT EXISTS idx_products_category ON products(category_id);
            CREATE INDEX IF NOT EXISTS idx_products_active ON products(is_active)
        """
        8, "SeedInitialData", """
            INSERT OR IGNORE INTO categories (id, name) VALUES
            (1, 'Electronics'),
            (2, 'Accessories'),
            (3, 'Software');

            INSERT OR IGNORE INTO products (id, name, price, stock, category_id) VALUES
            (1, 'Laptop Pro', 45999, 50, 1),
            (2, 'Wireless Mouse', 799, 200, 2),
            (3, 'VS Code', 0, 999, 3)
        """
    ]

    // Run pending migrations
    let getAppliedVersions () =
        use cmd = new SqliteCommand("SELECT version FROM schema_versions ORDER BY version", conn)
        use reader = cmd.ExecuteReader()
        [ while reader.Read() do yield reader.GetInt32(0) ]

    let applyMigration version name sql =
        use transaction = conn.BeginTransaction()
        try
            use cmd = new SqliteCommand(sql, conn, transaction)
            cmd.ExecuteNonQuery() |> ignore

            use versionCmd = new SqliteCommand(
                "INSERT INTO schema_versions (version, name) VALUES (@V, @N)",
                conn, transaction
            )
            versionCmd.Parameters.AddWithValue("@V", version) |> ignore
            versionCmd.Parameters.AddWithValue("@N", name) |> ignore
            versionCmd.ExecuteNonQuery() |> ignore

            transaction.Commit()
            printfn "  ✓ Applied migration %d: %s" version name
        with ex ->
            transaction.Rollback()
            printfn "  ✗ Failed migration %d: %s - %s" version name ex.Message
            raise ex

    // Ensure schema_versions table exists first
    use initCmd = new SqliteCommand(
        "CREATE TABLE IF NOT EXISTS schema_versions (id INTEGER PRIMARY KEY AUTOINCREMENT, version INTEGER NOT NULL UNIQUE, name TEXT NOT NULL, applied_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP)",
        conn
    )
    initCmd.ExecuteNonQuery() |> ignore

    let appliedVersions = getAppliedVersions() |> Set.ofList
    let pendingMigrations = migrations |> List.filter (fun (v, _, _) -> not (Set.contains v appliedVersions))

    if pendingMigrations.IsEmpty then
        printfn "Database is up to date"
    else
        printfn "Applying %d pending migration(s)..." pendingMigrations.Length
        for (version, name, sql) in pendingMigrations do
            applyMigration version name sql

    // ========================================
    // Verify schema
    // ========================================
    printfn "\n--- Schema Verification ---"

    let tables = ["users"; "categories"; "products"; "orders"; "order_items"; "schema_versions"]
    for tableName in tables do
        use cmd = new SqliteCommand(
            "SELECT COUNT(*) FROM sqlite_master WHERE type='table' AND name=@name", conn
        )
        cmd.Parameters.AddWithValue("@name", tableName) |> ignore
        let exists = cmd.ExecuteScalar() :?> int64 > 0L
        printfn "  Table '%s': %s" tableName (if exists then "✓ exists" else "✗ missing")

    // ========================================
    // Show applied migrations
    // ========================================
    printfn "\n--- Applied Migrations ---"
    use historyCmd = new SqliteCommand("SELECT version, name, applied_at FROM schema_versions ORDER BY version", conn)
    use reader = historyCmd.ExecuteReader()
    while reader.Read() do
        printfn "  [%d] %s (applied: %s)" (reader.GetInt32(0)) (reader.GetString(1)) (reader.GetString(2))

    // ========================================
    // Show current data
    // ========================================
    printfn "\n--- Current Data ---"
    use dataCmd = new SqliteCommand("SELECT id, name, price, stock FROM products", conn)
    use dataReader = dataCmd.ExecuteReader()
    printfn "Products:"
    while dataReader.Read() do
        printfn "  [%d] %s - ฿%.2f (stock: %d)"
            (dataReader.GetInt32(0))
            (dataReader.GetString(1))
            (dataReader.GetDouble(2))
            (dataReader.GetInt32(3))

    // ========================================
    // Migration best practices
    // ========================================
    printfn "\n--- Migration Best Practices ---"
    let practices = [
        "1. ทุก migration ต้องมี version number ที่ unique (เรียงลำดับ)"
        "2. Migration ต้องเป็น idempotent (รันซ้ำได้โดยไม่ error)"
        "3. ใช้ IF NOT EXISTS / IF EXISTS ป้องกัน duplicate errors"
        "4. Test migrations ใน dev/staging ก่อน production เสมอ"
        "5. เก็บ schema_versions table เพื่อติดตาม applied migrations"
        "6. Data migrations ต้อง batch ขนาดใหญ่เพื่อ performance"
        "7. Zero-downtime: ใช้ expand-contract pattern"
        "8. สำรองข้อมูลก่อน migrate production เสมอ"
        "9. ใช้ transactions - ถ้า migrate ล้มเหลว ต้อง rollback ได้"
        "10. Document ทุก migration: ทำอะไร ทำไม ผลกระทบคืออะไร"
    ]

    for p in practices do
        printfn "  %s" p

    printfn "\n=== Demo Complete ==="
    0
```

---

## สรุป (Summary)

Data Migration strategies ใน F#:

| เครื่องมือ | เหมาะกับ | ข้อดี | ข้อเสีย |
|---|---|---|---|
| **DbUp** | SQL script files | Simple, full SQL control | ต้องเขียน SQL เอง |
| **FluentMigrator** | C#-like migrations | Type-safe, fluent API | Verbose ใน F# |
| **EF Core Migrations** | EF Core projects | Auto-generate | ต้องใช้ EF Core |
| **Custom runner** | Special requirements | Full control | ต้องเขียนเอง |

```fsharp
// Key migration principles:
// 1. Version everything
// 2. Make migrations idempotent
// 3. Test Up and Down
// 4. Batch large data migrations
// 5. Zero-downtime with expand-contract
// 6. Always backup before production migration
```
