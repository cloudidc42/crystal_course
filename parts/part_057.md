# Part 57: Lazy Iterators ใน Crystal

## บทนำ

Lazy Iterators ช่วยให้เราประมวลผลข้อมูลโดยไม่ต้องสร้าง intermediate collections ทั้งหมด ทำให้ประหยัด memory และสามารถทำงานกับ infinite sequences ได้

---

## 57.1 พื้นฐาน Lazy

```crystal
# Eager evaluation (ปกติ)
numbers = (1..1_000_000).to_a
result = numbers.select { |n| n.odd? }.map { |n| n * 2 }.first(5)
puts result.inspect  # => [2, 6, 10, 14, 18]
# ปัญหา: สร้าง array ขนาด 1,000,000 ก่อน แล้วค่อย filter

# Lazy evaluation
result_lazy = (1..1_000_000).lazy
  .select { |n| n.odd? }
  .map { |n| n * 2 }
  .first(5)
puts result_lazy.inspect  # => [2, 6, 10, 14, 18]
# ดี: คำนวณเท่าที่ต้องการ ไม่สร้าง intermediate arrays
```

---

## 57.2 map.lazy

```crystal
# Lazy map
doubles = (1..10).lazy.map { |n| n * 2 }

# ยังไม่คำนวณ ณ ตอนนี้
# คำนวณเมื่อเรียก each, first, to_a ฯลฯ

puts doubles.first(5).inspect   # => [2, 4, 6, 8, 10]
puts doubles.first(3).inspect   # => [2, 4, 6]

# to_a บังคับ evaluate ทั้งหมด
all = doubles.to_a
puts all.inspect  # => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
```

---

## 57.3 select.lazy

```crystal
# Lazy select บน range
evens = (1..Float64::INFINITY.to_i).lazy.select { |n| n.even? }
# ไม่ crash แม้ range จะ "infinite"

puts evens.first(5).inspect  # => [2, 4, 6, 8, 10]

# Lazy select บน array
words = %w[apple banana cherry date elderberry fig grape]
long_words = words.lazy.select { |w| w.size > 5 }

puts long_words.first(3).inspect  # => ["banana", "cherry", "elderberry"]
```

---

## 57.4 first(n) บน lazy

```crystal
# first(n) terminate lazy evaluation เมื่อได้ n elements
primes_lazy = (2..Int32::MAX).lazy.select do |n|
  (2..Math.sqrt(n.to_f).to_i).none? { |i| n % i == 0 }
end

first_10_primes = primes_lazy.first(10)
puts first_10_primes.inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]

first_prime_over_100 = primes_lazy.find { |n| n > 100 }
puts first_prime_over_100  # => 101
```

---

## 57.5 take และ take_while

```crystal
lazy_nums = (1..Int32::MAX).lazy

# take(n): เอาแค่ n ตัวแรก
first_five = lazy_nums.take(5)
puts first_five.to_a.inspect  # => [1, 2, 3, 4, 5]

# take_while: เอาจนกว่าเงื่อนไขเป็น false
under_10 = lazy_nums.take_while { |n| n < 10 }
puts under_10.to_a.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

# ตัวอย่าง: fibonacci ที่ไม่เกิน 1000
def fibonacci_lazy
  (0..Int32::MAX).lazy.map do |i|
    a, b = 0, 1
    i.times { a, b = b, a + b }
    a
  end
end

fibs_under_1000 = fibonacci_lazy.take_while { |n| n <= 1000 }
puts fibs_under_1000.to_a.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89, 144, 233, 377, 610, 987]
```

---

## 57.6 skip และ skip_while

```crystal
lazy_nums = (1..20).lazy

# skip(n): ข้าม n ตัวแรก
after_5 = lazy_nums.skip(5)
puts after_5.to_a.inspect  # => [6, 7, 8, 9, 10, ...]

# skip_while: ข้ามจนกว่าเงื่อนไขเป็น false
after_odds = lazy_nums.skip_while { |n| n.odd? }
puts after_odds.first(5).inspect  # => [2, 3, 4, 5, 6, 7, ...]
# หมายเหตุ: skip_while หยุดเมื่อ condition เป็น false ครั้งแรก

# ตัวอย่าง: pagination
def paginate(data, page : Int32, per_page : Int32)
  data.lazy
    .skip((page - 1) * per_page)
    .take(per_page)
    .to_a
end

all_items = (1..100).to_a
page1 = paginate(all_items, 1, 10)
page2 = paginate(all_items, 2, 10)
page5 = paginate(all_items, 5, 10)

puts "Page 1: #{page1.inspect}"
# => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
puts "Page 2: #{page2.inspect}"
# => [11, 12, 13, 14, 15, 16, 17, 18, 19, 20]
puts "Page 5: #{page5.inspect}"
# => [41, 42, 43, 44, 45, 46, 47, 48, 49, 50]
```

