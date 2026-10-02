# Part 59: Bit Arrays ใน Crystal

## บทนำ

`BitArray` เป็น data structure ที่เก็บ array ของ boolean values อย่างมีประสิทธิภาพโดยใช้ 1 bit ต่อ element แทน 1 byte แบบ Array(Bool) ปกติ เหมาะสำหรับงานที่ต้องการ memory น้อยและจัดการ flags จำนวนมาก

---

## 59.1 สร้าง BitArray

```crystal
# สร้าง BitArray ขนาด 8 bits (ทุก bit เริ่มเป็น false/0)
bits = BitArray.new(8)
puts bits.inspect  # => BitArray[00000000]
puts bits.size     # => 8

# สร้างด้วยค่าเริ่มต้น true
all_true = BitArray.new(8, true)
puts all_true.inspect  # => BitArray[11111111]

# สร้างขนาดใหญ่
large = BitArray.new(64)
puts "Size: #{large.size}"  # => 64
```

---

## 59.2 Set และ Get Bits

```crystal
bits = BitArray.new(8)

# Set bits (index, value)
bits[0] = true
bits[3] = true
bits[7] = true
puts bits.inspect  # => BitArray[10010001]

# Get bits
puts bits[0]  # => true
puts bits[1]  # => false
puts bits[3]  # => true

# ใช้ [] notation
bits[5] = true
puts bits[5]  # => true

# Error: index out of bounds
# bits[8] = true  # => raises IndexError
```

---

## 59.3 toggle (flip)

```crystal
bits = BitArray.new(8)
bits[0] = true
bits[2] = true
puts bits.inspect  # => BitArray[10100000]

# toggle กลับค่า
bits.toggle(0)  # true -> false
bits.toggle(1)  # false -> true
bits.toggle(2)  # true -> false
puts bits.inspect  # => BitArray[01000000]

# toggle ทุก bit
bits = BitArray.new(8, true)
(0...bits.size).each { |i| bits.toggle(i) }
puts bits.inspect  # => BitArray[00000000]
```

---

## 59.4 count

```crystal
bits = BitArray.new(10)
bits[0] = true
bits[3] = true
bits[7] = true

# count true bits
true_count = bits.count(true)
puts "True bits: #{true_count}"  # => 3

# count false bits
false_count = bits.count(false)
puts "False bits: #{false_count}"  # => 7

# หรือนับ true ด้วย count
active = bits.count { |b| b }
puts "Active: #{active}"  # => 3
```

---

## 59.5 Bitwise Operations: & | ^ ~

```crystal
a = BitArray.new(8)
b = BitArray.new(8)

# Set bits
[0, 2, 4, 6].each { |i| a[i] = true }  # 10101010
[1, 2, 3, 6].each { |i| b[i] = true }  # 01110010

puts "a: #{a.inspect}"  # => BitArray[10101010]
puts "b: #{b.inspect}"  # => BitArray[01110010]

# AND (&)
puts "a & b: #{(a & b).inspect}"  # => BitArray[00100010]

# OR (|)
puts "a | b: #{(a | b).inspect}"  # => BitArray[11111010]

# XOR (^)
puts "a ^ b: #{(a ^ b).inspect}"  # => BitArray[11011000]

# NOT (~)
puts "~a: #{(~a).inspect}"  # => BitArray[01010101]
```

---

## 59.6 each และ to_a

```crystal
bits = BitArray.new(6)
[1, 3, 5].each { |i| bits[i] = true }

# each iterates boolean values
bits.each { |b| print "#{b ? 1 : 0}" }
puts  # => 010101

# each_with_index
bits.each_with_index do |bit, i|
  puts "bit[#{i}] = #{bit}"
end

# to_a แปลงเป็น Array(Bool)
arr = bits.to_a
puts arr.inspect  # => [false, true, false, true, false, true]

# หา indices ที่เป็น true
true_indices = (0...bits.size).select { |i| bits[i] }
puts "True at: #{true_indices.inspect}"  # => [1, 3, 5]
```

