# Part 124: UDP Sockets - การใช้งาน UDP Socket ใน Crystal

## บทนำ

UDP (User Datagram Protocol) เป็น protocol ที่ส่งข้อมูลแบบ connectionless ไม่รับประกันการส่ง แต่เร็วกว่า TCP เหมาะกับ real-time applications เช่น games, video streaming, DNS, VoIP

## UDPSocket พื้นฐาน

### UDP Server

```crystal
require "socket"

# UDP Server
socket = UDPSocket.new
socket.bind("0.0.0.0", 5000)
puts "UDP Server รอรับ datagrams บน port 5000..."

buffer = Bytes.new(4096)

loop do
  bytes_received, remote_addr = socket.receive(buffer)
  message = String.new(buffer[0, bytes_received])
  
  puts "รับจาก #{remote_addr}: #{message}"
  
  # ส่ง response กลับ
  response = "ได้รับ: #{message}"
  socket.send(response, to: remote_addr)
end
```

### UDP Client

```crystal
require "socket"

# UDP Client
socket = UDPSocket.new

server_addr = Socket::IPAddress.new("127.0.0.1", 5000)
buffer = Bytes.new(4096)

["Hello", "Crystal UDP", "สวัสดี", "Goodbye"].each do |msg|
  # ส่ง datagram
  socket.send(msg, to: server_addr)
  puts "ส่ง: #{msg}"
  
  # รับ response
  bytes, _ = socket.receive(buffer)
  puts "รับ: #{String.new(buffer[0, bytes])}"
end

socket.close
```

## Send และ Receive

```crystal
require "socket"

# ส่งและรับในแบบต่างๆ
socket = UDPSocket.new

# วิธีที่ 1: ส่ง String
server = Socket::IPAddress.new("127.0.0.1", 5000)
socket.send("Hello, UDP!", to: server)

# วิธีที่ 2: ส่ง Bytes
data = Bytes[0x01, 0x02, 0x03, 0x04]
socket.send(data, to: server)

# วิธีที่ 3: รับพร้อม address
buffer = Bytes.new(65535)  # Max UDP datagram size
bytes, addr = socket.receive(buffer)
puts "รับ #{bytes} bytes จาก #{addr}"

# วิธีที่ 4: bind แล้วรับ (ไม่ต้องรับ address)
socket.bind("0.0.0.0", 5001)
bytes = socket.read(buffer)
puts "รับ #{bytes} bytes"

socket.close
```

## Broadcast

```crystal
require "socket"

# UDP Broadcast: ส่งไปทุก devices ใน network

# Broadcast Sender
sender = UDPSocket.new
sender.broadcast = true  # เปิดใช้งาน broadcast

broadcast_addr = Socket::IPAddress.new("255.255.255.255", 5001)

10.times do |i|
  message = "Broadcast message #{i + 1}"
  sender.send(message, to: broadcast_addr)
  puts "Broadcast: #{message}"
  sleep 1.second
end

sender.close
```

```crystal
require "socket"

# Broadcast Receiver
receiver = UDPSocket.new
receiver.broadcast = true
receiver.reuse_address = true
receiver.bind("0.0.0.0", 5001)

puts "รอรับ broadcasts..."
buffer = Bytes.new(4096)

loop do
  bytes, addr = receiver.receive(buffer)
  puts "Broadcast จาก #{addr}: #{String.new(buffer[0, bytes])}"
end
```

## Multicast

```crystal
require "socket"

MULTICAST_GROUP = "224.0.0.1"
MULTICAST_PORT = 5002

# Multicast Sender
sender = UDPSocket.new
multicast_addr = Socket::IPAddress.new(MULTICAST_GROUP, MULTICAST_PORT)

# กำหนด TTL (Time To Live) - จำนวน network hops
sender.multicast_ttl = 1 # ส่งไปแค่ local network

5.times do |i|
  message = "Multicast: สวัสดีจาก Crystal #{i + 1}"
  sender.send(message, to: multicast_addr)
  puts "Sent: #{message}"
  sleep 0.5.seconds
end

sender.close
```

```crystal
require "socket"

# Multicast Receiver
receiver = UDPSocket.new
receiver.reuse_address = true
receiver.bind("0.0.0.0", MULTICAST_PORT)

# เข้าร่วม multicast group
receiver.join_group(Socket::IPAddress.new(MULTICAST_GROUP, 0))
puts "เข้าร่วม multicast group #{MULTICAST_GROUP}"

buffer = Bytes.new(4096)
5.times do
  bytes, addr = receiver.receive(buffer)
  puts "รับ multicast จาก #{addr}: #{String.new(buffer[0, bytes])}"
end

receiver.leave_group(Socket::IPAddress.new(MULTICAST_GROUP, 0))
receiver.close
```

## UDP Server แบบ Full-Duplex

