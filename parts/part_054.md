# Part 54: Hash Methods ขั้นสูง ใน Crystal

## บทนำ

Hash ใน Crystal มี methods ขั้นสูงที่ทรงพลังสำหรับการวิเคราะห์และแปลงข้อมูล บทนี้จะครอบคลุม group_by, tally, transform_*, filter_map และอื่นๆ

---

## 54.1 group_by

group_by แบ่งข้อมูลออกเป็นกลุ่มตาม key ที่กำหนด

```crystal
students = [
  {name: "Alice", grade: "A", year: 2},
  {name: "Bob", grade: "B", year: 1},
  {name: "Charlie", grade: "A", year: 1},
  {name: "Dave", grade: "C", year: 3},
  {name: "Eve", grade: "A", year: 2},
  {name: "Frank", grade: "B", year: 1}
]

# group_by grade
by_grade = students.group_by { |s| s[:grade] }
by_grade.each do |grade, group|
  names = group.map { |s| s[:name] }.join(", ")
  puts "Grade #{grade}: #{names}"
end
# Grade A: Alice, Charlie, Eve
# Grade B: Bob, Frank
# Grade C: Dave

# group_by บน Hash
scores = {"Alice" => 95, "Bob" => 87, "Charlie" => 92, "Dave" => 78, "Eve" => 95}

by_score = scores.group_by { |name, score| score >= 90 ? "high" : "low" }
by_score.each do |category, entries|
  puts "#{category}: #{entries.map { |n, _| n }.join(", ")}"
end
# high: Alice, Charlie, Eve
# low: Bob, Dave

# Nested group_by
transactions = [
  {date: "2024-01", category: "food", amount: 500.0},
  {date: "2024-01", category: "transport", amount: 300.0},
  {date: "2024-02", category: "food", amount: 600.0},
  {date: "2024-02", category: "food", amount: 400.0},
  {date: "2024-02", category: "entertainment", amount: 200.0}
]

# Group by date, then by category
by_date = transactions.group_by { |t| t[:date] }
by_date.each do |date, items|
  puts "#{date}:"
  by_cat = items.group_by { |t| t[:category] }
  by_cat.each do |cat, cat_items|
    total = cat_items.sum { |t| t[:amount] }
    puts "  #{cat}: $#{total}"
  end
end
```

---

## 54.2 tally บน Hash

```crystal
# tally นับความถี่จาก array
fruits = ["apple", "banana", "apple", "cherry", "banana", "apple"]
freq = fruits.tally
puts freq.inspect
# => {"apple" => 3, "banana" => 2, "cherry" => 1}

# tally บน Hash (ผ่าน values หรือ keys)
categories = {"item1" => "food", "item2" => "drink", "item3" => "food",
              "item4" => "drink", "item5" => "food"}

# นับ values
category_counts = categories.values.tally
puts category_counts.inspect  # => {"food" => 3, "drink" => 2}

# tally กับ map
words = "the quick brown fox jumps over the lazy dog".split
length_tally = words.map(&.size).tally
puts "Word lengths: #{length_tally.sort.inspect}"

# Most common
most_common = freq.max_by { |_, count| count }
puts "Most common: #{most_common[0]} (#{most_common[1]}x)"  # => apple (3x)
```

---

## 54.3 transform_keys

```crystal
# transform_keys สร้าง Hash ใหม่โดยแปลง keys
h = {"name" => "Alice", "age" => 30, "city" => "Bangkok"}

# แปลง key เป็น Symbol
symbolized = h.transform_keys(&.to_sym)
puts symbolized.inspect
# => {:name => "Alice", :age => 30, :city => "Bangkok"}
puts symbolized[:name]  # => "Alice"

# แปลง key เป็น uppercase
uppercased = h.transform_keys(&.upcase)
puts uppercased.inspect
# => {"NAME" => "Alice", "AGE" => 30, "CITY" => "Bangkok"}

# Prefix keys
prefixed = h.transform_keys { |k| "user_#{k}" }
puts prefixed.inspect
# => {"user_name" => "Alice", "user_age" => 30, "user_city" => "Bangkok"}

# transform_keys! (in-place)
mutable = {"x" => 1, "y" => 2, "z" => 3}
mutable.transform_keys!(&.upcase)
puts mutable.inspect  # => {"X" => 1, "Y" => 2, "Z" => 3}
```

---

## 54.4 transform_values

