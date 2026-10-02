# Part 73: IO Streams

## บทนำ

IO Module ใน Crystal เป็น abstract interface สำหรับ input/output operations ทุกอย่างที่รับ/ส่งข้อมูลใน Crystal ใช้ `IO` เป็น base รวมถึง File, Socket, HTTP และ pipe

---

## 1. IO Module พื้นฐาน

```crystal
# IO เป็น abstract class/module
# ทุก IO object รองรับ read/write operations

# เขียนไปยัง IO
def write_to(io : IO, data : String)
  io << data
  io.puts  # newline
end

# ทำงานกับ STDOUT, File, หรือ IO::Memory ได้เหมือนกัน
write_to(STDOUT, "Hello STDOUT")

buf = IO::Memory.new
write_to(buf, "Hello Buffer")
puts buf.to_s  # => "Hello Buffer\n"

File.open("/tmp/test.txt", "w") do |f|
  write_to(f, "Hello File")
end

# IO methods
io = IO::Memory.new("Hello World")
puts io.gets        # => "Hello World"
puts io.eof?        # => true

io.rewind
puts io.read_char   # => 'H'
puts io.read_char   # => 'e'

# อ่านทุกอย่าง
io.rewind
puts io.gets_to_end  # => "Hello World"
```

---

## 2. read/write/gets/puts บน IO

```crystal
# IO#write (bytes)
io = IO::Memory.new
io.write("Hello".to_slice)  # เขียน Bytes
io.rewind
puts io.gets_to_end  # => "Hello"

# IO#read (bytes)
io = IO::Memory.new("Hello World")
buf = Bytes.new(5)
n = io.read(buf)
puts String.new(buf[0, n])  # => "Hello"

# IO#puts
io = IO::Memory.new
io.puts "Line 1"
io.puts "Line 2"
io.puts "Line 3"
io.rewind
io.each_line { |l| puts l }

# IO#print
io = IO::Memory.new
io.print "No "
io.print "newline"
io.rewind
puts io.gets_to_end.inspect  # => "No newline"

# IO#gets
io = IO::Memory.new("line1\nline2\nline3")
puts io.gets           # => "line1\n"
puts io.gets(chomp: true)  # => "line2"

# IO#gets กับ delimiter
io = IO::Memory.new("a,b,c,d")
puts io.gets(',')  # => "a,"
puts io.gets(',')  # => "b,"

# IO#read_char
io = IO::Memory.new("ABC")
puts io.read_char.inspect  # => 'A'
puts io.read_char.inspect  # => 'B'
puts io.read_char.inspect  # => 'C'
puts io.read_char.inspect  # => nil (EOF)

# IO#each_line
IO::Memory.new("a\nb\nc").each_line do |line|
  puts "Line: #{line}"
end
```

---

## 3. Pipe IO

```crystal
# IO.pipe สร้าง pipe (reader, writer) pair
reader, writer = IO.pipe

# เขียนใน background fiber
spawn do
  writer.puts "Hello from pipe"
  writer.puts "Second message"
  writer.close
end

# อ่านใน main fiber
while line = reader.gets(chomp: true)
  puts "Got: #{line}"
end

# ตัวอย่าง: producer-consumer
def create_pipeline(items : Array(String)) : IO
  reader, writer = IO.pipe
  
  spawn do
    items.each do |item|
      writer.puts item
    end
    writer.close
  end
  
  reader
end

stream = create_pipeline(["item1", "item2", "item3"])
while line = stream.gets(chomp: true)
  puts "Processing: #{line}"
end

# Bidirectional pipes
child_reader, parent_writer = IO.pipe
parent_reader, child_writer = IO.pipe

# Parent process writes, child process reads and responds
spawn do
  # Child
  while msg = child_reader.gets(chomp: true)
    child_writer.puts msg.upcase
  end
  child_writer.close
end

# Parent
["hello", "world"].each do |msg|
  parent_writer.puts msg
end
parent_writer.close

while response = parent_reader.gets(chomp: true)
  puts "Response: #{response}"
end
# => Response: HELLO
# => Response: WORLD
```

---

## 4. Tee IO

```crystal
# IO::Tee: เขียนไปหลาย IO พร้อมกัน
# (ต้อง implement เอง หรือใช้ IO::MultiWriter)

class TeeIO < IO
  def initialize(*ios : IO)
    @ios = ios.to_a
  end
  
  def write(slice : Bytes) : Nil
    @ios.each { |io| io.write(slice) }
  end
  
  def read(slice : Bytes) : Int32
    raise IO::Error.new("TeeIO does not support reading")
  end
  
  def close
    @ios.each(&.close)
  end
end

# ใช้งาน: เขียนทั้ง STDOUT และ file พร้อมกัน
File.open("/tmp/output.txt", "w") do |file|
  tee = TeeIO.new(STDOUT, file)
  
  tee.puts "This goes to both stdout and file"
  tee.puts "So does this"
  tee.puts "And this too"
  
  # ไม่ปิด STDOUT
end

puts File.read("/tmp/output.txt")

# Logger ที่ใช้ TeeIO
class TeeLogger
  def initialize(log_path : String)
    @file = File.open(log_path, "a")
    @tee = TeeIO.new(STDOUT, @file)
  end
  
  def log(level : String, message : String)
    timestamp = Time.local.to_s("%Y-%m-%d %H:%M:%S")
    @tee.puts "[#{timestamp}] #{level.upcase}: #{message}"
    @tee.flush
  end
  
  def close
    @file.close
  end
end
```

