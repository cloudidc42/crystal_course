# ตอนที่ 14: Case / When

## บทนำ

`case/when` ใน Crystal เป็นเครื่องมือที่ทรงพลังสำหรับ pattern matching ที่ยืดหยุ่นกว่า if/elsif มาก รองรับการ match ด้วยค่า, type, range, regex, และ pattern ซับซ้อน

---

## 14.1 Case / When พื้นฐาน

```crystal
# รูปแบบพื้นฐาน
grade = "B"

case grade
when "A"
  puts "ดีเยี่ยม"
when "B"
  puts "ดี"
when "C"
  puts "พอใช้"
when "D"
  puts "ผ่าน"
when "F"
  puts "ไม่ผ่าน"
else
  puts "เกรดไม่ถูกต้อง"
end

# Output: ดี
```

### Match หลายค่าใน when เดียว

```crystal
day = "Saturday"

case day
when "Monday", "Tuesday", "Wednesday", "Thursday", "Friday"
  puts "วันทำงาน"
when "Saturday", "Sunday"
  puts "วันหยุด"
else
  puts "ไม่รู้จักวัน"
end

# หรือใช้ Array
case day
when *["Saturday", "Sunday"]
  puts "วันหยุดสุดสัปดาห์"
end
```

---

## 14.2 Case ไม่มี Expression

เมื่อไม่ระบุ expression จะทำงานเหมือน if/elsif

```crystal
temperature = 28

case
when temperature < 10
  puts "หนาวมาก"
when temperature < 20
  puts "เย็นสบาย"
when temperature < 30
  puts "อากาศดี"
when temperature < 40
  puts "ร้อน"
else
  puts "ร้อนมาก"
end

# Output: อากาศดี

# เทียบเท่ากับ if/elsif
```

---

## 14.3 Matching กับ Ranges

```crystal
score = 85

case score
when 90..100
  puts "A"
when 80..89
  puts "B"
when 70..79
  puts "C"
when 60..69
  puts "D"
when 0..59
  puts "F"
else
  puts "คะแนนไม่ถูกต้อง"
end

# Output: B

# ใช้ endless range
age = 20
case age
when ..12
  puts "เด็ก"
when 13..17
  puts "วัยรุ่น"
when 18..64
  puts "ผู้ใหญ่"
when 65..
  puts "ผู้สูงอายุ"
end
```

---

## 14.4 Matching กับ Types (is_a?)

case/when ใช้ `===` operator ซึ่ง Class ใช้ `is_a?` ตรวจสอบ

```crystal
def describe(value : Int32 | String | Float64 | Bool | Nil)
  case value
  when Int32
    "จำนวนเต็ม: #{value}"
  when Float64
    "ทศนิยม: #{value}"
  when String
    "ข้อความ: #{value}"
  when Bool
    "บูลีน: #{value}"
  when Nil
    "ไม่มีค่า"
  else
    "ไม่รู้จัก"
  end
end

puts describe(42)          # => จำนวนเต็ม: 42
puts describe(3.14)        # => ทศนิยม: 3.14
puts describe("hello")     # => ข้อความ: hello
puts describe(true)        # => บูลีน: true
puts describe(nil)         # => ไม่มีค่า

# Type narrowing อัตโนมัติ
def process(value : Int32 | String)
  case value
  when Int32
    value * 2          # compiler รู้ว่าเป็น Int32
  when String
    value.upcase       # compiler รู้ว่าเป็น String
  end
end

puts process(21).inspect      # => 42
puts process("hello").inspect # => "HELLO"
```

---

## 14.5 Matching กับ Regex

