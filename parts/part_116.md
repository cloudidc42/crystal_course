# Part 116: Concurrent Collections - คอลเลกชันสำหรับ Concurrent Programming

## บทนำ

ใน Crystal การเขียนโปรแกรมแบบ concurrent นั้น เราต้องระวังเรื่องการเข้าถึงข้อมูลพร้อมกันจากหลาย fiber หรือ thread ซึ่งอาจทำให้เกิด race condition ได้ ในบทนี้เราจะเรียนรู้เกี่ยวกับ Channel ในฐานะ queue และการใช้ Mutex เพื่อป้องกันโครงสร้างข้อมูลให้ thread-safe

## Channel ในฐานะ Queue

Channel ใน Crystal เป็นโครงสร้างข้อมูลที่ออกแบบมาสำหรับการสื่อสารระหว่าง fiber อย่างปลอดภัย

### Channel พื้นฐาน

```crystal
# Channel แบบ unbuffered - blocking จนกว่าจะมีคนรับ
channel = Channel(Int32).new

spawn do
  5.times do |i|
    channel.send(i)
    puts "ส่ง: #{i}"
  end
  channel.close
end

# รับข้อมูลจาก channel
while value = channel.receive?
  puts "รับ: #{value}"
end
```

### Buffered Channel เป็น Queue

```crystal
# Buffered channel ทำหน้าที่เป็น queue ได้
queue = Channel(String).new(10) # buffer size 10

# Producer
spawn do
  ["งาน1", "งาน2", "งาน3", "งาน4", "งาน5"].each do |job|
    queue.send(job)
    puts "เพิ่มงาน: #{job}"
  end
  queue.close
end

# Consumer
spawn do
  while job = queue.receive?
    puts "ประมวลผล: #{job}"
    sleep 0.1.seconds
  end
end

sleep 2.seconds
```

### Priority Queue ด้วย Channel

```crystal
# สร้าง Priority Queue อย่างง่าย
class PriorityJob
  property priority : Int32
  property data : String
  
  def initialize(@priority, @data)
  end
  
  def <=>(other : PriorityJob)
    priority <=> other.priority
  end
end

class PriorityQueue(T)
  def initialize
    @high_priority = Channel(T).new(100)
    @low_priority = Channel(T).new(100)
    @mutex = Mutex.new
  end
  
  def enqueue(item : T, high : Bool = false)
    if high
      @high_priority.send(item)
    else
      @low_priority.send(item)
    end
  end
  
  def dequeue : T?
    # ลองรับจาก high priority ก่อน
    select
    when item = @high_priority.receive?
      item
    when item = @low_priority.receive?
      item
    end
  end
end

pq = PriorityQueue(String).new

spawn do
  pq.enqueue("งานปกติ 1", high: false)
  pq.enqueue("งานเร่งด่วน!", high: true)
  pq.enqueue("งานปกติ 2", high: false)
  pq.enqueue("งานเร่งด่วน 2!", high: true)
end

sleep 0.1.seconds

4.times do
  if item = pq.dequeue
    puts "ดำเนินการ: #{item}"
  end
end
```

## Mutex-Protected Structures

Mutex (Mutual Exclusion) ใช้เพื่อป้องกันไม่ให้หลาย thread เข้าถึงข้อมูลพร้อมกัน

### Mutex พื้นฐาน

```crystal
# ตัวอย่างปัญหา race condition
counter = 0
fibers = [] of Fiber

10.times do
  fibers << spawn do
    1000.times { counter += 1 }
  end
end

# รอทุก fiber เสร็จ
Fiber.yield

puts "ผลลัพธ์ไม่แน่นอน: #{counter}" # อาจไม่ใช่ 10000
```

```crystal
# แก้ไขด้วย Mutex
mutex = Mutex.new
counter = 0

10.times do
  spawn do
    1000.times do
      mutex.synchronize { counter += 1 }
    end
  end
end

sleep 1.second
puts "ผลลัพธ์ถูกต้อง: #{counter}" # = 10000
```

### Thread-Safe Counter

```crystal
class SafeCounter
  def initialize
    @value = 0
    @mutex = Mutex.new
  end
  
  def increment
    @mutex.synchronize { @value += 1 }
  end
  
  def decrement
    @mutex.synchronize { @value -= 1 }
  end
  
  def value
    @mutex.synchronize { @value }
  end
  
  def reset
    @mutex.synchronize { @value = 0 }
  end
end

counter = SafeCounter.new

# หลาย fiber เพิ่มค่าพร้อมกัน
5.times do
  spawn do
    100.times { counter.increment }
  end
end

sleep 0.5.seconds
puts "ค่า counter: #{counter.value}" # = 500
```

