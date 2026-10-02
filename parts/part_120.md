# Part 120: Concurrency Patterns - รูปแบบการเขียนโปรแกรมแบบ Concurrent

## บทนำ

ในบทนี้เราจะเรียนรู้ concurrency patterns ที่ใช้บ่อยใน Crystal ได้แก่ Pipeline Pattern, Fan-Out/Fan-In, Rate Limiting ด้วย Channel, และ Semaphore Pattern

## Pipeline Pattern

Pipeline เป็น pattern ที่ข้อมูลไหลผ่านชุดของ stages โดยแต่ละ stage ประมวลผลและส่งต่อไปยัง stage ถัดไป

### Pipeline พื้นฐาน

```crystal
# Stage 1: Generator
def generate(items : Array(T)) : Channel(T) forall T
  out = Channel(T).new(len(items))
  spawn do
    items.each { |item| out.send(item) }
    out.close
  end
  out
end

# Stage 2: Transform
def transform(input : Channel(T), &block : T -> R) : Channel(R) forall T, R
  out = Channel(R).new(100)
  spawn do
    while item = input.receive?
      out.send(block.call(item))
    end
    out.close
  end
  out
end

# Stage 3: Filter
def filter_stage(input : Channel(T), &predicate : T -> Bool) : Channel(T) forall T
  out = Channel(T).new(100)
  spawn do
    while item = input.receive?
      out.send(item) if predicate.call(item)
    end
    out.close
  end
  out
end

# ใช้งาน
numbers = Channel(Int32).new(20)
spawn do
  (1..20).each { |n| numbers.send(n) }
  numbers.close
end

# Pipeline: numbers -> doubled -> filtered (even) -> print
doubled = transform(numbers) { |n| n * 2 }
evens = filter_stage(doubled) { |n| n % 4 == 0 }

while result = evens.receive?
  print "#{result} "
end
puts
```

### Multi-Stage Pipeline

```crystal
# Pipeline สำหรับประมวลผล log files
struct LogEntry
  property raw : String
  property level : String
  property message : String
  property timestamp : Time
  
  def initialize(@raw, @level, @message, @timestamp)
  end
end

# Stage 1: Parse
def parse_logs(input : Channel(String)) : Channel(LogEntry)
  out = Channel(LogEntry).new(100)
  
  spawn do
    while line = input.receive?
      # จำลอง parsing
      parts = line.split(" ", 3)
      if parts.size >= 3
        entry = LogEntry.new(
          raw: line,
          level: parts[0],
          message: parts[2],
          timestamp: Time.local
        )
        out.send(entry)
      end
    end
    out.close
  end
  
  out
end

# Stage 2: Enrich
def enrich_logs(input : Channel(LogEntry)) : Channel(LogEntry)
  out = Channel(LogEntry).new(100)
  
  spawn do
    while entry = input.receive?
      # เพิ่มข้อมูลเพิ่มเติม
      out.send(entry)
    end
    out.close
  end
  
  out
end

# Stage 3: Filter (errors only)
def filter_errors(input : Channel(LogEntry)) : Channel(LogEntry)
  out = Channel(LogEntry).new(100)
  
  spawn do
    while entry = input.receive?
      out.send(entry) if entry.level == "ERROR"
    end
    out.close
  end
  
  out
end

# Stage 4: Format
def format_output(input : Channel(LogEntry)) : Channel(String)
  out = Channel(String).new(100)
  
  spawn do
    while entry = input.receive?
      out.send("[#{entry.timestamp}] #{entry.level}: #{entry.message}")
    end
    out.close
  end
  
  out
end

# เชื่อม pipeline
raw_logs = Channel(String).new(100)

spawn do
  [
    "ERROR 2024-01-01 Database connection failed",
    "INFO 2024-01-01 Server started",
    "ERROR 2024-01-01 Out of memory",
    "WARN 2024-01-01 Slow query detected",
    "ERROR 2024-01-01 Authentication failed",
  ].each { |l| raw_logs.send(l) }
  raw_logs.close
end

pipeline = format_output(filter_errors(enrich_logs(parse_logs(raw_logs))))

while result = pipeline.receive?
  puts result
end
```

## Fan-Out/Fan-In Pattern

### Fan-Out: กระจายงานให้หลาย workers

