# Part 47: Hashes ใน Crystal

## บทนำ

Hash เป็น data structure ที่เก็บข้อมูลในรูปแบบ key-value pairs ช่วยให้เข้าถึงข้อมูลด้วย key แทน index Crystal Hash มี type safety และ methods ที่ครอบคลุม

---

## 47.1 การสร้าง Hash

```crystal
# วิธีที่ 1: Hash literal
person = {"name" => "Alice", "age" => 30, "city" => "Bangkok"}

# วิธีที่ 2: Hash.new
empty_hash = Hash(String, Int32).new
hash_with_default = Hash(String, Int32).new(0)  # default value = 0

# วิธีที่ 3: ระบุ type ชัดเจน
scores = {} of String => Int32
scores["Alice"] = 95
scores["Bob"] = 87

# วิธีที่ 4: Symbol keys (TypeAnnotation)
config = {:host => "localhost", :port => 8080, :debug => true}

# วิธีที่ 5: สร้างจาก arrays
keys = ["a", "b", "c"]
values = [1, 2, 3]
from_arrays = keys.zip(values).to_h
puts from_arrays.inspect  # => {"a" => 1, "b" => 2, "c" => 3}

# วิธีที่ 6: สร้างจาก array ด้วย block
square_map = (1..5).each_with_object({} of Int32 => Int32) do |n, h|
  h[n] = n * n
end
puts square_map.inspect  # => {1 => 1, 2 => 4, 3 => 9, 4 => 16, 5 => 25}
```

---

## 47.2 การเข้าถึงและแก้ไข

### [] และ []=

```crystal
h = {"name" => "Alice", "age" => 30}

# อ่าน
puts h["name"]  # => "Alice"
# puts h["missing"]  # => KeyError!

# อ่านแบบปลอดภัย
puts h["missing"]?     # => nil
puts h.fetch("name")   # => "Alice"
puts h.fetch("missing", "unknown")  # => "unknown"
puts h.fetch("missing") { |k| "Key '#{k}' not found" }

# เขียน
h["city"] = "Bangkok"
h["age"] = 31

puts h.inspect  # => {"name" => "Alice", "age" => 31, "city" => "Bangkok"}
```

### Default Values

```crystal
# Hash กับ default value
word_count = Hash(String, Int32).new(0)
text = "the quick brown fox jumps over the lazy dog the fox"

text.split.each do |word|
  word_count[word] += 1  # ไม่ raise KeyError เพราะมี default 0
end

puts word_count.sort_by { |_, v| -v }.first(3).inspect
# [{"the", 3}, {"fox", 2}, ...]

# Default กับ block
groups = Hash(String, Array(Int32)).new { |h, k| h[k] = [] of Int32 }
[1, 2, 3, 4, 5, 6].each do |n|
  groups[n.even? ? "even" : "odd"] << n
end
puts groups.inspect  # => {"odd" => [1, 3, 5], "even" => [2, 4, 6]}
```

---

## 47.3 Keys และ Values

```crystal
h = {"a" => 1, "b" => 2, "c" => 3, "d" => 4}

# Keys
puts h.keys.inspect         # => ["a", "b", "c", "d"]
puts h.has_key?("a")        # => true
puts h.has_key?("z")        # => false
puts h.key?("b")            # => true (alias)
puts h.includes?("c")       # => true (alias)

# Values
puts h.values.inspect       # => [1, 2, 3, 4]
puts h.has_value?(3)        # => true
puts h.has_value?(99)       # => false
puts h.value?("a")          # => 1 (คืน value ของ key "a" แต่ method นี้ไม่มีใน Crystal std)

# Key จาก value
puts h.key_for(3)           # => "c"
puts h.key_for?(99)         # => nil

# Count
puts h.size   # => 4
puts h.empty? # => false
```

---

## 47.4 Iteration

```crystal
h = {"apple" => 1.99, "banana" => 0.99, "cherry" => 3.49}

# each
h.each do |key, value|
  puts "#{key}: $#{value}"
end

# each_key
h.each_key { |k| print "#{k} " }
puts

# each_value
h.each_value { |v| print "#{v} " }
puts

# each_with_object
result = h.each_with_object([] of String) do |(k, v), arr|
  arr << "#{k}=$#{v}" if v < 2.0
end
puts result.inspect  # => ["apple=$1.99", "banana=$0.99"]

# map (คืน Array)
pairs = h.map { |k, v| "#{k}: #{v}" }
puts pairs.inspect

# map (คืน Hash)
doubled = h.transform_values { |v| v * 2 }
puts doubled.inspect  # => {"apple" => 3.98, "banana" => 1.98, "cherry" => 6.98}
```

