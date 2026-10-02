# Part 163: Mocking และ Stubbing ใน Crystal

## บทนำ

Mocking และ Stubbing เป็นเทคนิคการทดสอบที่ช่วยให้เราแทนที่ dependencies ของ code ที่ทดสอบด้วย objects จำลอง ทำให้ tests รันเร็วขึ้นและเป็น deterministic

## แนวคิดพื้นฐาน

```
Test Double Types:
- Stub:    คืนค่าที่กำหนดไว้ล่วงหน้า
- Mock:    ตรวจสอบว่า method ถูกเรียกอย่างถูกต้อง
- Spy:     บันทึกการเรียก method
- Fake:    implement จริงแต่ simplified (เช่น in-memory database)
- Double:  replacement object ทั่วไป
```

## Method Stubs

```crystal
# src/email_service.cr
abstract class EmailService
  abstract def send_welcome(email : String, name : String) : Bool
  abstract def send_password_reset(email : String, token : String) : Bool
  abstract def send_notification(email : String, message : String) : Bool
end

# src/real_email_service.cr
class RealEmailService < EmailService
  def send_welcome(email : String, name : String) : Bool
    # ส่ง email จริงๆ ผ่าน SMTP
    puts "Sending welcome email to #{email}"
    true
  end

  def send_password_reset(email : String, token : String) : Bool
    puts "Sending password reset to #{email}"
    true
  end

  def send_notification(email : String, message : String) : Bool
    puts "Sending notification to #{email}"
    true
  end
end

# spec/support/stub_email_service.cr
class StubEmailService < EmailService
  property sent_emails : Array({type: String, email: String}) = [] of NamedTuple(type: String, email: String)
  property should_fail : Bool = false

  def send_welcome(email : String, name : String) : Bool
    return false if should_fail
    @sent_emails << {type: "welcome", email: email}
    true
  end

  def send_password_reset(email : String, token : String) : Bool
    return false if should_fail
    @sent_emails << {type: "reset", email: email}
    true
  end

  def send_notification(email : String, message : String) : Bool
    return false if should_fail
    @sent_emails << {type: "notification", email: email}
    true
  end

  def welcome_sent_to?(email : String) : Bool
    @sent_emails.any? { |e| e[:type] == "welcome" && e[:email] == email }
  end

  def reset_sent_to?(email : String) : Bool
    @sent_emails.any? { |e| e[:type] == "reset" && e[:email] == email }
  end

  def total_sent : Int32
    @sent_emails.size
  end
end
```

### ใช้ Stub ใน Tests

```crystal
# spec/user_service_spec.cr
require "./spec_helper"

describe UserService do
  @email_stub : StubEmailService? = nil
  @service : UserService? = nil

  before_each do
    @email_stub = StubEmailService.new
    @service = UserService.new(
      db: TestDB.connection,
      email_service: @email_stub.not_nil!
    )
    TestDB.truncate("users")
  end

  private def email_stub
    @email_stub.not_nil!
  end

  private def service
    @service.not_nil!
  end

  describe "#register" do
    it "ส่ง welcome email หลัง register สำเร็จ" do
      user = service.register("alice@example.com", "alice", "Secure@123")

      email_stub.welcome_sent_to?("alice@example.com").should be_true
      email_stub.total_sent.should eq(1)
    end

    it "ไม่ส่ง email เมื่อ registration ล้มเหลว" do
      expect_raises(UserService::ValidationError) do
        service.register("invalid-email", "", "weak")
      end

      email_stub.total_sent.should eq(0)
    end

    it "จัดการเมื่อ email service ล้มเหลว" do
      email_stub.should_fail = true

      # Registration ควรสำเร็จแม้ email ส่งไม่ได้
      user = service.register("alice@example.com", "alice", "Secure@123")
      user.should_not be_nil

      # แต่ควร log ว่าส่ง email ไม่ได้
    end
  end

  describe "#request_password_reset" do
    it "ส่ง password reset email" do
      service.register("alice@example.com", "alice", "Secure@123")
      email_stub.sent_emails.clear

      service.request_password_reset("alice@example.com")

      email_stub.reset_sent_to?("alice@example.com").should be_true
    end

    it "ไม่แจ้งเมื่อ email ไม่มีในระบบ (security)" do
      # ไม่ควร raise error เพื่อป้องกัน user enumeration
      service.request_password_reset("unknown@example.com")
      email_stub.total_sent.should eq(0)
    end
  end
end
```

