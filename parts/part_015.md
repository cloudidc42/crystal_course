# ตอนที่ 15: Break, Next, Return

## บทนำ

Crystal มี keywords สำหรับควบคุม flow ของการทำงาน: `break` สำหรับออกจาก loop, `next` สำหรับข้ามไป iteration ถัดไป, และ `return` สำหรับออกจาก method เหล่านี้ทำให้โปรแกรมยืดหยุ่นและจัดการ edge cases ได้ดีขึ้น

---

## 15.1 Break จาก Loop

`break` ออกจาก loop ทันทีโดยไม่สนใจเงื่อนไข

### Break จาก While

```crystal
# break จาก while loop
i = 0
while true
  puts i
  i += 1
  break if i >= 5  # ออกเมื่อ i ถึง 5
end
# Output: 0, 1, 2, 3, 4

# break แบบ conditional
numbers = [3, 7, 2, 9, 1, 8, 4]
i = 0
while i < numbers.size
  current = numbers[i]
  if current > 8
    puts "พบเลข > 8 ที่ index #{i}: #{current}"
    break
  end
  i += 1
end
```

### Break จาก Each

```crystal
# break จาก each
[1, 2, 3, 4, 5].each do |n|
  break if n > 3
  puts n
end
# Output: 1, 2, 3

# หาตัวแรกที่เงื่อนไขตรง
target = nil
[10, 25, 3, 47, 8].each do |n|
  if n > 40
    target = n
    break
  end
end
puts "พบ: #{target}"  # => 47
```

### Break จาก loop do

```crystal
count = 0
loop do
  count += 1
  puts "iteration #{count}"
  break if count >= 3
end
# Output:
# iteration 1
# iteration 2
# iteration 3
```

---

## 15.2 Break กับค่า (Break Value)

`break` สามารถส่งค่ากลับได้เมื่อออกจาก loop

```crystal
# break ส่งค่ากลับ
result = (1..100).each do |n|
  break n if n * n > 50
end
puts "n แรกที่ n² > 50: #{result}"  # => 8 (เพราะ 8²=64 > 50)

# loop do กับ break value
random_prime = loop do
  n = rand(2..100)
  is_prime = (2..Math.sqrt(n.to_f).to_i).none? { |i| n % i == 0 }
  break n if is_prime
end
puts "จำนวนเฉพาะสุ่ม: #{random_prime}"

# while กับ break value
found = while true
  x = rand(10)
  break x if x > 7
end
puts "ได้ค่า > 7: #{found}"
```

### ใช้ break value ในการค้นหา

```crystal
# ค้นหาใน nested structure
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]

# หา coordinates ของค่าที่ต้องการ
target = 6
found_pos = matrix.each_with_index do |row, i|
  row.each_with_index do |val, j|
    break {row: i, col: j} if val == target
  end
end

puts found_pos.inspect  # => {row: 1, col: 2}
```

---

## 15.3 Next (Skip Iteration)

`next` ข้ามไปยัง iteration ถัดไป ทำงานคล้าย `continue` ในภาษา C/Java

```crystal
# ข้ามเลขคู่
(1..10).each do |n|
  next if n.even?
  puts n
end
# Output: 1, 3, 5, 7, 9

# ข้ามค่าที่ไม่ต้องการ
words = ["hello", "", "world", "  ", "crystal", nil].compact
words.each do |word|
  next if word.strip.empty?
  puts word.upcase
end
# Output: HELLO, WORLD, CRYSTAL

# next ใน while
i = 0
while i < 10
  i += 1
  next if i % 3 == 0  # ข้ามทวีคูณของ 3
  puts i
end
# Output: 1, 2, 4, 5, 7, 8, 10
```

---

## 15.4 Next ใน Block

```crystal
# next ออกจาก block และ return ค่า
results = (1..5).map do |n|
  next 0 if n.even?  # คืน 0 สำหรับเลขคู่
  n * n              # ยกกำลังสองสำหรับเลขคี่
end
puts results.inspect  # => [1, 0, 9, 0, 25]

# next ใน select
data = [1, nil, 2, nil, 3, nil, 4]
cleaned = data.each_with_object([] of Int32) do |item, arr|
  next if item.nil?
  arr << item
end
puts cleaned.inspect  # => [1, 2, 3, 4]

# การ filter ที่ซับซ้อน
records = [
  {name: "Alice", age: 25, active: true},
  {name: "Bob", age: 17, active: true},
  {name: "Charlie", age: 30, active: false},
  {name: "Diana", age: 28, active: true},
]

valid_users = records.each_with_object([] of String) do |record, names|
  next unless record[:active]
  next if record[:age] < 18
  names << record[:name]
end
puts valid_users.inspect  # => ["Alice", "Diana"]
```

