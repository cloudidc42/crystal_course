# Part 195: Design Patterns ใน Crystal

## บทนำ

Design Patterns คือ solutions ที่ reusable สำหรับ problems ที่พบบ่อยใน software design Crystal รองรับ patterns เหล่านี้ได้ดีด้วย modules, generics, และ blocks

## Singleton Pattern

```crystal
# Singleton: มีแค่ instance เดียวของ class
module Singleton
  macro included
    @@instance : self? = nil
    @@mutex = Mutex.new

    def self.instance : self
      @@instance ||= @@mutex.synchronize { @@instance ||= new }
    end

    private def initialize
    end
  end
end

class Config
  include Singleton

  getter database_url : String
  getter port : Int32
  getter environment : String

  def initialize
    @database_url = ENV["DATABASE_URL"]? || "postgres://localhost:5432/myapp"
    @port = (ENV["PORT"]? || "8080").to_i
    @environment = ENV["APP_ENV"]? || "development"
  end
end

# ใช้งาน
config = Config.instance
puts config.port
puts config.database_url

# ได้ instance เดิมเสมอ
puts Config.instance.object_id == config.object_id  # true
```

## Factory Pattern

```crystal
# Factory: สร้าง objects โดยไม่ต้อง specify concrete class
abstract class Notifier
  abstract def send(to : String, message : String)
end

class EmailNotifier < Notifier
  def send(to : String, message : String)
    puts "Email to #{to}: #{message}"
    # ส่ง email จริงๆ
  end
end

class SMSNotifier < Notifier
  def send(to : String, message : String)
    puts "SMS to #{to}: #{message}"
    # ส่ง SMS จริงๆ
  end
end

class SlackNotifier < Notifier
  def send(to : String, message : String)
    puts "Slack to #{to}: #{message}"
    # ส่ง Slack message จริงๆ
  end
end

# Factory method
module NotifierFactory
  def self.create(type : String) : Notifier
    case type
    when "email" then EmailNotifier.new
    when "sms"   then SMSNotifier.new
    when "slack" then SlackNotifier.new
    else raise ArgumentError.new("Unknown notifier type: #{type}")
    end
  end
end

# ใช้งาน
notifier = NotifierFactory.create("email")
notifier.send("user@example.com", "Your order is ready!")
```

## Observer Pattern

```crystal
# Observer: notify multiple objects เมื่อ state เปลี่ยน
module Observable(T)
  def self.included(base)
    base.class_eval do
      @observers : Array(Proc(T, Nil)) = [] of Proc(T, Nil)

      def subscribe(&handler : T ->)
        @observers << handler
      end

      def unsubscribe(&handler : T ->)
        @observers.delete(handler)
      end

      private def notify_observers(event : T)
        @observers.each(&.call(event))
      end
    end
  end
end

# Event types
record OrderCreated, order_id : String, total : Float64, user_id : String
record OrderShipped, order_id : String, tracking_number : String
record OrderCancelled, order_id : String, reason : String

alias OrderEvent = OrderCreated | OrderShipped | OrderCancelled

class OrderService
  @observers : Array(Proc(OrderEvent, Nil)) = [] of Proc(OrderEvent, Nil)

  def subscribe(&handler : OrderEvent ->)
    @observers << handler
  end

  def create_order(user_id : String, total : Float64) : String
    order_id = Random::Secure.hex(8)
    # ... create in database ...
    notify(OrderCreated.new(order_id, total, user_id))
    order_id
  end

  def ship_order(order_id : String, tracking : String)
    # ... update database ...
    notify(OrderShipped.new(order_id, tracking))
  end

  def cancel_order(order_id : String, reason : String)
    # ... update database ...
    notify(OrderCancelled.new(order_id, reason))
  end

  private def notify(event : OrderEvent)
    @observers.each(&.call(event))
  end
end

# Observers
orders = OrderService.new

orders.subscribe do |event|
  case event
  when OrderCreated
    puts "Email: Order #{event.order_id} created for user #{event.user_id}"
  when OrderShipped
    puts "Email: Order #{event.order_id} shipped, tracking: #{event.tracking_number}"
  when OrderCancelled
    puts "Email: Order #{event.order_id} cancelled: #{event.reason}"
  end
end

orders.subscribe do |event|
  case event
  when OrderCreated
    puts "Analytics: New order #{event.order_id} worth $#{event.total}"
  end
end

id = orders.create_order("user-123", 99.99)
orders.ship_order(id, "TRK-12345")
```

