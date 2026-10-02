# Part 85: Binary Data

## บทนำ

Binary data manipulation เป็นทักษะสำคัญสำหรับ systems programming, network protocols, file formats, และ cryptography Crystal มี `Bytes` (Slice(UInt8)) สำหรับจัดการ binary data

---

## 1. Bytes (Slice(UInt8))

```crystal
# สร้าง Bytes
bytes = Bytes[0x48, 0x65, 0x6C, 0x6C, 0x6F]
puts bytes.inspect        # => Bytes[72, 101, 108, 108, 111]
puts String.new(bytes)    # => Hello

# จาก String
str = "Hello, World!"
bytes = str.to_slice
puts bytes.class          # => Bytes
puts bytes.size           # => 13
puts bytes[0]             # => 72 (H)

# จาก Array
arr = [1_u8, 2_u8, 3_u8, 255_u8]
bytes = Bytes.new(arr.size) { |i| arr[i] }
puts bytes.inspect        # => Bytes[1, 2, 3, 255]

# Bytes.new กับ size
zeros = Bytes.new(10)     # 10 zero bytes
puts zeros.inspect        # => Bytes[0, 0, 0, 0, 0, 0, 0, 0, 0, 0]

# Bytes.new กับ initial value
ones = Bytes.new(5, 0xFF_u8)
puts ones.inspect         # => Bytes[255, 255, 255, 255, 255]

# Slice operations
bytes = Bytes[1, 2, 3, 4, 5, 6, 7, 8]
puts bytes[2, 4].inspect  # => Bytes[3, 4, 5, 6]  (offset, count)
puts bytes[2..5].inspect  # => Bytes[3, 4, 5, 6]
puts bytes.first(3).inspect  # => Bytes[1, 2, 3]
puts bytes.last(3).inspect   # => Bytes[6, 7, 8]
```

---

## 2. Reading Binary Data

```crystal
# เขียนและอ่าน binary data
binary_data = Bytes[
  0x89, 0x50, 0x4E, 0x47,  # PNG magic bytes
  0x0D, 0x0A, 0x1A, 0x0A   # more PNG header
]

# ตรวจสอบ magic bytes
def is_png?(data : Bytes) : Bool
  data.size >= 8 &&
  data[0] == 0x89 &&
  data[1] == 0x50 &&  # P
  data[2] == 0x4E &&  # N
  data[3] == 0x47     # G
end

puts is_png?(binary_data)  # => true

# อ่าน binary file
def read_binary_file(path : String) : Bytes
  File.open(path, "rb") do |f|
    buf = Bytes.new(f.size.to_i)
    f.read_fully(buf)
    buf
  end
end

# ตรวจสอบ file type จาก magic bytes
def detect_file_type(data : Bytes) : String
  return "unknown" if data.size < 4
  
  case
  when data[0..1] == Bytes[0xFF, 0xD8]              then "JPEG"
  when data[0..3] == Bytes[0x89, 0x50, 0x4E, 0x47]  then "PNG"
  when data[0..3] == Bytes[0x47, 0x49, 0x46, 0x38]  then "GIF"
  when data[0..1] == Bytes[0x42, 0x4D]              then "BMP"
  when data[0..3] == Bytes[0x25, 0x50, 0x44, 0x46]  then "PDF"
  when data[0..1] == Bytes[0x1F, 0x8B]              then "GZIP"
  when data[0..3] == Bytes[0x50, 0x4B, 0x03, 0x04]  then "ZIP"
  else "unknown"
  end
end

# ตัวอย่าง magic bytes
puts detect_file_type(Bytes[0xFF, 0xD8, 0xFF, 0xE0])  # => JPEG
puts detect_file_type(Bytes[0x89, 0x50, 0x4E, 0x47])  # => PNG
puts detect_file_type(Bytes[0x25, 0x50, 0x44, 0x46])  # => PDF
```

---

## 3. IO::ByteFormat - Multi-byte Integers

