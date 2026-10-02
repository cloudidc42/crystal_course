# Part 44: Protected และ Private ใน Crystal

## บทนำ

Visibility modifiers ใน Crystal กำหนดว่า methods และ instance variables สามารถเข้าถึงได้จากที่ไหน Crystal มี 3 ระดับ: `public` (default), `protected`, และ `private` ในบทนี้เราจะเรียนรู้ความแตกต่างและกรณีการใช้งานที่เหมาะสม

---

## 44.1 Public Methods (Default)

โดย default methods ทุกตัวใน Crystal เป็น `public` หมายความว่าสามารถเรียกได้จากทุกที่

```crystal
class BankAccount
  def initialize(owner : String, initial_balance : Float64 = 0.0)
    @owner = owner
    @balance = initial_balance
  end
  
  # Public methods - เรียกได้จากทุกที่
  def deposit(amount : Float64)
    validate_amount(amount)
    @balance += amount
    log_transaction("deposit", amount)
    self
  end
  
  def withdraw(amount : Float64)
    validate_amount(amount)
    check_sufficient_funds(amount)
    @balance -= amount
    log_transaction("withdrawal", amount)
    self
  end
  
  def balance : Float64
    @balance
  end
  
  def owner : String
    @owner
  end
  
  # Private methods - เรียกได้เฉพาะใน class เท่านั้น
  private def validate_amount(amount : Float64)
    raise ArgumentError.new("Amount must be positive") if amount <= 0
  end
  
  private def check_sufficient_funds(amount : Float64)
    raise "Insufficient funds" if amount > @balance
  end
  
  private def log_transaction(type : String, amount : Float64)
    puts "[LOG] #{@owner}: #{type} #{amount} (balance: #{@balance})"
  end
end

account = BankAccount.new("Alice", 1000.0)
account.deposit(500.0)
account.withdraw(200.0)
puts account.balance  # => 1300.0

# account.validate_amount(100.0)  # Error! private method
```

---

## 44.2 Private Methods

`private` methods สามารถเรียกได้เฉพาะภายใน class เดียวกันเท่านั้น ไม่สามารถเรียกจากภายนอกหรือ subclass

```crystal
class PasswordManager
  def initialize
    @passwords = {} of String => String
  end
  
  def store(service : String, password : String)
    encrypted = encrypt(password)
    @passwords[service] = encrypted
    puts "Password stored for #{service}"
  end
  
  def retrieve(service : String) : String?
    encrypted = @passwords[service]?
    encrypted ? decrypt(encrypted) : nil
  end
  
  def has_password?(service : String) : Bool
    @passwords.has_key?(service)
  end
  
  private def encrypt(text : String) : String
    # จำลองการ encrypt ด้วย simple caesar cipher
    text.chars.map { |c| (c.ord + 3).chr }.join
  end
  
  private def decrypt(text : String) : String
    # จำลองการ decrypt
    text.chars.map { |c| (c.ord - 3).chr }.join
  end
  
  private def hash_key(key : String) : String
    # จำลอง hashing
    key.reverse.upcase
  end
end

pm = PasswordManager.new
pm.store("gmail", "mypassword123")
pm.store("github", "secretkey456")

puts pm.retrieve("gmail")  # => "mypassword123"
puts pm.has_password?("github")  # => true

# pm.encrypt("test")  # Error! private method
```

### Private Methods ใน Subclass

```crystal
class Shape
  def area : Float64
    calculate_area
  end
  
  def describe : String
    "Shape with area #{area.round(2)}"
  end
  
  private def calculate_area : Float64
    0.0  # default implementation
  end
end

class Circle < Shape
  def initialize(@radius : Float64)
  end
  
  private def calculate_area : Float64
    Math::PI * @radius ** 2
  end
end

class Square < Shape
  def initialize(@side : Float64)
  end
  
  private def calculate_area : Float64
    @side ** 2
  end
end

shapes = [Circle.new(5.0), Square.new(4.0)] of Shape
shapes.each { |s| puts s.describe }
# Shape with area 78.54
# Shape with area 16.0
```

---

## 44.3 Protected Methods

`protected` methods สามารถเรียกได้:
1. ภายใน class เอง
2. จาก subclass
3. จาก instances อื่นของ class เดียวกัน

