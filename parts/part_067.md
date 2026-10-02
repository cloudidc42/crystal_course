# Part 67: StringIO และ String Builder

## บทนำ

การสร้าง strings ขนาดใหญ่หรือการสร้าง strings ในลักษณะ incremental นั้นมีประสิทธิภาพมากกว่าเมื่อใช้ `IO::Memory` หรือ `String.build` แทนการ concatenate strings ซ้ำๆ

---

## 1. IO::Memory พื้นฐาน

```crystal
# IO::Memory เป็น in-memory IO buffer
io = IO::Memory.new

# เขียนข้อมูล
io << "Hello"
io << ", "
io << "World!"

# อ่านเนื้อหาทั้งหมด
puts io.to_s  # => "Hello, World!"

# IO::Memory รองรับ IO interface
io = IO::Memory.new
io.print("Name: ")
io.puts("Alice")
io.print("Age: ")
io.puts(25)

puts io.to_s
# => Name: Alice
# => Age: 25

# IO::Memory กับ printf
io = IO::Memory.new
io.printf("%-10s: %d\n", "Score", 95)
io.printf("%-10s: %.2f\n", "Average", 87.5)
puts io.to_s
# => Score     : 95
# => Average   : 87.50

# rewind และอ่านใหม่
io = IO::Memory.new
io << "Hello World"
io.rewind
puts io.gets_to_end  # => "Hello World"

# ตรวจสอบ position
io = IO::Memory.new
io << "Hello"
puts io.pos   # => 5 (current position)
io.pos = 0    # กลับไปต้น
puts io.gets  # => "Hello"
```

---

## 2. String.build

```crystal
# String.build เป็นวิธีที่ Crystal แนะนำ
result = String.build do |io|
  io << "Hello"
  io << ", "
  io << "World!"
end
puts result  # => "Hello, World!"

# String.build กับ method calls
result = String.build do |io|
  (1..5).each do |i|
    io << i
    io << ", " unless i == 5
  end
end
puts result  # => "1, 2, 3, 4, 5"

# String.build กับ conditional
def build_list(items : Array(String), bullets : Bool = true) : String
  String.build do |io|
    items.each_with_index do |item, i|
      if bullets
        io << "• #{item}\n"
      else
        io << "#{i + 1}. #{item}\n"
      end
    end
  end
end

items = ["Crystal", "Ruby", "Python"]
puts build_list(items)
puts build_list(items, bullets: false)

# String.build initial_capacity (performance hint)
result = String.build(initial_capacity: 1024) do |io|
  1000.times { |i| io << i << " " }
end
puts result.size
```

---

## 3. Efficient String Building

```crystal
# เปรียบเทียบ performance
require "benchmark"

N = 10_000

Benchmark.ips do |x|
  x.report("concatenation (+)") do
    result = ""
    N.times { |i| result += i.to_s }
    result
  end
  
  x.report("String.build") do
    String.build do |io|
      N.times { |i| io << i }
    end
  end
  
  x.report("Array.join") do
    Array(String).new(N) { |i| i.to_s }.join
  end
end

# String.build ดีที่สุดสำหรับ incremental building
# Array.join ดีเมื่อมี elements อยู่แล้ว

# Pattern สำหรับ building complex strings
class HtmlBuilder
  def initialize
    @io = IO::Memory.new
    @indent = 0
  end
  
  def tag(name : String, attrs : Hash(String, String) = {} of String => String, &block)
    attr_str = attrs.empty? ? "" : " " + attrs.map { |k, v| "#{k}=\"#{v}\"" }.join(" ")
    @io << "  " * @indent << "<#{name}#{attr_str}>\n"
    @indent += 1
    block.call
    @indent -= 1
    @io << "  " * @indent << "</#{name}>\n"
  end
  
  def text(content : String)
    @io << "  " * @indent << content << "\n"
  end
  
  def to_s : String
    @io.to_s
  end
end

builder = HtmlBuilder.new
builder.tag("div", {"class" => "container"}) do
  builder.tag("h1") do
    builder.text("Hello World")
  end
  builder.tag("p", {"id" => "intro"}) do
    builder.text("This is a paragraph.")
  end
end

puts builder.to_s
```

---

## 4. IO::Memory เป็น Buffer

