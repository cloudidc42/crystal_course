# Part 93: Type Restrictions ใน Crystal

## บทนำ

Type restrictions ใน Crystal ช่วยให้เราระบุ type ที่ method รับได้ เป็นกลไกหลักในการสร้าง type-safe APIs

## Type Restriction พื้นฐาน

```crystal
# ระบุ type ที่ parameter รับได้
def greet(name : String)
  puts "สวัสดี #{name}!"
end

def add(a : Int32, b : Int32) : Int32
  a + b
end

def sqrt(x : Float64) : Float64
  Math.sqrt(x)
end

greet("สมชาย")
puts add(3, 4)   # => 7
puts sqrt(16.0)  # => 4.0

# จะ compile error ถ้าส่ง wrong type:
# add(3.5, 2.0)  # Error: no overload matches...
```

## Method Overloading

```crystal
# หลาย overloads สำหรับ types ต่างๆ
def double(x : Int32) : Int32
  x * 2
end

def double(x : Float64) : Float64
  x * 2.0
end

def double(x : String) : String
  x + x
end

def double(x : Array) : Array
  x + x
end

puts double(5)       # => 10 (Int32 version)
puts double(3.14)    # => 6.28 (Float64 version)
puts double("Hi")    # => HiHi (String version)
puts double([1,2,3]) # => [1, 2, 3, 1, 2, 3] (Array version)
```

## Number Type Hierarchy

```crystal
# Number รองรับ type hierarchy ของ Crystal
def sum_numbers(a : Number, b : Number)
  a + b
end

puts sum_numbers(1, 2)         # => 3 (Int32)
puts sum_numbers(1.5, 2.5)     # => 4.0 (Float64)
puts sum_numbers(1_i64, 2_i64) # => 3 (Int64)
puts sum_numbers(1, 2.5)       # => 3.5 (mixed)

# Comparable
def max_value(a : T, b : T) : T forall T
  a > b ? a : b
end

puts max_value(3, 7)       # => 7
puts max_value(3.14, 2.71) # => 3.14
puts max_value("abc", "xyz") # => xyz
```

## Multiple Type Restrictions (Union)

```crystal
# Parameter ที่รับได้หลาย types
def process(value : String | Int32 | Float64)
  case value
  when String  then "String: #{value.upcase}"
  when Int32   then "Int32: #{value * 2}"
  when Float64 then "Float64: #{value.round(2)}"
  end
end

puts process("hello")  # => String: HELLO
puts process(42)       # => Int32: 84
puts process(3.14159)  # => Float64: 3.14

# Nilable parameter
def find_user(id : Int32 | Nil) : String
  if id
    "User ##{id}"
  else
    "Anonymous"
  end
end

puts find_user(42)   # => User #42
puts find_user(nil)  # => Anonymous
```

## Comparable Constraint

```crystal
# ใช้ Comparable เป็น constraint
def sort_pair(a : T, b : T) : Tuple(T, T) forall T
  a <= b ? {a, b} : {b, a}
end

puts sort_pair(5, 3).inspect    # => {3, 5}
puts sort_pair("b", "a").inspect # => {"a", "b"}

# Clamp value
def clamp(value : T, min : T, max : T) : T forall T
  if value < min
    min
  elsif value > max
    max
  else
    value
  end
end

puts clamp(5, 0, 10)    # => 5
puts clamp(-5, 0, 10)   # => 0
puts clamp(15, 0, 10)   # => 10
puts clamp(3.5, 0.0, 5.0) # => 3.5
```

## Enumerable Constraint

```crystal
# รับ Enumerable types
def sum_all(collection : Enumerable(T)) : T forall T
  collection.reduce { |acc, val| acc + val }
end

puts sum_all([1, 2, 3, 4, 5])         # => 15
puts sum_all({1.0, 2.0, 3.0})         # => 6.0 (Tuple)
puts sum_all(Set{1, 2, 3, 4, 5})      # => 15

def count_items(collection : Enumerable) : Int32
  count = 0
  collection.each { count += 1 }
  count
end

puts count_items([1, 2, 3])    # => 3
puts count_items({"a", "b"})   # => 2
```

