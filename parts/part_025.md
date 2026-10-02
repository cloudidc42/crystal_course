# Part 025: Named Arguments

## บทนำ

**Named Arguments** (หรือ Keyword Arguments) คือการระบุชื่อ parameter เมื่อเรียก method ทำให้โค้ดอ่านง่ายขึ้นมาก โดยเฉพาะ method ที่มี parameter จำนวนมาก หรือ parameter ที่มี type เหมือนกันหลายตัว

---

## 1. การเรียก Method ด้วย Named Arguments

```crystal
# method ปกติ
def connect(host : String, port : Int32, ssl : Bool) : String
  "#{ssl ? "https" : "http"}://#{host}:#{port}"
end

# เรียกแบบ positional (ลำดับสำคัญ)
puts connect("localhost", 5432, false)

# เรียกแบบ named (ลำดับไม่สำคัญ)
puts connect(host: "localhost", port: 5432, ssl: false)
puts connect(ssl: false, port: 5432, host: "localhost")  # ลำดับต่างกัน - ผลเหมือนกัน
```

### Named Arguments ทำให้โค้ดอ่านง่าย

```crystal
# ยากอ่าน: บูลีนที่ไม่รู้ว่าหมายถึงอะไร
send_email("alice@example.com", "subject", "body", true, false, true)

# ง่ายอ่าน: รู้ทันทีว่าแต่ละค่าคืออะไร
send_email(
  to: "alice@example.com",
  subject: "Hello",
  body: "Hi there!",
  html: true,
  urgent: false,
  cc_manager: true
)
```

---

## 2. ความยืดหยุ่นของลำดับ

```crystal
def create_user(
  name : String,
  email : String,
  age : Int32,
  role : String = "user"
) : String
  "User: #{name}, #{email}, age #{age}, role: #{role}"
end

# named args - ลำดับใดก็ได้
puts create_user(name: "Alice", email: "alice@example.com", age: 30)
puts create_user(age: 25, name: "Bob", email: "bob@example.com", role: "admin")
puts create_user(email: "charlie@example.com", name: "Charlie", age: 28)
```

---

## 3. ผสม Positional และ Named Arguments

```crystal
def configure(app_name : String, version : String, debug : Bool = false, verbose : Bool = false)
  puts "#{app_name} v#{version} (debug: #{debug}, verbose: #{verbose})"
end

# positional ก่อน named ที่เหลือ
configure("MyApp", "1.0")
configure("MyApp", "1.0", debug: true)
configure("MyApp", "1.0", verbose: true, debug: true)

# ไม่ได้: named ก่อน positional
# configure(debug: true, "MyApp", "1.0")  # Error!
```

### ลำดับที่ถูกต้อง

```crystal
def build_request(
  method : String,
  url : String,
  body : String? = nil,
  timeout : Int32 = 30,
  retries : Int32 = 3
)
  puts "#{method} #{url} (body: #{body.inspect}, timeout: #{timeout}, retries: #{retries})"
end

# ถูกต้อง: positional ก่อน
build_request("GET", "https://api.example.com/users")
build_request("POST", "https://api.example.com/users", body: "{\"name\": \"Alice\"}")
build_request("GET", "https://api.example.com", timeout: 60, retries: 5)

# ถูกต้อง: named ทั้งหมด
build_request(method: "DELETE", url: "https://api.example.com/users/1")
```

---

## 4. Double Splat **kwargs

`**` รับ named arguments จำนวนไม่จำกัด:

```crystal
# **kwargs รับเป็น NamedTuple
def render(**options)
  options.each do |key, value|
    puts "  #{key}: #{value}"
  end
end

render(color: "red", size: 14, bold: true, italic: false)
# color: red
# size: 14
# bold: true
# italic: false
```

### **kwargs พร้อม Type Restriction

```crystal
# กำหนด type ของ values
def set_config(**settings : String)
  settings.each do |key, value|
    puts "config.#{key} = #{value}"
  end
end

set_config(
  database_url: "postgresql://localhost/mydb",
  redis_url: "redis://localhost:6379",
  secret_key: "super-secret-key"
)
```

### **kwargs ผสมกับ Named Parameters ปกติ

