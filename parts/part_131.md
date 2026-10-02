# Part 131: Kemal Framework Introduction - เริ่มต้นกับ Kemal

## บทนำ

Kemal เป็น web framework ที่ได้รับแรงบันดาลใจจาก Sinatra (Ruby) สำหรับ Crystal มีความเรียบง่าย เร็ว และใช้งานได้ทันที เหมาะกับการสร้าง REST APIs และ web applications

## การติดตั้ง

เพิ่มใน `shard.yml`:

```yaml
dependencies:
  kemal:
    github: kemalcr/kemal
    version: ~> 1.4.0
```

แล้วรัน:
```bash
shards install
```

## Hello World

```crystal
require "kemal"

# Route พื้นฐาน
get "/" do
  "สวัสดี, Kemal!"
end

Kemal.run
```

รัน: `crystal run src/app.cr`

## GET, POST, PUT, DELETE Routes

```crystal
require "kemal"

# GET - ดึงข้อมูล
get "/hello" do
  "Hello from GET"
end

get "/hello/:name" do |env|
  name = env.params.url["name"]
  "สวัสดี, #{name}!"
end

# POST - สร้างข้อมูล
post "/users" do |env|
  name = env.params.body["name"]?
  email = env.params.body["email"]?
  "สร้างผู้ใช้: #{name} (#{email})"
end

# PUT - อัปเดตข้อมูล
put "/users/:id" do |env|
  id = env.params.url["id"]
  "อัปเดตผู้ใช้ #{id}"
end

# PATCH - อัปเดตบางส่วน
patch "/users/:id" do |env|
  id = env.params.url["id"]
  "Partial update user #{id}"
end

# DELETE - ลบข้อมูล
delete "/users/:id" do |env|
  id = env.params.url["id"]
  env.response.status_code = 204
end

Kemal.run
```

## Request และ Response

```crystal
require "kemal"

get "/request-info" do |env|
  # Request
  method = env.request.method
  path = env.request.path
  query = env.request.query
  user_agent = env.request.headers["User-Agent"]?
  remote_ip = env.request.remote_address
  
  # Response
  env.response.status_code = 200
  env.response.content_type = "application/json"
  env.response.headers["X-Custom-Header"] = "Kemal"
  
  {
    "method" => method,
    "path" => path,
    "query" => query,
    "user_agent" => user_agent || "",
    "remote_ip" => remote_ip.to_s,
  }.to_json
end

# JSON Response
get "/json" do |env|
  env.response.content_type = "application/json"
  {"message" => "Hello JSON", "status" => "ok"}.to_json
end

# Redirect
get "/old-path" do |env|
  env.redirect "/new-path"
end

get "/new-path" do
  "This is the new path"
end

Kemal.run
```

## Query Parameters

```crystal
require "kemal"

get "/search" do |env|
  # query params
  q = env.params.query["q"]?
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = env.params.query["per_page"]?.try(&.to_i?) || 20
  
  "ค้นหา: #{q || "(ว่าง)"}, หน้า #{page}, จำนวน #{per_page}"
end

# URL: GET /search?q=crystal&page=2&per_page=10
```

## Running Server

```crystal
require "kemal"

# ตั้งค่า Server
Kemal.config do |config|
  config.port = 3000          # default 3000
  config.host_binding = "0.0.0.0"
  config.env = "production"  # "development" | "production" | "test"
  config.logging = true
end

get "/" do
  "Server running on port #{Kemal.config.port}"
end

# รัน server
Kemal.run
```

## JSON API Example

