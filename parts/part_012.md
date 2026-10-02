# ตอนที่ 12: While / Until Loops

## บทนำ

Loops ช่วยให้เราทำซ้ำการทำงานได้โดยไม่ต้องเขียนโค้ดซ้ำ Crystal มี while, until, loop do และ keywords พิเศษอย่าง break, next, redo สำหรับควบคุม loop

---

## 12.1 While Loop

`while` ทำงานซ้ำตราบใดที่เงื่อนไขยังเป็น true

```crystal
# รูปแบบพื้นฐาน
count = 0
while count < 5
  puts "นับ: #{count}"
  count += 1
end

# Output:
# นับ: 0
# นับ: 1
# นับ: 2
# นับ: 3
# นับ: 4

# while กับเงื่อนไขซับซ้อน
x = 1
while x < 100 && x % 7 != 0
  x += 1
end
puts "จำนวนแรกที่หาร 7 ลงตัวและน้อยกว่า 100: #{x}"  # => 7
```

### While กับ Array

```crystal
fruits = ["apple", "banana", "cherry"]
index = 0

while index < fruits.size
  puts "ผลไม้ #{index + 1}: #{fruits[index]}"
  index += 1
end

# Output:
# ผลไม้ 1: apple
# ผลไม้ 2: banana
# ผลไม้ 3: cherry
```

---

## 12.2 Until Loop

`until` ทำงานซ้ำตราบใดที่เงื่อนไขยังเป็น false (ตรงข้ามกับ while)

```crystal
# until = while !condition
count = 0
until count >= 5
  puts "นับ: #{count}"
  count += 1
end

# เทียบเท่ากับ
# while count < 5

# ตัวอย่างจริง: รอจนกว่าจะพบสิ่งที่ต้องการ
numbers = [2, 4, 6, 7, 8, 10]
index = 0
until numbers[index] % 2 != 0
  index += 1
end
puts "เลขคี่ตัวแรก: #{numbers[index]}"  # => 7

# until กับการตรวจสอบ
done = false
attempts = 0
until done
  attempts += 1
  done = true if attempts >= 3
  puts "พยายามครั้งที่ #{attempts}"
end
```

### เมื่อใดใช้ while vs until

```crystal
# ใช้ while เมื่อเงื่อนไขบวก
password_valid = false
while !password_valid
  # ไม่ดี: double negation อ่านยาก
  password_valid = true
end

# ดีกว่า: ใช้ until
until password_valid
  # ชัดเจนกว่า
  password_valid = true
end
```

---

## 12.3 Loop Do (Infinite Loop)

`loop do...end` สร้าง infinite loop ต้องใช้ `break` เพื่อออก

```crystal
# Infinite loop ต้องมี break!
loop do
  print "กรอกตัวเลข (0 เพื่อออก): "
  # ใน context จริงจะอ่าน input
  input = "5"  # จำลอง
  num = input.to_i?
  
  if num.nil?
    puts "กรุณากรอกตัวเลข"
    next
  end
  
  break if num == 0
  puts "คุณกรอก: #{num}"
  break  # ใน demo นี้ break เพื่อไม่ loop ต่อ
end

# Loop ที่มี counter
loop do |i|
  # หมายเหตุ: loop do ไม่รับ block parameter โดยตรง
  # ต้องใช้ตัวแปรภายนอก
  break
end

# ใช้ตัวแปรภายนอก
i = 0
loop do
  puts "iteration #{i}"
  i += 1
  break if i >= 3
end
```

---

## 12.4 Break

`break` ออกจาก loop ทันที

```crystal
# break พื้นฐาน
(1..10).each do |i|
  break if i > 5
  puts i
end
# Output: 1, 2, 3, 4, 5

# break ใน while
i = 0
while true
  i += 1
  break if i == 5
end
puts "break ที่ i = #{i}"  # => 5

# break ออกจาก loop do
found = nil
data = [3, 7, 2, 9, 1, 8]
loop do
  data.each do |n|
    if n > 7
      found = n
      break
    end
  end
  break  # break loop do
end
puts "พบ: #{found}"  # => 9
```

### Break กับค่า

```crystal
# break สามารถ return ค่าได้
result = while true
  x = rand(10)
  break x if x > 7
end
puts "ได้เลข > 7: #{result}"

# ใช้ break กับ loop เพื่อ return ค่า
random_value = loop do
  n = rand(100)
  break n if n.even? && n > 50
end
puts "เลขคู่ที่มากกว่า 50: #{random_value}"
```

