# Part 79: Signals

## บทนำ

Signals เป็น notifications ที่ส่งจาก OS ไปยัง process เพื่อแจ้งเหตุการณ์ต่างๆ เช่น Ctrl+C (SIGINT), kill command (SIGTERM), หรือ system events อื่นๆ Crystal มีระบบจัดการ signals ที่ใช้งานง่าย

---

## 1. Signal Types

```crystal
# Signals ที่ใช้บ่อย
# Signal::INT  - Ctrl+C (SIGINT, 2)
# Signal::TERM - kill (SIGTERM, 15) - terminate gracefully
# Signal::HUP  - hangup (SIGHUP, 1) - reload config
# Signal::KILL - kill -9 (SIGKILL, 9) - cannot be caught!
# Signal::QUIT - quit with core dump (SIGQUIT, 3)
# Signal::USR1 - user-defined signal 1 (SIGUSR1, 10)
# Signal::USR2 - user-defined signal 2 (SIGUSR2, 12)
# Signal::PIPE - broken pipe (SIGPIPE, 13)
# Signal::ALRM - alarm (SIGALRM, 14)
# Signal::CHLD - child process changed (SIGCHLD, 17)
# Signal::WINCH - terminal resize (SIGWINCH, 28)
# Signal::ZERO - signal 0 (ไม่ส่งจริง, ใช้ตรวจสอบ process)

puts Signal::INT.value   # => 2
puts Signal::TERM.value  # => 15
puts Signal::HUP.value   # => 1

# ส่ง signal ไปยัง process
# Process.signal(Signal::TERM, pid)
```

---

## 2. Signal::INT (Ctrl+C)

```crystal
# Handle Ctrl+C อย่าง graceful
running = true

Signal::INT.trap do
  puts "\nInterrupted! Shutting down..."
  running = false
end

puts "Running... press Ctrl+C to stop"
while running
  sleep 0.1
end
puts "Done!"

# ตัวอย่างจริง: long-running process
Signal::INT.trap do
  STDERR.puts "\nReceived SIGINT, cleaning up..."
  # cleanup code here
  exit 0  # exit cleanly
end

# ป้องกัน Ctrl+C ระหว่าง critical section
def critical_section(&block)
  # Block INT during critical work
  old_handler = Signal::INT.trap { puts "Waiting for critical section to finish..." }
  begin
    block.call
  ensure
    Signal::INT.trap { old_handler.call(Signal::INT) }
  end
end
```

---

## 3. Signal::TERM

```crystal
# SIGTERM - default terminate signal จาก kill command
# ควร handle เพื่อ shutdown gracefully

Signal::TERM.trap do
  puts "Received SIGTERM, shutting down gracefully..."
  # cleanup
  exit 0
end

# SIGTERM กับ SIGINT บ่อยครั้งทำงานเหมือนกัน
shutdown_handler = Proc(Signal, Nil).new do |sig|
  puts "Received #{sig}, shutting down..."
  exit 0
end

Signal::INT.trap  { shutdown_handler.call(Signal::INT) }
Signal::TERM.trap { shutdown_handler.call(Signal::TERM) }

# Server shutdown example
class Server
  def initialize(@port : Int32)
    @running = false
    @connections = [] of String  # simulate connections
    
    setup_signals
  end
  
  private def setup_signals
    Signal::INT.trap  { shutdown }
    Signal::TERM.trap { shutdown }
  end
  
  def start
    @running = true
    puts "Server started on port #{@port}"
    
    while @running
      # Accept connections...
      sleep 0.1
    end
    
    cleanup
  end
  
  private def shutdown
    puts "\nShutdown signal received"
    @running = false
  end
  
  private def cleanup
    puts "Closing #{@connections.size} connections..."
    @connections.clear
    puts "Server stopped"
  end
end
```

---

## 4. Signal::HUP (Reload Configuration)

```crystal
# SIGHUP - hangup, ปกติใช้สำหรับ reload configuration

config = {
  "log_level" => "info",
  "max_connections" => "100",
}

Signal::HUP.trap do
  puts "Reloading configuration..."
  
  # โหลด config ใหม่
  if File.exists?("config.env")
    File.each_line("config.env") do |line|
      next if line.strip.starts_with?("#") || line.strip.empty?
      if line =~ /^(\w+)=(.+)$/
        config[$~[1]] = $~[2].strip
      end
    end
  end
  
  puts "Configuration reloaded: #{config.inspect}"
end

# ส่ง HUP จาก terminal:
# kill -HUP <pid>
# หรือ
# kill -1 <pid>

# Daemon ที่ reload config
class Daemon
  def initialize
    @config = load_config
    
    Signal::HUP.trap do
      reload_config
    end
  end
  
  private def load_config : Hash(String, String)
    config = {} of String => String
    if File.exists?("daemon.conf")
      File.each_line("daemon.conf") do |line|
        next if line.strip.starts_with?("#")
        if line =~ /^(\w+)\s*=\s*(.+)$/
          config[$~[1]] = $~[2].strip
        end
      end
    end
    config
  end
  
  private def reload_config
    new_config = load_config
    @config = new_config
    puts "Config reloaded at #{Time.local}"
  end
  
  def run
    loop do
      # use @config
      sleep 1
    end
  end
end
```