```crystal
prices = {"apple" => 1.99, "banana" => 0.99, "cherry" => 3.49}

# แปลง values
discounted = prices.transform_values { |price| (price * 0.9).round(2) }
puts discounted.inspect
# => {"apple" => 1.79, "banana" => 0.89, "cherry" => 3.14}

# แปลงเป็น string
as_strings = prices.transform_values { |price| "$#{price}" }
puts as_strings.inspect
# => {"apple" => "$1.99", "banana" => "$0.99", "cherry" => "$3.49"}

# transform_values! (in-place)
inventory = {"A" => 10, "B" => 5, "C" => 0}
inventory.transform_values! { |qty| qty * 2 }
puts inventory.inspect  # => {"A" => 20, "B" => 10, "C" => 0}

# Double transform (both keys and values)
config = {"HOST" => "localhost", "PORT" => "8080"}
normalized = config
  .transform_keys(&.downcase)
  .transform_values { |v| v == v.to_i?.to_s ? v.to_i : v }
puts normalized.inspect
# => {"host" => "localhost", "port" => "8080"}
```

---

## 54.5 filter_map

```crystal
# filter_map = select + map ในขั้นตอนเดียว
products = {
  "A001" => {name: "Apple", price: 1.99, stock: 10},
  "A002" => {name: "Banana", price: 0.99, stock: 0},
  "A003" => {name: "Cherry", price: 3.49, stock: 5},
  "A004" => {name: "Date", price: 5.99, stock: 0},
  "A005" => {name: "Elderberry", price: 8.99, stock: 2}
}

# เฉพาะสินค้าที่มี stock และ price > 2
available_expensive = products.filter_map do |id, product|
  if product[:stock] > 0 && product[:price] > 2.0
    "#{id}: #{product[:name]} ($#{product[:price]})"
  end
end

puts available_expensive.inspect
# => ["A003: Cherry ($3.49)", "A005: Elderberry ($8.99)"]

# filter_map บน array of hashes
records = [
  {"name" => "Alice", "score" => "95"},
  {"name" => "Bob", "score" => "N/A"},
  {"name" => "Charlie", "score" => "87"},
  {"name" => "Dave", "score" => "invalid"}
]

valid_scores = records.filter_map do |r|
  score = r["score"].to_i?
  score ? {r["name"], score} : nil
end
puts valid_scores.inspect
# => [{"Alice", 95}, {"Charlie", 87}]
```

---

## 54.6 any?, all?, none?

```crystal
inventory = {"apples" => 50, "bananas" => 0, "cherries" => 25, "dates" => 0}

# any?: มีอย่างน้อยหนึ่งที่ตรงเงื่อนไข
puts inventory.any? { |_, qty| qty > 0 }      # => true
puts inventory.any? { |_, qty| qty > 100 }    # => false

# all?: ทุกตัวตรงเงื่อนไข
puts inventory.all? { |_, qty| qty >= 0 }     # => true
puts inventory.all? { |_, qty| qty > 0 }      # => false

# none?: ไม่มีตัวใดตรงเงื่อนไข
puts inventory.none? { |_, qty| qty < 0 }     # => true
puts inventory.none? { |name, _| name.size > 10 }  # => false (cherries = 8, ok)

# count: นับที่ตรงเงื่อนไข
in_stock = inventory.count { |_, qty| qty > 0 }
puts "In stock: #{in_stock}"  # => 2

# สาธิต: Permission checking
user_permissions = {"read" => true, "write" => false, "admin" => false}

puts user_permissions.any? { |_, has| has }    # => true (has read)
puts user_permissions.all? { |_, has| has }    # => false (write/admin false)
puts user_permissions.none? { |k, has| k == "admin" && has }  # => true
```

---

## 54.7 count

```crystal
h = {"a" => 1, "b" => 2, "c" => 3, "d" => 4, "e" => 5}

# count ทั้งหมด
puts h.count   # => 5

# count ด้วย condition
puts h.count { |_, v| v.even? }  # => 2
puts h.count { |k, _| k < "c" }  # => 2

# sum, min_by, max_by
scores = {"Alice" => 95, "Bob" => 87, "Charlie" => 92, "Dave" => 78}

total = scores.sum { |_, score| score }
puts "Total: #{total}"  # => 352

avg = total.to_f / scores.size
puts "Average: #{avg}"  # => 88.0

min_name, min_score = scores.min_by { |_, s| s }
max_name, max_score = scores.max_by { |_, s| s }
puts "Min: #{min_name} (#{min_score})"  # => Dave (78)
puts "Max: #{max_name} (#{max_score})"  # => Alice (95)
```

---

## 54.8 min_by และ max_by

```crystal
products = {
  "laptop" => 999.99,
  "mouse" => 29.99,
  "keyboard" => 79.99,
  "monitor" => 499.99,
  "headset" => 149.99
}

# min_by
cheapest = products.min_by { |_, price| price }
puts "Cheapest: #{cheapest[0]} ($#{cheapest[1]})"
# => mouse ($29.99)

# max_by
most_expensive = products.max_by { |_, price| price }
puts "Most expensive: #{most_expensive[0]} ($#{most_expensive[1]})"
# => laptop ($999.99)

# minmax_by
min, max = products.minmax_by { |_, price| price }
puts "Price range: $#{min[1]} - $#{max[1]}"
# => $29.99 - $999.99

# Sort by value
by_price = products.sort_by { |_, price| price }
puts "\nProducts by price:"
by_price.each { |name, price| puts "  #{name}: $#{price}" }
```

