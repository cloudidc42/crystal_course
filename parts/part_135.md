# Part 135: Kemal Static Files - การจัดการไฟล์ Static ใน Kemal

## บทนำ

Static files คือ HTML, CSS, JavaScript, รูปภาพ และไฟล์อื่นๆ ที่ serve ตรงๆ โดยไม่ผ่าน Crystal code Kemal มี built-in support สำหรับ static file serving

## Public Directory

```
my_app/
├── public/          <- static files directory
│   ├── css/
│   │   └── style.css
│   ├── js/
│   │   └── app.js
│   ├── images/
│   │   └── logo.png
│   └── index.html   <- served at /index.html
├── src/
│   └── app.cr
└── shard.yml
```

## การ Serve Static Files

```crystal
require "kemal"

# Kemal serve static files จาก public/ directory โดย default
# ไม่ต้องตั้งค่าเพิ่มเติม

get "/" do
  render "src/views/index.ecr"
end

# รัน server - public/ ถูก serve โดยอัตโนมัติ
Kemal.run
```

## ปิด/เปิด Static File Serving

```crystal
require "kemal"

# เปิด static file serving (default = เปิด)
serve_static true

# ปิด static file serving
serve_static false

# ตั้งค่า directory
serve_static({"dir" => "public"})

# ตั้งค่า headers สำหรับ static files
serve_static({
  "dir" => "public",
  "gzip" => true,      # enable gzip compression
  "headers" => {
    "Cache-Control" => "public, max-age=31536000"
  }
})

Kemal.run
```

## Custom Static Handler

```crystal
require "kemal"
require "mime"

class StaticFileHandler < Kemal::Handler
  def initialize(
    @public_dir : String = "public",
    @cache_max_age : Int32 = 3600,
    @enable_gzip : Bool = false
  )
  end
  
  def call(context : HTTP::Server::Context)
    path = context.request.path
    
    # ป้องกัน path traversal
    safe_path = File.expand_path(path, "/")
    file_path = File.join(@public_dir, safe_path)
    
    unless File.exists?(file_path) && File.file?(file_path)
      call_next(context)
      return
    end
    
    # MIME type
    mime = MIME.from_filename(file_path) rescue "application/octet-stream"
    
    # Last-Modified header
    mtime = File.info(file_path).modification_time
    last_modified = mtime.to_rfc2822
    
    # Check If-Modified-Since
    if ims = context.request.headers["If-Modified-Since"]?
      client_time = Time.parse_rfc2822(ims) rescue nil
      
      if client_time && mtime <= client_time
        context.response.status_code = 304
        return
      end
    end
    
    # ETag
    etag = "\"#{mtime.to_unix}-#{File.size(file_path)}\""
    
    if context.request.headers["If-None-Match"]? == etag
      context.response.status_code = 304
      return
    end
    
    # Set headers
    context.response.content_type = mime
    context.response.headers["Cache-Control"] = "public, max-age=#{@cache_max_age}"
    context.response.headers["Last-Modified"] = last_modified
    context.response.headers["ETag"] = etag
    context.response.headers["Accept-Ranges"] = "bytes"
    
    file_size = File.size(file_path)
    
    # Handle Range requests (partial content)
    if range = context.request.headers["Range"]?
      serve_range(context, file_path, file_size, mime, range)
    else
      # Serve full file
      context.response.headers["Content-Length"] = file_size.to_s
      File.open(file_path, "rb") do |f|
        IO.copy(f, context.response.output)
      end
    end
  end
  
  private def serve_range(context, file_path, file_size, mime, range_header)
    # Parse Range: bytes=start-end
    if range_header =~ /bytes=(\d*)-(\d*)/
      start_byte = $1.empty? ? 0_i64 : $1.to_i64
      end_byte = $2.empty? ? file_size - 1 : $2.to_i64
      
      end_byte = [end_byte, file_size - 1].min
      
      if start_byte > end_byte
        context.response.status_code = 416  # Range Not Satisfiable
        context.response.headers["Content-Range"] = "bytes */#{file_size}"
        return
      end
      
      content_length = end_byte - start_byte + 1
      
      context.response.status_code = 206  # Partial Content
      context.response.headers["Content-Range"] = "bytes #{start_byte}-#{end_byte}/#{file_size}"
      context.response.headers["Content-Length"] = content_length.to_s
      
      File.open(file_path, "rb") do |f|
        f.seek(start_byte)
        bytes_remaining = content_length
        buf = Bytes.new(8192)
        
        while bytes_remaining > 0
          to_read = [bytes_remaining, 8192_i64].min.to_i
          n = f.read(buf[0, to_read])
          break if n == 0
          context.response.output.write(buf[0, n])
          bytes_remaining -= n
        end
      end
    else
      context.response.status_code = 400
    end
  end
end
```