```crystal
class Employee
  def initialize(@name : String, @salary : Float64, @department : String)
  end
  
  getter name : String
  getter department : String
  
  # Public: ทุกคนเห็น
  def salary_range : String
    if salary < 30000
      "Entry Level"
    elsif salary < 60000
      "Mid Level"
    else
      "Senior Level"
    end
  end
  
  # Protected: เฉพาะ Employee instances เท่านั้น
  protected def salary : Float64
    @salary
  end
  
  # เปรียบเทียบ salary กับ employee อื่น
  # สามารถเข้าถึง protected salary ของ other ได้
  def earns_more_than?(other : Employee) : Bool
    @salary > other.salary  # เข้าถึง protected method ของ other ได้!
  end
  
  def to_s(io : IO) : Nil
    io << "#{@name} (#{department}) - #{salary_range}"
  end
end

emp1 = Employee.new("Alice", 75000.0, "Engineering")
emp2 = Employee.new("Bob", 55000.0, "Marketing")

puts emp1.earns_more_than?(emp2)  # => true
puts emp2.earns_more_than?(emp1)  # => false

# emp1.salary  # Error! salary เป็น protected
puts emp1.salary_range  # => "Senior Level"
```

### Protected ใน Inheritance

```crystal
class Animal
  def initialize(@name : String, @age : Int32)
  end
  
  getter name : String
  
  protected getter age : Int32
  
  def older_than?(other : Animal) : Bool
    @age > other.age  # เข้าถึง protected age ได้ใน class hierarchy
  end
  
  protected def make_sound : String
    "..."
  end
  
  def speak : String
    "#{@name} says: #{make_sound}"
  end
end

class Dog < Animal
  def initialize(name : String, age : Int32, @breed : String)
    super(name, age)
  end
  
  protected def make_sound : String
    "Woof!"
  end
  
  def compare_age(other : Dog) : String
    # สามารถเรียก protected method ของ other ได้ถ้าเป็น class เดียวกัน
    if older_than?(other)
      "#{name} is older than #{other.name}"
    else
      "#{name} is younger than #{other.name}"
    end
  end
end

class Cat < Animal
  protected def make_sound : String
    "Meow!"
  end
end

fido = Dog.new("Fido", 5, "Labrador")
rex = Dog.new("Rex", 3, "German Shepherd")
whiskers = Cat.new("Whiskers", 4)

puts fido.speak               # => "Fido says: Woof!"
puts whiskers.speak           # => "Whiskers says: Meow!"
puts fido.compare_age(rex)    # => "Fido is older than Rex"
puts fido.older_than?(whiskers)  # => true (Animal level)

# fido.make_sound  # Error! protected
# fido.age         # Error! protected
```

---

## 44.4 Private Instance Variables

ใน Crystal instance variables (`@var`) เป็น private โดย default อยู่แล้ว

```crystal
class Secret
  def initialize(@secret_value : String, @public_name : String)
  end
  
  # @secret_value ไม่สามารถเข้าถึงได้จากภายนอก
  getter public_name : String
  
  def reveal_to(password : String) : String?
    if password == "correct_password"
      @secret_value
    else
      nil
    end
  end
  
  def verify(value : String) : Bool
    @secret_value == value
  end
end

s = Secret.new("my_secret_123", "MyService")
puts s.public_name  # => "MyService"
# puts s.secret_value  # Error! no method
puts s.verify("wrong")         # => false
puts s.verify("my_secret_123") # => true
puts s.reveal_to("wrong_pass") # => nil
puts s.reveal_to("correct_password")  # => "my_secret_123"
```

---

## 44.5 Private Class Methods

Class methods ก็สามารถเป็น private ได้

```crystal
class DatabaseConnection
  @@instance : DatabaseConnection?
  
  def self.get_instance : DatabaseConnection
    @@instance ||= create_connection
  end
  
  def query(sql : String) : String
    "Result of: #{sql}"
  end
  
  def close
    puts "Connection closed"
    @@instance = nil
  end
  
  # Private class method - ห้ามสร้าง instance โดยตรง
  private def self.create_connection : DatabaseConnection
    puts "Creating new database connection..."
    new
  end
  
  private def initialize
    @connected = true
  end
end

db = DatabaseConnection.get_instance
puts db.query("SELECT * FROM users")  # => "Result of: SELECT * FROM users"

# DatabaseConnection.create_connection  # Error! private class method
# DatabaseConnection.new                # Error! private constructor
```

### Class-level vs Instance-level Private

