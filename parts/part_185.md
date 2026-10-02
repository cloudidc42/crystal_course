# Part 185: SIMD and Vectorization ใน Crystal

## บทนำ

SIMD (Single Instruction, Multiple Data) ช่วยให้ CPU ทำงานกับ multiple data elements พร้อมกันในคำสั่งเดียว ทำให้ numeric/media processing เร็วขึ้นมาก (2-16x ขึ้นอยู่กับ operation)

## LLVM Auto-vectorization

```crystal
# LLVM สามารถ auto-vectorize loops อัตโนมัติด้วย --release
# เพียงแค่เขียน code ที่ vectorizable

# Vectorizable loop: no dependencies between iterations
def add_arrays(a : Array(Float64), b : Array(Float64)) : Array(Float64)
  result = Array(Float64).new(a.size)
  a.size.times { |i| result[i] = a[i] + b[i] }
  result
end

# หรือใช้ Slice สำหรับ better vectorization hints
def add_slices(a : Slice(Float64), b : Slice(Float64), result : Slice(Float64))
  a.size.times { |i| result[i] = a[i] + b[i] }
end

# ดู assembly ว่า vectorized ไหม
# crystal build --release --emit asm src/app.cr
# grep "ymm\|xmm" app.s  # AVX/SSE registers

# ตัวอย่าง operations ที่ LLVM vectorize ได้ดี
def dot_product(a : Array(Float64), b : Array(Float64)) : Float64
  sum = 0.0
  a.size.times { |i| sum += a[i] * b[i] }
  sum
end

def scale_array(arr : Array(Float64), factor : Float64) : Array(Float64)
  arr.map { |x| x * factor }
end

def clamp_array(arr : Array(Float64), min : Float64, max : Float64) : Array(Float64)
  arr.map { |x| x.clamp(min, max) }
end
```

## Manual SIMD ผ่าน C Bindings

```crystal
# Intel SSE/AVX ผ่าน C bindings
# สำหรับ x86-64 เท่านั้น

lib LibSIMD
  # SSE2: 128-bit registers, 2x Float64 หรือ 4x Float32
  # ใช้ typedef สำหรับ SIMD types
  type M128D = StaticArray(Float64, 2)   # __m128d
  type M256D = StaticArray(Float64, 4)   # __m256d (AVX)

  # Load/Store
  fun mm_loadu_pd(mem_addr : Float64*) : M128D           # _mm_loadu_pd
  fun mm_storeu_pd(mem_addr : Float64*, a : M128D) : Void # _mm_storeu_pd

  # Arithmetic
  fun mm_add_pd(a : M128D, b : M128D) : M128D   # _mm_add_pd
  fun mm_mul_pd(a : M128D, b : M128D) : M128D   # _mm_mul_pd
  fun mm_sub_pd(a : M128D, b : M128D) : M128D   # _mm_sub_pd
  fun mm_div_pd(a : M128D, b : M128D) : M128D   # _mm_div_pd

  # Horizontal sum
  fun mm_hadd_pd(a : M128D, b : M128D) : M128D  # _mm_hadd_pd (SSE3)
end

# ใช้ SIMD helper C file สำหรับ intrinsics
# (C intrinsics ทำงานได้ดีกว่าผ่าน raw bindings)
```

```c
/* simd_helpers.c - compile กับ Crystal */
#include <immintrin.h>
#include <stdint.h>
#include <string.h>

/* Add two float64 arrays using SIMD */
void simd_add_f64(const double* a, const double* b, double* result, int n) {
    int i;
    /* Process 4 doubles at a time with AVX */
    for (i = 0; i <= n - 4; i += 4) {
        __m256d va = _mm256_loadu_pd(a + i);
        __m256d vb = _mm256_loadu_pd(b + i);
        __m256d vr = _mm256_add_pd(va, vb);
        _mm256_storeu_pd(result + i, vr);
    }
    /* Handle remainder */
    for (; i < n; i++) {
        result[i] = a[i] + b[i];
    }
}

/* Dot product using SIMD */
double simd_dot_product(const double* a, const double* b, int n) {
    __m256d sum = _mm256_setzero_pd();
    int i;
    for (i = 0; i <= n - 4; i += 4) {
        __m256d va = _mm256_loadu_pd(a + i);
        __m256d vb = _mm256_loadu_pd(b + i);
        sum = _mm256_fmadd_pd(va, vb, sum);  /* FMA: a*b + c */
    }

    /* Horizontal sum */
    __m128d low  = _mm256_castpd256_pd128(sum);
    __m128d high = _mm256_extractf128_pd(sum, 1);
    low = _mm_add_pd(low, high);
    low = _mm_hadd_pd(low, low);

    double result;
    _mm_store_sd(&result, low);

    /* Handle remainder */
    for (; i < n; i++) {
        result += a[i] * b[i];
    }
    return result;
}

/* Max element using SIMD */
double simd_max(const double* arr, int n) {
    if (n == 0) return 0.0;
    __m256d vmax = _mm256_set1_pd(arr[0]);
    int i;
    for (i = 0; i <= n - 4; i += 4) {
        __m256d v = _mm256_loadu_pd(arr + i);
        vmax = _mm256_max_pd(vmax, v);
    }

    double tmp[4];
    _mm256_storeu_pd(tmp, vmax);
    double result = tmp[0];
    for (int j = 1; j < 4; j++) {
        if (tmp[j] > result) result = tmp[j];
    }
    for (; i < n; i++) {
        if (arr[i] > result) result = arr[i];
    }
    return result;
}
```

