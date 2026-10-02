# Part 197: Security Best Practices ใน Crystal

## บทนำ

Security เป็นสิ่งสำคัญในทุก web application Crystal มีข้อได้เปรียบด้าน security จาก type system ที่ strict แต่ยังต้องปฏิบัติตาม best practices

## Input Validation

```crystal
# Validate inputs เสมอก่อนประมวลผล
module Validators
  def self.validate_email!(email : String) : String
    email = email.strip.downcase
    unless email.matches?(/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i)
      raise ArgumentError.new("Invalid email format: #{email}")
    end
    email
  end

  def self.validate_password!(password : String) : String
    errors = [] of String
    errors << "at least 8 characters" if password.size < 8
    errors << "uppercase letter" unless password.matches?(/[A-Z]/)
    errors << "lowercase letter" unless password.matches?(/[a-z]/)
    errors << "digit" unless password.matches?(/\d/)
    errors << "special character" unless password.matches?(/[!@#$%^&*]/)

    unless errors.empty?
      raise ArgumentError.new("Password must contain: #{errors.join(", ")}")
    end
    password
  end

  def self.validate_integer!(value : String, min : Int32? = nil, max : Int32? = nil) : Int32
    n = value.to_i32?
    raise ArgumentError.new("Not a valid integer: #{value}") unless n

    if min && n < min
      raise ArgumentError.new("Value must be >= #{min}")
    end
    if max && n > max
      raise ArgumentError.new("Value must be <= #{max}")
    end
    n
  end

  def self.sanitize_html(input : String) : String
    # Strip dangerous HTML tags and attributes
    input
      .gsub(/<script[^>]*>.*?<\/script>/mi, "")
      .gsub(/<iframe[^>]*>.*?<\/iframe>/mi, "")
      .gsub(/on\w+\s*=\s*["'][^"']*["']/i, "")
      .gsub(/javascript:/i, "")
  end
end

# ใช้งาน
begin
  email = Validators.validate_email!(params["email"])
  password = Validators.validate_password!(params["password"])
rescue ArgumentError => ex
  halt(env, status_code: 422, response: {error: ex.message}.to_json)
end
```

## SQL Injection Prevention

```crystal
# NEVER concatenate user input ใน SQL

# BAD: SQL Injection vulnerability
def find_user_bad(email : String)
  db.query("SELECT * FROM users WHERE email = '#{email}'")
  # ถ้า email = "' OR '1'='1" จะได้ users ทั้งหมด!
end

# GOOD: parameterized queries เสมอ
def find_user(email : String)
  db.query_one?("SELECT * FROM users WHERE email = $1", email, as: User)
end

def search_users(name : String, active : Bool)
  db.query_all(
    "SELECT * FROM users WHERE name ILIKE $1 AND active = $2",
    "%#{name.gsub("%", "\\%").gsub("_", "\\_")}%",  # escape LIKE special chars
    active,
    as: User
  )
end

# Query builder แบบ safe
class SafeQuery
  @conditions : Array(String) = [] of String
  @params : Array(DB::Any) = [] of DB::Any
  @param_count = 0

  def where(condition : String, value : DB::Any) : self
    @param_count += 1
    @conditions << condition.gsub("?", "$#{@param_count}")
    @params << value
    self
  end

  def build_where : {String, Array(DB::Any)}
    {
      @conditions.empty? ? "" : "WHERE #{@conditions.join(" AND ")}",
      @params
    }
  end
end
```

## XSS Prevention

```crystal
# Cross-Site Scripting (XSS) prevention

# HTML escaping
def html_escape(input : String) : String
  input
    .gsub("&", "&amp;")
    .gsub("<", "&lt;")
    .gsub(">", "&gt;")
    .gsub("\"", "&quot;")
    .gsub("'", "&#x27;")
    .gsub("/", "&#x2F;")
end

# Response headers สำหรับ XSS protection
class SecurityHeaders
  include HTTP::Handler

  def call(context : HTTP::Server::Context)
    response = context.response

    # XSS Protection
    response.headers["X-XSS-Protection"] = "1; mode=block"
    response.headers["X-Content-Type-Options"] = "nosniff"
    response.headers["X-Frame-Options"] = "DENY"

    # Content Security Policy
    response.headers["Content-Security-Policy"] = [
      "default-src 'self'",
      "script-src 'self' 'nonce-#{generate_nonce}'",
      "style-src 'self' fonts.googleapis.com",
      "img-src 'self' data: https:",
      "font-src 'self' fonts.gstatic.com",
      "connect-src 'self'",
      "frame-ancestors 'none'",
    ].join("; ")

    # HSTS
    response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains; preload"

    # Referrer Policy
    response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

    call_next(context)
  end

  private def generate_nonce : String
    Random::Secure.base64(16)
  end
end
```

## CSRF Protection

