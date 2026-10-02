# Part 128: Email (SMTP) - การส่งอีเมลใน Crystal

## บทนำ

Crystal ไม่มี built-in SMTP library แต่เราสามารถใช้ external shards หรือสร้าง SMTP client เองได้ ในบทนี้เราจะเรียนรู้การส่งอีเมลแบบต่างๆ ทั้ง text, HTML, และ attachments

## การตั้งค่า shard

เพิ่มใน `shard.yml`:

```yaml
dependencies:
  smtp:
    github: arcage/crystal-email
    version: ~> 0.5.0
```

หรือใช้ shard อื่น:

```yaml
dependencies:
  action-mailer:
    github: crystal-garage/action-mailer
```

## SMTP พื้นฐาน

```crystal
# การส่งอีเมลผ่าน SMTP แบบ raw
require "socket"
require "openssl"
require "base64"

class SMTPClient
  def initialize(@host : String, @port : Int32, @username : String, @password : String, @tls : Bool = true)
  end
  
  def send_email(from : String, to : Array(String), subject : String, body : String, html_body : String? = nil)
    socket = TCPSocket.new(@host, @port)
    
    io = if @tls && @port == 465
      # SSL/TLS direct
      ctx = OpenSSL::SSL::Context::Client.new
      ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
      ctx.add_trust_store_defaults
      OpenSSL::SSL::Socket::Client.new(socket, ctx, hostname: @host)
    else
      socket
    end
    
    begin
      smtp_conversation(io, from, to, subject, body, html_body)
    ensure
      io.close rescue nil
      socket.close rescue nil
    end
  end
  
  private def smtp_conversation(io : IO, from : String, to : Array(String), subject : String, body : String, html_body : String?)
    # อ่าน greeting
    greeting = io.gets
    raise "SMTP greeting failed: #{greeting}" unless greeting&.starts_with?("220")
    
    # EHLO
    io.puts "EHLO #{@host}"
    while line = io.gets
      break if line.starts_with?("250 ")
      raise "EHLO failed: #{line}" if !line.starts_with?("250")
    end
    
    # STARTTLS (ถ้า port 587)
    if @port == 587
      io.puts "STARTTLS"
      response = io.gets
      raise "STARTTLS failed: #{response}" unless response&.starts_with?("220")
      
      # Upgrade to TLS
      ctx = OpenSSL::SSL::Context::Client.new
      ctx.verify_mode = OpenSSL::SSL::VerifyMode::PEER
      ctx.add_trust_store_defaults
      io = OpenSSL::SSL::Socket::Client.new(io.as(TCPSocket), ctx, hostname: @host)
      
      io.puts "EHLO #{@host}"
      while line = io.gets
        break if line.starts_with?("250 ")
      end
    end
    
    # AUTH LOGIN
    io.puts "AUTH LOGIN"
    io.gets # 334 Username
    io.puts Base64.strict_encode(@username)
    io.gets # 334 Password
    io.puts Base64.strict_encode(@password)
    auth_response = io.gets
    raise "Authentication failed: #{auth_response}" unless auth_response&.starts_with?("235")
    
    # MAIL FROM
    io.puts "MAIL FROM:<#{from}>"
    response = io.gets
    raise "MAIL FROM failed: #{response}" unless response&.starts_with?("250")
    
    # RCPT TO
    to.each do |recipient|
      io.puts "RCPT TO:<#{recipient}>"
      response = io.gets
      raise "RCPT TO failed for #{recipient}: #{response}" unless response&.starts_with?("250")
    end
    
    # DATA
    io.puts "DATA"
    response = io.gets
    raise "DATA failed: #{response}" unless response&.starts_with?("354")
    
    # Headers
    io.puts "From: #{from}"
    io.puts "To: #{to.join(", ")}"
    io.puts "Subject: #{subject}"
    io.puts "Date: #{Time.local.to_rfc2822}"
    io.puts "MIME-Version: 1.0"
    
    if html_body
      boundary = "boundary_#{Random::Secure.hex(16)}"
      io.puts "Content-Type: multipart/alternative; boundary=\"#{boundary}\""
      io.puts ""
      
      # Text part
      io.puts "--#{boundary}"
      io.puts "Content-Type: text/plain; charset=UTF-8"
      io.puts ""
      io.puts body
      
      # HTML part
      io.puts "--#{boundary}"
      io.puts "Content-Type: text/html; charset=UTF-8"
      io.puts ""
      io.puts html_body
      io.puts "--#{boundary}--"
    else
      io.puts "Content-Type: text/plain; charset=UTF-8"
      io.puts ""
      io.puts body
    end
    
    io.puts "."
    response = io.gets
    raise "Message not accepted: #{response}" unless response&.starts_with?("250")
    
    # QUIT
    io.puts "QUIT"
    
    puts "Email sent successfully!"
  end
end

# การใช้งาน
smtp = SMTPClient.new(
  host: "smtp.gmail.com",
  port: 587,
  username: "your-email@gmail.com",
  password: "your-app-password"
)

smtp.send_email(
  from: "your-email@gmail.com",
  to: ["recipient@example.com"],
  subject: "ทดสอบจาก Crystal",
  body: "นี่คืออีเมลทดสอบจาก Crystal",
  html_body: "<h1>ทดสอบ HTML Email</h1><p>นี่คืออีเมลจาก <strong>Crystal</strong></p>"
)
```

