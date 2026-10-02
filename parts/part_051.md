# Part 51: Deque ใน Crystal

## บทนำ

Deque (Double-Ended Queue) คือ data structure ที่สามารถเพิ่มและลบ elements ได้ทั้งสองฝั่ง (หน้าและหลัง) อย่างมีประสิทธิภาพ ต่างจาก Array ที่การ insert/delete ที่ตำแหน่งหน้าจะช้า (O(n)) Deque ทำได้ใน O(1)

---

## 51.1 พื้นฐาน Deque

```crystal
require "deque"  # หรือจะใช้โดยไม่ require ก็ได้ใน Crystal std

deque = Deque(Int32).new

# push_back (เพิ่มท้าย)
deque.push_back(1)
deque.push_back(2)
deque.push_back(3)
puts deque.inspect  # => Deque{1, 2, 3}

# push_front (เพิ่มหน้า)
deque.push_front(0)
deque.push_front(-1)
puts deque.inspect  # => Deque{-1, 0, 1, 2, 3}

# pop_back (ลบและคืนท้าย)
back = deque.pop_back
puts back           # => 3
puts deque.inspect  # => Deque{-1, 0, 1, 2}

# pop_front (ลบและคืนหน้า)
front = deque.pop_front
puts front          # => -1
puts deque.inspect  # => Deque{0, 1, 2}
```

---

## 51.2 push_back และ push_front

```crystal
dq = Deque(String).new

# push_back
dq.push_back("middle")
dq.push_back("end")
puts dq.inspect  # => Deque{"middle", "end"}

# push_front
dq.push_front("start")
puts dq.inspect  # => Deque{"start", "middle", "end"}

# << operator (push_back)
dq << "last"
puts dq.inspect  # => Deque{"start", "middle", "end", "last"}

# unshift (alias ของ push_front)
dq.unshift("first")
puts dq.inspect  # => Deque{"first", "start", "middle", "end", "last"}
```

---

## 51.3 pop_back และ pop_front

```crystal
dq = Deque{1, 2, 3, 4, 5}

# pop_back
puts dq.pop_back    # => 5
puts dq.pop_back    # => 4
puts dq.inspect     # => Deque{1, 2, 3}

# pop_front
puts dq.pop_front   # => 1
puts dq.pop_front   # => 2
puts dq.inspect     # => Deque{3}

# pop? และ shift? (safe versions)
empty_dq = Deque(Int32).new
puts empty_dq.pop?    # => nil (ไม่ raise)
puts empty_dq.shift?  # => nil

# shift (alias ของ pop_front)
dq2 = Deque{10, 20, 30}
puts dq2.shift    # => 10
puts dq2.inspect  # => Deque{20, 30}
```

---

## 51.4 first และ last

```crystal
dq = Deque{10, 20, 30, 40, 50}

# first (peek หน้า)
puts dq.first    # => 10
puts dq.first?   # => 10 (safe)

# last (peek ท้าย)
puts dq.last     # => 50
puts dq.last?    # => 50 (safe)

# ไม่เปลี่ยน deque
puts dq.inspect  # => Deque{10, 20, 30, 40, 50}

# first(n) และ last(n)
puts dq.first(3).inspect  # => [10, 20, 30]
puts dq.last(2).inspect   # => [40, 50]
```

---

## 51.5 Deque เป็น Queue (FIFO)

Queue คือ "First In, First Out" - เพิ่มท้าย ลบหน้า

```crystal
class Queue(T)
  def initialize
    @data = Deque(T).new
  end
  
  def enqueue(item : T)
    @data.push_back(item)  # เพิ่มท้าย
  end
  
  def dequeue : T
    raise "Queue is empty" if empty?
    @data.pop_front  # ลบหน้า
  end
  
  def peek : T
    @data.first
  end
  
  def size : Int32
    @data.size
  end
  
  def empty? : Bool
    @data.empty?
  end
  
  def to_s(io : IO) : Nil
    io << "Queue#{@data}"
  end
end

queue = Queue(String).new
queue.enqueue("Task 1")
queue.enqueue("Task 2")
queue.enqueue("Task 3")

puts queue        # => Queue Deque{"Task 1", "Task 2", "Task 3"}
puts queue.peek   # => "Task 1"

puts queue.dequeue  # => "Task 1"
puts queue.dequeue  # => "Task 2"
puts queue.size     # => 1
```

