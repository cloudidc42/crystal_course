# Part 121: HTTP Client - การใช้งาน HTTP Client ใน Crystal

## บทนำ

Crystal มี HTTP client ในตัว (`http/client`) ที่ใช้งานง่ายและมีประสิทธิภาพสูง รองรับ HTTP/1.1, HTTPS, headers, body, redirects, authentication และอื่นๆ

## การเริ่มต้นใช้งาน

```crystal
require "http/client"

# GET request แบบง่าย
response = HTTP::Client.get("https://httpbin.org/get")
puts response.status_code      # 200
puts response.status.ok?       # true
puts response.body             # JSON body
```

## HTTP Methods

### GET

```crystal
require "http/client"
require "json"

# GET พื้นฐาน
response = HTTP::Client.get("https://httpbin.org/get")
puts "Status: #{response.status_code}"
puts "Content-Type: #{response.headers["Content-Type"]?}"

# Parse JSON response
if response.status.ok?
  data = JSON.parse(response.body)
  puts "Origin: #{data["origin"]}"
end
```

### POST

```crystal
require "http/client"
require "json"

# POST ด้วย JSON body
body = {"name" => "Crystal", "version" => "1.0"}.to_json
headers = HTTP::Headers{"Content-Type" => "application/json"}

response = HTTP::Client.post(
  "https://httpbin.org/post",
  headers: headers,
  body: body
)

puts response.status_code
data = JSON.parse(response.body)
puts data["json"].inspect # JSON body ที่ส่งไป
```

### PUT และ PATCH

```crystal
# PUT
response = HTTP::Client.put(
  "https://httpbin.org/put",
  headers: HTTP::Headers{"Content-Type" => "application/json"},
  body: {"id" => 1, "name" => "Updated"}.to_json
)
puts "PUT Status: #{response.status_code}"

# PATCH
response = HTTP::Client.patch(
  "https://httpbin.org/patch",
  headers: HTTP::Headers{"Content-Type" => "application/json"},
  body: {"name" => "Partially Updated"}.to_json
)
puts "PATCH Status: #{response.status_code}"
```

### DELETE

```crystal
# DELETE
response = HTTP::Client.delete("https://httpbin.org/delete")
puts "DELETE Status: #{response.status_code}"

# DELETE with body
response = HTTP::Client.delete(
  "https://httpbin.org/delete",
  headers: HTTP::Headers{"Content-Type" => "application/json"},
  body: {"id" => 123}.to_json
)
puts response.status_code
```

## Headers

```crystal
require "http/client"

# กำหนด Headers
headers = HTTP::Headers{
  "Authorization" => "Bearer mytoken123",
  "Content-Type" => "application/json",
  "Accept" => "application/json",
  "User-Agent" => "MyCrystalApp/1.0",
  "X-Request-ID" => "req-#{Random::Secure.hex(8)}",
}

response = HTTP::Client.get("https://httpbin.org/headers", headers: headers)
puts JSON.parse(response.body)["headers"].inspect

# เพิ่ม headers ทีละอัน
headers = HTTP::Headers.new
headers.add("Accept", "text/html")
headers.add("Accept", "application/json") # เพิ่ม value ให้ key เดิม
headers["X-Custom"] = "value"

puts headers["Accept"] # "text/html,application/json"
puts headers["X-Custom"]? # "value"
```

## Request Body

```crystal
require "http/client"
require "json"

# 1. String body
HTTP::Client.post("https://httpbin.org/post", body: "raw string body")

# 2. JSON body
HTTP::Client.post(
  "https://httpbin.org/post",
  headers: HTTP::Headers{"Content-Type" => "application/json"},
  body: {name: "test", value: 42}.to_json
)

# 3. Form-encoded body
form_data = HTTP::Params.encode({"username" => "user1", "password" => "pass123"})
HTTP::Client.post(
  "https://httpbin.org/post",
  headers: HTTP::Headers{"Content-Type" => "application/x-www-form-urlencoded"},
  body: form_data
)

# 4. Multipart form data
body_io = IO::Memory.new
multipart_builder = HTTP::FormData::Builder.new(body_io)
multipart_builder.field("name", "Crystal")
multipart_builder.file("file", IO::Memory.new("file content"), HTTP::FormData::FileMetadata.new(filename: "test.txt"))
multipart_builder.finish

HTTP::Client.post(
  "https://httpbin.org/post",
  headers: HTTP::Headers{"Content-Type" => multipart_builder.content_type},
  body: body_io.to_s
)
```

## Response

