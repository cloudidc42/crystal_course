# Part 108: Code Generation

## บทนำ

Code generation ด้วย macros ช่วยให้เราสร้าง repetitive code อัตโนมัติ ลด boilerplate และ ensure consistency ทั่วทั้ง codebase

## Generate Repetitive Code

```crystal
# สร้าง accessor methods สำหรับ collections
macro define_collection_methods(name, type)
  @{{name.id}} = Array({{type.id}}).new

  def add_{{name.id}}(item : {{type.id}}) : self
    @{{name.id}} << item
    self
  end

  def remove_{{name.id}}(item : {{type.id}}) : Bool
    @{{name.id}}.delete(item) != nil
  end

  def {{name.id}} : Array({{type.id}})
    @{{name.id}}.dup
  end

  def has_{{name.id}}?(item : {{type.id}}) : Bool
    @{{name.id}}.includes?(item)
  end

  def {{name.id}}_count : Int32
    @{{name.id}}.size
  end

  def clear_{{name.id}}! : self
    @{{name.id}}.clear
    self
  end
end

class BlogPost
  property title : String
  property content : String

  define_collection_methods :tags, String
  define_collection_methods :categories, String

  def initialize(@title, @content)
  end
end

post = BlogPost.new("Crystal Tips", "Great language!")
post.add_tags("crystal").add_tags("programming").add_tags("tutorial")
post.add_categories("tech").add_categories("coding")

puts post.tags.inspect         # => ["crystal", "programming", "tutorial"]
puts post.tags_count           # => 3
puts post.has_tags?("crystal") # => true
post.remove_tags("tutorial")
puts post.tags.inspect         # => ["crystal", "programming"]
```

## Generate จาก Type Information

```crystal
# Auto-generate comparison methods จาก type info
macro auto_comparable
  include Comparable({{@type.name.id}})

  def <=>(other : {{@type.name.id}}) : Int32
    {% first_ivar = @type.instance_vars.first %}
    @{{first_ivar.name.id}} <=> other.{{first_ivar.name.id}}
  end
end

macro auto_hashable
  def hash(hasher)
    {% for ivar in @type.instance_vars %}
      @{{ivar.name.id}}.hash(hasher)
    {% end %}
    hasher
  end
end

macro auto_equality
  def ==(other : {{@type.name.id}}) : Bool
    {% for ivar in @type.instance_vars %}
      return false unless @{{ivar.name.id}} == other.{{ivar.name.id}}
    {% end %}
    true
  end
end

struct Point
  getter x : Float64
  getter y : Float64

  def initialize(@x, @y)
  end

  auto_comparable
  auto_equality
  auto_hashable
end

p1 = Point.new(1.0, 2.0)
p2 = Point.new(1.0, 2.0)
p3 = Point.new(3.0, 4.0)

puts p1 == p2  # => true
puts p1 == p3  # => false
puts p1 < p3   # => true (x comparison)

points = [p3, p1, p2]
puts points.sort.map { |p| "(#{p.x}, #{p.y})" }.inspect
```

## Generate SQL CRUD

```crystal
# Generate CRUD methods สำหรับ data models
annotation Table
  getter name : String
end

annotation Column
  getter name : String?
  getter primary_key : Bool = false
end

macro generate_crud
  # Generate CREATE
  def self.create(**fields) : String
    field_names = [] of String
    field_values = [] of String

    {% for ivar in @type.instance_vars %}
      {% ann = ivar.annotation(Column) %}
      {% if ann && !ann[:primary_key] %}
        field_names << {{ann[:name] || ivar.name.stringify}}
        # In real code, would escape values
        field_values << fields[{{ivar.name.symbolize}}].to_s
      {% end %}
    {% end %}

    "INSERT INTO #{table_name} (#{field_names.join(", ")}) VALUES (#{field_values.map { |v| "'#{v}'" }.join(", ")})"
  end

  # Generate SELECT
  def self.find(id : Int32) : String
    "SELECT * FROM #{table_name} WHERE id = #{id}"
  end

  def self.all : String
    "SELECT * FROM #{table_name}"
  end

  # Generate UPDATE
  def self.update(id : Int32, **fields) : String
    sets = fields.map { |k, v| "#{k} = '#{v}'" }.join(", ")
    "UPDATE #{table_name} SET #{sets} WHERE id = #{id}"
  end

  # Generate DELETE
  def self.delete(id : Int32) : String
    "DELETE FROM #{table_name} WHERE id = #{id}"
  end

  def self.table_name : String
    {% ann = @type.annotation(Table) %}
    {% if ann %}
      {{ann[:name]}}
    {% else %}
      {{@type.name.stringify.downcase}}
    {% end %}
  end
end

@[Table(name: "users")]
class User
  @[Column(primary_key: true)]
  property id : Int32 = 0

  @[Column]
  property name : String = ""

  @[Column]
  property email : String = ""

  generate_crud
end

puts User.all
# => SELECT * FROM users

puts User.find(42)
# => SELECT * FROM users WHERE id = 42

puts User.create(name: "สมชาย", email: "test@example.com")
# => INSERT INTO users (name, email) VALUES ('สมชาย', 'test@example.com')

puts User.update(42, name: "สมหญิง")
# => UPDATE users SET name = 'สมหญิง' WHERE id = 42

puts User.delete(42)
# => DELETE FROM users WHERE id = 42
```

