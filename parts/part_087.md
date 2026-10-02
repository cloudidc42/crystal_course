# Part 87: Data Serialization ขั้นสูง

## บทนำ

การ serialize ข้อมูลในระดับ production ต้องคำนึงถึงมากกว่าแค่การแปลงข้อมูลไป/มา เราต้องจัดการกับ schema versioning, migration, และ performance optimization

## Custom Serializers

### สร้าง Custom JSON Serializer

```crystal
require "json"

# Base serializer interface
module Serializer(T)
  abstract def serialize(value : T) : String
  abstract def deserialize(data : String) : T
end

# Custom Money type ที่ต้องการ serialize แบบพิเศษ
struct Money
  getter amount_satang : Int64
  getter currency : String

  def initialize(amount : Float64, @currency = "THB")
    @amount_satang = (amount * 100).round.to_i64
  end

  def amount : Float64
    @amount_satang / 100.0
  end

  def to_s : String
    "#{amount} #{currency}"
  end
end

# Custom serializer สำหรับ Money
class MoneySerializer
  include Serializer(Money)

  def serialize(value : Money) : String
    JSON.build do |json|
      json.object do
        json.field "amount_satang", value.amount_satang
        json.field "currency", value.currency
      end
    end
  end

  def deserialize(data : String) : Money
    parsed = JSON.parse(data)
    amount_satang = parsed["amount_satang"].as_i64
    currency = parsed["currency"].as_s

    money = Money.allocate
    # ใช้ pointer เพื่อ initialize struct ที่ allocate แล้ว
    money
  end
end

# ทดสอบ
price = Money.new(1250.99)
serializer = MoneySerializer.new
json_str = serializer.serialize(price)
puts json_str  # => {"amount_satang":125099,"currency":"THB"}
```

### Converter Pattern

```crystal
require "json"

# Converter สำหรับ custom types
module TimeConverter
  def self.from_json(pull : JSON::PullParser) : Time
    unix = pull.read_int
    Time.unix(unix)
  end

  def self.to_json(value : Time, builder : JSON::Builder)
    builder.number(value.to_unix)
  end
end

module DateConverter
  def self.from_json(pull : JSON::PullParser) : Time
    str = pull.read_string
    Time.parse_utc(str, "%Y-%m-%d")
  end

  def self.to_json(value : Time, builder : JSON::Builder)
    builder.string(value.to_s("%Y-%m-%d"))
  end
end

struct Event
  include JSON::Serializable

  property name : String

  @[JSON::Field(converter: TimeConverter)]
  property created_at : Time

  @[JSON::Field(converter: DateConverter)]
  property event_date : Time

  def initialize(@name, @created_at, @event_date)
  end
end

event = Event.new(
  "งานสัมมนา Crystal",
  Time.utc,
  Time.utc(2024, 6, 15)
)

json = event.to_json
puts json

restored = Event.from_json(json)
puts restored.created_at
puts restored.event_date.to_s("%Y-%m-%d")
```

## Versioned Serialization

### Schema Versioning ด้วย JSON

```crystal
require "json"

# Version 1 ของ UserProfile
struct UserProfileV1
  include JSON::Serializable

  property id : Int32
  property name : String
  property email : String

  def initialize(@id, @name, @email)
  end

  def migrate_to_v2 : UserProfileV2
    UserProfileV2.new(
      id: @id,
      first_name: @name.split(" ").first,
      last_name: @name.split(" ").last? || "",
      email: @email,
      created_at: Time.utc.to_unix
    )
  end
end

# Version 2 ของ UserProfile (เพิ่ม fields)
struct UserProfileV2
  include JSON::Serializable

  property id : Int32
  property first_name : String
  property last_name : String
  property email : String
  property created_at : Int64
  property avatar_url : String?

  def initialize(@id, @first_name, @last_name, @email, @created_at, @avatar_url = nil)
  end
end

# Versioned container
struct VersionedData
  include JSON::Serializable

  property schema_version : Int32
  property data : JSON::Any

  def initialize(@schema_version, @data)
  end
end

# Serialize กับ version
v1_user = UserProfileV1.new(1, "สมชาย ใจดี", "somchai@example.com")
versioned = VersionedData.new(1, JSON.parse(v1_user.to_json))
puts versioned.to_json
```

### Schema Migration System

