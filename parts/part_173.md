# Part 173: External Libraries ใน Crystal

## บทนำ

Crystal สามารถ link กับ C libraries ภายนอกได้อย่างง่ายดาย ในบทนี้จะครอบคลุมการ link กับ OpenSSL, SQLite, และ libraries อื่นๆ

## การ Link ด้วย @[Link]

```crystal
# @[Link] options:
# @[Link("libname")]         - link กับ library name
# @[Link(ldflags: "flags")]  - pass flags ไปยัง linker โดยตรง
# @[Link(framework: "Name")] - macOS framework
# @[Link(static: true)]      - force static linking
# @[Link("name", static: true)] - static + name
```

## pkg-config

```crystal
# ใช้ pkg-config เพื่อหา compiler/linker flags อัตโนมัติ
# @[Link(ldflags: "`pkg-config --libs openssl`")]

@[Link(ldflags: "`pkg-config --libs libcurl`")]
lib LibCurlPkg
  # ...
end
```

## Linking กับ OpenSSL

```crystal
# shard.yml ไม่ต้องเพิ่มอะไร แค่ link library

@[Link(ldflags: "`pkg-config --libs openssl 2>/dev/null || echo '-lssl -lcrypto'`")]
lib LibSSL
  # SSL Context
  SSL_CTX = Void
  SSL = Void

  # Methods
  fun SSL_library_init() : Int32
  fun SSL_load_error_strings() : Void
  fun OpenSSL_add_all_algorithms() : Void

  # Context
  fun TLS_method() : Void*
  fun SSL_CTX_new(method : Void*) : SSL_CTX*
  fun SSL_CTX_free(ctx : SSL_CTX*) : Void
  fun SSL_CTX_use_certificate_file(ctx : SSL_CTX*, file : UInt8*, type : Int32) : Int32
  fun SSL_CTX_use_PrivateKey_file(ctx : SSL_CTX*, file : UInt8*, type : Int32) : Int32

  # Connection
  fun SSL_new(ctx : SSL_CTX*) : SSL*
  fun SSL_free(ssl : SSL*) : Void
  fun SSL_set_fd(ssl : SSL*, fd : Int32) : Int32
  fun SSL_connect(ssl : SSL*) : Int32
  fun SSL_accept(ssl : SSL*) : Int32
  fun SSL_read(ssl : SSL*, buf : Void*, num : Int32) : Int32
  fun SSL_write(ssl : SSL*, buf : Void*, num : Int32) : Int32
  fun SSL_shutdown(ssl : SSL*) : Int32

  # Error
  fun SSL_get_error(ssl : SSL*, ret : Int32) : Int32
  fun ERR_get_error() : UInt64
  fun ERR_error_string(e : UInt64, buf : UInt8*) : UInt8*
end

lib LibCrypto
  # Digest
  EVP_MD_CTX = Void
  EVP_MD = Void

  fun EVP_MD_CTX_new() : EVP_MD_CTX*
  fun EVP_MD_CTX_free(ctx : EVP_MD_CTX*) : Void
  fun EVP_DigestInit_ex(ctx : EVP_MD_CTX*, type : EVP_MD*, impl : Void*) : Int32
  fun EVP_DigestUpdate(ctx : EVP_MD_CTX*, d : Void*, cnt : LibC::SizeT) : Int32
  fun EVP_DigestFinal_ex(ctx : EVP_MD_CTX*, md : UInt8*, s : UInt32*) : Int32
  fun EVP_sha256() : EVP_MD*
  fun EVP_sha512() : EVP_MD*

  # HMAC
  fun HMAC(evp_md : EVP_MD*, key : Void*, key_len : Int32, d : UInt8*, n : LibC::SizeT, md : UInt8*, md_len : UInt32*) : UInt8*

  # Random
  fun RAND_bytes(buf : UInt8*, num : Int32) : Int32

  # Base64
  fun EVP_EncodeBlock(t : UInt8*, f : UInt8*, dlen : Int32) : Int32
  fun EVP_DecodeBlock(t : UInt8*, f : UInt8*, n : Int32) : Int32
end

# Crystal wrapper สำหรับ OpenSSL
module Crypto
  def self.sha256(data : String | Bytes) : Bytes
    ctx = LibCrypto.EVP_MD_CTX_new
    raise "Failed to create context" unless ctx

    begin
      md = LibCrypto.EVP_sha256
      LibCrypto.EVP_DigestInit_ex(ctx, md, nil)

      bytes = data.is_a?(String) ? data.to_slice : data
      LibCrypto.EVP_DigestUpdate(ctx, bytes.to_unsafe.as(Void*), bytes.size.to_u64)

      digest = Bytes.new(32)
      len = 32_u32
      LibCrypto.EVP_DigestFinal_ex(ctx, digest.to_unsafe, pointerof(len))
      digest[0, len.to_i]
    ensure
      LibCrypto.EVP_MD_CTX_free(ctx)
    end
  end

  def self.hmac_sha256(key : String, data : String) : Bytes
    digest = Bytes.new(32)
    len = 32_u32

    LibCrypto.HMAC(
      LibCrypto.EVP_sha256,
      key.to_unsafe.as(Void*),
      key.size,
      data.to_unsafe,
      data.size.to_u64,
      digest.to_unsafe,
      pointerof(len)
    )

    digest[0, len.to_i]
  end

  def self.random_bytes(n : Int32) : Bytes
    bytes = Bytes.new(n)
    result = LibCrypto.RAND_bytes(bytes.to_unsafe, n)
    raise "Failed to generate random bytes" unless result == 1
    bytes
  end
end

# ใช้งาน
hash = Crypto.sha256("Hello, World!")
puts "SHA256: #{hash.hexstring}"

hmac = Crypto.hmac_sha256("secret_key", "message")
puts "HMAC-SHA256: #{hmac.hexstring}"

random = Crypto.random_bytes(16)
puts "Random: #{random.hexstring}"
```

