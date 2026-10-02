# Part 98: Type Variables ใน Crystal

## บทนำ

Type variables ใน Crystal ช่วยให้เราเขียน code ที่ operate บน types เอง ไม่ใช่แค่ instances ของ types เช่น การสร้าง instances จาก type, การใช้ class methods, และการ introspect type information

## T.class - Class Type

```crystal
# T.class เป็น type ของ class T เอง
def create_empty(klass : T.class) : T forall T
  klass.new
end

class Animal
  def initialize
    puts "#{self.class} created"
  end
end

class Dog < Animal
end

class Cat < Animal
end

dog = create_empty(Dog)  # => Dog created
cat = create_empty(Cat)  # => Cat created

# ใช้ type เป็น argument
def describe_class(klass : T.class) : String forall T
  "Class: #{klass.name}, Size: #{instance_sizeof(T)}"
end

puts describe_class(Int32)   # => Class: Int32, Size: 4
puts describe_class(String)  # => Class: String, Size: 32
```

## typeof(T) - Compile-time Type Info

```crystal
# typeof ใช้ได้กับ expressions, ไม่ใช่แค่ variables
x = 42
puts typeof(x)          # => Int32

arr = [1, 2, 3]
puts typeof(arr)        # => Array(Int32)
puts typeof(arr.first)  # => Int32
puts typeof(arr.size)   # => Int32
puts typeof(arr.sum)    # => Int32

# typeof ใน generic context
def type_name_of(value : T) : String forall T
  typeof(value).to_s
end

puts type_name_of(42)        # => Int32
puts type_name_of("hello")   # => String
puts type_name_of(3.14)      # => Float64
puts type_name_of([1, 2, 3]) # => Array(Int32)

# typeof กับ union types
def show_union_type(a : Int32 | String | Nil)
  puts typeof(a)  # => Int32 | String | Nil
  if a.is_a?(Int32)
    puts typeof(a)  # => Int32 (narrowed)
  end
end

show_union_type(42)
```

## T.new - Creating Instances from Type Variable

```crystal
# T.new ใน generic context
def create_array_of(klass : T.class, size : Int32) : Array(T) forall T
  Array(T).new(size) { klass.new }
end

# Works กับ types ที่มี no-arg constructor
class Config
  property loaded : Bool = false

  def initialize
    @loaded = false
  end
end

configs = create_array_of(Config, 3)
puts configs.size  # => 3
puts configs.first.loaded  # => false

# Generic factory
class Factory(T)
  def create : T
    T.new
  end

  def create_many(count : Int32) : Array(T)
    Array(T).new(count) { T.new }
  end
end

class Widget
  getter id : Int32
  @@counter = 0

  def initialize
    @@counter += 1
    @id = @@counter
  end
end

factory = Factory(Widget).new
widgets = factory.create_many(5)
widgets.each { |w| puts "Widget ##{w.id}" }
```

## Accessing Type at Runtime

```crystal
# .class - runtime class access
value : Int32 | String = 42
puts value.class  # => Int32

value = "hello"
puts value.class  # => String

# .class เปรียบเทียบ
def same_type?(a, b) : Bool
  a.class == b.class
end

puts same_type?(1, 2)       # => true
puts same_type?(1, "hello") # => false
puts same_type?(1, 1.0)     # => false

# instance_of? vs is_a?
class Animal; end
class Dog < Animal; end

dog = Dog.new
puts dog.is_a?(Animal)       # => true (inheritance)
puts dog.is_a?(Dog)          # => true
puts dog.class == Dog        # => true
puts dog.class == Animal     # => false
```

## Type Metadata

```crystal
# Introspect type information
class DataClass
  getter name : String
  getter age : Int32
  property email : String?

  def initialize(@name, @age)
    @email = nil
  end
end

# Crystal ใช้ macros สำหรับ type reflection
# (runtime reflection ใน Crystal มีจำกัด - ส่วนใหญ่ compile-time)

# ขนาดของ type
puts sizeof(Int32)         # => 4
puts sizeof(Float64)       # => 8
puts sizeof(Bool)          # => 1
puts instance_sizeof(DataClass)  # runtime class size

# Type checking ระบบ
def process_number(n : T) forall T
  if n.is_a?(Float)
    puts "Float: #{n.round(2)}"
  elsif n.is_a?(Int)
    puts "Int: #{n}"
  else
    puts "Other number: #{n}"
  end
end

process_number(42)
process_number(3.14)
process_number(100_i64)
```

## Compile-time Type Registry

```crystal
# ใช้ macros สร้าง type registry (advanced)
# แสดง concept พื้นฐาน

# Runtime type mapping ด้วย Hash
module TypeRegistry
  @@registry = Hash(String, Proc(String, String)).new

  def self.register(type_name : String, handler : Proc(String, String))
    @@registry[type_name] = handler
  end

  def self.handle(type_name : String, data : String) : String
    handler = @@registry[type_name]?
    if handler
      handler.call(data)
    else
      "Unknown type: #{type_name}"
    end
  end
end

TypeRegistry.register("int", ->(s : String) { "Int: #{s.to_i? || "invalid"}" })
TypeRegistry.register("float", ->(s : String) { "Float: #{s.to_f? || "invalid"}" })
TypeRegistry.register("string", ->(s : String) { "String: #{s}" })

puts TypeRegistry.handle("int", "42")
puts TypeRegistry.handle("float", "3.14")
puts TypeRegistry.handle("string", "hello")
puts TypeRegistry.handle("bool", "true")
```

