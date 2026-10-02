# Part 129: OAuth และ Authentication - การพิสูจน์ตัวตนใน Crystal

## บทนำ

Authentication เป็นส่วนสำคัญของ web applications ในบทนี้เราจะเรียนรู้การทำ HTTP Basic Auth, Token Authentication, JWT (JSON Web Tokens), และ OAuth2 flow

## HTTP Basic Authentication

```crystal
require "http/client"
require "base64"

# Basic Auth Client
def basic_auth_header(username : String, password : String) : String
  credentials = Base64.strict_encode("#{username}:#{password}")
  "Basic #{credentials}"
end

# ส่ง request พร้อม Basic Auth
headers = HTTP::Headers{
  "Authorization" => basic_auth_header("user", "password123"),
}

response = HTTP::Client.get("https://httpbin.org/basic-auth/user/password123", headers: headers)
puts "Status: #{response.status_code}"  # 200 = success, 401 = unauthorized

# ใช้ built-in basic_auth
HTTP::Client.new("httpbin.org", tls: true) do |client|
  client.basic_auth("user", "password123")
  response = client.get("/basic-auth/user/password123")
  puts response.status_code
end
```

### Basic Auth Server

```crystal
require "http/server"
require "base64"

# Server ที่ใช้ Basic Auth
class BasicAuthHandler
  include HTTP::Handler
  
  # ฐานข้อมูล users (ใน production ควรเก็บ hashed passwords)
  USERS = {
    "admin" => "secret123",
    "user1" => "pass456",
  }
  
  def initialize(@realm : String = "Protected Area")
  end
  
  def call(context : HTTP::Server::Context)
    if authenticated?(context)
      call_next(context)
    else
      context.response.headers["WWW-Authenticate"] = "Basic realm=\"#{@realm}\""
      context.response.status_code = 401
      context.response.print("Unauthorized")
    end
  end
  
  private def authenticated?(context : HTTP::Server::Context) : Bool
    auth_header = context.request.headers["Authorization"]?
    return false unless auth_header
    return false unless auth_header.starts_with?("Basic ")
    
    decoded = Base64.decode_string(auth_header[6..]) rescue return false
    parts = decoded.split(":", 2)
    return false unless parts.size == 2
    
    username, password = parts
    USERS[username]? == password
  end
end

server = HTTP::Server.new([
  BasicAuthHandler.new("API"),
  HTTP::Handler.proc do |context|
    context.response.print("Welcome, authenticated user!")
  end,
])

server.listen("0.0.0.0", 8080)
```

## Token Authentication

```crystal
require "http/server"
require "random/secure"

# Token-based Authentication
class TokenStore
  def initialize
    @tokens = {} of String => TokenInfo
    @mutex = Mutex.new
    start_cleanup
  end
  
  struct TokenInfo
    property user_id : String
    property username : String
    property created_at : Time
    property expires_at : Time
    property scopes : Array(String)
    
    def initialize(@user_id, @username, @scopes = ["read"], expires_in : Time::Span = 24.hours)
      @created_at = Time.local
      @expires_at = @created_at + expires_in
    end
    
    def expired?
      Time.local > expires_at
    end
    
    def has_scope?(scope : String) : Bool
      scopes.includes?(scope) || scopes.includes?("admin")
    end
  end
  
  def create_token(user_id : String, username : String, scopes : Array(String) = ["read"]) : String
    token = Random::Secure.hex(32)
    @mutex.synchronize do
      @tokens[token] = TokenInfo.new(user_id, username, scopes)
    end
    token
  end
  
  def verify_token(token : String) : TokenInfo?
    @mutex.synchronize do
      info = @tokens[token]?
      return nil unless info
      return nil if info.expired?
      info
    end
  end
  
  def revoke_token(token : String)
    @mutex.synchronize { @tokens.delete(token) }
  end
  
  def revoke_user_tokens(user_id : String)
    @mutex.synchronize do
      @tokens.reject! { |_, info| info.user_id == user_id }
    end
  end
  
  private def start_cleanup
    spawn do
      loop do
        sleep 1.hour
        @mutex.synchronize do
          @tokens.reject! { |_, info| info.expired? }
        end
      end
    end
  end
end

# Token Auth Handler
class TokenAuthHandler
  include HTTP::Handler
  
  def initialize(@token_store : TokenStore)
  end
  
  def call(context : HTTP::Server::Context)
    if token_info = authenticate(context)
      # เพิ่มข้อมูล user ใน context
      context.request.headers["X-User-ID"] = token_info.user_id
      context.request.headers["X-Username"] = token_info.username
      call_next(context)
    else
      context.response.status_code = 401
      context.response.content_type = "application/json"
      context.response.print({"error" => "Unauthorized"}.to_json)
    end
  end
  
  private def authenticate(context : HTTP::Server::Context) : TokenStore::TokenInfo?
    auth = context.request.headers["Authorization"]?
    return nil unless auth
    return nil unless auth.starts_with?("Bearer ")
    
    token = auth[7..]
    @token_store.verify_token(token)
  end
end
```

