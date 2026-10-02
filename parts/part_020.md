# Part 020: Range

## บทนำ

**Range** ใน Crystal คือ sequence ของค่าระหว่างจุดเริ่มต้นและจุดสิ้นสุด เป็นหนึ่งในโครงสร้างข้อมูลที่ใช้บ่อยที่สุด ทั้งใน loop, array slicing, pattern matching และการตรวจสอบช่วงค่า

---

## 1. รูปแบบของ Range

### Inclusive Range: a..b

```crystal
# รวมทั้ง a และ b
range = 1..10
puts range.to_a.inspect
# => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# ใช้ใน loop
(1..5).each { |n| print "#{n} " }
puts
# => 1 2 3 4 5
```

### Exclusive Range: a...b

```crystal
# รวม a แต่ไม่รวม b
range = 1...10
puts range.to_a.inspect
# => [1, 2, 3, 4, 5, 6, 7, 8, 9]

# มีประโยชน์กับ array index
arr = ["a", "b", "c", "d", "e"]
(0...arr.size).each do |i|
  print arr[i]
end
puts
```

### ความแตกต่างระหว่าง .. และ ...

```crystal
inclusive = 1..5    # [1, 2, 3, 4, 5]
exclusive = 1...5   # [1, 2, 3, 4]

puts inclusive.includes?(5)  # => true
puts exclusive.includes?(5)  # => false

puts inclusive.to_a.last  # => 5
puts exclusive.to_a.last  # => 4
```

---

## 2. Beginless Range: ..b

```crystal
# Range ที่ไม่มีจุดเริ่มต้น
r = ..10
puts r.includes?(5)    # => true
puts r.includes?(10)   # => true
puts r.includes?(11)   # => false
puts r.includes?(-100) # => true

# ใช้ใน case
def classify_score(score : Int32)
  case score
  when ..49   then "F"
  when 50..59 then "D"
  when 60..69 then "C"
  when 70..79 then "B"
  when 80..   then "A"
  end
end

[30, 55, 65, 75, 90].each do |score|
  puts "#{score}: #{classify_score(score)}"
end
```

### ใช้ Beginless Range ใน Array slicing

```crystal
arr = [10, 20, 30, 40, 50]

# เอาตั้งแต่ต้นจนถึง index 2
puts arr[..2].inspect    # => [10, 20, 30]
puts arr[...2].inspect   # => [10, 20]
```

---

## 3. Endless Range: a..

```crystal
# Range ที่ไม่มีจุดสิ้นสุด
r = 5..
puts r.includes?(5)   # => true
puts r.includes?(100) # => true
puts r.includes?(4)   # => false

# ใช้ใน case
def age_group(age : Int32) : String
  case age
  when 0..12   then "เด็ก"
  when 13..17  then "วัยรุ่น"
  when 18..25  then "วัยผู้ใหญ่ตอนต้น"
  when 26..60  then "วัยทำงาน"
  when 61..    then "ผู้อาวุโส"
  else              "ไม่ทราบ"
  end
end

[5, 15, 22, 45, 70].each do |age|
  puts "อายุ #{age}: #{age_group(age)}"
end
```

### Endless Range ใน Array slicing

```crystal
arr = [10, 20, 30, 40, 50]

# เอาตั้งแต่ index 2 จนถึงสุด
puts arr[2..].inspect    # => [30, 40, 50]
puts arr[2...].inspect   # สับสน - ระวัง! เหมือนกัน
```

---

## 4. String Ranges

```crystal
# Range ของ String
("apple".."orange").each { |s| print s + " " } # ทุก string lexicographically
# (ระวัง: อาจช้ามากถ้าช่วงกว้าง)

# ใช้ include? ตรวจสอบ
names = "apple".."orange"
puts names.includes?("mango")   # => true
puts names.includes?("strawberry") # => false

# String range ใน case
def categorize_filename(filename : String) : String
  case filename
  when /\.txt$/, /\.md$/
    "เอกสาร"
  when /\.jpg$/, /\.png$/, /\.gif$/
    "รูปภาพ"
  when /\.mp3$/, /\.wav$/
    "เสียง"
  else
    "อื่นๆ"
  end
end

["readme.md", "photo.jpg", "music.mp3", "data.csv"].each do |f|
  puts "#{f}: #{categorize_filename(f)}"
end
```

---

## 5. Char Ranges