```crystal
require "socket"
require "json"

# UDP Server ที่รองรับ multiple clients
class UDPServer
  MAX_DATAGRAM_SIZE = 65535
  
  def initialize(@port : Int32)
    @socket = UDPSocket.new
    @socket.bind("0.0.0.0", @port)
    @socket.read_timeout = 1.second
    @clients = {} of String => Time
    @mutex = Mutex.new
  end
  
  def start
    puts "UDP Server on port #{@port}"
    
    # Thread สำหรับ receive
    spawn receive_loop
    
    # Thread สำหรับ cleanup inactive clients
    spawn cleanup_loop
    
    sleep
  end
  
  private def receive_loop
    buffer = Bytes.new(MAX_DATAGRAM_SIZE)
    
    loop do
      begin
        bytes, addr = @socket.receive(buffer)
        message = String.new(buffer[0, bytes])
        
        # Register client
        client_key = addr.to_s
        @mutex.synchronize { @clients[client_key] = Time.local }
        
        # Process message
        spawn process_message(addr, message)
      rescue IO::TimeoutError
        # timeout - loop again
      rescue IO::Error => ex
        puts "Receive error: #{ex.message}"
        break
      end
    end
  end
  
  private def process_message(addr : Socket::IPAddress, message : String)
    puts "#{addr}: #{message}"
    
    begin
      data = JSON.parse(message)
      cmd = data["cmd"]?.try(&.as_s?)
      
      response = case cmd
      when "ping"
        {"type" => "pong", "timestamp" => Time.local.to_unix.to_s}.to_json
      when "echo"
        {"type" => "echo", "data" => data["data"]?}.to_json
      when "time"
        {"type" => "time", "value" => Time.local.to_s}.to_json
      else
        {"type" => "error", "message" => "Unknown command"}.to_json
      end
      
      @socket.send(response, to: addr)
    rescue JSON::ParseException
      @socket.send({"type" => "error", "message" => "Invalid JSON"}.to_json, to: addr)
    end
  end
  
  private def cleanup_loop
    loop do
      sleep 30.seconds
      
      cutoff = Time.local - 60.seconds
      @mutex.synchronize do
        @clients.reject! { |_, last_seen| last_seen < cutoff }
      end
      
      puts "Active clients: #{@mutex.synchronize { @clients.size }}"
    end
  end
end

server = UDPServer.new(5003)
server.start
```

## UDP Client แบบ Reliable

```crystal
require "socket"

# UDP ไม่รับประกันการส่ง ต้องทำ reliability เอง
class ReliableUDPClient
  MAX_RETRIES = 3
  TIMEOUT = 2.seconds
  
  def initialize(@server_host : String, @server_port : Int32)
    @socket = UDPSocket.new
    @socket.read_timeout = TIMEOUT
    @server_addr = Socket::IPAddress.new(@server_host, @server_port)
    @seq_num = Atomic(UInt32).new(0)
  end
  
  def send_reliable(message : String) : String?
    seq = @seq_num.add(1)
    packet = "#{seq}:#{message}"
    
    MAX_RETRIES.times do |attempt|
      @socket.send(packet, to: @server_addr)
      
      begin
        buffer = Bytes.new(65535)
        bytes, _ = @socket.receive(buffer)
        response = String.new(buffer[0, bytes])
        
        # ตรวจสอบ sequence number
        parts = response.split(":", 2)
        if parts.size == 2 && parts[0].to_u32? == seq
          return parts[1]
        end
      rescue IO::TimeoutError
        puts "Timeout (attempt #{attempt + 1}/#{MAX_RETRIES}), retrying..."
      end
    end
    
    nil
  end
  
  def close
    @socket.close
  end
end

# ทดสอบ
client = ReliableUDPClient.new("127.0.0.1", 5003)

if response = client.send_reliable({"cmd" => "ping"}.to_json)
  puts "Response: #{response}"
else
  puts "Failed to get response"
end

client.close
```

## UDP Hole Punching

```crystal
require "socket"

# UDP Hole Punching สำหรับ P2P connections
# ทำให้ clients สามารถคุยกันโดยตรงได้ แม้อยู่หลัง NAT

class P2PCoordinator
  def initialize(@port : Int32)
    @socket = UDPSocket.new
    @socket.bind("0.0.0.0", @port)
    @peers = {} of String => Socket::IPAddress
    @mutex = Mutex.new
  end
  
  def start
    puts "P2P Coordinator on port #{@port}"
    buffer = Bytes.new(4096)
    
    loop do
      bytes, addr = @socket.receive(buffer)
      message = String.new(buffer[0, bytes])
      parts = message.split(":", 2)
      
      case parts[0]
      when "REGISTER"
        peer_id = parts[1]? || "unknown"
        @mutex.synchronize { @peers[peer_id] = addr }
        puts "Registered: #{peer_id} @ #{addr}"
        @socket.send("OK:#{addr}", to: addr)
        
      when "CONNECT"
        target_id = parts[1]? || ""
        if target = @mutex.synchronize { @peers[target_id]? }
          # ส่ง address ของ target ให้ requester
          @socket.send("PEER:#{target}", to: addr)
          # ส่ง address ของ requester ให้ target
          @socket.send("PEER:#{addr}", to: target)
          puts "Connecting #{addr} <-> #{target}"
        else
          @socket.send("ERROR:Peer not found", to: addr)
        end
      end
    end
  end
end
```

