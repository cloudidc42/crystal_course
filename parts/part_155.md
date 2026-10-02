# Part 155: Background Jobs และ Queue ใน Crystal

## บทนำ

ในการพัฒนาแอปพลิเคชันจริง มักมีงานที่ต้องทำในเบื้องหลัง (Background Jobs) เช่น การส่งอีเมล, การประมวลผลรูปภาพ, การนำเข้าข้อมูล หรืองานที่ใช้เวลานาน ในบทนี้เราจะเรียนรู้วิธีสร้าง Background Job System ใน Crystal

## ทำไมต้องใช้ Background Jobs?

```crystal
# ปัญหา: งานที่ใช้เวลานาน block HTTP request
post "/users" do |env|
  user = create_user(env.params)
  
  # ❌ บล็อก request 5-10 วินาที
  send_welcome_email(user)        # ช้า
  resize_profile_image(user)      # ช้า
  update_search_index(user)       # ช้า
  
  env.response.print user.to_json
end

# ✓ วิธีที่ดีกว่า: ทำงานใน background
post "/users" do |env|
  user = create_user(env.params)
  
  # ส่งงานเข้า queue แล้วตอบกลับทันที
  WelcomeEmailJob.perform_later(user.id)
  ImageResizeJob.perform_later(user.id)
  SearchIndexJob.perform_later(user.id)
  
  env.response.status_code = 201
  env.response.print user.to_json
end
```

## Simple In-Memory Queue

```crystal
# simple_queue.cr
require "json"

# โครงสร้าง Job พื้นฐาน
abstract class Job
  abstract def perform

  def self.name : String
    {{ @type.name.stringify }}
  end
end

# In-memory job queue
class JobQueue
  @@instance = new
  @@mutex = Mutex.new

  def self.instance
    @@instance
  end

  def initialize
    @queue = Channel(Job).new(1000)
    @running = false
  end

  def push(job : Job)
    @queue.send(job)
  end

  def start_workers(count : Int32 = 4)
    @running = true
    count.times do |i|
      spawn do
        puts "Worker #{i} started"
        while @running
          job = @queue.receive
          begin
            job.perform
          rescue ex
            puts "Job failed: #{ex.message}"
          end
        end
      end
    end
  end

  def stop
    @running = false
  end
end

# ตัวอย่าง Jobs
class SendEmailJob < Job
  def initialize(@to : String, @subject : String, @body : String)
  end

  def perform
    puts "Sending email to #{@to}: #{@subject}"
    sleep 0.1  # จำลองการส่งอีเมล
    puts "Email sent to #{@to}"
  end
end

class ResizeImageJob < Job
  def initialize(@user_id : Int32, @image_path : String)
  end

  def perform
    puts "Resizing image #{@image_path} for user #{@user_id}"
    sleep 0.2  # จำลองการ resize
    puts "Image resized for user #{@user_id}"
  end
end

# การใช้งาน
queue = JobQueue.instance
queue.start_workers(4)

# เพิ่มงานเข้า queue
queue.push(SendEmailJob.new("alice@example.com", "Welcome!", "Welcome to our app!"))
queue.push(ResizeImageJob.new(1, "/uploads/avatar.jpg"))
queue.push(SendEmailJob.new("bob@example.com", "Verify Email", "Please verify your email"))

sleep 2  # รอให้งานเสร็จ
queue.stop
```

## Redis-backed Job Queue

