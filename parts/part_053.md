# Part 53: Array Methods ขั้นสูง ใน Crystal

## บทนำ

ในบทนี้เราจะเรียนรู้ Array methods ขั้นสูงที่ใช้งานบ่อยในการประมวลผลข้อมูลจริง ตั้งแต่ flatten, zip, tally ไปจนถึง chunk และ rotate

---

## 53.1 flatten กับ depth

```crystal
# flatten ลด nesting ทั้งหมด
nested = [1, [2, [3, [4, [5]]]]]
puts nested.flatten.inspect  # => [1, 2, 3, 4, 5]

# flatten(depth) กำหนดระดับ
puts nested.flatten(1).inspect  # => [1, 2, [3, [4, [5]]]]
puts nested.flatten(2).inspect  # => [1, 2, 3, [4, [5]]]
puts nested.flatten(3).inspect  # => [1, 2, 3, 4, [5]]

# flatten! (in-place)
arr = [[1, 2], [3, [4, 5]], 6]
arr.flatten!
puts arr.inspect  # => [1, 2, 3, 4, 5, 6]

# ตัวอย่าง: matrix flattening
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
flat = matrix.flatten
puts flat.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
puts flat.sum      # => 45
```

---

## 53.2 flatten! และ zip กับ block

```crystal
# zip สร้าง array of tuples
a = [1, 2, 3]
b = ["a", "b", "c"]
c = [:x, :y, :z]

puts a.zip(b).inspect     # => [{1, "a"}, {2, "b"}, {3, "c"}]
puts a.zip(b, c).inspect  # => [{1, "a", :x}, {2, "b", :y}, {3, "c", :z}]

# zip กับ block
a.zip(b) { |num, letter|
  puts "#{num} -> #{letter}"
}
# 1 -> a
# 2 -> b
# 3 -> c

# zip กับ arrays ขนาดต่างกัน (ใช้ nil padding)
short = [1, 2, 3]
long = ["a", "b", "c", "d", "e"]
puts short.zip(long).inspect
# => [{1, "a"}, {2, "b"}, {3, "c"}]
# หมายเหตุ: Crystal zip จะ truncate ตาม receiver

# Transpose (zip เหมือน inverse of group)
data = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
transposed = data[0].zip(*data[1..])
puts transposed.inspect
```

---

## 53.3 each_slice

```crystal
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# each_slice แบ่ง array เป็น chunks ขนาดเท่ากัน
arr.each_slice(3) do |slice|
  puts slice.inspect
end
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10]        ← chunk สุดท้ายอาจไม่เต็ม

# เก็บเป็น array
slices = arr.each_slice(3).to_a
puts slices.inspect
# => [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10]]

# ใช้กับ processing
# เช่น batch processing
def process_batch(items : Array(Int32))
  items.each_slice(3) do |batch|
    sum = batch.sum
    puts "Batch #{batch.inspect}: sum = #{sum}"
  end
end

process_batch([5, 10, 15, 20, 25])
# Batch [5, 10, 15]: sum = 30
# Batch [20, 25]: sum = 45
```

---

## 53.4 each_cons

```crystal
arr = [1, 2, 3, 4, 5, 6, 7]

# each_cons สร้าง sliding window
arr.each_cons(3) do |window|
  puts window.inspect
end
# [1, 2, 3]
# [2, 3, 4]
# [3, 4, 5]
# [4, 5, 6]
# [5, 6, 7]

# เก็บเป็น array
windows = arr.each_cons(3).to_a
puts windows.size  # => 5

# ใช้กับ moving average
prices = [100.0, 105.0, 102.0, 108.0, 115.0, 110.0, 120.0]
moving_avg = prices.each_cons(3).map { |w| (w.sum / 3).round(2) }
puts "3-day moving average: #{moving_avg.to_a.inspect}"
# => [102.33, 105.0, 108.33, 111.0, 115.0]

# ตรวจสอบ trend
def increasing_trend?(prices : Array(Float64), window : Int32) : Array(Bool)
  prices.each_cons(window).map do |w|
    w.each_cons(2).all? { |a, b| b > a }
  end.to_a
end

puts increasing_trend?([1.0, 2.0, 3.0, 2.0, 4.0, 5.0], 3).inspect
# => [true, false, false, true]
```

---

## 53.5 tally