```crystal
require "kemal"
require "json"

# In-memory database
struct User
  include JSON::Serializable
  
  property id : Int32
  property name : String
  property email : String
  property created_at : String
  
  def initialize(@id, @name, @email)
    @created_at = Time.local.to_rfc3339
  end
end

users = [] of User
next_id = 1

# Helper
macro json_response(env, data, status = 200)
  {{env}}.response.content_type = "application/json"
  {{env}}.response.status_code = {{status}}
  {{data}}.to_json
end

# GET /users - ดึง users ทั้งหมด
get "/users" do |env|
  json_response(env, users)
end

# GET /users/:id - ดึง user คนเดียว
get "/users/:id" do |env|
  id = env.params.url["id"].to_i?
  
  unless id
    halt env, status_code: 400, response: {"error" => "Invalid ID"}.to_json
  end
  
  user = users.find { |u| u.id == id }
  
  if user
    json_response(env, user)
  else
    halt env, status_code: 404, response: {"error" => "User not found"}.to_json
  end
end

# POST /users - สร้าง user ใหม่
post "/users" do |env|
  begin
    body = env.request.body.try(&.gets_to_end) || ""
    data = JSON.parse(body)
    
    name = data["name"]?.try(&.as_s?)
    email = data["email"]?.try(&.as_s?)
    
    unless name && email
      halt env, status_code: 400, response: {"error" => "name and email required"}.to_json
    end
    
    user = User.new(next_id, name, email)
    users << user
    next_id += 1
    
    env.response.content_type = "application/json"
    env.response.status_code = 201
    user.to_json
  rescue JSON::ParseException
    halt env, status_code: 400, response: {"error" => "Invalid JSON"}.to_json
  end
end

# PUT /users/:id - อัปเดต user
put "/users/:id" do |env|
  id = env.params.url["id"].to_i?
  
  unless id
    halt env, status_code: 400, response: {"error" => "Invalid ID"}.to_json
  end
  
  idx = users.index { |u| u.id == id }
  
  unless idx
    halt env, status_code: 404, response: {"error" => "User not found"}.to_json
  end
  
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body)
  
  name = data["name"]?.try(&.as_s?) || users[idx].name
  email = data["email"]?.try(&.as_s?) || users[idx].email
  
  users[idx] = User.new(id, name, email)
  json_response(env, users[idx])
end

# DELETE /users/:id - ลบ user
delete "/users/:id" do |env|
  id = env.params.url["id"].to_i?
  
  unless id
    halt env, status_code: 400, response: {"error" => "Invalid ID"}.to_json
  end
  
  initial_size = users.size
  users.reject! { |u| u.id == id }
  
  if users.size < initial_size
    env.response.status_code = 204
    ""
  else
    halt env, status_code: 404, response: {"error" => "User not found"}.to_json
  end
end

# Error handlers
error 404 do |env|
  env.response.content_type = "application/json"
  {"error" => "Not Found"}.to_json
end

error 500 do |env|
  env.response.content_type = "application/json"
  {"error" => "Internal Server Error"}.to_json
end

Kemal.run
```

## Handling Content Types

```crystal
require "kemal"
require "json"

# รับข้อมูลในรูปแบบต่างๆ
post "/data" do |env|
  content_type = env.request.headers["Content-Type"]? || ""
  
  if content_type.includes?("application/json")
    body = env.request.body.try(&.gets_to_end) || "{}"
    data = JSON.parse(body)
    "JSON: #{data.inspect}"
  elsif content_type.includes?("application/x-www-form-urlencoded")
    name = env.params.body["name"]?
    "Form: name=#{name}"
  elsif content_type.includes?("multipart/form-data")
    file = env.params.files["file"]?
    "File: #{file.try(&.filename)}"
  else
    "Unknown content type"
  end
end
```

## Static Files

```crystal
require "kemal"

# serve static files จาก public/ directory
# Kemal จะ serve ไฟล์จาก public/ โดยอัตโนมัติ
# เช่น public/style.css -> GET /style.css

serve_static({"dir" => "public"})

get "/" do
  render "src/views/index.ecr"
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Simple Blog API

```crystal
require "kemal"
require "json"

struct Post
  include JSON::Serializable
  
  property id : Int32
  property title : String
  property content : String
  property author : String
  property created_at : String
  property tags : Array(String)
  
  def initialize(@id, @title, @content, @author, @tags = [] of String)
    @created_at = Time.local.to_rfc3339
  end
end

# Blog API
posts = [] of Post
next_id = 1

def json(env, data, status = 200)
  env.response.content_type = "application/json"
  env.response.status_code = status
  data.to_json
end