---

## 15.5 Return จาก Method

`return` ออกจาก method และส่งค่ากลับ

```crystal
# return พื้นฐาน
def find_first_even(numbers : Array(Int32)) : Int32?
  numbers.each do |n|
    return n if n.even?  # return จาก method ทันที
  end
  nil  # ไม่พบ
end

puts find_first_even([1, 3, 5, 4, 7]).inspect  # => 4
puts find_first_even([1, 3, 5, 7]).inspect      # => nil

# implicit return: ค่าสุดท้ายเป็น return value
def double(n : Int32) : Int32
  n * 2  # คืนค่านี้อัตโนมัติ
end

puts double(5)  # => 10
```

---

## 15.6 Return กับ Multiple Values (Tuple)

```crystal
# return หลายค่าด้วย Tuple
def min_max(numbers : Array(Int32)) : {Int32, Int32}
  return {0, 0} if numbers.empty?
  {numbers.min, numbers.max}
end

min, max = min_max([3, 1, 4, 1, 5, 9, 2, 6])
puts "Min: #{min}, Max: #{max}"  # => Min: 1, Max: 9

# return NamedTuple
def analyze(text : String) : {words: Int32, chars: Int32, avg_word_len: Float64}
  words = text.split.size
  chars = text.gsub(/\s+/, "").size
  avg = words > 0 ? chars.to_f / words : 0.0
  {words: words, chars: chars, avg_word_len: avg.round(2)}
end

result = analyze("Hello world from Crystal")
puts "คำ: #{result[:words]}, ตัวอักษร: #{result[:chars]}, เฉลี่ย: #{result[:avg_word_len]}"
```

---

## 15.7 Early Return Pattern

Early return เพื่อ guard clause และลด nesting

```crystal
def process_payment(amount : Float64, card_number : String?, balance : Float64) : String
  # Guard: ตรวจสอบ card
  return "ไม่ระบุบัตร" if card_number.nil?
  return "หมายเลขบัตรไม่ถูกต้อง" unless card_number.size == 16
  
  # Guard: ตรวจสอบ amount
  return "จำนวนเงินต้องเป็นบวก" if amount <= 0
  return "จำนวนเงินเกิน limit" if amount > 100_000
  
  # Guard: ตรวจสอบ balance
  return "ยอดเงินไม่เพียงพอ" if amount > balance
  
  # ทุกอย่าง OK
  "ชำระเงิน #{amount} บาท สำเร็จ"
end

puts process_payment(500.0, nil, 1000.0)                      # => ไม่ระบุบัตร
puts process_payment(500.0, "1234567890123", 1000.0)           # => หมายเลขบัตรไม่ถูกต้อง
puts process_payment(-100.0, "1234567890123456", 1000.0)       # => จำนวนเงินต้องเป็นบวก
puts process_payment(2000.0, "1234567890123456", 1000.0)       # => ยอดเงินไม่เพียงพอ
puts process_payment(500.0, "1234567890123456", 1000.0)        # => ชำระเงิน 500.0 บาท สำเร็จ
```

---

## 15.8 Next vs Break ความแตกต่าง

```crystal
# Break: ออกจาก loop ทั้งหมด
puts "--- Break ---"
(1..5).each do |i|
  break if i == 3
  puts i
end
puts "หลัง loop"
# Output:
# 1
# 2
# หลัง loop

# Next: ข้ามไป iteration ถัดไป
puts "\n--- Next ---"
(1..5).each do |i|
  next if i == 3
  puts i
end
puts "หลัง loop"
# Output:
# 1
# 2
# 4
# 5
# หลัง loop
```

---

## 15.9 Nested Loop Control

Crystal ไม่มี labeled break แบบ Java แต่มีวิธีแก้ไข

