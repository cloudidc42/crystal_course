# Part 103: Macro Control Flow

## บทนำ

Macro control flow ใน Crystal ใช้ `{% %}` syntax สำหรับ compile-time decisions ทำให้สามารถ generate โค้ดที่แตกต่างกันตาม conditions ณ compile time

## {% if %} พื้นฐาน

```crystal
# if ณ compile time
macro platform_message
  {% if flag?(:windows) %}
    "Running on Windows"
  {% elsif flag?(:macos) %}
    "Running on macOS"
  {% elsif flag?(:linux) %}
    "Running on Linux"
  {% else %}
    "Unknown platform"
  {% end %}
end

puts platform_message

# if กับ argument types
macro format_value(x)
  {% if x.is_a?(StringLiteral) %}
    "\"{{x.id}}\""
  {% elsif x.is_a?(NumberLiteral) %}
    {{x}}
  {% elsif x.is_a?(BoolLiteral) %}
    {{x}} ? "yes" : "no"
  {% end %}
end

puts format_value("hello")  # => "hello"
puts format_value(42)       # => 42
puts format_value(true)     # => yes
```

## {% unless %}

```crystal
# unless เป็น inverse ของ if
macro check_production
  {% unless flag?(:production) %}
    puts "[DEV] This is a development build"
  {% end %}
end

check_production

# unless กับ argument check
macro require_string(x)
  {% unless x.is_a?(StringLiteral) %}
    {{ raise "Expected string literal, got #{x.class_name}" }}
  {% end %}
  {{x.id}}
end

# name = require_string(42)  # Compile error!
name = require_string("Crystal")  # OK
puts name
```

## {% for %} - Compile-time Iteration

```crystal
# for ใน array literal
macro generate_math_methods(*operations)
  {% for op in operations %}
    def {{op[0].id}}(a : Float64, b : Float64) : Float64
      a {{op[1].id}} b
    end
  {% end %}
end

class MathHelper
  generate_math_methods(
    {add, +},
    {subtract, -},
    {multiply, *},
    {divide, /}
  )
end

m = MathHelper.new
puts m.add(3.0, 4.0)       # => 7.0
puts m.subtract(10.0, 3.0) # => 7.0
puts m.multiply(3.0, 4.0)  # => 12.0
puts m.divide(10.0, 2.0)   # => 5.0

# for กับ range
macro times_table(n)
  {% for i in 1..n %}
    {% for j in 1..n %}
      puts "{{i}} x {{j}} = #{{{i}} * {{j}}}"
    {% end %}
  {% end %}
end

# times_table(3)
# => 1 x 1 = 1
# => 1 x 2 = 2
# ... etc
```

## {% begin %} - Grouping

```crystal
# begin ใน macro context สำหรับ grouping
macro define_with_check(name, type, default)
  {% begin %}
    property {{name.id}} : {{type.id}} = {{default}}

    def {{name.id}}? : Bool
      !!@{{name.id}}
    end
  {% end %}
end

class Settings
  define_with_check host, String, "localhost"
  define_with_check port, Int32, 8080
  define_with_check debug, Bool, false
end

s = Settings.new
puts s.host   # => localhost
puts s.port   # => 8080
puts s.debug? # => false
```

## {% for %} กับ Hash

```crystal
# iterate over hash literal
macro create_enum_like(**values)
  module {{@type.name}}Constants
    {% for name, val in values %}
      {{name.upcase.id}} = {{val}}
    {% end %}

    def self.all : Hash(Symbol, Int32)
      {
        {% for name, val in values %}
          {{name}} => {{val}},
        {% end %}
      }
    end
  end
end

module AppConfig
  create_enum_like(
    max_retries: 3,
    timeout_seconds: 30,
    max_connections: 100,
    buffer_size: 4096
  )
end

puts AppConfig::Constants::MAX_RETRIES    # => 3
puts AppConfig::Constants::TIMEOUT_SECONDS # => 30
puts AppConfig::Constants.all.inspect
```

## Conditional Compilation ด้วย flag?

```crystal
# flag? - check สำหรับ compile flags
macro conditional_feature
  {% if flag?(:experimental) %}
    def experimental_feature : String
      "This is experimental!"
    end
  {% end %}

  {% if flag?(:legacy) %}
    def legacy_method : String
      "Old API (deprecated)"
    end
  {% else %}
    def legacy_method : String
      raise "Method removed in v2.0"
    end
  {% end %}
end

class Feature
  conditional_feature
end

f = Feature.new
# Compile ด้วย -Dexperimental เพื่อเปิด experimental_feature
# f.experimental_feature  # Available ถ้า -Dexperimental
puts f.legacy_method
```

## {% for %} กับ @type.instance_vars