## Email Builder

```crystal
require "base64"

class Email
  property from : String = ""
  property reply_to : String? = nil
  property to : Array(String) = [] of String
  property cc : Array(String) = [] of String
  property bcc : Array(String) = [] of String
  property subject : String = ""
  property text_body : String? = nil
  property html_body : String? = nil
  property attachments : Array(Attachment) = [] of Attachment
  property headers : Hash(String, String) = {} of String => String
  
  struct Attachment
    property filename : String
    property content : Bytes
    property mime_type : String
    
    def initialize(@filename, @content, @mime_type = "application/octet-stream")
    end
    
    def self.from_file(path : String) : Attachment
      filename = File.basename(path)
      content = File.read(path).to_slice
      mime_type = detect_mime_type(filename)
      new(filename, content, mime_type)
    end
    
    def self.detect_mime_type(filename : String) : String
      case File.extname(filename).downcase
      when ".pdf"  then "application/pdf"
      when ".jpg", ".jpeg" then "image/jpeg"
      when ".png"  then "image/png"
      when ".gif"  then "image/gif"
      when ".txt"  then "text/plain"
      when ".csv"  then "text/csv"
      when ".zip"  then "application/zip"
      else "application/octet-stream"
      end
    end
  end
  
  def self.build(&block : Email ->)
    email = new
    block.call(email)
    email
  end
  
  def to_mime : String
    io = IO::Memory.new
    
    boundary_mixed = "mixed_#{Random::Secure.hex(8)}"
    boundary_alt = "alt_#{Random::Secure.hex(8)}"
    has_attachments = !attachments.empty?
    is_multipart = text_body && html_body
    
    # Headers
    io.puts "From: #{from}"
    io.puts "Reply-To: #{reply_to}" if reply_to
    io.puts "To: #{to.join(", ")}"
    io.puts "Cc: #{cc.join(", ")}" unless cc.empty?
    io.puts "Bcc: #{bcc.join(", ")}" unless bcc.empty?
    io.puts "Subject: =?UTF-8?B?#{Base64.strict_encode(subject.to_slice)}?="
    io.puts "Date: #{Time.local.to_rfc2822}"
    io.puts "MIME-Version: 1.0"
    
    headers.each { |k, v| io.puts "#{k}: #{v}" }
    
    if has_attachments
      io.puts "Content-Type: multipart/mixed; boundary=\"#{boundary_mixed}\""
      io.puts ""
      io.puts "--#{boundary_mixed}"
    end
    
    if is_multipart
      io.puts "Content-Type: multipart/alternative; boundary=\"#{boundary_alt}\""
      io.puts ""
      
      if text = text_body
        io.puts "--#{boundary_alt}"
        io.puts "Content-Type: text/plain; charset=UTF-8"
        io.puts "Content-Transfer-Encoding: quoted-printable"
        io.puts ""
        io.puts text
        io.puts ""
      end
      
      if html = html_body
        io.puts "--#{boundary_alt}"
        io.puts "Content-Type: text/html; charset=UTF-8"
        io.puts "Content-Transfer-Encoding: quoted-printable"
        io.puts ""
        io.puts html
        io.puts ""
      end
      
      io.puts "--#{boundary_alt}--"
    elsif text = text_body
      io.puts "Content-Type: text/plain; charset=UTF-8"
      io.puts ""
      io.puts text
    elsif html = html_body
      io.puts "Content-Type: text/html; charset=UTF-8"
      io.puts ""
      io.puts html
    end
    
    # Attachments
    attachments.each do |att|
      io.puts "--#{boundary_mixed}"
      io.puts "Content-Type: #{att.mime_type}; name=\"#{att.filename}\""
      io.puts "Content-Transfer-Encoding: base64"
      io.puts "Content-Disposition: attachment; filename=\"#{att.filename}\""
      io.puts ""
      io.puts Base64.encode(att.content)
    end
    
    io.puts "--#{boundary_mixed}--" if has_attachments
    
    io.to_s
  end
end

# ใช้งาน Email builder
email = Email.build do |e|
  e.from = "sender@example.com"
  e.to = ["recipient1@example.com", "recipient2@example.com"]
  e.cc = ["cc@example.com"]
  e.subject = "รายงานประจำเดือน"
  e.text_body = "แนบรายงานประจำเดือนกุมภาพันธ์"
  e.html_body = """
    <html>
    <body>
      <h2>รายงานประจำเดือนกุมภาพันธ์</h2>
      <p>กรุณาตรวจสอบไฟล์แนบ</p>
    </body>
    </html>
  """
  # เพิ่ม attachment
  # e.attachments << Email::Attachment.from_file("report.pdf")
end

puts email.to_mime[0..500]
```

