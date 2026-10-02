# Part 30: Method Overloading

## บทนำ

Method Overloading คือความสามารถในการกำหนดเมธอดที่มีชื่อเดียวกันหลายตัว แต่มี parameter types หรือจำนวน parameters แตกต่างกัน Crystal รองรับ method overloading แบบ full โดยระบบ type จะเลือก implementation ที่ตรงที่สุดในช่วง compile time

---

## 1. Overloading ตาม Parameter Types

### 1.1 ตัวอย่างพื้นฐาน

```crystal
# เมธอดชื่อเดียวกัน ต่าง parameter types
def process(value : Int32)
  puts "Int32: #{value * 2}"
end

def process(value : Float64)
  puts "Float64: #{value.round(2)}"
end

def process(value : String)
  puts "String: #{value.upcase}"
end

def process(value : Bool)
  puts "Bool: #{!value}"
end

process(42)       # => Int32: 84
process(3.14)     # => Float64: 3.14
process("hello")  # => String: HELLO
process(true)     # => Bool: false
```

### 1.2 Overloading กับ Complex Types

```crystal
def describe(items : Array(Int32))
  puts "Array ของ Int32 มี #{items.size} ตัว"
  puts "ผลรวม: #{items.sum}"
end

def describe(items : Array(String))
  puts "Array ของ String มี #{items.size} รายการ"
  puts "รวม: #{items.join(", ")}"
end

def describe(item : Hash(String, Int32))
  puts "Hash มี #{item.size} คู่"
  item.each { |k, v| puts "  #{k}: #{v}" }
end

describe([1, 2, 3, 4, 5])
describe(["apple", "banana", "cherry"])
describe({"a" => 1, "b" => 2})
```

### 1.3 Union Type vs Overload

```crystal
# วิธีที่ 1: Union type (รวมทุกอย่างในเมธอดเดียว)
def handle_union(value : Int32 | String | Bool)
  case value
  when Int32 then puts "Int: #{value}"
  when String then puts "Str: #{value}"
  when Bool then puts "Bool: #{value}"
  end
end

# วิธีที่ 2: Overloading (แยก implementation)
def handle_overloaded(value : Int32)
  puts "Int: #{value}"
end

def handle_overloaded(value : String)
  puts "Str: #{value}"
end

def handle_overloaded(value : Bool)
  puts "Bool: #{value}"
end

# ทั้งสองวิธีให้ผลเหมือนกัน แต่ overloading ชัดเจนกว่า
```

---

## 2. Overloading ตามจำนวน Parameters (Arity)

### 2.1 ต่าง Arity

```crystal
def connect(host : String)
  puts "เชื่อมต่อ #{host}:80 (default)"
end

def connect(host : String, port : Int32)
  puts "เชื่อมต่อ #{host}:#{port}"
end

def connect(host : String, port : Int32, timeout : Int32)
  puts "เชื่อมต่อ #{host}:#{port} (timeout: #{timeout}s)"
end

connect("localhost")               # => เชื่อมต่อ localhost:80 (default)
connect("example.com", 8080)      # => เชื่อมต่อ example.com:8080
connect("db.server", 5432, 30)    # => เชื่อมต่อ db.server:5432 (timeout: 30s)
```

### 2.2 Point สามมิติ

```crystal
struct Point
  getter x : Float64
  getter y : Float64
  getter z : Float64

  def initialize(@x : Float64, @y : Float64)
    @z = 0.0
  end

  def initialize(@x : Float64, @y : Float64, @z : Float64)
  end
end

p2d = Point.new(1.0, 2.0)
p3d = Point.new(1.0, 2.0, 3.0)

puts "2D: (#{p2d.x}, #{p2d.y}, #{p2d.z})"  # => 2D: (1.0, 2.0, 0.0)
puts "3D: (#{p3d.x}, #{p3d.y}, #{p3d.z})"  # => 3D: (1.0, 2.0, 3.0)
```

---

## 3. Overloading กับ Default Parameters

### 3.1 Default Parameters เป็น Shorthand ของ Overload

