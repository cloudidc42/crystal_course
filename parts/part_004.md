# Part 004: ตัวเลขและการคำนวณทางคณิตศาสตร์

## Integer Operations ขั้นสูง

```crystal
# Division types
puts 10 / 3      # 3 (Int32 division = truncated)
puts 10.0 / 3    # 3.3333...
puts 10 / 3.0    # 3.3333...
puts 10.0 / 3.0  # 3.3333...

# Floor division
puts 10.fdiv(3)  # 3.3333... (Float division from Int)
puts (-7) / 2    # -4 (floor division for negative in Crystal)
puts (-7) // 2   # -4

# Modulo vs Remainder
puts (-7) % 3    # 2 (modulo - result has sign of divisor)
puts (-7).remainder(3)  # -1 (remainder - result has sign of dividend)

# Power
puts 2 ** 10     # 1024
puts 2 ** 0      # 1
puts 2 ** -1     # 0 (integer power, truncated)
puts 2.0 ** -1   # 0.5

# Overflow behavior
max_i32 = Int32::MAX  # 2147483647
puts max_i32 &+ 1     # -2147483648 (wrapping add)
puts max_i32.to_i64 + 1  # 2147483648 (use larger type)

# Absolute value
puts (-42).abs   # 42
puts 42.abs      # 42

# Min/Max
puts 5.clamp(1, 10)   # 5
puts 15.clamp(1, 10)  # 10
puts (-5).clamp(1, 10) # 1

# Number properties
puts 42.even?    # true
puts 42.odd?     # false
puts 7.zero?     # false
puts 0.zero?     # true
puts 7.positive? # true
puts (-7).negative? # true
puts 7.sign      # 1
puts (-7).sign   # -1
puts 0.sign      # 0
```

## BigInt - Integer ขนาดไม่จำกัด

```crystal
require "big"

# BigInt สำหรับตัวเลขขนาดใหญ่มาก
big = BigInt.new(10) ** 100  # 10^100 (1 googol)
puts big

# Fibonacci numbers ขนาดใหญ่
def fibonacci_big(n : Int32) : BigInt
  a, b = BigInt.new(0), BigInt.new(1)
  n.times { a, b = b, a + b }
  a
end

puts fibonacci_big(100)   # 354224848179261915075
puts fibonacci_big(200)   # ตัวเลขขนาดใหญ่มาก!
puts fibonacci_big(1000)  # ตัวเลขมหึมา!

# Factorial
def factorial(n : Int32) : BigInt
  return BigInt.new(1) if n <= 1
  result = BigInt.new(1)
  (2..n).each { |i| result *= i }
  result
end

puts factorial(20)   # 2432902008176640000
puts factorial(100)  # ตัวเลขขนาดใหญ่มาก!

# BigInt operations
a = BigInt.new("123456789012345678901234567890")
b = BigInt.new("987654321098765432109876543210")
puts a + b
puts a * b
puts b / a  # BigInt division
puts b % a  # BigInt modulo
puts a.gcd(b)  # GCD of big numbers
```

## BigDecimal - Decimal ความแม่นยำสูง

```crystal
require "big"

# Float64 มีปัญหา floating point precision
puts 0.1 + 0.2         # 0.30000000000000004 (!)
puts 0.1 + 0.2 == 0.3  # false (!)

# BigDecimal แก้ปัญหานี้
d1 = BigDecimal.new("0.1")
d2 = BigDecimal.new("0.2")
puts d1 + d2           # 0.3
puts d1 + d2 == BigDecimal.new("0.3")  # true

# Financial calculations - ต้องใช้ BigDecimal
price = BigDecimal.new("19.99")
quantity = BigDecimal.new("3")
tax_rate = BigDecimal.new("0.07")

subtotal = price * quantity
tax = subtotal * tax_rate
total = subtotal + tax

puts "Subtotal: #{subtotal}"  # 59.97
puts "Tax: #{tax}"            # 4.1979
puts "Total: #{total}"        # 64.1679

# Rounding
puts tax.round(2)   # 4.20
puts total.round(2) # 64.17

# Precision
result = BigDecimal.new("1") / BigDecimal.new("3")
puts result               # ยาวมาก
puts result.round(10)     # 0.3333333333

# BigDecimal from Float
d = BigDecimal.new(3.14)      # อาจไม่แม่นยำ
d2 = BigDecimal.new("3.14")   # แม่นยำกว่า
```

## Math Library

