# Part 95: Nilable Types และ Safety ใน Crystal

## บทนำ

Nil safety เป็นหนึ่งในคุณสมบัติที่สำคัญที่สุดของ Crystal ทุก variable ที่อาจเป็น nil ต้องเป็น nilable type (`Type | Nil` หรือ `Type?`) และ compiler จะบังคับให้จัดการ nil ก่อนใช้งาน

## Nilable Types พื้นฐาน

```crystal
# Type? เป็น shorthand สำหรับ Type | Nil
name : String? = nil
age : Int32? = nil

# Assign values
name = "สมชาย"
age = 30

# ใช้งาน
# puts name.upcase  # Error! - อาจเป็น nil
# ต้อง check ก่อน
if name
  puts name.upcase  # => สมชาย (compiler รู้ว่าไม่ nil ใน block นี้)
end

# not_nil! - raise ถ้าเป็น nil
non_nil_name = name.not_nil!
puts non_nil_name.upcase  # ปลอดภัยถ้าแน่ใจว่าไม่ nil
```

## Patterns สำหรับ Nil Handling

### Pattern 1: if guard

```crystal
def find_user(id : Int32) : String?
  users = {1 => "สมชาย", 2 => "สมหญิง"}
  users[id]?
end

# Pattern 1: if guard
if user = find_user(1)
  puts "Found: #{user.upcase}"  # user เป็น String ที่นี่
else
  puts "Not found"
end

# Pattern 2: unless nil
user = find_user(2)
unless user.nil?
  puts "User: #{user}"
end

# Pattern 3: explicit nil check
user2 = find_user(99)
if user2.nil?
  puts "User 99 not found"
else
  puts user2  # String ที่นี่
end
```

### Pattern 2: Safe Navigation Operator

```crystal
struct Address
  getter city : String?
  getter zip : String?

  def initialize(@city = nil, @zip = nil)
  end
end

struct User
  getter name : String
  getter address : Address?

  def initialize(@name, @address = nil)
  end
end

user1 = User.new("สมชาย", Address.new("กรุงเทพ", "10110"))
user2 = User.new("สมหญิง")  # No address

# Safe navigation operator (&.)
puts user1.address&.city     # => "กรุงเทพ"
puts user2.address&.city     # => nil (ไม่ crash!)
puts user1.address&.city&.upcase  # => "กรุงเทพ" (upcase String)
puts user2.address&.city&.upcase  # => nil

# Chain ยาวๆ
city_length = user1.address&.city&.size
puts city_length.inspect  # => 7
```

### Pattern 3: || Default Value

```crystal
def get_config(key : String) : String?
  configs = {"host" => "localhost", "port" => "8080"}
  configs[key]?
end

host = get_config("host") || "default-host"
port = get_config("port") || "3000"
timeout = get_config("timeout") || "30"

puts "#{host}:#{port} (timeout: #{timeout}s)"
# => localhost:8080 (timeout: 30s)
```

### Pattern 4: try block

```crystal
def parse_number(s : String?) : Float64?
  s.try { |str| str.to_f? }
end

result1 = parse_number("3.14")
result2 = parse_number(nil)
result3 = parse_number("invalid")

puts result1.inspect  # => 3.14
puts result2.inspect  # => nil
puts result3.inspect  # => nil
```

## Option-Type Pattern

```crystal
# Implement Option type ด้วย Crystal
abstract class Option(T)
  def self.some(value : T) : Option(T)
    Some(T).new(value)
  end

  def self.none : Option(T)
    None(T).new
  end

  abstract def some? : Bool
  abstract def none? : Bool
  abstract def value_or(default : T) : T

  def map(&block : T -> U) : Option(U) forall U
    if some?
      Option(U).some(block.call(value!))
    else
      Option(U).none
    end
  end

  def flat_map(&block : T -> Option(U)) : Option(U) forall U
    if some?
      block.call(value!)
    else
      Option(U).none
    end
  end

  def filter(&pred : T -> Bool) : Option(T)
    if some? && pred.call(value!)
      self
    else
      Option(T).none
    end
  end

  def or_else(alternative : Option(T)) : Option(T)
    some? ? self : alternative
  end

  abstract def value! : T
end

class Some(T) < Option(T)
  def initialize(@value : T)
  end

  def some? : Bool
    true
  end

  def none? : Bool
    false
  end

  def value! : T
    @value
  end

  def value_or(default : T) : T
    @value
  end

  def to_s : String
    "Some(#{@value})"
  end
end

class None(T) < Option(T)
  def some? : Bool
    false
  end

  def none? : Bool
    true
  end

  def value! : T
    raise "Called value! on None"
  end

  def value_or(default : T) : T
    default
  end

  def to_s : String
    "None"
  end
end

# ใช้งาน
def safe_divide(a : Float64, b : Float64) : Option(Float64)
  if b == 0.0
    Option(Float64).none
  else
    Option(Float64).some(a / b)
  end
end

result = safe_divide(10.0, 2.0)
  .map { |x| x * 2 }
  .filter { |x| x > 5.0 }

puts result              # => Some(10.0)
puts result.value_or(0.0)  # => 10.0

no_result = safe_divide(10.0, 0.0)
  .map { |x| x * 2 }

puts no_result              # => None
puts no_result.value_or(0.0)  # => 0.0
```

