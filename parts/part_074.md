# Part 74: STDIN/STDOUT/STDERR

## บทนำ

Crystal มี standard IO streams สามอัน: STDIN, STDOUT, STDERR ซึ่งเป็น IO objects ที่ใช้สำหรับ input จาก keyboard, output ไปหน้าจอ, และ error messages ตามลำดับ

---

## 1. STDIN

```crystal
# อ่านจาก STDIN
line = STDIN.gets(chomp: true)
puts "You typed: #{line}"

# gets ใน top-level ก็คือ STDIN.gets
line = gets(chomp: true)
puts "You typed: #{line}"

# อ่านทีละบรรทัด
while line = STDIN.gets(chomp: true)
  break if line == "quit"
  puts "Echo: #{line}"
end

# อ่านทั้งหมด
all_input = STDIN.gets_to_end
puts "Total: #{all_input.size} chars"

# อ่าน binary จาก STDIN
binary_data = STDIN.getb_to_end  # ถ้า API support
# หรือ
buffer = IO::Memory.new
IO.copy(STDIN, buffer)
data = buffer.to_s

# ตรวจสอบว่า STDIN มีข้อมูล (pipe or redirect)
def stdin_has_data? : Bool
  !STDIN.tty?
end

if stdin_has_data?
  content = STDIN.gets_to_end
  puts "Processing #{content.size} chars from stdin"
else
  puts "No stdin data, using interactive mode"
end
```

---

## 2. STDOUT

```crystal
# เขียนไปยัง STDOUT
STDOUT.puts "Hello, World!"
STDOUT.print "No newline"
STDOUT.print "\n"
STDOUT.printf("Formatted: %d\n", 42)

# << operator
STDOUT << "Hello" << " " << "World" << "\n"

# top-level puts/print ก็คือ STDOUT
puts "Hello"     # same as STDOUT.puts "Hello"
print "World\n"  # same as STDOUT.print "World\n"

# flush - ส่ง buffer ออกทันที
STDOUT.print "Processing..."
STDOUT.flush  # ให้แสดงก่อน sleep
sleep 1
STDOUT.puts " done!"

# sync mode
STDOUT.sync = true  # auto-flush ทุก write

# redirect STDOUT ไปยัง variable (ใน test)
# ทำได้โดยใช้ IO::Memory แทน

# ตรวจสอบว่าเป็น terminal
if STDOUT.tty?
  puts "Running in terminal (may use colors)"
else
  puts "Output is redirected (use plain text)"
end

# ANSI colors (เฉพาะ terminal)
def colorize(text : String, color_code : Int32) : String
  if STDOUT.tty?
    "\e[#{color_code}m#{text}\e[0m"
  else
    text  # ไม่ใส่ ANSI เมื่อ redirect
  end
end

puts colorize("Success", 32)  # green
puts colorize("Warning", 33)  # yellow
puts colorize("Error", 31)    # red
```

---

## 3. STDERR

```crystal
# เขียน errors ไปยัง STDERR
STDERR.puts "Error: Something went wrong"
STDERR.printf("Error code: %d\n", 42)

# ใช้สำหรับ debug output
def debug(message : String)
  STDERR.puts "[DEBUG] #{message}" if ENV["DEBUG"]?
end

debug("Processing started")

# Error logging
class ErrorLogger
  def self.log(error : Exception, context : String = "")
    STDERR.puts "=" * 60
    STDERR.puts "ERROR: #{error.message}"
    STDERR.puts "Context: #{context}" unless context.empty?
    STDERR.puts "Backtrace:"
    error.backtrace?.try do |bt|
      bt.each { |line| STDERR.puts "  #{line}" }
    end
    STDERR.puts "=" * 60
  end
end

begin
  raise "Test error"
rescue ex
  ErrorLogger.log(ex, "in main process")
end

# Warning utility
def warn(message : String)
  STDERR.puts "WARNING: #{message}"
end

warn("Deprecated feature used")
warn("High memory usage detected")
```

---

## 4. Redirecting Output

```crystal
# ใน Crystal สามารถ redirect โดยส่ง IO objects
def process(input : IO = STDIN, output : IO = STDOUT, error : IO = STDERR)
  while line = input.gets(chomp: true)
    if line.starts_with?("ERROR:")
      error.puts "Caught error: #{line}"
    else
      output.puts "Processed: #{line}"
    end
  end
end

# ทดสอบด้วย IO::Memory
input = IO::Memory.new("hello\nERROR: bad thing\nworld")
output = IO::Memory.new
errors = IO::Memory.new

process(input, output, errors)

puts "Output:"
puts output.to_s
puts "Errors:"
puts errors.to_s

# Capture all output ของ function
def capture_output(&block : IO ->) : String
  io = IO::Memory.new
  block.call(io)
  io.to_s
end

result = capture_output do |io|
  io.puts "Line 1"
  io.puts "Line 2"
  io.puts "Line 3"
end

puts result.lines.size  # => 3
```

