# ตอนที่ 83: CSV ใน Crystal

## บทนำ

CSV (Comma-Separated Values) เป็นรูปแบบไฟล์ที่นิยมใช้สำหรับการนำเข้าและส่งออกข้อมูลตาราง Crystal มี built-in support สำหรับ CSV ผ่าน `require "csv"` ที่รองรับทั้งการอ่าน, เขียน และจัดการข้อมูล CSV

---

## 1. การ require "csv"

```crystal
require "csv"
```

---

## 2. CSV.parse - การแปลง CSV String เป็น Array

`CSV.parse` แปลง CSV string ทั้งหมดเป็น `Array(Array(String))`:

```crystal
require "csv"

csv_string = "สมชาย,30,กรุงเทพ\nมานี,25,เชียงใหม่\nวิชัย,35,ขอนแก่น"

rows = CSV.parse(csv_string)
puts rows.class  # => Array(Array(String))
puts rows.size   # => 3

rows.each do |row|
  puts "#{row[0]}, #{row[1]} ปี, #{row[2]}"
end
```

### การ parse CSV พร้อม headers:

```crystal
require "csv"

csv_with_headers = "name,age,city\nสมชาย,30,กรุงเทพ\nมานี,25,เชียงใหม่"

rows = CSV.parse(csv_with_headers, headers: true)
rows.each do |row|
  puts "ชื่อ: #{row[0]}, อายุ: #{row[1]}, เมือง: #{row[2]}"
end

# แถวแรกจะถูกข้ามไปเพราะเป็น header
puts "จำนวนแถวข้อมูล: #{rows.size}"  # => 2
```

---

## 3. CSV.each_row - การอ่านทีละแถว

`CSV.each_row` ประหยัดหน่วยความจำกว่า parse เพราะอ่านทีละแถว:

```crystal
require "csv"

csv_data = <<-CSV
  ชื่อ,คะแนน,เกรด
  สมชาย,85,B
  มานี,95,A
  วิชัย,72,C+
  สมศรี,91,A
  CSV

CSV.each_row(csv_data) do |row|
  puts "#{row[0]}: #{row[1]} (#{row[2]})"
end
```

### ข้ามแถว header:

```crystal
require "csv"

csv_data = "name,score,grade\nสมชาย,85,B\nมานี,95,A\nวิชัย,72,C+"

first_row = true
CSV.each_row(csv_data) do |row|
  if first_row
    puts "Headers: #{row.join(", ")}"
    first_row = false
    next
  end
  puts "#{row[0]}: #{row[1]} (#{row[2]})"
end
```

---

## 4. การอ่าน CSV Files

```crystal
require "csv"

# จำลองเนื้อหาไฟล์ CSV
file_content = <<-CSV
  id,product_name,price,quantity,category
  1,แล็ปท็อป Asus,25000,5,electronics
  2,โทรศัพท์ Samsung,15000,10,electronics
  3,หูฟัง Sony,3500,20,accessories
  4,เมาส์ Logitech,800,50,accessories
  5,คีย์บอร์ด Mechanical,2500,15,accessories
  CSV

# อ่านทั้งหมดพร้อมกัน
rows = CSV.parse(file_content)

# แถวแรกคือ header
headers = rows.first
puts "Columns: #{headers.join(" | ")}"

# แถวที่เหลือคือข้อมูล
data_rows = rows[1..]
data_rows.each do |row|
  id       = row[0]
  name     = row[1]
  price    = row[2].to_i
  quantity = row[3].to_i
  category = row[4]

  total_value = price * quantity
  puts "#{id}. #{name} (#{category}): #{price} บาท x #{quantity} = #{total_value} บาท"
end

# คำนวณมูลค่ารวม
total = data_rows.sum { |row| row[2].to_i * row[3].to_i }
puts "\nมูลค่าสินค้าคงคลังรวม: #{total} บาท"
```

### การอ่าน CSV file จริง:

```crystal
require "csv"

# วิธีอ่านจาก file จริง
# File.open("products.csv") do |file|
#   CSV.each_row(file) do |row|
#     # process each row
#   end
# end

# หรืออ่านทั้งไฟล์
# content = File.read("products.csv")
# rows = CSV.parse(content)
```

