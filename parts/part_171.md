# Part 171: C Bindings พื้นฐาน ใน Crystal

## บทนำ

Crystal สามารถเรียก C libraries ได้โดยตรงด้วย `lib` keyword ซึ่งเป็น zero-overhead FFI (Foreign Function Interface) ทำให้ใช้ ecosystem ของ C ได้ทั้งหมด

## lib และ @[Link]

```crystal
# lib LibName - ประกาศ C library
@[Link("m")]  # link กับ libm (math library)
lib LibMath
  fun sqrt(x : Float64) : Float64
  fun pow(x : Float64, y : Float64) : Float64
  fun abs(x : Float64) : Float64
  fun cos(x : Float64) : Float64
  fun sin(x : Float64) : Float64
  fun log(x : Float64) : Float64
  fun log2(x : Float64) : Float64
  fun log10(x : Float64) : Float64
end

# เรียกใช้
puts LibMath.sqrt(16.0)   # 4.0
puts LibMath.pow(2.0, 10.0)  # 1024.0
puts LibMath.cos(0.0)     # 1.0
```

## Fun Declarations

```crystal
# การประกาศ C functions

# Function ง่ายๆ
lib LibSimple
  fun hello() : Void
  fun add(a : Int32, b : Int32) : Int32
  fun strlen(str : UInt8*) : LibC::SizeT
end

# Function พร้อม C types
lib LibTypes
  # Integer types
  fun get_int8() : Int8
  fun get_uint16() : UInt16
  fun get_int32() : Int32
  fun get_uint64() : UInt64

  # Float types
  fun get_float32() : Float32
  fun get_float64() : Float64

  # Pointer types
  fun get_string() : UInt8*
  fun get_int_ptr() : Int32*

  # Void pointer
  fun malloc(size : LibC::SizeT) : Void*
  fun free(ptr : Void*) : Void

  # Boolean (C ใช้ Int32 แทน Bool)
  fun is_valid(x : Int32) : Int32
end
```

## C Types ใน Crystal

```crystal
# C types mapping:
# C Type         Crystal Type
# void           Void
# char           UInt8
# unsigned char  UInt8
# short          Int16
# unsigned short UInt16
# int            Int32
# unsigned int   UInt32
# long           Int64 (หรือ Int32 ขึ้นกับ platform)
# unsigned long  UInt64
# long long      Int64
# size_t         LibC::SizeT
# ptrdiff_t      LibC::PtrdiffT
# float          Float32
# double         Float64
# void*          Void*
# char*          UInt8*
# int*           Int32*

# Struct ใน C
lib LibPoint
  struct Point
    x : Float64
    y : Float64
  end

  fun distance(p1 : Point, p2 : Point) : Float64
  fun midpoint(p1 : Point, p2 : Point) : Point
end

# Union ใน C
lib LibUnion
  union Data
    i : Int32
    f : Float32
    b : UInt8[4]
  end
end

# Enum ใน C
lib LibStatus
  enum Status
    OK      = 0
    ERROR   = 1
    TIMEOUT = 2
  end

  fun check_status() : Status
end
```

## Calling C Functions

```crystal
# เรียก C function ง่ายๆ
@[Link("c")]
lib LibC
  fun printf(format : UInt8*, ...) : Int32
  fun scanf(format : UInt8*, ...) : Int32
  fun puts(str : UInt8*) : Int32
  fun getchar() : Int32
  fun putchar(c : Int32) : Int32
end

# เรียก puts จาก C
LibC.puts("Hello from C!".to_unsafe)

# เรียก printf จาก C
LibC.printf("Number: %d\n".to_unsafe, 42)
LibC.printf("Float: %.2f\n".to_unsafe, 3.14159)
```

## Passing Strings

