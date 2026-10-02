# Part 006: Boolean และตัวดำเนินการ (Operators)

## Boolean Type

```crystal
# Boolean values
true_val = true
false_val = false

puts true.class   # Bool
puts false.class  # Bool

# Truthy and Falsy - สำคัญมาก!
# Crystal: ONLY nil and false are falsy
# Everything else is truthy (including 0, "", [])

puts "=== Truthy/Falsy ==="

# Falsy values
[false, nil].each do |v|
  puts "#{v.inspect} is #{v ? "truthy" : "falsy"}"
end

# Truthy values (ต่างจาก JavaScript/Python!)
[0, "", [], {}, 0.0].each do |v|
  puts "#{v.inspect} is #{v ? "truthy" : "falsy"}"
end
# 0 is truthy!
# "" is truthy!
# [] is truthy!
```

## Comparison Operators

```crystal
# Equality
puts 5 == 5     # true
puts 5 == 5.0   # true (numeric equality)
puts "5" == 5   # false (different types)
puts :foo == :foo  # true (symbols)

# Inequality
puts 5 != 6     # true
puts 5 != 5     # false

# Relational
puts 5 > 3      # true
puts 5 < 3      # false
puts 5 >= 5     # true
puts 5 <= 4     # false

# Three-way comparison (spaceship operator)
puts 5 <=> 3    # 1 (left > right)
puts 3 <=> 5    # -1 (left < right)
puts 5 <=> 5    # 0 (equal)

# Used for sorting
arr = [3, 1, 4, 1, 5, 9, 2, 6]
arr.sort { |a, b| a <=> b }   # ascending
arr.sort { |a, b| b <=> a }   # descending

# Object identity
a = "hello"
b = "hello"
c = a

puts a == b       # true (value equality)
puts a.same?(b)   # false (different objects)
puts a.same?(c)   # true (same object)
```

## Logical Operators

```crystal
# Boolean AND
puts true && true    # true
puts true && false   # false
puts false && true   # false
puts false && false  # false

# Boolean OR
puts true || true    # true
puts true || false   # true
puts false || true   # true
puts false || false  # false

# Boolean NOT
puts !true           # false
puts !false          # true
puts !!nil           # false (double negation converts to bool)
puts !!42            # true

# Word operators (lower precedence)
puts true and false  # true (but confusing - use && instead)
puts false or true   # false (but confusing - use || instead)
puts not true        # false (but use ! instead)

# Short-circuit evaluation
def side_effect(val)
  puts "Called with #{val}"
  val
end

# && stops at first false
side_effect(false) && side_effect(true)
# Prints: "Called with false" (side_effect(true) NOT called)

# || stops at first true
side_effect(true) || side_effect(false)
# Prints: "Called with true" (side_effect(false) NOT called)

# Practical uses
user = nil
# Safe - won't call methods on nil
if user && user.active?
  puts "User is active"
end

# Default value pattern
name = nil
display_name = name || "Anonymous"
puts display_name  # "Anonymous"
```

## Nil-related Operators

```crystal
# Nil check
x = nil
puts x.nil?     # true
puts x.is_a?(Nil)  # true

# Nil coalescing (not a built-in operator, use ||)
value = nil
result = value || "default"  # "default"

# But be careful with falsy nil vs falsy false
flag = false
result = flag || "default"  # "default" (wrong if flag is intentionally false!)

# Better: use nil? explicitly
result = flag.nil? ? "default" : flag

# Safe navigation operator (&.)
user = nil
puts user&.name    # nil (no error)
puts user&.name&.upcase  # nil (chaining)

user2 = {name: "Alice"}
# user2&.name  # Would work for object with .name method

# not_nil! - assert not nil (raises if nil)
value = find_value()
puts value.not_nil!  # Raises NilAssertionError if nil
```

## Bitwise Operators

