# Part 113: Select Statement

## บทนำ

`select` ใน Crystal ช่วยให้ handle หลาย channel operations พร้อมกัน เลือก operation แรกที่พร้อม คล้ายกับ `select` ใน Go

## select พื้นฐาน

```crystal
# select เลือก channel ที่พร้อมก่อน
ch1 = Channel(String).new
ch2 = Channel(String).new

spawn { ch1.send "from ch1" }
spawn { ch2.send "from ch2" }

Fiber.yield  # ให้ spawned fibers มีเวลา queue

select
when msg = ch1.receive
  puts "Received from ch1: #{msg}"
when msg = ch2.receive
  puts "Received from ch2: #{msg}"
end
```

## select กับ Multiple Channels

```crystal
# select กับหลาย channels
def process_messages(errors : Channel(String), results : Channel(Int32))
  loop do
    select
    when err = errors.receive
      puts "Error: #{err}"
    when result = results.receive
      puts "Result: #{result}"
    end
  end
end

errors = Channel(String).new(5)
results = Channel(Int32).new(5)

# Simulate concurrent operations
spawn do
  results.send(42)
  errors.send("connection failed")
  results.send(100)
  results.send(200)
  errors.send("timeout")
end

Fiber.yield

5.times do
  select
  when err = errors.receive?
    if err
      puts "Error: #{err}"
    else
      break
    end
  when result = results.receive?
    if result
      puts "Result: #{result}"
    else
      break
    end
  end
end
```

## Timeout select

```crystal
# Timeout ด้วย select
def fetch_with_timeout(url : String, timeout_ms : Int32) : String?
  result_ch = Channel(String).new
  timeout_ch = Channel(Nil).new

  # Simulated fetch
  spawn do
    sleep(rand(timeout_ms * 2).milliseconds)
    result_ch.send("Response from #{url}")
  end

  # Timeout
  spawn do
    sleep(timeout_ms.milliseconds)
    timeout_ch.send(nil)
  end

  select
  when response = result_ch.receive
    response
  when timeout_ch.receive
    nil
  end
end

# ทดสอบ
result = fetch_with_timeout("http://example.com", 100)
if result
  puts "Got: #{result}"
else
  puts "Timeout!"
end
```

## Default Case ใน select

```crystal
# default case - ไม่ block ถ้าไม่มี channel พร้อม
ch = Channel(Int32).new(5)

# Non-blocking send
spawn do
  ch.send(1)
  ch.send(2)
  ch.send(3)
end

Fiber.yield

# Try to receive, don't block
10.times do
  select
  when value = ch.receive?
    puts "Got: #{value}"
  else
    puts "No data available"
    break
  end
end
```

## Select กับ Send และ Receive

```crystal
# select รองรับทั้ง send และ receive
data_ch = Channel(Int32).new
ack_ch = Channel(Bool).new

# Producer
spawn do
  data_ch.send(42)
  success = ack_ch.receive
  puts "Acknowledged: #{success}"
end

# Consumer/Processor
spawn do
  select
  when value = data_ch.receive
    puts "Processing: #{value}"
    ack_ch.send(true)
  end
end

Fiber.yield
sleep 0.01
```

## Multiple Channel Coordination

```crystal
# Coordinate หลาย operations
struct Request
  getter id : Int32
  getter data : String

  def initialize(@id, @data)
  end
end

struct Response
  getter request_id : Int32
  getter result : String

  def initialize(@request_id, @result)
  end
end

class RequestProcessor
  def initialize
    @requests = Channel(Request).new(10)
    @responses = Channel(Response).new(10)
    @shutdown = Channel(Nil).new

    start_workers(3)
  end

  def submit(req : Request) : Bool
    select
    when @requests.send(req)
      true
    else
      false
    end
  end

  def next_response : Response?
    select
    when resp = @responses.receive?
      resp
    when @shutdown.receive?
      nil
    end
  end

  def stop
    @requests.close
    @shutdown.send(nil)
  end

  private def start_workers(count : Int32)
    count.times do |i|
      spawn do
        while req = @requests.receive?
          # Process request
          result = Response.new(
            req.id,
            "Worker #{i}: #{req.data.upcase}"
          )
          @responses.send(result)
        end
      end
    end
  end
end

processor = RequestProcessor.new

# Submit requests
5.times do |i|
  processor.submit(Request.new(i, "data_#{i}"))
end

# Collect responses
5.times do
  if resp = processor.next_response
    puts "Response #{resp.request_id}: #{resp.result}"
  end
end

processor.stop
```

