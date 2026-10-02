# Part 176: Static Libraries ใน Crystal

## บทนำ

Static Libraries (.a files บน Linux, .lib บน Windows) คือ collections ของ object files ที่ถูก compile ล่วงหน้าและ link เข้าไปใน binary ตอน compile time ทำให้ binary เป็น self-contained ไม่ต้องพึ่ง external dependencies ตอน runtime

## ทำไมต้องใช้ Static Libraries

```
Static Library (.a):
+ Binary เป็น self-contained
+ ไม่มี version conflict ตอน runtime
+ Deploy ง่าย (แค่ copy binary)
- Binary size ใหญ่กว่า
- อัปเดต library ต้อง recompile ทุกครั้ง

Shared Library (.so/.dylib):
+ Binary size เล็กกว่า
+ หลาย programs share library ได้
- ต้อง install library บน target machine
- อาจมี version conflict
```

## สร้าง Static Library จาก C

```c
/* math_utils.c */
#include "math_utils.h"
#include <math.h>

double fast_sqrt(double x) {
    return sqrt(x);
}

double fast_pow(double base, int exp) {
    double result = 1.0;
    while (exp > 0) {
        if (exp & 1) result *= base;
        base *= base;
        exp >>= 1;
    }
    return result;
}

long long fibonacci(int n) {
    if (n <= 1) return n;
    long long a = 0, b = 1;
    for (int i = 2; i <= n; i++) {
        long long c = a + b;
        a = b;
        b = c;
    }
    return b;
}

/* string_utils.c */
#include "string_utils.h"
#include <string.h>
#include <ctype.h>

void str_to_upper(char *str) {
    for (int i = 0; str[i]; i++) {
        str[i] = toupper(str[i]);
    }
}

int str_count_words(const char *str) {
    int count = 0;
    int in_word = 0;
    for (int i = 0; str[i]; i++) {
        if (str[i] != ' ' && str[i] != '\t' && str[i] != '\n') {
            if (!in_word) { count++; in_word = 1; }
        } else {
            in_word = 0;
        }
    }
    return count;
}
```

```bash
# compile เป็น object files
gcc -c -O2 math_utils.c -o math_utils.o
gcc -c -O2 string_utils.c -o string_utils.o

# สร้าง static library
ar rcs libmyutils.a math_utils.o string_utils.o

# ดู contents ของ library
ar t libmyutils.a
# math_utils.o
# string_utils.o

# ดู symbols ใน library
nm libmyutils.a
```

## Linking กับ Static Library ใน Crystal

```crystal
# @[Link] กับ static library
@[Link("myutils", static: true)]
@[Link(ldflags: "-L./lib")]
lib LibMyUtils
  fun fast_sqrt(x : Float64) : Float64
  fun fast_pow(base : Float64, exp : Int32) : Float64
  fun fibonacci(n : Int32) : Int64
  fun str_to_upper(str : UInt8*) : Void
  fun str_count_words(str : UInt8*) : Int32
end

# ใช้งาน
puts LibMyUtils.fast_sqrt(16.0)    # 4.0
puts LibMyUtils.fast_pow(2.0, 10)  # 1024.0
puts LibMyUtils.fibonacci(10)       # 55

buffer = "hello world crystal".dup
LibMyUtils.str_to_upper(buffer)
puts String.new(buffer)  # HELLO WORLD CRYSTAL

words = LibMyUtils.str_count_words("count these words please")
puts "Words: #{words}"  # 4
```

## สร้าง Static Library จาก Crystal เอง

```crystal
# Crystal ไม่ compile เป็น .a โดยตรงง่ายๆ
# แต่สามารถ emit object file ได้

# Compile Crystal ไปยัง object file
# crystal build --emit obj -o mylib.o src/mylib.cr

# src/mylib.cr
# ส่วน public API ที่ต้องการ export

def add(a : Int32, b : Int32) : Int32
  a + b
end

def multiply(a : Float64, b : Float64) : Float64
  a * b
end
```

```makefile
# Makefile
CC = gcc
CRYSTAL = crystal
AR = ar

SRCS = math_utils.c string_utils.c
OBJS = $(SRCS:.c=.o)
LIB = libmyutils.a

.PHONY: all clean

all: $(LIB)

%.o: %.c
	$(CC) -c -O2 -fPIC $< -o $@

$(LIB): $(OBJS)
	$(AR) rcs $@ $^

clean:
	rm -f $(OBJS) $(LIB)
```

## Embedded Resources

```crystal
# macros/embed_resource.cr
# embed file content ลงใน binary

macro embed_file(filename)
  {{ run("./scripts/embed_file.cr", filename) }}
end

# scripts/embed_file.cr
filename = ARGV[0]
content = File.read(filename)
content.inspect  # output as Crystal string literal
```

```crystal
# วิธีที่ดีกว่า - ใช้ macro read_file
module Resources
  # Embed file content ตอน compile time
  HTML_TEMPLATE = {{ read_file("./templates/index.html") }}
  CSS_CONTENT   = {{ read_file("./assets/style.css") }}
  CONFIG_JSON   = {{ read_file("./config/default.json") }}

  def self.html_template : String
    HTML_TEMPLATE
  end

  def self.css : String
    CSS_CONTENT
  end
end

puts Resources::HTML_TEMPLATE.size
puts Resources::CSS_CONTENT.size
```

## Complete Example: Crypto Library