### Thread-Safe Hash

```crystal
class SafeHash(K, V)
  def initialize
    @hash = {} of K => V
    @mutex = Mutex.new
  end
  
  def []=(key : K, value : V)
    @mutex.synchronize { @hash[key] = value }
  end
  
  def [](key : K)
    @mutex.synchronize { @hash[key] }
  end
  
  def []?(key : K)
    @mutex.synchronize { @hash[key]? }
  end
  
  def delete(key : K)
    @mutex.synchronize { @hash.delete(key) }
  end
  
  def keys
    @mutex.synchronize { @hash.keys.dup }
  end
  
  def values
    @mutex.synchronize { @hash.values.dup }
  end
  
  def size
    @mutex.synchronize { @hash.size }
  end
  
  def each(&block : K, V ->)
    # สร้าง snapshot เพื่อหลีกเลี่ยง deadlock
    snapshot = @mutex.synchronize { @hash.dup }
    snapshot.each { |k, v| block.call(k, v) }
  end
end

# การใช้งาน
cache = SafeHash(String, Int32).new

# หลาย fiber เขียนพร้อมกัน
10.times do |i|
  spawn do
    cache["key_#{i}"] = i * 10
  end
end

sleep 0.2.seconds
cache.each { |k, v| puts "#{k}: #{v}" }
```

### Thread-Safe Array

```crystal
class SafeArray(T)
  include Enumerable(T)
  
  def initialize
    @array = [] of T
    @mutex = Mutex.new
  end
  
  def push(item : T)
    @mutex.synchronize { @array.push(item) }
    self
  end
  
  def pop : T?
    @mutex.synchronize { @array.pop? }
  end
  
  def shift : T?
    @mutex.synchronize { @array.shift? }
  end
  
  def size
    @mutex.synchronize { @array.size }
  end
  
  def empty?
    @mutex.synchronize { @array.empty? }
  end
  
  def each(&block : T ->)
    snapshot = @mutex.synchronize { @array.dup }
    snapshot.each { |item| block.call(item) }
  end
  
  def to_a
    @mutex.synchronize { @array.dup }
  end
end

results = SafeArray(String).new

5.times do |i|
  spawn do
    results.push("ผลลัพธ์จาก fiber #{i}")
  end
end

sleep 0.2.seconds
results.each { |r| puts r }
```

## Thread-Safe Patterns

### Read-Write Lock Pattern

```crystal
# RWLock: อนุญาตให้อ่านพร้อมกันได้หลายคน แต่เขียนได้ทีละคน
class RWLock
  def initialize
    @readers = 0
    @mutex = Mutex.new
    @write_mutex = Mutex.new
    @read_mutex = Mutex.new
  end
  
  def read_lock
    @read_mutex.synchronize do
      @readers += 1
      @write_mutex.lock if @readers == 1
    end
  end
  
  def read_unlock
    @read_mutex.synchronize do
      @readers -= 1
      @write_mutex.unlock if @readers == 0
    end
  end
  
  def write_lock
    @write_mutex.lock
  end
  
  def write_unlock
    @write_mutex.unlock
  end
  
  def with_read(&block)
    read_lock
    begin
      block.call
    ensure
      read_unlock
    end
  end
  
  def with_write(&block)
    write_lock
    begin
      block.call
    ensure
      write_unlock
    end
  end
end

# การใช้งาน
data = {} of String => String
rwlock = RWLock.new

# Writer
spawn do
  3.times do |i|
    rwlock.with_write do
      data["key#{i}"] = "value#{i}"
      puts "เขียน: key#{i} = value#{i}"
    end
    sleep 0.2.seconds
  end
end

# หลาย Reader พร้อมกัน
3.times do |reader_id|
  spawn do
    5.times do
      rwlock.with_read do
        puts "Reader #{reader_id} อ่าน: #{data.inspect}"
      end
      sleep 0.1.seconds
    end
  end
end

sleep 2.seconds
```

### Double-Checked Locking Pattern

```crystal
# Singleton pattern ที่ thread-safe
class DatabaseConnection
  @@instance : DatabaseConnection? = nil
  @@mutex = Mutex.new
  
  def self.instance
    # เช็คครั้งแรกโดยไม่ lock (เร็ว)
    return @@instance.not_nil! if @@instance
    
    # lock และเช็คอีกครั้ง
    @@mutex.synchronize do
      @@instance ||= new
    end
    
    @@instance.not_nil!
  end
  
  private def initialize
    puts "สร้าง DatabaseConnection..."
    @connected = true
  end
  
  def query(sql : String)
    puts "ประมวลผล: #{sql}"
  end
end

# ทดสอบ thread-safety
5.times do
  spawn do
    db = DatabaseConnection.instance
    db.query("SELECT * FROM users")
  end
end

sleep 0.5.seconds
```

