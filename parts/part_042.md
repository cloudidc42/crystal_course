# Part 42: Operator Overloading ใน Crystal

## บทนำ

Operator Overloading คือการกำหนดความหมายของ operators (+, -, *, /, ==, ฯลฯ) สำหรับ class ที่เราสร้างเอง Crystal รองรับ operator overloading อย่างเต็มรูปแบบ ทำให้เราสร้าง types ที่ใช้งานได้เหมือน built-in types

---

## 42.1 พื้นฐาน Operator Overloading

ใน Crystal operators เป็น methods พิเศษที่มีชื่อตาม operator นั้นๆ

```crystal
class Vector2D
  getter x : Float64
  getter y : Float64
  
  def initialize(@x : Float64, @y : Float64)
  end
  
  # Arithmetic operators
  def +(other : Vector2D) : Vector2D
    Vector2D.new(@x + other.x, @y + other.y)
  end
  
  def -(other : Vector2D) : Vector2D
    Vector2D.new(@x - other.x, @y - other.y)
  end
  
  def *(scalar : Float64) : Vector2D
    Vector2D.new(@x * scalar, @y * scalar)
  end
  
  def /(scalar : Float64) : Vector2D
    raise "Cannot divide by zero" if scalar == 0
    Vector2D.new(@x / scalar, @y / scalar)
  end
  
  # Equality
  def ==(other : Vector2D) : Bool
    @x == other.x && @y == other.y
  end
  
  # Unary minus
  def - : Vector2D
    Vector2D.new(-@x, -@y)
  end
  
  def to_s(io : IO) : Nil
    io << "(#{@x}, #{@y})"
  end
  
  def magnitude : Float64
    Math.sqrt(@x ** 2 + @y ** 2)
  end
end

v1 = Vector2D.new(3.0, 4.0)
v2 = Vector2D.new(1.0, 2.0)

puts v1 + v2      # => (4.0, 6.0)
puts v1 - v2      # => (2.0, 2.0)
puts v1 * 2.0     # => (6.0, 8.0)
puts v1 / 2.0     # => (1.5, 2.0)
puts -v1          # => (-3.0, -4.0)
puts v1 == v2     # => false
puts v1.magnitude # => 5.0
```

---

## 42.2 Arithmetic Operators

### def + (บวก)

```crystal
class Money
  getter amount : Float64
  getter currency : String
  
  def initialize(@amount : Float64, @currency : String = "THB")
  end
  
  def +(other : Money) : Money
    raise "Currency mismatch" if @currency != other.currency
    Money.new(@amount + other.amount, @currency)
  end
  
  def +(value : Float64 | Int32) : Money
    Money.new(@amount + value.to_f, @currency)
  end
  
  def to_s(io : IO) : Nil
    io << "#{@currency} #{@amount.round(2)}"
  end
end

price1 = Money.new(100.0)
price2 = Money.new(50.50)
total = price1 + price2
puts total  # => THB 150.5

# บวกกับตัวเลขโดยตรง
puts price1 + 25.0  # => THB 125.0
```

### def - (ลบ)

```crystal
class Temperature
  getter value : Float64
  getter unit : Symbol
  
  def initialize(@value : Float64, @unit : Symbol = :celsius)
  end
  
  def -(other : Temperature) : Temperature
    # แปลงเป็น celsius ก่อนคำนวณ
    diff = to_celsius - other.to_celsius
    Temperature.new(diff)
  end
  
  def -(degrees : Float64) : Temperature
    Temperature.new(@value - degrees, @unit)
  end
  
  def to_celsius : Float64
    case @unit
    when :celsius
      @value
    when :fahrenheit
      (@value - 32) * 5 / 9
    when :kelvin
      @value - 273.15
    else
      @value
    end
  end
  
  def to_s(io : IO) : Nil
    io << "#{@value}°#{@unit.to_s.upcase[0]}"
  end
end

t1 = Temperature.new(100.0)
t2 = Temperature.new(37.0)
diff = t1 - t2
puts diff  # => 63.0°C

puts t1 - 10.0  # => 90.0°C
```

### def * (คูณ)

