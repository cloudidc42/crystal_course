# Part 184: Zero-copy Operations ใน Crystal

## บทนำ

Zero-copy คือ technique ที่หลีกเลี่ยงการ copy data ระหว่าง buffers ลด CPU usage และ memory bandwidth โดยเฉพาะสำหรับ I/O-intensive applications

## Slice(T) และ Bytes

```crystal
# Slice(T): view เข้าไปใน memory region โดยไม่ copy
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# สร้าง slice ที่ reference ส่วนหนึ่งของ array
slice1 = arr.to_slice          # entire array
slice2 = arr.to_slice[2..5]    # elements 2-5 (zero-copy view!)
slice3 = arr.to_slice[0, 4]    # elements 0-3 (4 elements)

puts slice2.inspect  # [3, 4, 5, 6]
puts "Same memory? #{slice2.to_unsafe == (arr.to_unsafe + 2)}"  # true!

# แก้ไขผ่าน slice จะแก้ original array
slice2[0] = 100
puts arr[2]  # 100 - original array changed!

# Bytes: alias สำหรับ Slice(UInt8)
data = Bytes[0xDE, 0xAD, 0xBE, 0xEF]
header = data[0, 2]   # bytes 0-1
payload = data[2, 2]  # bytes 2-3

puts header.hexstring   # dead
puts payload.hexstring  # beef

# ไม่มีการ copy เลย!
```

## IO::Memory - In-Memory IO

```crystal
# IO::Memory: string/bytes ที่ทำงานเป็น IO stream
# ดีกว่า String concatenation เพราะ append ทำครั้งเดียว

# Write mode
io = IO::Memory.new
io << "Hello"
io << ", "
io << "World"
io << "!"

puts io.to_s    # "Hello, World!"
puts io.size    # 13

# Read mode
data = Bytes[72, 101, 108, 108, 111]  # "Hello"
io = IO::Memory.new(data)

puts io.read_string(5)  # "Hello"
puts io.pos             # 5

# Peek without advancing position
io.rewind
puts io.peek.inspect    # Slice of next bytes

# IO::Memory สำหรับ serialize/deserialize
class Packet
  property type : UInt8
  property length : UInt16
  property data : Bytes

  def initialize(@type, @length, @data)
  end

  def serialize : Bytes
    io = IO::Memory.new
    io.write_byte(@type)
    io.write_bytes(@length, IO::ByteFormat::BigEndian)
    io.write(@data)
    io.to_slice
  end

  def self.deserialize(bytes : Bytes) : Packet
    io = IO::Memory.new(bytes)
    type = io.read_byte || 0_u8
    length = io.read_bytes(UInt16, IO::ByteFormat::BigEndian)
    data = Bytes.new(length)
    io.read(data)
    new(type, length, data)
  end
end

packet = Packet.new(0x01_u8, 5_u16, Bytes[1, 2, 3, 4, 5])
serialized = packet.serialize
puts serialized.hexstring

restored = Packet.deserialize(serialized)
puts "Type: #{restored.type}"
puts "Data: #{restored.data.hexstring}"
```

## Zero-copy String Operations

```crystal
# String#byte_slice: get view ไม่มีการ copy
str = "Hello, World!"

# สร้าง string slice ใหม่ (ยังคง allocate)
substr = str[0, 5]  # "Hello"

# ใช้ Bytes view ไม่ allocate
bytes = str.to_slice
hello_bytes = bytes[0, 5]  # View of first 5 bytes
puts String.new(hello_bytes)  # "Hello"

# StringView pattern - อ่านส่วนของ string โดยไม่ allocate
struct StringView
  getter bytes : Bytes
  getter start : Int32
  getter length : Int32

  def initialize(str : String, @start : Int32, @length : Int32)
    @bytes = str.to_slice
  end

  def to_s : String
    String.new(@bytes[@start, @length])
  end

  def ==(other : String) : Bool
    return false if @length != other.bytesize
    other_bytes = other.to_slice
    length.times.all? { |i| @bytes[@start + i] == other_bytes[i] }
  end

  def starts_with?(prefix : String) : Bool
    return false if prefix.bytesize > @length
    prefix_bytes = prefix.to_slice
    prefix.bytesize.times.all? { |i| @bytes[@start + i] == prefix_bytes[i] }
  end
end

text = "GET /api/users HTTP/1.1"
method_view = StringView.new(text, 0, 3)
path_view = StringView.new(text, 4, 11)

puts method_view == "GET"           # true
puts path_view.starts_with?("/api") # true
```

