# Part 66: Char และ Character Operations

## บทนำ

`Char` เป็น type ที่แทน Unicode codepoint เดียวใน Crystal Char เป็น value type (struct) ที่มีขนาด 32 bits ทำให้รองรับ Unicode characters ทั้งหมด

---

## 1. Char Literals และพื้นฐาน

```crystal
# Char literal ใช้ single quotes
c = 'A'
puts c        # => A
puts c.class  # => Char

# Unicode chars
thai = 'ก'
puts thai.class    # => Char
puts thai.ord      # => 3585 (Unicode codepoint)

# Escape sequences
newline  = '\n'   # newline
tab      = '\t'   # tab
null     = '\0'   # null
backslash = '\\'  # backslash
quote    = '\''   # single quote

# Unicode escape
heart = '❤'   # ❤
star  = '★'   # ★
puts heart  # => ❤
puts star   # => ★

# Char สร้างจาก codepoint
puts 65.chr    # => A
puts 0x0E01.chr  # => ก
puts 0x1F600.chr # => 😀

# ตรวจสอบ Char
puts 'A'.class   # => Char
puts "A"[0].class  # => Char (accessing string by index)
```

---

## 2. ord และ chr

```crystal
# ord: Char -> Int32 (codepoint)
puts 'A'.ord      # => 65
puts 'a'.ord      # => 97
puts '0'.ord      # => 48
puts 'ก'.ord      # => 3585
puts '😀'.ord     # => 128512

# chr: Int32 -> Char (codepoint to char)
puts 65.chr       # => A
puts 97.chr       # => a
puts 48.chr       # => 0
puts 3585.chr     # => ก
puts 128512.chr   # => 😀

# ใช้ ord และ chr สำหรับ arithmetic
# เลื่อน alphabet
def shift_char(c : Char, n : Int32) : Char
  if c.ascii_letter?
    base = c.uppercase? ? 'A'.ord : 'a'.ord
    ((c.ord - base + n) % 26 + base).chr
  else
    c
  end
end

puts shift_char('A', 1)   # => B
puts shift_char('Z', 1)   # => A (wrap around)
puts shift_char('a', 3)   # => d
puts shift_char('!', 1)   # => ! (unchanged, not letter)

# Caesar cipher
def caesar(text : String, shift : Int32) : String
  text.chars.map { |c| shift_char(c, shift) }.join
end

puts caesar("Hello World", 13)  # ROT13
puts caesar(caesar("Hello World", 13), -13)  # decode

# เปรียบเทียบ chars
puts 'A' < 'B'    # => true
puts 'Z' > 'A'    # => true
puts 'a'.ord - 'A'.ord  # => 32 (difference between cases)
```

---

## 3. alpha?, digit?, whitespace?

```crystal
# alpha? - ตัวอักษร (a-z, A-Z)
puts 'a'.alpha?     # => true
puts 'Z'.alpha?     # => true
puts '1'.alpha?     # => false
puts ' '.alpha?     # => false
puts 'ก'.alpha?     # => false (non-ASCII)

# alphanumeric?
puts 'a'.alphanumeric?  # => true
puts '1'.alphanumeric?  # => true
puts ' '.alphanumeric?  # => false

# digit? - ตัวเลข (0-9)
puts '0'.ascii_number?  # => true
puts '9'.ascii_number?  # => true
puts 'a'.ascii_number?  # => false

# whitespace? - ช่องว่าง
puts ' '.whitespace?    # => true
puts '\t'.whitespace?   # => true
puts '\n'.whitespace?   # => true
puts 'a'.whitespace?    # => false

# ascii? - ASCII character (0-127)
puts 'a'.ascii?    # => true
puts 'ก'.ascii?    # => false
puts ' '.ascii?    # => true

# เขียน character classifier
def classify_char(c : Char) : String
  if c.ascii_letter?
    c.uppercase? ? "uppercase letter" : "lowercase letter"
  elsif c.ascii_number?
    "digit"
  elsif c.whitespace?
    "whitespace"
  elsif c.ascii?
    "ASCII punctuation/symbol"
  else
    "non-ASCII (Unicode)"
  end
end

['A', 'a', '5', ' ', '\t', 'ก', '@', '€'].each do |c|
  puts "'#{c}' (U+#{c.ord.to_s(16).upcase}): #{classify_char(c)}"
end
```

---

## 4. uppercase? / lowercase?

