# Part 155: Background Jobs และ Queue ใน Crystal

## บทนำ

Background Jobs คืองานที่ทำงานในพื้นหลังโดยไม่บล็อก HTTP request หลัก เช่น การส่งอีเมล, การประมวลผลรูปภาพ, หรือการส่ง notification Crystal มี fiber-based concurrency ที่ยอดเยี่ยม และสามารถใช้งานร่วมกับ Redis เพื่อสร้าง job queue ที่มีประสิทธิภาพ

## Job Queue พื้นฐานด้วย Channel

### Simple In-Memory Queue

```crystal
# src/jobs/simple_queue.cr

# Job interface
abstract class Job
  abstract def perform
  
  def job_name : String
    self.class.name
  end
end

# Worker ที่ทำงานใน background fiber
class JobQueue
  MAX_WORKERS = 5
  
  def initialize
    @queue = Channel(Job).new(capacity: 1000)
    @workers_started = false
  end
  
  # เพิ่มงานเข้า queue
  def enqueue(job : Job)
    start_workers unless @workers_started
    @queue.send(job)
  end
  
  # รอให้งานทั้งหมดเสร็จ
  def wait_for_completion
    @queue.close
  end
  
  private def start_workers
    @workers_started = true
    
    MAX_WORKERS.times do |i|
      spawn do
        puts "Worker #{i} เริ่มทำงาน"
        
        loop do
          job = @queue.receive?
          break unless job
          
          begin
            puts "Worker #{i} กำลังทำงาน: #{job.job_name}"
            job.perform
            puts "Worker #{i} เสร็จ: #{job.job_name}"
          rescue ex
            puts "Worker #{i} เจอข้อผิดพลาด: #{ex.message}"
          end
        end
        
        puts "Worker #{i} หยุดทำงาน"
      end
    end
  end
end
```

### ตัวอย่าง Job classes

```crystal
# src/jobs/email_job.cr
class SendEmailJob < Job
  def initialize(
    @to : String,
    @subject : String,
    @body : String
  )
  end
  
  def perform
    # จำลองการส่งอีเมล
    puts "ส่งอีเมลไปยัง #{@to}"
    puts "หัวข้อ: #{@subject}"
    
    # ใน production ใช้ SMTP library จริงๆ
    sleep(0.1.seconds)  # จำลองการส่ง
    puts "ส่งอีเมลสำเร็จ!"
  end
end

# src/jobs/image_process_job.cr
class ProcessImageJob < Job
  def initialize(
    @file_path : String,
    @output_path : String,
    @width : Int32,
    @height : Int32
  )
  end
  
  def perform
    puts "ประมวลผลรูปภาพ: #{@file_path}"
    puts "Resize เป็น #{@width}x#{@height}"
    
    # ใน production ใช้ ImageMagick หรือ similar
    sleep(0.5.seconds)
    puts "ประมวลผลรูปภาพสำเร็จ: #{@output_path}"
  end
end

# src/jobs/notification_job.cr  
class SendNotificationJob < Job
  def initialize(
    @user_id : Int64,
    @message : String,
    @channel : String = "push"
  )
  end
  
  def perform
    puts "ส่ง notification ไปยัง user #{@user_id}"
    puts "ข้อความ: #{@message}"
    puts "ช่องทาง: #{@channel}"
    
    sleep(0.05.seconds)
    puts "ส่ง notification สำเร็จ!"
  end
end

# การใช้งาน
queue = JobQueue.new

# เพิ่มงาน
queue.enqueue(SendEmailJob.new("user@example.com", "ยืนยันการสมัคร", "ยินดีต้อนรับ!"))
queue.enqueue(ProcessImageJob.new("/tmp/photo.jpg", "/tmp/thumb.jpg", 200, 200))
queue.enqueue(SendNotificationJob.new(123_i64, "โพสต์ใหม่จากที่คุณติดตาม"))

Fiber.yield  # ให้ fibers ทำงาน
sleep(2.seconds)
```

## Redis-Backed Queue

### shard.yml สำหรับ Redis Queue

```yaml
# shard.yml
name: background_jobs_app
version: 1.0.0

dependencies:
  redis:
    github: stefanwille/crystal-redis
    version: ~> 2.9.0
```

