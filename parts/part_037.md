# Part 37: Abstract Classes

## บทนำ

Abstract Class เป็น class ที่ไม่สามารถ instantiate โดยตรงได้ มีจุดประสงค์เพื่อเป็น "สัญญา" (contract) ที่ subclasses ต้องปฏิบัติตาม ใน Crystal ใช้ keyword `abstract` สำหรับทั้ง class และ methods

---

## 1. Abstract Class พื้นฐาน

### 1.1 การนิยาม Abstract Class

```crystal
# Abstract class - ไม่สามารถ instantiate ได้โดยตรง
abstract class Shape
  # Abstract method - subclass ต้อง implement
  abstract def area : Float64
  abstract def perimeter : Float64

  # Concrete method - มี implementation ใน abstract class
  def describe
    puts "รูปร่าง: #{self.class.name}"
    puts "พื้นที่: #{area.round(2)}"
    puts "เส้นรอบรูป: #{perimeter.round(2)}"
  end

  def larger_than?(other : Shape) : Bool
    area > other.area
  end
end

# Shape.new  # => Error: cannot instantiate abstract class Shape
```

### 1.2 Concrete Subclasses

```crystal
abstract class Shape
  abstract def area : Float64
  abstract def perimeter : Float64

  def describe
    puts "#{self.class.name}: พื้นที่=#{area.round(2)}, รอบ=#{perimeter.round(2)}"
  end
end

class Circle < Shape
  def initialize(@radius : Float64)
  end

  def area : Float64
    Math::PI * @radius ** 2
  end

  def perimeter : Float64
    2 * Math::PI * @radius
  end
end

class Rectangle < Shape
  def initialize(@width : Float64, @height : Float64)
  end

  def area : Float64
    @width * @height
  end

  def perimeter : Float64
    2 * (@width + @height)
  end
end

class Triangle < Shape
  def initialize(@a : Float64, @b : Float64, @c : Float64)
  end

  def area : Float64
    # Heron's formula
    s = perimeter / 2
    Math.sqrt(s * (s - @a) * (s - @b) * (s - @c))
  end

  def perimeter : Float64
    @a + @b + @c
  end
end

shapes = [
  Circle.new(5.0),
  Rectangle.new(4.0, 6.0),
  Triangle.new(3.0, 4.0, 5.0),
]

shapes.each(&.describe)

puts "\nรูปที่ใหญ่ที่สุด:"
largest = shapes.max_by(&.area)
puts largest.class.name
```

---

## 2. Abstract Methods - การบังคับ Implementation

### 2.1 Abstract Methods คืออะไร

```crystal
abstract class Animal
  getter name : String

  def initialize(@name : String)
  end

  # Concrete methods - มี implementation
  def breathe
    puts "#{@name} หายใจ"
  end

  def eat(food : String)
    puts "#{@name} กิน #{food}"
  end

  # Abstract methods - subclass ต้อง implement
  abstract def speak : String
  abstract def move : String
  abstract def habitat : String
end

class Dog < Animal
  def initialize(name : String)
    super(name)
  end

  def speak : String
    "โฮ่ง! โฮ่ง!"
  end

  def move : String
    "วิ่งด้วยสี่ขา"
  end

  def habitat : String
    "ในบ้าน"
  end
end

class Eagle < Animal
  def initialize(name : String)
    super(name)
  end

  def speak : String
    "กี๊ด! กี๊ด!"
  end

  def move : String
    "บินด้วยปีก"
  end

  def habitat : String
    "บนยอดเขา"
  end
end

class Fish < Animal
  def initialize(name : String)
    super(name)
  end

  def speak : String
    "..." # ปลาไม่ส่งเสียง
  end

  def move : String
    "ว่ายน้ำด้วยครีบ"
  end

  def habitat : String
    "ในน้ำ"
  end
end

animals = [Dog.new("บัดดี้"), Eagle.new("อินทรี"), Fish.new("นีโม")]

animals.each do |animal|
  puts "\n#{animal.name}:"
  puts "  เสียง: #{animal.speak}"
  puts "  การเคลื่อนที่: #{animal.move}"
  puts "  ที่อยู่: #{animal.habitat}"
  animal.breathe  # inherited concrete method
end
```

### 2.2 Compile Error เมื่อไม่ implement abstract methods

```crystal
abstract class Vehicle
  abstract def fuel_type : String
  abstract def max_speed : Int32
end

# นี่จะทำให้เกิด compile error:
# class IncompleteVehicle < Vehicle
#   # ไม่ได้ implement fuel_type และ max_speed
# end

# ถูกต้อง: implement ทุก abstract method
class GasCar < Vehicle
  def fuel_type : String
    "น้ำมันเบนซิน"
  end

  def max_speed : Int32
    200
  end
end

class ElectricCar < Vehicle
  def fuel_type : String
    "ไฟฟ้า"
  end

  def max_speed : Int32
    250
  end
end
```