## Zero-copy File I/O

```crystal
# sendfile: zero-copy file transfer (Linux)
lib LibC
  fun sendfile(out_fd : Int32, in_fd : Int32, offset : Int64*, count : LibC::SizeT) : LibC::SSizeT
end

# Crystal's built-in IO.copy เป็น zero-copy เมื่อ platform support
def serve_file(socket : IO, path : String)
  File.open(path, "r") do |file|
    IO.copy(file, socket)  # ใช้ sendfile syscall บน Linux
  end
end

# Memory-mapped file: zero-copy file reading
lib LibMmap
  PROT_READ  = 1
  MAP_SHARED = 1
  MAP_FAILED = Pointer(Void).new(UInt64::MAX)

  fun mmap(addr : Void*, length : LibC::SizeT, prot : Int32, flags : Int32, fd : Int32, offset : Int64) : Void*
  fun munmap(addr : Void*, length : LibC::SizeT) : Int32
end

class MemoryMappedFile
  getter data : Bytes

  def initialize(path : String)
    file = File.open(path)
    @size = file.size.to_i
    fd = file.fd

    ptr = LibMmap.mmap(nil, @size.to_u64, LibMmap::PROT_READ, LibMmap::MAP_SHARED, fd, 0_i64)
    raise "mmap failed" if ptr == LibMmap::MAP_FAILED

    @data = Slice.new(ptr.as(UInt8*), @size)
    @ptr = ptr
    file.close
  end

  def finalize
    LibMmap.munmap(@ptr, @size.to_u64) if @ptr
  end

  def size : Int32
    @size
  end
end

mf = MemoryMappedFile.new("/etc/hosts")
puts "File size: #{mf.size}"
puts "First line: #{String.new(mf.data[0..mf.data.index('\n'.ord.to_u8) || 0])}"
```

## Buffer Chain (Scatter-Gather I/O)

```crystal
# Scatter-Gather: write หลาย buffers ใน single syscall
lib LibC
  struct IOVec
    iov_base : Void*
    iov_len : LibC::SizeT
  end

  fun writev(fd : Int32, iov : IOVec*, iovcnt : Int32) : LibC::SSizeT
  fun readv(fd : Int32, iov : IOVec*, iovcnt : Int32) : LibC::SSizeT
end

def scatter_write(fd : Int32, buffers : Array(Bytes)) : Int32
  iov = buffers.map do |buf|
    LibC::IOVec.new(iov_base: buf.to_unsafe.as(Void*), iov_len: buf.size.to_u64)
  end
  LibC.writev(fd, iov.to_unsafe, iov.size).to_i
end

# ใช้สำหรับ HTTP response: headers + body โดยไม่ copy
def send_http_response(socket_fd : Int32, headers : String, body : Bytes)
  header_bytes = headers.to_slice
  scatter_write(socket_fd, [header_bytes, body])
end

# Crystal built-in: IO#write_v สำหรับ gather write
io = IO::Memory.new
buffers = ["HTTP/1.1 200 OK\r\n", "Content-Type: text/plain\r\n", "\r\n", "Hello"]
buffers.each { |b| io << b }  # Still copies internally
```

## แบบฝึกหัด

1. สร้าง HTTP request parser ที่ใช้ `StringView` แทน `String` เพื่อหลีกเลี่ยง allocations
2. Implement memory-mapped log file reader ที่ scan logs โดยไม่ copy ข้อมูล
3. สร้าง zero-copy serializer สำหรับ binary protocol (network packets)
4. เปรียบเทียบ performance ของ `IO.copy` vs manual buffer copy สำหรับ large files

## สรุป

Zero-copy Operations ใน Crystal:
- **Slice(T)**: view เข้าไปใน memory region โดยไม่ copy
- **Bytes**: Slice(UInt8) สำหรับ raw binary data
- **IO::Memory**: in-memory stream หลีกเลี่ยง string concatenation
- **to_slice**: get read-only view ของ string/array bytes
- **StringView**: custom struct สำหรับ string region without copy
- **IO.copy**: ใช้ sendfile syscall เมื่อ platform support
- **mmap**: memory-mapped files สำหรับ large file reading
- **writev/readv**: scatter-gather I/O ลด syscall count
