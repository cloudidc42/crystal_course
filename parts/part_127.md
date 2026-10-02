# Part 127: SSL/TLS - การเข้ารหัสด้วย OpenSSL ใน Crystal

## บทนำ

SSL/TLS (Secure Sockets Layer/Transport Layer Security) เป็นโปรโตคอลสำหรับเข้ารหัสการสื่อสารทางเครือข่าย Crystal ใช้ OpenSSL ผ่าน `openssl` library

## HTTPS Client

```crystal
require "http/client"

# HTTPS request พื้นฐาน (default ตรวจสอบ certificate)
response = HTTP::Client.get("https://httpbin.org/get")
puts response.status_code
puts response.body[0..200]

# ด้วย custom TLS context
tls_context = OpenSSL::SSL::Context::Client.new
tls_context.verify_mode = OpenSSL::SSL::VerifyMode::PEER  # ตรวจสอบ cert (default)
tls_context.add_trust_store_defaults  # ใช้ system CA store

HTTP::Client.new("httpbin.org", tls: tls_context) do |client|
  response = client.get("/get")
  puts "Status: #{response.status_code}"
end
```

## SSL Certificate Verification

```crystal
require "http/client"
require "openssl"

# ตรวจสอบ certificate อย่างเข้มงวด
class SecureHTTPClient
  def initialize(@host : String)
    @context = create_strict_context
  end
  
  def get(path : String) : HTTP::Client::Response
    HTTP::Client.new(@host, tls: @context) do |client|
      client.get(path)
    end
  end
  
  private def create_strict_context : OpenSSL::SSL::Context::Client
    ctx = OpenSSL::SSL::Context::Client.new
    
    # ต้องการ PEER verification
    ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
    
    # โหลด system CA certificates
    ctx.add_trust_store_defaults
    
    # กำหนด cipher suites ที่ปลอดภัย
    ctx.ciphers = "ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-GCM-SHA256"
    
    ctx
  end
end

client = SecureHTTPClient.new("httpbin.org")
begin
  response = client.get("/get")
  puts "Secure connection: #{response.status_code}"
rescue OpenSSL::SSL::Error => ex
  puts "SSL Error: #{ex.message}"
end
```

## Self-Signed Certificates

```crystal
require "openssl"

# สร้าง self-signed certificate ด้วย Crystal
# (ต้องใช้ command line tool)

# สร้างด้วย openssl command
def generate_self_signed_cert(output_dir : String = ".")
  # สร้าง private key
  system("openssl genrsa -out #{output_dir}/server.key 2048")
  
  # สร้าง self-signed certificate
  system(
    "openssl req -new -x509 -key #{output_dir}/server.key " +
    "-out #{output_dir}/server.crt -days 365 " +
    "-subj \"/C=TH/ST=Bangkok/O=Crystal Test/CN=localhost\""
  )
  
  puts "สร้าง certificate สำเร็จ"
end

# Server ที่ใช้ self-signed cert
def start_tls_server(port : Int32, cert_file : String, key_file : String)
  server = TCPServer.new("0.0.0.0", port)
  
  tls_context = OpenSSL::SSL::Context::Server.new
  tls_context.certificate_chain = cert_file
  tls_context.private_key = key_file
  
  puts "TLS Server on port #{port}"
  
  while client = server.accept?
    spawn do
      begin
        tls_socket = OpenSSL::SSL::Socket::Server.new(client, tls_context)
        
        while line = tls_socket.gets
          tls_socket.puts "Secure Echo: #{line}"
        end
      rescue OpenSSL::SSL::Error => ex
        puts "TLS Error: #{ex.message}"
      ensure
        client.close rescue nil
      end
    end
  end
end
```

## TLS Context

