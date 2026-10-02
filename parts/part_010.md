# ตอนที่ 10: Comments และ Documentation

## บทนำ

การเขียน Comment และ Documentation ที่ดีเป็นทักษะสำคัญของโปรแกรมเมอร์มืออาชีพ Crystal มีระบบ documentation ที่ทรงพลัง สามารถ generate API docs ได้อัตโนมัติจาก comment ในโค้ด บทนี้จะสอนวิธีเขียน comment และ documentation ที่มีคุณภาพ

---

## 10.1 Single-line Comments (#)

```crystal
# นี่คือ single-line comment
puts "Hello, Crystal!"  # comment ท้ายบรรทัดก็ได้

# ใช้สำหรับอธิบายโค้ดที่ซับซ้อน
# หรือบอกวัตถุประสงค์ของโค้ด

# TODO: เพิ่ม error handling
# FIXME: แก้ bug นี้ก่อน release
# HACK: วิธีแก้ชั่วคราว
# NOTE: ข้อสังเกตสำคัญ

x = 42  # ค่าเริ่มต้น
```

### Convention ของ Comment

```crystal
# ดี: อธิบาย "ทำไม" ไม่ใช่ "อะไร"
# คูณด้วย 1000 เพื่อแปลง milliseconds เป็น microseconds
delay = response_time * 1000

# ไม่ดี: พูดสิ่งที่โค้ดบอกอยู่แล้ว
# กำหนด x เป็น 5
x = 5

# ดี: อธิบาย edge case
# ต้องตรวจสอบ edge case เมื่อ array ว่างเปล่า
# เพราะ first จะ return nil ใน Crystal 1.x
first_item = items.first?
```

---

## 10.2 Multiline Comments (=begin/=end)

```crystal
=begin
นี่คือ multiline comment
สามารถเขียนได้หลายบรรทัด
ใช้สำหรับ comment ที่ยาว
เช่น การอธิบาย algorithm หรือ business logic
=end

puts "code continues here"

=begin
Algorithm: Quick Sort
Time Complexity: O(n log n) โดยเฉลี่ย, O(n²) ในกรณีเลวร้ายที่สุด
Space Complexity: O(log n)

วิธีการทำงาน:
1. เลือก pivot element
2. แบ่ง array เป็นสองส่วน
3. ส่วนที่น้อยกว่า pivot อยู่ทางซ้าย
4. ส่วนที่มากกว่า pivot อยู่ทางขวา
5. Recursively sort ทั้งสองส่วน
=end

def quick_sort(arr : Array(Int32)) : Array(Int32)
  return arr if arr.size <= 1
  
  pivot = arr[arr.size // 2]
  left = arr.select { |x| x < pivot }
  middle = arr.select { |x| x == pivot }
  right = arr.select { |x| x > pivot }
  
  quick_sort(left) + middle + quick_sort(right)
end
```

### ข้อควรระวัง

```crystal
# =begin ต้องอยู่ต้นบรรทัด (ไม่มี space นำหน้า)
# =end ต้องอยู่ต้นบรรทัดเช่นกัน

=begin
  นี่ถูกต้อง
  สามารถ indent เนื้อหาข้างในได้
=end

# ข้อผิดพลาดที่พบบ่อย:
#   =begin  (ถ้ามี space นำหน้าจะ error!)
# หรือ
#   =end    (ต้องไม่มี space นำหน้า)
```

---

## 10.3 Documentation Comments (##)

Documentation comments ใช้ `##` (สองตัว hash) และ Crystal Docs จะ generate HTML docs จาก comment เหล่านี้

```crystal
## คลาส Person แสดงข้อมูลบุคคล
##
## ตัวอย่างการใช้งาน:
## ```crystal
## person = Person.new("Alice", 30)
## person.greet  # => "สวัสดี ฉันชื่อ Alice อายุ 30 ปี"
## ```
class Person
  ## ชื่อของบุคคล
  getter name : String

  ## อายุของบุคคล (ปี)
  getter age : Int32

  ## สร้าง Person ใหม่
  ##
  ## - *name* : ชื่อของบุคคล
  ## - *age* : อายุในหน่วยปี (ต้องเป็นบวก)
  def initialize(@name : String, @age : Int32)
    raise ArgumentError.new("อายุต้องเป็นบวก") if @age < 0
  end

  ## สร้างข้อความทักทาย
  ##
  ## คืนค่า String ที่แนะนำตัวเอง
  def greet : String
    "สวัสดี ฉันชื่อ #{@name} อายุ #{@age} ปี"
  end
