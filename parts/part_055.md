# Part 55: Map, Select, Reject, Reduce ใน Crystal

## บทนำ

สี่ methods นี้เป็นเครื่องมือหลักของ functional programming ใน Crystal:
- **map**: แปลงแต่ละ element
- **select**: กรองเฉพาะที่ตรงเงื่อนไข
- **reject**: กรองออกที่ตรงเงื่อนไข
- **reduce**: พับ elements เป็นค่าเดียว

---

## 55.1 map - การแปลงข้อมูล

```crystal
numbers = [1, 2, 3, 4, 5]

# map พื้นฐาน
doubled = numbers.map { |n| n * 2 }
puts doubled.inspect  # => [2, 4, 6, 8, 10]

# map กับ string
words = ["hello", "world", "crystal"]
uppercase = words.map { |w| w.upcase }
puts uppercase.inspect  # => ["HELLO", "WORLD", "CRYSTAL"]

# map กับ method reference
lengths = words.map(&.size)
puts lengths.inspect  # => [5, 5, 7]

# map เปลี่ยน type
strings = numbers.map(&.to_s)
puts strings.inspect   # => ["1", "2", "3", "4", "5"]
puts typeof(strings)   # => Array(String)

# map กับ complex transformation
data = [{name: "Alice", score: 85}, {name: "Bob", score: 92}, {name: "Charlie", score: 78}]

formatted = data.map { |d|
  grade = d[:score] >= 90 ? "A" : d[:score] >= 80 ? "B" : "C"
  "#{d[:name]}: #{d[:score]} (#{grade})"
}
formatted.each { |line| puts line }
# Alice: 85 (B)
# Bob: 92 (A)
# Charlie: 78 (C)
```

---

## 55.2 map กับ index

```crystal
arr = ["a", "b", "c", "d", "e"]

# map กับ index
indexed = arr.map_with_index { |item, i| "#{i+1}. #{item}" }
puts indexed.inspect
# => ["1. a", "2. b", "3. c", "4. d", "5. e"]

# หรือใช้ each_with_index
indexed2 = arr.each_with_index.map { |item, i| {i, item.upcase} }
puts indexed2.inspect
# => [{0, "A"}, {1, "B"}, {2, "C"}, {3, "D"}, {4, "E"}]

# Numbered list
items = ["Crystal", "Ruby", "Python"]
numbered = items.map_with_index(1) { |item, i| "#{i}. #{item}" }
numbered.each { |line| puts line }
# 1. Crystal
# 2. Ruby
# 3. Python
```

---

## 55.3 map! - In-Place Transformation

```crystal
numbers = [1, 2, 3, 4, 5]

# map! แก้ไข array เดิม
numbers.map! { |n| n ** 2 }
puts numbers.inspect  # => [1, 4, 9, 16, 25]

# map! บน strings
names = ["alice", "bob", "charlie"]
names.map!(&.capitalize)
puts names.inspect  # => ["Alice", "Bob", "Charlie"]

# ข้อควรระวัง: map! เปลี่ยน original
original = [1, 2, 3]
copy = original.dup

copy.map! { |n| n * 10 }
puts original.inspect  # => [1, 2, 3] (ไม่เปลี่ยน)
puts copy.inspect      # => [10, 20, 30] (เปลี่ยน)
```

---

## 55.4 select - Filter Truthy

```crystal
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# select เก็บเฉพาะที่ block คืน truthy
evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

big = numbers.select { |n| n > 7 }
puts big.inspect  # => [8, 9, 10]

# select บน strings
words = ["apple", "banana", "cherry", "date", "fig"]
long_words = words.select { |w| w.size > 4 }
puts long_words.inspect  # => ["apple", "banana", "cherry"]

starts_with_b = words.select { |w| w.starts_with?("b") }
puts starts_with_b.inspect  # => ["banana"]

# select กับ method reference
has_a = words.select { |w| w.includes?("a") }
puts has_a.inspect  # => ["apple", "banana", "date"]

# select บน Hash
scores = {"Alice" => 95, "Bob" => 72, "Charlie" => 88, "Dave" => 65}
high_scorers = scores.select { |_, score| score >= 80 }
puts high_scorers.inspect  # => {"Alice" => 95, "Charlie" => 88}
```

---

## 55.5 select! - In-Place Filtering

