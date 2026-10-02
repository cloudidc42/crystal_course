# Part 46: Arrays ใน Crystal

## บทนำ

Array เป็น data structure พื้นฐานที่สุดใน Crystal เก็บข้อมูลหลายชิ้นในลำดับที่เฉพาะเจาะจง Crystal Arrays มี type safety และ methods ที่ครอบคลุมทุกการใช้งาน

---

## 46.1 การสร้าง Array

### วิธีต่างๆ

```crystal
# วิธีที่ 1: Array literal
numbers = [1, 2, 3, 4, 5]
names = ["Alice", "Bob", "Charlie"]
mixed = [1, "two", 3.0, true]  # Array(Int32 | String | Float64 | Bool)

# วิธีที่ 2: Array.new
empty_ints = Array(Int32).new         # []
sized = Array(Int32).new(5)           # [0, 0, 0, 0, 0]
filled = Array(Int32).new(5, 42)      # [42, 42, 42, 42, 42]

# วิธีที่ 3: Array.new กับ block
computed = Array(Int32).new(5) { |i| i * i }  # [0, 1, 4, 9, 16]
fibonacci = Array(Int32).new(10) do |i|
  if i <= 1
    i
  else
    a, b = 0, 1
    (i - 1).times { a, b = b, a + b }
    b
  end
end
puts fibonacci.inspect  # => [0, 1, 1, 2, 3, 5, 8, 13, 21, 34]

# วิธีที่ 4: Range to Array
range_arr = (1..10).to_a     # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
steps = (0..20).step(5).to_a  # [0, 5, 10, 15, 20]

# วิธีที่ 5: String shorthand
words = %w[apple banana cherry]   # ["apple", "banana", "cherry"]
symbols = %i[foo bar baz]          # [:foo, :bar, :baz]

puts numbers.inspect
puts names.inspect
puts computed.inspect
```

---

## 46.2 เพิ่มและลบ Elements

### push, pop, shift, unshift

```crystal
arr = [1, 2, 3]

# Push: เพิ่มท้าย
arr.push(4)     # => [1, 2, 3, 4]
arr << 5        # => [1, 2, 3, 4, 5]  (shorthand)

# Pop: ลบและคืนท้าย
last = arr.pop  # => 5
puts arr.inspect  # => [1, 2, 3, 4]

# Unshift: เพิ่มหน้า
arr.unshift(0)  # => [0, 1, 2, 3, 4]

# Shift: ลบและคืนหน้า
first = arr.shift  # => 0
puts arr.inspect   # => [1, 2, 3, 4]

# Push หลายตัว
arr.push(5, 6, 7)  # => [1, 2, 3, 4, 5, 6, 7]
puts arr.inspect

# pop? และ shift? (ไม่ raise เมื่อ empty)
empty = [] of Int32
puts empty.pop?    # => nil
puts empty.shift?  # => nil
```

### insert และ delete

```crystal
arr = [1, 2, 3, 4, 5]

# insert ที่ index
arr.insert(2, 99)
puts arr.inspect  # => [1, 2, 99, 3, 4, 5]

# insert หลายตัว
arr.insert(0, -2, -1, 0)
puts arr.inspect  # => [-2, -1, 0, 1, 2, 99, 3, 4, 5]

# delete ค่า
arr.delete(99)
puts arr.inspect  # => [-2, -1, 0, 1, 2, 3, 4, 5]

# delete_at index
arr.delete_at(0)
puts arr.inspect  # => [-1, 0, 1, 2, 3, 4, 5]

# delete_at range
arr.delete_at(0, 2)  # ลบตั้งแต่ index 0 จำนวน 2 ตัว
puts arr.inspect  # => [1, 2, 3, 4, 5]
```

---

## 46.3 Accessing Elements

