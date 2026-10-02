# Part 80: Pipes และ Subprocesses

## บทนำ

Pipes เป็นกลไก inter-process communication (IPC) ที่ให้ processes สื่อสารกันผ่าน data stream Crystal รองรับ pipe ทั้งแบบ anonymous pipe (IO.pipe) และ named pipe รวมถึงการ spawn subprocess พร้อม pipe

---

## 1. IO.pipe

```crystal
# IO.pipe สร้าง anonymous pipe
# คืน tuple {read_end, write_end}
reader, writer = IO.pipe

# เขียนข้อมูลเข้า writer
writer.puts "Hello from parent"
writer.puts "Second line"
writer.close  # ต้อง close เพื่อส่ง EOF

# อ่านจาก reader
puts reader.gets  # => "Hello from parent"
puts reader.gets  # => "Second line"
puts reader.gets.inspect  # => nil (EOF)
reader.close

# Pipe กับ Fiber/spawn
reader, writer = IO.pipe

spawn do
  writer.puts "Message from fiber"
  writer.puts "Another message"
  writer.close
end

Fiber.yield

while line = reader.gets
  puts "Got: #{line}"
end
reader.close

# Binary pipe
reader, writer = IO.pipe

spawn do
  writer.write(Bytes[1, 2, 3, 4, 5])
  writer.close
end

Fiber.yield

buf = Bytes.new(10)
n = reader.read(buf)
puts buf[0, n].inspect  # => Bytes[1, 2, 3, 4, 5]
reader.close
```

---

## 2. Spawning Processes with Pipes

```crystal
# Process.new กับ stdin/stdout pipe
stdout = IO::Memory.new
Process.run("echo", args: ["hello world"], output: stdout)
puts stdout.to_s.strip  # => "hello world"

# ส่งข้อมูลผ่าน stdin
stdin_data = IO::Memory.new("line1\nline2\nline3\n")
stdout = IO::Memory.new
Process.run("sort", input: stdin_data, output: stdout)
puts stdout.to_s

# Pipeline simulation: cat | sort | uniq
data = "banana\napple\ncherry\napple\nbanana\ndate\n"
step1 = IO::Memory.new(data)

# sort
sorted = IO::Memory.new
Process.run("sort", input: step1, output: sorted)

# uniq
sorted.rewind
unique = IO::Memory.new
Process.run("uniq", input: sorted, output: unique)

puts unique.to_s
```

---

## 3. Reading Process Output

```crystal
# อ่าน output ทีละบรรทัด
reader, writer = IO.pipe

process = Process.new("ls", args: ["-la"], output: writer)
writer.close  # parent ไม่ต้องการ write end

lines = [] of String
while line = reader.gets
  lines << line
end
reader.close
process.wait

puts "Got #{lines.size} lines"
lines.each { |l| puts l }

# Streaming output processing
def process_output_streaming(command : String, args : Array(String) = [] of String, &block : String ->)
  reader, writer = IO.pipe
  
  process = Process.new(command, args: args, output: writer)
  writer.close
  
  while line = reader.gets
    block.call(line)
  end
  
  reader.close
  process.wait
end

process_output_streaming("find", ["/tmp", "-name", "*.log", "-type", "f"]) do |line|
  puts "Found: #{line}"
end

# Capture with timeout
def capture_with_timeout(command : String, args : Array(String), timeout_seconds : Float64) : String?
  stdout = IO::Memory.new
  stderr = IO::Memory.new
  
  process = Process.new(command, args: args, output: stdout, error: stderr)
  
  # Simple timeout using channel
  done = Channel(Bool).new(1)
  
  spawn do
    process.wait
    done.send(true)
  end
  
  select
  when done.receive
    stdout.to_s
  when timeout(timeout_seconds.seconds)
    process.terminate
    nil
  end
end
```

---

## 4. Piping to stdin of Subprocess

