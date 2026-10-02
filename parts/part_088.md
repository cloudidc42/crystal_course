# Part 88: Generics ใน Crystal

## บทนำ

Generics ใน Crystal ช่วยให้เราเขียนโค้ดที่ทำงานได้กับ type หลากหลายโดยไม่สูญเสีย type safety หรือประสิทธิภาพ Crystal ใช้ approach ที่เรียกว่า "monomorphization" ซึ่งสร้าง specialized code สำหรับแต่ละ type ที่ใช้จริง

## Generic Class พื้นฐาน

```crystal
# Box(T) - container สำหรับ type ใดๆ
class Box(T)
  getter value : T

  def initialize(@value : T)
  end

  def map(&block : T -> U) : Box(U) forall U
    Box.new(block.call(@value))
  end

  def transform(new_value : T) : Box(T)
    Box.new(new_value)
  end

  def to_s : String
    "Box(#{@value})"
  end
end

# ใช้งาน
int_box = Box.new(42)
puts int_box.value         # => 42
puts int_box.class         # => Box(Int32)

str_box = Box.new("Crystal")
puts str_box.value         # => Crystal
puts str_box.class         # => Box(String)

# Map เปลี่ยน type
str_from_int = int_box.map { |v| v.to_s }
puts str_from_int.value    # => "42"
puts str_from_int.class    # => Box(String)
```

## Generic Stack

```crystal
class Stack(T)
  def initialize
    @data = Array(T).new
  end

  def push(value : T) : self
    @data.push(value)
    self
  end

  def pop : T
    raise "Stack is empty" if empty?
    @data.pop
  end

  def peek : T
    raise "Stack is empty" if empty?
    @data.last
  end

  def empty? : Bool
    @data.empty?
  end

  def size : Int32
    @data.size
  end

  def to_a : Array(T)
    @data.dup
  end
end

# ใช้งาน
stack = Stack(Int32).new
stack.push(1).push(2).push(3)

puts stack.peek  # => 3
puts stack.pop   # => 3
puts stack.size  # => 2

# String stack
words = Stack(String).new
words.push("Crystal").push("is").push("awesome")
puts words.pop  # => "awesome"
```

## Generic Queue

```crystal
class Queue(T)
  def initialize
    @data = Deque(T).new
  end

  def enqueue(value : T) : self
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

  def empty? : Bool
    @data.empty?
  end

  def size : Int32
    @data.size
  end
end

queue = Queue(String).new
queue.enqueue("task1").enqueue("task2").enqueue("task3")

while !queue.empty?
  puts "Processing: #{queue.dequeue}"
end
```

## Generic Methods ด้วย forall

```crystal
# Generic method ที่ทำงานกับ type ใดๆ
def identity(x : T) : T forall T
  x
end

puts identity(42)       # => 42
puts identity("hello")  # => hello
puts identity(3.14)     # => 3.14

# Generic method ที่ return type ต่างกัน
def first_and_last(arr : Array(T)) : Tuple(T, T) forall T
  raise "Array must have at least 2 elements" if arr.size < 2
  {arr.first, arr.last}
end

first, last = first_and_last([1, 2, 3, 4, 5])
puts "First: #{first}, Last: #{last}"

# Multiple type params
def zip_with(a : Array(T), b : Array(U), &block : T, U -> V) : Array(V) forall T, U, V
  size = [a.size, b.size].min
  Array(V).new(size) do |i|
    block.call(a[i], b[i])
  end
end

nums = [1, 2, 3]
strs = ["a", "b", "c"]
zipped = zip_with(nums, strs) { |n, s| "#{n}#{s}" }
puts zipped.inspect  # => ["1a", "2b", "3c"]
```

## Generic Constraints

```crystal
# Comparable constraint
def max_of(a : T, b : T) : T forall T
  a > b ? a : b
end

puts max_of(3, 7)        # => 7
puts max_of("abc", "xyz")  # => xyz

# Number constraint
def sum_array(arr : Array(T)) : T forall T
  arr.reduce { |acc, val| acc + val }
end

puts sum_array([1, 2, 3, 4, 5])    # => 15
puts sum_array([1.5, 2.5, 3.0])    # => 7.0

# Constraint ด้วย module
module Printable
  abstract def print_info : String
end

def display_all(items : Array(T)) forall T
  items.each do |item|
    if item.responds_to?(:print_info)
      puts item.print_info
    else
      puts item.inspect
    end
  end
end

struct Point
  include Printable

  getter x : Float64
  getter y : Float64

  def initialize(@x, @y)
  end

  def print_info : String
    "(#{@x}, #{@y})"
  end
end

points = [Point.new(1.0, 2.0), Point.new(3.0, 4.0)]
display_all(points)
```