```crystal
# Fan-Out: ส่งงานให้ worker หลายตัว
def fan_out(input : Channel(T), num_workers : Int32, &worker : T -> R) : Array(Channel(R)) forall T, R
  outputs = Array(Channel(R)).new(num_workers) { Channel(R).new(100) }
  
  num_workers.times do |i|
    spawn do
      while item = input.receive?
        outputs[i].send(worker.call(item))
      end
      outputs[i].close
    end
  end
  
  outputs
end

# ทดสอบ Fan-Out
jobs = Channel(Int32).new(20)
spawn do
  20.times { |i| jobs.send(i + 1) }
  jobs.close
end

# กระจายงานให้ 4 workers
worker_outputs = fan_out(jobs, 4) do |n|
  sleep rand(0.01..0.05).seconds
  n * n
end

# Fan-In: รวมผลลัพธ์
```

### Fan-In: รวมผลลัพธ์จากหลาย channels

```crystal
# Fan-In: รวม channels หลายอันเป็นอันเดียว
def fan_in(channels : Array(Channel(T))) : Channel(T) forall T
  output = Channel(T).new(100)
  remaining = Atomic(Int32).new(channels.size)
  
  channels.each do |ch|
    spawn do
      while item = ch.receive?
        output.send(item)
      end
      # ปิด output เมื่อทุก input channel ปิด
      if remaining.sub(1) == 1
        output.close
      end
    end
  end
  
  output
end

# Fan-Out + Fan-In combined
jobs = Channel(Int32).new(30)
spawn do
  30.times { |i| jobs.send(i + 1) }
  jobs.close
end

# Fan-Out: กระจายงาน
worker_channels = (1..4).map do |worker_id|
  out_ch = Channel(String).new(100)
  
  spawn do
    while n = jobs.receive?
      sleep rand(0.01..0.05).seconds
      out_ch.send("Worker #{worker_id}: #{n}^2 = #{n*n}")
    end
    out_ch.close
  end
  
  out_ch
end

# Fan-In: รวมผลลัพธ์
merged = fan_in(worker_channels)

count = 0
while result = merged.receive?
  puts result
  count += 1
end
puts "Total results: #{count}"
```

### Scatter-Gather Pattern

```crystal
# Scatter-Gather: ส่งงานเดิมให้ทุก worker แล้วรอทุก worker เสร็จ
class ScatterGather(T, R)
  def initialize(@workers : Array(Proc(T, R)))
  end
  
  def execute(input : T) : Array(R)
    results_channel = Channel(R).new(@workers.size)
    
    @workers.each do |worker|
      spawn do
        results_channel.send(worker.call(input))
      end
    end
    
    Array(R).new(@workers.size) { results_channel.receive }
  end
end

# ตัวอย่าง: ค้นหาในหลาย databases พร้อมกัน
databases = [
  ->(query : String) { sleep 0.1.seconds; "DB1: ผลจาก #{query}" },
  ->(query : String) { sleep 0.2.seconds; "DB2: ผลจาก #{query}" },
  ->(query : String) { sleep 0.15.seconds; "DB3: ผลจาก #{query}" },
]

scatter = ScatterGather(String, String).new(databases)
results = scatter.execute("สินค้า")
results.each { |r| puts r }
```

## Rate Limiting ด้วย Channel

### Token Bucket Algorithm

```crystal
class RateLimiter
  def initialize(@rate : Int32, @burst : Int32)
    @tokens = Channel(Nil).new(@burst)
    # เติม burst tokens
    @burst.times { @tokens.send(nil) }
    
    # Refill tokens ตาม rate
    spawn do
      loop do
        sleep (1.0 / @rate).seconds
        # เพิ่ม token ถ้ายังไม่เต็ม
        select
        when @tokens.send(nil)
          # token เพิ่มสำเร็จ
        else
          # bucket เต็มแล้ว
        end
      end
    end
  end
  
  def acquire
    @tokens.receive
  end
  
  def try_acquire : Bool
    select
    when @tokens.receive?
      true
    else
      false
    end
  end
  
  def with_rate_limit(&block)
    acquire
    block.call
  end
end

# ทดสอบ Rate Limiter
# 5 requests/second, burst 10
limiter = RateLimiter.new(5, 10)

start = Time.monotonic

20.times do |i|
  spawn do
    limiter.with_rate_limit do
      elapsed = (Time.monotonic - start).total_milliseconds
      puts "Request #{i + 1} at #{elapsed.round(0)}ms"
    end
  end
end

sleep 5.seconds
```

### Sliding Window Rate Limiter

