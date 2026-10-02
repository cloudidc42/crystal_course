# Part 56: Sorting และ Filtering ใน Crystal

## บทนำ

การเรียงลำดับและกรองข้อมูลเป็นหัวใจของการประมวลผลข้อมูล Crystal มี methods ที่ครอบคลุมและมีประสิทธิภาพสูงสำหรับงานเหล่านี้

---

## 56.1 sort และ sort!

```crystal
# sort พื้นฐาน
numbers = [5, 3, 1, 4, 2]
puts numbers.sort.inspect   # => [1, 2, 3, 4, 5] (ascending)

# sort! (in-place)
numbers.sort!
puts numbers.inspect        # => [1, 2, 3, 4, 5]

# sort กับ block (custom comparison)
puts numbers.sort { |a, b| b <=> a }.inspect  # => [5, 4, 3, 2, 1] (descending)

# sort strings
words = ["banana", "apple", "cherry", "date"]
puts words.sort.inspect
# => ["apple", "banana", "cherry", "date"]

# sort ตาม criteria ที่กำหนด
puts words.sort { |a, b| a.size <=> b.size }.inspect
# => ["date", "apple", "banana", "cherry"]

# sort บน Hash (ได้ array of pairs)
scores = {"Charlie" => 88, "Alice" => 95, "Bob" => 72}
sorted_scores = scores.sort_by { |_, score| -score }
sorted_scores.each { |name, score| puts "  #{name}: #{score}" }
```

---

## 56.2 sort_by

```crystal
# sort_by ใช้งานง่ายกว่า sort กับ block
users = [
  {name: "Charlie", age: 35, salary: 75000},
  {name: "Alice", age: 25, salary: 95000},
  {name: "Bob", age: 30, salary: 65000},
  {name: "Dave", age: 28, salary: 85000}
]

# เรียงตาม age
by_age = users.sort_by { |u| u[:age] }
by_age.each { |u| puts "  #{u[:name]}: #{u[:age]}" }
# Alice: 25
# Dave: 28
# Bob: 30
# Charlie: 35

# เรียง descending ด้วย negation
by_salary_desc = users.sort_by { |u| -u[:salary] }
by_salary_desc.each { |u| puts "  #{u[:name]}: $#{u[:salary]}" }
# Alice: $95000
# Dave: $85000
# Charlie: $75000
# Bob: $65000

# เรียงหลายเงื่อนไข (multi-criteria sort)
# เรียงตาม department แล้วตาม salary ลดลง
employees = [
  {name: "Alice", dept: "Engineering", salary: 95000},
  {name: "Bob", dept: "Marketing", salary: 65000},
  {name: "Charlie", dept: "Engineering", salary: 80000},
  {name: "Dave", dept: "Marketing", salary: 75000}
]

by_dept_salary = employees.sort_by { |e| [e[:dept], -e[:salary]] }
by_dept_salary.each { |e| puts "  #{e[:dept]}: #{e[:name]} ($#{e[:salary]})" }
# Engineering: Alice ($95000)
# Engineering: Charlie ($80000)
# Marketing: Dave ($75000)
# Marketing: Bob ($65000)
```

---

## 56.3 sort_by! - In-Place Sort

```crystal
products = [
  {name: "Cherry", price: 3.49},
  {name: "Apple", price: 1.99},
  {name: "Banana", price: 0.99}
]

# sort_by! แก้ไข array เดิม
products.sort_by! { |p| p[:price] }
products.each { |p| puts "  #{p[:name]}: $#{p[:price]}" }
# Banana: $0.99
# Apple: $1.99
# Cherry: $3.49
```

---

## 56.4 min_by และ max_by

```crystal
data = [
  {name: "Alpha", score: 85, time: 120},
  {name: "Beta", score: 92, time: 95},
  {name: "Gamma", score: 78, time: 105},
  {name: "Delta", score: 92, time: 110}
]

# min_by
lowest = data.min_by { |d| d[:score] }
puts "Lowest score: #{lowest[:name]} (#{lowest[:score]})"
# => Lowest score: Gamma (78)

# max_by
highest = data.max_by { |d| d[:score] }
puts "Highest score: #{highest[:name]} (#{highest[:score]})"
# => Highest score: Beta (92)

# min_by กับหลาย criteria
# หาคนที่มี score สูงที่สุด แต่ time น้อยที่สุด (best overall)
best = data.min_by { |d| [-d[:score], d[:time]] }
puts "Best: #{best[:name]} (score: #{best[:score]}, time: #{best[:time]})"
# => Best: Beta (score: 92, time: 95)
```

---

## 56.5 min_of และ max_of

