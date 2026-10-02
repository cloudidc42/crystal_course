# Part 109: Macro Hooks

## บทนำ

Macro hooks ใน Crystal เป็น special macros ที่เรียกโดยอัตโนมัติ ณ specific compilation events เช่น เมื่อ class ถูก inherit หรือ module ถูก include

## inherited Hook

```crystal
# inherited - เรียกเมื่อ class ถูก subclass
class Plugin
  macro inherited
    puts "New plugin registered: {{@type.name}}"
    @@all_plugins = [] of Plugin.class unless defined?(@@all_plugins)
    @@all_plugins << {{@type}}
  end

  def self.all_plugins : Array(Plugin.class)
    @@all_plugins ||= [] of Plugin.class
  end

  abstract def run : String
end

class DatabasePlugin < Plugin
  def run : String
    "Database connected"
  end
end

class CachePlugin < Plugin
  def run : String
    "Cache initialized"
  end
end

class LogPlugin < Plugin
  def run : String
    "Logger started"
  end
end

# ทุก plugin ถูก register อัตโนมัติ
plugins = [DatabasePlugin.new, CachePlugin.new, LogPlugin.new]
plugins.each { |p| puts p.run }
```

## included Hook

```crystal
# included - เรียกเมื่อ module ถูก include
module Trackable
  macro included
    puts "Trackable included in: {{@type.name}}"

    @@instances = [] of {{@type.name.id}}

    def self.all : Array({{@type.name.id}})
      @@instances
    end

    def self.count : Int32
      @@instances.size
    end
  end

  def initialize
    @@instances << self
    super
  end
end

class User
  include Trackable

  getter name : String

  def initialize(@name)
    super()
  end
end

class Product
  include Trackable

  getter title : String

  def initialize(@title)
    super()
  end
end

u1 = User.new("สมชาย")
u2 = User.new("สมหญิง")
p1 = Product.new("โน้ตบุ๊ค")

puts User.count    # => 2
puts Product.count # => 1
puts User.all.map(&.name).inspect   # => ["สมชาย", "สมหญิง"]
```

## extended Hook

```crystal
# extended - เรียกเมื่อ module ถูก extend
module ClassMethods
  macro extended
    puts "ClassMethods extended by: {{@type.name}}"

    def self.class_name : String
      {{@type.name.stringify}}
    end

    def self.created_at : Time
      @@created_at ||= Time.utc
    end
  end

  def self.describe : String
    "#{class_name} (extended module)"
  end
end

class Widget
  extend ClassMethods
end

class Gadget
  extend ClassMethods
end

puts Widget.class_name   # => Widget
puts Gadget.class_name   # => Gadget
```

## method_added Hook

```crystal
# method_added - เรียกเมื่อมีการเพิ่ม method ใหม่
module Logged
  macro method_added(method)
    {% if !method.name.starts_with?("_") && method.visibility == :public %}
      puts "Method added to {{@type.name}}: {{method.name}}"
    {% end %}
  end
end

class Service
  include Logged

  def process(data : String) : String
    data.upcase
  end

  def validate(input : String) : Bool
    !input.empty?
  end

  private def _helper : String
    "helper"
  end
end

# Output ณ compile time:
# Method added to Service: process
# Method added to Service: validate
```

## finished Hook

```crystal
# finished - เรียกหลังจาก class/module definition เสร็จสิ้น
module AutoValidation
  macro finished
    puts "Finished processing: {{@type.name}}"
    {% for ivar in @type.instance_vars %}
      {% if ivar.has_annotation?(Required) %}
        puts "  Required field: {{ivar.name}}"
      {% end %}
    {% end %}
  end
end

annotation Required; end

class Form
  include AutoValidation

  @[Required]
  property name : String = ""

  property optional_field : String = ""

  @[Required]
  property email : String = ""
end

# Output ณ compile time:
# Finished processing: Form
#   Required field: name
#   Required field: email
```

## ตัวอย่างการใช้งานจริง: Plugin System