```crystal
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# select! เก็บเฉพาะที่ตรงเงื่อนไข (แก้ไขต้นฉบับ)
arr.select! { |n| n.odd? }
puts arr.inspect  # => [1, 3, 5, 7, 9]

# Hash select!
h = {"a" => 1, "b" => 2, "c" => 3, "d" => 4}
h.select! { |_, v| v.even? }
puts h.inspect  # => {"b" => 2, "d" => 4}

# ใช้เมื่อต้องการประหยัด memory
large_data = (1..1000).to_a
large_data.select! { |n| n % 100 == 0 }  # เก็บแค่ multiples of 100
puts large_data.inspect
```

---

## 55.6 reject - Filter Falsy

```crystal
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# reject ลบออกสิ่งที่ block คืน truthy
odds = numbers.reject { |n| n.even? }
puts odds.inspect  # => [1, 3, 5, 7, 9]

# ตรงข้ามกับ select
selected = numbers.select { |n| n > 5 }
rejected = numbers.reject { |n| n <= 5 }
puts selected == rejected  # => true

# reject กับ nil
mixed = [1, nil, 2, nil, 3, nil]
without_nil = mixed.reject { |x| x.nil? }
puts without_nil.inspect  # => [1, 2, 3]

# หรือใช้ compact
puts mixed.compact.inspect  # => [1, 2, 3]

# reject บน Hash
inventory = {"A" => 10, "B" => 0, "C" => 5, "D" => 0}
in_stock = inventory.reject { |_, qty| qty == 0 }
puts in_stock.inspect  # => {"A" => 10, "C" => 5}
```

---

## 55.7 reject! - In-Place Rejection

```crystal
arr = ["", "hello", "", "world", ""]

# reject! ลบที่ตรงเงื่อนไข (แก้ไขต้นฉบับ)
arr.reject! { |s| s.empty? }
puts arr.inspect  # => ["hello", "world"]

# ลบ nil values
nullable = [1, nil, 2, nil, 3]
nullable.reject! { |x| x.nil? }
puts nullable.inspect  # => [1, 2, 3]

# Hash reject!
h = {"active" => true, "deleted" => false, "pending" => false, "running" => true}
h.reject! { |_, active| !active }
puts h.inspect  # => {"active" => true, "running" => true}
```

---

## 55.8 reduce/inject - พับ Collection

```crystal
numbers = [1, 2, 3, 4, 5]

# reduce พื้นฐาน
sum = numbers.reduce { |acc, n| acc + n }
puts sum  # => 15

product = numbers.reduce { |acc, n| acc * n }
puts product  # => 120

# reduce กับ initial value
sum_with_init = numbers.reduce(100) { |acc, n| acc + n }
puts sum_with_init  # => 115

# inject (alias)
max = numbers.inject { |m, n| m > n ? m : n }
puts max  # => 5

# reduce เปลี่ยน type
words = ["Crystal", "is", "awesome"]
sentence = words.reduce { |acc, word| "#{acc} #{word}" }
puts sentence  # => "Crystal is awesome"

# reduce สร้าง Hash
pairs = [["a", 1], ["b", 2], ["c", 3]]
hash_from_pairs = pairs.reduce({} of String => Int32) do |hash, pair|
  hash[pair[0]] = pair[1]
  hash
end
puts hash_from_pairs.inspect  # => {"a" => 1, "b" => 2, "c" => 3}
```

---

## 55.9 each_with_object

```crystal
# each_with_object: สะสมใน object ที่กำหนด
# ต่างจาก reduce ตรงที่ block argument เป็น (element, accumulator)
# และ accumulator ไม่ถูก return จาก block (object ยังเป็น reference เดิม)

numbers = [1, 2, 3, 4, 5, 6]

# สร้าง Hash จาก array
grouped = numbers.each_with_object({} of String => Array(Int32)) do |n, hash|
  key = n.even? ? "even" : "odd"
  (hash[key] ||= [] of Int32) << n
end
puts grouped.inspect  # => {"odd" => [1, 3, 5], "even" => [2, 4, 6]}

# สร้าง string
result_str = ["hello", "world", "!"].each_with_object("") do |word, str|
  str << " " unless str.empty?
  str << word
end
# Note: String ใน Crystal เป็น mutable reference ใน block นี้

# ใช้กับ Array
words = ["apple", "banana", "cherry", "blueberry", "avocado"]
starting_with = words.each_with_object(Hash(Char, Array(String)).new) do |word, hash|
  first = word[0]
  (hash[first] ||= [] of String) << word
end
starting_with.each { |letter, ws| puts "#{letter}: #{ws.join(", ")}" }
# a: apple, avocado
# b: banana, blueberry
# c: cherry
```

