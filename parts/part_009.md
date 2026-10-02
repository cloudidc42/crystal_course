# ตอนที่ 9: Nil และ Union Types

## บทนำ

หนึ่งในฟีเจอร์ที่ทรงพลังและปลอดภัยที่สุดของ Crystal คือระบบ **Nil Safety** ซึ่งช่วยป้องกัน NullPointerException ที่เป็นปัญหาพบบ่อยในภาษาโปรแกรมอื่น Crystal บังคับให้โปรแกรมเมอร์จัดการกับค่า `nil` อย่างชัดเจนผ่านระบบ Type ที่แข็งแกร่ง

---

## 9.1 Nil คืออะไร?

`nil` ใน Crystal เป็นค่าที่แสดงถึง "ไม่มีค่า" หรือ "ว่างเปล่า" มันเป็น instance ของ class `Nil`

```crystal
# nil เป็น object ของ class Nil
puts nil.class  # => Nil
puts nil.nil?   # => true
puts nil.inspect # => "nil"

# nil มีค่า falsy
if nil
  puts "นี่จะไม่ถูกพิมพ์"
else
  puts "nil เป็น falsy"  # => nil เป็น falsy
end
```

### ความแตกต่างระหว่าง nil และ false

```crystal
# ทั้ง nil และ false เป็น falsy
# แต่เป็น object คนละประเภท

puts nil.class    # => Nil
puts false.class  # => Bool

# เปรียบเทียบ
puts nil == false    # => false
puts nil.nil?        # => true
puts false.nil?      # => false
puts nil.is_a?(Nil)  # => true
puts false.is_a?(Bool) # => true
```

---

## 9.2 Nil Safety ใน Crystal

ใน Crystal ตัวแปรปกติไม่สามารถเป็น nil ได้ ต้องระบุอย่างชัดเจนว่าตัวแปรอาจเป็น nil ได้

```crystal
# ตัวแปรปกติ - ไม่สามารถเป็น nil ได้
name : String = "Alice"
# name = nil  # Error: type must be String, not (String | Nil)

# ตัวแปรที่อาจเป็น nil - ใช้ ? หลัง type
nullable_name : String? = "Bob"
nullable_name = nil  # OK! เพราะ String? หมายถึง String | Nil

puts nullable_name.class  # => Nil
```

### การประกาศตัวแปรที่อาจเป็น nil

```crystal
# วิธีที่ 1: ใช้ ? หลัง type name
age : Int32? = nil
age = 25
puts age  # => 25

# วิธีที่ 2: ใช้ Union type โดยตรง
score : (Int32 | Nil) = nil
score = 100
puts score  # => 100

# ทั้งสองวิธีเทียบเท่ากัน
# String? เท่ากับ (String | Nil)
x : String? = "hello"
y : (String | Nil) = "world"
puts x.class  # => String
puts y.class  # => String
```

---

## 9.3 Union Types

Union Type คือ type ที่สามารถเป็นได้หลาย type

```crystal
# Union Type พื้นฐาน
value : Int32 | String = 42
puts value  # => 42
puts value.class  # => Int32

value = "hello"
puts value  # => hello
puts value.class  # => String

# Union Type ซับซ้อน
complex : Int32 | String | Float64 | Nil = nil
complex = 3.14
puts complex  # => 3.14
puts complex.class  # => Float64
```

### Union Types ในพารามิเตอร์ method

```crystal
def process(value : Int32 | String)
  case value
  when Int32
    puts "ตัวเลข: #{value * 2}"
  when String
    puts "ข้อความ: #{value.upcase}"
  end
end

process(42)       # => ตัวเลข: 84
process("hello")  # => ข้อความ: HELLO
```

---

## 9.4 Nil Coalescing Operator (||)

Operator `||` ใช้ให้ค่าเริ่มต้นเมื่อค่าเป็น nil หรือ false

```crystal
name : String? = nil
display_name = name || "ไม่ระบุชื่อ"
puts display_name  # => ไม่ระบุชื่อ

name = "Alice"
display_name = name || "ไม่ระบุชื่อ"
puts display_name  # => Alice
```

