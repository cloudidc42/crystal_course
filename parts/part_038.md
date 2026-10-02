# Part 38: Modules

## บทนำ

Module ใน Crystal เป็นเครื่องมือที่ใช้สำหรับ:
1. **Namespacing** - จัดกลุ่มโค้ดเพื่อหลีกเลี่ยงชื่อที่ชนกัน
2. **Mixins** - แบ่ง behavior ให้ classes ต่างๆ ใช้ร่วมกัน
3. **Module methods** - ฟังก์ชัน utility ที่ไม่ต้องการ instance

---

## 1. Module Definition พื้นฐาน

### 1.1 นิยาม Module

```crystal
# Module ง่ายๆ - เหมือน namespace
module Greeter
  def greet(name : String)
    "สวัสดี, #{name}!"
  end

  def farewell(name : String)
    "ลาก่อน, #{name}!"
  end
end

# Module ไม่สามารถ instantiate ได้โดยตรง
# Greeter.new  # Error!

# แต่สามารถ include ใน class ได้
class Person
  include Greeter

  def initialize(@name : String)
  end

  def introduce
    puts greet(@name)
  end
end

p = Person.new("สมชาย")
p.introduce  # => สวัสดี, สมชาย!
puts p.farewell("สมหญิง")  # => ลาก่อน, สมหญิง!
```

### 1.2 Module Body

```crystal
module MathHelpers
  # Constants
  PI = 3.14159265358979
  E = 2.71828182845905
  GOLDEN_RATIO = 1.61803398874989

  # Module-level methods
  def self.circle_area(radius : Float64) : Float64
    PI * radius ** 2
  end

  def self.factorial(n : Int32) : Int64
    return 1_i64 if n <= 1
    (2..n).reduce(1_i64) { |acc, i| acc * i }
  end

  def self.fibonacci(n : Int32) : Int32
    return n if n <= 1
    a, b = 0, 1
    (n - 1).times { a, b = b, a + b }
    b
  end
end

# เรียกใช้ผ่าน module name
puts MathHelpers.circle_area(5.0).round(2)  # => 78.54
puts MathHelpers.factorial(10)               # => 3628800
puts MathHelpers.fibonacci(10)               # => 55
puts MathHelpers::PI                         # => 3.14159265358979
```

---

## 2. Module Methods

### 2.1 Module Methods ด้วย self

```crystal
module StringUtils
  def self.capitalize_words(text : String) : String
    text.split.map { |word|
      word.empty? ? word : word[0..0].upcase + word[1..]
    }.join(" ")
  end

  def self.snake_case(text : String) : String
    text
      .gsub(/([A-Z]+)([A-Z][a-z])/, "\\1_\\2")
      .gsub(/([a-z\d])([A-Z])/, "\\1_\\2")
      .downcase
  end

  def self.camel_case(text : String) : String
    text.split("_").map_with_index { |word, i|
      i == 0 ? word : word.capitalize
    }.join
  end

  def self.truncate(text : String, limit : Int32, ellipsis : String = "...") : String
    return text if text.size <= limit
    text[0, limit - ellipsis.size] + ellipsis
  end

  def self.word_count(text : String) : Int32
    text.split.size
  end
end

puts StringUtils.capitalize_words("hello world from crystal")
# => Hello World From Crystal

puts StringUtils.snake_case("HelloWorldFromCrystal")
# => hello_world_from_crystal

puts StringUtils.camel_case("hello_world_from_crystal")
# => helloWorldFromCrystal

puts StringUtils.truncate("ประโยคที่ยาวมากๆ ไม่ควรแสดงทั้งหมด", 20)
# => ประโยคที่ยาวมากๆ ไม่ควรแ...
```

### 2.2 Module กับ Instance Methods

```crystal
module Printable
  # Instance methods (เมื่อ include จะกลายเป็นของ class)
  def print_info
    puts to_s  # ใช้ to_s ของ class ที่ include
  end

  def print_separator
    puts "-" * 40
  end

  # Abstract-like method ที่ class ต้อง implement
  def to_s : String
    raise "#{self.class.name} ต้อง implement to_s"
  end
end

class Product
  include Printable

  getter name : String
  getter price : Float64

  def initialize(@name : String, @price : Float64)
  end

  def to_s : String
    "#{@name}: ฿#{@price}"
  end
end

p = Product.new("แล็ปท็อป", 25000.0)
p.print_separator
p.print_info
p.print_separator
```

