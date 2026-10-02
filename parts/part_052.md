# Part 52: Iterators ใน Crystal

## บทนำ

Iterator ใน Crystal เป็นกลไกที่ช่วย traverse collection อย่าง lazy ไม่คำนวณค่าทั้งหมดทันที แต่คำนวณเมื่อต้องการ เหมาะสำหรับ infinite sequences และการประมวลผล large datasets

---

## 52.1 Iterator Module

```crystal
# Crystal มี Iterator module ที่ต้องการ implement next
# Iterator ส่งค่าออกมาทีละตัวหรือคืน Iterator::Stop เมื่อสิ้นสุด

class CountUp
  include Iterator(Int32)
  
  def initialize(@current : Int32 = 0, @max : Int32 = Int32::MAX)
  end
  
  def next : Int32 | Stop
    if @current > @max
      stop
    else
      value = @current
      @current += 1
      value
    end
  end
end

# ใช้งาน
counter = CountUp.new(1, 5)
puts counter.next  # => 1
puts counter.next  # => 2
puts counter.next  # => 3

# วนลูปจนจบ
counter2 = CountUp.new(1, 5)
counter2.each { |n| print "#{n} " }
puts
# => 1 2 3 4 5
```

---

## 52.2 next และ stop

```crystal
class FibonacciIterator
  include Iterator(Int64)
  
  def initialize
    @a = 0_i64
    @b = 1_i64
    @count = 0
  end
  
  def initialize(@limit : Int32)
    @a = 0_i64
    @b = 1_i64
    @count = 0
  end
  
  def next : Int64 | Stop
    return stop if @count >= @limit
    
    value = @a
    @a, @b = @b, @a + @b
    @count += 1
    value
  end
  
  def reset
    @a = 0_i64
    @b = 1_i64
    @count = 0
  end
end

fib = FibonacciIterator.new(10)
puts fib.to_a.inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# ใช้ each
fib2 = FibonacciIterator.new(8)
fib2.each { |n| print "#{n} " }
puts
# => 0 1 1 2 3 5 8 13
```

---

## 52.3 chain

chain เชื่อม iterators หลายตัวเข้าด้วยกัน

```crystal
class RangeIterator
  include Iterator(Int32)
  
  def initialize(@start : Int32, @stop : Int32, @step : Int32 = 1)
    @current = @start
  end
  
  def next : Int32 | Stop
    return stop if (@step > 0 && @current > @stop) || (@step < 0 && @current < @stop)
    value = @current
    @current += @step
    value
  end
end

iter1 = RangeIterator.new(1, 3)
iter2 = RangeIterator.new(7, 9)
iter3 = RangeIterator.new(13, 15)

# chain: ต่อกัน
chained = iter1.chain(iter2).chain(iter3)
puts chained.to_a.inspect  # => [1, 2, 3, 7, 8, 9, 13, 14, 15]
```

---

## 52.4 cycle

cycle วนซ้ำ iterator ไปเรื่อยๆ

```crystal
class SeasonIterator
  include Iterator(String)
  
  SEASONS = ["Spring", "Summer", "Fall", "Winter"]
  
  def initialize
    @index = 0
  end
  
  def next : String | Stop
    value = SEASONS[@index % 4]
    @index += 1
    value
  end
end

# cycle สร้าง infinite sequence
seasons = SeasonIterator.new
# เอาแค่ 8 ตัวแรก (2 รอบ)
puts seasons.first(8).inspect
# => ["Spring", "Summer", "Fall", "Winter", "Spring", "Summer", "Fall", "Winter"]
```

### cycle บน Array

```crystal
# Array สร้าง cyclic iterator
colors = ["red", "green", "blue"]
cycle_iter = colors.cycle

# เอา 7 สี
result = [] of String
7.times { result << cycle_iter.next.as(String) }
puts result.inspect
# => ["red", "green", "blue", "red", "green", "blue", "red"]
```

---

## 52.5 each

Iterator มี each จาก Iterator module

```crystal
class PrimeIterator
  include Iterator(Int32)
  
  def initialize(@limit : Int32)
    @current = 2
  end
  
  def next : Int32 | Stop
    while @current <= @limit
      if prime?(@current)
        value = @current
        @current += 1
        return value
      end
      @current += 1
    end
    stop
  end
  
  private def prime?(n : Int32) : Bool
    return false if n < 2
    (2..Math.sqrt(n).to_i).each { |i| return false if n % i == 0 }
    true
  end
end

primes = PrimeIterator.new(50)
puts primes.to_a.inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

puts "Sum of primes up to 50: #{PrimeIterator.new(50).sum}"
# => 328

puts "First 5 primes: #{PrimeIterator.new(100).first(5).inspect}"
# => [2, 3, 5, 7, 11]
```

---

## 52.6 zip

zip รวม iterators คู่กัน

