# Part 167: Property-based Testing ใน Crystal

## บทนำ

Property-based Testing (PBT) คือการทดสอบที่แทนที่จะระบุ input/output ที่แน่นอน เราระบุ "properties" หรือ "invariants" ที่ code ต้องรักษาไว้ เครื่องมือจะสุ่ม generate inputs เพื่อหา edge cases ที่ทำให้ property ล้มเหลว

## แนวคิด Fuzzing

```
Traditional Testing:
  input = [3, 1, 4, 1, 5]
  expected = [1, 1, 3, 4, 5]
  assert sorted(input) == expected

Property-based Testing:
  for any list L:
    sorted_L = sort(L)
    1. len(sorted_L) == len(L)              # ความยาวไม่เปลี่ยน
    2. sorted_L contains same elements as L # elements เหมือนกัน
    3. sorted_L[i] <= sorted_L[i+1]         # เรียงลำดับ
```

## ติดตั้ง Property Testing Library

```yaml
# shard.yml - ใช้ hypothesis.cr หรือ สร้างเอง
dependencies:
  # ตัวเลือก 1: lucky_hasher สำหรับ random generation
  lucky_hasher:
    github: luckyframework/lucky_hasher
```

## สร้าง Property Testing Framework เอง

```crystal
# src/property_testing/generator.cr
module PropertyTesting
  class Generator(T)
    def initialize(&@generate : Random -> T)
    end

    def sample(rng : Random = Random.new) : T
      @generate.call(rng)
    end

    def sample_n(n : Int32, rng : Random = Random.new) : Array(T)
      Array(T).new(n) { sample(rng) }
    end

    def map(&transform : T -> U) forall U
      Generator(U).new { |rng| transform.call(sample(rng)) }
    end

    def filter(&predicate : T -> Bool) : Generator(T)
      Generator(T).new do |rng|
        loop do
          val = sample(rng)
          return val if predicate.call(val)
        end
      end
    end
  end

  # Built-in Generators
  module Gen
    def self.int(min : Int32 = -1000, max : Int32 = 1000) : Generator(Int32)
      Generator(Int32).new { |rng| rng.rand(min..max) }
    end

    def self.positive_int(max : Int32 = 1000) : Generator(Int32)
      Generator(Int32).new { |rng| rng.rand(1..max) }
    end

    def self.float(min : Float64 = -1000.0, max : Float64 = 1000.0) : Generator(Float64)
      Generator(Float64).new { |rng| rng.rand * (max - min) + min }
    end

    def self.string(max_length : Int32 = 20) : Generator(String)
      Generator(String).new do |rng|
        length = rng.rand(0..max_length)
        String.build(length) do |s|
          length.times { s << (rng.rand(32..127)).chr }
        end
      end
    end

    def self.alpha_string(max_length : Int32 = 20) : Generator(String)
      chars = ('a'..'z').to_a + ('A'..'Z').to_a
      Generator(String).new do |rng|
        length = rng.rand(1..max_length)
        String.build(length) do |s|
          length.times { s << chars[rng.rand(chars.size)] }
        end
      end
    end

    def self.bool : Generator(Bool)
      Generator(Bool).new { |rng| rng.rand < 0.5 }
    end

    def self.array_of(gen : Generator(T), max_size : Int32 = 20) forall T : Generator(Array(T))
      Generator(Array(T)).new do |rng|
        size = rng.rand(0..max_size)
        Array(T).new(size) { gen.sample(rng) }
      end
    end

    def self.non_empty_array_of(gen : Generator(T), max_size : Int32 = 20) forall T : Generator(Array(T))
      Generator(Array(T)).new do |rng|
        size = rng.rand(1..max_size)
        Array(T).new(size) { gen.sample(rng) }
      end
    end

    def self.one_of(*values : T) forall T : Generator(T)
      Generator(T).new { |rng| values[rng.rand(values.size)] }
    end
  end

  # Property checker
  class PropertyChecker
    def self.check(
      description : String,
      runs : Int32 = 100,
      seed : UInt64? = nil,
      &property : Random -> Bool
    )
      rng = seed ? Random.new(seed) : Random.new

      runs.times do |i|
        unless property.call(rng)
          raise "Property '#{description}' failed on run #{i + 1}"
        end
      end

      puts "✓ #{description} (#{runs} runs)"
    end
  end
end
```

## Invariant Testing

