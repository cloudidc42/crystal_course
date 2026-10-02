# ตอนที่ 11: If / Unless / Elsif

## บทนำ

การควบคุมการทำงานของโปรแกรมด้วย conditional statements เป็นทักษะพื้นฐานที่สำคัญ Crystal มี if, unless, elsif และรูปแบบ inline ที่ทำให้โค้ดกระชับและอ่านง่าย

---

## 11.1 If / Else / End พื้นฐาน

```crystal
# รูปแบบพื้นฐาน
temperature = 35

if temperature > 30
  puts "ร้อนมาก!"
else
  puts "อากาศเย็นสบาย"
end

# ไม่จำเป็นต้องมี else
age = 20
if age >= 18
  puts "ผู้ใหญ่แล้ว"
end

# if เดี่ยว
score = 85
if score >= 60
  puts "ผ่าน"
end
```

---

## 11.2 Elsif

```crystal
grade = 75

if grade >= 90
  puts "A"
elsif grade >= 80
  puts "B"
elsif grade >= 70
  puts "C"
elsif grade >= 60
  puts "D"
else
  puts "F"
end
# => C

# หลาย elsif ได้ไม่จำกัด
day = "Wednesday"

if day == "Monday"
  puts "วันจันทร์"
elsif day == "Tuesday"
  puts "วันอังคาร"
elsif day == "Wednesday"
  puts "วันพุธ"
elsif day == "Thursday"
  puts "วันพฤหัส"
elsif day == "Friday"
  puts "วันศุกร์"
else
  puts "วันหยุด"
end
# => วันพุธ
```

---

## 11.3 Unless

`unless` คือ `if not` - ทำงานเมื่อเงื่อนไขเป็น false

```crystal
logged_in = false

unless logged_in
  puts "กรุณาเข้าสู่ระบบก่อน"
end
# => กรุณาเข้าสู่ระบบก่อน

# เทียบเท่ากับ
if !logged_in
  puts "กรุณาเข้าสู่ระบบก่อน"
end

# unless กับ else
is_holiday = false

unless is_holiday
  puts "วันทำงาน - ต้องไปทำงาน"
else
  puts "วันหยุด - พักผ่อนได้"
end
# => วันทำงาน - ต้องไปทำงาน
```

### เมื่อใดควรใช้ unless

```crystal
# ดี: ใช้ unless เมื่อเงื่อนไขเป็น negative
errors = [] of String
unless errors.empty?
  puts "มีข้อผิดพลาด: #{errors.join(", ")}"
end

# ดี: ใช้ unless สำหรับ guard condition
def send_email(address : String)
  unless address.includes?("@")
    puts "Email ไม่ถูกต้อง"
    return
  end
  puts "ส่ง email ไปที่ #{address}"
end

# ไม่ดี: อย่าใช้ unless กับ else เมื่อทำให้สับสน
# ถ้า unless...else ทำให้อ่านยากกว่า ให้ใช้ if แทน
```

---

## 11.4 Inline If / Unless (Postfix)

```crystal
# Inline if: statement if condition
puts "ผ่าน!" if score >= 60
puts "ไม่ผ่าน!" if score < 60

# Inline unless: statement unless condition
puts "ยังไม่ถึงอายุ" unless age >= 18

# ตัวอย่างจริง
name = "Alice"
puts "สวัสดี, #{name}!" unless name.empty?

items = [1, 2, 3]
puts "มีสินค้า #{items.size} ชิ้น" unless items.empty?

# เหมาะสำหรับ guard clause
def process(value : Int32)
  return if value < 0
  puts "ประมวลผล: #{value}"
end

process(-5)   # ไม่มี output
process(10)   # => ประมวลผล: 10
```

### ข้อควรระวัง

```crystal
# ใช้ inline ได้กับ statement เดียวเท่านั้น
# ถ้ามีหลาย statement ต้องใช้แบบ block
x = 10

# ดี
puts x if x > 5

# ไม่ดี (อ่านยาก)
# x = x * 2; puts x if x > 5  # อย่าทำแบบนี้

# ควรเป็น
if x > 5
  x = x * 2
  puts x
end
```

---

## 11.5 If เป็น Expression

