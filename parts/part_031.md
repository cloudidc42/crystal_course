# Part 31: Classes พื้นฐาน

## บทนำ

Class เป็นแม่แบบ (blueprint) สำหรับสร้าง objects Class กำหนดโครงสร้างข้อมูล (attributes) และพฤติกรรม (methods) ของ objects ที่จะสร้างจาก class นั้น Crystal เป็นภาษาที่มุ่งเน้น Object-Oriented Programming (OOP) อย่างเต็มที่

---

## 1. การนิยาม Class พื้นฐาน

### 1.1 syntax class/end

```crystal
# นิยาม class อย่างง่าย
class Dog
  # เนื้อหาของ class อยู่ระหว่าง class และ end
end

# สร้าง instance ของ class
my_dog = Dog.new
puts my_dog.class  # => Dog
```

### 1.2 Class Naming Convention

```crystal
# ชื่อ class ต้องขึ้นต้นด้วยตัวใหญ่ (PascalCase)
class Person
end

class BankAccount
end

class HttpRequest
end

class ApiController
end

# ไม่ถูกต้อง (จะเกิด syntax error)
# class person  # ต้องตัวใหญ่
# class bank_account  # ต้องเป็น PascalCase
```

### 1.3 Class Body

```crystal
class Cat
  # Constants - ค่าคงที่ของ class
  LEGS = 4
  SPECIES = "Felis catus"

  # Instance methods - เมธอดของ instance
  def speak
    puts "เมี้ยว!"
  end

  def describe
    puts "ฉันเป็นแมว มี #{LEGS} ขา"
  end
end

kitty = Cat.new
kitty.speak     # => เมี้ยว!
kitty.describe  # => ฉันเป็นแมว มี 4 ขา

puts Cat::LEGS     # => 4
puts Cat::SPECIES  # => Felis catus
```

---

## 2. สร้าง Instance ด้วย new

### 2.1 การสร้าง Instance

```crystal
class Book
  def initialize
    puts "สร้างหนังสือใหม่!"
  end
end

# สร้าง instance ด้วย .new
book1 = Book.new  # => สร้างหนังสือใหม่!
book2 = Book.new  # => สร้างหนังสือใหม่!

# แต่ละ instance เป็น object แยกกัน
puts book1.equal?(book2)  # => false
puts book1.class          # => Book
```

### 2.2 Instance ที่แยกกัน

```crystal
class Counter
  def initialize
    @count = 0
  end

  def increment
    @count += 1
  end

  def value
    @count
  end
end

c1 = Counter.new
c2 = Counter.new

c1.increment
c1.increment
c1.increment

c2.increment

puts c1.value  # => 3
puts c2.value  # => 1 (แยกจากกัน ไม่มีผลกระทบ)
```

---

## 3. Methods - การกำหนดเมธอด

### 3.1 Instance Methods

```crystal
class Calculator
  def add(a : Int32, b : Int32) : Int32
    a + b
  end

  def subtract(a : Int32, b : Int32) : Int32
    a - b
  end

  def multiply(a : Int32, b : Int32) : Int32
    a * b
  end

  def divide(a : Int32, b : Int32) : Float64
    a.to_f / b
  end
end

calc = Calculator.new
puts calc.add(10, 5)       # => 15
puts calc.subtract(10, 5)  # => 5
puts calc.multiply(10, 5)  # => 50
puts calc.divide(10, 3)    # => 3.3333...
```

### 3.2 เมธอดที่คืนค่า

```crystal
class Circle
  PI = 3.14159265

  def initialize(@radius : Float64)
  end

  def area : Float64
    PI * @radius ** 2
  end

  def perimeter : Float64
    2 * PI * @radius
  end

  def diameter : Float64
    @radius * 2
  end
end

circle = Circle.new(5.0)
puts "รัศมี: #{circle.diameter / 2}"
puts "เส้นผ่านศูนย์กลาง: #{circle.diameter}"
puts "พื้นที่: #{circle.area.round(2)}"
puts "เส้นรอบวง: #{circle.perimeter.round(2)}"
```

### 3.3 Predicate Methods

