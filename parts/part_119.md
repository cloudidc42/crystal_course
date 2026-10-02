# Part 119: Reactive Programming - การเขียนโปรแกรมแบบ Reactive ใน Crystal

## บทนำ

Reactive Programming เป็น paradigm ที่โปรแกรมตอบสนองต่อการเปลี่ยนแปลงของข้อมูล (data streams) โดยอัตโนมัติ แทนที่จะ poll หาข้อมูล เราจะ subscribe และได้รับการแจ้งเตือนเมื่อมีการเปลี่ยนแปลง

## Observable Pattern

### Observable พื้นฐาน

```crystal
# Observable/Observer pattern
module Observable(T)
  alias Observer = Proc(T, Nil)
  
  def initialize
    @observers = [] of Observer
    @mutex = Mutex.new
  end
  
  def subscribe(&observer : T ->)
    @mutex.synchronize { @observers << observer }
  end
  
  def unsubscribe(&observer : T ->)
    @mutex.synchronize { @observers.delete(observer) }
  end
  
  protected def notify_observers(value : T)
    observers = @mutex.synchronize { @observers.dup }
    observers.each { |obs| obs.call(value) }
  end
end

class DataStore
  include Observable(String)
  
  getter value : String = ""
  
  def initialize
    super
    @value = ""
  end
  
  def update(new_value : String)
    @value = new_value
    notify_observers(new_value)
  end
end

store = DataStore.new

store.subscribe { |v| puts "Observer 1: #{v}" }
store.subscribe { |v| puts "Observer 2 (UPPER): #{v.upcase}" }
store.subscribe { |v| puts "Observer 3 (length): #{v.size}" }

store.update("สวัสดี")
store.update("Hello World")
```

## EventEmitter

```crystal
class EventEmitter
  alias Handler = Proc(Hash(String, String), Nil)
  
  def initialize
    @handlers = Hash(String, Array(Handler)).new { |h, k| h[k] = [] of Handler }
    @once_handlers = Hash(String, Array(Handler)).new { |h, k| h[k] = [] of Handler }
    @mutex = Mutex.new
    @event_channel = Channel(Tuple(String, Hash(String, String))).new(100)
    start_dispatcher
  end
  
  def on(event : String, &handler : Hash(String, String) ->)
    @mutex.synchronize { @handlers[event] << handler }
    self
  end
  
  def once(event : String, &handler : Hash(String, String) ->)
    @mutex.synchronize { @once_handlers[event] << handler }
    self
  end
  
  def off(event : String)
    @mutex.synchronize do
      @handlers[event].clear
      @once_handlers[event].clear
    end
    self
  end
  
  def emit(event : String, data : Hash(String, String) = {} of String => String)
    @event_channel.send({event, data})
    self
  end
  
  def emit(event : String, **kwargs)
    data = kwargs.to_h.transform_keys(&.to_s).transform_values(&.to_s)
    emit(event, data)
  end
  
  private def start_dispatcher
    spawn do
      while msg = @event_channel.receive?
        event, data = msg
        
        # Regular handlers
        handlers = @mutex.synchronize { @handlers[event].dup }
        handlers.each { |h| h.call(data) }
        
        # Once handlers (ทำงานครั้งเดียว)
        once = @mutex.synchronize do
          h = @once_handlers[event].dup
          @once_handlers[event].clear
          h
        end
        once.each { |h| h.call(data) }
      end
    end
  end
end

# การใช้งาน
emitter = EventEmitter.new

emitter.on("data") { |d| puts "Received data: #{d["value"]?}" }
emitter.on("data") { |d| puts "Processing: #{d.inspect}" }
emitter.once("ready") { |d| puts "Ready! (once only)" }

emitter.emit("ready", {"status" => "ok"})
emitter.emit("data", {"value" => "Hello", "id" => "1"})
emitter.emit("ready", {"status" => "ok"}) # ไม่ทำงานอีก (once)
emitter.emit("data", {"value" => "World", "id" => "2"})

sleep 0.3.seconds
```

## Simple Reactive Streams