```crystal
require "./property_testing/generator"

include PropertyTesting

# Property 1: Sort invariants
describe "Array#sort properties" do
  it "ความยาวไม่เปลี่ยนหลัง sort" do
    PropertyChecker.check("sort preserves length", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      arr.sort.size == arr.size
    end
  end

  it "เรียงลำดับจากน้อยไปมาก" do
    PropertyChecker.check("sort produces ordered array", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      sorted = arr.sort
      # ตรวจสอบว่าแต่ละ element น้อยกว่าหรือเท่ากับ element ถัดไป
      (0...sorted.size - 1).all? { |i| sorted[i] <= sorted[i + 1] }
    end
  end

  it "elements เหมือนกันหลัง sort" do
    PropertyChecker.check("sort contains same elements", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      sorted = arr.sort
      arr.tally == sorted.tally
    end
  end

  it "sort idempotent: sort(sort(x)) == sort(x)" do
    PropertyChecker.check("sort is idempotent", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      arr.sort == arr.sort.sort
    end
  end
end

# Property 2: Reverse invariants
describe "Array#reverse properties" do
  it "reverse reverse = original" do
    PropertyChecker.check("reverse is involution", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      arr.reverse.reverse == arr
    end
  end

  it "ความยาวไม่เปลี่ยน" do
    PropertyChecker.check("reverse preserves length", runs: 1000) do |rng|
      arr = Gen.array_of(Gen.int).sample(rng)
      arr.reverse.size == arr.size
    end
  end
end
```

## Property Testing สำหรับ Math Operations

```crystal
# Properties ของ arithmetic
describe "Math properties" do
  it "commutativity: a + b = b + a" do
    PropertyChecker.check("addition is commutative", runs: 1000) do |rng|
      a = Gen.int.sample(rng)
      b = Gen.int.sample(rng)
      a + b == b + a
    end
  end

  it "associativity: (a + b) + c = a + (b + c)" do
    PropertyChecker.check("addition is associative", runs: 1000) do |rng|
      a = Gen.int(-100, 100).sample(rng)
      b = Gen.int(-100, 100).sample(rng)
      c = Gen.int(-100, 100).sample(rng)
      (a + b) + c == a + (b + c)
    end
  end

  it "identity: a * 1 = a" do
    PropertyChecker.check("multiplication identity", runs: 1000) do |rng|
      a = Gen.int.sample(rng)
      a * 1 == a
    end
  end

  it "กฎของการหาร: (a * b) / b = a เมื่อ b != 0" do
    PropertyChecker.check("division inverse of multiplication", runs: 1000) do |rng|
      a = Gen.int.sample(rng).to_f64
      b = Gen.int(-100, 100).filter { |x| x != 0 }.sample(rng).to_f64
      ((a * b) / b - a).abs < 0.0001
    end
  end
end
```

## Property Testing สำหรับ String Operations

```crystal
describe "String properties" do
  it "upcase แล้ว downcase คืนค่า original ASCII" do
    PropertyChecker.check("downcase(upcase(s)) == downcase(s)", runs: 1000) do |rng|
      s = Gen.alpha_string.sample(rng)
      s.downcase.upcase.downcase == s.downcase
    end
  end

  it "length ของ concatenation = sum of lengths" do
    PropertyChecker.check("concat preserves length", runs: 1000) do |rng|
      a = Gen.string.sample(rng)
      b = Gen.string.sample(rng)
      (a + b).size == a.size + b.size
    end
  end

  it "reverse ของ reverse = original" do
    PropertyChecker.check("string reverse is involution", runs: 1000) do |rng|
      s = Gen.string.sample(rng)
      s.reverse.reverse == s
    end
  end

  it "split แล้ว join คืนค่า original" do
    PropertyChecker.check("split and join are inverse", runs: 1000) do |rng|
      words = Gen.array_of(Gen.alpha_string(10), max_size: 10).sample(rng)
      separator = ","
      joined = words.join(separator)
      joined.split(separator) == words
    end
  end
end
```

## Shrinking (หา minimal failing case)

```crystal
# Shrinking: เมื่อ property ล้มเหลว หา input ที่เล็กที่สุดที่ทำให้ล้มเหลว
class Shrinker(T)
  def shrink(value : T, fails? : T -> Bool) : T
    current = value
    loop do
      candidates = shrink_candidates(current)
      smaller = candidates.find { |c| fails?.call(c) }
      break unless smaller
      current = smaller
    end
    current
  end

  private def shrink_candidates(value : T) : Array(T)
    [] of T
  end
end

class IntShrinker < Shrinker(Int32)
  private def shrink_candidates(n : Int32) : Array(Int32)
    return [] of Int32 if n == 0
    [
      0,
      n // 2,
      n - 1,
      -n
    ].uniq
  end
end

class ArrayShrinker(T) < Shrinker(Array(T))
  def initialize(@element_shrinker : Shrinker(T))
  end

  private def shrink_candidates(arr : Array(T)) : Array(Array(T))
    return [] of Array(T) if arr.empty?

    candidates = [] of Array(T)

    # ลบ element ทีละตัว
    arr.size.times do |i|
      candidates << arr[0...i] + arr[(i+1)..]
    end

    # Shrink element แต่ละตัว
    arr.each_with_index do |elem, i|
      @element_shrinker.shrink_candidates(elem).each do |shrunk|
        new_arr = arr.dup
        new_arr[i] = shrunk
        candidates << new_arr
      end
    end

    candidates.uniq
  end
end
```