## Linking กับ SQLite

```crystal
@[Link("sqlite3")]
lib LibSQLite3
  DATABASE = Void
  STMT = Void

  # Return codes
  SQLITE_OK    = 0
  SQLITE_ERROR = 1
  SQLITE_ROW   = 100
  SQLITE_DONE  = 101

  # Column types
  SQLITE_INTEGER = 1
  SQLITE_FLOAT   = 2
  SQLITE_TEXT    = 3
  SQLITE_BLOB    = 4
  SQLITE_NULL    = 5

  # Open flags
  SQLITE_OPEN_READONLY  =  1
  SQLITE_OPEN_READWRITE =  2
  SQLITE_OPEN_CREATE    =  4

  fun sqlite3_open(filename : UInt8*, ppDb : DATABASE**) : Int32
  fun sqlite3_open_v2(filename : UInt8*, ppDb : DATABASE**, flags : Int32, zVfs : UInt8*) : Int32
  fun sqlite3_close(db : DATABASE*) : Int32
  fun sqlite3_errmsg(db : DATABASE*) : UInt8*
  fun sqlite3_errcode(db : DATABASE*) : Int32

  fun sqlite3_exec(db : DATABASE*, sql : UInt8*, callback : Void*, arg : Void*, errmsg : UInt8**) : Int32

  fun sqlite3_prepare_v2(db : DATABASE*, sql : UInt8*, nByte : Int32, ppStmt : STMT**, pzTail : UInt8**) : Int32
  fun sqlite3_step(stmt : STMT*) : Int32
  fun sqlite3_finalize(stmt : STMT*) : Int32
  fun sqlite3_reset(stmt : STMT*) : Int32

  fun sqlite3_bind_int(stmt : STMT*, idx : Int32, value : Int32) : Int32
  fun sqlite3_bind_int64(stmt : STMT*, idx : Int32, value : Int64) : Int32
  fun sqlite3_bind_double(stmt : STMT*, idx : Int32, value : Float64) : Int32
  fun sqlite3_bind_text(stmt : STMT*, idx : Int32, value : UInt8*, n : Int32, destructor : Void*) : Int32
  fun sqlite3_bind_null(stmt : STMT*, idx : Int32) : Int32
  fun sqlite3_bind_blob(stmt : STMT*, idx : Int32, value : Void*, n : Int32, destructor : Void*) : Int32

  fun sqlite3_column_count(stmt : STMT*) : Int32
  fun sqlite3_column_name(stmt : STMT*, n : Int32) : UInt8*
  fun sqlite3_column_type(stmt : STMT*, iCol : Int32) : Int32
  fun sqlite3_column_int(stmt : STMT*, iCol : Int32) : Int32
  fun sqlite3_column_int64(stmt : STMT*, iCol : Int32) : Int64
  fun sqlite3_column_double(stmt : STMT*, iCol : Int32) : Float64
  fun sqlite3_column_text(stmt : STMT*, iCol : Int32) : UInt8*
  fun sqlite3_column_bytes(stmt : STMT*, iCol : Int32) : Int32

  fun sqlite3_last_insert_rowid(db : DATABASE*) : Int64
  fun sqlite3_changes(db : DATABASE*) : Int32
end

# Crystal wrapper สำหรับ SQLite
class SQLiteDB
  class Error < Exception
    def initialize(msg : String)
      super("SQLite Error: #{msg}")
    end
  end

  def initialize(path : String)
    db_ptr = Pointer(LibSQLite3::DATABASE).null
    result = LibSQLite3.sqlite3_open(path, pointerof(db_ptr))
    raise Error.new("Cannot open database: #{path}") unless result == LibSQLite3::SQLITE_OK
    @db = db_ptr
  end

  def close
    LibSQLite3.sqlite3_close(@db)
  end

  def exec(sql : String)
    errmsg = Pointer(UInt8).null
    result = LibSQLite3.sqlite3_exec(@db, sql, nil, nil, pointerof(errmsg))
    if result != LibSQLite3::SQLITE_OK && errmsg
      msg = String.new(errmsg)
      raise Error.new(msg)
    end
  end

  def query(sql : String, *params) : Array(Hash(String, String?))
    stmt_ptr = Pointer(LibSQLite3::STMT).null
    result = LibSQLite3.sqlite3_prepare_v2(@db, sql, -1, pointerof(stmt_ptr), nil)
    raise Error.new(error_message) unless result == LibSQLite3::SQLITE_OK

    stmt = stmt_ptr

    # Bind parameters
    params.each_with_index do |param, i|
      bind_param(stmt, i + 1, param)
    end

    rows = [] of Hash(String, String?)
    col_count = LibSQLite3.sqlite3_column_count(stmt)

    while LibSQLite3.sqlite3_step(stmt) == LibSQLite3::SQLITE_ROW
      row = {} of String => String?
      col_count.times do |col|
        name_ptr = LibSQLite3.sqlite3_column_name(stmt, col)
        name = name_ptr ? String.new(name_ptr) : "col#{col}"

        value = case LibSQLite3.sqlite3_column_type(stmt, col)
        when LibSQLite3::SQLITE_NULL    then nil
        when LibSQLite3::SQLITE_INTEGER then LibSQLite3.sqlite3_column_int64(stmt, col).to_s
        when LibSQLite3::SQLITE_FLOAT   then LibSQLite3.sqlite3_column_double(stmt, col).to_s
        when LibSQLite3::SQLITE_TEXT
          text_ptr = LibSQLite3.sqlite3_column_text(stmt, col)
          text_ptr ? String.new(text_ptr) : nil
        else nil
        end

        row[name] = value
      end
      rows << row
    end

    LibSQLite3.sqlite3_finalize(stmt)
    rows
  end

  def last_insert_id : Int64
    LibSQLite3.sqlite3_last_insert_rowid(@db)
  end

  def changes : Int32
    LibSQLite3.sqlite3_changes(@db)
  end

  private def bind_param(stmt : LibSQLite3::STMT*, idx : Int32, value)
    SQLITE_TRANSIENT = Pointer(Void).new(UInt64::MAX)

    case value
    when Int32         then LibSQLite3.sqlite3_bind_int(stmt, idx, value)
    when Int64         then LibSQLite3.sqlite3_bind_int64(stmt, idx, value)
    when Float64       then LibSQLite3.sqlite3_bind_double(stmt, idx, value)
    when String        then LibSQLite3.sqlite3_bind_text(stmt, idx, value, value.size, SQLITE_TRANSIENT)
    when Nil           then LibSQLite3.sqlite3_bind_null(stmt, idx)
    end
  end

  private def error_message : String
    msg_ptr = LibSQLite3.sqlite3_errmsg(@db)
    msg_ptr ? String.new(msg_ptr) : "Unknown error"
  end
end

# ใช้งาน
db = SQLiteDB.new(":memory:")

db.exec(<<-SQL)
  CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name TEXT NOT NULL,
    email TEXT UNIQUE NOT NULL,
    age INTEGER
  )
SQL

db.exec("INSERT INTO users (name, email, age) VALUES ('Alice', 'alice@example.com', 30)")
db.exec("INSERT INTO users (name, email, age) VALUES ('Bob', 'bob@example.com', 25)")

puts "Last insert ID: #{db.last_insert_id}"

rows = db.query("SELECT * FROM users WHERE age > ?", 20)
rows.each do |row|
  puts "#{row["id"]}: #{row["name"]} (#{row["age"]})"
end

db.close
```