```crystal
# ปัญหา: ต้องการ break จาก nested loop
# วิธีที่ 1: ใช้ flag
found = false
outer_result = -1
inner_result = -1

matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
matrix.each_with_index do |row, i|
  row.each_with_index do |val, j|
    if val == 5
      outer_result = i
      inner_result = j
      found = true
      break
    end
  end
  break if found
end

puts "พบ 5 ที่ [#{outer_result}][#{inner_result}]"  # => [1][1]

# วิธีที่ 2: ใช้ method
def find_in_matrix(matrix : Array(Array(Int32)), target : Int32) : {Int32, Int32}?
  matrix.each_with_index do |row, i|
    row.each_with_index do |val, j|
      return {i, j} if val == target  # return จาก method ทั้งหมด
    end
  end
  nil
end

pos = find_in_matrix(matrix, 5)
puts "พบที่: #{pos.inspect}"  # => {1, 1}

# วิธีที่ 3: ใช้ catch/throw (สำหรับ advanced cases)
result = catch(:found) do
  matrix.each_with_index do |row, i|
    row.each_with_index do |val, j|
      throw :found, {i, j} if val == 8
    end
  end
  nil
end
puts "พบ 8 ที่: #{result.inspect}"  # => {2, 1}
```

---

## 15.10 Return ใน Block vs Method

```crystal
# return ใน block: return จาก method ที่ครอบอยู่
def find_positive(numbers : Array(Int32)) : Int32?
  numbers.each do |n|
    return n if n > 0  # return จาก find_positive ไม่ใช่จาก each
  end
  nil
end

puts find_positive([-1, -2, 3, 4]).inspect  # => 3

# Proc ที่ return จาก proc เท่านั้น
counter = Proc(Int32, Bool).new do |n|
  next true if n > 5  # next ใน Proc = ออกจาก Proc
  false
end

puts counter.call(10)  # => true
puts counter.call(3)   # => false
```

---

## 15.11 ตัวอย่างจริง: Search Algorithms

```crystal
# Linear Search กับ early return
def linear_search(arr : Array(Int32), target : Int32) : Int32?
  arr.each_with_index do |val, i|
    return i if val == target
  end
  nil
end

# Binary Search กับ break
def binary_search_iterative(arr : Array(Int32), target : Int32) : Int32?
  low = 0
  high = arr.size - 1
  result = nil

  while low <= high
    mid = (low + high) // 2
    if arr[mid] == target
      result = mid
      break
    elsif arr[mid] < target
      low = mid + 1
    else
      high = mid - 1
    end
  end

  result
end

data = (1..20).to_a
puts linear_search(data, 15).inspect   # => 14
puts binary_search_iterative(data, 15).inspect  # => 14
puts linear_search(data, 25).inspect   # => nil
```

---

## 15.12 ตัวอย่างจริง: Data Validator

```crystal
class FormValidator
  alias Rule = Proc(String, String?)

  property rules : Hash(String, Array(Rule))
  property data : Hash(String, String)

  def initialize(@data : Hash(String, String))
    @rules = {} of String => Array(Rule)
  end

  def required(field : String) : self
    add_rule(field) do |value|
      value.strip.empty? ? "#{field} จำเป็นต้องกรอก" : nil
    end
    self
  end

  def min_length(field : String, min : Int32) : self
    add_rule(field) do |value|
      value.size < min ? "#{field} ต้องมีอย่างน้อย #{min} ตัวอักษร" : nil
    end
    self
  end

  def max_length(field : String, max : Int32) : self
    add_rule(field) do |value|
      value.size > max ? "#{field} ต้องไม่เกิน #{max} ตัวอักษร" : nil
    end
    self
  end

  def format(field : String, pattern : Regex, message : String) : self
    add_rule(field) do |value|
      value.matches?(pattern) ? nil : message
    end
    self
  end

  def validate : Hash(String, Array(String))
    errors = {} of String => Array(String)

    @rules.each do |field, field_rules|
      value = @data[field]? || ""
      field_errors = [] of String

      field_rules.each do |rule|
        error = rule.call(value)
        next if error.nil?  # ผ่าน rule นี้ ไปต่อ
        field_errors << error
        break  # พบ error แรก หยุดตรวจสอบ field นี้
      end

      errors[field] = field_errors unless field_errors.empty?
    end

    errors
  end

  private def add_rule(field : String, &block : String -> String?) : Void
    @rules[field] ||= [] of Rule
    @rules[field] << Rule.new { |v| block.call(v) }
  end
end

# ทดสอบ
data = {
  "username" => "al",
  "email"    => "not-email",
  "password" => "12345",
}

validator = FormValidator.new(data)
  .required("username")
  .min_length("username", 3)
  .max_length("username", 20)
  .required("email")
  .format("email", /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i, "email รูปแบบไม่ถูกต้อง")
  .required("password")
  .min_length("password", 8)

errors = validator.validate

if errors.empty?
  puts "ข้อมูลถูกต้องทั้งหมด!"
else
  puts "พบข้อผิดพลาด:"
  errors.each do |field, msgs|
    msgs.each { |msg| puts "  - #{msg}" }
  end
end

# Output:
# พบข้อผิดพลาด:
#   - username ต้องมีอย่างน้อย 3 ตัวอักษร
#   - email รูปแบบไม่ถูกต้อง
#   - password ต้องมีอย่างน้อย 8 ตัวอักษร
```

