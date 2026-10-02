# Part 58: Comprehensions ใน Crystal

## บทนำ

Crystal ไม่มี "list comprehension" syntax เหมือน Python (เช่น `[x*2 for x in range(10) if x % 2 == 0]`) แต่เราสามารถบรรลุผลลัพธ์เดียวกันได้ผ่าน `map`, `select`, `flat_map` และ `each_with_object` ในบทนี้เราจะเรียนรู้ patterns เหล่านี้และเปรียบเทียบกับ Python

---

## 58.1 เปรียบเทียบ Python vs Crystal

### Python List Comprehension

```python
# Python
squares = [x**2 for x in range(10)]
even_squares = [x**2 for x in range(10) if x % 2 == 0]
flat = [x*y for x in range(3) for y in range(3)]
nested = [[x*y for y in range(3)] for x in range(3)]
```

### Crystal Equivalent

```crystal
# Crystal
squares = (0..9).map { |x| x ** 2 }
puts squares.inspect
# => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

even_squares = (0..9).select { |x| x.even? }.map { |x| x ** 2 }
puts even_squares.inspect
# => [0, 4, 16, 36, 64]

# หรือใช้ filter_map
even_squares2 = (0..9).filter_map { |x| x ** 2 if x.even? }
puts even_squares2.inspect
# => [0, 4, 16, 36, 64]

# Nested comprehension
flat = (0..2).flat_map { |x| (0..2).map { |y| x * y } }
puts flat.inspect
# => [0, 0, 0, 0, 1, 2, 0, 2, 4]

nested = (0..2).map { |x| (0..2).map { |y| x * y } }
puts nested.inspect
# => [[0, 0, 0], [0, 1, 2], [0, 2, 4]]
```

---

## 58.2 map/select Patterns

```crystal
# Pattern 1: Simple transformation
words = ["hello", "world", "crystal"]
upper = words.map(&.upcase)
puts upper.inspect  # => ["HELLO", "WORLD", "CRYSTAL"]

# Pattern 2: Filter then transform
numbers = (1..20).to_a
result = numbers.select { |n| n % 3 == 0 }.map { |n| "#{n} is divisible by 3" }
result.each { |s| puts s }
# 3 is divisible by 3
# 6 is divisible by 3
# ...

# Pattern 3: Transform then filter
students = [
  {name: "Alice", score: 85},
  {name: "Bob", score: 62},
  {name: "Charlie", score: 91}
]

honor_roll = students
  .map { |s| {name: s[:name], grade: s[:score] >= 90 ? "A" : s[:score] >= 80 ? "B" : "C"} }
  .select { |s| s[:grade] == "A" }
  .map { |s| s[:name] }

puts honor_roll.inspect  # => ["Charlie"]
```

---

## 58.3 flat_map สำหรับ Nested Comprehensions

```crystal
# Python: [f(x,y) for x in xs for y in ys]
# Crystal: xs.flat_map { |x| ys.map { |y| f(x, y) } }

# ตัวอย่าง: Cartesian product
xs = [1, 2, 3]
ys = [10, 20]

product = xs.flat_map { |x| ys.map { |y| x * y } }
puts product.inspect  # => [10, 20, 20, 40, 30, 60]

# ตัวอย่าง: สร้าง pairs
pairs = xs.flat_map { |x| ys.map { |y| {x, y} } }
puts pairs.inspect  # => [{1, 10}, {1, 20}, {2, 10}, {2, 20}, {3, 10}, {3, 20}]

# ตัวอย่าง: combinations with condition
# Python: [(x, y) for x in range(5) for y in range(5) if x != y]
combos = (0..4).flat_map { |x|
  (0..4).filter_map { |y| {x, y} if x != y }
}
puts "Unique pairs count: #{combos.size}"  # => 20
puts combos.first(5).inspect               # => [{0, 1}, {0, 2}, {0, 3}, {0, 4}, {1, 0}]
```

---

## 58.4 each_with_object Pattern

