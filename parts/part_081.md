# ตอนที่ 81: JSON ใน Crystal

## บทนำ

JSON (JavaScript Object Notation) เป็นรูปแบบการแลกเปลี่ยนข้อมูลที่นิยมใช้กันอย่างแพร่หลาย Crystal มี built-in support สำหรับ JSON ผ่าน `require "json"` ซึ่งมีความสามารถทั้งการ parse, serialize และ deserialize ข้อมูล JSON

---

## 1. การ require "json"

ก่อนใช้งาน JSON ใน Crystal ต้อง require library ก่อน:

```crystal
require "json"
```

---

## 2. JSON.parse - การแปลง JSON String เป็น Object

`JSON.parse` ใช้แปลง JSON string ให้เป็น `JSON::Any` object:

```crystal
require "json"

json_string = %({"name": "สมชาย", "age": 30, "city": "กรุงเทพ"})

data = JSON.parse(json_string)
puts data          # => {"name" => "สมชาย", "age" => 30, "city" => "กรุงเทพ"}
puts data.class    # => JSON::Any
```

### การ parse JSON Array:

```crystal
require "json"

json_array = %([1, 2, 3, "hello", true, null])
data = JSON.parse(json_array)

puts data          # => [1, 2, 3, "hello", true, nil]
puts data.class    # => JSON::Any
```

---

## 3. JSON::Any - ประเภทข้อมูลสำหรับ JSON

`JSON::Any` เป็น type ที่ใช้แทนค่า JSON ใดๆ ก็ได้ (string, number, boolean, null, array, object)

```crystal
require "json"

json_string = %({"name": "มานี", "scores": [95, 87, 92], "active": true})
data = JSON.parse(json_string)

# ตรวจสอบประเภทข้อมูล
puts data["name"].class    # => JSON::Any
puts data["scores"].class  # => JSON::Any
puts data["active"].class  # => JSON::Any
```

### การแปลง JSON::Any เป็น Crystal types:

```crystal
require "json"

json_string = %({"name": "มานี", "age": 25, "score": 98.5, "active": true})
data = JSON.parse(json_string)

# แปลงด้วย as_* methods
name   = data["name"].as_s    # String
age    = data["age"].as_i     # Int32
score  = data["score"].as_f   # Float64
active = data["active"].as_bool # Bool

puts "ชื่อ: #{name}"
puts "อายุ: #{age}"
puts "คะแนน: #{score}"
puts "สถานะ: #{active}"
```

### ตารางแสดง as_* methods:

| method    | Crystal type | ตัวอย่าง JSON value |
|-----------|-------------|---------------------|
| as_s      | String      | "hello"             |
| as_i      | Int32       | 42                  |
| as_i64    | Int64       | 9999999999          |
| as_f      | Float64     | 3.14                |
| as_f32    | Float32     | 3.14                |
| as_bool   | Bool        | true                |
| as_nil    | Nil         | null                |
| as_a      | Array(Any)  | [1, 2, 3]           |
| as_h      | Hash        | {"key": "value"}    |

---

## 4. การเข้าถึงค่าด้วย []

```crystal
require "json"

json = %({
  "user": {
    "name": "สมศรี",
    "age": 28,
    "address": {
      "city": "เชียงใหม่",
      "zip": "50000"
    },
    "hobbies": ["อ่านหนังสือ", "ปั่นจักรยาน", "ทำอาหาร"]
  }
})

data = JSON.parse(json)

# เข้าถึงข้อมูลซ้อนกัน
name = data["user"]["name"].as_s
city = data["user"]["address"]["city"].as_s
first_hobby = data["user"]["hobbies"][0].as_s

puts "ชื่อ: #{name}"
puts "เมือง: #{city}"
puts "งานอดิเรกแรก: #{first_hobby}"
```

### การใช้ []? สำหรับ optional values:

```crystal
require "json"

json = %({"name": "สมชาย", "age": 30})
data = JSON.parse(json)

# []? จะ return nil แทนที่จะ raise exception ถ้าไม่มี key
email = data["email"]?
puts email.nil? ? "ไม่มีอีเมล" : email.as_s
```

---

