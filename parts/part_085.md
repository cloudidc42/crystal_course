# ตอนที่ 85: Binary Data ใน Crystal

## บทนำ

การทำงานกับข้อมูล binary เป็นทักษะสำคัญในการเขียนโปรแกรม systems programming Crystal มีความสามารถที่แข็งแกร่งสำหรับการจัดการ binary data ผ่าน `Bytes` (Slice(UInt8)), IO binary mode, endianness handling และ bit manipulation

---

## 1. Bytes - Slice(UInt8)

`Bytes` ใน Crystal คือ `Slice(UInt8)` หรือ array ของ unsigned 8-bit integers:

```crystal
# Bytes เป็น alias สำหรับ Slice(UInt8)
bytes = Bytes.new(5)  # 5 bytes เริ่มต้นด้วย 0
puts bytes.class   # => Slice(UInt8)
puts bytes.size    # => 5
puts bytes.inspect # => Bytes[0, 0, 0, 0, 0]
```

### การสร้าง Bytes:

```crystal
# จาก literal values
bytes1 = Bytes[0x48, 0x65, 0x6C, 0x6C, 0x6F]  # "Hello"
puts bytes1.inspect  # => Bytes[72, 101, 108, 108, 111]

# จาก String
str = "Hello"
bytes2 = str.to_slice
puts bytes2.inspect  # => Bytes[72, 101, 108, 108, 111]

# กลับเป็น String
puts String.new(bytes2)  # => Hello

# สร้างพร้อมขนาดและ initializer
bytes3 = Bytes.new(4) { |i| (i * 10).to_u8 }
puts bytes3.inspect  # => Bytes[0, 10, 20, 30]
```

---

## 2. การสร้างและจัดการ Byte Slices

```crystal
# สร้าง Bytes จาก array
bytes = Bytes.new(8)

# กำหนดค่า
bytes[0] = 0xFF_u8
bytes[1] = 0x0A_u8
bytes[2] = 0x1B_u8
bytes[3] = 0x2C_u8

puts bytes.inspect  # => Bytes[255, 10, 27, 44, 0, 0, 0, 0]

# copy
source = Bytes[1_u8, 2_u8, 3_u8, 4_u8]
dest = Bytes.new(4)
source.copy_to(dest)
puts dest.inspect  # => Bytes[1, 2, 3, 4]
```

### Slice operations:

```crystal
bytes = Bytes[0_u8, 1_u8, 2_u8, 3_u8, 4_u8, 5_u8, 6_u8, 7_u8]

# Slicing
first_four  = bytes[0, 4]   # Bytes[0, 1, 2, 3]
last_four   = bytes[4, 4]   # Bytes[4, 5, 6, 7]
middle      = bytes[2..5]   # Bytes[2, 3, 4, 5]

puts first_four.inspect
puts last_four.inspect
puts middle.inspect

# Iteration
bytes.each do |b|
  print "%02X " % b
end
puts ""  # => 00 01 02 03 04 05 06 07
```

---

## 3. การอ่านและเขียน Binary Files

### เขียนไฟล์ binary:

```crystal
# สร้าง binary data
data = Bytes[
  0x89_u8, 0x50_u8, 0x4E_u8, 0x47_u8,  # PNG magic bytes
  0x0D_u8, 0x0A_u8, 0x1A_u8, 0x0A_u8,  # More PNG header
]

# เขียนไฟล์ binary (จำลอง - ไม่ได้เขียนจริง)
# File.open("test.bin", "wb") do |file|
#   file.write(data)
# end

puts "Data to write:"
data.each { |b| print "%02X " % b }
puts ""
```

### อ่านไฟล์ binary:

```crystal
# อ่านไฟล์ binary (จำลอง)
# bytes = File.open("test.bin", "rb") do |file|
#   file.getb_to_end
# end

# จำลองการอ่าน
simulated_data = Bytes[
  0x89_u8, 0x50_u8, 0x4E_u8, 0x47_u8,
  0x0D_u8, 0x0A_u8, 0x1A_u8, 0x0A_u8,
]

# ตรวจสอบ magic bytes
is_png = simulated_data[0..3] == Bytes[0x89_u8, 0x50_u8, 0x4E_u8, 0x47_u8]
puts "Is PNG: #{is_png}"

# แสดง hex dump
simulated_data.each_with_index do |byte, i|
  print "%02X " % byte
  if (i + 1) % 8 == 0
    puts ""
  end
end
```

### IO binary mode:

