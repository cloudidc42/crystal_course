# Part 005: Strings - การทำงานกับข้อความ

## String Basics

```crystal
# String literals
s1 = "Hello, World!"
s2 = "It's a beautiful day"  # Single quote inside double quotes

# Multi-line
multi = "First line
Second line
Third line"

# Heredoc
heredoc = <<-TEXT
  This is a heredoc.
  Leading whitespace is stripped.
  TEXT

# Raw string (no escape sequences)
raw1 = "Line1\nLine2"  # Contains actual newline
raw2 = %(Line1\nLine2) # Also a string, \n is literal

# String with escapes
escapes = "Tab:\t\nNewline:\nQuote:\"Back:\"

# Unicode
thai = "สวัสดีชาวโลก"
emoji = "Hello 🌍!"
puts thai.size    # 13 (number of chars)
puts emoji.size   # 8

# Frozen string literals (optimization)
# Crystal strings are already immutable by default
```

## String Creation Methods

```crystal
# Basic creation
s = String.new("Hello")  # Same as "Hello"
s = " " * 10             # "          " (10 spaces)
s = "ha" * 3             # "hahaha"

# String.build (efficient concatenation)
result = String.build do |io|
  io << "Hello"
  io << ", "
  io << "World"
  io << "!"
end
puts result  # Hello, World!

# Large string building (efficient)
result = String.build(capacity: 1000) do |io|
  100.times { |i| io << "Item #{i}\n" }
end

# Format strings
name = "Alice"
age = 25
s = "Name: %s, Age: %d" % [name, age]
s = sprintf("Name: %s, Age: %d", name, age)
s = "Name: #{name}, Age: #{age}"  # String interpolation (preferred)
```

## String Access and Slicing

```crystal
s = "Hello, World!"

# Character access
puts s[0]     # H (Char)
puts s[-1]    # ! (last char)
puts s[0..4]  # Hello (range)
puts s[7..]   # World! (from index to end)
puts s[..4]   # Hello (from start to index)
puts s[0, 5]  # Hello (start, length)

# Safe access (returns nil if out of bounds)
puts s[100]?  # nil
puts s[-100]? # nil

# Chars and bytes
puts s.chars.first(5).inspect    # ['H', 'e', 'l', 'l', 'o']
puts s.bytes.first(5).inspect    # [72, 101, 108, 108, 111]
puts s.codepoints.first(5).inspect  # [72, 101, 108, 108, 111]

# Iteration
s.each_char { |c| print c }
puts ""

s.each_char.with_index { |c, i| puts "#{i}: #{c}" }

s.each_byte { |b| print "#{b} " }
puts ""
```

## String Comparison

```crystal
# Equality
puts "hello" == "hello"  # true
puts "hello" == "Hello"  # false
puts "hello".eql?("hello")  # true (same as ==)

# Case-insensitive comparison
puts "Hello".casecmp("hello")  # 0 (equal)
puts "Hello".casecmp?("hello") # true

# Ordering (lexicographic)
puts "apple" < "banana"   # true
puts "zebra" > "apple"    # true
puts "abc" <=> "abd"      # -1 (spaceship operator)
puts "abc" <=> "abc"      # 0
puts "abd" <=> "abc"      # 1

# Sort strings
words = ["banana", "apple", "cherry", "date"]
puts words.sort.inspect        # ["apple", "banana", "cherry", "date"]
puts words.sort_by(&.size).inspect  # ["date", "apple", "banana", "cherry"]

# Natural sort (1, 2, 10 vs 1, 10, 2)
files = ["file10.txt", "file2.txt", "file1.txt"]
puts files.sort.inspect  # ["file1.txt", "file10.txt", "file2.txt"]
```

## String Searching

