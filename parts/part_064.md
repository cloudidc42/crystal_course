# Part 64: Regular Expressions ขั้นสูง

## บทนำ

ในบทนี้เราจะเรียนรู้ Regex ขั้นสูงใน Crystal ได้แก่ named captures, lookahead, lookbehind, non-greedy matching, alternation, back-references และ practical validation patterns

---

## 1. Named Captures (?<name>...)

```crystal
# named capture (?<name>...)
str = "John Smith, age 25"
if str =~ /(?<first>\w+)\s+(?<last>\w+),\s+age\s+(?<age>\d+)/
  puts $~["first"]  # => "John"
  puts $~["last"]   # => "Smith"
  puts $~["age"]    # => "25"
end

# named captures กับ match
m = "2024-01-15".match(/(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/)
if m
  puts "Year: #{m["year"]}"   # => "Year: 2024"
  puts "Month: #{m["month"]}" # => "Month: 01"
  puts "Day: #{m["day"]}"     # => "Day: 15"
end

# named captures กับ gsub
str = "John Smith"
result = str.gsub(/(?<first>\w+)\s+(?<last>\w+)/) do
  "#{$~["last"]}, #{$~["first"]}"
end
puts result  # => "Smith, John"

# ใช้ named captures ในการ parse URL
def parse_url(url : String) : Hash(String, String)?
  pattern = /
    \A
    (?<scheme>https?|ftp):\/\/
    (?<host>[^\/\?#]+)
    (?<path>\/[^\?#]*)?
    (?:\?(?<query>[^#]*))?
    (?:\#(?<fragment>.*))?
    \Z
  /x
  
  if url =~ pattern
    result = {} of String => String
    ["scheme", "host", "path", "query", "fragment"].each do |part|
      value = $~[part]?
      result[part] = value || ""
    end
    result
  end
end

url = "https://example.com/path/to/page?q=crystal#section"
if parts = parse_url(url)
  parts.each { |k, v| puts "#{k}: #{v}" unless v.empty? }
end
# scheme: https
# host: example.com
# path: /path/to/page
# query: q=crystal
# fragment: section
```

---

## 2. Lookahead (?=...) และ Negative Lookahead (?!...)

```crystal
# Positive lookahead (?=...) - match ถ้าตามด้วย pattern
# แต่ไม่ include ใน match

str = "foo123 bar456 baz789"

# หา words ที่ตามด้วยตัวเลข
puts str.scan(/\w+(?=\d)/).inspect  # words before digits (partial)

# ตัวอย่างที่ชัดเจนกว่า
str = "price: $100, cost: $200, total: $300"
# หาตัวเลขที่อยู่หลัง $
prices = str.scan(/(?<=\$)\d+/)
puts prices.inspect  # => ["100", "200", "300"]

# Password validation ด้วย lookahead
def check_password_strength(password : String) : String
  checks = {
    "uppercase" => password.match?(/(?=.*[A-Z])/),
    "lowercase" => password.match?(/(?=.*[a-z])/),
    "digit" => password.match?(/(?=.*\d)/),
    "special" => password.match?(/(?=.*[!@#$%^&*])/),
    "length_8+" => password.size >= 8,
  }
  
  passed = checks.count { |_, v| v }
  case passed
  when 5 then "Strong"
  when 3..4 then "Medium"
  else "Weak"
  end
end

puts check_password_strength("Password1!")  # => "Strong"
puts check_password_strength("password")    # => "Weak"
puts check_password_strength("Pass1!")      # => "Medium"

# Negative lookahead (?!...)
# หา "file" ที่ไม่ตามด้วย ".rb"
str = "file.rb, file.cr, file.txt, file.rb"
puts str.scan(/file(?!\.rb)\.\w+/).inspect  # => ["file.cr", "file.txt"]

# หา integers ที่ไม่ใช่ float (ไม่มี decimal point)
str = "42, 3.14, 100, 2.71"
puts str.scan(/\d+(?!\.\d)(?!\d)/).inspect  # => ["42", "100"]
```

---

## 3. Lookbehind (?<=...) และ Negative Lookbehind (?<!...)

