# Part 43: Property Accessors ใน Crystal

## บทนำ

Property accessors คือ methods พิเศษที่ให้เราอ่านและเขียน instance variables ได้อย่างสะดวก Crystal มี macros ในตัวที่สร้าง getters และ setters ให้อัตโนมัติ ได้แก่ `getter`, `setter`, และ `property`

---

## 43.1 ปัญหาที่ Accessor แก้ไข

```crystal
# โดยไม่มี accessor macros
class Person
  def initialize(@name : String, @age : Int32)
  end
  
  # ต้องเขียน getter เอง
  def name : String
    @name
  end
  
  # ต้องเขียน setter เอง
  def name=(value : String)
    @name = value
  end
  
  def age : Int32
    @age
  end
  
  def age=(value : Int32)
    @age = value
  end
end

# ซ้ำซ้อนมาก! Crystal มี macros ช่วย
```

---

## 43.2 getter Macro

`getter` สร้าง getter method สำหรับอ่าน instance variable

```crystal
class Car
  # getter สร้าง method ที่คืน instance variable
  getter make : String
  getter model : String
  getter year : Int32
  getter mileage : Float64
  
  def initialize(@make : String, @model : String, @year : Int32)
    @mileage = 0.0
  end
  
  def drive(distance : Float64)
    @mileage += distance
  end
end

car = Car.new("Toyota", "Camry", 2023)
puts car.make    # => "Toyota"
puts car.model   # => "Camry"
puts car.year    # => 2023
puts car.mileage # => 0.0

car.drive(100.0)
puts car.mileage # => 100.0

# car.make = "Honda"  # Error! ไม่มี setter
```

### getter กับ default value

```crystal
class Config
  getter host : String = "localhost"
  getter port : Int32 = 8080
  getter debug : Bool = false
  getter max_connections : Int32 = 100
  
  def initialize
    # instance variables มี default value แล้ว
  end
  
  # หรือ override ใน initialize
  def initialize(@host : String, @port : Int32)
    @debug = false
    @max_connections = 100
  end
end

config1 = Config.new
puts config1.host  # => "localhost"
puts config1.port  # => 8080

config2 = Config.new("example.com", 443)
puts config2.host  # => "example.com"
puts config2.port  # => 443
```

### getter หลายตัวในบรรทัดเดียว

```crystal
class Point
  getter x : Float64
  getter y : Float64
  getter z : Float64
  
  # หรือเขียนรวมกัน
  # getter x : Float64, y : Float64, z : Float64
  
  def initialize(@x : Float64, @y : Float64, @z : Float64 = 0.0)
  end
  
  def distance_to(other : Point) : Float64
    Math.sqrt((x - other.x)**2 + (y - other.y)**2 + (z - other.z)**2)
  end
  
  def to_s(io : IO) : Nil
    io << "(#{x}, #{y}, #{z})"
  end
end

p1 = Point.new(0.0, 0.0, 0.0)
p2 = Point.new(3.0, 4.0, 0.0)
puts p1.distance_to(p2)  # => 5.0
```

---

## 43.3 setter Macro

`setter` สร้าง setter method สำหรับเขียน instance variable

```crystal
class UserProfile
  getter username : String
  getter email : String
  
  # setter เท่านั้น (ไม่มี getter)
  setter password : String
  
  def initialize(@username : String, @email : String, @password : String)
  end
  
  def authenticate(password : String) : Bool
    @password == password  # ใช้ @password โดยตรงใน class
  end
end

user = UserProfile.new("alice", "alice@example.com", "secret123")
puts user.username   # => "alice"
puts user.email      # => "alice@example.com"
# puts user.password  # Error! ไม่มี getter

user.password = "newpassword"
puts user.authenticate("newpassword")  # => true
puts user.authenticate("wrongpass")    # => false
```

### setter กับ type conversion

```crystal
class Temperature
  def initialize(@celsius : Float64 = 0.0)
  end
  
  getter celsius : Float64
  
  # Custom setter ที่แปลง Fahrenheit เป็น Celsius
  def fahrenheit=(f : Float64)
    @celsius = (f - 32) * 5.0 / 9.0
  end
  
  def fahrenheit : Float64
    @celsius * 9.0 / 5.0 + 32
  end
  
  def to_s(io : IO) : Nil
    io << "#{@celsius.round(2)}°C (#{fahrenheit.round(2)}°F)"
  end
end

temp = Temperature.new(100.0)
puts temp  # => 100.0°C (212.0°F)

temp.fahrenheit = 32.0
puts temp  # => 0.0°C (32.0°F)
```

