# Part 182: Profiling ใน Crystal

## บทนำ

Profiling คือกระบวนการวัดว่า program ใช้เวลาส่วนใหญ่ทำอะไร ช่วยหา bottlenecks และ optimize ได้อย่างมีหลักฐาน ไม่ใช่ "guess and optimize"

## Crystal Built-in Profiling

```crystal
# Basic manual profiling ด้วย Time.monotonic
class Profiler
  alias Section = NamedTuple(elapsed: Time::Span, calls: Int32)

  @sections : Hash(String, {elapsed: Time::Span, calls: Int32}) = {} of String => {elapsed: Time::Span, calls: Int32}
  @current_section : String? = nil
  @section_start : Time::Span? = nil

  def start(name : String)
    @current_section = name
    @section_start = Time.monotonic
  end

  def stop
    name = @current_section
    started = @section_start
    return unless name && started

    elapsed = Time.monotonic - started
    if existing = @sections[name]?
      @sections[name] = {elapsed: existing[:elapsed] + elapsed, calls: existing[:calls] + 1}
    else
      @sections[name] = {elapsed: elapsed, calls: 1}
    end

    @current_section = nil
    @section_start = nil
  end

  def section(name : String, &block)
    start(name)
    block.call
  ensure
    stop
  end

  def report
    puts "=== Profile Report ==="
    total = @sections.values.sum { |s| s[:elapsed].total_milliseconds }

    sorted = @sections.to_a.sort_by { |_, v| -v[:elapsed].total_milliseconds }

    sorted.each do |name, data|
      elapsed_ms = data[:elapsed].total_milliseconds
      pct = total > 0 ? (elapsed_ms / total * 100).round(1) : 0
      avg_us = elapsed_ms * 1000 / data[:calls]
      puts "  #{name.ljust(30)} #{elapsed_ms.round(3).to_s.rjust(10)}ms  #{pct.to_s.rjust(5)}%  #{data[:calls]} calls  #{avg_us.round(1)}µs avg"
    end
    puts "  Total: #{total.round(3)}ms"
  end
end

# ใช้งาน
profiler = Profiler.new

1000.times do
  profiler.section("parse") { sleep(0.000001) }
  profiler.section("validate") { sleep(0.000002) }
  profiler.section("transform") { sleep(0.000005) }
  profiler.section("serialize") { sleep(0.000001) }
end

profiler.report
```

## Sampling Profiler

```crystal
require "fiber"

# Simple sampling profiler
class SamplingProfiler
  @samples : Hash(String, Int32) = {} of String => Int32
  @running = false
  @interval_ms : Int32

  def initialize(@interval_ms = 1)
  end

  def start
    @running = true
    spawn do
      while @running
        # Crystal ไม่มี built-in stack inspection
        # ในการใช้งานจริงต้องใช้ perf หรือ valgrind
        sleep(@interval_ms.milliseconds)
      end
    end
  end

  def stop
    @running = false
  end

  def record(location : String)
    @samples[location] = (@samples[location]? || 0) + 1
  end

  def report
    total = @samples.values.sum
    sorted = @samples.to_a.sort_by { |_, v| -v }
    puts "=== Sampling Profile ==="
    sorted.each do |loc, count|
      pct = (count.to_f / total * 100).round(1)
      puts "  #{loc.ljust(40)} #{count.to_s.rjust(6)} samples  #{pct}%"
    end
  end
end
```

## Valgrind Profiling

```bash
# ติดตั้ง valgrind
# Ubuntu/Debian
sudo apt-get install valgrind

# Build สำหรับ profiling (debug info แต่ optimized)
crystal build --release --debug src/app.cr -o app_profile

# Callgrind - CPU profiling
valgrind --tool=callgrind --callgrind-out-file=callgrind.out ./app_profile

# Massif - Memory profiling
valgrind --tool=massif --massif-out-file=massif.out ./app_profile

# ดูผล callgrind
callgrind_annotate callgrind.out | head -100
# หรือ kcachegrind (GUI)
kcachegrind callgrind.out

# ดูผล massif
ms_print massif.out | head -50
```

## Linux perf

```bash
# perf - kernel-level profiling (Linux เท่านั้น)
# ต้องการ kernel symbols

# Build
crystal build --release --debug src/app.cr -o app

# Record performance data
sudo perf record -g --call-graph dwarf ./app

# ดูผล
sudo perf report --stdio | head -60

# Flame Graph ด้วย perf
sudo perf record -F 99 -g ./app -- sleep 10
sudo perf script > out.perf
./FlameGraph/stackcollapse-perf.pl out.perf > out.folded
./FlameGraph/flamegraph.pl out.folded > flamegraph.svg

# perf stat: overview stats
sudo perf stat ./app
# Output:
#  Performance counter stats for './app':
#    123.456 ms      task-clock
#          0      context-switches
#          0      cpu-migrations
#      1,234      page-faults
#    456,789,012   cycles
#    234,567,890   instructions
#          1.94   insn per cycle
```

