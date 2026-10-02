# Part 75: Paths และ File System

## บทนำ

Crystal มี `Path` class สำหรับการจัดการ file paths อย่างมีโครงสร้าง รวมถึง methods ใน `File` และ `Dir` สำหรับ path manipulation

---

## 1. Path Class

```crystal
# สร้าง Path
p = Path.new("hello/world.txt")
puts p          # => hello/world.txt
puts p.class    # => Path

# Path จาก parts
p = Path.new("hello", "world", "file.txt")
puts p  # => hello/world/file.txt

# Path constants
puts Path::SEPARATOR         # => "/" on Unix, "\\" on Windows
puts Path::ALT_SEPARATOR     # => nil on Unix

# เปรียบเทียบ
puts Path.new("a/b") == Path.new("a/b")  # => true
puts Path.new("a/b") == Path.new("a/c")  # => false

# Posix vs Windows paths
posix_path = Path.posix("usr/local/bin")
windows_path = Path.windows("C:\\Users\\Alice")

puts posix_path    # => usr/local/bin
puts windows_path  # => C:\Users\Alice

# String ที่เป็น path
def is_path?(str : String) : Bool
  str.includes?(Path::SEPARATOR) || str.includes?(".")
end
```

---

## 2. Path Joining

```crystal
# join paths ด้วย / operator
base = Path.new("/home/user")
result = base / "documents" / "file.txt"
puts result  # => /home/user/documents/file.txt

# Path.new กับ multiple parts
p = Path.new("/home", "user", "documents", "file.txt")
puts p  # => /home/user/documents/file.txt

# join method
p = Path.new("/home/user")
puts p.join("documents", "file.txt")  # => /home/user/documents/file.txt

# File.join
puts File.join("home", "user", "file.txt")  # => home/user/file.txt
puts File.join("/", "home", "user")          # => /home/user

# เพิ่ม segment
path = Path.new("projects")
["src", "main.cr"].each do |part|
  path = path / part
end
puts path  # => projects/src/main.cr

# normalize path
puts Path.new("./hello/../world").normalize  # => world
puts Path.new("a//b///c").normalize          # => a/b/c
puts Path.new("./a/./b").normalize           # => a/b
```

---

## 3. dirname, basename, extname

```crystal
path = Path.new("/home/user/documents/file.txt")

# dirname - directory part
puts path.dirname  # => /home/user/documents

# basename - filename part
puts path.basename           # => file.txt
puts path.basename(".txt")   # => file (remove extension)

# stem - filename without extension
puts path.stem               # => file

# extname - extension
puts path.extension          # => .txt
puts Path.new("file.tar.gz").extension    # => .gz
puts Path.new("no_extension").extension  # => ""

# ตัวอย่าง
def file_info(path_str : String)
  p = Path.new(path_str)
  puts "Path: #{p}"
  puts "Dir:  #{p.dirname}"
  puts "Base: #{p.basename}"
  puts "Stem: #{p.stem}"
  puts "Ext:  #{p.extension}"
end

file_info("/home/user/docs/report.pdf")
puts "---"
file_info("relative/path/to/file.tar.gz")
puts "---"
file_info("nopath.txt")

# เปลี่ยน extension
def change_extension(path : String, new_ext : String) : String
  p = Path.new(path)
  (p.dirname / (p.stem + new_ext)).to_s
end

puts change_extension("file.txt", ".md")           # => file.md
puts change_extension("src/main.cr", ".js")        # => src/main.js
puts change_extension("/path/to/file.txt", ".bak") # => /path/to/file.bak
```

---

## 4. absolute? / relative?

```crystal
# ตรวจสอบ absolute/relative
puts Path.new("/home/user").absolute?   # => true
puts Path.new("relative").absolute?     # => false
puts Path.new("./local").absolute?      # => false

puts Path.new("/home/user").relative?   # => false
puts Path.new("relative").relative?     # => true

# แปลงเป็น absolute
relative = Path.new("documents/file.txt")
absolute = relative.expand  # ใช้ current directory
puts absolute

# ระบุ base directory
absolute = relative.expand(from: Path.new("/home/user"))
puts absolute  # => /home/user/documents/file.txt

# File.expand_path
puts File.expand_path("~/documents")  # expand ~ to home
puts File.expand_path("./relative", "/base")
puts File.expand_path("../parent")

# relative_to
from = Path.new("/home/user/projects")
to = Path.new("/home/user/projects/src/main.cr")
puts to.relative_to(from)  # => src/main.cr

# กลับกัน
from = Path.new("/home/user/projects/src")
to = Path.new("/home/user/documents/file.txt")
puts to.relative_to(from)  # => ../../documents/file.txt
```

---

## 5. Glob Patterns

