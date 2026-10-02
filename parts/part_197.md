# Part 197: Crystal Ecosystem และ Community

## บทนำ

Crystal มี ecosystem ที่กำลังเติบโตอย่างรวดเร็ว โดยมี shards (packages) มากกว่า 3,000+ รายการใน shards.info ในบทนี้เราจะสำรวจ shards ยอดนิยม, แหล่งข้อมูล, วิธีการ contribute และเปรียบเทียบ Crystal กับภาษาอื่นๆ

## Popular Web Frameworks

### Kemal - Micro Web Framework

```crystal
# kemal: github.com/kemalcr/kemal
# ง่าย รวดเร็ว คล้าย Sinatra/Express

require "kemal"

# Route พื้นฐาน
get "/" do
  "สวัสดี Crystal!"
end

get "/users/:id" do |env|
  id = env.params.url["id"]
  # ดึงข้อมูลจาก database
  {id: id, name: "ผู้ใช้ #{id}"}.to_json
end

post "/users" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body)
  
  env.response.status_code = 201
  {message: "สร้างผู้ใช้สำเร็จ", data: data}.to_json
end

# Middleware
before_all do |env|
  env.response.headers["X-Powered-By"] = "Crystal/Kemal"
end

# WebSocket
ws "/chat" do |socket|
  socket.on_message do |message|
    socket.send "Echo: #{message}"
  end
end

Kemal.run(port: 3000)
```

### Lucky Framework - Full-Stack Framework

```crystal
# lucky: github.com/luckyframework/lucky
# Full-stack framework คล้าย Rails

# src/actions/users/index.cr
class Users::Index < BrowserAction
  get "/users" do
    users = UserQuery.new.order_by_name
    html IndexPage, users: users
  end
end

# src/actions/users/create.cr
class Users::Create < BrowserAction
  post "/users" do
    SaveUser.create(params) do |operation, user|
      if user
        redirect to: Users::Show.with(user.id)
      else
        html NewPage, operation: operation
      end
    end
  end
end

# src/queries/user_query.cr
class UserQuery < User::BaseQuery
  def order_by_name
    name.asc
  end
  
  def active
    is_active(true)
  end
  
  def admins
    role("admin")
  end
end

# src/operations/save_user.cr
class SaveUser < User::SaveOperation
  permit_columns :name, :email, :role
  
  before_save do
    validate_uniqueness_of email
    validate_format_of email, with: /.+@.+\..+/
    validate_presence_of name
  end
end
```

### Amber Framework

```crystal
# amber: github.com/amberframework/amber
# MVC framework พร้อม ORM

# config/routes.cr
Amber::Server.configure do
  pipeline :web do
    plug Amber::Pipe::PoweredByAmber.new
    plug Amber::Pipe::Logger.new
    plug Amber::Pipe::Session.new
    plug Amber::Pipe::CSRF.new
  end
  
  routes :web do
    get "/", HomeController, :index
    resources "/articles", ArticleController
    
    scope "/api" do
      resources "/users", Api::UserController, only: [:index, :show, :create]
    end
  end
end

# src/controllers/article_controller.cr
class ArticleController < ApplicationController
  def index
    articles = Article.all.order(:created_at, :desc)
    render template: "articles/index.slang"
  end
  
  def create
    article = Article.new(article_params)
    
    if article.save
      flash[:success] = "สร้างบทความสำเร็จ"
      redirect_to action: :show, id: article.id
    else
      render template: "articles/new.slang", status: 422
    end
  end
end
```

## ORMs และ Database Libraries

### Jennifer - Active Record ORM

