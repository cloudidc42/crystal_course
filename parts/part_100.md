# Part 100: Type Inference ขั้นสูง

## บทนำ

Crystal ใช้ global type inference algorithm ที่เรียกว่า "type narrowing" ซึ่งทำให้ไม่ต้องระบุ types ในหลายกรณี แต่ยังคง compile-time type safety ไว้

## พื้นฐาน Type Inference

```crystal
# Crystal infers types จาก assignments
x = 42           # Int32
y = 3.14         # Float64
z = "hello"      # String
flag = true      # Bool
arr = [1, 2, 3]  # Array(Int32)

# Infer จาก method return type
doubled = x * 2       # Int32
upcase = z.upcase     # String
length = z.size       # Int32
negated = !flag       # Bool

puts typeof(x)        # => Int32
puts typeof(doubled)  # => Int32
puts typeof(upcase)   # => String

# Infer จาก literal
small = 127_i8    # Int8
big = 9999_i64    # Int64
hex = 0xFF_u32    # UInt32
```

## Union Type Growth

```crystal
# Type ขยายเป็น union เมื่อ assign ต่าง types
value = 1        # Int32
value = "hello"  # Now: Int32 | String

puts typeof(value)  # => Int32 | String

# ใน conditional
def might_be_nil(flag : Bool)
  if flag
    42  # Int32
  end
  # implicit nil ถ้า flag เป็น false
end

result = might_be_nil(true)
puts typeof(result)  # => Int32 | Nil

# Array ของ mixed types
mixed = [1, "two", 3.0]
puts typeof(mixed)  # => Array(Int32 | String | Float64)
```

## Type Restrictions Narrowing

```crystal
# Compiler narrows type หลัง checks
def process(value : Int32 | String | Nil)
  # Type: Int32 | String | Nil

  if value.nil?
    puts "Got nil"
    return
  end
  # Type: Int32 | String (nil excluded)

  if value.is_a?(Int32)
    # Type: Int32
    puts "Int: #{value * 2}"
  else
    # Type: String (Int32 and Nil excluded)
    puts "String: #{value.upcase}"
  end
end

process(42)
process("hello")
process(nil)

# Narrowing ใน case
def describe(x : Int32 | Float64 | String | Bool | Nil) : String
  case x
  when Int32   then "Int: #{x}"     # x is Int32
  when Float64 then "Float: #{x}"   # x is Float64
  when String  then "Str: #{x}"     # x is String
  when Bool    then "Bool: #{x}"    # x is Bool
  when Nil     then "Nil"           # x is Nil
  end
end

puts describe(42)      # => Int: 42
puts describe(3.14)    # => Float: 3.14
puts describe("hi")   # => Str: hi
puts describe(true)   # => Bool: true
puts describe(nil)    # => Nil
```

## Crystal's Type Inference Algorithm

```crystal
# Crystal ใช้ "type widening" ไม่ใช่ "type inference"
# ทุก assignment expand type ไม่เคย shrink

def example
  x = 1      # x: Int32
  x = "two"  # x: Int32 | String

  if x.is_a?(Int32)
    x = x + 1      # x: Int32 here
    # หลัง if block: x: Int32 | String (กลับมา)
  end
  # x: Int32 | String
  x
end

puts typeof(example)  # => Int32 | String

# Flow-sensitive typing
def flow_example(n : Int32) : String
  result = ""    # String

  if n > 0
    result = "positive"  # Still String
  elsif n < 0
    result = "negative"  # Still String
  else
    result = "zero"      # Still String
  end

  result  # String
end
```

## Compile-time Guarantees

```crystal
# Crystal's type system รับประกัน:
# 1. No null pointer exceptions (ใช้ nilable types)
# 2. Type mismatches ถูกจับ ณ compile time
# 3. Method calls ตรวจสอบ ณ compile time

struct Account
  getter id : Int32
  getter balance : Float64
  getter owner_name : String

  def initialize(@id, @balance, @owner_name)
  end

  def deposit(amount : Float64) : Account
    raise ArgumentError.new("Amount must be positive") if amount <= 0
    Account.new(@id, @balance + amount, @owner_name)
  end

  def withdraw(amount : Float64) : Account
    raise ArgumentError.new("Insufficient funds") if amount > @balance
    Account.new(@id, @balance - amount, @owner_name)
  end
end

# Compile-time: ทุก call เป็น type-safe
acc = Account.new(1, 1000.0, "สมชาย")
acc2 = acc.deposit(500.0)
puts acc2.balance  # => 1500.0

# acc.deposit("500")  # Compile error!
# acc.balance = 0     # Compile error! (getter, not property)
```

## Type Inference กับ Generics

```crystal
# Generic method - Crystal infers T จาก arguments
def wrap(x : T) : Array(T) forall T
  [x]
end

a1 = wrap(1)     # Array(Int32) - T inferred as Int32
a2 = wrap("hi")  # Array(String) - T inferred as String

puts typeof(a1)  # => Array(Int32)
puts typeof(a2)  # => Array(String)

# Complex inference
def transform_all(items : Array(T), &block : T -> U) : Array(U) forall T, U
  items.map { |x| block.call(x) }
end

nums = [1, 2, 3, 4, 5]
strings = transform_all(nums) { |x| x.to_s }
# T inferred as Int32, U inferred as String

puts typeof(strings)  # => Array(String)
puts strings.inspect  # => ["1", "2", "3", "4", "5"]
```

