# Part 71: File I/O พื้นฐาน

## บทนำ

File I/O เป็นทักษะสำคัญในการเขียนโปรแกรมที่ต้องจัดการกับไฟล์ Crystal มี `File` class ที่ครอบคลุมทุกอย่างตั้งแต่การอ่าน เขียน ไปจนถึงการจัดการ metadata ของไฟล์

---

## 1. File.read - อ่านไฟล์

```crystal
# อ่านไฟล์ทั้งหมดเป็น String
content = File.read("example.txt")
puts content

# อ่านไฟล์ที่อาจไม่มีอยู่ (handle error)
begin
  content = File.read("nonexistent.txt")
rescue File::NotFoundError
  puts "File not found!"
rescue IO::Error => e
  puts "IO error: #{e.message}"
end

# อ่านเป็น Array of lines
lines = File.read_lines("example.txt")
lines.each_with_index do |line, i|
  puts "#{i + 1}: #{line}"
end

# อ่านทีละ line (memory efficient)
File.each_line("example.txt") do |line|
  puts line
end

# อ่านด้วย encoding (Crystal ใช้ UTF-8 เป็น default)
content = File.read("thai.txt")  # UTF-8
puts content.valid_encoding?

# อ่าน binary file
binary_data = File.read("image.png")
puts binary_data.bytesize
```

---

## 2. File.write - เขียนไฟล์

```crystal
# เขียน string ลงไฟล์ (overwrite)
File.write("output.txt", "Hello, World!\n")

# เขียนหลายบรรทัด
content = "Line 1\nLine 2\nLine 3\n"
File.write("multiline.txt", content)

# เขียนด้วย array of lines
lines = ["Line 1", "Line 2", "Line 3"]
File.write("lines.txt", lines.join("\n"))

# เขียน binary data
bytes = Bytes[72, 101, 108, 108, 111]  # "Hello"
File.write("binary.bin", String.new(bytes))

# append mode
File.write("log.txt", "First entry\n")
File.write("log.txt", "Second entry\n", mode: "a")  # append
File.write("log.txt", "Third entry\n", mode: "a")

puts File.read("log.txt")
# => First entry
# => Second entry
# => Third entry
```

---

## 3. File.open กับ Block

```crystal
# File.open กับ block (auto-close)
File.open("example.txt", "r") do |f|
  puts f.gets_to_end
end  # ปิดอัตโนมัติ

# เขียนไฟล์ด้วย block
File.open("output.txt", "w") do |f|
  f.puts "Line 1"
  f.puts "Line 2"
  f.printf("Formatted: %05d\n", 42)
end

# append
File.open("log.txt", "a") do |f|
  f.puts "#{Time.local}: New log entry"
end

# read + write mode
File.open("data.txt", "r+") do |f|
  content = f.gets_to_end
  f.rewind
  f.print(content.upcase)
end

# ทำงานกับ lines
File.open("data.txt", "w") do |f|
  1.upto(5) do |i|
    f.puts "Line #{i}: #{Random::Secure.hex(4)}"
  end
end

File.open("data.txt") do |f|
  while line = f.gets(chomp: true)
    puts "Read: #{line}"
  end
end
```

---

## 4. IO::FileDescriptor

```crystal
# IO::FileDescriptor เป็น low-level file access
# ใช้ File แทนในงานทั่วไป

fd = File.open("test.txt", "w")
fd.write("Hello ".to_slice)
fd.write("World".to_slice)
fd.close

# อ่าน bytes
fd = File.open("test.txt", "r")
buffer = Bytes.new(1024)
bytes_read = fd.read(buffer)
puts String.new(buffer[0, bytes_read])
fd.close

# seek operations
fd = File.open("data.bin", "rb")
fd.seek(10, IO::Seek::Set)    # ไปตำแหน่ง 10 จากต้น
fd.seek(5, IO::Seek::Current) # ไป 5 จากตำแหน่งปัจจุบัน
fd.seek(-3, IO::Seek::End)    # ไป 3 ก่อนจบ
puts fd.pos  # ตำแหน่งปัจจุบัน
fd.close

# ไฟล์ขนาดใหญ่ - อ่านทีละ chunk
def process_large_file(path : String, chunk_size : Int32 = 65536)
  total_lines = 0
  File.open(path, "r") do |f|
    buffer = Bytes.new(chunk_size)
    while (n = f.read(buffer)) > 0
      chunk = String.new(buffer[0, n])
      total_lines += chunk.count('\n')
    end
  end
  total_lines
end
```