---

## 12.5 Next

`next` ข้ามไปยัง iteration ถัดไป (เทียบกับ continue ในภาษาอื่น)

```crystal
# ข้ามเลขคู่
(1..10).each do |i|
  next if i.even?
  puts i  # พิมพ์เฉพาะเลขคี่
end
# Output: 1, 3, 5, 7, 9

# next ใน while
i = 0
while i < 10
  i += 1
  next if i % 3 == 0  # ข้ามเลขที่หาร 3 ลงตัว
  puts i
end
# Output: 1, 2, 4, 5, 7, 8, 10

# ใช้ next เพื่อ filter
words = ["hello", "", "world", "  ", "crystal"]
words.each do |word|
  next if word.strip.empty?  # ข้าม empty strings
  puts word.upcase
end
# Output: HELLO, WORLD, CRYSTAL
```

---

## 12.6 Redo

`redo` ทำ iteration ปัจจุบันซ้ำโดยไม่เพิ่ม counter

```crystal
# redo - ระวัง infinite loop!
attempts = 0
(1..3).each do |i|
  attempts += 1
  
  if attempts < 2 && i == 1
    puts "redo iteration #{i} (attempt #{attempts})"
    redo  # ทำซ้ำ iteration นี้
  end
  
  puts "ผ่าน iteration #{i}"
end

# Output:
# redo iteration 1 (attempt 1)
# ผ่าน iteration 1
# ผ่าน iteration 2
# ผ่าน iteration 3
```

---

## 12.7 Loop กับ Counter

```crystal
# Counting loop แบบต่างๆ
# วิธีที่ 1: while
i = 1
while i <= 5
  puts "#{i} "
  i += 1
end

# วิธีที่ 2: times (ดูบทถัดไป)
5.times { |i| print "#{i + 1} " }
puts

# วิธีที่ 3: upto
1.upto(5) { |i| print "#{i} " }
puts

# วิธีที่ 4: range each
(1..5).each { |i| print "#{i} " }
puts
```

### Loop Counter Patterns

```crystal
# Loop ที่นับถอยหลัง
i = 10
while i > 0
  puts "นับถอยหลัง: #{i}"
  i -= 2
end

# Loop ที่เพิ่มแบบ exponential
value = 1
while value < 1000
  puts value
  value *= 2
end
# Output: 1, 2, 4, 8, 16, 32, 64, 128, 256, 512

# Collatz conjecture
n = 27
steps = 0
while n != 1
  n = n.even? ? n // 2 : n * 3 + 1
  steps += 1
end
puts "Collatz: 27 ใช้ #{steps} steps"  # => 111 steps
```

---

## 12.8 Infinite Loop Patterns

```crystal
# Pattern 1: Game Loop
game_running = true
frame = 0

while game_running
  frame += 1
  
  # Update game state
  puts "Frame #{frame}"
  
  # Check win/lose condition
  game_running = false if frame >= 3
end
puts "Game over!"

# Pattern 2: Retry with limit
MAX_RETRIES = 3
retries = 0

loop do
  success = retries >= 2  # จำลองว่าสำเร็จครั้งที่ 3
  
  if success
    puts "สำเร็จหลังพยายาม #{retries + 1} ครั้ง"
    break
  end
  
  retries += 1
  if retries >= MAX_RETRIES
    puts "ล้มเหลวหลังพยายาม #{MAX_RETRIES} ครั้ง"
    break
  end
  
  puts "ลองใหม่ครั้งที่ #{retries}..."
end

# Pattern 3: Producer/Consumer
buffer = [] of Int32
produced = 0

loop do
  # Produce
  if buffer.size < 3 && produced < 10
    item = produced + 1
    buffer << item
    produced += 1
    puts "ผลิต: #{item}"
  end
  
  # Consume
  unless buffer.empty?
    item = buffer.shift
    puts "ใช้: #{item}"
  end
  
  break if produced >= 5 && buffer.empty?
end
```

---

## 12.9 While เป็น Expression

```crystal
# while คืนค่า nil เสมอ (ไม่เหมือน if)
result = while false
  # ไม่ทำงาน
end
puts result.inspect  # => nil

# ใช้ตัวแปรภายนอกรับค่า
found_index = nil
items = [3, 1, 4, 1, 5, 9, 2, 6]
i = 0

while i < items.size
  if items[i] > 7
    found_index = i
    break
  end
  i += 1
end

puts "พบค่า > 7 ที่ index: #{found_index}"  # => 5
```

