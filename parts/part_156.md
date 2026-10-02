# Part 156: MongoDB ใน Crystal

## บทนำ

MongoDB เป็นฐานข้อมูล NoSQL ที่เก็บข้อมูลในรูปแบบ BSON (Binary JSON) Crystal มี driver สำหรับ MongoDB ที่ช่วยให้เราสามารถทำงานกับฐานข้อมูลแบบ document-based ได้อย่างมีประสิทธิภาพ

## การติดตั้ง

เพิ่ม shard ใน `shard.yml`:

```yaml
dependencies:
  mongo.cr:
    github: elbywan/mongo.cr
    version: ~> 0.1
```

รัน:
```bash
shards install
```

## การเชื่อมต่อ MongoDB

```crystal
require "mongo"

# เชื่อมต่อกับ MongoDB
client = Mongo::Client.new("mongodb://localhost:27017")
db = client["myapp"]
collection = db["users"]

puts "เชื่อมต่อกับ MongoDB สำเร็จ"
```

## MongoDB Sharding พื้นฐาน

Sharding คือการกระจายข้อมูลไปยังหลาย server เพื่อรองรับการขยายตัวในแนวนอน

```
+------------------+
|   mongos         |  <- Query Router
+------------------+
        |
   +----+----+
   |         |
+-------+ +-------+
| Shard1| | Shard2|  <- Data Shards
+-------+ +-------+
   |         |
+-------+ +-------+
| Rep   | | Rep   |  <- Replica Sets
+-------+ +-------+
```

### Shard Key เลือกอย่างไร

```crystal
# ตัวอย่าง Shard Key ที่ดี: user_id ที่กระจายข้อมูลได้สม่ำเสมอ
# สร้าง collection ที่รองรับ sharding

# ใน mongo shell:
# sh.enableSharding("myapp")
# sh.shardCollection("myapp.users", { "user_id": 1 })

# ใน Crystal: สร้าง index บน shard key
collection.create_index(
  keys: {"user_id" => 1},
  options: {unique: true}
)
```

## CRUD Operations

### Create - การเพิ่มข้อมูล

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# Insert เอกสารเดียว
result = products.insert_one({
  name:        "Crystal Programming Book",
  price:       599.0,
  category:    "programming",
  tags:        ["crystal", "backend", "systems"],
  stock:       100,
  created_at:  Time.utc
})

puts "สร้างเอกสาร ID: #{result.inserted_id}"

# Insert หลายเอกสาร
docs = [
  {name: "Ruby on Rails", price: 499.0, category: "web", stock: 50},
  {name: "Rust Systems", price: 649.0, category: "systems", stock: 30},
  {name: "Go Microservices", price: 549.0, category: "backend", stock: 75},
]

many_result = products.insert_many(docs)
puts "เพิ่ม #{many_result.inserted_count} เอกสาร"
```

### Read - การอ่านข้อมูล

```crystal
require "mongo"
require "json"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# หาเอกสารทั้งหมด
puts "=== สินค้าทั้งหมด ==="
products.find.each do |doc|
  puts "- #{doc["name"]}: ฿#{doc["price"]}"
end

# หาด้วยเงื่อนไข
puts "\n=== สินค้าราคาต่ำกว่า 600 ==="
products.find({"price" => {"$lt" => 600.0}}).each do |doc|
  puts "- #{doc["name"]}: ฿#{doc["price"]}"
end

# หาเอกสารเดียว
book = products.find_one({"name" => "Crystal Programming Book"})
if book
  puts "\nพบสินค้า: #{book["name"]}"
  puts "ราคา: ฿#{book["price"]}"
  puts "หมวด: #{book["category"]}"
end

# นับจำนวนเอกสาร
total = products.count_documents
puts "\nสินค้าทั้งหมด: #{total} รายการ"