```crystal
class SlidingWindowLimiter
  def initialize(@limit : Int32, @window : Time::Span)
    @timestamps = Deque(Time).new
    @mutex = Mutex.new
  end
  
  def allow? : Bool
    @mutex.synchronize do
      now = Time.local
      cutoff = now - @window
      
      # ลบ timestamps ที่เก่าเกินไป
      while !@timestamps.empty? && @timestamps.first < cutoff
        @timestamps.shift
      end
      
      if @timestamps.size < @limit
        @timestamps.push(now)
        true
      else
        false
      end
    end
  end
  
  def wait_and_proceed
    until allow?
      sleep 10.milliseconds
    end
  end
end

limiter = SlidingWindowLimiter.new(5, 1.second)

15.times do |i|
  spawn do
    if limiter.allow?
      puts "Request #{i + 1}: อนุญาต"
    else
      puts "Request #{i + 1}: ปฏิเสธ (rate limit)"
    end
  end
end

sleep 0.2.seconds
puts "\nรอ 1 วินาที..."
sleep 1.second

5.times do |i|
  if limiter.allow?
    puts "Request #{i + 16}: อนุญาต"
  end
end
```

### Leaky Bucket Algorithm

```crystal
class LeakyBucket
  def initialize(@capacity : Float64, @leak_rate : Float64)
    @current = 0.0
    @last_leak = Time.monotonic
    @mutex = Mutex.new
  end
  
  def pour(amount : Float64 = 1.0) : Bool
    @mutex.synchronize do
      now = Time.monotonic
      elapsed = (now - @last_leak).total_seconds
      
      # น้ำรั่วออก
      @current = [@current - elapsed * @leak_rate, 0.0].max
      @last_leak = now
      
      if @current + amount <= @capacity
        @current += amount
        true
      else
        false
      end
    end
  end
end

bucket = LeakyBucket.new(10.0, 2.0) # capacity 10, leak 2/sec

20.times do |i|
  if bucket.pour
    puts "Request #{i + 1}: accepted"
  else
    puts "Request #{i + 1}: dropped (bucket full)"
  end
  sleep 0.1.seconds
end
```

## Semaphore Pattern

```crystal
class Semaphore
  def initialize(@permits : Int32)
    @channel = Channel(Nil).new(@permits)
    @permits.times { @channel.send(nil) }
    @waiting = Atomic(Int32).new(0)
  end
  
  def acquire
    @waiting.add(1)
    @channel.receive
    @waiting.sub(1)
  end
  
  def release
    @channel.send(nil)
  end
  
  def try_acquire : Bool
    select
    when @channel.receive?
      true
    else
      false
    end
  end
  
  def available_permits : Int32
    # ประมาณค่า
    @permits - @waiting.get
  end
  
  def synchronize(&block)
    acquire
    begin
      block.call
    ensure
      release
    end
  end
end

# Database Connection Pool ด้วย Semaphore
class DatabasePool
  def initialize(@max_connections : Int32)
    @semaphore = Semaphore.new(@max_connections)
    @connections = [] of String
    @max_connections.times { |i| @connections << "connection_#{i}" }
    @available = @connections.dup
    @mutex = Mutex.new
  end
  
  def with_connection(&block : String ->)
    @semaphore.acquire
    conn = @mutex.synchronize { @available.pop? }
    
    if conn
      begin
        block.call(conn)
      ensure
        @mutex.synchronize { @available.push(conn) }
        @semaphore.release
      end
    else
      @semaphore.release
      raise "No available connection"
    end
  end
end

pool = DatabasePool.new(3)

10.times do |i|
  spawn do
    pool.with_connection do |conn|
      puts "Task #{i} ใช้ #{conn}"
      sleep rand(0.1..0.5).seconds
      puts "Task #{i} เสร็จ"
    end
  end
end

sleep 3.seconds
```

## Circuit Breaker Pattern