```crystal
def create_element(
  tag : String,
  content : String = "",
  **attributes
) : String
  attr_str = attributes.map { |k, v| " #{k.to_s.gsub("_", "-")}=\"#{v}\"" }.join
  if content.empty?
    "<#{tag}#{attr_str} />"
  else
    "<#{tag}#{attr_str}>#{content}</#{tag}>"
  end
end

puts create_element("h1", "Welcome")
puts create_element("a", "Click", href: "/home", class_name: "nav-link", target: "_blank")
puts create_element("input", type: "text", placeholder: "Enter name", required: "true")
```

---

## 5. NamedTuple

`NamedTuple` เป็น struct ที่มี named fields ใช้สำหรับ pass หลาย named values:

```crystal
# สร้าง NamedTuple
person = {name: "Alice", age: 30, city: "Bangkok"}
puts person.class  # => NamedTuple(name: String, age: Int32, city: String)

# access ด้วย symbol
puts person[:name]  # => Alice
puts person[:age]   # => 30

# access ด้วย method-like syntax (compile-time)
puts person[:city]  # => Bangkok
```

### NamedTuple เป็น Parameter

```crystal
def greet_from_tuple(info : NamedTuple(name: String, greeting: String)) : String
  "#{info[:greeting]}, #{info[:name]}!"
end

data = {name: "Bob", greeting: "Hello"}
puts greet_from_tuple(data)
# => Hello, Bob!
```

### NamedTuple เป็น Return Value

```crystal
def get_server_info : NamedTuple(
  host: String,
  port: Int32,
  protocol: String,
  uptime: Float64
)
  {
    host: "server01.example.com",
    port: 443,
    protocol: "HTTPS",
    uptime: 99.97,
  }
end

info = get_server_info
puts "Server: #{info[:host]}:#{info[:port]}"
puts "Protocol: #{info[:protocol]}"
puts "Uptime: #{info[:uptime]}%"
```

---

## 6. Forwarding Named Arguments

```crystal
# ส่ง **kwargs ต่อไปยัง method อื่น
def create_connection(**options)
  # forward options ไปยัง DatabaseConnection
  puts "Creating connection with: #{options}"
end

def setup_database(**database_options)
  puts "Setting up database..."
  create_connection(**database_options)  # forward ด้วย **
end

setup_database(
  host: "localhost",
  port: "5432",
  database: "myapp",
  user: "postgres"
)
```

### Wrapper Method Pattern

```crystal
def log_and_call(**options)
  puts "Calling with options: #{options.keys.join(", ")}"
  configure(**options)
end

def configure(
  debug : Bool = false,
  verbose : Bool = false,
  log_level : String = "INFO"
) : String
  "debug=#{debug}, verbose=#{verbose}, log_level=#{log_level}"
end

result = log_and_call(debug: true, log_level: "DEBUG")
puts result
```

---

## 7. Named Arguments กับ Default Values

```crystal
# Named arguments ร่วมกับ default values ทำงานได้ดีมาก
def render_pagination(
  current_page : Int32,
  total_pages : Int32,
  show_first_last : Bool = true,
  show_prev_next : Bool = true,
  window_size : Int32 = 2,
  separator : String = "..."
) : String
  pages = [] of String
  
  pages << "« First" if show_first_last && current_page > 1
  pages << "‹ Prev" if show_prev_next && current_page > 1
  
  start_page = [1, current_page - window_size].max
  end_page = [total_pages, current_page + window_size].min
  
  pages << "1..." if start_page > 1
  
  (start_page..end_page).each do |p|
    pages << (p == current_page ? "[#{p}]" : p.to_s)
  end
  
  pages << "...#{total_pages}" if end_page < total_pages
  
  pages << "Next ›" if show_prev_next && current_page < total_pages
  pages << "Last »" if show_first_last && current_page < total_pages
  
  pages.join(" ")
end

puts render_pagination(5, 20)
puts render_pagination(5, 20, show_first_last: false)
puts render_pagination(5, 20, window_size: 3)
puts render_pagination(1, 20, show_prev_next: false)
```

---

## 8. ตัวอย่าง Real-World

### Chart Configuration