```crystal
# Path.glob
Dir.glob("**/*.cr").each { |f| puts f }

# ใช้กับ specific patterns
Dir.glob("src/**/*.cr") do |path|
  # process Crystal files in src/
  puts path
end

# หลาย patterns
patterns = ["**/*.txt", "**/*.md", "**/*.json"]
all_files = patterns.flat_map { |p| Dir.glob(p) }.uniq
all_files.each { |f| puts f }

# Pattern matching ใน Path
def matches_pattern?(path : String, pattern : String) : Bool
  File.match?(pattern, path)
end

puts matches_pattern?("src/main.cr", "**/*.cr")    # => true
puts matches_pattern?("test/spec.cr", "**/*.cr")   # => true
puts matches_pattern?("README.md", "**/*.cr")      # => false

# สร้าง glob helper
def find_source_files(dir : String) : Array(String)
  Dir.glob("#{dir}/**/*.cr")
    .reject { |f| f.includes?("spec") }
    .reject { |f| f.includes?(".crystal") }
    .sort
end

def find_test_files(dir : String) : Array(String)
  Dir.glob("#{dir}/**/spec_*.cr") +
  Dir.glob("#{dir}/**/*_spec.cr")
end
```

---

## 6. File.join

```crystal
# File.join สำหรับ cross-platform path building
puts File.join("home", "user", "file.txt")
puts File.join("/", "home", "user")
puts File.join("path", "", "to", "file")  # empty ignored

# กับ array
parts = ["home", "user", "documents", "file.txt"]
puts File.join(parts)

# สร้าง paths อย่างปลอดภัย
def safe_join(*parts : String) : String
  # ลบ path traversal attacks
  cleaned = parts.map { |p| p.gsub("..", "").gsub("//", "/") }
  File.join(cleaned)
end

# Project-relative paths
class ProjectPaths
  def initialize(@root : String)
  end
  
  def root : String
    @root
  end
  
  def src(*parts : String) : String
    File.join(@root, "src", *parts)
  end
  
  def spec(*parts : String) : String
    File.join(@root, "spec", *parts)
  end
  
  def lib(*parts : String) : String
    File.join(@root, "lib", *parts)
  end
  
  def build(*parts : String) : String
    File.join(@root, ".build", *parts)
  end
end

project = ProjectPaths.new("/home/user/myproject")
puts project.src("models", "user.cr")
puts project.spec("models", "user_spec.cr")
puts project.lib("mylib", "lib.cr")
```

---

## 7. expand_path

```crystal
# expand_path: แปลง relative/~ เป็น absolute
puts File.expand_path("~/documents")     # => /home/user/documents
puts File.expand_path("~/.config")       # => /home/user/.config
puts File.expand_path("./relative")      # => /current/dir/relative
puts File.expand_path("../parent")       # => /current/parent
puts File.expand_path("..")              # => /current

# กับ base path
puts File.expand_path("relative", "/base")  # => /base/relative
puts File.expand_path("../up", "/a/b/c")    # => /a/b

# ตรวจสอบว่า path อยู่ภายใน directory (security)
def within_directory?(path : String, base : String) : Bool
  expanded_path = File.expand_path(path)
  expanded_base = File.expand_path(base)
  expanded_path.starts_with?(expanded_base + "/") || expanded_path == expanded_base
end

puts within_directory?("/home/user/docs/file.txt", "/home/user")  # => true
puts within_directory?("/etc/passwd", "/home/user")              # => false
puts within_directory?("/home/user/../etc/passwd", "/home/user") # => false

# ป้องกัน path traversal
def safe_path(user_input : String, base_dir : String) : String?
  full_path = File.expand_path(File.join(base_dir, user_input))
  within_directory?(full_path, base_dir) ? full_path : nil
end

base = "/var/www/uploads"
puts safe_path("image.jpg", base).inspect   # => "/var/www/uploads/image.jpg"
puts safe_path("../etc/passwd", base).inspect # => nil (path traversal blocked)
```

---

## 8. Path Utilities

