# Part 39: Include และ Extend

## บทนำ

Crystal มีสองวิธีหลักในการนำ module มาใช้กับ class:
- **`include`** - เพิ่ม module methods เป็น instance methods ของ class
- **`extend`** - เพิ่ม module methods เป็น class methods

---

## 1. include - Instance Methods

### 1.1 include พื้นฐาน

```crystal
module Greetable
  def greet
    "สวัสดี ฉันชื่อ #{name}"
  end

  def farewell
    "ลาก่อนจาก #{name}"
  end
end

class Person
  include Greetable  # เพิ่มเป็น instance methods

  getter name : String

  def initialize(@name : String)
  end
end

p = Person.new("สมชาย")
puts p.greet    # => สวัสดี ฉันชื่อ สมชาย
puts p.farewell # => ลาก่อนจาก สมชาย
```

### 1.2 Module Methods กลายเป็น Instance Methods

```crystal
module Serializable
  def to_csv : String
    instance_variable_names.map { |var|
      instance_variable_get(var).to_s
    }.join(",")
  end

  def to_key_value : String
    instance_variable_names.map { |var|
      "#{var.lstrip('@')}=#{instance_variable_get(var)}"
    }.join(", ")
  end
end

class User
  include Serializable

  def initialize(@name : String, @age : Int32, @email : String)
  end
end

u = User.new("สมชาย", 25, "somchai@example.com")
# (Note: instance_variable_names is a simplified example)
```

### 1.3 Include หลาย Modules

```crystal
module Walkable
  def walk
    puts "#{name} เดิน"
  end
end

module Swimmable
  def swim
    puts "#{name} ว่ายน้ำ"
  end
end

module Flyable
  def fly
    puts "#{name} บิน"
  end
end

class Duck
  include Walkable
  include Swimmable
  include Flyable

  getter name : String

  def initialize(@name : String)
  end
end

class Human
  include Walkable
  include Swimmable

  getter name : String

  def initialize(@name : String)
  end
end

duck = Duck.new("โดนัลด์")
duck.walk   # => โดนัลด์ เดิน
duck.swim   # => โดนัลด์ ว่ายน้ำ
duck.fly    # => โดนัลด์ บิน

human = Human.new("สมชาย")
human.walk  # => สมชาย เดิน
human.swim  # => สมชาย ว่ายน้ำ
# human.fly  # Error! Human ไม่สามารถบินได้
```

---

## 2. extend - Class Methods

### 2.1 extend พื้นฐาน

```crystal
module ClassUtils
  def create_from_hash(hash : Hash(String, String))
    new(**hash.transform_keys(&.to_sym))
  end

  def count : Int32
    @@count ||= 0
  end
end

class Product
  extend ClassUtils  # เพิ่มเป็น class methods

  getter name : String
  getter price : Float64

  def initialize(@name : String, @price : Float64)
  end
end

# ใช้ class method จาก module
# Product.create_from_hash({"name" => "Apple", "price" => "30.0"})
```

### 2.2 extend สำหรับ Class-Level Behavior

```crystal
module Findable
  # Methods เหล่านี้จะกลายเป็น class methods เมื่อ extend
  def find(id : Int32)
    all.find { |item| item.id == id }
  end

  def where(&predicate : self ->)
    all.select { |item| predicate.call(item) }
  end

  def all : Array(self)
    @instances ||= [] of self
  end
end

class User
  extend Findable

  @@all_users = [] of User

  getter id : Int32
  getter name : String
  getter? admin : Bool

  def self.all : Array(User)
    @@all_users
  end

  def initialize(@id : Int32, @name : String, admin : Bool = false)
    @admin = admin
    @@all_users << self
  end
end

User.new(1, "สมชาย")
User.new(2, "สมหญิง", admin: true)
User.new(3, "สมศักดิ์")

found = User.find(2)
puts found.try(&.name)  # => สมหญิง

admins = User.where { |u| u.admin? }
puts admins.map(&.name).inspect  # => ["สมหญิง"]
```

---

## 3. Include และ Extend ร่วมกัน

### 3.1 ใช้ทั้งสองใน Class เดียว