---

## 5. Buffered IO

```crystal
# IO::Buffered - buffer reads/writes
# File operations ปกติ ใช้ buffering แล้ว

# flush - force write buffer ออก
File.open("/tmp/test.txt", "w") do |f|
  f.puts "Line 1"
  f.flush  # force write to disk
  f.puts "Line 2"
  # flush อัตโนมัติเมื่อ close
end

# sync= mode - ไม่ buffer (ทุก write ไปที่ disk ทันที)
File.open("/tmp/test.txt", "w") do |f|
  f.sync = true  # disable buffering
  f.puts "Important data"  # written immediately
end

# สร้าง buffered IO wrapper
class BufferedWriter
  BUFFER_SIZE = 65536  # 64KB
  
  def initialize(@io : IO)
    @buffer = IO::Memory.new(BUFFER_SIZE)
    @bytes_written = 0
  end
  
  def write(data : String)
    @buffer << data
    flush if @buffer.pos >= BUFFER_SIZE
  end
  
  def flush
    return if @buffer.pos == 0
    @buffer.rewind
    IO.copy(@buffer, @io)
    @bytes_written += @buffer.pos
    @buffer = IO::Memory.new(BUFFER_SIZE)
  end
  
  def close
    flush
    @io.close
  end
  
  def bytes_written : Int64
    @bytes_written.to_i64
  end
end

# ใช้งาน
File.open("/tmp/large.txt", "w") do |f|
  writer = BufferedWriter.new(f)
  10000.times { |i| writer.write("Line #{i}\n") }
  writer.flush
end
```

---

## 6. Custom IO Implementation

```crystal
# สร้าง custom IO class
class UppercaseIO < IO
  def initialize(@underlying : IO)
  end
  
  def write(slice : Bytes) : Nil
    # แปลง uppercase ก่อนเขียน
    str = String.new(slice)
    @underlying.write(str.upcase.to_slice)
  end
  
  def read(slice : Bytes) : Int32
    @underlying.read(slice)
  end
end

# ใช้งาน
upper_io = UppercaseIO.new(STDOUT)
upper_io.puts "hello world"  # => HELLO WORLD
upper_io.puts "crystal rocks!"  # => CRYSTAL ROCKS!

# LineCountIO
class LineCountIO < IO
  getter line_count : Int32 = 0
  getter byte_count : Int64 = 0
  
  def initialize(@underlying : IO)
  end
  
  def write(slice : Bytes) : Nil
    @underlying.write(slice)
    @byte_count += slice.size
    @line_count += slice.count('\n'.ord.to_u8)
  end
  
  def read(slice : Bytes) : Int32
    @underlying.read(slice)
  end
end

counter = LineCountIO.new(IO::Memory.new)
counter.puts "Line 1"
counter.puts "Line 2"
counter.puts "Line 3"
puts "Lines: #{counter.line_count}"   # => 3
puts "Bytes: #{counter.byte_count}"   # includes newlines

# HashIO - compute hash while writing
require "digest/sha256"

class HashIO < IO
  getter digest : String { @sha256.final.hexstring }
  
  def initialize(@underlying : IO)
    @sha256 = Digest::SHA256.new
  end
  
  def write(slice : Bytes) : Nil
    @underlying.write(slice)
    @sha256.update(slice)
  end
  
  def read(slice : Bytes) : Int32
    @underlying.read(slice)
  end
end

hash_io = HashIO.new(IO::Memory.new)
hash_io.puts "Hello, World!"
puts "SHA256: #{hash_io.digest}"
```

---

## 7. IO.pipe ขั้นสูง

```crystal
# ใช้ pipe กับ fibers สำหรับ concurrent processing
def parallel_process(items : Array(String), workers : Int32 = 4) : Array(String)
  results = [] of String
  mutex = Mutex.new
  
  # สร้าง work queue
  work_reader, work_writer = IO.pipe
  result_reader, result_writer = IO.pipe
  
  # Workers
  workers.times do
    spawn do
      while task = work_reader.gets(chomp: true)
        result = task.upcase  # simulate processing
        result_writer.puts result
      end
    end
  end
  
  # Feed work
  spawn do
    items.each { |item| work_writer.puts item }
    work_writer.close
  end
  
  # Collect results
  items.size.times do
    if result = result_reader.gets(chomp: true)
      results << result
    end
  end
  
  results
end

# ใช้ IO::Memory เพื่อ capture output ใน tests
class OutputCapture
  getter output : String = ""
  
  def capture(&block : IO ->) : String
    io = IO::Memory.new
    block.call(io)
    io.to_s
  end
end

def greet(name : String, io : IO = STDOUT)
  io.puts "Hello, #{name}!"
end

capture = OutputCapture.new
result = capture.capture { |io| greet("Alice", io) }
puts result.strip  # => "Hello, Alice!"
```

