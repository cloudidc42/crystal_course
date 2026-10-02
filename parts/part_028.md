# Part 28: Procs และ Lambdas

## บทนำ

Procs และ Lambdas เป็น first-class functions ใน Crystal ซึ่งหมายความว่าเราสามารถเก็บฟังก์ชันไว้ในตัวแปร ส่งผ่านเป็น argument และส่งคืนจากฟังก์ชันได้ Crystal มีวิธีสร้าง Proc/Lambda หลายแบบพร้อมความแตกต่างที่สำคัญระหว่าง Proc และ Lambda

---

## 1. Proc.new - การสร้าง Proc

### 1.1 วิธีสร้าง Proc

```crystal
# วิธีที่ 1: Proc.new พร้อมระบุ type
greeter = Proc(String, String).new { |name| "สวัสดี, #{name}!" }
puts greeter.call("สมชาย")  # => สวัสดี, สมชาย!

# วิธีที่ 2: ใช้ -> shorthand syntax (Lambda)
double = ->(x : Int32) { x * 2 }
puts double.call(5)  # => 10

# วิธีที่ 3: สร้างจาก block ใน method parameter
def capture(&block : Int32 -> Int32)
  block
end

square = capture { |x| x * x }
puts square.call(4)  # => 16
```

### 1.2 Proc.new ไม่มี Parameter

```crystal
action = Proc(Nil).new do
  puts "กำลังทำงาน..."
  puts "เสร็จแล้ว!"
end

action.call
# => กำลังทำงาน...
# => เสร็จแล้ว!
```

### 1.3 Proc ที่มีหลาย Parameters

```crystal
add = Proc(Int32, Int32, Int32).new { |a, b| a + b }
puts add.call(3, 4)  # => 7

multiply = Proc(Float64, Float64, Float64).new do |x, y|
  x * y
end
puts multiply.call(2.5, 4.0)  # => 10.0
```

---

## 2. Lambda - การสร้าง Lambda

### 2.1 Lambda Syntax

```crystal
# รูปแบบ -> syntax (shorthand)
greet = ->(name : String) { "Hello, #{name}!" }
puts greet.call("World")  # => Hello, World!

# Lambda หลายบรรทัด
calculate_tax = ->(amount : Float64, rate : Float64) {
  tax = amount * rate
  total = amount + tax
  {tax: tax, total: total}
}

result = calculate_tax.call(1000.0, 0.07)
puts "ภาษี: #{result[:tax]}"   # => ภาษี: 70.0
puts "รวม: #{result[:total]}"  # => รวม: 1070.0
```

### 2.2 Lambda ใน Array

```crystal
# เก็บ Lambda หลายตัวใน Array
operations = [
  ->(x : Int32) { x + 1 },
  ->(x : Int32) { x * 2 },
  ->(x : Int32) { x * x },
  ->(x : Int32) { x - 5 },
]

value = 3
operations.each_with_index do |op, i|
  puts "op[#{i}](#{value}) = #{op.call(value)}"
end
# => op[0](3) = 4
# => op[1](3) = 6
# => op[2](3) = 9
# => op[3](3) = -2
```

### 2.3 Lambda เป็น Return Value

```crystal
def make_adder(n : Int32)
  ->(x : Int32) { x + n }
end

add5 = make_adder(5)
add10 = make_adder(10)

puts add5.call(3)   # => 8
puts add10.call(3)  # => 13
puts add5.call(add10.call(1))  # => 16
```

---

## 3. วิธีการเรียกใช้: .call, .()

### 3.1 วิธีต่างๆ ในการเรียก Proc/Lambda

```crystal
double = ->(x : Int32) { x * 2 }

# วิธีที่ 1: .call() - ชัดเจนที่สุด
puts double.call(5)   # => 10

# วิธีที่ 2: .() - shorthand
puts double.(5)       # => 10

# วิธีที่ 3: [] - array-like syntax (ใน Crystal ใช้ .call เป็นหลัก)
# Crystal ไม่รองรับ [] syntax สำหรับ Proc โดยตรง
```

### 3.2 การเรียกด้วย Named Arguments

```crystal
greet = ->(first_name : String, last_name : String) {
  "#{first_name} #{last_name}"
}

# เรียกตามลำดับ
puts greet.call("สม", "ชาย")  # => สมชาย

# ใช้ .() syntax
puts greet.("สม", "ชาย")  # => สมชาย
```

