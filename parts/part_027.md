# Part 27: Blocks และ Yield

## บทนำ

Blocks เป็นหนึ่งในคุณสมบัติที่ทรงพลังที่สุดของ Crystal โดยได้รับแรงบันดาลใจจาก Ruby Block คือโค้ดชิ้นหนึ่งที่สามารถส่งผ่านไปยังเมธอดได้ และเมธอดสามารถเรียกใช้โค้ดนั้นได้ด้วยคำสั่ง `yield`

---

## 1. Block Syntax พื้นฐาน

### 1.1 สองรูปแบบของ Block

```crystal
# รูปแบบที่ 1: Inline block ด้วย { }
[1, 2, 3].each { |x| puts x }

# รูปแบบที่ 2: Multi-line block ด้วย do...end
[1, 2, 3].each do |x|
  puts x
end
```

### 1.2 Block กับ Parameter

```crystal
# Block ไม่มี parameter
3.times { puts "สวัสดี" }

# Block มี 1 parameter
[10, 20, 30].each { |n| puts "ตัวเลข: #{n}" }

# Block มีหลาย parameter
{a: 1, b: 2, c: 3}.each { |key, value| puts "#{key} = #{value}" }

# Block มีหลาย parameter แบบ do...end
[[1, "a"], [2, "b"], [3, "c"]].each do |num, letter|
  puts "#{num} -> #{letter}"
end
```

### 1.3 Block Return Value

```crystal
# Block ส่งคืนค่าของ expression สุดท้าย
result = [1, 2, 3, 4, 5].map { |x| x * 2 }
puts result.inspect  # => [2, 4, 6, 8, 10]

# block ที่มีหลาย expression
squares = [1, 2, 3].map do |x|
  doubled = x * 2
  doubled * doubled  # ค่านี้คือค่าที่ส่งคืน
end
puts squares.inspect  # => [4, 16, 36]
```

---

## 2. yield - การเรียก Block

### 2.1 yield พื้นฐาน

```crystal
# สร้างเมธอดที่รับ block
def say_hello
  puts "ก่อน block"
  yield  # เรียกใช้ block
  puts "หลัง block"
end

say_hello { puts "อยู่ใน block!" }
# => ก่อน block
# => อยู่ใน block!
# => หลัง block
```

### 2.2 yield หลายครั้ง

```crystal
def repeat(n : Int32)
  n.times { yield }
end

repeat(3) { puts "ทำซ้ำ!" }
# => ทำซ้ำ!
# => ทำซ้ำ!
# => ทำซ้ำ!
```

### 2.3 yield กับ Block ที่ไม่มี parameter

```crystal
def benchmark
  start_time = Time.monotonic
  yield
  elapsed = Time.monotonic - start_time
  puts "ใช้เวลา: #{elapsed.total_milliseconds.round(2)}ms"
end

benchmark do
  # โค้ดที่ต้องการวัดเวลา
  sum = 0
  1000.times { |i| sum += i }
  puts "ผลรวม: #{sum}"
end
```

---

## 3. yield with Arguments

### 3.1 ส่ง argument ไปยัง block

```crystal
def count_up(from : Int32, to : Int32)
  current = from
  while current <= to
    yield current
    current += 1
  end
end

count_up(1, 5) { |n| puts n }
# => 1
# => 2
# => 3
# => 4
# => 5
```

### 3.2 ส่ง argument หลายตัว

```crystal
def each_with_square(array : Array(Int32))
  array.each do |item|
    yield item, item * item
  end
end

each_with_square([1, 2, 3, 4, 5]) do |num, square|
  puts "#{num}² = #{square}"
end
# => 1² = 1
# => 2² = 4
# => 3² = 9
# => 4² = 16
# => 5² = 25
```

### 3.3 รับค่าจาก yield

```crystal
def transform(array : Array(Int32))
  result = [] of Int32
  array.each do |item|
    transformed = yield item  # รับค่าที่ block ส่งคืน
    result << transformed
  end
  result
end

doubled = transform([1, 2, 3, 4]) { |x| x * 2 }
puts doubled.inspect  # => [2, 4, 6, 8]

squared = transform([1, 2, 3, 4]) { |x| x ** 2 }
puts squared.inspect  # => [1, 4, 9, 16]
```

### 3.4 สร้าง Custom Iterators

```crystal
# Iterator สำหรับ fibonacci
def fibonacci
  a, b = 0, 1
  loop do
    yield a
    a, b = b, a + b
  end
end

# เอาแค่ 10 ตัวแรก
count = 0
fibonacci do |n|
  puts n
  count += 1
  break if count >= 10
end
```

---

## 4. block_given? - ตรวจสอบการส่ง Block

