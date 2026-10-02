# Part 97: Forall ใน Crystal

## บทนำ

`forall` ใน Crystal เป็น keyword ที่ใช้กำหนด type variables ใน generic methods ให้ compiler สามารถ infer types ได้อย่างถูกต้อง

## forall พื้นฐาน

```crystal
# forall T - declare type variable T
def identity(x : T) : T forall T
  x
end

puts identity(42)       # => 42 (T = Int32)
puts identity("hello")  # => hello (T = String)
puts identity(3.14)     # => 3.14 (T = Float64)
puts identity([1, 2, 3]).inspect  # => [1, 2, 3] (T = Array(Int32))

# หลาย type variables
def swap(a : T, b : U) : Tuple(U, T) forall T, U
  {b, a}
end

puts swap(1, "hello").inspect      # => {"hello", 1}
puts swap("world", 42).inspect     # => {42, "world"}
puts swap(3.14, true).inspect      # => {true, 3.14}
```

## forall กับ Containers

```crystal
# Generic container operations
def first_of(arr : Array(T)) : T forall T
  arr.first
end

def last_of(arr : Array(T)) : T forall T
  arr.last
end

def wrap_in_array(x : T) : Array(T) forall T
  [x]
end

puts first_of([1, 2, 3])     # => 1 (T = Int32)
puts first_of(["a", "b"])    # => a (T = String)
puts last_of([1.0, 2.0])    # => 2.0 (T = Float64)
puts wrap_in_array(42).inspect    # => [42]
puts wrap_in_array("hi").inspect  # => ["hi"]
```

## forall กับ Constraints (ด้วย is_a? หรือ responds_to?)

```crystal
# Constrain T ต้องมี certain methods
def sum_all(items : Array(T)) : T forall T
  items.reduce { |acc, val| acc + val }
end

puts sum_all([1, 2, 3, 4, 5])       # => 15
puts sum_all([1.5, 2.5, 3.0])       # => 7.0
puts sum_all(["a", "b", "c"])        # => "abc"
# จะ compile error ถ้า T ไม่มี +

# Constraint กับ Comparable
def sorted_pair(a : T, b : T) : Tuple(T, T) forall T
  a <= b ? {a, b} : {b, a}
end

puts sorted_pair(5, 3).inspect       # => {3, 5}
puts sorted_pair("b", "a").inspect   # => {"a", "b"}
puts sorted_pair(3.14, 2.71).inspect # => {2.71, 3.14}

# ถ้า T ไม่ implement <=> จะได้ compile error
```

## Generic Methods ที่ Return Different Types

```crystal
# T และ U เป็น independent type variables
def map_value(value : T, &block : T -> U) : U forall T, U
  block.call(value)
end

puts map_value(5) { |x| x * 2 }          # => 10 (T=Int32, U=Int32)
puts map_value(5) { |x| x.to_s }         # => "5" (T=Int32, U=String)
puts map_value("hello") { |s| s.size }   # => 5 (T=String, U=Int32)
puts map_value(3.14) { |f| f > 3.0 }    # => true (T=Float64, U=Bool)

# Chain type transformations
def double_map(value : T, f : T -> U, g : U -> V) : V forall T, U, V
  g.call(f.call(value))
end

result = double_map(42, ->(x : Int32) { x.to_s }, ->(s : String) { s.size })
puts result  # => 2 (length of "42")
```

## Practical Generic Programming

```crystal
# Binary search generic
def binary_search(arr : Array(T), target : T) : Int32? forall T
  low = 0
  high = arr.size - 1

  while low <= high
    mid = (low + high) // 2
    if arr[mid] == target
      return mid
    elsif arr[mid] < target
      low = mid + 1
    else
      high = mid - 1
    end
  end

  nil
end

sorted_ints = [1, 3, 5, 7, 9, 11, 13, 15]
puts binary_search(sorted_ints, 7).inspect   # => 3
puts binary_search(sorted_ints, 6).inspect   # => nil

sorted_strs = ["apple", "banana", "cherry", "date"]
puts binary_search(sorted_strs, "cherry").inspect  # => 2
puts binary_search(sorted_strs, "grape").inspect   # => nil

# Merge sort generic
def merge_sort(arr : Array(T)) : Array(T) forall T
  return arr if arr.size <= 1

  mid = arr.size // 2
  left = merge_sort(arr[0...mid])
  right = merge_sort(arr[mid..])

  merge(left, right)
end

private def merge(left : Array(T), right : Array(T)) : Array(T) forall T
  result = Array(T).new
  i = 0
  j = 0

  while i < left.size && j < right.size
    if left[i] <= right[j]
      result << left[i]
      i += 1
    else
      result << right[j]
      j += 1
    end
  end

  result.concat(left[i..])
  result.concat(right[j..])
  result
end

puts merge_sort([5, 3, 8, 1, 9, 2, 7]).inspect
# => [1, 2, 3, 5, 7, 8, 9]

puts merge_sort(["banana", "apple", "cherry", "date"]).inspect
# => ["apple", "banana", "cherry", "date"]
```

## forall กับ Blocks และ Procs

```crystal
# Generic block type
def transform_array(arr : Array(T), &block : T -> U) : Array(U) forall T, U
  arr.map { |x| block.call(x) }
end

numbers = [1, 2, 3, 4, 5]
doubled = transform_array(numbers) { |x| x * 2 }
strings = transform_array(numbers) { |x| x.to_s }
bools = transform_array(numbers) { |x| x.even? }

puts doubled.inspect  # => [2, 4, 6, 8, 10]
puts strings.inspect  # => ["1", "2", "3", "4", "5"]
puts bools.inspect    # => [false, true, false, true, false]

# Generic predicate
def partition_by(arr : Array(T), &pred : T -> Bool) : Tuple(Array(T), Array(T)) forall T
  yes = Array(T).new
  no = Array(T).new

  arr.each do |item|
    if pred.call(item)
      yes << item
    else
      no << item
    end
  end

  {yes, no}
end

evens, odds = partition_by([1, 2, 3, 4, 5, 6]) { |x| x.even? }
puts "Evens: #{evens.inspect}"  # => [2, 4, 6]
puts "Odds: #{odds.inspect}"    # => [1, 3, 5]
```