### Redis Job Queue Implementation

```crystal
# src/queue/redis_queue.cr
require "redis"
require "json"
require "uuid"

module Queue
  class RedisQueue
    QUEUE_PREFIX = "queue:"
    FAILED_PREFIX = "failed:"
    PROCESSING_PREFIX = "processing:"
    DEFAULT_TTL = 86400  # 24 ชั่วโมง
    
    def initialize(
      host : String = "localhost",
      port : Int32 = 6379,
      password : String? = nil
    )
      @redis = Redis::PooledClient.new(
        host: host,
        port: port,
        password: password,
        pool_size: 10
      )
    end
    
    # เพิ่มงานเข้า queue
    def enqueue(
      queue_name : String,
      job_class : String,
      args : Array(JSON::Any),
      priority : Int32 = 0,
      delay : Time::Span = 0.seconds,
      retry_count : Int32 = 0
    ) : String
      job_id = UUID.random.to_s
      
      job_data = {
        "id"          => job_id,
        "class"       => job_class,
        "args"        => args,
        "queue"       => queue_name,
        "priority"    => priority,
        "retry_count" => retry_count,
        "max_retries" => 3,
        "created_at"  => Time.utc.to_unix,
        "run_at"      => (Time.utc + delay).to_unix
      }.to_json
      
      if delay > 0.seconds
        # เพิ่มใน scheduled queue (sorted set ตาม timestamp)
        score = (Time.utc + delay).to_unix_f
        @redis.zadd("#{QUEUE_PREFIX}scheduled", score, job_data)
      else
        # เพิ่มใน priority queue (sorted set ตาม priority)
        score = -priority.to_f  # ยิ่ง priority สูง ยิ่งทำงานก่อน
        @redis.zadd("#{QUEUE_PREFIX}#{queue_name}", score, job_data)
      end
      
      puts "เพิ่มงาน #{job_class} (#{job_id}) เข้า queue '#{queue_name}'"
      job_id
    end
    
    # ดึงงานจาก queue
    def dequeue(queue_name : String) : Hash(String, JSON::Any)?
      key = "#{QUEUE_PREFIX}#{queue_name}"
      
      # ดึงงานที่มี priority สูงสุด (score ต่ำสุด)
      results = @redis.zrangebyscore(key, "-inf", "+inf", limit: [0, 1])
      return nil if results.empty?
      
      job_json = results.first.as(String)
      
      # ลบออกจาก queue
      @redis.zrem(key, job_json)
      
      # ย้ายไป processing queue
      processing_data = {
        "job"       => JSON.parse(job_json),
        "worker_id" => Process.pid.to_s,
        "started_at" => Time.utc.to_unix
      }.to_json
      
      job = JSON.parse(job_json)
      job_id = job["id"].as_s
      
      @redis.setex("#{PROCESSING_PREFIX}#{job_id}", 300, processing_data)  # 5 min timeout
      
      job.as_h
    end
    
    # งานสำเร็จ
    def complete(job_id : String)
      @redis.del("#{PROCESSING_PREFIX}#{job_id}")
    end
    
    # งานล้มเหลว
    def fail(job_id : String, queue_name : String, error : String, job_data : Hash(String, JSON::Any))
      @redis.del("#{PROCESSING_PREFIX}#{job_id}")
      
      retry_count = job_data["retry_count"].as_i
      max_retries = job_data["max_retries"]?.try(&.as_i) || 3
      
      if retry_count < max_retries
        # Retry ด้วย exponential backoff
        delay = (2 ** retry_count).seconds
        new_data = job_data.dup
        new_data["retry_count"] = JSON::Any.new((retry_count + 1).to_i64)
        new_data["last_error"] = JSON::Any.new(error)
        new_data["run_at"] = JSON::Any.new((Time.utc + delay).to_unix.to_i64)
        
        score = (Time.utc + delay).to_unix_f
        @redis.zadd("#{QUEUE_PREFIX}scheduled", score, new_data.to_json)
        puts "ลองใหม่ #{retry_count + 1}/#{max_retries}: #{job_id} หลัง #{delay}"
      else
        # บันทึกใน failed queue
        failed_data = job_data.merge({
          "failed_at" => JSON::Any.new(Time.utc.to_unix.to_i64),
          "error" => JSON::Any.new(error)
        })
        @redis.lpush("#{FAILED_PREFIX}#{queue_name}", failed_data.to_json)
        @redis.ltrim("#{FAILED_PREFIX}#{queue_name}", 0, 999)  # เก็บแค่ 1000 รายการ
        puts "งานล้มเหลวถาวร: #{job_id}"
      end
    end
    
    # ย้ายงาน scheduled ที่ถึงเวลาแล้วไปยัง queue ปกติ
    def move_scheduled_jobs
      now = Time.utc.to_unix_f
      
      # ดึงงานที่ถึงเวลาแล้ว
      jobs = @redis.zrangebyscore("#{QUEUE_PREFIX}scheduled", "-inf", now.to_s)
      
      jobs.each do |job_json|
        @redis.zrem("#{QUEUE_PREFIX}scheduled", job_json)
        
        job = JSON.parse(job_json)
        queue_name = job["queue"].as_s
        priority = job["priority"].as_i
        
        score = -priority.to_f
        @redis.zadd("#{QUEUE_PREFIX}#{queue_name}", score, job_json)
      end
      
      jobs.size
    end
    
    # สถิติ queue
    def stats(queue_name : String) : NamedTuple(
      pending: Int64,
      processing: Int64,
      failed: Int64,
      scheduled: Int64
    )
      {
        pending:    @redis.zcard("#{QUEUE_PREFIX}#{queue_name}"),
        processing: @redis.keys("#{PROCESSING_PREFIX}*").size.to_i64,
        failed:     @redis.llen("#{FAILED_PREFIX}#{queue_name}"),
        scheduled:  @redis.zcard("#{QUEUE_PREFIX}scheduled")
      }
    end
    
    # Queue length
    def length(queue_name : String) : Int64
      @redis.zcard("#{QUEUE_PREFIX}#{queue_name}")
    end
    
    # ดูงานที่ล้มเหลว
    def failed_jobs(queue_name : String, limit : Int32 = 10) : Array(JSON::Any)
      jobs = @redis.lrange("#{FAILED_PREFIX}#{queue_name}", 0, limit - 1)
      jobs.map { |j| JSON.parse(j.as(String)) }
    end
    
    # Retry งานที่ล้มเหลว
    def retry_failed(queue_name : String, job_id : String) : Bool
      failed = failed_jobs(queue_name, 100)
      job = failed.find { |j| j["id"].as_s == job_id }
      return false unless job
      
      # Reset retry count
      new_data = job.as_h.dup
      new_data["retry_count"] = JSON::Any.new(0_i64)
      new_data.delete("failed_at")
      new_data.delete("error")
      
      priority = job["priority"].as_i
      @redis.zadd("#{QUEUE_PREFIX}#{queue_name}", -priority.to_f, new_data.to_json)
      
      # ลบออกจาก failed list
      @redis.lrem("#{FAILED_PREFIX}#{queue_name}", 1, job.to_json)
      
      true
    end
  end
end
```

