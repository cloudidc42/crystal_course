# Part 174: Memory Management ใน Crystal

## บทนำ

Crystal ใช้ Boehm Garbage Collector (GC) สำหรับจัดการ memory อัตโนมัติ แต่ก็มีเครื่องมือสำหรับควบคุม GC, ใช้ finalizers, weak references, และ raw pointers เมื่อจำเป็น

## GC พื้นฐานใน Crystal

```crystal
require "gc"

# ดู GC stats
stats = GC.stats
puts "Heap size: #{stats.heap_size / 1024 / 1024}MB"
puts "Free bytes: #{stats.free_bytes / 1024 / 1024}MB"
puts "Total allocated bytes: #{stats.total_bytes / 1024 / 1024}MB"

# ชนิดของ GC
puts GC.is_thread_suspended  # false ถ้า GC ทำงาน
```

## GC.collect

```crystal
require "gc"

# Force garbage collection
puts "Before GC:"
puts "Heap: #{GC.stats.heap_size / 1024}KB"

# สร้าง objects จำนวนมาก
1000.times do
  Array.new(1000, 0)  # สร้าง arrays ที่จะถูก GC
end

puts "After creating objects:"
puts "Heap: #{GC.stats.heap_size / 1024}KB"

# Force GC
GC.collect

puts "After GC.collect:"
puts "Heap: #{GC.stats.heap_size / 1024}KB"
```

## GC.stats

```crystal
require "gc"

struct GCReport
  getter before : GC::Stats
  getter after : GC::Stats
  getter allocated : UInt64

  def initialize
    @before = GC.stats
    @after = @before
    @allocated = 0_u64
  end

  def measure(&block)
    GC.collect
    @before = GC.stats
    block.call
    GC.collect
    @after = GC.stats
    @allocated = (@after.total_bytes - @before.total_bytes)
    self
  end

  def report
    puts "=== GC Report ==="
    puts "Heap before: #{@before.heap_size / 1024}KB"
    puts "Heap after:  #{@after.heap_size / 1024}KB"
    puts "Allocated:   #{@allocated / 1024}KB"
    puts "Delta:       #{(@after.heap_size.to_i64 - @before.heap_size.to_i64) / 1024}KB"
  end
end

# ใช้งาน
report = GCReport.new
report.measure do
  # code ที่ต้องการวัด memory
  data = Array.new(10_000) { |i| "item #{i}" }
  data.map { |s| s.upcase }
end
report.report
```

## Finalizers

```crystal
# Finalizer ถูกเรียกเมื่อ object ถูก GC
class FileHandle
  @@open_count = Atomic(Int32).new(0)

  def initialize(@path : String)
    @fd = open_file(@path)
    @@open_count.add(1)
    puts "Opened: #{@path} (total: #{@@open_count.get})"
  end

  def finalize
    close_file(@fd) if @fd >= 0
    @@open_count.sub(1)
    puts "Closed: #{@path} (total: #{@@open_count.get})"
  end

  def self.open_count
    @@open_count.get
  end

  private def open_file(path : String) : Int32
    # simplified
    0
  end

  private def close_file(fd : Int32)
    # simplified close
  end
end

# Demo finalizer
puts "Creating handles..."
5.times do |i|
  FileHandle.new("/tmp/file#{i}.txt")
end

puts "Open handles: #{FileHandle.open_count}"

# Force GC - finalizers จะถูกเรียก
GC.collect
sleep(0.1)  # ให้ finalizers มีเวลาทำงาน

puts "After GC, Open handles: #{FileHandle.open_count}"
```

### Finalizer ที่ดี

```crystal
class DatabaseConnection
  @connection : LibPQ::Conn* = Pointer(LibPQ::Conn).null
  @closed = false

  def initialize(url : String)
    @connection = LibPQ.connectdb(url)
    check_connection
  end

  def close
    return if @closed
    @closed = true
    LibPQ.finish(@connection) if @connection
    @connection = Pointer(LibPQ::Conn).null
  end

  def finalize
    # safety net - ปิด connection ถ้า user ลืม close
    unless @closed
      Log.warn { "DatabaseConnection was not explicitly closed - closing in finalizer" }
      close
    end
  end

  private def check_connection
    status = LibPQ.status(@connection)
    raise "Connection failed" if status != LibPQ::CONNECTION_OK
  end
end

# best practice: ใช้ ensure
db = DatabaseConnection.new(ENV["DATABASE_URL"])
begin
  # use db
  db.query("SELECT 1")
ensure
  db.close  # ปิด explicitly ดีกว่า รอ GC
end
```

## Weak References

```crystal
# Crystal ไม่มี WeakReference built-in แต่สร้างได้
# ใช้สำหรับ caching โดยไม่ prevent GC

# Simulated WeakReference ด้วย Pointer
class WeakRef(T)
  def initialize(object : T)
    # ไม่ hold strong reference
    @ptr = object.as(Void*)
  end

  def get : T?
    # ใน production ต้องตรวจสอบว่า pointer ยังมีอยู่
    # Crystal ไม่มี built-in weak reference support
    @ptr.as(T)
  rescue
    nil
  end
end

# Cache ที่ไม่ prevent GC (conceptual)
class SoftCache(K, V)
  @store : Hash(K, V) = {} of K => V
  @max_size : Int32

  def initialize(@max_size = 100)
  end

  def get(key : K) : V?
    @store[key]?
  end

  def set(key : K, value : V)
    if @store.size >= @max_size
      # ลบ entries เก่าๆ เมื่อ cache เต็ม
      evict_oldest
    end
    @store[key] = value
  end

  private def evict_oldest
    # ลบ 10% ของ cache
    count = (@store.size * 0.1).to_i + 1
    keys_to_remove = @store.keys.first(count)
    keys_to_remove.each { |k| @store.delete(k) }
    GC.collect  # suggest GC run
  end
end
```

