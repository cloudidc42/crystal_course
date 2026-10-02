# Part 62: String Interpolation และ Formatting

## บทนำ

การจัดรูปแบบสตริงเป็นทักษะสำคัญในการเขียนโปรแกรม Crystal มีหลายวิธีในการ format ข้อมูลให้อยู่ในรูปแบบที่ต้องการ ตั้งแต่ string interpolation พื้นฐานไปจนถึง sprintf ขั้นสูง

---

## 1. String Interpolation พื้นฐาน `#{}`

```crystal
# interpolation พื้นฐาน
name = "Crystal"
puts "Hello, #{name}!"  # => "Hello, Crystal!"

# expression ภายใน #{}
x = 10
puts "x squared = #{x ** 2}"  # => "x squared = 100"

# method call
words = ["hello", "world"]
puts "Words: #{words.join(", ")}"  # => "Words: hello, world"

# conditional
age = 20
puts "You are #{age >= 18 ? "adult" : "minor"}"  # => "You are adult"

# multiline expression
result = "Sum: #{
  a = 10
  b = 20
  a + b
}"
puts result  # => "Sum: 30"

# nested interpolation
inner = "inner"
puts "outer #{" #{inner} "}"  # => "outer   inner  "

# interpolation กับ special types
arr = [1, 2, 3]
hash = {"a" => 1}
puts "Array: #{arr}"     # => "Array: [1, 2, 3]"
puts "Hash: #{hash}"     # => "Hash: {"a" => 1}"

# ใช้ to_s explicitly
puts "Value: #{42.to_s(16)}"  # hex: => "Value: 2a"
puts "Value: #{42.to_s(2)}"   # binary: => "Value: 101010"
puts "Value: #{42.to_s(8)}"   # octal: => "Value: 52"
```

---

## 2. sprintf - String Formatting ขั้นสูง

```crystal
# sprintf พื้นฐาน
puts sprintf("Hello, %s!", "World")     # => "Hello, World!"
puts sprintf("Pi = %.4f", 3.14159265)  # => "Pi = 3.1416"

# format specifiers
puts sprintf("%d", 42)        # integer: => "42"
puts sprintf("%05d", 42)      # zero-padded: => "00042"
puts sprintf("%+d", 42)       # with sign: => "+42"
puts sprintf("%+d", -42)      # negative: => "-42"

puts sprintf("%f", 3.14)      # float: => "3.140000"
puts sprintf("%.2f", 3.14)    # 2 decimal places: => "3.14"
puts sprintf("%10.2f", 3.14)  # width 10, 2 dec: => "      3.14"
puts sprintf("%-10.2f", 3.14) # left-aligned: => "3.14      "

puts sprintf("%s", "hello")   # string: => "hello"
puts sprintf("%10s", "hello") # right-aligned: => "     hello"
puts sprintf("%-10s", "hello")# left-aligned: => "hello     "

puts sprintf("%x", 255)  # hex lowercase: => "ff"
puts sprintf("%X", 255)  # hex uppercase: => "FF"
puts sprintf("%#x", 255) # hex with prefix: => "0xff"
puts sprintf("%o", 8)    # octal: => "10"
puts sprintf("%b", 10)   # binary: => "1010"
puts sprintf("%e", 1234.5) # scientific: => "1.234500e+03"
puts sprintf("%E", 1234.5) # scientific upper: => "1.234500E+03"
puts sprintf("%g", 1234.5) # shorter of f/e: => "1234.5"

# multiple arguments
puts sprintf("Name: %s, Age: %d, Score: %.1f", "Alice", 25, 95.5)
# => "Name: Alice, Age: 25, Score: 95.5"
```

---

## 3. % Operator

```crystal
# % เป็น shorthand สำหรับ sprintf
puts "Hello, %s!" % "World"        # => "Hello, World!"
puts "Pi = %.4f" % 3.14159265      # => "Pi = 3.1416"
puts "%d + %d = %d" % {1, 2, 3}   # => "1 + 2 = 3"

# ใช้กับ Array
values = ["Alice", 25, 95.5]
puts "Name: %s, Age: %d, Score: %.1f" % values
# => "Name: Alice, Age: 25, Score: 95.5"

# format ตัวเลขเป็น currency
def format_currency(amount : Float64, symbol : String = "฿") : String
  "#{symbol}%,.2f" % amount
end

# Crystal ไม่รองรับ %,f โดยตรง ต้องทำ manual
def format_number_with_commas(n : Float64, decimals : Int32 = 2) : String
  integer_part = n.to_i.to_s
  decimal_part = sprintf("%.#{decimals}f", n - n.to_i)[1..]  # ".XX"
  
  # ใส่ comma ทุก 3 หลัก
  with_commas = integer_part.reverse.chars
    .each_slice(3)
    .map(&.join)
    .join(",")
    .reverse
  
  with_commas + decimal_part
end

puts format_number_with_commas(1234567.89)  # => "1,234,567.89"
puts format_number_with_commas(42.5, 1)     # => "42.5"
```