```crystal
# ส่งข้อมูลไปยัง stdin ของ subprocess
def pipe_to_process(command : String, args : Array(String), input : String) : {Int32, String}
  stdin_io = IO::Memory.new(input)
  stdout_io = IO::Memory.new
  
  result = Process.run(command, args: args, input: stdin_io, output: stdout_io)
  
  {result.exit_code, stdout_io.to_s}
end

# grep ผ่าน stdin
text = "apple\nbanana\ncherry\napricot\navocado\n"
code, output = pipe_to_process("grep", ["^a"], text)
puts "Matches:\n#{output}"

# wc -l ผ่าน stdin
lines = "line1\nline2\nline3\n"
code, count = pipe_to_process("wc", ["-l"], lines)
puts "Line count: #{count.strip}"

# ส่ง large data
def stream_to_process(command : String, args : Array(String), &block : IO ->)
  reader, writer = IO.pipe
  
  # spawn writer fiber
  spawn do
    begin
      block.call(writer)
    ensure
      writer.close
    end
  end
  
  stdout = IO::Memory.new
  process = Process.new(command, args: args, input: reader, output: stdout)
  reader.close  # parent ไม่ต้องการ read end หลังจาก pass ให้ process แล้ว
  process.wait
  
  stdout.to_s
end

result = stream_to_process("sort") do |io|
  ["banana", "apple", "cherry", "date"].each do |word|
    io.puts word
  end
end
puts result
```

---

## 5. Bidirectional Pipes

```crystal
# Two-way communication กับ subprocess
class ProcessPipe
  getter process : Process
  getter input : IO
  getter output : IO
  
  def initialize(command : String, args : Array(String) = [] of String)
    # stdin pipe: parent writes, child reads
    stdin_r, @input = IO.pipe
    
    # stdout pipe: child writes, parent reads
    @output, stdout_w = IO.pipe
    
    @process = Process.new(command, args: args, 
                           input: stdin_r, 
                           output: stdout_w)
    
    # ปิด ends ที่ parent ไม่ต้องการ
    stdin_r.close
    stdout_w.close
  end
  
  def send(message : String)
    @input.puts message
    @input.flush
  end
  
  def receive : String?
    @output.gets
  end
  
  def close
    @input.close
    @output.close
    @process.wait
  end
end

# ตัวอย่าง: bc (calculator)
# pipe = ProcessPipe.new("bc")
# pipe.send("2 + 3")
# puts pipe.receive  # => "5"
# pipe.send("10 * 5")
# puts pipe.receive  # => "50"
# pipe.close
```

---

## 6. Process Pipelines

```crystal
# สร้าง pipeline จาก multiple processes
class Pipeline
  def initialize
    @stages = [] of {String, Array(String)}
  end
  
  def pipe(command : String, *args : String) : self
    @stages << {command, args.to_a}
    self
  end
  
  def run(input : String = "") : {Int32, String}
    return {0, input} if @stages.empty?
    
    current_input = IO::Memory.new(input)
    
    @stages.each_with_index do |(cmd, args), i|
      output = IO::Memory.new
      result = Process.run(cmd, args: args, input: current_input, output: output)
      
      output.rewind
      current_input = output
      
      return {result.exit_code, output.to_s} if !result.success? && i < @stages.size - 1
    end
    
    current_input.rewind
    {0, current_input.gets_to_end}
  end
end

# ใช้งาน
text = "Hello World\nhello crystal\nHELLO FIBER\nGoodbye\n"

code, result = Pipeline.new
  .pipe("grep", "-i", "hello")
  .pipe("sort")
  .run(text)

puts "Exit: #{code}"
puts "Result:\n#{result}"

# Shell-style pipeline helper
def sh_pipe(*commands : String, input : String = "") : String
  current = input
  commands.each do |cmd|
    parts = cmd.split
    stdout = IO::Memory.new
    stdin = IO::Memory.new(current)
    Process.run(parts.first, args: parts[1..], input: stdin, output: stdout)
    current = stdout.to_s
  end
  current
end

result = sh_pipe("sort", "uniq", input: "b\na\nc\nb\na\n")
puts result
```

---

## 7. Named Pipes (FIFO)

```crystal
# Named pipes (FIFO) สำหรับ inter-process communication
# ต้องใช้ system calls ผ่าน LibC

# สร้าง FIFO
fifo_path = "/tmp/myfifo"

# ลบ old FIFO ถ้ามี
File.delete(fifo_path) if File.exists?(fifo_path)

# สร้าง FIFO ด้วย mkfifo command
Process.run("mkfifo", args: [fifo_path])

if File.exists?(fifo_path)
  puts "FIFO created: #{fifo_path}"
  
  # Writer process
  spawn do
    File.open(fifo_path, "w") do |f|
      3.times do |i|
        f.puts "Message #{i + 1}"
        sleep 0.1
      end
    end
  end
  
  # Reader
  File.open(fifo_path, "r") do |f|
    while line = f.gets
      puts "Read: #{line}"
    end
  end
  
  File.delete(fifo_path)
end
```

