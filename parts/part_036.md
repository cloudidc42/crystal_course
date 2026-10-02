# Part 36: Super

## บทนำ

`super` เป็น keyword ใน Crystal ที่ใช้เรียก method ที่มีชื่อเดียวกันใน parent class เป็นเครื่องมือสำคัญในการ extend พฤติกรรมของ parent โดยไม่ต้องเขียนโค้ดซ้ำ

---

## 1. super พื้นฐาน

### 1.1 เรียก Parent Method ด้วย super

```crystal
class Animal
  def speak
    puts "สัตว์ส่งเสียง..."
  end

  def describe
    puts "ฉันคือสัตว์"
  end
end

class Dog < Animal
  def speak
    super          # เรียก Animal#speak ก่อน
    puts "โฮ่ง!"   # แล้วทำสิ่งเพิ่มเติม
  end

  def describe
    super          # เรียก Animal#describe ก่อน
    puts "โดยเฉพาะคือสุนัข"
  end
end

dog = Dog.new
dog.speak
# => สัตว์ส่งเสียง...
# => โฮ่ง!

dog.describe
# => ฉันคือสัตว์
# => โดยเฉพาะคือสุนัข
```

### 1.2 super กับ Return Value

```crystal
class Tax
  def calculate(amount : Float64) : Float64
    amount * 0.07  # 7% VAT
  end
end

class TaxWithSurcharge < Tax
  def calculate(amount : Float64) : Float64
    base_tax = super(amount)  # ได้ tax จาก parent
    surcharge = amount * 0.01  # เพิ่ม 1% surcharge
    base_tax + surcharge
  end
end

t1 = Tax.new
t2 = TaxWithSurcharge.new

puts t1.calculate(1000.0)  # => 70.0
puts t2.calculate(1000.0)  # => 80.0 (70 + 10)
```

---

## 2. super กับ Arguments

### 2.1 super ส่ง Arguments เดิม

```crystal
class Greeter
  def greet(name : String, language : String)
    puts "Using #{language}: Hello, #{name}!"
  end
end

class FormalGreeter < Greeter
  def greet(name : String, language : String)
    puts "=== Formal Greeting ==="
    super   # ส่ง arguments เดิมทั้งหมดไปให้ parent โดยอัตโนมัติ
    puts "=== End ==="
  end
end

fg = FormalGreeter.new
fg.greet("สมชาย", "Thai")
# => === Formal Greeting ===
# => Using Thai: Hello, สมชาย!
# => === End ===
```

### 2.2 super กับ Arguments ที่เปลี่ยนแปลง

```crystal
class Logger
  def log(message : String, level : String = "INFO")
    puts "[#{level}] #{message}"
  end
end

class TimestampLogger < Logger
  def log(message : String, level : String = "INFO")
    # เปลี่ยน message โดยเพิ่ม timestamp
    timestamped = "[#{Time.local.to_s("%H:%M:%S")}] #{message}"
    super(timestamped, level)  # ส่ง arguments ใหม่
  end
end

class PrefixLogger < TimestampLogger
  def initialize(@prefix : String)
  end

  def log(message : String, level : String = "INFO")
    # เพิ่ม prefix
    super("#{@prefix} #{message}", level)
  end
end

base = Logger.new
ts = TimestampLogger.new
prefix = PrefixLogger.new("[APP]")

base.log("Hello")
ts.log("Hello")
prefix.log("Hello")
prefix.log("Error!", "ERROR")
```

### 2.3 super กับ Arguments ต่างจาก Original

```crystal
class Validator
  def validate(value : String) : Array(String)
    errors = [] of String
    errors << "ต้องไม่ว่าง" if value.empty?
    errors
  end
end

class StrictValidator < Validator
  def validate(value : String) : Array(String)
    errors = super(value)  # ได้ errors จาก parent ก่อน
    # เพิ่ม validation เพิ่มเติม
    errors << "ต้องมีความยาวอย่างน้อย 8 ตัว" if value.size < 8
    errors << "ต้องมีตัวเลข" unless value.chars.any?(&.number?)
    errors
  end
end

class PasswordValidator < StrictValidator
  def validate(value : String) : Array(String)
    errors = super(value)
    errors << "ต้องมีตัวพิมพ์ใหญ่" unless value.chars.any?(&.uppercase?)
    errors << "ต้องมีอักขระพิเศษ" unless value.matches?(/[!@#$%^&*]/)
    errors
  end
end

v = PasswordValidator.new
puts v.validate("").inspect
# => ["ต้องไม่ว่าง", "ต้องมีความยาวอย่างน้อย 8 ตัว", ...]

puts v.validate("Password1!").inspect
# => []
```