## Linking กับ libpng

```crystal
@[Link("png")]
lib LibPNG
  PNG_LIBPNG_VER_STRING = "1.6.40"
  PNG_COLOR_TYPE_RGB  = 2
  PNG_COLOR_TYPE_RGBA = 6
  PNG_INTERLACE_NONE  = 0
  PNG_COMPRESSION_TYPE_DEFAULT = 0
  PNG_FILTER_TYPE_DEFAULT = 0

  PNG_STRUCT = Void
  PNG_INFO   = Void

  fun png_create_read_struct(user_png_ver : UInt8*, error_ptr : Void*, error_fn : Void*, warn_fn : Void*) : PNG_STRUCT*
  fun png_create_write_struct(user_png_ver : UInt8*, error_ptr : Void*, error_fn : Void*, warn_fn : Void*) : PNG_STRUCT*
  fun png_create_info_struct(png_ptr : PNG_STRUCT*) : PNG_INFO*
  fun png_destroy_read_struct(png_ptr_ptr : PNG_STRUCT**, info_ptr_ptr : PNG_INFO**, end_info_ptr_ptr : PNG_INFO**) : Void
  fun png_destroy_write_struct(png_ptr_ptr : PNG_STRUCT**, info_ptr_ptr : PNG_INFO**) : Void
  fun png_init_io(png_ptr : PNG_STRUCT*, fp : Void*) : Void
  fun png_read_info(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*) : Void
  fun png_write_info(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*) : Void
  fun png_get_IHDR(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*, width : UInt32*, height : UInt32*, bit_depth : Int32*, color_type : Int32*, interlace_method : Int32*, compression_method : Int32*, filter_method : Int32*) : UInt32
  fun png_set_IHDR(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*, width : UInt32, height : UInt32, bit_depth : Int32, color_type : Int32, interlace_method : Int32, compression_method : Int32, filter_method : Int32) : Void
  fun png_read_image(png_ptr : PNG_STRUCT*, image : UInt8**) : Void
  fun png_write_image(png_ptr : PNG_STRUCT*, image : UInt8**) : Void
  fun png_write_end(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*) : Void
  fun png_get_rowbytes(png_ptr : PNG_STRUCT*, info_ptr : PNG_INFO*) : LibC::SizeT
end
```

## แบบฝึกหัด

1. สร้าง Crystal wrapper สำหรับ `libsodium` ครอบคลุม box, secretbox, sign
2. Wrap `libjpeg-turbo` เพื่อ decode JPEG ภาพ
3. สร้าง binding สำหรับ `libyaml` เพื่อ parse YAML ด้วย C
4. Link กับ `libpcre2` สำหรับ regex ที่เร็วกว่า Ruby-style

## สรุป

External Libraries ใน Crystal:
- **@[Link]**: ระบุ library ที่ต้อง link
- **pkg-config**: หา flags อัตโนมัติจาก system
- **OpenSSL**: crypto operations, SSL/TLS
- **SQLite**: embedded database
- **libpng**: PNG image processing
- **Custom wrappers**: สร้าง Crystal API ที่ idiomatic ห่อ C library

Crystal's C FFI ไม่มี overhead ทำให้ performance เท่ากับการเขียน C โดยตรง
