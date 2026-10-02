# Part 101: Macros พื้นฐาน

## บทนำ

Macros ใน Crystal เป็น metaprogramming tool ที่ทำงาน ณ compile time Macros รับ AST (Abstract Syntax Tree) เป็น input และสร้างโค้ดใหม่ขึ้นมา ต่างจาก runtime functions ตรงที่ macros ถูก "expand" ก่อนที่ compilation จะเสร็จ

## Macro Definition พื้นฐาน

```crystal
# macro keyword ใช้กำหนด macro
macro say_hello
  puts "สวัสดีครับ!"
end

# เรียกใช้ macro (ไม่ต้องมี parentheses สำหรับ no-arg macro)
say_hello  # expanded เป็น: puts "สวัสดีครับ!"

# Macro ที่มี argument
macro double_puts(value)
  puts {{value}}
  puts {{value}}
end

double_puts "Crystal"
# Expands เป็น:
# puts "Crystal"
# puts "Crystal"
```

## Macro Variables ด้วย {{ }}

```crystal
# {{ }} interpolate macro arguments เข้าไปใน code
macro square(x)
  {{x}} * {{x}}
end

result = square(5)
puts result  # => 25

# หมายเหตุ: ถ้าใช้กับ side-effect expressions
# square(print_and_return(5)) จะ print 2 ครั้ง!

# ป้องกันด้วยการ assign ก่อน
macro safe_square(x)
  %tmp = {{x}}  # % prefix = hygienic local variable
  %tmp * %tmp
end

result2 = safe_square(5)
puts result2  # => 25
```

## % Prefix - Hygienic Variables

```crystal
# %name - สร้าง unique local variable ที่ไม่ clash กับ outer scope
macro safe_double_puts(value)
  %val = {{value}}  # %val เป็น unique variable name
  puts %val
  puts %val
end

val = "outer"  # ไม่ clash กับ %val ใน macro
safe_double_puts "Crystal"
puts val  # => outer (ยังคงค่าเดิม)
```

## {% if %} - Conditional Compilation

```crystal
# {% if %} ทำงาน ณ compile time
macro debug_log(message)
  {% if flag?(:debug) %}
    puts "[DEBUG] #{{{message}}}"
  {% end %}
end

debug_log "Starting server"
# ถ้า compile ด้วย -Ddebug จะแสดง debug message

# if กับ type check
macro process(value)
  {% if value.is_a?(NumberLiteral) %}
    puts "Number: #{{{value}} * 2}"
  {% elsif value.is_a?(StringLiteral) %}
    puts "String: #{{{value}}.upcase}"
  {% else %}
    puts "Other: #{{{value}}}"
  {% end %}
end

process 42      # => Number: 84
process "hello" # => String: HELLO
process true    # => Other: true
```

## {% for %} - Iteration ณ Compile Time

```crystal
# {% for %} iterate over collections ณ compile time
macro generate_getters(*names)
  {% for name in names %}
    def {{name.id}}
      @{{name.id}}
    end
  {% end %}
end

class Person
  def initialize(@name : String, @age : Int32, @email : String)
  end

  generate_getters :name, :age, :email
end

p = Person.new("สมชาย", 30, "somchai@example.com")
puts p.name   # => สมชาย
puts p.age    # => 30
puts p.email  # => somchai@example.com

# {% for %} กับ range
macro unroll_sum(n)
  result = 0
  {% for i in 1..n %}
    result += {{i}}
  {% end %}
  result
end

puts unroll_sum(5)  # => 15 (1+2+3+4+5, computed at compile time concept)
```

## Macro Expansion

```crystal
# ดู macro expansion ด้วย {{ ... puts }}
macro verbose_add(a, b)
  {{ puts "Expanding add(#{a}, #{b})" }}
  {{a}} + {{b}}
end

result = verbose_add(3, 4)
# ระหว่าง compilation: "Expanding add(3, 4)"
puts result  # => 7 (ณ runtime)

# pp macro arguments
macro inspect_args(x, y)
  {{ puts "x type: #{x.class_name}, y type: #{y.class_name}" }}
  {{ puts "x value: #{x}" }}
end

inspect_args 42, "hello"
# ระหว่าง compilation จะ print info เกี่ยวกับ arguments
```

## define_method Pattern

