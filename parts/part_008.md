# Part 008: Constants และ Literals

## Constants

```crystal
# Constants ใน Crystal เริ่มต้นด้วยตัวพิมพ์ใหญ่
MAX_SIZE = 100
PI = 3.14159265358979
DEFAULT_TIMEOUT = 30.0
APP_NAME = "Crystal App"

# Constants ใน Module/Class
module Config
  HOST = "localhost"
  PORT = 8080
  MAX_CONNECTIONS = 100
  
  DATABASE_URL = "postgresql://localhost/mydb"
end

puts Config::HOST  # localhost
puts Config::PORT  # 8080

# Nested constants
module App
  VERSION = "1.0.0"
  
  module Database
    ADAPTER = "postgresql"
    POOL_SIZE = 5
  end
  
  module Cache
    TTL = 3600
    MAX_SIZE = 1000
  end
end

puts App::VERSION                 # 1.0.0
puts App::Database::ADAPTER      # postgresql
puts App::Cache::TTL             # 3600

# Constants คือ compile-time values
puts PI * 2  # 6.283185307179586

# เปลี่ยน constant ได้ (แต่จะได้ warning)
MAX_SIZE = 200  # Warning: already initialized constant MAX_SIZE
```

## Integer Literals

```crystal
# Decimal
decimal = 42
negative = -100
large = 1_000_000    # underscore เป็น separator
big = 9_999_999_999

# Hexadecimal (0x prefix)
hex1 = 0xFF          # 255
hex2 = 0xDEAD        # 57005
hex3 = 0xDEAD_BEEF   # 3735928559
hex4 = 0x1A2B3C4D    # 439,041,869

# Octal (0o prefix)
oct1 = 0o777         # 511
oct2 = 0o644         # 420
oct3 = 0o755         # 493

# Binary (0b prefix)
bin1 = 0b1111_0000   # 240
bin2 = 0b1010_1010   # 170
bin3 = 0b0000_0001   # 1

# Type suffixes
i8_val   = 127_i8
i16_val  = 32767_i16
i32_val  = 42_i32     # default
i64_val  = 1000000_i64
i128_val = 123456789_i128

u8_val   = 255_u8
u16_val  = 65535_u16
u32_val  = 1000_u32
u64_val  = 9999999_u64

# Special integer methods
puts 42.to_s       # "42"
puts 42.to_s(2)    # "101010" (binary)
puts 42.to_s(8)    # "52" (octal)
puts 42.to_s(16)   # "2a" (hex)
puts 42.to_s(36)   # "16" (base 36)

# Bitwise representation
puts 255.to_s(2).rjust(8, '0')  # "11111111"
puts 170.to_s(2).rjust(8, '0')  # "10101010"
```

## Float Literals

```crystal
# Basic floats
f1 = 3.14
f2 = 0.5
f3 = -2.7
f4 = 100.0

# Scientific notation
sci1 = 1.5e10    # 15,000,000,000.0
sci2 = 2.5e-3    # 0.0025
sci3 = 1.0E+6    # 1,000,000.0

# Underscore separator
big_float = 1_234_567.89
precise = 3.141_592_653

# Type suffixes
f32_val = 3.14_f32   # Float32
f64_val = 3.14_f64   # Float64 (default)
f64_val2 = 3.14      # Also Float64

# Special float values
puts Float64::INFINITY         # Infinity
puts -Float64::INFINITY        # -Infinity
puts Float64::NAN              # NaN
puts Float64::MAX              # 1.7976931348623157e+308
puts Float64::MIN_POSITIVE     # 2.2250738585072014e-308
puts Float64::EPSILON          # 2.220446049250313e-16

# Float checks
inf = Float64::INFINITY
nan = Float64::NAN

puts inf.infinite?   # 1 (positive infinity)
puts (-inf).infinite?  # -1 (negative infinity)
puts 1.0.infinite?   # nil
puts nan.nan?        # true
puts 1.0.nan?        # false
puts 1.0.finite?     # true
puts inf.finite?     # false
```

## String Literals

