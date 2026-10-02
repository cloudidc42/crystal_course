# Part 106: DSL ด้วย Macros

## บทนำ

Domain-Specific Language (DSL) ช่วยให้เราเขียนโค้ดในรูปแบบที่ใกล้เคียงกับ domain ที่เราทำงานอยู่ Crystal's macros เป็นเครื่องมือที่ทรงพลังสำหรับสร้าง internal DSLs

## Route DSL (เหมือน Sinatra/Kemal)

```crystal
# Router DSL
macro get(path, &block)
  @routes << {
    method: "GET",
    path: {{path}},
    handler: -> (context : HTTP::Server::Context) {
      {{block.body}}
      nil
    }
  }
end

macro post(path, &block)
  @routes << {
    method: "POST",
    path: {{path}},
    handler: -> (context : HTTP::Server::Context) {
      {{block.body}}
      nil
    }
  }
end

# Simple implementation
class Router
  alias Route = NamedTuple(method: String, path: String, handler: Proc(String, String))

  macro route(method, path, &block)
    {
      method: {{method}},
      path: {{path}},
      handler: ->(params : String) {
        {{block.body}}
      }
    }
  end

  def self.define(&block : Router -> Nil) : Router
    router = new
    block.call(router)
    router
  end

  def initialize
    @routes = Array(Route).new
  end

  def get(path : String, &handler : String -> String)
    @routes << {method: "GET", path: path, handler: handler}
  end

  def post(path : String, &handler : String -> String)
    @routes << {method: "POST", path: path, handler: handler}
  end

  def match(method : String, path : String) : String?
    @routes.each do |route|
      if route[:method] == method && route[:path] == path
        return route[:handler].call(path)
      end
    end
    nil
  end

  def list_routes
    @routes.each do |route|
      puts "#{route[:method]} #{route[:path]}"
    end
  end
end

# ใช้งาน DSL
router = Router.new
router.get("/") { |_| "Welcome!" }
router.get("/users") { |_| "List users" }
router.post("/users") { |_| "Create user" }

router.list_routes
puts router.match("GET", "/users")  # => List users
```

## Validation DSL

```crystal
# Validation DSL
module Validations
  macro validates(field, **rules)
    @[__validates__({{field}}, {{**rules}})]
  end

  macro included
    def validate! : Nil
      errors = validate
      raise "Validation failed: #{errors.join(", ")}" unless errors.empty?
    end

    def valid? : Bool
      validate.empty?
    end

    def validate : Array(String)
      errors = [] of String
      {% for ivar in @type.instance_vars %}
        {% ann = ivar.annotation(__validates__) %}
        {% if ann %}
          val = @{{ivar.name.id}}

          {% if ann[:presence] %}
            if val.nil? || (val.responds_to?(:empty?) && val.empty?)
              errors << "{{ivar.name}} cannot be blank"
            end
          {% end %}

          {% if ann[:min_length] %}
            if v = val.as?(String)
              errors << "{{ivar.name}} is too short (min {{ann[:min_length]}})" if v.size < {{ann[:min_length]}}
            end
          {% end %}

          {% if ann[:max_length] %}
            if v = val.as?(String)
              errors << "{{ivar.name}} is too long (max {{ann[:max_length]}})" if v.size > {{ann[:max_length]}}
            end
          {% end %}

          {% if ann[:min] %}
            if v = val.as?(Int32)
              errors << "{{ivar.name}} must be at least {{ann[:min]}}" if v < {{ann[:min]}}
            end
          {% end %}

          {% if ann[:max] %}
            if v = val.as?(Int32)
              errors << "{{ivar.name}} must be at most {{ann[:max]}}" if v > {{ann[:max]}}
            end
          {% end %}
        {% end %}
      {% end %}
      errors
    end
  end
end

annotation __validates__
  getter presence : Bool = false
  getter min_length : Int32?
  getter max_length : Int32?
  getter min : Int32?
  getter max : Int32?
end

class RegistrationForm
  include Validations

  @[__validates__(presence: true, min_length: 3, max_length: 50)]
  property username : String = ""

  @[__validates__(presence: true, min_length: 8)]
  property password : String = ""

  @[__validates__(min: 18, max: 120)]
  property age : Int32 = 0
end

form = RegistrationForm.new
form.username = "Jo"
form.age = 15

errors = form.validate
errors.each { |e| puts e }
puts form.valid?  # => false

form.username = "สมชาย"
form.password = "secure123"
form.age = 25
puts form.valid?  # => true
```

## State Machine DSL