```crystal
# Python dict comprehension: {k: v for k, v in pairs}
# Crystal: pairs.each_with_object({}) { |(k, v), h| h[k] = v }

pairs = [{"a", 1}, {"b", 2}, {"c", 3}]
hash = pairs.each_with_object({} of String => Int32) do |(k, v), h|
  h[k] = v
end
puts hash.inspect  # => {"a" => 1, "b" => 2, "c" => 3}

# หรือใช้ to_h
hash2 = pairs.to_h
puts hash2.inspect

# Python: {word: len(word) for word in words}
words = ["apple", "banana", "cherry"]
word_lengths = words.each_with_object({} of String => Int32) do |word, h|
  h[word] = word.size
end
puts word_lengths.inspect  # => {"apple" => 5, "banana" => 6, "cherry" => 6}

# หรือสั้นกว่า
word_lengths2 = words.map { |w| {w, w.size} }.to_h
puts word_lengths2.inspect
```

---

## 58.5 filter_map Pattern

```crystal
# filter_map เป็น idiom ที่ใกล้เคียง comprehension with condition มากที่สุด
# Python: [f(x) for x in xs if condition(x)]
# Crystal: xs.filter_map { |x| f(x) if condition(x) }

# ตัวอย่าง 1: Parse และกรองพร้อมกัน
lines = ["1,Alice,A", "invalid", "2,Bob,B", "", "3,Charlie,A"]

parsed = lines.filter_map do |line|
  parts = line.split(",")
  {parts[0].to_i, parts[1], parts[2]} if parts.size == 3
end
puts parsed.inspect
# => [{1, "Alice", "A"}, {2, "Bob", "B"}, {3, "Charlie", "A"}]

# ตัวอย่าง 2: Extract และแปลง
data = [1, -2, 3, -4, 5, -6]
positive_squares = data.filter_map { |n| n * n if n > 0 }
puts positive_squares.inspect  # => [1, 9, 25]

# ตัวอย่าง 3: Nullable transformations
def safe_sqrt(n : Int32) : Float64?
  n >= 0 ? Math.sqrt(n.to_f) : nil
end

values = [-1, 4, -9, 16, 25]
roots = values.filter_map { |n| safe_sqrt(n) }
puts roots.inspect  # => [2.0, 4.0, 5.0]
```

---

## 58.6 Hash Comprehension Pattern

```crystal
# Python: {k: f(v) for k, v in d.items() if condition(v)}
# Crystal: hash.select { ... }.transform_values { ... }

prices = {"apple" => 1.99, "banana" => 0.99, "cherry" => 3.49, "date" => 5.99}

# Discounted prices for expensive items
discounted = prices
  .select { |_, price| price > 2.0 }
  .transform_values { |price| (price * 0.9).round(2) }

puts discounted.inspect
# => {"cherry" => 3.14, "date" => 5.39}

# หรือด้วย each_with_object
discounted2 = prices.each_with_object({} of String => Float64) do |(name, price), h|
  h[name] = (price * 0.9).round(2) if price > 2.0
end
puts discounted2.inspect

# Invert with condition
word_index = ["apple", "banana", "cherry"].each_with_object({} of String => Int32) do |word, h|
  h[word] = word.size
end
puts word_index.inspect  # => {"apple" => 5, "banana" => 6, "cherry" => 6}
```

---

## 58.7 Multi-level Comprehension

```crystal
# Python: [f(x,y,z) for x in xs for y in ys for z in zs if cond]

departments = ["Engineering", "Marketing"]
levels = [1, 2, 3]
locations = ["Bangkok", "Chiang Mai"]

# Generate all combinations
positions = departments.flat_map { |dept|
  levels.flat_map { |level|
    locations.map { |loc| "#{dept} L#{level} (#{loc})" }
  }
}

puts "Total positions: #{positions.size}"  # => 12
positions.first(4).each { |p| puts "  #{p}" }

# กับ condition
filtered_positions = departments.flat_map { |dept|
  levels.flat_map { |level|
    locations.filter_map { |loc|
      "#{dept} L#{level} (#{loc})" if !(dept == "Marketing" && level == 3)
    }
  }
}
puts "Filtered positions: #{filtered_positions.size}"  # => 10
```

---

## 58.8 Set Comprehension Pattern

