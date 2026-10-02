# Part 183: GC Tuning ใน Crystal

## บทนำ

Crystal ใช้ Boehm-Demers-Weiser Garbage Collector (BDW-GC) ซึ่ง tunable ผ่าน environment variables และ Crystal API การ tune GC ช่วยลด pause times และ memory overhead

## GC Environment Variables

```bash
# ตัวแปร environment สำหรับ tuning GC

# Initial heap size (default: 4MB)
GC_INITIAL_HEAP_SIZE=64M ./app

# Maximum heap size
GC_MAXIMUM_HEAP_SIZE=2G ./app

# Heap growth factor (default: 1.2 = 20% growth)
GC_HEAP_GROWTH_FACTOR=1.5 ./app

# Free space ratio (default: 0.2 = 20%)
GC_FREE_SPACE_RATIO=0.3 ./app

# Force full GC after N bytes allocated
GC_FULL_FREQ=5 ./app  # Full GC ทุก 5 allocations

# Parallel GC marking (ต้องการ --preview_mt)
GC_NPROCS=4 ./app

# GC verbose logging
GC_PRINT_STATS=1 ./app

# รัน app พร้อม GC tuning
GC_INITIAL_HEAP_SIZE=128M GC_MAXIMUM_HEAP_SIZE=1G ./app
```

## GC.collect และ GC.compact

```crystal
require "gc"

# Force full GC collection
GC.collect

# GC.compact - defragment heap (BDW-GC support)
# รวม free memory ลด fragmentation
GC.compact

# ดู stats หลัง operations
def gc_report
  stats = GC.stats
  puts "Heap: #{stats.heap_size / 1024 / 1024}MB"
  puts "Free: #{stats.free_bytes / 1024 / 1024}MB"
  puts "Total allocated: #{stats.total_bytes / 1024 / 1024}MB"
  puts "Used: #{(stats.heap_size - stats.free_bytes) / 1024 / 1024}MB"
end

puts "=== Before large allocation ==="
gc_report

# สร้าง lots of temporary objects
data = (1..100_000).map { |i| "string_#{i}" }

puts "\n=== After allocation ==="
gc_report

data = nil  # Remove reference

GC.collect
puts "\n=== After GC.collect ==="
gc_report

GC.compact
puts "\n=== After GC.compact ==="
gc_report
```

## Reducing GC Pressure

```crystal
# Strategy 1: Use value types (struct) instead of reference types (class)
# Less heap allocations = less GC work

# BAD: class สำหรับ small value objects
class Point3D_Bad
  property x, y, z : Float64

  def initialize(@x, @y, @z)
  end
end

# GOOD: struct ไม่ allocate บน heap
struct Point3D_Good
  property x, y, z : Float64

  def initialize(@x, @y, @z)
  end
end

# Strategy 2: Reuse objects (Object Pooling)
class RequestContext
  property path : String = ""
  property method : String = ""
  property headers : Hash(String, String) = {} of String => String
  property body : String = ""

  def reset
    @path = ""
    @method = ""
    @headers.clear
    @body = ""
    self
  end
end

# Pool ของ RequestContext objects
class ContextPool
  @pool : Array(RequestContext)
  @mutex = Mutex.new

  def initialize(size : Int32 = 100)
    @pool = Array.new(size) { RequestContext.new }
  end

  def acquire : RequestContext
    @mutex.synchronize do
      @pool.pop? || RequestContext.new
    end
  end

  def release(ctx : RequestContext)
    @mutex.synchronize do
      @pool << ctx.reset if @pool.size < 100
    end
  end
end

# Strategy 3: Use String.build ไม่ใช่ concatenation
# BAD: สร้าง string ใหม่ทุกครั้ง
def build_json_bad(items : Array(String)) : String
  result = "["
  items.each_with_index do |item, i|
    result += "\"#{item}\""
    result += "," if i < items.size - 1
  end
  result + "]"
end

# GOOD: ใช้ IO::Memory หรือ String.build
def build_json_good(items : Array(String)) : String
  String.build do |sb|
    sb << "["
    items.each_with_index do |item, i|
      sb << "\""
      sb << item
      sb << "\""
      sb << "," if i < items.size - 1
    end
    sb << "]"
  end
end
```

## Arena Allocator