## 5. to_json - การแปลง Crystal Object เป็น JSON String

```crystal
require "json"

# การแปลงประเภทพื้นฐาน
puts 42.to_json          # => 42
puts "hello".to_json     # => "hello"
puts true.to_json        # => true
puts nil.to_json         # => null

# Array
arr = [1, 2, 3, "สวัสดี"]
puts arr.to_json         # => [1,2,3,"สวัสดี"]

# Hash
hash = {"name" => "มานี", "age" => 25}
puts hash.to_json        # => {"name":"มานี","age":25}
```

---

## 6. from_json - การแปลง JSON String เป็น Crystal Type

```crystal
require "json"

# แปลง JSON array เป็น Array(Int32)
numbers = Array(Int32).from_json("[1, 2, 3, 4, 5]")
puts numbers.sum  # => 15
puts numbers.class  # => Array(Int32)

# แปลง JSON string เป็น String
name = String.from_json(%("สมชาย"))
puts name  # => สมชาย

# แปลง JSON number เป็น Int32
age = Int32.from_json("30")
puts age + 1  # => 31
```

---

## 7. JSON::Serializable - Annotation สำหรับ Struct/Class

`JSON::Serializable` ทำให้ struct หรือ class สามารถ serialize/deserialize ได้อัตโนมัติ:

```crystal
require "json"

struct Person
  include JSON::Serializable

  property name : String
  property age : Int32
  property email : String?
end

# from_json
json_str = %({"name": "สมชาย", "age": 30, "email": "somchai@example.com"})
person = Person.from_json(json_str)

puts person.name   # => สมชาย
puts person.age    # => 30
puts person.email  # => somchai@example.com

# to_json
puts person.to_json
# => {"name":"สมชาย","age":30,"email":"somchai@example.com"}
```

### ตัวอย่างเพิ่มเติมกับ Class:

```crystal
require "json"

class Product
  include JSON::Serializable

  property id : Int32
  property name : String
  property price : Float64
  property in_stock : Bool
  property tags : Array(String)

  def initialize(@id, @name, @price, @in_stock, @tags)
  end
end

product = Product.new(
  id: 1,
  name: "แล็ปท็อป",
  price: 35000.0,
  in_stock: true,
  tags: ["electronics", "computer", "laptop"]
)

json = product.to_json
puts json
# => {"id":1,"name":"แล็ปท็อป","price":35000.0,"in_stock":true,"tags":["electronics","computer","laptop"]}

# แปลงกลับ
restored = Product.from_json(json)
puts restored.name   # => แล็ปท็อป
puts restored.price  # => 35000.0
```

---

## 8. Property Mapping - การกำหนดชื่อ field ใน JSON

ใช้ `@[JSON::Field]` annotation เพื่อกำหนดชื่อ field ใน JSON ให้แตกต่างจาก property name:

```crystal
require "json"

struct UserProfile
  include JSON::Serializable

  # เปลี่ยนชื่อ field ใน JSON
  @[JSON::Field(key: "first_name")]
  property first_name : String

  @[JSON::Field(key: "last_name")]
  property last_name : String

  @[JSON::Field(key: "user_age")]
  property age : Int32

  # ไม่ใส่ใน JSON เลย
  @[JSON::Field(ignore: true)]
  property internal_id : Int32 = 0
end

json = %({"first_name": "สม", "last_name": "ชาย", "user_age": 30})
user = UserProfile.from_json(json)

puts user.first_name  # => สม
puts user.last_name   # => ชาย
puts user.age         # => 30
puts user.internal_id # => 0

puts user.to_json
# => {"first_name":"สม","last_name":"ชาย","user_age":30}
```

### ใช้ emit_null สำหรับ nil values:

```crystal
require "json"

struct Config
  include JSON::Serializable

  property name : String

  # โดยปกติ nil จะไม่ถูกใส่ใน JSON
  # emit_null: true จะบังคับให้ใส่ null
  @[JSON::Field(emit_null: true)]
  property description : String?
end

c = Config.new
c.name = "test"
c.description = nil

puts c.to_json
# => {"name":"test","description":null}
```

---

## 9. Custom Serialization/Deserialization

