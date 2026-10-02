# Part 132: Kemal Routes - การจัดการ Routes ใน Kemal

## บทนำ

Routes ใน Kemal กำหนดว่าจะตอบสนองต่อ HTTP request อย่างไร ในบทนี้เราจะเรียนรู้เรื่อง URL parameters, query params, wildcard routes, route groups, และ before/after filters

## Route Parameters (:id)

```crystal
require "kemal"

# URL parameters ด้วย :param_name
get "/users/:id" do |env|
  id = env.params.url["id"]
  "User ID: #{id}"
end

get "/posts/:post_id/comments/:comment_id" do |env|
  post_id = env.params.url["post_id"]
  comment_id = env.params.url["comment_id"]
  "Post: #{post_id}, Comment: #{comment_id}"
end

# แปลง type
get "/products/:id" do |env|
  id = env.params.url["id"].to_i?
  
  if id
    "Product #{id}"
  else
    halt env, status_code: 400, response: "Invalid ID"
  end
end

Kemal.run
```

## Query Parameters

```crystal
require "kemal"

# Query params: GET /search?q=crystal&page=2
get "/search" do |env|
  q = env.params.query["q"]?
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = env.params.query["per_page"]?.try(&.to_i?) || 20
  sort = env.params.query["sort"]? || "created_at"
  order = env.params.query["order"]? || "desc"
  
  env.response.content_type = "application/json"
  {
    "query" => q || "",
    "page" => page,
    "per_page" => per_page,
    "sort" => sort,
    "order" => order,
  }.to_json
end

# หลาย values สำหรับ key เดียวกัน: GET /items?ids[]=1&ids[]=2&ids[]=3
get "/items" do |env|
  ids = env.params.query["ids[]"]?
  "IDs: #{ids}"
end

# Optional params
get "/filter" do |env|
  filters = {} of String => String
  
  ["category", "brand", "min_price", "max_price", "in_stock"].each do |param|
    if value = env.params.query[param]?
      filters[param] = value
    end
  end
  
  env.response.content_type = "application/json"
  {"filters" => filters}.to_json
end
```

## Wildcard Routes

```crystal
require "kemal"

# Wildcard: จับทุก path ที่เริ่มด้วย /static/
get "/static/*" do |env|
  path = env.params.url["glob"]? || ""
  "Static file: #{path}"
end

# Wildcard สำหรับ catch-all
get "/*" do |env|
  path = env.params.url["glob"]? || ""
  "Not found: /#{path}"
end
```

## Route Groups

```crystal
require "kemal"
require "json"

# Route Groups (prefix)
# Kemal ไม่มี built-in route group แต่เราสามารถจัดการได้ด้วย helper

module Routes
  module API
    module V1
      def self.setup
        # GET /api/v1/users
        get "/api/v1/users" do |env|
          env.response.content_type = "application/json"
          [{"id" => 1, "name" => "สมชาย"}].to_json
        end
        
        # GET /api/v1/users/:id
        get "/api/v1/users/:id" do |env|
          id = env.params.url["id"]
          env.response.content_type = "application/json"
          {"id" => id.to_i?, "name" => "User #{id}"}.to_json
        end
        
        # POST /api/v1/users
        post "/api/v1/users" do |env|
          body = env.request.body.try(&.gets_to_end) || "{}"
          data = JSON.parse(body)
          env.response.status_code = 201
          env.response.content_type = "application/json"
          {"created" => true, "data" => data}.to_json
        end
      end
    end
    
    module V2
      def self.setup
        get "/api/v2/users" do |env|
          env.response.content_type = "application/json"
          {
            "data" => [{"id" => 1, "name" => "สมชาย"}],
            "meta" => {"version" => "v2", "total" => 1},
          }.to_json
        end
      end
    end
  end
end

# ตั้งค่า routes
Routes::API::V1.setup
Routes::API::V2.setup

Kemal.run
```

## Before/After Filters