---

## 15.13 ตัวอย่างจริง: Pipeline กับ Early Return

```crystal
struct Pipeline(T)
  @value : T
  @errors : Array(String)

  def initialize(@value : T)
    @errors = [] of String
  end

  def self.of(value : T) : self
    new(value)
  end

  def transform(&block : T -> T) : self
    return self unless @errors.empty?  # หยุดถ้ามี error
    @value = block.call(@value)
    self
  rescue ex
    @errors << ex.message.to_s
    self
  end

  def validate(&block : T -> String?) : self
    return self unless @errors.empty?
    if error = block.call(@value)
      @errors << error
    end
    self
  end

  def result : T?
    @errors.empty? ? @value : nil
  end

  def errors : Array(String)
    @errors
  end
end

# ใช้ Pipeline
def process_number(input : String)
  result = Pipeline.of(input)
    .validate { |s| s.empty? ? "ต้องไม่ว่างเปล่า" : nil }
    .transform { |s| s.strip }
    .validate { |s| s.to_i? ? nil : "ต้องเป็นตัวเลข" }
    .transform { |s| (s.to_i * 2).to_s }
    .validate { |s| s.to_i > 0 ? nil : "ผลลัพธ์ต้องเป็นบวก" }

  if result.errors.empty?
    puts "ผลลัพธ์: #{result.result}"
  else
    puts "Error: #{result.errors.join(", ")}"
  end
end

process_number("21")   # => ผลลัพธ์: 42
process_number("")     # => Error: ต้องไม่ว่างเปล่า
process_number("abc")  # => Error: ต้องเป็นตัวเลข
```

---

## 15.14 ตัวอย่างจริง: Iterator กับ Break/Next

```crystal
# Custom iterator class
class NumberIterator
  include Iterator(Int32)

  def initialize(@current : Int32 = 0, @max : Int32 = 10)
  end

  def next : Int32 | Iterator::Stop
    return stop if @current >= @max
    value = @current
    @current += 1
    value
  end
end

# ใช้ custom iterator
iter = NumberIterator.new(1, 20)

# เอาเฉพาะเลขคี่ที่น้อยกว่า 10
iter
  .select { |n| n.odd? }
  .select { |n| n < 10 }
  .each { |n| puts n }
# Output: 1, 3, 5, 7, 9

# Fibonacci iterator
class FibIterator
  include Iterator(Int64)

  def initialize(@limit : Int64)
    @a = 0_i64
    @b = 1_i64
  end

  def next : Int64 | Iterator::Stop
    return stop if @a > @limit
    value = @a
    @a, @b = @b, @a + @b
    value
  end
end

# เอา Fibonacci ที่เป็นเลขคู่
FibIterator.new(1000)
  .select { |n| n.even? }
  .to_a
  .tap { |arr| puts "Fibonacci คู่ <= 1000: #{arr.inspect}" }
# => [0, 2, 8, 34, 144, 610]
```

---

## 15.15 ตัวอย่างโปรแกรมจริง: Text Processor

