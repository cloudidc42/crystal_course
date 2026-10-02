# Part 161: Crystal Spec - Unit Testing

## บทนำ

Crystal มี built-in testing framework ชื่อ `crystal spec` ที่ทรงพลังและใช้งานง่าย ได้รับแรงบันดาลใจจาก RSpec ของ Ruby

## โครงสร้าง Spec

```
my_project/
├── src/
│   ├── calculator.cr
│   └── user.cr
└── spec/
    ├── spec_helper.cr
    ├── calculator_spec.cr
    └── user_spec.cr
```

### spec_helper.cr

```crystal
# spec/spec_helper.cr
require "spec"

# เพิ่ม global setup ที่นี่
# require "../src/my_app"
```

## describe และ it

```crystal
# spec/calculator_spec.cr
require "./spec_helper"
require "../src/calculator"

describe Calculator do
  describe "#add" do
    it "บวกเลขสองตัวได้ถูกต้อง" do
      calc = Calculator.new
      result = calc.add(2, 3)
      result.should eq(5)
    end

    it "บวกเลขลบได้" do
      calc = Calculator.new
      result = calc.add(-5, 3)
      result.should eq(-2)
    end

    it "บวกกับศูนย์ได้" do
      calc = Calculator.new
      result = calc.add(10, 0)
      result.should eq(10)
    end
  end

  describe "#divide" do
    it "หารตัวเลขได้ถูกต้อง" do
      calc = Calculator.new
      result = calc.divide(10.0, 2.0)
      result.should eq(5.0)
    end

    it "raise DivisionByZero เมื่อหารด้วยศูนย์" do
      calc = Calculator.new
      expect_raises(Calculator::DivisionByZero) do
        calc.divide(10.0, 0.0)
      end
    end
  end
end
```

### Calculator Implementation

```crystal
# src/calculator.cr
class Calculator
  class DivisionByZero < Exception
    def initialize
      super("ไม่สามารถหารด้วยศูนย์")
    end
  end

  def add(a : Number, b : Number)
    a + b
  end

  def subtract(a : Number, b : Number)
    a - b
  end

  def multiply(a : Number, b : Number)
    a * b
  end

  def divide(a : Float64, b : Float64) : Float64
    raise DivisionByZero.new if b == 0.0
    a / b
  end

  def power(base : Float64, exp : Int32) : Float64
    base ** exp
  end

  def factorial(n : Int32) : Int64
    raise ArgumentError.new("n must be >= 0") if n < 0
    return 1_i64 if n <= 1
    (1..n).reduce(1_i64) { |acc, i| acc * i }
  end
end
```

## should และ should_not

```crystal
describe "Matchers" do
  # Equality
  it "tests equality" do
    42.should eq(42)
    "hello".should eq("hello")
    [1, 2, 3].should eq([1, 2, 3])
  end

  # Inequality
  it "tests inequality" do
    42.should_not eq(0)
    "hello".should_not eq("world")
  end

  # Type checking
  it "tests types" do
    42.should be_a(Int32)
    "hello".should be_a(String)
    nil.should be_nil
    42.should_not be_nil
  end

  # Boolean
  it "tests boolean" do
    true.should be_true
    false.should be_false
    1.should be_truthy
    nil.should be_falsey
    0.should be_falsey
  end

  # Comparison
  it "tests comparison" do
    5.should be > 3
    3.should be < 5
    5.should be >= 5
    3.should be <= 3
  end

  # String matchers
  it "tests strings" do
    "hello world".should contain("world")
    "hello".should start_with("he")
    "hello".should end_with("lo")
    "hello123".should match(/\d+/)
  end

  # Array/Collection matchers
  it "tests collections" do
    [1, 2, 3].should contain(2)
    [].should be_empty
    [1, 2].should_not be_empty
    [1, 2, 3].size.should eq(3)
  end

  # Range
  it "tests range" do
    7.should be_close(7.001, 0.01)
    3.14159.should be_close(Math::PI, 0.001)
  end
end
```