```crystal
# Cross-Site Request Forgery protection
class CSRFProtection
  include HTTP::Handler

  CSRF_TOKEN_SESSION_KEY = "_csrf_token"
  CSRF_TOKEN_PARAM = "_csrf_token"
  SAFE_METHODS = %w[GET HEAD OPTIONS TRACE]

  def call(context : HTTP::Server::Context)
    method = context.request.method

    unless SAFE_METHODS.includes?(method)
      # Verify CSRF token สำหรับ state-changing requests
      unless valid_csrf_token?(context)
        context.response.status_code = 403
        context.response.print "CSRF token validation failed"
        return
      end
    end

    call_next(context)
  end

  def self.generate_token : String
    Random::Secure.base64(32)
  end

  private def valid_csrf_token?(context : HTTP::Server::Context) : Bool
    session_token = context.session[CSRF_TOKEN_SESSION_KEY]?
    request_token = context.params.body[CSRF_TOKEN_PARAM]? ||
                    context.request.headers["X-CSRF-Token"]?

    return false unless session_token && request_token
    Crypto::Subtle.constant_time_compare(session_token, request_token)
  end
end
```

## Authentication & Password Hashing

```crystal
# Password hashing ด้วย bcrypt (ผ่าน C binding หรือ shard)
# ใช้ crypto shard: github: bcrypt/bcrypt.cr

require "bcrypt"

class AuthService
  BCRYPT_COST = 12  # ปรับตาม server speed

  def self.hash_password(plaintext : String) : String
    BCrypt::Password.create(plaintext, cost: BCRYPT_COST).to_s
  end

  def self.verify_password(plaintext : String, hash : String) : Bool
    BCrypt::Password.new(hash).verify(plaintext)
  rescue
    false
  end
end

# JWT Token
require "jwt"

class TokenService
  SECRET = ENV["JWT_SECRET"]? || raise "JWT_SECRET not set"
  EXPIRY = 24.hours

  def self.generate(payload : Hash(String, JSON::Any)) : String
    claims = payload.merge({
      "exp" => JSON::Any.new(EXPIRY.from_now.to_unix),
      "iat" => JSON::Any.new(Time.local.to_unix),
    })
    JWT.encode(claims, SECRET, JWT::Algorithm::HS256)
  end

  def self.verify(token : String) : Hash(String, JSON::Any)?
    payload, = JWT.decode(token, SECRET, JWT::Algorithm::HS256)
    payload
  rescue JWT::ExpiredSignatureError
    Log.warn { "Expired JWT token" }
    nil
  rescue JWT::VerificationError
    Log.warn { "Invalid JWT signature" }
    nil
  end
end

# ใช้งาน
token = TokenService.generate({"user_id" => JSON::Any.new(123_i64), "role" => JSON::Any.new("admin")})
payload = TokenService.verify(token)
puts payload.try(&.["user_id"])
```

## Rate Limiting

```crystal
# Rate limiting: ป้องกัน brute force และ DDoS
class RateLimiter
  def initialize(
    @redis : Redis::Client,
    @max_requests : Int32 = 100,
    @window : Time::Span = 60.seconds
  )
  end

  def allow?(identifier : String) : Bool
    key = "rate_limit:#{identifier}"
    window_seconds = @window.total_seconds.to_i

    count = @redis.incr(key).as(Int64)

    if count == 1
      @redis.expire(key, window_seconds)
    end

    count <= @max_requests
  end

  def remaining(identifier : String) : Int32
    key = "rate_limit:#{identifier}"
    count = @redis.get(key).try(&.to_i32) || 0
    [@max_requests - count, 0].max
  end
end

# Middleware
class RateLimitMiddleware
  include HTTP::Handler

  def initialize(@limiter : RateLimiter)
  end

  def call(context : HTTP::Server::Context)
    identifier = context.request.remote_address.try(&.to_s) || "unknown"

    unless @limiter.allow?(identifier)
      context.response.status_code = 429
      context.response.headers["Retry-After"] = "60"
      context.response.print "Too Many Requests"
      return
    end

    context.response.headers["X-RateLimit-Remaining"] = @limiter.remaining(identifier).to_s
    call_next(context)
  end
end
```

## แบบฝึกหัด

1. สร้าง input validation middleware ที่ validate และ sanitize ทุก request body
2. Implement CSRF protection สำหรับ form submissions
3. เพิ่ม rate limiting แยกตาม endpoint (auth endpoints เข้มงวดกว่า)
4. สร้าง security audit ที่ check headers, HTTPS, และ dependencies

## สรุป

Security Best Practices ใน Crystal:
- **Input validation**: validate ทุก user input ก่อนประมวลผล
- **Parameterized queries**: ป้องกัน SQL injection 100%
- **HTML escaping**: escape user-generated content ก่อน render
- **CSP headers**: Content-Security-Policy ป้องกัน XSS
- **CSRF tokens**: ป้องกัน cross-site request forgery
- **bcrypt**: hash passwords ด้วย adaptive algorithm
- **JWT**: stateless authentication tokens
- **Rate limiting**: ป้องกัน brute force และ abuse
- **HSTS**: บังคับ HTTPS ผ่าน Strict-Transport-Security header
- Crystal's type system ช่วย prevent type confusion bugs
