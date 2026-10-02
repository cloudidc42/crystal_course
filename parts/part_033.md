# Part 33: Class Variables และ Class Methods

## บทนำ

ในขณะที่ Instance Variables (`@var`) เก็บ state ของแต่ละ object แยกกัน Class Variables (`@@var`) เก็บ state ร่วมกันสำหรับทุก instance ของ class นั้น Class Methods (`def self.method`) เป็นเมธอดที่เรียกผ่าน class โดยตรง ไม่ใช่ผ่าน instance

---

## 1. Class Variables (@@)

### 1.1 พื้นฐาน Class Variables

```crystal
class Counter
  @@count = 0  # class variable - ใช้ร่วมกันทุก instance

  def initialize
    @@count += 1
    puts "สร้าง Counter ลำดับที่ #{@@count}"
  end

  def self.count : Int32  # class method
    @@count
  end

  def self.reset
    @@count = 0
  end
end

puts Counter.count  # => 0
c1 = Counter.new    # => สร้าง Counter ลำดับที่ 1
c2 = Counter.new    # => สร้าง Counter ลำดับที่ 2
c3 = Counter.new    # => สร้าง Counter ลำดับที่ 3
puts Counter.count  # => 3

Counter.reset
puts Counter.count  # => 0
```

### 1.2 Class Variables กับ State ร่วม

```crystal
class DatabaseConnection
  @@connections = 0
  @@max_connections = 10

  def initialize
    if @@connections >= @@max_connections
      raise "เกิน connection limit (#{@@max_connections})"
    end
    @@connections += 1
    puts "เปิด connection ##{@@connections}"
  end

  def close
    @@connections -= 1
    puts "ปิด connection (เหลือ #{@@connections})"
  end

  def self.connection_count : Int32
    @@connections
  end

  def self.max_connections : Int32
    @@max_connections
  end

  def self.max_connections=(value : Int32)
    @@max_connections = value
  end
end

DatabaseConnection.max_connections = 3
conn1 = DatabaseConnection.new
conn2 = DatabaseConnection.new
conn3 = DatabaseConnection.new
# DatabaseConnection.new  # => Error: เกิน connection limit

puts DatabaseConnection.connection_count  # => 3
conn1.close
puts DatabaseConnection.connection_count  # => 2
```

### 1.3 Registry Pattern

```crystal
class Plugin
  @@registry = {} of String => Plugin.class

  def self.register(name : String, plugin_class : Plugin.class)
    @@registry[name] = plugin_class
    puts "ลงทะเบียน plugin: #{name}"
  end

  def self.find(name : String) : Plugin.class?
    @@registry[name]?
  end

  def self.all_plugins : Array(String)
    @@registry.keys
  end

  def run
    puts "#{self.class} กำลังทำงาน..."
  end
end

class LogPlugin < Plugin
  Plugin.register("logger", self)

  def run
    puts "Logger Plugin: บันทึก log..."
  end
end

class AuthPlugin < Plugin
  Plugin.register("auth", self)

  def run
    puts "Auth Plugin: ตรวจสอบสิทธิ์..."
  end
end

puts Plugin.all_plugins.inspect  # => ["logger", "auth"]

if plugin_class = Plugin.find("logger")
  plugin = plugin_class.new
  plugin.run
end
```

---

## 2. Class Methods (def self.method)

### 2.1 Class Methods พื้นฐาน

```crystal
class MathUtils
  # Class methods - เรียกผ่าน class
  def self.square(n : Int32) : Int32
    n * n
  end

  def self.cube(n : Int32) : Int32
    n ** 3
  end

  def self.factorial(n : Int32) : Int32
    return 1 if n <= 1
    n * factorial(n - 1)
  end

  def self.fibonacci(n : Int32) : Int32
    return n if n <= 1
    fibonacci(n - 1) + fibonacci(n - 2)
  end
end

puts MathUtils.square(5)       # => 25
puts MathUtils.cube(3)         # => 27
puts MathUtils.factorial(5)    # => 120
puts MathUtils.fibonacci(10)   # => 55
```

### 2.2 Class Methods ใช้แทน Module Functions

