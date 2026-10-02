# Part 147: Rate Limiting - การจำกัด Request Rate

## บทนำ

Rate Limiting ป้องกัน API จากการถูก abuse เช่น DDoS attacks, brute force และ excessive usage มีหลาย algorithms ที่ใช้กัน

## Token Bucket Algorithm

```crystal
require "kemal"

class TokenBucket
  property tokens : Float64
  property last_refill : Time
  
  def initialize(@capacity : Float64, @refill_rate : Float64)
    @tokens = @capacity
    @last_refill = Time.local
  end
  
  def consume(count : Float64 = 1.0) : Bool
    refill
    
    if @tokens >= count
      @tokens -= count
      true
    else
      false
    end
  end
  
  def tokens_available : Float64
    refill
    @tokens
  end
  
  private def refill
    now = Time.local
    elapsed = (now - @last_refill).total_seconds
    new_tokens = elapsed * @refill_rate
    @tokens = [@tokens + new_tokens, @capacity].min
    @last_refill = now
  end
end

class TokenBucketRateLimiter
  def initialize(
    @capacity : Float64 = 100.0,
    @refill_rate : Float64 = 10.0  # tokens per second
  )
    @buckets = {} of String => TokenBucket
    @mutex = Mutex.new
  end
  
  def allow?(key : String, cost : Float64 = 1.0) : {Bool, Float64}
    @mutex.synchronize do
      bucket = @buckets[key] ||= TokenBucket.new(@capacity, @refill_rate)
      allowed = bucket.consume(cost)
      {allowed, bucket.tokens_available}
    end
  end
  
  def cleanup(older_than : Time::Span = 1.hour)
    @mutex.synchronize do
      cutoff = Time.local - older_than
      @buckets.reject! { |_, b| b.last_refill < cutoff }
    end
  end
end

# Global rate limiter
RATE_LIMITER = TokenBucketRateLimiter.new(capacity: 100.0, refill_rate: 10.0)

# Cleanup background task
spawn do
  loop do
    sleep 10.minutes
    RATE_LIMITER.cleanup
  end
end

# Middleware
before_all "/api/*" do |env|
  client_ip = env.request.remote_address.to_s.split(":").first
  allowed, remaining = RATE_LIMITER.allow?(client_ip)
  
  env.response.headers["X-RateLimit-Limit"] = "100"
  env.response.headers["X-RateLimit-Remaining"] = remaining.to_i.to_s
  env.response.headers["X-RateLimit-Reset"] = (Time.local + 10.seconds).to_unix.to_s
  
  unless allowed
    env.response.status_code = 429
    env.response.content_type = "application/json"
    env.response.headers["Retry-After"] = "10"
    env.response.print({"error" => "Rate limit exceeded", "retry_after" => 10}.to_json)
    next
  end
end

Kemal.run
```

## Sliding Window Algorithm

```crystal
require "kemal"

class SlidingWindowRateLimiter
  def initialize(@limit : Int32, @window : Time::Span)
    @requests = Hash(String, Array(Time)).new { |h, k| h[k] = [] of Time }
    @mutex = Mutex.new
  end
  
  def check(key : String) : {Bool, Int32, Time?}
    @mutex.synchronize do
      now = Time.local
      cutoff = now - @window
      
      # ลบ requests เก่า
      @requests[key].reject! { |t| t < cutoff }
      
      count = @requests[key].size
      
      if count < @limit
        @requests[key] << now
        {true, @limit - count - 1, nil}
      else
        # คำนวณเวลาที่ earliest request จะ expire
        reset_at = @requests[key].first? ? @requests[key].first + @window : nil
        {false, 0, reset_at}
      end
    end
  end
  
  def reset(key : String)
    @mutex.synchronize do
      @requests.delete(key)
    end
  end
end

# ตัวอย่าง: 60 requests ต่อ 1 นาที
SLIDING_LIMITER = SlidingWindowRateLimiter.new(60, 1.minute)

before_all "/api/*" do |env|
  key = env.request.headers["X-Forwarded-For"]? ||
        env.request.remote_address.to_s
  
  allowed, remaining, reset_at = SLIDING_LIMITER.check(key)
  
  env.response.headers["X-RateLimit-Limit"] = "60"
  env.response.headers["X-RateLimit-Remaining"] = remaining.to_s
  
  if reset_at
    env.response.headers["X-RateLimit-Reset"] = reset_at.to_unix.to_s
  end
  
  unless allowed
    env.response.status_code = 429
    env.response.content_type = "application/json"
    
    retry_after = reset_at ? (reset_at - Time.local).total_seconds.ceil.to_i : 60
    env.response.headers["Retry-After"] = retry_after.to_s
    
    env.response.print({
      "error" => "Too Many Requests",
      "message" => "Rate limit exceeded. Please wait #{retry_after} seconds.",
      "retry_after" => retry_after
    }.to_json)
    next
  end
end
```

