# Part 024: Default Parameters

## บทนำ

**Default Parameters** ช่วยให้ method มีความยืดหยุ่น โดยกำหนดค่า default สำหรับ parameter ที่ caller ไม่จำเป็นต้องระบุ ทำให้ method ใช้งานได้ง่ายในกรณีทั่วไป แต่ยังปรับแต่งได้เมื่อต้องการ

---

## 1. Default Values พื้นฐาน

```crystal
# parameter ที่มี default value
def greet(name : String, greeting : String = "Hello") : String
  "#{greeting}, #{name}!"
end

puts greet("Alice")           # => Hello, Alice!
puts greet("Bob", "Hi")       # => Hi, Bob!
puts greet("Charlie", "Yo")   # => Yo, Charlie!

# method พร้อม default หลายตัว
def create_button(
  label : String,
  color : String = "blue",
  size : Int32 = 12,
  bold : Bool = false
) : String
  style = "color: #{color}; font-size: #{size}px; #{bold ? "font-weight: bold;" : ""}"
  "<button style=\"#{style}\">#{label}</button>"
end

puts create_button("Click")
puts create_button("Submit", "green")
puts create_button("Cancel", "red", 14, true)
```

---

## 2. หลักการวาง Default Parameters

Default parameters ต้องอยู่หลัง required parameters:

```crystal
# ถูกต้อง: required ก่อน, default ทีหลัง
def connect(host : String, port : Int32 = 80, ssl : Bool = false)
  puts "connecting to #{host}:#{port} (ssl: #{ssl})"
end

connect("example.com")             # port=80, ssl=false
connect("example.com", 443)        # port=443, ssl=false
connect("example.com", 443, true)  # port=443, ssl=true
```

### Default ใน Middle (ต้องระวัง)

```crystal
# ถ้า default อยู่กลาง ต้องใช้ keyword argument
def format_number(
  value : Float64,
  decimals : Int32 = 2,
  prefix : String = "",
  suffix : String = ""
) : String
  formatted = "%.#{decimals}f" % value
  "#{prefix}#{formatted}#{suffix}"
end

puts format_number(1234.567)              # => "1234.57"
puts format_number(1234.567, 3)           # => "1234.567"
puts format_number(1234.567, 2, "$")      # => "$1234.57"
puts format_number(1234.567, 2, "", " บาท") # => "1234.57 บาท"

# ข้ามตัวกลางไม่ได้แบบ positional
# format_number(1234.567, prefix: "$") => ใช้ keyword แทน
puts format_number(1234.567, prefix: "$")  # => "$1234.57"
```

---

## 3. Complex Default Expressions

Default value ไม่จำเป็นต้องเป็น literal อาจเป็น expression:

```crystal
# Default เป็น method call
def log(message : String, timestamp : Time = Time.local) : String
  "[#{timestamp}] #{message}"
end

# Default เป็น computation
def create_array(size : Int32 = 10, fill : Int32 = 0) : Array(Int32)
  Array.new(size, fill)
end

puts create_array.inspect         # => [0, 0, 0, 0, 0, 0, 0, 0, 0, 0]
puts create_array(5, 1).inspect   # => [1, 1, 1, 1, 1]

# Default อิงกับ parameter ก่อนหน้า
def range_array(start : Int32 = 0, stop : Int32 = 10, step : Int32 = 1) : Array(Int32)
  result = [] of Int32
  current = start
  while current < stop
    result << current
    current += step
  end
  result
end

puts range_array.inspect           # => [0, 1, 2, ..., 9]
puts range_array(5).inspect        # => [5, 6, 7, 8, 9]
puts range_array(0, 20, 5).inspect # => [0, 5, 10, 15]
```

### Default เป็น nil พร้อม Logic

```crystal
def send_email(
  to : String,
  subject : String,
  body : String,
  cc : String? = nil,
  bcc : String? = nil
) : String
  parts = ["To: #{to}", "Subject: #{subject}"]
  parts << "CC: #{cc}" if cc
  parts << "BCC: #{bcc}" if bcc
  parts << "\n#{body}"
  parts.join("\n")
end

puts send_email("alice@example.com", "Hello", "Hi there!")
puts "\n---\n"
puts send_email("alice@example.com", "Meeting", "See you at 3pm",
  cc: "bob@example.com",
  bcc: "manager@example.com"
)
```

---