```crystal
arr = ["apple", "banana", "apple", "cherry", "banana", "apple"]

# tally นับความถี่
freq = arr.tally
puts freq.inspect
# => {"apple" => 3, "banana" => 2, "cherry" => 1}

# เรียงตามความถี่
sorted_freq = freq.sort_by { |_, count| -count }
sorted_freq.each { |word, count| puts "  #{word}: #{count}" }
# apple: 3
# banana: 2
# cherry: 1

# ตัวอย่าง: mode (ค่าที่พบบ่อยที่สุด)
def mode(arr : Array)
  arr.tally.max_by { |_, count| count }[0]
end

puts mode([1, 2, 2, 3, 3, 3, 4])  # => 3
puts mode(["a", "b", "b", "c"])    # => "b"

# tally กับ map
votes = ["Alice", "Bob", "Alice", "Charlie", "Alice", "Bob"]
result = votes.tally.sort_by { |_, v| -v }.first
puts "Winner: #{result[0]} with #{result[1]} votes"
```

---

## 53.6 chunk

```crystal
arr = [1, 1, 2, 2, 2, 3, 1, 1, 4]

# chunk จัดกลุ่ม elements ที่ต่อเนื่องกัน
arr.chunk { |n| n }.each do |key, values|
  puts "#{key}: #{values.inspect}"
end
# 1: [1, 1]
# 2: [2, 2, 2]
# 3: [3]
# 1: [1, 1]
# 4: [4]

# chunk_while
[1, 2, 3, 5, 6, 7, 10, 11].chunk_while { |a, b| b == a + 1 }.each do |group|
  puts group.inspect
end
# [1, 2, 3]
# [5, 6, 7]
# [10, 11]

# ตัวอย่าง: หา consecutive runs
scores = [85, 90, 88, 72, 68, 95, 92, 88]
threshold = 80

# แบ่งเป็น "pass" และ "fail" streaks
runs = scores.chunk { |s| s >= threshold ? "pass" : "fail" }
runs.each do |status, group|
  puts "#{status}: #{group.inspect} (avg: #{(group.sum / group.size).round(1)})"
end
```

---

## 53.7 chunk_while

```crystal
arr = [1, 2, 4, 9, 10, 11, 12, 15, 16, 19, 20, 21]

# chunk_while จัดกลุ่มตาม condition ระหว่าง elements ที่ต่อเนื่อง
consecutive_groups = arr.chunk_while { |a, b| b - a == 1 }
consecutive_groups.each { |g| puts g.inspect }
# [1, 2]
# [4]
# [9, 10, 11, 12]
# [15, 16]
# [19, 20, 21]

# ตัวอย่าง: หา sequences ในหุ้น
stock_prices = [100.0, 102.5, 101.0, 99.5, 103.0, 105.5, 104.0, 106.0, 108.0]
rising_sequences = stock_prices.each_cons(2).map { |a, b| b > a }.to_a

puts "Price changes: #{rising_sequences.inspect}"
# => [true, false, false, true, true, false, true, true]

# หา longest rising streak
def longest_streak(arr : Array(Bool)) : Int32
  max_streak = 0
  current = 0
  arr.each do |val|
    if val
      current += 1
      max_streak = [max_streak, current].max
    else
      current = 0
    end
  end
  max_streak
end

puts "Longest rising streak: #{longest_streak(rising_sequences)}"
```

---

## 53.8 each_with_object

```crystal
# each_with_object สะสมผลลัพธ์ใน object ที่กำหนด

words = ["hello", "world", "crystal", "programming"]

# สร้าง index ความยาว -> array of words
length_groups = words.each_with_object(Hash(Int32, Array(String)).new) do |word, hash|
  (hash[word.size] ||= [] of String) << word
end
puts length_groups.inspect
# => {5 => ["hello", "world"], 7 => ["crystal"], 11 => ["programming"]}

# สร้าง reversed lookup
lookup = words.each_with_object({} of String => Int32) do |word, hash|
  hash[word] = word.size
end
puts lookup.inspect
# => {"hello" => 5, "world" => 5, "crystal" => 7, "programming" => 11}

# Flatten กับ each_with_object
sentences = ["hello world", "crystal lang"]
all_words = sentences.each_with_object([] of String) do |sentence, arr|
  arr.concat(sentence.split)
end
puts all_words.inspect
# => ["hello", "world", "crystal", "lang"]
```

---

## 53.9 inject/reduce กับ initial value

```crystal
numbers = [1, 2, 3, 4, 5]

# reduce กับ initial value
sum = numbers.reduce(0) { |acc, n| acc + n }
puts sum  # => 15

product = numbers.reduce(1) { |acc, n| acc * n }
puts product  # => 120

# inject (alias)
max = numbers.inject { |m, n| n > m ? n : m }
puts max  # => 5

# สะสม string
words = ["crystal", "is", "awesome"]
sentence = words.reduce("") { |acc, word|
  acc.empty? ? word : "#{acc} #{word}"
}
puts sentence  # => "crystal is awesome"

# สะสม hash
pairs = [["a", 1], ["b", 2], ["c", 3]]
hash = pairs.reduce({} of String => Int32) { |h, pair|
  h[pair[0]] = pair[1]
  h
}
puts hash.inspect  # => {"a" => 1, "b" => 2, "c" => 3}

# factorial
def factorial(n : Int32) : Int64
  (1..n).reduce(1_i64) { |acc, i| acc * i }
end
puts factorial(10)  # => 3628800
```