---

## 3. Module Constants

### 3.1 Constants ใน Module

```crystal
module HTTP
  module Status
    OK = 200
    CREATED = 201
    NO_CONTENT = 204
    BAD_REQUEST = 400
    UNAUTHORIZED = 401
    FORBIDDEN = 403
    NOT_FOUND = 404
    INTERNAL_SERVER_ERROR = 500

    MESSAGES = {
      200 => "OK",
      201 => "Created",
      204 => "No Content",
      400 => "Bad Request",
      401 => "Unauthorized",
      403 => "Forbidden",
      404 => "Not Found",
      500 => "Internal Server Error",
    }

    def self.message_for(code : Int32) : String
      MESSAGES[code]? || "Unknown Status"
    end
  end

  module Methods
    GET = "GET"
    POST = "POST"
    PUT = "PUT"
    PATCH = "PATCH"
    DELETE = "DELETE"
    HEAD = "HEAD"
    OPTIONS = "OPTIONS"
  end
end

puts HTTP::Status::OK              # => 200
puts HTTP::Status::NOT_FOUND      # => 404
puts HTTP::Status.message_for(200) # => OK
puts HTTP::Status.message_for(404) # => Not Found
puts HTTP::Methods::GET            # => GET
```

---

## 4. Namespacing ด้วย Modules

### 4.1 หลีกเลี่ยงชื่อที่ชนกัน

```crystal
# สมมติมี User ในหลาย context
module Admin
  class User
    getter username : String
    getter role : String

    def initialize(@username : String)
      @role = "admin"
    end

    def to_s : String
      "Admin::User(#{@username})"
    end
  end
end

module Customer
  class User
    getter username : String
    getter email : String

    def initialize(@username : String, @email : String)
    end

    def to_s : String
      "Customer::User(#{@username})"
    end
  end
end

admin = Admin::User.new("admin1")
customer = Customer::User.new("customer1", "cust@example.com")

puts admin     # => Admin::User(admin1)
puts customer  # => Customer::User(customer1)
```

### 4.2 Nested Modules

```crystal
module App
  module Database
    module Migrations
      def self.run
        puts "Running migrations..."
      end

      def self.rollback
        puts "Rolling back..."
      end
    end

    module Connections
      MAX_POOL = 10

      def self.create(url : String)
        puts "Creating connection to: #{url}"
      end
    end
  end

  module API
    module V1
      module Users
        def self.list
          puts "GET /api/v1/users"
        end

        def self.create
          puts "POST /api/v1/users"
        end
      end
    end

    module V2
      module Users
        def self.list
          puts "GET /api/v2/users (with pagination)"
        end
      end
    end
  end
end

App::Database::Migrations.run
App::Database::Connections.create("postgresql://localhost/mydb")
App::API::V1::Users.list
App::API::V2::Users.list
puts App::Database::Connections::MAX_POOL  # => 10
```

### 4.3 require กับ Modules

```crystal
# ในโปรเจกต์จริง จะแยกไฟล์แล้วใช้ require
# my_app/models/user.cr
module MyApp
  module Models
    class User
      getter id : Int32
      getter name : String

      def initialize(@id : Int32, @name : String)
      end
    end
  end
end

# my_app/services/user_service.cr
module MyApp
  module Services
    class UserService
      def find(id : Int32) : MyApp::Models::User?
        # Implementation
        nil
      end
    end
  end
end

# ใช้ alias เพื่อความกระชับ
alias User = MyApp::Models::User
alias UserService = MyApp::Services::UserService
```

---

## 5. Module เป็น Namespace สำหรับ Utilities

### 5.1 Date/Time Utilities