end
```

---

## 10.4 Crystal Docs Command

```bash
# Generate documentation
crystal docs

# ระบุ output directory
crystal docs --output=./docs

# ระบุ source files
crystal docs src/myapp.cr

# เปิดใน browser หลัง generate
crystal docs --open

# เพิ่ม project name
crystal docs --project-name="My Crystal App"

# เพิ่ม project version
crystal docs --project-version="1.0.0"
```

### โครงสร้าง docs ที่ generate

```
docs/
├── index.html          # หน้าแรก
├── search-index.js     # ข้อมูลสำหรับ search
├── css/
│   └── style.css
└── MyApp/
    ├── index.html      # index ของ module
    ├── Person.html     # docs สำหรับ class Person
    └── Calculator.html # docs สำหรับ class Calculator
```

---

## 10.5 YARD-style Documentation

Crystal ใช้ syntax ที่คล้าย YARD (Yet Another Ruby Documenter)

### พารามิเตอร์และ Return type

```crystal
## คำนวณพื้นที่สี่เหลี่ยมผืนผ้า
##
## คืนพื้นที่ในหน่วยตารางเมตร
##
## - *width* : ความกว้าง (เมตร, ต้องเป็นบวก)
## - *height* : ความสูง (เมตร, ต้องเป็นบวก)
##
## ตัวอย่าง:
## ```
## area = calculate_area(5.0, 3.0)  # => 15.0
## ```
##
## Raises `ArgumentError` ถ้า width หรือ height เป็น 0 หรือลบ
def calculate_area(width : Float64, height : Float64) : Float64
  raise ArgumentError.new("ขนาดต้องเป็นบวก") if width <= 0 || height <= 0
  width * height
end
```

### การ document exceptions

```crystal
## บันทึกข้อมูลลงฐานข้อมูล
##
## - *data* : Hash ของข้อมูลที่จะบันทึก
## - *table* : ชื่อตาราง
##
## Raises `DatabaseError` ถ้าการเชื่อมต่อล้มเหลว
## Raises `ValidationError` ถ้าข้อมูลไม่ผ่านการตรวจสอบ
## Raises `DuplicateError` ถ้า record นี้มีอยู่แล้ว
def save_to_db(data : Hash(String, String), table : String) : Bool
  # implementation
  true
end
```

---

## 10.6 Documenting Classes

```crystal
## `BankAccount` จัดการบัญชีธนาคาร
##
## ให้บริการ deposit, withdraw, และ transfer เงิน
## พร้อมตรวจสอบ balance และ transaction history
##
## ตัวอย่างการใช้งาน:
## ```crystal
## account = BankAccount.new("ACC001", 1000.0)
## account.deposit(500.0)
## account.withdraw(200.0)
## puts account.balance  # => 1300.0
## ```
##
## หมายเหตุ: ไม่รองรับ concurrent access
class BankAccount
  ## หมายเลขบัญชี (ไม่สามารถเปลี่ยนได้)
  getter account_number : String

  ## ยอดเงินปัจจุบัน
  getter balance : Float64

  ## ประวัติ transactions
  getter transactions : Array(Transaction)

  ## สร้างบัญชีใหม่
  ##
  ## - *account_number* : หมายเลขบัญชีไม่ซ้ำ
  ## - *initial_balance* : ยอดเงินเริ่มต้น (ค่าเริ่มต้น: 0.0)
  def initialize(@account_number : String, @initial_balance : Float64 = 0.0)
    @balance = @initial_balance
    @transactions = [] of Transaction
  end

  ## ฝากเงินเข้าบัญชี
  ##
  ## - *amount* : จำนวนเงินที่ฝาก (ต้องเป็นบวก)
  ##
  ## Raises `ArgumentError` ถ้า amount <= 0
  def deposit(amount : Float64) : self
    raise ArgumentError.new("จำนวนเงินต้องเป็นบวก") if amount <= 0
    @balance += amount
    @transactions << Transaction.new(:deposit, amount)
    self  # สำหรับ method chaining
  end

  ## ถอนเงินออกจากบัญชี
  ##
  ## - *amount* : จำนวนเงินที่ถอน (ต้องเป็นบวก)
  ##
  ## Raises `ArgumentError` ถ้า amount <= 0
  ## Raises `InsufficientFundsError` ถ้ายอดเงินไม่พอ
  def withdraw(amount : Float64) : self
    raise ArgumentError.new("จำนวนเงินต้องเป็นบวก") if amount <= 0
    raise InsufficientFundsError.new("ยอดเงินไม่เพียงพอ") if amount > @balance
    @balance -= amount
    @transactions << Transaction.new(:withdraw, amount)
    self
  end

  ## โอนเงินไปยังบัญชีอื่น
  ##
  ## - *target* : บัญชีปลายทาง
  ## - *amount* : จำนวนเงินที่โอน
  ##
  ## Raises `ArgumentError` ถ้าโอนไปยังบัญชีตัวเอง
  def transfer(target : BankAccount, amount : Float64) : self
    raise ArgumentError.new("ไม่สามารถโอนไปยังบัญชีตัวเองได้") if target == self
    withdraw(amount)
    target.deposit(amount)
    self
  end