## Pointer(T)

```crystal
# Pointer(T) สำหรับ unsafe, low-level operations

# Pointer พื้นฐาน
value = 42
ptr = pointerof(value)
puts "Value: #{ptr.value}"    # 42
puts "Address: #{ptr}"        # memory address

# แก้ค่าผ่าน pointer
ptr.value = 100
puts "Changed value: #{value}"  # 100

# Array pointer
arr = [1, 2, 3, 4, 5]
ptr = arr.to_unsafe

5.times do |i|
  puts "arr[#{i}] = #{(ptr + i).value}"
end

# Pointer arithmetic
first = ptr.value
second = (ptr + 1).value
puts "First: #{first}, Second: #{second}"

# Unsafe pointer cast
int_ptr = Pointer(Int32).malloc(1)
int_ptr.value = 42
puts "Int: #{int_ptr.value}"
Pointer(Void).new(int_ptr.address)  # cast ไปยัง void*
```

## Memory Allocation

```crystal
# Pointer.malloc - allocate memory manually
ptr = Pointer(Int32).malloc(10)  # allocate 10 ints

begin
  10.times { |i| (ptr + i).value = i * i }
  10.times { |i| puts (ptr + i).value }
ensure
  ptr.free  # ต้อง free เอง!
end

# Pointer.realloc
ptr = Pointer(Int32).malloc(5)
5.times { |i| (ptr + i).value = i }

# ขยาย array
ptr = ptr.realloc(10)
5.times { |i| (ptr + 5 + i).value = (5 + i) * (5 + i) }
10.times { |i| print "#{(ptr + i).value} " }
puts

ptr.free

# Stack allocation ด้วย static arrays
buffer = StaticArray(UInt8, 256).new(0)
# buffer อยู่บน stack ไม่ต้อง free
```

## Memory Zones / Custom Allocators

```crystal
# ใช้ arena allocator สำหรับ batch allocations
class ArenaAllocator
  def initialize(capacity : Int32)
    @buffer = Bytes.new(capacity)
    @offset = 0
    @capacity = capacity
  end

  def allocate(size : Int32) : Slice(UInt8)
    raise "Out of arena memory" if @offset + size > @capacity
    result = @buffer[@offset, size]
    @offset += size
    result
  end

  def reset
    @offset = 0
    # ไม่ต้อง free เพราะใช้ @buffer เดิม
  end

  def used : Int32
    @offset
  end

  def available : Int32
    @capacity - @offset
  end
end

arena = ArenaAllocator.new(1024 * 1024)  # 1MB arena

# Allocate objects จาก arena
1000.times do |i|
  mem = arena.allocate(64)
  mem.to_unsafe.as(UInt8*).value = i.to_u8
end

puts "Used: #{arena.used / 1024}KB"

# Reset ทันที ไม่ต้องรอ GC
arena.reset
puts "After reset: #{arena.used}B"
```

## Monitoring Memory Usage

```crystal
require "gc"

class MemoryMonitor
  def self.report(label : String)
    stats = GC.stats
    heap_mb = stats.heap_size / 1024.0 / 1024.0
    free_mb = stats.free_bytes / 1024.0 / 1024.0
    used_mb = (stats.heap_size - stats.free_bytes) / 1024.0 / 1024.0

    puts "#{label}:"
    puts "  Heap:  #{heap_mb.round(2)}MB"
    puts "  Used:  #{used_mb.round(2)}MB"
    puts "  Free:  #{free_mb.round(2)}MB"
    puts "  Total allocated: #{(stats.total_bytes / 1024.0 / 1024.0).round(2)}MB"
  end

  def self.measure(label : String, &block)
    GC.collect
    before = GC.stats.heap_size
    block.call
    GC.collect
    after = GC.stats.heap_size
    diff_kb = (after.to_i64 - before.to_i64) / 1024
    puts "#{label}: #{diff_kb > 0 ? "+" : ""}#{diff_kb}KB"
  end
end

# ใช้งาน
MemoryMonitor.report("Initial")

MemoryMonitor.measure("Create 1000 strings") do
  1000.times { "Hello World #{rand(1000)}" }
end

MemoryMonitor.measure("Create large array") do
  Array.new(100_000, 42)
end

MemoryMonitor.report("After operations")
```

## แบบฝึกหัด

1. สร้าง object pool ที่ reuse objects แทนการสร้างใหม่ทุกครั้ง
2. วัด memory usage ของ `Array(Int32)` vs `Pointer(Int32)` สำหรับ 1 million elements
3. สร้าง bump pointer allocator ที่ allocate memory ได้เร็วกว่า malloc
4. เขียน finalizer สำหรับ resource wrapper ที่ log เมื่อ resource ถูก GC โดยไม่ได้ explicit close

## สรุป

Memory Management ใน Crystal:
- **GC**: Boehm GC จัดการ memory อัตโนมัติ
- **GC.collect**: Force garbage collection
- **GC.stats**: ดู heap size, free bytes, total allocated
- **Finalizers**: cleanup code ที่รันก่อน object ถูก GC
- **Weak References**: reference ที่ไม่ prevent GC
- **Pointer(T)**: low-level memory access สำหรับ unsafe operations
- **malloc/free**: manual memory management เมื่อจำเป็น

Crystal ออกแบบให้ safe by default แต่ยังให้ unsafe escape hatch เมื่อต้องการ performance สูงสุด