## HTML Email Templates

```crystal
# Email Template Engine
class EmailTemplate
  TEMPLATES = {
    "welcome" => {
      subject: "ยินดีต้อนรับสู่ {{app_name}}!",
      html: """
        <!DOCTYPE html>
        <html>
        <head>
          <meta charset="UTF-8">
          <style>
            body { font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto; }
            .header { background: #4a90e2; color: white; padding: 20px; text-align: center; }
            .content { padding: 20px; }
            .button { background: #4a90e2; color: white; padding: 10px 20px; 
                     text-decoration: none; border-radius: 4px; }
          </style>
        </head>
        <body>
          <div class="header">
            <h1>ยินดีต้อนรับ!</h1>
          </div>
          <div class="content">
            <p>สวัสดีคุณ <strong>{{username}}</strong>,</p>
            <p>ขอบคุณที่สมัครใช้บริการ <strong>{{app_name}}</strong></p>
            <p>กรุณาคลิกปุ่มด้านล่างเพื่อยืนยันอีเมล</p>
            <p>
              <a href="{{confirm_url}}" class="button">ยืนยันอีเมล</a>
            </p>
            <p>ลิงก์นี้จะหมดอายุใน 24 ชั่วโมง</p>
          </div>
        </body>
        </html>
      """,
    },
    "reset_password" => {
      subject: "รีเซ็ตรหัสผ่าน {{app_name}}",
      html: """
        <html>
        <body>
          <h2>รีเซ็ตรหัสผ่าน</h2>
          <p>คุณได้ขอรีเซ็ตรหัสผ่านสำหรับบัญชี <strong>{{username}}</strong></p>
          <p><a href="{{reset_url}}">คลิกที่นี่เพื่อรีเซ็ตรหัสผ่าน</a></p>
          <p>ถ้าคุณไม่ได้ร้องขอ กรุณาเพิกเฉยอีเมลนี้</p>
        </body>
        </html>
      """,
    },
  }
  
  def self.render(template_name : String, vars : Hash(String, String)) : {String, String}
    template = TEMPLATES[template_name]? || raise "Template not found: #{template_name}"
    
    subject = template[:subject]
    html = template[:html]
    
    vars.each do |key, value|
      subject = subject.gsub("{{#{key}}}", value)
      html = html.gsub("{{#{key}}}", value)
    end
    
    {subject, html}
  end
end

# ส่ง welcome email
subject, html = EmailTemplate.render("welcome", {
  "app_name" => "Crystal App",
  "username" => "สมชาย",
  "confirm_url" => "https://app.example.com/confirm/abc123",
})

puts "Subject: #{subject}"
puts "HTML: #{html[0..200]}..."
```

