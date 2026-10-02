# Part 61: String Methods ขั้นสูง

## บทนำ

Crystal มี String methods ที่ทรงพลังมากมาย ในบทนี้เราจะเรียนรู้ methods ขั้นสูงที่ช่วยในการจัดการข้อความอย่างมืออาชีพ รวมถึงการทำงานกับ Unicode และ Grapheme Clusters

---

## 1. scan - ค้นหาและเก็บผลลัพธ์

`scan` ใช้สำหรับค้นหา pattern ทั้งหมดในสตริงและคืนค่าเป็น Array

```crystal
# scan พื้นฐาน
str = "hello world hello crystal"
results = str.scan("hello")
puts results.inspect  # => ["hello", "hello"]

# scan ด้วย Regex
str = "abc123def456ghi789"
numbers = str.scan(/\d+/)
puts numbers.inspect  # => ["123", "456", "789"]

# scan ที่คืนค่า Array of Array (กรณีมี capture groups)
str = "2024-01-15, 2024-02-20, 2024-03-25"
dates = str.scan(/(\d{4})-(\d{2})-(\d{2})/)
dates.each do |match|
  puts "Year: #{match[1]}, Month: #{match[2]}, Day: #{match[3]}"
end

# นับจำนวน words
text = "the quick brown fox jumps over the lazy dog"
words = text.scan(/\b\w+\b/)
puts "Word count: #{words.size}"  # => Word count: 9

# หา email addresses
emails_text = "ติดต่อ john@example.com หรือ jane@test.org"
emails = emails_text.scan(/[\w.]+@[\w.]+\.\w+/)
puts emails.inspect  # => ["john@example.com", "jane@test.org"]
```

---

## 2. gsub กับ Block

`gsub` พร้อม block ช่วยให้แปลงแต่ละ match ได้อย่างยืดหยุ่น

```crystal
# gsub พื้นฐานกับ block
str = "hello world"
result = str.gsub(/\b\w/) { |match| match.upcase }
puts result  # => "Hello World"

# แปลงตัวเลขเป็นคำ
def number_to_thai(n : String) : String
  map = {"1" => "หนึ่ง", "2" => "สอง", "3" => "สาม",
         "4" => "สี่", "5" => "ห้า", "6" => "หก",
         "7" => "เจ็ด", "8" => "แปด", "9" => "เก้า", "0" => "ศูนย์"}
  map[n]? || n
end

str = "I have 3 cats and 7 dogs"
result = str.gsub(/\d/) { |m| number_to_thai(m) }
puts result  # => "I have สาม cats and เจ็ด dogs"

# encode HTML entities
def html_escape(str : String) : String
  str.gsub(/[&<>"']/) do |char|
    case char
    when "&" then "&amp;"
    when "<" then "&lt;"
    when ">" then "&gt;"
    when "\"" then "&quot;"
    when "'" then "&#39;"
    else char
    end
  end
end

html = "<script>alert('XSS')</script>"
puts html_escape(html)
# => &lt;script&gt;alert(&#39;XSS&#39;)&lt;/script&gt;

# gsub กับ named captures
str = "John Smith, Jane Doe, Bob Jones"
result = str.gsub(/(?<first>\w+) (?<last>\w+)/) do |match|
  "#{$~["last"]}, #{$~["first"]}"
end
puts result  # => "Smith, John, Doe, Jane, Jones, Bob"

# Markdown bold ไปเป็น HTML
def markdown_to_html(text : String) : String
  text
    .gsub(/\*\*(.+?)\*\*/) { "<strong>#{$~[1]}</strong>" }
    .gsub(/\*(.+?)\*/) { "<em>#{$~[1]}</em>" }
    .gsub(/`(.+?)`/) { "<code>#{$~[1]}</code>" }
end

md = "This is **bold** and *italic* and `code`"
puts markdown_to_html(md)
# => This is <strong>bold</strong> and <em>italic</em> and <code>code</code>
```

---

## 3. tr - แปลงตัวอักษร

