# Part 194: Microservices ใน Crystal

## บทนำ

Microservices architecture แบ่ง application ออกเป็น services เล็กๆ ที่ communicate ผ่าน APIs Crystal เหมาะมากสำหรับ microservices เพราะเร็ว, เล็ก binary, และ low memory footprint

## Service Structure

```crystal
# Service แต่ละตัวเป็น standalone Crystal app
# users-service/src/main.cr
require "kemal"
require "db"
require "pg"

class UsersService
  DATABASE_URL = ENV["DATABASE_URL"]

  def start
    configure_routes
    Kemal.config.port = (ENV["PORT"]? || "3001").to_i
    Kemal.config.env = ENV["APP_ENV"]? || "production"
    Kemal.run
  end

  private def configure_routes
    get "/users" do |env|
      users = fetch_all_users
      env.response.content_type = "application/json"
      users.to_json
    end

    get "/users/:id" do |env|
      id = env.params.url["id"].to_i64
      user = find_user(id)
      halt(env, status_code: 404, response: {error: "Not found"}.to_json) unless user
      env.response.content_type = "application/json"
      user.to_json
    end

    post "/users" do |env|
      data = JSON.parse(env.request.body.try(&.gets_to_end) || "{}")
      user = create_user(data["name"].as_s, data["email"].as_s)
      env.response.status_code = 201
      env.response.content_type = "application/json"
      user.to_json
    end

    get "/health" do |env|
      env.response.content_type = "application/json"
      {status: "ok", service: "users-service"}.to_json
    end
  end

  private def fetch_all_users
    DB.open(DATABASE_URL) do |db|
      db.query_all("SELECT id, name, email FROM users", as: {id: Int64, name: String, email: String})
    end
  end

  private def find_user(id : Int64)
    DB.open(DATABASE_URL) do |db|
      db.query_one?("SELECT id, name, email FROM users WHERE id = $1", id, as: {id: Int64, name: String, email: String})
    end
  end

  private def create_user(name : String, email : String)
    DB.open(DATABASE_URL) do |db|
      db.query_one("INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id, name, email",
        name, email, as: {id: Int64, name: String, email: String})
    end
  end
end

UsersService.new.start
```

## Service Communication - HTTP

```crystal
# Service client สำหรับ inter-service communication
class UsersServiceClient
  BASE_URL = ENV["USERS_SERVICE_URL"]? || "http://users-service:3001"
  TIMEOUT = 5.seconds

  def initialize
    @client = HTTP::Client.new(URI.parse(BASE_URL))
    @client.connect_timeout = TIMEOUT
    @client.read_timeout = TIMEOUT
  end

  def get_user(id : Int64)
    response = @client.get("/users/#{id}")

    case response.status_code
    when 200
      JSON.parse(response.body)
    when 404
      nil
    else
      raise ServiceError.new("Users service returned #{response.status_code}")
    end
  end

  def create_user(name : String, email : String)
    response = @client.post("/users",
      headers: HTTP::Headers{"Content-Type" => "application/json"},
      body: {name: name, email: email}.to_json
    )

    raise ServiceError.new("Failed to create user: #{response.status_code}") unless response.status_code == 201
    JSON.parse(response.body)
  end

  class ServiceError < Exception; end
end

# ใช้ใน orders service
users_client = UsersServiceClient.new
user = users_client.get_user(123_i64)
puts "User: #{user.try(&.["name"])}"
```

## Circuit Breaker Pattern

