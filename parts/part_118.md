# Part 118: Event Loops - Event Loop ของ Crystal และ Async IO

## บทนำ

Event Loop เป็นหัวใจสำคัญของ concurrent programming ใน Crystal โดย Crystal ใช้ event-driven model ที่อยู่บน libevent2 หรือ io_uring (ขึ้นอยู่กับ platform) ทำให้สามารถรองรับ I/O operations จำนวนมากโดยไม่ต้องสร้าง thread จริงๆ

## Event Loop ทำงานอย่างไร

```
┌─────────────────────────────────────────┐
│              Event Loop                  │
│                                          │
│  ┌──────────┐    ┌──────────────────┐   │
│  │  Ready   │    │  Waiting (I/O)   │   │
│  │  Queue   │    │  - socket read   │   │
│  │          │    │  - socket write  │   │
│  │ Fiber A  │    │  - timer         │   │
│  │ Fiber B  │    │  - channel recv  │   │
│  └────┬─────┘    └────────┬─────────┘   │
│       │                   │             │
│       ▼                   │             │
│  ┌──────────┐             │             │
│  │ Scheduler│◄────────────┘             │
│  │          │  (I/O ready)              │
│  └──────────┘                           │
└─────────────────────────────────────────┘
```

## spawn และ Event Loop

```crystal
# spawn สร้าง fiber ใหม่ใน event loop
puts "เริ่ม main fiber"

spawn do
  puts "Fiber 1: เริ่ม"
  sleep 1.second  # ไม่ block event loop!
  puts "Fiber 1: ตื่น"
end

spawn do
  puts "Fiber 2: เริ่ม"
  sleep 0.5.seconds
  puts "Fiber 2: ตื่น"
end

puts "Main fiber: รอ..."
sleep 2.seconds
puts "Main fiber: เสร็จ"
```

ผลลัพธ์:
```
เริ่ม main fiber
Main fiber: รอ...
Fiber 2: เริ่ม
Fiber 1: เริ่ม
Fiber 2: ตื่น
Fiber 1: ตื่น
Main fiber: เสร็จ
```

## Fiber.yield

```crystal
# Fiber.yield ส่งคืนการควบคุมให้ scheduler
def busy_work(name : String, steps : Int32)
  steps.times do |i|
    puts "#{name}: step #{i + 1}"
    Fiber.yield # ให้ fiber อื่นทำงานบ้าง
  end
end

spawn { busy_work("A", 3) }
spawn { busy_work("B", 3) }

sleep 0.1.seconds
```

## Non-Blocking I/O Pattern

```crystal
require "socket"

# Non-blocking TCP server
server = TCPServer.new("localhost", 8080)
puts "รอการเชื่อมต่อ..."

# แต่ละ connection ได้รับ fiber ของตัวเอง
while client = server.accept?
  spawn handle_client(client)
end

def handle_client(socket : TCPSocket)
  puts "Client connected: #{socket.remote_address}"
  
  while line = socket.gets
    # I/O operations ไม่ block event loop
    socket.puts "Echo: #{line}"
  end
  
  socket.close
  puts "Client disconnected"
end
```

## Async File I/O

```crystal
require "file"

# การอ่านไฟล์แบบ async
def read_file_async(path : String, channel : Channel(String))
  spawn do
    begin
      content = File.read(path)
      channel.send(content)
    rescue ex
      channel.send("Error: #{ex.message}")
    end
  end
end

# อ่านหลายไฟล์พร้อมกัน
results = Channel(String).new(5)

["file1.txt", "file2.txt", "file3.txt"].each do |file|
  read_file_async(file, results)
end

# อ่านผลลัพธ์ แต่ในที่นี้เป็นแค่ตัวอย่าง
```

## Timer และ Scheduled Tasks

