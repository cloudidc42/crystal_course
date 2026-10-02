# Part 007: ระบบ Type และ Type Inference

## Crystal Type System Overview

Crystal ใช้ **Static Type System** ที่ทรงพลัง แต่มี **Type Inference** ที่ทำให้ไม่ต้องระบุ type ตลอดเวลา

```crystal
# Type inference - Crystal รู้ type โดยอัตโนมัติ
x = 42          # Crystal รู้ว่าเป็น Int32
name = "Alice"  # Crystal รู้ว่าเป็น String
pi = 3.14       # Crystal รู้ว่าเป็น Float64

# ระบุ type ชัดเจนก็ได้
y : Int32 = 42
z : String = "hello"

# TypeOf - ดู type ณ compile time
puts typeof(x)     # Int32
puts typeof(name)  # String
puts typeof(pi)    # Float64

# x.class - ดู type ณ runtime
puts x.class     # Int32
puts name.class  # String
```

## Primitive Types

```crystal
# Integer types
i8  : Int8   = 127
i16 : Int16  = 32767
i32 : Int32  = 2147483647   # default integer
i64 : Int64  = 9223372036854775807
i128: Int128 = big_number

u8  : UInt8  = 255
u16 : UInt16 = 65535
u32 : UInt32 = 4294967295
u64 : UInt64 = 18446744073709551615

# Float types
f32 : Float32 = 3.14_f32
f64 : Float64 = 3.14         # default float

# Boolean
b : Bool = true

# Character
c : Char = 'A'

# String
s : String = "hello"

# Symbol
sym : Symbol = :hello

# Nil
n : Nil = nil

# Type hierarchy
# Number
#   Int
#     Int8, Int16, Int32, Int64, Int128
#     UInt8, UInt16, UInt32, UInt64, UInt128
#   Float
#     Float32, Float64
```

## Union Types

```crystal
# Union Type - ค่าอาจเป็น type ใดก็ได้ใน union
value : Int32 | String = 42
value = "hello"  # OK - still Int32 | String

# Nil Union (Nullable)
name : String | Nil = nil
name : String? = nil  # Shorthand for String | Nil

# Multiple types in union
result : Int32 | Float64 | String | Nil = nil

# Working with union types
def process(value : Int32 | String) : String
  case value
  when Int32
    "Integer: #{value}"
  when String
    "String: #{value}"
  end
end

puts process(42)      # "Integer: 42"
puts process("hello") # "String: hello"

# Type narrowing in conditions
def safe_process(value : Int32 | String | Nil)
  if value.nil?
    "nil value"
  elsif value.is_a?(Int32)
    "Int: #{value + 1}"  # Crystal knows it's Int32 here
  else
    "String: #{value.upcase}"  # Crystal knows it's String here
  end
end

# Union type inference
values = [1, "two", 3.0]  # Array(Int32 | String | Float64)
puts typeof(values)         # Array(Int32 | String | Float64)
```

## Type Inference in Detail

```crystal
# Method return type inference
def add(a, b)
  a + b
end

# Crystal infers return type from usage
x = add(1, 2)      # x is Int32
y = add(1.0, 2.0)  # y is Float64
z = add("a", "b")  # z is String

puts typeof(x)  # Int32
puts typeof(y)  # Float64
puts typeof(z)  # String

# Block return type inference
nums = [1, 2, 3]
doubled = nums.map { |n| n * 2 }
# Crystal infers: Array(Int32)

strings = nums.map { |n| n.to_s }
# Crystal infers: Array(String)

mixed = nums.map { |n| n.odd? ? n : n.to_s }
# Crystal infers: Array(Int32 | String)

# Variable type changes
x = 1        # x: Int32
x = "hello"  # ERROR! Cannot change type of existing variable
              # Crystal is statically typed!

# But union types work
y : Int32 | String = 1
y = "hello"  # OK - still Int32 | String

# Anonymous union
z = rand > 0.5 ? 1 : "one"
puts typeof(z)  # Int32 | String
```