ใน Crystal, `if` คือ expression ที่คืนค่าได้

```crystal
# if คืนค่าจาก branch ที่ทำงาน
result = if score >= 60
  "ผ่าน"
else
  "ไม่ผ่าน"
end
puts result  # => ผ่าน

# ใช้ใน interpolation
status = "สถานะ: #{if temperature > 30 then "ร้อน" else "เย็น" end}"
puts status

# กำหนดค่าด้วย if expression
category = if age < 13
  "เด็ก"
elsif age < 18
  "วัยรุ่น"
elsif age < 65
  "ผู้ใหญ่"
else
  "ผู้สูงอายุ"
end

puts category
```

### ค่าที่คืนเมื่อไม่มี else

```crystal
# ถ้าไม่มี else และเงื่อนไขเป็น false จะคืน nil
result = if false
  "จะไม่ได้ค่านี้"
end
puts result.inspect  # => nil

# ระวัง type inference
value : String | Nil = if age >= 18
  "ผู้ใหญ่"
end
# ถ้าไม่มี else value อาจเป็น nil
```

---

## 11.6 Truthiness Rules ใน Crystal

```crystal
# เฉพาะ nil และ false เท่านั้นที่เป็น falsy
# ทุกอย่างอื่นเป็น truthy!

# Falsy values
puts "nil เป็น falsy" unless nil
puts "false เป็น falsy" unless false

# Truthy values (ทุกอย่างอื่น)
puts "0 เป็น truthy" if 0          # 0 เป็น truthy ใน Crystal!
puts "\"\" เป็น truthy" if ""      # string ว่างก็เป็น truthy!
puts "[] เป็น truthy" if [] of Int32  # array ว่างก็เป็น truthy!

# เปรียบเทียบกับภาษาอื่น
# ใน Ruby: 0, "" เป็น truthy (เหมือนกัน)
# ใน JavaScript: 0, "" เป็น falsy (ต่างกัน!)
# ใน Python: 0, "" เป็น falsy (ต่างกัน!)
```

### ผลกระทบต่อการเขียนโค้ด

```crystal
# ระวัง: ตรวจสอบ empty? แทน truthy check สำหรับ collections
items = [] of String

# ไม่ดี (ใช้ truthy check)
if items  # นี่จะ true เสมอ แม้จะว่างเปล่า!
  puts "มีสินค้า"
end

# ดี (ตรวจสอบ empty)
if !items.empty?
  puts "มีสินค้า"
end

# หรือ
unless items.empty?
  puts "มีสินค้า"
end

# ระวัง: ตรวจสอบ 0 สำหรับตัวเลข
count = 0
if count  # true! เพราะ 0 เป็น truthy ใน Crystal
  puts "มีจำนวน: #{count}"  # จะถูกพิมพ์!
end

# ถูกต้องกว่า
if count > 0
  puts "มีจำนวน: #{count}"
end
```

---

## 11.7 Complex Conditions

### Boolean Operators

```crystal
age = 25
has_license = true
has_car = false

# AND (&&)
if age >= 18 && has_license
  puts "ขับรถได้"
end

# OR (||)
if has_license || has_car
  puts "มีวิธีเดินทาง"
end

# NOT (!)
if !has_car
  puts "ไม่มีรถ"
end

# ผสมกัน
if age >= 18 && (has_license || has_car)
  puts "เดินทางได้"
end
```

### Short-circuit Evaluation

```crystal
# && หยุดทันทีเมื่อพบ false
def check_age(age : Int32) : Bool
  puts "ตรวจสอบอายุ"
  age >= 18
end

def check_license : Bool
  puts "ตรวจสอบใบขับขี่"
  true
end

# ถ้า age < 18 จะไม่เรียก check_license
if check_age(15) && check_license
  puts "ผ่านทุกเงื่อนไข"
end
# Output:
# ตรวจสอบอายุ
# (ไม่มี "ตรวจสอบใบขับขี่")

# || หยุดทันทีเมื่อพบ true
def is_admin : Bool
  puts "ตรวจสอบ admin"
  true
end

def is_superuser : Bool
  puts "ตรวจสอบ superuser"
  true
end

if is_admin || is_superuser
  puts "มีสิทธิ์"
end
# Output:
# ตรวจสอบ admin
# มีสิทธิ์
# (ไม่เรียก is_superuser)
```

