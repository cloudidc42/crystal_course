# Part 152: Crystal กับ MySQL/MariaDB

## บทนำ

MySQL และ MariaDB เป็นฐานข้อมูลเชิงสัมพันธ์ที่ได้รับความนิยมสูงสุดในโลก Crystal สามารถเชื่อมต่อกับฐานข้อมูลเหล่านี้ได้อย่างมีประสิทธิภาพผ่าน shard ที่ชื่อว่า `crystal-mysql` ในบทนี้เราจะเรียนรู้การใช้งานตั้งแต่การติดตั้ง ไปจนถึงการทำงานขั้นสูงอย่าง Connection Pooling และ Transactions

## การติดตั้ง crystal-mysql Shard

### เพิ่ม Dependency ใน shard.yml

```yaml
# shard.yml
name: my_mysql_app
version: 1.0.0

dependencies:
  mysql:
    github: crystal-lang/crystal-mysql
    version: ~> 0.16.0
  
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
```

### ติดตั้ง Shard

```bash
shards install
```

### โครงสร้างโปรเจกต์

```
my_mysql_app/
├── shard.yml
├── shard.lock
├── src/
│   ├── main.cr
│   ├── database/
│   │   ├── connection.cr
│   │   ├── migrations/
│   │   │   ├── 001_create_users.cr
│   │   │   └── 002_create_posts.cr
│   │   └── models/
│   │       ├── user.cr
│   │       └── post.cr
│   └── repositories/
│       ├── user_repository.cr
│       └── post_repository.cr
└── spec/
    └── spec_helper.cr
```

## การเชื่อมต่อฐานข้อมูล

### การสร้าง Connection พื้นฐาน

```crystal
# src/database/connection.cr
require "mysql"
require "db"

module Database
  # สร้าง connection string
  DATABASE_URL = ENV.fetch("DATABASE_URL", "mysql://root:password@localhost:3306/myapp_db")
  
  # สร้าง connection pool
  def self.pool
    @@pool ||= DB.open(DATABASE_URL)
  end
  
  # ทดสอบการเชื่อมต่อ
  def self.test_connection
    pool.query_one("SELECT 1", as: Int32)
    puts "เชื่อมต่อฐานข้อมูลสำเร็จ!"
    true
  rescue ex
    puts "เชื่อมต่อฐานข้อมูลล้มเหลว: #{ex.message}"
    false
  end
  
  # ปิด connection pool
  def self.close
    @@pool.try(&.close)
  end
end
```

### Configuration ด้วย Environment Variables

```crystal
# src/config/database.cr
module Config
  module Database
    # MySQL Connection Settings
    HOST     = ENV.fetch("DB_HOST", "localhost")
    PORT     = ENV.fetch("DB_PORT", "3306").to_i
    NAME     = ENV.fetch("DB_NAME", "myapp_development")
    USER     = ENV.fetch("DB_USER", "root")
    PASSWORD = ENV.fetch("DB_PASSWORD", "")
    
    # Connection Pool Settings
    POOL_SIZE     = ENV.fetch("DB_POOL_SIZE", "10").to_i
    IDLE_TIMEOUT  = ENV.fetch("DB_IDLE_TIMEOUT", "300").to_f
    CHECKOUT_TIME = ENV.fetch("DB_CHECKOUT_TIMEOUT", "5.0").to_f
    
    def self.connection_string
      "mysql://#{USER}:#{PASSWORD}@#{HOST}:#{PORT}/#{NAME}?" \
      "max_pool_size=#{POOL_SIZE}&" \
      "initial_pool_size=2&" \
      "max_idle_pool_size=#{POOL_SIZE / 2}&" \
      "checkout_timeout=#{CHECKOUT_TIME}&" \
      "retry_attempts=3&" \
      "retry_delay=1.0"
    end
  end
end
```

## CRUD Operations

### การสร้าง Schema