```crystal
# Default parameters ทำงานคล้าย overloading
def greet(name : String, greeting : String = "สวัสดี")
  puts "#{greeting}, #{name}!"
end

# เทียบเท่ากับ
def greet_overloaded(name : String)
  greet_overloaded(name, "สวัสดี")
end

def greet_overloaded(name : String, greeting : String)
  puts "#{greeting}, #{name}!"
end

greet("สมชาย")            # => สวัสดี, สมชาย!
greet("สมหญิง", "หวัดดี")  # => หวัดดี, สมหญิง!
```

### 3.2 Overload ที่ต่างจาก Default

```crystal
# บางครั้ง overloading ให้ flexibility มากกว่า default params
def format_date(date : Time)
  date.to_s("%Y-%m-%d")
end

def format_date(date : Time, format : String)
  date.to_s(format)
end

def format_date(year : Int32, month : Int32, day : Int32)
  Time.local(year, month, day).to_s("%Y-%m-%d")
end

now = Time.local
puts format_date(now)                     # => 2024-01-15 (current date)
puts format_date(now, "%d/%m/%Y")         # => 15/01/2024
puts format_date(2024, 1, 15)             # => 2024-01-15
```

---

## 4. Overload Resolution Rules

### 4.1 Crystal เลือก Overload ที่ Specific ที่สุด

```crystal
def show(value)  # รับทุกประเภท
  puts "Generic: #{value}"
end

def show(value : Int32)  # เฉพาะ Int32
  puts "Int32: #{value}"
end

def show(value : Number)  # สำหรับ Number (supertype)
  puts "Number: #{value}"
end

show(42)          # => Int32: 42 (เลือก specific ที่สุด)
show(3.14)        # => Number: 3.14 (Float64 เป็น Number)
show("hello")     # => Generic: hello (ไม่ match type อื่น)
show(true)        # => Generic: true
```

### 4.2 Inheritance กับ Overload Resolution

```crystal
class Animal
end

class Dog < Animal
end

class GoldenRetriever < Dog
end

def pet(animal : Animal)
  puts "สัตว์ทั่วไป"
end

def pet(dog : Dog)
  puts "สุนัข"
end

def pet(golden : GoldenRetriever)
  puts "โกลเดนรีทรีฟเวอร์"
end

pet(Animal.new)          # => สัตว์ทั่วไป
pet(Dog.new)             # => สุนัข
pet(GoldenRetriever.new) # => โกลเดนรีทรีฟเวอร์
```

### 4.3 Ambiguous Overloads

```crystal
def ambiguous(a : Int32, b : String)
  puts "Int32, String"
end

def ambiguous(a : String, b : Int32)
  puts "String, Int32"
end

ambiguous(1, "hello")   # => Int32, String
ambiguous("hello", 1)   # => String, Int32
# ambiguous(1, 1)  # จะเลือก overload ที่ match
```

---

## 5. Practical Overloading Patterns

### 5.1 Serializer/Converter

```crystal
class Converter
  def to_string(value : Int32) : String
    value.to_s
  end

  def to_string(value : Float64) : String
    "%.4f" % value
  end

  def to_string(value : Bool) : String
    value ? "true" : "false"
  end

  def to_string(value : Array) : String
    "[#{value.map { |v| to_string(v) }.join(", ")}]"
  end

  def to_string(value : Nil) : String
    "null"
  end
end

conv = Converter.new
puts conv.to_string(42)          # => "42"
puts conv.to_string(3.14159)     # => "3.1416"
puts conv.to_string(true)        # => "true"
puts conv.to_string(nil)         # => "null"
```

### 5.2 HTTP Response Builder

```crystal
class Response
  getter status : Int32
  getter body : String
  getter headers : Hash(String, String)

  def initialize(@status : Int32, @body : String)
    @headers = {} of String => String
  end

  def initialize(@status : Int32, @body : String, @headers : Hash(String, String))
  end
end

def respond(body : String)
  Response.new(200, body)
end

def respond(status : Int32, body : String)
  Response.new(status, body)
end

def respond(status : Int32, body : String, headers : Hash(String, String))
  Response.new(status, body, headers)
end

r1 = respond("Hello!")
r2 = respond(404, "Not Found")
r3 = respond(200, "OK", {"Content-Type" => "application/json"})

puts "#{r1.status}: #{r1.body}"
puts "#{r2.status}: #{r2.body}"
puts "#{r3.status}: #{r3.body} (#{r3.headers})"
```

