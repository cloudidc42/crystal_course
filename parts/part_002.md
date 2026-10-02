# Part 002: Hello World และโครงสร้างพื้นฐาน

## โปรแกรม Hello World

```crystal
# hello.cr
puts "Hello, World!"
```

```bash
crystal run hello.cr
# Output: Hello, World!
```

นี่คือโปรแกรม Crystal ที่ง่ายที่สุด - เพียงหนึ่งบรรทัด!

---

## การแสดงผลข้อมูล (Output Methods)

Crystal มีหลายวิธีในการแสดงผล:

### puts - Print with newline
```crystal
puts "Hello, World!"    # Hello, World!\n
puts 42                 # 42\n
puts 3.14               # 3.14\n
puts true               # true\n
puts nil                # (blank line)\n

# หลายค่า
puts "First", "Second", "Third"
# First
# Second
# Third

# Array
puts [1, 2, 3]
# 1
# 2
# 3
```

### print - Print without newline
```crystal
print "Hello"
print ", "
print "World"
print "!\n"
# Output: Hello, World!

# print หลายค่า
print "A", "B", "C"
# Output: ABC (ไม่มี newline ต่อท้าย)
```

### p - Print with type info (for debugging)
```crystal
p "Hello"        # "Hello"
p 42             # 42
p 3.14           # 3.14
p true           # true
p nil            # nil
p [1, 2, 3]      # [1, 2, 3]
p({a: 1, b: 2})  # {a: 1, b: 2}

# p แสดงผลแบบที่เป็นประโยชน์สำหรับ debugging
name = "Alice"
p name   # "Alice" (มี quotes แสดงว่าเป็น String)
```

### pp - Pretty print
```crystal
data = {name: "Alice", age: 25, hobbies: ["reading", "coding"]}
pp data
# {name: "Alice", age: 25, hobbies: ["reading", "coding"]}

# สำหรับ nested structures ที่ซับซ้อน
complex = {
  users: [
    {id: 1, name: "Alice"},
    {id: 2, name: "Bob"}
  ],
  total: 2
}
pp complex
```

### printf - C-style formatting
```crystal
printf("Name: %s, Age: %d\n", "Alice", 25)
# Name: Alice, Age: 25

printf("%.2f\n", 3.14159)
# 3.14

# Format specifiers
# %s - String
# %d - Integer
# %f - Float
# %e - Scientific notation
# %g - Shorter of %e or %f
# %b - Binary
# %o - Octal
# %x - Hexadecimal (lowercase)
# %X - Hexadecimal (uppercase)
```

### STDOUT และ STDERR
```crystal
# เขียนไปยัง STDOUT
STDOUT.puts "This is stdout"
STDOUT.print "No newline"
STDOUT.flush  # Flush buffer

# เขียนไปยัง STDERR
STDERR.puts "This is an error message"
STDERR.print "Error details"

# ตัวอย่างการใช้งาน
begin
  # code ที่อาจ error
  raise "Something went wrong"
rescue e
  STDERR.puts "Error: #{e.message}"
  exit 1
end
```

---

## String Interpolation

การแทรกค่าตัวแปรในข้อความ:

```crystal
name = "Alice"
age = 25

# String interpolation ด้วย #{}
puts "My name is #{name} and I am #{age} years old."
# Output: My name is Alice and I am 25 years old.

# นิพจน์ใดก็ได้ใน #{}
puts "Next year I will be #{age + 1} years old."
# Output: Next year I will be 26 years old.

# Method calls
puts "UPPERCASE: #{name.upcase}"
# Output: UPPERCASE: ALICE

# Complex expressions
numbers = [1, 2, 3, 4, 5]
puts "Sum: #{numbers.sum}, Average: #{numbers.sum / numbers.size.to_f}"
# Output: Sum: 15, Average: 3.0

# Multi-line string
message = "Hello #{name}!
How are you doing?
Your age is #{age}."
puts message
```

---

## Comments (ความคิดเห็น)

```crystal
# นี่คือ single-line comment

# Crystal ใช้ # สำหรับ comment เหมือน Ruby และ Python

puts "Hello"  # comment ต่อท้าย code ก็ได้

=begin
นี่คือ multi-line comment
สามารถเขียนข้อความยาวๆ ได้
Crystal รองรับ =begin และ =end
=end

# หรือใช้หลาย #
# บรรทัดที่ 1
# บรรทัดที่ 2
# บรรทัดที่ 3

# Documentation comments ใช้ ##
## This method greets a person
## Returns a greeting string
def greet(name : String) : String
  "Hello, #{name}!"
end
```

---

## โครงสร้างพื้นฐานของ Crystal Program