`tr` แปลงตัวอักษรแบบตัวต่อตัว คล้าย `tr` command ใน Unix

```crystal
# tr พื้นฐาน
str = "hello"
puts str.tr("aeiou", "*")   # => "h*ll*"
puts str.tr("el", "ip")     # => "hippo"

# เปลี่ยน lowercase เป็น uppercase แบบ manual
puts "hello world".tr("a-z", "A-Z")  # => "HELLO WORLD"

# ROT13 encoding
def rot13(str : String) : String
  str.tr("A-Za-z", "N-ZA-Mn-za-m")
end

puts rot13("Hello World")   # => "Uryyb Jbeyq"
puts rot13(rot13("Hello World"))  # => "Hello World" (decode)

# แทนที่ตัวเลข
puts "ph0n3 numb3r".tr("0-9", "zero-nine is wrong here")
# tr ใช้แต่ละตัวอักษร ไม่ใช่ string ทั้งหมด

# ลบตัวอักษรที่ไม่ต้องการด้วย tr กับ ^ (complement)
puts "abc123def".tr("^0-9", "")  # => "123" (เก็บเฉพาะตัวเลข)
puts "abc123def".tr("0-9", "")   # => "abcdef" (ลบตัวเลข)

# แปลง accented characters
def remove_accents(str : String) : String
  str.tr("àáâãäåæèéêëìíîïòóôõöùúûü",
         "aaaaaaaeeeeiiiioooooouuuu")
end

puts remove_accents("café")    # => "cafe"
puts remove_accents("naïve")   # => "naive"
```

---

## 4. squeeze - ลด repeated characters

```crystal
# squeeze พื้นฐาน
puts "aaabbbccc".squeeze       # => "abc"
puts "aabbcc".squeeze("a")     # => "abbcc" (squeeze เฉพาะ a)
puts "  hello   world  ".squeeze(" ")  # => " hello world "

# ทำความสะอาดข้อความ
def clean_spaces(str : String) : String
  str.strip.squeeze(" ")
end

puts clean_spaces("  hello   world  ")  # => "hello world"

# ใช้ใน URL normalization
def normalize_url_path(path : String) : String
  path.squeeze("/").gsub(/\/$/, "")
end

puts normalize_url_path("//foo///bar//")  # => "/foo/bar"

# รวมกับ methods อื่นๆ
def normalize_text(str : String) : String
  str
    .strip
    .squeeze(" ")
    .downcase
end

puts normalize_text("  HELLO   WORLD  ")  # => "hello world"
```

---

## 5. delete - ลบตัวอักษร

```crystal
# delete พื้นฐาน
puts "hello world".delete("lo")   # => "he wrd"
puts "hello".delete("a-e")        # => "hllo"  (ลบ a ถึง e)
puts "hello123".delete("0-9")     # => "hello"

# ลบ non-alphanumeric
def keep_alphanumeric(str : String) : String
  str.delete("^a-zA-Z0-9")
end

puts keep_alphanumeric("Hello, World! 123")  # => "HelloWorld123"

# ลบ whitespace ทั้งหมด
puts "h e l l o".delete(" \t\n")  # => "hello"

# สร้าง slug จาก title
def to_slug(title : String) : String
  title
    .downcase
    .gsub(/[^a-z0-9\s-]/, "")
    .gsub(/\s+/, "-")
    .delete("-")  # จะไม่ทำแบบนี้จริงๆ แต่แสดงการใช้ delete
end

# ตัวอย่างจริงของ slug
def make_slug(title : String) : String
  title
    .downcase
    .gsub(/[^\w\s-]/, "")
    .gsub(/[\s_]+/, "-")
    .gsub(/^-+|-+$/, "")
end

puts make_slug("Hello, World! This is Crystal")
# => "hello-world-this-is-crystal"
```

---

## 6. String Formatting