## Mock Objects

```crystal
# Mock ที่ตรวจสอบว่า method ถูกเรียก
class MockPaymentGateway
  property expected_calls : Array({method: String, args: Array(String)}) = [] of NamedTuple(method: String, args: Array(String))
  property actual_calls : Array({method: String, args: Array(String)}) = [] of NamedTuple(method: String, args: Array(String))
  property responses : Hash(String, NamedTuple(success: Bool, charge_id: String?)) = {} of String => NamedTuple(success: Bool, charge_id: String?)

  # ตั้งค่า expected behavior
  def expect_charge(amount : String, currency : String, response : NamedTuple(success: Bool, charge_id: String?))
    expected_calls << {method: "charge", args: [amount, currency]}
    responses["charge:#{amount}:#{currency}"] = response
  end

  # Implementation
  def charge(amount : Int32, currency : String, token : String) : NamedTuple(success: Bool, charge_id: String?)
    actual_calls << {method: "charge", args: [amount.to_s, currency]}
    responses["charge:#{amount}:#{currency}"]? || {success: false, charge_id: nil}
  end

  # Verify expectations
  def verify!
    expected_calls.each_with_index do |expected, i|
      actual = actual_calls[i]?
      raise "Expected call #{i+1}: #{expected[:method]}(#{expected[:args].join(", ")}) but got #{actual ? "#{actual[:method]}(#{actual[:args].join(", ")})" : "nothing"}" unless actual

      unless actual[:method] == expected[:method] && actual[:args] == expected[:args]
        raise "Call #{i+1} mismatch: expected #{expected[:method]}(#{expected[:args].join(", ")}) got #{actual[:method]}(#{actual[:args].join(", ")})"
      end
    end

    unless actual_calls.size == expected_calls.size
      raise "Expected #{expected_calls.size} calls but got #{actual_calls.size}"
    end
  end

  def reset
    @expected_calls.clear
    @actual_calls.clear
    @responses.clear
  end
end
```

### ใช้งาน Mock

```crystal
describe CheckoutService do
  @payment_mock : MockPaymentGateway? = nil
  @service : CheckoutService? = nil

  before_each do
    @payment_mock = MockPaymentGateway.new
    @service = CheckoutService.new(payment: @payment_mock.not_nil!)
  end

  after_each do
    @payment_mock.try(&.verify!)
  end

  private def mock
    @payment_mock.not_nil!
  end

  it "charges the correct amount" do
    # ตั้งค่า expectation
    mock.expect_charge(
      "1000", "thb",
      {success: true, charge_id: "ch_12345"}
    )

    order = Order.new(total: 1000.0, currency: "thb")
    result = @service.not_nil!.process(order, token: "tok_test")

    result.success?.should be_true
    result.charge_id.should eq("ch_12345")
    # verify! จะถูกเรียกใน after_each
  end
end
```

## Spy Objects

```crystal
# Spy บันทึกทุก call โดยไม่เปลี่ยน behavior
class SpyLogger
  property calls : Array({level: String, message: String, context: Hash(String, String)}) = [] of NamedTuple(level: String, message: String, context: Hash(String, String))

  def info(message : String, **context)
    @calls << {level: "info", message: message, context: context.to_h.transform_values(&.to_s)}
  end

  def warn(message : String, **context)
    @calls << {level: "warn", message: message, context: context.to_h.transform_values(&.to_s)}
  end

  def error(message : String, **context)
    @calls << {level: "error", message: message, context: context.to_h.transform_values(&.to_s)}
  end

  def logged?(level : String, message_pattern : String) : Bool
    @calls.any? { |c|
      c[:level] == level && c[:message].includes?(message_pattern)
    }
  end

  def error_count : Int32
    @calls.count { |c| c[:level] == "error" }
  end

  def last_error : NamedTuple(level: String, message: String, context: Hash(String, String))?
    @calls.reverse.find { |c| c[:level] == "error" }
  end

  def clear
    @calls.clear
  end
end

# ใช้งาน Spy
describe "Authentication Service" do
  @spy_logger : SpyLogger? = nil
  @auth_service : AuthService? = nil

  before_each do
    @spy_logger = SpyLogger.new
    @auth_service = AuthService.new(logger: @spy_logger.not_nil!)
  end

  it "log เมื่อ login สำเร็จ" do
    auth = @auth_service.not_nil!
    logger = @spy_logger.not_nil!

    user = create_test_user
    auth.login(user.email, "correct_password")

    logger.logged?("info", "login").should be_true
    logger.error_count.should eq(0)
  end

  it "log เมื่อ login ล้มเหลว" do
    auth = @auth_service.not_nil!
    logger = @spy_logger.not_nil!

    auth.login("unknown@example.com", "wrong_pass")

    logger.logged?("warn", "failed login").should be_true
  end

  it "log error สำหรับ database error" do
    # Mock database ให้ throw error
    auth = @auth_service.not_nil!
    logger = @spy_logger.not_nil!

    auth.login("db_error@trigger.com", "password") rescue nil

    logger.error_count.should be > 0
  end
end
```

