# Part 162: Integration Testing ใน Crystal

## บทนำ

Integration Testing ทดสอบ components หลายอย่างทำงานร่วมกัน เช่น HTTP endpoints, database operations, และ external services ต่างจาก unit testing ที่ทดสอบ component เดียวในแบบ isolation

## Testing HTTP Endpoints

### ตั้งค่า Test Server

```crystal
# spec/spec_helper.cr
require "spec"
require "http/client"
require "../src/app"

# Start test server
TEST_PORT = 8765

def start_test_server
  server = HTTP::Server.new(TestApp.handler)
  spawn { server.listen("127.0.0.1", TEST_PORT) }
  sleep(0.1)  # รอ server พร้อม
  server
end

BASE_URL = "http://127.0.0.1:#{TEST_PORT}"
```

### HTTP Client Tests

```crystal
# spec/integration/api_spec.cr
require "../spec_helper"

describe "Users API" do
  @@server : HTTP::Server? = nil
  @@client : HTTP::Client? = nil

  before_all do
    @@server = start_test_server
    @@client = HTTP::Client.new("127.0.0.1", TEST_PORT)
  end

  after_all do
    @@server.try(&.close)
    @@client.try(&.close)
  end

  before_each do
    TestDB.truncate("users")
  end

  private def client
    @@client.not_nil!
  end

  describe "GET /api/users" do
    it "คืนค่า 200 พร้อม empty array เมื่อไม่มี users" do
      response = client.get("/api/users")
      response.status_code.should eq(200)

      body = JSON.parse(response.body)
      body["data"].as_a.should be_empty
      body["total"].as_i.should eq(0)
    end

    it "คืนค่า users ทั้งหมด" do
      TestDB.insert_user(email: "alice@example.com")
      TestDB.insert_user(email: "bob@example.com")

      response = client.get("/api/users")
      response.status_code.should eq(200)

      body = JSON.parse(response.body)
      body["data"].as_a.size.should eq(2)
      body["total"].as_i.should eq(2)
    end

    it "รองรับ pagination" do
      5.times { |i| TestDB.insert_user(email: "user#{i}@example.com") }

      response = client.get("/api/users?page=1&per_page=2")
      body = JSON.parse(response.body)
      body["data"].as_a.size.should eq(2)
      body["page"].as_i.should eq(1)
      body["per_page"].as_i.should eq(2)
    end
  end

  describe "POST /api/users" do
    it "สร้าง user ใหม่" do
      headers = HTTP::Headers{"Content-Type" => "application/json"}
      body = {email: "new@example.com", username: "newuser", password: "Secure@123"}.to_json

      response = client.post("/api/users", headers: headers, body: body)
      response.status_code.should eq(201)

      result = JSON.parse(response.body)
      result["id"].as_i64.should be > 0
      result["email"].as_s.should eq("new@example.com")
      result["password"]?.should be_nil  # ไม่ส่ง password กลับ
    end

    it "คืนค่า 422 สำหรับ invalid data" do
      headers = HTTP::Headers{"Content-Type" => "application/json"}
      body = {email: "invalid-email", username: ""}.to_json

      response = client.post("/api/users", headers: headers, body: body)
      response.status_code.should eq(422)

      result = JSON.parse(response.body)
      result["errors"].as_h.keys.should contain("email")
    end

    it "คืนค่า 409 เมื่อ email ซ้ำ" do
      TestDB.insert_user(email: "existing@example.com")

      headers = HTTP::Headers{"Content-Type" => "application/json"}
      body = {email: "existing@example.com", username: "user", password: "Secure@123"}.to_json

      response = client.post("/api/users", headers: headers, body: body)
      response.status_code.should eq(409)
    end
  end

  describe "GET /api/users/:id" do
    it "คืนค่า user ที่ต้องการ" do
      id = TestDB.insert_user(email: "alice@example.com")

      response = client.get("/api/users/#{id}")
      response.status_code.should eq(200)

      result = JSON.parse(response.body)
      result["email"].as_s.should eq("alice@example.com")
    end

    it "คืนค่า 404 สำหรับ user ที่ไม่มี" do
      response = client.get("/api/users/99999")
      response.status_code.should eq(404)
    end
  end

  describe "PUT /api/users/:id" do
    it "อัปเดต user" do
      id = TestDB.insert_user(email: "alice@example.com")

      headers = HTTP::Headers{
        "Content-Type"  => "application/json",
        "Authorization" => "Bearer #{generate_test_token(id)}"
      }
      body = {first_name: "Alice", last_name: "Smith"}.to_json

      response = client.put("/api/users/#{id}", headers: headers, body: body)
      response.status_code.should eq(200)

      result = JSON.parse(response.body)
      result["first_name"].as_s.should eq("Alice")
    end

    it "คืนค่า 401 เมื่อไม่ได้ authenticate" do
      id = TestDB.insert_user(email: "alice@example.com")

      headers = HTTP::Headers{"Content-Type" => "application/json"}
      response = client.put("/api/users/#{id}", headers: headers, body: "{}")

      response.status_code.should eq(401)
    end
  end

  describe "DELETE /api/users/:id" do
    it "ลบ user" do
      id = TestDB.insert_user(email: "alice@example.com")

      headers = HTTP::Headers{
        "Authorization" => "Bearer #{generate_test_token(id, role: "admin")}"
      }

      response = client.delete("/api/users/#{id}", headers: headers)
      response.status_code.should eq(204)

      # ตรวจสอบว่าลบจริง
      get_response = client.get("/api/users/#{id}")
      get_response.status_code.should eq(404)
    end
  end
end
```

