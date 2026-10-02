# Part 133: Kemal Middleware - การสร้าง Middleware ใน Kemal

## บทนำ

Middleware ใน Kemal ช่วยให้เราสามารถประมวลผล request ก่อนหรือหลัง routes handlers เหมาะสำหรับ logging, authentication, rate limiting, caching และอื่นๆ

## before_all และ after_all

```crystal
require "kemal"

# ทำงานก่อนทุก routes
before_all do |env|
  puts "Before: #{env.request.method} #{env.request.path}"
  env.set("request_start", Time.monotonic)
end

# ทำงานหลังทุก routes
after_all do |env|
  if start = env.get?("request_start")
    elapsed = (Time.monotonic - start.as(Time::Span)).total_milliseconds
    env.response.headers["X-Response-Time"] = "#{elapsed.round(2)}ms"
    puts "After: #{env.response.status_code} (#{elapsed.round(2)}ms)"
  end
end

get "/" do
  "Hello, Kemal!"
end

Kemal.run
```

## Custom Middleware Handlers

```crystal
require "kemal"

# Custom Handler
class RequestIDHandler < Kemal::Handler
  exclude ["/health"]  # ยกเว้น health endpoint
  
  def call(context : HTTP::Server::Context)
    request_id = context.request.headers["X-Request-ID"]? || Random::Secure.hex(16)
    context.request.headers["X-Request-ID"] = request_id
    context.response.headers["X-Request-ID"] = request_id
    
    call_next(context)
  end
end

# เพิ่ม handler
add_handler RequestIDHandler.new

get "/" do |env|
  "Request ID: #{env.request.headers["X-Request-ID"]}"
end

Kemal.run
```

## Logging Middleware

```crystal
require "kemal"

class LoggingHandler < Kemal::Handler
  LOG_FILE = "logs/access.log"
  
  def initialize
    @file = nil
    setup_file
  end
  
  def call(context : HTTP::Server::Context)
    start = Time.monotonic
    
    call_next(context)
    
    elapsed = (Time.monotonic - start).total_milliseconds
    
    log_entry = {
      "timestamp" => Time.local.to_rfc3339,
      "method" => context.request.method,
      "path" => context.request.path,
      "status" => context.response.status_code,
      "elapsed_ms" => elapsed.round(2),
      "ip" => context.request.remote_address.to_s,
      "user_agent" => context.request.headers["User-Agent"]? || "",
      "request_id" => context.request.headers["X-Request-ID"]? || "",
    }
    
    # Log to stdout
    status_color = case context.response.status_code
    when 200..299 then "\e[32m"  # green
    when 300..399 then "\e[33m"  # yellow
    when 400..499 then "\e[31m"  # red
    when 500..599 then "\e[35m"  # magenta
    else "\e[0m"
    end
    
    puts "#{status_color}#{log_entry["status"]}\e[0m #{log_entry["method"]} #{log_entry["path"]} (#{elapsed.round(1)}ms)"
    
    # Log to file
    @file.try { |f| f.puts(log_entry.to_json) }
  end
  
  private def setup_file
    Dir.mkdir_p("logs")
    @file = File.open(LOG_FILE, "a") rescue nil
  end
end

add_handler LoggingHandler.new

get "/" do
  "Logged request!"
end

Kemal.run
```

## Authentication Middleware

```crystal
require "kemal"
require "json"

# JWT/Token Authentication Middleware
class AuthHandler < Kemal::Handler
  # Routes ที่ไม่ต้อง authenticate
  PUBLIC_PATHS = ["/", "/login", "/register", "/health", "/api/v1/auth/login"]
  
  def call(context : HTTP::Server::Context)
    # ข้าม public routes
    if PUBLIC_PATHS.any? { |path| context.request.path.starts_with?(path) }
      call_next(context)
      return
    end
    
    # ตรวจสอบ Authorization header
    auth = context.request.headers["Authorization"]?
    
    unless auth && auth.starts_with?("Bearer ")
      unauthorized(context)
      return
    end
    
    token = auth[7..]
    
    # ตรวจสอบ token
    user = verify_token(token)
    
    unless user
      unauthorized(context)
      return
    end
    
    # เก็บ user info ใน context
    context.set("current_user", user)
    context.request.headers["X-User-ID"] = user["id"]
    context.request.headers["X-User-Role"] = user["role"]
    
    call_next(context)
  end
  
  private def verify_token(token : String) : Hash(String, String)?
    # จำลอง token verification
    valid_tokens = {
      "token_admin_123" => {"id" => "1", "name" => "Admin", "role" => "admin"},
      "token_user_456" => {"id" => "2", "name" => "User", "role" => "user"},
    }
    
    valid_tokens[token]?
  end
  
  private def unauthorized(context : HTTP::Server::Context)
    context.response.status_code = 401
    context.response.content_type = "application/json"
    context.response.print({"error" => "Unauthorized", "message" => "Valid Bearer token required"}.to_json)
  end
end

# Role-based Authorization Middleware
class AuthorizationHandler < Kemal::Handler
  ADMIN_PATHS = ["/admin/"]
  
  def call(context : HTTP::Server::Context)
    # ตรวจสอบ admin routes
    if ADMIN_PATHS.any? { |path| context.request.path.starts_with?(path) }
      user = context.get?("current_user").try(&.as(Hash(String, String))?)
      
      unless user && user["role"] == "admin"
        context.response.status_code = 403
        context.response.content_type = "application/json"
        context.response.print({"error" => "Forbidden", "message" => "Admin access required"}.to_json)
        return
      end
    end
    
    call_next(context)
  end
end

add_handler AuthHandler.new
add_handler AuthorizationHandler.new

get "/login" do |env|
  env.response.content_type = "application/json"
  {"token" => "token_user_456", "expires_in" => 3600}.to_json
end

get "/profile" do |env|
  user = env.get?("current_user").try(&.as(Hash(String, String)?))
  env.response.content_type = "application/json"
  {"user" => user}.to_json
end

get "/admin/dashboard" do |env|
  env.response.content_type = "application/json"
  {"admin" => true, "data" => "Secret admin data"}.to_json
end

Kemal.run
```