```crystal
# Circuit Breaker: ป้องกัน cascade failures
class CircuitBreaker
  enum State
    Closed      # ปกติ, requests ผ่าน
    Open        # หยุด requests (service failed)
    HalfOpen    # ทดสอบว่า service กลับมาไหม
  end

  getter state : State = State::Closed
  getter failure_count : Int32 = 0
  getter last_failure_time : Time?

  def initialize(
    @failure_threshold : Int32 = 5,
    @timeout : Time::Span = 30.seconds,
    @name : String = "circuit_breaker"
  )
  end

  def call(&block : -> T) : T forall T
    case @state
    when .open?
      # ตรวจว่า timeout ผ่านไปแล้วไหม
      if @last_failure_time && Time.local - @last_failure_time.not_nil! > @timeout
        @state = State::HalfOpen
        Log.info { "Circuit #{@name}: trying half-open" }
      else
        raise CircuitOpenError.new("Circuit #{@name} is OPEN")
      end
    end

    begin
      result = block.call
      on_success
      result
    rescue ex
      on_failure(ex)
      raise ex
    end
  end

  private def on_success
    @failure_count = 0
    if @state.half_open?
      @state = State::Closed
      Log.info { "Circuit #{@name}: closed (recovered)" }
    end
  end

  private def on_failure(ex : Exception)
    @failure_count += 1
    @last_failure_time = Time.local
    Log.warn { "Circuit #{@name}: failure #{@failure_count}/#{@failure_threshold}: #{ex.message}" }

    if @failure_count >= @failure_threshold
      @state = State::Open
      Log.error { "Circuit #{@name}: OPENED after #{@failure_count} failures" }
    end
  end

  class CircuitOpenError < Exception; end
end

# ใช้ circuit breaker
users_breaker = CircuitBreaker.new(
  failure_threshold: 3,
  timeout: 10.seconds,
  name: "users-service"
)

begin
  result = users_breaker.call { users_client.get_user(123_i64) }
rescue CircuitBreaker::CircuitOpenError
  # Fallback: return cached data หรือ default
  Log.warn { "Circuit open, using cached user data" }
  cached_user
rescue UsersServiceClient::ServiceError => ex
  Log.error { "Service error: #{ex.message}" }
  nil
end
```

## Service Discovery

```crystal
# Service discovery ด้วย environment variables (simple)
module ServiceRegistry
  def self.users_url : String
    ENV["USERS_SERVICE_URL"]? || "http://users-service:3001"
  end

  def self.orders_url : String
    ENV["ORDERS_SERVICE_URL"]? || "http://orders-service:3002"
  end

  def self.notifications_url : String
    ENV["NOTIFICATIONS_SERVICE_URL"]? || "http://notifications-service:3003"
  end
end

# หรือ service discovery ด้วย Consul/etcd (ตัวอย่าง conceptual)
class ConsulServiceDiscovery
  def initialize(@consul_url : String = "http://consul:8500")
  end

  def discover(service_name : String) : String?
    response = HTTP::Client.get("#{@consul_url}/v1/health/service/#{service_name}?passing=true")
    return nil unless response.status_code == 200

    instances = Array(JSON::Any).from_json(response.body)
    return nil if instances.empty?

    # Load balance ด้วย random selection
    instance = instances.sample
    service = instance["Service"]
    address = service["Address"].as_s
    port = service["Port"].as_i

    "http://#{address}:#{port}"
  end
end
```

## Message Queue สำหรับ Async Communication

```crystal
# Async communication ผ่าน message queue (Redis Pub/Sub)
require "redis"

class EventBus
  def initialize(redis_url : String = ENV["REDIS_URL"]? || "redis://localhost:6379")
    @redis = Redis::Client.new(url: redis_url)
    @subscribers = {} of String => Array(Proc(String, Nil))
  end

  def publish(event : String, data : Hash)
    @redis.publish(event, data.to_json)
  end

  def subscribe(event : String, &handler : String ->)
    @subscribers[event] ||= [] of Proc(String, Nil)
    @subscribers[event] << handler

    spawn do
      @redis.subscribe(event) do |on|
        on.message do |channel, message|
          @subscribers[channel]?.try(&.each(&.call(message)))
        end
      end
    end
  end
end

# Orders service publish event
event_bus = EventBus.new
event_bus.publish("order.created", {
  order_id: "123",
  user_id: "456",
  total: 99.99
})

# Notification service subscribe
event_bus.subscribe("order.created") do |data|
  order = JSON.parse(data)
  send_confirmation_email(order["user_id"].as_s, order["order_id"].as_s)
end
```

## แบบฝึกหัด

1. สร้าง 2 microservices (Users, Orders) ที่ communicate ผ่าน HTTP แล้วรันด้วย docker-compose
2. Implement circuit breaker สำหรับ service calls และทดสอบ failure scenarios
3. สร้าง event bus ด้วย Redis Pub/Sub สำหรับ async communication
4. เพิ่ม distributed tracing ที่ propagate trace IDs ระหว่าง services

## สรุป

Microservices ใน Crystal:
- **Small services**: แต่ละ service เป็น Crystal app ที่ compile เป็น binary เดียว
- **HTTP APIs**: communicate ด้วย JSON over HTTP
- **Circuit Breaker**: ป้องกัน cascade failures
- **Service Discovery**: environment variables หรือ Consul/etcd
- **Message Queues**: async communication ด้วย Redis Pub/Sub หรือ RabbitMQ
- **Crystal advantages**: low memory (5-50MB per service), fast startup, static binary
- **Docker**: ง่ายสำหรับ containerize Crystal microservices