## Worker Process

### Worker หลักที่ประมวลผลงาน

```crystal
# src/workers/worker.cr
require "../queue/redis_queue"
require "../jobs/*"

class Worker
  QUEUES = ["critical", "default", "low"]
  
  def initialize(
    @queue : Queue::RedisQueue,
    @worker_id : String = UUID.random.to_s[0..7]
  )
    @running = true
    @jobs_processed = 0
    @jobs_failed = 0
  end
  
  # เริ่ม worker
  def start
    puts "Worker #{@worker_id} เริ่มทำงาน"
    puts "รับผิดชอบ queues: #{QUEUES.join(", ")}"
    
    # Signal handler สำหรับ graceful shutdown
    Signal::INT.trap do
      puts "\nหยุด worker..."
      @running = false
    end
    
    Signal::TERM.trap do
      puts "\nได้รับ SIGTERM, หยุด worker..."
      @running = false
    end
    
    # Scheduler fiber สำหรับ scheduled jobs
    spawn do
      while @running
        moved = @queue.move_scheduled_jobs
        puts "ย้าย #{moved} scheduled jobs" if moved > 0
        sleep(1.second)
      end
    end
    
    # Main loop
    while @running
      processed = false
      
      QUEUES.each do |queue_name|
        job_data = @queue.dequeue(queue_name)
        next unless job_data
        
        process_job(queue_name, job_data)
        processed = true
        break  # ประมวลผลทีละงาน แล้วกลับไปเช็ค priority queue อีกครั้ง
      end
      
      # ถ้าไม่มีงาน รอสักครู่
      sleep(0.1.seconds) unless processed
    end
    
    puts "Worker #{@worker_id} หยุดทำงาน"
    puts "ประมวลผล #{@jobs_processed} งาน, ล้มเหลว #{@jobs_failed} งาน"
  end
  
  private def process_job(queue_name : String, job_data : Hash(String, JSON::Any))
    job_id    = job_data["id"].as_s
    job_class = job_data["class"].as_s
    args      = job_data["args"].as_a
    
    puts "[#{@worker_id}] ทำงาน: #{job_class} (#{job_id})"
    start_time = Time.monotonic
    
    begin
      # สร้าง job instance และรัน
      job = create_job(job_class, args)
      job.perform
      
      @queue.complete(job_id)
      @jobs_processed += 1
      
      elapsed = (Time.monotonic - start_time).total_milliseconds
      puts "[#{@worker_id}] สำเร็จ: #{job_class} ใช้เวลา #{elapsed.round(1)}ms"
      
    rescue ex
      @jobs_failed += 1
      @queue.fail(job_id, queue_name, ex.message || "Unknown error", job_data)
      
      elapsed = (Time.monotonic - start_time).total_milliseconds
      puts "[#{@worker_id}] ล้มเหลว: #{job_class} - #{ex.message} (#{elapsed.round(1)}ms)"
    end
  end
  
  private def create_job(class_name : String, args : Array(JSON::Any)) : Job
    case class_name
    when "SendEmailJob"
      SendEmailJob.new(
        to:      args[0].as_s,
        subject: args[1].as_s,
        body:    args[2].as_s
      )
    when "ProcessImageJob"
      ProcessImageJob.new(
        file_path:   args[0].as_s,
        output_path: args[1].as_s,
        width:       args[2].as_i,
        height:      args[3].as_i
      )
    when "SendNotificationJob"
      SendNotificationJob.new(
        user_id: args[0].as_i64,
        message: args[1].as_s
      )
    when "ReportGenerationJob"
      ReportGenerationJob.new(
        report_type: args[0].as_s,
        user_id:     args[1].as_i64
      )
    else
      raise "ไม่รู้จัก Job class: #{class_name}"
    end
  end
end
```

