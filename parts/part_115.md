# Part 115: Async/Await Pattern ใน Crystal

## บทนำ

Crystal ไม่มี `async/await` แบบ JavaScript หรือ Python แต่ใช้ `spawn` + Channels + Fibers เพื่อให้ได้ผลลัพธ์เดียวกัน ใน part นี้เราจะเรียนรู้ patterns สำหรับ concurrent programming ใน Crystal

## Crystal's spawn (ไม่ใช่ async/await)

```crystal
# spawn สร้าง Fiber ใหม่ที่ run ใน event loop
# เทียบกับ JavaScript: await fetch(url) ≈ ch.receive หลัง spawn

# JavaScript:
# async function fetchData() {
#   const result = await fetch(url);
#   return result.json();
# }

# Crystal equivalent:
def fetch_data(url : String) : String
  ch = Channel(String).new

  spawn do
    # Simulate HTTP request (non-blocking I/O)
    sleep 0.1.seconds
    ch.send("Response from #{url}")
  end

  ch.receive  # "await" equivalent
end

result = fetch_data("http://api.example.com/data")
puts result  # => Response from http://api.example.com/data
```

## Future-like Pattern

```crystal
# Future(T) - ค่าที่จะมาในอนาคต
class Future(T)
  @channel : Channel(T | Exception)
  @value : T?
  @error : Exception?
  @resolved = false

  def initialize
    @channel = Channel(T | Exception).new(1)
  end

  # Set the future's value (called from worker fiber)
  def resolve(value : T)
    @channel.send(value)
  end

  # Set the future's error
  def reject(error : Exception)
    @channel.send(error)
  end

  # Wait for and get the value (blocking)
  def await : T
    return @value.not_nil! if @resolved

    result = @channel.receive
    @resolved = true

    case result
    when Exception
      @error = result
      raise result
    else
      @value = result.as(T)
      result.as(T)
    end
  end

  # Non-blocking check
  def resolved? : Bool
    @resolved
  end

  # Map transform (like Promise.then)
  def map(&block : T -> U) : Future(U) forall U
    new_future = Future(U).new
    spawn do
      begin
        value = await
        new_future.resolve(block.call(value))
      rescue ex
        new_future.reject(ex)
      end
    end
    new_future
  end
end

# Helper สร้าง Future จาก block
def async(&block : -> T) : Future(T) forall T
  future = Future(T).new
  spawn do
    begin
      result = block.call
      future.resolve(result)
    rescue ex
      future.reject(ex)
    end
  end
  future
end

# ใช้งาน Future
future = async { sleep(0.05.seconds); 42 }
puts "Working..."
value = future.await
puts "Got: #{value}"  # => Got: 42

# Chaining futures
result = async { sleep(0.01.seconds); 10 }
  .map { |n| n * 2 }
  .map { |n| "Result: #{n}" }

puts result.await  # => Result: 20
```

## Promise-like Pattern

```crystal
# Promise chain pattern
class Promise(T)
  alias Resolver(U) = Proc(U, Nil)
  alias Rejector = Proc(Exception, Nil)

  @channel : Channel(Result(T))
  @fulfilled = false

  private alias Result(U) = {value: U?, error: Exception?}

  def initialize
    @channel = Channel(Result(T)).new(1)
  end

  def self.resolve(value : T) : Promise(T)
    p = Promise(T).new
    p.fulfill(value)
    p
  end

  def self.reject(error : Exception) : Promise(T)
    p = Promise(T).new
    p.fail(error)
    p
  end

  def fulfill(value : T)
    @channel.send({value: value, error: nil})
  end

  def fail(error : Exception)
    @channel.send({value: nil, error: error})
  end

  def then(&block : T -> U) : Promise(U) forall U
    next_promise = Promise(U).new
    spawn do
      result = @channel.receive
      if err = result[:error]
        next_promise.fail(err)
      elsif val = result[:value]
        begin
          next_promise.fulfill(block.call(val))
        rescue ex
          next_promise.fail(ex)
        end
      end
    end
    next_promise
  end

  def catch(&block : Exception -> T) : Promise(T)
    next_promise = Promise(T).new
    spawn do
      result = @channel.receive
      if err = result[:error]
        begin
          next_promise.fulfill(block.call(err))
        rescue ex
          next_promise.fail(ex)
        end
      elsif val = result[:value]
        next_promise.fulfill(val)
      end
    end
    next_promise
  end

  def await : T
    result = @channel.receive
    if err = result[:error]
      raise err
    elsif val = result[:value]
      val
    else
      raise "Promise resolved with no value"
    end
  end
end

# ใช้งาน
promise = Promise(Int32).new

spawn do
  sleep 0.01.seconds
  promise.fulfill(100)
end

final = promise
  .then { |n| n * 2 }
  .then { |n| n.to_s }
  .catch { |e| "Error: #{e.message}" }

puts final.await  # => 200
```

## EventLoop-based Async