## NoReturn Type

```crystal
# NoReturn - function ที่ไม่ return กลับ
def fatal_error(message : String) : NoReturn
  puts "FATAL: #{message}"
  exit(1)
end

def divide(a : Int32, b : Int32) : Int32
  if b == 0
    fatal_error("Division by zero!")  # NoReturn - compiler รู้ว่าไม่ต้อง handle return
  end
  a / b
end

# raise ก็มี type NoReturn
def always_raises : NoReturn
  raise "Always raises"
end

# NoReturn ใน abstract methods
abstract class AbstractProcessor
  abstract def process(data : String) : String
  abstract def handle_error(error : Exception) : NoReturn
end

class StrictProcessor < AbstractProcessor
  def process(data : String) : String
    data.upcase
  end

  def handle_error(error : Exception) : NoReturn
    STDERR.puts "Critical error: #{error.message}"
    exit(1)
  end
end
```

## Type Restriction กับ Blocks

```crystal
# กำหนด type ของ block
def transform(value : Int32, &block : Int32 -> String) : String
  block.call(value)
end

result = transform(42) { |x| "Value is: #{x}" }
puts result  # => Value is: 42

# Block ที่รับหลาย parameters
def pair_transform(a : Int32, b : Int32, &block : Int32, Int32 -> String) : String
  block.call(a, b)
end

result2 = pair_transform(3, 4) { |x, y| "#{x} + #{y} = #{x + y}" }
puts result2  # => 3 + 4 = 7

# Block ที่ return Bool (predicate)
def filter_with(arr : Array(T), &pred : T -> Bool) : Array(T) forall T
  arr.select { |x| pred.call(x) }
end

evens = filter_with([1, 2, 3, 4, 5, 6]) { |x| x.even? }
puts evens.inspect  # => [2, 4, 6]
```

## Self Type Restriction

```crystal
# self เป็น type restriction สำหรับ fluent interface
class Builder
  @parts = Array(String).new

  def add(part : String) : self
    @parts << part
    self
  end

  def build : String
    @parts.join(", ")
  end
end

class EnhancedBuilder < Builder
  def add_with_prefix(prefix : String, part : String) : self
    add("#{prefix}: #{part}")
  end
end

# self ทำให้ chain ทำงานถูก type
result = EnhancedBuilder.new
  .add("first")
  .add_with_prefix("important", "second")
  .add("third")
  .build

puts result
```

## Class Type Restriction

```crystal
# .class type restriction
def create_instance(klass : Int32.class) : Int32
  42
end

def create_instance(klass : String.class) : String
  "hello"
end

# Generic version
def factory(klass : T.class, value : T) : T forall T
  value
end

puts factory(Int32, 42)     # => 42
puts factory(String, "hi")  # => hi

# ใช้เพื่อ pass type ไปยัง function
def zero_value_of(klass : Int32.class) : Int32
  0
end

def zero_value_of(klass : Float64.class) : Float64
  0.0
end

def zero_value_of(klass : String.class) : String
  ""
end

def zero_value_of(klass : Bool.class) : Bool
  false
end

puts zero_value_of(Int32)   # => 0
puts zero_value_of(Float64) # => 0.0
puts zero_value_of(String)  # => ""
puts zero_value_of(Bool)    # => false
```

## Overloading กับ Union Return Types

```crystal
# Method ที่ return type ขึ้นอยู่กับ input
def parse(value : String) : Int32 | Float64 | String | Bool | Nil
  case value
  when "true"   then true
  when "false"  then false
  when "null", "nil" then nil
  else
    # Try Int32
    if i = value.to_i?
      i
    # Try Float64
    elsif f = value.to_f?
      f
    else
      value
    end
  end
end

values = ["42", "3.14", "true", "hello", "null", "-5"]
values.each do |v|
  result = parse(v)
  puts "#{v.inspect} => #{result.inspect} (#{result.class})"
end
```

