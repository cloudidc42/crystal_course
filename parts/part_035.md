# Part 35: Inheritance - การสืบทอด

## บทนำ

Inheritance (การสืบทอด) เป็นหลักการสำคัญของ OOP ที่ให้ class ลูก (child/subclass) สามารถสืบทอด properties และ methods จาก class พ่อแม่ (parent/superclass) ได้ ใน Crystal ใช้ `class Child < Parent` เพื่อระบุการสืบทอด

---

## 1. พื้นฐาน Inheritance

### 1.1 Syntax class Child < Parent

```crystal
# Parent class (Superclass)
class Animal
  getter name : String
  getter sound : String

  def initialize(@name : String, @sound : String)
  end

  def speak
    puts "#{@name} พูดว่า: #{@sound}!"
  end

  def describe
    puts "ฉันคือ #{@name}"
  end
end

# Child class สืบทอดจาก Animal
class Dog < Animal
  def initialize(name : String)
    super(name, "โฮ่ง")  # เรียก parent's initialize
  end

  def fetch(item : String)
    puts "#{@name} วิ่งไปเอา #{item} มาให้!"
  end
end

class Cat < Animal
  def initialize(name : String)
    super(name, "เมี้ยว")
  end

  def purr
    puts "#{@name} ร้องเป็นเสียงครางอย่างพึงพอใจ..."
  end
end

dog = Dog.new("บัดดี้")
cat = Cat.new("วิสเกอร์")

dog.speak    # => บัดดี้ พูดว่า: โฮ่ง!
dog.describe # => ฉันคือ บัดดี้
dog.fetch("ลูกบอล")  # เมธอดของ Dog

cat.speak    # => วิสเกอร์ พูดว่า: เมี้ยว!
cat.purr     # เมธอดของ Cat
```

### 1.2 Method Inheritance

```crystal
class Shape
  def area : Float64
    0.0  # Default implementation
  end

  def perimeter : Float64
    0.0
  end

  def describe
    puts "รูปร่าง: #{self.class.name}"
    puts "พื้นที่: #{area.round(2)}"
    puts "เส้นรอบรูป: #{perimeter.round(2)}"
  end
end

class Circle < Shape
  def initialize(@radius : Float64)
  end

  # Override parent methods
  def area : Float64
    Math::PI * @radius ** 2
  end

  def perimeter : Float64
    2 * Math::PI * @radius
  end
end

class Rectangle < Shape
  def initialize(@width : Float64, @height : Float64)
  end

  def area : Float64
    @width * @height
  end

  def perimeter : Float64
    2 * (@width + @height)
  end
end

shapes = [
  Circle.new(5.0),
  Rectangle.new(4.0, 6.0),
]

shapes.each do |shape|
  shape.describe
  puts "---"
end
```

---

## 2. Overriding Methods

### 2.1 การ Override พื้นฐาน

```crystal
class Vehicle
  getter speed : Float64

  def initialize
    @speed = 0.0
  end

  def accelerate(amount : Float64)
    @speed += amount
    puts "เร่งความเร็ว: #{@speed} km/h"
  end

  def brake(amount : Float64)
    @speed = [@speed - amount, 0.0].max
    puts "ลดความเร็ว: #{@speed} km/h"
  end

  def fuel_type : String
    "น้ำมัน"
  end

  def status
    puts "ความเร็ว: #{@speed} km/h | เชื้อเพลิง: #{fuel_type}"
  end
end

class ElectricCar < Vehicle
  getter battery_level : Float64

  def initialize
    super
    @battery_level = 100.0
  end

  # Override fuel_type
  def fuel_type : String
    "ไฟฟ้า (#{@battery_level.round(1)}%)"
  end

  # Override accelerate เพิ่ม battery drain
  def accelerate(amount : Float64)
    super(amount)  # เรียก parent method
    @battery_level -= amount * 0.1  # ใช้แบต
    @battery_level = [@battery_level, 0.0].max
  end
end

ev = ElectricCar.new
ev.accelerate(50.0)
ev.accelerate(30.0)
ev.status
```

### 2.2 Override to_s

```crystal
class Product
  getter name : String
  getter price : Float64

  def initialize(@name : String, @price : Float64)
  end

  def to_s : String
    "#{@name}: ฿#{@price}"
  end
end

class DiscountedProduct < Product
  getter discount_percent : Float64

  def initialize(name : String, price : Float64, @discount_percent : Float64)
    super(name, price)
  end

  def discounted_price : Float64
    price * (1 - @discount_percent / 100)
  end

  # Override to_s
  def to_s : String
    original = super  # เรียก parent's to_s
    "#{original} (ลด #{@discount_percent}% = ฿#{discounted_price.round(2)})"
  end
end

p1 = Product.new("หนังสือ", 350.0)
p2 = DiscountedProduct.new("หนังสือ", 350.0, 20.0)

puts p1  # => หนังสือ: ฿350.0
puts p2  # => หนังสือ: ฿350.0 (ลด 20.0% = ฿280.0)
```