```crystal
# redis_queue.cr
require "redis"
require "json"
require "uuid"

# Job serialization
struct JobPayload
  include JSON::Serializable

  property job_class : String
  property args : Array(JSON::Any)
  property id : String
  property created_at : Time
  property attempts : Int32
  property max_attempts : Int32

  def initialize(@job_class : String, @args : Array(JSON::Any),
                 @max_attempts : Int32 = 3)
    @id = UUID.random.to_s
    @created_at = Time.utc
    @attempts = 0
  end
end

# Abstract base job
abstract class BaseJob
  abstract def perform(args : Array(JSON::Any))

  def self.job_name : String
    name
  end

  def self.perform_later(*args)
    payload_args = args.map { |a| JSON::Any.new(a.to_json) }
    RedisJobQueue.instance.enqueue(job_name, payload_args)
  end
end

# Redis Job Queue
class RedisJobQueue
  QUEUE_KEY    = "crystal:jobs:default"
  FAILED_KEY   = "crystal:jobs:failed"
  RETRY_KEY    = "crystal:jobs:retry"

  @@instance : RedisJobQueue?

  def self.instance
    @@instance ||= new
  end

  def initialize
    @redis = Redis::Client.new
    @registry = {} of String => BaseJob.class
  end

  def register(job_class : BaseJob.class)
    @registry[job_class.job_name] = job_class
  end

  def enqueue(job_class : String, args : Array(JSON::Any), delay : Time::Span? = nil)
    payload = JobPayload.new(job_class, args)

    if delay
      score = (Time.utc + delay).to_unix_f
      @redis.zadd(RETRY_KEY, score, payload.to_json)
    else
      @redis.lpush(QUEUE_KEY, payload.to_json)
    end

    puts "Enqueued job #{payload.id} (#{job_class})"
    payload.id
  end

  def process_due_retries
    now = Time.utc.to_unix_f
    jobs = @redis.zrangebyscore(RETRY_KEY, "-inf", now.to_s)

    jobs.each do |job_json|
      @redis.zrem(RETRY_KEY, job_json)
      @redis.lpush(QUEUE_KEY, job_json)
    end
  end

  def work(queue : String = QUEUE_KEY)
    puts "Worker started, listening on #{queue}"

    loop do
      # Move due retries back to main queue
      process_due_retries

      # Blocking pop with 1 second timeout
      result = @redis.brpop(queue, 1)
      next unless result

      _, job_json = result
      process_job(job_json)
    end
  end

  private def process_job(job_json : String)
    payload = JobPayload.from_json(job_json)
    job_class = @registry[payload.job_class]?

    unless job_class
      puts "Unknown job class: #{payload.job_class}"
      return
    end

    payload = JobPayload.new(payload.job_class, payload.args, payload.max_attempts)

    begin
      puts "Processing job #{payload.id} (#{payload.job_class}), attempt #{payload.attempts + 1}"
      job = job_class.new
      job.perform(payload.args)
      puts "Job #{payload.id} completed successfully"
    rescue ex
      handle_failure(payload, ex)
    end
  end

  private def handle_failure(payload : JobPayload, ex : Exception)
    puts "Job #{payload.id} failed: #{ex.message}"

    if payload.attempts < payload.max_attempts - 1
      # Exponential backoff retry
      delay = (2 ** payload.attempts).seconds
      puts "Retrying job #{payload.id} in #{delay.total_seconds}s"

      retry_payload = JobPayload.new(payload.job_class, payload.args, payload.max_attempts)
      score = (Time.utc + delay).to_unix_f
      @redis.zadd(RETRY_KEY, score, retry_payload.to_json)
    else
      puts "Job #{payload.id} exhausted retries, moving to failed queue"
      failed_data = {
        payload: payload,
        error: ex.message,
        backtrace: ex.backtrace?.try(&.first(5)),
        failed_at: Time.utc.to_rfc3339
      }
      @redis.lpush(FAILED_KEY, failed_data.to_json)
    end
  end
end
```

## Job Classes ตัวอย่าง

```crystal
# jobs/email_job.cr
class EmailJob < BaseJob
  def perform(args : Array(JSON::Any))
    to = args[0].as_s
    subject = args[1].as_s
    body = args[2].as_s

    puts "📧 Sending email to #{to}: #{subject}"
    # ใช้ SMTP หรือ email service จริง
    simulate_email_send(to, subject, body)
  end

  private def simulate_email_send(to : String, subject : String, body : String)
    sleep 0.5
    puts "✅ Email delivered to #{to}"
  end
end

# jobs/image_job.cr
class ImageProcessingJob < BaseJob
  def perform(args : Array(JSON::Any))
    user_id = args[0].as_i
    image_path = args[1].as_s
    operations = args[2].as_a.map(&.as_s)

    puts "🖼️ Processing image #{image_path} for user #{user_id}"
    operations.each do |op|
      case op
      when "resize"   then resize_image(image_path)
      when "compress" then compress_image(image_path)
      when "watermark" then add_watermark(image_path)
      end
    end
    puts "✅ Image processing complete for user #{user_id}"
  end

  private def resize_image(path : String)
    sleep 0.3
    puts "  Resized: #{path}"
  end

  private def compress_image(path : String)
    sleep 0.2
    puts "  Compressed: #{path}"
  end

  private def add_watermark(path : String)
    sleep 0.1
    puts "  Watermarked: #{path}"
  end
end

# jobs/report_job.cr
class GenerateReportJob < BaseJob
  def perform(args : Array(JSON::Any))
    report_type = args[0].as_s
    user_id = args[1].as_i
    date_range = args[2].as_s

    puts "📊 Generating #{report_type} report for user #{user_id}"
    sleep 1.0  # จำลองการสร้าง report

    report_path = "/reports/#{user_id}/#{report_type}_#{Time.utc.to_unix}.pdf"
    puts "✅ Report saved to #{report_path}"

    # แจ้งเตือนผู้ใช้ว่า report พร้อมแล้ว
    EmailJob.perform_later(
      "user#{user_id}@example.com",
      "Your report is ready",
      "Download your report at #{report_path}"
    )
  end
end
```