## JWT (JSON Web Tokens)

```crystal
require "json"
require "base64"
require "openssl/hmac"

# JWT Implementation ใน Crystal
module JWT
  class Error < Exception; end
  class InvalidSignature < Error; end
  class TokenExpired < Error; end
  class InvalidToken < Error; end
  
  def self.encode(payload : Hash(String, JSON::Any::Type), secret : String, algorithm : String = "HS256") : String
    header = {"alg" => algorithm, "typ" => "JWT"}
    
    header_b64 = Base64.urlsafe_encode(header.to_json, padding: false)
    payload_b64 = Base64.urlsafe_encode(payload.to_json, padding: false)
    
    message = "#{header_b64}.#{payload_b64}"
    signature = sign(message, secret, algorithm)
    
    "#{message}.#{signature}"
  end
  
  def self.decode(token : String, secret : String) : Hash(String, JSON::Any)
    parts = token.split(".")
    raise InvalidToken.new("Invalid JWT format") unless parts.size == 3
    
    header_b64, payload_b64, signature = parts
    
    # ตรวจสอบ signature
    message = "#{header_b64}.#{payload_b64}"
    header = JSON.parse(Base64.decode_string("#{header_b64}=="))
    algorithm = header["alg"]?.try(&.as_s?) || "HS256"
    
    expected_sig = sign(message, secret, algorithm)
    
    raise InvalidSignature.new("Invalid JWT signature") unless constant_time_compare(signature, expected_sig)
    
    # Decode payload
    payload = JSON.parse(Base64.decode_string("#{payload_b64}=="))
    
    # ตรวจสอบ expiration
    if exp = payload["exp"]?.try(&.as_i64?)
      raise TokenExpired.new("JWT token has expired") if Time.local.to_unix > exp
    end
    
    payload.as_h
  end
  
  private def self.sign(message : String, secret : String, algorithm : String) : String
    hmac = case algorithm
    when "HS256"
      OpenSSL::HMAC.digest(:sha256, secret, message)
    when "HS384"
      OpenSSL::HMAC.digest(:sha384, secret, message)
    when "HS512"
      OpenSSL::HMAC.digest(:sha512, secret, message)
    else
      raise InvalidToken.new("Unsupported algorithm: #{algorithm}")
    end
    
    Base64.urlsafe_encode(hmac, padding: false)
  end
  
  private def self.constant_time_compare(a : String, b : String) : Bool
    return false if a.size != b.size
    
    result = 0
    a.bytes.zip(b.bytes).each { |x, y| result |= x ^ y }
    result == 0
  end
end

# ใช้งาน JWT
secret = "my-super-secret-key-123"

# สร้าง token
payload = {
  "sub" => JSON::Any.new("user123"),
  "name" => JSON::Any.new("สมชาย"),
  "role" => JSON::Any.new("admin"),
  "iat" => JSON::Any.new(Time.local.to_unix),
  "exp" => JSON::Any.new((Time.local + 1.hour).to_unix),
}

token = JWT.encode(payload, secret)
puts "Token: #{token[0..50]}..."

# ถอดรหัส token
begin
  decoded = JWT.decode(token, secret)
  puts "Decoded: #{decoded["name"]}, role: #{decoded["role"]}"
rescue JWT::TokenExpired
  puts "Token expired!"
rescue JWT::InvalidSignature
  puts "Invalid signature!"
end
```

## OAuth2 Flow

