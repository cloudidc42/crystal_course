# Part 175: Pointers ใน Crystal

## บทนำ

Crystal มี Pointer(T) ซึ่งเป็น raw memory pointer คล้ายกับ C pointers ใช้สำหรับ low-level programming, C FFI, และการ optimize performance-critical code

## Pointer(T) พื้นฐาน

```crystal
# สร้าง pointer ด้วย pointerof()
value = 42
ptr = pointerof(value)

puts ptr.class        # Pointer(Int32)
puts ptr.value        # 42
puts ptr.address      # memory address (เช่น 140234567890)

# เปลี่ยนค่าผ่าน pointer
ptr.value = 100
puts value            # 100 - ค่าเปลี่ยนเพราะ point ไปที่ memory เดียวกัน

# Pointer to struct
struct Point
  property x : Float64
  property y : Float64

  def initialize(@x, @y)
  end
end

pt = Point.new(1.0, 2.0)
pt_ptr = pointerof(pt)
puts pt_ptr.value.x   # 1.0
pt_ptr.value.x = 3.0
puts pt.x             # 3.0 - เปลี่ยนผ่าน pointer
```

## malloc, realloc, free

```crystal
# malloc - allocate memory ใหม่
ptr = Pointer(Int32).malloc(5)
puts "Allocated: #{ptr}"

begin
  # เขียนค่าลง allocated memory
  5.times { |i| (ptr + i).value = i * 10 }

  # อ่านค่ากลับมา
  5.times { |i| puts "ptr[#{i}] = #{(ptr + i).value}" }
ensure
  # ต้อง free เสมอ!
  ptr.free
  puts "Freed"
end

# malloc array แล้ว zero-initialize
ptr = Pointer(Int32).malloc(10, 0)  # ให้ค่า initial = 0
10.times { |i| puts (ptr + i).value }  # all 0
ptr.free

# realloc - ขยาย/หด buffer
ptr = Pointer(UInt8).malloc(10)
10.times { |i| (ptr + i).value = i.to_u8 }

# ขยายเป็น 20 bytes
ptr = ptr.realloc(20)
# bytes แรก 10 ยังคงค่าเดิม, bytes 10-19 undefined
puts (ptr + 0).value  # 0
puts (ptr + 9).value  # 9

ptr.free
```

## Pointer Arithmetic

```crystal
# Pointer arithmetic ทำงานเป็น element-based (ไม่ใช่ byte-based)
arr = [10, 20, 30, 40, 50]
ptr = arr.to_unsafe

puts "ptr points to: #{ptr.value}"      # 10
puts "ptr+1 points to: #{(ptr+1).value}" # 20
puts "ptr+4 points to: #{(ptr+4).value}" # 50

# Walk array ด้วย pointer
current = ptr
end_ptr = ptr + arr.size

while current < end_ptr
  print "#{current.value} "
  current += 1
end
puts

# ค่า byte address ของแต่ละ element
base = ptr.address
puts "sizeof(Int32) = #{sizeof(Int32)}"  # 4
puts "Address of [0] = #{ptr.address}"
puts "Address of [1] = #{(ptr+1).address}"
puts "Difference = #{(ptr+1).address - ptr.address}"  # 4 (sizeof Int32)

# Pointer comparison
p1 = ptr
p2 = ptr + 3

puts p1 < p2    # true
puts p1 > p2    # false
puts p1 == p2   # false
puts (p2 - p1)  # 3 (distance in elements)
```

## Unsafe Operations

```crystal
# unsafe block ป้องกัน accidental use
# Crystal บางครั้งต้องการ explicit casting

# แปลง type ของ pointer (unsafe!)
float_val = 3.14_f32
float_ptr = pointerof(float_val)

# Reinterpret bytes เป็น Int32
int_ptr = float_ptr.as(Int32*)
int_bits = int_ptr.value

puts "Float bytes as int: #{int_bits}"
puts "In hex: #{int_bits.to_s(16)}"
# Float 3.14 ใน IEEE 754 = 0x4048F5C3

# Void pointer
buffer = Bytes.new(16, 0_u8)
void_ptr = buffer.to_unsafe.as(Void*)

# ส่ง void pointer ไปยัง C function
lib LibMemset
  fun memset(s : Void*, c : Int32, n : LibC::SizeT) : Void*
end

LibMemset.memset(void_ptr, 0xFF, 16)
puts buffer.hexstring  # ffffffffffffffffffffffffffffffff

# Struct layout
struct Header
  property magic : UInt32
  property version : UInt16
  property flags : UInt16
  property size : UInt64

  def initialize(@magic, @version, @flags, @size)
  end
end

header = Header.new(0xDEADBEEF_u32, 1_u16, 0x0001_u16, 1024_u64)
ptr = pointerof(header).as(UInt8*)

puts "Header bytes:"
sizeof(Header).times { |i| print "#{(ptr + i).value.to_s(16).rjust(2, '0')} " }
puts
```

## Passing Pointers to C

