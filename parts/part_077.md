# Part 77: Command Line Arguments

## บทนำ

Crystal มี `OptionParser` class สำหรับ parse command line arguments อย่างมีโครงสร้าง รองรับ flags, options, subcommands, และ help text

---

## 1. ARGV พื้นฐาน

```crystal
# ARGV เป็น Array(String) ของ command line args
# เช่น: ./myapp arg1 arg2 --flag
puts ARGV.inspect

# ตรวจสอบ args
if ARGV.empty?
  puts "No arguments provided"
  exit 0
end

# ดูแต่ละ arg
ARGV.each_with_index do |arg, i|
  puts "Arg #{i}: #{arg}"
end

# ตรวจสอบ flag
has_verbose = ARGV.includes?("--verbose")
has_help = ARGV.includes?("--help") || ARGV.includes?("-h")

# ดึง value หลัง flag
def get_flag_value(args : Array(String), flag : String) : String?
  idx = args.index(flag)
  idx && idx + 1 < args.size ? args[idx + 1] : nil
end

port = get_flag_value(ARGV, "--port").try(&.to_i) || 8080

# Simple manual parsing
args = ARGV.dup
while arg = args.shift?
  case arg
  when "--help", "-h"
    puts "Usage: myapp [options]"
    exit 0
  when "--verbose", "-v"
    puts "Verbose mode"
  when "--port"
    port = args.shift?.try(&.to_i) || 8080
  end
end
```

---

## 2. OptionParser

```crystal
require "option_parser"

# OptionParser พื้นฐาน
options = {
  host: "localhost",
  port: 8080,
  verbose: false,
  output: nil.as(String?),
}

parser = OptionParser.new do |p|
  p.banner = "Usage: myapp [options]"
  
  p.on("-h", "--help", "Show this help") do
    puts p
    exit 0
  end
  
  p.on("-v", "--verbose", "Enable verbose output") do
    options[:verbose] = true
  end
  
  p.on("--host HOST", "Server host (default: localhost)") do |host|
    options[:host] = host
  end
  
  p.on("--port PORT", "Server port (default: 8080)") do |port|
    options[:port] = port.to_i
  end
  
  p.on("-o FILE", "--output FILE", "Output file") do |file|
    options[:output] = file
  end
  
  p.invalid_option do |flag|
    STDERR.puts "Error: Unknown option #{flag}"
    STDERR.puts p
    exit 1
  end
end

parser.parse

puts "Host: #{options[:host]}"
puts "Port: #{options[:port]}"
puts "Verbose: #{options[:verbose]}"
puts "Output: #{options[:output].inspect}"
```

---

## 3. Defining Flags

```crystal
require "option_parser"

# Flag types

# Boolean flags
debug = false
OptionParser.parse do |p|
  p.on("--debug", "Enable debug mode") { debug = true }
  p.on("--no-debug", "Disable debug mode") { debug = false }
end

# String flags
name = "world"
OptionParser.parse do |p|
  p.on("-n NAME", "--name NAME", "Your name") { |n| name = n }
end

# Integer flags
count = 1
OptionParser.parse do |p|
  p.on("-c COUNT", "--count COUNT", "Number of times") do |n|
    count = n.to_i? || (STDERR.puts "Invalid count: #{n}"; exit 1)
  end
end

# Float flags
rate = 1.0
OptionParser.parse do |p|
  p.on("-r RATE", "--rate RATE", "Processing rate") do |r|
    rate = r.to_f? || (STDERR.puts "Invalid rate: #{r}"; exit 1)
  end
end

# Array flags (สะสม values)
tags = [] of String
OptionParser.parse do |p|
  p.on("-t TAG", "--tag TAG", "Add a tag (can use multiple times)") do |tag|
    tags << tag
  end
end

# Enum-like flags
format = "text"
valid_formats = ["text", "json", "csv", "xml"]
OptionParser.parse do |p|
  p.on("-f FORMAT", "--format FORMAT", "Output format: #{valid_formats.join(", ")}") do |f|
    unless valid_formats.includes?(f)
      STDERR.puts "Invalid format: #{f}. Valid: #{valid_formats.join(", ")}"
      exit 1
    end
    format = f
  end
end
```

---

## 4. Subcommands