---

## 8. Reading Chunks

```crystal
# อ่านไฟล์ทีละ chunk (memory efficient)
def read_in_chunks(path : String, chunk_size : Int32 = 65536, &block : Bytes ->)
  File.open(path, "rb") do |f|
    buffer = Bytes.new(chunk_size)
    while (n = f.read(buffer)) > 0
      block.call(buffer[0, n])
    end
  end
end

# นับ bytes ใน file
total_bytes = 0i64
read_in_chunks("large_file.txt") do |chunk|
  total_bytes += chunk.size
end
puts "Total: #{total_bytes} bytes"

# คำนวณ hash ของ file ขนาดใหญ่
def file_sha256(path : String) : String
  require "digest/sha256"
  digest = Digest::SHA256.new
  
  read_in_chunks(path) do |chunk|
    digest.update(chunk)
  end
  
  digest.final.hexstring
end

# copy file ทีละ chunk (สำหรับไฟล์ขนาดใหญ่)
def copy_with_progress(src : String, dst : String, &progress : Int64, Int64 ->)
  total = File.size(src)
  copied = 0i64
  
  File.open(src, "rb") do |source|
    File.open(dst, "wb") do |dest|
      buffer = Bytes.new(65536)
      while (n = source.read(buffer)) > 0
        dest.write(buffer[0, n])
        copied += n
        progress.call(copied, total)
      end
    end
  end
end

# IO.copy utility
File.open("source.txt", "rb") do |src|
  File.open("dest.txt", "wb") do |dst|
    IO.copy(src, dst)
  end
end

# IO.copy กับ limit
File.open("large.bin", "rb") do |src|
  File.open("first_1mb.bin", "wb") do |dst|
    IO.copy(src, dst, read_size: 1_000_000)  # copy first 1MB
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `ThrottledIO` ที่ limit write speed (simulate slow network)

### แบบฝึกหัดที่ 2
เขียน `EncryptedIO` ที่ encrypt/decrypt ข้อมูลขณะ read/write ด้วย simple XOR

### แบบฝึกหัดที่ 3
เขียน `MultiReaderIO` ที่อ่านจากหลาย IO sources พร้อมกัน (round-robin)

### เฉลย

```crystal
# แบบฝึกหัดที่ 2: XOR EncryptedIO
class XorEncryptedIO < IO
  def initialize(@underlying : IO, @key : UInt8)
  end
  
  def write(slice : Bytes) : Nil
    encrypted = slice.map { |b| b ^ @key }
    @underlying.write(encrypted)
  end
  
  def read(slice : Bytes) : Int32
    n = @underlying.read(slice)
    n.times { |i| slice[i] = slice[i] ^ @key }
    n
  end
end

# ทดสอบ
buffer = IO::Memory.new

# Encrypt
enc = XorEncryptedIO.new(buffer, 42u8)
enc.puts "Secret message"
enc.puts "Another secret"

# Decrypt
buffer.rewind
dec = XorEncryptedIO.new(buffer, 42u8)
puts dec.gets_to_end

# แบบฝึกหัดที่ 3: MultiReaderIO
class MultiReaderIO < IO
  def initialize(*sources : IO)
    @sources = sources.to_a
    @current = 0
  end
  
  def read(slice : Bytes) : Int32
    while @current < @sources.size
      n = @sources[@current].read(slice)
      return n if n > 0
      @current += 1
    end
    0
  end
  
  def write(slice : Bytes) : Nil
    raise IO::Error.new("MultiReaderIO is read-only")
  end
end

s1 = IO::Memory.new("Hello ")
s2 = IO::Memory.new("World")
s3 = IO::Memory.new("!")

multi = MultiReaderIO.new(s1, s2, s3)
puts multi.gets_to_end  # => "Hello World!"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **IO Module** - abstract interface สำหรับ I/O operations
2. **read/write/gets/puts** - methods พื้นฐานของ IO
3. **IO.pipe** - สร้าง pipe ระหว่าง IOs
4. **TeeIO** - เขียนไปหลาย IO พร้อมกัน
5. **Buffered IO** - optimize performance ด้วย buffering
6. **Custom IO** - สร้าง IO class ของตัวเอง
7. **Reading Chunks** - อ่านไฟล์ขนาดใหญ่อย่างมีประสิทธิภาพ

IO abstraction ใน Crystal ทำให้เขียน code ที่ reusable กับ IO sources ต่างๆ ได้ง่าย
