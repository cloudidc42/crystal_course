# Part 016: Exception Handling - Begin/Rescue/Ensure

## บทนำ

ในการเขียนโปรแกรมจริง สิ่งที่ผิดพลาดมักเกิดขึ้นเสมอ ไฟล์อาจไม่มีอยู่ การเชื่อมต่อเครือข่ายอาจขาด หรือผู้ใช้ป้อนข้อมูลผิดรูปแบบ Crystal มีระบบ **Exception Handling** ที่ช่วยให้เราจัดการกับข้อผิดพลาดเหล่านี้ได้อย่างเป็นระเบียบ

---

## โครงสร้างพื้นฐาน: begin/rescue/end

```crystal
begin
  # โค้ดที่อาจเกิด exception
  result = 10 / 0
rescue ex : DivisionByZeroError
  # จัดการ exception
  puts "เกิดข้อผิดพลาด: หารด้วยศูนย์ไม่ได้"
end
```

โครงสร้าง `begin...rescue...end` คือหัวใจของ exception handling ใน Crystal:
- **begin** - เริ่มต้นบล็อกที่อาจเกิด exception
- **rescue** - ดักจับ exception และจัดการ
- **end** - จบบล็อก

---

## การใช้ raise

`raise` ใช้สำหรับ "โยน" exception ออกมา:

```crystal
# raise ด้วย string message
raise "เกิดข้อผิดพลาดร้ายแรง"

# raise ด้วย exception object
raise RuntimeError.new("ข้อผิดพลาด runtime")

# raise ด้วย class (Crystal จะสร้าง instance ให้)
raise ArgumentError.new("argument ไม่ถูกต้อง")
```

### ตัวอย่างการใช้ raise ในฟังก์ชัน

```crystal
def divide(a : Int32, b : Int32) : Float64
  if b == 0
    raise ArgumentError.new("ตัวหาร (b) ต้องไม่เป็น 0")
  end
  a.to_f / b
end

begin
  puts divide(10, 2)   # => 5.0
  puts divide(10, 0)   # raises ArgumentError
rescue ex : ArgumentError
  puts "ArgumentError: #{ex.message}"
end
```

---

## rescue พร้อม Exception Object

เราสามารถดักจับ exception object เพื่อดูรายละเอียดได้:

```crystal
begin
  arr = [1, 2, 3]
  puts arr[10]  # IndexError
rescue ex
  puts "ประเภท exception: #{ex.class}"
  puts "ข้อความ: #{ex.message}"
  puts "Backtrace:"
  ex.backtrace.each { |line| puts "  #{line}" }
end
```

### ex.message และ ex.backtrace

```crystal
def risky_method
  raise RuntimeError.new("บางอย่างผิดพลาด")
end

def calling_method
  risky_method
end

begin
  calling_method
rescue ex : RuntimeError
  puts "Message: #{ex.message}"
  # backtrace แสดง call stack
  puts "\nBacktrace:"
  ex.backtrace.first(5).each_with_index do |line, i|
    puts "  #{i}: #{line}"
  end
end
```

---

## rescue หลายประเภท

สามารถดักจับ exception หลายประเภทในบล็อกเดียวหรือแยกบล็อกได้:

```crystal
def process_input(input : String)
  begin
    # อาจเกิด ArgumentError หรือ RuntimeError
    if input.empty?
      raise ArgumentError.new("input ว่างเปล่า")
    end
    
    number = input.to_i
    result = 100 / number
    puts "ผลลัพธ์: #{result}"
    
  rescue ex : ArgumentError
    puts "Argument ผิดพลาด: #{ex.message}"
    
  rescue ex : DivisionByZeroError
    puts "หารด้วยศูนย์ไม่ได้"
    
  rescue ex : Exception
    puts "Exception ทั่วไป: #{ex.message}"
  end
end

process_input("")      # ArgumentError
process_input("0")     # DivisionByZeroError
process_input("abc")   # Exception (to_i raises)
process_input("5")     # ทำงานปกติ => 20
```

### rescue หลายประเภทในบรรทัดเดียว

```crystal
begin
  # โค้ดที่อาจเกิด exception
  raise ArgumentError.new("ตัวอย่าง")
rescue ex : ArgumentError | TypeError | RuntimeError
  puts "เกิดหนึ่งในสาม exception เหล่านี้: #{ex.class}"
  puts "Message: #{ex.message}"
end
```

---

## ensure

`ensure` รันเสมอ ไม่ว่าจะเกิด exception หรือไม่:

