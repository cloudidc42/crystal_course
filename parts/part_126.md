# Part 126: DNS - การทำงานกับ DNS ใน Crystal

## บทนำ

DNS (Domain Name System) เป็นระบบที่แปลง domain names เป็น IP addresses Crystal มี `Socket::Addrinfo` สำหรับทำ DNS lookups รวมถึง reverse lookup และการจัดการ multiple addresses

## Socket::Addrinfo.resolve

```crystal
require "socket"

# DNS Lookup พื้นฐาน
addresses = Socket::Addrinfo.resolve("google.com", 80, type: Socket::Type::STREAM)

addresses.each do |addr|
  puts "Address: #{addr.ip_address}"
  puts "Family: #{addr.family}"
  puts "Type: #{addr.type}"
  puts "---"
end
```

## DNS Lookup

```crystal
require "socket"

# Lookup hostname
def dns_lookup(hostname : String, port : Int32 = 80) : Array(String)
  addresses = Socket::Addrinfo.resolve(hostname, port)
  addresses.map { |addr| addr.ip_address.to_s }
rescue Socket::Error => ex
  puts "DNS Error for #{hostname}: #{ex.message}"
  [] of String
end

# ทดสอบ
puts "=== DNS Lookup ==="
["google.com", "cloudflare.com", "github.com"].each do |host|
  ips = dns_lookup(host)
  puts "#{host}: #{ips.join(", ")}"
end

# แยก IPv4 และ IPv6
def lookup_with_family(hostname : String)
  ipv4 = Socket::Addrinfo.resolve(hostname, 80, family: Socket::Family::INET)
  ipv6 = Socket::Addrinfo.resolve(hostname, 80, family: Socket::Family::INET6)
  
  puts "#{hostname}:"
  puts "  IPv4: #{ipv4.map { |a| a.ip_address.to_s }.join(", ")}"
  puts "  IPv6: #{ipv6.map { |a| a.ip_address.to_s }.join(", ")}"
rescue Socket::Error => ex
  puts "#{hostname}: Error - #{ex.message}"
end

lookup_with_family("google.com")
```

## Reverse Lookup

```crystal
require "socket"

# Reverse DNS Lookup (IP -> hostname)
def reverse_lookup(ip : String) : String?
  begin
    # Crystal ไม่มี built-in reverse lookup
    # ต้องใช้ system call หรือ external tool
    result = `host #{ip} 2>&1`.strip
    
    if result.includes?("domain name pointer")
      # ดึง hostname จาก output
      result.split("domain name pointer").last.strip.chomp(".")
    else
      nil
    end
  rescue
    nil
  end
end

# หรือใช้ getaddrinfo กับ NI_NAMEREQD flag
def getnameinfo(ip : String) : String?
  # ใช้ LibC สำหรับ getnameinfo
  addr = Socket::IPAddress.new(ip, 0)
  # ใน Crystal ปัจจุบัน ต้องใช้ binding หรือ system command
  nil
end

# ทดสอบด้วย system command
puts "=== Reverse Lookup ==="
["8.8.8.8", "1.1.1.1", "208.80.154.224"].each do |ip|
  hostname = reverse_lookup(ip)
  puts "#{ip} => #{hostname || "ไม่พบ"}"
end
```

## Multiple Addresses

```crystal
require "socket"

# จัดการ multiple IP addresses
class DNSResult
  property hostname : String
  property addresses : Array(Socket::Addrinfo)
  property resolved_at : Time
  property ttl : Time::Span
  
  def initialize(@hostname, @addresses, @ttl = 300.seconds)
    @resolved_at = Time.local
  end
  
  def expired?
    Time.local - @resolved_at > @ttl
  end
  
  def ipv4_addresses : Array(String)
    @addresses
      .select { |a| a.family == Socket::Family::INET }
      .map { |a| a.ip_address.to_s }
  end
  
  def ipv6_addresses : Array(String)
    @addresses
      .select { |a| a.family == Socket::Family::INET6 }
      .map { |a| a.ip_address.to_s }
  end
  
  def first_ipv4 : String?
    ipv4_addresses.first?
  end
  
  def first_ipv6 : String?
    ipv6_addresses.first?
  end
  
  def best_address : String?
    # Prefer IPv6 if available
    first_ipv6 || first_ipv4
  end
