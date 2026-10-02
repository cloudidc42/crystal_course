# Part 143: Lucky Pipes - Pipes ใน Lucky Framework

## บทนำ

Pipes ใน Lucky คือ before/after hooks ที่รันก่อนหรือหลัง action handler เหมาะสำหรับ authentication, authorization, logging, rate limiting และ validation

## Pipe พื้นฐาน

```crystal
# before pipe
class Users::Index < BrowserAction
  before require_auth
  
  get "/users" do
    render Users::IndexPage, users: UserQuery.new.select
  end
  
  private def require_auth
    if signed_in?
      continue
    else
      redirect to: SignIns::New
    end
  end
end

# after pipe
class Posts::Show < BrowserAction
  after track_view
  
  get "/posts/:post_id" do
    post = PostQuery.find(post_id)
    render Posts::ShowPage, post: post
  end
  
  private def track_view
    spawn do
      PostQuery.increment_view(post_id)
    end
    continue
  end
end
```

## Pipes ด้วย Module

```crystal
# src/actions/mixins/require_sign_in.cr
module RequireSignIn
  macro included
    before sign_in_required
  end
  
  private def sign_in_required
    if current_user?
      continue
    else
      session.set(:redirect_after_login, request.path)
      redirect to: SignIns::New
    end
  end
end

# src/actions/mixins/require_admin.cr
module RequireAdmin
  macro included
    include RequireSignIn
    before admin_required
  end
  
  private def admin_required
    if current_user?.try(&.admin?)
      continue
    else
      flash.failure = "ไม่มีสิทธิ์เข้าถึง"
      redirect to: Home::Index
    end
  end
end
```

## Authorization Pipes

```crystal
# src/actions/mixins/authorize_owner.cr
module AuthorizeOwner(T)
  macro included
    before authorize_resource_owner
  end
  
  abstract def resource : T
  
  private def authorize_resource_owner
    res = resource
    
    owner_id = case res
    when Post    then res.author_id
    when Comment then res.user_id
    else nil
    end
    
    if owner_id && (owner_id == current_user.id || current_user.admin?)
      continue
    else
      if json?
        json({error: "Forbidden"}, status: 403)
      else
        flash.failure = "คุณไม่มีสิทธิ์ดำเนินการนี้"
        redirect to: Home::Index
      end
    end
  end
  
  private def json? : Bool
    request.headers["Accept"]?.try(&.includes?("application/json")) || false
  end
end

# ใช้งาน
class Posts::Edit < BrowserAction
  include RequireSignIn
  include AuthorizeOwner(Post)
  
  get "/posts/:post_id/edit" do
    render Posts::EditPage, operation: SavePost.new(record: resource)
  end
  
  def resource : Post
    @post ||= PostQuery.find(post_id)
  end
end
```

## Rate Limiting Pipe

```crystal
# src/actions/mixins/rate_limit.cr
module RateLimit
  # Override ใน subclass
  def rate_limit_key : String
    request.remote_ip || "unknown"
  end
  
  def rate_limit : Int32
    100
  end
  
  def rate_window : Time::Span
    1.minute
  end
  
  macro rate_limited
    before check_rate_limit
    
    private def check_rate_limit
      key = "rate_limit:#{rate_limit_key}:#{self.class.name}"
      
      # ใช้ Redis หรือ in-memory store
      count = RateLimitStore.increment(key, rate_window)
      
      response.headers["X-RateLimit-Limit"] = rate_limit.to_s
      response.headers["X-RateLimit-Remaining"] = [rate_limit - count, 0].max.to_s
      
      if count > rate_limit
        if request.headers["Accept"]?.try(&.includes?("application/json"))
          json({error: "Rate limit exceeded", retry_after: rate_window.total_seconds.to_i}, status: 429)
        else
          flash.failure = "คุณส่งคำขอเร็วเกินไป กรุณารอสักครู่"
          redirect to: Home::Index
        end
      else
        continue
      end
    end
  end
end

# In-memory rate limit store (ใน production ใช้ Redis)
class RateLimitStore
  @@store = {} of String => {Int32, Time}
  @@mutex = Mutex.new
  
  def self.increment(key : String, window : Time::Span) : Int32
    @@mutex.synchronize do
      now = Time.local
      
      if entry = @@store[key]?
        count, window_start = entry
        
        if now - window_start < window
          @@store[key] = {count + 1, window_start}
          count + 1
        else
          @@store[key] = {1, now}
          1
        end
      else
        @@store[key] = {1, now}
        1
      end
    end
  end
end

# ใช้งาน
class Api::V1::Posts::Create < ApiAction
  include RateLimit
  rate_limited
  
  def rate_limit : Int32
    10  # 10 posts per minute
  end
  
  post "/api/v1/posts" do
    # ...
  end
end
```

## Logging Pipe

```crystal
# src/actions/mixins/request_logger.cr
module RequestLogger
  macro included
    before log_request_start
    after log_request_end
  end
  
  @@logger = ::Log.for("requests")
  
  private def log_request_start
    context.set(:request_start, Time.monotonic)
    @@logger.info { "#{request.method} #{request.path} started" }
    continue
  end
  
  private def log_request_end
    start = context.get?(:request_start)
    elapsed = start ? (Time.monotonic - start.as(Time::Span)).total_milliseconds.round(2) : 0
    
    @@logger.info {
      status = response.status_code
      "#{request.method} #{request.path} #{status} (#{elapsed}ms) - #{current_user?.try(&.email) || "anonymous"}"
    }
    continue
  end
end
```