## expect/to (Crystal ใช้ should แต่มี expect ด้วย)

```crystal
# Crystal ใช้รูปแบบ .should เป็นหลัก
# แต่สามารถใช้ expect ผ่าน custom matchers

describe "Custom Expectations" do
  it "ตรวจสอบ HTTP status" do
    response = HTTP::Client.get("http://example.com")
    response.status_code.should eq(200)
  end

  it "ตรวจสอบ exception message" do
    exception = expect_raises(ArgumentError, "invalid") do
      raise ArgumentError.new("invalid argument")
    end
    exception.message.should contain("invalid")
  end

  it "ตรวจสอบ JSON" do
    json = JSON.parse(%({"name": "Alice", "age": 30}))
    json["name"].as_s.should eq("Alice")
    json["age"].as_i.should eq(30)
  end
end
```

## context

```crystal
describe User do
  describe "#can_admin?" do
    context "เมื่อ user เป็น admin" do
      it "คืนค่า true" do
        user = User.new(email: "admin@example.com", role: "admin")
        user.can_admin?.should be_true
      end
    end

    context "เมื่อ user เป็น regular user" do
      it "คืนค่า false" do
        user = User.new(email: "user@example.com", role: "user")
        user.can_admin?.should be_false
      end
    end

    context "เมื่อ user ถูก deactivate" do
      it "คืนค่า false แม้จะเป็น admin role" do
        user = User.new(email: "admin@example.com", role: "admin", active: false)
        user.can_admin?.should be_false
      end
    end
  end

  describe "#full_name" do
    context "มีทั้ง first_name และ last_name" do
      it "คืนค่า full name" do
        user = User.new(first_name: "Alice", last_name: "Smith")
        user.full_name.should eq("Alice Smith")
      end
    end

    context "มีแค่ first_name" do
      it "คืนค่า first name เท่านั้น" do
        user = User.new(first_name: "Alice")
        user.full_name.should eq("Alice")
      end
    end

    context "ไม่มีชื่อ" do
      it "คืนค่า email" do
        user = User.new(email: "alice@example.com")
        user.full_name.should eq("alice@example.com")
      end
    end
  end
end
```

## before_each และ after_each

```crystal
describe "Database Tests" do
  before_each do
    # ล้าง database ก่อน test แต่ละตัว
    TestDB.truncate_all
    puts "ล้าง DB แล้ว"
  end

  after_each do
    # cleanup หลัง test
    TestDB.cleanup
  end

  before_all do
    # รันครั้งเดียวก่อน describe block
    TestDB.setup
  end

  after_all do
    # รันครั้งเดียวหลัง describe block
    TestDB.teardown
  end

  it "สร้าง user ได้" do
    User.create!(email: "test@example.com")
    User.count.should eq(1)
  end

  it "ลบ user ได้" do
    user = User.create!(email: "test@example.com")
    user.destroy
    User.count.should eq(0)
  end
end

describe UserService do
  @@service : UserService? = nil

  before_all do
    @@service = UserService.new(TestDB.connection)
  end

  before_each do
    TestDB.truncate("users")
  end

  it "สร้าง user ใหม่" do
    service = @@service.not_nil!
    user = service.create_user("alice@example.com", "Alice")
    user.id.should_not be_nil
    user.email.should eq("alice@example.com")
  end
end
```

## let และ subject

```crystal
# Crystal ไม่มี let/subject built-in แต่เราจำลองได้

describe OrderCalculator do
  # ใช้ instance variables แทน let
  @calculator : OrderCalculator? = nil
  @items : Array(OrderItem)? = nil

  before_each do
    @calculator = OrderCalculator.new
    @items = [
      OrderItem.new(name: "Book", price: 299.0, quantity: 2),
      OrderItem.new(name: "Pen", price: 15.0, quantity: 5),
    ]
  end

  # Helper methods แทน let
  private def calculator
    @calculator.not_nil!
  end

  private def items
    @items.not_nil!
  end

  it "คำนวณ subtotal ถูกต้อง" do
    result = calculator.subtotal(items)
    result.should eq(673.0)  # 299*2 + 15*5
  end

  it "คำนวณ tax ถูกต้อง" do
    result = calculator.tax(673.0, 0.07)
    result.should be_close(47.11, 0.01)
  end

  it "คำนวณ total ถูกต้อง" do
    result = calculator.total(items, discount: 0.1, tax_rate: 0.07)
    result.should be_close(647.2, 0.1)
  end
end
```