```crystal
# Double-quoted strings (with interpolation and escapes)
s1 = "Hello, World!"
s2 = "Name: #{name}"     # interpolation
s3 = "Tab:\tNewline:\n"  # escape sequences

# Single-quoted strings DON'T EXIST in Crystal
# 'A' is a Char, not a String!
c = 'A'   # Char
# s = 'hello'  # Error!

# Escape sequences
puts "Newline: \n"
puts "Tab: \t"
puts "Quote: \""
puts "Backslash: \\"
puts "Null: \0"
puts "Bell: \a"
puts "Backspace: \b"
puts "Form feed: \f"
puts "Carriage return: \r"
puts "Vertical tab: \v"
puts "Unicode: A"    # 'A'
puts "Unicode: \u{1F600}" # '😀' (emoji)

# Heredoc
text = <<-TEXT
  This is a heredoc.
  Indentation is stripped to match the closing marker.
  Variables: #{1 + 1}
  TEXT

puts text
# "This is a heredoc.\nIndentation is stripped...\nVariables: 2\n"

# Heredoc without interpolation
raw_text = <<-'TEXT'
  No #{interpolation} here.
  Backslashes: \n \t are literal.
  TEXT

puts raw_text

# Alternative string literals
# %(...) - same as double-quoted string
s = %(Hello, #{name}!)
s = %(It's easy to use "quotes" here)

# %q(...) - no interpolation, no escape
s = %q(Hello #{name})  # literal "Hello #{name}"

# %w(...) - array of strings
words = %w(apple banana cherry)  # ["apple", "banana", "cherry"]

# %i(...) - array of symbols
symbols = %i(foo bar baz)  # [:foo, :bar, :baz]

# String with special chars (easier quoting)
s = "He said \"hello\" and she said \"world\""
s = %(He said "hello" and she said "world")  # easier!
```

## Char Literals

```crystal
# Single character
c1 = 'A'    # uppercase A
c2 = 'z'    # lowercase z
c3 = '5'    # digit 5
c4 = ' '    # space
c5 = '!'    # exclamation

# Special characters
newline = '\n'
tab = '\t'
quote = '\''
backslash = '\\'

# Unicode characters
heart = '♥'   # ♥
smiley = '\u{1F600}'  # 😀
thai_ko = 'ก'    # ก

# Char methods
puts 'A'.ord       # 65 (Unicode code point)
puts 65.chr        # 'A'
puts 'A'.upcase    # 'A'
puts 'a'.upcase    # 'A'
puts 'A'.downcase  # 'a'
puts 'a'.to_i      # 97 (ord value)
puts '5'.to_i      # 5 (digit to integer)
puts 'a'.alpha?    # true
puts '5'.number?   # true (digit)
puts ' '.whitespace? # true
puts 'A'.ascii?    # true
puts '♥'.ascii?   # false

# Char comparison
puts 'a' < 'b'  # true
puts 'A' < 'a'  # true (uppercase has smaller ord)
puts 'a'.ord    # 97
puts 'A'.ord    # 65
```

## Symbol Literals

```crystal
# Symbol - immutable identifier
sym1 = :hello
sym2 = :world
sym3 = :my_method
sym4 = :with_underscore
sym5 = :"with spaces"  # quoted symbol
sym6 = :"complex-symbol-123"

# Symbol type
puts sym1.class  # Symbol
puts sym1.to_s   # "hello"
puts "hello".to_sym  # hello

# Symbols are unique (same symbol = same object)
puts :foo.object_id == :foo.object_id  # true
puts :foo.object_id == :bar.object_id  # false

# Symbol use cases
# 1. Hash keys (common pattern)
config = {
  host: "localhost",
  port: 8080,
  debug: false
}

puts config[:host]   # localhost
puts config[:port]   # 8080

# 2. Method references
puts [1, 2, 3].map(&method(:puts))
puts ["hello", "world"].map(&:upcase)  # ["HELLO", "WORLD"]

# 3. Enum-like usage
STATUS = {
  pending: 0,
  active: 1,
  inactive: 2,
  deleted: 3
}

def status_name(code : Int32) : Symbol?
  STATUS.find { |k, v| v == code }.try(&.[0])
end

# 4. Compare (faster than String comparison)
color = :red
case color
when :red   then puts "Red"
when :green then puts "Green"
when :blue  then puts "Blue"
end
```

## Nil Literal

```crystal
# nil is the only value of type Nil
n = nil
puts n.class   # Nil
puts n.nil?    # true
puts n.to_s    # ""
puts n.inspect # "nil"

# Nil in conditions
if nil
  puts "never printed"  # nil is falsy
end

unless nil
  puts "always printed"  # nil is falsy
end

# Nil safe operations
name : String? = nil
puts name&.upcase     # nil (no error)
puts name.try(&.upcase)  # nil (same thing)

# Nil coalescing
display = name || "Anonymous"  # "Anonymous"
count = nil || 0               # 0

# Check for nil
puts name.nil?        # true
puts name.is_a?(Nil)  # true
puts typeof(name)     # String | Nil

# Not nil assertion
# name.not_nil!  # Would raise NilAssertionError
```

