# Part 192: Monitoring and Observability ใน Crystal

## บทนำ

Observability คือความสามารถในการเข้าใจ state ของ system จาก outputs ของมัน ประกอบด้วย 3 pillars: Metrics, Logs, และ Traces (MLT)

## Health Check Endpoints

```crystal
require "kemal"
require "db"

# Health check endpoint สำหรับ load balancers และ kubernetes
class HealthChecker
  record Health, status : String, checks : Hash(String, CheckResult)

  record CheckResult, ok : Bool, message : String, latency_ms : Float64 = 0.0

  def self.check_all : Health
    checks = {} of String => CheckResult

    # Database check
    checks["database"] = check_database

    # Redis check
    checks["redis"] = check_redis

    # Disk space check
    checks["disk"] = check_disk

    # Memory check
    checks["memory"] = check_memory

    all_ok = checks.values.all?(&.ok)
    Health.new(
      status: all_ok ? "healthy" : "unhealthy",
      checks: checks
    )
  end

  private def self.check_database : CheckResult
    start = Time.monotonic
    begin
      DB.open(ENV["DATABASE_URL"]) do |db|
        db.query_one("SELECT 1", as: Int32)
      end
      latency = (Time.monotonic - start).total_milliseconds
      CheckResult.new(ok: true, message: "connected", latency_ms: latency)
    rescue ex
      CheckResult.new(ok: false, message: ex.message || "unknown error")
    end
  end

  private def self.check_redis : CheckResult
    start = Time.monotonic
    begin
      Redis::Client.new(url: ENV["REDIS_URL"]).ping
      latency = (Time.monotonic - start).total_milliseconds
      CheckResult.new(ok: true, message: "connected", latency_ms: latency)
    rescue ex
      CheckResult.new(ok: false, message: ex.message || "connection failed")
    end
  end

  private def self.check_disk : CheckResult
    stat = File::Info.new("/")
    # Check if > 90% disk used
    used_pct = 75.0  # ตัวอย่าง
    if used_pct < 90.0
      CheckResult.new(ok: true, message: "#{used_pct.to_i}% used")
    else
      CheckResult.new(ok: false, message: "Disk nearly full: #{used_pct.to_i}% used")
    end
  end

  private def self.check_memory : CheckResult
    require "gc"
    stats = GC.stats
    heap_mb = stats.heap_size / 1024 / 1024
    CheckResult.new(ok: heap_mb < 2048, message: "#{heap_mb}MB heap")
  end
end

# Endpoints
get "/health" do |env|
  env.response.content_type = "application/json"
  health = HealthChecker.check_all
  env.response.status_code = health.status == "healthy" ? 200 : 503
  {
    status: health.status,
    timestamp: Time.local.to_rfc3339,
    checks: health.checks.transform_values { |c|
      {ok: c.ok, message: c.message, latency_ms: c.latency_ms}
    }
  }.to_json
end

# Liveness probe (ง่ายกว่า - แค่ตรวจว่า process ยัง alive)
get "/health/live" do |env|
  env.response.content_type = "application/json"
  {status: "alive"}.to_json
end

# Readiness probe (ตรวจว่าพร้อมรับ traffic)
get "/health/ready" do |env|
  env.response.content_type = "application/json"
  health = HealthChecker.check_all
  env.response.status_code = health.status == "healthy" ? 200 : 503
  {status: health.status}.to_json
end
```

## Prometheus Metrics