```crystal
# Plugin system ด้วย inherited hook
abstract class Command
  @@registry = Hash(String, Command.class).new

  macro inherited
    @@registry[{{@type.name.stringify.downcase.gsub(/command$/, "")}}] = {{@type}}
  end

  def self.registry : Hash(String, Command.class)
    @@registry
  end

  def self.find(name : String) : Command.class?
    @@registry[name]?
  end

  abstract def execute(args : Array(String)) : String
  abstract def description : String
end

class HelpCommand < Command
  def execute(args : Array(String)) : String
    Command.registry.map { |name, cmd|
      "  #{name}: #{cmd.new.description}"
    }.join("\n")
  end

  def description : String
    "Show this help message"
  end
end

class EchoCommand < Command
  def execute(args : Array(String)) : String
    args.join(" ")
  end

  def description : String
    "Echo the arguments"
  end
end

class VersionCommand < Command
  def execute(args : Array(String)) : String
    "Crystal #{Crystal::VERSION}"
  end

  def description : String
    "Show version"
  end
end

# ใช้งาน
def run_command(name : String, args : Array(String)) : String
  if cmd_class = Command.find(name)
    cmd_class.new.execute(args)
  else
    "Unknown command: #{name}"
  end
end

puts run_command("echo", ["hello", "world"])
puts run_command("version", [] of String)
puts run_command("help", [] of String)
```

## Observer Pattern ด้วย Hooks

```crystal
# Auto-register observers ด้วย included hook
module Observable
  macro included
    @@observers = [] of {{@type.name.id}}Observer

    def self.add_observer(observer : {{@type.name.id}}Observer)
      @@observers << observer
    end

    def notify_observers(event : Symbol, data : String)
      @@observers.each(&.update(event, data))
    end
  end
end

abstract class Observer(T)
  abstract def update(event : Symbol, data : String)
end

# Define observer interface
abstract class UserObserver < Observer(User)
end

class User
  include Observable

  property name : String

  def initialize(@name)
  end

  def update_name(new_name : String)
    old_name = @name
    @name = new_name
    notify_observers(:name_changed, "#{old_name} -> #{new_name}")
  end
end

class AuditObserver < UserObserver
  def update(event : Symbol, data : String)
    puts "[AUDIT] #{event}: #{data}"
  end
end

class NotificationObserver < UserObserver
  def update(event : Symbol, data : String)
    puts "[NOTIFY] Sending notification: #{data}"
  end
end

user = User.new("สมชาย")
User.add_observer(AuditObserver.new)
User.add_observer(NotificationObserver.new)

user.update_name("สมหญิง")
# => [AUDIT] name_changed: สมชาย -> สมหญิง
# => [NOTIFY] Sending notification: สมชาย -> สมหญิง
```

## Mixin ด้วย included Hook

```crystal
# Automatic mixin setup
module Pagination
  macro included
    property page : Int32 = 1
    property per_page : Int32 = 20

    def paginate(collection : Array(T)) : Array(T) forall T
      start = (page - 1) * per_page
      collection[start, per_page] || [] of T
    end

    def total_pages(total_count : Int32) : Int32
      (total_count.to_f / per_page).ceil.to_i
    end

    def next_page? : Bool
      # would check against total in real implementation
      true
    end
  end
end

module Sortable
  macro included
    property sort_by : String = "id"
    property sort_direction : String = "ASC"

    def sort_params : String
      "ORDER BY #{sort_by} #{sort_direction}"
    end
  end
end

class UserIndex
  include Pagination
  include Sortable

  def initialize
    # Sets defaults from included hooks
  end
end

ui = UserIndex.new
ui.page = 2
ui.per_page = 10
ui.sort_by = "name"

users = (1..100).map { |i| "User #{i}" }
page_users = ui.paginate(users)
puts page_users.inspect  # Users 11-20
puts ui.sort_params      # => ORDER BY name ASC
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Event Bus ด้วย Hooks
สร้าง event bus ที่:
- ใช้ `inherited` hook สำหรับ auto-registration
- ใช้ `method_added` สำหรับ detect event handlers
- Type-safe dispatch

### แบบฝึกหัดที่ 2: Dependency Injection
สร้าง DI container ที่:
- `included` hook สำหรับ register dependencies
- Auto-resolve ด้วย `finished` hook
- Support singletons และ transients

### แบบฝึกหัดที่ 3: Schema Evolution Tracking
ใช้ hooks เพื่อ track schema changes:
- `method_added` จับ field additions
- Generate migration scripts
- Version tracking

### แบบฝึกหัดที่ 4: Benchmark Registry
สร้าง automatic benchmark registry:
- `method_added` จับ methods ที่มี `@[Benchmark]`
- Auto-run benchmarks
- Generate report

## สรุป

Macro Hooks ใน Crystal:
- **inherited**: เมื่อ class ถูก subclass
- **included**: เมื่อ module ถูก include
- **extended**: เมื่อ module ถูก extend
- **method_added**: เมื่อมี method ใหม่
- **finished**: หลัง type definition เสร็จ

Key use cases:
1. Plugin/extension registration
2. Automatic observer setup
3. Mixin initialization
4. Compile-time code analysis