### ไฟล์เดียว (Single File)
```crystal
# simple.cr

# 1. require statements (ถ้าต้องการ library)
require "json"
require "http/client"

# 2. Constants
VERSION = "1.0.0"
MAX_SIZE = 100

# 3. Type definitions
struct Point
  getter x : Float64
  getter y : Float64
  
  def initialize(@x : Float64, @y : Float64)
  end
end

# 4. Module/Class definitions
module Utils
  def self.distance(p1 : Point, p2 : Point) : Float64
    Math.sqrt((p2.x - p1.x) ** 2 + (p2.y - p1.y) ** 2)
  end
end

# 5. Main code
p1 = Point.new(0.0, 0.0)
p2 = Point.new(3.0, 4.0)
puts "Distance: #{Utils.distance(p1, p2)}"  # Distance: 5.0
```

### โปรแกรมหลายไฟล์ (Multi-file)
```
src/
├── main.cr        # Entry point
├── models/
│   ├── user.cr
│   └── post.cr
└── utils/
    ├── math.cr
    └── string.cr
```

```crystal
# src/main.cr
require "./models/user"
require "./models/post"
require "./utils/math"

# code here...
```

```crystal
# src/models/user.cr
class User
  property name : String
  property email : String
  
  def initialize(@name : String, @email : String)
  end
end
```

---

## การทำงานกับ Input

### อ่านจาก STDIN
```crystal
# อ่านบรรทัดเดียว
print "Enter your name: "
name = gets
puts "Hello, #{name}!"

# gets.chomp - ตัด newline ออก
print "Enter your age: "
age_str = gets.not_nil!.chomp
age = age_str.to_i
puts "You are #{age} years old"

# อ่านหลายบรรทัด
puts "Enter lines (Ctrl+D to stop):"
while line = gets
  puts "You entered: #{line}"
end
```

### อ่าน Command Line Arguments
```crystal
# ARGV contains command line arguments
if ARGV.empty?
  puts "No arguments provided"
else
  puts "Arguments:"
  ARGV.each_with_index do |arg, i|
    puts "  #{i}: #{arg}"
  end
end

# Usage: crystal run program.cr -- arg1 arg2 arg3
# หรือ: ./program arg1 arg2 arg3
```

### อ่านจาก Environment Variables
```crystal
# อ่าน environment variable
home = ENV["HOME"]?
puts "Home: #{home || "not set"}"

# ต้องการค่า (raise error ถ้าไม่มี)
path = ENV["PATH"]
puts "PATH: #{path}"

# Default value
debug = ENV.fetch("DEBUG", "false")
puts "Debug mode: #{debug}"
```

---

## โปรแกรมตัวอย่าง: Calculator

```crystal
# calculator.cr

# Simple calculator program

def calculate(a : Float64, op : String, b : Float64) : Float64
  case op
  when "+", "add", "plus"
    a + b
  when "-", "subtract", "minus"
    a - b
  when "*", "multiply", "times"
    a * b
  when "/", "divide"
    if b == 0.0
      raise ArgumentError.new("Division by zero!")
    end
    a / b
  when "**", "power", "pow"
    a ** b
  when "%", "modulo", "mod"
    a % b
  else
    raise ArgumentError.new("Unknown operator: #{op}")
  end
end

# Main program
puts "=== Crystal Calculator ==="
puts "Supported operators: +, -, *, /, **, %"
puts "Type 'quit' to exit"
puts ""

loop do
  print "Enter expression (e.g., '5 + 3'): "
  input = gets
  
  break if input.nil?
  
  input = input.chomp
  break if input == "quit" || input == "exit"
  
  parts = input.split
  
  if parts.size != 3
    puts "Invalid format. Use: number operator number"
    next
  end
  
  begin
    a = parts[0].to_f
    op = parts[1]
    b = parts[2].to_f
    
    result = calculate(a, op, b)
    
    # Format result nicely
    if result == result.to_i.to_f
      puts "= #{result.to_i}"
    else
      puts "= #{result.round(6)}"
    end
  rescue ArgumentError => e
    puts "Error: #{e.message}"
  rescue Exception => e
    puts "Invalid input: #{e.message}"
  end
  
  puts ""
end

puts "Goodbye!"
```

---

## โปรแกรมตัวอย่าง: FizzBuzz

```crystal
# fizzbuzz.cr

# Classic FizzBuzz - คลาสสิก interview question

def fizzbuzz(n : Int32) : String
  case
  when n % 15 == 0 then "FizzBuzz"
  when n % 3 == 0  then "Fizz"
  when n % 5 == 0  then "Buzz"
  else n.to_s
  end
end

# Print 1 to 100
(1..100).each do |n|
  puts fizzbuzz(n)
end

# แบบกระชับ
puts (1..100).map { |n|
  case
  when n % 15 == 0 then "FizzBuzz"
  when n % 3 == 0  then "Fizz"
  when n % 5 == 0  then "Buzz"
  else n.to_s
  end
}.join("\n")
```

---

## โปรแกรมตัวอย่าง: Number Guessing Game

