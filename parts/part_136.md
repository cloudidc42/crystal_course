# Part 136: Kemal Sessions and Cookies - การจัดการ Sessions และ Cookies

## บทนำ

Sessions และ Cookies เป็นกลไกสำคัญสำหรับ web applications ในการจดจำข้อมูลผู้ใช้ระหว่าง requests

## Cookies พื้นฐาน

```crystal
require "kemal"

# อ่าน cookie
get "/cookies" do |env|
  username = env.request.cookies["username"]?.try(&.value)
  "Cookie username: #{username || "ไม่มี"}"
end

# ตั้งค่า cookie
get "/set-cookie" do |env|
  cookie = HTTP::Cookie.new(
    name: "username",
    value: "สมชาย",
    path: "/",
    expires: Time.local + 7.days,
    http_only: true,
    secure: false,  # true ใน production (HTTPS only)
    samesite: HTTP::Cookie::SameSite::Lax
  )
  
  env.response.cookies << cookie
  "Cookie set!"
end

# ลบ cookie
get "/clear-cookie" do |env|
  cookie = HTTP::Cookie.new(
    name: "username",
    value: "",
    expires: Time.epoch(0)  # เวลาในอดีต = ลบ cookie
  )
  
  env.response.cookies << cookie
  "Cookie cleared!"
end

Kemal.run
```

## Cookie Options

```crystal
require "kemal"

# Cookie พร้อม options ครบถ้วน
def set_auth_cookie(env, token : String, remember_me : Bool = false)
  expires = remember_me ? Time.local + 30.days : Time.local + 24.hours
  
  auth_cookie = HTTP::Cookie.new(
    name: "auth_token",
    value: token,
    path: "/",
    domain: nil,        # nil = current domain
    expires: expires,
    secure: true,       # HTTPS only
    http_only: true,    # JavaScript ไม่สามารถ access ได้
    samesite: HTTP::Cookie::SameSite::Strict,
    extension: nil
  )
  
  env.response.cookies << auth_cookie
end

# Session cookie (หมดอายุเมื่อปิด browser)
def set_session_cookie(env, session_id : String)
  cookie = HTTP::Cookie.new(
    name: "session_id",
    value: session_id,
    path: "/",
    http_only: true,
    secure: true,
    samesite: HTTP::Cookie::SameSite::Lax
    # ไม่ตั้ง expires = session cookie
  )
  
  env.response.cookies << cookie
end

post "/login" do |env|
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  
  username = data["username"]?.try(&.as_s?) || ""
  remember_me = data["remember_me"]?.try(&.as_bool?) || false
  
  # จำลอง authentication
  if username == "admin"
    token = Random::Secure.hex(32)
    set_auth_cookie(env, token, remember_me)
    
    env.response.content_type = "application/json"
    {"success" => true, "message" => "Login successful"}.to_json
  else
    halt env, status_code: 401, response: {"error" => "Invalid credentials"}.to_json
  end
end

Kemal.run
```

## Session Store

```crystal
require "kemal"
require "json"
require "digest"

# In-memory session store
class SessionStore
  SESSION_COOKIE = "session_id"
  SESSION_EXPIRE = 24.hours
  
  struct Session
    property id : String
    property data : Hash(String, JSON::Any)
    property created_at : Time
    property last_accessed : Time
    
    def initialize(@id : String)
      @data = {} of String => JSON::Any
      @created_at = Time.local
      @last_accessed = Time.local
    end
    
    def touch
      @last_accessed = Time.local
    end
    
    def expired? : Bool
      Time.local - @last_accessed > SESSION_EXPIRE
    end
    
    def []?(key : String) : JSON::Any?
      @data[key]?
    end
    
    def []=(key : String, value : JSON::Any)
      @data[key] = value
    end
    
    def delete(key : String)
      @data.delete(key)
    end
    
    def clear
      @data.clear
    end
  end
  
  @@sessions = {} of String => Session
  @@mutex = Mutex.new
  
  def self.get_or_create(env : HTTP::Server::Context) : Session
    session_id = env.request.cookies[SESSION_COOKIE]?.try(&.value)
    
    session = nil
    
    if session_id
      @@mutex.synchronize do
        sess = @@sessions[session_id]?
        if sess && !sess.expired?
          sess.touch
          session = sess
        end
      end
    end
    
    unless session
      session = create_new(env)
    end
    
    session
  end
  
  def self.create_new(env : HTTP::Server::Context) : Session
    session_id = Random::Secure.hex(32)
    session = Session.new(session_id)
    
    @@mutex.synchronize do
      @@sessions[session_id] = session
    end
    
    cookie = HTTP::Cookie.new(
      name: SESSION_COOKIE,
      value: session_id,
      path: "/",
      http_only: true,
      secure: false,  # true ใน production
      samesite: HTTP::Cookie::SameSite::Lax
    )
    env.response.cookies << cookie
    
    session
  end
  
  def self.destroy(env : HTTP::Server::Context)
    if session_id = env.request.cookies[SESSION_COOKIE]?.try(&.value)
      @@mutex.synchronize do
        @@sessions.delete(session_id)
      end
      
      # Clear cookie
      expired_cookie = HTTP::Cookie.new(
        name: SESSION_COOKIE,
        value: "",
        expires: Time.epoch(0)
      )
      env.response.cookies << expired_cookie
    end
  end
  
  # Cleanup expired sessions (รัน periodically)
  def self.cleanup
    @@mutex.synchronize do
      @@sessions.reject! { |_, sess| sess.expired? }
    end
  end
  
  def self.count : Int32
    @@mutex.synchronize { @@sessions.size }
  end
end

# Background cleanup
spawn do
  loop do
    sleep 30.minutes
    SessionStore.cleanup
  end
end
```

