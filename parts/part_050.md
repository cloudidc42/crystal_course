# Part 50: Sets ใน Crystal

## บทนำ

Set เป็น collection ที่เก็บข้อมูล **unique** โดยไม่มีลำดับ (unordered) เหมาะสำหรับการตรวจสอบการมีอยู่ของ element และการดำเนินการทาง set theory (union, intersection, difference)

---

## 50.1 การเริ่มต้น

```crystal
require "set"

# สร้าง Set
set1 = Set(Int32).new
set2 = Set{1, 2, 3, 4, 5}  # Set literal

# สร้างจาก Array
arr = [1, 2, 2, 3, 3, 3, 4]
set_from_arr = arr.to_set
puts set_from_arr.inspect  # => Set{1, 2, 3, 4}

# สร้างและ initialize
numbers = Set(Int32).new([1, 2, 3, 4, 5])
puts numbers.inspect  # => Set{1, 2, 3, 4, 5}
```

---

## 50.2 add

```crystal
require "set"

fruits = Set(String).new

# add
fruits.add("apple")
fruits.add("banana")
fruits.add("cherry")
fruits.add("apple")  # ซ้ำ - ไม่ถูกเพิ่ม

puts fruits.inspect  # => Set{"apple", "banana", "cherry"}
puts fruits.size     # => 3

# add? (คืน nil ถ้าซ้ำ)
result = fruits.add?("mango")
puts result.inspect   # => Set{"apple", "banana", "cherry", "mango"}

already = fruits.add?("apple")
puts already.nil?   # => true (แสดงว่าไม่ได้เพิ่ม)

# concat (เพิ่มหลายตัว)
more = ["fig", "grape", "kiwi"]
fruits.concat(more)
puts fruits.size  # => 7
```

---

## 50.3 delete

```crystal
require "set"

numbers = Set{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}

# delete
numbers.delete(5)
puts numbers.includes?(5)  # => false

# delete? (คืน element ถ้าลบสำเร็จ)
deleted = numbers.delete?(3)
puts deleted  # => 3

not_found = numbers.delete?(99)
puts not_found.nil?  # => true

# ลบหลายตัว
[2, 4, 6, 8].each { |n| numbers.delete(n) }
puts numbers.to_a.sort.inspect  # => [1, 7, 9, 10]
```

---

## 50.4 includes?

```crystal
require "set"

allowed_roles = Set{"admin", "editor", "viewer"}

# includes?
puts allowed_roles.includes?("admin")   # => true
puts allowed_roles.includes?("guest")   # => false

# includes? (alias)
puts allowed_roles.includes?("editor")  # => true

# ใช้ตรวจสอบ permission
def can_access?(role : String, required_roles : Set(String)) : Bool
  required_roles.includes?(role)
end

puts can_access?("admin", allowed_roles)   # => true
puts can_access?("moderator", allowed_roles) # => false

# Subset check ใช้ทีหลัง
user_permissions = Set{"read", "write"}
required = Set{"read", "write", "delete"}
puts user_permissions.subset_of?(required)   # => true
puts required.subset_of?(user_permissions)   # => false
```

---

## 50.5 Union (|)

```crystal
require "set"

set_a = Set{1, 2, 3, 4, 5}
set_b = Set{3, 4, 5, 6, 7}

# Union: รวมทุก element จากทั้งสอง set
union = set_a | set_b
puts union.to_a.sort.inspect  # => [1, 2, 3, 4, 5, 6, 7]

# หรือใช้ union method
puts set_a.union(set_b).to_a.sort.inspect

# Use case: รวม permissions
admin_perms = Set{"read", "write", "delete", "admin"}
editor_perms = Set{"read", "write"}
viewer_perms = Set{"read"}

all_perms = admin_perms | editor_perms | viewer_perms
puts all_perms.to_a.sort.inspect
# => ["admin", "delete", "read", "write"]

# Union ไม่เปลี่ยน original
puts set_a.inspect  # ยังเป็น Set{1, 2, 3, 4, 5}
```

---

## 50.6 Intersection (&)

