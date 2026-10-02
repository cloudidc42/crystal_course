# Part 149: Security Headers - Security Headers ใน Web Application

## บทนำ

Security headers ช่วยป้องกัน web applications จาก attacks ต่างๆ เช่น XSS, clickjacking, MIME sniffing และอื่นๆ

## Essential Security Headers

```crystal
require "kemal"

# Middleware สำหรับ security headers
class SecurityHeadersHandler < Kemal::Handler
  def call(context : HTTP::Server::Context)
    call_next(context)
    
    # 1. X-Content-Type-Options - ป้องกัน MIME sniffing
    context.response.headers["X-Content-Type-Options"] = "nosniff"
    
    # 2. X-Frame-Options - ป้องกัน clickjacking
    context.response.headers["X-Frame-Options"] = "SAMEORIGIN"
    
    # 3. X-XSS-Protection - เปิดใช้ browser XSS filter (legacy)
    context.response.headers["X-XSS-Protection"] = "1; mode=block"
    
    # 4. Referrer-Policy - ควบคุมข้อมูลที่ส่งใน Referer header
    context.response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    
    # 5. X-Permitted-Cross-Domain-Policies - ป้องกัน Flash/PDF cross-domain
    context.response.headers["X-Permitted-Cross-Domain-Policies"] = "none"
    
    # 6. X-Download-Options - ป้องกัน IE จากการ open files
    context.response.headers["X-Download-Options"] = "noopen"
  end
end

add_handler SecurityHeadersHandler.new
```

## Content Security Policy (CSP)

```crystal
require "kemal"

class CSPBuilder
  def initialize
    @directives = {} of String => Array(String)
  end
  
  def default_src(*sources)
    @directives["default-src"] = sources.to_a
    self
  end
  
  def script_src(*sources)
    @directives["script-src"] = sources.to_a
    self
  end
  
  def style_src(*sources)
    @directives["style-src"] = sources.to_a
    self
  end
  
  def img_src(*sources)
    @directives["img-src"] = sources.to_a
    self
  end
  
  def connect_src(*sources)
    @directives["connect-src"] = sources.to_a
    self
  end
  
  def font_src(*sources)
    @directives["font-src"] = sources.to_a
    self
  end
  
  def frame_src(*sources)
    @directives["frame-src"] = sources.to_a
    self
  end
  
  def object_src(*sources)
    @directives["object-src"] = sources.to_a
    self
  end
  
  def media_src(*sources)
    @directives["media-src"] = sources.to_a
    self
  end
  
  def form_action(*sources)
    @directives["form-action"] = sources.to_a
    self
  end
  
  def frame_ancestors(*sources)
    @directives["frame-ancestors"] = sources.to_a
    self
  end
  
  def upgrade_insecure_requests
    @directives["upgrade-insecure-requests"] = [] of String
    self
  end
  
  def block_all_mixed_content
    @directives["block-all-mixed-content"] = [] of String
    self
  end
  
  def report_uri(uri : String)
    @directives["report-uri"] = [uri]
    self
  end
  
  def build : String
    @directives.map do |directive, sources|
      if sources.empty?
        directive
      else
        "#{directive} #{sources.join(" ")}"
      end
    end.join("; ")
  end
end

# สร้าง CSP
PRODUCTION_CSP = CSPBuilder.new
  .default_src("'self'")
  .script_src("'self'", "'unsafe-inline'", "https://cdnjs.cloudflare.com")  # ระวัง unsafe-inline
  .style_src("'self'", "'unsafe-inline'", "https://fonts.googleapis.com")
  .font_src("'self'", "https://fonts.gstatic.com")
  .img_src("'self'", "data:", "https:")
  .connect_src("'self'", "https://api.example.com")
  .frame_src("'none'")
  .object_src("'none'")
  .media_src("'self'")
  .form_action("'self'")
  .frame_ancestors("'none'")
  .upgrade_insecure_requests
  .report_uri("/csp-report")
  .build

# Nonce-based CSP (ปลอดภัยกว่า unsafe-inline)
class NonceCSPHandler < Kemal::Handler
  def call(context : HTTP::Server::Context)
    # สร้าง nonce ใหม่ทุก request
    nonce = Random::Secure.base64(16)
    context.set("csp_nonce", nonce)
    
    call_next(context)
    
    csp = CSPBuilder.new
      .default_src("'self'")
      .script_src("'self'", "'nonce-#{nonce}'")
      .style_src("'self'", "'nonce-#{nonce}'")
      .img_src("'self'", "data:", "https:")
      .object_src("'none'")
      .frame_ancestors("'none'")
      .build
    
    context.response.headers["Content-Security-Policy"] = csp
    # Report-only mode สำหรับ testing
    # context.response.headers["Content-Security-Policy-Report-Only"] = csp
  end
end

add_handler NonceCSPHandler.new
```

