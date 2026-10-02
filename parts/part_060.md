# Part 60: Linked Lists และ Custom Collections ใน Crystal

## บทนำ

ในบทนี้เราจะเรียนรู้การสร้าง custom data structures ใน Crystal โดยเริ่มจาก Linked List แบบ manual จากนั้นใช้ `Enumerable` module เพื่อให้ collection ของเราทำงานเหมือน built-in collections

---

## 60.1 Crystal's Standard Deque

```crystal
require "deque"

# Deque = Double-Ended Queue (built-in)
# ใช้แทน LinkedList ได้ในหลายกรณี
deque = Deque(Int32).new

# push/pop front and back
deque.push(1)
deque.push(2)
deque.push(3)
deque.unshift(0)  # add to front

puts deque.inspect  # => Deque{0, 1, 2, 3}
puts deque.first    # => 0
puts deque.last     # => 3
puts deque.shift    # => 0 (remove from front)
puts deque.pop      # => 3 (remove from back)
puts deque.inspect  # => Deque{1, 2}

# Deque supports all Array methods
deque.push(5)
deque.push(3)
puts deque.sort.inspect  # => [1, 2, 3, 5]
puts deque.map { |n| n * 2 }.inspect  # => [2, 4, 10, 6]
```

---

## 60.2 Node Class

```crystal
# Building block ของ Linked List
class Node(T)
  property value : T
  property next_node : Node(T)?

  def initialize(@value : T, @next_node : Node(T)? = nil)
  end

  def to_s(io : IO)
    io << @value
    if nxt = @next_node
      io << " -> "
      nxt.to_s(io)
    end
  end
end

# สร้าง nodes manually
n3 = Node(Int32).new(30)
n2 = Node(Int32).new(20, n3)
n1 = Node(Int32).new(10, n2)

puts n1.to_s  # => 10 -> 20 -> 30
puts n1.value  # => 10
puts n1.next_node.try(&.value)  # => 20
```

---

## 60.3 Singly Linked List

```crystal
class LinkedList(T)
  include Enumerable(T)

  @head : Node(T)?
  @size : Int32 = 0

  def initialize
    @head = nil
  end

  # เพิ่มที่ต้น (O(1))
  def prepend(value : T)
    @head = Node(T).new(value, @head)
    @size += 1
    self
  end

  # เพิ่มที่ท้าย (O(n))
  def append(value : T)
    new_node = Node(T).new(value)
    if @head.nil?
      @head = new_node
    else
      current = @head
      while current && current.next_node
        current = current.next_node
      end
      current.not_nil!.next_node = new_node
    end
    @size += 1
    self
  end

  # ลบค่า (O(n))
  def delete(value : T) : Bool
    return false if @head.nil?

    if @head.not_nil!.value == value
      @head = @head.not_nil!.next_node
      @size -= 1
      return true
    end

    current = @head
    while current && current.next_node
      if current.next_node.not_nil!.value == value
        current.next_node = current.next_node.not_nil!.next_node
        @size -= 1
        return true
      end
      current = current.next_node
    end
    false
  end

  # Enumerable ต้องการ each
  def each(& : T ->)
    current = @head
    while node = current
      yield node.value
      current = node.next_node
    end
  end

  def size : Int32
    @size
  end

  def empty? : Bool
    @size == 0
  end

  def to_s(io : IO)
    io << "["
    first = true
    each do |val|
      io << ", " unless first
      io << val
      first = false
    end
    io << "]"
  end
end

# ใช้งาน
list = LinkedList(Int32).new
list.append(1).append(2).append(3)
list.prepend(0)

puts list.to_s    # => [0, 1, 2, 3]
puts list.size    # => 4

# เนื่องจาก include Enumerable - ได้ methods เหล่านี้ฟรี!
puts list.map { |n| n * 2 }.inspect        # => [0, 2, 4, 6]
puts list.select { |n| n.even? }.inspect   # => [0, 2]
puts list.sum                               # => 6
puts list.min                               # => 0
puts list.max                               # => 3
puts list.includes?(2)                      # => true
puts list.to_a.inspect                      # => [0, 1, 2, 3]
```

---

## 60.4 Doubly Linked List

