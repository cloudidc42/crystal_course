# Part 018: Ternary Operator และ Conditional Expressions

## บทนำ

Crystal รองรับการเขียน conditional ที่กระชับและ expressive หลายรูปแบบ ตั้งแต่ ternary operator ที่คุ้นเคย ไปจนถึงการใช้ `if` และ `case` เป็น expression ที่ส่งค่ากลับได้ ทำให้โค้ดกระชับและอ่านง่ายขึ้นมาก

---

## 1. Ternary Operator: condition ? a : b

```crystal
# รูปแบบ: condition ? value_if_true : value_if_false
age = 20
status = age >= 18 ? "ผู้ใหญ่" : "เยาวชน"
puts status  # => ผู้ใหญ่

# เทียบกับ if แบบยาว
status = if age >= 18
  "ผู้ใหญ่"
else
  "เยาวชน"
end
```

### ternary แบบต่างๆ

```crystal
# ternary ใน string interpolation
score = 85
grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : "F"
puts "เกรด: #{grade}"  # => เกรด: B

# ternary เป็น argument
def greet(is_morning : Bool)
  puts "สวัสดี#{is_morning ? "ตอนเช้า" : "ตอนเย็น"}!"
end

greet(true)   # => สวัสดีตอนเช้า!
greet(false)  # => สวัสดีตอนเย็น!

# ternary assign
x = 10
y = x > 5 ? x * 2 : x / 2
puts y  # => 20
```

### ternary ซ้อนกัน (nested) - ควรระวัง

```crystal
# ternary ซ้อนอ่านยาก
n = 7
result = n > 10 ? "ใหญ่มาก" : n > 5 ? "ปานกลาง" : "เล็ก"
puts result  # => ปานกลาง

# แนะนำให้ใช้ case แทนเมื่อมีหลายเงื่อนไข
result = case n
         when .> 10 then "ใหญ่มาก"
         when .> 5  then "ปานกลาง"
         else            "เล็ก"
         end
puts result  # => ปานกลาง
```

---

## 2. if as Expression

ใน Crystal `if` เป็น expression ที่ส่งค่ากลับได้:

```crystal
# if ส่งค่ากลับ
message = if true
  "เป็นจริง"
else
  "เป็นเท็จ"
end
puts message  # => เป็นจริง

# ค่าที่ส่งกลับคือนิพจน์สุดท้ายในแต่ละ branch
score = 75
grade = if score >= 90
  "A"
elsif score >= 80
  "B"
elsif score >= 70
  "C"
elsif score >= 60
  "D"
else
  "F"
end
puts "เกรด: #{grade}"  # => เกรด: C
```

### if expression ใน argument

```crystal
def process(value : String)
  puts "ประมวลผล: #{value}"
end

temperature = 35
process(if temperature > 30
  "ร้อน"
elsif temperature > 20
  "อบอุ่น"
else
  "เย็น"
end)
# => ประมวลผล: ร้อน
```

### one-line if

```crystal
# if แบบ one-line (postfix if)
x = 10
puts "x เป็นบวก" if x > 0
puts "x เป็นลบ" if x < 0  # ไม่แสดง

# unless (เงื่อนไขกลับ)
puts "x ไม่ใช่ศูนย์" unless x == 0

# กำหนดค่าด้วย postfix if
result = "ok"
result = "error" if x < 0
puts result  # => ok
```

---

## 3. case as Expression

`case` ก็เป็น expression เช่นกัน:

```crystal
# case ส่งค่ากลับ
day = "Monday"
day_type = case day
           when "Saturday", "Sunday"
             "วันหยุด"
           when "Monday", "Tuesday", "Wednesday", "Thursday", "Friday"
             "วันทำงาน"
           else
             "ไม่รู้จัก"
           end
puts day_type  # => วันทำงาน
```

### case กับ type matching

```crystal
def describe(value) : String
  case value
  when Int32
    "จำนวนเต็ม: #{value}"
  when Float64
    "ทศนิยม: #{value}"
  when String
    "ข้อความ: \"#{value}\""
  when Bool
    "บูลีน: #{value}"
  when Nil
    "ไม่มีค่า"
  else
    "ประเภทอื่น: #{value.class}"
  end
end

puts describe(42)       # => จำนวนเต็ม: 42
puts describe(3.14)     # => ทศนิยม: 3.14
puts describe("hello")  # => ข้อความ: "hello"
puts describe(true)     # => บูลีน: true
puts describe(nil)      # => ไม่มีค่า
```

### case กับ Range