```crystal
# jennifer: github.com/imdrasil/jennifer.cr
require "jennifer"

# Model
class User < Jennifer::Model::Base
  with_timestamps
  
  mapping(
    id:    {type: Int32, primary: true},
    email: String,
    name:  String,
    role:  {type: String, default: "user"},
    active: {type: Bool, default: true}
  )
  
  has_many :posts, Post
  has_many :comments, Comment
  
  validates_presence :name, :email
  validates_uniqueness :email
  validates_format :email, /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  
  scope :active, -> { where(active: true) }
  scope :admins, -> { where(role: "admin") }
end

class Post < Jennifer::Model::Base
  with_timestamps
  
  mapping(
    id:      {type: Int32, primary: true},
    user_id: Int32,
    title:   String,
    content: {type: String?, default: nil},
    status:  {type: String, default: "draft"}
  )
  
  belongs_to :user, User
  
  validates_presence :title, :user_id
  validates_inclusion :status, %w[draft published archived]
  
  scope :published, -> { where(status: "published") }
end

# การใช้งาน
users = User.all.active.order(:name)
admins = User.admins.to_a
user = User.find!(1)
user_posts = user.posts.published.to_a

# Query Builder
User
  .where(role: "user")
  .where { age > 18 }
  .order("created_at DESC")
  .limit(10)
  .to_a
```

### Crecto - Ecto-inspired ORM

```crystal
# crecto: github.com/Crecto/crecto
require "crecto"

# Schema
class User
  include Crecto::Schema
  
  schema "users" do
    field :name, String
    field :email, String
    field :age, Int32
    has_many :posts, Post
  end
end

# Repo queries
users = Crecto::Repo.all(User)
user = Crecto::Repo.get(User, 1)

changeset = User.changeset(%{name: "ใหม่", email: "new@example.com"})
Crecto::Repo.insert(changeset)
```

## Popular Utility Shards

### การทำงานกับ JSON

```crystal
# การ serialize/deserialize JSON ใน Crystal
require "json"

struct Product
  include JSON::Serializable
  
  property id : Int64
  property name : String
  property price : Float64
  property tags : Array(String)
  
  @[JSON::Field(key: "created_at")]
  property created_at : Time
  
  @[JSON::Field(ignore: true)]
  property internal_code : String?
  
  def initialize(@id, @name, @price, @tags, @created_at = Time.utc)
  end
end

# Serialize
product = Product.new(1_i64, "สินค้าทดสอบ", 299.99, ["test", "product"])
puts product.to_json
# => {"id":1,"name":"สินค้าทดสอบ","price":299.99,"tags":["test","product"],"created_at":"..."}

# Deserialize
json_str = %|{"id": 2, "name": "อีกชิ้น", "price": 150.0, "tags": [], "created_at": "2024-01-01T00:00:00Z"}|
product2 = Product.from_json(json_str)
puts product2.name
```

### HTTP Client Library

```crystal
# crystal-http-client: ใช้ built-in HTTP::Client
require "http/client"
require "json"

class APIClient
  def initialize(
    @base_url : String,
    @api_key : String? = nil
  )
  end
  
  def get(path : String, params : Hash(String, String) = {} of String => String) : JSON::Any
    uri = URI.parse("#{@base_url}#{path}")
    unless params.empty?
      uri.query = URI::Params.build { |p| params.each { |k, v| p.add(k, v) } }
    end
    
    response = HTTP::Client.get(uri, headers: default_headers)
    JSON.parse(response.body)
  end
  
  def post(path : String, body : Hash) : JSON::Any
    response = HTTP::Client.post(
      "#{@base_url}#{path}",
      headers: default_headers,
      body: body.to_json
    )
    JSON.parse(response.body)
  end
  
  private def default_headers : HTTP::Headers
    headers = HTTP::Headers{"Content-Type" => "application/json"}
    headers["Authorization"] = "Bearer #{@api_key}" if @api_key
    headers
  end
end

# การใช้งาน
client = APIClient.new("https://api.example.com", api_key: "secret")
data = client.get("/users", {"page" => "1", "per_page" => "10"})
puts data["users"].as_a.size
```

### Cossack - HTTP Client Shard

```crystal
# cossack: github.com/crystal-community/cossack
# HTTP client ที่ใช้งานง่าย

# require "cossack"
# 
# client = Cossack::Client.new("https://httpbin.org") do |conn|
#   conn.headers["User-Agent"] = "Crystal App/1.0"
#   conn.use Cossack::LoggingMiddleware
#   conn.use Cossack::RetryMiddleware, max_retries: 3
# end
# 
# response = client.get("/json")
# puts response.status
# puts response.body
```