```crystal
# Crystal bindings สำหรับ SIMD helpers
@[Link(ldflags: "-L. -lsimd_helpers -mavx2 -mfma")]
lib LibSIMDHelpers
  fun simd_add_f64(a : Float64*, b : Float64*, result : Float64*, n : Int32) : Void
  fun simd_dot_product(a : Float64*, b : Float64*, n : Int32) : Float64
  fun simd_max(arr : Float64*, n : Int32) : Float64
end

# Crystal wrapper
module SIMD
  def self.add(a : Array(Float64), b : Array(Float64)) : Array(Float64)
    raise ArgumentError.new("Arrays must have same size") unless a.size == b.size
    result = Array(Float64).new(a.size, 0.0)
    LibSIMDHelpers.simd_add_f64(
      a.to_unsafe,
      b.to_unsafe,
      result.to_unsafe,
      a.size
    )
    result
  end

  def self.dot(a : Array(Float64), b : Array(Float64)) : Float64
    raise ArgumentError.new("Arrays must have same size") unless a.size == b.size
    LibSIMDHelpers.simd_dot_product(a.to_unsafe, b.to_unsafe, a.size)
  end

  def self.max(arr : Array(Float64)) : Float64
    return 0.0 if arr.empty?
    LibSIMDHelpers.simd_max(arr.to_unsafe, arr.size)
  end
end

# ใช้งาน
n = 1_000_000
a = Array.new(n) { rand }
b = Array.new(n) { rand }

# SIMD operations
result = SIMD.add(a, b)
dot = SIMD.dot(a, b)
maximum = SIMD.max(a)

puts "First 3: #{result[0..2].map { |x| x.round(4) }}"
puts "Dot product: #{dot.round(4)}"
puts "Maximum: #{maximum.round(4)}"
```

## เมื่อไหร่ใช้ SIMD

```crystal
# Use SIMD เมื่อ:
# 1. Large array operations (>1000 elements)
# 2. Same operation บน many values
# 3. Hot path ที่ benchmark แล้วว่า slow

# ตัวอย่าง use cases ที่เหมาะ:
# - Image processing (pixel manipulation)
# - Signal processing (audio/video)
# - Machine learning (matrix multiply, activation functions)
# - Physics simulation (particle systems)
# - Cryptography (AES, hashing)
# - Financial calculations (bulk price updates)

# อย่าใช้ SIMD เมื่อ:
# 1. Array เล็ก (overhead ของ setup > benefit)
# 2. Complex branching
# 3. Non-sequential memory access
# 4. Code ที่ต้องการ portable ข้าม architectures

# Test: SIMD vs scalar benchmark
require "benchmark"

n = 100_000
a_arr = Array.new(n) { rand }
b_arr = Array.new(n) { rand }

Benchmark.ips do |x|
  x.report("scalar dot product") do
    a_arr.zip(b_arr).sum { |a, b| a * b }
  end

  x.report("Crystal reduce") do
    sum = 0.0
    n.times { |i| sum += a_arr[i] * b_arr[i] }
    sum
  end

  # SIMD version (requires simd_helpers.so)
  # x.report("SIMD dot product") do
  #   SIMD.dot(a_arr, b_arr)
  # end

  x.compare!
end
```

## แบบฝึกหัด

1. เขียน C SIMD function สำหรับ RGB to grayscale conversion ด้วย SSE2/AVX
2. Benchmark LLVM auto-vectorized loop vs manual SIMD implementation
3. สร้าง Crystal wrapper สำหรับ SIMD string operations (strcmp, strlen)
4. วัดผลต่างของ `--mcpu=native` ที่ enable AVX2/FMA vs default

## สรุป

SIMD และ Vectorization ใน Crystal:
- **LLVM auto-vectorization**: เกิดอัตโนมัติเมื่อ `--release` สำหรับ simple loops
- **--mcpu=native**: เปิดใช้ CPU instructions ที่ดีที่สุดสำหรับ machine ปัจจุบัน
- **C bindings**: ใช้ Intel intrinsics ผ่าน C code สำหรับ manual SIMD
- **SSE2/AVX**: 128/256-bit registers สำหรับ packed Float64 operations
- **FMA**: Fused Multiply-Add ลด latency สำหรับ dot products
- **Use cases**: image processing, signal processing, ML, physics simulation
- **Trade-offs**: SIMD มี overhead setup, ดีสำหรับ n > 1000 elements เท่านั้น