```crystal
class DNode(T)
  property value : T
  property next_node : DNode(T)?
  property prev_node : DNode(T)?

  def initialize(@value : T)
    @next_node = nil
    @prev_node = nil
  end
end

class DoublyLinkedList(T)
  include Enumerable(T)

  @head : DNode(T)?
  @tail : DNode(T)?
  @size : Int32 = 0

  def push_back(value : T)
    node = DNode(T).new(value)
    if @tail.nil?
      @head = @tail = node
    else
      node.prev_node = @tail
      @tail.not_nil!.next_node = node
      @tail = node
    end
    @size += 1
    self
  end

  def push_front(value : T)
    node = DNode(T).new(value)
    if @head.nil?
      @head = @tail = node
    else
      node.next_node = @head
      @head.not_nil!.prev_node = node
      @head = node
    end
    @size += 1
    self
  end

  def pop_back : T?
    return nil if @tail.nil?
    val = @tail.not_nil!.value
    @tail = @tail.not_nil!.prev_node
    if @tail
      @tail.not_nil!.next_node = nil
    else
      @head = nil
    end
    @size -= 1
    val
  end

  def pop_front : T?
    return nil if @head.nil?
    val = @head.not_nil!.value
    @head = @head.not_nil!.next_node
    if @head
      @head.not_nil!.prev_node = nil
    else
      @tail = nil
    end
    @size -= 1
    val
  end

  def each(& : T ->)
    current = @head
    while node = current
      yield node.value
      current = node.next_node
    end
  end

  def each_reverse(& : T ->)
    current = @tail
    while node = current
      yield node.value
      current = node.prev_node
    end
  end

  def size : Int32
    @size
  end
end

# ใช้งาน
dll = DoublyLinkedList(String).new
dll.push_back("b")
dll.push_back("c")
dll.push_front("a")

puts "Forward: #{dll.to_a.inspect}"   # => ["a", "b", "c"]

reverse = [] of String
dll.each_reverse { |v| reverse << v }
puts "Reverse: #{reverse.inspect}"    # => ["c", "b", "a"]

puts dll.pop_front   # => "a"
puts dll.pop_back    # => "c"
puts dll.to_a.inspect  # => ["b"]
```

---

## 60.5 Custom Collection ที่ใช้ Enumerable

```crystal
# Custom sorted collection
class SortedArray(T)
  include Enumerable(T)
  include Comparable(SortedArray(T))

  def initialize
    @data = [] of T
  end

  def insert(value : T)
    # Binary search for insertion point
    lo, hi = 0, @data.size
    while lo < hi
      mid = (lo + hi) // 2
      if @data[mid] < value
        lo = mid + 1
      else
        hi = mid
      end
    end
    @data.insert(lo, value)
    self
  end

  def delete(value : T) : Bool
    idx = @data.index(value)
    if idx
      @data.delete_at(idx)
      true
    else
      false
    end
  end

  def includes?(value : T) : Bool
    # Binary search O(log n)
    lo, hi = 0, @data.size - 1
    while lo <= hi
      mid = (lo + hi) // 2
      cmp = @data[mid] <=> value
      return true if cmp == 0
      if cmp < 0
        lo = mid + 1
      else
        hi = mid - 1
      end
    end
    false
  end

  def each(& : T ->)
    @data.each { |v| yield v }
  end

  def size : Int32
    @data.size
  end

  def <=>(other : SortedArray(T)) : Int32
    size <=> other.size
  end

  def to_s(io : IO)
    io << "SortedArray"
    @data.to_s(io)
  end
end

# ใช้งาน
sorted = SortedArray(Int32).new
[5, 2, 8, 1, 9, 3].each { |n| sorted.insert(n) }

puts sorted.to_s             # => SortedArray[1, 2, 3, 5, 8, 9]
puts sorted.includes?(5)     # => true
puts sorted.includes?(4)     # => false
puts sorted.min              # => 1
puts sorted.max              # => 9
puts sorted.select(&.even?).inspect  # => [2, 8]
sorted.delete(5)
puts sorted.to_s             # => SortedArray[1, 2, 3, 8, 9]
```

---

## 60.6 Stack Implementation

```crystal
class Stack(T)
  include Enumerable(T)

  def initialize
    @data = [] of T
  end

  def push(value : T)
    @data << value
    self
  end

  def pop : T
    raise "Stack underflow" if empty?
    @data.pop
  end

  def peek : T
    raise "Stack is empty" if empty?
    @data.last
  end

  def each(& : T ->)
    # Iterate from top to bottom
    @data.reverse_each { |v| yield v }
  end

  def size : Int32
    @data.size
  end

  def empty? : Bool
    @data.empty?
  end

  def to_s(io : IO)
    io << "Stack(top) "
    @data.reverse.each { |v| io << v << " " }
    io << "(bottom)"
  end
end

# ใช้งาน: Balanced parentheses checker
def balanced_parens?(str : String) : Bool
  stack = Stack(Char).new
  pairs = {'(' => ')', '[' => ']', '{' => '}'}

  str.each_char do |ch|
    if pairs.has_key?(ch)
      stack.push(ch)
    elsif pairs.values.includes?(ch)
      return false if stack.empty?
      open = stack.pop
      return false unless pairs[open] == ch
    end
  end

  stack.empty?
end

["(())", "({[]})", "([)]", "((()"].each do |s|
  puts "#{s}: #{balanced_parens?(s)}"
end
# (()): true
# ({[]}): true
# ([)]: false
# ((((): false
```