บางครั้งเราต้องการควบคุมกระบวนการ serialize/deserialize เอง:

```crystal
require "json"

struct Temperature
  property value : Float64
  property unit : String

  def initialize(@value, @unit)
  end

  # Custom to_json
  def to_json(json : JSON::Builder)
    json.object do
      json.field "celsius" do
        case @unit
        when "F"
          json.number ((@value - 32) * 5 / 9).round(2)
        when "K"
          json.number (@value - 273.15).round(2)
        else
          json.number @value
        end
      end
      json.field "unit", "C"
    end
  end

  # Custom from_json
  def self.from_json(parser : JSON::PullParser) : self
    value = 0.0
    unit = "C"
    parser.read_object do |key|
      case key
      when "value" then value = parser.read_float
      when "unit"  then unit = parser.read_string
      end
    end
    new(value, unit)
  end
end

temp = Temperature.new(100.0, "C")
puts temp.to_json
# => {"celsius":100.0,"unit":"C"}

temp_f = Temperature.new(212.0, "F")
puts temp_f.to_json
# => {"celsius":100.0,"unit":"C"}
```

---

## 10. การจัดการ nil - Nil Handling

```crystal
require "json"

struct UserData
  include JSON::Serializable

  property name : String
  property age : Int32?           # optional - อาจเป็น nil
  property bio : String?          # optional
  property score : Float64 = 0.0  # มีค่า default
end

# JSON ที่ไม่มีบาง field
json1 = %({"name": "สมชาย"})
user1 = UserData.from_json(json1)
puts user1.name   # => สมชาย
puts user1.age    # => nil (ไม่ได้ระบุ)
puts user1.score  # => 0.0 (ใช้ค่า default)

# JSON ที่มี null อย่างชัดเจน
json2 = %({"name": "มานี", "age": null, "bio": null})
user2 = UserData.from_json(json2)
puts user2.age.nil?  # => true
```

### การจัดการกับ nil ในการ access:

```crystal
require "json"

json = %({"user": {"name": "สมชาย"}, "product": null})
data = JSON.parse(json)

# ใช้ try เพื่อป้องกัน NilAssertionError
if product = data["product"]?
  puts product.as_s?
else
  puts "ไม่มีสินค้า"
end

# Safe navigation
name = data.dig?("user", "name").try(&.as_s)
puts name  # => สมชาย

missing = data.dig?("user", "email").try(&.as_s)
puts missing.nil? ? "ไม่มีอีเมล" : missing
```

---

## 11. Pretty Print - การแสดง JSON แบบอ่านง่าย

```crystal
require "json"

data = {
  "name" => "สมชาย",
  "age"  => 30,
  "address" => {
    "city"    => "กรุงเทพ",
    "country" => "ไทย"
  },
  "hobbies" => ["อ่านหนังสือ", "ฟังเพลง"]
}

# แบบปกติ (compact)
puts data.to_json
# => {"name":"สมชาย","age":30,"address":{"city":"กรุงเทพ","country":"ไทย"},"hobbies":["อ่านหนังสือ","ฟังเพลง"]}

# แบบ pretty print
puts data.to_pretty_json
# => {
#      "name": "สมชาย",
#      "age": 30,
#      "address": {
#        "city": "กรุงเทพ",
#        "country": "ไทย"
#      },
#      "hobbies": [
#        "อ่านหนังสือ",
#        "ฟังเพลง"
#      ]
#    }

# กำหนด indent
puts data.to_pretty_json("    ")  # 4 spaces indent
```

---

## 12. JSON.build - การสร้าง JSON แบบ Programmatic

`JSON.build` ช่วยสร้าง JSON string โดยไม่ต้องสร้าง Hash หรือ Array ก่อน:

```crystal
require "json"

json = JSON.build do |json|
  json.object do
    json.field "name", "มานี"
    json.field "age", 25

    json.field "scores" do
      json.array do
        json.number 95
        json.number 87
        json.number 92
      end
    end

    json.field "address" do
      json.object do
        json.field "city", "เชียงใหม่"
        json.field "zip", "50000"
      end
    end
  end
end

puts json
# => {"name":"มานี","age":25,"scores":[95,87,92],"address":{"city":"เชียงใหม่","zip":"50000"}}
```