```crystal
require "set"

# Python: {x**2 for x in range(10)}
# Crystal: (0..9).map { |x| x**2 }.to_set

squares_set = (0..9).map { |x| x ** 2 }.to_set
puts squares_set.to_a.sort.inspect
# => [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]

# Set ของ even squares
even_squares_set = (0..9).filter_map { |x| x ** 2 if x.even? }.to_set
puts even_squares_set.to_a.sort.inspect
# => [0, 4, 16, 36, 64]

# Union of two set comprehensions
set_a = (1..5).map { |n| n * 2 }.to_set     # {2, 4, 6, 8, 10}
set_b = (1..5).map { |n| n * 3 }.to_set     # {3, 6, 9, 12, 15}
union = set_a | set_b
puts union.to_a.sort.inspect
# => [2, 3, 4, 6, 8, 9, 10, 12, 15]
```

---

## 58.9 Generator Expression Pattern (Lazy)

```crystal
# Python generator expressions ใช้ () แทน []
# Python: (x**2 for x in range(10))  <- generator, lazy
# Crystal: (0..9).lazy.map { |x| x**2 }  <- lazy iterator

# Lazy computation
gen = (0..Int32::MAX).lazy
  .select { |n| n % 2 == 0 }  # evens
  .map { |n| n ** 2 }          # squared
  .take_while { |n| n < 100 }  # until 100

puts gen.to_a.inspect  # => [0, 4, 16, 36, 64]

# Lazy pipeline ที่ซับซ้อน
words = ["hello world", "crystal lang", "foo bar baz"]
result = words.lazy
  .flat_map { |sentence| sentence.split }
  .map { |word| word.upcase }
  .select { |word| word.size > 3 }
  .first(4)
puts result.inspect  # => ["HELLO", "WORLD", "CRYSTAL"]
```

---

## 58.10 Comparison Table

```crystal
# เปรียบเทียบ patterns ทั้งหมด

# 1. Basic: [f(x) for x in xs]
result1 = (1..5).map { |x| x * 2 }
puts result1.inspect  # => [2, 4, 6, 8, 10]

# 2. With condition: [f(x) for x in xs if cond(x)]
result2 = (1..10).filter_map { |x| x * 2 if x.odd? }
puts result2.inspect  # => [2, 6, 10, 14, 18]

# 3. Nested: [f(x,y) for x in xs for y in ys]
result3 = (1..3).flat_map { |x| (1..3).map { |y| {x, y} } }
puts result3.inspect  # => [{1, 1}, {1, 2}, ..., {3, 3}]

# 4. Nested with cond: [f(x,y) for x in xs for y in ys if cond]
result4 = (1..3).flat_map { |x|
  (1..3).filter_map { |y| {x, y} if x != y }
}
puts result4.inspect

# 5. Dict comprehension: {k: f(v) for k, v in d.items()}
d = {"a" => 1, "b" => 2, "c" => 3}
result5 = d.transform_values { |v| v * 10 }
puts result5.inspect  # => {"a" => 10, "b" => 20, "c" => 30}

# 6. Set comprehension: {f(x) for x in xs}
result6 = (1..5).map { |x| x % 3 }.to_set
puts result6.to_a.sort.inspect  # => [0, 1, 2]
```

---

## 58.11 ตัวอย่างในชีวิตจริง

### Matrix Operations

```crystal
# สร้าง matrix ด้วย comprehension-style
def identity_matrix(n : Int32) : Array(Array(Int32))
  (0...n).map { |i| (0...n).map { |j| i == j ? 1 : 0 } }
end

def multiply_matrix(a : Array(Array(Int32)), b : Array(Array(Int32))) : Array(Array(Int32))
  n = a.size
  (0...n).map { |i|
    (0...n).map { |j|
      (0...n).sum { |k| a[i][k] * b[k][j] }
    }
  }
end

id = identity_matrix(3)
id.each { |row| puts row.inspect }
# [1, 0, 0]
# [0, 1, 0]
# [0, 0, 1]

a = [[1, 2], [3, 4]]
b = [[5, 6], [7, 8]]
product = multiply_matrix(a, b)
product.each { |row| puts row.inspect }
# [19, 22]
# [43, 50]
```

### Data Transformation Pipeline

