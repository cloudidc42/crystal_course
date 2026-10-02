# Part 019: Multiple Assignment และ Destructuring

## บทนำ

Crystal รองรับ **Multiple Assignment** และ **Destructuring** ที่ช่วยให้เราแยกข้อมูลจาก tuple, array, และ named tuple ออกมาเป็น variable ต่างๆ ได้ในบรรทัดเดียว ทำให้โค้ดกระชับและอ่านง่ายขึ้นมาก

---

## 1. Multiple Assignment พื้นฐาน

```crystal
# กำหนดหลายค่าพร้อมกัน
a, b = 1, 2
puts a  # => 1
puts b  # => 2

# กำหนดจาก array
x, y, z = [10, 20, 30]
puts "#{x}, #{y}, #{z}"  # => 10, 20, 30

# กำหนดจาก tuple
first, second = {100, 200}
puts "#{first}, #{second}"  # => 100, 200
```

### ค่าไม่ครบ

```crystal
# ถ้าค่าน้อยกว่าตัวแปร ตัวแปรที่เกินจะเป็น nil
a, b, c = 1, 2
puts a.inspect  # => 1
puts b.inspect  # => 2
puts c.inspect  # => nil

# ถ้าค่ามากกว่าตัวแปร ค่าที่เกินจะถูกละทิ้ง
x, y = 1, 2, 3, 4
puts x  # => 1
puts y  # => 2
# 3 และ 4 ถูกละทิ้ง
```

---

## 2. Swap ด้วย Multiple Assignment

วิธีที่สะอาดที่สุดในการสลับค่าระหว่างตัวแปร:

```crystal
# swap โดยไม่ต้องใช้ temp variable!
a, b = 5, 10
puts "ก่อน: a=#{a}, b=#{b}"

a, b = b, a  # swap!
puts "หลัง: a=#{a}, b=#{b}"
# ก่อน: a=5, b=10
# หลัง: a=10, b=5
```

### Swap หลายตัวพร้อมกัน

```crystal
# Rotate 3 ค่า
x, y, z = 1, 2, 3
puts "ก่อน: #{x}, #{y}, #{z}"

x, y, z = y, z, x
puts "หลัง: #{x}, #{y}, #{z}"
# ก่อน: 1, 2, 3
# หลัง: 2, 3, 1
```

### ใช้งานจริง: Bubble Sort

```crystal
arr = [5, 3, 8, 1, 9, 2, 7, 4, 6]

(arr.size - 1).times do
  (arr.size - 1).times do |i|
    arr[i], arr[i + 1] = arr[i + 1], arr[i] if arr[i] > arr[i + 1]
  end
end

puts arr.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

---

## 3. Splat Operator ใน Assignment

`*` ใช้รับค่าที่เหลือเป็น array:

```crystal
# splat ที่ท้าย
first, *rest = [1, 2, 3, 4, 5]
puts first       # => 1
puts rest.inspect  # => [2, 3, 4, 5]

# splat ที่หน้า
*init, last = [1, 2, 3, 4, 5]
puts init.inspect  # => [1, 2, 3, 4]
puts last          # => 5

# splat ตรงกลาง
head, *middle, tail = [1, 2, 3, 4, 5]
puts head          # => 1
puts middle.inspect # => [2, 3, 4]
puts tail          # => 5
```

### Splat กับ String split

```crystal
# แยก protocol, domain, path จาก URL
url = "https://www.example.com/path/to/page"
protocol, *parts = url.split("://")
puts protocol  # => https
domain, *path_parts = parts.first.split("/")
puts domain       # => www.example.com
puts path_parts.inspect  # => ["path", "to", "page"]
```

### Splat กับ CSV parsing

```crystal
# parse CSV line
line = "Alice,25,Bangkok,Engineer"
name, age_str, *other_fields = line.split(",")

puts "ชื่อ: #{name}"
puts "อายุ: #{age_str.to_i}"
puts "ข้อมูลอื่น: #{other_fields.join(", ")}"
```

---

## 4. Tuple Destructuring

Tuple ใน Crystal เป็น ordered, fixed-size collection:

```crystal
# สร้างและ destructure tuple
point = {10, 20}
x, y = point
puts "x=#{x}, y=#{y}"