```crystal
require "openssl"

# Server TLS Context
def create_server_context(cert_file : String, key_file : String, ca_file : String? = nil) : OpenSSL::SSL::Context::Server
  ctx = OpenSSL::SSL::Context::Server.new
  
  # Certificate และ private key
  ctx.certificate_chain = cert_file
  ctx.private_key = key_file
  
  # CA certificate สำหรับ mutual TLS (optional)
  if ca = ca_file
    ctx.ca_certificates = ca
    ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER | OpenSSL::SSL::VerifyMode::FAIL_IF_NO_PEER_CERT
  end
  
  # TLS version
  ctx.minimum_version = OpenSSL::SSL::Version::TLS1_2
  
  # Cipher suites
  ctx.ciphers = "HIGH:!aNULL:!MD5"
  
  ctx
end

# Client TLS Context
def create_client_context(ca_file : String? = nil, verify : Bool = true) : OpenSSL::SSL::Context::Client
  ctx = OpenSSL::SSL::Context::Client.new
  
  if verify
    ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
    
    if ca = ca_file
      ctx.ca_certificates = ca
    else
      ctx.add_trust_store_defaults
    end
  else
    # ไม่ตรวจสอบ certificate (ไม่ปลอดภัย! ใช้ใน dev เท่านั้น)
    ctx.verify_mode = OpenSSL::SSL::VerifyMode::NONE
  end
  
  ctx.minimum_version = OpenSSL::SSL::Version::TLS1_2
  ctx
end
```

## Mutual TLS (mTLS)

```crystal
require "openssl"
require "socket"

# Mutual TLS: ทั้ง client และ server ต้องมี certificate

# mTLS Server
def start_mtls_server(port : Int32, server_cert : String, server_key : String, ca_cert : String)
  server = TCPServer.new("0.0.0.0", port)
  
  ctx = OpenSSL::SSL::Context::Server.new
  ctx.certificate_chain = server_cert
  ctx.private_key = server_key
  ctx.ca_certificates = ca_cert
  # บังคับให้ client ส่ง certificate
  ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER | OpenSSL::SSL::VerifyMode::FAIL_IF_NO_PEER_CERT
  
  puts "mTLS Server on port #{port}"
  
  while client = server.accept?
    spawn do
      begin
        tls = OpenSSL::SSL::Socket::Server.new(client, ctx)
        
        # ดู client certificate
        cert = tls.peer_certificate
        puts "Client cert subject: #{cert.subject}" if cert
        
        tls.puts "Hello from mTLS Server!"
        tls.close
      rescue OpenSSL::SSL::Error => ex
        puts "mTLS Error: #{ex.message}"
      ensure
        client.close
      end
    end
  end
end

# mTLS Client
def connect_mtls(host : String, port : Int32, client_cert : String, client_key : String, ca_cert : String)
  socket = TCPSocket.new(host, port)
  
  ctx = OpenSSL::SSL::Context::Client.new
  ctx.certificate_chain = client_cert
  ctx.private_key = client_key
  ctx.ca_certificates = ca_cert
  ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
  
  tls = OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: host)
  
  response = tls.gets
  puts "Server: #{response}"
  
  tls.close
  socket.close
end
```

## Certificate Inspection

```crystal
require "openssl"

# อ่านและตรวจสอบ certificate
def inspect_certificate(cert_file : String)
  cert = OpenSSL::X509::Certificate.new(File.read(cert_file))
  
  puts "Subject: #{cert.subject}"
  puts "Issuer: #{cert.issuer}"
  puts "Serial: #{cert.serial}"
  puts "Not Before: #{cert.not_before}"
  puts "Not After: #{cert.not_after}"
  puts "Expired: #{cert.not_after < Time.local}"
  
  # Days until expiry
  days_left = (cert.not_after - Time.local).total_days.to_i
  puts "Days until expiry: #{days_left}"
  
  if days_left < 30
    puts "WARNING: Certificate expires soon!"
  end
end

# ตรวจสอบ certificate จาก URL
def check_remote_cert(host : String, port : Int32 = 443)
  socket = TCPSocket.new(host, port)
  
  ctx = OpenSSL::SSL::Context::Client.new
  ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
  ctx.add_trust_store_defaults
  
  tls = OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: host)
  
  cert = tls.peer_certificate
  if cert
    puts "=== Certificate for #{host} ==="
    puts "Subject: #{cert.subject}"
    puts "Issuer: #{cert.issuer}"
    days_left = (cert.not_after - Time.local).total_days.to_i
    puts "Expires in: #{days_left} days"
    puts "Valid: #{cert.not_after > Time.local}"
  end
  
  tls.close
  socket.close
rescue OpenSSL::SSL::Error => ex
  puts "SSL Error: #{ex.message}"
rescue Socket::Error => ex
  puts "Connection Error: #{ex.message}"
end

check_remote_cert("google.com")
```

## HTTPS Server

