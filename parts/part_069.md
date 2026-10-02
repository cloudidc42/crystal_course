# Part 69: Parsing และ Tokenizing

## บทนำ

Parsing คือการแปลงข้อความเป็นโครงสร้างข้อมูล ซึ่งเป็นพื้นฐานสำคัญในการสร้าง configuration parsers, data processors, และ compilers การ tokenizing คือขั้นตอนแรกของ parsing ที่แบ่งข้อความเป็น tokens

---

## 1. split สำหรับ Simple Parsing

```crystal
# split พื้นฐาน
csv = "Alice,25,Bangkok"
parts = csv.split(",")
puts parts.inspect  # => ["Alice", "25", "Bangkok"]

# split ด้วย regex
words = "hello   world\tfoo".split(/\s+/)
puts words.inspect  # => ["hello", "world", "foo"]

# split with limit
line = "key=value=with=equals"
key, value = line.split("=", 2)
puts "key: #{key}, value: #{value}"
# => key: key, value: value=with=equals

# parse key-value pairs
def parse_pairs(text : String, pair_sep : String = ",", kv_sep : String = "=") : Hash(String, String)
  result = {} of String => String
  text.split(pair_sep).each do |pair|
    parts = pair.split(kv_sep, 2)
    result[parts[0].strip] = parts[1]?.try(&.strip) || ""
  end
  result
end

config = parse_pairs("host=localhost, port=8080, debug=true")
config.each { |k, v| puts "#{k} = #{v}" }
# => host = localhost
# => port = 8080
# => debug = true

# parse URL query string
def parse_query_string(qs : String) : Hash(String, String)
  result = {} of String => String
  qs.lstrip("?").split("&").each do |pair|
    parts = pair.split("=", 2)
    if parts.size == 2
      key = URI.decode_www_form(parts[0])
      value = URI.decode_www_form(parts[1])
      result[key] = value
    end
  end
  result
end

# Basic version without URI
def parse_query_simple(qs : String) : Hash(String, String)
  result = {} of String => String
  qs.lstrip("?").split("&").each do |pair|
    k, v = pair.split("=", 2)
    result[k] = v || ""
  end
  result
end

params = parse_query_simple("name=Alice&age=25&city=Bangkok")
params.each { |k, v| puts "#{k}: #{v}" }
```

---

## 2. scan สำหรับ Parsing

```crystal
# scan สำหรับดึง patterns หลายๆ อัน
text = "Prices: $10.99, $25.00, $7.50"
prices = text.scan(/\$(\d+\.\d{2})/).map(&.[1].to_f)
puts prices.inspect  # => [10.99, 25.0, 7.5]

# scan สำหรับ tokenizing
def tokenize_math(expr : String) : Array({String, String})
  tokens = [] of {String, String}
  expr.scan(/(\d+\.\d+|\d+)|([-+*\/^()])|\s+/) do |m|
    if !m[0].empty?
      tokens << {"NUMBER", m[0]}
    elsif !m[1].empty?
      tokens << {"OP", m[1]}
    end
    # skip whitespace
  end
  tokens
end

tokens = tokenize_math("3.14 + 2 * (10 - 4)")
tokens.each { |type, val| puts "#{type}: #{val}" }

# scan สำหรับ extract structured data
log_text = <<-LOG
  [2024-01-15 10:00:01] INFO Request from 192.168.1.1
  [2024-01-15 10:00:02] ERROR Failed to connect: timeout
  [2024-01-15 10:00:03] INFO Response sent (200 OK)
  LOG

log_text.scan(/\[(\d{4}-\d{2}-\d{2} \d{2}:\d{2}:\d{2})\] (\w+) (.+)/) do |m|
  puts "Time: #{m[1]}, Level: #{m[2]}, Message: #{m[3]}"
end
```

---

## 3. Simple CSV Parser

