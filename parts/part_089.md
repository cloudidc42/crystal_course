# Part 89: Type Aliases ใน Crystal

## บทนำ

Type aliases ใน Crystal ช่วยให้เราตั้งชื่อที่สื่อความหมายให้กับ types ที่ซับซ้อน ทำให้โค้ดอ่านง่ายขึ้นและลด duplication

## Type Alias พื้นฐาน

```crystal
# alias ชื่อง่ายๆ
alias Filename = String
alias UserId = Int32
alias Score = Float64
alias Tags = Array(String)

def read_file(path : Filename) : String
  File.read(path)
rescue File::NotFoundError
  ""
end

def get_user_score(user_id : UserId) : Score
  # mock data
  user_id * 1.5
end

# ใช้งาน
path : Filename = "config.json"
user_id : UserId = 42
score : Score = get_user_score(user_id)
tags : Tags = ["crystal", "programming", "backend"]

puts "Score for user #{user_id}: #{score}"
puts "Tags: #{tags.join(", ")}"
```

## Type Alias สำหรับ Complex Types

```crystal
require "json"

# Alias สำหรับ Hash types ที่ซับซ้อน
alias Config = Hash(String, String | Int32 | Bool | Array(String))
alias JsonObject = Hash(String, JSON::Any)
alias Callback = Proc(String, Nil)
alias ErrorHandler = Proc(Exception, Nil)

# ใช้งาน
config : Config = {
  "host"     => "localhost",
  "port"     => 8080,
  "debug"    => true,
  "features" => ["auth", "cache", "log"],
}

puts config["host"]   # => localhost
puts config["port"]   # => 8080

# Alias สำหรับ Proc types
def register_callback(cb : Callback)
  cb.call("Event triggered")
end

my_callback : Callback = ->(msg : String) { puts "Received: #{msg}"; nil }
register_callback(my_callback)
```

## Type Alias กับ Generic Types

```crystal
# Alias สำหรับ generic types
alias StringArray = Array(String)
alias IntSet = Set(Int32)
alias StringMap(V) = Hash(String, V)

# ใช้งาน generic alias
names : StringArray = ["สมชาย", "สมหญิง", "สมศักดิ์"]
ids : IntSet = Set(Int32).new([1, 2, 3, 4, 5])
data : StringMap(Int32) = {"a" => 1, "b" => 2, "c" => 3}

puts names.first   # => สมชาย
puts ids.size      # => 5
puts data["a"]     # => 1

# Alias สำหรับ tuple types
alias Coordinates = Tuple(Float64, Float64)
alias RGB = Tuple(UInt8, UInt8, UInt8)
alias NameAge = Tuple(String, Int32)

origin : Coordinates = {0.0, 0.0}
red : RGB = {255_u8, 0_u8, 0_u8}
person : NameAge = {"สมชาย", 30}

puts "Origin: #{origin}"
puts "Red: #{red}"
puts "Name: #{person[0]}, Age: #{person[1]}"
```

## Alias สำหรับ Union Types

```crystal
# Union type alias
alias Number = Int32 | Int64 | Float32 | Float64
alias StringOrInt = String | Int32
alias Nullable(T) = T | Nil
alias Result = String | Exception

def parse_value(s : String) : StringOrInt
  if s.starts_with?("\"")
    s[1..-2]  # strip quotes
  else
    s.to_i
  end
rescue
  s
end

val1 = parse_value("\"hello\"")
val2 = parse_value("42")

case val1
when String then puts "String: #{val1}"
when Int32  then puts "Int: #{val1}"
end

case val2
when String then puts "String: #{val2}"
when Int32  then puts "Int: #{val2}"
end
```

## Recursive Type Aliases ด้วย Forward Declarations

```crystal
# Forward declaration สำหรับ recursive types
# Crystal ต้องการให้ประกาศ type ก่อนถ้าจะ reference ตัวเอง

# JSON-like type
alias JsonValue = Nil | Bool | Int64 | Float64 | String | JsonArray | JsonObject

# ต้องใช้ class/struct สำหรับ recursive aliases
class JsonArray < Array(JsonValue)
end

class JsonObject < Hash(String, JsonValue)
end

# ใช้งาน
obj = JsonObject.new
obj["name"] = "Crystal"
obj["version"] = 1_i64
obj["stable"] = true
obj["features"] = JsonArray.new.tap do |arr|
  arr << "fast"
  arr << "safe"
  arr << "concurrent"
end

puts obj["name"].inspect     # => "Crystal"
puts obj["version"].inspect  # => 1
puts obj["features"].inspect # => ["fast", "safe", "concurrent"]
```