```crystal
require "math"

# Basic functions
puts Math.sqrt(144.0)     # 12.0
puts Math.cbrt(27.0)      # 3.0
puts Math.abs(-5.0)       # 5.0

# Powers and logarithms
puts Math.exp(1.0)        # 2.718... (e^1)
puts Math.exp2(8.0)       # 256.0 (2^8)
puts Math.exp10(3.0)      # 1000.0 (10^3)
puts Math.log(Math::E)    # 1.0 (natural log)
puts Math.log2(256.0)     # 8.0
puts Math.log10(1000.0)   # 3.0
puts Math.pow(2.0, 8.0)   # 256.0

# Trigonometric functions (in radians)
puts Math.sin(Math::PI / 2)  # 1.0
puts Math.cos(0.0)            # 1.0
puts Math.tan(Math::PI / 4)  # ~1.0
puts Math.asin(1.0)          # PI/2
puts Math.acos(1.0)          # 0.0
puts Math.atan(1.0)          # PI/4
puts Math.atan2(1.0, 1.0)    # PI/4

# Hyperbolic functions
puts Math.sinh(1.0)   # 1.1752...
puts Math.cosh(0.0)   # 1.0
puts Math.tanh(0.0)   # 0.0

# Rounding
puts Math.floor(3.7)  # 3.0
puts Math.ceil(3.2)   # 4.0
puts Math.round(3.5)  # 4.0

# Constants
puts Math::PI    # 3.141592653589793
puts Math::E     # 2.718281828459045
puts Math::TAU   # 6.283185307179586 (2*PI)
puts Math::LOG2  # 0.6931471805599453
puts Math::LOG10 # 2.302585092994046
puts Math::SQRT2 # 1.4142135623730951

# Geometry calculations
def circle_area(radius : Float64) : Float64
  Math::PI * radius ** 2
end

def circle_circumference(radius : Float64) : Float64
  2 * Math::PI * radius
end

def sphere_volume(radius : Float64) : Float64
  (4.0 / 3.0) * Math::PI * radius ** 3
end

puts "Circle (r=5):"
puts "  Area: #{circle_area(5.0).round(4)}"
puts "  Circumference: #{circle_circumference(5.0).round(4)}"
puts "Sphere (r=5):"
puts "  Volume: #{sphere_volume(5.0).round(4)}"
```

## Random Numbers

```crystal
# Random module
rng = Random.new  # New random number generator

# Random integers
puts Random.rand(10)       # 0 to 9
puts Random.rand(1..10)    # 1 to 10 (inclusive)
puts Random.rand(1...10)   # 1 to 9 (exclusive end)

# Random floats
puts Random.rand       # 0.0 to 1.0
puts Random.rand(1.0)  # 0.0 to 1.0

# With seed (reproducible)
seeded = Random.new(42)
puts seeded.rand(100)  # Always same value with same seed
puts seeded.rand(100)  # Next value in sequence

# Using instance
rng = Random.new
puts rng.rand(100)
puts rng.rand(100)
puts rng.rand(100)

# Random from array
fruits = ["apple", "banana", "cherry", "date"]
puts fruits.sample          # Random element
puts fruits.sample(2)       # 2 random elements (no duplicates)
puts fruits.shuffle         # Shuffled array
puts fruits.shuffle.first   # Random element (same as sample)

# Secure random
puts Random::Secure.hex(16)     # Secure random hex string
puts Random::Secure.base64(16)  # Secure random base64 string
puts Random::Secure.rand(100)   # Secure random integer

# UUID generation
require "uuid"
puts UUID.random.to_s   # e.g., "550e8400-e29b-41d4-a716-446655440000"
```

## Number Formatting

```crystal
# Basic formatting
puts "%.2f" % 3.14159    # 3.14
puts "%08.2f" % 3.14     # 00003.14
puts "%+.2f" % 3.14      # +3.14
puts "%-10s|" % "hello"  # "hello     |" (left-align)

# sprintf (safe version of printf)
formatted = sprintf("%.4f", Math::PI)
puts formatted  # 3.1416

# Number to string with format
n = 1234567
puts n.to_s                    # "1234567"
puts "%d" % n                  # "1234567"
puts n.format(",")             # Hmm, Crystal doesn't have this natively

# Custom number formatting
def format_number(n : Number) : String
  n.to_s.reverse.chars.each_slice(3).map(&.join).join(",").reverse
end

puts format_number(1234567)    # 1,234,567
puts format_number(1234567890) # 1,234,567,890

# Currency formatting
def format_currency(amount : Float64, symbol = "$") : String
  formatted = "%.2f" % amount
  int_part, dec_part = formatted.split(".")
  int_formatted = int_part.reverse.chars.each_slice(3).map(&.join).join(",").reverse
  "#{symbol}#{int_formatted}.#{dec_part}"
end

puts format_currency(1234567.89)   # $1,234,567.89
puts format_currency(0.99, "€")    # €0.99
```

## Statistics Module

