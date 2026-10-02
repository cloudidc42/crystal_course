# Part 165: Performance Testing ใน Crystal

## บทนำ

Performance Testing ช่วยวัดและเปรียบเทียบประสิทธิภาพของ code Crystal มี `Benchmark` module ที่ built-in มาให้ใช้ได้ทันที

## Benchmark Module พื้นฐาน

```crystal
require "benchmark"

# Benchmark.bm - วัดเวลาแต่ละ block
Benchmark.bm do |x|
  x.report("string concat") do
    result = ""
    10000.times { |i| result += i.to_s }
  end

  x.report("string builder") do
    result = String.build do |s|
      10000.times { |i| s << i.to_s }
    end
  end

  x.report("array join") do
    arr = Array(String).new(10000) { |i| i.to_s }
    result = arr.join
  end
end
```

ผลลัพธ์ตัวอย่าง:
```
                    user     system      total        real
string concat   0.890000   0.000000   0.890000   (  0.892314)
string builder  0.001000   0.000000   0.001000   (  0.001234)
array join      0.002000   0.000000   0.002000   (  0.002456)
```

## Benchmark.ips - Iterations Per Second

```crystal
require "benchmark"

# วัด iterations per second (มีประสิทธิภาพมากกว่า bm)
Benchmark.ips do |x|
  x.report("fibonacci recursive") do
    fib_recursive(20)
  end

  x.report("fibonacci iterative") do
    fib_iterative(20)
  end

  x.report("fibonacci memoized") do
    fib_memoized(20)
  end

  x.compare!  # แสดงการเปรียบเทียบ
end

def fib_recursive(n : Int32) : Int64
  return n.to_i64 if n <= 1
  fib_recursive(n - 1) + fib_recursive(n - 2)
end

def fib_iterative(n : Int32) : Int64
  return n.to_i64 if n <= 1
  a, b = 0_i64, 1_i64
  (n - 1).times { a, b = b, a + b }
  b
end

FIB_CACHE = {} of Int32 => Int64

def fib_memoized(n : Int32) : Int64
  return n.to_i64 if n <= 1
  FIB_CACHE[n] ||= fib_memoized(n - 1) + fib_memoized(n - 2)
end
```

ผลลัพธ์ตัวอย่าง:
```
fibonacci recursive  1.45k  (688.68µs) (±12.8%)  18.95k slower
fibonacci iterative  27.53M (  36.32ns) (± 2.1%)       fastest
fibonacci memoized   18.41M (  54.32ns) (± 3.5%)   1.49× slower
```

## Benchmark.memory

```crystal
require "benchmark"

# วัด memory allocation
Benchmark.ips do |x|
  x.report("array push") do
    arr = [] of Int32
    1000.times { |i| arr << i }
    arr
  end

  x.report("array new with block") do
    Array(Int32).new(1000) { |i| i }
  end

  x.report("array from range") do
    (0...1000).to_a
  end
end
```

## การวัดเวลาแบบ Manual

```crystal
require "benchmark"

# วัดเวลาแบบ manual
start = Time.monotonic

# code ที่ต้องการวัด
result = (1..1_000_000).sum

elapsed = Time.monotonic - start
puts "Elapsed: #{elapsed.total_milliseconds.round(2)}ms"
puts "Result: #{result}"

# วัดด้วย monotonic clock (แม่นยำกว่า)
def measure(&block) : Time::Span
  start = Time.monotonic
  block.call
  Time.monotonic - start
end

time = measure { heavy_computation }
puts "Time: #{time.total_seconds}s"
```

## Comparing Implementations

