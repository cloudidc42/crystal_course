# Part 78: Process และ System Calls

## บทนำ

Crystal มี `Process` class สำหรับจัดการ child processes ไม่ว่าจะเป็นการ run commands, capture output, หรือ exec ไปยัง process อื่น

---

## 1. Process.run

```crystal
# Process.run รัน command และรอให้เสร็จ
result = Process.run("ls", args: ["-la"])
puts result.exit_code   # => 0 (success)
puts result.success?    # => true

# รัน command พร้อม arguments
result = Process.run("echo", args: ["hello", "world"])
puts result.success?

# ตรวจสอบ exit code
result = Process.run("false")  # command ที่ return 1
puts result.exit_code   # => 1
puts result.success?    # => false

# รัน shell command
result = Process.run("bash", args: ["-c", "echo hello && echo world"])
puts result.success?

# หา executable ก่อน run
if Process.find_executable("git")
  result = Process.run("git", args: ["--version"])
  puts result.success?
end

# Environment variables สำหรับ child process
env = {"MY_VAR" => "hello", "DEBUG" => "1"}
result = Process.run("printenv", args: ["MY_VAR"], env: env)
puts result.success?
```

---

## 2. Process.exec

```crystal
# Process.exec แทนที่ current process ด้วย process ใหม่
# ไม่ return กลับมา!

# ใช้เมื่อต้องการ exec into another program
# เช่น เปลี่ยน process เป็น editor
if ARGV.includes?("--edit")
  editor = ENV["EDITOR"]? || "nano"
  file = ARGV.last
  Process.exec(editor, args: [file])  # ไม่ return
end

# เปลี่ยนเป็น subcommand
case ARGV.first?
when "git"
  Process.exec("git", args: ARGV[1..])
when "npm"
  Process.exec("npm", args: ARGV[1..])
end

# ตัวอย่าง: wrapper script
def run_with_setup(command : String, args : Array(String) = [] of String)
  puts "Setting up environment..."
  
  # Setup environment
  env = ENV.to_h.merge({
    "MY_APP_ROOT" => Dir.current,
    "PATH" => "#{Dir.current}/bin:#{ENV["PATH"]}",
  })
  
  puts "Exec: #{command} #{args.join(" ")}"
  Process.exec(command, args: args, env: env)
end
```

---

## 3. Process::Status

```crystal
# Process::Status มีข้อมูลเกี่ยวกับ exit status
result = Process.run("ls")
status = result

puts status.exit_code    # exit code (0-255)
puts status.success?     # true ถ้า exit code == 0
puts status.normal_exit? # true ถ้า exit normally (ไม่ใช่ signal)
puts status.signal_exit? # true ถ้า exit จาก signal

# ตรวจสอบ exit status
def run_or_die(command : String, args : Array(String) = [] of String)
  result = Process.run(command, args: args)
  unless result.success?
    raise "Command failed: #{command} #{args.join(" ")} (exit #{result.exit_code})"
  end
  result
end

begin
  run_or_die("ls", ["-la"])
  run_or_die("nonexistent_command")
rescue ex
  puts "Error: #{ex.message}"
end

# Exit codes convention
# 0 = success
# 1 = general error
# 2 = misuse of command
# 127 = command not found
# 130 = interrupted by Ctrl+C (SIGINT)
```

---

## 4. Capturing stdout/stderr

```crystal
# capture stdout
output = IO::Memory.new
Process.run("ls", output: output)
puts output.to_s

# capture stderr
error_output = IO::Memory.new
Process.run("ls", args: ["nonexistent"], error: error_output)
puts "Error: #{error_output.to_s}"

# capture ทั้ง stdout และ stderr
stdout = IO::Memory.new
stderr = IO::Memory.new

result = Process.run("command_that_may_fail",
                     output: stdout,
                     error: stderr)

if result.success?
  puts stdout.to_s
else
  STDERR.puts stderr.to_s
end

# ส่ง stdin ไปยัง process
stdin_data = IO::Memory.new("line1\nline2\nline3")
output = IO::Memory.new
Process.run("sort", input: stdin_data, output: output)
puts output.to_s

# Shell command ด้วย capture
def shell(cmd : String) : {Int32, String, String}
  stdout = IO::Memory.new
  stderr = IO::Memory.new
  
  result = Process.run(
    "bash",
    args: ["-c", cmd],
    output: stdout,
    error: stderr
  )
  
  {result.exit_code, stdout.to_s, stderr.to_s}
end

code, out, err = shell("echo hello && echo world")
puts "Exit: #{code}"
puts "Stdout: #{out}"
puts "Stderr: #{err}"

code, out, err = shell("ls nonexistent_file")
puts "Exit: #{code}"
puts "Error: #{err}"
```

