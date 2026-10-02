# Part 91: Enums ใน Crystal

## บทนำ

Enum (enumeration) ใน Crystal เป็น type ที่มี set ของค่าที่กำหนดไว้ล่วงหน้า Crystal enums เป็น type-safe และมีประสิทธิภาพสูงเพราะ compile เป็น integer ภายใน

## Enum พื้นฐาน

```crystal
# Enum ง่ายๆ
enum Direction
  North
  South
  East
  West
end

# ใช้งาน
dir = Direction::North
puts dir          # => North
puts dir.to_s     # => North
puts dir.value    # => 0 (auto-assigned starting from 0)

# Enum values
puts Direction::North.value  # => 0
puts Direction::South.value  # => 1
puts Direction::East.value   # => 2
puts Direction::West.value   # => 3

# Type checking
puts dir.is_a?(Direction)  # => true
puts dir == Direction::North  # => true
puts dir == Direction::South  # => false
```

## Custom Values

```crystal
# กำหนด values เอง
enum HTTPStatus
  OK           = 200
  Created      = 201
  NoContent    = 204
  BadRequest   = 400
  Unauthorized = 401
  Forbidden    = 403
  NotFound     = 404
  ServerError  = 500
end

status = HTTPStatus::NotFound
puts status.value   # => 404
puts status.to_s    # => NotFound

# แปลง value กลับเป็น enum
code = 200
if status = HTTPStatus.from_value?(code)
  puts "Status: #{status}"  # => Status: OK
else
  puts "Unknown status code: #{code}"
end

# จะ raise ถ้า value ไม่ถูกต้อง
begin
  invalid = HTTPStatus.from_value(999)
rescue ex : ArgumentError
  puts "Error: #{ex.message}"
end
```

## Enum กับ Underlying Type

```crystal
# กำหนด underlying type
enum Priority : UInt8
  Low      = 1
  Medium   = 2
  High     = 3
  Critical = 4
end

p = Priority::High
puts p.value       # => 3
puts p.value.class # => UInt8

# enum ขนาดใหญ่
enum Permission : UInt64
  Read    = 1_u64
  Write   = 2_u64
  Execute = 4_u64
  Admin   = 8_u64
end
```

## @[Flags] Enum

```crystal
# Flags enum - แต่ละ bit เป็น flag อิสระ
@[Flags]
enum FilePermission
  Read
  Write
  Execute
end

# ค่าเริ่มต้นสำหรับ flags: 1, 2, 4, 8, ...
puts FilePermission::Read.value    # => 1
puts FilePermission::Write.value   # => 2
puts FilePermission::Execute.value # => 4

# รวม flags ด้วย |
perms = FilePermission::Read | FilePermission::Write
puts perms           # => Read | Write
puts perms.includes?(FilePermission::Read)    # => true
puts perms.includes?(FilePermission::Execute) # => false

# None และ All ถูกสร้างอัตโนมัติสำหรับ @[Flags]
puts FilePermission::None.value  # => 0
puts FilePermission::All.value   # => 7 (1|2|4)

# Operations
all = FilePermission::All
no_write = all ^ FilePermission::Write  # XOR
puts no_write  # => Read | Execute

# Intersection
rw = FilePermission::Read | FilePermission::Write
r_only = rw & FilePermission::Read
puts r_only  # => Read
```

## @[Flags] Enum ขั้นสูง

```crystal
@[Flags]
enum UserRole
  Guest
  Member
  Moderator
  Admin
  SuperAdmin
end

# Assign roles
user_roles = UserRole::Member | UserRole::Moderator
puts user_roles  # => Member | Moderator

# Check permissions
def can_delete_post?(roles : UserRole) : Bool
  roles.includes?(UserRole::Moderator) ||
  roles.includes?(UserRole::Admin) ||
  roles.includes?(UserRole::SuperAdmin)
end

def can_manage_users?(roles : UserRole) : Bool
  roles.includes?(UserRole::Admin) ||
  roles.includes?(UserRole::SuperAdmin)
end

puts can_delete_post?(user_roles)    # => true (Moderator)
puts can_manage_users?(user_roles)   # => false

admin_roles = UserRole::Admin | UserRole::Moderator
puts can_manage_users?(admin_roles)  # => true
```

## Enum Methods