## Strategy Pattern

```crystal
# Strategy: เปลี่ยน algorithm ใน runtime
abstract class SortStrategy(T)
  abstract def sort(data : Array(T)) : Array(T)
end

class BubbleSort(T) < SortStrategy(T)
  def sort(data : Array(T)) : Array(T)
    arr = data.dup
    (arr.size - 1).times do |i|
      (arr.size - i - 1).times do |j|
        arr[j], arr[j + 1] = arr[j + 1], arr[j] if arr[j] > arr[j + 1]
      end
    end
    arr
  end
end

class QuickSort(T) < SortStrategy(T)
  def sort(data : Array(T)) : Array(T)
    return data if data.size <= 1
    pivot = data[data.size // 2]
    left = data.select { |x| x < pivot }
    middle = data.select { |x| x == pivot }
    right = data.select { |x| x > pivot }
    sort(left) + middle + sort(right)
  end
end

class BuiltinSort(T) < SortStrategy(T)
  def sort(data : Array(T)) : Array(T)
    data.sort
  end
end

class DataProcessor(T)
  @strategy : SortStrategy(T)

  def initialize(@strategy : SortStrategy(T))
  end

  def strategy=(new_strategy : SortStrategy(T))
    @strategy = new_strategy
  end

  def process(data : Array(T)) : Array(T)
    @strategy.sort(data)
  end
end

data = [5, 2, 8, 1, 9, 3, 7, 4, 6]
processor = DataProcessor(Int32).new(BuiltinSort(Int32).new)
puts processor.process(data).inspect

processor.strategy = QuickSort(Int32).new
puts processor.process(data).inspect
```

## Decorator Pattern

```crystal
# Decorator: เพิ่ม behavior ให้ object โดยไม่ modify class
abstract class Logger
  abstract def log(message : String)
end

class ConsoleLogger < Logger
  def log(message : String)
    puts message
  end
end

class TimestampDecorator < Logger
  def initialize(@logger : Logger)
  end

  def log(message : String)
    @logger.log("[#{Time.local.to_rfc3339}] #{message}")
  end
end

class LevelDecorator < Logger
  def initialize(@logger : Logger, @level : String = "INFO")
  end

  def log(message : String)
    @logger.log("[#{@level}] #{message}")
  end
end

class FileDecorator < Logger
  def initialize(@logger : Logger, @path : String)
    @file = File.open(@path, "a")
  end

  def log(message : String)
    @logger.log(message)  # log to wrapped logger
    @file.puts(message)   # also write to file
    @file.flush
  end
end

# Compose decorators
logger = ConsoleLogger.new
logger = TimestampDecorator.new(logger)
logger = LevelDecorator.new(logger, "WARN")

logger.log("Something happened!")
# Output: [WARN] [2025-01-01T12:00:00Z] Something happened!
```

## Command Pattern