```crystal
# เขียนด้วย IO::Memory ในโหมด binary
io = IO::Memory.new

# เขียน bytes โดยตรง
io.write Bytes[0x01_u8, 0x02_u8, 0x03_u8, 0x04_u8]
io.write_byte 0xFF_u8
io.write_byte 0x00_u8

# อ่านกลับ
io.rewind

puts "Read byte: #{io.read_byte.try { |b| "%02X" % b }}"
puts "Peek: #{io.peek.inspect}"

buffer = Bytes.new(4)
io.read_fully(buffer)
puts "Read 4 bytes: #{buffer.inspect}"
```

---

## 4. IO::ByteFormat - Big/Little Endian

Endianness กำหนดลำดับการจัดเก็บ bytes ของ multi-byte values:

```crystal
# Big Endian: byte ที่มีนัยสำคัญที่สุดอยู่ก่อน
# Little Endian: byte ที่มีนัยสำคัญน้อยที่สุดอยู่ก่อน

value = 0x12345678_i32

# Big Endian (Network byte order)
io_be = IO::Memory.new
value.to_io(io_be, IO::ByteFormat::BigEndian)
io_be.rewind
puts "Big Endian: #{io_be.to_slice.map { |b| "%02X" % b }.join(" ")}"
# => 12 34 56 78

# Little Endian (Most desktop CPUs)
io_le = IO::Memory.new
value.to_io(io_le, IO::ByteFormat::LittleEndian)
io_le.rewind
puts "Little Endian: #{io_le.to_slice.map { |b| "%02X" % b }.join(" ")}"
# => 78 56 34 12

# System Native
io_ne = IO::Memory.new
value.to_io(io_ne, IO::ByteFormat::SystemEndian)
io_ne.rewind
puts "System Endian: #{io_ne.to_slice.map { |b| "%02X" % b }.join(" ")}"
```

### อ่านกลับด้วย Endian ที่ถูกต้อง:

```crystal
# เขียน
io = IO::Memory.new
0x0001_i16.to_io(io, IO::ByteFormat::BigEndian)
0x0002_i16.to_io(io, IO::ByteFormat::BigEndian)
0x00000003_i32.to_io(io, IO::ByteFormat::BigEndian)
io.rewind

# อ่านกลับ
val1 = Int16.from_io(io, IO::ByteFormat::BigEndian)
val2 = Int16.from_io(io, IO::ByteFormat::BigEndian)
val3 = Int32.from_io(io, IO::ByteFormat::BigEndian)

puts "val1: #{val1}"  # => 1
puts "val2: #{val2}"  # => 2
puts "val3: #{val3}"  # => 3
```

---

## 5. การ Pack Integers เป็น Bytes

```crystal
# Pack Int32 เป็น bytes (Big Endian)
def pack_int32_be(value : Int32) : Bytes
  Bytes[
    ((value >> 24) & 0xFF).to_u8,
    ((value >> 16) & 0xFF).to_u8,
    ((value >> 8) & 0xFF).to_u8,
    (value & 0xFF).to_u8,
  ]
end

# Pack Int32 เป็น bytes (Little Endian)
def pack_int32_le(value : Int32) : Bytes
  Bytes[
    (value & 0xFF).to_u8,
    ((value >> 8) & 0xFF).to_u8,
    ((value >> 16) & 0xFF).to_u8,
    ((value >> 24) & 0xFF).to_u8,
  ]
end

value = 0x12345678
be = pack_int32_be(value)
le = pack_int32_le(value)

puts "Value: 0x#{value.to_s(16).upcase}"
puts "Big Endian:    #{be.map { |b| "%02X" % b }.join(" ")}"
puts "Little Endian: #{le.map { |b| "%02X" % b }.join(" ")}"
```

### ใช้ IO method:

```crystal
def int_to_bytes(value : Int32, format : IO::ByteFormat) : Bytes
  io = IO::Memory.new(4)
  value.to_io(io, format)
  io.to_slice
end

def int16_to_bytes(value : Int16, format : IO::ByteFormat) : Bytes
  io = IO::Memory.new(2)
  value.to_io(io, format)
  io.to_slice
end

puts int_to_bytes(300, IO::ByteFormat::BigEndian)
   .map { |b| "%02X" % b }.join(" ")   # => 00 00 01 2C

puts int_to_bytes(300, IO::ByteFormat::LittleEndian)
   .map { |b| "%02X" % b }.join(" ")   # => 2C 01 00 00
```

---

## 6. การ Unpack Bytes เป็น Integers