```crystal
# guessing_game.cr

# Number guessing game

def play_game
  secret = Random.rand(1..100)
  attempts = 0
  max_attempts = 7
  
  puts "=== Number Guessing Game ==="
  puts "Guess a number between 1 and 100"
  puts "You have #{max_attempts} attempts"
  puts ""
  
  max_attempts.times do |attempt|
    remaining = max_attempts - attempt
    print "Guess #{attempt + 1}/#{max_attempts} (#{remaining} remaining): "
    
    input = gets
    break if input.nil?
    
    begin
      guess = input.chomp.to_i
      attempts += 1
      
      if guess == secret
        puts ""
        puts "🎉 Correct! You guessed it in #{attempts} attempts!"
        return true
      elsif guess < secret
        puts "Too low! Try higher."
      else
        puts "Too high! Try lower."
      end
      
      # Hint for last attempt
      if remaining == 2
        diff = (secret - guess).abs
        hint = diff <= 5 ? "very close" : diff <= 10 ? "close" : "far"
        puts "(Hint: You are #{hint})"
      end
      
    rescue ArgumentError
      puts "Please enter a valid number"
      redo  # Repeat this iteration
    end
    
    puts ""
  end
  
  puts "Game over! The number was #{secret}"
  false
end

# Play multiple rounds
loop do
  won = play_game
  
  puts ""
  print "Play again? (y/n): "
  answer = gets
  break if answer.nil?
  break unless answer.chomp.downcase == "y"
  puts ""
end

puts "Thanks for playing!"
```

---

## Crystal REPL (Interactive Mode)

Crystal ไม่มี built-in REPL เหมือน Ruby's irb แต่มีทางเลือก:

```bash
# ใช้ icr (Unofficial REPL)
# ติดตั้ง: https://github.com/crystal-community/icr
icr

# หรือใช้ crystal eval สำหรับ one-liners
crystal eval "puts 1 + 1"
crystal eval "puts [1,2,3].map { |x| x * 2 }"

# หรือสร้างไฟล์ temp และรัน
echo 'puts "Hello"' | crystal eval
```

---

## Compile vs Run Mode

```bash
# Run mode: compile แล้วรันทันที (development)
crystal run hello.cr

# Build mode: compile เป็น binary (production)
crystal build hello.cr
./hello

# Release build: optimized binary
crystal build --release hello.cr
./hello

# Static binary (Linux)
crystal build --static hello.cr

# Build for different target (cross-compile)
crystal build --cross-compile --target x86_64-linux-gnu hello.cr
```

### ขนาดและความเร็ว
```bash
# เปรียบเทียบ build modes
time crystal run hello.cr          # ~3 seconds (includes compile)
time crystal build hello.cr && time ./hello  # compile ~3s, run <1ms
time crystal build --release hello.cr && time ./hello  # compile ~10s, run <0.5ms
```

---

## Error Messages และการ Debug

```crystal
# Crystal error messages มีรายละเอียดมาก

# ตัวอย่าง type error
age = 25
puts age + "years old"  
# Error: no overload matches 'Int32#+' with type String
# 
# Overloads are:
#   Int32#+(other : Int8)
#   Int32#+(other : Int16)
#   ...
# 
# Couldn't find overloads for these types:
# - Int32#+(other : String)

# แก้ไข:
puts "#{age} years old"       # String interpolation
puts age.to_s + " years old"  # Convert to String
```

### การ Debug ด้วย p และ pp
```crystal
def complex_calculation(numbers : Array(Int32)) : Int32
  filtered = numbers.select { |n| n > 0 }
  p filtered  # Debug: แสดง filtered values
  
  doubled = filtered.map { |n| n * 2 }
  pp doubled  # Debug: แสดง doubled values
  
  doubled.sum
end

result = complex_calculation([-3, 1, -1, 4, 2, -5])
puts "Result: #{result}"
```

### Logging
```crystal
require "log"

# Setup logging
Log.setup(:debug)
logger = Log.for("MyApp")

logger.debug { "Debug message" }
logger.info { "Application started" }
logger.warn { "Something might be wrong" }
logger.error { "An error occurred" }
logger.fatal { "Fatal error - shutting down" }
```

---

## สรุป Part 002

ในบทนี้เราได้เรียนรู้:

1. **Output methods**: puts, print, p, pp, printf
2. **String interpolation**: การใช้ `#{}`
3. **Comments**: # สำหรับ single-line, =begin/=end สำหรับ multi-line
4. **โครงสร้าง program**: require, constants, types, main code
5. **Input**: gets, ARGV, ENV
6. **Compile modes**: run, build, release
7. **Debugging**: p, pp, Log

---

## แบบฝึกหัด

1. เขียนโปรแกรมรับชื่อและอายุจาก input แล้วแสดงผล
2. เขียน FizzBuzz ของตัวเองโดยใช้วิธีอื่น
3. ปรับแต่ง Calculator ให้รองรับการคำนวณซับซ้อนขึ้น
4. เพิ่ม high score tracking ในเกม Number Guessing

---

## ขั้นตอนต่อไป

ไปที่ [Part 003](part_003.md) เพื่อเรียนรู้:
- ตัวแปรและชนิดข้อมูลพื้นฐาน
- Integer, Float, String, Bool, Char, Symbol
- Type inference
- Type checking
