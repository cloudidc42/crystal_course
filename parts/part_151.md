# Part 151: Database Basics (crystal-db) - พื้นฐาน Database ใน Crystal

## บทนำ

`crystal-db` เป็น database abstraction layer สำหรับ Crystal รองรับ PostgreSQL, MySQL, SQLite และอื่นๆ ผ่าน adapter

## การติดตั้ง

```yaml
# shard.yml
dependencies:
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
  pg:
    github: will/crystal-pg   # PostgreSQL
  mysql:
    github: crystal-lang/crystal-mysql  # MySQL
  sqlite3:
    github: crystal-lang/crystal-sqlite3  # SQLite
```

## การเชื่อมต่อ

```crystal
require "db"
require "pg"

# เปิด connection เดียว
DB.open("postgres://user:pass@localhost/mydb") do |db|
  # ใช้ db ใน block นี้
  result = db.query_one("SELECT 1+1", as: Int32)
  puts result  # => 2
end

# Connection pool (แนะนำสำหรับ web apps)
db = DB.open("postgres://user:pass@localhost/mydb?max_pool_size=25&initial_pool_size=5")
# ใช้ db ตลอด lifetime ของ app
```

## Connection String Format

```
postgres://[user[:password]@][host][:port][/dbname][?option=value...]

# ตัวอย่าง
postgres://admin:secret@localhost:5432/myapp
postgres://admin:secret@localhost/myapp?application_name=myapp&max_pool_size=20

# SQLite
sqlite3://./data.db
sqlite3://:memory:

# MySQL
mysql://user:pass@localhost:3306/mydb
```

## Query Operations

```crystal
require "db"
require "pg"

DB.open("postgres://user:pass@localhost/mydb") do |db|
  
  # ===== query_one =====
  # ดึง 1 row (raises หากไม่พบ หรือพบมากกว่า 1)
  user = db.query_one(
    "SELECT id, name, email FROM users WHERE id = $1",
    1,
    as: {id: Int64, name: String, email: String}
  )
  puts user[:name]
  
  # query_one? - returns nil ถ้าไม่พบ
  user2 = db.query_one?(
    "SELECT * FROM users WHERE email = $1",
    "test@example.com",
    as: {id: Int64, name: String}
  )
  puts user2.try(&.[:name]) || "Not found"
  
  # ===== query_all =====
  # ดึงหลาย rows
  users = db.query_all(
    "SELECT id, name, email FROM users ORDER BY name",
    as: {id: Int64, name: String, email: String}
  )
  users.each { |u| puts "#{u[:id]}: #{u[:name]}" }
  
  # ===== query =====
  # Manual iteration
  db.query("SELECT id, name FROM users") do |rs|
    rs.each do
      id = rs.read(Int64)
      name = rs.read(String)
      puts "#{id}: #{name}"
    end
  end
  
  # ===== scalar =====
  # ดึง single value
  count = db.query_one("SELECT COUNT(*) FROM users", as: Int64)
  puts "Users: #{count}"
  
  max_id = db.query_one("SELECT MAX(id) FROM users", as: Int64?)
  puts "Max ID: #{max_id || 0}"
  
end
```

## Exec Operations

```crystal
require "db"
require "pg"

DB.open("postgres://user:pass@localhost/mydb") do |db|
  
  # INSERT
  result = db.exec(
    "INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW())",
    "สมชาย", "somchai@example.com"
  )
  puts "Rows affected: #{result.rows_affected}"
  puts "Last insert ID: #{result.last_insert_id}"
  
  # INSERT ... RETURNING (PostgreSQL)
  user = db.query_one(
    "INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW()) RETURNING id, name, email",
    "สมหญิง", "somying@example.com",
    as: {id: Int64, name: String, email: String}
  )
  puts "Created user ID: #{user[:id]}"
  
  # UPDATE
  result = db.exec(
    "UPDATE users SET name = $1 WHERE id = $2",
    "สมชาย ใจดี", 1
  )
  puts "Updated: #{result.rows_affected} rows"
  
  # DELETE
  result = db.exec("DELETE FROM users WHERE id = $1", 99)
  puts "Deleted: #{result.rows_affected} rows"
  
end
```

## DB::Serializable