## Job Scheduling

```crystal
# scheduler.cr - Cron-like job scheduler

struct CronSchedule
  getter minute : String
  getter hour : String
  getter day_of_month : String
  getter month : String
  getter day_of_week : String

  def initialize(cron_expr : String)
    parts = cron_expr.split
    raise ArgumentError.new("Invalid cron expression") if parts.size != 5
    @minute, @hour, @day_of_month, @month, @day_of_week = parts
  end

  def next_run(from : Time = Time.utc) : Time
    # Simplified: calculate next run time
    # In production use a proper cron parser
    from + 1.minute
  end

  def matches?(time : Time) : Bool
    check_field(@minute, time.minute) &&
    check_field(@hour, time.hour) &&
    check_field(@day_of_month, time.day) &&
    check_field(@month, time.month) &&
    check_field(@day_of_week, time.day_of_week.value)
  end

  private def check_field(field : String, value : Int32) : Bool
    return true if field == "*"

    if field.includes?("/")
      parts = field.split("/")
      interval = parts[1].to_i
      return value % interval == 0
    end

    if field.includes?(",")
      return field.split(",").any? { |f| f.to_i == value }
    end

    if field.includes?("-")
      parts = field.split("-")
      return value >= parts[0].to_i && value <= parts[1].to_i
    end

    field.to_i == value
  end
end

class JobScheduler
  record ScheduledJob, name : String, cron : CronSchedule, job_class : BaseJob.class, args : Array(JSON::Any)

  def initialize
    @jobs = [] of ScheduledJob
    @running = false
  end

  def schedule(name : String, cron_expr : String, job_class : BaseJob.class, *args)
    cron = CronSchedule.new(cron_expr)
    json_args = args.map { |a| JSON::Any.new(a.to_json) }
    @jobs << ScheduledJob.new(name, cron, job_class, json_args)
    puts "Scheduled job '#{name}' with cron: #{cron_expr}"
  end

  def start
    @running = true
    spawn do
      loop do
        break unless @running
        now = Time.utc

        @jobs.each do |job|
          if job.cron.matches?(now)
            puts "⏰ Running scheduled job: #{job.name}"
            spawn { job.job_class.new.perform(job.args) }
          end
        end

        # Sleep until next minute
        next_minute = now.at_beginning_of_minute + 1.minute
        sleep_duration = next_minute - Time.utc
        sleep sleep_duration.total_seconds if sleep_duration.total_seconds > 0
      end
    end
  end

  def stop
    @running = false
  end
end

# การใช้งาน scheduler
scheduler = JobScheduler.new

# ทุกนาที
scheduler.schedule("heartbeat", "* * * * *", EmailJob, "admin@example.com", "Heartbeat", "Server is alive")

# ทุกวันตี 2
scheduler.schedule("daily_report", "0 2 * * *", GenerateReportJob, "daily", 0, "yesterday")

# ทุกวันจันทร์เวลา 9 โมง
scheduler.schedule("weekly_summary", "0 9 * * 1", GenerateReportJob, "weekly", 0, "last_week")

scheduler.start
```

## Worker Pool Pattern