```crystal
class StringUtils
  def self.capitalize_words(text : String) : String
    text.split.map { |word|
      word[0..0].upcase + word[1..]
    }.join(" ")
  end

  def self.truncate(text : String, max_length : Int32, suffix : String = "...") : String
    if text.size <= max_length
      text
    else
      text[0, max_length - suffix.size] + suffix
    end
  end

  def self.slugify(text : String) : String
    text.downcase.gsub(/[^a-z0-9\s]/, "").gsub(/\s+/, "-").strip
  end

  def self.count_words(text : String) : Int32
    text.split.size
  end
end

puts StringUtils.capitalize_words("hello world from crystal")
# => Hello World From Crystal

puts StringUtils.truncate("นี่คือข้อความที่ยาวมากๆ", 10)
# => นี่คือข้อความที...

puts StringUtils.slugify("Hello, World! This is Crystal.")
# => hello-world-this-is-crystal

puts StringUtils.count_words("สวัสดี โลก คริสตัล")  # => 3
```

### 2.3 Class Body Syntax

```crystal
# อีกวิธีในการนิยาม class methods
class Configuration
  class << self  # เปิด class body ของ self
    def default
      new(host: "localhost", port: 3000)
    end

    def from_env
      new(
        host: ENV["HOST"]? || "localhost",
        port: (ENV["PORT"]? || "3000").to_i
      )
    end
  end

  getter host : String
  getter port : Int32

  def initialize(host : String = "localhost", port : Int32 = 3000)
    @host = host
    @port = port
  end

  def to_s : String
    "#{@host}:#{@port}"
  end
end

dev = Configuration.default
puts dev  # => localhost:3000
```

---

## 3. Class-Level State

### 3.1 Singleton Pattern

```crystal
class AppSettings
  @@instance : AppSettings? = nil

  def self.instance : AppSettings
    @@instance ||= new
  end

  # ป้องกันการสร้าง instance โดยตรง
  private def initialize
    @settings = {} of String => String
    load_defaults
  end

  def get(key : String) : String?
    @settings[key]?
  end

  def set(key : String, value : String)
    @settings[key] = value
  end

  def all : Hash(String, String)
    @settings.dup
  end

  private def load_defaults
    @settings["app_name"] = "My Crystal App"
    @settings["version"] = "1.0.0"
    @settings["debug"] = "false"
  end
end

# ทุกครั้งที่เรียก AppSettings.instance จะได้ object เดิม
settings = AppSettings.instance
settings.set("theme", "dark")
settings.set("language", "th")

puts AppSettings.instance.get("app_name")  # => My Crystal App
puts AppSettings.instance.get("theme")     # => dark

# ยืนยันว่าเป็น object เดิม
puts AppSettings.instance.equal?(settings)  # => true
```

### 3.2 Class-Level Cache

```crystal
class ExchangeRate
  @@rates = {} of String => Float64
  @@last_updated : Time? = nil

  def self.set_rate(currency : String, rate : Float64)
    @@rates[currency] = rate
    @@last_updated = Time.local
  end

  def self.get_rate(currency : String) : Float64?
    @@rates[currency]?
  end

  def self.convert(amount : Float64, from : String, to : String) : Float64?
    from_rate = @@rates[from]?
    to_rate = @@rates[to]?

    return nil unless from_rate && to_rate

    # แปลงเป็น USD ก่อน แล้วแปลงเป็นสกุลเป้าหมาย
    usd_amount = amount / from_rate
    usd_amount * to_rate
  end

  def self.last_updated : Time?
    @@last_updated
  end
end

ExchangeRate.set_rate("USD", 1.0)
ExchangeRate.set_rate("THB", 35.5)
ExchangeRate.set_rate("JPY", 150.0)
ExchangeRate.set_rate("EUR", 0.92)

result = ExchangeRate.convert(1000.0, "THB", "JPY")
puts "1000 บาท = #{result.try(&.round(2))} เยน"

result2 = ExchangeRate.convert(100.0, "USD", "THB")
puts "100 USD = #{result2.try(&.round(2))} บาท"
```

---

## 4. Class Constants

### 4.1 Constants ใน Class

```crystal
class Physics
  # ค่าคงที่ทางฟิสิกส์
  SPEED_OF_LIGHT = 299_792_458.0  # m/s
  GRAVITY = 9.81                   # m/s²
  PLANCK_CONSTANT = 6.626e-34     # J·s
  AVOGADRO = 6.022e23              # mol⁻¹

  def self.kinetic_energy(mass : Float64, velocity : Float64) : Float64
    0.5 * mass * velocity ** 2
  end

  def self.gravitational_potential(mass : Float64, height : Float64) : Float64
    mass * GRAVITY * height
  end

  def self.free_fall_time(height : Float64) : Float64
    Math.sqrt(2 * height / GRAVITY)
  end
end

puts Physics::SPEED_OF_LIGHT      # => 299792458.0
puts Physics.kinetic_energy(1.0, 10.0)  # => 50.0
puts Physics.free_fall_time(45.0).round(2)  # => 3.03
```

