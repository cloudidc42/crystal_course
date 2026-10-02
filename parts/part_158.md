# Part 158: Database Migrations ใน Crystal

## บทนำ

Database Migration คือกระบวนการจัดการการเปลี่ยนแปลง schema ของฐานข้อมูลอย่างมีระเบียบ แทนที่จะแก้ไข schema โดยตรง เราสร้างไฟล์ migration ที่สามารถรัน ย้อนกลับ และติดตามประวัติได้

## ทำไมต้องใช้ Migrations

```
ปัญหาที่เกิดขึ้นโดยไม่มี Migrations:
- ทีมแต่ละคนมี schema ที่ต่างกัน
- ไม่รู้ว่า production มี schema เวอร์ชันไหน
- การ rollback ทำยาก
- ขาด audit trail ของการเปลี่ยนแปลง

ข้อดีของ Migrations:
+ ทุกคนมี schema เดียวกัน
+ ติดตามประวัติการเปลี่ยนแปลง
+ Rollback ง่าย
+ Deploy อัตโนมัติได้
```

## เครื่องมือ Migrations ใน Crystal

### 1. Micrate (ยอดนิยมที่สุด)

```yaml
# shard.yml
dependencies:
  micrate:
    github: amberframework/micrate
    version: ~> 0.14
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13
  pg:
    github: will/crystal-pg
    version: ~> 0.28
```

### 2. Jennifer (ORM พร้อม migration)

```yaml
dependencies:
  jennifer:
    github: imdrasil/jennifer.cr
    version: ~> 0.12
```

## Micrate - การใช้งาน

### ตั้งค่า

```crystal
# src/db.cr
require "micrate"
require "db"
require "pg"

Micrate::DB.connection_url = ENV["DATABASE_URL"]? ||
  "postgresql://postgres:password@localhost/myapp_dev"

# เรียกใช้ migration
Micrate::DB.migrate!
```

### สร้าง Migration

Migration files อยู่ใน `db/migrations/` ชื่อไฟล์รูปแบบ: `TIMESTAMP_description.sql`

```bash
# สร้าง migration ด้วย micrate CLI
micrate create create_users
# สร้างไฟล์: db/migrations/20241015120000_create_users.sql
```

### โครงสร้างไฟล์ Migration

```sql
-- db/migrations/20241015120000_create_users.sql

-- +micrate Up
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL,
  username VARCHAR(100) NOT NULL,
  password_digest VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  role VARCHAR(50) DEFAULT 'user' NOT NULL,
  active BOOLEAN DEFAULT true NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  CONSTRAINT users_email_unique UNIQUE (email),
  CONSTRAINT users_username_unique UNIQUE (username)
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);
CREATE INDEX idx_users_created_at ON users(created_at);

-- +micrate Down
DROP TABLE IF EXISTS users;
```

## การสร้าง Migrations ต่างๆ

### Create Table

```sql
-- db/migrations/20241015120100_create_products.sql

-- +micrate Up
CREATE TABLE products (
  id BIGSERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  description TEXT,
  price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
  stock INTEGER DEFAULT 0 NOT NULL CHECK (stock >= 0),
  sku VARCHAR(100) NOT NULL,
  category VARCHAR(100) NOT NULL,
  active BOOLEAN DEFAULT true NOT NULL,
  featured BOOLEAN DEFAULT false NOT NULL,
  weight DECIMAL(8, 3),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  CONSTRAINT products_sku_unique UNIQUE (sku)
);

CREATE INDEX idx_products_category ON products(category);
CREATE INDEX idx_products_active ON products(active);
CREATE INDEX idx_products_featured ON products(featured);
CREATE INDEX idx_products_price ON products(price);

-- ทำ full-text search
CREATE INDEX idx_products_name_fts ON products
  USING GIN (to_tsvector('thai', name || ' ' || COALESCE(description, '')));

-- +micrate Down
DROP TABLE IF EXISTS products;
```

### Add Column

```sql
-- db/migrations/20241015130000_add_phone_to_users.sql

-- +micrate Up
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
ALTER TABLE users ADD COLUMN address TEXT;
ALTER TABLE users ADD COLUMN avatar_url VARCHAR(500);
ALTER TABLE users ADD COLUMN last_login_at TIMESTAMP WITH TIME ZONE;

-- +micrate Down
ALTER TABLE users DROP COLUMN IF EXISTS phone;
ALTER TABLE users DROP COLUMN IF EXISTS address;
ALTER TABLE users DROP COLUMN IF EXISTS avatar_url;
ALTER TABLE users DROP COLUMN IF EXISTS last_login_at;
```