## Fixed Window Algorithm

```crystal
class FixedWindowRateLimiter
  def initialize(@limit : Int32, @window : Time::Span)
    @windows = {} of String => {Int32, Time}
    @mutex = Mutex.new
  end
  
  def check(key : String) : {Bool, Int32}
    @mutex.synchronize do
      now = Time.local
      
      if entry = @windows[key]?
        count, window_start = entry
        
        if now - window_start < @window
          # ยังอยู่ใน window เดิม
          if count < @limit
            @windows[key] = {count + 1, window_start}
            {true, @limit - count - 1}
          else
            {false, 0}
          end
        else
          # เริ่ม window ใหม่
          @windows[key] = {1, now}
          {true, @limit - 1}
        end
      else
        @windows[key] = {1, now}
        {true, @limit - 1}
      end
    end
  end
end
```

## Redis-backed Rate Limiting

```crystal
require "kemal"
require "redis"

class RedisRateLimiter
  def initialize(
    @redis : Redis::Client,
    @limit : Int32,
    @window : Int32  # seconds
  )
  end
  
  def check(key : String) : {Bool, Int32, Int32}
    redis_key = "ratelimit:#{key}"
    
    # Lua script สำหรับ atomic operation
    lua_script = <<-LUA
      local key = KEYS[1]
      local limit = tonumber(ARGV[1])
      local window = tonumber(ARGV[2])
      
      local current = redis.call('GET', key)
      
      if current == false then
        redis.call('SETEX', key, window, 1)
        return {1, limit - 1, window}
      end
      
      current = tonumber(current)
      
      if current < limit then
        redis.call('INCR', key)
        local ttl = redis.call('TTL', key)
        return {1, limit - current - 1, ttl}
      else
        local ttl = redis.call('TTL', key)
        return {0, 0, ttl}
      end
    LUA
    
    result = @redis.eval(
      lua_script,
      keys: [redis_key],
      args: [@limit.to_s, @window.to_s]
    ).as(Array)
    
    allowed = result[0].as(Int64) == 1_i64
    remaining = result[1].as(Int64).to_i32
    reset_in = result[2].as(Int64).to_i32
    
    {allowed, remaining, reset_in}
  rescue ex
    # Redis unavailable: fail open (อนุญาต request)
    puts "Rate limiter error: #{ex.message}"
    {true, -1, 0}
  end
end

# ใช้งาน
redis_limiter = RedisRateLimiter.new(
  Redis::Client.new(host: "localhost", port: 6379),
  limit: 100,
  window: 60  # 1 minute
)

before_all "/api/*" do |env|
  key = env.request.headers["X-API-Key"]? ||
        env.request.remote_address.to_s
  
  allowed, remaining, reset_in = redis_limiter.check(key)
  
  env.response.headers["X-RateLimit-Limit"] = "100"
  env.response.headers["X-RateLimit-Remaining"] = remaining.to_s
  env.response.headers["X-RateLimit-Reset"] = (Time.local + reset_in.seconds).to_unix.to_s
  
  unless allowed
    env.response.status_code = 429
    env.response.content_type = "application/json"
    env.response.headers["Retry-After"] = reset_in.to_s
    env.response.print({
      "error" => "Rate limit exceeded",
      "retry_after" => reset_in,
      "limit" => 100
    }.to_json)
    next
  end
end
```

## Per-Endpoint Rate Limiting