```crystal
# Unpack Big Endian bytes เป็น Int32
def unpack_int32_be(bytes : Bytes) : Int32
  raise "Need 4 bytes" unless bytes.size >= 4
  (bytes[0].to_i32 << 24) |
  (bytes[1].to_i32 << 16) |
  (bytes[2].to_i32 << 8)  |
   bytes[3].to_i32
end

# Unpack Little Endian bytes เป็น Int32
def unpack_int32_le(bytes : Bytes) : Int32
  raise "Need 4 bytes" unless bytes.size >= 4
   bytes[0].to_i32        |
  (bytes[1].to_i32 << 8)  |
  (bytes[2].to_i32 << 16) |
  (bytes[3].to_i32 << 24)
end

be_bytes = Bytes[0x12_u8, 0x34_u8, 0x56_u8, 0x78_u8]
le_bytes = Bytes[0x78_u8, 0x56_u8, 0x34_u8, 0x12_u8]

puts "BE: 0x#{unpack_int32_be(be_bytes).to_s(16).upcase}"  # => 0x12345678
puts "LE: 0x#{unpack_int32_le(le_bytes).to_s(16).upcase}"  # => 0x12345678
```

### ใช้ IO method:

```crystal
def bytes_to_int32(bytes : Bytes, format : IO::ByteFormat) : Int32
  io = IO::Memory.new(bytes)
  Int32.from_io(io, format)
end

def bytes_to_float64(bytes : Bytes, format : IO::ByteFormat) : Float64
  io = IO::Memory.new(bytes)
  Float64.from_io(io, format)
end

int_bytes = Bytes[0x00_u8, 0x00_u8, 0x01_u8, 0x2C_u8]
puts bytes_to_int32(int_bytes, IO::ByteFormat::BigEndian)  # => 300
```

---

## 7. Binary Protocol - Custom Protocol

ตัวอย่างการสร้าง binary protocol สำหรับ message:

```crystal
# Protocol format:
# [2 bytes: magic] [1 byte: version] [1 byte: type] [4 bytes: length] [N bytes: payload]
# Magic: 0xCA 0xFE
# Version: 0x01
# Type: 0x01=data, 0x02=ping, 0x03=pong

MAGIC_BYTES = Bytes[0xCA_u8, 0xFE_u8]

enum MessageType : UInt8
  Data = 0x01_u8
  Ping = 0x02_u8
  Pong = 0x03_u8
end

struct Message
  property type : MessageType
  property payload : Bytes

  def initialize(@type, @payload)
  end

  # Serialize เป็น binary
  def to_bytes : Bytes
    io = IO::Memory.new

    # Magic bytes
    io.write(MAGIC_BYTES)

    # Version
    io.write_byte(0x01_u8)

    # Type
    io.write_byte(@type.value)

    # Length (4 bytes, big endian)
    @payload.size.to_i32.to_io(io, IO::ByteFormat::BigEndian)

    # Payload
    io.write(@payload)

    io.to_slice
  end

  # Deserialize จาก binary
  def self.from_bytes(bytes : Bytes) : self?
    return nil if bytes.size < 8  # minimum header size

    io = IO::Memory.new(bytes)

    # Check magic
    magic = Bytes.new(2)
    io.read_fully(magic)
    return nil unless magic == MAGIC_BYTES

    # Check version
    version = io.read_byte
    return nil unless version == 0x01_u8

    # Read type
    type_byte = io.read_byte
    return nil unless type_byte
    msg_type = MessageType.new(type_byte)

    # Read length
    length = Int32.from_io(io, IO::ByteFormat::BigEndian)
    return nil if length < 0

    # Read payload
    payload = Bytes.new(length)
    io.read_fully(payload)

    new(msg_type, payload)
  end
end

# Test
original = Message.new(MessageType::Data, "Hello, Crystal!".to_slice)
serialized = original.to_bytes

puts "Serialized (#{serialized.size} bytes):"
serialized.each_with_index do |b, i|
  print "%02X " % b
  puts "" if (i + 1) % 8 == 0
end
puts ""

# Deserialize กลับ
restored = Message.from_bytes(serialized)
if restored
  puts "Type: #{restored.type}"
  puts "Payload: #{String.new(restored.payload)}"
else
  puts "Failed to parse message"
end
```

---

## 8. Bit Manipulation