```crystal
# Observable Stream
class Observable(T)
  alias Subscriber = Proc(T, Nil)
  alias ErrorHandler = Proc(Exception, Nil)
  alias CompleteHandler = Proc(Nil)
  
  def initialize(&@source : (T ->, Exception ->, -> ->)  )
  end
  
  def self.create(&block : (T ->, Exception ->, -> ->) )
    new(&block)
  end
  
  def self.from(items : Array(T)) : Observable(T)
    create do |on_next, on_error, on_complete|
      begin
        items.each { |item| on_next.call(item) }
        on_complete.call
      rescue ex
        on_error.call(ex)
      end
    end
  end
  
  def self.interval(period : Time::Span) : Observable(Int32)
    create do |on_next, on_error, on_complete|
      i = 0
      loop do
        sleep period
        on_next.call(i)
        i += 1
      end
    end
  end
  
  def subscribe(
    on_next : T -> = ->(v : T) {},
    on_error : Exception -> = ->(e : Exception) { puts "Error: #{e.message}" },
    on_complete : -> = -> {}
  )
    spawn do
      @source.call(on_next, on_error, on_complete)
    end
  end
  
  def map(& transform : T -> R) : Observable(R) forall R
    Observable(R).create do |on_next, on_error, on_complete|
      subscribe(
        on_next: ->(v : T) { on_next.call(transform.call(v)) },
        on_error: on_error,
        on_complete: on_complete
      )
    end
  end
  
  def filter(&predicate : T -> Bool) : Observable(T)
    Observable(T).create do |on_next, on_error, on_complete|
      subscribe(
        on_next: ->(v : T) { on_next.call(v) if predicate.call(v) },
        on_error: on_error,
        on_complete: on_complete
      )
    end
  end
  
  def take(n : Int32) : Observable(T)
    count = Atomic(Int32).new(0)
    stop = Channel(Nil).new(1)
    
    Observable(T).create do |on_next, on_error, on_complete|
      subscribe(
        on_next: ->(v : T) {
          c = count.add(1)
          if c <= n
            on_next.call(v)
            if c == n
              on_complete.call
            end
          end
        },
        on_error: on_error,
        on_complete: on_complete
      )
    end
  end
end

# การใช้งาน
numbers = Observable(Int32).from([1, 2, 3, 4, 5, 6, 7, 8, 9, 10])

numbers
  .filter { |n| n.even? }
  .map { |n| n * n }
  .subscribe(
    on_next: ->(v : Int32) { puts "Value: #{v}" },
    on_complete: -> { puts "Complete!" }
  )

sleep 0.2.seconds
```

## Subject (Observable + Observer)

```crystal
class Subject(T)
  def initialize
    @channel = Channel(T?).new(100)
    @observers = [] of Channel(T?).new
    @mutex = Mutex.new
    @closed = false
    start_broadcast
  end
  
  def subscribe : Channel(T?)
    obs = Channel(T?).new(100)
    @mutex.synchronize { @observers << obs }
    obs
  end
  
  def next(value : T)
    @channel.send(value) unless @closed
  end
  
  def complete
    @closed = true
    @channel.send(nil)
  end
  
  private def start_broadcast
    spawn do
      while msg = @channel.receive?
        observers = @mutex.synchronize { @observers.dup }
        observers.each { |obs| obs.send(msg) rescue nil }
      end
    end
  end
end

# BehaviorSubject: เก็บค่าล่าสุดไว้เสมอ
class BehaviorSubject(T)
  getter current_value : T
  
  def initialize(@current_value : T)
    @subscribers = [] of Proc(T, Nil)
    @mutex = Mutex.new
  end
  
  def subscribe(&handler : T ->)
    @mutex.synchronize { @subscribers << handler }
    handler.call(@current_value) # ส่งค่าปัจจุบันทันที
    self
  end
  
  def next(value : T)
    @current_value = value
    notify(value)
  end
  
  private def notify(value : T)
    subs = @mutex.synchronize { @subscribers.dup }
    subs.each { |s| s.call(value) }
  end
end

# ทดสอบ BehaviorSubject
counter = BehaviorSubject(Int32).new(0)

counter.subscribe { |v| puts "Observer A: #{v}" }
counter.next(1)
counter.next(2)
counter.subscribe { |v| puts "Observer B (late): #{v}" } # ได้รับค่า 2 ทันที
counter.next(3)
```