### Actor Pattern

```crystal
# Actor pattern: ข้อความ-based concurrency
class Actor(T)
  def initialize(&@handler : T ->)
    @mailbox = Channel(T).new(100)
    start
  end
  
  def send(message : T)
    @mailbox.send(message)
  end
  
  private def start
    spawn do
      while msg = @mailbox.receive?
        @handler.call(msg)
      end
    end
  end
  
  def stop
    @mailbox.close
  end
end

# Actor สำหรับนับจำนวน
counter_actor = Actor(Symbol).new do |msg|
  @@count = 0 unless defined?(@@count)
  case msg
  when :increment
    @@count += 1
  when :print
    puts "Count: #{@@count}"
  end
end

10.times { counter_actor.send(:increment) }
counter_actor.send(:print)
sleep 0.2.seconds
counter_actor.stop
```

### Future/Promise Pattern

```crystal
class Future(T)
  def initialize(&computation : -> T)
    @channel = Channel(T | Exception).new(1)
    @value : T? = nil
    @exception : Exception? = nil
    @done = false
    @mutex = Mutex.new
    
    spawn do
      begin
        result = computation.call
        @channel.send(result)
      rescue ex
        @channel.send(ex)
      end
    end
  end
  
  def get : T
    unless @done
      result = @channel.receive
      @mutex.synchronize do
        case result
        when Exception
          @exception = result
        else
          @value = result.as(T)
        end
        @done = true
      end
    end
    
    if ex = @exception
      raise ex
    end
    
    @value.not_nil!
  end
  
  def completed?
    @done
  end
end

# การใช้งาน
future1 = Future(Int32).new { sleep 0.5.seconds; 42 }
future2 = Future(String).new { sleep 0.3.seconds; "สวัสดี" }

puts "รอผลลัพธ์..."
puts "Future1: #{future1.get}"
puts "Future2: #{future2.get}"
```

## Concurrent Queue Implementations

### Blocking Queue

```crystal
class BlockingQueue(T)
  def initialize(@max_size : Int32 = Int32::MAX)
    @queue = Deque(T).new
    @mutex = Mutex.new
    @not_empty = Channel(Nil).new(1)
    @not_full = Channel(Nil).new(1)
  end
  
  def enqueue(item : T)
    @mutex.synchronize do
      while @queue.size >= @max_size
        # รอจนกว่า queue จะว่าง
        @mutex.unlock
        @not_full.receive
        @mutex.lock
      end
      
      @queue.push(item)
      @not_empty.send(nil) rescue nil
    end
  end
  
  def dequeue : T
    @mutex.synchronize do
      while @queue.empty?
        @mutex.unlock
        @not_empty.receive
        @mutex.lock
      end
      
      item = @queue.shift
      @not_full.send(nil) rescue nil
      item
    end
  end
  
  def size
    @mutex.synchronize { @queue.size }
  end
end

bq = BlockingQueue(Int32).new(5)

# Producer
spawn do
  10.times do |i|
    bq.enqueue(i)
    puts "เพิ่ม: #{i} (size: #{bq.size})"
  end
end

# Consumer
spawn do
  10.times do
    item = bq.dequeue
    puts "นำออก: #{item}"
    sleep 0.1.seconds
  end
end

sleep 3.seconds
```

### Ring Buffer (Circular Buffer)

```crystal
class RingBuffer(T)
  def initialize(@capacity : Int32)
    @buffer = Array(T?).new(@capacity, nil)
    @head = 0
    @tail = 0
    @size = 0
    @mutex = Mutex.new
  end
  
  def push(item : T) : Bool
    @mutex.synchronize do
      return false if @size == @capacity
      
      @buffer[@tail] = item
      @tail = (@tail + 1) % @capacity
      @size += 1
      true
    end
  end
  
  def pop : T?
    @mutex.synchronize do
      return nil if @size == 0
      
      item = @buffer[@head]
      @buffer[@head] = nil
      @head = (@head + 1) % @capacity
      @size -= 1
      item
    end
  end
  
  def size
    @mutex.synchronize { @size }
  end
  
  def full?
    @mutex.synchronize { @size == @capacity }
  end
  
  def empty?
    @mutex.synchronize { @size == 0 }
  end
end

rb = RingBuffer(String).new(3)

rb.push("a")
rb.push("b")
rb.push("c")
puts "เต็ม: #{rb.full?}"
puts rb.push("d") # false - เต็มแล้ว

puts rb.pop # "a"
puts rb.push("d") # true
puts "Size: #{rb.size}"
```

