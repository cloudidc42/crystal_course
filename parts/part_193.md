# Part 193: Logging Best Practices ใน Crystal

## บทนำ

Logging ที่ดีช่วย debug production issues ได้อย่างรวดเร็ว Crystal มี `Log` module ใน standard library ที่ flexible และ performant

## Log Module พื้นฐาน

```crystal
require "log"

# Log levels: TRACE < DEBUG < INFO < NOTICE < WARN < ERROR < FATAL
Log.info { "Application starting" }
Log.debug { "Config loaded: #{config.inspect}" }
Log.warn { "Deprecated feature used" }
Log.error { "Database connection failed" }
Log.fatal { "Cannot continue, shutting down" }

# Log ด้วย named logger (แยก component)
log = Log.for("MyApp::Database")
log.info { "Connected to database" }
log.warn { "Slow query detected: #{query_time}ms" }

# Logger hierarchy
app_log = Log.for("MyApp")
db_log = Log.for("MyApp::Database")    # child ของ MyApp
api_log = Log.for("MyApp::API")        # child ของ MyApp
auth_log = Log.for("MyApp::API::Auth") # child ของ API

# ตั้ง log level
Log.setup do |c|
  c.bind("*", :info, Log::IOBackend.new)
  c.bind("MyApp::Database", :debug, Log::IOBackend.new)  # Debug logs สำหรับ DB
end
```

## Log Levels

```crystal
require "log"

# ใช้ log levels อย่างถูกต้อง

# TRACE: very detailed, เปิดแค่ตอน deep debugging
Log.trace { "Entering function process_payment with #{args.inspect}" }

# DEBUG: ข้อมูลที่มีประโยชน์สำหรับ debugging
Log.debug { "Cache miss for key: #{cache_key}" }
Log.debug { "Query executed in #{query_time}ms" }

# INFO: normal operations ที่สำคัญ
Log.info { "User #{user.id} logged in" }
Log.info { "Order #{order.id} created" }
Log.info { "Server started on port #{port}" }

# NOTICE: important normal operations
Log.notice { "Database connection pool exhausted, waiting..." }
Log.notice { "Cache warming completed: #{cache.size} entries" }

# WARN: unexpected but recoverable situations
Log.warn { "Retry attempt #{retry_count}/3 for payment" }
Log.warn { "Request timeout, using cached data" }
Log.warn { "Deprecated API endpoint called" }

# ERROR: errors ที่ต้อง handle
Log.error { "Payment failed for order #{order.id}: #{error.message}" }
Log.error(exception: ex) { "Failed to process request" }

# FATAL: unrecoverable errors
Log.fatal { "Cannot connect to database after #{retries} attempts" }
```

## Structured Logging

```crystal
require "log"

# Structured logging: log เป็น key-value pairs สำหรับ parsing

# Custom backend ที่ output JSON
class JsonLogBackend < Log::Backend
  def write(entry : Log::Entry)
    output = {
      timestamp:  entry.timestamp.to_rfc3339,
      level:      entry.severity.to_s.downcase,
      source:     entry.source,
      message:    entry.message,
      context:    entry.context.metadata.to_h,
    }

    if data = entry.context.metadata.to_h
      output = output.merge(data)
    end

    if ex = entry.exception
      output = output.merge({
        error:       ex.class.name,
        error_msg:   ex.message,
      })
    end

    puts output.to_json
    $stdout.flush
  end
end

# Setup JSON logging
Log.setup do |c|
  backend = JsonLogBackend.new
  c.bind("*", :info, backend)
end

# ใช้ context สำหรับ structured data
Log.with_context(user_id: "123", request_id: "req-456") do
  Log.info { "Processing order" }
  Log.info { "Payment processed" }
  # ทุก log ในนี้จะมี user_id และ request_id
end

# Output:
# {"timestamp":"2025-01-01T12:00:00Z","level":"info","source":"","message":"Processing order","user_id":"123","request_id":"req-456"}
```

## Context Logging ใน Web Apps