## Type Restrictions

```crystal
# Restrict method parameters to specific types
def double(n : Int32) : Int32
  n * 2
end

def greet(name : String) : String
  "Hello, #{name}!"
end

puts double(5)      # 10
puts double(5.0)    # Error! No overload for Float64

# Multiple overloads
def process(n : Int32) : String
  "Int: #{n}"
end

def process(s : String) : String
  "String: #{s}"
end

def process(f : Float64) : String
  "Float: #{f}"
end

puts process(42)      # "Int: 42"
puts process("hello") # "String: hello"
puts process(3.14)    # "Float: 3.14"

# Abstract type restrictions
def sum_all(numbers : Array(Number)) : Float64
  numbers.sum.to_f
end

puts sum_all([1, 2, 3])        # 6.0
puts sum_all([1.5, 2.5, 3.0])  # 7.0
puts sum_all([1, 2.5, 3])      # 6.5

# Using type alias in restrictions
alias NumberOrString = Int32 | Float64 | String

def display(value : NumberOrString)
  puts value.to_s
end
```

## Type Aliases

```crystal
# Simple alias
alias Filename = String
alias UserId = Int32
alias UserMap = Hash(UserId, String)

file : Filename = "hello.txt"
id : UserId = 42
users : UserMap = {1 => "Alice", 2 => "Bob"}

# Complex aliases
alias JsonValue = Int32 | Float64 | String | Bool | Nil | Array(JsonValue) | Hash(String, JsonValue)
alias Callback = Proc(String, Nil)
alias Transform(T, U) = Proc(T, U)

# Using aliases
transform : Transform(Int32, String) = ->(n : Int32) { n.to_s }
puts transform.call(42)  # "42"

# Useful for readability
alias RGB = {r: UInt8, g: UInt8, b: UInt8}
red : RGB = {r: 255_u8, g: 0_u8, b: 0_u8}
```

## Type Checking

```crystal
# is_a? - check type
x = 42
puts x.is_a?(Int32)   # true
puts x.is_a?(Int64)   # false
puts x.is_a?(Number)  # true (Int32 is a subtype of Number)

# nil? - check nil
puts nil.nil?    # true
puts 42.nil?     # false

# responds_to? - duck typing
puts 42.responds_to?(:to_s)    # true
puts "hi".responds_to?(:upcase) # true
puts 42.responds_to?(:upcase)  # false

# class - get runtime class
puts 42.class         # Int32
puts "hi".class       # String
puts [1,2].class      # Array(Int32)
puts nil.class        # Nil

# typeof - compile time type
x = 42
puts typeof(x)        # Int32

# Case/when with types
def describe(value)
  case value
  when Int32
    "an integer"
  when Float64
    "a float"
  when String
    "a string"
  when Array
    "an array of #{value.size} elements"
  when Nil
    "nil"
  else
    "something else (#{typeof(value)})"
  end
end

puts describe(42)          # "an integer"
puts describe(3.14)        # "a float"
puts describe("hello")     # "a string"
puts describe([1,2,3])     # "an array of 3 elements"
puts describe(nil)         # "nil"
```

## Type Casting

```crystal
# as - unsafe cast (raises TypeCastError if wrong)
obj : Int32 | String = 42
num = obj.as(Int32)    # OK
# str = obj.as(String) # Raises TypeCastError!

# as? - safe cast (returns nil if wrong)
num2 = obj.as?(Int32)   # 42
str = obj.as?(String)   # nil

# to_* - conversion methods
puts 42.to_f        # 42.0
puts 42.to_s        # "42"
puts 42.to_i64      # 42_i64
puts 3.14.to_i      # 3 (truncated)
puts "42".to_i      # 42
puts "3.14".to_f    # 3.14

# Safe conversion with ?
puts "abc".to_i?    # nil
puts "42".to_i?     # 42
puts "3.x".to_f?    # nil
puts "3.14".to_f?   # 3.14

# Numeric widening (automatic in expressions)
i32 : Int32 = 5
i64 : Int64 = 10_i64
# result : Int64 = i32 + i64  # Error! Must be same type
result : Int64 = i32.to_i64 + i64  # OK
```

