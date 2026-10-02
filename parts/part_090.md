# Part 90: Union Types ขั้นสูง

## บทนำ

Union types ใน Crystal ให้ความยืดหยุ่นในการจัดการค่าที่อาจเป็น type ต่างกัน บทนี้จะลงลึกเกี่ยวกับ patterns การใช้ union types ขั้นสูง

## JSON::Any Union

```crystal
require "json"

# JSON::Any เป็น union ของ JSON types ทั้งหมด
# type JSON::Any = Nil | Bool | Int64 | Float64 | String | Array(Any) | Hash(String, Any)

# อ่าน dynamic JSON
json_str = %({
  "name": "สมชาย",
  "age": 30,
  "score": 95.5,
  "active": true,
  "tags": ["admin", "user"],
  "address": {
    "city": "กรุงเทพ",
    "zip": "10110"
  },
  "metadata": null
})

data = JSON.parse(json_str)

# เข้าถึงค่าแบบ safe
name = data["name"].as_s
age = data["age"].as_i
score = data["score"].as_f

puts "Name: #{name}"
puts "Age: #{age}"
puts "Score: #{score}"

# Nested access
city = data["address"]["city"].as_s
puts "City: #{city}"

# Array access
tags = data["tags"].as_a.map(&.as_s)
puts "Tags: #{tags.join(", ")}"

# Nil check
metadata = data["metadata"]
puts "Metadata is nil: #{metadata.raw.nil?}"

# Safe navigation
unknown = data["unknown"]?.try(&.as_s?) || "default"
puts "Unknown: #{unknown}"
```

## Traversing Complex JSON

```crystal
require "json"

# Helper methods สำหรับ JSON::Any traversal
module JsonHelper
  def self.dig(json : JSON::Any, *keys : String | Int32) : JSON::Any?
    current = json
    keys.each do |key|
      case key
      when String
        current = current[key]? || return nil
      when Int32
        arr = current.as_a? || return nil
        current = arr[key]? || return nil
      end
    end
    current
  end

  def self.to_h(json : JSON::Any) : Hash(String, JSON::Any)
    json.as_h? || Hash(String, JSON::Any).new
  end

  def self.to_a(json : JSON::Any) : Array(JSON::Any)
    json.as_a? || Array(JSON::Any).new
  end
end

complex_json = JSON.parse(%({
  "users": [
    {"id": 1, "name": "Alice", "role": {"name": "admin"}},
    {"id": 2, "name": "Bob", "role": {"name": "user"}}
  ]
}))

# Dig ลึกเข้าไป
first_user_role = JsonHelper.dig(complex_json, "users", 0, "role", "name")
puts first_user_role&.as_s  # => admin

# Iterate users
users = JsonHelper.to_a(complex_json["users"])
users.each do |user|
  user_h = JsonHelper.to_h(user)
  puts "#{user_h["id"]?.try(&.as_i)}: #{user_h["name"]?.try(&.as_s)}"
end
```

## Result(T) Pattern

```crystal
# Generic Result type สำหรับ error handling
abstract class Result(T)
  def self.ok(value : T) : Ok(T)
    Ok(T).new(value)
  end

  def self.error(message : String) : Err(T)
    Err(T).new(message)
  end

  abstract def ok? : Bool
  abstract def error? : Bool
  abstract def value_or(default : T) : T

  def map(&block : T -> U) : Result(U) forall U
    case self
    when Ok(T)
      begin
        Result(U).ok(block.call(self.as(Ok(T)).value))
      rescue e
        Result(U).error(e.message || "Unknown error")
      end
    when Err(T)
      Result(U).error(self.as(Err(T)).message)
    else
      Result(U).error("Unknown result type")
    end
  end

  def flat_map(&block : T -> Result(U)) : Result(U) forall U
    case self
    when Ok(T)
      block.call(self.as(Ok(T)).value)
    when Err(T)
      Result(U).error(self.as(Err(T)).message)
    else
      Result(U).error("Unknown result type")
    end
  end
end

class Ok(T) < Result(T)
  getter value : T

  def initialize(@value : T)
  end

  def ok? : Bool
    true
  end

  def error? : Bool
    false
  end

  def value_or(default : T) : T
    @value
  end

  def to_s : String
    "Ok(#{@value})"
  end
end

class Err(T) < Result(T)
  getter message : String

  def initialize(@message : String)
  end

  def ok? : Bool
    false
  end

  def error? : Bool
    true
  end

  def value_or(default : T) : T
    default
  end

  def to_s : String
    "Err(#{@message})"
  end
end

# ใช้งาน
def divide(a : Float64, b : Float64) : Result(Float64)
  if b == 0.0
    Result(Float64).error("Division by zero")
  else
    Result(Float64).ok(a / b)
  end
end

def sqrt_safe(x : Float64) : Result(Float64)
  if x < 0.0
    Result(Float64).error("Cannot take sqrt of negative number")
  else
    Result(Float64).ok(Math.sqrt(x))
  end
end

# Chaining
result = divide(10.0, 2.0)
  .flat_map { |v| sqrt_safe(v) }
  .map { |v| v.round(4) }

case result
when Ok(Float64)
  puts "Result: #{result.as(Ok(Float64)).value}"
when Err(Float64)
  puts "Error: #{result.as(Err(Float64)).message}"
end

# Error case
bad_result = divide(10.0, 0.0)
  .flat_map { |v| sqrt_safe(v) }

puts bad_result  # => Err(Division by zero)
```