---

## 3. super - เรียก Parent Method

### 3.1 super ใน initialize

```crystal
class Person
  getter name : String
  getter age : Int32

  def initialize(@name : String, @age : Int32)
    puts "สร้าง Person: #{@name}"
  end
end

class Employee < Person
  getter company : String
  getter salary : Float64

  def initialize(name : String, age : Int32, @company : String, @salary : Float64)
    super(name, age)  # เรียก Person#initialize
    puts "สร้าง Employee ที่ #{@company}"
  end

  def info : String
    "#{@name} (#{@age}) - #{@company} ฿#{@salary}"
  end
end

class Manager < Employee
  getter team_size : Int32

  def initialize(name : String, age : Int32, company : String,
                 salary : Float64, @team_size : Int32)
    super(name, age, company, salary)  # เรียก Employee#initialize
    puts "สร้าง Manager ดูแล #{@team_size} คน"
  end
end

emp = Employee.new("สมชาย", 30, "ABC Corp", 50000.0)
puts emp.info
puts "---"
mgr = Manager.new("สมหญิง", 35, "ABC Corp", 80000.0, 10)
```

### 3.2 super ใน Instance Methods

```crystal
class Logger
  def log(message : String)
    puts "[LOG] #{message}"
  end
end

class TimestampLogger < Logger
  def log(message : String)
    timestamp = Time.local.to_s("%H:%M:%S")
    super("[#{timestamp}] #{message}")  # เพิ่ม timestamp แล้วส่งต่อ
  end
end

class FileLogger < TimestampLogger
  def initialize(@filename : String)
    @buffer = [] of String
  end

  def log(message : String)
    super(message)  # ใช้ timestamp จาก parent
    @buffer << message
  end

  def flush
    puts "บันทึก #{@buffer.size} บรรทัดลงไฟล์ #{@filename}"
    @buffer.clear
  end
end

logger = FileLogger.new("app.log")
logger.log("เริ่มต้นระบบ")
logger.log("ผู้ใช้ล็อกอิน")
logger.log("ทำรายการเสร็จ")
logger.flush
```

---

## 4. Multiple Levels of Inheritance

### 4.1 Hierarchy หลายชั้น

```crystal
class LivingThing
  def breathe
    puts "#{self.class.name} หายใจ"
  end

  def metabolize
    puts "#{self.class.name} เผาผลาญพลังงาน"
  end
end

class Animal < LivingThing
  getter name : String

  def initialize(@name : String)
  end

  def move
    puts "#{@name} เคลื่อนไหว"
  end

  def eat(food : String)
    puts "#{@name} กิน #{food}"
  end
end

class Mammal < Animal
  def nurse_young
    puts "#{@name} เลี้ยงลูกด้วยนม"
  end

  def body_temperature : String
    "อุ่น (warm-blooded)"
  end
end

class Dog < Mammal
  def initialize(name : String)
    super(name)
  end

  def fetch(item : String)
    puts "#{@name} เอา #{item} มาให้"
  end

  def bark
    puts "#{@name}: โฮ่ง!"
  end
end

class GoldenRetriever < Dog
  def initialize(name : String)
    super(name)
  end

  def swim
    puts "#{@name} ว่ายน้ำได้ดีมาก!"
  end
end

buddy = GoldenRetriever.new("บัดดี้")

# สืบทอดจากทุก level
buddy.breathe          # จาก LivingThing
buddy.move             # จาก Animal
buddy.eat("อาหารสุนัข")  # จาก Animal
buddy.nurse_young      # จาก Mammal
buddy.bark             # จาก Dog
buddy.fetch("ลูกบอล")  # จาก Dog
buddy.swim             # ของ GoldenRetriever เอง

puts buddy.body_temperature  # จาก Mammal
```

---

## 5. is_a? กับ Inheritance

### 5.1 Type Checking

```crystal
class Animal; end
class Dog < Animal; end
class Cat < Animal; end
class GoldenRetriever < Dog; end

buddy = GoldenRetriever.new

puts buddy.is_a?(GoldenRetriever)  # => true
puts buddy.is_a?(Dog)              # => true
puts buddy.is_a?(Animal)           # => true
puts buddy.is_a?(Cat)              # => false

# class method
puts buddy.class             # => GoldenRetriever
puts buddy.class.name        # => "GoldenRetriever"
puts buddy.class.superclass  # => Dog
```

### 5.2 Polymorphism ด้วย is_a?