---

## 3. super ใน initialize

### 3.1 super ใน Constructor

```crystal
class Vehicle
  getter make : String
  getter model : String
  getter year : Int32

  def initialize(@make : String, @model : String, @year : Int32)
    @mileage = 0.0
    puts "สร้าง Vehicle: #{@make} #{@model} #{@year}"
  end
end

class ElectricVehicle < Vehicle
  getter battery_capacity : Float64
  getter range_km : Int32

  def initialize(make : String, model : String, year : Int32,
                 @battery_capacity : Float64, @range_km : Int32)
    super(make, model, year)  # เรียก Vehicle#initialize
    @charge_level = 100.0
    puts "รถไฟฟ้า: แบต #{@battery_capacity}kWh วิ่งได้ #{@range_km}km"
  end
end

class SelfDrivingEV < ElectricVehicle
  getter autopilot_version : String

  def initialize(make : String, model : String, year : Int32,
                 battery : Float64, range : Int32, @autopilot_version : String)
    super(make, model, year, battery, range)  # เรียก ElectricVehicle#initialize
    puts "ขับเคลื่อนอัตโนมัติ v#{@autopilot_version}"
  end
end

# Chain ของ initializer calls
car = SelfDrivingEV.new("Tesla", "Model S", 2024, 100.0, 500, "4.0")
```

### 3.2 super ใน initialize พร้อม Default Values

```crystal
class BaseUser
  getter username : String
  getter email : String

  def initialize(@username : String, @email : String)
    @created_at = Time.local
  end
end

class AdminUser < BaseUser
  getter permissions : Array(String)

  def initialize(username : String, email : String,
                 permissions : Array(String) = ["read", "write", "admin"])
    super(username, email)
    @permissions = permissions
  end

  def can?(action : String) : Bool
    @permissions.includes?(action)
  end
end

class SuperAdmin < AdminUser
  def initialize(username : String, email : String)
    all_permissions = ["read", "write", "admin", "delete", "super"]
    super(username, email, all_permissions)
  end
end

admin = AdminUser.new("admin1", "admin@example.com")
super_admin = SuperAdmin.new("superadmin", "super@example.com")

puts admin.can?("delete")     # => false
puts super_admin.can?("super")  # => true
```

---

## 4. super กับ Blocks

### 4.1 super ส่ง Block ต่อ

```crystal
class Iterator
  def each
    yield 1
    yield 2
    yield 3
  end
end

class FilteredIterator < Iterator
  def initialize(@min : Int32)
  end

  def each
    # เรียก parent's each แล้ว filter
    super do |value|
      yield value if value >= @min
    end
  end
end

class TransformedIterator < FilteredIterator
  def initialize(min : Int32, @multiplier : Int32)
    super(min)
  end

  def each
    super do |value|
      yield value * @multiplier
    end
  end
end

fi = FilteredIterator.new(2)
fi.each { |v| puts v }
# => 2
# => 3

ti = TransformedIterator.new(2, 10)
ti.each { |v| puts v }
# => 20
# => 30
```

### 4.2 super กับ yield

```crystal
class Processor
  def process(items : Array(Int32))
    items.map { |item| yield item }
  end
end

class LoggedProcessor < Processor
  def process(items : Array(Int32))
    puts "เริ่มประมวลผล #{items.size} รายการ"
    result = super(items) { |item| yield item }
    puts "ประมวลผลเสร็จ: #{result.inspect}"
    result
  end
end

lp = LoggedProcessor.new
lp.process([1, 2, 3, 4, 5]) { |x| x * 2 }
```

---

## 5. super ใน Chain Inheritance

### 5.1 Multi-level super Chain

```crystal
class Base
  def greet(name : String) : String
    "สวัสดี #{name}"
  end
end

class Middle < Base
  def greet(name : String) : String
    base_greeting = super(name)  # เรียก Base#greet
    "#{base_greeting}, ยินดีต้อนรับ"
  end
end

class Top < Middle
  def greet(name : String) : String
    middle_greeting = super(name)  # เรียก Middle#greet (ซึ่งเรียก Base#greet)
    "#{middle_greeting}! 🎉"
  end
end

puts Base.new.greet("สมชาย")
# => สวัสดี สมชาย

puts Middle.new.greet("สมชาย")
# => สวัสดี สมชาย, ยินดีต้อนรับ

puts Top.new.greet("สมชาย")
# => สวัสดี สมชาย, ยินดีต้อนรับ! 🎉
```