---

## 5. IO::Memory เป็น Fake STDIN/STDOUT

```crystal
# Fake STDIN/STDOUT สำหรับ testing
class TestIO
  getter output : IO::Memory
  
  def initialize(input_data : String = "")
    @input = IO::Memory.new(input_data)
    @output = IO::Memory.new
  end
  
  def stdin : IO
    @input
  end
  
  def stdout : IO
    @output
  end
  
  def output_lines : Array(String)
    @output.to_s.lines(chomp: true).reject(&.empty?)
  end
end

# Application function ที่รับ IO
def interactive_calculator(input : IO = STDIN, output : IO = STDOUT)
  output.print "Enter expression (e.g., 2 + 3): "
  
  while line = input.gets(chomp: true)
    break if line == "quit"
    
    if line =~ /(\d+)\s*([-+*\/])\s*(\d+)/
      a = $~[1].to_f
      op = $~[2]
      b = $~[3].to_f
      result = case op
      when "+" then a + b
      when "-" then a - b
      when "*" then a * b
      when "/" then a / b
      end
      output.puts "= #{result}"
    else
      output.puts "Invalid expression"
    end
    
    output.print "Enter expression (or 'quit'): "
  end
  
  output.puts "Goodbye!"
end

# Test
test = TestIO.new("2 + 3\n10 * 5\nquit\n")
interactive_calculator(test.stdin, test.stdout)

test.output_lines.each { |l| puts "Got: #{l}" }

# Progress reporter
class ProgressReporter
  def initialize(@total : Int32, @output : IO = STDOUT)
    @current = 0
    @start_time = Time.monotonic
  end
  
  def increment(n : Int32 = 1)
    @current += n
    report
  end
  
  private def report
    pct = (@current.to_f / @total * 100).round(1)
    elapsed = (Time.monotonic - @start_time).total_seconds
    rate = elapsed > 0 ? @current / elapsed : 0
    eta = rate > 0 ? (@total - @current) / rate : 0
    
    # ลบบรรทัดก่อนหน้า
    @output.print "\r" if @output.tty?
    @output.print "Progress: #{@current}/#{@total} (#{pct}%) "
    @output.print "ETA: #{eta.round.to_i}s " if eta > 0
    @output.flush
    
    @output.puts if @current >= @total
  end
end

progress = ProgressReporter.new(100)
100.times do |i|
  sleep 0.01
  progress.increment
end
```

---

## 6. Interactive Input Handling

```crystal
# Prompt utilities
def prompt(message : String, default : String? = nil) : String
  if default
    STDOUT.print "#{message} [#{default}]: "
  else
    STDOUT.print "#{message}: "
  end
  STDOUT.flush
  
  input = gets(chomp: true) || ""
  input.empty? && default ? default : input
end

def confirm(message : String, default : Bool = false) : Bool
  options = default ? "[Y/n]" : "[y/N]"
  STDOUT.print "#{message} #{options}: "
  STDOUT.flush
  
  input = gets(chomp: true) || ""
  
  case input.downcase
  when "y", "yes" then true
  when "n", "no"  then false
  else default
  end
end

def choose(message : String, options : Array(String)) : String
  puts message
  options.each_with_index do |opt, i|
    puts "  #{i + 1}) #{opt}"
  end
  STDOUT.print "Choose (1-#{options.size}): "
  STDOUT.flush
  
  loop do
    input = gets(chomp: true) || "1"
    idx = input.to_i? || 0
    return options[idx - 1] if idx.in?(1..options.size)
    STDOUT.print "Invalid choice, try again: "
    STDOUT.flush
  end
end

# ใช้งาน (uncomment เพื่อทดสอบ interactive)
# name = prompt("Your name", "Anonymous")
# if confirm("Continue?")
#   lang = choose("Select language:", ["Crystal", "Ruby", "Go"])
# end

# Read password (no echo)
def read_password(prompt_text : String = "Password: ") : String
  STDOUT.print prompt_text
  STDOUT.flush
  
  # ปิด echo (Unix only)
  # ในงานจริงต้องใช้ termios หรือ library
  password = gets(chomp: true) || ""
  STDOUT.puts  # newline after hidden input
  password
end

# Multi-line input
def read_multiline(prompt_text : String = "Enter text (empty line to end):") : String
  puts prompt_text
  lines = [] of String
  
  while line = gets(chomp: true)
    break if line.empty?
    lines << line
  end
  
  lines.join("\n")
end
```

