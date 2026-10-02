# Part 142: Lucky Authentication - Authentication ใน Lucky

## บทนำ

Lucky มี authentication system ผ่าน `LuckyAuth` shard หรือสร้างเองด้วย session management ในบทนี้จะครอบคลุมทั้งสองแบบ

## Basic Session Authentication

```crystal
# src/models/user.cr
class User < BaseModel
  table do
    primary_key id : Int64
    timestamps
    column email : String
    column name : String
    column hashed_password : String
    column role : String
    column active : Bool
    column last_login_at : Time?
    column failed_login_count : Int32
    column locked_until : Time?
  end
  
  def admin? : Bool
    role == "admin"
  end
  
  def locked? : Bool
    if until_time = locked_until
      until_time > Time.local
    else
      false
    end
  end
  
  def verify_password(password : String) : Bool
    Crypto::Bcrypt::Password.new(hashed_password).verify(password)
  rescue
    false
  end
end
```

## Session Helper

```crystal
# src/actions/mixins/auth_helpers.cr
module AuthHelpers
  def sign_in(user : User)
    session.set(:user_id, user.id.to_s)
    session.set(:signed_in_at, Time.local.to_unix.to_s)
  end
  
  def sign_out
    session.delete(:user_id)
    session.delete(:signed_in_at)
  end
  
  def current_user? : User?
    @current_user ||= begin
      if user_id_str = session.get?(:user_id)
        if user_id = user_id_str.to_i64?
          UserQuery.find?(user_id)
        end
      end
    end
  end
  
  def current_user : User
    current_user? || raise Lucky::RouteNotFoundError.new(request)
  end
  
  def signed_in? : Bool
    !current_user?.nil?
  end
end
```

## Sign In Action

```crystal
# src/actions/sign_ins/new.cr
class SignIns::New < BrowserAction
  get "/sign_in" do
    render SignIns::NewPage
  end
end

# src/actions/sign_ins/create.cr
class SignIns::Create < BrowserAction
  include AuthHelpers
  
  post "/sign_in" do
    email = params.get?(:email) || ""
    password = params.get?(:password) || ""
    
    user = UserQuery.new.email(email.downcase).first?
    
    if user.nil?
      # ใช้เวลาเท่ากันเพื่อป้องกัน timing attack
      Crypto::Bcrypt::Password.create("dummy_check_to_prevent_timing_attack")
      flash.failure = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
      redirect to: SignIns::New
      next
    end
    
    if user.locked?
      flash.failure = "บัญชีถูกล็อก กรุณาลองใหม่ในภายหลัง"
      redirect to: SignIns::New
      next
    end
    
    if user.verify_password(password)
      # Login สำเร็จ
      UpdateUser.update!(user,
        failed_login_count: 0,
        last_login_at: Time.local
      )
      
      sign_in(user)
      
      # Redirect ไปยัง page ที่ต้องการก่อน login
      redirect_path = session.get?(:redirect_after_login) || "/"
      session.delete(:redirect_after_login)
      redirect redirect_path
    else
      # Login ล้มเหลว
      failed_count = user.failed_login_count + 1
      lock_until = failed_count >= 5 ? Time.local + 15.minutes : nil
      
      UpdateUser.update!(user,
        failed_login_count: failed_count,
        locked_until: lock_until
      )
      
      if failed_count >= 5
        flash.failure = "บัญชีถูกล็อก 15 นาที เนื่องจากพยายาม login ผิดหลายครั้ง"
      else
        flash.failure = "อีเมลหรือรหัสผ่านไม่ถูกต้อง (เหลืออีก #{5 - failed_count} ครั้ง)"
      end
      
      redirect to: SignIns::New
    end
  end
end

# src/actions/sign_outs/create.cr
class SignOuts::Create < BrowserAction
  include AuthHelpers
  
  delete "/sign_out" do
    sign_out
    flash.success = "ออกจากระบบแล้ว"
    redirect to: SignIns::New
  end
end
```

