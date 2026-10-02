# Part 45: Comparable และ Enumerable ใน Crystal

## บทนำ

Crystal มี modules พิเศษสองตัวที่ให้ functionality มากมายเพียงแค่ implement method เดียว:
- **Comparable(T)**: ให้ operators เปรียบเทียบ (<, >, <=, >=, ==) เพียงแค่ implement `<=>`
- **Enumerable(T)**: ให้ methods ทั้งหมดของ collection เพียงแค่ implement `each`

---

## 45.1 Comparable Module

### พื้นฐาน

```crystal
class Product
  include Comparable(Product)
  
  getter name : String
  getter price : Float64
  
  def initialize(@name : String, @price : Float64)
  end
  
  # เพียงแค่ implement <=> เดียว
  def <=>(other : Product) : Int32
    @price <=> other.price
  end
  
  def to_s(io : IO) : Nil
    io << "#{@name} ($#{@price})"
  end
end

p1 = Product.new("Apple", 1.99)
p2 = Product.new("Banana", 0.99)
p3 = Product.new("Cherry", 3.49)

# ได้ operators ทั้งหมดจาก <=>
puts p1 > p2    # => true
puts p2 < p3    # => true
puts p1 >= p1   # => true
puts p1 == p3   # => false
puts p2 <= p1   # => true

products = [p3, p1, p2]
puts products.sort.map(&.to_s).join(", ")
# => Banana ($0.99), Apple ($1.99), Cherry ($3.49)

puts products.min   # => Banana ($0.99)
puts products.max   # => Cherry ($3.49)
```

### def <=> รายละเอียด

```crystal
# <=> คืน:
# -1 (หรือค่าลบ) เมื่อ self < other
#  0 เมื่อ self == other
#  1 (หรือค่าบวก) เมื่อ self > other
# nil เมื่อไม่สามารถเปรียบเทียบได้

class Version
  include Comparable(Version)
  
  def initialize(@major : Int32, @minor : Int32, @patch : Int32)
  end
  
  def <=>(other : Version) : Int32
    # เปรียบเทียบ major ก่อน
    cmp = @major <=> other.@major
    return cmp unless cmp == 0
    
    # ถ้า major เท่ากัน เปรียบเทียบ minor
    cmp = @minor <=> other.@minor
    return cmp unless cmp == 0
    
    # ถ้า minor เท่ากัน เปรียบเทียบ patch
    @patch <=> other.@patch
  end
  
  def to_s(io : IO) : Nil
    io << "#{@major}.#{@minor}.#{@patch}"
  end
end

v1 = Version.new(1, 0, 0)
v2 = Version.new(1, 2, 0)
v3 = Version.new(2, 0, 0)
v4 = Version.new(1, 2, 3)

versions = [v3, v1, v4, v2]
puts versions.sort.map(&.to_s).join(" -> ")
# => 1.0.0 -> 1.2.0 -> 1.2.3 -> 2.0.0

puts v1 < v2   # => true
puts v3 > v4   # => true
```

---

## 45.2 Sorting ด้วย Comparable

```crystal
class Student
  include Comparable(Student)
  
  getter name : String
  getter gpa : Float64
  getter age : Int32
  
  def initialize(@name : String, @gpa : Float64, @age : Int32)
  end
  
  # เรียงตาม GPA (มากไปน้อย) แล้วตาม name
  def <=>(other : Student) : Int32
    cmp = other.gpa <=> @gpa  # กลับ order เพื่อ descending
    cmp != 0 ? cmp : @name <=> other.name
  end
  
  def to_s(io : IO) : Nil
    io << "#{@name} (GPA: #{@gpa})"
  end
end

students = [
  Student.new("Charlie", 3.5, 20),
  Student.new("Alice", 3.8, 21),
  Student.new("Bob", 3.5, 22),
  Student.new("Dave", 4.0, 19)
]

puts "Ranked students:"
students.sort.each_with_index do |s, i|
  puts "#{i+1}. #{s}"
end
# 1. Dave (GPA: 4.0)
# 2. Alice (GPA: 3.8)
# 3. Bob (GPA: 3.5)    <- B ก่อน C
# 4. Charlie (GPA: 3.5)
```

