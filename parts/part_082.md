# ตอนที่ 82: YAML ใน Crystal

## บทนำ

YAML (YAML Ain't Markup Language) เป็นรูปแบบการเก็บข้อมูลที่อ่านง่ายสำหรับมนุษย์ นิยมใช้กับ configuration files, data serialization และการแลกเปลี่ยนข้อมูล Crystal มี built-in support สำหรับ YAML ผ่าน `require "yaml"`

---

## 1. การ require "yaml"

```crystal
require "yaml"
```

---

## 2. YAML.parse - การแปลง YAML String เป็น Object

`YAML.parse` แปลง YAML string เป็น `YAML::Any` object:

```crystal
require "yaml"

yaml_string = <<-YAML
  name: สมชาย
  age: 30
  city: กรุงเทพ
  YAML

data = YAML.parse(yaml_string)
puts data          # => {"name" => "สมชาย", "age" => 30, "city" => "กรุงเทพ"}
puts data.class    # => YAML::Any
```

### YAML รูปแบบต่างๆ:

```crystal
require "yaml"

# Block style (แบบ indented)
block_yaml = <<-YAML
  person:
    name: มานี
    age: 25
    hobbies:
      - อ่านหนังสือ
      - วาดรูป
      - ดนตรี
  YAML

# Flow style (แบบ inline)
flow_yaml = "{name: มานี, age: 25}"

data1 = YAML.parse(block_yaml)
data2 = YAML.parse(flow_yaml)

puts data1["person"]["name"]  # => มานี
puts data2["name"]            # => มานี
```

---

## 3. YAML::Any - ประเภทข้อมูลสำหรับ YAML

`YAML::Any` คล้ายกับ `JSON::Any` - เป็น union type สำหรับแทนค่า YAML ใดๆ:

```crystal
require "yaml"

yaml = <<-YAML
  name: สมชาย
  age: 30
  score: 98.5
  active: true
  tags:
    - crystal
    - programming
  address:
    city: กรุงเทพ
    zip: "10110"
  YAML

data = YAML.parse(yaml)

# การแปลงประเภทข้อมูล
name   = data["name"].as_s        # String
age    = data["age"].as_i         # Int32 (หรือ Int64)
score  = data["score"].as_f       # Float64
active = data["active"].as_bool   # Bool
tags   = data["tags"].as_a        # Array(YAML::Any)
city   = data["address"]["city"].as_s

puts "ชื่อ: #{name} (#{name.class})"
puts "อายุ: #{age} (#{age.class})"
puts "คะแนน: #{score}"
puts "สถานะ: #{active}"
puts "แท็ก: #{tags.map(&.as_s).join(", ")}"
puts "เมือง: #{city}"
```

### ตาราง as_* methods สำหรับ YAML::Any:

| method    | Crystal type  | YAML value example |
|-----------|--------------|---------------------|
| as_s      | String       | name: John          |
| as_i      | Int32        | age: 30             |
| as_i64    | Int64        | big: 9999999999     |
| as_f      | Float64      | score: 3.14         |
| as_bool   | Bool         | active: true        |
| as_nil    | Nil          | value: ~            |
| as_a      | Array(Any)   | - item1 \n - item2  |
| as_h      | Hash(Any, Any) | key: value        |

---

## 4. to_yaml - การแปลง Crystal Object เป็น YAML String

```crystal
require "yaml"

# ประเภทพื้นฐาน
puts 42.to_yaml          # => --- \n42\n
puts "hello".to_yaml     # => --- \nhello\n
puts true.to_yaml        # => --- \ntrue\n
puts nil.to_yaml         # => --- \n\n

# Array
arr = [1, 2, 3, "สวัสดี"]
puts arr.to_yaml
# ---
# - 1
# - 2
# - 3
# - สวัสดี

# Hash
hash = {"name" => "มานี", "age" => 25}
puts hash.to_yaml
# ---
# name: มานี
# age: 25
```

---

## 5. from_yaml - การแปลง YAML String เป็น Crystal Type

```crystal
require "yaml"

# แปลง YAML เป็น Array(String)
yaml_list = "- apple\n- banana\n- cherry"
fruits = Array(String).from_yaml(yaml_list)
puts fruits.join(", ")  # => apple, banana, cherry

# แปลง YAML เป็น Hash
yaml_hash = "name: สมชาย\nage: 30"
# Hash ต้องการ explicit type
```

---

## 6. YAML::Serializable - Annotation

`YAML::Serializable` ทำงานคล้ายกับ `JSON::Serializable`:

```crystal
require "yaml"

struct DatabaseConfig
  include YAML::Serializable

  property host : String
  property port : Int32
  property database : String
  property username : String
  property password : String
  property ssl : Bool = false
end

yaml = <<-YAML
  host: localhost
  port: 5432
  database: myapp_db
  username: admin
  password: secret123
  ssl: true
  YAML

config = DatabaseConfig.from_yaml(yaml)
puts config.host      # => localhost
puts config.port      # => 5432
puts config.database  # => myapp_db
puts config.ssl       # => true

# แปลงกลับเป็น YAML
puts config.to_yaml
```

### YAML::Field annotation:

```crystal
require "yaml"

struct ServerConfig
  include YAML::Serializable

  @[YAML::Field(key: "server_name")]
  property name : String

  @[YAML::Field(key: "listen_port")]
  property port : Int32

  @[YAML::Field(ignore: true)]
  property internal_token : String = ""

  property workers : Int32 = 4
end

yaml = <<-YAML
  server_name: MyWebApp
  listen_port: 8080
  workers: 8
  YAML

server = ServerConfig.from_yaml(yaml)
puts server.name     # => MyWebApp
puts server.port     # => 8080
puts server.workers  # => 8
```

---

## 7. Config Files กับ YAML - ตัวอย่างจริง

การใช้ YAML สำหรับ application configuration ซึ่งเป็นการใช้งานที่พบบ่อยที่สุด:

```crystal
require "yaml"

struct AppConfig
  include YAML::Serializable

  property app : AppSection
  property database : DatabaseSection
  property cache : CacheSection
  property logging : LoggingSection
end

struct AppSection
  include YAML::Serializable

  property name : String
  property version : String
  property environment : String
  property secret_key : String
  property allowed_hosts : Array(String)
end

struct DatabaseSection
  include YAML::Serializable

  property adapter : String
  property host : String
  property port : Int32
  property name : String
  property pool_size : Int32 = 5
end

struct CacheSection
  include YAML::Serializable

  property enabled : Bool
  property backend : String
  property ttl : Int32
end

struct LoggingSection
  include YAML::Serializable

  property level : String
  property file : String?
  property format : String
end

config_yaml = <<-YAML
  app:
    name: CrystalShop
    version: "2.1.0"
    environment: production
    secret_key: "abc123xyz"
    allowed_hosts:
      - crystalshop.com
      - www.crystalshop.com
      - api.crystalshop.com
  
  database:
    adapter: postgresql
    host: db.crystalshop.com
    port: 5432
    name: crystalshop_prod
    pool_size: 20
  
  cache:
    enabled: true
    backend: redis
    ttl: 3600
  
  logging:
    level: info
    file: /var/log/crystalshop.log
    format: json
  YAML

config = AppConfig.from_yaml(config_yaml)

puts "แอปพลิเคชัน: #{config.app.name} v#{config.app.version}"
puts "Environment: #{config.app.environment}"
puts "Database: #{config.database.adapter}://#{config.database.host}:#{config.database.port}/#{config.database.name}"
puts "Cache enabled: #{config.cache.enabled} (TTL: #{config.cache.ttl}s)"
puts "Allowed hosts: #{config.app.allowed_hosts.join(", ")}"
```

---

## 8. Multi-document YAML

YAML รองรับหลาย documents ในไฟล์เดียวกัน โดยแยกด้วย `---`:

```crystal
require "yaml"

multi_yaml = <<-YAML
  ---
  name: document_one
  version: 1
  ---
  name: document_two
  version: 2
  ---
  name: document_three
  version: 3
  YAML

# YAML::Each_document ใช้สำหรับอ่าน multi-document
# วิธีแบบ manual
documents = [] of YAML::Any
remaining = multi_yaml

# แยก documents ด้วย --- separator
parts = multi_yaml.split(/^---\s*$/m).reject(&.strip.empty?)

parts.each do |part|
  doc = YAML.parse(part.strip)
  documents << doc
end

documents.each_with_index do |doc, i|
  puts "Document #{i + 1}: #{doc["name"].as_s} (v#{doc["version"].as_i})"
end
```

### การใช้ YAML::Any.each_document_from_yaml:

```crystal
require "yaml"

# อีกวิธีในการอ่าน multi-document YAML
yaml_content = <<-YAML
  --- # Production config
  environment: production
  debug: false
  workers: 16
  ---
  environment: development
  debug: true
  workers: 2
  YAML

# ใช้ YAML.parse_all สำหรับ multi-document
YAML.parse_all(yaml_content) do |doc|
  env     = doc["environment"].as_s
  debug   = doc["debug"].as_bool
  workers = doc["workers"].as_i
  puts "#{env}: debug=#{debug}, workers=#{workers}"
end
```

---

## 9. Anchors และ Aliases - การนำค่ากลับมาใช้ใหม่

YAML มีฟีเจอร์ anchor (`&`) และ alias (`*`) สำหรับนำค่ากลับมาใช้ซ้ำ:

```crystal
require "yaml"

yaml_with_anchors = <<-YAML
  # กำหนด anchor
  defaults: &defaults
    timeout: 30
    retries: 3
    ssl: true
  
  production:
    <<: *defaults  # merge defaults
    host: prod.example.com
    port: 443
  
  staging:
    <<: *defaults  # merge defaults
    host: staging.example.com
    port: 8443
    ssl: false  # override ssl
  
  development:
    host: localhost
    port: 3000
    timeout: 60
    retries: 1
    ssl: false
  YAML

config = YAML.parse(yaml_with_anchors)

# Crystal YAML parser จัดการ anchors/aliases อัตโนมัติ
prod    = config["production"]
staging = config["staging"]
dev     = config["development"]

puts "Production:"
puts "  host: #{prod["host"].as_s}"
puts "  timeout: #{prod["timeout"].as_i}"
puts "  ssl: #{prod["ssl"].as_bool}"

puts "Staging:"
puts "  host: #{staging["host"].as_s}"
puts "  ssl: #{staging["ssl"].as_bool}"  # overridden to false

puts "Development:"
puts "  host: #{dev["host"].as_s}"
puts "  timeout: #{dev["timeout"].as_i}"
```

### Anchor ใน arrays:

```crystal
require "yaml"

yaml = <<-YAML
  base_packages: &base
    - crystal
    - yaml
    - json
  
  development_packages:
    - *base
    - spec
    - ameba
  
  production_packages:
    - *base
    - kemal
  YAML

config = YAML.parse(yaml)

# อ่าน array ของ packages
base_pkgs = config["base_packages"].as_a.map(&.as_s)
puts "Base: #{base_pkgs.join(", ")}"

# development_packages มี array ใน array จาก alias
dev_pkgs = config["development_packages"].as_a
dev_pkgs.each do |pkg|
  # แต่ละ element อาจเป็น String หรือ Array
  if pkg.as_s?
    print "#{pkg.as_s} "
  elsif pkg.as_a?
    pkg.as_a.each { |p| print "#{p.as_s} " }
  end
end
puts ""
```

---

## 10. Complex Types ใน YAML

### Nested structs:

```crystal
require "yaml"

struct Permission
  include YAML::Serializable

  property read : Bool
  property write : Bool
  property delete : Bool = false
end

struct Role
  include YAML::Serializable

  property name : String
  property level : Int32
  property permissions : Permission
  property allowed_routes : Array(String)
end

struct AccessControl
  include YAML::Serializable

  property roles : Array(Role)
end

yaml = <<-YAML
  roles:
    - name: admin
      level: 100
      permissions:
        read: true
        write: true
        delete: true
      allowed_routes:
        - /admin
        - /users
        - /settings
        - /reports
    
    - name: editor
      level: 50
      permissions:
        read: true
        write: true
        delete: false
      allowed_routes:
        - /posts
        - /media
        - /comments
    
    - name: viewer
      level: 10
      permissions:
        read: true
        write: false
      allowed_routes:
        - /posts
        - /media
  YAML

acl = AccessControl.from_yaml(yaml)

acl.roles.each do |role|
  puts "Role: #{role.name} (Level #{role.level})"
  puts "  Read: #{role.permissions.read}"
  puts "  Write: #{role.permissions.write}"
  puts "  Delete: #{role.permissions.delete}"
  puts "  Routes: #{role.allowed_routes.size} routes"
end
```

### YAML กับ Union types:

```crystal
require "yaml"

# YAML::Serializable รองรับ union types
struct FlexValue
  include YAML::Serializable

  property name : String
  property value : Int32 | Float64 | String | Bool | Nil
end

yaml_examples = [
  "name: integer\nvalue: 42",
  "name: float\nvalue: 3.14",
  "name: string\nvalue: hello",
  "name: boolean\nvalue: true",
  "name: null\nvalue: ~",
]

yaml_examples.each do |yaml|
  fv = FlexValue.from_yaml(yaml)
  puts "#{fv.name}: #{fv.value.inspect} (#{fv.value.class})"
end
```

---

## 11. YAML กับ Time และ Date

YAML รองรับ timestamp format:

```crystal
require "yaml"
require "time"

yaml = <<-YAML
  event: Crystal Conference
  date: 2024-03-15
  start_time: 2024-03-15T09:00:00+07:00
  duration: 480  # minutes
  YAML

data = YAML.parse(yaml)

event = data["event"].as_s
puts "งาน: #{event}"

# YAML date/time string ต้องแปลงเอง
date_str = data["date"].as_s
date = Time.parse!(date_str, "%Y-%m-%d", Time::Location::UTC)
puts "วันที่: #{date.day}/#{date.month}/#{date.year}"

# ISO 8601 timestamp
time_str = data["start_time"].as_s
# time = Time.parse_iso8601(time_str)
puts "เวลาเริ่ม: #{time_str}"
```

---

## 12. การอ่านและเขียน YAML File

```crystal
require "yaml"

struct Contact
  include YAML::Serializable

  property name : String
  property email : String
  property phone : String?
  property tags : Array(String) = [] of String
end

struct AddressBook
  include YAML::Serializable

  property contacts : Array(Contact)
  property last_updated : String
end

# สร้าง address book
book = AddressBook.new
book.contacts = [
  Contact.new.tap { |c|
    c.name = "สมชาย ใจดี"
    c.email = "somchai@example.com"
    c.phone = "081-234-5678"
    c.tags = ["friend", "colleague"]
  },
  Contact.new.tap { |c|
    c.name = "มานี สวยงาม"
    c.email = "mani@example.com"
    c.tags = ["family"]
  }
]
book.last_updated = Time.local.to_s("%Y-%m-%d")

# เขียนเป็น YAML
yaml_content = book.to_yaml
puts yaml_content

# จำลองการเขียนไฟล์
# File.write("contacts.yaml", yaml_content)

# จำลองการอ่านไฟล์
# loaded_book = AddressBook.from_yaml(File.read("contacts.yaml"))
loaded_book = AddressBook.from_yaml(yaml_content)

puts "\nผู้ติดต่อทั้งหมด:"
loaded_book.contacts.each do |contact|
  puts "  #{contact.name} - #{contact.email}"
  puts "    แท็ก: #{contact.tags.join(", ")}" unless contact.tags.empty?
end
```

---

## 13. YAML Merge Keys

YAML รองรับ merge key (`<<`) สำหรับ merge maps:

```crystal
require "yaml"

yaml = <<-YAML
  # Base configuration
  _base: &base_config
    max_connections: 100
    timeout: 30
    retry_count: 3
    compress: true
  
  # Service configurations using merge
  api_service:
    <<: *base_config
    host: api.example.com
    port: 8080
    # override timeout
    timeout: 60
  
  auth_service:
    <<: *base_config
    host: auth.example.com
    port: 8081
  
  static_service:
    <<: *base_config
    host: static.example.com
    port: 8082
    # static service doesn't need retries
    retry_count: 0
  YAML

config = YAML.parse(yaml)

["api_service", "auth_service", "static_service"].each do |service|
  svc = config[service]
  puts "#{service}:"
  puts "  host: #{svc["host"].as_s}"
  puts "  timeout: #{svc["timeout"].as_i}"
  puts "  retry_count: #{svc["retry_count"].as_i}"
  puts ""
end
```

---

## 14. Custom Serialization ใน YAML

```crystal
require "yaml"

# Custom YAML serialization สำหรับ Color type
struct Color
  property r : UInt8
  property g : UInt8
  property b : UInt8

  def initialize(@r, @g, @b)
  end

  # แปลง Color เป็น YAML (hex string)
  def to_yaml(yaml : YAML::Nodes::Builder)
    hex = "#%02X%02X%02X" % [@r, @g, @b]
    yaml.scalar hex
  end

  # แปลง YAML เป็น Color
  def self.new(ctx : YAML::ParseContext, node : YAML::Nodes::Node)
    unless node.is_a?(YAML::Nodes::Scalar)
      node.raise "Expected scalar for Color"
    end
    hex = node.value.lstrip('#')
    r = hex[0..1].to_u8(16)
    g = hex[2..3].to_u8(16)
    b = hex[4..5].to_u8(16)
    new(r, g, b)
  end
end

struct Theme
  include YAML::Serializable

  property name : String
  property primary : Color
  property secondary : Color
  property background : Color
end

yaml = <<-YAML
  name: Ocean Theme
  primary: "#0066CC"
  secondary: "#00AAFF"
  background: "#F0F8FF"
  YAML

theme = Theme.from_yaml(yaml)
puts "Theme: #{theme.name}"
puts "Primary: RGB(#{theme.primary.r}, #{theme.primary.g}, #{theme.primary.b})"
puts "Secondary: #{theme.secondary.to_yaml}"

puts theme.to_yaml
```

---

## 15. การเปรียบเทียบ YAML และ JSON

```crystal
require "yaml"
require "json"

# ข้อมูลเดียวกันใน YAML และ JSON
data = {
  "name"    => "Crystal Language",
  "version" => "1.11",
  "features" => ["statically typed", "compiled", "garbage collected"],
  "performance" => {
    "speed"  => "C-like",
    "memory" => "low"
  }
}

yaml_output = data.to_yaml
json_output = data.to_json

puts "=== YAML ==="
puts yaml_output
puts "=== JSON ==="
puts json_output

puts "\nYAML size: #{yaml_output.bytesize} bytes"
puts "JSON size: #{json_output.bytesize} bytes"
```

---

## 16. Environment-based Config Loading

```crystal
require "yaml"

struct Environment
  include YAML::Serializable

  property name : String
  property debug : Bool
  property log_level : String
  property api_url : String
  property workers : Int32
end

struct Environments
  include YAML::Serializable

  property development : Environment
  property staging : Environment
  property production : Environment
end

config_yaml = <<-YAML
  development:
    name: development
    debug: true
    log_level: debug
    api_url: http://localhost:3000
    workers: 2
  
  staging:
    name: staging
    debug: false
    log_level: info
    api_url: https://staging-api.example.com
    workers: 4
  
  production:
    name: production
    debug: false
    log_level: warn
    api_url: https://api.example.com
    workers: 16
  YAML

envs = Environments.from_yaml(config_yaml)

# เลือก environment ตาม ENV variable (จำลอง)
current_env = "development"  # ENV["APP_ENV"]? || "development"

env = case current_env
      when "production" then envs.production
      when "staging"    then envs.staging
      else                   envs.development
      end

puts "Environment: #{env.name}"
puts "Debug: #{env.debug}"
puts "Log Level: #{env.log_level}"
puts "API URL: #{env.api_url}"
puts "Workers: #{env.workers}"
```

---

## 17. YAML กับ Optional Values

```crystal
require "yaml"

struct UserProfile
  include YAML::Serializable

  property username : String
  property email : String
  property bio : String?
  property website : String?
  property social : SocialLinks?
  property preferences : UserPreferences = UserPreferences.new
end

struct SocialLinks
  include YAML::Serializable

  property twitter : String?
  property github : String?
  property linkedin : String?
end

struct UserPreferences
  include YAML::Serializable

  property theme : String = "light"
  property language : String = "th"
  property notifications : Bool = true

  def initialize
  end
end

# User ที่มีข้อมูลครบถ้วน
full_yaml = <<-YAML
  username: somchai
  email: somchai@example.com
  bio: Crystal programmer and coffee enthusiast
  website: https://somchai.dev
  social:
    twitter: "@somchai_dev"
    github: somchai
    linkedin: somchai-jaidee
  preferences:
    theme: dark
    language: th
    notifications: false
  YAML

# User ที่มีข้อมูลน้อย
minimal_yaml = <<-YAML
  username: mani
  email: mani@example.com
  YAML

full_user    = UserProfile.from_yaml(full_yaml)
minimal_user = UserProfile.from_yaml(minimal_yaml)

puts "Full user: #{full_user.username}"
puts "Bio: #{full_user.bio || "ไม่มี bio"}"
puts "Twitter: #{full_user.social.try(&.twitter) || "ไม่มี Twitter"}"
puts "Theme: #{full_user.preferences.theme}"

puts ""
puts "Minimal user: #{minimal_user.username}"
puts "Bio: #{minimal_user.bio || "ไม่มี bio"}"
puts "Theme: #{minimal_user.preferences.theme} (default)"
```

---

## 18. YAML Validation

```crystal
require "yaml"

struct ServerConfig
  include YAML::Serializable

  property host : String
  property port : Int32
  property workers : Int32

  def valid? : Bool
    return false if host.empty?
    return false unless (1..65535).includes?(port)
    return false unless workers > 0
    true
  end

  def errors : Array(String)
    errs = [] of String
    errs << "Host ต้องไม่ว่าง" if host.empty?
    errs << "Port ต้องอยู่ระหว่าง 1-65535" unless (1..65535).includes?(port)
    errs << "Workers ต้องมากกว่า 0" unless workers > 0
    errs
  end
end

good_config = <<-YAML
  host: example.com
  port: 8080
  workers: 4
  YAML

bad_config = <<-YAML
  host: ""
  port: 99999
  workers: -1
  YAML

good = ServerConfig.from_yaml(good_config)
bad  = ServerConfig.from_yaml(bad_config)

puts "Config ดี: #{good.valid?}"

unless bad.valid?
  puts "Config ผิดพลาด:"
  bad.errors.each { |e| puts "  - #{e}" }
end
```

---

## 19. ตัวอย่างโปรแกรมสมบูรณ์ - Deployment Config System

```crystal
require "yaml"

struct ServiceConfig
  include YAML::Serializable

  property name : String
  property image : String
  property tag : String = "latest"
  property replicas : Int32 = 1
  property ports : Array(PortMapping)
  property environment : Hash(String, String) = {} of String => String
  property resources : ResourceLimits?
end

struct PortMapping
  include YAML::Serializable

  property host : Int32
  property container : Int32
  property protocol : String = "tcp"
end

struct ResourceLimits
  include YAML::Serializable

  property cpu : String
  property memory : String
end

struct DeploymentConfig
  include YAML::Serializable

  property version : String
  property project : String
  property services : Array(ServiceConfig)
end

deployment_yaml = <<-YAML
  version: "1.0"
  project: my-crystal-app
  services:
    - name: web
      image: crystalapp/web
      tag: "2.1.0"
      replicas: 3
      ports:
        - host: 80
          container: 8080
          protocol: tcp
        - host: 443
          container: 8443
          protocol: tcp
      environment:
        APP_ENV: production
        LOG_LEVEL: info
        MAX_CONNECTIONS: "100"
      resources:
        cpu: "0.5"
        memory: 512M
    
    - name: worker
      image: crystalapp/worker
      tag: "2.1.0"
      replicas: 2
      ports: []
      environment:
        APP_ENV: production
        QUEUE_CONCURRENCY: "4"
      resources:
        cpu: "1.0"
        memory: 1G
    
    - name: redis
      image: redis
      tag: "7.0-alpine"
      replicas: 1
      ports:
        - host: 6379
          container: 6379
  YAML

deployment = DeploymentConfig.from_yaml(deployment_yaml)

puts "=== Deployment: #{deployment.project} v#{deployment.version} ==="
deployment.services.each do |svc|
  puts ""
  puts "Service: #{svc.name}"
  puts "  Image: #{svc.image}:#{svc.tag}"
  puts "  Replicas: #{svc.replicas}"

  unless svc.ports.empty?
    puts "  Ports:"
    svc.ports.each { |p| puts "    #{p.host}:#{p.container}/#{p.protocol}" }
  end

  unless svc.environment.empty?
    puts "  Environment variables: #{svc.environment.size}"
  end

  if resources = svc.resources
    puts "  CPU: #{resources.cpu}, Memory: #{resources.memory}"
  end
end
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Application Config Reader
สร้าง struct สำหรับอ่าน configuration ต่อไปนี้:

```yaml
app:
  name: "MyBlog"
  port: 4000
  assets_path: "/public"

database:
  url: "postgres://localhost/myblog"
  pool: 10

mail:
  smtp_host: "smtp.example.com"
  from: "noreply@myblog.com"
  use_tls: true
```

### แบบฝึกหัดที่ 2: Multi-environment Config
สร้างระบบ config ที่รองรับหลาย environment โดยใช้ anchors และ aliases:
1. สร้าง `_defaults` section ด้วย anchor
2. สร้าง `development`, `staging`, `production` sections ที่ inherit จาก defaults
3. เขียนโค้ด Crystal อ่าน config และแสดงผล

### แบบฝึกหัดที่ 3: YAML vs JSON Performance
สร้างโปรแกรมที่:
1. สร้าง data structure ขนาดใหญ่ (1000 items)
2. Serialize เป็นทั้ง YAML และ JSON
3. เปรียบเทียบขนาดและ round-trip ได้ถูกต้อง

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

1. **`require "yaml"`** - การนำเข้า YAML library
2. **`YAML.parse`** - การแปลง YAML string เป็น `YAML::Any`
3. **`YAML::Any`** - type สำหรับแทนค่า YAML และ `as_*` methods
4. **`to_yaml`** - การแปลง Crystal values เป็น YAML string
5. **`from_yaml`** - การแปลง YAML string เป็น Crystal types
6. **`YAML::Serializable`** - annotation สำหรับ auto serialize/deserialize
7. **`@[YAML::Field]`** - การ map ชื่อ field ใน YAML
8. **Config files** - การใช้ YAML สำหรับ application configuration
9. **Multi-document YAML** - การทำงานกับหลาย document ในไฟล์เดียว
10. **Anchors และ Aliases** - การนำค่ากลับมาใช้ซ้ำใน YAML
11. **Complex types** - Nested structs, Union types, Optional values
12. **Merge keys** - การ merge configurations ด้วย `<<`

YAML เหมาะสำหรับ configuration files เพราะอ่านง่ายกว่า JSON และมีฟีเจอร์อย่าง anchors/aliases ที่ช่วยลดความซ้ำซ้อน
