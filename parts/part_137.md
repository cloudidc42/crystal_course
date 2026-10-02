# Part 137: Kemal WebSockets - การใช้งาน WebSockets ใน Kemal

## บทนำ

Kemal มี built-in WebSocket support ผ่าน `ws` route ช่วยให้สร้าง real-time applications ได้ง่าย

## WebSocket พื้นฐาน

```crystal
require "kemal"

# WebSocket endpoint
ws "/ws" do |socket|
  puts "Client connected"
  
  socket.on_message do |message|
    puts "Received: #{message}"
    socket.send("Echo: #{message}")
  end
  
  socket.on_close do
    puts "Client disconnected"
  end
end

get "/" do
  render "src/views/index.ecr"
end

Kemal.run
```

## WebSocket Events

```crystal
require "kemal"

ws "/events" do |socket|
  # เมื่อ client เชื่อมต่อ
  socket.send({"type" => "connected", "time" => Time.local.to_rfc3339}.to_json)
  
  # รับ text message
  socket.on_message do |message|
    puts "Text message: #{message}"
    
    begin
      data = JSON.parse(message)
      event_type = data["type"]?.try(&.as_s?) || "unknown"
      
      case event_type
      when "ping"
        socket.send({"type" => "pong", "time" => Time.local.to_rfc3339}.to_json)
      when "echo"
        socket.send({"type" => "echo", "data" => data["data"]?}.to_json)
      else
        socket.send({"type" => "error", "message" => "Unknown event type"}.to_json)
      end
    rescue JSON::ParseException
      socket.send({"type" => "error", "message" => "Invalid JSON"}.to_json)
    end
  end
  
  # รับ binary message
  socket.on_binary do |bytes|
    puts "Binary message: #{bytes.size} bytes"
    socket.send_binary(bytes)  # echo back
  end
  
  # เมื่อ connection ปิด
  socket.on_close do |code, message|
    puts "Disconnected: code=#{code}, reason=#{message}"
  end
end

Kemal.run
```

## Chat Room

```crystal
require "kemal"
require "json"

class ChatRoom
  struct Message
    include JSON::Serializable
    property type : String
    property username : String
    property text : String
    property time : String
    property room : String
    
    def initialize(@type, @username, @text, @room)
      @time = Time.local.to_rfc3339
    end
  end
  
  @@rooms = Hash(String, Array(HTTP::WebSocket)).new { |h, k| h[k] = [] of HTTP::WebSocket }
  @@usernames = {} of HTTP::WebSocket => String
  @@mutex = Mutex.new
  
  def self.join(socket : HTTP::WebSocket, room : String, username : String)
    @@mutex.synchronize do
      @@rooms[room] << socket
      @@usernames[socket] = username
    end
    
    # ประกาศว่ามีคนเข้าร่วม
    broadcast(room, Message.new("join", username, "#{username} เข้าร่วม #{room}", room))
  end
  
  def self.leave(socket : HTTP::WebSocket)
    @@mutex.synchronize do
      username = @@usernames.delete(socket) || "Unknown"
      
      @@rooms.each do |room, sockets|
        if sockets.includes?(socket)
          sockets.delete(socket)
          broadcast_raw(room, Message.new("leave", username, "#{username} ออกจาก #{room}", room).to_json, except: socket)
          break
        end
      end
    end
  end
  
  def self.send_message(socket : HTTP::WebSocket, room : String, text : String)
    username = @@mutex.synchronize { @@usernames[socket]? } || "Anonymous"
    message = Message.new("message", username, text, room)
    broadcast(room, message)
  end
  
  def self.broadcast(room : String, message : Message)
    broadcast_raw(room, message.to_json)
  end
  
  def self.broadcast_raw(room : String, json : String, except : HTTP::WebSocket? = nil)
    sockets = @@mutex.synchronize { @@rooms[room]?.try(&.dup) || [] of HTTP::WebSocket }
    
    sockets.each do |sock|
      next if sock == except
      sock.send(json) rescue nil
    end
  end
  
  def self.list_rooms : Array(String)
    @@mutex.synchronize { @@rooms.keys.select { |r| !@@rooms[r].empty? } }
  end
  
  def self.room_count(room : String) : Int32
    @@mutex.synchronize { @@rooms[room]?.try(&.size) || 0 }
  end
end

# WebSocket chat endpoint
ws "/chat/:room" do |socket, env|
  room = env.params.url["room"]
  username = env.params.query["username"]? || "Anonymous_#{Random.rand(1000)}"
  
  ChatRoom.join(socket, room, username)
  
  socket.on_message do |message|
    begin
      data = JSON.parse(message)
      text = data["text"]?.try(&.as_s?) || ""
      ChatRoom.send_message(socket, room, text) unless text.empty?
    rescue
      ChatRoom.send_message(socket, room, message)
    end
  end
  
  socket.on_close do
    ChatRoom.leave(socket)
  end
end

# REST API
get "/api/rooms" do |env|
  env.response.content_type = "application/json"
  {
    "rooms" => ChatRoom.list_rooms.map { |r| {"name" => r, "users" => ChatRoom.room_count(r)} }
  }.to_json
end

Kemal.run
```

