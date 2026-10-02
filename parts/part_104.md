# Part 104: Macro Methods

## บทนำ

Macro methods ใน Crystal ช่วยให้เราสร้าง methods จำนวนมากโดยอัตโนมัติ รวมถึงเข้าใจ internals ของ built-in macros เช่น `getter`, `setter`, `property`, และ `delegate`

## Generating Methods กับ Macros

```crystal
# Generate getter methods
macro make_getters(*names)
  {% for name in names %}
    def {{name.id}} : typeof(@{{name.id}})
      @{{name.id}}
    end
  {% end %}
end

class Profile
  def initialize
    @username = "guest"
    @level = 1
    @score = 0.0
  end

  make_getters :username, :level, :score
end

p = Profile.new
puts p.username  # => guest
puts p.level     # => 1
puts p.score     # => 0.0

# Generate setter methods
macro make_setters(*names)
  {% for name in names %}
    def {{name.id}}=(value)
      @{{name.id}} = value
    end
  {% end %}
end

# Generate reader+writer pair
macro make_property(name, type)
  def {{name.id}} : {{type.id}}
    @{{name.id}}
  end

  def {{name.id}}=(value : {{type.id}})
    @{{name.id}} = value
  end
end
```

## getter/setter Macro Internals

```crystal
# Crystal's built-in getter macro ทำงานแบบนี้
# (simplified version)
macro getter(*names)
  {% for name in names %}
    {% if name.is_a?(TypeDeclaration) %}
      @{{name.var}} : {{name.type}}

      def {{name.var}} : {{name.type}}
        @{{name.var}}
      end
    {% else %}
      def {{name.id}}
        @{{name.id}}
      end
    {% end %}
  {% end %}
end

# setter macro
macro setter(*names)
  {% for name in names %}
    {% if name.is_a?(TypeDeclaration) %}
      def {{name.var}}=(value : {{name.type}})
        @{{name.var}} = value
      end
    {% else %}
      def {{name.id}}=(value)
        @{{name.id}} = value
      end
    {% end %}
  {% end %}
end

# property = getter + setter
macro property(*names)
  getter {{*names}}
  setter {{*names}}
end
```

## property Macro Internals

```crystal
# Custom property ด้วย validation
macro validated_property(name, type, **opts)
  @{{name.id}} : {{type.id}} = {% if opts[:default] %} {{opts[:default]}} {% else %} uninitialized {{type.id}} {% end %}

  def {{name.id}} : {{type.id}}
    @{{name.id}}
  end

  def {{name.id}}=(value : {{type.id}})
    {% if opts[:min] %}
      raise ArgumentError.new("{{name}} must be >= {{opts[:min]}}") if value < {{opts[:min]}}
    {% end %}
    {% if opts[:max] %}
      raise ArgumentError.new("{{name}} must be <= {{opts[:max]}}") if value > {{opts[:max]}}
    {% end %}
    {% if opts[:min_length] %}
      raise ArgumentError.new("{{name}} too short") if value.responds_to?(:size) && value.size < {{opts[:min_length]}}
    {% end %}
    @{{name.id}} = value
  end
end

class Player
  validated_property name, String, min_length: 3, default: "Player"
  validated_property level, Int32, min: 1, max: 100, default: 1
  validated_property health, Float64, min: 0.0, max: 100.0, default: 100.0
end

p = Player.new

p.name = "สมชาย"  # OK
begin
  p.name = "AB"  # Error: too short
rescue e
  puts e.message
end

p.level = 50
begin
  p.level = 0  # Error: must be >= 1
rescue e
  puts e.message
end

puts p.name   # => สมชาย
puts p.level  # => 50
```

## delegate Macro

```crystal
# delegate - delegate methods ไปยัง other object
macro delegate(*methods, to object)
  {% for method in methods %}
    def {{method.id}}(*args, **kwargs)
      {{object.id}}.{{method.id}}(*args, **kwargs)
    end
  {% end %}
end

class Logger
  def log(msg : String)
    puts "[LOG] #{msg}"
  end

  def warn(msg : String)
    puts "[WARN] #{msg}"
  end

  def error(msg : String)
    puts "[ERROR] #{msg}"
  end
end

class Service
  def initialize
    @logger = Logger.new
  end

  delegate :log, :warn, :error, to: @logger

  def process(data : String)
    log "Processing: #{data}"
    data.upcase
  end
end

svc = Service.new
svc.process("hello")
svc.warn("memory high")
svc.error("connection failed")
```

## forward_missing_to Macro

```crystal
# forward_missing_to - delegate unknown methods
class Wrapper(T)
  def initialize(@inner : T)
  end

  def to_s : String
    "Wrapped(#{@inner})"
  end

  # Crystal built-in macro ที่ forward calls
  forward_missing_to @inner
end

# ใช้งาน
wrapper = Wrapper(Array(Int32)).new([1, 2, 3])
puts wrapper.to_s    # => Wrapped([1, 2, 3])
puts wrapper.size    # => 3 (forwarded to Array)
puts wrapper.first   # => 1 (forwarded to Array)
wrapper.push(4)
puts wrapper.to_a.inspect  # => [1, 2, 3, 4]
```

## Method Generation ด้วย Hash