```crystal
# Positive lookbehind (?<=...) - match ถ้านำหน้าด้วย pattern
# ไม่ include ส่วน lookbehind ใน match

str = "USD 100, EUR 200, GBP 300"
# หาตัวเลขที่อยู่หลัง currency code
amounts = str.scan(/(?<=\b[A-Z]{3}\s)\d+/)
puts amounts.inspect  # => ["100", "200", "300"]

# หา words หลัง "the"
str = "the cat sat on the mat near the bat"
after_the = str.scan(/(?<=the\s)\w+/)
puts after_the.inspect  # => ["cat", "mat", "bat"]

# Negative lookbehind (?<!...)
str = "hello world, helloworld, say hello"
# หา "hello" ที่ไม่อยู่ติดกับคำอื่น (standalone)
puts str.scan(/(?<!\w)hello(?!\w)/).inspect  # => ["hello", "hello"]

# ตัวอย่าง: extract prices ที่ไม่ใช่ negative
str = "price is -10 or +20 or just 30"
positive_numbers = str.scan(/(?<!-)\b\d+\b/)
puts positive_numbers.inspect  # => ["20", "30"] (ไม่รวม -10)

# Split สตริงโดยใช้ lookbehind
# แยก camelCase เป็น words
def split_camel_case(str : String) : Array(String)
  str.gsub(/(?<=[a-z])(?=[A-Z])/, " ").split
end

puts split_camel_case("helloWorldFooBar").inspect
# => ["hello", "World", "Foo", "Bar"]

puts split_camel_case("getUserByEmail").inspect
# => ["get", "User", "By", "Email"]
```

---

## 4. Non-Greedy Matching

```crystal
# Greedy (default): match ให้มากที่สุด
str = "<b>bold</b> and <i>italic</i>"

greedy = str.scan(/<.+>/)
puts "Greedy: #{greedy.inspect}"
# => ["<b>bold</b> and <i>italic</i>"]  (match ทั้งหมด)

# Non-greedy: match ให้น้อยที่สุด
non_greedy = str.scan(/<.+?>/)
puts "Non-greedy: #{non_greedy.inspect}"
# => ["<b>", "</b>", "<i>", "</i>"]  (match แต่ละ tag)

# ตัวอย่าง: extract HTML tags content
def extract_tag_content(html : String, tag : String) : Array(String)
  html.scan(/<#{tag}[^>]*>(.*?)<\/#{tag}>/m).map(&.[1])
end

html = "<p>First paragraph</p><p>Second paragraph</p>"
puts extract_tag_content(html, "p").inspect
# => ["First paragraph", "Second paragraph"]

# *? +? ?? {n,m}?
str = "aXbXXcXXXd"
puts str.scan(/X+/).inspect   # greedy: => ["X", "XX", "XXX"]
puts str.scan(/X+?/).inspect  # non-greedy: => ["X", "X", "X", "X", "X", "X"]

# ใช้งานจริง: extract JSON values
json = '{"name": "Alice", "city": "Bangkok"}'
values = json.scan(/"([^"]+)":\s*"([^"]+)"/)
values.each do |m|
  puts "#{m[1]}: #{m[2]}"
end
# => name: Alice
# => city: Bangkok

# extract content ระหว่าง delimiters
def extract_between(str : String, open : String, close : String) : Array(String)
  open_e = Regex.escape(open)
  close_e = Regex.escape(close)
  str.scan(/#{open_e}(.*?)#{close_e}/).map(&.[1])
end

puts extract_between("{{name}} and {{city}}", "{{", "}}").inspect
# => ["name", "city"]

puts extract_between("<<start>>content<<end>>", "<<", ">>").inspect
# => ["start", "content", "end"]
```

---

## 5. Alternation |

```crystal
# | แทนที่ OR ระหว่าง patterns
str = "I have a cat"
puts str.match?(/cat|dog|bird/)  # => true

str2 = "I have a fish"
puts str2.match?(/cat|dog|bird/)  # => false

# alternation ใน group
dates = ["01/15/2024", "2024-01-15", "January 15 2024"]
dates.each do |date|
  if date =~ /(\d{2})\/(\d{2})\/(\d{4})|(\d{4})-(\d{2})-(\d{2})/
    puts "Match: #{date}"
  end
end

# ลำดับความสำคัญ: | มีความสำคัญต่ำสุด
puts "cat".match?(/cat|cats/)  # => true
puts "cats".match?(/cat|cats/) # => true (match "cat" ก่อน)
puts "cats".match?(/cats|cat/) # => true (match "cats")

# ใช้ group เพื่อ limit alternation scope
pattern = /\b(cat|dog)\s+food\b/
puts "cat food is good".match?(pattern)   # => true
puts "dog food is good".match?(pattern)   # => true
puts "fish food is good".match?(pattern)  # => false

# filename extension matcher
def code_file?(filename : String) : Bool
  filename.match?(/\.(rb|cr|py|js|ts|go|rs)\Z/)
end

puts code_file?("main.cr")      # => true
puts code_file?("style.css")    # => false
puts code_file?("index.html")   # => false

# ดึง protocol จาก URL
def get_protocol(url : String) : String?
  if url =~ /\A(https?|ftp|ssh|git):\/\//
    $~[1]
  end
end

puts get_protocol("https://example.com")  # => "https"
puts get_protocol("ftp://files.com")      # => "ftp"
puts get_protocol("not-a-url").inspect    # => nil
```