```crystal
class Matrix2x2
  def initialize(@a : Float64, @b : Float64, @c : Float64, @d : Float64)
  end
  
  def *(other : Matrix2x2) : Matrix2x2
    Matrix2x2.new(
      @a * other.a + @b * other.c,
      @a * other.b + @b * other.d,
      @c * other.a + @d * other.c,
      @c * other.b + @d * other.d
    )
  end
  
  def *(scalar : Float64) : Matrix2x2
    Matrix2x2.new(@a * scalar, @b * scalar, @c * scalar, @d * scalar)
  end
  
  protected getter a, b, c, d
  
  def to_s(io : IO) : Nil
    io << "[#{@a}, #{@b}; #{@c}, #{@d}]"
  end
  
  def determinant : Float64
    @a * @d - @b * @c
  end
end

m1 = Matrix2x2.new(1.0, 2.0, 3.0, 4.0)
m2 = Matrix2x2.new(5.0, 6.0, 7.0, 8.0)

puts m1 * m2      # => [19.0, 22.0; 43.0, 50.0]
puts m1 * 2.0     # => [2.0, 4.0; 6.0, 8.0]
puts m1.determinant  # => -2.0
```

### def / (หาร) และ def % (โมดูลัส)

```crystal
class Fraction
  getter numerator : Int32
  getter denominator : Int32
  
  def initialize(num : Int32, den : Int32)
    raise "Denominator cannot be zero" if den == 0
    gcd_val = gcd(num.abs, den.abs)
    sign = den < 0 ? -1 : 1
    @numerator = sign * num / gcd_val
    @denominator = sign * den / gcd_val
  end
  
  def +(other : Fraction) : Fraction
    Fraction.new(
      @numerator * other.denominator + other.numerator * @denominator,
      @denominator * other.denominator
    )
  end
  
  def -(other : Fraction) : Fraction
    Fraction.new(
      @numerator * other.denominator - other.numerator * @denominator,
      @denominator * other.denominator
    )
  end
  
  def *(other : Fraction) : Fraction
    Fraction.new(@numerator * other.numerator, @denominator * other.denominator)
  end
  
  def /(other : Fraction) : Fraction
    Fraction.new(@numerator * other.denominator, @denominator * other.numerator)
  end
  
  def ==(other : Fraction) : Bool
    @numerator == other.numerator && @denominator == other.denominator
  end
  
  private def gcd(a : Int32, b : Int32) : Int32
    b == 0 ? a : gcd(b, a % b)
  end
  
  def to_s(io : IO) : Nil
    if @denominator == 1
      io << @numerator
    else
      io << "#{@numerator}/#{@denominator}"
    end
  end
  
  def to_f : Float64
    @numerator.to_f / @denominator
  end
end

a = Fraction.new(1, 2)
b = Fraction.new(1, 3)

puts a + b   # => 5/6
puts a - b   # => 1/6
puts a * b   # => 1/6
puts a / b   # => 3/2
puts a == b  # => false
puts Fraction.new(2, 4) == Fraction.new(1, 2)  # => true
```

---

## 42.3 Comparison Operators

### def == (เท่ากับ)

```crystal
class Point3D
  def initialize(@x : Float64, @y : Float64, @z : Float64)
  end
  
  def ==(other : Point3D) : Bool
    @x == other.x && @y == other.y && @z == other.z
  end
  
  # != จะถูก generate อัตโนมัติจาก ==
  
  protected getter x, y, z
  
  def to_s(io : IO) : Nil
    io << "(#{@x}, #{@y}, #{@z})"
  end
end

p1 = Point3D.new(1.0, 2.0, 3.0)
p2 = Point3D.new(1.0, 2.0, 3.0)
p3 = Point3D.new(4.0, 5.0, 6.0)

puts p1 == p2  # => true
puts p1 != p3  # => true
puts p1 == p3  # => false
```

### def <=> (Spaceship Operator)

Spaceship operator เป็นรากฐานของการ sorting และ comparison

```crystal
class Student
  include Comparable(Student)
  
  getter name : String
  getter gpa : Float64
  
  def initialize(@name : String, @gpa : Float64)
  end
  
  # <=> เป็นพื้นฐานของ Comparable module
  def <=>(other : Student) : Int32
    @gpa <=> other.gpa
  end
  
  def to_s(io : IO) : Nil
    io << "#{@name} (GPA: #{@gpa})"
  end
end

students = [
  Student.new("Charlie", 3.2),
  Student.new("Alice", 3.8),
  Student.new("Bob", 3.5),
  Student.new("Dave", 2.9)
]

sorted = students.sort
sorted.each { |s| puts s }
# Dave (GPA: 2.9)
# Charlie (GPA: 3.2)
# Bob (GPA: 3.5)
# Alice (GPA: 3.8)

puts students.max  # => Alice (GPA: 3.8)
puts students.min  # => Dave (GPA: 2.9)

# Comparable ให้ <, >, <=, >= อัตโนมัติ
alice = Student.new("Alice", 3.8)
bob = Student.new("Bob", 3.5)
puts alice > bob   # => true
puts bob < alice   # => true
```

