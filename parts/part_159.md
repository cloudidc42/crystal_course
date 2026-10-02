# Part 159: Query Builder ใน Crystal

## บทนำ

Query Builder คือเครื่องมือสร้าง SQL queries แบบ programmatic แทนการเขียน SQL string โดยตรง ช่วยป้องกัน SQL injection และทำให้ code อ่านง่ายขึ้น

## ทำไมต้องใช้ Query Builder

```crystal
# แบบไม่ดี - SQL injection risk!
name = params["name"]  # อาจเป็น "'; DROP TABLE users; --"
sql = "SELECT * FROM users WHERE name = '#{name}'"

# แบบดี - parameterized query
sql = "SELECT * FROM users WHERE name = $1"
db.query(sql, name)

# แบบดีที่สุด - Query Builder
User.where(name: name).all
```

## Parameterized Queries

### Crystal DB Native

```crystal
require "db"
require "pg"

DB.open("postgresql://localhost/myapp") do |db|
  # Single parameter
  user = db.query_one?(
    "SELECT id, email FROM users WHERE id = $1",
    42,
    as: {id: Int64, email: String}
  )

  # Multiple parameters
  users = db.query_all(
    "SELECT * FROM users WHERE role = $1 AND active = $2",
    "admin", true,
    as: {id: Int64, email: String, role: String}
  )

  # Insert with returning
  id = db.query_one(
    "INSERT INTO users (email, role) VALUES ($1, $2) RETURNING id",
    "alice@example.com", "user",
    as: Int64
  )

  puts "Created user ID: #{id}"
end
```

### ป้องกัน SQL Injection

```crystal
require "db"

# อันตราย - ห้ามทำ!
def bad_search(term : String)
  "SELECT * FROM products WHERE name LIKE '%#{term}%'"
end

# ปลอดภัย - ใช้ parameterized
def safe_search(db : DB::Database, term : String)
  db.query_all(
    "SELECT id, name, price FROM products WHERE name ILIKE $1",
    "%#{term}%",
    as: {id: Int64, name: String, price: Float64}
  )
end

# ปลอดภัยยิ่งขึ้น - ใช้ Query Builder
class ProductQuery
  def search(term : String)
    sanitized = term.gsub(/[%_\\]/) { |c| "\\#{c}" }
    QueryBuilder.new("products")
      .where("name ILIKE ?", "%#{sanitized}%")
      .select("id, name, price")
      .build
  end
end
```

## สร้าง Query Builder เอง