```crystal
class Calculator
  def initialize
    @history = [] of String
  end
  
  # Public instance method
  def calculate(expression : String) : Float64
    result = evaluate(expression)
    record_history(expression, result)
    result
  end
  
  def history : Array(String)
    @history.dup
  end
  
  # Private instance method
  private def evaluate(expression : String) : Float64
    # จำลองการคำนวณ
    42.0
  end
  
  private def record_history(expression : String, result : Float64)
    @history << "#{expression} = #{result}"
  end
  
  # Public class method
  def self.simple_add(a : Float64, b : Float64) : Float64
    a + b
  end
  
  # Private class method
  private def self.validate_inputs(*values : Float64)
    values.each do |v|
      raise ArgumentError.new("NaN not allowed") if v.nan?
    end
  end
end

calc = Calculator.new
calc.calculate("1 + 2")
calc.calculate("3 * 4")
puts calc.history

puts Calculator.simple_add(3.0, 4.0)  # => 7.0
```

---

## 44.6 Visibility Rules สรุป

```crystal
class VisibilityDemo
  def public_method
    "Everyone can call this"
  end
  
  protected def protected_method
    "Only class and subclass instances can call this"
  end
  
  private def private_method
    "Only called within this class"
  end
  
  def demo_calling_private
    # OK - เรียก private จากภายใน class
    private_method
  end
  
  def demo_calling_protected
    # OK - เรียก protected จากภายใน class
    protected_method
  end
end

class VisibilitySubclass < VisibilityDemo
  def call_from_subclass
    # OK - เรียก protected จาก subclass
    protected_method
  end
  
  def try_private_from_subclass
    # Error! ไม่สามารถเรียก private จาก subclass
    # private_method
  end
end

obj = VisibilityDemo.new
obj.public_method        # OK
# obj.protected_method   # Error!
# obj.private_method     # Error!

sub = VisibilitySubclass.new
sub.public_method        # OK
sub.call_from_subclass   # OK (เรียก protected ผ่าน instance method)
# sub.protected_method   # Error! ถึงแม้ subclass จะ access ได้ แต่ from outside ไม่ได้
```

---

## 44.7 Use Cases สำหรับ Private

### Helper Methods

```crystal
class TextProcessor
  def process(text : String) : String
    text
      .pipe { |t| normalize_whitespace(t) }
      .pipe { |t| remove_special_chars(t) }
      .pipe { |t| capitalize_words(t) }
  end
  
  private def normalize_whitespace(text : String) : String
    text.split.join(" ")
  end
  
  private def remove_special_chars(text : String) : String
    text.gsub(/[^a-zA-Z0-9\s]/, "")
  end
  
  private def capitalize_words(text : String) : String
    text.split.map(&.capitalize).join(" ")
  end
end

class String
  def pipe(&block : String -> String) : String
    block.call(self)
  end
end

processor = TextProcessor.new
puts processor.process("  hello,   WORLD!  how are   you?  ")
# => "Hello World How Are You"
```

### Validation Helpers

```crystal
class UserRegistration
  def initialize(
    @username : String,
    @email : String,
    @password : String,
    @age : Int32
  )
    validate!
  end
  
  def valid? : Bool
    validate_username && validate_email && validate_password && validate_age
  end
  
  def to_s(io : IO) : Nil
    io << "User: #{@username} <#{@email}>"
  end
  
  private def validate!
    errors = [] of String
    errors << "Invalid username" unless validate_username
    errors << "Invalid email" unless validate_email
    errors << "Password too weak" unless validate_password
    errors << "Must be 18+" unless validate_age
    
    raise ArgumentError.new("Validation failed: #{errors.join(", ")}") unless errors.empty?
  end
  
  private def validate_username : Bool
    @username.size >= 3 && @username.size <= 20 && @username.matches?(/^[a-zA-Z0-9_]+$/)
  end
  
  private def validate_email : Bool
    @email.includes?("@") && @email.includes?(".")
  end
  
  private def validate_password : Bool
    @password.size >= 8 &&
    @password.matches?(/[A-Z]/) &&
    @password.matches?(/[0-9]/)
  end
  
  private def validate_age : Bool
    @age >= 18
  end
end

begin
  user = UserRegistration.new("alice_dev", "alice@example.com", "SecurePass123", 25)
  puts user  # => User: alice_dev <alice@example.com>
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

begin
  bad_user = UserRegistration.new("ab", "notanemail", "weak", 15)
rescue ArgumentError => e
  puts "Error: #{e.message}"
  # Error: Validation failed: Invalid username, Invalid email, Password too weak, Must be 18+
end
```

---

## 44.8 Use Cases สำหรับ Protected

### Comparing Objects

