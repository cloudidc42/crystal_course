# Part 172: LibC และ System Calls ใน Crystal

## บทนำ

`lib LibC` ใน Crystal เป็น built-in library ที่ให้ access C standard library และ system functions ทำให้เราใช้ low-level OS functions ได้โดยตรง

## lib LibC - Built-in

```crystal
# Crystal มี LibC built-in อยู่แล้ว
# ไม่ต้อง require อะไร

# ดู types ที่มีใน LibC
puts LibC::SizeT.new(100)     # platform-specific size type
puts LibC::SSizeT.new(-1)     # signed size type
puts LibC::PtrdiffT.new(0)    # pointer difference type
puts LibC::TimeT.new(0)       # time_t
puts LibC::ClockT.new(0)      # clock_t
```

## C Standard Library Functions

### String Functions

```crystal
@[Link("c")]
lib LibCStr
  fun strlen(str : UInt8*) : LibC::SizeT
  fun strcmp(s1 : UInt8*, s2 : UInt8*) : Int32
  fun strncmp(s1 : UInt8*, s2 : UInt8*, n : LibC::SizeT) : Int32
  fun strcpy(dst : UInt8*, src : UInt8*) : UInt8*
  fun strncpy(dst : UInt8*, src : UInt8*, n : LibC::SizeT) : UInt8*
  fun strcat(dst : UInt8*, src : UInt8*) : UInt8*
  fun strchr(s : UInt8*, c : Int32) : UInt8*
  fun strstr(haystack : UInt8*, needle : UInt8*) : UInt8*
  fun strtol(str : UInt8*, endptr : UInt8**, base : Int32) : Int64
  fun strtod(str : UInt8*, endptr : UInt8**) : Float64
  fun sprintf(buf : UInt8*, fmt : UInt8*, ...) : Int32
  fun snprintf(buf : UInt8*, size : LibC::SizeT, fmt : UInt8*, ...) : Int32
  fun memcpy(dst : Void*, src : Void*, n : LibC::SizeT) : Void*
  fun memset(s : Void*, c : Int32, n : LibC::SizeT) : Void*
  fun memcmp(s1 : Void*, s2 : Void*, n : LibC::SizeT) : Int32
end

# ใช้งาน
str = "Hello, World!"
len = LibCStr.strlen(str)
puts "Length: #{len}"  # 13

# String comparison
cmp = LibCStr.strcmp("abc", "abd")
puts cmp < 0 ? "abc < abd" : (cmp > 0 ? "abc > abd" : "equal")

# ค้นหา substring
haystack = "The quick brown fox"
if (pos = LibCStr.strstr(haystack, "brown"))
  offset = pos - haystack.to_unsafe
  puts "Found 'brown' at position #{offset}"
end

# Convert string to number
endptr = Pointer(UInt8).null
num = LibCStr.strtol("42abc", pointerof(endptr), 10)
puts "Parsed: #{num}"  # 42
```

### Time Functions

```crystal
lib LibTime
  struct Tm
    sec    : Int32   # 0-60
    min    : Int32   # 0-59
    hour   : Int32   # 0-23
    mday   : Int32   # 1-31
    mon    : Int32   # 0-11
    year   : Int32   # years since 1900
    wday   : Int32   # 0-6 (Sunday=0)
    yday   : Int32   # 0-365
    isdst  : Int32   # DST flag
  end

  fun time(t : LibC::TimeT*) : LibC::TimeT
  fun localtime(timep : LibC::TimeT*) : Tm*
  fun gmtime(timep : LibC::TimeT*) : Tm*
  fun mktime(tm : Tm*) : LibC::TimeT
  fun strftime(s : UInt8*, max : LibC::SizeT, fmt : UInt8*, tm : Tm*) : LibC::SizeT
  fun difftime(time1 : LibC::TimeT, time0 : LibC::TimeT) : Float64

  # High resolution time
  struct TimeSpec
    tv_sec  : LibC::TimeT    # seconds
    tv_nsec : Int64           # nanoseconds
  end

  CLOCK_REALTIME  = 0
  CLOCK_MONOTONIC = 1

  fun clock_gettime(clockid : Int32, tp : TimeSpec*) : Int32
  fun nanosleep(req : TimeSpec*, rem : TimeSpec*) : Int32
end

# ใช้งาน time functions
current = LibC::TimeT.new(0)
LibTime.time(pointerof(current))
puts "Unix timestamp: #{current}"

# แปลงเป็น local time
local_tm = LibTime.localtime(pointerof(current))
if local_tm
  tm = local_tm.value
  year = tm.year + 1900
  month = tm.mon + 1
  day = tm.mday
  puts "#{year}-#{month.to_s.rjust(2, '0')}-#{day.to_s.rjust(2, '0')}"
end

# Format time
if local_tm
  buffer = Bytes.new(64)
  LibTime.strftime(buffer.to_unsafe, 64, "%Y-%m-%d %H:%M:%S", local_tm)
  puts String.new(buffer.to_unsafe)
end

# High resolution time
ts = LibTime::TimeSpec.new
LibTime.clock_gettime(LibTime::CLOCK_MONOTONIC, pointerof(ts))
puts "Nanoseconds: #{ts.tv_nsec}"

# Sleep น้อยกว่า 1 วินาที
sleep_time = LibTime::TimeSpec.new(tv_sec: 0, tv_nsec: 100_000_000)  # 100ms
remaining = LibTime::TimeSpec.new
LibTime.nanosleep(pointerof(sleep_time), pointerof(remaining))
```