## Generics

```crystal
# Generic class
class Box(T)
  getter value : T
  
  def initialize(@value : T)
  end
  
  def map(&block : T -> U) : Box(U) forall U
    Box(U).new(block.call(@value))
  end
end

int_box = Box(Int32).new(42)
str_box = Box(String).new("hello")

puts int_box.value  # 42
puts str_box.value  # "hello"

# Type inference with generics
box = Box.new(42)      # Box(Int32) inferred
box2 = Box.new("hi")   # Box(String) inferred

# Generic method
def identity(x : T) : T forall T
  x
end

puts identity(42)      # 42
puts identity("hello") # "hello"
puts identity(3.14)    # 3.14

# Generic with multiple type params
class Pair(A, B)
  getter first : A
  getter second : B
  
  def initialize(@first : A, @second : B)
  end
  
  def swap : Pair(B, A)
    Pair(B, A).new(@second, @first)
  end
end

pair = Pair.new("hello", 42)
puts pair.first    # hello
puts pair.second   # 42

swapped = pair.swap
puts swapped.first   # 42
puts swapped.second  # hello
```

## Struct vs Class Types

```crystal
# Struct - value type (stack allocated, copied)
struct Point
  property x : Float64
  property y : Float64
  
  def initialize(@x : Float64, @y : Float64)
  end
  
  def distance_to(other : Point) : Float64
    Math.sqrt((other.x - @x) ** 2 + (other.y - @y) ** 2)
  end
end

# Class - reference type (heap allocated)
class Circle
  property center : Point
  property radius : Float64
  
  def initialize(@center : Point, @radius : Float64)
  end
  
  def area : Float64
    Math::PI * @radius ** 2
  end
  
  def circumference : Float64
    2 * Math::PI * @radius
  end
end

# Struct is copied on assignment
p1 = Point.new(0.0, 0.0)
p2 = p1  # p2 is a COPY of p1
p2.x = 5.0
puts p1.x  # 0.0 (unchanged - struct is value type)
puts p2.x  # 5.0

# Class is referenced
c1 = Circle.new(p1, 5.0)
c2 = c1  # c2 REFERS to same object
c2.radius = 10.0
puts c1.radius  # 10.0 (changed! - class is reference type)
```

## Enum Type

```crystal
# Basic enum
enum Direction
  North
  South
  East
  West
end

dir = Direction::North
puts dir        # North
puts dir.value  # 0 (default: starts at 0)

# Enum with custom values
enum Color
  Red = 0xFF0000
  Green = 0x00FF00
  Blue = 0x0000FF
end

puts Color::Red.value    # 16711680 (0xFF0000)
puts Color::Green.value  # 65280 (0x00FF00)

# Flags enum (bitflags)
@[Flags]
enum Permission
  Read    # 1
  Write   # 2
  Execute # 4
end

perms = Permission::Read | Permission::Write
puts perms.includes?(Permission::Read)     # true
puts perms.includes?(Permission::Execute)  # false

# Enum methods
Direction.each { |d| puts d }
puts Direction.names    # ["North", "South", "East", "West"]
puts Direction.values   # [0, 1, 2, 3]
puts Direction.from_value(0)  # North
```

## Proc and Lambda Types

