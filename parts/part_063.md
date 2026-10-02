# Part 63: Regular Expressions พื้นฐาน

## บทนำ

Regular Expressions (Regex) เป็นเครื่องมือทรงพลังสำหรับการค้นหาและจัดการข้อความตาม pattern Crystal มี built-in regex support ที่ใช้ PCRE (Perl Compatible Regular Expressions)

---

## 1. การสร้าง Regex Pattern

```crystal
# สร้าง regex literal ด้วย /pattern/
pattern = /hello/
puts pattern.class  # => Regex

# case insensitive
pattern_i = /hello/i
puts "Hello" =~ pattern_i  # => 0 (match ที่ position 0)

# สร้างด้วย Regex.new
pattern = Regex.new("hello")
pattern_i = Regex.new("hello", Regex::Options::IGNORE_CASE)

# ตรวจสอบ match
puts /hello/ === "say hello"    # => true
puts /hello/ === "say goodbye"  # => false
```

---

## 2. =~ Operator

```crystal
# =~ คืนค่า position ของ match หรือ nil
str = "hello world"
pos = str =~ /world/
puts pos  # => 6

pos = str =~ /xyz/
puts pos.inspect  # => nil

# ใช้ใน conditional
if str =~ /world/
  puts "Found 'world' at position #{$~.begin}"
end

# =~ กับ $~ (special variable)
"hello world" =~ /(\w+)\s+(\w+)/
puts $~[0]  # => "hello world" (full match)
puts $~[1]  # => "hello" (first capture)
puts $~[2]  # => "world" (second capture)

# เช็ค match ก่อนใช้ $~
if "2024-01-15" =~ /(\d{4})-(\d{2})-(\d{2})/
  year = $~[1]
  month = $~[2]
  day = $~[3]
  puts "Year: #{year}, Month: #{month}, Day: #{day}"
end

# =~ กลับทาง (Regex =~ String)
pos = /world/ =~ "hello world"
puts pos  # => 6
```

---

## 3. match Method

```crystal
# match คืนค่า Regex::MatchData หรือ nil
str = "hello world"
m = str.match(/(\w+)\s+(\w+)/)

if m
  puts m[0]  # full match: "hello world"
  puts m[1]  # first group: "hello"
  puts m[2]  # second group: "world"
  puts m.begin  # start position: 0
  puts m.end    # end position: 11
end

# match ที่ไม่เจอ
m = str.match(/xyz/)
puts m.inspect  # => nil

# match_all / scan
str = "one 1, two 2, three 3"
str.scan(/(\w+)\s+(\d)/) do |m|
  puts "Word: #{m[1]}, Number: #{m[2]}"
end
# => Word: one, Number: 1
# => Word: two, Number: 2
# => Word: three, Number: 3

# ใช้ match? สำหรับ boolean check
puts str.match?(/\d+/)  # => true

# match with offset
str = "hello hello"
m1 = str.match(/hello/)           # ค้นหาจากต้น
m2 = str.match(/hello/, offset: 1) # ค้นหาจาก offset 1

puts m1.try(&.begin)  # => 0
puts m2.try(&.begin)  # => 6
```

---

## 4. scan Method

```crystal
# scan คืนค่า Array of String (หรือ Array of Array ถ้ามี groups)
str = "the cat sat on the mat"
words = str.scan(/\b\w{3}\b/)
puts words.inspect  # => ["the", "cat", "sat", "the", "mat"]

# scan ด้วย capture groups
str = "name=Alice, age=25, city=Bangkok"
pairs = str.scan(/(\w+)=(\w+)/)
pairs.each do |m|
  puts "#{m[1]} => #{m[2]}"
end
# => name => Alice
# => age => 25
# => city => Bangkok

# สร้าง Hash จาก scan
config = {} of String => String
"key1=val1 key2=val2 key3=val3".scan(/(\w+)=(\w+)/) do |m|
  config[m[1]] = m[2]
end
puts config.inspect  # => {"key1" => "val1", "key2" => "val2", "key3" => "val3"}

# scan สำหรับ tokenizing
def tokenize(expr : String) : Array(String)
  expr.scan(/\d+\.\d+|\d+|[+\-*\/()]|\w+/)
      .map(&.[0])
end

puts tokenize("x = 42 + 3.14 * (y - 1)").inspect
# => ["x", "=", "42", "+", "3.14", "*", "(", "y", "-", "1", ")"]
```

