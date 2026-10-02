# Part 83: CSV

## บทนำ

CSV (Comma-Separated Values) เป็น format ธรรมดาสำหรับ tabular data Crystal มี standard library `csv` ที่รองรับ parsing และ generating CSV ครบถ้วน

---

## 1. require "csv" และ CSV.parse

```crystal
require "csv"

# parse CSV string
csv_str = "Alice,30,Bangkok\nBob,25,Chiang Mai\nCarol,35,Phuket"
CSV.parse(csv_str).each do |row|
  puts "Name: #{row[0]}, Age: #{row[1]}, City: #{row[2]}"
end

# CSV.parse คืน Array(Array(String))
rows = CSV.parse(csv_str)
puts rows.size       # => 3
puts rows[0].inspect # => ["Alice", "30", "Bangkok"]

# Parse กับ headers
csv_with_headers = "name,age,city\nAlice,30,Bangkok\nBob,25,Chiang Mai"
rows = CSV.parse(csv_with_headers)
headers = rows.first
data = rows[1..]

data.each do |row|
  record = headers.zip(row).to_h
  puts record.inspect
end

# Quoted fields
csv_str = %("Alice, Jr.",30,"Bangkok, Thailand"\n"Bob ""The Builder""",25,Phuket)
CSV.parse(csv_str).each do |row|
  puts row.inspect
end
# => ["Alice, Jr.", "30", "Bangkok, Thailand"]
# => ["Bob \"The Builder\"", "25", "Phuket"]

# Custom separator
tsv_str = "Alice\t30\tBangkok\nBob\t25\tChiang Mai"
CSV.parse(tsv_str, separator: '\t').each do |row|
  puts row.inspect
end
```

---

## 2. CSV.each_row

```crystal
require "csv"

# วนลูปผ่านแต่ละแถว (memory efficient)
csv_data = "Alice,30\nBob,25\nCarol,35\n"

CSV.each_row(csv_data) do |row|
  puts "#{row[0]}: #{row[1]}"
end

# จาก IO
csv_io = IO::Memory.new("Alice,30\nBob,25\n")
CSV.each_row(csv_io) do |row|
  puts row.inspect
end

# กับ File
# CSV.each_row(File.read("data.csv")) do |row|
#   process(row)
# end

# กรองข้อมูล
csv_data = "Alice,30,developer\nBob,25,designer\nCarol,35,developer\n"

developers = [] of Array(String)
CSV.each_row(csv_data) do |row|
  developers << row if row[2] == "developer"
end
developers.each { |r| puts r.inspect }

# นับแถว
count = 0
CSV.each_row(csv_data) { count += 1 }
puts "Total rows: #{count}"

# จัดเก็บเฉพาะบาง columns
CSV.each_row(csv_data) do |row|
  puts "#{row[0]} (#{row[2]})"  # name, role
end
```

---

## 3. CSV.build

```crystal
require "csv"

# สร้าง CSV string
result = CSV.build do |csv|
  csv.row "name", "age", "city"  # header
  csv.row "Alice", 30, "Bangkok"
  csv.row "Bob", 25, "Chiang Mai"
  csv.row "Carol", 35, "Phuket"
end

puts result

# build กับ IO
output = IO::Memory.new
CSV.build(output) do |csv|
  csv.row "id", "product", "price"
  csv.row 1, "Apple", 15.50
  csv.row 2, "Banana", 8.00
  csv.row 3, "Cherry", 35.00
end
puts output.to_s

# Custom separator และ quote char
result = CSV.build(separator: '\t') do |csv|
  csv.row "Alice", "30", "Bangkok"
  csv.row "Bob", "25", "Chiang Mai"
end
puts result.inspect  # TSV output

# Quoting ค่าที่มีตัวอักษรพิเศษ
result = CSV.build do |csv|
  csv.row "Alice, Jr.", "Bangkok, Thailand", "Has \"quotes\""
end
puts result  # => "Alice, Jr.","Bangkok, Thailand","Has ""quotes"""
```

---

## 4. Reading CSV Files

```crystal
require "csv"

# สร้าง test CSV file
File.write("/tmp/test.csv", <<-CSV)
  name,age,department,salary
  Alice,30,Engineering,75000
  Bob,25,Design,55000
  Carol,35,Engineering,85000
  Dave,28,Marketing,60000
  Eve,32,Design,65000
  CSV

# อ่าน CSV file
rows = CSV.parse(File.read("/tmp/test.csv"))
headers = rows.first
puts "Headers: #{headers.inspect}"

data = rows[1..]
puts "\nAll employees:"
data.each do |row|
  puts "  #{row[0]} (#{row[2]}): #{row[1]} years, ฿#{row[3]}"
end

# กรองตาม department
puts "\nEngineers:"
data.select { |r| r[2] == "Engineering" }.each do |r|
  puts "  #{r[0]}"
end

# หาเงินเดือนเฉลี่ย
salaries = data.map { |r| r[3].to_f }
avg = salaries.sum / salaries.size
puts "\nAverage salary: ฿#{avg.round(2)}"

# Sort ตาม salary
sorted = data.sort_by { |r| r[3].to_i }
puts "\nBy salary (ascending):"
sorted.each { |r| puts "  #{r[0]}: ฿#{r[3]}" }

File.delete("/tmp/test.csv")
```

