# Part 144: REST API Design - การออกแบบ REST API

## บทนำ

REST (Representational State Transfer) เป็น architectural style สำหรับ web APIs หลักการสำคัญคือ: stateless, resource-based, และใช้ HTTP methods อย่างถูกต้อง

## Resource URLs

```
# หลักการตั้งชื่อ URL

# ✓ ถูกต้อง - ใช้ noun (คำนาม) เอกพจน์/พหูพจน์
GET    /users              # List users
GET    /users/1            # Get specific user
POST   /users              # Create user
PUT    /users/1            # Update user (full)
PATCH  /users/1            # Update user (partial)
DELETE /users/1            # Delete user

# ✓ Nested resources
GET    /users/1/posts      # User's posts
GET    /users/1/posts/5    # Specific post of user
POST   /users/1/posts      # Create post for user

# ✗ ผิด - ไม่ควรมี verbs ใน URL
GET    /getUsers           # ผิด - ใช้ GET /users
POST   /createUser         # ผิด - ใช้ POST /users
GET    /users/delete/1     # ผิด - ใช้ DELETE /users/1
```

## HTTP Methods

```crystal
require "kemal"
require "json"

# GET - ดึงข้อมูล (idempotent, safe)
get "/api/v1/users" do |env|
  env.response.content_type = "application/json"
  users = fetch_users(env)
  users.to_json
end

# POST - สร้างข้อมูล (not idempotent)
post "/api/v1/users" do |env|
  data = parse_json_body(env)
  user = create_user(data)
  
  env.response.status_code = 201
  env.response.headers["Location"] = "/api/v1/users/#{user["id"]}"
  env.response.content_type = "application/json"
  user.to_json
end

# GET - ดึงข้อมูลชิ้นเดียว
get "/api/v1/users/:id" do |env|
  id = env.params.url["id"].to_i?
  user = find_user(id)
  
  if user
    env.response.content_type = "application/json"
    user.to_json
  else
    halt env, status_code: 404, response: {error: "User not found"}.to_json
  end
end

# PUT - แทนที่ข้อมูลทั้งหมด (idempotent)
put "/api/v1/users/:id" do |env|
  id = env.params.url["id"].to_i?
  data = parse_json_body(env)
  
  user = replace_user(id, data)
  
  env.response.content_type = "application/json"
  user.to_json
end

# PATCH - อัปเดตบางส่วน (idempotent)
patch "/api/v1/users/:id" do |env|
  id = env.params.url["id"].to_i?
  data = parse_json_body(env)
  
  user = update_user(id, data)
  
  env.response.content_type = "application/json"
  user.to_json
end

# DELETE - ลบข้อมูล (idempotent)
delete "/api/v1/users/:id" do |env|
  id = env.params.url["id"].to_i?
  delete_user(id)
  
  env.response.status_code = 204
  ""
end
```

## HTTP Status Codes