---

## 45.3 min, max, clamp

```crystal
class Temperature
  include Comparable(Temperature)
  
  getter celsius : Float64
  
  def initialize(@celsius : Float64)
  end
  
  def self.from_fahrenheit(f : Float64) : Temperature
    new((f - 32) * 5 / 9)
  end
  
  def <=>(other : Temperature) : Int32
    @celsius <=> other.celsius
  end
  
  def to_fahrenheit : Float64
    @celsius * 9 / 5 + 32
  end
  
  def to_s(io : IO) : Nil
    io << "#{@celsius.round(1)}°C"
  end
end

temps = [
  Temperature.new(37.0),
  Temperature.new(100.0),
  Temperature.new(-10.0),
  Temperature.new(22.5)
]

puts temps.min  # => -10.0°C
puts temps.max  # => 100.0°C

# clamp จาก Comparable
body_temp = Temperature.new(37.0)
normal_range_low = Temperature.new(36.0)
normal_range_high = Temperature.new(37.5)

# clamp ให้ค่าอยู่ใน range
too_hot = Temperature.new(41.0)
clamped = too_hot.clamp(normal_range_low, normal_range_high)
puts clamped  # => 37.5°C
```

---

## 45.4 Enumerable Module

```crystal
class NumberRange
  include Enumerable(Int32)
  
  def initialize(@start : Int32, @stop : Int32, @step : Int32 = 1)
  end
  
  # เพียงแค่ implement each เดียว!
  def each : Nil
    current = @start
    while current <= @stop
      yield current
      current += @step
    end
  end
  
  # ได้ methods ทั้งหมดมาฟรี!
end

evens = NumberRange.new(0, 20, 2)

puts evens.to_a.inspect
# => [0, 2, 4, 6, 8, 10, 12, 14, 16, 18, 20]

puts evens.sum          # => 110
puts evens.count        # => 11
puts evens.min          # => 0
puts evens.max          # => 20
puts evens.first        # => 0
puts evens.first(3).inspect   # => [0, 2, 4]
puts evens.include?(8)  # => true
puts evens.include?(7)  # => false

puts evens.map { |n| n * n }.first(5).inspect  # => [0, 4, 16, 36, 64]
puts evens.select { |n| n > 10 }.to_a.inspect  # => [12, 14, 16, 18, 20]
puts evens.reject { |n| n > 10 }.to_a.inspect  # => [0, 2, 4, 6, 8, 10]
```

---

## 45.5 Enumerable Methods ทั้งหมด

```crystal
class WordList
  include Enumerable(String)
  
  def initialize(@words : Array(String))
  end
  
  def each : Nil
    @words.each { |w| yield w }
  end
end

words = WordList.new(["apple", "banana", "cherry", "date", "elderberry", "fig"])

# Filtering
puts words.select { |w| w.size > 5 }.to_a.inspect
# => ["banana", "cherry", "elderberry"]

puts words.reject { |w| w.includes?("a") }.to_a.inspect
# => ["cherry", "elderberry", "fig"]

# Transformation
puts words.map(&.upcase).to_a.inspect
# => ["APPLE", "BANANA", "CHERRY", "DATE", "ELDERBERRY", "FIG"]

puts words.flat_map { |w| [w, w.size] }.to_a.inspect
# => ["apple", 5, "banana", 6, "cherry", 6, "date", 4, "elderberry", 10, "fig", 3]

# Aggregation
puts words.reduce("") { |acc, w| acc.empty? ? w : "#{acc}, #{w}" }
# => "apple, banana, cherry, date, elderberry, fig"

puts words.count { |w| w.size > 4 }  # => 5

# Finding
puts words.find { |w| w.starts_with?("c") }  # => "cherry"
puts words.any? { |w| w.size > 8 }   # => true
puts words.all? { |w| w.size > 2 }   # => true
puts words.none? { |w| w.size > 15 } # => true

# Grouping
grouped = words.group_by { |w| w.size }
grouped.each { |size, ws| puts "Length #{size}: #{ws.inspect}" }

# Min/Max
puts words.min_by { |w| w.size }   # => "fig"
puts words.max_by { |w| w.size }   # => "elderberry"

# Sorting
puts words.sort_by(&.size).to_a.inspect
# => ["fig", "date", "apple", "banana", "cherry", "elderberry"]

# each_with_index
words.each_with_index do |w, i|
  puts "#{i}: #{w}"
end

# each_with_object
result = words.each_with_object({} of Int32 => Array(String)) do |word, hash|
  size = word.size
  hash[size] ||= [] of String
  hash[size] << word
end
puts result.inspect
```

