# Part 123: TCP Sockets - การใช้งาน TCP Socket ใน Crystal

## บทนำ

TCP (Transmission Control Protocol) เป็น protocol ที่ใช้สื่อสารแบบ connection-oriented ซึ่งรับประกันการส่งข้อมูลให้ครบและถูกต้อง Crystal มี `TCPSocket` และ `TCPServer` ใน standard library ที่ใช้งานได้ง่าย

## TCPServer และ TCPSocket

### Echo Server

```crystal
require "socket"

# Echo Server: รับข้อความแล้วส่งกลับ
server = TCPServer.new("0.0.0.0", 3000)
puts "Echo Server รอรับ connection บน port 3000..."

while client = server.accept?
  spawn handle_echo_client(client)
end

def handle_echo_client(socket : TCPSocket)
  remote = socket.remote_address
  puts "Client เชื่อมต่อ: #{remote}"
  
  while line = socket.gets
    puts "Received from #{remote}: #{line}"
    socket.puts "Echo: #{line}"
  end
  
  puts "Client ตัดการเชื่อมต่อ: #{remote}"
  socket.close
end
```

### Echo Client

```crystal
require "socket"

# เชื่อมต่อกับ Echo Server
socket = TCPSocket.new("localhost", 3000)
puts "เชื่อมต่อสำเร็จ"

messages = ["สวัสดี", "Hello", "Crystal is awesome!", "Bye!"]

messages.each do |msg|
  socket.puts msg
  response = socket.gets
  puts "Response: #{response}"
end

socket.close
puts "ปิดการเชื่อมต่อ"
```

## Send และ Receive

```crystal
require "socket"

# Server ที่รับ binary data
server = TCPServer.new("0.0.0.0", 3001)

while client = server.accept?
  spawn do
    buffer = Bytes.new(1024)
    
    while bytes_read = client.read(buffer)
      break if bytes_read == 0
      
      data = buffer[0, bytes_read]
      puts "Received #{bytes_read} bytes: #{String.new(data)}"
      
      # ส่งกลับ
      client.write(data)
    end
    
    client.close
  end
end
```

```crystal
require "socket"

# Client ส่ง binary data
socket = TCPSocket.new("localhost", 3001)

# ส่ง bytes
data = "Binary message".to_slice
bytes_written = socket.write(data)
puts "Sent #{bytes_written} bytes"

# รับ bytes
buffer = Bytes.new(1024)
bytes_read = socket.read(buffer)
puts "Received: #{String.new(buffer[0, bytes_read])}"

socket.close
```

## Protocol Design

### Length-Prefixed Protocol

```crystal
require "socket"
require "io"

# Protocol: [4 bytes length][data]
module Protocol
  def self.send_message(socket : IO, message : String)
    data = message.to_slice
    # ส่ง length (4 bytes, big-endian)
    socket.write_bytes(data.size.to_u32, IO::ByteFormat::BigEndian)
    # ส่ง data
    socket.write(data)
    socket.flush
  end
  
  def self.receive_message(socket : IO) : String?
    # อ่าน length
    length_bytes = Bytes.new(4)
    return nil if socket.read(length_bytes) < 4
    
    length = IO::ByteFormat::BigEndian.decode(UInt32, length_bytes)
    return nil if length == 0 || length > 1_000_000  # max 1MB
    
    # อ่าน data
    data = Bytes.new(length)
    socket.read_fully(data)
    String.new(data)
  end
end

# Server
spawn do
  server = TCPServer.new("0.0.0.0", 3002)
  
  while client = server.accept?
    spawn do
      while msg = Protocol.receive_message(client)
        puts "Server received: #{msg}"
        Protocol.send_message(client, "Received: #{msg}")
      end
      client.close
    end
  end
end

sleep 0.1.seconds

# Client
socket = TCPSocket.new("localhost", 3002)

["Hello", "Crystal TCP", "ข้อความยาวๆ " + "x" * 100].each do |msg|
  Protocol.send_message(socket, msg)
  if response = Protocol.receive_message(socket)
    puts "Client received: #{response[0..50]}"
  end
end

socket.close
```