```crystal
# 2xx - Success
# 200 OK - ดึงข้อมูลสำเร็จ
# 201 Created - สร้างข้อมูลสำเร็จ
# 202 Accepted - รับคำขอแล้ว (async processing)
# 204 No Content - สำเร็จแต่ไม่มีข้อมูลส่งกลับ

# 3xx - Redirection
# 301 Moved Permanently - URL เปลี่ยนถาวร
# 302 Found - Redirect ชั่วคราว
# 304 Not Modified - ข้อมูลไม่เปลี่ยน (cache valid)

# 4xx - Client Errors
# 400 Bad Request - ข้อมูลที่ส่งมาไม่ถูกต้อง
# 401 Unauthorized - ยังไม่ได้ authenticate
# 403 Forbidden - authenticate แล้วแต่ไม่มีสิทธิ์
# 404 Not Found - ไม่พบข้อมูล
# 405 Method Not Allowed - HTTP method ไม่รองรับ
# 409 Conflict - ข้อมูลซ้ำหรือขัดแย้ง
# 410 Gone - ลบถาวรแล้ว
# 422 Unprocessable Entity - Validation ล้มเหลว
# 429 Too Many Requests - Rate limit exceeded

# 5xx - Server Errors
# 500 Internal Server Error - เกิดข้อผิดพลาดที่ server
# 502 Bad Gateway - upstream server error
# 503 Service Unavailable - server ไม่พร้อมให้บริการ

def json_response(env, data, status : Int32 = 200)
  env.response.status_code = status
  env.response.content_type = "application/json"
  data.to_json
end

# ตัวอย่างการใช้ status codes
post "/api/v1/users" do |env|
  begin
    data = JSON.parse(env.request.body.try(&.gets_to_end) || "{}")
    
    unless data["email"]? && data["name"]?
      # 422 - Missing required fields
      halt env, status_code: 422,
        response: {
          error: "Validation failed",
          details: ["email is required", "name is required"]
        }.to_json
    end
    
    # ตรวจสอบ duplicate email
    if user_exists?(data["email"].as_s)
      # 409 - Conflict
      halt env, status_code: 409,
        response: {error: "Email already exists"}.to_json
    end
    
    user = create_user(data)
    
    # 201 - Created
    env.response.status_code = 201
    env.response.headers["Location"] = "/api/v1/users/#{user[:id]}"
    env.response.content_type = "application/json"
    user.to_json
    
  rescue JSON::ParseException
    # 400 - Bad JSON
    halt env, status_code: 400,
      response: {error: "Invalid JSON"}.to_json
  rescue ex
    # 500 - Server error
    STDERR.puts "Error: #{ex.message}"
    halt env, status_code: 500,
      response: {error: "Internal server error"}.to_json
  end
end
```

## Response Format

```crystal
# Standard API response format

# Success response
def success_response(data, meta = nil)
  response = {"data" => data}
  response["meta"] = meta if meta
  response
end

# Error response
def error_response(message : String, code : String? = nil, details : Array(String)? = nil)
  response = {"error" => message}
  response["code"] = code if code
  response["details"] = details if details
  response
end

# List response with pagination
def list_response(items, total : Int64, page : Int32, per_page : Int32)
  {
    "data" => items,
    "meta" => {
      "total" => total,
      "page" => page,
      "per_page" => per_page,
      "pages" => (total.to_f / per_page).ceil.to_i,
    },
    "links" => {
      "self" => "/api/v1/items?page=#{page}",
      "first" => "/api/v1/items?page=1",
      "last" => "/api/v1/items?page=#{(total.to_f / per_page).ceil.to_i}",
      "prev" => page > 1 ? "/api/v1/items?page=#{page - 1}" : nil,
      "next" => page < (total.to_f / per_page).ceil.to_i ? "/api/v1/items?page=#{page + 1}" : nil,
    }
  }
end
```

## Filtering, Sorting, Pagination

```crystal
require "kemal"

get "/api/v1/products" do |env|
  # Pagination
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = [env.params.query["per_page"]?.try(&.to_i?) || 20, 100].min
  
  # Filtering
  category = env.params.query["category"]?
  min_price = env.params.query["min_price"]?.try(&.to_f?)
  max_price = env.params.query["max_price"]?.try(&.to_f?)
  in_stock = env.params.query["in_stock"]?.try { |v| v == "true" }
  search = env.params.query["q"]?
  
  # Sorting
  sort_field = env.params.query["sort"]? || "created_at"
  sort_order = env.params.query["order"]? || "desc"
  
  # Validate sort
  valid_sort_fields = ["name", "price", "created_at", "updated_at"]
  unless valid_sort_fields.includes?(sort_field)
    halt env, status_code: 400,
      response: {
        error: "Invalid sort field",
        valid_fields: valid_sort_fields
      }.to_json
  end
  
  # Apply filters (จำลอง)
  products = filter_products(
    category: category,
    min_price: min_price,
    max_price: max_price,
    in_stock: in_stock,
    search: search
  )
  
  total = products.size.to_i64
  sorted = sort_products(products, sort_field, sort_order)
  paginated = sorted.skip((page - 1) * per_page).first(per_page)
  
  env.response.content_type = "application/json"
  list_response(paginated, total, page, per_page).to_json
end
```