```crystal
# min_of และ max_of คล้ายกับ min_by/max_by
# แต่คืนค่าที่ extract แทนที่จะคืน element

data = [{value: 10}, {value: 3}, {value: 7}, {value: 15}, {value: 1}]

# min_of
min_val = data.min_of { |d| d[:value] }
puts "Min value: #{min_val}"  # => 1

# max_of
max_val = data.max_of { |d| d[:value] }
puts "Max value: #{max_val}"  # => 15

# ต่างจาก min_by ตรงที่:
# min_by คืน {value: 1} (element)
# min_of คืน 1 (extracted value)

numbers = [3, 1, 4, 1, 5, 9, 2, 6]
puts numbers.min_of { |n| n }    # => 1 (เหมือน min)
puts numbers.max_of { |n| n }    # => 9 (เหมือน max)
puts numbers.min_of { |n| n.abs }  # => 1
```

---

## 56.6 minmax และ minmax_by

```crystal
numbers = [5, 3, 8, 1, 9, 2, 7]

# minmax คืน {min, max} tuple
min_val, max_val = numbers.minmax
puts "Min: #{min_val}, Max: #{max_val}"  # => Min: 1, Max: 9

# minmax_by
products = [
  {name: "A", price: 9.99},
  {name: "B", price: 1.99},
  {name: "C", price: 5.99}
]

cheapest, most_expensive = products.minmax_by { |p| p[:price] }
puts "Cheapest: #{cheapest[:name]} ($#{cheapest[:price]})"
puts "Most expensive: #{most_expensive[:name]} ($#{most_expensive[:price]})"

# ใช้กับ string ความยาว
words = ["hello", "hi", "hey", "greetings", "yo"]
shortest, longest = words.minmax_by(&.size)
puts "Shortest: #{shortest}"  # => yo
puts "Longest: #{longest}"    # => greetings
```

---

## 56.7 partition

```crystal
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# partition แบ่งออกเป็น 2 arrays
evens, odds = numbers.partition { |n| n.even? }
puts "Evens: #{evens.inspect}"  # => [2, 4, 6, 8, 10]
puts "Odds: #{odds.inspect}"    # => [1, 3, 5, 7, 9]

# partition กับ complex condition
students = [
  {name: "Alice", score: 92},
  {name: "Bob", score: 68},
  {name: "Charlie", score: 85},
  {name: "Dave", score: 72},
  {name: "Eve", score: 95}
]

passing, failing = students.partition { |s| s[:score] >= 75 }
puts "Passing: #{passing.map { |s| s[:name] }.join(", ")}"
puts "Failing: #{failing.map { |s| s[:name] }.join(", ")}"
# Passing: Alice, Charlie, Eve
# Failing: Bob, Dave

# ประหยัดกว่า 2 select calls
# (เดิม: เรียก select 2 ครั้ง traverse 2 ครั้ง)
# (partition: traverse 1 ครั้ง)
```

---

## 56.8 group_by

```crystal
items = ["apple", "banana", "avocado", "blueberry", "cherry", "cranberry"]

# group_by ตัวอักษรแรก
by_first_letter = items.group_by { |item| item[0] }
by_first_letter.each do |letter, words|
  puts "#{letter}: #{words.join(", ")}"
end
# a: apple, avocado
# b: banana, blueberry
# c: cherry, cranberry

# group_by ความยาว
by_length = items.group_by(&.size)
by_length.sort_by { |len, _| len }.each do |len, words|
  puts "Length #{len}: #{words.join(", ")}"
end

# group_by บน Array of objects
transactions = [
  {date: "2024-01", amount: 100.0},
  {date: "2024-01", amount: 200.0},
  {date: "2024-02", amount: 150.0},
  {date: "2024-02", amount: 300.0},
  {date: "2024-03", amount: 250.0}
]

monthly = transactions.group_by { |t| t[:date] }
  .transform_values { |txs| txs.sum { |t| t[:amount] } }

monthly.sort.each { |month, total| puts "  #{month}: $#{total}" }
```

---

## 56.9 Stable Sort

Crystal's sort เป็น stable sort ซึ่งหมายความว่า elements ที่เท่ากันจะรักษาลำดับเดิม

```crystal
students = [
  {name: "Alice", grade: "A"},
  {name: "Bob", grade: "B"},
  {name: "Charlie", grade: "A"},
  {name: "Dave", grade: "B"},
  {name: "Eve", grade: "A"}
]

# Stable sort: Alice, Charlie, Eve ยังเรียง A-Z เพราะลำดับเดิมของพวกเขา
by_grade = students.sort_by { |s| s[:grade] }
by_grade.each { |s| puts "  #{s[:grade]}: #{s[:name]}" }
# A: Alice
# A: Charlie
# A: Eve
# B: Bob
# B: Dave

# ถ้าต้องการ unstable sort (เร็วกว่าในบางกรณี)
# Crystal ใช้ unstable sort ใน sort_by สำหรับ performance
```