---

## 5. การเขียน CSV ด้วย CSV.build

`CSV.build` ใช้สร้าง CSV string:

```crystal
require "csv"

students = [
  {name: "สมชาย", age: 20, gpa: 3.5, major: "Computer Science"},
  {name: "มานี", age: 21, gpa: 3.8, major: "Mathematics"},
  {name: "วิชัย", age: 19, gpa: 3.2, major: "Physics"},
  {name: "สมศรี", age: 22, gpa: 3.9, major: "Chemistry"},
]

csv_output = CSV.build do |csv|
  # เขียน header
  csv.row "name", "age", "gpa", "major"

  # เขียนข้อมูล
  students.each do |s|
    csv.row s[:name], s[:age], s[:gpa], s[:major]
  end
end

puts csv_output
```

### เขียน CSV file จริง:

```crystal
require "csv"

data = [
  ["สมชาย", "somchai@example.com", "081-234-5678"],
  ["มานี", "mani@example.com", "082-345-6789"],
]

# เขียนลงไฟล์
csv_content = CSV.build do |csv|
  csv.row "ชื่อ", "อีเมล", "โทรศัพท์"
  data.each { |row| csv.row(*row) }
end

# File.write("contacts.csv", csv_content)
puts csv_content
```

---

## 6. CSV กับ Headers

### การใช้ CSV::CSV object:

```crystal
require "csv"

csv_data = <<-CSV
  id,first_name,last_name,email,department
  1,สม,ชาย,somchai@company.com,Engineering
  2,มา,นี,mani@company.com,Marketing
  3,วิ,ชัย,vichai@company.com,Engineering
  4,สม,ศรี,somsri@company.com,HR
  CSV

# สร้าง CSV object พร้อม header support
csv = CSV.new(csv_data, headers: true)

# เข้าถึงด้วยชื่อ column
csv.each do |row|
  id         = row["id"]
  first_name = row["first_name"]
  last_name  = row["last_name"]
  email      = row["email"]
  dept       = row["department"]

  puts "#{id}: #{first_name} #{last_name} (#{dept}) - #{email}"
end
```

### ตรวจสอบ headers:

```crystal
require "csv"

csv_data = "name,age,city\nสมชาย,30,กรุงเทพ"

csv = CSV.new(csv_data, headers: true)
puts "Headers: #{csv.headers.inspect}"  # => ["name", "age", "city"]
```

---

## 7. CSV::Row - การทำงานกับแถว CSV

```crystal
require "csv"

csv_data = <<-CSV
  product,price,stock,rating
  กาแฟอาราบิก้า,250,100,4.8
  ชาอู่หลง,180,50,4.5
  น้ำผลไม้รวม,90,200,4.2
  โกโก้ร้อน,120,75,4.6
  CSV

csv = CSV.new(csv_data, headers: true)

csv.each do |row|
  # เข้าถึงด้วย index
  product = row[0]

  # เข้าถึงด้วย header name
  price   = row["price"].to_f
  stock   = row["stock"].to_i
  rating  = row["rating"].to_f

  # ตรวจสอบสินค้าขายดี
  status = rating >= 4.5 ? "⭐ ขายดี" : "ปกติ"
  puts "#{product}: #{price} บาท (คงเหลือ: #{stock}) #{status}"
end
```

### Row size และ column count:

```crystal
require "csv"

csv_data = "a,b,c,d,e\n1,2,3,4,5"
csv = CSV.new(csv_data, headers: true)

csv.each do |row|
  puts "จำนวน columns: #{row.size}"
  puts "Values: #{row.to_a.join(", ")}"
end
```

---

## 8. Field Types - การแปลงประเภทข้อมูล

CSV เก็บข้อมูลเป็น String ทั้งหมด ต้องแปลงเอง:

```crystal
require "csv"

csv_data = <<-CSV
  name,age,salary,hire_date,active
  สมชาย ใจดี,35,75000.50,2020-01-15,true
  มานี สวยงาม,28,65000.00,2021-06-01,true
  วิชัย เก่งกล้า,42,95000.75,2018-03-20,false
  CSV

csv = CSV.new(csv_data, headers: true)

csv.each do |row|
  name      = row["name"]             # String
  age       = row["age"].to_i         # Int32
  salary    = row["salary"].to_f      # Float64
  hire_date = row["hire_date"]        # String (ต้องแปลงเป็น Time เอง)
  active    = row["active"] == "true" # Bool

  puts "#{name}: อายุ #{age}, เงินเดือน #{salary.format(decimal_places: 2)} บาท"
  puts "  วันที่เข้างาน: #{hire_date}, สถานะ: #{active ? "ทำงานอยู่" : "ออกแล้ว"}"
end
```

### Helper method สำหรับแปลงข้อมูล:

```crystal
require "csv"

def parse_bool(value : String) : Bool
  case value.downcase
  when "true", "yes", "1", "y" then true
  else false
  end
end

def parse_int?(value : String) : Int32?
  value.to_i?
end

def parse_float?(value : String) : Float64?
  value.to_f?
end

csv_data = "item,count,price,available\napple,10,25.5,yes\nbanana,0,15.0,no\ncherry,5,80.0,true"

csv = CSV.new(csv_data, headers: true)
csv.each do |row|
  name  = row["item"]
  count = parse_int?(row["count"]) || 0
  price = parse_float?(row["price"]) || 0.0
  avail = parse_bool(row["available"])

  puts "#{name}: #{count} ชิ้น, #{price} บาท, #{avail ? "มีสินค้า" : "หมด"}"
end
```

---

## 9. Quoted Fields - ฟิลด์ที่มี Special Characters

CSV รองรับ quoted fields สำหรับข้อมูลที่มีเครื่องหมายจุลภาค หรือ newlines:

```crystal
require "csv"

# ฟิลด์ที่มี comma ต้อง quote
csv_data = <<-CSV
  name,address,phone
  สมชาย,"123, ถนนสุขุมวิท, กรุงเทพ",081-234-5678
  มานี,"456 ถนนนิมมาน, เชียงใหม่",082-345-6789
  วิชัย,"789 ถนนมิตรภาพ, ขอนแก่น",083-456-7890
  CSV

csv = CSV.new(csv_data, headers: true)
csv.each do |row|
  puts "#{row["name"]}: #{row["address"]}"
end
```

### ฟิลด์ที่มี quotation marks:

```crystal
require "csv"

# ใช้ "" เพื่อ escape " ใน quoted field
csv_data = %(name,description\nสินค้า A,"สินค้า ""พิเศษ"" ราคาถูก"\nสินค้า B,ปกติธรรมดา)

csv = CSV.new(csv_data, headers: true)
csv.each do |row|
  puts "#{row["name"]}: #{row["description"]}"
end
```

### ฟิลด์ที่มี newlines:

```crystal
require "csv"

# ฟิลด์ที่มี newline ต้อง wrap ใน quotes
csv_data = "title,content\nข่าวที่ 1,\"บรรทัดที่ 1\nบรรทัดที่ 2\nบรรทัดที่ 3\"\nข่าวที่ 2,ข้อความปกติ"

csv = CSV.new(csv_data, headers: true)
csv.each do |row|
  puts "Title: #{row["title"]}"
  puts "Content:"
  row["content"].split("\n").each { |line| puts "  #{line}" }
  puts ""
end
```

---

## 10. Custom Separator - ตัวคั่นที่กำหนดเอง

```crystal
require "csv"

# TSV (Tab-Separated Values)
tsv_data = "name\tage\tcity\nสมชาย\t30\tกรุงเทพ\nมานี\t25\tเชียงใหม่"

csv = CSV.new(tsv_data, headers: true, separator: '\t')
csv.each do |row|
  puts "#{row["name"]} (#{row["age"]}) - #{row["city"]}"
end
```

### Pipe-separated:

```crystal
require "csv"

# Pipe (|) separator
pipe_data = "id|name|value\n1|สมชาย|100\n2|มานี|200\n3|วิชัย|300"

csv = CSV.new(pipe_data, headers: true, separator: '|')
csv.each do |row|
  puts "#{row["id"]}: #{row["name"]} = #{row["value"]}"
end
```