### 5.3 Math Operations

```crystal
class MathOps
  def add(a : Int32, b : Int32) : Int32
    a + b
  end

  def add(a : Float64, b : Float64) : Float64
    a + b
  end

  def add(a : Int32, b : Float64) : Float64
    a.to_f + b
  end

  def add(items : Array(Int32)) : Int32
    items.sum
  end

  def add(items : Array(Float64)) : Float64
    items.sum
  end
end

math = MathOps.new
puts math.add(1, 2)                        # => 3
puts math.add(1.5, 2.3)                    # => 3.8
puts math.add(1, 2.5)                      # => 3.5
puts math.add([1, 2, 3, 4, 5])             # => 15
puts math.add([1.1, 2.2, 3.3])             # => 6.6
```

### 5.4 Logger ที่ Overloaded

```crystal
enum LogLevel
  Debug
  Info
  Warning
  Error
end

class Logger
  def initialize(@name : String, @min_level : LogLevel = LogLevel::Info)
  end

  # Overload 1: level enum + message
  def log(level : LogLevel, message : String)
    return if level < @min_level
    puts "[#{@name}] [#{level}] #{message}"
  end

  # Overload 2: string level + message
  def log(level : String, message : String)
    log_level = case level.downcase
    when "debug" then LogLevel::Debug
    when "info" then LogLevel::Info
    when "warning", "warn" then LogLevel::Warning
    when "error" then LogLevel::Error
    else LogLevel::Info
    end
    log(log_level, message)
  end

  # Overload 3: message only (default Info level)
  def log(message : String)
    log(LogLevel::Info, message)
  end

  # Overload 4: message + exception
  def log(level : LogLevel, message : String, exception : Exception)
    log(level, "#{message}: #{exception.message}")
  end
end

logger = Logger.new("App", LogLevel::Info)
logger.log("เริ่มต้นระบบ")
logger.log(LogLevel::Warning, "ใกล้หน่วยความจำเต็ม")
logger.log("error", "ไม่สามารถเชื่อมต่อ")
```

---

## 6. Overloading กับ Generics

### 6.1 Generic Method Overload

```crystal
# เมธอด generic ทำงานร่วมกับ overloaded
def format(value : T) : String forall T
  value.to_s
end

def format(value : Float64) : String
  "%.2f" % value
end

def format(value : Array(T)) : String forall T
  "[#{value.map { |v| format(v) }.join(", ")}]"
end

puts format(42)           # => "42"
puts format(3.14159)      # => "3.14"
puts format("hello")      # => "hello"
puts format([1, 2, 3])    # => "[1, 2, 3]"
puts format([1.5, 2.7])   # => "[1.50, 2.70]"
```

### 6.2 Collection Operations

```crystal
def sum(a : Int32, b : Int32) : Int32
  a + b
end

def sum(a : Float64, b : Float64) : Float64
  a + b
end

def sum(items : Array(Int32)) : Int32
  items.reduce(0) { |acc, x| acc + x }
end

def sum(items : Array(Float64)) : Float64
  items.reduce(0.0) { |acc, x| acc + x }
end

puts sum(1, 2)                      # => 3
puts sum(1.5, 2.5)                  # => 4.0
puts sum([1, 2, 3, 4, 5])           # => 15
puts sum([1.1, 2.2, 3.3])           # => 6.6
```

---

## 7. Overloading กับ Modules และ Classes

### 7.1 Overload ใน Class

```crystal
class Vector
  getter x : Float64
  getter y : Float64

  def initialize(@x : Float64, @y : Float64)
  end

  # Overload +: Vector + Vector
  def +(other : Vector) : Vector
    Vector.new(@x + other.x, @y + other.y)
  end

  # Overload +: Vector + scalar
  def +(scalar : Float64) : Vector
    Vector.new(@x + scalar, @y + scalar)
  end

  # Overload *: Vector * scalar
  def *(scalar : Float64) : Vector
    Vector.new(@x * scalar, @y * scalar)
  end

  # Overload ==: Vector == Vector
  def ==(other : Vector) : Bool
    @x == other.x && @y == other.y
  end

  def to_s
    "(#{@x}, #{@y})"
  end
end

v1 = Vector.new(1.0, 2.0)
v2 = Vector.new(3.0, 4.0)

puts v1 + v2        # => (4.0, 6.0)
puts v1 + 5.0       # => (6.0, 7.0)
puts v1 * 3.0       # => (3.0, 6.0)
puts v1 == Vector.new(1.0, 2.0)  # => true
```