```crystal
# Range ของตัวอักษร
('a'..'z').each { |c| print c }
puts
# abcdefghijklmnopqrstuvwxyz

# ตรวจสอบ
puts ('a'..'z').includes?('m')  # => true
puts ('A'..'Z').includes?('a')  # => false

# สร้าง alphabet array
lowercase = ('a'..'z').to_a
uppercase = ('A'..'Z').to_a
digits = ('0'..'9').to_a

puts "ตัวพิมพ์เล็ก: #{lowercase.join}"
puts "ตัวพิมพ์ใหญ่: #{uppercase.join}"
puts "ตัวเลข: #{digits.join}"

# ROT13 cipher
def rot13(text : String) : String
  text.chars.map do |c|
    case c
    when 'a'..'z'
      ((c.ord - 'a'.ord + 13) % 26 + 'a'.ord).chr
    when 'A'..'Z'
      ((c.ord - 'A'.ord + 13) % 26 + 'A'.ord).chr
    else
      c
    end
  end.join
end

puts rot13("Hello World")  # => Uryyb Jbeyq
puts rot13("Uryyb Jbeyq")  # => Hello World
```

---

## 6. Range Methods

### include? และ covers?

```crystal
range = 1..100

# include? ตรวจสอบว่าค่าอยู่ใน range
puts range.includes?(50)  # => true
puts range.includes?(101) # => false

# covers? เหมือนกับ includes? แต่ใช้กับ range อื่น
# (ตรวจสอบว่า range ย่อยอยู่ใน range หลักทั้งหมด)
puts range.covers?(10..50)   # => true
puts range.covers?(50..150)  # => false
```

### each

```crystal
# วน loop
(1..5).each do |n|
  print "#{n} "
end
puts

# each กับ step (ใช้ step method)
(0..20).step(5).each do |n|
  print "#{n} "
end
puts
# => 0 5 10 15 20
```

### to_a

```crystal
# แปลงเป็น array
nums = (1..10).to_a
puts nums.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

chars = ('a'..'e').to_a
puts chars.inspect  # => ['a', 'b', 'c', 'd', 'e']

# Exclusive
exclusive = (1...10).to_a
puts exclusive.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### step

```crystal
# step ใช้กำหนดขนาดก้าว
(0..20).step(4).each { |n| print "#{n} " }
puts
# => 0 4 8 12 16 20

# step กับ float
(0.0..1.0).step(0.25).each { |n| print "#{n} " }
puts
# => 0.0 0.25 0.5 0.75 1.0

# step ใช้สร้าง array
squares = (1..10).step(1).map { |n| n ** 2 }.to_a
puts squares.inspect
# => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
```

### min, max, sum, size

```crystal
range = (1..100)

puts range.min    # => 1
puts range.max    # => 100
puts range.sum    # => 5050
puts range.size   # => 100

# min/max กับ condition
puts range.min_by { |n| (n - 50).abs }  # ใกล้ 50 มากที่สุด => 50
puts range.max_by { |n| -(n - 50).abs } # ไกล 50 มากที่สุด => 1 หรือ 100
```

### first, last

```crystal
range = (1..100)

puts range.first       # => 1
puts range.first(5).inspect   # => [1, 2, 3, 4, 5]
puts range.last        # => 100
puts range.last(5).inspect    # => [96, 97, 98, 99, 100]
```

### sample, shuffle (เฉพาะบน to_a)

```crystal
# Range ไม่มี sample โดยตรง แต่แปลงเป็น array ได้
pool = (1..100).to_a
random_number = pool.sample
puts "สุ่ม: #{random_number}"

# สุ่ม 5 ตัวไม่ซ้ำ
five_random = (1..100).to_a.sample(5)
puts five_random.inspect

# shuffle
shuffled = (1..10).to_a.shuffle
puts shuffled.inspect
```

---

## 7. ตัวอย่างการใช้งานจริง

### Pagination

```crystal
def paginate(total_items : Int32, items_per_page : Int32, current_page : Int32) : Range(Int32, Int32)
  start_idx = (current_page - 1) * items_per_page
  end_idx = [start_idx + items_per_page - 1, total_items - 1].min
  start_idx..end_idx
end

items = (1..50).to_a  # 50 รายการ
total = items.size
per_page = 10

# แสดงหน้า 1, 2, 5
[1, 2, 5].each do |page|
  range = paginate(total, per_page, page)
  page_items = items[range]
  puts "หน้า #{page}: #{page_items.inspect}"