```crystal
s = "The quick brown fox jumps over the lazy dog"

# Checking presence
puts s.includes?("fox")     # true
puts s.includes?("cat")     # false
puts s.starts_with?("The")  # true
puts s.ends_with?("dog")    # true

# Finding position
puts s.index("fox")         # 16 (first occurrence)
puts s.index("the")         # 31 (case sensitive)
puts s.rindex("the")        # 31 (last occurrence)
puts s.index("o")           # 12 (first 'o')
puts s.rindex("o")          # 41 (last 'o')

# Safe index (returns nil if not found)
puts s.index("cat")         # nil
puts s.index?("fox")        # 16

# All occurrences
positions = [] of Int32
offset = 0
while (idx = s.index("the", offset))
  positions << idx
  offset = idx + 1
end
puts positions.inspect  # [31] (case sensitive)

# Count occurrences
puts s.count("o")  # 4 (count of char 'o')
```

## String Modification

```crystal
s = "  Hello, World!  "

# Trimming whitespace
puts s.strip      # "Hello, World!"
puts s.lstrip     # "Hello, World!  "
puts s.rstrip     # "  Hello, World!"

# Custom trim
puts "xxxHelloxx".strip('x')   # "Hello"
puts "---Hello---".strip('-')  # "Hello"

# Case changes
puts "hello world".upcase      # "HELLO WORLD"
puts "HELLO WORLD".downcase    # "hello world"
puts "hello world".capitalize  # "Hello world"
puts "hello world".titleize    # (not in stdlib, see below)
puts "Hello World".swapcase    # "hELLO wORLD"

# Custom titleize
def titleize(s : String) : String
  s.split.map(&.capitalize).join(" ")
end
puts titleize("the quick brown fox")  # "The Quick Brown Fox"

# Padding
puts "hello".ljust(10)        # "hello     "
puts "hello".rjust(10)        # "     hello"
puts "hello".center(11)       # "   hello   "
puts "hello".ljust(10, '*')   # "hello*****"
puts "hello".rjust(10, '0')   # "00000hello"
puts "42".rjust(8, '0')       # "00000042"
```

## String Replacement

```crystal
s = "Hello, World! Hello, Crystal!"

# sub - replace first occurrence
puts s.sub("Hello", "Hi")           # "Hi, World! Hello, Crystal!"
puts s.sub(/Hello/, "Hi")           # "Hi, World! Hello, Crystal!"

# gsub - replace all occurrences
puts s.gsub("Hello", "Hi")          # "Hi, World! Hi, Crystal!"
puts s.gsub(/[aeiou]/, "*")         # "H*ll*, W*rld! H*ll*, Cryst*l!"

# gsub with block
puts s.gsub(/\w+/) { |w| w.upcase } # "HELLO, WORLD! HELLO, CRYSTAL!"

# gsub with hash mapping
puts s.gsub(/\b\w+\b/, {"Hello" => "Hi", "World" => "Earth"})
# "Hi, Earth! Hello, Crystal!"

# delete - remove characters
puts "Hello, World!".delete("aeiou")  # "Hll, Wrld!"
puts "Hello, World!".delete("a-z")    # "H, W!"

# tr - transliterate (replace char by char)
puts "Hello".tr("aeiou", "*")         # "H*ll*"
puts "Hello".tr("a-z", "A-Z")         # "HELLO"
puts "Hello World".tr("a-z", "n-za-m") # ROT13!

# squeeze - remove consecutive duplicates
puts "aaabbbccc".squeeze              # "abc"
puts "aabbcc".squeeze("a")            # "abcc"

# encode/decode
puts "hello world".gsub(" ", "%20")   # URL encode spaces manually
```

## String Splitting

```crystal
s = "apple,banana,cherry,date"

# Basic split
puts s.split(",").inspect     # ["apple", "banana", "cherry", "date"]
puts s.split(",", 2).inspect  # ["apple", "banana,cherry,date"] (limit)

# Split on whitespace
"  hello   world  ".split.inspect  # ["hello", "world"]

# Split on regex
"one1two2three3four".split(/\d/).inspect  # ["one", "two", "three", "four"]

# Split into chars
"hello".chars.inspect  # ['h', 'e', 'l', 'l', 'o']

# Split into words
text = "The quick brown fox"
words = text.split
puts words.inspect  # ["The", "quick", "brown", "fox"]

# Partition - splits into 3 parts
before, match, after = "hello=world".partition("=")
puts before  # "hello"
puts match   # "="
puts after   # "world"

# Lines
"line1\nline2\nline3".lines.inspect  # ["line1\n", "line2\n", "line3"]
"line1\nline2\nline3".lines.map(&.chomp)  # ["line1", "line2", "line3"]
```