```crystal
require "http/client"

response = HTTP::Client.get("https://httpbin.org/get")

# Status
puts response.status_code         # 200
puts response.status              # HTTP::Status::OK
puts response.status.ok?          # true
puts response.status.not_found?   # false
puts response.status.description  # "OK"

# Headers
puts response.headers["Content-Type"]?
puts response.headers.to_a.inspect

# Body
puts response.body          # String
puts response.body_io       # IO
puts response.content_type  # "application/json"

# ตรวจสอบ status
case response.status_code
when 200..299
  puts "สำเร็จ: #{response.body}"
when 400..499
  puts "Client Error: #{response.status_code}"
when 500..599
  puts "Server Error: #{response.status_code}"
end
```

## Query Parameters

```crystal
require "http/client"

# วิธีที่ 1: ใส่ใน URL ตรงๆ
response = HTTP::Client.get("https://httpbin.org/get?key1=value1&key2=value2")

# วิธีที่ 2: ใช้ URI
require "uri"
uri = URI.new("https", "httpbin.org", path: "/get")
uri.query = HTTP::Params.encode({"name" => "Crystal", "version" => "1.0", "page" => "1"})
response = HTTP::Client.get(uri)
puts response.body

# วิธีที่ 3: Build URL
params = HTTP::Params.new
params.add("search", "crystal lang")
params.add("page", "1")
params.add("per_page", "20")
url = "https://httpbin.org/get?#{params.to_s}"
response = HTTP::Client.get(url)
```

## Redirects

```crystal
require "http/client"

# ติดตาม redirect อัตโนมัติ
response = HTTP::Client.get("https://httpbin.org/redirect/3")
puts "Final status: #{response.status_code}"
# Crystal ไม่ติดตาม redirect อัตโนมัติ ต้องทำเอง

# ติดตาม redirect ด้วยตัวเอง
def get_with_redirect(url : String, max_redirects : Int32 = 5) : HTTP::Client::Response
  redirects = 0
  current_url = url
  
  loop do
    response = HTTP::Client.get(current_url)
    
    if response.status.redirection?
      raise "Too many redirects" if redirects >= max_redirects
      
      location = response.headers["Location"]?
      raise "No Location header in redirect" unless location
      
      redirects += 1
      current_url = location
      puts "Redirect #{redirects}: #{current_url}"
    else
      return response
    end
  end
end

response = get_with_redirect("https://httpbin.org/redirect/3")
puts "Final: #{response.status_code}"
```

## Timeouts

```crystal
require "http/client"

# กำหนด timeout
client = HTTP::Client.new("httpbin.org", tls: true)
client.connect_timeout = 5.seconds
client.read_timeout = 10.seconds
client.write_timeout = 5.seconds

begin
  response = client.get("/get")
  puts response.status_code
rescue IO::TimeoutError
  puts "Connection timed out!"
ensure
  client.close
end

# หรือใช้ block form
HTTP::Client.new("httpbin.org", tls: true) do |client|
  client.connect_timeout = 5.seconds
  client.read_timeout = 10.seconds
  
  response = client.get("/get")
  puts response.status_code
end
```

## Basic Authentication

```crystal
require "http/client"

# Basic Auth
client = HTTP::Client.new("httpbin.org", tls: true)
client.basic_auth("user", "password")

response = client.get("/basic-auth/user/password")
puts response.status_code # 200 ถ้า credentials ถูกต้อง

# หรือใช้ headers
import_secret = Base64.strict_encode("username:password")
headers = HTTP::Headers{"Authorization" => "Basic #{import_secret}"}
response = HTTP::Client.get("https://httpbin.org/basic-auth/username/password", headers: headers)
puts response.status_code
```

## Bearer Token Authentication

```crystal
require "http/client"

# Bearer Token
def authenticated_request(url : String, token : String) : HTTP::Client::Response
  headers = HTTP::Headers{"Authorization" => "Bearer #{token}"}
  HTTP::Client.get(url, headers: headers)
end

response = authenticated_request("https://api.example.com/data", "mytoken123")
```

## การใช้งานกับ JSON API