```crystal
# CSV parser ที่รองรับ quoted fields
class CSVParser
  def initialize(@source : String)
  end
  
  def parse : Array(Array(String))
    rows = [] of Array(String)
    @source.each_line(chomp: true) do |line|
      rows << parse_line(line)
    end
    rows
  end
  
  private def parse_line(line : String) : Array(String)
    fields = [] of String
    field = IO::Memory.new
    in_quotes = false
    i = 0
    
    while i < line.size
      c = line[i]
      
      if c == '"'
        if in_quotes && i + 1 < line.size && line[i + 1] == '"'
          # Escaped quote
          field << '"'
          i += 2
        else
          in_quotes = !in_quotes
          i += 1
        end
      elsif c == ',' && !in_quotes
        fields << field.to_s
        field = IO::Memory.new
        i += 1
      else
        field << c
        i += 1
      end
    end
    
    fields << field.to_s
    fields
  end
  
  def self.parse(source : String) : Array(Array(String))
    new(source).parse
  end
end

# ทดสอบ
csv = <<-CSV
  Name,Age,"City, Country",Note
  Alice,25,"Bangkok, Thailand","Has ""double"" quotes"
  Bob,30,"New York, USA",
  CSV

rows = CSVParser.parse(csv)
rows.each do |row|
  puts row.inspect
end

# CSV สำหรับ data ขนาดเล็ก
def csv_to_hashes(csv : String) : Array(Hash(String, String))
  rows = CSVParser.parse(csv)
  return [] of Hash(String, String) if rows.empty?
  
  headers = rows.first
  rows[1..].map do |row|
    hash = {} of String => String
    headers.each_with_index do |header, i|
      hash[header.strip] = row[i]?.try(&.strip) || ""
    end
    hash
  end
end

data = csv_to_hashes(<<-CSV)
  Name,Age,Score
  Alice,25,95
  Bob,30,87
  Charlie,28,92
  CSV

data.each { |row| puts row.inspect }
```

---

## 4. Simple JSON-like Parser

```crystal
# Simple JSON parser (สำหรับการศึกษา - ใช้ require "json" ในงานจริง)
class SimpleJSONParser
  alias Value = String | Int64 | Float64 | Bool | Nil | Array(Value) | Hash(String, Value)
  
  def initialize(@source : String)
    @pos = 0
  end
  
  def parse : Value
    skip_whitespace
    value = parse_value
    skip_whitespace
    raise "Unexpected character at position #{@pos}" unless @pos >= @source.size
    value
  end
  
  private def parse_value : Value
    return nil if @pos >= @source.size
    
    case @source[@pos]
    when '"' then parse_string
    when '{' then parse_object
    when '[' then parse_array
    when 't' then parse_true
    when 'f' then parse_false
    when 'n' then parse_null
    when '-', '0'..'9' then parse_number
    else raise "Unexpected character '#{@source[@pos]}' at position #{@pos}"
    end
  end
  
  private def parse_string : String
    @pos += 1  # skip opening quote
    result = IO::Memory.new
    
    while @pos < @source.size && @source[@pos] != '"'
      if @source[@pos] == '\\'
        @pos += 1
        case @source[@pos]
        when '"'  then result << '"'
        when '\\' then result << '\\'
        when '/'  then result << '/'
        when 'n'  then result << '\n'
        when 't'  then result << '\t'
        when 'r'  then result << '\r'
        end
      else
        result << @source[@pos]
      end
      @pos += 1
    end
    
    @pos += 1  # skip closing quote
    result.to_s
  end
  
  private def parse_number : Value
    start = @pos
    is_float = false
    
    @pos += 1 if @source[@pos] == '-'
    while @pos < @source.size && @source[@pos].ascii_number?
      @pos += 1
    end
    
    if @pos < @source.size && @source[@pos] == '.'
      is_float = true
      @pos += 1
      while @pos < @source.size && @source[@pos].ascii_number?
        @pos += 1
      end
    end
    
    num_str = @source[start, @pos - start]
    is_float ? num_str.to_f64 : num_str.to_i64
  end
  
  private def parse_object : Hash(String, Value)
    @pos += 1  # skip {
    result = {} of String => Value
    skip_whitespace
    
    unless @source[@pos] == '}'
      loop do
        skip_whitespace
        key = parse_string.as(String)
        skip_whitespace
        @pos += 1  # skip :
        skip_whitespace
        value = parse_value
        result[key] = value
        skip_whitespace
        break if @source[@pos] == '}'
        @pos += 1  # skip ,
      end
    end
    
    @pos += 1  # skip }
    result
  end
  
  private def parse_array : Array(Value)
    @pos += 1  # skip [
    result = [] of Value
    skip_whitespace
    
    unless @source[@pos] == ']'
      loop do
        skip_whitespace
        result << parse_value
        skip_whitespace
        break if @source[@pos] == ']'
        @pos += 1  # skip ,
      end
    end
    
    @pos += 1  # skip ]
    result
  end
  
  private def parse_true : Bool
    @pos += 4
    true
  end
  
  private def parse_false : Bool
    @pos += 5
    false
  end
  
  private def parse_null : Nil
    @pos += 4
    nil
  end
  
  private def skip_whitespace
    while @pos < @source.size && @source[@pos].whitespace?
      @pos += 1
    end
  end
end

# ทดสอบ
json = '{"name": "Alice", "age": 25, "active": true, "scores": [90, 85, 92]}'
result = SimpleJSONParser.new(json).parse
puts result.inspect
```

