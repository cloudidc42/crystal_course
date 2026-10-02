# Part 65: String Encoding และ Unicode

## บทนำ

Crystal ใช้ UTF-8 เป็น encoding หลักสำหรับ String ทุก String ใน Crystal คือ sequence of Unicode codepoints ที่เก็บใน UTF-8 format การเข้าใจ encoding และ Unicode ช่วยให้จัดการข้อความภาษาต่างๆ รวมถึง emoji ได้อย่างถูกต้อง

---

## 1. UTF-8 Strings พื้นฐาน

```crystal
# Crystal strings เป็น UTF-8
str = "สวัสดี"
puts str           # => สวัสดี
puts str.class    # => String
puts str.bytesize # จำนวน bytes (UTF-8)
puts str.size     # จำนวน characters (codepoints)

# ASCII text
ascii = "hello"
puts ascii.bytesize  # => 5 (1 byte per char)
puts ascii.size      # => 5

# Thai text
thai = "สวัสดี"
puts thai.bytesize   # > thai.size (multi-byte chars)
puts thai.size       # จำนวน chars ที่ผู้ใช้เห็น

# emoji
emoji = "Hello 🌍"
puts emoji.bytesize  # 9+ bytes
puts emoji.size      # 8 chars (H,e,l,l,o, ,🌍 = 7 codepoints + null?)

# แสดงความแตกต่าง
strings = ["hello", "สวัสดี", "こんにちは", "🎉🎊🎈"]
strings.each do |s|
  puts "#{s.inspect.ljust(20)} bytes: #{s.bytesize.to_s.rjust(3)}, chars: #{s.size}"
end
```

---

## 2. bytesize vs size

```crystal
# size = จำนวน Unicode codepoints
# bytesize = จำนวน bytes ใน UTF-8 encoding

# UTF-8 encoding:
# ASCII (U+0000 to U+007F): 1 byte
# U+0080 to U+07FF: 2 bytes
# U+0800 to U+FFFF: 3 bytes (รวมภาษาไทย, ญี่ปุ่น, จีน)
# U+10000 to U+10FFFF: 4 bytes (emoji, rare chars)

def encoding_info(str : String)
  puts "String: #{str.inspect}"
  puts "  size (codepoints): #{str.size}"
  puts "  bytesize: #{str.bytesize}"
  puts "  bytes per char avg: #{str.bytesize.to_f / str.size}"
  puts ""
end

encoding_info("hello")     # 1 byte/char
encoding_info("สวัสดี")    # ~3 bytes/char
encoding_info("😀")        # 4 bytes/char
encoding_info("こんにちは") # 3 bytes/char

# bytes method ดู raw bytes
"hello".bytes.each { |b| print "#{b} " }
puts  # => 104 101 108 108 111

"ก".bytes.each { |b| print "0x#{b.to_s(16)} " }
puts  # => 0xe0 0xb8 0x81 (3 bytes for ก in UTF-8)

# ตรวจสอบ valid UTF-8
def valid_utf8?(str : String) : Bool
  str.valid_encoding?
end

puts valid_utf8?("hello")   # => true
puts valid_utf8?("สวัสดี")  # => true

# สร้าง string จาก bytes
bytes = Bytes[104, 101, 108, 108, 111]  # "hello"
str = String.new(bytes)
puts str  # => "hello"
```

---

## 3. chars vs bytes

```crystal
# chars returns Array of chars (codepoints)
"hello".chars.each { |c| print "#{c} " }
puts  # => h e l l o

"สวัสดี".chars.each { |c| print "#{c} " }
puts  # => ส ว ั ส ด ี

# bytes returns Array of UInt8
"hello".bytes.inspect

# การวนลูปที่ถูกต้อง
str = "Hello สวัสดี 🌍"

# วนลูปผ่าน codepoints (correct for Unicode)
str.each_char.with_index do |char, i|
  puts "char[#{i}]: #{char.inspect} (U+#{char.ord.to_s(16).upcase.rjust(4, '0')})"
end

# การ index - ใช้ chars ไม่ใช่ bytes
str = "สวัสดี"
chars = str.chars
puts chars[0]  # => ส (first char)
puts chars[1]  # => ว (second char)

# ระวัง: str[i] ใน Crystal เป็น byte index
# ใช้ chars[i] สำหรับ char index
str = "สวัสดี"
# puts str[0]  # อาจไม่ใช่สิ่งที่คาดหวัง

# แปลง char array กลับเป็น string
chars = "hello".chars
reversed = chars.reverse.join
puts reversed  # => "olleh"

# Unicode codepoint
char = 'ก'
puts char.ord        # => 3585 (decimal)
puts char.ord.to_s(16)  # => "e01" (hex)

# สร้าง char จาก codepoint
puts 0x0E01.chr  # => ก
puts 65.chr      # => A
```