```crystal
# ตั้งเวลาทำงาน
def schedule_after(seconds : Float64, &block)
  spawn do
    sleep seconds.seconds
    block.call
  end
end

def schedule_every(seconds : Float64, &block) : Channel(Nil)
  stop = Channel(Nil).new
  
  spawn do
    loop do
      select
      when stop.receive?
        puts "หยุด periodic task"
        break
      when timeout(seconds.seconds)
        block.call
      end
    end
  end
  
  stop
end

# ตัวอย่างการใช้งาน
schedule_after(1.0) { puts "ทำงานหลังจาก 1 วินาที" }
schedule_after(2.0) { puts "ทำงานหลังจาก 2 วินาที" }

stop_timer = schedule_every(0.5) do
  puts "ทำงานทุก 0.5 วินาที: #{Time.local}"
end

sleep 3.seconds
stop_timer.send(nil) # หยุด periodic task
sleep 0.1.seconds
```

## Select Statement และ Event Loop

```crystal
# select รอหลาย event พร้อมกัน
channel1 = Channel(String).new
channel2 = Channel(Int32).new
done = Channel(Nil).new

spawn do
  sleep 1.second
  channel1.send("สวัสดีจาก channel1")
end

spawn do
  sleep 0.5.seconds
  channel2.send(42)
end

spawn do
  sleep 2.seconds
  done.send(nil)
end

puts "รอ events..."

loop do
  select
  when msg = channel1.receive?
    puts "Channel1: #{msg}"
  when num = channel2.receive?
    puts "Channel2: #{num}"
  when done.receive?
    puts "เสร็จสิ้น"
    break
  when timeout(3.seconds)
    puts "Timeout!"
    break
  end
end
```

## Non-Blocking Patterns

```crystal
# Non-blocking channel operations
channel = Channel(Int32).new(5)

# Non-blocking send
select
when channel.send(42)
  puts "ส่งสำเร็จ"
else
  puts "Channel เต็ม!"
end

# Non-blocking receive
select
when value = channel.receive?
  puts "ได้รับ: #{value}"
else
  puts "Channel ว่าง!"
end
```

### Async HTTP Request Pattern

```crystal
require "http/client"

# Fetch หลาย URL พร้อมกัน
def fetch_url(url : String, results : Channel(Tuple(String, String)))
  spawn do
    begin
      response = HTTP::Client.get(url)
      results.send({url, "Status: #{response.status_code}"})
    rescue ex
      results.send({url, "Error: #{ex.message}"})
    end
  end
end

urls = [
  "https://httpbin.org/get",
  "https://httpbin.org/ip",
  "https://httpbin.org/user-agent",
]

results = Channel(Tuple(String, String)).new(urls.size)

urls.each { |url| fetch_url(url, results) }

urls.size.times do
  url, result = results.receive
  puts "#{url}: #{result}"
end
```

## Reactor Pattern

```crystal
# Reactor Pattern: ลงทะเบียน event handlers
class Reactor
  alias Handler = Proc(String, Nil)
  
  def initialize
    @handlers = {} of String => Array(Handler)
    @event_channel = Channel(Tuple(String, String)).new(100)
    @running = false
  end
  
  def on(event : String, &handler : String ->)
    @handlers[event] ||= [] of Handler
    @handlers[event] << handler
  end
  
  def emit(event : String, data : String = "")
    @event_channel.send({event, data})
  end
  
  def start
    @running = true
    spawn do
      while @running
        select
        when event_data = @event_channel.receive?
          event, data = event_data
          if handlers = @handlers[event]?
            handlers.each { |h| h.call(data) }
          end
        when timeout(100.milliseconds)
          # ตรวจสอบ running flag
        end
      end
    end
  end
  
  def stop
    @running = false
  end
end

reactor = Reactor.new

reactor.on("click") { |data| puts "คลิก: #{data}" }
reactor.on("click") { |data| puts "จัดการคลิก: #{data}" }
reactor.on("hover") { |data| puts "โฮเวอร์: #{data}" }

reactor.start

sleep 0.1.seconds
reactor.emit("click", "button_submit")
reactor.emit("hover", "menu_item")
reactor.emit("click", "button_cancel")

sleep 0.2.seconds
reactor.stop
```