end

## ข้อมูล Transaction
record Transaction, type : Symbol, amount : Float64

## ข้อผิดพลาดเมื่อยอดเงินไม่พอ
class InsufficientFundsError < Exception; end
```

---

## 10.7 Documenting Modules

```crystal
## Module `MathUtils` รวบรวม utilities สำหรับการคำนวณทางคณิตศาสตร์
##
## ประกอบด้วย:
## - ฟังก์ชันทางสถิติ
## - ฟังก์ชัน geometric
## - ฟังก์ชัน number theory
##
## ตัวอย่าง:
## ```crystal
## include MathUtils
## puts factorial(5)    # => 120
## puts fibonacci(10)   # => 55
## puts is_prime?(17)   # => true
## ```
module MathUtils
  ## คำนวณ factorial ของตัวเลข
  ##
  ## Factorial คือ n! = n × (n-1) × ... × 2 × 1
  ##
  ## - *n* : ตัวเลขที่ต้องการหา factorial (ต้องไม่ติดลบ)
  ##
  ## ตัวอย่าง:
  ## ```
  ## factorial(0) # => 1
  ## factorial(5) # => 120
  ## ```
  def factorial(n : Int32) : Int64
    raise ArgumentError.new("n ต้องไม่ติดลบ") if n < 0
    return 1_i64 if n <= 1
    (2..n).reduce(1_i64) { |acc, i| acc * i }
  end

  ## หาตัวเลข Fibonacci ลำดับที่ n
  ##
  ## ใช้ dynamic programming เพื่อประสิทธิภาพ O(n)
  ##
  ## - *n* : ลำดับที่ต้องการ (เริ่มจาก 0)
  ##
  ## ตัวอย่าง:
  ## ```
  ## fibonacci(0)  # => 0
  ## fibonacci(1)  # => 1
  ## fibonacci(10) # => 55
  ## ```
  def fibonacci(n : Int32) : Int64
    return n.to_i64 if n <= 1
    prev, curr = 0_i64, 1_i64
    (n - 1).times { prev, curr = curr, prev + curr }
    curr
  end

  ## ตรวจสอบว่าตัวเลขเป็นจำนวนเฉพาะหรือไม่
  ##
  ## ใช้ trial division algorithm
  ##
  ## - *n* : ตัวเลขที่ต้องการตรวจสอบ
  ##
  ## ตัวอย่าง:
  ## ```
  ## is_prime?(2)  # => true
  ## is_prime?(17) # => true
  ## is_prime?(15) # => false
  ## ```
  def is_prime?(n : Int32) : Bool
    return false if n < 2
    return true if n == 2
    return false if n.even?
    (3..Math.sqrt(n.to_f).to_i).step(2).none? { |i| n % i == 0 }
  end