end

# DNS Resolver พร้อม Cache
class DNSResolver
  def initialize(@ttl : Time::Span = 300.seconds)
    @cache = {} of String => DNSResult
    @mutex = Mutex.new
  end
  
  def resolve(hostname : String) : DNSResult?
    # เช็ค cache
    @mutex.synchronize do
      if cached = @cache[hostname]?
        return cached unless cached.expired?
        @cache.delete(hostname)
      end
    end
    
    # Resolve ใหม่
    begin
      addresses = Socket::Addrinfo.resolve(hostname, 80)
      result = DNSResult.new(hostname, addresses, @ttl)
      @mutex.synchronize { @cache[hostname] = result }
      result
    rescue Socket::Error => ex
      puts "DNS Error: #{ex.message}"
      nil
    end
  end
  
  def clear_cache
    @mutex.synchronize { @cache.clear }
  end
  
  def cache_size
    @mutex.synchronize { @cache.size }
  end
end

resolver = DNSResolver.new(ttl: 60.seconds)

puts "=== DNS Resolver with Cache ==="
["google.com", "cloudflare.com", "google.com"].each do |host|
  if result = resolver.resolve(host)
    puts "#{host}:"
    puts "  IPv4: #{result.ipv4_addresses.join(", ")}"
    puts "  Best: #{result.best_address}"
    puts "  Expired: #{result.expired?}"
  end
end

puts "Cache size: #{resolver.cache_size}"
```

## DNS Round Robin

```crystal
require "socket"

# DNS Round Robin Load Balancing
class RoundRobinDNS
  def initialize(@hostname : String, @port : Int32)
    @addresses = [] of Socket::IPAddress
    @current = Atomic(Int32).new(0)
    @last_refresh = Time.monotonic
    @refresh_interval = 60.seconds
    @mutex = Mutex.new
    refresh
  end
  
  def next_address : Socket::IPAddress?
    refresh_if_needed
    
    addresses = @mutex.synchronize { @addresses.dup }
    return nil if addresses.empty?
    
    idx = @current.add(1) % addresses.size
    addresses[idx]
  end
  
  def all_addresses : Array(Socket::IPAddress)
    refresh_if_needed
    @mutex.synchronize { @addresses.dup }
  end
  
  private def refresh
    begin
      new_addresses = Socket::Addrinfo.resolve(@hostname, @port)
        .select { |a| a.family == Socket::Family::INET }
        .map { |a| Socket::IPAddress.new(a.ip_address.to_s, @port) }
      
      @mutex.synchronize do
        @addresses = new_addresses
        @last_refresh = Time.monotonic
      end
      
      puts "Refreshed DNS for #{@hostname}: #{new_addresses.size} addresses"
    rescue Socket::Error => ex
      puts "DNS refresh failed: #{ex.message}"
    end
  end
  
  private def refresh_if_needed
    if Time.monotonic - @last_refresh > @refresh_interval
      refresh
    end
  end
end

dns = RoundRobinDNS.new("google.com", 443)

puts "=== Round Robin DNS ==="
10.times do |i|
  if addr = dns.next_address
    puts "Request #{i + 1}: #{addr}"
  end
end
```

## DNS-over-HTTPS (DoH)

```crystal
require "http/client"
require "json"