```crystal
# uppercase? / lowercase? สำหรับ ASCII
puts 'A'.uppercase?   # => true
puts 'a'.uppercase?   # => false
puts 'A'.lowercase?   # => false
puts 'a'.lowercase?   # => true
puts '1'.uppercase?   # => false
puts '1'.lowercase?   # => false

# upcase / downcase
puts 'a'.upcase   # => A
puts 'A'.downcase # => a
puts 'ก'.upcase   # => ก (ไม่เปลี่ยน, ภาษาไทยไม่มี case)

# ตรวจสอบ case
def title_case_char?(c : Char, prev : Char?) : Bool
  # uppercase ถ้าเป็น char แรก หรือหลัง space
  c.ascii_letter? && (prev.nil? || prev.whitespace?)
end

# สร้าง title case
def to_title_case(str : String) : String
  prev = nil
  String.build do |io|
    str.each_char do |c|
      if title_case_char?(c, prev)
        io << c.upcase
      else
        io << c
      end
      prev = c
    end
  end
end

puts to_title_case("hello world foo bar")  # => "Hello World Foo Bar"

# count uppercase/lowercase
def char_case_stats(str : String) : {upper: Int32, lower: Int32, other: Int32}
  upper = str.count { |c| c.uppercase? }
  lower = str.count { |c| c.lowercase? }
  other = str.size - upper - lower
  {upper: upper, lower: lower, other: other}
end

stats = char_case_stats("Hello World 123!")
puts "Upper: #{stats[:upper]}, Lower: #{stats[:lower]}, Other: #{stats[:other]}"
# => Upper: 2, Lower: 8, Other: 6
```

---

## 5. Unicode Categories

```crystal
# Crystal's Char มี methods สำหรับ Unicode categories

# letter? - Unicode letter (รวม non-ASCII)
puts 'a'.ascii_letter?    # ASCII letter
puts 'ก'.ascii_letter?    # => false (non-ASCII)

# ตรวจสอบ Thai characters
def thai_char?(c : Char) : Bool
  c.ord.in?(0x0E00..0x0E7F)
end

puts thai_char?('ก')  # => true
puts thai_char?('า')  # => true
puts thai_char?('A')  # => false

# ตรวจสอบ Unicode ranges
def char_unicode_block(c : Char) : String
  case c.ord
  when 0x0000..0x007F then "Basic Latin"
  when 0x0080..0x00FF then "Latin-1 Supplement"
  when 0x0100..0x017F then "Latin Extended-A"
  when 0x0E00..0x0E7F then "Thai"
  when 0x3040..0x309F then "Hiragana"
  when 0x30A0..0x30FF then "Katakana"
  when 0x4E00..0x9FFF then "CJK Unified Ideographs"
  when 0x1F600..0x1F64F then "Emoticons"
  when 0x1F300..0x1F5FF then "Misc Symbols and Pictographs"
  when 0x1F900..0x1F9FF then "Supplemental Symbols"
  else "Other Unicode Block"
  end
end

['A', 'ก', 'あ', '漢', '😀', '🌍', '€'].each do |c|
  puts "'#{c}' U+#{c.ord.to_s(16).upcase.rjust(4, '0')}: #{char_unicode_block(c)}"
end

# ตรวจสอบ ASCII control characters
def control_char?(c : Char) : Bool
  (c.ord < 32 || c.ord == 127) && c.ascii?
end

puts control_char?('\n')  # => true
puts control_char?('\t')  # => true
puts control_char?('A')   # => false

# printable?
def printable_char?(c : Char) : Bool
  !control_char?(c) && !c.whitespace?
end

puts printable_char?('A')   # => true
puts printable_char?('\n')  # => false
puts printable_char?(' ')   # => false (whitespace)
```

---

## 6. Char Arithmetic