### Halite - HTTP Client with Middleware

```crystal
# halite: github.com/icyleaf/halite
# HTTP client แบบ chainable

# require "halite"
# 
# # GET
# response = Halite.get("https://httpbin.org/get",
#   params: {search: "crystal", page: 1}
# )
# 
# # POST JSON
# response = Halite.post("https://httpbin.org/post",
#   json: {name: "Crystal", version: "1.x"}
# )
# 
# # With Authentication
# Halite.auth("Bearer", "your-token")
#   .get("https://api.github.com/user")
# 
# # Persistent Client
# github = Halite::Client.new do |c|
#   c.endpoint "https://api.github.com"
#   c.auth "Bearer", ENV["GITHUB_TOKEN"]
#   c.headers accept: "application/vnd.github.v3+json"
# end
```

## Testing Libraries

### Spec - Built-in Testing

```crystal
# Crystal มี built-in testing framework
require "spec"

describe "User" do
  subject { User.new(name: "ทดสอบ", email: "test@example.com") }
  
  it "มีชื่อ" do
    subject.name.should eq("ทดสอบ")
  end
  
  it "มีอีเมลที่ถูกต้อง" do
    subject.email.should match(/.+@.+\..+/)
  end
  
  context "เมื่อ admin" do
    subject { User.new(name: "Admin", email: "admin@example.com", role: "admin") }
    
    it "ตรวจสอบว่าเป็น admin ได้" do
      subject.admin?.should be_true
    end
  end
  
  describe "#validate" do
    it "ต้องการชื่อ" do
      user = User.new(name: "", email: "test@test.com")
      user.valid?.should be_false
      user.errors[:name].should_not be_empty
    end
  end
end
```

### Mocks ใน Crystal Tests

```crystal
# webmock.cr: github.com/manastech/webmock.cr
# สำหรับ mock HTTP requests ใน tests

# require "webmock"
# 
# WebMock.stub(:get, "https://api.example.com/users")
#   .to_return(body: [{id: 1, name: "Test"}].to_json)
# 
# describe "UserService" do
#   it "ดึงผู้ใช้จาก API" do
#     service = UserService.new
#     users = service.fetch_users
#     users.size.should eq(1)
#     users.first.name.should eq("Test")
#   end
# end
```

## CLI Tools

### option_parser - Built-in CLI

```crystal
# Crystal มี OptionParser built-in
require "option_parser"

class CLI
  def self.parse(args = ARGV)
    options = {
      verbose: false,
      output: "stdout",
      format: "json",
      limit: 100
    }
    
    OptionParser.parse(args) do |parser|
      parser.banner = "การใช้งาน: myapp [options]"
      
      parser.on("-v", "--verbose", "แสดงข้อมูลเพิ่มเติม") do
        options = options.merge(verbose: true)
      end
      
      parser.on("-o FILE", "--output=FILE", "ไฟล์ output") do |file|
        options = options.merge(output: file)
      end
      
      parser.on("-f FORMAT", "--format=FORMAT", "รูปแบบ output (json/csv/table)") do |fmt|
        options = options.merge(format: fmt)
      end
      
      parser.on("-l N", "--limit=N", "จำกัดจำนวน records") do |n|
        options = options.merge(limit: n.to_i)
      end
      
      parser.on("-h", "--help", "แสดง help") do
        puts parser
        exit(0)
      end
      
      parser.on("-V", "--version", "แสดงเวอร์ชัน") do
        puts "myapp 1.0.0"
        exit(0)
      end
      
      parser.invalid_option do |flag|
        STDERR.puts "ไม่รู้จัก option: #{flag}"
        STDERR.puts parser
        exit(1)
      end
    end
    
    options
  end
end
```

## Community Resources

### แหล่งข้อมูลหลัก

