# Part 86: MessagePack ใน Crystal

## บทนำ

MessagePack เป็น format การ serialize ข้อมูลแบบ binary ที่มีประสิทธิภาพสูง เร็วกว่าและมีขนาดเล็กกว่า JSON ใน Crystal เราสามารถใช้ MessagePack ผ่าน shard `msgpack-crystal`

## การติดตั้ง

เพิ่ม dependency ใน `shard.yml`:

```yaml
dependencies:
  msgpack:
    github: crystal-community/msgpack-crystal
```

จากนั้นรัน:
```bash
shards install
```

## การใช้งานพื้นฐาน

```crystal
require "msgpack"

# Packing ข้อมูลพื้นฐาน
data = {"name" => "สมชาย", "age" => 30, "active" => true}

# Pack เป็น bytes
packed = data.to_msgpack
puts packed.class  # => Bytes
puts packed.size   # ขนาดน้อยกว่า JSON

# Unpack กลับมา
unpacked = MessagePack::Unpacker.new(packed)
result = Hash(String, MessagePack::Type).from_msgpack(packed)
puts result["name"]  # => "สมชาย"
puts result["age"]   # => 30
```

## Packing ประเภทข้อมูลต่างๆ

```crystal
require "msgpack"

# Integer
i = 42
packed_int = i.to_msgpack
puts Int32.from_msgpack(packed_int)  # => 42

# Float
f = 3.14
packed_float = f.to_msgpack
puts Float64.from_msgpack(packed_float)  # => 3.14

# String
s = "สวัสดี Crystal"
packed_str = s.to_msgpack
puts String.from_msgpack(packed_str)  # => "สวัสดี Crystal"

# Array
arr = [1, 2, 3, 4, 5]
packed_arr = arr.to_msgpack
puts Array(Int32).from_msgpack(packed_arr).inspect  # => [1, 2, 3, 4, 5]

# Nil
packed_nil = nil.to_msgpack
puts Nil.from_msgpack(packed_nil).inspect  # => nil

# Bool
packed_bool = true.to_msgpack
puts Bool.from_msgpack(packed_bool)  # => true
```

## MessagePack::Serializable

คล้ายกับ `JSON::Serializable` เราสามารถทำให้ struct/class รองรับ MessagePack ได้:

```crystal
require "msgpack"

struct User
  include MessagePack::Serializable

  property name : String
  property age : Int32
  property email : String

  def initialize(@name, @age, @email)
  end
end

# Serialize
user = User.new("สมหญิง", 25, "somying@example.com")
packed = user.to_msgpack

puts "Packed size: #{packed.size} bytes"

# Deserialize
restored = User.from_msgpack(packed)
puts restored.name   # => "สมหญิง"
puts restored.age    # => 25
puts restored.email  # => "somying@example.com"
```

## MessagePack::Serializable::Strict

ใช้เมื่อต้องการ reject unknown fields:

```crystal
require "msgpack"

struct StrictUser
  include MessagePack::Serializable
  include MessagePack::Serializable::Strict

  property id : Int32
  property name : String
end

# จะ raise exception ถ้ามี field ที่ไม่รู้จัก
```

## การกำหนดชื่อ Key ที่แตกต่างกัน

```crystal
require "msgpack"

struct Config
  include MessagePack::Serializable

  @[MessagePack::Field(key: "max_connections")]
  property max_connections : Int32

  @[MessagePack::Field(key: "timeout_ms")]
  property timeout : Int32

  def initialize(@max_connections, @timeout)
  end
end

config = Config.new(100, 5000)
packed = config.to_msgpack

restored = Config.from_msgpack(packed)
puts restored.max_connections  # => 100
puts restored.timeout          # => 5000
```

## การจัดการ Nil และ Optional Fields