## 4. เรียก Method พร้อมและไม่พร้อม Defaults

```crystal
def http_get(
  url : String,
  timeout : Int32 = 30,
  follow_redirects : Bool = true,
  max_redirects : Int32 = 5,
  verify_ssl : Bool = true
) : String
  "GET #{url} (timeout: #{timeout}s, redirects: #{follow_redirects ? max_redirects : 0}, ssl: #{verify_ssl})"
end

# เรียกแบบต่างๆ
puts http_get("https://example.com")
puts http_get("https://example.com", 60)
puts http_get("https://example.com", follow_redirects: false)
puts http_get("https://example.com", 60, false)
puts http_get("https://example.com",
  timeout: 120,
  verify_ssl: false
)
```

---

## 5. Mutable Default Values - ข้อควรระวัง

ใน Crystal default values ถูก evaluate ทุกครั้งที่เรียก method (ต่างจาก Python):

```crystal
# Crystal - safe! default evaluate ใหม่ทุกครั้ง
def append_to(item : String, list : Array(String) = [] of String) : Array(String)
  list << item
  list
end

result1 = append_to("apple")
result2 = append_to("banana")
puts result1.inspect  # => ["apple"]   - ไม่มี side effect!
puts result2.inspect  # => ["banana"]  - safe!

# เปรียบเทียบกับ Python ที่มีปัญหา:
# def append_to(item, lst=[]):  # Python - อันตราย! list เดิมถูกใช้ซ้ำ
#     lst.append(item)
#     return lst
```

### Shared State ด้วยความตั้งใจ

```crystal
# ถ้าต้องการ shared state ต้องทำชัดเจน
shared_list = [] of String

def append_to_shared(item : String, list : Array(String)) : Array(String)
  list << item
  list
end

append_to_shared("first", shared_list)
append_to_shared("second", shared_list)
puts shared_list.inspect  # => ["first", "second"]
```

---

## 6. Nil Defaults vs Optional Parameters

```crystal
# ความแตกต่างระหว่าง nil default กับ optional parameter

# 1. Nil default: caller ต้องรู้ว่าอาจเป็น nil
def process_with_nil_default(
  data : String,
  transform : (String -> String)? = nil
) : String
  transform ? transform.call(data) : data
end

puts process_with_nil_default("hello")
puts process_with_nil_default("hello", ->(s : String) { s.upcase })

# 2. Overloading: method signature ชัดเจนกว่า
def process(data : String) : String
  data
end

def process(data : String, transform : String -> String) : String
  transform.call(data)
end

puts process("hello")
puts process("hello") { |s| s.upcase }
```

### Default สำหรับ Optional Behavior

```crystal
# Pattern ที่ดีสำหรับ optional behavior
def find_users(
  filter : String? = nil,
  sort_by : String = "name",
  limit : Int32 = 50,
  offset : Int32 = 0
) : Array(String)
  # จำลอง
  all_users = ["Alice", "Bob", "Charlie", "Diana", "Eve"]
  
  result = if filter
    all_users.select { |u| u.downcase.includes?(filter.downcase) }
  else
    all_users
  end
  
  result.sort.drop(offset).first(limit)
end

puts find_users.inspect                    # ทั้งหมด
puts find_users(filter: "a").inspect       # กรอง "a"
puts find_users(limit: 2).inspect          # 2 รายการแรก
puts find_users(limit: 2, offset: 2).inspect # skip 2
```

---

## 7. Default Parameters กับ Class Methods

```crystal
class DatabaseConnection
  getter host : String
  getter port : Int32
  getter database : String
  getter pool_size : Int32
  
  def initialize(
    @host : String = "localhost",
    @port : Int32 = 5432,
    @database : String = "myapp",
    @pool_size : Int32 = 10
  )
  end
  
  def connection_string : String
    "postgresql://#{host}:#{port}/#{database}?pool_size=#{pool_size}"
  end
end

# Development
dev_db = DatabaseConnection.new
puts dev_db.connection_string
# => postgresql://localhost:5432/myapp?pool_size=10

# Production
prod_db = DatabaseConnection.new(
  host: "db.prod.example.com",
  port: 5432,
  database: "myapp_production",
  pool_size: 50
)
puts prod_db.connection_string
```

---

## 8. Default Parameters กับ Inheritance

