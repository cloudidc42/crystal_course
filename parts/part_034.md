# Part 34: Constructors และ initialize

## บทนำ

Constructor เป็นเมธอดพิเศษที่ถูกเรียกโดยอัตโนมัติเมื่อสร้าง object ใหม่ ใน Crystal constructor คือเมธอด `initialize` Crystal มีวิธีสร้าง Constructor ที่ยืดหยุ่นหลายแบบ ทั้งการ overload, named parameters, validation และ factory methods

---

## 1. def initialize พื้นฐาน

### 1.1 Constructor ง่ายที่สุด

```crystal
class Person
  def initialize
    puts "สร้าง Person ใหม่!"
    @name = "ไม่ระบุ"
    @age = 0
  end

  def info
    "#{@name}, #{@age} ปี"
  end
end

p = Person.new
puts p.info  # => ไม่ระบุ, 0 ปี
```

### 1.2 Constructor พร้อม Parameters

```crystal
class Rectangle
  def initialize(width : Float64, height : Float64)
    @width = width
    @height = height
  end

  # Shorthand: ใช้ @ ใน parameters โดยตรง
  # def initialize(@width : Float64, @height : Float64)

  def area : Float64
    @width * @height
  end
end

rect = Rectangle.new(5.0, 3.0)
puts rect.area  # => 15.0
```

### 1.3 @shorthand ใน Parameters

```crystal
# วิธีที่กระชับที่สุด - ใช้ @ นำหน้า parameter name
class Car
  getter brand : String
  getter model : String
  getter year : Int32

  def initialize(@brand : String, @model : String, @year : Int32)
    # Crystal จะ assign ค่าให้ @brand, @model, @year อัตโนมัติ
  end

  def to_s : String
    "#{@year} #{@brand} #{@model}"
  end
end

car = Car.new("Toyota", "Camry", 2023)
puts car  # => 2023 Toyota Camry
```

---

## 2. Multiple Initializers ด้วย Overloading

### 2.1 Constructors หลายตัว

```crystal
class Point
  getter x : Float64
  getter y : Float64
  getter z : Float64

  # 2D point
  def initialize(@x : Float64, @y : Float64)
    @z = 0.0
  end

  # 3D point
  def initialize(@x : Float64, @y : Float64, @z : Float64)
  end

  # สร้างจาก string "x,y" หรือ "x,y,z"
  def initialize(coords : String)
    parts = coords.split(",").map(&.to_f)
    @x = parts[0]? || 0.0
    @y = parts[1]? || 0.0
    @z = parts[2]? || 0.0
  end

  def distance_to(other : Point) : Float64
    Math.sqrt(
      (@x - other.x) ** 2 +
      (@y - other.y) ** 2 +
      (@z - other.z) ** 2
    )
  end

  def to_s : String
    @z == 0.0 ? "(#{@x}, #{@y})" : "(#{@x}, #{@y}, #{@z})"
  end
end

p1 = Point.new(1.0, 2.0)          # 2D
p2 = Point.new(1.0, 2.0, 3.0)    # 3D
p3 = Point.new("4.0,5.0")         # from string
p4 = Point.new("1.0,2.0,3.0")    # 3D from string

puts p1  # => (1.0, 2.0)
puts p2  # => (1.0, 2.0, 3.0)
puts p3  # => (4.0, 5.0)
puts p4  # => (1.0, 2.0, 3.0)

puts p1.distance_to(p2).round(2)  # => 3.0
```

### 2.2 Overloading ด้วย Type ต่างกัน

```crystal
class Duration
  getter total_seconds : Int64

  # จากวินาที
  def initialize(seconds : Int64)
    @total_seconds = seconds
  end

  # จากนาที
  def initialize(minutes : Int32)
    @total_seconds = minutes.to_i64 * 60
  end

  # จากชั่วโมงและนาที
  def initialize(hours : Int32, minutes : Int32)
    @total_seconds = (hours * 3600 + minutes * 60).to_i64
  end

  # จากชั่วโมง, นาที, วินาที
  def initialize(hours : Int32, minutes : Int32, seconds : Int32)
    @total_seconds = (hours * 3600 + minutes * 60 + seconds).to_i64
  end

  def hours : Int64
    @total_seconds / 3600
  end

  def minutes : Int64
    (@total_seconds % 3600) / 60
  end

  def seconds : Int64
    @total_seconds % 60
  end

  def to_s : String
    "%02d:%02d:%02d" % [hours, minutes, seconds]
  end
end

d1 = Duration.new(3661_i64)     # 1 ชั่วโมง 1 นาที 1 วินาที
d2 = Duration.new(90)           # 90 นาที
d3 = Duration.new(2, 30)        # 2 ชั่วโมง 30 นาที
d4 = Duration.new(1, 30, 45)    # 1:30:45

puts d1  # => 01:01:01
puts d2  # => 01:30:00
puts d3  # => 02:30:00
puts d4  # => 01:30:45
```

