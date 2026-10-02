# Part 170: Test-Driven Development (TDD) ใน Crystal

## บทนำ

TDD (Test-Driven Development) คือกระบวนการพัฒนาที่เขียน test ก่อน แล้วจึงเขียน code ที่ทำให้ test ผ่าน หลักการนี้ช่วยออกแบบ API ที่ดีและได้ code ที่มี test coverage สูง

## TDD Cycle: Red → Green → Refactor

```
1. RED:    เขียน test ที่ fail
           ↓
2. GREEN:  เขียน code ขั้นต่ำที่สุดให้ test ผ่าน
           ↓
3. REFACTOR: ปรับปรุง code โดยที่ tests ยังผ่าน
           ↓
           กลับไป 1.
```

## TDD Example: Building a Calculator

### Iteration 1: Red

```crystal
# spec/calculator_spec.cr - เริ่มด้วย test ที่ fail
require "./spec_helper"

describe Calculator do
  it "บวกเลขสองตัว" do
    calc = Calculator.new
    calc.add(2, 3).should eq(5)
  end
end
```

รัน: `crystal spec` → FAIL (Calculator ยังไม่มี)

### Iteration 1: Green

```crystal
# src/calculator.cr - เขียน code น้อยที่สุด
class Calculator
  def add(a : Number, b : Number)
    a + b
  end
end
```

รัน: `crystal spec` → PASS ✓

### Iteration 2: Red (เพิ่ม feature)

```crystal
# เพิ่ม test ใหม่
describe Calculator do
  it "บวกเลขสองตัว" do
    Calculator.new.add(2, 3).should eq(5)
  end

  it "ลบเลขสองตัว" do
    Calculator.new.subtract(10, 4).should eq(6)
  end

  it "คูณเลขสองตัว" do
    Calculator.new.multiply(3, 4).should eq(12)
  end

  it "หารเลขสองตัว" do
    Calculator.new.divide(10.0, 2.0).should eq(5.0)
  end
end
```

รัน → FAIL (subtract, multiply, divide ไม่มี)

### Iteration 2: Green

```crystal
class Calculator
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
    a / b
  end
end
```

รัน → PASS ✓

### Iteration 3: Red (Error handling)

```crystal
describe Calculator do
  # ... tests ก่อนหน้า ...

  it "raise เมื่อหารด้วยศูนย์" do
    expect_raises(Calculator::DivisionByZero) do
      Calculator.new.divide(5.0, 0.0)
    end
  end
end
```

### Iteration 3: Green

```crystal
class Calculator
  class DivisionByZero < Exception
    def initialize
      super("ไม่สามารถหารด้วยศูนย์ได้")
    end
  end

  # ... methods อื่น ...

  def divide(a : Float64, b : Float64) : Float64
    raise DivisionByZero.new if b == 0.0
    a / b
  end
end
```

### Iteration 4: Refactor

```crystal
# หลัง tests ผ่านแล้ว refactor ให้ดีขึ้น
class Calculator
  class DivisionByZero < Exception
    def initialize(numerator : Float64)
      super("ไม่สามารถหาร #{numerator} ด้วยศูนย์ได้")
    end
  end

  def add(a : Number, b : Number) : typeof(a + b)
    a + b
  end

  def subtract(a : Number, b : Number) : typeof(a - b)
    a - b
  end

  def multiply(a : Number, b : Number) : typeof(a * b)
    a * b
  end

  def divide(a : Float64, b : Float64) : Float64
    raise DivisionByZero.new(a) if b.zero?
    a / b
  end

  # เพิ่ม operations ที่มีประโยชน์
  def power(base : Float64, exp : Int32) : Float64
    base ** exp
  end

  def square_root(n : Float64) : Float64
    raise ArgumentError.new("Cannot sqrt negative number") if n < 0
    Math.sqrt(n)
  end
end
```

## TDD สำหรับ REST API

### Step 1: เขียน failing tests

```crystal
# spec/api/users_api_spec.cr
require "../spec_helper"

describe "Users API" do
  before_each { TestDB.truncate("users") }

  describe "GET /api/users" do
    it "คืนค่า empty list" do
      response = client.get("/api/users")
      response.status_code.should eq(200)
      JSON.parse(response.body)["data"].as_a.should be_empty
    end

    it "คืนค่า users ทั้งหมด" do
      TestDB.insert_user("alice@example.com")
      TestDB.insert_user("bob@example.com")

      response = client.get("/api/users")
      body = JSON.parse(response.body)
      body["data"].as_a.size.should eq(2)
    end
  end

  describe "POST /api/users" do
    it "สร้าง user ใหม่" do
      response = client.post(
        "/api/users",
        headers: json_headers,
        body: {email: "new@example.com", username: "newuser", password: "Secure@123"}.to_json
      )

      response.status_code.should eq(201)
      data = JSON.parse(response.body)
      data["id"].as_i64.should be > 0
      data["email"].as_s.should eq("new@example.com")
    end

    it "คืนค่า 422 สำหรับ invalid data" do
      response = client.post(
        "/api/users",
        headers: json_headers,
        body: {email: "bad", username: ""}.to_json
      )

      response.status_code.should eq(422)
      errors = JSON.parse(response.body)["errors"]
      errors["email"]?.should_not be_nil
    end
  end
end
```