---

## 59.7 ตัวอย่างจริง: Sieve of Eratosthenes

```crystal
def sieve_of_eratosthenes(limit : Int32) : Array(Int32)
  is_prime = BitArray.new(limit + 1, true)
  is_prime[0] = false
  is_prime[1] = false

  (2..Math.sqrt(limit.to_f).to_i).each do |i|
    if is_prime[i]
      # Mark multiples as composite
      j = i * i
      while j <= limit
        is_prime[j] = false
        j += i
      end
    end
  end

  # Collect prime indices
  (2..limit).select { |i| is_prime[i] }
end

primes = sieve_of_eratosthenes(50)
puts "Primes up to 50: #{primes.inspect}"
# => [2, 3, 5, 7, 11, 13, 17, 19, 23, 29, 31, 37, 41, 43, 47]

puts "Count: #{primes.size}"  # => 15
puts "Memory: 1 bit per number vs 4 bytes for Int32"
```

---

## 59.8 ตัวอย่างจริง: Permission Flags

```crystal
# แทน permission ด้วย bits
# bit 0 = read, bit 1 = write, bit 2 = execute, bit 3 = admin
module Permission
  READ    = 0
  WRITE   = 1
  EXECUTE = 2
  ADMIN   = 3
end

class UserPermissions
  getter bits : BitArray

  def initialize
    @bits = BitArray.new(8)
  end

  def grant(perm : Int32)
    @bits[perm] = true
  end

  def revoke(perm : Int32)
    @bits[perm] = false
  end

  def has?(perm : Int32) : Bool
    @bits[perm]
  end

  def to_s
    names = [] of String
    names << "read"    if @bits[Permission::READ]
    names << "write"   if @bits[Permission::WRITE]
    names << "execute" if @bits[Permission::EXECUTE]
    names << "admin"   if @bits[Permission::ADMIN]
    names.empty? ? "no permissions" : names.join(", ")
  end
end

# ใช้งาน
user = UserPermissions.new
user.grant(Permission::READ)
user.grant(Permission::WRITE)

puts "User permissions: #{user.to_s}"  # => read, write
puts "Can read: #{user.has?(Permission::READ)}"    # => true
puts "Can admin: #{user.has?(Permission::ADMIN)}"  # => false

user.grant(Permission::ADMIN)
user.revoke(Permission::WRITE)
puts "Updated: #{user.to_s}"  # => read, admin
```

---

## 59.9 ตัวอย่างจริง: Bloom Filter

```crystal
# Bloom Filter: probabilistic set membership (no false negatives, possible false positives)
class BloomFilter
  def initialize(size : Int32, @num_hashes : Int32 = 3)
    @bits = BitArray.new(size)
    @size = size
  end

  def add(item : String)
    @num_hashes.times do |i|
      index = hash(item, i)
      @bits[index] = true
    end
  end

  def possibly_contains?(item : String) : Bool
    @num_hashes.times do |i|
      index = hash(item, i)
      return false unless @bits[index]
    end
    true
  end

  def definitely_not_contains?(item : String) : Bool
    !possibly_contains?(item)
  end

  private def hash(item : String, seed : Int32) : Int32
    # Simple hash function combining item and seed
    h = seed * 2654435761_u32
    item.each_byte { |b| h = h ^ (b.to_u32 * 2246822519_u32) }
    (h % @size.to_u32).to_i32.abs
  end

  def bit_count_true : Int32
    @bits.count(true)
  end

  def fill_ratio : Float64
    bit_count_true.to_f / @size
  end
end

# ใช้งาน
filter = BloomFilter.new(100, 3)

# เพิ่ม words
["apple", "banana", "cherry", "date", "elderberry"].each do |word|
  filter.add(word)
end

puts "Testing membership:"
["apple", "grape", "banana", "fig"].each do |word|
  result = filter.possibly_contains?(word)
  puts "  '#{word}': #{result ? "possibly in set" : "definitely NOT in set"}"
end

puts "\nFill ratio: #{(filter.fill_ratio * 100).round(1)}%"
```

