# Part 181: LLVM Optimizations ใน Crystal

## บทนำ

Crystal ใช้ LLVM เป็น backend สำหรับ code generation LLVM มี optimization passes ที่ทรงพลังมาก เมื่อ compile ด้วย `--release` flag Crystal จะเปิด optimizations เหล่านี้ให้อัตโนมัติ

## --release Flag

```bash
# Debug build (default): เร็วในการ compile, มี debug info
crystal build src/app.cr

# Release build: ช้าในการ compile, แต่ binary เร็วกว่ามาก
crystal build --release src/app.cr

# Release + static binary
crystal build --release --static src/app.cr

# ดู LLVM IR ที่ generate
crystal build --release --emit llvm-ir -o /dev/null src/app.cr

# ดู assembly ที่ generate
crystal build --release --emit asm -o /dev/null src/app.cr

# ดู optimization อื่นๆ
crystal build --release --no-debug src/app.cr  # ไม่มี debug info = เล็กกว่า

# Build ด้วย specific optimization level
crystal build --release -Dpreview_mt src/app.cr  # multi-threading
```

## สิ่งที่ LLVM ทำ

```crystal
# 1. CONSTANT FOLDING: คำนวณ constants ตอน compile time
def circle_area(r : Float64) : Float64
  Math::PI * r * r  # Math::PI เป็น constant
end
# LLVM แทนที่ด้วย r * r * 3.141592653589793 (ไม่มี runtime lookup)

# 2. DEAD CODE ELIMINATION
def maybe_used(n : Int32) : Int32
  x = n * 100  # ถ้า x ไม่ถูกใช้ จะถูกลบออก
  y = n + 1
  y  # แค่ y ที่ return
end

# 3. INLINING อัตโนมัติ
@[AlwaysInline]
def add(a : Int32, b : Int32) : Int32
  a + b
end

def compute(x : Int32) : Int32
  add(x, 10)  # LLVM inline เป็น x + 10 โดยตรง
end

# 4. STRENGTH REDUCTION
def multiply_by_8(n : Int32) : Int32
  n * 8  # LLVM แปลงเป็น n << 3 (bit shift เร็วกว่า)
end

def divide_by_2(n : Int32) : Int32
  n / 2  # LLVM แปลงเป็น n >> 1
end
```

## Loop Optimizations

```crystal
# Loop Unrolling: LLVM จะ unroll loops อัตโนมัติ
def sum_array(arr : Array(Int32)) : Int32
  total = 0
  arr.each { |x| total += x }
  total
end
# LLVM อาจ unroll เป็น:
# total += arr[0]; total += arr[1]; total += arr[2]; total += arr[3]; ...

# Loop Vectorization: ใช้ SIMD instructions อัตโนมัติ
def add_arrays(a : Array(Float64), b : Array(Float64)) : Array(Float64)
  Array.new(a.size) { |i| a[i] + b[i] }
end
# LLVM อาจใช้ AVX instructions เพื่อ add 4 doubles พร้อมกัน

# Auto-vectorization hint: ใช้ StaticArray หรือ Slice สำหรับ fixed-size
def dot_product_static(a : StaticArray(Float64, 4), b : StaticArray(Float64, 4)) : Float64
  result = 0.0
  4.times { |i| result += a[i] * b[i] }
  result
end

# ช่วย LLVM vectorize: ใช้ Slice แทน Array สำหรับ known-size data
def sum_slice(data : Slice(Float64)) : Float64
  total = 0.0
  data.each { |x| total += x }
  total
end
```

## Inlining ใน Crystal