```crystal
# Simulate database query results
raw_data = [
  {user_id: 1, event: "login", timestamp: 1000},
  {user_id: 2, event: "login", timestamp: 1001},
  {user_id: 1, event: "purchase", timestamp: 1005},
  {user_id: 3, event: "login", timestamp: 1010},
  {user_id: 1, event: "logout", timestamp: 1020},
  {user_id: 2, event: "purchase", timestamp: 1025},
  {user_id: 2, event: "logout", timestamp: 1030}
]

# "Comprehension" style pipeline
user_purchases = raw_data
  .select { |e| e[:event] == "purchase" }           # Filter purchases
  .map { |e| e[:user_id] }                           # Extract user IDs
  .tally                                              # Count per user
  .select { |_, count| count > 0 }                   # Only buyers
  .transform_values { |count| "#{count} purchase(s)"}  # Format

puts "Buyers:"
user_purchases.each { |user, info| puts "  User #{user}: #{info}" }

# Session analysis
sessions = raw_data
  .group_by { |e| e[:user_id] }
  .map { |user_id, events|
    sorted = events.sort_by { |e| e[:timestamp] }
    login = sorted.find { |e| e[:event] == "login" }
    logout = sorted.find { |e| e[:event] == "logout" }
    duration = login && logout ? logout[:timestamp] - login[:timestamp] : nil
    {user_id: user_id, event_count: events.size, duration: duration}
  }

puts "\nSession analysis:"
sessions.each do |s|
  dur_str = s[:duration] ? "#{s[:duration]}s" : "still active"
  puts "  User #{s[:user_id]}: #{s[:event_count]} events, #{dur_str}"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Math Comprehensions

```crystal
# 1. Pythagorean triples ถึง n
def pythagorean_triples(n : Int32) : Array({Int32, Int32, Int32})
  (1..n).flat_map { |a|
    (a..n).flat_map { |b|
      c = Math.sqrt((a**2 + b**2).to_f).to_i
      c <= n && a**2 + b**2 == c**2 ? [{a, b, c}] : [] of {Int32, Int32, Int32}
    }
  }
end

puts "Pythagorean triples up to 20:"
pythagorean_triples(20).each { |a, b, c| puts "  #{a}² + #{b}² = #{c}²" }

# 2. Pascal's triangle
def pascal_row(n : Int32) : Array(Int32)
  (0..n).map { |k|
    # C(n, k) = n! / (k! * (n-k)!)
    (1..n).reduce(1_i64) { |acc, i| acc * i } /
    ((1..k).reduce(1_i64) { |acc, i| acc * i } *
     (1..(n-k)).reduce(1_i64) { |acc, i| acc * i })
  }.map(&.to_i32)
end

5.times { |n| puts "  #{pascal_row(n).join(" ")}" }
```

### แบบฝึกหัดที่ 2: Text Analysis

```crystal
text = "The quick brown fox jumps over the lazy dog. Pack my box with five dozen liquor jugs."

# Word analysis "comprehension"
word_analysis = text.split(/\W+/)
  .reject { |w| w.empty? }
  .map { |w| w.downcase }
  .group_by { |w| w[0] }
  .select { |_, words| words.size > 1 }
  .transform_values { |words|
    words.map { |w| {word: w, length: w.size} }
         .sort_by { |item| -item[:length] }
  }

puts "Letters with multiple words:"
word_analysis.sort_by { |k, _| k }.each do |letter, words|
  top2 = words.first(2).map { |w| w[:word] }.join(", ")
  puts "  #{letter}: #{top2}"
end
```

---

## สรุป

Crystal ไม่มี built-in list comprehension syntax แต่ทำได้ผ่าน:

| Pattern | Crystal code |
|---------|-------------|
| `[f(x) for x in xs]` | `xs.map { \|x\| f(x) }` |
| `[f(x) for x in xs if c(x)]` | `xs.filter_map { \|x\| f(x) if c(x) }` |
| `[f(x,y) for x in xs for y in ys]` | `xs.flat_map { \|x\| ys.map { \|y\| f(x,y) } }` |
| `{k: f(v) for k,v in d}` | `d.transform_values { \|v\| f(v) }` |
| `{f(x) for x in xs}` | `xs.map { \|x\| f(x) }.to_set` |
| lazy generator | `.lazy.map.select...` |

**Crystal ได้เปรียบ Python:**
- Type safety ณ compile time
- Performance ดีกว่ามาก
- `filter_map` สะดวกกว่า `[f(x) for x in xs if c(x)]`

---

*ต่อไป: Part 59 - Bit Arrays*
