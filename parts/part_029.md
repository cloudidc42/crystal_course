# Part 29: Closures

## บทนำ

Closure คือฟังก์ชันที่ "จำ" สภาพแวดล้อม (environment) ที่ถูกสร้างขึ้น โดยสามารถเข้าถึงตัวแปรจาก scope ที่ครอบอยู่ได้แม้ว่า scope นั้นจะสิ้นสุดแล้วก็ตาม ใน Crystal, blocks, procs, และ lambdas ล้วนเป็น closures

---

## 1. Closure คืออะไร

### 1.1 คำนิยามพื้นฐาน

```crystal
# ตัวอย่างง่ายๆ ของ closure
def make_greeting(prefix : String)
  # prefix ถูก "จับ" โดย lambda นี้
  ->(name : String) { "#{prefix}, #{name}!" }
end

hello = make_greeting("สวัสดี")
hi = make_greeting("หวัดดี")

puts hello.call("สมชาย")  # => สวัสดี, สมชาย!
puts hi.call("สมหญิง")    # => หวัดดี, สมหญิง!

# prefix ยังคงอยู่แม้ make_greeting จะ return แล้ว
puts hello.call("สมศักดิ์")  # => สวัสดี, สมศักดิ์!
```

### 1.2 ทำไม Closures ถึงสำคัญ

```crystal
# โดยไม่มี closure เราต้องส่ง context ทุกครั้ง
def greet_no_closure(name : String, prefix : String)
  "#{prefix}, #{name}!"
end

# ด้วย closure เราแยก context ออกจาก call
hello = make_greeting("สวัสดี")
hello.call("ทุกคน")  # ไม่ต้องส่ง prefix ซ้ำ
```

### 1.3 Closure vs Regular Function

```crystal
# Regular function - ไม่มี captured state
def add_five(x : Int32)
  x + 5
end

# Closure - มี captured state
offset = 5
add_offset = ->(x : Int32) { x + offset }

puts add_five(10)       # => 15
puts add_offset.call(10)  # => 15

offset = 100
puts add_five(10)       # => 15 (ไม่เปลี่ยน)
puts add_offset.call(10)  # => 110 (เปลี่ยนตาม offset)
```

---

## 2. Capturing Variables - การจับตัวแปร

### 2.1 การจับตัวแปร Mutable

```crystal
# Closure จับ reference ไม่ใช่ค่า
count = 0

increment = -> { count += 1 }
get_count = -> { count }

increment.call
increment.call
increment.call
puts get_count.call  # => 3
puts count           # => 3 (ตัวแปรเดิมถูกเปลี่ยน)
```

### 2.2 Closure กับ Loop

```crystal
# ระวัง! closure ใน loop จับตัวแปรเดียวกัน
closures = [] of -> Int32

# ปัญหา: ทุก closure ใช้ตัวแปร i เดียวกัน
# (ใน Crystal, loop variable เป็น immutable ต่อ iteration)
[1, 2, 3].each do |i|
  closures << -> { i }  # แต่ละ closure จับ i ของ iteration นั้น
end

closures.each { |c| puts c.call }
# => 1
# => 2
# => 3
```

### 2.3 จับตัวแปรหลายตัว

```crystal
def make_range_checker(min : Int32, max : Int32)
  ->(value : Int32) { value >= min && value <= max }
end

in_teens = make_range_checker(13, 19)
in_twenties = make_range_checker(20, 29)
child = make_range_checker(0, 12)

ages = [5, 14, 22, 8, 17, 25]
ages.each do |age|
  category = if child.call(age)
    "เด็ก"
  elsif in_teens.call(age)
    "วัยรุ่น"
  elsif in_twenties.call(age)
    "วัยยี่สิบ"
  else
    "อื่นๆ"
  end
  puts "#{age} ปี: #{category}"
end
```

---

## 3. Closures ใน Procs และ Lambdas

### 3.1 Proc Closure

```crystal
def make_multiplier(factor : Int32)
  Proc(Int32, Int32).new { |x| x * factor }
end

double = make_multiplier(2)
triple = make_multiplier(3)
quadruple = make_multiplier(4)

[1, 2, 3, 4, 5].each do |n|
  puts "#{n} * 2 = #{double.call(n)}, " \
       "* 3 = #{triple.call(n)}, " \
       "* 4 = #{quadruple.call(n)}"
end
```

### 3.2 Lambda Closure ที่ Mutable State

