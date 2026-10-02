# Part 32: Instance Variables และ Methods

## บทนำ

Instance Variables คือตัวแปรที่เก็บ state ของแต่ละ object โดยใช้สัญลักษณ์ `@` นำหน้า ใน Crystal ทุก instance variable ต้องมีการกำหนดค่าใน `initialize` หรือมี type annotation ที่ชัดเจน

---

## 1. Instance Variables พื้นฐาน

### 1.1 การประกาศและใช้งาน

```crystal
class Person
  def initialize(name : String, age : Int32)
    @name = name  # instance variable
    @age = age    # instance variable
  end

  def introduce
    "สวัสดี ฉันชื่อ #{@name} อายุ #{@age} ปี"
  end
end

p = Person.new("สมชาย", 25)
puts p.introduce  # => สวัสดี ฉันชื่อ สมชาย อายุ 25 ปี
```

### 1.2 Instance Variables กับ Type Inference

```crystal
class Box
  def initialize(width : Int32, height : Int32)
    @width = width    # Crystal รู้ว่าเป็น Int32
    @height = height  # Crystal รู้ว่าเป็น Int32
    @label = ""       # String
    @items = [] of String  # Array(String)
  end

  def area : Int32
    @width * @height
  end

  def add_item(item : String)
    @items << item
  end
end
```

### 1.3 Type Annotation สำหรับ Instance Variables

```crystal
class Config
  # ประกาศ type ของ instance variables อย่างชัดเจน
  @host : String
  @port : Int32
  @debug : Bool
  @tags : Array(String)

  def initialize
    @host = "localhost"
    @port = 3000
    @debug = false
    @tags = [] of String
  end
end
```

---

## 2. Getter และ Setter

### 2.1 Manual Getter และ Setter

```crystal
class Temperature
  def initialize(celsius : Float64)
    @celsius = celsius
  end

  # Getter - อ่านค่า
  def celsius : Float64
    @celsius
  end

  # Setter - กำหนดค่า (ชื่อ method + =)
  def celsius=(value : Float64)
    @celsius = value
  end

  def fahrenheit : Float64
    @celsius * 9.0 / 5.0 + 32
  end
end

temp = Temperature.new(25.0)
puts temp.celsius      # => 25.0
puts temp.fahrenheit   # => 77.0

temp.celsius = 100.0
puts temp.celsius      # => 100.0
puts temp.fahrenheit   # => 212.0
```

### 2.2 Crystal Getter/Setter Macros

```crystal
class Student
  # getter: สร้างเมธอด @name สำหรับอ่านค่า
  getter name : String

  # setter: สร้างเมธอด name= สำหรับกำหนดค่า
  setter age : Int32

  # property: สร้างทั้ง getter และ setter
  property email : String

  def initialize(@name : String, @age : Int32, @email : String)
  end
end

s = Student.new("สมชาย", 20, "somchai@example.com")

# ใช้ getter
puts s.name   # => สมชาย

# ใช้ setter
s.age = 21
# s.name = "อื่น"  # Error! ไม่มี setter สำหรับ name

# ใช้ property (ทั้ง get และ set)
puts s.email           # => somchai@example.com
s.email = "new@email.com"
puts s.email           # => new@email.com
```

### 2.3 getter? - Boolean Getter

```crystal
class User
  property name : String
  property? active : Bool  # สร้าง active? method
  property? admin : Bool

  def initialize(@name : String)
    @active = true
    @admin = false
  end
end

user = User.new("สมชาย")
puts user.active?  # => true
puts user.admin?   # => false

user.admin = true
puts user.admin?   # => true
```

---

## 3. Property Macro - Accessor Patterns

### 3.1 getter, setter, property

```crystal
class Product
  getter id : Int32           # read-only
  getter name : String        # read-only
  property price : Float64    # read-write
  property? available : Bool  # boolean read-write

  def initialize(@id : Int32, @name : String, @price : Float64)
    @available = true
  end

  def discount(percent : Float64)
    @price *= (1 - percent / 100)
  end
end

product = Product.new(1, "แล็ปท็อป", 25000.0)

puts product.id       # => 1
puts product.name     # => แล็ปท็อป
puts product.price    # => 25000.0
puts product.available?  # => true

product.price = 22000.0
product.available = false

puts product.price       # => 22000.0
puts product.available?  # => false

# product.id = 2  # Error! id เป็น getter-only
```

### 3.2 Custom Getter/Setter กับ Validation