---

## 60.7 Queue Implementation

```crystal
class Queue(T)
  include Enumerable(T)

  def initialize
    @data = Deque(T).new
  end

  def enqueue(value : T)
    @data.push(value)
    self
  end

  def dequeue : T
    raise "Queue is empty" if empty?
    @data.shift
  end

  def front : T
    raise "Queue is empty" if empty?
    @data.first
  end

  def each(& : T ->)
    @data.each { |v| yield v }
  end

  def size : Int32
    @data.size
  end

  def empty? : Bool
    @data.empty?
  end
end

# ใช้งาน: Task queue
task_queue = Queue(String).new
task_queue.enqueue("Send email")
task_queue.enqueue("Process payment")
task_queue.enqueue("Update inventory")

puts "Queue size: #{task_queue.size}"  # => 3
puts "Next task: #{task_queue.front}"  # => Send email
puts "Processing: #{task_queue.dequeue}"  # => Send email
puts "Queue size: #{task_queue.size}"  # => 2
```

---

## 60.8 Priority Queue

```crystal
# Min-heap based priority queue
class PriorityQueue(T)
  include Enumerable(T)

  def initialize
    @heap = [] of T
  end

  def push(value : T)
    @heap << value
    sift_up(@heap.size - 1)
    self
  end

  def pop : T
    raise "Priority queue is empty" if empty?
    result = @heap[0]
    last = @heap.pop
    unless @heap.empty?
      @heap[0] = last
      sift_down(0)
    end
    result
  end

  def peek : T
    raise "Priority queue is empty" if empty?
    @heap[0]
  end

  def each(& : T ->)
    @heap.sort.each { |v| yield v }
  end

  def size : Int32
    @heap.size
  end

  def empty? : Bool
    @heap.empty?
  end

  private def sift_up(i : Int32)
    while i > 0
      parent = (i - 1) // 2
      break if @heap[parent] <= @heap[i]
      @heap[i], @heap[parent] = @heap[parent], @heap[i]
      i = parent
    end
  end

  private def sift_down(i : Int32)
    n = @heap.size
    loop do
      smallest = i
      left = 2 * i + 1
      right = 2 * i + 2
      smallest = left  if left < n  && @heap[left]  < @heap[smallest]
      smallest = right if right < n && @heap[right] < @heap[smallest]
      break if smallest == i
      @heap[i], @heap[smallest] = @heap[smallest], @heap[i]
      i = smallest
    end
  end
end

pq = PriorityQueue(Int32).new
[5, 2, 8, 1, 9, 3].each { |n| pq.push(n) }

print "Sorted: "
until pq.empty?
  print "#{pq.pop} "
end
puts  # => 1 2 3 5 8 9
```

---

## 60.9 Circular Buffer

```crystal
# Fixed-size ring buffer
class CircularBuffer(T)
  include Enumerable(T)

  def initialize(@capacity : Int32)
    @data = Array(T?).new(@capacity, nil)
    @head = 0
    @tail = 0
    @size = 0
  end

  def push(value : T) : Bool
    if full?
      # Overwrite oldest element
      @data[@tail] = value
      @head = (@head + 1) % @capacity
    else
      @data[@tail] = value
      @tail = (@tail + 1) % @capacity
      @size += 1
    end
    true
  end

  def pop : T?
    return nil if empty?
    value = @data[@head]
    @data[@head] = nil
    @head = (@head + 1) % @capacity
    @size -= 1
    value
  end

  def each(& : T ->)
    @size.times do |i|
      idx = (@head + i) % @capacity
      val = @data[idx]
      yield val.not_nil! if val
    end
  end

  def size : Int32
    @size
  end

  def full? : Bool
    @size == @capacity
  end

  def empty? : Bool
    @size == 0
  end
end

# ใช้งาน: เก็บ 5 records ล่าสุด
log = CircularBuffer(String).new(5)
["A", "B", "C", "D", "E", "F", "G"].each do |item|
  log.push(item)
end

print "Last 5: "
log.each { |item| print "#{item} " }
puts  # => C D E F G (A, B ถูก overwrite)
```

---

## 60.10 ตัวอย่างใช้งานจริง: LRU Cache