---

## 57.7 force - บังคับ Evaluate

```crystal
# force เหมือน to_a แต่ชื่อชัดเจนกว่า
lazy = (1..10).lazy.map { |n| n * 3 }.select { |n| n > 15 }

result = lazy.force  # บังคับ evaluate
puts result.inspect  # => [18, 21, 24, 27, 30]

# to_a ก็ทำแบบเดียวกัน
result2 = lazy.to_a
puts result2.inspect  # => [18, 21, 24, 27, 30]
```

---

## 57.8 Lazy Chains สำหรับ Infinite Sequences

```crystal
# Collatz sequence
def collatz(n : Int64) : Array(Int64)
  seq = [n]
  while n != 1
    n = n.even? ? n / 2 : n * 3 + 1
    seq << n
  end
  seq
end

# หา number ที่มี Collatz sequence ยาวที่สุดใน 1-100
longest = (1..100).lazy
  .map { |n| {n, collatz(n.to_i64).size} }
  .max_by { |_, len| len }

puts "Longest Collatz in 1-100: #{longest[0]} (length: #{longest[1]})"

# Natural numbers divisible by both 3 and 5
fizzbuzz = (1..Int32::MAX).lazy
  .select { |n| n % 3 == 0 && n % 5 == 0 }
  .first(10)
puts fizzbuzz.inspect  # => [15, 30, 45, 60, 75, 90, 105, 120, 135, 150]

# Powers of 2 under 1000
powers_of_2 = (0..Int32::MAX).lazy
  .map { |n| 2 ** n }
  .take_while { |n| n < 1000 }
puts powers_of_2.to_a.inspect
# => [1, 2, 4, 8, 16, 32, 64, 128, 256, 512]
```

---

## 57.9 Lazy กับ Custom Iterators

```crystal
# Custom lazy sequence
class LazyMap(T, U)
  include Iterator(U)
  
  def initialize(@source : Iterator(T), @transform : T -> U)
  end
  
  def next : U | Stop
    value = @source.next
    return stop if value.is_a?(Iterator::Stop)
    @transform.call(value.as(T))
  end
end

class LazyFilter(T)
  include Iterator(T)
  
  def initialize(@source : Iterator(T), @predicate : T -> Bool)
  end
  
  def next : T | Stop
    loop do
      value = @source.next
      return stop if value.is_a?(Iterator::Stop)
      val = value.as(T)
      return val if @predicate.call(val)
    end
  end
end

# ใช้งาน
source = (1..100).each

filtered = LazyFilter(Int32).new(source) { |n| n % 7 == 0 }
puts filtered.first(5).inspect  # => [7, 14, 21, 28, 35]
```

---

## 57.10 Comparison: Lazy vs Eager

```crystal
require "benchmark"

# Test data
n = 1_000_000

# Eager approach
eager_time = Time.measure do
  result = (1..n).to_a
    .select { |i| i % 2 == 0 }
    .map { |i| i * i }
    .first(10)
end

# Lazy approach
lazy_time = Time.measure do
  result = (1..n).lazy
    .select { |i| i % 2 == 0 }
    .map { |i| i * i }
    .first(10)
end

puts "Eager: #{eager_time.total_milliseconds.round(2)}ms"
puts "Lazy: #{lazy_time.total_milliseconds.round(2)}ms"
puts "Speedup: #{(eager_time / lazy_time).round(1)}x"
```

---

## 57.11 Memory Efficiency

```crystal
# Lazy ใช้ memory น้อยกว่ามาก

# ตัวอย่าง: process large file line by line
def process_lines_lazy(lines : Array(String))
  lines.lazy
    .reject { |line| line.starts_with?("#") }
    .reject { |line| line.strip.empty? }
    .map { |line| line.strip }
    .map { |line| line.split("=", 2) }
    .select { |parts| parts.size == 2 }
    .map { |parts| {parts[0].strip, parts[1].strip} }
    .to_h
end

config_lines = [
  "# Database config",
  "host = localhost",
  "port = 5432",
  "",
  "# App config",
  "debug = true",
  "max_conn = 100"
]

config = process_lines_lazy(config_lines)
puts config.inspect
# => {"host" => "localhost", "port" => "5432", "debug" => "true", "max_conn" => "100"}
```

---

## 57.12 Lazy Zip

