# Part 105: Annotations ใน Crystal

## บทนำ

Annotations ใน Crystal เป็น metadata ที่แนบกับ classes, methods, หรือ instance variables โดยใช้ `@[AnnotationName]` syntax คล้ายกับ Java Annotations หรือ Python Decorators

## Annotations พื้นฐาน

```crystal
# สร้าง custom annotation
annotation MyAnnotation; end

# ใช้ annotation
@[MyAnnotation]
class MyClass
  @[MyAnnotation]
  property name : String = ""

  @[MyAnnotation]
  def process : String
    "processing"
  end
end

# Annotation กับ arguments
annotation Validated
  getter required : Bool = true
  getter min_length : Int32?
  getter max_length : Int32?
  getter pattern : String?
end

class UserForm
  @[Validated(required: true, min_length: 3)]
  property username : String = ""

  @[Validated(required: true, pattern: ".+@.+")]
  property email : String = ""

  @[Validated(min_length: 8, max_length: 100)]
  property password : String = ""
end
```

## @[JSON::Field] Annotation

```crystal
require "json"

struct Person
  include JSON::Serializable

  # เปลี่ยนชื่อ field ใน JSON
  @[JSON::Field(key: "full_name")]
  property name : String

  # ข้าม field นี้ใน JSON
  @[JSON::Field(ignore: true)]
  property internal_id : Int32

  # emit nil เป็น null ใน JSON
  @[JSON::Field(emit_null: true)]
  property email : String?

  # Custom converter
  @[JSON::Field(key: "birth_timestamp", converter: Time::EpochConverter)]
  property birthday : Time?

  def initialize(@name, @internal_id = 0, @email = nil, @birthday = nil)
  end
end

person = Person.new("สมชาย ใจดี", internal_id: 99)
json = person.to_json

puts json
# => {"full_name":"สมชาย ใจดี","email":null}
# Note: internal_id is ignored

restored = Person.from_json(json)
puts restored.name  # => สมชาย ใจดี
```

## @[YAML::Field] Annotation

```crystal
require "yaml"

struct Config
  include YAML::Serializable

  @[YAML::Field(key: "server_host")]
  property host : String = "localhost"

  @[YAML::Field(key: "server_port")]
  property port : Int32 = 8080

  @[YAML::Field(ignore: true)]
  property secret_key : String = "secret"

  @[YAML::Field(key: "max_conn")]
  property max_connections : Int32 = 100
end

yaml_str = <<-YAML
server_host: example.com
server_port: 443
max_conn: 50
YAML

config = Config.from_yaml(yaml_str)
puts config.host             # => example.com
puts config.port             # => 443
puts config.max_connections  # => 50
puts config.secret_key       # => secret (ไม่ได้ load จาก YAML)
```

## @[Flags] Annotation

```crystal
# @[Flags] ทำให้ enum เป็น bitmask
@[Flags]
enum Permission
  Read
  Write
  Execute
  Admin
end

puts Permission::Read.value    # => 1
puts Permission::Write.value   # => 2
puts Permission::Execute.value # => 4
puts Permission::Admin.value   # => 8

# Combine permissions
user_perms = Permission::Read | Permission::Write
puts user_perms  # => Read | Write

admin_perms = Permission::All
puts admin_perms  # => Read | Write | Execute | Admin

# Check permissions
puts user_perms.includes?(Permission::Read)    # => true
puts user_perms.includes?(Permission::Execute) # => false
```

## Custom Annotations กับ Macro Introspection

```crystal
# สร้าง custom annotation system
annotation Table
  getter name : String
end

annotation Column
  getter name : String?
  getter primary_key : Bool = false
  getter not_null : Bool = false
  getter default : String?
end

# Macro ที่ read annotations
macro generate_sql_schema
  puts "CREATE TABLE {{@type.annotation(Table).[:name]}} ("
  {% for ivar in @type.instance_vars %}
    {% ann = ivar.annotation(Column) %}
    {% if ann %}
      %col_name = {{ann[:name] || ivar.name.stringify}}
      %col_type = typeof(@{{ivar.name.id}}).to_s.gsub("Int32", "INTEGER").gsub("String", "TEXT").gsub("Float64", "REAL")
      puts "  #{%col_name} #{%col_type}{% if ann[:primary_key] %} PRIMARY KEY{% end %}{% if ann[:not_null] %} NOT NULL{% end %}{% if ann[:default] %} DEFAULT {{ann[:default]}}{% end %},"
    {% end %}
  {% end %}
  puts ");"
end

@[Table(name: "users")]
class User
  @[Column(name: "id", primary_key: true)]
  property id : Int32 = 0

  @[Column(name: "username", not_null: true)]
  property name : String = ""

  @[Column(name: "user_email")]
  property email : String?

  @[Column(not_null: true, default: "0")]
  property score : Float64 = 0.0

  # ไม่มี annotation - ไม่ถูก include
  property internal_data : String = ""

  generate_sql_schema
end
```

## Annotation Usage ใน Macros

