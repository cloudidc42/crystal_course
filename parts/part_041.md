# Part 41: Interfaces และ Duck Typing ใน Crystal

## บทนำ

Crystal เป็นภาษาที่มีระบบ type แบบ static แต่ยังคงความยืดหยุ่นผ่านแนวคิดที่เรียกว่า **Duck Typing** ซึ่งมาจากสุภาษิตที่ว่า "ถ้ามันเดินเหมือนเป็ด และร้องเหมือนเป็ด มันก็คือเป็ด" ในบทนี้เราจะเรียนรู้วิธีที่ Crystal จัดการกับ interfaces ผ่านกลไกต่างๆ

---

## 41.1 Duck Typing คืออะไร?

Duck Typing คือแนวคิดที่บอกว่า เราไม่ได้สนใจว่า object เป็น type อะไร แต่สนใจว่ามันมี method ที่เราต้องการใช้หรือไม่

### ตัวอย่างพื้นฐาน

```crystal
# ใน Ruby/dynamic languages เราทำแบบนี้ได้
# Crystal ต้องการ type information แต่ก็ยืดหยุ่นมาก

class Duck
  def quack
    "Quack!"
  end

  def walk
    "เดินแบบเป็ด"
  end
end

class Person
  def quack
    "ฉันทำเสียงแบบเป็ด: Quack!"
  end

  def walk
    "เดินสองขา"
  end
end

class Robot
  def quack
    "BEEP BOOP QUACK"
  end

  def walk
    "เดินแบบหุ่นยนต์"
  end
end

# ฟังก์ชันที่รับ duck typing
def make_it_quack(duck)
  puts duck.quack
end

duck = Duck.new
person = Person.new
robot = Robot.new

make_it_quack(duck)    # => "Quack!"
make_it_quack(person)  # => "ฉันทำเสียงแบบเป็ด: Quack!"
make_it_quack(robot)   # => "BEEP BOOP QUACK"
```

Crystal จะ infer type และ compile สำเร็จเพราะทุก object มี method `quack`

---

## 41.2 Type Restrictions

Crystal ให้เราระบุ type ที่ยอมรับได้อย่างชัดเจนผ่าน type restrictions

### การระบุ type เดียว

```crystal
def greet(name : String)
  "สวัสดี #{name}!"
end

puts greet("Alice")  # => "สวัสดี Alice!"
# greet(42)  # => Error: argument 'name' must be String
```

### การรับหลาย types ด้วย Union Types

```crystal
def process(value : String | Int32 | Float64)
  case value
  when String
    "String: #{value}"
  when Int32
    "Integer: #{value}"
  when Float64
    "Float: #{value}"
  end
end

puts process("hello")  # => "String: hello"
puts process(42)       # => "Integer: 42"
puts process(3.14)     # => "Float: 3.14"
```

### Generic type restrictions

```crystal
# รับ type ใดก็ได้ที่ implement method ที่ต้องการ
def double(value)
  value * 2
end

puts double(5)      # => 10
puts double(3.14)   # => 6.28
puts double("ha")   # => "haha"
```

---

## 41.3 responds_to? Method

`responds_to?` ช่วยให้เราตรวจสอบว่า object มี method ที่ต้องการหรือไม่ โดยไม่ต้องรู้ type

```crystal
class Cat
  def speak
    "Meow!"
  end

  def purr
    "Purrr..."
  end
end

class Dog
  def speak
    "Woof!"
  end

  def fetch
    "กำลัง fetch!"
  end
end

def make_sound(animal)
  if animal.responds_to?(:speak)
    animal.speak
  else
    "ไม่รู้จะพูดอะไร"
  end
end

cat = Cat.new
dog = Dog.new

puts make_sound(cat)  # => "Meow!"
puts make_sound(dog)  # => "Woof!"
```

### ใช้ responds_to? กับ conditional behavior