## Job Scheduling ด้วย Cron-like Pattern

### Job Scheduler

```crystal
# src/scheduler/job_scheduler.cr
require "cron_parser"

class JobScheduler
  record ScheduledTask,
    name : String,
    cron_expression : String,
    job_class : String,
    args : Array(JSON::Any),
    queue : String,
    next_run : Time
  
  def initialize(@queue : Queue::RedisQueue)
    @tasks = [] of ScheduledTask
    @running = false
  end
  
  # เพิ่ม cron task
  def schedule(
    name : String,
    cron : String,
    job_class : String,
    args : Array(JSON::Any) = [] of JSON::Any,
    queue : String = "default"
  )
    next_run = calculate_next_run(cron)
    
    @tasks << ScheduledTask.new(
      name: name,
      cron_expression: cron,
      job_class: job_class,
      args: args,
      queue: queue,
      next_run: next_run
    )
    
    puts "เพิ่ม task '#{name}' - รันถัดไป: #{next_run}"
  end
  
  # เริ่ม scheduler
  def start
    @running = true
    puts "Job Scheduler เริ่มทำงาน"
    puts "Tasks ที่กำหนด: #{@tasks.size}"
    
    spawn do
      while @running
        now = Time.utc
        
        @tasks.each_with_index do |task, i|
          if now >= task.next_run
            puts "รัน scheduled task: #{task.name}"
            
            @queue.enqueue(
              queue_name: task.queue,
              job_class: task.job_class,
              args: task.args
            )
            
            # คำนวณเวลารันครั้งต่อไป
            @tasks[i] = ScheduledTask.new(
              name: task.name,
              cron_expression: task.cron_expression,
              job_class: task.job_class,
              args: task.args,
              queue: task.queue,
              next_run: calculate_next_run(task.cron_expression)
            )
          end
        end
        
        sleep(1.second)
      end
    end
  end
  
  def stop
    @running = false
    puts "Job Scheduler หยุดทำงาน"
  end
  
  private def calculate_next_run(cron : String) : Time
    # Simple cron parser สำหรับ demo
    # ใน production ใช้ library จริงๆ
    parts = cron.split(" ")
    now = Time.utc
    
    case cron
    when "* * * * *"        # ทุกนาที
      now + 1.minute
    when "*/5 * * * *"      # ทุก 5 นาที
      now + 5.minutes
    when "0 * * * *"        # ทุกชั่วโมง
      now + 1.hour
    when "0 0 * * *"        # ทุกวัน เที่ยงคืน
      now + 1.day
    when "0 0 * * 1"        # ทุกสัปดาห์ จันทร์
      now + 7.days
    else
      now + 1.minute  # default
    end
  end
  
  # แสดงสถานะ tasks
  def status
    puts "\nJob Scheduler Status:"
    puts "-" * 60
    @tasks.each do |task|
      time_until = task.next_run - Time.utc
      puts "#{task.name.ljust(30)} รันอีก: #{format_duration(time_until)}"
    end
  end
  
  private def format_duration(span : Time::Span) : String
    if span.total_seconds < 60
      "#{span.total_seconds.round.to_i}วินาที"
    elsif span.total_minutes < 60
      "#{span.total_minutes.round.to_i}นาที"
    else
      "#{span.total_hours.round(1)}ชั่วโมง"
    end
  end
end
```