```crystal
require "http/client"
require "json"
require "uri"

# OAuth2 Client
class OAuth2Client
  struct TokenResponse
    include JSON::Serializable
    
    property access_token : String
    property token_type : String
    property expires_in : Int32?
    property refresh_token : String?
    property scope : String?
    
    def expires_at : Time?
      return nil unless exp = expires_in
      Time.local + exp.seconds
    end
  end
  
  def initialize(
    @client_id : String,
    @client_secret : String,
    @redirect_uri : String,
    @authorize_url : String,
    @token_url : String
  )
  end
  
  # Step 1: สร้าง Authorization URL
  def authorization_url(scope : String, state : String? = nil) : String
    params = HTTP::Params.encode({
      "client_id" => @client_id,
      "redirect_uri" => @redirect_uri,
      "response_type" => "code",
      "scope" => scope,
      "state" => state || Random::Secure.hex(16),
    })
    
    "#{@authorize_url}?#{params}"
  end
  
  # Step 2: แลก authorization code เป็น access token
  def exchange_code(code : String) : TokenResponse
    body = HTTP::Params.encode({
      "grant_type" => "authorization_code",
      "code" => code,
      "redirect_uri" => @redirect_uri,
      "client_id" => @client_id,
      "client_secret" => @client_secret,
    })
    
    headers = HTTP::Headers{
      "Content-Type" => "application/x-www-form-urlencoded",
      "Accept" => "application/json",
    }
    
    response = HTTP::Client.post(@token_url, headers: headers, body: body)
    
    unless response.status.ok?
      raise "Token exchange failed: #{response.body}"
    end
    
    TokenResponse.from_json(response.body)
  end
  
  # Refresh token
  def refresh_token(refresh_token : String) : TokenResponse
    body = HTTP::Params.encode({
      "grant_type" => "refresh_token",
      "refresh_token" => refresh_token,
      "client_id" => @client_id,
      "client_secret" => @client_secret,
    })
    
    headers = HTTP::Headers{
      "Content-Type" => "application/x-www-form-urlencoded",
    }
    
    response = HTTP::Client.post(@token_url, headers: headers, body: body)
    
    unless response.status.ok?
      raise "Token refresh failed: #{response.body}"
    end
    
    TokenResponse.from_json(response.body)
  end
  
  # Client Credentials Flow (machine-to-machine)
  def client_credentials(scope : String) : TokenResponse
    body = HTTP::Params.encode({
      "grant_type" => "client_credentials",
      "client_id" => @client_id,
      "client_secret" => @client_secret,
      "scope" => scope,
    })
    
    headers = HTTP::Headers{
      "Content-Type" => "application/x-www-form-urlencoded",
    }
    
    response = HTTP::Client.post(@token_url, headers: headers, body: body)
    TokenResponse.from_json(response.body)
  end
end

# GitHub OAuth2 Example
github_oauth = OAuth2Client.new(
  client_id: ENV["GITHUB_CLIENT_ID"]? || "your-client-id",
  client_secret: ENV["GITHUB_CLIENT_SECRET"]? || "your-client-secret",
  redirect_uri: "http://localhost:3000/callback",
  authorize_url: "https://github.com/login/oauth/authorize",
  token_url: "https://github.com/login/oauth/access_token"
)

# สร้าง URL สำหรับ redirect ผู้ใช้
auth_url = github_oauth.authorization_url("user:email read:user", state: "csrf_token_123")
puts "Redirect user to: #{auth_url}"
```

## OAuth2 Server Flow ใน Crystal Web App

```crystal
require "http/server"
require "json"
require "uri"

# OAuth2 Authorization Server Flow
class OAuthServer
  def initialize
    @token_store = TokenStore.new
    @auth_codes = {} of String => AuthCode
    @mutex = Mutex.new
    start_cleanup
  end
  
  struct AuthCode
    property code : String
    property user_id : String
    property scope : String
    property expires_at : Time
    property redirect_uri : String
    
    def initialize(@code, @user_id, @scope, @redirect_uri, expires_in : Time::Span = 10.minutes)
      @expires_at = Time.local + expires_in
    end
    
    def expired?
      Time.local > @expires_at
    end
  end
  
  # สร้าง authorization code
  def create_auth_code(user_id : String, scope : String, redirect_uri : String) : String
    code = Random::Secure.hex(32)
    @mutex.synchronize { @auth_codes[code] = AuthCode.new(code, user_id, scope, redirect_uri) }
    code
  end
  
  # แลก code เป็น token
  def exchange_code(code : String, redirect_uri : String) : TokenStore::TokenInfo?
    auth_code = @mutex.synchronize { @auth_codes.delete(code) }
    
    return nil unless auth_code
    return nil if auth_code.expired?
    return nil unless auth_code.redirect_uri == redirect_uri
    
    scopes = auth_code.scope.split(" ")
    token = @token_store.create_token(auth_code.user_id, auth_code.user_id, scopes)
    @token_store.verify_token(token)
  end
  
  private def start_cleanup
    spawn do
      loop do
        sleep 10.minutes
        @mutex.synchronize do
          @auth_codes.reject! { |_, code| code.expired? }
        end
      end
    end
  end
end
```

## Password Hashing