### 4.1 การใช้ block_given?

```crystal
def optional_block
  if block_given?
    puts "มี block ถูกส่งมา"
    yield
  else
    puts "ไม่มี block"
  end
end

optional_block { puts "Hello from block!" }
# => มี block ถูกส่งมา
# => Hello from block!

optional_block
# => ไม่มี block
```

### 4.2 เมธอดที่ทำงานได้ทั้งมีและไม่มี block

```crystal
def find_or_default(array : Array(Int32), default : Int32)
  if block_given?
    array.find { |x| yield x } || default
  else
    array.first? || default
  end
end

numbers = [1, 5, 3, 8, 2]

# ใช้แบบไม่มี block
puts find_or_default(numbers, -1)  # => 1

# ใช้แบบมี block
puts find_or_default(numbers, -1) { |x| x > 4 }  # => 5
puts find_or_default(numbers, -1) { |x| x > 10 }  # => -1
```

### 4.3 ตัวอย่าง Database-like API

```crystal
class Collection
  def initialize(@items : Array(String))
  end

  def select_items
    if block_given?
      @items.select { |item| yield item }
    else
      @items.dup
    end
  end

  def map_items
    if block_given?
      @items.map { |item| yield item }
    else
      @items.dup
    end
  end
end

coll = Collection.new(["apple", "banana", "cherry", "apricot"])

puts coll.select_items.inspect                         # ทั้งหมด
puts coll.select_items { |x| x.starts_with?("a") }.inspect  # เฉพาะที่ขึ้นต้น a
puts coll.map_items { |x| x.upcase }.inspect           # แปลงเป็นตัวใหญ่
```

---

## 5. &block Parameter

### 5.1 การจับ Block เป็น Parameter

```crystal
# &block แปลง block เป็น Proc object
def apply(&block : Int32 -> Int32)
  block.call(10)
end

result = apply { |x| x * 2 }
puts result  # => 20

result = apply { |x| x + 5 }
puts result  # => 15
```

### 5.2 เก็บ Block สำหรับใช้ภายหลัง

```crystal
class EventHandler
  def initialize
    @handlers = {} of String => Proc(String, Nil)
  end

  def on(event : String, &handler : String ->)
    @handlers[event] = handler
  end

  def trigger(event : String, data : String)
    if handler = @handlers[event]?
      handler.call(data)
    else
      puts "ไม่มี handler สำหรับ event: #{event}"
    end
  end
end

eh = EventHandler.new

eh.on("login") { |user| puts "#{user} เข้าสู่ระบบแล้ว" }
eh.on("logout") { |user| puts "#{user} ออกจากระบบแล้ว" }

eh.trigger("login", "สมชาย")   # => สมชาย เข้าสู่ระบบแล้ว
eh.trigger("logout", "สมหญิง") # => สมหญิง ออกจากระบบแล้ว
eh.trigger("click", "button")  # => ไม่มี handler สำหรับ event: click
```

### 5.3 Block Type Specification

```crystal
# ระบุประเภทของ block อย่างชัดเจน
def filter_numbers(nums : Array(Int32), &predicate : Int32 -> Bool)
  nums.select { |n| predicate.call(n) }
end

evens = filter_numbers([1, 2, 3, 4, 5, 6]) { |n| n.even? }
puts evens.inspect  # => [2, 4, 6]

odds = filter_numbers([1, 2, 3, 4, 5, 6]) { |n| n.odd? }
puts odds.inspect  # => [1, 3, 5]
```

---

## 6. Proc.new จาก Block

### 6.1 การสร้าง Proc จาก Block

```crystal
# สร้าง Proc โดยตรง
double = Proc(Int32, Int32).new { |x| x * 2 }
puts double.call(5)   # => 10
puts double.call(10)  # => 20

# หรือใช้ -> syntax
triple = ->(x : Int32) { x * 3 }
puts triple.call(5)  # => 15
```

### 6.2 แปลง Block เป็น Proc และส่งต่อ

```crystal
def create_transformer(&block : Int32 -> Int32)
  block  # ส่งคืน Proc
end

add_ten = create_transformer { |x| x + 10 }
puts add_ten.call(5)   # => 15
puts add_ten.call(20)  # => 30

# ใช้กับ map
result = [1, 2, 3].map { |x| add_ten.call(x) }
puts result.inspect  # => [11, 12, 13]
```

### 6.3 การแปลง Proc กลับเป็น Block ด้วย &

```crystal
double = ->(x : Int32) { x * 2 }

# ใช้ & เพื่อแปลง Proc เป็น block
result = [1, 2, 3, 4, 5].map(&double)
puts result.inspect  # => [2, 4, 6, 8, 10]

# เทียบเท่ากับ
result2 = [1, 2, 3, 4, 5].map { |x| double.call(x) }
puts result2.inspect  # => [2, 4, 6, 8, 10]
```