```crystal
require "db"
require "pg"

# ใช้ DB::Serializable เพื่อ map columns เป็น struct
struct User
  include DB::Serializable
  
  property id : Int64
  property name : String
  property email : String
  property role : String
  property active : Bool
  property created_at : Time
  
  # Optional column ที่อาจเป็น nil
  property bio : String?
  property avatar_url : String?
end

struct UserStats
  include DB::Serializable
  
  @[DB::Field(key: "user_id")]  # ชื่อ column ต่างจาก property
  property id : Int64
  
  property name : String
  property post_count : Int64
  property comment_count : Int64
end

DB.open("postgres://user:pass@localhost/mydb") do |db|
  # ดึง users ทั้งหมด
  users = db.query_all("SELECT * FROM users WHERE active = true", as: User)
  users.each { |u| puts u.name }
  
  # ดึง user เดียว
  user = db.query_one?("SELECT * FROM users WHERE id = $1", 1, as: User)
  puts user.try(&.name) || "Not found"
  
  # ดึง stats ด้วย JOIN
  stats = db.query_all(<<-SQL, as: UserStats)
    SELECT u.id as user_id, u.name,
           COUNT(DISTINCT p.id) as post_count,
           COUNT(DISTINCT c.id) as comment_count
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    LEFT JOIN comments c ON c.user_id = u.id
    GROUP BY u.id, u.name
  SQL
end
```

## Transactions

```crystal
require "db"
require "pg"

DB.open("postgres://user:pass@localhost/mydb") do |db|
  
  # Transaction พื้นฐาน
  db.transaction do |tx|
    cnn = tx.connection
    
    # สร้าง user
    user_id = cnn.query_one(
      "INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id",
      "สมชาย", "test@example.com",
      as: Int64
    )
    
    # สร้าง profile
    cnn.exec(
      "INSERT INTO user_profiles (user_id, bio) VALUES ($1, $2)",
      user_id, "Hello World"
    )
    
    # ถ้าไม่มี exception จะ commit อัตโนมัติ
    puts "Transaction committed"
  end
  
  # Transaction พร้อม rollback
  db.transaction do |tx|
    cnn = tx.connection
    
    begin
      cnn.exec("UPDATE accounts SET balance = balance - $1 WHERE id = $2", 500, 1)
      cnn.exec("UPDATE accounts SET balance = balance + $1 WHERE id = $2", 500, 2)
      
      # ตรวจสอบ balance
      balance = cnn.query_one("SELECT balance FROM accounts WHERE id = 1", as: Int64)
      if balance < 0
        tx.rollback
        puts "Insufficient funds"
      end
    rescue ex
      tx.rollback
      puts "Transaction failed: #{ex.message}"
    end
  end
  
end
```

## Connection Pool

```crystal
require "db"
require "pg"

# Connection pool ด้วย query parameters
DATABASE_URL = "postgres://user:pass@localhost/mydb" +
               "?max_pool_size=25" +
               "&initial_pool_size=5" +
               "&max_idle_pool_size=10" +
               "&checkout_timeout=5.0" +
               "&retry_attempts=1" +
               "&retry_delay=0.2"

DB_POOL = DB.open(DATABASE_URL)

# ใช้ DB_POOL ทั่ว application
def find_user(id : Int64) : User?
  DB_POOL.query_one?("SELECT * FROM users WHERE id = $1", id, as: User)
end

def create_user(name : String, email : String) : User?
  DB_POOL.query_one?(
    "INSERT INTO users (name, email, created_at) VALUES ($1, $2, NOW()) RETURNING *",
    name, email,
    as: User
  )
end

# ปิด pool เมื่อเลิกใช้
at_exit { DB_POOL.close }
```

## Prepared Statements

```crystal
require "db"
require "pg"

DB.open("postgres://user:pass@localhost/mydb") do |db|
  
  # Prepared statement ช่วย optimize repeated queries
  db.using_connection do |cnn|
    # Prepare ครั้งเดียว
    stmt = cnn.prepared("SELECT * FROM users WHERE id = $1")
    
    # ใช้หลายครั้ง
    [1, 2, 3, 4, 5].each do |id|
      user = stmt.query_one?(id, as: {id: Int64, name: String}) rescue nil
      puts user.try(&.[:name]) || "Not found"
    end
  end
  
end
```

## Error Handling