end
```

### Business Hours Check

```crystal
def is_business_hours?(hour : Int32, day_of_week : Int32) : Bool
  # day_of_week: 1=Monday, 7=Sunday
  workday = (1..5).includes?(day_of_week)
  working_hour = (9..17).includes?(hour)
  workday && working_hour
end

# ทดสอบ
[
  {hour: 10, day: 1},  # Monday 10am
  {hour: 8, day: 1},   # Monday 8am
  {hour: 14, day: 6},  # Saturday 2pm
  {hour: 17, day: 5},  # Friday 5pm
].each do |t|
  open = is_business_hours?(t[:hour], t[:day])
  day_names = %w[_ จันทร์ อังคาร พุธ พฤหัส ศุกร์ เสาร์ อาทิตย์]
  puts "วัน#{day_names[t[:day]]} #{t[:hour]}:00 - #{open ? "เปิด" : "ปิด"}"
end
```

### Number Formatting (Thousands)

```crystal
def format_with_commas(n : Int32) : String
  str = n.abs.to_s
  result = String::Builder.new
  
  str.chars.each_with_index do |c, i|
    result << "," if i > 0 && (str.size - i) % 3 == 0
    result << c
  end
  
  n < 0 ? "-" + result.to_s : result.to_s
end

[1234567, 9876543, 12345, 100].each do |n|
  puts format_with_commas(n)
end
```

### Sliding Window Analysis

```crystal
# วิเคราะห์ข้อมูลด้วย sliding window
def detect_trend(prices : Array(Float64), window : Int32) : Array(String)
  trends = [] of String
  
  (0..prices.size - window - 1).each do |i|
    current_window = prices[i, window]
    next_val = prices[i + window]
    
    avg = current_window.sum / current_window.size
    trend = if next_val > avg * 1.05
      "ขึ้น"
    elsif next_val < avg * 0.95
      "ลง"
    else
      "คงที่"
    end
    
    trends << trend
  end
  
  trends
end

prices = [100.0, 102.0, 98.0, 105.0, 110.0, 108.0, 115.0, 112.0]
trends = detect_trend(prices, 3)
puts "แนวโน้ม: #{trends.join(", ")}"
```

### Character Frequency Analysis

```crystal
def char_frequency(text : String) : Hash(Char, Int32)
  text.downcase.chars
    .select { |c| ('a'..'z').includes?(c) }
    .tally
    .to_h
end

text = "Hello Crystal Programming Language"
freq = char_frequency(text)

# แสดง top 5 ตัวอักษร
puts "ตัวอักษรที่พบบ่อยสุด 5 อันดับ:"
freq.to_a
  .sort_by { |c, count| -count }
  .first(5)
  .each_with_index do |(char, count), i|
    puts "#{i + 1}. '#{char}': #{count} ครั้ง"
  end
```

---

## 8. Range ขั้นสูง

### Custom Object ใน Range

สำหรับ object ของเราเองให้ใช้ใน range ต้อง implement `<=>`

```crystal
struct Version
  include Comparable(Version)
  
  getter major : Int32
  getter minor : Int32
  getter patch : Int32
  
  def initialize(@major : Int32, @minor : Int32, @patch : Int32)
  end
  
  def <=>(other : Version) : Int32
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    patch <=> other.patch
  end
  
  def to_s(io : IO) : Nil
    io << "#{major}.#{minor}.#{patch}"
  end
  
  def succ : Version
    Version.new(major, minor, patch + 1)
  end
end

v1 = Version.new(1, 0, 0)
v2 = Version.new(1, 2, 3)
v3 = Version.new(2, 0, 0)

range = v1..v2
puts range.includes?(Version.new(1, 1, 0))  # => true
puts range.includes?(v3)                     # => false
```

### Range เป็น Argument

```crystal
def generate_random_in_range(range : Range(Int32, Int32)) : Int32
  range.begin + Random.rand(range.end - range.begin + (range.exclusive? ? 0 : 1))
end

10.times { print "#{generate_random_in_range(1..6)} " }
puts  # จำลองการโยนลูกเต๋า
```

### Overlapping Ranges

```crystal
def ranges_overlap?(r1 : Range(Int32, Int32), r2 : Range(Int32, Int32)) : Bool
  r1.begin <= r2.end && r2.begin <= r1.end
end

def range_intersection(r1 : Range(Int32, Int32), r2 : Range(Int32, Int32)) : Range(Int32, Int32)?
  return nil unless ranges_overlap?(r1, r2)
  [r1.begin, r2.begin].max..[r1.end, r2.end].min
