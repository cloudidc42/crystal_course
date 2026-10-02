# Part 76: Environment Variables

## บทนำ

Environment Variables คือ key-value pairs ที่ OS จัดเก็บและ process สามารถ access ได้ ใน Crystal ใช้ `ENV` สำหรับจัดการ environment variables ทั้งการอ่านและเขียน

---

## 1. ENV["KEY"] - อ่าน Environment Variable

```crystal
# อ่าน env var (raise KeyError ถ้าไม่มี)
home = ENV["HOME"]
puts home  # => /home/user

path = ENV["PATH"]
puts path.split(":").first(3).inspect  # first 3 paths

# อ่านด้วย [] (raises KeyError)
begin
  missing = ENV["NONEXISTENT_VAR"]
rescue KeyError => e
  puts "Not found: #{e.message}"
end

# อ่านทั่วๆ ไปที่มักใช้
puts ENV["HOME"]           # home directory
puts ENV["USER"]           # current user
puts ENV["SHELL"]          # current shell
puts ENV["TERM"]? || "unknown"  # terminal type
puts ENV["LANG"]? || "C"        # locale
puts ENV["PWD"]? || Dir.current  # current directory
```

---

## 2. ENV["KEY"]? - Safe Read

```crystal
# อ่านแบบ safe (คืน nil ถ้าไม่มี)
debug = ENV["DEBUG"]?
puts debug.inspect  # => nil ถ้าไม่ set

# ใช้กับ ||
port = ENV["PORT"]?.try(&.to_i) || 8080
puts "Port: #{port}"

# ใช้ใน conditional
if debug = ENV["DEBUG"]?
  puts "Debug mode: #{debug}"
end

unless ENV["PRODUCTION"]?
  puts "Not in production mode"
end

# อ่านหลาย vars
db_config = {
  host: ENV["DB_HOST"]? || "localhost",
  port: (ENV["DB_PORT"]?.try(&.to_i) || 5432),
  name: ENV["DB_NAME"]? || "development",
  user: ENV["DB_USER"]? || "postgres",
  pass: ENV["DB_PASS"]? || "",
}

puts db_config.inspect
```

---

## 3. ENV.fetch

```crystal
# fetch กับ default value
port = ENV.fetch("PORT", "8080")
puts port  # => "8080" ถ้าไม่ set

# fetch กับ block (ถ้าไม่มี key)
secret = ENV.fetch("SECRET_KEY") do
  raise "SECRET_KEY environment variable is required!"
end

# fetch กับ type conversion
port = ENV.fetch("PORT", "8080").to_i
max_connections = ENV.fetch("MAX_CONNECTIONS", "100").to_i

# หลาย default
def env_or_default(key : String, default : String) : String
  ENV.fetch(key, default)
end

puts env_or_default("REDIS_URL", "redis://localhost:6379")
puts env_or_default("LOG_LEVEL", "info")

# Type-safe env vars
class TypedEnv
  def self.string(key : String, default : String? = nil) : String
    ENV[key]? || default || raise "Missing required env var: #{key}"
  end
  
  def self.integer(key : String, default : Int32? = nil) : Int32
    value = ENV[key]?
    return default || raise "Missing required env var: #{key}" unless value
    value.to_i? || raise "Env var #{key}=#{value.inspect} is not a valid integer"
  end
  
  def self.boolean(key : String, default : Bool? = nil) : Bool
    value = ENV[key]?.try(&.downcase)
    case value
    when "true", "1", "yes", "on" then true
    when "false", "0", "no", "off" then false
    when nil
      default || raise "Missing required env var: #{key}"
    else
      raise "Env var #{key}=#{ENV[key]?.inspect} is not a valid boolean"
    end
  end
  
  def self.float(key : String, default : Float64? = nil) : Float64
    value = ENV[key]?
    return default || raise "Missing required env var: #{key}" unless value
    value.to_f64? || raise "Env var #{key}=#{value.inspect} is not a valid float"
  end
  
  def self.list(key : String, separator : String = ",", default : Array(String) = [] of String) : Array(String)
    ENV[key]?.try { |v| v.split(separator).map(&.strip) } || default
  end
end

# ใช้งาน
# ENV["PORT"] = "8080"  # ต้อง set ก่อน
# puts TypedEnv.integer("PORT", 3000)     # => 8080
# puts TypedEnv.boolean("DEBUG", false)   # => false
# puts TypedEnv.list("ALLOWED_HOSTS", ",", ["localhost"])
```

---

## 4. ENV.each - วนลูปผ่าน Env Vars