## Rate Limiting Middleware

```crystal
require "kemal"
require "json"

class RateLimitHandler < Kemal::Handler
  def initialize(@limit : Int32, @window : Time::Span)
    @requests = Hash(String, Array(Time)).new { |h, k| h[k] = [] of Time }
    @mutex = Mutex.new
  end
  
  def call(context : HTTP::Server::Context)
    client_ip = context.request.remote_address.to_s.split(":").first
    
    if rate_limited?(client_ip)
      context.response.status_code = 429
      context.response.headers["Retry-After"] = @window.total_seconds.to_i.to_s
      context.response.headers["X-RateLimit-Limit"] = @limit.to_s
      context.response.headers["X-RateLimit-Remaining"] = "0"
      context.response.content_type = "application/json"
      context.response.print({"error" => "Rate limit exceeded", "retry_after" => @window.total_seconds.to_i}.to_json)
      return
    end
    
    remaining = remaining_requests(client_ip)
    context.response.headers["X-RateLimit-Limit"] = @limit.to_s
    context.response.headers["X-RateLimit-Remaining"] = remaining.to_s
    
    call_next(context)
  end
  
  private def rate_limited?(client_ip : String) : Bool
    @mutex.synchronize do
      now = Time.local
      cutoff = now - @window
      
      # ลบ requests เก่า
      @requests[client_ip].reject! { |t| t < cutoff }
      
      if @requests[client_ip].size >= @limit
        true
      else
        @requests[client_ip] << now
        false
      end
    end
  end
  
  private def remaining_requests(client_ip : String) : Int32
    @mutex.synchronize do
      [@limit - @requests[client_ip].size, 0].max
    end
  end
end

# 100 requests per minute
add_handler RateLimitHandler.new(100, 1.minute)

get "/api/data" do
  "API data"
end

Kemal.run
```

## CORS Middleware

```crystal
require "kemal"

class CORSHandler < Kemal::Handler
  def initialize(
    @allowed_origins : Array(String) = ["*"],
    @allowed_methods : Array(String) = ["GET", "POST", "PUT", "DELETE", "OPTIONS", "PATCH"],
    @allowed_headers : Array(String) = ["Content-Type", "Authorization", "X-Requested-With"],
    @max_age : Int32 = 86400,
    @allow_credentials : Bool = false
  )
  end
  
  def call(context : HTTP::Server::Context)
    origin = context.request.headers["Origin"]?
    
    if origin && should_allow?(origin)
      context.response.headers["Access-Control-Allow-Origin"] = 
        @allowed_origins.includes?("*") ? "*" : origin
      
      context.response.headers["Access-Control-Allow-Methods"] = @allowed_methods.join(", ")
      context.response.headers["Access-Control-Allow-Headers"] = @allowed_headers.join(", ")
      context.response.headers["Access-Control-Max-Age"] = @max_age.to_s
      
      if @allow_credentials
        context.response.headers["Access-Control-Allow-Credentials"] = "true"
      end
      
      # Handle preflight
      if context.request.method == "OPTIONS"
        context.response.status_code = 204
        return
      end
    end
    
    call_next(context)
  end
  
  private def should_allow?(origin : String) : Bool
    @allowed_origins.includes?("*") || @allowed_origins.includes?(origin)
  end
end

add_handler CORSHandler.new(
  allowed_origins: ["http://localhost:3000", "https://myapp.com"],
  allow_credentials: true
)

Kemal.run
```

## Caching Middleware