---

## 3. Named Parameters ใน initialize

### 3.1 Named Parameters พื้นฐาน

```crystal
class Config
  getter host : String
  getter port : Int32
  getter timeout : Int32
  getter debug : Bool

  def initialize(
    host : String = "localhost",
    port : Int32 = 8080,
    timeout : Int32 = 30,
    debug : Bool = false
  )
    @host = host
    @port = port
    @timeout = timeout
    @debug = debug
  end

  def to_s : String
    "#{@host}:#{@port} (timeout: #{@timeout}s, debug: #{@debug})"
  end
end

# ใช้ default ทั้งหมด
c1 = Config.new
puts c1  # => localhost:8080 (timeout: 30s, debug: false)

# ระบุเฉพาะบางตัว
c2 = Config.new(host: "api.example.com", port: 443, debug: true)
puts c2  # => api.example.com:443 (timeout: 30s, debug: true)

# ระบุบางตัวในลำดับใดก็ได้
c3 = Config.new(debug: true, host: "staging.example.com")
puts c3  # => staging.example.com:8080 (timeout: 30s, debug: true)
```

### 3.2 Named Parameters Required และ Optional ผสมกัน

```crystal
class Employee
  getter name : String
  getter department : String
  getter salary : Float64
  getter? manager : Bool

  def initialize(
    @name : String,           # required (positional)
    @department : String,     # required (positional)
    salary : Float64 = 30000.0,  # optional keyword
    manager : Bool = false       # optional keyword
  )
    @salary = salary
    @manager = manager
  end

  def to_s : String
    role = @manager ? "ผู้จัดการ" : "พนักงาน"
    "#{@name} (#{@department}) - #{role} - ฿#{@salary}"
  end
end

e1 = Employee.new("สมชาย", "IT")
puts e1  # => สมชาย (IT) - พนักงาน - ฿30000.0

e2 = Employee.new("สมหญิง", "HR", salary: 50000.0, manager: true)
puts e2  # => สมหญิง (HR) - ผู้จัดการ - ฿50000.0

e3 = Employee.new("สมศักดิ์", "Finance", salary: 45000.0)
puts e3  # => สมศักดิ์ (Finance) - พนักงาน - ฿45000.0
```

---

## 4. Validation ใน initialize

### 4.1 Basic Validation

```crystal
class Age
  getter value : Int32

  def initialize(value : Int32)
    unless value >= 0 && value <= 150
      raise ArgumentError.new("อายุต้องอยู่ระหว่าง 0-150 ปี (ได้รับ: #{value})")
    end
    @value = value
  end

  def adult? : Bool
    @value >= 18
  end

  def to_s : String
    "#{@value} ปี"
  end
end

valid_age = Age.new(25)
puts valid_age        # => 25 ปี
puts valid_age.adult? # => true

begin
  invalid_age = Age.new(-5)
rescue ex : ArgumentError
  puts "Error: #{ex.message}"
  # => Error: อายุต้องอยู่ระหว่าง 0-150 ปี (ได้รับ: -5)
end
```

### 4.2 Complex Validation

```crystal
class Email
  getter address : String

  def initialize(address : String)
    address = address.strip.downcase

    if address.empty?
      raise ArgumentError.new("อีเมลต้องไม่ว่างเปล่า")
    end

    unless address.matches?(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
      raise ArgumentError.new("รูปแบบอีเมลไม่ถูกต้อง: #{address}")
    end

    @address = address
  end

  def domain : String
    @address.split("@")[1]
  end

  def local_part : String
    @address.split("@")[0]
  end

  def to_s : String
    @address
  end
end

begin
  e1 = Email.new("user@example.com")
  puts e1          # => user@example.com
  puts e1.domain   # => example.com

  e2 = Email.new("invalid-email")  # จะ raise error
rescue ex : ArgumentError
  puts "Error: #{ex.message}"
  # => Error: รูปแบบอีเมลไม่ถูกต้อง: invalid-email
end
```

### 4.3 Multi-field Validation