## Maybe Monad-like Pattern

```crystal
# Maybe monad pattern สำหรับ chaining operations
class Maybe(T)
  def initialize(@value : T?)
  end

  def self.wrap(value : T?) : Maybe(T)
    new(value)
  end

  def bind(&block : T -> Maybe(U)) : Maybe(U) forall U
    if v = @value
      block.call(v)
    else
      Maybe(U).new(nil)
    end
  end

  def fmap(&block : T -> U) : Maybe(U) forall U
    if v = @value
      Maybe(U).new(block.call(v))
    else
      Maybe(U).new(nil)
    end
  end

  def value_or(default : T) : T
    @value || default
  end

  def to_s : String
    if v = @value
      "Just(#{v})"
    else
      "Nothing"
    end
  end
end

# ใช้งาน
def find_user_name(id : Int32) : Maybe(String)
  users = {1 => "สมชาย", 2 => "สมหญิง"}
  Maybe.wrap(users[id]?)
end

def find_user_email(name : String) : Maybe(String)
  emails = {"สมชาย" => "somchai@example.com"}
  Maybe.wrap(emails[name]?)
end

# Chain operations
result = Maybe.wrap(1)
  .bind { |id| find_user_name(id) }
  .bind { |name| find_user_email(name) }
  .fmap { |email| email.split("@").first }

puts result  # => Just(somchai)

# None propagation
result2 = Maybe.wrap(99)
  .bind { |id| find_user_name(id) }
  .bind { |name| find_user_email(name) }

puts result2  # => Nothing
```

## compact! และ compact_map

```crystal
# compact! - ลบ nil values จาก array
arr_with_nil = [1, nil, 2, nil, 3, nil, 4]
puts arr_with_nil.inspect  # => [1, nil, 2, nil, 3, nil, 4]

# compact - return new array ที่ไม่มี nil
compacted = arr_with_nil.compact
puts compacted.inspect  # => [1, 2, 3, 4]

# compact_map - map แล้ว compact
strings = ["1", "abc", "2", "xyz", "3"]
numbers = strings.compact_map(&.to_i?)
puts numbers.inspect  # => [1, 2, 3]

# ตัวอย่างที่ซับซ้อน
user_ids = ["1", "invalid", "2", "3", "not_a_number", "4"]
valid_ids = user_ids.compact_map { |s|
  if id = s.to_i?
    id if id > 0  # return nil สำหรับ id <= 0
  end
}
puts valid_ids.inspect  # => [1, 2, 3, 4]
```

## Nil-safe Collections

```crystal
# สร้าง nil-safe wrapper สำหรับ collection operations
module NilSafe
  def self.map(arr : Array(T?), &block : T -> U) : Array(U) forall T, U
    arr.compact_map do |item|
      item.try { |v| block.call(v) }
    end
  end

  def self.flat_map(arr : Array(T?), &block : T -> Array(U)?) : Array(U) forall T, U
    result = Array(U).new
    arr.each do |item|
      if item
        block.call(item)&.each { |v| result << v }
      end
    end
    result
  end
end

nullable_numbers = [1, nil, 2, nil, 3]
doubled = NilSafe.map(nullable_numbers) { |n| n * 2 }
puts doubled.inspect  # => [2, 4, 6]
```

## Deep Nil Safety