```crystal
require "json"

# Migration interface
abstract class Migration
  abstract def version : Int32
  abstract def up(data : JSON::Any) : JSON::Any
end

# Migration จาก V1 ไป V2
class MigrateV1ToV2 < Migration
  def version : Int32
    2
  end

  def up(data : JSON::Any) : JSON::Any
    obj = data.as_h.dup

    # แยก name เป็น first_name + last_name
    if name = obj["name"]?.try(&.as_s)
      parts = name.split(" ", 2)
      obj["first_name"] = JSON::Any.new(parts[0])
      obj["last_name"] = JSON::Any.new(parts[1]? || "")
      obj.delete("name")
    end

    # เพิ่ม created_at ถ้าไม่มี
    unless obj["created_at"]?
      obj["created_at"] = JSON::Any.new(Time.utc.to_unix)
    end

    JSON::Any.new(obj)
  end
end

# Migration จาก V2 ไป V3
class MigrateV2ToV3 < Migration
  def version : Int32
    3
  end

  def up(data : JSON::Any) : JSON::Any
    obj = data.as_h.dup

    # เพิ่ม preferences object
    unless obj["preferences"]?
      preferences = {
        "language"       => JSON::Any.new("th"),
        "notifications"  => JSON::Any.new(true),
        "theme"          => JSON::Any.new("light"),
      }
      obj["preferences"] = JSON::Any.new(preferences)
    end

    JSON::Any.new(obj)
  end
end

# Migration manager
class MigrationManager
  def initialize
    @migrations = Array(Migration).new
  end

  def register(migration : Migration)
    @migrations << migration
    @migrations.sort_by!(&.version)
  end

  def migrate(data : JSON::Any, from_version : Int32, to_version : Int32) : JSON::Any
    current = data
    current_version = from_version

    @migrations.each do |migration|
      next if migration.version <= current_version
      break if migration.version > to_version

      current = migration.up(current)
      current_version = migration.version
    end

    current
  end
end

# ทดสอบ
manager = MigrationManager.new
manager.register(MigrateV1ToV2.new)
manager.register(MigrateV2ToV3.new)

v1_data = JSON.parse(%({
  "id": 1,
  "name": "สมชาย ใจดี",
  "email": "somchai@example.com"
}))

migrated = manager.migrate(v1_data, from_version: 1, to_version: 3)
puts migrated.to_json
```

## Serialization for Caching

### Cache-Optimized Serialization

```crystal
require "json"
require "compress/zlib"

module CacheSerializer
  # Serialize + compress สำหรับ cache
  def self.pack(data : JSON::Serializable.class, value) : Bytes
    json_str = value.to_json
    compress(json_str.to_slice)
  end

  def self.unpack(klass : T.class, bytes : Bytes) : T forall T
    json_bytes = decompress(bytes)
    T.from_json(String.new(json_bytes))
  end

  private def self.compress(data : Bytes) : Bytes
    io = IO::Memory.new
    Compress::Zlib::Writer.open(io) do |writer|
      writer.write(data)
    end
    io.to_slice
  end

  private def self.decompress(data : Bytes) : Bytes
    io = IO::Memory.new(data)
    Compress::Zlib::Reader.open(io) do |reader|
      reader.gets_to_end.to_slice
    end
  end
end

struct CachedProduct
  include JSON::Serializable

  property id : Int32
  property name : String
  property description : String
  property price : Float64
  property stock : Int32

  def initialize(@id, @name, @description, @price, @stock)
  end
end

product = CachedProduct.new(
  1,
  "โน้ตบุ๊ค Crystal",
  "โน้ตบุ๊คสำหรับนักพัฒนา Crystal ผู้เชี่ยวชาญ รองรับการทำงาน concurrency",
  45000.0,
  50
)

# Pack สำหรับ cache
cached = CacheSerializer.pack(CachedProduct, product)
puts "Compressed size: #{cached.size} bytes"
puts "Original JSON size: #{product.to_json.bytesize} bytes"

# Unpack จาก cache
restored = CacheSerializer.unpack(CachedProduct, cached)
puts restored.name
```

### Cache Entry ที่รองรับ TTL และ Metadata