### Remove Column

```sql
-- db/migrations/20241015140000_remove_old_fields.sql

-- +micrate Up
-- สำรองข้อมูลก่อน (optional)
-- CREATE TABLE users_backup AS SELECT * FROM users;

ALTER TABLE users DROP COLUMN IF EXISTS legacy_field;
ALTER TABLE users DROP COLUMN IF EXISTS deprecated_status;

-- +micrate Down
ALTER TABLE users ADD COLUMN legacy_field VARCHAR(255);
ALTER TABLE users ADD COLUMN deprecated_status INTEGER;
```

### Rename Column

```sql
-- db/migrations/20241015150000_rename_user_columns.sql

-- +micrate Up
ALTER TABLE users RENAME COLUMN user_name TO username;
ALTER TABLE users RENAME COLUMN pwd TO password_digest;

-- +micrate Down
ALTER TABLE users RENAME COLUMN username TO user_name;
ALTER TABLE users RENAME COLUMN password_digest TO pwd;
```

### Change Column Type

```sql
-- db/migrations/20241015160000_change_product_price_type.sql

-- +micrate Up
-- เปลี่ยนจาก INTEGER เป็น DECIMAL
ALTER TABLE products
  ALTER COLUMN price TYPE DECIMAL(10,2)
  USING price::DECIMAL;

-- +micrate Down
ALTER TABLE products
  ALTER COLUMN price TYPE INTEGER
  USING price::INTEGER;
```

### Add Index

```sql
-- db/migrations/20241015170000_add_indexes.sql

-- +micrate Up
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);

-- Composite index
CREATE INDEX CONCURRENTLY idx_orders_user_status
  ON orders(user_id, status);

-- Partial index
CREATE INDEX CONCURRENTLY idx_orders_pending
  ON orders(created_at)
  WHERE status = 'pending';

-- +micrate Down
DROP INDEX IF EXISTS idx_orders_user_id;
DROP INDEX IF EXISTS idx_orders_status;
DROP INDEX IF EXISTS idx_orders_created_at;
DROP INDEX IF EXISTS idx_orders_user_status;
DROP INDEX IF EXISTS idx_orders_pending;
```

### Add Foreign Key

```sql
-- db/migrations/20241015180000_add_foreign_keys.sql

-- +micrate Up
ALTER TABLE orders
  ADD COLUMN user_id BIGINT,
  ADD CONSTRAINT fk_orders_user
    FOREIGN KEY (user_id)
    REFERENCES users(id)
    ON DELETE SET NULL
    ON UPDATE CASCADE;

ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_order
    FOREIGN KEY (order_id)
    REFERENCES orders(id)
    ON DELETE CASCADE;

ALTER TABLE order_items
  ADD CONSTRAINT fk_order_items_product
    FOREIGN KEY (product_id)
    REFERENCES products(id)
    ON DELETE RESTRICT;

-- +micrate Down
ALTER TABLE orders DROP CONSTRAINT IF EXISTS fk_orders_user;
ALTER TABLE orders DROP COLUMN IF EXISTS user_id;
ALTER TABLE order_items DROP CONSTRAINT IF EXISTS fk_order_items_order;
ALTER TABLE order_items DROP CONSTRAINT IF EXISTS fk_order_items_product;
```

### Create Join Table

```sql
-- db/migrations/20241015190000_create_post_tags.sql

-- +micrate Up
CREATE TABLE post_tags (
  id BIGSERIAL PRIMARY KEY,
  post_id BIGINT NOT NULL,
  tag_id BIGINT NOT NULL,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW() NOT NULL,
  CONSTRAINT fk_post_tags_post
    FOREIGN KEY (post_id) REFERENCES posts(id) ON DELETE CASCADE,
  CONSTRAINT fk_post_tags_tag
    FOREIGN KEY (tag_id) REFERENCES tags(id) ON DELETE CASCADE,
  CONSTRAINT unique_post_tag UNIQUE (post_id, tag_id)
);

CREATE INDEX idx_post_tags_post_id ON post_tags(post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags(tag_id);

-- +micrate Down
DROP TABLE IF EXISTS post_tags;
```

