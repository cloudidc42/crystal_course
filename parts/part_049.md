# Part 49: NamedTuples ใน Crystal

## บทนำ

NamedTuple คือ Tuple ที่ใช้ symbol keys แทน index ทำให้ code อ่านง่ายขึ้น ต่างจาก Hash ตรงที่:
1. **Immutable**: ไม่สามารถเพิ่ม/ลบ/แก้ไข
2. **Fixed structure**: keys และ types รู้ตอน compile time
3. **Type-safe**: แต่ละ key มี type เฉพาะ
4. **Stack allocated**: มี performance ดีกว่า Hash

---

## 49.1 การสร้าง NamedTuple

```crystal
# วิธีที่ 1: NamedTuple literal
person = {name: "Alice", age: 30, city: "Bangkok"}
puts typeof(person)  # => NamedTuple(name: String, age: Int32, city: String)

# วิธีที่ 2: ระบุ type ชัดเจน
config : NamedTuple(host: String, port: Int32) = {host: "localhost", port: 8080}

# วิธีที่ 3: ใน function
def create_point(x : Float64, y : Float64) : NamedTuple(x: Float64, y: Float64)
  {x: x, y: y}
end

point = create_point(3.0, 4.0)
puts point.inspect  # => {x: 3.0, y: 4.0}

# Keys ต้องเป็น symbols
user = {
  first_name: "John",
  last_name: "Doe",
  email: "john@example.com",
  age: 28,
  active: true
}
```

---

## 49.2 การเข้าถึง Values

```crystal
person = {name: "Alice", age: 30, city: "Bangkok"}

# เข้าถึงด้วย symbol
puts person[:name]   # => "Alice"
puts person[:age]    # => 30
puts person[:city]   # => "Bangkok"

# เข้าถึงด้วย method syntax (dot notation)
# Crystal รองรับผ่าน [] เท่านั้น ไม่มี dot notation
# puts person.name  # Error ใน Crystal

# ความปลอดภัย
# puts person[:missing]  # Error ตอน compile time! (ไม่ใช่ runtime)

# เข้าถึงด้วย string (convert)
key = :name
puts person[key]  # => "Alice"

# Type safety: รู้ type ณ compile time
name = person[:name]    # เป็น String
age = person[:age]      # เป็น Int32
puts name.upcase  # => "ALICE" (String method)
puts age + 5      # => 35 (Int32 arithmetic)
```

---

## 49.3 เข้าถึงด้วย []

```crystal
config = {
  host: "api.example.com",
  port: 443,
  timeout: 30.0,
  ssl: true
}

# เข้าถึงด้วย symbol literal
puts config[:host]    # => "api.example.com"
puts config[:port]    # => 443
puts config[:ssl]     # => true

# ใช้ variable symbol
fields = [:host, :port]
fields.each do |field|
  puts "#{field}: #{config[field]}"
end

# to_s
puts config[:host].as(String)  # เมื่อ type เป็น union ต้อง cast
```

---

## 49.4 merge

```crystal
base = {name: "Alice", age: 30}
extra = {city: "Bangkok", email: "alice@example.com"}

# merge สร้าง NamedTuple ใหม่
combined = base.merge(extra)
puts combined.inspect
# => {name: "Alice", age: 30, city: "Bangkok", email: "alice@example.com"}

# merge override ค่าที่ซ้ำ
defaults = {host: "localhost", port: 8080, debug: false}
custom = {host: "api.example.com", debug: true}

final = defaults.merge(custom)
puts final.inspect
# => {host: "api.example.com", port: 8080, debug: true}

# Type ของ merge result
puts typeof(final)
# => NamedTuple(host: String, port: Int32, debug: Bool)
```

---

## 49.5 to_h