---

## 4. IO#printf และ printf

```crystal
# printf พิมพ์โดยตรงไปที่ STDOUT
printf("Hello, %s!\n", "World")
printf("Pi = %.4f\n", 3.14159)

# STDOUT.printf
STDOUT.printf("Score: %d%%\n", 95)

# STDERR.printf สำหรับ error messages
STDERR.printf("Error: %s (code: %d)\n", "Not found", 404)

# fprintf equivalent - ส่ง output ไปยัง IO ที่ระบุ
io = IO::Memory.new
io.printf("Name: %s\n", "Alice")
io.printf("Age: %d\n", 25)
puts io.to_s

# ใช้ในการสร้าง log
def log(level : String, message : String, io : IO = STDOUT)
  io.printf("[%s] %s: %s\n", Time.local.to_s("%H:%M:%S"), level, message)
end

log("INFO", "Server started")
log("ERROR", "Connection failed", STDERR)
```

---

## 5. การจัดระเบียบข้อความ - Alignment

```crystal
# Right alignment (default)
puts "%10s" % "hello"   # => "     hello"
puts "%10d" % 42        # => "        42"

# Left alignment ด้วย -
puts "%-10s" % "hello"  # => "hello     "
puts "%-10d" % 42       # => "42        "

# Center alignment ไม่มี built-in ต้องทำเอง
def center(str : String, width : Int32, pad : Char = ' ') : String
  return str if str.size >= width
  total_pad = width - str.size
  left_pad = total_pad // 2
  right_pad = total_pad - left_pad
  pad.to_s * left_pad + str + pad.to_s * right_pad
end

puts center("hello", 11)        # => "   hello   "
puts center("hello", 11, '-')   # => "---hello---"
puts center("hi", 10)           # => "    hi    "

# สร้างตาราง aligned
def print_table(headers : Array(String), rows : Array(Array(String)))
  # คำนวณ column widths
  col_widths = headers.map(&.size)
  rows.each do |row|
    row.each_with_index do |cell, i|
      col_widths[i] = [col_widths[i], cell.size].max if i < col_widths.size
    end
  end
  
  # print header
  header_str = headers.each_with_index.map { |h, i| "%-#{col_widths[i]}s" % h }.join(" | ")
  separator = col_widths.map { |w| "-" * w }.join("-+-")
  
  puts header_str
  puts separator
  rows.each do |row|
    puts row.each_with_index.map { |cell, i| "%-#{col_widths[i]}s" % cell }.join(" | ")
  end
end

headers = ["Name", "Department", "Salary"]
rows = [
  ["Alice", "Engineering", "85,000"],
  ["Bob", "Marketing", "72,000"],
  ["Charlie", "Engineering", "92,000"],
  ["Diana", "HR", "68,000"],
]

print_table(headers, rows)
```

---

## 6. Zero Padding และ Number Formatting

```crystal
# Zero padding
puts "%05d" % 42       # => "00042"
puts "%08.2f" % 3.14   # => "00003.14"
puts "%010x" % 255     # => "0x000000ff" (ถ้าใช้ %#x)

# Leading zeros สำหรับ time
def format_time(h : Int32, m : Int32, s : Int32) : String
  "%02d:%02d:%02d" % {h, m, s}
end

puts format_time(9, 5, 3)   # => "09:05:03"
puts format_time(14, 30, 0)  # => "14:30:00"

# Padding สำหรับ ID
def format_id(n : Int32, prefix : String = "ID", digits : Int32 = 6) : String
  "#{prefix}-%0#{digits}d" % n
end

puts format_id(42)      # => "ID-000042"
puts format_id(1234, "ORD", 8)  # => "ORD-00001234"

# Phone number formatting
def format_phone(digits : String) : String
  cleaned = digits.delete("^0-9")
  if cleaned.size == 10
    "(%s) %s-%s" % {cleaned[0, 3], cleaned[3, 3], cleaned[6, 4]}
  else
    digits  # return as-is if invalid
  end
end

puts format_phone("0812345678")   # => "(081) 234-5678"
puts format_phone("081-234-5678") # => "(081) 234-5678"
```