## Testing with Dependencies

```crystal
# src/user_repository.cr
class UserRepository
  def initialize(@db : DB::Database)
  end

  def find_by_email(email : String) : User?
    # ดึงจาก database
  end

  def create(attrs : Hash) : User
    # บันทึกลง database
  end

  def update(id : Int64, attrs : Hash) : Bool
    # อัปเดต database
  end
end

# src/user_service.cr
class UserService
  def initialize(
    @repo : UserRepository,
    @email : EmailService,
    @cache : CacheService,
    @logger : Logger
  )
  end

  def get_user(id : Int64) : User?
    # ลอง cache ก่อน
    if cached = @cache.get("user:#{id}")
      return User.from_json(cached)
    end

    # ดึงจาก db
    user = @repo.find_by_id(id)
    if user
      @cache.set("user:#{id}", user.to_json, ttl: 300)
      @logger.info("User loaded", id: id.to_s)
    end
    user
  end
end
```

```crystal
# spec ด้วย Fake Repository
class FakeUserRepository
  property users : Hash(Int64, User) = {} of Int64 => User
  @next_id : Int64 = 1

  def find_by_id(id : Int64) : User?
    @users[id]?
  end

  def find_by_email(email : String) : User?
    @users.values.find { |u| u.email == email }
  end

  def create(attrs : Hash) : User
    user = User.new(id: @next_id, **attrs)
    @users[@next_id] = user
    @next_id += 1
    user
  end

  def update(id : Int64, attrs : Hash) : Bool
    return false unless @users[id]?
    @users[id] = @users[id].merge(attrs)
    true
  end

  def count : Int32
    @users.size
  end

  def clear
    @users.clear
    @next_id = 1
  end
end

class FakeCacheService
  property store : Hash(String, {value: String, expires_at: Time}) = {} of String => NamedTuple(value: String, expires_at: Time)

  def get(key : String) : String?
    entry = @store[key]?
    return nil unless entry
    return nil if Time.utc > entry[:expires_at]
    entry[:value]
  end

  def set(key : String, value : String, ttl : Int32 = 300)
    @store[key] = {value: value, expires_at: Time.utc + ttl.seconds}
  end

  def delete(key : String)
    @store.delete(key)
  end

  def clear
    @store.clear
  end

  def size : Int32
    @store.size
  end
end

# Tests ที่ใช้ Fakes
describe UserService do
  @repo : FakeUserRepository? = nil
  @email : StubEmailService? = nil
  @cache : FakeCacheService? = nil
  @logger : SpyLogger? = nil
  @service : UserService? = nil

  before_each do
    @repo = FakeUserRepository.new
    @email = StubEmailService.new
    @cache = FakeCacheService.new
    @logger = SpyLogger.new
    @service = UserService.new(
      repo:   @repo.not_nil!,
      email:  @email.not_nil!,
      cache:  @cache.not_nil!,
      logger: @logger.not_nil!
    )
  end

  describe "#get_user" do
    it "คืนค่า user จาก cache" do
      user = User.new(id: 1_i64, email: "alice@example.com")
      @cache.not_nil!.set("user:1", user.to_json)

      result = @service.not_nil!.get_user(1_i64)

      result.should_not be_nil
      result.not_nil!.email.should eq("alice@example.com")
      @repo.not_nil!.count.should eq(0)  # ไม่ได้ดึงจาก db
    end

    it "ดึงจาก repository เมื่อไม่มีใน cache" do
      @repo.not_nil!.create({"email" => "alice@example.com"})

      result = @service.not_nil!.get_user(1_i64)

      result.should_not be_nil
      @cache.not_nil!.store.has_key?("user:1").should be_true  # เก็บลง cache
    end

    it "log เมื่อโหลด user" do
      @repo.not_nil!.create({"email" => "alice@example.com"})
      @service.not_nil!.get_user(1_i64)

      @logger.not_nil!.logged?("info", "User loaded").should be_true
    end
  end
end
```

