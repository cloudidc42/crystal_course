# Part 72: Directory Operations

## บทนำ

การจัดการ directories เป็นส่วนสำคัญของ file system programming Crystal มี `Dir` module และ `FileUtils` สำหรับการดำเนินการต่างๆ กับ directories

---

## 1. Dir.mkdir และ Dir.mkdir_p

```crystal
# สร้าง directory
Dir.mkdir("new_dir")  # สร้าง directory ใหม่
# raises File::AlreadyExistsError ถ้ามีอยู่แล้ว

# สร้างพร้อม error handling
begin
  Dir.mkdir("my_dir")
  puts "Created directory"
rescue File::AlreadyExistsError
  puts "Directory already exists"
rescue File::Error => e
  puts "Error: #{e.message}"
end

# mkdir_p - สร้าง nested directories
Dir.mkdir_p("path/to/deeply/nested/dir")
# ไม่ raise error ถ้ามีอยู่แล้ว

# สร้าง directory พร้อม permissions
Dir.mkdir("secured_dir", 0o700)  # rwx------

# ตรวจสอบก่อนสร้าง
def ensure_dir(path : String)
  Dir.mkdir_p(path) unless Dir.exists?(path)
end

ensure_dir("logs/2024/01")
ensure_dir("cache/images")
ensure_dir("tmp/uploads")
```

---

## 2. Dir.ls / Dir[]

```crystal
# Dir.entries - list all entries (รวม . และ ..)
entries = Dir.entries(".")
puts entries.inspect

# Dir.children - เฉพาะ children (ไม่รวม . และ ..)
children = Dir.children(".")
puts children.inspect

# Dir[] - glob pattern
crystal_files = Dir["*.cr"]
puts crystal_files.inspect

# รูปแบบต่างๆ
Dir["*.txt"].each { |f| puts f }          # txt files ใน current dir
Dir["**/*.cr"].each { |f| puts f }        # cr files recursively
Dir["src/**/*.cr"].each { |f| puts f }    # cr files in src/
Dir["test/*.{rb,cr}"].each { |f| puts f } # rb หรือ cr ใน test/

# เฉพาะ files (ไม่รวม directories)
Dir["*"].select { |f| File.file?(f) }.each { |f| puts f }

# เฉพาะ directories
Dir["*"].select { |d| Dir.exists?(d) }.each { |d| puts d }

# hidden files (ขึ้นต้นด้วย .)
Dir[".*"].each { |f| puts f }  # hidden files

# sort by modification time
Dir["*.txt"]
  .sort_by { |f| File.info(f).modification_time }
  .reverse
  .each { |f| puts "#{f}: #{File.size(f)} bytes" }
```

---

## 3. Dir.cd

```crystal
# เปลี่ยน working directory
original = Dir.current
puts "Original: #{original}"

Dir.cd("/tmp") do
  puts "Inside: #{Dir.current}"
  # do something in /tmp
end

puts "Back to: #{Dir.current}"  # กลับมา original

# Dir.cd ไม่มี block (permanent change)
Dir.cd("/tmp")
puts Dir.current  # => /tmp

# กลับไปเอง
Dir.cd(original)

# ใช้ in project structure
def run_in_project_root(&block)
  root = find_project_root
  Dir.cd(root) { block.call }
end

def find_project_root : String
  dir = Dir.current
  loop do
    return dir if File.exists?(File.join(dir, "shard.yml"))
    parent = File.dirname(dir)
    return Dir.current if parent == dir  # reached filesystem root
    dir = parent
  end
end
```

---

## 4. Dir.current

```crystal
# รับ current working directory
cwd = Dir.current
puts cwd

# ตรวจสอบว่าอยู่ใน right directory
def in_project_root? : Bool
  File.exists?(File.join(Dir.current, "shard.yml"))
end

if in_project_root?
  puts "Running from project root"
else
  puts "Warning: Not in project root"
end

# relative vs absolute paths
puts Dir.current          # absolute path
puts File.expand_path(".")  # same thing
puts File.expand_path("..")  # parent directory
puts File.expand_path("~")   # home directory
```

---

## 5. Dir.glob

```crystal
# Dir.glob - pattern matching
Dir.glob("**/*.cr") do |path|
  puts path
end

# ส่ง results เป็น array
files = Dir.glob("src/**/*.cr")

# รูปแบบ patterns:
# * - matches any string (not including /)
# ** - matches any string (including /)
# ? - matches any single character
# {a,b} - matches a or b
# [abc] - matches a, b, or c

# หาไฟล์ที่ modified ล่าสุด
def latest_modified_file(pattern : String) : String?
  Dir.glob(pattern)
    .select { |f| File.file?(f) }
    .max_by? { |f| File.info(f).modification_time }
end

puts latest_modified_file("**/*.cr")

# หาไฟล์ขนาดใหญ่
def large_files(dir : String, min_size_mb : Float64) : Array(String)
  min_bytes = (min_size_mb * 1_000_000).to_i64
  Dir.glob("#{dir}/**/*")
    .select { |f| File.file?(f) && File.size(f) >= min_bytes }
    .sort_by { |f| -File.size(f) }
end

large = large_files(".", 0.1)  # files > 100KB
large.each do |f|
  size_kb = File.size(f) / 1000.0
  puts "#{f}: #{size_kb.round(1)} KB"
end

# ลบ files ที่ match pattern
def clean_temp_files(pattern : String)
  count = 0
  Dir.glob(pattern) do |f|
    File.delete(f)
    count += 1
  end
  puts "Deleted #{count} files"
end
```