## Caching Headers

```crystal
require "kemal"

# เพิ่ม Cache headers สำหรับ static files
before_all do |env|
  path = env.request.path
  
  # กำหนด cache policy ตามประเภทไฟล์
  cache_policy = case path
  when /\.(css|js)$/
    # CSS/JS: cache 1 วัน
    "public, max-age=86400"
  when /\.(png|jpg|jpeg|gif|svg|ico|webp)$/
    # รูปภาพ: cache 30 วัน
    "public, max-age=2592000"
  when /\.(woff|woff2|ttf|eot)$/
    # Fonts: cache 1 ปี
    "public, max-age=31536000, immutable"
  when /\.html$/
    # HTML: no cache
    "no-cache, must-revalidate"
  else
    "no-store"
  end
  
  env.response.headers["Cache-Control"] = cache_policy
end

Kemal.run
```

## Content Versioning (Cache Busting)

```crystal
require "kemal"
require "digest"

class AssetHelper
  @@manifest = {} of String => String
  
  def self.load_manifest(manifest_path : String)
    if File.exists?(manifest_path)
      data = JSON.parse(File.read(manifest_path))
      data.as_h.each do |k, v|
        @@manifest[k.as_s] = v.as_s
      end
    end
  end
  
  def self.asset_url(path : String) : String
    # วิธีที่ 1: ใช้ manifest file (webpack/vite style)
    if versioned = @@manifest[path]?
      return "/#{versioned}"
    end
    
    # วิธีที่ 2: เพิ่ม ?v=hash ท้าย URL
    full_path = "public#{path}"
    if File.exists?(full_path)
      hash = Digest::MD5.hexdigest(File.read(full_path))[0..7]
      "#{path}?v=#{hash}"
    else
      path
    end
  end
  
  def self.fingerprint(path : String) : String
    # วิธีที่ 3: fingerprint ในชื่อไฟล์
    full_path = "public#{path}"
    return path unless File.exists?(full_path)
    
    content = File.read(full_path)
    hash = Digest::MD5.hexdigest(content)[0..7]
    ext = File.extname(path)
    base = path.chomp(ext)
    "#{base}-#{hash}#{ext}"
  end
end

# ใช้งานใน templates:
# <%= AssetHelper.asset_url("/css/style.css") %>
# => /css/style.css?v=abc12345
```

## Directory Listing