---

## 7. Custom Format Methods

```crystal
# สร้าง custom formatter
class NumberFormatter
  def initialize(@locale : String = "en")
  end
  
  def currency(amount : Float64, symbol : String = "$") : String
    "#{symbol}#{format(amount, 2)}"
  end
  
  def percentage(value : Float64, decimals : Int32 = 1) : String
    "#{format(value, decimals)}%"
  end
  
  def format(number : Float64, decimals : Int32 = 2) : String
    integer_part = number.abs.to_i
    decimal_part = (number.abs - integer_part).round(decimals)
    
    int_str = insert_thousands_separator(integer_part.to_s)
    dec_str = sprintf("%.#{decimals}f", decimal_part)[1..]  # ".XX"
    
    prefix = number < 0 ? "-" : ""
    prefix + int_str + dec_str
  end
  
  private def insert_thousands_separator(str : String) : String
    result = [] of String
    str.reverse.chars.each_slice(3) { |s| result << s.join }
    result.join(",").reverse
  end
end

fmt = NumberFormatter.new
puts fmt.currency(1234567.89)       # => "$1,234,567.89"
puts fmt.percentage(85.5)           # => "85.5%"
puts fmt.format(9876543.21)         # => "9,876,543.21"
puts fmt.format(-1234.5, 3)        # => "-1,234.500"

# DSL-style formatter
module Fmt
  def self.duration(seconds : Int64) : String
    hours = seconds // 3600
    minutes = (seconds % 3600) // 60
    secs = seconds % 60
    
    if hours > 0
      "%d:%02d:%02d" % {hours, minutes, secs}
    else
      "%d:%02d" % {minutes, secs}
    end
  end
  
  def self.bytes(n : Int64) : String
    units = ["B", "KB", "MB", "GB", "TB"]
    i = 0
    value = n.to_f64
    
    while value >= 1024 && i < units.size - 1
      value /= 1024
      i += 1
    end
    
    if i == 0
      "#{n} B"
    else
      "%.2f %s" % {value, units[i]}
    end
  end
  
  def self.date(time : Time) : String
    time.to_s("%d/%m/%Y")
  end
end

puts Fmt.duration(3661)       # => "1:01:01"
puts Fmt.duration(125)        # => "2:05"
puts Fmt.bytes(1024)          # => "1.00 KB"
puts Fmt.bytes(1_048_576)     # => "1.00 MB"
puts Fmt.bytes(500)           # => "500 B"
puts Fmt.date(Time.local)     # date ปัจจุบัน
```

---

## 8. Heredoc

```crystal
# heredoc พื้นฐาน
text = <<-TEXT
  Hello, World!
  This is a heredoc.
  TEXT

puts text
# => Hello, World!
# => This is a heredoc.

# heredoc กับ interpolation
name = "Crystal"
version = "1.0"

message = <<-MSG
  Welcome to #{name}!
  Version: #{version}
  
  Happy coding!
  MSG

puts message

# heredoc ไม่ต้องการ indentation ถ้าใช้ <<-
code = <<-CRYSTAL
  def hello
    puts "Hello!"
  end
  CRYSTAL

puts code

# heredoc กับ method chaining
html = <<-HTML.gsub("{{name}}", "Alice")
  <html>
  <body>
    <h1>Hello, {{name}}!</h1>
  </body>
  </html>
  HTML

puts html

# heredoc สำหรับ SQL
def build_query(table : String, conditions : String) : String
  <<-SQL
    SELECT *
    FROM #{table}
    WHERE #{conditions}
    ORDER BY created_at DESC
    LIMIT 100
    SQL
end

puts build_query("users", "age > 18 AND active = true")
```

---

## 9. Format Strings สำหรับ Time