```crystal
# sprintf style formatting
formatted = sprintf("Name: %-20s Age: %3d", "Alice", 25)
puts formatted  # => "Name: Alice                Age:  25"

# % operator
puts "Hello, %s!" % "World"           # => "Hello, World!"
puts "Pi is %.2f" % 3.14159           # => "Pi is 3.14"
puts "%d + %d = %d" % {1, 2, 3}      # => "1 + 2 = 3"

# padding
puts "%10s" % "right"    # => "     right"
puts "%-10s" % "left"    # => "left      "
puts "%010d" % 42        # => "0000000042"

# number formatting
puts "%e" % 123456.789   # => "1.234568e+05"
puts "%g" % 0.0001234    # => "0.0001234"
puts "%x" % 255          # => "ff"
puts "%X" % 255          # => "FF"
puts "%o" % 8            # => "10"
puts "%b" % 10           # => "1010"

# ตารางข้อมูล
data = [
  {"Alice", 25, 75000.50},
  {"Bob", 30, 85000.00},
  {"Charlie", 28, 92500.75},
]

puts "%-15s %5s %12s" % {"Name", "Age", "Salary"}
puts "-" * 35
data.each do |name, age, salary|
  puts "%-15s %5d %12.2f" % {name, age, salary}
end
```

---

## 7. each_char - วนลูปผ่านแต่ละตัวอักษร

```crystal
# each_char พื้นฐาน
"hello".each_char do |char|
  print "#{char} "
end
puts  # => "h e l l o "

# นับตัวอักษร
str = "Hello, World!"
char_count = Hash(Char, Int32).new(0)
str.each_char { |c| char_count[c] += 1 }

char_count.each do |char, count|
  puts "'#{char}': #{count}"
end

# ตรวจสอบ palindrome
def palindrome?(str : String) : Bool
  cleaned = str.downcase.delete("^a-z")
  chars = [] of Char
  cleaned.each_char { |c| chars << c }
  chars == chars.reverse
end

puts palindrome?("racecar")    # => true
puts palindrome?("A man a plan a canal Panama")  # => true
puts palindrome?("hello")      # => false

# แปลงเป็น char array
chars = "crystal".chars  # ใช้ .chars แทน each_char + collect
puts chars.inspect  # => ['c', 'r', 'y', 's', 't', 'a', 'l']

# reverse string ด้วย chars
puts "hello".chars.reverse.join  # => "olleh"

# uppercase first letter of each word
def title_case(str : String) : String
  str.split.map do |word|
    chars = word.chars
    if chars.empty?
      word
    else
      chars[0].upcase.to_s + chars[1..].join
    end
  end.join(" ")
end

puts title_case("the quick brown fox")  # => "The Quick Brown Fox"
```

---

## 8. each_line - วนลูปผ่านแต่ละบรรทัด

```crystal
# each_line พื้นฐาน
text = "line 1\nline 2\nline 3"
text.each_line do |line|
  puts ">> #{line}"
end

# นับบรรทัด
def count_lines(text : String) : Int32
  count = 0
  text.each_line { count += 1 }
  count
end

# ประมวลผล log file
log = <<-LOG
  2024-01-15 10:00:00 INFO Server started
  2024-01-15 10:01:00 DEBUG Processing request
  2024-01-15 10:01:01 ERROR Connection failed
  2024-01-15 10:02:00 INFO Request completed
  LOG

errors = [] of String
log.each_line do |line|
  errors << line.strip if line.includes?("ERROR")
end

puts "Errors found:"
errors.each { |e| puts "  #{e}" }

# แปลง lines เป็น array
lines = "a\nb\nc".lines
puts lines.inspect  # => ["a\n", "b\n", "c"]

# lines ที่ไม่มี newline (chomp: true)
lines_clean = "a\nb\nc".lines(chomp: true)
puts lines_clean.inspect  # => ["a", "b", "c"]

# process each line with index
text.each_line.with_index(1) do |line, num|
  puts "#{num}: #{line}"
end
```

---

## 9. String.build - สร้าง String อย่างมีประสิทธิภาพ

