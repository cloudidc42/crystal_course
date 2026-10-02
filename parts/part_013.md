# ตอนที่ 13: Each และ Iteration

## บทนำ

Crystal มี collection methods ที่ทรงพลังสำหรับ iterate และ transform ข้อมูล การเรียนรู้ methods เหล่านี้จะทำให้เขียนโค้ดได้กระชับและชัดเจนกว่าการใช้ while loop

---

## 13.1 each

`each` iterate ผ่านทุก element ใน collection

```crystal
# each กับ Array
numbers = [1, 2, 3, 4, 5]
numbers.each do |n|
  puts n
end

# each แบบ one-liner
numbers.each { |n| puts n }

# each กับ String
"hello".each_char { |c| print "#{c}-" }
puts
# => h-e-l-l-o-

# each กับ Hash
scores = {"Alice" => 95, "Bob" => 82, "Charlie" => 71}
scores.each do |name, score|
  puts "#{name}: #{score} คะแนน"
end

# each กับ Range
(1..5).each { |i| print "#{i} " }
puts
# => 1 2 3 4 5
```

---

## 13.2 each_with_index

รู้ index ของแต่ละ element ระหว่าง iteration

```crystal
fruits = ["apple", "banana", "cherry"]

fruits.each_with_index do |fruit, index|
  puts "#{index + 1}. #{fruit}"
end

# Output:
# 1. apple
# 2. banana
# 3. cherry

# each_with_index กับ starting index
fruits.each_with_index(1) do |fruit, index|
  puts "#{index}. #{fruit}"
end
# Output: 1. apple, 2. banana, 3. cherry

# Hash กับ each_with_index
data = {"a" => 1, "b" => 2, "c" => 3}
data.each_with_index do |(key, value), index|
  puts "#{index}: #{key} = #{value}"
end
```

---

## 13.3 each_with_object

Accumulate ผลลัพธ์ไว้ใน object

```crystal
# สร้าง Hash จาก Array
words = ["hello", "world", "crystal"]
lengths = words.each_with_object({} of String => Int32) do |word, hash|
  hash[word] = word.size
end
puts lengths.inspect
# => {"hello" => 5, "world" => 5, "crystal" => 7}

# รวม items ที่เงื่อนไขตรง
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
result = numbers.each_with_object([] of Int32) do |n, arr|
  arr << n * n if n.odd?
end
puts result.inspect
# => [1, 9, 25, 49, 81]

# แปลง Array เป็น grouped Hash
people = [
  {name: "Alice", dept: "Engineering"},
  {name: "Bob", dept: "Marketing"},
  {name: "Charlie", dept: "Engineering"},
  {name: "Dave", dept: "Marketing"},
]

by_dept = people.each_with_object({} of String => Array(String)) do |person, groups|
  groups[person[:dept]] ||= [] of String
  groups[person[:dept]] << person[:name]
end

puts by_dept.inspect
# => {"Engineering" => ["Alice", "Charlie"], "Marketing" => ["Bob", "Dave"]}
```

---

## 13.4 times, upto, downto, step

```crystal
# times: ทำ n ครั้ง (0 ถึง n-1)
5.times do |i|
  print "#{i} "
end
puts
# => 0 1 2 3 4

5.times { puts "สวัสดี!" }  # พิมพ์ 5 ครั้ง

# upto: จาก a ถึง b
1.upto(5) { |i| print "#{i} " }
puts
# => 1 2 3 4 5

# downto: จาก a ลงมา b
10.downto(1) { |i| print "#{i} " }
puts
# => 10 9 8 7 6 5 4 3 2 1

# step: กระโดดทีละ step
# from.step(to, step_size)
0.step(to: 20, by: 5) { |i| print "#{i} " }
puts
# => 0 5 10 15 20

# step ลง
10.step(to: 0, by: -2) { |i| print "#{i} " }
puts
# => 10 8 6 4 2 0

# step กับ Float
0.0.step(to: 1.0, by: 0.25) { |f| print "#{f} " }
puts
# => 0.0 0.25 0.5 0.75 1.0
```

---

## 13.5 map / collect

แปลง collection เป็น collection ใหม่