---

## 5. Read/Write Modes

```crystal
# Mode strings:
# "r"  - read (default)
# "w"  - write (create/truncate)
# "a"  - append (create if not exists)
# "r+" - read + write
# "w+" - read + write (create/truncate)
# "a+" - read + append

# ตัวอย่างแต่ละ mode
# Read mode
File.open("file.txt", "r") do |f|
  content = f.gets_to_end
  puts "Read: #{content.size} bytes"
end

# Write mode (สร้างใหม่หรือลบเนื้อหาเก่า)
File.open("file.txt", "w") do |f|
  f.puts "New content"
end

# Append mode
File.open("file.txt", "a") do |f|
  f.puts "Appended line"
end

# สร้าง temp file
def with_temp_file(prefix : String = "tmp", suffix : String = ".txt", &block : File ->)
  path = File.tempname(prefix, suffix)
  begin
    File.open(path, "w+") do |f|
      block.call(f)
    end
  ensure
    File.delete(path) if File.exists?(path)
  end
end

with_temp_file("test", ".csv") do |f|
  f.puts "name,age"
  f.puts "Alice,25"
  f.rewind
  puts f.gets_to_end
end
```

---

## 6. Binary vs Text

```crystal
# Text mode: แปลง line endings (บน Windows)
# Binary mode: ไม่แปลง

# เขียน binary file
File.open("binary.dat", "wb") do |f|
  # เขียน magic bytes
  f.write(Bytes[0x89, 0x50, 0x4E, 0x47])  # PNG header
  # เขียน int32 ใน big-endian
  f.write_bytes(12345i32, IO::ByteFormat::BigEndian)
  # เขียน float64
  f.write_bytes(3.14f64, IO::ByteFormat::BigEndian)
end

# อ่าน binary file
File.open("binary.dat", "rb") do |f|
  magic = Bytes.new(4)
  f.read(magic)
  puts "Magic: #{magic.map { |b| "0x#{b.to_s(16).upcase}" }.join(" ")}"
  
  int_val = f.read_bytes(Int32, IO::ByteFormat::BigEndian)
  float_val = f.read_bytes(Float64, IO::ByteFormat::BigEndian)
  
  puts "Int: #{int_val}"
  puts "Float: #{float_val}"
end

# ตรวจสอบ file type จาก magic bytes
def detect_file_type(path : String) : String
  return "empty" if File.size(path) < 4
  
  File.open(path, "rb") do |f|
    magic = Bytes.new(8)
    f.read(magic)
    
    if magic[0..2] == Bytes[0x89, 0x50, 0x4E]
      "PNG image"
    elsif magic[0..2] == Bytes[0xFF, 0xD8, 0xFF]
      "JPEG image"
    elsif magic[0..3] == Bytes[0x47, 0x49, 0x46, 0x38]
      "GIF image"
    elsif magic[0..3] == Bytes[0x25, 0x50, 0x44, 0x46]
      "PDF document"
    elsif String.new(magic[0..3]) == "PK\x03\x04"
      "ZIP archive"
    else
      "Unknown"
    end
  end
end
```

---

## 7. File Metadata