```crystal
# Time formatting
now = Time.local

puts now.to_s                         # default format
puts now.to_s("%Y-%m-%d")             # => "2024-01-15"
puts now.to_s("%d/%m/%Y")             # => "15/01/2024"
puts now.to_s("%H:%M:%S")             # => "14:30:00"
puts now.to_s("%Y-%m-%d %H:%M:%S")   # => "2024-01-15 14:30:00"
puts now.to_s("%A, %B %d, %Y")       # => "Monday, January 15, 2024"
puts now.to_s("%I:%M %p")             # => "02:30 PM"

# ISO 8601
puts now.to_s(Time::Format::ISO_8601_DATE_TIME)
# => "2024-01-15T14:30:00+07:00"

# RFC 2822 (email format)
puts now.to_s(Time::Format::RFC_2822)
# => "Mon, 15 Jan 2024 14:30:00 +0700"

# วันภาษาไทย (manual)
THAI_MONTHS = {
  "January" => "มกราคม", "February" => "กุมภาพันธ์",
  "March" => "มีนาคม", "April" => "เมษายน",
  "May" => "พฤษภาคม", "June" => "มิถุนายน",
  "July" => "กรกฎาคม", "August" => "สิงหาคม",
  "September" => "กันยายน", "October" => "ตุลาคม",
  "November" => "พฤศจิกายน", "December" => "ธันวาคม",
}

def to_thai_date(time : Time) : String
  en_month = time.to_s("%B")
  thai_month = THAI_MONTHS[en_month]
  thai_year = time.year + 543
  "#{time.day} #{thai_month} พ.ศ. #{thai_year}"
end

puts to_thai_date(Time.local)  # => "15 มกราคม พ.ศ. 2567"
```

---

## 10. Template Engine อย่างง่าย

```crystal
# Simple template engine
class SimpleTemplate
  def initialize(@template : String)
  end
  
  def render(vars : Hash(String, String)) : String
    result = @template
    vars.each do |key, value|
      result = result.gsub("{{#{key}}}", value)
    end
    result
  end
  
  def render(**vars) : String
    hash = {} of String => String
    vars.each { |k, v| hash[k.to_s] = v.to_s }
    render(hash)
  end
end

template = SimpleTemplate.new(<<-TEMPLATE)
  Dear {{name}},
  
  Your order #{{order_id}} has been {{status}}.
  Total: {{total}}
  
  Thank you for shopping with us!
  TEMPLATE

puts template.render(
  name: "Alice",
  order_id: "ORD-001234",
  status: "confirmed",
  total: "฿1,234.56"
)

# Template กับ loops (more complex)
class Template
  def initialize(@template : String)
  end
  
  def render(context : Hash(String, String | Array(Hash(String, String)))) : String
    result = @template
    
    # Handle simple variables
    context.each do |key, value|
      next if value.is_a?(Array)
      result = result.gsub("{{#{key}}}", value.as(String))
    end
    
    # Handle loops: {{#items}}...{{/items}}
    context.each do |key, value|
      next unless value.is_a?(Array)
      items = value.as(Array(Hash(String, String)))
      
      pattern = /\{\{##{Regex.escape(key)}\}\}(.*?)\{\{\/#{Regex.escape(key)}\}\}/ms
      result = result.gsub(pattern) do
        block_template = $~[1]
        items.map do |item|
          block_result = block_template
          item.each do |k, v|
            block_result = block_result.gsub("{{#{k}}}", v)
          end
          block_result
        end.join
      end
    end
    
    result
  end
end

tmpl = Template.new(<<-TMPL)
  Order Summary for {{customer}}:
  {{#items}}
    - {{name}}: {{price}}
  {{/items}}
  Total: {{total}}
  TMPL

result = tmpl.render({
  "customer" => "Bob",
  "items" => [
    {"name" => "Widget", "price" => "฿100"},
    {"name" => "Gadget", "price" => "฿250"},
  ] of Hash(String, String),
  "total" => "฿350",
} of String => String | Array(Hash(String, String)))

puts result
```

---

## 11. Format สำหรับ Debugging และ Logging

