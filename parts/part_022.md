# Part 022: Parameters และ Arguments

## บทนำ

การออกแบบ parameter ของ method ให้ดีเป็นเรื่องสำคัญ Crystal มีระบบ parameter ที่ยืดหยุ่นและทรงพลัง ตั้งแต่ positional parameters ธรรมดา ไปจนถึง variadic parameters, keyword arguments และ double splat สำหรับ named arguments จำนวนไม่จำกัด

---

## 1. Positional Parameters

```crystal
# Parameter พื้นฐาน
def add(a : Int32, b : Int32) : Int32
  a + b
end

puts add(3, 4)  # => 7

# ต้องส่ง argument ครบตามลำดับ
def greet(greeting : String, name : String)
  puts "#{greeting}, #{name}!"
end

greet("Hello", "Alice")  # => Hello, Alice!
greet("Alice", "Hello")  # => Alice, Hello! (สลับกัน - ผิดลำดับ)
```

### Multiple Positional Parameters

```crystal
def calculate_bmi(weight_kg : Float64, height_m : Float64) : Float64
  weight_kg / (height_m ** 2)
end

bmi = calculate_bmi(70.0, 1.75)
puts "BMI: #{bmi.round(1)}"  # => BMI: 22.9

def create_rectangle(x : Int32, y : Int32, width : Int32, height : Int32)
  puts "Rectangle at (#{x}, #{y}) size #{width}x#{height}"
end

create_rectangle(10, 20, 100, 50)
```

---

## 2. Typed Parameters

Crystal ใช้ type restriction บน parameter เพื่อ type safety:

```crystal
# Type restriction ชัดเจน
def multiply(a : Int32, b : Int32) : Int32
  a * b
end

def concat(a : String, b : String) : String
  a + b
end

# Union type
def display(value : Int32 | String | Float64)
  puts "Value: #{value} (#{value.class})"
end

display(42)       # => Value: 42 (Int32)
display("hello")  # => Value: hello (String)
display(3.14)     # => Value: 3.14 (Float64)
```

### Nilable Parameters

```crystal
# Parameter ที่อาจเป็น nil
def greet_optional(name : String?)
  if name
    puts "Hello, #{name}!"
  else
    puts "Hello, Stranger!"
  end
end

greet_optional("Alice")  # => Hello, Alice!
greet_optional(nil)      # => Hello, Stranger!
```

### Generic Type Parameters

```crystal
# Generic method (ใช้กับ type ใดก็ได้ที่ implement Comparable)
def maximum(a : T, b : T) : T forall T
  a > b ? a : b
end

puts maximum(3, 7)          # => 7
puts maximum("apple", "banana")  # => banana
puts maximum(3.14, 2.71)    # => 3.14
```

---

## 3. Variable Number of Arguments (*args)

`*` ใช้รับ argument จำนวนไม่จำกัด:

```crystal
# *args รับเป็น Tuple
def sum(*numbers : Int32) : Int32
  numbers.sum
end

puts sum(1)              # => 1
puts sum(1, 2, 3)        # => 6
puts sum(1, 2, 3, 4, 5)  # => 15

# ดูประเภทของ *args
def show_type(*args)
  puts args.class  # => Tuple(...)
  args.each { |a| puts "  #{a}: #{a.class}" }
end

show_type(1, "hello", 3.14)
```

### *args ร่วมกับ Positional Parameters

```crystal
# *args ต้องอยู่หลัง positional parameters ปกติ
def log(level : String, *messages : String)
  messages.each do |msg|
    puts "[#{level}] #{msg}"
  end
end

log("INFO", "เริ่มต้นระบบ")
log("ERROR", "เชื่อมต่อ DB ล้มเหลว", "Retrying...", "Still failed")
```

### Splat ใน Method Call

```crystal
# ส่ง array เป็น individual arguments ด้วย *
def add_three(a : Int32, b : Int32, c : Int32) : Int32
  a + b + c
end

numbers = [1, 2, 3]
puts add_three(*numbers)  # => 6 (splat เป็น individual args)

# ใช้กับ range
args = (1..5).to_a
def sum5(a : Int32, b : Int32, c : Int32, d : Int32, e : Int32) : Int32
  a + b + c + d + e
end
puts sum5(*args)  # => 15
```

---

## 4. Keyword Arguments

```crystal
# Keyword arguments ใช้ชื่อ parameter เมื่อเรียก
def create_user(name : String, age : Int32, email : String)
  puts "สร้างผู้ใช้: #{name}, อายุ #{age}, email: #{email}"
end

# เรียกแบบ positional
create_user("Alice", 30, "alice@example.com")

# เรียกแบบ keyword (ลำดับใดก็ได้)
create_user(email: "alice@example.com", name: "Alice", age: 30)
create_user(name: "Bob", age: 25, email: "bob@example.com")
```

### ผสม Positional และ Keyword