```crystal
require "kemal"
require "json"

# Before filter - ทำงานก่อนทุก routes
before_all do |env|
  puts "[#{Time.local}] #{env.request.method} #{env.request.path}"
  env.set("start_time", Time.monotonic)
end

# Before filter สำหรับ path pattern
before_get "/api/*" do |env|
  # ตรวจสอบ Authorization สำหรับ API routes
  token = env.request.headers["Authorization"]?
  
  unless token == "Bearer valid-token"
    halt env, status_code: 401, response: {"error" => "Unauthorized"}.to_json
  end
end

before_post "/api/*" do |env|
  # ตรวจสอบ Content-Type
  content_type = env.request.headers["Content-Type"]? || ""
  
  unless content_type.includes?("application/json")
    halt env, status_code: 415, response: {"error" => "Content-Type must be application/json"}.to_json
  end
end

# After filter - ทำงานหลังทุก routes
after_all do |env|
  if start = env.get?("start_time")
    elapsed = (Time.monotonic - start.as(Time::Span)).total_milliseconds
    env.response.headers["X-Response-Time"] = "#{elapsed.round(2)}ms"
  end
end

# Routes
get "/" do
  "Public route - no auth needed"
end

get "/api/data" do |env|
  env.response.content_type = "application/json"
  {"data" => "Protected data", "user" => "authenticated"}.to_json
end

post "/api/items" do |env|
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  env.response.status_code = 201
  env.response.content_type = "application/json"
  {"created" => data}.to_json
end

Kemal.run
```

## Named Routes

```crystal
require "kemal"

# สร้าง helper สำหรับ named routes
class NamedRoutes
  @@routes = {} of String => String
  
  def self.register(name : String, path : String)
    @@routes[name] = path
  end
  
  def self.url_for(name : String, params : Hash(String, String) = {} of String => String) : String
    path = @@routes[name]? || raise "Unknown route: #{name}"
    
    params.each { |key, value| path = path.gsub(":#{key}", value) }
    path
  end
end

# ลงทะเบียน routes
NamedRoutes.register("home", "/")
NamedRoutes.register("user_show", "/users/:id")
NamedRoutes.register("post_show", "/posts/:post_id/comments/:comment_id")

get "/" do
  "Home"
end

get "/users/:id" do |env|
  id = env.params.url["id"]
  profile_url = NamedRoutes.url_for("user_show", {"id" => id})
  "User #{id} - URL: #{profile_url}"
end

# ใช้งาน
puts NamedRoutes.url_for("user_show", {"id" => "42"})  # /users/42
puts NamedRoutes.url_for("home")  # /

Kemal.run
```

## Route Constraints

```crystal
require "kemal"

# จำกัด route ด้วย constraints
def integer_route(env, param : String, &block : Int32 ->)
  id = env.params.url[param].to_i?
  
  if id
    block.call(id)
  else
    halt env, status_code: 400, response: {"error" => "#{param} must be an integer"}.to_json
  end
end

get "/products/:id" do |env|
  integer_route(env, "id") do |id|
    env.response.content_type = "application/json"
    {"id" => id, "name" => "Product #{id}"}.to_json
  end
end

# Route ที่รับแค่ lowercase letters
get "/categories/:slug" do |env|
  slug = env.params.url["slug"]
  
  unless slug.matches?(/\A[a-z][a-z0-9-]*\z/)
    halt env, status_code: 400, response: "Invalid slug format"
  end
  
  "Category: #{slug}"
end
```

## Nested Resources

```crystal
require "kemal"
require "json"

# Nested resources: /users/:user_id/posts/:post_id

# สร้าง data structures
struct Post
  include JSON::Serializable
  property id : Int32
  property user_id : Int32
  property title : String
  property content : String
  
  def initialize(@id, @user_id, @title, @content)
  end
end

POSTS = [
  Post.new(1, 1, "First Post", "Content 1"),
  Post.new(2, 1, "Second Post", "Content 2"),
  Post.new(3, 2, "Another Post", "Content 3"),
]

# GET /users/:user_id/posts
get "/users/:user_id/posts" do |env|
  user_id = env.params.url["user_id"].to_i?
  
  unless user_id
    halt env, status_code: 400, response: "Invalid user ID"
  end
  
  user_posts = POSTS.select { |p| p.user_id == user_id }
  env.response.content_type = "application/json"
  user_posts.to_json
end

# GET /users/:user_id/posts/:post_id
get "/users/:user_id/posts/:post_id" do |env|
  user_id = env.params.url["user_id"].to_i?
  post_id = env.params.url["post_id"].to_i?
  
  unless user_id && post_id
    halt env, status_code: 400, response: "Invalid IDs"
  end
  
  post = POSTS.find { |p| p.user_id == user_id && p.id == post_id }
  
  if post
    env.response.content_type = "application/json"
    post.to_json
  else
    halt env, status_code: 404, response: {"error" => "Post not found"}.to_json
  end
end

Kemal.run
```

## Route Versioning