---

## 54.9 sum

```crystal
orders = {
  "order_1" => {total: 150.0, items: 3},
  "order_2" => {total: 89.5, items: 1},
  "order_3" => {total: 299.0, items: 5},
  "order_4" => {total: 45.0, items: 2}
}

# sum ค่าทั้งหมด
total_revenue = orders.sum { |_, order| order[:total] }
puts "Total revenue: $#{total_revenue}"  # => $583.5

total_items = orders.sum { |_, order| order[:items] }
puts "Total items sold: #{total_items}"  # => 11

# Average order value
avg_order = total_revenue / orders.size
puts "Average order: $#{avg_order.round(2)}"  # => $145.88

# Weighted sum
inventory = {"A" => {qty: 10, price: 5.0}, "B" => {qty: 20, price: 3.0}}
total_value = inventory.sum { |_, item| item[:qty] * item[:price] }
puts "Total inventory value: $#{total_value}"  # => $110.0
```

---

## 54.10 flat_map บน Hash

```crystal
students_skills = {
  "Alice" => ["Crystal", "Ruby", "Python"],
  "Bob" => ["JavaScript", "TypeScript"],
  "Charlie" => ["Go", "Rust", "Crystal"]
}

# flat_map สร้าง all skill-student pairs
all_pairs = students_skills.flat_map { |student, skills|
  skills.map { |skill| {skill, student} }
}
puts all_pairs.size  # => 8

# หา students ที่รู้ Crystal
crystal_devs = all_pairs
  .select { |skill, _| skill == "Crystal" }
  .map { |_, student| student }
puts "Crystal devs: #{crystal_devs.inspect}"  # => ["Alice", "Charlie"]

# skill frequency
skill_freq = all_pairs.map { |skill, _| skill }.tally
puts "Skills: #{skill_freq.sort_by { |_, c| -c }.inspect}"
```

---

## 54.11 merge กับ block

```crystal
# merge ธรรมดา
h1 = {"a" => 1, "b" => 2}
h2 = {"b" => 3, "c" => 4}

# h2 overrides h1 ถ้า key ซ้ำ
puts h1.merge(h2).inspect  # => {"a" => 1, "b" => 3, "c" => 4}

# merge กับ block: resolve conflicts
# block รับ (key, old_value, new_value)
merged_sum = h1.merge(h2) { |key, old, new_val| old + new_val }
puts merged_sum.inspect  # => {"a" => 1, "b" => 5, "c" => 4}

merged_keep_old = h1.merge(h2) { |key, old, _| old }
puts merged_keep_old.inspect  # => {"a" => 1, "b" => 2, "c" => 4}

# ตัวอย่าง: Merge configurations
default_config = {
  "host" => "localhost",
  "port" => "8080",
  "debug" => "false",
  "max_conn" => "100"
}

user_config = {
  "host" => "api.example.com",
  "debug" => "true"
}

final_config = default_config.merge(user_config)
puts final_config.inspect
# => {"host" => "api.example.com", "port" => "8080", "debug" => "true", "max_conn" => "100"}

# ตัวอย่าง: สะสม scores
all_scores = [
  {"Alice" => 10, "Bob" => 8},
  {"Alice" => 7, "Charlie" => 9},
  {"Bob" => 6, "Charlie" => 10}
]

combined = all_scores.reduce({} of String => Int32) do |acc, round|
  acc.merge(round) { |_, s1, s2| s1 + s2 }
end
puts combined.inspect  # => {"Alice" => 17, "Bob" => 14, "Charlie" => 19}
```

---

## 54.12 ตัวอย่างในชีวิตจริง: Analytics Dashboard