---

## 42.4 Index Operators

### def [] (อ่าน)

```crystal
class Grid
  def initialize(@width : Int32, @height : Int32)
    @data = Array.new(@width * @height, 0)
  end
  
  def [](x : Int32, y : Int32) : Int32
    raise IndexError.new("Out of bounds") if x < 0 || x >= @width || y < 0 || y >= @height
    @data[y * @width + x]
  end
  
  def []=(x : Int32, y : Int32, value : Int32) : Int32
    raise IndexError.new("Out of bounds") if x < 0 || x >= @width || y < 0 || y >= @height
    @data[y * @width + x] = value
  end
  
  def to_s(io : IO) : Nil
    @height.times do |y|
      @width.times do |x|
        io << self[x, y].to_s.rjust(3)
      end
      io << "\n"
    end
  end
end

grid = Grid.new(3, 3)
grid[0, 0] = 1
grid[1, 1] = 5
grid[2, 2] = 9
puts grid
#   1  0  0
#   0  5  0
#   0  0  9
```

### def [] กับ Range

```crystal
class TextBuffer
  def initialize(@content : String)
  end
  
  def [](index : Int32) : Char
    @content[index]
  end
  
  def [](range : Range) : String
    @content[range]
  end
  
  def []=(index : Int32, char : Char)
    chars = @content.chars
    chars[index] = char
    @content = chars.join
  end
  
  def to_s(io : IO) : Nil
    io << @content
  end
  
  def size : Int32
    @content.size
  end
end

buf = TextBuffer.new("Hello, World!")
puts buf[0]      # => H
puts buf[7..]    # => World!
puts buf[0..4]   # => Hello
buf[0] = 'J'
puts buf         # => Jello, World!
```

---

## 42.5 Shift Operators

### def << และ def >>

```crystal
class Stack(T)
  def initialize
    @data = [] of T
  end
  
  # << สำหรับ push
  def <<(item : T) : self
    @data.push(item)
    self  # คืน self เพื่อ chaining
  end
  
  # >> สำหรับ pop ออกไปยัง variable
  def >>(target : Array(T)) : self
    target << @data.pop if !@data.empty?
    self
  end
  
  def pop : T?
    @data.pop?
  end
  
  def peek : T?
    @data.last?
  end
  
  def size : Int32
    @data.size
  end
  
  def empty? : Bool
    @data.empty?
  end
  
  def to_s(io : IO) : Nil
    io << "Stack#{@data}"
  end
end

stack = Stack(Int32).new
stack << 1 << 2 << 3 << 4 << 5

puts stack        # => Stack[1, 2, 3, 4, 5]
puts stack.peek   # => 5
puts stack.pop    # => 5
puts stack        # => Stack[1, 2, 3, 4]

output = [] of Int32
stack >> output >> output
puts output  # => [4, 3]
```

### ตัวอย่าง << สำหรับ Builder Pattern

```crystal
class HtmlBuilder
  def initialize
    @parts = [] of String
    @indent = 0
  end
  
  def <<(tag_content : {String, String}) : self
    tag, content = tag_content
    @parts << "#{"  " * @indent}<#{tag}>#{content}</#{tag}>"
    self
  end
  
  def <<(html : String) : self
    @parts << "#{"  " * @indent}#{html}"
    self
  end
  
  def open(tag : String) : self
    @parts << "#{"  " * @indent}<#{tag}>"
    @indent += 1
    self
  end
  
  def close(tag : String) : self
    @indent -= 1
    @parts << "#{"  " * @indent}</#{tag}>"
    self
  end
  
  def to_s : String
    @parts.join("\n")
  end
end

html = HtmlBuilder.new
html.open("div")
html << {"h1", "Title"}
html << {"p", "Some content"}
html.close("div")

puts html
```

---

## 42.6 Custom Operators

Crystal อนุญาตให้ใช้ symbols บางตัวเป็น custom operators