```crystal
# src/query_builder.cr

class QueryBuilder
  alias Value = String | Int32 | Int64 | Float64 | Bool | Nil | Time

  @table : String
  @conditions : Array(String) = [] of String
  @parameters : Array(Value) = [] of Value
  @param_counter : Int32 = 0
  @select_cols : String = "*"
  @order_by : Array(String) = [] of String
  @group_by : Array(String) = [] of String
  @having_clause : String? = nil
  @limit_val : Int32? = nil
  @offset_val : Int32? = nil
  @joins : Array(String) = [] of String

  def initialize(@table : String)
  end

  def select(*columns : String)
    @select_cols = columns.join(", ")
    self
  end

  def select(columns : String)
    @select_cols = columns
    self
  end

  def where(condition : String, *values : Value)
    # แปลง ? เป็น $1, $2, ...
    converted = condition.gsub("?") do
      @param_counter += 1
      "$#{@param_counter}"
    end
    @conditions << converted
    values.each { |v| @parameters << v }
    self
  end

  def where(**conditions)
    conditions.each do |key, value|
      @param_counter += 1
      case value
      when Nil
        @conditions << "#{key} IS NULL"
      else
        @conditions << "#{key} = $#{@param_counter}"
        @parameters << value
      end
    end
    self
  end

  def where_not(**conditions)
    conditions.each do |key, value|
      @param_counter += 1
      case value
      when Nil
        @conditions << "#{key} IS NOT NULL"
      else
        @conditions << "#{key} != $#{@param_counter}"
        @parameters << value
      end
    end
    self
  end

  def where_in(column : String, values : Array(Value))
    return self if values.empty?

    placeholders = values.map do |v|
      @param_counter += 1
      @parameters << v
      "$#{@param_counter}"
    end.join(", ")

    @conditions << "#{column} IN (#{placeholders})"
    self
  end

  def where_between(column : String, min : Value, max : Value)
    @param_counter += 1
    min_placeholder = "$#{@param_counter}"
    @parameters << min

    @param_counter += 1
    max_placeholder = "$#{@param_counter}"
    @parameters << max

    @conditions << "#{column} BETWEEN #{min_placeholder} AND #{max_placeholder}"
    self
  end

  def where_like(column : String, pattern : String)
    @param_counter += 1
    @conditions << "#{column} ILIKE $#{@param_counter}"
    @parameters << pattern
    self
  end

  def join(table : String, on : String, type : String = "INNER")
    @joins << "#{type} JOIN #{table} ON #{on}"
    self
  end

  def left_join(table : String, on : String)
    join(table, on, "LEFT")
  end

  def order(column : String, direction : String = "ASC")
    direction = direction.upcase
    raise ArgumentError.new("Invalid direction") unless %w[ASC DESC].includes?(direction)
    @order_by << "#{column} #{direction}"
    self
  end

  def order(**columns)
    columns.each do |col, dir|
      order(col.to_s, dir.to_s)
    end
    self
  end

  def group_by(*columns : String)
    columns.each { |c| @group_by << c }
    self
  end

  def having(condition : String, *values : Value)
    converted = condition.gsub("?") do
      @param_counter += 1
      "$#{@param_counter}"
    end
    @having_clause = converted
    values.each { |v| @parameters << v }
    self
  end

  def limit(n : Int32)
    @limit_val = n
    self
  end

  def offset(n : Int32)
    @offset_val = n
    self
  end

  def paginate(page : Int32, per_page : Int32 = 20)
    limit(per_page).offset((page - 1) * per_page)
  end

  def build : {String, Array(Value)}
    sql = String.build do |s|
      s << "SELECT #{@select_cols} FROM #{@table}"

      @joins.each { |j| s << " #{j}" }

      unless @conditions.empty?
        s << " WHERE " << @conditions.join(" AND ")
      end

      unless @group_by.empty?
        s << " GROUP BY " << @group_by.join(", ")
      end

      if having = @having_clause
        s << " HAVING " << having
      end

      unless @order_by.empty?
        s << " ORDER BY " << @order_by.join(", ")
      end

      if lim = @limit_val
        s << " LIMIT #{lim}"
      end

      if off = @offset_val
        s << " OFFSET #{off}"
      end
    end

    {sql, @parameters}
  end

  def to_sql : String
    build.first
  end

  def execute(db : DB::Database)
    sql, params = build
    db.query_all(sql, *params)
  end
end
```

### ใช้งาน QueryBuilder

```crystal
require "db"
require "pg"

DB.open("postgresql://localhost/myapp") do |db|
  # Simple query
  sql, params = QueryBuilder.new("users")
    .where(active: true)
    .order("created_at", "DESC")
    .limit(10)
    .build

  puts sql
  # SELECT * FROM users WHERE active = $1 ORDER BY created_at DESC LIMIT 10

  # Complex query
  sql, params = QueryBuilder.new("products")
    .select("id, name, price, categories.name AS category_name")
    .join("categories", "products.category_id = categories.id")
    .where(active: true)
    .where("price > ?", 100.0)
    .where_in("category_id", [1_i64, 2_i64, 3_i64])
    .order("price", "ASC")
    .paginate(2, 20)
    .build

  puts sql
  # SELECT id, name, price, categories.name AS category_name
  # FROM products
  # INNER JOIN categories ON products.category_id = categories.id
  # WHERE active = $1 AND price > $2 AND category_id IN ($3, $4, $5)
  # ORDER BY price ASC
  # LIMIT 20 OFFSET 20

  # Search query
  sql, params = QueryBuilder.new("products")
    .where_like("name", "%crystal%")
    .where_between("price", 100.0, 500.0)
    .order("name")
    .build

  puts sql
  # SELECT * FROM products WHERE name ILIKE $1 AND price BETWEEN $2 AND $3 ORDER BY name ASC

  # Aggregation
  sql, params = QueryBuilder.new("orders")
    .select("user_id, COUNT(*) AS order_count, SUM(total) AS total_spent")
    .where(status: "completed")
    .group_by("user_id")
    .having("COUNT(*) > ?", 5)
    .order("total_spent", "DESC")
    .limit(10)
    .build

  puts sql
end
```

## Insert Builder