```crystal
numbers = [1, 2, 3, 4, 5]

# map คูณทุกตัวด้วย 2
doubled = numbers.map { |n| n * 2 }
puts doubled.inspect  # => [2, 4, 6, 8, 10]

# map แปลง Int32 เป็น String
strings = numbers.map { |n| "Number #{n}" }
puts strings.inspect  # => ["Number 1", "Number 2", ...]

# map กับ String
words = ["hello", "world", "crystal"]
uppercased = words.map(&.upcase)
puts uppercased.inspect  # => ["HELLO", "WORLD", "CRYSTAL"]

# Short syntax: &.method_name
lengths = words.map(&.size)
puts lengths.inspect  # => [5, 5, 7]

# map กับ index (ใช้ map_with_index)
indexed = words.map_with_index { |w, i| "#{i}: #{w}" }
puts indexed.inspect
# => ["0: hello", "1: world", "2: crystal"]

# map! แก้ไข array ใน place
arr = [1, 2, 3]
arr.map! { |n| n * 10 }
puts arr.inspect  # => [10, 20, 30]
```

---

## 13.6 select / filter และ reject

```crystal
numbers = (1..10).to_a

# select: เลือก element ที่ผ่าน condition
evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

# Short syntax
evens = numbers.select(&.even?)

# reject: เลือก element ที่ไม่ผ่าน condition (ตรงข้าม select)
odds = numbers.reject { |n| n.even? }
puts odds.inspect  # => [1, 3, 5, 7, 9]

# ซ้อนกัน
big_odds = numbers.select { |n| n.odd? }.select { |n| n > 5 }
puts big_odds.inspect  # => [7, 9]

# select กับ String
words = ["Apple", "banana", "Cherry", "date"]
capitalized = words.select { |w| w[0].uppercase? }
puts capitalized.inspect  # => ["Apple", "Cherry"]

# select! แก้ไข in place
arr = [1, 2, 3, 4, 5, 6]
arr.select!(&.even?)
puts arr.inspect  # => [2, 4, 6]
```

---

## 13.7 reduce / inject

Accumulate ค่าทั้งหมดเป็นค่าเดียว

```crystal
numbers = [1, 2, 3, 4, 5]

# reduce ด้วย initial value
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # => 15

# reduce ไม่มี initial value (ใช้ element แรก)
product = numbers.reduce { |acc, n| acc * n }
puts product  # => 120

# inject เป็น alias ของ reduce
sum2 = numbers.inject(0, :+)  # สั้นกว่า
puts sum2  # => 15

product2 = numbers.inject(:*)
puts product2  # => 120

# ตัวอย่างจริง: หาค่าสูงสุด
max = numbers.reduce { |a, b| a > b ? a : b }
puts max  # => 5

# สร้าง String จาก Array
words = ["Crystal", "is", "awesome"]
sentence = words.reduce { |acc, w| "#{acc} #{w}" }
puts sentence  # => "Crystal is awesome"

# count characters
char_count = "hello world".chars.reduce(0) { |count, c| count + (c.letter? ? 1 : 0) }
puts char_count  # => 10
```

---

## 13.8 find / find_index

```crystal
numbers = [3, 1, 4, 1, 5, 9, 2, 6]

# find: หา element แรกที่ผ่าน condition
first_big = numbers.find { |n| n > 4 }
puts first_big.inspect  # => 5

# find คืน nil ถ้าไม่พบ
not_found = numbers.find { |n| n > 100 }
puts not_found.inspect  # => nil

# find_index: หา index
index = numbers.find_index { |n| n > 4 }
puts index.inspect  # => 4

# find_index ด้วยค่าโดยตรง
index2 = numbers.find_index(9)
puts index2.inspect  # => 5

# rindex: หาจากด้านหลัง
last_index = numbers.rindex { |n| n == 1 }
puts last_index.inspect  # => 3
```

---

## 13.9 any? / all? / none? / one?

```crystal
numbers = [1, 2, 3, 4, 5]

# any?: มี element อย่างน้อย 1 ตัวที่ผ่าน condition?
puts numbers.any? { |n| n > 4 }   # => true
puts numbers.any? { |n| n > 10 }  # => false
puts numbers.any?(&.even?)         # => true

# all?: ทุก element ผ่าน condition?
puts numbers.all? { |n| n > 0 }   # => true
puts numbers.all? { |n| n > 3 }   # => false
puts numbers.all?(&.positive?)     # => true

# none?: ไม่มี element เลยที่ผ่าน condition
puts numbers.none? { |n| n > 10 }  # => true
puts numbers.none? { |n| n > 4 }   # => false
puts numbers.none?(&.negative?)     # => true

# one?: มี element เพียง 1 ตัวที่ผ่าน condition
puts numbers.one? { |n| n == 3 }   # => true
puts numbers.one? { |n| n > 3 }    # => false (มี 2 ตัว: 4, 5)
puts numbers.one?(&.even?)          # => false

# ตัวอย่างจริง
words = ["hello", "world", "crystal"]
puts words.all? { |w| w.size > 3 }      # => true
puts words.any? { |w| w.starts_with?("c") }  # => true
puts words.none? { |w| w.empty? }       # => true
```