## Event Loop Monitoring

```crystal
# ติดตาม event loop metrics
class EventLoopMonitor
  property iterations : Int32 = 0
  property fibers_spawned : Int32 = 0
  property avg_latency_ms : Float64 = 0.0
  
  def start
    spawn monitor_loop
  end
  
  private def monitor_loop
    loop do
      start = Time.monotonic
      
      # Yield และวัดเวลา
      Fiber.yield
      
      latency = (Time.monotonic - start).total_milliseconds
      @iterations += 1
      
      # Running average
      @avg_latency_ms = (@avg_latency_ms * (@iterations - 1) + latency) / @iterations
      
      sleep 10.milliseconds
    end
  end
  
  def report
    puts "Event loop stats:"
    puts "  iterations: #{iterations}"
    puts "  avg latency: #{avg_latency_ms.round(3)} ms"
  end
end

monitor = EventLoopMonitor.new
monitor.start

# จำลองการทำงาน
50.times do |i|
  spawn { sleep rand(0.001..0.01).seconds }
end

sleep 1.second
monitor.report
```

## Cooperative Multitasking

```crystal
# Crystal ใช้ cooperative multitasking
# fiber ต้องยอมส่งคืน control เองผ่าน blocking operations

# ตัวอย่างที่ดี: fiber ยอมส่งคืน control
def good_fiber(name : String)
  10.times do |i|
    puts "#{name}: #{i}"
    sleep 0.01.seconds  # ส่งคืน control ผ่าน sleep
  end
end

# ตัวอย่างที่ไม่ดี: fiber ไม่ยอมส่งคืน control (CPU-intensive)
def bad_fiber(name : String)
  # loop นี้จะ block event loop ทั้งหมด!
  # sum = 0
  # 1_000_000.times { |i| sum += i }
  
  # แก้ไข: เพิ่ม Fiber.yield ระหว่างทาง
  sum = 0
  1_000_000.times do |i|
    sum += i
    Fiber.yield if i % 10_000 == 0  # yield ทุก 10k iterations
  end
  puts "Sum: #{sum}"
end

spawn { good_fiber("A") }
spawn { good_fiber("B") }
spawn { bad_fiber("C") }

sleep 1.second
```

## IO::Evented

```crystal
# Crystal's I/O classes ใช้ evented I/O อัตโนมัติ

require "socket"

# TCPSocket.new จะไม่ block event loop
# เมื่อรอข้อมูล socket จะ yield ให้ fiber อื่นทำงาน

def make_connection(host : String, port : Int32, id : Int32, results : Channel(String))
  spawn do
    begin
      socket = TCPSocket.new(host, port)
      socket.puts "Hello from connection #{id}"
      response = socket.gets
      socket.close
      results.send("Connection #{id}: #{response}")
    rescue ex
      results.send("Connection #{id} failed: #{ex.message}")
    end
  end
end

# ตัวอย่าง: เชื่อมต่อหลายอันพร้อมกัน (ถ้ามี server)
# results = Channel(String).new(10)
# 10.times { |i| make_connection("localhost", 8080, i, results) }
# 10.times { puts results.receive }
```

## Backpressure