---

## 43.4 property Macro

`property` สร้างทั้ง getter และ setter พร้อมกัน

```crystal
class Article
  property title : String
  property content : String
  property published : Bool = false
  property views : Int32 = 0
  
  getter created_at : Time  # อ่านได้อย่างเดียว
  
  def initialize(@title : String, @content : String)
    @created_at = Time.local
  end
  
  def publish!
    @published = true
  end
  
  def view!
    @views += 1
  end
end

article = Article.new("Crystal Tutorial", "...")
puts article.title      # => "Crystal Tutorial"
puts article.published  # => false

article.title = "Advanced Crystal"
article.publish!
article.view!
article.view!

puts article.title    # => "Advanced Crystal"
puts article.published  # => true
puts article.views    # => 2
# article.created_at = ...  # Error! read-only
```

### property กับ complex types

```crystal
class Config
  property database_url : String = "localhost:5432"
  property pool_size : Int32 = 5
  property timeout : Float64 = 30.0
  property options : Hash(String, String) = {} of String => String
  property tags : Array(String) = [] of String
  
  def initialize
  end
  
  def add_option(key : String, value : String)
    @options[key] = value
  end
  
  def add_tag(tag : String)
    @tags << tag unless @tags.includes?(tag)
  end
end

config = Config.new
config.database_url = "postgresql://localhost/mydb"
config.pool_size = 10
config.add_option("charset", "utf8")
config.add_tag("production")
config.add_tag("v2")

puts config.database_url  # => postgresql://localhost/mydb
puts config.options        # => {"charset" => "utf8"}
puts config.tags           # => ["production", "v2"]
```

---

## 43.5 getter? สำหรับ Boolean Properties

`getter?` และ `property?` สร้าง method ที่ลงท้ายด้วย `?` สำหรับ boolean

```crystal
class User
  getter name : String
  getter email : String
  
  # Boolean properties ด้วย ?
  getter? active : Bool = true
  getter? email_verified : Bool = false
  getter? admin : Bool = false
  getter? premium : Bool = false
  
  def initialize(@name : String, @email : String)
  end
  
  def verify_email!
    @email_verified = true
  end
  
  def make_admin!
    @admin = true
  end
  
  def deactivate!
    @active = false
  end
end

user = User.new("Alice", "alice@example.com")
puts user.active?          # => true
puts user.email_verified?  # => false
puts user.admin?           # => false

user.verify_email!
user.make_admin!

puts user.email_verified?  # => true
puts user.admin?           # => true

user.deactivate!
puts user.active?          # => false
```

### property? รวม getter และ setter

```crystal
class Feature
  property? enabled : Bool = false
  property? beta : Bool = false
  
  getter name : String
  getter description : String
  
  def initialize(@name : String, @description : String)
  end
  
  def to_s(io : IO) : Nil
    status = enabled? ? "ON" : "OFF"
    beta_str = beta? ? " [BETA]" : ""
    io << "#{@name}#{beta_str}: #{status} - #{@description}"
  end
end

dark_mode = Feature.new("Dark Mode", "Switch to dark theme")
new_editor = Feature.new("New Editor", "Redesigned text editor")

dark_mode.enabled = true
new_editor.enabled = true
new_editor.beta = true

puts dark_mode   # => Dark Mode: ON - Switch to dark theme
puts new_editor  # => New Editor [BETA]: ON - Redesigned text editor
```

---

## 43.6 Custom Getters ด้วย Validation

บางครั้งเราต้องการ validation หรือ transformation เมื่ออ่านหรือเขียน