## Crystal Spec Command

```bash
# รัน tests ทั้งหมด
crystal spec

# รัน test ไฟล์เดียว
crystal spec spec/calculator_spec.cr

# รัน test ที่ชื่อตรงกับ pattern
crystal spec --example "บวกเลข"

# รัน test ที่บรรทัดที่กำหนด
crystal spec spec/calculator_spec.cr:15

# รัน พร้อม verbose output
crystal spec --verbose

# รัน พร้อม progress dots
crystal spec --format progress

# รัน พร้อม tap output
crystal spec --format tap

# Fail fast (หยุดเมื่อ test แรกล้มเหลว)
crystal spec --fail-fast

# รัน ด้วย release mode (เร็วกว่า แต่ debug ยากกว่า)
crystal spec --release

# รัน test ใน file หลายไฟล์
crystal spec spec/user_spec.cr spec/calculator_spec.cr

# ดู output format ที่ใช้ได้
crystal spec --help
```

## Custom Matchers

```crystal
# เพิ่ม matchers ใน spec_helper.cr
module Spec
  # Custom matcher สำหรับ HTTP response
  struct HaveStatusCode
    def initialize(@expected : Int32)
    end

    def match(actual : HTTP::Client::Response)
      actual.status_code == @expected
    end

    def failure_message(actual : HTTP::Client::Response)
      "Expected status #{@expected} but got #{actual.status_code}"
    end

    def negative_failure_message(actual : HTTP::Client::Response)
      "Expected status NOT to be #{@expected}"
    end
  end

  def have_status(code : Int32)
    HaveStatusCode.new(code)
  end
end

# ใช้งาน custom matcher
describe "API Endpoints" do
  it "returns 200 for valid request" do
    response = HTTP::Client.get("http://localhost:3000/api/users")
    response.should have_status(200)
  end

  it "returns 404 for unknown route" do
    response = HTTP::Client.get("http://localhost:3000/unknown")
    response.should have_status(404)
  end
end
```

## Testing Exceptions

```crystal
describe "Exception Handling" do
  it "raises ArgumentError สำหรับ negative number" do
    expect_raises(ArgumentError) do
      Calculator.new.factorial(-1)
    end
  end

  it "raises ด้วย message ที่ถูกต้อง" do
    error = expect_raises(ArgumentError, "n must be >= 0") do
      Calculator.new.factorial(-1)
    end
    error.message.should eq("n must be >= 0")
  end

  it "ไม่ raises สำหรับ valid input" do
    # ถ้าไม่ต้องการ raise ใช้ normal assertion
    result = Calculator.new.factorial(5)
    result.should eq(120)
  end

  it "raises DivisionByZero" do
    expect_raises(Calculator::DivisionByZero, "ไม่สามารถหารด้วยศูนย์") do
      Calculator.new.divide(10.0, 0.0)
    end
  end
end
```

## Test ที่ครอบคลุมมากขึ้น

```crystal
# src/password_validator.cr
class PasswordValidator
  MIN_LENGTH = 8
  MAX_LENGTH = 128

  record Result,
    valid : Bool,
    errors : Array(String)

  def validate(password : String) : Result
    errors = [] of String

    if password.size < MIN_LENGTH
      errors << "รหัสผ่านต้องมีอย่างน้อย #{MIN_LENGTH} ตัวอักษร"
    end

    if password.size > MAX_LENGTH
      errors << "รหัสผ่านต้องไม่เกิน #{MAX_LENGTH} ตัวอักษร"
    end

    unless password.match(/[A-Z]/)
      errors << "ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว"
    end

    unless password.match(/[a-z]/)
      errors << "ต้องมีตัวพิมพ์เล็กอย่างน้อย 1 ตัว"
    end

    unless password.match(/[0-9]/)
      errors << "ต้องมีตัวเลขอย่างน้อย 1 ตัว"
    end

    unless password.match(/[!@#$%^&*]/)
      errors << "ต้องมีอักขระพิเศษ (!@#$%^&*) อย่างน้อย 1 ตัว"
    end

    Result.new(valid: errors.empty?, errors: errors)
  end
end
```