```crystal
# Sales analytics
sales_data = [
  {date: "2024-01-01", product: "laptop", category: "electronics", revenue: 999.0, units: 1},
  {date: "2024-01-02", product: "phone", category: "electronics", revenue: 599.0, units: 2},
  {date: "2024-01-03", product: "book", category: "education", revenue: 29.0, units: 5},
  {date: "2024-01-04", product: "laptop", category: "electronics", revenue: 999.0, units: 1},
  {date: "2024-01-05", product: "course", category: "education", revenue: 199.0, units: 3},
  {date: "2024-01-06", product: "headset", category: "electronics", revenue: 149.0, units: 4},
  {date: "2024-01-07", product: "book", category: "education", revenue: 29.0, units: 8}
]

# 1. Revenue by category
category_revenue = sales_data.group_by { |s| s[:category] }
  .transform_values { |sales| sales.sum { |s| s[:revenue] * s[:units] } }

puts "Revenue by category:"
category_revenue.sort_by { |_, r| -r }.each do |cat, rev|
  puts "  #{cat}: $#{rev}"
end

# 2. Top products by units
product_units = sales_data.each_with_object(Hash(String, Int32).new(0)) do |s, h|
  h[s[:product]] += s[:units]
end

puts "\nTop products by units:"
product_units.sort_by { |_, u| -u }.first(3).each do |product, units|
  puts "  #{product}: #{units} units"
end

# 3. Daily revenue
daily = sales_data.group_by { |s| s[:date] }
  .transform_values { |sales| sales.sum { |s| s[:revenue] * s[:units] } }

puts "\nDaily revenue:"
daily.sort_by { |d, _| d }.each { |date, rev| puts "  #{date}: $#{rev}" }

# 4. Category percentage
total = category_revenue.values.sum
puts "\nCategory breakdown:"
category_revenue.each do |cat, rev|
  pct = (rev / total * 100).round(1)
  puts "  #{cat}: #{pct}%"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Student Grades Analyzer

```crystal
students = {
  "Alice" => [85, 92, 78, 96, 88],
  "Bob" => [72, 68, 75, 80, 71],
  "Charlie" => [95, 98, 92, 97, 99],
  "Dave" => [60, 55, 65, 70, 58],
  "Eve" => [88, 90, 85, 92, 87]
}

# คำนวณสถิติสำหรับแต่ละ student
def grade_stats(scores : Array(Int32)) : NamedTuple(avg: Float64, min: Int32, max: Int32, grade: String)
  avg = scores.sum.to_f / scores.size
  grade = case avg
    when (90..) then "A"
    when (80..) then "B"
    when (70..) then "C"
    when (60..) then "D"
    else "F"
    end
  {avg: avg.round(2), min: scores.min, max: scores.max, grade: grade}
end

# คำนวณสถิติ
stats = students.transform_values { |scores| grade_stats(scores) }

# แสดงผล
puts "Student Report:"
stats.sort_by { |_, s| -s[:avg] }.each do |name, s|
  puts "  #{name}: avg=#{s[:avg]}, grade=#{s[:grade]}, range=#{s[:min]}-#{s[:max]}"
end

# Group by grade
by_grade = stats.group_by { |_, s| s[:grade] }
puts "\nGrade distribution:"
["A", "B", "C", "D", "F"].each do |grade|
  students_in_grade = by_grade[grade]?.try { |sg| sg.map { |n, _| n }.join(", ") } || "none"
  puts "  #{grade}: #{students_in_grade}"
end
```

### แบบฝึกหัดที่ 2: Inventory Management

```crystal
inventory = {
  "A001" => {name: "Laptop", qty: 15, price: 999.0, category: "electronics"},
  "A002" => {name: "Mouse", qty: 50, price: 29.0, category: "electronics"},
  "A003" => {name: "Desk", qty: 8, price: 299.0, category: "furniture"},
  "A004" => {name: "Chair", qty: 0, price: 199.0, category: "furniture"},
  "A005" => {name: "Monitor", qty: 20, price: 499.0, category: "electronics"}
}

# สินค้าที่หมดสต็อก
out_of_stock = inventory.select { |_, item| item[:qty] == 0 }
puts "Out of stock: #{out_of_stock.keys.inspect}"

# มูลค่าสินค้าแยกตาม category
category_value = inventory
  .reject { |_, item| item[:qty] == 0 }
  .group_by { |_, item| item[:category] }
  .transform_values { |items|
    items.sum { |_, item| item[:qty] * item[:price] }
  }

puts "\nInventory value by category:"
category_value.sort_by { |_, v| -v }.each do |cat, value|
  puts "  #{cat}: $#{value}"
end

# อัพเดทสต็อก
updated = inventory.transform_values do |item|
  if item[:qty] < 10
    {name: item[:name], qty: item[:qty] + 20, price: item[:price], category: item[:category]}
  else
    item
  end
end

puts "\nAfter restocking:"
updated.each { |id, item| puts "  #{id}: #{item[:name]} qty=#{item[:qty]}" }
```

---

## สรุป

Hash methods ขั้นสูงใน Crystal:

| Method | คำอธิบาย |
|--------|---------|
| `group_by` | แบ่งกลุ่มตาม key |
| `tally` | นับความถี่ (ใน array) |
| `transform_keys` | แปลง keys |
| `transform_values` | แปลง values |
| `filter_map` | select + map |
| `any?/all?/none?` | boolean checks |
| `count` | นับที่ตรงเงื่อนไข |
| `min_by/max_by` | หา min/max |
| `sum` | รวมค่า |
| `flat_map` | flatten + map |
| `merge` with block | merge พร้อม conflict resolution |
| `sort_by` | เรียง |

---

*ต่อไป: Part 55 - Map, Select, Reject, Reduce*