---

## 5. Anchors: ^ $ \A \Z \b

```crystal
# ^ matches start of line (in multiline mode)
# \A matches start of string
# $ matches end of line (in multiline mode)  
# \Z matches end of string
# \b matches word boundary

# ^ และ $
text = "hello\nworld"
puts text.match?(/^hello$/)  # => true (^ และ $ match per line)
puts text.match?(/^world$/)  # => true

# \A และ \Z (start/end of entire string)
puts text.match?(/\Ahello/)   # => true (starts with hello)
puts text.match?(/\Aworld/)   # => false
puts text.match?(/world\Z/)   # => true (ends with world)

# word boundary \b
str = "hello world hello"
puts str.scan(/\bhello\b/).size  # => 2 (exact word)
puts str.scan(/hello/).size      # => 2

str2 = "helloworld"
puts str2.match?(/\bhello\b/)  # => false (ไม่มี word boundary)
puts str2.match?(/hello/)      # => true

# ตัวอย่างจริง
def valid_identifier?(str : String) : Bool
  str.match?(/\A[a-zA-Z_]\w*\Z/)
end

puts valid_identifier?("hello")      # => true
puts valid_identifier?("_private")   # => true
puts valid_identifier?("123abc")     # => false
puts valid_identifier?("hello-world")# => false

# Email validation
def valid_email?(email : String) : Bool
  email.match?(/\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/)
end

puts valid_email?("user@example.com")    # => true
puts valid_email?("invalid-email")       # => false
puts valid_email?("user@.com")           # => false
```

---

## 6. Character Classes

```crystal
# [] สร้าง character class
puts "hello123".match?(/[aeiou]/)    # => true (vowel)
puts "hello123".match?(/[0-9]/)      # => true (digit)
puts "HELLO".match?(/[a-z]/)         # => false (no lowercase)

# ^ ใน [] หมายถึง complement (NOT)
puts "hello".match?(/[^aeiou]/)    # => true (has consonant)
puts "aeiou".match?(/[^aeiou]/)   # => false (only vowels)

# predefined character classes
puts "hello123".scan(/\w+/).inspect  # word chars: => ["hello123"]
puts "hello 123".scan(/\W+/).inspect # non-word: => [" "]
puts "abc123".scan(/\d+/).inspect    # digits: => ["123"]
puts "abc123".scan(/\D+/).inspect    # non-digits: => ["abc"]
puts "hello world".scan(/\s+/).inspect  # whitespace: => [" "]
puts "hello world".scan(/\S+/).inspect  # non-whitespace: => ["hello", "world"]

# character class examples
def count_vowels(str : String) : Int32
  str.scan(/[aeiouAEIOU]/).size
end

def remove_punctuation(str : String) : String
  str.gsub(/[^\w\s]/, "")
end

def extract_numbers(str : String) : Array(String)
  str.scan(/\d+(?:\.\d+)?/).map(&.[0])
end

puts count_vowels("Hello World")        # => 3
puts remove_punctuation("Hello, World!") # => "Hello World"
puts extract_numbers("Price: 9.99, Qty: 3, Total: 29.97").inspect
# => ["9.99", "3", "29.97"]

# Thai character class (Unicode range)
def contains_thai?(str : String) : Bool
  str.match?(/[\u0E00-\u0E7F]/)
end

puts contains_thai?("สวัสดี")     # => true
puts contains_thai?("Hello")     # => false
puts contains_thai?("Hello สวัสดี") # => true
```

---

## 7. Quantifiers: *, +, ?, {n,m}