## Chat Server

```crystal
require "socket"
require "json"

# Chat Server พร้อม rooms
class ChatServer
  struct Message
    include JSON::Serializable
    
    property type : String
    property from : String
    property content : String
    property room : String
    property timestamp : String
    
    def initialize(@type, @from, @content, @room = "general")
      @timestamp = Time.local.to_s
    end
  end
  
  def initialize(@port : Int32)
    @clients = Hash(TCPSocket, String).new  # socket => username
    @rooms = Hash(String, Set(TCPSocket)).new { |h, k| h[k] = Set(TCPSocket).new }
    @mutex = Mutex.new
  end
  
  def start
    server = TCPServer.new("0.0.0.0", @port)
    puts "Chat Server started on port #{@port}"
    
    while client = server.accept?
      spawn handle_client(client)
    end
  end
  
  private def handle_client(socket : TCPSocket)
    # Register user
    username = nil
    
    socket.puts "ยินดีต้อนรับสู่ Crystal Chat! กรุณาใส่ชื่อของคุณ:"
    username = socket.gets.try(&.chomp)
    
    unless username && !username.empty?
      socket.close
      return
    end
    
    @mutex.synchronize do
      @clients[socket] = username
      @rooms["general"] << socket
    end
    
    puts "#{username} เข้าร่วม"
    broadcast_to_room("general", Message.new("join", "System", "#{username} เข้าร่วม"))
    
    socket.puts "ยินดีต้อนรับ #{username}! คุณอยู่ใน #general"
    socket.puts "Commands: /join <room>, /rooms, /users, /quit"
    
    # Handle messages
    while line = socket.gets
      line = line.chomp
      
      if line.starts_with?("/")
        handle_command(socket, username, line)
      elsif !line.empty?
        room = current_room(socket)
        msg = Message.new("message", username, line, room)
        broadcast_to_room(room, msg, exclude: socket)
        puts "[#{room}] #{username}: #{line}"
      end
    end
  rescue IO::Error
    # Client disconnected
  ensure
    cleanup_client(socket, username || "Unknown")
  end
  
  private def handle_command(socket : TCPSocket, username : String, cmd : String)
    parts = cmd.split(" ", 2)
    
    case parts[0]
    when "/join"
      room = parts[1]? || "general"
      join_room(socket, username, room)
    when "/rooms"
      rooms = @mutex.synchronize { @rooms.keys.sort }
      socket.puts "Rooms: #{rooms.join(", ")}"
    when "/users"
      room = current_room(socket)
      users = @mutex.synchronize do
        @rooms[room].map { |s| @clients[s]? || "Unknown" }
      end
      socket.puts "Users in ##{room}: #{users.join(", ")}"
    when "/quit"
      socket.puts "Goodbye!"
      socket.close
    else
      socket.puts "Unknown command: #{parts[0]}"
    end
  end
  
  private def join_room(socket : TCPSocket, username : String, room : String)
    old_room = current_room(socket)
    
    @mutex.synchronize do
      @rooms[old_room].delete(socket)
      @rooms[room] << socket
    end
    
    socket.puts "คุณย้ายไปห้อง ##{room}"
    broadcast_to_room(room, Message.new("join", "System", "#{username} เข้าร่วม #{room}"), exclude: socket)
  end
  
  private def current_room(socket : TCPSocket) : String
    @mutex.synchronize do
      @rooms.find { |_, sockets| sockets.includes?(socket) }.try(&.first) || "general"
    end
  end
  
  private def broadcast_to_room(room : String, msg : Message, exclude : TCPSocket? = nil)
    sockets = @mutex.synchronize { @rooms[room].dup }
    
    sockets.each do |socket|
      next if socket == exclude
      begin
        socket.puts msg.to_json
      rescue
        # Socket closed
      end
    end
  end
  
  private def cleanup_client(socket : TCPSocket, username : String)
    room = current_room(socket)
    
    @mutex.synchronize do
      @clients.delete(socket)
      @rooms.each_value { |sockets| sockets.delete(socket) }
    end
    
    broadcast_to_room(room, Message.new("leave", "System", "#{username} ออกจากห้อง"))
    puts "#{username} ออกจาก"
    socket.close rescue nil
  end
end

# Start chat server
server = ChatServer.new(3003)
server.start
```

