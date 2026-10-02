# Part 145: API Versioning - การจัดการ Version ใน API

## บทนำ

API Versioning ช่วยให้เราสามารถเปลี่ยนแปลง API โดยไม่กระทบ clients เดิม มีหลายวิธีในการทำ versioning

## 1. URL Path Versioning

```crystal
require "kemal"
require "json"

# /api/v1/users
# /api/v2/users

# V1 API
get "/api/v1/users" do |env|
  users = UserStore.all.map do |u|
    {id: u.id, name: u.name, email: u.email}
  end
  
  env.response.content_type = "application/json"
  users.to_json
end

# V2 API - เพิ่ม role และ pagination
get "/api/v2/users" do |env|
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = 20
  
  all_users = UserStore.all
  total = all_users.size.to_i64
  
  users = all_users.skip((page - 1) * per_page).first(per_page).map do |u|
    {id: u.id, name: u.name, email: u.email, role: u.role, created_at: u.created_at}
  end
  
  env.response.content_type = "application/json"
  {
    "data" => users,
    "meta" => {
      "total" => total,
      "page" => page,
      "per_page" => per_page,
    }
  }.to_json
end

# Redirect v1 endpoints ที่ deprecated
get "/api/v1/profile" do |env|
  env.response.headers["Deprecation"] = "true"
  env.response.headers["Sunset"] = "2025-12-31T00:00:00Z"
  env.response.headers["Link"] = "</api/v2/profile>; rel=\"successor-version\""
  env.redirect "/api/v2/profile", 301
end

Kemal.run
```

## Route Organization

```crystal
require "kemal"
require "json"

# จัดกลุ่ม routes ด้วย module
module API
  module V1
    def self.setup
      get "/api/v1/products" do |env|
        env.response.content_type = "application/json"
        ProductService.list_v1.to_json
      end
      
      get "/api/v1/products/:id" do |env|
        id = env.params.url["id"].to_i?
        product = ProductService.find_v1(id)
        product ? (env.response.content_type = "application/json"; product.to_json) :
          halt(env, status_code: 404, response: {error: "Not found"}.to_json)
      end
      
      post "/api/v1/products" do |env|
        body = env.request.body.try(&.gets_to_end) || "{}"
        data = JSON.parse(body)
        product = ProductService.create_v1(data)
        
        env.response.status_code = 201
        env.response.content_type = "application/json"
        product.to_json
      end
    end
  end
  
  module V2
    def self.setup
      get "/api/v2/products" do |env|
        # V2 มี additional fields และ pagination
        page = env.params.query["page"]?.try(&.to_i?) || 1
        per_page = env.params.query["per_page"]?.try(&.to_i?) || 20
        sort = env.params.query["sort"]? || "name"
        
        result = ProductService.list_v2(page: page, per_page: per_page, sort: sort)
        
        env.response.content_type = "application/json"
        result.to_json
      end
      
      get "/api/v2/products/:id" do |env|
        id = env.params.url["id"].to_i?
        product = ProductService.find_v2(id)
        
        if product
          env.response.content_type = "application/json"
          product.to_json
        else
          halt env, status_code: 404, response: {error: "Not found"}.to_json
        end
      end
    end
  end
end

# Setup routes
API::V1.setup
API::V2.setup

# Latest version shortcut
get "/api/products" do |env|
  env.response.headers["API-Version"] = "2"
  env.redirect "/api/v2/products#{env.request.query ? "?#{env.request.query}" : ""}"
end

Kemal.run
```

## 2. Header-based Versioning