### ตัวอย่างการสร้าง JSON จาก collection:

```crystal
require "json"

students = [
  {name: "สมชาย", grade: "A", score: 95},
  {name: "มานี", grade: "B+", score: 88},
  {name: "วิชัย", grade: "A+", score: 98},
]

json = JSON.build do |json|
  json.object do
    json.field "total", students.size
    json.field "students" do
      json.array do
        students.each do |s|
          json.object do
            json.field "name", s[:name]
            json.field "grade", s[:grade]
            json.field "score", s[:score]
          end
        end
      end
    end
  end
end

puts json
# {"total":3,"students":[{"name":"สมชาย","grade":"A","score":95},{"name":"มานี","grade":"B+","score":88},{"name":"วิชัย","grade":"A+","score":98}]}
```

---

## 13. Nested Objects - Object ซ้อนกัน

```crystal
require "json"

struct Address
  include JSON::Serializable

  property street : String
  property city : String
  property postal_code : String
end

struct ContactInfo
  include JSON::Serializable

  property email : String
  property phone : String?
end

struct Person
  include JSON::Serializable

  property name : String
  property age : Int32
  property address : Address
  property contact : ContactInfo
end

json = %({
  "name": "สมชาย ใจดี",
  "age": 35,
  "address": {
    "street": "123 ถนนสุขุมวิท",
    "city": "กรุงเทพมหานคร",
    "postal_code": "10110"
  },
  "contact": {
    "email": "somchai@example.com",
    "phone": "081-234-5678"
  }
})

person = Person.from_json(json)
puts person.name
puts person.address.city
puts person.contact.email

# แปลงกลับเป็น JSON
puts person.to_json
```

---

## 14. Arrays ใน JSON

```crystal
require "json"

struct Course
  include JSON::Serializable

  property id : Int32
  property name : String
  property tags : Array(String)
  property scores : Array(Float64)
  property metadata : Hash(String, String)?
end

json = %({
  "id": 1,
  "name": "Crystal Programming",
  "tags": ["programming", "systems", "compiled"],
  "scores": [4.8, 4.9, 4.7, 5.0],
  "metadata": {"level": "intermediate", "duration": "10 hours"}
})

course = Course.from_json(json)
puts course.name
puts course.tags.join(", ")
puts course.scores.sum / course.scores.size
puts course.metadata.try { |m| m["level"] }

# แปลงกลับ
puts course.to_json
```

### การ iterate ผ่าน JSON array:

```crystal
require "json"

json = %({
  "products": [
    {"id": 1, "name": "กาแฟ", "price": 80},
    {"id": 2, "name": "ชา", "price": 60},
    {"id": 3, "name": "น้ำผลไม้", "price": 90}
  ]
})

data = JSON.parse(json)
products = data["products"].as_a

products.each do |product|
  id    = product["id"].as_i
  name  = product["name"].as_s
  price = product["price"].as_i
  puts "#{id}. #{name} - #{price} บาท"
end

# หาสินค้าที่แพงที่สุด
most_expensive = products.max_by { |p| p["price"].as_i }
puts "แพงที่สุด: #{most_expensive["name"].as_s}"
```

---

## 15. Error Handling - การจัดการข้อผิดพลาด

```crystal
require "json"

# จัดการ JSON parse error
def safe_parse(json_string : String) : JSON::Any?
  JSON.parse(json_string)
rescue JSON::ParseException => e
  puts "JSON ไม่ถูกต้อง: #{e.message}"
  nil
end

valid_json   = %({"name": "สมชาย"})
invalid_json = %({name: สมชาย})  # ไม่ถูกต้อง

result1 = safe_parse(valid_json)
puts result1.try { |r| r["name"].as_s }  # => สมชาย

result2 = safe_parse(invalid_json)
puts result2.nil?  # => true
```

### จัดการ type mismatch:

```crystal
require "json"

json = %({"value": "not a number"})
data = JSON.parse(json)

begin
  number = data["value"].as_i
rescue TypeCastError => e
  puts "ไม่สามารถแปลงเป็นตัวเลขได้: #{e.message}"
end

# ใช้ as_i? แบบ safe
number = data["value"].as_i?
puts number.nil? ? "ไม่ใช่ตัวเลข" : number
```