```crystal
require "msgpack"

struct Profile
  include MessagePack::Serializable

  property name : String
  property bio : String?  # Optional field

  @[MessagePack::Field(emit_null: true)]
  property avatar : String?

  def initialize(@name, @bio = nil, @avatar = nil)
  end
end

# ไม่มี bio
p1 = Profile.new("สมชาย")
packed1 = p1.to_msgpack
restored1 = Profile.from_msgpack(packed1)
puts restored1.bio.inspect  # => nil

# มี bio
p2 = Profile.new("สมหญิง", "นักพัฒนา Crystal")
packed2 = p2.to_msgpack
restored2 = Profile.from_msgpack(packed2)
puts restored2.bio  # => "นักพัฒนา Crystal"
```

## Nested Structures

```crystal
require "msgpack"

struct Address
  include MessagePack::Serializable

  property street : String
  property city : String
  property country : String

  def initialize(@street, @city, @country)
  end
end

struct Person
  include MessagePack::Serializable

  property name : String
  property age : Int32
  property address : Address

  def initialize(@name, @age, @address)
  end
end

addr = Address.new("123 ถนนสุขุมวิท", "กรุงเทพ", "ไทย")
person = Person.new("สมชาย", 30, addr)

packed = person.to_msgpack
puts "Size: #{packed.size} bytes"

restored = Person.from_msgpack(packed)
puts restored.name              # => "สมชาย"
puts restored.address.city      # => "กรุงเทพ"
puts restored.address.country   # => "ไทย"
```

## Array ของ Structs

```crystal
require "msgpack"

struct Product
  include MessagePack::Serializable

  property id : Int32
  property name : String
  property price : Float64

  def initialize(@id, @name, @price)
  end
end

products = [
  Product.new(1, "สมุดโน้ต", 45.0),
  Product.new(2, "ดินสอ", 12.5),
  Product.new(3, "ปากกา", 25.0),
]

# Pack array
packed = products.to_msgpack
puts "Packed size: #{packed.size} bytes"

# Unpack
restored = Array(Product).from_msgpack(packed)
restored.each do |p|
  puts "#{p.name}: #{p.price} บาท"
end
```

## การเปรียบเทียบกับ JSON

```crystal
require "json"
require "msgpack"

struct DataPoint
  include JSON::Serializable
  include MessagePack::Serializable

  property timestamp : Int64
  property value : Float64
  property label : String

  def initialize(@timestamp, @value, @label)
  end
end

# สร้าง data points จำนวนมาก
data = Array(DataPoint).new(1000) do |i|
  DataPoint.new(
    timestamp: Time.utc.to_unix + i,
    value: rand(100.0),
    label: "sensor_#{i}"
  )
end

# วัดขนาด JSON
json_data = data.to_json
puts "JSON size: #{json_data.bytesize} bytes"

# วัดขนาด MessagePack
msgpack_data = data.to_msgpack
puts "MessagePack size: #{msgpack_data.size} bytes"

# คำนวณ ratio
ratio = json_data.bytesize.to_f / msgpack_data.size
puts "MessagePack เล็กกว่า #{ratio.round(2)}x"

# เปรียบเทียบความเร็ว
start = Time.monotonic

1000.times do
  Array(DataPoint).from_json(json_data)
end
json_time = Time.monotonic - start

start = Time.monotonic
1000.times do
  Array(DataPoint).from_msgpack(msgpack_data)
end
msgpack_time = Time.monotonic - start

puts "JSON parse time: #{json_time.total_milliseconds.round(2)}ms"
puts "MessagePack parse time: #{msgpack_time.total_milliseconds.round(2)}ms"
puts "MessagePack เร็วกว่า #{(json_time / msgpack_time).round(2)}x"
```

## การใช้ MessagePack ใน Network Protocol

```crystal
require "msgpack"
require "socket"

# Message types สำหรับ protocol
enum MessageType : UInt8
  Ping = 0
  Pong = 1
  Data = 2
  Error = 3
end

struct Message
  include MessagePack::Serializable

  property type : UInt8
  property id : UInt32
  property payload : Bytes?

  def initialize(@type, @id, @payload = nil)
  end
end

# Encode message
msg = Message.new(MessageType::Data.value, 12345_u32, "Hello".to_slice)
encoded = msg.to_msgpack

puts "Encoded size: #{encoded.size} bytes"

# Decode message
decoded = Message.from_msgpack(encoded)
puts "Type: #{decoded.type}"
puts "ID: #{decoded.id}"
puts "Payload: #{String.new(decoded.payload.not_nil!)}" if decoded.payload
```