---

## 55.10 filter_map

```crystal
# filter_map = select + map ในขั้นตอนเดียว
# คืน nil จาก block = skip element

data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# แปลงเฉพาะ even numbers
even_squares = data.filter_map do |n|
  n * n if n.even?
end
puts even_squares.inspect  # => [4, 16, 36, 64, 100]

# Parse strings เป็น int (skip invalid)
strings = ["1", "two", "3", "four", "5"]
numbers = strings.filter_map { |s| s.to_i? }
puts numbers.inspect  # => [1, 3, 5]

# Extract และ transform
users = [
  {name: "Alice", age: 25, premium: true},
  {name: "Bob", age: 30, premium: false},
  {name: "Charlie", age: 22, premium: true}
]

premium_names = users.filter_map do |user|
  user[:name].upcase if user[:premium]
end
puts premium_names.inspect  # => ["ALICE", "CHARLIE"]

# ตัวอย่างจริง: Parse CSV
csv_rows = ["1,Alice,95", "invalid", "2,Bob,87", "", "3,Charlie,92"]

records = csv_rows.filter_map do |row|
  parts = row.split(",")
  if parts.size == 3
    id = parts[0].to_i?
    score = parts[2].to_i?
    id && score ? {id: id.not_nil!, name: parts[1], score: score.not_nil!} : nil
  end
end

records.each { |r| puts "#{r[:id]}. #{r[:name]}: #{r[:score]}" }
```

---

## 55.11 Chaining Operations

```crystal
data = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# Chain map, select, reduce
result = data
  .select { |n| n > 3 }    # [4, 5, 9, 6, 5, 5]
  .map { |n| n * 2 }        # [8, 10, 18, 12, 10, 10]
  .reject { |n| n > 15 }   # [8, 10, 12, 10, 10]
  .reduce { |sum, n| sum + n }  # 50

puts result  # => 50

# ตัวอย่างจริง: Data pipeline
students = [
  {name: "Alice", scores: [85, 92, 78, 96]},
  {name: "Bob", scores: [65, 70, 68, 72]},
  {name: "Charlie", scores: [92, 95, 88, 97]},
  {name: "Dave", scores: [55, 60, 58, 62]}
]

# หา top students ที่มี average >= 80
top_students = students
  .map { |s| {name: s[:name], avg: s[:scores].sum.to_f / s[:scores].size} }
  .select { |s| s[:avg] >= 80.0 }
  .sort_by { |s| -s[:avg] }
  .map { |s| "#{s[:name]} (#{s[:avg].round(1)})" }

puts "Top students: #{top_students.join(", ")}"
# => Top students: Charlie (93.0), Alice (87.75)
```

---

## 55.12 Lazy Versions

```crystal
# ใช้ .lazy สำหรับ large datasets
numbers = (1..1_000_000)

# ไม่มี lazy: สร้าง array ขนาดใหญ่ก่อน
# result = numbers.to_a.select { |n| n % 3 == 0 }.map { |n| n * n }.first(5)

# มี lazy: ประมวลผลเท่าที่ต้องการ
result = numbers.lazy
  .select { |n| n % 3 == 0 }
  .map { |n| n * n }
  .first(5)

puts result.inspect  # => [9, 36, 81, 144, 225]

# Lazy กับ infinite sequences
primes_lazy = (2..Int32::MAX).lazy.select { |n|
  (2..Math.sqrt(n.to_f).to_i).none? { |i| n % i == 0 }
}

puts primes_lazy.first(10).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29]
```

---

## 55.13 ตัวอย่างในชีวิตจริง: Pipeline Processing

