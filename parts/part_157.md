# Part 157: ORM Granite ใน Crystal

## บทนำ

Granite เป็น ORM (Object-Relational Mapper) สำหรับ Crystal ที่พัฒนาโดย Amber Framework ช่วยให้เราทำงานกับฐานข้อมูลผ่าน Crystal objects แทนการเขียน SQL โดยตรง

## การติดตั้ง

เพิ่มใน `shard.yml`:

```yaml
dependencies:
  granite:
    github: amberframework/granite
    version: ~> 0.26
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13
  pg:
    github: will/crystal-pg
    version: ~> 0.28
  sqlite3:
    github: crystal-lang/crystal-sqlite3
    version: ~> 0.20
```

```bash
shards install
```

## การตั้งค่า Database

```crystal
require "granite"
require "granite/adapter/pg"
require "granite/adapter/sqlite"

# PostgreSQL
Granite::Connections << Granite::Adapter::Pg.new(
  name: "pg",
  url:  ENV["DATABASE_URL"]? || "postgresql://localhost/myapp_dev"
)

# SQLite (สำหรับ development)
Granite::Connections << Granite::Adapter::Sqlite.new(
  name: "sqlite",
  url:  "sqlite3://./db/development.sqlite3"
)
```

## Model Class

```crystal
require "granite"
require "granite/adapter/pg"

class User < Granite::Base
  # กำหนด adapter
  connection "pg"
  table "users"

  # Primary key (auto increment by default)
  column id : Int64, primary: true

  # Columns
  column email : String
  column username : String
  column password_digest : String
  column first_name : String?
  column last_name : String?
  column role : String = "user"
  column active : Bool = true
  column login_count : Int32 = 0

  # Timestamps (อัปเดตอัตโนมัติ)
  timestamps

  # Callbacks
  before_save :normalize_email
  before_create :set_defaults
  after_create :send_welcome_email

  private def normalize_email
    self.email = email.downcase.strip
  end

  private def set_defaults
    self.role = "user" if role.empty?
  end

  private def send_welcome_email
    # ส่ง email...
    puts "ส่ง welcome email ไปยัง #{email}"
  end
end
```

## Validate

```crystal
class Product < Granite::Base
  connection "pg"
  table "products"

  column id : Int64, primary: true
  column name : String
  column price : Float64
  column description : String?
  column stock : Int32 = 0
  column category : String
  column sku : String

  timestamps

  # Validations
  validate :name, presence: true
  validate :name, length: {min: 3, max: 100}

  validate :price, presence: true
  validate :price, numericality: {greater_than: 0}

  validate :stock, numericality: {greater_than_or_equal_to: 0}

  validate :sku, presence: true
  validate :sku, uniqueness: true

  validate :category, inclusion: {in: %w[electronics clothing food books other]}

  # Custom validation
  validate :valid_price_range

  private def valid_price_range
    if price && price > 1_000_000.0
      errors.add(:price, "ราคาสูงเกินไป")
    end
  end

  # Validate กับ custom message
  validate :name, presence: true, message: "ต้องระบุชื่อสินค้า"
end

# ตรวจสอบ validation
product = Product.new
product.name = "A"  # สั้นเกินไป
product.price = -100.0  # ไม่ถูกต้อง
product.stock = 50
product.category = "invalid"
product.sku = "SKU001"

if product.valid?
  puts "ข้อมูลถูกต้อง"
else
  puts "ข้อผิดพลาด:"
  product.errors.each do |error|
    puts "  #{error.field}: #{error.message}"
  end
end
```

## Save, Create, Update, Delete