## Streaming MessagePack

```crystal
require "msgpack"

# เขียน multiple messages ลง IO
io = IO::Memory.new

packer = MessagePack::Packer.new(io)
packer.write(1)
packer.write("hello")
packer.write([1, 2, 3])
packer.write({"key" => "value"})

# อ่านกลับ
io.rewind
unpacker = MessagePack::Unpacker.new(io)

puts unpacker.read_value.inspect  # => 1
puts unpacker.read_value.inspect  # => "hello"
puts unpacker.read_value.inspect  # => [1, 2, 3]
puts unpacker.read_value.inspect  # => {"key" => "value"}
```

## Custom Serialization ด้วย MessagePack

```crystal
require "msgpack"

# Custom type ที่ต้องการ serialize แบบพิเศษ
struct Money
  getter amount : Int64  # เก็บเป็น satang (1/100 บาท)
  getter currency : String

  def initialize(baht : Float64, @currency = "THB")
    @amount = (baht * 100).to_i64
  end

  def to_baht : Float64
    @amount / 100.0
  end

  def to_msgpack(packer : MessagePack::Packer)
    # Serialize เป็น [amount, currency]
    packer.write_array_start(2)
    packer.write(@amount)
    packer.write(@currency)
  end

  def self.from_msgpack(unpacker : MessagePack::Unpacker) : self
    # อ่าน array size
    size = unpacker.read_array_size
    raise "Invalid Money format" unless size == 2

    amount = unpacker.read(Int64)
    currency = unpacker.read(String)

    money = allocate
    money.initialize_from_components(amount, currency)
    money
  end

  protected def initialize_from_components(@amount, @currency)
  end
end

# ใช้งาน
price = Money.new(1250.50)
packed = price.to_msgpack

puts "Packed: #{packed.bytes}"

restored = Money.from_msgpack(packed)
puts "Amount: #{restored.to_baht} #{restored.currency}"  # => 1250.5 THB
```

## MessagePack Extensions (Ext types)

```crystal
require "msgpack"

# Custom Ext type สำหรับ Time
struct TimePacker
  EXT_TYPE = 1_i8

  def self.pack(time : Time) : Bytes
    io = IO::Memory.new
    packer = MessagePack::Packer.new(io)
    packer.write(time.to_unix)
    io.to_slice
  end

  def self.unpack(bytes : Bytes) : Time
    unpacker = MessagePack::Unpacker.new(bytes)
    unix = unpacker.read(Int64)
    Time.unix(unix)
  end
end

# ใช้ Ext type ใน custom serialization
struct Event
  include MessagePack::Serializable

  property name : String
  property created_at_unix : Int64

  def initialize(@name, created_at : Time)
    @created_at_unix = created_at.to_unix
  end

  def created_at : Time
    Time.unix(@created_at_unix)
  end
end

event = Event.new("ประชุมทีม", Time.utc)
packed = event.to_msgpack
restored = Event.from_msgpack(packed)

puts restored.name
puts restored.created_at
```

## MessagePack ใน Redis Cache