---

## 53.10 flat_map

```crystal
# flat_map = map + flatten(1)
words = ["hello world", "crystal lang", "foo bar baz"]

# แบบยาว
all_words_long = words.map { |s| s.split }.flatten
puts all_words_long.inspect
# => ["hello", "world", "crystal", "lang", "foo", "bar", "baz"]

# แบบสั้นด้วย flat_map
all_words = words.flat_map { |s| s.split }
puts all_words.inspect
# => ["hello", "world", "crystal", "lang", "foo", "bar", "baz"]

# flat_map กับ transformations
matrix = [[1, 2, 3], [4, 5, 6], [7, 8, 9]]
doubled = matrix.flat_map { |row| row.map { |n| n * 2 } }
puts doubled.inspect
# => [2, 4, 6, 8, 10, 12, 14, 16, 18]

# สร้าง range ของ ranges
result = [1, 2, 3].flat_map { |n| (1..n).to_a }
puts result.inspect  # => [1, 1, 2, 1, 2, 3]

# Expand relationships
students_courses = {
  "Alice" => ["Math", "Science"],
  "Bob" => ["English", "Art", "PE"]
}

all_enrollments = students_courses.flat_map { |student, courses|
  courses.map { |course| {student, course} }
}
all_enrollments.each { |student, course| puts "#{student} -> #{course}" }
```

---

## 53.11 rotate

```crystal
arr = [1, 2, 3, 4, 5]

# rotate ซ้าย (default)
puts arr.rotate(1).inspect    # => [2, 3, 4, 5, 1]
puts arr.rotate(2).inspect    # => [3, 4, 5, 1, 2]
puts arr.rotate(3).inspect    # => [4, 5, 1, 2, 3]

# rotate ขวา (negative)
puts arr.rotate(-1).inspect   # => [5, 1, 2, 3, 4]
puts arr.rotate(-2).inspect   # => [4, 5, 1, 2, 3]

# rotate! (in-place)
arr2 = [1, 2, 3, 4, 5]
arr2.rotate!(2)
puts arr2.inspect  # => [3, 4, 5, 1, 2]

# ตัวอย่าง: Round-robin scheduling
def round_robin_schedule(teams : Array(String), rounds : Int32) : Array(Array({String, String}))
  schedule = [] of Array({String, String})
  
  n = teams.size
  fixed = teams[0]
  rotating = teams[1..].dup
  
  rounds.times do |round|
    round_matches = [] of {String, String}
    all = [fixed] + rotating
    
    (n / 2).times do |i|
      round_matches << {all[i], all[n - 1 - i]}
    end
    
    schedule << round_matches
    rotating.rotate!
  end
  
  schedule
end

teams = ["A", "B", "C", "D"]
schedule = round_robin_schedule(teams, 3)
schedule.each_with_index do |round, i|
  puts "Round #{i+1}: #{round.map { |a, b| "#{a} vs #{b}" }.join(", ")}"
end
```

---

## 53.12 sample และ shuffle

```crystal
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# sample: สุ่มตัว
puts arr.sample         # สุ่ม 1 ตัว
puts arr.sample(3).inspect  # สุ่ม 3 ตัว (ไม่ซ้ำ)

# shuffle: สุ่มเรียง
shuffled = arr.shuffle
puts shuffled.inspect  # ลำดับสุ่ม

# shuffle! (in-place)
arr.shuffle!
puts arr.inspect

# ตัวอย่าง: Card dealing
suits = ["♠", "♥", "♦", "♣"]
ranks = ["A", "2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K"]

deck = suits.flat_map { |s| ranks.map { |r| "#{r}#{s}" } }
puts "Deck size: #{deck.size}"  # => 52

deck.shuffle!
# Deal 5 cards to each of 4 players
hands = deck.each_slice(5).first(4).to_a
hands.each_with_index do |hand, i|
  puts "Player #{i+1}: #{hand.join(", ")}"
end
```

---

## 53.13 ตัวอย่างในชีวิตจริง: Data Analysis