```crystal
# วนลูปผ่านทั้งหมด
ENV.each do |key, value|
  puts "#{key}=#{value}"
end

# กรองเฉพาะที่ต้องการ
ENV.each do |key, value|
  puts "#{key}=#{value}" if key.starts_with?("APP_")
end

# แปลงเป็น Hash
all_env = {} of String => String
ENV.each { |k, v| all_env[k] = v }

# หา env vars ที่ match pattern
def find_env(pattern : Regex) : Hash(String, String)
  result = {} of String => String
  ENV.each do |key, value|
    result[key] = value if key.match?(pattern)
  end
  result
end

db_vars = find_env(/^DB_/)
puts "Database config:"
db_vars.each { |k, v| puts "  #{k}=#{v}" }

# แสดง PATH แบบ formatted
puts "\nPATH directories:"
ENV.fetch("PATH", "").split(":").each_with_index do |dir, i|
  exists = Dir.exists?(dir)
  puts "  #{i + 1}. #{dir} #{exists ? "✓" : "✗"}"
end
```

---

## 5. Setting Env Vars ใน Process

```crystal
# set env var (ใน current process เท่านั้น)
ENV["MY_VAR"] = "hello"
puts ENV["MY_VAR"]  # => "hello"

# delete env var
ENV.delete("MY_VAR")
puts ENV["MY_VAR"]?.inspect  # => nil

# ตั้งค่า env vars ชั่วคราว
def with_env(vars : Hash(String, String), &block)
  # save originals
  originals = {} of String => String?
  vars.each do |k, _|
    originals[k] = ENV[k]?
  end
  
  # set new values
  vars.each { |k, v| ENV[k] = v }
  
  begin
    block.call
  ensure
    # restore originals
    originals.each do |k, orig|
      if orig
        ENV[k] = orig
      else
        ENV.delete(k)
      end
    end
  end
end

puts ENV["APP_ENV"]?.inspect  # => nil

with_env({"APP_ENV" => "test", "DEBUG" => "true"}) do
  puts ENV["APP_ENV"]  # => "test"
  puts ENV["DEBUG"]    # => "true"
end

puts ENV["APP_ENV"]?.inspect  # => nil (restored)
```

---

## 6. ARGV vs ENV

```crystal
# ARGV: command line arguments
# ENV: environment variables

# ARGV - array ของ command line args
puts ARGV.inspect  # หลังจาก -- ทุกอย่าง

# ใช้ ARGV สำหรับ subcommand
if ARGV.empty?
  puts "Usage: myapp [command] [options]"
  exit 1
end

command = ARGV[0]

case command
when "start"
  puts "Starting..."
when "stop"
  puts "Stopping..."
when "status"
  puts "Running"
else
  puts "Unknown command: #{command}"
  exit 1
end

# ARGV กับ flags
def parse_simple_args(args : Array(String)) : Hash(String, String | Bool)
  result = {} of String => String | Bool
  i = 0
  
  while i < args.size
    arg = args[i]
    if arg.starts_with?("--")
      key = arg[2..]
      if i + 1 < args.size && !args[i + 1].starts_with?("-")
        result[key] = args[i + 1]
        i += 2
      else
        result[key] = true
        i += 1
      end
    else
      result["_args"] = (result["_args"]?.as?(String) || "") + " " + arg
      i += 1
    end
  end
  
  result
end

# Config: ENV มีความสำคัญสูงกว่า default
class Config
  def initialize
    @values = {} of String => String
  end
  
  def load_defaults(defaults : Hash(String, String))
    @values.merge!(defaults)
  end
  
  def load_env(prefix : String = "APP_")
    ENV.each do |key, value|
      if key.starts_with?(prefix)
        config_key = key[prefix.size..].downcase
        @values[config_key] = value
      end
    end
  end
  
  def [](key : String) : String
    @values[key]? || raise KeyError.new("Config key not found: #{key}")
  end
  
  def []?(key : String) : String?
    @values[key]?
  end
  
  def to_h : Hash(String, String)
    @values.dup
  end
end

config = Config.new
config.load_defaults({
  "host" => "localhost",
  "port" => "8080",
  "debug" => "false",
})
config.load_env("APP_")  # APP_HOST, APP_PORT, etc.

puts config["host"]
puts config["port"]
```

---

## 7. Dotenv สไตล์

```crystal
# โหลด .env file (ถ้ามี)
def load_dotenv(path : String = ".env") : Int32
  return 0 unless File.exists?(path)
  
  count = 0
  File.each_line(path) do |line|
    # Skip comments and empty lines
    line = line.strip
    next if line.empty? || line.starts_with?("#")
    
    # Parse KEY=VALUE
    if line =~ /^([A-Za-z_]\w*)=(.*)$/
      key = $~[1]
      value = $~[2].strip
      
      # Remove quotes if present
      if (value.starts_with?('"') && value.ends_with?('"')) ||
         (value.starts_with?("'") && value.ends_with?("'"))
        value = value[1..-2]
        # Handle escape sequences in double-quoted values
        value = value.gsub("\\n", "\n").gsub("\\t", "\t")
      end
      
      # Only set if not already in environment
      ENV[key] = value unless ENV[key]?
      count += 1
    end
  end
  
  count
end

# สร้าง .env file ตัวอย่าง
File.write(".env.example", <<-ENV)
  # Application
  APP_NAME=MyApp
  APP_ENV=development
  APP_PORT=8080
  
  # Database
  DB_HOST=localhost
  DB_PORT=5432
  DB_NAME=myapp_dev
  DB_USER=postgres
  DB_PASS=secret
  
  # Secret keys
  SECRET_KEY=change-me-in-production
  JWT_SECRET=another-secret
  ENV

# โหลด
loaded = load_dotenv(".env")
puts "Loaded #{loaded} vars from .env"

# หรือโหลด .env.test สำหรับ testing
load_dotenv(".env.test") if ENV["APP_ENV"]? == "test"

# cleanup
File.delete(".env.example")
```