```crystal
require "kemal"

# สร้าง directory listing handler (สำหรับ development)
class DirectoryListingHandler < Kemal::Handler
  def initialize(@root : String = "public", @enabled : Bool = false)
  end
  
  def call(context : HTTP::Server::Context)
    return call_next(context) unless @enabled
    
    path = context.request.path
    dir_path = File.join(@root, path)
    
    unless File.directory?(dir_path)
      call_next(context)
      return
    end
    
    entries = Dir.entries(dir_path).sort.reject { |e| e.starts_with?(".") }
    
    html = String.build do |s|
      s << "<!DOCTYPE html><html><head>"
      s << "<title>Directory: #{path}</title>"
      s << "<style>body{font-family:monospace;padding:1rem} a{display:block;padding:0.25rem} .dir{color:#2563eb} .file{color:#374151}</style>"
      s << "</head><body>"
      s << "<h2>#{path}</h2><hr>"
      
      if path != "/"
        parent = File.dirname(path)
        s << "<a href='#{parent}' class='dir'>.. (parent)</a>"
      end
      
      entries.each do |entry|
        full = File.join(dir_path, entry)
        if File.directory?(full)
          s << "<a href='#{File.join(path, entry)}/' class='dir'>📁 #{entry}/</a>"
        else
          size = File.size(full)
          s << "<a href='#{File.join(path, entry)}' class='file'>📄 #{entry} (#{format_size(size)})</a>"
        end
      end
      
      s << "</body></html>"
    end
    
    context.response.content_type = "text/html"
    context.response.print(html)
  end
  
  private def format_size(bytes : Int64 | UInt64) : String
    case bytes
    when .< 1024          then "#{bytes} B"
    when .< 1024 * 1024   then "#{(bytes / 1024.0).round(1)} KB"
    when .< 1024**3        then "#{(bytes / (1024.0**2)).round(1)} MB"
    else                       "#{(bytes / (1024.0**3)).round(1)} GB"
    end
  end
end

# เฉพาะ development เท่านั้น
if Kemal.config.env == "development"
  add_handler DirectoryListingHandler.new(enabled: true)
end
```

## File Download

```crystal
require "kemal"

# File download endpoint
get "/download/:filename" do |env|
  filename = env.params.url["filename"]
  
  # ป้องกัน path traversal
  unless filename.matches?(/\A[\w\-. ]+\z/)
    halt env, status_code: 400, response: "Invalid filename"
  end
  
  file_path = "downloads/#{filename}"
  
  unless File.exists?(file_path)
    halt env, status_code: 404, response: "File not found"
  end
  
  # ตั้งค่า headers สำหรับ download
  env.response.headers["Content-Disposition"] = "attachment; filename=\"#{filename}\""
  env.response.headers["Content-Type"] = "application/octet-stream"
  env.response.headers["Content-Length"] = File.size(file_path).to_s
  
  File.open(file_path, "rb") do |f|
    IO.copy(f, env.response.output)
  end
end

# Inline file view (PDF, images)
get "/view/:filename" do |env|
  filename = env.params.url["filename"]
  file_path = "files/#{filename}"
  
  unless File.exists?(file_path)
    halt env, status_code: 404, response: "Not found"
  end
  
  mime = MIME.from_filename(file_path) rescue "application/octet-stream"
  
  env.response.content_type = mime
  env.response.headers["Content-Disposition"] = "inline; filename=\"#{filename}\""
  
  File.open(file_path, "rb") do |f|
    IO.copy(f, env.response.output)
  end
end
```

## File Upload

```crystal
require "kemal"

UPLOAD_DIR = "uploads"

# สร้าง upload directory
Dir.mkdir_p(UPLOAD_DIR)

# File upload endpoint
post "/upload" do |env|
  unless env.params.files.has_key?("file")
    halt env, status_code: 400, response: {"error" => "No file provided"}.to_json
  end
  
  file = env.params.files["file"]
  filename = file.filename || "unknown"
  
  # ตรวจสอบ filename
  safe_filename = File.basename(filename).gsub(/[^a-zA-Z0-9._-]/, "_")
  
  # ตรวจสอบขนาด (max 10MB)
  max_size = 10 * 1024 * 1024
  content = file.tmpfile.gets_to_end
  
  if content.bytesize > max_size
    halt env, status_code: 413, response: {"error" => "File too large (max 10MB)"}.to_json
  end
  
  # ตรวจสอบประเภทไฟล์
  allowed_types = [".jpg", ".jpeg", ".png", ".gif", ".pdf", ".txt", ".csv"]
  ext = File.extname(safe_filename).downcase
  
  unless allowed_types.includes?(ext)
    halt env, status_code: 415, response: {"error" => "File type not allowed"}.to_json
  end
  
  # สร้างชื่อไฟล์ unique
  timestamp = Time.local.to_unix.to_s
  unique_name = "#{timestamp}_#{safe_filename}"
  dest_path = File.join(UPLOAD_DIR, unique_name)
  
  File.write(dest_path, content)
  
  env.response.content_type = "application/json"
  env.response.status_code = 201
  {
    "success" => true,
    "filename" => unique_name,
    "size" => content.bytesize,
    "url" => "/files/#{unique_name}",
  }.to_json
end

# Serve uploaded files
get "/files/:filename" do |env|
  filename = env.params.url["filename"]
  
  unless filename.matches?(/\A[\w\-. ]+\z/)
    halt env, status_code: 400, response: "Invalid filename"
  end
  
  file_path = File.join(UPLOAD_DIR, filename)
  
  unless File.exists?(file_path)
    halt env, status_code: 404, response: "Not found"
  end
  
  mime = MIME.from_filename(file_path) rescue "application/octet-stream"
  env.response.content_type = mime
  
  File.open(file_path, "rb") do |f|
    IO.copy(f, env.response.output)
  end
end
```