### จัดการกับ key ที่ไม่มี:

```crystal
require "json"

json = %({"name": "มานี"})
data = JSON.parse(json)

# วิธีที่ 1: ใช้ []?
age = data["age"]?.try(&.as_i?)
puts age.nil? ? "ไม่มีอายุ" : age

# วิธีที่ 2: ตรวจสอบด้วย has_key? (Hash)
hash = data.as_h
if hash.has_key?("email")
  puts hash["email"].as_s
else
  puts "ไม่มีอีเมล"
end
```

---

## 16. JSON::PullParser - การอ่าน JSON แบบ Streaming

สำหรับ JSON ขนาดใหญ่ ใช้ `JSON::PullParser` เพื่อประหยัดหน่วยความจำ:

```crystal
require "json"

json = %([
  {"id": 1, "name": "สมชาย"},
  {"id": 2, "name": "มานี"},
  {"id": 3, "name": "วิชัย"}
])

parser = JSON::PullParser.new(json)
parser.read_array do
  parser.read_object do |key|
    case key
    when "id"
      id = parser.read_int
      print "ID: #{id}, "
    when "name"
      name = parser.read_string
      puts "ชื่อ: #{name}"
    end
  end
end
```

---

## 17. ตัวอย่างโปรแกรมสมบูรณ์ - API Response Handler

```crystal
require "json"

# จำลอง API Response สำหรับระบบห้องสมุด
struct Author
  include JSON::Serializable

  property id : Int32
  property name : String

  @[JSON::Field(key: "birth_year")]
  property birth_year : Int32?
end

struct Book
  include JSON::Serializable

  property id : Int32
  property title : String
  property author : Author
  property genres : Array(String)
  property rating : Float64
  property available : Bool

  @[JSON::Field(key: "published_year")]
  property published_year : Int32

  @[JSON::Field(ignore: true)]
  property checkout_count : Int32 = 0
end

struct LibraryResponse
  include JSON::Serializable

  property status : String
  property total : Int32
  property books : Array(Book)
end

api_response = %({
  "status": "success",
  "total": 3,
  "books": [
    {
      "id": 1,
      "title": "Crystal Programming Guide",
      "author": {"id": 101, "name": "John Doe", "birth_year": 1985},
      "genres": ["programming", "technology"],
      "rating": 4.8,
      "available": true,
      "published_year": 2023
    },
    {
      "id": 2,
      "title": "Advanced Crystal Patterns",
      "author": {"id": 102, "name": "Jane Smith", "birth_year": null},
      "genres": ["programming", "advanced"],
      "rating": 4.5,
      "available": false,
      "published_year": 2024
    }
  ]
})

library = LibraryResponse.from_json(api_response)

puts "สถานะ: #{library.status}"
puts "จำนวนหนังสือทั้งหมด: #{library.total}"
puts ""

library.books.each do |book|
  puts "📚 #{book.title}"
  puts "   ผู้เขียน: #{book.author.name}"
  puts "   ประเภท: #{book.genres.join(", ")}"
  puts "   คะแนน: #{book.rating}"
  puts "   สถานะ: #{book.available ? "ว่าง" : "ถูกยืม"}"
  puts "   ปีที่พิมพ์: #{book.published_year}"
  puts ""
end

# แปลงกลับเป็น JSON แบบ pretty
puts library.to_pretty_json
```

---

## 18. การทำงานกับ JSON ขนาดใหญ่

```crystal
require "json"

# สร้าง JSON ขนาดใหญ่แบบ streaming
output = String::Builder.new
builder = JSON::Builder.new(output)

builder.document do
  builder.object do
    builder.field "users" do
      builder.array do
        1000.times do |i|
          builder.object do
            builder.field "id", i + 1
            builder.field "name", "ผู้ใช้ #{i + 1}"
            builder.field "email", "user#{i + 1}@example.com"
          end
        end
      end
    end
  end
end

# แสดงขนาด (ไม่แสดงทั้งหมด)
json_result = output.to_s
puts "ขนาด JSON: #{json_result.bytesize} bytes"
puts "ตัวอย่าง 100 ตัวอักษรแรก: #{json_result[0..100]}..."
```