```crystal
iter_a = [1, 2, 3].each
iter_b = ["a", "b", "c"].each

zipped = iter_a.zip(iter_b)
puts zipped.to_a.inspect
# => [{1, "a"}, {2, "b"}, {3, "c"}]

# zip กับ multiple iterators
a = [1, 2, 3].each
b = ["x", "y", "z"].each
c = [true, false, true].each

result = a.zip(b, c)
result.each do |num, letter, bool|
  puts "#{num}, #{letter}, #{bool}"
end
# 1, x, true
# 2, y, false
# 3, z, true
```

---

## 52.7 map บน Iterators

```crystal
# Lazy map - ไม่คำนวณทันที
numbers = (1..Float64::INFINITY.to_i).each  # ไม่ได้ใช้จริง

# ใช้ Array range แทน
lazy_squares = (1..100).each.map { |n| n * n }
puts lazy_squares.first(5).inspect  # => [1, 4, 9, 16, 25]

# Custom iterator + map
class EveryNth
  include Iterator(Int32)
  
  def initialize(@n : Int32, @max : Int32)
    @current = 0
  end
  
  def next : Int32 | Stop
    @current += @n
    return stop if @current > @max
    @current
  end
end

# Multiples of 3 up to 30
thirds = EveryNth.new(3, 30)
puts thirds.to_a.inspect  # => [3, 6, 9, 12, 15, 18, 21, 24, 27, 30]

# map บน iterator
squares_of_thirds = thirds.map { |n| n * n }
thirds2 = EveryNth.new(3, 30)
puts thirds2.map { |n| n * n }.to_a.inspect
# => [9, 36, 81, 144, 225, 324, 441, 576, 729, 900]
```

---

## 52.8 Lazy Evaluation

Iterator ทำ lazy evaluation - คำนวณเมื่อต้องการเท่านั้น

```crystal
# Infinite sequence ด้วย Iterator
class NaturalNumbers
  include Iterator(Int32)
  
  def initialize
    @current = 1
  end
  
  def next : Int32 | Stop
    value = @current
    @current += 1
    value  # ไม่มี stop - infinite!
  end
end

naturals = NaturalNumbers.new

# เอาแค่ที่ต้องการ
puts naturals.first(10).inspect
# => [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# หา 5 ตัวแรกที่หาร 7 ลงตัว
naturals2 = NaturalNumbers.new
multiples_of_7 = naturals2.select { |n| n % 7 == 0 }.first(5)
puts multiples_of_7.inspect  # => [7, 14, 21, 28, 35]

# หา fibonacci ตัวแรกที่มากกว่า 1000
class InfiniteFib
  include Iterator(Int64)
  
  def initialize
    @a = 0_i64
    @b = 1_i64
  end
  
  def next : Int64 | Stop
    value = @a
    @a, @b = @b, @a + @b
    value
  end
end

fib = InfiniteFib.new
first_over_1000 = fib.find { |n| n > 1000 }
puts first_over_1000  # => 1597
```

---

## 52.9 Custom Iterators

### Tree Iterator

```crystal
class TreeNode(T)
  property value : T
  property left : TreeNode(T)?
  property right : TreeNode(T)?
  
  def initialize(@value : T)
  end
end

class InOrderIterator(T)
  include Iterator(T)
  
  def initialize(root : TreeNode(T)?)
    @stack = Deque(TreeNode(T)).new
    push_left(root)
  end
  
  def next : T | Stop
    return stop if @stack.empty?
    
    node = @stack.pop_back
    push_left(node.right)
    node.value
  end
  
  private def push_left(node : TreeNode(T)?)
    while node
      @stack.push_back(node)
      node = node.left
    end
  end
end

# สร้าง tree
root = TreeNode(Int32).new(5)
root.left = TreeNode(Int32).new(3)
root.right = TreeNode(Int32).new(7)
root.left.not_nil!.left = TreeNode(Int32).new(1)
root.left.not_nil!.right = TreeNode(Int32).new(4)
root.right.not_nil!.left = TreeNode(Int32).new(6)
root.right.not_nil!.right = TreeNode(Int32).new(9)

iter = InOrderIterator(Int32).new(root)
puts iter.to_a.inspect  # => [1, 3, 4, 5, 6, 7, 9]
```

### File Line Iterator

```crystal
class LineIterator
  include Iterator(String)
  
  def initialize(lines : Array(String))
    @lines = lines
    @index = 0
  end
  
  def next : String | Stop
    return stop if @index >= @lines.size
    line = @lines[@index]
    @index += 1
    line
  end
  
  def reset
    @index = 0
    self
  end
end

# จำลองการอ่านไฟล์
lines = LineIterator.new([
  "# Comment",
  "name=Alice",
  "",
  "# Another comment",
  "age=30",
  "city=Bangkok"
])

# Skip comments และ empty lines
config = lines
  .reject { |line| line.starts_with?("#") || line.empty? }
  .map { |line|
    key, value = line.split("=", 2)
    {key.strip, value.strip}
  }
  .to_h

puts config.inspect
# => {"name" => "Alice", "age" => "30", "city" => "Bangkok"}
```