```crystal
# Utility functions
module PathUtils
  # หา common ancestor
  def self.common_ancestor(paths : Array(String)) : String
    return "" if paths.empty?
    return paths.first if paths.size == 1
    
    expanded = paths.map { |p| File.expand_path(p) }
    parts = expanded.map { |p| p.split("/") }
    
    min_parts = parts.map(&.size).min
    common = [] of String
    
    (0...min_parts).each do |i|
      part = parts.first[i]
      if parts.all? { |p| p[i] == part }
        common << part
      else
        break
      end
    end
    
    common.join("/").gsub(%r{^$}, "/")
  end
  
  # สร้าง unique filename ถ้า file มีอยู่แล้ว
  def self.unique_name(path : String) : String
    return path unless File.exists?(path)
    
    dir = File.dirname(path)
    name = File.basename(path, File.extname(path))
    ext = File.extname(path)
    
    i = 1
    loop do
      candidate = File.join(dir, "#{name}_#{i}#{ext}")
      return candidate unless File.exists?(candidate)
      i += 1
    end
  end
  
  # ขนาด directory ทั้งหมด
  def self.directory_size(dir : String) : Int64
    total = 0i64
    Dir.glob("#{dir}/**/*") do |path|
      total += File.size(path) if File.file?(path)
    end
    total
  end
  
  # Relative path จาก src ไป dst
  def self.relative_path(from : String, to : String) : String
    from_parts = File.expand_path(from).split("/").reject(&.empty?)
    to_parts = File.expand_path(to).split("/").reject(&.empty?)
    
    # หา common prefix
    common_len = 0
    [from_parts.size, to_parts.size].min.times do |i|
      break unless from_parts[i] == to_parts[i]
      common_len += 1
    end
    
    # สร้าง relative path
    up_count = from_parts.size - common_len
    remaining = to_parts[common_len..]
    
    parts = Array.new(up_count, "..") + remaining
    parts.empty? ? "." : parts.join("/")
  end
end

paths = [
  "/home/user/projects/app/src",
  "/home/user/projects/app/test",
  "/home/user/projects/app/lib",
]
puts PathUtils.common_ancestor(paths)  # => /home/user/projects/app

puts PathUtils.relative_path("/a/b/c", "/a/d/e")  # => ../../d/e
puts PathUtils.relative_path("/a/b", "/a/b/c/d")  # => c/d
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `PathWatcher` ที่ monitor directory สำหรับ new files และ trigger callbacks

### แบบฝึกหัดที่ 2
เขียน `FileOrganizer` ที่จัด files ตาม date modified (ย้ายไป YYYY/MM/ subdirectories)

### แบบฝึกหัดที่ 3
เขียน `PathValidator` ที่ตรวจสอบว่า path ถูกต้องตาม rules ที่กำหนด

### เฉลย

```crystal
# แบบฝึกหัดที่ 2: FileOrganizer
require "file_utils"

def organize_by_date(source_dir : String, dest_dir : String, dry_run : Bool = false)
  moved = 0
  errors = 0
  
  Dir.each_child(source_dir) do |filename|
    src_path = File.join(source_dir, filename)
    next unless File.file?(src_path)
    
    begin
      mtime = File.info(src_path).modification_time
      dest_subdir = File.join(dest_dir, mtime.year.to_s, mtime.month.to_s.rjust(2, '0'))
      dest_path = File.join(dest_subdir, filename)
      
      puts "#{src_path} -> #{dest_path}"
      
      unless dry_run
        FileUtils.mkdir_p(dest_subdir)
        FileUtils.mv(src_path, dest_path)
      end
      
      moved += 1
    rescue ex
      STDERR.puts "Error processing #{src_path}: #{ex.message}"
      errors += 1
    end
  end
  
  puts "\nSummary: #{moved} files moved, #{errors} errors"
end

# แบบฝึกหัดที่ 3: PathValidator
class PathValidator
  record Rule, name : String, check : String -> Bool
  
  def initialize
    @rules = [] of Rule
  end
  
  def must_be_absolute
    @rules << Rule.new("must be absolute") { |p| File.expand_path(p) == p }
    self
  end
  
  def must_exist
    @rules << Rule.new("must exist") { |p| File.exists?(p) }
    self
  end
  
  def must_be_readable
    @rules << Rule.new("must be readable") { |p| File.readable?(p) }
    self
  end
  
  def extension_must_be(*exts : String)
    @rules << Rule.new("must have extension #{exts.join("|")}") do |p|
      exts.any? { |e| p.ends_with?(e) }
    end
    self
  end
  
  def max_depth(n : Int32)
    @rules << Rule.new("depth must be <= #{n}") do |p|
      p.split("/").reject(&.empty?).size <= n
    end
    self
  end
  
  def validate(path : String) : Array(String)
    @rules.each_with_object([] of String) do |rule, errors|
      errors << rule.name unless rule.check.call(path)
    end
  end
  
  def valid?(path : String) : Bool
    validate(path).empty?
  end
end

validator = PathValidator.new
  .must_be_absolute
  .extension_must_be(".cr", ".rb")

puts validator.valid?("/home/user/main.cr")   # => true
puts validator.valid?("relative/path.cr")     # => false (not absolute)
puts validator.valid?("/home/user/file.txt")  # => false (wrong extension)
puts validator.validate("/home/user/file.txt").inspect
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Path class** - structured path handling
2. **Path joining** - `/` operator, `join`, `File.join`
3. **dirname, basename, extname** - แยกส่วนต่างๆ ของ path
4. **absolute? / relative?** - ตรวจสอบ path type
5. **Glob patterns** - ค้นหาไฟล์ด้วย patterns
6. **expand_path** - แปลงเป็น absolute path
7. **Path security** - ป้องกัน path traversal
8. **Path utilities** - common ancestor, unique names

Path manipulation ที่ถูกต้องเป็นสิ่งสำคัญสำหรับ file system security และ cross-platform compatibility