```crystal
class SmartDevice
  def power_on
    "เปิดเครื่องแล้ว"
  end

  def wifi_connect
    "เชื่อมต่อ WiFi แล้ว"
  end
end

class BasicDevice
  def power_on
    "เปิดเครื่องแล้ว"
  end
end

def setup_device(device)
  result = [] of String
  result << device.power_on

  if device.responds_to?(:wifi_connect)
    result << device.wifi_connect
  end

  result.join(", ")
end

smart = SmartDevice.new
basic = BasicDevice.new

puts setup_device(smart)  # => "เปิดเครื่องแล้ว, เชื่อมต่อ WiFi แล้ว"
puts setup_device(basic)  # => "เปิดเครื่องแล้ว"
```

---

## 41.4 Structural Subtyping ผ่าน Modules

Crystal ใช้ modules เป็น "informal interfaces" ที่ทำให้เกิด structural subtyping

### Module เป็น Interface

```crystal
module Drawable
  abstract def draw
  abstract def area : Float64
end

module Resizable
  abstract def resize(factor : Float64)
end

class Circle
  include Drawable
  include Resizable

  def initialize(@radius : Float64)
  end

  def draw
    "วาดวงกลม รัศมี #{@radius}"
  end

  def area : Float64
    Math::PI * @radius ** 2
  end

  def resize(factor : Float64)
    @radius *= factor
    self
  end
end

class Rectangle
  include Drawable
  include Resizable

  def initialize(@width : Float64, @height : Float64)
  end

  def draw
    "วาดสี่เหลี่ยม #{@width} x #{@height}"
  end

  def area : Float64
    @width * @height
  end

  def resize(factor : Float64)
    @width *= factor
    @height *= factor
    self
  end
end

shapes = [Circle.new(5.0), Rectangle.new(4.0, 6.0)] of Drawable

shapes.each do |shape|
  puts shape.draw
  puts "พื้นที่: #{shape.area.round(2)}"
  puts "---"
end
```

### Abstract Methods ใน Modules

```crystal
module Logger
  abstract def log(message : String)
  
  def info(message : String)
    log("[INFO] #{message}")
  end
  
  def error(message : String)
    log("[ERROR] #{message}")
  end
  
  def warn(message : String)
    log("[WARN] #{message}")
  end
end

class ConsoleLogger
  include Logger
  
  def log(message : String)
    puts "#{Time.local}: #{message}"
  end
end

class FileLogger
  include Logger
  
  def initialize(@filename : String)
    @file = [] of String
  end
  
  def log(message : String)
    @file << "#{Time.local}: #{message}"
  end
  
  def dump
    @file.each { |line| puts line }
  end
end

console_log = ConsoleLogger.new
file_log = FileLogger.new("app.log")

console_log.info("Application started")
console_log.warn("Low memory")
console_log.error("Connection failed")

file_log.info("Database connected")
file_log.error("Query timeout")
file_log.dump
```

---

## 41.5 Informal Interfaces ผ่าน Modules

ต่างจาก Java/C# ที่มี `interface` keyword Crystal ใช้ modules สร้าง informal interfaces

### Pattern: Protocol Modules

```crystal
# กำหนด "protocol" ผ่าน module
module Serializable
  abstract def to_json : String
  abstract def to_csv : String
end

module Persistable
  abstract def save : Bool
  abstract def delete : Bool
  
  def exists? : Bool
    false  # default implementation
  end
end

class User
  include Serializable
  include Persistable
  
  def initialize(@name : String, @email : String, @id : Int32 = 0)
  end
  
  def to_json : String
    %({ "name": "#{@name}", "email": "#{@email}", "id": #{@id} })
  end
  
  def to_csv : String
    "#{@id},#{@name},#{@email}"
  end
  
  def save : Bool
    # จำลองการ save
    @id = rand(1000)
    puts "Saved user #{@name} with id #{@id}"
    true
  end
  
  def delete : Bool
    puts "Deleted user #{@name}"
    true
  end
  
  def exists? : Bool
    @id > 0
  end
end

user = User.new("Alice", "alice@example.com")
puts user.to_json
puts user.to_csv
puts "Exists: #{user.exists?}"
user.save
puts "Exists after save: #{user.exists?}"
puts user.to_json
```

### Type restriction ด้วย Module