```crystal
# Char สามารถทำ arithmetic ผ่าน ord/chr
def next_char(c : Char) : Char
  (c.ord + 1).chr
end

def prev_char(c : Char) : Char
  (c.ord - 1).chr
end

puts next_char('A')  # => B
puts next_char('z')  # => {  (next ASCII after z)
puts prev_char('B')  # => A

# Range ของ chars
('A'..'Z').each { |c| print c }
puts  # => ABCDEFGHIJKLMNOPQRSTUVWXYZ

('a'..'z').each { |c| print c }
puts  # => abcdefghijklmnopqrstuvwxyz

('0'..'9').each { |c| print c }
puts  # => 0123456789

# ใช้ Char range ใน case/when
def classify_ascii(c : Char) : String
  case c
  when 'A'..'Z' then "uppercase"
  when 'a'..'z' then "lowercase"
  when '0'..'9' then "digit"
  when ' ', '\t', '\n', '\r' then "whitespace"
  when '!'..'/' then "punctuation"
  else "other"
  end
end

['A', 'z', '5', ' ', '!', 'ก'].each do |c|
  puts "'#{c}': #{classify_ascii(c)}"
end

# สร้าง alphabet variants
def reverse_alphabet_map : Hash(Char, Char)
  map = {} of Char => Char
  ('a'..'z').each.with_index do |c, i|
    map[c] = ('z'.ord - i).chr
    map[c.upcase] = ('Z'.ord - i).chr
  end
  map
end

ATBASH = reverse_alphabet_map

def atbash_cipher(text : String) : String
  text.chars.map { |c| ATBASH[c]? || c }.join
end

puts atbash_cipher("Hello World")  # => Svool Dliow
puts atbash_cipher(atbash_cipher("Hello World"))  # => Hello World (symmetric)
```

---

## 7. Char to String Conversion

```crystal
# Char -> String
c = 'A'
puts c.to_s          # => "A"
puts c.to_s.class    # => String

# String -> Char (เมื่อมีตัวอักษรเดียว)
str = "A"
puts str[0]          # => A (Char)
puts str[0].class    # => Char

# char to string ด้วยวิธีต่างๆ
c = 'ก'
s1 = c.to_s            # => "ก"
s2 = String.new(c)     # => "ก" (ถ้า API support)
s3 = "#{c}"            # => "ก" (interpolation)

# Array of chars -> String
chars = ['H', 'e', 'l', 'l', 'o']
str = chars.join         # => "Hello"
str2 = String.new(chars) # => "Hello" (ถ้า API support)

# สร้าง string จาก char
def repeat_char(c : Char, n : Int32) : String
  c.to_s * n
end

puts repeat_char('=', 20)  # => ====================
puts repeat_char('ก', 5)   # => กกกกก

# แปลง Char เป็น เลขฐานต่างๆ
c = 'A'
puts "Char: #{c}"
puts "Decimal: #{c.ord}"
puts "Hex: 0x#{c.ord.to_s(16).upcase}"
puts "Binary: #{c.ord.to_s(2)}"
puts "Octal: #{c.ord.to_s(8)}"

# Char ใน Array
chars = "Hello สวัสดี".chars
puts chars.inspect  # Array of Chars

# กรอง chars
letters_only = "Hello, World! 123".chars.select(&.ascii_letter?)
puts letters_only.join  # => "HelloWorld"

digits_only = "Hello 123 World 456".chars.select(&.ascii_number?)
puts digits_only.join  # => "123456"
```

---

## 8. Char Predicates

```crystal
# Built-in predicates
c = 'A'
puts c.ascii?           # ASCII? (0-127)
puts c.ascii_letter?    # ASCII letter? (a-z, A-Z)
puts c.ascii_number?    # ASCII digit? (0-9)
puts c.alphanumeric?    # Alphanumeric?
puts c.whitespace?      # Whitespace?
puts c.uppercase?       # Uppercase?
puts c.lowercase?       # Lowercase?

# Custom predicates
def vowel?(c : Char) : Bool
  "aeiouAEIOU".includes?(c)
end

def consonant?(c : Char) : Bool
  c.ascii_letter? && !vowel?(c)
end

def hex_digit?(c : Char) : Bool
  ('0'..'9').includes?(c) || ('a'..'f').includes?(c) || ('A'..'F').includes?(c)
end

"Hello World".each_char do |c|
  next unless c.ascii_letter?
  puts "#{c}: #{vowel?(c) ? "vowel" : "consonant"}"
end

# ตัวอย่างจริง: parser
def parse_number(str : String) : Int64?
  return nil if str.empty?
  
  neg = str[0] == '-'
  start = neg ? 1 : 0
  
  digits = str[start..]
  return nil unless digits.all? { |c| c.ascii_number? }
  return nil if digits.empty?
  
  result = digits.chars.reduce(0i64) { |acc, c| acc * 10 + (c.ord - '0'.ord) }
  neg ? -result : result
end

puts parse_number("12345").inspect    # => 12345
puts parse_number("-42").inspect      # => -42
puts parse_number("abc").inspect      # => nil
puts parse_number("12.3").inspect     # => nil (has decimal)
```

---

## 9. Character Encoding Operations

