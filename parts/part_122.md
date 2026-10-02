# Part 122: HTTP Server พื้นฐาน - การสร้าง HTTP Server ใน Crystal

## บทนำ

Crystal มี built-in HTTP server ที่มีประสิทธิภาพสูงด้วย `HTTP::Server` ในบทนี้เราจะเรียนรู้การสร้าง HTTP server, จัดการ request/response, และทำ routing ด้วยตัวเอง

## HTTP::Server พื้นฐาน

```crystal
require "http/server"

# Server แบบง่ายที่สุด
server = HTTP::Server.new do |context|
  context.response.content_type = "text/plain"
  context.response.print "สวัสดี, Crystal HTTP Server!"
end

puts "รอรับ request บน http://localhost:8080"
server.listen("0.0.0.0", 8080)
```

## HTTP::Handler

Handler เป็น middleware ที่สามารถเชื่อมต่อกันเป็น chain ได้

```crystal
require "http/server"

# Custom Handler
class LoggingHandler
  include HTTP::Handler
  
  def call(context : HTTP::Server::Context)
    start = Time.monotonic
    call_next(context) # ส่งต่อให้ handler ถัดไป
    elapsed = (Time.monotonic - start).total_milliseconds
    
    puts "[#{Time.local}] #{context.request.method} #{context.request.path} - #{context.response.status_code} (#{elapsed.round(1)}ms)"
  end
end

class ErrorHandler
  include HTTP::Handler
  
  def call(context : HTTP::Server::Context)
    begin
      call_next(context)
    rescue ex
      context.response.status_code = 500
      context.response.content_type = "text/plain"
      context.response.print "Internal Server Error: #{ex.message}"
    end
  end
end

class MainHandler
  include HTTP::Handler
  
  def call(context : HTTP::Server::Context)
    context.response.content_type = "text/html"
    context.response.print "<h1>Hello from Crystal!</h1>"
  end
end

# เชื่อม handlers เข้าด้วยกัน
server = HTTP::Server.new([
  LoggingHandler.new,
  ErrorHandler.new,
  MainHandler.new,
])

server.listen("0.0.0.0", 8080)
```

## HTTP::Context

Context มีข้อมูลทั้ง request และ response

```crystal
require "http/server"

server = HTTP::Server.new do |context|
  req = context.request
  res = context.response
  
  # Request information
  puts "Method: #{req.method}"
  puts "Path: #{req.path}"
  puts "HTTP Version: #{req.version}"
  puts "Host: #{req.headers["Host"]?}"
  puts "User-Agent: #{req.headers["User-Agent"]?}"
  
  # Query parameters
  params = req.query_params
  puts "Params: #{params.to_h.inspect}"
  
  # Request body
  body = req.body.try(&.gets_to_end)
  puts "Body: #{body}" if body
  
  # Response
  res.status_code = 200
  res.content_type = "application/json"
  res.headers["X-Custom-Header"] = "Crystal"
  res.print({"message" => "OK", "path" => req.path}.to_json)
end

server.listen("0.0.0.0", 8080)
```

## Manual Routing

