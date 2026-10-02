# Part 168: Testing Best Practices ใน Crystal

## บทนำ

การเขียน tests ที่ดีไม่ใช่แค่การทำให้ tests ผ่าน แต่เกี่ยวกับการสร้าง test suite ที่ maintainable reliable และ meaningful

## Test Organization

### โครงสร้าง Directory

```
project/
├── src/
│   ├── models/
│   │   ├── user.cr
│   │   └── product.cr
│   ├── services/
│   │   ├── user_service.cr
│   │   └── order_service.cr
│   └── app.cr
└── spec/
    ├── spec_helper.cr
    ├── models/
    │   ├── user_spec.cr
    │   └── product_spec.cr
    ├── services/
    │   ├── user_service_spec.cr
    │   └── order_service_spec.cr
    ├── integration/
    │   ├── api_spec.cr
    │   └── checkout_spec.cr
    └── support/
        ├── factory.cr
        ├── helpers.cr
        └── matchers.cr
```

### spec_helper.cr ที่ดี

```crystal
# spec/spec_helper.cr
require "spec"

# Load all support files
require "./support/factory"
require "./support/helpers"
require "./support/matchers"
require "./support/test_database"

# Load all source files
require "../src/**"

# Global hooks
Spec.before_suite do
  TestDatabase.setup
  puts "\n🧪 Test Suite Started"
end

Spec.after_suite do
  TestDatabase.teardown
  puts "\n✓ Test Suite Completed"
end

# Configure test env
ENV["APP_ENV"] = "test"
```

## สิ่งที่ควร Test

### Test Pyramid

```
                    /\
                   /  \
                  / E2E \      <- น้อย (ช้า, แพง)
                 /--------\
                /Integration\   <- ปานกลาง
               /--------------\
              /   Unit Tests    \  <- เยอะ (เร็ว, ถูก)
             /--------------------\
```

### What to Test

```crystal
# 1. Business Logic ทุกอย่าง
describe PriceCalculator do
  it "คำนวณราคาพร้อม VAT" do
    result = PriceCalculator.new.with_vat(100.0)
    result.should eq(107.0)
  end
end

# 2. Validations ครบ
describe User do
  it "validate email format" do
    user = User.new(email: "not-email")
    user.valid?.should be_false
  end
end

# 3. Edge Cases
describe Divider do
  it "หารด้วยศูนย์" do
    expect_raises(ZeroDivisionError) { Divider.new.divide(5, 0) }
  end

  it "หาร 0 ด้วยจำนวนใดๆ = 0" do
    Divider.new.divide(0, 100).should eq(0)
  end
end

# 4. Public API ทุก method
# 5. Error paths (exceptions, error returns)
```

### อะไรที่ไม่ควร Test

```crystal
# 1. Implementation details (private methods)
# ไม่ดี - test private method โดยตรง
# user.__send__(:hash_password, "abc")  # ใน Crystal ทำไม่ได้ตรงๆ

# ดี - test ผ่าน public interface
it "password ถูก hash" do
  user = User.create(password: "plaintext")
  user.password_digest.should_not eq("plaintext")
end

# 2. Framework/Library code
# ไม่ควร test ว่า HTTP::Server ส่ง headers ถูกต้อง

# 3. Simple getters/setters
# อาจไม่จำเป็น เว้นแต่มี logic
struct Product
  getter name : String  # ไม่จำเป็นต้อง test getter นี้
  getter price : Float64

  def discounted_price(pct : Float64) : Float64  # ควร test
    price * (1 - pct)
  end
end
```

## Fast Tests

