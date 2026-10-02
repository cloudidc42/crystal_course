# Part 26: Splat Operators (*args, **kwargs)

## บทนำ

Splat operators เป็นเครื่องมือที่ทรงพลังใน Crystal ที่ช่วยให้เราสามารถรับจำนวน argument ที่ไม่แน่นอนได้ในเมธอด รวมถึงการแพร่กระจาย (spread) ค่าจาก array หรือ hash เข้าไปใน method call

Crystal มี splat operators สองแบบหลัก:
- `*args` - รับ positional arguments จำนวนไม่จำกัด
- `**kwargs` - รับ keyword arguments จำนวนไม่จำกัด

---

## 1. *args - Variable Positional Arguments

### 1.1 พื้นฐานของ *args

```crystal
# เมธอดที่รับจำนวน argument ไม่จำกัด
def sum(*numbers)
  total = 0
  numbers.each { |n| total += n }
  total
end

puts sum(1, 2, 3)          # => 6
puts sum(10, 20)            # => 30
puts sum(1, 2, 3, 4, 5)    # => 15
puts sum                    # => 0 (ไม่มี argument)
```

### 1.2 ประเภทของ *args

```crystal
# *args มีประเภทเป็น Tuple
def show_args(*args)
  puts args.class    # => Tuple(...)
  puts args.inspect  # แสดงค่าทั้งหมด
end

show_args(1, "hello", true)
# => Tuple(Int32, String, Bool)
# => {1, "hello", true}
```

### 1.3 *args ร่วมกับ argument ปกติ

```crystal
# *args ต้องอยู่หลัง positional arguments ปกติ
def greet(greeting, *names)
  names.each do |name|
    puts "#{greeting}, #{name}!"
  end
end

greet("สวัสดี", "สมชาย", "สมหญิง", "สมศักดิ์")
# => สวัสดี, สมชาย!
# => สวัสดี, สมหญิง!
# => สวัสดี, สมศักดิ์!
```

### 1.4 *args ก่อน argument อื่น

```crystal
# *args สามารถอยู่ก่อน argument สุดท้ายได้
def process(*items, separator : String)
  items.join(separator)
end

result = process("a", "b", "c", separator: " - ")
puts result  # => a - b - c
```

### 1.5 ตัวอย่างการใช้งานจริง

```crystal
# ฟังก์ชันสำหรับ logging
def log(level : String, *messages)
  timestamp = Time.local.to_s
  messages.each do |msg|
    puts "[#{timestamp}] [#{level.upcase}] #{msg}"
  end
end

log("info", "แอปพลิเคชันเริ่มทำงาน")
log("error", "ไม่สามารถเชื่อมต่อฐานข้อมูล", "รายละเอียด: timeout")
log("debug", "req id: 123", "user: สมชาย", "action: login")
```

---

## 2. **kwargs - Variable Keyword Arguments

### 2.1 พื้นฐานของ **kwargs

```crystal
# เมธอดที่รับ keyword arguments จำนวนไม่จำกัด
def configure(**options)
  options.each do |key, value|
    puts "#{key}: #{value}"
  end
end

configure(host: "localhost", port: 3000, debug: true)
# => host: localhost
# => port: 3000
# => debug: true
```

### 2.2 ประเภทของ **kwargs

```crystal
# **kwargs มีประเภทเป็น NamedTuple
def show_kwargs(**kwargs)
  puts kwargs.class    # => NamedTuple(...)
  puts kwargs.inspect
end

show_kwargs(name: "Crystal", version: "1.0")
# NamedTuple(name: String, version: String)
# {name: "Crystal", version: "1.0"}
```

### 2.3 **kwargs ร่วมกับ argument ปกติ

```crystal
def create_user(name : String, email : String, **extra)
  puts "ชื่อ: #{name}"
  puts "อีเมล: #{email}"
  
  unless extra.empty?
    puts "ข้อมูลเพิ่มเติม:"
    extra.each do |key, val|
      puts "  #{key}: #{val}"
    end
  end
end

create_user("สมชาย", "somchai@example.com", age: 25, city: "กรุงเทพ")
# => ชื่อ: สมชาย
# => อีเมล: somchai@example.com
# => ข้อมูลเพิ่มเติม:
# =>   age: 25
# =>   city: กรุงเทพ
```

### 2.4 ตัวอย่าง HTTP request builder