### 5.2 Template Method Pattern ด้วย super

```crystal
class Report
  def generate : String
    header = build_header
    body = build_body
    footer = build_footer
    "#{header}\n#{body}\n#{footer}"
  end

  protected def build_header : String
    "=== รายงาน ==="
  end

  protected def build_body : String
    "(เนื้อหาว่าง)"
  end

  protected def build_footer : String
    "=== สิ้นสุดรายงาน ==="
  end
end

class SalesReport < Report
  def initialize(@data : Array({product: String, amount: Float64}))
  end

  protected def build_header : String
    "#{super}\nรายงานยอดขาย - #{Time.local.to_s("%Y-%m-%d")}"
  end

  protected def build_body : String
    lines = @data.map { |item| "  #{item[:product]}: ฿#{item[:amount]}" }
    total = @data.sum { |item| item[:amount] }
    lines.join("\n") + "\n  รวม: ฿#{total}"
  end

  protected def build_footer : String
    "#{super}\nสร้างโดย: ระบบอัตโนมัติ"
  end
end

data = [
  {product: "แอปเปิล", amount: 1500.0},
  {product: "กล้วย", amount: 800.0},
  {product: "ส้ม", amount: 1200.0},
]

report = SalesReport.new(data)
puts report.generate
```

---

## 6. ตัวอย่างขั้นสูง

### 6.1 Middleware Stack

```crystal
class Middleware
  def call(request : String) : String
    puts "Middleware base: processing '#{request}'"
    request
  end
end

class AuthMiddleware < Middleware
  def call(request : String) : String
    puts "AuthMiddleware: checking authentication"
    if request.starts_with?("authenticated:")
      result = super(request.sub("authenticated:", "").strip)
      puts "AuthMiddleware: auth passed"
      result
    else
      "401 Unauthorized"
    end
  end
end

class LoggingMiddleware < AuthMiddleware
  def call(request : String) : String
    puts "[LOG] Incoming: #{request}"
    result = super(request)
    puts "[LOG] Result: #{result}"
    result
  end
end

class RateLimitMiddleware < LoggingMiddleware
  def initialize(@max_per_minute : Int32)
    @count = 0
  end

  def call(request : String) : String
    @count += 1
    if @count > @max_per_minute
      "429 Too Many Requests"
    else
      super(request)
    end
  end
end

stack = RateLimitMiddleware.new(100)
puts stack.call("authenticated: GET /users")
puts "---"
puts stack.call("GET /public")
```

### 6.2 Event System กับ super

```crystal
class BaseEventHandler
  def handle(event : String, data : Hash(String, String)) : Bool
    puts "[BaseHandler] Processing: #{event}"
    true  # return true ถ้าจัดการสำเร็จ
  end
end

class LoggingHandler < BaseEventHandler
  def handle(event : String, data : Hash(String, String)) : Bool
    puts "[LOG] Event: #{event}, Data: #{data}"
    super(event, data)
  end
end

class ValidationHandler < LoggingHandler
  def handle(event : String, data : Hash(String, String)) : Bool
    unless data.has_key?("user_id")
      puts "[VALIDATION] Missing user_id!"
      return false
    end
    super(event, data)
  end
end

class BusinessHandler < ValidationHandler
  def handle(event : String, data : Hash(String, String)) : Bool
    return false unless super(event, data)  # เรียก chain ก่อน

    case event
    when "purchase"
      puts "[BUSINESS] Processing purchase for user #{data["user_id"]}"
    when "refund"
      puts "[BUSINESS] Processing refund for user #{data["user_id"]}"
    end
    true
  end
end

handler = BusinessHandler.new

puts "=== Valid Request ==="
handler.handle("purchase", {"user_id" => "123", "amount" => "500"})

puts "\n=== Invalid Request ==="
handler.handle("purchase", {"amount" => "500"})
```

### 6.3 Decorator Pattern ด้วย super

