# Part 114: Mutex และ Thread Safety

## บทนำ

เมื่อ Crystal ทำงาน multi-threaded (ด้วย `-Dpreview_mt` flag) การ protect shared data เป็นสิ่งสำคัญ Crystal มี Mutex, RWLock, และ Atomic types สำหรับ thread safety

## Mutex พื้นฐาน

```crystal
# Mutex - mutual exclusion lock
mutex = Mutex.new

# ตัวแปร shared
counter = 0

# ใช้ mutex.synchronize เพื่อ protect critical section
mutex.synchronize do
  counter += 1
end

# หรือ lock/unlock
mutex.lock
counter += 1
mutex.unlock

# ตัวอย่าง - concurrent increment (ใช้ -Dpreview_mt)
shared_count = 0
mutex = Mutex.new

workers = Array.new(10) do
  spawn do
    100.times do
      mutex.synchronize { shared_count += 1 }
    end
  end
end

# ให้ workers ทำงาน (ใน multi-threaded mode)
# shared_count should be 1000 after all workers finish

puts "Final count: #{shared_count}"
```

## mutex.synchronize

```crystal
# synchronize เป็น block form ของ lock/unlock
class BankAccount
  getter balance : Float64

  def initialize(initial_balance : Float64)
    @balance = initial_balance
    @mutex = Mutex.new
  end

  def deposit(amount : Float64) : Bool
    return false if amount <= 0

    @mutex.synchronize do
      @balance += amount
      puts "Deposited #{amount}, balance: #{@balance}"
    end
    true
  end

  def withdraw(amount : Float64) : Bool
    @mutex.synchronize do
      if amount > @balance
        puts "Insufficient funds"
        return false
      end
      @balance -= amount
      puts "Withdrew #{amount}, balance: #{@balance}"
    end
    true
  end

  def transfer(amount : Float64, target : BankAccount) : Bool
    return false if amount <= 0

    # ต้อง lock ทั้งสอง accounts - ระวัง deadlock!
    # ใช้ consistent ordering เพื่อป้องกัน deadlock
    first, second = object_id < target.object_id ? {self, target} : {target, self}

    first.mutex.lock
    second.mutex.lock

    begin
      if @balance >= amount
        @balance -= amount
        target.balance_add_unsafe(amount)
        true
      else
        false
      end
    ensure
      second.mutex.unlock
      first.mutex.unlock
    end
  end

  protected def mutex : Mutex
    @mutex
  end

  protected def balance_add_unsafe(amount : Float64)
    @balance += amount
  end
end

acc1 = BankAccount.new(1000.0)
acc2 = BankAccount.new(500.0)

acc1.deposit(200.0)
acc1.withdraw(100.0)
acc1.transfer(300.0, acc2)

puts "Account 1: #{acc1.balance}"  # => 800.0
puts "Account 2: #{acc2.balance}"  # => 800.0
```

## RWLock (Read-Write Lock)

```crystal
# RWLock - หลาย readers, หนึ่ง writer
# Crystal มี Thread::Mutex แต่ RWLock ต้องสร้างเอง (ใน standard lib เรียก RWLock)

class ReadWriteCache(K, V)
  def initialize
    @data = Hash(K, V).new
    @lock = RWLock.new
  end

  # Multiple concurrent reads allowed
  def get(key : K) : V?
    @lock.read_lock do
      @data[key]?
    end
  end

  # Exclusive write access
  def set(key : K, value : V)
    @lock.write_lock do
      @data[key] = value
    end
  end

  def delete(key : K) : V?
    @lock.write_lock do
      @data.delete(key)
    end
  end

  def size : Int32
    @lock.read_lock { @data.size }
  end
end

# ใช้ Mutex เป็น approximation (ใน single-threaded env)
class SimpleCache(K, V)
  def initialize
    @data = Hash(K, V).new
    @mutex = Mutex.new
  end

  def get(key : K) : V?
    @mutex.synchronize { @data[key]? }
  end

  def set(key : K, value : V)
    @mutex.synchronize { @data[key] = value }
  end
end

cache = SimpleCache(String, String).new
cache.set("greeting", "สวัสดีครับ")
puts cache.get("greeting")  # => สวัสดีครับ
```

## Thread-safe Data Structures

```crystal
# Thread-safe Queue
class ConcurrentQueue(T)
  def initialize
    @data = [] of T
    @mutex = Mutex.new
  end

  def enqueue(item : T)
    @mutex.synchronize { @data.push(item) }
  end

  def dequeue : T?
    @mutex.synchronize { @data.shift? }
  end

  def peek : T?
    @mutex.synchronize { @data.first? }
  end

  def size : Int32
    @mutex.synchronize { @data.size }
  end

  def empty? : Bool
    @mutex.synchronize { @data.empty? }
  end

  def to_a : Array(T)
    @mutex.synchronize { @data.dup }
  end
end

# ใช้งาน
queue = ConcurrentQueue(String).new
queue.enqueue("task1")
queue.enqueue("task2")
queue.enqueue("task3")

while item = queue.dequeue
  puts "Processing: #{item}"
end

# Thread-safe Stack
class ConcurrentStack(T)
  def initialize
    @data = [] of T
    @mutex = Mutex.new
  end

  def push(item : T) : self
    @mutex.synchronize { @data.push(item) }
    self
  end

  def pop : T?
    @mutex.synchronize { @data.pop? }
  end

  def size : Int32
    @mutex.synchronize { @data.size }
  end
end

stack = ConcurrentStack(Int32).new
stack.push(1).push(2).push(3)

while item = stack.pop
  puts item
end
```