end
```

---

## 10.8 Documenting Constants และ Types

```crystal
## ค่าคงที่ทางคณิตศาสตร์
module MathConstants
  ## ค่า Pi (π) - อัตราส่วนเส้นรอบวงต่อเส้นผ่านศูนย์กลาง
  PI = 3.14159265358979323846

  ## ค่า Euler's number (e) - ฐานของ natural logarithm
  E = 2.71828182845904523536

  ## ค่า Golden Ratio (φ) - อัตราส่วนทอง
  PHI = 1.61803398874989484820

  ## ค่า Square root of 2
  SQRT2 = 1.41421356237309504880
end

## ประเภทข้อมูลสำหรับระบบ
module Types
  ## ID ของ entity ในระบบ
  alias EntityId = String

  ## Timestamp ในรูปแบบ Unix epoch (milliseconds)
  alias Timestamp = Int64

  ## ข้อมูล key-value สำหรับ metadata
  alias Metadata = Hash(String, String)

  ## Result type สำหรับ operations ที่อาจล้มเหลว
  alias Result(T) = T | Error
end
```

---

## 10.9 Code Examples ใน Documentation

Crystal Docs รองรับ code examples ใน markdown format

```crystal
## คำนวณ Body Mass Index (BMI)
##
## สูตร: BMI = น้ำหนัก(kg) / ส่วนสูง²(m)
##
## ตัวอย่างการใช้งาน:
## ```crystal
## bmi = calculate_bmi(70.0, 1.75)
## puts bmi        # => 22.857142857142858
## puts bmi.round(1)  # => 22.9
##
## case bmi
## when ..18.5
##   puts "น้ำหนักน้อย"
## when 18.5..25.0
##   puts "ปกติ"
## when 25.0..30.0
##   puts "น้ำหนักเกิน"
## else
##   puts "อ้วน"
## end
## ```
def calculate_bmi(weight_kg : Float64, height_m : Float64) : Float64
  raise ArgumentError.new("ส่วนสูงต้องเป็นบวก") if height_m <= 0
  weight_kg / (height_m ** 2)
end
```

### Code ที่ไม่ควร run ใน test

```crystal
## เปิดไฟล์และอ่านเนื้อหา
##
## ```crystal
## # ตัวอย่างนี้ไม่ run ใน doc tests
## # :nodoc: ใช้ปิดการทดสอบ
## content = read_file("/path/to/file")  # skip
## ```
def read_file(path : String) : String
  File.read(path)
end
```

---

## 10.10 :nodoc: Directive

ใช้ `:nodoc:` เพื่อซ่อน API จาก documentation

```crystal
class PublicClass
  ## Method นี้ถูก document
  def public_method
    "สาธารณะ"
  end

  # :nodoc:
  # Method นี้จะไม่ปรากฏใน docs
  def internal_helper
    "internal"
  end
end

# :nodoc:
# Class นี้จะไม่ปรากฏใน docs ทั้งหมด
class InternalClass
  def do_something
    "internal"
  end
end
```

---

## 10.11 ditto Directive

`:ditto:` ใช้คัดลอก documentation จาก method ก่อนหน้า

```crystal
class StringUtils
  ## แปลงข้อความเป็นตัวพิมพ์ใหญ่
  def uppercase(str : String) : String
    str.upcase
  end

  # :ditto:
  def to_upper(str : String) : String
    str.upcase
  end

  # :ditto:
  def upper(str : String) : String
    str.upcase
  end