```crystal
require "kemal"

# Accept: application/vnd.myapp.v1+json
# Accept: application/vnd.myapp.v2+json

before_all "/api/*" do |env|
  version = extract_version(env)
  env.set("api_version", version)
end

def extract_version(env : HTTP::Server::Context) : String
  # ลอง Accept header ก่อน
  if accept = env.request.headers["Accept"]?
    if m = accept.match(/vnd\.myapp\.v(\d+)\+json/)
      return m[1]
    end
  end
  
  # ลอง X-API-Version header
  if version = env.request.headers["X-API-Version"]?
    return version
  end
  
  # Default version
  "2"
end

get "/api/users" do |env|
  version = env.get("api_version").as(String)
  
  case version
  when "1"
    # V1 response format
    env.response.content_type = "application/vnd.myapp.v1+json"
    {"users" => UserStore.all.map { |u| {id: u.id, name: u.name} }}.to_json
  when "2"
    # V2 response format
    env.response.content_type = "application/vnd.myapp.v2+json"
    {
      "data" => UserStore.all.map { |u| {id: u.id, name: u.name, role: u.role} },
      "version" => "2"
    }.to_json
  else
    halt env, status_code: 400,
      response: {error: "Unsupported API version: #{version}", supported: ["1", "2"]}.to_json
  end
end

Kemal.run
```

## 3. Query Parameter Versioning

```crystal
require "kemal"

before_all "/api/*" do |env|
  version = env.params.query["version"]? || env.params.query["v"]? || "2"
  env.set("api_version", version)
end

get "/api/items" do |env|
  version = env.get("api_version").as(String)
  
  env.response.content_type = "application/json"
  
  case version
  when "1"
    [{id: 1, name: "Item 1"}, {id: 2, name: "Item 2"}].to_json
  when "2"
    {
      "items" => [{id: 1, name: "Item 1", created_at: Time.local.to_rfc3339}],
      "version" => "2"
    }.to_json
  else
    halt env, status_code: 400,
      response: {error: "Unknown version"}.to_json
  end
end

Kemal.run
```

## Version Router Class

```crystal
require "kemal"

class APIRouter
  @@routes = {} of Tuple(String, String, String) => Proc(HTTP::Server::Context, String)
  
  def self.register(version : String, method : String, path : String, &handler : HTTP::Server::Context -> String)
    @@routes[{version, method, path}] = handler
  end
  
  def self.handle(env : HTTP::Server::Context) : String?
    version = env.request.headers["X-API-Version"]? || "2"
    method = env.request.method
    path = env.request.path
    
    if handler = @@routes[{version, method, path}]?
      handler.call(env)
    elsif handler = @@routes[{"latest", method, path}]?
      handler.call(env)
    else
      nil
    end
  end
end

# ลงทะเบียน handlers
APIRouter.register("1", "GET", "/api/users") do |env|
  [{id: 1, name: "User 1"}].to_json
end

APIRouter.register("2", "GET", "/api/users") do |env|
  {
    "data" => [{id: 1, name: "User 1", role: "admin"}],
    "meta" => {"total" => 1}
  }.to_json
end

APIRouter.register("latest", "GET", "/api/users") do |env|
  # Same as v2
  {
    "data" => [{id: 1, name: "User 1", role: "admin"}],
    "meta" => {"total" => 1},
    "api_version" => "2"
  }.to_json
end
```

## Deprecation Notices

```crystal
require "kemal"

# Middleware สำหรับ deprecation warnings
before_all "/api/v1/*" do |env|
  env.response.headers["Deprecation"] = "true"
  env.response.headers["Sunset"] = "2025-06-30T23:59:59Z"
  env.response.headers["Link"] = [
    "</api/v2#{env.request.path[7..]}> rel=\"successor-version\"",
    "<https://docs.example.com/migration>; rel=\"deprecation\""
  ].join(", ")
  env.response.headers["Warning"] = "299 - \"This API version is deprecated. Migrate to v2.\""
end

# Legacy support
class LegacyAdapter
  # แปลง V2 format เป็น V1 format
  def self.to_v1_user(user : Hash) : Hash
    {
      "id" => user["id"],
      "username" => user["name"],  # V1 ใช้ username แทน name
      "mail" => user["email"],     # V1 ใช้ mail แทน email
    }
  end
  
  # แปลง V1 input เป็น V2 format
  def self.from_v1_input(data : JSON::Any) : Hash(String, JSON::Any)
    {
      "name" => data["username"]? || data["name"]?,
      "email" => data["mail"]? || data["email"]?,
    }.compact.transform_values { |v| v.not_nil! }
  end
end

get "/api/v1/users" do |env|
  users = UserStore.all.map { |u|
    LegacyAdapter.to_v1_user({"id" => u.id, "name" => u.name, "email" => u.email})
  }
  
  env.response.content_type = "application/json"
  users.to_json
end
```