### Semicolon separator (European format):

```crystal
require "csv"

# European format ใช้ ; เพราะ , ใช้เป็น decimal separator
euro_data = "Artikel;Preis;Menge\nApfel;1,50;10\nBanane;0,99;20"

csv = CSV.new(euro_data, headers: true, separator: ';')
csv.each do |row|
  puts "#{row["Artikel"]}: #{row["Preis"]} (#{row["Menge"]} Stk.)"
end
```

---

## 11. Streaming Large Files - การอ่านไฟล์ขนาดใหญ่

สำหรับไฟล์ CSV ขนาดใหญ่ ใช้ streaming เพื่อประหยัดหน่วยความจำ:

```crystal
require "csv"

# จำลองไฟล์ขนาดใหญ่
def generate_large_csv(rows : Int32) : String
  CSV.build do |csv|
    csv.row "id", "name", "value", "timestamp"
    rows.times do |i|
      csv.row i + 1, "item_#{i + 1}", rand(1000), Time.local.to_s
    end
  end
end

large_csv = generate_large_csv(10000)

# นับแถว
row_count = 0
total_value = 0
max_value = 0
min_value = Int32::MAX

first_row = true
CSV.each_row(large_csv) do |row|
  if first_row
    first_row = false
    next  # ข้าม header
  end

  row_count += 1
  value = row[2].to_i
  total_value += value
  max_value = value if value > max_value
  min_value = value if value < min_value
end

puts "จำนวนแถว: #{row_count}"
puts "ค่ารวม: #{total_value}"
puts "ค่าเฉลี่ย: #{total_value / row_count}"
puts "ค่าสูงสุด: #{max_value}"
puts "ค่าต่ำสุด: #{min_value}"
puts "ขนาด CSV: #{large_csv.bytesize / 1024} KB"
```

### การ filter ระหว่างอ่าน:

```crystal
require "csv"

csv_data = <<-CSV
  name,department,salary,years
  สมชาย,Engineering,80000,5
  มานี,Marketing,65000,3
  วิชัย,Engineering,95000,8
  สมศรี,HR,55000,2
  ประยุทธ์,Engineering,110000,12
  ดวงใจ,Marketing,70000,6
  CSV

# Filter เฉพาะ Engineering department
engineering_staff = [] of Array(String)

first_row = true
CSV.each_row(csv_data) do |row|
  if first_row
    first_row = false
    next
  end
  engineering_staff << row.to_a if row[1] == "Engineering"
end

puts "Engineering Department:"
engineering_staff.each do |row|
  puts "  #{row[0]}: #{row[2].to_i.format} บาท (#{row[3]} ปี)"
end

total_salary = engineering_staff.sum { |r| r[2].to_i }
puts "เงินเดือนรวม Engineering: #{total_salary.format} บาท"
```

---

## 12. การ Transform CSV

### เปลี่ยนรูปแบบข้อมูล:

```crystal
require "csv"

# Input CSV
input_csv = <<-CSV
  first_name,last_name,birth_year,score
  สม,ชาย,1990,85
  มา,นี,1995,92
  วิ,ชัย,1988,78
  CSV

# Transform เป็น output format ใหม่
output = CSV.build do |csv|
  csv.row "full_name", "age", "grade", "passed"

  first_row = true
  CSV.each_row(input_csv) do |row|
    if first_row
      first_row = false
      next
    end

    full_name = "#{row[0]}#{row[1]}"
    age       = Time.local.year - row[2].to_i
    score     = row[3].to_i
    grade     = case score
                when 90..100 then "A"
                when 80..89  then "B"
                when 70..79  then "C"
                else              "F"
                end
    passed    = score >= 70

    csv.row full_name, age, grade, passed
  end
end

puts output
```

---

## 13. CSV ที่มีหลาย Sheet (จำลอง)