end
```

---

## 10.12 ตัวอย่าง API Documentation สมบูรณ์

```crystal
## `Cache` จัดการ in-memory cache พร้อม TTL (Time-To-Live)
##
## รองรับการเก็บค่าใดก็ได้พร้อมกำหนดเวลาหมดอายุ
## ใช้ Hash เป็น storage ภายใน
##
## ตัวอย่างการใช้งาน:
## ```crystal
## cache = Cache(String).new
## cache.set("user:1", "Alice", ttl: 300)  # หมดอายุใน 5 นาที
## puts cache.get("user:1")  # => "Alice"
## sleep 301
## puts cache.get("user:1")  # => nil (หมดอายุแล้ว)
## ```
##
## Thread Safety: ไม่รองรับ concurrent access โดยตรง
## ถ้าต้องการ thread-safe cache ให้ใช้ `ThreadSafeCache`
class Cache(T)
  ## โครงสร้างเก็บข้อมูล cache entry
  private record Entry(T),
    value : T,
    expires_at : Time

  ## จำนวน entries ปัจจุบันใน cache
  getter size : Int32

  ## จำนวน cache hits ตั้งแต่เริ่มต้น
  getter hits : Int32

  ## จำนวน cache misses ตั้งแต่เริ่มต้น
  getter misses : Int32

  ## สร้าง Cache ใหม่
  ##
  ## - *max_size* : จำนวน entries สูงสุด (ค่าเริ่มต้น: 1000)
  ## - *default_ttl* : TTL เริ่มต้นในวินาที (ค่าเริ่มต้น: 3600)
  def initialize(@max_size : Int32 = 1000, @default_ttl : Int32 = 3600)
    @store = {} of String => Entry(T)
    @size = 0
    @hits = 0
    @misses = 0
  end

  ## เก็บค่าลง cache
  ##
  ## - *key* : key สำหรับค้นหา
  ## - *value* : ค่าที่ต้องการเก็บ
  ## - *ttl* : เวลาหมดอายุในวินาที (ใช้ default_ttl ถ้าไม่ระบุ)
  ##
  ## Raises `CacheFullError` ถ้า cache เต็มและไม่สามารถ evict ได้
  def set(key : String, value : T, ttl : Int32 = @default_ttl) : self
    evict! if @size >= @max_size && !@store.has_key?(key)
    
    expires_at = Time.utc + ttl.seconds
    @store[key] = Entry(T).new(value, expires_at)
    @size = @store.size
    self
  end

  ## ดึงค่าจาก cache
  ##
  ## คืน nil ถ้าไม่พบ key หรือ entry หมดอายุแล้ว
  ##
  ## - *key* : key ที่ต้องการค้นหา
  ##
  ## ตัวอย่าง:
  ## ```crystal
  ## cache.set("x", 42)
  ## cache.get("x")  # => 42
  ## cache.get("y")  # => nil
  ## ```
  def get(key : String) : T?
    entry = @store[key]?
    
    if entry.nil?
      @misses += 1
      return nil
    end
    
    if Time.utc > entry.expires_at
      @store.delete(key)
      @size -= 1
      @misses += 1
      return nil
    end
    
    @hits += 1
    entry.value
  end

  ## ลบ entry ออกจาก cache
  ##
  ## - *key* : key ที่ต้องการลบ
  ##
  ## คืน `true` ถ้าลบสำเร็จ, `false` ถ้าไม่พบ key
  def delete(key : String) : Bool
    if @store.delete(key)
      @size -= 1
      true
    else
      false
    end
  end

  ## ล้าง cache ทั้งหมด
  def clear : self
    @store.clear
    @size = 0
    self
  end

  ## ตรวจสอบว่า key มีอยู่ใน cache และยังไม่หมดอายุ
  ##
  ## - *key* : key ที่ต้องการตรวจสอบ
  def has?(key : String) : Bool
    !get(key).nil?
  end

  ## สถิติการใช้ cache
  ##
  ## คืน Hash ที่มีข้อมูล: size, hits, misses, hit_rate
  def stats : Hash(String, Float64 | Int32)
    total = @hits + @misses
    hit_rate = total > 0 ? (@hits.to_f / total * 100).round(2) : 0.0
    
    {
      "size"     => @size,
      "hits"     => @hits,
      "misses"   => @misses,
      "hit_rate" => hit_rate
    } of String => Float64 | Int32
  end

  private def evict!
    # ลบ entries ที่หมดอายุก่อน
    expired_keys = @store.select { |_, entry| Time.utc > entry.expires_at }.keys
    expired_keys.each { |k| @store.delete(k) }
    @size = @store.size
    
    # ถ้ายังเต็มอยู่ ลบ entry เก่าที่สุด (FIFO)
    if @size >= @max_size
      first_key = @store.first_key?
      @store.delete(first_key) if first_key
      @size = @store.size
    end
  end
end

