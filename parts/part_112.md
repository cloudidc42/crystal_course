# Part 112: Channels

## บทนำ

Channel ใน Crystal เป็นวิธีหลักในการสื่อสารระหว่าง Fibers แบบ type-safe ตาม CSP (Communicating Sequential Processes) model ที่ Go ก็ใช้

## Channel พื้นฐาน

```crystal
# สร้าง Channel
channel = Channel(Int32).new

# Send ค่าไปยัง channel
spawn do
  channel.send(42)
  puts "Sent 42"
end

# Receive ค่าจาก channel
value = channel.receive
puts "Received: #{value}"  # => Received: 42

# String channel
str_ch = Channel(String).new

spawn do
  str_ch.send "สวัสดี Crystal"
end

msg = str_ch.receive
puts msg  # => สวัสดี Crystal
```

## Buffered Channels

```crystal
# Unbuffered channel (default) - block จนมี receiver
unbuffered = Channel(Int32).new

# Buffered channel - ไม่ block จนกว่า buffer จะเต็ม
buffered = Channel(Int32).new(10)

# Buffered channel - ส่งได้ทันที (ถ้า buffer ไม่เต็ย)
spawn do
  10.times do |i|
    buffered.send(i)
    puts "Sent #{i}"
  end
  buffered.close
end

# Receive ทีละค่า
while value = buffered.receive?
  puts "Got: #{value}"
end
```

## Channel Directions

```crystal
# กำหนด direction ของ channel เพื่อ safety
# Crystal ไม่มี syntax สำหรับ unidirectional channels
# แต่สามารถ enforce ด้วย pattern

# Producer - only sends
def producer(ch : Channel(Int32))
  10.times do |i|
    ch.send(i * i)
  end
  ch.close
end

# Consumer - only receives
def consumer(ch : Channel(Int32))
  while value = ch.receive?
    puts "Consuming: #{value}"
  end
end

# ใช้งาน
ch = Channel(Int32).new(5)

spawn { producer(ch) }
spawn { consumer(ch) }

Fiber.yield
sleep 0.01  # ให้เวลา fibers ทำงาน
```

## Channel Close

```crystal
# close channel - ส่งสัญญาณว่าไม่มีข้อมูลอีกแล้ว
ch = Channel(String).new(3)

spawn do
  ["ข้อมูล 1", "ข้อมูล 2", "ข้อมูล 3"].each do |item|
    ch.send(item)
  end
  ch.close
  puts "Channel closed"
end

# receive? return nil เมื่อ channel closed และ empty
while item = ch.receive?
  puts "Received: #{item}"
end
puts "Done receiving"

# Output:
# Received: ข้อมูล 1
# Received: ข้อมูล 2
# Received: ข้อมูล 3
# Channel closed
# Done receiving
```

## Multiple Producers

```crystal
# หลาย producers, หนึ่ง consumer
def start_producers(ch : Channel(String), count : Int32)
  count.times do |i|
    spawn do
      3.times do |j|
        ch.send("Producer #{i}, item #{j}")
      end
    end
  end
end

results = Channel(String).new(20)
start_producers(results, 3)

# Collect results
received = 0
total = 3 * 3  # 3 producers * 3 items

total.times do
  puts results.receive
end

puts "All done"
```

## Pipeline ด้วย Channels

```crystal
# Pipeline: generate → transform → collect
def generate(ch : Channel(Int32))
  spawn do
    (1..10).each { |n| ch.send(n) }
    ch.close
  end
end

def square(in_ch : Channel(Int32), out_ch : Channel(Int32))
  spawn do
    while n = in_ch.receive?
      out_ch.send(n * n)
    end
    out_ch.close
  end
end

def filter_even(in_ch : Channel(Int32), out_ch : Channel(Int32))
  spawn do
    while n = in_ch.receive?
      out_ch.send(n) if n.even?
    end
    out_ch.close
  end
end

# สร้าง pipeline
nums = Channel(Int32).new(10)
squared = Channel(Int32).new(10)
evens = Channel(Int32).new(10)

generate(nums)
square(nums, squared)
filter_even(squared, evens)

# Collect results
results = [] of Int32
while n = evens.receive?
  results << n
end

puts results.inspect  # => [4, 16, 36, 64, 100]
```

## Fan-out Pattern