```crystal
# Generate methods สำหรับทุก instance variable
macro auto_getters
  {% for ivar in @type.instance_vars %}
    def {{ivar.name.id}} : {{ivar.type}}
      @{{ivar.name.id}}
    end
  {% end %}
end

macro auto_setters
  {% for ivar in @type.instance_vars %}
    def {{ivar.name.id}}=(value : {{ivar.type}})
      @{{ivar.name.id}} = value
    end
  {% end %}
end

macro auto_to_s
  def to_s : String
    parts = [] of String
    {% for ivar in @type.instance_vars %}
      parts << "{{ivar.name}}=#{@{{ivar.name.id}}.inspect}"
    {% end %}
    "#{self.class.name}(#{parts.join(", ")})"
  end
end

class Config
  @host : String
  @port : Int32
  @ssl : Bool

  def initialize(@host = "localhost", @port = 8080, @ssl = false)
  end

  auto_getters
  auto_setters
  auto_to_s
end

config = Config.new
config.port = 443
config.ssl = true
puts config  # => Config(@host="localhost", @port=443, @ssl=true)
```

## Macro กับ Splat

```crystal
# Macro ที่รับ splat arguments
macro log_all(*messages)
  {% for msg in messages %}
    puts "[LOG] {{msg}}"
  {% end %}
end

log_all "Starting", "Loading config", "Connecting to database"

# Macro ที่รับ typed splat
macro type_list(*types)
  alias Combined = {% for type in types %} {{type.id}} | {% end %} Nil
end

# สร้าง union type จาก list
type_list Int32, String, Float64
# ขยายเป็น: alias Combined = Int32 | String | Float64 | Nil

val : Combined = 42
puts typeof(val)  # => Int32 | String | Float64 | Nil
```

## Nested Control Flow

```crystal
# ซ้อน if ใน for
macro generate_validators(*fields)
  def validate : Array(String)
    errors = [] of String
    {% for field in fields %}
      {% if field.is_a?(NamedTupleLiteral) %}
        if @{{field[:name].id}}.nil?
          errors << "{{field[:name]}} is required"
        end
        {% if field[:min_length] %}
          if v = @{{field[:name].id}}
            if v.responds_to?(:size) && v.size < {{field[:min_length]}}
              errors << "{{field[:name]}} is too short (minimum {{field[:min_length]}} chars)"
            end
          end
        {% end %}
      {% end %}
    {% end %}
    errors
  end
end

class RegistrationForm
  property username : String?
  property email : String?
  property password : String?

  generate_validators(
    {name: username, min_length: 3},
    {name: email},
    {name: password, min_length: 8}
  )
end

form = RegistrationForm.new
form.username = "ab"  # too short

errors = form.validate
errors.each { |e| puts e }
```

## {% raise %} - Compile-time Errors

```crystal
# raise ใน macro - compile-time error
macro assert_positive(n)
  {% unless n > 0 %}
    {% raise "Expected positive number, got #{n}" %}
  {% end %}
  {{n}}
end

puts assert_positive(5)    # => 5
# assert_positive(-1)      # Compile error!
# assert_positive(0)       # Compile error!

# Type assertion
macro ensure_number(x)
  {% unless x.is_a?(NumberLiteral) %}
    {% raise "#{x.class_name} is not a number literal" %}
  {% end %}
  {{x}}
end

puts ensure_number(42)     # => 42
puts ensure_number(3.14)   # => 3.14
# ensure_number("hello")   # Compile error!
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Platform-specific Code
สร้าง macro ที่:
- ตรวจสอบ OS (linux/macos/windows)
- Generate platform-specific implementations
- Fallback สำหรับ unknown platforms

### แบบฝึกหัดที่ 2: Feature Flags
สร้าง feature flag system:
- `@[Feature("name")]` annotation
- Macro ที่ generate code ตาม enabled features
- Compile-time on/off

### แบบฝึกหัดที่ 3: Code Generator
สร้าง macro ที่ generate:
- CRUD methods สำหรับ data classes
- Validation methods
- Serialization/deserialization

### แบบฝึกหัดที่ 4: Conditional Logging
สร้าง logging system ที่:
- ปิด log statements ณ compile time ใน production
- Support log levels
- Zero overhead เมื่อ disabled

## สรุป

Macro Control Flow ใน Crystal:
- **{% if %}/{% elsif %}/{% else %}**: conditional compilation
- **{% unless %}**: inverse conditional
- **{% for %}**: compile-time iteration
- **{% begin %}/{% end %}**: grouping
- **{% raise %}**: compile-time errors

Key use cases:
1. Platform-specific code generation
2. Feature flags ที่ zero overhead
3. Code generation จาก metadata
4. Compile-time validation