# DNS-over-HTTPS (Cloudflare)
class DoHResolver
  DOH_ENDPOINT = "https://cloudflare-dns.com/dns-query"
  
  def resolve(hostname : String, record_type : String = "A") : Array(String)
    headers = HTTP::Headers{
      "Accept" => "application/dns-json",
    }
    
    url = "#{DOH_ENDPOINT}?name=#{URI.encode_path(hostname)}&type=#{record_type}"
    
    response = HTTP::Client.get(url, headers: headers)
    
    unless response.status.ok?
      raise "DoH request failed: #{response.status_code}"
    end
    
    data = JSON.parse(response.body)
    answers = data["Answer"]?.try(&.as_a?) || [] of JSON::Any
    
    answers
      .select { |r| r["type"]? == JSON::Any.new(record_type == "A" ? 1 : 28) }
      .map { |r| r["data"]?.try(&.as_s?) || "" }
      .reject(&.empty?)
  rescue ex
    puts "DoH error for #{hostname}: #{ex.message}"
    [] of String
  end
  
  def resolve_mx(hostname : String) : Array(Hash(String, String))
    headers = HTTP::Headers{"Accept" => "application/dns-json"}
    url = "#{DOH_ENDPOINT}?name=#{URI.encode_path(hostname)}&type=MX"
    
    response = HTTP::Client.get(url, headers: headers)
    data = JSON.parse(response.body)
    
    answers = data["Answer"]?.try(&.as_a?) || [] of JSON::Any
    answers.map do |r|
      data_parts = r["data"]?.try(&.as_s?)&.split(" ") || ["0", ""]
      {
        "priority" => data_parts[0],
        "exchange" => data_parts[1]? || "",
      }
    end
  end
end

doh = DoHResolver.new

puts "=== DNS-over-HTTPS ==="
puts "google.com A records: #{doh.resolve("google.com").join(", ")}"
puts "google.com AAAA records: #{doh.resolve("google.com", "AAAA").join(", ")}"
puts "google.com MX records: #{doh.resolve_mx("google.com").inspect}"
```

## Health Check ด้วย DNS

```crystal
require "socket"

class ServiceDiscovery
  struct ServiceEndpoint
    property host : String
    property port : Int32
    property healthy : Bool
    property last_check : Time
    
    def initialize(@host, @port, @healthy = true)
      @last_check = Time.local
    end
  end
  
  def initialize
    @endpoints = [] of ServiceEndpoint
    @mutex = Mutex.new
  end
  
  def discover(service_name : String, port : Int32)
    begin
      addresses = Socket::Addrinfo.resolve(service_name, port)
      
      endpoints = addresses.map do |addr|
        ServiceEndpoint.new(addr.ip_address.to_s, port)
      end
      
      @mutex.synchronize { @endpoints = endpoints }
      puts "Discovered #{endpoints.size} endpoints for #{service_name}"
    rescue Socket::Error => ex
      puts "Discovery failed: #{ex.message}"
    end
  end
  
  def start_health_checks(interval : Time::Span = 10.seconds)
    spawn do
      loop do
        sleep interval
        check_health
      end
    end
  end
  
  def healthy_endpoints : Array(ServiceEndpoint)
    @mutex.synchronize { @endpoints.select(&.healthy).dup }
  end
  
  private def check_health
    endpoints = @mutex.synchronize { @endpoints.dup }
    
    endpoints.each_with_index do |endpoint, i|
      spawn do
        healthy = tcp_check(endpoint.host, endpoint.port)
        
        @mutex.synchronize do
          if i < @endpoints.size
            @endpoints[i] = ServiceEndpoint.new(endpoint.host, endpoint.port, healthy)
          end
        end
        
        puts "Health check #{endpoint.host}:#{endpoint.port}: #{healthy ? "OK" : "FAIL"}"
      end
    end
  end
  
  private def tcp_check(host : String, port : Int32, timeout : Time::Span = 2.seconds) : Bool
    socket = TCPSocket.new(host, port, connect_timeout: timeout)
    socket.close
    true
  rescue
    false
  end
end

sd = ServiceDiscovery.new
sd.discover("google.com", 80)
sd.start_health_checks(30.seconds)