```crystal
arr = [10, 20, 30, 40, 50]

# Index ปกติ
puts arr[0]    # => 10 (ตัวแรก)
puts arr[4]    # => 50 (ตัวสุดท้าย)
puts arr[-1]   # => 50 (นับจากท้าย)
puts arr[-2]   # => 40

# Range
puts arr[1..3].inspect   # => [20, 30, 40]
puts arr[1...3].inspect  # => [20, 30]
puts arr[2..].inspect    # => [30, 40, 50]
puts arr[..2].inspect    # => [10, 20, 30]

# Size และ index
puts arr[1, 3].inspect   # => [20, 30, 40] (start, length)

# first/last
puts arr.first    # => 10
puts arr.last     # => 50
puts arr.first(3).inspect  # => [10, 20, 30]
puts arr.last(2).inspect   # => [40, 50]

# at? (ไม่ raise เมื่อ out of bounds)
puts arr.at?(10)   # => nil
puts arr.at?(2)    # => 30

# fetch กับ default
puts arr.fetch(10, 0)    # => 0
puts arr.fetch(2)        # => 30
```

---

## 46.4 Size และ Information

```crystal
arr = [1, 2, 3, 4, 5, 2, 3]

puts arr.size     # => 7
puts arr.length   # => 7 (alias ของ size)
puts arr.count    # => 7
puts arr.count(2)            # => 2 (นับ 2 ที่อยู่ใน array)
puts arr.count { |n| n > 3 }  # => 2

puts arr.empty?   # => false
puts [].empty?    # => true

puts arr.any?     # => true
puts [].any?      # => false
puts arr.any? { |n| n > 4 }  # => true
puts arr.all? { |n| n > 0 }  # => true
puts arr.none? { |n| n > 10 } # => true

puts arr.sum       # => 20
puts arr.sum { |n| n * 2 }  # => 40
puts arr.product   # => 288

puts arr.min       # => 1
puts arr.max       # => 5
puts arr.minmax    # => {1, 5}
```

---

## 46.5 Searching

```crystal
arr = ["apple", "banana", "cherry", "date", "elderberry"]

# include?
puts arr.includes?("banana")  # => true
puts arr.includes?("grape")   # => false

# index
puts arr.index("cherry")      # => 2
puts arr.index("grape")       # => nil

puts arr.index { |s| s.size > 6 }   # => 1 (banana)

puts arr.rindex("cherry")           # => 2 (จากขวา)
puts arr.rindex { |s| s.size > 4 }  # => 4 (elderberry)

# find
puts arr.find { |s| s.starts_with?("c") }  # => "cherry"
puts arr.find_index { |s| s.ends_with?("y") }  # => 2

# select ที่ match
matches = arr.select { |s| s.size == 6 }
puts matches.inspect  # => ["banana", "cherry"]
```

---

## 46.6 Transformations

### map

```crystal
numbers = [1, 2, 3, 4, 5]

# map: สร้าง array ใหม่จาก transformation
doubled = numbers.map { |n| n * 2 }
puts doubled.inspect  # => [2, 4, 6, 8, 10]

# map กับ method reference
words = ["hello", "world", "crystal"]
upcase_words = words.map(&.upcase)
puts upcase_words.inspect  # => ["HELLO", "WORLD", "CRYSTAL"]

sizes = words.map(&.size)
puts sizes.inspect  # => [5, 5, 7]

# map! (in-place)
numbers.map! { |n| n ** 2 }
puts numbers.inspect  # => [1, 4, 9, 16, 25]
```

### flat_map

```crystal
nested = [[1, 2], [3, 4], [5, 6]]
flat = nested.flat_map { |arr| arr }
puts flat.inspect  # => [1, 2, 3, 4, 5, 6]

# เพิ่มค่าแล้ว flatten
result = [1, 2, 3].flat_map { |n| [n, n * 10] }
puts result.inspect  # => [1, 10, 2, 20, 3, 30]
```

### flatten

```crystal
deep = [1, [2, [3, [4]]]]
puts deep.flatten.inspect     # => [1, 2, 3, 4]
puts deep.flatten(1).inspect  # => [1, 2, [3, [4]]]
puts deep.flatten(2).inspect  # => [1, 2, 3, [4]]
```

---

## 46.7 Filtering

