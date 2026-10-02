# Part 117: Worker Pools - การสร้าง Worker Pool ด้วย Fiber และ Channel

## บทนำ

Worker Pool เป็น pattern ที่ใช้บ่อยมากในการเขียน concurrent programs โดยแนวคิดหลักคือการสร้าง "pool" ของ worker fiber จำนวนหนึ่ง แล้วกระจายงาน (jobs) ให้กับ worker เหล่านั้นผ่าน channel ทำให้สามารถควบคุมจำนวน concurrent operations ได้

## Worker Pool พื้นฐาน

```crystal
# Worker Pool แบบง่าย
def create_worker_pool(num_workers : Int32, jobs : Channel(Int32), results : Channel(Int32))
  num_workers.times do |worker_id|
    spawn do
      while job = jobs.receive?
        # ทำงาน
        result = job * job # สมมติว่า compute job^2
        puts "Worker #{worker_id} ประมวลผล #{job} => #{result}"
        results.send(result)
      end
      puts "Worker #{worker_id} หยุดทำงาน"
    end
  end
end

jobs = Channel(Int32).new(20)
results = Channel(Int32).new(20)

# สร้าง 3 workers
create_worker_pool(3, jobs, results)

# ส่งงาน 10 ชิ้น
10.times do |i|
  jobs.send(i + 1)
end
jobs.close

# รับผลลัพธ์
total = 0
10.times do
  total += results.receive
end

puts "ผลรวม: #{total}"
```

## Worker Pool แบบ Generic

```crystal
class WorkerPool(T, R)
  getter completed : Int32 = 0
  
  def initialize(@num_workers : Int32, &@processor : T -> R)
    @jobs = Channel(T).new(100)
    @results = Channel(R).new(100)
    @active_workers = Atomic(Int32).new(@num_workers)
    start_workers
  end
  
  def submit(job : T)
    @jobs.send(job)
  end
  
  def submit_all(jobs : Enumerable(T))
    jobs.each { |job| submit(job) }
  end
  
  def close
    @jobs.close
  end
  
  def results_channel
    @results
  end
  
  def collect_results(count : Int32) : Array(R)
    Array(R).new(count) { @results.receive }
  end
  
  private def start_workers
    @num_workers.times do |id|
      spawn do
        while job = @jobs.receive?
          begin
            result = @processor.call(job)
            @results.send(result)
          rescue ex
            puts "Worker #{id} error: #{ex.message}"
          end
        end
        
        # ตรวจสอบว่าทุก worker หยุดแล้วหรือยัง
        remaining = @active_workers.sub(1)
        @results.close if remaining == 1
      end
    end
  end
end

# การใช้งาน: ประมวลผล URL แบบ parallel
pool = WorkerPool(String, String).new(5) do |url|
  # จำลองการ fetch URL
  sleep rand(0.1..0.5).seconds
  "ผลลัพธ์จาก #{url}"
end

urls = (1..20).map { |i| "https://example.com/page/#{i}" }
pool.submit_all(urls)
pool.close

20.times do
  if result = pool.results_channel.receive?
    puts result
  end
end
```

## Job Queue