```crystal
# src/database/schema.cr
require "../database/connection"

module Schema
  def self.create_tables
    db = Database.pool
    
    # สร้างตาราง users
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS users (
        id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
        email      VARCHAR(255) NOT NULL UNIQUE,
        name       VARCHAR(100) NOT NULL,
        age        INT,
        role       ENUM('admin', 'user', 'moderator') DEFAULT 'user',
        active     BOOLEAN DEFAULT TRUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        deleted_at TIMESTAMP NULL,
        INDEX idx_email (email),
        INDEX idx_role (role),
        INDEX idx_created_at (created_at)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    SQL
    
    # สร้างตาราง posts
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS posts (
        id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
        user_id    BIGINT UNSIGNED NOT NULL,
        title      VARCHAR(500) NOT NULL,
        content    TEXT,
        slug       VARCHAR(500) UNIQUE,
        status     ENUM('draft', 'published', 'archived') DEFAULT 'draft',
        views      INT DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
        INDEX idx_user_id (user_id),
        INDEX idx_status (status),
        FULLTEXT INDEX idx_content (title, content)
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
    SQL
    
    puts "สร้างตารางสำเร็จ!"
  end
end
```

### Model: User

```crystal
# src/models/user.cr
require "json"
require "time"

struct User
  include JSON::Serializable
  
  property id : UInt64?
  property email : String
  property name : String
  property age : Int32?
  property role : String
  property active : Bool
  property created_at : Time?
  property updated_at : Time?
  
  def initialize(
    @email : String,
    @name : String,
    @age : Int32? = nil,
    @role : String = "user",
    @active : Bool = true,
    @id : UInt64? = nil,
    @created_at : Time? = nil,
    @updated_at : Time? = nil
  )
  end
  
  def admin?
    role == "admin"
  end
  
  def active?
    active
  end
  
  def to_s
    "User(id=#{id}, email=#{email}, name=#{name})"
  end
end
```

### Repository: UserRepository