## Type Inference ข้ามกับ Control Flow

```crystal
# If expression type = union ของทุก branches
def coin_flip : Int32 | String
  if rand(2) == 0
    42      # Int32
  else
    "heads"  # String
  end
end

result = coin_flip
puts typeof(result)  # => Int32 | String

# Ternary operator
x = rand(2) == 0 ? 1 : "one"
puts typeof(x)  # => Int32 | String

# While loop - type ขยายตาม loop body
val = 0       # Int32
while rand(10) > 5
  val = "changed" if rand(2) == 0  # val: Int32 | String
end
puts typeof(val)  # => Int32 | String
```

## Type Widening ที่ซับซ้อน

```crystal
# Method return type เป็น union ของทุก return paths
def complex_return(n : Int32)
  if n < 0
    return "negative"    # String
  end

  if n == 0
    return nil           # Nil
  end

  if n < 100
    return n             # Int32
  end

  return n.to_f          # Float64
end

result = complex_return(50)
puts typeof(result)  # => String | Nil | Int32 | Float64

# Handle all cases
case result
when String  then puts "String: #{result}"
when Nil     then puts "Nil"
when Int32   then puts "Int32: #{result}"
when Float64 then puts "Float64: #{result}"
end
```

## Limitations และ Workarounds

```crystal
# บางครั้ง type inference อาจ unexpected
arr = [] of Int32 | String  # Explicit type needed
arr << 1
arr << "hello"

# ไม่สามารถ infer empty array ได้
# arr = []  # Error: cannot use inferred literal type

# Workaround: ระบุ type ชัดเจน
arr2 = Array(Int32).new
arr2 << 1 << 2 << 3

# หรือ
arr3 : Array(Int32) = []
arr3 << 4 << 5

# Type annotation ช่วย
my_hash : Hash(String, Int32) = {}
my_hash["a"] = 1

# Hash.new กับ type
my_hash2 = Hash(String, Array(Int32)).new { |h, k| h[k] = [] of Int32 }
my_hash2["numbers"] << 1
my_hash2["numbers"] << 2
```

## Inference กับ Blocks

```crystal
# Crystal infers block argument types
[1, 2, 3].each { |x|
  puts typeof(x)  # => Int32 (inferred from Array(Int32))
}

["a", "b"].map { |s|
  puts typeof(s)  # => String
  s.upcase
}

# Block return type inferred
result = [1, 2, 3].map { |x| x * 2.0 }
puts typeof(result)  # => Array(Float64)

result2 = [1, 2, 3].select { |x| x > 1 }
puts typeof(result2)  # => Array(Int32)

# Mixed return type ใน block
result3 = [1, 2, 3].map { |x|
  if x.even?
    x.to_s
  else
    x
  end
}
puts typeof(result3)  # => Array(Int32 | String)
```

## Type Inference กับ Struct/Class Fields

```crystal
# Field types ต้องระบุ ไม่สามารถ infer ได้
struct Config
  # ต้องระบุ type ของ fields
  getter host : String
  getter port : Int32
  getter debug : Bool

  def initialize(@host, @port, @debug = false)
  end
end

# แต่ instance variables inferred จาก constructor
class DynamicConfig
  def initialize(host : String, port : Int32)
    @host = host    # String inferred
    @port = port    # Int32 inferred
    @connections = 0  # Int32 inferred
    @name = nil     # Nil inferred (จะเป็น Nil | ... ถ้า assign อย่างอื่น)
  end

  def set_name(name : String)
    @name = name   # Now: Nil | String
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Type Exploration
เขียนโปรแกรมที่:
- แสดง typeof ใน various situations
- สังเกต union type growth
- วาง type annotations เพื่อป้องกัน unexpected unions

### แบบฝึกหัดที่ 2: Return Type Analysis
สร้าง method ที่มี complex return types:
- ทดสอบ exhaustive case handling
- ดูว่า compiler บังคับ handle ทุก case

### แบบฝึกหัดที่ 3: Generic Constraints
เขียน generic algorithms ที่:
- ใช้ implicit constraints จาก method usage
- Document expected capabilities
- Test กับ types ที่ไม่ support

### แบบฝึกหัดที่ 4: Refactoring for Type Safety
Take existing code ที่ใช้ Object หรือ Any:
- Refactor ให้ใช้ proper union types
- Add narrowing
- Ensure all paths handled

## สรุป

Type Inference ขั้นสูงใน Crystal:
- **Global**: ใช้ข้อมูลจากทั้ง codebase ไม่ใช่แค่ local
- **Widening**: types ขยาย ไม่ shrink
- **Flow-sensitive**: type narrowing หลัง checks
- **Exhaustive**: compiler ตรวจสอบทุก case ใน pattern matching

Compile-time guarantees:
1. ไม่มี null pointer exceptions ถ้าใช้ nilable types ถูกต้อง
2. Type mismatches ถูกจับ compile time
3. Method calls verified ณ compile time
4. Exhaustive handling ของ union types