```crystal
# ใช้ IO::Memory เป็น buffer สำหรับ processing
def process_text(input : String) : String
  buffer = IO::Memory.new
  
  input.each_line do |line|
    # process แต่ละ line
    processed = line.strip.upcase
    buffer.puts processed unless processed.empty?
  end
  
  buffer.to_s
end

input = """
  hello world
  
  foo bar
  
  crystal rocks
  """
puts process_text(input)

# ใช้ IO::Memory สำหรับ accumulate output
class Report
  def initialize
    @buffer = IO::Memory.new
    @line_count = 0
  end
  
  def add_section(title : String, content : String)
    @buffer << "## #{title}\n\n"
    content.each_line do |line|
      @buffer << line.rstrip << "\n"
      @line_count += 1
    end
    @buffer << "\n"
  end
  
  def add_table(headers : Array(String), rows : Array(Array(String)))
    # Header
    @buffer << headers.join(" | ") << "\n"
    @buffer << headers.map { |h| "-" * h.size }.join("-+-") << "\n"
    
    # Rows
    rows.each do |row|
      @buffer << row.join(" | ") << "\n"
      @line_count += 1
    end
    @buffer << "\n"
  end
  
  def to_s : String
    header = "Report (#{@line_count} lines)\n" + "=" * 40 + "\n\n"
    header + @buffer.to_s
  end
end

report = Report.new
report.add_section("Introduction", "This is the intro text.\nWith multiple lines.")
report.add_table(
  ["Name", "Score"],
  [["Alice", "95"], ["Bob", "87"], ["Charlie", "92"]]
)
report.add_section("Conclusion", "That's all folks!")
puts report.to_s
```

---

## 5. Reading from IO::Memory

```crystal
# IO::Memory รองรับ read operations
io = IO::Memory.new("Hello\nWorld\nCrystal")

# อ่านทีละ line
while line = io.gets
  puts "Line: #{line}"
end

# อ่านทีละ char
io.rewind
while c = io.read_char
  print "#{c}|"
end
puts

# อ่าน n bytes
io.rewind
bytes = Bytes.new(5)
io.read(bytes)
puts String.new(bytes)  # => "Hello"

# gets_to_end
io.rewind
io.gets  # skip first line
puts io.gets_to_end  # => "World\nCrystal"

# ใช้ IO::Memory ในการ parse
def parse_config(config_text : String) : Hash(String, String)
  result = {} of String => String
  io = IO::Memory.new(config_text)
  
  while line = io.gets(chomp: true)
    next if line.strip.starts_with?("#")  # skip comments
    next if line.strip.empty?
    
    if line =~ /^(\w+)\s*=\s*(.*)$/
      result[$~[1]] = $~[2].strip
    end
  end
  
  result
end

config_text = """
  # Server configuration
  host = localhost
  port = 8080
  
  # Database
  db_host = 127.0.0.1
  db_name = myapp
  db_port = 5432
  """

config = parse_config(config_text)
config.each { |k, v| puts "#{k} = #{v}" }
```

---

## 6. Large String Construction

```crystal
# สำหรับ strings ขนาดใหญ่ ใช้ String.build ที่มี capacity
def generate_csv(rows : Int32, cols : Int32) : String
  # ประมาณ capacity
  estimated_size = rows * cols * 10  # 10 bytes per cell average
  
  String.build(initial_capacity: estimated_size) do |io|
    # Header
    cols.times { |i| io << "col#{i}#{i < cols - 1 ? "," : "\n"}" }
    
    # Data
    rows.times do |r|
      cols.times do |c|
        io << "#{r},#{c}"
        io << (c < cols - 1 ? "," : "\n")
      end
    end
  end
end

csv = generate_csv(100, 5)
puts "CSV size: #{csv.bytesize} bytes"
puts csv.lines.first  # header

# Streaming approach สำหรับ very large data
def write_large_report(io : IO, items : Array(Hash(String, String)))
  io << "REPORT\n"
  io << "=" * 60 << "\n"
  
  items.each_with_index do |item, i|
    io << "\n#{i + 1}. "
    item.each do |k, v|
      io << "#{k}: #{v}  "
    end
    io << "\n"
  end
  
  io << "\nTotal: #{items.size} items\n"
end

# ใช้กับ STDOUT หรือ IO::Memory
output = IO::Memory.new
items = Array.new(10) { |i| {"name" => "Item #{i}", "value" => "#{i * 100}"} }
write_large_report(output, items)
puts output.to_s[0..200] + "..."  # แสดงแค่ส่วนแรก

# สร้าง String ด้วย blocks
module StringUtils
  def self.build_table(headers : Array(String), rows : Array(Array(String))) : String
    col_widths = headers.map(&.size)
    rows.each do |row|
      row.each_with_index do |cell, i|
        col_widths[i] = [col_widths[i], cell.size].max if i < col_widths.size
      end
    end
    
    String.build do |io|
      # Header
      headers.each_with_index do |h, i|
        io << h.ljust(col_widths[i])
        io << " | " unless i == headers.size - 1
      end
      io << "\n"
      
      # Separator
      col_widths.each_with_index do |w, i|
        io << "-" * w
        io << "-+-" unless i == col_widths.size - 1
      end
      io << "\n"
      
      # Rows
      rows.each do |row|
        row.each_with_index do |cell, i|
          io << cell.ljust(col_widths[i])
          io << " | " unless i == row.size - 1
        end
        io << "\n"
      end
    end
  end
end

puts StringUtils.build_table(
  ["Name", "Age", "City"],
  [["Alice", "25", "Bangkok"], ["Bob", "30", "Chiang Mai"]]
)
```

---

## 7. IO::Memory ใน Testing