```crystal
def http_get(url : String, **params)
  query_string = params.map { |k, v| "#{k}=#{v}" }.join("&")
  full_url = params.empty? ? url : "#{url}?#{query_string}"
  puts "GET #{full_url}"
end

http_get("https://api.example.com/users")
# => GET https://api.example.com/users

http_get("https://api.example.com/users",
  page: 1,
  limit: 20,
  sort: "created_at"
)
# => GET https://api.example.com/users?page=1&limit=20&sort=created_at
```

---

## 3. Splat ใน Method Calls (Spreading)

### 3.1 การแพร่กระจาย Array เข้าเมธอด

```crystal
def add(a, b, c)
  a + b + c
end

numbers = [1, 2, 3]
result = add(*numbers)  # แพร่กระจาย array เป็น arguments
puts result  # => 6
```

### 3.2 Splat กับ Array บางส่วน

```crystal
def show(a, b, c, d, e)
  puts "#{a}, #{b}, #{c}, #{d}, #{e}"
end

first = [1, 2]
last = [4, 5]
show(*first, 3, *last)  # => 1, 2, 3, 4, 5
```

### 3.3 ตัวอย่าง Range กับ Splat

```crystal
def sum_three(a, b, c)
  a + b + c
end

range = (1..3).to_a
puts sum_three(*range)  # => 6

# หรือโดยตรง
puts sum_three(*(1..3))  # => 6
```

### 3.4 การใช้ Splat กับ Tuple

```crystal
def coordinates(x, y, z)
  puts "X: #{x}, Y: #{y}, Z: #{z}"
end

point = {10, 20, 30}
coordinates(*point)
# => X: 10, Y: 20, Z: 30
```

---

## 4. Double Splat (**) ใน Method Calls

### 4.1 การแพร่กระจาย NamedTuple

```crystal
def describe(name : String, age : Int32, city : String)
  puts "#{name} อายุ #{age} ปี อาศัยที่ #{city}"
end

person = {name: "สมชาย", age: 30, city: "เชียงใหม่"}
describe(**person)
# => สมชาย อายุ 30 ปี อาศัยที่ เชียงใหม่
```

### 4.2 การแพร่กระจาย Hash

```crystal
def configure(host : String, port : Int32)
  puts "เชื่อมต่อ #{host}:#{port}"
end

# Hash ต้องมี String keys
settings = {"host" => "localhost", "port" => 8080}
# ไม่สามารถใช้ ** กับ Hash โดยตรง ต้องใช้ NamedTuple

config = {host: "localhost", port: 8080}
configure(**config)
# => เชื่อมต่อ localhost:8080
```

### 4.3 ผสม * และ ** ใน Call

```crystal
def full_info(*names, **details)
  puts "ชื่อ: #{names.join(", ")}"
  details.each { |k, v| puts "#{k}: #{v}" }
end

people = ["สมชาย", "สมหญิง"]
info = {department: "IT", floor: 3}

full_info(*people, **info)
# => ชื่อ: สมชาย, สมหญิง
# => department: IT
# => floor: 3
```

---

## 5. การ Forward Arguments

### 5.1 Forwarding ทั้งหมด

```crystal
# เมธอด wrapper ที่ส่ง arguments ต่อ
def logged_call(*args, **kwargs)
  puts "กำลังเรียกเมธอดด้วย args: #{args}, kwargs: #{kwargs}"
  # ส่งต่อไปยังเมธอดอื่น
  do_something(*args, **kwargs)
end

def do_something(*args, **kwargs)
  puts "ทำงานกับ: #{args}, #{kwargs}"
end

logged_call(1, 2, name: "test", debug: true)
# => กำลังเรียกเมธอดด้วย args: {1, 2}, kwargs: {name: "test", debug: true}
# => ทำงานกับ: {1, 2}, {name: "test", debug: true}
```

### 5.2 Decorator Pattern ด้วย Splat

```crystal
# Middleware/Decorator pattern
def with_timing(*args, **kwargs)
  start = Time.monotonic
  result = yield(*args, **kwargs)
  elapsed = Time.monotonic - start
  puts "ใช้เวลา: #{elapsed.total_milliseconds}ms"
  result
end
```

### 5.3 การ Forward arguments บางส่วน

```crystal
def create_connection(host : String, port : Int32, **options)
  puts "เชื่อมต่อ #{host}:#{port}"
  options.each { |k, v| puts "  option #{k}: #{v}" }
end

def create_local_connection(**options)
  # Forward เฉพาะบางส่วน พร้อมค่า default
  create_connection("localhost", 3000, **options)
end

create_local_connection(timeout: 30, ssl: false)
# => เชื่อมต่อ localhost:3000
# =>   option timeout: 30
# =>   option ssl: false
```

---

## 6. Splat ใน Array Literals