## Error Handling และ Retries

### Retry Strategy

```crystal
# src/jobs/retry_strategy.cr
module RetryStrategy
  # Linear backoff: รอ N วินาทีตาม attempt
  def self.linear(attempt : Int32, base_delay : Int32 = 5) : Time::Span
    (base_delay * attempt).seconds
  end
  
  # Exponential backoff: 2^attempt วินาที
  def self.exponential(attempt : Int32, max_delay : Int32 = 3600) : Time::Span
    delay = [2 ** attempt, max_delay].min
    delay.seconds
  end
  
  # Exponential backoff พร้อม jitter
  def self.exponential_with_jitter(attempt : Int32, max_delay : Int32 = 3600) : Time::Span
    base = [2 ** attempt, max_delay].min
    jitter = rand(base / 4)
    (base + jitter).seconds
  end
  
  # Fixed delay
  def self.fixed(delay_seconds : Int32 = 60) : Time::Span
    delay_seconds.seconds
  end
end

# Job พร้อม retry logic
abstract class RetryableJob < Job
  abstract def max_retries : Int32
  abstract def retry_strategy : RetryStrategy.class
  
  def should_retry?(error : Exception, attempt : Int32) : Bool
    attempt < max_retries
  end
  
  def retry_delay(attempt : Int32) : Time::Span
    RetryStrategy.exponential_with_jitter(attempt)
  end
end

# ตัวอย่าง Job ที่มี retry
class SendWebhookJob < RetryableJob
  def initialize(
    @url : String,
    @payload : String,
    @secret : String
  )
  end
  
  def max_retries : Int32
    5
  end
  
  def retry_strategy : RetryStrategy.class
    RetryStrategy
  end
  
  def perform
    puts "ส่ง webhook ไปยัง #{@url}"
    
    # จำลองการส่ง HTTP request
    response = HTTP::Client.post(
      @url,
      headers: HTTP::Headers{
        "Content-Type" => "application/json",
        "X-Webhook-Secret" => @secret
      },
      body: @payload
    )
    
    unless response.success?
      raise "Webhook ล้มเหลว: HTTP #{response.status_code}"
    end
    
    puts "ส่ง webhook สำเร็จ!"
  end
  
  def should_retry?(error : Exception, attempt : Int32) : Bool
    # Retry เฉพาะ network errors, ไม่ retry 4xx errors
    case error.message
    when /HTTP 4[0-9]{2}/
      false  # Client error ไม่ retry
    else
      attempt < max_retries
    end
  end
end
```

## Monitoring Jobs

### Job Monitor Dashboard