## Select Loop Pattern

```crystal
# Continuous select loop
def message_router(
  high_priority : Channel(String),
  normal : Channel(String),
  low_priority : Channel(String),
  done : Channel(Nil)
)
  loop do
    select
    when msg = high_priority.receive?
      puts "[HIGH] #{msg}"
    when msg = normal.receive?
      puts "[NORM] #{msg}"
    when msg = low_priority.receive?
      puts "[LOW] #{msg}"
    when done.receive?
      puts "Router shutting down"
      return
    end
  end
end

high = Channel(String).new(5)
norm = Channel(String).new(5)
low = Channel(String).new(5)
done = Channel(Nil).new

spawn { message_router(high, norm, low, done) }

# Send messages
spawn do
  high.send "Critical alert!"
  norm.send "User logged in"
  low.send "Debug info"
  norm.send "Data processed"
  high.send "Server error!"
  done.send nil
end

sleep 0.05
```

## Merge Channels กับ select

```crystal
# Merge หลาย channels เป็น single output
def merge(channels : Array(Channel(Int32))) : Channel(Int32)
  out = Channel(Int32).new(channels.size * 5)
  remaining = Channel(Int32).new  # counter

  channels.each do |ch|
    spawn do
      while value = ch.receive?
        out.send(value)
      end
      remaining.send(1)
    end
  end

  # Close output เมื่อทุก channels ปิด
  spawn do
    channels.size.times { remaining.receive }
    out.close
  end

  out
end

# สร้าง source channels
sources = Array.new(3) do |i|
  ch = Channel(Int32).new(5)
  spawn do
    5.times { |j| ch.send(i * 10 + j) }
    ch.close
  end
  ch
end

merged = merge(sources)

all_values = [] of Int32
while v = merged.receive?
  all_values << v
end

puts all_values.sort.inspect
```

## select กับ Struct Messages

```crystal
# Type-safe messages ด้วย struct
abstract struct Message; end

struct PingMessage < Message
  getter id : Int32
  def initialize(@id); end
end

struct PongMessage < Message
  getter id : Int32
  def initialize(@id); end
end

struct QuitMessage < Message; end

# Typed channels สำหรับแต่ละ message type
ping_ch = Channel(PingMessage).new
pong_ch = Channel(PongMessage).new
quit_ch = Channel(QuitMessage).new

# Ping-pong game
spawn do
  5.times do |i|
    ping_ch.send(PingMessage.new(i))
    pong = pong_ch.receive
    puts "Got pong for ping #{pong.id}"
  end
  quit_ch.send(QuitMessage.new)
end

loop do
  select
  when ping = ping_ch.receive
    puts "Got ping #{ping.id}, sending pong"
    pong_ch.send(PongMessage.new(ping.id))
  when quit_ch.receive
    puts "Quit!"
    break
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Load Balancer
สร้าง load balancer ด้วย select:
- Round-robin distribution
- Select available workers
- Health check ด้วย heartbeat channel

### แบบฝึกหัดที่ 2: Circuit Breaker
Implement circuit breaker:
- Open/closed/half-open states
- Select ระหว่าง success/failure channels
- Timeout handling

### แบบฝึกหัดที่ 3: Pub/Sub System
สร้าง pub/sub:
- Subscribe ไปหลาย topics
- Select ด้วย topic channels
- Unsubscribe

### แบบฝึกหัดที่ 4: Backpressure
สร้าง backpressure mechanism:
- ใช้ select default case ตรวจ pressure
- Slow down producer เมื่อ consumer ช้า
- Metrics

## สรุป

Select ใน Crystal:
- **First-ready**: เลือก operation แรกที่พร้อม
- **Non-blocking**: ใช้ `else` clause สำหรับ default
- **Both send/receive**: รองรับทั้ง operations
- **Loop**: สร้าง message processing loops

Patterns ที่สำคัญ:
1. Timeout patterns ด้วย timeout channel
2. Priority queues ด้วย ordered select
3. Fan-in / Fan-out
4. Non-blocking polling ด้วย `else`
