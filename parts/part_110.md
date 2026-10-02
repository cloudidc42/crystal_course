# Part 110: Metaprogramming ขั้นสูง

## บทนำ

Metaprogramming ขั้นสูงใน Crystal ใช้ macros เพื่อ introspect type hierarchy, instance variables, และ method signatures ณ compile time ทำให้สร้าง frameworks และ libraries ที่ทรงพลังได้

## Reflection ด้วย Macros

```crystal
# Introspect type information
macro type_info
  puts "=== {{@type.name}} ==="
  puts "Is abstract: #{{{@type.abstract?}}}"
  puts "Is struct: #{{{@type.struct?}}}"
  puts "Is module: #{{{@type.module?}}}"
  puts "Superclass: #{{{@type.superclass || "none"}}}"
  puts "Instance vars:"
  {% for ivar in @type.instance_vars %}
    puts "  @{{ivar.name}} : {{ivar.type}}"
  {% end %}
  puts "Methods (public):"
  {% for method in @type.methods %}
    {% if method.visibility == :public %}
      puts "  {{method.name}}({{method.args.map { |a| "#{a.name}: #{a.restriction}" }.join(", ")}})"
    {% end %}
  {% end %}
end

class Animal
  getter name : String
  getter age : Int32

  def initialize(@name, @age)
  end

  def speak : String
    "..."
  end

  def move : String
    "moving"
  end

  type_info
end
```

## instance_vars ที่ Compile Time

```crystal
# ใช้ instance_vars สร้าง comprehensive toString
macro auto_inspect
  def inspect : String
    parts = ["#{self.class.name}("]
    values = [] of String
    {% for ivar in @type.instance_vars %}
      values << "@{{ivar.name}}=#{@{{ivar.name.id}}.inspect}"
    {% end %}
    parts << values.join(", ")
    parts << ")"
    parts.join
  end

  def to_s : String
    inspect
  end
end

# JSON generation จาก instance_vars
macro auto_json
  require "json"

  def to_json(builder : JSON::Builder)
    builder.object do
      {% for ivar in @type.instance_vars %}
        builder.field {{ivar.name.stringify}}, @{{ivar.name.id}}
      {% end %}
    end
  end
end

struct DatabaseConfig
  @host : String
  @port : Int32
  @name : String
  @username : String
  @ssl : Bool

  def initialize(@host = "localhost", @port = 5432,
                 @name = "mydb", @username = "postgres",
                 @ssl = false)
  end

  auto_inspect
  auto_json
end

config = DatabaseConfig.new("db.example.com", 5432, "production")
puts config.inspect
# => DatabaseConfig(@host="db.example.com", @port=5432, @name="production", ...)
```

## Class Hierarchy ณ Compile Time

```crystal
# Build class hierarchy information
macro print_hierarchy
  {% puts "#{@type.name} hierarchy:" %}
  {% ancestors = [@type] %}
  {% current = @type %}
  {% while current.superclass %}
    {% current = current.superclass %}
    {% ancestors << current %}
  {% end %}
  {% for ancestor in ancestors %}
    {% puts "  #{ancestor.name}" %}
  {% end %}
end

class Vehicle
  print_hierarchy
end

class Car < Vehicle
  print_hierarchy
end

class ElectricCar < Car
  print_hierarchy
end

# Output ณ compile time:
# Vehicle hierarchy:
#   Vehicle
#   Reference
#   Object
# Car hierarchy:
#   Car
#   Vehicle
#   Reference
#   Object
# ElectricCar hierarchy:
#   ElectricCar
#   Car
#   Vehicle
#   Reference
#   Object
```

## Methods Introspection

```crystal
# Analyze method signatures
macro analyze_methods
  {% puts "Methods of #{@type.name}:" %}
  {% for method in @type.methods %}
    {% puts "  #{method.visibility} #{method.name}(#{method.args.map { |a| "#{a.name}: #{a.restriction}" }.join(", ")}) : #{method.return_type || "?"}" %}
  {% end %}
end

class DataProcessor
  def process(data : String, factor : Int32 = 1) : String
    data * factor
  end

  def validate(input : String) : Bool
    !input.empty?
  end

  private def _internal : Nil
    # private method
  end

  analyze_methods
end
```

## Compile-time Serialization Framework