### ความแตกต่างระหว่าง || และ ||= 

```crystal
# || ใช้ในนิพจน์
score : Int32? = nil
result = score || 0
puts result  # => 0

# ||= กำหนดค่าเมื่อ nil หรือ false
config : String? = nil
config ||= "default"
puts config  # => default

config ||= "other"  # ไม่เปลี่ยน เพราะ config มีค่าแล้ว
puts config  # => default
```

### ระวัง: || กับค่า false

```crystal
# || ให้ค่าเริ่มต้นเมื่อค่าเป็น falsy (ทั้ง nil และ false)
flag : Bool? = false
result = flag || true
puts result  # => true  (อาจไม่ใช่สิ่งที่ต้องการ!)

# ถ้าต้องการจัดการเฉพาะ nil ใช้ if nil?
flag = false
result = flag.nil? ? true : flag
puts result  # => false  (ถูกต้องแล้ว)
```

---

## 9.5 Safe Navigation Operator (&.)

Operator `&.` เรียก method เฉพาะเมื่อ object ไม่ใช่ nil

```crystal
name : String? = "Alice"

# แบบปลอดภัย: ใช้ &.
puts name&.upcase  # => ALICE
puts name&.length  # => 5

name = nil
puts name&.upcase  # => (ไม่มี output เพราะเป็น nil)
puts name&.length.inspect  # => nil
```

### เปรียบเทียบวิธีการต่างๆ

```crystal
user_name : String? = "bob"

# วิธีที่ 1: ตรวจสอบด้วย if
if user_name
  puts user_name.upcase
end

# วิธีที่ 2: ใช้ &.
puts user_name&.upcase

# วิธีที่ 3: ใช้ try (มีใน standard library บางส่วน)
# puts user_name.try(&.upcase)  # ทำงานเหมือนกัน

# &. กับ method chaining
text : String? = "  hello world  "
result = text&.strip&.upcase&.split(" ")
puts result.inspect  # => ["HELLO", "WORLD"]

text = nil
result = text&.strip&.upcase&.split(" ")
puts result.inspect  # => nil
```

### &. กับ block

```crystal
numbers : Array(Int32)? = [1, 2, 3, 4, 5]

# เรียก method ที่รับ block
sum = numbers&.sum
puts sum.inspect  # => 15

numbers = nil
sum = numbers&.sum
puts sum.inspect  # => nil
```

---

## 9.6 not_nil! Method

`not_nil!` บอก Crystal ว่าค่านี้ไม่ใช่ nil อย่างแน่นอน ถ้าเป็น nil จะ raise RuntimeError

```crystal
value : String? = "hello"

# บอกว่าไม่ใช่ nil แน่ๆ
str = value.not_nil!
puts str.upcase  # => HELLO
puts str.class   # => String (ไม่ใช่ String?)

# ถ้าเป็น nil จะ raise exception
value = nil
begin
  str = value.not_nil!
rescue NilAssertionError => e
  puts "Error: #{e.message}"  # => Error: not_nil! called on nil
end
```

### เมื่อใดควรใช้ not_nil!

```crystal
# ใช้เมื่อมั่นใจ 100% ว่าไม่ใช่ nil แต่ compiler ยังไม่รู้
class Config
  @@instance : Config? = nil

  def self.instance : Config
    @@instance ||= new
    @@instance.not_nil!  # เราแน่ใจว่ามีค่าแล้วหลัง ||=
  end
end

# หรือใช้หลังตรวจสอบแล้ว
def find_user(id : Int32) : String?
  users = {"1" => "Alice", "2" => "Bob"}
  users[id.to_s]?
end

user = find_user(1)
if user
  # ใน block นี้ Crystal รู้ว่า user เป็น String แล้ว
  puts user.upcase  # ไม่ต้องใช้ not_nil!
end
```

---

## 9.7 compact และ compact_map

`compact` ใช้กรอง nil ออกจาก Array