### Math Functions

```crystal
@[Link("m")]
lib LibMath
  fun sin(x : Float64) : Float64
  fun cos(x : Float64) : Float64
  fun tan(x : Float64) : Float64
  fun asin(x : Float64) : Float64
  fun acos(x : Float64) : Float64
  fun atan(x : Float64) : Float64
  fun atan2(y : Float64, x : Float64) : Float64
  fun sqrt(x : Float64) : Float64
  fun cbrt(x : Float64) : Float64
  fun pow(x : Float64, y : Float64) : Float64
  fun exp(x : Float64) : Float64
  fun exp2(x : Float64) : Float64
  fun log(x : Float64) : Float64
  fun log2(x : Float64) : Float64
  fun log10(x : Float64) : Float64
  fun floor(x : Float64) : Float64
  fun ceil(x : Float64) : Float64
  fun round(x : Float64) : Float64
  fun fabs(x : Float64) : Float64
  fun fmod(x : Float64, y : Float64) : Float64
  fun hypot(x : Float64, y : Float64) : Float64
  fun isnan(x : Float64) : Int32
  fun isinf(x : Float64) : Int32
  fun isfinite(x : Float64) : Int32

  # Constants
  fun M_PI : Float64  # ไม่มีจริง แต่ Crystal ใช้ Math::PI
end

# ใช้งาน
angle = Math::PI / 4  # 45 degrees
puts "sin(45°) = #{LibMath.sin(angle).round(4)}"
puts "cos(45°) = #{LibMath.cos(angle).round(4)}"
puts "tan(45°) = #{LibMath.tan(angle).round(4)}"
puts "atan2(1, 1) = #{LibMath.atan2(1.0, 1.0)}"

# Hypotenuse
puts "hypot(3, 4) = #{LibMath.hypot(3.0, 4.0)}"  # 5.0

# Log
puts "log(e) = #{LibMath.log(Math::E)}"  # 1.0
puts "log2(8) = #{LibMath.log2(8.0)}"    # 3.0
puts "log10(100) = #{LibMath.log10(100.0)}"  # 2.0

# Check special values
nan = 0.0 / 0.0
puts "isnan(NaN) = #{LibMath.isnan(nan) != 0}"
inf = 1.0 / 0.0
puts "isinf(Inf) = #{LibMath.isinf(inf) != 0}"
```

### File I/O Functions

```crystal
lib LibFile
  FILE = Void

  O_RDONLY = 0
  O_WRONLY = 1
  O_RDWR   = 2
  O_CREAT  = 64
  O_TRUNC  = 512
  O_APPEND = 1024

  SEEK_SET = 0
  SEEK_CUR = 1
  SEEK_END = 2

  fun open(pathname : UInt8*, flags : Int32, mode : Int32) : Int32
  fun close(fd : Int32) : Int32
  fun read(fd : Int32, buf : Void*, count : LibC::SizeT) : LibC::SSizeT
  fun write(fd : Int32, buf : Void*, count : LibC::SizeT) : LibC::SSizeT
  fun lseek(fd : Int32, offset : Int64, whence : Int32) : Int64
  fun ftruncate(fd : Int32, length : Int64) : Int32
  fun fsync(fd : Int32) : Int32

  # stat
  struct Stat
    st_dev   : UInt64
    st_ino   : UInt64
    st_mode  : UInt32
    st_nlink : UInt64
    st_uid   : UInt32
    st_gid   : UInt32
    st_size  : Int64
    st_atime : LibC::TimeT
    st_mtime : LibC::TimeT
    st_ctime : LibC::TimeT
  end

  fun stat(pathname : UInt8*, statbuf : Stat*) : Int32
  fun fstat(fd : Int32, statbuf : Stat*) : Int32
end

# ใช้งาน file functions
def read_file_c(path : String) : String?
  fd = LibFile.open(path, LibFile::O_RDONLY, 0)
  return nil if fd < 0

  begin
    # หาขนาดไฟล์
    stat = LibFile::Stat.new
    LibFile.fstat(fd, pointerof(stat))
    size = stat.st_size.to_i

    # อ่านไฟล์
    buffer = Bytes.new(size)
    bytes = LibFile.read(fd, buffer.to_unsafe, size.to_u64)
    return nil if bytes < 0

    String.new(buffer[0, bytes.to_i])
  ensure
    LibFile.close(fd)
  end
end

# ใช้งาน
if content = read_file_c("/etc/hostname")
  puts "Hostname: #{content.strip}"
end
```