### การใช้ parentheses

```crystal
# ใส่วงเล็บเพื่อความชัดเจน
a = true
b = false
c = true

# ไม่ชัดเจน
result = a || b && c

# ชัดเจนกว่า
result = a || (b && c)   # false || (true && true) = false || true = true
result = (a || b) && c   # (true || false) && true = true && true = true
```

---

## 11.8 Nested If

```crystal
# Nested if - ระวังไม่ให้ลึกเกินไป
user_role = "admin"
action = "delete"
resource_owner = false

if user_role == "admin"
  if action == "delete"
    if resource_owner
      puts "ลบได้ (เจ้าของ)"
    else
      puts "ลบได้ (admin)"
    end
  else
    puts "admin: action อื่น"
  end
else
  puts "ไม่ใช่ admin"
end

# ดีกว่า: แบน nested ด้วย && 
if user_role == "admin" && action == "delete"
  puts "admin ลบได้"
elsif user_role == "admin"
  puts "admin: action อื่น"
else
  puts "ไม่ใช่ admin"
end
```

### Refactoring Nested If

```crystal
# ก่อน: nested if ลึก
def can_access?(user_role : String, resource : String, action : String) : Bool
  if user_role == "admin"
    if resource == "users"
      if action == "delete"
        true
      elsif action == "read"
        true
      else
        false
      end
    else
      true
    end
  elsif user_role == "manager"
    if action == "delete"
      false
    else
      true
    end
  else
    action == "read"
  end
end

# หลัง: แบน ด้วย guard clauses
def can_access_v2?(user_role : String, resource : String, action : String) : Bool
  return true if user_role == "admin" && resource == "users" && action.in?("delete", "read")
  return true if user_role == "admin" && resource != "users"
  return false if user_role == "manager" && action == "delete"
  return true if user_role == "manager"
  action == "read"
end
```

---

## 11.9 Guard Clauses

Guard clauses คือการใช้ early return เพื่อลด nesting

```crystal
# แบบ nested (ไม่ดี)
def process_order(order : Hash(String, String))
  if order["status"]? == "pending"
    if order["items"]? && !order["items"].not_nil!.empty?
      if order["payment"]? == "paid"
        puts "ประมวลผล order: #{order}"
      else
        puts "รอการชำระเงิน"
      end
    else
      puts "ไม่มีสินค้าใน order"
    end
  else
    puts "Order ไม่ได้อยู่ใน pending status"
  end
end

# แบบ guard clause (ดีกว่า)
def process_order_v2(order : Hash(String, String))
  unless order["status"]? == "pending"
    puts "Order ไม่ได้อยู่ใน pending status"
    return
  end

  items = order["items"]?
  if items.nil? || items.empty?
    puts "ไม่มีสินค้าใน order"
    return
  end

  unless order["payment"]? == "paid"
    puts "รอการชำระเงิน"
    return
  end

  puts "ประมวลผล order: #{order}"
end
```

---

## 11.10 Early Returns

```crystal
# Early return ทำให้โค้ดอ่านง่ายขึ้น
def validate_username(username : String) : String?
  # Guard: ตรวจสอบ empty
  return "username ต้องไม่ว่างเปล่า" if username.empty?
  
  # Guard: ตรวจสอบความยาว
  return "username ต้องมีอย่างน้อย 3 ตัวอักษร" if username.size < 3
  return "username ต้องไม่เกิน 20 ตัวอักษร" if username.size > 20
  
  # Guard: ตรวจสอบ characters
  unless username.matches?(/\A[a-zA-Z0-9_]+\z/)
    return "username ใช้ได้เฉพาะตัวอักษร ตัวเลข และ _"
  end
  
  # ถ้าผ่านทุก guard แสดงว่า valid
  nil  # nil = ไม่มี error
end

# ทดสอบ
puts validate_username("").inspect          # => "username ต้องไม่ว่างเปล่า"
puts validate_username("ab").inspect        # => "username ต้องมีอย่างน้อย 3 ตัวอักษร"
puts validate_username("alice!").inspect    # => "username ใช้ได้เฉพาะตัวอักษร..."
puts validate_username("alice123").inspect  # => nil (valid)
```