```
Crystal Programming Language:
  เว็บไซต์หลัก:  https://crystal-lang.org
  Documentation: https://crystal-lang.org/docs/
  API Reference: https://crystal-lang.org/api/
  
Community:
  Forum:     https://forum.crystal-lang.org
  Discord:   https://discord.gg/YS7YvQy
  GitHub:    https://github.com/crystal-lang/crystal
  Twitter:   @CrystalLanguage
  
Shards:
  shards.info:    https://shards.info (ค้นหา shards)
  crystalshards:  https://crystalshards.xyz
  
Blogs & Tutorials:
  Crystal Blog:      https://crystal-lang.org/blog/
  Awesome Crystal:   https://github.com/veelenga/awesome-crystal
  DEV Community:     https://dev.to/t/crystal
```

### สมาชิก Community ที่มีชื่อเสียง

```
Core Team:
  - Ary Borenszweig (@asterite) - Crystal co-creator
  - Juan Wajnerman (@waj)       - Crystal co-creator
  - Brian Cardiff (@bcardiff)   - Core contributor
  
Notable Contributors:
  - Manas Teknoloji (manas.tech) - Development team
  - Julien Portalier (@ysbaddaden) - HTTP, testing
  - Sijawusz Pur Rahnama           - many shards
  
Active Shard Authors:
  - Nikolay Edigaryev - Lucky Framework
  - Mike Perham       - Sidekiq.cr
  - Andrea Foltyn     - Crystal DB
```

## Contributing to Crystal

### วิธีการ Contribute

```bash
# 1. Fork repository
git clone https://github.com/crystal-lang/crystal.git
cd crystal

# 2. Build Crystal จาก source
make

# 3. รัน test suite
make spec

# 4. สร้าง branch
git checkout -b fix/my-improvement

# 5. แก้ไขโค้ด
# ...

# 6. รัน tests อีกครั้ง
make spec

# 7. Push และสร้าง PR
git push origin fix/my-improvement
```

### การสร้าง Shard ใหม่

```bash
# สร้าง project structure
crystal init lib my_shard
cd my_shard

# โครงสร้าง
# my_shard/
# ├── .github/
# │   └── workflows/
# │       └── crystal.yml
# ├── src/
# │   └── my_shard.cr
# ├── spec/
# │   ├── my_shard_spec.cr
# │   └── spec_helper.cr
# ├── shard.yml
# ├── README.md
# └── LICENSE
```

### shard.yml ที่ดี

```yaml
# shard.yml
name: my_awesome_shard
version: 0.1.0
description: |
  Shard ที่ทำให้ชีวิตง่ายขึ้น

authors:
  - Your Name <your.email@example.com>

crystal: ">= 1.0.0"

license: MIT

dependencies:
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0

development_dependencies:
  webmock:
    github: manastech/webmock.cr
    version: ~> 0.7.0
```

### CI/CD สำหรับ Shard

```yaml
# .github/workflows/crystal.yml
name: Crystal CI

on: [push, pull_request]

jobs:
  build:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        crystal: [latest, nightly]
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: ${{ matrix.crystal }}
      
      - name: Install dependencies
        run: shards install
      
      - name: Check formatting
        run: crystal tool format --check
      
      - name: Run linter
        run: bin/ameba
        continue-on-error: true
      
      - name: Run tests
        run: crystal spec
      
      - name: Build
        run: shards build --release
```

## Notable Crystal Projects

### โปรเจกต์ที่น่าสนใจ

```
1. Mirth - Static Site Generator
   github.com/thebmartin/mirth

2. DeBot - Discord bot framework
   github.com/discordcr/discordcr

3. Mint - Frontend language (compiles to JS)
   github.com/mint-lang/mint

4. Cride - Crypto library
   github.com/niclas-ahden/crystal-cryptonight

5. Amber Framework
   github.com/amberframework/amber

6. Lucky Framework
   github.com/luckyframework/lucky

7. Marten - Django-inspired web framework
   github.com/martenframework/marten

8. Grotto - PostgreSQL driver
   github.com/will/crystal-pg

9. Clear ORM
   github.com/anykeyh/clear

10. Tourmaline - Telegram bot framework
    github.com/protoncr/tourmaline
```