## ใช้งาน Session

```crystal
require "kemal"

# Route ที่ใช้ session
get "/" do |env|
  session = SessionStore.get_or_create(env)
  visits = session["visits"]?.try(&.as_i?) || 0
  visits += 1
  session["visits"] = JSON::Any.new(visits.to_i64)
  
  env.response.content_type = "text/html"
  "<p>เยี่ยมชม: #{visits} ครั้ง</p>"
end

post "/login" do |env|
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  
  username = data["username"]?.try(&.as_s?) || ""
  password = data["password"]?.try(&.as_s?) || ""
  
  # จำลอง auth (ใน production ใช้ database + bcrypt)
  if username == "admin" && password == "secret"
    session = SessionStore.get_or_create(env)
    session["user_id"] = JSON::Any.new("1")
    session["username"] = JSON::Any.new(username)
    session["role"] = JSON::Any.new("admin")
    session["logged_in_at"] = JSON::Any.new(Time.local.to_rfc3339)
    
    env.response.content_type = "application/json"
    {"success" => true}.to_json
  else
    halt env, status_code: 401, response: {"error" => "Invalid credentials"}.to_json
  end
end

get "/profile" do |env|
  session = SessionStore.get_or_create(env)
  
  unless session["user_id"]?
    halt env, status_code: 401, response: {"error" => "Not logged in"}.to_json
  end
  
  env.response.content_type = "application/json"
  {
    "user_id" => session["user_id"]?.try(&.as_s?),
    "username" => session["username"]?.try(&.as_s?),
    "role" => session["role"]?.try(&.as_s?),
  }.to_json
end

post "/logout" do |env|
  SessionStore.destroy(env)
  
  env.response.content_type = "application/json"
  {"success" => true, "message" => "Logged out"}.to_json
end

Kemal.run
```

## Flash Messages

```crystal
require "kemal"

# Flash messages - ข้อความที่แสดงครั้งเดียวแล้วลบ
module Flash
  def self.set(session : SessionStore::Session, type : String, message : String)
    flash_data = session["_flash"]?.try(&.as_h?) || {} of String => JSON::Any
    flash_data[type] = JSON::Any.new(message)
    session["_flash"] = JSON::Any.new(flash_data)
  end
  
  def self.get(session : SessionStore::Session, type : String) : String?
    if flash_data = session["_flash"]?.try(&.as_h?)
      message = flash_data[type]?.try(&.as_s?)
      
      # ลบหลังอ่าน
      flash_data.delete(type)
      if flash_data.empty?
        session.delete("_flash")
      else
        session["_flash"] = JSON::Any.new(flash_data)
      end
      
      message
    end
  end
  
  def self.get_all(session : SessionStore::Session) : Hash(String, String)
    result = {} of String => String
    
    if flash_data = session["_flash"]?.try(&.as_h?)
      flash_data.each do |k, v|
        result[k] = v.as_s? || ""
      end
      session.delete("_flash")
    end
    
    result
  end
end

# ใช้งาน Flash messages
post "/users" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  
  session = SessionStore.get_or_create(env)
  
  # Validation
  data = JSON.parse(body) rescue nil
  
  if data.nil?
    Flash.set(session, "error", "ข้อมูลไม่ถูกต้อง")
    env.redirect "/users/new"
    next
  end
  
  name = data["name"]?.try(&.as_s?)
  
  if name.nil? || name.empty?
    Flash.set(session, "error", "กรุณาใส่ชื่อ")
    env.redirect "/users/new"
    next
  end
  
  # สร้าง user สำเร็จ
  Flash.set(session, "success", "สร้างผู้ใช้ #{name} สำเร็จแล้ว!")
  env.redirect "/users"
end

get "/users" do |env|
  session = SessionStore.get_or_create(env)
  messages = Flash.get_all(session)
  
  html = String.build do |s|
    messages.each do |type, msg|
      color = type == "success" ? "green" : "red"
      s << "<div style='color:#{color};padding:1rem;border:1px solid #{color};margin-bottom:1rem'>#{msg}</div>"
    end
    s << "<h1>รายการผู้ใช้</h1>"
  end
  
  env.response.content_type = "text/html"
  html
end
```

## Signed Cookies (HMAC)