## Flame Graphs

```bash
# สร้าง Flame Graph ด้วย Brendan Gregg's tools
git clone https://github.com/brendangregg/FlameGraph.git
cd FlameGraph

# Method 1: ด้วย perf (Linux)
sudo perf record -F 99 -g ./app
sudo perf script | ./stackcollapse-perf.pl > out.folded
./flamegraph.pl out.folded > flamegraph.svg

# Method 2: ด้วย dtrace (macOS)
sudo dtrace -x ustackframes=100 -n \
  'profile-99 /execname == "app"/ { @[ustack(100)] = count(); }' \
  -c ./app

# ดู SVG ใน browser
open flamegraph.svg
```

## Application-Level Profiling

```crystal
# Middleware-based profiling สำหรับ web apps
module ProfilingMiddleware
  class Timing
    include HTTP::Handler

    def call(context : HTTP::Server::Context)
      start = Time.monotonic
      call_next(context)
      elapsed = Time.monotonic - start

      path = context.request.path
      method = context.request.method
      status = context.response.status_code

      puts "[PROFILE] #{method} #{path} -> #{status} (#{(elapsed.total_milliseconds).round(2)}ms)"
    end
  end
end

# Query profiling
class ProfilingDB
  @queries : Array({sql: String, elapsed: Time::Span}) = [] of {sql: String, elapsed: Time::Span}

  def query(sql : String, &block)
    start = Time.monotonic
    result = block.call
    elapsed = Time.monotonic - start
    @queries << {sql: sql, elapsed: elapsed}
    result
  end

  def slow_queries(threshold : Time::Span = 100.milliseconds)
    @queries.select { |q| q[:elapsed] >= threshold }
  end

  def report
    puts "\n=== Query Profile ==="
    sorted = @queries.sort_by { |q| -q[:elapsed].total_milliseconds }
    sorted.first(10).each do |q|
      ms = q[:elapsed].total_milliseconds.round(2)
      puts "  #{ms.to_s.rjust(8)}ms: #{q[:sql][0..60]}"
    end
  end
end
```

## Memory Profiling

```crystal
require "gc"

# Track memory over time
class MemoryProfiler
  record Snapshot, label : String, heap : UInt64, total : UInt64, at : Time::Span

  @snapshots : Array(Snapshot) = [] of Snapshot
  @start : Time::Span = Time.monotonic

  def snapshot(label : String)
    GC.collect
    stats = GC.stats
    @snapshots << Snapshot.new(
      label: label,
      heap: stats.heap_size,
      total: stats.total_bytes,
      at: Time.monotonic - @start
    )
  end

  def report
    puts "\n=== Memory Profile ==="
    @snapshots.each_with_index do |snap, i|
      heap_kb = snap.heap / 1024
      total_kb = snap.total / 1024

      delta_kb = if i > 0
        prev = @snapshots[i - 1]
        (snap.total.to_i64 - prev.total.to_i64) / 1024
      else
        0_i64
      end

      delta_str = delta_kb >= 0 ? "+#{delta_kb}KB" : "#{delta_kb}KB"
      puts "  #{snap.at.total_milliseconds.round(0).to_i.to_s.rjust(6)}ms  #{snap.label.ljust(30)}  heap: #{heap_kb}KB  total: #{total_kb}KB  delta: #{delta_str}"
    end
  end
end

mem = MemoryProfiler.new
mem.snapshot("start")

data = Array.new(100_000) { rand(1000) }
mem.snapshot("after create array")

sorted = data.sort
mem.snapshot("after sort")

result = sorted.select { |x| x > 500 }.map { |x| x * 2 }
mem.snapshot("after filter+map")

result = nil
GC.collect
mem.snapshot("after GC")

mem.report
```

## แบบฝึกหัด

1. สร้าง profiler middleware สำหรับ HTTP server ที่ track slow requests (> 100ms)
2. ใช้ Valgrind Massif วัด memory usage ของ program ที่สร้าง lots of strings
3. สร้าง query profiler ที่ log และ aggregate database query times
4. วิเคราะห์ flame graph และระบุ hotspot ใน application

## สรุป

Profiling ใน Crystal:
- **Manual timing**: `Time.monotonic` สำหรับ section timing
- **Valgrind callgrind**: CPU profiling พร้อม call graph
- **Valgrind massif**: memory profiling และ allocation tracking
- **Linux perf**: kernel-level CPU profiling
- **Flame Graphs**: visualize CPU time distribution
- **GC.stats**: monitor memory usage
- **Application profiling**: middleware สำหรับ request timing
- **Query profiling**: track slow database queries

"Measure, don't guess" - ทำ profile ก่อน optimize เสมอ