```crystal
# AND (&) - ตรวจสอบและเคลียร์ bits
a = 0b10110110_u8
b = 0b11001100_u8

puts "AND: #{(a & b).to_s(2).rjust(8, '0')}"   # => 10000100
puts "OR:  #{(a | b).to_s(2).rjust(8, '0')}"   # => 11111110
puts "XOR: #{(a ^ b).to_s(2).rjust(8, '0')}"   # => 01111010
puts "NOT: #{(~a & 0xFF).to_s(2).rjust(8, '0')}" # => 01001001

# Shift
value = 0b00001111_u8
puts "Left shift 2:  #{(value << 2).to_s(2).rjust(8, '0')}"  # => 00111100
puts "Right shift 2: #{(value >> 2).to_s(2).rjust(8, '0')}"  # => 00000011
```

### Bit flags:

```crystal
# ใช้ bits เป็น flags
module Permission
  READ    = 0b00000001_u8  # bit 0
  WRITE   = 0b00000010_u8  # bit 1
  EXECUTE = 0b00000100_u8  # bit 2
  DELETE  = 0b00001000_u8  # bit 3
  ADMIN   = 0b10000000_u8  # bit 7
end

# กำหนด permissions
user_perms = Permission::READ | Permission::WRITE
admin_perms = Permission::READ | Permission::WRITE | Permission::DELETE | Permission::ADMIN

# ตรวจสอบ permission
def has_permission?(perms : UInt8, flag : UInt8) : Bool
  (perms & flag) != 0_u8
end

puts "User can read:    #{has_permission?(user_perms, Permission::READ)}"
puts "User can write:   #{has_permission?(user_perms, Permission::WRITE)}"
puts "User can execute: #{has_permission?(user_perms, Permission::EXECUTE)}"
puts "User can delete:  #{has_permission?(user_perms, Permission::DELETE)}"
puts "User is admin:    #{has_permission?(user_perms, Permission::ADMIN)}"

puts "Admin can delete: #{has_permission?(admin_perms, Permission::DELETE)}"

# เพิ่ม permission
user_perms |= Permission::EXECUTE
puts "\nAfter adding EXECUTE:"
puts "User can execute: #{has_permission?(user_perms, Permission::EXECUTE)}"

# ลบ permission
user_perms &= ~Permission::WRITE
puts "\nAfter removing WRITE:"
puts "User can write: #{has_permission?(user_perms, Permission::WRITE)}"
```

---

## 9. XOR Operations

XOR เป็น operation ที่มีประโยชน์มากในการทำงานกับ binary data:

```crystal
# XOR สมบัติ: a XOR b XOR b = a
# ใช้สำหรับ simple encryption
def xor_cipher(data : Bytes, key : UInt8) : Bytes
  result = Bytes.new(data.size)
  data.each_with_index { |byte, i| result[i] = (byte ^ key).to_u8 }
  result
end

message = "สวัสดี Crystal!".encode("UTF-8")
key = 0x42_u8

encrypted = xor_cipher(message, key)
decrypted = xor_cipher(encrypted, key)  # XOR สองครั้ง = ค่าเดิม

puts "Original:  #{String.new(message)}"
puts "Encrypted: #{encrypted.map { |b| "%02X" % b }.join(" ")}"
puts "Decrypted: #{String.new(decrypted)}"
puts "Match: #{message == decrypted}"
```

### XOR กับ key หลาย bytes:

```crystal
def xor_encrypt(data : Bytes, key : Bytes) : Bytes
  result = Bytes.new(data.size)
  data.each_with_index do |byte, i|
    result[i] = (byte ^ key[i % key.size]).to_u8
  end
  result
end

data = "Hello, Binary World!".to_slice
key  = Bytes[0x1A_u8, 0x2B_u8, 0x3C_u8, 0x4D_u8]

encrypted = xor_encrypt(data, key)
decrypted = xor_encrypt(encrypted, key)

puts "Original:  #{String.new(data)}"
puts "Decrypted: #{String.new(decrypted)}"
puts "Round-trip OK: #{String.new(data) == String.new(decrypted)}"
```

---

## 10. Hex Dump Utility

```crystal
def hex_dump(bytes : Bytes, address_offset : Int32 = 0, width : Int32 = 16)
  bytes.each_slice(width).with_index do |row, row_idx|
    # Address column
    address = address_offset + row_idx * width
    print "%08X  " % address

    # Hex values
    row.each_with_index do |byte, i|
      print "%02X " % byte
      print " " if i == 7  # extra space in middle
    end

    # Padding if last row is incomplete
    if row.size < width
      remaining = width - row.size
      print "   " * remaining
      print " " if row.size <= 8
    end

    # ASCII representation
    print " |"
    row.each do |byte|
      char = byte >= 0x20_u8 && byte <= 0x7E_u8 ? byte.chr : '.'
      print char
    end
    puts "|"
  end
end

# ทดสอบ hex dump
test_data = "Hello, World! Crystal Binary Data 1234567890!@#$%".to_slice
puts "Hex dump of test data:"
hex_dump(test_data)
```