```crystal
require "kemal"
require "openssl/hmac"

module SignedCookie
  SECRET_KEY = ENV["COOKIE_SECRET"]? || "development-secret-key-change-in-production"
  
  def self.sign(value : String) : String
    signature = OpenSSL::HMAC.hexdigest(
      OpenSSL::Algorithm::SHA256,
      SECRET_KEY,
      value
    )
    "#{value}--#{signature}"
  end
  
  def self.verify(signed_value : String) : String?
    parts = signed_value.split("--")
    return nil if parts.size < 2
    
    signature = parts.last
    value = parts[0...-1].join("--")
    
    expected_sig = OpenSSL::HMAC.hexdigest(
      OpenSSL::Algorithm::SHA256,
      SECRET_KEY,
      value
    )
    
    # Constant-time comparison เพื่อป้องกัน timing attacks
    if signature.bytesize == expected_sig.bytesize
      result = 0
      signature.bytes.zip(expected_sig.bytes) { |a, b| result |= a ^ b }
      result == 0 ? value : nil
    else
      nil
    end
  end
  
  def self.set(env : HTTP::Server::Context, name : String, value : String, **opts)
    signed = sign(value)
    cookie = HTTP::Cookie.new(name: name, value: signed, **opts)
    env.response.cookies << cookie
  end
  
  def self.get(env : HTTP::Server::Context, name : String) : String?
    if raw = env.request.cookies[name]?.try(&.value)
      verify(raw)
    end
  end
end

# ใช้งาน
post "/set-user" do |env|
  SignedCookie.set(
    env,
    "user_id",
    "123",
    path: "/",
    http_only: true,
    expires: Time.local + 7.days
  )
  "User ID cookie set (signed)"
end

get "/get-user" do |env|
  if user_id = SignedCookie.get(env, "user_id")
    "User ID: #{user_id} (verified)"
  else
    "No valid user_id cookie"
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Shopping Cart with Session

```crystal
require "kemal"
require "json"

struct CartItem
  include JSON::Serializable
  property product_id : Int32
  property name : String
  property price : Float64
  property quantity : Int32
  
  def initialize(@product_id, @name, @price, @quantity = 1)
  end
  
  def total : Float64
    price * quantity
  end
end

# Products database (จำลอง)
PRODUCTS = {
  1 => {name: "Crystal Book", price: 299.0},
  2 => {name: "Kemal Mug", price: 199.0},
  3 => {name: "Dev T-Shirt", price: 499.0},
}

def get_cart(session : SessionStore::Session) : Array(CartItem)
  cart_data = session["cart"]?.try(&.as_a?) || [] of JSON::Any
  cart_data.compact_map { |item| CartItem.from_json(item.to_json) rescue nil }
end

def save_cart(session : SessionStore::Session, cart : Array(CartItem))
  session["cart"] = JSON::Any.new(cart.map { |item| JSON.parse(item.to_json) })
end

# ดู cart
get "/cart" do |env|
  session = SessionStore.get_or_create(env)
  cart = get_cart(session)
  total = cart.sum(&.total)
  
  env.response.content_type = "application/json"
  {"items" => cart, "total" => total, "count" => cart.size}.to_json
end

# เพิ่ม item ใน cart
post "/cart/items" do |env|
  session = SessionStore.get_or_create(env)
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  
  product_id = data["product_id"]?.try(&.as_i?)
  quantity = data["quantity"]?.try(&.as_i?) || 1
  
  product = PRODUCTS[product_id]? if product_id
  
  unless product_id && product
    halt env, status_code: 404, response: {"error" => "Product not found"}.to_json
  end
  
  cart = get_cart(session)
  
  if idx = cart.index { |i| i.product_id == product_id }
    cart[idx] = CartItem.new(product_id, product[:name], product[:price], cart[idx].quantity + quantity)
  else
    cart << CartItem.new(product_id, product[:name], product[:price], quantity)
  end
  
  save_cart(session, cart)
  
  env.response.content_type = "application/json"
  env.response.status_code = 201
  {"success" => true, "cart_count" => cart.size}.to_json
end

# ลบ item จาก cart
delete "/cart/items/:product_id" do |env|
  session = SessionStore.get_or_create(env)
  product_id = env.params.url["product_id"].to_i?
  
  cart = get_cart(session)
  cart.reject! { |item| item.product_id == product_id }
  save_cart(session, cart)
  
  env.response.status_code = 204
  ""
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Cookies**: HTTP::Cookie, ตั้งค่า/อ่าน/ลบ cookie
2. **Cookie Options**: expires, http_only, secure, samesite
3. **Session Store**: in-memory session management
4. **Session Lifecycle**: create, access, destroy
5. **Session Cleanup**: ลบ expired sessions
6. **Flash Messages**: ข้อความชั่วคราวระหว่าง redirects
7. **Signed Cookies**: HMAC signature ป้องกันการแก้ไข
8. **Shopping Cart**: ตัวอย่าง practical session usage

Session ที่ดีควรมี: expiration, secure storage, CSRF protection