### 7.2 Operator Overloading

```crystal
class Money
  getter amount : Float64
  getter currency : String

  def initialize(@amount : Float64, @currency : String = "THB")
  end

  def +(other : Money) : Money
    raise "ไม่สามารถบวกสกุลเงินต่างกัน" unless @currency == other.currency
    Money.new(@amount + other.amount, @currency)
  end

  def +(amount : Float64) : Money
    Money.new(@amount + amount, @currency)
  end

  def *(multiplier : Float64) : Money
    Money.new(@amount * multiplier, @currency)
  end

  def >(other : Money) : Bool
    @amount > other.amount
  end

  def to_s
    "#{@currency} #{@amount.round(2)}"
  end
end

price = Money.new(100.0)
tax = Money.new(7.0)
total = price + tax
puts total         # => THB 107.0

doubled = price * 2.0
puts doubled       # => THB 200.0

puts price > Money.new(50.0)   # => true
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Shape Calculator

```crystal
struct Circle
  getter radius : Float64
  def initialize(@radius : Float64); end
end

struct Rectangle
  getter width : Float64
  getter height : Float64
  def initialize(@width : Float64, @height : Float64); end
end

struct Triangle
  getter base : Float64
  getter height : Float64
  def initialize(@base : Float64, @height : Float64); end
end

def area(shape : Circle) : Float64
  Math::PI * shape.radius ** 2
end

def area(shape : Rectangle) : Float64
  shape.width * shape.height
end

def area(shape : Triangle) : Float64
  0.5 * shape.base * shape.height
end

def perimeter(shape : Circle) : Float64
  2 * Math::PI * shape.radius
end

def perimeter(shape : Rectangle) : Float64
  2 * (shape.width + shape.height)
end

shapes = [
  Circle.new(5.0),
  Rectangle.new(4.0, 6.0),
  Triangle.new(3.0, 4.0),
]

shapes.each do |shape|
  puts "#{shape.class}: area = #{area(shape).round(2)}"
end
```

### แบบฝึกหัดที่ 2: String Parser

```crystal
def parse(text : String) : Int32?
  text.to_i?
end

def parse(text : String, as_type : Float64.class) : Float64?
  text.to_f?
end

def parse(text : String, as_type : Bool.class) : Bool?
  case text.downcase
  when "true", "1", "yes" then true
  when "false", "0", "no" then false
  else nil
  end
end

def parse(texts : Array(String)) : Array(Int32?)
  texts.map { |t| t.to_i? }
end

puts parse("42").inspect         # => 42
puts parse("3.14", Float64).inspect  # => 3.14
puts parse("true", Bool).inspect # => true
puts parse("yes", Bool).inspect  # => true
puts parse(["1", "2", "abc"]).inspect  # => [1, 2, nil]
```

---

## สรุป

Method Overloading ใน Crystal เป็นคุณสมบัติที่ทรงพลังและ type-safe:

| หัวข้อ | คำอธิบาย |
|--------|---------|
| Type-based overloading | เมธอดเดียวกัน ต่าง parameter types |
| Arity-based overloading | เมธอดเดียวกัน ต่างจำนวน parameters |
| Resolution rules | Crystal เลือก overload ที่ specific ที่สุด |
| Operator overloading | นิยาม `+`, `-`, `*`, `==` และอื่นๆ |

**Best Practices:**
- ใช้ overloading เมื่อเมธอดมีความหมายเดียวกัน แต่รับ input ต่างกัน
- หลีกเลี่ยง overloading ที่ทำให้สับสน
- ใช้ default parameters เมื่อ logic เหมือนกัน แต่มีค่า default
- Overloading ทำงานดีกว่า Union types ในหลายกรณี เพราะชัดเจนกว่า