```crystal
class Animal
  def speak(message : String = "...", volume : Int32 = 5) : String
    "#{"!" * volume} #{message}"
  end
end

class Dog < Animal
  def speak(message : String = "Woof", volume : Int32 = 7) : String
    super(message, volume)
  end
end

class Cat < Animal
  def speak(message : String = "Meow", volume : Int32 = 3) : String
    super(message, volume)
  end
end

animal = Animal.new
dog = Dog.new
cat = Cat.new

puts animal.speak           # => !!!!! ...
puts dog.speak              # => !!!!!!! Woof
puts cat.speak              # => !!! Meow
puts dog.speak("Bark!", 10) # => !!!!!!!!!!! Bark!
```

---

## 9. Best Practices

### Principle of Least Surprise

```crystal
# ดี: default ที่เป็นธรรมชาติ
def log(message : String, level : String = "INFO") : Nil
  puts "[#{level}] #{message}"
end

# ดี: default ที่ทำงานได้ทันที
def retry_operation(max_attempts : Int32 = 3, delay_seconds : Float64 = 1.0)
  # ...
end

# ควรระวัง: default ที่อาจเซอร์ไพรส์
# ไม่ดี - default เปลี่ยนได้ตาม environment
def connect(host : String = ENV["DB_HOST"]? || "localhost")
  # default ขึ้นกับ ENV variable - อาจ confuse
end
```

### Documentation ผ่าน Default Values

```crystal
# Default values ช่วย document behavior ที่คาดหวัง
def create_config(
  env : String = "development",
  debug : Bool = true,
  log_level : String = "debug",
  max_connections : Int32 = 5,
  timeout_seconds : Int32 = 30
) : Hash(String, String | Bool | Int32)
  {
    "environment" => env,
    "debug" => debug,
    "log_level" => log_level,
    "max_connections" => max_connections,
    "timeout" => timeout_seconds,
  }
end

# Development config (ใช้ defaults)
dev_config = create_config

# Production config
prod_config = create_config(
  env: "production",
  debug: false,
  log_level: "warn",
  max_connections: 100,
  timeout_seconds: 60
)
```

---

## 10. ตัวอย่าง Real-World

### API Client

```crystal
class ApiClient
  def initialize(
    @base_url : String,
    @timeout : Int32 = 30,
    @retry_count : Int32 = 3,
    @verify_ssl : Bool = true,
    @user_agent : String = "CrystalApiClient/1.0"
  )
  end
  
  def get(
    path : String,
    params : Hash(String, String) = {} of String => String
  ) : String
    query = params.map { |k, v| "#{k}=#{v}" }.join("&")
    url = query.empty? ? "#{@base_url}#{path}" : "#{@base_url}#{path}?#{query}"
    
    # จำลอง request
    "GET #{url} (timeout: #{@timeout}s, retries: #{@retry_count}, ssl: #{@verify_ssl})"
  end
  
  def post(
    path : String,
    body : String = "",
    content_type : String = "application/json"
  ) : String
    "POST #{@base_url}#{path} (Content-Type: #{content_type}, body_size: #{body.size})"
  end
end

client = ApiClient.new("https://api.example.com")
puts client.get("/users")
puts client.get("/users", {"page" => "1", "limit" => "10"})
puts client.post("/users", "{\"name\": \"Alice\"}")

# Production client
prod_client = ApiClient.new(
  "https://api.production.com",
  timeout: 60,
  retry_count: 5,
  user_agent: "MyApp/2.0"
)
```

### Text Formatter