```crystal
# ตรวจสอบว่าไฟล์มีอยู่
puts File.exists?("file.txt")     # => true/false
puts File.file?("file.txt")       # ใช่ file (ไม่ใช่ directory)?
puts File.directory?("mydir")      # ใช่ directory?
puts File.readable?("file.txt")    # อ่านได้?
puts File.writable?("file.txt")    # เขียนได้?
puts File.executable?("script.sh") # รันได้?
puts File.symlink?("link.txt")     # เป็น symlink?

# File size
puts File.size("file.txt")  # size in bytes

# File info
info = File.info("file.txt")
puts info.size          # size
puts info.modification_time  # last modified time
puts info.access_time        # last accessed time
puts info.creation_time      # creation time (ถ้า OS support)
puts info.permissions        # permissions

# Timestamps
puts File.info("file.txt").modification_time.to_s("%Y-%m-%d %H:%M:%S")

# chmod equivalent
File.chmod("script.sh", 0o755)  # rwxr-xr-x

# chown equivalent (ต้องมี permissions)
# File.chown("file.txt", uid: 1000, gid: 1000)

# expand path
puts File.expand_path("~/myfile.txt")  # => /home/user/myfile.txt
puts File.expand_path("../other.txt", "/home/user/projects")  
# => /home/user/other.txt
```

---

## 8. File Delete และ Rename

```crystal
# ลบไฟล์
if File.exists?("temp.txt")
  File.delete("temp.txt")
  puts "Deleted"
end

# ลบโดยไม่ raise error
begin
  File.delete("might_not_exist.txt")
rescue File::NotFoundError
  # ignore
end

# rename / move
File.rename("old_name.txt", "new_name.txt")

# copy file
def copy_file(src : String, dst : String)
  File.open(src, "rb") do |source|
    File.open(dst, "wb") do |dest|
      IO.copy(source, dest)
    end
  end
end

copy_file("original.txt", "backup.txt")

# move (rename cross-filesystem)
def move_file(src : String, dst : String)
  begin
    File.rename(src, dst)
  rescue
    # Cross-filesystem move
    copy_file(src, dst)
    File.delete(src)
  end
end

# ลบไฟล์เก่า (older than N days)
def cleanup_old_files(directory : String, max_age_days : Int32)
  cutoff = Time.local - max_age_days.days
  
  Dir.each_child(directory) do |filename|
    path = File.join(directory, filename)
    next unless File.file?(path)
    
    if File.info(path).modification_time < cutoff
      File.delete(path)
      puts "Deleted: #{path}"
    end
  end
end
```

---

## 9. Practical Examples