---

## 47.5 Filtering

```crystal
inventory = {
  "apple" => 50,
  "banana" => 0,
  "cherry" => 25,
  "date" => 0,
  "elderberry" => 10
}

# select (คืน Hash ใหม่)
in_stock = inventory.select { |_, qty| qty > 0 }
puts in_stock.inspect
# => {"apple" => 50, "cherry" => 25, "elderberry" => 10}

# reject (คืน Hash ใหม่)
out_of_stock = inventory.reject { |_, qty| qty > 0 }
puts out_of_stock.inspect
# => {"banana" => 0, "date" => 0}

# filter_map
expensive = {"a" => 1.99, "b" => 9.99, "c" => 4.99, "d" => 0.50}
pricey = expensive.filter_map { |k, v| "#{k}:$#{v}" if v > 5.0 }
puts pricey.inspect  # => ["b:$9.99"]

# any? / all? / none?
puts inventory.any? { |_, qty| qty > 40 }    # => true
puts inventory.all? { |_, qty| qty >= 0 }     # => true
puts inventory.none? { |_, qty| qty > 100 }   # => true
puts inventory.count { |_, qty| qty == 0 }    # => 2
```

---

## 47.6 Transformations

### transform_keys

```crystal
h = {"name" => "Alice", "age" => 30, "city" => "Bangkok"}

# transform_keys
symbolized = h.transform_keys(&.to_sym)
puts symbolized.inspect
# => {:name => "Alice", :age => 30, :city => "Bangkok"}

uppercased = h.transform_keys(&.upcase)
puts uppercased.inspect
# => {"NAME" => "Alice", "AGE" => 30, "CITY" => "Bangkok"}
```

### transform_values

```crystal
prices = {"apple" => 1.99, "banana" => 0.99, "cherry" => 3.49}

# transform_values
discounted = prices.transform_values { |price| (price * 0.9).round(2) }
puts discounted.inspect
# => {"apple" => 1.79, "banana" => 0.89, "cherry" => 3.14}

# transform_values!  (in-place)
prices.transform_values! { |price| price * 1.1 }  # 10% increase
```

---

## 47.7 Merging

```crystal
h1 = {"a" => 1, "b" => 2}
h2 = {"b" => 3, "c" => 4}

# merge (h2 overwrites h1)
puts h1.merge(h2).inspect   # => {"a" => 1, "b" => 3, "c" => 4}

# merge กับ block (resolve conflicts)
merged = h1.merge(h2) { |key, old_val, new_val| old_val + new_val }
puts merged.inspect  # => {"a" => 1, "b" => 5, "c" => 4}

# merge! (in-place)
h3 = {"x" => 10}
h3.merge!({"y" => 20, "z" => 30})
puts h3.inspect  # => {"x" => 10, "y" => 20, "z" => 30}

# update (alias ของ merge!)
h3.update({"x" => 99})
puts h3.inspect  # => {"x" => 99, "y" => 20, "z" => 30}
```

---

## 47.8 Deletion

```crystal
h = {"a" => 1, "b" => 2, "c" => 3, "d" => 4}

# delete by key
deleted = h.delete("b")
puts deleted  # => 2
puts h.inspect  # => {"a" => 1, "c" => 3, "d" => 4}

# delete กับ block (ถ้าไม่มี key)
result = h.delete("z") { |k| "Key #{k} not found" }
puts result  # => "Key z not found"

# reject! (in-place filter)
h.reject! { |_, v| v > 2 }
puts h.inspect  # => {"a" => 1}

# clear
h.clear
puts h.empty?  # => true
```

---

## 47.9 Aggregation Methods