## RequireSignIn Pipe

```crystal
# src/actions/mixins/require_sign_in.cr
module RequireSignIn
  macro included
    before sign_in_required
  end
  
  private def sign_in_required
    if signed_in?
      continue
    else
      # บันทึก requested path
      session.set(:redirect_after_login, request.path)
      
      if json_request?
        json({error: "Authentication required"}, status: 401)
      else
        flash.info = "กรุณาเข้าสู่ระบบก่อน"
        redirect to: SignIns::New
      end
    end
  end
  
  private def signed_in? : Bool
    !current_user?.nil?
  end
  
  private def json_request? : Bool
    request.headers["Accept"]?.try(&.includes?("application/json")) || false
  end
end
```

## Role-Based Authorization

```crystal
# src/actions/mixins/require_admin.cr
module RequireAdmin
  macro included
    before admin_required
  end
  
  private def admin_required
    if current_user?.try(&.admin?)
      continue
    elsif signed_in?
      flash.failure = "คุณไม่มีสิทธิ์เข้าถึงหน้านี้"
      redirect to: Home::Index
    else
      session.set(:redirect_after_login, request.path)
      redirect to: SignIns::New
    end
  end
end

# Authorization helper ทั่วไป
module Authorize
  macro included
    before check_authorization
  end
  
  # Override ใน subclasses
  abstract def authorize? : Bool
  
  private def check_authorization
    if authorize?
      continue
    else
      flash.failure = "ไม่มีสิทธิ์"
      redirect to: Home::Index
    end
  end
end

# ใช้งาน
class Admin::Users::Index < BrowserAction
  include RequireSignIn
  include RequireAdmin
  
  get "/admin/users" do
    users = UserQuery.new.order_alphabetically.select
    render Admin::Users::IndexPage, users: users
  end
end
```

## Password Reset

```crystal
# Token model
class PasswordResetToken < BaseModel
  table do
    primary_key id : Int64
    add_timestamps
    column token : String
    column expires_at : Time
    column used : Bool
    belongs_to user : User
  end
  
  def expired? : Bool
    expires_at < Time.local
  end
  
  def valid_for_use? : Bool
    !expired? && !used
  end
end

# Request reset
class PasswordResets::Create < BrowserAction
  post "/password_resets" do
    email = params.get?(:email) || ""
    user = UserQuery.new.email(email.downcase).first?
    
    if user
      # สร้าง token
      token = Random::Secure.urlsafe_base64(32)
      
      SavePasswordResetToken.create!(
        user_id: user.id,
        token: token,
        expires_at: Time.local + 1.hour,
        used: false
      )
      
      # ส่ง email
      spawn do
        PasswordResetMailer.deliver(user: user, token: token)
      end
    end
    
    # แสดงข้อความเดิมเสมอ (ป้องกัน email enumeration)
    flash.success = "ถ้าอีเมลนี้มีในระบบ คุณจะได้รับอีเมลรีเซ็ตรหัสผ่าน"
    redirect to: SignIns::New
  end
end

# Process reset
class PasswordResets::Update < BrowserAction
  put "/password_resets/:token" do
    reset_token = PasswordResetTokenQuery.new.token(token).first?
    
    unless reset_token && reset_token.valid_for_use?
      flash.failure = "ลิงก์รีเซ็ตรหัสผ่านไม่ถูกต้องหรือหมดอายุแล้ว"
      redirect to: SignIns::New
      next
    end
    
    new_password = params.get?(:password) || ""
    confirmation = params.get?(:password_confirmation) || ""
    
    if new_password.size < 8
      flash.failure = "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"
      redirect to: PasswordResets::Edit.with(token: token)
      next
    end
    
    if new_password != confirmation
      flash.failure = "รหัสผ่านไม่ตรงกัน"
      redirect to: PasswordResets::Edit.with(token: token)
      next
    end
    
    # อัปเดต password
    user = reset_token.user
    UpdateUserPassword.update!(user,
      hashed_password: Crypto::Bcrypt::Password.create(new_password).to_s
    )
    
    # Mark token as used
    SavePasswordResetToken.update!(reset_token, used: true)
    
    flash.success = "รีเซ็ตรหัสผ่านสำเร็จ กรุณาเข้าสู่ระบบด้วยรหัสผ่านใหม่"
    redirect to: SignIns::New
  end
end
```

