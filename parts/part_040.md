# Part 40: Mixins

## บทนำ

Mixin เป็น pattern ที่ใช้ Modules เพื่อเพิ่ม functionality ให้กับ classes โดยไม่ต้องใช้ inheritance Mixin ช่วยแก้ปัญหา "multiple inheritance" ในภาษา OOP และเป็นวิธีที่ดีในการแบ่ง behavior ที่ใช้ร่วมกัน

---

## 1. Mixin Pattern พื้นฐาน

### 1.1 ทำไมต้องใช้ Mixin

```crystal
# ปัญหา: ถ้าไม่ใช้ Mixin ต้องเขียนโค้ดซ้ำ
class Dog
  def speak; "โฮ่ง!"; end
  def to_s; "Dog"; end

  # โค้ดซ้ำ
  def log(msg); puts "[#{self.class.name}] #{msg}"; end
  def save; puts "Saving #{self.class.name}..."; end
end

class Cat
  def speak; "เมี้ยว!"; end
  def to_s; "Cat"; end

  # โค้ดซ้ำ
  def log(msg); puts "[#{self.class.name}] #{msg}"; end
  def save; puts "Saving #{self.class.name}..."; end
end

# แก้ปัญหาด้วย Mixin
module Loggable
  def log(message : String)
    puts "[#{self.class.name}] #{message}"
  end
end

module Persistable
  def save
    puts "Saving #{self.class.name}..."
  end

  def delete
    puts "Deleting #{self.class.name}..."
  end
end

class DogV2
  include Loggable
  include Persistable

  def speak; "โฮ่ง!"; end
end

class CatV2
  include Loggable
  include Persistable

  def speak; "เมี้ยว!"; end
end

dog = DogV2.new
dog.log("เริ่มต้น")
dog.save
```

### 1.2 Behavior Composition

```crystal
# ประกอบ behavior จาก module หลายตัว
module Flyable
  def fly
    puts "#{self.class.name} บิน"
  end

  def altitude : Int32
    1000  # default
  end
end

module Swimmable
  def swim
    puts "#{self.class.name} ว่ายน้ำ"
  end

  def depth : Int32
    10  # default
  end
end

module Walkable
  def walk
    puts "#{self.class.name} เดิน"
  end

  def speed : Int32
    5  # default km/h
  end
end

# Duck สามารถทำได้ทั้งสาม
class Duck
  include Flyable
  include Swimmable
  include Walkable

  def what_can_i_do
    puts "ฉันสามารถ:"
    puts "  - บินที่ระดับ #{altitude} เมตร"
    puts "  - ว่ายน้ำลึก #{depth} เมตร"
    puts "  - เดินด้วยความเร็ว #{speed} km/h"
  end
end

# Human ทำได้แค่สอง
class Human
  include Swimmable
  include Walkable

  def speed : Int32
    6  # คนเดินเร็วกว่า Duck
  end
end

duck = Duck.new
duck.what_can_i_do
duck.fly
duck.swim

human = Human.new
human.walk
human.swim
```

---

## 2. Comparable Mixin

### 2.1 Implement Comparable

```crystal
class Product
  include Comparable(Product)

  getter name : String
  getter price : Float64
  getter rating : Float64

  def initialize(@name : String, @price : Float64, @rating : Float64)
  end

  # ต้อง implement <=> เพื่อใช้ Comparable
  def <=>(other : Product) : Int32
    @price <=> other.price  # เปรียบตามราคา
  end

  def to_s : String
    "#{@name} ฿#{@price} (★#{@rating})"
  end
end

products = [
  Product.new("แล็ปท็อป", 25000.0, 4.5),
  Product.new("หูฟัง", 1500.0, 4.0),
  Product.new("เมาส์", 500.0, 3.8),
  Product.new("คีย์บอร์ด", 2000.0, 4.2),
]

# ได้ methods ทั้งหมดจาก Comparable
puts products.sort.map(&.to_s)
puts products.min.try(&.to_s)  # => เมาส์ ฿500.0
puts products.max.try(&.to_s)  # => แล็ปท็อป ฿25000.0

laptop = products[0]
keyboard = products[3]
puts laptop > keyboard    # => true
puts keyboard < laptop    # => true
```