```crystal
# Crystal มี annotations สำหรับ control inlining
# @[AlwaysInline]: บังคับ inline (ใช้เมื่อ function เล็กมาก หรือ hot path)
# @[NoInline]: บังคับไม่ inline (ใช้เมื่อ function ใหญ่ หรือ cold path)

# Pattern: inline hot path, don't inline cold path
class Router
  @routes : Hash(String, Proc(String)) = {} of String => Proc(String)

  @[AlwaysInline]
  def lookup(path : String) : Proc(String)?
    @routes[path]?  # Hot path: ถูกเรียกทุก request
  end

  @[NoInline]
  def register_all_routes
    # Cold path: ถูกเรียกครั้งเดียวตอน startup
    @routes["/"] = -> { "home" }
    @routes["/about"] = -> { "about" }
    # ... routes อื่นๆ
  end
end

# Generic functions: LLVM specialized version สำหรับแต่ละ type
def process(x : T) : T forall T
  x * 2  # LLVM สร้าง specialized version สำหรับ Int32, Float64, etc.
end

# ทุก call ด้วย concrete type ได้ specialized code
puts process(5_i32)    # Int32 version
puts process(3.14_f64) # Float64 version
```

## Link Time Optimization (LTO)

```bash
# LTO ช่วย optimize ข้าม compilation units
# Crystal รวม LTO กับ --release อัตโนมัติ

# ดู LLVM optimization passes
crystal build --release --emit llvm-bc -o output.bc src/app.cr
llvm-opt -O3 -print-passes output.bc -o /dev/null 2>&1 | head -20
```

## Profile-Guided Optimization

```crystal
# Crystal ยังไม่ support PGO โดยตรง
# แต่เราสามารถใช้ manual hints เพื่อ guide LLVM

# Hint: branch ไหน likely/unlikely
# ใช้ crystal macro สำหรับ likely/unlikely hints (conceptual)
macro likely(expr)
  {{ expr }}
end

macro unlikely(expr)
  {{ expr }}
end

def process_request(request : String) : String
  if unlikely(request.empty?)
    # Error case (rare)
    return "empty request"
  end

  if likely(request.starts_with?("/api/"))
    # Common case
    handle_api(request)
  else
    handle_static(request)
  end
end

def handle_api(path : String) : String
  "API: #{path}"
end

def handle_static(path : String) : String
  "Static: #{path}"
end
```

## เปรียบเทียบ Debug vs Release

```crystal
# test_perf.cr
require "benchmark"

def fibonacci(n : Int32) : Int64
  return n.to_i64 if n <= 1
  fibonacci(n - 1) + fibonacci(n - 2)
end

Benchmark.ips do |x|
  x.report("fibonacci(30)") { fibonacci(30) }
end
```

```bash
# Debug build
crystal build test_perf.cr
time ./test_perf

# Release build
crystal build --release test_perf.cr
time ./test_perf

# ความแตกต่างมักจะ 2-10x
# สำหรับ numeric code อาจสูงถึง 50x
```

## Optimization Flags

```bash
# ดู available CPU features
crystal build --release --mcpu=native src/app.cr  # ใช้ CPU ปัจจุบัน
crystal build --release --mcpu=x86-64-v3 src/app.cr  # AVX2 support

# ดู LLVM version
crystal --version

# ดู compiler flags ที่ใช้
crystal build --release --verbose src/app.cr 2>&1 | grep "cc "

# Environment variable สำหรับ GC tuning ใน release
CRYSTAL_BOEHM_GC_INITIAL_HEAP_SIZE=256M ./app
```

## แบบฝึกหัด

1. เขียน Fibonacci function แล้วเปรียบเทียบ debug vs release performance
2. สร้าง matrix multiply function แล้วดู LLVM IR ด้วย `--emit llvm-ir`
3. เปรียบเทียบ `Array(Float64)` vs `StaticArray(Float64, N)` ใน tight loop
4. วัด speedup ของ `--mcpu=native` vs default สำหรับ numeric workload

## สรุป

LLVM Optimizations ใน Crystal:
- **--release**: เปิด full LLVM optimization passes (-O3)
- **Constant folding**: คำนวณ constants ตอน compile time
- **Dead code elimination**: ลบ code ที่ไม่ถูกใช้
- **Inlining**: รวม function bodies ลดค่าใช้จ่าย function call
- **Loop unrolling**: ขยาย loops ลด branch overhead
- **Auto-vectorization**: ใช้ SIMD instructions อัตโนมัติ
- **Strength reduction**: แปลง expensive ops เป็น cheap ones (mul->shift)
- **@[AlwaysInline]/@[NoInline]**: control manual inlining hints
- **--mcpu=native**: optimize สำหรับ CPU ปัจจุบัน
