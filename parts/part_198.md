# Part 198: Performance Best Practices ใน Crystal

## บทนำ

Crystal เร็วมาก แต่ยังมีโอกาสเพิ่ม performance ด้วย Crystal-specific techniques การ profile ก่อน optimize เสมอ อย่า "premature optimization"

## Crystal-Specific Optimizations

```crystal
# 1. ใช้ struct สำหรับ value objects ที่สร้างบ่อย
# BAD: class allocates on heap ทุกครั้ง
class Point
  property x, y : Float64
  def initialize(@x, @y); end
end

# GOOD: struct allocates on stack
struct Point
  property x, y : Float64
  def initialize(@x, @y); end
end

# 2. String.build แทน concatenation
# BAD
def build_response(items : Array(String)) : String
  result = ""
  items.each { |item| result += item + "\n" }
  result
end

# GOOD
def build_response(items : Array(String)) : String
  String.build do |sb|
    items.each { |item| sb << item << "\n" }
  end
end

# 3. Pre-allocate collections
# BAD
def process_numbers(n : Int32) : Array(Int32)
  result = [] of Int32  # realloc เมื่อ grow
  n.times { |i| result << i * 2 }
  result
end

# GOOD
def process_numbers(n : Int32) : Array(Int32)
  Array.new(n) { |i| i * 2 }  # allocate once
end

# 4. Use lazy evaluation
# BAD: สร้าง intermediate arrays
large_data = (1..1_000_000).to_a
result = large_data.map { |x| x * 2 }.select { |x| x > 100 }.first(10)

# GOOD: lazy - ไม่สร้าง intermediate arrays
result = (1..1_000_000)
  .lazy
  .map { |x| x * 2 }
  .select { |x| x > 100 }
  .first(10)
```

## Profiling Workflow

```crystal
require "benchmark"
require "gc"

# Step 1: Identify hotspots
module Profiler
  @@timings : Hash(String, Array(Float64)) = {} of String => Array(Float64)

  def self.measure(name : String, &block)
    start = Time.monotonic
    result = block.call
    elapsed = (Time.monotonic - start).total_microseconds

    @@timings[name] ||= [] of Float64
    @@timings[name] << elapsed
    result
  end

  def self.report
    puts "=== Performance Report ==="
    @@timings.each do |name, times|
      avg = times.sum / times.size
      max = times.max
      p95 = times.sort[(times.size * 0.95).to_i]
      puts "#{name}:"
      puts "  avg: #{avg.round(1)}µs  max: #{max.round(1)}µs  p95: #{p95.round(1)}µs  calls: #{times.size}"
    end
  end
end

# Step 2: Benchmark alternatives
Benchmark.ips do |x|
  x.report("Array#map + select") do
    (1..10_000).to_a.map { |x| x * 2 }.select { |x| x > 100 }
  end

  x.report("Lazy chain") do
    (1..10_000).lazy.map { |x| x * 2 }.select { |x| x > 100 }.to_a
  end

  x.report("Single pass loop") do
    result = [] of Int32
    (1..10_000).each do |x|
      doubled = x * 2
      result << doubled if doubled > 100
    end
    result
  end

  x.compare!
end
```

## Caching Strategies

```crystal
require "redis"

# Multi-level caching: L1 (memory) + L2 (Redis)
class MultiLevelCache(T)
  @l1_cache : Hash(String, {value: T, expires_at: Time}) = {} of String => {value: T, expires_at: Time}
  @l1_max_size : Int32
  @l1_ttl : Time::Span
  @l2_ttl : Time::Span
  @mutex = Mutex.new

  def initialize(
    @redis : Redis::Client,
    @l1_max_size : Int32 = 1000,
    @l1_ttl : Time::Span = 1.minute,
    @l2_ttl : Time::Span = 1.hour
  )
  end

  def get(key : String, &fetch : -> T) : T
    # L1 cache check
    if entry = get_l1(key)
      return entry
    end

    # L2 cache check
    if data = @redis.get("cache:#{key}")
      value = T.from_json(data)
      set_l1(key, value)
      return value
    end

    # Fetch from source
    value = fetch.call
    set_l1(key, value)
    @redis.setex("cache:#{key}", @l2_ttl.total_seconds.to_i, value.to_json)
    value
  end

  def invalidate(key : String)
    @mutex.synchronize { @l1_cache.delete(key) }
    @redis.del("cache:#{key}")
  end

  private def get_l1(key : String) : T?
    @mutex.synchronize do
      if entry = @l1_cache[key]?
        if Time.local < entry[:expires_at]
          return entry[:value]
        end
        @l1_cache.delete(key)
      end
    end
    nil
  end

  private def set_l1(key : String, value : T)
    @mutex.synchronize do
      evict_l1 if @l1_cache.size >= @l1_max_size
      @l1_cache[key] = {value: value, expires_at: Time.local + @l1_ttl}
    end
  end

  private def evict_l1
    oldest_key = @l1_cache.min_by { |_, v| v[:expires_at] }.first
    @l1_cache.delete(oldest_key)
  end
end

# ใช้งาน
cache = MultiLevelCache(User).new(redis, l1_max_size: 500)

user = cache.get("user:123") do
  UserRepository.find(123)  # เรียกแค่เมื่อ cache miss
end
```