## DNS-like UDP Server

```crystal
require "socket"

# Simple UDP-based key-value store (คล้าย DNS)
class UDPKeyValueStore
  def initialize(@port : Int32)
    @data = {} of String => String
    @socket = UDPSocket.new
    @socket.bind("0.0.0.0", @port)
    @mutex = Mutex.new
  end
  
  def start
    puts "UDP KV Store on port #{@port}"
    buffer = Bytes.new(4096)
    
    loop do
      bytes, addr = @socket.receive(buffer)
      request = String.new(buffer[0, bytes]).chomp
      parts = request.split(" ", 3)
      
      response = case parts[0].upcase
      when "SET"
        key, value = parts[1]?, parts[2]?
        if key && value
          @mutex.synchronize { @data[key] = value }
          "OK"
        else
          "ERR Invalid SET"
        end
      when "GET"
        key = parts[1]?
        if key
          @mutex.synchronize { @data[key]? } || "NIL"
        else
          "ERR Invalid GET"
        end
      when "DEL"
        key = parts[1]?
        if key
          @mutex.synchronize { @data.delete(key) }
          "OK"
        else
          "ERR Invalid DEL"
        end
      when "KEYS"
        @mutex.synchronize { @data.keys.join(",") }
      else
        "ERR Unknown command"
      end
      
      @socket.send(response, to: addr)
    end
  end
end

# ทดสอบ
spawn do
  store = UDPKeyValueStore.new(5004)
  store.start
end

sleep 0.1.seconds

client = UDPSocket.new
server = Socket::IPAddress.new("127.0.0.1", 5004)
buffer = Bytes.new(4096)

def kv_command(client, server, buffer, cmd)
  client.send(cmd, to: server)
  bytes, _ = client.receive(buffer)
  String.new(buffer[0, bytes])
end

puts kv_command(client, server, buffer, "SET name Crystal")
puts kv_command(client, server, buffer, "SET version 1.14")
puts kv_command(client, server, buffer, "GET name")
puts kv_command(client, server, buffer, "KEYS")
puts kv_command(client, server, buffer, "DEL name")
puts kv_command(client, server, buffer, "GET name")

client.close
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: UDP Game Server

```crystal
require "socket"
require "json"

# Simple multiplayer game state server
class GameServer
  struct Player
    property id : String
    property x : Float64
    property y : Float64
    property score : Int32
    property last_seen : Time
    
    def initialize(@id, @x = 0.0, @y = 0.0, @score = 0)
      @last_seen = Time.local
    end
  end
  
  def initialize(@port : Int32)
    @socket = UDPSocket.new
    @socket.bind("0.0.0.0", @port)
    @players = {} of String => {Player, Socket::IPAddress}
    @mutex = Mutex.new
    @tick_rate = 20  # 20 updates/second
  end
  
  def start
    puts "Game Server on port #{@port}"
    
    spawn receive_loop
    spawn game_loop
    spawn cleanup_loop
    
    sleep
  end
  
  private def receive_loop
    buffer = Bytes.new(4096)
    loop do
      bytes, addr = @socket.receive(buffer)
      data = JSON.parse(String.new(buffer[0, bytes]))
      
      type = data["type"]?.try(&.as_s?)
      player_id = data["id"]?.try(&.as_s?)
      
      next unless type && player_id
      
      case type
      when "join"
        player = Player.new(player_id, rand(0.0..800.0), rand(0.0..600.0))
        @mutex.synchronize { @players[player_id] = {player, addr} }
        puts "#{player_id} joined"
        
      when "move"
        dx = data["dx"]?.try(&.as_f?) || 0.0
        dy = data["dy"]?.try(&.as_f?) || 0.0
        
        @mutex.synchronize do
          if tuple = @players[player_id]?
            p, a = tuple
            new_x = (p.x + dx).clamp(0.0, 800.0)
            new_y = (p.y + dy).clamp(0.0, 600.0)
            @players[player_id] = {Player.new(player_id, new_x, new_y, p.score), addr}
          end
        end
        
      when "leave"
        @mutex.synchronize { @players.delete(player_id) }
        puts "#{player_id} left"
      end
    rescue JSON::ParseException
      # ignore invalid JSON
    end
  end
  
  private def game_loop
    interval = (1.0 / @tick_rate).seconds
    loop do
      sleep interval
      broadcast_state
    end
  end
  
  private def broadcast_state
    state = @mutex.synchronize do
      @players.map do |id, tuple|
        p, _ = tuple
        {"id" => id, "x" => p.x, "y" => p.y, "score" => p.score}
      end
    end
    
    state_json = {"type" => "state", "players" => state}.to_json
    
    @mutex.synchronize { @players.values.map(&.last) }.each do |addr|
      @socket.send(state_json, to: addr) rescue nil
    end
  end
  
  private def cleanup_loop
    loop do
      sleep 5.seconds
      cutoff = Time.local - 10.seconds
      @mutex.synchronize do
        @players.reject! { |_, tuple| tuple.first.last_seen < cutoff }
      end
    end
  end