```crystal
module Logging
  # Class methods (ใช้ extend)
  module ClassMethods
    def log_class(msg : String)
      puts "[#{self.name}] #{msg}"
    end
  end

  # Instance methods (ใช้ include)
  def log(msg : String)
    puts "[#{self.class.name}##{object_id}] #{msg}"
  end
end

class Service
  include Logging             # instance methods
  extend Logging::ClassMethods  # class methods

  def run
    log("กำลังทำงาน...")  # instance method
  end
end

Service.log_class("Class initialized")  # class method
svc = Service.new
svc.run  # instance method
```

### 3.2 self.included Hook

```crystal
module AutoExtend
  def self.included(base : Class)
    # เมื่อ module ถูก include, extend ClassMethods อัตโนมัติ
    base.extend(ClassMethods)
  end

  module ClassMethods
    def class_info
      puts "Class: #{self.name}"
    end
  end

  def instance_info
    puts "Instance of: #{self.class.name}"
  end
end

class Widget
  include AutoExtend
end

Widget.class_info     # => Class: Widget
Widget.new.instance_info  # => Instance of: Widget
```

---

## 4. Module Method Resolution Order (MRO)

### 4.1 ลำดับการค้นหา Method

```crystal
module A
  def who_am_i
    "Module A"
  end
end

module B
  def who_am_i
    "Module B"
  end
end

class C
  include A
  include B  # B ถูก include ทีหลัง จึง override A

  def describe
    puts who_am_i  # จะใช้ B เพราะ include ทีหลัง
  end
end

c = C.new
puts c.who_am_i  # => Module B
c.describe       # => Module B
```

### 4.2 Class Override Module

```crystal
module Describable
  def describe
    "Object จาก module Describable"
  end
end

class MyClass
  include Describable

  # Class method override module method
  def describe
    "#{super} - แต่ถูก override โดย MyClass"
  end
end

obj = MyClass.new
puts obj.describe
# => Object จาก module Describable - แต่ถูก override โดย MyClass
```

### 4.3 MRO ใน Crystal

```crystal
# Crystal ใช้ C3 linearization algorithm
# ลำดับ: Class -> ทุก include (จากหลังสุดก่อน) -> superclass

module M1
  def who
    "M1"
  end
end

module M2
  def who
    "M2"
  end
end

module M3
  def who
    "M3"
  end
end

class Base
  include M1

  def who
    "Base"
  end
end

class Child < Base
  include M2
  include M3
end

puts Child.new.who  # => Child method ถ้ามี, ไม่งั้น M3

# ตรวจสอบ ancestors
puts Child.ancestors.inspect
# => [Child, M3, M2, Base, M1, ...]
```

### 4.4 super ใน Module

```crystal
module Logging
  def save
    puts "Logging: กำลัง save..."
    super  # เรียก method ถัดไปใน MRO chain
    puts "Logging: save เสร็จแล้ว"
  end
end

module Validation
  def save
    puts "Validation: ตรวจสอบข้อมูล..."
    super  # เรียก method ถัดไป
    puts "Validation: ผ่านการตรวจสอบ"
  end
end

class Record
  include Logging
  include Validation

  def save
    puts "Record: บันทึกข้อมูล"
  end
end

Record.new.save
# => Validation: ตรวจสอบข้อมูล...
# => Logging: กำลัง save...
# => Record: บันทึกข้อมูล
# => Logging: save เสร็จแล้ว
# => Validation: ผ่านการตรวจสอบ
```

---

## 5. Practical Examples

### 5.1 Comparable Module

```crystal
module Comparable(T)
  # เมื่อ include จะต้อง implement <=>
  abstract def <=>(other : T) : Int32

  def <(other : T) : Bool
    (self <=> other) < 0
  end

  def >(other : T) : Bool
    (self <=> other) > 0
  end

  def <=(other : T) : Bool
    (self <=> other) <= 0
  end

  def >=(other : T) : Bool
    (self <=> other) >= 0
  end

  def ==(other : T) : Bool
    (self <=> other) == 0
  end

  def clamp(min : T, max : T) : T
    if self < min then min
    elsif self > max then max
    else self
    end
  end
end

class Temperature
  include Comparable(Temperature)

  getter celsius : Float64

  def initialize(@celsius : Float64)
  end

  def <=>(other : Temperature) : Int32
    @celsius <=> other.celsius
  end

  def to_s : String
    "#{@celsius}°C"
  end
end

temps = [
  Temperature.new(25.0),
  Temperature.new(-10.0),
  Temperature.new(100.0),
  Temperature.new(37.5),
]

sorted = temps.sort
puts sorted.map(&.to_s).inspect
# => ["-10.0°C", "25.0°C", "37.5°C", "100.0°C"]

t = Temperature.new(15.0)
min = Temperature.new(0.0)
max = Temperature.new(40.0)
puts t.clamp(min, max)  # => 15.0°C

puts Temperature.new(5.0) < Temperature.new(10.0)   # => true
```