```crystal
def connect(host : String, port : Int32, timeout : Int32, ssl : Bool)
  puts "Connecting to #{host}:#{port} (timeout: #{timeout}s, ssl: #{ssl})"
end

# positional ก่อน keyword ได้
connect("localhost", 5432, timeout: 30, ssl: false)
connect("db.example.com", 5432, timeout: 60, ssl: true)
```

---

## 5. Double Splat **kwargs

`**` ใช้รับ keyword arguments จำนวนไม่จำกัดเป็น NamedTuple:

```crystal
# **kwargs รับเป็น NamedTuple
def show_options(**options)
  puts "Options:"
  options.each do |key, value|
    puts "  #{key}: #{value}"
  end
end

show_options(color: "red", size: 12, bold: true)
# Options:
#   color: red
#   size: 12
#   bold: true
```

### **kwargs กับ Type Restriction

```crystal
# ระบุ type ของ value
def configure(**settings : Int32)
  settings.each do |key, value|
    puts "#{key} = #{value}"
  end
end

configure(timeout: 30, max_retries: 3, pool_size: 10)
```

### **kwargs ใน Real Application

```crystal
def render_html_tag(tag : String, content : String, **attributes)
  attrs = attributes.map { |k, v| " #{k}=\"#{v}\"" }.join
  "<#{tag}#{attrs}>#{content}</#{tag}>"
end

puts render_html_tag("a", "Click here",
  href: "https://example.com",
  target: "_blank",
  class: "btn btn-primary"
)
# => <a href="https://example.com" target="_blank" class="btn btn-primary">Click here</a>

puts render_html_tag("img", "",
  src: "/images/logo.png",
  alt: "Logo",
  width: "200"
)
# => <img src="/images/logo.png" alt="Logo" width="200"></img>
```

---

## 6. Type Restrictions บน Parameters

```crystal
# Restriction ด้วย interface (module)
def process_each(collection, &block)
  collection.each { |item| block.call(item) }
end

# Restriction ด้วย abstract type
def format_value(value : Number) : String
  value.to_s
end

puts format_value(42)     # => "42"
puts format_value(3.14)   # => "3.14"
# format_value("hello") => Error: ไม่ match

# Restriction แบบ Union
def stringify(value : Int32 | Float64 | Bool) : String
  value.to_s
end

puts stringify(42)     # => "42"
puts stringify(3.14)   # => "3.14"
puts stringify(true)   # => "true"
```

### Restriction ด้วย Struct/Class

```crystal
struct Point
  getter x : Float64
  getter y : Float64
  
  def initialize(@x : Float64, @y : Float64)
  end
end

def distance(p1 : Point, p2 : Point) : Float64
  Math.sqrt((p2.x - p1.x) ** 2 + (p2.y - p1.y) ** 2)
end

a = Point.new(0.0, 0.0)
b = Point.new(3.0, 4.0)
puts distance(a, b)  # => 5.0
```

---

## 7. Block Parameters (&block)

```crystal
# รับ block เป็น parameter ชัดเจน
def apply_twice(value : Int32, &operation : Int32 -> Int32) : Int32
  operation.call(operation.call(value))
end

result = apply_twice(3) { |n| n * 2 }
puts result  # => 12 (3*2=6, 6*2=12)

# ส่ง block ต่อด้วย &
def transform_all(numbers : Array(Int32), &block : Int32 -> Int32) : Array(Int32)
  numbers.map(&block)
end

result = transform_all([1, 2, 3, 4, 5]) { |n| n ** 2 }
puts result.inspect  # => [1, 4, 9, 16, 25]
```

### ส่ง Proc เป็น Block

```crystal
doubler = ->(n : Int32) { n * 2 }
result = [1, 2, 3, 4, 5].map(&doubler)
puts result.inspect  # => [2, 4, 6, 8, 10]
```

---

## 8. ตัวอย่าง Real-World

### HTTP Request Builder

```crystal
def http_request(
  method : String,
  url : String,
  body : String? = nil,
  timeout : Int32 = 30,
  **headers
) : String
  parts = ["#{method} #{url}"]
  parts << "Timeout: #{timeout}s"
  
  headers.each do |key, value|
    parts << "#{key}: #{value}"
  end
  
  parts << "Body: #{body}" if body
  
  parts.join("\n")
end

puts http_request("GET", "https://api.example.com/users",
  Authorization: "Bearer token123",
  Accept: "application/json"
)

puts "\n---\n"

puts http_request("POST", "https://api.example.com/users",
  body: "{\"name\": \"Alice\"}",
  timeout: 60,
  Content_Type: "application/json",
  Authorization: "Bearer token123"
)
```

### Database Query Builder

```crystal
def build_select_query(
  table : String,
  *columns : String,
  where : String? = nil,
  order_by : String? = nil,
  limit : Int32? = nil
) : String
  cols = columns.empty? ? "*" : columns.join(", ")
  query = "SELECT #{cols} FROM #{table}"
  query += " WHERE #{where}" if where
  query += " ORDER BY #{order_by}" if order_by
  query += " LIMIT #{limit}" if limit
  query
end

puts build_select_query("users")
puts build_select_query("users", "name", "email")
puts build_select_query("users", "name", "age",
  where: "age > 18",
  order_by: "name ASC",
  limit: 10
)
```