```crystal
require "crypto/bcrypt/password"

# BCrypt Password Hashing
module PasswordHasher
  COST = 12  # ยิ่งสูงยิ่งปลอดภัยแต่ช้า (8-12 เหมาะสม)
  
  def self.hash(password : String) : String
    Crypto::Bcrypt::Password.create(password, cost: COST).to_s
  end
  
  def self.verify(password : String, hash : String) : Bool
    Crypto::Bcrypt::Password.new(hash).verify(password)
  rescue
    false
  end
end

# ใช้งาน
hashed = PasswordHasher.hash("myPassword123!")
puts "Hash: #{hashed}"
puts "Valid: #{PasswordHasher.verify("myPassword123!", hashed)}"  # true
puts "Invalid: #{PasswordHasher.verify("wrongPassword", hashed)}" # false
```

## Complete Auth System

```crystal
require "http/server"
require "json"

class AuthSystem
  # Users DB (ใน production ใช้ database จริง)
  USERS = {
    "user1" => {
      "id" => "u001",
      "name" => "สมชาย",
      "email" => "somchai@example.com",
      "password_hash" => "$2a$12$...", # bcrypt hash
    },
  }
  
  def initialize
    @token_store = TokenStore.new
    @jwt_secret = ENV["JWT_SECRET"]? || Random::Secure.hex(32)
  end
  
  def handle(context : HTTP::Server::Context)
    path = context.request.path
    method = context.request.method
    
    case {method, path}
    when {"POST", "/auth/login"}
      handle_login(context)
    when {"POST", "/auth/refresh"}
      handle_refresh(context)
    when {"DELETE", "/auth/logout"}
      handle_logout(context)
    when {"GET", "/auth/me"}
      handle_me(context)
    else
      json(context, {"error" => "Not found"}, 404)
    end
  end
  
  private def handle_login(context)
    body = context.request.body.try(&.gets_to_end)
    data = JSON.parse(body || "{}") rescue nil
    
    unless data
      return json(context, {"error" => "Invalid body"}, 400)
    end
    
    email = data["email"]?.try(&.as_s?)
    password = data["password"]?.try(&.as_s?)
    
    unless email && password
      return json(context, {"error" => "Email and password required"}, 400)
    end
    
    # หา user
    user = USERS.values.find { |u| u["email"] == email }
    
    unless user && verify_password(password, user["password_hash"])
      return json(context, {"error" => "Invalid credentials"}, 401)
    end
    
    # สร้าง tokens
    access_token = create_access_token(user["id"], user["name"])
    refresh_token = create_refresh_token(user["id"])
    
    json(context, {
      "access_token" => access_token,
      "refresh_token" => refresh_token,
      "expires_in" => 3600,
      "token_type" => "Bearer",
    })
  end
  
  private def handle_refresh(context)
    body = context.request.body.try(&.gets_to_end)
    data = JSON.parse(body || "{}") rescue nil
    refresh_token = data.try { |d| d["refresh_token"]?.try(&.as_s?) }
    
    unless refresh_token
      return json(context, {"error" => "Refresh token required"}, 400)
    end
    
    # ตรวจสอบ refresh token
    info = verify_refresh_token(refresh_token)
    unless info
      return json(context, {"error" => "Invalid refresh token"}, 401)
    end
    
    # สร้าง access token ใหม่
    new_access_token = create_access_token(info[:user_id], info[:username])
    
    json(context, {
      "access_token" => new_access_token,
      "expires_in" => 3600,
      "token_type" => "Bearer",
    })
  end
  
  private def handle_logout(context)
    token = extract_bearer_token(context)
    @token_store.revoke_token(token) if token
    context.response.status_code = 204
  end
  
  private def handle_me(context)
    token = extract_bearer_token(context)
    
    unless token
      return json(context, {"error" => "Unauthorized"}, 401)
    end
    
    begin
      payload = JWT.decode(token, @jwt_secret)
      user_id = payload["sub"]?.try(&.as_s?)
      
      user = USERS.values.find { |u| u["id"] == user_id }
      
      if user
        json(context, {
          "id" => user["id"],
          "name" => user["name"],
          "email" => user["email"],
        })
      else
        json(context, {"error" => "User not found"}, 404)
      end
    rescue JWT::TokenExpired
      json(context, {"error" => "Token expired"}, 401)
    rescue JWT::InvalidSignature
      json(context, {"error" => "Invalid token"}, 401)
    end
  end
  
  private def create_access_token(user_id : String, username : String) : String
    payload = {
      "sub" => JSON::Any.new(user_id),
      "name" => JSON::Any.new(username),
      "type" => JSON::Any.new("access"),
      "iat" => JSON::Any.new(Time.local.to_unix),
      "exp" => JSON::Any.new((Time.local + 1.hour).to_unix),
    }
    JWT.encode(payload, @jwt_secret)
  end
  
  private def create_refresh_token(user_id : String) : String
    @token_store.create_token(user_id, user_id, ["refresh"])
  end
  
  private def verify_refresh_token(token : String) : {user_id: String, username: String}?
    info = @token_store.verify_token(token)
    return nil unless info
    return nil unless info.has_scope?("refresh")
    {user_id: info.user_id, username: info.username}
  end
  
  private def extract_bearer_token(context : HTTP::Server::Context) : String?
    auth = context.request.headers["Authorization"]?
    return nil unless auth&.starts_with?("Bearer ")
    auth[7..]
  end
  
  private def verify_password(password : String, hash : String) : Bool
    # จำลอง password check
    hash == "hashed_#{password}" rescue false
  end
  
  private def json(context, data, status = 200)
    context.response.status_code = status
    context.response.content_type = "application/json"
    context.response.print data.to_json
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: OAuth2 GitHub Login

```crystal
require "http/server"
require "http/client"
require "json"