```crystal
# Arena allocator: batch allocate, batch free
# ดีสำหรับ request-scoped objects
class Arena
  BLOCK_SIZE = 65536  # 64KB blocks

  @blocks : Array(Bytes)
  @current_block : Bytes
  @offset : Int32

  def initialize
    @blocks = [] of Bytes
    @current_block = Bytes.new(BLOCK_SIZE)
    @blocks << @current_block
    @offset = 0
  end

  def allocate(size : Int32) : Bytes
    aligned_size = (size + 7) & ~7  # Align to 8 bytes

    if @offset + aligned_size > @current_block.size
      # ต้องการ block ใหม่
      new_block_size = [BLOCK_SIZE, aligned_size + 8].max
      @current_block = Bytes.new(new_block_size)
      @blocks << @current_block
      @offset = 0
    end

    result = @current_block[@offset, aligned_size]
    @offset += aligned_size
    result
  end

  def reset
    # Reset ทั้งหมดโดยไม่ต้อง free แต่ละ object
    @offset = 0
    @current_block = @blocks[0]
    # ทิ้ง blocks ส่วนเกิน
    if @blocks.size > 1
      @blocks = [@blocks[0]]
    end
  end

  def total_allocated : Int32
    total = 0
    @blocks.each_with_index do |block, i|
      total += (i == @blocks.size - 1) ? @offset : block.size
    end
    total
  end
end

# ใช้ Arena สำหรับ request processing
arena = Arena.new

1000.times do |req|
  # Allocate request-scoped objects จาก arena
  buf = arena.allocate(256)
  # ... ใช้ buf ...

  # ท้าย request, reset arena แทน GC
  arena.reset
end

puts "Total allocated: #{arena.total_allocated / 1024}KB"
```

## GC Tuning สำหรับ Long-Running Servers

```crystal
require "gc"

# GC tuning สำหรับ web server
class GCManager
  COLLECT_INTERVAL = 60.seconds
  COMPACT_INTERVAL = 3600.seconds  # 1 hour

  def self.start_background_gc
    spawn do
      last_collect = Time.monotonic
      last_compact = Time.monotonic

      loop do
        sleep(5.seconds)

        now = Time.monotonic

        if now - last_collect >= COLLECT_INTERVAL
          GC.collect
          last_collect = now
        end

        if now - last_compact >= COMPACT_INTERVAL
          GC.compact
          last_compact = now
          log_stats("After compact")
        end
      end
    end
  end

  def self.log_stats(label : String = "GC Stats")
    stats = GC.stats
    heap_mb = stats.heap_size.to_f / 1024 / 1024
    free_mb = stats.free_bytes.to_f / 1024 / 1024
    total_mb = stats.total_bytes.to_f / 1024 / 1024

    Log.info { "#{label}: heap=#{heap_mb.round(1)}MB free=#{free_mb.round(1)}MB total_allocated=#{total_mb.round(1)}MB" }
  end
end

# สำหรับ production server
GCManager.start_background_gc
```

## Memory Leak Detection

```crystal
require "gc"

# ตรวจ memory leaks ด้วย tracking
class LeakDetector
  @baseline : UInt64 = 0_u64
  @check_points : Array({label: String, total: UInt64}) = [] of {label: String, total: UInt64}

  def baseline
    GC.collect
    @baseline = GC.stats.total_bytes
  end

  def checkpoint(label : String)
    GC.collect
    total = GC.stats.total_bytes
    @check_points << {label: label, total: total}
  end

  def report(threshold_kb : Int32 = 100)
    puts "=== Leak Detection Report ==="
    puts "Baseline: #{@baseline / 1024}KB"
    @check_points.each do |cp|
      growth = cp[:total].to_i64 - @baseline.to_i64
      growth_kb = growth / 1024
      flag = growth_kb > threshold_kb ? " *** POTENTIAL LEAK ***" : ""
      puts "  #{cp[:label].ljust(30)}: +#{growth_kb}KB#{flag}"
    end
  end
end

detector = LeakDetector.new
detector.baseline

# Simulate operations
1000.times do
  data = Array.new(100) { "leaked_#{rand(1000)}" }
  # Intentional leak: ไม่ release data
end

detector.checkpoint("after 1000 iterations")
detector.report
```

## แบบฝึกหัด

1. เปรียบเทียบ GC pressure ระหว่าง `class` และ `struct` สำหรับ high-frequency object creation
2. ใช้ Arena allocator สำหรับ HTTP request processing และวัดผล
3. สร้าง GC monitoring ที่ alert เมื่อ heap size เกิน threshold
4. Tune `GC_INITIAL_HEAP_SIZE` และ `GC_HEAP_GROWTH_FACTOR` สำหรับ web server workload

## สรุป

GC Tuning ใน Crystal:
- **GC_INITIAL_HEAP_SIZE**: ตั้ง initial heap ใหญ่ลด early collection
- **GC_MAXIMUM_HEAP_SIZE**: จำกัด memory usage
- **GC_HEAP_GROWTH_FACTOR**: ควบคุมความเร็ว heap growth
- **GC.collect**: force collection เมื่อต้องการ
- **GC.compact**: defragment heap ลด fragmentation
- **struct**: ลด heap allocations, ลด GC pressure
- **Object pooling**: reuse objects หลีกเลี่ยง frequent allocation
- **Arena allocator**: batch allocate/free สำหรับ request-scoped data
- **String.build**: ลด string allocation