---

## 12.10 ตัวอย่างจริง: Binary Search

```crystal
def binary_search(arr : Array(Int32), target : Int32) : Int32?
  low = 0
  high = arr.size - 1
  
  while low <= high
    mid = (low + high) // 2
    
    if arr[mid] == target
      return mid  # พบแล้ว
    elsif arr[mid] < target
      low = mid + 1  # ค้นหาในครึ่งขวา
    else
      high = mid - 1  # ค้นหาในครึ่งซ้าย
    end
  end
  
  nil  # ไม่พบ
end

sorted = [1, 3, 5, 7, 9, 11, 13, 15, 17, 19]
puts binary_search(sorted, 7).inspect   # => 3
puts binary_search(sorted, 13).inspect  # => 6
puts binary_search(sorted, 4).inspect   # => nil
```

---

## 12.11 ตัวอย่างจริง: Input Validation Loop

```crystal
# จำลองการรับ input จากผู้ใช้
def simulate_input(inputs : Array(String))
  index = 0
  -> {
    input = inputs[index]?
    index += 1
    input || ""
  }
end

def get_valid_number(get_input : Proc(String)) : Int32
  loop do
    print "กรอกตัวเลข (1-100): "
    input = get_input.call
    puts input  # แสดง input ที่จำลอง
    
    num = input.to_i?
    
    if num.nil?
      puts "Error: กรุณากรอกตัวเลขเท่านั้น"
      next
    end
    
    unless num >= 1 && num <= 100
      puts "Error: ตัวเลขต้องอยู่ระหว่าง 1-100"
      next
    end
    
    return num  # คืนค่าถ้า valid
  end
end

# ทดสอบ
inputs = simulate_input(["abc", "0", "150", "42"])
num = get_valid_number(inputs)
puts "ได้ตัวเลข: #{num}"

# Output:
# กรอกตัวเลข (1-100): abc
# Error: กรุณากรอกตัวเลขเท่านั้น
# กรอกตัวเลข (1-100): 0
# Error: ตัวเลขต้องอยู่ระหว่าง 1-100
# กรอกตัวเลข (1-100): 150
# Error: ตัวเลขต้องอยู่ระหว่าง 1-100
# กรอกตัวเลข (1-100): 42
# ได้ตัวเลข: 42
```

---

## 12.12 ตัวอย่างจริง: Number Guessing Game

```crystal
def play_guessing_game(secret : Int32, get_guess : Proc(String)) : Int32
  attempts = 0
  
  loop do
    attempts += 1
    print "เดาตัวเลข (1-100): "
    input = get_guess.call
    puts input
    
    guess = input.to_i?
    
    if guess.nil?
      puts "กรุณากรอกตัวเลข!"
      attempts -= 1  # ไม่นับ invalid input
      next
    end
    
    if guess < secret
      puts "น้อยไป!"
    elsif guess > secret
      puts "มากไป!"
    else
      puts "ถูกต้อง! ใช้ #{attempts} ครั้ง"
      break
    end
  end
  
  attempts
end

# จำลองการเล่น
secret = 42
guesses = ["50", "25", "37", "42"]
index = 0
get_guess = -> { guesses[index].tap { index += 1 } }

attempts = play_guessing_game(secret, get_guess)
puts "จบเกมใน #{attempts} ครั้ง"
```

---

## 12.13 ตัวอย่างจริง: Fibonacci Sequence

```crystal
# สร้าง Fibonacci sequence ด้วย while
def fibonacci_sequence(limit : Int32) : Array(Int64)
  sequence = [] of Int64
  a, b = 0_i64, 1_i64
  
  while a <= limit
    sequence << a
    a, b = b, a + b
  end
  
  sequence
end

puts fibonacci_sequence(100).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34, 55, 89]

# หา Fibonacci ลำดับที่ n
def nth_fibonacci(n : Int32) : Int64
  return n.to_i64 if n <= 1
  
  prev, curr = 0_i64, 1_i64
  count = 2
  
  while count <= n
    prev, curr = curr, prev + curr
    count += 1
  end
  
  curr
end

(0..10).each do |i|
  puts "F(#{i}) = #{nth_fibonacci(i)}"
end
```

---

## 12.14 Nested Loops