```crystal
# 1. หลีกเลี่ยง sleep ใน tests
# ไม่ดี
it "processes async task" do
  start_async_task
  sleep(1.0)  # รอ 1 วินาที - ช้า!
  result.should be_done
end

# ดี - ใช้ channel/fiber
it "processes async task" do
  channel = Channel(String).new
  spawn { channel.send(process()) }
  result = channel.receive_timeout(1.second)
  result.should_not be_nil
end

# 2. ใช้ in-memory database สำหรับ unit tests
class InMemoryUserRepo
  @@users : Hash(Int64, User) = {} of Int64 => User
  @@next_id = Atomic(Int64).new(1)

  def self.create(attrs) : User
    id = @@next_id.add(1)
    user = User.new(id: id, **attrs)
    @@users[id] = user
    user
  end

  def self.find(id : Int64) : User?
    @@users[id]?
  end

  def self.clear
    @@users.clear
  end
end

# 3. Avoid network calls ใน unit tests
# ใช้ WebMock หรือ stub

# 4. Parallelize test suites
# crystal spec --workers=4  (ถ้า Crystal รองรับ)
```

## Isolated Tests

```crystal
# ทุก test ต้องไม่พึ่ง state จาก test อื่น

# ไม่ดี - global state
@@count = 0

it "first test" do
  @@count += 1
  @@count.should eq(1)
end

it "second test" do
  @@count.should eq(0)  # อาจล้มเหลวถ้า tests รันต่อกัน
end

# ดี - reset state ในแต่ละ test
describe Counter do
  @counter : Counter? = nil

  before_each do
    @counter = Counter.new  # fresh instance ทุก test
  end

  it "starts at 0" do
    @counter.not_nil!.count.should eq(0)
  end

  it "increments" do
    c = @counter.not_nil!
    c.increment
    c.count.should eq(1)
  end
end
```

## Test Data Builders

```crystal
# spec/support/builders.cr

# Builder pattern สำหรับ test data
class UserBuilder
  property email : String = "user@example.com"
  property username : String = "user"
  property password : String = "Secure@123"
  property role : String = "user"
  property active : Bool = true

  def with_email(email : String) : self
    @email = email
    self
  end

  def with_role(role : String) : self
    @role = role
    self
  end

  def inactive : self
    @active = false
    self
  end

  def as_admin : self
    @role = "admin"
    self
  end

  def build : User
    User.new(
      email:    @email,
      username: @username,
      password: @password,
      role:     @role,
      active:   @active
    )
  end

  def create! : User
    user = build
    user.save!
    user
  end
end

class ProductBuilder
  @@counter = Atomic(Int32).new(0)

  property name : String
  property price : Float64 = 299.0
  property stock : Int32 = 10
  property category : String = "general"
  property active : Bool = true

  def initialize
    n = @@counter.add(1)
    @name = "Product #{n}"
  end

  def expensive : self
    @price = 9999.0
    self
  end

  def out_of_stock : self
    @stock = 0
    self
  end

  def in_category(cat : String) : self
    @category = cat
    self
  end

  def build : Product
    Product.new(
      name:     @name,
      price:    @price,
      stock:    @stock,
      category: @category,
      active:   @active
    )
  end
end

# Factory shortcuts
module Factory
  def self.user(**attrs) : User
    builder = UserBuilder.new
    attrs.each { |k, v| builder.send("#{k}=", v) }
    builder.build
  end

  def self.admin : User
    UserBuilder.new.as_admin.build
  end

  def self.product(**attrs) : Product
    builder = ProductBuilder.new
    attrs.each { |k, v| builder.send("#{k}=", v) }
    builder.build
  end

  def self.create_user(**attrs) : User
    user = self.user(**attrs)
    user.save!
    user
  end
end
```

### ใช้ Builders ใน Tests

```crystal
describe OrderService do
  it "สร้าง order สำหรับ active user" do
    user = Factory.create_user
    product = Factory.product(price: 599.0, stock: 5)

    service = OrderService.new
    order = service.create(user, [product])

    order.user_id.should eq(user.id)
    order.total.should eq(599.0)
  end

  it "ปฏิเสธ order จาก inactive user" do
    inactive = Factory.user(active: false)
    product = Factory.product

    expect_raises(OrderService::InactiveUserError) do
      OrderService.new.create(inactive, [product])
    end
  end

  it "ปฏิเสธ order เมื่อสินค้าหมด" do
    user = Factory.user
    out_of_stock = ProductBuilder.new.out_of_stock.build

    expect_raises(OrderService::OutOfStockError) do
      OrderService.new.create(user, [out_of_stock])
    end
  end
end
```