```crystal
def read_file(filename : String)
  file = nil
  begin
    file = File.open(filename)
    content = file.read
    puts content
  rescue ex : File::NotFoundError
    puts "ไม่พบไฟล์: #{filename}"
  ensure
    # ปิดไฟล์เสมอ แม้เกิด exception
    file.close if file
    puts "ปิดไฟล์แล้ว"
  end
end
```

### ensure กับ return

```crystal
def important_function : String
  begin
    puts "เริ่มทำงาน"
    return "สำเร็จ"
  rescue ex
    puts "เกิด exception: #{ex.message}"
    return "ล้มเหลว"
  ensure
    # ensure รันเสมอ แม้จะ return แล้ว
    puts "cleanup รันเสมอ"
  end
end

result = important_function
puts "ผลลัพธ์: #{result}"
# Output:
# เริ่มทำงาน
# cleanup รันเสมอ
# ผลลัพธ์: สำเร็จ
```

---

## retry

`retry` ใช้เพื่อลองทำใหม่อีกครั้งหลังเกิด exception:

```crystal
attempts = 0
max_attempts = 3

begin
  attempts += 1
  puts "ครั้งที่ #{attempts}"
  
  # จำลองการเชื่อมต่อที่ล้มเหลวในครั้งแรก
  raise RuntimeError.new("การเชื่อมต่อล้มเหลว") if attempts < 3
  
  puts "เชื่อมต่อสำเร็จในครั้งที่ #{attempts}!"
  
rescue ex : RuntimeError
  if attempts < max_attempts
    puts "ลองอีกครั้ง... (#{ex.message})"
    retry
  else
    puts "เกินจำนวนครั้งที่กำหนด: #{ex.message}"
  end
end
```

### retry pattern แบบ real-world

```crystal
def connect_to_server(host : String, max_retries : Int32 = 3) : Bool
  retries = 0
  
  begin
    puts "กำลังเชื่อมต่อ #{host}..."
    
    # จำลอง: สุ่ม fail 70% ของครั้งแรกๆ
    if retries < 2
      raise RuntimeError.new("Connection timeout")
    end
    
    puts "เชื่อมต่อสำเร็จ!"
    true
    
  rescue ex : RuntimeError
    retries += 1
    if retries <= max_retries
      puts "ล้มเหลว (#{ex.message}), retry #{retries}/#{max_retries}"
      retry
    else
      puts "เชื่อมต่อไม่ได้หลังจาก #{max_retries} ครั้ง"
      false
    end
  end
end

connect_to_server("example.com")
```

---

## Custom Exceptions

การสร้าง exception ของตัวเองช่วยให้โค้ดอ่านง่ายและจัดการได้ตรงจุด:

```crystal
# Custom exception พื้นฐาน
class MyCustomError < Exception
  def initialize(message : String)
    super(message)
  end
end

# Custom exception พร้อม context data
class ValidationError < Exception
  getter field : String
  getter value : String
  
  def initialize(@field : String, @value : String, message : String)
    super("Validation failed for #{field}: #{message} (got: #{value})")
  end
end

# ใช้งาน
begin
  raise ValidationError.new("email", "not-an-email", "ต้องมีรูปแบบ email ที่ถูกต้อง")
rescue ex : ValidationError
  puts "Validation Error!"
  puts "Field: #{ex.field}"
  puts "Value: #{ex.value}"
  puts "Message: #{ex.message}"
end
```

### Hierarchy ของ Custom Exceptions

```crystal
# Base exception สำหรับ application
class AppError < Exception; end

# Sub-exceptions
class DatabaseError < AppError
  getter query : String
  
  def initialize(@query : String, message : String)
    super("Database error on query '#{query}': #{message}")
  end
end

class NetworkError < AppError
  getter url : String
  getter status_code : Int32
  
  def initialize(@url : String, @status_code : Int32)
    super("Network error #{status_code} for URL: #{url}")
  end
end

class AuthenticationError < AppError
  def initialize(username : String)
    super("Authentication failed for user: #{username}")
  end
end

# ใช้งาน
def perform_operation(type : String)
  case type
  when "db"
    raise DatabaseError.new("SELECT * FROM users", "connection refused")
  when "network"
    raise NetworkError.new("https://api.example.com", 503)
  when "auth"
    raise AuthenticationError.new("john_doe")
  end
end

["db", "network", "auth"].each do |type|
  begin
    perform_operation(type)
  rescue ex : DatabaseError
    puts "DB Error: #{ex.message}"
  rescue ex : NetworkError
    puts "Network Error (#{ex.status_code}): #{ex.url}"
  rescue ex : AppError
    puts "App Error: #{ex.message}"
  end
end
```