## Error Handling กับ Union Types

```crystal
# สร้าง typed errors
module AppError
  class NotFound < Exception
    getter resource : String
    getter id : String | Int32

    def initialize(@resource, @id)
      super("#{resource} with id #{id} not found")
    end
  end

  class ValidationError < Exception
    getter field : String
    getter reason : String

    def initialize(@field, @reason)
      super("Validation failed for #{field}: #{reason}")
    end
  end

  class DatabaseError < Exception
    getter query : String
    getter cause : String

    def initialize(@query, @cause)
      super("Database error in query '#{query}': #{cause}")
    end
  end

  class AuthError < Exception
    getter user_id : Int32?
    getter action : String

    def initialize(@user_id, @action)
      super("Unauthorized: user #{user_id || "anonymous"} cannot #{action}")
    end
  end
end

# Union ของ error types
alias AppErrors = AppError::NotFound | AppError::ValidationError |
                  AppError::DatabaseError | AppError::AuthError

# ใช้ union ใน return types
def find_user(id : Int32) : Hash(String, String) | AppError::NotFound
  # Simulate database lookup
  users = {
    1 => {"name" => "สมชาย", "email" => "somchai@example.com"},
    2 => {"name" => "สมหญิง", "email" => "somying@example.com"},
  }

  users[id]? || AppError::NotFound.new("User", id)
end

def update_user_email(id : Int32, email : String) : Bool | AppError::NotFound | AppError::ValidationError
  return AppError::ValidationError.new("email", "Invalid email format") unless email.includes?("@")

  case find_user(id)
  when AppError::NotFound then AppError::NotFound.new("User", id)
  else true
  end
end

# Handle errors
case find_user(1)
when Hash(String, String) => user
  puts "Found: #{user["name"]}"
when AppError::NotFound => err
  puts "Error: #{err.message}"
end

case update_user_email(99, "bad-email")
when Bool => success
  puts "Updated: #{success}"
when AppError::NotFound => err
  puts "Not found: #{err.resource} #{err.id}"
when AppError::ValidationError => err
  puts "Validation: #{err.field} - #{err.reason}"
end
```

## Narrowing Complex Unions

```crystal
# Narrowing techniques
def process_value(value : Int32 | String | Float64 | Nil | Array(Int32))
  # Pattern 1: is_a? check
  if value.is_a?(Int32)
    puts "Integer: #{value * 2}"
    return
  end

  # Pattern 2: case/when
  case value
  when String
    puts "String length: #{value.size}"
  when Float64
    puts "Float rounded: #{value.round(2)}"
  when Nil
    puts "Got nil"
  when Array(Int32)
    puts "Array sum: #{value.sum}"
  end
end

process_value(42)
process_value("hello")
process_value(3.14159)
process_value(nil)
process_value([1, 2, 3, 4, 5])

# as? สำหรับ safe cast
def safe_int(value : Int32 | String | Float64) : Int32?
  case value
  when Int32 then value
  when String then value.to_i?
  when Float64 then value.to_i
  end
end

puts safe_int(42)       # => 42
puts safe_int("100")    # => 100
puts safe_int(3.7)      # => 3
puts safe_int("abc").inspect  # => nil
```

## Union Types สำหรับ State Machine

```crystal
# Typed state machine ด้วย union types
module TrafficLight
  struct Red
    def next_state : Yellow
      Yellow.new
    end

    def can_go? : Bool
      false
    end

    def color : String
      "แดง"
    end
  end

  struct Yellow
    def next_state : Green
      Green.new
    end

    def can_go? : Bool
      false
    end

    def color : String
      "เหลือง"
    end
  end

  struct Green
    def next_state : Red
      Red.new
    end

    def can_go? : Bool
      true
    end

    def color : String
      "เขียว"
    end
  end

  alias State = Red | Yellow | Green
end

# ใช้งาน
state : TrafficLight::State = TrafficLight::Red.new

5.times do
  puts "Light: #{state.color}, Can go: #{state.can_go?}"
  state = case state
  when TrafficLight::Red    then state.next_state
  when TrafficLight::Yellow then state.next_state
  when TrafficLight::Green  then state.next_state
  end
end
```

