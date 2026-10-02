# Part 92: Structs ใน Crystal

## บทนำ

Struct ใน Crystal เป็น value type ต่างจาก class ที่เป็น reference type Structs มีประสิทธิภาพสูงกว่าสำหรับข้อมูลขนาดเล็กเพราะเก็บบน stack และ copy by value

## Struct vs Class

```crystal
# Class - reference type
class PointClass
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end
end

# Struct - value type
struct PointStruct
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end
end

# ความแตกต่างในการ copy
class_point1 = PointClass.new(1.0, 2.0)
class_point2 = class_point1  # Share reference
class_point2.x = 99.0
puts class_point1.x  # => 99.0 (เปลี่ยนตาม!)

struct_point1 = PointStruct.new(1.0, 2.0)
struct_point2 = struct_point1  # Copy value
struct_point2.x = 99.0
puts struct_point1.x  # => 1.0 (ไม่เปลี่ยน!)

# Memory
puts class_point1.object_id  # Address บน heap
puts struct_point1.object_id # ไม่มี object_id (value type)
```

## Value Semantics

```crystal
struct Color
  getter red : UInt8
  getter green : UInt8
  getter blue : UInt8
  getter alpha : UInt8

  def initialize(@red, @green, @blue, @alpha = 255_u8)
  end

  def with_alpha(alpha : UInt8) : Color
    Color.new(@red, @green, @blue, alpha)
  end

  def blend(other : Color, ratio : Float64 = 0.5) : Color
    r = (@red * (1 - ratio) + other.red * ratio).to_u8
    g = (@green * (1 - ratio) + other.green * ratio).to_u8
    b = (@blue * (1 - ratio) + other.blue * ratio).to_u8
    Color.new(r, g, b)
  end

  def to_hex : String
    "#%02X%02X%02X" % [@red, @green, @blue]
  end

  def +(other : Color) : Color
    blend(other)
  end

  def to_s : String
    "rgba(#{@red}, #{@green}, #{@blue}, #{@alpha})"
  end
end

red = Color.new(255_u8, 0_u8, 0_u8)
blue = Color.new(0_u8, 0_u8, 255_u8)
purple = red + blue

puts red.to_hex     # => #FF0000
puts blue.to_hex    # => #0000FF
puts purple.to_hex  # => #7F007F (blended)

# Value semantics ใน function
def darken(color : Color, factor : Float64 = 0.5) : Color
  Color.new(
    (color.red * factor).to_u8,
    (color.green * factor).to_u8,
    (color.blue * factor).to_u8,
    color.alpha
  )
end

original = Color.new(200_u8, 100_u8, 50_u8)
darker = darken(original)

puts original.to_hex  # ไม่เปลี่ยน
puts darker.to_hex    # เปลี่ยน
```

## Struct Methods

```crystal
struct Vector3
  getter x : Float64
  getter y : Float64
  getter z : Float64

  def initialize(@x = 0.0, @y = 0.0, @z = 0.0)
  end

  # Arithmetic operators
  def +(other : Vector3) : Vector3
    Vector3.new(@x + other.x, @y + other.y, @z + other.z)
  end

  def -(other : Vector3) : Vector3
    Vector3.new(@x - other.x, @y - other.y, @z - other.z)
  end

  def *(scalar : Float64) : Vector3
    Vector3.new(@x * scalar, @y * scalar, @z * scalar)
  end

  def /(scalar : Float64) : Vector3
    raise ArgumentError.new("Cannot divide by zero") if scalar == 0.0
    Vector3.new(@x / scalar, @y / scalar, @z / scalar)
  end

  # Dot product
  def dot(other : Vector3) : Float64
    @x * other.x + @y * other.y + @z * other.z
  end

  # Cross product
  def cross(other : Vector3) : Vector3
    Vector3.new(
      @y * other.z - @z * other.y,
      @z * other.x - @x * other.z,
      @x * other.y - @y * other.x
    )
  end

  def magnitude : Float64
    Math.sqrt(@x ** 2 + @y ** 2 + @z ** 2)
  end

  def normalize : Vector3
    mag = magnitude
    raise "Cannot normalize zero vector" if mag == 0.0
    self / mag
  end

  def ==(other : Vector3) : Bool
    @x == other.x && @y == other.y && @z == other.z
  end

  def to_s : String
    "(#{@x.round(4)}, #{@y.round(4)}, #{@z.round(4)})"
  end

  # Class methods
  def self.zero : Vector3
    new(0.0, 0.0, 0.0)
  end

  def self.up : Vector3
    new(0.0, 1.0, 0.0)
  end

  def self.right : Vector3
    new(1.0, 0.0, 0.0)
  end

  def self.forward : Vector3
    new(0.0, 0.0, 1.0)
  end
end

v1 = Vector3.new(1.0, 2.0, 3.0)
v2 = Vector3.new(4.0, 5.0, 6.0)

puts v1 + v2           # (5.0, 7.0, 9.0)
puts v1.dot(v2)        # 32.0
puts v1.cross(v2)      # (-3.0, 6.0, -3.0)
puts v1.magnitude      # ~3.742
puts v1.normalize      # normalized vector
```