```crystal
require "http/server"
require "json"

class Router
  include HTTP::Handler
  
  alias RouteHandler = Proc(HTTP::Server::Context, Nil)
  
  def initialize
    @routes = {} of String => Hash(String, RouteHandler)
  end
  
  def get(path : String, &handler : HTTP::Server::Context ->)
    add_route("GET", path, handler)
  end
  
  def post(path : String, &handler : HTTP::Server::Context ->)
    add_route("POST", path, handler)
  end
  
  def put(path : String, &handler : HTTP::Server::Context ->)
    add_route("PUT", path, handler)
  end
  
  def delete(path : String, &handler : HTTP::Server::Context ->)
    add_route("DELETE", path, handler)
  end
  
  def call(context : HTTP::Server::Context)
    method = context.request.method
    path = context.request.path
    
    if route_map = @routes[method]?
      if handler = route_map[path]?
        handler.call(context)
      else
        # หา dynamic route
        matched = false
        route_map.each do |route_path, handler|
          if params = match_route(route_path, path)
            context.request.url_params = params  # custom extension
            handler.call(context)
            matched = true
            break
          end
        end
        
        unless matched
          not_found(context)
        end
      end
    else
      method_not_allowed(context)
    end
  end
  
  private def add_route(method : String, path : String, handler : RouteHandler)
    @routes[method] ||= {} of String => RouteHandler
    @routes[method][path] = handler
  end
  
  private def match_route(route : String, path : String) : Hash(String, String)?
    route_parts = route.split("/")
    path_parts = path.split("/")
    
    return nil if route_parts.size != path_parts.size
    
    params = {} of String => String
    
    route_parts.each_with_index do |part, i|
      if part.starts_with?(":")
        params[part[1..]] = path_parts[i]
      elsif part != path_parts[i]
        return nil
      end
    end
    
    params
  end
  
  private def not_found(context : HTTP::Server::Context)
    context.response.status_code = 404
    context.response.content_type = "application/json"
    context.response.print({"error" => "Not Found"}.to_json)
  end
  
  private def method_not_allowed(context : HTTP::Server::Context)
    context.response.status_code = 405
    context.response.content_type = "application/json"
    context.response.print({"error" => "Method Not Allowed"}.to_json)
  end
end
```

## ตัวอย่าง Full REST API

```crystal
require "http/server"
require "json"

# Simple in-memory store
class UserStore
  struct User
    include JSON::Serializable
    
    property id : Int32
    property name : String
    property email : String
    
    def initialize(@id, @name, @email)
    end
  end
  
  def initialize
    @users = {} of Int32 => User
    @next_id = 1
    # เพิ่มข้อมูลตัวอย่าง
    create("สมชาย ใจดี", "somchai@example.com")
    create("สมหญิง รักดี", "somying@example.com")
  end
  
  def all : Array(User)
    @users.values.sort_by(&.id)
  end
  
  def find(id : Int32) : User?
    @users[id]?
  end
  
  def create(name : String, email : String) : User
    user = User.new(@next_id, name, email)
    @users[@next_id] = user
    @next_id += 1
    user
  end
  
  def update(id : Int32, name : String, email : String) : User?
    return nil unless @users.has_key?(id)
    user = User.new(id, name, email)
    @users[id] = user
    user
  end
  
  def delete(id : Int32) : Bool
    if @users.has_key?(id)
      @users.delete(id)
      true
    else
      false
    end
  end
end

# Helpers
def json_response(context : HTTP::Server::Context, data, status : Int32 = 200)
  context.response.status_code = status
  context.response.content_type = "application/json"
  context.response.print data.to_json
end

def parse_id(context : HTTP::Server::Context) : Int32?
  path_parts = context.request.path.split("/")
  path_parts.last.to_i?
end

def parse_body(context : HTTP::Server::Context) : JSON::Any?
  body = context.request.body.try(&.gets_to_end)
  return nil unless body && !body.empty?
  JSON.parse(body) rescue nil
end

# Server setup
store = UserStore.new

server = HTTP::Server.new do |context|
  path = context.request.path
  method = context.request.method
  
  case {method, path}
  when {"GET", "/"}
    json_response(context, {"message" => "Crystal REST API", "version" => "1.0"})
  
  when {"GET", "/users"}
    json_response(context, store.all)
  
  when {"POST", "/users"}
    if body = parse_body(context)
      name = body["name"]?.try(&.as_s?)
      email = body["email"]?.try(&.as_s?)
      
      if name && email
        user = store.create(name, email)
        json_response(context, user, 201)
      else
        json_response(context, {"error" => "name and email required"}, 400)
      end
    else
      json_response(context, {"error" => "Invalid JSON body"}, 400)
    end
  
  else
    # Dynamic routes สำหรับ /users/:id
    if path.starts_with?("/users/")
      id = path.split("/").last.to_i?
      
      if id.nil?
        json_response(context, {"error" => "Invalid ID"}, 400)
        next
      end
      
      case method
      when "GET"
        if user = store.find(id)
          json_response(context, user)
        else
          json_response(context, {"error" => "User not found"}, 404)
        end
      
      when "PUT"
        if body = parse_body(context)
          name = body["name"]?.try(&.as_s?)
          email = body["email"]?.try(&.as_s?)
          
          if name && email
            if user = store.update(id, name, email)
              json_response(context, user)
            else
              json_response(context, {"error" => "User not found"}, 404)
            end
          else
            json_response(context, {"error" => "name and email required"}, 400)
          end
        else
          json_response(context, {"error" => "Invalid JSON body"}, 400)
        end
      
      when "DELETE"
        if store.delete(id)
          context.response.status_code = 204
        else
          json_response(context, {"error" => "User not found"}, 404)
        end
      
      else
        json_response(context, {"error" => "Method not allowed"}, 405)
      end
    else
      json_response(context, {"error" => "Not found"}, 404)
    end
  end
end

puts "Server running on http://localhost:8080"
server.listen("0.0.0.0", 8080)
```