---

## 7. Streaming Output

```crystal
# Streaming สำหรับ real-time output
def stream_process(items : Array(String), output : IO = STDOUT)
  items.each_with_index do |item, i|
    output.printf("[%d/%d] Processing: %s\n", i + 1, items.size, item)
    output.flush
    sleep 0.1  # simulate work
  end
end

stream_process(["File A", "File B", "File C", "File D"])

# Spinner animation
def with_spinner(message : String, output : IO = STDOUT, &block)
  return block.call unless output.tty?
  
  spinner_chars = ["|", "/", "-", "\\"]
  done = false
  result = nil
  
  spinner_fiber = spawn do
    i = 0
    until done
      output.print "\r#{spinner_chars[i % 4]} #{message}..."
      output.flush
      sleep 0.1
      i += 1
    end
  end
  
  result = block.call
  done = true
  Fiber.yield
  
  output.print "\r✓ #{message}    \n"
  output.flush
  
  result
end

with_spinner("Processing") do
  sleep 2
  "done"
end

# ANSI escape codes
module Terminal
  def self.clear_line
    "\r\e[2K"
  end
  
  def self.move_up(n : Int32 = 1)
    "\e[#{n}A"
  end
  
  def self.bold(text : String)
    "\e[1m#{text}\e[0m"
  end
  
  def self.color(text : String, code : Int32)
    "\e[#{code}m#{text}\e[0m"
  end
  
  def self.red(text : String)    = color(text, 31)
  def self.green(text : String)  = color(text, 32)
  def self.yellow(text : String) = color(text, 33)
  def self.blue(text : String)   = color(text, 34)
  def self.cyan(text : String)   = color(text, 36)
end

if STDOUT.tty?
  puts Terminal.bold("Bold text")
  puts Terminal.red("Error message")
  puts Terminal.green("Success!")
  puts Terminal.yellow("Warning!")
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `InteractiveMenu` class ที่แสดง menu และรับ input จาก user พร้อม arrow key navigation

### แบบฝึกหัดที่ 2
เขียน `PipelineProcessor` ที่อ่านจาก STDIN, process, และเขียนไป STDOUT (filter/transform mode)

### แบบฝึกหัดที่ 3
เขียน `LogTee` ที่ส่ง output ทั้งไปยัง STDOUT และ log file พร้อม timestamp

### เฉลย

```crystal
# แบบฝึกหัดที่ 2: Pipeline Processor
class PipelineProcessor
  def initialize(@transform : String -> String?)
  end
  
  def process(input : IO = STDIN, output : IO = STDOUT)
    input.each_line(chomp: true) do |line|
      if result = @transform.call(line)
        output.puts result
      end
    end
  end
end

# ตัวอย่าง: filter และ transform
upcase_non_empty = PipelineProcessor.new do |line|
  line.empty? ? nil : line.upcase
end

# ทดสอบ
input_io = IO::Memory.new("hello\n\nworld\n\nfoo")
output_io = IO::Memory.new
upcase_non_empty.process(input_io, output_io)
puts output_io.to_s
# => HELLO
# => WORLD
# => FOO

# แบบฝึกหัดที่ 3: LogTee
class LogTee
  def initialize(log_path : String, output : IO = STDOUT)
    @output = output
    @log_file = File.open(log_path, "a")
  end
  
  def puts(message : String = "")
    timestamp = Time.local.to_s("[%Y-%m-%d %H:%M:%S] ")
    @output.puts message
    @log_file.puts timestamp + message
    @log_file.flush
  end
  
  def print(message : String)
    @output.print message
    @log_file.print message
    @log_file.flush
  end
  
  def close
    @log_file.close
  end
end

# ใช้งาน
log = LogTee.new("/tmp/app.log")
log.puts "Application started"
log.puts "Processing data..."
log.puts "Done!"
log.close

puts "\nLog file contents:"
puts File.read("/tmp/app.log")
File.delete("/tmp/app.log")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **STDIN** - อ่าน input จาก keyboard หรือ pipe
2. **STDOUT** - เขียน output ปกติ
3. **STDERR** - เขียน error messages
4. **Redirecting** - ส่ง IO object แทน standard streams
5. **IO::Memory** - fake stdin/stdout สำหรับ testing
6. **Interactive Input** - prompt, confirm, choose
7. **Streaming Output** - real-time progress และ animations
8. **ANSI Escape Codes** - colorize terminal output

การใช้ IO abstraction ทำให้ code flexible และ testable มากขึ้น