```crystal
# compact กรอง nil ออก
values : Array(String?) = ["hello", nil, "world", nil, "!"]
filtered = values.compact
puts filtered.inspect  # => ["hello", "world", "!"]
puts filtered.class    # => Array(String)  (ไม่มี ? แล้ว!)

# ตัวอย่างจริง
names = ["Alice", nil, "Bob", nil, "Charlie"]
puts names.compact.join(", ")  # => Alice, Bob, Charlie
```

### compact_map

```crystal
# compact_map รวม map และ compact เข้าด้วยกัน
numbers = [1, 2, 3, 4, 5, 6]

# หาเลขคี่แล้วคูณ 2 (คืน nil ถ้าเป็นเลขคู่)
result = numbers.compact_map do |n|
  n.odd? ? n * 2 : nil
end
puts result.inspect  # => [2, 6, 10]

# เทียบกับวิธีอื่น
result2 = numbers.select(&.odd?).map { |n| n * 2 }
puts result2.inspect  # => [2, 6, 10]
```

### ตัวอย่างจริงกับ compact_map

```crystal
# แปลง string เป็น int ที่ valid
inputs = ["1", "abc", "3", "xyz", "5"]

results = inputs.compact_map do |s|
  s.to_i?  # คืน Int32? (nil ถ้าแปลงไม่ได้)
end

puts results.inspect  # => [1, 3, 5]
```

---

## 9.8 nil? Checks

การตรวจสอบว่าค่าเป็น nil หรือไม่

```crystal
value : String? = nil

# วิธีที่ 1: nil? method
puts value.nil?   # => true

value = "hello"
puts value.nil?   # => false

# วิธีที่ 2: เปรียบเทียบโดยตรง
value = nil
puts value == nil  # => true
puts value.nil?    # => true

# วิธีที่ 3: truthy/falsy check
if value
  puts "มีค่า: #{value}"
else
  puts "ไม่มีค่า"  # จะได้ผลนี้เมื่อ value เป็น nil
end
```

### ระวังความแตกต่างระหว่าง nil? และ truthy check

```crystal
# nil? เป็น false ถ้าเป็น false
flag : Bool? = false
puts flag.nil?   # => false
puts flag ? "truthy" : "falsy"  # => falsy

# ใช้ nil? เมื่อต้องการตรวจสอบเฉพาะ nil
if !flag.nil?
  puts "ไม่ใช่ nil: #{flag}"  # => ไม่ใช่ nil: false
end
```

---

## 9.9 Union Type Narrowing

Crystal สามารถ narrow type ใน Union ได้โดยอัตโนมัติ

### การ narrow ด้วย if

```crystal
value : Int32 | String | Nil = "hello"

if value.is_a?(String)
  # ใน block นี้ compiler รู้ว่า value เป็น String
  puts value.upcase  # => HELLO
  puts value.class   # => String
elsif value.is_a?(Int32)
  # ที่นี่รู้ว่าเป็น Int32
  puts value * 2
else
  # ที่นี่รู้ว่าเป็น Nil
  puts "ค่าว่างเปล่า"
end
```

### การ narrow ด้วย case/when

```crystal
def describe(value : Int32 | String | Float64 | Nil) : String
  case value
  when Int32
    "จำนวนเต็ม: #{value}"
  when String
    "ข้อความ: #{value.upcase}"
  when Float64
    "ทศนิยม: #{value.round(2)}"
  when Nil
    "ไม่มีค่า"
  else
    "ไม่รู้จัก"
  end
end

puts describe(42)          # => จำนวนเต็ม: 42
puts describe("hello")     # => ข้อความ: HELLO
puts describe(3.14159)     # => ทศนิยม: 3.14
puts describe(nil)         # => ไม่มีค่า
```

### การ narrow ด้วย is_a?

```crystal
values : Array(Int32 | String | Nil) = [1, "two", nil, 3, "four", nil]

values.each do |v|
  if v.is_a?(Int32)
    puts "Int: #{v + 10}"
  elsif v.is_a?(String)
    puts "Str: #{v.reverse}"
  else
    puts "Nil!"
  end
end
```