---

## 4. Unicode Normalization

```crystal
# Unicode บางครั้งแทนตัวอักษรเดียวกันได้หลายวิธี
# เช่น é สามารถเป็น:
# 1. U+00E9 (é precomposed - NFC)
# 2. U+0065 U+0301 (e + combining accent - NFD)

e_nfc = "é"     # é as single codepoint
e_nfd = "é"   # e + combining accent

puts e_nfc.size   # => 1
puts e_nfd.size   # => 2
puts e_nfc == e_nfd  # => false (different representations)
puts e_nfc.chars.map { |c| c.ord.to_s(16) }.inspect
# => ["e9"]
puts e_nfd.chars.map { |c| c.ord.to_s(16) }.inspect  
# => ["65", "301"]

# ใช้ unicode_normalize เพื่อ normalize
nfc_normalized = e_nfd.unicode_normalize(:nfc)
puts nfc_normalized == e_nfc  # => true

nfd_normalized = e_nfc.unicode_normalize(:nfd)
puts nfd_normalized == e_nfd  # => true

# Forms:
# :nfc  - Canonical Decomposition, Canonical Composition (ใช้บ่อยที่สุด)
# :nfd  - Canonical Decomposition
# :nfkc - Compatibility Decomposition, Canonical Composition
# :nfkd - Compatibility Decomposition

# NFKC ทำ compatibility normalization ด้วย
# เช่น ① -> 1, ＡＢＣ -> ABC
fullwidth = "ＡＢＣＤ"  # fullwidth Latin
puts fullwidth.unicode_normalize(:nfkc)  # => "ABCD"

ligature = "ﬁ"  # fi ligature
puts ligature.unicode_normalize(:nfkc)  # => "fi"

# Case-insensitive comparison ที่ถูกต้อง
def unicode_case_insensitive_equal?(a : String, b : String) : Bool
  a.unicode_normalize(:nfkc).downcase == b.unicode_normalize(:nfkc).downcase
end

puts unicode_case_insensitive_equal?("Café", "café")    # => true
puts unicode_case_insensitive_equal?("ＨＥＬＬＯ", "hello")  # => true

# การ sort ที่ถูกต้อง
words = ["café", "apple", "banane", "Âge"]
sorted_wrong = words.sort  # lexicographic byte order
sorted_correct = words.sort_by { |w| w.unicode_normalize(:nfkd).downcase }

puts "Wrong sort: #{sorted_wrong.inspect}"
puts "Correct sort: #{sorted_correct.inspect}"
```

---

## 5. Emoji Handling

```crystal
# Emoji อาจใช้หลาย codepoints
simple_emoji = "😀"    # single codepoint
puts simple_emoji.size  # => 1

# Emoji ที่ซับซ้อน: family emoji
family = "👨‍👩‍👧‍👦"
puts family.size          # หลาย codepoints
puts family.bytesize      # bytes มาก
puts family.grapheme_size # => 1 (1 visible unit)

# Flag emoji ก็ใช้หลาย codepoints
thailand_flag = "🇹🇭"
puts thailand_flag.size          # => 2 (regional indicators)
puts thailand_flag.grapheme_size # => 1 (1 visible flag)

# Skin tone modifier
wave = "👋"
wave_dark = "👋🏿"  # wave + dark skin tone
puts wave.size           # => 1
puts wave_dark.size      # => 2
puts wave_dark.grapheme_size  # => 1

# ความยาวที่ถูกต้องสำหรับ user display
def visible_length(str : String) : Int32
  str.grapheme_size
end

puts visible_length("Hello")   # => 5
puts visible_length("Hi 👋")   # => 4
puts visible_length("👨‍👩‍👧‍👦")    # => 1

# ตัดสตริงตาม visible characters
def truncate(str : String, max_visible : Int32, ellipsis : String = "...") : String
  graphemes = [] of String
  str.each_grapheme { |g| graphemes << g }
  
  return str if graphemes.size <= max_visible
  
  graphemes[0, max_visible - ellipsis.grapheme_size].join + ellipsis
end

puts truncate("Hello 👋 World 🌍", 8)
# => "Hello 👋 ..."

# นับ emoji ในข้อความ
def count_emoji(str : String) : Int32
  count = 0
  str.each_grapheme do |g|
    # emoji อยู่ใน range U+1F000+
    first_cp = g.chars.first?.try(&.ord) || 0
    count += 1 if first_cp >= 0x1F000 || (first_cp >= 0x2600 && first_cp <= 0x27BF)
  end
  count
end

puts count_emoji("Hello 😀 World 🌍 Yes! 🎉")  # => 3
```