```crystal
require "benchmark"

# เปรียบเทียบ sorting algorithms
module Sorters
  def self.bubble_sort(arr : Array(Int32)) : Array(Int32)
    a = arr.dup
    n = a.size
    loop do
      swapped = false
      (n - 1).times do |i|
        if a[i] > a[i + 1]
          a[i], a[i + 1] = a[i + 1], a[i]
          swapped = true
        end
      end
      break unless swapped
    end
    a
  end

  def self.quick_sort(arr : Array(Int32)) : Array(Int32)
    return arr if arr.size <= 1
    pivot = arr[arr.size // 2]
    left  = arr.select { |x| x < pivot }
    mid   = arr.select { |x| x == pivot }
    right = arr.select { |x| x > pivot }
    quick_sort(left) + mid + quick_sort(right)
  end

  def self.merge_sort(arr : Array(Int32)) : Array(Int32)
    return arr if arr.size <= 1
    mid = arr.size // 2
    left = merge_sort(arr[0...mid])
    right = merge_sort(arr[mid..])
    merge(left, right)
  end

  private def self.merge(left : Array(Int32), right : Array(Int32)) : Array(Int32)
    result = [] of Int32
    i = j = 0
    while i < left.size && j < right.size
      if left[i] <= right[j]
        result << left[i]
        i += 1
      else
        result << right[j]
        j += 1
      end
    end
    result + left[i..] + right[j..]
  end
end

# สร้างข้อมูลทดสอบ
data_100  = Array.new(100)  { rand(1000) }
data_1000 = Array.new(1000) { rand(10000) }

puts "=== Sorting 100 elements ==="
Benchmark.ips do |x|
  x.report("bubble sort")  { Sorters.bubble_sort(data_100) }
  x.report("quick sort")   { Sorters.quick_sort(data_100) }
  x.report("merge sort")   { Sorters.merge_sort(data_100) }
  x.report("built-in sort") { data_100.sort }
  x.compare!
end

puts "\n=== Sorting 1000 elements ==="
Benchmark.ips do |x|
  x.report("quick sort")   { Sorters.quick_sort(data_1000) }
  x.report("merge sort")   { Sorters.merge_sort(data_1000) }
  x.report("built-in sort") { data_1000.sort }
  x.compare!
end
```

## Memory Allocation Testing

```crystal
require "benchmark"
require "gc"

# วัด memory usage
def memory_used : Int64
  GC.stats.heap_size.to_i64
end

def measure_memory(&block) : Int64
  GC.collect
  before = memory_used
  block.call
  GC.collect
  after = memory_used
  after - before
end

# เปรียบเทียบ memory usage ของ data structures
puts "=== Memory Usage Comparison ==="

# Array ธรรมดา
arr_mem = measure_memory do
  arr = Array(Int32).new(100_000) { |i| i }
  arr.size  # prevent optimization
end
puts "Array(Int32) 100k: ~#{arr_mem / 1024}KB"

# Tuple array
tuple_mem = measure_memory do
  arr = Array(Tuple(Int32, Int32)).new(100_000) { |i| {i, i * 2} }
  arr.size
end
puts "Array(Tuple) 100k: ~#{tuple_mem / 1024}KB"

# Struct array
struct Point
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

struct_mem = measure_memory do
  arr = Array(Point).new(100_000) { |i| Point.new(i, i * 2) }
  arr.size
end
puts "Array(Struct) 100k: ~#{struct_mem / 1024}KB"

# Class array (heap allocated)
class PointClass
  property x : Int32
  property y : Int32
  def initialize(@x, @y); end
end

class_mem = measure_memory do
  arr = Array(PointClass).new(100_000) { |i| PointClass.new(i, i * 2) }
  arr.size
end
puts "Array(Class) 100k: ~#{class_mem / 1024}KB"
```

## Benchmark HTTP Endpoints

```crystal
require "benchmark"
require "http/client"

# Load test HTTP endpoint
puts "=== HTTP Endpoint Benchmark ==="

client = HTTP::Client.new("localhost", 3000)

# Warmup
10.times { client.get("/api/health") }

Benchmark.ips do |x|
  x.report("GET /health") do
    client.get("/api/health")
  end

  x.report("GET /users") do
    client.get("/api/users")
  end

  x.report("POST /users") do
    client.post(
      "/api/users",
      headers: HTTP::Headers{"Content-Type" => "application/json"},
      body: %({"email":"test@example.com","password":"pass"})
    )
  end
end
```

## Microbenchmarks vs Macrobenchmarks

```crystal
require "benchmark"

# Microbenchmark: ทดสอบ operation เดี่ยวๆ
puts "=== Microbenchmarks ==="
Benchmark.ips do |x|
  x.report("hash lookup") do
    h = {"key" => "value"}
    h["key"]?
  end

  x.report("regex match") do
    "hello@example.com".match(/\A[\w+\-.]+@[a-z\d\-]+(\.[a-z\d\-]+)*\.[a-z]+\z/i)
  end

  x.report("string split") do
    "one,two,three,four,five".split(",")
  end

  x.report("json parse") do
    JSON.parse(%({"name":"Alice","age":30}))
  end
end

# Macrobenchmark: ทดสอบ workflow ครบ
puts "\n=== Macrobenchmarks ==="

def process_order(items : Array(NamedTuple(name: String, price: Float64, qty: Int32)))
  subtotal = items.sum { |i| i[:price] * i[:qty] }
  tax = subtotal * 0.07
  shipping = subtotal >= 500 ? 0.0 : 50.0
  total = subtotal + tax + shipping

  {
    subtotal: subtotal.round(2),
    tax:      tax.round(2),
    shipping: shipping,
    total:    total.round(2)
  }
end

items = [
  {name: "Book A", price: 299.0, qty: 2},
  {name: "Book B", price: 149.0, qty: 1},
  {name: "Pen",    price: 15.0,  qty: 5}
]

Benchmark.ips do |x|
  x.report("process order") { process_order(items) }
end
```