---

## 5. Tokenizer Pattern

```crystal
# Tokenizer สำหรับ simple expression language
enum TokenType
  NUMBER
  STRING
  IDENTIFIER
  OPERATOR
  LPAREN
  RPAREN
  COMMA
  EOF
end

struct Token
  getter type : TokenType
  getter value : String
  getter position : Int32
  
  def initialize(@type, @value, @position)
  end
  
  def to_s : String
    "#{@type}(#{@value.inspect})"
  end
end

class Tokenizer
  def initialize(@source : String)
    @pos = 0
    @tokens = [] of Token
  end
  
  def tokenize : Array(Token)
    while @pos < @source.size
      skip_whitespace
      break if @pos >= @source.size
      
      token = case @source[@pos]
      when '0'..'9'
        read_number
      when '"'
        read_string
      when 'a'..'z', 'A'..'Z', '_'
        read_identifier
      when '+', '-', '*', '/', '=', '<', '>', '!'
        read_operator
      when '('
        Token.new(TokenType::LPAREN, "(", @pos).tap { @pos += 1 }
      when ')'
        Token.new(TokenType::RPAREN, ")", @pos).tap { @pos += 1 }
      when ','
        Token.new(TokenType::COMMA, ",", @pos).tap { @pos += 1 }
      else
        raise "Unexpected character '#{@source[@pos]}' at position #{@pos}"
      end
      
      @tokens << token
    end
    
    @tokens << Token.new(TokenType::EOF, "", @pos)
    @tokens
  end
  
  private def skip_whitespace
    @pos += 1 while @pos < @source.size && @source[@pos].whitespace?
  end
  
  private def read_number : Token
    start = @pos
    while @pos < @source.size && (@source[@pos].ascii_number? || @source[@pos] == '.')
      @pos += 1
    end
    Token.new(TokenType::NUMBER, @source[start, @pos - start], start)
  end
  
  private def read_string : Token
    start = @pos
    @pos += 1  # skip "
    while @pos < @source.size && @source[@pos] != '"'
      @pos += 2 if @source[@pos] == '\\'  # skip escaped char
      @pos += 1
    end
    @pos += 1  # skip closing "
    Token.new(TokenType::STRING, @source[start + 1, @pos - start - 2], start)
  end
  
  private def read_identifier : Token
    start = @pos
    while @pos < @source.size && (@source[@pos].ascii_alphanumeric? || @source[@pos] == '_')
      @pos += 1
    end
    Token.new(TokenType::IDENTIFIER, @source[start, @pos - start], start)
  end
  
  private def read_operator : Token
    start = @pos
    op = @source[@pos].to_s
    @pos += 1
    # Multi-char operators
    if @pos < @source.size && "=<>!".includes?(@source[@pos])
      op += @source[@pos].to_s
      @pos += 1
    end
    Token.new(TokenType::OPERATOR, op, start)
  end
end

# ทดสอบ
expr = "greet(name, 42 + 3.14)"
tokens = Tokenizer.new(expr).tokenize
tokens.each { |t| puts t.to_s }
```

---

## 6. Recursive Descent Parser

```crystal
# Simple arithmetic expression parser
# Grammar:
# expr   = term (('+' | '-') term)*
# term   = factor (('*' | '/') factor)*
# factor = NUMBER | '(' expr ')'

class ExprParser
  def initialize(@tokens : Array(Token))
    @pos = 0
  end
  
  def parse : Float64
    result = parse_expr
    expect(TokenType::EOF)
    result
  end
  
  private def parse_expr : Float64
    result = parse_term
    
    while current.type == TokenType::OPERATOR && ["+", "-"].includes?(current.value)
      op = current.value
      advance
      right = parse_term
      result = op == "+" ? result + right : result - right
    end
    
    result
  end
  
  private def parse_term : Float64
    result = parse_factor
    
    while current.type == TokenType::OPERATOR && ["*", "/"].includes?(current.value)
      op = current.value
      advance
      right = parse_factor
      result = op == "*" ? result * right : result / right
    end
    
    result
  end
  
  private def parse_factor : Float64
    if current.type == TokenType::NUMBER
      value = current.value.to_f
      advance
      value
    elsif current.type == TokenType::LPAREN
      advance
      result = parse_expr
      expect(TokenType::RPAREN)
      result
    elsif current.type == TokenType::OPERATOR && current.value == "-"
      advance
      -parse_factor
    else
      raise "Expected number or '(', got #{current}"
    end
  end
  
  private def current : Token
    @tokens[@pos]
  end
  
  private def advance
    @pos += 1
  end
  
  private def expect(type : TokenType)
    if current.type != type
      raise "Expected #{type}, got #{current.type}"
    end
    advance
  end
end

def evaluate(expr : String) : Float64
  tokens = Tokenizer.new(expr).tokenize
  ExprParser.new(tokens).parse
end

puts evaluate("3 + 4 * 2")        # => 11.0
puts evaluate("(3 + 4) * 2")      # => 14.0
puts evaluate("10 / 2 - 3")       # => 2.0
puts evaluate("3.14 * 2")         # => 6.28
puts evaluate("-(5 + 3) * 2")     # => -16.0
```