```crystal
# Command: encapsulate request เป็น object (รองรับ undo/redo)
abstract class Command
  abstract def execute
  abstract def undo
end

class AddTextCommand < Command
  def initialize(@editor : TextEditor, @text : String, @position : Int32)
  end

  def execute
    @editor.insert(@text, @position)
  end

  def undo
    @editor.delete(@position, @text.size)
  end
end

class DeleteTextCommand < Command
  @deleted_text : String = ""

  def initialize(@editor : TextEditor, @start : Int32, @length : Int32)
  end

  def execute
    @deleted_text = @editor.get(@start, @length)
    @editor.delete(@start, @length)
  end

  def undo
    @editor.insert(@deleted_text, @start)
  end
end

class TextEditor
  @content : String = ""

  def insert(text : String, pos : Int32)
    @content = @content[0, pos] + text + @content[pos..]
  end

  def delete(start : Int32, length : Int32)
    @content = @content[0, start] + @content[start + length..]
  end

  def get(start : Int32, length : Int32) : String
    @content[start, length]
  end

  def content : String
    @content
  end
end

class CommandHistory
  @history : Array(Command) = [] of Command
  @redo_stack : Array(Command) = [] of Command

  def execute(cmd : Command)
    cmd.execute
    @history << cmd
    @redo_stack.clear
  end

  def undo
    return if @history.empty?
    cmd = @history.pop
    cmd.undo
    @redo_stack << cmd
  end

  def redo
    return if @redo_stack.empty?
    cmd = @redo_stack.pop
    cmd.execute
    @history << cmd
  end
end

editor = TextEditor.new
history = CommandHistory.new

history.execute(AddTextCommand.new(editor, "Hello", 0))
history.execute(AddTextCommand.new(editor, " World", 5))
puts editor.content  # "Hello World"

history.undo
puts editor.content  # "Hello"

history.redo
puts editor.content  # "Hello World"
```

## Builder Pattern

```crystal
# Builder: สร้าง complex objects ทีละขั้นตอน
class QueryBuilder
  @table : String = ""
  @conditions : Array(String) = [] of String
  @columns : Array(String) = [] of String
  @order_by : String? = nil
  @limit : Int32? = nil
  @offset : Int32? = nil
  @joins : Array(String) = [] of String
  @params : Array(DB::Any) = [] of DB::Any

  def from(table : String) : self
    @table = table
    self
  end

  def select(*columns : String) : self
    @columns = columns.to_a
    self
  end

  def where(condition : String, *params) : self
    @conditions << condition
    @params.concat(params.to_a)
    self
  end

  def join(table : String, on : String) : self
    @joins << "JOIN #{table} ON #{on}"
    self
  end

  def order(column : String, direction : String = "ASC") : self
    @order_by = "#{column} #{direction}"
    self
  end

  def limit(n : Int32) : self
    @limit = n
    self
  end

  def offset(n : Int32) : self
    @offset = n
    self
  end

  def build : {String, Array(DB::Any)}
    cols = @columns.empty? ? "*" : @columns.join(", ")
    sql = "SELECT #{cols} FROM #{@table}"
    sql += " #{@joins.join(" ")}" unless @joins.empty?
    sql += " WHERE #{@conditions.join(" AND ")}" unless @conditions.empty?
    sql += " ORDER BY #{@order_by}" if @order_by
    sql += " LIMIT #{@limit}" if @limit
    sql += " OFFSET #{@offset}" if @offset
    {sql, @params}
  end
end

# ใช้งาน
sql, params = QueryBuilder.new
  .from("users")
  .select("id", "name", "email")
  .join("orders", "orders.user_id = users.id")
  .where("users.active = $1", true)
  .where("orders.total > $2", 100.0)
  .order("users.name")
  .limit(20)
  .offset(40)
  .build

puts sql
puts params.inspect
```

## แบบฝึกหัด

1. สร้าง Payment processor ที่ใช้ Strategy pattern สำหรับ Stripe, PayPal, และ PromptPay
2. Implement Observer ใน Event system ที่ support async handlers
3. สร้าง HTTP middleware chain ด้วย Decorator pattern
4. เพิ่ม Builder สำหรับ configuration objects ที่ validate ตอน build

## สรุป

Design Patterns ใน Crystal:
- **Singleton**: module include ที่ thread-safe ด้วย Mutex
- **Factory**: method ที่ return abstract type ตาม input
- **Observer**: event-driven notification pattern
- **Strategy**: inject algorithm ผ่าน abstract class
- **Decorator**: wrap object เพิ่ม behavior โดยไม่ modify
- **Command**: encapsulate actions สนับสนุน undo/redo
- **Builder**: chainable API สำหรับ complex object construction
- Crystal's blocks, generics, and modules ช่วยให้ patterns กระชับกว่า Java/C++