```crystal
class DataPipeline
  def initialize(@data : Array(Hash(String, String | Int32 | Float64 | Bool)))
  end
  
  def filter(& : Hash(String, String | Int32 | Float64 | Bool) -> Bool) : DataPipeline
    DataPipeline.new(@data.select { |item| yield item })
  end
  
  def transform(& : Hash(String, String | Int32 | Float64 | Bool) -> Hash(String, String | Int32 | Float64 | Bool)) : DataPipeline
    DataPipeline.new(@data.map { |item| yield item })
  end
  
  def aggregate : Hash(String, Float64 | Int32)
    {
      "count" => @data.size.to_i32,
      "total" => @data.sum { |item|
        item["amount"]?.try { |v| v.as(Float64) } || 0.0
      }
    }
  end
  
  def to_a : Array(Hash(String, String | Int32 | Float64 | Bool))
    @data
  end
end

# Sales data
sales = [
  {"region" => "North", "amount" => 1500.0, "category" => "A", "active" => true},
  {"region" => "South", "amount" => 2200.0, "category" => "B", "active" => true},
  {"region" => "North", "amount" => 800.0, "category" => "A", "active" => false},
  {"region" => "East", "amount" => 3000.0, "category" => "A", "active" => true},
  {"region" => "West", "amount" => 1100.0, "category" => "B", "active" => true}
]

pipeline = DataPipeline.new(sales)
  .filter { |s| s["active"].as(Bool) }
  .filter { |s| s["amount"].as(Float64) > 1000.0 }

result = pipeline.aggregate
puts "Active sales > 1000: #{result["count"]}"
puts "Total: $#{result["total"]}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Map + Select Chain

```crystal
# ข้อมูลพนักงาน
employees = [
  {name: "Alice", dept: "Engineering", salary: 85000, years: 3},
  {name: "Bob", dept: "Marketing", salary: 65000, years: 7},
  {name: "Charlie", dept: "Engineering", salary: 95000, years: 5},
  {name: "Dave", dept: "HR", salary: 55000, years: 2},
  {name: "Eve", dept: "Engineering", salary: 75000, years: 1}
]

# 1. หา Engineering employees ที่ทำงาน >= 3 ปี
senior_engineers = employees
  .select { |e| e[:dept] == "Engineering" && e[:years] >= 3 }
  .map { |e| {name: e[:name], salary: e[:salary]} }

puts "Senior Engineers:"
senior_engineers.each { |e| puts "  #{e[:name]}: $#{e[:salary]}" }

# 2. คำนวณ total salary ของแต่ละ department
dept_total = employees
  .each_with_object(Hash(String, Int32).new(0)) do |emp, totals|
    totals[emp[:dept]] += emp[:salary]
  end

puts "\nDepartment salary totals:"
dept_total.sort_by { |_, t| -t }.each { |dept, total| puts "  #{dept}: $#{total}" }

# 3. หา employees ที่ได้รับ raise
with_raise = employees.filter_map do |emp|
  if emp[:salary] > 70000
    {name: emp[:name], new_salary: (emp[:salary] * 1.1).to_i}
  end
end

puts "\nEmployees getting raise:"
with_raise.each { |e| puts "  #{e[:name]}: $#{e[:new_salary]}" }
```

### แบบฝึกหัดที่ 2: Custom Reduce

```crystal
# Implement custom operations using reduce

def flatten_once(nested : Array(Array(Int32))) : Array(Int32)
  nested.reduce([] of Int32) { |acc, arr| acc + arr }
end

def deep_sum(nested : Array(Array(Int32))) : Int32
  nested.reduce(0) { |acc, arr| acc + arr.sum }
end

def running_total(numbers : Array(Int32)) : Array(Int32)
  numbers.reduce([] of Int32) { |acc, n|
    acc + [acc.last? ? acc.last + n : n]
  }
end

matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
puts flatten_once(matrix).inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts deep_sum(matrix)              # => 45

numbers = [1, 2, 3, 4, 5]
puts running_total(numbers).inspect  # => [1, 3, 6, 10, 15]
```

---

## สรุป

| Method | คำอธิบาย | คืน |
|--------|---------|-----|
| `map` | แปลงแต่ละ element | Array ใหม่ |
| `map!` | แปลง in-place | self |
| `map_with_index` | แปลงพร้อม index | Array ใหม่ |
| `select` | กรองเฉพาะ truthy | Array/Hash ใหม่ |
| `select!` | กรอง in-place | self |
| `reject` | กรองออก truthy | Array/Hash ใหม่ |
| `reject!` | กรองออก in-place | self |
| `reduce` / `inject` | พับเป็นค่าเดียว | ค่าสุดท้าย |
| `each_with_object` | สะสมใน object | object |
| `filter_map` | select + map | Array ใหม่ |
| `.lazy` | lazy evaluation | LazyEnumerator |

**หลักการเลือกใช้:**
- `map`: เมื่อต้องการแปลงทุก element
- `select/reject`: เมื่อต้องการกรอง
- `reduce`: เมื่อต้องการรวมเป็นค่าเดียว
- `filter_map`: เมื่อต้องการทั้งกรองและแปลง
- `each_with_object`: เมื่อต้องการสะสมใน mutable container

---

*ต่อไป: Part 56 - Sorting และ Filtering*