## ข้อผิดพลาดเมื่อ cache เต็ม
class CacheFullError < Exception; end
```

---

## 10.13 Documentation ใน Struct

```crystal
## ตำแหน่งในระบบพิกัด 2 มิติ
##
## ```crystal
## pos = Point.new(3.0, 4.0)
## pos.distance_from_origin  # => 5.0
## ```
record Point, x : Float64, y : Float64 do
  ## คำนวณระยะห่างจากจุด origin (0, 0)
  ##
  ## ใช้ Pythagorean theorem: √(x² + y²)
  def distance_from_origin : Float64
    Math.sqrt(x ** 2 + y ** 2)
  end

  ## คำนวณระยะห่างระหว่างสองจุด
  ##
  ## - *other* : จุดปลายทาง
  def distance_to(other : Point) : Float64
    Math.sqrt((x - other.x) ** 2 + (y - other.y) ** 2)
  end

  ## แปลง Point เป็น String
  ##
  ## ตัวอย่าง: `"(3.0, 4.0)"`
  def to_s(io : IO) : Nil
    io << "(#{x}, #{y})"
  end
end
```

---

## 10.14 Best Practices

### 1. เขียน Documentation ก่อนเขียน Code

```crystal
## คำนวณค่าเฉลี่ยของตัวเลขใน array
##
## - *numbers* : array ของตัวเลข (ต้องไม่ว่างเปล่า)
##
## Raises `ArgumentError` ถ้า array ว่างเปล่า
##
## ตัวอย่าง:
## ```crystal
## average([1, 2, 3, 4, 5])  # => 3.0
## average([10.0, 20.0])      # => 15.0
## ```
def average(numbers : Array(Number)) : Float64
  raise ArgumentError.new("Array ต้องไม่ว่างเปล่า") if numbers.empty?
  numbers.sum.to_f / numbers.size
end
```

### 2. Document Public API เท่านั้น

```crystal
class PasswordHasher
  ## Hash รหัสผ่านด้วย bcrypt
  ##
  ## - *password* : รหัสผ่านที่ต้องการ hash
  ## - *cost* : ความยาก (2^cost iterations, ค่าเริ่มต้น: 12)
  def hash_password(password : String, cost : Int32 = 12) : String
    # public method - ต้อง document
    bcrypt_hash(password, cost)
  end

  private def bcrypt_hash(password : String, cost : Int32) : String
    # private method - ไม่ต้อง document (แต่ comment ได้)
    # implementation...
    "hashed_#{password}"
  end
end
```

### 3. ใช้ Examples ที่ทดสอบได้

```crystal
## แปลง CamelCase เป็น snake_case
##
## ตัวอย่าง:
## ```crystal
## to_snake_case("HelloWorld")    # => "hello_world"
## to_snake_case("myAPIMethod")   # => "my_api_method"
## to_snake_case("simple")        # => "simple"
## ```
def to_snake_case(str : String) : String
  str.gsub(/([A-Z]+)([A-Z][a-z])/, "\\1_\\2")
     .gsub(/([a-z\d])([A-Z])/, "\\1_\\2")
     .downcase
end
```

---

## 10.15 ตัวอย่างโปรแกรม: HTTP Client Documentation

```crystal
## Simple HTTP client สำหรับทำ API requests
##
## รองรับ GET, POST, PUT, DELETE methods
## พร้อม automatic JSON parsing และ error handling
##
## ตัวอย่างการใช้งาน:
## ```crystal
## client = HttpClient.new("https://api.example.com")
## client.headers["Authorization"] = "Bearer token123"
##
## response = client.get("/users/1")
## puts response.status  # => 200
## puts response.body    # => {"id": 1, "name": "Alice"}
##
## data = {"name" => "Bob", "email" => "bob@example.com"}
## response = client.post("/users", data)
## puts response.status  # => 201
## ```
class HttpClient
  ## Base URL ของ API
  getter base_url : String

  ## Headers ที่จะส่งทุก request
  getter headers : Hash(String, String)

  ## Timeout ในวินาที (ค่าเริ่มต้น: 30)
  property timeout : Int32

  ## สร้าง HTTP client ใหม่
  ##
  ## - *base_url* : URL พื้นฐานของ API
  ## - *timeout* : timeout ในวินาที
  def initialize(@base_url : String, @timeout : Int32 = 30)
    @headers = {} of String => String
    @headers["Content-Type"] = "application/json"
    @headers["Accept"] = "application/json"
  end

  ## ทำ GET request
  ##
  ## - *path* : path ของ endpoint (เช่น "/users/1")
  ## - *params* : query parameters (optional)
  ##
  ## คืน `Response` พร้อม status code และ body
  ##
  ## Raises `ConnectionError` ถ้าเชื่อมต่อไม่ได้
  ## Raises `TimeoutError` ถ้า request timeout
  def get(path : String, params : Hash(String, String)? = nil) : Response
    url = build_url(path, params)
    make_request("GET", url, nil)
  end

  ## ทำ POST request พร้อม body
  ##
  ## - *path* : path ของ endpoint
  ## - *body* : ข้อมูลที่จะส่ง (จะถูกแปลงเป็น JSON อัตโนมัติ)
  ##
  ## คืน `Response` พร้อม status code และ body
  def post(path : String, body : Hash(String, String) | Nil = nil) : Response
    url = build_url(path, nil)
    make_request("POST", url, body.try(&.to_json))
  end

  ## โครงสร้าง HTTP Response
  record Response,
    status : Int32,
    body : String,
    headers : Hash(String, String)

  private def build_url(path : String, params : Hash(String, String)?) : String
    url = "#{@base_url}#{path}"
    if params && !params.empty?
      query = params.map { |k, v| "#{k}=#{v}" }.join("&")
      url = "#{url}?#{query}"
    end
    url
  end

  private def make_request(method : String, url : String, body : String?) : Response
    # จำลองการทำงาน
    Response.new(200, "{}", {} of String => String)
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Document Stack Class
เพิ่ม documentation ให้กับ Stack class นี้ให้ครบถ้วน