```crystal
# Job Queue พร้อม priority และ retry
struct Job
  property id : String
  property priority : Int32
  property data : String
  property retry_count : Int32
  property max_retries : Int32
  
  def initialize(@id, @data, @priority = 0, @retry_count = 0, @max_retries = 3)
  end
  
  def can_retry?
    @retry_count < @max_retries
  end
  
  def increment_retry
    Job.new(@id, @data, @priority, @retry_count + 1, @max_retries)
  end
end

class JobQueue
  def initialize(@num_workers : Int32)
    @high_queue = Channel(Job).new(50)
    @normal_queue = Channel(Job).new(200)
    @failed_queue = Channel(Job).new(50)
    @results = Channel(String).new(200)
    @pending = Atomic(Int32).new(0)
    
    start_workers
    start_failure_handler
  end
  
  def enqueue(job : Job)
    @pending.add(1)
    if job.priority > 5
      @high_queue.send(job)
    else
      @normal_queue.send(job)
    end
  end
  
  def pending_count
    @pending.get
  end
  
  def wait_for_results(count : Int32) : Array(String)
    Array(String).new(count) { @results.receive }
  end
  
  private def process_job(job : Job) : String
    # จำลองการทำงาน - อาจ fail บ้างบางครั้ง
    if rand(0.0..1.0) < 0.2 # 20% chance of failure
      raise "งาน #{job.id} ล้มเหลว"
    end
    
    sleep 0.05.seconds
    "งาน #{job.id} เสร็จแล้ว"
  end
  
  private def start_workers
    @num_workers.times do |worker_id|
      spawn do
        loop do
          job = select
            when j = @high_queue.receive?
              j
            when j = @normal_queue.receive?
              j
            else
              nil
          end
          
          break unless job
          
          begin
            result = process_job(job.not_nil!)
            @results.send(result)
            @pending.sub(1)
          rescue ex
            if job.not_nil!.can_retry?
              @failed_queue.send(job.not_nil!.increment_retry)
            else
              @results.send("งาน #{job.not_nil!.id} ล้มเหลวถาวร")
              @pending.sub(1)
            end
          end
        end
      end
    end
  end
  
  private def start_failure_handler
    spawn do
      while failed_job = @failed_queue.receive?
        sleep 0.5.seconds # รอก่อน retry
        puts "Retry งาน #{failed_job.id} (ครั้งที่ #{failed_job.retry_count})"
        
        if failed_job.priority > 5
          @high_queue.send(failed_job)
        else
          @normal_queue.send(failed_job)
        end
      end
    end
  end
end

# ทดสอบ Job Queue
queue = JobQueue.new(4)

20.times do |i|
  priority = i < 5 ? 10 : 2 # 5 งานแรกมี priority สูง
  queue.enqueue(Job.new("job_#{i}", "ข้อมูลงาน #{i}", priority))
end

results = queue.wait_for_results(20)
results.each { |r| puts r }
```

## Results Aggregation

```crystal
# รวบรวมผลลัพธ์จากหลาย worker
class ResultAggregator(T)
  def initialize
    @results = [] of T
    @errors = [] of Exception
    @mutex = Mutex.new
    @done_channel = Channel(Nil).new
  end
  
  def add_result(result : T)
    @mutex.synchronize { @results << result }
  end
  
  def add_error(error : Exception)
    @mutex.synchronize { @errors << error }
  end
  
  def results
    @mutex.synchronize { @results.dup }
  end
  
  def errors
    @mutex.synchronize { @errors.dup }
  end
  
  def success_count
    @mutex.synchronize { @results.size }
  end
  
  def error_count
    @mutex.synchronize { @errors.size }
  end
end

# Parallel map ด้วย Worker Pool
def parallel_map(items : Array(T), num_workers : Int32, &block : T -> R) : Array(R) forall T, R
  results = Array(R?).new(items.size, nil)
  jobs = Channel(Tuple(Int32, T)).new(items.size)
  done = Channel(Nil).new(num_workers)
  mutex = Mutex.new
  
  # ส่งงานพร้อม index
  items.each_with_index { |item, i| jobs.send({i, item}) }
  jobs.close
  
  # Worker
  num_workers.times do
    spawn do
      while tuple = jobs.receive?
        idx, item = tuple
        result = block.call(item)
        mutex.synchronize { results[idx] = result }
      end
      done.send(nil)
    end
  end
  
  # รอทุก worker
  num_workers.times { done.receive }
  
  results.map(&.not_nil!)
end

# ทดสอบ parallel_map
numbers = (1..20).to_a
squares = parallel_map(numbers, 4) { |n| n * n }
puts squares.inspect
```

## Worker Pool พร้อม Metrics

```crystal
class MetricWorkerPool(T, R)
  struct Metrics
    property total_jobs : Int32 = 0
    property completed_jobs : Int32 = 0
    property failed_jobs : Int32 = 0
    property total_time_ms : Float64 = 0.0
    
    def avg_time_ms
      return 0.0 if completed_jobs == 0
      total_time_ms / completed_jobs
    end
    
    def success_rate
      return 0.0 if total_jobs == 0
      (completed_jobs.to_f / total_jobs * 100).round(2)
    end
  end
  
  def initialize(@num_workers : Int32, &@processor : T -> R)
    @jobs = Channel(T).new(200)
    @results = Channel(R).new(200)
    @metrics = Metrics.new
    @metrics_mutex = Mutex.new
    start_workers
  end
  
  def submit(job : T)
    @metrics_mutex.synchronize { @metrics.total_jobs += 1 }
    @jobs.send(job)
  end
  
  def close
    @jobs.close
  end
  
  def receive_result : R?
    @results.receive?
  end
  
  def metrics : Metrics
    @metrics_mutex.synchronize { @metrics }
  end
  
  private def start_workers
    @num_workers.times do |id|
      spawn do
        while job = @jobs.receive?
          start_time = Time.monotonic
          
          begin
            result = @processor.call(job)
            elapsed = (Time.monotonic - start_time).total_milliseconds
            
            @metrics_mutex.synchronize do
              @metrics.completed_jobs += 1
              @metrics.total_time_ms += elapsed
            end
            
            @results.send(result)
          rescue ex
            @metrics_mutex.synchronize { @metrics.failed_jobs += 1 }
            puts "Worker #{id} error: #{ex.message}"
          end
        end
      end
    end
  end
end

# ทดสอบพร้อม metrics
pool = MetricWorkerPool(Int32, Int32).new(5) do |n|
  sleep rand(0.01..0.1).seconds
  n * n
end

100.times { |i| pool.submit(i + 1) }
pool.close

100.times { pool.receive_result }

m = pool.metrics
puts "งานทั้งหมด: #{m.total_jobs}"
puts "สำเร็จ: #{m.completed_jobs}"
puts "ล้มเหลว: #{m.failed_jobs}"
puts "เวลาเฉลี่ย: #{m.avg_time_ms.round(2)} ms"
puts "อัตราความสำเร็จ: #{m.success_rate}%"
```