## String Joining

```crystal
words = ["Hello", "World", "from", "Crystal"]

# join with separator
puts words.join(", ")    # "Hello, World, from, Crystal"
puts words.join(" ")     # "Hello World from Crystal"
puts words.join          # "HelloWorldfromCrystal"
puts words.join(" | ")   # "Hello | World | from | Crystal"

# join with prefix/suffix
puts words.join(", ", prefix: "[", suffix: "]")  # "[Hello, World, from, Crystal]"

# Concatenation
s1 = "Hello"
s2 = " World"
puts s1 + s2            # "Hello World"
puts s1.concat(s2)      # "Hello World" (returns new string)
puts [s1, s2].join      # "Hello World"

# String interpolation (preferred)
name = "Alice"
puts "Hello, #{name}!"  # "Hello, Alice!"
```

## String Encoding

```crystal
# Crystal strings are UTF-8 by default

# Thai string
thai = "สวัสดี"
puts thai.size       # 6 (characters)
puts thai.bytesize   # 18 (bytes - Thai is 3 bytes per char in UTF-8)

# Emoji
emoji = "Hello 🌍"
puts emoji.size      # 7 (characters)
puts emoji.bytesize  # 11 (bytes - emoji is 4 bytes)

# Encoding detection
puts "hello".encoding  # "UTF-8"

# Bytes
"A".bytes.inspect  # [65]
"A".ord            # 65
65.chr             # 'A'

# Unicode code points
"❤".codepoints.inspect  # [10084]
10084.chr                # '❤'

# String from bytes
bytes = [72_u8, 101_u8, 108_u8, 108_u8, 111_u8]
String.new(bytes.to_unsafe, bytes.size)  # "Hello"

# Base64 encoding/decoding
require "base64"
encoded = Base64.encode("Hello, World!")  # "SGVsbG8sIFdvcmxkIQ==\n"
decoded = Base64.decode_string(encoded)   # "Hello, World!"

# URL encoding
require "uri"
encoded = URI.encode_path_segment("hello world & stuff")
# "hello%20world%20%26%20stuff"
decoded = URI.decode(encoded)
# "hello world & stuff"

# HTML encoding
require "html"
html_encoded = HTML.escape("<script>alert('xss')</script>")
# "&lt;script&gt;alert(&#39;xss&#39;)&lt;/script&gt;"
```

## String IO (Efficient Building)

```crystal
require "io"

# StringIO - string yang bisa ditulis
io = IO::Memory.new
io << "Hello"
io << ", "
io << "World"
io << "!"
puts io.to_s  # "Hello, World!"

# String.build (simpler API)
result = String.build do |io|
  io << "Hello"
  10.times { |i| io << " #{i}" }
end
puts result

# String::Builder for high-performance string building
class Report
  def initialize
    @sections = [] of String
  end
  
  def add_section(title : String, content : String)
    @sections << String.build do |io|
      io << "## #{title}\n"
      io << content
      io << "\n\n"
    end
  end
  
  def to_s
    @sections.join
  end
end

report = Report.new
report.add_section("Introduction", "This is the intro...")
report.add_section("Details", "Here are the details...")
puts report.to_s
```

## Regular Expressions Basics