```crystal
# src/repositories/user_repository.cr
require "../models/user"
require "../database/connection"

class UserRepository
  include DB
  
  def initialize(@db : DB::Database = Database.pool)
  end
  
  # CREATE - สร้างผู้ใช้ใหม่
  def create(user : User) : User
    result = @db.exec(
      "INSERT INTO users (email, name, age, role, active) VALUES (?, ?, ?, ?, ?)",
      user.email, user.name, user.age, user.role, user.active
    )
    
    new_id = result.last_insert_id.to_u64
    user.id = new_id
    user.created_at = Time.utc
    user.updated_at = Time.utc
    user
  rescue ex : DB::Error
    if ex.message.try(&.includes?("Duplicate entry"))
      raise "อีเมล #{user.email} มีอยู่แล้วในระบบ"
    end
    raise ex
  end
  
  # READ - ดึงข้อมูลผู้ใช้ทั้งหมด
  def find_all(limit : Int32 = 100, offset : Int32 = 0) : Array(User)
    users = [] of User
    
    @db.query(
      "SELECT id, email, name, age, role, active, created_at, updated_at 
       FROM users 
       WHERE deleted_at IS NULL 
       ORDER BY created_at DESC 
       LIMIT ? OFFSET ?",
      limit, offset
    ) do |rs|
      rs.each do
        users << User.new(
          id: rs.read(UInt64),
          email: rs.read(String),
          name: rs.read(String),
          age: rs.read(Int32?),
          role: rs.read(String),
          active: rs.read(Bool),
          created_at: rs.read(Time?),
          updated_at: rs.read(Time?)
        )
      end
    end
    
    users
  end
  
  # READ - ค้นหาผู้ใช้ด้วย ID
  def find_by_id(id : UInt64) : User?
    @db.query_one?(
      "SELECT id, email, name, age, role, active, created_at, updated_at 
       FROM users 
       WHERE id = ? AND deleted_at IS NULL",
      id
    ) do |rs|
      User.new(
        id: rs.read(UInt64),
        email: rs.read(String),
        name: rs.read(String),
        age: rs.read(Int32?),
        role: rs.read(String),
        active: rs.read(Bool),
        created_at: rs.read(Time?),
        updated_at: rs.read(Time?)
      )
    end
  end
  
  # READ - ค้นหาผู้ใช้ด้วย Email
  def find_by_email(email : String) : User?
    @db.query_one?(
      "SELECT id, email, name, age, role, active, created_at, updated_at 
       FROM users 
       WHERE email = ? AND deleted_at IS NULL",
      email
    ) do |rs|
      User.new(
        id: rs.read(UInt64),
        email: rs.read(String),
        name: rs.read(String),
        age: rs.read(Int32?),
        role: rs.read(String),
        active: rs.read(Bool),
        created_at: rs.read(Time?),
        updated_at: rs.read(Time?)
      )
    end
  end
  
  # UPDATE - อัปเดตข้อมูลผู้ใช้
  def update(user : User) : Bool
    raise "ไม่มี ID ผู้ใช้" unless user.id
    
    result = @db.exec(
      "UPDATE users 
       SET email = ?, name = ?, age = ?, role = ?, active = ?, updated_at = NOW() 
       WHERE id = ? AND deleted_at IS NULL",
      user.email, user.name, user.age, user.role, user.active, user.id
    )
    
    result.rows_affected > 0
  end
  
  # DELETE - ลบผู้ใช้ (Soft Delete)
  def delete(id : UInt64) : Bool
    result = @db.exec(
      "UPDATE users SET deleted_at = NOW() WHERE id = ? AND deleted_at IS NULL",
      id
    )
    result.rows_affected > 0
  end
  
  # DELETE - ลบผู้ใช้จริงๆ (Hard Delete)
  def hard_delete(id : UInt64) : Bool
    result = @db.exec("DELETE FROM users WHERE id = ?", id)
    result.rows_affected > 0
  end
  
  # COUNT - นับจำนวนผู้ใช้
  def count : Int64
    @db.query_one("SELECT COUNT(*) FROM users WHERE deleted_at IS NULL", as: Int64)
  end
  
  # SEARCH - ค้นหาผู้ใช้
  def search(query : String, limit : Int32 = 20) : Array(User)
    users = [] of User
    search_term = "%#{query}%"
    
    @db.query(
      "SELECT id, email, name, age, role, active, created_at, updated_at 
       FROM users 
       WHERE (name LIKE ? OR email LIKE ?) AND deleted_at IS NULL 
       ORDER BY name 
       LIMIT ?",
      search_term, search_term, limit
    ) do |rs|
      rs.each do
        users << User.new(
          id: rs.read(UInt64),
          email: rs.read(String),
          name: rs.read(String),
          age: rs.read(Int32?),
          role: rs.read(String),
          active: rs.read(Bool),
          created_at: rs.read(Time?),
          updated_at: rs.read(Time?)
        )
      end
    end
    
    users
  end
  
  # BULK INSERT - เพิ่มข้อมูลหลายรายการพร้อมกัน
  def bulk_create(users : Array(User)) : Int32
    return 0 if users.empty?
    
    placeholders = users.map { "(?, ?, ?, ?, ?)" }.join(", ")
    values = users.flat_map { |u| [u.email, u.name, u.age, u.role, u.active] }
    
    result = @db.exec(
      "INSERT INTO users (email, name, age, role, active) VALUES #{placeholders}",
      *values
    )
    
    result.rows_affected.to_i
  end
end
```

## Connection Pooling

### การตั้งค่า Connection Pool