---

## Exception Hierarchy ใน Crystal

```
Exception
├── Error
│   ├── ArgumentError
│   ├── IndexError
│   ├── KeyError
│   ├── TypeError
│   ├── DivisionByZeroError
│   ├── NotImplementedError
│   ├── OverflowError
│   └── RuntimeError
└── (custom exceptions)
```

### ตัวอย่าง Exception ที่พบบ่อย

```crystal
# ArgumentError
begin
  raise ArgumentError.new("argument ไม่ถูกต้อง")
rescue ex : ArgumentError
  puts "ArgumentError: #{ex.message}"
end

# IndexError
begin
  arr = [1, 2, 3]
  _ = arr[10]
rescue ex : IndexError
  puts "IndexError: #{ex.message}"
end

# KeyError
begin
  hash = {"a" => 1, "b" => 2}
  _ = hash["z"]
rescue ex : KeyError
  puts "KeyError: #{ex.message}"
end

# TypeError
begin
  raise TypeError.new("ประเภทข้อมูลไม่ถูกต้อง")
rescue ex : TypeError
  puts "TypeError: #{ex.message}"
end

# DivisionByZeroError
begin
  _ = 10 / 0
rescue ex : DivisionByZeroError
  puts "DivisionByZeroError: #{ex.message}"
end
```

---

## rescue ใน Methods

แทนที่จะใช้ begin...rescue ใน method เราสามารถใช้ rescue โดยตรงได้:

```crystal
def safe_divide(a : Int32, b : Int32) : Float64?
  a.to_f / b
rescue DivisionByZeroError
  puts "ไม่สามารถหารด้วยศูนย์"
  nil
end

# หรือแบบ inline rescue
def parse_int(str : String) : Int32?
  str.to_i
rescue ArgumentError
  nil
end

# เรียกใช้
puts safe_divide(10, 2).inspect   # => 5.0
puts safe_divide(10, 0).inspect   # => nil (หลัง print error message)
puts parse_int("123").inspect     # => 123
puts parse_int("abc").inspect     # => nil
```

### Method ที่ใช้ rescue แบบ inline

```crystal
def load_config(path : String) : Hash(String, String)
  content = File.read(path)
  # parse content...
  {"key" => "value"}
rescue File::NotFoundError
  puts "ไม่พบไฟล์ config: #{path}"
  {} of String => String
rescue ex
  puts "เกิดข้อผิดพลาดในการโหลด config: #{ex.message}"
  {} of String => String
end
```

---

## โครงสร้าง begin/rescue/else/ensure

Crystal รองรับ `else` ด้วย ซึ่งรันเมื่อ **ไม่เกิด** exception:

```crystal
begin
  result = 10 / 2
rescue ex : DivisionByZeroError
  puts "เกิดข้อผิดพลาด"
else
  # รันเฉพาะเมื่อไม่เกิด exception
  puts "คำนวณสำเร็จ: #{result}"
ensure
  # รันเสมอ
  puts "เสร็จสิ้น"
end
# Output:
# คำนวณสำเร็จ: 5
# เสร็จสิ้น
```

---

## Best Practices

### 1. ดักจับ Exception ที่เฉพาะเจาะจง

```crystal
# ไม่ดี - ดักจับ Exception ทั้งหมด
begin
  risky_operation
rescue ex
  puts "เกิดข้อผิดพลาด"  # ไม่รู้ว่าเกิดอะไร
end

# ดี - ดักจับเฉพาะที่คาดหวัง
begin
  risky_operation
rescue ex : NetworkError
  handle_network_error(ex)
rescue ex : DatabaseError
  handle_db_error(ex)
end
```

### 2. อย่า Swallow Exceptions

```crystal
# ไม่ดี - ซ่อน exception
begin
  important_operation
rescue
  # ไม่ทำอะไรเลย - อันตราย!
end

# ดี - log อย่างน้อย
begin
  important_operation
rescue ex
  STDERR.puts "Warning: #{ex.class}: #{ex.message}"
  # หรือ log ไปยัง logging system
end
```

### 3. ใช้ ensure สำหรับ Cleanup

```crystal
def process_file(path : String)
  file = File.open(path)
  begin
    # ทำงานกับไฟล์
    process(file)
  ensure
    file.close  # ปิดไฟล์เสมอ
  end
end

# หรือใช้ block pattern (แนะนำมากกว่า)
def process_file_better(path : String)
  File.open(path) do |file|
    process(file)
  end  # Crystal ปิดไฟล์อัตโนมัติ
end
```

