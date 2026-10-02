# Part 178: Benchmarking ใน Crystal

## บทนำ

Crystal มี `Benchmark` module ใน standard library สำหรับวัด performance ของ code อย่างแม่นยำ ช่วยเปรียบเทียบ implementations ต่างๆ และหา bottlenecks

## Benchmark.bm - Basic Timing

```crystal
require "benchmark"

# วัดเวลาแบบพื้นฐาน
Benchmark.bm do |x|
  x.report("Array.new") do
    Array.new(10_000, 0)
  end

  x.report("[] * 10000") do
    [] of Int32
    10_000.times { |i| i }
  end

  x.report("push loop") do
    arr = [] of Int32
    10_000.times { |i| arr << i }
    arr
  end
end

# Output ตัวอย่าง:
#                 user     system      total        real
# Array.new   0.000010   0.000001   0.000011   0.000012
# [] * 10000  0.000025   0.000000   0.000025   0.000026
# push loop   0.000050   0.000000   0.000050   0.000051
```

## Benchmark.ips - Iterations Per Second

```crystal
require "benchmark"

# ips (iterations per second) ดีกว่าสำหรับเปรียบเทียบ
Benchmark.ips do |x|
  x.report("string concat (+)") do
    result = ""
    10.times { |i| result = result + "item#{i}" }
    result
  end

  x.report("string concat (<<)") do
    result = ""
    10.times { |i| result << "item#{i}" }
    result
  end

  x.report("String.build") do
    String.build do |sb|
      10.times { |i| sb << "item#{i}" }
    end
  end

  x.compare!  # เปรียบเทียบและบอกว่าอันไหนเร็วกว่า
end

# Output ตัวอย่าง:
# string concat (+)   153.12k ( 6.53µs) (± 5.12%)  ...  fastest
# string concat (<<)  892.34k ( 1.12µs) (± 3.21%)  ...  5.83x slower
# String.build      1.24M  ( 0.81µs) (± 2.98%)  ... 8.10x slower
```

## Benchmark.ips Options

```crystal
require "benchmark"

# ปรับ warmup และ calculation time
Benchmark.ips(warmup: 2.0, calculation: 5.0) do |x|
  x.report("fibonacci recursive") do
    fib_recursive(20)
  end

  x.report("fibonacci iterative") do
    fib_iterative(20)
  end

  x.report("fibonacci memoized") do
    fib_memo(20)
  end

  x.compare!
end

def fib_recursive(n : Int32) : Int32
  return n if n <= 1
  fib_recursive(n - 1) + fib_recursive(n - 2)
end

def fib_iterative(n : Int32) : Int32
  return n if n <= 1
  a, b = 0, 1
  (n - 1).times { a, b = b, a + b }
  b
end

MEMO = {} of Int32 => Int32
def fib_memo(n : Int32) : Int32
  return n if n <= 1
  MEMO[n] ||= fib_memo(n - 1) + fib_memo(n - 2)
end
```

## Manual Timing ด้วย Time.monotonic

```crystal
# วัดเวลาด้วย monotonic clock (ไม่ขึ้นกับ wall clock)
def measure(label : String, iterations : Int32 = 1000, &block)
  # Warmup
  100.times { block.call }

  # วัดจริง
  start = Time.monotonic
  iterations.times { block.call }
  elapsed = Time.monotonic - start

  avg_ns = elapsed.total_nanoseconds / iterations
  avg_us = avg_ns / 1000.0
  avg_ms = avg_us / 1000.0

  puts "#{label}:"
  puts "  Total: #{elapsed.total_milliseconds.round(3)}ms"
  puts "  Per iteration: #{avg_us.round(3)}µs"
  puts "  Iterations/sec: #{(1_000_000_000.0 / avg_ns).round(0).to_i}"
  puts
end

measure("Hash lookup", 100_000) do
  h = {"key1" => 1, "key2" => 2, "key3" => 3}
  h["key2"]
end

measure("Array binary search", 100_000) do
  arr = (1..1000).to_a
  arr.bsearch { |x| x >= 500 }
end

measure("Regex match", 10_000) do
  "hello@example.com".match(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
end
```

## Memory Measurement