```crystal
# Bitwise AND
puts 0b1010 & 0b1100   # 0b1000 = 8
puts 5 & 3              # 1

# Bitwise OR
puts 0b1010 | 0b1100   # 0b1110 = 14
puts 5 | 3              # 7

# Bitwise XOR
puts 0b1010 ^ 0b1100   # 0b0110 = 6
puts 5 ^ 3              # 6

# Bitwise NOT
puts ~0b1010            # -11 (two's complement)
puts ~5                 # -6

# Left shift
puts 1 << 4             # 16 (1 * 2^4)
puts 5 << 2             # 20 (5 * 2^2)

# Right shift
puts 16 >> 2            # 4 (16 / 2^2)
puts 20 >> 2            # 5

# Practical: Bitflags
PERMISSION_READ    = 0b001  # 1
PERMISSION_WRITE   = 0b010  # 2
PERMISSION_EXECUTE = 0b100  # 4

# Set permissions
user_perms = PERMISSION_READ | PERMISSION_WRITE  # 3

# Check permission
puts (user_perms & PERMISSION_READ) != 0    # true (can read)
puts (user_perms & PERMISSION_WRITE) != 0   # true (can write)
puts (user_perms & PERMISSION_EXECUTE) != 0 # false (can't execute)

# Toggle permission
user_perms ^= PERMISSION_WRITE  # Remove write
puts (user_perms & PERMISSION_WRITE) != 0  # false

# Byte manipulation
byte = 0b10110101
high_nibble = (byte >> 4) & 0xF  # 0b1011 = 11
low_nibble = byte & 0xF           # 0b0101 = 5
puts "High: #{high_nibble}, Low: #{low_nibble}"
```

## Assignment Operators

```crystal
x = 10

# Compound assignment
x += 5    # x = x + 5 = 15
x -= 3    # x = x - 3 = 12
x *= 2    # x = x * 2 = 24
x /= 4    # x = x / 4 = 6
x //= 2   # x = x // 2 = 3
x %= 2    # x = x % 2 = 1
x **= 3   # x = x ** 3 = 1

# Bitwise compound
y = 0b1010
y &= 0b1100  # y = 0b1000
y |= 0b0001  # y = 0b1001
y ^= 0b1111  # y = 0b0110
y <<= 2      # y = 0b11000
y >>= 1      # y = 0b1100

# Conditional assignment
a = nil
a ||= "default"   # a = "default" (assigned because nil)
a ||= "other"     # a = "default" (NOT reassigned - already truthy)
puts a            # "default"

b = false
b ||= "default"   # b = "default" (assigned because false is falsy)
puts b            # "default"

# AND assignment
c = "value"
c &&= c.upcase  # c = "VALUE" (only if c is truthy)
puts c          # "VALUE"

d = nil
d &&= "something"  # d remains nil (not truthy)
puts d.inspect     # nil
```

## Ternary Operator

```crystal
# condition ? true_value : false_value
age = 20
status = age >= 18 ? "adult" : "minor"
puts status  # "adult"

# Nested ternary (avoid for readability)
score = 75
grade = score >= 90 ? "A" : score >= 80 ? "B" : score >= 70 ? "C" : score >= 60 ? "D" : "F"
puts grade  # "C"

# Better: use case/when for multiple conditions
grade = case score
        when 90.. then "A"
        when 80..89 then "B"
        when 70..79 then "C"
        when 60..69 then "D"
        else "F"
        end

# Ternary in string interpolation
n = 5
puts "#{n} is #{n.even? ? "even" : "odd"}"
```

## Operator Precedence

```crystal
# Precedence (highest to lowest):
# 1. Method calls, [], []=
# 2. !, ~, + (unary), - (unary)
# 3. **, (right associative)
# 4. *, /, //, %
# 5. +, -
# 6. <<, >>
# 7. &
# 8. ^
# 9. |
# 10. <=, <, >, >=
# 11. ==, !=, =~, !~, ===, <=>
# 12. &&
# 13. ||
# 14. .., ...
# 15. ?, : (ternary)
# 16. =, op=
# 17. not
# 18. or, and

# Examples
puts 2 + 3 * 4      # 14 (not 20)
puts (2 + 3) * 4    # 20

puts 2 ** 3 ** 2    # 512 (right assoc: 2^(3^2) = 2^9)
puts (2 ** 3) ** 2  # 64

puts true || false && false  # true (&& has higher precedence)
puts (true || false) && false  # false

puts 5 > 3 == true   # true (comparison then equality)
puts (5 > 3) == true # true

# Be explicit with parentheses for clarity
x = 5
puts x > 0 && x < 10  # Readable
puts (x > 0) && (x < 10)  # More explicit
```

## Comparison Operators with Types