## Reactive Store

```crystal
# Reactive Store คล้าย Redux/Vuex
class Store(State, Action)
  alias Reducer = Proc(State, Action, State)
  alias Listener = Proc(State, Nil)
  
  getter state : State
  
  def initialize(@state : State, &@reducer : State, Action -> State)
    @listeners = [] of Listener
    @mutex = Mutex.new
  end
  
  def dispatch(action : Action)
    new_state = @mutex.synchronize do
      @state = @reducer.call(@state, action)
      @state
    end
    
    notify_listeners(new_state)
  end
  
  def subscribe(&listener : State ->)
    @mutex.synchronize { @listeners << listener }
  end
  
  private def notify_listeners(state : State)
    listeners = @mutex.synchronize { @listeners.dup }
    listeners.each { |l| l.call(state) }
  end
end

# ตัวอย่าง: Counter Store
struct CounterState
  property count : Int32
  
  def initialize(@count = 0)
  end
end

enum CounterAction
  Increment
  Decrement
  Reset
end

store = Store(CounterState, CounterAction).new(CounterState.new) do |state, action|
  case action
  when .increment? then CounterState.new(state.count + 1)
  when .decrement? then CounterState.new(state.count - 1)
  when .reset?     then CounterState.new(0)
  else state
  end
end

store.subscribe { |state| puts "Count: #{state.count}" }

store.dispatch(CounterAction::Increment)
store.dispatch(CounterAction::Increment)
store.dispatch(CounterAction::Increment)
store.dispatch(CounterAction::Decrement)
store.dispatch(CounterAction::Reset)
```

## Reactive Properties

```crystal
# Computed properties ที่ reactive
class ReactiveProperty(T)
  getter value : T
  
  def initialize(@value : T)
    @listeners = [] of Proc(T, T, Nil)
    @mutex = Mutex.new
  end
  
  def value=(new_val : T)
    old_val = @value
    @value = new_val
    notify(old_val, new_val) if old_val != new_val
  end
  
  def on_change(&block : T, T ->)
    @mutex.synchronize { @listeners << block }
  end
  
  def bind_to(other : ReactiveProperty(T))
    other.on_change { |_, new_val| self.value = new_val }
  end
  
  private def notify(old_val : T, new_val : T)
    listeners = @mutex.synchronize { @listeners.dup }
    listeners.each { |l| l.call(old_val, new_val) }
  end
end

# Two-way binding simulation
name = ReactiveProperty(String).new("")
greeting = ReactiveProperty(String).new("")

name.on_change do |old, new_val|
  greeting.value = "สวัสดี, #{new_val}!"
end

greeting.on_change do |old, new_val|
  puts "Greeting changed: #{old.inspect} => #{new_val.inspect}"
end

name.value = "สมชาย"
name.value = "สมหญิง"
puts "Greeting: #{greeting.value}"
```

## Stream Operators