---

## 5. Process.pid

```crystal
# Process.pid - PID ของ current process
puts Process.pid

# PID ของ child process (ระหว่าง running)
process = Process.new("sleep", args: ["10"])
puts "Child PID: #{process.pid}"
process.terminate
process.wait

# ตรวจสอบว่า process กำลัง run
def process_running?(pid : Int32) : Bool
  # Send signal 0 to check existence
  begin
    Process.signal(Signal::ZERO, pid)
    true
  rescue
    false
  end
end

# Lock file สำหรับ single instance
class PidFile
  def initialize(@path : String)
  end
  
  def acquire : Bool
    if File.exists?(@path)
      existing_pid = File.read(@path).strip.to_i?
      if existing_pid && process_running?(existing_pid)
        puts "Process already running (PID: #{existing_pid})"
        return false
      end
    end
    
    File.write(@path, Process.pid.to_s)
    true
  end
  
  def release
    File.delete(@path) if File.exists?(@path)
  end
  
  private def process_running?(pid : Int32) : Bool
    File.exists?("/proc/#{pid}")  # Linux
  end
end

pid_file = PidFile.new("/tmp/myapp.pid")
if pid_file.acquire
  at_exit { pid_file.release }
  puts "Running as PID #{Process.pid}"
  # do work...
end
```

---

## 6. Process.exit

```crystal
# Process.exit - ออกจาก program
Process.exit(0)    # success
Process.exit(1)    # error
Process.exit       # exit(0)

# ใช้ exit ใน error handling
def validate_input(input : String)
  if input.empty?
    STDERR.puts "Error: Input cannot be empty"
    Process.exit(1)
  end
end

# at_exit hooks - รันก่อน exit
at_exit do
  puts "Cleaning up..."
  # cleanup resources
end

at_exit do
  puts "Saving state..."
end

# at_exit รันในลำดับ LIFO (last in, first out)

# Graceful shutdown
class Application
  @@instance = nil.as(Application?)
  
  def self.instance : Application
    @@instance ||= new
  end
  
  def initialize
    @running = false
    
    at_exit do
      stop if @running
    end
  end
  
  def start
    @running = true
    puts "Application started"
    
    # Setup signal handlers
    Signal::INT.trap { stop; exit 0 }
    Signal::TERM.trap { stop; exit 0 }
  end
  
  def stop
    return unless @running
    @running = false
    puts "Application stopping..."
    # cleanup
    puts "Application stopped"
  end
  
  def run
    start
    while @running
      sleep 0.1
    end
  end
end
```

---

## 7. Running Shell Commands อย่างปลอดภัย