---

## 9.10 Practical Nil Safety Patterns

### Pattern 1: Early Return

```crystal
def process_name(name : String?) : String
  return "ไม่ระบุ" if name.nil?
  return "ว่างเปล่า" if name.empty?
  
  # ที่นี่ name เป็น String แน่ๆ
  name.strip.capitalize
end

puts process_name(nil)         # => ไม่ระบุ
puts process_name("")          # => ว่างเปล่า
puts process_name("  alice  ") # => Alice
```

### Pattern 2: Default Values

```crystal
struct Config
  property host : String
  property port : Int32
  property timeout : Int32

  def initialize(
    @host : String = "localhost",
    @port : Int32 = 8080,
    @timeout : Int32 = 30
  )
  end
end

# สร้าง config จาก Hash ที่อาจมีค่า nil
def create_config(options : Hash(String, String?)) : Config
  Config.new(
    host: options["host"]? || "localhost",
    port: (options["port"]? || "8080").to_i,
    timeout: (options["timeout"]? || "30").to_i
  )
end

opts = {"host" => "example.com", "port" => nil, "timeout" => "60"}
config = create_config(opts)
puts config.host     # => example.com
puts config.port     # => 8080
puts config.timeout  # => 60
```

### Pattern 3: Safe Chaining

```crystal
class User
  property name : String
  property email : String?
  property address : Address?

  def initialize(@name : String, @email : String? = nil, @address : Address? = nil)
  end
end

class Address
  property city : String
  property country : String?

  def initialize(@city : String, @country : String? = nil)
  end
end

user = User.new("Alice", "alice@example.com", Address.new("Bangkok", "Thailand"))

# Safe navigation chaining
puts user.address&.city          # => Bangkok
puts user.address&.country       # => Thailand
puts user.address&.country&.upcase  # => THAILAND

user2 = User.new("Bob")
puts user2.address&.city.inspect    # => nil
puts user2.address&.country.inspect # => nil
```

### Pattern 4: Result Type Pattern

```crystal
# Pattern สำหรับ error handling ที่ดีกว่า nil
alias Result(T) = T | Exception

def safe_divide(a : Int32, b : Int32) : Result(Float64)
  return Exception.new("หารด้วยศูนย์ไม่ได้!") if b == 0
  a.to_f / b.to_f
end

result = safe_divide(10, 2)
case result
when Float64
  puts "ผลลัพธ์: #{result}"  # => ผลลัพธ์: 5.0
when Exception
  puts "เกิดข้อผิดพลาด: #{result.message}"
end

result2 = safe_divide(10, 0)
case result2
when Float64
  puts "ผลลัพธ์: #{result2}"
when Exception
  puts "เกิดข้อผิดพลาด: #{result2.message}"  # => เกิดข้อผิดพลาด: หารด้วยศูนย์ไม่ได้!
end
```

### Pattern 5: Nil Object Pattern

```crystal
# แทนที่จะใช้ nil ใช้ Null Object
abstract class Shape
  abstract def area : Float64
  abstract def name : String
end

class Rectangle < Shape
  def initialize(@width : Float64, @height : Float64)
  end

  def area : Float64
    @width * @height
  end

  def name : String
    "สี่เหลี่ยม"
  end
end

class NullShape < Shape
  def area : Float64
    0.0
  end

  def name : String
    "ไม่มีรูปร่าง"
  end
end

# ใช้ NullShape แทน nil
def find_shape(id : Int32) : Shape
  shapes = {1 => Rectangle.new(5.0, 3.0)}
  shapes[id]? || NullShape.new
end

shape1 = find_shape(1)
puts "#{shape1.name}: #{shape1.area}"  # => สี่เหลี่ยม: 15.0

shape2 = find_shape(99)
puts "#{shape2.name}: #{shape2.area}"  # => ไม่มีรูปร่าง: 0.0
```

---

## 9.11 Union Types ใน Collections

