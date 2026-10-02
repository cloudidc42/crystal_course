# Part 148: CORS - Cross-Origin Resource Sharing

## บทนำ

CORS (Cross-Origin Resource Sharing) เป็น security mechanism ที่ browser บังคับใช้เพื่อป้องกัน cross-origin requests ที่ไม่ได้รับอนุญาต

## CORS พื้นฐาน

```crystal
require "kemal"

# CORS headers พื้นฐาน
before_all do |env|
  env.response.headers["Access-Control-Allow-Origin"] = "*"
  env.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
  env.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
end

# Handle OPTIONS preflight
options "/*" do |env|
  env.response.status_code = 204
  ""
end

get "/api/data" do |env|
  env.response.content_type = "application/json"
  {"data" => "Hello CORS"}.to_json
end

Kemal.run
```

## Production CORS Configuration

```crystal
require "kemal"

class CORSConfig
  property allowed_origins : Array(String)
  property allowed_methods : Array(String)
  property allowed_headers : Array(String)
  property exposed_headers : Array(String)
  property max_age : Int32
  property allow_credentials : Bool
  
  def initialize
    @allowed_origins = [] of String
    @allowed_methods = ["GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"]
    @allowed_headers = ["Content-Type", "Authorization", "X-Requested-With", "Accept", "Origin"]
    @exposed_headers = ["X-Total-Count", "X-Page", "X-Per-Page"]
    @max_age = 86400  # 1 day
    @allow_credentials = false
  end
end

class CORSHandler < Kemal::Handler
  def initialize(@config : CORSConfig)
  end
  
  def call(context : HTTP::Server::Context)
    origin = context.request.headers["Origin"]?
    
    if origin
      if allowed_origin?(origin)
        set_cors_headers(context, origin)
        
        # Handle preflight
        if context.request.method == "OPTIONS"
          context.response.status_code = 204
          return
        end
      else
        # Origin ไม่ได้รับอนุญาต - ไม่ set headers
        # browser จะ block request
        if context.request.method == "OPTIONS"
          context.response.status_code = 403
          context.response.content_type = "application/json"
          context.response.print({"error" => "CORS: Origin not allowed"}.to_json)
          return
        end
      end
    end
    
    call_next(context)
  end
  
  private def allowed_origin?(origin : String) : Bool
    return true if @config.allowed_origins.includes?("*")
    return true if @config.allowed_origins.includes?(origin)
    
    # รองรับ wildcard subdomain เช่น "*.example.com"
    @config.allowed_origins.any? do |allowed|
      if allowed.starts_with?("*.")
        domain = allowed[2..]
        origin.ends_with?(".#{domain}") || origin == "https://#{domain}" || origin == "http://#{domain}"
      else
        false
      end
    end
  end
  
  private def set_cors_headers(context : HTTP::Server::Context, origin : String)
    # Allow-Origin
    if @config.allowed_origins.includes?("*") && !@config.allow_credentials
      context.response.headers["Access-Control-Allow-Origin"] = "*"
    else
      context.response.headers["Access-Control-Allow-Origin"] = origin
      context.response.headers["Vary"] = "Origin"
    end
    
    # Allow-Methods
    context.response.headers["Access-Control-Allow-Methods"] = 
      @config.allowed_methods.join(", ")
    
    # Allow-Headers
    if requested = context.request.headers["Access-Control-Request-Headers"]?
      # Echo requested headers ถ้าอยู่ใน allowed list
      allowed = requested.split(",").map(&.strip)
        .select { |h| @config.allowed_headers.any? { |a| a.downcase == h.downcase } }
      context.response.headers["Access-Control-Allow-Headers"] = allowed.join(", ")
    else
      context.response.headers["Access-Control-Allow-Headers"] = 
        @config.allowed_headers.join(", ")
    end
    
    # Expose headers
    unless @config.exposed_headers.empty?
      context.response.headers["Access-Control-Expose-Headers"] = 
        @config.exposed_headers.join(", ")
    end
    
    # Max-Age (cache preflight response)
    context.response.headers["Access-Control-Max-Age"] = @config.max_age.to_s
    
    # Allow-Credentials
    if @config.allow_credentials
      context.response.headers["Access-Control-Allow-Credentials"] = "true"
    end
  end
end

# Configuration สำหรับ production
cors_config = CORSConfig.new
cors_config.allowed_origins = [
  "https://myapp.com",
  "https://www.myapp.com",
  "*.myapp.com",             # wildcard subdomain
  "http://localhost:3000",   # development
  "http://localhost:8080",   # development alternative
]
cors_config.allow_credentials = true
cors_config.allowed_headers = [
  "Content-Type",
  "Authorization",
  "X-API-Key",
  "X-Requested-With",
  "Accept",
  "Origin",
  "X-CSRF-Token",
]
cors_config.exposed_headers = [
  "X-Total-Count",
  "X-RateLimit-Remaining",
  "X-Request-ID",
]

add_handler CORSHandler.new(cors_config)

Kemal.run
```

## Environment-based CORS