```crystal
# ระวัง: อย่าส่ง user input ไปยัง shell โดยตรง
# เสี่ยงต่อ command injection

# BAD: ไม่ปลอดภัย
user_input = "file.txt; rm -rf /"
# system("cat #{user_input}")  # DANGEROUS!

# GOOD: ส่ง args แยก ไม่ผ่าน shell
Process.run("cat", args: [user_input])  # args เป็น separate strings

# Sanitize filenames
def safe_filename?(name : String) : Bool
  name.match?(/\A[\w\-. ]+\Z/) && !name.includes?("..")
end

def safe_path?(path : String, base : String) : Bool
  expanded = File.expand_path(File.join(base, path))
  expanded.starts_with?(File.expand_path(base))
end

# Helper สำหรับ run commands อย่างปลอดภัย
def run_command(command : String, *args : String,
                env : Hash(String, String) = {} of String => String,
                cwd : String? = nil,
                timeout : Float64? = nil) : {Int32, String, String}
  stdout_io = IO::Memory.new
  stderr_io = IO::Memory.new
  
  process_options = {
    output: stdout_io,
    error: stderr_io,
    env: env.empty? ? nil : env,
    chdir: cwd,
  }
  
  result = Process.run(command, args: args.to_a, **process_options)
  
  {result.exit_code, stdout_io.to_s, stderr_io.to_s}
end

# Git helper
module Git
  def self.run(*args : String, cwd : String = Dir.current) : {Bool, String}
    code, out, err = run_command("git", *args, cwd: cwd)
    {code == 0, code == 0 ? out : err}
  end
  
  def self.status(cwd : String = Dir.current) : String
    _, output = run("status", "--porcelain", cwd: cwd)
    output
  end
  
  def self.current_branch(cwd : String = Dir.current) : String?
    success, output = run("branch", "--show-current", cwd: cwd)
    success ? output.strip : nil
  end
  
  def self.commit(message : String, cwd : String = Dir.current) : Bool
    run("add", "-A", cwd: cwd)[0] &&
    run("commit", "-m", message, cwd: cwd)[0]
  end
end

# ตัวอย่าง
if Git.run("--version")[0]
  puts "Git available"
  if branch = Git.current_branch
    puts "Current branch: #{branch}"
  end
end
```

---

## 8. Process Management

```crystal
# สร้าง long-running process
process = Process.new("tail", args: ["-f", "/var/log/syslog"])
puts "Started tail with PID #{process.pid}"

# terminate gracefully
process.terminate  # SIGTERM
sleep 1

# force kill if still running
process.kill  # SIGKILL

# wait ให้ process เสร็จ
status = process.wait
puts "Process exited with #{status.exit_code}"

# Non-blocking check
process = Process.new("sleep", args: ["5"])
3.times do
  sleep 1
  puts "Still running..." # process ยังไม่เสร็จ
end
process.terminate
process.wait

# Process pool สำหรับ parallel jobs
class ProcessPool
  def initialize(@max_workers : Int32 = 4)
    @processes = [] of Process
  end
  
  def run(command : String, args : Array(String) = [] of String)
    wait_for_slot
    process = Process.new(command, args: args)
    @processes << process
  end
  
  def wait_all
    @processes.each(&.wait)
    @processes.clear
  end
  
  private def wait_for_slot
    while @processes.size >= @max_workers
      # ลบ processes ที่เสร็จแล้ว
      @processes.select!(&.terminated?)
      Fiber.yield
    end
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `ShellPipeline` ที่เชื่อมต่อหลาย commands เข้าด้วยกัน (เช่น `cat file | grep pattern | sort | uniq`)

### แบบฝึกหัดที่ 2
เขียน `CommandRunner` ที่รัน commands พร้อม timeout และ retry

### เฉลย

```crystal
# แบบฝึกหัดที่ 2: CommandRunner with timeout and retry
class CommandRunner
  def initialize(
    @max_retries : Int32 = 3,
    @retry_delay : Float64 = 1.0,
    @timeout : Float64? = nil
  )
  end
  
  def run(command : String, args : Array(String) = [] of String) : {Bool, String, String}
    attempt = 0
    
    loop do
      attempt += 1
      stdout = IO::Memory.new
      stderr = IO::Memory.new
      
      result = Process.run(command, args: args, output: stdout, error: stderr)
      
      if result.success?
        return {true, stdout.to_s, stderr.to_s}
      end
      
      if attempt >= @max_retries
        return {false, stdout.to_s, stderr.to_s}
      end
      
      puts "Command failed (attempt #{attempt}/#{@max_retries}), retrying in #{@retry_delay}s..."
      sleep @retry_delay.seconds
    end
  end
end

runner = CommandRunner.new(max_retries: 3, retry_delay: 0.5)
success, out, err = runner.run("ls", ["-la"])
puts success ? "Success: #{out}" : "Failed: #{err}"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Process.run** - รัน command และรอให้เสร็จ
2. **Process.exec** - แทนที่ current process
3. **Process::Status** - ตรวจสอบ exit status
4. **Capturing output** - stdout, stderr
5. **Process.pid** - PID management
6. **Process.exit** - ออกจาก program
7. **Shell safety** - ป้องกัน command injection
8. **Process management** - spawn, terminate, wait

Process management เป็นส่วนสำคัญของ system programming และ automation tools