### 5.2 Enumerable-like Module

```crystal
module Enumerable(T)
  abstract def each(&block : T ->)

  def map(&block : T -> U) : Array(U) forall U
    result = [] of U
    each { |item| result << block.call(item) }
    result
  end

  def select(&block : T -> Bool) : Array(T)
    result = [] of T
    each { |item| result << item if block.call(item) }
    result
  end

  def reject(&block : T -> Bool) : Array(T)
    select { |item| !block.call(item) }
  end

  def count : Int32
    total = 0
    each { total += 1 }
    total
  end

  def first : T?
    each { |item| return item }
    nil
  end

  def any?(&block : T -> Bool) : Bool
    each { |item| return true if block.call(item) }
    false
  end

  def all?(&block : T -> Bool) : Bool
    each { |item| return false unless block.call(item) }
    true
  end
end

class NumberRange
  include Enumerable(Int32)

  def initialize(@from : Int32, @to : Int32)
  end

  def each(&block : Int32 ->)
    (@from..@to).each { |n| block.call(n) }
  end
end

range = NumberRange.new(1, 10)
puts range.count                        # => 10
puts range.map { |x| x * 2 }.inspect   # => [2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
puts range.select { |x| x.even? }.inspect  # => [2, 4, 6, 8, 10]
puts range.any? { |x| x > 5 }          # => true
puts range.all? { |x| x > 0 }          # => true
```

### 5.3 Observer Pattern ด้วย Modules

```crystal
module Observable
  def self.included(base)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def observers : Array(Proc(Symbol, Hash(Symbol, String), Nil))
      @observers ||= [] of Proc(Symbol, Hash(Symbol, String), Nil)
    end

    def on(event : Symbol, &handler : Hash(Symbol, String) ->)
      observers << handler
    end
  end

  def notify(event : Symbol, data : Hash(Symbol, String) = {} of Symbol => String)
    self.class.observers.each { |obs| obs.call(data) }
  end
end

class Order
  include Observable

  getter id : String
  getter status : String

  def initialize(@id : String)
    @status = "pending"
  end

  def complete
    @status = "completed"
    notify(:completed, {order_id: @id, status: @status})
  end

  def cancel
    @status = "cancelled"
    notify(:cancelled, {order_id: @id, reason: "user cancelled"})
  end
end

Order.on(:completed) { |data| puts "📧 ส่งอีเมลยืนยัน #{data[:order_id]}" }
Order.on(:cancelled) { |data| puts "💔 Order #{data[:order_id]} ถูกยกเลิก" }

order = Order.new("ORD001")
order.complete
# => 📧 ส่งอีเมลยืนยัน ORD001
```

---

## 6. Advanced Patterns

### 6.1 Plugin System ด้วย Module

```crystal
module Plugin
  @@registry = {} of String => Plugin.class

  def self.included(base : Class)
    if base.responds_to?(:plugin_name)
      @@registry[base.plugin_name] = base
    end
  end

  def self.find(name : String) : Plugin.class?
    @@registry[name]?
  end

  def self.all : Hash(String, Plugin.class)
    @@registry
  end

  abstract def execute(context : Hash(String, String)) : String
end

class AuthPlugin
  include Plugin

  def self.plugin_name : String
    "auth"
  end

  def execute(context : Hash(String, String)) : String
    "ตรวจสอบ token: #{context["token"]? || "ไม่มี"}"
  end
end

class LogPlugin
  include Plugin

  def self.plugin_name : String
    "log"
  end

  def execute(context : Hash(String, String)) : String
    "บันทึก: #{context["message"]? || ""}"
  end
end

# ใช้งาน
if plugin_class = Plugin.find("auth")
  plugin = plugin_class.new
  puts plugin.execute({"token" => "abc123"})
end
```

### 6.2 Mixin with State