```crystal
input = "user@example.com"

case input
when /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  puts "อีเมลถูกต้อง"
when /\A\d{10}\z/
  puts "เบอร์โทรศัพท์ 10 หลัก"
when /\A\d{4}-\d{4}-\d{4}-\d{4}\z/
  puts "หมายเลขบัตรเครดิต"
else
  puts "ไม่รู้จักรูปแบบ"
end

# Output: อีเมลถูกต้อง

# ใช้ special variable $~ สำหรับ match data
phone = "+66-02-123-4567"
case phone
when /\+(\d{2})-(\d{2,3})-(\d{3})-(\d{4})/
  match = $~
  puts "Country code: #{match[1]}"  # => 66
  puts "Area code: #{match[2]}"     # => 02
end
```

---

## 14.6 Case เป็น Expression

```crystal
# case คืนค่า
score = 78

message = case score
when 90..100 then "ยอดเยี่ยม"
when 80..89  then "ดีมาก"
when 70..79  then "ดี"
when 60..69  then "พอใช้"
else              "ต้องปรับปรุง"
end

puts message  # => ดี

# ใช้ใน interpolation
puts "ผลการเรียน: #{case score
when 90.. then "A"
when 80..89 then "B"
when 70..79 then "C"
when 60..69 then "D"
else "F"
end}"

# กำหนดค่าด้วย case expression
discount = case customer_type = "premium"
when "premium" then 0.20
when "regular" then 0.10
else                0.05
end
puts "ส่วนลด: #{(discount * 100).to_i}%"  # => ส่วนลด: 20%
```

---

## 14.7 Case กับ Arrays

```crystal
def classify_array(arr : Array(Int32)) : String
  case arr
  when [] of Int32
    "ว่างเปล่า"
  when [_]  # มี 1 element
    "มีค่าเดียว: #{arr[0]}"
  when [_, _]  # มี 2 elements
    "มีสองค่า: #{arr[0]}, #{arr[1]}"
  else
    "หลายค่า (#{arr.size} ตัว)"
  end
end

puts classify_array([] of Int32)  # => ว่างเปล่า
puts classify_array([42])          # => มีค่าเดียว: 42
puts classify_array([1, 2])        # => มีสองค่า: 1, 2
puts classify_array([1, 2, 3])     # => หลายค่า (3 ตัว)
```

---

## 14.8 Pattern Matching (Crystal 1.x)

Crystal รองรับ pattern matching ที่ทรงพลัง

### Tuple Pattern

```crystal
point = {1, 2}

case point
when {0, 0}
  puts "จุด origin"
when {_, 0}
  puts "อยู่บน x-axis"
when {0, _}
  puts "อยู่บน y-axis"
when {x, y} if x == y
  puts "บน diagonal (#{x}, #{y})"
when {x, y}
  puts "จุด (#{x}, #{y})"
end

# Output: จุด (1, 2)
```

### Named Tuple Pattern

```crystal
config = {host: "localhost", port: 8080, ssl: false}

case config
when {host: "localhost", port: 80}
  puts "HTTP local"
when {host: "localhost", port: 443, ssl: true}
  puts "HTTPS local"
when {host: "localhost", port: port}
  puts "Local port: #{port}"
when {ssl: true}
  puts "SSL enabled"
else
  puts "Custom config: #{config}"
end

# Output: Local port: 8080
```

---

## 14.9 Deconstruct

Classes สามารถ implement `deconstruct` เพื่อใช้กับ pattern matching

```crystal
class Point
  getter x : Int32
  getter y : Int32

  def initialize(@x : Int32, @y : Int32)
  end

  # implement deconstruct สำหรับ array pattern
  def deconstruct : Array(Int32)
    [@x, @y]
  end

  # implement deconstruct_keys สำหรับ named pattern
  def deconstruct_keys(keys : Array(Symbol)?) : Hash(Symbol, Int32)
    {x: @x, y: @y}
  end
end

point = Point.new(3, 4)

# Array pattern
case point
in [0, 0]
  puts "Origin"
in [x, 0]
  puts "On x-axis at #{x}"
in [0, y]
  puts "On y-axis at #{y}"
in [x, y]
  puts "Point at (#{x}, #{y})"
end
# Output: Point at (3, 4)

# Named pattern
case point
in {x: 0, y: 0}
  puts "Origin"
in {x:, y: 0}  # x: captures to variable x
  puts "On x-axis at #{x}"
in {x:, y:}
  puts "Point at (#{x}, #{y})"
end
```