## Performance Regression Testing

```crystal
# spec/performance_spec.cr
require "spec"
require "benchmark"

describe "Performance" do
  # กำหนด time limits
  MAX_TIME_MS = {
    "order_calculation" => 10.0,
    "user_search"       => 50.0,
    "report_generation" => 500.0
  }

  it "order calculation เสร็จภายใน #{MAX_TIME_MS["order_calculation"]}ms" do
    items = Array.new(100) { |i|
      {name: "item#{i}", price: rand(100.0..1000.0), qty: rand(1..5)}
    }

    elapsed = Time.measure do
      1000.times { OrderCalculator.new.calculate(items) }
    end

    avg_ms = elapsed.total_milliseconds / 1000
    avg_ms.should be < MAX_TIME_MS["order_calculation"]
  end

  it "user search เสร็จภายใน #{MAX_TIME_MS["user_search"]}ms" do
    # ใส่ข้อมูล test 10000 users
    db = TestDB.connection
    10000.times { |i| db.exec("INSERT INTO users ...") }

    elapsed = Time.measure do
      UserSearch.new(db).search("alice")
    end

    elapsed.total_milliseconds.should be < MAX_TIME_MS["user_search"]
  end
end
```

## สร้าง Performance Monitor

```crystal
# src/performance_monitor.cr
class PerformanceMonitor
  record Metric,
    name : String,
    duration_ms : Float64,
    memory_bytes : Int64,
    timestamp : Time

  @@metrics : Array(Metric) = [] of Metric

  def self.measure(name : String, &block)
    GC.collect
    mem_before = GC.stats.heap_size.to_i64
    start = Time.monotonic

    result = block.call

    elapsed = Time.monotonic - start
    mem_after = GC.stats.heap_size.to_i64

    metric = Metric.new(
      name:         name,
      duration_ms:  elapsed.total_milliseconds,
      memory_bytes: mem_after - mem_before,
      timestamp:    Time.utc
    )

    @@metrics << metric

    if metric.duration_ms > 100
      Log.warn { "Slow operation: #{name} took #{metric.duration_ms.round(2)}ms" }
    end

    result
  end

  def self.report
    return if @@metrics.empty?

    puts "\n=== Performance Summary ==="
    grouped = @@metrics.group_by(&.name)

    grouped.each do |name, metrics|
      durations = metrics.map(&.duration_ms)
      avg = durations.sum / durations.size
      max = durations.max
      min = durations.min

      puts "#{name}:"
      puts "  count: #{metrics.size}"
      puts "  avg: #{avg.round(2)}ms"
      puts "  min: #{min.round(2)}ms"
      puts "  max: #{max.round(2)}ms"
    end
  end

  def self.reset
    @@metrics.clear
  end
end

# ใช้งาน
result = PerformanceMonitor.measure("database_query") do
  User.where(active: true).limit(100).to_a
end

result2 = PerformanceMonitor.measure("json_serialization") do
  result.map(&.to_json)
end

PerformanceMonitor.report
```

## แบบฝึกหัด

1. เปรียบเทียบ performance ของ `Array#select` vs manual loop vs `Array#reject`
2. วัด throughput ของ HTTP server ด้วย concurrent requests (10, 100, 1000 concurrent)
3. เปรียบเทียบ memory ของ `Hash` vs `Array` ของ struct สำหรับ 1 million records
4. สร้าง performance regression test ที่รันใน CI และ fail เมื่อ performance ลดลง > 20%

## สรุป

Performance Testing ใน Crystal:
- **Benchmark.bm**: วัดเวลา รัน code block หนึ่งครั้ง
- **Benchmark.ips**: วัด iterations/second รองรับ warmup และ compare
- **Memory measurement**: วัด allocation ด้วย GC.stats
- **Microbenchmarks**: ทดสอบ operations เดี่ยวๆ
- **Macrobenchmarks**: ทดสอบ workflows ครบ
- **Regression testing**: ป้องกัน performance degradation ใน CI

Crystal ทำงานเร็วมากด้วย LLVM backend แต่ยังต้องวัดและ optimize ด้วย data จริง