```crystal
class Stream(T)
  def initialize(@channel : Channel(T?))
  end
  
  def self.from(items : Array(T)) : Stream(T)
    ch = Channel(T?).new(items.size + 1)
    items.each { |item| ch.send(item) }
    ch.send(nil) # sentinel
    new(ch)
  end
  
  def map(&transform : T -> R) : Stream(R) forall R
    out_ch = Channel(R?).new(100)
    
    spawn do
      while item = @channel.receive?
        out_ch.send(transform.call(item))
      end
      out_ch.send(nil)
    end
    
    Stream(R).new(out_ch)
  end
  
  def filter(&predicate : T -> Bool) : Stream(T)
    out_ch = Channel(T?).new(100)
    
    spawn do
      while item = @channel.receive?
        out_ch.send(item) if predicate.call(item)
      end
      out_ch.send(nil)
    end
    
    Stream(T).new(out_ch)
  end
  
  def reduce(initial : R, &accumulator : R, T -> R) : R forall R
    result = initial
    while item = @channel.receive?
      result = accumulator.call(result, item)
    end
    result
  end
  
  def to_a : Array(T)
    items = [] of T
    while item = @channel.receive?
      items << item
    end
    items
  end
  
  def each(&block : T ->)
    while item = @channel.receive?
      block.call(item)
    end
  end
  
  def take(n : Int32) : Stream(T)
    out_ch = Channel(T?).new(n + 1)
    
    spawn do
      count = 0
      while item = @channel.receive?
        break if count >= n
        out_ch.send(item)
        count += 1
      end
      out_ch.send(nil)
    end
    
    Stream(T).new(out_ch)
  end
  
  def flat_map(&transform : T -> Array(R)) : Stream(R) forall R
    out_ch = Channel(R?).new(100)
    
    spawn do
      while item = @channel.receive?
        transform.call(item).each { |r| out_ch.send(r) }
      end
      out_ch.send(nil)
    end
    
    Stream(R).new(out_ch)
  end
  
  def zip(other : Stream(R)) : Stream(Tuple(T, R)) forall R
    out_ch = Channel(Tuple(T, R)?).new(100)
    
    spawn do
      loop do
        a = @channel.receive?
        b = other.@channel.receive?
        
        if a && b
          out_ch.send({a.not_nil!, b.not_nil!})
        else
          break
        end
      end
      out_ch.send(nil)
    end
    
    Stream(Tuple(T, R)).new(out_ch)
  end
end

# ทดสอบ Stream operators
stream = Stream(Int32).from((1..20).to_a)

result = stream
  .filter { |n| n % 2 == 0 }  # เลขคู่
  .map { |n| n * n }           # ยกกำลังสอง
  .take(5)                      # แค่ 5 ตัวแรก
  .to_a

puts "Result: #{result.inspect}"
# [4, 16, 36, 64, 100]

# FlatMap
words = Stream(String).from(["Hello World", "Crystal Lang", "สวัสดี โลก"])
chars = words.flat_map { |s| s.chars.map(&.to_s) }
puts "First 10 chars: #{chars.take(10).to_a.inspect}"
```

## Hot vs Cold Observables

```crystal
# Cold Observable: ทุก subscriber ได้รับข้อมูลจากต้น
class ColdObservable(T)
  def initialize(&@producer : Channel(T) ->)
  end
  
  def subscribe : Channel(T)
    ch = Channel(T).new(100)
    spawn { @producer.call(ch) }
    ch
  end
end

# Hot Observable: subscribers ทุกคนได้รับข้อมูลพร้อมกัน
class HotObservable(T)
  def initialize
    @subscribers = [] of Channel(T)
    @mutex = Mutex.new
  end
  
  def subscribe : Channel(T)
    ch = Channel(T).new(100)
    @mutex.synchronize { @subscribers << ch }
    ch
  end
  
  def emit(value : T)
    subs = @mutex.synchronize { @subscribers.dup }
    subs.each { |ch| ch.send(value) rescue nil }
  end
end

# ทดสอบ
puts "=== Cold Observable ==="
cold = ColdObservable(Int32).new do |ch|
  5.times { |i| ch.send(i) }
end

sub1 = cold.subscribe
sub2 = cold.subscribe

puts "Sub1: #{Array(Int32).new(5) { sub1.receive }}"
puts "Sub2: #{Array(Int32).new(5) { sub2.receive }}"

puts "\n=== Hot Observable ==="
hot = HotObservable(String).new
h_sub1 = hot.subscribe
h_sub2 = hot.subscribe

hot.emit("event 1")
hot.emit("event 2")

# sub2 สมัครใหม่หลังจาก 2 events
h_sub3 = hot.subscribe
hot.emit("event 3")
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Reactive Form Validation

```crystal
class FormField
  getter value : String
  getter error : String?
  getter valid : Bool
  
  def initialize(name : String)
    @name = name
    @value = ""
    @error = nil
    @valid = false
    @validators = [] of Proc(String, String?)
    @change_listeners = [] of Proc(String, Nil)
    @valid_listeners = [] of Proc(Bool, Nil)
  end
  
  def add_validator(&validator : String -> String?)
    @validators << validator
    self
  end
  
  def on_change(&listener : String ->)
    @change_listeners << listener
    self
  end
  
  def on_valid_change(&listener : Bool ->)
    @valid_listeners << listener
    self
  end
  
  def value=(new_value : String)
    @value = new_value
    validate
    @change_listeners.each { |l| l.call(new_value) }
  end
  
  private def validate
    @error = nil
    @validators.each do |validator|
      if err = validator.call(@value)
        @error = err
        break
      end
    end
    
    new_valid = @error.nil?
    if new_valid != @valid
      @valid = new_valid
      @valid_listeners.each { |l| l.call(new_valid) }
    end
  end