## Heartbeat / Ping-Pong

```crystal
require "kemal"

class WebSocketManager
  @@connections = {} of String => HTTP::WebSocket
  @@mutex = Mutex.new
  
  def self.add(id : String, socket : HTTP::WebSocket)
    @@mutex.synchronize { @@connections[id] = socket }
    
    # Start heartbeat
    spawn do
      loop do
        sleep 30.seconds
        break unless alive?(id, socket)
        socket.ping("heartbeat") rescue break
      end
      remove(id)
    end
  end
  
  def self.remove(id : String)
    @@mutex.synchronize { @@connections.delete(id) }
  end
  
  def self.alive?(id : String, socket : HTTP::WebSocket) : Bool
    @@mutex.synchronize { @@connections[id]? == socket }
  end
  
  def self.broadcast(message : String)
    sockets = @@mutex.synchronize { @@connections.values.dup }
    sockets.each { |s| s.send(message) rescue nil }
  end
  
  def self.count : Int32
    @@mutex.synchronize { @@connections.size }
  end
end

ws "/live" do |socket|
  connection_id = Random::Secure.hex(16)
  WebSocketManager.add(connection_id, socket)
  
  socket.send({"id" => connection_id, "connected" => true}.to_json)
  
  socket.on_pong do |message|
    puts "Pong from #{connection_id}"
  end
  
  socket.on_message do |msg|
    # Echo
    socket.send(msg)
  end
  
  socket.on_close do
    WebSocketManager.remove(connection_id)
  end
end

# Broadcast endpoint
post "/api/broadcast" do |env|
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  message = data["message"]?.try(&.as_s?) || ""
  
  WebSocketManager.broadcast({"type" => "broadcast", "message" => message}.to_json)
  
  env.response.content_type = "application/json"
  {"success" => true, "sent_to" => WebSocketManager.count}.to_json
end

Kemal.run
```

## Live Notifications