```crystal
# Framework ที่ generate serialization code จาก type info
module AutoSerializable
  macro included
    def serialize : Hash(String, String)
      result = Hash(String, String).new
      {% for ivar in @type.instance_vars %}
        result[{{ivar.name.stringify}}] = @{{ivar.name.id}}.to_s
      {% end %}
      result
    end

    def self.deserialize(data : Hash(String, String)) : {{@type.name.id}}
      obj = allocate
      {% for ivar in @type.instance_vars %}
        if value = data[{{ivar.name.stringify}}]?
          obj.@{{ivar.name.id}} = begin
            {% if ivar.type <= Int32 %}
              value.to_i
            {% elsif ivar.type <= Float64 %}
              value.to_f
            {% elsif ivar.type <= Bool %}
              value == "true"
            {% else %}
              value
            {% end %}
          end
        end
      {% end %}
      obj
    end

    def ==(other : {{@type.name.id}}) : Bool
      {% for ivar in @type.instance_vars %}
        return false unless @{{ivar.name.id}} == other.@{{ivar.name.id}}
      {% end %}
      true
    end
  end
end

struct UserRecord
  include AutoSerializable

  @id : Int32
  @username : String
  @email : String
  @active : Bool
  @score : Float64

  def initialize(@id = 0, @username = "", @email = "", @active = true, @score = 0.0)
  end
end

user = UserRecord.new(1, "somchai", "test@example.com", true, 95.5)
serialized = user.serialize
puts serialized.inspect

restored = UserRecord.deserialize(serialized)
puts user == restored  # => true
```

## Dynamic Method Dispatch

```crystal
# Generate dispatch table ณ compile time
macro define_dispatch_table(*methods)
  DISPATCH = {
    {% for method_name in methods %}
      {{method_name.stringify}} => ->(obj : {{@type.name.id}}, *args : String) {
        obj.{{method_name.id}}(*args)
      },
    {% end %}
  }

  def dispatch(method_name : String, *args : String) : String
    if handler = DISPATCH[method_name]?
      handler.call(self, *args)
    else
      raise "Unknown method: #{method_name}"
    end
  end
end

class CommandProcessor
  def greet(name : String) : String
    "สวัสดี #{name}!"
  end

  def farewell(name : String) : String
    "ลาก่อน #{name}!"
  end

  def echo(*args : String) : String
    args.join(" ")
  end

  # define_dispatch_table :greet, :farewell  # (method signature mismatch example)
end
```

## Type-safe Builder Framework

```crystal
# Auto-generate builder ด้วย macro introspection
macro generate_builder
  class {{@type.name.id}}Builder
    {% for ivar in @type.instance_vars %}
      @{{ivar.name.id}} : {{ivar.type}}?

      def {{ivar.name.id}}(val : {{ivar.type}}) : self
        @{{ivar.name.id}} = val
        self
      end
    {% end %}

    def build : {{@type.name.id}}
      {{@type.name.id}}.new(
        {% for ivar in @type.instance_vars %}
          @{{ivar.name.id}}.not_nil!,
        {% end %}
      )
    end
  end
end

struct Address
  getter street : String
  getter city : String
  getter country : String
  getter zip : String

  def initialize(@street, @city, @country, @zip)
  end

  generate_builder
end

address = AddressBuilder.new
  .street("123 ถนนสุขุมวิท")
  .city("กรุงเทพ")
  .country("ไทย")
  .zip("10110")
  .build

puts address.city  # => กรุงเทพ
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ORM with Full Reflection
สร้าง ORM framework ที่ใช้ macro reflection:
- Auto-detect primary keys
- Foreign key relationships
- Query generation จาก type info

### แบบฝึกหัดที่ 2: Auto-Documentation
Generate documentation จาก type info:
- Method signatures
- Parameter types
- Return types
- Output เป็น Markdown

### แบบฝึกหัดที่ 3: Schema Diff Tool
สร้าง tool ที่:
- Compare สอง class definitions
- Detect added/removed fields
- Generate migration code

### แบบฝึกหัดที่ 4: Test Generation
Auto-generate test stubs จาก class:
- Test method สำหรับทุก public method
- Type-checking tests
- Boundary tests

## สรุป

Metaprogramming ขั้นสูงใน Crystal:
- **@type.instance_vars**: list และ types ของ fields
- **@type.methods**: method definitions
- **@type.superclass**: inheritance chain
- **@type.annotations**: custom metadata
- **method.args**: parameter information

Powerful capabilities:
1. Auto-generate serialization/deserialization
2. Build frameworks ที่ zero boilerplate
3. Type-safe reflection ณ compile time
4. Generate documentation และ schema

Crystal's metaprogramming เหนือกว่า Ruby เพราะ:
- ทำงาน ณ compile time (zero runtime overhead)
- Type-safe ทุกอย่างที่ generate
- ข้อผิดพลาดถูกจับ ณ compile time