```crystal
module Printable
  abstract def to_print_string : String
end

class Report
  include Printable
  
  def initialize(@title : String, @content : String)
  end
  
  def to_print_string : String
    "=== #{@title} ===\n#{@content}"
  end
end

class Invoice
  include Printable
  
  def initialize(@number : Int32, @amount : Float64)
  end
  
  def to_print_string : String
    "Invoice ##{@number}: $#{@amount}"
  end
end

# รับเฉพาะ Printable objects
def print_document(doc : Printable)
  puts doc.to_print_string
  puts "(Printed successfully)"
end

report = Report.new("Annual Report", "Revenue increased by 20%")
invoice = Invoice.new(1001, 599.99)

print_document(report)
print_document(invoice)
```

---

## 41.6 Duck Typing กับ Generic Methods

```crystal
# Generic method ที่ทำงานกับ type ใดก็ได้ที่มี method ที่ต้องการ
def sum_all(items)
  items.reduce(0) { |acc, item| acc + item }
end

puts sum_all([1, 2, 3, 4, 5])        # => 15
puts sum_all([1.5, 2.5, 3.0])        # => 7.0
puts sum_all(["a", "b", "c"])        # => "abc"
```

### Generic Constraints

```crystal
# ใช้ forall สำหรับ generic type constraints
def compare_and_return_larger(a : T, b : T) : T forall T
  a > b ? a : b
end

puts compare_and_return_larger(3, 7)        # => 7
puts compare_and_return_larger("apple", "banana")  # => "banana"
puts compare_and_return_larger(3.14, 2.71)  # => 3.14
```

---

## 41.7 Mixin Patterns

Modules ใน Crystal ทำงานเป็น mixins ซึ่งให้ reusable behavior

### Timestamp Mixin

```crystal
module Timestamps
  getter created_at : Time
  getter updated_at : Time
  
  def initialize
    @created_at = Time.local
    @updated_at = Time.local
  end
  
  def touch
    @updated_at = Time.local
  end
  
  def age : Time::Span
    Time.local - @created_at
  end
end

module Auditable
  getter created_by : String
  getter updated_by : String?
  
  def created_by=(creator : String)
    @created_by = creator
  end
  
  def updated_by=(updater : String)
    @updated_by = updater
  end
end

class Article
  include Timestamps
  include Auditable
  
  getter title : String
  getter content : String
  
  def initialize(@title : String, @content : String, @created_by : String)
    super()  # เรียก Timestamps initialize
    @updated_by = nil
  end
  
  def update(new_content : String, updater : String)
    @content = new_content
    @updated_by = updater
    touch
  end
end

article = Article.new("Crystal Tutorial", "เนื้อหา...", "Alice")
puts "Created: #{article.created_at}"
puts "By: #{article.created_by}"

sleep 0.1.seconds

article.update("เนื้อหาใหม่...", "Bob")
puts "Updated: #{article.updated_at}"
puts "By: #{article.updated_by}"
```

---

## 41.8 Interface Segregation

หลักการ Interface Segregation บอกว่าควรแยก interfaces ที่ใหญ่เป็น interfaces ย่อยๆ

```crystal
# แทนที่จะมี interface ขนาดใหญ่
module Readable
  abstract def read : String
end

module Writable
  abstract def write(data : String) : Bool
end

module Seekable
  abstract def seek(position : Int32)
  abstract def position : Int32
end

# แต่ละ class implement เฉพาะ interfaces ที่ต้องการ
class ReadOnlyFile
  include Readable
  
  def initialize(@content : String)
  end
  
  def read : String
    @content
  end
end

class WriteOnlyLog
  include Writable
  
  def initialize
    @entries = [] of String
  end
  
  def write(data : String) : Bool
    @entries << data
    true
  end
  
  def entries
    @entries
  end
end

class StreamFile
  include Readable
  include Writable
  include Seekable
  
  def initialize
    @content = ""
    @position = 0
  end
  
  def read : String
    @content[@position..]
  end
  
  def write(data : String) : Bool
    @content += data
    true
  end
  
  def seek(position : Int32)
    @position = position.clamp(0, @content.size)
  end
  
  def position : Int32
    @position
  end
end

# ฟังก์ชัน utility
def read_from(source : Readable)
  source.read
end

def write_to(dest : Writable, data : String)
  dest.write(data)
end

stream = StreamFile.new
write_to(stream, "Hello, Crystal!")
stream.seek(7)
puts read_from(stream)  # => "Crystal!"
```