```crystal
module DateUtils
  def self.days_between(from : Time, to : Time) : Int32
    (to - from).total_days.to_i.abs
  end

  def self.add_business_days(date : Time, days : Int32) : Time
    result = date
    added = 0
    direction = days < 0 ? -1 : 1
    days = days.abs

    while added < days
      result += direction.days
      # ข้าม weekend
      added += 1 unless result.day_of_week.saturday? || result.day_of_week.sunday?
    end
    result
  end

  def self.format_relative(time : Time) : String
    diff = Time.local - time
    case diff.total_seconds.abs.to_i
    when 0..59 then "เมื่อสักครู่"
    when 60..3599 then "#{(diff.total_minutes).to_i} นาทีที่แล้ว"
    when 3600..86399 then "#{(diff.total_hours).to_i} ชั่วโมงที่แล้ว"
    when 86400..2591999 then "#{(diff.total_days).to_i} วันที่แล้ว"
    else time.to_s("%d/%m/%Y")
    end
  end

  def self.quarters_in_year : Array(NamedTuple(start: Time, end: Time))
    year = Time.local.year
    [
      {start: Time.local(year, 1, 1), end: Time.local(year, 3, 31)},
      {start: Time.local(year, 4, 1), end: Time.local(year, 6, 30)},
      {start: Time.local(year, 7, 1), end: Time.local(year, 9, 30)},
      {start: Time.local(year, 10, 1), end: Time.local(year, 12, 31)},
    ]
  end
end

now = Time.local
past = now - 2.hours
puts DateUtils.format_relative(past)  # => 2 ชั่วโมงที่แล้ว
```

### 5.2 File Path Utilities

```crystal
module PathUtils
  SEPARATOR = "/"

  def self.join(*parts : String) : String
    parts.map { |p| p.strip("/") }.reject(&.empty?).join(SEPARATOR)
  end

  def self.extension(path : String) : String
    file = path.split(SEPARATOR).last
    dot_idx = file.rindex('.')
    dot_idx ? file[dot_idx..] : ""
  end

  def self.basename(path : String) : String
    path.split(SEPARATOR).last
  end

  def self.dirname(path : String) : String
    parts = path.split(SEPARATOR)
    parts[0..-2].join(SEPARATOR)
  end

  def self.normalize(path : String) : String
    parts = path.split(SEPARATOR)
    result = [] of String
    parts.each do |part|
      case part
      when ".", "" then next
      when ".." then result.pop? rescue nil
      else result << part
      end
    end
    SEPARATOR + result.join(SEPARATOR)
  end
end

puts PathUtils.join("app", "models", "user.cr")
# => app/models/user.cr

puts PathUtils.extension("document.pdf")  # => .pdf
puts PathUtils.basename("/home/user/file.txt")  # => file.txt
puts PathUtils.dirname("/home/user/file.txt")   # => /home/user
puts PathUtils.normalize("/app/../config/./settings.yml")
# => /config/settings.yml
```

---

## 6. Module เป็น Mixin (Preview)

### 6.1 Module ใน Include

```crystal
module Serializable
  def to_json : String
    fields = instance_vars_to_hash
    "{#{fields.map { |k, v| "\"#{k}\": \"#{v}\"" }.join(", ")}}"
  end

  private def instance_vars_to_hash : Hash(String, String)
    # Simplified - ในทางปฏิบัติจะซับซ้อนกว่า
    {} of String => String
  end
end

module Timestampable
  def created_at : Time
    @created_at ||= Time.local
  end

  def updated_at : Time
    @updated_at ||= Time.local
  end

  def touch
    @updated_at = Time.local
  end
end

class Article
  include Timestampable

  getter title : String
  getter content : String

  def initialize(@title : String, @content : String)
    @created_at = Time.local
    @updated_at = Time.local
  end
end

article = Article.new("Crystal Programming", "Crystal เป็นภาษาที่น่าสนใจ...")
puts article.created_at
puts article.updated_at
article.touch
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Math Module

```crystal
module Geometry
  module TwoD
    def self.circle_area(radius : Float64) : Float64
      Math::PI * radius ** 2
    end

    def self.rectangle_area(width : Float64, height : Float64) : Float64
      width * height
    end

    def self.triangle_area(base : Float64, height : Float64) : Float64
      0.5 * base * height
    end
  end

  module ThreeD
    def self.sphere_volume(radius : Float64) : Float64
      (4.0 / 3.0) * Math::PI * radius ** 3
    end

    def self.cube_volume(side : Float64) : Float64
      side ** 3
    end

    def self.cylinder_volume(radius : Float64, height : Float64) : Float64
      Math::PI * radius ** 2 * height
    end
  end