```crystal
scores = {"Alice" => 95, "Bob" => 87, "Charlie" => 92, "Dave" => 78}

# sum
total = scores.sum { |_, score| score }
puts "Total: #{total}"  # => 352

# min_by / max_by
top = scores.max_by { |_, score| score }
puts "Top scorer: #{top[0]} (#{top[1]})"  # => Alice (95)

lowest = scores.min_by { |_, score| score }
puts "Lowest: #{lowest[0]} (#{lowest[1]})"  # => Dave (78)

# sort_by
sorted = scores.sort_by { |_, score| -score }
sorted.each { |name, score| puts "  #{name}: #{score}" }

# count
above_90 = scores.count { |_, score| score >= 90 }
puts "Above 90: #{above_90}"  # => 2
```

---

## 47.10 Conversion

```crystal
h = {"a" => 1, "b" => 2, "c" => 3}

# to_a
pairs = h.to_a
puts pairs.inspect  # => [{"a", 1}, {"b", 2}, {"c", 3}]

# flat_map
flattened = h.flat_map { |k, v| [k, v] }
puts flattened.inspect  # => ["a", 1, "b", 2, "c", 3]

# from pairs back to hash
back = pairs.to_h
puts back.inspect  # => {"a" => 1, "b" => 2, "c" => 3}

# invert (swap keys and values)
inverted = h.invert
puts inverted.inspect  # => {1 => "a", 2 => "b", 3 => "c"}
```

---

## 47.11 ตัวอย่างในชีวิตจริง

### Configuration Manager

```crystal
class Config
  def initialize
    @data = {} of String => String | Int32 | Bool | Float64
    @defaults = {} of String => String | Int32 | Bool | Float64
  end
  
  def set(key : String, value : String | Int32 | Bool | Float64) : self
    @data[key] = value
    self
  end
  
  def default(key : String, value : String | Int32 | Bool | Float64) : self
    @defaults[key] = value
    self
  end
  
  def get(key : String) : String | Int32 | Bool | Float64
    @data.fetch(key) { @defaults.fetch(key) { raise "Config key '#{key}' not found" } }
  end
  
  def get?(key : String) : String | Int32 | Bool | Float64 | Nil
    @data[key]? || @defaults[key]?
  end
  
  def string(key : String) : String
    get(key).as(String)
  end
  
  def int(key : String) : Int32
    get(key).as(Int32)
  end
  
  def bool(key : String) : Bool
    get(key).as(Bool)
  end
  
  def merge_from(other : Hash(String, String | Int32 | Bool | Float64)) : self
    @data.merge!(other)
    self
  end
  
  def to_s(io : IO) : Nil
    all = @defaults.merge(@data)
    all.each { |k, v| io << "#{k} = #{v}\n" }
  end
end

config = Config.new
config.default("host", "localhost")
      .default("port", 8080)
      .default("debug", false)
      .default("max_connections", 100)
      .set("host", "api.example.com")
      .set("port", 443)

puts config.string("host")   # => "api.example.com"
puts config.int("port")      # => 443
puts config.bool("debug")    # => false
puts config.int("max_connections")  # => 100
puts config
```

### Inventory System

```crystal
class Inventory
  record Product, name : String, price : Float64, quantity : Int32
  
  def initialize
    @products = {} of String => Product
  end
  
  def add(id : String, name : String, price : Float64, quantity : Int32 = 0)
    @products[id] = Product.new(name, price, quantity)
    self
  end
  
  def restock(id : String, quantity : Int32)
    product = @products.fetch(id) { raise "Product #{id} not found" }
    @products[id] = Product.new(product.name, product.price, product.quantity + quantity)
  end
  
  def sell(id : String, quantity : Int32) : Float64
    product = @products.fetch(id) { raise "Product #{id} not found" }
    raise "Insufficient stock" if product.quantity < quantity
    @products[id] = Product.new(product.name, product.price, product.quantity - quantity)
    product.price * quantity
  end
  
  def in_stock : Hash(String, Product)
    @products.select { |_, p| p.quantity > 0 }
  end
  
  def out_of_stock : Hash(String, Product)
    @products.reject { |_, p| p.quantity > 0 }
  end
  
  def total_value : Float64
    @products.sum { |_, p| p.price * p.quantity }
  end
  
  def low_stock(threshold : Int32 = 5) : Hash(String, Product)
    @products.select { |_, p| p.quantity in (1..threshold) }
  end
  
  def report : String
    lines = ["=== Inventory Report ==="]
    @products.sort_by { |_, p| -p.quantity }.each do |id, p|
      status = p.quantity == 0 ? " [OUT OF STOCK]" : ""
      lines << "#{id}: #{p.name} - $#{p.price} x #{p.quantity}#{status}"
    end
    lines << "Total Value: $#{total_value.round(2)}"
    lines.join("\n")
  end
end

inv = Inventory.new
inv.add("A001", "Crystal Book", 39.99, 10)
   .add("A002", "Programming Notebook", 15.99, 50)
   .add("A003", "USB Cable", 8.99, 3)
   .add("A004", "Keyboard", 89.99, 0)

inv.sell("A001", 3)
inv.restock("A003", 20)

puts inv.report
puts "\nLow stock:"
inv.low_stock.each { |id, p| puts "  #{id}: #{p.name} (#{p.quantity} left)" }
```