```crystal
# spec/password_validator_spec.cr
require "./spec_helper"
require "../src/password_validator"

describe PasswordValidator do
  @validator : PasswordValidator? = nil

  before_each do
    @validator = PasswordValidator.new
  end

  private def validator
    @validator.not_nil!
  end

  describe "#validate" do
    context "รหัสผ่านที่ถูกต้อง" do
      valid_passwords = [
        "Secure@123",
        "MyP@ssw0rd!",
        "Tr0ub4dor&3",
      ]

      valid_passwords.each do |pwd|
        it "ยอมรับ #{pwd}" do
          result = validator.validate(pwd)
          result.valid.should be_true
          result.errors.should be_empty
        end
      end
    end

    context "รหัสผ่านที่สั้นเกินไป" do
      it "ไม่ยอมรับรหัสผ่านที่สั้นกว่า 8 ตัว" do
        result = validator.validate("Ab@1")
        result.valid.should be_false
        result.errors.should contain("รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร")
      end
    end

    context "ไม่มีตัวพิมพ์ใหญ่" do
      it "ปฏิเสธรหัสผ่านที่ไม่มีตัวพิมพ์ใหญ่" do
        result = validator.validate("secure@123")
        result.valid.should be_false
        result.errors.should contain("ต้องมีตัวพิมพ์ใหญ่อย่างน้อย 1 ตัว")
      end
    end

    context "ไม่มีอักขระพิเศษ" do
      it "ปฏิเสธรหัสผ่านที่ไม่มีอักขระพิเศษ" do
        result = validator.validate("Secure1234")
        result.valid.should be_false
        result.errors.should contain("ต้องมีอักขระพิเศษ (!@#$%^&*) อย่างน้อย 1 ตัว")
      end
    end

    context "มีหลาย errors" do
      it "รายงาน errors ทั้งหมด" do
        result = validator.validate("weak")
        result.valid.should be_false
        result.errors.size.should be > 1
      end
    end
  end
end
```

## Pending Tests

```crystal
describe "Feature ที่ยังไม่พัฒนา" do
  it "feature X ที่กำลังพัฒนา" do
    pending "รอ implement"
  end

  pending "feature Y"

  it "feature Z" do
    skip "ข้าม test นี้ชั่วคราว"
  end
end
```

## แบบฝึกหัด

1. เขียน unit tests ครบถ้วนสำหรับ `Stack<T>` ที่ implement push, pop, peek, empty?, size
2. สร้าง `EmailValidator` พร้อม spec ที่ test valid และ invalid emails หลายรูปแบบ
3. เขียน spec สำหรับ `Fibonacci` ที่คำนวณ fibonacci ด้วย memoization
4. สร้าง custom matcher `be_valid_email` และใช้ใน spec

## สรุป

Crystal Spec สำหรับ Unit Testing:
- **describe/it**: จัดกลุ่ม tests ด้วย describe และเขียน test cases ด้วย it
- **context**: สร้าง sub-groups สำหรับ scenarios ต่างๆ
- **should/should_not**: matchers พื้นฐาน eq, contain, be_a, be_nil, match
- **before_each/after_each**: setup และ teardown
- **before_all/after_all**: รันครั้งเดียวต่อ describe block
- **expect_raises**: ทดสอบ exceptions
- **crystal spec**: command สำหรับรัน tests พร้อม options มากมาย
- **Custom matchers**: สร้าง matchers เพิ่มเติมตามความต้องการ