---

## 11.11 If กับ Type Narrowing

Crystal narrowing type หลัง if

```crystal
value : Int32 | String | Nil = "hello"

if value.is_a?(String)
  # ใน block นี้ value เป็น String แน่นอน
  puts value.upcase  # => HELLO
end

if value.nil?
  # ที่นี่ value เป็น Nil
  puts "ว่างเปล่า"
else
  # ที่นี่ value เป็น Int32 | String
  puts value.to_s
end
```

### nil-safe narrowing

```crystal
name : String? = "Alice"

if name
  # ที่นี่ name เป็น String แน่ๆ (compiler ตรวจสอบ)
  puts name.upcase  # ไม่ต้องใช้ &. หรือ not_nil!
end

# อีกแบบ
def greet(name : String?)
  return "ไม่ระบุชื่อ" unless name
  "สวัสดี, #{name.upcase}!"  # name เป็น String ที่นี่
end

puts greet("alice")  # => สวัสดี, ALICE!
puts greet(nil)      # => ไม่ระบุชื่อ
```

---

## 11.12 Pattern Matching เบื้องต้น

```crystal
# if กับ is_a?
def process(value : Int32 | String | Array(Int32))
  if value.is_a?(Int32)
    puts "จำนวนเต็ม: #{value * 2}"
  elsif value.is_a?(String)
    puts "ข้อความ: #{value.upcase}"
  elsif value.is_a?(Array(Int32))
    puts "Array: #{value.sum}"
  end
end

process(21)           # => จำนวนเต็ม: 42
process("hello")      # => ข้อความ: HELLO
process([1, 2, 3, 4]) # => Array: 10
```

### Responds_to?

```crystal
def to_string(obj)
  if obj.responds_to?(:to_s)
    obj.to_s
  else
    "ไม่สามารถแปลงได้"
  end
end
```

---

## 11.13 ตัวอย่างโปรแกรมจริง: Grade Calculator

```crystal
class GradeCalculator
  def self.letter_grade(score : Float64) : String
    if score >= 95
      "A+"
    elsif score >= 90
      "A"
    elsif score >= 85
      "B+"
    elsif score >= 80
      "B"
    elsif score >= 75
      "C+"
    elsif score >= 70
      "C"
    elsif score >= 65
      "D+"
    elsif score >= 60
      "D"
    else
      "F"
    end
  end

  def self.gpa(letter : String) : Float64
    case letter
    when "A+" then 4.0
    when "A"  then 4.0
    when "B+" then 3.5
    when "B"  then 3.0
    when "C+" then 2.5
    when "C"  then 2.0
    when "D+" then 1.5
    when "D"  then 1.0
    else           0.0
    end
  end

  def self.status(score : Float64) : String
    if score >= 80
      "เกียรตินิยม"
    elsif score >= 60
      "ผ่าน"
    else
      "ไม่ผ่าน"
    end
  end
end

# ทดสอบ
scores = [95.0, 82.5, 71.0, 58.0, 100.0]

scores.each do |score|
  letter = GradeCalculator.letter_grade(score)
  gpa = GradeCalculator.gpa(letter)
  status = GradeCalculator.status(score)
  
  puts "คะแนน #{score}: #{letter} (GPA: #{gpa}) - #{status}"
end

# Output:
# คะแนน 95.0: A+ (GPA: 4.0) - เกียรตินิยม
# คะแนน 82.5: B (GPA: 3.0) - เกียรตินิยม
# คะแนน 71.0: C (GPA: 2.0) - ผ่าน
# คะแนน 58.0: F (GPA: 0.0) - ไม่ผ่าน
# คะแนน 100.0: A+ (GPA: 4.0) - เกียรตินิยม
```

---

## 11.14 ตัวอย่างโปรแกรมจริง: FizzBuzz