# GitHub OAuth2 integration
class GitHubOAuthHandler
  include HTTP::Handler
  
  GITHUB_AUTHORIZE = "https://github.com/login/oauth/authorize"
  GITHUB_TOKEN = "https://github.com/login/oauth/access_token"
  GITHUB_USER_API = "https://api.github.com/user"
  
  def initialize(@client_id : String, @client_secret : String, @callback_path : String = "/callback")
  end
  
  def call(context : HTTP::Server::Context)
    case context.request.path
    when "/login"
      state = Random::Secure.hex(16)
      # Store state in session (simplified)
      context.response.headers["Set-Cookie"] = "oauth_state=#{state}; HttpOnly; Secure"
      
      params = HTTP::Params.encode({
        "client_id" => @client_id,
        "redirect_uri" => "http://localhost:3000#{@callback_path}",
        "scope" => "user:email",
        "state" => state,
      })
      
      context.response.headers["Location"] = "#{GITHUB_AUTHORIZE}?#{params}"
      context.response.status_code = 302
      
    when @callback_path
      handle_callback(context)
      
    else
      call_next(context)
    end
  end
  
  private def handle_callback(context : HTTP::Server::Context)
    params = HTTP::Params.parse(context.request.query || "")
    code = params["code"]?
    state = params["state"]?
    
    unless code && state
      context.response.status_code = 400
      context.response.print "Missing code or state"
      return
    end
    
    # แลก code เป็น access token
    token_response = exchange_code(code)
    
    unless token_response
      context.response.status_code = 500
      context.response.print "Token exchange failed"
      return
    end
    
    # ดึงข้อมูล user
    user = get_github_user(token_response["access_token"])
    
    context.response.content_type = "application/json"
    context.response.print user.to_json
  end
  
  private def exchange_code(code : String) : JSON::Any?
    body = HTTP::Params.encode({
      "client_id" => @client_id,
      "client_secret" => @client_secret,
      "code" => code,
    })
    
    response = HTTP::Client.post(
      GITHUB_TOKEN,
      headers: HTTP::Headers{"Accept" => "application/json", "Content-Type" => "application/x-www-form-urlencoded"},
      body: body
    )
    
    JSON.parse(response.body) if response.status.ok?
  rescue
    nil
  end
  
  private def get_github_user(token : String) : JSON::Any
    response = HTTP::Client.get(
      GITHUB_USER_API,
      headers: HTTP::Headers{
        "Authorization" => "token #{token}",
        "Accept" => "application/vnd.github.v3+json",
      }
    )
    
    JSON.parse(response.body)
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HTTP Basic Auth**: username/password ใน Authorization header
2. **Token Authentication**: Bearer tokens สำหรับ stateless auth
3. **JWT**: สร้างและตรวจสอบ JSON Web Tokens
   - Header, Payload, Signature
   - HMAC-SHA256 signing
   - Expiration validation
4. **OAuth2 Authorization Code Flow**:
   - Step 1: สร้าง authorization URL
   - Step 2: User login ที่ provider
   - Step 3: แลก code เป็น token
5. **Password Hashing**: BCrypt
6. **Complete Auth System**: login, refresh, logout, me endpoint
7. **GitHub OAuth2**: ตัวอย่างการ integrate กับ third-party provider

หลักการสำคัญ:
- ไม่เก็บ plain text passwords
- ใช้ HTTPS เสมอ
- JWT secret ต้องแข็งแกร่ง
- Refresh token lifetime ยาวกว่า access token
- ตรวจสอบ state parameter เพื่อป้องกัน CSRF
