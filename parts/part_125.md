# Part 125: WebSockets - การใช้งาน WebSocket ใน Crystal

## บทนำ

WebSocket เป็น protocol ที่ทำให้สามารถสื่อสารแบบ full-duplex ระหว่าง client และ server ผ่าน single TCP connection ทำให้เหมาะกับ real-time applications เช่น chat, live updates, gaming

## HTTP::WebSocket Server

```crystal
require "http/server"

# WebSocket Server พื้นฐาน
server = HTTP::Server.new do |context|
  if context.request.headers.includes_word?("Upgrade", "websocket")
    ws_handler = HTTP::WebSocketHandler.new do |ws, ctx|
      puts "Client เชื่อมต่อ: #{ctx.request.remote_address}"
      
      ws.on_message do |message|
        puts "รับ: #{message}"
        ws.send("Echo: #{message}")
      end
      
      ws.on_close do
        puts "Client ตัดการเชื่อมต่อ"
      end
    end
    
    ws_handler.call(context)
  else
    context.response.print "WebSocket Server is running"
  end
end

server.listen("0.0.0.0", 3000)
```

## WebSocket Handler พร้อม Routes

```crystal
require "http/server"

# Handler สำหรับ WebSocket
class WebSocketManager
  include HTTP::Handler
  
  alias MessageHandler = Proc(HTTP::WebSocket, String, Nil)
  
  def initialize
    @ws_handlers = {} of String => MessageHandler
  end
  
  def ws(path : String, &handler : HTTP::WebSocket, String ->)
    @ws_handlers[path] = handler
  end
  
  def call(context : HTTP::Server::Context)
    path = context.request.path
    
    if context.request.headers.includes_word?("Upgrade", "websocket")
      if handler = @ws_handlers[path]?
        HTTP::WebSocketHandler.new { |ws, ctx| handler.call(ws, path) }.call(context)
      else
        context.response.status_code = 404
        context.response.print "WebSocket path not found"
      end
    else
      call_next(context)
    end
  end
end

ws_manager = WebSocketManager.new

ws_manager.ws("/echo") do |ws, path|
  ws.on_message { |msg| ws.send(msg) }
end

ws_manager.ws("/time") do |ws, path|
  spawn do
    loop do
      ws.send(Time.local.to_s)
      sleep 1.second
    end
  rescue IO::Error
    # Client disconnected
  end
end

server = HTTP::Server.new([ws_manager])
server.listen("0.0.0.0", 3000)
```

## Ping/Pong

```crystal
require "http/server"

server = HTTP::Server.new do |context|
  next unless context.request.headers.includes_word?("Upgrade", "websocket")
  
  HTTP::WebSocketHandler.new do |ws|
    # Ping/Pong สำหรับ keep-alive
    spawn do
      loop do
        sleep 30.seconds
        begin
          ws.ping("heartbeat")
        rescue IO::Error
          break
        end
      end
    end
    
    ws.on_ping do |message|
      puts "Received ping: #{message}"
      ws.pong(message)  # ตอบ pong
    end
    
    ws.on_pong do |message|
      puts "Received pong: #{message}"
    end
    
    ws.on_message do |msg|
      ws.send("Reply: #{msg}")
    end
    
    ws.on_close do |code, message|
      puts "Closed: code=#{code}, message=#{message}"
    end
  end.call(context)
end

server.listen("0.0.0.0", 3000)
```

## Chat Application

