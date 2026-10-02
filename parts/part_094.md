# Part 94: Proc Types ใน Crystal

## บทนำ

Proc ใน Crystal เป็น first-class function object ที่สามารถเก็บไว้ในตัวแปร ส่งเป็น argument หรือ return จาก function ได้

## Proc พื้นฐาน

```crystal
# สร้าง Proc ด้วย -> syntax
square = ->(x : Int32) { x * x }
puts square.call(5)  # => 25

# Proc ที่ไม่รับ parameter
greet = -> { puts "สวัสดีครับ!" }
greet.call

# Proc ที่รับหลาย parameters
add = ->(a : Int32, b : Int32) { a + b }
puts add.call(3, 4)  # => 7

# Type annotation
multiply : Proc(Int32, Int32, Int32) = ->(a : Int32, b : Int32) { a * b }
puts multiply.call(3, 4)  # => 12
```

## Proc Types

```crystal
# Proc(T1, T2, ..., ReturnType)
# Format: Proc(ArgTypes..., ReturnType)

# ไม่รับ argument, return String
no_arg : Proc(String) = -> { "hello" }
puts no_arg.call  # => hello

# รับ Int32, return Int32
int_to_int : Proc(Int32, Int32) = ->(x : Int32) { x * 2 }
puts int_to_int.call(5)  # => 10

# รับ String, return Int32
str_to_int : Proc(String, Int32) = ->(s : String) { s.size }
puts str_to_int.call("hello")  # => 5

# รับ Int32 และ String, return Bool
two_arg : Proc(Int32, String, Bool) = ->(n : Int32, s : String) {
  s.size > n
}
puts two_arg.call(3, "hello")  # => true
```

## Method Reference เป็น Proc

```crystal
# แปลง method เป็น Proc ด้วย ->
def double(x : Int32) : Int32
  x * 2
end

# Method reference
proc = ->double(Int32)
puts proc.call(5)    # => 10
puts proc.class      # => Proc(Int32, Int32)

# Instance method reference
class Calculator
  def initialize(@factor : Int32)
  end

  def multiply(x : Int32) : Int32
    x * @factor
  end
end

calc = Calculator.new(3)
multiply_by_3 = ->calc.multiply(Int32)
puts multiply_by_3.call(5)   # => 15
puts multiply_by_3.call(10)  # => 30

# Built-in method references
to_s_proc = ->((42).to_s)
puts to_s_proc.class  # Proc(String)
```

## Proc.call และ .()

```crystal
square = ->(x : Int32) { x * x }

# วิธีที่ 1: .call()
puts square.call(5)  # => 25

# วิธีที่ 2: .() shorthand
puts square.(5)  # => 25

# Proc ใน array
operations = [
  ->(x : Int32) { x + 1 },
  ->(x : Int32) { x * 2 },
  ->(x : Int32) { x ** 2 },
]

value = 3
operations.each do |op|
  puts op.call(value)
end
# => 4, 6, 9
```

## Curry

```crystal
# Curry - แปลง multi-arg function เป็น chain ของ single-arg functions
add = ->(a : Int32, b : Int32) { a + b }

# curry ส่ง curried proc กลับมา
curried_add = add.curry

add5 = curried_add.call(5)   # => Proc ที่รับ b
puts add5.call(3)             # => 8
puts add5.call(10)            # => 15

# 3-argument curry
multiply_and_add = ->(a : Int32, b : Int32, c : Int32) { a * b + c }
curried = multiply_and_add.curry

double_then_add = curried.call(2)        # fix a=2
double_then_add5 = double_then_add.call(5)  # fix b=5
puts double_then_add5.call(3)            # => 2*5+3 = 13
puts double_then_add5.call(7)            # => 2*5+7 = 17
```

## Partial Application