---

## 7. INI/Config File Parser

```crystal
# Parse .ini /.conf style configuration
class IniParser
  struct Section
    getter name : String
    getter entries : Hash(String, String)
    
    def initialize(@name, @entries = {} of String => String)
    end
  end
  
  def initialize(@source : String)
  end
  
  def parse : Hash(String, Hash(String, String))
    result = {"" => {} of String => String}  # default section
    current_section = ""
    
    @source.each_line do |line|
      # Remove comment
      line = line.sub(/[;#].*$/, "").strip
      next if line.empty?
      
      if line =~ /^\[(.+)\]$/
        # Section header
        current_section = $~[1].strip
        result[current_section] ||= {} of String => String
      elsif line =~ /^([^=]+)=(.*)$/
        # Key-value pair
        key = $~[1].strip
        value = $~[2].strip
        
        # Handle quoted values
        if value.starts_with?('"') && value.ends_with?('"')
          value = value[1..-2]
        end
        
        result[current_section][key] = value
      end
    end
    
    result
  end
  
  def self.parse(source : String) : Hash(String, Hash(String, String))
    new(source).parse
  end
end

config_text = <<-INI
  ; Server Configuration
  [server]
  host = localhost
  port = 8080
  debug = true
  
  [database]
  host = 127.0.0.1
  port = 5432
  name = myapp
  user = admin
  password = "secret password"
  
  [logging]
  level = INFO
  file = /var/log/app.log
  INI

config = IniParser.parse(config_text)
config.each do |section, entries|
  next if section.empty?
  puts "[#{section}]"
  entries.each { |k, v| puts "  #{k} = #{v}" }
  puts
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน TOML-inspired parser ที่รองรับ strings, integers, booleans, arrays, และ tables

### แบบฝึกหัดที่ 2
เขียน DSL parser สำหรับ simple query language: `SELECT name, age FROM users WHERE age > 18`

### แบบฝึกหัดที่ 3
เขียน Markdown front matter parser ที่ parse YAML-style front matter จาก Markdown files

### เฉลย

```crystal
# แบบฝึกหัดที่ 3: Markdown front matter parser
def parse_front_matter(content : String) : {Hash(String, String), String}
  return ({} of String => String, content) unless content.starts_with?("---")
  
  lines = content.lines
  return ({} of String => String, content) if lines.size < 3
  
  end_index = lines.index(1..) { |l| l.chomp == "---" }
  return ({} of String => String, content) unless end_index
  
  front_matter_lines = lines[1...end_index]
  body_lines = lines[(end_index + 1)..]
  
  meta = {} of String => String
  front_matter_lines.each do |line|
    if line =~ /^(\w+):\s*(.+)$/
      key = $~[1]
      value = $~[2].strip.gsub(/^["']|["']$/, "")
      meta[key] = value
    end
  end
  
  {meta, body_lines.join}
end

markdown = <<-MD
  ---
  title: My Blog Post
  date: 2024-01-15
  author: Alice
  tags: crystal, programming
  ---
  
  # Content starts here
  
  This is the main content.
  MD

meta, body = parse_front_matter(markdown)
puts "Meta:"
meta.each { |k, v| puts "  #{k}: #{v}" }
puts "\nBody:"
puts body
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **split** สำหรับ simple parsing ของ delimited data
2. **scan** สำหรับดึง patterns หลายๆ อัน
3. **CSV Parser** รองรับ quoted fields และ escaped characters
4. **Simple JSON Parser** สร้าง parser จาก scratch
5. **Tokenizer Pattern** สำหรับแบ่งข้อความเป็น tokens
6. **Recursive Descent Parser** สำหรับ expressions
7. **Config File Parser** สำหรับ INI format

Parsing เป็นทักษะที่มีประโยชน์มากในการสร้าง config readers, DSLs, และ data processors