```crystal
# * - 0 or more
puts "color" =~ /colou*r/  # => 0 (matches "color")
puts "colour" =~ /colou*r/ # => 0 (matches "colour")
puts "colouuur" =~ /colou*r/ # => 0 (matches "colouuur")

# + - 1 or more
puts "color".match?(/colou+r/)    # => false (needs at least 1 u)
puts "colour".match?(/colou+r/)   # => true
puts "colouuur".match?(/colou+r/) # => true

# ? - 0 or 1 (optional)
puts "color".match?(/colou?r/)    # => true
puts "colour".match?(/colou?r/)   # => true
puts "colouuur".match?(/colou?r/) # => false (too many u)

# {n} - exactly n times
puts "aaa".match?(/a{3}/)  # => true
puts "aa".match?(/a{3}/)   # => false
puts "aaaa".match?(/a{3}/) # => true (3 found in 4)

# {n,m} - between n and m times
puts "aa".match?(/a{2,4}/)    # => true
puts "aaa".match?(/a{2,4}/)   # => true
puts "aaaa".match?(/a{2,4}/)  # => true
puts "aaaaa".match?(/a{2,4}/) # => true (2-4 found in 5)

# {n,} - n or more
puts "aa".match?(/a{3,}/)   # => false
puts "aaa".match?(/a{3,}/)  # => true
puts "aaaa".match?(/a{3,}/) # => true

# ตัวอย่างจริง
def valid_password?(password : String) : Bool
  # ต้องมี 8-20 chars, uppercase, lowercase, digit, special
  password.size.in?(8..20) &&
    password.match?(/[A-Z]/) &&
    password.match?(/[a-z]/) &&
    password.match?(/\d/) &&
    password.match?(/[!@#$%^&*]/)
end

puts valid_password?("Password1!")  # => true
puts valid_password?("short1!")     # => false (too short)
puts valid_password?("nouppercase1!") # => false

# Phone number pattern
def valid_thai_phone?(phone : String) : Bool
  # 08x-xxx-xxxx หรือ 08xxxxxxxx
  phone.match?(/\A0[689]\d{1}[-]?\d{3}[-]?\d{4}\Z/)
end

puts valid_thai_phone?("089-123-4567")  # => true
puts valid_thai_phone?("0891234567")    # => true
puts valid_thai_phone?("123-456-7890")  # => false

# Greedy vs non-greedy (จะเรียนละเอียดใน part 64)
str = "<b>bold</b> and <i>italic</i>"
puts str.scan(/<.+>/).inspect   # greedy: matches whole string
puts str.scan(/<.+?>/).inspect  # non-greedy: matches each tag
```

---

## 8. Groups และ Captures

```crystal
# () สร้าง capture group
"2024-01-15" =~ /(\d{4})-(\d{2})-(\d{2})/
puts $~[0]  # => "2024-01-15" (full match)
puts $~[1]  # => "2024"
puts $~[2]  # => "01"
puts $~[3]  # => "15"

# non-capturing group (?:...)
"2024-01-15" =~ /(?:\d{4})-(\d{2})-(\d{2})/
puts $~[0]  # => "2024-01-15" (full match)
puts $~[1]  # => "01" (first captured group, year ไม่ถูก capture)
puts $~[2]  # => "15"

# nested groups
str = "John Smith (age 25)"
if str =~ /(\w+)\s+(\w+)\s+\(age\s+(\d+)\)/
  puts "First: #{$~[1]}"   # => John
  puts "Last: #{$~[2]}"    # => Smith
  puts "Age: #{$~[3]}"     # => 25
end

# alternation | ใน groups
str = "I have a cat"
if str =~ /(cat|dog|bird)/
  puts "Pet found: #{$~[1]}"  # => cat
end

# แยก IPv4 address
def parse_ipv4(ip : String) : Array(Int32)?
  if ip =~ /\A(\d{1,3})\.(\d{1,3})\.(\d{1,3})\.(\d{1,3})\Z/
    parts = (1..4).map { |i| $~[i].to_i }
    if parts.all? { |p| p.in?(0..255) }
      return parts
    end
  end
  nil
end

if parts = parse_ipv4("192.168.1.1")
  puts parts.inspect  # => [192, 168, 1, 1]
end

puts parse_ipv4("256.1.1.1").inspect   # => nil (invalid)
puts parse_ipv4("not.an.ip").inspect   # => nil
```

---

## 9. Common Regex Patterns