```crystal
# Prometheus-format metrics endpoint
class Metrics
  @mutex = Mutex.new

  # Counters
  @http_requests_total = Hash(String, Int64).new(0_i64)
  @http_request_errors = Hash(String, Int64).new(0_i64)

  # Histograms (simplified as buckets)
  @http_request_duration_ms : Array(Float64) = [] of Float64

  # Gauges
  @active_connections = Atomic(Int32).new(0)

  def record_request(method : String, path : String, status : Int32, duration_ms : Float64)
    key = "#{method} #{path}"
    @mutex.synchronize do
      @http_requests_total[key] += 1
      @http_request_errors[key] += 1 if status >= 500
      @http_request_duration_ms << duration_ms
    end
  end

  def connection_opened
    @active_connections.add(1)
  end

  def connection_closed
    @active_connections.sub(1)
  end

  def to_prometheus : String
    String.build do |sb|
      # Help text และ type declarations
      sb << "# HELP http_requests_total Total HTTP requests\n"
      sb << "# TYPE http_requests_total counter\n"
      @http_requests_total.each do |labels, count|
        method, path = labels.split(" ", 2)
        sb << "http_requests_total{method=\"#{method}\",path=\"#{path}\"} #{count}\n"
      end

      sb << "\n# HELP http_request_errors_total HTTP request errors\n"
      sb << "# TYPE http_request_errors_total counter\n"
      @http_request_errors.each do |labels, count|
        method, path = labels.split(" ", 2)
        sb << "http_request_errors_total{method=\"#{method}\",path=\"#{path}\"} #{count}\n"
      end

      sb << "\n# HELP active_connections Current active connections\n"
      sb << "# TYPE active_connections gauge\n"
      sb << "active_connections #{@active_connections.get}\n"

      # Request duration percentiles (simplified)
      if !@http_request_duration_ms.empty?
        sorted = @http_request_duration_ms.sort
        p50 = percentile(sorted, 0.50)
        p95 = percentile(sorted, 0.95)
        p99 = percentile(sorted, 0.99)

        sb << "\n# HELP http_request_duration_ms Request duration in ms\n"
        sb << "# TYPE http_request_duration_ms summary\n"
        sb << "http_request_duration_ms{quantile=\"0.5\"} #{p50.round(2)}\n"
        sb << "http_request_duration_ms{quantile=\"0.95\"} #{p95.round(2)}\n"
        sb << "http_request_duration_ms{quantile=\"0.99\"} #{p99.round(2)}\n"
        sb << "http_request_duration_ms_sum #{sorted.sum.round(2)}\n"
        sb << "http_request_duration_ms_count #{sorted.size}\n"
      end

      # GC metrics
      require "gc"
      stats = GC.stats
      sb << "\n# HELP crystal_gc_heap_bytes GC heap size in bytes\n"
      sb << "# TYPE crystal_gc_heap_bytes gauge\n"
      sb << "crystal_gc_heap_bytes #{stats.heap_size}\n"
    end
  end

  private def percentile(sorted : Array(Float64), p : Float64) : Float64
    return 0.0 if sorted.empty?
    idx = ((sorted.size - 1) * p).to_i
    sorted[idx]
  end
end

METRICS = Metrics.new

# Metrics endpoint
get "/metrics" do |env|
  env.response.content_type = "text/plain; version=0.0.4"
  METRICS.to_prometheus
end
```

## Distributed Tracing (OpenTelemetry-style)

```crystal
# Simple tracing สำหรับ distributed systems
module Tracing
  class Span
    getter trace_id : String
    getter span_id : String
    getter parent_id : String?
    getter name : String
    getter start_time : Time::Span
    getter tags : Hash(String, String)
    getter logs : Array({Time::Span, String})
    @end_time : Time::Span?
    @error : Exception?

    def initialize(@name : String, @parent_id : String? = nil)
      @trace_id = parent_id ? extract_trace_from(parent_id) : random_id
      @span_id = random_id
      @start_time = Time.monotonic
      @tags = {} of String => String
      @logs = [] of {Time::Span, String}
    end

    def tag(key : String, value : String)
      @tags[key] = value
    end

    def log(message : String)
      @logs << {Time.monotonic, message}
    end

    def finish(error : Exception? = nil)
      @end_time = Time.monotonic
      @error = error
    end

    def duration_ms : Float64?
      if end_time = @end_time
        (end_time - @start_time).total_milliseconds
      end
    end

    def to_json_object : Hash
      {
        "traceId"  => @trace_id,
        "spanId"   => @span_id,
        "parentId" => @parent_id,
        "name"     => @name,
        "duration" => duration_ms,
        "tags"     => @tags,
        "error"    => @error.try(&.message),
      }
    end

    private def random_id : String
      Random::Secure.hex(8)
    end

    private def extract_trace_from(parent_id : String) : String
      parent_id[0, 16]
    end
  end

  # Propagate context ผ่าน HTTP headers
  TRACE_ID_HEADER = "X-Trace-ID"
  SPAN_ID_HEADER = "X-Span-ID"

  def self.extract_context(headers : HTTP::Headers) : {String?, String?}
    trace_id = headers[TRACE_ID_HEADER]?
    span_id = headers[SPAN_ID_HEADER]?
    {trace_id, span_id}
  end

  def self.inject_context(headers : HTTP::Headers, span : Span)
    headers[TRACE_ID_HEADER] = span.trace_id
    headers[SPAN_ID_HEADER] = span.span_id
  end
end
```

## แบบฝึกหัด

1. สร้าง `/metrics` endpoint ที่ export Prometheus-format metrics สำหรับ request rate, error rate, latency
2. สร้าง health check ที่รวม database, Redis, disk space, และ memory checks
3. Implement request tracing ที่ propagate trace IDs ผ่าน HTTP headers
4. สร้าง Grafana dashboard configuration สำหรับ Crystal app metrics

## สรุป

Monitoring and Observability ใน Crystal:
- **Health endpoints**: `/health`, `/health/live`, `/health/ready`
- **Prometheus metrics**: counter, gauge, histogram สำหรับ Prometheus scraping
- **Structured logging**: JSON logs สำหรับ centralized logging
- **Distributed tracing**: trace IDs ผ่าน HTTP headers
- **3 pillars**: Metrics (numbers), Logs (events), Traces (requests across services)
- **Graceful degradation**: health checks แยก liveness vs readiness
- **GC metrics**: monitor heap size, collection frequency