```crystal
# String.build พื้นฐาน
result = String.build do |io|
  io << "Hello"
  io << ", "
  io << "World"
  io << "!"
end
puts result  # => "Hello, World!"

# ดีกว่าการ concatenate
# BAD: สร้าง String ใหม่ทุกครั้ง
result = ""
100.times { |i| result += i.to_s }

# GOOD: ใช้ String.build
result = String.build do |io|
  100.times { |i| io << i }
end

# สร้าง HTML
def build_html_table(headers : Array(String), rows : Array(Array(String))) : String
  String.build do |io|
    io << "<table>\n"
    io << "  <thead>\n    <tr>\n"
    headers.each { |h| io << "      <th>#{h}</th>\n" }
    io << "    </tr>\n  </thead>\n"
    io << "  <tbody>\n"
    rows.each do |row|
      io << "    <tr>\n"
      row.each { |cell| io << "      <td>#{cell}</td>\n" }
      io << "    </tr>\n"
    end
    io << "  </tbody>\n</table>"
  end
end

headers = ["Name", "Age", "City"]
rows = [
  ["Alice", "25", "Bangkok"],
  ["Bob", "30", "Chiang Mai"],
]

puts build_html_table(headers, rows)

# String.build กับ conditional
def build_report(items : Array(String), show_numbers : Bool = true) : String
  String.build do |io|
    io << "Report:\n"
    items.each_with_index do |item, i|
      if show_numbers
        io << "#{i + 1}. #{item}\n"
      else
        io << "- #{item}\n"
      end
    end
  end
end

items = ["First item", "Second item", "Third item"]
puts build_report(items)
puts build_report(items, show_numbers: false)
```

---

## 10. Grapheme Clusters

```crystal
# ความแตกต่างระหว่าง chars และ grapheme clusters
str = "é"  # e + combining accent (2 codepoints)
puts str.size        # จำนวน characters (codepoints)
puts str.bytesize    # จำนวน bytes

# emoji ที่ซับซ้อน
emoji = "👨‍👩‍👧‍👦"  # family emoji = หลาย codepoints
puts emoji.size      # > 1 (codepoints)
puts emoji.bytesize  # bytes จำนวนมาก

# การนับ grapheme clusters ที่ถูกต้อง
# Crystal มี String#grapheme_size
str = "Hello 👋"
puts str.size             # codepoints
puts str.grapheme_size    # grapheme clusters (what user sees)

# วนลูปผ่าน grapheme clusters
"Hello 🌍".each_grapheme do |g|
  print "[#{g}]"
end
puts

# ตัดสตริงตาม grapheme clusters
def truncate_graphemes(str : String, max : Int32, suffix : String = "...") : String
  graphemes = [] of String
  str.each_grapheme { |g| graphemes << g }
  
  if graphemes.size <= max
    str
  else
    graphemes[0, max - suffix.size].join + suffix
  end
end

puts truncate_graphemes("Hello 🌍 World", 8)  # => "Hello 🌍..."
```

---

## 11. Unicode Normalization

```crystal
# Unicode normalization
# NFC: Canonical Decomposition followed by Canonical Composition
# NFD: Canonical Decomposition
# NFKC: Compatibility Decomposition followed by Canonical Composition
# NFKD: Compatibility Decomposition

# ตัวอย่าง: "é" สามารถแทนได้สองแบบ
# 1. U+00E9 (precomposed)
# 2. U+0065 U+0301 (e + combining accent)

e_precomposed = "é"    # é precomposed
e_decomposed = "é"  # e + combining accent

puts e_precomposed == e_decomposed  # => false (different bytes)
puts e_precomposed.size             # => 1
puts e_decomposed.size              # => 2

# ใช้ Unicode.normalize
require "unicode/collation" # ถ้าต้องการ collation

# การ normalize สตริงไทย
thai = "กา"  # ก + า (สระ)
puts thai.size    # ขึ้นอยู่กับ encoding

# String comparison ที่ถูกต้องสำหรับ Unicode
def unicode_equal?(s1 : String, s2 : String) : Bool
  # normalize ก่อน compare
  s1.unicode_normalize == s2.unicode_normalize
end

# การ normalize ใน Crystal
str = "café"
nfc = str.unicode_normalize(:nfc)
nfd = str.unicode_normalize(:nfd)

puts nfc.size  # ขนาดอาจต่างกัน
puts nfd.size

# ทำ case-insensitive comparison ที่ถูกต้อง
def case_insensitive_equal?(s1 : String, s2 : String) : Bool
  s1.unicode_normalize(:nfkc).downcase == s2.unicode_normalize(:nfkc).downcase
end

puts case_insensitive_equal?("CAFÉ", "café")  # => true
```