```crystal
# Create ด้วย new + save
user = User.new
user.email = "alice@example.com"
user.username = "alice"
user.password_digest = hash_password("secret123")

if user.save
  puts "บันทึกสำเร็จ! ID: #{user.id}"
else
  user.errors.each { |e| puts "Error: #{e.field} - #{e.message}" }
end

# Create ด้วย create (new + save ในขั้นตอนเดียว)
user = User.create(
  email:           "bob@example.com",
  username:        "bob",
  password_digest: hash_password("pass456")
)

if user.persisted?
  puts "สร้างสำเร็จ: #{user.id}"
end

# Update
user = User.find(1)
if user
  user.first_name = "Alice"
  user.last_name = "Smith"
  user.save
  puts "อัปเดตสำเร็จ"
end

# Update แบบ direct
User.update(1, {first_name: "Alice", last_name: "Smith"})

# Delete
user = User.find(1)
user.try(&.destroy)

# Delete แบบ direct
User.destroy(1)

# Delete ทั้งหมด
User.destroy_all
```

## All, Find, Where

```crystal
# หาทั้งหมด
users = User.all
users.each { |u| puts u.email }

# หาด้วย ID
user = User.find(1)
puts user.try(&.email) || "ไม่พบ"

# หาด้วย ID หรือ raise
begin
  user = User.find!(42)
rescue Granite::RecordNotFound
  puts "ไม่พบ user ID 42"
end

# หาตามเงื่อนไข
admin = User.find_by(role: "admin")
puts admin.try(&.email) || "ไม่มี admin"

# หลายตัวด้วย where
active_users = User.where(active: true)
active_users.each { |u| puts u.username }

# Where แบบ SQL string
expensive = Product.where("price > ?", 500.0)
expensive.each { |p| puts "#{p.name}: ฿#{p.price}" }

# Where แบบ Hash
programming = Product.where(category: "programming", active: true)

# Order
recent = User.order(created_at: :desc).limit(10)

# Complex query
results = User
  .where(active: true)
  .where("created_at > ?", 30.days.ago)
  .order(username: :asc)
  .limit(20)
  .offset(40)

results.each { |u| puts u.username }
```

## Associations

### Has Many / Belongs To

```crystal
class Author < Granite::Base
  connection "pg"
  table "authors"

  column id : Int64, primary: true
  column name : String
  column email : String
  timestamps

  has_many :books
  has_many :reviews, through: :books
end

class Book < Granite::Base
  connection "pg"
  table "books"

  column id : Int64, primary: true
  column title : String
  column isbn : String
  column price : Float64
  column published_at : Time?
  column author_id : Int64

  timestamps

  belongs_to :author
  has_many :reviews

  validate :title, presence: true
  validate :isbn, uniqueness: true
end

class Review < Granite::Base
  connection "pg"
  table "reviews"

  column id : Int64, primary: true
  column content : String
  column rating : Int32
  column book_id : Int64
  column user_id : Int64

  timestamps

  belongs_to :book
  belongs_to :user
end
```

### ใช้งาน Associations

```crystal
# หา books ของ author
author = Author.find!(1)
author.books.each do |book|
  puts "#{book.title} - ฿#{book.price}"
end

# หา author ของ book
book = Book.find!(1)
puts "เขียนโดย: #{book.author.try(&.name)}"

# สร้าง book พร้อม author
author = Author.find!(1)
book = Book.new
book.title = "Crystal Programming"
book.isbn = "978-0-000-00001-1"
book.price = 599.0
book.author_id = author.id!
book.save

# นับ books ของแต่ละ author
Author.all.each do |a|
  count = a.books.count
  puts "#{a.name}: #{count} เล่ม"
end
```

### Has One

```crystal
class User < Granite::Base
  connection "pg"
  table "users"

  column id : Int64, primary: true
  column email : String
  column username : String
  timestamps

  has_one :profile
  has_many :orders
  has_many :products, through: :orders
end

class Profile < Granite::Base
  connection "pg"
  table "profiles"

  column id : Int64, primary: true
  column user_id : Int64
  column bio : String?
  column avatar_url : String?
  column website : String?
  timestamps

  belongs_to :user
end

# ใช้งาน
user = User.find!(1)
profile = user.profile

if profile
  puts "Bio: #{profile.bio}"
else
  # สร้าง profile
  new_profile = Profile.new
  new_profile.user_id = user.id!
  new_profile.bio = "Crystal developer"
  new_profile.save
end
```

### Many-to-Many