### ขั้นสูงกว่า:

```crystal
def hex_dump_string(bytes : Bytes, width : Int32 = 16) : String
  String.build do |str|
    bytes.each_slice(width).with_index do |row, row_idx|
      # offset
      str.printf("%08x  ", row_idx * width)

      # hex bytes
      row.each_with_index do |byte, i|
        str.printf("%02x ", byte)
        str << " " if i == 7
      end

      # padding
      if row.size < width
        remaining = width - row.size
        str << "   " * remaining
        str << " " if row.size <= 8
      end

      # ascii
      str << " |"
      row.each do |byte|
        str << (byte >= 0x20_u8 && byte <= 0x7E_u8 ? byte.chr : '.')
      end
      str << "|\n"
    end
  end
end

data = Bytes[
  0x43_u8, 0x72_u8, 0x79_u8, 0x73_u8, 0x74_u8, 0x61_u8, 0x6C_u8, 0x20_u8,
  0x42_u8, 0x69_u8, 0x6E_u8, 0x61_u8, 0x72_u8, 0x79_u8, 0x00_u8, 0xFF_u8,
]

puts hex_dump_string(data)
```

---

## 11. Binary File Formats - PNG Header

```crystal
# ตรวจสอบ file format จาก magic bytes
module FileSignature
  PNG  = Bytes[0x89_u8, 0x50_u8, 0x4E_u8, 0x47_u8, 0x0D_u8, 0x0A_u8, 0x1A_u8, 0x0A_u8]
  JPEG = Bytes[0xFF_u8, 0xD8_u8, 0xFF_u8]
  PDF  = Bytes[0x25_u8, 0x50_u8, 0x44_u8, 0x46_u8]  # %PDF
  ZIP  = Bytes[0x50_u8, 0x4B_u8, 0x03_u8, 0x04_u8]  # PK..
  GIF  = Bytes[0x47_u8, 0x49_u8, 0x46_u8, 0x38_u8]  # GIF8
end

def detect_file_type(bytes : Bytes) : String
  return "PNG"  if bytes.size >= 8 && bytes[0, 8] == FileSignature::PNG
  return "JPEG" if bytes.size >= 3 && bytes[0, 3] == FileSignature::JPEG
  return "PDF"  if bytes.size >= 4 && bytes[0, 4] == FileSignature::PDF
  return "ZIP"  if bytes.size >= 4 && bytes[0, 4] == FileSignature::ZIP
  return "GIF"  if bytes.size >= 4 && bytes[0, 4] == FileSignature::GIF
  "Unknown"
end

# ทดสอบ
test_files = [
  {name: "test.png",  data: FileSignature::PNG},
  {name: "test.jpg",  data: FileSignature::JPEG + Bytes[0x00_u8, 0x00_u8]},
  {name: "test.pdf",  data: FileSignature::PDF + Bytes[0x2D_u8, 0x31_u8]},
  {name: "unknown",   data: Bytes[0x00_u8, 0x00_u8, 0x00_u8, 0x00_u8]},
]

test_files.each do |file|
  type = detect_file_type(file[:data])
  puts "#{file[:name]}: #{type}"
end
```

---

## 12. Checksum และ CRC

```crystal
# Simple checksum (sum of bytes mod 256)
def simple_checksum(bytes : Bytes) : UInt8
  bytes.reduce(0_u32) { |sum, b| sum + b }.to_u8
end

# XOR checksum
def xor_checksum(bytes : Bytes) : UInt8
  bytes.reduce(0_u8) { |xor, b| (xor ^ b).to_u8 }
end

data = "Hello, Crystal!".to_slice

sum_checksum = simple_checksum(data)
xor_cs = xor_checksum(data)

puts "Data: #{String.new(data)}"
puts "Sum checksum: 0x#{sum_checksum.to_s(16).upcase}"
puts "XOR checksum: 0x#{xor_cs.to_s(16).upcase}"

# ตรวจสอบความสมบูรณ์
corrupted_data = data.dup
corrupted_data[0] = (corrupted_data[0] ^ 0x01_u8).to_u8  # เปลี่ยน 1 bit

puts "\nCorrupted data check:"
puts "Original checksum:  0x#{sum_checksum.to_s(16).upcase}"
puts "Corrupted checksum: 0x#{simple_checksum(corrupted_data).to_s(16).upcase}"
puts "Match: #{sum_checksum == simple_checksum(corrupted_data)}"
```

---

## 13. Binary Struct Packing