### ตัวอย่าง BFS ด้วย Queue

```crystal
class Graph
  def initialize
    @edges = Hash(String, Array(String)).new { |h, k| h[k] = [] of String }
  end
  
  def add_edge(from : String, to : String)
    @edges[from] << to
  end
  
  def bfs(start : String) : Array(String)
    visited = [] of String
    queue = Deque(String).new
    seen = Set(String).new
    
    queue.push_back(start)
    seen.add(start)
    
    while !queue.empty?
      node = queue.pop_front
      visited << node
      
      @edges[node].each do |neighbor|
        unless seen.includes?(neighbor)
          seen.add(neighbor)
          queue.push_back(neighbor)
        end
      end
    end
    
    visited
  end
end

require "set"
g = Graph.new
g.add_edge("A", "B")
g.add_edge("A", "C")
g.add_edge("B", "D")
g.add_edge("C", "D")
g.add_edge("D", "E")

puts g.bfs("A").inspect  # => ["A", "B", "C", "D", "E"]
```

---

## 51.6 Deque เป็น Stack (LIFO)

Stack คือ "Last In, First Out" - เพิ่มและลบที่เดียวกัน

```crystal
class Stack(T)
  def initialize
    @data = Deque(T).new
  end
  
  def push(item : T)
    @data.push_back(item)  # เพิ่มท้าย
  end
  
  def pop : T
    raise "Stack is empty" if empty?
    @data.pop_back  # ลบท้าย
  end
  
  def peek : T
    @data.last
  end
  
  def size : Int32
    @data.size
  end
  
  def empty? : Bool
    @data.empty?
  end
  
  def to_s(io : IO) : Nil
    io << "Stack#{@data}"
  end
end

stack = Stack(Int32).new
stack.push(1)
stack.push(2)
stack.push(3)

puts stack        # => Stack Deque{1, 2, 3}
puts stack.peek   # => 3

puts stack.pop  # => 3
puts stack.pop  # => 2
puts stack.size # => 1
```

### ตัวอย่าง: Expression Evaluator

```crystal
def evaluate_rpn(expression : String) : Float64
  stack = Deque(Float64).new
  
  expression.split.each do |token|
    case token
    when /^-?\d+(\.\d+)?$/
      stack.push_back(token.to_f)
    when "+"
      b, a = stack.pop_back, stack.pop_back
      stack.push_back(a + b)
    when "-"
      b, a = stack.pop_back, stack.pop_back
      stack.push_back(a - b)
    when "*"
      b, a = stack.pop_back, stack.pop_back
      stack.push_back(a * b)
    when "/"
      b, a = stack.pop_back, stack.pop_back
      stack.push_back(a / b)
    end
  end
  
  stack.pop_back
end

# RPN: 3 4 + 2 * = (3+4)*2 = 14
puts evaluate_rpn("3 4 + 2 *")  # => 14.0

# RPN: 5 1 2 + 4 * + 3 - = 5 + (1+2)*4 - 3 = 14
puts evaluate_rpn("5 1 2 + 4 * + 3 -")  # => 14.0
```

---

## 51.7 Performance Benefits

```crystal
# Performance comparison: Array vs Deque
# สำหรับการ insert ที่ตำแหน่งหน้า

# Array: O(n) เพราะต้องเลื่อน elements
arr = [] of Int32
10000.times { |i| arr.unshift(i) }  # ช้ามากสำหรับ array ขนาดใหญ่

# Deque: O(1) amortized
dq = Deque(Int32).new
10000.times { |i| dq.push_front(i) }  # เร็วมาก

puts "Array size: #{arr.size}"   # => 10000
puts "Deque size: #{dq.size}"    # => 10000

# สำหรับการ access ตรงกลาง: Array ดีกว่า
arr_access = (1..1000).to_a
dq_access = Deque(Int32).new
arr_access.each { |n| dq_access.push_back(n) }

puts arr_access[500]    # O(1)
puts dq_access[500]     # O(1) แต่ใน practice อาจช้ากว่า
```