```crystal
# Simulate event-driven async pattern
class EventLoop
  alias Callback = Proc(Nil)

  def initialize
    @tasks = [] of {at: Time, callback: Callback}
    @running = false
  end

  def set_timeout(ms : Int32, &callback : -> Nil)
    run_at = Time.utc + ms.milliseconds
    @tasks << {at: run_at, callback: callback}
    @tasks.sort_by! { |t| t[:at] }
  end

  def run
    @running = true
    while @running && !@tasks.empty?
      now = Time.utc
      ready = @tasks.select { |t| t[:at] <= now }
      @tasks.reject! { |t| t[:at] <= now }

      ready.each { |t| t[:callback].call }

      sleep 0.001.seconds unless @tasks.empty?
    end
    @running = false
  end

  def stop
    @running = false
  end
end

loop = EventLoop.new

loop.set_timeout(50) { puts "50ms later" }
loop.set_timeout(10) { puts "10ms later" }
loop.set_timeout(100) { puts "100ms later" }
loop.set_timeout(150) { loop.stop }

loop.run

# Output (in order):
# 10ms later
# 50ms later
# 100ms later
```

## Concurrent HTTP Requests Pattern

```crystal
# Concurrent requests - เทียบกับ Promise.all
def fetch_all(urls : Array(String)) : Array(String)
  results = Array(String | Nil).new(urls.size, nil)
  done_channel = Channel(Int32).new(urls.size)

  urls.each_with_index do |url, i|
    spawn do
      # Simulate HTTP request
      sleep (rand(100) + 10).milliseconds
      results[i] = "Response from #{url}"
      done_channel.send(i)
    end
  end

  # Wait for all
  urls.size.times { done_channel.receive }

  results.compact
end

urls = [
  "http://api1.example.com",
  "http://api2.example.com",
  "http://api3.example.com",
]

start = Time.utc
responses = fetch_all(urls)
elapsed = Time.utc - start

responses.each { |r| puts r }
puts "All done in #{elapsed.total_milliseconds.round}ms (concurrent!)"
```

## Promise.all Equivalent

```crystal
# รอทุก futures พร้อมกัน
def await_all(futures : Array(Future(T))) : Array(T) forall T
  results = Array(T?).new(futures.size, nil)
  mutex = Mutex.new
  done = Channel(Int32).new(futures.size)

  futures.each_with_index do |future, i|
    spawn do
      value = future.await
      mutex.synchronize { results[i] = value }
      done.send(i)
    end
  end

  futures.size.times { done.receive }
  results.compact
end

# ใช้งาน
futures = (1..5).map do |i|
  async { sleep(rand(50).milliseconds); i * i }
end

results = await_all(futures)
puts results.sort.inspect  # => [1, 4, 9, 16, 25]
```

## Async with Timeout

```crystal
# Async operation ที่มี timeout
def with_timeout(ms : Int32, &block : -> T) : T? forall T
  result_ch = Channel(T).new(1)
  timeout_ch = Channel(Nil).new(1)

  spawn do
    result_ch.send(block.call)
  end

  spawn do
    sleep ms.milliseconds
    timeout_ch.send(nil)
  end

  select
  when result = result_ch.receive
    result
  when timeout_ch.receive
    nil
  end
end

# ใช้งาน
result = with_timeout(100) do
  sleep 50.milliseconds
  "fast operation"
end
puts result.inspect  # => "fast operation"

slow_result = with_timeout(50) do
  sleep 200.milliseconds
  "slow operation"
end
puts slow_result.inspect  # => nil (timeout)
```

## Async Queue (Task Queue)

```crystal
# Async task queue - คล้ายกับ JavaScript's async job queue
class AsyncQueue(T)
  def initialize(@concurrency : Int32 = 1)
    @pending = Channel(Proc(T)).new(100)
    @results = Channel(T).new(100)
    @active = Atomic(Int32).new(0)
    start_workers
  end

  def enqueue(&task : -> T) : Future(T)
    future = Future(T).new
    spawn do
      @pending.send(task)
      # Wait for this specific task's result
      # (simplified - real impl needs task tracking)
    end
    future
  end

  def push(&task : -> T)
    @pending.send(task)
  end

  def results : Channel(T)
    @results
  end

  def close
    @pending.close
  end

  private def start_workers
    @concurrency.times do
      spawn do
        while task = @pending.receive?
          @active.add(1)
          result = task.call
          @results.send(result)
          @active.add(-1)
        end
      end
    end
  end
end

# ใช้งาน
queue = AsyncQueue(String).new(concurrency: 3)

10.times do |i|
  queue.push do
    sleep (rand(20) + 5).milliseconds
    "Task #{i} completed"
  end
end
queue.close

10.times do
  if result = queue.results.receive?
    puts result
  end
end
```

## Retry Pattern

```crystal
# Async retry with exponential backoff
def with_retry(
  max_attempts : Int32 = 3,
  base_delay_ms : Int32 = 100,
  &block : -> T
) : T forall T
  attempts = 0
  last_error : Exception? = nil

  loop do
    attempts += 1
    begin
      return block.call
    rescue ex
      last_error = ex
      break if attempts >= max_attempts

      delay = base_delay_ms * (2 ** (attempts - 1))
      puts "Attempt #{attempts} failed: #{ex.message}. Retrying in #{delay}ms..."
      sleep delay.milliseconds
    end
  end

  raise last_error.not_nil!
end

# ใช้งาน
attempt_count = 0

begin
  result = with_retry(max_attempts: 3, base_delay_ms: 10) do
    attempt_count += 1
    raise "Temporary error" if attempt_count < 3
    "Success on attempt #{attempt_count}"
  end
  puts result  # => Success on attempt 3
rescue ex
  puts "All retries failed: #{ex.message}"
end
```

