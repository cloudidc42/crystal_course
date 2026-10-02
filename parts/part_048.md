# Part 48: Tuples ใน Crystal

## บทนำ

Tuple เป็น immutable collection ที่มีขนาดคงที่ (fixed-size) เก็บข้อมูล heterogeneous ได้ (หลาย type) ต่างจาก Array ตรงที่:
1. **Fixed size**: ไม่สามารถเพิ่มหรือลบ elements
2. **Immutable**: ไม่สามารถแก้ไข elements
3. **Type-safe**: แต่ละ index มี type ที่รู้ตอน compile time
4. **Stack allocated**: มี performance ดีกว่า

---

## 48.1 การสร้าง Tuple

```crystal
# วิธีที่ 1: Tuple literal
point = {1.0, 2.0, 3.0}
person = {"Alice", 30, "Bangkok"}
rgb = {255_u8, 128_u8, 0_u8}

# วิธีที่ 2: ระบุ type ชัดเจน
coords : Tuple(Float64, Float64) = {3.0, 4.0}
data : Tuple(String, Int32, Bool) = {"Alice", 25, true}

# วิธีที่ 3: Empty tuple
empty = Tuple.new

# Type ของ Tuple
p = {1, "hello", 3.14, true}
puts typeof(p)  # => Tuple(Int32, String, Float64, Bool)

# Tuple ใน function return
def min_max(arr : Array(Int32)) : {Int32, Int32}
  {arr.min, arr.max}
end

result = min_max([3, 1, 4, 1, 5, 9])
puts result.inspect  # => {1, 9}
```

---

## 48.2 การเข้าถึง Elements

```crystal
t = {"Alice", 30, "Bangkok", true}

# เข้าถึงด้วย index
puts t[0]   # => "Alice"
puts t[1]   # => 30
puts t[2]   # => "Bangkok"
puts t[3]   # => true

# Negative index
puts t[-1]  # => true
puts t[-2]  # => "Bangkok"

# Type safety: แต่ละ index รู้ type ณ compile time
name = t[0]   # name เป็น String
age = t[1]    # age เป็น Int32
city = t[2]   # city เป็น String
active = t[3] # active เป็น Bool

puts name.upcase  # => "ALICE" (String method ทำงานได้ทันที)
puts age + 5      # => 35 (Int32 method ทำงานได้)

# at compile time รู้ว่าแต่ละ index เป็น type ใด
person = {"Alice", 30}
typeof(person[0])  # String
typeof(person[1])  # Int32
```

---

## 48.3 Destructuring

```crystal
# Destructuring assignment
name, age, city = {"Alice", 30, "Bangkok"}
puts name  # => "Alice"
puts age   # => 30
puts city  # => "Bangkok"

# ใน functions
def get_user : {String, Int32, String}
  {"Bob", 25, "Chiang Mai"}
end

username, user_age, user_city = get_user
puts "#{username} is #{user_age} years old from #{user_city}"

# Skip values ด้วย _
_, score, _ = {1, 99, "winner"}
puts score  # => 99

# Nested destructuring
nested = {{1, 2}, {3, 4}}
(a, b), (c, d) = nested
puts "#{a}, #{b}, #{c}, #{d}"  # => 1, 2, 3, 4

# Destructuring ใน each
pairs = [{1, "one"}, {2, "two"}, {3, "three"}]
pairs.each do |num, word|
  puts "#{num} = #{word}"
end
```

---

## 48.4 Tuple(T1, T2) Type Annotation