```crystal
def make_stack
  storage = [] of Int32

  push = ->(x : Int32) { storage << x; nil }
  pop = -> { storage.pop? }
  peek = -> { storage.last? }
  size = -> { storage.size }
  empty = -> { storage.empty? }

  {push: push, pop: pop, peek: peek, size: size, empty: empty}
end

stack = make_stack

stack[:push].call(1)
stack[:push].call(2)
stack[:push].call(3)

puts stack[:size].call   # => 3
puts stack[:peek].call.inspect  # => 3
puts stack[:pop].call.inspect   # => 3
puts stack[:size].call   # => 2
```

### 3.3 Closure กับ Block

```crystal
def create_counter(initial : Int32 = 0)
  count = initial

  {
    increment: ->(step : Int32) { count += step },
    decrement: ->(step : Int32) { count -= step },
    reset: -> { count = initial },
    value: -> { count },
  }
end

c = create_counter(10)
puts c[:value].call   # => 10
c[:increment].call(5)
c[:increment].call(3)
puts c[:value].call   # => 18
c[:decrement].call(8)
puts c[:value].call   # => 10
c[:reset].call
puts c[:value].call   # => 10
```

---

## 4. Memoization ด้วย Closures

### 4.1 Memoization พื้นฐาน

```crystal
def memoize(&computation : Int32 -> Int32)
  cache = {} of Int32 => Int32
  
  ->(n : Int32) {
    unless cache.has_key?(n)
      puts "คำนวณ #{n}..."
      cache[n] = computation.call(n)
    end
    cache[n]
  }
end

# คำนวณ fibonacci แบบ naive (ช้ามาก)
fib_raw = ->(n : Int32) {
  n <= 1 ? n : n * (n - 1) / 2  # simplified
}

fib_memo = memoize { |n| n <= 1 ? n : n * (n - 1) / 2 }

puts fib_memo.call(5)   # คำนวณ 5... => 10
puts fib_memo.call(5)   # ไม่คำนวณ => 10 (cached)
puts fib_memo.call(10)  # คำนวณ 10... => 45
```

### 4.2 Recursive Memoization

```crystal
# Fibonacci ที่แท้จริงด้วย memoization
def make_fib
  cache = {} of Int32 => Int64
  
  # ต้องใช้ self-referential closure
  fib = uninitialized Int32 -> Int64
  fib = ->(n : Int32) {
    return n.to_i64 if n <= 1
    cache[n] ||= fib.call(n - 1) + fib.call(n - 2)
  }
  
  fib
end

fibonacci = make_fib
(0..15).each { |n| puts "fib(#{n}) = #{fibonacci.call(n)}" }
```

### 4.3 Generic Memoizer Class

```crystal
class Memo
  def initialize
    @cache = {} of String => String
  end

  def get_or_compute(key : String, &block : -> String) : String
    @cache[key] ||= block.call
  end

  def invalidate(key : String)
    @cache.delete(key)
  end

  def clear
    @cache.clear
  end
end

memo = Memo.new

result1 = memo.get_or_compute("expensive_op") do
  puts "กำลังคำนวณ..."
  "ผลลัพธ์ที่มีค่า"
end

result2 = memo.get_or_compute("expensive_op") do
  puts "จะไม่ถูกเรียก"
  "ค่าอื่น"
end

puts result1  # => ผลลัพธ์ที่มีค่า
puts result2  # => ผลลัพธ์ที่มีค่า (cached)
```

---

## 5. Factory Functions ด้วย Closures

### 5.1 Simple Factory

```crystal
def make_greeting_factory(language : String)
  case language
  when "thai"
    ->(name : String) { "สวัสดี, #{name}!" }
  when "english"
    ->(name : String) { "Hello, #{name}!" }
  when "japanese"
    ->(name : String) { "こんにちは, #{name}！" }
  else
    ->(name : String) { "Hi, #{name}!" }
  end
end

thai_greet = make_greeting_factory("thai")
english_greet = make_greeting_factory("english")
japanese_greet = make_greeting_factory("japanese")

["สมชาย", "Somchai", "田中"].each_with_index do |name, i|
  procs = [thai_greet, english_greet, japanese_greet]
  puts procs[i].call(name)
end
```

### 5.2 Configurable Formatter