```crystal
# เขียน/อ่าน multi-byte integers

# เขียน Int32 เป็น big-endian bytes
io = IO::Memory.new
io.write_bytes(0x12345678_u32, IO::ByteFormat::BigEndian)
io.rewind
bytes = io.to_slice
puts bytes.inspect  # => Bytes[18, 52, 86, 120]  (0x12, 0x34, 0x56, 0x78)

# เขียน Int32 เป็น little-endian bytes
io = IO::Memory.new
io.write_bytes(0x12345678_u32, IO::ByteFormat::LittleEndian)
io.rewind
bytes = io.to_slice
puts bytes.inspect  # => Bytes[120, 86, 52, 18]  (0x78, 0x56, 0x34, 0x12)

# อ่าน multi-byte integers
data = Bytes[0x00, 0x00, 0x03, 0xE8]  # big-endian 1000
io = IO::Memory.new(data)
value = io.read_bytes(UInt32, IO::ByteFormat::BigEndian)
puts value  # => 1000

data = Bytes[0xE8, 0x03, 0x00, 0x00]  # little-endian 1000
io = IO::Memory.new(data)
value = io.read_bytes(UInt32, IO::ByteFormat::LittleEndian)
puts value  # => 1000

# Network byte order = Big Endian
alias NetworkByteOrder = IO::ByteFormat::BigEndian

def encode_packet(command : UInt16, payload : Bytes) : Bytes
  buf = IO::Memory.new
  buf.write_bytes(command, NetworkByteOrder)
  buf.write_bytes(payload.size.to_u32, NetworkByteOrder)
  buf.write(payload)
  buf.to_slice
end

def decode_packet(data : Bytes) : {UInt16, Bytes}
  io = IO::Memory.new(data)
  command = io.read_bytes(UInt16, NetworkByteOrder)
  length = io.read_bytes(UInt32, NetworkByteOrder).to_i
  payload = Bytes.new(length)
  io.read_fully(payload)
  {command, payload}
end

payload = "Hello".to_slice
packet = encode_packet(0x0001_u16, payload)
puts packet.inspect

cmd, data = decode_packet(packet)
puts "Command: #{cmd}, Data: #{String.new(data)}"
```

---

## 4. Bit Manipulation

```crystal
# Bit operations บน integers
n = 0b1010_1100_u8
puts n.to_s(2).rjust(8, '0')   # => 10101100

# AND, OR, XOR, NOT
a = 0b1100_u8
b = 0b1010_u8

puts (a & b).to_s(2)   # => 1000  (AND)
puts (a | b).to_s(2)   # => 1110  (OR)
puts (a ^ b).to_s(2)   # => 110   (XOR)
puts (~a).to_s(2)       # => -13   (NOT, สำหรับ signed)

# Bit shifting
puts (1_u8 << 3).to_s(2)   # => 1000  (left shift 3)
puts (0b1000_u8 >> 2).to_s(2)  # => 10  (right shift 2)

# เช็ค specific bits
flags = 0b0101_u8

def bit_set?(value : UInt8, bit : Int32) : Bool
  (value & (1_u8 << bit)) != 0
end

puts bit_set?(flags, 0)  # => true  (bit 0 = 1)
puts bit_set?(flags, 1)  # => false (bit 1 = 0)
puts bit_set?(flags, 2)  # => true  (bit 2 = 1)

# Set/clear/toggle bits
def set_bit(value : UInt8, bit : Int32) : UInt8
  value | (1_u8 << bit)
end

def clear_bit(value : UInt8, bit : Int32) : UInt8
  value & ~(1_u8 << bit)
end

def toggle_bit(value : UInt8, bit : Int32) : UInt8
  value ^ (1_u8 << bit)
end

n = 0b0000_u8
n = set_bit(n, 0)   # => 00000001
n = set_bit(n, 3)   # => 00001001
n = toggle_bit(n, 0) # => 00001000
n = clear_bit(n, 3)  # => 00000000
puts n.to_s(2)       # => 0

# Flags bitmask
PERMISSION_READ    = 0b001_u8
PERMISSION_WRITE   = 0b010_u8
PERMISSION_EXECUTE = 0b100_u8

user_perms = PERMISSION_READ | PERMISSION_WRITE
puts "Can read: #{(user_perms & PERMISSION_READ) != 0}"
puts "Can write: #{(user_perms & PERMISSION_WRITE) != 0}"
puts "Can execute: #{(user_perms & PERMISSION_EXECUTE) != 0}"
```

---

## 5. Protocol Implementation