### 3.3 Proc ใน Method Arguments

```crystal
def apply_twice(value : Int32, transform : Int32 -> Int32)
  transform.call(transform.call(value))
end

triple = ->(x : Int32) { x * 3 }
puts apply_twice(2, triple)  # => 18 (2 * 3 = 6, 6 * 3 = 18)

# ส่ง block แล้วแปลงเป็น Proc อัตโนมัติ
puts apply_twice(2) { |x| x * 3 }  # => 18
```

---

## 4. ความแตกต่างระหว่าง Proc และ Lambda

### 4.1 ความแตกต่างหลัก

```crystal
# ใน Crystal, -> สร้าง Proc (ไม่ใช่ Lambda แบบ Ruby)
# แต่มีพฤติกรรมคล้าย Lambda

# 1. Type checking - Crystal บังคับ type อยู่แล้ว
my_proc = Proc(Int32, String).new { |x| x.to_s }
puts my_proc.call(42)  # => "42"

# 2. arity - จำนวน parameters ต้องตรง
add = ->(a : Int32, b : Int32) { a + b }
puts add.call(1, 2)  # => 3
# add.call(1)  # Error: ต้องการ 2 arguments
```

### 4.2 Proc Arity

```crystal
# ตรวจสอบจำนวน parameters
no_args = -> { 42 }
one_arg = ->(x : Int32) { x }
two_args = ->(x : Int32, y : Int32) { x + y }

puts no_args.arity   # => 0
puts one_arg.arity   # => 1
puts two_args.arity  # => 2
```

### 4.3 Return Behavior

```crystal
# ใน Crystal, return ใน Proc ส่งคืนจาก Proc นั้นๆ
def test_proc
  my_proc = -> { return 10 }
  result = my_proc.call
  puts "หลัง proc: #{result}"  # จะพิมพ์
  result
end

puts test_proc  # => หลัง proc: 10 \n 10
```

---

## 5. Closure Behavior - พฤติกรรมการจับตัวแปร

### 5.1 Proc จับตัวแปรจาก Scope รอบข้าง

```crystal
# Closure - จับตัวแปรจากภายนอก
x = 10

add_x = ->(n : Int32) { n + x }
puts add_x.call(5)  # => 15

x = 20
puts add_x.call(5)  # => 25 (ใช้ค่า x ล่าสุด)
```

### 5.2 Counter ด้วย Closure

```crystal
def make_counter(start : Int32 = 0)
  count = start
  {
    increment: -> { count += 1; count },
    decrement: -> { count -= 1; count },
    reset: -> { count = start; nil },
    value: -> { count },
  }
end

counter = make_counter(0)

puts counter[:increment].call  # => 1
puts counter[:increment].call  # => 2
puts counter[:increment].call  # => 3
puts counter[:decrement].call  # => 2
puts counter[:value].call      # => 2
counter[:reset].call
puts counter[:value].call      # => 0
```

### 5.3 การแชร์ State ระหว่าง Procs

```crystal
def make_shared_state
  shared = [] of Int32

  adder = ->(x : Int32) { shared << x }
  remover = ->(x : Int32) { shared.delete(x) }
  getter = -> { shared.dup }

  {add: adder, remove: remover, get: getter}
end

state = make_shared_state

state[:add].call(1)
state[:add].call(2)
state[:add].call(3)
puts state[:get].call.inspect  # => [1, 2, 3]

state[:remove].call(2)
puts state[:get].call.inspect  # => [1, 3]
```

---

## 6. Method References - method(:name)

### 6.1 การอ้างอิงเมธอด

```crystal
def double(x : Int32) : Int32
  x * 2
end

# สร้าง method reference
ref = method(:double)
puts ref.call(5)  # => 10

# ใช้กับ map
result = [1, 2, 3, 4, 5].map { |x| double(x) }
puts result.inspect  # => [2, 4, 6, 8, 10]
```

### 6.2 Method Reference กับ Instance Methods