end

puts "วงกลม r=5: #{Geometry::TwoD.circle_area(5.0).round(2)}"
puts "สี่เหลี่ยม 4x6: #{Geometry::TwoD.rectangle_area(4.0, 6.0)}"
puts "ทรงกลม r=3: #{Geometry::ThreeD.sphere_volume(3.0).round(2)}"
puts "ทรงกระบอก r=2 h=5: #{Geometry::ThreeD.cylinder_volume(2.0, 5.0).round(2)}"
```

### แบบฝึกหัดที่ 2: Validation Module

```crystal
module Validators
  module String
    def self.email?(value : ::String) : Bool
      value.matches?(/^[^\s@]+@[^\s@]+\.[^\s@]+$/)
    end

    def self.phone_thai?(value : ::String) : Bool
      value.matches?(/^0[0-9]{9}$/)
    end

    def self.url?(value : ::String) : Bool
      value.matches?(/^https?:\/\/.+/)
    end

    def self.thai_id?(value : ::String) : Bool
      return false unless value.matches?(/^\d{13}$/)
      digits = value.chars.map(&.to_i)
      sum = digits[0..11].each_with_index.sum { |d, i| d * (13 - i) }
      check = (11 - (sum % 11)) % 10
      check == digits[12]
    end
  end

  module Number
    def self.positive?(n : Float64) : Bool
      n > 0
    end

    def self.in_range?(n : Float64, min : Float64, max : Float64) : Bool
      n >= min && n <= max
    end

    def self.percentage?(n : Float64) : Bool
      in_range?(n, 0.0, 100.0)
    end
  end
end

puts Validators::String.email?("user@example.com")    # => true
puts Validators::String.email?("invalid")             # => false
puts Validators::String.phone_thai?("0812345678")     # => true
puts Validators::Number.in_range?(75.0, 0.0, 100.0)  # => true
puts Validators::Number.percentage?(101.0)            # => false
```

### แบบฝึกหัดที่ 3: Configuration Module

```crystal
module Config
  module Database
    HOST = ENV["DB_HOST"]? || "localhost"
    PORT = (ENV["DB_PORT"]? || "5432").to_i
    NAME = ENV["DB_NAME"]? || "development"

    def self.url : String
      "postgresql://#{HOST}:#{PORT}/#{NAME}"
    end
  end

  module Redis
    HOST = ENV["REDIS_HOST"]? || "localhost"
    PORT = (ENV["REDIS_PORT"]? || "6379").to_i

    def self.url : String
      "redis://#{HOST}:#{PORT}"
    end
  end

  module App
    NAME = "My Crystal App"
    VERSION = "1.0.0"
    ENV_NAME = ENV["CRYSTAL_ENV"]? || "development"

    def self.production? : Bool
      ENV_NAME == "production"
    end

    def self.development? : Bool
      ENV_NAME == "development"
    end
  end

  def self.all_settings
    puts "=== App Configuration ==="
    puts "App: #{App::NAME} v#{App::VERSION}"
    puts "Env: #{App::ENV_NAME}"
    puts "Database: #{Database.url}"
    puts "Redis: #{Redis.url}"
  end
end

Config.all_settings
```

---

## สรุป

Modules ใน Crystal มีบทบาทหลายอย่าง:

| การใช้งาน | คำอธิบาย |
|---------|---------|
| Namespace | จัดกลุ่มโค้ด หลีกเลี่ยงชื่อชนกัน |
| Module methods | Utility functions ที่ไม่ต้องการ instance |
| Constants | ค่าคงที่ที่จัดกลุ่ม |
| Mixin (include) | แบ่ง behavior ให้ classes ใช้ร่วมกัน |

**ความแตกต่างจาก Class:**
- ไม่สามารถ instantiate ได้
- ไม่มี constructor (initialize)
- ไม่มี inheritance chain (แต่ใช้ include ได้)
- Module methods ใช้ `def self.method_name`

**Best Practices:**
- ใช้ Module สำหรับ utility functions ที่ไม่มี state
- ใช้ Nested modules สำหรับ hierarchical namespacing
- ตั้งชื่อ Module ให้อธิบายหน้าที่ชัดเจน (StringUtils, DateHelpers)