```c
/* src/crypto_utils.c */
#include <openssl/evp.h>
#include <string.h>
#include <stdlib.h>

/* SHA-256 hash */
int sha256_hash(const unsigned char *input, size_t input_len,
                unsigned char *output, unsigned int *output_len) {
    EVP_MD_CTX *ctx = EVP_MD_CTX_new();
    if (!ctx) return 0;

    int ret = EVP_DigestInit_ex(ctx, EVP_sha256(), NULL) &&
              EVP_DigestUpdate(ctx, input, input_len) &&
              EVP_DigestFinal_ex(ctx, output, output_len);

    EVP_MD_CTX_free(ctx);
    return ret;
}

/* HMAC-SHA256 */
int hmac_sha256(const unsigned char *key, size_t key_len,
                const unsigned char *data, size_t data_len,
                unsigned char *output, unsigned int *output_len) {
    return HMAC(EVP_sha256(), key, (int)key_len,
                data, data_len, output, output_len) != NULL;
}

/* Constant time comparison */
int constant_time_compare(const unsigned char *a, const unsigned char *b, size_t n) {
    unsigned char result = 0;
    for (size_t i = 0; i < n; i++) {
        result |= a[i] ^ b[i];
    }
    return result == 0;
}
```

```bash
# Build crypto static library
gcc -c -O2 -fPIC src/crypto_utils.c -o crypto_utils.o \
    $(pkg-config --cflags openssl)
ar rcs libcrypto_utils.a crypto_utils.o
```

```crystal
# Crystal binding สำหรับ crypto library
@[Link("crypto_utils", static: true)]
@[Link(ldflags: "-L./lib `pkg-config --libs openssl`")]
lib LibCryptoUtils
  SHA256_DIGEST_SIZE = 32
  HMAC_SHA256_SIZE   = 32

  fun sha256_hash(
    input : UInt8*, input_len : LibC::SizeT,
    output : UInt8*, output_len : UInt32*
  ) : Int32

  fun hmac_sha256(
    key : UInt8*, key_len : LibC::SizeT,
    data : UInt8*, data_len : LibC::SizeT,
    output : UInt8*, output_len : UInt32*
  ) : Int32

  fun constant_time_compare(
    a : UInt8*, b : UInt8*, n : LibC::SizeT
  ) : Int32
end

module CryptoLib
  def self.sha256(data : String | Bytes) : Bytes
    bytes = data.is_a?(String) ? data.to_slice : data
    output = Bytes.new(LibCryptoUtils::SHA256_DIGEST_SIZE)
    len = LibCryptoUtils::SHA256_DIGEST_SIZE.to_u32

    result = LibCryptoUtils.sha256_hash(
      bytes.to_unsafe, bytes.size.to_u64,
      output.to_unsafe, pointerof(len)
    )

    raise "SHA256 failed" unless result == 1
    output[0, len.to_i]
  end

  def self.hmac_sha256(key : String, data : String) : Bytes
    output = Bytes.new(LibCryptoUtils::HMAC_SHA256_SIZE)
    len = LibCryptoUtils::HMAC_SHA256_SIZE.to_u32

    result = LibCryptoUtils.hmac_sha256(
      key.to_unsafe, key.size.to_u64,
      data.to_unsafe, data.size.to_u64,
      output.to_unsafe, pointerof(len)
    )

    raise "HMAC-SHA256 failed" unless result == 1
    output[0, len.to_i]
  end

  def self.secure_compare(a : Bytes, b : Bytes) : Bool
    return false if a.size != b.size
    LibCryptoUtils.constant_time_compare(
      a.to_unsafe, b.to_unsafe, a.size.to_u64
    ) == 1
  end
end

# ใช้งาน
hash = CryptoLib.sha256("Hello, Crystal!")
puts "SHA256: #{hash.hexstring}"

hmac = CryptoLib.hmac_sha256("secret_key", "message")
puts "HMAC: #{hmac.hexstring}"

a = Bytes[1, 2, 3]
b = Bytes[1, 2, 3]
c = Bytes[1, 2, 4]

puts CryptoLib.secure_compare(a, b)  # true
puts CryptoLib.secure_compare(a, c)  # false
```

## Build Script

```crystal
# build.cr - build script
require "compiler/crystal/*"

def build_static_lib
  # Compile C sources
  system("gcc -c -O2 src/*.c -I include/ -o build/*.o")

  # Create static library
  system("ar rcs lib/libmyapp.a build/*.o")

  puts "Static library created: lib/libmyapp.a"
end

def build_crystal_app
  system("crystal build src/main.cr -o bin/myapp --link-flags '-L./lib'")
  puts "Crystal app built: bin/myapp"
end

build_static_lib
build_crystal_app
```

## แบบฝึกหัด

1. สร้าง static library สำหรับ matrix operations (multiply, transpose, determinant) ใน C แล้วใช้จาก Crystal
2. Embed SQLite3 source code ลงใน binary (amalgamation build) แล้วใช้ผ่าน Crystal bindings
3. สร้าง static library จาก Crystal code ที่ export C-compatible functions
4. Build ระบบที่ compile ต่าง platforms (Linux/macOS) โดยใช้ conditional linking

## สรุป

Static Libraries ใน Crystal:
- **@[Link("name", static: true)]**: link กับ static library
- **@[Link(ldflags: "-L./path")]**: ระบุ path ของ library
- **ar rcs**: สร้าง static library จาก object files
- **Embedded resources**: ใช้ `{{ read_file }}` embed file content ตอน compile time
- **Self-contained binaries**: static linking ทำให้ deploy ง่าย

Static libraries เหมาะสำหรับ production deployments ที่ต้องการความ portable