## Resource Representation

```crystal
require "json"

# Full resource
struct UserFull
  include JSON::Serializable
  
  property id : Int32
  property email : String
  property name : String
  property role : String
  property bio : String?
  property avatar_url : String?
  property created_at : Time
  property updated_at : Time
  
  def initialize(@id, @email, @name, @role, @bio = nil, @avatar_url = nil)
    @created_at = Time.local
    @updated_at = Time.local
  end
end

# Summary (เฉพาะ fields จำเป็น)
struct UserSummary
  include JSON::Serializable
  
  property id : Int32
  property name : String
  property avatar_url : String?
  
  def initialize(@id, @name, @avatar_url = nil)
  end
end

# Response wrapper
struct ApiResponse(T)
  include JSON::Serializable
  
  property data : T
  property meta : Hash(String, JSON::Any)?
  
  def initialize(@data, @meta = nil)
  end
end

# ใช้งาน
get "/api/v1/users/:id" do |env|
  user = find_user(env.params.url["id"].to_i?)
  
  # ตรวจสอบว่าต้องการ fields อะไร
  fields = env.params.query["fields"]?.try(&.split(","))
  
  response = if fields && fields.includes?("summary")
    ApiResponse.new(UserSummary.new(user[:id], user[:name]))
  else
    ApiResponse.new(UserFull.new(user[:id], user[:email], user[:name], user[:role]))
  end
  
  env.response.content_type = "application/json"
  response.to_json
end
```

## HATEOAS Links

```crystal
# HATEOAS - Hypermedia as the Engine of Application State
# เพิ่ม links ใน response เพื่อบอก client ว่าทำอะไรได้บ้าง

struct UserWithLinks
  include JSON::Serializable
  
  property id : Int32
  property name : String
  property email : String
  property _links : Hash(String, String)
  
  def initialize(@id, @name, @email)
    @_links = {
      "self" => "/api/v1/users/#{@id}",
      "posts" => "/api/v1/users/#{@id}/posts",
      "update" => "/api/v1/users/#{@id}",
      "delete" => "/api/v1/users/#{@id}",
    }
  end
end
```

## Complete REST API Example