### 4.2 Enum เป็น Class Constant

```crystal
class Order
  enum Status
    Pending
    Processing
    Shipped
    Delivered
    Cancelled
  end

  STATUS_COLORS = {
    Status::Pending => "เหลือง",
    Status::Processing => "น้ำเงิน",
    Status::Shipped => "ส้ม",
    Status::Delivered => "เขียว",
    Status::Cancelled => "แดง",
  }

  getter id : String
  getter status : Status

  def initialize(@id : String)
    @status = Status::Pending
  end

  def advance
    @status = case @status
    when Status::Pending then Status::Processing
    when Status::Processing then Status::Shipped
    when Status::Shipped then Status::Delivered
    else @status
    end
  end

  def cancel
    @status = Status::Cancelled unless @status == Status::Delivered
  end

  def status_color : String
    STATUS_COLORS[@status]? || "เทา"
  end

  def to_s : String
    "Order##{@id} [#{@status}] (#{status_color})"
  end
end

order = Order.new("ORD001")
puts order

order.advance
puts order

order.advance
puts order

order.cancel
puts order
```

---

## 5. Factory Methods

### 5.1 Alternative Constructors

```crystal
class Color
  getter r : UInt8
  getter g : UInt8
  getter b : UInt8

  def initialize(@r : UInt8, @g : UInt8, @b : UInt8)
  end

  # Factory methods - สร้าง Color ด้วยวิธีต่างๆ
  def self.from_hex(hex : String) : Color
    hex = hex.lstrip('#')
    r = hex[0..1].to_u8(16)
    g = hex[2..3].to_u8(16)
    b = hex[4..5].to_u8(16)
    new(r, g, b)
  end

  def self.from_hsl(h : Float64, s : Float64, l : Float64) : Color
    # แปลง HSL เป็น RGB (simplified)
    c = (1 - (2*l - 1).abs) * s
    x = c * (1 - ((h / 60) % 2 - 1).abs)
    m = l - c / 2

    r, g, b = case h.to_i / 60
    when 0 then {c, x, 0.0}
    when 1 then {x, c, 0.0}
    when 2 then {0.0, c, x}
    when 3 then {0.0, x, c}
    when 4 then {x, 0.0, c}
    else {c, 0.0, x}
    end

    new(
      ((r + m) * 255).round.to_u8,
      ((g + m) * 255).round.to_u8,
      ((b + m) * 255).round.to_u8
    )
  end

  # Named colors
  def self.red; new(255_u8, 0_u8, 0_u8); end
  def self.green; new(0_u8, 128_u8, 0_u8); end
  def self.blue; new(0_u8, 0_u8, 255_u8); end
  def self.white; new(255_u8, 255_u8, 255_u8); end
  def self.black; new(0_u8, 0_u8, 0_u8); end

  def to_hex : String
    "#%02X%02X%02X" % [@r, @g, @b]
  end

  def to_s : String
    "rgb(#{@r}, #{@g}, #{@b})"
  end
end

red = Color.red
puts red            # => rgb(255, 0, 0)
puts red.to_hex     # => #FF0000

coral = Color.from_hex("#FF7F50")
puts coral          # => rgb(255, 127, 80)

custom = Color.new(100_u8, 150_u8, 200_u8)
puts custom         # => rgb(100, 150, 200)
```

### 5.2 Builder ด้วย Class Methods

```crystal
class QueryBuilder
  @@default_limit = 100

  def self.default_limit : Int32
    @@default_limit
  end

  def self.default_limit=(value : Int32)
    @@default_limit = value
  end

  def self.select_all(table : String) : QueryBuilder
    new(table)
  end

  def initialize(@table : String)
    @conditions = [] of String
    @limit = @@default_limit
    @order = nil.as(String?)
  end

  def where(condition : String) : self
    @conditions << condition
    self
  end

  def order_by(column : String) : self
    @order = column
    self
  end

  def limit(n : Int32) : self
    @limit = n
    self
  end

  def build : String
    sql = "SELECT * FROM #{@table}"
    sql += " WHERE #{@conditions.join(" AND ")}" unless @conditions.empty?
    sql += " ORDER BY #{@order}" if @order
    sql += " LIMIT #{@limit}"
    sql
  end
end

QueryBuilder.default_limit = 50

query = QueryBuilder.select_all("users")
  .where("age >= 18")
  .where("active = true")
  .order_by("name ASC")
  .limit(10)
  .build

puts query
# => SELECT * FROM users WHERE age >= 18 AND active = true ORDER BY name ASC LIMIT 10
```

