# Part 107: Compile-time Computation

## บทนำ

Crystal สามารถ execute code ณ compile time ผ่าน macros ทำให้สร้าง constants, lookup tables, และ optimized code โดยไม่มี runtime overhead

## Macro Expressions ณ Compile Time

```crystal
# การ compute ณ compile time
macro compute_at_compile_time(n)
  {% result = 0 %}
  {% for i in 1..n %}
    {% result = result + i %}
  {% end %}
  {{result}}
end

# ค่านี้ถูก compute ณ compile time
SUM_1_TO_100 = compute_at_compile_time(100)
puts SUM_1_TO_100  # => 5050

# Fibonacci ณ compile time
macro ct_fib(n)
  {% if n <= 1 %}
    {{n}}
  {% else %}
    ct_fib({{n - 1}}) + ct_fib({{n - 2}})
  {% end %}
end

FIB_10 = ct_fib(10)
puts FIB_10  # => 55 (computed at compile time)
```

## {% begin %} Blocks

```crystal
# begin block ใน macro context
macro build_lookup_table
  {% begin %}
    TABLE = {
      {% for i in 0..255 %}
        {{i}} => {{i * i}},
      {% end %}
    }
  {% end %}
end

build_lookup_table

puts TABLE[5]    # => 25
puts TABLE[10]   # => 100
puts TABLE[255]  # => 65025

# ใช้ begin เพื่อ group multiple statements
macro define_constants_for(mod)
  {% begin %}
    {% m = mod %}
    SQRT_{{m.upcase.id}} = Math.sqrt({{m == "two" ? 2 : m == "three" ? 3 : 5}})
  {% end %}
end

define_constants_for "two"
define_constants_for "three"

puts SQRT_TWO.round(4)    # => 1.4142
puts SQRT_THREE.round(4)  # => 1.7321
```

## Compile-time Constants

```crystal
# Crystal มี compile-time constants
puts Crystal::VERSION        # => เวอร์ชัน Crystal
puts Crystal::VERSION_MAJOR  # => major version number
puts Crystal::VERSION_MINOR  # => minor version number

# Custom compile-time constants
VERSION = {{`cat VERSION`.chomp}} rescue "unknown"
# อ่าน file ระหว่าง compilation (ถ้ามี)

# Build info
BUILD_DATE = {{ `date +"%Y-%m-%d"`.chomp.stringify }}
BUILD_HOST = {{ `hostname`.chomp.stringify }}

puts "Built on #{BUILD_DATE} at #{BUILD_HOST}"

# Computed constants
MAX_UINT16 = 2_u16 ** 16 - 1
puts MAX_UINT16  # => 65535

GOLDEN_RATIO = (1 + Math.sqrt(5)) / 2
puts GOLDEN_RATIO  # => 1.618033988749895
```

## env_flag และ env

```crystal
# อ่าน environment variables ณ compile time
{% if env("CI") %}
  IS_CI = true
{% else %}
  IS_CI = false
{% end %}

puts "Running in CI: #{IS_CI}"

# Compile flag
{% if flag?(:release) %}
  LOG_LEVEL = "error"
{% elsif flag?(:debug) %}
  LOG_LEVEL = "debug"
{% else %}
  LOG_LEVEL = "info"
{% end %}

puts "Log level: #{LOG_LEVEL}"

# อ่าน env var เพื่อใช้ใน code
DATABASE_URL = {{env("DATABASE_URL") || "postgres://localhost/dev"}}
puts "DB: #{DATABASE_URL}"
```

## Compile-time String Operations

```crystal
# String operations ณ compile time ใน macros
macro upcase_const(name, value)
  {{name.upcase.id}} = {{value}}
end

upcase_const greeting, "hello"
upcase_const message, "world"

puts GREETING  # => hello
puts MESSAGE   # => world

# String interpolation ณ compile time
macro version_string(major, minor, patch)
  "{{major}}.{{minor}}.{{patch}}"
end

VERSION = version_string(2, 1, 0)
puts VERSION  # => 2.1.0

# Generate identifiers จาก strings
macro define_getter_from_string(field_name)
  def {{field_name.id}} : String
    @{{field_name.id}} || ""
  end
end

class Entity
  @username : String?
  @email : String?

  define_getter_from_string "username"
  define_getter_from_string "email"
end
```

