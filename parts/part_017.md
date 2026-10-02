# Part 017: Loop Control Flow ขั้นสูง

## บทนำ

Crystal มีเครื่องมือสำหรับการทำ loop ที่ทรงพลังมากกว่า `while` และ `loop` ธรรมดา ในบทนี้เราจะสำรวจเทคนิค loop ขั้นสูง ตั้งแต่การติดตาม index, nested loops ที่ซับซ้อน, ไปจนถึง lazy evaluation และ generators pattern

---

## 1. Loop พื้นฐานพร้อม Index Tracking

### each_with_index

```crystal
fruits = ["แอปเปิ้ล", "กล้วย", "ส้ม", "มะม่วง"]

fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end
# Output:
# 1. แอปเปิ้ล
# 2. กล้วย
# 3. ส้ม
# 4. มะม่วง
```

### each_with_index พร้อม offset

```crystal
# เริ่ม index จาก 1 แทน 0
fruits = ["แอปเปิ้ล", "กล้วย", "ส้ม"]
fruits.each_with_index(1) do |fruit, index|
  puts "รายการที่ #{index}: #{fruit}"
end
```

### each_with_object

```crystal
# สะสมผลลัพธ์ระหว่าง loop
numbers = [1, 2, 3, 4, 5]
result = numbers.each_with_object({} of Int32 => Int32) do |n, hash|
  hash[n] = n * n
end
puts result  # => {1 => 1, 2 => 4, 3 => 9, 4 => 16, 5 => 25}
```

### loop พร้อม manual index

```crystal
# ควบคุม index เอง
index = 0
data = [10, 20, 30, 40, 50]

while index < data.size
  puts "data[#{index}] = #{data[index]}"
  index += 2  # ข้ามทีละ 2
end
# Output:
# data[0] = 10
# data[2] = 30
# data[4] = 50
```

---

## 2. Nested Loops

### Nested loops พื้นฐาน

```crystal
# ตาราง multiplication
(1..5).each do |i|
  (1..5).each do |j|
    print "#{(i * j).to_s.rjust(4)}"
  end
  puts
end
```

### Nested loops กับ break/next

```crystal
# ค้นหาใน 2D array
matrix = [
  [1, 2, 3],
  [4, 5, 6],
  [7, 8, 9]
]

target = 5
found = false

matrix.each_with_index do |row, i|
  row.each_with_index do |val, j|
    if val == target
      puts "พบ #{target} ที่ตำแหน่ง [#{i}][#{j}]"
      found = true
      break  # break เฉพาะ inner loop
    end
  end
  break if found  # break outer loop
end
```

### ออกจาก Nested Loops ด้วย Exception

```crystal
# วิธีหนึ่งที่ใช้ break ออกจาก nested loops
class BreakNested < Exception; end

matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
target = 5
found_pos = {0, 0}

begin
  matrix.each_with_index do |row, i|
    row.each_with_index do |val, j|
      if val == target
        found_pos = {i, j}
        raise BreakNested.new
      end
    end
  end
rescue BreakNested
  row, col = found_pos
  puts "พบ #{target} ที่ row=#{row}, col=#{col}"
end
```

### Nested Loops สร้าง Pattern

```crystal
# สร้างรูปสามเหลี่ยม
def triangle(n : Int32)
  (1..n).each do |i|
    (1..i).each { print "* " }
    puts
  end
end

triangle(5)
# *
# * *
# * * *
# * * * *
# * * * * *

# สร้างรูปสี่เหลี่ยมกลวง
def hollow_square(n : Int32)
  n.times do |i|
    n.times do |j|
      if i == 0 || i == n - 1 || j == 0 || j == n - 1
        print "* "
      else
        print "  "
      end
    end
    puts
  end
end

hollow_square(5)
```

---

## 3. Loop Guards

Loop guard คือเงื่อนไขที่กำหนดว่า loop จะดำเนินต่อหรือไม่:

### next (skip iteration)