```crystal
# worker_pool.cr

class WorkerPool
  getter active_count : Atomic(Int32)
  getter processed_count : Atomic(Int32)
  getter failed_count : Atomic(Int32)

  def initialize(@size : Int32, @queue : RedisJobQueue)
    @active_count = Atomic(Int32).new(0)
    @processed_count = Atomic(Int32).new(0)
    @failed_count = Atomic(Int32).new(0)
    @workers = [] of Fiber
    @running = false
  end

  def start
    @running = true

    @size.times do |i|
      fiber = spawn do
        puts "Worker #{i + 1}/#{@size} started"
        worker_loop(i + 1)
      end
      @workers << fiber
    end

    # Stats reporter
    spawn do
      loop do
        sleep 10
        break unless @running
        print_stats
      end
    end

    puts "Worker pool started with #{@size} workers"
  end

  def stop
    @running = false
    print_stats
    puts "Worker pool stopped"
  end

  def print_stats
    puts "📊 Stats: active=#{@active_count.get}, processed=#{@processed_count.get}, failed=#{@failed_count.get}"
  end

  private def worker_loop(worker_id : Int32)
    loop do
      break unless @running

      @active_count.add(1)
      begin
        @queue.work
      rescue ex
        @failed_count.add(1)
        puts "Worker #{worker_id} error: #{ex.message}"
        sleep 1
      ensure
        @active_count.sub(1)
      end
    end
  end
end
```

## Graceful Shutdown

```crystal
# graceful_shutdown.cr

class GracefulWorker
  def initialize(@queue : RedisJobQueue, @workers : Int32 = 4)
    @pool = WorkerPool.new(@workers, @queue)
    @shutting_down = false
  end

  def start
    # จัดการ signals สำหรับ graceful shutdown
    Signal::INT.trap do
      puts "\nReceived INT signal, shutting down gracefully..."
      shutdown
    end

    Signal::TERM.trap do
      puts "\nReceived TERM signal, shutting down gracefully..."
      shutdown
    end

    @pool.start
    puts "Worker service started. Press Ctrl+C to stop."

    # Keep main fiber alive
    loop do
      sleep 1
      break if @shutting_down
    end
  end

  private def shutdown
    @shutting_down = true
    puts "Waiting for active jobs to complete..."
    @pool.stop
    puts "Shutdown complete."
    exit 0
  end
end

# main.cr
queue = RedisJobQueue.instance

# ลงทะเบียน job classes
queue.register(EmailJob)
queue.register(ImageProcessingJob)
queue.register(GenerateReportJob)

worker = GracefulWorker.new(queue, workers: 4)
worker.start
```

## Job Monitoring Dashboard

```crystal
# monitor.cr - Simple monitoring via HTTP

require "http/server"

class JobMonitor
  def initialize(@redis : Redis::Client, @port : Int32 = 3001)
  end

  def start
    server = HTTP::Server.new do |context|
      case context.request.path
      when "/status"
        handle_status(context)
      when "/jobs/failed"
        handle_failed_jobs(context)
      when "/jobs/retry"
        handle_retry_jobs(context)
      when "/jobs/clear-failed"
        handle_clear_failed(context)
      else
        context.response.status_code = 404
        context.response.print "Not Found"
      end
    end

    puts "Job monitor running on http://localhost:#{@port}"
    server.listen("0.0.0.0", @port)
  end

  private def handle_status(ctx)
    pending = @redis.llen("crystal:jobs:default")
    failed = @redis.llen("crystal:jobs:failed")
    retry_count = @redis.zcard("crystal:jobs:retry")

    status = {
      pending_jobs: pending,
      failed_jobs:  failed,
      retry_jobs:   retry_count,
      timestamp:    Time.utc.to_rfc3339
    }

    ctx.response.content_type = "application/json"
    ctx.response.print status.to_json
  end

  private def handle_failed_jobs(ctx)
    jobs = @redis.lrange("crystal:jobs:failed", 0, 49)
    ctx.response.content_type = "application/json"
    ctx.response.print jobs.to_json
  end

  private def handle_retry_jobs(ctx)
    jobs = @redis.zrange("crystal:jobs:retry", 0, 49)
    ctx.response.content_type = "application/json"
    ctx.response.print jobs.to_json
  end

  private def handle_clear_failed(ctx)
    if ctx.request.method == "DELETE"
      @redis.del("crystal:jobs:failed")
      ctx.response.print({cleared: true}.to_json)
    else
      ctx.response.status_code = 405
    end
  end
end

# เปิด monitor ใน background
monitor = JobMonitor.new(Redis::Client.new)
spawn { monitor.start }
```