```crystal
class BankAccount
  getter account_number : String
  getter owner : String
  
  def initialize(@account_number : String, @owner : String, initial_balance : Float64 = 0.0)
    @balance = initial_balance
    @transactions = [] of {Float64, String, Time}
  end
  
  # Custom getter ที่ไม่ให้เห็น balance โดยตรง
  def balance : Float64
    @balance
  end
  
  # Custom getter ที่ mask account number
  def masked_account : String
    "****" + @account_number[-4..]
  end
  
  # Custom setter พร้อม validation
  def deposit(amount : Float64, description : String = "Deposit")
    raise ArgumentError.new("Amount must be positive") if amount <= 0
    @balance += amount
    @transactions << {amount, description, Time.local}
  end
  
  def withdraw(amount : Float64, description : String = "Withdrawal")
    raise ArgumentError.new("Amount must be positive") if amount <= 0
    raise "Insufficient funds" if amount > @balance
    @balance -= amount
    @transactions << {-amount, description, Time.local}
  end
  
  def transaction_history : Array({Float64, String, Time})
    @transactions.dup  # คืน copy ไม่ใช่ reference
  end
  
  def statement : String
    lines = ["Account: #{masked_account}", "Owner: #{owner}", "Balance: #{balance}"]
    lines << "--- Transactions ---"
    @transactions.each do |amount, desc, time|
      lines << "#{amount > 0 ? "+" : ""}#{amount} #{desc}"
    end
    lines.join("\n")
  end
end

account = BankAccount.new("1234567890", "Alice", 1000.0)
account.deposit(500.0, "Salary")
account.withdraw(200.0, "Rent")
account.deposit(100.0, "Bonus")

puts account.statement
puts "Masked: #{account.masked_account}"
```

---

## 43.7 Read-Only Properties

```crystal
class ImmutableConfig
  # getter เท่านั้น - ไม่มี setter
  getter host : String
  getter port : Int32
  getter ssl : Bool
  getter timeout : Float64
  
  def initialize(
    @host : String,
    @port : Int32,
    @ssl : Bool = false,
    @timeout : Float64 = 30.0
  )
  end
  
  # สร้าง copy ด้วยค่าที่เปลี่ยน (immutable pattern)
  def with_host(host : String) : ImmutableConfig
    ImmutableConfig.new(host, @port, @ssl, @timeout)
  end
  
  def with_port(port : Int32) : ImmutableConfig
    ImmutableConfig.new(@host, port, @ssl, @timeout)
  end
  
  def with_ssl(ssl : Bool = true) : ImmutableConfig
    ImmutableConfig.new(@host, @port, ssl, @timeout)
  end
  
  def to_s(io : IO) : Nil
    protocol = @ssl ? "https" : "http"
    io << "#{protocol}://#{@host}:#{@port}"
  end
end

config = ImmutableConfig.new("localhost", 8080)
puts config  # => http://localhost:8080

# Method chaining กับ immutable config
secure_config = config.with_host("api.example.com").with_port(443).with_ssl
puts secure_config  # => https://api.example.com:443
puts config         # => http://localhost:8080 (ไม่เปลี่ยน)
```

---

## 43.8 Protected Getters

```crystal
class Node
  property value : Int32
  
  # Protected: accessible ใน subclass และ class เดียวกัน
  protected property next_node : Node?
  protected property prev_node : Node?
  
  def initialize(@value : Int32)
    @next_node = nil
    @prev_node = nil
  end
  
  def has_next? : Bool
    !@next_node.nil?
  end
end

class LinkedList
  def initialize
    @head = nil.as(Node?)
    @tail = nil.as(Node?)
    @size = 0
  end
  
  def push(value : Int32)
    node = Node.new(value)
    if @tail
      @tail.not_nil!.next_node = node
      node.prev_node = @tail
    else
      @head = node
    end
    @tail = node
    @size += 1
  end
  
  def each
    current = @head
    while current
      yield current.value
      current = current.next_node  # ใช้ได้เพราะอยู่ใน class เดียวกัน
    end
  end
  
  def size : Int32
    @size
  end
end

list = LinkedList.new
list.push(1)
list.push(2)
list.push(3)

list.each { |v| print "#{v} " }
puts
# 1 2 3
```

---

## 43.9 Computed Properties

Properties ที่คำนวณจาก properties อื่น