---

## 45.6 Implementing Custom Enumerable

### Linked List ที่เป็น Enumerable

```crystal
class LinkedListNode(T)
  property value : T
  property next_node : LinkedListNode(T)?
  
  def initialize(@value : T, @next_node : LinkedListNode(T)? = nil)
  end
end

class LinkedList(T)
  include Enumerable(T)
  
  @head : LinkedListNode(T)?
  @size : Int32 = 0
  
  def initialize
    @head = nil
  end
  
  def push(value : T)
    @head = LinkedListNode(T).new(value, @head)
    @size += 1
    self
  end
  
  def pop : T?
    return nil if @head.nil?
    value = @head.not_nil!.value
    @head = @head.not_nil!.next_node
    @size -= 1
    value
  end
  
  # implement each สำหรับ Enumerable
  def each : Nil
    current = @head
    while current
      yield current.value
      current = current.next_node
    end
  end
  
  def size : Int32
    @size
  end
  
  def to_s(io : IO) : Nil
    io << "["
    io << to_a.join(", ")
    io << "]"
  end
end

list = LinkedList(Int32).new
list.push(3).push(2).push(1)

puts list.to_a.inspect   # => [1, 2, 3]
puts list.sum            # => 6
puts list.map { |n| n * 2 }.to_a.inspect  # => [2, 4, 6]
puts list.select { |n| n > 1 }.to_a.inspect  # => [2, 3]
puts list.include?(2)    # => true
puts list.count          # => 3
```

### Tree ที่เป็น Enumerable

```crystal
class TreeNode(T)
  getter value : T
  property left : TreeNode(T)?
  property right : TreeNode(T)?
  
  def initialize(@value : T)
    @left = nil
    @right = nil
  end
end

class BinaryTree(T)
  include Enumerable(T)
  include Comparable(BinaryTree(T))
  
  @root : TreeNode(T)?
  
  def initialize
    @root = nil
  end
  
  def insert(value : T)
    @root = insert_node(@root, value)
    self
  end
  
  # In-order traversal
  def each : Nil
    traverse_inorder(@root) { |v| yield v }
  end
  
  def <=>(other : BinaryTree(T)) : Int32
    to_a <=> other.to_a
  end
  
  private def insert_node(node : TreeNode(T)?, value : T) : TreeNode(T)
    return TreeNode(T).new(value) if node.nil?
    
    if value < node.value
      node.left = insert_node(node.left, value)
    elsif value > node.value
      node.right = insert_node(node.right, value)
    end
    
    node
  end
  
  private def traverse_inorder(node : TreeNode(T)?, &block : T ->)
    return if node.nil?
    traverse_inorder(node.left, &block)
    block.call(node.value)
    traverse_inorder(node.right, &block)
  end
end

tree = BinaryTree(Int32).new
[5, 3, 7, 1, 4, 6, 8].each { |n| tree.insert(n) }

puts tree.to_a.inspect  # => [1, 3, 4, 5, 6, 7, 8] (sorted!)
puts tree.sum           # => 34
puts tree.min           # => 1
puts tree.max           # => 8
puts tree.include?(4)   # => true
puts tree.select { |n| n > 5 }.to_a.inspect  # => [6, 7, 8]
```