## Multiple Type Parameters

```crystal
# Pair(K, V) - คู่ของค่า
struct Pair(K, V)
  getter first : K
  getter second : V

  def initialize(@first : K, @second : V)
  end

  def swap : Pair(V, K)
    Pair.new(@second, @first)
  end

  def map_first(&block : K -> U) : Pair(U, V) forall U
    Pair.new(block.call(@first), @second)
  end

  def map_second(&block : V -> U) : Pair(K, U) forall U
    Pair.new(@first, block.call(@second))
  end

  def to_s : String
    "(#{@first}, #{@second})"
  end
end

pair = Pair.new("age", 25)
puts pair           # => (age, 25)
puts pair.swap      # => (25, age)
puts pair.map_second { |v| v * 2 }  # => (age, 50)

# Result type ด้วย generics
struct Result(T, E)
  getter value : T | E

  def self.ok(value : T) : self
    new(value.as(T | E))
  end

  def self.error(error : E) : self
    new(error.as(T | E))
  end

  def ok? : Bool
    @value.is_a?(T)
  end

  def error? : Bool
    @value.is_a?(E)
  end

  def unwrap : T
    if v = @value.as?(T)
      v
    else
      raise "Result is an error: #{@value}"
    end
  end

  def unwrap_error : E
    if e = @value.as?(E)
      e
    else
      raise "Result is not an error"
    end
  end

  private def initialize(@value : T | E)
  end
end

def parse_int(s : String) : Result(Int32, String)
  Result(Int32, String).ok(s.to_i)
rescue
  Result(Int32, String).error("Invalid integer: #{s}")
end

r1 = parse_int("42")
r2 = parse_int("not_a_number")

puts r1.ok?      # => true
puts r1.unwrap   # => 42

puts r2.error?            # => true
puts r2.unwrap_error      # => Invalid integer: not_a_number
```

## Generic Modules

```crystal
# Generic module ที่เป็น mixin
module Container(T)
  abstract def add(item : T)
  abstract def remove : T
  abstract def empty? : Bool
  abstract def size : Int32

  def any? : Bool
    !empty?
  end

  def count(&block : T -> Bool) : Int32
    # Default implementation
    0
  end
end

class BoundedStack(T)
  include Container(T)

  def initialize(@max_size : Int32)
    @data = Array(T).new
  end

  def add(item : T)
    raise "Stack overflow" if @data.size >= @max_size
    @data.push(item)
  end

  def remove : T
    raise "Stack underflow" if empty?
    @data.pop
  end

  def empty? : Bool
    @data.empty?
  end

  def size : Int32
    @data.size
  end

  def full? : Bool
    @data.size >= @max_size
  end
end

stack = BoundedStack(Int32).new(3)
stack.add(1)
stack.add(2)
stack.add(3)

puts stack.full?   # => true
puts stack.any?    # => true
puts stack.remove  # => 3
```

## Comparable Generic

```crystal
# Generic binary search tree
class BST(T)
  private class Node(T)
    property value : T
    property left : Node(T)?
    property right : Node(T)?

    def initialize(@value : T)
      @left = nil
      @right = nil
    end
  end

  def initialize
    @root = Node(T) | Nil
    @root = nil
  end

  def insert(value : T)
    @root = insert_node(@root, value)
  end

  def contains?(value : T) : Bool
    contains_node?(@root, value)
  end

  def to_sorted_array : Array(T)
    result = [] of T
    inorder(@root, result)
    result
  end

  private def insert_node(node : Node(T)?, value : T) : Node(T)
    if node.nil?
      Node(T).new(value)
    elsif value < node.value
      node.left = insert_node(node.left, value)
      node
    elsif value > node.value
      node.right = insert_node(node.right, value)
      node
    else
      node  # Duplicate - ignore
    end
  end

  private def contains_node?(node : Node(T)?, value : T) : Bool
    return false if node.nil?
    return true if node.value == value
    if value < node.value
      contains_node?(node.left, value)
    else
      contains_node?(node.right, value)
    end
  end

  private def inorder(node : Node(T)?, result : Array(T))
    return if node.nil?
    inorder(node.left, result)
    result << node.value
    inorder(node.right, result)
  end
end

tree = BST(Int32).new
[5, 3, 7, 1, 4, 6, 8].each { |v| tree.insert(v) }

puts tree.contains?(4)  # => true
puts tree.contains?(9)  # => false
puts tree.to_sorted_array.inspect  # => [1, 3, 4, 5, 6, 7, 8]

# String BST
str_tree = BST(String).new
["banana", "apple", "cherry", "date"].each { |v| str_tree.insert(v) }
puts str_tree.to_sorted_array.inspect  # => ["apple", "banana", "cherry", "date"]
```