```crystal
# ข้ามค่าที่เป็นเลขคู่
(1..10).each do |n|
  next if n.even?  # guard: ข้ามถ้าเป็นเลขคู่
  print "#{n} "
end
puts
# Output: 1 3 5 7 9
```

### break (หยุด loop)

```crystal
# หยุดเมื่อพบค่าที่ต้องการ
data = [3, 7, 2, 8, 1, 9, 4]
sum = 0

data.each do |n|
  break if n > 7  # guard: หยุดถ้าค่ามากกว่า 7
  sum += n
  puts "เพิ่ม #{n}, sum = #{sum}"
end
```

### break with value

```crystal
# break สามารถส่งค่ากลับได้
result = [1, 2, 3, 4, 5].each do |n|
  break n * 10 if n == 3  # ส่งค่า 30 กลับ
end

puts result.inspect  # => 30

# ถ้าไม่ break จะได้ nil
result2 = [1, 2, 3].each do |n|
  puts n
end
puts result2.inspect  # => nil
```

### Complex Guards

```crystal
users = [
  {name: "Alice", age: 30, active: true},
  {name: "Bob", age: 17, active: true},
  {name: "Charlie", age: 25, active: false},
  {name: "Diana", age: 28, active: true},
]

puts "ผู้ใช้ที่ active และอายุมากกว่า 18:"
users.each do |user|
  next unless user[:active]         # guard 1: ต้อง active
  next if user[:age] < 18          # guard 2: ต้องอายุ >= 18
  puts "  #{user[:name]} (#{user[:age]})"
end
```

---

## 4. Lazy Evaluation

Lazy evaluation หมายถึงการคำนวณเมื่อต้องการใช้จริงๆ เท่านั้น ช่วยประหยัดหน่วยความจำและเวลาสำหรับ collection ขนาดใหญ่หรือ infinite sequences:

### lazy basics

```crystal
# โดยไม่ใช้ lazy - สร้าง array ทั้งหมดก่อน
result = (1..Float::INFINITY).to_a.first(10)  # !! อันตราย - infinite!

# ใช้ lazy - คำนวณทีละตัวเมื่อต้องการ
result = (1..Float::INFINITY).lazy.first(10)
puts result.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
```

### lazy.select และ lazy.map

```crystal
# หาตัวเลขคู่ 10 ตัวแรกจาก infinite sequence
even_numbers = (1..Float::INFINITY).lazy
  .select { |n| n.even? }
  .first(10)
puts even_numbers.inspect
# => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

# หาตัวเลขที่หารด้วย 3 ได้และกำลังสองมากกว่า 100
result = (1..Float::INFINITY).lazy
  .select { |n| n % 3 == 0 }
  .map { |n| n * n }
  .select { |n| n > 100 }
  .first(5)
puts result.inspect
# => [144, 225, 324, 441, 576]
```

### Lazy กับ Chaining

```crystal
words = ["hello", "world", "crystal", "programming", "language", "fun"]

result = words.lazy
  .select { |w| w.size > 4 }     # คัดกรองคำยาวกว่า 4 ตัว
  .map { |w| w.upcase }           # แปลงเป็นตัวพิมพ์ใหญ่
  .reject { |w| w.includes?("A") } # ตัดที่มีตัว A ออก
  .first(3)

puts result.inspect
```

### Iterator Pattern

```crystal
# สร้าง custom iterator
class CountUp
  include Iterator(Int32)
  
  def initialize(@start : Int32, @step : Int32 = 1)
    @current = @start
  end
  
  def next
    value = @current
    @current += @step
    value
  end
end

# ใช้งาน
counter = CountUp.new(0, 5)
10.times { print "#{counter.next} " }
puts
# Output: 0 5 10 15 20 25 30 35 40 45
```

---

## 5. Loop with State

การเก็บ state ระหว่าง loop:

### Accumulator Pattern

```crystal
# คำนวณสถิติระหว่าง loop
numbers = [4, 8, 15, 16, 23, 42]
count = 0
sum = 0
min = numbers.first
max = numbers.first

numbers.each do |n|
  count += 1
  sum += n
  min = n if n < min
  max = n if n > max
end

avg = sum.to_f / count
puts "Count: #{count}"
puts "Sum: #{sum}"
puts "Min: #{min}, Max: #{max}"
puts "Avg: #{avg.round(2)}"
```