```crystal
require "json"

# Deep null safety ใน JSON traversal
def safely_get(json : JSON::Any?, *keys : String | Int32) : JSON::Any?
  current = json
  keys.each do |key|
    break if current.nil?
    current = case key
    when String then current.as_h?.[key]?
    when Int32  then current.as_a?.[key]?
    end
  end
  current
end

data = JSON.parse(%({
  "user": {
    "profile": {
      "name": "สมชาย",
      "contact": {
        "email": "test@example.com"
      }
    }
  }
}))

email = safely_get(data, "user", "profile", "contact", "email")
puts email&.as_s  # => test@example.com

missing = safely_get(data, "user", "settings", "theme")
puts missing.inspect  # => nil (ไม่ crash!)
```

## Nil Safety ใน Struct/Class

```crystal
struct UserPreference
  getter theme : String
  getter language : String
  getter notifications_enabled : Bool

  def initialize(
    @theme = "light",
    @language = "th",
    @notifications_enabled = true
  )
  end
end

class User
  getter id : Int32
  getter name : String
  property email : String?
  property phone : String?
  property preferences : UserPreference?

  def initialize(@id, @name)
    @email = nil
    @phone = nil
    @preferences = nil
  end

  # Nil-safe accessors
  def effective_theme : String
    preferences&.theme || "light"
  end

  def contact_info : String
    parts = [] of String
    parts << "Email: #{@email}" if @email
    parts << "Phone: #{@phone}" if @phone
    parts.empty? ? "No contact info" : parts.join(", ")
  end

  def has_contact? : Bool
    !@email.nil? || !@phone.nil?
  end
end

user = User.new(1, "สมชาย")
puts user.effective_theme   # => light (default)
puts user.contact_info      # => No contact info
puts user.has_contact?      # => false

user.email = "somchai@example.com"
user.preferences = UserPreference.new(theme: "dark")
puts user.effective_theme   # => dark
puts user.contact_info      # => Email: somchai@example.com
```

## Defensive Programming กับ Nil

```crystal
# Defensive patterns
class SafeArray(T)
  def initialize(@data : Array(T))
  end

  # Safe first - return nil แทน raise
  def first? : T?
    @data.first?
  end

  # Safe last
  def last? : T?
    @data.last?
  end

  # Safe index
  def [](i : Int32) : T?
    @data[i]?
  end

  # Safe find
  def find?(&block : T -> Bool) : T?
    @data.find { |x| block.call(x) }
  end

  # Safe min/max
  def min? : T?
    @data.min? rescue nil
  end

  def max? : T?
    @data.max? rescue nil
  end
end

arr = SafeArray(Int32).new([3, 1, 4, 1, 5, 9, 2, 6])

puts arr.first?.inspect     # => 3
puts arr.last?.inspect      # => 6
puts arr[100]?.inspect      # => nil
puts arr.find? { |x| x > 7 }.inspect  # => 9
puts arr.max?.inspect       # => 9

empty = SafeArray(Int32).new([] of Int32)
puts empty.first?.inspect   # => nil
puts empty.min?.inspect     # => nil
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Safe Parser
สร้าง safe parser ที่:
- `parse_int?(s : String) : Int32?`
- `parse_float?(s : String) : Float64?`
- `parse_date?(s : String) : Time?`
- Chain ด้วย `try` และ `&.`

### แบบฝึกหัดที่ 2: Null Object Pattern
Implement Null Object pattern:
- `NullUser` ที่ implement interface เดียวกับ `User`
- ไม่ต้องมี nil check ในโค้ด
- เปรียบเทียบ approach สองแบบ

### แบบฝึกหัดที่ 3: Safe Configuration Reader
สร้าง config reader ที่:
- อ่าน config file (JSON/YAML)
- Nil-safe access ทุก level
- Default values ที่ชัดเจน
- Error messages ที่บอกว่า key ไหนขาดไป

### แบบฝึกหัดที่ 4: Database Result Wrapper
สร้าง `DbResult(T)` ที่:
- Wrap nullable database results
- Chain transformations
- Handle not-found vs errors ต่างกัน
- Integrate กับ logging

## สรุป

Nil safety ใน Crystal:
- **Compile-time**: ตรวจจับ potential nil errors ก่อน runtime
- **Explicit**: ต้อง declare nilable types ชัดเจน
- **Safe navigation**: `&.` สำหรับ chaining บน nilable
- **compact/compact_map**: ลบ nil จาก collections

Best practices:
1. ใช้ `&.` แทน explicit nil checks เมื่อทำได้
2. ใช้ `|| default` สำหรับ fallback values
3. ใช้ `try` สำหรับ transformations บน nilable
4. หลีกเลี่ยง `not_nil!` ยกเว้นเมื่อแน่ใจ 100%
5. ใช้ compact_map แทน map แล้ว select