## forall กับ Recursive Types

```crystal
# Tree structure ด้วย forall
class Tree(T)
  getter value : T
  getter children : Array(Tree(T))

  def initialize(@value : T)
    @children = Array(Tree(T)).new
  end

  def add_child(child : Tree(T)) : self
    @children << child
    self
  end

  def add_child(value : T) : self
    @children << Tree(T).new(value)
    self
  end

  def depth : Int32
    if @children.empty?
      1
    else
      1 + @children.map(&.depth).max
    end
  end

  def each(&block : T -> Nil)
    block.call(@value)
    @children.each { |child| child.each(&block) }
  end

  def map(&block : T -> U) : Tree(U) forall U
    new_tree = Tree(U).new(block.call(@value))
    @children.each do |child|
      new_tree.add_child(child.map(&block))
    end
    new_tree
  end

  def find(&pred : T -> Bool) : T? forall T
    return @value if pred.call(@value)
    @children.each do |child|
      result = child.find(&pred)
      return result if result
    end
    nil
  end
end

# สร้าง tree
root = Tree(Int32).new(1)
root.add_child(Tree(Int32).new(2).tap { |n|
  n.add_child(4).add_child(5)
})
root.add_child(Tree(Int32).new(3).tap { |n|
  n.add_child(6)
})

puts "Depth: #{root.depth}"  # => 3

# Collect all values
all_values = [] of Int32
root.each { |v| all_values << v }
puts all_values.sort.inspect  # => [1, 2, 3, 4, 5, 6]

# Map tree
doubled_tree = root.map { |v| v * 2 }
doubled_values = [] of Int32
doubled_tree.each { |v| doubled_values << v }
puts doubled_values.sort.inspect  # => [2, 4, 6, 8, 10, 12]

# Find value
found = root.find { |v| v > 4 }
puts found.inspect  # => 5
```

## forall กับ Multiple Constraints

```crystal
# รวม multiple forall constraints
def zip_and_transform(a : Array(T), b : Array(U), &block : T, U -> V) : Array(V) forall T, U, V
  size = [a.size, b.size].min
  Array(V).new(size) { |i| block.call(a[i], b[i]) }
end

names = ["Alice", "Bob", "Charlie"]
ages = [30, 25, 35]
scores = [95.5, 82.3, 88.7]

combined = zip_and_transform(names, ages) { |name, age| "#{name}(#{age})" }
puts combined.inspect  # => ["Alice(30)", "Bob(25)", "Charlie(35)"]

weighted = zip_and_transform(ages, scores) { |age, score| age.to_f * score / 100 }
puts weighted.map { |x| x.round(2) }.inspect
```

## forall ใน Abstract Methods

```crystal
# Generic abstract methods
abstract class Transformer(T)
  abstract def transform(value : T) : T
  abstract def identity : T

  def apply_n_times(value : T, n : Int32) : T
    n.times.reduce(value) { |acc, _| transform(acc) }
  end
end

class DoubleTransformer < Transformer(Int32)
  def transform(value : Int32) : Int32
    value * 2
  end

  def identity : Int32
    1
  end
end

class UppercaseTransformer < Transformer(String)
  def transform(value : String) : String
    value.upcase
  end

  def identity : String
    ""
  end
end

dt = DoubleTransformer.new
puts dt.apply_n_times(1, 5)  # => 32 (1 * 2^5)

ut = UppercaseTransformer.new
puts ut.transform("hello")   # => HELLO
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Generic Functional Utilities
สร้าง utility functions:
- `flatten(arr : Array(Array(T))) : Array(T)`
- `group_by(arr, &key_fn) : Hash(K, Array(T))`
- `zip_with_index(arr) : Array(Tuple(Int32, T))`
- `chunk(arr, size) : Array(Array(T))`

### แบบฝึกหัดที่ 2: Priority Queue
สร้าง `PriorityQueue(T)` ที่:
- Generic ที่ใช้ forall
- Min-heap หรือ Max-heap ตาม Comparable
- รองรับ custom comparator

### แบบฝึกหัดที่ 3: Generic Cache
สร้าง `Cache(K, V)` ที่:
- LRU eviction
- TTL support
- Thread-safe
- รองรับ generic key และ value types

### แบบฝึกหัดที่ 4: Type-safe Event System
สร้าง event system ที่:
- `Event(T)` base type
- `EventBus` ที่ dispatch based on event type
- Type-safe handlers
- forall สำหรับ handler registration

## สรุป

`forall` ใน Crystal:
- **Type Variables**: `T`, `U`, `V` - inferred by compiler
- **Multiple Variables**: `forall T, U, V` สำหรับหลาย types
- **Monomorphization**: compiler สร้าง specialized code สำหรับแต่ละ type combination
- **Implicit Constraints**: ถ้าใช้ method ใน body, type ต้องรองรับ

Key differences จาก Java/C# generics:
1. ไม่ต้องระบุ type bounds อย่างชัดเจน (ยกเว้น abstract)
2. Compiler infer constraints จาก method calls
3. No boxing - structs stay as values
4. Monomorphization ทำให้เร็วกว่า virtual dispatch