```crystal
# src/database/pool_manager.cr
require "db"
require "mysql"

class PoolManager
  # Singleton pattern สำหรับ connection pool
  @@instance : DB::Database? = nil
  @@mutex = Mutex.new
  
  def self.instance : DB::Database
    @@mutex.synchronize do
      @@instance ||= create_pool
    end
  end
  
  private def self.create_pool : DB::Database
    # Connection string พร้อม pool settings
    url = build_connection_string
    
    db = DB.open(url)
    
    # ทดสอบการเชื่อมต่อ
    db.query_one("SELECT VERSION()", as: String).tap do |version|
      puts "เชื่อมต่อ MySQL #{version} สำเร็จ"
      puts "Pool size: #{Config::Database::POOL_SIZE}"
    end
    
    # Register shutdown hook
    at_exit { db.close }
    
    db
  end
  
  private def self.build_connection_string : String
    params = URI::Params.build do |p|
      p.add("max_pool_size", Config::Database::POOL_SIZE.to_s)
      p.add("initial_pool_size", "2")
      p.add("max_idle_pool_size", (Config::Database::POOL_SIZE / 2).to_s)
      p.add("checkout_timeout", Config::Database::CHECKOUT_TIME.to_s)
      p.add("retry_attempts", "3")
      p.add("retry_delay", "0.5")
    end
    
    "mysql://#{Config::Database::USER}:#{Config::Database::PASSWORD}" \
    "@#{Config::Database::HOST}:#{Config::Database::PORT}" \
    "/#{Config::Database::NAME}?#{params}"
  end
  
  # ตรวจสอบสถานะ pool
  def self.stats
    db = instance
    {
      pool_size: Config::Database::POOL_SIZE,
      database: Config::Database::NAME,
      host: Config::Database::HOST
    }
  end
end
```

### การใช้งาน Pool ใน Concurrent Environment

```crystal
# src/examples/concurrent_queries.cr
require "wait_group"

class ConcurrentQueryExample
  def self.run
    db = PoolManager.instance
    results = [] of String
    mutex = Mutex.new
    
    # สร้าง concurrent queries
    wg = WaitGroup.new(10)
    
    10.times do |i|
      spawn do
        begin
          # แต่ละ fiber จะได้ connection จาก pool
          name = db.query_one(
            "SELECT name FROM users WHERE id = ?",
            i + 1,
            as: String
          )
          mutex.synchronize { results << "Worker #{i}: #{name}" }
        rescue ex
          mutex.synchronize { results << "Worker #{i}: Error - #{ex.message}" }
        ensure
          wg.done
        end
      end
    end
    
    wg.wait
    results.each { |r| puts r }
  end
end
```

## Transactions

### การใช้งาน Transaction พื้นฐาน

```crystal
# src/database/transaction_example.cr
class TransactionExample
  def self.transfer_credits(
    db : DB::Database,
    from_user_id : UInt64,
    to_user_id : UInt64,
    amount : Float64
  ) : Bool
    db.transaction do |tx|
      cnn = tx.connection
      
      # ตรวจสอบยอดเงินผู้ส่ง
      balance = cnn.query_one(
        "SELECT balance FROM accounts WHERE user_id = ? FOR UPDATE",
        from_user_id,
        as: Float64
      )
      
      if balance < amount
        raise "ยอดเงินไม่เพียงพอ: มี #{balance}, ต้องการ #{amount}"
      end
      
      # หักเงินผู้ส่ง
      cnn.exec(
        "UPDATE accounts SET balance = balance - ? WHERE user_id = ?",
        amount, from_user_id
      )
      
      # เพิ่มเงินผู้รับ
      cnn.exec(
        "UPDATE accounts SET balance = balance + ? WHERE user_id = ?",
        amount, to_user_id
      )
      
      # บันทึกประวัติ
      cnn.exec(
        "INSERT INTO transactions (from_user_id, to_user_id, amount, status) 
         VALUES (?, ?, ?, 'completed')",
        from_user_id, to_user_id, amount
      )
      
      puts "โอนเงิน #{amount} สำเร็จ"
      true
    end
  rescue ex
    puts "โอนเงินล้มเหลว: #{ex.message}"
    false
  end
end
```

### Nested Transaction และ Savepoints