```crystal
require "kemal"
require "log"

# Middleware สำหรับ request context logging
class RequestLogger
  include HTTP::Handler

  def call(context : HTTP::Server::Context)
    request_id = context.request.headers["X-Request-ID"]? || Random::Secure.hex(8)
    start_time = Time.monotonic

    # เพิ่ม request_id ใน response header
    context.response.headers["X-Request-ID"] = request_id

    Log.with_context(
      request_id: request_id,
      method: context.request.method,
      path: context.request.path,
      remote_ip: context.request.remote_address.try(&.to_s)
    ) do
      Log.info { "Request started" }

      begin
        call_next(context)
      rescue ex
        Log.error(exception: ex) { "Request failed with exception" }
        raise ex
      end

      elapsed_ms = (Time.monotonic - start_time).total_milliseconds
      status = context.response.status_code

      if elapsed_ms > 1000
        Log.warn { "Slow request: #{elapsed_ms.round(0)}ms status=#{status}" }
      else
        Log.info { "Request completed status=#{status} duration_ms=#{elapsed_ms.round(2)}" }
      end
    end
  end
end
```

## Log Rotation

```crystal
require "log"

# File backend ด้วย rotation
class RotatingFileBackend < Log::Backend
  def initialize(
    @path : String,
    @max_size : Int64 = 100 * 1024 * 1024,  # 100MB
    @max_files : Int32 = 10
  )
    @current_file = open_file
    @current_size = @current_file.size
  end

  def write(entry : Log::Entry)
    line = format(entry)
    size = line.bytesize.to_i64

    if @current_size + size > @max_size
      rotate
    end

    @current_file.puts(line)
    @current_file.flush
    @current_size += size
  end

  private def format(entry : Log::Entry) : String
    "[#{entry.timestamp.to_rfc3339}] #{entry.severity.to_s.upcase.ljust(7)} #{entry.source}: #{entry.message}"
  end

  private def rotate
    @current_file.close

    # Rename existing files
    (@max_files - 1).downto(1) do |i|
      old = "#{@path}.#{i}"
      new = "#{@path}.#{i + 1}"
      File.rename(old, new) if File.exists?(old)
    end

    File.rename(@path, "#{@path}.1") if File.exists?(@path)

    @current_file = open_file
    @current_size = 0_i64
  end

  private def open_file : File
    File.open(@path, "a")
  end
end

# Setup rotating file logging
Log.setup do |c|
  c.bind("*", :info, RotatingFileBackend.new(
    path: "/var/log/myapp/app.log",
    max_size: 50 * 1024 * 1024,  # 50MB per file
    max_files: 5                   # Keep 5 files
  ))
end
```

## Log Configuration

```crystal
require "log"

# Configure logging จาก environment
def configure_logging
  level = case ENV["LOG_LEVEL"]?.try(&.upcase)
  when "TRACE"  then Log::Severity::Trace
  when "DEBUG"  then Log::Severity::Debug
  when "INFO"   then Log::Severity::Info
  when "WARN"   then Log::Severity::Warn
  when "ERROR"  then Log::Severity::Error
  when "FATAL"  then Log::Severity::Fatal
  else               Log::Severity::Info
  end

  format = ENV["LOG_FORMAT"]? == "json" ? :json : :text

  Log.setup do |c|
    backend = if format == :json
      JsonLogBackend.new
    else
      Log::IOBackend.new
    end

    c.bind("*", level, backend)

    # Debug สำหรับ specific modules ถ้าต้องการ
    if ENV["LOG_DB_DEBUG"]?
      c.bind("MyApp::Database", :debug, backend)
    end
  end

  Log.info { "Logging configured: level=#{level} format=#{format}" }
end

configure_logging
```

## แบบฝึกหัด

1. สร้าง JSON logging backend ที่ compatible กับ Elasticsearch/Datadog format
2. Implement request ID propagation ผ่าน log context ใน web app
3. สร้าง rotating file logger ที่ compress old log files ด้วย gzip
4. เขียน log analyzer script ที่ parse JSON logs และ output statistics

## สรุป

Logging Best Practices ใน Crystal:
- **Log module**: built-in, hierarchical loggers
- **Log levels**: TRACE/DEBUG/INFO/NOTICE/WARN/ERROR/FATAL
- **Structured logging**: JSON output สำหรับ log aggregation
- **Log.with_context**: thread-local context ที่ ติดมากับทุก log
- **Request ID**: propagate ผ่าน HTTP headers สำหรับ request tracing
- **Rotating files**: ไม่ให้ disk เต็ม, keep history
- **Level configuration**: ตั้งจาก environment variables
- **Performance**: lazy evaluation ด้วย block syntax `Log.debug { expensive }`