## Crystal Mock Library - mocks.cr

```yaml
# shard.yml
dependencies:
  mocks:
    github: waterlink/mocks.cr
    version: ~> 0.0
```

```crystal
# ใช้ mocks.cr library
require "mocks"

# สร้าง mock class
mock ExternalAPIClient do
  mock_method get : String
  mock_method post : Bool
  mock_method delete : Bool
end

describe "Service with Mock" do
  it "calls the API correctly" do
    api_mock = mock(ExternalAPIClient)

    # ตั้งค่า stub
    allow(api_mock).to receive(:get).and_return(%({"data": "value"}))
    allow(api_mock).to receive(:post).and_return(true)

    service = MyService.new(api: api_mock)
    result = service.fetch_data

    # ตรวจสอบ calls
    expect(api_mock).to have_received(:get)
    expect(api_mock).to have_received(:post).exactly(1).times

    result.should_not be_nil
  end
end
```

## Time Stubbing

```crystal
# ปัญหา: tests ที่พึ่งพา Time.utc ทดสอบยาก
class SessionService
  SESSION_EXPIRY = 24.hours

  def create_session(user_id : Int64) : String
    token = generate_token
    expiry = Time.utc + SESSION_EXPIRY
    store_session(token, user_id, expiry)
    token
  end

  def valid_session?(token : String) : Bool
    session = find_session(token)
    return false unless session
    Time.utc < session.expires_at
  end
end

# แก้ด้วย injectable time
class TimeProvider
  def self.now : Time
    Time.utc
  end
end

class FakeTimeProvider
  class_property current_time : Time = Time.utc

  def self.now : Time
    current_time
  end

  def self.advance(duration : Time::Span)
    current_time += duration
  end

  def self.reset
    current_time = Time.utc
  end
end

class SessionService
  def initialize(@time : TimeProvider.class = TimeProvider)
  end

  def create_session(user_id : Int64) : String
    token = generate_token
    expiry = @time.now + SESSION_EXPIRY
    store_session(token, user_id, expiry)
    token
  end

  def valid_session?(token : String) : Bool
    session = find_session(token)
    return false unless session
    @time.now < session.expires_at
  end
end

# Tests ด้วย FakeTimeProvider
describe SessionService do
  before_each do
    FakeTimeProvider.reset
    FakeTimeProvider.current_time = Time.utc(2024, 1, 1, 12, 0, 0)
  end

  it "session ยังไม่หมดอายุ" do
    service = SessionService.new(FakeTimeProvider)
    token = service.create_session(1_i64)

    service.valid_session?(token).should be_true
  end

  it "session หมดอายุหลัง 24 ชั่วโมง" do
    service = SessionService.new(FakeTimeProvider)
    token = service.create_session(1_i64)

    FakeTimeProvider.advance(25.hours)

    service.valid_session?(token).should be_false
  end
end
```

## แบบฝึกหัด

1. สร้าง `FakeDatabase` ที่ implement ท database interface โดยใช้ Hash ใน memory แทน real database
2. เขียน Spy สำหรับ HTTP client ที่บันทึก requests ทุกตัว แล้วตรวจสอบใน test ว่า endpoint ที่ถูกต้องถูกเรียก
3. สร้าง Mock สำหรับ SMS service พร้อม verify expectations ว่า message ถูกส่งพร้อม content ที่ถูกต้อง
4. ทดสอบ rate limiter ด้วย fake time provider ที่ควบคุมเวลาได้

## สรุป

Mocking และ Stubbing ใน Crystal:
- **Stub**: คืนค่าที่กำหนด ใช้เมื่อต้องการควบคุม behavior ของ dependency
- **Mock**: ตรวจสอบว่า method ถูกเรียกตาม expectations
- **Spy**: บันทึกทุก call สำหรับ verification ภายหลัง
- **Fake**: simplified implementation ที่ใช้จริงในการทดสอบ (เช่น in-memory db)
- **Dependency Injection**: inject dependencies เพื่อให้ swap ได้ใน tests
- **Interface-based Design**: ใช้ abstract classes ทำให้ mock ง่ายขึ้น

การใช้ test doubles ช่วยให้ unit tests เร็ว deterministic และ isolated