```crystal
struct ChartConfig
  getter title : String
  getter x_label : String
  getter y_label : String
  getter color : String
  getter show_grid : Bool
  getter legend_position : String
  
  def initialize(
    @title : String = "Chart",
    @x_label : String = "X",
    @y_label : String = "Y",
    @color : String = "blue",
    @show_grid : Bool = true,
    @legend_position : String = "top-right"
  )
  end
end

def create_chart(data : Array(Float64), **options) : String
  config = ChartConfig.new(
    title: options[:title]?.to_s.presence || "Chart",
    color: options[:color]?.to_s.presence || "blue",
    show_grid: options[:show_grid]? != false
  )
  
  "Chart: #{config.title} | #{data.size} points | color: #{config.color}"
end

puts create_chart([1.0, 2.0, 3.0], title: "Sales", color: "green")
puts create_chart([5.0, 3.0, 7.0])
```

### Request Builder

```crystal
class Request
  getter method : String
  getter path : String
  getter headers : Hash(String, String)
  getter body : String?
  getter params : Hash(String, String)
  
  def initialize(
    @method : String,
    @path : String,
    headers : Hash(String, String) = {} of String => String,
    @body : String? = nil,
    params : Hash(String, String) = {} of String => String
  )
    @headers = headers
    @params = params
  end
  
  def to_s(io : IO) : Nil
    io << "#{method} #{path}"
    unless params.empty?
      io << "?" << params.map { |k, v| "#{k}=#{v}" }.join("&")
    end
    unless headers.empty?
      io << "\nHeaders: " << headers.map { |k, v| "#{k}: #{v}" }.join(", ")
    end
    io << "\nBody: #{body}" if body
  end
end

def get(path : String, **kwargs) : Request
  headers = {} of String => String
  headers["Authorization"] = kwargs[:auth].to_s if kwargs[:auth]?
  headers["Accept"] = kwargs[:accept]?.to_s || "application/json"
  
  params = {} of String => String
  params["page"] = kwargs[:page].to_s if kwargs[:page]?
  params["limit"] = kwargs[:limit].to_s if kwargs[:limit]?
  
  Request.new("GET", path, headers: headers, params: params)
end

puts get("/api/users")
puts "\n---"
puts get("/api/users", auth: "Bearer token123", page: 1, limit: 20)
```

### Event System

```crystal
class EventEmitter
  alias Callback = NamedTuple(event: String, data: String) -> Nil
  
  @listeners = {} of String => Array(Callback)
  
  def on(event : String, &callback : Callback) : self
    @listeners[event] ||= [] of Callback
    @listeners[event] << callback
    self
  end
  
  def emit(event : String, **data) : Nil
    return unless @listeners[event]?
    
    payload = {event: event, data: data.to_s}
    @listeners[event].each { |cb| cb.call(payload) }
  end
end

emitter = EventEmitter.new

emitter.on("user.created") do |ev|
  puts "New user event: #{ev[:data]}"
end

emitter.on("user.created") do |ev|
  puts "Send welcome email for: #{ev[:event]}"
end

emitter.emit("user.created", name: "Alice", email: "alice@example.com", role: "user")
```

---

## 9. Named Arguments กับ Proc/Lambda

```crystal
# Proc ที่รับ named arguments (ผ่าน Hash หรือ NamedTuple)
process_config = ->(config : NamedTuple(debug: Bool, level: String)) {
  "debug=#{config[:debug]}, level=#{config[:level]}"
}

result = process_config.call({debug: true, level: "INFO"})
puts result

# Array ของ NamedTuples
configs = [
  {name: "dev", debug: true, verbose: true},
  {name: "staging", debug: false, verbose: true},
  {name: "prod", debug: false, verbose: false},
]

configs.each do |config|
  puts "#{config[:name]}: debug=#{config[:debug]}, verbose=#{config[:verbose]}"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Flexible Logger

```crystal
def log(
  message : String,
  level : String = "INFO",
  **context
) : String
  # TODO: สร้าง log message ในรูปแบบ:
  # [LEVEL] message {key1: val1, key2: val2}
  # ถ้าไม่มี context ไม่ต้องแสดง {}
end