---

## 47.12 Nested Hashes

```crystal
# Nested Hash
config = {
  "database" => {
    "host" => "localhost",
    "port" => "5432",
    "name" => "mydb"
  },
  "cache" => {
    "host" => "localhost",
    "port" => "6379"
  }
}

# เข้าถึง nested
puts config["database"]["host"]  # => "localhost"

# Safe navigation
puts config["redis"]?.try { |r| r["host"] }  # => nil

# deep merge (ต้องเขียนเอง)
def deep_merge(base : Hash, override : Hash) : Hash
  result = base.dup
  override.each do |k, v|
    if result.has_key?(k) && result[k].is_a?(Hash) && v.is_a?(Hash)
      result[k] = deep_merge(result[k].as(Hash), v.as(Hash))
    else
      result[k] = v
    end
  end
  result
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Word Frequency Analyzer

```crystal
def analyze_text(text : String) : Hash(String, Int32)
  # ลบ punctuation และ normalize
  cleaned = text.downcase.gsub(/[^a-z\s]/, "")
  
  # นับความถี่
  freq = Hash(String, Int32).new(0)
  cleaned.split.each { |word| freq[word] += 1 }
  
  freq
end

def top_words(freq : Hash(String, Int32), n : Int32 = 10) : Array({String, Int32})
  freq.sort_by { |_, count| -count }.first(n).map { |k, v| {k, v} }
end

text = "Crystal is a programming language with Ruby-like syntax. Crystal is compiled and has type safety. Crystal is fast and efficient."
freq = analyze_text(text)

puts "Top 5 words:"
top_words(freq, 5).each_with_index do |(word, count), i|
  puts "  #{i+1}. '#{word}': #{count} times"
end
```

### แบบฝึกหัดที่ 2: Group By Multiple Criteria

```crystal
students = [
  {"name" => "Alice", "grade" => "A", "year" => 2},
  {"name" => "Bob", "grade" => "B", "year" => 1},
  {"name" => "Charlie", "grade" => "A", "year" => 1},
  {"name" => "Dave", "grade" => "C", "year" => 3},
  {"name" => "Eve", "grade" => "A", "year" => 2},
  {"name" => "Frank", "grade" => "B", "year" => 1}
]

# Group by grade
by_grade = students.group_by { |s| s["grade"] }
by_grade.each do |grade, group|
  names = group.map { |s| s["name"] }
  puts "Grade #{grade}: #{names.join(", ")}"
end

puts ""

# Group by year
by_year = students.group_by { |s| s["year"] }
by_year.sort_by { |year, _| year.as(Int32) }.each do |year, group|
  puts "Year #{year}: #{group.map { |s| s["name"] }.join(", ")}"
end
```

---

## สรุป

Hash ใน Crystal คือ dictionary ที่มี type safety:

| หมวด | Methods |
|------|---------|
| สร้าง | `{}`, `Hash.new`, `.to_h` |
| อ่าน/เขียน | `[]`, `[]=`, `fetch`, `[]?` |
| ตรวจสอบ | `has_key?`, `has_value?`, `key?`, `includes?` |
| Keys/Values | `keys`, `values`, `key_for` |
| วนลูป | `each`, `each_key`, `each_value` |
| กรอง | `select`, `reject`, `filter_map` |
| แปลง | `map`, `transform_keys`, `transform_values` |
| รวม | `merge`, `merge!`, `update` |
| ลบ | `delete`, `reject!`, `clear` |
| รวมข้อมูล | `sum`, `count`, `min_by`, `max_by`, `sort_by` |
| แปลงชนิด | `to_a`, `flat_map`, `invert` |

---

*ต่อไป: Part 48 - Tuples*