```crystal
require "msgpack"
# require "redis"  # ต้องติดตั้ง redis shard

struct CacheEntry(T)
  include MessagePack::Serializable

  property data : T
  property cached_at : Int64
  property ttl : Int32

  def initialize(@data : T, @ttl : Int32 = 3600)
    @cached_at = Time.utc.to_unix
  end

  def expired? : Bool
    Time.utc.to_unix - @cached_at > @ttl
  end
end

# ตัวอย่างการใช้ (ไม่ต้องมี Redis จริง)
struct UserProfile
  include MessagePack::Serializable

  property id : Int32
  property name : String
  property level : Int32

  def initialize(@id, @name, @level)
  end
end

profile = UserProfile.new(1, "สมชาย", 5)
entry = CacheEntry(UserProfile).new(profile, 300)

# Serialize สำหรับ cache
cached_bytes = entry.to_msgpack
puts "Cache entry size: #{cached_bytes.size} bytes"

# Deserialize จาก cache
restored_entry = CacheEntry(UserProfile).from_msgpack(cached_bytes)
puts "Expired: #{restored_entry.expired?}"
puts "User: #{restored_entry.data.name}"
```

## ประสิทธิภาพ - Benchmarking

```crystal
require "msgpack"
require "json"
require "benchmark"

struct BenchData
  include JSON::Serializable
  include MessagePack::Serializable

  property id : Int32
  property name : String
  property values : Array(Float64)
  property tags : Array(String)
  property metadata : Hash(String, String)

  def initialize
    @id = rand(1000)
    @name = "item_#{@id}"
    @values = Array(Float64).new(10) { rand(100.0) }
    @tags = ["tag1", "tag2", "tag3"]
    @metadata = {"key1" => "value1", "key2" => "value2"}
  end
end

data = Array(BenchData).new(100) { BenchData.new }
json_str = data.to_json
msgpack_bytes = data.to_msgpack

puts "JSON size: #{json_str.bytesize} bytes"
puts "MessagePack size: #{msgpack_bytes.size} bytes"
puts "Compression ratio: #{(json_str.bytesize.to_f / msgpack_bytes.size).round(2)}x"

Benchmark.ips do |x|
  x.report("JSON serialize") { data.to_json }
  x.report("MsgPack serialize") { data.to_msgpack }
  x.report("JSON deserialize") { Array(BenchData).from_json(json_str) }
  x.report("MsgPack deserialize") { Array(BenchData).from_msgpack(msgpack_bytes) }
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Student Grade System
สร้าง struct `Grade` และ `StudentReport` ที่รองรับ MessagePack serialization:
- `Grade` มี subject (String), score (Float64), grade_letter (String)
- `StudentReport` มี student_id, name, grades (Array(Grade)), semester
- serialize/deserialize และเปรียบเทียบขนาดกับ JSON

### แบบฝึกหัดที่ 2: Protocol Messages
ออกแบบ message protocol สำหรับระบบ chat:
- สร้าง enum `MessageKind` (Text, Image, File, System)
- สร้าง struct `ChatMessage` ที่มี kind, sender_id, content, timestamp
- ทดสอบ pack/unpack หลาย messages

### แบบฝึกหัดที่ 3: Cache System
สร้าง simple in-memory cache ที่ใช้ MessagePack:
- `Cache(K, V)` struct ที่ serialize/deserialize ด้วย MessagePack
- รองรับ TTL (time-to-live)
- save/load cache จาก file

### แบบฝึกหัดที่ 4: Streaming Data
เขียนโปรแกรมที่:
- Generate 10,000 data points
- เขียนลงไฟล์ด้วย MessagePack streaming
- อ่านกลับมาและ verify ข้อมูล
- เปรียบเทียบขนาดไฟล์กับ JSON

## สรุป

MessagePack ใน Crystal มีข้อดีหลัก:
- **ขนาดเล็กกว่า**: โดยทั่วไป 30-60% เล็กกว่า JSON
- **เร็วกว่า**: parse/serialize เร็วกว่า JSON อย่างมีนัยสำคัญ
- **Type-safe**: ด้วย `MessagePack::Serializable` ใช้งานง่ายและปลอดภัย
- **Binary format**: เหมาะสำหรับ network protocols, caching, และ storage

ควรใช้ MessagePack เมื่อ:
- ประสิทธิภาพเป็นสิ่งสำคัญ
- ไม่ต้องการให้มนุษย์อ่าน format ได้
- ต้องการประหยัด bandwidth หรือ storage
- ทำงานกับ high-throughput systems