```crystal
# Proc type
add : Proc(Int32, Int32, Int32) = ->(a : Int32, b : Int32) { a + b }
puts add.call(1, 2)  # 3

# Lambda (same as Proc but strict with arity)
multiply = ->(a : Int32, b : Int32) { a * b }
puts multiply.call(3, 4)  # 12

# Proc as parameter
def apply(n : Int32, f : Proc(Int32, Int32)) : Int32
  f.call(n)
end

double = ->(n : Int32) { n * 2 }
puts apply(5, double)  # 10

# Block to Proc
def transform(arr : Array(Int32), &block : Int32 -> Int32) : Array(Int32)
  arr.map { |n| block.call(n) }
end

puts transform([1, 2, 3]) { |n| n * n }  # [1, 4, 9]

# Method reference
puts [1, 2, 3].map(&.to_s)   # ["1", "2", "3"]
puts ["a", "b"].map(&.upcase) # ["A", "B"]
```

## Type-safe Collections

```crystal
# Typed arrays
ints : Array(Int32) = [1, 2, 3]
strings : Array(String) = ["a", "b", "c"]

# Inferred
nums = [1, 2, 3]          # Array(Int32)
names = ["Alice", "Bob"]  # Array(String)

# Mixed types create union
mixed = [1, "two", 3.0]   # Array(Int32 | String | Float64)

# Typed hash
scores : Hash(String, Int32) = {"Alice" => 95, "Bob" => 87}
config : Hash(Symbol, String) = {host: "localhost", port: "8080"}

# Typed tuple
point : Tuple(Int32, Int32) = {0, 0}
rgb : Tuple(UInt8, UInt8, UInt8) = {255_u8, 128_u8, 0_u8}

# Named tuple
user : NamedTuple(name: String, age: Int32) = {name: "Alice", age: 25}
puts user[:name]  # Alice
puts user[:age]   # 25
```

## Practical: Type-Safe Configuration

```crystal
# Type-safe configuration system

struct DatabaseConfig
  getter host : String
  getter port : Int32
  getter name : String
  getter username : String
  getter password : String
  getter pool_size : Int32
  getter timeout : Float64
  
  def initialize(
    @host : String = "localhost",
    @port : Int32 = 5432,
    @name : String = "mydb",
    @username : String = "postgres",
    @password : String = "",
    @pool_size : Int32 = 5,
    @timeout : Float64 = 30.0
  )
  end
  
  def connection_string : String
    "postgresql://#{@username}:#{@password}@#{@host}:#{@port}/#{@name}"
  end
end

struct AppConfig
  getter database : DatabaseConfig
  getter debug : Bool
  getter port : Int32
  getter allowed_origins : Array(String)
  
  def initialize(
    @database : DatabaseConfig = DatabaseConfig.new,
    @debug : Bool = false,
    @port : Int32 = 8080,
    @allowed_origins : Array(String) = ["*"]
  )
  end
end

# Usage
config = AppConfig.new(
  database: DatabaseConfig.new(
    host: "db.example.com",
    name: "production_db",
    username: "app_user",
    password: "secret"
  ),
  debug: false,
  port: 443
)

puts config.database.connection_string
puts "Port: #{config.port}"
puts "Debug: #{config.debug}"
```

---

## สรุป Part 007

ในบทนี้เราได้เรียนรู้:

1. **Type System**: Crystal ใช้ static typing with type inference
2. **Primitive Types**: Int, Float, Bool, Char, String, Symbol, Nil
3. **Union Types**: Int32 | String, String? (nullable)
4. **Type Inference**: compiler อนุมาน type โดยอัตโนมัติ
5. **Type Restrictions**: จำกัด parameter types ใน methods
6. **Type Aliases**: alias สำหรับ readability
7. **Type Checking**: is_a?, nil?, responds_to?, class, typeof
8. **Type Casting**: as, as?, to_* methods
9. **Generics**: Box(T), Pair(A, B)
10. **Struct vs Class**: value type vs reference type

---

## ขั้นตอนต่อไป

ไปที่ [Part 008](part_008.md) เพื่อเรียนรู้:
- Constants และ Literals
- Symbol literals
- Number literals
- String literals ขั้นสูง