## Stateful Property Testing

```crystal
# ทดสอบ stateful operations เช่น Stack
class StackModel(T)
  property items : Array(T)

  def initialize
    @items = [] of T
  end

  def push(item : T) : self
    @items << item
    self
  end

  def pop : T?
    @items.pop?
  end

  def peek : T?
    @items.last?
  end

  def size : Int32
    @items.size
  end

  def empty? : Bool
    @items.empty?
  end
end

describe "Stack properties" do
  it "push แล้ว pop คืนค่าเดิม (LIFO)" do
    PropertyChecker.check("push then pop returns same value", runs: 1000) do |rng|
      stack = StackModel(Int32).new
      value = Gen.int.sample(rng)
      stack.push(value)
      stack.pop == value
    end
  end

  it "size เพิ่มขึ้น 1 ทุก push" do
    PropertyChecker.check("push increases size by 1", runs: 1000) do |rng|
      stack = StackModel(Int32).new
      n = Gen.int(0, 10).sample(rng)
      values = Gen.array_of(Gen.int, max_size: n).sample(rng)

      values.each { |v| stack.push(v) }
      stack.size == values.size
    end
  end

  it "pop ลด size ลง 1" do
    PropertyChecker.check("pop decreases size by 1", runs: 1000) do |rng|
      stack = StackModel(Int32).new
      values = Gen.non_empty_array_of(Gen.int).sample(rng)
      values.each { |v| stack.push(v) }

      before_size = stack.size
      stack.pop
      stack.size == before_size - 1
    end
  end

  it "push หลาย values แล้ว pop ลำดับถูกต้อง" do
    PropertyChecker.check("LIFO ordering", runs: 1000) do |rng|
      stack = StackModel(Int32).new
      values = Gen.array_of(Gen.int, max_size: 10).sample(rng)
      values.each { |v| stack.push(v) }

      popped = [] of Int32
      while !stack.empty?
        popped << stack.pop.not_nil!
      end

      popped == values.reverse
    end
  end
end
```

## Encoding/Decoding Properties

```crystal
describe "JSON encode/decode properties" do
  struct TestData
    include JSON::Serializable

    property name : String
    property value : Int32
    property active : Bool

    def initialize(@name, @value, @active)
    end

    def ==(other : TestData)
      name == other.name && value == other.value && active == other.active
    end
  end

  it "encode แล้ว decode คืนค่าเดิม" do
    PropertyChecker.check("JSON roundtrip", runs: 500) do |rng|
      name = Gen.alpha_string(10).sample(rng)
      value = Gen.int.sample(rng)
      active = Gen.bool.sample(rng)

      data = TestData.new(name, value, active)
      decoded = TestData.from_json(data.to_json)

      decoded == data
    end
  end
end

describe "Base64 encode/decode" do
  it "encode แล้ว decode คืนค่าเดิม" do
    PropertyChecker.check("Base64 roundtrip", runs: 1000) do |rng|
      data = Gen.string.sample(rng)
      Base64.strict_decode64(Base64.strict_encode64(data)) == data
    end
  end
end
```

## แบบฝึกหัด

1. เขียน property tests สำหรับ `Set` operations: union, intersection, difference
2. ทดสอบ `Compress.zlib` ว่า compress แล้ว decompress คืนค่าเดิม
3. สร้าง generator สำหรับ valid email addresses แล้วทดสอบ email validator
4. เขียน stateful property tests สำหรับ `Queue` (FIFO)

## สรุป

Property-based Testing ใน Crystal:
- **Properties/Invariants**: กำหนดสิ่งที่ต้องเป็นจริงเสมอ
- **Random Generation**: สุ่ม inputs เพื่อหา edge cases อัตโนมัติ
- **Shrinking**: หา minimal failing case ที่อ่านเข้าใจง่าย
- **Fuzzing**: ทดสอบด้วย inputs หลากหลายโดยไม่ต้องคิดเอง

Property testing เสริม example-based testing ด้วยการ explore input space ที่กว้างกว่า