---

## 13.10 count

```crystal
numbers = [1, 2, 3, 2, 1, 4, 5, 2]

# count ทั้งหมด
puts numbers.count  # => 8

# count ที่ผ่าน condition
puts numbers.count { |n| n > 2 }   # => 3
puts numbers.count { |n| n.even? } # => 4

# count ด้วยค่าโดยตรง
puts numbers.count(2)  # => 3

# ตัวอย่าง: count characters
sentence = "Hello, World!"
vowels = "aeiouAEIOU"
vowel_count = sentence.chars.count { |c| vowels.includes?(c) }
puts "สระ: #{vowel_count}"  # => 3
```

---

## 13.11 group_by

จัดกลุ่ม elements ตาม key

```crystal
numbers = (1..10).to_a

# กลุ่มตามเลขคู่/คี่
grouped = numbers.group_by { |n| n.even? ? "คู่" : "คี่" }
puts grouped.inspect
# => {"คี่" => [1, 3, 5, 7, 9], "คู่" => [2, 4, 6, 8, 10]}

# กลุ่มตามเศษจากการหาร 3
mod_groups = numbers.group_by { |n| n % 3 }
puts mod_groups.inspect
# => {1 => [1, 4, 7, 10], 2 => [2, 5, 8], 0 => [3, 6, 9]}

# group_by กับ objects
people = [
  {name: "Alice", age: 25},
  {name: "Bob", age: 30},
  {name: "Charlie", age: 25},
  {name: "Dave", age: 35},
]

by_age = people.group_by { |p| p[:age] }
by_age.each do |age, persons|
  names = persons.map { |p| p[:name] }.join(", ")
  puts "อายุ #{age}: #{names}"
end
# Output:
# อายุ 25: Alice, Charlie
# อายุ 30: Bob
# อายุ 35: Dave

# group_by กับ String
words = ["apple", "ant", "banana", "bear", "cherry"]
by_letter = words.group_by { |w| w[0] }
by_letter.each { |letter, ws| puts "#{letter}: #{ws.join(", ")}" }
```

---

## 13.12 flat_map

Map แล้ว flatten ผลลัพธ์

```crystal
# flat_map แปลงและ flatten ในขั้นตอนเดียว
words = ["Hello World", "Crystal Lang"]
chars = words.flat_map { |w| w.split }
puts chars.inspect  # => ["Hello", "World", "Crystal", "Lang"]

# เทียบกับ map แล้ว flatten
chars2 = words.map { |w| w.split }.flatten
puts chars2.inspect  # เหมือนกัน

# flat_map กับ nested arrays
nested = [[1, 2], [3, 4], [5, 6]]
flat = nested.flat_map { |arr| arr.map { |n| n * 2 } }
puts flat.inspect  # => [2, 4, 6, 8, 10, 12]

# สร้าง pairs
a = [1, 2, 3]
b = ['a', 'b']
pairs = a.flat_map { |x| b.map { |y| {x, y} } }
puts pairs.inspect
# => [{1, 'a'}, {1, 'b'}, {2, 'a'}, {2, 'b'}, {3, 'a'}, {3, 'b'}]
```

---

## 13.13 each_slice / each_cons

```crystal
numbers = (1..10).to_a

# each_slice: แบ่งเป็นกลุ่มขนาด n
numbers.each_slice(3) do |slice|
  puts slice.inspect
end
# Output:
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10]

# each_cons: sliding window ขนาด n
numbers.each_cons(3) do |window|
  puts window.inspect
end
# Output:
# [1, 2, 3]
# [2, 3, 4]
# ...
# [8, 9, 10]

# ใช้ in_groups_of (มีใน Crystal)
numbers.each_slice(4).to_a.each do |group|
  puts group.inspect
end
```

---

## 13.14 zip

รวม arrays เข้าด้วยกัน element ต่อ element

```crystal
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]
scores = [95, 82, 71]

# zip รวม 2 arrays
pairs = names.zip(ages)
pairs.each { |(name, age)| puts "#{name}: #{age}" }

# zip รวม 3 arrays
triples = names.zip(ages, scores)
triples.each { |(name, age, score)| puts "#{name}, #{age} ปี, #{score} คะแนน" }

# zip! แก้ไข in place
a = [1, 2, 3]
b = [4, 5, 6]
a.zip(b).each { |(x, y)| puts "#{x} + #{y} = #{x + y}" }
```