```crystal
def tax_rate(income : Int32) : Float64
  case income
  when 0..150_000
    0.0
  when 150_001..300_000
    0.05
  when 300_001..500_000
    0.10
  when 500_001..750_000
    0.15
  when 750_001..1_000_000
    0.20
  else
    0.35
  end
end

[100_000, 250_000, 450_000, 800_000, 1_500_000].each do |income|
  rate = tax_rate(income)
  puts "รายได้ #{income.format}: อัตราภาษี #{(rate * 100).to_i}%"
end
```

### case กับ Regex

```crystal
def categorize_input(input : String) : String
  case input
  when /^\d+$/
    "ตัวเลขทั้งหมด"
  when /^[a-zA-Z]+$/
    "ตัวอักษรทั้งหมด"
  when /^[a-zA-Z0-9]+$/
    "ตัวเลขและตัวอักษร"
  when /^\s*$/
    "ว่างเปล่า"
  else
    "ผสมกัน"
  end
end

puts categorize_input("12345")    # => ตัวเลขทั้งหมด
puts categorize_input("hello")    # => ตัวอักษรทั้งหมด
puts categorize_input("hello1")   # => ตัวเลขและตัวอักษร
puts categorize_input("   ")      # => ว่างเปล่า
puts categorize_input("hi!")      # => ผสมกัน
```

---

## 4. Compound Conditionals

### &&, || และ !

```crystal
x = 15
y = 20

# AND
puts "x อยู่ใน [10, 20]" if x >= 10 && x <= 20

# OR
puts "อย่างน้อยหนึ่งตัวเป็นบวก" if x > 0 || y > 0

# NOT
active = true
puts "inactive" if !active

# ใช้ && และ || ในการ assign
name = nil
display_name = name && name.upcase || "ไม่ระบุชื่อ"
puts display_name  # => ไม่ระบุชื่อ
```

### Short-circuit Evaluation

```crystal
# && หยุดประเมินถ้าตัวแรก false
def check_name(name : String?) : Bool
  puts "กำลังตรวจสอบ: #{name.inspect}"
  !name.nil? && name.size > 0
end

puts check_name(nil)    # ไม่แสดง "กำลังตรวจสอบ" เพราะ short-circuit
puts check_name("")     # => false
puts check_name("Alice") # => true

# || หยุดประเมินถ้าตัวแรก true
def expensive_default : String
  puts "คำนวณค่า default..."
  "default_value"
end

config_value = nil
result = config_value || expensive_default
# "คำนวณค่า default..." จะแสดงเพราะ config_value เป็น nil
```

### Compound condition ที่ซับซ้อน

```crystal
def can_access?(user_age : Int32, has_permission : Bool, is_admin : Bool) : Bool
  # เงื่อนไข: (อายุ >= 18 หรือ เป็น admin) และ มีสิทธิ์
  (user_age >= 18 || is_admin) && has_permission
end

puts can_access?(20, true, false)   # => true  (ผู้ใหญ่ + มีสิทธิ์)
puts can_access?(15, true, true)    # => true  (admin + มีสิทธิ์)
puts can_access?(15, true, false)   # => false (เยาวชน + ไม่ใช่ admin)
puts can_access?(20, false, false)  # => false (ผู้ใหญ่ แต่ไม่มีสิทธิ์)
```

---

## 5. Conditional Assignment

### ||= (assign ถ้าค่าปัจจุบันเป็น falsy)

```crystal
# ||= assign ค่าถ้า variable เป็น nil หรือ false
name = nil
name ||= "Anonymous"
puts name  # => Anonymous

# ถ้ามีค่าอยู่แล้ว จะไม่เปลี่ยน
name ||= "Alice"
puts name  # => Anonymous (ยังคง anonymous เพราะมีค่าแล้ว)

# ใช้กับ Hash
cache = {} of String => Int32
cache["key"] ||= expensive_computation = 42
cache["key"] ||= 99  # ไม่เปลี่ยน
puts cache["key"]  # => 42
```

### &&= (assign ถ้าค่าปัจจุบันเป็น truthy)

```crystal
# &&= assign ค่าถ้า variable ไม่เป็น nil/false
name = "Alice"
name &&= name.upcase
puts name  # => ALICE

# ถ้าเป็น nil จะไม่ assign
other = nil
other &&= "changed"
puts other.inspect  # => nil (ไม่เปลี่ยน)
```

### ตัวอย่างการใช้จริง