```crystal
# Struct packing - บีบข้อมูล struct เป็น binary
# Header format: version(1) + flags(1) + timestamp(4) + length(2)

struct PacketHeader
  property version : UInt8
  property flags : UInt8
  property timestamp : UInt32
  property length : UInt16

  HEADER_SIZE = 8  # 1 + 1 + 4 + 2

  def initialize(@version, @flags, @timestamp, @length)
  end

  def to_bytes : Bytes
    io = IO::Memory.new(HEADER_SIZE)
    io.write_byte(@version)
    io.write_byte(@flags)
    @timestamp.to_io(io, IO::ByteFormat::BigEndian)
    @length.to_io(io, IO::ByteFormat::BigEndian)
    io.to_slice
  end

  def self.from_bytes(bytes : Bytes) : self?
    return nil if bytes.size < HEADER_SIZE

    io = IO::Memory.new(bytes)
    version   = io.read_byte || return nil
    flags     = io.read_byte || return nil
    timestamp = UInt32.from_io(io, IO::ByteFormat::BigEndian)
    length    = UInt16.from_io(io, IO::ByteFormat::BigEndian)

    new(version, flags, timestamp, length)
  end
end

# Create and serialize header
header = PacketHeader.new(
  version:   0x01_u8,
  flags:     0b00001100_u8,
  timestamp: 1705305600_u32,  # 2024-01-15 00:00:00 UTC
  length:    42_u16
)

bytes = header.to_bytes
puts "Header bytes:"
bytes.each { |b| print "%02X " % b }
puts ""
puts "Size: #{bytes.size} bytes"

# Deserialize
restored = PacketHeader.from_bytes(bytes)
if restored
  puts "\nRestored header:"
  puts "  Version:   #{restored.version}"
  puts "  Flags:     #{restored.flags.to_s(2).rjust(8, '0')}"
  puts "  Timestamp: #{restored.timestamp}"
  puts "  Length:    #{restored.length}"
end
```

---

## 14. Memory Mapped Files (จำลอง)

```crystal
# จำลองการทำงานกับ large binary files แบบ chunked reading
def process_binary_chunks(data : Bytes, chunk_size : Int32 = 1024)
  total_bytes   = data.size
  chunks        = (total_bytes / chunk_size.to_f).ceil.to_i
  processed     = 0
  null_count    = 0
  nonzero_count = 0

  data.each_slice(chunk_size) do |chunk|
    # Process each chunk
    chunk.each do |byte|
      if byte == 0
        null_count += 1
      else
        nonzero_count += 1
      end
    end
    processed += chunk.size
  end

  {
    total:    total_bytes,
    chunks:   chunks,
    nulls:    null_count,
    nonzero:  nonzero_count,
  }
end

# สร้าง test data
test_data = Bytes.new(10000) { |i| (i % 256).to_u8 }
# เพิ่ม null bytes
500.times { |i| test_data[i * 2] = 0_u8 }

stats = process_binary_chunks(test_data)
puts "Total bytes: #{stats[:total]}"
puts "Null bytes:  #{stats[:nulls]}"
puts "Non-zero:    #{stats[:nonzero]}"
```

---

## 15. Protocol Buffer Basics (Manual)

```crystal
# Protocol Buffers ใช้ encoding เฉพาะทาง
# นี่เป็นการ implement แบบง่าย เพื่อเรียนรู้ concept

# Varint encoding (variable-length integer)
def encode_varint(value : Int32) : Bytes
  result = [] of UInt8
  val = value.to_u32

  loop do
    byte = (val & 0x7F).to_u8
    val >>= 7
    if val == 0
      result << byte
      break
    else
      result << (byte | 0x80).to_u8
    end
  end

  Bytes.new(result.size) { |i| result[i] }
end

def decode_varint(bytes : Bytes) : {Int32, Int32}
  result = 0_u32
  shift = 0

  bytes.each_with_index do |byte, i|
    result |= ((byte & 0x7F).to_u32) << shift
    shift += 7

    unless (byte & 0x80) != 0
      return {result.to_i32, i + 1}
    end
  end

  {result.to_i32, bytes.size}
end

# Test varint
[0, 1, 127, 128, 300, 16384].each do |num|
  encoded = encode_varint(num)
  decoded, bytes_used = decode_varint(encoded)

  encoded_hex = encoded.map { |b| "%02X" % b }.join(" ")
  puts "#{num.to_s.rjust(6)} => [#{encoded_hex}] => #{decoded} (#{bytes_used} bytes)"
end
```

---

## 16. ตัวอย่างโปรแกรมสมบูรณ์ - Binary Log File