```crystal
require "http/server"
require "json"

# Chat Room implementation
class ChatRoom
  struct Message
    include JSON::Serializable
    
    property type : String
    property user : String
    property content : String
    property timestamp : String
    
    def initialize(@type, @user, @content)
      @timestamp = Time.local.to_rfc3339
    end
  end
  
  def initialize(@name : String)
    @connections = {} of String => HTTP::WebSocket  # username => ws
    @mutex = Mutex.new
    @message_history = [] of Message
  end
  
  def join(username : String, ws : HTTP::WebSocket)
    @mutex.synchronize { @connections[username] = ws }
    
    # ส่ง history ให้ user ใหม่
    history = @mutex.synchronize { @message_history.last(20).dup }
    history_json = {"type" => "history", "messages" => history}.to_json
    ws.send(history_json) rescue nil
    
    # แจ้ง users อื่น
    join_msg = Message.new("join", "System", "#{username} เข้าร่วมห้อง #{@name}")
    broadcast(join_msg, except: username)
    save_message(join_msg)
    
    puts "#{username} joined #{@name}"
  end
  
  def leave(username : String)
    @mutex.synchronize { @connections.delete(username) }
    
    leave_msg = Message.new("leave", "System", "#{username} ออกจากห้อง")
    broadcast(leave_msg)
    save_message(leave_msg)
    
    puts "#{username} left #{@name}"
  end
  
  def send_message(username : String, content : String)
    msg = Message.new("message", username, content)
    broadcast(msg)
    save_message(msg)
  end
  
  def user_count
    @mutex.synchronize { @connections.size }
  end
  
  def users
    @mutex.synchronize { @connections.keys.dup }
  end
  
  private def broadcast(message : Message, except : String? = nil)
    json = message.to_json
    connections = @mutex.synchronize { @connections.dup }
    
    connections.each do |username, ws|
      next if username == except
      begin
        ws.send(json)
      rescue IO::Error
        @mutex.synchronize { @connections.delete(username) }
      end
    end
  end
  
  private def save_message(msg : Message)
    @mutex.synchronize do
      @message_history << msg
      @message_history.shift if @message_history.size > 100
    end
  end
end

class ChatServer
  def initialize
    @rooms = Hash(String, ChatRoom).new { |h, k| h[k] = ChatRoom.new(k) }
    @mutex = Mutex.new
  end
  
  def get_or_create_room(name : String) : ChatRoom
    @mutex.synchronize { @rooms[name] }
  end
  
  def handle_connection(ws : HTTP::WebSocket, ctx : HTTP::Server::Context)
    # รับ username และ room จาก query params
    params = HTTP::Params.parse(ctx.request.query || "")
    username = params["username"]? || "Anonymous_#{Random.rand(1000)}"
    room_name = params["room"]? || "general"
    
    room = get_or_create_room(room_name)
    room.join(username, ws)
    
    ws.on_message do |message|
      begin
        data = JSON.parse(message)
        cmd = data["cmd"]?.try(&.as_s?)
        
        case cmd
        when "message"
          content = data["content"]?.try(&.as_s?) || ""
          room.send_message(username, content) unless content.empty?
        when "switch_room"
          new_room_name = data["room"]?.try(&.as_s?) || "general"
          room.leave(username)
          room = get_or_create_room(new_room_name)
          room.join(username, ws)
        when "users"
          ws.send({"type" => "users", "users" => room.users}.to_json)
        end
      rescue JSON::ParseException
        # ส่ง text ปกติ
        room.send_message(username, message)
      end
    end
    
    ws.on_close do
      room.leave(username)
    end
  end
  
  def start(port : Int32)
    server = HTTP::Server.new do |context|
      if context.request.headers.includes_word?("Upgrade", "websocket")
        HTTP::WebSocketHandler.new { |ws, ctx| handle_connection(ws, ctx) }.call(context)
      else
        # Serve chat client HTML
        context.response.content_type = "text/html"
        context.response.print chat_html
      end
    end
    
    puts "Chat Server on http://localhost:#{port}"
    server.listen("0.0.0.0", port)
  end
  
  private def chat_html
    <<-HTML
    <!DOCTYPE html>
    <html>
    <head><title>Crystal Chat</title></head>
    <body>
      <h1>Crystal WebSocket Chat</h1>
      <div id="messages" style="height:300px;overflow:auto;border:1px solid #ccc;padding:10px"></div>
      <input id="msg" type="text" placeholder="ข้อความ...">
      <button onclick="send()">ส่ง</button>
      <script>
        const ws = new WebSocket(`ws://${location.host}?username=User${Math.floor(Math.random()*1000)}&room=general`);
        ws.onmessage = (e) => {
          const data = JSON.parse(e.data);
          if (data.type === 'message' || data.type === 'join' || data.type === 'leave') {
            const div = document.getElementById('messages');
            div.innerHTML += `<p><b>${data.user}:</b> ${data.content}</p>`;
            div.scrollTop = div.scrollHeight;
          }
        };
        function send() {
          const input = document.getElementById('msg');
          ws.send(JSON.stringify({cmd: 'message', content: input.value}));
          input.value = '';
        }
        document.getElementById('msg').onkeypress = (e) => { if(e.key==='Enter') send(); };
      </script>
    </body>
    </html>
    HTML
  end
end

chat = ChatServer.new
chat.start(3000)
```

## WebSocket Client

```crystal
require "http/web_socket"

# WebSocket Client
ws = HTTP::WebSocket.new("localhost", "/chat?username=TestBot&room=general", port: 3000)

ws.on_message do |message|
  puts "Server: #{message}"
end

ws.on_close do
  puts "Connection closed"
end

# ส่งข้อความ
spawn do
  5.times do |i|
    sleep 1.second
    ws.send({"cmd" => "message", "content" => "สวัสดี #{i + 1}"}.to_json)
  end
  ws.close
end

ws.run
```

## Close Handshake

```crystal
require "http/server"

