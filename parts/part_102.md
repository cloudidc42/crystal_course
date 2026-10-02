# Part 102: Macro Variables

## บทนำ

Macro variables คือวิธีการ access และ manipulate AST nodes ใน macros ด้วย `{{ }}` syntax Crystal มี special macro variables เช่น `@type`, `@method_name`, `@def` ที่ให้ข้อมูลเกี่ยวกับ context

## {{ ... }} Interpolation

```crystal
# พื้นฐาน interpolation
macro greet(name)
  puts "Hello, #{{{name}}}!"
end

greet "Crystal"  # => Hello, Crystal!
greet "World"    # => Hello, World!

# Interpolation ใน method names
macro define_checker(type_name)
  def {{type_name.id}}? : Bool
    self.is_a?({{type_name.id}})
  end
end

class Object
  define_checker(Int32)
  define_checker(String)
  define_checker(Float64)
end

puts 42.int32?   # => true
puts "hi".int32? # => false
puts 3.14.float64? # => true
```

## @type - Type Context

```crystal
# @type ให้ข้อมูลเกี่ยวกับ current type context
macro show_type_info
  puts "Current type: #{{{@type.name}}}"
  puts "Is module: #{{{@type.module?}}}"
  puts "Is struct: #{{{@type.struct?}}}"
  puts "Superclass: #{{{@type.superclass}}}"
end

class MyClass
  show_type_info
end

struct MyStruct
  show_type_info
end

# @type ใน instance method
macro log_class_name
  def class_info : String
    "#{{{@type.name}}}"
  end
end

class Animal
  log_class_name
end

class Dog < Animal
  log_class_name
end

puts Animal.new.class_info  # => Animal
puts Dog.new.class_info     # => Dog
```

## @type.instance_vars

```crystal
# @type.instance_vars - list ของ instance variables
macro print_instance_vars
  puts "Instance vars of #{{{@type.name}}}:"
  {% for ivar in @type.instance_vars %}
    puts "  @{{ivar.name}} : {{ivar.type}}"
  {% end %}
end

class Config
  property host : String = "localhost"
  property port : Int32 = 8080
  property debug : Bool = false
  property name : String?

  print_instance_vars
end
# Output:
# Instance vars of Config:
#   @host : String
#   @port : Int32
#   @debug : Bool
#   @name : String | Nil

# ใช้สร้าง auto-methods
macro auto_to_h
  def to_h : Hash(String, String)
    h = Hash(String, String).new
    {% for ivar in @type.instance_vars %}
      h[{{ivar.name.stringify}}] = @{{ivar.name}}.to_s
    {% end %}
    h
  end
end

class Product
  property name : String
  property price : Float64
  property stock : Int32

  def initialize(@name, @price, @stock)
  end

  auto_to_h
end

p = Product.new("สมุดโน้ต", 45.0, 100)
puts p.to_h.inspect
# => {"name" => "สมุดโน้ต", "price" => "45.0", "stock" => "100"}
```

## @method_name

```crystal
# @method_name ให้ชื่อของ current method
macro debug_method
  puts "Entering method: #{{{@method_name}}}"
end

class Service
  def process_data
    debug_method  # => Entering method: process_data
    "processed"
  end

  def validate_input
    debug_method  # => Entering method: validate_input
    true
  end
end

svc = Service.new
svc.process_data
svc.validate_input
```

## @def - Method Definition Info

```crystal
# @def ให้ข้อมูลเกี่ยวกับ method ปัจจุบัน
macro log_args
  {% puts "Method: #{@def.name}" %}
  {% puts "Args: #{@def.args.map { |a| "#{a.name}: #{a.restriction}" }.join(", ")}" %}
end

class DataProcessor
  def process(input : String, factor : Int32) : String
    log_args
    input * factor
  end

  def transform(value : Float64) : Int32
    log_args
    value.to_i
  end
end

# Output during compilation:
# Method: process
# Args: input: String, factor: Int32
# Method: transform
# Args: value: Float64
```

## @def.args - Method Arguments Info