---

## 45.7 Combining Comparable และ Enumerable

```crystal
class PriorityQueue(T)
  include Enumerable(T)
  
  def initialize
    @items = [] of {Int32, T}
    @counter = 0
  end
  
  def enqueue(item : T, priority : Int32)
    @items << {priority, item}
    @items.sort_by! { |p, _| -p }  # Sort by priority descending
    @counter += 1
    self
  end
  
  def dequeue : T?
    return nil if @items.empty?
    _, item = @items.shift
    item
  end
  
  def peek : T?
    @items.first?.try { |_, item| item }
  end
  
  def each : Nil
    @items.each { |_, item| yield item }
  end
  
  def size : Int32
    @items.size
  end
  
  def empty? : Bool
    @items.empty?
  end
end

pq = PriorityQueue(String).new
pq.enqueue("Low priority task", 1)
pq.enqueue("High priority task", 10)
pq.enqueue("Medium priority task", 5)
pq.enqueue("Critical task", 100)

puts "Queue contents (by priority):"
pq.each { |item| puts "  - #{item}" }

puts "\nDequeuing:"
while !pq.empty?
  puts pq.dequeue
end
```

---

## 45.8 Lazy Enumerable

```crystal
class InfiniteSequence
  include Enumerable(Int32)
  
  def initialize(@formula : Int32 -> Int32)
    @limit = Int32::MAX
  end
  
  def each : Nil
    n = 0
    @limit.times do
      yield @formula.call(n)
      n += 1
    end
  end
  
  def take(n : Int32) : Array(Int32)
    result = [] of Int32
    each do |val|
      result << val
      break if result.size >= n
    end
    result
  end
end

# Fibonacci
fibonacci = InfiniteSequence.new do |n|
  if n <= 1
    n
  else
    a, b = 0, 1
    (n - 1).times { a, b = b, a + b }
    b
  end
end

puts fibonacci.take(10).inspect
# => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# Squares
squares = InfiniteSequence.new { |n| n * n }
puts squares.take(8).inspect
# => [0, 1, 4, 9, 16, 25, 36, 49]
```

---

## 45.9 ตัวอย่างในชีวิตจริง: TaskQueue

```crystal
class Task
  include Comparable(Task)
  
  getter id : Int32
  getter name : String
  getter priority : Int32
  getter created_at : Time
  getter? completed : Bool
  
  def initialize(@id : Int32, @name : String, @priority : Int32)
    @created_at = Time.local
    @completed = false
  end
  
  def complete!
    @completed = true
  end
  
  def <=>(other : Task) : Int32
    # เรียงตาม priority (มากไปน้อย)
    cmp = other.priority <=> @priority
    # ถ้าเท่ากัน เรียงตาม created_at
    cmp != 0 ? cmp : @created_at <=> other.created_at
  end
  
  def to_s(io : IO) : Nil
    status = completed? ? "[✓]" : "[ ]"
    io << "#{status} [P#{priority}] #{name}"
  end
end

class TaskQueue
  include Enumerable(Task)
  
  def initialize
    @tasks = [] of Task
    @next_id = 1
  end
  
  def add(name : String, priority : Int32 = 1) : Task
    task = Task.new(@next_id, name, priority)
    @next_id += 1
    @tasks << task
    @tasks.sort!
    task
  end
  
  def complete(id : Int32) : Bool
    task = @tasks.find { |t| t.id == id }
    return false if task.nil?
    task.complete!
    true
  end
  
  def next_task : Task?
    @tasks.find { |t| !t.completed? }
  end
  
  def each : Nil
    @tasks.each { |t| yield t }
  end
  
  def pending : Array(Task)
    select { |t| !t.completed? }.to_a
  end
  
  def completed : Array(Task)
    select { |t| t.completed? }.to_a
  end
  
  def stats : String
    total = count
    done = completed.size
    pending_count = pending.size
    "Total: #{total}, Done: #{done}, Pending: #{pending_count}"
  end
end

queue = TaskQueue.new
t1 = queue.add("Fix critical bug", 10)
t2 = queue.add("Write documentation", 2)
t3 = queue.add("Code review", 5)
t4 = queue.add("Deploy to production", 8)
t5 = queue.add("Update tests", 3)

puts "All tasks:"
queue.each { |t| puts "  #{t}" }

puts "\nNext task: #{queue.next_task}"

queue.complete(t1.id)
queue.complete(t4.id)

puts "\nAfter completing critical tasks:"
puts "Stats: #{queue.stats}"
puts "\nPending tasks:"
queue.pending.each { |t| puts "  #{t}" }
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Implement Comparable สำหรับ Card

```crystal
class Card
  include Comparable(Card)
  
  SUITS = ["♣", "♦", "♥", "♠"]
  RANKS = ["2", "3", "4", "5", "6", "7", "8", "9", "10", "J", "Q", "K", "A"]
  
  getter rank : Int32  # 0-12
  getter suit : Int32  # 0-3
  
  def initialize(@rank : Int32, @suit : Int32)
  end
  
  def <=>(other : Card) : Int32
    cmp = @rank <=> other.rank
    cmp != 0 ? cmp : @suit <=> other.suit
  end
  
  def to_s(io : IO) : Nil
    io << "#{RANKS[@rank]}#{SUITS[@suit]}"
  end
