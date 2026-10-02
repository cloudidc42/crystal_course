# Part 021: Methods พื้นฐาน

## บทนำ

**Method** (เมธอด) คือหน่วยของโค้ดที่มีชื่อ ทำงานอย่างใดอย่างหนึ่ง และสามารถเรียกใช้ซ้ำได้ Method เป็นส่วนสำคัญที่สุดในการจัดระเบียบโปรแกรม ทำให้โค้ด DRY (Don't Repeat Yourself) และอ่านง่าย

---

## 1. def/end - การนิยาม Method

```crystal
# รูปแบบพื้นฐาน
def greet
  puts "สวัสดี!"
end

greet  # เรียกใช้

# Method พร้อม parameter
def greet_person(name : String)
  puts "สวัสดี #{name}!"
end

greet_person("Alice")  # => สวัสดี Alice!
```

### Method กับ Return Value

```crystal
# Method ส่งค่ากลับ (implicit return)
def add(a : Int32, b : Int32) : Int32
  a + b  # ค่าสุดท้ายเป็น return value
end

result = add(3, 4)
puts result  # => 7

# Method กับ explicit return
def absolute_value(n : Int32) : Int32
  return n if n >= 0
  -n
end

puts absolute_value(-5)  # => 5
puts absolute_value(3)   # => 3
```

---

## 2. การตั้งชื่อ Method

Crystal มีหลักการตั้งชื่อ method ที่ชัดเจน:

```crystal
# snake_case สำหรับชื่อปกติ
def calculate_total
end

def get_user_name
end

# ตั้งชื่อให้สื่อความหมาย
def find_maximum_value(numbers : Array(Int32)) : Int32
  numbers.max
end

# ชื่อที่ชัดเจนกว่า
def is_valid_email?(email : String) : Bool  # ? method
  email.includes?("@") && email.includes?(".")
end

def sort_users!  # ! method
  # mutates something in place
end
```

---

## 3. Return Statement

### Implicit Return

```crystal
# ใน Crystal ค่าสุดท้ายใน method จะถูก return อัตโนมัติ
def double(n : Int32) : Int32
  n * 2  # implicit return
end

def max_of_three(a : Int32, b : Int32, c : Int32) : Int32
  if a >= b && a >= c
    a
  elsif b >= c
    b
  else
    c
  end
  # ค่าสุดท้ายของ if expression จะถูก return
end

puts double(5)          # => 10
puts max_of_three(3, 7, 5)  # => 7
```

### Explicit Return

```crystal
# explicit return ออกจาก method ทันที
def find_first_even(numbers : Array(Int32)) : Int32?
  numbers.each do |n|
    return n if n.even?  # return ทันทีเมื่อพบ
  end
  nil  # ถ้าไม่พบ
end

puts find_first_even([1, 3, 7, 8, 9]).inspect  # => 8
puts find_first_even([1, 3, 7, 9]).inspect     # => nil
```

### Return หลายค่า (Tuple)

```crystal
def min_max(arr : Array(Int32)) : {Int32, Int32}
  {arr.min, arr.max}
end

min, max = min_max([3, 1, 4, 1, 5, 9, 2, 6])
puts "Min: #{min}, Max: #{max}"
# => Min: 1, Max: 9
```

---

## 4. การเรียก Method

### เรียกพร้อมและไม่พร้อม Parentheses

```crystal
def say_hello
  puts "Hello!"
end

say_hello()   # พร้อม parentheses
say_hello     # ไม่พร้อม parentheses - ใช้ได้เหมือนกัน

def add(a : Int32, b : Int32) : Int32
  a + b
end

puts add(3, 4)   # พร้อม parentheses
puts add 3, 4    # ไม่พร้อม - ใช้ได้แต่อาจสับสน
```

### Method Chaining

```crystal
# Crystal รองรับ method chaining
result = "hello world"
  .split(" ")
  .map { |w| w.capitalize }
  .join(", ")

puts result  # => Hello, World

# หรือแบบ inline
puts "crystal".upcase.reverse  # => LATSHRC
```

### Method เป็น Argument

```crystal
def apply(value : Int32, &operation : Int32 -> Int32) : Int32
  operation.call(value)
end

result = apply(5) { |n| n * n }
puts result  # => 25
```

---

## 5. Method ที่ลงท้ายด้วย ?

Method ที่ลงท้ายด้วย `?` มักส่งค่า Bool กลับ:

```crystal
# predicate methods
def positive?(n : Int32) : Bool
  n > 0
end

def empty_string?(s : String) : Bool
  s.strip.empty?
end

def valid_age?(age : Int32) : Bool
  (0..150).includes?(age)
end

puts positive?(5)       # => true
puts positive?(-3)      # => false
puts empty_string?("  ") # => true
puts valid_age?(25)     # => true
puts valid_age?(200)    # => false
```

### Crystal's built-in ? methods

```crystal
str = "Hello"
arr = [1, 2, 3]
num = 10

puts str.empty?      # => false
puts arr.empty?      # => false
puts arr.includes?(2) # => true
puts num.even?       # => true
puts num.odd?        # => false
puts num.zero?       # => false
puts str.starts_with?("He")  # => true
puts str.ends_with?("lo")    # => true
```

---

## 6. Method ที่ลงท้ายด้วย !

Method ที่ลงท้ายด้วย `!` มักหมายถึง "เปลี่ยนแปลง in-place" หรือ "อันตราย":

```crystal
# ! หมายถึง mutate in-place
arr = [3, 1, 4, 1, 5, 9, 2, 6]

# sort (ส่ง array ใหม่)
sorted_new = arr.sort
puts arr.inspect       # => [3, 1, 4, 1, 5, 9, 2, 6] (เดิม)
puts sorted_new.inspect  # => [1, 1, 2, 3, 4, 5, 6, 9]

# sort! (เปลี่ยน in-place)
arr.sort!
puts arr.inspect  # => [1, 1, 2, 3, 4, 5, 6, 9] (เปลี่ยนแล้ว)
```

### Custom ! Methods

```crystal
class TextBuffer
  @content : String = ""
  
  def content
    @content
  end
  
  # ไม่ mutate - ส่งค่าใหม่กลับ
  def upcase
    @content.upcase
  end
  
  # mutate in-place
  def upcase!
    @content = @content.upcase
    self
  end
  
  def append(text : String)
    TextBuffer.new.tap { |b| b.@content = @content + text }
  end
  
  def append!(text : String)
    @content += text
    self  # return self เพื่อ chaining
  end
end

buf = TextBuffer.new
buf.append!("hello")
buf.append!(" world")
buf.upcase!
puts buf.content  # => HELLO WORLD
```

---

## 7. Method ใน Class vs Top-level

```crystal
# Top-level method (global-ish)
def format_number(n : Float64, decimals : Int32 = 2) : String
  "%.#{decimals}f" % n
end

puts format_number(3.14159)   # => 3.14
puts format_number(3.14159, 4) # => 3.1416

# Method ใน class
class Calculator
  def add(a : Float64, b : Float64) : Float64
    a + b
  end
  
  def subtract(a : Float64, b : Float64) : Float64
    a - b
  end
  
  # class method
  def self.version : String
    "1.0.0"
  end
end

calc = Calculator.new
puts calc.add(3.5, 2.1)       # => 5.6
puts Calculator.version        # => 1.0.0
```

---

## 8. Method Visibility

```crystal
class BankAccount
  def initialize(@balance : Float64)
  end
  
  # public method (เรียกได้จากภายนอก)
  def deposit(amount : Float64)
    validate_amount!(amount)
    @balance += amount
  end
  
  def balance : Float64
    @balance
  end
  
  # protected method (เรียกได้ใน class และ subclass)
  protected def transfer_to(other : BankAccount, amount : Float64)
    validate_amount!(amount)
    raise "ยอดเงินไม่เพียงพอ" if amount > @balance
    @balance -= amount
    other.@balance += amount
  end
  
  # private method (เรียกได้เฉพาะใน class)
  private def validate_amount!(amount : Float64)
    raise ArgumentError.new("จำนวนต้องมากกว่า 0") if amount <= 0
  end
end

account = BankAccount.new(1000.0)
account.deposit(500.0)
puts account.balance  # => 1500.0
```

---

## 9. Recursive Methods

```crystal
# Factorial
def factorial(n : Int32) : Int64
  return 1 if n <= 1
  n * factorial(n - 1)
end

puts factorial(5)   # => 120
puts factorial(10)  # => 3628800

# Fibonacci (recursive - ช้า)
def fib(n : Int32) : Int32
  return n if n <= 1
  fib(n - 1) + fib(n - 2)
end

puts fib(10)  # => 55

# Fibonacci (iterative - เร็วกว่า)
def fib_fast(n : Int32) : Int64
  return n.to_i64 if n <= 1
  a, b = 0_i64, 1_i64
  (n - 1).times { a, b = b, a + b }
  b
end

puts fib_fast(50)  # => 12586269025
```

### Recursive Tree Traversal

```crystal
# Binary Tree
class TreeNode(T)
  property value : T
  property left : TreeNode(T)?
  property right : TreeNode(T)?
  
  def initialize(@value : T)
  end
end

def inorder_traversal(node : TreeNode(Int32)?)
  return unless node
  inorder_traversal(node.left)
  print "#{node.value} "
  inorder_traversal(node.right)
end

# สร้าง BST ง่ายๆ
root = TreeNode.new(5)
root.left = TreeNode.new(3)
root.right = TreeNode.new(7)
root.left.not_nil!.left = TreeNode.new(1)
root.left.not_nil!.right = TreeNode.new(4)

inorder_traversal(root)
puts
# => 1 3 4 5 7
```

---

## 10. Method Overloading

Crystal รองรับ method overloading ตาม type:

```crystal
def process(value : Int32) : String
  "จำนวนเต็ม: #{value}"
end

def process(value : Float64) : String
  "ทศนิยม: #{value.round(2)}"
end

def process(value : String) : String
  "ข้อความ: #{value.upcase}"
end

def process(value : Array(Int32)) : String
  "Array: #{value.sum} (ผลรวม)"
end

puts process(42)           # => จำนวนเต็ม: 42
puts process(3.14)         # => ทศนิยม: 3.14
puts process("hello")      # => ข้อความ: HELLO
puts process([1, 2, 3])    # => Array: 6 (ผลรวม)
```

---

## 11. ตัวอย่าง Methods จริง

### Text Processing

```crystal
def word_count(text : String) : Int32
  text.split(/\s+/).reject(&.empty?).size
end

def capitalize_words(text : String) : String
  text.split(" ").map(&.capitalize).join(" ")
end

def truncate(text : String, max_length : Int32, suffix : String = "...") : String
  return text if text.size <= max_length
  text[0, max_length - suffix.size] + suffix
end

puts word_count("Hello World Crystal")  # => 3
puts capitalize_words("hello world")     # => Hello World
puts truncate("This is a long text that needs to be truncated", 20)
# => This is a long te...
```

### Mathematical Methods

```crystal
def clamp(value : Float64, min : Float64, max : Float64) : Float64
  [min, [value, max].min].max
end

def lerp(a : Float64, b : Float64, t : Float64) : Float64
  a + (b - a) * clamp(t, 0.0, 1.0)
end

def degrees_to_radians(degrees : Float64) : Float64
  degrees * Math::PI / 180.0
end

def radians_to_degrees(radians : Float64) : Float64
  radians * 180.0 / Math::PI
end

puts clamp(15.0, 0.0, 10.0)    # => 10.0
puts lerp(0.0, 100.0, 0.25)    # => 25.0
puts degrees_to_radians(90.0).round(4)  # => 1.5708
```

### Collection Helpers

```crystal
def flatten_once(arr : Array(Array(Int32))) : Array(Int32)
  result = [] of Int32
  arr.each { |sub| sub.each { |item| result << item } }
  result
end

def chunk_array(arr : Array(Int32), size : Int32) : Array(Array(Int32))
  result = [] of Array(Int32)
  i = 0
  while i < arr.size
    result << arr[i, size]
    i += size
  end
  result
end

nested = [[1, 2], [3, 4], [5, 6]]
puts flatten_once(nested).inspect    # => [1, 2, 3, 4, 5, 6]

data = (1..10).to_a
chunks = chunk_array(data, 3)
chunks.each { |c| puts c.inspect }
# [1, 2, 3]
# [4, 5, 6]
# [7, 8, 9]
# [10]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Utility Methods

สร้าง methods utility สำหรับ string:

```crystal
# TODO: implement แต่ละ method

def palindrome?(s : String) : Bool
  # ตรวจสอบว่าเป็น palindrome หรือไม่
  # "racecar" => true, "hello" => false
  # ไม่สนใจตัวพิมพ์ใหญ่/เล็กและช่องว่าง
end

def count_vowels(s : String) : Int32
  # นับจำนวน vowels (a, e, i, o, u)
end

def reverse_words(s : String) : String
  # กลับลำดับคำ: "Hello World" => "World Hello"
end

def remove_duplicates(s : String) : String
  # ลบตัวอักษรซ้ำ: "programming" => "progamin"
end

# Test
puts palindrome?("racecar")      # => true
puts palindrome?("A man a plan") # => false
puts count_vowels("Hello World") # => 3
puts reverse_words("Hello World Crystal")  # => "Crystal World Hello"
puts remove_duplicates("programming")       # => "progamin"
```

### แบบฝึกหัดที่ 2: Math Helpers

```crystal
def is_prime?(n : Int32) : Bool
  # TODO: ตรวจสอบว่าเป็นจำนวนเฉพาะ
end

def prime_factors(n : Int32) : Array(Int32)
  # TODO: หาตัวประกอบเฉพาะ
  # prime_factors(12) => [2, 2, 3]
end

def gcd(a : Int32, b : Int32) : Int32
  # TODO: หาตัวหารร่วมมาก (Greatest Common Divisor)
  # gcd(12, 8) => 4
end

def lcm(a : Int32, b : Int32) : Int32
  # TODO: หาตัวคูณร่วมน้อย (Least Common Multiple)
  # lcm(4, 6) => 12
end
```

### แบบฝึกหัดที่ 3: ? และ ! Methods

```crystal
class StringStack
  def initialize
    @data = [] of String
  end
  
  def push(item : String) : self
    # TODO: เพิ่ม item เข้า stack
  end
  
  def pop : String?
    # TODO: ลบและส่งคืน item บนสุด
  end
  
  def peek : String?
    # TODO: ดู item บนสุดโดยไม่ลบ
  end
  
  def empty? : Bool
    # TODO: ตรวจสอบว่าว่างหรือไม่
  end
  
  def clear! : self
    # TODO: ล้าง stack ทั้งหมด
  end
  
  def size : Int32
    # TODO: จำนวน item
  end
end

stack = StringStack.new
stack.push("first").push("second").push("third")
puts stack.size       # => 3
puts stack.peek       # => "third"
puts stack.pop        # => "third"
puts stack.size       # => 2
puts stack.empty?     # => false
stack.clear!
puts stack.empty?     # => true
```

---

## สรุป

| หัวข้อ | รูปแบบ | หมายเหตุ |
|--------|--------|----------|
| นิยาม method | `def name ... end` | |
| Return type | `def name : ReturnType` | optional แต่แนะนำ |
| Implicit return | ค่าสุดท้าย | ไม่ต้อง return |
| Explicit return | `return value` | ออกก่อนสิ้น method |
| ? method | `def valid?` | ส่ง Bool กลับ |
| ! method | `def sort!` | mutate in-place |
| Method call | `name()` หรือ `name` | parentheses optional |
| Overloading | type ต่างกัน | Crystal เลือก method ที่ตรง |

### Best Practices

1. **ตั้งชื่อให้สื่อความหมาย** - ชื่อ method ควรบอกว่าทำอะไร
2. **Method สั้น** - ถ้ายาวเกิน 15-20 บรรทัด ให้แยกเป็น helper methods
3. **ใช้ implicit return** - ไม่ต้องเขียน `return` บรรทัดสุดท้าย
4. **Type annotation** - ระบุ return type เพื่อความชัดเจน
5. **? สำหรับ predicate** - method ที่ถามว่า "ใช่ไหม?"
6. **! สำหรับ mutating** - method ที่เปลี่ยนแปลงข้อมูล in-place