```crystal
class InsertBuilder
  alias Value = String | Int32 | Int64 | Float64 | Bool | Nil | Time

  @table : String
  @columns : Array(String) = [] of String
  @values : Array(Value) = [] of Value
  @returning : Array(String) = [] of String

  def initialize(@table : String)
  end

  def set(column : String, value : Value)
    @columns << column
    @values << value
    self
  end

  def set(**columns)
    columns.each do |col, val|
      set(col.to_s, val)
    end
    self
  end

  def returning(*columns : String)
    columns.each { |c| @returning << c }
    self
  end

  def build : {String, Array(Value)}
    placeholders = (1..@columns.size).map { |i| "$#{i}" }.join(", ")

    sql = String.build do |s|
      s << "INSERT INTO #{@table}"
      s << " (#{@columns.join(", ")})"
      s << " VALUES (#{placeholders})"

      unless @returning.empty?
        s << " RETURNING #{@returning.join(", ")}"
      end
    end

    {sql, @values}
  end
end

# Bulk Insert Builder
class BulkInsertBuilder
  alias Value = String | Int32 | Int64 | Float64 | Bool | Nil | Time
  alias Row = Hash(String, Value)

  @table : String
  @columns : Array(String) = [] of String
  @rows : Array(Array(Value)) = [] of Array(Value)

  def initialize(@table : String)
  end

  def columns(*cols : String)
    @columns = cols.to_a
    self
  end

  def add_row(**values)
    row = @columns.map { |col| values[col]? }
    @rows << row
    self
  end

  def add_rows(rows : Array(Row))
    rows.each do |row|
      @rows << @columns.map { |col| row[col]? }
    end
    self
  end

  def build : {String, Array(Value)}
    params = [] of Value
    param_counter = 0

    value_groups = @rows.map do |row|
      placeholders = row.map do |val|
        param_counter += 1
        params << val
        "$#{param_counter}"
      end
      "(#{placeholders.join(", ")})"
    end

    sql = "INSERT INTO #{@table} (#{@columns.join(", ")}) VALUES #{value_groups.join(", ")}"
    {sql, params}
  end
end
```

### ใช้งาน Insert Builder

```crystal
# Single insert
sql, params = InsertBuilder.new("users")
  .set(email: "alice@example.com")
  .set(username: "alice")
  .set(role: "user")
  .set(created_at: Time.utc)
  .returning("id", "created_at")
  .build

puts sql
# INSERT INTO users (email, username, role, created_at)
# VALUES ($1, $2, $3, $4) RETURNING id, created_at

# Bulk insert
sql, params = BulkInsertBuilder.new("products")
  .columns("name", "price", "category")
  .add_row(name: "Book A", price: 299.0, category: "books")
  .add_row(name: "Book B", price: 399.0, category: "books")
  .add_row(name: "Phone X", price: 15000.0, category: "electronics")
  .build

puts sql
# INSERT INTO products (name, price, category)
# VALUES ($1, $2, $3), ($4, $5, $6), ($7, $8, $9)
```

## Update Builder

```crystal
class UpdateBuilder
  alias Value = String | Int32 | Int64 | Float64 | Bool | Nil | Time

  @table : String
  @set_clauses : Array(String) = [] of String
  @conditions : Array(String) = [] of String
  @parameters : Array(Value) = [] of Value
  @param_counter : Int32 = 0
  @returning : Array(String) = [] of String

  def initialize(@table : String)
  end

  def set(column : String, value : Value)
    @param_counter += 1
    @set_clauses << "#{column} = $#{@param_counter}"
    @parameters << value
    self
  end

  def set(**columns)
    columns.each { |col, val| set(col.to_s, val) }
    self
  end

  def increment(column : String, by : Value = 1)
    @set_clauses << "#{column} = #{column} + #{by}"
    self
  end

  def where(condition : String, *values : Value)
    converted = condition.gsub("?") do
      @param_counter += 1
      "$#{@param_counter}"
    end
    @conditions << converted
    values.each { |v| @parameters << v }
    self
  end

  def where(**conditions)
    conditions.each do |key, value|
      @param_counter += 1
      @conditions << "#{key} = $#{@param_counter}"
      @parameters << value
    end
    self
  end

  def returning(*columns : String)
    columns.each { |c| @returning << c }
    self
  end

  def build : {String, Array(Value)}
    raise "No SET clauses" if @set_clauses.empty?
    raise "No WHERE conditions (use delete_all for unconditional)" if @conditions.empty?

    sql = String.build do |s|
      s << "UPDATE #{@table}"
      s << " SET #{@set_clauses.join(", ")}"
      s << " WHERE #{@conditions.join(" AND ")}"
      unless @returning.empty?
        s << " RETURNING #{@returning.join(", ")}"
      end
    end

    {sql, @parameters}
  end
end

# ใช้งาน
sql, params = UpdateBuilder.new("users")
  .set(first_name: "Alice", updated_at: Time.utc)
  .increment("login_count")
  .where(id: 42_i64)
  .returning("id", "login_count")
  .build

puts sql
# UPDATE users SET first_name = $1, updated_at = $2, login_count = login_count + 1
# WHERE id = $3 RETURNING id, login_count
```