## Static File Handler

```crystal
require "http/server"

# Built-in static file handler
server = HTTP::Server.new([
  HTTP::StaticFileHandler.new("public/", fallthrough: false),
  HTTP::ErrorHandler.new,
])

server.listen("0.0.0.0", 8080)
```

## Middleware Chain

```crystal
require "http/server"

# Logging middleware
class LoggingHandler
  include HTTP::Handler
  
  def call(context)
    puts "→ #{context.request.method} #{context.request.path}"
    call_next(context)
    puts "← #{context.response.status_code}"
  end
end

# CORS middleware
class CORSHandler
  include HTTP::Handler
  
  def initialize(@allowed_origins : Array(String) = ["*"])
  end
  
  def call(context)
    origin = context.request.headers["Origin"]?
    
    if origin && (@allowed_origins.includes?("*") || @allowed_origins.includes?(origin))
      context.response.headers["Access-Control-Allow-Origin"] = origin
      context.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE, OPTIONS"
      context.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"
    end
    
    # Handle preflight
    if context.request.method == "OPTIONS"
      context.response.status_code = 204
      return
    end
    
    call_next(context)
  end
end

# Auth middleware
class AuthHandler
  include HTTP::Handler
  
  VALID_TOKENS = ["token123", "secretkey456"]
  
  def call(context)
    # ยกเว้น public routes
    if context.request.path.starts_with?("/public/")
      call_next(context)
      return
    end
    
    auth = context.request.headers["Authorization"]?
    
    if auth && auth.starts_with?("Bearer ")
      token = auth[7..]
      if VALID_TOKENS.includes?(token)
        call_next(context)
        return
      end
    end
    
    context.response.status_code = 401
    context.response.content_type = "application/json"
    context.response.print({"error" => "Unauthorized"}.to_json)
  end
end

server = HTTP::Server.new([
  LoggingHandler.new,
  CORSHandler.new(["http://localhost:3000", "https://myapp.com"]),
  AuthHandler.new,
  HTTP::StaticFileHandler.new("public/"),
])

server.listen("0.0.0.0", 8080)
```

## Response Helpers

```crystal
require "http/server"
require "json"

module ResponseHelper
  def self.ok(context : HTTP::Server::Context, data = nil, message = "Success")
    context.response.status_code = 200
    context.response.content_type = "application/json"
    context.response.print({
      "success" => true,
      "message" => message,
      "data" => data,
    }.to_json)
  end
  
  def self.created(context : HTTP::Server::Context, data)
    context.response.status_code = 201
    context.response.content_type = "application/json"
    context.response.print({
      "success" => true,
      "data" => data,
    }.to_json)
  end
  
  def self.no_content(context : HTTP::Server::Context)
    context.response.status_code = 204
  end
  
  def self.bad_request(context : HTTP::Server::Context, message : String)
    context.response.status_code = 400
    context.response.content_type = "application/json"
    context.response.print({
      "success" => false,
      "error" => message,
    }.to_json)
  end
  
  def self.not_found(context : HTTP::Server::Context, message : String = "Not Found")
    context.response.status_code = 404
    context.response.content_type = "application/json"
    context.response.print({
      "success" => false,
      "error" => message,
    }.to_json)
  end
  
  def self.internal_error(context : HTTP::Server::Context, message : String = "Internal Server Error")
    context.response.status_code = 500
    context.response.content_type = "application/json"
    context.response.print({
      "success" => false,
      "error" => message,
    }.to_json)
  end
  
  def self.redirect(context : HTTP::Server::Context, url : String, permanent : Bool = false)
    context.response.status_code = permanent ? 301 : 302
    context.response.headers["Location"] = url
  end
end

# ใช้งาน
server = HTTP::Server.new do |context|
  case context.request.path
  when "/"
    ResponseHelper.ok(context, {"version" => "1.0"})
  when "/redirect"
    ResponseHelper.redirect(context, "/new-location")
  when "/new-location"
    ResponseHelper.ok(context, nil, "Redirected here!")
  else
    ResponseHelper.not_found(context)
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Todo API

```crystal
require "http/server"
require "json"