```crystal
class Circle
  getter radius : Float64

  def initialize(radius : Float64)
    validate_radius(radius)
    @radius = radius
  end

  # Custom setter พร้อม validation
  def radius=(value : Float64)
    validate_radius(value)
    @radius = value
  end

  def area : Float64
    Math::PI * @radius ** 2
  end

  private def validate_radius(value : Float64)
    raise ArgumentError.new("รัศมีต้องมากกว่า 0") if value <= 0
  end
end

c = Circle.new(5.0)
puts c.area.round(2)  # => 78.54

c.radius = 10.0
puts c.area.round(2)  # => 314.16

# c.radius = -1.0  # => ArgumentError: รัศมีต้องมากกว่า 0
```

### 3.3 Computed Properties

```crystal
class Rectangle
  property width : Float64
  property height : Float64

  def initialize(@width : Float64, @height : Float64)
  end

  # Computed properties (ไม่ใช่ instance variable)
  def area : Float64
    @width * @height
  end

  def perimeter : Float64
    2 * (@width + @height)
  end

  def diagonal : Float64
    Math.sqrt(@width ** 2 + @height ** 2)
  end

  def square? : Bool
    @width == @height
  end
end

rect = Rectangle.new(4.0, 6.0)
puts "พื้นที่: #{rect.area}"
puts "เส้นรอบรูป: #{rect.perimeter}"
puts "เส้นทแยง: #{rect.diagonal.round(2)}"
puts "เป็นสี่เหลี่ยมจัตุรัส: #{rect.square?}"
```

---

## 4. initialize - Constructor

### 4.1 Constructor พื้นฐาน

```crystal
class Car
  getter brand : String
  getter model : String
  getter year : Int32
  property mileage : Float64

  def initialize(@brand : String, @model : String, @year : Int32)
    @mileage = 0.0
    @running = false
  end

  def drive(km : Float64)
    @mileage += km
    puts "ขับ #{@brand} #{@model} ระยะทาง #{km} กม."
  end

  def info : String
    "#{@year} #{@brand} #{@model} - #{@mileage.round(1)} กม."
  end
end

car = Car.new("Toyota", "Camry", 2023)
car.drive(150.5)
car.drive(80.2)
puts car.info  # => 2023 Toyota Camry - 230.7 กม.
```

### 4.2 Named Parameters ใน initialize

```crystal
class DatabaseConfig
  getter host : String
  getter port : Int32
  getter database : String
  getter username : String
  getter password : String

  def initialize(
    @host : String = "localhost",
    @port : Int32 = 5432,
    @database : String = "mydb",
    @username : String = "postgres",
    @password : String = ""
  )
  end

  def connection_string : String
    "postgresql://#{@username}:#{@password}@#{@host}:#{@port}/#{@database}"
  end
end

# ใช้ default values
dev_db = DatabaseConfig.new
puts dev_db.connection_string
# => postgresql://postgres:@localhost:5432/mydb

# ระบุเฉพาะค่าที่ต้องการ
prod_db = DatabaseConfig.new(
  host: "db.example.com",
  database: "production",
  username: "app_user",
  password: "s3cr3t"
)
puts prod_db.connection_string
```

---

## 5. Instance Methods

### 5.1 เมธอดที่ทำงานกับ State

```crystal
class ShoppingCart
  def initialize
    @items = {} of String => Int32
    @prices = {} of String => Float64
  end

  def add(item : String, price : Float64, quantity : Int32 = 1)
    @items[item] = (@items[item]? || 0) + quantity
    @prices[item] = price
    puts "เพิ่ม #{item} x#{quantity}"
  end

  def remove(item : String)
    if @items.has_key?(item)
      @items.delete(item)
      @prices.delete(item)
      puts "ลบ #{item} ออกจากตะกร้า"
    else
      puts "ไม่มี #{item} ในตะกร้า"
    end
  end

  def total : Float64
    @items.sum { |item, qty| (@prices[item]? || 0.0) * qty }
  end

  def item_count : Int32
    @items.values.sum
  end

  def show
    puts "=== ตะกร้าสินค้า ==="
    @items.each do |item, qty|
      price = @prices[item]? || 0.0
      puts "  #{item} x#{qty} = ฿#{(price * qty).round(2)}"
    end
    puts "รวม: ฿#{total.round(2)}"
  end
end

cart = ShoppingCart.new
cart.add("แอปเปิล", 30.0, 3)
cart.add("กล้วย", 15.0, 5)
cart.add("ส้ม", 25.0, 2)
cart.show
cart.remove("กล้วย")
cart.show
```