## Array Literals

```crystal
# Empty arrays need type annotation
empty = [] of Int32
empty2 : Array(Int32) = []

# Type inferred from values
ints = [1, 2, 3, 4, 5]       # Array(Int32)
floats = [1.0, 2.0, 3.0]     # Array(Float64)
strings = ["a", "b", "c"]    # Array(String)
bools = [true, false, true]  # Array(Bool)

# Mixed type creates union
mixed = [1, "two", 3.0]  # Array(Int32 | String | Float64)

# Nested arrays
matrix = [[1, 2], [3, 4], [5, 6]]  # Array(Array(Int32))

# Word array literal
words = %w[apple banana cherry]  # ["apple", "banana", "cherry"]

# Array with same value
zeros = Array.new(5, 0)           # [0, 0, 0, 0, 0]
nils = Array(String?).new(3)      # [nil, nil, nil]
computed = Array.new(5) { |i| i * 2 }  # [0, 2, 4, 6, 8]

# Range to array
range_arr = (1..5).to_a          # [1, 2, 3, 4, 5]
char_arr = ('a'..'f').to_a       # ['a', 'b', 'c', 'd', 'e', 'f']
```

## Hash Literals

```crystal
# Basic hash (type inferred)
h1 = {"key" => "value"}          # Hash(String, String)
h2 = {1 => "one", 2 => "two"}   # Hash(Int32, String)
h3 = {:foo => 1, :bar => 2}      # Hash(Symbol, Int32)

# Symbol key shorthand
config = {host: "localhost", port: 8080}  # Hash(Symbol, String | Int32)

# Empty hash needs type annotation
empty = {} of String => Int32
empty2 : Hash(String, Int32) = {}

# Nested hash
nested = {
  "user" => {
    "name" => "Alice",
    "age" => "25"
  },
  "settings" => {
    "theme" => "dark"
  }
}

# Array of hashes
users = [
  {id: 1, name: "Alice"},
  {id: 2, name: "Bob"}
]
```

## Tuple Literals

```crystal
# Tuple - fixed-size, typed sequence
t1 = {1, 2, 3}              # Tuple(Int32, Int32, Int32)
t2 = {"hello", 42, true}    # Tuple(String, Int32, Bool)
t3 = {1, "two", 3.0, false} # Tuple(Int32, String, Float64, Bool)

# Named tuple
point = {x: 1.0, y: 2.0}   # NamedTuple(x: Float64, y: Float64)
user = {name: "Alice", age: 25, active: true}

# Access by index
puts t1[0]  # 1
puts t1[1]  # 2

# Access by name (NamedTuple)
puts point[:x]   # 1.0
puts user[:name] # "Alice"

# Destructuring
a, b, c = {1, 2, 3}
puts a  # 1
puts b  # 2
puts c  # 3

# Tuple types
t : Tuple(Int32, String, Bool) = {42, "hello", true}
puts typeof(t)   # Tuple(Int32, String, Bool)
puts t[0].class  # Int32
puts t[1].class  # String
```

## Range Literals

```crystal
# Inclusive range (.. includes both ends)
r1 = 1..10    # 1 to 10 inclusive
r2 = 'a'..'z' # 'a' to 'z' inclusive

# Exclusive range (... excludes end)
r3 = 1...10   # 1 to 9 (excludes 10)
r4 = 0...array.size  # common pattern for array indexing

# Beginless range (..end)
r5 = ..10     # -∞ to 10
r6 = ...'z'   # -∞ to 'y'

# Endless range (begin..)
r7 = 1..      # 1 to +∞
r8 = 'a'..    # 'a' to +∞

# Range operations
(1..10).each { |n| print "#{n} " }
puts ""

puts (1..10).include?(5)   # true
puts (1..10).include?(11)  # false
puts (1...10).include?(10) # false (exclusive)

puts (1..10).min   # 1
puts (1..10).max   # 10
puts (1..10).sum   # 55
puts (1..10).to_a.inspect  # [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Step
(0..20).step(5) { |n| print "#{n} " }  # 0 5 10 15 20
puts ""

# Sample from range
puts (1..100).sample  # random number 1-100

# String ranges
("a".."f").each { |s| print "#{s} " }  # a b c d e f
puts ""
```