## Readable Test Names

```crystal
# ไม่ดี - ชื่อไม่บอกอะไร
it "test1" do ... end
it "works" do ... end
it "user" do ... end

# ดี - ชื่อบอก behavior ชัดเจน
it "คืนค่า nil เมื่อ user ไม่อยู่ใน database" do ... end
it "raise InvalidEmail เมื่อ email ไม่มี @ sign" do ... end
it "ส่ง welcome email หลัง registration สำเร็จ" do ... end

# ดีมาก - ชื่อเป็น living documentation
describe User do
  describe "#authenticate" do
    context "ด้วย valid credentials" do
      it "คืนค่า true" do ... end
    end

    context "ด้วย wrong password" do
      it "คืนค่า false" do ... end
      it "เพิ่ม failed_attempts" do ... end
      it "lock account หลัง 5 ครั้ง" do ... end
    end

    context "ด้วย locked account" do
      it "คืนค่า false แม้ password ถูกต้อง" do ... end
    end
  end
end
```

## AAA Pattern (Arrange-Act-Assert)

```crystal
it "คำนวณ order total พร้อม discount" do
  # Arrange - ตั้งค่า prerequisites
  user = Factory.create_user
  product = Factory.product(price: 1000.0)
  discount = Discount.new(code: "SAVE20", percentage: 20)

  # Act - รัน action ที่ต้องการทดสอบ
  order = OrderService.new.create(
    user:     user,
    items:    [product],
    discount: discount
  )

  # Assert - ตรวจสอบผลลัพธ์
  order.total.should eq(800.0)
  order.discount_amount.should eq(200.0)
  order.status.should eq("pending")
end
```

## Test Helpers

```crystal
# spec/support/helpers.cr
module Helpers
  # Helper สำหรับ authenticated requests
  def with_auth_token(user : User)
    token = JWT.encode({user_id: user.id, exp: 1.hour.from_now.to_unix})
    HTTP::Headers{"Authorization" => "Bearer #{token}"}
  end

  # Helper สำหรับ JSON requests
  def json_headers
    HTTP::Headers{"Content-Type" => "application/json"}
  end

  # Helper สร้าง paginated response
  def paginated_response(data : Array, total : Int32, page : Int32, per_page : Int32)
    {
      data:     data,
      total:    total,
      page:     page,
      per_page: per_page,
      pages:    (total.to_f / per_page).ceil.to_i
    }
  end

  # Helper check JSON response
  def parse_json_response(response : HTTP::Client::Response)
    JSON.parse(response.body)
  end

  def assert_json_error(response : HTTP::Client::Response, field : String)
    body = parse_json_response(response)
    body["errors"][field]?.should_not be_nil
  end
end

# Include ใน tests
describe "API" do
  include Helpers

  it "GET /users returns 200" do
    user = Factory.create_user
    response = client.get("/users", headers: with_auth_token(user))
    response.status_code.should eq(200)
  end
end
```

## แบบฝึกหัด

1. refactor test suite ที่ใช้ magic numbers และ strings ให้ใช้ builders แทน
2. สร้าง custom matchers สำหรับ domain-specific assertions เช่น `be_valid_order` หรือ `have_status`
3. ระบุ tests ใน test suite ของคุณที่ test implementation details แล้ว refactor ให้ test behavior แทน
4. วัดเวลา test suite แล้ว optimize tests ที่ช้าที่สุด

## สรุป

Testing Best Practices ใน Crystal:
- **Organization**: โครงสร้าง directories ที่ชัดเจน mirrors source code
- **What to test**: business logic, validations, edge cases, public API
- **Fast tests**: หลีกเลี่ยง sleep, network calls, ใช้ in-memory storage
- **Isolation**: ทุก test เป็น independent ใช้ before_each reset state
- **Data builders**: Factory/Builder pattern สำหรับ test data ที่ readable
- **AAA pattern**: Arrange-Act-Assert ทำให้ tests อ่านง่าย
- **Readable names**: ชื่อ tests เป็น documentation

Test suite ที่ดีช่วยให้ refactor code ได้อย่างมั่นใจ