---

## 6. String.build กับ IO สำหรับ Encoding

```crystal
# String.build ด้วย IO::Memory
result = String.build do |io|
  io << "Hello"
  io << " "
  io << "สวัสดี"
  io << " "
  io << "🌍"
end
puts result  # => "Hello สวัสดี 🌍"
puts result.valid_encoding?  # => true

# เขียน Unicode escapes
def write_unicode_escape(codepoint : Int32, io : IO)
  if codepoint <= 0xFFFF
    io.printf("\\u%04X", codepoint)
  else
    # Surrogate pair (สำหรับ > U+FFFF)
    cp = codepoint - 0x10000
    high = 0xD800 + (cp >> 10)
    low = 0xDC00 + (cp & 0x3FF)
    io.printf("\\u%04X\\u%04X", high, low)
  end
end

io = IO::Memory.new
write_unicode_escape(0x0E2A, io)  # ส
puts io.to_s  # => ส

# encode string เป็น unicode escapes
def to_unicode_escapes(str : String) : String
  String.build do |io|
    str.each_char do |c|
      if c.ord > 127
        io.printf("\\u%04X", c.ord)
      else
        io << c
      end
    end
  end
end

puts to_unicode_escapes("Hello สวัสดี")
# => Hello สวัสดี

# decode unicode escapes
def from_unicode_escapes(str : String) : String
  str.gsub(/\\u([0-9a-fA-F]{4})/) do
    $~[1].to_i(16).chr
  end
end

puts from_unicode_escapes("Hello \\u0E2A\\u0E27\\u0E31\\u0E2A\\u0E14\\u0E35")
# => Hello สวัสดี
```

---

## 7. Encoding Conversion

```crystal
# Crystal ไม่มี built-in encoding conversion library
# แต่เราสามารถทำ manual conversion ได้

# แปลง Windows-1252 bytes เป็น UTF-8 (ตัวอย่าง)
# (ในชีวิตจริงใช้ iconv หรือ libiconv)

# ตาราง Windows-1252 -> Unicode สำหรับ 0x80-0x9F
WIN1252_MAP = {
  0x80u8 => 0x20AC,  # €
  0x82u8 => 0x201A,  # ‚
  0x83u8 => 0x0192,  # ƒ
  0x84u8 => 0x201E,  # „
  0x85u8 => 0x2026,  # …
  0x86u8 => 0x2020,  # †
  0x87u8 => 0x2021,  # ‡
  0x88u8 => 0x02C6,  # ˆ
  0x89u8 => 0x2030,  # ‰
  0x8Au8 => 0x0160,  # Š
  0x8Bu8 => 0x2039,  # ‹
  0x8Cu8 => 0x0152,  # Œ
  0x8Eu8 => 0x017D,  # Ž
  0x91u8 => 0x2018,  # '
  0x92u8 => 0x2019,  # '
  0x93u8 => 0x201C,  # "
  0x94u8 => 0x201D,  # "
  0x95u8 => 0x2022,  # •
  0x96u8 => 0x2013,  # –
  0x97u8 => 0x2014,  # —
  0x98u8 => 0x02DC,  # ˜
  0x99u8 => 0x2122,  # ™
  0x9Au8 => 0x0161,  # š
  0x9Bu8 => 0x203A,  # ›
  0x9Cu8 => 0x0153,  # œ
  0x9Eu8 => 0x017E,  # ž
  0x9Fu8 => 0x0178,  # Ÿ
}

def win1252_to_utf8(bytes : Bytes) : String
  String.build do |io|
    bytes.each do |byte|
      if byte < 0x80
        io << byte.chr
      elsif byte >= 0xA0
        # Latin-1 supplement (direct Unicode codepoint)
        io << byte.to_i32.chr
      else
        # 0x80-0x9F: use Windows-1252 map
        if cp = WIN1252_MAP[byte]?
          io << cp.chr
        else
          io << "?"  # unmapped
        end
      end
    end
  end
end

# ทดสอบ
test_bytes = Bytes[72, 101, 108, 108, 111, 0x99, 33]  # Hello™!
puts win1252_to_utf8(test_bytes)  # => Hello™!

# ตรวจสอบ encoding
def detect_encoding_hint(str : String) : String
  has_bom = str.starts_with?("\xEF\xBB\xBF")
  
  if has_bom
    "UTF-8 with BOM"
  elsif str.valid_encoding?
    "UTF-8"
  else
    "Unknown/Invalid"
  end
end

# สร้าง UTF-8 string จาก codepoints
def codepoints_to_string(codepoints : Array(Int32)) : String
  String.build do |io|
    codepoints.each { |cp| io << cp.chr }
  end
end

puts codepoints_to_string([0x0E2A, 0x0E27, 0x0E31, 0x0E2A, 0x0E14, 0x0E35])
# => สวัสดี
```