## Discriminated Union Pattern

```crystal
require "json"

# Discriminated union สำหรับ API responses
abstract class ApiResponse
  include JSON::Serializable

  use_json_discriminator "status", {
    "success" => SuccessResponse,
    "error"   => ErrorResponse,
    "loading" => LoadingResponse,
  }
end

class SuccessResponse < ApiResponse
  property status = "success"
  property data : JSON::Any
  property timestamp : Int64

  def initialize(@data, @timestamp = Time.utc.to_unix)
  end
end

class ErrorResponse < ApiResponse
  property status = "error"
  property code : Int32
  property message : String
  property details : Array(String)?

  def initialize(@code, @message, @details = nil)
  end
end

class LoadingResponse < ApiResponse
  property status = "loading"
  property progress : Float64

  def initialize(@progress = 0.0)
  end
end

# Simulate API
def fetch_user(id : Int32) : ApiResponse
  case id
  when 1
    SuccessResponse.new(JSON.parse(%({ "id": 1, "name": "สมชาย" })))
  when -1
    LoadingResponse.new(0.5)
  else
    ErrorResponse.new(404, "User not found", ["Check user ID"])
  end
end

[1, 2, -1].each do |id|
  response = fetch_user(id)
  case response
  when SuccessResponse
    puts "Success: #{response.data["name"]?}"
  when ErrorResponse
    puts "Error #{response.code}: #{response.message}"
  when LoadingResponse
    puts "Loading: #{(response.progress * 100).round}%"
  end
end
```

## Union Types กับ Pattern Matching ขั้นสูง

```crystal
# Advanced pattern matching
struct Point
  getter x : Float64
  getter y : Float64

  def initialize(@x, @y)
  end
end

struct Circle
  getter center : Point
  getter radius : Float64

  def initialize(@center, @radius)
  end

  def area : Float64
    Math::PI * @radius ** 2
  end
end

struct Rectangle
  getter top_left : Point
  getter bottom_right : Point

  def initialize(@top_left, @bottom_right)
  end

  def area : Float64
    width = (@bottom_right.x - @top_left.x).abs
    height = (@bottom_right.y - @top_left.y).abs
    width * height
  end
end

alias Shape = Circle | Rectangle

def describe_shape(shape : Shape) : String
  case shape
  when Circle
    "วงกลมรัศมี #{shape.radius} พื้นที่ #{shape.area.round(2)}"
  when Rectangle
    "สี่เหลี่ยมพื้นที่ #{shape.area.round(2)}"
  end
end

shapes : Array(Shape) = [
  Circle.new(Point.new(0.0, 0.0), 5.0),
  Rectangle.new(Point.new(0.0, 0.0), Point.new(4.0, 3.0)),
  Circle.new(Point.new(1.0, 1.0), 2.5),
]

shapes.each do |shape|
  puts describe_shape(shape)
end

total_area = shapes.sum do |shape|
  case shape
  when Circle    then shape.area
  when Rectangle then shape.area
  end
end

puts "Total area: #{total_area.round(2)}"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Extended Result Type
สร้าง `Result(T, E)` ที่:
- รองรับ multiple error types
- มี `recover` method
- มี `tap` สำหรับ side effects
- ทำงานกับ collections

### แบบฝึกหัดที่ 2: JSON Schema Validator
สร้าง validator ที่:
- รับ JSON::Any และ schema
- Return union ของ validated data หรือ validation errors
- รองรับ nested schemas

### แบบฝึกหัดที่ 3: Command Pattern
ใช้ union types สำหรับ command system:
- Commands ต่างๆ เช่น CreateUser, UpdateEmail, DeleteAccount
- ผ่าน processor ที่ handle แต่ละ command
- Return typed results

### แบบฝึกหัดที่ 4: Expression Evaluator
สร้าง expression tree:
- Node types: Number, Add, Subtract, Multiply, Divide, Variable
- Evaluator ที่ traverse tree
- Error handling สำหรับ division by zero, undefined variables

## สรุป

Union types ขั้นสูงใน Crystal:
- **Expressive**: แทนความหมาย domain ได้ชัดเจน
- **Exhaustive**: compiler บังคับให้ handle ทุก case
- **Type-safe**: ป้องกัน invalid state ณ compile time
- **Composable**: รวมกับ pattern matching ได้ดี

Patterns ที่ใช้บ่อย:
1. `Result(T)` สำหรับ error handling
2. Discriminated unions สำหรับ polymorphism
3. State machine ด้วย union ของ state types
4. API response types