## Type Alias ใน Module

```crystal
module HTTP
  alias Headers = Hash(String, String)
  alias QueryParams = Hash(String, String | Array(String))
  alias StatusCode = Int32
  alias Body = String | Bytes | Nil

  struct Response
    property status : StatusCode
    property headers : Headers
    property body : Body

    def initialize(@status, @headers = Headers.new, @body = nil)
    end

    def ok? : Bool
      @status >= 200 && @status < 300
    end
  end

  struct Request
    property method : String
    property path : String
    property headers : Headers
    property params : QueryParams
    property body : Body

    def initialize(@method, @path, @headers = Headers.new,
                   @params = QueryParams.new, @body = nil)
    end
  end
end

request = HTTP::Request.new("GET", "/api/users")
request.headers["Authorization"] = "Bearer token123"
request.params["page"] = "1"
request.params["tags"] = ["crystal", "backend"]

response = HTTP::Response.new(200, {"Content-Type" => "application/json"}, "{}")
puts response.ok?  # => true
```

## Alias สำหรับ Callback Patterns

```crystal
# Event system ด้วย type aliases
alias EventListener(T) = T -> Nil
alias AsyncCallback(T) = T -> Nil  # ในระบบที่มี async

module EventBus
  alias Handler = JSON::Any -> Nil

  @@listeners = Hash(String, Array(Handler)).new

  def self.on(event : String, &handler : Handler)
    @@listeners[event] ||= Array(Handler).new
    @@listeners[event] << handler
  end

  def self.emit(event : String, data : JSON::Any)
    @@listeners[event]?.try &.each(&.call(data))
  end
end

EventBus.on("user.created") do |data|
  puts "New user: #{data["name"]?}"
end

EventBus.on("user.created") do |data|
  puts "Sending welcome email to: #{data["email"]?}"
end

EventBus.emit("user.created", JSON.parse(%({ "name": "สมชาย", "email": "test@example.com" })))
```

## Complex Type Alias Patterns

```crystal
# State machine ด้วย type aliases
alias StateTransition(S, E) = Proc(S, E, S)
alias Guard(S, E) = Proc(S, E, Bool)
alias Action(S, E) = Proc(S, E, Nil)

enum OrderState
  Pending
  Confirmed
  Shipped
  Delivered
  Cancelled
end

enum OrderEvent
  Confirm
  Ship
  Deliver
  Cancel
end

class OrderStateMachine
  alias Transition = StateTransition(OrderState, OrderEvent)

  @state : OrderState
  @transitions : Hash(Tuple(OrderState, OrderEvent), Transition)

  def initialize(@state = OrderState::Pending)
    @transitions = Hash(Tuple(OrderState, OrderEvent), Transition).new
    setup_transitions
  end

  def current_state : OrderState
    @state
  end

  def trigger(event : OrderEvent) : Bool
    key = {@state, event}
    if transition = @transitions[key]?
      @state = transition.call(@state, event)
      true
    else
      false
    end
  end

  private def setup_transitions
    add_transition(OrderState::Pending, OrderEvent::Confirm, OrderState::Confirmed)
    add_transition(OrderState::Pending, OrderEvent::Cancel, OrderState::Cancelled)
    add_transition(OrderState::Confirmed, OrderEvent::Ship, OrderState::Shipped)
    add_transition(OrderState::Confirmed, OrderEvent::Cancel, OrderState::Cancelled)
    add_transition(OrderState::Shipped, OrderEvent::Deliver, OrderState::Delivered)
  end

  private def add_transition(from : OrderState, event : OrderEvent, to : OrderState)
    @transitions[{from, event}] = ->(state : OrderState, ev : OrderEvent) { to }
  end
end

order = OrderStateMachine.new
puts order.current_state  # => Pending

order.trigger(OrderEvent::Confirm)
puts order.current_state  # => Confirmed

order.trigger(OrderEvent::Ship)
puts order.current_state  # => Shipped

order.trigger(OrderEvent::Deliver)
puts order.current_state  # => Delivered

puts order.trigger(OrderEvent::Cancel)  # => false (ไม่สามารถ cancel ที่ Delivered)
```