---

## 8. Multi-language Text Processing

```crystal
# ตรวจสอบ script/language
def detect_script(str : String) : Array(String)
  scripts = [] of String
  
  str.each_char do |c|
    ord = c.ord
    script = case ord
    when 0x0000..0x007F then "Latin/ASCII"
    when 0x0E00..0x0E7F then "Thai"
    when 0x3040..0x30FF then "Japanese"
    when 0x4E00..0x9FFF then "CJK"
    when 0x0600..0x06FF then "Arabic"
    when 0x0400..0x04FF then "Cyrillic"
    when 0x0370..0x03FF then "Greek"
    when 0x0900..0x097F then "Devanagari"
    when 0x1F000..0x1FFFF then "Emoji"
    else "Other"
    end
    scripts << script unless scripts.includes?(script)
  end
  
  scripts.uniq
end

puts detect_script("Hello สวัสดี 🌍").inspect
# => ["Latin/ASCII", "Thai", "Emoji"]

puts detect_script("日本語テスト").inspect
# => ["CJK", "Japanese"]

# Word segmentation สำหรับ CJK (simplified)
def segment_cjk(text : String) : Array(String)
  # ง่ายๆ แค่แยกทีละ character สำหรับ CJK
  result = [] of String
  current = ""
  
  text.each_char do |c|
    if c.ord.in?(0x4E00..0x9FFF) || c.ord.in?(0x3040..0x30FF)
      unless current.empty?
        result << current
        current = ""
      end
      result << c.to_s
    elsif c == ' '
      unless current.empty?
        result << current
        current = ""
      end
    else
      current += c.to_s
    end
  end
  
  result << current unless current.empty?
  result
end

puts segment_cjk("Hello世界test日本語").inspect
# => ["Hello", "世", "界", "test", "日", "本", "語"]

# Text statistics
def text_stats(text : String) : Hash(String, Int32 | Float64)
  char_count = text.size
  byte_count = text.bytesize
  grapheme_count = text.grapheme_size
  word_count = text.scan(/\S+/).size
  line_count = text.lines.size
  
  {
    "characters" => char_count,
    "bytes" => byte_count,
    "graphemes" => grapheme_count,
    "words" => word_count,
    "lines" => line_count,
    "bytes_per_char" => (byte_count.to_f / char_count).round(2),
  } of String => Int32 | Float64
end

sample = "Hello สวัสดี 🌍\nSecond line"
stats = text_stats(sample)
stats.each { |k, v| puts "#{k}: #{v}" }
```

---

## 9. String Validation และ Sanitization