```crystal
# Basic statistics
data = [4, 7, 13, 2, 1, 8, 5, 9, 3, 6]

# Min/Max
puts data.min    # 1
puts data.max    # 13
puts data.minmax # {1, 13}

# Sum and average
puts data.sum                                # 58
puts data.sum / data.size.to_f               # 5.8 (mean)
puts data.sum.to_f / data.size               # 5.8

# Sorted
sorted = data.sort
puts sorted.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9, 13]

# Median
def median(arr : Array(Number)) : Float64
  sorted = arr.sort
  n = sorted.size
  if n.odd?
    sorted[n / 2].to_f
  else
    (sorted[n/2 - 1] + sorted[n/2]).to_f / 2
  end
end

puts median(data)  # 5.5

# Mode
def mode(arr : Array) : Array
  freq = Hash(typeof(arr.first), Int32).new(0)
  arr.each { |x| freq[x] += 1 }
  max_freq = freq.values.max
  freq.select { |k, v| v == max_freq }.keys
end

puts mode([1, 2, 2, 3, 3, 3, 4]).inspect  # [3]

# Standard deviation
def std_dev(arr : Array(Float64)) : Float64
  mean = arr.sum / arr.size
  variance = arr.map { |x| (x - mean) ** 2 }.sum / arr.size
  Math.sqrt(variance)
end

float_data = data.map(&.to_f)
puts "Mean: #{float_data.sum / float_data.size}"
puts "Std Dev: #{std_dev(float_data).round(4)}"

# Percentile
def percentile(arr : Array, p : Float64) : Float64
  sorted = arr.sort.map(&.to_f)
  idx = (p / 100.0) * (sorted.size - 1)
  lower = sorted[idx.floor.to_i]
  upper = sorted[idx.ceil.to_i]
  lower + (upper - lower) * (idx - idx.floor)
end

puts "25th percentile: #{percentile(data, 25.0)}"
puts "75th percentile: #{percentile(data, 75.0)}"
puts "Median (50th): #{percentile(data, 50.0)}"
```

## ตัวอย่าง: เครื่องคิดเลขทางการเงิน

```crystal
require "big"
require "math"

# Compound Interest Calculator
# A = P(1 + r/n)^(nt)
def compound_interest(
  principal : Float64,
  annual_rate : Float64,  # as decimal, e.g., 0.05 for 5%
  years : Int32,
  compounds_per_year : Int32 = 12
) : Float64
  principal * (1 + annual_rate / compounds_per_year) ** (compounds_per_year * years)
end

puts "=== Compound Interest Calculator ==="
principal = 10000.0
rate = 0.05  # 5%

[1, 5, 10, 20, 30].each do |years|
  amount = compound_interest(principal, rate, years)
  interest = amount - principal
  puts "#{years} years: $#{amount.round(2)} (interest: $#{interest.round(2)})"
end

# Loan Payment Calculator
# M = P[r(1+r)^n]/[(1+r)^n - 1]
def monthly_payment(
  loan : Float64,
  annual_rate : Float64,
  years : Int32
) : Float64
  monthly_rate = annual_rate / 12
  n_payments = years * 12
  
  if monthly_rate == 0
    return loan / n_payments
  end
  
  loan * (monthly_rate * (1 + monthly_rate) ** n_payments) / 
        ((1 + monthly_rate) ** n_payments - 1)
end

puts "\n=== Loan Calculator ==="
loan = 200000.0
rate = 0.035  # 3.5%

[10, 15, 20, 30].each do |years|
  payment = monthly_payment(loan, rate, years)
  total_paid = payment * years * 12
  total_interest = total_paid - loan
  
  puts "#{years}-year loan at #{rate*100}%:"
  puts "  Monthly payment: $#{payment.round(2)}"
  puts "  Total paid: $#{total_paid.round(2)}"
  puts "  Total interest: $#{total_interest.round(2)}"
  puts ""
end

# Investment Portfolio Returns
puts "=== Portfolio Returns ==="
investments = [
  {name: "Stocks", amount: 50000.0, annual_return: 0.10},
  {name: "Bonds",  amount: 30000.0, annual_return: 0.04},
  {name: "Real Estate", amount: 20000.0, annual_return: 0.07},
]

puts "After 10 years:"
total_start = 0.0
total_end = 0.0

investments.each do |inv|
  future = inv[:amount] * (1 + inv[:annual_return]) ** 10
  puts "  #{inv[:name]}: $#{inv[:amount].to_i} -> $#{future.round(2)}"
  total_start += inv[:amount]
  total_end += future
end

puts "  Total: $#{total_start.to_i} -> $#{total_end.round(2)}"
puts "  Return: #{(((total_end - total_start) / total_start) * 100).round(2)}%"
```

## ตัวอย่าง: Prime Numbers