```crystal
enum Color
  Red
  Green
  Blue
  Yellow
  Purple

  # Method บน enum
  def complementary : Color
    case self
    when Red    then Green
    when Green  then Red
    when Blue   then Yellow
    when Yellow then Blue
    when Purple then Purple
    end
  end

  def hex : String
    case self
    when Red    then "#FF0000"
    when Green  then "#00FF00"
    when Blue   then "#0000FF"
    when Yellow then "#FFFF00"
    when Purple then "#800080"
    end
  end

  def rgb : Tuple(UInt8, UInt8, UInt8)
    case self
    when Red    then {255_u8, 0_u8, 0_u8}
    when Green  then {0_u8, 255_u8, 0_u8}
    when Blue   then {0_u8, 0_u8, 255_u8}
    when Yellow then {255_u8, 255_u8, 0_u8}
    when Purple then {128_u8, 0_u8, 128_u8}
    end
  end

  def dark? : Bool
    r, g, b = rgb
    (0.299 * r + 0.587 * g + 0.114 * b) < 128
  end
end

color = Color::Blue
puts color.hex            # => #0000FF
puts color.complementary  # => Yellow
puts color.dark?          # => true

# Iterate ทุก color
Color.each do |c|
  puts "#{c}: #{c.hex}"
end
```

## enum.each, enum.names, enum.values

```crystal
enum Season
  Spring = 1
  Summer = 2
  Autumn = 3
  Winter = 4
end

# .names - array ของชื่อ
puts Season.names.inspect   # => ["Spring", "Summer", "Autumn", "Winter"]

# .values - array ของ values
puts Season.values.inspect  # => [Spring, Summer, Autumn, Winter] (enum instances)

# .each - iterate
Season.each do |season|
  puts "#{season.value}: #{season}"
end

# นับจำนวน
puts "Seasons: #{Season.names.size}"

# Map ไปเป็น array ของค่า
season_names_th = {
  Season::Spring => "ใบไม้ผลิ",
  Season::Summer => "ฤดูร้อน",
  Season::Autumn => "ใบไม้ร่วง",
  Season::Winter => "ฤดูหนาว",
}

Season.each do |s|
  puts "#{s} = #{season_names_th[s]}"
end
```

## from_value และ parse

```crystal
enum DayOfWeek
  Monday    = 1
  Tuesday   = 2
  Wednesday = 3
  Thursday  = 4
  Friday    = 5
  Saturday  = 6
  Sunday    = 7
end

# from_value - จาก Int ไปเป็น enum (raise ถ้าไม่พบ)
begin
  day = DayOfWeek.from_value(3)
  puts day  # => Wednesday
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

# from_value? - safe version
day_opt = DayOfWeek.from_value?(8)
puts day_opt.inspect  # => nil

day_ok = DayOfWeek.from_value?(5)
puts day_ok.inspect   # => Friday

# parse - จาก String ไปเป็น enum
begin
  friday = DayOfWeek.parse("Friday")
  puts friday.value  # => 5
rescue ArgumentError => e
  puts "Error: #{e.message}"
end

# parse? - safe version
invalid = DayOfWeek.parse?("Holiday")
puts invalid.inspect  # => nil

valid = DayOfWeek.parse?("Monday")
puts valid.inspect    # => Monday

# Case insensitive parse
puts DayOfWeek.parse?("monday").inspect  # => nil (case sensitive!)
```

## Case กับ Enum

```crystal
enum TrafficLight
  Red
  Yellow
  Green
end

def can_proceed?(light : TrafficLight) : Bool
  case light
  when TrafficLight::Green  then true
  when TrafficLight::Yellow then false  # หยุดถ้าทำได้
  when TrafficLight::Red    then false
  end
end

def next_light(current : TrafficLight) : TrafficLight
  case current
  when TrafficLight::Red    then TrafficLight::Green
  when TrafficLight::Green  then TrafficLight::Yellow
  when TrafficLight::Yellow then TrafficLight::Red
  end
end

light = TrafficLight::Red
5.times do
  puts "#{light}: #{can_proceed?(light) ? "ไป" : "หยุด"}"
  light = next_light(light)
end
```

## Enum ใน Struct/Class

```crystal
struct Task
  enum Status
    Todo
    InProgress
    Done
    Cancelled
  end

  enum Priority
    Low
    Medium
    High
  end

  getter id : Int32
  getter title : String
  getter status : Status
  getter priority : Priority
  getter created_at : Time

  def initialize(@id, @title, @status = Status::Todo, @priority = Priority::Medium)
    @created_at = Time.utc
  end

  def start! : Task
    Task.new(@id, @title, Status::InProgress, @priority)
  end

  def complete! : Task
    Task.new(@id, @title, Status::Done, @priority)
  end

  def cancel! : Task
    Task.new(@id, @title, Status::Cancelled, @priority)
  end

  def active? : Bool
    @status == Status::InProgress
  end

  def done? : Bool
    @status == Status::Done || @status == Status::Cancelled
  end

  def to_s : String
    "[#{@priority}] #{@title} (#{@status})"
  end
end

tasks = [
  Task.new(1, "เขียน tests", Task::Priority::High),
  Task.new(2, "อัพเดท docs", Task::Priority::Low),
  Task.new(3, "Fix bugs", Task::Priority::High),
]

# กรอง tasks ตาม priority
high_priority = tasks.select { |t| t.priority == Task::Priority::High }
high_priority.each { |t| puts t }

# Sort ตาม priority
sorted = tasks.sort_by { |t| t.priority.value }
sorted.each { |t| puts t }
```