```crystal
require "json"

struct CacheMetadata
  include JSON::Serializable

  property version : Int32
  property created_at : Int64
  property expires_at : Int64
  property hit_count : Int32
  property tags : Array(String)

  def initialize(
    @version : Int32 = 1,
    ttl_seconds : Int32 = 3600,
    @tags : Array(String) = [] of String
  )
    @created_at = Time.utc.to_unix
    @expires_at = @created_at + ttl_seconds
    @hit_count = 0
  end

  def expired? : Bool
    Time.utc.to_unix >= @expires_at
  end

  def remaining_ttl : Int64
    [@expires_at - Time.utc.to_unix, 0_i64].max
  end
end

class CacheEntry(T)
  include JSON::Serializable

  property metadata : CacheMetadata
  property data : T

  def initialize(@data : T, ttl : Int32 = 3600, tags : Array(String) = [] of String)
    @metadata = CacheMetadata.new(ttl_seconds: ttl, tags: tags)
  end

  def valid? : Bool
    !@metadata.expired?
  end

  def record_hit!
    @metadata.hit_count += 1
  end
end

# ใช้งาน
struct UserSession
  include JSON::Serializable

  property user_id : Int32
  property username : String
  property permissions : Array(String)

  def initialize(@user_id, @username, @permissions)
  end
end

session = UserSession.new(42, "admin", ["read", "write", "delete"])
entry = CacheEntry(UserSession).new(session, ttl: 1800, tags: ["user", "auth"])

# Serialize
json = entry.to_json
puts json

# Deserialize
restored = CacheEntry(UserSession).from_json(json)
puts "Valid: #{restored.valid?}"
puts "TTL remaining: #{restored.metadata.remaining_ttl}s"
puts "Username: #{restored.data.username}"
```

## Lazy Serialization

```crystal
require "json"

# Lazy-load ข้อมูล heavy เฉพาะเมื่อต้องการ
class LazyDocument
  @data : JSON::Any?
  @raw_json : String

  def initialize(@raw_json)
    @data = nil
  end

  def [](key : String) : JSON::Any
    parse_if_needed[key]
  end

  def []?(key : String) : JSON::Any?
    parse_if_needed[key]?
  end

  def to_json : String
    @raw_json
  end

  private def parse_if_needed : JSON::Any
    @data ||= JSON.parse(@raw_json)
  end
end

# ใช้งาน
raw = %({ "id": 1, "name": "test", "data": [1,2,3,4,5] })
doc = LazyDocument.new(raw)

# ไม่ parse จนกว่าจะเข้าถึง
puts doc["name"]  # parse ตอนนี้
puts doc["id"]    # ใช้ cached parse result
```

## Differential Serialization

```crystal
require "json"

# Serialize เฉพาะ fields ที่เปลี่ยนแปลง
class Trackable
  macro property_tracked(name, type)
    @{{name.id}} : {{type}}
    @{{name.id}}_changed : Bool = false

    def {{name.id}}=(@{{name.id}} : {{type}})
      @{{name.id}}_changed = true
    end

    def {{name.id}} : {{type}}
      @{{name.id}}
    end
  end
end

struct DirtyTracker
  getter changes : Hash(String, JSON::Any)

  def initialize
    @changes = Hash(String, JSON::Any).new
    @originals = Hash(String, JSON::Any).new
  end

  def track(field : String, old_value : JSON::Any, new_value : JSON::Any)
    @originals[field] = old_value unless @originals.has_key?(field)
    if new_value != @originals[field]
      @changes[field] = new_value
    else
      @changes.delete(field)
    end
  end

  def dirty? : Bool
    !@changes.empty?
  end

  def diff_json : String
    @changes.to_json
  end

  def reset!
    @changes.clear
    @originals.clear
  end
end

# ใช้งาน
tracker = DirtyTracker.new

# Simulate การเปลี่ยนแปลง
tracker.track("name", JSON::Any.new("สมชาย"), JSON::Any.new("สมหญิง"))
tracker.track("age", JSON::Any.new(25_i64), JSON::Any.new(26_i64))

puts tracker.dirty?      # => true
puts tracker.diff_json   # => {"name":"สมหญิง","age":26}
```

## Polymorphic Serialization

```crystal
require "json"

# Base type สำหรับ polymorphic serialization
abstract class Shape
  include JSON::Serializable

  use_json_discriminator "type", {
    "circle"    => Circle,
    "rectangle" => Rectangle,
    "triangle"  => Triangle,
  }

  abstract def area : Float64
  abstract def perimeter : Float64
end

class Circle < Shape
  property type = "circle"
  property radius : Float64

  def initialize(@radius)
  end

  def area : Float64
    Math::PI * @radius ** 2
  end

  def perimeter : Float64
    2 * Math::PI * @radius
  end
end

class Rectangle < Shape
  property type = "rectangle"
  property width : Float64
  property height : Float64

  def initialize(@width, @height)
  end

  def area : Float64
    @width * @height
  end

  def perimeter : Float64
    2 * (@width + @height)
  end
end

class Triangle < Shape
  property type = "triangle"
  property a : Float64
  property b : Float64
  property c : Float64

  def initialize(@a, @b, @c)
  end

  def area : Float64
    s = (@a + @b + @c) / 2
    Math.sqrt(s * (s - @a) * (s - @b) * (s - @c))
  end

  def perimeter : Float64
    @a + @b + @c
  end
end

# ทดสอบ polymorphic serialization
shapes = [
  Circle.new(5.0),
  Rectangle.new(4.0, 6.0),
  Triangle.new(3.0, 4.0, 5.0),
] of Shape

json = shapes.to_json
puts json

# Deserialize กลับมาเป็น correct types
restored = Array(Shape).from_json(json)
restored.each do |shape|
  puts "#{shape.class}: area=#{shape.area.round(2)}, perimeter=#{shape.perimeter.round(2)}"
end
```