```crystal
# Lazy zip รวม iterators โดยไม่ materialize
a = (1..Int32::MAX).lazy
b = ('a'..'z').lazy

# เอาแค่ 5 คู่
pairs = a.zip(b).first(5)
puts pairs.inspect
# => [{1, 'a'}, {2, 'b'}, {3, 'c'}, {4, 'd'}, {5, 'e'}]

# Fibonacci + Natural numbers
def fibonacci_seq
  (0..Int32::MAX).lazy.map { |i|
    a, b = 0, 1
    i.times { a, b = b, a + b }
    a
  }
end

naturals = (1..Int32::MAX).lazy
fib = fibonacci_seq

combined = naturals.zip(fib).first(8)
combined.each { |n, f| puts "#{n}: #{f}" }
# 1: 0
# 2: 1
# 3: 1
# 4: 2
# 5: 3
# 6: 5
# 7: 8
# 8: 13
```

---

## 57.13 ตัวอย่างในชีวิตจริง: Data Processing Pipeline

```crystal
# Simulating a data processing pipeline
class DataProcessor
  def self.process_orders(orders : Array(NamedTuple(
    id: Int32, 
    customer: String, 
    amount: Float64, 
    status: String
  )))
    
    # Lazy pipeline
    orders.lazy
      .select { |o| o[:status] == "completed" }
      .select { |o| o[:amount] > 100.0 }
      .map { |o| 
        tax = o[:amount] * 0.1
        {id: o[:id], customer: o[:customer], amount: o[:amount], tax: tax, total: o[:amount] + tax}
      }
      .to_a
  end
end

orders = [
  {id: 1, customer: "Alice", amount: 150.0, status: "completed"},
  {id: 2, customer: "Bob", amount: 50.0, status: "completed"},
  {id: 3, customer: "Charlie", amount: 200.0, status: "pending"},
  {id: 4, customer: "Dave", amount: 300.0, status: "completed"},
  {id: 5, customer: "Eve", amount: 75.0, status: "cancelled"},
  {id: 6, customer: "Frank", amount: 500.0, status: "completed"}
]

processed = DataProcessor.process_orders(orders)
puts "Qualifying orders:"
processed.each do |o|
  puts "  ##{o[:id]} #{o[:customer]}: $#{o[:amount]} + $#{o[:tax].round(2)} tax = $#{o[:total].round(2)}"
end

total_revenue = processed.sum { |o| o[:total] }
puts "Total revenue: $#{total_revenue.round(2)}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sieve of Eratosthenes ด้วย Lazy

```crystal
# Lazy prime generation
def primes_up_to(n : Int32) : Array(Int32)
  (2..n).lazy.select { |num|
    (2..Math.sqrt(num.to_f).to_i).none? { |i| num % i == 0 }
  }.to_a
end

puts primes_up_to(50).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

# Twin primes (primes that differ by 2)
def twin_primes(limit : Int32) : Array({Int32, Int32})
  primes = primes_up_to(limit)
  primes.lazy.zip(primes[1..].lazy)
        .select { |a, b| b - a == 2 }
        .to_a
end

puts "Twin primes up to 50: #{twin_primes(50).inspect}"
# => [{3, 5}, {5, 7}, {11, 13}, {17, 19}, {29, 31}, {41, 43}]
```

### แบบฝึกหัดที่ 2: Lazy Text Processing

```crystal
def word_frequency_lazy(text : String, top_n : Int32 = 10)
  # Process lazily
  words = text.split(/\s+/).lazy
    .map { |w| w.downcase.gsub(/[^a-z]/, "") }
    .reject { |w| w.empty? }
    .to_a
  
  freq = words.tally
  
  freq.to_a
    .lazy
    .sort_by { |_, count| -count }
    .first(top_n)
    .to_a
end

text = "Crystal is a compiled statically typed programming language. Crystal is fast. Crystal is type safe. Programming in Crystal is fun."
top = word_frequency_lazy(text, 5)
puts "Top 5 words:"
top.each { |word, count| puts "  '#{word}': #{count}" }
```

---

## สรุป

Lazy Iterators ใน Crystal:

| Method | คำอธิบาย |
|--------|---------|
| `.lazy` | เปลี่ยน collection เป็น lazy |
| `map` (lazy) | แปลงแบบ lazy |
| `select` (lazy) | กรองแบบ lazy |
| `reject` (lazy) | กรองออกแบบ lazy |
| `take(n)` | เอาแค่ n ตัว |
| `take_while` | เอาจนเงื่อนไขเป็น false |
| `skip(n)` | ข้าม n ตัว |
| `skip_while` | ข้ามจนเงื่อนไขเป็น false |
| `first(n)` | บังคับ evaluate แค่ n ตัว |
| `force` / `to_a` | บังคับ evaluate ทั้งหมด |
| `zip` | รวม lazy iterators |

**เมื่อใช้ Lazy:**
- ข้อมูลมีขนาดใหญ่มาก
- ต้องการ infinite sequences
- ต้องการ early termination
- Memory เป็นข้อจำกัด
- Pipeline processing ที่ซับซ้อน

---

*ต่อไป: Part 58 - Comprehensions*