### 5.2 เมธอดที่ Modify State

```crystal
class Stack
  def initialize
    @data = [] of Int32
  end

  def push(value : Int32)
    @data << value
  end

  def pop : Int32?
    @data.pop?
  end

  def peek : Int32?
    @data.last?
  end

  def empty? : Bool
    @data.empty?
  end

  def size : Int32
    @data.size
  end

  def clear
    @data.clear
  end

  def to_s : String
    "Stack#{@data.inspect}"
  end
end

stack = Stack.new
stack.push(1)
stack.push(2)
stack.push(3)

puts stack         # => Stack[1, 2, 3]
puts stack.peek    # => 3
puts stack.pop     # => 3
puts stack         # => Stack[1, 2]
puts stack.size    # => 2
```

---

## 6. self - การอ้างอิง Instance ปัจจุบัน

### 6.1 การใช้ self

```crystal
class Node
  property value : Int32
  property next_node : Node?

  def initialize(@value : Int32)
    @next_node = nil
  end

  def append(value : Int32) : Node
    new_node = Node.new(value)
    if last = find_last
      last.next_node = new_node
    else
      self.next_node = new_node
    end
    self  # ส่งคืน self เพื่อ chaining
  end

  def find_last : Node?
    current = self
    while next_n = current.next_node
      current = next_n
    end
    current
  end

  def to_a : Array(Int32)
    result = [] of Int32
    current : Node? = self
    while node = current
      result << node.value
      current = node.next_node
    end
    result
  end
end

head = Node.new(1)
head.append(2).append(3).append(4).append(5)
puts head.to_a.inspect  # => [1, 2, 3, 4, 5]
```

### 6.2 self ใน Comparison Methods

```crystal
class Version
  include Comparable(Version)

  getter major : Int32
  getter minor : Int32
  getter patch : Int32

  def initialize(@major : Int32, @minor : Int32, @patch : Int32)
  end

  def <=>(other : Version) : Int32
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    patch <=> other.patch
  end

  def to_s : String
    "#{@major}.#{@minor}.#{@patch}"
  end
end

v1 = Version.new(1, 2, 3)
v2 = Version.new(1, 3, 0)
v3 = Version.new(2, 0, 0)
v4 = Version.new(1, 2, 3)

puts v1 < v2   # => true
puts v2 < v3   # => true
puts v1 == v4  # => true

versions = [v3, v1, v2]
puts versions.sort.map(&.to_s).inspect
# => ["1.2.3", "1.3.0", "2.0.0"]
```

---

## 7. ตัวอย่างขั้นสูง

### 7.1 Linked List

```crystal
class LinkedList(T)
  private class Node(T)
    property value : T
    property next_node : Node(T)?

    def initialize(@value : T)
      @next_node = nil
    end
  end

  def initialize
    @head : Node(T)? = nil
    @size = 0
  end

  def push(value : T)
    new_node = Node(T).new(value)
    new_node.next_node = @head
    @head = new_node
    @size += 1
  end

  def pop : T?
    return nil unless head = @head
    @head = head.next_node
    @size -= 1
    head.value
  end

  def size : Int32
    @size
  end

  def to_a : Array(T)
    result = [] of T
    current = @head
    while node = current
      result << node.value
      current = node.next_node
    end
    result
  end
end

list = LinkedList(Int32).new
list.push(1)
list.push(2)
list.push(3)
puts list.to_a.inspect  # => [3, 2, 1]
puts list.pop           # => 3
puts list.size          # => 2
```

### 7.2 Observable Property Pattern