```crystal
# Partial application pattern
def partial(proc : Proc(A, B, C), first_arg : A) : Proc(B, C) forall A, B, C
  ->(second : B) { proc.call(first_arg, second) }
end

add = ->(a : Int32, b : Int32) { a + b }
add10 = partial(add, 10)
puts add10.call(5)   # => 15
puts add10.call(20)  # => 30

# ด้วย curry
multiply = ->(a : Int32, b : Int32) { a * b }
double = multiply.curry.call(2)
triple = multiply.curry.call(3)

puts [1, 2, 3, 4, 5].map { |x| double.call(x) }.inspect  # [2, 4, 6, 8, 10]
puts [1, 2, 3, 4, 5].map { |x| triple.call(x) }.inspect  # [3, 6, 9, 12, 15]
```

## Composing Procs

```crystal
# Function composition
def compose(f : Proc(B, C), g : Proc(A, B)) : Proc(A, C) forall A, B, C
  ->(x : A) { f.call(g.call(x)) }
end

double = ->(x : Int32) { x * 2 }
increment = ->(x : Int32) { x + 1 }

# double ∘ increment = double(increment(x))
double_after_increment = compose(double, increment)
puts double_after_increment.call(5)  # => 12 (5+1=6, 6*2=12)

# increment ∘ double = increment(double(x))
increment_after_double = compose(increment, double)
puts increment_after_double.call(5)  # => 11 (5*2=10, 10+1=11)

# Pipeline ของหลาย functions
def pipeline(*funcs : Proc(Int32, Int32)) : Proc(Int32, Int32)
  ->(x : Int32) {
    funcs.reduce(x) { |acc, f| f.call(acc) }
  }
end

process = pipeline(
  ->(x : Int32) { x + 1 },
  ->(x : Int32) { x * 2 },
  ->(x : Int32) { x - 3 },
)

puts process.call(5)  # => ((5+1)*2)-3 = 9
```

## Proc เป็น Higher-Order Functions

```crystal
# map, filter, reduce ด้วย explicit Proc
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# แปลง block เป็น Proc
square_proc = ->(x : Int32) { x * x }
even_proc = ->(x : Int32) { x.even? }
sum_proc = ->(acc : Int32, val : Int32) { acc + val }

squares = numbers.map { |x| square_proc.call(x) }
evens = numbers.select { |x| even_proc.call(x) }
total = numbers.reduce(0) { |acc, val| sum_proc.call(acc, val) }

puts squares.inspect  # => [1, 4, 9, 16, 25, 36, 49, 64, 81, 100]
puts evens.inspect    # => [2, 4, 6, 8, 10]
puts total            # => 55

# ใช้ & เพื่อส่ง Proc เป็น block
puts numbers.map(&square_proc).inspect
puts numbers.select(&even_proc).inspect
```

## Memoization ด้วย Proc

```crystal
# Memoize expensive computation
def memoize(f : Proc(Int32, Int32)) : Proc(Int32, Int32)
  cache = Hash(Int32, Int32).new
  ->(x : Int32) {
    cache[x] ||= f.call(x)
  }
end

# Fibonacci ที่ slow (exponential)
fib_slow = ->(n : Int32) {
  n <= 1 ? n : fib_slow.call(n - 1) + fib_slow.call(n - 2)
}

# Memoized version
fib_memo = memoize(fib_slow)

# Note: ใน real code ควรใช้ explicit memoization
# ตัวอย่างนี้แสดง concept

# Memoize สำหรับ String key
class Memoizer(A, B)
  def initialize(@func : Proc(A, B))
    @cache = Hash(A, B).new
  end

  def call(arg : A) : B
    @cache[arg] ||= @func.call(arg)
  end
end

expensive = Memoizer(String, Int32).new(->(s : String) {
  puts "Computing for #{s}..."
  s.size * 2
})

puts expensive.call("hello")  # Computes
puts expensive.call("hello")  # Cache hit
puts expensive.call("world")  # Computes
```

## Event Handlers กับ Proc