### 2.2 Custom Comparison

```crystal
class Student
  include Comparable(Student)

  getter name : String
  getter gpa : Float64
  getter graduation_year : Int32

  def initialize(@name : String, @gpa : Float64, @graduation_year : Int32)
  end

  # เปรียบตาม GPA ก่อน แล้วตาม graduation year
  def <=>(other : Student) : Int32
    gpa_cmp = @gpa <=> other.gpa
    return gpa_cmp unless gpa_cmp == 0
    @graduation_year <=> other.graduation_year
  end

  def to_s : String
    "#{@name} (GPA: #{@gpa}, ปี #{@graduation_year})"
  end
end

students = [
  Student.new("สมชาย", 3.5, 2024),
  Student.new("สมหญิง", 3.8, 2023),
  Student.new("สมศักดิ์", 3.5, 2023),
  Student.new("สมรัก", 3.9, 2024),
]

puts "เรียงตาม GPA และปีจบ:"
students.sort.each { |s| puts "  #{s}" }

puts "\nนักศึกษาที่ดีที่สุด: #{students.max}"
```

---

## 3. Enumerable Mixin

### 3.1 Custom Enumerable

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

  def find(&block : T -> Bool) : T?
    each { |item| return item if block.call(item) }
    nil
  end

  def count : Int32
    n = 0
    each { n += 1 }
    n
  end

  def count(&block : T -> Bool) : Int32
    n = 0
    each { |item| n += 1 if block.call(item) }
    n
  end

  def any?(&block : T -> Bool) : Bool
    each { |item| return true if block.call(item) }
    false
  end

  def all?(&block : T -> Bool) : Bool
    each { |item| return false unless block.call(item) }
    true
  end

  def none?(&block : T -> Bool) : Bool
    !any? { |item| block.call(item) }
  end

  def flat_map(&block : T -> Array(U)) : Array(U) forall U
    result = [] of U
    each { |item| result.concat(block.call(item)) }
    result
  end

  def each_with_index(&block : T, Int32 ->)
    i = 0
    each { |item| block.call(item, i); i += 1 }
  end

  def reduce(initial : U, &block : U, T -> U) : U forall U
    acc = initial
    each { |item| acc = block.call(acc, item) }
    acc
  end

  def to_a : Array(T)
    result = [] of T
    each { |item| result << item }
    result
  end
end

class Tree(T)
  include Enumerable(T)

  getter value : T
  getter children : Array(Tree(T))

  def initialize(@value : T)
    @children = [] of Tree(T)
  end

  def add_child(child : Tree(T))
    @children << child
    self
  end

  def each(&block : T ->)
    block.call(@value)
    @children.each { |child| child.each { |v| block.call(v) } }
  end
end

# สร้าง tree ของตัวเลข
root = Tree(Int32).new(1)
child1 = Tree(Int32).new(2)
child2 = Tree(Int32).new(3)
child1.add_child(Tree(Int32).new(4)).add_child(Tree(Int32).new(5))
child2.add_child(Tree(Int32).new(6))
root.add_child(child1).add_child(child2)

puts root.to_a.inspect            # => [1, 2, 4, 5, 3, 6]
puts root.map { |v| v * 2 }.inspect  # => [2, 4, 8, 10, 6, 12]
puts root.select { |v| v.even? }.inspect  # => [2, 4, 6]
puts root.count                   # => 6
puts root.sum                     # => 21
```

---

## 4. Custom Mixins

### 4.1 Timestamps Mixin

```crystal
module Timestamps
  def self.included(base : Class)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def created_after(time : Time) : Array(self)
      all.select { |r| r.created_at > time }
    end

    def created_today : Array(self)
      today = Time.local.at_beginning_of_day
      tomorrow = today + 1.day
      all.select { |r| r.created_at >= today && r.created_at < tomorrow }
    end
  end

  getter created_at : Time
  getter updated_at : Time

  def initialize_timestamps
    @created_at = Time.local
    @updated_at = Time.local
  end

  def touch
    @updated_at = Time.local
  end

  def age : Time::Span
    Time.local - @created_at
  end

  def recently_created? : Bool
    age.total_minutes < 5
  end