# 3D point
coords = {1.5, 2.7, 3.2}
x, y, z = coords
puts "3D: (#{x}, #{y}, #{z})"

# Tuple จาก method
def min_max(arr : Array(Int32)) : {Int32, Int32}
  {arr.min, arr.max}
end

min, max = min_max([3, 1, 4, 1, 5, 9, 2, 6])
puts "Min: #{min}, Max: #{max}"
# => Min: 1, Max: 9
```

### Tuple ใน each

```crystal
# Array of tuples
coordinates = [{0, 0}, {1, 0}, {1, 1}, {0, 1}]

coordinates.each do |(x, y)|
  puts "จุด: (#{x}, #{y})"
end

# Hash iteration returns key-value tuple
scores = {"Alice" => 95, "Bob" => 87, "Charlie" => 92}
scores.each do |name, score|
  puts "#{name}: #{score}"
end
```

---

## 5. Array Destructuring

```crystal
# Array destructuring
numbers = [1, 2, 3, 4, 5]

# เอาค่าแรกและที่เหลือ
first, *rest = numbers
puts "first: #{first}, rest: #{rest}"

# nested array (ต้องใช้ index หรือ flatten)
matrix = [[1, 2], [3, 4], [5, 6]]

matrix.each do |row|
  a, b = row
  puts "#{a} + #{b} = #{a + b}"
end
```

### Destructuring ใน block parameters

```crystal
# zip สร้าง array of tuples
names = ["Alice", "Bob", "Charlie"]
scores = [95, 87, 92]

names.zip(scores).each do |(name, score)|
  grade = score >= 90 ? "A" : score >= 80 ? "B" : "C"
  puts "#{name}: #{score} (#{grade})"
end

# each_with_index ก็ destructure ได้
["apple", "banana", "cherry"].each_with_index do |(fruit, index)|
  puts "#{index}: #{fruit}"
end
# Note: หรือใช้แบบนี้
["apple", "banana", "cherry"].each_with_index do |fruit, index|
  puts "#{index}: #{fruit}"
end
```

---

## 6. Named Tuple Destructuring

Named Tuple ให้ access ด้วยชื่อ:

```crystal
# Named Tuple พื้นฐาน
person = {name: "Alice", age: 30, city: "Bangkok"}

# access ด้วยชื่อ
puts person[:name]  # => Alice
puts person[:age]   # => 30

# ไม่สามารถ destructure ด้วย multiple assignment โดยตรง
# แต่สามารถ extract หลายค่าพร้อมกันได้ด้วย values_at
name, age = person.values_at(:name, :age)
puts "#{name} อายุ #{age}"
```

### Named Tuple ใน array

```crystal
users = [
  {name: "Alice", age: 30, role: "admin"},
  {name: "Bob", age: 25, role: "user"},
  {name: "Charlie", age: 35, role: "moderator"},
]

users.each do |user|
  name = user[:name]
  role = user[:role]
  puts "#{name} (#{role})"
end

# กรองและ extract
admins = users
  .select { |u| u[:role] == "admin" }
  .map { |u| u[:name] }
puts "Admins: #{admins.join(", ")}"
```

---

## 7. Method Returning Multiple Values

ใน Crystal method สามารถส่งค่ากลับหลายค่าผ่าน Tuple:

```crystal
# Method ที่ส่งค่ากลับ 2 ค่า
def divide_with_remainder(a : Int32, b : Int32) : {Int32, Int32}
  {a / b, a % b}
end

quotient, remainder = divide_with_remainder(17, 5)
puts "17 ÷ 5 = #{quotient} เศษ #{remainder}"
# => 17 ÷ 5 = 3 เศษ 2
```

### Method ส่งค่ากลับ Named Tuple

```crystal
def analyze_text(text : String) : NamedTuple(
  word_count: Int32,
  char_count: Int32,
  avg_word_length: Float64
)
  words = text.split(/\s+/)
  {
    word_count: words.size,
    char_count: text.size,
    avg_word_length: words.sum(&.size).to_f / words.size,
  }
end