### Production Users

```
Companies using Crystal:
- Manas Technology (Argentina) - Crystal creator/maintainer
- 84codes               - CloudAMQP messaging platform
- GitLab                - Uses Crystal for some tools
- Shopify               - Experimenting with Crystal
- VOIQ                  - Telecom AI platform
- Ollivier              - AI voice tools

Open Source Projects:
- AvalancheMQ           - High-performance AMQP server in Crystal
- Opentelemetry Crystal - Distributed tracing
- Spectator             - Full-featured test framework
```

## Crystal เทียบกับ Ruby

```crystal
# Crystal ไวยากรณ์คล้าย Ruby มาก แต่มีความแตกต่าง

# Ruby: Dynamic typing
# x = 5
# x = "string"  # ทำได้ใน Ruby

# Crystal: Static typing (type inference)
x = 5
# x = "string"  # Error! ประเภทไม่ตรง

# Ruby: Nil safety ไม่มี
# user = find_user(1)  # อาจเป็น nil
# user.name            # อาจ crash ตอน runtime

# Crystal: Nil safety บังคับ
user = find_user(1)  # ได้ User | Nil
# user.name           # Compile error!
user.try(&.name)     # ปลอดภัย
# หรือ
user.not_nil!.name   # ยืนยันว่าไม่ nil (crash ถ้า nil)

# ความเร็ว: Crystal เร็วกว่า Ruby 10-100x
# Memory: Crystal ใช้ memory น้อยกว่า
# Compilation: Crystal compile ไปเป็น native binary
```

## Crystal เทียบกับ Go

```
Crystal vs Go Comparison:

| Feature          | Crystal        | Go             |
|------------------|----------------|----------------|
| Type System      | Static + OOP   | Static + Struct|
| Null Safety      | Nil union type | Interface nil  |
| Concurrency      | Fiber-based    | Goroutine      |
| Memory           | GC             | GC             |
| Compile Speed    | Slow           | Very Fast      |
| Runtime Speed    | Very Fast      | Very Fast      |
| Syntax           | Ruby-like      | C-like         |
| Generics         | Yes            | Yes (Go 1.18+) |
| Macros           | Yes            | No             |
| Standard Library | Good           | Excellent      |
| Ecosystem        | Growing        | Mature         |
| Concurrency Model| M:N Fibers     | Goroutines     |
| Error Handling   | Exceptions+    | Multiple returns|
```

## Crystal เทียบกับ Rust

```
Crystal vs Rust Comparison:

| Feature          | Crystal        | Rust           |
|------------------|----------------|----------------|
| Memory Safety    | GC             | Ownership      |
| Learning Curve   | Low            | High           |
| Syntax           | Ruby-like      | C-like         |
| Performance      | Very Fast      | Fastest        |
| Memory Usage     | Moderate       | Minimal        |
| Web Development  | Good           | Good           |
| Systems Prog.    | Limited        | Excellent      |
| Error Handling   | Exceptions     | Result type    |
| Concurrency      | Fibers + Chan  | async/await    |

Crystal เหมาะสำหรับ:
- Web applications
- API services
- Scripting ขนาดใหญ่
- Tools & CLIs

Rust เหมาะสำหรับ:
- Systems programming
- Embedded systems
- WebAssembly
- Performance critical apps
```

## อนาคตของ Crystal

### Roadmap สำคัญ

```
Crystal ที่กำลังพัฒนา:
1. Parallel execution (multi-threading)
   - ปัจจุบันใช้ GIL (Global Interpreter Lock)
   - กำลังพัฒนา thread-safe runtime
   
2. Windows support improvements
   - Native Windows binary
   - Better tooling support
   
3. WebAssembly compilation target
   - รัน Crystal code ใน browser
   
4. Language Server Protocol (LSP)
   - Better IDE integration
   - VSCode/Neovim support
   
5. Package manager improvements
   - Better dependency resolution
   - Faster shard installation
   
6. Performance improvements
   - Faster GC
   - Better compile time optimization
```

