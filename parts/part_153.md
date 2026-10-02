# Part 153: MySQL/MariaDB - การใช้งาน MySQL ใน Crystal

## บทนำ

MySQL และ MariaDB เป็น relational databases ที่นิยมใช้กันอย่างแพร่หลาย Crystal รองรับผ่าน `crystal-mysql` shard

## การติดตั้ง

```yaml
# shard.yml
dependencies:
  mysql:
    github: crystal-lang/crystal-mysql
    version: ~> 0.14.0
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
```

## เชื่อมต่อ MySQL

```crystal
require "db"
require "mysql"

# Single connection
DB.open("mysql://user:password@localhost:3306/myapp") do |db|
  version = db.query_one("SELECT VERSION()", as: String)
  puts "MySQL version: #{version}"
end

# Connection pool
DATABASE = DB.open(
  "mysql://user:password@localhost:3306/myapp" +
  "?max_pool_size=20&initial_pool_size=5"
)

# ปิด pool เมื่อ app ปิด
at_exit { DATABASE.close }
```

## CRUD พื้นฐาน

```crystal
require "db"
require "mysql"

struct Product
  include DB::Serializable

  property id : Int64
  property name : String
  property price : Float64
  property stock : Int32
  property category : String
  property active : Bool
  property created_at : Time
end

DB.open("mysql://user:pass@localhost/shop") do |db|

  # CREATE TABLE
  db.exec <<-SQL
    CREATE TABLE IF NOT EXISTS products (
      id BIGINT AUTO_INCREMENT PRIMARY KEY,
      name VARCHAR(255) NOT NULL,
      price DECIMAL(10,2) NOT NULL DEFAULT 0.00,
      stock INT NOT NULL DEFAULT 0,
      category VARCHAR(100) NOT NULL,
      active TINYINT(1) NOT NULL DEFAULT 1,
      created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
  SQL

  # INSERT
  result = db.exec(
    "INSERT INTO products (name, price, stock, category) VALUES (?, ?, ?, ?)",
    "Crystal Book", 299.0, 100, "books"
  )
  puts "Inserted ID: #{result.last_insert_id}"

  # SELECT
  products = db.query_all(
    "SELECT * FROM products WHERE active = 1 ORDER BY name",
    as: Product
  )
  products.each { |p| puts "#{p.name}: #{p.price}" }

  # SELECT ONE
  product = db.query_one?(
    "SELECT * FROM products WHERE id = ?",
    1_i64, as: Product
  )
  puts product.try(&.name) || "Not found"

  # UPDATE
  db.exec(
    "UPDATE products SET price = ?, stock = stock - ? WHERE id = ?",
    250.0, 1, 1_i64
  )

  # DELETE
  db.exec("DELETE FROM products WHERE id = ?", 99_i64)

end
```

## Placeholder ใน MySQL

```crystal
require "db"
require "mysql"

# MySQL ใช้ ? เป็น placeholder (ต่างจาก PostgreSQL ที่ใช้ $1, $2)
DB.open("mysql://user:pass@localhost/mydb") do |db|

  # ถูกต้อง - ใช้ ?
  users = db.query_all(
    "SELECT * FROM users WHERE role = ? AND active = ?",
    "admin", true,
    as: {id: Int64, name: String, email: String}
  )

  # Multiple placeholders
  db.exec(
    "INSERT INTO orders (user_id, product_id, quantity, total) VALUES (?, ?, ?, ?)",
    1_i64, 5_i64, 2, 598.0
  )

  # IN clause ต้องสร้าง placeholder เอง
  ids = [1, 2, 3, 4, 5]
  placeholders = ids.map { "?" }.join(", ")
  users_by_ids = db.query_all(
    "SELECT * FROM users WHERE id IN (#{placeholders})",
    *ids,
    as: {id: Int64, name: String}
  )

end
```

## Transactions

```crystal
require "db"
require "mysql"

DB.open("mysql://user:pass@localhost/mydb") do |db|

  # Transaction พื้นฐาน
  db.transaction do |tx|
    cnn = tx.connection

    # ลด stock
    result = cnn.exec(
      "UPDATE products SET stock = stock - ? WHERE id = ? AND stock >= ?",
      1, 5_i64, 1
    )

    if result.rows_affected == 0
      tx.rollback
      puts "Stock ไม่เพียงพอ"
      next
    end

    # สร้าง order
    order_id = cnn.query_one(
      "INSERT INTO orders (product_id, quantity) VALUES (?, ?) RETURNING id",
      5_i64, 1,
      as: Int64
    )

    puts "Order created: #{order_id}"
    # auto commit เมื่อ block จบ
  end

  # Nested transaction ด้วย SAVEPOINT
  db.transaction do |tx|
    cnn = tx.connection

    cnn.exec("INSERT INTO audit_log (action) VALUES (?)", "start_batch")

    [1_i64, 2_i64, 3_i64].each do |user_id|
      cnn.exec("SAVEPOINT sp_#{user_id}")

      begin
        cnn.exec("UPDATE users SET points = points + 10 WHERE id = ?", user_id)
        cnn.exec("INSERT INTO point_log (user_id, amount) VALUES (?, 10)", user_id)
      rescue ex
        cnn.exec("ROLLBACK TO SAVEPOINT sp_#{user_id}")
        puts "Failed for user #{user_id}: #{ex.message}"
      end

      cnn.exec("RELEASE SAVEPOINT sp_#{user_id}")
    end
  end

end
```