```crystal
require "kemal"

CORS_CONFIG = case ENV["CRYSTAL_ENV"]? || "development"
when "production"
  {
    "origins" => ["https://myapp.com", "https://app.myapp.com"],
    "credentials" => true,
    "max_age" => 86400
  }
when "staging"
  {
    "origins" => ["https://staging.myapp.com", "http://localhost:3000"],
    "credentials" => true,
    "max_age" => 3600
  }
else  # development
  {
    "origins" => ["*"],
    "credentials" => false,
    "max_age" => 300
  }
end

before_all do |env|
  origins = CORS_CONFIG["origins"].as(Array(String))
  request_origin = env.request.headers["Origin"]?
  
  if request_origin
    is_allowed = origins.includes?("*") || origins.includes?(request_origin)
    
    if is_allowed
      allow_origin = origins.includes?("*") ? "*" : request_origin
      env.response.headers["Access-Control-Allow-Origin"] = allow_origin
      
      if CORS_CONFIG["credentials"].as(Bool)
        env.response.headers["Access-Control-Allow-Credentials"] = "true"
        # ถ้า credentials=true ต้องไม่ใช้ *
        env.response.headers["Access-Control-Allow-Origin"] = request_origin
      end
      
      env.response.headers["Access-Control-Max-Age"] = CORS_CONFIG["max_age"].to_s
    end
  end
end

options "/*" do |env|
  env.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
  env.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization, X-API-Key"
  env.response.status_code = 204
  ""
end

Kemal.run
```

## CORS สำหรับ API

```crystal
require "kemal"

# Simple CORS สำหรับ public API
module PublicAPICors
  def self.setup
    before_all "/api/public/*" do |env|
      # Public API - อนุญาต origin ใดก็ได้
      env.response.headers["Access-Control-Allow-Origin"] = "*"
      env.response.headers["Access-Control-Allow-Methods"] = "GET, OPTIONS"
      env.response.headers["Access-Control-Allow-Headers"] = "Content-Type"
      env.response.headers["Access-Control-Max-Age"] = "3600"
      env.response.headers["Cache-Control"] = "public, max-age=300"
    end
    
    options "/api/public/*" do |env|
      env.response.status_code = 204
      ""
    end
  end
end

# Strict CORS สำหรับ private API
module PrivateAPICors
  ALLOWED_ORIGINS = Set.new(["https://app.example.com", "https://admin.example.com"])
  
  def self.setup
    before_all "/api/private/*" do |env|
      origin = env.request.headers["Origin"]?
      
      if origin && ALLOWED_ORIGINS.includes?(origin)
        env.response.headers["Access-Control-Allow-Origin"] = origin
        env.response.headers["Access-Control-Allow-Credentials"] = "true"
        env.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
        env.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization, X-CSRF-Token"
        env.response.headers["Vary"] = "Origin"
      elsif origin
        # Origin ไม่ได้รับอนุญาต
        if env.request.method == "OPTIONS"
          env.response.status_code = 403
          env.response.print({"error" => "Origin not allowed"}.to_json)
          next
        end
        # Simple requests จะ proceed แต่ response จะถูก block โดย browser
      end
    end
    
    options "/api/private/*" do |env|
      env.response.status_code = 204
      ""
    end
  end
end

PublicAPICors.setup
PrivateAPICors.setup

# Public routes
get "/api/public/products" do |env|
  env.response.content_type = "application/json"
  [{"id" => 1, "name" => "Product 1"}].to_json
end

# Private routes
get "/api/private/orders" do |env|
  # ต้อง authenticate
  env.response.content_type = "application/json"
  [{"id" => 1, "status" => "pending"}].to_json
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Dynamic CORS

```crystal
require "kemal"
require "json"

# Dynamic CORS - อ่าน allowed origins จาก database/config
class DynamicCORSHandler < Kemal::Handler
  def initialize
    @allowed_origins_cache = [] of String
    @cache_updated_at = Time::UNIX_EPOCH
    @cache_ttl = 5.minutes
    @mutex = Mutex.new
  end
  
  def call(context : HTTP::Server::Context)
    origin = context.request.headers["Origin"]?
    
    if origin && allowed_origin?(origin)
      context.response.headers["Access-Control-Allow-Origin"] = origin
      context.response.headers["Access-Control-Allow-Credentials"] = "true"
      context.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
      context.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
      context.response.headers["Vary"] = "Origin"
      
      if context.request.method == "OPTIONS"
        context.response.status_code = 204
        return
      end
    end
    
    call_next(context)
  end
  
  private def allowed_origin?(origin : String) : Bool
    origins = get_allowed_origins
    origins.includes?(origin) || origins.includes?("*")
  end
  
  private def get_allowed_origins : Array(String)
    @mutex.synchronize do
      if Time.local - @cache_updated_at > @cache_ttl
        # Reload from config/database
        @allowed_origins_cache = load_origins
        @cache_updated_at = Time.local
      end
      @allowed_origins_cache
    end
  end
  
  private def load_origins : Array(String)
    # ใน production: อ่านจาก database หรือ config file
    if origins_str = ENV["ALLOWED_ORIGINS"]?
      origins_str.split(",").map(&.strip)
    else
      ["http://localhost:3000", "https://myapp.com"]
    end
  end
end

add_handler DynamicCORSHandler.new

# Admin endpoint สำหรับ reload CORS config
post "/admin/cors/reload" do |env|
  # trigger reload ใน next request
  env.response.content_type = "application/json"
  {"message" => "CORS configuration will be reloaded on next request"}.to_json
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CORS Headers**: Access-Control-Allow-Origin, Methods, Headers
2. **Preflight Request**: OPTIONS method
3. **Credentials**: cookies/auth ข้าม origins
4. **Max-Age**: cache preflight response
5. **Vary Header**: บอก CDN ว่า response แตกต่างตาม Origin
6. **Wildcard vs Specific**: *, specific domain, wildcard subdomain
7. **Environment Config**: production/staging/development
8. **Dynamic CORS**: อ่าน config จาก database
9. **Public vs Private API**: policy ต่างกัน

CORS ที่ secure: ใช้ specific origins, set Vary: Origin, และระวัง credentials