## Database Optimization

```crystal
# N+1 query problem และการแก้
# BAD: N+1 queries
def get_orders_with_users_bad(order_ids : Array(Int64))
  orders = Order.where(id: order_ids)
  orders.each do |order|
    user = User.find(order.user_id)  # N queries!
    puts "#{user.name}: #{order.total}"
  end
end

# GOOD: JOIN query
def get_orders_with_users_good(order_ids : Array(Int64))
  db.query_all(<<-SQL, order_ids, as: {order_id: Int64, total: Float64, user_name: String})
    SELECT o.id, o.total, u.name
    FROM orders o
    JOIN users u ON u.id = o.user_id
    WHERE o.id = ANY($1)
    ORDER BY o.id
  SQL
end

# GOOD: Batch loading
def get_orders_with_users_batch(order_ids : Array(Int64))
  orders = Order.where(id: order_ids)
  user_ids = orders.map(&.user_id).uniq
  users = User.where(id: user_ids).index_by(&.id)  # 1 query

  orders.each do |order|
    user = users[order.user_id]?
    puts "#{user.try(&.name)}: #{order.total}"
  end
end

# Connection pooling
DB_POOL_SIZE = (ENV["DB_POOL_SIZE"]? || "10").to_i
DB_POOL = DB.open("#{ENV["DATABASE_URL"]}?max_pool_size=#{DB_POOL_SIZE}&initial_pool_size=2")
```

## Memory สำหรับ Performance

```crystal
# Object pooling สำหรับ high-throughput paths
class BufferPool
  BUFFER_SIZE = 16 * 1024  # 16KB

  @pool : Array(IO::Memory) = [] of IO::Memory
  @mutex = Mutex.new

  def acquire : IO::Memory
    @mutex.synchronize { @pool.pop? } || IO::Memory.new(BUFFER_SIZE)
  end

  def release(buf : IO::Memory)
    buf.clear
    @mutex.synchronize { @pool << buf if @pool.size < 100 }
  end
end

BUFFER_POOL = BufferPool.new

def handle_request(context : HTTP::Server::Context)
  buf = BUFFER_POOL.acquire
  begin
    build_response(buf, context)
    context.response.print(buf.to_s)
  ensure
    BUFFER_POOL.release(buf)
  end
end

private def build_response(io : IO, context : HTTP::Server::Context)
  io << "{\"status\":\"ok\",\"time\":\""
  io << Time.local.to_rfc3339
  io << "\"}"
end
```

## แบบฝึกหัด

1. Profile web application และ identify top 3 slowest endpoints
2. Implement multi-level cache สำหรับ database queries ที่ expensive
3. ลด memory usage ของ high-traffic endpoint โดยใช้ buffer pool
4. วัด throughput ก่อนและหลัง optimization ด้วย wrk หรือ ab

## สรุป

Performance Best Practices ใน Crystal:
- **Profile first**: วัดก่อน optimize อย่า guess
- **struct**: ลด heap allocations สำหรับ value objects
- **String.build**: ลด string reallocations
- **Pre-allocate**: ใช้ `Array.new(n)` แทน empty array
- **Lazy evaluation**: ลด intermediate collections
- **Multi-level cache**: L1 memory + L2 Redis ลด latency
- **N+1 elimination**: JOIN หรือ batch loading
- **Connection pooling**: reuse DB connections
- **Object pooling**: reuse expensive objects
- **--release**: compile optimizations เสมอใน production