end

class Post
  include Timestamps

  @@all = [] of Post

  def self.all : Array(Post)
    @@all
  end

  getter title : String
  getter content : String

  def initialize(@title : String, @content : String)
    initialize_timestamps
    @@all << self
  end

  def update(content : String)
    @content = content
    touch
  end
end

p1 = Post.new("First Post", "เนื้อหาแรก")
sleep(0.001)
p2 = Post.new("Second Post", "เนื้อหาสอง")

puts p1.recently_created?  # => true
puts p1.age.total_seconds.round(3)

p1.update("เนื้อหาที่แก้ไขแล้ว")
puts p1.updated_at >= p1.created_at  # => true
```

### 4.2 Soft Delete Mixin

```crystal
module SoftDeletable
  getter? deleted : Bool
  getter deleted_at : Time?

  def initialize_soft_delete
    @deleted = false
    @deleted_at = nil
  end

  def soft_delete
    @deleted = true
    @deleted_at = Time.local
    puts "#{self.class.name} ถูก soft delete แล้ว"
  end

  def restore
    @deleted = false
    @deleted_at = nil
    puts "#{self.class.name} ถูก restore แล้ว"
  end

  def active? : Bool
    !@deleted
  end
end

class User
  include SoftDeletable

  getter name : String
  getter email : String

  @@all = [] of User

  def self.all : Array(User)
    @@all
  end

  def self.active : Array(User)
    @@all.select(&.active?)
  end

  def initialize(@name : String, @email : String)
    initialize_soft_delete
    @@all << self
  end

  def to_s : String
    status = @deleted ? " [DELETED]" : ""
    "#{@name} (#{@email})#{status}"
  end
end

u1 = User.new("สมชาย", "somchai@example.com")
u2 = User.new("สมหญิง", "somsri@example.com")
u3 = User.new("สมศักดิ์", "somsak@example.com")

puts "Users ทั้งหมด: #{User.all.size}"
puts "Users active: #{User.active.size}"

u2.soft_delete

puts "\nหลัง soft delete:"
puts "Users ทั้งหมด: #{User.all.size}"
puts "Users active: #{User.active.size}"
User.all.each { |u| puts "  #{u}" }

u2.restore
puts "\nหลัง restore:"
puts "Users active: #{User.active.size}"
```

---

## 5. Hooks - included และ extended

### 5.1 included Hook

```crystal
module Trackable
  def self.included(base : Class)
    puts "#{base.name} กำลัง include Trackable"
    base.extend(ClassMethods)
    # เพิ่ม class-level state
  end

  module ClassMethods
    def tracked_count : Int32
      @count ||= 0
    end

    def increment_count
      @count = (@count ||= 0) + 1
    end
  end

  def initialize_tracking
    self.class.increment_count
  end
end

class Event
  include Trackable

  getter name : String

  def initialize(@name : String)
    initialize_tracking
  end
end

class Session
  include Trackable

  getter id : String

  def initialize(@id : String)
    initialize_tracking
  end
end

Event.new("Login")
Event.new("Logout")
Event.new("Purchase")
Session.new("sess_001")
Session.new("sess_002")

puts "Events: #{Event.tracked_count}"    # => 3
puts "Sessions: #{Session.tracked_count}"  # => 2
```

### 5.2 extended Hook

```crystal
module ClassBehavior
  def self.extended(base : Class)
    puts "#{base.name} กำลัง extend ClassBehavior"
  end

  def factory_create(*args)
    puts "สร้าง #{self.name} ด้วย factory..."
    new(*args)
  end
end

class Widget
  extend ClassBehavior

  getter name : String

  def initialize(@name : String)
  end
end