```crystal
# src/database/savepoint_example.cr
class SavepointExample
  def self.create_order_with_items(
    db : DB::Database,
    user_id : UInt64,
    items : Array(NamedTuple(product_id: UInt64, quantity: Int32, price: Float64))
  ) : UInt64?
    db.transaction do |tx|
      cnn = tx.connection
      
      # สร้าง Order หลัก
      order_result = cnn.exec(
        "INSERT INTO orders (user_id, status, total) VALUES (?, 'pending', 0)",
        user_id
      )
      order_id = order_result.last_insert_id.to_u64
      total = 0.0
      
      items.each_with_index do |item, index|
        begin
          # ใช้ Savepoint สำหรับแต่ละ item
          cnn.exec("SAVEPOINT item_#{index}")
          
          # ตรวจสอบสต็อก
          stock = cnn.query_one(
            "SELECT stock FROM products WHERE id = ? FOR UPDATE",
            item[:product_id],
            as: Int32
          )
          
          if stock < item[:quantity]
            puts "สินค้า #{item[:product_id]} สต็อกไม่พอ ข้ามรายการนี้"
            cnn.exec("ROLLBACK TO SAVEPOINT item_#{index}")
            next
          end
          
          # เพิ่มรายการสั่งซื้อ
          subtotal = item[:quantity] * item[:price]
          cnn.exec(
            "INSERT INTO order_items (order_id, product_id, quantity, price, subtotal) 
             VALUES (?, ?, ?, ?, ?)",
            order_id, item[:product_id], item[:quantity], item[:price], subtotal
          )
          
          # อัปเดตสต็อก
          cnn.exec(
            "UPDATE products SET stock = stock - ? WHERE id = ?",
            item[:quantity], item[:product_id]
          )
          
          total += subtotal
          cnn.exec("RELEASE SAVEPOINT item_#{index}")
          
        rescue ex
          puts "ข้ามรายการ #{index}: #{ex.message}"
          cnn.exec("ROLLBACK TO SAVEPOINT item_#{index}")
        end
      end
      
      # อัปเดตยอดรวม
      cnn.exec(
        "UPDATE orders SET total = ?, status = 'confirmed' WHERE id = ?",
        total, order_id
      )
      
      order_id
    end
  end
end
```

## Prepared Statements

### การสร้างและใช้ Prepared Statements

```crystal
# src/database/prepared_statements.cr
class PreparedStatementExample
  def initialize(@db : DB::Database)
  end
  
  # ใช้ prepared statement สำหรับการ query ที่ทำบ่อย
  def find_users_by_role(role : String) : Array(NamedTuple(id: UInt64, name: String, email: String))
    results = [] of NamedTuple(id: UInt64, name: String, email: String)
    
    # Crystal-DB ใช้ prepared statements โดยอัตโนมัติ
    @db.query(
      "SELECT id, name, email FROM users WHERE role = ? AND active = TRUE ORDER BY name",
      role
    ) do |rs|
      rs.each do
        results << {
          id: rs.read(UInt64),
          name: rs.read(String),
          email: rs.read(String)
        }
      end
    end
    
    results
  end
  
  # Batch operations ด้วย prepared statement
  def batch_update_status(user_ids : Array(UInt64), new_status : Bool)
    return if user_ids.empty?
    
    # ใช้ transaction สำหรับ batch operation
    @db.transaction do |tx|
      cnn = tx.connection
      
      user_ids.each do |id|
        cnn.exec(
          "UPDATE users SET active = ?, updated_at = NOW() WHERE id = ?",
          new_status, id
        )
      end
    end
    
    puts "อัปเดตสถานะ #{user_ids.size} ผู้ใช้สำเร็จ"
  end
  
  # Parameterized query ป้องกัน SQL Injection
  def safe_search(term : String) : Array(String)
    results = [] of String
    
    # ใช้ ? เป็น placeholder แทนการ interpolate string โดยตรง
    @db.query(
      "SELECT name FROM users WHERE name LIKE ? AND deleted_at IS NULL",
      "%#{term.gsub('%', "\\%").gsub('_', "\\_")}%"
    ) do |rs|
      rs.each { results << rs.read(String) }
    end
    
    results
  end
end
```