## Generate REST Client

```crystal
# Generate REST client methods ด้วย macros
annotation Endpoint
  getter path : String
  getter method : String = "GET"
  getter body_param : String?
end

macro generate_rest_client
  {% for method in @type.class.methods %}
    {% ann = method.annotation(Endpoint) %}
    {% if ann %}
      def self.{{method.name}}_impl(**params) : String
        path = {{ann[:path]}}
        method = {{ann[:method]}}

        # Replace path params
        params.each do |key, val|
          path = path.gsub(":#{key}", val.to_s)
        end

        "#{method} #{path}"
      end
    {% end %}
  {% end %}
end

class UserApi
  # In real code, these would make HTTP requests
  @[Endpoint(path: "/users", method: "GET")]
  def self.list : String
    "GET /users"
  end

  @[Endpoint(path: "/users/:id", method: "GET")]
  def self.find(id : Int32) : String
    "GET /users/#{id}"
  end

  @[Endpoint(path: "/users", method: "POST")]
  def self.create(name : String, email : String) : String
    "POST /users {name: #{name}, email: #{email}}"
  end
end

puts UserApi.list
puts UserApi.find(42)
puts UserApi.create("สมชาย", "test@example.com")
```

## Generate Serialization Code

```crystal
# Generate custom serialization ไม่ต้องการ JSON::Serializable
macro generate_serialization
  def to_json_manual : String
    parts = [] of String
    {% for ivar in @type.instance_vars %}
      val = @{{ivar.name.id}}
      json_val = case val
      when String  then "\"#{val.gsub("\"", "\\\"")}\""
      when Int32, Int64, Float64 then val.to_s
      when Bool    then val.to_s
      when Nil     then "null"
      else          "\"#{val}\""
      end
      parts << "\"{{ivar.name}}\": #{json_val}"
    {% end %}
    "{#{parts.join(", ")}}"
  end

  def self.from_json_manual(json_str : String) : self
    require "json"
    data = JSON.parse(json_str)
    obj = allocate
    {% for ivar in @type.instance_vars %}
      # simplified - real version would handle types properly
      if val = data[{{ivar.name.stringify}}]?
        obj.@{{ivar.name.id}} = val.raw.as(typeof(obj.@{{ivar.name.id}}))
      end
    {% end %}
    obj
  end
end

struct SimpleConfig
  @host : String
  @port : Int32
  @debug : Bool

  def initialize(@host = "localhost", @port = 8080, @debug = false)
  end

  generate_serialization
end

config = SimpleConfig.new("example.com", 443, true)
json = config.to_json_manual
puts json  # => {"host": "example.com", "port": 443, "debug": true}
```

## @[Primitive] Macros Pattern

```crystal
# Simulate @[Primitive] pattern - generating low-level operations
macro define_bit_operations(type)
  struct {{type.id}}BitOps
    def initialize(@value : {{type.id}})
    end

    def bit(n : Int32) : Bool
      (@value >> n) & 1 == 1
    end

    def set_bit(n : Int32) : {{type.id}}
      @value | (1 << n).to_{{type.id.downcase}}
    end

    def clear_bit(n : Int32) : {{type.id}}
      @value & ~(1 << n).to_{{type.id.downcase}}
    end

    def toggle_bit(n : Int32) : {{type.id}}
      @value ^ (1 << n).to_{{type.id.downcase}}
    end

    def popcount : Int32
      count = 0
      v = @value
      while v != 0
        count += 1 if v & 1 == 1
        v >>= 1
      end
      count
    end
  end
end

define_bit_operations UInt8
define_bit_operations UInt32

ops = UInt8BitOps.new(0b10110100_u8)
puts ops.bit(2)        # => true
puts ops.bit(0)        # => false
puts ops.popcount      # => 4 (number of 1 bits)

# Test bit manipulation
val = 0b00000000_u8
ops2 = UInt8BitOps.new(val)
puts ops2.set_bit(3).to_s(2).rjust(8, '0')   # => 00001000
puts ops2.set_bit(7).to_s(2).rjust(8, '0')   # => 10000000
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ORM Generator
สร้าง full ORM code generator:
- Migrations ด้วยแค่ class definitions
- Relationship macros (has_many, belongs_to)
- Query interface

### แบบฝึกหัดที่ 2: API Client Generator
สร้าง REST API client generator:
- Generate ด้วย OpenAPI spec
- Type-safe parameters
- Error handling

### แบบฝึกหัดที่ 3: Event Sourcing
Generate event sourcing boilerplate:
- Command และ Event types
- Aggregate patterns
- Event store interface

### แบบฝึกหัดที่ 4: Protocol Buffer-like
สร้าง simple binary serializer:
- Define schema ด้วย macros
- Generate encode/decode
- Versioned fields

## สรุป

Code Generation ใน Crystal:
- **Type info**: ใช้ `@type.instance_vars`, `@type.methods`
- **Annotations**: เพิ่ม metadata สำหรับ generator
- **Method generation**: สร้าง consistent APIs
- **SQL/JSON generation**: declarative schemas

Key benefits:
1. ลด boilerplate 90%+
2. Ensure consistency ทั่ว codebase
3. Easy to change ที่เดียว → propagate ทั่ว
4. Zero runtime overhead (code ถูก generate ณ compile time)
