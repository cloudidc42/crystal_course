# Part 146: API Documentation - การทำ API Documentation

## บทนำ

API Documentation ที่ดีช่วยให้ developers เข้าใจและใช้งาน API ได้อย่างถูกต้อง OpenAPI (Swagger) เป็น standard ที่นิยมใช้กันมากที่สุด

## OpenAPI Specification

```yaml
# openapi.yaml
openapi: "3.0.3"
info:
  title: "Crystal API"
  description: "API สำหรับ Crystal Web Application"
  version: "1.0.0"
  contact:
    name: "API Support"
    email: "api@example.com"
  license:
    name: "MIT"

servers:
  - url: "https://api.example.com/v1"
    description: "Production"
  - url: "https://api-staging.example.com/v1"
    description: "Staging"
  - url: "http://localhost:3000/api/v1"
    description: "Development"

tags:
  - name: "users"
    description: "User management"
  - name: "posts"
    description: "Blog posts"
  - name: "auth"
    description: "Authentication"

paths:
  /users:
    get:
      tags: ["users"]
      summary: "List users"
      description: "ดึงรายการผู้ใช้ทั้งหมด รองรับ pagination และ filtering"
      operationId: "listUsers"
      security:
        - bearerAuth: []
      parameters:
        - name: page
          in: query
          description: "หมายเลขหน้า"
          schema:
            type: integer
            minimum: 1
            default: 1
        - name: per_page
          in: query
          description: "จำนวนต่อหน้า"
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: q
          in: query
          description: "ค้นหาจากชื่อหรืออีเมล"
          schema:
            type: string
        - name: role
          in: query
          description: "กรองตาม role"
          schema:
            type: string
            enum: [admin, user, moderator]
      responses:
        "200":
          description: "สำเร็จ"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserListResponse"
        "401":
          $ref: "#/components/responses/Unauthorized"
        "403":
          $ref: "#/components/responses/Forbidden"
    
    post:
      tags: ["users"]
      summary: "Create user"
      description: "สร้างผู้ใช้ใหม่"
      operationId: "createUser"
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/CreateUserRequest"
            examples:
              basic:
                summary: "ตัวอย่างพื้นฐาน"
                value:
                  email: "user@example.com"
                  name: "สมชาย ใจดี"
                  password: "SecurePass123!"
      responses:
        "201":
          description: "สร้างสำเร็จ"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserResponse"
        "422":
          $ref: "#/components/responses/ValidationError"

  /users/{id}:
    parameters:
      - name: id
        in: path
        required: true
        description: "User ID"
        schema:
          type: integer
          format: int64
    
    get:
      tags: ["users"]
      summary: "Get user"
      operationId: "getUser"
      responses:
        "200":
          description: "สำเร็จ"
          content:
            application/json:
              schema:
                $ref: "#/components/schemas/UserResponse"
        "404":
          $ref: "#/components/responses/NotFound"
    
    patch:
      tags: ["users"]
      summary: "Update user"
      operationId: "updateUser"
      requestBody:
        content:
          application/json:
            schema:
              $ref: "#/components/schemas/UpdateUserRequest"
      responses:
        "200":
          description: "อัปเดตสำเร็จ"
        "404":
          $ref: "#/components/responses/NotFound"
    
    delete:
      tags: ["users"]
      summary: "Delete user"
      operationId: "deleteUser"
      responses:
        "204":
          description: "ลบสำเร็จ"
        "404":
          $ref: "#/components/responses/NotFound"

components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
  
  schemas:
    User:
      type: object
      properties:
        id:
          type: integer
          format: int64
          example: 1
        email:
          type: string
          format: email
          example: "user@example.com"
        name:
          type: string
          example: "สมชาย ใจดี"
        role:
          type: string
          enum: [admin, user, moderator]
          example: "user"
        active:
          type: boolean
          example: true
        created_at:
          type: string
          format: date-time
        updated_at:
          type: string
          format: date-time
      required: [id, email, name, role, active, created_at, updated_at]
    
    UserResponse:
      type: object
      properties:
        data:
          $ref: "#/components/schemas/User"
    
    UserListResponse:
      type: object
      properties:
        data:
          type: array
          items:
            $ref: "#/components/schemas/User"
        meta:
          $ref: "#/components/schemas/PaginationMeta"
    
    CreateUserRequest:
      type: object
      required: [email, name, password]
      properties:
        email:
          type: string
          format: email
        name:
          type: string
          minLength: 2
          maxLength: 100
        password:
          type: string
          minLength: 8
          format: password
        role:
          type: string
          enum: [admin, user, moderator]
          default: user
    
    UpdateUserRequest:
      type: object
      properties:
        name:
          type: string
        bio:
          type: string
    
    PaginationMeta:
      type: object
      properties:
        total:
          type: integer
        page:
          type: integer
        per_page:
          type: integer
        pages:
          type: integer
    
    Error:
      type: object
      properties:
        error:
          type: string
        message:
          type: string
        details:
          type: array
          items:
            type: string
    
    ValidationError:
      type: object
      properties:
        error:
          type: string
          example: "Validation failed"
        details:
          type: array
          items:
            type: string
          example: ["email is required", "name is too short"]
  
  responses:
    NotFound:
      description: "ไม่พบข้อมูล"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
          example:
            error: "Not Found"
            message: "ไม่พบผู้ใช้ที่ระบุ"
    
    Unauthorized:
      description: "ยังไม่ได้ authenticate"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
    
    Forbidden:
      description: "ไม่มีสิทธิ์"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/Error"
    
    ValidationError:
      description: "Validation ล้มเหลว"
      content:
        application/json:
          schema:
            $ref: "#/components/schemas/ValidationError"
```