---

## 8. Process Communication Patterns

```crystal
# Pattern 1: Producer-Consumer ผ่าน pipe
def producer_consumer_pattern
  reader, writer = IO.pipe
  results = [] of String
  
  # Producer
  producer = spawn do
    10.times do |i|
      writer.puts "item_#{i}"
      sleep 0.01
    end
    writer.close
  end
  
  # Consumer
  consumer = spawn do
    while item = reader.gets
      results << item.upcase
    end
    reader.close
  end
  
  Fiber.yield
  Fiber.yield
  
  puts "Processed: #{results.size} items"
  results.first(3).each { |r| puts "  #{r}" }
end

producer_consumer_pattern

# Pattern 2: Worker pool ด้วย processes
class WorkerPool
  def initialize(@num_workers : Int32)
    @work_channel = Channel(String).new(100)
    @result_channel = Channel(String).new(100)
  end
  
  def start_workers(command : String)
    @num_workers.times do
      spawn do
        while job = @work_channel.receive?
          stdout = IO::Memory.new
          Process.run(command, args: [job], output: stdout)
          @result_channel.send(stdout.to_s.strip)
        end
      end
    end
  end
  
  def submit(job : String)
    @work_channel.send(job)
  end
  
  def collect(count : Int32) : Array(String)
    Array.new(count) { @result_channel.receive }
  end
  
  def shutdown
    @num_workers.times { @work_channel.close }
  end
end

# Pattern 3: Subprocess supervisor
class SubprocessSupervisor
  def initialize
    @processes = {} of String => Process
  end
  
  def start(name : String, command : String, args : Array(String) = [] of String)
    return if @processes[name]?
    @processes[name] = Process.new(command, args: args)
    puts "Started #{name} (PID: #{@processes[name].pid})"
  end
  
  def stop(name : String)
    process = @processes.delete(name)
    if process
      process.terminate
      process.wait
      puts "Stopped #{name}"
    end
  end
  
  def stop_all
    @processes.keys.dup.each { |name| stop(name) }
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `TextProcessor` ที่รับ text input และประมวลผลผ่าน pipeline ของ commands

### แบบฝึกหัดที่ 2
เขียน `ProcessMonitor` ที่ monitor stdout ของ subprocess และ alert เมื่อพบ pattern

### เฉลย

```crystal
# แบบฝึกหัดที่ 1: TextProcessor Pipeline
class TextProcessor
  def initialize(@text : String)
  end
  
  def grep(pattern : String) : TextProcessor
    result = @text.lines.select { |l| l.match?(Regex.new(pattern)) }.join
    TextProcessor.new(result)
  end
  
  def sort : TextProcessor
    TextProcessor.new(@text.lines.sort.join)
  end
  
  def uniq : TextProcessor
    TextProcessor.new(@text.lines.uniq.join)
  end
  
  def head(n : Int32) : TextProcessor
    TextProcessor.new(@text.lines.first(n).join)
  end
  
  def tail(n : Int32) : TextProcessor
    TextProcessor.new(@text.lines.last(n).join)
  end
  
  def tr(from : String, to : String) : TextProcessor
    TextProcessor.new(@text.tr(from, to))
  end
  
  def result : String
    @text
  end
  
  def count : Int32
    @text.lines.size
  end
end

sample = "banana\napple\ncherry\nAPPLE\nBanana\ndate\n"

result = TextProcessor.new(sample)
  .tr("A-Z", "a-z")
  .sort
  .uniq
  .result

puts result
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **IO.pipe** - สร้าง anonymous pipe
2. **Process กับ pipes** - input/output redirection
3. **Reading process output** - streaming, buffered
4. **Piping to stdin** - ส่งข้อมูลเข้า subprocess
5. **Bidirectional pipes** - two-way communication
6. **Process pipelines** - chaining commands
7. **Named pipes** - FIFO สำหรับ IPC
8. **Communication patterns** - producer-consumer, worker pool

Pipes เป็นเครื่องมือสำคัญสำหรับ system programming และการเชื่อมต่อ processes เข้าด้วยกัน