```crystal
class Temperature
  def initialize(@celsius : Float64)
  end

  def freezing?
    @celsius <= 0
  end

  def boiling?
    @celsius >= 100
  end

  def comfortable?
    @celsius >= 20 && @celsius <= 30
  end

  def to_fahrenheit : Float64
    @celsius * 9.0 / 5.0 + 32
  end

  def to_s : String
    "#{@celsius}°C (#{to_fahrenheit.round(1)}°F)"
  end
end

temps = [
  Temperature.new(-10.0),
  Temperature.new(25.0),
  Temperature.new(100.0),
]

temps.each do |t|
  status = if t.freezing?
    "แช่แข็ง"
  elsif t.boiling?
    "เดือด"
  elsif t.comfortable?
    "สบาย"
  else
    "ปกติ"
  end
  puts "#{t}: #{status}"
end
```

---

## 4. Class Body และโครงสร้าง

### 4.1 โครงสร้าง Class ที่ดี

```crystal
class BankAccount
  # 1. Constants
  MIN_BALANCE = 100.0
  INTEREST_RATE = 0.05

  # 2. Initialize
  def initialize(owner : String, initial_balance : Float64 = 0.0)
    @owner = owner
    @balance = initial_balance
    @transactions = [] of Float64
  end

  # 3. Core methods
  def deposit(amount : Float64)
    validate_positive(amount)
    @balance += amount
    @transactions << amount
    puts "ฝากเงิน ฿#{amount} สำเร็จ"
  end

  def withdraw(amount : Float64)
    validate_positive(amount)
    raise "ยอดเงินไม่เพียงพอ" if @balance - amount < MIN_BALANCE
    @balance -= amount
    @transactions << -amount
    puts "ถอนเงิน ฿#{amount} สำเร็จ"
  end

  # 4. Query methods
  def balance : Float64
    @balance
  end

  def transaction_count : Int32
    @transactions.size
  end

  # 5. Display methods
  def statement
    puts "=== บัญชีของ #{@owner} ==="
    puts "ยอดคงเหลือ: ฿#{@balance}"
    puts "จำนวนรายการ: #{@transactions.size}"
  end

  # Private methods
  private def validate_positive(amount : Float64)
    raise "จำนวนเงินต้องมากกว่า 0" unless amount > 0
  end
end

account = BankAccount.new("สมชาย", 5000.0)
account.deposit(1000.0)
account.withdraw(500.0)
account.statement
```

### 4.2 Reopening Classes

```crystal
# Crystal อนุญาตให้เปิด class เพื่อเพิ่มเมธอดได้
class String
  def palindrome?
    self == self.reverse
  end

  def word_count : Int32
    self.split.size
  end
end

puts "racecar".palindrome?     # => true
puts "hello".palindrome?       # => false
puts "สวัสดีโลก crystal".word_count  # => 3
```

---

## 5. Method Calls - การเรียกเมธอด

### 5.1 การเรียกเมธอดพื้นฐาน

```crystal
class Robot
  def initialize(@name : String)
    @energy = 100
  end

  def greet
    "สวัสดี ฉันชื่อ #{@name}"
  end

  def work(task : String)
    puts "#{@name} กำลัง #{task}"
    @energy -= 10
  end

  def charge
    @energy = 100
    puts "#{@name} ชาร์จพลังงานเต็มแล้ว"
  end

  def status
    "#{@name}: พลังงาน #{@energy}%"
  end
end

r2d2 = Robot.new("R2D2")
puts r2d2.greet
r2d2.work("ทำความสะอาด")
r2d2.work("ส่งของ")
puts r2d2.status
r2d2.charge
puts r2d2.status
```

### 5.2 Method Chaining

```crystal
class StringBuilder
  def initialize
    @parts = [] of String
  end

  def add(text : String) : self
    @parts << text
    self  # ส่งคืน self เพื่อ chaining
  end

  def add_line(text : String) : self
    @parts << text + "\n"
    self
  end

  def build : String
    @parts.join
  end
end

result = StringBuilder.new
  .add("สวัสดี ")
  .add("โลก")
  .add_line("!")
  .add("Crystal ")
  .add("is awesome!")
  .build

puts result
# => สวัสดี โลก!
# => Crystal is awesome!
```

### 5.3 puts กับ instance.method

```crystal
class Person
  def initialize(@name : String, @age : Int32)
  end

  def to_s : String
    "#{@name} (#{@age} ปี)"
  end

  def greeting : String
    "สวัสดี ฉันชื่อ #{@name}"
  end
end

alice = Person.new("อลิซ", 25)
bob = Person.new("บ็อบ", 30)

# วิธีต่างๆ ในการแสดงผล
puts alice             # ใช้ to_s อัตโนมัติ
puts alice.to_s        # เรียก to_s โดยตรง
puts alice.greeting    # เรียกเมธอดเฉพาะ
puts "บุคคล: #{alice}" # string interpolation ใช้ to_s

people = [alice, bob]
people.each { |p| puts p }
```