```crystal
# Pattern สำคัญที่ใช้บ่อย
module Patterns
  # Email
  EMAIL = /\A[\w.!#$%&'*+\-\/=?^`{|}~]+@[\w\-]+(?:\.[\w\-]+)+\Z/
  
  # URL
  URL = /\Ahttps?:\/\/[\w\-._~:\/\?#\[\]@!$&'()*+,;=%]+\Z/
  
  # Thai phone number
  THAI_PHONE = /\A(?:\+66|0)([689]\d{1})[-\s]?(\d{3})[-\s]?(\d{4})\Z/
  
  # Thai ID card
  THAI_ID = /\A\d{13}\Z/
  
  # Date (YYYY-MM-DD)
  DATE_ISO = /\A(\d{4})-(\d{2})-(\d{2})\Z/
  
  # Time (HH:MM:SS)
  TIME = /\A(\d{2}):(\d{2})(?::(\d{2}))?\Z/
  
  # IP Address
  IPV4 = /\A(?:\d{1,3}\.){3}\d{1,3}\Z/
  
  # Credit card (basic)
  CREDIT_CARD = /\A\d{4}[- ]?\d{4}[- ]?\d{4}[- ]?\d{4}\Z/
  
  # Postal code (Thai)
  THAI_POSTAL = /\A\d{5}\Z/
  
  # Username
  USERNAME = /\A[a-zA-Z][\w]{2,19}\Z/
  
  # Hex color
  HEX_COLOR = /\A#(?:[0-9a-fA-F]{3}|[0-9a-fA-F]{6})\Z/
end

# ทดสอบ patterns
puts "Test Email:"
emails = ["user@example.com", "invalid", "user@.com", "a@b.co"]
emails.each do |e|
  puts "  #{e}: #{Patterns::EMAIL.match?(e)}"
end

puts "\nTest URL:"
urls = ["https://example.com", "http://test.org/path", "not-a-url"]
urls.each do |u|
  puts "  #{u}: #{Patterns::URL.match?(u)}"
end

puts "\nTest HEX Color:"
colors = ["#FFF", "#aabbcc", "#GGGGGG", "FFF"]
colors.each do |c|
  puts "  #{c}: #{Patterns::HEX_COLOR.match?(c)}"
end
```

---

## 10. Regex Flags

```crystal
# i - case insensitive
puts "HELLO" =~ /hello/i     # => 0
puts "Hello" =~ /hello/i     # => 0

# m - multiline (. matches newline)
text = "hello\nworld"
puts text =~ /hello.world/    # => nil (. ไม่ match newline)
puts text =~ /hello.world/m   # => 0 (. match newline)

# x - extended mode (allows comments and whitespace)
email_pattern = /
  \A          # start of string
  [\w.+-]+    # local part
  @           # @ symbol
  [\w-]+      # domain
  (?:\.       # dot
    [\w.-]+   # TLD
  )+          # one or more TLD parts
  \Z          # end of string
/x

puts email_pattern.match?("user@example.com")  # => true
puts email_pattern.match?("invalid")            # => false

# combining flags
text = "Hello\nWorld"
puts text.match?(/hello\nworld/im)  # => true (both i and m)

# Regex constants
WORD = /\b\w+\b/i  # reusable pattern

def count_words(text : String) : Int32
  text.scan(WORD).size
end

puts count_words("Hello World")    # => 2
puts count_words("The Quick Brown Fox")  # => 4
```

---

## 11. Practical Examples

```crystal
# 1. Sanitize filename
def sanitize_filename(name : String) : String
  name
    .gsub(/[^\w\s.-]/, "")    # ลบ special chars
    .gsub(/\s+/, "_")          # แทน spaces ด้วย _
    .gsub(/_+/, "_")           # ลด multiple _
    .gsub(/^[._]+/, "")        # ลบ leading . หรือ _
    .downcase
end

puts sanitize_filename("Hello World! (2024).txt")
# => "hello_world_2024.txt"

# 2. Extract data from text
def extract_prices(text : String) : Array(Float64)
  text.scan(/[$฿€]?\s*(\d+(?:,\d{3})*(?:\.\d{2})?)/)
      .map { |m| m[1].delete(",").to_f }
end

puts extract_prices("Total: $1,234.56 and ฿9,999.00 and €42.50").inspect
# => [1234.56, 9999.0, 42.5]

# 3. Log parser
def parse_log_line(line : String) : Hash(String, String)?
  pattern = /\[(\d{4}-\d{2}-\d{2}\s\d{2}:\d{2}:\d{2})\]\s+(\w+)\s+(.+)/
  if line =~ pattern
    {
      "timestamp" => $~[1],
      "level" => $~[2],
      "message" => $~[3],
    }
  end
end

log = "[2024-01-15 10:30:00] ERROR Database connection failed"
if data = parse_log_line(log)
  data.each { |k, v| puts "#{k}: #{v}" }
end

# 4. Markdown link extractor
def extract_links(markdown : String) : Array({String, String})
  links = [] of {String, String}
  markdown.scan(/\[([^\]]+)\]\(([^)]+)\)/) do |m|
    links << {m[1], m[2]}
  end
  links