### 4. Custom Exception พร้อม Context

```crystal
# ดี - มี context ที่เป็นประโยชน์
class InsufficientFundsError < Exception
  getter account_id : String
  getter required : Float64
  getter available : Float64
  
  def initialize(@account_id : String, @required : Float64, @available : Float64)
    super("บัญชี #{account_id}: ต้องการ #{required} แต่มีเพียง #{available}")
  end
end

# ใช้งาน
begin
  # จำลองการถอนเงิน
  balance = 100.0
  amount = 500.0
  if amount > balance
    raise InsufficientFundsError.new("ACC001", amount, balance)
  end
rescue ex : InsufficientFundsError
  puts ex.message
  puts "บัญชี: #{ex.account_id}"
  puts "ขาด: #{ex.required - ex.available}"
end
```

### 5. Re-raise Exception

```crystal
def handle_with_logging(operation : String)
  begin
    yield
  rescue ex
    STDERR.puts "Error in #{operation}: #{ex.class}: #{ex.message}"
    raise  # re-raise ต้นฉบับ
  end
end

begin
  handle_with_logging("database query") do
    raise RuntimeError.new("query failed")
  end
rescue ex : RuntimeError
  puts "จัดการ RuntimeError: #{ex.message}"
end
```

---

## ตัวอย่างระบบจริง: Form Validation

```crystal
class FormValidationError < Exception
  getter errors : Array(String)
  
  def initialize(@errors : Array(String))
    super("Validation failed: #{errors.join(", ")}")
  end
end

class UserForm
  getter name : String
  getter email : String
  getter age : Int32
  
  def initialize(@name : String, @email : String, @age : Int32)
  end
  
  def validate!
    errors = [] of String
    
    errors << "ชื่อต้องมีความยาวอย่างน้อย 2 ตัวอักษร" if name.size < 2
    errors << "อีเมลต้องมี @" unless email.includes?("@")
    errors << "อายุต้องอยู่ระหว่าง 0 ถึง 150" unless (0..150).includes?(age)
    
    raise FormValidationError.new(errors) unless errors.empty?
    
    self
  end
end

def create_user(name : String, email : String, age : Int32)
  begin
    form = UserForm.new(name, email, age)
    form.validate!
    puts "สร้างผู้ใช้สำเร็จ: #{form.name} (#{form.email})"
  rescue ex : FormValidationError
    puts "Validation ล้มเหลว:"
    ex.errors.each { |error| puts "  - #{error}" }
  end
end

create_user("Alice", "alice@example.com", 30)  # สำเร็จ
create_user("A", "not-email", 200)              # validation ล้มเหลว
```

---

## ตัวอย่างระบบจริง: Database Connection (Simulation)

```crystal
class DatabaseConnectionError < Exception
  getter host : String
  getter port : Int32
  
  def initialize(@host : String, @port : Int32, message : String)
    super("Cannot connect to #{host}:#{port} - #{message}")
  end
end

class QueryError < Exception
  getter query : String
  
  def initialize(@query : String, message : String)
    super("Query failed: #{message}\nQuery: #{query}")
  end
end

class Database
  getter connected : Bool = false
  
  def connect(host : String, port : Int32)
    # จำลองการเชื่อมต่อ
    if host == "localhost" && port == 5432
      @connected = true
      puts "เชื่อมต่อฐานข้อมูลสำเร็จ"
    else
      raise DatabaseConnectionError.new(host, port, "host ไม่ถูกต้อง")
    end
  end
  
  def query(sql : String) : Array(String)
    raise RuntimeError.new("ยังไม่ได้เชื่อมต่อ") unless connected
    
    if sql.downcase.includes?("drop")
      raise QueryError.new(sql, "คำสั่ง DROP ไม่ได้รับอนุญาต")
    end
    
    ["row1", "row2", "row3"]  # จำลองผลลัพธ์
  end
  
  def disconnect
    @connected = false
    puts "ตัดการเชื่อมต่อแล้ว"
  end
end

db = Database.new

begin
  db.connect("localhost", 5432)
  
  results = db.query("SELECT * FROM users")
  puts "พบ #{results.size} แถว"
  
  # ทดสอบ QueryError
  begin
    db.query("DROP TABLE users")
  rescue ex : QueryError
    puts "Query Error: #{ex.message}"
  end
  
rescue ex : DatabaseConnectionError
  puts "Connection Error: #{ex.message}"
  puts "Host: #{ex.host}, Port: #{ex.port}"
ensure
  db.disconnect if db.connected
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom Exception Hierarchy

สร้างระบบ exception สำหรับ e-commerce:
- `ShopError` (base)
  - `OutOfStockError` (มี product_id และ requested_quantity)
  - `PaymentError` (มี amount และ reason)
  - `ShippingError` (มี address และ reason)

```crystal
# เติมโค้ดตรงนี้
class ShopError < Exception; end