```crystal
# Prime number algorithms

# Simple trial division
def prime?(n : Int32) : Bool
  return false if n < 2
  return true if n == 2
  return false if n.even?
  
  i = 3
  while i * i <= n
    return false if n % i == 0
    i += 2
  end
  true
end

# Sieve of Eratosthenes
def sieve(limit : Int32) : Array(Int32)
  is_prime = Array.new(limit + 1, true)
  is_prime[0] = is_prime[1] = false
  
  i = 2
  while i * i <= limit
    if is_prime[i]
      (i*i..limit).step(i) { |j| is_prime[j] = false }
    end
    i += 1
  end
  
  (2..limit).select { |n| is_prime[n] }.to_a
end

# Usage
primes = sieve(100)
puts "Primes up to 100: #{primes.inspect}"
puts "Count: #{primes.size}"

# Prime factorization
def prime_factors(n : Int32) : Array(Int32)
  factors = [] of Int32
  d = 2
  
  while d * d <= n
    while n % d == 0
      factors << d
      n //= d
    end
    d += 1
  end
  
  factors << n if n > 1
  factors
end

puts "\nPrime factorizations:"
[12, 36, 100, 360, 1000].each do |n|
  factors = prime_factors(n)
  puts "#{n} = #{factors.join(" × ")}"
end

# GCD and LCM using prime factorization
puts "\nGCD and LCM:"
pairs = [{12, 18}, {100, 75}, {24, 36}]
pairs.each do |a, b|
  g = a.gcd(b)
  l = a.lcm(b)
  puts "gcd(#{a}, #{b}) = #{g}, lcm(#{a}, #{b}) = #{l}"
end
```

## ตัวอย่าง: Matrix Operations

```crystal
# Basic matrix operations

alias Matrix = Array(Array(Float64))

def matrix_add(a : Matrix, b : Matrix) : Matrix
  a.map_with_index { |row, i| row.map_with_index { |val, j| val + b[i][j] } }
end

def matrix_multiply(a : Matrix, b : Matrix) : Matrix
  rows = a.size
  cols = b[0].size
  inner = b.size
  
  Matrix.new(rows) { |i| Array.new(cols) { |j|
    (0...inner).sum { |k| a[i][k] * b[k][j] }
  }}
end

def matrix_transpose(m : Matrix) : Matrix
  return Matrix.new if m.empty?
  m[0].map_with_index { |_, j| m.map { |row| row[j] } }
end

def matrix_print(m : Matrix)
  m.each do |row|
    puts row.map { |x| "%6.2f" % x }.join(" ")
  end
end

# Example usage
a : Matrix = [[1.0, 2.0], [3.0, 4.0]]
b : Matrix = [[5.0, 6.0], [7.0, 8.0]]

puts "Matrix A:"
matrix_print(a)

puts "\nMatrix B:"
matrix_print(b)

puts "\nA + B:"
matrix_print(matrix_add(a, b))

puts "\nA × B:"
matrix_print(matrix_multiply(a, b))

puts "\nTranspose of A:"
matrix_print(matrix_transpose(a))

# Determinant (2x2)
def determinant_2x2(m : Matrix) : Float64
  m[0][0] * m[1][1] - m[0][1] * m[1][0]
end

puts "\nDeterminant of A: #{determinant_2x2(a)}"
```

---

## สรุป Part 004

ในบทนี้เราได้เรียนรู้:

1. **Integer Operations**: division, modulo, bitwise, overflow
2. **BigInt**: ตัวเลขขนาดใหญ่ไม่จำกัด
3. **BigDecimal**: ความแม่นยำสูงสำหรับการเงิน
4. **Math Library**: sqrt, log, trig functions, constants
5. **Random Numbers**: rand, sample, shuffle, secure random
6. **Number Formatting**: printf, sprintf, custom formatters
7. **Statistics**: min, max, mean, median, std dev
8. **Applications**: compound interest, loans, primes, matrices

---

## แบบฝึกหัด

```crystal
# 1. คำนวณ BMI
# BMI = weight(kg) / height(m)^2
# Underweight: < 18.5
# Normal: 18.5 - 24.9
# Overweight: 25.0 - 29.9
# Obese: >= 30.0

# 2. Binary to Decimal converter
# รับ binary string แล้วแปลงเป็น decimal โดยไม่ใช้ to_i(2)

# 3. Pascal's Triangle
# แสดง n แถวแรกของ Pascal's Triangle

# 4. Euler's Number
# คำนวณ e ด้วย series: e = 1 + 1/1! + 1/2! + 1/3! + ...
# จนกว่าความแม่นยำจะถึง 1e-15
```

---

## ขั้นตอนต่อไป

ไปที่ [Part 005](part_005.md) เพื่อเรียนรู้:
- String methods ขั้นสูง
- String manipulation
- Regular expressions พื้นฐาน
- String encoding