```crystal
# Least Recently Used Cache ด้วย Doubly Linked List + Hash
class LRUCache(K, V)
  class CacheNode(K, V)
    property key : K
    property value : V
    property prev_node : CacheNode(K, V)?
    property next_node : CacheNode(K, V)?

    def initialize(@key : K, @value : V)
    end
  end

  def initialize(@capacity : Int32)
    @map = {} of K => CacheNode(K, V)
    @head = CacheNode(K, V).new(uninitialized K, uninitialized V)  # dummy head
    @tail = CacheNode(K, V).new(uninitialized K, uninitialized V)  # dummy tail
    @head.next_node = @tail
    @tail.prev_node = @head
  end

  def get(key : K) : V?
    node = @map[key]?
    return nil unless node
    move_to_front(node)
    node.value
  end

  def put(key : K, value : V)
    if node = @map[key]?
      node.value = value
      move_to_front(node)
    else
      if @map.size >= @capacity
        # Remove LRU (before tail)
        lru = @tail.prev_node.not_nil!
        remove_node(lru)
        @map.delete(lru.key)
      end
      new_node = CacheNode(K, V).new(key, value)
      @map[key] = new_node
      add_to_front(new_node)
    end
  end

  def size : Int32
    @map.size
  end

  private def remove_node(node : CacheNode(K, V))
    node.prev_node.not_nil!.next_node = node.next_node
    node.next_node.not_nil!.prev_node = node.prev_node
  end

  private def add_to_front(node : CacheNode(K, V))
    node.next_node = @head.next_node
    node.prev_node = @head
    @head.next_node.not_nil!.prev_node = node
    @head.next_node = node
  end

  private def move_to_front(node : CacheNode(K, V))
    remove_node(node)
    add_to_front(node)
  end
end

# ใช้งาน
cache = LRUCache(String, Int32).new(3)
cache.put("a", 1)
cache.put("b", 2)
cache.put("c", 3)

puts cache.get("a")  # => 1 (a is now most recent)
cache.put("d", 4)    # evicts "b" (least recently used)

puts cache.get("b")  # => nil (evicted)
puts cache.get("c")  # => 3
puts cache.get("d")  # => 4
puts "Cache size: #{cache.size}"  # => 3
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Reverse Linked List

```crystal
def reverse_list(list : LinkedList(Int32)) : LinkedList(Int32)
  result = LinkedList(Int32).new
  list.each { |v| result.prepend(v) }
  result
end

original = LinkedList(Int32).new
[1, 2, 3, 4, 5].each { |n| original.append(n) }
reversed = reverse_list(original)

puts "Original: #{original.to_s}"   # => [1, 2, 3, 4, 5]
puts "Reversed: #{reversed.to_s}"   # => [5, 4, 3, 2, 1]
```

### แบบฝึกหัดที่ 2: Merge Two Sorted Lists

```crystal
def merge_sorted(a : LinkedList(Int32), b : LinkedList(Int32)) : LinkedList(Int32)
  result = LinkedList(Int32).new
  arr_a = a.to_a
  arr_b = b.to_a
  merged = (arr_a + arr_b).sort
  merged.each { |v| result.append(v) }
  result
end

list_a = LinkedList(Int32).new
[1, 3, 5].each { |n| list_a.append(n) }
list_b = LinkedList(Int32).new
[2, 4, 6].each { |n| list_b.append(n) }

merged = merge_sorted(list_a, list_b)
puts "Merged: #{merged.to_s}"  # => [1, 2, 3, 4, 5, 6]
```

### แบบฝึกหัดที่ 3: Custom Range Collection

```crystal
class NumberRange
  include Enumerable(Int32)

  def initialize(@start : Int32, @stop : Int32, @step : Int32 = 1)
  end

  def each(& : Int32 ->)
    n = @start
    while n <= @stop
      yield n
      n += @step
    end
  end

  def size : Int32
    [@stop - @start + 1, 0].max // @step
  end
end

evens = NumberRange.new(0, 20, 2)
puts evens.to_a.inspect   # => [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]
puts evens.sum            # => 110
puts evens.select { |n| n % 6 == 0 }.inspect  # => [0, 6, 12, 18]
```

---

## สรุป

Custom Collections ใน Crystal:

| Collection | Use Case |
|-----------|---------|
| `LinkedList(T)` | Fast prepend, iterator |
| `DoublyLinkedList(T)` | Fast insert/delete anywhere |
| `SortedArray(T)` | Always sorted O(log n) search |
| `Stack(T)` | LIFO - undo, parenthesis matching |
| `Queue(T)` | FIFO - task processing |
| `PriorityQueue(T)` | Min/max heap |
| `CircularBuffer(T)` | Fixed-size ring buffer |
| `LRUCache(K,V)` | Cached with eviction |

**หลักการสร้าง Custom Collection:**
1. สร้าง Node class ถ้าต้องการ linked structure
2. `include Enumerable(T)` เพื่อรับ methods ฟรี
3. implement `each` method
4. implement `size` method
5. เพิ่ม methods เฉพาะที่ต้องการ

---

*จบหมวด Collections - ต่อไป: Part 61 - Exceptions*