## Email Queue

```crystal
# Email Queue สำหรับส่ง email แบบ async
class EmailQueue
  struct EmailJob
    property email : Email
    property attempts : Int32
    property scheduled_at : Time
    
    def initialize(@email, @attempts = 0)
      @scheduled_at = Time.local
    end
  end
  
  def initialize(@smtp_client : SMTPClient, @workers : Int32 = 2)
    @queue = Channel(EmailJob).new(1000)
    @failed = [] of EmailJob
    @mutex = Mutex.new
    @sent_count = Atomic(Int32).new(0)
    @failed_count = Atomic(Int32).new(0)
    start_workers
  end
  
  def enqueue(email : Email)
    @queue.send(EmailJob.new(email))
  end
  
  def stats
    {
      "queued" => @queue.@size,
      "sent" => @sent_count.get,
      "failed" => @failed_count.get,
    }
  end
  
  private def start_workers
    @workers.times do |id|
      spawn do
        while job = @queue.receive?
          begin
            @smtp_client.send_email(
              from: job.email.from,
              to: job.email.to,
              subject: job.email.subject,
              body: job.email.text_body || "",
              html_body: job.email.html_body
            )
            @sent_count.add(1)
            puts "Worker #{id}: Sent email to #{job.email.to.join(", ")}"
          rescue ex
            puts "Worker #{id}: Failed to send: #{ex.message}"
            
            if job.attempts < 3
              # Retry
              sleep (2 ** job.attempts).seconds
              new_job = EmailJob.new(job.email, job.attempts + 1)
              @queue.send(new_job) rescue nil
            else
              @failed_count.add(1)
              @mutex.synchronize { @failed << job }
            end
          end
        end
      end
    end
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Notification System

```crystal
# ระบบแจ้งเตือนผ่านอีเมล
class NotificationService
  enum Type
    Alert
    Info
    Success
    Warning
  end
  
  struct Notification
    property recipient : String
    property type : Type
    property subject : String
    property message : String
    
    def initialize(@recipient, @type, @subject, @message)
    end
  end
  
  def initialize(@smtp : SMTPClient, @from : String)
    @queue = Channel(Notification).new(500)
    @templates = load_templates
    start_processor
  end
  
  def notify(recipient : String, type : Type, subject : String, message : String)
    @queue.send(Notification.new(recipient, type, subject, message))
  end
  
  def alert(recipient : String, message : String)
    notify(recipient, Type::Alert, "⚠️ Alert", message)
  end
  
  def info(recipient : String, message : String)
    notify(recipient, Type::Info, "ℹ️ Information", message)
  end
  
  private def load_templates : Hash(Type, String)
    {
      Type::Alert => """
        <div style="border-left:4px solid #e53e3e;padding:10px;background:#fff5f5">
        <h3 style="color:#e53e3e">⚠️ Alert</h3>
        <p>{{message}}</p>
        </div>
      """,
      Type::Info => """
        <div style="border-left:4px solid #3182ce;padding:10px;background:#ebf8ff">
        <h3 style="color:#3182ce">ℹ️ Information</h3>
        <p>{{message}}</p>
        </div>
      """,
      Type::Success => """
        <div style="border-left:4px solid #38a169;padding:10px;background:#f0fff4">
        <h3 style="color:#38a169">✅ Success</h3>
        <p>{{message}}</p>
        </div>
      """,
      Type::Warning => """
        <div style="border-left:4px solid #d69e2e;padding:10px;background:#fffff0">
        <h3 style="color:#d69e2e">⚠️ Warning</h3>
        <p>{{message}}</p>
        </div>
      """,
    }
  end
  
  private def start_processor
    spawn do
      while notification = @queue.receive?
        begin
          template = @templates[notification.type]
          html = template.gsub("{{message}}", notification.message)
          
          @smtp.send_email(
            from: @from,
            to: [notification.recipient],
            subject: notification.subject,
            body: notification.message,
            html_body: html
          )
          
          puts "Sent #{notification.type} to #{notification.recipient}"
        rescue ex
          puts "Failed to send notification: #{ex.message}"
        end
      end
    end
  end