## Delete Builder

```crystal
class DeleteBuilder
  alias Value = String | Int32 | Int64 | Float64 | Bool | Nil | Time

  @table : String
  @conditions : Array(String) = [] of String
  @parameters : Array(Value) = [] of Value
  @param_counter : Int32 = 0
  @soft_delete : Bool = false

  def initialize(@table : String)
  end

  def where(condition : String, *values : Value)
    converted = condition.gsub("?") do
      @param_counter += 1
      "$#{@param_counter}"
    end
    @conditions << converted
    values.each { |v| @parameters << v }
    self
  end

  def where(**conditions)
    conditions.each do |key, value|
      @param_counter += 1
      @conditions << "#{key} = $#{@param_counter}"
      @parameters << value
    end
    self
  end

  def soft_delete
    @soft_delete = true
    self
  end

  def build : {String, Array(Value)}
    sql = if @soft_delete
      @param_counter += 1
      @parameters << Time.utc
      conditions_sql = @conditions.empty? ? "" : " WHERE #{@conditions.join(" AND ")}"
      "UPDATE #{@table} SET deleted_at = $#{@param_counter}#{conditions_sql}"
    else
      conditions_sql = @conditions.empty? ? "" : " WHERE #{@conditions.join(" AND ")}"
      "DELETE FROM #{@table}#{conditions_sql}"
    end

    {sql, @parameters}
  end
end

# ใช้งาน
sql, params = DeleteBuilder.new("sessions")
  .where("expires_at < ?", Time.utc)
  .build

puts sql  # DELETE FROM sessions WHERE expires_at < $1

# Soft delete
sql, params = DeleteBuilder.new("users")
  .where(id: 42_i64)
  .soft_delete
  .build

puts sql  # UPDATE users SET deleted_at = $1 WHERE id = $2
```

## Complete Query Builder Library

```crystal
# รวม Query Builder ทั้งหมดเป็น module
module QB
  def self.from(table : String)
    QueryBuilder.new(table)
  end

  def self.insert_into(table : String)
    InsertBuilder.new(table)
  end

  def self.update(table : String)
    UpdateBuilder.new(table)
  end

  def self.delete_from(table : String)
    DeleteBuilder.new(table)
  end
end

# ใช้งานแบบ fluent API
DB.open(ENV["DATABASE_URL"]) do |db|
  # SELECT
  sql, params = QB.from("products")
    .select("id, name, price")
    .where(active: true)
    .where("price > ?", 100.0)
    .order("price")
    .limit(20)
    .build

  products = db.query_all(sql, args: params)

  # INSERT
  sql, params = QB.insert_into("orders")
    .set(user_id: 1_i64, status: "pending", total: 599.0, created_at: Time.utc)
    .returning("id")
    .build

  order_id = db.query_one(sql, args: params, as: Int64)

  # UPDATE
  sql, params = QB.update("orders")
    .set(status: "processing", updated_at: Time.utc)
    .where(id: order_id)
    .build

  db.exec(sql, args: params)

  # DELETE
  sql, params = QB.delete_from("sessions")
    .where("expires_at < ?", Time.utc)
    .build

  db.exec(sql, args: params)
end
```

## แบบฝึกหัด

1. เพิ่ม method `or_where` ให้ QueryBuilder ที่รองรับ `WHERE (a = 1 OR b = 2)` conditions
2. สร้าง `SubqueryBuilder` ที่ใช้ query builder อื่นเป็น subquery: `WHERE id IN (SELECT user_id FROM ...)`
3. เพิ่ม `upsert` ใน InsertBuilder: `INSERT ... ON CONFLICT (email) DO UPDATE SET ...`
4. สร้าง query builder สำหรับ MySQL ที่ใช้ `?` แทน `$1` สำหรับ parameters

## สรุป

Query Builder ใน Crystal:
- **Parameterized queries**: ป้องกัน SQL injection ด้วยการแยก SQL และ parameters
- **Fluent API**: method chaining ที่อ่านง่าย
- **Type-safe**: Crystal's type system ป้องกัน runtime errors
- **Flexible**: รองรับ complex queries, joins, aggregations
- **Reusable**: สร้าง query fragments แล้วนำมาต่อกัน

Query Builder ช่วยให้ code อ่านง่ายกว่า raw SQL strings และปลอดภัยกว่าการ concatenate strings