```crystal
person = {name: "Alice", age: 30, city: "Bangkok"}

# แปลงเป็น Hash
hash = person.to_h
puts hash.class    # => Hash(Symbol, String | Int32)
puts hash.inspect  # => {:name => "Alice", :age => 30, :city => "Bangkok"}

# แก้ไข hash ได้แล้ว
hash[:email] = "alice@example.com"
hash[:age] = 31

# ใช้ stringify keys
string_hash = person.to_h.transform_keys(&.to_s)
puts string_hash.inspect
# => {"name" => "Alice", "age" => 30, "city" => "Bangkok"}
```

---

## 49.6 each

```crystal
config = {
  host: "localhost",
  port: 8080,
  debug: false,
  max_connections: 100
}

# each กับ key-value pair
config.each do |key, value|
  puts "#{key} = #{value}"
end

# keys
puts config.keys.inspect    # => [:host, :port, :debug, :max_connections]

# values
puts config.values.inspect  # => ["localhost", 8080, false, 100]

# each_with_index
config.each_with_index do |(key, value), index|
  puts "  [#{index}] #{key}: #{value}"
end

# map กับ NamedTuple (คืน Array)
descriptions = config.map { |k, v| "#{k}=#{v}" }
puts descriptions.inspect
```

---

## 49.7 keys และ values

```crystal
person = {name: "Alice", age: 30, city: "Bangkok"}

# keys
puts person.keys.inspect    # => [:name, :age, :city]
puts person.has_key?(:name)  # => true
puts person.has_key?(:email)  # => false

# values
puts person.values.inspect  # => ["Alice", 30, "Bangkok"]

# size
puts person.size  # => 3

# empty?
puts person.empty?     # => false
puts NamedTuple.new.empty?  # => true
```

---

## 49.8 NamedTuple Type Annotation

```crystal
# กำหนด type alias
alias UserRecord = NamedTuple(
  id: Int32,
  name: String,
  email: String,
  age: Int32,
  active: Bool
)

def create_user(id : Int32, name : String, email : String, age : Int32) : UserRecord
  {id: id, name: name, email: email, age: age, active: true}
end

user = create_user(1, "Alice", "alice@example.com", 30)
puts user[:name]    # => "Alice"
puts user[:active]  # => true

# Array of NamedTuples
users : Array(UserRecord) = [
  {id: 1, name: "Alice", email: "alice@example.com", age: 30, active: true},
  {id: 2, name: "Bob", email: "bob@example.com", age: 25, active: false},
  {id: 3, name: "Charlie", email: "charlie@example.com", age: 35, active: true}
]

active_users = users.select { |u| u[:active] }
puts "Active users: #{active_users.map { |u| u[:name] }.join(", ")}"
# => Active users: Alice, Charlie
```

---

## 49.9 NamedTuple กับ Struct

```crystal
# NamedTuple เหมาะกับ lightweight, temporary data
# Struct เหมาะกับ complex data ที่ต้องการ methods

# NamedTuple: simple และ immutable
point = {x: 3.0, y: 4.0}
distance = Math.sqrt(point[:x]**2 + point[:y]**2)
puts distance  # => 5.0

# Struct: มี methods และ encapsulation ดีกว่า
struct Point
  getter x : Float64
  getter y : Float64
  
  def initialize(@x : Float64, @y : Float64)
  end
  
  def distance_to(other : Point) : Float64
    Math.sqrt((x - other.x)**2 + (y - other.y)**2)
  end
  
  def to_named_tuple : NamedTuple(x: Float64, y: Float64)
    {x: @x, y: @y}
  end
end

p1 = Point.new(0.0, 0.0)
p2 = Point.new(3.0, 4.0)
puts p1.distance_to(p2)  # => 5.0
puts p2.to_named_tuple.inspect  # => {x: 3.0, y: 4.0}
```

---

## 49.10 ตัวอย่างในชีวิตจริง

### API Response