```crystal
require "option_parser"

# Subcommand pattern
command = ""
global_verbose = false

parser = OptionParser.new do |p|
  p.banner = """
    Usage: tool [global options] COMMAND [command options]
    
    Commands:
      init    Initialize a new project
      build   Build the project
      test    Run tests
      deploy  Deploy the project
    """
  
  p.on("-v", "--verbose", "Verbose output") { global_verbose = true }
  p.on("-h", "--help", "Show this help") { puts p; exit 0 }
  
  p.unknown_args do |args|
    command = args.first? || ""
  end
end

parser.parse

case command
when "init"
  # Parse init-specific options
  project_name = "myproject"
  template = "default"
  
  OptionParser.parse(ARGV[1..]) do |p|
    p.banner = "Usage: tool init [options] [name]"
    p.on("-t TEMPLATE", "--template TEMPLATE", "Project template") { |t| template = t }
    p.on("-h", "--help", "Show help") { puts p; exit 0 }
    p.unknown_args do |args|
      project_name = args.first? || project_name
    end
  end
  
  puts "Initializing #{project_name} with template #{template}"
  
when "build"
  release = false
  target = nil.as(String?)
  
  OptionParser.parse(ARGV[1..]) do |p|
    p.banner = "Usage: tool build [options]"
    p.on("--release", "Build in release mode") { release = true }
    p.on("--target TARGET", "Target architecture") { |t| target = t }
    p.on("-h", "--help", "Show help") { puts p; exit 0 }
  end
  
  puts "Building #{release ? "release" : "debug"} #{target ? "for #{target}" : ""}"
  
when "test"
  test_file = nil.as(String?)
  watch = false
  
  OptionParser.parse(ARGV[1..]) do |p|
    p.banner = "Usage: tool test [options] [file]"
    p.on("-w", "--watch", "Watch for changes") { watch = true }
    p.on("-h", "--help", "Show help") { puts p; exit 0 }
    p.unknown_args { |args| test_file = args.first? }
  end
  
  puts "Testing #{test_file || "all"} #{watch ? "(watching)" : ""}"
  
when ""
  puts "No command specified. Use --help for usage."
  exit 1
else
  puts "Unknown command: #{command}"
  exit 1
end
```

---

## 5. Help Text

```crystal
require "option_parser"

# Help text ที่ดี
parser = OptionParser.new do |p|
  p.banner = <<-BANNER
    Crystal File Processor v1.0.0
    
    Usage:
      process [options] FILE [FILE...]
      process --help
    
    Description:
      Process one or more files with various transformations.
    BANNER
  
  p.separator "\nOptions:"
  
  p.on("-v", "--verbose", "Enable verbose output") {}
  p.on("-d", "--dry-run", "Show what would be done without doing it") {}
  p.on("-q", "--quiet", "Suppress all output") {}
  
  p.separator "\nOutput Options:"
  
  p.on("-o FILE", "--output FILE", "Write output to FILE") {}
  p.on("-f FORMAT", "--format FORMAT", "Output format: text, json, csv (default: text)") {}
  p.on("--encoding ENC", "Output encoding (default: utf-8)") {}
  
  p.separator "\nFilter Options:"
  
  p.on("-i PATTERN", "--include PATTERN", "Include files matching PATTERN") {}
  p.on("-e PATTERN", "--exclude PATTERN", "Exclude files matching PATTERN") {}
  p.on("--min-size SIZE", "Minimum file size in bytes") {}
  p.on("--max-size SIZE", "Maximum file size in bytes") {}
  
  p.separator "\nExamples:"
  p.separator "  process file.txt                  # Process single file"
  p.separator "  process -o out.json -f json *.txt  # Process all txt to json"
  p.separator "  process --dry-run *.csv            # Preview processing"
  
  p.separator "\nFor more information, visit https://example.com"
  
  p.on("-h", "--help", "Show this help message") do
    puts p
    exit 0
  end
  
  p.on("--version", "Show version") do
    puts "process 1.0.0"
    exit 0
  end
end

parser.parse
puts "Help:"
puts parser
```

---

## 6. Required/Optional Arguments