stats = analyze_text("Hello World Crystal Programming")
puts "จำนวนคำ: #{stats[:word_count]}"
puts "จำนวนตัวอักษร: #{stats[:char_count]}"
puts "ความยาวเฉลี่ย: #{stats[:avg_word_length].round(2)}"
```

### Method ส่งค่ากลับแบบ structured

```crystal
def parse_date(date_str : String) : {Int32, Int32, Int32}?
  parts = date_str.split("-")
  return nil unless parts.size == 3
  
  begin
    year = parts[0].to_i
    month = parts[1].to_i
    day = parts[2].to_i
    {year, month, day}
  rescue ArgumentError
    nil
  end
end

# ใช้งาน
if result = parse_date("2024-03-15")
  year, month, day = result
  puts "ปี: #{year}, เดือน: #{month}, วัน: #{day}"
else
  puts "รูปแบบวันที่ไม่ถูกต้อง"
end

puts parse_date("2024-13-45").inspect  # => nil (เดือนและวันไม่ถูกต้อง)
```

---

## 8. Underscore _ (Ignore Value)

`_` ใช้เพื่อบอกว่าไม่สนใจค่านั้น:

```crystal
# ไม่สนใจค่าแรก
_, second, third = [1, 2, 3]
puts "second: #{second}, third: #{third}"

# ไม่สนใจกลาง
first, _, last = [10, 20, 30]
puts "first: #{first}, last: #{last}"

# ไม่สนใจหลายค่า
_, _, third_value, *_ = [1, 2, 3, 4, 5, 6]
puts third_value  # => 3
```

### _ ใน each

```crystal
# ไม่สนใจ index
["a", "b", "c"].each_with_index do |value, _|
  print value + " "
end
puts

# ไม่สนใจ value ใน hash
scores = {"Alice" => 95, "Bob" => 87, "Charlie" => 92}
scores.each do |name, _|
  puts name  # เฉพาะชื่อ
end

# ไม่สนใจ key
scores.each do |_, score|
  print "#{score} "  # เฉพาะคะแนน
end
puts
```

### _ ในการ parse

```crystal
# parse CSV และไม่สนใจบาง field
csv_data = [
  "Alice,25,Bangkok,Engineer,100000",
  "Bob,30,Chiang Mai,Designer,90000",
]

csv_data.each do |line|
  name, _, city, job, _ = line.split(",")
  puts "#{name} ทำงานเป็น #{job} ที่ #{city}"
end
```

---

## 9. Destructuring ใน Real-World Scenarios

### Parsing HTTP Response

```crystal
# จำลอง HTTP response parsing
def parse_http_status(status_line : String) : {String, Int32, String}
  # "HTTP/1.1 200 OK" => {"HTTP/1.1", 200, "OK"}
  parts = status_line.split(" ", 3)
  {parts[0], parts[1].to_i, parts[2]}
end

status_line = "HTTP/1.1 200 OK"
version, code, message = parse_http_status(status_line)
puts "Version: #{version}"
puts "Code: #{code}"
puts "Message: #{message}"

# ตัวอย่างอื่น
["HTTP/1.1 404 Not Found", "HTTP/2 503 Service Unavailable"].each do |line|
  _, code, msg = parse_http_status(line)
  puts "Status #{code}: #{msg}"
end
```

### Coordinate Transformation

```crystal
def rotate_90(x : Float64, y : Float64) : {Float64, Float64}
  {-y, x}
end

def scale(x : Float64, y : Float64, factor : Float64) : {Float64, Float64}
  {x * factor, y * factor}
end

# chain transformations
x, y = 3.0, 4.0
puts "Original: (#{x}, #{y})"

x, y = rotate_90(x, y)
puts "After rotate: (#{x}, #{y})"

x, y = scale(x, y, 2.0)
puts "After scale: (#{x}, #{y})"
```

### RGB Color Manipulation

```crystal
def hex_to_rgb(hex : String) : {Int32, Int32, Int32}
  hex = hex.lstrip("#")
  r = hex[0..1].to_i(16)
  g = hex[2..3].to_i(16)
  b = hex[4..5].to_i(16)
  {r, g, b}
end

def rgb_to_grayscale(r : Int32, g : Int32, b : Int32) : Int32
  (0.299 * r + 0.587 * g + 0.114 * b).to_i
end