### Logger ที่ยืดหยุ่น

```crystal
enum LogLevel
  DEBUG
  INFO
  WARN
  ERROR
end

def log(
  level : LogLevel,
  message : String,
  *context : String,
  timestamp : Bool = true,
  prefix : String = ""
)
  time_str = timestamp ? "[#{Time.local}] " : ""
  prefix_str = prefix.empty? ? "" : "[#{prefix}] "
  ctx_str = context.empty? ? "" : " (#{context.join(", ")})"
  
  puts "#{time_str}#{prefix_str}[#{level}] #{message}#{ctx_str}"
end

log(LogLevel::INFO, "เริ่มต้นระบบ")
log(LogLevel::ERROR, "การเชื่อมต่อล้มเหลว",
  "host: localhost",
  "port: 5432",
  prefix: "DATABASE"
)
log(LogLevel::DEBUG, "Cache miss",
  "key: user_123",
  timestamp: false
)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Variadic Sum

```crystal
# สร้าง method ที่รับตัวเลขจำนวนไม่จำกัด
# แล้วคำนวณ sum, average, min, max

def statistics(*numbers : Float64) : NamedTuple(
  sum: Float64,
  average: Float64,
  min: Float64,
  max: Float64
)
  # TODO: implement
end

stats = statistics(1.0, 2.0, 3.0, 4.0, 5.0)
puts "Sum: #{stats[:sum]}"
puts "Average: #{stats[:average]}"
puts "Min: #{stats[:min]}"
puts "Max: #{stats[:max]}"
```

### แบบฝึกหัดที่ 2: HTML Builder

```crystal
# สร้าง HTML builder ด้วย **attributes
def tag(name : String, content : String = "", **attrs) : String
  # TODO: สร้าง HTML tag พร้อม attributes
  # tag("a", "Click", href: "/home", class: "nav-link")
  # => <a href="/home" class="nav-link">Click</a>
end

def self_closing_tag(name : String, **attrs) : String
  # TODO: self-closing tag
  # self_closing_tag("input", type: "text", name: "username")
  # => <input type="text" name="username" />
end

puts tag("h1", "Welcome")
puts tag("a", "Click here", href: "/home", class: "link")
puts self_closing_tag("input", type: "email", placeholder: "กรอกอีเมล")
```

### แบบฝึกหัดที่ 3: Type-safe Calculator

```crystal
# Method ที่รับ type ต่างกันและทำงานต่างกัน
def add(a : Int32, b : Int32) : Int32
  # TODO
end

def add(a : Float64, b : Float64) : Float64
  # TODO
end

def add(a : String, b : String) : String
  # TODO: concatenate strings
end

def add(a : Array(Int32), b : Array(Int32)) : Array(Int32)
  # TODO: merge arrays
end

puts add(1, 2)            # => 3
puts add(1.5, 2.5)        # => 4.0
puts add("Hello", " World") # => "Hello World"
puts add([1, 2], [3, 4]).inspect  # => [1, 2, 3, 4]
```

### แบบฝึกหัดที่ 4: Config Builder

```crystal
# สร้าง config system ด้วย **kwargs
class AppConfig
  def initialize
    @settings = {} of String => String
  end
  
  def set(**options)
    # TODO: เก็บ options ใน @settings
    # รองรับ: database_host, database_port, cache_ttl เป็นต้น
    self
  end
  
  def get(key : String) : String?
    # TODO: ดึงค่า
  end
  
  def to_s(io : IO) : Nil
    # TODO: แสดงค่าทั้งหมด
  end
end

config = AppConfig.new
config
  .set(database_host: "localhost", database_port: "5432")
  .set(cache_ttl: "3600", debug: "false")

puts config.get("database_host")  # => localhost
puts config
```

---

## สรุป

| Parameter Type | Syntax | ตัวอย่าง |
|---------------|--------|----------|
| Positional | `def f(a : T)` | `f(42)` |
| Typed | `def f(a : Int32 \| String)` | Union type |
| Nilable | `def f(a : T?)` | `f(nil)` |
| Variadic | `def f(*args)` | `f(1, 2, 3)` |
| Keyword | `def f(name : T)` | `f(name: "Alice")` |
| Double Splat | `def f(**kwargs)` | `f(x: 1, y: 2)` |
| Block | `def f(&block : T -> U)` | `f { \|v\| v * 2 }` |
| Generic | `def f(a : T) forall T` | Works with any type |

### Best Practices

1. **ระบุ type** บน parameter เสมอ เพื่อ documentation และ compile-time check
2. **ใช้ keyword args** เมื่อมี parameter มากกว่า 2-3 ตัว
3. **ระวัง *args** - อาจทำให้ signature ไม่ชัดเจน ควรระบุ type
4. **default values** ไว้ด้านหลัง required parameters
5. **block parameter** ใช้ `yield` เมื่อไม่ต้องการ store block; ใช้ `&block` เมื่อต้องส่งต่อ