```crystal
class StringProcessor
  def initialize(@prefix : String)
  end

  def process(s : String) : String
    "#{@prefix}: #{s.upcase}"
  end
end

processor = StringProcessor.new("INFO")
ref = processor.method(:process)

puts ref.call("hello")   # => INFO: HELLO
puts ref.call("world")   # => INFO: WORLD

# ใช้กับ map
results = ["apple", "banana"].map { |s| processor.process(s) }
puts results.inspect  # => ["INFO: APPLE", "INFO: BANANA"]
```

### 6.3 Built-in Method References

```crystal
# อ้างอิง built-in methods
to_s_ref = 42.method(:to_s)
puts to_s_ref.call  # => "42"

# กับ Array methods
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
sorted = numbers.sort
puts sorted.inspect  # => [1, 1, 2, 3, 4, 5, 6, 9]
```

---

## 7. Proc Composition

### 7.1 การต่อ Proc เข้าด้วยกัน

```crystal
# สร้าง Proc composition function
def compose(f : Int32 -> Int32, g : Int32 -> Int32)
  ->(x : Int32) { f.call(g.call(x)) }
end

double = ->(x : Int32) { x * 2 }
add_one = ->(x : Int32) { x + 1 }

# double หลังจาก add_one
double_after_add = compose(double, add_one)
puts double_after_add.call(3)  # => 8 ((3+1)*2)

# add_one หลังจาก double
add_after_double = compose(add_one, double)
puts add_after_double.call(3)  # => 7 ((3*2)+1)
```

### 7.2 Pipeline ด้วย Array ของ Procs

```crystal
def pipeline(*procs : Int32 -> Int32)
  ->(x : Int32) {
    procs.reduce(x) { |val, proc| proc.call(val) }
  }
end

process = pipeline(
  ->(x : Int32) { x * 2 },      # คูณสอง
  ->(x : Int32) { x + 10 },     # บวกสิบ
  ->(x : Int32) { x * x },      # ยกกำลังสอง
)

puts process.call(3)  # => ((3*2)+10)^2 = 16^2 = 256
```

---

## 8. ตัวอย่างการใช้งานขั้นสูง

### 8.1 Currying

```crystal
# Curry function - แปลงฟังก์ชัน n-ary เป็น unary
def curry(f : Int32, Int32 -> Int32, x : Int32)
  ->(y : Int32) { f.call(x, y) }
end

add = ->(a : Int32, b : Int32) { a + b }
add5 = curry(add, 5)
puts add5.call(3)   # => 8
puts add5.call(10)  # => 15

multiply = ->(a : Int32, b : Int32) { a * b }
times3 = curry(multiply, 3)
puts times3.call(4)   # => 12
puts times3.call(7)   # => 21
```

### 8.2 Memoization

```crystal
def memoize(f : Int32 -> Int32)
  cache = {} of Int32 => Int32
  ->(x : Int32) {
    cache[x] ||= f.call(x)
  }
end

expensive_calc = ->(n : Int32) {
  puts "คำนวณ #{n}..."
  n * n + n * 2 + 1
}

memoized = memoize(expensive_calc)
puts memoized.call(5)   # คำนวณ 5... => 36
puts memoized.call(5)   # ไม่คำนวณซ้ำ => 36
puts memoized.call(10)  # คำนวณ 10... => 121
puts memoized.call(10)  # ไม่คำนวณซ้ำ => 121
```

### 8.3 Strategy Pattern

```crystal
class Sorter
  def initialize(@strategy : Array(Int32) -> Array(Int32))
  end

  def sort(data : Array(Int32))
    @strategy.call(data)
  end
end

# กลยุทธ์ต่างๆ
ascending = ->(arr : Array(Int32)) { arr.sort }
descending = ->(arr : Array(Int32)) { arr.sort.reverse }
by_absolute = ->(arr : Array(Int32)) { arr.sort_by(&.abs) }

data = [-3, 1, -7, 4, -2, 8]

puts Sorter.new(ascending).sort(data).inspect   # => [-7, -3, -2, 1, 4, 8]
puts Sorter.new(descending).sort(data).inspect  # => [8, 4, 1, -2, -3, -7]
puts Sorter.new(by_absolute).sort(data).inspect # => [1, -2, -3, 4, -7, 8]
```

### 8.4 Functional Programming Utilities