## Sign In Page

```crystal
# src/pages/sign_ins/new_page.cr
class SignIns::NewPage < AuthLayout
  def content
    div class: "sign-in-container" do
      h1 "เข้าสู่ระบบ"
      
      if flash["failure"]?
        div class: "alert alert-danger" do
          text flash["failure"]
        end
      end
      
      form_for SignIns::Create do
        div class: "form-group" do
          label "อีเมล"
          email_input :email, class: "form-control", autofocus: "true"
        end
        
        div class: "form-group" do
          label "รหัสผ่าน"
          password_input :password, class: "form-control"
        end
        
        div class: "form-check" do
          checkbox_input :remember_me, class: "form-check-input"
          label "จำฉันไว้", class: "form-check-label"
        end
        
        submit "เข้าสู่ระบบ", class: "btn btn-primary w-100"
      end
      
      div class: "auth-links" do
        link "ลืมรหัสผ่าน?", to: PasswordResets::New
        text " · "
        link "สร้างบัญชีใหม่", to: Users::New
      end
    end
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Remember Me Functionality

```crystal
# Remember me ด้วย persistent cookie

module RememberMe
  COOKIE_NAME = "remember_token"
  TOKEN_EXPIRE = 30.days
  
  def remember_user(user : User)
    token = Random::Secure.urlsafe_base64(32)
    
    # บันทึก token ใน database
    SaveRememberToken.create!(
      user_id: user.id,
      token: Digest::SHA256.hexdigest(token),  # hash ก่อนเก็บ
      expires_at: Time.local + TOKEN_EXPIRE
    )
    
    # Set cookie
    cookies[COOKIE_NAME] = HTTP::Cookie.new(
      name: COOKIE_NAME,
      value: token,
      expires: Time.local + TOKEN_EXPIRE,
      http_only: true,
      secure: Lucky.env.production?
    )
  end
  
  def forget_user
    if raw_token = cookies[COOKIE_NAME]?.try(&.value)
      token_hash = Digest::SHA256.hexdigest(raw_token)
      RememberTokenQuery.new.token(token_hash).delete
    end
    
    cookies.delete(COOKIE_NAME)
  end
  
  def current_user? : User?
    @current_user ||= begin
      # ลอง session ก่อน
      if user_id_str = session.get?(:user_id)
        UserQuery.find?(user_id_str.to_i64? || 0_i64)
      elsif raw_token = cookies[COOKIE_NAME]?.try(&.value)
        # ลอง remember token
        token_hash = Digest::SHA256.hexdigest(raw_token)
        remember_token = RememberTokenQuery.new
          .token(token_hash)
          .not_expired
          .first?
        
        if remember_token
          user = remember_token.user
          sign_in(user)  # refresh session
          user
        end
      end
    end
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Session Authentication**: session.set/get/delete
2. **AuthHelpers**: sign_in, sign_out, current_user?
3. **Sign In Flow**: verify password, handle failed attempts
4. **Account Locking**: lock account หลัง login ผิดหลายครั้ง
5. **RequireSignIn Pipe**: ป้องกัน protected routes
6. **Role-Based Authorization**: RequireAdmin pipe
7. **Password Reset**: token-based reset flow
8. **Remember Me**: persistent login cookie
9. **Security**: timing attack prevention, token hashing

Authentication ที่ดีต้องคำนึงถึง: brute force protection, timing attacks, secure token storage