---

## 6. Class พื้นฐานที่ใช้งานจริง

### 6.1 Student Class

```crystal
class Student
  def initialize(@name : String, @student_id : String)
    @grades = {} of String => Float64
  end

  def add_grade(subject : String, score : Float64)
    @grades[subject] = score
  end

  def gpa : Float64
    return 0.0 if @grades.empty?
    @grades.values.sum / @grades.size
  end

  def passed?(subject : String) : Bool
    (@grades[subject]? || 0.0) >= 50.0
  end

  def report
    puts "=== รายงานผลการเรียน ==="
    puts "ชื่อ: #{@name}"
    puts "รหัสนักศึกษา: #{@student_id}"
    puts "\nวิชาที่เรียน:"
    @grades.each do |subject, score|
      status = score >= 50 ? "ผ่าน" : "ไม่ผ่าน"
      puts "  #{subject}: #{score} (#{status})"
    end
    puts "\nเกรดเฉลี่ย: #{gpa.round(2)}"
  end
end

student = Student.new("สมชาย ใจดี", "STD001")
student.add_grade("คณิตศาสตร์", 85.0)
student.add_grade("ภาษาไทย", 72.0)
student.add_grade("วิทยาศาสตร์", 90.0)
student.add_grade("ภาษาอังกฤษ", 45.0)

student.report
puts "\nผ่านภาษาอังกฤษ: #{student.passed?("ภาษาอังกฤษ")}"
```

### 6.2 Task Manager

```crystal
class Task
  enum Status
    Todo
    InProgress
    Done
  end

  getter title : String
  getter status : Status
  getter created_at : Time

  def initialize(@title : String)
    @status = Status::Todo
    @created_at = Time.local
  end

  def start
    @status = Status::InProgress
    puts "'#{@title}' เริ่มทำงานแล้ว"
  end

  def complete
    @status = Status::Done
    puts "'#{@title}' เสร็จแล้ว"
  end

  def done? : Bool
    @status == Status::Done
  end

  def to_s : String
    status_icon = case @status
    when Status::Todo then "○"
    when Status::InProgress then "◉"
    when Status::Done then "●"
    else "?"
    end
    "#{status_icon} #{@title}"
  end
end

class TaskList
  def initialize(@name : String)
    @tasks = [] of Task
  end

  def add(title : String) : Task
    task = Task.new(title)
    @tasks << task
    task
  end

  def completed_count : Int32
    @tasks.count(&.done?)
  end

  def show
    puts "=== #{@name} ==="
    @tasks.each { |t| puts "  #{t}" }
    puts "เสร็จแล้ว: #{completed_count}/#{@tasks.size}"
  end
end

list = TaskList.new("งานวันนี้")
t1 = list.add("อ่านอีเมล")
t2 = list.add("ประชุมทีม")
t3 = list.add("เขียน report")

t1.complete
t2.start

list.show
```

### 6.3 Product Catalog