```crystal
class Stack(T)
  def initialize
    @data = [] of T
  end

  def push(item : T) : self
    @data.push(item)
    self
  end

  def pop : T?
    @data.pop?
  end

  def peek : T?
    @data.last?
  end

  def empty? : Bool
    @data.empty?
  end

  def size : Int32
    @data.size
  end

  def to_s(io : IO) : Nil
    io << "Stack(#{@data.join(", ")})"
  end
end
```

### แบบฝึกหัดที่ 2: เขียน Module Documentation
สร้าง module `StringExtensions` พร้อม documentation ครบถ้วน

```crystal
# เพิ่ม documentation ให้ methods เหล่านี้
module StringExtensions
  def palindrome?(str : String) : Bool
    str == str.reverse
  end

  def word_count(str : String) : Hash(String, Int32)
    str.split.each_with_object({} of String => Int32) do |word, counts|
      counts[word.downcase] = (counts[word.downcase]? || 0) + 1
    end
  end

  def truncate(str : String, length : Int32, suffix : String = "...") : String
    return str if str.size <= length
    str[0, length - suffix.size] + suffix
  end
end
```

### แบบฝึกหัดที่ 3: Generate Docs
```bash
# สร้างไฟล์ Crystal ที่มี documentation ครบถ้วน
# แล้วรัน crystal docs และตรวจสอบผลลัพธ์
crystal docs your_file.cr --output=./docs
```

---

## สรุป

ในบทนี้เราได้เรียนรู้เกี่ยวกับ:

1. **Single-line Comments (#)**: อธิบายโค้ดบรรทัดเดียวหรือท้ายบรรทัด
2. **Multiline Comments (=begin/=end)**: อธิบายโค้ดหลายบรรทัด
3. **Documentation Comments (##)**: ใช้ generate API docs อัตโนมัติ
4. **crystal docs**: คำสั่ง generate HTML documentation
5. **Code Examples**: ใส่ตัวอย่างใน docs เพื่อให้เข้าใจง่าย
6. **:nodoc:**: ซ่อน API จาก documentation
7. **Best Practices**: เขียน docs ที่ดี ชัดเจน ทดสอบได้

### Checklist สำหรับ Good Documentation

- [ ] Document ทุก public method
- [ ] อธิบาย parameters และ return type
- [ ] ระบุ exceptions ที่อาจเกิดขึ้น
- [ ] ใส่ code examples ที่ runnable
- [ ] อธิบาย edge cases
- [ ] ใช้ภาษาที่ชัดเจนและตรงประเด็น
- [ ] Update docs เมื่อแก้ไข code

Documentation ที่ดีช่วยให้ผู้ใช้ library เข้าใจวิธีใช้งานได้โดยไม่ต้องอ่านโค้ด และช่วยให้ตัวเองจำการทำงานของโค้ดได้ในอนาคต!