end

# สร้าง deck
deck = [] of Card
4.times do |suit|
  13.times do |rank|
    deck << Card.new(rank, suit)
  end
end

deck.shuffle!

puts "Random 5 cards: #{deck.first(5).map(&.to_s).join(", ")}"
puts "Best card: #{deck.max}"
puts "Worst card: #{deck.min}"

sorted_hand = deck.first(7).sort
puts "Sorted hand: #{sorted_hand.map(&.to_s).join(", ")}"
```

### แบบฝึกหัดที่ 2: Custom Collection

```crystal
class NumberSet
  include Enumerable(Int32)
  
  def initialize
    @data = Set(Int32).new
  end
  
  def add(*numbers : Int32) : self
    numbers.each { |n| @data.add(n) }
    self
  end
  
  def each : Nil
    @data.each { |n| yield n }
  end
  
  def size : Int32
    @data.size
  end
  
  # สามารถใช้ Enumerable methods ทั้งหมดได้
  def statistics : String
    sorted = to_a.sort
    avg = sum.to_f / count
    "Count: #{count}, Sum: #{sum}, Avg: #{avg.round(2)}, Min: #{min}, Max: #{max}"
  end
end

require "set"

ns = NumberSet.new
ns.add(5, 3, 8, 1, 9, 2, 7, 4, 6)

puts ns.statistics
puts ns.select { |n| n.even? }.to_a.sort.inspect
puts ns.map { |n| n * 2 }.to_a.sort.inspect
```

---

## สรุป

### Comparable(T)
- Include `Comparable(T)` ใน class
- Implement `def <=>(other : T) : Int32`
- ได้ `<`, `>`, `<=`, `>=` มาฟรี
- ได้ `.min`, `.max`, `.sort`, `.clamp` บน Array

### Enumerable(T)
- Include `Enumerable(T)` ใน class
- Implement `def each : Nil` ที่ yield T
- ได้ methods มากมาย:
  - `map`, `select`, `reject`, `flat_map`
  - `find`, `any?`, `all?`, `none?`
  - `count`, `sum`, `min`, `max`
  - `sort`, `sort_by`, `min_by`, `max_by`
  - `group_by`, `chunk`, `each_with_index`
  - `reduce`, `inject`, `each_with_object`
  - `first`, `last`, `include?`
  - `to_a`, `zip`
  - และอีกมากมาย!

สองสิ่งนี้เป็น "power multipliers" ของ Crystal ที่ทำให้ custom types ทำงานได้เหมือน built-in types

---

*ต่อไป: Part 46 - Arrays*