```crystal
class Shape
  def area : Float64
    0.0
  end

  def name : String
    self.class.name
  end
end

class Circle < Shape
  def initialize(@radius : Float64); end
  def area : Float64; Math::PI * @radius ** 2; end
end

class Square < Shape
  def initialize(@side : Float64); end
  def area : Float64; @side ** 2; end
end

class Triangle < Shape
  def initialize(@base : Float64, @height : Float64); end
  def area : Float64; 0.5 * @base * @height; end
end

shapes = [
  Circle.new(5.0),
  Square.new(4.0),
  Triangle.new(3.0, 6.0),
  Circle.new(2.0),
]

# Polymorphism - ทุก shape ใช้ method เดียวกัน
total_area = shapes.sum(&.area)
puts "พื้นที่รวม: #{total_area.round(2)}"

circles = shapes.select { |s| s.is_a?(Circle) }
puts "มี #{circles.size} วงกลม"

# as? สำหรับ safe casting
shapes.each do |shape|
  if circle = shape.as?(Circle)
    puts "วงกลมรัศมี #{circle.@radius}"
  end
end
```

### 5.3 responds_to?

```crystal
class Bird
  def fly
    puts "#{self.class.name} บินได้"
  end
end

class Penguin < Bird
  # Override - Penguin ว่ายน้ำแทน
  def fly
    puts "Penguin บินไม่ได้! แต่ว่ายน้ำเก่ง"
  end

  def swim
    puts "Penguin ว่ายน้ำ"
  end
end

class Eagle < Bird
  def soar
    puts "Eagle บินสูงมาก"
  end
end

birds = [Eagle.new, Penguin.new, Bird.new]

birds.each do |bird|
  bird.fly
  if bird.responds_to?(:soar)
    bird.as(Eagle).soar
  end
  if bird.responds_to?(:swim)
    bird.as(Penguin).swim
  end
end
```

---

## 6. Abstract Classes (Preview)

### 6.1 Class ที่ต้อง Override

```crystal
# Crystal รองรับ abstract class แบบ convention
class AbstractSerializer
  def serialize(data) : String
    raise NotImplementedError.new("#{self.class.name} ต้อง implement serialize")
  end

  def deserialize(str : String)
    raise NotImplementedError.new("#{self.class.name} ต้อง implement deserialize")
  end
end

class JsonSerializer < AbstractSerializer
  def serialize(data) : String
    # Simplified
    data.to_s
  end
end

# Crystal มี abstract keyword แบบ explicit ด้วย (ดู part_037)
```

---

## 7. ตัวอย่างขั้นสูง

### 7.1 Game Character Hierarchy

```crystal
class GameCharacter
  getter name : String
  getter hp : Int32
  getter max_hp : Int32
  getter level : Int32

  def initialize(@name : String, @level : Int32)
    @max_hp = base_hp * @level
    @hp = @max_hp
  end

  def base_hp : Int32
    100  # Default
  end

  def attack_power : Int32
    @level * 10
  end

  def defense : Int32
    @level * 5
  end

  def take_damage(damage : Int32)
    actual_damage = [damage - defense, 1].max
    @hp = [@hp - actual_damage, 0].max
    puts "#{@name} รับความเสียหาย #{actual_damage} (HP: #{@hp}/#{@max_hp})"
  end

  def heal(amount : Int32)
    @hp = [@hp + amount, @max_hp].min
    puts "#{@name} ฟื้นฟู #{amount} HP"
  end

  def alive? : Bool
    @hp > 0
  end

  def attack(target : GameCharacter)
    damage = attack_power
    puts "#{@name} โจมตี #{target.name} ด้วยพลัง #{damage}"
    target.take_damage(damage)
  end

  def status
    puts "#{@name} Lv.#{@level} HP:#{@hp}/#{@max_hp} ATK:#{attack_power} DEF:#{defense}"
  end
end

class Warrior < GameCharacter
  def base_hp : Int32
    150  # มากกว่า default
  end

  def defense : Int32
    super + 10  # เพิ่ม defense
  end

  def shield_bash(target : GameCharacter)
    damage = attack_power / 2
    puts "#{@name} ใช้ Shield Bash ทำ #{damage} ดาเมจ"
    target.take_damage(damage)
  end
end

class Mage < GameCharacter
  getter mana : Int32

  def initialize(name : String, level : Int32)
    super(name, level)
    @mana = level * 50
  end

  def base_hp : Int32
    70  # น้อยกว่า default
  end

  def attack_power : Int32
    super * 2  # พลังโจมตีสูง
  end

  def cast_spell(target : GameCharacter, spell : String)
    mana_cost = 30
    if @mana >= mana_cost
      @mana -= mana_cost
      damage = attack_power + 20
      puts "#{@name} ใช้เวท #{spell} ทำ #{damage} ดาเมจ"
      target.take_damage(damage)
    else
      puts "#{@name} มานะไม่พอ!"
    end
  end
end

warrior = Warrior.new("อัศวิน", 5)
mage = Mage.new("นักเวท", 5)
monster = GameCharacter.new("มังกร", 3)

warrior.status
mage.status
monster.status
puts "---"

warrior.attack(monster)
mage.cast_spell(monster, "Fireball")
monster.attack(warrior)
warrior.shield_bash(monster)
```