### Step 2: ขั้นต่ำที่สุดให้ pass

```crystal
# src/handlers/users_handler.cr
class UsersHandler
  include HTTP::Handler

  def call(context : HTTP::Server::Context)
    path = context.request.path
    method = context.request.method

    case {method, path}
    when {"GET", "/api/users"}
      handle_list(context)
    when {"POST", "/api/users"}
      handle_create(context)
    else
      call_next(context)
    end
  end

  private def handle_list(ctx)
    users = UserRepository.new(DB.connection).all
    ctx.response.content_type = "application/json"
    ctx.response.status_code = 200
    ctx.response.print({data: users, total: users.size}.to_json)
  end

  private def handle_create(ctx)
    body = JSON.parse(ctx.request.body.try(&.gets_to_end) || "{}")

    email = body["email"]?.try(&.as_s) || ""
    username = body["username"]?.try(&.as_s) || ""
    password = body["password"]?.try(&.as_s) || ""

    user = User.new(email: email, username: username)
    user.password = password

    if user.valid?
      user.save!
      ctx.response.content_type = "application/json"
      ctx.response.status_code = 201
      ctx.response.print(user.to_json)
    else
      ctx.response.content_type = "application/json"
      ctx.response.status_code = 422
      ctx.response.print({errors: user.errors}.to_json)
    end
  end
end
```

### Step 3: Refactor

```crystal
# Refactor: แยก concerns ออก

# src/handlers/users_handler.cr
class UsersHandler
  include HTTP::Handler

  def initialize(@service : UserService)
  end

  def call(context : HTTP::Server::Context)
    case context
    in {GET, /^\/api\/users$/}
      list_users(context)
    in {POST, /^\/api\/users$/}
      create_user(context)
    in {GET, /^\/api\/users\/(\d+)$/}
      show_user(context, $~[1].to_i64)
    else
      call_next(context)
    end
  end

  private def list_users(ctx)
    page = ctx.request.query_params["page"]?.try(&.to_i) || 1
    per_page = ctx.request.query_params["per_page"]?.try(&.to_i) || 20

    result = @service.list(page: page, per_page: per_page)
    json_response(ctx, 200, result.to_json)
  end

  private def create_user(ctx)
    params = parse_json(ctx)
    result = @service.create(params)

    if result.success?
      json_response(ctx, 201, result.user.to_json)
    else
      json_response(ctx, 422, {errors: result.errors}.to_json)
    end
  end

  private def json_response(ctx, status, body)
    ctx.response.content_type = "application/json"
    ctx.response.status_code = status
    ctx.response.print(body)
  end

  private def parse_json(ctx) : Hash(String, JSON::Any)
    body = ctx.request.body.try(&.gets_to_end) || "{}"
    JSON.parse(body).as_h
  rescue
    {} of String => JSON::Any
  end
end
```

## TDD กับ Outside-In Development

Outside-In TDD เริ่มจาก high-level (acceptance/integration) tests ลงไปถึง unit tests

```crystal
# Level 1: Acceptance Test (ล้มเหลวก่อน)
describe "User can reset password" do
  it "ได้รับ email พร้อม reset link" do
    user = Factory.create_user(email: "alice@example.com")
    email_spy = SpyEmailService.new
    service = PasswordResetService.new(email: email_spy)

    service.request_reset("alice@example.com")

    email_spy.reset_email_sent_to?("alice@example.com").should be_true
  end
end

# Level 2: Service Unit Test
describe PasswordResetService do
  describe "#request_reset" do
    it "สร้าง reset token" do
      user = Factory.create_user
      service = PasswordResetService.new(repo: FakeRepo.new([user]))

      service.request_reset(user.email)

      FakeRepo.last_token.should_not be_nil
    end

    it "token หมดอายุหลัง 1 ชั่วโมง" do
      user = Factory.create_user
      service = PasswordResetService.new(repo: FakeRepo.new([user]))

      service.request_reset(user.email)

      FakeRepo.last_token_expiry.should be_close(
        1.hour.from_now,
        within: 1.minute
      )
    end
  end
end

# Level 3: Repository Unit Test
describe UserRepository do
  describe "#create_reset_token" do
    it "บันทึก token ลง database" do
      user = TestDB.insert_user("alice@example.com")
      repo = UserRepository.new(TestDB.connection)

      token = repo.create_reset_token(user.id, expires_at: 1.hour.from_now)

      token.should_not be_empty
      token.size.should eq(64)
    end
  end
end
```