```crystal
class Product
  getter name : String
  getter price : Float64
  getter category : String

  def initialize(@name : String, @price : Float64, @category : String)
    @stock = 0
  end

  def add_stock(quantity : Int32)
    @stock += quantity
    puts "เพิ่มสต็อก #{@name}: #{@stock} ชิ้น"
  end

  def sell(quantity : Int32) : Bool
    if @stock >= quantity
      @stock -= quantity
      puts "ขาย #{@name} จำนวน #{quantity} ชิ้น"
      true
    else
      puts "สต็อกไม่เพียงพอ (มีแค่ #{@stock} ชิ้น)"
      false
    end
  end

  def available? : Bool
    @stock > 0
  end

  def total_value : Float64
    @price * @stock
  end

  def to_s : String
    "#{@name} - ฿#{@price} (สต็อก: #{@stock})"
  end
end

# สร้างสินค้า
laptop = Product.new("แล็ปท็อป", 25000.0, "อิเล็กทรอนิกส์")
phone = Product.new("สมาร์ทโฟน", 15000.0, "อิเล็กทรอนิกส์")
book = Product.new("หนังสือ Crystal", 350.0, "หนังสือ")

laptop.add_stock(10)
phone.add_stock(5)
book.add_stock(100)

puts laptop
puts phone
puts book

laptop.sell(3)
book.sell(50)
phone.sell(10)  # ไม่พอ
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Library Book

```crystal
class LibraryBook
  getter title : String
  getter author : String
  getter isbn : String

  def initialize(@title : String, @author : String, @isbn : String)
    @checked_out = false
    @borrower = ""
  end

  def checkout(borrower : String)
    if @checked_out
      puts "หนังสือ '#{@title}' ถูกยืมออกไปแล้ว"
    else
      @checked_out = true
      @borrower = borrower
      puts "#{borrower} ยืมหนังสือ '#{@title}' สำเร็จ"
    end
  end

  def return_book
    if @checked_out
      puts "#{@borrower} คืนหนังสือ '#{@title}' แล้ว"
      @checked_out = false
      @borrower = ""
    else
      puts "หนังสือ '#{@title}' ไม่ได้ถูกยืมออก"
    end
  end

  def available? : Bool
    !@checked_out
  end

  def to_s : String
    status = @checked_out ? "ถูกยืม (โดย #{@borrower})" : "พร้อมให้ยืม"
    "'#{@title}' โดย #{@author} - #{status}"
  end
end

books = [
  LibraryBook.new("Crystal Programming", "Manas", "978-001"),
  LibraryBook.new("Clean Code", "Robert Martin", "978-002"),
  LibraryBook.new("Design Patterns", "Gang of Four", "978-003"),
]

books[0].checkout("สมชาย")
books[1].checkout("สมหญิง")
books[0].checkout("สมศักดิ์")  # ไม่สำเร็จ

puts "\n=== รายการหนังสือ ==="
books.each { |b| puts b }

books[0].return_book
puts "\n=== หลังคืนหนังสือ ==="
books.each { |b| puts b }
```

### แบบฝึกหัดที่ 2: Simple Game Character

```crystal
class GameCharacter
  getter name : String
  getter level : Int32
  getter hp : Int32
  getter max_hp : Int32

  def initialize(@name : String, @level : Int32 = 1)
    @max_hp = @level * 100
    @hp = @max_hp
    @attack = @level * 10
    @alive = true
  end

  def attack_target(target : GameCharacter)
    damage = @attack
    puts "#{@name} โจมตี #{target.name} สร้างความเสียหาย #{damage}"
    target.take_damage(damage)
  end

  def take_damage(amount : Int32)
    @hp = [@hp - amount, 0].max
    if @hp <= 0 && @alive
      @alive = false
      puts "#{@name} ถูกกำจัดแล้ว!"
    end
  end

  def heal(amount : Int32)
    @hp = [@hp + amount, @max_hp].min
    puts "#{@name} ฟื้นฟู #{amount} HP (ปัจจุบัน: #{@hp}/#{@max_hp})"
  end

  def alive? : Bool
    @alive
  end

  def status
    puts "#{@name} - Level #{@level} - HP: #{@hp}/#{@max_hp}"
  end
end

hero = GameCharacter.new("นักรบ", 5)
monster = GameCharacter.new("มังกร", 3)

hero.status
monster.status

hero.attack_target(monster)
monster.attack_target(hero)
hero.heal(50)
hero.attack_target(monster)
hero.attack_target(monster)
```

---

## สรุป

Classes พื้นฐานใน Crystal:

| องค์ประกอบ | Syntax | คำอธิบาย |
|-----------|--------|---------|
| นิยาม class | `class Name ... end` | สร้าง class ใหม่ |
| สร้าง instance | `ClassName.new` | สร้าง object จาก class |
| เมธอด | `def method_name ... end` | กำหนดพฤติกรรม |
| Constructor | `def initialize ... end` | เรียกเมื่อ .new |
| Constants | `CONSTANT = value` | ค่าคงที่ใน class |
| Method call | `object.method` | เรียกเมธอด |

**หลักการสำคัญ:**
- ชื่อ class ต้องเป็น PascalCase
- `def initialize` เป็น constructor ที่เรียกเมื่อ `new`
- เมธอดใน instance เข้าถึง instance variables ด้วย `@`
- `self` หมายถึง instance ปัจจุบัน
- Crystal อนุญาตให้ "reopen" class เพื่อเพิ่มเมธอดได้