## Enum กับ JSON

```crystal
require "json"

enum UserStatus
  Active
  Inactive
  Banned
  Pending
end

struct User
  include JSON::Serializable

  property id : Int32
  property name : String

  @[JSON::Field(converter: UserStatus::ValueConverter)]
  property status : UserStatus

  def initialize(@id, @name, @status = UserStatus::Active)
  end
end

# Custom converter
module UserStatus::ValueConverter
  def self.to_json(value : UserStatus, json : JSON::Builder)
    json.string(value.to_s.downcase)
  end

  def self.from_json(pull : JSON::PullParser) : UserStatus
    str = pull.read_string
    UserStatus.parse(str.capitalize)
  end
end

user = User.new(1, "สมชาย", UserStatus::Active)
json = user.to_json
puts json  # => {"id":1,"name":"สมชาย","status":"active"}

restored = User.from_json(json)
puts restored.status  # => Active
```

## Enum สำหรับ Configuration

```crystal
@[Flags]
enum LogLevel
  Debug
  Info
  Warning
  Error
  Critical
end

class Logger
  def initialize(@level : LogLevel = LogLevel::Info | LogLevel::Warning | LogLevel::Error | LogLevel::Critical)
  end

  def log(message : String, level : LogLevel)
    return unless @level.includes?(level)

    prefix = case level
    when LogLevel::Debug    then "[DEBUG]"
    when LogLevel::Info     then "[INFO]"
    when LogLevel::Warning  then "[WARN]"
    when LogLevel::Error    then "[ERROR]"
    when LogLevel::Critical then "[CRIT]"
    else                         "[LOG]"
    end

    puts "#{prefix} #{message}"
  end

  def debug(msg : String)   = log(msg, LogLevel::Debug)
  def info(msg : String)    = log(msg, LogLevel::Info)
  def warning(msg : String) = log(msg, LogLevel::Warning)
  def error(msg : String)   = log(msg, LogLevel::Error)
  def critical(msg : String) = log(msg, LogLevel::Critical)
end

# ใช้งาน
logger = Logger.new(LogLevel::Debug | LogLevel::Info | LogLevel::Error)
logger.debug("Starting application")
logger.info("Server listening on port 8080")
logger.warning("High memory usage")  # ไม่แสดง (Warning ไม่อยู่ใน level)
logger.error("Database connection failed")
logger.critical("System crash")  # ไม่แสดง (Critical ไม่อยู่ใน level)
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Card Game
สร้าง enum สำหรับ playing cards:
- `Suit`: Hearts, Diamonds, Clubs, Spades
- `Rank`: Ace, Two, ..., King
- ใช้ enums เพื่อสร้าง deck และเล่น simple card game

### แบบฝึกหัดที่ 2: Permission System
ใช้ `@[Flags]` enum สร้าง permission system:
- รองรับ multiple roles และ permissions
- Check permissions อย่าง efficient
- Serialize/deserialize permissions

### แบบฝึกหัดที่ 3: State Machine
สร้าง order state machine:
- Enums สำหรับ states และ events
- Transition table
- Validate transitions

### แบบฝึกหัดที่ 4: Configuration Builder
ใช้ enums สร้าง configuration system:
- Database type (PostgreSQL, MySQL, SQLite)
- Environment (Development, Staging, Production)
- Feature flags ด้วย @[Flags]

## สรุป

Enums ใน Crystal มีคุณสมบัติสำคัญ:
- **Type-safe**: ป้องกัน invalid values
- **Methods**: เพิ่ม behavior บน enum values
- **@[Flags]**: bitwise operations สำหรับ flag sets
- **Iteration**: each, names, values
- **Parsing**: from_value, from_value?, parse, parse?

Best practices:
1. ใช้ PascalCase สำหรับชื่อ enum และ members
2. ใช้ @[Flags] สำหรับ bitmask permissions/options
3. เพิ่ม methods บน enum แทนที่จะใช้ global functions
4. ระวังว่า parse เป็น case-sensitive
