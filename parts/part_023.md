# Part 023: Return Values

## บทนำ

ใน Crystal ทุก expression มีค่า และ method ก็ส่งค่ากลับเสมอ การเข้าใจระบบ return value ของ Crystal จะช่วยให้เขียนโค้ดที่กระชับ ปลอดภัย และอ่านง่ายมากขึ้น

---

## 1. Implicit Return

ใน Crystal ค่าของ expression สุดท้ายในบล็อกคือ return value อัตโนมัติ:

```crystal
def add(a : Int32, b : Int32) : Int32
  a + b  # implicit return - ไม่ต้องเขียน return
end

def greeting(name : String) : String
  "Hello, #{name}!"  # implicit return
end

def is_adult(age : Int32) : Bool
  age >= 18  # implicit return
end

puts add(3, 4)         # => 7
puts greeting("Alice") # => Hello, Alice!
puts is_adult(20)      # => true
```

### Implicit Return จาก if/case/begin

```crystal
# if expression ส่งค่ากลับ
def classify(n : Int32) : String
  if n > 0
    "บวก"
  elsif n < 0
    "ลบ"
  else
    "ศูนย์"
  end
  # return value คือผลลัพธ์ของ if expression
end

# case expression ส่งค่ากลับ
def day_name(n : Int32) : String
  case n
  when 1 then "จันทร์"
  when 2 then "อังคาร"
  when 3 then "พุธ"
  when 4 then "พฤหัสบดี"
  when 5 then "ศุกร์"
  when 6 then "เสาร์"
  when 7 then "อาทิตย์"
  else "ไม่ทราบ"
  end
end

puts classify(-5)     # => ลบ
puts day_name(3)      # => พุธ
```

---

## 2. Explicit Return

`return` ออกจาก method ทันที:

```crystal
# Early return (guard clause)
def process(data : String?) : String
  return "ไม่มีข้อมูล" unless data         # early return 1
  return "ข้อมูลว่าง" if data.empty?      # early return 2
  return "สั้นเกินไป" if data.size < 3   # early return 3
  
  # ถึงที่นี่: data ไม่ nil, ไม่ว่าง, ยาว >= 3
  data.upcase
end

puts process(nil)      # => ไม่มีข้อมูล
puts process("")       # => ข้อมูลว่าง
puts process("ab")     # => สั้นเกินไป
puts process("hello")  # => HELLO
```

### Return หลายจุด

```crystal
def find_user(id : Int32) : String?
  users = {1 => "Alice", 2 => "Bob", 3 => "Charlie"}
  
  return nil if id <= 0           # invalid id
  return nil unless users[id]?    # not found
  users[id]
end

puts find_user(0).inspect   # => nil
puts find_user(2).inspect   # => "Bob"
puts find_user(99).inspect  # => nil
```

---

## 3. Returning Nil

```crystal
# Method ที่อาจส่ง nil
def find_first_odd(numbers : Array(Int32)) : Int32?
  numbers.each do |n|
    return n if n.odd?
  end
  nil  # ไม่พบ
end

result = find_first_odd([2, 4, 6, 8])
puts result.inspect  # => nil

result = find_first_odd([2, 4, 5, 8])
puts result.inspect  # => 5

# การจัดการ nil return
if value = find_first_odd([1, 2, 3])
  puts "พบ: #{value}"
else
  puts "ไม่พบ"
end
```

### Nil Safety กับ Return Values

```crystal
def get_user_email(user_id : Int32) : String?
  # จำลอง database lookup
  db = {1 => "alice@example.com", 2 => "bob@example.com"}
  db[user_id]?
end

# ต้องตรวจสอบ nil ก่อนใช้
email = get_user_email(1)
if email
  puts email.upcase  # ปลอดภัย - Crystal รู้ว่า email ไม่ nil ใน block นี้
end

# หรือใช้ try (&.)
puts get_user_email(1)&.upcase.inspect   # => "ALICE@EXAMPLE.COM"
puts get_user_email(99)&.upcase.inspect  # => nil
```

---

## 4. Returning Multiple Values as Tuple

```crystal
# ส่งกลับ Tuple ที่มีหลายค่า
def min_max_sum(arr : Array(Int32)) : {Int32, Int32, Int32}
  {arr.min, arr.max, arr.sum}
end

min, max, sum = min_max_sum([3, 1, 4, 1, 5, 9, 2, 6])
puts "Min: #{min}, Max: #{max}, Sum: #{sum}"
# => Min: 1, Max: 9, Sum: 31

# ส่งกลับ Named Tuple
def parse_name(full_name : String) : NamedTuple(first: String, last: String, middle: String?)
  parts = full_name.split(" ")
  case parts.size
  when 1
    {first: parts[0], last: "", middle: nil}
  when 2
    {first: parts[0], last: parts[1], middle: nil}
  else
    {first: parts[0], last: parts[-1], middle: parts[1..-2].join(" ")}
  end
end

name = parse_name("John Michael Doe")
puts "First: #{name[:first]}"
puts "Middle: #{name[:middle]}"
puts "Last: #{name[:last]}"
```