```crystal
# ดู bytes ของ Char
def char_bytes(c : Char) : Array(UInt8)
  c.to_s.bytes.to_a
end

['A', 'ก', '😀'].each do |c|
  bytes = char_bytes(c)
  puts "#{c.inspect}: #{bytes.map { |b| "0x#{b.to_s(16).upcase}" }.join(" ")}"
end
# 'A': 0x41
# 'ก': 0xE0 0xB8 0x81
# '😀': 0xF0 0x9F 0x98 0x80

# check byte length ของ char
def char_byte_length(c : Char) : Int32
  c.to_s.bytesize
end

puts char_byte_length('A')   # => 1
puts char_byte_length('ก')   # => 3
puts char_byte_length('😀')  # => 4

# สร้าง lookup table
def build_char_info_table(str : String)
  puts "%-10s %-10s %-10s %-20s" % {"Char", "Decimal", "Hex", "Bytes"}
  puts "-" * 55
  str.each_char do |c|
    bytes = char_bytes(c).map { |b| "0x#{b.to_s(16).upcase}" }.join(" ")
    puts "%-10s %-10d %-10s %-20s" % {c.inspect, c.ord, "U+#{c.ord.to_s(16).upcase.rjust(4, '0')}", bytes}
  end
end

build_char_info_table("AaกBb")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `morse_code(text)` ที่แปลงข้อความ ASCII เป็น Morse code โดยใช้ `Char` operations

### แบบฝึกหัดที่ 2
เขียนฟังก์ชัน `is_pangram(text)` ที่ตรวจสอบว่าข้อความมีตัวอักษร a-z ครบทุกตัวหรือไม่

### แบบฝึกหัดที่ 3
เขียน `char_frequency_histogram(text)` ที่แสดง histogram ของความถี่ของแต่ละตัวอักษร

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
MORSE = {
  'A' => ".-",   'B' => "-...", 'C' => "-.-.", 'D' => "-..",
  'E' => ".",    'F' => "..-.", 'G' => "--.",  'H' => "....",
  'I' => "..",   'J' => ".---", 'K' => "-.-",  'L' => ".-..",
  'M' => "--",   'N' => "-.",   'O' => "---",  'P' => ".--.",
  'Q' => "--.-", 'R' => ".-.",  'S' => "...",  'T' => "-",
  'U' => "..-",  'V' => "...-", 'W' => ".--",  'X' => "-..-",
  'Y' => "-.--", 'Z' => "--..", '0' => "-----", '1' => ".----",
  '2' => "..---", '3' => "...--", '4' => "....-", '5' => ".....",
  '6' => "-....", '7' => "--...", '8' => "---..", '9' => "----.",
}

def morse_code(text : String) : String
  text.upcase.chars.map do |c|
    if c == ' '
      "/"
    elsif code = MORSE[c]?
      code
    else
      "?"
    end
  end.join(" ")
end

puts morse_code("Hello World")
# => .... . .-.. .-.. --- / .-- --- .-. .-.. -..

# แบบฝึกหัดที่ 2
def pangram?(text : String) : Bool
  ('a'..'z').all? { |c| text.downcase.includes?(c) }
end

puts pangram?("The quick brown fox jumps over the lazy dog")  # => true
puts pangram?("Hello World")  # => false

# แบบฝึกหัดที่ 3
def char_frequency_histogram(text : String, max_width : Int32 = 40)
  freq = Hash(Char, Int32).new(0)
  text.downcase.each_char do |c|
    freq[c] += 1 if c.ascii_letter?
  end
  
  max_count = freq.values.max? || 0
  return if max_count == 0
  
  freq.to_a.sort_by { |c, _| c }.each do |char, count|
    bar_width = (count.to_f / max_count * max_width).round.to_i
    puts "#{char} #{count.to_s.rjust(3)} #{"|" * bar_width}"
  end
end

char_frequency_histogram("the quick brown fox jumps over the lazy dog")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Char literal** ด้วย single quotes และ escape sequences
2. **ord** แปลง Char เป็น Int32 (codepoint)
3. **chr** แปลง Int32 เป็น Char
4. **Predicates**: `alpha?`, `digit?`, `whitespace?`, `uppercase?`, `lowercase?`
5. **Unicode categories** และ Unicode blocks
6. **Char arithmetic** ผ่าน ord/chr
7. **Char to String** conversion
8. **Character encoding** และ byte representation

Char เป็น type พื้นฐานที่สำคัญเมื่อต้องการจัดการตัวอักษรแบบ individual โดยเฉพาะใน text processing และ encoding operations