```crystal
# State machine DSL
module StateMachine
  macro state(*names)
    enum State
      {% for name in names %}
        {{name.id.camelcase}}
      {% end %}
    end

    getter state : State = State::{{names.first.id.camelcase}}

    {% for name in names %}
      def {{name.id}}? : Bool
        @state == State::{{name.id.camelcase}}
      end
    {% end %}
  end

  macro transition(from, to, on event_name)
    def {{event_name.id}}! : Bool
      if @state == State::{{from.id.camelcase}}
        @state = State::{{to.id.camelcase}}
        on_{{event_name.id}} if responds_to?(:on_{{event_name.id}})
        true
      else
        false
      end
    end
  end
end

class TrafficLight
  include StateMachine

  state :red, :yellow, :green

  transition from: :red, to: :green, on: :go
  transition from: :green, to: :yellow, on: :slow_down
  transition from: :yellow, to: :red, on: :stop

  def description : String
    case @state
    when State::Red    then "หยุด! ไฟแดง"
    when State::Yellow then "เตรียมหยุด ไฟเหลือง"
    when State::Green  then "ไปได้ ไฟเขียว"
    else "สัญญาณไม่รู้จัก"
    end
  end
end

light = TrafficLight.new
puts light.description  # => หยุด! ไฟแดง
puts light.red?         # => true

light.go!
puts light.description  # => ไปได้ ไฟเขียว
puts light.green?       # => true

light.slow_down!
puts light.description  # => เตรียมหยุด ไฟเหลือง

light.stop!
puts light.description  # => หยุด! ไฟแดง

puts light.go!          # => true (valid transition)
puts light.stop!        # => false (invalid from green)
```

## Macro Hooks

```crystal
# Macro hooks - run code at specific compilation points

# inherited - เรียกเมื่อ class ถูก inherit
module Registerable
  macro included
    @@registry = [] of String

    def self.registry : Array(String)
      @@registry
    end
  end

  macro inherited
    @@registry << {{@type.name.stringify}}
    puts "Registered: {{@type.name}}"  # ณ compile time
  end
end

class BaseModel
  include Registerable
end

class User < BaseModel
end

class Product < BaseModel
end

class Order < BaseModel
end

# ณ runtime
puts BaseModel.registry.inspect  # => ["User", "Product", "Order"]
```

## Config DSL

```crystal
# Configuration DSL
module Configurable
  macro config_option(name, type, default)
    @@_config_{{name.id}} : {{type.id}} = {{default}}

    def self.{{name.id}} : {{type.id}}
      @@_config_{{name.id}}
    end

    def self.{{name.id}}=(value : {{type.id}})
      @@_config_{{name.id}} = value
    end
  end

  macro configure(&block)
    {{yield}}
  end
end

module AppConfig
  include Configurable

  config_option database_url, String, "postgres://localhost/dev"
  config_option max_connections, Int32, 10
  config_option timeout_seconds, Int32, 30
  config_option debug_mode, Bool, false
  config_option log_level, String, "info"
end

# ใช้งาน DSL
AppConfig.database_url = "postgres://prod-server/myapp"
AppConfig.max_connections = 50
AppConfig.debug_mode = true

puts AppConfig.database_url    # => postgres://prod-server/myapp
puts AppConfig.max_connections # => 50
puts AppConfig.debug_mode      # => true
puts AppConfig.log_level       # => info (default)
```

## Builder DSL

```crystal
# HTML Builder DSL
class HtmlBuilder
  @buffer = IO::Memory.new
  @indent = 0

  def tag(name : String, **attrs, &block)
    write_indent
    @buffer << "<#{name}"
    attrs.each do |key, val|
      @buffer << " #{key}=\"#{val}\""
    end
    @buffer << ">"
    @indent += 2
    block.call
    @indent -= 2
    write_indent
    @buffer << "</#{name}>\n"
  end

  def text(content : String)
    write_indent
    @buffer << content << "\n"
  end

  def to_html : String
    @buffer.to_s
  end

  private def write_indent
    @buffer << " " * @indent
  end
end

# DSL macros
macro html_tag(name)
  def {{name.id}}(**attrs, &block)
    tag({{name.stringify}}, **attrs, &block)
  end
end

class Html < HtmlBuilder
  html_tag div
  html_tag p
  html_tag h1
  html_tag h2
  html_tag ul
  html_tag li
  html_tag span
  html_tag a
end

builder = Html.new

builder.div(class: "container") do
  builder.h1 { builder.text "สวัสดี Crystal DSL" }
  builder.p { builder.text "นี่คือ HTML DSL" }
  builder.ul do
    ["item1", "item2", "item3"].each do |item|
      builder.li { builder.text item }
    end
  end
end

puts builder.to_html
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Test Framework DSL
สร้าง mini test framework:
- `describe("name") { ... }`
- `context("name") { ... }`
- `it("should ...") { ... }`
- `expect(val).to eq(expected)`
- `expect(val).to be_truthy`

### แบบฝึกหัดที่ 2: Query DSL
สร้าง SQL query DSL:
- `from("users").where("age > ?", 18).select("name", "email").limit(10)`
- Type-safe ด้วยการ use macros
- Prevent SQL injection

### แบบฝึกหัดที่ 3: Pipeline DSL
สร้าง data pipeline DSL:
- `source.transform { ... }.filter { ... }.sink`
- Lazy evaluation
- Error handling

### แบบฝึกหัดที่ 4: Event DSL
สร้าง event-driven DSL:
- `on(:user_created) { |event| ... }`
- `emit(:user_created, data: user)`
- Event priority
- Async events

## สรุป

DSL ด้วย Macros ใน Crystal:
- **Internal DSL**: ใช้ Ruby/Crystal syntax ที่เป็น natural
- **Macro hooks**: `inherited`, `included`, `extended`
- **Declarative syntax**: annotations + macros
- **Zero overhead**: code generated ณ compile time

Key techniques:
1. ใช้ block-based syntax สำหรับ scope
2. ใช้ named parameters สำหรับ options
3. Combine macros กับ inheritance
4. ใช้ macro hooks สำหรับ registration patterns