---

## 3. Abstract vs Module

### 3.1 เปรียบเทียบ Abstract Class กับ Module

```crystal
# Abstract Class - ใช้เมื่อมี shared state และ partial implementation
abstract class BaseRepository
  @records = [] of String  # shared state
  
  abstract def find(id : String) : String?
  abstract def save(record : String) : Bool

  # Concrete implementation ที่ subclass ใช้ร่วมกัน
  def count : Int32
    @records.size
  end

  def all : Array(String)
    @records.dup
  end
end

# Module - ใช้เมื่อต้องการแบ่ง behavior โดยไม่มี shared state
module Printable
  abstract def to_print_format : String

  def print
    puts to_print_format
  end
end

# Class สามารถ include Module ได้หลายตัว แต่ inherit ได้แค่ตัวเดียว
class Document
  include Printable

  def initialize(@content : String)
  end

  def to_print_format : String
    "=== Document ===\n#{@content}\n=================="
  end
end

doc = Document.new("Crystal is awesome!")
doc.print
```

### 3.2 เมื่อใช้ Abstract Class

```crystal
# ใช้ Abstract Class เมื่อ:
# 1. มี state ที่ใช้ร่วมกัน (instance variables)
# 2. มี default implementation บางส่วน
# 3. subclasses ล้วนมีความสัมพันธ์ "is-a" กับ parent

abstract class Cache
  @store = {} of String => String
  @hits = 0
  @misses = 0

  abstract def serialize(value : String) : String
  abstract def deserialize(data : String) : String

  def get(key : String) : String?
    if value = @store[key]?
      @hits += 1
      deserialize(value)
    else
      @misses += 1
      nil
    end
  end

  def set(key : String, value : String)
    @store[key] = serialize(value)
  end

  def stats
    puts "Cache hits: #{@hits}, misses: #{@misses}"
  end
end

class PlainCache < Cache
  def serialize(value : String) : String
    value  # ไม่ทำอะไร
  end

  def deserialize(data : String) : String
    data
  end
end

class Base64Cache < Cache
  def serialize(value : String) : String
    # Simplified base64-like encoding
    value.bytes.map { |b| b.to_s(16) }.join
  end

  def deserialize(data : String) : String
    # Simplified base64-like decoding
    data.scan(/../).map { |h| h[0].to_i(16).chr }.join
  end
end

cache = PlainCache.new
cache.set("user:1", "สมชาย")
cache.set("user:2", "สมหญิง")
puts cache.get("user:1")  # => สมชาย
puts cache.get("user:99").inspect  # => nil
cache.stats
```

---

## 4. Template Method Pattern

### 4.1 Template Method กับ Abstract Class

```crystal
# Template Method Pattern - abstract class กำหนด "algorithm skeleton"
abstract class DataProcessor
  # Template method - กำหนดขั้นตอน
  def process(data : String) : String
    validated = validate(data)
    parsed = parse(validated)
    transformed = transform(parsed)
    formatted = format(transformed)
    formatted
  end

  # Steps ที่ subclass ต้อง implement
  abstract def validate(data : String) : String
  abstract def parse(data : String) : Array(String)
  abstract def transform(items : Array(String)) : Array(String)
  abstract def format(items : Array(String)) : String
end

class CsvProcessor < DataProcessor
  def validate(data : String) : String
    raise "ข้อมูลว่าง" if data.empty?
    data.strip
  end

  def parse(data : String) : Array(String)
    data.split(",").map(&.strip)
  end

  def transform(items : Array(String)) : Array(String)
    items.map(&.upcase)
  end

  def format(items : Array(String)) : String
    items.join(" | ")
  end
end

class JsonLikeProcessor < DataProcessor
  def validate(data : String) : String
    raise "ต้องเป็นรูปแบบ key=value" unless data.includes?("=")
    data.strip
  end

  def parse(data : String) : Array(String)
    data.split(";").map(&.strip)
  end

  def transform(items : Array(String)) : Array(String)
    items.map do |item|
      parts = item.split("=")
      "\"#{parts[0]}\": \"#{parts[1]? || ""}\""
    end
  end

  def format(items : Array(String)) : String
    "{ #{items.join(", ")} }"
  end
end

csv = CsvProcessor.new
puts csv.process("apple, banana, cherry")
# => APPLE | BANANA | CHERRY

json = JsonLikeProcessor.new
puts json.process("name=สมชาย; age=25; city=กรุงเทพ")
# => { "name": "สมชาย", "age": "25", "city": "กรุงเทพ" }
```