## Swagger UI ใน Kemal

```crystal
require "kemal"

# Serve Swagger UI
get "/api/docs" do |env|
  env.response.content_type = "text/html"
  <<-HTML
  <!DOCTYPE html>
  <html>
    <head>
      <title>API Documentation</title>
      <meta charset="utf-8"/>
      <meta name="viewport" content="width=device-width, initial-scale=1">
      <link rel="stylesheet" type="text/css" href="https://unpkg.com/swagger-ui-dist@5/swagger-ui.css">
    </head>
    <body>
      <div id="swagger-ui"></div>
      <script src="https://unpkg.com/swagger-ui-dist@5/swagger-ui-bundle.js"></script>
      <script>
        SwaggerUIBundle({
          url: "/api/openapi.yaml",
          dom_id: '#swagger-ui',
          presets: [
            SwaggerUIBundle.presets.apis,
            SwaggerUIBundle.SwaggerUIStandalonePreset
          ],
          layout: "BaseLayout",
          deepLinking: true,
          tryItOutEnabled: true,
        })
      </script>
    </body>
  </html>
  HTML
end

# Serve OpenAPI spec
get "/api/openapi.yaml" do |env|
  env.response.content_type = "application/yaml"
  File.read("openapi.yaml")
end

get "/api/openapi.json" do |env|
  env.response.content_type = "application/json"
  # แปลง YAML เป็น JSON
  require "yaml"
  yaml_content = File.read("openapi.yaml")
  YAML.parse(yaml_content).to_json
end

Kemal.run
```

## Auto-generated Docs

```crystal
require "kemal"

# กำหนด doc metadata สำหรับแต่ละ route
module RouteDoc
  @@docs = [] of NamedTuple(
    method: String,
    path: String,
    summary: String,
    description: String,
    tags: Array(String),
    params: Array(NamedTuple(name: String, in: String, description: String, required: Bool, type: String)),
    responses: Hash(Int32, String)
  )
  
  def self.add(
    method : String,
    path : String,
    summary : String,
    description : String = "",
    tags : Array(String) = [] of String,
    params = [] of NamedTuple(name: String, in: String, description: String, required: Bool, type: String),
    responses : Hash(Int32, String) = {200 => "สำเร็จ"}
  )
    @@docs << {
      method: method,
      path: path,
      summary: summary,
      description: description,
      tags: tags,
      params: params,
      responses: responses
    }
  end
  
  def self.generate_openapi : Hash
    paths = {} of String => Hash(String, JSON::Any)
    
    @@docs.each do |doc|
      path_key = doc[:path]
      method_key = doc[:method].downcase
      
      paths[path_key] ||= {} of String => JSON::Any
      
      operation = {
        "summary" => JSON::Any.new(doc[:summary]),
        "tags" => JSON::Any.new(doc[:tags].map { |t| JSON::Any.new(t) }),
        "description" => JSON::Any.new(doc[:description]),
      } of String => JSON::Any
      
      paths[path_key][method_key] = JSON::Any.new(operation)
    end
    
    {
      "openapi" => "3.0.3",
      "info" => {"title" => "Crystal API", "version" => "1.0.0"},
      "paths" => paths
    }
  end
end

# ใช้งาน
RouteDoc.add(
  method: "GET",
  path: "/api/v1/users",
  summary: "List all users",
  tags: ["users"],
  params: [
    {name: "page", in: "query", description: "Page number", required: false, type: "integer"},
    {name: "per_page", in: "query", description: "Items per page", required: false, type: "integer"},
  ],
  responses: {200 => "List of users", 401 => "Unauthorized"}
)

get "/api/docs.json" do |env|
  env.response.content_type = "application/json"
  RouteDoc.generate_openapi.to_json
end
```

