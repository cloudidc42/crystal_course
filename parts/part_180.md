# Part 180: CPU Optimization ใน Crystal

## บทนำ

CPU optimization เกี่ยวกับการทำให้ code ทำงานได้มากที่สุดต่อ CPU cycle การเข้าใจ algorithmic complexity, cache behavior, branch prediction, และ inlining จะช่วยเพิ่มประสิทธิภาพอย่างมาก

## Algorithmic Complexity

```crystal
# O(n²) - bubble sort
def bubble_sort(arr : Array(Int32)) : Array(Int32)
  n = arr.size
  (n - 1).times do |i|
    (n - i - 1).times do |j|
      if arr[j] > arr[j + 1]
        arr[j], arr[j + 1] = arr[j + 1], arr[j]
      end
    end
  end
  arr
end

# O(n log n) - merge sort
def merge_sort(arr : Array(Int32)) : Array(Int32)
  return arr if arr.size <= 1

  mid = arr.size // 2
  left = merge_sort(arr[0...mid])
  right = merge_sort(arr[mid..])
  merge(left, right)
end

def merge(left : Array(Int32), right : Array(Int32)) : Array(Int32)
  result = Array(Int32).new(left.size + right.size)
  i, j = 0, 0

  while i < left.size && j < right.size
    if left[i] <= right[j]
      result << left[i]
      i += 1
    else
      result << right[j]
      j += 1
    end
  end

  result.concat(left[i..]) if i < left.size
  result.concat(right[j..]) if j < right.size
  result
end

# เลือก algorithm ที่เหมาะสมตาม input size
def smart_sort(arr : Array(Int32)) : Array(Int32)
  if arr.size < 16
    insertion_sort(arr)  # Fast for small arrays
  else
    arr.sort  # Built-in (introsort)
  end
end

def insertion_sort(arr : Array(Int32)) : Array(Int32)
  (1...arr.size).each do |i|
    key = arr[i]
    j = i - 1
    while j >= 0 && arr[j] > key
      arr[j + 1] = arr[j]
      j -= 1
    end
    arr[j + 1] = key
  end
  arr
end
```

## Cache-Friendly Data Structures

```crystal
# Cache-friendly: ข้อมูลที่ต่อเนื่องใน memory (Structure of Arrays)
# แทนที่ Array of Structures

# BAD: Array of Structures (AoS) - cache unfriendly สำหรับ bulk operations
class Particle
  property x : Float64
  property y : Float64
  property z : Float64
  property vx : Float64
  property vy : Float64
  property vz : Float64
  property mass : Float64

  def initialize(@x, @y, @z, @vx, @vy, @vz, @mass)
  end
end

# GOOD: Structure of Arrays (SoA) - cache friendly สำหรับ bulk operations
class ParticleSystem
  getter xs : Array(Float64)
  getter ys : Array(Float64)
  getter zs : Array(Float64)
  getter vxs : Array(Float64)
  getter vys : Array(Float64)
  getter vzs : Array(Float64)
  getter masses : Array(Float64)
  getter count : Int32

  def initialize(n : Int32)
    @count = n
    @xs = Array.new(n, 0.0)
    @ys = Array.new(n, 0.0)
    @zs = Array.new(n, 0.0)
    @vxs = Array.new(n, 0.0)
    @vys = Array.new(n, 0.0)
    @vzs = Array.new(n, 0.0)
    @masses = Array.new(n, 1.0)
  end

  # Cache-friendly: เข้าถึง xs ทั้งหมดก่อน (sequential memory access)
  def update_positions(dt : Float64)
    count.times do |i|
      @xs[i] += @vxs[i] * dt
      @ys[i] += @vys[i] * dt
      @zs[i] += @vzs[i] * dt
    end
  end

  def kinetic_energy : Float64
    total = 0.0
    count.times do |i|
      v2 = @vxs[i] ** 2 + @vys[i] ** 2 + @vzs[i] ** 2
      total += 0.5 * @masses[i] * v2
    end
    total
  end
end

particles = ParticleSystem.new(10_000)
dt = 0.016
particles.update_positions(dt)
puts "KE: #{particles.kinetic_energy}"
```

## Loop Optimization

```crystal
# Hoist loop-invariant computations
# BAD
def process_bad(data : Array(Float64), scale : Float64) : Array(Float64)
  data.map { |x| x * Math.sqrt(scale) }  # Math.sqrt คำนวณทุก iteration
end

# GOOD
def process_good(data : Array(Float64), scale : Float64) : Array(Float64)
  sqrt_scale = Math.sqrt(scale)  # คำนวณครั้งเดียว
  data.map { |x| x * sqrt_scale }
end

# Loop unrolling สำหรับ simple operations
def sum_unrolled(arr : Array(Float64)) : Float64
  total = 0.0
  n = arr.size
  i = 0

  # Process 4 elements ต่อ iteration
  while i + 4 <= n
    total += arr[i] + arr[i + 1] + arr[i + 2] + arr[i + 3]
    i += 4
  end

  # Handle remainder
  while i < n
    total += arr[i]
    i += 1
  end
  total
end

# หลีกเลี่ยง bounds checking ด้วย unsafe access
def sum_unsafe(arr : Array(Float64)) : Float64
  total = 0.0
  ptr = arr.to_unsafe
  arr.size.times { |i| total += (ptr + i).value }
  total
end
```

