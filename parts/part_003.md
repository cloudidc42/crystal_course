# Part 003: ตัวแปรและชนิดข้อมูลพื้นฐาน

## ตัวแปร (Variables)

### การประกาศตัวแปร
```crystal
# Crystal ใช้ type inference - ไม่ต้องระบุ type
name = "Alice"        # String
age = 25              # Int32
height = 1.75         # Float64
is_active = true      # Bool
initial = 'A'         # Char

# ระบุ type ชัดเจนก็ได้
name : String = "Alice"
age : Int32 = 25
height : Float64 = 1.75
```

### Naming Conventions
```crystal
# snake_case สำหรับตัวแปรและ methods
first_name = "Alice"
last_name = "Smith"
user_age = 25

# SCREAMING_SNAKE_CASE สำหรับ constants
MAX_SIZE = 100
PI = 3.14159
DEFAULT_TIMEOUT = 30

# PascalCase สำหรับ Classes, Modules, Types
class UserAccount
end

module DatabaseHelper
end

# ขึ้นต้นด้วย _ สำหรับตัวแปรที่ไม่ใช้
_ = "unused"
_result = some_function()

# ขึ้นต้นด้วย @ สำหรับ instance variables
class Person
  @name : String
  @age : Int32
end

# ขึ้นต้นด้วย @@ สำหรับ class variables
class Counter
  @@count : Int32 = 0
end

# ขึ้นต้นด้วย $ สำหรับ global variables (ใช้น้อย)
$global_var = "global"
```

### Multiple Assignment
```crystal
# Assign หลายตัวแปรพร้อมกัน
a, b, c = 1, 2, 3
puts a  # 1
puts b  # 2
puts c  # 3

# Swap values
x, y = 10, 20
x, y = y, x
puts x  # 20
puts y  # 10

# Array destructuring
first, *rest = [1, 2, 3, 4, 5]
puts first  # 1
puts rest   # [2, 3, 4, 5]

*head, last = [1, 2, 3, 4, 5]
puts head   # [1, 2, 3, 4]
puts last   # 5
```

---

## Integer Types

Crystal มี integer types หลายขนาด:

```crystal
# Signed integers
i8 : Int8 = 127              # -128 to 127
i16 : Int16 = 32767          # -32,768 to 32,767
i32 : Int32 = 2147483647     # ±2.1 billion
i64 : Int64 = 9223372036854775807  # ±9.2 quintillion
i128 : Int128 = 170141183460469231731687303715884105727

# Unsigned integers
u8 : UInt8 = 255             # 0 to 255
u16 : UInt16 = 65535         # 0 to 65,535
u32 : UInt32 = 4294967295    # 0 to ~4.3 billion
u64 : UInt64 = 18446744073709551615  # 0 to ~18.4 quintillion
u128 : UInt128 = 340282366920938463463374607431768211455

# Int (platform-dependent: Int32 on 32-bit, Int64 on 64-bit)
# ในทางปฏิบัติส่วนใหญ่ใช้ Int32
default_int = 42    # Int32
```

### Integer Literals
```crystal
# Decimal
decimal = 1000000

# ใช้ _ เป็น separator (อ่านง่าย)
readable = 1_000_000
also_readable = 1_234_567_890

# Hexadecimal
hex = 0xFF          # 255
hex2 = 0xDEAD_BEEF  # 3735928559

# Octal
octal = 0o777       # 511

# Binary
binary = 0b1111_0000  # 240

# Type suffix
int64_val = 42_i64
uint32_val = 100_u32
int8_val = 127_i8
```

### Integer Operations
```crystal
a = 10
b = 3

# Basic arithmetic
puts a + b    # 13
puts a - b    # 7
puts a * b    # 30
puts a / b    # 3 (integer division!)
puts a // b   # 3 (explicit integer division)
puts a % b    # 1 (modulo)
puts a ** b   # 1000 (power)

# Division produces integer
puts 10 / 3     # 3 (NOT 3.333...)
puts 10.0 / 3   # 3.3333... (Float division)
puts 10 / 3.0   # 3.3333... (Float division)

# Bitwise operations
x = 0b1010  # 10
y = 0b1100  # 12

puts (x & y).to_s(2)   # 1000 (AND)
puts (x | y).to_s(2)   # 1110 (OR)
puts (x ^ y).to_s(2)   # 0110 (XOR)
puts (~x).to_s          # -11 (NOT, two's complement)
puts (x << 2).to_s(2)  # 101000 (Left shift)
puts (x >> 1).to_s(2)  # 101 (Right shift)

# Comparison
puts 5 == 5    # true
puts 5 != 6    # true
puts 5 > 3     # true
puts 5 < 3     # false
puts 5 >= 5    # true
puts 5 <= 4    # false

# Auto-increment
count = 0
count += 1    # count = 1
count -= 1    # count = 0
count *= 2    # count = 0
count //= 1   # count = 0

# Useful methods
puts 42.abs       # 42
puts (-42).abs    # 42
puts 5.gcd(3)     # 1 (greatest common divisor)
puts 5.lcm(3)     # 15 (least common multiple)
puts 42.to_s      # "42"
puts 42.to_f      # 42.0
puts 42.to_s(2)   # "101010" (binary)
puts 42.to_s(16)  # "2a" (hex)
puts 255.to_s(8)  # "377" (octal)
```