```crystal
# src/monitoring/job_monitor.cr
require "http/server"
require "json"
require "../queue/redis_queue"

class JobMonitor
  def initialize(@queue : Queue::RedisQueue)
  end
  
  def start_dashboard(port : Int32 = 9090)
    server = HTTP::Server.new do |context|
      context.response.content_type = "application/json"
      
      case context.request.path
      when "/health"
        context.response.print({"status" => "ok"}.to_json)
        
      when /^\/stats\/(.+)/
        queue_name = $1
        stats = @queue.stats(queue_name)
        context.response.print(stats.to_json)
        
      when /^\/failed\/(.+)/
        queue_name = $1
        failed = @queue.failed_jobs(queue_name)
        context.response.print(failed.map(&.to_json).to_json)
        
      when "/overview"
        overview = generate_overview
        context.response.print(overview.to_json)
        
      else
        context.response.status = HTTP::Status::NOT_FOUND
        context.response.print({"error" => "Not Found"}.to_json)
      end
    end
    
    puts "Job Monitor Dashboard ที่ http://localhost:#{port}"
    server.listen("0.0.0.0", port)
  end
  
  private def generate_overview : Hash(String, JSON::Any)
    queues = ["critical", "default", "low"]
    
    stats = queues.map do |q|
      s = @queue.stats(q)
      {
        "queue"      => q,
        "pending"    => s[:pending],
        "processing" => s[:processing],
        "failed"     => s[:failed],
        "scheduled"  => s[:scheduled]
      }
    end
    
    {
      "timestamp" => JSON::Any.new(Time.utc.to_rfc3339),
      "queues"    => JSON::Any.new(stats.map { |s| JSON.parse(s.to_json) })
    }
  end
end

# Metrics logger
class MetricsLogger
  def initialize(@output : IO = STDOUT)
  end
  
  def log_job_start(job_class : String, job_id : String)
    log({
      "event"      => "job_started",
      "job_class"  => job_class,
      "job_id"     => job_id,
      "timestamp"  => Time.utc.to_rfc3339
    })
  end
  
  def log_job_success(job_class : String, job_id : String, duration_ms : Float64)
    log({
      "event"       => "job_completed",
      "job_class"   => job_class,
      "job_id"      => job_id,
      "duration_ms" => duration_ms,
      "timestamp"   => Time.utc.to_rfc3339
    })
  end
  
  def log_job_failure(job_class : String, job_id : String, error : String, retry_count : Int32)
    log({
      "event"       => "job_failed",
      "job_class"   => job_class,
      "job_id"      => job_id,
      "error"       => error,
      "retry_count" => retry_count,
      "timestamp"   => Time.utc.to_rfc3339
    })
  end
  
  private def log(data : Hash)
    @output.puts(data.to_json)
    @output.flush
  end
end
```

## Production Patterns

### Worker Pool Management

```crystal
# src/workers/worker_pool.cr
class WorkerPool
  def initialize(
    @queue : Queue::RedisQueue,
    @pool_size : Int32 = 5
  )
    @workers = [] of Worker
    @running = false
  end
  
  def start
    @running = true
    puts "เริ่ม Worker Pool ขนาด #{@pool_size}"
    
    # สร้าง workers
    @pool_size.times do |i|
      worker = Worker.new(@queue, "worker-#{i + 1}")
      @workers << worker
      
      spawn { worker.start }
    end
    
    puts "Worker Pool พร้อมทำงาน"
    
    # Monitor loop
    monitor_loop
  end
  
  def stop
    @running = false
    puts "หยุด Worker Pool..."
  end
  
  private def monitor_loop
    spawn do
      while @running
        sleep(30.seconds)
        print_stats
      end
    end
  end
  
  private def print_stats
    puts "\n=== Worker Pool Stats ==="
    puts "Workers ทำงาน: #{@workers.size}"
    puts "เวลา: #{Time.utc.to_s("%Y-%m-%d %H:%M:%S")}"
    
    ["critical", "default", "low"].each do |queue_name|
      stats = @queue.stats(queue_name)
      puts "Queue '#{queue_name}': pending=#{stats[:pending]}, failed=#{stats[:failed]}"
    end
  end
end
```

### Application Integration