```crystal
class Set(T)
  def initialize
    @data = [] of T
  end
  
  def add(item : T) : self
    @data << item unless @data.includes?(item)
    self
  end
  
  # Union operator
  def |(other : Set(T)) : Set(T)
    result = Set(T).new
    @data.each { |item| result.add(item) }
    other.each { |item| result.add(item) }
    result
  end
  
  # Intersection operator
  def &(other : Set(T)) : Set(T)
    result = Set(T).new
    @data.each { |item| result.add(item) if other.includes?(item) }
    result
  end
  
  # Difference operator
  def -(other : Set(T)) : Set(T)
    result = Set(T).new
    @data.each { |item| result.add(item) unless other.includes?(item) }
    result
  end
  
  def includes?(item : T) : Bool
    @data.includes?(item)
  end
  
  def each(&block : T ->)
    @data.each(&block)
  end
  
  def size : Int32
    @data.size
  end
  
  def to_s(io : IO) : Nil
    io << "{"
    io << @data.join(", ")
    io << "}"
  end
end

a = Set(Int32).new
b = Set(Int32).new

[1, 2, 3, 4].each { |n| a.add(n) }
[3, 4, 5, 6].each { |n| b.add(n) }

puts a | b  # => {1, 2, 3, 4, 5, 6}  (union)
puts a & b  # => {3, 4}              (intersection)
puts a - b  # => {1, 2}              (difference)
```

---

## 42.7 Unary Operators

```crystal
class Complex
  getter real : Float64
  getter imaginary : Float64
  
  def initialize(@real : Float64, @imaginary : Float64 = 0.0)
  end
  
  # Unary minus
  def - : Complex
    Complex.new(-@real, -@imaginary)
  end
  
  # Unary plus (identity)
  def + : Complex
    self
  end
  
  # Bitwise NOT สำหรับ conjugate
  def ~ : Complex
    Complex.new(@real, -@imaginary)
  end
  
  def +(other : Complex) : Complex
    Complex.new(@real + other.real, @imaginary + other.imaginary)
  end
  
  def *(other : Complex) : Complex
    Complex.new(
      @real * other.real - @imaginary * other.imaginary,
      @real * other.imaginary + @imaginary * other.real
    )
  end
  
  def abs : Float64
    Math.sqrt(@real ** 2 + @imaginary ** 2)
  end
  
  def to_s(io : IO) : Nil
    if @imaginary >= 0
      io << "#{@real} + #{@imaginary}i"
    else
      io << "#{@real} - #{@imaginary.abs}i"
    end
  end
end

c1 = Complex.new(3.0, 4.0)
c2 = Complex.new(1.0, -2.0)

puts c1          # => 3.0 + 4.0i
puts -c1         # => -3.0 + -4.0i
puts ~c1         # => 3.0 - 4.0i  (conjugate)
puts c1 + c2     # => 4.0 + 2.0i
puts c1 * c2     # => 11.0 + -2.0i
puts c1.abs      # => 5.0
```

---

## 42.8 Coercion Methods

เราสามารถ override type coercion ได้

```crystal
class BigInteger
  def initialize(@value : Int64)
  end
  
  # Operator overloading กับ type coercion
  def +(other : Int32) : BigInteger
    BigInteger.new(@value + other.to_i64)
  end
  
  def +(other : BigInteger) : BigInteger
    BigInteger.new(@value + other.value)
  end
  
  def *(other : Int32) : BigInteger
    BigInteger.new(@value * other.to_i64)
  end
  
  def ==(other : BigInteger) : Bool
    @value == other.value
  end
  
  def ==(other : Int32 | Int64) : Bool
    @value == other.to_i64
  end
  
  # allow 5 + BigInteger.new(3) ด้วย
  def coerce(other : Int32) : {BigInteger, BigInteger}
    {BigInteger.new(other.to_i64), self}
  end
  
  protected getter value : Int64
  
  def to_s(io : IO) : Nil
    io << "BigInt(#{@value})"
  end
end

big = BigInteger.new(1_000_000_000_i64)
puts big + 5       # => BigInt(1000000005)
puts big * 2       # => BigInt(2000000000)
puts big == BigInteger.new(1_000_000_000_i64)  # => true
```

---

## 42.9 Bitwise Operators