## HSTS (HTTP Strict Transport Security)

```crystal
require "kemal"

class HSTSHandler < Kemal::Handler
  def initialize(
    @max_age : Int32 = 31536000,  # 1 year
    @include_subdomains : Bool = true,
    @preload : Bool = false
  )
  end
  
  def call(context : HTTP::Server::Context)
    # เฉพาะ HTTPS เท่านั้น
    is_https = context.request.headers["X-Forwarded-Proto"]? == "https" ||
               context.request.headers["X-Forwarded-SSL"]? == "on"
    
    call_next(context)
    
    if is_https
      hsts = "max-age=#{@max_age}"
      hsts += "; includeSubDomains" if @include_subdomains
      hsts += "; preload" if @preload
      
      context.response.headers["Strict-Transport-Security"] = hsts
    end
  end
end

# HTTP -> HTTPS redirect
before_all do |env|
  is_https = env.request.headers["X-Forwarded-Proto"]? == "https"
  is_secure_env = ENV["HTTPS_ONLY"]? == "true"
  
  if is_secure_env && !is_https && env.request.path != "/health"
    env.response.headers["Location"] = "https://#{env.request.headers["Host"]}#{env.request.path}"
    halt env, status_code: 301, response: "Redirecting to HTTPS"
  end
end

add_handler HSTSHandler.new(
  max_age: 31536000,
  include_subdomains: true,
  preload: true  # สำหรับ HSTS preload list
)
```

## Permissions Policy

```crystal
require "kemal"

class PermissionsPolicyHandler < Kemal::Handler
  def call(context : HTTP::Server::Context)
    call_next(context)
    
    # ควบคุม browser features
    policy = [
      "geolocation=()",        # ปิด geolocation
      "microphone=()",          # ปิด microphone
      "camera=()",              # ปิด camera
      "payment=()",             # ปิด payment
      "usb=()",                 # ปิด USB
      "magnetometer=()",        # ปิด magnetometer
      "gyroscope=()",           # ปิด gyroscope
      "accelerometer=()",       # ปิด accelerometer
      "ambient-light-sensor=()", # ปิด ambient light
      "autoplay=(self)",        # อนุญาต autoplay เฉพาะ self
      "fullscreen=(self)",      # อนุญาต fullscreen เฉพาะ self
    ]
    
    context.response.headers["Permissions-Policy"] = policy.join(", ")
  end
end

add_handler PermissionsPolicyHandler.new
```

## Complete Security Headers Setup

