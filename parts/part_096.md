# Part 96: Casting และ Type Checking ใน Crystal

## บทนำ

Crystal มีเครื่องมือหลายอย่างสำหรับ type checking และ casting ซึ่งทำงานทั้ง compile time และ runtime

## as - Type Cast

```crystal
# as - cast type (raise ถ้าผิด type)
value : Int32 | String = 42

int_val = value.as(Int32)
puts int_val + 1  # => 43

# จะ raise TypeCastError ถ้า cast ผิด
begin
  str_val = value.as(String)  # Error ณ runtime
rescue TypeCastError => e
  puts "Cast failed: #{e.message}"
end

# as กับ nil
nilable : String? = "hello"
non_nil = nilable.as(String)  # ok ถ้า nilable ไม่ nil
puts non_nil.upcase  # => HELLO

nil_value : String? = nil
begin
  not_nil = nil_value.as(String)  # TypeCastError
rescue TypeCastError => e
  puts "Cannot cast nil to String"
end
```

## as? - Safe Type Cast

```crystal
# as? - safe cast (return nil ถ้าผิด type)
value : Int32 | String | Float64 = "hello"

int_result = value.as?(Int32)      # => nil
str_result = value.as?(String)     # => "hello"
float_result = value.as?(Float64)  # => nil

puts int_result.inspect    # => nil
puts str_result.inspect    # => "hello"
puts float_result.inspect  # => nil

# Pattern: as? + nil handling
if i = value.as?(Int32)
  puts "Is Int32: #{i}"
elsif s = value.as?(String)
  puts "Is String: #{s}"
end

# Safe array processing
mixed = [1, "hello", 2, "world", 3, 4.5] of Int32 | String | Float64
strings = mixed.compact_map(&.as?(String))
puts strings.inspect  # => ["hello", "world"]

ints = mixed.compact_map(&.as?(Int32))
puts ints.inspect  # => [1, 2, 3]
```

## is_a? - Type Check

```crystal
# is_a? - check type ณ runtime
def describe_type(value)
  if value.is_a?(Int32)
    puts "Integer: #{value}"
  elsif value.is_a?(Float64)
    puts "Float: #{value}"
  elsif value.is_a?(String)
    puts "String: #{value}"
  elsif value.is_a?(Bool)
    puts "Bool: #{value}"
  elsif value.is_a?(Nil)
    puts "Nil"
  else
    puts "Unknown type: #{value.class}"
  end
end

describe_type(42)
describe_type(3.14)
describe_type("hello")
describe_type(true)
describe_type(nil)

# is_a? ใน case statement
def process(value : Int32 | String | Array(Int32))
  case value
  when Int32        then puts "Int: #{value * 2}"
  when String       then puts "Str: #{value.upcase}"
  when Array(Int32) then puts "Arr: #{value.sum}"
  end
end

process(5)
process("crystal")
process([1, 2, 3, 4, 5])
```

## responds_to? - Duck Typing

```crystal
# responds_to? - check ว่า object มี method
def safe_to_string(obj) : String
  if obj.responds_to?(:to_s)
    obj.to_s
  else
    "(no to_s)"
  end
end

def display_size(obj) : String
  if obj.responds_to?(:size)
    "Size: #{obj.size}"
  elsif obj.responds_to?(:length)
    "Length: #{obj.length}"
  else
    "Cannot determine size"
  end
end

puts display_size("hello")       # => Size: 5
puts display_size([1, 2, 3])     # => Size: 3
puts display_size({a: 1, b: 2}) # => Size: 2
puts display_size(42)            # => Cannot determine size

# ใช้ใน generic programming
def sort_if_possible(collection) : Array
  if collection.responds_to?(:sort)
    collection.sort
  else
    collection.to_a
  end
end
```

## typeof - Compile-time Type

```crystal
# typeof - return type ณ compile time (ไม่ใช่ runtime!)
x = 42
puts typeof(x)      # => Int32

y = "hello"
puts typeof(y)      # => String

z : Int32 | String = 42
puts typeof(z)      # => Int32 | String (union type)

# typeof ใน generic context
def print_type(x : T) forall T
  puts "#{x.inspect} is #{typeof(x)}"
end

print_type(42)       # => 42 is Int32
print_type("hello")  # => "hello" is String
print_type(3.14)     # => 3.14 is Float64

# typeof กับ array
arr = [1, "hello", 3.14]
puts typeof(arr)     # => Array(Int32 | String | Float64)

# ใช้เพื่อ infer types
def zero_of(x : T) : T forall T
  typeof(x).zero
end

puts zero_of(42)    # => 0
puts zero_of(3.14)  # => 0.0
```

## instance_sizeof

```crystal
# instance_sizeof - ขนาดของ instance ในหน่วย bytes
class Empty
end

class WithInt
  @x : Int32 = 0
end

class WithInts
  @x : Int32 = 0
  @y : Int32 = 0
  @z : Int32 = 0
end

struct StructWithInts
  @x : Int32
  @y : Int32
  @z : Int32

  def initialize(@x, @y, @z)
  end
end

puts instance_sizeof(Empty)          # => 8 (minimum object size)
puts instance_sizeof(WithInt)        # => 16 (header + Int32 + padding)
puts instance_sizeof(WithInts)       # => 24 (header + 3 * Int32 + padding)
puts instance_sizeof(StructWithInts) # => 12 (3 * 4 bytes)

# Useful สำหรับ optimization
puts "Int32: #{sizeof(Int32)} bytes"
puts "Int64: #{sizeof(Int64)} bytes"
puts "Float64: #{sizeof(Float64)} bytes"
puts "Bool: #{sizeof(Bool)} byte"
puts "Char: #{sizeof(Char)} bytes"
```