```crystal
require "http/client"
require "json"

# JSON API Client
module JSONApi
  BASE_URL = "https://jsonplaceholder.typicode.com"
  
  def self.get_post(id : Int32) : JSON::Any
    response = HTTP::Client.get("#{BASE_URL}/posts/#{id}")
    raise "HTTP Error: #{response.status_code}" unless response.status.ok?
    JSON.parse(response.body)
  end
  
  def self.create_post(title : String, body : String, user_id : Int32) : JSON::Any
    data = {title: title, body: body, userId: user_id}.to_json
    headers = HTTP::Headers{"Content-Type" => "application/json"}
    
    response = HTTP::Client.post("#{BASE_URL}/posts", headers: headers, body: data)
    raise "HTTP Error: #{response.status_code}" unless response.status.created?
    JSON.parse(response.body)
  end
  
  def self.get_comments(post_id : Int32) : Array(JSON::Any)
    response = HTTP::Client.get("#{BASE_URL}/posts/#{post_id}/comments")
    raise "HTTP Error: #{response.status_code}" unless response.status.ok?
    JSON.parse(response.body).as_a
  end
end

# ใช้งาน
post = JSONApi.get_post(1)
puts "Title: #{post["title"]}"
puts "Body: #{post["body"]}"

new_post = JSONApi.create_post("ทดสอบ Crystal", "Crystal เป็นภาษาที่เร็วมาก", 1)
puts "Created post ID: #{new_post["id"]}"

comments = JSONApi.get_comments(1)
puts "Comments: #{comments.size}"
comments.first(2).each { |c| puts "  - #{c["name"]}: #{c["body"][0..50]}..." }
```

## HTTP Client Pool

```crystal
require "http/client"

# Connection pool สำหรับ reuse connections
class HTTPClientPool
  def initialize(@host : String, @port : Int32 = 443, @tls : Bool = true, @pool_size : Int32 = 5)
    @clients = Channel(HTTP::Client).new(@pool_size)
    @pool_size.times { @clients.send(create_client) }
  end
  
  def get(path : String, headers : HTTP::Headers? = nil) : HTTP::Client::Response
    with_client { |c| c.get(path, headers: headers) }
  end
  
  def post(path : String, body : String? = nil, headers : HTTP::Headers? = nil) : HTTP::Client::Response
    with_client { |c| c.post(path, body: body, headers: headers) }
  end
  
  private def with_client(&block : HTTP::Client -> HTTP::Client::Response) : HTTP::Client::Response
    client = @clients.receive
    begin
      block.call(client)
    ensure
      @clients.send(client)
    end
  end
  
  private def create_client : HTTP::Client
    client = HTTP::Client.new(@host, @port, tls: @tls)
    client.connect_timeout = 5.seconds
    client.read_timeout = 10.seconds
    client
  end
  
  def close
    @pool_size.times do
      @clients.receive.close
    end
    @clients.close
  end
end

# ใช้งาน pool
pool = HTTPClientPool.new("httpbin.org")

10.times do |i|
  spawn do
    response = pool.get("/get")
    puts "Request #{i + 1}: #{response.status_code}"
  end
end

sleep 5.seconds
pool.close
```

## Streaming Response

```crystal
require "http/client"

# อ่าน streaming response
HTTP::Client.get("https://httpbin.org/stream/5") do |response|
  puts "Status: #{response.status_code}"
  
  while line = response.body_io.gets
    puts "Received: #{line[0..50]}..."
  end
end
```

## Error Handling

```crystal
require "http/client"
require "socket"

def safe_http_get(url : String) : String?
  begin
    response = HTTP::Client.get(url)
    
    case response.status_code
    when 200..299
      response.body
    when 301, 302, 303, 307, 308
      puts "Redirect to: #{response.headers["Location"]?}"
      nil
    when 401
      puts "Unauthorized"
      nil
    when 403
      puts "Forbidden"
      nil
    when 404
      puts "Not Found"
      nil
    when 429
      retry_after = response.headers["Retry-After"]?.try(&.to_i) || 60
      puts "Rate limited. Retry after #{retry_after}s"
      nil
    when 500..599
      puts "Server error: #{response.status_code}"
      nil
    else
      puts "Unexpected status: #{response.status_code}"
      nil
    end
  rescue ex : IO::TimeoutError
    puts "Timeout: #{ex.message}"
    nil
  rescue ex : Socket::Error
    puts "Connection error: #{ex.message}"
    nil
  rescue ex : OpenSSL::SSL::Error
    puts "SSL error: #{ex.message}"
    nil
  rescue ex
    puts "Error: #{ex.message}"
    nil
  end
end

result = safe_http_get("https://httpbin.org/get")
puts result[0..100] if result
```

## Retry Mechanism