end

md = "See [Google](https://google.com) and [Crystal](https://crystal-lang.org)"
extract_links(md).each do |text, url|
  puts "#{text} -> #{url}"
end

# 5. Word wrap
def word_wrap(text : String, width : Int32 = 80) : String
  text.gsub(/(.{1,#{width}})(\s+|$)/, "\\1\n").strip
end

long_text = "This is a very long text that needs to be wrapped at a certain width for better readability in terminals and fixed-width displays."
puts word_wrap(long_text, 40)
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน validator ที่ตรวจสอบ Thai National ID (13 หลัก) รวมถึง checksum validation

### แบบฝึกหัดที่ 2
เขียน `parse_csv_line` ที่แยก CSV line โดยรองรับ quoted fields (ที่มีคอมม่าอยู่ใน quotes)

### แบบฝึกหัดที่ 3
เขียน `highlight_code_keywords` ที่รับ Crystal code และ wrap keywords ด้วย `<keyword>...</keyword>` tags

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
def valid_thai_id?(id : String) : Bool
  return false unless id.match?(/\A\d{13}\Z/)
  
  digits = id.chars.map(&.to_i)
  sum = (0..11).reduce(0) { |acc, i| acc + digits[i] * (13 - i) }
  checksum = (11 - (sum % 11)) % 10
  checksum == digits[12]
end

# ทดสอบด้วยเลขที่ valid (สมมุติ)
puts valid_thai_id?("1234567890123")  # ขึ้นอยู่กับ checksum

# แบบฝึกหัดที่ 2
def parse_csv_line(line : String, delimiter : Char = ',') : Array(String)
  fields = [] of String
  line.scan(/(?:^|#{Regex.escape(delimiter.to_s)})("(?:[^"\\]|\\.)*"|[^#{Regex.escape(delimiter.to_s)}]*)/) do |m|
    field = m[1]
    if field.starts_with?('"') && field.ends_with?('"')
      field = field[1..-2].gsub("\"\"", "\"")
    end
    fields << field
  end
  fields
end

puts parse_csv_line('Alice,25,"Bangkok, Thailand"').inspect
# => ["Alice", "25", "Bangkok, Thailand"]

# แบบฝึกหัดที่ 3
CRYSTAL_KEYWORDS = %w[def end if else elsif unless while until for do
                       return nil true false class module struct enum
                       include extend require macro property getter setter
                       initialize new type abstract]

def highlight_code_keywords(code : String) : String
  keywords_pattern = Regex.new("\\b(#{CRYSTAL_KEYWORDS.join("|")})\\b")
  code.gsub(keywords_pattern) { "<keyword>#{$~[1]}</keyword>" }
end

code = "def hello\n  if true\n    puts \"Hello!\"\n  end\nend"
puts highlight_code_keywords(code)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Regular Expressions พื้นฐานใน Crystal:

1. **การสร้าง Regex** ด้วย `/pattern/` literal หรือ `Regex.new`
2. **=~ operator** สำหรับหา match position
3. **match method** สำหรับได้ MatchData object
4. **scan method** สำหรับหาทุก matches
5. **Anchors** `^`, `$`, `\A`, `\Z`, `\b` สำหรับกำหนดตำแหน่ง
6. **Character classes** `[...]`, `\w`, `\d`, `\s` และ complements
7. **Quantifiers** `*`, `+`, `?`, `{n,m}` สำหรับจำนวนครั้ง
8. **Groups** `()` สำหรับ capture
9. **Flags** `i`, `m`, `x` สำหรับ modify behavior
10. **Practical patterns** สำหรับการใช้งานจริง

Regex เป็นเครื่องมือที่ทรงพลัง แต่ต้องระวังไม่ให้ complex จนอ่านยาก ในบทถัดไปเราจะเรียนรู้ Regex ขั้นสูงเพิ่มเติม