```crystal
def make_number_formatter(
  prefix : String = "",
  suffix : String = "",
  decimal_places : Int32 = 2,
  thousands_sep : String = ","
)
  ->(num : Float64) {
    # จัดรูปแบบตัวเลข
    formatted = ("%.#{decimal_places}f" % num)
    # เพิ่ม thousands separator (simplified)
    "#{prefix}#{formatted}#{suffix}"
  }
end

baht = make_number_formatter(prefix: "฿", decimal_places: 2)
usd = make_number_formatter(prefix: "$", decimal_places: 2)
percent = make_number_formatter(suffix: "%", decimal_places: 1)

puts baht.call(1234567.89)    # => ฿1234567.89
puts usd.call(9999.99)        # => $9999.99
puts percent.call(0.756 * 100)  # => 75.6%
```

### 5.3 Validator Factory

```crystal
def make_validator(
  min_length : Int32? = nil,
  max_length : Int32? = nil,
  pattern : Regex? = nil,
  required : Bool = false
)
  ->(value : String) {
    errors = [] of String
    
    errors << "ต้องระบุค่า" if required && value.empty?
    
    if min_len = min_length
      errors << "ต้องมีความยาวอย่างน้อย #{min_len} ตัวอักษร" if value.size < min_len
    end
    
    if max_len = max_length
      errors << "ต้องมีความยาวไม่เกิน #{max_len} ตัวอักษร" if value.size > max_len
    end
    
    if pat = pattern
      errors << "รูปแบบไม่ถูกต้อง" unless value.match(pat)
    end
    
    errors
  }
end

email_validator = make_validator(
  required: true,
  pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/
)

password_validator = make_validator(
  required: true,
  min_length: 8,
  max_length: 50,
  pattern: /[A-Z]/  # ต้องมีตัวใหญ่
)

puts email_validator.call("").inspect               # error
puts email_validator.call("invalid").inspect        # error
puts email_validator.call("user@example.com").inspect  # []

puts password_validator.call("abc").inspect         # error (too short)
puts password_validator.call("abcdefgh").inspect    # error (no uppercase)
puts password_validator.call("Abcdefgh").inspect    # []
```

---

## 6. Practical Closure Patterns

### 6.1 Once Pattern - ทำงานครั้งเดียว

```crystal
def once(&block : ->)
  executed = false
  result_storage = nil
  
  -> {
    unless executed
      block.call
      executed = true
    end
  }
end

initialize_db = once do
  puts "กำลัง initialize database..."
  # โค้ด initialization จริงๆ
end

initialize_db.call  # => กำลัง initialize database...
initialize_db.call  # ไม่ทำซ้ำ
initialize_db.call  # ไม่ทำซ้ำ
```

### 6.2 Throttle Pattern

```crystal
def throttle(interval_ms : Int32, &block : ->)
  last_called = Time::Span::ZERO
  
  -> {
    now = Time.monotonic
    elapsed = (now - Time.monotonic).total_milliseconds.abs
    
    if last_called == Time::Span::ZERO || elapsed >= interval_ms
      last_called = now
      block.call
    end
  }
end
```

### 6.3 Partial Application

```crystal
def partial(f : Int32, Int32 -> Int32, first_arg : Int32)
  ->(second_arg : Int32) { f.call(first_arg, second_arg) }
end

add = ->(a : Int32, b : Int32) { a + b }
add10 = partial(add, 10)

puts add10.call(5)   # => 15
puts add10.call(20)  # => 30

multiply = ->(a : Int32, b : Int32) { a * b }
triple = partial(multiply, 3)
puts triple.call(7)  # => 21
```

### 6.4 Builder Pattern ด้วย Closure

```crystal
def build_sql
  parts = {
    table: "",
    conditions: [] of String,
    order: nil.as(String?),
    limit: nil.as(Int32?),
  }

  builder = {
    from: ->(table : String) { parts = parts.merge(table: table) },
    where: ->(cond : String) {
      conditions = parts[:conditions] + [cond]
      parts = parts.merge(conditions: conditions)
    },
    order_by: ->(col : String) { parts = parts.merge(order: col) },
    limit: ->(n : Int32) { parts = parts.merge(limit: n) },
    build: -> {
      sql = "SELECT * FROM #{parts[:table]}"
      unless parts[:conditions].empty?
        sql += " WHERE " + parts[:conditions].join(" AND ")
      end
      if order = parts[:order]
        sql += " ORDER BY #{order}"
      end
      if lim = parts[:limit]
        sql += " LIMIT #{lim}"
      end
      sql
    },
  }

  builder
end

sql = build_sql
sql[:from].call("users")
sql[:where].call("age > 18")
sql[:where].call("active = true")
sql[:order_by].call("created_at DESC")
sql[:limit].call(10)

puts sql[:build].call
# => SELECT * FROM users WHERE age > 18 AND active = true ORDER BY created_at DESC LIMIT 10
```