```crystal
class BankTransfer
  getter from_account : String
  getter to_account : String
  getter amount : Float64
  getter description : String

  def initialize(
    from_account : String,
    to_account : String,
    amount : Float64,
    description : String = ""
  )
    errors = [] of String

    errors << "บัญชีต้นทางต้องไม่ว่าง" if from_account.empty?
    errors << "บัญชีปลายทางต้องไม่ว่าง" if to_account.empty?
    errors << "จำนวนเงินต้องมากกว่า 0" unless amount > 0
    errors << "ต้นทางและปลายทางต้องไม่เหมือนกัน" if from_account == to_account
    errors << "จำนวนเงินต้องไม่เกิน 1,000,000 บาท" if amount > 1_000_000

    unless errors.empty?
      raise ArgumentError.new("ข้อมูลไม่ถูกต้อง:\n" + errors.map { |e| "  - #{e}" }.join("\n"))
    end

    @from_account = from_account
    @to_account = to_account
    @amount = amount
    @description = description
  end

  def to_s : String
    "โอน ฿#{@amount} จาก #{@from_account} ไป #{@to_account}"
  end
end

transfer = BankTransfer.new("ACC001", "ACC002", 5000.0, "ค่าเช่า")
puts transfer  # => โอน ฿5000.0 จาก ACC001 ไป ACC002

begin
  bad_transfer = BankTransfer.new("ACC001", "ACC001", -100.0)
rescue ex : ArgumentError
  puts ex.message
end
```

---

## 5. Factory Class Methods เป็น Constructors

### 5.1 Named Factory Methods

```crystal
class Color
  getter r : UInt8
  getter g : UInt8
  getter b : UInt8
  getter a : UInt8  # alpha

  private def initialize(@r : UInt8, @g : UInt8, @b : UInt8, @a : UInt8 = 255_u8)
  end

  # Factory methods
  def self.rgb(r : UInt8, g : UInt8, b : UInt8) : Color
    new(r, g, b)
  end

  def self.rgba(r : UInt8, g : UInt8, b : UInt8, a : UInt8) : Color
    new(r, g, b, a)
  end

  def self.from_hex(hex : String) : Color
    hex = hex.lstrip('#')
    case hex.size
    when 3
      r = (hex[0..0] * 2).to_u8(16)
      g = (hex[1..1] * 2).to_u8(16)
      b = (hex[2..2] * 2).to_u8(16)
      new(r, g, b)
    when 6
      new(hex[0..1].to_u8(16), hex[2..3].to_u8(16), hex[4..5].to_u8(16))
    when 8
      new(hex[0..1].to_u8(16), hex[2..3].to_u8(16),
          hex[4..5].to_u8(16), hex[6..7].to_u8(16))
    else
      raise ArgumentError.new("รูปแบบ hex ไม่ถูกต้อง: ##{hex}")
    end
  end

  def self.random : Color
    new(rand(256).to_u8, rand(256).to_u8, rand(256).to_u8)
  end

  def transparent : Color
    Color.rgba(@r, @g, @b, 0_u8)
  end

  def to_hex : String
    "#%02X%02X%02X" % [@r, @g, @b]
  end

  def to_s : String
    @a == 255 ? "rgb(#{@r},#{@g},#{@b})" : "rgba(#{@r},#{@g},#{@b},#{@a})"
  end
end

c1 = Color.rgb(255_u8, 0_u8, 0_u8)
c2 = Color.from_hex("#00FF00")
c3 = Color.from_hex("FFF")  # shorthand
c4 = Color.random

puts c1  # => rgb(255,0,0)
puts c2  # => rgb(0,255,0)
puts c3  # => rgb(255,255,255)
```

### 5.2 Parse Factory Method

```crystal
class IpAddress
  getter octets : Array(Int32)

  private def initialize(@octets : Array(Int32))
  end

  def self.parse(ip_string : String) : IpAddress?
    parts = ip_string.split(".")
    return nil unless parts.size == 4

    octets = parts.map { |p|
      n = p.to_i?
      return nil if n.nil? || n < 0 || n > 255
      n
    }

    new(octets)
  end

  def self.parse!(ip_string : String) : IpAddress
    parse(ip_string) || raise ArgumentError.new("IP ไม่ถูกต้อง: #{ip_string}")
  end

  def self.localhost : IpAddress
    new([127, 0, 0, 1])
  end

  def private? : Bool
    case @octets[0]
    when 10 then true
    when 172 then @octets[1].in?(16..31)
    when 192 then @octets[1] == 168
    else false
    end
  end

  def to_s : String
    @octets.join(".")
  end
end

ip1 = IpAddress.parse("192.168.1.1")
puts ip1.try(&.to_s)      # => 192.168.1.1
puts ip1.try(&.private?)  # => true

ip2 = IpAddress.parse("invalid")
puts ip2.inspect  # => nil

ip3 = IpAddress.localhost
puts ip3  # => 127.0.0.1

begin
  IpAddress.parse!("999.0.0.1")
rescue ex : ArgumentError
  puts ex.message  # => IP ไม่ถูกต้อง: 999.0.0.1
end
```

---

## 6. initialize กับ Inheritance

### 6.1 super ใน initialize