## Struct กับ Modules

```crystal
# Struct สามารถ include modules ได้
module Describable
  abstract def describe : String
end

module Serializable
  abstract def to_dict : Hash(String, String)
end

struct Employee
  include Describable
  include Serializable
  include Comparable(Employee)

  getter id : Int32
  getter name : String
  getter department : String
  getter salary : Float64

  def initialize(@id, @name, @department, @salary)
  end

  def describe : String
    "#{@name} (#{@department}) - ฿#{@salary}"
  end

  def to_dict : Hash(String, String)
    {
      "id"         => @id.to_s,
      "name"       => @name,
      "department" => @department,
      "salary"     => @salary.to_s,
    }
  end

  def <=>(other : Employee) : Int32
    @salary <=> other.salary
  end

  def to_s : String
    describe
  end
end

employees = [
  Employee.new(1, "สมชาย", "Engineering", 85000.0),
  Employee.new(2, "สมหญิง", "Marketing", 75000.0),
  Employee.new(3, "สมศักดิ์", "Engineering", 90000.0),
]

# Sort ด้วย Comparable
sorted = employees.sort
sorted.each { |e| puts e }

# Max salary
highest = employees.max
puts "Highest paid: #{highest.name}"
```

## Struct Inheritance ไม่มีใน Crystal

```crystal
# Crystal structs ไม่รองรับ inheritance จาก struct อื่น
# แต่สามารถ inherit จาก abstract struct ได้

abstract struct Shape
  abstract def area : Float64
  abstract def perimeter : Float64

  def describe : String
    "#{self.class}: area=#{area.round(2)}, perimeter=#{perimeter.round(2)}"
  end
end

struct Circle < Shape
  getter radius : Float64

  def initialize(@radius)
  end

  def area : Float64
    Math::PI * @radius ** 2
  end

  def perimeter : Float64
    2 * Math::PI * @radius
  end
end

struct Square < Shape
  getter side : Float64

  def initialize(@side)
  end

  def area : Float64
    @side ** 2
  end

  def perimeter : Float64
    4 * @side
  end
end

shapes = [Circle.new(5.0), Square.new(4.0)] of Shape
shapes.each { |s| puts s.describe }
```

## Struct Performance

```crystal
# Struct เหมาะสำหรับ small, immutable data
struct Point2D
  getter x : Float64
  getter y : Float64

  def initialize(@x, @y)
  end

  def distance_to(other : Point2D) : Float64
    dx = @x - other.x
    dy = @y - other.y
    Math.sqrt(dx * dx + dy * dy)
  end
end

# Struct array - เก็บ values ต่อเนื่องใน memory (cache friendly)
points = Array(Point2D).new(1000) do |i|
  Point2D.new(rand(100.0), rand(100.0))
end

# คำนวณ centroid
total_x = points.sum(&.x)
total_y = points.sum(&.y)
centroid = Point2D.new(total_x / points.size, total_y / points.size)

puts "Centroid: (#{centroid.x.round(2)}, #{centroid.y.round(2)})"

# หา closest point
closest = points.min_by { |p| p.distance_to(centroid) }
puts "Closest to centroid: (#{closest.x.round(2)}, #{closest.y.round(2)})"
```

## Record Macro

```crystal
# record macro - shorthand สำหรับ immutable struct
record Point, x : Float64, y : Float64

# เท่ากับ:
# struct Point
#   getter x : Float64
#   getter y : Float64
#   def initialize(@x : Float64, @y : Float64); end
# end

p = Point.new(3.0, 4.0)
puts p.x  # => 3.0
puts p.y  # => 4.0

# record รองรับ copy with
p2 = p.copy_with(x: 10.0)
puts p2.x  # => 10.0
puts p2.y  # => 4.0 (ยังเดิม)

# record ที่ซับซ้อน
record Person, name : String, age : Int32, email : String = "unknown"

alice = Person.new("Alice", 30)
bob = Person.new("Bob", 25, "bob@example.com")

puts alice  # => Person(@name="Alice", @age=30, @email="unknown")
puts bob.email  # => bob@example.com

# record กับ default values
record Config,
  host : String = "localhost",
  port : Int32 = 8080,
  debug : Bool = false

default_config = Config.new
custom_config = Config.new(host: "example.com", port: 443, debug: true)
debug_local = default_config.copy_with(debug: true)

puts default_config
puts custom_config
puts debug_local
```