```crystal
# Memoization ด้วย ||=
class ExpensiveCalculator
  @result : Int32? = nil
  
  def result : Int32
    @result ||= begin
      puts "กำลังคำนวณ (ครั้งเดียวเท่านั้น)..."
      sleep(0)  # จำลองการคำนวณนาน
      42 * 1000
    end
  end
end

calc = ExpensiveCalculator.new
puts calc.result  # คำนวณ
puts calc.result  # ใช้ cache (ไม่คำนวณซ้ำ)
puts calc.result  # ใช้ cache

# Counter pattern ด้วย ||=
visits = {} of String => Int32
pages = ["home", "about", "home", "contact", "home"]

pages.each do |page|
  visits[page] ||= 0
  visits[page] += 1
end

visits.each { |page, count| puts "#{page}: #{count}" }
```

---

## 6. unless as Expression

`unless` คือ `if not` และก็เป็น expression ด้วย:

```crystal
# unless พื้นฐาน
logged_in = false
message = unless logged_in
  "กรุณาเข้าสู่ระบบก่อน"
else
  "ยินดีต้อนรับ!"
end
puts message  # => กรุณาเข้าสู่ระบบก่อน

# postfix unless
age = 15
puts "ไม่อนุญาต" unless age >= 18
puts "อนุญาต" unless age < 18

# unless ใน method
def process(data : String?)
  return "ไม่มีข้อมูล" unless data
  return "ข้อมูลว่าง" unless data.size > 0
  "ผลลัพธ์: #{data.upcase}"
end

puts process(nil)     # => ไม่มีข้อมูล
puts process("")      # => ข้อมูลว่าง
puts process("hello") # => ผลลัพธ์: HELLO
```

---

## 7. Conditional Expressions ขั้นสูง

### Pattern Matching ใน condition

```crystal
# ตรวจสอบ type และ assign พร้อมกัน
def process_value(value : Int32 | String | Nil)
  if value.is_a?(Int32)
    puts "จำนวนเต็ม x 2 = #{value * 2}"
  elsif value.is_a?(String)
    puts "ข้อความ uppercase = #{value.upcase}"
  else
    puts "ไม่มีค่า"
  end
end

process_value(21)
process_value("hello")
process_value(nil)
```

### Conditional Expression ใน Interpolation

```crystal
items = ["แอปเปิ้ล", "กล้วย", "ส้ม"]
puts "มี #{items.size} รายการ (#{items.size == 1 ? "1 ชิ้น" : "หลายชิ้น"})"

# ใช้ case ใน interpolation (ต้องมี begin...end)
score = 85
puts "คะแนน #{score}: #{
  case score
  when 90..100 then "A"
  when 80..89 then "B"
  when 70..79 then "C"
  else "F"
  end
}"
```

### Guard Clauses

```crystal
# แทนที่ if/else ลึกๆ ด้วย guard clauses
def process_order(order : Hash(String, String | Int32 | Bool))
  # guard clauses - return early เมื่อเงื่อนไขไม่ผ่าน
  return "ไม่มีรายการสินค้า" unless order["items"]?
  return "ยังไม่ได้ระบุที่อยู่" unless order["address"]?
  return "ยอดรวมต้องมากกว่า 0" unless (order["total"]? || 0) > 0
  
  # ถ้าผ่านทุก guard แล้วค่อยทำงานหลัก
  "ดำเนินการสั่งซื้อสำเร็จ"
end

# Test
puts process_order({} of String => String | Int32 | Bool)
puts process_order({"items" => "book", "address" => "Bangkok", "total" => 100})
```

---

## 8. ตัวอย่างการใช้งานจริง

### Discount Calculator

```crystal
def calculate_discount(price : Float64, member : Bool, quantity : Int32) : Float64
  base_discount = case quantity
                  when .>= 100 then 0.20
                  when .>= 50  then 0.15
                  when .>= 20  then 0.10
                  when .>= 10  then 0.05
                  else              0.0
                  end
  
  member_bonus = member ? 0.05 : 0.0
  
  total_discount = [base_discount + member_bonus, 0.30].min  # สูงสุด 30%
  price * (1 - total_discount)
end

[
  {price: 1000.0, member: true, qty: 25},
  {price: 500.0, member: false, qty: 5},
  {price: 2000.0, member: true, qty: 100},
].each do |order|
  final = calculate_discount(order[:price], order[:member], order[:qty])
  puts "ราคา #{order[:price]}, สมาชิก: #{order[:member]}, จำนวน: #{order[:qty]} => #{final.round(2)}"
end
```

### Form Validator