---

## 13.15 tally

นับความถี่ของ elements

```crystal
# tally นับ occurrences
words = ["hello", "world", "hello", "crystal", "hello", "world"]
counts = words.tally
puts counts.inspect
# => {"hello" => 3, "world" => 2, "crystal" => 1}

# tally_by (Crystal ใหม่)
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
even_odd = numbers.group_by { |n| n.even? ? "even" : "odd" }
              .transform_values(&.size)
puts even_odd.inspect
# => {"odd" => 5, "even" => 5}
```

---

## 13.16 Chaining Iterators

```crystal
# Chain หลาย operations
result = (1..20)
  .select { |n| n.odd? }           # เลือกเลขคี่
  .map { |n| n * n }               # ยกกำลังสอง
  .select { |n| n > 20 }           # เลือกที่มากกว่า 20
  .first(3)                         # เอาแค่ 3 ตัวแรก

puts result.inspect  # => [25, 49, 81]

# Chain กับ Hash
word_freq = "hello world hello crystal world hello"
  .split
  .tally
  .select { |_, count| count > 1 }
  .to_a
  .sort_by { |(_, count)| -count }

word_freq.each { |(word, count)| puts "#{word}: #{count}" }
# Output:
# hello: 3
# world: 2
```

---

## 13.17 Lazy Evaluation

สำหรับ collection ขนาดใหญ่

```crystal
# ไม่ใช้ lazy - ประมวลผลทั้งหมดก่อน
result = (1..Float::INFINITY.to_i).select(&.even?).first(5)
# นี่จะ error หรือใช้เวลานานมาก!

# ใช้ lazy - ประมวลผลเท่าที่ต้องการ
result = (1..1000000)
  .lazy
  .select(&.even?)
  .map { |n| n * n }
  .first(5)

puts result.inspect  # => [4, 16, 36, 64, 100]

# Lazy กับ infinite sequence
fib = (0..).lazy.each_with_object({prev: 0, curr: 1}) do |_, obj|
  # นี่เป็นตัวอย่าง conceptual
end
```

---

## 13.18 ตัวอย่างโปรแกรมจริง: Student Report

```crystal
struct Student
  getter name : String
  getter scores : Array(Int32)

  def initialize(@name : String, @scores : Array(Int32))
  end

  def average : Float64
    return 0.0 if @scores.empty?
    @scores.sum.to_f / @scores.size
  end

  def highest : Int32
    @scores.max
  end

  def lowest : Int32
    @scores.min
  end

  def grade : String
    avg = average
    if avg >= 90 then "A"
    elsif avg >= 80 then "B"
    elsif avg >= 70 then "C"
    elsif avg >= 60 then "D"
    else "F"
    end
  end
end

students = [
  Student.new("Alice", [95, 87, 92, 88, 90]),
  Student.new("Bob", [72, 68, 75, 80, 70]),
  Student.new("Charlie", [55, 62, 58, 65, 60]),
  Student.new("Diana", [98, 95, 97, 100, 96]),
  Student.new("Eve", [45, 50, 48, 55, 52]),
]

puts "=== รายงานผลการเรียน ==="
puts

# แสดงคะแนนทุกคน
students.each_with_index do |student, i|
  puts "#{i + 1}. #{student.name}: เฉลี่ย #{student.average.round(1)} (#{student.grade})"
end

puts "\n=== สถิติ ==="

# คะแนนเฉลี่ยของทั้งห้อง
class_avg = students.map(&.average).sum / students.size
puts "คะแนนเฉลี่ยทั้งห้อง: #{class_avg.round(2)}"

# นักเรียนที่ผ่าน (เกรด >= D)
passing = students.select { |s| s.average >= 60 }
puts "ผ่าน: #{passing.size} คน"
puts "  #{passing.map(&.name).join(", ")}"

# นักเรียนที่ไม่ผ่าน
failing = students.reject { |s| s.average >= 60 }
puts "ไม่ผ่าน: #{failing.size} คน"
puts "  #{failing.map(&.name).join(", ")}" unless failing.empty?

# เรียงตามคะแนน
ranked = students.sort_by { |s| -s.average }
puts "\n=== จัดอันดับ ==="
ranked.each_with_index do |student, i|
  puts "อันดับ #{i + 1}: #{student.name} (#{student.average.round(1)})"
end

# จัดกลุ่มตามเกรด
by_grade = students.group_by(&.grade)
puts "\n=== จัดกลุ่มตามเกรด ==="
by_grade.each do |grade, group|
  names = group.map(&.name).join(", ")
  puts "เกรด #{grade}: #{names}"
end
```