```crystal
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# select
evens = numbers.select { |n| n.even? }
puts evens.inspect  # => [2, 4, 6, 8, 10]

# reject
odds = numbers.reject { |n| n.even? }
puts odds.inspect  # => [1, 3, 5, 7, 9]

# filter_map (select + map)
doubled_evens = numbers.filter_map { |n| n * 2 if n.even? }
puts doubled_evens.inspect  # => [4, 8, 12, 16, 20]

# compact (ลบ nil)
mixed = [1, nil, 2, nil, 3]
puts mixed.compact.inspect  # => [1, 2, 3]

# uniq (ลบซ้ำ)
dupes = [1, 2, 2, 3, 3, 3, 4]
puts dupes.uniq.inspect  # => [1, 2, 3, 4]
puts dupes.uniq { |n| n % 2 }.inspect  # => [1, 2]

# select! (in-place)
arr = [1, 2, 3, 4, 5, 6]
arr.select! { |n| n > 3 }
puts arr.inspect  # => [4, 5, 6]
```

---

## 46.8 Sorting

```crystal
numbers = [5, 3, 1, 4, 2]
words = ["banana", "apple", "cherry", "date"]

# sort (คืน array ใหม่)
puts numbers.sort.inspect           # => [1, 2, 3, 4, 5]
puts numbers.sort { |a, b| b <=> a }.inspect  # => [5, 4, 3, 2, 1]

# sort! (in-place)
numbers.sort!
puts numbers.inspect  # => [1, 2, 3, 4, 5]

# sort_by
puts words.sort_by { |w| w.size }.inspect
# => ["date", "apple", "banana", "cherry"]

puts words.sort_by { |w| [-w.size, w] }.inspect
# => ["banana", "cherry", "apple", "date"]

# sort_by! (in-place)
words.sort_by!(&.size)
puts words.inspect  # => ["date", "apple", "banana", "cherry"]

# reverse
puts [1, 2, 3].reverse.inspect  # => [3, 2, 1]
```

---

## 46.9 Reduce/Inject

```crystal
numbers = [1, 2, 3, 4, 5]

# reduce
sum = numbers.reduce { |acc, n| acc + n }
puts sum  # => 15

# reduce กับ initial value
product = numbers.reduce(1) { |acc, n| acc * n }
puts product  # => 120

# inject (alias)
max = numbers.inject { |m, n| n > m ? n : m }
puts max  # => 5

# each_with_object
grouped = numbers.each_with_object({} of String => Array(Int32)) do |n, hash|
  key = n.even? ? "even" : "odd"
  (hash[key] ||= [] of Int32) << n
end
puts grouped.inspect
# => {"odd" => [1, 3, 5], "even" => [2, 4]}
```

---

## 46.10 Combining Arrays

```crystal
a = [1, 2, 3]
b = [4, 5, 6]

# Concatenation
puts (a + b).inspect  # => [1, 2, 3, 4, 5, 6]

# concat (in-place)
c = [1, 2, 3]
c.concat([4, 5, 6])
puts c.inspect  # => [1, 2, 3, 4, 5, 6]

# zip
puts a.zip(b).inspect  # => [{1, 4}, {2, 5}, {3, 6}]

# zip กับ 3 arrays
d = [7, 8, 9]
puts a.zip(b, d).inspect  # => [{1, 4, 7}, {2, 5, 8}, {3, 6, 9}]

# zip กับ block
a.zip(b) { |x, y| print "#{x+y} " }
puts  # => 5 7 9

# product (Cartesian product)
puts [1, 2].product([3, 4]).inspect  # => [[1, 3], [1, 4], [2, 3], [2, 4]]

# flatten หลังจาก product
combos = [1, 2].product([3, 4]).map { |a, b| a * b }
puts combos.inspect  # => [3, 4, 6, 8]
```

---

## 46.11 Combinations และ Permutations

```crystal
arr = [1, 2, 3, 4]

# combination
puts arr.combination(2).to_a.inspect
# => [[1, 2], [1, 3], [1, 4], [2, 3], [2, 4], [3, 4]]

puts arr.combination(3).to_a.inspect
# => [[1, 2, 3], [1, 2, 4], [1, 3, 4], [2, 3, 4]]

# permutation
puts [1, 2, 3].permutation(2).to_a.inspect
# => [[1, 2], [1, 3], [2, 1], [2, 3], [3, 1], [3, 2]]

puts [1, 2, 3].permutation.to_a.size  # => 6 (3!)
```