```crystal
# จัดการ backpressure เพื่อป้องกัน memory overflow
class BackpressureProcessor(T)
  def initialize(@concurrency : Int32)
    @semaphore = Channel(Nil).new(@concurrency)
    @concurrency.times { @semaphore.send(nil) }
    @results = Channel(T).new(@concurrency * 2)
  end
  
  def process(item : T, &block : T -> T)
    @semaphore.receive # acquire slot
    
    spawn do
      begin
        result = block.call(item)
        @results.send(result)
      ensure
        @semaphore.send(nil) # release slot
      end
    end
  end
  
  def results
    @results
  end
end

processor = BackpressureProcessor(Int32).new(3)

# ส่ง items มากกว่า concurrency
20.times do |i|
  processor.process(i) do |n|
    sleep 0.1.seconds
    n * 2
  end
end

20.times do
  puts processor.results.receive
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Event Loop Visualizer

```crystal
# สร้าง visualizer สำหรับดู event loop
class FiberScheduler
  struct FiberInfo
    property id : Int32
    property name : String
    property status : String
    property created_at : Time
    
    def initialize(@id, @name, @status = "running", @created_at = Time.local)
    end
  end
  
  @@fibers = {} of Int32 => FiberInfo
  @@mutex = Mutex.new
  @@counter = Atomic(Int32).new(0)
  
  def self.register(name : String) : Int32
    id = @@counter.add(1)
    @@mutex.synchronize { @@fibers[id] = FiberInfo.new(id, name) }
    id
  end
  
  def self.update_status(id : Int32, status : String)
    @@mutex.synchronize do
      if info = @@fibers[id]?
        @@fibers[id] = FiberInfo.new(id, info.name, status, info.created_at)
      end
    end
  end
  
  def self.report
    @@mutex.synchronize do
      puts "\n=== Fiber Status ==="
      @@fibers.each_value do |info|
        puts "  [#{info.id}] #{info.name}: #{info.status}"
      end
    end
  end
end

# ใช้งาน
3.times do |i|
  id = FiberScheduler.register("Worker #{i}")
  
  spawn do
    FiberScheduler.update_status(id, "working")
    sleep rand(0.1..0.5).seconds
    FiberScheduler.update_status(id, "done")
  end
end

sleep 0.6.seconds
FiberScheduler.report
```

### แบบฝึกหัดที่ 2: Async Pipeline

```crystal
# สร้าง async pipeline
def async_stage(input : Channel(T), output : Channel(R), &transform : T -> R) forall T, R
  spawn do
    while value = input.receive?
      result = transform.call(value)
      output.send(result)
    end
    output.close
  end
end

# ตัวอย่าง: pipeline การประมวลผลข้อมูล
numbers = Channel(Int32).new(10)
doubled = Channel(Int32).new(10)
filtered = Channel(Int32).new(10)
formatted = Channel(String).new(10)

# Stage 1: คูณด้วย 2
async_stage(numbers, doubled) { |n| n * 2 }

# Stage 2: กรองเลขคี่
spawn do
  while n = doubled.receive?
    filtered.send(n) if n.odd?
  end
  filtered.close
end

# Stage 3: แปลงเป็น string
async_stage(filtered, formatted) { |n| "จำนวน: #{n}" }

# ส่งข้อมูล
20.times { |i| numbers.send(i + 1) }
numbers.close

# รับผลลัพธ์
while result = formatted.receive?
  puts result
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Event Loop**: Crystal ใช้ libevent/io_uring ทำให้ I/O operations ไม่ block
2. **spawn**: สร้าง fiber ใหม่ใน event loop
3. **Fiber.yield**: ส่งคืน control ให้ scheduler
4. **Non-blocking I/O**: socket, file, timer ทั้งหมดเป็น non-blocking
5. **select**: รอหลาย events พร้อมกัน
6. **Cooperative multitasking**: fiber ต้อง yield เองผ่าน blocking ops
7. **Backpressure**: ป้องกัน memory overflow ด้วย semaphore
8. **Reactor Pattern**: event-driven architecture

จุดสำคัญ:
- Crystal ไม่ใช้ thread จริงๆ ในการทำ concurrency (ยกเว้น multi-threading)
- I/O operations ทั้งหมดจะ yield ให้ event loop อัตโนมัติ
- CPU-intensive tasks ต้องเพิ่ม Fiber.yield เองเพื่อไม่ block event loop