```crystal
# Log rotation
class LogRotator
  def initialize(
    @base_path : String,
    @max_size : Int64 = 10_000_000,  # 10MB
    @keep_count : Int32 = 5
  )
  end
  
  def write(message : String)
    rotate if should_rotate?
    
    File.open(@base_path, "a") do |f|
      f.puts "[#{Time.local.to_s("%Y-%m-%d %H:%M:%S")}] #{message}"
    end
  end
  
  private def should_rotate? : Bool
    File.exists?(@base_path) && File.size(@base_path) >= @max_size
  end
  
  private def rotate
    # ลบ log เก่าสุดถ้าเกิน keep_count
    oldest = "#{@base_path}.#{@keep_count}"
    File.delete(oldest) if File.exists?(oldest)
    
    # เลื่อน logs
    (@keep_count - 1).downto(1) do |i|
      old = "#{@base_path}.#{i}"
      new = "#{@base_path}.#{i + 1}"
      File.rename(old, new) if File.exists?(old)
    end
    
    # Rename current log
    File.rename(@base_path, "#{@base_path}.1")
  end
end

# CSV writer
class CSVWriter
  def initialize(@path : String)
    @file = File.open(@path, "w")
  end
  
  def write_row(row : Array(String))
    @file.puts row.map { |cell| escape_csv(cell) }.join(",")
  end
  
  def close
    @file.close
  end
  
  private def escape_csv(value : String) : String
    if value.includes?(",") || value.includes?('"') || value.includes?("\n")
      '"' + value.gsub('"', '""') + '"'
    else
      value
    end
  end
end

writer = CSVWriter.new("data.csv")
writer.write_row(["Name", "Age", "City"])
writer.write_row(["Alice", "25", "Bangkok, Thailand"])
writer.write_row(["Bob", "30", "New York"])
writer.close

puts File.read("data.csv")

# File diff
def file_diff(path1 : String, path2 : String)
  lines1 = File.read_lines(path1, chomp: true)
  lines2 = File.read_lines(path2, chomp: true)
  
  max_lines = [lines1.size, lines2.size].max
  
  max_lines.times do |i|
    l1 = lines1[i]? || ""
    l2 = lines2[i]? || ""
    
    if l1 != l2
      puts "Line #{i + 1}:"
      puts "  < #{l1}"
      puts "  > #{l2}"
    end
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `WordCounter` class ที่อ่านไฟล์และนับจำนวนคำ, บรรทัด, ตัวอักษร (คล้าย wc command)

### แบบฝึกหัดที่ 2
เขียน `FileEncryptor` ที่ encrypt/decrypt ไฟล์ด้วย XOR cipher

### แบบฝึกหัดที่ 3
เขียน `LogAnalyzer` ที่อ่าน log file และสรุป error/warning counts ตาม pattern

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
class WordCounter
  def initialize(@path : String)
  end
  
  def count : {lines: Int32, words: Int32, chars: Int32, bytes: Int64}
    lines = 0
    words = 0
    chars = 0
    
    File.each_line(@path) do |line|
      lines += 1
      words += line.split(/\s+/).reject(&.empty?).size
      chars += line.size
    end
    
    {lines: lines, words: words, chars: chars, bytes: File.size(@path)}
  end
  
  def report : String
    c = count
    String.build do |io|
      io << "File: #{@path}\n"
      io << "Lines: #{c[:lines]}\n"
      io << "Words: #{c[:words]}\n"
      io << "Characters: #{c[:chars]}\n"
      io << "Bytes: #{c[:bytes]}\n"
    end
  end
end

# สร้าง test file
File.write("test.txt", "Hello World\nFoo Bar Baz\nCrystal Programming\n")
counter = WordCounter.new("test.txt")
puts counter.report
File.delete("test.txt")

# แบบฝึกหัดที่ 2
class FileEncryptor
  def initialize(@key : UInt8)
  end
  
  def encrypt(input_path : String, output_path : String)
    File.open(input_path, "rb") do |inp|
      File.open(output_path, "wb") do |out|
        buffer = Bytes.new(4096)
        while (n = inp.read(buffer)) > 0
          encrypted = buffer[0, n].map { |b| b ^ @key }
          out.write(encrypted)
        end
      end
    end
  end
  
  def decrypt(input_path : String, output_path : String)
    encrypt(input_path, output_path)  # XOR is symmetric
  end
end

File.write("original.txt", "Secret message!")
enc = FileEncryptor.new(42u8)
enc.encrypt("original.txt", "encrypted.bin")
enc.decrypt("encrypted.bin", "decrypted.txt")
puts File.read("decrypted.txt")  # => "Secret message!"
File.delete("original.txt")
File.delete("encrypted.bin")
File.delete("decrypted.txt")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **File.read** - อ่านไฟล์ทั้งหมดหรือทีละบรรทัด
2. **File.write** - เขียนไฟล์ในโหมดต่างๆ
3. **File.open กับ block** - จัดการ file lifecycle อัตโนมัติ
4. **IO::FileDescriptor** - low-level file access
5. **Modes** - r, w, a, r+, w+, a+
6. **Binary vs Text** - การจัดการ binary files
7. **File metadata** - ตรวจสอบ properties ของไฟล์
8. **Delete และ Rename** - จัดการ lifecycle ของไฟล์
9. **Practical patterns** - log rotation, CSV writing

File I/O เป็นพื้นฐานสำคัญที่ใช้ในเกือบทุก application