---

## 46.12 Iteration Methods

```crystal
arr = [1, 2, 3, 4, 5, 6, 7, 8]

# each
arr.each { |n| print "#{n} " }
puts

# each_with_index
arr.each_with_index { |n, i| print "#{i}:#{n} " }
puts

# each_index
arr.each_index { |i| print "#{i} " }
puts

# each_slice
arr.each_slice(3) { |slice| puts slice.inspect }
# => [1, 2, 3]
# => [4, 5, 6]
# => [7, 8]

# each_cons
arr.each_cons(3) { |cons| puts cons.inspect }
# => [1, 2, 3]
# => [2, 3, 4]
# => [3, 4, 5]
# => [4, 5, 6]
# => [5, 6, 7]
# => [6, 7, 8]

# each_cartesian_product
[1, 2].each_cartesian([3, 4]) { |a, b| print "(#{a},#{b}) " }
puts
```

---

## 46.13 Utility Methods

```crystal
arr = [3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5]

# tally (นับความถี่)
puts arr.tally.inspect  # => {3 => 2, 1 => 2, 4 => 1, 5 => 3, 9 => 1, 2 => 1, 6 => 1}

# chunk (จัดกลุ่มตามค่าที่ต่อเนื่อง)
[1, 1, 2, 2, 3, 1, 1].chunk { |n| n }.each do |key, values|
  puts "#{key}: #{values.inspect}"
end
# 1: [1, 1]
# 2: [2, 2]
# 3: [3]
# 1: [1, 1]

# chunk_while
[1, 2, 3, 5, 6, 10, 11, 12].chunk_while { |a, b| b - a == 1 }.each do |group|
  puts group.inspect
end
# [1, 2, 3]
# [5, 6]
# [10, 11, 12]

# rotate
puts arr.rotate(3).inspect  # rotate left by 3
puts arr.rotate(-1).inspect  # rotate right by 1

# sample (สุ่ม)
puts arr.sample         # สุ่ม 1 ตัว
puts arr.sample(3).inspect  # สุ่ม 3 ตัว

# shuffle
puts arr.shuffle.inspect   # สุ่มเรียง (คืน array ใหม่)
```

---

## 46.14 Set Operations

```crystal
a = [1, 2, 3, 4, 5]
b = [3, 4, 5, 6, 7]

# Union
puts (a | b).inspect  # => [1, 2, 3, 4, 5, 6, 7]

# Intersection
puts (a & b).inspect  # => [3, 4, 5]

# Difference
puts (a - b).inspect  # => [1, 2]
puts (b - a).inspect  # => [6, 7]
```

---

## 46.15 ตัวอย่างในชีวิตจริง

### Shopping Cart