## Concurrent Data Processing

```crystal
# Parallel map ด้วย channels
def parallel_map(items : Array(T), workers : Int32 = 4, &block : T -> U) : Array(U) forall T, U
  input_ch = Channel(T).new(items.size)
  output_ch = Channel({index: Int32, value: U}).new(items.size)

  # Fill input
  items.each { |item| input_ch.send(item) }
  input_ch.close

  # Start workers
  workers.times do
    spawn do
      while item = input_ch.receive?
        idx = items.index(item) || 0
        result = block.call(item)
        output_ch.send({index: idx, value: result})
      end
    end
  end

  # Collect results
  results = Array({index: Int32, value: U}).new

  items.size.times do
    results << output_ch.receive
  end

  results.sort_by { |r| r[:index] }.map { |r| r[:value] }
end

# ใช้งาน
numbers = (1..10).to_a

start = Time.utc
squares = parallel_map(numbers, workers: 4) do |n|
  sleep 10.milliseconds  # Simulate work
  n * n
end
elapsed = Time.utc - start

puts squares.inspect
# => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
puts "Processed in #{elapsed.total_milliseconds.round}ms"
```

## Fiber-based State Machine

```crystal
# Async state machine ด้วย Fibers
enum ConnectionState
  Disconnected
  Connecting
  Connected
  Disconnecting
  Error
end

class AsyncConnection
  getter state : ConnectionState
  getter error : String?

  def initialize(@host : String, @port : Int32)
    @state = ConnectionState::Disconnected
    @events = Channel(Symbol).new(10)
    @state_changes = Channel(ConnectionState).new(10)
  end

  def connect : Future(Bool)
    future = Future(Bool).new

    spawn do
      @state = ConnectionState::Connecting
      @state_changes.send(@state)

      begin
        # Simulate connection
        sleep 50.milliseconds
        @state = ConnectionState::Connected
        @state_changes.send(@state)
        future.resolve(true)
      rescue ex
        @state = ConnectionState::Error
        @error = ex.message
        @state_changes.send(@state)
        future.resolve(false)
      end
    end

    future
  end

  def disconnect : Future(Nil)
    future = Future(Nil).new

    spawn do
      @state = ConnectionState::Disconnecting
      @state_changes.send(@state)
      sleep 10.milliseconds
      @state = ConnectionState::Disconnected
      @state_changes.send(@state)
      future.resolve(nil)
    end

    future
  end

  def on_state_change : Channel(ConnectionState)
    @state_changes
  end
end

# ใช้งาน
conn = AsyncConnection.new("localhost", 8080)

# Monitor state changes
spawn do
  while state = conn.on_state_change.receive?
    puts "State: #{state}"
  end
end

connected = conn.connect.await
puts "Connected: #{connected}"
conn.disconnect.await
puts "Done"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Promise.race Equivalent
สร้าง function ที่:
- รับ array ของ Futures
- Return ผลของ Future แรกที่สำเร็จ
- Cancel futures ที่เหลือ

### แบบฝึกหัดที่ 2: Async Semaphore
สร้าง semaphore ที่จำกัดจำนวน concurrent operations:
- `acquire` - รอถ้า semaphore เต็ม
- `release` - คืน slot
- `with_semaphore` - convenience wrapper

### แบบฝึกหัดที่ 3: Async Cache
สร้าง cache ที่:
- คำนวณ value แบบ async
- Prevent duplicate computation (request deduplication)
- TTL expiration

### แบบฝึกหัดที่ 4: Pipeline Async
สร้าง async pipeline:
- Multiple async stages
- Error propagation
- Cancellation support
- Backpressure

## สรุป

Async patterns ใน Crystal:
- **spawn**: สร้าง lightweight fiber
- **Channel**: สื่อสารและ synchronize
- **Future(T)**: represent ค่าในอนาคต
- **select**: handle หลาย channels

เทียบกับ async/await ภาษาอื่น:
| JavaScript | Crystal |
|---|---|
| `async function f()` | `def f; ch = Channel.new; spawn { ... }` |
| `await promise` | `channel.receive` |
| `Promise.all` | `await_all(futures)` |
| `Promise.race` | `select { when ch1.receive ... when ch2.receive ... }` |
| `setTimeout` | `sleep n.milliseconds` |

Crystal's concurrency advantages:
1. **No callback hell**: Channels ทำให้ code flow เป็น sequential
2. **Type safety**: `Channel(T)` และ `Future(T)` type-safe เต็มที่
3. **Lightweight**: Fibers ใช้ memory น้อยกว่า threads มาก
4. **Composable**: Patterns build กันได้ง่าย
5. **Zero overhead**: ไม่มี runtime cost ของ async state machine