## Type Inference กับ Generics

```crystal
# Crystal infers type parameters
arr = [1, 2, 3]                 # Array(Int32)
hash = {"a" => 1, "b" => 2}    # Hash(String, Int32)

# Type inference ใน generic methods
def wrap_in_array(x : T) : Array(T) forall T
  [x]
end

result = wrap_in_array(42)      # Array(Int32) - inferred
puts result.class               # => Array(Int32)

# เมื่อ type inference ซับซ้อน
def transform_pairs(pairs : Array(Tuple(K, V))) : Hash(K, V) forall K, V
  Hash(K, V).zip(pairs.map(&.[0]), pairs.map(&.[1]))
end

pairs = [{"a", 1}, {"b", 2}, {"c", 3}]
hash = transform_pairs(pairs)
puts hash  # => {"a" => 1, "b" => 2, "c" => 3}
```

## Generic Observable Pattern

```crystal
# Generic event system
class EventEmitter(T)
  alias Handler = T -> Nil

  def initialize
    @handlers = Array(Handler).new
  end

  def on(&block : Handler)
    @handlers << block
  end

  def emit(event : T)
    @handlers.each(&.call(event))
  end

  def off_all
    @handlers.clear
  end
end

# ใช้งานกับ custom event type
struct UserEvent
  enum Kind
    Created
    Updated
    Deleted
  end

  getter kind : Kind
  getter user_id : Int32
  getter details : String

  def initialize(@kind, @user_id, @details = "")
  end
end

emitter = EventEmitter(UserEvent).new

emitter.on do |event|
  puts "Handler 1: #{event.kind} user #{event.user_id}"
end

emitter.on do |event|
  puts "Handler 2: #{event.details}" unless event.details.empty?
end

emitter.emit(UserEvent.new(UserEvent::Kind::Created, 1, "New user created"))
emitter.emit(UserEvent.new(UserEvent::Kind::Updated, 1))
```

## Generic Pipeline

```crystal
# Functional pipeline ด้วย generics
class Pipeline(T)
  def initialize(@value : T)
  end

  def pipe(&block : T -> U) : Pipeline(U) forall U
    Pipeline.new(block.call(@value))
  end

  def value : T
    @value
  end
end

# Helper function
def pipeline(value : T) : Pipeline(T) forall T
  Pipeline.new(value)
end

result = pipeline(5)
  .pipe { |x| x * 2 }
  .pipe { |x| x.to_s }
  .pipe { |x| "Result: #{x}" }
  .value

puts result  # => "Result: 10"

# Pipeline สำหรับ data transformation
result2 = pipeline([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])
  .pipe { |arr| arr.select(&.even?) }
  .pipe { |arr| arr.map { |x| x ** 2 } }
  .pipe { |arr| arr.sum }
  .value

puts result2  # => 220 (4+16+36+64+100)
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Generic LRU Cache
สร้าง `LRUCache(K, V)` ที่:
- รองรับ get(key), put(key, value)
- มี max capacity
- Evict ข้อมูลเก่าสุดที่ใช้เมื่อเต็ม

### แบบฝึกหัดที่ 2: Generic Graph
สร้าง `Graph(T)` ที่:
- รองรับ add_vertex, add_edge
- มี BFS และ DFS traversal
- หา shortest path ได้

### แบบฝึกหัดที่ 3: Generic Functional Combinators
สร้าง functions:
- `compose(f, g)` - function composition
- `memoize(f)` - caching function results
- `retry(f, n)` - retry on failure

### แบบฝึกหัดที่ 4: Option(T) Type
Implement `Option(T)` ที่:
- มี `Some(value)` และ `None` states
- รองรับ `map`, `flat_map`, `or_else`
- Interop กับ Crystal nilable types

## สรุป

Generics ใน Crystal มีประสิทธิภาพสูงเพราะ:
- **Monomorphization**: สร้าง specialized code ที่ optimize สำหรับแต่ละ type
- **Type Safety**: ตรวจสอบ type ณ compile time ทั้งหมด
- **Zero Runtime Overhead**: ไม่มี boxing/unboxing
- **forall**: syntax ที่ชัดเจนสำหรับ generic type variables

Best practices:
1. ตั้งชื่อ type variable เป็น T, U, V หรือชื่อที่มีความหมาย
2. ใช้ constraints เพื่อจำกัด type ที่ accept ได้
3. หลีกเลี่ยง type variables ที่ไม่จำเป็น
4. Document generic constraints ใน comment