## Dynamic Worker Pool

```crystal
# Worker Pool ที่ปรับขนาดได้ตามโหลด
class DynamicWorkerPool(T, R)
  MIN_WORKERS = 2
  MAX_WORKERS = 20
  SCALE_UP_THRESHOLD = 80   # queue 80% เต็ม => เพิ่ม worker
  SCALE_DOWN_THRESHOLD = 20 # queue 20% เต็ม => ลด worker
  
  def initialize(&@processor : T -> R)
    @jobs = Channel(T).new(100)
    @results = Channel(R).new(100)
    @worker_count = Atomic(Int32).new(0)
    @mutex = Mutex.new
    
    # เริ่มต้นด้วย minimum workers
    MIN_WORKERS.times { add_worker }
    
    # เริ่ม auto-scaler
    start_auto_scaler
  end
  
  def submit(job : T)
    @jobs.send(job)
  end
  
  def receive : R
    @results.receive
  end
  
  def worker_count
    @worker_count.get
  end
  
  private def add_worker
    @worker_count.add(1)
    spawn do
      while job = @jobs.receive?
        result = @processor.call(job)
        @results.send(result)
      end
      @worker_count.sub(1)
    end
  end
  
  private def start_auto_scaler
    spawn do
      loop do
        sleep 0.5.seconds
        
        queue_usage = @jobs.@capacity > 0 ? (@jobs.@size.to_f / @jobs.@capacity * 100) : 0
        current = @worker_count.get
        
        if queue_usage > SCALE_UP_THRESHOLD && current < MAX_WORKERS
          puts "Scale UP: #{current} => #{current + 1} workers (queue: #{queue_usage.round(1)}%)"
          add_worker
        end
      end
    end
  end
end
```

## Batch Processing Worker Pool

```crystal
class BatchWorker(T, R)
  def initialize(@num_workers : Int32, @batch_size : Int32, &@processor : Array(T) -> Array(R))
    @jobs = Channel(T).new(1000)
    @results = Channel(R).new(1000)
    start_workers
  end
  
  def submit(item : T)
    @jobs.send(item)
  end
  
  def submit_all(items : Array(T))
    items.each { |item| submit(item) }
  end
  
  def close
    @jobs.close
  end
  
  def collect(count : Int32) : Array(R)
    Array(R).new(count) { @results.receive }
  end
  
  private def start_workers
    @num_workers.times do |worker_id|
      spawn do
        loop do
          # รวบรวม batch
          batch = Array(T).new
          
          begin
            # รับ item แรก (blocking)
            first = @jobs.receive?
            break unless first
            batch << first.not_nil!
            
            # รับ items เพิ่มเติม (non-blocking)
            (@batch_size - 1).times do
              select
              when item = @jobs.receive?
                batch << item.not_nil!
              else
                break
              end
            end
          rescue Channel::ClosedError
            break
          end
          
          next if batch.empty?
          
          # ประมวลผล batch
          puts "Worker #{worker_id} ประมวลผล batch ขนาด #{batch.size}"
          results = @processor.call(batch)
          results.each { |r| @results.send(r) }
        end
      end
    end
  end
end

# ทดสอบ Batch Processing
batch_worker = BatchWorker(Int32, String).new(3, 5) do |batch|
  sleep 0.1.seconds # จำลองการประมวลผล
  batch.map { |n| "ผลลัพธ์ #{n}" }
end

50.times { |i| batch_worker.submit(i + 1) }
batch_worker.close

results = batch_worker.collect(50)
puts "ได้รับผลลัพธ์ #{results.size} รายการ"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Image Processing Pool

```crystal
# จำลองการ resize รูปภาพแบบ parallel
struct ImageJob
  property filename : String
  property width : Int32
  property height : Int32
  
  def initialize(@filename, @width, @height)
  end