```crystal
alias ApiResponse = NamedTuple(
  status: Int32,
  message: String,
  data: Array(Hash(String, String)) | Nil
)

def fetch_users : ApiResponse
  # จำลอง API response
  {
    status: 200,
    message: "Success",
    data: [
      {"id" => "1", "name" => "Alice"},
      {"id" => "2", "name" => "Bob"}
    ]
  }
end

def handle_error : ApiResponse
  {status: 404, message: "Not Found", data: nil}
end

response = fetch_users
if response[:status] == 200
  puts "Users: #{response[:data].try { |d| d.map { |u| u["name"] }.join(", ") }}"
end

error = handle_error
puts "Error: #{error[:message]} (#{error[:status]})"
```

### Form Validation

```crystal
alias ValidationResult = NamedTuple(
  valid: Bool,
  errors: Array(String),
  data: Hash(String, String) | Nil
)

def validate_registration(
  username : String,
  email : String,
  password : String,
  confirm : String
) : ValidationResult
  errors = [] of String
  
  errors << "Username too short" if username.size < 3
  errors << "Invalid email" unless email.includes?("@")
  errors << "Password too short" if password.size < 8
  errors << "Passwords don't match" if password != confirm
  
  if errors.empty?
    {
      valid: true,
      errors: errors,
      data: {"username" => username, "email" => email}
    }
  else
    {valid: false, errors: errors, data: nil}
  end
end

result = validate_registration("alice", "alice@example.com", "SecurePass1", "SecurePass1")

if result[:valid]
  puts "Registration successful!"
  puts "Data: #{result[:data]}"
else
  puts "Validation failed:"
  result[:errors].each { |e| puts "  - #{e}" }
end

bad_result = validate_registration("ab", "notanemail", "short", "different")
puts "\nBad input errors:"
bad_result[:errors].each { |e| puts "  - #{e}" }
```

### Configuration

```crystal
alias DatabaseConfig = NamedTuple(
  host: String,
  port: Int32,
  database: String,
  username: String,
  max_pool_size: Int32,
  timeout: Float64
)

alias AppConfig = NamedTuple(
  database: DatabaseConfig,
  server_port: Int32,
  debug: Bool,
  log_level: String
)

def load_config : AppConfig
  db_config : DatabaseConfig = {
    host: "localhost",
    port: 5432,
    database: "myapp",
    username: "admin",
    max_pool_size: 10,
    timeout: 30.0
  }
  
  {
    database: db_config,
    server_port: 8080,
    debug: false,
    log_level: "info"
  }
end

config = load_config
puts "Server: http://localhost:#{config[:server_port]}"
puts "DB: #{config[:database][:host]}:#{config[:database][:port]}"
puts "Debug: #{config[:debug]}"
puts "Log: #{config[:log_level]}"
```

---

## 49.11 NamedTuple กับ JSON-like Data

```crystal
# Simulate JSON-like nested data
alias JSONValue = String | Int32 | Float64 | Bool | Nil

user_data = {
  id: 1,
  profile: {
    first_name: "Alice",
    last_name: "Smith",
    bio: "Crystal developer"
  },
  settings: {
    theme: "dark",
    notifications: true
  }
}

# เข้าถึง nested
puts user_data[:profile][:first_name]  # => "Alice"
puts user_data[:settings][:theme]      # => "dark"

# แปลงเป็น Hash สำหรับ serialization
profile_hash = user_data[:profile].to_h
puts profile_hash.inspect
```

---

## 49.12 Comparing NamedTuple กับ Hash

```crystal
# Hash: dynamic keys, mutable
hash = {"name" => "Alice", "age" => 30} of String => String | Int32
hash["email"] = "alice@example.com"  # เพิ่มได้
hash["name"] = "Bob"                 # แก้ไขได้

# NamedTuple: fixed keys, immutable
ntuple = {name: "Alice", age: 30}
# ntuple[:email] = ...  # Error! compile time
# ntuple[:name] = "Bob"  # Error! immutable

# Performance: NamedTuple เร็วกว่า Hash มาก
# (allocated on stack, no hash computation)

# ใช้ NamedTuple เมื่อ:
# - โครงสร้างข้อมูลรู้ล่วงหน้า
# - ไม่ต้องการ modify
# - Return type ของ function
# - Configuration records

# ใช้ Hash เมื่อ:
# - Keys ไม่รู้ล่วงหน้า
# - ต้องการเพิ่ม/ลบ keys
# - Dynamic data
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function returns NamedTuple

```crystal
alias Statistics = NamedTuple(
  count: Int32,
  sum: Float64,
  mean: Float64,
  min: Float64,
  max: Float64
)