---

## 5. Setting Up Signal Handlers

```crystal
# Signal handler พื้นฐาน
Signal::INT.trap do
  puts "Got INT"
end

# Handler ที่ receive signal info
Signal::USR1.trap do |sig|
  puts "Received signal: #{sig}"
end

# Reset to default handler
Signal::INT.reset

# Ignore signal
Signal::PIPE.ignore  # ignore broken pipe

# ตรวจสอบ current handler
# (Crystal ไม่มี built-in สำหรับ query handler)

# Handler chain
handlers = [] of Signal ->

def add_signal_handler(signal : Signal, &handler : Signal ->)
  existing = nil
  signal.trap do |sig|
    existing.try { |h| h.call(sig) }
    handler.call(sig)
  end
end

# ตัวอย่าง: stack multiple handlers
add_signal_handler(Signal::INT) { |_| puts "Handler 1" }
add_signal_handler(Signal::INT) { |_| puts "Handler 2" }
add_signal_handler(Signal::INT) { |_| exit 0 }

# Signal ใน Fiber context
spawn do
  Signal::INT.trap do
    puts "Fiber caught INT"
  end
end

Fiber.yield
```

---

## 6. Graceful Shutdown Pattern

```crystal
# Complete graceful shutdown pattern
class GracefulApp
  @shutdown_requested = false
  @shutdown_mutex = Mutex.new
  
  def initialize
    setup_signals
  end
  
  private def setup_signals
    Signal::INT.trap  { request_shutdown("SIGINT") }
    Signal::TERM.trap { request_shutdown("SIGTERM") }
    Signal::HUP.trap  { reload_config }
    Signal::PIPE.ignore  # ignore broken pipes
  end
  
  private def request_shutdown(signal : String)
    @shutdown_mutex.synchronize do
      return if @shutdown_requested
      @shutdown_requested = true
      STDERR.puts "\n#{signal} received, initiating shutdown..."
    end
  end
  
  private def reload_config
    STDERR.puts "Reloading configuration..."
    # reload logic
  end
  
  def shutdown_requested? : Bool
    @shutdown_mutex.synchronize { @shutdown_requested }
  end
  
  def run
    puts "Application started (PID: #{Process.pid})"
    
    # Main loop
    until shutdown_requested?
      work
    end
    
    # Cleanup
    cleanup
    puts "Application shut down cleanly"
  end
  
  private def work
    # Do application work
    sleep 0.1
  end
  
  private def cleanup
    puts "Performing cleanup..."
    # Close connections, flush buffers, etc.
    sleep 0.5  # simulate cleanup
  end
end

app = GracefulApp.new
app.run
```

---

## 7. Signal กับ Process Management

```crystal
# ส่ง signal ไปยัง child process
child = Process.new("sleep", args: ["60"])
puts "Child PID: #{child.pid}"

# ส่ง SIGTERM เพื่อ terminate gracefully
sleep 1
child.terminate  # sends SIGTERM

# รอให้ child จบ
status = child.wait
puts "Child exited with: #{status.exit_code}"

# ส่ง SIGKILL เพื่อ force kill
child2 = Process.new("sleep", args: ["60"])
sleep 0.5
child2.kill  # sends SIGKILL
child2.wait

# Signal ไปยัง process group
# Process.signal(Signal::TERM, -pgid)  # ส่งถึง process group

# ตรวจสอบว่า process ยังอยู่
def process_alive?(pid : Int32) : Bool
  begin
    Process.signal(Signal::ZERO, pid)
    true
  rescue Errno
    false
  end
end

# watchdog สำหรับ child process
class Watchdog
  def initialize(@command : String, @args : Array(String) = [] of String)
    @process = nil.as(Process?)
    @running = false
  end
  
  def start
    @running = true
    
    spawn do
      while @running
        ensure_running
        sleep 1
      end
    end
  end
  
  def stop
    @running = false
    @process.try(&.terminate)
    @process.try(&.wait)
  end
  
  private def ensure_running
    process = @process
    if process.nil? || process.terminated?
      puts "Starting/restarting #{@command}..."
      @process = Process.new(@command, args: @args)
    end
  end
end
```

---

## 8. Signal ใน Real Applications