```crystal
class TextProcessor
  getter text : String

  def initialize(@text : String)
  end

  # ประมวลผลแต่ละ word ด้วย block
  # ถ้า block คืน nil ให้ข้าม word นั้น
  def process_words(&block : String -> String?) : String
    result = [] of String

    @text.split.each do |word|
      processed = block.call(word)
      next if processed.nil?
      result << processed
    end

    result.join(" ")
  end

  # หา word แรกที่ผ่าน condition
  def find_word(&block : String -> Bool) : String?
    @text.split.each do |word|
      return word if block.call(word)
    end
    nil
  end

  # รวบรวม words จนกว่าจะพบ stop word
  def words_until(stop_word : String) : Array(String)
    result = [] of String

    @text.split.each do |word|
      break if word.downcase == stop_word.downcase
      result << word
    end

    result
  end

  # ประมวลผล n words แรก
  def first_n_words(n : Int32, &block : String -> Void) : Void
    count = 0
    @text.split.each do |word|
      break if count >= n
      block.call(word)
      count += 1
    end
  end
end

text = TextProcessor.new(
  "the quick brown fox STOP jumps over the lazy dog"
)

# กรอง stop words และแปลง
stop_words = ["the", "a", "an", "of", "in"]
filtered = text.process_words do |word|
  lower = word.downcase
  stop_words.includes?(lower) ? nil : word.upcase
end
puts "Filtered: #{filtered}"

# หา word แรกที่เป็นตัวพิมพ์ใหญ่ทั้งหมด
first_upper = text.find_word { |w| w == w.upcase && w.size > 1 }
puts "First ALL CAPS: #{first_upper.inspect}"  # => "STOP"

# รวบรวม words ก่อน STOP
before_stop = text.words_until("STOP")
puts "Before STOP: #{before_stop.join(" ")}"  # => "the quick brown fox"

# ประมวลผล 3 words แรก
print "First 3: "
text.first_n_words(3) { |w| print "#{w} " }
puts
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Break with Value
```crystal
# ใช้ break value เพื่อหา:
# 1. ผลรวม Fibonacci ที่น้อยกว่า 1000 (เอาผลรวมออกมา)
# 2. จำนวนเฉพาะที่ 100th
# 3. หมายเลขที่ palindrome แรกที่มากกว่า 1000

def sum_fibonacci_under(limit : Int32) : Int64
  # TODO: ใช้ loop do และ break ที่มีค่า
end
```

### แบบฝึกหัดที่ 2: Next Filter
```crystal
# สร้าง method ที่ใช้ next เพื่อ:
# - รับ Array(String)
# - ข้ามค่าที่ empty, nil, หรือมีแต่ whitespace
# - ข้ามค่าที่มีตัวเลขทั้งหมด
# - แปลงที่เหลือเป็น Title Case

def clean_names(names : Array(String?)) : Array(String)
  # TODO: ใช้ each_with_object กับ next
end
```

### แบบฝึกหัดที่ 3: Nested Loop Control
```crystal
# หา Pythagorean triples (a² + b² = c²) ที่ a + b + c = 1000
# ใช้ return เมื่อพบ

def find_pythagorean_triple(sum : Int32) : {Int32, Int32, Int32}?
  # TODO: ใช้ nested loop และ return เมื่อพบ
  # a <= b <= c
  # a + b + c = sum
  # a² + b² = c²
end

result = find_pythagorean_triple(1000)
puts result.inspect  # => {200, 375, 425}
```

### แบบฝึกหัดที่ 4: State Machine
```crystal
# สร้าง Traffic Light state machine
# ใช้ loop do กับ break เมื่อผ่านครบ 3 สถานะ
enum TrafficLight
  Red
  Yellow
  Green
end

def simulate_traffic_light(cycles : Int32) : Array(TrafficLight)
  # TODO: loop และ collect states
  # Red -> Green -> Yellow -> Red ...
  # break หลัง n cycles (1 cycle = Red+Green+Yellow)
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **break**: ออกจาก loop ทันที
   - `break` ธรรมดา
   - `break value` ส่งค่ากลับ
   - ใช้กับ while, until, loop do, each

2. **next**: ข้ามไป iteration ถัดไป
   - `next` ธรรมดา
   - `next value` ส่งค่าให้ accumulator (ใน each_with_object)
   - ใช้เหมือน `continue` ในภาษาอื่น

3. **return**: ออกจาก method
   - `return` ไม่มีค่า
   - `return value` ส่งค่ากลับ
   - `return a, b` คืน Tuple
   - Early return pattern สำหรับ guard clauses

### เปรียบเทียบ break, next, return

| Keyword | ออกจาก | ค่าที่ส่งกลับ |
|---------|--------|--------------|
| `break` | loop ปัจจุบัน | ค่าของ loop expression |
| `next` | iteration ปัจจุบัน | ค่าให้ block (บางกรณี) |
| `return` | method ทั้งหมด | ค่า return ของ method |

### เมื่อใดใช้อะไร

- **break**: เมื่อพบสิ่งที่ต้องการแล้ว ไม่ต้องดูต่อ
- **next**: เมื่อ element นี้ไม่ตรงเงื่อนไข ข้ามไปตัวถัดไป
- **return**: เมื่อ method ทำงานเสร็จหรือพบ error

การใช้ break, next, return อย่างเหมาะสมทำให้โค้ดอ่านง่าย มี intent ที่ชัดเจน และ ลด nesting ที่ไม่จำเป็น!