---

## 5. Writing CSV Files

```crystal
require "csv"

# เขียน CSV file
File.open("/tmp/output.csv", "w") do |file|
  CSV.build(file) do |csv|
    # Headers
    csv.row "ID", "Name", "Email", "Score", "Grade"
    
    # Data
    students = [
      {1, "Alice", "alice@school.com", 95, "A"},
      {2, "Bob", "bob@school.com", 82, "B"},
      {3, "Carol", "carol@school.com", 78, "C"},
      {4, "Dave", "dave@school.com", 91, "A"},
    ]
    
    students.each do |(id, name, email, score, grade)|
      csv.row id, name, email, score, grade
    end
  end
end

puts File.read("/tmp/output.csv")
File.delete("/tmp/output.csv")

# Append to existing CSV
def append_to_csv(path : String, row : Array(_))
  File.open(path, "a") do |file|
    CSV.build(file) do |csv|
      csv.row *row
    end
  end
end

# CSV export ของ struct/class
struct Employee
  getter id : Int32
  getter name : String
  getter department : String
  getter salary : Float64
  
  def initialize(@id, @name, @department, @salary)
  end
  
  def to_csv_row : Array(String)
    [@id.to_s, @name, @department, @salary.to_s]
  end
end

employees = [
  Employee.new(1, "Alice", "Engineering", 75000.0),
  Employee.new(2, "Bob", "Design", 55000.0),
  Employee.new(3, "Carol", "Engineering", 85000.0),
]

result = CSV.build do |csv|
  csv.row "id", "name", "department", "salary"
  employees.each { |e| csv.row *e.to_csv_row }
end
puts result
```

---

## 6. Headers Support

```crystal
require "csv"

# ใช้ CSV::Reader กับ headers
csv_str = "name,age,city\nAlice,30,Bangkok\nBob,25,Chiang Mai\nCarol,35,Phuket\n"

CSV.new(csv_str, headers: true).each do |row|
  puts "#{row["name"]} is #{row["age"]} from #{row["city"]}"
end

# headers as Hash
records = [] of Hash(String, String)
CSV.new(csv_str, headers: true).each do |row|
  record = {} of String => String
  row.headers.each do |header|
    record[header] = row[header]
  end
  records << record
end

records.each { |r| puts r.inspect }

# ค้นหาตาม header
CSV.new(csv_str, headers: true).each do |row|
  puts row["name"] if row["city"] == "Bangkok"
end

# สร้าง Array of Hashes จาก CSV
def csv_to_records(csv_str : String) : Array(Hash(String, String))
  rows = CSV.parse(csv_str)
  return [] of Hash(String, String) if rows.empty?
  
  headers = rows.first
  rows[1..].map do |row|
    hash = {} of String => String
    headers.each_with_index { |h, i| hash[h] = row[i]? || "" }
    hash
  end
end

records = csv_to_records(csv_str)
puts records.first.inspect
puts records.find { |r| r["city"] == "Phuket" }.inspect
```

---

## 7. CSV Transformation

```crystal
require "csv"

# Transform CSV
def transform_csv(input : String, &block : Array(String) -> Array(String)?) : String
  rows = CSV.parse(input)
  return "" if rows.empty?
  
  transformed = rows.filter_map { |row| block.call(row) }
  
  CSV.build do |csv|
    transformed.each { |row| csv.row *row }
  end
end

# uppercase ทุก value
csv_data = "alice,developer,bangkok\nbob,designer,phuket\n"
result = transform_csv(csv_data) do |row|
  row.map(&.upcase)
end
puts result

# กรองและ transform
csv_data = "name,age,city\nAlice,30,Bangkok\nBob,25,Chiang Mai\nCarol,35,Phuket\n"
rows = CSV.parse(csv_data)
headers = rows.first

result = CSV.build do |csv|
  csv.row *headers
  rows[1..].each do |row|
    age = row[1].to_i
    csv.row *row if age >= 30
  end
end
puts result

# เพิ่ม computed column
csv_data = "name,price,quantity\nApple,15,100\nBanana,8,200\nCherry,35,50\n"
rows = CSV.parse(csv_data)
headers = rows.first + ["total"]

result = CSV.build do |csv|
  csv.row *headers
  rows[1..].each do |row|
    total = row[1].to_f * row[2].to_i
    csv.row *(row + [total.to_s])
  end
end
puts result

# Merge two CSVs
def merge_csv(csv1 : String, csv2 : String, join_col : Int32) : String
  rows1 = CSV.parse(csv1)
  rows2 = CSV.parse(csv2)
  
  index = {} of String => Array(String)
  rows2.each { |row| index[row[join_col]] = row }
  
  CSV.build do |csv|
    rows1.each do |row|
      if match = index[row[join_col]]?
        csv.row *(row + match[1..])
      else
        csv.row *row
      end
    end
  end
end
```