```crystal
# Binary log file format
# Per entry: [4: timestamp][1: level][2: message_length][N: message]

enum LogLevel : UInt8
  Debug = 0
  Info  = 1
  Warn  = 2
  Error = 3
  Fatal = 4
end

struct LogEntry
  property timestamp : UInt32
  property level : LogLevel
  property message : String

  def initialize(@timestamp, @level, @message)
  end

  HEADER_SIZE = 7  # 4 + 1 + 2

  def to_bytes : Bytes
    msg_bytes = @message.to_slice
    total_size = HEADER_SIZE + msg_bytes.size

    io = IO::Memory.new(total_size)
    @timestamp.to_io(io, IO::ByteFormat::BigEndian)
    io.write_byte(@level.value)
    msg_bytes.size.to_u16.to_io(io, IO::ByteFormat::BigEndian)
    io.write(msg_bytes)
    io.to_slice
  end

  def self.from_io(io : IO) : self?
    begin
      timestamp = UInt32.from_io(io, IO::ByteFormat::BigEndian)
      level_byte = io.read_byte || return nil
      level = LogLevel.new(level_byte)
      msg_len = UInt16.from_io(io, IO::ByteFormat::BigEndian)
      msg_bytes = Bytes.new(msg_len)
      io.read_fully(msg_bytes)
      new(timestamp, level, String.new(msg_bytes))
    rescue IO::Error
      nil
    end
  end
end

class BinaryLogger
  @entries = [] of LogEntry

  def log(level : LogLevel, message : String)
    @entries << LogEntry.new(Time.local.to_unix.to_u32, level, message)
  end

  def debug(msg) = log(LogLevel::Debug, msg)
  def info(msg)  = log(LogLevel::Info, msg)
  def warn(msg)  = log(LogLevel::Warn, msg)
  def error(msg) = log(LogLevel::Error, msg)

  def to_bytes : Bytes
    io = IO::Memory.new
    # File header: magic(4) + version(1) + count(4)
    io.write(Bytes[0x42_u8, 0x4C_u8, 0x4F_u8, 0x47_u8])  # BLOG
    io.write_byte(0x01_u8)
    @entries.size.to_u32.to_io(io, IO::ByteFormat::BigEndian)

    # Entries
    @entries.each { |entry| io.write(entry.to_bytes) }
    io.to_slice
  end

  def self.from_bytes(bytes : Bytes) : Array(LogEntry)
    entries = [] of LogEntry
    io = IO::Memory.new(bytes)

    # Read header
    magic = Bytes.new(4)
    io.read_fully(magic)
    return entries unless String.new(magic) == "BLOG"

    version = io.read_byte
    return entries unless version == 0x01_u8

    count = UInt32.from_io(io, IO::ByteFormat::BigEndian)

    count.times do
      entry = LogEntry.from_io(io)
      entries << entry if entry
    end

    entries
  end
end

# Demo
logger = BinaryLogger.new
logger.debug("Application starting")
logger.info("Server listening on port 8080")
logger.info("Database connected")
logger.warn("Memory usage at 75%")
logger.error("Failed to process request: timeout")
logger.info("Graceful shutdown initiated")

# Serialize
binary_log = logger.to_bytes
puts "Log file size: #{binary_log.size} bytes"

# Show hex dump of first 64 bytes
puts "\nHex dump (first 64 bytes):"
hex_part = binary_log[0, [64, binary_log.size].min]
hex_part.each_with_index do |b, i|
  print "%02X " % b
  puts "" if (i + 1) % 16 == 0
end
puts ""

# Deserialize and display
entries = BinaryLogger.from_bytes(binary_log)
puts "\n=== Log entries (#{entries.size} total) ==="
entries.each do |entry|
  time = Time.unix(entry.timestamp)
  level_name = entry.level.to_s.upcase.rjust(5)
  puts "[#{level_name}] #{entry.message}"
end
```

---

## 17. Bit Counting และ Operations