---

## 7. Capturing Blocks - การจับและเก็บ Blocks

### 7.1 การจับ Block สำหรับ Lazy Evaluation

```crystal
class LazyValue
  def initialize(&block : -> Int32)
    @block = block
    @computed = false
    @value = 0
  end

  def value
    unless @computed
      @value = @block.call
      @computed = true
      puts "(คำนวณครั้งแรก)"
    end
    @value
  end
end

expensive = LazyValue.new do
  puts "กำลังคำนวณ..."
  sleep(0.001)  # จำลองการคำนวณที่ใช้เวลา
  42
end

puts "ยังไม่ได้คำนวณ"
puts expensive.value  # คำนวณตอนนี้
puts expensive.value  # ใช้ค่าที่เก็บไว้
```

### 7.2 Callback System

```crystal
class Button
  property label : String

  def initialize(@label : String)
    @click_handlers = [] of Proc(Nil)
  end

  def on_click(&handler)
    @click_handlers << handler
  end

  def click
    puts "คลิก #{@label}"
    @click_handlers.each(&.call)
  end
end

btn = Button.new("ส่งข้อมูล")

btn.on_click { puts "Handler 1: บันทึกข้อมูล..." }
btn.on_click { puts "Handler 2: ส่งอีเมลแจ้งเตือน..." }
btn.on_click { puts "Handler 3: อัปเดต UI..." }

btn.click
# => คลิก ส่งข้อมูล
# => Handler 1: บันทึกข้อมูล...
# => Handler 2: ส่งอีเมลแจ้งเตือน...
# => Handler 3: อัปเดต UI...
```

### 7.3 Middleware Chain

```crystal
class Pipeline
  def initialize
    @steps = [] of Proc(String, String)
  end

  def use(&step : String -> String)
    @steps << step
    self
  end

  def run(input : String) : String
    @steps.reduce(input) { |data, step| step.call(data) }
  end
end

result = Pipeline.new
  .use { |s| s.strip }
  .use { |s| s.downcase }
  .use { |s| s.gsub(" ", "_") }
  .use { |s| "prefix_#{s}" }
  .run("  Hello World  ")

puts result  # => prefix_hello_world
```

---

## 8. Blocks กับ Exception Handling

### 8.1 ensure ด้วย Block

```crystal
def with_file(filename : String)
  puts "เปิดไฟล์ #{filename}"
  begin
    yield filename
  ensure
    puts "ปิดไฟล์ #{filename}"
  end
end

with_file("data.txt") do |f|
  puts "อ่านข้อมูลจาก #{f}"
  # ถ้าเกิด exception ก็จะยัง ensure ทำงาน
end
# => เปิดไฟล์ data.txt
# => อ่านข้อมูลจาก data.txt
# => ปิดไฟล์ data.txt
```

### 8.2 Transaction Pattern

```crystal
class Database
  def transaction
    puts "BEGIN TRANSACTION"
    begin
      yield
      puts "COMMIT"
    rescue ex
      puts "ROLLBACK - #{ex.message}"
      raise ex
    end
  end
end

db = Database.new

db.transaction do
  puts "  INSERT INTO users ..."
  puts "  UPDATE accounts ..."
  puts "  การทำงานสำเร็จ"
end

# เมื่อเกิด error
begin
  db.transaction do
    puts "  INSERT INTO orders ..."
    raise "ข้อมูลไม่ครบถ้วน"
  end
rescue ex
  puts "จัดการ error: #{ex.message}"
end
```

---

## 9. ตัวอย่างขั้นสูง

### 9.1 DSL Builder

```crystal
class HtmlBuilder
  def initialize
    @html = ""
    @indent = 0
  end

  def tag(name : String, **attrs)
    attr_str = attrs.map { |k, v| " #{k}=\"#{v}\"" }.join
    @html += "  " * @indent + "<#{name}#{attr_str}>\n"
    @indent += 1
    yield if block_given?
    @indent -= 1
    @html += "  " * @indent + "</#{name}>\n"
  end

  def text(content : String)
    @html += "  " * @indent + content + "\n"
  end

  def build
    @html
  end
end

builder = HtmlBuilder.new
builder.tag("div", id: "main", class: "container") do
  builder.tag("h1") do
    builder.text("สวัสดีโลก")
  end
  builder.tag("p", class: "intro") do
    builder.text("ยินดีต้อนรับสู่เว็บไซต์ของเรา")
  end
end

puts builder.build
```

### 9.2 Retry ด้วย Block