```crystal
# map, filter, reduce ด้วย Proc

def my_map(arr : Array(Int32), &f : Int32 -> Int32)
  result = [] of Int32
  arr.each { |x| result << f.call(x) }
  result
end

def my_filter(arr : Array(Int32), &pred : Int32 -> Bool)
  result = [] of Int32
  arr.each { |x| result << x if pred.call(x) }
  result
end

def my_reduce(arr : Array(Int32), initial : Int32, &f : Int32, Int32 -> Int32)
  acc = initial
  arr.each { |x| acc = f.call(acc, x) }
  acc
end

numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

doubled = my_map(numbers) { |x| x * 2 }
evens = my_filter(numbers) { |x| x.even? }
sum = my_reduce(numbers, 0) { |acc, x| acc + x }

puts doubled.inspect  # => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
puts evens.inspect    # => [2, 4, 6, 8, 10]
puts sum              # => 55
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function Factory

```crystal
def make_validator(rules : Array(String -> Bool))
  ->(value : String) {
    rules.all? { |rule| rule.call(value) }
  }
end

not_empty = ->(s : String) { !s.empty? }
min_length = ->(min : Int32) { ->(s : String) { s.size >= min } }
has_uppercase = ->(s : String) { s.chars.any?(&.uppercase?) }

password_validator = make_validator([
  not_empty,
  min_length.call(8),
  has_uppercase,
])

puts password_validator.call("abc")       # => false (too short)
puts password_validator.call("abcdefgh")  # => false (no uppercase)
puts password_validator.call("Abcdefgh")  # => true
```

### แบบฝึกหัดที่ 2: Event System ด้วย Lambda

```crystal
class EventBus
  def initialize
    @handlers = {} of Symbol => Array(Proc(Hash(Symbol, String), Nil))
  end

  def subscribe(event : Symbol, handler : Hash(Symbol, String) ->)
    @handlers[event] ||= [] of Proc(Hash(Symbol, String), Nil)
    @handlers[event] << handler
  end

  def publish(event : Symbol, payload : Hash(Symbol, String))
    @handlers[event]?.try do |handlers|
      handlers.each { |h| h.call(payload) }
    end
  end
end

bus = EventBus.new

logger = ->(data : Hash(Symbol, String)) {
  puts "[LOG] #{data[:event_type]}: #{data[:message]}"
}

notifier = ->(data : Hash(Symbol, String)) {
  puts "[NOTIFY] แจ้งเตือน: #{data[:message]}"
}

bus.subscribe(:user_action, logger)
bus.subscribe(:user_action, notifier)

bus.publish(:user_action, {event_type: "login", message: "สมชาย เข้าสู่ระบบ"})
```

### แบบฝึกหัดที่ 3: Transformation Pipeline

```crystal
class DataPipeline
  def initialize
    @transforms = [] of String -> String
  end

  def add_step(transform : String -> String)
    @transforms << transform
    self
  end

  def process(input : String)
    @transforms.reduce(input) { |data, t| t.call(data) }
  end
end

pipeline = DataPipeline.new
  .add_step(->(s : String) { s.strip })
  .add_step(->(s : String) { s.downcase })
  .add_step(->(s : String) { s.gsub(/\s+/, "_") })
  .add_step(->(s : String) { s.gsub(/[^a-z0-9_]/, "") })

puts pipeline.process("  Hello, World! 123  ")
# => hello_world_123
```

---

## สรุป

Procs และ Lambdas ใน Crystal ช่วยให้เขียนโค้ดแบบ functional ได้อย่างมีประสิทธิภาพ:

| Feature | Syntax | หมายเหตุ |
|---------|--------|---------|
| Proc.new | `Proc(T, R).new { \|x\| ... }` | สร้าง Proc พร้อม type |
| Lambda | `->(x : T) { ... }` | Shorthand syntax |
| Call | `.call(args)` หรือ `.(args)` | เรียกใช้งาน |
| Method ref | `method(:name)` | อ้างอิงเมธอด |
| Arity | `.arity` | ดูจำนวน params |
| Closure | จับ local vars | อัตโนมัติ |

**ความแตกต่างหลักจาก Ruby:**
- Crystal บังคับ type ทำให้ type safe กว่า
- ไม่มี `lambda?` method เพราะทุกอย่างเป็น Proc
- `return` ใน Proc ส่งคืนจาก Proc ไม่ใช่จาก enclosing method