## CSRF Protection Pipe

```crystal
# Built-in Lucky CSRF protection
abstract class BrowserAction < Lucky::Action
  include Lucky::ProtectFromForgery
  # ...
end

# ใน forms ต้อง include CSRF token
form_for Users::Create do
  csrf_hidden_input  # auto-generated
  # ...
end
```

## Validation Pipe

```crystal
# Validate JSON body
module ValidateJsonBody
  macro included
    before validate_content_type
  end
  
  private def validate_content_type
    content_type = request.headers["Content-Type"]?
    
    if request.method.in?("POST", "PUT", "PATCH")
      unless content_type && content_type.starts_with?("application/json")
        json({
          error: "Invalid Content-Type",
          message: "Content-Type must be application/json"
        }, status: 415)
      else
        continue
      end
    else
      continue
    end
  end
end
```

## Pagination Pipe

```crystal
# src/actions/mixins/paginate.cr
module Paginate
  def page : Int32
    params.get?(:page).try(&.to_i32) || 1
  end
  
  def per_page : Int32
    requested = params.get?(:per_page).try(&.to_i32) || 20
    [requested, max_per_page].min
  end
  
  def max_per_page : Int32
    100
  end
  
  def offset : Int32
    (page - 1) * per_page
  end
  
  def pagination_meta(total : Int64) : NamedTuple(page: Int32, per_page: Int32, total: Int64, pages: Int32)
    {
      page: page,
      per_page: per_page,
      total: total,
      pages: (total.to_f / per_page).ceil.to_i32,
    }
  end
end

# ใช้งาน
class Api::V1::Users::Index < ApiAction
  include Paginate
  
  get "/api/v1/users" do
    total = UserQuery.new.active.select_count
    users = UserQuery.new.active
      .order_by_created_at(:desc)
      .limit(per_page)
      .offset(offset)
      .select
    
    json({
      data: users.map { |u| serialize(u) },
      meta: pagination_meta(total)
    })
  end
end
```

## Complete BrowserAction with Pipes

```crystal
# src/actions/browser_action.cr
abstract class BrowserAction < Lucky::Action
  include Lucky::SecureHeaders::DisableFLoC
  include Lucky::ProtectFromForgery
  include AuthHelpers
  include FlashMessages
  
  accepted_formats [:html]
  
  # Shared before pipe
  before set_locale
  before set_time_zone
  
  private def set_locale
    locale = params.get?(:lang) || session.get?(:locale) || "th"
    I18n.locale = locale
    continue
  end
  
  private def set_time_zone
    tz = current_user?.try(&.timezone) || "Asia/Bangkok"
    Time::Location.load(tz) rescue nil
    continue
  end
  
  # Error handling
  rescue_from Avram::RecordNotFoundError do
    if json?
      json({error: "Not Found"}, status: 404)
    else
      flash.failure = "ไม่พบข้อมูลที่ต้องการ"
      redirect to: Home::Index
    end
  end
  
  private def json? : Bool
    request.headers["Accept"]?.try(&.includes?("application/json")) || false
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: API Authentication Pipe

```crystal
# JWT-based API auth
module RequireApiAuth
  macro included
    before authenticate_api_request
  end
  
  private def authenticate_api_request
    auth = request.headers["Authorization"]?
    
    unless auth && auth.starts_with?("Bearer ")
      json({error: "Missing authentication token"}, status: 401)
      return
    end
    
    token = auth[7..]
    payload = decode_jwt(token)
    
    if payload.nil?
      json({error: "Invalid or expired token"}, status: 401)
      return
    end
    
    user_id = payload["user_id"]?.try(&.as_i64?)
    user = UserQuery.find?(user_id || 0_i64)
    
    unless user && user.active
      json({error: "User not found or inactive"}, status: 401)
      return
    end
    
    context.set(:api_user, user)
    continue
  end
  
  def api_user : User
    context.get(:api_user).as(User)
  end
  
  private def decode_jwt(token : String) : JSON::Any?
    # JWT decode logic
    parts = token.split(".")
    return nil unless parts.size == 3
    
    begin
      payload_json = Base64.decode_string(parts[1])
      payload = JSON.parse(payload_json)
      
      # Check expiration
      if exp = payload["exp"]?.try(&.as_i64?)
        return nil if exp < Time.local.to_unix
      end
      
      payload
    rescue
      nil
    end
  end
end

# ใช้งาน
class Api::V1::Profile::Show < ApiAction
  include RequireApiAuth
  
  get "/api/v1/profile" do
    user = api_user
    
    json({
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
    })
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **before/after**: pipes ก่อน/หลัง action
2. **continue**: ดำเนินการต่อ
3. **Pipe Modules**: แยก logic เป็น mixins
4. **RequireSignIn**: ตรวจสอบ authentication
5. **RequireAdmin**: ตรวจสอบ authorization
6. **AuthorizeOwner**: ตรวจสอบว่าเป็น owner
7. **RateLimit**: จำกัด request rate
8. **Logging Pipe**: log requests
9. **CSRF Protection**: built-in Lucky CSRF
10. **ValidationPipe**: ตรวจสอบ Content-Type
11. **Pagination**: reusable pagination

Pipes เป็น mechanism สำคัญใน Lucky สำหรับ separation of concerns และ code reuse