## Atomic Types

```crystal
# Atomic operations สำหรับ simple counter ที่ thread-safe
# Crystal มี Atomic(T) สำหรับ atomic operations

# ตัวอย่าง Atomic counter
atomic_counter = Atomic(Int32).new(0)

# Atomic add - thread-safe ไม่ต้องใช้ mutex
atomic_counter.add(1)
atomic_counter.add(1)
puts atomic_counter.get  # => 2

# Compare and swap
success = atomic_counter.compare_and_set(2, 10)
puts "CAS success: #{success}"    # => true
puts atomic_counter.get           # => 10

# ล้มเหลวถ้า expected value ไม่ตรง
success2 = atomic_counter.compare_and_set(5, 20)  # expected 5, actual 10
puts "CAS success: #{success2}"   # => false
puts atomic_counter.get           # => 10 (ไม่เปลี่ยน)

# Fetch and add
old_val = atomic_counter.add(5)  # return old value
puts "Old: #{old_val}, New: #{atomic_counter.get}"  # => Old: 10, New: 15
```

## Concurrent Counter ด้วย Atomic

```crystal
# Counter ที่ thread-safe ด้วย Atomic
class AtomicCounter
  def initialize(initial : Int32 = 0)
    @value = Atomic(Int32).new(initial)
  end

  def increment : Int32
    @value.add(1)
  end

  def decrement : Int32
    @value.add(-1)
  end

  def add(n : Int32) : Int32
    @value.add(n)
  end

  def get : Int32
    @value.get
  end

  def reset : Int32
    @value.swap(0)
  end
end

# สร้าง metrics collector ที่ thread-safe
class Metrics
  def initialize
    @request_count = AtomicCounter.new
    @error_count = AtomicCounter.new
    @bytes_sent = Atomic(Int64).new(0_i64)
  end

  def record_request(bytes : Int64 = 0)
    @request_count.increment
    @bytes_sent.add(bytes) if bytes > 0
  end

  def record_error
    @error_count.increment
    @request_count.increment
  end

  def stats : String
    total = @request_count.get
    errors = @error_count.get
    bytes = @bytes_sent.get

    "Requests: #{total}, Errors: #{errors}, Bytes: #{bytes}"
  end
end

metrics = Metrics.new
metrics.record_request(1024_i64)
metrics.record_request(2048_i64)
metrics.record_error
metrics.record_request(512_i64)

puts metrics.stats
# => Requests: 4, Errors: 1, Bytes: 3584
```

## Mutex กับ Condition Variables

```crystal
# Condition variable-like pattern ด้วย Channel
class BoundedBuffer(T)
  def initialize(@capacity : Int32)
    @buffer = [] of T
    @mutex = Mutex.new
    @not_full = Channel(Nil).new(@capacity)
    @not_empty = Channel(Nil).new(@capacity)
  end

  def put(item : T)
    @mutex.synchronize do
      while @buffer.size >= @capacity
        # Wait for space
        @mutex.unlock
        Fiber.yield
        @mutex.lock
      end
      @buffer << item
    end
  end

  def take : T
    @mutex.synchronize do
      while @buffer.empty?
        @mutex.unlock
        Fiber.yield
        @mutex.lock
      end
      @buffer.shift
    end
  end

  def size : Int32
    @mutex.synchronize { @buffer.size }
  end
end

# ตัวอย่าง simple thread-safe producer-consumer
buffer = BoundedBuffer(String).new(5)

spawn do
  ["item1", "item2", "item3"].each do |item|
    buffer.put(item)
    puts "Put: #{item}"
  end
end

spawn do
  3.times do
    item = buffer.take
    puts "Took: #{item}"
  end
end

sleep 0.01
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Read-Heavy Cache
สร้าง cache ที่ optimize สำหรับ reads:
- Many concurrent readers
- Occasional writer
- Cache invalidation

### แบบฝึกหัดที่ 2: Task Queue with Workers
สร้าง thread-safe task queue:
- Multiple workers consume tasks
- Priority-based ordering
- Cancellation support

### แบบฝึกหัดที่ 3: Lock-free Stack
Implement lock-free stack ด้วย Atomic:
- Compare-and-swap operations
- No mutex needed
- ABA problem prevention

### แบบฝึกหัดที่ 4: Rate Limiter
สร้าง thread-safe rate limiter:
- Token bucket algorithm
- Atomic operations
- Per-client limits

## สรุป

Thread Safety ใน Crystal:
- **Mutex**: exclusive access, `synchronize` block
- **RWLock**: multiple readers OR one writer
- **Atomic(T)**: lock-free operations สำหรับ simple types
- **Channels**: ปลอดภัยที่สุดสำหรับ communication

Best practices:
1. ใช้ Channels แทน shared memory เมื่อทำได้
2. ใช้ Atomic สำหรับ simple counters/flags
3. ใช้ Mutex สำหรับ critical sections ที่ซับซ้อน
4. ระวัง deadlock ใน nested locks
5. Prefer immutable data ที่ไม่ต้องการ synchronization