### State Machine ใน Loop

```crystal
# จำลอง traffic light
states = [:red, :green, :yellow]
current = 0
durations = {red: 30, green: 25, yellow: 5}

5.times do |tick|
  state = states[current]
  puts "Tick #{tick}: สัญญาณ#{state} (#{durations[state]} วินาที)"
  current = (current + 1) % states.size
end
```

### Running Average

```crystal
# คำนวณค่าเฉลี่ยแบบ running (สำหรับ streaming data)
class RunningAverage
  getter count : Int32 = 0
  getter average : Float64 = 0.0
  
  def add(value : Float64)
    @count += 1
    @average += (value - @average) / @count
  end
end

avg = RunningAverage.new
[10.0, 20.0, 30.0, 40.0, 50.0].each do |val|
  avg.add(val)
  puts "หลังเพิ่ม #{val}: ค่าเฉลี่ย = #{avg.average.round(2)}"
end
```

---

## 6. Generators Pattern

Generator คือ object ที่สร้างค่าทีละตัวเมื่อถูกเรียก:

### Fibonacci Generator

```crystal
class FibonacciGenerator
  include Iterator(Int64)
  
  def initialize
    @a = 0_i64
    @b = 1_i64
  end
  
  def next : Int64
    value = @a
    @a, @b = @b, @a + @b
    value
  end
end

fib = FibonacciGenerator.new

# สร้าง Fibonacci 10 ตัวแรก
fibs = [] of Int64
10.times { fibs << fib.next }
puts fibs.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# หา Fibonacci ที่น้อยกว่า 100
fib2 = FibonacciGenerator.new
result = [] of Int64
loop do
  val = fib2.next
  break if val >= 100
  result << val
end
puts result.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]
```

### Prime Number Generator

```crystal
class PrimeGenerator
  include Iterator(Int32)
  
  def initialize
    @current = 2
  end
  
  def next : Int32
    while !prime?(@current)
      @current += 1
    end
    result = @current
    @current += 1
    result
  end
  
  private def prime?(n : Int32) : Bool
    return false if n < 2
    return true if n == 2
    return false if n.even?
    
    (3..Math.sqrt(n.to_f).to_i).step(2).each do |i|
      return false if n % i == 0
    end
    true
  end
end

primes = PrimeGenerator.new
first_10 = Array.new(10) { primes.next }
puts "จำนวนเฉพาะ 10 ตัวแรก: #{first_10.inspect}"
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

### Range Generator แบบ Custom

```crystal
class StepGenerator
  include Iterator(Float64)
  
  def initialize(@start : Float64, @stop : Float64, @step : Float64)
    @current = @start
  end
  
  def next : Float64 | Iterator::Stop
    return stop if @current > @stop
    value = @current
    @current += @step
    value
  end
end

gen = StepGenerator.new(0.0, 1.0, 0.25)
gen.each { |v| print "#{v} " }
puts
# Output: 0.0 0.25 0.5 0.75 1.0
```

---

## 7. Enumerable Chaining

Crystal's Enumerable module มีเมธอดมากมายที่ chain ต่อกันได้:

### Method Chaining พื้นฐาน

```crystal
data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

result = data
  .select(&.odd?)        # [1, 3, 5, 7, 9]
  .map { |n| n ** 2 }    # [1, 9, 25, 49, 81]
  .reject { |n| n > 50 } # [1, 9, 25, 49]
  .sum                   # 84

puts result  # => 84
```

### group_by และ Chaining

```crystal
words = ["apple", "ant", "banana", "ball", "cherry", "cat"]

# จัดกลุ่มตามตัวอักษรแรก แล้ว sort ในแต่ละกลุ่ม
grouped = words
  .group_by { |w| w[0] }
  .transform_values { |arr| arr.sort }