```crystal
module Cacheable
  def self.included(base : Class)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def cache : Hash(String, String)
      @cache ||= {} of String => String
    end

    def cache_key(id : String) : String
      "#{self.name.downcase}:#{id}"
    end

    def find_cached(id : String) : String?
      cache[cache_key(id)]?
    end

    def store_cached(id : String, value : String)
      cache[cache_key(id)] = value
    end

    def clear_cache
      @cache = {} of String => String
    end
  end
end

class Product
  include Cacheable

  getter id : String
  getter name : String

  def initialize(@id : String, @name : String)
    # Cache เมื่อสร้าง
    self.class.store_cached(@id, @name)
  end

  def self.find(id : String) : Product?
    if cached = find_cached(id)
      puts "Cache hit: #{id}"
      Product.new(id, cached)
    else
      puts "Cache miss: #{id}"
      nil
    end
  end
end

p1 = Product.new("1", "แอปเปิล")
p2 = Product.new("2", "กล้วย")

Product.find("1")  # => Cache hit: 1
Product.find("3")  # => Cache miss: 3
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Validatable Mixin

```crystal
module Validatable
  def valid? : Bool
    validate.empty?
  end

  def validate : Array(String)
    [] of String  # subclass override
  end

  def validate!
    errors = validate
    raise "Validation failed: #{errors.join(", ")}" unless errors.empty?
  end
end

class RegistrationForm
  include Validatable

  property username : String
  property email : String
  property password : String
  property password_confirm : String

  def initialize(@username : String, @email : String,
                 @password : String, @password_confirm : String)
  end

  def validate : Array(String)
    errors = [] of String
    errors << "username ต้องไม่ว่าง" if @username.empty?
    errors << "username ต้องมีความยาว 3-20 ตัวอักษร" unless (3..20).includes?(@username.size)
    errors << "email ไม่ถูกต้อง" unless @email.includes?("@")
    errors << "password ต้องยาวอย่างน้อย 8 ตัว" if @password.size < 8
    errors << "password ไม่ตรงกัน" if @password != @password_confirm
    errors
  end
end

form = RegistrationForm.new("สมชาย", "somchai@example.com", "password123", "password123")
puts form.valid?       # => true
puts form.validate.inspect  # => []

bad_form = RegistrationForm.new("", "invalid", "short", "different")
puts bad_form.valid?   # => false
puts bad_form.validate.each { |e| puts "  - #{e}" }
```

### แบบฝึกหัดที่ 2: Sortable กับ extend

```crystal
module Sortable
  # เมื่อ extend จะกลายเป็น class methods
  def sort_by_field(field : String) : Array(self)
    all.sort_by { |item| item.respond_to?(field) ? item.send(field).to_s : "" }
  end

  def order_by(field : String, direction : String = "asc") : Array(self)
    sorted = sort_by_field(field)
    direction == "desc" ? sorted.reverse : sorted
  end
end

class Employee
  extend Sortable

  @@employees = [] of Employee

  def self.all : Array(Employee)
    @@employees
  end

  getter name : String
  getter salary : Float64
  getter department : String

  def initialize(@name : String, @salary : Float64, @department : String)
    @@employees << self
  end
end

Employee.new("สมชาย", 50000.0, "IT")
Employee.new("สมหญิง", 65000.0, "HR")
Employee.new("สมศักดิ์", 45000.0, "IT")
Employee.new("สมรัก", 70000.0, "Finance")

puts Employee.all.map(&.name).inspect
```

---

## สรุป

Include และ Extend ใน Crystal:

| Feature | Syntax | ผลลัพธ์ |
|---------|--------|---------|
| include | `include ModuleName` | Module methods → instance methods |
| extend | `extend ModuleName` | Module methods → class methods |
| self.included hook | `def self.included(base)` | เรียกเมื่อ module ถูก include |
| MRO | ลำดับ include | กำหนดว่า method ไหนจะ win |

**MRO Rules:**
- Class methods override module methods
- Module ที่ include ทีหลัง override module ที่ include ก่อน
- `super` เรียก method ถัดไปใน MRO chain
- ลำดับ: Class → modules (reverse order) → superclass

**Best Practices:**
- ใช้ `include` สำหรับ instance behavior (what objects *do*)
- ใช้ `extend` สำหรับ class behavior (factory methods, class-level utilities)
- ใช้ `self.included` hook สำหรับ auto-extend patterns
- หลีกเลี่ยงการ include module จำนวนมากใน class เดียว