```crystal
require "kemal"

# API Versioning ด้วย path prefix
CURRENT_API_VERSION = "v2"

# V1 API
get "/api/v1/status" do |env|
  env.response.content_type = "application/json"
  {"version" => "1", "status" => "deprecated", "message" => "Please upgrade to v2"}.to_json
end

# V2 API
get "/api/v2/status" do |env|
  env.response.content_type = "application/json"
  {"version" => "2", "status" => "ok"}.to_json
end

# Latest version redirect
get "/api/status" do |env|
  env.redirect "/api/#{CURRENT_API_VERSION}/status"
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: RESTful Resource Routes

```crystal
require "kemal"
require "json"

# สร้าง RESTful routes สำหรับ Product resource
struct Product
  include JSON::Serializable
  
  property id : Int32
  property name : String
  property price : Float64
  property category : String
  property in_stock : Bool
  property created_at : String
  
  def initialize(@id, @name, @price, @category, @in_stock = true)
    @created_at = Time.local.to_rfc3339
  end
end

products = [
  Product.new(1, "Crystal Book", 299.0, "books"),
  Product.new(2, "Kemal T-Shirt", 499.0, "clothing"),
  Product.new(3, "Crystal Mug", 199.0, "accessories"),
]
next_id = 4

def json(env, data, status = 200)
  env.response.content_type = "application/json"
  env.response.status_code = status
  data.to_json
end

# Index: GET /products
get "/products" do |env|
  category = env.params.query["category"]?
  search = env.params.query["q"]?
  min_price = env.params.query["min_price"]?.try(&.to_f?)
  max_price = env.params.query["max_price"]?.try(&.to_f?)
  
  filtered = products.select do |p|
    (category.nil? || p.category == category) &&
    (search.nil? || p.name.downcase.includes?(search.downcase)) &&
    (min_price.nil? || p.price >= min_price) &&
    (max_price.nil? || p.price <= max_price)
  end
  
  json(env, {"products" => filtered, "total" => filtered.size})
end

# Show: GET /products/:id
get "/products/:id" do |env|
  id = env.params.url["id"].to_i?
  product = products.find { |p| p.id == id }
  product ? json(env, product) : halt(env, status_code: 404, response: {"error" => "Not found"}.to_json)
end

# Create: POST /products
post "/products" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body) rescue halt(env, status_code: 400, response: "Bad JSON")
  
  name = data["name"]?.try(&.as_s?)
  price = data["price"]?.try(&.as_f?)
  category = data["category"]?.try(&.as_s?)
  
  unless name && price && category
    halt env, status_code: 422, response: {"error" => "name, price, category required"}.to_json
  end
  
  product = Product.new(next_id, name, price, category)
  products << product
  next_id += 1
  json(env, product, 201)
end

# Update: PUT /products/:id
put "/products/:id" do |env|
  id = env.params.url["id"].to_i?
  idx = products.index { |p| p.id == id }
  halt env, status_code: 404, response: {"error" => "Not found"}.to_json unless idx
  
  body = env.request.body.try(&.gets_to_end) || ""
  data = JSON.parse(body) rescue halt(env, status_code: 400, response: "Bad JSON")
  
  p = products[idx]
  name = data["name"]?.try(&.as_s?) || p.name
  price = data["price"]?.try(&.as_f?) || p.price
  category = data["category"]?.try(&.as_s?) || p.category
  in_stock = data["in_stock"]?.try(&.as_bool?) != nil ? data["in_stock"].as_bool : p.in_stock
  
  products[idx] = Product.new(id.not_nil!, name, price, category, in_stock)
  json(env, products[idx])
end

# Destroy: DELETE /products/:id
delete "/products/:id" do |env|
  id = env.params.url["id"].to_i?
  old_size = products.size
  products.reject! { |p| p.id == id }
  
  if products.size < old_size
    env.response.status_code = 204
    ""
  else
    halt env, status_code: 404, response: {"error" => "Not found"}.to_json
  end
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Route Parameters**: `env.params.url["id"]`
2. **Query Parameters**: `env.params.query["key"]`
3. **Wildcard Routes**: `get "/static/*"`
4. **Route Groups**: จัดกลุ่ม routes ด้วย module
5. **Before Filters**: `before_all`, `before_get`, `before_post`
6. **After Filters**: `after_all`
7. **Named Routes**: helper สำหรับสร้าง URLs
8. **Route Constraints**: ตรวจสอบ format ของ params
9. **Nested Resources**: `/users/:user_id/posts/:post_id`
10. **API Versioning**: prefix `/api/v1/`, `/api/v2/`
11. **RESTful Routes**: index, show, create, update, destroy