```crystal
# ตัวอย่าง simple binary protocol
# Message format:
# [1 byte: message type]
# [4 bytes: payload length (big-endian)]
# [N bytes: payload]

enum MessageType : UInt8
  Ping     = 0x01
  Pong     = 0x02
  Data     = 0x03
  Error    = 0xFF
end

struct Message
  getter type : MessageType
  getter payload : Bytes
  
  def initialize(@type, @payload = Bytes.empty)
  end
  
  def encode : Bytes
    buf = IO::Memory.new
    buf.write_byte(@type.value)
    buf.write_bytes(@payload.size.to_u32, IO::ByteFormat::BigEndian)
    buf.write(@payload)
    buf.to_slice
  end
  
  def self.decode(data : Bytes) : Message
    io = IO::Memory.new(data)
    
    type_byte = io.read_byte || raise "Unexpected end of data"
    type = MessageType.new(type_byte)
    
    length = io.read_bytes(UInt32, IO::ByteFormat::BigEndian).to_i
    
    payload = Bytes.new(length)
    io.read_fully(payload)
    
    new(type, payload)
  end
  
  def to_s : String
    "Message(#{type}, #{payload.size} bytes)"
  end
end

# ใช้งาน
ping = Message.new(MessageType::Ping)
encoded = ping.encode
puts "Encoded: #{encoded.inspect}"

decoded = Message.decode(encoded)
puts decoded

data_msg = Message.new(MessageType::Data, "Hello, Binary World!".to_slice)
encoded2 = data_msg.encode
decoded2 = Message.decode(encoded2)
puts "#{decoded2.type}: '#{String.new(decoded2.payload)}'"
```

---

## 6. Hex Encoding/Decoding

```crystal
# แปลง Bytes เป็น Hex string
def bytes_to_hex(bytes : Bytes) : String
  bytes.map { |b| b.to_s(16).rjust(2, '0') }.join
end

def bytes_to_hex_pretty(bytes : Bytes, group : Int32 = 16) : String
  result = String.build do |s|
    bytes.each_slice(group).with_index do |row, i|
      # Hex offset
      s << "%08X  " % (i * group)
      
      # Hex bytes
      row.each { |b| s << "%02X " % b }
      (group - row.size).times { s << "   " }  # padding
      
      # ASCII representation
      s << " |"
      row.each { |b| s << (b >= 32 && b < 127 ? b.chr.to_s : ".") }
      s << "|\n"
    end
  end
  result
end

# แปลง Hex string เป็น Bytes
def hex_to_bytes(hex : String) : Bytes
  # ลบ spaces และ colons
  clean = hex.gsub(/[\s:]/, "")
  raise ArgumentError.new("Invalid hex string length") if clean.size.odd?
  
  size = clean.size // 2
  Bytes.new(size) do |i|
    clean[i * 2, 2].to_u8(16)
  end
end

# ตัวอย่าง
data = "Hello, World!".to_slice
hex = bytes_to_hex(data)
puts "Hex: #{hex}"
puts "Pretty:\n#{bytes_to_hex_pretty(data)}"

decoded = hex_to_bytes(hex)
puts "Decoded: #{String.new(decoded)}"

# SHA1/MD5 style output
hash_bytes = Bytes[0xDA, 0x39, 0xA3, 0xEE, 0x5E, 0x6B, 0x4B, 0x0D]
puts bytes_to_hex(hash_bytes)  # => da39a3ee5e6b4b0d

# Colon-separated
def bytes_to_mac(bytes : Bytes) : String
  bytes.map { |b| b.to_s(16).rjust(2, '0') }.join(":")
end

mac = Bytes[0x00, 0x1A, 0x2B, 0x3C, 0x4D, 0x5E]
puts bytes_to_mac(mac)  # => 00:1a:2b:3c:4d:5e
```

---

## 7. Base64 Encoding

```crystal
require "base64"

# encode
data = "Hello, World!"
encoded = Base64.encode(data)
puts encoded         # => SGVsbG8sIFdvcmxkIQ==\n

encoded_strict = Base64.strict_encode(data)
puts encoded_strict  # => SGVsbG8sIFdvcmxkIQ==

encoded_url = Base64.urlsafe_encode(data)
puts encoded_url     # => SGVsbG8sIFdvcmxkIQ== (URL-safe chars)

# decode
decoded = Base64.decode_string(encoded_strict)
puts decoded  # => Hello, World!

# Binary data
binary = Bytes[0xFF, 0xFE, 0xFD, 0x00, 0x01, 0x02]
b64 = Base64.strict_encode(binary)
puts b64  # => //79AAEC

decoded_binary = Base64.decode(b64)
puts decoded_binary.inspect  # => Bytes[255, 254, 253, 0, 1, 2]

# Data URL
def to_data_url(data : Bytes, mime_type : String) : String
  "data:#{mime_type};base64,#{Base64.strict_encode(data)}"
end

# Image data URL
small_png = Bytes[0x89, 0x50, 0x4E, 0x47]  # PNG magic bytes only (invalid PNG, just example)
puts to_data_url(small_png, "image/png")[0..50]

# JWT-style base64url
def base64url_encode(data : String | Bytes) : String
  Base64.urlsafe_encode(data).rstrip('=')
end

def base64url_decode(str : String) : Bytes
  # เพิ่ม padding
  padded = str + "=" * ((4 - str.size % 4) % 4)
  Base64.decode(padded)
end
```