## Socket Options

```crystal
require "socket"

server = TCPServer.new("0.0.0.0", 3004)

# เปิด SO_REUSEADDR เพื่อ reuse port ได้ทันที
server.reuse_address = true
server.reuse_port = true  # Linux เท่านั้น

# TCP KeepAlive
client = server.accept
client.keepalive = true

# Buffer sizes
client.recv_buffer_size = 65536
client.send_buffer_size = 65536

# TCP No Delay (disable Nagle's algorithm)
client.tcp_nodelay = true

# Timeout
client.read_timeout = 30.seconds
client.write_timeout = 30.seconds
```

## Non-Blocking Connect

```crystal
require "socket"

def connect_with_timeout(host : String, port : Int32, timeout : Time::Span) : TCPSocket
  channel = Channel(TCPSocket | Exception).new(1)
  
  spawn do
    begin
      socket = TCPSocket.new(host, port, connect_timeout: timeout)
      channel.send(socket)
    rescue ex
      channel.send(ex)
    end
  end
  
  result = channel.receive
  case result
  when TCPSocket
    result
  when Exception
    raise result
  else
    raise "Unknown error"
  end
end

begin
  socket = connect_with_timeout("localhost", 3000, 5.seconds)
  socket.puts "Hello!"
  socket.close
rescue ex
  puts "Cannot connect: #{ex.message}"
end
```

## SSL/TLS บน TCP

```crystal
require "socket"
require "openssl"

# TLS Server
server = TCPServer.new("0.0.0.0", 8443)
tls_context = OpenSSL::SSL::Context::Server.new
tls_context.certificate_chain = "server.crt"
tls_context.private_key = "server.key"

while client = server.accept?
  spawn do
    tls_socket = OpenSSL::SSL::Socket::Server.new(client, tls_context)
    begin
      while line = tls_socket.gets
        tls_socket.puts "Secure echo: #{line}"
      end
    ensure
      tls_socket.close
      client.close
    end
  end
end
```

## Connection Pool

```crystal
require "socket"

class TCPConnectionPool
  def initialize(@host : String, @port : Int32, @size : Int32)
    @connections = Channel(TCPSocket).new(@size)
    @size.times { @connections.send(create_connection) }
  end
  
  def with_connection(&block : TCPSocket -> T) : T forall T
    conn = @connections.receive
    begin
      block.call(conn)
    rescue ex : IO::Error
      # Connection ขาด, สร้างใหม่
      conn.close rescue nil
      conn = create_connection
      raise ex
    ensure
      @connections.send(conn)
    end
  end
  
  def close
    @size.times do
      @connections.receive.close rescue nil
    end
    @connections.close
  end
  
  private def create_connection : TCPSocket
    socket = TCPSocket.new(@host, @port)
    socket.tcp_nodelay = true
    socket.keepalive = true
    socket
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: HTTP/1.0 Server

```crystal
require "socket"