```crystal
# Fan-out: หนึ่ง producer → หลาย consumers
def fan_out(source : Channel(Int32), consumers : Array(Channel(Int32)))
  spawn do
    idx = 0
    while item = source.receive?
      consumers[idx % consumers.size].send(item)
      idx += 1
    end
    consumers.each(&.close)
  end
end

source = Channel(Int32).new(10)
worker_channels = Array.new(3) { Channel(Int32).new(10) }

# Populate source
spawn do
  20.times { |i| source.send(i) }
  source.close
end

fan_out(source, worker_channels)

# Start workers
mutex = Mutex.new
results = Array(Array(Int32)).new(3) { [] of Int32 }

worker_channels.each_with_index do |ch, i|
  spawn do
    while item = ch.receive?
      mutex.synchronize { results[i] << item }
    end
  end
end

sleep 0.05

puts "Worker 0: #{results[0].sort.inspect}"
puts "Worker 1: #{results[1].sort.inspect}"
puts "Worker 2: #{results[2].sort.inspect}"
```

## Fan-in Pattern

```crystal
# Fan-in: หลาย producers → หนึ่ง consumer
def fan_in(sources : Array(Channel(Int32)), dest : Channel(Int32))
  done = Channel(Nil).new

  sources.each do |source|
    spawn do
      while item = source.receive?
        dest.send(item)
      end
      done.send(nil)
    end
  end

  spawn do
    sources.size.times { done.receive }
    dest.close
  end
end

result_channel = Channel(Int32).new(20)
source_channels = Array.new(3) do |i|
  ch = Channel(Int32).new(5)
  spawn do
    5.times { |j| ch.send(i * 10 + j) }
    ch.close
  end
  ch
end

fan_in(source_channels, result_channel)

all_values = [] of Int32
while v = result_channel.receive?
  all_values << v
end

puts all_values.sort.inspect
```

## Channel สำหรับ Work Distribution

```crystal
# Work stealing / distribution pattern
struct Job
  getter id : Int32
  getter data : String

  def initialize(@id, @data)
  end
end

class WorkPool
  def initialize(worker_count : Int32)
    @job_channel = Channel(Job).new(worker_count * 2)
    @result_channel = Channel(String).new(worker_count * 2)
    @workers = worker_count

    start_workers
  end

  def submit(job : Job)
    @job_channel.send(job)
  end

  def close
    @job_channel.close
  end

  def results : Channel(String)
    @result_channel
  end

  private def start_workers
    @workers.times do |i|
      spawn do
        while job = @job_channel.receive?
          # Process job
          result = "Worker #{i} processed job #{job.id}: #{job.data.upcase}"
          @result_channel.send(result)
        end
        @result_channel.close if i == @workers - 1
      end
    end
  end
end

pool = WorkPool.new(3)

# Submit jobs
spawn do
  10.times do |i|
    pool.submit(Job.new(i, "task_#{i}"))
  end
  pool.close
end

# Collect results
count = 0
while result = pool.results.receive?
  puts result
  count += 1
  break if count >= 10
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Timeout Channel
สร้าง channel timeout:
- Receive ด้วย timeout
- Return nil ถ้า timeout
- Use cases: HTTP requests, database queries

### แบบฝึกหัดที่ 2: Merge Channels
สร้าง function ที่ merge หลาย channels เป็นหนึ่ง:
- Generic `merge(*channels)`
- Ordered vs unordered merge
- Close เมื่อทุก sources closed

### แบบฝึกหัดที่ 3: Rate Limiter
สร้าง rate limiter ด้วย channels:
- Token bucket algorithm
- N requests per second
- Burst capacity

### แบบฝึกหัดที่ 4: MapReduce
Implement MapReduce ด้วย channels:
- Map phase: parallel processing
- Reduce phase: combine results
- Configurable workers

## สรุป

Channels ใน Crystal:
- **Type-safe**: `Channel(T)` specify ชนิดของ messages
- **Buffered**: กำหนด capacity
- **Close**: ส่งสัญญาณ end of stream
- **receive?**: safe receive ที่ return nil เมื่อ closed

Best practices:
1. ใช้ buffered channels เพื่อลด blocking
2. Sender ควร close channel
3. ใช้ `receive?` แทน `receive` เพื่อ handle closed
4. Pipeline pattern เหมาะสำหรับ data processing