```crystal
# ใช้ @def.args เพื่อ generate validation
macro validate_args
  {% for arg in @def.args %}
    {% if arg.restriction.is_a?(Path) && !arg.restriction.resolve.nilable? %}
      raise ArgumentError.new("{{arg.name}} cannot be nil") if {{arg.name}}.nil?
    {% end %}
  {% end %}
end

class UserService
  def create_user(name : String, email : String, age : Int32)
    # validate_args  # ใน context นี้ไม่จำเป็นเพราะ types ไม่ nilable
    "Created user #{name}"
  end
end

# Macro ที่ generate logging wrapper
macro logged_method(name, *args, &body)
  def {{name.id}}({{*args}})
    puts "Calling {{name}}"
    start = Time.monotonic
    result = begin
      {{body.body}}
    end
    elapsed = Time.monotonic - start
    puts "{{name}} took #{elapsed.total_milliseconds.round(2)}ms"
    result
  end
end
```

## Macro Arguments Types

```crystal
# Macro arguments เป็น AST nodes
macro inspect_arg(x)
  {% puts "Class: #{x.class_name}" %}
  {% if x.is_a?(NumberLiteral) %}
    {% puts "Number value: #{x}" %}
  {% elsif x.is_a?(StringLiteral) %}
    {% puts "String value: #{x}" %}
  {% elsif x.is_a?(BoolLiteral) %}
    {% puts "Bool value: #{x}" %}
  {% elsif x.is_a?(ArrayLiteral) %}
    {% puts "Array size: #{x.size}" %}
  {% elsif x.is_a?(SymbolLiteral) %}
    {% puts "Symbol: #{x.id}" %}
  {% end %}
end

inspect_arg 42
inspect_arg "hello"
inspect_arg true
inspect_arg [1, 2, 3]
inspect_arg :my_symbol

# Output (ณ compile time):
# Class: NumberLiteral
# Number value: 42
# etc.
```

## Named Arguments ใน Macro

```crystal
# Macro รับ named arguments
macro configure(name, **options)
  class {{name.id}}Config
    {% for key, value in options %}
      CONSTANT_{{key.upcase.id}} = {{value}}
    {% end %}

    def self.settings
      result = {} of String => String
      {% for key, value in options %}
        result[{{key.stringify}}] = CONSTANT_{{key.upcase.id}}.to_s
      {% end %}
      result
    end
  end
end

configure Server,
  host: "localhost",
  port: 8080,
  timeout: 30,
  max_connections: 100

puts ServerConfig::CONSTANT_HOST   # => localhost
puts ServerConfig::CONSTANT_PORT   # => 8080
puts ServerConfig.settings.inspect
```

## @type.methods - Introspect Methods

```crystal
# @type.methods - list ของ methods ที่ defined
macro list_methods
  {% for method in @type.methods %}
    {% if !method.name.starts_with?("_") %}
      puts "  {{method.name}}({{method.args.map { |a| "#{a.name}" }.join(", ")}})"
    {% end %}
  {% end %}
end

class Calculator
  def add(a : Int32, b : Int32) : Int32
    a + b
  end

  def subtract(a : Int32, b : Int32) : Int32
    a - b
  end

  def multiply(a : Int32, b : Int32) : Int32
    a * b
  end

  puts "Calculator methods:"
  list_methods
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Auto-Inspect
สร้าง macro `auto_inspect` ที่:
- ใช้ `@type.instance_vars`
- Generate `inspect` method
- Format ที่อ่านง่าย

### แบบฝึกหัดที่ 2: Field Documentation
สร้าง macro `document_fields` ที่:
- ใช้ `@type.instance_vars`
- Generate method ที่คืน field names และ types เป็น Hash
- มีประโยชน์สำหรับ API documentation

### แบบฝึกหัดที่ 3: Auto-Equality
สร้าง `auto_eq` macro ที่:
- Generate `==` method จาก instance variables
- Compare ทุก fields
- Handles nil fields ถูกต้อง

### แบบฝึกหัดที่ 4: Serialization Macro
สร้าง macro ที่:
- ใช้ `@type.instance_vars` และ `@type.name`
- Generate `to_msgpack` equivalent
- Custom field names via annotation

## สรุป

Macro Variables ใน Crystal:
- **{{ expr }}**: interpolate AST expression
- **@type**: current type ที่ macro run ใน
- **@type.instance_vars**: list ของ instance variables
- **@type.methods**: list ของ methods
- **@method_name**: ชื่อ current method
- **@def**: ข้อมูลเกี่ยวกับ method definition

Best practices:
1. ใช้ `@type.instance_vars` เพื่อ generate code จาก fields
2. ใช้ `@method_name` สำหรับ logging/debugging
3. ตรวจสอบ argument types ด้วย `.is_a?()`
4. ใช้ `%name` สำหรับ hygienic variables