puts log("Server started")
puts log("Connection failed", level: "ERROR", host: "localhost", port: 5432)
puts log("User logged in", level: "INFO", user_id: 123, ip: "192.168.1.1")
```

### แบบฝึกหัดที่ 2: Named Tuple Builder

```crystal
# สร้าง NamedTuple จาก Hash
def hash_to_named_tuple(hash : Hash(String, String)) : NamedTuple
  # Crystal ไม่รองรับ dynamic NamedTuple โดยตรง
  # TODO: สร้าง method ที่ validate required keys
  # และส่งกลับ structured data
end

# Typed builder pattern
def build_person(
  name : String,
  age : Int32,
  **extra_info
) : NamedTuple(
  name: String,
  age: Int32,
  has_extra: Bool
)
  {
    name: name,
    age: age,
    has_extra: !extra_info.empty?,
  }
end

person = build_person("Alice", 30, city: "Bangkok", job: "Engineer")
puts person[:name]
puts person[:has_extra]
```

### แบบฝึกหัดที่ 3: Query DSL

```crystal
class QueryDSL
  def select(**fields) : self
    # TODO: store fields to select
    self
  end
  
  def where(**conditions) : self
    # TODO: store where conditions
    self
  end
  
  def order(**sort_fields) : self
    # TODO: store sort fields
    self
  end
  
  def to_sql : String
    # TODO: generate SQL
  end
end

query = QueryDSL.new
  .select(name: "users.name", email: "users.email", role: "roles.name")
  .where(active: true, age_gt: 18)
  .order(name: :asc, created_at: :desc)

puts query.to_sql
# Expected: SELECT users.name, users.email, roles.name
#           WHERE active = true AND age > 18
#           ORDER BY name ASC, created_at DESC
```

### แบบฝึกหัดที่ 4: Component System

```crystal
# สร้าง component rendering system
abstract class Component
  abstract def render(**props) : String
end

class Button < Component
  def render(
    label : String = "Button",
    color : String = "blue",
    size : String = "medium",
    disabled : Bool = false,
    **extra_attrs
  ) : String
    # TODO: สร้าง HTML button element
    # <button class="btn btn-{color} btn-{size}" {disabled?}>label</button>
    # extra_attrs ถูกเพิ่มเป็น data attributes
  end
end

class Input < Component
  def render(
    type : String = "text",
    name : String = "",
    placeholder : String = "",
    required : Bool = false,
    **extra_attrs
  ) : String
    # TODO: สร้าง HTML input element
  end
end

btn = Button.new
puts btn.render(label: "Submit", color: "green", size: "large")
puts btn.render(label: "Delete", color: "red", disabled: true, data_confirm: "Are you sure?")

input = Input.new
puts input.render(type: "email", name: "email", placeholder: "Enter email", required: true)
```

---

## สรุป

| Syntax | ตัวอย่าง | ใช้เมื่อ |
|--------|----------|---------|
| Named call | `f(name: "Alice")` | มี parameter มาก/ชัดเจน |
| Mixed call | `f("Alice", age: 30)` | positional + named |
| Any order | `f(b: 2, a: 1)` | ยืดหยุ่น |
| `**kwargs` | `def f(**opts)` | รับ named args ไม่จำกัด |
| Forward | `f(**opts)` | ส่งต่อ kwargs |
| NamedTuple | `{key: val}` | Structured data |

### เมื่อไหร่ควรใช้ Named Arguments

1. **Method มี parameter มากกว่า 3 ตัว** - ใช้ named ทั้งหมด
2. **Parameter เป็น boolean** - `send_email(..., urgent: true)` ชัดกว่า `send_email(..., true)`
3. **Parameter ที่ type เหมือนกัน** - `create_rect(width: 100, height: 50)` ชัดกว่า `create_rect(100, 50)`
4. **Optional parameters** - ใช้ named เพื่อ skip บาง defaults
5. **Configuration objects** - `configure(debug: true, level: "WARN")`

### Best Practices

1. **ใช้ named args** สำหรับ method ที่มี parameter มาก
2. **NamedTuple** สำหรับ return multiple named values
3. **`**kwargs`** สำหรับ pass-through patterns (wrapper, decorator)
4. **ระวัง**: named args ใน Crystal ไม่ enforce keyword-only (ยังใช้ positional ได้)
5. **Document intent** ด้วยชื่อ parameter ที่สื่อความหมาย