### Crystal 2.0 Features

```crystal
# Future features ที่คาดหวัง:

# 1. Named tuples improvements
# record Person, name: String, age: Int32
# p = Person.new(name: "สมชาย", age: 25)

# 2. Better error messages
# Compile errors ที่อ่านเข้าใจง่ายขึ้น

# 3. Improved generics
# class Container(T) where T : Comparable
#   def max : T
#     ...
#   end
# end

# 4. Pattern matching (Crystal 1.9+ มีแล้ว)
case value
in {name: String => name, age: (18..) => age}
  puts "#{name} อายุ #{age} เป็นผู้ใหญ่แล้ว"
in {name: String => name}
  puts "#{name} ยังเป็นเด็ก"
end
```

## Awesome Crystal Resources

### Shards ที่ควรรู้จัก

```
Web Frameworks:
- kemal          (github.com/kemalcr/kemal)        - Micro framework
- lucky          (github.com/luckyframework/lucky)  - Full-stack
- amber          (github.com/amberframework/amber)  - MVC framework
- marten         (github.com/martenframework/marten)- Django-inspired

Database:
- crystal-db     (github.com/crystal-lang/crystal-db)     - DB abstraction
- crystal-mysql  (github.com/crystal-lang/crystal-mysql)  - MySQL
- crystal-pg     (github.com/will/crystal-pg)             - PostgreSQL
- jennifer.cr    (github.com/imdrasil/jennifer.cr)        - Active Record ORM
- granite        (github.com/amberframework/granite)       - ORM
- cryomongo      (github.com/elbywan/cryomongo)           - MongoDB

HTTP:
- halite         (github.com/icyleaf/halite)     - HTTP client
- cossack        (github.com/crystal-community/cossack)
- http           (built-in)

Testing:
- spec           (built-in)
- spectator      (github.com/icy-arctic-fox/spectator)  - Full-featured
- webmock        (github.com/manastech/webmock.cr)      - HTTP mocking
- factory_bot    (github.com/matthewmcgarvey/factory_bot.cr)

CLI:
- option_parser  (built-in)
- commander      (github.com/mrrooijen/commander)

Background Jobs:
- sidekiq.cr     (github.com/mperham/sidekiq.cr)
- mosquito        (github.com/robacarp/mosquito)

Serialization:
- json           (built-in)
- yaml           (built-in)
- csv            (built-in)
- messagepack    (github.com/crystal-community/msgpack-crystal)

Auth:
- devise.cr      
- jwt            (github.com/crystal-community/jwt)
- crypto         (built-in)

Utilities:
- crinja         (github.com/straight-shoota/crinja)  - Jinja2 templates
- markd          (github.com/icyleaf/markd)           - Markdown parser
- i18n           (github.com/TechMagister/i18n.cr)   - Internationalization
- redis          (github.com/stefanwille/crystal-redis)
- email          (github.com/arcage/crystal-email)
```

## สรุป

Crystal ecosystem มีสิ่งสำคัญ:
- **Web Frameworks**: Kemal (ง่าย), Lucky (full-stack), Amber, Marten
- **Database**: crystal-db abstraction, Jennifer ORM, native drivers
- **Community**: Forum, Discord, GitHub อยู่ที่ crystal-lang.org
- **Contributing**: Fork + PR process เหมือน GitHub standard
- **เทียบกับภาษาอื่น**: เร็วกว่า Ruby, ง่ายกว่า Rust, คล้าย Go
- **อนาคต**: Multi-threading, WASM, better tooling

Crystal เป็นภาษาที่มีอนาคต เหมาะสำหรับ developer ที่ต้องการความเร็วของ compiled language แต่ยังต้องการความอ่านง่ายของ Ruby

## ขั้นตอนต่อไป

ใน **Part 198** เราจะเริ่มต้น **Final Project** ซึ่งเป็นการสร้าง Full-Stack Web Application แบบ Blog/CMS ที่รวมทุกสิ่งที่เรียนมาตลอดคอร์ส