## WebMock - Mocking HTTP Requests

```yaml
# shard.yml
dependencies:
  webmock:
    github: manastech/webmock.cr
    version: ~> 0.13
```

```crystal
# spec/spec_helper.cr
require "webmock"

WebMock.enable!

Spec.after_each do
  WebMock.reset
end
```

```crystal
# spec/integration/payment_spec.cr
require "../spec_helper"
require "../src/payment_service"

describe PaymentService do
  describe "#charge" do
    it "ประมวลผล payment สำเร็จ" do
      # Mock external payment API
      WebMock.stub(:post, "https://api.payment.com/charge")
        .with(
          headers: {"Content-Type" => "application/json"},
          body: /amount.*1000/
        )
        .to_return(
          status: 200,
          body: {
            id:      "ch_12345",
            status:  "succeeded",
            amount:  1000,
            currency: "thb"
          }.to_json,
          headers: {"Content-Type" => "application/json"}
        )

      service = PaymentService.new
      result = service.charge(
        amount: 1000,
        currency: "thb",
        token: "tok_test_123"
      )

      result.success?.should be_true
      result.charge_id.should eq("ch_12345")
    end

    it "จัดการ payment failure" do
      WebMock.stub(:post, "https://api.payment.com/charge")
        .to_return(
          status: 402,
          body: {
            error: {
              code:    "card_declined",
              message: "Your card was declined."
            }
          }.to_json
        )

      service = PaymentService.new
      result = service.charge(amount: 1000, currency: "thb", token: "tok_bad")

      result.success?.should be_false
      result.error_message.should contain("declined")
    end

    it "จัดการ network timeout" do
      WebMock.stub(:post, "https://api.payment.com/charge")
        .to_raise(IO::TimeoutError.new("Connection timed out"))

      service = PaymentService.new
      expect_raises(PaymentService::NetworkError) do
        service.charge(amount: 1000, currency: "thb", token: "tok_test")
      end
    end

    it "retry เมื่อ server error" do
      # ล้มเหลวครั้งแรก สำเร็จครั้งที่สอง
      call_count = 0
      WebMock.stub(:post, "https://api.payment.com/charge") do
        call_count += 1
        if call_count == 1
          WebMock::Response.new(500, "{\"error\": \"Server Error\"}")
        else
          WebMock::Response.new(200, {id: "ch_retry", status: "succeeded"}.to_json)
        end
      end

      service = PaymentService.new(max_retries: 3)
      result = service.charge(amount: 1000, currency: "thb", token: "tok_test")

      result.success?.should be_true
      call_count.should eq(2)
    end
  end
end
```