---

## 41.9 Protocols และ Self Types

```crystal
module Cloneable
  abstract def clone : self
end

class Point
  include Cloneable
  
  def initialize(@x : Float64, @y : Float64)
  end
  
  def clone : self
    self.class.new(@x, @y)
  end
  
  def to_s
    "(#{@x}, #{@y})"
  end
end

class Color
  include Cloneable
  
  def initialize(@r : UInt8, @g : UInt8, @b : UInt8)
  end
  
  def clone : self
    self.class.new(@r, @g, @b)
  end
  
  def to_s
    "rgb(#{@r}, #{@g}, #{@b})"
  end
end

p1 = Point.new(3.0, 4.0)
p2 = p1.clone
puts p1  # => "(3.0, 4.0)"
puts p2  # => "(3.0, 4.0)"
puts p1.object_id == p2.object_id  # => false (different objects)

c1 = Color.new(255_u8, 128_u8, 0_u8)
c2 = c1.clone
puts c1  # => "rgb(255, 128, 0)"
puts c2  # => "rgb(255, 128, 0)"
```

---

## 41.10 Type Checking ด้วย is_a?

```crystal
class Animal
  def breathe
    "หายใจอยู่"
  end
end

class Dog < Animal
  def bark
    "Woof!"
  end
end

class Cat < Animal
  def meow
    "Meow!"
  end
end

def interact_with(animal : Animal)
  puts animal.breathe
  
  if animal.is_a?(Dog)
    puts animal.bark
  elsif animal.is_a?(Cat)
    puts animal.meow
  end
end

dog = Dog.new
cat = Cat.new

interact_with(dog)
interact_with(cat)
```

### Type narrowing ด้วย case/when

```crystal
def describe_type(obj : String | Int32 | Float64 | Bool)
  case obj
  when String
    "เป็น String: '#{obj}' ความยาว #{obj.size}"
  when Int32
    "เป็น Integer: #{obj}"
  when Float64
    "เป็น Float: #{obj}"
  when Bool
    "เป็น Boolean: #{obj}"
  else
    "ไม่รู้จัก type"
  end
end

puts describe_type("hello")
puts describe_type(42)
puts describe_type(3.14)
puts describe_type(true)
```

---

## 41.11 Extending Modules

Crystal ให้เพิ่ม module methods เข้าไปใน class ที่มีอยู่แล้ว

```crystal
module StringExtensions
  def palindrome?
    self == self.reverse
  end
  
  def word_count : Int32
    split.size
  end
  
  def truncate(length : Int32, omission : String = "...") : String
    if self.size <= length
      self
    else
      self[0, length - omission.size] + omission
    end
  end
end

class String
  include StringExtensions
end

puts "racecar".palindrome?   # => true
puts "hello".palindrome?     # => false
puts "Hello World Crystal".word_count  # => 3
puts "This is a very long string".truncate(15)  # => "This is a ve..."
```

---

## 41.12 Comparison กับ Java Interfaces

| Feature | Java Interface | Crystal Module |
|---------|---------------|----------------|
| Multiple implementation | ✓ | ✓ |
| Default methods | ✓ (Java 8+) | ✓ |
| Abstract methods | ✓ | ✓ |
| State (instance vars) | ✗ | ✓ |
| Multiple inheritance | ✓ | ✓ |
| Type checking | Compile-time | Compile-time |