## Migration Strategies

### Migration System ง่ายๆ

```crystal
# src/database/migrations/migration.cr
abstract class Migration
  abstract def up(db : DB::Database)
  abstract def down(db : DB::Database)
  abstract def version : String
  abstract def description : String
end

# src/database/migrations/001_create_users.cr
class CreateUsersMigration < Migration
  def version : String
    "20240101000001"
  end
  
  def description : String
    "สร้างตาราง users"
  end
  
  def up(db : DB::Database)
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS users (
        id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
        email      VARCHAR(255) NOT NULL UNIQUE,
        name       VARCHAR(100) NOT NULL,
        password_hash VARCHAR(255),
        age        INT,
        role       ENUM('admin', 'user', 'moderator') DEFAULT 'user',
        active     BOOLEAN DEFAULT TRUE,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        deleted_at TIMESTAMP NULL
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
    SQL
    
    db.exec "CREATE INDEX idx_email ON users (email)"
    db.exec "CREATE INDEX idx_role ON users (role)"
    db.exec "CREATE INDEX idx_active ON users (active)"
  end
  
  def down(db : DB::Database)
    db.exec "DROP TABLE IF EXISTS users"
  end
end

# src/database/migrations/002_create_posts.cr
class CreatePostsMigration < Migration
  def version : String
    "20240101000002"
  end
  
  def description : String
    "สร้างตาราง posts"
  end
  
  def up(db : DB::Database)
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS posts (
        id         BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
        user_id    BIGINT UNSIGNED NOT NULL,
        title      VARCHAR(500) NOT NULL,
        content    TEXT,
        slug       VARCHAR(500),
        status     ENUM('draft', 'published', 'archived') DEFAULT 'draft',
        views      INT DEFAULT 0,
        created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
        updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
        FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
      ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci
    SQL
    
    db.exec "CREATE INDEX idx_user_id ON posts (user_id)"
    db.exec "CREATE INDEX idx_status ON posts (status)"
    db.exec "CREATE UNIQUE INDEX idx_slug ON posts (slug)"
    db.exec "ALTER TABLE posts ADD FULLTEXT INDEX idx_fulltext (title, content)"
  end
  
  def down(db : DB::Database)
    db.exec "DROP TABLE IF EXISTS posts"
  end
end
```

### Migration Runner

```crystal
# src/database/migrator.cr
class Migrator
  def initialize(@db : DB::Database)
    ensure_migrations_table
  end
  
  def migrations : Array(Migration)
    [
      CreateUsersMigration.new,
      CreatePostsMigration.new,
      # เพิ่ม migrations ใหม่ที่นี่
    ] of Migration
  end
  
  def run!
    pending = pending_migrations
    
    if pending.empty?
      puts "ไม่มี migration ที่ต้องรัน"
      return
    end
    
    puts "พบ #{pending.size} migrations ที่ต้องรัน"
    
    pending.each do |migration|
      puts "กำลังรัน: #{migration.version} - #{migration.description}"
      
      @db.transaction do |tx|
        migration.up(tx.connection)
        tx.connection.exec(
          "INSERT INTO schema_migrations (version, description) VALUES (?, ?)",
          migration.version, migration.description
        )
      end
      
      puts "  สำเร็จ: #{migration.version}"
    end
    
    puts "Migration เสร็จสมบูรณ์!"
  end
  
  def rollback!(steps : Int32 = 1)
    ran = ran_migrations.last(steps)
    
    ran.reverse_each do |version|
      migration = migrations.find { |m| m.version == version }
      next unless migration
      
      puts "ย้อนกลับ: #{version} - #{migration.description}"
      
      @db.transaction do |tx|
        migration.down(tx.connection)
        tx.connection.exec(
          "DELETE FROM schema_migrations WHERE version = ?",
          version
        )
      end
      
      puts "  สำเร็จ: #{version}"
    end
  end
  
  def status
    puts "\nสถานะ Migrations:"
    puts "-" * 60
    printf("%-20s %-30s %-10s\n", "Version", "Description", "Status")
    puts "-" * 60
    
    ran = ran_migrations
    migrations.each do |m|
      status = ran.includes?(m.version) ? "รันแล้ว" : "รอรัน"
      printf("%-20s %-30s %-10s\n", m.version, m.description, status)
    end
  end
  
  private def ensure_migrations_table
    @db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS schema_migrations (
        version     VARCHAR(20) PRIMARY KEY,
        description VARCHAR(255),
        ran_at      TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    SQL
  end
  
  private def ran_migrations : Array(String)
    versions = [] of String
    @db.query("SELECT version FROM schema_migrations ORDER BY version") do |rs|
      rs.each { versions << rs.read(String) }
    end
    versions
  end
  
  private def pending_migrations : Array(Migration)
    ran = ran_migrations
    migrations.reject { |m| ran.includes?(m.version) }
  end
end
```