```crystal
require "set"

set_a = Set{1, 2, 3, 4, 5}
set_b = Set{3, 4, 5, 6, 7}

# Intersection: เฉพาะ elements ที่อยู่ในทั้งสอง set
intersection = set_a & set_b
puts intersection.to_a.sort.inspect  # => [3, 4, 5]

# Use case: หา skills ที่ทับซ้อน
alice_skills = Set{"crystal", "ruby", "python", "go"}
bob_skills = Set{"python", "javascript", "go", "rust"}

shared = alice_skills & bob_skills
puts "Shared skills: #{shared.to_a.sort.inspect}"
# => Shared skills: ["go", "python"]

# หา common interests
set1 = Set{:music, :sports, :cooking, :reading}
set2 = Set{:sports, :gaming, :cooking, :travel}
set3 = Set{:sports, :cooking, :reading, :art}

common_all = set1 & set2 & set3
puts "Common to all: #{common_all.to_a.inspect}"
# => Common to all: [:sports, :cooking]
```

---

## 50.7 Difference (-)

```crystal
require "set"

all_numbers = Set{1, 2, 3, 4, 5, 6, 7, 8, 9, 10}
even_numbers = Set{2, 4, 6, 8, 10}

# Difference: elements ใน set_a แต่ไม่ใน set_b
odd_numbers = all_numbers - even_numbers
puts odd_numbers.to_a.sort.inspect  # => [1, 3, 5, 7, 9]

# Use case: หา skills ที่ Alice มีแต่ Bob ไม่มี
alice_skills = Set{"crystal", "ruby", "python", "go"}
bob_skills = Set{"python", "javascript", "go", "rust"}

alice_unique = alice_skills - bob_skills
puts "Alice's unique skills: #{alice_unique.to_a.sort.inspect}"
# => Alice's unique skills: ["crystal", "ruby"]

bob_unique = bob_skills - alice_skills
puts "Bob's unique skills: #{bob_unique.to_a.sort.inspect}"
# => Bob's unique skills: ["javascript", "rust"]

# Symmetric difference (XOR - อยู่ใน A หรือ B แต่ไม่ทั้งคู่)
symmetric_diff = (alice_skills - bob_skills) | (bob_skills - alice_skills)
puts "Either but not both: #{symmetric_diff.to_a.sort.inspect}"
```

---

## 50.8 subset? และ superset?

```crystal
require "set"

full = Set{1, 2, 3, 4, 5}
partial = Set{2, 3, 4}
other = Set{6, 7, 8}

# subset? (A ⊆ B)
puts partial.subset_of?(full)      # => true
puts full.subset_of?(full)         # => true (set เป็น subset ของตัวเอง)
puts full.subset_of?(partial)      # => false

# superset? (A ⊇ B)
puts full.superset_of?(partial)    # => true
puts partial.superset_of?(full)    # => false

# proper_subset? (A ⊂ B - strict subset)
puts partial.proper_subset_of?(full)  # => true
puts full.proper_subset_of?(full)     # => false (ไม่ใช่ proper)

# Use case: Permission checking
required_permissions = Set{"read", "write"}
user_permissions = Set{"read", "write", "admin"}
minimal_user = Set{"read"}

puts "Full access: #{required_permissions.subset_of?(user_permissions)}"  # => true
puts "Limited access: #{required_permissions.subset_of?(minimal_user)}"   # => false
```

---

## 50.9 each

```crystal
require "set"

tags = Set{"crystal", "programming", "tutorial", "backend"}

# each
tags.each do |tag|
  puts "Tag: #{tag}"
end

# each_with_index
tags.each_with_index do |tag, i|
  puts "#{i}: #{tag}"
end

# map (คืน Array)
upper_tags = tags.map(&.upcase)
puts upper_tags.class    # => Array(String)
puts upper_tags.inspect

# select (คืน Array)
long_tags = tags.select { |t| t.size > 7 }
puts long_tags.inspect  # => ["programming", "tutorial"]

# to_a
arr = tags.to_a
puts arr.class    # => Array(String)
```

---

## 50.10 to_a และ size