---

## 13.19 ตัวอย่างโปรแกรมจริง: Data Pipeline

```crystal
# จำลอง data pipeline
raw_data = [
  "Alice,25,Engineering,90000",
  "Bob,30,Marketing,75000",
  "Charlie,28,Engineering,85000",
  "Diana,35,HR,70000",
  "Eve,27,Marketing,80000",
  "Frank,32,Engineering,95000",
]

struct Employee
  getter name : String
  getter age : Int32
  getter dept : String
  getter salary : Int32

  def initialize(@name : String, @age : Int32, @dept : String, @salary : Int32)
  end
end

# Parse data
employees = raw_data.map do |line|
  parts = line.split(",")
  Employee.new(parts[0], parts[1].to_i, parts[2], parts[3].to_i)
end

# Analysis pipeline
puts "=== วิเคราะห์ข้อมูลพนักงาน ==="

# หาเงินเดือนเฉลี่ยตามแผนก
dept_avg = employees
  .group_by(&.dept)
  .transform_values do |emps|
    (emps.sum(&.salary).to_f / emps.size).round(0).to_i
  end

puts "\nเงินเดือนเฉลี่ยตามแผนก:"
dept_avg.to_a.sort_by { |(_, avg)| -avg }.each do |dept, avg|
  puts "  #{dept}: #{avg.to_s.rjust(6)} บาท"
end

# หาพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ย
total_avg = employees.sum(&.salary).to_f / employees.size
above_avg = employees
  .select { |e| e.salary > total_avg }
  .sort_by { |e| -e.salary }

puts "\nพนักงานที่มีเงินเดือนสูงกว่าค่าเฉลี่ย (#{total_avg.round(0).to_i} บาท):"
above_avg.each { |e| puts "  #{e.name}: #{e.salary} บาท" }

# อายุเฉลี่ยตามแผนก
puts "\nอายุเฉลี่ยตามแผนก:"
employees
  .group_by(&.dept)
  .transform_values { |emps| (emps.sum(&.age).to_f / emps.size).round(1) }
  .each { |dept, avg_age| puts "  #{dept}: #{avg_age} ปี" }
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Number Analysis
```crystal
numbers = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# ใช้ iteration methods เพื่อ:
# 1. หาผลรวม (ใช้ sum หรือ reduce)
# 2. หาค่าเฉลี่ย
# 3. หาค่าที่ไม่ซ้ำ (unique values)
# 4. นับความถี่แต่ละค่า
# 5. เรียงและหา median
```

### แบบฝึกหัดที่ 2: Word Counter
```crystal
text = "the quick brown fox jumps over the lazy dog the fox"
# ใช้ iteration methods เพื่อ:
# 1. นับคำทั้งหมด
# 2. หาคำที่ไม่ซ้ำ
# 3. หาคำที่ปรากฏบ่อยที่สุด
# 4. หาความยาวเฉลี่ยของคำ
# 5. แสดง top 3 คำ
```

### แบบฝึกหัดที่ 3: Matrix Operations
```crystal
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
# ใช้ map และ reduce เพื่อ:
# 1. Transpose (สลับแถวกับคอลัมน์)
# 2. หา row sums
# 3. หา column sums
# 4. แปลงทุก element เป็น string พร้อม padding
```

### แบบฝึกหัดที่ 4: Pipeline
```crystal
# สร้าง data processing pipeline ที่:
# รับ array of strings (CSV format: "name,age,score")
# Parse เป็น objects
# กรองเฉพาะ score >= 70
# แปลง score เป็น grade
# เรียงตาม name
# แสดงผล
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ iteration methods ที่สำคัญ:

| Method | วัตถุประสงค์ |
|--------|------------|
| `each` | iterate ผ่านทุก element |
| `each_with_index` | iterate พร้อม index |
| `each_with_object` | iterate พร้อม accumulator |
| `times` / `upto` / `downto` | numeric iteration |
| `map` | แปลง collection |
| `select` / `reject` | กรอง collection |
| `reduce` / `inject` | รวมเป็นค่าเดียว |
| `find` | หา element แรก |
| `any?` / `all?` / `none?` | ตรวจสอบเงื่อนไข |
| `count` | นับ elements |
| `group_by` | จัดกลุ่ม |
| `flat_map` | map และ flatten |
| `tally` | นับความถี่ |

การใช้ iteration methods แทน while loop ทำให้โค้ดอ่านง่าย กระชับ และ expressive มากขึ้น!