```crystal
# Basic regex
if "hello world" =~ /hello/
  puts "Match found!"
end

# Match method
m = "hello world".match(/(\w+)\s(\w+)/)
if m
  puts m[0]  # "hello world" (full match)
  puts m[1]  # "hello" (first capture group)
  puts m[2]  # "world" (second capture group)
end

# Named captures
m = "2024-01-15".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/)
if m
  puts m["year"]   # "2024"
  puts m["month"]  # "01"
  puts m["day"]    # "15"
end

# Check with regex
puts "alice@example.com".matches?(/\A[\w.+\-]+@[a-z\d\-.]+\.[a-z]+\z/i)  # true
puts "not-an-email".matches?(/\A[\w.+\-]+@[a-z\d\-.]+\.[a-z]+\z/i)       # false

# Scan - all matches
"one1two2three3".scan(/\d+/).each { |m| puts m[0] }
# 1, 2, 3

matches = "one1two2three3".scan(/\d+/).map { |m| m[0] }
puts matches.inspect  # ["1", "2", "3"]

# Replace with regex
puts "hello world".gsub(/\b\w/, &.upcase)  # "Hello World"
puts "2024-01-15".gsub(/-/, "/")           # "2024/01/15"
```

## Advanced String Operations

```crystal
# String similarity (Levenshtein distance)
def levenshtein(s1 : String, s2 : String) : Int32
  m, n = s1.size, s2.size
  dp = Array.new(m + 1) { |i| Array.new(n + 1) { |j| i == 0 ? j : j == 0 ? i : 0 } }
  
  (1..m).each do |i|
    (1..n).each do |j|
      if s1[i-1] == s2[j-1]
        dp[i][j] = dp[i-1][j-1]
      else
        dp[i][j] = 1 + {dp[i-1][j], dp[i][j-1], dp[i-1][j-1]}.min
      end
    end
  end
  
  dp[m][n]
end

puts levenshtein("kitten", "sitting")   # 3
puts levenshtein("hello", "hello")      # 0
puts levenshtein("", "abc")             # 3

# String palindrome
def palindrome?(s : String) : Bool
  cleaned = s.downcase.gsub(/[^a-z0-9]/, "")
  cleaned == cleaned.reverse
end

puts palindrome?("racecar")              # true
puts palindrome?("A man a plan a canal Panama")  # true
puts palindrome?("hello")               # false

# Word frequency counter
def word_frequency(text : String) : Hash(String, Int32)
  freq = Hash(String, Int32).new(0)
  text.downcase.scan(/\b\w+\b/) { |m| freq[m[0]] += 1 }
  freq.to_a.sort_by { |_, v| -v }.to_h
end

text = "the quick brown fox jumps over the lazy dog the fox"
freq = word_frequency(text)
freq.first(5).each { |word, count| puts "#{word}: #{count}" }
# the: 3
# fox: 2
# quick: 1
# ...

# Caesar cipher
def caesar(text : String, shift : Int32) : String
  text.tr("a-zA-Z", 
    ('a'.ord..('a'.ord + 25)).map { |c| ((c - 'a'.ord + shift) % 26 + 'a'.ord).chr }.join +
    ('A'.ord..('A'.ord + 25)).map { |c| ((c - 'A'.ord + shift) % 26 + 'A'.ord).chr }.join
  )
end

puts caesar("Hello, World!", 13)  # Uryyb, Jbeyq!
puts caesar("Uryyb, Jbeyq!", 13)  # Hello, World! (decode)

# ROT13
def rot13(s : String) : String
  caesar(s, 13)
end
```

## Template Strings

```crystal
# Simple template engine

class Template
  def initialize(@template : String)
  end
  
  def render(vars : Hash(String, String)) : String
    result = @template
    vars.each do |key, value|
      result = result.gsub("{{#{key}}}", value)
    end
    result
  end
  
  def render(**vars) : String
    result = @template
    vars.each do |key, value|
      result = result.gsub("{{#{key}}}", value.to_s)
    end
    result
  end
end

# Usage
t = Template.new("Hello, {{name}}! You are {{age}} years old.")
puts t.render(name: "Alice", age: "25")
# Hello, Alice! You are 25 years old.

t2 = Template.new(<<-HTML)
  <html>
  <head><title>{{title}}</title></head>
  <body>
    <h1>{{heading}}</h1>
    <p>{{content}}</p>
  </body>
  </html>
  HTML

puts t2.render(
  title: "My Page",
  heading: "Welcome",
  content: "This is my page."
)
```