```crystal
require "option_parser"

# ตรวจสอบ required arguments
class CLI
  property input_file : String?
  property output_file : String?
  property format : String = "text"
  property verbose : Bool = false
  
  def parse!
    parser = OptionParser.new do |p|
      p.banner = "Usage: process -i INPUT [options]"
      
      p.on("-i FILE", "--input FILE", "Input file (required)") do |f|
        @input_file = f
      end
      
      p.on("-o FILE", "--output FILE", "Output file (optional)") do |f|
        @output_file = f
      end
      
      p.on("-f FORMAT", "--format FORMAT", "Format: text, json") do |f|
        @format = f
      end
      
      p.on("-v", "--verbose", "Verbose") { @verbose = true }
      
      p.on("-h", "--help", "Help") { puts p; exit 0 }
      
      p.invalid_option do |opt|
        STDERR.puts "Unknown option: #{opt}"
        STDERR.puts p
        exit 1
      end
    end
    
    parser.parse
    
    # Validate required
    unless @input_file
      STDERR.puts "Error: --input is required"
      exit 1
    end
    
    # Validate values
    unless ["text", "json", "csv"].includes?(@format)
      STDERR.puts "Error: Invalid format '#{@format}'"
      exit 1
    end
    
    # Validate file exists
    if input = @input_file
      unless File.exists?(input)
        STDERR.puts "Error: Input file not found: #{input}"
        exit 1
      end
    end
    
    self
  end
end

cli = CLI.new.parse!
puts "Processing #{cli.input_file} as #{cli.format}"
```

---

## 7. Practical CLI App

```crystal
require "option_parser"

# ตัวอย่างจริง: file statistics tool
module FileStat
  VERSION = "1.0.0"
  
  struct Options
    property files : Array(String) = [] of String
    property recursive : Bool = false
    property format : String = "table"
    property sort_by : String = "name"
    property show_hidden : Bool = false
    property min_size : Int64? = nil
    property max_size : Int64? = nil
  end
  
  def self.run(args : Array(String) = ARGV)
    opts = Options.new
    
    parser = OptionParser.new do |p|
      p.banner = "Usage: filestat [options] [FILE...]\n\nShow file statistics"
      
      p.on("-r", "--recursive", "Process directories recursively") do
        opts.recursive = true
      end
      
      p.on("-f FORMAT", "--format FORMAT", "Output format: table, json, csv") do |f|
        opts.format = f
      end
      
      p.on("-s FIELD", "--sort FIELD", "Sort by: name, size, mtime") do |s|
        opts.sort_by = s
      end
      
      p.on("-a", "--all", "Show hidden files") { opts.show_hidden = true }
      
      p.on("--min-size SIZE", "Minimum file size (e.g., 1k, 1m)") do |s|
        opts.min_size = parse_size(s)
      end
      
      p.on("--max-size SIZE", "Maximum file size") do |s|
        opts.max_size = parse_size(s)
      end
      
      p.on("-v", "--version", "Show version") do
        puts VERSION
        exit 0
      end
      
      p.on("-h", "--help", "Show help") do
        puts p
        exit 0
      end
      
      p.unknown_args { |a| opts.files.concat(a) }
    end
    
    parser.parse(args)
    
    # Default to current directory
    opts.files << "." if opts.files.empty?
    
    # Process and display
    files = collect_files(opts)
    display(files, opts)
  end
  
  private def self.parse_size(size_str : String) : Int64
    mult = case size_str.downcase[-1..]
    when "k" then 1024i64
    when "m" then 1024i64 * 1024
    when "g" then 1024i64 * 1024 * 1024
    else 1i64
    end
    
    num = size_str.downcase.rstrip("kmg")
    (num.to_f64 * mult).to_i64
  rescue
    size_str.to_i64
  end
  
  private def self.collect_files(opts : Options) : Array({String, Int64, Time})
    files = [] of {String, Int64, Time}
    
    opts.files.each do |path|
      if Dir.exists?(path) && opts.recursive
        Dir.glob("#{path}/**/*") do |f|
          next if File.directory?(f)
          next if !opts.show_hidden && File.basename(f).starts_with?(".")
          size = File.size(f)
          mtime = File.info(f).modification_time
          next if (min = opts.min_size) && size < min
          next if (max = opts.max_size) && size > max
          files << {f, size, mtime}
        end
      elsif File.exists?(path)
        size = File.size(path)
        mtime = File.info(path).modification_time
        files << {path, size, mtime}
      end
    end
    
    files.sort_by do |f, size, mtime|
      case opts.sort_by
      when "size" then [-size, f]
      when "mtime" then [mtime.to_unix * -1i64, f.hash.to_i64]
      else [0i64, f.hash.to_i64]
      end
    end
  end
  
  private def self.display(files : Array({String, Int64, Time}), opts : Options)
    case opts.format
    when "json"
      puts "["
      files.each_with_index do |(path, size, mtime), i|
        comma = i < files.size - 1 ? "," : ""
        puts %<  {"path": "#{path}", "size": #{size}, "mtime": "#{mtime.to_s("%Y-%m-%dT%H:%M:%S")}"}#{comma}>
      end
      puts "]"
    when "csv"
      puts "path,size,mtime"
      files.each do |path, size, mtime|
        puts "#{path},#{size},#{mtime.to_s("%Y-%m-%d %H:%M:%S")}"
      end
    else
      puts "%-40s %10s  %s" % {"Path", "Size", "Modified"}
      puts "-" * 65
      files.each do |path, size, mtime|
        size_str = format_bytes(size)
        puts "%-40s %10s  %s" % {path, size_str, mtime.to_s("%Y-%m-%d %H:%M:%S")}
      end
      puts "\nTotal: #{files.size} files"
    end
  end
  
  private def self.format_bytes(n : Int64) : String
    if n < 1024
      "#{n} B"
    elsif n < 1_048_576
      "#{(n / 1024.0).round(1)} KB"
    else
      "#{(n / 1_048_576.0).round(1)} MB"
    end
  end
end

FileStat.run
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง CLI tool สำหรับ convert text case: upper, lower, title, snake, camel

### แบบฝึกหัดที่ 2
สร้าง `find` command ที่รองรับ `--name`, `--type`, `--size`, `--newer` flags

### เฉลย

```crystal
# แบบฝึกหัดที่ 1: Text case converter
require "option_parser"