## MySQL-specific Features

```crystal
require "db"
require "mysql"

DB.open("mysql://user:pass@localhost/mydb") do |db|

  # AUTO_INCREMENT last insert id
  result = db.exec(
    "INSERT INTO categories (name) VALUES (?)",
    "Electronics"
  )
  category_id = result.last_insert_id
  puts "New category ID: #{category_id}"

  # ON DUPLICATE KEY UPDATE (upsert)
  db.exec(
    <<-SQL,
      INSERT INTO user_stats (user_id, login_count, last_login)
      VALUES (?, 1, NOW())
      ON DUPLICATE KEY UPDATE
        login_count = login_count + 1,
        last_login = NOW()
    SQL
    42_i64
  )

  # REPLACE INTO (delete + insert)
  db.exec(
    "REPLACE INTO settings (key_name, value) VALUES (?, ?)",
    "theme", "dark"
  )

  # INSERT IGNORE (ignore duplicates)
  db.exec(
    "INSERT IGNORE INTO user_roles (user_id, role) VALUES (?, ?)",
    1_i64, "admin"
  )

  # Full-text search
  db.exec(<<-SQL)
    ALTER TABLE products ADD FULLTEXT INDEX ft_name_desc (name, description)
  SQL

  results = db.query_all(
    "SELECT *, MATCH(name, description) AGAINST(?) as relevance FROM products WHERE MATCH(name, description) AGAINST(?)",
    "crystal programming", "crystal programming",
    as: {id: Int64, name: String, relevance: Float64}
  )

  results.sort_by!(&.[:relevance]).reverse!
  results.each { |r| puts "#{r[:name]} (score: #{r[:relevance].round(3)})" }

  # JSON column (MySQL 5.7+)
  db.exec(
    "ALTER TABLE users ADD COLUMN metadata JSON"
  )

  db.exec(
    "UPDATE users SET metadata = ? WHERE id = ?",
    {"theme" => "dark", "lang" => "th"}.to_json, 1_i64
  )

  # Query JSON
  users = db.query_all(
    "SELECT id, name FROM users WHERE JSON_EXTRACT(metadata, '$.theme') = ?",
    "\"dark\"",
    as: {id: Int64, name: String}
  )

end
```

## Repository Pattern

```crystal
require "db"
require "mysql"

struct User
  include DB::Serializable

  property id : Int64
  property email : String
  property name : String
  property role : String
  property active : Bool
  property created_at : Time
end

class UserRepository
  def initialize(@db : DB::Database)
  end

  def find(id : Int64) : User?
    @db.query_one?("SELECT * FROM users WHERE id = ?", id, as: User)
  end

  def find_by_email(email : String) : User?
    @db.query_one?("SELECT * FROM users WHERE email = ?", email, as: User)
  end

  def all(active_only : Bool = true) : Array(User)
    if active_only
      @db.query_all("SELECT * FROM users WHERE active = 1 ORDER BY name", as: User)
    else
      @db.query_all("SELECT * FROM users ORDER BY name", as: User)
    end
  end

  def paginate(page : Int32, per_page : Int32 = 20) : {Array(User), Int64}
    offset = (page - 1) * per_page
    users = @db.query_all(
      "SELECT * FROM users ORDER BY created_at DESC LIMIT ? OFFSET ?",
      per_page, offset, as: User
    )
    total = @db.query_one("SELECT COUNT(*) FROM users", as: Int64)
    {users, total}
  end

  def create(email : String, name : String, password_hash : String) : User?
    result = @db.exec(
      "INSERT INTO users (email, name, password_hash, role, active) VALUES (?, ?, ?, 'user', 1)",
      email, name, password_hash
    )
    find(result.last_insert_id)
  rescue
    nil
  end

  def update(id : Int64, name : String? = nil, role : String? = nil) : Bool
    parts = [] of String
    args = [] of DB::Any

    if name
      parts << "name = ?"
      args << name
    end

    if role
      parts << "role = ?"
      args << role
    end

    return false if parts.empty?

    args << id
    result = @db.exec(
      "UPDATE users SET #{parts.join(", ")} WHERE id = ?",
      *args
    )
    result.rows_affected > 0
  end

  def soft_delete(id : Int64) : Bool
    result = @db.exec("UPDATE users SET active = 0 WHERE id = ?", id)
    result.rows_affected > 0
  end

  def count_by_role : Hash(String, Int64)
    rows = @db.query_all(
      "SELECT role, COUNT(*) as cnt FROM users WHERE active = 1 GROUP BY role",
      as: {role: String, cnt: Int64}
    )
    rows.each_with_object({} of String => Int64) do |r, h|
      h[r[:role]] = r[:cnt]
    end
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Order Management System

```crystal
require "db"
require "mysql"