---

## 59.10 ตัวอย่างจริง: Visited Nodes Tracker

```crystal
# ใช้ BitArray ติดตาม visited nodes ใน graph traversal
class GraphTraversal
  def initialize(@adjacency : Array(Array(Int32)))
    @n = @adjacency.size
  end

  def bfs(start : Int32) : Array(Int32)
    visited = BitArray.new(@n)
    queue = [start]
    visited[start] = true
    order = [] of Int32

    while !queue.empty?
      node = queue.shift
      order << node

      @adjacency[node].each do |neighbor|
        unless visited[neighbor]
          visited[neighbor] = true
          queue << neighbor
        end
      end
    end

    order
  end

  def dfs(start : Int32) : Array(Int32)
    visited = BitArray.new(@n)
    order = [] of Int32
    dfs_helper(start, visited, order)
    order
  end

  private def dfs_helper(node : Int32, visited : BitArray, order : Array(Int32))
    visited[node] = true
    order << node
    @adjacency[node].each do |neighbor|
      dfs_helper(neighbor, visited, order) unless visited[neighbor]
    end
  end
end

# Graph: 0-1-2-3-4-5 with some edges
adj = [
  [1, 2],      # 0 connects to 1, 2
  [0, 3, 4],   # 1 connects to 0, 3, 4
  [0, 5],      # 2 connects to 0, 5
  [1],         # 3 connects to 1
  [1],         # 4 connects to 1
  [2],         # 5 connects to 2
]

graph = GraphTraversal.new(adj)
puts "BFS from 0: #{graph.bfs(0).inspect}"  # => [0, 1, 2, 3, 4, 5]
puts "DFS from 0: #{graph.dfs(0).inspect}"  # => [0, 1, 3, 4, 2, 5]
```

---

## 59.11 Memory Comparison

```crystal
# เปรียบเทียบ memory usage
n = 1_000_000

# Array(Bool): 1 byte per element
bool_array_size = n * 1  # bytes approximately
puts "Array(Bool) approx: #{bool_array_size / 1024} KB"  # => ~977 KB

# BitArray: 1 bit per element
bit_array_size = (n / 8) + 1
puts "BitArray approx: #{bit_array_size / 1024} KB"  # => ~122 KB

puts "BitArray is #{bool_array_size / bit_array_size}x more memory efficient"  # => ~8x

# สร้างจริงและวัด
arr = Array(Bool).new(100, false)
bits = BitArray.new(100)

# ทั้งคู่ทำงานเหมือนกัน
50.times { |i| arr[i*2] = true }
50.times { |i| bits[i*2] = true }

puts "Array true count: #{arr.count(true)}"   # => 50
puts "BitArray true count: #{bits.count(true)}"  # => 50
```

---

## 59.12 XOR สำหรับ Encryption