---

## Float Types

```crystal
# Float32 (single precision)
f32 : Float32 = 3.14_f32

# Float64 (double precision) - default
f64 : Float64 = 3.14
default_float = 3.14   # Float64

# Scientific notation
scientific = 1.5e10    # 15,000,000,000.0
small = 1.5e-5         # 0.000015

# Special values
puts Float64::INFINITY        # Infinity
puts Float64::NAN             # NaN
puts Float64::MAX             # 1.7976931348623157e+308
puts Float64::MIN             # 2.2250738585072014e-308
puts Float64::MIN_POSITIVE    # 2.2250738585072014e-308
```

### Float Operations
```crystal
a = 10.0
b = 3.0

puts a + b    # 13.0
puts a - b    # 7.0
puts a * b    # 30.0
puts a / b    # 3.3333333333333335
puts a % b    # 1.0
puts a ** b   # 1000.0

# Rounding
x = 3.14159
puts x.round        # 3.0
puts x.round(2)     # 3.14
puts x.ceil         # 4.0
puts x.floor        # 3.0
puts x.truncate     # 3.0

# Negative rounding
y = -3.7
puts y.round    # -4.0
puts y.ceil     # -3.0
puts y.floor    # -4.0
puts y.truncate # -3.0

# Math functions
require "math"

puts Math.sqrt(16.0)    # 4.0
puts Math.cbrt(27.0)    # 3.0
puts Math.log(Math::E)  # 1.0
puts Math.log10(100.0)  # 2.0
puts Math.log2(8.0)     # 3.0
puts Math.sin(0.0)      # 0.0
puts Math.cos(0.0)      # 1.0
puts Math.tan(Math::PI/4) # ~1.0

# Constants
puts Math::PI    # 3.141592653589793
puts Math::E     # 2.718281828459045

# Float conversions
puts 3.14.to_i    # 3 (truncates)
puts 3.99.to_i    # 3 (truncates, not rounds!)
puts 3.14.round.to_i  # 3
puts 3.7.round.to_i   # 4
puts 3.14.to_s    # "3.14"
puts 3.14.ceil.to_i   # 4
```

---

## String Type

```crystal
# String literals
single_quotes = 'Hello'    # ใน Crystal นี้คือ Char!
double_quotes = "Hello"    # นี่คือ String

name = "Alice"
greeting = "Hello, #{name}!"

# Multiline strings
multiline = "This is
a multiline
string"

# Here document (heredoc)
heredoc = <<-TEXT
  This is a heredoc
  It preserves newlines
  But strips leading whitespace
  TEXT

# Raw string (no interpolation)
raw = %(Hello, \#{name}!)  # ไม่ interpolate

# String with special characters
special = "Tab:\t\nNewline:\nQuote:\"Backslash:\\"
```

### String Methods
```crystal
s = "Hello, World!"

# Length
puts s.size         # 13
puts s.bytesize     # 13 (bytes)
puts s.empty?       # false
puts "".empty?      # true

# Case
puts s.upcase       # HELLO, WORLD!
puts s.downcase     # hello, world!
puts s.capitalize   # Hello, world! (only first letter)
puts s.swapcase     # hELLO, wORLD!

# Searching
puts s.includes?("World")  # true
puts s.starts_with?("Hello")  # true
puts s.ends_with?("!")  # true
puts s.index("o")   # 4
puts s.rindex("o")  # 8
puts s.count("l")   # 3

# Substrings
puts s[0]           # H (Char)
puts s[0, 5]        # Hello (from index 0, length 5)
puts s[7..]         # World! (from index 7 to end)
puts s[0..4]        # Hello (inclusive range)
puts s[-6..-2]      # World (negative index)
puts s.first(5)     # Hello
puts s.last(6)      # World!

# Modifying (returns new String, immutable)
puts s.strip        # Remove leading/trailing whitespace
puts "  hello  ".strip     # "hello"
puts "  hello  ".lstrip    # "hello  "
puts "  hello  ".rstrip    # "  hello"

puts s.delete("l")  # Heo, Word!
puts s.squeeze("l") # Helo, World!
puts s.tr("aeiou", "*")  # H*ll*, W*rld!

# Splitting and joining
puts s.split(", ")  # ["Hello", "World!"]
puts s.split("")    # Array of chars
parts = ["Hello", "World"]
puts parts.join(", ")  # Hello, World

# Replace
puts s.sub("World", "Crystal")      # Hello, Crystal! (first match)
puts s.gsub("l", "L")               # HeLLo, WorLd!
puts s.gsub(/[aeiou]/i, "*")        # H*ll*, W*rld! (regex)
```