```crystal
@[Link("c")]
lib LibC
  fun strlen(str : UInt8*) : LibC::SizeT
  fun strcmp(s1 : UInt8*, s2 : UInt8*) : Int32
  fun strcpy(dst : UInt8*, src : UInt8*) : UInt8*
  fun strcat(dst : UInt8*, src : UInt8*) : UInt8*
  fun toupper(c : Int32) : Int32
end

# String ใน Crystal → UInt8* สำหรับ C
str = "Hello, World!"
length = LibC.strlen(str)  # Crystal ส่ง String ได้โดยตรง

puts "Length: #{length}"

# String comparison
result = LibC.strcmp("abc", "abc")
puts result == 0 ? "Equal" : "Not equal"

# ต้องระวัง null terminator!
# Crystal's String.to_unsafe คืน null-terminated pointer
buffer = Bytes.new(256)
LibC.strcpy(buffer.to_unsafe.as(UInt8*), "Hello")
puts String.new(buffer.to_unsafe)
```

## Returning Values

```crystal
@[Link("ssl")]
lib LibSSL
  fun SSL_library_init() : Int32
  fun RAND_bytes(buf : UInt8*, num : Int32) : Int32
  fun ERR_get_error() : UInt64
  fun ERR_error_string(e : UInt64, buf : UInt8*) : UInt8*
end

# รับค่า integer
result = LibSSL.SSL_library_init
puts "SSL initialized: #{result == 1}"

# รับ random bytes
buffer = Bytes.new(32)
result = LibSSL.RAND_bytes(buffer.to_unsafe, 32)
if result == 1
  puts "Random bytes: #{buffer.hexstring}"
end

# รับ string pointer จาก C
lib LibDynamo
  fun get_version() : UInt8*
end

version_ptr = LibDynamo.get_version
if version_ptr
  version = String.new(version_ptr)
  puts "Version: #{version}"
end
```

## Error Handling ด้วย C

```crystal
lib LibSystem
  fun open(pathname : UInt8*, flags : Int32) : Int32
  fun close(fd : Int32) : Int32
  fun read(fd : Int32, buf : Void*, count : LibC::SizeT) : LibC::SSizeT

  # errno
  $errno : Int32
end

# เปิดไฟล์ด้วย C
fd = LibSystem.open("/etc/hostname", 0)  # O_RDONLY = 0

if fd < 0
  puts "Error opening file: errno=#{LibSystem.errno}"
else
  buffer = Bytes.new(256)
  bytes_read = LibSystem.read(fd, buffer.to_unsafe, 255)

  if bytes_read > 0
    content = String.new(buffer[0, bytes_read.to_i].to_unsafe)
    puts "Hostname: #{content.strip}"
  end

  LibSystem.close(fd)
end
```

## Callbacks

```crystal
# C callback function types
lib LibSort
  alias CompareFn = (Void*, Void*) -> Int32

  fun qsort(
    base  : Void*,
    nmemb : LibC::SizeT,
    size  : LibC::SizeT,
    compar : CompareFn
  ) : Void
end

# สร้าง callback ใน Crystal
compare = ->(a : Void*, b : Void*) {
  va = a.as(Int32*)
  vb = b.as(Int32*)
  va.value <=> vb.value
}

# ใช้ callback กับ qsort
arr = [5, 2, 8, 1, 9, 3].map(&.to_i32)
LibSort.qsort(
  arr.to_unsafe.as(Void*),
  arr.size.to_u64,
  sizeof(Int32).to_u64,
  compare
)
puts arr.inspect  # [1, 2, 3, 5, 8, 9]
```

## Struct Operations

```crystal
lib LibGeometry
  struct Vector2
    x : Float64
    y : Float64
  end

  struct Matrix2x2
    a11 : Float64
    a12 : Float64
    a21 : Float64
    a22 : Float64
  end

  fun vec_add(a : Vector2, b : Vector2) : Vector2
  fun vec_dot(a : Vector2, b : Vector2) : Float64
  fun mat_multiply(a : Matrix2x2, b : Matrix2x2) : Matrix2x2
  fun vec_transform(v : Vector2, m : Matrix2x2) : Vector2
end

# สร้าง struct
v1 = LibGeometry::Vector2.new(x: 1.0, y: 0.0)
v2 = LibGeometry::Vector2.new(x: 0.0, y: 1.0)

# เรียก C function พร้อม struct
result = LibGeometry.vec_add(v1, v2)
puts "(#{result.x}, #{result.y})"  # (1.0, 1.0)

dot = LibGeometry.vec_dot(v1, v2)
puts "Dot product: #{dot}"  # 0.0
```