```crystal
# นับจำนวน bits ที่เป็น 1 (Hamming Weight / popcount)
def popcount(n : UInt32) : Int32
  count = 0
  while n > 0
    count += 1 if (n & 1) == 1
    n >>= 1
  end
  count
end

# หาตำแหน่งของ bit สูงสุดที่เป็น 1
def highest_bit(n : UInt32) : Int32
  return -1 if n == 0
  pos = 0
  temp = n
  while temp > 1
    temp >>= 1
    pos += 1
  end
  pos
end

# ตรวจสอบว่าเป็น power of 2
def power_of_two?(n : UInt32) : Bool
  n > 0 && (n & (n - 1)) == 0
end

# Reverse bits ใน byte
def reverse_bits(byte : UInt8) : UInt8
  result = 0_u8
  8.times do |i|
    result = (result | (((byte >> i) & 1) << (7 - i))).to_u8
  end
  result
end

# Tests
[0_u32, 1_u32, 5_u32, 15_u32, 255_u32, 256_u32].each do |n|
  puts "#{n.to_s.rjust(5)}: popcount=#{popcount(n)}, highest_bit=#{highest_bit(n)}, power_of_2=#{power_of_two?(n)}"
end

puts ""
[0b10110100_u8, 0b00001111_u8, 0xFF_u8].each do |b|
  rev = reverse_bits(b)
  puts "Reverse #{b.to_s(2).rjust(8, '0')} => #{rev.to_s(2).rjust(8, '0')}"
end
```

---

## 18. Binary Data กับ IO Pipes

```crystal
# ใช้ IO::Memory เป็น pipe สำหรับ binary data processing
def process_binary_stream(input : IO, output : IO)
  buffer = Bytes.new(1024)

  loop do
    bytes_read = input.read(buffer)
    break if bytes_read == 0

    # Process: XOR ทุก byte ด้วย 0x20 (toggle case สำหรับ ASCII letters)
    chunk = buffer[0, bytes_read]
    processed = Bytes.new(bytes_read) { |i|
      b = chunk[i]
      if (b >= 0x41_u8 && b <= 0x5A_u8) || (b >= 0x61_u8 && b <= 0x7A_u8)
        (b ^ 0x20_u8).to_u8
      else
        b
      end
    }

    output.write(processed)
  end
end

input_data = "Hello, World! 123 ABC xyz"
input  = IO::Memory.new(input_data.to_slice)
output = IO::Memory.new

process_binary_stream(input, output)

puts "Input:  #{input_data}"
puts "Output: #{String.new(output.to_slice)}"
# => Output: hELLO, wORLD! 123 abc XYZ
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Binary File Header
สร้าง struct `FileHeader` ที่ serialize/deserialize ตาม format:
- Magic: 4 bytes ("CRYS")
- Version: 2 bytes (big endian UInt16)
- Created: 4 bytes (unix timestamp, big endian UInt32)
- Checksum: 1 byte (XOR of all previous bytes)

```crystal
struct FileHeader
  MAGIC = "CRYS"
  property version : UInt16
  property created : UInt32

  # TODO: implement to_bytes และ from_bytes
  # TODO: verify checksum
end
```

### แบบฝึกหัดที่ 2: Hex Dump Formatter
ปรับปรุง hex dump utility ให้รองรับ:
1. แสดง address ในรูปแบบที่กำหนดได้ (hex หรือ decimal)
2. ปรับ width ของ display ได้
3. Highlight bytes ที่เป็น 0x00 และ 0xFF

### แบบฝึกหัดที่ 3: Simple Binary Serializer
สร้าง generic binary serializer/deserializer สำหรับ struct ที่มี fields:
- String (นำหน้าด้วย length UInt16)
- Int32 (big endian)
- Float64 (big endian)
- Bool (1 byte: 0 หรือ 1)

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

1. **`Bytes` (Slice(UInt8))** - การสร้างและจัดการ byte slices
2. **Binary file I/O** - การอ่านและเขียนไฟล์ในโหมด binary
3. **`IO::ByteFormat`** - Big Endian, Little Endian และ System Endian
4. **Pack integers** - การแปลง integers เป็น byte arrays
5. **Unpack bytes** - การแปลง byte arrays กลับเป็น integers
6. **Binary protocols** - การออกแบบและ implement binary message formats
7. **Bit manipulation** - AND, OR, XOR, NOT, shift operations
8. **Bit flags** - การใช้ bits เป็น boolean flags
9. **XOR operations** - การใช้ XOR สำหรับ encryption และ checksum
10. **Hex dump** - utility สำหรับแสดง binary data
11. **File signatures** - การตรวจสอบประเภทไฟล์จาก magic bytes
12. **Checksum/CRC** - การตรวจสอบความสมบูรณ์ของข้อมูล
13. **Struct packing** - การ serialize structs เป็น binary
14. **Protocol Buffers basics** - Varint encoding concept
15. **Bit counting** - popcount, highest bit, power of 2

Binary data manipulation เป็นพื้นฐานสำคัญของ systems programming ที่ Crystal ทำได้อย่างมีประสิทธิภาพด้วย type safety ที่แข็งแกร่ง