struct Order
  include DB::Serializable

  @[DB::Field(key: "id")]
  property id : Int64

  @[DB::Field(key: "user_id")]
  property user_id : Int64

  property status : String
  property total : Float64

  @[DB::Field(key: "created_at")]
  property created_at : Time
end

struct OrderItem
  include DB::Serializable

  property id : Int64
  property order_id : Int64
  property product_id : Int64
  property quantity : Int32
  property price : Float64
end

class OrderService
  def initialize(@db : DB::Database)
  end

  def create_order(user_id : Int64, items : Array({product_id: Int64, quantity: Int32})) : {Order?, String}
    @db.transaction do |tx|
      cnn = tx.connection

      # ตรวจสอบ stock ทุก item
      items.each do |item|
        stock = cnn.query_one?(
          "SELECT stock FROM products WHERE id = ? AND active = 1 FOR UPDATE",
          item[:product_id], as: Int32
        )

        unless stock
          tx.rollback
          return {nil, "Product #{item[:product_id]} not found"}
        end

        if stock < item[:quantity]
          tx.rollback
          return {nil, "Insufficient stock for product #{item[:product_id]}"}
        end
      end

      # คำนวณ total
      total = 0.0
      item_data = items.map do |item|
        price = cnn.query_one(
          "SELECT price FROM products WHERE id = ?",
          item[:product_id], as: Float64
        )
        total += price * item[:quantity]
        {product_id: item[:product_id], quantity: item[:quantity], price: price}
      end

      # สร้าง order
      order_result = cnn.exec(
        "INSERT INTO orders (user_id, status, total) VALUES (?, 'pending', ?)",
        user_id, total
      )
      order_id = order_result.last_insert_id

      # สร้าง order items และ ลด stock
      item_data.each do |item|
        cnn.exec(
          "INSERT INTO order_items (order_id, product_id, quantity, price) VALUES (?, ?, ?, ?)",
          order_id, item[:product_id], item[:quantity], item[:price]
        )

        cnn.exec(
          "UPDATE products SET stock = stock - ? WHERE id = ?",
          item[:quantity], item[:product_id]
        )
      end

      order = cnn.query_one?(
        "SELECT * FROM orders WHERE id = ?",
        order_id, as: Order
      )

      {order, ""}
    end
  rescue ex
    {nil, ex.message || "Unknown error"}
  end

  def update_status(order_id : Int64, status : String) : Bool
    valid_statuses = ["pending", "confirmed", "shipped", "delivered", "cancelled"]
    return false unless valid_statuses.includes?(status)

    result = @db.exec(
      "UPDATE orders SET status = ? WHERE id = ?",
      status, order_id
    )
    result.rows_affected > 0
  end

  def user_orders(user_id : Int64) : Array(Order)
    @db.query_all(
      "SELECT * FROM orders WHERE user_id = ? ORDER BY created_at DESC",
      user_id, as: Order
    )
  end
end

# ใช้งาน
DB.open("mysql://user:pass@localhost/shop") do |db|
  service = OrderService.new(db)

  order, error = service.create_order(
    1_i64,
    [
      {product_id: 1_i64, quantity: 2},
      {product_id: 3_i64, quantity: 1}
    ]
  )

  if order
    puts "Order created: ##{order.id}, Total: #{order.total}"
  else
    puts "Error: #{error}"
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Crystal-MySQL**: shard สำหรับเชื่อมต่อ MySQL/MariaDB
2. **Connection String**: `mysql://user:pass@host/db`
3. **Placeholder**: ใช้ `?` (ต่างจาก PostgreSQL ที่ใช้ `$1`)
4. **CRUD**: INSERT, SELECT, UPDATE, DELETE
5. **last_insert_id**: ดึง AUTO_INCREMENT id
6. **ON DUPLICATE KEY**: upsert operation
7. **REPLACE INTO**: delete + insert
8. **INSERT IGNORE**: skip duplicates
9. **Full-text Search**: FULLTEXT index
10. **JSON Column**: MySQL 5.7+ feature
11. **Transactions**: atomic operations + savepoints
12. **Repository Pattern**: clean data access layer

MySQL + Crystal = type-safe, high-performance web applications