## ตัวอย่าง Real-World: ระบบ Blog

### Blog Service

```crystal
# src/services/blog_service.cr
class BlogService
  def initialize(@db : DB::Database)
    @user_repo = UserRepository.new(@db)
    @post_repo = PostRepository.new(@db)
  end
  
  # สร้างโพสต์ใหม่พร้อมการตรวจสอบ
  def create_post(
    user_id : UInt64,
    title : String,
    content : String,
    status : String = "draft"
  ) : NamedTuple(success: Bool, post_id: UInt64?, error: String?)
    # ตรวจสอบผู้ใช้
    user = @user_repo.find_by_id(user_id)
    return {success: false, post_id: nil, error: "ไม่พบผู้ใช้"} unless user
    return {success: false, post_id: nil, error: "บัญชีถูกระงับ"} unless user.active?
    
    # สร้าง slug จาก title
    slug = generate_slug(title)
    
    # ตรวจสอบว่า slug ซ้ำไหม
    slug = ensure_unique_slug(slug)
    
    @db.transaction do |tx|
      cnn = tx.connection
      
      result = cnn.exec(
        "INSERT INTO posts (user_id, title, content, slug, status) VALUES (?, ?, ?, ?, ?)",
        user_id, title, content, slug, status
      )
      
      post_id = result.last_insert_id.to_u64
      
      # บันทึก activity log
      cnn.exec(
        "INSERT INTO activity_logs (user_id, action, resource_type, resource_id) 
         VALUES (?, 'create_post', 'post', ?)",
        user_id, post_id
      )
      
      {success: true, post_id: post_id, error: nil}
    end
  rescue ex
    {success: false, post_id: nil, error: ex.message}
  end
  
  # ดึงโพสต์พร้อมข้อมูลผู้เขียน
  def get_published_posts(
    page : Int32 = 1,
    per_page : Int32 = 10
  ) : Array(NamedTuple(
    id: UInt64,
    title: String,
    slug: String,
    author_name: String,
    views: Int32,
    created_at: Time
  ))
    offset = (page - 1) * per_page
    results = [] of NamedTuple(
      id: UInt64,
      title: String,
      slug: String,
      author_name: String,
      views: Int32,
      created_at: Time
    )
    
    @db.query(
      "SELECT p.id, p.title, p.slug, u.name as author_name, p.views, p.created_at
       FROM posts p
       JOIN users u ON p.user_id = u.id
       WHERE p.status = 'published' AND u.deleted_at IS NULL
       ORDER BY p.created_at DESC
       LIMIT ? OFFSET ?",
      per_page, offset
    ) do |rs|
      rs.each do
        results << {
          id: rs.read(UInt64),
          title: rs.read(String),
          slug: rs.read(String),
          author_name: rs.read(String),
          views: rs.read(Int32),
          created_at: rs.read(Time)
        }
      end
    end
    
    results
  end
  
  # เพิ่มจำนวนการดู
  def increment_views(post_id : UInt64)
    @db.exec(
      "UPDATE posts SET views = views + 1 WHERE id = ?",
      post_id
    )
  end
  
  # สถิติ Blog
  def statistics : Hash(String, Int64)
    {
      "total_users" => @db.query_one("SELECT COUNT(*) FROM users WHERE deleted_at IS NULL", as: Int64),
      "total_posts" => @db.query_one("SELECT COUNT(*) FROM posts", as: Int64),
      "published_posts" => @db.query_one("SELECT COUNT(*) FROM posts WHERE status = 'published'", as: Int64),
      "total_views" => @db.query_one("SELECT COALESCE(SUM(views), 0) FROM posts", as: Int64)
    }
  end
  
  private def generate_slug(title : String) : String
    title
      .downcase
      .gsub(/[^a-z0-9\s-]/, "")
      .gsub(/\s+/, "-")
      .gsub(/-+/, "-")
      .strip("-")
  end
  
  private def ensure_unique_slug(slug : String) : String
    count = @db.query_one(
      "SELECT COUNT(*) FROM posts WHERE slug LIKE ?",
      "#{slug}%",
      as: Int64
    )
    
    count == 0 ? slug : "#{slug}-#{count + 1}"
  end
end
```