---

## 51.8 Deque Methods ครบถ้วน

```crystal
dq = Deque{1, 2, 3, 4, 5}

# เข้าถึง
puts dq[0]        # => 1
puts dq[-1]       # => 5
puts dq[1..3].inspect  # => [2, 3, 4]

# Modification
dq[2] = 99
puts dq.inspect  # => Deque{1, 2, 99, 4, 5}

# Insert
dq.insert(2, 50)
puts dq.inspect  # => Deque{1, 2, 50, 99, 4, 5}

# Delete
dq.delete(99)
puts dq.inspect  # => Deque{1, 2, 50, 4, 5}

# Rotation
dq.rotate!(2)
puts dq.inspect  # => Deque{50, 4, 5, 1, 2}

# Sorting
dq.sort!
puts dq.inspect  # => Deque{1, 2, 4, 5, 50}

# Iteration
dq.each { |n| print "#{n} " }
puts

dq.each_with_index { |n, i| print "#{i}:#{n} " }
puts

# Map, select, etc.
doubled = dq.map { |n| n * 2 }
puts doubled.inspect   # => [2, 4, 8, 10, 100]

evens = dq.select { |n| n.even? }
puts evens.inspect     # => [2, 4, 50]
```

---

## 51.9 ตัวอย่างในชีวิตจริง

### Sliding Window Algorithm

```crystal
def sliding_window_max(arr : Array(Int32), k : Int32) : Array(Int32)
  result = [] of Int32
  window = Deque(Int32).new  # เก็บ indices
  
  arr.each_with_index do |num, i|
    # ลบ elements ที่ไม่อยู่ใน window
    while !window.empty? && window.first < i - k + 1
      window.pop_front
    end
    
    # ลบ elements ที่น้อยกว่า current
    while !window.empty? && arr[window.last] < num
      window.pop_back
    end
    
    window.push_back(i)
    
    if i >= k - 1
      result << arr[window.first]
    end
  end
  
  result
end

arr = [1, 3, -1, -3, 5, 3, 6, 7]
puts sliding_window_max(arr, 3).inspect
# => [3, 3, 5, 5, 6, 7]
```

### Recent Items Cache

```crystal
class RecentItems(T)
  def initialize(@max_size : Int32)
    @items = Deque(T).new
  end
  
  def add(item : T)
    # ลบ duplicate
    @items.delete(item)
    
    # เพิ่มที่ด้านหน้า (most recent)
    @items.push_front(item)
    
    # จำกัดขนาด
    while @items.size > @max_size
      @items.pop_back
    end
  end
  
  def recent : Array(T)
    @items.to_a
  end
  
  def most_recent : T?
    @items.first?
  end
  
  def size : Int32
    @items.size
  end
  
  def to_s(io : IO) : Nil
    io << "Recent#{@items}"
  end
end

# Browser history
history = RecentItems(String).new(5)
history.add("google.com")
history.add("github.com")
history.add("crystal-lang.org")
history.add("example.com")
history.add("google.com")  # Visit google again - moves to front

puts history  # => Recent Deque{"google.com", "example.com", "crystal-lang.org", "github.com"}
puts "Most recent: #{history.most_recent}"
puts "History: #{history.recent.join(" -> ")}"
```

### Event Queue