## API Version Negotiation

```crystal
require "kemal"

class VersionNegotiator
  VERSIONS = ["3", "2", "1"]
  LATEST_VERSION = "3"
  MIN_VERSION = "1"
  
  def self.negotiate(env : HTTP::Server::Context) : String
    requested = extract_version(env)
    
    if VERSIONS.includes?(requested)
      requested
    elsif requested == "latest"
      LATEST_VERSION
    else
      # ใช้ version ใกล้เคียงที่สุด
      LATEST_VERSION
    end
  end
  
  private def self.extract_version(env) : String
    # Priority order:
    # 1. URL path (/api/v2/...)
    # 2. Accept header
    # 3. X-API-Version header
    # 4. query parameter
    # 5. Default
    
    path = env.request.path
    if m = path.match(/\/api\/v(\d+)\//)
      return m[1]
    end
    
    if accept = env.request.headers["Accept"]?
      if m = accept.match(/version=(\d+)/)
        return m[1]
      end
    end
    
    env.request.headers["X-API-Version"]? ||
      env.params.query["v"]? ||
      LATEST_VERSION
  end
end

before_all "/api/*" do |env|
  version = VersionNegotiator.negotiate(env)
  env.set("version", version)
  env.response.headers["X-API-Version"] = version
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Migration Guide API

```crystal
# สร้าง API ที่รองรับทั้ง V1 และ V2 พร้อม migration guide

# V1: GET /api/v1/orders -> Array ตรงๆ
# V2: GET /api/v2/orders -> {data, meta, links}

struct OrderV1
  include JSON::Serializable
  property id : Int32
  property customer : String
  property amount : Float64
  property status : String
  
  def initialize(@id, @customer, @amount, @status)
  end
end

struct OrderV2
  include JSON::Serializable
  property id : Int32
  property customer_name : String  # V2 เปลี่ยนชื่อ field
  property total_amount : Float64  # V2 เปลี่ยนชื่อ field
  property status : String
  property created_at : String
  property _links : Hash(String, String)
  
  def initialize(@id, @customer_name, @total_amount, @status)
    @created_at = Time.local.to_rfc3339
    @_links = {
      "self" => "/api/v2/orders/#{@id}",
      "cancel" => "/api/v2/orders/#{@id}/cancel",
    }
  end
end

orders_data = [
  {id: 1, customer: "สมชาย", amount: 1500.0, status: "pending"},
  {id: 2, customer: "สมหญิง", amount: 2300.0, status: "completed"},
]

get "/api/v1/orders" do |env|
  # V1: simple array
  result = orders_data.map { |o|
    OrderV1.new(o[:id], o[:customer], o[:amount], o[:status])
  }
  
  env.response.content_type = "application/json"
  result.to_json
end

get "/api/v2/orders" do |env|
  # V2: wrapped response with pagination
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = 20
  
  result = orders_data.map { |o|
    OrderV2.new(o[:id], o[:customer], o[:amount], o[:status])
  }
  
  env.response.content_type = "application/json"
  {
    "data" => result,
    "meta" => {"total" => result.size, "page" => page, "per_page" => per_page},
    "links" => {"docs" => "https://docs.example.com/api/v2/orders"}
  }.to_json
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **URL Path Versioning**: `/api/v1/`, `/api/v2/`
2. **Header Versioning**: `Accept: application/vnd.app.v2+json`
3. **Query Parameter**: `?version=2` หรือ `?v=2`
4. **Route Organization**: module-based route grouping
5. **Deprecation Headers**: Deprecation, Sunset, Link
6. **Legacy Adapters**: แปลง format ระหว่าง versions
7. **Version Negotiation**: เลือก version อัตโนมัติ
8. **Migration Guide**: ช่วย clients migrate

URL path versioning เป็น approach ที่ใช้งานบ่อยที่สุดเพราะ explicit และง่ายต่อ debugging