```crystal
class CircuitBreaker
  enum State
    Closed   # ปกติ
    Open     # ปิดกั้น (เกิด error เยอะ)
    HalfOpen # ทดสอบว่ากลับมาปกติแล้วหรือยัง
  end
  
  getter state : State = State::Closed
  getter failure_count : Int32 = 0
  
  def initialize(
    @failure_threshold : Int32 = 5,
    @recovery_timeout : Time::Span = 30.seconds,
    @success_threshold : Int32 = 2
  )
    @last_failure_time : Time? = nil
    @success_count = 0
    @mutex = Mutex.new
  end
  
  def call(&block : -> T) : T forall T
    @mutex.synchronize do
      case @state
      when .open?
        if should_attempt_recovery?
          @state = State::HalfOpen
          puts "Circuit Breaker: Half-Open (ทดสอบ)"
        else
          raise "Circuit is OPEN - ไม่สามารถดำเนินการได้"
        end
      end
    end
    
    begin
      result = block.call
      on_success
      result
    rescue ex
      on_failure
      raise ex
    end
  end
  
  private def on_success
    @mutex.synchronize do
      case @state
      when .half_open?
        @success_count += 1
        if @success_count >= @success_threshold
          @state = State::Closed
          @failure_count = 0
          @success_count = 0
          puts "Circuit Breaker: Closed (กลับมาปกติ)"
        end
      when .closed?
        @failure_count = 0
      end
    end
  end
  
  private def on_failure
    @mutex.synchronize do
      @failure_count += 1
      @last_failure_time = Time.local
      
      if @failure_count >= @failure_threshold
        @state = State::Open
        puts "Circuit Breaker: Open (เกิด #{@failure_count} errors)"
      end
    end
  end
  
  private def should_attempt_recovery?
    if last = @last_failure_time
      Time.local - last > @recovery_timeout
    else
      true
    end
  end
end

# ทดสอบ Circuit Breaker
cb = CircuitBreaker.new(failure_threshold: 3, recovery_timeout: 2.seconds)

15.times do |i|
  begin
    cb.call do
      # 60% โอกาส fail
      raise "Service error" if rand < 0.6
      puts "Success #{i + 1}"
    end
  rescue ex
    puts "Error #{i + 1}: #{ex.message} (State: #{cb.state})"
  end
  sleep 0.2.seconds
end
```

## Bulkhead Pattern

```crystal
# Bulkhead: แยก resource pools เพื่อป้องกัน cascading failures
class Bulkhead
  def initialize(@name : String, @max_concurrent : Int32, @max_queue : Int32)
    @semaphore = Semaphore.new(@max_concurrent)
    @queue = Channel(Proc(Nil)).new(@max_queue)
    @running = Atomic(Int32).new(0)
    start_workers
  end
  
  def execute(&block)
    # ตรวจสอบ queue capacity
    select
    when @queue.send(block)
      true
    else
      puts "#{@name}: Queue เต็ม! ปฏิเสธงาน"
      false
    end
  end
  
  private def start_workers
    @max_concurrent.times do
      spawn do
        while job = @queue.receive?
          @semaphore.synchronize do
            @running.add(1)
            begin
              job.call
            ensure
              @running.sub(1)
            end
          end
        end
      end
    end
  end
end

# แยก resource pool สำหรับ services ต่างๆ
payment_bulkhead = Bulkhead.new("Payment", 5, 20)
inventory_bulkhead = Bulkhead.new("Inventory", 10, 50)

20.times do |i|
  payment_bulkhead.execute do
    sleep rand(0.1..0.3).seconds
    puts "Payment #{i} processed"
  end
  
  inventory_bulkhead.execute do
    sleep rand(0.05..0.1).seconds
    puts "Inventory #{i} checked"
  end
end

sleep 5.seconds
```

## Throttle Pattern

```crystal
# Throttle: จำกัดความถี่การเรียก function
class Throttler(T, R)
  def initialize(@min_interval : Time::Span)
    @last_called : Time? = nil
    @mutex = Mutex.new
  end
  
  def throttle(arg : T, &block : T -> R) : R?
    @mutex.synchronize do
      now = Time.local
      
      if last = @last_called
        if now - last < @min_interval
          return nil # throttled
        end
      end
      
      @last_called = now
      block.call(arg)
    end
  end
end

# Debounce: รอให้หยุดเรียกก่อน ถึงทำงาน
class Debouncer(T)
  def initialize(@wait : Time::Span, &@action : T ->)
    @timer_channel = Channel(T?).new(1)
    @last_value : T? = nil
    start_debounce_loop
  end
  
  def call(value : T)
    @last_value = value
    # ยกเลิก timer เดิม แล้วเริ่มใหม่
    select
    when @timer_channel.send(value)
    else
      # เพิ่ม non-blocking
    end
  end
  
  private def start_debounce_loop
    spawn do
      loop do
        if value = @timer_channel.receive?
          # รอ debounce period
          sleep @wait
          
          # ตรวจสอบว่าไม่มีการเรียกใหม่
          if @last_value == value
            @action.call(value.not_nil!)
          end
        end
      end
    end
  end
end

# ทดสอบ Debounce (search input)
debouncer = Debouncer(String).new(0.3.seconds) do |query|
  puts "ค้นหา: #{query}"
end

puts "จำลองการพิมพ์..."
debouncer.call("s")
sleep 0.05.seconds
debouncer.call("se")
sleep 0.05.seconds
debouncer.call("sea")
sleep 0.05.seconds
debouncer.call("sear")
sleep 0.05.seconds
debouncer.call("searc")
sleep 0.05.seconds
debouncer.call("search")
sleep 0.5.seconds # รอ debounce
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Parallel HTTP Fetcher

```crystal
require "http/client"

