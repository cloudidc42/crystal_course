# Part 179: Memory Optimization ใน Crystal

## บทนำ

การ optimize memory ใน Crystal ช่วยลด GC pressure, เพิ่ม cache efficiency, และทำให้ applications ทำงานเร็วขึ้น เราจะเรียนรู้ patterns สำคัญสำหรับการใช้ memory อย่างมีประสิทธิภาพ

## Struct vs Class

```crystal
# Class: allocated on heap, GC pressure
class PointClass
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end

  def distance_to(other : PointClass) : Float64
    Math.sqrt((x - other.x) ** 2 + (y - other.y) ** 2)
  end
end

# Struct: allocated on stack (or inline in other struct/array), no GC pressure
struct PointStruct
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end

  def distance_to(other : PointStruct) : Float64
    Math.sqrt((x - other.x) ** 2 + (y - other.y) ** 2)
  end
end

# เมื่อไหร่ใช้ struct:
# - Value types: coordinates, colors, dimensions
# - Immutable data
# - สร้าง/ทำลายบ่อย
# - ขนาดเล็ก (< ~64 bytes)

# ตัวอย่าง value objects ที่เหมาะกับ struct
struct Color
  getter r : UInt8
  getter g : UInt8
  getter b : UInt8
  getter a : UInt8

  def initialize(@r, @g, @b, @a = 255_u8)
  end

  def blend(other : Color, alpha : Float64) : Color
    Color.new(
      ((r * (1 - alpha) + other.r * alpha).to_u8),
      ((g * (1 - alpha) + other.g * alpha).to_u8),
      ((b * (1 - alpha) + other.b * alpha).to_u8)
    )
  end

  def to_hex : String
    "#%02x%02x%02x" % [r, g, b]
  end
end

struct Rect
  getter x : Float64
  getter y : Float64
  getter width : Float64
  getter height : Float64

  def initialize(@x, @y, @width, @height)
  end

  def area : Float64
    width * height
  end

  def contains?(px : Float64, py : Float64) : Bool
    px >= x && px <= x + width && py >= y && py <= y + height
  end

  def intersects?(other : Rect) : Bool
    x < other.x + other.width && x + width > other.x &&
    y < other.y + other.height && y + height > other.y
  end
end
```

## หลีกเลี่ยง Allocations ใน Hot Paths

```crystal
# BAD: สร้าง intermediate arrays ใน hot path
def process_bad(numbers : Array(Int32)) : Array(Int32)
  numbers
    .map { |n| n * 2 }      # สร้าง array ใหม่
    .select { |n| n > 10 }  # สร้าง array ใหม่อีก
    .map { |n| n + 1 }      # สร้าง array ใหม่อีก
end

# GOOD: ใช้ lazy evaluation
def process_good(numbers : Array(Int32)) : Array(Int32)
  numbers
    .each
    .map { |n| n * 2 }
    .select { |n| n > 10 }
    .map { |n| n + 1 }
    .to_a  # สร้าง array แค่ครั้งเดียว
end

# BETTER: เขียน loop เดียว ไม่มี intermediate
def process_better(numbers : Array(Int32)) : Array(Int32)
  result = [] of Int32
  numbers.each do |n|
    doubled = n * 2
    result << (doubled + 1) if doubled > 10
  end
  result
end

# Pre-allocate ขนาดที่รู้ล่วงหน้า
def process_preallocate(numbers : Array(Int32)) : Array(Int32)
  result = Array(Int32).new(numbers.size)  # pre-allocate
  numbers.each do |n|
    doubled = n * 2
    result << (doubled + 1) if doubled > 10
  end
  result
end
```

## Object Pooling

```crystal
# Object pool: reuse objects แทนสร้างใหม่
class ObjectPool(T)
  @pool : Array(T)
  @factory : -> T
  @max_size : Int32

  def initialize(@max_size : Int32, &@factory : -> T)
    @pool = Array(T).new(@max_size)
  end

  def acquire : T
    @pool.pop? || @factory.call
  end

  def release(obj : T)
    @pool << obj if @pool.size < @max_size
  end

  def with_object(&block : T ->)
    obj = acquire
    begin
      block.call(obj)
    ensure
      release(obj)
    end
  end

  def size : Int32
    @pool.size
  end
end

# ตัวอย่าง: pool ของ String::Builder
string_pool = ObjectPool(String::Builder).new(10) do
  String::Builder.new
end

# ใช้ pool
1000.times do
  string_pool.with_object do |sb|
    sb.clear
    10.times { |i| sb << "item#{i}" }
    result = sb.to_s
    # ใช้ result...
  end
end

# ตัวอย่าง: pool ของ buffer
class BufferPool
  BUFFER_SIZE = 4096

  @pool : Channel(Bytes)

  def initialize(pool_size : Int32 = 10)
    @pool = Channel(Bytes).new(pool_size)
    pool_size.times do
      @pool.send(Bytes.new(BUFFER_SIZE))
    end
  end

  def acquire : Bytes
    @pool.receive? || Bytes.new(BUFFER_SIZE)
  end

  def release(buf : Bytes)
    @pool.send(buf) rescue nil
  end
end

pool = BufferPool.new(20)

# เรียกใช้
buf = pool.acquire
begin
  # ใช้ buf สำหรับ I/O operations
  buf.fill(0_u8)
ensure
  pool.release(buf)
end
```