### Process Functions

```crystal
lib LibProcess
  fun getpid() : Int32
  fun getppid() : Int32
  fun getuid() : UInt32
  fun geteuid() : UInt32
  fun getgid() : UInt32
  fun getegid() : UInt32
  fun fork() : Int32
  fun waitpid(pid : Int32, status : Int32*, options : Int32) : Int32
  fun execvp(file : UInt8*, argv : UInt8**) : Int32
  fun system(command : UInt8*) : Int32
  fun exit(status : Int32) : NoReturn
  fun abort() : NoReturn
  fun getenv(name : UInt8*) : UInt8*
  fun setenv(name : UInt8*, value : UInt8*, overwrite : Int32) : Int32
  fun unsetenv(name : UInt8*) : Int32

  fun getcwd(buf : UInt8*, size : LibC::SizeT) : UInt8*
  fun chdir(path : UInt8*) : Int32
  fun mkdir(pathname : UInt8*, mode : UInt32) : Int32
  fun rmdir(pathname : UInt8*) : Int32
  fun unlink(pathname : UInt8*) : Int32
  fun rename(oldpath : UInt8*, newpath : UInt8*) : Int32
end

# Process info
puts "PID: #{LibProcess.getpid}"
puts "Parent PID: #{LibProcess.getppid}"
puts "UID: #{LibProcess.getuid}"

# Environment variables
if val = LibProcess.getenv("HOME")
  puts "HOME: #{String.new(val)}"
end

LibProcess.setenv("MY_VAR", "hello", 1)
if val = LibProcess.getenv("MY_VAR")
  puts "MY_VAR: #{String.new(val)}"
end

# Current directory
buffer = Bytes.new(4096)
if (ptr = LibProcess.getcwd(buffer.to_unsafe, 4096_u64))
  puts "CWD: #{String.new(ptr)}"
end
```

### Signal Handling

```crystal
lib LibSignal
  alias SigHandlerFn = Int32 -> Void

  SIGINT  =  2
  SIGTERM = 15
  SIGHUP  =  1
  SIGUSR1 = 10
  SIGUSR2 = 12
  SIGPIPE = 13
  SIGALRM = 14
  SIGCHLD = 17

  SIG_DFL = Pointer(Void).null
  SIG_IGN = Pointer(Void).new(1)

  fun signal(signum : Int32, handler : SigHandlerFn) : SigHandlerFn
  fun raise(sig : Int32) : Int32
  fun kill(pid : Int32, sig : Int32) : Int32
  fun alarm(seconds : UInt32) : UInt32
end

# ตั้ง signal handler
handler = ->(sig : Int32) {
  puts "\nReceived signal #{sig}. Cleaning up..."
  exit(0)
}

LibSignal.signal(LibSignal::SIGINT, handler)
LibSignal.signal(LibSignal::SIGTERM, handler)

puts "Running... (press Ctrl+C to stop)"
loop { sleep(1) }
```

## แบบฝึกหัด

1. เขียน Crystal program ที่ใช้ `fork()` สร้าง child process แล้ว parent รอด้วย `waitpid()`
2. สร้าง timer ด้วย `SIGALRM` ที่ส่ง signal ทุก N วินาที
3. Wrap `getaddrinfo` เพื่อ DNS resolution
4. ใช้ `mmap` สำหรับ memory-mapped files

## สรุป

LibC ใน Crystal:
- **String functions**: strlen, strcmp, strstr, strtol ฯลฯ
- **Time functions**: time, localtime, clock_gettime สำหรับ precision timing
- **Math functions**: sin, cos, sqrt, log ฯลฯ
- **File I/O**: open, read, write, lseek ระดับ system
- **Process**: getpid, fork, execvp, getenv ฯลฯ
- **Signals**: SIGINT, SIGTERM handlers

LibC เปิด access ไปยัง OS primitives ที่ Crystal stdlib อาจยังไม่ wrap