end

server = GameServer.new(5005)
server.start
```

### แบบฝึกหัดที่ 2: UDP Stats Collector

```crystal
require "socket"
require "json"

# Collect metrics ผ่าน UDP (คล้าย StatsD)
class StatsCollector
  struct Metric
    property name : String
    property value : Float64
    property type : String
    property timestamp : Time
    
    def initialize(@name, @value, @type)
      @timestamp = Time.local
    end
  end
  
  def initialize(@port : Int32)
    @socket = UDPSocket.new
    @socket.bind("0.0.0.0", @port)
    @metrics = {} of String => Array(Metric)
    @mutex = Mutex.new
    @counters = Hash(String, Float64).new(0.0)
    @gauges = Hash(String, Float64).new(0.0)
    @timings = Hash(String, Array(Float64)).new { |h, k| h[k] = [] of Float64 }
  end
  
  def start
    puts "Stats Collector on port #{@port}"
    
    spawn receive_loop
    spawn report_loop
    
    sleep
  end
  
  private def receive_loop
    buffer = Bytes.new(4096)
    loop do
      bytes, addr = @socket.receive(buffer)
      parse_metric(String.new(buffer[0, bytes]).chomp)
    end
  end
  
  private def parse_metric(line : String)
    # Format: metric.name:value|type
    # Types: c (counter), g (gauge), ms (timing)
    if match = line.match(/^([^:]+):([^|]+)\|(.+)$/)
      name = match[1]
      value = match[2].to_f? || 0.0
      type = match[3]
      
      @mutex.synchronize do
        case type
        when "c"
          @counters[name] += value
        when "g"
          @gauges[name] = value
        when "ms"
          @timings[name] << value
        end
      end
    end
  end
  
  private def report_loop
    loop do
      sleep 5.seconds
      
      @mutex.synchronize do
        puts "\n=== Stats Report ==="
        
        @counters.each { |k, v| puts "Counter #{k}: #{v}" }
        @gauges.each { |k, v| puts "Gauge #{k}: #{v}" }
        @timings.each do |k, values|
          next if values.empty?
          sorted = values.sort
          puts "Timing #{k}: mean=#{(values.sum/values.size).round(2)}ms p95=#{sorted[(values.size*0.95).to_i]}ms"
        end
      end
    end
  end
end

# Stats Client
class StatsClient
  def initialize(@host : String, @port : Int32)
    @socket = UDPSocket.new
    @server = Socket::IPAddress.new(@host, @port)
  end
  
  def increment(name : String, value : Float64 = 1.0)
    send_metric("#{name}:#{value}|c")
  end
  
  def gauge(name : String, value : Float64)
    send_metric("#{name}:#{value}|g")
  end
  
  def timing(name : String, ms : Float64)
    send_metric("#{name}:#{ms}|ms")
  end
  
  def time(name : String, &block)
    start = Time.monotonic
    result = block.call
    elapsed = (Time.monotonic - start).total_milliseconds
    timing(name, elapsed)
    result
  end
  
  private def send_metric(data : String)
    @socket.send(data, to: @server) rescue nil
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **UDPSocket**: การสร้าง UDP socket สำหรับส่งและรับข้อมูล
2. **bind**: การผูก socket กับ port
3. **send/receive**: ส่งและรับ datagrams
4. **Broadcast**: ส่งไปทุก devices ใน network
5. **Multicast**: ส่งไปกลุ่ม subscribers
6. **Reliable UDP**: การทำ reliability layer บน UDP
7. **UDP Hole Punching**: P2P connections ผ่าน NAT
8. **Game Server**: real-time multiplayer game state
9. **Stats Collection**: metrics collection ด้วย UDP (StatsD pattern)

เมื่อใช้ UDP:
- ต้องจัดการ out-of-order packets เอง
- ต้องทำ retry mechanism เองถ้าต้องการ reliability
- เหมาะกับ real-time, latency-sensitive applications
- Max datagram size ประมาณ 65535 bytes (ในทางปฏิบัติ ไม่ควรเกิน 1400 bytes)