```crystal
require "csv"

# จำลองการทำงานกับหลาย CSV sections
def process_section(csv_string : String, section_name : String)
  puts "=== #{section_name} ==="
  rows = CSV.parse(csv_string)
  return if rows.empty?

  headers = rows.first
  puts "Columns: #{headers.join(", ")}"

  rows[1..].each do |row|
    row.each_with_index do |value, i|
      print "#{headers[i]}: #{value}  "
    end
    puts ""
  end
  puts ""
end

sales_csv    = "month,amount,orders\nมกราคม,150000,45\nกุมภาพันธ์,180000,52"
products_csv = "id,name,price\n1,สินค้า A,500\n2,สินค้า B,750"
customers_csv = "id,name,total_orders\n1,ลูกค้า A,15\n2,ลูกค้า B,8"

process_section(sales_csv, "Sales")
process_section(products_csv, "Products")
process_section(customers_csv, "Customers")
```

---

## 14. CSV Data Analysis

```crystal
require "csv"

csv_data = <<-CSV
  date,category,amount,description
  2024-01-05,food,350,อาหารกลางวัน
  2024-01-07,transport,120,ค่าแท็กซี่
  2024-01-10,food,180,กาแฟและขนม
  2024-01-12,entertainment,500,ดูหนัง
  2024-01-15,food,420,อาหารเย็น
  2024-01-18,transport,85,รถเมล์
  2024-01-20,shopping,1200,เสื้อผ้า
  2024-01-22,food,250,อาหารกลางวัน
  2024-01-25,entertainment,300,คอนเสิร์ต
  2024-01-28,transport,200,แกร็บ
  CSV

# วิเคราะห์ข้อมูล
category_totals = Hash(String, Float64).new(0.0)
daily_spending  = Hash(String, Float64).new(0.0)
transactions    = [] of {date: String, category: String, amount: Float64}

first_row = true
CSV.each_row(csv_data) do |row|
  if first_row
    first_row = false
    next
  end

  date     = row[0]
  category = row[1]
  amount   = row[2].to_f

  category_totals[category] += amount
  daily_spending[date] += amount
  transactions << {date: date, category: category, amount: amount}
end

puts "=== รายงานค่าใช้จ่าย ==="
puts ""
puts "รายจ่ายตามหมวดหมู่:"
category_totals.each do |cat, total|
  puts "  #{cat}: #{total.format(decimal_places: 2)} บาท"
end

total = category_totals.values.sum
puts "\nยอดรวม: #{total.format(decimal_places: 2)} บาท"

puts "\nวันที่ใช้จ่ายสูงสุด:"
max_day = daily_spending.max_by { |_, v| v }
puts "  #{max_day[0]}: #{max_day[1].format(decimal_places: 2)} บาท"

puts "\nค่าเฉลี่ยต่อครั้ง: #{(total / transactions.size).format(decimal_places: 2)} บาท"
```

---

## 15. การ Generate CSV Report

```crystal
require "csv"

struct SalesRecord
  property month : String
  property product : String
  property units : Int32
  property revenue : Float64
  property cost : Float64

  def initialize(@month, @product, @units, @revenue, @cost)
  end

  def profit : Float64
    @revenue - @cost
  end

  def margin : Float64
    @revenue > 0 ? (@profit / @revenue * 100).round(2) : 0.0
  end
end

records = [
  SalesRecord.new("มกราคม", "สินค้า A", 150, 75000.0, 45000.0),
  SalesRecord.new("มกราคม", "สินค้า B", 80,  56000.0, 32000.0),
  SalesRecord.new("กุมภาพันธ์", "สินค้า A", 120, 60000.0, 36000.0),
  SalesRecord.new("กุมภาพันธ์", "สินค้า B", 95,  66500.0, 38000.0),
  SalesRecord.new("มีนาคม", "สินค้า A", 180, 90000.0, 54000.0),
  SalesRecord.new("มีนาคม", "สินค้า B", 100, 70000.0, 40000.0),
]

# สร้าง CSV report
report = CSV.build do |csv|
  csv.row "เดือน", "สินค้า", "จำนวนหน่วย", "รายได้", "ต้นทุน", "กำไร", "Margin%"

  records.each do |r|
    csv.row r.month, r.product, r.units, r.revenue, r.cost,
            r.profit.round(2), r.margin
  end

  # Summary row
  total_units   = records.sum(&.units)
  total_revenue = records.sum(&.revenue)
  total_cost    = records.sum(&.cost)
  total_profit  = records.sum(&.profit)
  avg_margin    = (total_profit / total_revenue * 100).round(2)

  csv.row "รวมทั้งหมด", "", total_units, total_revenue, total_cost,
          total_profit.round(2), avg_margin
end

puts report
```