struct Todo
  include JSON::Serializable
  
  property id : Int32
  property title : String
  property done : Bool
  property created_at : String
  
  def initialize(@id, @title, @done = false)
    @created_at = Time.local.to_s
  end
end

class TodoApp
  def initialize
    @todos = {} of Int32 => Todo
    @next_id = 1
    @mutex = Mutex.new
  end
  
  def create(title : String) : Todo
    @mutex.synchronize do
      todo = Todo.new(@next_id, title)
      @todos[@next_id] = todo
      @next_id += 1
      todo
    end
  end
  
  def all : Array(Todo)
    @mutex.synchronize { @todos.values.sort_by(&.id) }
  end
  
  def find(id : Int32) : Todo?
    @mutex.synchronize { @todos[id]? }
  end
  
  def toggle(id : Int32) : Todo?
    @mutex.synchronize do
      return nil unless t = @todos[id]?
      @todos[id] = Todo.new(t.id, t.title, !t.done)
      @todos[id]
    end
  end
  
  def delete(id : Int32) : Bool
    @mutex.synchronize do
      @todos.delete(id)
      !@todos.has_key?(id)
    end
  end
end

app = TodoApp.new

def json(context, data, status = 200)
  context.response.status_code = status
  context.response.content_type = "application/json"
  context.response.print data.to_json
end

server = HTTP::Server.new do |context|
  path = context.request.path
  method = context.request.method
  
  case {method, path}
  when {"GET", "/todos"}
    json(context, app.all)
  
  when {"POST", "/todos"}
    body = context.request.body.try(&.gets_to_end) || ""
    if !body.empty?
      data = JSON.parse(body) rescue nil
      title = data.try { |d| d["title"]?.try(&.as_s?) }
      
      if title
        todo = app.create(title)
        json(context, todo, 201)
      else
        json(context, {"error" => "title required"}, 400)
      end
    else
      json(context, {"error" => "Body required"}, 400)
    end
  
  else
    if path.starts_with?("/todos/")
      id = path.split("/").last.to_i?
      
      unless id
        json(context, {"error" => "Invalid ID"}, 400)
        next
      end
      
      case method
      when "GET"
        if todo = app.find(id)
          json(context, todo)
        else
          json(context, {"error" => "Not found"}, 404)
        end
      when "PATCH"
        if todo = app.toggle(id)
          json(context, todo)
        else
          json(context, {"error" => "Not found"}, 404)
        end
      when "DELETE"
        if app.delete(id)
          context.response.status_code = 204
        else
          json(context, {"error" => "Not found"}, 404)
        end
      else
        json(context, {"error" => "Method not allowed"}, 405)
      end
    else
      json(context, {"error" => "Not found"}, 404)
    end
  end
end

puts "Todo API running on http://localhost:8080"
server.listen("0.0.0.0", 8080)
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HTTP::Server**: การสร้าง server พื้นฐาน
2. **HTTP::Handler**: middleware pattern สำหรับ handler chain
3. **HTTP::Context**: เข้าถึง request และ response
4. **Manual Routing**: การทำ routing ด้วยตัวเอง
5. **Dynamic Routes**: รองรับ path parameters เช่น `/users/:id`
6. **Middleware**: Logging, CORS, Authentication
7. **Response Helpers**: helper methods สำหรับ JSON responses
8. **Static Files**: serve static files
