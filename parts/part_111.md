# Part 111: Fibers พื้นฐาน

## บทนำ

Fiber ใน Crystal เป็น lightweight cooperative concurrency primitive ที่ทำงานแบบ coroutine Fiber ต่างจาก Thread ตรงที่ทำงานแบบ cooperative (ต้อง yield เอง) ไม่ใช่ preemptive (interrupt ได้ทุกเวลา)

## Fiber.new และ Fiber#resume

```crystal
# สร้าง Fiber พื้นฐาน
fiber = Fiber.new do
  puts "Fiber: ก่อน yield"
  Fiber.yield
  puts "Fiber: หลัง yield"
end

puts "Main: ก่อน resume ครั้งแรก"
fiber.resume
puts "Main: หลัง resume ครั้งแรก"
fiber.resume
puts "Main: หลัง resume ครั้งที่สอง"

# Output:
# Main: ก่อน resume ครั้งแรก
# Fiber: ก่อน yield
# Main: หลัง resume ครั้งแรก
# Fiber: หลัง yield
# Main: หลัง resume ครั้งที่สอง
```

## Fiber.yield

```crystal
# Fiber.yield ส่ง control กลับไปยัง caller
def fibonacci_generator
  Fiber.new do
    a, b = 0, 1
    loop do
      Fiber.yield a
      a, b = b, a + b
    end
  end
end

fib = fibonacci_generator

# Generate 10 Fibonacci numbers
10.times do
  fib.resume
end

# สิ่งที่น่าสังเกต: ใช้ spawn แทน manual resume ใน production code
```

## Generator Pattern กับ Fibers

```crystal
# Generator - ส่งค่ากลับด้วย Fiber.yield
def number_generator(start : Int32, step : Int32 = 1) : Fiber
  Fiber.new do
    current = start
    loop do
      Fiber.yield current
      current += step
    end
  end
end

gen = number_generator(1, 2)  # 1, 3, 5, 7, ...

puts "First 5 odd numbers:"
5.times do
  gen.resume
end

# ใน Crystal ทั่วไป ใช้ Enumerator pattern แทน

# Infinite sequence generator
def range_generator(from : Int32, to : Int32) : Fiber
  Fiber.new do
    (from..to).each do |i|
      Fiber.yield i
    end
  end
end

range_gen = range_generator(1, 5)
5.times { range_gen.resume }
```

## Producer/Consumer Pattern

```crystal
# Producer/Consumer ด้วย Fibers
# ใน Crystal ทั่วไปใช้ Channels แต่ Fibers ก็ทำได้

class FiberQueue(T)
  def initialize
    @buffer = [] of T
    @consumers = [] of Fiber
    @producers = [] of Fiber
  end

  def put(item : T)
    @buffer << item
    if consumer = @consumers.shift?
      consumer.resume
    end
  end

  def get : T
    while @buffer.empty?
      @consumers << Fiber.current
      Fiber.yield
    end
    @buffer.shift
  end
end

# ตัวอย่าง producer/consumer แบบ simple
queue = [] of String
produced = 0
consumed = 0

producer = Fiber.new do
  5.times do |i|
    item = "item_#{i}"
    puts "Producing: #{item}"
    queue << item
    produced += 1
    Fiber.yield
  end
end

consumer = Fiber.new do
  loop do
    if item = queue.shift?
      puts "Consuming: #{item}"
      consumed += 1
    end
    Fiber.yield
  end
end

# Interleave producer and consumer
10.times do
  producer.resume
  consumer.resume
end

puts "Produced: #{produced}, Consumed: #{consumed}"
```

## Cooperative Scheduling

```crystal
# Simple scheduler ด้วย Fibers
class Scheduler
  def initialize
    @fibers = [] of Fiber
  end

  def add(&block)
    @fibers << Fiber.new { block.call }
  end

  def run
    until @fibers.empty?
      @fibers.each_with_index do |fiber, i|
        fiber.resume
        if fiber.dead?
          @fibers.delete_at(i)
        end
      end
    end
  end
end

scheduler = Scheduler.new

scheduler.add do
  3.times do |i|
    puts "Task 1: step #{i}"
    Fiber.yield
  end
end

scheduler.add do
  3.times do |i|
    puts "Task 2: step #{i}"
    Fiber.yield
  end
end

scheduler.add do
  3.times do |i|
    puts "Task 3: step #{i}"
    Fiber.yield
  end
end

puts "Running scheduler:"
scheduler.run

# Output:
# Task 1: step 0
# Task 2: step 0
# Task 3: step 0
# Task 1: step 1
# Task 2: step 1
# Task 3: step 1
# Task 1: step 2
# Task 2: step 2
# Task 3: step 2
```