### String Type Conversion
```crystal
# String to Number
"42".to_i           # 42
"3.14".to_f         # 3.14
"0xFF".to_i(16)     # 255 (hex)
"0b1010".to_i(2)    # 10 (binary)
"0o17".to_i(8)      # 15 (octal)

# Safe conversion (returns nil if fails)
"42".to_i?          # 42
"abc".to_i?         # nil
"3.14".to_f?        # 3.14
"xyz".to_f?         # nil

# Number to String
42.to_s             # "42"
3.14.to_s           # "3.14"
42.to_s(2)          # "101010" (binary)
42.to_s(16)         # "2a" (hex)
255.to_s(8)         # "377" (octal)
```

---

## Bool Type

```crystal
# Boolean values
t = true
f = false

# Boolean type
puts t.class   # Bool
puts f.class   # Bool

# Logical operators
puts true && false   # false (AND)
puts true || false   # true (OR)
puts !true           # false (NOT)

# Short-circuit evaluation
# && stops at first false
puts false && (1/0 > 0)   # false (doesn't evaluate right side)

# || stops at first true
puts true || (1/0 > 0)    # true (doesn't evaluate right side)

# Boolean methods
puts true.to_s    # "true"
puts false.to_s   # "false"

# Truthy and Falsy
# Crystal: ONLY false and nil are falsy!
# Everything else is truthy (including 0, "", [])

if 0          # TRUTHY in Crystal!
  puts "0 is truthy"
end

if ""         # TRUTHY in Crystal!
  puts "Empty string is truthy"
end

if []         # TRUTHY in Crystal!
  puts "Empty array is truthy"
end

if nil        # FALSY
  puts "This won't print"
end

if false      # FALSY
  puts "This won't print"
end
```

---

## Char Type

```crystal
# Char คือตัวอักษรเดียว
c = 'A'
digit = '5'
special = '!'
space = ' '

# Unicode chars
heart = '❤'
thai = 'ก'

# Char type
puts c.class   # Char

# Char properties
puts 'A'.ord        # 65 (ASCII/Unicode code point)
puts 65.chr         # A (code point to char)
puts 'a'.upcase     # A
puts 'A'.downcase   # a
puts '5'.to_i       # 5
puts 'a'.alpha?     # true (Crystal 1.2+)
puts '5'.alphanumeric?  # true
puts '5'.number?    # true (digit)
puts ' '.whitespace? # true
puts 'A'.uppercase? # true
puts 'a'.lowercase? # true

# Char arithmetic
puts 'A' + 1   # B
puts 'Z' - 'A' # 25 (difference)

# Comparison
puts 'a' < 'b'   # true
puts 'A' == 65.chr  # true
```

---

## Symbol Type

```crystal
# Symbol คือ immutable string ที่ใช้เป็น identifier
sym = :hello
sym2 = :world
sym3 = :"with spaces"

puts sym         # hello
puts sym.class   # Symbol
puts sym.to_s    # "hello"
puts "hello".to_sym  # hello

# Symbols are unique
puts :hello.object_id == :hello.object_id  # true (same object)
puts "hello".object_id == "hello".object_id  # might be false

# ใช้กับ Hash
config = {
  host: "localhost",    # :host => "localhost"
  port: 8080,           # :port => 8080
  debug: false          # :debug => false
}

puts config[:host]   # localhost
puts config[:port]   # 8080
```

---

## Nil Type

```crystal
# nil คือ "ไม่มีค่า"
nothing = nil
puts nothing        # (blank)
puts nothing.nil?   # true
puts nothing.class  # Nil

# Nil safety
name : String? = nil   # String | Nil (Nullable String)

# ต้องตรวจสอบก่อนใช้
if name
  puts name.upcase    # ปลอดภัย
end

# Safe navigation operator (&.)
puts name&.upcase    # nil (ไม่ error)

# Nil coalescing (|| หรือ ? with default)
display = name || "Anonymous"
display = name.nil? ? "Anonymous" : name

# not_nil! - force unwrap (อาจ raise NilAssertionError)
# ใช้เมื่อแน่ใจ 100% ว่าไม่ nil
puts name.not_nil!.upcase  # Raise error ถ้า nil
```