```crystal
# Function ที่รับ Tuple type ที่ระบุ
def swap(pair : Tuple(Int32, String)) : Tuple(String, Int32)
  {pair[1], pair[0]}
end

result = swap({42, "hello"})
puts result.inspect       # => {"hello", 42}
puts typeof(result)       # => Tuple(String, Int32)

# Generic function กับ Tuple
def first_element(t : Tuple) 
  t[0]
end

puts first_element({"Alice", 30})  # => "Alice"
puts first_element({1, 2, 3})      # => 1

# Tuple เป็น return type
def divide_with_remainder(a : Int32, b : Int32) : {Int32, Int32}
  {a / b, a % b}
end

quotient, remainder = divide_with_remainder(17, 5)
puts "17 ÷ 5 = #{quotient} remainder #{remainder}"
# => 17 ÷ 5 = 3 remainder 2
```

---

## 48.5 size และ each

```crystal
t = {1, "hello", 3.14, true, :symbol}

puts t.size  # => 5

# each
t.each do |element|
  puts "#{element} (#{element.class})"
end
# 1 (Int32)
# hello (String)
# 3.14 (Float64)
# true (Bool)
# symbol (Symbol)

# each_with_index
t.each_with_index do |elem, i|
  puts "  [#{i}] #{elem}"
end
```

---

## 48.6 map

```crystal
nums = {1, 2, 3, 4, 5}

# map บน Tuple
doubled = nums.map { |n| n * 2 }
puts doubled.inspect  # => {2, 4, 6, 8, 10}

# map กับ type change (คืน Array ไม่ใช่ Tuple)
strings = nums.map { |n| n.to_s }
puts strings.inspect       # => ["1", "2", "3", "4", "5"]
puts strings.class         # => Array(String)

# Tuple map กับ mixed types คืน Array of union
mixed = {1, "hello", 3.14}
result = mixed.map { |e| e.to_s }
puts result.inspect  # => ["1", "hello", "3.14"]
```

---

## 48.7 to_a

```crystal
t = {1, 2, 3, 4, 5}

arr = t.to_a
puts arr.inspect        # => [1, 2, 3, 4, 5]
puts arr.class          # => Array(Int32)

# Mixed type tuple to_a
mixed = {1, "two", 3.0}
arr2 = mixed.to_a
puts arr2.inspect       # => [1, "two", 3.0]
puts typeof(arr2)       # => Array(Int32 | String | Float64)

# แปลง array กลับเป็น tuple
arr_back = [1, 2, 3]
# Crystal ไม่มี direct array to tuple conversion
# ต้องทำ manually
```

---

## 48.8 Comparison

```crystal
t1 = {1, 2, 3}
t2 = {1, 2, 3}
t3 = {1, 2, 4}
t4 = {1, 2}  # ขนาดต่างกัน

puts t1 == t2  # => true
puts t1 == t3  # => false
# puts t1 == t4  # Error! ขนาดต่างกันไม่สามารถเปรียบเทียบได้

# Comparison (<=>)
puts ({1, 2, 3} <=> {1, 2, 4})  # => -1 (น้อยกว่า)
puts ({1, 2, 3} <=> {1, 2, 3})  # => 0  (เท่ากัน)
puts ({1, 3, 3} <=> {1, 2, 4})  # => 1  (มากกว่า)

# Sort array of tuples
coords = [{3, 1}, {1, 4}, {2, 2}, {1, 1}]
sorted = coords.sort
puts sorted.inspect  # => [{1, 1}, {1, 4}, {2, 2}, {3, 1}]
```

---

## 48.9 ตัวอย่างการใช้งาน Tuple

### Return Multiple Values

```crystal
def parse_date(date_str : String) : {Int32, Int32, Int32}
  parts = date_str.split("-")
  {parts[0].to_i, parts[1].to_i, parts[2].to_i}
end

year, month, day = parse_date("2024-03-15")
puts "Year: #{year}, Month: #{month}, Day: #{day}"

# Range check
def clamp_to_range(value : Int32, min : Int32, max : Int32) : {Int32, Bool}
  if value < min
    {min, true}
  elsif value > max
    {max, true}
  else
    {value, false}
  end
end

result, was_clamped = clamp_to_range(150, 0, 100)
puts "Value: #{result}, Was clamped: #{was_clamped}"
# => Value: 100, Was clamped: true
```