## Semaphore Pattern

```crystal
class Semaphore
  def initialize(@count : Int32)
    @channel = Channel(Nil).new(@count)
    @count.times { @channel.send(nil) }
  end
  
  def acquire
    @channel.receive
  end
  
  def release
    @channel.send(nil)
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

# จำกัดจำนวน concurrent operations
sem = Semaphore.new(3) # อนุญาตให้ทำงานพร้อมกันได้ 3 อัน

10.times do |i|
  spawn do
    sem.synchronize do
      puts "เริ่มงาน #{i}"
      sleep 0.5.seconds
      puts "เสร็จงาน #{i}"
    end
  end
end

sleep 3.seconds
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Thread-Safe Cache

สร้าง LRU Cache ที่ thread-safe:

```crystal
class LRUCache(K, V)
  def initialize(@max_size : Int32)
    @cache = {} of K => V
    @order = Deque(K).new
    @mutex = Mutex.new
  end
  
  def get(key : K) : V?
    @mutex.synchronize do
      if @cache.has_key?(key)
        # ย้าย key ไปท้าย (most recently used)
        @order.delete(key)
        @order.push(key)
        @cache[key]
      else
        nil
      end
    end
  end
  
  def put(key : K, value : V)
    @mutex.synchronize do
      if @cache.has_key?(key)
        @order.delete(key)
      elsif @cache.size >= @max_size
        # ลบ least recently used
        oldest = @order.shift
        @cache.delete(oldest)
      end
      
      @cache[key] = value
      @order.push(key)
    end
  end
  
  def size
    @mutex.synchronize { @cache.size }
  end
end

# ทดสอบ
cache = LRUCache(String, Int32).new(3)
cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)
puts cache.get("a") # 1
cache.put("d", 4)   # ลบ "b" (LRU)
puts cache.get("b") # nil
puts cache.size     # 3
```

### แบบฝึกหัดที่ 2: Event Bus

```crystal
class EventBus
  def initialize
    @handlers = Hash(String, Array(Proc(String, Nil))).new { |h, k| h[k] = [] of Proc(String, Nil) }
    @mutex = Mutex.new
    @channel = Channel(Tuple(String, String)).new(100)
    start_dispatcher
  end
  
  def subscribe(event : String, &handler : String ->)
    @mutex.synchronize do
      @handlers[event] << handler
    end
  end
  
  def publish(event : String, data : String)
    @channel.send({event, data})
  end
  
  private def start_dispatcher
    spawn do
      while msg = @channel.receive?
        event, data = msg
        handlers = @mutex.synchronize { @handlers[event].dup }
        handlers.each { |h| h.call(data) }
      end
    end
  end
end

bus = EventBus.new

bus.subscribe("user.created") { |data| puts "ผู้ใช้ใหม่: #{data}" }
bus.subscribe("user.created") { |data| puts "ส่งอีเมลต้อนรับ: #{data}" }
bus.subscribe("order.placed") { |data| puts "คำสั่งซื้อ: #{data}" }

bus.publish("user.created", "สมชาย ใจดี")
bus.publish("order.placed", "#12345")
bus.publish("user.created", "สมหญิง ใจงาม")

sleep 0.5.seconds
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Channel เป็น Queue**: การใช้ Channel แบบ buffered เพื่อสร้าง queue ที่ thread-safe
2. **Mutex**: การใช้ Mutex.synchronize เพื่อป้องกัน race condition
3. **Thread-Safe Collections**: SafeCounter, SafeHash, SafeArray
4. **Design Patterns**: 
   - Read-Write Lock สำหรับ read-heavy workloads
   - Double-Checked Locking สำหรับ Singleton
   - Actor Pattern สำหรับ message-based concurrency
   - Future/Promise สำหรับ async results
5. **Semaphore**: จำกัดจำนวน concurrent operations

หลักสำคัญที่ต้องจำ:
- ใช้ Mutex เมื่อต้องการ mutual exclusion
- ใช้ Channel เมื่อต้องการสื่อสารระหว่าง fiber
- หลีกเลี่ยง deadlock โดยระวังลำดับการ lock
- ทำ snapshot ก่อน iterate เพื่อหลีกเลี่ยง deadlock