---

## 6. Dir.each

```crystal
# วนลูปผ่านทุก entries
Dir.each(".") do |entry|
  puts entry
end

# เทียบเท่า
Dir.each_child(".") do |child|
  path = File.join(".", child)
  if File.file?(path)
    puts "FILE: #{child}"
  elsif Dir.exists?(path)
    puts "DIR:  #{child}"
  end
end

# Recursive directory walk (ด้วย each_child)
def walk_directory(dir : String, indent : Int32 = 0)
  puts "  " * indent + File.basename(dir) + "/"
  
  Dir.each_child(dir) do |child|
    path = File.join(dir, child)
    if File.directory?(path) && !File.symlink?(path)
      walk_directory(path, indent + 1)
    else
      puts "  " * (indent + 1) + child
    end
  end
end

walk_directory(".")

# Dir.open สำหรับ manual iteration
Dir.open(".") do |dir|
  while entry = dir.read
    next if entry == "." || entry == ".."
    puts entry
  end
end
```

---

## 7. Dir.exists?

```crystal
# ตรวจสอบว่า directory มีอยู่
puts Dir.exists?(".")         # => true
puts Dir.exists?("nonexistent")  # => false
puts Dir.exists?("file.txt")     # => false (เป็น file ไม่ใช่ dir)

# File.directory? ก็ใช้ได้
puts File.directory?(".")     # => true

# ตรวจสอบก่อน create
def create_if_not_exists(path : String) : Bool
  if Dir.exists?(path)
    false  # already exists
  else
    Dir.mkdir_p(path)
    true  # created
  end
end

puts create_if_not_exists("new_dir")   # => true (created)
puts create_if_not_exists("new_dir")   # => false (exists)
Dir.delete("new_dir")

# ตรวจสอบ path เป็น file หรือ directory
def path_type(path : String) : String
  if File.exists?(path)
    if File.directory?(path)
      "directory"
    elsif File.symlink?(path)
      "symlink"
    else
      "file"
    end
  else
    "not found"
  end
end

puts path_type(".")       # => directory
puts path_type("file.cr") # depends
```

---

## 8. FileUtils

```crystal
require "file_utils"

# FileUtils มี high-level operations

# copy file
FileUtils.cp("source.txt", "dest.txt")

# copy directory recursively
FileUtils.cp_r("source_dir", "dest_dir")

# move/rename
FileUtils.mv("old.txt", "new.txt")
FileUtils.mv("old_dir", "new_dir")

# remove file
FileUtils.rm("file.txt")

# remove directory recursively (อันตราย!)
FileUtils.rm_r("temp_dir")
FileUtils.rm_rf("temp_dir")  # ไม่ raise error ถ้าไม่มี

# mkdir
FileUtils.mkdir("new_dir")
FileUtils.mkdir_p("a/b/c")

# สร้าง multiple directories
FileUtils.mkdir(["dir1", "dir2", "dir3"])

# touch (สร้างหรืออัปเดต timestamp)
FileUtils.touch("file.txt")
FileUtils.touch(["file1.txt", "file2.txt"])

# ตัวอย่างจริง: organize files by extension
def organize_by_extension(source_dir : String, dest_dir : String)
  FileUtils.mkdir_p(dest_dir)
  
  Dir.each_child(source_dir) do |filename|
    path = File.join(source_dir, filename)
    next unless File.file?(path)
    
    ext = File.extname(filename).lstrip(".").downcase
    ext = "misc" if ext.empty?
    
    target_dir = File.join(dest_dir, ext)
    FileUtils.mkdir_p(target_dir)
    FileUtils.mv(path, File.join(target_dir, filename))
  end
end
```

---

## 9. Walking Directory Tree