---

## 52.10 Iterator Transformations

```crystal
class TransformIterator(A, B)
  include Iterator(B)
  
  def initialize(@source : Iterator(A), @transform : A -> B)
  end
  
  def next : B | Stop
    value = @source.next
    return stop if value.is_a?(Iterator::Stop)
    @transform.call(value.as(A))
  end
end

class FilterIterator(T)
  include Iterator(T)
  
  def initialize(@source : Iterator(T), @predicate : T -> Bool)
  end
  
  def next : T | Stop
    loop do
      value = @source.next
      return stop if value.is_a?(Iterator::Stop)
      val = value.as(T)
      return val if @predicate.call(val)
    end
  end
end

class TakeIterator(T)
  include Iterator(T)
  
  def initialize(@source : Iterator(T), @n : Int32)
    @count = 0
  end
  
  def next : T | Stop
    return stop if @count >= @n
    value = @source.next
    return stop if value.is_a?(Iterator::Stop)
    @count += 1
    value.as(T)
  end
end

# ใช้งาน
source = (1..100).each
evens = FilterIterator(Int32).new(source, ->(n : Int32) { n.even? })
first_5_evens = TakeIterator(Int32).new(evens, 5)
puts first_5_evens.to_a.inspect  # => [2, 4, 6, 8, 10]
```

---

## 52.11 ตัวอย่างในชีวิตจริง: Data Pipeline

```crystal
class DataSource(T)
  include Iterator(T)
  
  def initialize(@data : Array(T))
    @index = 0
  end
  
  def next : T | Stop
    return stop if @index >= @data.size
    value = @data[@index]
    @index += 1
    value
  end
end

class Pipeline(T, U)
  def initialize(@source : Iterator(T))
    @transforms = [] of Iterator(T) -> Iterator(T)
  end
  
  def map(& : T -> T) : self
    # Simplified - just collect
    self
  end
  
  def execute : Array(T)
    @source.to_a
  end
end

# Simple pipeline
data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
source = DataSource(Int32).new(data)

result = source
  .select { |n| n.odd? }
  .map { |n| n * n }
  .select { |n| n > 10 }
  .first(3)

puts result.inspect  # => [25, 49, 81]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Range Iterator

```crystal
class StepRange
  include Iterator(Float64)
  
  def initialize(@start : Float64, @stop : Float64, @step : Float64)
    @current = @start
  end
  
  def next : Float64 | Stop
    return stop if @current > @stop
    value = @current
    @current += @step
    value
  end
end

# ค่า sin ทุก 0.1 ตั้งแต่ 0 ถึง PI
sin_vals = StepRange.new(0.0, Math::PI, 0.1)
               .map { |x| {x.round(2), Math.sin(x).round(4)} }
               .first(5)
               .to_a
sin_vals.each { |x, y| puts "sin(#{x}) = #{y}" }
```

### แบบฝึกหัดที่ 2: Collatz Iterator

```crystal
class CollatzIterator
  include Iterator(Int64)
  
  def initialize(n : Int64)
    @current = n
    @done = false
  end
  
  def next : Int64 | Stop
    return stop if @done
    
    value = @current
    
    if @current == 1
      @done = true
    elsif @current.even?
      @current = @current / 2
    else
      @current = @current * 3 + 1
    end
    
    value
  end
end

# Collatz sequence ของ 27
collatz = CollatzIterator.new(27_i64)
sequence = collatz.to_a
puts "Length: #{sequence.size}"  # => 112
puts "Max: #{sequence.max}"      # => 9232
puts "First 10: #{sequence.first(10).inspect}"
```

---

## สรุป

Iterator ใน Crystal:

| Concept | คำอธิบาย |
|---------|---------|
| `Iterator(T)` | Module สำหรับ custom iterators |
| `def next` | คืน T หรือ `Iterator::Stop` |
| `stop` | สัญญาณสิ้นสุด |
| `chain` | เชื่อม iterators |
| `cycle` | วนซ้ำ |
| `map` | แปลงค่า lazy |
| `select/reject` | กรองค่า lazy |
| `zip` | รวม iterators |
| `first(n)` | เอาแค่ n ตัว |
| `to_a` | แปลงเป็น Array |
| Lazy evaluation | คำนวณเมื่อต้องการ |

**ข้อดีของ Iterator:**
- Lazy evaluation ประหยัด memory
- สามารถ represent infinite sequences
- Composable: chain หลาย operations
- Efficient สำหรับ large datasets

---

*ต่อไป: Part 53 - Array Methods ขั้นสูง*