```crystal
# Simple XOR cipher ด้วย BitArray
def xor_bits(data : BitArray, key : BitArray) : BitArray
  raise "Size mismatch" unless data.size == key.size
  data ^ key
end

# แปลง string เป็น BitArray
def string_to_bits(str : String) : BitArray
  bits = BitArray.new(str.size * 8)
  str.each_byte.with_index do |byte, i|
    8.times do |j|
      bits[i * 8 + j] = ((byte >> (7 - j)) & 1) == 1
    end
  end
  bits
end

def bits_to_string(bits : BitArray) : String
  String.build do |sb|
    (0...bits.size // 8).each do |i|
      byte = 0_u8
      8.times do |j|
        byte = (byte << 1) | (bits[i * 8 + j] ? 1 : 0)
      end
      sb << byte.chr
    end
  end
end

# Demo
message = "Hi!"
key_str = "KEY"

msg_bits = string_to_bits(message)
key_bits = string_to_bits(key_str)

encrypted = xor_bits(msg_bits, key_bits)
decrypted = xor_bits(encrypted, key_bits)

puts "Original: #{message}"
puts "Decrypted: #{bits_to_string(decrypted)}"  # => Hi!
puts "Match: #{message == bits_to_string(decrypted)}"  # => true
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: BitArray Sudoku Solver Helper

```crystal
# ใช้ BitArray ติดตาม available numbers ใน Sudoku
class SudokuTracker
  def initialize
    # 9 rows x 9 possible values (1-9)
    @rows = Array.new(9) { BitArray.new(10) }
    @cols = Array.new(9) { BitArray.new(10) }
    @boxes = Array.new(9) { BitArray.new(10) }
  end

  def mark_used(row : Int32, col : Int32, num : Int32)
    @rows[row][num] = true
    @cols[col][num] = true
    @boxes[box_index(row, col)][num] = true
  end

  def available?(row : Int32, col : Int32, num : Int32) : Bool
    !@rows[row][num] && !@cols[col][num] && !@boxes[box_index(row, col)][num]
  end

  def available_numbers(row : Int32, col : Int32) : Array(Int32)
    (1..9).select { |n| available?(row, col, n) }
  end

  private def box_index(row : Int32, col : Int32) : Int32
    (row // 3) * 3 + (col // 3)
  end
end

tracker = SudokuTracker.new
tracker.mark_used(0, 0, 5)
tracker.mark_used(0, 1, 3)
tracker.mark_used(0, 5, 7)

puts "Available for (0,2): #{tracker.available_numbers(0, 2).inspect}"
# Would show numbers not used in row 0, col 2, or box 0
```

### แบบฝึกหัดที่ 2: Feature Flags

```crystal
# Feature flag system ด้วย BitArray
class FeatureFlags
  FEATURES = {
    "dark_mode" => 0,
    "beta_api" => 1,
    "analytics" => 2,
    "notifications" => 3,
    "new_ui" => 4
  }

  def initialize
    @flags = BitArray.new(FEATURES.size)
  end

  def enable(feature : String)
    idx = FEATURES[feature]?
    @flags[idx] = true if idx
  end

  def disable(feature : String)
    idx = FEATURES[feature]?
    @flags[idx] = false if idx
  end

  def enabled?(feature : String) : Bool
    idx = FEATURES[feature]?
    idx ? @flags[idx] : false
  end

  def enabled_features : Array(String)
    FEATURES.select { |_, idx| @flags[idx] }.keys
  end
end

flags = FeatureFlags.new
flags.enable("dark_mode")
flags.enable("analytics")
flags.enable("new_ui")

puts "Enabled: #{flags.enabled_features.inspect}"
# => ["dark_mode", "analytics", "new_ui"]
puts "Beta API: #{flags.enabled?("beta_api")}"  # => false
puts "Dark Mode: #{flags.enabled?("dark_mode")}"  # => true
```

---

## สรุป

BitArray ใน Crystal:

| Operation | Method |
|-----------|--------|
| สร้าง | `BitArray.new(size)` |
| Set bit | `bits[i] = true/false` |
| Get bit | `bits[i]` |
| Toggle | `bits.toggle(i)` |
| Count true | `bits.count(true)` |
| AND | `a & b` |
| OR | `a \| b` |
| XOR | `a ^ b` |
| NOT | `~a` |
| Convert | `bits.to_a` |
| Iterate | `bits.each { \|b\| }` |

**ใช้ BitArray เมื่อ:**
- ต้องการเก็บ boolean values จำนวนมาก
- Memory เป็นข้อจำกัด (ประหยัดกว่า Array(Bool) 8 เท่า)
- ต้องการ bitwise operations
- Sieve algorithms
- Feature flags / Permission bits
- Visited tracking ใน graph algorithms

---

*ต่อไป: Part 60 - Linked Lists และ Custom Collections*