```crystal
class Post < Granite::Base
  connection "pg"
  table "posts"

  column id : Int64, primary: true
  column title : String
  column content : String
  column user_id : Int64
  timestamps

  belongs_to :user
  has_many :post_tags
  has_many :tags, through: :post_tags
end

class Tag < Granite::Base
  connection "pg"
  table "tags"

  column id : Int64, primary: true
  column name : String
  column slug : String
  timestamps

  has_many :post_tags
  has_many :posts, through: :post_tags

  validate :name, uniqueness: true
end

class PostTag < Granite::Base
  connection "pg"
  table "post_tags"

  column id : Int64, primary: true
  column post_id : Int64
  column tag_id : Int64

  belongs_to :post
  belongs_to :tag
end

# ใช้งาน
post = Post.find!(1)
post.tags.each { |tag| puts tag.name }

# เพิ่ม tag ให้ post
tag = Tag.find_by(name: "crystal")
if tag
  post_tag = PostTag.new
  post_tag.post_id = post.id!
  post_tag.tag_id = tag.id!
  post_tag.save
end
```

## Migrations

```crystal
# Granite ไม่มี built-in migration tool
# แต่เราสามารถใช้ micrate หรือ crecto-migrations

# สร้างไฟล์ migration แบบ manual
# db/migrations/001_create_users.sql
```

```sql
-- db/migrations/001_create_users.sql
CREATE TABLE users (
  id BIGSERIAL PRIMARY KEY,
  email VARCHAR(255) NOT NULL UNIQUE,
  username VARCHAR(100) NOT NULL UNIQUE,
  password_digest VARCHAR(255) NOT NULL,
  first_name VARCHAR(100),
  last_name VARCHAR(100),
  role VARCHAR(50) DEFAULT 'user',
  active BOOLEAN DEFAULT true,
  login_count INTEGER DEFAULT 0,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_active ON users(active);
```

```crystal
# Migration runner ง่ายๆ
require "db"
require "pg"

class Migrator
  def initialize(@db : DB::Database)
  end

  def run_migrations(dir : String)
    # สร้าง migrations table ถ้ายังไม่มี
    @db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS schema_migrations (
        version VARCHAR(255) PRIMARY KEY,
        applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
      )
    SQL

    # หา migrations ที่ยังไม่ได้รัน
    Dir.glob("#{dir}/*.sql").sort.each do |file|
      version = File.basename(file, ".sql")

      applied = @db.query_one?(
        "SELECT COUNT(*) FROM schema_migrations WHERE version = $1",
        version, as: Int64
      )

      if applied == 0
        puts "Running migration: #{version}"
        sql = File.read(file)
        @db.exec(sql)
        @db.exec(
          "INSERT INTO schema_migrations (version) VALUES ($1)",
          version
        )
        puts "  Done"
      else
        puts "Skipping: #{version} (already applied)"
      end
    end
  end
end

# รัน migrations
DB.open(ENV["DATABASE_URL"]) do |db|
  migrator = Migrator.new(db)
  migrator.run_migrations("db/migrations")
end
```

## Scopes

```crystal
class Product < Granite::Base
  connection "pg"
  table "products"

  column id : Int64, primary: true
  column name : String
  column price : Float64
  column stock : Int32
  column active : Bool = true
  column featured : Bool = false
  column category : String
  timestamps

  # Named scopes
  scope :active, ->{
    where(active: true)
  }

  scope :featured, ->{
    where(featured: true)
  }

  scope :in_stock, ->{
    where("stock > 0")
  }

  scope :by_category, ->(category : String){
    where(category: category)
  }

  scope :price_range, ->(min : Float64, max : Float64){
    where("price BETWEEN ? AND ?", min, max)
  }

  scope :recent, ->{
    order(created_at: :desc).limit(10)
  }
end

# ใช้งาน scopes
Product.active.each { |p| puts p.name }
Product.active.in_stock.featured.each { |p| puts p.name }
Product.by_category("electronics").price_range(100.0, 500.0)
Product.recent
```

## Callbacks