---

## 19. JSON Merge และ Update

```crystal
require "json"

# Merge สอง JSON objects
def merge_json(base : String, update : String) : String
  base_hash = JSON.parse(base).as_h
  update_hash = JSON.parse(update).as_h

  merged = base_hash.merge(update_hash)
  merged.to_json
end

base   = %({"name": "สมชาย", "age": 30, "city": "กรุงเทพ"})
update = %({"age": 31, "email": "somchai@example.com"})

result = merge_json(base, update)
puts result
# => {"name":"สมชาย","age":31,"city":"กรุงเทพ","email":"somchai@example.com"}
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Student Grade System
สร้าง struct `Student` ที่ include `JSON::Serializable` มี properties: name (String), grades (Array(Int32)), subject (String) และเพิ่ม method `average` ที่คำนวณค่าเฉลี่ย จากนั้น:
1. แปลง JSON string เป็น Student object
2. คำนวณค่าเฉลี่ย
3. แปลงกลับเป็น JSON

```crystal
# โค้ดเริ่มต้น:
require "json"

struct Student
  include JSON::Serializable
  # TODO: เพิ่ม properties
  # TODO: เพิ่ม method average
end

json = %({"name": "สมชาย", "subject": "คณิตศาสตร์", "grades": [85, 92, 78, 96, 88]})
# TODO: แปลงและแสดงผล
```

### แบบฝึกหัดที่ 2: Config File Reader
สร้างโปรแกรมที่อ่าน JSON configuration และแสดงข้อมูล:

```crystal
require "json"

config_json = %({
  "app": {
    "name": "MyApp",
    "version": "1.0.0",
    "debug": false
  },
  "database": {
    "host": "localhost",
    "port": 5432,
    "name": "mydb",
    "ssl": true
  },
  "features": ["auth", "logging", "caching"]
})

# TODO: Parse และแสดงข้อมูล config ทั้งหมด
# TODO: ตรวจสอบว่า feature "auth" มีหรือไม่
```

### แบบฝึกหัดที่ 3: JSON Transformer
สร้าง function ที่รับ JSON array ของ numbers และ return JSON object พร้อม statistics:

```crystal
require "json"

def analyze_numbers(json_array : String) : String
  # TODO: parse JSON array
  # TODO: คำนวณ min, max, sum, average
  # TODO: return JSON object พร้อม statistics
end

puts analyze_numbers("[10, 5, 8, 3, 15, 7, 12]")
# Expected: {"min":3,"max":15,"sum":60,"average":8.571...,"count":7}
```

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

1. **`require "json"`** - การนำเข้า JSON library
2. **`JSON.parse`** - การแปลง JSON string เป็น `JSON::Any`
3. **`JSON::Any`** - type สำหรับแทนค่า JSON และ `as_*` methods
4. **`[]` และ `[]?`** - การเข้าถึงค่าใน JSON objects และ arrays
5. **`to_json`** - การแปลง Crystal values เป็น JSON string
6. **`from_json`** - การแปลง JSON string เป็น Crystal types
7. **`JSON::Serializable`** - annotation สำหรับ auto serialize/deserialize
8. **`@[JSON::Field]`** - การ map ชื่อ field ใน JSON
9. **Custom serialization** - การควบคุมกระบวนการ serialize เอง
10. **Nil handling** - การจัดการ null values ใน JSON
11. **`to_pretty_json`** - การแสดง JSON แบบอ่านง่าย
12. **`JSON.build`** - การสร้าง JSON แบบ programmatic
13. **Nested objects** - การทำงานกับ JSON ซ้อนกัน
14. **Arrays ใน JSON** - การจัดการ JSON arrays
15. **Error handling** - การจัดการข้อผิดพลาดใน JSON parsing

JSON ใน Crystal มีความสามารถครบถ้วนและมีประสิทธิภาพสูง ทั้งสำหรับการทำงานกับ API, configuration files, และการแลกเปลี่ยนข้อมูลระหว่างระบบ