end

# ทดสอบ
email = FormField.new("email")
  .add_validator { |v| "ต้องไม่ว่าง" if v.empty? }
  .add_validator { |v| "ต้องมี @" unless v.includes?("@") }
  .add_validator { |v| "รูปแบบไม่ถูกต้อง" unless v.matches?(/\A[^@]+@[^@]+\.[^@]+\z/) }

email.on_change { |v| puts "Email changed: #{v}" }
email.on_valid_change { |valid| puts "Valid: #{valid}" }

email.value = "test"
puts "Error: #{email.error}"
email.value = "test@"
puts "Error: #{email.error}"
email.value = "test@example.com"
puts "Error: #{email.error}"
puts "Valid: #{email.valid}"
```

### แบบฝึกหัดที่ 2: Reactive Data Pipeline

```crystal
# สร้าง reactive data pipeline สำหรับประมวลผล stock prices
struct StockPrice
  property symbol : String
  property price : Float64
  property timestamp : Time
  
  def initialize(@symbol, @price, @timestamp = Time.local)
  end
end

class StockFeed
  def initialize
    @subject = Channel(StockPrice).new(1000)
    @running = false
  end
  
  def subscribe : Channel(StockPrice)
    @subject
  end
  
  def start_simulation
    @running = true
    
    spawn do
      symbols = ["AAPL", "GOOGL", "MSFT", "AMZN"]
      prices = {"AAPL" => 150.0, "GOOGL" => 2800.0, "MSFT" => 400.0, "AMZN" => 3500.0}
      
      while @running
        symbol = symbols.sample
        change = rand(-5.0..5.0)
        prices[symbol] = (prices[symbol]? || 100.0) + change
        
        @subject.send(StockPrice.new(symbol, prices[symbol].not_nil!))
        sleep 0.1.seconds
      end
    end
  end
  
  def stop
    @running = false
  end
end

feed = StockFeed.new
prices_ch = feed.subscribe
feed.start_simulation

# สร้าง alert สำหรับ price spikes
spawn do
  last_prices = {} of String => Float64
  
  10.times do
    if price = prices_ch.receive?
      if last = last_prices[price.symbol]?
        change_pct = ((price.price - last) / last * 100).abs
        if change_pct > 2.0
          puts "ALERT: #{price.symbol} เปลี่ยนแปลง #{change_pct.round(2)}%"
        end
      end
      
      last_prices[price.symbol] = price.price
      puts "#{price.symbol}: #{price.price.round(2)}"
    end
  end
  
  feed.stop
end

sleep 3.seconds
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Observable Pattern**: แจ้งเตือน observers เมื่อข้อมูลเปลี่ยนแปลง
2. **EventEmitter**: จัดการ events แบบ Node.js style
3. **Reactive Streams**: stream ของ data ที่มี operators (map, filter, etc.)
4. **Subject**: ทั้ง Observable และ Observer ในเวลาเดียวกัน
5. **BehaviorSubject**: เก็บค่าล่าสุดและส่งให้ subscriber ทันที
6. **Reactive Store**: state management แบบ reactive (คล้าย Redux)
7. **Hot vs Cold Observables**: ความแตกต่างระหว่าง shared และ per-subscriber streams
8. **Reactive Properties**: Two-way binding

Reactive Programming เหมาะกับ:
- UI frameworks
- Event-driven systems
- Real-time data processing
- State management