## Regex Literals

```crystal
# Regex literal
r1 = /hello/          # basic
r2 = /hello/i         # case insensitive
r3 = /hello/m         # multiline
r4 = /hello/x         # extended (allows whitespace/comments)

# Named captures
r5 = /(?<year>\d{4})-(?<month>\d{2})-(?<day>\d{2})/

# Using regex
if "Hello, World" =~ /hello/i
  puts "Match found"
end

m = "2024-01-15".match(r5)
if m
  puts m["year"]   # 2024
  puts m["month"]  # 01
  puts m["day"]    # 15
end

# Pattern constants
EMAIL_REGEX = /\A[\w.+\-]+@[a-z\d\-.]+\.[a-z]+\z/i
PHONE_REGEX = /\A\+?[\d\s\-\(\)]{10,15}\z/
URL_REGEX   = /\Ahttps?:\/\/[\w\-.]+\.[a-z]+(\/.*)?/i

# Validate
puts "test@example.com".matches?(EMAIL_REGEX)  # true
puts "not-email".matches?(EMAIL_REGEX)          # false
```

## Practical: Configuration with Typed Constants

```crystal
# Typed constants for application configuration

module App
  # Application info
  NAME    = "Crystal Web App"
  VERSION = "1.0.0"
  
  # Server settings
  module Server
    HOST = ENV.fetch("HOST", "0.0.0.0")
    PORT = ENV.fetch("PORT", "8080").to_i
    WORKERS = ENV.fetch("WORKERS", "4").to_i
    
    # Timeouts in seconds
    READ_TIMEOUT  = 30
    WRITE_TIMEOUT = 30
    IDLE_TIMEOUT  = 60
  end
  
  # Database settings
  module Database
    URL = ENV.fetch("DATABASE_URL", "postgresql://localhost/myapp")
    POOL_SIZE = ENV.fetch("DB_POOL", "10").to_i
    TIMEOUT = 5.0
    
    MAX_RETRIES = 3
    RETRY_DELAY = 0.5  # seconds
  end
  
  # Cache settings
  module Cache
    TTL = 3600       # 1 hour in seconds
    MAX_SIZE = 10000 # max items
    
    KEY_PREFIX = "app:"
  end
  
  # Security
  module Security
    MIN_PASSWORD_LENGTH = 8
    MAX_LOGIN_ATTEMPTS = 5
    SESSION_TIMEOUT = 86400  # 24 hours
    
    ALLOWED_ORIGINS = %w[
      http://localhost:3000
      https://app.example.com
    ]
  end
  
  # Validation patterns
  module Patterns
    EMAIL    = /\A[\w.+\-]+@[a-z\d\-.]+\.[a-z]+\z/i
    PHONE    = /\A\+?[\d\s\-\(\)]{10,15}\z/
    USERNAME = /\A[a-z][a-z0-9_]{2,19}\z/
    URL      = /\Ahttps?:\/\/[\w\-.]+\.[a-z]+(\/.*)?/i
  end
end

# Usage
puts "#{App::NAME} v#{App::VERSION}"
puts "Server: #{App::Server::HOST}:#{App::Server::PORT}"
puts "DB pool: #{App::Database::POOL_SIZE}"
puts "Cache TTL: #{App::Cache::TTL}s"
puts "Session: #{App::Security::SESSION_TIMEOUT / 3600}h"
```

---

## สรุป Part 008

ในบทนี้เราได้เรียนรู้:

1. **Constants**: SCREAMING_SNAKE_CASE, module-scoped constants
2. **Integer literals**: decimal, hex (0x), octal (0o), binary (0b), type suffixes
3. **Float literals**: decimal, scientific notation, type suffixes, special values
4. **String literals**: double-quoted, heredoc, %(...), %q(...), escape sequences
5. **Char literals**: single character, escape, Unicode
6. **Symbol literals**: :name, :"complex name"
7. **Nil literal**: nil (only falsy with false)
8. **Array literals**: [], %w[], Array.new
9. **Hash literals**: {"key" => val}, {key: val}
10. **Tuple literals**: {a, b, c}, {x: 1, y: 2}
11. **Range literals**: a..b, a...b, beginless/endless
12. **Regex literals**: /pattern/flags

---

## ขั้นตอนต่อไป

ไปที่ [Part 009](part_009.md) เพื่อเรียนรู้:
- Nil และ Union Types เชิงลึก
- Nullable types
- Nil safety patterns
- Optional chaining