```crystal
# IO::Memory ใช้สำหรับ mock stdin/stdout ใน tests
require "spec"

def greet(name : String, output : IO = STDOUT)
  output.puts "Hello, #{name}!"
  output.puts "Welcome to Crystal."
end

describe "greet" do
  it "outputs correct greeting" do
    output = IO::Memory.new
    greet("Alice", output)
    
    lines = output.to_s.lines(chomp: true)
    lines[0].should eq "Hello, Alice!"
    lines[1].should eq "Welcome to Crystal."
  end
end

# อ่านจาก mock stdin
def process_input(input : IO) : String
  result = [] of String
  while line = input.gets(chomp: true)
    result << line.upcase
  end
  result.join("\n")
end

input_mock = IO::Memory.new("hello\nworld\ncrystal")
puts process_input(input_mock)
# => HELLO
# => WORLD
# => CRYSTAL

# Capture output
class OutputCapture
  def initialize
    @io = IO::Memory.new
  end
  
  def puts(msg = "")
    @io.puts msg
  end
  
  def print(msg)
    @io.print msg
  end
  
  def output : String
    @io.to_s
  end
  
  def lines : Array(String)
    @io.to_s.lines(chomp: true)
  end
end

capture = OutputCapture.new
capture.puts "Line 1"
capture.puts "Line 2"
capture.puts "Line 3"
puts capture.lines.inspect  # => ["Line 1", "Line 2", "Line 3"]
```

---

## 8. Pattern: Text Transformation Pipeline

```crystal
# Pipeline สำหรับ text transformation
class TextPipeline
  alias Transform = String -> String
  
  def initialize(@transforms = [] of Transform)
  end
  
  def add(&transform : Transform) : self
    @transforms << transform
    self
  end
  
  def process(text : String) : String
    @transforms.reduce(text) { |t, f| f.call(t) }
  end
  
  def process_io(input : IO, output : IO)
    input.each_line(chomp: true) do |line|
      output.puts process(line)
    end
  end
end

pipeline = TextPipeline.new
  .add { |s| s.strip }
  .add { |s| s.gsub(/\s+/, " ") }
  .add { |s| s.capitalize }
  .add { |s| ">> #{s}" }

inputs = [
  "  hello   world  ",
  "crystal   PROGRAMMING   language  ",
  "  foo bar  ",
]

inputs.each { |s| puts pipeline.process(s) }
# >> Hello world
# >> Crystal programming language
# >> Foo bar

# Pipeline ที่ใช้ IO::Memory
input_io = IO::Memory.new("  hello world  \n  foo bar  \n")
output_io = IO::Memory.new
pipeline.process_io(input_io, output_io)
puts output_io.to_s
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `JsonBuilder` class ที่ใช้ `IO::Memory` สร้าง JSON string โดยรองรับ nested objects และ arrays

### แบบฝึกหัดที่ 2
เขียน `template_engine` ที่ใช้ `String.build` render templates พร้อม loops `{{#each items}}...{{/each}}`

### แบบฝึกหัดที่ 3
เขียน `diff_strings` ที่เปรียบเทียบ 2 strings และแสดง diff เหมือน Unix diff command

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
class JsonBuilder
  def initialize
    @io = IO::Memory.new
    @first = [true]  # stack for comma tracking
  end
  
  def object(&block) : self
    @io << "{"
    @first.push(true)
    block.call
    @first.pop
    @io << "}"
    self
  end
  
  def array(&block) : self
    @io << "["
    @first.push(true)
    block.call
    @first.pop
    @io << "]"
    self
  end
  
  def field(key : String, value : String | Int32 | Float64 | Bool | Nil) : self
    comma_if_needed
    @io << "\"#{key}\": "
    write_value(value)
    self
  end
  
  def element(value : String | Int32 | Float64 | Bool | Nil) : self
    comma_if_needed
    write_value(value)
    self
  end
  
  def to_s : String
    @io.to_s
  end
  
  private def comma_if_needed
    if @first.last
      @first[-1] = false
    else
      @io << ", "
    end
  end
  
  private def write_value(value)
    case value
    when String then @io << "\"#{value}\""
    when Bool   then @io << value.to_s
    when Nil    then @io << "null"
    else             @io << value.to_s
    end
  end
end

json = JsonBuilder.new
json.object do
  json.field("name", "Alice")
  json.field("age", 25)
  json.field("active", true)
  json.field("score", nil)
end
puts json.to_s
# => {"name": "Alice", "age": 25, "active": true, "score": null}
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **IO::Memory** - in-memory IO buffer สำหรับ read/write operations
2. **String.build** - efficient string building pattern
3. **Performance** - เปรียบเทียบวิธีการสร้าง strings
4. **Reading from IO::Memory** - gets, read, rewind
5. **Large strings** - สร้าง strings ขนาดใหญ่อย่างมีประสิทธิภาพ
6. **Testing** - ใช้ IO::Memory สำหรับ mock IO
7. **Transformation pipeline** - pattern สำหรับ text processing

การใช้ `String.build` และ `IO::Memory` เป็น best practice ใน Crystal สำหรับการสร้าง strings ขนาดใหญ่หรือ strings ที่สร้างแบบ incremental