```crystal
# Event system ด้วย Proc
class EventEmitter(T)
  alias Handler = Proc(T, Nil)

  def initialize
    @handlers = Array(Handler).new
  end

  def on(handler : Handler)
    @handlers << handler
    handler  # return handler เพื่อ remove ได้
  end

  def off(handler : Handler)
    @handlers.delete(handler)
  end

  def emit(event : T)
    @handlers.each(&.call(event))
  end
end

# ใช้งาน
emitter = EventEmitter(String).new

handler1 = emitter.on(->(msg : String) {
  puts "Handler 1: #{msg}"
  nil
})

handler2 = emitter.on(->(msg : String) {
  puts "Handler 2: #{msg.upcase}"
  nil
})

emitter.emit("hello")  # Both handlers fire
emitter.off(handler1)
emitter.emit("world")  # Only handler2 fires
```

## Proc สำหรับ Strategy Pattern

```crystal
# Strategy pattern ด้วย Proc
alias SortStrategy(T) = Proc(T, T, Int32)

class Sorter(T)
  def initialize(@strategy : SortStrategy(T))
  end

  def sort(items : Array(T)) : Array(T)
    items.sort { |a, b| @strategy.call(a, b) }
  end
end

# Sorting strategies
ascending = SortStrategy(Int32).new { |a, b| a <=> b }
descending = SortStrategy(Int32).new { |a, b| b <=> a }
by_remainder = SortStrategy(Int32).new { |a, b| (a % 3) <=> (b % 3) }

numbers = [5, 3, 8, 1, 9, 2, 7, 4, 6]

asc_sorter = Sorter(Int32).new(ascending)
puts asc_sorter.sort(numbers).inspect  # => [1, 2, 3, 4, 5, 6, 7, 8, 9]

desc_sorter = Sorter(Int32).new(descending)
puts desc_sorter.sort(numbers).inspect  # => [9, 8, 7, 6, 5, 4, 3, 2, 1]

rem_sorter = Sorter(Int32).new(by_remainder)
puts rem_sorter.sort(numbers).inspect  # sorted by remainder when div by 3
```

## Proc กับ Lazy Evaluation

```crystal
# Lazy evaluation ด้วย Proc
class Lazy(T)
  def initialize(&@computation : -> T)
    @computed = false
    @value = uninitialized T
  end

  def value : T
    unless @computed
      @value = @computation.call
      @computed = true
    end
    @value
  end

  def computed? : Bool
    @computed
  end
end

# สร้าง lazy value
expensive = Lazy(Int32).new {
  puts "Computing expensive value..."
  sleep 0.001  # simulate expensive computation
  42
}

puts "Before access"
puts expensive.computed?  # => false
puts expensive.value      # => "Computing expensive value...", 42
puts expensive.computed?  # => true
puts expensive.value      # => 42 (ไม่ compute อีก)
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Function Pipeline Builder
สร้าง `Pipeline(A, B)` class ที่:
- สร้าง pipeline จาก sequence ของ Procs
- รองรับ `map`, `filter`, `reduce`
- Type-safe ทุก step

### แบบฝึกหัดที่ 2: Retry with Backoff
สร้าง `retry_with_backoff` function ที่:
- รับ Proc ที่อาจ fail
- Retry ตามจำนวนครั้งที่กำหนด
- Exponential backoff
- Return Result(T, Exception)

### แบบฝึกหัดที่ 3: Middleware Chain
สร้าง HTTP middleware chain:
- `Middleware = Proc(Request, Proc(Request, Response), Response)`
- Chain middlewares
- รองรับ logging, auth, rate-limiting

### แบบฝึกหัดที่ 4: Reactive Properties
สร้าง `Reactive(T)` ที่:
- เก็บ value
- เรียก observers เมื่อ value เปลี่ยน
- รองรับ computed properties ด้วย Proc

## สรุป

Proc types ใน Crystal:
- **First-class**: เก็บในตัวแปร, ส่งเป็น argument, return ได้
- **Type parameters**: `Proc(ArgTypes..., ReturnType)` - explicit types
- **Curry**: แปลง multi-arg เป็น series ของ single-arg Procs
- **Composition**: รวม Procs เป็น new Proc
- **Method references**: แปลง named method เป็น Proc

Key uses:
1. Callbacks และ event handlers
2. Strategy pattern
3. Higher-order functions
4. Lazy evaluation
5. Memoization