```crystal
require "kemal"
require "json"

class CachingHandler < Kemal::Handler
  def initialize(@ttl : Time::Span = 5.minutes)
    @cache = {} of String => {String, String, Time}  # key => {body, content_type, expires_at}
    @mutex = Mutex.new
  end
  
  def call(context : HTTP::Server::Context)
    # เฉพาะ GET requests เท่านั้น
    unless context.request.method == "GET"
      call_next(context)
      return
    end
    
    # Skip cache ถ้ามี Cache-Control: no-cache
    if context.request.headers["Cache-Control"]? == "no-cache"
      call_next(context)
      return
    end
    
    cache_key = context.request.path + "?" + (context.request.query || "")
    
    # ตรวจสอบ cache
    if cached = get_cached(cache_key)
      body, content_type, expires_at = cached
      
      context.response.content_type = content_type
      context.response.headers["X-Cache"] = "HIT"
      context.response.headers["Cache-Control"] = "max-age=#{@ttl.total_seconds.to_i}"
      context.response.print(body)
      return
    end
    
    # ไม่มีใน cache, ดำเนินการต่อ
    call_next(context)
    
    # Cache response
    if context.response.status_code == 200
      body = context.response.output.to_s rescue ""
      content_type = context.response.headers["Content-Type"]? || "text/plain"
      cache_response(cache_key, body, content_type)
      context.response.headers["X-Cache"] = "MISS"
    end
  end
  
  private def get_cached(key : String) : {String, String, Time}?
    @mutex.synchronize do
      if cached = @cache[key]?
        body, ct, expires = cached
        return nil if Time.local > expires
        cached
      else
        nil
      end
    end
  end
  
  private def cache_response(key : String, body : String, content_type : String)
    @mutex.synchronize do
      @cache[key] = {body, content_type, Time.local + @ttl}
    end
  end
end

add_handler CachingHandler.new(ttl: 30.seconds)

get "/expensive-data" do
  sleep 1.second  # จำลอง expensive operation
  {"data" => "Computed at #{Time.local}", "heavy" => true}.to_json
end

Kemal.run
```

## Compression Middleware

```crystal
require "kemal"

class CompressionHandler < Kemal::Handler
  MIN_SIZE = 1024  # compress responses > 1KB
  
  def call(context : HTTP::Server::Context)
    accept_encoding = context.request.headers["Accept-Encoding"]? || ""
    
    call_next(context)
    
    # ตรวจสอบว่า client รองรับ gzip
    if accept_encoding.includes?("gzip")
      # ใน production ควรใช้ Compress::Gzip
      context.response.headers["Content-Encoding"] = "gzip"
    end
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Audit Log Middleware

```crystal
require "kemal"
require "json"

class AuditLogHandler < Kemal::Handler
  struct AuditEntry
    include JSON::Serializable
    
    property timestamp : String
    property method : String
    property path : String
    property status : Int32
    property user_id : String?
    property ip : String
    property body : String?
    property elapsed_ms : Float64
    
    def initialize(@timestamp, @method, @path, @status, @user_id, @ip, @body, @elapsed_ms)
    end
  end
  
  AUDITED_METHODS = ["POST", "PUT", "PATCH", "DELETE"]
  
  def initialize
    @entries = Channel(AuditEntry).new(1000)
    start_writer
  end
  
  def call(context : HTTP::Server::Context)
    start = Time.monotonic
    
    # อ่าน body ก่อน (สำหรับ POST/PUT/PATCH)
    body = nil
    if AUDITED_METHODS.includes?(context.request.method)
      body = context.request.body.try(&.gets_to_end)
      # Reset body เพื่อให้ handler อ่านได้อีกครั้ง
      context.request.body = IO::Memory.new(body || "") if body
    end
    
    call_next(context)
    
    elapsed = (Time.monotonic - start).total_milliseconds
    
    if AUDITED_METHODS.includes?(context.request.method)
      entry = AuditEntry.new(
        timestamp: Time.local.to_rfc3339,
        method: context.request.method,
        path: context.request.path,
        status: context.response.status_code,
        user_id: context.request.headers["X-User-ID"]?,
        ip: context.request.remote_address.to_s,
        body: body.try { |b| b[0..200] },  # แค่ 200 chars
        elapsed_ms: elapsed.round(2)
      )
      
      @entries.send(entry) rescue nil
    end
  end
  
  private def start_writer
    spawn do
      while entry = @entries.receive?
        puts "AUDIT: #{entry.to_json}"
        # ใน production: บันทึกลง database หรือ log service
      end
    end
  end
end

add_handler AuditLogHandler.new
add_handler AuthHandler.new

post "/api/users" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body) rescue nil
  
  env.response.content_type = "application/json"
  env.response.status_code = 201
  {"created" => true, "data" => data}.to_json
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **before_all/after_all**: ทำงานก่อน/หลังทุก routes
2. **Custom Handler**: สร้าง middleware ด้วย `Kemal::Handler`
3. **call_next**: ส่งต่อให้ handler ถัดไป
4. **exclude**: ยกเว้น paths บางส่วน
5. **Logging Middleware**: log requests
6. **Authentication**: ตรวจสอบ token
7. **Authorization**: ตรวจสอบ role
8. **Rate Limiting**: จำกัด requests per window
9. **CORS**: จัดการ Cross-Origin requests
10. **Caching**: cache response
11. **Audit Log**: บันทึก sensitive operations

ลำดับ Middleware สำคัญ: ใช้ `add_handler` ตามลำดับที่ต้องการ