```crystal
require "gc"

# วัด memory allocation
def measure_memory(label : String, &block)
  GC.collect
  before_stats = GC.stats
  before_heap = before_stats.heap_size
  before_total = before_stats.total_bytes

  result = block.call

  GC.collect
  after_stats = GC.stats
  after_total = after_stats.total_bytes

  allocated = after_total - before_total

  puts "#{label}:"
  puts "  Allocated: #{(allocated / 1024.0).round(2)}KB"
  puts "  Heap: #{(after_stats.heap_size / 1024.0 / 1024.0).round(2)}MB"
  puts

  result
end

measure_memory("Array(Int32) x 100k") do
  Array.new(100_000, 0)
end

measure_memory("Hash(String, Int32) x 10k") do
  h = {} of String => Int32
  10_000.times { |i| h["key#{i}"] = i }
  h
end

measure_memory("String concat x 1k") do
  result = ""
  1_000.times { |i| result += "item#{i}" }
  result
end

measure_memory("String.build x 1k") do
  String.build do |sb|
    1_000.times { |i| sb << "item#{i}" }
  end
end
```

## Microbenchmarks vs Macrobenchmarks

```crystal
require "benchmark"

# MICROBENCHMARK: วัด specific operations
puts "=== Microbenchmarks ==="
Benchmark.ips do |x|
  x.report("Int32 addition") { 1_i32 + 2_i32 }
  x.report("Float64 addition") { 1.0_f64 + 2.0_f64 }
  x.report("String comparison") { "hello" == "world" }
  x.report("Nil check") { value = nil; value.nil? }
  x.compare!
end

# MACROBENCHMARK: วัด full operation
puts "\n=== Macrobenchmarks ==="

def process_data_v1(data : Array(Int32)) : Int32
  # Version 1: ใช้ map + sum
  data.map { |x| x * x }.sum
end

def process_data_v2(data : Array(Int32)) : Int32
  # Version 2: ใช้ reduce
  data.reduce(0) { |acc, x| acc + x * x }
end

def process_data_v3(data : Array(Int32)) : Int32
  # Version 3: imperative loop
  total = 0
  data.each { |x| total += x * x }
  total
end

data = Array.new(10_000) { rand(1000) }

Benchmark.ips do |x|
  x.report("map + sum") { process_data_v1(data) }
  x.report("reduce") { process_data_v2(data) }
  x.report("imperative") { process_data_v3(data) }
  x.compare!
end
```

## Benchmark Suite

```crystal
require "benchmark"

# สร้าง benchmark suite ที่ครบถ้วน
class BenchmarkSuite
  record Result, name : String, ips : Float64, std_dev : Float64

  def self.run(name : String, &block) : Float64
    samples = [] of Float64
    warmup_count = 100
    measure_count = 1000

    # Warmup
    warmup_count.times { block.call }

    # Measure
    measure_count.times do
      start = Time.monotonic
      block.call
      elapsed = (Time.monotonic - start).total_microseconds
      samples << elapsed
    end

    avg = samples.sum / samples.size
    variance = samples.sum { |s| (s - avg) ** 2 } / samples.size
    std_dev = Math.sqrt(variance)

    ips = 1_000_000.0 / avg

    puts "#{name.ljust(30)}: #{ips.round(0).to_i.to_s.rjust(10)} ips  ±#{std_dev.round(2)}µs"
    ips
  end
end

puts "=== Sorting Algorithms ==="
n = 1000
BenchmarkSuite.run("Array#sort (built-in)") do
  Array.new(n) { rand(10000) }.sort
end

BenchmarkSuite.run("bubble sort (n=#{n})") do
  arr = Array.new(n) { rand(10000) }
  (n - 1).times do |i|
    (n - i - 1).times do |j|
      if arr[j] > arr[j + 1]
        arr[j], arr[j + 1] = arr[j + 1], arr[j]
      end
    end
  end
  arr
end
```

## แบบฝึกหัด

1. เปรียบเทียบ `Array#include?` vs `Set#includes?` สำหรับ collection ขนาดต่างๆ (10, 100, 1000, 10000 elements)
2. วัด memory และ speed ของ `struct` vs `class` สำหรับ value objects
3. สร้าง benchmark ที่วัด JSON serialization/deserialization ของ different data sizes
4. วัดความแตกต่างระหว่าง `--release` flag กับ debug build

## สรุป

Benchmarking ใน Crystal:
- **Benchmark.bm**: วัด wall clock time สำหรับ sequential comparisons
- **Benchmark.ips**: วัด iterations/second เหมาะสำหรับ fast operations
- **Time.monotonic**: manual timing ที่แม่นยำกว่า `Time.now`
- **GC.stats**: วัด memory allocation
- **warmup**: ให้ CPU warm up cache ก่อนวัด
- **compare!**: แสดง relative performance เปรียบเทียบ
- **microbenchmark**: วัด specific operations
- **macrobenchmark**: วัด end-to-end scenarios

Benchmark ที่ดีต้องมี warmup, ทำซ้ำหลายรอบ, และ build ด้วย `--release` flag