# นับด้วยเงื่อนไข
programming_count = products.count_documents({"category" => "programming"})
puts "หมวด programming: #{programming_count} รายการ"
```

### Update - การอัปเดต

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# อัปเดตเอกสารเดียว
result = products.update_one(
  {"name" => "Crystal Programming Book"},
  {"$set" => {"price" => 649.0, "updated_at" => Time.utc}}
)
puts "อัปเดต #{result.modified_count} เอกสาร"

# อัปเดตหลายเอกสาร
result = products.update_many(
  {"category" => "programming"},
  {"$inc" => {"stock" => 10}}  # เพิ่ม stock 10 ชิ้น
)
puts "อัปเดต #{result.modified_count} เอกสาร"

# Upsert - อัปเดตถ้ามี หรือสร้างใหม่ถ้าไม่มี
result = products.update_one(
  {"name" => "Elixir Phoenix"},
  {"$set" => {
    "name"     => "Elixir Phoenix",
    "price"    => 579.0,
    "category" => "web",
    "stock"    => 40
  }},
  upsert: true
)

if result.upserted_id
  puts "สร้างใหม่: #{result.upserted_id}"
else
  puts "อัปเดต: #{result.modified_count}"
end

# Replace เอกสารทั้งหมด
products.replace_one(
  {"name" => "Rust Systems"},
  {
    name:     "Rust Systems Programming",
    price:    699.0,
    category: "systems",
    stock:    25,
    edition:  2
  }
)
puts "Replace สำเร็จ"
```

### Delete - การลบข้อมูล

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# ลบเอกสารเดียว
result = products.delete_one({"stock" => {"$eq" => 0}})
puts "ลบ #{result.deleted_count} เอกสาร"

# ลบหลายเอกสาร
result = products.delete_many({"category" => "outdated"})
puts "ลบ #{result.deleted_count} เอกสาร"

# ลบทั้งหมด (ระวัง!)
# products.delete_many({})
```

## Queries ขั้นสูง

### Comparison Operators

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# $eq, $ne, $gt, $gte, $lt, $lte
puts "=== ราคา 500-650 ==="
products.find({
  "price" => {
    "$gte" => 500.0,
    "$lte" => 650.0
  }
}).each { |doc| puts "  #{doc["name"]}: ฿#{doc["price"]}" }

# $in, $nin
puts "\n=== หมวด programming หรือ web ==="
products.find({
  "category" => {"$in" => ["programming", "web"]}
}).each { |doc| puts "  #{doc["name"]}" }

puts "\n=== ไม่ใช่หมวด systems ==="
products.find({
  "category" => {"$nin" => ["systems"]}
}).each { |doc| puts "  #{doc["name"]}" }
```

### Logical Operators

```crystal
# $and
products.find({
  "$and" => [
    {"price" => {"$gt" => 500.0}},
    {"stock" => {"$gt" => 30}}
  ]
}).each { |doc| puts "#{doc["name"]}" }

# $or
products.find({
  "$or" => [
    {"category" => "programming"},
    {"price" => {"$lt" => 500.0}}
  ]
}).each { |doc| puts "#{doc["name"]}" }

# $not
products.find({
  "price" => {"$not" => {"$gt" => 600.0}}
}).each { |doc| puts "#{doc["name"]}" }

# $nor
products.find({
  "$nor" => [
    {"category" => "systems"},
    {"price" => {"$lt" => 500.0}}
  ]
}).each { |doc| puts "#{doc["name"]}" }
```

### Array Queries

```crystal
# หาเอกสารที่ tags มี "crystal"
products.find({"tags" => "crystal"}).each do |doc|
  puts "#{doc["name"]} - tags: #{doc["tags"]}"
end

# $all - ต้องมีทุก element ที่ระบุ
products.find({
  "tags" => {"$all" => ["crystal", "backend"]}
}).each { |doc| puts doc["name"] }

# $size - array มีขนาดเท่าที่ระบุ
products.find({
  "tags" => {"$size" => 3}
}).each { |doc| puts doc["name"] }

# $elemMatch - element ตรงตามเงื่อนไขหลายข้อ
products.find({
  "reviews" => {
    "$elemMatch" => {
      "rating" => {"$gte" => 4},
      "verified" => true
    }
  }
}).each { |doc| puts doc["name"] }
```