```crystal
# src/app.cr
require "redis"
require "./queue/redis_queue"
require "./workers/worker_pool"
require "./scheduler/job_scheduler"
require "./monitoring/job_monitor"
require "./jobs/*"

# เชื่อมต่อ Redis
queue = Queue::RedisQueue.new(
  host: ENV.fetch("REDIS_HOST", "localhost"),
  port: ENV.fetch("REDIS_PORT", "6379").to_i
)

# ตั้งค่า Job Scheduler
scheduler = JobScheduler.new(queue)

# กำหนดงานประจำ
scheduler.schedule(
  name:       "cleanup_old_sessions",
  cron:       "0 2 * * *",   # ทุกวัน ตี 2
  job_class:  "CleanupSessionsJob",
  queue:      "low"
)

scheduler.schedule(
  name:       "send_daily_digest",
  cron:       "0 8 * * *",   # ทุกวัน 8 โมงเช้า
  job_class:  "DailyDigestJob",
  queue:      "default"
)

scheduler.schedule(
  name:       "generate_reports",
  cron:       "0 6 * * 1",   # ทุกจันทร์ 6 โมงเช้า
  job_class:  "WeeklyReportJob",
  queue:      "low"
)

scheduler.schedule(
  name:       "health_check",
  cron:       "*/5 * * * *",  # ทุก 5 นาที
  job_class:  "HealthCheckJob",
  queue:      "critical"
)

# เริ่ม scheduler
scheduler.start

# ตัวอย่างการเพิ่มงานจาก application code
puts "\n=== เพิ่มงานทดสอบ ==="

# Email job
queue.enqueue(
  queue_name: "default",
  job_class:  "SendEmailJob",
  args:       [JSON::Any.new("user@example.com"), JSON::Any.new("ยืนยันบัญชี"), JSON::Any.new("คลิกลิงก์เพื่อยืนยัน")]
)

# High priority notification
queue.enqueue(
  queue_name: "critical",
  job_class:  "SendNotificationJob",
  args:       [JSON::Any.new(123_i64), JSON::Any.new("ระบบตรวจพบความผิดปกติ!")],
  priority:   10
)

# Delayed job (รัน 5 นาทีข้างหน้า)
queue.enqueue(
  queue_name: "default",
  job_class:  "SendEmailJob",
  args:       [JSON::Any.new("promo@example.com"), JSON::Any.new("โปรโมชั่น"), JSON::Any.new("ลด 50%!")],
  delay:      5.minutes
)

# แสดงสถิติ
["critical", "default", "low"].each do |q|
  stats = queue.stats(q)
  puts "Queue '#{q}': #{stats}"
end

# เริ่ม Worker Pool
pool = WorkerPool.new(queue, pool_size: 3)
pool.start
```

### การใช้ Sidekiq.cr (Alternative)

```crystal
# shard.yml พร้อม Sidekiq
# dependencies:
#   sidekiq:
#     github: mperham/sidekiq.cr
#     version: ~> 6.0

# src/jobs/sidekiq_example.cr
# require "sidekiq"
# 
# class EmailWorker
#   include Sidekiq::Worker
#   
#   sidekiq_options queue: "emails", retry: 3
#   
#   def perform(to : String, subject : String, body : String)
#     puts "ส่งอีเมล: #{to}"
#     # ส่งอีเมลจริงๆ
#   end
# end
# 
# # เพิ่มงาน
# EmailWorker.async.perform("user@test.com", "Test", "Hello!")
# 
# # กำหนดเวลา
# EmailWorker.async(in: 5.minutes).perform("user@test.com", "Reminder", "อย่าลืม!")
```

## สรุป

ในบทนี้เราได้เรียนรู้:
- การสร้าง In-Memory Queue ด้วย Crystal Channels
- Redis-backed Queue พร้อม Priority, Delay และ Retry
- Worker Process ที่จัดการงานหลายชนิด
- Job Scheduler ด้วย Cron-like expressions
- Error Handling และ Retry Strategy แบบต่างๆ
- Monitoring Dashboard สำหรับดูสถานะ
- Worker Pool สำหรับ Production

## ขั้นตอนต่อไป

ใน **Part 197** เราจะสำรวจ **Crystal Ecosystem และ Community** รวมถึง shards ยอดนิยม, แหล่งข้อมูล และทิศทางอนาคตของภาษา Crystal