## Branch Prediction

```crystal
# Branch prediction: CPU ทำนาย branches ล่วงหน้า
# Predictable branches เร็วกว่า unpredictable

# BAD: unpredictable branch
def classify_random(n : Int32) : String
  if rand(2) == 0
    "even"
  else
    "odd"
  end
end

# GOOD: predictable branch (data-driven, not random)
def classify_number(n : Int32) : String
  n.even? ? "even" : "odd"
end

# หลีกเลี่ยง branches ด้วย arithmetic
# BAD
def abs_with_branch(n : Int32) : Int32
  n < 0 ? -n : n
end

# GOOD: branchless
def abs_branchless(n : Int32) : Int32
  mask = n >> 31  # -1 ถ้าลบ, 0 ถ้าบวก
  (n ^ mask) - mask
end

# Branchless min/max
def min_branchless(a : Int32, b : Int32) : Int32
  b + ((a - b) & ((a - b) >> 31))
end

def max_branchless(a : Int32, b : Int32) : Int32
  a - ((a - b) & ((a - b) >> 31))
end

# Sort น้อยๆ ด้วย branchless
def sort2(a : Int32, b : Int32) : {Int32, Int32}
  diff = a - b
  mask = diff >> 31
  a2 = b + (diff & mask)
  b2 = a - (diff & mask)
  {a2, b2}
end
```

## Inlining

```crystal
# Crystal compiler inline functions อัตโนมัติ
# แต่ใช้ @[AlwaysInline] บังคับ inline ได้

@[AlwaysInline]
def fast_add(a : Int32, b : Int32) : Int32
  a + b
end

@[NoInline]
def slow_function(a : Int32, b : Int32) : Int32
  # บังคับให้ไม่ inline (เช่น function ใหญ่)
  a + b
end

# Hot path function ควร inline
struct Vec2
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end

  @[AlwaysInline]
  def +(other : Vec2) : Vec2
    Vec2.new(@x + other.x, @y + other.y)
  end

  @[AlwaysInline]
  def *(scalar : Float64) : Vec2
    Vec2.new(@x * scalar, @y * scalar)
  end

  @[AlwaysInline]
  def dot(other : Vec2) : Float64
    @x * other.x + @y * other.y
  end

  @[AlwaysInline]
  def length : Float64
    Math.sqrt(@x * @x + @y * @y)
  end

  @[AlwaysInline]
  def normalize : Vec2
    len = length
    len > 0 ? self * (1.0 / len) : self
  end
end
```

## Data-Oriented Design

```crystal
# Data-Oriented Design: organize data สำหรับ processing efficiency

# Entity Component System (ECS) - data-oriented game/simulation pattern
module ECS
  alias EntityId = Int32

  # Components เป็น plain data (struct)
  record Position, x : Float64, y : Float64
  record Velocity, dx : Float64, dy : Float64
  record Health, current : Int32, max : Int32

  class World
    @positions : Hash(EntityId, Position) = {} of EntityId => Position
    @velocities : Hash(EntityId, Velocity) = {} of EntityId => Velocity
    @healths : Hash(EntityId, Health) = {} of EntityId => Health
    @next_id = Atomic(Int32).new(0)

    def create_entity : EntityId
      @next_id.add(1)
    end

    def add_position(entity : EntityId, pos : Position)
      @positions[entity] = pos
    end

    def add_velocity(entity : EntityId, vel : Velocity)
      @velocities[entity] = vel
    end

    # System: process ทุก entities ที่มีทั้ง Position และ Velocity
    def movement_system(dt : Float64)
      @velocities.each do |id, vel|
        if pos = @positions[id]?
          @positions[id] = Position.new(
            pos.x + vel.dx * dt,
            pos.y + vel.dy * dt
          )
        end
      end
    end

    def entity_count : Int32
      @next_id.get
    end
  end
end

world = ECS::World.new

# สร้าง entities
100.times do |i|
  entity = world.create_entity
  world.add_position(entity, ECS::Position.new(i.to_f64, 0.0))
  world.add_velocity(entity, ECS::Velocity.new(1.0, 0.5))
end

# Update simulation
10.times { world.movement_system(0.016) }
puts "Entities: #{world.entity_count}"
```

## แบบฝึกหัด

1. เปลี่ยน Array of Structures เป็น Structure of Arrays สำหรับ particle simulation และวัด speedup
2. สร้าง branchless clamp function `clamp(value, min, max)` และเปรียบเทียบกับ version ที่มี branches
3. Implement cache-friendly matrix multiplication (blocked/tiled algorithm)
4. วัดความแตกต่างของ `@[AlwaysInline]` vs normal function call สำหรับ tight loops

## สรุป

CPU Optimization ใน Crystal:
- **Algorithmic complexity**: เลือก algorithm ที่ดี O(n log n) แทน O(n²)
- **Cache-friendly**: Structure of Arrays ดีกว่า Array of Structures สำหรับ bulk processing
- **Loop hoisting**: ย้าย invariant computations ออกนอก loop
- **Branch prediction**: code predictable paths, ใช้ branchless arithmetic
- **Inlining**: @[AlwaysInline] สำหรับ hot path functions
- **Data-Oriented Design**: organize data ให้ทำ sequential access ได้
- **Loop unrolling**: process หลาย elements ต่อ iteration ลด loop overhead