w = Widget.factory_create("MyWidget")
puts w.name  # => MyWidget
```

---

## 6. ตัวอย่างขั้นสูง

### 6.1 Full ActiveRecord-like Mixin

```crystal
module ActiveRecord
  def self.included(base : Class)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def table_name : String
      self.name.downcase + "s"
    end

    def find(id : Int32) : self?
      all_records.find { |r| r.id == id }
    end

    def where(&block : self -> Bool) : Array(self)
      all_records.select { |r| block.call(r) }
    end

    def all_records : Array(self)
      @records ||= [] of self
    end

    def count : Int32
      all_records.size
    end
  end

  abstract def id : Int32

  def save
    unless self.class.all_records.any? { |r| r.id == id }
      self.class.all_records << self
    end
    puts "บันทึก #{self.class.name}##{id} ลงใน #{self.class.table_name}"
  end

  def destroy
    self.class.all_records.reject! { |r| r.id == id }
    puts "ลบ #{self.class.name}##{id}"
  end
end

class Book
  include ActiveRecord

  getter id : Int32
  getter title : String
  getter author : String
  getter? published : Bool

  def initialize(@id : Int32, @title : String, @author : String)
    @published = false
  end

  def publish
    @published = true
  end
end

b1 = Book.new(1, "Crystal Programming", "Manas")
b2 = Book.new(2, "Clean Code", "Robert Martin")
b3 = Book.new(3, "Design Patterns", "Gang of Four")

b1.save
b2.save
b3.save

b1.publish

puts "จำนวนหนังสือ: #{Book.count}"  # => 3
puts "Table: #{Book.table_name}"    # => books

found = Book.find(2)
puts found.try(&.title)  # => Clean Code

published = Book.where { |b| b.published? }
puts published.map(&.title).inspect  # => ["Crystal Programming"]

b2.destroy
puts "หลังลบ: #{Book.count}"  # => 2
```

### 6.2 Decorator Mixin

```crystal
module Decoratable
  def self.included(base : Class)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def decorators : Array(Proc(String, String))
      @decorators ||= [] of Proc(String, String)
    end

    def add_decorator(&block : String -> String)
      decorators << block
    end
  end

  def decorate(value : String) : String
    self.class.decorators.reduce(value) { |val, decorator| decorator.call(val) }
  end
end

class TextProcessor
  include Decoratable

  add_decorator { |s| s.strip }
  add_decorator { |s| s.downcase }
  add_decorator { |s| s.gsub(/\s+/, "_") }

  def process(input : String) : String
    decorate(input)
  end
end

proc = TextProcessor.new
puts proc.process("  Hello World  ")  # => hello_world
puts proc.process("  Crystal LANG  ")  # => crystal_lang
```

### 6.3 Policy Mixin

```crystal
module Authorizable
  def self.included(base : Class)
    base.extend(ClassMethods)
  end

  module ClassMethods
    def policies : Array(Proc(self, Symbol, Bool))
      @policies ||= [] of Proc(self, Symbol, Bool)
    end

    def allow(action : Symbol, &policy : self -> Bool)
      policies << ->(resource : self) { policy.call(resource) }
    end
  end

  def authorized?(action : Symbol) : Bool
    self.class.policies.all? { |policy| policy.call(self) }
  end
end

class Document
  include Authorizable

  getter title : String
  getter? published : Bool
  getter? restricted : Bool
  getter owner_id : Int32

  allow(:view) { |doc| !doc.restricted? }
  allow(:edit) { |doc| !doc.published? }
  allow(:delete) { |doc| !doc.published? }

  def initialize(@title : String, @owner_id : Int32)
    @published = false
    @restricted = false
  end

  def publish
    @published = true
  end
end

doc = Document.new("My Document", 1)
puts doc.authorized?(:view)    # => true
puts doc.authorized?(:edit)    # => true

doc.publish
puts doc.authorized?(:edit)   # => false (published)
puts doc.authorized?(:view)   # => true (not restricted)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Full Mixin System