```crystal
class Vehicle
  getter make : String
  getter model : String
  getter year : Int32

  def initialize(@make : String, @model : String, @year : Int32)
    puts "สร้าง Vehicle: #{@year} #{@make} #{@model}"
  end
end

class ElectricVehicle < Vehicle
  getter battery_kwh : Float64
  getter range_km : Int32

  def initialize(make : String, model : String, year : Int32,
                 battery_kwh : Float64, range_km : Int32)
    super(make, model, year)  # เรียก parent constructor
    @battery_kwh = battery_kwh
    @range_km = range_km
    puts "รถไฟฟ้า: แบต #{@battery_kwh}kWh, วิ่งได้ #{@range_km}km"
  end

  def to_s : String
    "#{@year} #{@make} #{@model} (EV: #{@battery_kwh}kWh)"
  end
end

tesla = ElectricVehicle.new("Tesla", "Model 3", 2023, 75.0, 350)
puts tesla
```

---

## 7. ตัวอย่างครบวงจร

### 7.1 HTTP Request

```crystal
class HttpRequest
  enum Method
    Get; Post; Put; Patch; Delete
  end

  getter method : Method
  getter url : String
  getter headers : Hash(String, String)
  getter body : String?

  private def initialize(
    @method : Method,
    @url : String,
    @headers : Hash(String, String),
    @body : String?
  )
    validate!
  end

  def self.get(url : String, headers : Hash(String, String) = {} of String => String)
    new(Method::Get, url, headers, nil)
  end

  def self.post(url : String, body : String, headers : Hash(String, String) = {} of String => String)
    new(Method::Post, url, headers, body)
  end

  def self.put(url : String, body : String, headers : Hash(String, String) = {} of String => String)
    new(Method::Put, url, headers, body)
  end

  def self.delete(url : String, headers : Hash(String, String) = {} of String => String)
    new(Method::Delete, url, headers, nil)
  end

  def with_header(key : String, value : String) : HttpRequest
    new_headers = @headers.merge({key => value})
    self.class.new(@method, @url, new_headers, @body)
  end

  def to_s : String
    "#{@method} #{@url}"
  end

  private def validate!
    raise ArgumentError.new("URL ต้องไม่ว่าง") if @url.empty?
    raise ArgumentError.new("URL ต้องเริ่มด้วย http/https") unless @url.starts_with?("http")
  end
end

req1 = HttpRequest.get("https://api.example.com/users")
req2 = HttpRequest.post("https://api.example.com/users",
  body: "{\"name\": \"สมชาย\"}",
  headers: {"Content-Type" => "application/json"}
)

puts req1  # => Get https://api.example.com/users
puts req2  # => Post https://api.example.com/users
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Money Class

```crystal
class Money
  getter amount : Float64
  getter currency : String

  VALID_CURRENCIES = ["THB", "USD", "EUR", "JPY", "GBP"]

  def initialize(amount : Float64, currency : String = "THB")
    raise ArgumentError.new("จำนวนเงินต้องไม่ติดลบ") if amount < 0
    unless VALID_CURRENCIES.includes?(currency)
      raise ArgumentError.new("สกุลเงินไม่รองรับ: #{currency}")
    end
    @amount = amount.round(2)
    @currency = currency
  end

  def self.zero(currency : String = "THB") : Money
    new(0.0, currency)
  end

  def self.from_string(str : String) : Money
    parts = str.split
    raise ArgumentError.new("รูปแบบไม่ถูกต้อง") unless parts.size == 2
    new(parts[0].to_f, parts[1])
  end

  def +(other : Money) : Money
    raise "สกุลเงินต่างกัน" unless @currency == other.currency
    Money.new(@amount + other.amount, @currency)
  end

  def to_s : String
    "#{@currency} #{@amount}"
  end
end

m1 = Money.new(100.0)
m2 = Money.new(50.0, "THB")
m3 = Money.from_string("200.0 USD")

puts m1            # => THB 100.0
puts m3            # => USD 200.0
puts (m1 + m2)     # => THB 150.0
puts Money.zero    # => THB 0.0
```

---

## สรุป

Constructor และ initialize ใน Crystal:

| Pattern | ใช้เมื่อ |
|---------|---------|
| Simple initialize | Object ง่าย ไม่มี validation |
| @shorthand params | กระชับ สำหรับ simple assignment |
| Named parameters | มี optional params หลายตัว |
| Overloaded initialize | รับ input หลายรูปแบบ |
| Validation in initialize | ตรวจสอบความถูกต้องก่อน create |
| Private initialize + factories | ควบคุมวิธีการสร้าง |
| super in initialize | ส่งต่อ initialization ให้ parent |

**Best Practices:**
- ทำให้ constructor ง่ายที่สุดเท่าที่จะทำได้
- ทำ validation ใน constructor เพื่อ fail fast
- ใช้ factory methods เมื่อ construction logic ซับซ้อน
- Named parameters ช่วยให้อ่านโค้ดเข้าใจง่ายขึ้น