### 6.1 Spreading Array ใน Array

```crystal
first = [1, 2, 3]
second = [7, 8, 9]
middle = [4, 5, 6]

combined = [*first, *middle, *second]
puts combined.inspect
# => [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### 6.2 การสร้าง Array ใหม่

```crystal
defaults = [0, 0, 0]
custom = [1, 2]

# เพิ่มค่าเข้า array ที่มีอยู่
extended = [*custom, *defaults]
puts extended.inspect  # => [1, 2, 0, 0, 0]
```

---

## 7. ตัวอย่างการใช้งานขั้นสูง

### 7.1 Variadic String Formatter

```crystal
def format_message(template : String, *values)
  result = template
  values.each_with_index do |val, idx|
    result = result.sub("{#{idx}}", val.to_s)
  end
  result
end

msg = format_message("สวัสดี {0}, คุณอายุ {1} ปี และอาศัยที่ {2}",
  "สมชาย", 25, "กรุงเทพ")
puts msg
# => สวัสดี สมชาย, คุณอายุ 25 ปี และอาศัยที่ กรุงเทพ
```

### 7.2 Configuration Builder

```crystal
class AppConfig
  getter settings = {} of String => String

  def configure(**options)
    options.each do |key, value|
      @settings[key.to_s] = value.to_s
    end
    self
  end

  def show
    puts "=== Configuration ==="
    @settings.each { |k, v| puts "  #{k}: #{v}" }
  end
end

config = AppConfig.new
config.configure(
  database_url: "postgresql://localhost/mydb",
  redis_url: "redis://localhost:6379",
  log_level: "debug",
  max_connections: "10"
)
config.show
```

### 7.3 Pipeline ด้วย Splat

```crystal
def pipeline(*steps)
  ->(input : String) {
    result = input
    steps.each do |step|
      result = step.call(result)
    end
    result
  }
end

upcase = ->(s : String) { s.upcase }
trim = ->(s : String) { s.strip }
add_prefix = ->(s : String) { ">>> #{s}" }

process = pipeline(trim, upcase, add_prefix)
puts process.call("  hello world  ")
# => >>> HELLO WORLD
```

### 7.4 Event System

```crystal
class EventEmitter
  def initialize
    @listeners = {} of String => Array(Proc(Array(String), Nil))
  end

  def on(event : String, &handler : Array(String) ->)
    @listeners[event] ||= [] of Proc(Array(String), Nil)
    @listeners[event] << handler
  end

  def emit(event : String, *args)
    return unless @listeners.has_key?(event)
    @listeners[event].each { |handler| handler.call(args.to_a.map(&.to_s)) }
  end
end

emitter = EventEmitter.new

emitter.on("login") do |args|
  puts "ผู้ใช้ #{args[0]} เข้าสู่ระบบจาก #{args[1]}"
end

emitter.emit("login", "สมชาย", "192.168.1.100")
# => ผู้ใช้ สมชาย เข้าสู่ระบบจาก 192.168.1.100
```

---

## 8. ข้อควรระวังและ Best Practices

### 8.1 ตำแหน่งของ *args

```crystal
# ถูกต้อง: *args อยู่ระหว่าง positional args
def valid1(a, *b, c)
  puts "a=#{a}, b=#{b}, c=#{c}"
end

valid1(1, 2, 3, 4, c: 5)  # c ต้องระบุเป็น keyword arg

# ถูกต้อง: *args อยู่ท้าย
def valid2(a, b, *rest)
  puts "a=#{a}, b=#{b}, rest=#{rest}"
end

valid2(1, 2, 3, 4, 5)
```

### 8.2 Type Restrictions กับ *args

```crystal
# จำกัดประเภทของ splat args
def sum_ints(*numbers : Int32)
  numbers.sum
end

puts sum_ints(1, 2, 3)  # => 6
# sum_ints(1, 2.0, 3)  # Error: ต้องเป็น Int32

# Union types
def display(*items : Int32 | String)
  items.each { |item| print "#{item} " }
  puts
end

display(1, "hello", 2, "world")  # => 1 hello 2 world
```

### 8.3 Performance Considerations

```crystal
# *args สร้าง Tuple ซึ่งใช้หน่วยความจำ
# สำหรับ critical path ควรพิจารณาใช้ Array แทน