sleep 0.5.seconds
puts "Healthy endpoints: #{sd.healthy_endpoints.size}"
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: DNS Benchmark

```crystal
require "socket"

# วัดเวลา DNS resolution
def dns_benchmark(hostname : String, n : Int32 = 10) : Hash(String, Float64)
  times = [] of Float64
  errors = 0
  
  n.times do
    start = Time.monotonic
    begin
      Socket::Addrinfo.resolve(hostname, 80)
      times << (Time.monotonic - start).total_milliseconds
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
    "errors" => errors.to_f,
    "error_rate" => (errors.to_f / n * 100),
  }
end

puts "=== DNS Benchmark ==="
["google.com", "cloudflare.com", "github.com"].each do |host|
  stats = dns_benchmark(host, 5)
  puts "#{host}: mean=#{stats["mean"].round(2)}ms, min=#{stats["min"].round(2)}ms, errors=#{stats["error_rate"].round(1)}%"
end
```

### แบบฝึกหัดที่ 2: DNS Monitor

```crystal
require "socket"

class DNSMonitor
  def initialize(@hostnames : Array(String), @interval : Time::Span = 60.seconds)
    @records = Hash(String, Array(String)).new
    @changes = Channel(Tuple(String, Array(String), Array(String))).new(100)
    @mutex = Mutex.new
  end
  
  def start
    @hostnames.each { |h| initial_lookup(h) }
    
    spawn monitor_loop
    spawn report_loop
    
    sleep
  end
  
  private def initial_lookup(hostname : String)
    ips = resolve(hostname)
    @mutex.synchronize { @records[hostname] = ips }
    puts "Initial: #{hostname} => #{ips.join(", ")}"
  end
  
  private def monitor_loop
    loop do
      sleep @interval
      
      @hostnames.each do |hostname|
        new_ips = resolve(hostname)
        old_ips = @mutex.synchronize { @records[hostname]?.dup || [] of String }
        
        unless new_ips.sort == old_ips.sort
          @changes.send({hostname, old_ips, new_ips})
          @mutex.synchronize { @records[hostname] = new_ips }
        end
      end
    end
  end
  
  private def report_loop
    while change = @changes.receive?
      hostname, old_ips, new_ips = change
      puts "DNS CHANGE: #{hostname}"
      puts "  Old: #{old_ips.join(", ")}"
      puts "  New: #{new_ips.join(", ")}"
      
      added = new_ips - old_ips
      removed = old_ips - new_ips
      puts "  Added: #{added.join(", ")}" unless added.empty?
      puts "  Removed: #{removed.join(", ")}" unless removed.empty?
    end
  end
  
  private def resolve(hostname : String) : Array(String)
    Socket::Addrinfo.resolve(hostname, 80)
      .map { |a| a.ip_address.to_s }
      .sort
  rescue Socket::Error
    [] of String
  end
end

monitor = DNSMonitor.new(
  ["google.com", "cloudflare.com", "github.com"],
  interval: 30.seconds
)
monitor.start
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Socket::Addrinfo.resolve**: DNS lookup พื้นฐาน
2. **Multiple Addresses**: จัดการ IPv4 และ IPv6
3. **DNS Caching**: cache results เพื่อลด latency
4. **Round Robin DNS**: load balancing ด้วย DNS
5. **Reverse Lookup**: IP -> hostname
6. **DNS-over-HTTPS (DoH)**: DNS ผ่าน HTTPS
7. **Service Discovery**: ค้นหา services ผ่าน DNS
8. **Health Checks**: ตรวจสอบ endpoint health
9. **DNS Monitor**: ตรวจสอบการเปลี่ยนแปลง DNS records

ข้อควรระวัง:
- DNS resolution อาจช้า ควร cache results
- DNS records มี TTL ควรนับถึง TTL ก่อน cache expire
- IPv6 อาจไม่รองรับใน network บางอัน