---

## 14.10 Guard Conditions ใน case

```crystal
data = {name: "Alice", age: 25, score: 85}

case data
when {age: age} if age < 18
  puts "ยังไม่บรรลุนิติภาวะ (#{age} ปี)"
when {score: score} if score >= 90
  puts "ได้คะแนนสูงมาก (#{score})"
when {name: name, score: score} if score >= 70
  puts "#{name} ผ่านด้วยคะแนน #{score}"
else
  puts "กรณีทั่วไป"
end

# Output: Alice ผ่านด้วยคะแนน 85
```

---

## 14.11 Case กับ Custom === Operator

Class สามารถ define `===` เพื่อใช้กับ case/when

```crystal
struct Range(T)
  # Crystal's built-in Range already has ===
  # แต่นี่คือตัวอย่างการ custom ===
end

# สร้าง custom matcher
struct DigitCount
  getter count : Int32

  def initialize(@count : Int32)
  end

  def ===(other : String) : Bool
    other.size == @count
  end
end

short  = DigitCount.new(3)
medium = DigitCount.new(6)
long   = DigitCount.new(10)

phone = "1234567890"
case phone
when short
  puts "รหัสสั้น (3 หลัก)"
when medium
  puts "รหัสปานกลาง (6 หลัก)"
when long
  puts "รหัสยาว (10 หลัก)"
end

# Output: รหัสยาว (10 หลัก)
```

---

## 14.12 ตัวอย่างจริง: HTTP Router

```crystal
struct Request
  getter method : String
  getter path : String

  def initialize(@method : String, @path : String)
  end
end

struct Response
  getter status : Int32
  getter body : String

  def initialize(@status : Int32, @body : String)
  end
end

def route(request : Request) : Response
  case {request.method, request.path}
  when {"GET", "/"}
    Response.new(200, "Welcome to Crystal API!")
  when {"GET", "/health"}
    Response.new(200, "OK")
  when {"GET", "/users"}
    Response.new(200, "[{\"id\": 1, \"name\": \"Alice\"}]")
  when {"POST", "/users"}
    Response.new(201, "User created")
  when {"GET", /\A\/users\/(\d+)\z/}
    match = $~
    user_id = match[1].to_i
    Response.new(200, "{\"id\": #{user_id}}")
  when {"DELETE", /\A\/users\/(\d+)\z/}
    Response.new(204, "")
  when {"GET", _}
    Response.new(404, "Not Found")
  else
    Response.new(405, "Method Not Allowed")
  end
end

# ทดสอบ
requests = [
  Request.new("GET", "/"),
  Request.new("GET", "/users"),
  Request.new("GET", "/users/42"),
  Request.new("POST", "/users"),
  Request.new("GET", "/unknown"),
  Request.new("PATCH", "/users"),
]

requests.each do |req|
  resp = route(req)
  puts "#{req.method} #{req.path} => #{resp.status}: #{resp.body}"
end
```

---

## 14.13 ตัวอย่างจริง: State Machine