end
```

### แบบฝึกหัดที่ 2: Email Digest

```crystal
# รวบรวมอีเมลและส่งเป็น digest รายวัน
class EmailDigest
  struct DigestItem
    property title : String
    property content : String
    property url : String?
    property added_at : Time
    
    def initialize(@title, @content, @url = nil)
      @added_at = Time.local
    end
  end
  
  def initialize(@smtp : SMTPClient, @from : String)
    @items = [] of DigestItem
    @subscribers = Set(String).new
    @mutex = Mutex.new
  end
  
  def subscribe(email : String)
    @mutex.synchronize { @subscribers << email }
  end
  
  def add_item(title : String, content : String, url : String? = nil)
    @mutex.synchronize { @items << DigestItem.new(title, content, url) }
  end
  
  def send_digest(subject : String = "Daily Digest")
    items = @mutex.synchronize { @items.dup }
    subscribers = @mutex.synchronize { @subscribers.to_a }
    
    return if items.empty? || subscribers.empty?
    
    html = build_digest_html(items)
    text = build_digest_text(items)
    
    subscribers.each do |email|
      @smtp.send_email(
        from: @from,
        to: [email],
        subject: "#{subject} - #{Time.local.to_s("%d %b %Y")}",
        body: text,
        html_body: html
      )
    end
    
    # Clear sent items
    @mutex.synchronize { @items.clear }
    
    puts "Sent digest to #{subscribers.size} subscribers (#{items.size} items)"
  end
  
  private def build_digest_html(items : Array(DigestItem)) : String
    items_html = items.map do |item|
      url_html = item.url ? "<a href=\"#{item.url}\">อ่านเพิ่มเติม</a>" : ""
      "<li><strong>#{item.title}</strong><br>#{item.content}<br>#{url_html}</li>"
    end.join("\n")
    
    """
    <html>
    <body>
      <h2>Daily Digest - #{Time.local.to_s("%d %b %Y")}</h2>
      <ul>#{items_html}</ul>
    </body>
    </html>
    """
  end
  
  private def build_digest_text(items : Array(DigestItem)) : String
    items.map { |i| "• #{i.title}\n  #{i.content}\n  #{i.url || ""}" }.join("\n\n")
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **SMTP Protocol**: การสื่อสารผ่าน SMTP (EHLO, AUTH, MAIL FROM, RCPT TO, DATA)
2. **STARTTLS**: การ upgrade จาก plain text เป็น TLS
3. **Authentication**: SMTP AUTH LOGIN
4. **HTML Emails**: ส่งอีเมล HTML พร้อม text fallback
5. **Attachments**: แนบไฟล์ด้วย Base64 encoding
6. **Email Builder**: สร้าง MIME message
7. **Email Templates**: template engine สำหรับ email
8. **Email Queue**: ส่งอีเมลแบบ async พร้อม retry
9. **Notification System**: ระบบแจ้งเตือนผ่านอีเมล
10. **Email Digest**: รวบรวมและส่ง digest

ข้อแนะนำ:
- ใช้ App Password แทน password จริงสำหรับ Gmail
- ใช้ environment variables สำหรับ credentials
- ทดสอบด้วย Mailhog หรือ Mailtrap ใน development
- ส่งอีเมลแบบ async เสมอ ไม่ควร block request