```crystal
require "kemal"

class NotificationHub
  struct Notification
    include JSON::Serializable
    property id : String
    property type : String
    property title : String
    property message : String
    property timestamp : String
    property user_id : String?
    
    def initialize(@type, @title, @message, @user_id = nil)
      @id = Random::Secure.hex(8)
      @timestamp = Time.local.to_rfc3339
    end
  end
  
  @@user_sockets = Hash(String, Array(HTTP::WebSocket)).new { |h, k| h[k] = [] of HTTP::WebSocket }
  @@global_sockets = [] of HTTP::WebSocket
  @@mutex = Mutex.new
  
  def self.subscribe_user(user_id : String, socket : HTTP::WebSocket)
    @@mutex.synchronize { @@user_sockets[user_id] << socket }
  end
  
  def self.subscribe_global(socket : HTTP::WebSocket)
    @@mutex.synchronize { @@global_sockets << socket }
  end
  
  def self.unsubscribe(socket : HTTP::WebSocket)
    @@mutex.synchronize do
      @@global_sockets.delete(socket)
      @@user_sockets.each_value { |sockets| sockets.delete(socket) }
    end
  end
  
  def self.notify_user(user_id : String, notification : Notification)
    sockets = @@mutex.synchronize { @@user_sockets[user_id]?.try(&.dup) || [] of HTTP::WebSocket }
    sockets.each { |s| s.send(notification.to_json) rescue nil }
  end
  
  def self.notify_all(notification : Notification)
    sockets = @@mutex.synchronize { @@global_sockets.dup }
    sockets.each { |s| s.send(notification.to_json) rescue nil }
  end
end

# User-specific notifications
ws "/notifications/user/:user_id" do |socket, env|
  user_id = env.params.url["user_id"]
  NotificationHub.subscribe_user(user_id, socket)
  
  socket.send({"type" => "connected", "user_id" => user_id}.to_json)
  
  socket.on_close { NotificationHub.unsubscribe(socket) }
end

# Global notifications
ws "/notifications/global" do |socket|
  NotificationHub.subscribe_global(socket)
  
  socket.send({"type" => "connected", "scope" => "global"}.to_json)
  
  socket.on_close { NotificationHub.unsubscribe(socket) }
end

# Send notification via API
post "/api/notifications" do |env|
  body = env.request.body.try(&.gets_to_end) || "{}"
  data = JSON.parse(body)
  
  type = data["type"]?.try(&.as_s?) || "info"
  title = data["title"]?.try(&.as_s?) || ""
  message = data["message"]?.try(&.as_s?) || ""
  user_id = data["user_id"]?.try(&.as_s?)
  
  notification = NotificationHub::Notification.new(type, title, message, user_id)
  
  if user_id
    NotificationHub.notify_user(user_id, notification)
  else
    NotificationHub.notify_all(notification)
  end
  
  env.response.content_type = "application/json"
  {"success" => true, "notification_id" => notification.id}.to_json
end

Kemal.run
```

## HTML Client