["#FF5733", "#2ECC71", "#3498DB"].each do |color|
  r, g, b = hex_to_rgb(color)
  gray = rgb_to_grayscale(r, g, b)
  puts "#{color} => RGB(#{r}, #{g}, #{b}) => Gray: #{gray}"
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Swap Algorithm

ใช้ multiple assignment เพื่อ implement selection sort:

```crystal
def selection_sort(arr : Array(Int32)) : Array(Int32)
  result = arr.dup
  
  (0...result.size - 1).each do |i|
    min_idx = i
    (i + 1...result.size).each do |j|
      min_idx = j if result[j] < result[min_idx]
    end
    # TODO: ใช้ multiple assignment swap result[i] กับ result[min_idx]
  end
  
  result
end

arr = [64, 25, 12, 22, 11]
puts selection_sort(arr).inspect  # => [11, 12, 22, 25, 64]
```

### แบบฝึกหัดที่ 2: Destructure Complex Data

```crystal
# ข้อมูล GPS coordinates
gps_data = [
  {lat: 13.7563, lon: 100.5018, name: "กรุงเทพฯ"},
  {lat: 18.7883, lon: 98.9853, name: "เชียงใหม่"},
  {lat: 7.8804, lon: 98.3923, name: "ภูเก็ต"},
]

def distance(lat1 : Float64, lon1 : Float64, lat2 : Float64, lon2 : Float64) : Float64
  # simplified distance (not real haversine)
  Math.sqrt((lat2 - lat1) ** 2 + (lon2 - lon1) ** 2)
end

# TODO: คำนวณระยะห่างระหว่างทุกคู่ของเมือง
# ใช้ destructuring ดึง lat, lon, name ออกมา
```

### แบบฝึกหัดที่ 3: Multiple Return Values

```crystal
# สร้างฟังก์ชัน stats ที่ส่งกลับหลายค่า
def calculate_stats(data : Array(Float64)) : NamedTuple(
  mean: Float64,
  median: Float64,
  std_dev: Float64,
  min: Float64,
  max: Float64
)
  # TODO: คำนวณและส่งกลับ Named Tuple
end

data = [2.5, 3.7, 1.2, 4.8, 3.3, 2.9, 4.1, 3.5]
stats = calculate_stats(data)
puts "Mean: #{stats[:mean].round(2)}"
puts "Median: #{stats[:median].round(2)}"
puts "Std Dev: #{stats[:std_dev].round(2)}"
puts "Min: #{stats[:min]}, Max: #{stats[:max]}"
```

### แบบฝึกหัดที่ 4: Splat Processing

```crystal
# สร้างฟังก์ชัน pipeline ที่รับค่าแรกเป็น initial value
# และค่าที่เหลือเป็น transformation functions
def pipeline(initial : Int32, *transforms : Proc(Int32, Int32)) : Int32
  # TODO: apply แต่ละ transform ตามลำดับ
end

double = ->(n : Int32) { n * 2 }
add_ten = ->(n : Int32) { n + 10 }
square = ->(n : Int32) { n * n }

result = pipeline(5, double, add_ten, square)
puts result  # => (5*2 + 10)^2 = 400
```

---

## สรุป

| Syntax | ตัวอย่าง | ใช้เมื่อ |
|--------|----------|---------|
| `a, b = 1, 2` | กำหนดหลายค่าพร้อมกัน | assign multiple vars |
| `a, b = b, a` | swap ไม่ต้องใช้ temp | สลับค่า |
| `first, *rest = arr` | splat รับค่าที่เหลือ | แยก head/tail |
| `*init, last = arr` | splat ที่หน้า | แยก init/last |
| `x, y = tuple` | unpack tuple | ใช้กับ return value |
| `_, second = arr` | `_` ไม่สนใจค่า | skip unwanted values |
| NamedTuple | `{name: "Alice"}` | structured return value |

### Best Practices

1. **ใช้ swap idiom** `a, b = b, a` แทน temp variable
2. **Named Tuple** สำหรับ method ที่ return หลายค่าที่มีความหมาย
3. **`_`** เพื่อแสดงเจตนาว่าไม่ใช้ค่านั้น (ไม่ใช่แค่ลืม)
4. **Splat** สำหรับ variadic processing ที่ยืดหยุ่น
5. **Type annotation** บน return tuple เพื่อความชัดเจน