```crystal
class ShoppingCart
  record Item, name : String, price : Float64, quantity : Int32
  
  def initialize
    @items = [] of Item
  end
  
  def add(name : String, price : Float64, quantity : Int32 = 1)
    existing = @items.find_index { |item| item.name == name }
    if existing
      old = @items[existing]
      @items[existing] = Item.new(old.name, old.price, old.quantity + quantity)
    else
      @items << Item.new(name, price, quantity)
    end
    self
  end
  
  def remove(name : String) : Bool
    index = @items.find_index { |item| item.name == name }
    return false unless index
    @items.delete_at(index)
    true
  end
  
  def total : Float64
    @items.sum { |item| item.price * item.quantity }
  end
  
  def item_count : Int32
    @items.sum(&.quantity)
  end
  
  def most_expensive : Item?
    @items.max_by? { |item| item.price }
  end
  
  def items_above(price : Float64) : Array(Item)
    @items.select { |item| item.price > price }
  end
  
  def sorted_by_price : Array(Item)
    @items.sort_by { |item| -item.price }
  end
  
  def to_s : String
    lines = ["=== Shopping Cart ==="]
    sorted_by_price.each do |item|
      lines << "  #{item.name} x#{item.quantity} @ $#{item.price} = $#{(item.price * item.quantity).round(2)}"
    end
    lines << "Total: $#{total.round(2)} (#{item_count} items)"
    lines.join("\n")
  end
end

cart = ShoppingCart.new
cart.add("Crystal Book", 39.99, 2)
    .add("Programming Notebook", 15.99)
    .add("Coffee", 8.50, 3)
    .add("Pen Set", 12.99)

puts cart
puts "\nMost expensive: #{cart.most_expensive.try(&.name)}"
puts "Items > $15: #{cart.items_above(15.0).map(&.name).inspect}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Matrix Operations

```crystal
class Matrix
  def initialize(@rows : Array(Array(Float64)))
    @height = @rows.size
    @width = @rows[0]?.try(&.size) || 0
  end
  
  def self.zeros(rows : Int32, cols : Int32) : Matrix
    new(Array.new(rows) { Array.new(cols, 0.0) })
  end
  
  def self.identity(size : Int32) : Matrix
    new(Array.new(size) { |i| Array.new(size) { |j| i == j ? 1.0 : 0.0 } })
  end
  
  def [](row : Int32, col : Int32) : Float64
    @rows[row][col]
  end
  
  def +(other : Matrix) : Matrix
    Matrix.new(@rows.zip(other.@rows).map { |r1, r2|
      r1.zip(r2).map { |a, b| a + b }
    })
  end
  
  def *(scalar : Float64) : Matrix
    Matrix.new(@rows.map { |row| row.map { |val| val * scalar } })
  end
  
  def transpose : Matrix
    Matrix.new(Array.new(@width) { |j| Array.new(@height) { |i| @rows[i][j] } })
  end
  
  def to_s : String
    @rows.map { |row| row.map { |v| v.round(1).to_s.rjust(6) }.join }.join("\n")
  end
end

m = Matrix.new([[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]])
puts m
puts "---"
puts (m * 2.0)
puts "---"
puts m.transpose
```

### แบบฝึกหัดที่ 2: Statistics

```crystal
def statistics(data : Array(Float64)) : Hash(String, Float64)
  sorted = data.sort
  n = data.size.to_f
  
  mean = data.sum / n
  variance = data.sum { |x| (x - mean) ** 2 } / n
  std_dev = Math.sqrt(variance)
  median = n.even? ? 
    (sorted[(n/2 - 1).to_i] + sorted[(n/2).to_i]) / 2 :
    sorted[(n/2).to_i]
  
  {
    "count" => n,
    "sum" => data.sum,
    "mean" => mean,
    "median" => median,
    "std_dev" => std_dev,
    "min" => sorted.first.to_f,
    "max" => sorted.last.to_f
  }
end

data = [23.5, 18.2, 31.7, 25.0, 19.8, 28.3, 22.1, 35.6, 21.4, 27.9]
stats = statistics(data)

stats.each do |key, value|
  puts "#{key}: #{value.round(2)}"
end
```

---

## สรุป

Array ใน Crystal มี methods ครอบคลุมทุกการใช้งาน:

| หมวด | Methods |
|------|---------|
| สร้าง | `[]`, `Array.new`, `(range).to_a` |
| เพิ่ม/ลบ | `push`, `<<`, `pop`, `shift`, `unshift`, `insert`, `delete` |
| เข้าถึง | `[]`, `first`, `last`, `at?`, `fetch` |
| ค้นหา | `find`, `index`, `includes?`, `select`, `reject` |
| แปลง | `map`, `flat_map`, `flatten`, `compact`, `uniq` |
| เรียง | `sort`, `sort_by`, `reverse` |
| รวม | `reduce`, `sum`, `inject`, `each_with_object` |
| วนลูป | `each`, `each_with_index`, `each_slice`, `each_cons` |
| ต่อกัน | `+`, `concat`, `zip`, `product` |
| ข้อมูล | `size`, `empty?`, `any?`, `all?`, `none?` |

---

*ต่อไป: Part 47 - Hashes*