### Result Type Pattern

```crystal
# Pattern ที่ใช้บ่อยในภาษาต่างๆ: Result{ok, error}
def safe_divide(a : Float64, b : Float64) : {Float64?, String?}
  if b == 0.0
    {nil, "หารด้วยศูนย์ไม่ได้"}
  else
    {a / b, nil}
  end
end

result, error = safe_divide(10.0, 2.0)
if error
  puts "Error: #{error}"
else
  puts "Result: #{result}"
end

result2, error2 = safe_divide(10.0, 0.0)
if error2
  puts "Error: #{error2}"
else
  puts "Result: #{result2}"
end
```

---

## 5. Return Type Annotations

```crystal
# ระบุ return type เพื่อ documentation และ compile-time check
def square(n : Int32) : Int32
  n * n
end

def name_length(name : String) : Int32
  name.size
end

# Return type สำหรับ generic
def identity(value : T) : T forall T
  value
end

puts identity(42)        # => 42
puts identity("hello")   # => hello
puts identity(3.14)      # => 3.14
```

### Return Type และ Nil

```crystal
# : String? บอกว่า method อาจ return nil
def find_by_id(id : Int32) : String?
  data = {1 => "Alice", 2 => "Bob"}
  data[id]?
end

# Crystal บังคับให้จัดการ nil
name = find_by_id(1)
# puts name.upcase  # Error: ไม่สามารถใช้ upcase กับ String | Nil
puts name.upcase if name  # OK: ตรวจสอบก่อน
```

### Return Type Union

```crystal
# Method อาจส่งค่าหลายประเภท
def parse_value(input : String) : Int32 | Float64 | String
  if input =~ /^\d+$/
    input.to_i
  elsif input =~ /^\d+\.\d+$/
    input.to_f
  else
    input
  end
end

result = parse_value("42")
case result
when Int32
  puts "Integer: #{result}"
when Float64
  puts "Float: #{result}"
when String
  puts "String: #{result}"
end
```

---

## 6. Void Methods

Method ที่ไม่ส่งค่ากลับที่มีความหมาย:

```crystal
# : Nil บอกว่า method ไม่ส่งค่ากลับที่มีความหมาย
def print_table(data : Array(Array(String))) : Nil
  data.each do |row|
    puts row.join(" | ")
  end
end

# หรือ : Void ก็ใช้ได้ (alias ของ Nil)
def clear_screen : Nil
  print "\033[2J\033[H"
end

# Method ที่ mutate state ส่วนใหญ่เป็น void
class Counter
  @count = 0
  
  def increment : Nil
    @count += 1
  end
  
  def reset : Nil
    @count = 0
  end
  
  def value : Int32
    @count
  end
end

counter = Counter.new
counter.increment
counter.increment
puts counter.value  # => 2
counter.reset
puts counter.value  # => 0
```

---

## 7. ตัวอย่างการใช้ Return Value จริง

### Parser

```crystal
def parse_csv_line(line : String) : Array(String)
  # Simple CSV parser (ไม่รองรับ quoted fields)
  line.split(",").map(&.strip)
end

def parse_key_value(line : String) : {String, String}?
  parts = line.split("=", 2)
  return nil unless parts.size == 2
  {parts[0].strip, parts[1].strip}
end

puts parse_csv_line("Alice, 30, Bangkok").inspect
# => ["Alice", "30", "Bangkok"]

puts parse_key_value("name = Alice").inspect
# => {"name", "Alice"}

puts parse_key_value("no_equals").inspect
# => nil
```

### Validator Chain

```crystal
# Return type แบบ Result Pattern
struct ValidationResult
  getter valid : Bool
  getter errors : Array(String)
  
  def initialize(@valid : Bool, @errors : Array(String) = [] of String)
  end
  
  def self.ok : ValidationResult
    new(true)
  end
  
  def self.fail(*errors : String) : ValidationResult
    new(false, errors.to_a)
  end
end

def validate_username(name : String) : ValidationResult
  errors = [] of String
  errors << "ต้องมีความยาว 3-20 ตัวอักษร" unless (3..20).includes?(name.size)
  errors << "ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _" unless name =~ /^[a-zA-Z0-9_]+$/
  errors << "ต้องเริ่มด้วยตัวอักษร" unless name =~ /^[a-zA-Z]/
  
  errors.empty? ? ValidationResult.ok : ValidationResult.fail(*errors)
end

["alice123", "ab", "123start", "valid_user", "has spaces"].each do |name|
  result = validate_username(name)
  if result.valid
    puts "✓ #{name}"
  else
    puts "✗ #{name}: #{result.errors.join(", ")}"
  end
end
```

### Pipeline Pattern