### Coordinates และ Geometry

```crystal
alias Point2D = Tuple(Float64, Float64)
alias Point3D = Tuple(Float64, Float64, Float64)
alias Color = Tuple(UInt8, UInt8, UInt8)

def distance(p1 : Point2D, p2 : Point2D) : Float64
  x1, y1 = p1
  x2, y2 = p2
  Math.sqrt((x2 - x1)**2 + (y2 - y1)**2)
end

def midpoint(p1 : Point2D, p2 : Point2D) : Point2D
  x1, y1 = p1
  x2, y2 = p2
  {(x1 + x2) / 2, (y1 + y2) / 2}
end

def blend_colors(c1 : Color, c2 : Color, ratio : Float64 = 0.5) : Color
  r1, g1, b1 = c1
  r2, g2, b2 = c2
  {
    ((r1 * (1 - ratio) + r2 * ratio).to_u8),
    ((g1 * (1 - ratio) + g2 * ratio).to_u8),
    ((b1 * (1 - ratio) + b2 * ratio).to_u8)
  }
end

p1 = {0.0, 0.0}
p2 = {6.0, 8.0}
puts "Distance: #{distance(p1, p2)}"   # => 10.0
puts "Midpoint: #{midpoint(p1, p2)}"   # => {3.0, 4.0}

red : Color = {255_u8, 0_u8, 0_u8}
blue : Color = {0_u8, 0_u8, 255_u8}
purple = blend_colors(red, blue)
puts "Purple: #{purple.inspect}"  # => {127, 0, 127}
```

### State Machine

```crystal
# Tuple เป็น state + event
alias State = String
alias Event = String
alias Transition = Tuple(State, Event)

class StateMachine
  def initialize(@initial_state : State)
    @current = @initial_state
    @transitions = {} of Transition => State
  end
  
  def add_transition(from : State, event : Event, to : State)
    @transitions[{from, event}] = to
  end
  
  def trigger(event : Event) : Bool
    next_state = @transitions[{@current, event}]?
    if next_state
      @current = next_state
      true
    else
      false
    end
  end
  
  def current : State
    @current
  end
end

# Traffic light state machine
light = StateMachine.new("red")
light.add_transition("red", "go", "green")
light.add_transition("green", "slow", "yellow")
light.add_transition("yellow", "stop", "red")

puts light.current  # => "red"
light.trigger("go")
puts light.current  # => "green"
light.trigger("slow")
puts light.current  # => "yellow"
light.trigger("stop")
puts light.current  # => "red"
puts light.trigger("invalid")  # => false
```

---

## 48.10 Named Tuple Overview

Named Tuples เป็น Tuples ที่มีชื่อ keys แทน index

```crystal
# Named Tuple ใช้ symbol keys
person = {name: "Alice", age: 30, city: "Bangkok"}

puts person[:name]  # => "Alice"
puts person[:age]   # => 30

# Type annotation
user : NamedTuple(name: String, age: Int32) = {name: "Bob", age: 25}

# เปรียบเทียบ Tuple กับ NamedTuple
tuple = {"Alice", 30}         # Tuple(String, Int32)
named = {name: "Alice", age: 30}  # NamedTuple(name: String, age: Int32)

# Named tuple ชัดเจนกว่าเมื่อมีหลาย fields
config = {
  host: "localhost",
  port: 8080,
  debug: false,
  max_connections: 100
}

puts config[:host]  # => "localhost"
puts config[:port]  # => 8080
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Tuple-based Records

```crystal
alias Employee = Tuple(Int32, String, String, Float64)
# (id, name, department, salary)

def create_employee(id : Int32, name : String, dept : String, salary : Float64) : Employee
  {id, name, dept, salary}
end

def employee_name(emp : Employee) : String
  emp[1]
end

def employee_salary(emp : Employee) : Float64
  emp[3]
end