## Mutable vs Immutable Structs

```crystal
# Immutable struct (ใช้ getter)
struct ImmutablePoint
  getter x : Float64
  getter y : Float64

  def initialize(@x, @y)
  end

  # Return new struct แทน mutate
  def move(dx : Float64, dy : Float64) : ImmutablePoint
    ImmutablePoint.new(@x + dx, @y + dy)
  end
end

# Mutable struct (ใช้ property)
struct MutablePoint
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end

  # Mutate in place
  def move!(dx : Float64, dy : Float64)
    @x += dx
    @y += dy
  end
end

# Immutable - functional style
ip = ImmutablePoint.new(0.0, 0.0)
ip2 = ip.move(3.0, 4.0)
puts ip   # ยังอยู่ที่ 0,0
puts ip2  # 3,4

# Mutable - imperative style
mp = MutablePoint.new(0.0, 0.0)
mp.move!(3.0, 4.0)
puts mp  # 3,4
```

## Struct ใน Collections

```crystal
# Struct ใน Array - stored by value
struct Measurement
  getter value : Float64
  getter unit : String
  getter timestamp : Int64

  def initialize(@value, @unit, @timestamp = Time.utc.to_unix)
  end

  def to_si : Float64
    case @unit
    when "km"   then @value * 1000
    when "cm"   then @value / 100
    when "kg"   then @value
    when "lb"   then @value * 0.453592
    else             @value
    end
  end

  def to_s : String
    "#{@value} #{@unit}"
  end
end

# Array ของ struct values
measurements = [
  Measurement.new(5.0, "km"),
  Measurement.new(150.0, "cm"),
  Measurement.new(70.0, "kg"),
  Measurement.new(165.0, "lb"),
]

# Process ด้วย functional methods
si_values = measurements.map(&.to_si)
puts si_values.inspect  # => [5000.0, 1.5, 70.0, 74.8...]

# Group by unit
grouped = measurements.group_by(&.unit)
grouped.each do |unit, measures|
  puts "#{unit}: #{measures.map(&.value).join(", ")}"
end
```

## Struct กับ Generics

```crystal
# Generic struct
struct Pair(A, B)
  getter first : A
  getter second : B

  def initialize(@first : A, @second : B)
  end

  def swap : Pair(B, A)
    Pair(B, A).new(@second, @first)
  end

  def map_first(&block : A -> C) : Pair(C, B) forall C
    Pair(C, B).new(block.call(@first), @second)
  end

  def both_satisfy?(&block : A -> Bool) : Bool
    # ต้องการ A == B ซึ่งไม่ enforce ที่นี่
    block.call(@first)
  end

  def to_tuple : Tuple(A, B)
    {@first, @second}
  end

  def to_s : String
    "(#{@first}, #{@second})"
  end
end

pair = Pair.new("age", 25)
puts pair           # => (age, 25)
puts pair.swap      # => (25, age)

# Generic struct ใน Array
pairs = [
  Pair.new(1, "one"),
  Pair.new(2, "two"),
  Pair.new(3, "three"),
]

pairs.each { |p| puts p }

# แปลงเป็น Hash
hash = pairs.to_h { |p| {p.first, p.second} }
puts hash  # => {1 => "one", 2 => "two", 3 => "three"}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: 2D Matrix Struct
สร้าง `Matrix2x2` struct ที่:
- รองรับ arithmetic (+, -, *)
- หา determinant
- หา inverse
- Transpose

### แบบฝึกหัดที่ 2: Date Range
สร้าง `DateRange` struct ที่:
- มี start_date, end_date
- คำนวณ duration
- Check overlap กับ range อื่น
- Generate array ของ dates ใน range

### แบบฝึกหัดที่ 3: Money Arithmetic
สร้าง `Money` struct ที่:
- รองรับ arithmetic (+, -, *)
- Currency conversion
- Formatting

### แบบฝึกหัดที่ 4: Statistics Struct
สร้าง `Stats` struct ที่:
- คำนวณ mean, median, mode, std_dev
- รับ Array(Float64) ใน constructor
- Immutable - return new Stats เมื่อ add data

## สรุป

Structs ใน Crystal:
- **Value semantics**: copy by value ไม่ใช่ reference
- **Stack allocated**: ไม่ต้องการ garbage collection
- **Performance**: เร็วกว่า class สำหรับ small data
- **Immutable by default**: ถ้าใช้ getter แทน property
- **record macro**: shorthand สำหรับ simple structs

เมื่อใช้ struct:
- ข้อมูลขนาดเล็ก (< ~16 bytes แนะนำ)
- ข้อมูลที่ immutable หรือ copy semantics สมเหตุสมผล
- ต้องการ performance สูง
- Value objects ใน domain model