```crystal
# Multiplication table
puts "ตารางสูตรคูณ"
i = 1
while i <= 5
  j = 1
  while j <= 5
    print "#{(i * j).to_s.rjust(4)}"
    j += 1
  end
  puts
  i += 1
end

# Output:
#    1   2   3   4   5
#    2   4   6   8  10
#    3   6   9  12  15
#    4   8  12  16  20
#    5  10  15  20  25

# Break ออกจาก nested loop
found_i = -1
found_j = -1

i = 0
while i < 5
  j = 0
  while j < 5
    if i * j == 12
      found_i = i
      found_j = j
      break  # ออกแค่ loop ใน
    end
    j += 1
  end
  break if found_i >= 0  # ออก loop นอก
  i += 1
end

puts "พบ: #{found_i} * #{found_j} = 12"  # => 3 * 4 = 12
```

---

## 12.15 ตัวอย่างโปรแกรมจริง: Bubble Sort

```crystal
def bubble_sort(arr : Array(Int32)) : Array(Int32)
  sorted = arr.dup
  n = sorted.size
  
  i = 0
  while i < n - 1
    swapped = false
    
    j = 0
    while j < n - i - 1
      if sorted[j] > sorted[j + 1]
        # Swap
        sorted[j], sorted[j + 1] = sorted[j + 1], sorted[j]
        swapped = true
      end
      j += 1
    end
    
    # ถ้าไม่มีการสลับเลย แสดงว่าเรียงแล้ว
    break unless swapped
    i += 1
  end
  
  sorted
end

data = [64, 34, 25, 12, 22, 11, 90]
puts "ก่อนเรียง: #{data.inspect}"
puts "หลังเรียง: #{bubble_sort(data).inspect}"

# Output:
# ก่อนเรียง: [64, 34, 25, 12, 22, 11, 90]
# หลังเรียง: [11, 12, 22, 25, 34, 64, 90]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Sum of Digits
```crystal
# คำนวณผลรวมของหลักตัวเลข
# เช่น 12345 -> 1+2+3+4+5 = 15
def sum_of_digits(n : Int32) : Int32
  # TODO: ใช้ while loop
  # hint: ใช้ n % 10 เพื่อได้หลักสุดท้าย
  # และ n // 10 เพื่อตัดหลักสุดท้ายออก
end

puts sum_of_digits(12345)  # => 15
puts sum_of_digits(999)    # => 27
```

### แบบฝึกหัดที่ 2: GCD
```crystal
# หา Greatest Common Divisor (ห.ร.ม.)
# ใช้ Euclidean algorithm
def gcd(a : Int32, b : Int32) : Int32
  # TODO: implement
  # while b != 0: a, b = b, a % b
end

puts gcd(48, 18)  # => 6
puts gcd(100, 75) # => 25
```

### แบบฝึกหัดที่ 3: Palindrome Check
```crystal
# ตรวจสอบว่า string เป็น palindrome โดยใช้ while
def palindrome?(str : String) : Bool
  # TODO: ใช้ while loop (ไม่ใช้ reverse)
  # เปรียบเทียบตัวอักษรจากซ้ายและขวาเข้าหากัน
end

puts palindrome?("racecar")  # => true
puts palindrome?("hello")    # => false
puts palindrome?("madam")    # => true
```

### แบบฝึกหัดที่ 4: Prime Numbers
```crystal
# หาจำนวนเฉพาะทั้งหมดที่น้อยกว่า n
# ใช้ Sieve of Eratosthenes กับ while loop
def primes_up_to(n : Int32) : Array(Int32)
  # TODO: implement
end

puts primes_up_to(50).inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **while**: loop ที่ทำงานเมื่อเงื่อนไขเป็น true
2. **until**: loop ที่ทำงานเมื่อเงื่อนไขเป็น false
3. **loop do**: infinite loop ที่ต้องใช้ break
4. **break**: ออกจาก loop ทันที (สามารถ return ค่าได้)
5. **next**: ข้ามไป iteration ถัดไป
6. **redo**: ทำ iteration ปัจจุบันซ้ำ
7. **Infinite Loop Patterns**: game loop, retry pattern
8. **Nested Loops**: ระวังการใช้ break ใน nested loop

### เปรียบเทียบ while vs until vs loop

| Construct | ใช้เมื่อ |
|-----------|---------|
| `while cond` | รู้เงื่อนไขการหยุดล่วงหน้า |
| `until cond` | เงื่อนไขเป็น negative |
| `loop do` | ต้องการ infinite loop หรือเงื่อนไขซับซ้อน |

ใน Crystal จริงๆ เรามักใช้ `each`, `map`, `select` แทน while/until สำหรับ collections ซึ่งจะเรียนในบทถัดไป!