```crystal
require "kemal"

class FullSecurityHeadersHandler < Kemal::Handler
  property csp : String
  property hsts_max_age : Int32
  property environment : String
  
  def initialize(@environment = "production")
    @hsts_max_age = 31536000
    @csp = build_csp
  end
  
  def call(context : HTTP::Server::Context)
    call_next(context)
    set_all_headers(context)
  end
  
  private def set_all_headers(context : HTTP::Server::Context)
    res = context.response
    
    # Prevent MIME sniffing
    res.headers["X-Content-Type-Options"] = "nosniff"
    
    # Prevent clickjacking
    res.headers["X-Frame-Options"] = "DENY"
    
    # XSS protection (legacy browsers)
    res.headers["X-XSS-Protection"] = "1; mode=block"
    
    # Referrer policy
    res.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"
    
    # Content Security Policy
    res.headers["Content-Security-Policy"] = @csp
    
    # Permissions Policy
    res.headers["Permissions-Policy"] = permissions_policy
    
    # HSTS (HTTPS only)
    if production? && is_https?(context)
      res.headers["Strict-Transport-Security"] = 
        "max-age=#{@hsts_max_age}; includeSubDomains; preload"
    end
    
    # Remove revealing headers
    res.headers.delete("Server")
    res.headers.delete("X-Powered-By")
    
    # Add custom headers
    res.headers["X-API-Version"] = "1.0"
  end
  
  private def build_csp : String
    if @environment == "development"
      # Relaxed CSP สำหรับ development
      "default-src 'self' 'unsafe-inline' 'unsafe-eval'; img-src *; connect-src *"
    else
      CSPBuilder.new
        .default_src("'self'")
        .script_src("'self'")
        .style_src("'self'", "'unsafe-inline'")
        .img_src("'self'", "data:", "https:")
        .connect_src("'self'")
        .font_src("'self'")
        .object_src("'none'")
        .frame_ancestors("'none'")
        .upgrade_insecure_requests
        .build
    end
  end
  
  private def permissions_policy : String
    "geolocation=(), microphone=(), camera=(), payment=(), usb=()"
  end
  
  private def production? : Bool
    @environment == "production"
  end
  
  private def is_https?(context : HTTP::Server::Context) : Bool
    context.request.headers["X-Forwarded-Proto"]? == "https"
  end
end

add_handler FullSecurityHeadersHandler.new(
  environment: ENV["CRYSTAL_ENV"]? || "development"
)

# CSP Report endpoint
post "/csp-report" do |env|
  body = env.request.body.try(&.gets_to_end) || ""
  
  begin
    report = JSON.parse(body)
    puts "CSP Violation: #{report["csp-report"]?.to_json}"
    # ใน production: บันทึกลง log service
  rescue
    puts "Invalid CSP report"
  end
  
  env.response.status_code = 204
  ""
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Security Audit Middleware

```crystal
require "kemal"

class SecurityAuditHandler < Kemal::Handler
  def call(context : HTTP::Server::Context)
    call_next(context)
    
    issues = [] of String
    
    # ตรวจสอบ security headers ที่ควรมี
    required_headers = {
      "X-Content-Type-Options" => "nosniff",
      "X-Frame-Options" => "DENY",
      "Content-Security-Policy" => nil,  # ต้องมีแต่ไม่ต้องตรวจ value
    }
    
    required_headers.each do |header, expected_value|
      if actual = context.response.headers[header]?
        if expected_value && actual != expected_value
          issues << "#{header}: expected '#{expected_value}', got '#{actual}'"
        end
      else
        issues << "Missing header: #{header}"
      end
    end
    
    # Log issues ใน development
    if ENV["CRYSTAL_ENV"]? == "development" && issues.any?
      puts "Security header issues for #{context.request.path}:"
      issues.each { |issue| puts "  - #{issue}" }
    end
    
    # เพิ่ม debug header ใน development
    if ENV["CRYSTAL_ENV"]? == "development"
      context.response.headers["X-Security-Issues"] = issues.size.to_s
    end
  end
end

add_handler SecurityAuditHandler.new
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **X-Content-Type-Options**: ป้องกัน MIME sniffing
2. **X-Frame-Options**: ป้องกัน clickjacking
3. **CSP**: กำหนดแหล่งที่มาของ resources
4. **Nonce-based CSP**: ปลอดภัยกว่า unsafe-inline
5. **HSTS**: บังคับ HTTPS
6. **Permissions Policy**: ควบคุม browser APIs
7. **Referrer-Policy**: ควบคุม Referer header
8. **Remove Server headers**: ซ่อนข้อมูล server
9. **CSP Reporting**: รายงาน violations
10. **Security Audit**: ตรวจสอบ headers อัตโนมัติ

Security headers เป็น defense-in-depth: แต่ละ header ป้องกัน attack เฉพาะแบบ