---

## 8. Binary File Format

```crystal
# อ่าน/เขียน custom binary format

# Simple binary format:
# Header: magic[4] + version[2] + flags[2] + num_records[4]
# Records: key_len[2] + key[n] + value_len[4] + value[m]

MAGIC_BYTES = Bytes[0x42, 0x44, 0x46, 0x00]  # "BDF\0"

struct BinaryDB
  getter records : Hash(String, Bytes)
  
  def initialize
    @records = {} of String => Bytes
  end
  
  def self.new(&) : BinaryDB
    db = new
    yield db
    db
  end
  
  def set(key : String, value : String | Bytes)
    @records[key] = case value
    when String then value.to_slice
    when Bytes  then value
    else value.to_slice
    end
  end
  
  def get(key : String) : Bytes?
    @records[key]?
  end
  
  def get_string(key : String) : String?
    @records[key]?.try { |b| String.new(b) }
  end
  
  def serialize : Bytes
    io = IO::Memory.new
    
    # Header
    io.write(MAGIC_BYTES)
    io.write_bytes(1_u16, IO::ByteFormat::BigEndian)   # version
    io.write_bytes(0_u16, IO::ByteFormat::BigEndian)   # flags
    io.write_bytes(@records.size.to_u32, IO::ByteFormat::BigEndian)
    
    # Records
    @records.each do |key, value|
      key_bytes = key.to_slice
      io.write_bytes(key_bytes.size.to_u16, IO::ByteFormat::BigEndian)
      io.write(key_bytes)
      io.write_bytes(value.size.to_u32, IO::ByteFormat::BigEndian)
      io.write(value)
    end
    
    io.to_slice
  end
  
  def self.deserialize(data : Bytes) : BinaryDB
    io = IO::Memory.new(data)
    
    # Verify magic
    magic = Bytes.new(4)
    io.read_fully(magic)
    raise "Invalid file format" unless magic == MAGIC_BYTES
    
    version = io.read_bytes(UInt16, IO::ByteFormat::BigEndian)
    _flags = io.read_bytes(UInt16, IO::ByteFormat::BigEndian)
    num_records = io.read_bytes(UInt32, IO::ByteFormat::BigEndian).to_i
    
    db = new
    num_records.times do
      key_len = io.read_bytes(UInt16, IO::ByteFormat::BigEndian).to_i
      key_bytes = Bytes.new(key_len)
      io.read_fully(key_bytes)
      key = String.new(key_bytes)
      
      value_len = io.read_bytes(UInt32, IO::ByteFormat::BigEndian).to_i
      value = Bytes.new(value_len)
      io.read_fully(value)
      
      db.set(key, value)
    end
    
    db
  end
end

# ใช้งาน
db = BinaryDB.new
db.set("name", "Alice")
db.set("version", "1.0.0")
db.set("data", Bytes[1, 2, 3, 4, 5])

serialized = db.serialize
puts "Serialized size: #{serialized.size} bytes"
puts bytes_to_hex_pretty(serialized)

restored = BinaryDB.deserialize(serialized)
puts restored.get_string("name")     # => Alice
puts restored.get_string("version")  # => 1.0.0
puts restored.get("data").inspect    # => Bytes[1, 2, 3, 4, 5]

# บันทึกและโหลด
File.open("/tmp/test.bdf", "wb") { |f| f.write(serialized) }
loaded_data = File.open("/tmp/test.bdf", "rb") do |f|
  buf = Bytes.new(f.size.to_i)
  f.read_fully(buf)
  buf
end
db2 = BinaryDB.deserialize(loaded_data)
puts db2.get_string("name")  # => Alice
File.delete("/tmp/test.bdf")
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `BinaryBuffer` ที่รองรับ read/write สำหรับ primitive types (Int8, Int16, Int32, Int64, Float32, Float64) ทั้ง big-endian และ little-endian

### แบบฝึกหัดที่ 2
เขียน checksum functions: CRC8 และ simple XOR checksum

### แบบฝึกหัดที่ 3
เขียน simple binary diff: เปรียบเทียบสอง Bytes arrays และแสดงความแตกต่าง

### เฉลย

```crystal
# แบบฝึกหัดที่ 1: BinaryBuffer
class BinaryBuffer
  def initialize(capacity : Int32 = 64)
    @io = IO::Memory.new(capacity)
    @byte_format = IO::ByteFormat::BigEndian
  end
  
  def self.new(data : Bytes) : BinaryBuffer
    buf = new(data.size)
    buf.write_raw(data)
    buf.rewind
    buf
  end
  
  def big_endian! : self
    @byte_format = IO::ByteFormat::BigEndian
    self
  end
  
  def little_endian! : self
    @byte_format = IO::ByteFormat::LittleEndian
    self
  end
  
  def write_int8(v : Int8) : self
    @io.write_byte(v.unsafe_as(UInt8))
    self
  end
  
  def write_int16(v : Int16) : self
    @io.write_bytes(v, @byte_format)
    self
  end
  
  def write_int32(v : Int32) : self
    @io.write_bytes(v, @byte_format)
    self
  end
  
  def write_int64(v : Int64) : self
    @io.write_bytes(v, @byte_format)
    self
  end
  
  def write_float32(v : Float32) : self
    @io.write_bytes(v, @byte_format)
    self
  end
  
  def write_float64(v : Float64) : self
    @io.write_bytes(v, @byte_format)
    self
  end
  
  def write_string(s : String) : self
    bytes = s.to_slice
    write_int32(bytes.size.to_i32)
    @io.write(bytes)
    self
  end
  
  def write_raw(data : Bytes) : self
    @io.write(data)
    self
  end
  
  def read_int8 : Int8
    @io.read_byte.not_nil!.unsafe_as(Int8)
  end
  
  def read_int16 : Int16
    @io.read_bytes(Int16, @byte_format)
  end
  
  def read_int32 : Int32
    @io.read_bytes(Int32, @byte_format)
  end
  
  def read_int64 : Int64
    @io.read_bytes(Int64, @byte_format)
  end
  
  def read_float32 : Float32
    @io.read_bytes(Float32, @byte_format)
  end
  
  def read_float64 : Float64
    @io.read_bytes(Float64, @byte_format)
  end
  
  def read_string : String
    len = read_int32
    buf = Bytes.new(len)
    @io.read_fully(buf)
    String.new(buf)
  end
  
  def rewind : self
    @io.rewind
    self
  end
  
  def to_bytes : Bytes
    @io.to_slice
  end
  
  def size : Int32
    @io.size.to_i
  end