class ParallelFetcher
  struct FetchResult
    property url : String
    property status : Int32
    property body_size : Int32
    property elapsed_ms : Float64
    property error : String?
    
    def initialize(@url, @status, @body_size, @elapsed_ms, @error = nil)
    end
  end
  
  def initialize(@concurrency : Int32 = 5)
    @semaphore = Semaphore.new(@concurrency)
    @results = Channel(FetchResult).new(100)
  end
  
  def fetch_all(urls : Array(String)) : Array(FetchResult)
    urls.each do |url|
      spawn do
        @semaphore.synchronize do
          start = Time.monotonic
          begin
            response = HTTP::Client.get(url)
            elapsed = (Time.monotonic - start).total_milliseconds
            @results.send(FetchResult.new(url, response.status_code, response.body.size, elapsed))
          rescue ex
            elapsed = (Time.monotonic - start).total_milliseconds
            @results.send(FetchResult.new(url, 0, 0, elapsed, ex.message))
          end
        end
      end
    end
    
    Array(FetchResult).new(urls.size) { @results.receive }
  end
end

fetcher = ParallelFetcher.new(concurrency: 3)
urls = (1..10).map { |i| "https://httpbin.org/delay/#{rand(1..2)}" }

puts "เริ่ม fetch #{urls.size} URLs..."
start = Time.monotonic
results = fetcher.fetch_all(urls)
total_time = Time.monotonic - start

results.each do |r|
  if err = r.error
    puts "#{r.url}: ERROR - #{err}"
  else
    puts "#{r.url}: #{r.status} (#{r.body_size} bytes, #{r.elapsed_ms.round(0)}ms)"
  end
end

puts "\nเวลาทั้งหมด: #{total_time.total_seconds.round(2)}s"
```

### แบบฝึกหัดที่ 2: Pipeline ประมวลผลข้อมูล

```crystal
struct DataRecord
  property id : Int32
  property value : Float64
  property category : String
  
  def initialize(@id, @value, @category)
  end
end

# สร้าง pipeline
records = (1..100).map { |i| DataRecord.new(i, rand(0.0..100.0), ["A", "B", "C"].sample) }

# Stage 1: Generate
gen_ch = Channel(DataRecord).new(20)
spawn do
  records.each { |r| gen_ch.send(r) }
  gen_ch.close
end

# Stage 2: Filter (เฉพาะ category A ที่ value > 50)
filter_ch = Channel(DataRecord).new(20)
spawn do
  while r = gen_ch.receive?
    filter_ch.send(r) if r.category == "A" && r.value > 50.0
  end
  filter_ch.close
end

# Stage 3: Enrich (คำนวณ bonus)
enrich_ch = Channel(Hash(String, String)).new(20)
spawn do
  while r = filter_ch.receive?
    enrich_ch.send({
      "id" => r.id.to_s,
      "value" => r.value.round(2).to_s,
      "bonus" => (r.value * 0.1).round(2).to_s,
      "category" => r.category,
    })
  end
  enrich_ch.close
end

# Collect results
results = [] of Hash(String, String)
while r = enrich_ch.receive?
  results << r
end

puts "พบ #{results.size} records:"
results.first(5).each { |r| puts r.inspect }
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Pipeline Pattern**: ข้อมูลไหลผ่าน stages ต่างๆ อย่างต่อเนื่อง
2. **Fan-Out**: กระจายงานให้หลาย workers พร้อมกัน
3. **Fan-In**: รวมผลลัพธ์จากหลาย channels
4. **Scatter-Gather**: ส่งงานให้ทุก workers แล้วรวมผล
5. **Rate Limiting**:
   - Token Bucket: burst แล้ว limit
   - Sliding Window: นับ requests ใน window
   - Leaky Bucket: ประมวลผลในอัตราคงที่
6. **Semaphore**: จำกัดจำนวน concurrent operations
7. **Circuit Breaker**: ป้องกัน cascading failures
8. **Bulkhead**: แยก resource pools
9. **Throttle/Debounce**: จำกัดความถี่การเรียก