```crystal
require "set"

set = Set{5, 3, 1, 4, 2}

# to_a (ลำดับไม่รับประกัน)
arr = set.to_a
puts arr.sort.inspect  # => [1, 2, 3, 4, 5]

# size
puts set.size    # => 5
puts set.empty?  # => false
puts Set(Int32).new.empty?  # => true

# count
puts set.count   # => 5
puts set.count { |n| n > 3 }  # => 2

# any? / all? / none?
puts set.any? { |n| n > 4 }    # => true
puts set.all? { |n| n > 0 }    # => true
puts set.none? { |n| n > 10 }  # => true
```

---

## 50.11 Practical Use Cases

### Bloom Filter พื้นฐาน

```crystal
require "set"

# Bloom filter concept (simplified) ด้วย Set
class SimpleBloomFilter(T)
  def initialize
    @seen = Set(T).new
    @false_positive_count = 0
  end
  
  def add(item : T)
    @seen.add(item)
  end
  
  def might_contain?(item : T) : Bool
    @seen.includes?(item)
  end
  
  def definitely_not_contain?(item : T) : Bool
    !might_contain?(item)
  end
  
  def size : Int32
    @seen.size
  end
end

filter = SimpleBloomFilter(String).new
urls = ["google.com", "github.com", "crystal-lang.org", "example.com"]
urls.each { |url| filter.add(url) }

puts filter.might_contain?("google.com")     # => true
puts filter.might_contain?("unknown.com")    # => false
puts filter.definitely_not_contain?("evil.com")  # => true
```

### Unique Visitor Tracker

```crystal
require "set"

class UniqueVisitorTracker
  def initialize
    @daily_visitors = Hash(String, Set(String)).new { |h, k| h[k] = Set(String).new }
  end
  
  def record_visit(date : String, user_id : String)
    @daily_visitors[date].add(user_id)
  end
  
  def unique_count(date : String) : Int32
    @daily_visitors[date]?.try(&.size) || 0
  end
  
  def total_unique : Int32
    all_visitors = @daily_visitors.values.reduce(Set(String).new) { |acc, s| acc | s }
    all_visitors.size
  end
  
  def returning_visitors(date1 : String, date2 : String) : Set(String)
    @daily_visitors[date1] & @daily_visitors[date2]
  end
  
  def new_visitors(date1 : String, date2 : String) : Set(String)
    @daily_visitors[date2] - @daily_visitors[date1]
  end
end

tracker = UniqueVisitorTracker.new

# Day 1
["user_1", "user_2", "user_3", "user_1", "user_2"].each do |u|
  tracker.record_visit("2024-01-01", u)
end

# Day 2
["user_2", "user_3", "user_4", "user_5"].each do |u|
  tracker.record_visit("2024-01-02", u)
end

puts "Day 1 unique: #{tracker.unique_count("2024-01-01")}"  # => 3
puts "Day 2 unique: #{tracker.unique_count("2024-01-02")}"  # => 4
puts "Total unique: #{tracker.total_unique}"  # => 5

returning = tracker.returning_visitors("2024-01-01", "2024-01-02")
puts "Returning: #{returning.to_a.sort.inspect}"  # => ["user_2", "user_3"]

new_v = tracker.new_visitors("2024-01-01", "2024-01-02")
puts "New: #{new_v.to_a.sort.inspect}"  # => ["user_4", "user_5"]
```

### Graph Connectivity

```crystal
require "set"

class Graph
  def initialize
    @adjacency = Hash(String, Set(String)).new { |h, k| h[k] = Set(String).new }
  end
  
  def add_edge(from : String, to : String)
    @adjacency[from].add(to)
    @adjacency[to].add(from)  # undirected
  end
  
  def neighbors(node : String) : Set(String)
    @adjacency[node]? || Set(String).new
  end
  
  def connected?(start : String, target : String) : Bool
    visited = Set(String).new
    queue = [start]
    
    while !queue.empty?
      current = queue.shift
      return true if current == target
      next if visited.includes?(current)
      
      visited.add(current)
      queue.concat(neighbors(current).to_a.reject { |n| visited.includes?(n) })
    end
    
    false
  end
  
  def find_all_connected(start : String) : Set(String)
    visited = Set(String).new
    queue = [start]
    
    while !queue.empty?
      current = queue.shift
      next if visited.includes?(current)
      
      visited.add(current)
      queue.concat(neighbors(current).to_a)
    end
    
    visited
  end
end

g = Graph.new
g.add_edge("A", "B")
g.add_edge("B", "C")
g.add_edge("C", "D")
g.add_edge("E", "F")  # disconnected component

puts g.connected?("A", "D")  # => true
puts g.connected?("A", "F")  # => false

component = g.find_all_connected("A")
puts "Connected to A: #{component.to_a.sort.inspect}"
# => ["A", "B", "C", "D"]
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Tag System

```crystal
require "set"