```crystal
enum OrderStatus
  Pending
  Processing
  Shipped
  Delivered
  Cancelled
  Refunded
end

struct Order
  getter id : Int32
  getter status : OrderStatus
  getter amount : Float64

  def initialize(@id : Int32, @status : OrderStatus, @amount : Float64)
  end

  def with_status(new_status : OrderStatus) : Order
    Order.new(@id, new_status, @amount)
  end
end

def process_order_event(order : Order, event : String) : {Order, String}
  case {order.status, event}
  when {OrderStatus::Pending, "pay"}
    {order.with_status(OrderStatus::Processing), "ชำระเงินสำเร็จ กำลังประมวลผล"}
  when {OrderStatus::Processing, "ship"}
    {order.with_status(OrderStatus::Shipped), "จัดส่งสินค้าแล้ว"}
  when {OrderStatus::Shipped, "deliver"}
    {order.with_status(OrderStatus::Delivered), "ส่งสำเร็จ ลูกค้าได้รับสินค้า"}
  when {OrderStatus::Pending, "cancel"}, {OrderStatus::Processing, "cancel"}
    {order.with_status(OrderStatus::Cancelled), "ยกเลิก order แล้ว"}
  when {OrderStatus::Delivered, "refund"}
    {order.with_status(OrderStatus::Refunded), "คืนเงินสำเร็จ"}
  when {OrderStatus::Cancelled, _}, {OrderStatus::Refunded, _}
    {order, "ไม่สามารถดำเนินการได้ (#{order.status})"}
  else
    {order, "Event '#{event}' ไม่ถูกต้องสำหรับสถานะ #{order.status}"}
  end
end

# Simulation
order = Order.new(1001, OrderStatus::Pending, 500.0)
events = ["pay", "ship", "deliver"]

puts "Order ##{order.id} - เริ่มต้น: #{order.status}"
events.each do |event|
  order, message = process_order_event(order, event)
  puts "Event '#{event}': #{message} => #{order.status}"
end
```

---

## 14.14 ตัวอย่างจริง: Expression Evaluator

```crystal
abstract class Expr; end

class Num < Expr
  getter value : Float64
  def initialize(@value : Float64); end

  def deconstruct_keys(keys : Array(Symbol)?)
    {value: @value}
  end
end

class BinOp < Expr
  getter op : String
  getter left : Expr
  getter right : Expr

  def initialize(@op : String, @left : Expr, @right : Expr); end

  def deconstruct_keys(keys : Array(Symbol)?)
    {op: @op, left: @left, right: @right}
  end
end

class UnaryOp < Expr
  getter op : String
  getter operand : Expr

  def initialize(@op : String, @operand : Expr); end
end

def evaluate(expr : Expr) : Float64
  case expr
  when Num
    expr.value
  when BinOp
    left = evaluate(expr.left)
    right = evaluate(expr.right)
    case expr.op
    when "+" then left + right
    when "-" then left - right
    when "*" then left * right
    when "/" then right == 0.0 ? raise "หารด้วยศูนย์" : left / right
    when "**" then left ** right
    else raise "ไม่รู้จัก operator: #{expr.op}"
    end
  when UnaryOp
    operand = evaluate(expr.operand)
    case expr.op
    when "-" then -operand
    when "abs" then operand.abs
    when "sqrt" then Math.sqrt(operand)
    else raise "ไม่รู้จัก unary operator: #{expr.op}"
    end
  else
    raise "ไม่รู้จัก expression type"
  end
end

# (3 + 4) * 2
expr = BinOp.new("*",
  BinOp.new("+", Num.new(3.0), Num.new(4.0)),
  Num.new(2.0)
)
puts evaluate(expr)  # => 14.0

# sqrt(16) + abs(-3)
expr2 = BinOp.new("+",
  UnaryOp.new("sqrt", Num.new(16.0)),
  UnaryOp.new("abs", UnaryOp.new("-", Num.new(3.0)))
)
puts evaluate(expr2)  # => 7.0
```

---

## 14.15 ตัวอย่างจริง: Data Validation