```crystal
class Rectangle
  property width : Float64
  property height : Float64
  
  def initialize(@width : Float64, @height : Float64)
  end
  
  # Computed properties (ไม่มี setter)
  def area : Float64
    @width * @height
  end
  
  def perimeter : Float64
    2 * (@width + @height)
  end
  
  def diagonal : Float64
    Math.sqrt(@width**2 + @height**2)
  end
  
  def aspect_ratio : Float64
    @width / @height
  end
  
  def square? : Bool
    @width == @height
  end
  
  def to_s(io : IO) : Nil
    io << "Rectangle(#{@width}x#{@height})"
    io << " area=#{area.round(2)}"
    io << " perimeter=#{perimeter.round(2)}"
  end
end

rect = Rectangle.new(4.0, 6.0)
puts rect
# Rectangle(4.0x6.0) area=24.0 perimeter=20.0

puts rect.diagonal.round(4)   # => 7.2111
puts rect.aspect_ratio.round(4)  # => 0.6667
puts rect.square?             # => false

rect.width = 6.0
puts rect.square?             # => true
```

---

## 43.10 Property Pattern ขั้นสูง

### Lazy Properties

```crystal
class ExpensiveComputation
  property name : String
  
  def initialize(@name : String)
    @_result = nil.as(Int64?)
  end
  
  # Lazy property - คำนวณเมื่อต้องการเท่านั้น
  def result : Int64
    @_result ||= compute_result
  end
  
  def clear_cache
    @_result = nil
  end
  
  private def compute_result : Int64
    puts "Computing #{@name}..."
    # จำลองการคำนวณที่ใช้เวลานาน
    (1..1000).reduce(0_i64) { |sum, i| sum + i }
  end
end

comp = ExpensiveComputation.new("sum_1_to_1000")
puts "Before accessing result"
puts comp.result  # Computing sum_1_to_1000...
                  # 500500
puts comp.result  # (ไม่ compute ซ้ำ) 500500
```

### Observable Properties

```crystal
class Observable
  @observers = [] of Proc(String, Nil)
  
  def on_change(&block : String ->)
    @observers << block
  end
  
  protected def notify(property_name : String)
    @observers.each { |obs| obs.call(property_name) }
  end
end

class ObservableUser < Observable
  getter name : String
  getter email : String
  getter age : Int32
  
  def initialize(@name : String, @email : String, @age : Int32)
  end
  
  def name=(value : String)
    @name = value
    notify("name")
  end
  
  def email=(value : String)
    @email = value
    notify("email")
  end
  
  def age=(value : Int32)
    @age = value
    notify("age")
  end
end

user = ObservableUser.new("Alice", "alice@example.com", 30)
user.on_change { |prop| puts "Property '#{prop}' changed!" }

user.name = "Bob"    # => Property 'name' changed!
user.email = "bob@example.com"  # => Property 'email' changed!
user.age = 25        # => Property 'age' changed!
```

---

## 43.11 ตัวอย่างรวม: Product Class