# แบบที่ต้องการประสิทธิภาพสูง
def process_items(items : Array(Int32))
  items.each { |i| # ... }
end

# แบบทั่วไป (สะดวกกว่า)
def process_items(*items : Int32)
  items.each { |i| # ... }
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Calculator

สร้างเครื่องคิดเลขที่รับจำนวนตัวเลขไม่จำกัด:

```crystal
def calculate(operation : String, *numbers : Float64)
  case operation
  when "sum"
    numbers.sum
  when "avg"
    numbers.sum / numbers.size
  when "min"
    numbers.min
  when "max"
    numbers.max
  when "product"
    numbers.reduce(1.0) { |acc, n| acc * n }
  else
    raise "Unknown operation: #{operation}"
  end
end

puts calculate("sum", 1.0, 2.0, 3.0, 4.0, 5.0)      # => 15.0
puts calculate("avg", 10.0, 20.0, 30.0)               # => 20.0
puts calculate("product", 2.0, 3.0, 4.0)             # => 24.0
```

### แบบฝึกหัดที่ 2: HTML Builder

```crystal
def tag(name : String, *contents, **attributes)
  attrs = attributes.map { |k, v| " #{k}=\"#{v}\"" }.join
  inner = contents.join("\n")
  "<#{name}#{attrs}>\n#{inner}\n</#{name}>"
end

def p_tag(*contents)
  tag("p", *contents)
end

html = tag("div",
  tag("h1", "สวัสดีโลก", class: "title"),
  p_tag("นี่คือย่อหน้าแรก", "และนี่คือข้อความเพิ่มเติม"),
  id: "main",
  class: "container"
)

puts html
```

### แบบฝึกหัดที่ 3: Query Builder

```crystal
def build_query(table : String, *conditions, **options)
  sql = "SELECT * FROM #{table}"
  
  unless conditions.empty?
    where_clause = conditions.join(" AND ")
    sql += " WHERE #{where_clause}"
  end
  
  if options.has_key?(:order_by)
    sql += " ORDER BY #{options[:order_by]}"
  end
  
  if options.has_key?(:limit)
    sql += " LIMIT #{options[:limit]}"
  end
  
  sql
end

puts build_query("users")
puts build_query("users", "age > 18", "active = true")
puts build_query("products", "price < 1000", order_by: "price", limit: 10)
```

### แบบฝึกหัดที่ 4: Validation Framework

```crystal
def validate(value, *rules, **options)
  errors = [] of String
  
  rules.each do |rule|
    case rule
    when :required
      errors << "ต้องระบุค่า" if value.nil? || value.to_s.empty?
    when :numeric
      errors << "ต้องเป็นตัวเลข" unless value.to_s.match?(/^\d+$/)
    when :alpha
      errors << "ต้องเป็นตัวอักษร" unless value.to_s.match?(/^[a-zA-Z]+$/)
    end
  end
  
  if options.has_key?(:min_length)
    min = options[:min_length].to_i
    if value.to_s.size < min
      errors << "ต้องมีความยาวอย่างน้อย #{min} ตัวอักษร"
    end
  end
  
  if options.has_key?(:max_length)
    max = options[:max_length].to_i
    if value.to_s.size > max
      errors << "ต้องมีความยาวไม่เกิน #{max} ตัวอักษร"
    end
  end
  
  errors
end

puts validate("", :required).inspect
# => ["ต้องระบุค่า"]

puts validate("abc123", :alpha).inspect
# => ["ต้องเป็นตัวอักษร"]

puts validate("hi", :required, :alpha, min_length: 5).inspect
# => ["ต้องมีความยาวอย่างน้อย 5 ตัวอักษร"]

puts validate("crystal", :required, :alpha, min_length: 3, max_length: 10).inspect
# => []
```

---

## สรุป

Splat operators ใน Crystal เป็นเครื่องมือที่มีประโยชน์มากสำหรับ:

| Feature | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Variable positional args | `*args` | รับ arguments จำนวนไม่จำกัด |
| Variable keyword args | `**kwargs` | รับ keyword arguments จำนวนไม่จำกัด |
| Spread array | `*array` | แพร่กระจาย array เป็น arguments |
| Spread named tuple | `**named_tuple` | แพร่กระจาย named tuple เป็น kwargs |
| Forward all args | `*args, **kwargs` | ส่งต่อ arguments ไปยังเมธอดอื่น |

**ข้อสำคัญที่ควรจำ:**
- `*args` สร้าง `Tuple` ซึ่ง immutable และ fixed-size
- `**kwargs` สร้าง `NamedTuple` ซึ่งมี named keys
- สามารถใช้ type restrictions กับ splat args ได้
- `*args` ต้องอยู่ก่อน `**kwargs` เสมอ
- การใช้ splat ใน call จะแพร่กระจาย collection เป็น individual arguments