---

## 16. CSV Filter และ Sort

```crystal
require "csv"

csv_data = <<-CSV
  id,name,department,salary,years_of_service
  1,สมชาย,Engineering,85000,5
  2,มานี,Marketing,62000,3
  3,วิชัย,Engineering,95000,8
  4,สมศรี,HR,58000,2
  5,ประยุทธ์,Engineering,110000,12
  6,ดวงใจ,Marketing,72000,6
  7,นงนุช,HR,65000,4
  8,อภิชัย,Engineering,78000,4
  CSV

# โหลดข้อมูล
employees = [] of Hash(String, String)

csv = CSV.new(csv_data, headers: true)
csv.each do |row|
  employee = {} of String => String
  ["id", "name", "department", "salary", "years_of_service"].each do |h|
    employee[h] = row[h]
  end
  employees << employee
end

# Filter: Engineering เท่านั้น
eng = employees.select { |e| e["department"] == "Engineering" }

# Sort: เรียงตาม salary (สูง -> ต่ำ)
eng.sort_by! { |e| -e["salary"].to_i }

# Output ผลลัพธ์เป็น CSV
output = CSV.build do |csv|
  csv.row "อันดับ", "ชื่อ", "เงินเดือน", "อายุงาน"
  eng.each_with_index do |e, i|
    csv.row i + 1, e["name"], e["salary"].to_i, "#{e["years_of_service"]} ปี"
  end
end

puts "Engineering Department (เรียงตามเงินเดือน):"
puts output
```

---

## 17. CSV Merge - การรวมหลาย CSV

```crystal
require "csv"

# CSV 1: ข้อมูลพนักงาน
employees_csv = <<-CSV
  id,name,department
  1,สมชาย,Engineering
  2,มานี,Marketing
  3,วิชัย,HR
  CSV

# CSV 2: ข้อมูลเงินเดือน
salaries_csv = <<-CSV
  employee_id,monthly_salary,bonus
  1,85000,15000
  2,62000,8000
  3,58000,5000
  CSV

# โหลดข้อมูลพนักงาน
employees = {} of String => Array(String)
CSV.each_row(employees_csv) do |row|
  employees[row[0]] = row.to_a unless row[0] == "id"
end

# Merge กับ salary data
merged = CSV.build do |csv|
  csv.row "id", "name", "department", "monthly_salary", "bonus", "total"

  CSV.each_row(salaries_csv) do |row|
    next if row[0] == "employee_id"

    emp_id = row[0]
    if emp = employees[emp_id]?
      salary    = row[1].to_i
      bonus     = row[2].to_i
      total     = salary + bonus
      csv.row emp_id, emp[1], emp[2], salary, bonus, total
    end
  end
end

puts merged
```

---

## 18. ตัวอย่างโปรแกรมสมบูรณ์ - Student Grade System