```crystal
require "http/client"

def http_get_with_retry(
  url : String,
  max_retries : Int32 = 3,
  initial_delay : Float64 = 1.0
) : HTTP::Client::Response
  retries = 0
  delay = initial_delay
  
  loop do
    begin
      response = HTTP::Client.get(url)
      
      # Retry on server errors
      if response.status_code >= 500 && retries < max_retries
        retries += 1
        puts "Server error #{response.status_code}. Retry #{retries}/#{max_retries} in #{delay}s..."
        sleep delay.seconds
        delay *= 2 # exponential backoff
        next
      end
      
      return response
    rescue ex : IO::TimeoutError | Socket::Error
      raise ex if retries >= max_retries
      
      retries += 1
      puts "Network error. Retry #{retries}/#{max_retries} in #{delay}s..."
      sleep delay.seconds
      delay *= 2
    end
  end
end

begin
  response = http_get_with_retry("https://httpbin.org/status/500", max_retries: 2)
  puts "Final status: #{response.status_code}"
rescue ex
  puts "Failed after retries: #{ex.message}"
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: สร้าง REST API Client

```crystal
require "http/client"
require "json"

class GitHubClient
  BASE_URL = "https://api.github.com"
  
  def initialize(token : String? = nil)
    @headers = HTTP::Headers{
      "Accept" => "application/vnd.github.v3+json",
      "User-Agent" => "Crystal-GitHub-Client",
    }
    @headers["Authorization"] = "token #{token}" if token
  end
  
  def get_user(username : String) : JSON::Any
    get("/users/#{username}")
  end
  
  def get_repos(username : String, page : Int32 = 1, per_page : Int32 = 30) : Array(JSON::Any)
    params = HTTP::Params.encode({"page" => page.to_s, "per_page" => per_page.to_s})
    result = get("/users/#{username}/repos?#{params}")
    result.as_a
  end
  
  def get_repo(owner : String, repo : String) : JSON::Any
    get("/repos/#{owner}/#{repo}")
  end
  
  def search_repos(query : String, language : String? = nil) : JSON::Any
    q = language ? "#{query} language:#{language}" : query
    params = HTTP::Params.encode({"q" => q})
    get("/search/repositories?#{params}")
  end
  
  private def get(path : String) : JSON::Any
    response = HTTP::Client.get("#{BASE_URL}#{path}", headers: @headers)
    
    case response.status_code
    when 200..299
      JSON.parse(response.body)
    when 404
      raise "Not found: #{path}"
    when 403
      raise "Forbidden (Rate limit?)"
    when 401
      raise "Unauthorized"
    else
      raise "HTTP Error: #{response.status_code}"
    end
  end
end

# ใช้งาน
client = GitHubClient.new

begin
  user = client.get_user("crystal-lang")
  puts "Organization: #{user["name"]}"
  puts "Public repos: #{user["public_repos"]}"
  
  repos = client.get_repos("crystal-lang", per_page: 5)
  puts "\nTop 5 repos:"
  repos.each { |r| puts "  - #{r["name"]}: #{r["description"]}" }
rescue ex
  puts "Error: #{ex.message}"
end
```

### แบบฝึกหัดที่ 2: HTTP Benchmark

```crystal
require "http/client"

def benchmark_http(url : String, n : Int32 = 10) : Hash(String, Float64)
  times = [] of Float64
  errors = 0
  
  n.times do
    start = Time.monotonic
    begin
      HTTP::Client.get(url)
      elapsed = (Time.monotonic - start).total_milliseconds
      times << elapsed
    rescue
      errors += 1
    end
  end
  
  return {"error_rate" => 1.0} if times.empty?
  
  sorted = times.sort
  {
    "min" => sorted.first,
    "max" => sorted.last,
    "mean" => times.sum / times.size,
    "p50" => sorted[times.size // 2],
    "p95" => sorted[(times.size * 0.95).to_i],
    "p99" => sorted[(times.size * 0.99).to_i],
    "errors" => errors.to_f,
    "error_rate" => (errors.to_f / n * 100),
  }
end

url = "https://httpbin.org/get"
puts "Benchmarking #{url}..."
stats = benchmark_http(url, 10)

puts "Min: #{stats["min"].round(1)}ms"
puts "Max: #{stats["max"].round(1)}ms"
puts "Mean: #{stats["mean"].round(1)}ms"
puts "P50: #{stats["p50"].round(1)}ms"
puts "P95: #{stats["p95"].round(1)}ms"
puts "Error rate: #{stats["error_rate"].round(1)}%"
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HTTP Methods**: GET, POST, PUT, PATCH, DELETE
2. **Headers**: กำหนด request headers, อ่าน response headers
3. **Request Body**: JSON, form-encoded, multipart
4. **Response**: status code, headers, body
5. **Query Parameters**: encoding URL parameters
6. **Redirects**: ติดตาม redirects ด้วยตัวเอง
7. **Timeouts**: connect/read/write timeouts
8. **Authentication**: Basic Auth, Bearer Token
9. **Connection Pool**: reuse connections
10. **Streaming**: อ่าน streaming response
11. **Error Handling**: จัดการ error ต่างๆ
12. **Retry**: exponential backoff