grouped.each do |letter, words_in_group|
  puts "#{letter}: #{words_in_group.join(", ")}"
end
```

### flat_map

```crystal
# map แล้ว flatten
sentences = ["hello world", "crystal is fast", "bye now"]

words = sentences.flat_map { |s| s.split(" ") }
puts words.inspect
# => ["hello", "world", "crystal", "is", "fast", "bye", "now"]

# ตัวอย่างกับ nested arrays
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = matrix.flat_map { |row| row.map { |n| n * 2 } }
puts flat.inspect
# => [2, 4, 6, 8, 10, 12, 14, 16, 18]
```

### zip และ Chaining

```crystal
names = ["Alice", "Bob", "Charlie"]
scores = [95, 87, 92]
grades = ["A", "B+", "A-"]

# zip รวม arrays
result = names.zip(scores, grades)
  .sort_by { |item| -item[1] }  # เรียงตาม score มากไปน้อย
  .map { |name, score, grade| "#{name}: #{score} (#{grade})" }

result.each { |line| puts line }
# Output (sorted by score):
# Alice: 95 (A)
# Charlie: 92 (A-)
# Bob: 87 (B+)
```

### each_slice และ each_cons

```crystal
data = (1..12).to_a

puts "=== each_slice(4) - ตัดเป็นก้อนๆ ==="
data.each_slice(4) do |slice|
  puts slice.inspect
end
# [1, 2, 3, 4]
# [5, 6, 7, 8]
# [9, 10, 11, 12]

puts "\n=== each_cons(3) - sliding window ==="
data.each_cons(3) do |window|
  print window.inspect + " "
end
puts
# [1, 2, 3] [2, 3, 4] [3, 4, 5] ...
```

### tally และ group counting

```crystal
votes = ["Alice", "Bob", "Alice", "Charlie", "Bob", "Alice", "Bob"]

tally = votes.tally
puts "ผลการนับคะแนน:"
tally.to_a
  .sort_by { |name, count| -count }
  .each { |name, count| puts "  #{name}: #{count} คะแนน" }
```

---

## 8. Advanced Loop Patterns

### Loop with sliding window

```crystal
def moving_average(data : Array(Float64), window_size : Int32) : Array(Float64)
  averages = [] of Float64
  
  data.each_cons(window_size) do |window|
    averages << window.sum / window.size
  end
  
  averages
end

prices = [10.0, 12.0, 9.0, 11.0, 13.0, 15.0, 14.0, 16.0]
avgs = moving_average(prices, 3)
avgs.each_with_index do |avg, i|
  puts "ช่วง #{i+1}-#{i+3}: #{avg.round(2)}"
end
```

### Parallel Iteration

```crystal
# วนหลาย arrays พร้อมกัน
a = [1, 2, 3, 4, 5]
b = [10, 20, 30, 40, 50]
c = [100, 200, 300, 400, 500]

a.zip(b, c).each_with_index do |(x, y, z), i|
  puts "index #{i}: #{x} + #{y} + #{z} = #{x + y + z}"
end
```

### Loop ที่สร้างตัวเองจาก index

```crystal
# สร้าง array ของตัวเลขที่น่าสนใจ
result = Array.new(10) { |i| (i + 1) ** 2 }
puts result.inspect
# => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]

# สร้าง hash จาก index
lookup = Hash(Int32, String).new
(1..5).each do |i|
  lookup[i] = "item_#{i}"
end
puts lookup.inspect
```

### Loop พร้อม Early Termination และ Result

```crystal
# ค้นหาแบบ binary search
def binary_search(arr : Array(Int32), target : Int32) : Int32?
  low = 0
  high = arr.size - 1
  
  while low <= high
    mid = (low + high) / 2
    case arr[mid] <=> target
    when 0
      return mid     # พบแล้ว!
    when -1
      low = mid + 1  # ค้นหาครึ่งขวา
    when 1
      high = mid - 1 # ค้นหาครึ่งซ้าย
    end
  end
  
  nil  # ไม่พบ
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
puts binary_search(sorted, 11).inspect  # => 5
puts binary_search(sorted, 6).inspect   # => nil
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Nested Loop Matrix