```crystal
# Classic FizzBuzz ด้วย if/elsif
def fizzbuzz(n : Int32) : String
  if n % 15 == 0
    "FizzBuzz"
  elsif n % 3 == 0
    "Fizz"
  elsif n % 5 == 0
    "Buzz"
  else
    n.to_s
  end
end

(1..20).each do |i|
  print fizzbuzz(i)
  print i < 20 ? ", " : "\n"
end

# Output:
# 1, 2, Fizz, 4, Buzz, Fizz, 7, 8, Fizz, Buzz, 11, Fizz, 13, 14, FizzBuzz, 16, 17, Fizz, 19, Buzz
```

---

## 11.15 ตัวอย่างโปรแกรมจริง: Access Control

```crystal
module AccessControl
  PERMISSIONS = {
    "admin"   => ["read", "write", "delete", "admin"],
    "editor"  => ["read", "write"],
    "viewer"  => ["read"],
    "guest"   => [] of String,
  }

  def self.can?(role : String, action : String) : Bool
    permissions = PERMISSIONS[role]?
    return false if permissions.nil?
    permissions.includes?(action)
  end

  def self.check_access(role : String, action : String, resource : String) : String
    if can?(role, action)
      "✓ #{role} สามารถ #{action} #{resource}"
    elsif PERMISSIONS.has_key?(role)
      "✗ #{role} ไม่มีสิทธิ์ #{action} #{resource}"
    else
      "✗ ไม่รู้จัก role: #{role}"
    end
  end
end

# ทดสอบ
roles = ["admin", "editor", "viewer", "guest", "unknown"]
actions = ["read", "write", "delete"]

roles.each do |role|
  actions.each do |action|
    puts AccessControl.check_access(role, action, "document")
  end
  puts "---"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: BMI Calculator
```crystal
# สร้าง method ที่รับน้ำหนัก (kg) และส่วนสูง (cm)
# แล้วคืนข้อความบอก category
# Underweight: < 18.5
# Normal: 18.5 - 24.9
# Overweight: 25 - 29.9
# Obese: >= 30
def bmi_category(weight_kg : Float64, height_cm : Float64) : String
  # TODO: คำนวณ BMI และคืน category
  # BMI = weight / (height_m^2)
end
```

### แบบฝึกหัดที่ 2: Season Checker
```crystal
# รับเดือน (1-12) แล้วคืนชื่อฤดู
# ใน Thailand:
# Hot season: 3-5
# Rainy season: 6-10
# Cool season: 11-2
def season(month : Int32) : String
  # TODO: ใช้ if/elsif/else
end
```

### แบบฝึกหัดที่ 3: Leap Year
```crystal
# ตรวจสอบปีอธิกสุรทิน
# ปีอธิกสุรทิน: หาร 400 ลงตัว หรือ (หาร 4 ลงตัว แต่ไม่ใช่หาร 100 ลงตัว)
def leap_year?(year : Int32) : Bool
  # TODO: implement
end
```

### แบบฝึกหัดที่ 4: Temperature Converter
```crystal
# รับค่า, unit ต้นทาง และ unit ปลายทาง
# รองรับ: celsius, fahrenheit, kelvin
def convert_temperature(value : Float64, from : String, to : String) : Float64?
  # TODO: implement
  # คืน nil ถ้า unit ไม่รู้จัก
end
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **if/else/end**: conditional พื้นฐาน
2. **elsif**: เพิ่ม branch เพิ่มเติม
3. **unless**: ตรงข้ามกับ if
4. **Inline if/unless**: กระชับสำหรับ statement เดียว
5. **If เป็น Expression**: ใช้ if เพื่อ return ค่า
6. **Truthiness**: เฉพาะ nil และ false เป็น falsy ใน Crystal
7. **Complex Conditions**: &&, ||, ! และ short-circuit
8. **Guard Clauses**: early return ลด nesting
9. **Type Narrowing**: Crystal แคบ type หลัง if

### เปรียบเทียบ if กับ unless

| สถานการณ์ | ใช้ if หรือ unless |
|-----------|-------------------|
| เงื่อนไขบวก `age >= 18` | if |
| เงื่อนไขลบ `!errors.empty?` | unless |
| มี else clause | if (เพื่อความชัดเจน) |
| Guard clause | unless มักอ่านง่ายกว่า |

การเลือกใช้ if/unless อย่างเหมาะสมทำให้โค้ดอ่านง่ายและสื่อความหมายได้ชัดเจน!