```crystal
class Temperature
  include Comparable(Temperature)
  
  def initialize(@celsius : Float64)
  end
  
  protected getter celsius : Float64
  
  def <=>(other : Temperature) : Int32
    @celsius <=> other.celsius  # เข้าถึง protected celsius ของ other ได้
  end
  
  def to_fahrenheit : Float64
    @celsius * 9 / 5 + 32
  end
  
  def to_s(io : IO) : Nil
    io << "#{@celsius}°C"
  end
end

temps = [
  Temperature.new(100.0),
  Temperature.new(37.0),
  Temperature.new(-10.0),
  Temperature.new(0.0)
]

sorted = temps.sort
sorted.each { |t| puts t }
# -10.0°C
# 0.0°C
# 37.0°C
# 100.0°C

puts temps.max  # => 100.0°C
puts temps.min  # => -10.0°C
```

### Shared Behavior ใน Hierarchy

```crystal
class Account
  def initialize(@number : String, @balance : Float64)
  end
  
  getter number : String
  
  def balance : String
    "Balance: $#{@balance.round(2)}"
  end
  
  protected def raw_balance : Float64
    @balance
  end
  
  protected def adjust_balance(amount : Float64)
    @balance += amount
  end
end

class SavingsAccount < Account
  def initialize(number : String, balance : Float64, @interest_rate : Float64)
    super(number, balance)
  end
  
  def apply_interest
    interest = raw_balance * (@interest_rate / 100)  # ใช้ protected method
    adjust_balance(interest)                           # ใช้ protected method
    puts "Applied #{@interest_rate}% interest: +$#{interest.round(2)}"
  end
end

class CheckingAccount < Account
  def initialize(number : String, balance : Float64, @overdraft_limit : Float64)
    super(number, balance)
  end
  
  def withdraw(amount : Float64)
    available = raw_balance + @overdraft_limit  # ใช้ protected method
    if amount <= available
      adjust_balance(-amount)  # ใช้ protected method
      puts "Withdrew $#{amount}"
    else
      puts "Exceeds limit"
    end
  end
end

savings = SavingsAccount.new("SAV001", 5000.0, 2.5)
checking = CheckingAccount.new("CHK001", 1000.0, 500.0)

puts savings.balance
savings.apply_interest
puts savings.balance

puts checking.balance
checking.withdraw(1400.0)  # เกิน balance แต่ไม่เกิน overdraft
puts checking.balance

# savings.raw_balance  # Error! protected from outside
```

---

## 44.9 Private Class Variables

Class variables (`@@var`) ก็สามารถควบคุม visibility ได้ผ่าน methods

```crystal
class Counter
  @@count : Int32 = 0
  @@instances = [] of WeakRef(Counter)
  
  def initialize(@name : String)
    @@count += 1
    puts "Created counter ##{@@count}: #{@name}"
  end
  
  # Public: อ่าน count ได้
  def self.count : Int32
    @@count
  end
  
  getter name : String
  
  # Private: ห้ามรีเซ็ตจากภายนอก  
  private def self.reset_count
    @@count = 0
  end
  
  # Internal use only
  def self.internal_reset  # เรียก private class method
    # reset_count  # ไม่สามารถเรียก private class method จาก instance method
  end
end

c1 = Counter.new("First")
c2 = Counter.new("Second")
c3 = Counter.new("Third")

puts Counter.count  # => 3
# Counter.reset_count  # Error! private class method
```

---

## 44.10 ตัวอย่างในชีวิตจริง: HTTP Client

```crystal
class HttpClient
  def initialize(@base_url : String, @timeout : Float64 = 30.0)
    @headers = {"Content-Type" => "application/json"}
    @auth_token = nil.as(String?)
  end
  
  # Public API
  def get(path : String) : String
    url = build_url(path)
    make_request("GET", url, nil)
  end
  
  def post(path : String, body : String) : String
    url = build_url(path)
    make_request("POST", url, body)
  end
  
  def authenticate(token : String)
    @auth_token = token
    add_auth_header(token)
  end
  
  def set_header(key : String, value : String)
    @headers[key] = value
  end
  
  # Protected: subclass สามารถ override
  protected def make_request(method : String, url : String, body : String?) : String
    # จำลอง HTTP request
    puts "#{method} #{url}"
    body ? "Response to: #{body}" : "Response"
  end
  
  # Private: implementation details
  private def build_url(path : String) : String
    "#{@base_url}#{path.starts_with?("/") ? "" : "/"}#{path}"
  end
  
  private def add_auth_header(token : String)
    @headers["Authorization"] = "Bearer #{token}"
  end
  
  private def headers_string : String
    @headers.map { |k, v| "#{k}: #{v}" }.join(", ")
  end
end

class LoggingHttpClient < HttpClient
  def initialize(base_url : String)
    super
    @log = [] of String
  end
  
  protected def make_request(method : String, url : String, body : String?) : String
    @log << "#{Time.local}: #{method} #{url}"
    result = super
    @log << "Response received"
    result
  end
  
  def print_log
    @log.each { |entry| puts entry }
  end
end

client = LoggingHttpClient.new("https://api.example.com")
client.authenticate("my_token")
client.get("/users")
client.post("/users", "{\"name\": \"Alice\"}")
client.print_log
```