## SPA (Single Page Application) Support

```crystal
require "kemal"

# Serve SPA: ส่ง index.html สำหรับทุก route ที่ไม่ match
# (React, Vue, Angular routing)

get "/*" do |env|
  path = env.params.url["glob"]? || ""
  
  # ถ้าเป็น API path, ข้าม
  if path.starts_with?("api/")
    halt env, status_code: 404, response: {"error" => "Not found"}.to_json
  end
  
  # ถ้าเป็นไฟล์ static ที่มีอยู่จริง, ส่งตรงๆ
  file_path = "public/#{path}"
  if File.exists?(file_path) && File.file?(file_path)
    mime = MIME.from_filename(file_path) rescue "text/plain"
    env.response.content_type = mime
    File.open(file_path, "rb") { |f| IO.copy(f, env.response.output) }
  else
    # ส่ง index.html สำหรับ SPA routing
    env.response.content_type = "text/html"
    env.response.headers["Cache-Control"] = "no-cache"
    File.read("public/index.html")
  end
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Static File Server with Security

```crystal
require "kemal"

class SecureStaticHandler < Kemal::Handler
  ALLOWED_EXTENSIONS = [".html", ".css", ".js", ".png", ".jpg", ".svg", ".ico", ".woff2"]
  
  def initialize(@root : String = "public")
  end
  
  def call(context : HTTP::Server::Context)
    path = context.request.path
    
    # ป้องกัน path traversal
    normalized = File.expand_path(path, "/")
    file_path = "#{@root}#{normalized}"
    
    # ตรวจสอบ extension
    ext = File.extname(file_path).downcase
    unless ALLOWED_EXTENSIONS.includes?(ext)
      call_next(context)
      return
    end
    
    unless File.exists?(file_path) && File.file?(file_path)
      call_next(context)
      return
    end
    
    # Security headers
    context.response.headers["X-Content-Type-Options"] = "nosniff"
    context.response.headers["X-Frame-Options"] = "SAMEORIGIN"
    
    mime = MIME.from_filename(file_path) rescue "application/octet-stream"
    context.response.content_type = mime
    
    File.open(file_path, "rb") do |f|
      IO.copy(f, context.response.output)
    end
  end
end

add_handler SecureStaticHandler.new

get "/api/health" do
  {"status" => "ok"}.to_json
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Public Directory**: structure และการจัดวางไฟล์
2. **serve_static**: เปิด/ปิด static file serving
3. **Custom Static Handler**: handler ที่ปรับแต่งได้
4. **Caching Headers**: Cache-Control, ETag, Last-Modified
5. **Range Requests**: partial content สำหรับ video/audio
6. **Cache Busting**: versioning ด้วย hash
7. **Directory Listing**: สำหรับ development
8. **File Download**: Content-Disposition header
9. **File Upload**: multipart form data
10. **SPA Support**: serve index.html สำหรับ client-side routing
11. **Security**: ป้องกัน path traversal