```crystal
def validate_field(name : String, value : String?, rules : Hash(String, Bool | Int32)) : String?
  return "#{name} ห้ามว่าง" if (rules["required"]? == true) && (value.nil? || value.empty?)
  return nil unless value  # ถ้าไม่ required และ nil ก็โอเค
  
  min_length = rules["min_length"]?
  return "#{name} ต้องมีอย่างน้อย #{min_length} ตัวอักษร" if min_length.is_a?(Int32) && value.size < min_length
  
  max_length = rules["max_length"]?
  return "#{name} ต้องมีไม่เกิน #{max_length} ตัวอักษร" if max_length.is_a?(Int32) && value.size > max_length
  
  nil  # ผ่านทุก validation
end

# ทดสอบ
[
  {name: "ชื่อ", value: nil, rules: {"required" => true, "min_length" => 2} of String => Bool | Int32},
  {name: "ชื่อ", value: "A", rules: {"required" => true, "min_length" => 2} of String => Bool | Int32},
  {name: "ชื่อ", value: "Alice", rules: {"required" => true, "min_length" => 2} of String => Bool | Int32},
].each do |test|
  error = validate_field(test[:name], test[:value], test[:rules])
  puts error ? "Error: #{error}" : "ผ่าน!"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: FizzBuzz ด้วย Ternary

เขียน FizzBuzz แบบกระชับที่สุดโดยใช้ ternary:

```crystal
# TODO: เขียน FizzBuzz 1-20 โดยใช้ ternary operator
# FizzBuzz: หารด้วย 3 ได้ = "Fizz", หารด้วย 5 ได้ = "Buzz", ทั้งคู่ = "FizzBuzz"
(1..20).each do |n|
  # TODO: ใช้ ternary เพื่อแสดงผล
end
```

### แบบฝึกหัดที่ 2: Grade Calculator ด้วย case

```crystal
def letter_grade(score : Int32) : String
  # TODO: แปลงคะแนน (0-100) เป็นเกรด A, B+, B, C+, C, D+, D, F
  # A: 80-100, B+: 75-79, B: 70-74, C+: 65-69, C: 60-64, D+: 55-59, D: 50-54, F: 0-49
end

[95, 77, 72, 66, 61, 56, 51, 40].each do |score|
  puts "#{score}: #{letter_grade(score)}"
end
```

### แบบฝึกหัดที่ 3: Memoization ด้วย ||=

```crystal
# สร้าง class ที่ cache ผลการคำนวณ
class MemoizedCalc
  @cache = {} of Int32 => Int64
  
  def fibonacci(n : Int32) : Int64
    # TODO: ใช้ @cache[n] ||= สำหรับ memoization
    # base case: n <= 1 return n
    # recursive case: fibonacci(n-1) + fibonacci(n-2)
  end
end

calc = MemoizedCalc.new
puts calc.fibonacci(10)  # => 55
puts calc.fibonacci(20)  # => 6765
puts calc.fibonacci(30)  # => 832040
```

### แบบฝึกหัดที่ 4: Conditional Chain

```crystal
# สร้างฟังก์ชันที่ classify อุณหภูมิ
def weather_description(temp : Float64, humidity : Int32) : String
  # TODO: ใช้ case หรือ if/elsif เพื่อ classify
  # temp > 35 && humidity > 80 => "ร้อนอบอ้าว"
  # temp > 35 => "ร้อนมาก"
  # temp > 25 && humidity > 80 => "อบอ้าว"
  # temp > 25 => "อบอุ่น"
  # temp > 15 => "เย็นสบาย"
  # temp > 5  => "หนาว"
  # else      => "หนาวมาก"
end

puts weather_description(38.0, 85)  # => ร้อนอบอ้าว
puts weather_description(38.0, 60)  # => ร้อนมาก
puts weather_description(10.0, 50)  # => หนาว
```

---

## สรุป

| Syntax | รูปแบบ | เมื่อใช้ |
|--------|--------|---------|
| Ternary | `cond ? a : b` | เงื่อนไขง่ายๆ 2 ทาง |
| if expression | `x = if ... end` | กำหนดค่าจาก if |
| case expression | `x = case ... end` | กำหนดค่าจาก pattern matching |
| postfix if | `expr if cond` | กระชับ 1 บรรทัด |
| postfix unless | `expr unless cond` | กระชับกลับเงื่อนไข |
| `\|\|=` | `x \|\|= default` | กำหนด default ถ้า nil/false |
| `&&=` | `x &&= transform` | transform ถ้าไม่ nil/false |

### เมื่อไหร่ควรใช้อะไร

- **Ternary**: เงื่อนไขง่าย 2 ทาง เช่น `size == 1 ? "item" : "items"`
- **if expression**: เงื่อนไข 2-3 ทาง หรือเมื่อต้องการกำหนดค่า
- **case expression**: Pattern matching หลายรูปแบบ หรือ type-based logic
- **`||=`**: Default values, memoization, lazy initialization
- **Guard clauses**: Early return เพื่อลด nesting