# จัดการ close handshake อย่างถูกต้อง
server = HTTP::Server.new do |context|
  next unless context.request.headers.includes_word?("Upgrade", "websocket")
  
  HTTP::WebSocketHandler.new do |ws|
    ws.on_message do |msg|
      if msg == "bye"
        # ส่ง close frame พร้อม status code
        ws.close(HTTP::WebSocket::CloseCode::NormalClosure, "Goodbye!")
      else
        ws.send("You said: #{msg}")
      end
    end
    
    ws.on_close do |code, reason|
      case code
      when .normal_closure?
        puts "Normal close: #{reason}"
      when .going_away?
        puts "Client going away"
      when .protocol_error?
        puts "Protocol error"
      when .invalid_data?
        puts "Invalid data received"
      else
        puts "Closed with code #{code}: #{reason}"
      end
    end
  end.call(context)
end

server.listen("0.0.0.0", 3000)
```

## Broadcast Server

```crystal
require "http/server"
require "json"

# Server ที่ broadcast ไปทุก clients
class BroadcastServer
  def initialize
    @clients = [] of HTTP::WebSocket
    @mutex = Mutex.new
  end
  
  def add_client(ws : HTTP::WebSocket)
    @mutex.synchronize { @clients << ws }
    puts "Client เพิ่ม, total: #{@clients.size}"
  end
  
  def remove_client(ws : HTTP::WebSocket)
    @mutex.synchronize { @clients.delete(ws) }
    puts "Client ลบ, total: #{@clients.size}"
  end
  
  def broadcast(message : String)
    clients = @mutex.synchronize { @clients.dup }
    failed = [] of HTTP::WebSocket
    
    clients.each do |ws|
      begin
        ws.send(message)
      rescue IO::Error
        failed << ws
      end
    end
    
    # ลบ failed clients
    @mutex.synchronize { failed.each { |ws| @clients.delete(ws) } }
  end
  
  def client_count
    @mutex.synchronize { @clients.size }
  end
end

broadcaster = BroadcastServer.new

# Server
server = HTTP::Server.new do |context|
  if context.request.headers.includes_word?("Upgrade", "websocket")
    HTTP::WebSocketHandler.new do |ws|
      broadcaster.add_client(ws)
      
      ws.on_message do |msg|
        # Broadcast ข้อความไปทุก clients
        broadcaster.broadcast({"from" => "client", "message" => msg}.to_json)
      end
      
      ws.on_close do
        broadcaster.remove_client(ws)
      end
    end.call(context)
  end
end

# Start broadcast loop
spawn do
  loop do
    sleep 5.seconds
    if broadcaster.client_count > 0
      broadcaster.broadcast({"type" => "ping", "clients" => broadcaster.client_count, "time" => Time.local.to_rfc3339}.to_json)
    end
  end
end

server.listen("0.0.0.0", 3000)
```

## Binary WebSocket Messages

```crystal
require "http/server"

# ส่ง binary data ผ่าน WebSocket
server = HTTP::Server.new do |context|
  next unless context.request.headers.includes_word?("Upgrade", "websocket")
  
  HTTP::WebSocketHandler.new do |ws|
    ws.on_binary do |bytes|
      puts "Received binary: #{bytes.size} bytes"
      puts "Hex: #{bytes.hexstring}"
      
      # Process binary data
      # ตัวอย่าง: protocol message
      if bytes.size >= 4
        msg_type = bytes[0]
        payload_size = IO::ByteFormat::BigEndian.decode(UInt16, bytes[1, 2])
        payload = bytes[3, payload_size] if bytes.size >= 3 + payload_size
        
        puts "Type: #{msg_type}, Size: #{payload_size}"
        
        # ส่ง ACK กลับ
        ws.stream(binary: true) do |io|
          io.write_byte(0xFF_u8)  # ACK
          io.write_byte(0x00_u8)
        end
      end
    end
    
    # ส่ง binary message ไปยัง client
    ws.stream(binary: true) do |io|
      io.write_byte(0x01_u8)   # message type
      io.write_bytes(5_u16, IO::ByteFormat::BigEndian)  # payload size
      io.print "Hello"  # payload
    end
  end.call(context)
end

server.listen("0.0.0.0", 3000)
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Collaborative Drawing