# สร้าง HTTP/1.0 server อย่างง่าย
class SimpleHTTPServer
  def initialize(@port : Int32)
  end
  
  def start
    server = TCPServer.new("0.0.0.0", @port)
    puts "HTTP Server on port #{@port}"
    
    while client = server.accept?
      spawn handle_http(client)
    end
  end
  
  private def handle_http(socket : TCPSocket)
    # อ่าน request
    request_line = socket.gets
    return unless request_line
    
    method, path, version = request_line.split(" ")
    headers = {} of String => String
    
    while line = socket.gets
      break if line.chomp.empty?
      parts = line.split(": ", 2)
      headers[parts[0]] = parts[1].chomp if parts.size == 2
    end
    
    # สร้าง response
    body, status, content_type = generate_response(method, path)
    
    socket.print "HTTP/1.0 #{status}\r\n"
    socket.print "Content-Type: #{content_type}\r\n"
    socket.print "Content-Length: #{body.bytesize}\r\n"
    socket.print "Connection: close\r\n"
    socket.print "\r\n"
    socket.print body
    
    socket.close
  end
  
  private def generate_response(method : String, path : String) : {String, String, String}
    case path
    when "/"
      {"<html><body><h1>Crystal HTTP Server</h1></body></html>", "200 OK", "text/html"}
    when "/health"
      {{"status" => "ok"}.to_json, "200 OK", "application/json"}
    else
      {"<html><body><h1>404 Not Found</h1></body></html>", "404 Not Found", "text/html"}
    end
  end
end

server = SimpleHTTPServer.new(8888)
server.start
```

### แบบฝึกหัดที่ 2: Load Balancer

```crystal
require "socket"

class LoadBalancer
  def initialize(@listen_port : Int32, backends : Array(Tuple(String, Int32)))
    @backends = backends
    @current = Atomic(Int32).new(0)
    @healthy = Set(Tuple(String, Int32)).new(backends)
    @mutex = Mutex.new
  end
  
  def start
    server = TCPServer.new("0.0.0.0", @listen_port)
    puts "Load Balancer on port #{@listen_port}"
    puts "Backends: #{@backends.inspect}"
    
    start_health_checker
    
    while client = server.accept?
      spawn proxy_connection(client)
    end
  end
  
  private def next_backend : Tuple(String, Int32)?
    healthy = @mutex.synchronize { @healthy.to_a }
    return nil if healthy.empty?
    idx = @current.add(1) % healthy.size
    healthy[idx]
  end
  
  private def proxy_connection(client : TCPSocket)
    backend = next_backend
    
    unless backend
      client.close
      return
    end
    
    host, port = backend
    
    begin
      upstream = TCPSocket.new(host, port)
      
      # Bidirectional proxy
      done = Channel(Nil).new(2)
      
      spawn do
        pipe(client, upstream)
        done.send(nil)
      end
      
      spawn do
        pipe(upstream, client)
        done.send(nil)
      end
      
      done.receive
    rescue ex
      puts "Backend #{host}:#{port} failed: #{ex.message}"
      @mutex.synchronize { @healthy.delete(backend) }
    ensure
      client.close rescue nil
    end
  end
  
  private def pipe(src : IO, dst : IO)
    buffer = Bytes.new(4096)
    while n = src.read(buffer)
      break if n == 0
      dst.write(buffer[0, n])
    end
  rescue IO::Error
  end
  
  private def start_health_checker
    spawn do
      loop do
        sleep 10.seconds
        
        @backends.each do |host, port|
          begin
            socket = TCPSocket.new(host, port, connect_timeout: 2.seconds)
            socket.close
            @mutex.synchronize { @healthy << {host, port} }
          rescue
            @mutex.synchronize { @healthy.delete({host, port}) }
            puts "Backend #{host}:#{port} is DOWN"
          end
        end
      end
    end
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **TCPServer**: รอรับ connections บน port
2. **TCPSocket**: เชื่อมต่อ TCP
3. **accept/accept?**: รับ client connections
4. **send/receive**: ส่งและรับข้อมูล
5. **Protocol Design**: Length-prefixed protocol
6. **Echo Server**: ตัวอย่าง server อย่างง่าย
7. **Chat Server**: multi-client, multi-room chat
8. **Socket Options**: keepalive, nodelay, buffer sizes
9. **Connection Pool**: reuse connections
10. **SSL/TLS**: การเข้ารหัสบน TCP