```crystal
# ออกแบบ system ด้วย Mixin หลายตัว
module Identifiable
  getter id : String

  def initialize_id
    @id = "#{self.class.name.downcase}_#{Time.local.to_unix}_#{rand(1000)}"
  end
end

module Auditable
  getter created_by : String
  getter created_at : Time
  getter last_modified_by : String?
  getter last_modified_at : Time?

  def initialize_audit(creator : String)
    @created_by = creator
    @created_at = Time.local
    @last_modified_by = nil
    @last_modified_at = nil
  end

  def record_modification(by : String)
    @last_modified_by = by
    @last_modified_at = Time.local
  end
end

module Versionable
  getter version : Int32

  def initialize_version
    @version = 1
  end

  def increment_version
    @version += 1
  end
end

class Document
  include Identifiable
  include Auditable
  include Versionable

  property title : String
  property content : String

  def initialize(@title : String, @content : String, creator : String)
    initialize_id
    initialize_audit(creator)
    initialize_version
  end

  def update(content : String, by : String)
    @content = content
    record_modification(by)
    increment_version
  end

  def info
    puts "=== #{@title} ==="
    puts "ID: #{@id}"
    puts "Version: #{@version}"
    puts "สร้างโดย: #{@created_by} เมื่อ #{@created_at}"
    if modified_by = @last_modified_by
      puts "แก้ไขล่าสุดโดย: #{modified_by}"
    end
  end
end

doc = Document.new("Crystal Guide", "เนื้อหาเริ่มต้น", "admin")
doc.info

doc.update("เนื้อหาที่ปรับปรุงแล้ว", "editor1")
doc.update("เนื้อหาเวอร์ชันสุดท้าย", "editor2")
doc.info
```

### แบบฝึกหัดที่ 2: Serializable Mixin

```crystal
module JsonSerializable
  def to_json_value(value) : String
    case value
    when String then "\"#{value.gsub("\"", "\\\"")}\""
    when Int32, Float64 then value.to_s
    when Bool then value.to_s
    when Nil then "null"
    when Array then "[#{value.map { |v| to_json_value(v) }.join(", ")}]"
    else "\"#{value}\""
    end
  end
end

class User
  include JsonSerializable

  getter name : String
  getter age : Int32
  getter email : String
  getter? active : Bool

  def initialize(@name : String, @age : Int32, @email : String)
    @active = true
  end

  def to_json : String
    fields = {
      "name" => to_json_value(@name),
      "age" => to_json_value(@age),
      "email" => to_json_value(@email),
      "active" => to_json_value(@active),
    }
    "{#{fields.map { |k, v| "\"#{k}\": #{v}" }.join(", ")}}"
  end
end

user = User.new("สมชาย", 25, "somchai@example.com")
puts user.to_json
# => {"name": "สมชาย", "age": 25, "email": "somchai@example.com", "active": true}
```

---

## สรุป

Mixins เป็นหนึ่งในคุณสมบัติที่ทรงพลังที่สุดของ Crystal:

| แนวคิด | คำอธิบาย |
|--------|---------|
| Mixin pattern | ใช้ Module เพื่อแบ่ง behavior |
| Comparable | เปรียบเทียบ objects |
| Enumerable | iterate ผ่าน collection |
| Custom mixins | สร้าง behavior ของตัวเอง |
| included hook | ทำงานเมื่อ module ถูก include |
| extended hook | ทำงานเมื่อ module ถูก extend |

**Mixin Best Practices:**

1. **Single Responsibility** - Module หนึ่งตัวทำหน้าที่เดียว
2. **Naming Convention** - ตั้งชื่อตาม behavior (Printable, Serializable, Comparable)
3. **Minimal Interface** - กำหนด abstract methods เท่าที่จำเป็น
4. **Hooks สำหรับ auto-extend** - ใช้ `self.included` สำหรับ complex setup
5. **Avoid state ใน Mixin** - ถ้าจำเป็นต้อง state ให้ใช้ initialize method แยก

**เปรียบเทียบ Approaches:**

| Feature | Inheritance | Mixin |
|---------|------------|-------|
| Reuse | ผ่าน superclass | ผ่าน module |
| Multiple | ไม่รองรับ | รองรับหลาย modules |
| State | สืบทอด state | ต้อง explicit |
| "is-a" | ใช่ | ไม่จำเป็น |
| Flexibility | น้อยกว่า | มากกว่า |