```crystal
alias FieldValue = String | Int32 | Float64 | Bool | Nil

struct ValidationResult
  getter valid : Bool
  getter errors : Array(String)

  def initialize(@valid : Bool, @errors : Array(String))
  end

  def self.ok : self
    new(true, [] of String)
  end

  def self.fail(error : String) : self
    new(false, [error])
  end
end

def validate_field(name : String, value : FieldValue, rule : String) : ValidationResult
  case {rule, value}
  when {"required", nil}
    ValidationResult.fail("#{name} จำเป็นต้องมีค่า")
  when {"required", String}
    value.empty? ? ValidationResult.fail("#{name} ต้องไม่ว่างเปล่า") : ValidationResult.ok
  when {"positive", Int32}
    value > 0 ? ValidationResult.ok : ValidationResult.fail("#{name} ต้องเป็นบวก")
  when {"positive", Float64}
    value > 0.0 ? ValidationResult.ok : ValidationResult.fail("#{name} ต้องเป็นบวก")
  when {"email", String}
    value.includes?("@") ? ValidationResult.ok : ValidationResult.fail("#{name} รูปแบบอีเมลไม่ถูกต้อง")
  when {_, nil}
    ValidationResult.ok  # ค่า nil ผ่านทุก rule ยกเว้น required
  else
    ValidationResult.ok
  end
end

# ทดสอบ
tests = [
  {"name", nil, "required"},
  {"name", "", "required"},
  {"name", "Alice", "required"},
  {"age", -5, "positive"},
  {"age", 25, "positive"},
  {"email", "invalid", "email"},
  {"email", "user@example.com", "email"},
]

tests.each do |(field, value, rule)|
  result = validate_field(field, value, rule)
  status = result.valid ? "✓" : "✗"
  errors = result.errors.empty? ? "" : " - #{result.errors.join(", ")}"
  puts "#{status} #{field}=#{value.inspect} [#{rule}]#{errors}"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Calculator
```crystal
def calculate(a : Float64, op : String, b : Float64) : Float64?
  case op
  # TODO: implement +, -, *, /, %, **
  # คืน nil สำหรับ invalid operation
  end
end

puts calculate(10.0, "+", 3.0).inspect   # => 13.0
puts calculate(10.0, "/", 0.0).inspect   # => nil (หารด้วย 0)
puts calculate(10.0, "^", 2.0).inspect   # => nil (operator ไม่รู้จัก)
```

### แบบฝึกหัดที่ 2: Day Classifier
```crystal
def classify_day(month : Int32, day : Int32, year : Int32) : String
  case {month, day}
  # TODO:
  # - วันหยุดไทยสำคัญ (เช่น 1/1, 12/31, etc.)
  # - วันสำคัญอื่นๆ
  # - วันธรรมดา
  end
end
```

### แบบฝึกหัดที่ 3: Config Parser
```crystal
# รับ string config "key=value" และ validate ว่า type ถูกต้อง
def parse_config_value(key : String, raw_value : String) : String | Int32 | Bool | Nil
  case key
  # TODO: กำหนด type ตาม key:
  # - "port", "timeout", "max_connections" -> Int32
  # - "debug", "ssl", "verbose" -> Bool
  # - "host", "name", "path" -> String
  # - อื่นๆ -> String
  end
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ case/when:

1. **การ match ค่า**: เปรียบเทียบโดยตรง
2. **หลายค่าใน when เดียว**: `when "a", "b", "c"`
3. **Case ไม่มี expression**: ทำงานเหมือน if/elsif
4. **Range matching**: `when 1..10`
5. **Type matching**: `when Int32, String`
6. **Regex matching**: `when /pattern/`
7. **Case เป็น expression**: คืนค่าได้
8. **Pattern matching**: Tuple, NamedTuple patterns
9. **Deconstruct**: custom patterns สำหรับ objects
10. **Guard conditions**: `when pattern if condition`

### Case vs If เลือกใช้อะไร?

| สถานการณ์ | เลือกใช้ |
|-----------|---------|
| Match ค่าหลายแบบ | case/when |
| Match Type | case/when |
| Match Range | case/when |
| เงื่อนไขซับซ้อน (&&, ||) | if/elsif |
| Pattern matching | case/when/in |
| เงื่อนไขเดียว | if |

case/when ทำให้โค้ดอ่านง่ายและจัดการหลาย cases ได้อย่างมีประสิทธิภาพ!