### Projection - เลือก fields

```crystal
# แสดงเฉพาะ name และ price
products.find(
  filter: {},
  projection: {"name" => 1, "price" => 1, "_id" => 0}
).each do |doc|
  puts "#{doc["name"]}: ฿#{doc["price"]}"
end

# ซ่อน field บางอย่าง
products.find(
  filter: {},
  projection: {"stock" => 0, "created_at" => 0}
).each do |doc|
  puts doc.to_json
end
```

### Sort, Skip, Limit

```crystal
# เรียงลำดับราคาจากน้อยไปมาก
products.find.sort({"price" => 1}).each do |doc|
  puts "#{doc["name"]}: ฿#{doc["price"]}"
end

# Pagination
page = 2
per_page = 5

products
  .find
  .sort({"name" => 1})
  .skip((page - 1) * per_page)
  .limit(per_page)
  .each do |doc|
    puts doc["name"]
  end
```

## Aggregation Pipeline

Aggregation pipeline คือการประมวลผลข้อมูลแบบ stage-by-stage

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# Pipeline ง่ายๆ: จัดกลุ่มตาม category
pipeline = [
  # Stage 1: กรองสินค้าที่มี stock > 0
  {"$match" => {"stock" => {"$gt" => 0}}},

  # Stage 2: จัดกลุ่มตาม category
  {
    "$group" => {
      "_id"       => "$category",
      "count"     => {"$sum" => 1},
      "avg_price" => {"$avg" => "$price"},
      "max_price" => {"$max" => "$price"},
      "min_price" => {"$min" => "$price"},
      "total"     => {"$sum" => "$price"}
    }
  },

  # Stage 3: เรียงตาม count
  {"$sort" => {"count" => -1}},

  # Stage 4: Rename fields
  {
    "$project" => {
      "category"  => "$_id",
      "count"     => 1,
      "avg_price" => {"$round" => ["$avg_price", 2]},
      "max_price" => 1,
      "min_price" => 1,
      "_id"       => 0
    }
  }
]

puts "=== สรุปตาม Category ==="
products.aggregate(pipeline).each do |doc|
  puts "#{doc["category"]}: #{doc["count"]} รายการ, ราคาเฉลี่ย ฿#{doc["avg_price"]}"
end
```

### Aggregation ขั้นสูง

```crystal
# $lookup - JOIN กับ collection อื่น
orders = db["orders"]

pipeline = [
  {
    "$lookup" => {
      "from"         => "products",
      "localField"   => "product_id",
      "foreignField" => "_id",
      "as"           => "product_info"
    }
  },
  {
    "$unwind" => "$product_info"
  },
  {
    "$project" => {
      "order_id"     => 1,
      "product_name" => "$product_info.name",
      "quantity"     => 1,
      "total_price"  => {"$multiply" => ["$quantity", "$product_info.price"]}
    }
  }
]

orders.aggregate(pipeline).each do |doc|
  puts "Order #{doc["order_id"]}: #{doc["product_name"]} x#{doc["quantity"]} = ฿#{doc["total_price"]}"
end

# $facet - หลาย aggregation ในครั้งเดียว
facet_pipeline = [
  {
    "$facet" => {
      "by_category" => [
        {"$group" => {"_id" => "$category", "count" => {"$sum" => 1}}}
      ],
      "price_ranges" => [
        {
          "$bucket" => {
            "groupBy"    => "$price",
            "boundaries" => [0, 300, 500, 700, 1000],
            "default"    => "Other",
            "output"     => {"count" => {"$sum" => 1}}
          }
        }
      ],
      "top_products" => [
        {"$sort" => {"stock" => -1}},
        {"$limit" => 3},
        {"$project" => {"name" => 1, "stock" => 1, "_id" => 0}}
      ]
    }
  }
]