## Generic Type Patterns

```crystal
# Phantom types - types ที่ใช้เป็น markers เท่านั้น
struct Validated; end
struct Unvalidated; end

struct UserInput(State)
  getter value : String

  def initialize(@value : String)
  end

  def validate : UserInput(Validated) | Nil
    # Validate the input
    if @value.size >= 3 && @value.size <= 50
      UserInput(Validated).new(@value)
    else
      nil
    end
  end
end

def process_validated_input(input : UserInput(Validated)) : String
  "Processed: #{input.value}"
end

raw = UserInput(Unvalidated).new("Crystal")
validated = raw.validate

if validated
  # Type system ensures we only call this with validated input
  puts process_validated_input(validated)
else
  puts "Invalid input"
end

# process_validated_input(raw)  # Compile error!
```

## Type Variables ใน Module

```crystal
# Module ที่ใช้ type variables
module Buildable(T)
  abstract def build : T

  def build_many(count : Int32) : Array(T)
    Array(T).new(count) { build }
  end
end

struct ServerConfig
  getter host : String
  getter port : Int32
  getter ssl : Bool

  def initialize(@host = "localhost", @port = 8080, @ssl = false)
  end
end

class ServerConfigBuilder
  include Buildable(ServerConfig)

  def initialize
    @host = "localhost"
    @port = 8080
    @ssl = false
  end

  def host(h : String) : self
    @host = h
    self
  end

  def port(p : Int32) : self
    @port = p
    self
  end

  def ssl(s : Bool = true) : self
    @ssl = s
    self
  end

  def build : ServerConfig
    ServerConfig.new(@host, @port, @ssl)
  end
end

config = ServerConfigBuilder.new
  .host("example.com")
  .port(443)
  .ssl
  .build

puts "#{config.host}:#{config.port} SSL=#{config.ssl}"

# Build many
configs = ServerConfigBuilder.new.build_many(3)
puts configs.size  # => 3
```

## Accessing Type Class at Runtime

```crystal
# เปรียบเทียบ .class กับ typeof
value = 42

# typeof - compile-time, return type expression
puts typeof(value)    # => Int32 (at compile time)

# .class - runtime, return Class object
puts value.class      # => Int32 (at runtime)
puts value.class.name # => "Int32"

# Class comparison
def is_integer?(value) : Bool
  value.class == Int32 ||
  value.class == Int64 ||
  value.class == UInt32
end

puts is_integer?(42)      # => true
puts is_integer?(42_i64)  # => true
puts is_integer?(42.0)    # => false
puts is_integer?("42")    # => false
```

## Type-based Dispatch

```crystal
# Runtime dispatch based on type
class TypeDispatcher
  @handlers = Hash(String, Proc(Object, String)).new

  def register(klass : T.class, &handler : T -> String) forall T
    @handlers[klass.name] = ->(obj : Object) {
      if typed = obj.as?(T)
        handler.call(typed)
      else
        "Type mismatch"
      end
    }
  end

  def dispatch(obj : Object) : String
    handler = @handlers[obj.class.name]?
    if handler
      handler.call(obj)
    else
      "No handler for #{obj.class.name}"
    end
  end
end

dispatcher = TypeDispatcher.new

dispatcher.register(Int32) { |n| "Integer: #{n * 2}" }
dispatcher.register(String) { |s| "String: #{s.upcase}" }
dispatcher.register(Float64) { |f| "Float: #{f.round(2)}" }

objects = [42.as(Object), "hello".as(Object), 3.14.as(Object), true.as(Object)]

objects.each do |obj|
  puts dispatcher.dispatch(obj)
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Type-safe Factory
สร้าง `TypeSafeFactory(T)` ที่:
- Register builders สำหรับ subtypes ของ T
- Create instances based on type tag
- Type-safe ทุก operation

### แบบฝึกหัดที่ 2: Phantom Type State Machine
ใช้ phantom types สร้าง state machine:
- States เป็น empty structs (markers)
- Methods เฉพาะบาง states เท่านั้น
- Compile-time validation

### แบบฝึกหัดที่ 3: Type-Indexed Map
สร้าง `TypeMap` ที่:
- เก็บ values indexed by type
- `get(SomeType)` return value ของ type นั้น
- Type-safe โดยไม่ต้อง cast

### แบบฝึกหัดที่ 4: Generic Object Pool
สร้าง `ObjectPool(T)` ที่:
- สร้าง T instances ล่วงหน้า
- Checkout และ checkin objects
- Automatically reset objects on checkin

## สรุป

Type Variables ใน Crystal:
- **T.class**: type ของ class T เอง - ใช้เพื่อ pass types เป็น values
- **typeof(expr)**: compile-time type ของ expression
- **T.new**: สร้าง instance จาก type variable
- **.class**: runtime class ของ object

Use cases:
1. Factory patterns ที่ parameterized
2. Generic algorithms ที่ type-aware
3. Phantom types สำหรับ compile-time guarantees
4. Type-based dispatch systems