## String Performance Tips

```crystal
# ❌ Slow: String concatenation in loop
result = ""
10000.times { |i| result += "#{i}\n" }

# ✅ Fast: String.build
result = String.build do |io|
  10000.times { |i| io << i << "\n" }
end

# ❌ Slow: Multiple gsub
s = "Hello World"
s = s.gsub("Hello", "Hi")
s = s.gsub("World", "Earth")
s = s.gsub("!", ".")

# ✅ Fast: Single gsub with hash
s = "Hello World!"
replacements = {"Hello" => "Hi", "World" => "Earth", "!" => "."}
s = s.gsub(/Hello|World|!/, replacements)

# Benchmark
require "benchmark"

n = 10000

Benchmark.ips do |x|
  x.report("concatenation") do
    s = ""
    n.times { |i| s += i.to_s }
  end
  
  x.report("string_build") do
    String.build { |io| n.times { |i| io << i } }
  end
end
```

---

## ตัวอย่าง: Text Processor

```crystal
# text_processor.cr

class TextProcessor
  def initialize(@text : String)
  end
  
  def word_count : Int32
    @text.split.size
  end
  
  def char_count(include_spaces = true) : Int32
    include_spaces ? @text.size : @text.gsub(" ", "").size
  end
  
  def sentence_count : Int32
    @text.scan(/[.!?]+/).size
  end
  
  def paragraph_count : Int32
    @text.split(/\n\s*\n/).size
  end
  
  def most_common_words(top_n = 10) : Array({String, Int32})
    freq = Hash(String, Int32).new(0)
    stop_words = %w[the a an and or but in on at to for of with is are was were]
    
    @text.downcase.scan(/\b[a-z]+\b/) do |m|
      word = m[0]
      freq[word] += 1 unless stop_words.includes?(word)
    end
    
    freq.to_a.sort_by { |_, v| -v }.first(top_n)
  end
  
  def reading_time(wpm = 200) : Float64
    word_count / wpm.to_f
  end
  
  def summary : String
    String.build do |io|
      io << "=== Text Analysis ===\n"
      io << "Words: #{word_count}\n"
      io << "Characters: #{char_count}\n"
      io << "Characters (no spaces): #{char_count(false)}\n"
      io << "Sentences: #{sentence_count}\n"
      io << "Paragraphs: #{paragraph_count}\n"
      io << "Est. reading time: #{reading_time.round(1)} minutes\n"
      io << "\nTop 5 words:\n"
      most_common_words(5).each_with_index do |(word, count), i|
        io << "  #{i+1}. #{word}: #{count}\n"
      end
    end
  end
end

sample_text = <<-TEXT
  Crystal is a programming language with the following goals:
  Have a syntax similar to Ruby (but compatibility with it is not a goal).
  Statically type-checked but without having to specify the type of variables or method arguments.
  Be able to call C code by writing bindings to it in Crystal.
  Have compile-time evaluation and generation of code, to avoid boilerplate code.
  Compile to efficient native code.
  TEXT

processor = TextProcessor.new(sample_text)
puts processor.summary
```

---

## สรุป Part 005

ในบทนี้เราได้เรียนรู้:

1. **String creation**: literals, heredoc, String.build
2. **Access and slicing**: index, range, chars, bytes
3. **Comparison**: ==, casecmp, <=>
4. **Searching**: includes?, starts_with?, index, rindex, scan
5. **Modification**: strip, upcase, downcase, capitalize, pad
6. **Replacement**: sub, gsub, delete, tr, squeeze
7. **Splitting**: split, partition, chars, lines
8. **Joining**: join, +, concat, interpolation
9. **Encoding**: UTF-8, Base64, URL, HTML
10. **Regular expressions**: basics, match, scan, replace
11. **Performance**: String.build vs concatenation

---

## ขั้นตอนต่อไป

ไปที่ [Part 006](part_006.md) เพื่อเรียนรู้:
- Boolean operators
- Comparison operators
- Logical operators
- Bitwise operators
- Operator precedence