## Serialization Pipeline

```crystal
require "json"

# Middleware pattern สำหรับ serialization
abstract class SerializationMiddleware
  abstract def before_serialize(data : String) : String
  abstract def after_deserialize(data : String) : String
end

class EncryptionMiddleware < SerializationMiddleware
  def initialize(@key : String)
  end

  def before_serialize(data : String) : String
    # Simple XOR encryption (ใช้แค่เป็นตัวอย่าง)
    result = data.bytes.map_with_index do |byte, i|
      (byte ^ @key[i % @key.size].ord).chr
    end
    Base64.encode(result.join)
  end

  def after_deserialize(data : String) : String
    decoded = Base64.decode_string(data)
    decoded.bytes.map_with_index do |byte, i|
      (byte ^ @key[i % @key.size].ord).chr
    end.join
  end
end

class CompressionMiddleware < SerializationMiddleware
  def before_serialize(data : String) : String
    # ใน production ใช้ Compress::Zlib
    data  # placeholder
  end

  def after_deserialize(data : String) : String
    data  # placeholder
  end
end

class SerializationPipeline
  def initialize
    @middlewares = [] of SerializationMiddleware
  end

  def use(middleware : SerializationMiddleware) : self
    @middlewares << middleware
    self
  end

  def serialize(data : String) : String
    @middlewares.reduce(data) do |result, middleware|
      middleware.before_serialize(result)
    end
  end

  def deserialize(data : String) : String
    @middlewares.reverse.reduce(data) do |result, middleware|
      middleware.after_deserialize(result)
    end
  end
end

# ใช้งาน
pipeline = SerializationPipeline.new
pipeline.use(EncryptionMiddleware.new("secret-key-123"))

original = %({ "password": "my_secret", "user": "admin" })
encrypted = pipeline.serialize(original)
decrypted = pipeline.deserialize(encrypted)

puts "Original: #{original}"
puts "Encrypted: #{encrypted}"
puts "Decrypted: #{decrypted}"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Document Versioning
สร้างระบบ document versioning ที่:
- เก็บ history ของการเปลี่ยนแปลง
- สามารถ rollback ไป version ก่อนหน้าได้
- serialize/deserialize ทั้ง history

### แบบฝึกหัดที่ 2: Multi-format Serializer
สร้าง `MultiFormatSerializer` ที่รองรับทั้ง JSON และ MessagePack:
- Auto-detect format จาก header/magic bytes
- Serialize ไปทั้งสอง format
- Benchmark performance

### แบบฝึกหัดที่ 3: Schema Evolution
ออกแบบ Product schema ที่ evolve จาก V1 ไป V4:
- V1: id, name, price
- V2: เพิ่ม category, stock
- V3: แยก price เป็น base_price, tax_rate
- V4: เพิ่ม variants (Array ของ ProductVariant)
- สร้าง migration code และ test

### แบบฝึกหัดที่ 4: Cache-Aside Pattern
Implement cache-aside pattern ที่:
- Check cache ก่อน (ใช้ in-memory dict)
- Cache miss → load จาก "database" (mock)
- Serialize ด้วย JSON + compression
- Handle cache invalidation ด้วย tags

## สรุป

Data serialization ขั้นสูงใน Crystal ครอบคลุม:
- **Custom Serializers**: ควบคุม format ได้อย่างละเอียด
- **Versioning**: รองรับการ evolve ของ schema ตามเวลา
- **Migration**: แปลงข้อมูลเก่าให้ compatible กับ schema ใหม่
- **Caching**: optimize serialization สำหรับ cache use case
- **Polymorphism**: serialize/deserialize types ที่ซับซ้อน

Key principles:
1. เสมอ versioning schema เพื่อ backward compatibility
2. ใช้ migration chain สำหรับการ evolve
3. Compress ข้อมูลขนาดใหญ่เมื่อ cache
4. ใช้ discriminator สำหรับ polymorphic types