```crystal
require "db"
require "pg"

def safe_query(db : DB::Database, id : Int64) : User?
  db.query_one?("SELECT * FROM users WHERE id = $1", id, as: User)
rescue DB::NoResultsError
  nil
rescue DB::Error => ex
  puts "Database error: #{ex.message}"
  nil
rescue ex
  puts "Unexpected error: #{ex.message}"
  nil
end

# ตรวจสอบ constraint violations
def create_user_safe(db : DB::Database, email : String, name : String) : {User?, String?}
  user = db.query_one?(
    "INSERT INTO users (email, name) VALUES ($1, $2) RETURNING *",
    email, name,
    as: User
  )
  {user, nil}
rescue PQ::PQError => ex
  if ex.message.try(&.includes?("unique constraint"))
    {nil, "Email already exists"}
  elsif ex.message.try(&.includes?("not-null constraint"))
    {nil, "Required field missing"}
  else
    {nil, "Database error: #{ex.message}"}
  end
rescue ex
  {nil, "Error: #{ex.message}"}
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Inventory System

```crystal
require "db"
require "pg"

struct Product
  include DB::Serializable
  
  property id : Int64
  property name : String
  property sku : String
  property price : Float64
  property stock : Int32
  property category : String
  property created_at : Time
end

class InventoryManager
  def initialize(@db : DB::Database)
  end
  
  def list_products(category : String? = nil) : Array(Product)
    if category
      @db.query_all(
        "SELECT * FROM products WHERE category = $1 ORDER BY name",
        category,
        as: Product
      )
    else
      @db.query_all("SELECT * FROM products ORDER BY name", as: Product)
    end
  end
  
  def find_by_sku(sku : String) : Product?
    @db.query_one?("SELECT * FROM products WHERE sku = $1", sku, as: Product)
  end
  
  def add_stock(product_id : Int64, quantity : Int32) : Bool
    @db.transaction do |tx|
      cnn = tx.connection
      
      current = cnn.query_one(
        "SELECT stock FROM products WHERE id = $1 FOR UPDATE",
        product_id,
        as: Int32
      )
      
      new_stock = current + quantity
      
      cnn.exec(
        "UPDATE products SET stock = $1 WHERE id = $2",
        new_stock, product_id
      )
      
      # บันทึก log
      cnn.exec(
        "INSERT INTO inventory_log (product_id, change, reason) VALUES ($1, $2, 'restock')",
        product_id, quantity
      )
      
      true
    end
  rescue
    false
  end
  
  def deduct_stock(product_id : Int64, quantity : Int32) : {Bool, String}
    @db.transaction do |tx|
      cnn = tx.connection
      
      current = cnn.query_one(
        "SELECT stock FROM products WHERE id = $1 FOR UPDATE",
        product_id,
        as: Int32
      )
      
      if current < quantity
        tx.rollback
        return {false, "Stock ไม่เพียงพอ (มี #{current}, ต้องการ #{quantity})"}
      end
      
      cnn.exec(
        "UPDATE products SET stock = stock - $1 WHERE id = $2",
        quantity, product_id
      )
      
      {true, ""}
    end
  rescue ex
    {false, ex.message || "Unknown error"}
  end
  
  def low_stock_report(threshold : Int32 = 10) : Array(Product)
    @db.query_all(
      "SELECT * FROM products WHERE stock <= $1 ORDER BY stock ASC",
      threshold,
      as: Product
    )
  end
end

# ใช้งาน
DB.open("postgres://user:pass@localhost/inventory_db") do |db|
  manager = InventoryManager.new(db)
  
  low_stock = manager.low_stock_report(5)
  puts "สินค้าใกล้หมด (≤5):"
  low_stock.each { |p| puts "  #{p.name}: #{p.stock} ชิ้น" }
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **crystal-db**: abstraction layer
2. **Connection String**: format สำหรับแต่ละ database
3. **query_one/query_one?**: ดึง 1 row
4. **query_all**: ดึงหลาย rows
5. **query**: manual iteration
6. **exec**: INSERT/UPDATE/DELETE
7. **DB::Serializable**: map columns เป็น struct
8. **Transactions**: atomic operations
9. **Connection Pool**: จัดการ connections
10. **Prepared Statements**: optimize repeated queries
11. **Error Handling**: PQError, constraint violations

crystal-db ให้ type safety และ performance ที่ดีสำหรับ database operations