## sizeof

```crystal
# sizeof - ขนาดของ type ในหน่วย bytes
puts sizeof(Int8)    # => 1
puts sizeof(Int16)   # => 2
puts sizeof(Int32)   # => 4
puts sizeof(Int64)   # => 8
puts sizeof(UInt8)   # => 1
puts sizeof(Float32) # => 4
puts sizeof(Float64) # => 8
puts sizeof(Bool)    # => 1
puts sizeof(Char)    # => 4
puts sizeof(Pointer(Int32))  # => 8 (64-bit system)

# sizeof struct
struct Compact
  @a : UInt8
  @b : UInt8
  @c : UInt8

  def initialize(@a, @b, @c)
  end
end

struct WithPadding
  @a : UInt8
  @b : Int32  # ต้อง align ที่ 4 bytes
  @c : UInt8

  def initialize(@a, @b, @c)
  end
end

puts sizeof(Compact)      # => 3
puts sizeof(WithPadding)  # => 12 (due to alignment)
```

## offsetof

```crystal
# offsetof - offset ของ field ใน struct
struct Point3D
  @x : Float64
  @y : Float64
  @z : Float64

  def initialize(@x, @y, @z)
  end
end

puts offsetof(Point3D, @x)  # => 0
puts offsetof(Point3D, @y)  # => 8
puts offsetof(Point3D, @z)  # => 16

struct Mixed
  @flag : Bool    # 1 byte
  @value : Int32  # 4 bytes (aligned)
  @data : Int64   # 8 bytes
end

puts offsetof(Mixed, @flag)   # => 0
puts offsetof(Mixed, @value)  # => 4 (aligned)
puts offsetof(Mixed, @data)   # => 8

# Useful สำหรับ low-level programming และ C interop
```

## Type Narrowing

```crystal
# Compiler narrowing - type scope ลดลงหลัง check
value : Int32 | String | Nil = get_some_value  # hypothetical

# หลัง nil check
if value
  # type เป็น Int32 | String ที่นี่ (nil excluded)
  puts value.class
end

# หลัง type check
if value.is_a?(Int32)
  # type เป็น Int32 ที่นี่
  puts value + 1
end

# case statement narrowing
case value
when Int32
  puts "Int32: #{value * 2}"     # value เป็น Int32
when String
  puts "String: #{value.upcase}" # value เป็น String
when Nil
  puts "Nil"
end

def get_some_value : Int32 | String | Nil
  rand(3) == 0 ? nil : (rand(2) == 0 ? 42 : "hello")
end
```

## Type Introspection

```crystal
# Introspect types ณ runtime
def type_info(value)
  puts "Class: #{value.class}"
  puts "Type: #{typeof(value)}"
  puts "Is Int: #{value.is_a?(Int32)}"
  puts "Is Num: #{value.is_a?(Number)}"
  puts "Is Comparable: #{value.is_a?(Comparable)}"
  puts "Responds to +: #{value.responds_to?(:+)}"
  puts "Responds to size: #{value.responds_to?(:size)}"
  puts "---"
end

type_info(42)
type_info("hello")
type_info([1, 2, 3])

# Class hierarchy check
class Animal; end
class Dog < Animal; end
class Cat < Animal; end

dog = Dog.new
puts dog.is_a?(Dog)     # => true
puts dog.is_a?(Animal)  # => true
puts dog.is_a?(Cat)     # => false
puts dog.class          # => Dog
puts Dog.superclass     # => Animal
puts Dog.ancestors.inspect  # => [Dog, Animal, Reference, Object]
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Type-Safe Deserializer
สร้าง deserializer ที่:
- รับ `JSON::Any`
- แปลงเป็น strongly-typed structs
- ใช้ `as?`, `is_a?` อย่างถูกต้อง
- Error messages ที่ชัดเจน

### แบบฝึกหัดที่ 2: Memory Layout Analyzer
สร้าง tool ที่แสดง:
- `sizeof` ของแต่ละ type
- `offsetof` ของแต่ละ field ใน struct
- Memory padding calculation
- Compare layouts

### แบบฝึกหัดที่ 3: Dynamic Dispatch
ใช้ `responds_to?` สร้าง duck typing system:
- Interface ที่ไม่ต้องการ explicit include
- Safe method calling
- Fallback behavior

### แบบฝึกหัดที่ 4: Type Registry
สร้าง type registry ที่:
- ลงทะเบียน type handlers
- Dispatch based on `is_a?`
- Support inheritance chains
- Thread-safe

## สรุป

Type operations ใน Crystal:
- **as**: unsafe cast, raise ถ้าผิด type
- **as?**: safe cast, return nil ถ้าผิด type
- **is_a?**: type check, ทำงานทั้ง runtime และ narrow type
- **responds_to?**: duck typing check
- **typeof**: compile-time type (ไม่ใช่ runtime value)
- **sizeof/instance_sizeof**: memory sizes
- **offsetof**: field offsets ใน structs

Best practices:
1. ใช้ `as?` แทน `as` เมื่อไม่แน่ใจ type
2. ใช้ `case` เมื่อมีหลาย type ต้อง check
3. `typeof` เหมาะสำหรับ debugging type inference
4. `sizeof` มีประโยชน์สำหรับ performance optimization