```crystal
class Product
  # Read-only
  getter id : UUID
  getter created_at : Time
  
  # Read-write ทั่วไป
  property name : String
  property description : String
  
  # Bool properties
  property? active : Bool = true
  property? featured : Bool = false
  property? in_stock : Bool = true
  
  # Validated properties
  def initialize(@name : String, @description : String, price : Float64, stock : Int32 = 0)
    @id = UUID.random
    @created_at = Time.local
    @price = validate_price(price)
    @stock = validate_stock(stock)
    @discount = 0.0
    @in_stock = stock > 0
  end
  
  def price : Float64
    @price
  end
  
  def price=(value : Float64)
    @price = validate_price(value)
  end
  
  def stock : Int32
    @stock
  end
  
  def stock=(value : Int32)
    @stock = validate_stock(value)
    @in_stock = value > 0
  end
  
  def discount : Float64
    @discount
  end
  
  def discount=(value : Float64)
    raise ArgumentError.new("Discount must be 0-100%") unless (0.0..100.0).includes?(value)
    @discount = value
  end
  
  def final_price : Float64
    @price * (1 - @discount / 100.0)
  end
  
  def to_s(io : IO) : Nil
    io << "Product[#{@name}]"
    io << " $#{final_price.round(2)}"
    io << " (#{discount}% off)" if discount > 0
    io << " OUT OF STOCK" unless in_stock?
  end
  
  private def validate_price(price : Float64) : Float64
    raise ArgumentError.new("Price cannot be negative") if price < 0
    price
  end
  
  private def validate_stock(stock : Int32) : Int32
    raise ArgumentError.new("Stock cannot be negative") if stock < 0
    stock
  end
end

product = Product.new("Crystal Book", "Learn Crystal programming", 39.99, 10)
puts product  # => Product[Crystal Book] $39.99

product.discount = 20.0
puts product  # => Product[Crystal Book] $31.99 (20.0% off)

product.stock = 0
puts product  # => Product[Crystal Book] $31.99 (20.0% off) OUT OF STOCK
puts product.in_stock?   # => false
puts product.featured?   # => false

product.featured = true
puts product.featured?   # => true
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง Employee Class

```crystal
class Employee
  getter id : Int32
  getter hired_at : Time
  
  property name : String
  property department : String
  property? full_time : Bool = true
  property? remote : Bool = false
  
  def initialize(@id : Int32, @name : String, @department : String, salary : Float64)
    @hired_at = Time.local
    @salary = validate_salary(salary)
    @performance_score = 0.0
  end
  
  def salary : Float64
    @salary
  end
  
  def salary=(value : Float64)
    @salary = validate_salary(value)
  end
  
  def performance_score : Float64
    @performance_score
  end
  
  def performance_score=(value : Float64)
    raise "Score must be 0-5" unless (0.0..5.0).includes?(value)
    @performance_score = value
  end
  
  # Computed properties
  def annual_salary : Float64
    @salary * 12
  end
  
  def bonus : Float64
    annual_salary * (@performance_score / 5.0) * 0.1
  end
  
  def total_compensation : Float64
    annual_salary + bonus
  end
  
  def to_s(io : IO) : Nil
    type = full_time? ? "Full-time" : "Part-time"
    location = remote? ? "Remote" : "On-site"
    io << "#{@name} (##{@id}) - #{@department} - #{type} #{location}"
    io << " - Base: $#{annual_salary.round(2)}"
  end
  
  private def validate_salary(salary : Float64) : Float64
    raise ArgumentError.new("Salary must be positive") if salary <= 0
    salary
  end
end

emp = Employee.new(1, "Alice", "Engineering", 8000.0)
puts emp

emp.performance_score = 4.5
emp.remote = true
puts "Annual: $#{emp.annual_salary}"
puts "Bonus: $#{emp.bonus.round(2)}"
puts "Total: $#{emp.total_compensation.round(2)}"
puts emp
```

### แบบฝึกหัดที่ 2: Settings Class

```crystal
class AppSettings
  property app_name : String = "My App"
  property version : String = "1.0.0"
  property? debug_mode : Bool = false
  property? maintenance_mode : Bool = false
  property max_retries : Int32 = 3
  property timeout_seconds : Float64 = 30.0
  
  # Read-only computed
  def environment : String
    debug_mode? ? "development" : "production"
  end
  
  def status : String
    maintenance_mode? ? "maintenance" : "running"
  end
  
  def to_s(io : IO) : Nil
    io << "#{app_name} v#{version}"
    io << " [#{environment}]"
    io << " - #{status}"
  end
end

settings = AppSettings.new
puts settings  # => My App v1.0.0 [production] - running

settings.debug_mode = true
settings.app_name = "Crystal Course"
settings.version = "2.0.0"
puts settings  # => Crystal Course v2.0.0 [development] - running

settings.maintenance_mode = true
puts settings  # => Crystal Course v2.0.0 [development] - maintenance
```

---

## สรุป

Crystal มี macros สำหรับ property accessors ดังนี้:

| Macro | ผลลัพธ์ | ตัวอย่าง |
|-------|---------|---------|
| `getter` | อ่านได้อย่างเดียว | `getter name : String` |
| `setter` | เขียนได้อย่างเดียว | `setter password : String` |
| `property` | อ่านและเขียนได้ | `property age : Int32` |
| `getter?` | getter คืน bool | `getter? active : Bool` |
| `property?` | getter+setter bool | `property? enabled : Bool` |

**ข้อควรจำ:**
- `getter` ป้องกันการแก้ไขจากภายนอก
- `setter` เหมาะกับ write-only properties (เช่น password)
- `property` ใช้เมื่อต้องการทั้งอ่านและเขียน
- `?` suffix ทำให้ method name ลงท้ายด้วย `?` ซึ่งเป็น convention ของ boolean
- Custom getters/setters ให้ใส่ logic validation ได้

---

*ต่อไป: Part 44 - Protected และ Private*