---

## 6. Back-references

```crystal
# \1, \2 ... อ้างอิงกลับไปยัง captured groups
# ใช้หาคำที่ซ้ำกัน

str = "the the quick brown fox fox"
# หา doubled words
puts str.scan(/\b(\w+)\s+\1\b/).inspect
# => [["the"], ["fox"]]

# back-reference ใน gsub
str = "hello hello world world"
# ลบ duplicate words
result = str.gsub(/\b(\w+)(\s+\1)+\b/, "\\1")
puts result  # => "hello world"

# ตรวจสอบ matched quotes
def balanced_quotes?(str : String) : Bool
  str.match?(/\A(['"]).*\1\Z/)
end

puts balanced_quotes?("'hello'")   # => true
puts balanced_quotes?('"hello"')   # => true
puts balanced_quotes?("'hello\"")  # => false

# HTML tag matching
def valid_html_tag?(html : String) : Bool
  html.match?(/\A<(\w+)>.*<\/\1>\Z/m)
end

puts valid_html_tag?("<div>content</div>")  # => true
puts valid_html_tag?("<div>content</span>") # => false
puts valid_html_tag?("<p>paragraph</p>")    # => true

# named back-reference (\k<name>)
str = "hello hello"
puts str.match?(/(?<word>\w+)\s+\k<word>/)  # => true

str2 = "hello world"
puts str2.match?(/(?<word>\w+)\s+\k<word>/) # => false
```

---

## 7. Regex.new และ Dynamic Patterns

```crystal
# สร้าง Regex จาก String dynamically
words = ["cat", "dog", "bird"]
pattern = Regex.new("\\b(#{words.join("|")})\\b", Regex::Options::IGNORE_CASE)

puts "I have a Cat".match?(pattern)  # => true
puts "I have a fish".match?(pattern) # => false

# Build pattern ที่ซับซ้อน
def build_keyword_pattern(keywords : Array(String)) : Regex
  escaped = keywords.map { |k| Regex.escape(k) }
  Regex.new("\\b(#{escaped.join("|")})\\b", Regex::Options::IGNORE_CASE)
end

keywords = ["Crystal", "Ruby", "Python", "Go"]
pattern = build_keyword_pattern(keywords)

text = "I love Crystal and python programming"
matches = text.scan(pattern).map(&.[0])
puts matches.inspect  # => ["Crystal", "python"]

# Regex.escape สำหรับ escape special chars
user_input = "hello.world (test)"
safe = Regex.escape(user_input)
puts safe  # => "hello\\.world\\ \\(test\\)"

# ค้นหา literal string (ไม่ใช่ regex)
def find_literal(text : String, search : String) : Int32?
  pattern = Regex.new(Regex.escape(search))
  text.index(pattern)
end

puts find_literal("price $10.00", "$10.00").inspect  # => 6
puts find_literal("price $10.00", "£10").inspect      # => nil

# สร้าง router-like pattern
def path_to_regex(path : String) : Regex
  pattern = path
    .gsub(/\/:(\w+)/) { "/(?<#{$~[1]}>[^/]+)" }
    .gsub("/", "\\/")
  Regex.new("\\A#{pattern}\\Z")
end

route_pattern = path_to_regex("/users/:id/posts/:post_id")
url = "/users/42/posts/123"

if url =~ route_pattern
  puts "User ID: #{$~["id"]}"      # => "42"
  puts "Post ID: #{$~["post_id"]}" # => "123"
end
```

---

## 8. Practical Validation Patterns