```crystal
# Data analysis pipeline
sales_data = [
  {month: "Jan", product: "A", amount: 1500.0},
  {month: "Jan", product: "B", amount: 2200.0},
  {month: "Feb", product: "A", amount: 1800.0},
  {month: "Feb", product: "B", amount: 1900.0},
  {month: "Mar", product: "A", amount: 2100.0},
  {month: "Mar", product: "B", amount: 2500.0},
  {month: "Mar", product: "C", amount: 800.0}
]

# Monthly totals
monthly = sales_data.each_with_object(Hash(String, Float64).new(0.0)) do |sale, totals|
  totals[sale[:month]] += sale[:amount]
end
puts "Monthly totals:"
monthly.sort_by { |k, _| ["Jan", "Feb", "Mar"].index(k).not_nil! }
       .each { |month, total| puts "  #{month}: $#{total}" }

# Product performance
product_sales = sales_data.group_by { |s| s[:product] }
puts "\nProduct performance:"
product_sales.each do |product, sales|
  total = sales.sum { |s| s[:amount] }
  avg = total / sales.size
  puts "  #{product}: total=$#{total}, avg=$#{avg.round(2)}"
end

# Moving average of monthly totals
monthly_values = ["Jan", "Feb", "Mar"].map { |m| monthly[m] }
puts "\n2-month moving avg:"
monthly_values.each_cons(2).each_with_index do |pair, i|
  avg = (pair.sum / 2).round(2)
  months = ["Jan", "Feb", "Mar"]
  puts "  #{months[i]}-#{months[i+1]}: $#{avg}"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Text Processing

```crystal
text = """
The quick brown fox jumps over the lazy dog
Crystal is a compiled systems programming language
Hello world this is a test sentence for word analysis
"""

words = text.split.map(&.downcase).reject { |w| w.empty? }

# 1. Word frequency
freq = words.tally.sort_by { |_, c| -c }.first(5)
puts "Top 5 words:"
freq.each { |w, c| puts "  #{w}: #{c}" }

# 2. Words by length
length_groups = words.group_by(&.size)
puts "\nWords by length:"
length_groups.sort_by { |len, _| len }.each do |len, ws|
  puts "  Length #{len}: #{ws.uniq.first(3).join(", ")}..."
end

# 3. Sliding window
words_arr = text.split.map(&.downcase)
bigrams = words_arr.each_cons(2).map { |a, b| "#{a} #{b}" }
puts "\nTop 3 bigrams:"
bigrams.to_a.tally.sort_by { |_, c| -c }.first(3)
       .each { |bg, c| puts "  '#{bg}': #{c}" }
```

### แบบฝึกหัดที่ 2: Time Series Analysis

```crystal
# จำลอง time series data
temps = [20.0, 22.0, 25.0, 23.0, 19.0, 21.0, 26.0, 28.0, 27.0, 24.0,
         22.0, 25.0, 29.0, 31.0, 30.0, 28.0, 26.0, 27.0, 29.0, 32.0]

# 3-day moving average
moving_avg_3 = temps.each_cons(3).map { |w| (w.sum / 3).round(2) }.to_a

# 5-day moving average
moving_avg_5 = temps.each_cons(5).map { |w| (w.sum / 5).round(2) }.to_a

# Anomaly detection (more than 2 std dev from mean)
mean = temps.sum / temps.size
variance = temps.sum { |t| (t - mean) ** 2 } / temps.size
std = Math.sqrt(variance)

anomalies = temps.each_with_index
                 .select { |t, _| (t - mean).abs > 2 * std }
                 .map { |t, i| {i + 1, t} }
                 .to_a

puts "Temperatures: #{temps.first(5).inspect}..."
puts "3-day MA (first 5): #{moving_avg_3.first(5).inspect}"
puts "5-day MA (first 5): #{moving_avg_5.first(5).inspect}"
puts "Anomalies (day, temp): #{anomalies.inspect}"

# Trend detection
up_days = temps.each_cons(2).count { |a, b| b > a }
down_days = temps.each_cons(2).count { |a, b| b < a }
puts "\nTrend:"
puts "  Up days: #{up_days}"
puts "  Down days: #{down_days}"
```

---

## สรุป

Array methods ขั้นสูงใน Crystal:

| Method | ใช้งาน |
|--------|--------|
| `flatten(depth)` | ลด nesting level |
| `flatten!` | flatten in-place |
| `zip(arrays)` | จับคู่ elements |
| `each_slice(n)` | แบ่งเป็น chunks |
| `each_cons(n)` | sliding window |
| `tally` | นับความถี่ |
| `chunk` | จัดกลุ่ม consecutive |
| `chunk_while` | จัดกลุ่มตาม condition |
| `each_with_object` | สะสมใน object |
| `inject/reduce` | fold operation |
| `flat_map` | map + flatten |
| `rotate(n)` | หมุน elements |
| `sample(n)` | สุ่มตัว |
| `shuffle` | สุ่มเรียง |

---

*ต่อไป: Part 54 - Hash Methods ขั้นสูง*