```crystal
# สร้าง formatted log messages
module Logger
  enum Level
    DEBUG
    INFO
    WARN
    ERROR
    FATAL
  end
  
  LEVEL_COLORS = {
    Level::DEBUG => "\e[36m",  # cyan
    Level::INFO  => "\e[32m",  # green
    Level::WARN  => "\e[33m",  # yellow
    Level::ERROR => "\e[31m",  # red
    Level::FATAL => "\e[35m",  # magenta
  }
  RESET = "\e[0m"
  
  def self.log(level : Level, message : String, context : Hash(String, String) = {} of String => String)
    timestamp = Time.local.to_s("%Y-%m-%d %H:%M:%S.%3N")
    color = LEVEL_COLORS[level]
    
    ctx_str = context.empty? ? "" : " " + context.map { |k, v| "#{k}=#{v}" }.join(" ")
    
    STDOUT.printf("%s[%s]%s [%s] %s%s\n",
      color, level.to_s, RESET, timestamp, message, ctx_str)
  end
  
  def self.debug(msg : String, **ctx); log(Level::DEBUG, msg, ctx.to_h.transform_values(&.to_s)); end
  def self.info(msg : String, **ctx); log(Level::INFO, msg, ctx.to_h.transform_values(&.to_s)); end
  def self.warn(msg : String, **ctx); log(Level::WARN, msg, ctx.to_h.transform_values(&.to_s)); end
  def self.error(msg : String, **ctx); log(Level::ERROR, msg, ctx.to_h.transform_values(&.to_s)); end
end

Logger.info("Server started", port: 8080, env: "production")
Logger.warn("High memory usage", usage: "85%")
Logger.error("Database connection failed", host: "localhost", port: 5432)

# Pretty print สำหรับ debugging
def pp_hash(hash : Hash, indent : Int32 = 0) : String
  return "{}" if hash.empty?
  
  String.build do |io|
    io << "{\n"
    hash.each_with_index do |(key, value), i|
      io << "  " * (indent + 1)
      io << key.inspect << " => "
      io << (value.is_a?(Hash) ? pp_hash(value.as(Hash), indent + 1) : value.inspect)
      io << "," unless i == hash.size - 1
      io << "\n"
    end
    io << "  " * indent << "}"
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `format_invoice` ที่รับ items (name, quantity, price) และสร้าง invoice ที่ formatted

### แบบฝึกหัดที่ 2
เขียน template engine ที่รองรับ `{{if condition}}...{{/if}}` blocks

### แบบฝึกหัดที่ 3
เขียนฟังก์ชัน `humanize_duration(seconds)` ที่แปลงวินาทีเป็น "2 hours, 30 minutes, 15 seconds"

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
def format_invoice(items : Array(NamedTuple(name: String, qty: Int32, price: Float64))) : String
  String.build do |io|
    io << "=" * 50 << "\n"
    io << "%-25s %5s %10s %10s\n" % {"ITEM", "QTY", "PRICE", "TOTAL"}
    io << "-" * 50 << "\n"
    
    subtotal = 0.0
    items.each do |item|
      total = item[:qty] * item[:price]
      subtotal += total
      io << "%-25s %5d %10.2f %10.2f\n" % {item[:name], item[:qty], item[:price], total}
    end
    
    io << "=" * 50 << "\n"
    io << "%-41s %9.2f\n" % {"SUBTOTAL:", subtotal}
    vat = subtotal * 0.07
    io << "%-41s %9.2f\n" % {"VAT (7%):", vat}
    io << "=" * 50 << "\n"
    io << "%-41s %9.2f\n" % {"TOTAL:", subtotal + vat}
  end
end

items = [
  {name: "Widget A", qty: 3, price: 150.00},
  {name: "Widget B", qty: 1, price: 850.00},
  {name: "Service Fee", qty: 2, price: 200.00},
]

puts format_invoice(items)

# แบบฝึกหัดที่ 3
def humanize_duration(total_seconds : Int64) : String
  parts = [] of String
  
  days = total_seconds // 86400
  hours = (total_seconds % 86400) // 3600
  minutes = (total_seconds % 3600) // 60
  seconds = total_seconds % 60
  
  parts << "#{days} #{days == 1 ? "day" : "days"}" if days > 0
  parts << "#{hours} #{hours == 1 ? "hour" : "hours"}" if hours > 0
  parts << "#{minutes} #{minutes == 1 ? "minute" : "minutes"}" if minutes > 0
  parts << "#{seconds} #{seconds == 1 ? "second" : "seconds"}" if seconds > 0
  
  parts.empty? ? "0 seconds" : parts.join(", ")
end

puts humanize_duration(3661)     # => "1 hour, 1 minute, 1 second"
puts humanize_duration(90061)    # => "1 day, 1 hour, 1 minute, 1 second"
puts humanize_duration(45)       # => "45 seconds"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **String Interpolation** `#{}` - ฝัง expressions ในสตริงได้อย่างยืดหยุ่น
2. **sprintf** - format สตริงแบบ C-style
3. **% operator** - shorthand สำหรับ sprintf
4. **IO#printf** - พิมพ์โดยตรงไปยัง IO stream
5. **Alignment** - จัดตำแหน่งข้อความด้วย left/right/center
6. **Zero padding** - เติม 0 นำหน้าตัวเลข
7. **Custom formatters** - สร้าง formatter ของตัวเอง
8. **Heredoc** - multiline strings ที่อ่านง่าย
9. **Time formatting** - format วันเวลาในรูปแบบต่างๆ
10. **Template engines** - สร้าง template engine อย่างง่าย

การ format สตริงที่ดีทำให้โค้ดอ่านง่ายและ output ที่ผู้ใช้เห็นดูเป็นมืออาชีพ