```crystal
# อ่าน annotation จาก method
annotation Deprecated
  getter message : String
  getter since : String = "unknown"
  getter replacement : String?
end

annotation ApiEndpoint
  getter path : String
  getter method : String = "GET"
  getter auth_required : Bool = false
end

macro check_deprecated_calls
  {% for method in @type.methods %}
    {% ann = method.annotation(Deprecated) %}
    {% if ann %}
      {% puts "Warning: #{@type.name}##{method.name} is deprecated since #{ann[:since]}: #{ann[:message]}" %}
    {% end %}
  {% end %}
end

class UserController
  @[ApiEndpoint(path: "/users", method: "GET")]
  def index
    "List users"
  end

  @[ApiEndpoint(path: "/users", method: "POST", auth_required: true)]
  def create
    "Create user"
  end

  @[Deprecated(message: "Use #create instead", since: "v2.0", replacement: "create")]
  def add_user
    create
  end

  check_deprecated_calls
end

# Macro ที่ generate routes จาก annotations
macro generate_routes
  puts "Routes:"
  {% for method in @type.methods %}
    {% ann = method.annotation(ApiEndpoint) %}
    {% if ann %}
      puts "  {{ann[:method]}} {{ann[:path]}} -> {{method.name}}{% if ann[:auth_required] %} [AUTH REQUIRED]{% end %}"
    {% end %}
  {% end %}
end

class App
  generate_routes
end
```

## Annotation Inheritance

```crystal
# Annotations ไม่ถูก inherit โดยอัตโนมัติ
# ต้อง re-declare ใน subclasses

annotation Serializable
  getter format : String = "json"
end

@[Serializable(format: "json")]
class BaseModel
end

@[Serializable(format: "xml")]
class XmlModel < BaseModel
end

# Check annotation
{% if BaseModel.annotation(Serializable) %}
  puts "BaseModel format: #{BaseModel.annotation(Serializable)[:format]}"
{% end %}
```

## Built-in Crystal Annotations

```crystal
# @[AlwaysInline] - บอก compiler ให้ inline เสมอ
class MathOps
  @[AlwaysInline]
  def add(a : Int32, b : Int32) : Int32
    a + b
  end
end

# @[NoInline] - ห้าม inline
class BigFunction
  @[NoInline]
  def complex_operation : String
    # Complex code that shouldn't be inlined
    "result"
  end
end

# @[Pure] - บอกว่า method ไม่มี side effects
class Calculator
  @[Pure]
  def square(n : Int32) : Int32
    n * n
  end
end

# @[Extern] - C extern
lib LibC
  @[Extern]
  fun printf(format : UInt8*, ...) : Int32
end

# @[Primitive] - built-in primitive operations
# (ใช้ภายใน Crystal compiler)

# @[Link] สำหรับ C libraries
@[Link("pcre")]
lib LibPCRE
  fun compile(pattern : UInt8*, options : Int32, ...) : Void*
end
```

## Custom Annotation ใน JSON

```crystal
require "json"

# Custom annotation สำหรับ JSON serialization control
annotation JsonConfig
  getter key : String?
  getter skip : Bool = false
  getter transform : String?
end

macro json_serialize
  def to_custom_json : String
    JSON.build do |json|
      json.object do
        {% for ivar in @type.instance_vars %}
          {% ann = ivar.annotation(JsonConfig) %}
          {% if ann %}
            {% if !ann[:skip] %}
              %key = {{ann[:key]}} || {{ivar.name.stringify}}
              json.field %key, @{{ivar.name.id}}
            {% end %}
          {% else %}
            json.field {{ivar.name.stringify}}, @{{ivar.name.id}}
          {% end %}
        {% end %}
      end
    end
  end
end

struct Article
  @[JsonConfig(key: "article_title")]
  property title : String

  property content : String

  @[JsonConfig(skip: true)]
  property internal_notes : String

  @[JsonConfig(key: "author_name")]
  property author : String

  def initialize(@title, @content, @internal_notes = "", @author = "Anonymous")
  end

  json_serialize
end

article = Article.new("Crystal Tips", "Great language!", "Review later", "สมชาย")
puts article.to_custom_json
# => {"article_title":"Crystal Tips","content":"Great language!","author_name":"สมชาย"}
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: ORM Annotations
สร้าง annotation system สำหรับ simple ORM:
- `@[Table(name: "...")]`
- `@[Column(name: "...", type: "...", nullable: false)]`
- `@[PrimaryKey]`, `@[Index]`
- Generate SQL CREATE TABLE statements

### แบบฝึกหัดที่ 2: API Documentation
สร้าง annotation สำหรับ API docs:
- `@[Api::Route(method: "GET", path: "/users")]`
- `@[Api::Param(name: "id", type: "Int32", required: true)]`
- `@[Api::Response(status: 200, schema: "User")]`
- Generate OpenAPI-like spec

### แบบฝึกหัดที่ 3: Validation Framework
สร้าง validation annotation:
- `@[Validate::Required]`
- `@[Validate::Min(value: 0)]`
- `@[Validate::Pattern(regex: ".+@.+")]`
- Macro ที่ generate validate method

### แบบฝึกหัดที่ 4: Event System
ใช้ annotation สำหรับ event handling:
- `@[EventHandler("user.created")]`
- Auto-register handlers
- Type-safe event dispatch

## สรุป

Annotations ใน Crystal:
- **@[AnnotationName]**: กำหนด metadata บน types/fields/methods
- **Custom annotations**: สร้างด้วย `annotation Name; end`
- **Built-in**: JSON::Field, YAML::Field, Flags, AlwaysInline, etc.
- **Macro introspection**: อ่าน annotations ด้วย `.annotation()`

Best practices:
1. ใช้ annotations สำหรับ declarative configuration
2. Combine กับ macros สำหรับ code generation
3. Document annotation parameters ชัดเจน
4. ใช้ meaningful annotation names