---

## Type Checking และ Conversion

```crystal
# ตรวจสอบ type
value = 42
puts value.class          # Int32
puts value.is_a?(Int32)   # true
puts value.is_a?(String)  # false
puts value.responds_to?(:to_s)  # true

# Type casting
# to_i, to_f, to_s - safe conversion
puts "42".to_i    # 42
puts 42.to_f      # 42.0
puts 42.to_s      # "42"

# as - unsafe cast (raises error if wrong type)
obj : Int32 | String = 42
num = obj.as(Int32)   # OK
# str = obj.as(String)  # Raises TypeCastError

# as? - safe cast (returns nil if wrong type)
num2 = obj.as?(Int32)    # 42
str = obj.as?(String)    # nil

# Union type handling
def process(value : Int32 | String)
  case value
  when Int32
    puts "Integer: #{value * 2}"
  when String
    puts "String: #{value.upcase}"
  end
end

process(42)       # Integer: 84
process("hello")  # String: HELLO
```

---

## ตัวอย่าง: Type System ใน Action

```crystal
# data_types_demo.cr

# Integer operations
puts "=== Integers ==="
a : Int32 = 100
b : Int32 = 7

puts "#{a} / #{b} = #{a / b}"     # 14 (integer division)
puts "#{a} % #{b} = #{a % b}"     # 2 (remainder)
puts "#{a} ** 2 = #{a ** 2}"      # 10000

# Float precision
puts "\n=== Floats ==="
pi = Math::PI
puts "PI = #{pi}"
puts "Rounded: #{pi.round(5)}"
puts "sin(PI) ≈ #{Math.sin(pi).round(10)}"

# String operations
puts "\n=== Strings ==="
sentence = "The quick brown fox jumps over the lazy dog"
words = sentence.split
puts "Words: #{words.size}"
puts "Unique chars: #{sentence.chars.uniq.size}"
puts "Reversed: #{sentence.reverse}"

# Bool logic
puts "\n=== Booleans ==="
x = 5
puts "#{x} is positive and even: #{x > 0 && x.even?}"
puts "#{x} is negative or odd: #{x < 0 || x.odd?}"

# Nil handling
puts "\n=== Nil Safety ==="
values : Array(String?) = ["hello", nil, "world", nil, "!"]
non_nil = values.compact  # Remove nil values
puts "With nil: #{values}"
puts "Without nil: #{non_nil}"

# Type checking
puts "\n=== Types ==="
mixed : Array(Int32 | Float64 | String | Bool) = [1, 2.5, "three", true]
mixed.each do |item|
  case item
  when Int32   then puts "Int: #{item}"
  when Float64 then puts "Float: #{item}"
  when String  then puts "String: '#{item}'"
  when Bool    then puts "Bool: #{item}"
  end
end
```

---

## สรุป Part 003

ในบทนี้เราได้เรียนรู้:

1. **Variables**: การประกาศ, naming conventions, multiple assignment
2. **Integer types**: Int8/16/32/64/128, UInt8/16/32/64/128
3. **Float types**: Float32, Float64, special values
4. **String**: literals, methods, conversions
5. **Bool**: true/false, logical operators, truthy/falsy
6. **Char**: single character, operations
7. **Symbol**: immutable identifiers
8. **Nil**: null safety, safe navigation operator
9. **Type system**: checking, casting, union types

---

## แบบฝึกหัด

```crystal
# แบบฝึกหัด 1: Temperature converter
# แปลง Celsius เป็น Fahrenheit และ Kelvin

# แบบฝึกหัด 2: String analysis
# รับ string แล้ว:
# - นับจำนวน vowels (a,e,i,o,u)
# - นับจำนวน consonants
# - หาคำที่ยาวที่สุด

# แบบฝึกหัด 3: Number properties
# รับตัวเลขแล้ว:
# - บอกว่าเป็น positive/negative/zero
# - บอกว่าเป็น even/odd
# - บอกว่าเป็น prime หรือไม่
# - แสดงในรูป binary, octal, hex
```

---

## ขั้นตอนต่อไป

ไปที่ [Part 004](part_004.md) เพื่อเรียนรู้:
- ตัวเลขและคณิตศาสตร์ขั้นสูง
- BigDecimal และ BigInt
- Math library
- Random numbers