### 4.2 Report Generator

```crystal
abstract class ReportGenerator
  def generate(title : String, data : Array(Hash(String, String))) : String
    output = [] of String
    output << render_header(title)
    output << render_separator
    data.each { |row| output << render_row(row) }
    output << render_separator
    output << render_footer(data.size)
    output.join("\n")
  end

  abstract def render_header(title : String) : String
  abstract def render_row(row : Hash(String, String)) : String
  abstract def render_footer(count : Int32) : String

  protected def render_separator : String
    "-" * 50
  end
end

class TableReport < ReportGenerator
  def render_header(title : String) : String
    "| #{title.center(48)} |"
  end

  def render_row(row : Hash(String, String)) : String
    cells = row.map { |k, v| "#{k}: #{v}" }.join(" | ")
    "| #{cells.ljust(48)} |"
  end

  def render_footer(count : Int32) : String
    "| Total: #{count} records#{" " * (40 - count.to_s.size)} |"
  end
end

class TextReport < ReportGenerator
  def render_header(title : String) : String
    "=== #{title} ==="
  end

  def render_row(row : Hash(String, String)) : String
    row.map { |k, v| "  #{k}: #{v}" }.join("\n")
  end

  def render_footer(count : Int32) : String
    "จำนวนรายการ: #{count}"
  end
end

data = [
  {"ชื่อ" => "สมชาย", "อายุ" => "25", "แผนก" => "IT"},
  {"ชื่อ" => "สมหญิง", "อายุ" => "30", "แผนก" => "HR"},
]

table_gen = TableReport.new
text_gen = TextReport.new

puts table_gen.generate("รายชื่อพนักงาน", data)
puts "\n"
puts text_gen.generate("รายชื่อพนักงาน", data)
```

---

## 5. Abstract Classes ในระบบจริง

### 5.1 Storage Backend

```crystal
abstract class StorageBackend
  abstract def read(key : String) : String?
  abstract def write(key : String, value : String) : Bool
  abstract def delete(key : String) : Bool
  abstract def exists?(key : String) : Bool

  def get_or_default(key : String, default : String) : String
    read(key) || default
  end

  def get_or_compute(key : String, &block : -> String) : String
    read(key) || begin
      value = block.call
      write(key, value)
      value
    end
  end
end

class MemoryStorage < StorageBackend
  def initialize
    @store = {} of String => String
  end

  def read(key : String) : String?
    @store[key]?
  end

  def write(key : String, value : String) : Bool
    @store[key] = value
    true
  end

  def delete(key : String) : Bool
    @store.delete(key) != nil
  end

  def exists?(key : String) : Bool
    @store.has_key?(key)
  end
end

class PrefixedStorage < StorageBackend
  def initialize(@backend : StorageBackend, @prefix : String)
  end

  def read(key : String) : String?
    @backend.read(prefixed(key))
  end

  def write(key : String, value : String) : Bool
    @backend.write(prefixed(key), value)
  end

  def delete(key : String) : Bool
    @backend.delete(prefixed(key))
  end

  def exists?(key : String) : Bool
    @backend.exists?(prefixed(key))
  end

  private def prefixed(key : String) : String
    "#{@prefix}:#{key}"
  end
end

base = MemoryStorage.new
user_storage = PrefixedStorage.new(base, "user")
session_storage = PrefixedStorage.new(base, "session")

user_storage.write("1", "สมชาย")
session_storage.write("abc123", "user_id=1")

puts user_storage.read("1")           # => สมชาย
puts session_storage.read("abc123")   # => user_id=1
puts user_storage.read("2").inspect   # => nil
puts user_storage.get_or_default("2", "ไม่พบ")  # => ไม่พบ
```

### 5.2 Serializer Framework