end

# Test BinaryBuffer
buf = BinaryBuffer.new.big_endian!
buf.write_int32(12345)
buf.write_float64(3.14159)
buf.write_string("Hello")
buf.write_int8(42)

data = buf.to_bytes
puts "Encoded #{data.size} bytes"

reader = BinaryBuffer.new(data).big_endian!
puts reader.read_int32    # => 12345
puts reader.read_float64  # => 3.14159
puts reader.read_string   # => Hello
puts reader.read_int8     # => 42

# แบบฝึกหัดที่ 2: Checksums
def xor_checksum(data : Bytes) : UInt8
  data.reduce(0_u8) { |acc, b| acc ^ b }
end

def crc8(data : Bytes) : UInt8
  crc = 0_u8
  data.each do |byte|
    crc ^= byte
    8.times do
      if (crc & 0x80) != 0
        crc = ((crc << 1) ^ 0x07_u8) & 0xFF_u8
      else
        crc = (crc << 1) & 0xFF_u8
      end
    end
  end
  crc
end

test_data = "Hello, World!".to_slice
puts "XOR checksum: 0x%02X" % xor_checksum(test_data)
puts "CRC8: 0x%02X" % crc8(test_data)

# ตรวจสอบ integrity
packet = test_data + Bytes[crc8(test_data)]
puts "Valid: #{crc8(packet) == 0}"  # CRC ถูกต้อง => 0

corrupted = packet.dup
corrupted[0] ^= 0x01  # เปลี่ยน 1 bit
puts "Valid: #{crc8(corrupted) == 0}"  # ผิดพลาด
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Bytes** - Slice(UInt8) สำหรับ binary data
2. **Reading binary** - magic bytes, file detection
3. **IO::ByteFormat** - big-endian, little-endian
4. **Bit manipulation** - AND, OR, XOR, shifts, flags
5. **Protocol implementation** - custom binary protocol
6. **Hex encoding** - bytes to hex, hexdump
7. **Base64** - encode/decode, data URLs
8. **Binary file format** - custom binary database

Binary data manipulation เป็นพื้นฐานสำคัญสำหรับ systems programming, network protocols, และ high-performance applications