```crystal
def with_retry(max_attempts : Int32, delay : Float64 = 0.1)
  attempts = 0
  begin
    attempts += 1
    yield attempts
  rescue ex
    if attempts < max_attempts
      puts "ลองใหม่ครั้งที่ #{attempts}/#{max_attempts}: #{ex.message}"
      sleep(delay)
      retry
    else
      puts "เกินจำนวนครั้งที่กำหนด"
      raise ex
    end
  end
end

attempt_count = 0
with_retry(3, 0.01) do |attempt|
  attempt_count = attempt
  puts "กำลังลอง (ครั้งที่ #{attempt})"
  raise "ล้มเหลว" if attempt < 3
  puts "สำเร็จ!"
end
```

### 9.3 Resource Manager

```crystal
class ResourceManager
  @@resources = [] of String

  def self.acquire(name : String)
    @@resources << name
    puts "จัดสรร resource: #{name}"
    begin
      yield name
    ensure
      @@resources.delete(name)
      puts "คืน resource: #{name}"
    end
  end

  def self.active_resources
    @@resources.dup
  end
end

ResourceManager.acquire("database_connection") do |conn|
  puts "ใช้งาน: #{conn}"
  ResourceManager.acquire("file_handle") do |fh|
    puts "ใช้งาน: #{fh}"
    puts "Active: #{ResourceManager.active_resources}"
  end
  puts "Active: #{ResourceManager.active_resources}"
end
puts "Active: #{ResourceManager.active_resources}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง each_slice

```crystal
def each_slice(array : Array(Int32), size : Int32)
  i = 0
  while i < array.size
    slice = array[i, [size, array.size - i].min]
    yield slice
    i += size
  end
end

each_slice([1, 2, 3, 4, 5, 6, 7], 3) do |group|
  puts group.inspect
end
# => [1, 2, 3]
# => [4, 5, 6]
# => [7]
```

### แบบฝึกหัดที่ 2: Memoize ด้วย Block

```crystal
class Memoizer
  def initialize
    @cache = {} of String => Int32
  end

  def memoize(key : String, &block : -> Int32)
    @cache[key] ||= block.call
  end
end

memo = Memoizer.new

# จำลองการคำนวณที่ใช้เวลา
result1 = memo.memoize("fib_10") do
  puts "กำลังคำนวณ fib(10)..."
  55  # fibonacci(10)
end

result2 = memo.memoize("fib_10") do
  puts "กำลังคำนวณอีกครั้ง..."  # จะไม่ถูกเรียก
  55
end

puts result1  # => 55
puts result2  # => 55
```

### แบบฝึกหัดที่ 3: Observer Pattern

```crystal
class Observable
  def initialize
    @observers = {} of Symbol => Array(Proc(Hash(Symbol, String), Nil))
  end

  def subscribe(event : Symbol, &observer : Hash(Symbol, String) ->)
    @observers[event] ||= [] of Proc(Hash(Symbol, String), Nil)
    @observers[event] << observer
  end

  def notify(event : Symbol, data : Hash(Symbol, String))
    @observers[event]?.try &.each { |obs| obs.call(data) }
  end
end

class UserService < Observable
  def create(name : String, email : String)
    # สร้าง user
    user_data = {name: name, email: email}
    notify(:user_created, user_data)
  end
end

service = UserService.new

service.subscribe(:user_created) do |data|
  puts "ส่งอีเมลยินดีต้อนรับไปยัง #{data[:email]}"
end

service.subscribe(:user_created) do |data|
  puts "บันทึก audit log: สร้าง user #{data[:name]}"
end

service.create("สมชาย", "somchai@example.com")
```

---

## สรุป

Blocks และ Yield เป็นหัวใจสำคัญของการเขียนโค้ดแบบ functional ใน Crystal:

| Concept | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Inline block | `{ \|x\| ... }` | โค้ดสั้น บรรทัดเดียว |
| Multi-line block | `do \|x\| ... end` | โค้ดหลายบรรทัด |
| yield | `yield` | เรียกใช้ block |
| yield กับ args | `yield value` | ส่งข้อมูลให้ block |
| block_given? | `if block_given?` | ตรวจสอบว่ามี block |
| capture block | `&block` | เก็บ block เป็น Proc |
| convert to block | `&proc` | แปลง Proc กลับเป็น block |

**ข้อสำคัญ:**
- Block ใน Crystal เป็น compile-time feature ที่มีประสิทธิภาพสูง
- `yield` เร็วกว่าการเรียก `Proc.call` โดยตรง
- ใช้ `&block` เมื่อต้องการเก็บ block สำหรับใช้ภายหลัง
- Blocks จับ (capture) local variables จากสภาพแวดล้อมที่ถูกสร้าง