---

## 44.11 Anti-patterns และ Best Practices

### ไม่ควรทำ

```crystal
# Bad: Public methods ที่ไม่ควร public
class UserService
  def initialize(@db : Database)
  end
  
  def create_user(name : String, email : String)
    validate_data(name, email)
    insert_to_db(name, email)
  end
  
  # ควรเป็น private! ไม่ควร public
  def validate_data(name : String, email : String)
    raise "Invalid" if name.empty?
  end
  
  # ควรเป็น private! ไม่ควร public
  def insert_to_db(name : String, email : String)
    # database operations
  end
end
```

### ควรทำ

```crystal
class UserService
  def initialize(@db : Database)
  end
  
  def create_user(name : String, email : String)
    validate_data(name, email)
    insert_to_db(name, email)
  end
  
  private def validate_data(name : String, email : String)
    raise "Invalid" if name.empty?
  end
  
  private def insert_to_db(name : String, email : String)
    # database operations
  end
end

# Placeholder class เพื่อให้ compile ได้
class Database; end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Secure Configuration

```crystal
class SecureConfig
  def initialize(
    @host : String,
    @port : Int32,
    @username : String,
    password : String,
    @ssl : Bool = true
  )
    @password_hash = hash_password(password)
    validate_config!
  end
  
  getter host : String
  getter port : Int32
  getter username : String
  getter? ssl : Bool
  
  def authenticate(password : String) : Bool
    hash_password(password) == @password_hash
  end
  
  def connection_string : String
    protocol = ssl? ? "ssl" : "plain"
    "#{protocol}://#{username}@#{host}:#{port}"
  end
  
  private def hash_password(password : String) : String
    # Simple mock hash
    password.reverse.upcase + "_HASHED"
  end
  
  private def validate_config!
    raise "Invalid host" if @host.empty?
    raise "Invalid port" if @port < 1 || @port > 65535
    raise "Invalid username" if @username.empty?
  end
end

config = SecureConfig.new("db.example.com", 5432, "admin", "SecretPass!")
puts config.connection_string
puts config.authenticate("SecretPass!")  # => true
puts config.authenticate("wrongpass")   # => false
# config.hash_password("test")  # Error! private
```

### แบบฝึกหัดที่ 2: Compare Objects

```crystal
class Version
  include Comparable(Version)
  
  def initialize(version_string : String)
    parts = version_string.split(".")
    @major = parts[0].to_i
    @minor = parts[1]?.try(&.to_i) || 0
    @patch = parts[2]?.try(&.to_i) || 0
  end
  
  protected getter major : Int32
  protected getter minor : Int32
  protected getter patch : Int32
  
  def <=>(other : Version) : Int32
    return major <=> other.major unless major == other.major
    return minor <=> other.minor unless minor == other.minor
    patch <=> other.patch
  end
  
  def to_s(io : IO) : Nil
    io << "#{major}.#{minor}.#{patch}"
  end
end

versions = ["2.1.0", "1.0.0", "2.0.5", "1.5.3", "2.1.1"].map { |v| Version.new(v) }
puts versions.sort.map(&.to_s).join(", ")
# => 1.0.0, 1.5.3, 2.0.5, 2.1.0, 2.1.1

puts versions.max  # => 2.1.1
```

---

## สรุป

| Visibility | เรียกจากภายใน class | เรียกจาก subclass | เรียกจากภายนอก |
|------------|--------------------|--------------------|----------------|
| `public` | ✓ | ✓ | ✓ |
| `protected` | ✓ | ✓ | ✗ |
| `private` | ✓ | ✗ | ✗ |

**กฎทั่วไป:**
- ใช้ `private` สำหรับ implementation details และ helper methods
- ใช้ `protected` สำหรับ methods ที่ต้องการ share ใน class hierarchy
- เปิด `public` เฉพาะสิ่งที่ต้องการให้ผู้ใช้ class เข้าถึงได้
- Instance variables (`@var`) เป็น private โดย default

---

*ต่อไป: Part 45 - Comparable และ Enumerable*