result = products.aggregate(facet_pipeline).first
if result
  puts "หมวดหมู่: #{result["by_category"]}"
  puts "ช่วงราคา: #{result["price_ranges"]}"
  puts "สินค้าเด่น: #{result["top_products"]}"
end
```

## Indexes

Indexes ช่วยเร่งความเร็วในการ query

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

# Single field index
products.create_index({"price" => 1})

# Compound index
products.create_index({"category" => 1, "price" => -1})

# Unique index
products.create_index(
  {"name" => 1},
  options: {unique: true}
)

# Text index สำหรับ full-text search
products.create_index({"name" => "text", "description" => "text"})

# TTL index - ลบอัตโนมัติหลังจากเวลาที่กำหนด
sessions = db["sessions"]
sessions.create_index(
  {"created_at" => 1},
  options: {expire_after_seconds: 3600}  # หมดอายุหลัง 1 ชั่วโมง
)

# Sparse index - index เฉพาะเอกสารที่มี field นั้น
products.create_index(
  {"discount" => 1},
  options: {sparse: true}
)

# Partial index - index เฉพาะเอกสารที่ตรงเงื่อนไข
products.create_index(
  {"price" => 1},
  options: {
    partial_filter_expression: {"stock" => {"$gt" => 0}}
  }
)

# ดู indexes ทั้งหมด
puts "=== Indexes ==="
products.list_indexes.each do |idx|
  puts "  #{idx["name"]}: #{idx["key"]}"
end

# ลบ index
products.drop_index("price_1")
```

### Full-text Search

```crystal
# ค้นหาด้วย text index
results = products.find({
  "$text" => {"$search" => "crystal programming"}
})

results.each do |doc|
  puts "#{doc["name"]}"
end

# ค้นหาพร้อม relevance score
results_with_score = products.find(
  filter: {"$text" => {"$search" => "systems programming"}},
  projection: {
    "name"  => 1,
    "score" => {"$meta" => "textScore"}
  }
).sort({"score" => {"$meta" => "textScore"}})

results_with_score.each do |doc|
  puts "#{doc["name"]} (score: #{doc["score"]})"
end
```

## Connection Pooling

```crystal
require "mongo"

# กำหนดค่า connection pool
client = Mongo::Client.new(
  "mongodb://localhost:27017",
  options: {
    max_pool_size:     10,
    min_pool_size:     2,
    max_idle_time_ms:  30000,
    connect_timeout_ms: 5000,
    server_selection_timeout_ms: 30000
  }
)

# ใช้ในหลาย fiber พร้อมกัน
channel = Channel(String).new

10.times do |i|
  spawn do
    db = client["store"]
    result = db["products"].count_documents
    channel.send("Fiber #{i}: #{result} products")
  end
end

10.times { puts channel.receive }
```

## Error Handling

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")
db = client["store"]
products = db["products"]

begin
  # พยายามเพิ่มเอกสารที่ duplicate key
  products.insert_one({"_id" => "existing_id", "name" => "Test"})
rescue Mongo::Error::WriteErrors => e
  e.write_errors.each do |err|
    if err["code"]? == 11000
      puts "Duplicate key error: #{err["errmsg"]}"
    else
      puts "Write error: #{err["errmsg"]}"
    end
  end
rescue Mongo::Error => e
  puts "MongoDB error: #{e.message}"
rescue IO::Error => e
  puts "Connection error: #{e.message}"
end

# Retry logic
def with_retry(max_retries = 3, &block)
  retries = 0
  loop do
    return block.call
  rescue Mongo::Error => e
    retries += 1
    if retries >= max_retries
      raise e
    end
    puts "Retry #{retries}/#{max_retries}: #{e.message}"
    sleep(0.5 * retries)
  end
end

with_retry do
  products.find.count
end
```

## Transactions

```crystal
require "mongo"

client = Mongo::Client.new("mongodb://localhost:27017")