```crystal
# Method chain ที่ return self
class QueryBuilder
  @table : String = ""
  @conditions : Array(String) = [] of String
  @order : String? = nil
  @limit : Int32? = nil
  
  def from(table : String) : self
    @table = table
    self
  end
  
  def where(condition : String) : self
    @conditions << condition
    self
  end
  
  def order_by(field : String) : self
    @order = field
    self
  end
  
  def limit(n : Int32) : self
    @limit = n
    self
  end
  
  def build : String
    sql = "SELECT * FROM #{@table}"
    sql += " WHERE #{@conditions.join(" AND ")}" unless @conditions.empty?
    sql += " ORDER BY #{@order}" if @order
    sql += " LIMIT #{@limit}" if @limit
    sql
  end
end

query = QueryBuilder.new
  .from("users")
  .where("age > 18")
  .where("active = true")
  .order_by("name ASC")
  .limit(10)
  .build

puts query
# => SELECT * FROM users WHERE age > 18 AND active = true ORDER BY name ASC LIMIT 10
```

---

## 8. Return Values และ Block

```crystal
# map ส่งคืน array ใหม่
result = [1, 2, 3].map { |n| n * 2 }
puts result.inspect  # => [2, 4, 6]

# each ส่งคืน original array (หรือ nil ถ้า break)
original = [1, 2, 3]
returned = original.each { |n| n * 2 }
puts returned.inspect  # => [1, 2, 3]

# select ส่งคืน filtered array
evens = [1, 2, 3, 4, 5].select(&.even?)
puts evens.inspect  # => [2, 4]

# reduce ส่งคืน accumulated value
sum = [1, 2, 3, 4, 5].reduce(0) { |acc, n| acc + n }
puts sum  # => 15
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Safe Operations

```crystal
# สร้าง safe math operations ที่ return nil แทน exception
def safe_sqrt(n : Float64) : Float64?
  # TODO: return nil ถ้า n < 0
end

def safe_log(n : Float64) : Float64?
  # TODO: return nil ถ้า n <= 0
end

def safe_parse_int(s : String) : Int32?
  # TODO: return nil ถ้า parse ไม่ได้
end

# Test
puts safe_sqrt(16.0).inspect   # => 4.0
puts safe_sqrt(-4.0).inspect   # => nil
puts safe_log(100.0).inspect   # => 4.605...
puts safe_log(-1.0).inspect    # => nil
puts safe_parse_int("42").inspect  # => 42
puts safe_parse_int("abc").inspect # => nil
```

### แบบฝึกหัดที่ 2: Named Tuple Returns

```crystal
# สร้าง text analyzer ที่ return Named Tuple
def analyze_text(text : String) : NamedTuple(
  words: Int32,
  sentences: Int32,
  paragraphs: Int32,
  avg_sentence_length: Float64,
  most_common_word: String
)
  # TODO: implement
end

text = """
Crystal เป็นภาษาโปรแกรมที่เร็วมาก Crystal ทำงานได้เร็วเหมือน C
โปรแกรมเมอร์ชอบ Crystal เพราะมีไวยากรณ์ที่สวยงาม
Crystal ใช้ LLVM ในการ compile
"""

stats = analyze_text(text)
puts "คำ: #{stats[:words]}"
puts "ประโยค: #{stats[:sentences]}"
puts "คำที่พบบ่อยสุด: #{stats[:most_common_word]}"
```

### แบบฝึกหัดที่ 3: Void vs Value Methods

```crystal
class ShoppingCart
  @items : Array({name: String, price: Float64, qty: Int32})
  
  def initialize
    @items = [] of {name: String, price: Float64, qty: Int32}
  end
  
  # Void method - mutate state
  def add_item(name : String, price : Float64, qty : Int32 = 1) : Nil
    # TODO
  end
  
  # Void method - mutate state
  def remove_item(name : String) : Nil
    # TODO
  end
  
  # Value method - calculate and return
  def subtotal : Float64
    # TODO: sum of price * qty
  end
  
  # Value method
  def item_count : Int32
    # TODO: total quantity
  end
  
  # Value method
  def summary : String
    # TODO: formatted string
  end
end

cart = ShoppingCart.new
cart.add_item("แอปเปิ้ล", 30.0, 3)
cart.add_item("กล้วย", 20.0, 5)
cart.add_item("ส้ม", 25.0)
puts cart.subtotal      # => 220.0
puts cart.item_count    # => 9
puts cart.summary
```

---

## สรุป

| Pattern | Syntax | เมื่อใช้ |
|---------|--------|---------|
| Implicit return | ค่าสุดท้าย | ส่วนใหญ่ของ methods |
| Explicit return | `return value` | Early return, guard clauses |
| Return nil | `nil` หรือ `: T?` | ไม่พบ/ไม่มีค่า |
| Return tuple | `{a, b}` | Multiple values |
| Named tuple | `{key: val}` | Structured multiple values |
| Void | `: Nil` | Mutating methods |
| Method chaining | `return self` | Builder pattern |

### Best Practices

1. **ระบุ return type** เสมอสำหรับ public methods
2. **ใช้ implicit return** ทำให้โค้ดกระชับ
3. **Early return** สำหรับ guard clauses ลด nesting
4. **`T?` (nilable)** แทนการ raise exception เมื่อ "ไม่พบ" เป็นเรื่องปกติ
5. **Named Tuple** แทน Hash สำหรับ return หลายค่าที่มีชื่อ
6. **`: Nil`** บน mutating methods เพื่อสื่อสาร intent