```crystal
require "http/server"
require "openssl"

# สร้าง HTTPS server
def create_https_server(cert_file : String, key_file : String, port : Int32 = 443)
  tls_context = OpenSSL::SSL::Context::Server.new
  tls_context.certificate_chain = cert_file
  tls_context.private_key = key_file
  tls_context.minimum_version = OpenSSL::SSL::Version::TLS1_2
  
  server = HTTP::Server.new do |context|
    context.response.content_type = "application/json"
    context.response.print({"message" => "Secure Crystal Server!", "tls" => true}.to_json)
  end
  
  server.bind_tls("0.0.0.0", port, tls_context)
  
  puts "HTTPS Server on https://localhost:#{port}"
  server.listen
end

# ถ้ามี self-signed cert
# create_https_server("server.crt", "server.key", 8443)
```

## Certificate Pinning

```crystal
require "openssl"
require "http/client"

# Certificate Pinning: ตรวจสอบว่า certificate ตรงกับที่คาดไว้
class PinnedHTTPSClient
  # SHA256 fingerprint ของ certificate ที่ trust
  PINNED_FINGERPRINTS = {
    "google.com" => "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    # เพิ่ม pinned certs จริงๆ ที่นี่
  }
  
  def get(url : String) : HTTP::Client::Response?
    uri = URI.parse(url)
    host = uri.host || raise "Invalid URL"
    
    # ตรวจสอบ pinned fingerprint
    if expected = PINNED_FINGERPRINTS[host]?
      socket = TCPSocket.new(host, 443)
      ctx = OpenSSL::SSL::Context::Client.new
      ctx.verify_mode = OpenSSL::SSL::VerifyMode::NONE  # เราทำ pinning เอง
      
      tls = OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: host)
      
      cert = tls.peer_certificate
      actual = cert ? cert_fingerprint(cert) : ""
      
      tls.close
      socket.close
      
      unless actual == expected
        raise "Certificate pinning failed! Expected: #{expected}, Got: #{actual}"
      end
    end
    
    HTTP::Client.get(url)
  end
  
  private def cert_fingerprint(cert : OpenSSL::X509::Certificate) : String
    # คำนวณ SHA256 fingerprint
    digest = OpenSSL::Digest::SHA256.new
    digest.update(cert.to_der)
    digest.hexfinal
  end
end
```

## TLS Session Resumption

```crystal
require "openssl"
require "socket"

# TLS Session Caching สำหรับ performance
class TLSSessionCache
  def initialize
    @sessions = {} of String => OpenSSL::SSL::Session
    @mutex = Mutex.new
  end
  
  def get(key : String) : OpenSSL::SSL::Session?
    @mutex.synchronize { @sessions[key]? }
  end
  
  def store(key : String, session : OpenSSL::SSL::Session)
    @mutex.synchronize { @sessions[key] = session }
  end
  
  def remove(key : String)
    @mutex.synchronize { @sessions.delete(key) }
  end
end

# ใช้ session resumption
cache = TLSSessionCache.new

def connect_with_session_resumption(host : String, port : Int32, cache : TLSSessionCache) : OpenSSL::SSL::Socket::Client
  socket = TCPSocket.new(host, port)
  
  ctx = OpenSSL::SSL::Context::Client.new
  ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
  ctx.add_trust_store_defaults
  
  tls = OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: host)
  
  cache_key = "#{host}:#{port}"
  
  if session = cache.get(cache_key)
    # Resume session
    tls.session = session
    puts "Using cached TLS session"
  else
    puts "New TLS handshake"
  end
  
  # บันทึก session สำหรับครั้งหน้า
  cache.store(cache_key, tls.session)
  
  tls
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Certificate Monitor

```crystal
require "socket"
require "openssl"