---

## 7. Closure กับ Concurrency

### 7.1 Thread-Safe Counter

```crystal
require "mutex"

def make_thread_safe_counter
  count = 0
  mutex = Mutex.new

  {
    increment: -> { mutex.synchronize { count += 1 } },
    value: -> { mutex.synchronize { count } },
  }
end

counter = make_thread_safe_counter

# จำลองการ increment จาก หลาย fiber
10.times do
  spawn { counter[:increment].call }
end

Fiber.yield  # ให้ fibers ทำงาน
sleep(0.01)
puts counter[:value].call  # => ประมาณ 10
```

### 7.2 Lazy Evaluation

```crystal
def lazy(&computation : -> Int32)
  computed = false
  value = 0
  
  -> {
    unless computed
      puts "กำลังคำนวณ..."
      value = computation.call
      computed = true
    end
    value
  }
end

expensive = lazy do
  # จำลองการคำนวณที่ใช้เวลา
  (1..1000).sum
end

puts "ยังไม่ได้คำนวณ"
puts expensive.call  # กำลังคำนวณ... => 500500
puts expensive.call  # ไม่คำนวณซ้ำ => 500500
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function Composition

```crystal
# สร้าง compose function ที่รับ Proc หลายตัว
def compose(*fns : Int32 -> Int32)
  ->(x : Int32) {
    fns.reverse.reduce(x) { |val, fn| fn.call(val) }
  }
end

add1 = ->(x : Int32) { x + 1 }
mul2 = ->(x : Int32) { x * 2 }
sub3 = ->(x : Int32) { x - 3 }

# compose(f, g, h)(x) = f(g(h(x)))
transform = compose(add1, mul2, sub3)
puts transform.call(10)  # (10-3)*2+1 = 15
```

### แบบฝึกหัดที่ 2: Accumulator

```crystal
def make_accumulator(initial : Int32 = 0)
  total = initial
  history = [] of Int32

  {
    add: ->(n : Int32) {
      history << total
      total += n
      total
    },
    subtract: ->(n : Int32) {
      history << total
      total -= n
      total
    },
    undo: -> {
      if prev = history.pop?
        total = prev
      end
      total
    },
    current: -> { total },
    history: -> { history.dup },
  }
end

acc = make_accumulator(100)
puts acc[:add].call(50)      # => 150
puts acc[:add].call(25)      # => 175
puts acc[:subtract].call(30) # => 145
puts acc[:undo].call         # => 175
puts acc[:current].call      # => 175
```

### แบบฝึกหัดที่ 3: Cache with TTL

```crystal
def make_ttl_cache(ttl_seconds : Int32)
  cache = {} of String => {String, Time}
  
  {
    set: ->(key : String, value : String) {
      cache[key] = {value, Time.local}
      nil
    },
    get: ->(key : String) {
      if entry = cache[key]?
        value, timestamp = entry
        if (Time.local - timestamp).total_seconds < ttl_seconds
          value.as(String?)
        else
          cache.delete(key)
          nil.as(String?)
        end
      else
        nil.as(String?)
      end
    },
    delete: ->(key : String) { cache.delete(key); nil },
    size: -> { cache.size },
  }
end

cache = make_ttl_cache(60)  # TTL 60 วินาที
cache[:set].call("user:1", "สมชาย")
cache[:set].call("user:2", "สมหญิง")

puts cache[:get].call("user:1").inspect  # => "สมชาย"
puts cache[:get].call("user:99").inspect # => nil
puts cache[:size].call                   # => 2
```

---

## สรุป

Closures เป็นแนวคิดพื้นฐานที่สำคัญมากใน Crystal:

| แนวคิด | คำอธิบาย |
|--------|---------|
| Variable capture | Closure จับ reference ของตัวแปร ไม่ใช่ค่า |
| Lexical scope | Closure เห็นตัวแปรจาก scope ที่ถูกสร้าง |
| State encapsulation | ซ่อน state ไว้ใน closure แทน global/class variable |
| Factory function | ใช้ closure สร้างฟังก์ชันที่มี config ต่างกัน |
| Memoization | Cache ผลลัพธ์ด้วย closure เป็น private cache |

**Best Practices:**
- ใช้ closure เพื่อ encapsulate state แทนการใช้ global variables
- Memoization ด้วย closure ทำให้โค้ดสะอาดกว่าการใช้ instance variable
- ระวัง memory leak เมื่อ closure จับ objects ขนาดใหญ่
- Factory functions ด้วย closure ทำให้โค้ด reusable และ configurable