```crystal
# Generate methods จาก Hash
macro define_status_methods(**statuses)
  {% for name, value in statuses %}
    STATUS_{{name.upcase.id}} = {{value}}

    def {{name.id}}? : Bool
      @status == {{value}}
    end

    def set_{{name.id}}!
      @status = {{value}}
      self
    end
  {% end %}

  def status_name : String
    case @status
    {% for name, value in statuses %}
      when {{value}} then {{name.stringify}}
    {% end %}
    else "unknown"
    end
  end
end

class Order
  @status : Int32 = 0

  define_status_methods(
    pending: 0,
    confirmed: 1,
    shipped: 2,
    delivered: 3,
    cancelled: 4
  )

  def status : Int32
    @status
  end
end

order = Order.new
puts order.pending?        # => true
puts order.confirmed?      # => false
puts order.status_name     # => pending

order.set_confirmed!
puts order.status_name     # => confirmed
puts order.confirmed?      # => true
```

## Lazy Initialization Macro

```crystal
# lazy - initialize ครั้งเดียวเมื่อใช้งาน
macro lazy(name, type, &block)
  @{{name.id}} : {{type.id}}?

  def {{name.id}} : {{type.id}}
    @{{name.id}} ||= begin
      {{block.body}}
    end
  end
end

class ExpensiveService
  lazy database_connection, DatabaseConnection do
    puts "Connecting to database..."
    DatabaseConnection.new
  end

  lazy cache, Cache(String, String) do
    puts "Initializing cache..."
    Cache(String, String).new
  end
end

# Mock types
class DatabaseConnection
  def query(sql : String)
    "Results for: #{sql}"
  end
end

class Cache(K, V)
  def initialize
    @data = Hash(K, V).new
  end

  def get(key : K) : V?
    @data[key]?
  end

  def set(key : K, value : V)
    @data[key] = value
  end
end

svc = ExpensiveService.new
# Database ยังไม่ connect

result = svc.database_connection.query("SELECT 1")
# => "Connecting to database..."
# => "Results for: SELECT 1"

svc.database_connection  # Already connected, no output
```

## Memoize Macro

```crystal
# memoize - cache method results
macro memoize(method_def)
  {% method = method_def %}
  {% cache_name = "__memoize_#{method.name}".id %}

  @{{cache_name}} = Hash(Tuple({{*method.args.map { |a| a.restriction }}}), {{method.return_type}}).new

  def {{method.name}}({{*method.args}}) : {{method.return_type}}
    key = { {{*method.args.map { |a| a.name.id }}} }
    unless @{{cache_name}}.has_key?(key)
      @{{cache_name}}[key] = begin
        {{method.body}}
      end
    end
    @{{cache_name}}[key]
  end
end

class FibCalculator
  def initialize
    @cache = Hash(Int32, Int64).new
  end

  def fib(n : Int32) : Int64
    return n.to_i64 if n <= 1
    @cache[n] ||= fib(n - 1) + fib(n - 2)
  end
end

calc = FibCalculator.new
puts calc.fib(50)   # => 12586269025
puts calc.fib(100)  # Fast due to memoization
```

## Builder Method Generation

```crystal
# Generate fluent builder methods
macro builder_method(name, type)
  def {{name.id}}(value : {{type.id}}) : self
    @{{name.id}} = value
    self
  end
end

macro build_method(return_type)
  abstract def build : {{return_type.id}}
end

class QueryBuilder
  @table : String = ""
  @conditions = Array(String).new
  @limit : Int32? = nil
  @offset : Int32? = nil
  @order_by : String? = nil

  builder_method table, String
  builder_method limit, Int32
  builder_method offset, Int32
  builder_method order_by, String

  def where(condition : String) : self
    @conditions << condition
    self
  end

  def build : String
    parts = ["SELECT * FROM #{@table}"]
    parts << "WHERE #{@conditions.join(" AND ")}" unless @conditions.empty?
    parts << "ORDER BY #{@order_by}" if @order_by
    parts << "LIMIT #{@limit}" if @limit
    parts << "OFFSET #{@offset}" if @offset
    parts.join(" ")
  end
end

query = QueryBuilder.new
  .table("users")
  .where("age > 18")
  .where("active = true")
  .order_by("name ASC")
  .limit(10)
  .offset(20)
  .build

puts query
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Custom getter_setter
สร้าง macro `getter_setter` ที่:
- Generate getter และ setter
- รองรับ type constraints
- รองรับ validation callbacks

### แบบฝึกหัดที่ 2: Method Tracing
สร้าง `traced` macro ที่:
- Wrap method ด้วย timing
- Log arguments และ return value
- รองรับ conditional (debug mode only)

### แบบฝึกหัดที่ 3: Decorator Pattern
สร้าง macro ที่ implement decorator pattern:
- Wrap existing methods
- Add before/after hooks
- Preserve return types

### แบบฝึกหัดที่ 4: CRUD Generator
สร้าง macro `crud_methods(model)` ที่:
- Generate find, find_by, create, update, delete
- Mock database operations
- Type-safe ทุก operation

## สรุป

Macro Methods ใน Crystal:
- **generate methods**: สร้าง methods จาก patterns
- **getter/setter/property**: internals ที่ใช้ macros
- **delegate**: forward method calls
- **forward_missing_to**: catch-all delegation
- **lazy/memoize**: optimization patterns

Key patterns:
1. ใช้ `{% for %}` เพื่อ generate multiple methods
2. ใช้ `{{name.id}}` สำหรับ method names จาก arguments
3. Validate arguments ด้วย `{% if %}` ณ compile time
4. ใช้ hygienic variables `%name` เพื่อ avoid conflicts