```crystal
@[Link("c")]
lib LibC
  fun scanf(fmt : UInt8*, ...) : Int32
  fun memcpy(dst : Void*, src : Void*, n : LibC::SizeT) : Void*
  fun memmove(dst : Void*, src : Void*, n : LibC::SizeT) : Void*
  fun qsort(base : Void*, nmemb : LibC::SizeT, size : LibC::SizeT, compar : (Void*, Void*) -> Int32) : Void
  fun bsearch(key : Void*, base : Void*, nmemb : LibC::SizeT, size : LibC::SizeT, compar : (Void*, Void*) -> Int32) : Void*
end

# Pass pointer เข้า C function
src = [1, 2, 3, 4, 5]
dst = Array.new(5, 0)

LibC.memcpy(
  dst.to_unsafe.as(Void*),
  src.to_unsafe.as(Void*),
  (5 * sizeof(Int32)).to_u64
)

puts "Copied: #{dst.inspect}"

# qsort ด้วย callback
arr = [5, 2, 8, 1, 9, 3, 7, 4, 6].map(&.to_i32)

compare = ->(a : Void*, b : Void*) {
  a.as(Int32*).value <=> b.as(Int32*).value
}

LibC.qsort(
  arr.to_unsafe.as(Void*),
  arr.size.to_u64,
  sizeof(Int32).to_u64,
  compare
)

puts "Sorted: #{arr.inspect}"

# bsearch
target = 5_i32
result = LibC.bsearch(
  pointerof(target).as(Void*),
  arr.to_unsafe.as(Void*),
  arr.size.to_u64,
  sizeof(Int32).to_u64,
  compare
)

if result
  found = result.as(Int32*).value
  puts "Found: #{found}"
end
```

## Slice และ Pointer

```crystal
# Slice เป็น wrapper ที่ปลอดภัยกว่า Pointer
arr = [1, 2, 3, 4, 5]

# Slice จาก Array
slice = arr.to_slice
puts slice[0]   # 1
puts slice[-1]  # 5

# Slice จาก Pointer
ptr = arr.to_unsafe
slice2 = Slice.new(ptr, 5)
puts slice2[2]  # 3

# Slice จาก allocated memory
ptr3 = Pointer(UInt8).malloc(256)
slice3 = Slice.new(ptr3, 256)
slice3.fill(0_u8)
puts slice3.all? { |b| b == 0 }  # true
ptr3.free

# Sub-slice
full_slice = Slice.new(10) { |i| i.to_u8 }
sub = full_slice[2..5]  # [2, 3, 4, 5]
puts sub.inspect

# Bytes (alias สำหรับ Slice(UInt8))
bytes = Bytes.new(8)
bytes[0] = 0xDE_u8
bytes[1] = 0xAD_u8
puts bytes.hexstring  # dead000000000000
```

## Null Pointer

```crystal
# Null pointer ใน Crystal
null_ptr = Pointer(Int32).null
puts null_ptr.null?   # true
puts null_ptr.address # 0

# ตรวจสอบก่อนใช้
if !null_ptr.null?
  puts null_ptr.value  # Unsafe ถ้าไม่ตรวจสอบก่อน
end

# Pointer? type ไม่มีใน Crystal แต่ทำได้ด้วย null check
def safe_dereference(ptr : Pointer(T)) : T? forall T
  ptr.null? ? nil : ptr.value
end

ptr = Pointer(Int32).malloc(1)
ptr.value = 42

puts safe_dereference(ptr)  # 42
puts safe_dereference(Pointer(Int32).null)  # nil

ptr.free
```

## Memory Safety Patterns

```crystal
# RAII pattern สำหรับ manual memory
class OwnedPointer(T)
  getter ptr : Pointer(T)

  def initialize(size : Int32 = 1)
    @ptr = Pointer(T).malloc(size)
    @size = size
    @freed = false
  end

  def initialize(size : Int32, initial_value : T)
    @ptr = Pointer(T).malloc(size, initial_value)
    @size = size
    @freed = false
  end

  def [](index : Int32) : T
    bounds_check(index)
    (@ptr + index).value
  end

  def []=(index : Int32, value : T)
    bounds_check(index)
    (@ptr + index).value = value
  end

  def size : Int32
    @size
  end

  def free
    unless @freed
      @ptr.free
      @freed = true
    end
  end

  def finalize
    free  # Safety net
  end

  private def bounds_check(index : Int32)
    raise IndexError.new if index < 0 || index >= @size
  end
end

# ใช้งาน
owned = OwnedPointer(Float64).new(5)

5.times { |i| owned[i] = i.to_f64 * 1.1 }
5.times { |i| puts "owned[#{i}] = #{owned[i]}" }

owned.free  # explicit free

# หรือปล่อยให้ GC + finalizer จัดการ
owned2 = OwnedPointer(Int32).new(10, 0)
# owned2.finalize จะเรียก free อัตโนมัติเมื่อ GC collect
```

## แบบฝึกหัด

1. สร้าง `DynamicArray(T)` ที่ใช้ `Pointer(T)` แทน Crystal's built-in Array เพื่อ control memory manually
2. เขียน `memcpy` และ `memset` ใน Crystal ด้วย raw pointers แล้วเปรียบเทียบ performance
3. สร้าง circular buffer ด้วย `Pointer(T)` สำหรับ lock-free producer-consumer
4. Implement `String` concatenation ที่ allocate memory ครั้งเดียวแทนที่ทุก concat

## สรุป

Pointers ใน Crystal:
- **Pointer(T)**: raw memory pointer คล้าย C
- **malloc/realloc/free**: manual memory management
- **Pointer arithmetic**: ขยับ pointer ด้วย + และ - (element-based)
- **as()**: cast pointer type (unsafe)
- **pointerof()**: get address ของ variable/field
- **to_unsafe**: แปลง Crystal types เป็น pointer
- **Slice(T)**: safe wrapper รอบ pointer พร้อม bounds checking
- **null?**: ตรวจสอบ null pointer

Pointers ใน Crystal ให้ power เท่ากับ C แต่ต้องระมัดระวังมากเพราะไม่มี safety checks