class OutOfStockError < ShopError
  # TODO: เพิ่ม getter สำหรับ product_id และ requested_quantity
  # TODO: เพิ่ม initialize
end

class PaymentError < ShopError
  # TODO: เพิ่ม getter สำหรับ amount และ reason
  # TODO: เพิ่ม initialize
end

class ShippingError < ShopError
  # TODO: เพิ่ม getter สำหรับ address และ reason
  # TODO: เพิ่ม initialize
end

# ทดสอบ
def process_order(product_id : String, quantity : Int32, payment : Float64, address : String)
  # TODO: เพิ่มการตรวจสอบและ raise exception ต่างๆ
  puts "สั่งซื้อสำเร็จ"
end

begin
  process_order("P001", 100, 500.0, "กรุงเทพฯ")
rescue ex : OutOfStockError
  puts "สินค้าหมด: #{ex.message}"
rescue ex : PaymentError
  puts "การชำระเงินล้มเหลว: #{ex.message}"
rescue ex : ShippingError
  puts "การจัดส่งล้มเหลว: #{ex.message}"
rescue ex : ShopError
  puts "ข้อผิดพลาดทั่วไป: #{ex.message}"
end
```

### แบบฝึกหัดที่ 2: Retry Pattern

สร้างฟังก์ชัน `retry_operation` ที่รับ block และจำนวนครั้งสูงสุด:

```crystal
def retry_operation(max_attempts : Int32, &block : -> Void)
  # TODO: implement retry logic
  # - ลอง execute block
  # - ถ้าเกิด exception และยังไม่ครบ max_attempts ให้ retry
  # - ถ้าครบแล้วให้ re-raise exception
end

# ทดสอบ
attempt = 0
retry_operation(3) do
  attempt += 1
  raise RuntimeError.new("ล้มเหลว") if attempt < 3
  puts "สำเร็จในครั้งที่ #{attempt}!"
end
```

### แบบฝึกหัดที่ 3: Safe Calculator

สร้าง Calculator ที่จัดการ exception ทุกกรณี:

```crystal
class Calculator
  def calculate(expression : String) : Float64?
    # TODO: parse expression แบบ "10 + 5", "20 / 0", "abc * 2"
    # Handle: DivisionByZeroError, ArgumentError (parse ไม่ได้)
    # Return nil เมื่อเกิด error
  end
end

calc = Calculator.new
puts calc.calculate("10 + 5").inspect    # => 15.0
puts calc.calculate("20 / 4").inspect    # => 5.0
puts calc.calculate("10 / 0").inspect    # => nil
puts calc.calculate("abc + 1").inspect   # => nil
```

---

## สรุป

| หัวข้อ | สิ่งที่ต้องจำ |
|--------|--------------|
| `begin/rescue/end` | โครงสร้างพื้นฐานของ exception handling |
| `rescue ex : TypeName` | ดักจับ exception เฉพาะประเภท |
| `ex.message` | ข้อความของ exception |
| `ex.backtrace` | call stack ที่เกิด exception |
| `ensure` | รันเสมอ ใช้สำหรับ cleanup |
| `retry` | ลองทำใหม่อีกครั้ง |
| `raise` | โยน exception |
| Custom Exception | สืบทอดจาก `Exception` หรือ subclass |
| rescue in methods | ใช้ rescue โดยตรงใน method โดยไม่ต้องมี begin |

### หลักการสำคัญ

1. **ดักจับให้เฉพาะเจาะจง** - ระบุประเภท exception ให้ชัดเจน
2. **ไม่ Swallow Exceptions** - อย่าดักจับแล้วไม่ทำอะไร
3. **ใช้ ensure สำหรับ cleanup** - ปิดไฟล์, ตัดการเชื่อมต่อ
4. **สร้าง Custom Exception ที่มี context** - ใส่ข้อมูลที่เป็นประโยชน์
5. **Re-raise เมื่อจำเป็น** - ถ้าจัดการไม่ได้ ให้ส่งต่อขึ้นไป