def to_snake_case(text : String) : String
  text.gsub(/([A-Z])/) { "_#{$~[1].downcase}" }
      .gsub(/[\s-]+/, "_")
      .downcase
      .lstrip("_")
end

def to_camel_case(text : String) : String
  text.split(/[\s_-]+/)
      .each_with_index
      .map { |word, i| i == 0 ? word.downcase : word.capitalize }
      .join
end

def to_pascal_case(text : String) : String
  text.split(/[\s_-]+/).map(&.capitalize).join
end

case_type = "upper"
input_text = nil.as(String?)

OptionParser.parse do |p|
  p.banner = "Usage: caseconv [options] TEXT"
  
  p.on("--upper", "Convert to UPPERCASE") { case_type = "upper" }
  p.on("--lower", "Convert to lowercase") { case_type = "lower" }
  p.on("--title", "Convert to Title Case") { case_type = "title" }
  p.on("--snake", "Convert to snake_case") { case_type = "snake" }
  p.on("--camel", "Convert to camelCase") { case_type = "camel" }
  p.on("--pascal", "Convert to PascalCase") { case_type = "pascal" }
  p.on("-h", "--help", "Show help") { puts p; exit 0 }
  
  p.unknown_args { |args| input_text = args.join(" ") }
end

text = input_text || (STDIN.tty? ? "" : STDIN.gets_to_end.chomp)

if text.empty?
  STDERR.puts "Error: No input provided"
  exit 1
end

result = case case_type
when "upper"  then text.upcase
when "lower"  then text.downcase
when "title"  then text.split.map(&.capitalize).join(" ")
when "snake"  then to_snake_case(text)
when "camel"  then to_camel_case(text)
when "pascal" then to_pascal_case(text)
else text
end

puts result
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ARGV** - raw command line arguments
2. **OptionParser** - structured argument parsing
3. **Flags** - boolean, string, integer, array flags
4. **Subcommands** - nested command structure
5. **Help text** - formatting clear help messages
6. **Validation** - required args, valid values
7. **Practical CLI** - building real CLI applications

OptionParser เป็น tool ที่ทรงพลังสำหรับสร้าง CLI applications ที่ user-friendly ใน Crystal