## Running Migrations

```crystal
# src/migrate.cr
require "micrate"
require "db"
require "pg"

Micrate::DB.connection_url = ENV["DATABASE_URL"]

case ARGV[0]?
when "up"
  Micrate::DB.migrate!
  puts "Migration สำเร็จ"
when "down"
  Micrate::DB.rollback!
  puts "Rollback สำเร็จ"
when "status"
  Micrate::DB.status
when "create"
  name = ARGV[1]? || "new_migration"
  Micrate.create_migration(name)
else
  puts "Usage: crystal run src/migrate.cr -- [up|down|status|create <name>]"
end
```

```bash
# รัน migrations ทั้งหมด
crystal run src/migrate.cr -- up

# Rollback migration ล่าสุด
crystal run src/migrate.cr -- down

# ดูสถานะ
crystal run src/migrate.cr -- status

# สร้าง migration ใหม่
crystal run src/migrate.cr -- create add_payment_to_orders
```

## Version Tracking

Micrate เก็บ version ใน table `micrate_db_version`:

```sql
-- ดู migrations ที่รันแล้ว
SELECT * FROM micrate_db_version ORDER BY version_id;

-- ตัวอย่างผลลัพธ์:
-- version_id | dirty
-- 20241015120000 | false
-- 20241015120100 | false
-- 20241015130000 | false
```

```crystal
# ดู migration status ใน code
require "micrate"

status = Micrate::DB.status
status.each do |version, applied|
  puts "#{version}: #{applied ? "✓ Applied" : "✗ Pending"}"
end
```

## Jennifer ORM Migrations

Jennifer มี DSL สำหรับ migration ที่ใช้ Crystal แทน SQL:

```crystal
# db/migrations/20241015120000_create_users.cr
class CreateUsers < Jennifer::Migration::Base
  def up
    create_table :users do |t|
      t.string :email, null: false
      t.string :username, null: false, size: 100
      t.string :password_digest, null: false
      t.string :first_name, null: true
      t.string :last_name, null: true
      t.string :role, default: "user"
      t.bool :active, default: true
      t.timestamps

      t.index :email, unique: true
      t.index :username, unique: true
    end
  end

  def down
    drop_table :users
  end
end
```

```crystal
# db/migrations/20241015130000_add_columns_to_users.cr
class AddColumnsToUsers < Jennifer::Migration::Base
  def up
    change_table :users do |t|
      t.add_column :phone, :string, null: true, size: 20
      t.add_column :avatar_url, :string, null: true
      t.add_column :last_login_at, :timestamp, null: true
    end

    add_index :users, :phone
  end

  def down
    change_table :users do |t|
      t.drop_column :phone
      t.drop_column :avatar_url
      t.drop_column :last_login_at
    end

    drop_index :users, :phone
  end
end
```

### Jennifer Migration DSL

```crystal
class ComplexMigration < Jennifer::Migration::Base
  def up
    # Create table
    create_table :products do |t|
      t.string :name, null: false
      t.text :description
      t.decimal :price, precision: 10, scale: 2
      t.integer :stock, default: 0
      t.string :sku, null: false
      t.enum :category, values: %w[electronics clothing food books]
      t.bool :active, default: true
      t.timestamps

      t.index :sku, unique: true
      t.index :category
      t.index [:category, :active]  # composite index
    end

    # Create foreign key
    add_foreign_key :orders, :users, column: :user_id

    # Execute raw SQL
    exec "CREATE INDEX CONCURRENTLY idx_products_name_fts ON products USING GIN (to_tsvector('english', name))"
  end

  def down
    exec "DROP INDEX IF EXISTS idx_products_name_fts"
    drop_foreign_key :orders, :users
    drop_table :products
  end
end
```

## Migration Best Practices

### 1. ทำ Migration เป็น Idempotent

```sql
-- ดีกว่า
CREATE TABLE IF NOT EXISTS users (...);
DROP TABLE IF EXISTS users;

ALTER TABLE users
  ADD COLUMN IF NOT EXISTS phone VARCHAR(20);

CREATE INDEX IF NOT EXISTS idx_users_email ON users(email);
DROP INDEX IF EXISTS idx_users_email;
```