def calculate_stats(numbers : Array(Float64)) : Statistics
  {
    count: numbers.size,
    sum: numbers.sum,
    mean: numbers.sum / numbers.size,
    min: numbers.min,
    max: numbers.max
  }
end

data = [10.0, 25.0, 15.0, 30.0, 20.0]
stats = calculate_stats(data)

puts "Count: #{stats[:count]}"
puts "Sum: #{stats[:sum]}"
puts "Mean: #{stats[:mean]}"
puts "Min: #{stats[:min]}"
puts "Max: #{stats[:max]}"
```

### แบบฝึกหัดที่ 2: Config Builder

```crystal
alias ServerConfig = NamedTuple(
  host: String,
  port: Int32,
  ssl: Bool,
  max_connections: Int32,
  timeout: Float64
)

def default_config : ServerConfig
  {
    host: "localhost",
    port: 8080,
    ssl: false,
    max_connections: 100,
    timeout: 30.0
  }
end

def production_config : ServerConfig
  default_config.merge({
    host: "0.0.0.0",
    port: 443,
    ssl: true,
    max_connections: 1000
  })
end

puts "Development:"
dev = default_config
puts "  #{dev[:host]}:#{dev[:port]} (ssl: #{dev[:ssl]})"

puts "Production:"
prod = production_config
puts "  #{prod[:host]}:#{prod[:port]} (ssl: #{prod[:ssl]})"
puts "  Max connections: #{prod[:max_connections]}"
```

### แบบฝึกหัดที่ 3: Query Results

```crystal
alias QueryResult = NamedTuple(
  success: Bool,
  rows_affected: Int32,
  rows: Array(NamedTuple(id: Int32, name: String, score: Float64)) | Nil,
  error: String | Nil
)

def execute_query(query : String) : QueryResult
  # จำลองการ query
  if query.starts_with?("SELECT")
    {
      success: true,
      rows_affected: 3,
      rows: [
        {id: 1, name: "Alice", score: 95.5},
        {id: 2, name: "Bob", score: 87.0},
        {id: 3, name: "Charlie", score: 92.3}
      ],
      error: nil
    }
  else
    {
      success: false,
      rows_affected: 0,
      rows: nil,
      error: "Only SELECT queries supported in demo"
    }
  end
end

result = execute_query("SELECT * FROM students")
if result[:success]
  puts "Found #{result[:rows_affected]} rows:"
  result[:rows].try { |rows|
    rows.each { |row| puts "  ##{row[:id]}: #{row[:name]} - #{row[:score]}" }
  }
else
  puts "Error: #{result[:error]}"
end
```

---

## สรุป

NamedTuple ใน Crystal:

| Feature | NamedTuple | Hash | Struct |
|---------|------------|------|--------|
| Keys | Symbol, fixed | Any type | Field names |
| Values | Any type | Any type | Any type |
| Mutable | No | Yes | Yes (property) |
| Size | Fixed | Dynamic | Fixed |
| Memory | Stack | Heap | Stack/Heap |
| Type check | Compile-time | Runtime | Compile-time |
| Methods | No | No | Yes |

**เมื่อใช้ NamedTuple:**
- Return type ของ functions ที่มีหลาย values
- Configuration objects ที่ไม่เปลี่ยนแปลง
- Lightweight records ชั่วคราว
- Type-safe data transfer

**Syntax สำคัญ:**
```crystal
# สร้าง
nt = {key1: value1, key2: value2}

# เข้าถึง
nt[:key1]

# merge
nt2 = nt.merge({key3: value3})

# แปลง
hash = nt.to_h
keys = nt.keys
values = nt.values
```

---

*ต่อไป: Part 50 - Sets*