## Triangulation in TDD

```crystal
# Triangulation: ใช้หลาย test cases เพื่อ force generalization
describe "Prime checker" do
  # Test 1: 2 เป็น prime
  it "2 เป็น prime" do
    PrimeChecker.prime?(2).should be_true
  end

  # Implementation ง่ายมาก:
  # def prime?(n) = n == 2

  # Test 2: 3 เป็น prime
  it "3 เป็น prime" do
    PrimeChecker.prime?(3).should be_true
  end

  # Implementation อาจเป็น: def prime?(n) = n == 2 || n == 3
  # แต่ต้องทำให้ general ขึ้น

  # Test 3: 4 ไม่เป็น prime
  it "4 ไม่เป็น prime" do
    PrimeChecker.prime?(4).should be_false
  end

  # ตอนนี้ต้อง implement จริงๆ แล้ว
  # def prime?(n)
  #   return false if n < 2
  #   (2...n).none? { |i| n % i == 0 }
  # end

  # Test 4: 1 ไม่เป็น prime
  it "1 ไม่เป็น prime" do
    PrimeChecker.prime?(1).should be_false
  end

  # Test 5: negative ไม่เป็น prime
  it "negative numbers ไม่เป็น prime" do
    PrimeChecker.prime?(-5).should be_false
  end

  # Test 6: large prime
  it "97 เป็น prime" do
    PrimeChecker.prime?(97).should be_true
  end
end

# Implementation สุดท้าย
class PrimeChecker
  def self.prime?(n : Int32) : Bool
    return false if n < 2
    return true if n == 2
    return false if n.even?
    (3..Math.sqrt(n.to_f).to_i).step(2).none? { |i| n % i == 0 }
  end
end
```

## TDD Anti-patterns

```crystal
# Anti-pattern 1: Test ที่ขึ้นกับ order
# ไม่ดี
@@shared_user : User? = nil

it "สร้าง user" do
  @@shared_user = User.create!(email: "alice@example.com")
  @@shared_user.should_not be_nil
end

it "อัปเดต user" do
  # ขึ้นกับ test ก่อนหน้า!
  @@shared_user.not_nil!.update(name: "Alice")
end

# ดี - tests เป็น independent
describe User do
  @user : User? = nil

  before_each do
    @user = User.create!(email: "alice@example.com")
  end

  it "สร้าง user" do
    @user.not_nil!.id.should_not be_nil
  end

  it "อัปเดต user" do
    @user.not_nil!.update(name: "Alice")
    @user.not_nil!.name.should eq("Alice")
  end
end

# Anti-pattern 2: Test ที่ test implementation
# ไม่ดี
it "เรียก bcrypt เมื่อบันทึก password" do
  # test ว่า bcrypt ถูกเรียกใช้
  # นี่คือ implementation detail
end

# ดี - test behavior
it "password ถูก hash เมื่อบันทึก" do
  user = User.create!(email: "alice@example.com", password: "plaintext")
  user.password_digest.should_not eq("plaintext")
  user.verify_password("plaintext").should be_true
end
```

## แบบฝึกหัด

1. ใช้ TDD สร้าง `StringCalculator` ที่รับ string เช่น "1,2,3" และคืนผลรวม
2. ใช้ Outside-In TDD สร้าง blog post API ตั้งแต่ acceptance test ลงไปถึง repository
3. ทำ TDD cycle สำหรับ `LRU Cache` ที่มี capacity จำกัด
4. เขียน code โดยใช้ TDD: rate limiter ที่อนุญาต N requests ต่อ minute

## สรุป

TDD ใน Crystal:
- **Red-Green-Refactor**: cycle สำคัญของ TDD
- **Small steps**: เขียน test ทีละ case ไม่กระโดดข้าม
- **Minimum code**: เขียน code น้อยที่สุดให้ test ผ่าน
- **Refactor freely**: เมื่อ tests ผ่านแล้ว refactor ได้มั่นใจ
- **Outside-In**: เริ่มจาก high-level ลง low-level
- **Triangulation**: หลาย test cases บังคับ general implementation

TDD ทำให้ได้ design ที่ดี เพราะ testable code มักเป็น well-designed code