## Database Testing

```crystal
# spec/support/test_database.cr
require "db"
require "pg"

module TestDB
  DATABASE_URL = ENV["TEST_DATABASE_URL"]? ||
    "postgresql://postgres:password@localhost/myapp_test"

  @@db : DB::Database? = nil

  def self.connection : DB::Database
    @@db ||= DB.open(DATABASE_URL)
  end

  def self.setup
    run_migrations
  end

  def self.teardown
    connection.close
  end

  def self.truncate(*tables : String)
    tables.each do |table|
      connection.exec("TRUNCATE TABLE #{table} RESTART IDENTITY CASCADE")
    end
  end

  def self.truncate_all
    tables = connection.query_all(
      "SELECT tablename FROM pg_tables WHERE schemaname = 'public' AND tablename != 'schema_migrations'",
      as: String
    )
    return if tables.empty?

    connection.exec(
      "TRUNCATE TABLE #{tables.join(", ")} RESTART IDENTITY CASCADE"
    )
  end

  def self.insert_user(email : String, role : String = "user") : Int64
    connection.query_one(
      "INSERT INTO users (email, username, role, created_at, updated_at) VALUES ($1, $2, $3, NOW(), NOW()) RETURNING id",
      email, email.split("@").first, role,
      as: Int64
    )
  end

  def self.insert_product(name : String, price : Float64, category : String = "test") : Int64
    connection.query_one(
      "INSERT INTO products (name, price, category, sku, created_at, updated_at) VALUES ($1, $2, $3, $4, NOW(), NOW()) RETURNING id",
      name, price, category, "SKU-#{rand(100000)}",
      as: Int64
    )
  end

  private def self.run_migrations
    MigrationRunner.new(DATABASE_URL).run!
  end
end
```

### Transaction Rollback Pattern

```crystal
# spec/support/database_cleaner.cr
module DatabaseCleaner
  @@transaction : DB::Transaction? = nil

  def self.start_transaction
    @@transaction = TestDB.connection.begin_transaction
  end

  def self.rollback
    @@transaction.try(&.rollback)
    @@transaction = nil
  end

  def self.transaction_connection : DB::Connection?
    @@transaction.try(&.connection)
  end
end

# ใช้ใน spec
describe "Order Service" do
  before_each do
    DatabaseCleaner.start_transaction
  end

  after_each do
    DatabaseCleaner.rollback  # ล้างข้อมูลอัตโนมัติ
  end

  it "สร้าง order พร้อม items" do
    user_id = TestDB.insert_user("alice@example.com")
    product_id = TestDB.insert_product("Crystal Book", 599.0)

    service = OrderService.new(DatabaseCleaner.transaction_connection.not_nil!)
    order = service.create_order(
      user_id: user_id,
      items: [{product_id: product_id, quantity: 2}]
    )

    order.id.should_not be_nil
    order.total.should eq(1198.0)
    order.items.size.should eq(1)
    # หลัง test จบ transaction จะถูก rollback อัตโนมัติ
  end
end
```

## Test Isolation