# Transactions ต้องการ MongoDB 4.0+ และ Replica Set
client.start_session do |session|
  session.start_transaction

  begin
    db = client["store"]

    # หักสินค้าคงคลัง
    db["products"].update_one(
      {"_id" => "product_123", "stock" => {"$gte" => 1}},
      {"$inc" => {"stock" => -1}},
      session: session
    )

    # สร้าง order
    db["orders"].insert_one(
      {
        product_id: "product_123",
        quantity:   1,
        status:     "confirmed",
        created_at: Time.utc
      },
      session: session
    )

    session.commit_transaction
    puts "Transaction สำเร็จ"
  rescue ex
    session.abort_transaction
    puts "Transaction ยกเลิก: #{ex.message}"
    raise ex
  end
end
```

## Real-World Example: User Management

```crystal
require "mongo"
require "crypto/bcrypt/password"

class UserRepository
  def initialize(@collection : Mongo::Collection)
    setup_indexes
  end

  private def setup_indexes
    @collection.create_index({"email" => 1}, options: {unique: true})
    @collection.create_index({"username" => 1}, options: {unique: true})
    @collection.create_index({"created_at" => -1})
  end

  def create(email : String, username : String, password : String)
    hashed = Crypto::Bcrypt::Password.create(password)

    @collection.insert_one({
      email:      email,
      username:   username,
      password:   hashed.to_s,
      role:       "user",
      active:     true,
      created_at: Time.utc,
      updated_at: Time.utc
    })
  end

  def authenticate(email : String, password : String)
    user = @collection.find_one({"email" => email, "active" => true})
    return nil unless user

    hashed = Crypto::Bcrypt::Password.new(user["password"].as(String))
    return nil unless hashed.verify(password)

    user
  end

  def find_by_id(id : String)
    @collection.find_one({"_id" => BSON::ObjectId.new(id)})
  end

  def update_role(id : String, role : String)
    @collection.update_one(
      {"_id" => BSON::ObjectId.new(id)},
      {"$set" => {"role" => role, "updated_at" => Time.utc}}
    )
  end

  def deactivate(id : String)
    @collection.update_one(
      {"_id" => BSON::ObjectId.new(id)},
      {"$set" => {"active" => false, "updated_at" => Time.utc}}
    )
  end

  def list(page : Int32 = 1, per_page : Int32 = 20)
    @collection
      .find({"active" => true})
      .sort({"created_at" => -1})
      .skip((page - 1) * per_page)
      .limit(per_page)
      .to_a
  end
end

# ใช้งาน
client = Mongo::Client.new("mongodb://localhost:27017")
users = UserRepository.new(client["app"]["users"])

# สร้างผู้ใช้
users.create("alice@example.com", "alice", "secure_password_123")

# ยืนยันตัวตน
user = users.authenticate("alice@example.com", "secure_password_123")
puts user ? "Login สำเร็จ: #{user["username"]}" : "Login ล้มเหลว"
```

## แบบฝึกหัด

1. สร้างระบบ blog ที่มี collection: posts, comments, users และสามารถ query หา posts ที่ได้รับ comments มากที่สุด
2. เขียน aggregation pipeline ที่คำนวณยอดขายรายเดือนและแสดงเป็นกราฟข้อมูล
3. สร้าง index strategy สำหรับระบบ e-commerce ที่ต้องการ query: ตาม category, ราคา, และ full-text search
4. ทำ transaction สำหรับการโอนเงินระหว่างบัญชี

## สรุป

MongoDB ใน Crystal ให้ความสามารถ:
- **CRUD ครบครัน**: insert, find, update, delete ทั้งแบบเดียวและหลายเอกสาร
- **Queries ยืดหยุ่น**: comparison, logical, array operators
- **Aggregation Pipeline**: จัดกลุ่ม, join, transform ข้อมูลได้ซับซ้อน
- **Indexes หลากหลาย**: single, compound, text, TTL, partial
- **Transactions**: ACID transactions บน Replica Set
- **Connection Pooling**: จัดการ connections อย่างมีประสิทธิภาพ

MongoDB เหมาะกับข้อมูลที่โครงสร้างไม่แน่นอน หรือต้องการขยายตัวในแนวนอนด้วย sharding