---

## 12. Methods เพิ่มเติมที่มีประโยชน์

```crystal
# center, ljust, rjust
puts "hello".center(11)       # => "   hello   "
puts "hello".center(11, "-")  # => "---hello---"
puts "hello".ljust(10)        # => "hello     "
puts "hello".rjust(10)        # => "     hello"
puts "hello".rjust(10, "0")   # => "00000hello"

# count - นับตัวอักษร
puts "hello".count("l")      # => 2
puts "hello".count("aeiou")  # => 2 (สระ)

# start_with? / end_with?
puts "hello world".starts_with?("hello")   # => true
puts "hello world".ends_with?("world")     # => true
puts "hello world".starts_with?("world")  # => false

# includes?
puts "hello world".includes?("world")  # => true

# index / rindex
str = "hello world hello"
puts str.index("hello")   # => 0
puts str.rindex("hello")  # => 12
puts str.index("xyz")     # => nil

# sub - แทนที่แค่ครั้งแรก
puts "hello hello".sub("hello", "bye")   # => "bye hello"
puts "hello hello".gsub("hello", "bye")  # => "bye bye"

# chomp / chop / strip
puts "hello\n".chomp     # => "hello"
puts "hello\r\n".chomp   # => "hello"
puts "hello!".chop       # => "hello"
puts "  hello  ".strip   # => "hello"
puts "  hello  ".lstrip  # => "hello  "
puts "  hello  ".rstrip  # => "  hello"

# upcase / downcase / capitalize / swapcase
puts "hello WORLD".upcase     # => "HELLO WORLD"
puts "hello WORLD".downcase   # => "hello world"
puts "hello WORLD".capitalize # => "Hello world"
puts "hello WORLD".swapcase   # => "HELLO world"

# reverse
puts "hello".reverse  # => "olleh"

# split
puts "a,b,c".split(",").inspect   # => ["a", "b", "c"]
puts "a  b  c".split.inspect      # => ["a", "b", "c"]
puts "a,b,c".split(",", 2).inspect  # => ["a", "b,c"]

# join
puts ["a", "b", "c"].join(", ")  # => "a, b, c"
puts ["a", "b", "c"].join        # => "abc"
```

---

## 13. String Immutability และ Performance