### 2. ใช้ CONCURRENTLY สำหรับ Index บน Production

```sql
-- ไม่ lock table ระหว่าง create index
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

### 3. Zero-downtime Migrations

```sql
-- ขั้นตอนที่ 1: เพิ่ม column ที่ nullable
ALTER TABLE users ADD COLUMN new_status VARCHAR(50);

-- ขั้นตอนที่ 2: เติมข้อมูล (deploy application ที่ใช้ทั้ง old และ new column)
UPDATE users SET new_status = status WHERE new_status IS NULL;

-- ขั้นตอนที่ 3: เพิ่ม constraint
ALTER TABLE users ALTER COLUMN new_status SET NOT NULL;
ALTER TABLE users ALTER COLUMN new_status SET DEFAULT 'active';

-- ขั้นตอนที่ 4: ลบ old column (หลัง deploy application ที่ใช้แค่ new column)
ALTER TABLE users DROP COLUMN status;
```

### 4. สร้าง Custom Migration Runner

```crystal
# src/migration_runner.cr
require "db"
require "pg"

class MigrationRunner
  MIGRATIONS_DIR = "db/migrations"

  def initialize(@db_url : String)
  end

  def run!
    DB.open(@db_url) do |db|
      setup_migrations_table(db)
      pending = pending_migrations(db)

      if pending.empty?
        puts "ไม่มี migration ที่ต้องรัน"
        return
      end

      puts "กำลังรัน #{pending.size} migration(s)..."
      pending.each do |file|
        run_migration(db, file)
      end

      puts "เสร็จสิ้น!"
    end
  end

  def rollback!(steps = 1)
    DB.open(@db_url) do |db|
      applied = applied_migrations(db)
      to_rollback = applied.last(steps)

      to_rollback.reverse_each do |version|
        file = find_migration_file(version)
        rollback_migration(db, file, version)
      end
    end
  end

  def status
    DB.open(@db_url) do |db|
      setup_migrations_table(db)
      applied = applied_migrations(db).to_set

      all_migrations = Dir.glob("#{MIGRATIONS_DIR}/*.sql").sort.map do |f|
        version = File.basename(f, ".sql")
        {version, applied.includes?(version)}
      end

      all_migrations.each do |version, is_applied|
        status = is_applied ? "✓" : "✗"
        puts "#{status} #{version}"
      end
    end
  end

  private def setup_migrations_table(db)
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS schema_migrations (
        version VARCHAR(255) PRIMARY KEY,
        applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
      )
    SQL
  end

  private def pending_migrations(db) : Array(String)
    applied = applied_migrations(db).to_set
    Dir.glob("#{MIGRATIONS_DIR}/*.sql")
      .sort
      .select { |f| !applied.includes?(File.basename(f, ".sql")) }
  end

  private def applied_migrations(db) : Array(String)
    db.query_all(
      "SELECT version FROM schema_migrations ORDER BY version",
      as: String
    )
  end

  private def run_migration(db, file : String)
    version = File.basename(file, ".sql")
    content = File.read(file)

    # แยก Up section
    up_sql = extract_section(content, "Up")

    puts "  ▶ #{version}"
    db.transaction do |tx|
      tx.connection.exec(up_sql)
      tx.connection.exec(
        "INSERT INTO schema_migrations (version) VALUES ($1)",
        version
      )
    end
    puts "    ✓ Done"
  rescue ex
    puts "    ✗ Failed: #{ex.message}"
    raise ex
  end

  private def rollback_migration(db, file : String, version : String)
    content = File.read(file)
    down_sql = extract_section(content, "Down")

    puts "  ◀ #{version}"
    db.transaction do |tx|
      tx.connection.exec(down_sql)
      tx.connection.exec(
        "DELETE FROM schema_migrations WHERE version = $1",
        version
      )
    end
    puts "    ✓ Rolled back"
  rescue ex
    puts "    ✗ Failed: #{ex.message}"
    raise ex
  end

  private def extract_section(content : String, section : String) : String
    pattern = /-- \+micrate #{section}(.*?)(?=-- \+micrate |\z)/ms
    match = content.match(pattern)
    match ? match[1].strip : ""
  end

  private def find_migration_file(version : String) : String
    Dir.glob("#{MIGRATIONS_DIR}/#{version}*.sql").first? ||
      raise "Migration file not found: #{version}"
  end