### main.cr ทดสอบการทำงาน

```crystal
# src/main.cr
require "mysql"
require "db"
require "./config/database"
require "./database/connection"
require "./database/migrator"
require "./repositories/user_repository"
require "./services/blog_service"

# รัน migrations
db = DB.open(Config::Database.connection_string)
migrator = Migrator.new(db)
migrator.run!
migrator.status

# ทดสอบ CRUD
puts "\n=== ทดสอบ User Repository ==="
user_repo = UserRepository.new(db)

# สร้างผู้ใช้
user1 = User.new(email: "admin@example.com", name: "ผู้ดูแลระบบ", role: "admin")
user1 = user_repo.create(user1)
puts "สร้างผู้ใช้: #{user1}"

user2 = User.new(email: "john@example.com", name: "John Doe", age: 25)
user2 = user_repo.create(user2)
puts "สร้างผู้ใช้: #{user2}"

# ดึงข้อมูล
all_users = user_repo.find_all
puts "\nผู้ใช้ทั้งหมด: #{all_users.size} คน"
all_users.each { |u| puts "  - #{u.name} (#{u.email})" }

# อัปเดต
user2.name = "John Updated"
user_repo.update(user2)
puts "\nอัปเดตชื่อเป็น: #{user2.name}"

# ทดสอบ Blog Service
puts "\n=== ทดสอบ Blog Service ==="
blog = BlogService.new(db)

result = blog.create_post(
  user_id: user1.id.not_nil!,
  title: "บทความแรกของฉัน",
  content: "เนื้อหาบทความ...",
  status: "published"
)
puts "สร้างโพสต์: #{result}"

stats = blog.statistics
puts "\nสถิติ Blog:"
stats.each { |k, v| puts "  #{k}: #{v}" }

db.close
```

## สรุป

ในบทนี้เราได้เรียนรู้:
- การติดตั้งและตั้งค่า `crystal-mysql` shard
- การทำ CRUD operations พื้นฐาน
- การจัดการ Connection Pool อย่างมีประสิทธิภาพ
- การใช้ Transactions และ Savepoints
- การสร้าง Prepared Statements ที่ปลอดภัยจาก SQL Injection
- ระบบ Migration สำหรับจัดการ Schema
- ตัวอย่าง Blog Service จริงๆ

## ขั้นตอนต่อไป

ใน **Part 153** เราจะเรียนรู้การใช้งาน Crystal กับ **MongoDB** ซึ่งเป็นฐานข้อมูล NoSQL ที่ยืดหยุ่นกว่า เหมาะสำหรับข้อมูลที่มีโครงสร้างไม่แน่นอน