```crystal
class Permissions
  NONE    = 0b0000
  READ    = 0b0001
  WRITE   = 0b0010
  EXECUTE = 0b0100
  ADMIN   = 0b1000
  
  def initialize(@value : Int32 = NONE)
  end
  
  # Bitwise OR (union)
  def |(other : Permissions) : Permissions
    Permissions.new(@value | other.value)
  end
  
  # Bitwise AND (intersection)
  def &(other : Permissions) : Permissions
    Permissions.new(@value & other.value)
  end
  
  # Bitwise XOR
  def ^(other : Permissions) : Permissions
    Permissions.new(@value ^ other.value)
  end
  
  # Bitwise NOT
  def ~ : Permissions
    Permissions.new(~@value & 0b1111)
  end
  
  # Shift left
  def <<(n : Int32) : Permissions
    Permissions.new(@value << n)
  end
  
  # Shift right
  def >>(n : Int32) : Permissions
    Permissions.new(@value >> n)
  end
  
  def has?(perm : Int32) : Bool
    (@value & perm) == perm
  end
  
  def ==(other : Permissions) : Bool
    @value == other.value
  end
  
  protected getter value : Int32
  
  def to_s(io : IO) : Nil
    perms = [] of String
    perms << "READ" if has?(READ)
    perms << "WRITE" if has?(WRITE)
    perms << "EXECUTE" if has?(EXECUTE)
    perms << "ADMIN" if has?(ADMIN)
    io << perms.empty? ? "NONE" : perms.join("|")
  end
end

user_perms = Permissions.new(Permissions::READ | Permissions::WRITE)
admin_perms = Permissions.new(Permissions::READ | Permissions::WRITE | Permissions::ADMIN)

puts user_perms   # => READ|WRITE
puts admin_perms  # => READ|WRITE|ADMIN

puts user_perms.has?(Permissions::READ)   # => true
puts user_perms.has?(Permissions::ADMIN)  # => false

combined = user_perms | Permissions.new(Permissions::EXECUTE)
puts combined  # => READ|WRITE|EXECUTE
```

---

## 42.10 String Concatenation Operator

```crystal
class StringBuilder
  def initialize
    @parts = [] of String
  end
  
  def <<(text : String) : self
    @parts << text
    self
  end
  
  def <<(num : Int32 | Float64) : self
    @parts << num.to_s
    self
  end
  
  def +(other : StringBuilder) : StringBuilder
    result = StringBuilder.new
    @parts.each { |p| result << p }
    other.parts.each { |p| result << p }
    result
  end
  
  def to_s : String
    @parts.join
  end
  
  protected getter parts : Array(String)
  
  def to_s(io : IO) : Nil
    io << to_s
  end
end

sb = StringBuilder.new
sb << "Hello" << ", " << "World" << "!" << " (version " << 2 << ")"
puts sb  # => Hello, World! (version 2)
```

---

## 42.11 ตัวอย่างในชีวิตจริง: Color Class

```crystal
struct Color
  getter r : UInt8
  getter g : UInt8
  getter b : UInt8
  getter a : UInt8
  
  def initialize(@r : UInt8, @g : UInt8, @b : UInt8, @a : UInt8 = 255_u8)
  end
  
  # Blend สองสี
  def +(other : Color) : Color
    Color.new(
      ((r.to_i + other.r.to_i) / 2).to_u8,
      ((g.to_i + other.g.to_i) / 2).to_u8,
      ((b.to_i + other.b.to_i) / 2).to_u8
    )
  end
  
  # Darken
  def *(factor : Float64) : Color
    Color.new(
      (r * factor).clamp(0.0, 255.0).to_u8,
      (g * factor).clamp(0.0, 255.0).to_u8,
      (b * factor).clamp(0.0, 255.0).to_u8,
      a
    )
  end
  
  def ==(other : Color) : Bool
    r == other.r && g == other.g && b == other.b && a == other.a
  end
  
  def to_hex : String
    "#%02X%02X%02X" % [r, g, b]
  end
  
  def to_s(io : IO) : Nil
    io << to_hex
  end
  
  # Pre-defined colors
  RED   = Color.new(255_u8, 0_u8, 0_u8)
  GREEN = Color.new(0_u8, 255_u8, 0_u8)
  BLUE  = Color.new(0_u8, 0_u8, 255_u8)
  WHITE = Color.new(255_u8, 255_u8, 255_u8)
  BLACK = Color.new(0_u8, 0_u8, 0_u8)
end

red = Color::RED
blue = Color::BLUE
purple = red + blue

puts red     # => #FF0000
puts blue    # => #0000FF
puts purple  # => #7F007F (ประมาณ)

dark_red = red * 0.5
puts dark_red  # => #7F0000

puts red == Color::RED  # => true
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Duration Class

สร้าง `Duration` class ที่แทนระยะเวลา พร้อม operator overloading:

```crystal
class Duration
  getter seconds : Int64
  
  def initialize(@seconds : Int64)
  end
  
  def self.from_minutes(m : Int32) : Duration
    new(m.to_i64 * 60)
  end
  
  def self.from_hours(h : Int32) : Duration
    new(h.to_i64 * 3600)
  end
  
  def +(other : Duration) : Duration
    Duration.new(@seconds + other.seconds)
  end
  
  def -(other : Duration) : Duration
    Duration.new(@seconds - other.seconds)
  end
  
  def *(factor : Int32) : Duration
    Duration.new(@seconds * factor)
  end
  
  def /(divisor : Int32) : Duration
    Duration.new(@seconds / divisor)
  end
  
  def <=>(other : Duration) : Int32
    @seconds <=> other.seconds
  end
  
  include Comparable(Duration)
  
  def to_s(io : IO) : Nil
    h = @seconds / 3600
    m = (@seconds % 3600) / 60
    s = @seconds % 60
    io << "%02d:%02d:%02d" % [h, m, s]
  end