```crystal
# ตรวจสอบ valid UTF-8
def ensure_utf8(str : String) : String
  if str.valid_encoding?
    str
  else
    # ลบ invalid bytes
    str.scrub("?")
  end
end

# ลบ control characters
def remove_control_chars(str : String) : String
  str.gsub(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/, "")
end

# Normalize newlines
def normalize_newlines(str : String) : String
  str.gsub(/\r\n|\r/, "\n")
end

# Sanitize text สำหรับ display
def sanitize_text(str : String) : String
  str
    .tap { |s| s.valid_encoding? ? s : s.scrub }
    .gsub(/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/, "")  # control chars
    .gsub(/\r\n|\r/, "\n")  # normalize newlines
    .unicode_normalize(:nfc)  # normalize Unicode
    .strip
end

# ตัวอย่าง
text = "  Hello\r\n  World  "
puts sanitize_text(text)
# => "Hello\n  World"

# ตรวจสอบว่า string มีแค่ ASCII
def ascii_only?(str : String) : Bool
  str.each_char.all? { |c| c.ord < 128 }
end

puts ascii_only?("hello")    # => true
puts ascii_only?("héllo")    # => false
puts ascii_only?("สวัสดี")   # => false

# แปลง non-ASCII เป็น ASCII transliteration (simplified)
TRANSLITERATIONS = {
  'à' => 'a', 'á' => 'a', 'â' => 'a', 'ã' => 'a', 'ä' => 'a',
  'è' => 'e', 'é' => 'e', 'ê' => 'e', 'ë' => 'e',
  'ì' => 'i', 'í' => 'i', 'î' => 'i', 'ï' => 'i',
  'ò' => 'o', 'ó' => 'o', 'ô' => 'o', 'õ' => 'o', 'ö' => 'o',
  'ù' => 'u', 'ú' => 'u', 'û' => 'u', 'ü' => 'u',
  'ñ' => 'n', 'ç' => 'c', 'ý' => 'y', 'ÿ' => 'y',
}

def transliterate(str : String) : String
  String.build do |io|
    str.each_char do |c|
      io << (TRANSLITERATIONS[c]? || c)
    end
  end
end

puts transliterate("café résumé naïve")  # => "cafe resume naive"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียนฟังก์ชัน `unicode_word_count` ที่นับคำได้ถูกต้องสำหรับหลายภาษา (Thai ไม่มีช่องว่างระหว่างคำ ต้องใช้วิธีพิเศษ)

### แบบฝึกหัดที่ 2
เขียน `string_byte_analysis` ที่แสดง breakdown ของ bytes ในสตริง: กี่ chars เป็น 1-byte, 2-byte, 3-byte, 4-byte

### แบบฝึกหัดที่ 3
เขียน `safe_truncate` ที่ตัดสตริงตาม bytes (เช่น สำหรับ database column limit) โดยไม่ตัดกลาง UTF-8 sequence

### เฉลย

```crystal
# แบบฝึกหัดที่ 2
def string_byte_analysis(str : String) : Hash(String, Int32)
  counts = {"1_byte" => 0, "2_byte" => 0, "3_byte" => 0, "4_byte" => 0}
  
  str.each_char do |c|
    ord = c.ord
    bucket = case ord
    when 0..0x7F     then "1_byte"
    when 0x80..0x7FF  then "2_byte"
    when 0x800..0xFFFF then "3_byte"
    else                   "4_byte"
    end
    counts[bucket] += 1
  end
  
  counts
end

str = "Hello สวัสดี 🌍"
analysis = string_byte_analysis(str)
analysis.each { |k, v| puts "#{k}: #{v} chars" if v > 0 }

# แบบฝึกหัดที่ 3
def safe_truncate_bytes(str : String, max_bytes : Int32) : String
  return str if str.bytesize <= max_bytes
  
  result = String.build do |io|
    bytes_written = 0
    str.each_char do |c|
      char_bytes = c.to_s.bytesize
      break if bytes_written + char_bytes > max_bytes
      io << c
      bytes_written += char_bytes
    end
  end
  
  result
end

str = "Hello สวัสดี World"
puts safe_truncate_bytes(str, 10)  # ตัดที่ 10 bytes โดยไม่ตัดกลาง char
puts safe_truncate_bytes(str, 10).bytesize  # <= 10
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **UTF-8 Strings** - Crystal ใช้ UTF-8 เป็น encoding หลัก
2. **bytesize vs size** - ความแตกต่างระหว่าง bytes และ codepoints
3. **chars vs bytes** - วิธีวนลูปที่ถูกต้องสำหรับ Unicode
4. **Unicode Normalization** - NFC, NFD, NFKC, NFKD
5. **Emoji Handling** - จัดการ emoji และ grapheme clusters
6. **String.build** สำหรับสร้าง string ที่มี Unicode
7. **Encoding Conversion** - แปลง encoding อื่นเป็น UTF-8
8. **Multi-language** - ตรวจสอบและประมวลผลหลายภาษา
9. **Validation** - ตรวจสอบและ sanitize Unicode text

การเข้าใจ Unicode และ encoding เป็นสิ่งสำคัญสำหรับการสร้าง applications ที่รองรับหลายภาษา