```crystal
# Crystal module ทำได้มากกว่า Java interface
module ValueObject
  # มี state
  @validated : Bool = false
  
  # มี concrete methods
  def valid?
    @validated
  end
  
  def validate!
    @validated = true
    self
  end
  
  # มี abstract methods
  abstract def validate : Bool
end

class Email
  include ValueObject
  
  def initialize(@address : String)
  end
  
  def validate : Bool
    @address.includes?("@") && @address.includes?(".")
  end
  
  def to_s
    @address
  end
end

email = Email.new("user@example.com")
puts email.valid?     # => false
puts email.validate   # => true
email.validate!
puts email.valid?     # => true
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Payment Interface

สร้าง module `Payable` ที่มี abstract methods:
- `charge(amount : Float64) : Bool`
- `refund(amount : Float64) : Bool`
- `balance : Float64`

จากนั้นสร้าง 3 classes ที่ implement:
1. `CreditCard` - มี credit limit
2. `BankAccount` - มี balance ธรรมดา
3. `Wallet` - มี prepaid balance

```crystal
module Payable
  abstract def charge(amount : Float64) : Bool
  abstract def refund(amount : Float64) : Bool
  abstract def balance : Float64
  
  def can_charge?(amount : Float64) : Bool
    balance >= amount
  end
end

class CreditCard
  include Payable
  
  def initialize(@credit_limit : Float64)
    @used = 0.0
  end
  
  def balance : Float64
    @credit_limit - @used
  end
  
  def charge(amount : Float64) : Bool
    if can_charge?(amount)
      @used += amount
      puts "Charged #{amount} to credit card. Remaining: #{balance}"
      true
    else
      puts "Insufficient credit limit"
      false
    end
  end
  
  def refund(amount : Float64) : Bool
    @used = [@used - amount, 0.0].max
    puts "Refunded #{amount}. Used: #{@used}"
    true
  end
end

class BankAccount
  include Payable
  
  def initialize(@balance : Float64)
  end
  
  def balance : Float64
    @balance
  end
  
  def charge(amount : Float64) : Bool
    if can_charge?(amount)
      @balance -= amount
      puts "Charged #{amount} from bank account. Balance: #{@balance}"
      true
    else
      puts "Insufficient funds"
      false
    end
  end
  
  def refund(amount : Float64) : Bool
    @balance += amount
    puts "Refunded #{amount}. Balance: #{@balance}"
    true
  end
end

# ทดสอบ
card = CreditCard.new(5000.0)
bank = BankAccount.new(1000.0)

card.charge(200.0)
card.charge(300.0)
bank.charge(500.0)
bank.charge(600.0)  # ไม่เพียงพอ
card.refund(100.0)
```

### แบบฝึกหัดที่ 2: Duck Typing กับ Collection

```crystal
# สร้าง method ที่ทำงานกับ collection ใดๆ ที่มี each method
def print_all(collection)
  collection.each do |item|
    puts "- #{item}"
  end
end

print_all([1, 2, 3])
print_all({"a", "b", "c"})
print_all(1..5)
```

### แบบฝึกหัดที่ 3: responds_to? Pattern

```crystal
class Service
  def start
    "Service started"
  end

  def stop
    "Service stopped"
  end

  def health_check
    "Healthy"
  end
end

class BasicService
  def start
    "Basic service started"
  end

  def stop
    "Basic service stopped"
  end
end

def manage_service(service)
  puts service.start
  
  if service.responds_to?(:health_check)
    puts "Health: #{service.health_check}"
  end
  
  puts service.stop
end

manage_service(Service.new)
puts "---"
manage_service(BasicService.new)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Duck Typing** - Crystal ไม่สนใจ type แต่สนใจว่า object มี method ที่ต้องการหรือไม่
2. **Type Restrictions** - การระบุ type ที่ยอมรับในพารามิเตอร์
3. **responds_to?** - การตรวจสอบว่า object มี method หรือไม่ก่อนเรียก
4. **Structural Subtyping** - ผ่าน modules ที่ทำหน้าที่เป็น interfaces
5. **Abstract Methods** - บังคับให้ subclass implement methods
6. **Mixin Patterns** - การนำ modules มาใช้เพิ่ม behavior
7. **Interface Segregation** - แยก interfaces ใหญ่เป็นย่อยๆ

Crystal รวมความปลอดภัยของ static typing เข้ากับความยืดหยุ่นของ duck typing ได้อย่างลงตัว ทำให้โค้ดทั้งปลอดภัยและยืดหยุ่น

---

*ต่อไป: Part 42 - Operator Overloading*