```crystal
def format_table(
  rows : Array(Array(String)),
  header : Array(String) = [] of String,
  separator : String = " | ",
  align : Symbol = :left,
  max_width : Int32? = nil
) : String
  all_rows = header.empty? ? rows : [header] + rows
  
  # หาความกว้างของแต่ละ column
  col_count = all_rows.map(&.size).max || 0
  col_widths = Array.new(col_count, 0)
  
  all_rows.each do |row|
    row.each_with_index do |cell, i|
      width = max_width ? [cell.size, max_width].min : cell.size
      col_widths[i] = [col_widths[i], width].max
    end
  end
  
  lines = [] of String
  
  all_rows.each_with_index do |row, idx|
    cells = row.each_with_index.map do |cell, i|
      text = max_width && cell.size > max_width ? cell[0, max_width - 2] + ".." : cell
      case align
      when :right
        text.rjust(col_widths[i])
      when :center
        text.center(col_widths[i])
      else
        text.ljust(col_widths[i])
      end
    end
    lines << cells.join(separator)
    
    # เส้นหลัง header
    if idx == 0 && !header.empty?
      lines << col_widths.map { |w| "-" * w }.join(separator.gsub("|", "+"))
    end
  end
  
  lines.join("\n")
end

data = [
  ["Alice", "Engineer", "Bangkok"],
  ["Bob", "Designer", "Chiang Mai"],
  ["Charlie", "Manager", "Phuket"],
]

puts format_table(data, header: ["Name", "Role", "City"])
puts "\n"
puts format_table(data, header: ["Name", "Role", "City"], align: :center, separator: " │ ")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Pagination Method

```crystal
def paginate(
  total_items : Int32,
  page : Int32 = 1,
  per_page : Int32 = 10,
  max_pages : Int32 = 100
) : NamedTuple(
  items_start: Int32,
  items_end: Int32,
  current_page: Int32,
  total_pages: Int32,
  has_prev: Bool,
  has_next: Bool
)
  # TODO: implement
  # - คำนวณ items_start, items_end
  # - จำกัด page ให้อยู่ใน range ที่ valid
  # - จำกัด total_pages ให้ไม่เกิน max_pages
end

stats = paginate(95)
puts "หน้า #{stats[:current_page]}/#{stats[:total_pages]}"
puts "รายการ #{stats[:items_start]}-#{stats[:items_end]}"
puts "มีหน้าก่อน: #{stats[:has_prev]}, มีหน้าถัดไป: #{stats[:has_next]}"
```

### แบบฝึกหัดที่ 2: Color Parser

```crystal
def parse_color(
  input : String,
  default_alpha : Float64 = 1.0,
  allow_named : Bool = true
) : NamedTuple(r: Int32, g: Int32, b: Int32, a: Float64)?
  # TODO: parse สี ในรูปแบบ:
  # "#RRGGBB" => {r: Int32, g: Int32, b: Int32, a: 1.0}
  # "#RRGGBBAA" => {r, g, b, a}
  # "red", "green", "blue" (ถ้า allow_named = true)
  # return nil ถ้า parse ไม่ได้
end

puts parse_color("#FF5733").inspect
puts parse_color("#FF573380").inspect
puts parse_color("red", allow_named: true).inspect
puts parse_color("red", allow_named: false).inspect  # => nil
```

### แบบฝึกหัดที่ 3: CSV Writer

```crystal
def write_csv(
  data : Array(Array(String)),
  delimiter : String = ",",
  quote_char : String = "\"",
  line_ending : String = "\n",
  include_header : Bool = false,
  header : Array(String) = [] of String
) : String
  # TODO: สร้าง CSV string
  # รองรับ quoting สำหรับค่าที่มี delimiter
  # รองรับ header
end

data = [
  ["Alice", "30", "Bangkok, Thailand"],
  ["Bob", "25", "Chiang Mai"],
]

puts write_csv(data)
puts write_csv(data, delimiter: ";")
puts write_csv(data,
  include_header: true,
  header: ["Name", "Age", "City"]
)
```

---

## สรุป

| Pattern | ตัวอย่าง | เมื่อใช้ |
|---------|----------|---------|
| Simple default | `def f(x : Int32 = 0)` | ค่าเริ่มต้นง่ายๆ |
| Expression default | `def f(ts = Time.local)` | ค่า dynamic |
| Nil default | `def f(x : T? = nil)` | Optional parameter |
| Keyword with default | `def f(name: "Alice")` | Named optional |

### Gotchas ที่ต้องระวัง

1. **Crystal safe** - mutable default เช่น `[] of T` ถูก evaluate ใหม่ทุกครั้ง
2. **Default ที่ซับซ้อน** อาจทำให้ performance ได้รับผลกระทบ (evaluate ทุกครั้ง)
3. **ENV ใน default** - ระวัง! ค่าอาจเปลี่ยนระหว่าง calls
4. **ลำดับ** - required parameters ต้องมาก่อน default parameters

### Best Practices

1. ตั้ง default ที่สมเหตุสมผลสำหรับ usecase ทั่วไป
2. ใช้ `nil` default เมื่อ parameter เป็น optional จริงๆ
3. Document default values ที่ไม่ชัดเจนด้วย comment
4. ระวัง default ที่ขึ้นกับ global state หรือ environment