## Postman Collection

```crystal
# สร้าง Postman collection โดยอัตโนมัติ
def generate_postman_collection
  {
    "info" => {
      "name" => "Crystal API",
      "schema" => "https://schema.getpostman.com/json/collection/v2.1.0/collection.json",
    },
    "auth" => {
      "type" => "bearer",
      "bearer" => [{"key" => "token", "value" => "{{auth_token}}", "type" => "string"}],
    },
    "variable" => [
      {"key" => "base_url", "value" => "http://localhost:3000"},
      {"key" => "auth_token", "value" => ""},
    ],
    "item" => [
      {
        "name" => "Users",
        "item" => [
          {
            "name" => "List Users",
            "request" => {
              "method" => "GET",
              "url" => "{{base_url}}/api/v1/users?page=1&per_page=20",
            }
          },
          {
            "name" => "Create User",
            "request" => {
              "method" => "POST",
              "url" => "{{base_url}}/api/v1/users",
              "header" => [{"key" => "Content-Type", "value" => "application/json"}],
              "body" => {
                "mode" => "raw",
                "raw" => JSON.build { |j|
                  j.object do
                    j.field "email", "test@example.com"
                    j.field "name", "Test User"
                    j.field "password", "secret123"
                  end
                },
                "options" => {"raw" => {"language" => "json"}}
              }
            }
          },
        ]
      }
    ]
  }
end

get "/api/postman.json" do |env|
  env.response.content_type = "application/json"
  generate_postman_collection.to_json
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: API Documentation Endpoint

```crystal
require "kemal"
require "json"

# สร้าง self-documenting API
struct EndpointDoc
  include JSON::Serializable
  
  property method : String
  property path : String
  property summary : String
  property params : Array(String)
  property body_example : JSON::Any?
  property response_example : JSON::Any
  
  def initialize(@method, @path, @summary, @params = [] of String, @body_example = nil, @response_example = JSON::Any.new("{}"))
  end
end

ENDPOINTS = [
  EndpointDoc.new(
    "GET", "/api/v1/users",
    "List all users",
    ["page (int, optional)", "per_page (int, optional)", "q (string, optional)"],
    nil,
    JSON.parse({"data" => [{"id" => 1, "name" => "User", "email" => "user@example.com"}], "meta" => {"total" => 1}}.to_json)
  ),
  EndpointDoc.new(
    "POST", "/api/v1/users",
    "Create new user",
    [] of String,
    JSON.parse({"email" => "user@example.com", "name" => "User Name", "password" => "pass123"}.to_json),
    JSON.parse({"id" => 1, "email" => "user@example.com", "name" => "User Name"}.to_json)
  ),
]

get "/api/docs/endpoints" do |env|
  env.response.content_type = "application/json"
  ENDPOINTS.to_json
end

get "/api/docs" do
  render "src/views/api_docs.ecr"
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **OpenAPI/Swagger**: specification standard
2. **YAML Format**: paths, components, schemas
3. **Swagger UI**: visual documentation
4. **Security Schemes**: bearerAuth
5. **Reusable Components**: schemas, responses
6. **Examples**: request/response examples
7. **Auto-generation**: generate docs จาก code
8. **Postman Collection**: export สำหรับ testing

API documentation ที่ดีช่วยให้ team development และ client integration เร็วขึ้นมาก