---

## 8. Large CSV Files

```crystal
require "csv"

# สร้าง large CSV สำหรับ test
def generate_test_csv(path : String, rows : Int32)
  File.open(path, "w") do |file|
    CSV.build(file) do |csv|
      csv.row "id", "name", "value", "category"
      rows.times do |i|
        csv.row i + 1, "item_#{i + 1}", rand(1000), ["A", "B", "C"].sample
      end
    end
  end
end

# Process large CSV ทีละแถว (memory efficient)
def process_large_csv(path : String) : Hash(String, Float64)
  totals = {} of String => Float64
  counts = {} of String => Int32
  
  # skip header
  first = true
  CSV.each_row(File.read(path)) do |row|
    if first
      first = false
      next
    end
    
    category = row[3]
    value = row[2].to_f
    
    totals[category] = (totals[category]? || 0.0) + value
    counts[category] = (counts[category]? || 0) + 1
  end
  
  # คำนวณ average
  totals.transform_values.with_index do |total, i|
    key = totals.keys[i]
    total / counts[key]
  end
end

# generate_test_csv("/tmp/large.csv", 10000)
# averages = process_large_csv("/tmp/large.csv")
# averages.each { |cat, avg| puts "#{cat}: #{avg.round(2)}" }

# Stream processing pattern
def csv_stream_processor(path : String, batch_size : Int32 = 100, &block : Array(Array(String)) ->)
  batch = [] of Array(String)
  first = true
  
  CSV.each_row(File.read(path)) do |row|
    if first
      first = false
      next  # skip header
    end
    
    batch << row
    if batch.size >= batch_size
      block.call(batch)
      batch.clear
    end
  end
  
  block.call(batch) unless batch.empty?
end

# ตัวอย่าง CSV validation
def validate_csv(csv_str : String, expected_columns : Int32) : Array(String)
  errors = [] of String
  row_num = 0
  
  CSV.each_row(csv_str) do |row|
    row_num += 1
    if row.size != expected_columns
      errors << "Row #{row_num}: expected #{expected_columns} columns, got #{row.size}"
    end
  end
  
  errors
end

good_csv = "a,b,c\n1,2,3\n4,5,6\n"
bad_csv = "a,b,c\n1,2,3\n4,5\n7,8,9,10\n"

puts validate_csv(good_csv, 3).inspect  # => []
puts validate_csv(bad_csv, 3).inspect   # => errors
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `CSVReport` ที่อ่าน CSV file ของ transactions และสร้าง summary report (total, average, min, max ต่อ category)

### แบบฝึกหัดที่ 2
เขียน `CSVJoiner` ที่ join สอง CSV files บน common column

### แบบฝึกหัดที่ 3
เขียน `CSVValidator` ที่ validate CSV ตาม schema (required columns, types, ranges)

### เฉลย

```crystal
require "csv"

# แบบฝึกหัดที่ 1: CSVReport
struct Stats
  property total : Float64 = 0.0
  property count : Int32 = 0
  property min : Float64 = Float64::INFINITY
  property max : Float64 = -Float64::INFINITY
  
  def add(value : Float64)
    @total += value
    @count += 1
    @min = value if value < @min
    @max = value if value > @max
  end
  
  def average : Float64
    @count > 0 ? @total / @count : 0.0
  end
end

def csv_report(csv_str : String, category_col : Int32, value_col : Int32) : String
  rows = CSV.parse(csv_str)
  return "" if rows.empty?
  
  stats = {} of String => Stats
  
  rows[1..].each do |row|
    category = row[category_col]
    value = row[value_col].to_f
    stats[category] ||= Stats.new
    stats[category].add(value)
  end
  
  CSV.build do |csv|
    csv.row "Category", "Count", "Total", "Average", "Min", "Max"
    stats.each do |cat, s|
      csv.row cat, s.count, s.total.round(2), s.average.round(2), s.min.round(2), s.max.round(2)
    end
  end
end

transactions = "date,category,amount\n2024-01-01,Food,150\n2024-01-02,Transport,80\n2024-01-03,Food,200\n2024-01-04,Entertainment,350\n2024-01-05,Transport,60\n2024-01-06,Food,120\n"
puts csv_report(transactions, 1, 2)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **CSV.parse** - parse CSV string เป็น Array(Array(String))
2. **CSV.each_row** - วนลูปแบบ memory efficient
3. **CSV.build** - สร้าง CSV output
4. **Reading files** - อ่านและประมวลผล CSV files
5. **Writing files** - เขียน CSV files
6. **Headers** - จัดการ header row
7. **Transformation** - filter, transform, merge CSV
8. **Large files** - batch processing pattern

CSV เป็น format มาตรฐานสำหรับ data exchange โดยเฉพาะกับ spreadsheet programs และ databases