## Lazy Initialization

```crystal
# Lazy initialization: สร้าง object เมื่อจำเป็นครั้งแรกเท่านั้น
class Config
  # Lazy loading จาก disk/network
  @settings : Hash(String, String)?
  @cache : LRUCache(String, String)?

  def settings : Hash(String, String)
    @settings ||= load_settings
  end

  def cache : LRUCache(String, String)
    @cache ||= LRUCache(String, String).new(1000)
  end

  private def load_settings : Hash(String, String)
    puts "Loading settings from disk..."
    # อ่าน config file
    {"host" => "localhost", "port" => "8080"}
  end
end

# Lazy สำหรับ expensive computation
class Statistics
  def initialize(@data : Array(Float64))
  end

  @mean : Float64?
  @std_dev : Float64?
  @sorted : Array(Float64)?

  def mean : Float64
    @mean ||= @data.sum / @data.size
  end

  def std_dev : Float64
    @std_dev ||= begin
      m = mean
      variance = @data.sum { |x| (x - m) ** 2 } / @data.size
      Math.sqrt(variance)
    end
  end

  def median : Float64
    s = sorted
    mid = s.size // 2
    s.size.odd? ? s[mid] : (s[mid - 1] + s[mid]) / 2.0
  end

  private def sorted : Array(Float64)
    @sorted ||= @data.sort
  end
end

data = Array.new(10_000) { rand * 100.0 }
stats = Statistics.new(data)

puts stats.mean     # คำนวณ mean ครั้งแรก
puts stats.mean     # ใช้ cached value
puts stats.std_dev  # คำนวณ std_dev ครั้งแรก
```

## String Optimization

```crystal
# หลีกเลี่ยง string allocations ใน hot paths

# BAD: สร้าง string ใหม่ทุกครั้ง
def format_log_bad(level : String, message : String) : String
  "[#{Time.local}] #{level.upcase}: #{message}"
end

# GOOD: ใช้ String.build
def format_log_good(level : String, message : String) : String
  String.build do |sb|
    sb << "["
    sb << Time.local
    sb << "] "
    sb << level.upcase
    sb << ": "
    sb << message
  end
end

# BETTER: เขียนไปยัง IO โดยตรง ไม่สร้าง string เลย
def format_log_io(io : IO, level : String, message : String)
  io << "["
  io << Time.local
  io << "] "
  level.each_char { |c| io << c.upcase }
  io << ": "
  io << message
  io << "\n"
end

# ใช้ StringPool สำหรับ strings ที่ซ้ำกันมาก
require "string_pool"
pool = StringPool.new

# แทนที่จะสร้าง String ใหม่ทุกครั้ง
status_codes = ["GET", "POST", "PUT", "DELETE", "PATCH"]
request_methods = Array.new(10_000) { status_codes.sample }

# ใช้ pool เพื่อ intern strings
interned = request_methods.map { |m| pool.get(m) }
# ตอนนี้ strings เดียวกันใช้ object เดียวกัน
```

## Avoid Boxing

```crystal
# Boxing: การ wrap primitive ใน heap-allocated object

# BAD: Union type ทำให้เกิด boxing
def process(value : Int32 | Float64 | String)
  case value
  when Int32   then value * 2
  when Float64 then value * 2.0
  when String  then value.upcase
  end
end

# GOOD: overloaded methods ไม่มี boxing
def process(value : Int32) : Int32
  value * 2
end

def process(value : Float64) : Float64
  value * 2.0
end

def process(value : String) : String
  value.upcase
end

# หรือ use generics
def process_generic(value : T) : T forall T
  {% if T == Int32 %}
    value * 2
  {% elsif T == Float64 %}
    value * 2.0
  {% elsif T == String %}
    value.upcase
  {% else %}
    value
  {% end %}
end
```

## แบบฝึกหัด

1. เปลี่ยน `class Vector3D` เป็น `struct Vector3D` แล้ววัดความแตกต่าง memory/speed ใน benchmark
2. สร้าง object pool สำหรับ HTTP request objects ที่ reuse ระหว่าง requests
3. เขียน pipeline function ที่ process ข้อมูลด้วย lazy evaluation ไม่สร้าง intermediate arrays
4. วัด allocation ของ `String.build` vs concatenation สำหรับ strings ขนาดต่างๆ

## สรุป

Memory Optimization ใน Crystal:
- **struct vs class**: ใช้ struct สำหรับ value types เล็กๆ ลด heap allocations
- **Lazy evaluation**: `.each.map.select` ลด intermediate arrays
- **Object pooling**: reuse objects หลีกเลี่ยง frequent allocation
- **Lazy initialization**: สร้าง objects เมื่อจำเป็นเท่านั้น
- **String.build**: สร้าง string ครั้งเดียวแทน multiple concatenations
- **Write to IO**: หลีกเลี่ยง string creation โดยเขียนตรง IO
- **StringPool**: intern repeated strings ประหยัด memory
- **Avoid boxing**: ใช้ typed methods แทน union types เมื่อทำได้