```crystal
class Order < Granite::Base
  connection "pg"
  table "orders"

  column id : Int64, primary: true
  column user_id : Int64
  column total_amount : Float64
  column status : String = "pending"
  column payment_status : String = "unpaid"
  column order_number : String?
  timestamps

  before_create :generate_order_number
  before_save :calculate_total
  after_create :notify_admin
  after_save :update_inventory
  before_destroy :check_cancellable

  private def generate_order_number
    self.order_number = "ORD-#{Time.utc.to_unix_ms}-#{rand(1000)}"
  end

  private def calculate_total
    # คำนวณยอดรวม
  end

  private def notify_admin
    puts "แจ้งเตือน admin: Order #{order_number} ถูกสร้าง"
  end

  private def update_inventory
    # อัปเดต stock
  end

  private def check_cancellable
    if status == "shipped"
      errors.add(:status, "ไม่สามารถลบ order ที่จัดส่งแล้ว")
      false  # หยุดการลบ
    end
  end
end
```

## Custom Queries

```crystal
class User < Granite::Base
  connection "pg"
  table "users"

  column id : Int64, primary: true
  column email : String
  column username : String
  column role : String
  column active : Bool
  column login_count : Int32
  timestamps

  # Raw SQL query
  def self.top_active_users(limit : Int32 = 10)
    sql = <<-SQL
      SELECT * FROM users
      WHERE active = true
      ORDER BY login_count DESC
      LIMIT $1
    SQL

    query_all(sql, limit)
  end

  # Complex search
  def self.search(term : String)
    where(
      "email ILIKE ? OR username ILIKE ?",
      "%#{term}%",
      "%#{term}%"
    )
  end

  # Aggregation
  def self.stats
    DB.open(Granite::Connections.db_url) do |db|
      db.query_one(
        "SELECT COUNT(*) as total, COUNT(CASE WHEN active THEN 1 END) as active_count FROM users",
        as: {total: Int64, active_count: Int64}
      )
    end
  end
end

# ใช้งาน
User.top_active_users(5).each { |u| puts "#{u.username}: #{u.login_count} logins" }
User.search("alice").each { |u| puts u.email }
stats = User.stats
puts "Total: #{stats[:total]}, Active: #{stats[:active_count]}"
```

## Transaction Support

```crystal
require "granite"

# Transaction ด้วย Crystal DB
DB.open(ENV["DATABASE_URL"]) do |db|
  db.transaction do |tx|
    cnn = tx.connection

    # Deduct from source
    cnn.exec(
      "UPDATE accounts SET balance = balance - $1 WHERE id = $2",
      amount, source_id
    )

    # Add to destination
    cnn.exec(
      "UPDATE accounts SET balance = balance + $1 WHERE id = $2",
      amount, dest_id
    )

    # Record transaction
    cnn.exec(
      "INSERT INTO transfers (from_id, to_id, amount) VALUES ($1, $2, $3)",
      source_id, dest_id, amount
    )
  end
end
```

## แบบฝึกหัด

1. สร้าง Blog system ด้วย Granite ที่มี Models: User, Post, Comment, Tag พร้อม associations ครบ
2. เพิ่ม custom validations ที่ตรวจสอบ email format และ password strength
3. สร้าง scopes สำหรับ Product: available, on_sale, new_arrivals, best_sellers
4. ทำ search feature ที่ค้นหาได้ทั้ง title และ content ของ Post

## สรุป

Granite ORM สำหรับ Crystal:
- **Model Class**: สืบทอดจาก `Granite::Base` พร้อม column definitions
- **Validations**: presence, uniqueness, length, numericality, custom
- **CRUD**: save, create, find, where, update, destroy
- **Associations**: belongs_to, has_one, has_many, many-to-many
- **Callbacks**: before/after save, create, destroy
- **Scopes**: named scopes ที่ reusable
- **Custom Queries**: raw SQL เมื่อต้องการ flexibility

Granite ทำให้การทำงานกับ database ง่ายขึ้นโดยใช้ Crystal objects แทน SQL ตรงๆ