# ตรวจสอบ certificate หลายๆ domains
class CertificateMonitor
  struct CertInfo
    property domain : String
    property subject : String
    property issuer : String
    property not_after : Time
    property days_left : Int32
    property error : String?
    
    def initialize(@domain, @subject, @issuer, @not_after)
      @days_left = (@not_after - Time.local).total_days.to_i
      @error = nil
    end
    
    def self.error(domain : String, message : String) : CertInfo
      info = new(domain, "", "", Time.local)
      info.@error = message
      info
    end
    
    def valid?
      @error.nil? && @days_left > 0
    end
    
    def expiring_soon?
      valid? && @days_left < 30
    end
  end
  
  def check(domains : Array(String)) : Array(CertInfo)
    results = Channel(CertInfo).new(domains.size)
    
    domains.each do |domain|
      spawn do
        results.send(check_domain(domain))
      end
    end
    
    Array(CertInfo).new(domains.size) { results.receive }
  end
  
  def report(infos : Array(CertInfo))
    puts "=== Certificate Report ==="
    
    infos.sort_by { |i| i.days_left }.each do |info|
      if err = info.error
        puts "❌ #{info.domain}: #{err}"
      elsif info.expiring_soon?
        puts "⚠️  #{info.domain}: #{info.days_left} days left (#{info.not_after})"
      else
        puts "✅ #{info.domain}: #{info.days_left} days left"
      end
    end
    
    expiring = infos.count(&.expiring_soon?)
    puts "\nSummary: #{infos.count(&.valid?)} valid, #{expiring} expiring soon, #{infos.count { |i| !i.error.nil? }} errors"
  end
  
  private def check_domain(domain : String) : CertInfo
    socket = TCPSocket.new(domain, 443, connect_timeout: 5.seconds)
    
    ctx = OpenSSL::SSL::Context::Client.new
    ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
    ctx.add_trust_store_defaults
    
    tls = OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: domain)
    cert = tls.peer_certificate
    
    tls.close
    socket.close
    
    if cert
      CertInfo.new(domain, cert.subject.to_s, cert.issuer.to_s, cert.not_after)
    else
      CertInfo.error(domain, "No certificate")
    end
  rescue ex
    CertInfo.error(domain, ex.message || "Unknown error")
  end
end

monitor = CertificateMonitor.new
domains = ["google.com", "cloudflare.com", "github.com", "httpbin.org"]
infos = monitor.check(domains)
monitor.report(infos)
```

### แบบฝึกหัดที่ 2: Secure Config Loader

```crystal
require "openssl"

# โหลด config ที่เข้ารหัสด้วย OpenSSL AES
class SecureConfig
  def initialize(@key_file : String)
    @key = File.read(@key_file).strip
  end
  
  def encrypt(data : String) : String
    cipher = OpenSSL::Cipher.new("AES-256-CBC")
    cipher.encrypt
    
    # สร้าง key และ IV จาก passphrase
    key_iv = cipher.random_key_iv_from_password(@key, "salt")
    
    encrypted = cipher.update(data)
    encrypted += cipher.final
    
    Base64.strict_encode(encrypted)
  end
  
  def decrypt(encrypted_data : String) : String
    cipher = OpenSSL::Cipher.new("AES-256-CBC")
    cipher.decrypt
    
    data = Base64.decode(encrypted_data)
    cipher.random_key_iv_from_password(@key, "salt")
    
    decrypted = cipher.update(data)
    decrypted += cipher.final
    
    String.new(decrypted)
  end
  
  def save(config : Hash(String, String), output_file : String)
    json = config.to_json
    encrypted = encrypt(json)
    File.write(output_file, encrypted)
  end
  
  def load(config_file : String) : Hash(String, String)
    encrypted = File.read(config_file)
    json = decrypt(encrypted)
    Hash(String, String).from_json(json)
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HTTPS Client**: การทำ HTTPS requests พร้อม certificate verification
2. **SSL Certificate Verification**: ตรวจสอบ certificates
3. **TLS Context**: กำหนด TLS settings สำหรับ server และ client
4. **Self-Signed Certificates**: สร้างและใช้ self-signed certs
5. **Mutual TLS (mTLS)**: ทั้ง client และ server มี certificates
6. **Certificate Inspection**: อ่านและตรวจสอบ cert details
7. **HTTPS Server**: สร้าง secure HTTP server
8. **Certificate Pinning**: ป้องกัน MITM attacks
9. **Session Resumption**: เพิ่มประสิทธิภาพ TLS
10. **Certificate Monitoring**: แจ้งเตือนก่อน cert หมดอายุ

ข้อควรระวัง:
- อย่าปิด certificate verification ใน production
- ต่ออายุ certificates ก่อนหมดอายุ (แนะนำ 30+ วัน)
- ใช้ TLS 1.2+ เท่านั้น
- ใช้ strong cipher suites