```crystal
require "kemal"

class MultiRateLimiter
  def initialize
    @limiters = {} of String => SlidingWindowRateLimiter
  end
  
  def add(name : String, limit : Int32, window : Time::Span)
    @limiters[name] = SlidingWindowRateLimiter.new(limit, window)
    self
  end
  
  def check(name : String, key : String) : {Bool, Int32, Time?}
    @limiters[name]?.try(&.check(key)) || {true, 999, nil}
  end
end

LIMITERS = MultiRateLimiter.new
  .add("global", 1000, 1.minute)
  .add("auth", 5, 15.minutes)
  .add("search", 30, 1.minute)
  .add("upload", 10, 1.hour)

def rate_limit_check(env, limiter_name : String)
  key = env.request.remote_address.to_s
  allowed, remaining, reset_at = LIMITERS.check(limiter_name, key)
  
  env.response.headers["X-RateLimit-Remaining"] = remaining.to_s
  
  unless allowed
    env.response.status_code = 429
    env.response.content_type = "application/json"
    retry_after = reset_at ? (reset_at - Time.local).total_seconds.ceil.to_i : 60
    env.response.headers["Retry-After"] = retry_after.to_s
    env.response.print({"error" => "Rate limit exceeded"}.to_json)
    false
  else
    true
  end
end

# Routes
post "/api/auth/login" do |env|
  next unless rate_limit_check(env, "auth")
  # handle login...
  {"token" => "xxx"}.to_json
end

get "/api/search" do |env|
  next unless rate_limit_check(env, "search")
  # handle search...
  {"results" => []}.to_json
end

post "/api/upload" do |env|
  next unless rate_limit_check(env, "upload")
  # handle upload...
  {"uploaded" => true}.to_json
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: API Key Rate Limiting

```crystal
require "kemal"
require "json"

# API Key-based rate limiting
# Premium users ได้ limit สูงกว่า

class ApiKeyRateLimiter
  struct Tier
    property limit : Int32
    property window : Time::Span
    
    def initialize(@limit, @window)
    end
  end
  
  TIERS = {
    "free" => Tier.new(100, 1.hour),
    "basic" => Tier.new(1000, 1.hour),
    "premium" => Tier.new(10000, 1.hour),
    "enterprise" => Tier.new(100000, 1.hour),
  }
  
  def initialize
    @buckets = {} of String => TokenBucket
    @mutex = Mutex.new
  end
  
  def check(api_key : String, tier : String = "free") : {Bool, Int32, String}
    tier_config = TIERS[tier]? || TIERS["free"]
    
    @mutex.synchronize do
      bucket = @buckets[api_key] ||= TokenBucket.new(
        tier_config.limit.to_f,
        tier_config.limit.to_f / tier_config.window.total_seconds
      )
      
      allowed = bucket.consume
      remaining = bucket.tokens_available.to_i
      
      {allowed, remaining, tier}
    end
  end
end

API_KEYS = {
  "key_free_123" => "free",
  "key_basic_456" => "basic",
  "key_premium_789" => "premium",
}

LIMITER = ApiKeyRateLimiter.new

before_all "/api/*" do |env|
  api_key = env.request.headers["X-API-Key"]?
  
  unless api_key
    halt env, status_code: 401,
      response: {error: "API key required", header: "X-API-Key"}.to_json
  end
  
  tier = API_KEYS[api_key]?
  
  unless tier
    halt env, status_code: 401,
      response: {error: "Invalid API key"}.to_json
  end
  
  allowed, remaining, actual_tier = LIMITER.check(api_key, tier)
  
  env.response.headers["X-RateLimit-Tier"] = actual_tier
  env.response.headers["X-RateLimit-Remaining"] = remaining.to_s
  
  unless allowed
    halt env, status_code: 429,
      response: {
        error: "Rate limit exceeded",
        tier: actual_tier,
        upgrade_url: "https://api.example.com/pricing"
      }.to_json
  end
end

get "/api/v1/data" do |env|
  env.response.content_type = "application/json"
  {"data" => "Hello from protected API", "time" => Time.local.to_rfc3339}.to_json
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Token Bucket**: ยืดหยุ่น รองรับ burst traffic
2. **Sliding Window**: แม่นยำ ไม่มี edge cases ที่ window boundary
3. **Fixed Window**: เรียบง่าย แต่มี burst problem ที่ window boundary
4. **Redis-backed**: distributed rate limiting สำหรับ multi-server
5. **Per-Endpoint**: limit ต่าง endpoint ต่างกัน
6. **Response Headers**: X-RateLimit-Limit, Remaining, Reset
7. **Retry-After**: บอก client ว่ารอนานแค่ไหน
8. **API Key Tiers**: limit ต่าง tier ต่างกัน

Rate limiting ที่ดีต้องมี: fail-open fallback, proper headers, และ clear error messages