```html
<!-- src/views/websocket_demo.ecr -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title>WebSocket Demo</title>
  <style>
    body { font-family: sans-serif; max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
    #messages { height: 400px; overflow-y: auto; border: 1px solid #ddd; padding: 1rem; background: #f9f9f9; }
    .message { margin-bottom: 0.5rem; padding: 0.5rem; border-radius: 4px; }
    .message.sent { background: #dbeafe; text-align: right; }
    .message.received { background: #dcfce7; }
    .message.system { background: #fef9c3; color: #854d0e; font-style: italic; }
    #input-area { display: flex; gap: 0.5rem; margin-top: 1rem; }
    #input-area input { flex: 1; padding: 0.5rem; }
    button { padding: 0.5rem 1rem; cursor: pointer; }
    #status { padding: 0.5rem; margin-bottom: 1rem; border-radius: 4px; }
    .connected { background: #dcfce7; color: #166534; }
    .disconnected { background: #fee2e2; color: #991b1b; }
  </style>
</head>
<body>
  <h1>WebSocket Chat</h1>
  
  <div id="status" class="disconnected">ยังไม่ได้เชื่อมต่อ</div>
  
  <div id="messages"></div>
  
  <div id="input-area">
    <input type="text" id="msg-input" placeholder="พิมพ์ข้อความ..." onkeypress="handleKey(event)">
    <button onclick="sendMessage()">ส่ง</button>
    <button onclick="connect()">เชื่อมต่อ</button>
    <button onclick="disconnect()">ตัดการเชื่อมต่อ</button>
  </div>

  <script>
    let ws = null;
    const username = "User_" + Math.floor(Math.random() * 1000);
    
    function connect() {
      const protocol = location.protocol === 'https:' ? 'wss:' : 'ws:';
      ws = new WebSocket(`${protocol}//${location.host}/chat/general?username=${username}`);
      
      ws.onopen = () => {
        updateStatus(true);
        addMessage('system', `เชื่อมต่อแล้วในฐานะ ${username}`);
      };
      
      ws.onmessage = (event) => {
        const data = JSON.parse(event.data);
        
        switch(data.type) {
          case 'message':
            const isSelf = data.username === username;
            addMessage(isSelf ? 'sent' : 'received', `${data.username}: ${data.text}`);
            break;
          case 'join':
            addMessage('system', data.text);
            break;
          case 'leave':
            addMessage('system', data.text);
            break;
        }
      };
      
      ws.onclose = () => {
        updateStatus(false);
        addMessage('system', 'ตัดการเชื่อมต่อแล้ว');
      };
      
      ws.onerror = (err) => {
        addMessage('system', 'เกิดข้อผิดพลาด: ' + err);
      };
    }
    
    function disconnect() {
      ws?.close();
    }
    
    function sendMessage() {
      const input = document.getElementById('msg-input');
      const text = input.value.trim();
      
      if (text && ws?.readyState === WebSocket.OPEN) {
        ws.send(JSON.stringify({ text }));
        input.value = '';
      }
    }
    
    function handleKey(event) {
      if (event.key === 'Enter') sendMessage();
    }
    
    function addMessage(type, text) {
      const div = document.createElement('div');
      div.className = `message ${type}`;
      div.textContent = text;
      
      const messages = document.getElementById('messages');
      messages.appendChild(div);
      messages.scrollTop = messages.scrollHeight;
    }
    
    function updateStatus(connected) {
      const status = document.getElementById('status');
      status.textContent = connected ? 'เชื่อมต่อแล้ว' : 'ยังไม่ได้เชื่อมต่อ';
      status.className = connected ? 'connected' : 'disconnected';
    }
    
    // Auto-connect
    connect();
  </script>
</body>
</html>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Collaborative Counter

```crystal
require "kemal"

# Real-time shared counter
counter_value = Atomic(Int32).new(0)
counter_sockets = [] of HTTP::WebSocket
counter_mutex = Mutex.new

ws "/counter" do |socket|
  # ส่งค่าปัจจุบัน
  socket.send({"value" => counter_value.get}.to_json)
  
  counter_mutex.synchronize { counter_sockets << socket }
  
  socket.on_message do |msg|
    data = JSON.parse(msg) rescue next
    action = data["action"]?.try(&.as_s?)
    
    new_value = case action
    when "increment" then counter_value.add(1)
    when "decrement" then counter_value.sub(1)
    when "reset"     then counter_value.set(0); 0
    else              counter_value.get
    end
    
    # Broadcast ให้ทุกคน
    broadcast_msg = {"value" => counter_value.get, "changed_by" => action}.to_json
    sockets = counter_mutex.synchronize { counter_sockets.dup }
    sockets.each { |s| s.send(broadcast_msg) rescue nil }
  end
  
  socket.on_close do
    counter_mutex.synchronize { counter_sockets.delete(socket) }
  end
end

get "/counter" do
  render "src/views/counter.ecr"
end

Kemal.run
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ws route**: สร้าง WebSocket endpoint
2. **socket.on_message**: รับ text messages
3. **socket.on_binary**: รับ binary messages
4. **socket.on_close**: จัดการ disconnection
5. **socket.send**: ส่ง text message
6. **socket.send_binary**: ส่ง binary message
7. **socket.ping/pong**: heartbeat
8. **Chat Room**: multi-room chat system
9. **Notification Hub**: user-specific และ broadcast
10. **HTML Client**: JavaScript WebSocket client
11. **Collaborative Apps**: shared state via WebSocket

WebSockets เหมาะสำหรับ real-time apps: chat, live updates, collaborative editing