```crystal
class EventQueue(T)
  def initialize(@capacity : Int32)
    @queue = Deque(T).new
    @processed = 0
  end
  
  def publish(event : T, priority : Bool = false)
    if priority
      @queue.push_front(event)  # High priority goes to front
    else
      @queue.push_back(event)   # Normal goes to back
    end
    
    # Trim if over capacity
    if @queue.size > @capacity
      @queue.pop_back  # Drop oldest low priority
    end
  end
  
  def process : T?
    return nil if @queue.empty?
    @processed += 1
    @queue.pop_front
  end
  
  def process_all : Array(T)
    results = [] of T
    while !@queue.empty?
      results << process.not_nil!
    end
    results
  end
  
  def size : Int32
    @queue.size
  end
  
  def processed : Int32
    @processed
  end
end

events = EventQueue(String).new(10)
events.publish("Login user")
events.publish("Update profile")
events.publish("CRITICAL: System error!", priority: true)
events.publish("Send email")
events.publish("URGENT: Payment failed!", priority: true)

puts "Processing events:"
all = events.process_all
all.each { |e| puts "  - #{e}" }
puts "Processed: #{events.processed}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Palindrome Checker

```crystal
def palindrome?(str : String) : Bool
  chars = Deque(Char).new
  str.downcase.gsub(/[^a-z]/, "").each_char { |c| chars.push_back(c) }
  
  while chars.size > 1
    return false if chars.pop_front != chars.pop_back
  end
  
  true
end

puts palindrome?("racecar")     # => true
puts palindrome?("hello")       # => false
puts palindrome?("A man a plan a canal Panama")  # => true
puts palindrome?("Was it a car or a cat I saw")  # => true
```

### แบบฝึกหัดที่ 2: Task Scheduler

```crystal
class Task
  getter name : String
  getter priority : Int32
  
  def initialize(@name : String, @priority : Int32)
  end
  
  def to_s(io : IO) : Nil
    io << "[P#{priority}] #{name}"
  end
end

class TaskScheduler
  def initialize
    @high_priority = Deque(Task).new
    @normal_priority = Deque(Task).new
    @completed = [] of Task
  end
  
  def add_task(name : String, priority : Int32 = 1)
    task = Task.new(name, priority)
    if priority >= 5
      @high_priority.push_back(task)
    else
      @normal_priority.push_back(task)
    end
    puts "Added: #{task}"
  end
  
  def process_next : Task?
    # Process high priority first
    task = if !@high_priority.empty?
      @high_priority.pop_front
    elsif !@normal_priority.empty?
      @normal_priority.pop_front
    else
      nil
    end
    
    if task
      @completed << task
      puts "Processed: #{task}"
    end
    
    task
  end
  
  def pending_count : Int32
    @high_priority.size + @normal_priority.size
  end
  
  def stats
    puts "High priority queue: #{@high_priority.size}"
    puts "Normal priority queue: #{@normal_priority.size}"
    puts "Completed: #{@completed.size}"
  end
end

scheduler = TaskScheduler.new
scheduler.add_task("Check emails", 2)
scheduler.add_task("Fix critical bug", 9)
scheduler.add_task("Write docs", 1)
scheduler.add_task("Deploy hotfix", 8)
scheduler.add_task("Team meeting", 3)

puts "\nProcessing:"
5.times { scheduler.process_next }

puts "\nStats:"
scheduler.stats
```

---

## สรุป

Deque ใน Crystal:

| Operation | Complexity | Method |
|-----------|-----------|--------|
| push_back | O(1) | `push_back`, `<<` |
| push_front | O(1) | `push_front`, `unshift` |
| pop_back | O(1) | `pop_back`, `pop` |
| pop_front | O(1) | `pop_front`, `shift` |
| peek first | O(1) | `first`, `first?` |
| peek last | O(1) | `last`, `last?` |
| access by index | O(1) | `[]` |
| insert middle | O(n) | `insert` |

**เมื่อใช้ Deque แทน Array:**
- ต้องการ insert/delete ที่ทั้งสองฝั่งบ่อยๆ
- Implement Queue หรือ Stack
- Sliding window algorithms
- Recent items cache
- BFS algorithm

**Deque vs Array:**
- Deque: O(1) push_front/pop_front
- Array: O(n) unshift/shift
- ทั้งคู่: O(1) push_back/pop_back

---

*ต่อไป: Part 52 - Iterators*