## Type Alias สำหรับ Builder Pattern

```crystal
# Builder pattern ด้วย type aliases
alias BuilderStep(T, R) = Proc(T, R)

class QueryBuilder
  alias Condition = String
  alias OrderBy = Tuple(String, String)  # (field, direction)
  alias FieldList = Array(String)

  @table : String
  @conditions : Array(Condition)
  @order : Array(OrderBy)
  @fields : FieldList
  @limit : Int32?
  @offset : Int32?

  def initialize(@table)
    @conditions = Array(Condition).new
    @order = Array(OrderBy).new
    @fields = FieldList.new
    @limit = nil
    @offset = nil
  end

  def select(*fields : String) : self
    @fields = fields.to_a
    self
  end

  def where(condition : Condition) : self
    @conditions << condition
    self
  end

  def order_by(field : String, direction = "ASC") : self
    @order << {field, direction}
    self
  end

  def limit(@limit : Int32) : self
    self
  end

  def offset(@offset : Int32) : self
    self
  end

  def build : String
    parts = ["SELECT"]

    if @fields.empty?
      parts << "*"
    else
      parts << @fields.join(", ")
    end

    parts << "FROM #{@table}"

    unless @conditions.empty?
      parts << "WHERE #{@conditions.join(" AND ")}"
    end

    unless @order.empty?
      order_parts = @order.map { |f, d| "#{f} #{d}" }
      parts << "ORDER BY #{order_parts.join(", ")}"
    end

    @limit.try { |l| parts << "LIMIT #{l}" }
    @offset.try { |o| parts << "OFFSET #{o}" }

    parts.join(" ")
  end
end

query = QueryBuilder.new("users")
  .select("id", "name", "email")
  .where("age > 18")
  .where("active = true")
  .order_by("created_at", "DESC")
  .limit(10)
  .offset(20)
  .build

puts query
# => SELECT id, name, email FROM users WHERE age > 18 AND active = true ORDER BY created_at DESC LIMIT 10 OFFSET 20
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Domain Types
สร้าง type aliases สำหรับ e-commerce domain:
- `ProductId`, `CategoryId`, `UserId` สำหรับ ID types
- `Price`, `Quantity`, `Discount` สำหรับ numeric types
- `ProductCatalog`, `ShoppingCart`, `OrderHistory` สำหรับ complex types

### แบบฝึกหัดที่ 2: Function Type Aliases
สร้าง type aliases สำหรับ functional programming:
- `Predicate(T)` = T -> Bool
- `Transform(A, B)` = A -> B
- `Reducer(A, B)` = A, B -> A
- ใช้ใน higher-order functions

### แบบฝึกหัดที่ 3: Config System
สร้าง hierarchical config system ด้วย type aliases:
- `ConfigValue` = scalar types union
- `ConfigSection` = Hash ของ ConfigValue
- `ConfigTree` = recursive nested config

### แบบฝึกหัดที่ 4: Type-Safe Units
ใช้ type aliases เพื่อ prevent unit mixing:
- `Meters`, `Kilometers`, `Miles`
- Functions ที่ convert ระหว่าง units
- Arithmetic ที่ type-safe

## สรุป

Type aliases ใน Crystal มีประโยชน์หลัก:
- **Readability**: ทำให้โค้ดอ่านง่ายขึ้นด้วยชื่อที่มีความหมาย
- **DRY**: ลด duplication ของ type signatures ที่ซับซ้อน
- **Domain Modeling**: แสดง domain concepts ชัดเจน
- **Refactoring**: เปลี่ยน underlying type ได้จากที่เดียว

ข้อจำกัด:
- Type alias เป็นแค่ชื่ออื่นของ same type ไม่ใช่ distinct type
- ไม่ให้ type safety เต็มรูปแบบ (Int32 ยังใส่ใน UserId alias ได้)
- สำหรับ distinct types ที่แข็งแกร่งกว่า ควรใช้ struct wrapper