## Proc Type Restrictions

```crystal
# กำหนด type ของ Proc parameter
alias Transformer(A, B) = Proc(A, B)
alias Predicate(T) = Proc(T, Bool)
alias Consumer(T) = Proc(T, Nil)

def apply_transformer(value : A, transformer : Transformer(A, B)) : B forall A, B
  transformer.call(value)
end

def filter_list(list : Array(T), pred : Predicate(T)) : Array(T) forall T
  list.select { |x| pred.call(x) }
end

# ใช้งาน
to_string = Transformer(Int32, String).new { |x| x.to_s }
puts apply_transformer(42, to_string)  # => "42"

is_even = Predicate(Int32).new { |x| x.even? }
puts filter_list([1, 2, 3, 4, 5, 6], is_even).inspect  # => [2, 4, 6]
```

## Responds To Restriction

```crystal
# responds_to? check ใน type narrowing
def display(item)
  if item.responds_to?(:to_s)
    puts item.to_s
  else
    puts "(no to_s method)"
  end
end

# respond_to constraint ใน generic methods
def print_all(items : Array(T)) forall T
  items.each do |item|
    if item.responds_to?(:name)
      puts "Name: #{item.name}"
    elsif item.responds_to?(:to_s)
      puts item.to_s
    end
  end
end

struct Named
  getter name : String
  def initialize(@name); end
end

print_all([Named.new("Alice"), Named.new("Bob")])
print_all([1, 2, 3])
```

## Type Restriction กับ Splat

```crystal
# Splat args กับ type restrictions
def sum(*numbers : Int32) : Int32
  numbers.sum
end

def join(*strings : String) : String
  strings.join(", ")
end

puts sum(1, 2, 3, 4, 5)         # => 15
puts join("a", "b", "c")        # => "a, b, c"

# Named splat
def create_hash(**pairs : Int32) : Hash(String, Int32)
  pairs.to_h { |k, v| {k.to_s, v} }
end

result = create_hash(a: 1, b: 2, c: 3)
puts result  # => {"a" => 1, "b" => 2, "c" => 3}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Type-Safe Calculator
สร้าง calculator ด้วย type overloading:
- รองรับ Int32, Float64, Complex
- Operations: +, -, *, /, **
- การ mix types ต้องได้ผลถูกต้อง

### แบบฝึกหัดที่ 2: Generic Collection Operations
เขียน higher-order functions ที่ type-safe:
- `zip`, `unzip`, `partition`
- `group_by`, `chunk`
- ทุก function ต้องมี type restriction ที่เหมาะสม

### แบบฝึกหัดที่ 3: Validation Framework
สร้าง validation framework:
- `Validator(T)` type
- Chain validators ด้วย `and`, `or`
- Return Result หรือ error messages

### แบบฝึกหัดที่ 4: Type-Safe Builder
สร้าง query builder ที่:
- ใช้ type restrictions ป้องกัน invalid states
- Return types ที่แตกต่างกันตาม stage ของ building
- Compile-time verification

## สรุป

Type restrictions ใน Crystal:
- **Overloading**: หลาย methods ที่ชื่อเดียวกัน ต่าง types
- **Union restrictions**: รับได้หลาย types ด้วย `|`
- **Number/Comparable**: abstract types สำหรับ numeric operations
- **NoReturn**: สำหรับ functions ที่ไม่ return
- **Block types**: กำหนด type ของ block parameter

Key insights:
1. Crystal resolves overloads ณ compile time → zero overhead
2. Union types ใน parameters → compiler สร้าง code สำหรับทุก case
3. type restrictions ช่วยทำให้ error messages ชัดเจนขึ้น
4. ใช้ `forall T` เมื่อต้องการ generic behavior