class Article
  getter title : String
  getter tags : Set(String)
  
  def initialize(@title : String, tags : Array(String))
    @tags = tags.to_set
  end
  
  def add_tag(tag : String)
    @tags.add(tag)
  end
  
  def remove_tag(tag : String)
    @tags.delete(tag)
  end
  
  def has_tag?(tag : String) : Bool
    @tags.includes?(tag)
  end
  
  def related_to?(other : Article) : Bool
    !(@tags & other.tags).empty?
  end
  
  def common_tags_with(other : Article) : Set(String)
    @tags & other.tags
  end
end

a1 = Article.new("Crystal Basics", ["crystal", "programming", "tutorial"])
a2 = Article.new("Crystal Types", ["crystal", "types", "programming"])
a3 = Article.new("Python vs Ruby", ["python", "ruby", "comparison"])

puts "a1 related to a2: #{a1.related_to?(a2)}"  # => true
puts "a1 related to a3: #{a1.related_to?(a3)}"  # => false

common = a1.common_tags_with(a2)
puts "Common: #{common.to_a.sort.inspect}"
# => ["crystal", "programming"]

# Find articles by tag
articles = [a1, a2, a3]
crystal_articles = articles.select { |a| a.has_tag?("crystal") }
puts "Crystal articles: #{crystal_articles.map(&.title).join(", ")}"
```

### แบบฝึกหัดที่ 2: Set Operations

```crystal
require "set"

# ทดสอบ set operations
def demo_set_operations(a : Set(Int32), b : Set(Int32))
  puts "A = #{a.to_a.sort.inspect}"
  puts "B = #{b.to_a.sort.inspect}"
  puts "A ∪ B = #{(a | b).to_a.sort.inspect}"
  puts "A ∩ B = #{(a & b).to_a.sort.inspect}"
  puts "A - B = #{(a - b).to_a.sort.inspect}"
  puts "B - A = #{(b - a).to_a.sort.inspect}"
  puts "A ⊆ B: #{a.subset_of?(b)}"
  puts "B ⊆ A: #{b.subset_of?(a)}"
end

a = Set{1, 2, 3, 4, 5}
b = Set{3, 4, 5, 6, 7}

demo_set_operations(a, b)
```

---

## สรุป

Set ใน Crystal (ต้อง `require "set"`):

| Method | คำอธิบาย |
|--------|---------|
| `Set(T).new` | สร้าง empty set |
| `Set{1,2,3}` | สร้างด้วย literal |
| `add(item)` | เพิ่ม element |
| `delete(item)` | ลบ element |
| `includes?(item)` | ตรวจสอบการมีอยู่ |
| `\|` (union) | รวม elements |
| `&` (intersection) | elements ร่วม |
| `-` (difference) | ลบ elements |
| `subset_of?` | ตรวจว่าเป็น subset |
| `superset_of?` | ตรวจว่าเป็น superset |
| `each` | วนลูป |
| `to_a` | แปลงเป็น Array |
| `size` | จำนวน elements |
| `empty?` | ตรวจว่าว่างเปล่า |

**ข้อดีของ Set:**
- ค้นหาเร็ว O(1) เฉลี่ย
- ไม่มีซ้ำ
- Set operations ที่สะดวก (union, intersection, difference)
- เหมาะสำหรับ unique tracking, membership testing

---

*ต่อไป: Part 51 - Deque*