```crystal
class ObservableProperty(T)
  def initialize(@value : T)
    @callbacks = [] of T ->
  end

  def value : T
    @value
  end

  def value=(new_val : T)
    old_val = @value
    @value = new_val
    @callbacks.each { |cb| cb.call(new_val) } if old_val != new_val
  end

  def on_change(&callback : T ->)
    @callbacks << callback
  end
end

class UserProfile
  property name : String
  @score : ObservableProperty(Int32)

  def initialize(@name : String)
    @score = ObservableProperty(Int32).new(0)
    
    @score.on_change do |new_score|
      puts "คะแนนของ #{@name} เปลี่ยนเป็น #{new_score}"
    end
  end

  def score : Int32
    @score.value
  end

  def score=(value : Int32)
    @score.value = value
  end
end

user = UserProfile.new("สมชาย")
user.score = 100  # => คะแนนของ สมชาย เปลี่ยนเป็น 100
user.score = 150  # => คะแนนของ สมชาย เปลี่ยนเป็น 150
user.score = 150  # ไม่มีการเปลี่ยนแปลง
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Bank Account

```crystal
class BankAccount
  getter account_number : String
  getter owner_name : String
  getter balance : Float64

  def initialize(@account_number : String, @owner_name : String, initial_balance : Float64 = 0.0)
    @balance = initial_balance
    @transactions = [] of {type: String, amount: Float64, balance: Float64}
  end

  def deposit(amount : Float64)
    raise ArgumentError.new("จำนวนต้องมากกว่า 0") unless amount > 0
    @balance += amount
    record_transaction("ฝาก", amount)
    self
  end

  def withdraw(amount : Float64)
    raise ArgumentError.new("จำนวนต้องมากกว่า 0") unless amount > 0
    raise "ยอดเงินไม่เพียงพอ" if amount > @balance
    @balance -= amount
    record_transaction("ถอน", amount)
    self
  end

  def statement
    puts "=== บัญชีเลขที่ #{@account_number} ==="
    puts "เจ้าของ: #{@owner_name}"
    puts "ยอดปัจจุบัน: ฿#{@balance}"
    puts "\nประวัติรายการ:"
    @transactions.each do |t|
      puts "  #{t[:type]}: ฿#{t[:amount]} (คงเหลือ: ฿#{t[:balance]})"
    end
  end

  private def record_transaction(type : String, amount : Float64)
    @transactions << {type: type, amount: amount, balance: @balance}
  end
end

account = BankAccount.new("ACC001", "สมชาย ใจดี", 1000.0)
account.deposit(500.0).deposit(200.0).withdraw(300.0)
account.statement
```

### แบบฝึกหัดที่ 2: Inventory System

```crystal
class InventoryItem
  getter sku : String
  getter name : String
  property price : Float64
  getter quantity : Int32

  def initialize(@sku : String, @name : String, @price : Float64, initial_qty : Int32 = 0)
    @quantity = initial_qty
  end

  def restock(qty : Int32)
    raise ArgumentError.new("จำนวนต้องมากกว่า 0") unless qty > 0
    @quantity += qty
    puts "เติมสต็อก #{@name}: +#{qty} (รวม: #{@quantity})"
  end

  def sell(qty : Int32) : Float64
    raise ArgumentError.new("สต็อกไม่เพียงพอ") if qty > @quantity
    @quantity -= qty
    revenue = @price * qty
    puts "ขาย #{@name}: #{qty} ชิ้น = ฿#{revenue}"
    revenue
  end

  def low_stock? : Bool
    @quantity < 10
  end

  def out_of_stock? : Bool
    @quantity == 0
  end

  def total_value : Float64
    @price * @quantity
  end

  def to_s : String
    "#{@sku} | #{@name} | ฿#{@price} | #{@quantity} ชิ้น"
  end
end

items = [
  InventoryItem.new("SKU001", "แอปเปิล", 30.0, 100),
  InventoryItem.new("SKU002", "กล้วย", 15.0, 5),
  InventoryItem.new("SKU003", "ส้ม", 25.0, 0),
]

puts "=== รายการสต็อก ==="
items.each { |item| puts item }

puts "\n=== การซื้อขาย ==="
items[0].sell(20)
items[1].sell(3)

puts "\n=== สินค้าใกล้หมด ==="
items.select(&.low_stock?).each { |item| puts "⚠️  #{item.name}" }
```

---

## สรุป

Instance Variables และ Methods เป็นหัวใจของ OOP ใน Crystal:

| องค์ประกอบ | Syntax | คำอธิบาย |
|-----------|--------|---------|
| Instance variable | `@name` | เก็บ state ของ object |
| Getter | `getter name` | อ่านค่า |
| Setter | `setter name` | กำหนดค่า |
| Property | `property name` | ทั้ง getter และ setter |
| Boolean getter | `property? active` | สร้าง `active?` method |
| self | `self` | อ้างอิง instance ปัจจุบัน |

**Best Practices:**
- ใช้ `getter` สำหรับ read-only fields
- ใช้ `property` สำหรับ read-write fields
- เพิ่ม validation ใน custom setter เมื่อจำเป็น
- ใช้ `private` สำหรับ helper methods ที่ไม่ควรเรียกจากภายนอก
- ให้ `initialize` จัดการ initial state ทั้งหมด