---

## 6. Class Methods กับ Inheritance

### 6.1 Inheriting Class Methods

```crystal
class Animal
  @@count = 0

  def self.count : Int32
    @@count
  end

  def initialize
    @@count += 1
  end
end

class Dog < Animal
  @@dog_count = 0

  def self.dog_count : Int32
    @@dog_count
  end

  def initialize
    super
    @@dog_count += 1
  end
end

class Cat < Animal
  @@cat_count = 0

  def self.cat_count : Int32
    @@cat_count
  end

  def initialize
    super
    @@cat_count += 1
  end
end

Dog.new; Dog.new; Dog.new
Cat.new; Cat.new

puts "สัตว์ทั้งหมด: #{Animal.count}"  # => 5
puts "สุนัข: #{Dog.dog_count}"          # => 3
puts "แมว: #{Cat.cat_count}"            # => 2
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: User Registry

```crystal
class User
  @@users = {} of String => User
  @@next_id = 1

  getter id : Int32
  getter username : String
  getter email : String

  private def initialize(@id : Int32, @username : String, @email : String)
  end

  def self.create(username : String, email : String) : User?
    # ตรวจสอบ duplicate
    return nil if @@users.values.any? { |u| u.email == email }

    user = new(@@next_id, username, email)
    @@users[username] = user
    @@next_id += 1
    puts "สร้างผู้ใช้: #{username} (ID: #{user.id})"
    user
  end

  def self.find(username : String) : User?
    @@users[username]?
  end

  def self.all : Array(User)
    @@users.values
  end

  def self.count : Int32
    @@users.size
  end

  def to_s : String
    "User##{@id}: #{@username} (#{@email})"
  end
end

User.create("somchai", "somchai@example.com")
User.create("somsri", "somsri@example.com")
User.create("somsak", "somsak@example.com")
User.create("duplicate", "somchai@example.com")  # จะไม่สร้าง

puts "\nผู้ใช้ทั้งหมด (#{User.count} คน):"
User.all.each { |u| puts "  #{u}" }

if user = User.find("somchai")
  puts "\nพบผู้ใช้: #{user}"
end
```

### แบบฝึกหัดที่ 2: Statistics Tracker

```crystal
class Statistics
  @@data = [] of Float64
  @@labels = {} of String => Float64

  def self.record(value : Float64)
    @@data << value
  end

  def self.record(label : String, value : Float64)
    @@data << value
    @@labels[label] = value
  end

  def self.count : Int32
    @@data.size
  end

  def self.sum : Float64
    @@data.sum
  end

  def self.mean : Float64
    return 0.0 if @@data.empty?
    sum / count
  end

  def self.min : Float64?
    @@data.min?
  end

  def self.max : Float64?
    @@data.max?
  end

  def self.report
    puts "=== Statistics Report ==="
    puts "จำนวนข้อมูล: #{count}"
    puts "ผลรวม: #{sum.round(2)}"
    puts "เฉลี่ย: #{mean.round(2)}"
    puts "น้อยสุด: #{min}"
    puts "มากสุด: #{max}"
    unless @@labels.empty?
      puts "\nข้อมูลที่มีชื่อ:"
      @@labels.each { |label, val| puts "  #{label}: #{val}" }
    end
  end

  def self.reset
    @@data.clear
    @@labels.clear
  end
end

Statistics.record("เดือน ม.ค.", 15000.0)
Statistics.record("เดือน ก.พ.", 18000.0)
Statistics.record("เดือน มี.ค.", 12000.0)
Statistics.record(20000.0)
Statistics.record(16000.0)

Statistics.report
```

---

## สรุป

Class Variables และ Class Methods ใน Crystal:

| Feature | Syntax | ใช้เมื่อ |
|---------|--------|---------|
| Class variable | `@@var` | State ที่ใช้ร่วมกันทุก instance |
| Class method | `def self.method` | ทำงานกับ class ไม่ใช่ instance |
| Class constant | `NAME = value` | ค่าคงที่ระดับ class |
| Factory method | `def self.create(...)` | Alternative constructors |

**ข้อควรระวัง:**
- `@@var` ใช้ร่วมกันทุก instance รวมถึง subclasses ด้วย
- Class variables อาจทำให้ยาก debug ในโปรแกรมใหญ่
- Singleton pattern ควรใช้ด้วยความระมัดระวัง (testing ยาก)
- ใช้ Factory methods เพื่อสร้าง instances ด้วยวิธีต่างๆ อย่างชัดเจน