### 7.2 E-commerce Product Hierarchy

```crystal
class BaseProduct
  getter id : String
  getter name : String
  getter price : Float64
  getter? in_stock : Bool

  def initialize(@id : String, @name : String, @price : Float64)
    @in_stock = true
  end

  def display_price : String
    "฿#{@price}"
  end

  def to_s : String
    "[#{@id}] #{@name} - #{display_price}"
  end
end

class PhysicalProduct < BaseProduct
  getter weight_kg : Float64
  getter shipping_cost : Float64

  def initialize(id : String, name : String, price : Float64,
                 @weight_kg : Float64)
    super(id, name, price)
    @shipping_cost = calculate_shipping
  end

  def total_price : Float64
    @price + @shipping_cost
  end

  def display_price : String
    "฿#{@price} + ฿#{@shipping_cost} ค่าส่ง"
  end

  private def calculate_shipping : Float64
    @weight_kg * 30.0  # 30 บาท/kg
  end
end

class DigitalProduct < BaseProduct
  getter download_url : String
  getter file_size_mb : Float64

  def initialize(id : String, name : String, price : Float64,
                 @download_url : String, @file_size_mb : Float64)
    super(id, name, price)
  end

  def total_price : Float64
    @price  # ไม่มีค่าส่ง
  end

  def display_price : String
    "฿#{@price} (ดาวน์โหลด)"
  end
end

products = [
  PhysicalProduct.new("P001", "หนังสือ Crystal", 350.0, 0.5),
  PhysicalProduct.new("P002", "แล็ปท็อป", 25000.0, 2.0),
  DigitalProduct.new("D001", "Crystal Course PDF", 199.0,
    "https://example.com/download/d001", 50.0),
]

puts "=== รายการสินค้า ==="
products.each { |p| puts p }

puts "\n=== สินค้ากายภาพ ==="
products.select { |p| p.is_a?(PhysicalProduct) }.each do |p|
  if phys = p.as?(PhysicalProduct)
    puts "#{phys.name}: ราคาทั้งหมด ฿#{phys.total_price}"
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Vehicle Hierarchy

```crystal
class Vehicle
  getter make : String
  getter model : String

  def initialize(@make : String, @model : String)
  end

  def max_speed : Int32
    120  # default km/h
  end

  def fuel_efficiency : Float64
    10.0  # default km/L
  end

  def to_s : String
    "#{@make} #{@model}"
  end
end

class Car < Vehicle
  def max_speed : Int32
    200
  end
  def fuel_efficiency : Float64
    15.0
  end
end

class Truck < Vehicle
  getter payload_tons : Float64

  def initialize(make : String, model : String, @payload_tons : Float64)
    super(make, model)
  end

  def max_speed : Int32
    100
  end
  def fuel_efficiency : Float64
    5.0
  end
end

class Motorcycle < Vehicle
  def max_speed : Int32
    250
  end
  def fuel_efficiency : Float64
    25.0
  end
end

vehicles = [
  Car.new("Toyota", "Camry"),
  Truck.new("Isuzu", "D-Max", 1.5),
  Motorcycle.new("Honda", "CBR"),
]

puts "=== รายการยานพาหนะ ==="
vehicles.each do |v|
  puts "#{v}: ความเร็วสูงสุด #{v.max_speed} km/h, " \
       "ประหยัดเชื้อเพลิง #{v.fuel_efficiency} km/L"
end
```

---

## สรุป

Inheritance ใน Crystal:

| Concept | Syntax | คำอธิบาย |
|---------|--------|---------|
| การสืบทอด | `class Child < Parent` | Child ได้รับ methods ของ Parent |
| Override | `def method_name` | นิยาม method ใหม่ทับของ Parent |
| Call parent | `super` หรือ `super(args)` | เรียก method ของ Parent |
| Type check | `obj.is_a?(Type)` | ตรวจสอบประเภท |
| Safe cast | `obj.as?(Type)` | แปลงประเภทอย่างปลอดภัย |
| Method check | `obj.responds_to?(:method)` | ตรวจสอบว่ามี method |

**หลักการสำคัญ:**
- Crystal รองรับ single inheritance เท่านั้น (class หนึ่งมี parent ได้หนึ่งเดียว)
- ใช้ modules สำหรับ multiple inheritance-like behavior
- `super` เรียก method เดียวกันของ parent class
- `is_a?` ใช้กับทั้ง class และ module