```crystal
require "kemal"
require "json"

struct Article
  include JSON::Serializable
  
  property id : Int32
  property title : String
  property content : String
  property author : String
  property published : Bool
  property created_at : String
  property updated_at : String
  
  def initialize(@id, @title, @content, @author, @published = false)
    now = Time.local.to_rfc3339
    @created_at = now
    @updated_at = now
  end
end

articles = [
  Article.new(1, "Crystal Tutorial", "เนื้อหา...", "สมชาย", true),
  Article.new(2, "Kemal Guide", "เนื้อหา Kemal...", "สมหญิง", true),
]
next_id = 3

# Helper
private def json(env, data, status = 200)
  env.response.content_type = "application/json"
  env.response.status_code = status
  data.to_json
end

private def parse_body(env) : JSON::Any?
  JSON.parse(env.request.body.try(&.gets_to_end) || "{}") rescue nil
end

# GET /api/v1/articles
get "/api/v1/articles" do |env|
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = [env.params.query["per_page"]?.try(&.to_i?) || 10, 50].min
  q = env.params.query["q"]?
  author = env.params.query["author"]?
  published_only = env.params.query["published"]?.try { |v| v == "true" }
  
  filtered = articles.select do |a|
    (q.nil? || a.title.downcase.includes?(q.downcase)) &&
    (author.nil? || a.author == author) &&
    (!published_only || a.published)
  end
  
  total = filtered.size.to_i64
  offset = (page - 1) * per_page
  paginated = filtered[offset, per_page]? || [] of Article
  
  json(env, {
    "data" => paginated,
    "meta" => {
      "total" => total,
      "page" => page,
      "per_page" => per_page,
      "pages" => (total.to_f / per_page).ceil.to_i,
    }
  })
end

# GET /api/v1/articles/:id
get "/api/v1/articles/:id" do |env|
  id = env.params.url["id"].to_i?
  article = articles.find { |a| a.id == id }
  
  article ? json(env, article) : halt(env, status_code: 404, response: {error: "Not found"}.to_json)
end

# POST /api/v1/articles
post "/api/v1/articles" do |env|
  data = parse_body(env) || halt(env, status_code: 400, response: {error: "Invalid JSON"}.to_json)
  
  title = data["title"]?.try(&.as_s?)
  content = data["content"]?.try(&.as_s?)
  author = data["author"]?.try(&.as_s?)
  
  errors = [] of String
  errors << "title is required" unless title
  errors << "content must be at least 10 chars" if content && content.size < 10
  errors << "author is required" unless author
  
  unless errors.empty?
    halt env, status_code: 422, response: {error: "Validation failed", details: errors}.to_json
  end
  
  article = Article.new(next_id, title.not_nil!, content.not_nil!, author.not_nil!)
  articles << article
  next_id += 1
  
  env.response.headers["Location"] = "/api/v1/articles/#{article.id}"
  json(env, article, 201)
end

# PATCH /api/v1/articles/:id
patch "/api/v1/articles/:id" do |env|
  id = env.params.url["id"].to_i?
  idx = articles.index { |a| a.id == id }
  
  halt env, status_code: 404, response: {error: "Not found"}.to_json unless idx
  
  data = parse_body(env) || halt(env, status_code: 400, response: {error: "Invalid JSON"}.to_json)
  
  existing = articles[idx]
  articles[idx] = Article.new(
    existing.id,
    data["title"]?.try(&.as_s?) || existing.title,
    data["content"]?.try(&.as_s?) || existing.content,
    existing.author,
    data["published"]?.try(&.as_bool?) || existing.published
  )
  
  json(env, articles[idx])
end

# DELETE /api/v1/articles/:id
delete "/api/v1/articles/:id" do |env|
  id = env.params.url["id"].to_i?
  before_count = articles.size
  articles.reject! { |a| a.id == id }
  
  if articles.size < before_count
    env.response.status_code = 204
    ""
  else
    halt env, status_code: 404, response: {error: "Not found"}.to_json
  end
end

# Error handlers
error 404 do |env|
  env.response.content_type = "application/json"
  {error: "Endpoint not found"}.to_json
end

error 500 do |env|
  env.response.content_type = "application/json"
  {error: "Internal server error"}.to_json
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Order API

```crystal
# สร้าง REST API สำหรับระบบสั่งซื้อ
# GET    /api/v1/orders
# GET    /api/v1/orders/:id
# POST   /api/v1/orders
# PATCH  /api/v1/orders/:id/status (cancel, confirm, ship, deliver)
# DELETE /api/v1/orders/:id

struct OrderItem
  include JSON::Serializable
  property product_id : Int32
  property quantity : Int32
  property price : Float64
end

struct Order
  include JSON::Serializable
  property id : Int32
  property customer_id : Int32
  property items : Array(OrderItem)
  property status : String  # pending, confirmed, shipped, delivered, cancelled
  property total : Float64
  property created_at : String
  
  def initialize(@id, @customer_id, @items)
    @status = "pending"
    @total = items.sum { |i| i.price * i.quantity }
    @created_at = Time.local.to_rfc3339
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Resource URLs**: ใช้ noun ไม่ใช่ verb
2. **HTTP Methods**: GET, POST, PUT, PATCH, DELETE
3. **Status Codes**: 2xx, 4xx, 5xx
4. **Response Format**: data, meta, links
5. **Filtering**: query params
6. **Sorting**: sort + order params
7. **Pagination**: page + per_page + links
8. **Resource Representation**: full vs summary
9. **HATEOAS**: hypermedia links
10. **Error Handling**: validation errors, not found ฯลฯ

REST API ที่ดีต้องสม่ำเสมอ, เข้าใจง่าย และ predictable