```crystal
require "http/server"
require "json"

# Real-time collaborative drawing server
class DrawingBoard
  struct DrawEvent
    include JSON::Serializable
    
    property type : String
    property x : Float64
    property y : Float64
    property color : String
    property size : Float64
    property user_id : String
    
    def initialize(@type, @x, @y, @color, @size, @user_id)
    end
  end
  
  def initialize
    @clients = {} of String => HTTP::WebSocket
    @events = [] of DrawEvent
    @mutex = Mutex.new
  end
  
  def handle(ws : HTTP::WebSocket)
    user_id = "user_#{Random::Secure.hex(4)}"
    
    @mutex.synchronize { @clients[user_id] = ws }
    
    # ส่ง canvas state ให้ user ใหม่
    history = @mutex.synchronize { @events.last(1000).dup }
    ws.send({"type" => "init", "events" => history, "user_id" => user_id}.to_json) rescue nil
    
    ws.on_message do |msg|
      event = DrawEvent.from_json(msg) rescue nil
      next unless event
      
      event_with_user = DrawEvent.new(event.type, event.x, event.y, event.color, event.size, user_id)
      
      @mutex.synchronize do
        @events << event_with_user
        @events.shift if @events.size > 10000
      end
      
      # Broadcast ไปทุก clients ยกเว้นผู้ส่ง
      clients = @mutex.synchronize { @clients.dup }
      clients.each do |uid, client_ws|
        next if uid == user_id
        client_ws.send(event_with_user.to_json) rescue nil
      end
    end
    
    ws.on_close do
      @mutex.synchronize { @clients.delete(user_id) }
    end
  end
end

board = DrawingBoard.new

server = HTTP::Server.new do |context|
  if context.request.headers.includes_word?("Upgrade", "websocket")
    HTTP::WebSocketHandler.new { |ws, _| board.handle(ws) }.call(context)
  else
    context.response.content_type = "text/plain"
    context.response.print "Drawing Board WS server"
  end
end

server.listen("0.0.0.0", 3001)
```

### แบบฝึกหัดที่ 2: Live Dashboard

```crystal
require "http/server"
require "json"

# Live dashboard ที่ส่ง metrics แบบ real-time
class MetricsDashboard
  def initialize
    @clients = [] of HTTP::WebSocket
    @mutex = Mutex.new
    @metrics = {
      "cpu" => 0.0,
      "memory" => 0.0,
      "requests" => 0,
      "errors" => 0,
    }
    
    start_metrics_generator
    start_broadcaster
  end
  
  def add_client(ws : HTTP::WebSocket)
    @mutex.synchronize { @clients << ws }
    
    # ส่ง current state ทันที
    ws.send({"type" => "snapshot", "metrics" => @metrics}.to_json) rescue nil
  end
  
  def remove_client(ws : HTTP::WebSocket)
    @mutex.synchronize { @clients.delete(ws) }
  end
  
  private def start_metrics_generator
    spawn do
      loop do
        sleep 1.second
        @mutex.synchronize do
          # จำลอง metrics
          @metrics["cpu"] = (@metrics["cpu"].as(Float64) + rand(-5.0..5.0)).clamp(0.0, 100.0)
          @metrics["memory"] = (@metrics["memory"].as(Float64) + rand(-2.0..2.0)).clamp(0.0, 100.0)
          @metrics["requests"] = @metrics["requests"].as(Int32) + rand(0..10)
          @metrics["errors"] = @metrics["errors"].as(Int32) + (rand < 0.1 ? 1 : 0)
        end
      end
    end
  end
  
  private def start_broadcaster
    spawn do
      loop do
        sleep 0.5.seconds
        
        clients = @mutex.synchronize { @clients.dup }
        next if clients.empty?
        
        metrics = @mutex.synchronize { @metrics.dup }
        msg = {"type" => "update", "metrics" => metrics, "time" => Time.local.to_rfc3339}.to_json
        
        clients.each do |ws|
          ws.send(msg) rescue remove_client(ws)
        end
      end
    end
  end
end

dashboard = MetricsDashboard.new

server = HTTP::Server.new do |context|
  if context.request.headers.includes_word?("Upgrade", "websocket")
    HTTP::WebSocketHandler.new do |ws|
      dashboard.add_client(ws)
      ws.on_close { dashboard.remove_client(ws) }
    end.call(context)
  end
end

puts "Dashboard WS Server on port 3002"
server.listen("0.0.0.0", 3002)
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **HTTP::WebSocket**: การสร้าง WebSocket connection
2. **HTTP::WebSocketHandler**: จัดการ WebSocket upgrade
3. **on_message**: รับ text messages
4. **on_binary**: รับ binary messages
5. **on_ping/on_pong**: heartbeat mechanism
6. **on_close**: จัดการ close event
7. **ws.send**: ส่ง text message
8. **ws.stream**: ส่ง binary message
9. **ws.close**: ปิด connection ด้วย close code
10. **Chat Application**: multi-room chat ด้วย WebSocket
11. **Broadcast**: ส่งข้อความไปทุก connected clients
12. **WebSocket Client**: crystal client สำหรับ connect หา WS server