end

r1 = 1..10
r2 = 7..15
r3 = 12..20

puts ranges_overlap?(r1, r2).inspect   # => true
puts ranges_overlap?(r1, r3).inspect   # => false
puts range_intersection(r1, r2).inspect  # => 7..10
puts range_intersection(r1, r3).inspect  # => nil
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Date Range

```crystal
# สร้าง DateRange class
struct MyDate
  getter year : Int32
  getter month : Int32
  getter day : Int32
  
  def initialize(@year : Int32, @month : Int32, @day : Int32)
  end
  
  def to_s(io : IO) : Nil
    io << "#{year}-#{month.to_s.rjust(2, '0')}-#{day.to_s.rjust(2, '0')}"
  end
end

# TODO: สร้าง range ของวันที่ และ:
# 1. ตรวจสอบว่าวันที่อยู่ใน range หรือไม่
# 2. นับจำนวนวันใน range
# 3. แสดงทุกวันใน range
```

### แบบฝึกหัดที่ 2: Grade Distribution

```crystal
# ใช้ Range เพื่อวิเคราะห์การกระจายคะแนน
scores = [78, 92, 65, 88, 71, 95, 84, 67, 90, 73,
          82, 60, 87, 94, 76, 69, 85, 91, 63, 80]

grade_ranges = {
  "A" => (90..100),
  "B" => (80..89),
  "C" => (70..79),
  "D" => (60..69),
  "F" => (0..59),
}

# TODO: นับจำนวนนักเรียนในแต่ละเกรด
# แสดง histogram แบบง่ายๆ ด้วย *
```

### แบบฝึกหัดที่ 3: Time Slot Scheduler

```crystal
# ระบบจอง time slot
# เวลาทำงาน 9:00 - 17:00 (9..17)
# แต่ละ slot กว้าง 1 ชั่วโมง
# TODO:
# 1. สร้าง list ของ available slots
# 2. mark บาง slot ว่า booked
# 3. แสดง calendar ที่ว่างและที่จอง

work_hours = 9..16  # 9:00 to 16:00 (last slot starts at 16)
booked_slots = [9, 11, 14]

# TODO: implement
```

### แบบฝึกหัดที่ 4: Range Statistics

```crystal
# เขียน method ที่รับ array ของ Range
# แล้วหา:
# 1. ขนาดรวมของทุก range
# 2. ค่า min และ max โดยรวม
# 3. ว่ามี overlap กันหรือไม่

ranges = [1..10, 5..15, 20..30, 8..12]

# TODO: implement stats function
def range_stats(ranges : Array(Range(Int32, Int32)))
  # ส่งค่ากลับ: total_size, global_min, global_max, has_overlap
end
```

---

## สรุป

| Syntax | ความหมาย | ตัวอย่าง |
|--------|----------|----------|
| `1..10` | Inclusive (รวม 10) | `[1, 2, ..., 10]` |
| `1...10` | Exclusive (ไม่รวม 10) | `[1, 2, ..., 9]` |
| `..10` | Beginless (ตั้งแต่ -∞ ถึง 10) | includes? ค่าใดๆ ≤ 10 |
| `1..` | Endless (ตั้งแต่ 1 ถึง +∞) | includes? ค่าใดๆ ≥ 1 |

| Method | ความหมาย |
|--------|----------|
| `includes?(x)` | ตรวจสอบว่า x อยู่ใน range |
| `covers?(range)` | ตรวจสอบว่า range ย่อยอยู่ใน range หลัก |
| `each { }` | วน loop |
| `to_a` | แปลงเป็น array |
| `step(n)` | วน loop ด้วยขนาดก้าว n |
| `min` / `max` | หาค่าต่ำสุด/สูงสุด |
| `sum` | หาผลรวม |
| `size` | จำนวนสมาชิก |
| `first(n)` | n ค่าแรก |
| `last(n)` | n ค่าสุดท้าย |

### Tips

1. `..` (inclusive) มักใช้กับ pagination, index-based
2. `...` (exclusive) มักใช้กับ size-based loop เช่น `0...arr.size`
3. Beginless/Endless range มีประโยชน์มากในคำสั่ง `case`
4. ระวังการใช้ `.to_a` กับ String range ที่กว้างมาก
5. `step` ใช้แทน `* n` ใน loop เพื่อความชัดเจน