```crystal
# String ใน Crystal เป็น immutable
str = "hello"
# str[0] = 'H'  # => Error!

# ต้องสร้าง String ใหม่
str = "H" + str[1..]
puts str  # => "Hello"

# String.build สำหรับ performance
# BAD: O(n²) เพราะสร้าง string ใหม่ทุกครั้ง
def slow_build(n : Int32) : String
  result = ""
  n.times { |i| result += i.to_s + "," }
  result
end

# GOOD: O(n)
def fast_build(n : Int32) : String
  String.build do |io|
    n.times { |i| io << i << "," }
  end
end

# วัดเวลา
require "benchmark"

Benchmark.ips do |x|
  x.report("slow") { slow_build(1000) }
  x.report("fast") { fast_build(1000) }
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `word_frequency(text)` ที่รับข้อความและคืนค่า Hash ที่มีคำเป็น key และจำนวนครั้งที่ปรากฏเป็น value โดยใช้ `scan` หรือ `split`

### แบบฝึกหัดที่ 2
เขียนฟังก์ชัน `caesar_cipher(text, shift)` ที่เข้ารหัสข้อความโดยเลื่อนตัวอักษรตาม shift โดยใช้ `tr`

### แบบฝึกหัดที่ 3
เขียนฟังก์ชัน `format_table(data)` ที่รับ Array of Array และแสดงเป็นตารางที่จัดระเบียบแล้วโดยใช้ `String.build` และ `ljust`/`rjust`

### แบบฝึกหัดที่ 4
เขียนฟังก์ชัน `highlight_matches(text, pattern)` ที่รับสตริงและ Regex pattern แล้วคืนสตริงที่ส่วน match ถูกล้อมด้วย `[` และ `]` โดยใช้ `gsub` กับ block

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
def word_frequency(text : String) : Hash(String, Int32)
  freq = Hash(String, Int32).new(0)
  text.downcase.scan(/\b\w+\b/).each do |word|
    freq[word[0]] += 1
  end
  freq
end

freq = word_frequency("the cat sat on the mat the cat")
freq.to_a.sort_by { |_, v| -v }.each do |word, count|
  puts "#{word}: #{count}"
end

# แบบฝึกหัดที่ 2
def caesar_cipher(text : String, shift : Int32) : String
  shift = ((shift % 26) + 26) % 26
  from = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz"
  to = from[shift..25] + from[0..shift-1] + from[26+shift..51] + from[26..26+shift-1]
  text.tr(from, to)
end

puts caesar_cipher("Hello, World!", 3)   # => "Khoor, Zruog!"
puts caesar_cipher("Khoor, Zruog!", -3)  # => "Hello, World!"

# แบบฝึกหัดที่ 3
def format_table(data : Array(Array(String))) : String
  return "" if data.empty?
  
  # หาความกว้างสูงสุดของแต่ละคอลัมน์
  col_count = data.map(&.size).max
  col_widths = Array.new(col_count, 0)
  data.each do |row|
    row.each_with_index do |cell, i|
      col_widths[i] = [col_widths[i], cell.size].max
    end
  end
  
  String.build do |io|
    data.each_with_index do |row, row_idx|
      row.each_with_index do |cell, i|
        io << cell.ljust(col_widths[i])
        io << " | " unless i == row.size - 1
      end
      io << "\n"
      if row_idx == 0
        col_widths.each_with_index do |w, i|
          io << "-" * w
          io << "-+-" unless i == col_widths.size - 1
        end
        io << "\n"
      end
    end
  end
end

data = [
  ["Name", "Age", "City"],
  ["Alice", "25", "Bangkok"],
  ["Bob", "30", "Chiang Mai"],
  ["Charlie", "28", "Phuket"],
]
puts format_table(data)

# แบบฝึกหัดที่ 4
def highlight_matches(text : String, pattern : Regex) : String
  text.gsub(pattern) { |match| "[#{match}]" }
end

puts highlight_matches("the cat sat on the mat", /\bthe\b/)
# => "[the] cat sat on [the] mat"

puts highlight_matches("abc123def456", /\d+/)
# => "abc[123]def[456]"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **scan** - ค้นหาและเก็บ matches ทั้งหมดในสตริง
2. **gsub กับ block** - แปลงแต่ละ match ด้วย logic ที่ซับซ้อน
3. **tr** - แปลงตัวอักษรแบบตัวต่อตัว
4. **squeeze** - ลดตัวอักษรที่ซ้ำกันติดกัน
5. **delete** - ลบตัวอักษรที่ระบุ
6. **String formatting** - จัดรูปแบบสตริงด้วย sprintf และ %
7. **each_char** - วนลูปผ่านแต่ละ character
8. **each_line** - วนลูปผ่านแต่ละบรรทัด
9. **String.build** - สร้าง String อย่างมีประสิทธิภาพ
10. **Grapheme Clusters** - จัดการ Unicode ที่ซับซ้อน
11. **Unicode Normalization** - ทำให้ Unicode representation เป็นมาตรฐาน

String methods เหล่านี้เป็นเครื่องมือสำคัญในการจัดการข้อความใน Crystal ควรฝึกใช้บ่อยๆ เพื่อให้เขียนโค้ดได้อย่างมีประสิทธิภาพ