```crystal
# Array ของ Union Types
mixed : Array(Int32 | String) = [1, "two", 3, "four"]

mixed.each do |item|
  case item
  when Int32
    puts "Number: #{item}"
  when String
    puts "String: #{item}"
  end
end

# Hash ที่มี Union value
data : Hash(String, Int32 | String | Bool) = {
  "name"   => "Alice",
  "age"    => 30,
  "active" => true
}

data.each do |key, value|
  case value
  when String  then puts "#{key}: \"#{value}\""
  when Int32   then puts "#{key}: #{value} (int)"
  when Bool    then puts "#{key}: #{value} (bool)"
  end
end
```

---

## 9.12 as? และ as

```crystal
value : Int32 | String = "hello"

# as? - ลอง cast, คืน nil ถ้าไม่ได้
str = value.as?(String)
puts str.inspect  # => "hello"

num = value.as?(Int32)
puts num.inspect  # => nil

# as - cast โดยตรง, raise ถ้าไม่ได้
str2 = value.as(String)
puts str2.upcase  # => HELLO

begin
  num2 = value.as(Int32)  # จะ raise
rescue TypeCastError => e
  puts "ไม่สามารถ cast ได้: #{e.message}"
end
```

---

## 9.13 ตัวอย่างโปรแกรมจริง: User Registration Validator

```crystal
class UserRegistrationValidator
  record ValidationError, field : String, message : String

  def self.validate(
    username : String?,
    email : String?,
    age : Int32?
  ) : Array(ValidationError)
    errors = [] of ValidationError

    # ตรวจสอบ username
    if username.nil?
      errors << ValidationError.new("username", "กรุณากรอก username")
    elsif username.empty?
      errors << ValidationError.new("username", "username ต้องไม่ว่างเปล่า")
    elsif username.size < 3
      errors << ValidationError.new("username", "username ต้องมีอย่างน้อย 3 ตัวอักษร")
    end

    # ตรวจสอบ email
    if email.nil?
      errors << ValidationError.new("email", "กรุณากรอก email")
    elsif !email.includes?("@")
      errors << ValidationError.new("email", "รูปแบบ email ไม่ถูกต้อง")
    end

    # ตรวจสอบ age
    if age.nil?
      errors << ValidationError.new("age", "กรุณากรอกอายุ")
    elsif age < 13
      errors << ValidationError.new("age", "ต้องมีอายุอย่างน้อย 13 ปี")
    elsif age > 150
      errors << ValidationError.new("age", "อายุไม่ถูกต้อง")
    end

    errors
  end
end

# ทดสอบ
errors = UserRegistrationValidator.validate(
  username: "al",
  email: "not-an-email",
  age: 10
)

if errors.empty?
  puts "ลงทะเบียนสำเร็จ!"
else
  puts "มีข้อผิดพลาด:"
  errors.each { |e| puts "  - #{e.field}: #{e.message}" }
end

# Output:
# มีข้อผิดพลาด:
#   - username: username ต้องมีอย่างน้อย 3 ตัวอักษร
#   - email: รูปแบบ email ไม่ถูกต้อง
#   - age: ต้องมีอายุอย่างน้อย 13 ปี
```

---

## 9.14 ตัวอย่างโปรแกรมจริง: JSON-like Data Parser

```crystal
# จำลองการทำงานกับ JSON data ที่อาจมี nil
alias JsonValue = String | Int32 | Float64 | Bool | Nil | Array(JsonValue) | Hash(String, JsonValue)

def get_string(data : Hash(String, JsonValue), key : String, default : String = "") : String
  value = data[key]?
  case value
  when String then value
  when Nil    then default
  else             value.to_s
  end
end

def get_int(data : Hash(String, JsonValue), key : String, default : Int32 = 0) : Int32
  value = data[key]?
  case value
  when Int32  then value
  when String then value.to_i? || default
  when Nil    then default
  else             default
  end
end

# ทดสอบ
user_data = {
  "name"  => "Alice",
  "age"   => 30,
  "email" => nil
} of String => JsonValue

puts get_string(user_data, "name")          # => Alice
puts get_string(user_data, "email", "N/A")  # => N/A
puts get_string(user_data, "phone", "N/A")  # => N/A
puts get_int(user_data, "age")              # => 30
puts get_int(user_data, "score", -1)        # => -1
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Safe Calculator
สร้าง method `safe_calculate` ที่รับ `String?` สองตัวและ operator `String?` แล้วคืน `Float64?`
- ถ้า input ใดเป็น nil ให้คืน nil
- ถ้าแปลงตัวเลขไม่ได้ให้คืน nil
- รองรับ +, -, *, /
- ถ้าหารด้วยศูนย์ให้คืน nil

```crystal
def safe_calculate(a : String?, b : String?, op : String?) : Float64?
  # TODO: implement