```crystal
# spec/integration/isolated_spec.cr

# วิธีที่ 1: Transaction Rollback
describe "UserRepository with Transaction" do
  @tx : DB::Transaction? = nil

  before_each do
    @tx = TestDB.connection.begin_transaction
  end

  after_each do
    @tx.try(&.rollback)
  end

  it "test ใน isolated transaction" do
    conn = @tx.not_nil!.connection
    repo = UserRepository.new(conn)

    user = repo.create("test@example.com", "testuser")
    user.id.should_not be_nil

    found = repo.find_by_email("test@example.com")
    found.should_not be_nil
    # Rollback หลัง test - ไม่มีผลกับ tests อื่น
  end
end

# วิธีที่ 2: Truncate ก่อน test
describe "UserRepository with Truncate" do
  before_each do
    TestDB.truncate("users", "orders", "order_items")
  end

  it "test ใน clean state" do
    repo = UserRepository.new(TestDB.connection)

    repo.count.should eq(0)
    repo.create("test@example.com", "testuser")
    repo.count.should eq(1)
  end
end

# วิธีที่ 3: Factory ที่ unique data
module Factory
  @@counter = Atomic(Int32).new(0)

  def self.unique_email
    n = @@counter.add(1)
    "user#{n}_#{Time.utc.to_unix_ms}@example.com"
  end

  def self.create_user(**attrs)
    TestDB.insert_user(attrs[:email]? || unique_email)
  end
end

describe "Product Service" do
  it "หา products ตาม category" do
    # ใช้ unique data ป้องกัน conflict
    Factory.create_user
    product_id = TestDB.insert_product(
      "Unique Book #{Time.utc.to_unix_ms}",
      299.0,
      "test_category_#{rand(10000)}"
    )

    # test...
  end
end
```

## Full Stack Integration Test

```crystal
# spec/integration/full_stack_spec.cr
require "../spec_helper"

describe "Full Stack: User Registration Flow" do
  before_each do
    TestDB.truncate_all
  end

  it "ผู้ใช้สามารถลงทะเบียนและ login ได้" do
    client = HTTP::Client.new("127.0.0.1", TEST_PORT)

    # Step 1: Register
    register_response = client.post(
      "/api/auth/register",
      headers: HTTP::Headers{"Content-Type" => "application/json"},
      body: {
        email:    "newuser@example.com",
        username: "newuser",
        password: "Secure@123!"
      }.to_json
    )

    register_response.status_code.should eq(201)
    register_data = JSON.parse(register_response.body)
    user_id = register_data["id"].as_i64

    # Step 2: Login
    login_response = client.post(
      "/api/auth/login",
      headers: HTTP::Headers{"Content-Type" => "application/json"},
      body: {
        email:    "newuser@example.com",
        password: "Secure@123!"
      }.to_json
    )

    login_response.status_code.should eq(200)
    login_data = JSON.parse(login_response.body)
    token = login_data["token"].as_s
    token.should_not be_empty

    # Step 3: Access protected endpoint
    profile_response = client.get(
      "/api/users/#{user_id}",
      headers: HTTP::Headers{"Authorization" => "Bearer #{token}"}
    )

    profile_response.status_code.should eq(200)
    profile_data = JSON.parse(profile_response.body)
    profile_data["email"].as_s.should eq("newuser@example.com")

    # Step 4: Logout
    logout_response = client.post(
      "/api/auth/logout",
      headers: HTTP::Headers{"Authorization" => "Bearer #{token}"}
    )

    logout_response.status_code.should eq(204)

    # Step 5: Verify token is invalidated
    after_logout_response = client.get(
      "/api/users/#{user_id}",
      headers: HTTP::Headers{"Authorization" => "Bearer #{token}"}
    )

    after_logout_response.status_code.should eq(401)
  end
end

describe "Full Stack: Order Workflow" do
  before_each do
    TestDB.truncate_all
  end

  it "ผู้ใช้สามารถสั่งซื้อสินค้าได้" do
    client = HTTP::Client.new("127.0.0.1", TEST_PORT)

    # Setup: สร้าง user และ products
    user_id = TestDB.insert_user("buyer@example.com")
    product_id = TestDB.insert_product("Crystal Book", 599.0)
    token = generate_test_token(user_id)

    auth_headers = HTTP::Headers{
      "Content-Type"  => "application/json",
      "Authorization" => "Bearer #{token}"
    }

    # Step 1: ดู products
    products_response = client.get("/api/products")
    products_response.status_code.should eq(200)
    products = JSON.parse(products_response.body)["data"].as_a
    products.size.should be > 0

    # Step 2: เพิ่มสินค้าลง cart
    cart_response = client.post(
      "/api/cart/items",
      headers: auth_headers,
      body: {product_id: product_id, quantity: 2}.to_json
    )
    cart_response.status_code.should eq(200)

    # Step 3: ดู cart
    cart_view = client.get("/api/cart", headers: auth_headers)
    cart_data = JSON.parse(cart_view.body)
    cart_data["total"].as_f.should eq(1198.0)

    # Step 4: Checkout
    order_response = client.post(
      "/api/orders",
      headers: auth_headers,
      body: {payment_method: "credit_card", card_token: "tok_test"}.to_json
    )
    order_response.status_code.should eq(201)
    order_data = JSON.parse(order_response.body)
    order_id = order_data["id"].as_i64

    # Step 5: ตรวจสอบ order
    order_view = client.get("/api/orders/#{order_id}", headers: auth_headers)
    order_view.status_code.should eq(200)
    order = JSON.parse(order_view.body)
    order["status"].as_s.should eq("confirmed")
    order["total"].as_f.should eq(1198.0)
  end
end
```