---

## 8. Secure Env Vars

```crystal
# Masking sensitive values ใน logs
def mask_sensitive(value : String, show_chars : Int32 = 4) : String
  return "*" * value.size if value.size <= show_chars
  value[0, show_chars] + "*" * (value.size - show_chars)
end

SENSITIVE_KEYS = ["password", "secret", "key", "token", "auth"]

def log_config(config : Hash(String, String))
  config.each do |key, value|
    if SENSITIVE_KEYS.any? { |s| key.downcase.includes?(s) }
      puts "#{key}=#{mask_sensitive(value)}"
    else
      puts "#{key}=#{value}"
    end
  end
end

sample_config = {
  "APP_NAME" => "MyApp",
  "DB_HOST" => "localhost",
  "DB_PASS" => "supersecret123",
  "API_KEY" => "sk-1234567890abcdef",
  "PORT" => "8080",
}

log_config(sample_config)
# => APP_NAME=MyApp
# => DB_HOST=localhost
# => DB_PASS=supe************
# => API_KEY=sk-1***************
# => PORT=8080

# ตรวจสอบ required env vars
def require_env_vars(vars : Array(String)) : Hash(String, String)
  missing = vars.reject { |v| ENV[v]? }
  
  unless missing.empty?
    STDERR.puts "Missing required environment variables:"
    missing.each { |v| STDERR.puts "  - #{v}" }
    exit 1
  end
  
  vars.each_with_object({} of String => String) do |v, h|
    h[v] = ENV[v]
  end
end

# ใช้งาน (ใน production)
# required = require_env_vars([
#   "DATABASE_URL",
#   "SECRET_KEY",
#   "AWS_ACCESS_KEY_ID",
#   "AWS_SECRET_ACCESS_KEY",
# ])
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `ConfigLoader` ที่โหลด config จากหลายแหล่ง (defaults → .env file → env vars → CLI args) โดย priority สูงกว่า override ต่ำกว่า

### แบบฝึกหัดที่ 2
เขียน `EnvValidator` ที่ validate ว่า env vars มีค่าถูกต้อง (type, range, format)

### เฉลย

```crystal
# แบบฝึกหัดที่ 1: ConfigLoader
class ConfigLoader
  def initialize
    @config = {} of String => String
  end
  
  def load_defaults(defaults : Hash(String, String)) : self
    defaults.each { |k, v| @config[k] ||= v }
    self
  end
  
  def load_dotenv(path : String = ".env") : self
    return self unless File.exists?(path)
    File.each_line(path) do |line|
      line = line.strip
      next if line.empty? || line.starts_with?("#")
      if line =~ /^([A-Za-z_]\w*)=(.*)$/
        @config[$~[1]] ||= $~[2].strip.gsub(/^["']|["']$/, "")
      end
    end
    self
  end
  
  def load_env(prefix : String = "") : self
    ENV.each do |key, value|
      config_key = prefix.empty? ? key : (key.starts_with?(prefix) ? key[prefix.size..].downcase : nil)
      @config[config_key] = value if config_key
    end
    self
  end
  
  def load_args(args : Array(String)) : self
    args.each do |arg|
      if arg =~ /^--([a-z-]+)=(.+)$/
        @config[$~[1].gsub("-", "_")] = $~[2]
      end
    end
    self
  end
  
  def get(key : String, default : String? = nil) : String
    @config[key]? || default || raise KeyError.new("Config not found: #{key}")
  end
  
  def to_h : Hash(String, String)
    @config.dup
  end
end

config = ConfigLoader.new
  .load_defaults({"port" => "8080", "host" => "localhost", "debug" => "false"})
  .load_dotenv(".env")
  .load_env("APP_")
  .load_args(ARGV)

puts "Host: #{config.get("host")}"
puts "Port: #{config.get("port")}"
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ENV["KEY"]** - อ่าน env var (raises ถ้าไม่มี)
2. **ENV["KEY"]?** - safe read (คืน nil)
3. **ENV.fetch** - อ่านพร้อม default
4. **ENV.each** - วนลูปผ่านทั้งหมด
5. **Setting** - ENV["KEY"] = value
6. **ARGV vs ENV** - ความแตกต่างและการใช้
7. **Dotenv** - โหลด .env file
8. **Security** - mask sensitive values

Environment variables เป็น best practice สำหรับ configuration management ที่ไม่ต้อง hardcode secrets ใน code