```crystal
class Coffee
  def cost : Float64
    25.0
  end

  def description : String
    "กาแฟดำ"
  end
end

class MilkCoffee < Coffee
  def cost : Float64
    super + 10.0  # เพิ่มราคานม
  end

  def description : String
    "#{super} + นม"
  end
end

class SugarCoffee < MilkCoffee
  def cost : Float64
    super + 5.0  # เพิ่มราคาน้ำตาล
  end

  def description : String
    "#{super} + น้ำตาล"
  end
end

class VanillaCoffee < SugarCoffee
  def cost : Float64
    super + 15.0
  end

  def description : String
    "#{super} + วานิลลา"
  end
end

# Build up cost and description through chain
drinks = [Coffee.new, MilkCoffee.new, SugarCoffee.new, VanillaCoffee.new]

drinks.each do |d|
  puts "#{d.description}: ฿#{d.cost}"
end
# => กาแฟดำ: ฿25.0
# => กาแฟดำ + นม: ฿35.0
# => กาแฟดำ + นม + น้ำตาล: ฿40.0
# => กาแฟดำ + นม + น้ำตาล + วานิลลา: ฿55.0
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Permission System

```crystal
class Role
  getter permissions : Array(String)

  def initialize
    @permissions = base_permissions
  end

  def base_permissions : Array(String)
    ["read"]
  end

  def can?(action : String) : Bool
    @permissions.includes?(action)
  end
end

class EditorRole < Role
  def base_permissions : Array(String)
    super + ["write", "edit"]
  end
end

class AdminRole < EditorRole
  def base_permissions : Array(String)
    super + ["delete", "manage_users"]
  end
end

class SuperAdminRole < AdminRole
  def base_permissions : Array(String)
    super + ["system_config", "backup", "restore"]
  end
end

roles = [Role.new, EditorRole.new, AdminRole.new, SuperAdminRole.new]

actions = ["read", "write", "delete", "system_config"]
roles.each_with_index do |role, i|
  puts "\n#{role.class.name}:"
  actions.each do |action|
    status = role.can?(action) ? "✓" : "✗"
    puts "  #{status} #{action}"
  end
end
```

### แบบฝึกหัดที่ 2: Price Calculator Chain

```crystal
class BasePrice
  def calculate(quantity : Int32, unit_price : Float64) : Float64
    quantity * unit_price
  end
end

class TaxedPrice < BasePrice
  def initialize(@tax_rate : Float64 = 0.07)
  end

  def calculate(quantity : Int32, unit_price : Float64) : Float64
    subtotal = super(quantity, unit_price)
    subtotal * (1 + @tax_rate)
  end
end

class DiscountedPrice < TaxedPrice
  def initialize(@discount_percent : Float64, tax_rate : Float64 = 0.07)
    super(tax_rate)
  end

  def calculate(quantity : Int32, unit_price : Float64) : Float64
    # Apply discount first, then tax
    discounted_price = unit_price * (1 - @discount_percent / 100)
    super(quantity, discounted_price)
  end
end

base = BasePrice.new
taxed = TaxedPrice.new
discounted = DiscountedPrice.new(10.0)  # 10% ส่วนลด + 7% VAT

price = 100.0
qty = 3

puts "ราคาพื้นฐาน: ฿#{base.calculate(qty, price)}"
puts "หลังภาษี: ฿#{taxed.calculate(qty, price)}"
puts "หลังส่วนลดและภาษี: ฿#{discounted.calculate(qty, price).round(2)}"
```

---

## สรุป

super ใน Crystal:

| รูปแบบ | syntax | หมายถึง |
|-------|--------|---------|
| super เรียบง่าย | `super` | ส่ง arguments เดิมทั้งหมด |
| super กับ arguments | `super(a, b)` | ส่ง arguments ที่กำหนด |
| super ใน initialize | `super(args)` | เรียก parent constructor |
| super กับ block | `super { \|x\| ... }` | ส่ง block ต่อ |

**Best Practices:**
- ใช้ `super` เพื่อ extend พฤติกรรม ไม่ใช่ replace ทั้งหมด
- เรียก `super` ในตำแหน่งที่เหมาะสม (ก่อน/หลัง/ระหว่าง logic ของตัวเอง)
- ใน `initialize` ควรเรียก `super` ก่อนใช้ค่า จาก parent
- `super` เรียบง่าย (ไม่มี args) จะส่ง arguments ปัจจุบันทั้งหมด