## Fiber กับ Crystal's spawn

```crystal
# spawn ใน Crystal ใช้ Fibers + EventLoop
# (ไม่ใช่ OS threads จริงๆ ยกเว้นถ้า enable multi-threading)

# spawn เรียก Fiber ใหม่ที่ run ใน event loop
spawn do
  puts "Spawned fiber 1"
end

spawn do
  puts "Spawned fiber 2"
end

puts "Main fiber"

# ใช้ sleep เพื่อให้ event loop มีเวลา run spawned fibers
sleep 0

# Output:
# Main fiber
# Spawned fiber 1
# Spawned fiber 2
```

## Fiber.current

```crystal
# Fiber.current - reference ไปยัง current running Fiber
def who_am_i : String
  "Fiber object_id: #{Fiber.current.object_id}"
end

puts who_am_i  # Main fiber

f = Fiber.new do
  puts who_am_i  # Different fiber
end
f.resume
```

## Fibers เป็น Coroutines

```crystal
# Coroutine pattern - สร้าง iterators ด้วย Fiber
class FiberEnumerator(T)
  include Iterator(T)

  def initialize(&@block : -> Nil)
    @fiber = Fiber.new { @block.call }
    @value = uninitialized T
    @done = false
  end

  def next : T | Iterator::Stop
    return stop if @done

    @fiber.resume
    @done ? stop : @value
  end

  def yield_value(val : T)
    @value = val
    Fiber.yield
  end
end

# ใช้งาน
counter = FiberEnumerator(Int32).new do
  5.times do |i|
    # ต้อง yield กลับไปยัง enumerator
    Fiber.yield
  end
end

# ใน Crystal มี Iterator module ที่ดีกว่า สำหรับ generators
```

## Error Handling ใน Fibers

```crystal
# ข้อผิดพลาดใน Fiber จะ propagate เมื่อ resume
fiber = Fiber.new do
  raise "Error in fiber!"
end

begin
  fiber.resume
rescue ex
  puts "Caught: #{ex.message}"  # => Caught: Error in fiber!
end

# Fiber ที่ dead แล้ว - resume อีกครั้ง raise error
safe_fiber = Fiber.new do
  puts "Fiber running"
end

safe_fiber.resume   # OK
safe_fiber.resume   # Raises: dead fiber called
rescue
  puts "Fiber is already dead"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Number Generator
สร้าง generator ด้วย Fiber:
- Infinite sequence generator
- Fibonacci generator
- Prime number generator

### แบบฝึกหัดที่ 2: Coroutine Calculator
สร้าง calculator ที่:
- แต่ละ operation เป็น step
- รับ input ผ่าน Fiber.yield
- Return ผล intermediate

### แบบฝึกหัดที่ 3: Task Scheduler
สร้าง cooperative scheduler:
- Priority ของ tasks
- Deadline scheduling
- Statistics collection

### แบบฝึกหัดที่ 4: Pipeline ด้วย Fibers
สร้าง data processing pipeline:
- Source → Transform → Filter → Sink
- แต่ละ stage เป็น Fiber
- Backpressure handling

## สรุป

Fibers ใน Crystal:
- **Cooperative**: ต้อง yield เอง ไม่ถูก preempt
- **Lightweight**: เร็วกว่า OS threads มาก
- **Single-threaded**: ทำงานบน single CPU core (default)
- **Event loop**: Crystal ใช้ Fibers + event loop สำหรับ async I/O

Use cases:
1. Generators/Iterators
2. Cooperative multitasking
3. Coroutine patterns
4. State machines ที่มี complex state

Crystal's concurrency model:
- `spawn { }` สร้าง Fiber ใน event loop
- I/O operations yield อัตโนมัติ (non-blocking)
- Channels สำหรับ communication ระหว่าง Fibers