---

## 56.10 Custom Sort Functions

```crystal
# Sorting กับ complex objects

class Employee
  include Comparable(Employee)
  
  getter name : String
  getter department : String
  getter salary : Float64
  getter years : Int32
  
  def initialize(@name : String, @department : String, @salary : Float64, @years : Int32)
  end
  
  # Default: sort by name
  def <=>(other : Employee) : Int32
    @name <=> other.name
  end
  
  def to_s(io : IO) : Nil
    io << "#{@name} (#{@department}, $#{@salary}, #{@years}yr)"
  end
end

employees = [
  Employee.new("Charlie", "Engineering", 95000.0, 5),
  Employee.new("Alice", "Marketing", 75000.0, 3),
  Employee.new("Bob", "Engineering", 85000.0, 2),
  Employee.new("Dave", "HR", 65000.0, 7)
]

# Default sort (by name)
puts "By name:"
employees.sort.each { |e| puts "  #{e}" }

# Custom sort
puts "\nBy salary (desc):"
employees.sort_by { |e| -e.salary }.each { |e| puts "  #{e}" }

puts "\nBy department then salary:"
employees.sort_by { |e| [e.department, -e.salary] }.each { |e| puts "  #{e}" }
```

---

## 56.11 ตัวอย่างในชีวิตจริง: Leaderboard

```crystal
class Leaderboard
  record Entry, player : String, score : Int32, time : Float64
  
  def initialize
    @entries = [] of Entry
  end
  
  def add(player : String, score : Int32, time : Float64)
    # ลบ entry เก่าของ player นี้
    @entries.reject! { |e| e.player == player }
    @entries << Entry.new(player, score, time)
    sort!
  end
  
  def rank_of(player : String) : Int32?
    @entries.find_index { |e| e.player == player }.try { |i| i + 1 }
  end
  
  def top(n : Int32) : Array(Entry)
    @entries.first(n)
  end
  
  def all : Array(Entry)
    @entries
  end
  
  private def sort!
    @entries.sort_by! { |e| [-e.score, e.time] }
  end
end

lb = Leaderboard.new
lb.add("Alice", 1000, 120.5)
lb.add("Bob", 850, 95.2)
lb.add("Charlie", 1000, 110.3)  # Same score as Alice but faster
lb.add("Dave", 950, 130.0)

puts "=== Leaderboard ==="
lb.all.each_with_index do |entry, i|
  puts "#{i+1}. #{entry.player}: #{entry.score} pts (#{entry.time}s)"
end

puts "\nAlice's rank: #{lb.rank_of("Alice")}"  # => 2 (Charlie is faster)
puts "\nTop 3:"
lb.top(3).each_with_index do |e, i|
  puts "  #{i+1}. #{e.player}"
end

# Update score
lb.add("Bob", 1050, 100.0)
puts "\nAfter Bob's update:"
lb.all.each_with_index do |entry, i|
  puts "#{i+1}. #{entry.player}: #{entry.score}"
end
```

---

## 56.12 Advanced Filtering

```crystal
data = (1..100).to_a

# Multiple filters
result = data
  .select { |n| n % 3 == 0 }   # Divisible by 3
  .select { |n| n % 5 == 0 }   # Divisible by 5
puts result.inspect  # => [15, 30, 45, 60, 75, 90]

# เร็วกว่าด้วย single filter
result2 = data.select { |n| n % 15 == 0 }
puts result2.inspect  # => [15, 30, 45, 60, 75, 90]

# Hierarchical filtering
employees = [
  {name: "Alice", dept: "Eng", level: 3, salary: 95000},
  {name: "Bob", dept: "Eng", level: 1, salary: 65000},
  {name: "Charlie", dept: "HR", level: 2, salary: 70000},
  {name: "Dave", dept: "Eng", level: 2, salary: 80000},
  {name: "Eve", dept: "HR", level: 3, salary: 85000}
]

# Filter: Senior (level >= 2) Engineering employees, sorted by salary
senior_engineers = employees
  .select { |e| e[:dept] == "Eng" && e[:level] >= 2 }
  .sort_by { |e| -e[:salary] }

puts "Senior Engineers:"
senior_engineers.each { |e| puts "  L#{e[:level]} #{e[:name]}: $#{e[:salary]}" }
# L3 Alice: $95000
# L2 Dave: $80000
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Student Ranking System

```crystal
students = [
  {name: "Alice", scores: [85, 92, 78], subject: "Math"},
  {name: "Bob", scores: [65, 70, 68], subject: "Math"},
  {name: "Charlie", scores: [92, 95, 88], subject: "Science"},
  {name: "Dave", scores: [72, 68, 75], subject: "Math"},
  {name: "Eve", scores: [88, 91, 85], subject: "Science"}
]