```crystal
# Validation module
module Validators
  # Thai National ID validation
  def self.thai_id?(id : String) : Bool
    return false unless id.match?(/\A\d{13}\Z/)
    
    digits = id.chars.map(&.to_i)
    sum = (0..11).reduce(0) { |acc, i| acc + digits[i] * (13 - i) }
    (11 - sum % 11) % 10 == digits[12]
  end
  
  # Email validation
  def self.email?(email : String) : Bool
    email.match?(/\A[a-zA-Z0-9.!#$%&'*+\/=?^_`{|}~-]+@[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?(?:\.[a-zA-Z0-9](?:[a-zA-Z0-9-]{0,61}[a-zA-Z0-9])?)*\Z/)
  end
  
  # URL validation
  def self.url?(url : String) : Bool
    url.match?(/\Ahttps?:\/\/[^\s\/$.?#].[^\s]*\Z/)
  end
  
  # Luhn algorithm for credit card
  def self.credit_card?(number : String) : Bool
    cleaned = number.delete(" -")
    return false unless cleaned.match?(/\A\d{13,19}\Z/)
    
    digits = cleaned.chars.map(&.to_i).reverse
    sum = digits.each_with_index.reduce(0) do |acc, (digit, i)|
      if i.odd?
        doubled = digit * 2
        acc + (doubled > 9 ? doubled - 9 : doubled)
      else
        acc + digit
      end
    end
    
    sum % 10 == 0
  end
  
  # IPv4 validation
  def self.ipv4?(ip : String) : Bool
    return false unless ip.match?(/\A(?:\d{1,3}\.){3}\d{1,3}\Z/)
    ip.split(".").all? { |part| part.to_i.in?(0..255) }
  end
  
  # IPv6 (basic)
  def self.ipv6?(ip : String) : Bool
    ip.match?(/\A(?:[0-9a-fA-F]{1,4}:){7}[0-9a-fA-F]{1,4}\Z/) ||
    ip.match?(/\A(?:[0-9a-fA-F]{1,4}:)*::(?:[0-9a-fA-F]{1,4}:)*[0-9a-fA-F]{1,4}\Z/)
  end
  
  # Slug validation
  def self.slug?(slug : String) : Bool
    slug.match?(/\A[a-z0-9]+(?:-[a-z0-9]+)*\Z/)
  end
  
  # Strong password
  def self.strong_password?(password : String) : Bool
    password.size >= 8 &&
    password.match?(/[A-Z]/) &&
    password.match?(/[a-z]/) &&
    password.match?(/\d/) &&
    password.match?(/[^A-Za-z\d]/)
  end
end

# ทดสอบ validators
puts "Validators Test:"
puts "Email: #{Validators.email?("user@example.com")}"   # true
puts "Email: #{Validators.email?("invalid")}"            # false
puts "URL: #{Validators.url?("https://example.com")}"   # true
puts "IPv4: #{Validators.ipv4?("192.168.1.1")}"         # true
puts "IPv4: #{Validators.ipv4?("256.1.1.1")}"           # false
puts "Slug: #{Validators.slug?("hello-world")}"         # true
puts "Slug: #{Validators.slug?("Hello World")}"         # false
puts "Password: #{Validators.strong_password?("P@ssw0rd")}"  # true
```

---

## 9. Advanced Regex Techniques

```crystal
# Atomic groups และ Possessive quantifiers (ถ้า supported)
# Crystal ใช้ PCRE2 ซึ่งรองรับ possessive quantifiers

# Conditional patterns
# (?(?=condition)then|else) - conditional in regex

# Unicode properties \p{...}
# ตัวอักษรใน Unicode category
def extract_thai_text(str : String) : String
  str.scan(/[฀-๿]+/).join(" ")
end

puts extract_thai_text("Hello สวัสดี World ครับ")
# => "สวัสดี ครับ"

# Custom scanner
class Scanner
  def initialize(@source : String)
    @pos = 0
  end
  
  def scan(pattern : Regex) : String?
    if @source[@pos..] =~ /\A#{pattern.source}/
      match = $~[0]
      @pos += match.size
      match
    end
  end
  
  def skip_whitespace
    scan(/\s+/)
  end
  
  def eof? : Bool
    @pos >= @source.size
  end
  
  def remaining : String
    @source[@pos..]
  end
end

# สร้าง simple tokenizer
scanner = Scanner.new("42 + 3.14 * foo")
tokens = [] of {String, String}

until scanner.eof?
  scanner.skip_whitespace
  break if scanner.eof?
  
  token = if t = scanner.scan(/\d+\.\d+/)
    {"FLOAT", t}
  elsif t = scanner.scan(/\d+/)
    {"INT", t}
  elsif t = scanner.scan(/[a-zA-Z_]\w*/)
    {"IDENT", t}
  elsif t = scanner.scan(/[+\-*\/]/)
    {"OP", t}
  else
    {"UNKNOWN", scanner.remaining[0..0]}
  end
  
  tokens << token
end

tokens.each { |type, value| puts "#{type}: #{value}" }
```

---

## 10. Regex Performance Tips

```crystal
# 1. Compile regex ครั้งเดียว ไม่ใช่ใน loop
# BAD
def check_all_emails_slow(emails : Array(String)) : Array(Bool)
  emails.map { |e| e.match?(/\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/) }  # compile every time
end

# GOOD
EMAIL_RE = /\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/
def check_all_emails_fast(emails : Array(String)) : Array(Bool)
  emails.map { |e| e.match?(EMAIL_RE) }  # compiled once
end

# 2. ใช้ match? แทน match เมื่อไม่ต้องการ MatchData
# BAD (สร้าง MatchData object)
str = "hello@example.com"
if str.match(/\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/)
  puts "valid email"
end

# GOOD (ไม่สร้าง MatchData)
if str.match?(EMAIL_RE)
  puts "valid email"
end

# 3. หลีกเลี่ยง catastrophic backtracking
# BAD: (a+)+ - exponential backtracking
# ใช้ atomic groups หรือ possessive quantifiers แทน

# 4. ใช้ anchors เพื่อ fail fast
# BAD
"definitely not an email".match?(/[\w.+-]+@[\w-]+\.[\w.-]+/)

# GOOD - fail at start
"definitely not an email".match?(/\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/)

# 5. Profile regex ที่ช้า
require "benchmark"

test_str = "a" * 1000 + "@example.com"

Benchmark.ips do |x|
  x.report("with anchors") { test_str.match?(/\A[\w.+-]+@[\w-]+\.[\w.-]+\Z/) }
  x.report("no anchors") { test_str.match?(/[\w.+-]+@[\w-]+\.[\w.-]+/) }
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน parser สำหรับ simple arithmetic expression ที่ใช้ regex tokenizing แล้ว evaluate ผลลัพธ์ (รองรับ +, -, *, / และวงเล็บ)

### แบบฝึกหัดที่ 2
เขียน `extract_mentions_and_hashtags` ที่ดึง @mentions และ #hashtags จาก social media text

### แบบฝึกหัดที่ 3
เขียน URL router ที่ map path patterns เช่น `/users/:id` ไปยัง handler functions

### เฉลย

```crystal
# แบบฝึกหัดที่ 2
def extract_mentions_and_hashtags(text : String) : {mentions: Array(String), hashtags: Array(String)}
  mentions = text.scan(/@(\w+)/).map(&.[1])
  hashtags = text.scan(/#(\w+)/).map(&.[1])
  {mentions: mentions, hashtags: hashtags}
end

text = "Hello @alice and @bob! Check out #crystal and #programming"
result = extract_mentions_and_hashtags(text)
puts "Mentions: #{result[:mentions].inspect}"
# => ["alice", "bob"]
puts "Hashtags: #{result[:hashtags].inspect}"
# => ["crystal", "programming"]

# แบบฝึกหัดที่ 3
class Router
  alias Handler = Hash(String, String) -> String
  
  def initialize
    @routes = [] of {Regex, Array(String), Handler}
  end
  
  def add(path : String, &handler : Handler)
    param_names = [] of String
    pattern = path.gsub(/:(\w+)/) do
      param_names << $~[1]
      "([^/]+)"
    end
    regex = Regex.new("\\A#{pattern}\\Z")
    @routes << {regex, param_names, handler}
  end
  
  def match(path : String) : String?
    @routes.each do |regex, param_names, handler|
      if m = path.match(regex)
        params = {} of String => String
        param_names.each_with_index do |name, i|
          params[name] = m[i + 1]
        end
        return handler.call(params)
      end
    end
    nil
  end
end

router = Router.new
router.add("/users/:id") { |p| "User #{p["id"]}" }
router.add("/users/:id/posts/:post_id") { |p| "User #{p["id"]}, Post #{p["post_id"]}" }
router.add("/about") { |_| "About page" }

puts router.match("/users/42").inspect          # => "User 42"
puts router.match("/users/42/posts/7").inspect  # => "User 42, Post 7"
puts router.match("/about").inspect             # => "About page"
puts router.match("/unknown").inspect           # => nil
```

---

## สรุป

ในบทนี้เราได้เรียนรู้ Regex ขั้นสูงใน Crystal:

1. **Named captures** `(?<name>...)` สำหรับ readable pattern matching
2. **Lookahead** `(?=...)` และ `(?!...)` สำหรับ match โดยไม่ consume
3. **Lookbehind** `(?<=...)` และ `(?<!...)` สำหรับ check ก่อน match
4. **Non-greedy** `*?`, `+?`, `??` สำหรับ minimal matching
5. **Alternation** `|` สำหรับ OR patterns
6. **Back-references** `\1`, `\k<name>` สำหรับ refer to captures
7. **Regex.new** สำหรับ dynamic patterns
8. **Practical validation** patterns สำหรับ email, URL, etc.
9. **Performance tips** สำหรับ efficient regex usage

Regular Expressions ขั้นสูงเหล่านี้ช่วยให้เขียน text processing code ได้อย่างทรงพลังและยืดหยุ่น