## Production Configuration

```yaml
# shard.yml
name: myapp_workers
version: 1.0.0

dependencies:
  redis:
    github: jgaskins/redis
  kemal:
    github: kemalcr/kemal
```

```crystal
# config/workers.cr

module Workers
  REDIS_URL    = ENV.fetch("REDIS_URL", "redis://localhost:6379")
  QUEUE_NAME   = ENV.fetch("QUEUE_NAME", "crystal:jobs:default")
  WORKER_COUNT = ENV.fetch("WORKER_COUNT", "4").to_i
  LOG_LEVEL    = ENV.fetch("LOG_LEVEL", "info")

  def self.setup
    queue = RedisJobQueue.instance

    # Register all job classes
    queue.register(EmailJob)
    queue.register(ImageProcessingJob)
    queue.register(GenerateReportJob)

    queue
  end
end
```

```dockerfile
# Dockerfile.worker
FROM crystallang/crystal:1.10-alpine AS builder
WORKDIR /app
COPY shard.yml shard.lock ./
RUN shards install --production
COPY . .
RUN crystal build --release src/worker.cr -o worker

FROM alpine:3.18
RUN apk add --no-cache libgcc libssl3 libcrypto3
COPY --from=builder /app/worker /app/worker
ENTRYPOINT ["/app/worker"]
```

## Integration กับ Web App

```crystal
# src/app.cr - Web app ที่ใช้ background jobs

require "kemal"
require "./jobs/*"
require "./workers/redis_queue"

# Setup
queue = Workers.setup

post "/users/register" do |env|
  data = env.params.json

  # สร้าง user
  user_id = create_user(data["email"].as_s, data["password"].as_s)

  # ส่งงานเข้า background queue
  EmailJob.perform_later(
    data["email"].as_s,
    "Welcome to Our App!",
    "Thank you for registering, #{data["name"].as_s}!"
  )

  ImageProcessingJob.perform_later(
    user_id,
    "/uploads/default_avatar.png",
    ["resize", "compress"]
  )

  env.response.status_code = 201
  {id: user_id, message: "Registration successful"}.to_json
end

post "/reports/generate" do |env|
  data = env.params.json
  user_id = env.session["user_id"].as_i

  # Queue report generation (อาจใช้เวลานาน)
  job_id = GenerateReportJob.perform_later(
    data["type"].as_s,
    user_id,
    data["date_range"].as_s
  )

  {job_id: job_id, message: "Report generation started"}.to_json
end

Kemal.run
```

## แบบฝึกหัด

1. สร้าง background job สำหรับส่ง SMS notifications
2. เพิ่มระบบ priority queue (high/medium/low priority)
3. สร้าง web dashboard แสดงสถานะ jobs แบบ real-time ด้วย WebSocket
4. เพิ่มระบบ dead letter queue และ manual retry
5. สร้าง rate limiting สำหรับ jobs (จำกัดจำนวน jobs ต่อชั่วโมง)

---

## สรุป Part 155

ในบทนี้เราได้เรียนรู้:

1. **ทำไมต้องใช้ Background Jobs**: แยกงานหนักออกจาก HTTP request
2. **In-Memory Queue**: ใช้ Crystal Channel สำหรับ simple jobs
3. **Redis Queue**: persistent queue พร้อม retry logic
4. **Job Classes**: abstract base class สำหรับ job definitions
5. **Job Scheduling**: cron-like scheduler สำหรับงานประจำ
6. **Worker Pool**: จัดการ workers หลายตัวพร้อมกัน
7. **Graceful Shutdown**: ปิดระบบอย่างสวยงามด้วย Signal handling
8. **Monitoring**: HTTP dashboard ดูสถานะ jobs
9. **Production Setup**: Docker, configuration, integration กับ web app

---

## ขั้นตอนต่อไป

ไปที่ [Part 156](part_156.md) เพื่อเรียนรู้:
- Crystal กับ GraphQL
- Building GraphQL APIs
- Schema definition
- Resolvers และ mutations