```crystal
require "csv"

struct Student
  property id : Int32
  property name : String
  property scores : Array(Int32)

  def initialize(@id, @name, @scores)
  end

  def average : Float64
    return 0.0 if @scores.empty?
    (@scores.sum.to_f / @scores.size).round(2)
  end

  def grade : String
    avg = average
    case avg
    when 90.0..100.0 then "A"
    when 80.0..89.9  then "B"
    when 70.0..79.9  then "C"
    when 60.0..69.9  then "D"
    else                  "F"
    end
  end

  def passed? : Bool
    average >= 60.0
  end
end

# Input: CSV ข้อมูลนักเรียน
input_csv = <<-CSV
  id,name,exam1,exam2,exam3,exam4,exam5
  1,สมชาย ใจดี,85,90,78,92,88
  2,มานี สวยงาม,72,68,75,80,71
  3,วิชัย เก่งกล้า,95,98,92,96,94
  4,สมศรี อ่อนน้อม,55,60,58,62,50
  5,ประยุทธ์ แข็งแกร่ง,88,85,90,87,91
  6,ดวงใจ อ่อนหวาน,45,50,48,55,42
  CSV

students = [] of Student

csv = CSV.new(input_csv, headers: true)
csv.each do |row|
  id     = row["id"].to_i
  name   = row["name"]
  scores = [
    row["exam1"].to_i,
    row["exam2"].to_i,
    row["exam3"].to_i,
    row["exam4"].to_i,
    row["exam5"].to_i,
  ]
  students << Student.new(id, name, scores)
end

# สร้าง report CSV
report = CSV.build do |csv|
  csv.row "ลำดับ", "ชื่อ", "สอบ1", "สอบ2", "สอบ3", "สอบ4", "สอบ5",
          "เฉลี่ย", "เกรด", "ผ่าน/ตก"

  students.each do |s|
    csv.row s.id, s.name,
            s.scores[0], s.scores[1], s.scores[2], s.scores[3], s.scores[4],
            s.average, s.grade, s.passed? ? "ผ่าน" : "ตก"
  end
end

puts "=== รายงานผลการเรียน ==="
puts report

# Statistics
total     = students.size
passed    = students.count(&.passed?)
class_avg = (students.sum(&.average) / total).round(2)

puts "=== สถิติ ==="
puts "จำนวนนักเรียนทั้งหมด: #{total}"
puts "ผ่าน: #{passed} คน (#{(passed.to_f / total * 100).round(1)}%)"
puts "ตก: #{total - passed} คน"
puts "ค่าเฉลี่ยของชั้น: #{class_avg}"

top_student = students.max_by(&.average)
puts "นักเรียนที่ได้คะแนนสูงสุด: #{top_student.name} (#{top_student.average})"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Inventory Management
สร้างโปรแกรมอ่าน CSV ต่อไปนี้และ:
1. หาสินค้าที่ stock น้อยกว่า 10
2. คำนวณมูลค่ารวมของ inventory
3. สร้าง CSV report

```
id,product_name,price,stock
1,สินค้า A,500,25
2,สินค้า B,300,8
3,สินค้า C,1200,15
4,สินค้า D,750,5
5,สินค้า E,200,50
```

### แบบฝึกหัดที่ 2: CSV Transformer
เขียน function ที่รับ CSV string และ mapping Hash แล้ว return CSV string ใหม่ที่เปลี่ยนชื่อ column:

```crystal
def rename_columns(csv : String, mapping : Hash(String, String)) : String
  # เช่น mapping = {"first_name" => "ชื่อ", "last_name" => "นามสกุล"}
  # TODO: implement
end
```

### แบบฝึกหัดที่ 3: CSV Statistics
สร้างโปรแกรมที่รับ CSV ที่มี numeric columns และสร้าง statistics summary:
- min, max, average, sum สำหรับแต่ละ numeric column
- แสดงผลเป็น CSV

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

1. **`require "csv"`** - การนำเข้า CSV library
2. **`CSV.parse`** - การแปลง CSV string เป็น `Array(Array(String))`
3. **`CSV.each_row`** - การอ่าน CSV ทีละแถวแบบ streaming
4. **การอ่าน CSV files** - อ่านจาก string หรือ IO
5. **`CSV.build`** - การสร้าง CSV string แบบ programmatic
6. **Headers** - การทำงานกับ column names
7. **`CSV::Row`** - การเข้าถึงข้อมูลในแถวด้วย index หรือ name
8. **Field types** - การแปลง String เป็น Int, Float, Bool
9. **Quoted fields** - การจัดการ fields ที่มี special characters
10. **Custom separator** - การใช้ delimiter อื่นนอกจาก comma
11. **Streaming** - การอ่าน CSV ขนาดใหญ่แบบ memory-efficient
12. **Data analysis** - การวิเคราะห์และรายงานข้อมูลจาก CSV

CSV ใน Crystal เหมาะสำหรับการ import/export ข้อมูล, data processing, และการ generate reports