```crystal
abstract class Serializer(T)
  abstract def serialize(obj : T) : String
  abstract def deserialize(str : String) : T

  def serialize_many(objects : Array(T)) : String
    objects.map { |obj| serialize(obj) }.join(",")
  end

  def deserialize_many(str : String) : Array(T)
    str.split(",").map { |s| deserialize(s.strip) }
  end
end

class IntSerializer < Serializer(Int32)
  def serialize(obj : Int32) : String
    obj.to_s
  end

  def deserialize(str : String) : Int32
    str.to_i
  end
end

class PersonSerializer < Serializer({name: String, age: Int32})
  def serialize(obj : {name: String, age: Int32}) : String
    "#{obj[:name]}|#{obj[:age]}"
  end

  def deserialize(str : String) : {name: String, age: Int32}
    parts = str.split("|")
    {name: parts[0], age: parts[1].to_i}
  end
end

int_ser = IntSerializer.new
puts int_ser.serialize(42)         # => "42"
puts int_ser.deserialize("42")     # => 42

numbers = [1, 2, 3, 4, 5]
serialized = int_ser.serialize_many(numbers)
puts serialized  # => "1,2,3,4,5"
puts int_ser.deserialize_many(serialized).inspect  # => [1, 2, 3, 4, 5]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Payment Gateway

```crystal
abstract class PaymentGateway
  abstract def charge(amount : Float64, card_number : String) : Bool
  abstract def refund(transaction_id : String, amount : Float64) : Bool
  abstract def gateway_name : String

  def process_payment(amount : Float64, card_number : String) : String
    puts "กำลังดำเนินการชำระเงินผ่าน #{gateway_name}..."
    if amount <= 0
      "ยอดเงินไม่ถูกต้อง"
    elsif charge(amount, card_number)
      transaction_id = generate_transaction_id
      puts "ชำระเงิน ฿#{amount} สำเร็จ! Transaction: #{transaction_id}"
      transaction_id
    else
      "การชำระเงินล้มเหลว"
    end
  end

  private def generate_transaction_id : String
    "TXN#{Time.local.to_unix}#{rand(1000)}"
  end
end

class MockGateway < PaymentGateway
  def gateway_name : String
    "Mock Gateway (Testing)"
  end

  def charge(amount : Float64, card_number : String) : Bool
    # จำลอง: ถ้าหมายเลขขึ้นต้นด้วย 4 = สำเร็จ
    card_number.starts_with?("4")
  end

  def refund(transaction_id : String, amount : Float64) : Bool
    puts "คืนเงิน #{amount} สำหรับ #{transaction_id}"
    true
  end
end

gateway = MockGateway.new
gateway.process_payment(1000.0, "4111111111111111")  # สำเร็จ
gateway.process_payment(500.0, "5500000000000004")   # ล้มเหลว
```

### แบบฝึกหัดที่ 2: Notification System

```crystal
abstract class NotificationSender
  abstract def send(recipient : String, message : String) : Bool
  abstract def channel_name : String

  def send_bulk(recipients : Array(String), message : String) : Int32
    success_count = 0
    recipients.each do |recipient|
      success_count += 1 if send(recipient, message)
    end
    puts "ส่ง #{channel_name}: #{success_count}/#{recipients.size} สำเร็จ"
    success_count
  end
end

class EmailNotification < NotificationSender
  def channel_name : String
    "Email"
  end

  def send(recipient : String, message : String) : Bool
    puts "📧 ส่งอีเมลถึง #{recipient}: #{message[0..50]}"
    true
  end
end

class SmsNotification < NotificationSender
  def channel_name : String
    "SMS"
  end

  def send(recipient : String, message : String) : Bool
    if recipient.match?(/^0[0-9]{9}$/)
      puts "📱 ส่ง SMS ถึง #{recipient}: #{message[0..160]}"
      true
    else
      puts "❌ หมายเลขโทรศัพท์ไม่ถูกต้อง: #{recipient}"
      false
    end
  end
end

class PushNotification < NotificationSender
  def channel_name : String
    "Push"
  end

  def send(recipient : String, message : String) : Bool
    puts "🔔 Push notification ถึง device: #{recipient}"
    true
  end
end

email = EmailNotification.new
sms = SmsNotification.new
push = PushNotification.new

recipients = ["user1@example.com", "user2@example.com", "user3@example.com"]
email.send_bulk(recipients, "สวัสดี ยินดีต้อนรับสู่ระบบ!")

phone_numbers = ["0812345678", "invalid", "0987654321"]
sms.send_bulk(phone_numbers, "รหัส OTP ของคุณคือ 123456")
```

---

## สรุป

Abstract Classes ใน Crystal:

| Concept | Syntax | คำอธิบาย |
|---------|--------|---------|
| Abstract class | `abstract class Name` | ไม่สามารถ instantiate ได้ |
| Abstract method | `abstract def name : Type` | subclass ต้อง implement |
| Concrete method | `def name` (ปกติ) | มี implementation ใน abstract class |
| Template method | ใช้ abstract method ใน concrete method | กำหนด algorithm skeleton |

**เมื่อใช้ Abstract Class:**
- มีความสัมพันธ์แบบ "is-a" ระหว่าง subclass และ parent
- ต้องการ shared state (instance variables)
- ต้องการ partial implementation ที่ subclass ใช้ร่วมกัน
- ต้องการบังคับ subclass ให้ implement methods บางตัว

**เมื่อใช้ Module แทน:**
- ต้องการแบ่ง behavior โดยไม่มี shared state
- Class เดียวต้องการ mixin หลายตัว
- ไม่มีความสัมพันธ์แบบ "is-a"