def give_raise(emp : Employee, percent : Float64) : Employee
  id, name, dept, salary = emp
  {id, name, dept, salary * (1 + percent / 100)}
end

employees = [
  create_employee(1, "Alice", "Engineering", 80000.0),
  create_employee(2, "Bob", "Marketing", 60000.0),
  create_employee(3, "Charlie", "Engineering", 75000.0)
]

# Sort by salary
sorted = employees.sort_by { |emp| -employee_salary(emp) }
sorted.each do |emp|
  id, name, dept, salary = emp
  puts "#{id}. #{name} (#{dept}): $#{salary}"
end

# Give 10% raise to Engineering
engineers = employees.select { |emp| emp[2] == "Engineering" }
raised = engineers.map { |emp| give_raise(emp, 10.0) }
puts "\nAfter raise:"
raised.each do |emp|
  puts "#{emp[1]}: $#{emp[3]}"
end
```

### แบบฝึกหัดที่ 2: Function ที่คืน Tuple

```crystal
def statistical_summary(data : Array(Float64)) : {Float64, Float64, Float64, Float64}
  sorted = data.sort
  n = data.size.to_f
  mean = data.sum / n
  variance = data.sum { |x| (x - mean) ** 2 } / n
  std_dev = Math.sqrt(variance)
  median = n.even? ?
    (sorted[(n/2 - 1).to_i] + sorted[(n/2).to_i]) / 2 :
    sorted[(n/2).to_i]
  {mean, median, std_dev, data.max - data.min}
end

data = [23.5, 18.2, 31.7, 25.0, 19.8, 28.3, 22.1, 35.6, 21.4, 27.9]
mean, median, std_dev, range = statistical_summary(data)

puts "Mean: #{mean.round(2)}"
puts "Median: #{median.round(2)}"
puts "Std Dev: #{std_dev.round(2)}"
puts "Range: #{range.round(2)}"
```

### แบบฝึกหัดที่ 3: Tuple ใน collection

```crystal
# Database query result simulation
type Row = Tuple(Int32, String, String, Float64)

rows : Array(Row) = [
  {1, "Alice", "Engineering", 85000.0},
  {2, "Bob", "Marketing", 65000.0},
  {3, "Charlie", "Engineering", 75000.0},
  {4, "Dave", "HR", 55000.0},
  {5, "Eve", "Engineering", 90000.0}
]

# Query: Engineering department sorted by salary
engineering = rows
  .select { |row| row[2] == "Engineering" }
  .sort_by { |row| -row[3] }

puts "Engineering Department:"
engineering.each do |id, name, dept, salary|
  puts "  #{name}: $#{salary}"
end

# Average salary by department
dept_avg = rows.group_by { |row| row[2] }
               .transform_values { |dept_rows|
                 total = dept_rows.sum { |row| row[3] }
                 (total / dept_rows.size).round(2)
               }

puts "\nAverage Salary by Department:"
dept_avg.sort_by { |_, avg| -avg }.each do |dept, avg|
  puts "  #{dept}: $#{avg}"
end
```

---

## สรุป

| Feature | Array | Tuple |
|---------|-------|-------|
| Size | Dynamic | Fixed |
| Mutability | Mutable | Immutable |
| Types | Homogeneous | Heterogeneous |
| Type checking | Runtime | Compile-time |
| Memory | Heap | Stack |
| Use case | Collections | Multiple return values, records |

**เมื่อใช้ Tuple:**
- Return multiple values จาก function
- เก็บข้อมูลชั่วคราวที่มีโครงสร้างคงที่
- Pattern matching กับ fixed-structure data
- Key ใน Hash (เป็น pair)
- Coordinates, points, pairs

**เมื่อใช้ Array:**
- Collection ที่ขนาดเปลี่ยนได้
- ข้อมูล homogeneous
- ต้องการ modify หรือ add/remove elements

---

*ต่อไป: Part 49 - NamedTuples*