# Get all posts with filtering
get "/posts" do |env|
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = env.params.query["per_page"]?.try(&.to_i?) || 10
  author = env.params.query["author"]?
  tag = env.params.query["tag"]?
  
  filtered = posts.select do |p|
    (author.nil? || p.author == author) &&
    (tag.nil? || p.tags.includes?(tag))
  end
  
  start_idx = (page - 1) * per_page
  paginated = filtered[start_idx, per_page]? || [] of Post
  
  json(env, {
    "posts" => paginated,
    "total" => filtered.size,
    "page" => page,
    "per_page" => per_page,
    "pages" => (filtered.size.to_f / per_page).ceil.to_i,
  })
end

get "/posts/:id" do |env|
  id = env.params.url["id"].to_i? || halt(env, status_code: 400, response: "Bad Request")
  post = posts.find { |p| p.id == id }
  post ? json(env, post) : halt(env, status_code: 404, response: {"error" => "Not found"}.to_json)
end

post "/posts" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body) rescue halt(env, status_code: 400, response: "Invalid JSON")
  
  title = data["title"]?.try(&.as_s?)
  content = data["content"]?.try(&.as_s?)
  author = data["author"]?.try(&.as_s?)
  tags = data["tags"]?.try(&.as_a?.try { |a| a.map(&.as_s?) }.compact) || [] of String
  
  unless title && content && author
    halt env, status_code: 400, response: {"error" => "title, content, author required"}.to_json
  end
  
  post = Post.new(next_id, title, content, author, tags)
  posts << post
  next_id += 1
  json(env, post, 201)
end

delete "/posts/:id" do |env|
  id = env.params.url["id"].to_i? || 0
  count = posts.size
  posts.reject! { |p| p.id == id }
  
  if posts.size < count
    env.response.status_code = 204
    ""
  else
    halt env, status_code: 404, response: {"error" => "Not found"}.to_json
  end
end

# Seed data
posts << Post.new(1, "สวัสดี Crystal", "Crystal เป็นภาษาที่น่าสนใจ", "admin", ["crystal", "programming"])
posts << Post.new(2, "Kemal Framework", "Kemal ทำให้การสร้าง web app ง่ายขึ้น", "admin", ["crystal", "web"])
next_id = 3

Kemal.run
```

### แบบฝึกหัดที่ 2: Health Check API

```crystal
require "kemal"
require "json"

start_time = Time.local
request_count = Atomic(Int64).new(0)
error_count = Atomic(Int64).new(0)

before_all do |env|
  request_count.add(1)
end

get "/health" do |env|
  env.response.content_type = "application/json"
  {
    "status" => "ok",
    "uptime_seconds" => (Time.local - start_time).total_seconds.to_i,
    "requests" => request_count.get,
    "errors" => error_count.get,
    "timestamp" => Time.local.to_rfc3339,
    "version" => "1.0.0",
  }.to_json
end

get "/health/ready" do |env|
  # ตรวจสอบ dependencies
  checks = {
    "database" => check_database,
    "cache" => check_cache,
  }
  
  all_ok = checks.values.all? { |v| v == true }
  
  env.response.status_code = all_ok ? 200 : 503
  env.response.content_type = "application/json"
  
  {"ready" => all_ok, "checks" => checks}.to_json
end

def check_database : Bool
  # จำลองการตรวจสอบ database
  true
end

def check_cache : Bool
  # จำลองการตรวจสอบ cache
  true
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **การติดตั้ง Kemal**: เพิ่ม shard และ run
2. **HTTP Methods**: get, post, put, patch, delete
3. **URL Parameters**: `env.params.url["id"]`
4. **Query Parameters**: `env.params.query["key"]`
5. **Body Parameters**: `env.params.body["key"]`
6. **Request/Response**: `env.request`, `env.response`
7. **JSON Responses**: content_type + to_json
8. **Status Codes**: `env.response.status_code`
9. **halt**: หยุด request ทันที
10. **redirect**: redirect ไปยัง URL อื่น
11. **Error Handlers**: `error 404`, `error 500`
12. **Static Files**: serve files จาก public/

Kemal ใช้งานง่าย เหมาะกับ microservices และ REST APIs