end

# ทดสอบ
puts safe_calculate("10", "5", "+").inspect   # => 15.0
puts safe_calculate("10", "5", "/").inspect   # => 2.0
puts safe_calculate("10", "0", "/").inspect   # => nil
puts safe_calculate(nil, "5", "+").inspect    # => nil
puts safe_calculate("abc", "5", "+").inspect  # => nil
```

### แบบฝึกหัดที่ 2: Nil-safe Chain
```crystal
class Company
  property name : String
  property ceo : Employee?

  def initialize(@name : String, @ceo : Employee? = nil)
  end
end

class Employee
  property name : String
  property department : Department?

  def initialize(@name : String, @department : Department? = nil)
  end
end

class Department
  property name : String
  property budget : Int32?

  def initialize(@name : String, @budget : Int32? = nil)
  end
end

# สร้าง method ที่คืน budget ของ department ที่ CEO ทำงานอยู่
# ถ้าไม่มีข้อมูลให้คืน 0
def get_ceo_department_budget(company : Company) : Int32
  # TODO: implement using &. operator
end
```

### แบบฝึกหัดที่ 3: Union Type Collection
```crystal
# สร้าง method ที่รับ Array(Int32 | String | Nil)
# และคืน Hash ที่แยก elements ตาม type
def categorize(items : Array(Int32 | String | Nil)) : Hash(String, Array(Int32 | String))
  # TODO: implement
  # Return: {"numbers" => [...], "strings" => [...], "nil_count" => count}
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **Nil Safety**: Crystal บังคับให้จัดการ nil อย่างชัดเจน ป้องกัน null pointer exceptions
2. **String?**: การใช้ `?` หลัง type เพื่อบอกว่าอาจเป็น nil ได้
3. **Union Types**: `Int32 | String` สำหรับค่าที่เป็นได้หลาย type
4. **Nil Coalescing `||`**: ให้ค่าเริ่มต้นเมื่อค่าเป็น nil หรือ false
5. **Safe Navigation `&.`**: เรียก method โดยไม่ crash เมื่อ object เป็น nil
6. **not_nil!**: บอก compiler ว่าไม่ใช่ nil (อันตรายถ้าเป็น nil จริง)
7. **compact**: กรอง nil ออกจาก Array
8. **Type Narrowing**: Crystal จำกัด type อัตโนมัติใน if/case blocks
9. **Practical Patterns**: Early return, default values, safe chaining, null object

### ตารางสรุป Nil Safety Operations

| Operation | ใช้เมื่อ | ผลลัพธ์เมื่อ nil |
|-----------|---------|-----------------|
| `value || default` | ต้องการค่าเริ่มต้น | ได้ default |
| `value&.method` | เรียก method อาจเป็น nil | nil |
| `value.nil?` | ตรวจสอบเป็น nil ไหม | true |
| `value.not_nil!` | มั่นใจว่าไม่ nil | raise NilAssertionError |
| `array.compact` | กรอง nil จาก array | array ที่ไม่มี nil |
| `value.is_a?(Nil)` | ตรวจสอบ type | true |

ระบบ Nil Safety ของ Crystal ช่วยให้โค้ดปลอดภัยและชัดเจนมากขึ้น ทำให้ bug ที่เกี่ยวกับ null ลดลงอย่างมาก!