สร้างฟังก์ชันที่:
1. สร้าง matrix ขนาด n×n
2. เติมด้วยตัวเลข 1 ถึง n²
3. แสดงผลแบบเรียงสวยงาม

```crystal
def create_and_display_matrix(n : Int32)
  # TODO: สร้าง matrix และแสดงผล
end

create_and_display_matrix(4)
# Expected:
#  1  2  3  4
#  5  6  7  8
#  9 10 11 12
# 13 14 15 16
```

### แบบฝึกหัดที่ 2: Word Frequency Counter

```crystal
text = "the quick brown fox jumps over the lazy dog the fox"

# TODO: นับความถี่ของแต่ละคำ
# แสดงผลเรียงจากมากไปน้อย
# Expected:
# the: 3
# fox: 2
# quick: 1
# ...
```

### แบบฝึกหัดที่ 3: Lazy Pipeline

สร้าง lazy pipeline ที่:
1. Generate ตัวเลข 1 ถึง infinity
2. กรองเฉพาะตัวเลขที่หารด้วย 7 ลงตัว
3. คำนวณ digit sum ของแต่ละตัว
4. กรองเฉพาะที่ digit sum เป็นเลขคี่
5. เอา 10 ตัวแรก

```crystal
def digit_sum(n : Int32) : Int32
  n.to_s.chars.sum { |c| c.to_i }
end

result = (1..Float::INFINITY).lazy
  # TODO: เพิ่ม filter และ map steps
  .first(10)

puts result.inspect
```

### แบบฝึกหัดที่ 4: Custom Fibonacci with State

สร้าง class ที่:
1. Generate Fibonacci sequence
2. รองรับ `reset` เพื่อเริ่มใหม่
3. รองรับ `skip(n)` เพื่อข้ามไป n ตัว
4. รองรับ `peek` เพื่อดูค่าถัดไปโดยไม่เลื่อน

```crystal
class SmartFibonacci
  # TODO: implement
end

fib = SmartFibonacci.new
puts fib.next   # 0
puts fib.next   # 1
puts fib.peek   # 1 (ไม่เลื่อน)
puts fib.next   # 1
fib.skip(3)
puts fib.next   # ??? (ข้ามไป 3 ตัว)
fib.reset
puts fib.next   # 0 (เริ่มใหม่)
```

---

## สรุป

| เทคนิค | เมื่อใช้ | ตัวอย่าง |
|--------|----------|----------|
| `each_with_index` | ต้องการ index ระหว่าง loop | `arr.each_with_index { |v, i| }` |
| `next` | ข้าม iteration ปัจจุบัน | `next if condition` |
| `break` | หยุด loop ทันที | `break if found` |
| `break value` | หยุดและส่งค่ากลับ | `result = arr.each { \|v\| break v * 2 if ... }` |
| `lazy` | ประหยัดหน่วยความจำ/infinite sequence | `(1..).lazy.select { }.first(10)` |
| `flat_map` | map แล้ว flatten | `nested.flat_map { \|arr\| arr.map {...} }` |
| `each_slice` | ตัด array เป็นก้อน | `arr.each_slice(3) { \|chunk\| }` |
| `each_cons` | sliding window | `arr.each_cons(3) { \|window\| }` |
| `zip` | รวม arrays หลายชุด | `a.zip(b).each { \|(x, y)\| }` |
| `tally` | นับความถี่ | `arr.tally` |

### หลักการสำคัญ

1. **ใช้ lazy สำหรับ collection ขนาดใหญ่** - ประหยัดหน่วยความจำ
2. **Guard ก่อน logic** - `next if invalid` ลดการ indent
3. **Method chaining** ทำให้โค้ดอ่านง่ายเหมือนภาษาธรรมชาติ
4. **Iterator pattern** ใช้สำหรับ custom sequence ที่ซับซ้อน
5. **เลือก `select`/`map`/`reject`** แทน manual loop เมื่อทำได้