end

d1 = Duration.from_hours(2)
d2 = Duration.from_minutes(30)
d3 = Duration.new(45)

total = d1 + d2 + d3
puts total  # => 02:30:45

puts d1 > d2  # => true
puts (d2 * 4).to_s  # => 02:00:00
```

### แบบฝึกหัดที่ 2: Polynomial Class

```crystal
class Polynomial
  def initialize(@coefficients : Array(Float64))
  end
  
  # บวกสอง polynomials
  def +(other : Polynomial) : Polynomial
    size = [coefficients.size, other.coefficients.size].max
    result = Array.new(size, 0.0)
    
    coefficients.each_with_index { |c, i| result[i] += c }
    other.coefficients.each_with_index { |c, i| result[i] += c }
    
    Polynomial.new(result)
  end
  
  # คูณด้วย scalar
  def *(scalar : Float64) : Polynomial
    Polynomial.new(coefficients.map { |c| c * scalar })
  end
  
  # ประเมินค่าที่ x
  def [](x : Float64) : Float64
    coefficients.each_with_index.sum { |c, i| c * (x ** i) }
  end
  
  protected getter coefficients : Array(Float64)
  
  def to_s(io : IO) : Nil
    terms = coefficients.each_with_index.map do |c, i|
      next if c == 0
      case i
      when 0 then c.to_s
      when 1 then "#{c}x"
      else "#{c}x^#{i}"
      end
    end.compact
    io << terms.reverse.join(" + ")
  end
end

p1 = Polynomial.new([1.0, 2.0, 3.0])  # 1 + 2x + 3x²
p2 = Polynomial.new([0.0, 1.0])        # x

sum = p1 + p2
puts p1       # => 3.0x^2 + 2.0x + 1.0
puts sum      # => 3.0x^2 + 3.0x + 1.0
puts p1[2.0]  # => 17.0 (1 + 4 + 12)
```

---

## สรุป

ใน Crystal เราสามารถ overload operators ได้หลากหลาย:

| Operator | ชื่อ Method | ตัวอย่าง |
|----------|------------|---------|
| `+` | `def +` | `a + b` |
| `-` | `def -` | `a - b` |
| `*` | `def *` | `a * b` |
| `/` | `def /` | `a / b` |
| `%` | `def %` | `a % b` |
| `**` | `def **` | `a ** b` |
| `==` | `def ==` | `a == b` |
| `<=>` | `def <=>` | `a <=> b` |
| `[]` | `def []` | `a[i]` |
| `[]=` | `def []=` | `a[i] = v` |
| `<<` | `def <<` | `a << b` |
| `>>` | `def >>` | `a >> b` |
| `&` | `def &` | `a & b` |
| `\|` | `def \|` | `a \| b` |
| `^` | `def ^` | `a ^ b` |
| `~` | `def ~` | `~a` |

Operator overloading ทำให้ code อ่านง่ายขึ้นและทำงานกับ custom types ได้เหมือน built-in types

---

*ต่อไป: Part 43 - Property Accessors*