```crystal
# Generate methods ด้วย macros
macro define_method(name, &block)
  def {{name.id}}
    {{block.body}}
  end
end

class Calculator
  define_method(:pi) { 3.14159265358979 }
  define_method(:e) { 2.71828182845905 }
  define_method(:tau) { 6.28318530717958 }
end

calc = Calculator.new
puts calc.pi   # => 3.14159265358979
puts calc.e    # => 2.71828182845905
puts calc.tau  # => 6.28318530717958

# Generate methods จาก array
macro define_constants(*pairs)
  {% for pair in pairs %}
    def {{pair[0].id}}
      {{pair[1]}}
    end
  {% end %}
end

class MathConstants
  define_constants(
    {"sqrt2", 1.41421356237310},
    {"sqrt3", 1.73205080756888},
    {"golden_ratio", 1.61803398874989}
  )
end

mc = MathConstants.new
puts mc.sqrt2         # => 1.4142...
puts mc.golden_ratio  # => 1.6180...
```

## Macro ที่สร้าง Classes

```crystal
# Macro ที่ generate class
macro create_value_class(name, type)
  class {{name.id}}
    getter value : {{type.id}}

    def initialize(@value : {{type.id}})
    end

    def to_s : String
      "#{self.class.name}(#{@value})"
    end

    def ==(other : {{name.id}}) : Bool
      @value == other.value
    end

    def <=>(other : {{name.id}}) : Int32
      @value <=> other.value
    end

    include Comparable({{name.id}})
  end
end

create_value_class Name, String
create_value_class Temperature, Float64
create_value_class Score, Int32

name = Name.new("สมชาย")
temp = Temperature.new(36.5)
score = Score.new(95)

puts name    # => Name(สมชาย)
puts temp    # => Temperature(36.5)
puts score   # => Score(95)

puts name == Name.new("สมชาย")   # => true
puts temp > Temperature.new(37.0) # => false
```

## Macro ใน Module

```crystal
# Macros ใน module context
module Validatable
  macro validates(field, **options)
    def validate_{{field.id}} : String?
      value = @{{field.id}}

      {% if options[:presence] %}
        return "{{field}} cannot be blank" if value.nil? || (value.responds_to?(:empty?) && value.empty?)
      {% end %}

      {% if options[:min_length] %}
        if v = value.as?(String)
          return "{{field}} too short (min {{options[:min_length]}})" if v.size < {{options[:min_length]}}
        end
      {% end %}

      {% if options[:min] %}
        if v = value.as?(Int32)
          return "{{field}} too small (min {{options[:min]}})" if v < {{options[:min]}}
        end
      {% end %}

      nil  # valid
    end
  end

  def valid? : Bool
    errors.empty?
  end

  def errors : Array(String)
    # subclass can override
    [] of String
  end
end

class UserForm
  include Validatable

  property name : String = ""
  property age : Int32 = 0
  property email : String = ""

  validates :name, presence: true, min_length: 2
  validates :age, min: 18

  def errors : Array(String)
    errs = [] of String
    errs << validate_name if (e = validate_name)
    errs << validate_age  if (e = validate_age)
    errs
  end
end

form = UserForm.new
form.name = "A"  # Too short
form.age = 15    # Too young

puts form.valid?           # => false
form.errors.each { |e| puts e }
```

## Recursive Macros

```crystal
# Macro ที่เรียกตัวเอง
macro factorial(n)
  {% if n == 0 %}
    1
  {% else %}
    {{n}} * factorial({{n - 1}})
  {% end %}
end

# Compile-time factorial
puts factorial(5)   # => 120
puts factorial(10)  # => 3628800

# Fibonacci ณ compile time
macro fib(n)
  {% if n <= 1 %}
    {{n}}
  {% else %}
    fib({{n - 1}}) + fib({{n - 2}})
  {% end %}
end

puts fib(10)  # => 55
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Property Generator
สร้าง macro `auto_property` ที่:
- สร้าง getter, setter, และ presence check method
- รับ multiple fields
- รองรับ default values

### แบบฝึกหัดที่ 2: Test DSL
สร้าง testing DSL ด้วย macros:
- `describe "name" { ... }`
- `it "should ..." { ... }`
- `expect(x).to_eq(y)`

### แบบฝึกหัดที่ 3: Delegation Macro
สร้าง `delegate` macro:
- Delegate methods ไปยัง sub-object
- รองรับ multiple methods
- รองรับ rename

### แบบฝึกหัดที่ 4: Memoize Macro
สร้าง `memoize` macro ที่:
- Wrap method ด้วย caching
- รองรับ arguments (use as cache key)
- Thread-safe

## สรุป

Macros พื้นฐานใน Crystal:
- **{{ }}**: interpolate compile-time expression
- **{% %}**: compile-time control flow
- **%name**: hygienic variable ใน macros
- **forall**: iterate สำหรับ macro args

Key concepts:
1. Macros ทำงาน ณ compile time
2. Macro arguments คือ AST nodes ไม่ใช่ values
3. ใช้ % prefix สำหรับ hygienic variables
4. Macro expansion เพิ่ม code ไม่ใช่ call function