```crystal
# Full directory tree walker
struct DirEntry
  getter path : String
  getter name : String
  getter type : Symbol
  getter size : Int64
  getter depth : Int32
  
  def initialize(@path, @name, @type, @size, @depth)
  end
end

def walk_tree(root : String, max_depth : Int32 = -1, &block : DirEntry ->)
  _walk_tree(root, 0, max_depth, block)
end

private def _walk_tree(dir : String, depth : Int32, max_depth : Int32, block : DirEntry ->)
  return if max_depth >= 0 && depth >= max_depth
  
  Dir.each_child(dir) do |child|
    path = File.join(dir, child)
    
    begin
      if File.directory?(path) && !File.symlink?(path)
        entry = DirEntry.new(path, child, :dir, 0i64, depth)
        block.call(entry)
        _walk_tree(path, depth + 1, max_depth, block)
      else
        size = File.exists?(path) ? File.size(path) : 0i64
        entry = DirEntry.new(path, child, :file, size, depth)
        block.call(entry)
      end
    rescue ex
      STDERR.puts "Error accessing #{path}: #{ex.message}"
    end
  end
end

# สถิติของ directory
def directory_stats(root : String) : Hash(String, Int64 | Int32)
  stats = {
    "total_files" => 0i64,
    "total_dirs" => 0i64,
    "total_size" => 0i64,
    "max_depth" => 0i64,
  } of String => Int64 | Int32
  
  walk_tree(root) do |entry|
    if entry.type == :file
      stats["total_files"] = stats["total_files"].as(Int64) + 1
      stats["total_size"] = stats["total_size"].as(Int64) + entry.size
    else
      stats["total_dirs"] = stats["total_dirs"].as(Int64) + 1
    end
    depth = entry.depth.to_i64
    if depth > stats["max_depth"].as(Int64)
      stats["max_depth"] = depth
    end
  end
  
  stats
end

# Extension breakdown
def extension_summary(root : String) : Hash(String, {count: Int32, total_size: Int64})
  summary = {} of String => {count: Int32, total_size: Int64}
  
  walk_tree(root) do |entry|
    next if entry.type == :dir
    
    ext = File.extname(entry.name).downcase
    ext = "(no extension)" if ext.empty?
    
    existing = summary[ext]? || {count: 0, total_size: 0i64}
    summary[ext] = {
      count: existing[:count] + 1,
      total_size: existing[:total_size] + entry.size
    }
  end
  
  summary
end

# Print tree
def print_tree(root : String, max_depth : Int32 = 3)
  puts File.basename(root) + "/"
  
  walk_tree(root, max_depth) do |entry|
    indent = "  " * (entry.depth + 1)
    if entry.type == :dir
      puts "#{indent}#{entry.name}/"
    else
      size_str = format_size(entry.size)
      puts "#{indent}#{entry.name} (#{size_str})"
    end
  end
end

def format_size(bytes : Int64) : String
  if bytes < 1024
    "#{bytes} B"
  elsif bytes < 1_048_576
    "#{(bytes / 1024.0).round(1)} KB"
  elsif bytes < 1_073_741_824
    "#{(bytes / 1_048_576.0).round(1)} MB"
  else
    "#{(bytes / 1_073_741_824.0).round(1)} GB"
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `find_duplicates(dir)` ที่หาไฟล์ที่มีเนื้อหาซ้ำกันโดยเปรียบเทียบ hash ของไฟล์

### แบบฝึกหัดที่ 2
เขียน `backup(source, dest)` ที่ทำ incremental backup โดย copy เฉพาะไฟล์ที่เปลี่ยนแปลง

### แบบฝึกหัดที่ 3
เขียน `find_files(root, &block)` ที่วนหาไฟล์แบบ recursive พร้อม filter conditions

### เฉลย

```crystal
# แบบฝึกหัดที่ 1 (simplified - ใช้ file size comparison)
require "digest/md5"

def find_duplicates(dir : String) : Hash(String, Array(String))
  by_hash = {} of String => Array(String)
  
  walk_tree(dir) do |entry|
    next if entry.type == :dir
    
    begin
      content = File.read(entry.path)
      hash = Digest::MD5.hexdigest(content)
      by_hash[hash] ||= [] of String
      by_hash[hash] << entry.path
    rescue ex
      # skip files we can't read
    end
  end
  
  by_hash.select { |_, paths| paths.size > 1 }
end

dups = find_duplicates(".")
dups.each do |hash, paths|
  puts "Duplicates (#{hash[0..7]}...):"
  paths.each { |p| puts "  #{p}" }
end

# แบบฝึกหัดที่ 3
def find_files(root : String, name_pattern : Regex? = nil, 
               min_size : Int64? = nil, max_size : Int64? = nil,
               modified_after : Time? = nil) : Array(String)
  results = [] of String
  
  walk_tree(root) do |entry|
    next if entry.type == :dir
    
    if name_pattern
      next unless entry.name.match?(name_pattern)
    end
    
    if min_size
      next if entry.size < min_size
    end
    
    if max_size
      next if entry.size > max_size
    end
    
    if modified_after
      next if File.info(entry.path).modification_time < modified_after
    end
    
    results << entry.path
  end
  
  results
end

# หาไฟล์ Crystal ที่ใหญ่กว่า 1KB
puts find_files(".", name_pattern: /\.cr$/, min_size: 1024).inspect
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Dir.mkdir** และ **Dir.mkdir_p** - สร้าง directories
2. **Dir.ls** และ **Dir[]** - list และ glob files
3. **Dir.cd** - เปลี่ยน working directory
4. **Dir.current** - รับ current directory
5. **Dir.glob** - pattern-based file searching
6. **Dir.each** - วนลูปผ่าน directory entries
7. **Dir.exists?** - ตรวจสอบ directory
8. **FileUtils** - high-level file operations
9. **Directory tree walking** - recursive traversal

Directory operations เป็นพื้นฐานสำคัญสำหรับ file management, build tools, และ deployment scripts