end

struct ResizeResult
  property original : String
  property new_size : String
  property time_ms : Float64
  
  def initialize(@original, @new_size, @time_ms)
  end
end

class ImageProcessingPool
  def initialize(@num_workers : Int32)
    @jobs = Channel(ImageJob).new(100)
    @results = Channel(ResizeResult).new(100)
    start_workers
  end
  
  def process(job : ImageJob)
    @jobs.send(job)
  end
  
  def close
    @jobs.close
  end
  
  def next_result : ResizeResult?
    @results.receive?
  end
  
  private def resize_image(job : ImageJob) : ResizeResult
    start = Time.monotonic
    # จำลองการ resize
    sleep rand(0.1..0.5).seconds
    new_width = job.width / 2
    new_height = job.height / 2
    elapsed = (Time.monotonic - start).total_milliseconds
    
    ResizeResult.new(job.filename, "#{new_width}x#{new_height}", elapsed)
  end
  
  private def start_workers
    @num_workers.times do |id|
      spawn do
        while job = @jobs.receive?
          result = resize_image(job)
          @results.send(result)
          puts "Worker #{id}: #{result.original} => #{result.new_size} (#{result.time_ms.round(1)}ms)"
        end
      end
    end
  end
end

pool = ImageProcessingPool.new(4)

images = [
  ImageJob.new("photo1.jpg", 4096, 3072),
  ImageJob.new("photo2.jpg", 2048, 1536),
  ImageJob.new("photo3.png", 1920, 1080),
  ImageJob.new("photo4.jpg", 3840, 2160),
  ImageJob.new("photo5.png", 1280, 720),
]

images.each { |img| pool.process(img) }
pool.close

results = [] of ResizeResult
images.size.times do
  if r = pool.next_result
    results << r
  end
end

puts "\nสรุปผล:"
puts "ประมวลผลรูปภาพ #{results.size} รูป"
avg_time = results.sum(&.time_ms) / results.size
puts "เวลาเฉลี่ย: #{avg_time.round(1)} ms"
```

### แบบฝึกหัดที่ 2: เปรียบเทียบ Sequential vs Parallel

```crystal
# เปรียบเทียบความเร็ว
def fibonacci(n : Int32) : Int64
  return n.to_i64 if n <= 1
  a, b = 0_i64, 1_i64
  (n - 1).times { a, b = b, a + b }
  b
end

numbers = (30..40).to_a

# Sequential
start = Time.monotonic
sequential_results = numbers.map { |n| fibonacci(n) }
sequential_time = Time.monotonic - start
puts "Sequential: #{sequential_time.total_milliseconds.round(1)} ms"

# Parallel
jobs = Channel(Int32).new(20)
results = Channel(Tuple(Int32, Int64)).new(20)

4.times do
  spawn do
    while n = jobs.receive?
      results.send({n, fibonacci(n)})
    end
  end
end

start = Time.monotonic
numbers.each { |n| jobs.send(n) }
jobs.close

parallel_results = {} of Int32 => Int64
numbers.size.times do
  n, fib = results.receive
  parallel_results[n] = fib
end

parallel_time = Time.monotonic - start
puts "Parallel: #{parallel_time.total_milliseconds.round(1)} ms"
puts "Speedup: #{(sequential_time / parallel_time).round(2)}x"
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Worker Pool พื้นฐาน**: การสร้าง pool ของ fiber ที่รับงานจาก channel
2. **Generic Worker Pool**: การสร้าง worker pool ที่รองรับ type ทั่วไป
3. **Job Queue**: การจัดการงานพร้อม priority และ retry mechanism
4. **Results Aggregation**: การรวบรวมผลลัพธ์จากหลาย worker
5. **Metrics**: การติดตามประสิทธิภาพของ worker pool
6. **Batch Processing**: การรวมงานเป็น batch เพื่อลด overhead
7. **Dynamic Scaling**: การปรับจำนวน worker ตามโหลด

ข้อควรระวัง:
- ระวัง deadlock เมื่อ channel เต็มและไม่มีคนรับ
- ปิด channel เมื่อไม่มีงานส่งมาอีก
- ใช้ buffered channel เพื่อเพิ่ม throughput
- จำนวน worker ที่เหมาะสมขึ้นอยู่กับ CPU cores และลักษณะงาน