## Testing Concurrency

```crystal
describe "Concurrent Requests" do
  it "รองรับ concurrent requests ได้" do
    channel = Channel(HTTP::Client::Response).new(10)

    10.times do
      spawn do
        response = HTTP::Client.get("http://127.0.0.1:#{TEST_PORT}/api/health")
        channel.send(response)
      end
    end

    responses = 10.times.map { channel.receive }.to_a
    responses.all? { |r| r.status_code == 200 }.should be_true
  end

  it "จัดการ race condition สำหรับ inventory" do
    product_id = TestDB.insert_product("Limited Item", 999.0)
    TestDB.set_stock(product_id, 1)  # มีแค่ 1 ชิ้น

    user_ids = 5.times.map { |i| TestDB.insert_user("buyer#{i}@example.com") }.to_a

    # ส่ง order พร้อมกัน 5 คน แต่มีสินค้าแค่ 1 ชิ้น
    results = Channel(Int32).new(5)

    user_ids.each do |user_id|
      spawn do
        token = generate_test_token(user_id)
        response = HTTP::Client.post(
          "http://127.0.0.1:#{TEST_PORT}/api/orders",
          headers: HTTP::Headers{
            "Content-Type"  => "application/json",
            "Authorization" => "Bearer #{token}"
          },
          body: {items: [{product_id: product_id, quantity: 1}]}.to_json
        )
        results.send(response.status_code)
      end
    end

    status_codes = 5.times.map { results.receive }.to_a

    success_count = status_codes.count(201)
    fail_count = status_codes.count(409)

    success_count.should eq(1)   # มีแค่ 1 คนที่สั่งได้
    fail_count.should eq(4)      # อีก 4 คนไม่ได้ stock
  end
end
```

## แบบฝึกหัด

1. เขียน integration test ครบถ้วนสำหรับ Blog API: CRUD posts + comments + authentication
2. ใช้ WebMock mock Stripe payment API แล้วเขียน test สำหรับ checkout flow
3. สร้าง test helper ที่ mock email service แล้วตรวจสอบว่า emails ถูกส่ง
4. เขียน test สำหรับ WebSocket endpoint ที่ broadcast message ไปยัง subscribers

## สรุป

Integration Testing ใน Crystal:
- **HTTP Testing**: ใช้ `HTTP::Client` test จริง หรือ test server
- **WebMock**: mock external HTTP calls เพื่อ isolate ระบบ
- **Database Testing**: truncate หรือ transaction rollback สำหรับ clean state
- **Test Isolation**: แต่ละ test ต้องไม่พึ่งพา state จาก test อื่น
- **Full Stack Tests**: ทดสอบ user flow ตั้งแต่ต้นจนจบ
- **Concurrency Testing**: ทดสอบ race conditions และ concurrent requests

Integration tests ช่วยยืนยันว่าระบบทั้งหมดทำงานร่วมกันได้ถูกต้อง