## Example: Wrapping libcurl

```crystal
@[Link("curl")]
lib LibCurl
  enum CurlCode
    OK = 0
    FAILED_INIT = 2
    URL_MALFORMAT = 3
    COULDNT_RESOLVE_HOST = 6
    COULDNT_CONNECT = 7
    OPERATION_TIMEDOUT = 28
    # ... อื่นๆ
  end

  alias CURL = Void
  alias CurlSlist = Void
  alias WriteCallbackFn = (UInt8*, LibC::SizeT, LibC::SizeT, Void*) -> LibC::SizeT

  fun curl_easy_init() : CURL*
  fun curl_easy_setopt(curl : CURL*, option : Int32, ...) : CurlCode
  fun curl_easy_perform(curl : CURL*) : CurlCode
  fun curl_easy_cleanup(curl : CURL*) : Void
  fun curl_easy_strerror(code : CurlCode) : UInt8*
  fun curl_slist_append(list : CurlSlist*, header : UInt8*) : CurlSlist*
  fun curl_slist_free_all(list : CurlSlist*) : Void
end

# Constants
CURLOPT_URL            = 10002
CURLOPT_FOLLOWLOCATION = 52
CURLOPT_WRITEFUNCTION  = 20011
CURLOPT_WRITEDATA      = 10001
CURLOPT_TIMEOUT        = 13
CURLOPT_SSL_VERIFYPEER = 64

# Crystal wrapper
class SimpleCurl
  class Error < Exception
    def initialize(code : LibCurl::CurlCode)
      msg = String.new(LibCurl.curl_easy_strerror(code))
      super("CURL Error: #{msg} (#{code})")
    end
  end

  def self.get(url : String) : String
    curl = LibCurl.curl_easy_init
    raise "Failed to init curl" unless curl

    response = IO::Memory.new

    # Callback สำหรับรับ data
    write_cb = ->(ptr : UInt8*, size : LibC::SizeT, nmemb : LibC::SizeT, userdata : Void*) {
      data_io = userdata.as(IO::Memory*)
      bytes_read = size * nmemb
      data_io.value.write(Slice.new(ptr, bytes_read.to_i))
      bytes_read
    }

    begin
      LibCurl.curl_easy_setopt(curl, CURLOPT_URL, url.to_unsafe)
      LibCurl.curl_easy_setopt(curl, CURLOPT_FOLLOWLOCATION, 1_i64)
      LibCurl.curl_easy_setopt(curl, CURLOPT_TIMEOUT, 30_i64)
      LibCurl.curl_easy_setopt(curl, CURLOPT_SSL_VERIFYPEER, 1_i64)
      LibCurl.curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, write_cb)
      LibCurl.curl_easy_setopt(curl, CURLOPT_WRITEDATA, pointerof(response))

      code = LibCurl.curl_easy_perform(curl)
      raise Error.new(code) unless code == LibCurl::CurlCode::OK

      response.to_s
    ensure
      LibCurl.curl_easy_cleanup(curl)
    end
  end
end

# ใช้งาน
begin
  body = SimpleCurl.get("https://httpbin.org/get")
  puts body[0..200]
rescue SimpleCurl::Error => e
  puts "Error: #{e.message}"
end
```

## แบบฝึกหัด

1. สร้าง binding สำหรับ `libz` (zlib) เพื่อ compress/decompress strings
2. เขียน Crystal wrapper สำหรับ C's `getopt` เพื่อ parse command-line arguments
3. สร้าง binding สำหรับ SDL2 สำหรับ simple window creation
4. Wrap `libsodium` สำหรับ cryptographic operations

## สรุป

C Bindings ใน Crystal:
- **lib**: ประกาศ C interface
- **@[Link]**: ระบุ library ที่ต้อง link
- **fun**: ประกาศ C functions
- **C types**: Int8-Int64, UInt8-UInt64, Float32, Float64, Void*, pointer types
- **struct/union/enum**: C complex types
- **Callbacks**: ส่ง Crystal proc เป็น C function pointer

C bindings ทำให้ Crystal เข้าถึง C ecosystem ได้ทั้งหมดโดยไม่มี overhead