```crystal
# Web server graceful shutdown
class WebServer
  def initialize(@port : Int32)
    @active_requests = Atomic(Int32).new(0)
    @shutdown = Atomic(Bool).new(false)
  end
  
  def start
    puts "Starting server on port #{@port}..."
    
    setup_signals
    
    # Listen for connections
    while !@shutdown.get
      # simulate accepting connection
      spawn handle_request
      sleep 0.01
    end
    
    # Wait for active requests to finish
    wait_for_active_requests
    
    puts "Server stopped"
  end
  
  private def setup_signals
    Signal::INT.trap do
      puts "\nShutting down (waiting for #{@active_requests.get} requests)..."
      @shutdown.set(true)
    end
    
    Signal::TERM.trap do
      puts "\nForce shutdown requested"
      @shutdown.set(true)
    end
  end
  
  private def handle_request
    @active_requests.add(1)
    begin
      sleep rand(0.1..0.5)  # simulate request processing
    ensure
      @active_requests.sub(1)
    end
  end
  
  private def wait_for_active_requests
    timeout = 30  # seconds
    start = Time.monotonic
    
    while @active_requests.get > 0
      remaining = timeout - (Time.monotonic - start).total_seconds
      break if remaining <= 0
      puts "Waiting for #{@active_requests.get} requests (#{remaining.round.to_i}s remaining)..."
      sleep 1
    end
    
    if @active_requests.get > 0
      puts "Timeout! Forcing shutdown with #{@active_requests.get} active requests"
    end
  end
end

# Worker process ที่ reload config เมื่อได้รับ SIGHUP
class Worker
  def initialize
    @config = load_default_config
    @processed = 0
    
    Signal::HUP.trap { reload }
    Signal::TERM.trap { stop }
    Signal::INT.trap { stop }
    Signal::USR1.trap { print_stats }
  end
  
  private def load_default_config : Hash(String, Int32)
    {"batch_size" => 100, "sleep_ms" => 10}
  end
  
  private def reload
    puts "[#{Time.local}] Reloading configuration"
    @config = load_default_config  # reload from file in real app
  end
  
  private def stop
    puts "[#{Time.local}] Stopping (processed #{@processed} items)"
    exit 0
  end
  
  private def print_stats
    puts "[#{Time.local}] Stats: processed=#{@processed}"
  end
  
  def run
    loop do
      @processed += 1
      sleep(@config["sleep_ms"].milliseconds)
    end
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน daemon process ที่รัน background task, รองรับ SIGHUP เพื่อ reload config และ SIGTERM เพื่อ graceful shutdown

### แบบฝึกหัดที่ 2
เขียน watchdog ที่ monitor child processes และ restart เมื่อ crash

### เฉลย

```crystal
# แบบฝึกหัดที่ 1: Simple Daemon
class SimpleDaemon
  def initialize(@config_path : String)
    @config = {} of String => String
    @running = false
    @job_count = 0
    
    load_config
    setup_signals
  end
  
  private def load_config
    return unless File.exists?(@config_path)
    File.each_line(@config_path) do |line|
      next if line.strip.starts_with?("#") || line.empty?
      if line =~ /^(\w+)=(.+)$/
        @config[$~[1]] = $~[2].strip
      end
    end
    puts "Config loaded: #{@config.inspect}"
  end
  
  private def setup_signals
    Signal::HUP.trap do
      puts "[#{Time.local}] Reloading config..."
      load_config
    end
    
    Signal::TERM.trap do
      puts "[#{Time.local}] SIGTERM received, stopping after current job..."
      @running = false
    end
    
    Signal::INT.trap do
      puts "\n[#{Time.local}] SIGINT received, stopping..."
      @running = false
    end
    
    Signal::USR1.trap do
      puts "[#{Time.local}] Jobs processed: #{@job_count}"
    end
  end
  
  def run
    @running = true
    puts "Daemon started (PID: #{Process.pid})"
    
    while @running
      # Simulate job processing
      @job_count += 1
      interval = @config["interval"]?.try(&.to_f) || 1.0
      puts "[#{Time.local}] Processed job ##{@job_count}" if @job_count % 10 == 0
      sleep interval.seconds
    end
    
    puts "Daemon stopped (#{@job_count} jobs processed)"
  end
end

# สร้าง config file
File.write("/tmp/daemon.conf", "interval=0.1\nmax_jobs=1000\n")

daemon = SimpleDaemon.new("/tmp/daemon.conf")
# ใช้ spawn หรือ Thread ถ้าต้องการ non-blocking
# daemon.run

File.delete("/tmp/daemon.conf")
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Signal types** - INT, TERM, HUP, USR1, USR2 และอื่นๆ
2. **Signal::INT** - handle Ctrl+C
3. **Signal::TERM** - graceful shutdown
4. **Signal::HUP** - reload configuration
5. **Setting up handlers** - trap, reset, ignore
6. **Graceful shutdown** - pattern สำหรับ clean exit
7. **Signal กับ Process** - terminate, kill, wait
8. **Real applications** - web server, daemon, worker

Signal handling เป็นส่วนสำคัญของ production-quality applications ที่ต้องการ graceful shutdown และ dynamic reconfiguration