# คำนวณ average
with_avg = students.map { |s|
  avg = s[:scores].sum.to_f / s[:scores].size
  s.merge({avg: avg})
}

# เรียงตาม subject แล้วตาม avg
ranked = with_avg.sort_by { |s| [s[:subject], -s[:avg]] }

# แสดงผลแยกตาม subject
ranked.group_by { |s| s[:subject] }.each do |subject, group|
  puts "#{subject}:"
  group.each_with_index do |s, i|
    puts "  #{i+1}. #{s[:name]}: #{s[:avg].round(1)}"
  end
end

# Statistics per subject
puts "\nSubject stats:"
with_avg.group_by { |s| s[:subject] }.each do |subject, group|
  avgs = group.map { |s| s[:avg] }
  puts "  #{subject}: class avg = #{(avgs.sum / avgs.size).round(1)}"
end
```

### แบบฝึกหัดที่ 2: Product Catalog

```crystal
products = [
  {id: 1, name: "Laptop", price: 999.0, category: "electronics", in_stock: true, rating: 4.5},
  {id: 2, name: "Book", price: 29.0, category: "education", in_stock: true, rating: 4.8},
  {id: 3, name: "Mouse", price: 39.0, category: "electronics", in_stock: false, rating: 4.2},
  {id: 4, name: "Desk", price: 299.0, category: "furniture", in_stock: true, rating: 4.0},
  {id: 5, name: "Course", price: 199.0, category: "education", in_stock: true, rating: 4.9},
  {id: 6, name: "Monitor", price: 499.0, category: "electronics", in_stock: true, rating: 4.6}
]

# 1. In-stock products sorted by rating
in_stock_sorted = products
  .select { |p| p[:in_stock] }
  .sort_by { |p| -p[:rating] }

puts "In-stock by rating:"
in_stock_sorted.each { |p| puts "  #{p[:name]}: #{p[:rating]}⭐" }

# 2. By category, sorted by price
by_category = products
  .select { |p| p[:in_stock] }
  .group_by { |p| p[:category] }

puts "\nBy category:"
by_category.each do |cat, items|
  sorted = items.sort_by { |p| p[:price] }
  puts "  #{cat}: #{sorted.map { |p| "#{p[:name]}($#{p[:price]})" }.join(", ")}"
end

# 3. Best value (high rating, low price)
best_value = products
  .select { |p| p[:in_stock] }
  .sort_by { |p| [-p[:rating], p[:price]] }
  .first(3)

puts "\nBest value:"
best_value.each { |p| puts "  #{p[:name]}: $#{p[:price]}, #{p[:rating]}⭐" }

# 4. partition: budget vs premium
budget, premium = products.partition { |p| p[:price] < 100.0 }
puts "\nBudget (<$100): #{budget.map { |p| p[:name] }.join(", ")}"
puts "Premium (>=$100): #{premium.map { |p| p[:name] }.join(", ")}"
```

---

## สรุป

| Method | คำอธิบาย | คืน |
|--------|---------|-----|
| `sort` | เรียงใหม่ | Array ใหม่ |
| `sort!` | เรียงใน-place | self |
| `sort_by` | เรียงตาม key | Array ใหม่ |
| `sort_by!` | เรียงตาม key in-place | self |
| `min_by` | หา element ที่ค่าน้อยสุด | element |
| `max_by` | หา element ที่ค่ามากสุด | element |
| `min_of` | หาค่าน้อยสุด | ค่า |
| `max_of` | หาค่ามากสุด | ค่า |
| `minmax` | หา {min, max} | Tuple |
| `minmax_by` | หา {min, max} element | Tuple |
| `partition` | แบ่งเป็น 2 groups | {Array, Array} |
| `group_by` | จัดกลุ่ม | Hash |

**เคล็ดลับ:**
- ใช้ `sort_by` แทน `sort` เมื่อต้องการ custom key
- ใช้ `[-primary, secondary]` สำหรับ multi-criteria sort
- `partition` มีประสิทธิภาพดีกว่า `select` + `reject` คู่กัน
- Crystal ใช้ stable sort

---

*ต่อไป: Part 57 - Lazy Iterators*