end

# ใช้งาน
runner = MigrationRunner.new(ENV["DATABASE_URL"])

case ARGV[0]?
when "up"    then runner.run!
when "down"  then runner.rollback!(ARGV[1]?.try(&.to_i) || 1)
when "status" then runner.status
else
  puts "Usage: crystal run src/migration_runner.cr -- [up|down [N]|status]"
end
```

## Data Migrations

บางครั้งต้องการ migrate ข้อมูลด้วย ไม่ใช่แค่ schema:

```sql
-- db/migrations/20241020100000_migrate_user_roles.sql

-- +micrate Up
-- เพิ่ม column ใหม่
ALTER TABLE users ADD COLUMN role_id INTEGER;

-- ย้ายข้อมูลจาก role string เป็น role_id
UPDATE users SET role_id = CASE
  WHEN role = 'admin' THEN 1
  WHEN role = 'moderator' THEN 2
  ELSE 3
END;

-- เพิ่ม NOT NULL constraint หลังจากข้อมูลพร้อม
ALTER TABLE users ALTER COLUMN role_id SET NOT NULL;

-- เพิ่ม Foreign key
ALTER TABLE users
  ADD CONSTRAINT fk_users_role
  FOREIGN KEY (role_id) REFERENCES roles(id);

-- +micrate Down
ALTER TABLE users DROP CONSTRAINT IF EXISTS fk_users_role;
ALTER TABLE users DROP COLUMN IF EXISTS role_id;
```

## Testing Migrations

```crystal
# spec/migration_spec.cr
require "spec"
require "db"
require "pg"

describe "Database Migrations" do
  let(db_url) { ENV["TEST_DATABASE_URL"] }

  before_each do
    DB.open(db_url) do |db|
      db.exec("DROP SCHEMA public CASCADE")
      db.exec("CREATE SCHEMA public")
    end
  end

  it "creates users table with correct columns" do
    MigrationRunner.new(db_url).run!

    DB.open(db_url) do |db|
      # ตรวจสอบ columns
      columns = db.query_all(
        "SELECT column_name, data_type FROM information_schema.columns WHERE table_name = 'users'",
        as: {column_name: String, data_type: String}
      )

      column_names = columns.map(&.[:column_name])
      column_names.should contain("email")
      column_names.should contain("username")
      column_names.should contain("created_at")

      # ตรวจสอบ constraints
      indexes = db.query_all(
        "SELECT indexname FROM pg_indexes WHERE tablename = 'users'",
        as: String
      )
      indexes.should contain("idx_users_email")
    end
  end

  it "can rollback migrations" do
    runner = MigrationRunner.new(db_url)
    runner.run!
    runner.rollback!(1)

    DB.open(db_url) do |db|
      table_exists = db.query_one?(
        "SELECT COUNT(*) FROM information_schema.tables WHERE table_name = 'users'",
        as: Int64
      )
      table_exists.should eq(0_i64)
    end
  end
end
```

## แบบฝึกหัด

1. สร้าง migration ชุดสำหรับระบบ e-commerce: users, products, orders, order_items พร้อม foreign keys และ indexes
2. เขียน migration สำหรับเพิ่ม `soft_delete` ให้ทุก table (เพิ่ม `deleted_at` column)
3. สร้าง data migration ที่แปลง phone number format จาก `0812345678` เป็น `+66812345678`
4. เขียน test สำหรับ migration ที่ตรวจสอบว่า up และ down ทำงานถูกต้อง

## สรุป

Database Migrations ใน Crystal:
- **ไฟล์ .sql**: ชัดเจน ใช้ SQL ตรงๆ กับ `-- +micrate Up/Down` markers
- **Version tracking**: เก็บใน `schema_migrations` table
- **Up/Down**: สร้างทั้ง migration และ rollback เสมอ
- **Idempotent**: ใช้ `IF EXISTS` และ `IF NOT EXISTS`
- **Zero-downtime**: เพิ่ม column ที่ nullable ก่อน แล้วค่อยเพิ่ม constraint
- **Testing**: test migration ใน isolated environment

Migrations ช่วยให้ทีมทำงานร่วมกันได้โดยไม่มีปัญหา schema mismatch