## Lookup Tables ณ Compile Time

```crystal
# สร้าง lookup table ณ compile time
macro generate_sin_table(steps)
  SIN_TABLE = StaticArray(Float64, {{steps}}).new { |i|
    Math.sin(2 * Math::PI * i / {{steps}})
  }
end

generate_sin_table(256)

# Fast lookup แทนที่จะ call Math.sin ทุกครั้ง
def fast_sin(angle_radians : Float64) : Float64
  index = ((angle_radians / (2 * Math::PI)) * 256).to_i % 256
  SIN_TABLE[index.abs]
end

puts fast_sin(0.0)            # ≈ 0.0
puts fast_sin(Math::PI / 2)   # ≈ 1.0

# Compile-time prime sieve
macro sieve_of_eratosthenes(max)
  {% primes = (0..max).to_a %}
  {% primes[0] = 0 %}
  {% primes[1] = 0 %}
  {% i = 2 %}
  {% while i * i <= max %}
    {% if primes[i] != 0 %}
      {% j = i * i %}
      {% while j <= max %}
        {% primes[j] = 0 %}
        {% j = j + i %}
      {% end %}
    {% end %}
    {% i = i + 1 %}
  {% end %}
  {{ primes.select { |p| p != 0 } }}
end

PRIMES_UNDER_50 = sieve_of_eratosthenes(50)
puts PRIMES_UNDER_50.inspect
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]
```

## Compile-time Code Generation

```crystal
# Generate specialized code ณ compile time
macro specialize_for_types(*types)
  {% for t in types %}
    def process_{{t.id.downcase}}(value : {{t.id}}) : String
      "Processing {{t.id}}: #{value}"
    end
  {% end %}
end

class Processor
  specialize_for_types Int32, Float64, String, Bool
end

p = Processor.new
puts p.process_int32(42)      # => Processing Int32: 42
puts p.process_float64(3.14)  # => Processing Float64: 3.14
puts p.process_string("hi")   # => Processing String: hi
puts p.process_bool(true)     # => Processing Bool: true
```

## Compile-time Verification

```crystal
# Verify conditions ณ compile time
macro static_assert(condition, message)
  {% unless condition %}
    {% raise message %}
  {% end %}
end

# ตรวจสอบ platform compatibility
{% if sizeof(Pointer(Int32)) != 8 %}
  {% raise "This code requires 64-bit system" %}
{% end %}

# ตรวจสอบ Crystal version
{% if Crystal::VERSION_MAJOR < 1 %}
  {% raise "Requires Crystal 1.0 or higher" %}
{% end %}

# Custom static assertions
static_assert(sizeof(Int32) == 4, "Int32 must be 4 bytes")
static_assert(sizeof(Float64) == 8, "Float64 must be 8 bytes")

puts "All assertions passed!"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Compile-time Hash
สร้าง perfect hash function ณ compile time:
- Generate lookup table จาก constant data
- Zero collision
- O(1) lookup

### แบบฝึกหัดที่ 2: State Machine Tables
Generate transition tables ณ compile time:
- Define states/events/transitions ใน macro
- Generate optimized lookup arrays
- Validate completeness

### แบบฝึกหัดที่ 3: Compile-time Sorting
Sort constants ณ compile time:
- `{% for item in items.sort %}`
- Generate sorted lookup table
- Binary search compatible

### แบบฝึกหัดที่ 4: Platform Optimization
Generate optimized code based on:
- CPU architecture flags
- Endianness
- Available SIMD instructions

## สรุป

Compile-time Computation ใน Crystal:
- **Macro evaluation**: ทุก `{% %}` code runs ณ compile time
- **Constants**: computed values ที่ embed ใน binary
- **Lookup tables**: zero runtime computation
- **env()**: อ่าน environment ณ compile time
- **flag?()**: conditional compilation

Benefits:
1. Zero runtime overhead สำหรับ complex computations
2. Optimized code ตาม target platform
3. Catch errors ณ compile time
4. Generated lookup tables เร็วกว่า runtime computation