```crystal
# Integers vs Floats
puts 5 == 5.0    # true
puts 5 > 4.9     # true
puts 5 < 5.1     # true

# String comparison (lexicographic)
puts "apple" < "banana"   # true
puts "apple" > "Apple"    # true (lowercase > uppercase in ASCII)
puts "a".ord < "A".ord    # false (a=97, A=65)

# Array comparison (element by element)
puts [1, 2, 3] == [1, 2, 3]  # true
puts [1, 2, 3] < [1, 2, 4]   # true
puts [1, 2, 3] > [1, 2]      # true (longer array wins if same prefix)

# Custom comparison with spaceship
class Temperature
  include Comparable(Temperature)
  
  getter celsius : Float64
  
  def initialize(@celsius : Float64)
  end
  
  def <=>(other : Temperature) : Int32
    @celsius <=> other.celsius
  end
  
  def to_s
    "#{@celsius}°C"
  end
end

temps = [Temperature.new(30.0), Temperature.new(20.0), Temperature.new(25.0)]
puts temps.sort.inspect     # 20°C, 25°C, 30°C
puts temps.min              # 20°C
puts temps.max              # 30°C
```

## Regex Operators

```crystal
# Match operator (=~)
if "hello world" =~ /world/
  puts "Match!"
end

# Capture groups with =~
if "2024-01-15" =~ /(\d{4})-(\d{2})-(\d{2})/
  # $~ is the last match
  puts $~[0]  # "2024-01-15"
  puts $~[1]  # "2024"
  puts $~[2]  # "01"
  puts $~[3]  # "15"
end

# Not match (!~)
if "hello" !~ /\d/
  puts "No digits found"
end

# Case equality (===)
case "hello"
when /^h/     # calls /^h/ === "hello"
  puts "Starts with h"
when /\d/
  puts "Contains a digit"
end

# Integer === n (range check)
case 42
when 1..10
  puts "Small"
when 11..100
  puts "Medium"
else
  puts "Large"
end
```

## Practical Examples

```crystal
# Input validation
def valid_email?(email : String) : Bool
  email.matches?(/\A[\w.+\-]+@[a-z\d\-.]+\.[a-z]+\z/i)
end

def valid_phone?(phone : String) : Bool
  phone.matches?(/\A\+?[\d\s\-\(\)]{10,15}\z/)
end

def valid_password?(password : String) : Bool
  password.size >= 8 &&
    password.matches?(/[A-Z]/) &&  # Has uppercase
    password.matches?(/[a-z]/) &&  # Has lowercase
    password.matches?(/\d/)    &&  # Has digit
    password.matches?(/[!@#$%^&*]/)  # Has special char
end

puts valid_email?("alice@example.com")   # true
puts valid_email?("not-an-email")        # false
puts valid_phone?("+1 555-123-4567")     # true
puts valid_password?("MyPass@1")         # true
puts valid_password?("weak")             # false

# Precedence gotcha
x = true
y = false
z = false

puts x || y && z   # true (&& evaluated first: y && z = false; x || false = true)
puts (x || y) && z # false (x || y = true; true && z = false)

# Common pattern: nil-safe method chain
class User
  property name : String?
  property email : String?
  
  def initialize(@name, @email)
  end
end

users = [User.new("Alice", "alice@example.com"), User.new(nil, nil)]

users.each do |user|
  puts user.name&.upcase || "ANONYMOUS"
  puts user.email&.includes?("@") ? "Valid email" : "No/Invalid email"
end
```

---

## สรุป Part 006

ในบทนี้เราได้เรียนรู้:

1. **Boolean**: truthy/falsy (only nil and false are falsy!)
2. **Comparison**: ==, !=, <, >, <=, >=, <=>
3. **Logical**: &&, ||, !, short-circuit evaluation
4. **Nil operators**: &. (safe navigation), nil?, not_nil!
5. **Bitwise**: &, |, ^, ~, <<, >>
6. **Assignment**: =, +=, -=, ||=, &&=
7. **Ternary**: condition ? true : false
8. **Precedence**: * before +, && before ||
9. **Regex**: =~, !~, ===

---

## ขั้นตอนต่อไป

ไปที่ [Part 007](part_007.md) เพื่อเรียนรู้:
- Type system ของ Crystal
- Type inference
- Union types
- Type restrictions
