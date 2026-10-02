# Part 58 - WebSockets

## บทนำ

WebSocket เป็น protocol ที่ให้ full-duplex communication channel ผ่าน TCP connection เดียว ทำให้ server สามารถ push data ไปยัง client ได้โดยไม่ต้องรอ request จาก client เหมาะสำหรับ real-time applications เช่น chat, live notifications, dashboards

---

## 1. WebSocket Basics

```
HTTP vs WebSocket:

HTTP:
  Client -> Request -> Server
  Client <- Response <- Server
  (ต้องทำซ้ำทุกครั้ง)

WebSocket:
  Client -> Upgrade Request -> Server
  Client <-- Established connection --> Server
  Client <-> Bidirectional messages <-> Server
  (เปิดค้างไว้)

WebSocket Handshake:
  GET /ws HTTP/1.1
  Host: localhost:5000
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
  Sec-WebSocket-Version: 13
  
  HTTP/1.1 101 Switching Protocols
  Upgrade: websocket
  Connection: Upgrade
  Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

### Message Types

```fsharp
// WebSocket message types
type WebSocketMessageType =
    | Text      // UTF-8 text
    | Binary    // Binary data
    | Close     // Close connection
    | Ping      // Ping
    | Pong      // Pong (response to Ping)
```

---

## 2. ASP.NET Core WebSocket Middleware

```fsharp
// Program.fs
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection

let configureApp (app: IApplicationBuilder) =
    // Enable WebSocket support
    let wsOptions = WebSocketOptions(
        KeepAliveInterval = System.TimeSpan.FromSeconds(120),
        ReceiveBufferSize = 4096
    )
    
    app.UseWebSockets(wsOptions) |> ignore
    
    // Map WebSocket endpoints
    app.Map("/ws", fun ws ->
        ws.Run(WebSocketHandler.handleConnection)) |> ignore
    
    app.UseGiraffe(webApp)

// Configure services (no special services needed for basic WebSocket)
let configureServices (services: IServiceCollection) =
    services.AddGiraffe() |> ignore
    // WebSocket ไม่ต้องการ service registration พิเศษ
```

---

## 3. Accepting WebSocket Connections

```fsharp
open System.Net.WebSockets
open System.Threading
open Microsoft.AspNetCore.Http

// Accept connection handler
let handleConnection (ctx: HttpContext) =
    task {
        if ctx.WebSockets.IsWebSocketRequest then
            // Accept the WebSocket connection
            let! ws = ctx.WebSockets.AcceptWebSocketAsync()
            
            printfn $"WebSocket connected: {ctx.Connection.Id}"
            
            // Handle the connection
            do! processConnection ws ctx
            
            printfn $"WebSocket disconnected: {ctx.Connection.Id}"
        else
            ctx.Response.StatusCode <- 400
            do! ctx.Response.WriteAsync("WebSocket connections only")
    }

// Process connection lifecycle
let processConnection (ws: WebSocket) (ctx: HttpContext) =
    task {
        let buffer = Array.zeroCreate<byte> 4096
        
        // Send welcome message
        let welcomeMsg = System.Text.Encoding.UTF8.GetBytes("Welcome! You are connected.")
        do! ws.SendAsync(
            System.ArraySegment<byte>(welcomeMsg),
            WebSocketMessageType.Text,
            endOfMessage = true,
            cancellationToken = CancellationToken.None)
        
        // Message loop
        let mutable continueLoop = true
        while continueLoop && ws.State = WebSocketState.Open do
            let! result = ws.ReceiveAsync(
                System.ArraySegment<byte>(buffer),
                CancellationToken.None)
            
            match result.MessageType with
            | WebSocketMessageType.Text ->
                let message = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
                printfn $"Received: {message}"
                
                // Echo back
                let response = System.Text.Encoding.UTF8.GetBytes($"Echo: {message}")
                do! ws.SendAsync(
                    System.ArraySegment<byte>(response),
                    WebSocketMessageType.Text,
                    endOfMessage = true,
                    CancellationToken.None)
            
            | WebSocketMessageType.Binary ->
                printfn $"Received binary: {result.Count} bytes"
            
            | WebSocketMessageType.Close ->
                // Client initiated close
                printfn "Client requested close"
                do! ws.CloseAsync(
                    WebSocketCloseStatus.NormalClosure,
                    "Connection closed",
                    CancellationToken.None)
                continueLoop <- false
            
            | _ ->
                continueLoop <- false
    }
```

---

## 4. Sending and Receiving Messages

```fsharp
open System.Net.WebSockets
open System.Text
open System.Text.Json
open System.Threading

// Message wrapper type
type WsMessage = {
    Type: string
    Payload: System.Text.Json.JsonElement
    Timestamp: System.DateTime
    MessageId: string
}

// Send text message
let sendText (ws: WebSocket) (message: string) =
    task {
        if ws.State = WebSocketState.Open then
            let bytes = Encoding.UTF8.GetBytes(message)
            do! ws.SendAsync(
                System.ArraySegment<byte>(bytes),
                WebSocketMessageType.Text,
                endOfMessage = true,
                CancellationToken.None)
    }

// Send JSON message
let sendJson (ws: WebSocket) (data: 'T) =
    task {
        let json = JsonSerializer.Serialize(data)
        do! sendText ws json
    }

// Send message with type wrapper
let sendMessage (ws: WebSocket) (msgType: string) (payload: 'T) =
    task {
        let msg = {
            Type = msgType
            Payload = JsonSerializer.SerializeToElement(payload)
            Timestamp = System.DateTime.UtcNow
            MessageId = System.Guid.NewGuid().ToString("N")
        }
        do! sendJson ws msg
    }

// Receive complete message (handle fragmentation)
let receiveFullMessage (ws: WebSocket) (cancellationToken: CancellationToken) =
    task {
        use ms = new System.IO.MemoryStream()
        let buffer = Array.zeroCreate<byte> 4096
        let mutable result: WebSocketReceiveResult = null
        let mutable finished = false
        
        while not finished do
            let! r = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), cancellationToken)
            result <- r
            do! ms.WriteAsync(buffer, 0, r.Count)
            
            if r.EndOfMessage then
                finished <- true
        
        let bytes = ms.ToArray()
        return result, bytes
    }

// Receive typed message
let receiveMessage (ws: WebSocket) (cancellationToken: CancellationToken) =
    task {
        let! result, bytes = receiveFullMessage ws cancellationToken
        
        match result.MessageType with
        | WebSocketMessageType.Text ->
            let text = Encoding.UTF8.GetString(bytes)
            return Ok (text, false)
        | WebSocketMessageType.Binary ->
            return Ok (Encoding.UTF8.GetString(bytes), true)
        | WebSocketMessageType.Close ->
            return Error "Connection closed"
        | _ ->
            return Error "Unknown message type"
    }
```

---

## 5. Binary vs Text Messages

```fsharp
open System.Net.WebSockets
open System.IO

// Text message (UTF-8 JSON)
let sendTextMessage (ws: WebSocket) (data: obj) =
    task {
        let json = System.Text.Json.JsonSerializer.Serialize(data)
        let bytes = System.Text.Encoding.UTF8.GetBytes(json)
        do! ws.SendAsync(
            System.ArraySegment<byte>(bytes),
            WebSocketMessageType.Text,  // Text type
            true,
            System.Threading.CancellationToken.None)
    }

// Binary message (raw bytes)
let sendBinaryMessage (ws: WebSocket) (data: byte[]) =
    task {
        // ส่งทีละ chunk ถ้าข้อมูลใหญ่
        let chunkSize = 65536  // 64KB chunks
        let mutable offset = 0
        
        while offset < data.Length do
            let length = min chunkSize (data.Length - offset)
            let isLast = (offset + length) >= data.Length
            
            do! ws.SendAsync(
                System.ArraySegment<byte>(data, offset, length),
                WebSocketMessageType.Binary,  // Binary type
                endOfMessage = isLast,
                System.Threading.CancellationToken.None)
            
            offset <- offset + length
    }

// Send image file as binary
let sendImageFile (ws: WebSocket) (filePath: string) =
    task {
        let! bytes = File.ReadAllBytesAsync(filePath)
        
        // Send metadata first as text
        let metadata = {|
            filename = Path.GetFileName(filePath)
            size = bytes.Length
            mimeType = "image/jpeg"
        |}
        do! sendTextMessage ws metadata
        
        // Then send binary data
        do! sendBinaryMessage ws bytes
        
        printfn $"Sent image: {Path.GetFileName(filePath)} ({bytes.Length} bytes)"
    }

// Receive and distinguish message types
let receiveWithType (ws: WebSocket) =
    task {
        let buffer = Array.zeroCreate<byte> 65536
        let! result = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), System.Threading.CancellationToken.None)
        
        match result.MessageType with
        | WebSocketMessageType.Text ->
            let text = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
            return Choice1Of3 text
        | WebSocketMessageType.Binary ->
            let data = buffer.[0..result.Count-1]
            return Choice2Of3 data
        | _ ->
            return Choice3Of3 result.CloseStatus
    }
```

---

## 6. Connection Lifecycle

```fsharp
open System.Net.WebSockets
open System.Threading

// Connection state tracking
type ConnectionState = {
    ConnectionId: string
    UserId: int option
    ConnectedAt: System.DateTime
    LastActivity: System.DateTime
    mutable IsAlive: bool
}

// Connection lifecycle manager
type ConnectionManager() =
    let connections = System.Collections.Concurrent.ConcurrentDictionary<string, WebSocket * ConnectionState>()
    
    member _.Add (id: string) (ws: WebSocket) =
        let state = {
            ConnectionId = id
            UserId = None
            ConnectedAt = System.DateTime.UtcNow
            LastActivity = System.DateTime.UtcNow
            IsAlive = true
        }
        connections.[id] <- (ws, state)
        printfn $"Connection added: {id} (Total: {connections.Count})"
    
    member _.Remove (id: string) =
        connections.TryRemove(id) |> ignore
        printfn $"Connection removed: {id} (Total: {connections.Count})"
    
    member _.GetAll () = connections.Values |> Seq.map fst |> Seq.toList
    
    member _.Get (id: string) =
        match connections.TryGetValue(id) with
        | true, (ws, _) -> Some ws
        | _ -> None
    
    member _.Count = connections.Count
    
    member _.UpdateActivity (id: string) =
        match connections.TryGetValue(id) with
        | true, (ws, state) ->
            let newState = { state with LastActivity = System.DateTime.UtcNow }
            connections.[id] <- (ws, newState)
        | _ -> ()

// Full lifecycle handler
let handleFullLifecycle (manager: ConnectionManager) (ctx: Microsoft.AspNetCore.Http.HttpContext) =
    task {
        if ctx.WebSockets.IsWebSocketRequest then
            let! ws = ctx.WebSockets.AcceptWebSocketAsync()
            let id = ctx.Connection.Id
            
            // Add to manager
            manager.Add id ws
            
            try
                // Opening: send welcome
                do! sendJson ws {|
                    type' = "connected"
                    connectionId = id
                    timestamp = System.DateTime.UtcNow
                |}
                
                // Message loop
                let buffer = Array.zeroCreate<byte> 4096
                let mutable isConnected = true
                
                while isConnected && ws.State = WebSocketState.Open do
                    try
                        let! result = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), CancellationToken.None)
                        
                        manager.UpdateActivity id
                        
                        match result.MessageType with
                        | WebSocketMessageType.Text ->
                            let msg = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
                            // Process message...
                            printfn $"[{id}] Message: {msg}"
                            
                        | WebSocketMessageType.Close ->
                            isConnected <- false
                            printfn $"[{id}] Close requested"
                        
                        | _ -> ()
                    with
                    | :? System.OperationCanceledException ->
                        isConnected <- false
                    | ex ->
                        printfn $"[{id}] Error: {ex.Message}"
                        isConnected <- false
                
                // Graceful close
                if ws.State = WebSocketState.Open then
                    do! ws.CloseAsync(
                        WebSocketCloseStatus.NormalClosure,
                        "Server closing",
                        CancellationToken.None)
                
            finally
                // Always remove from manager
                manager.Remove id
        else
            ctx.Response.StatusCode <- 400
    }
```

---

## 7. Ping/Pong

```fsharp
open System.Net.WebSockets
open System.Threading
open System.Threading.Tasks

// WebSocket Ping/Pong mechanism
type PingPongManager(ws: WebSocket) =
    let mutable lastPingTime = System.DateTime.UtcNow
    let mutable lastPongTime = System.DateTime.UtcNow
    let mutable isAlive = true
    
    // Send ping
    member _.SendPing () =
        task {
            if ws.State = WebSocketState.Open then
                let pingData = System.Text.Encoding.UTF8.GetBytes($"ping-{System.DateTime.UtcNow.Ticks}")
                do! ws.SendAsync(
                    System.ArraySegment<byte>(pingData),
                    WebSocketMessageType.Text,  // ASP.NET Core ใช้ Text/Binary แทน Ping
                    true,
                    CancellationToken.None)
                lastPingTime <- System.DateTime.UtcNow
                printfn "Ping sent"
        }
    
    member _.ReceivedPong () =
        lastPongTime <- System.DateTime.UtcNow
        printfn "Pong received"
    
    member _.IsAlive = isAlive
    
    // Heartbeat task
    member this.StartHeartbeat (intervalSeconds: int) (timeoutSeconds: int) (cts: CancellationTokenSource) =
        task {
            while not cts.IsCancellationRequested && ws.State = WebSocketState.Open do
                do! Task.Delay(intervalSeconds * 1000, cts.Token)
                
                // Check if last pong was received within timeout
                let elapsed = (System.DateTime.UtcNow - lastPongTime).TotalSeconds
                if elapsed > float timeoutSeconds then
                    printfn "Connection timeout - no pong received"
                    isAlive <- false
                    cts.Cancel()
                else
                    do! this.SendPing()
        }

// Handle ping/pong in message loop
let handlePingPong (ws: WebSocket) (message: string) =
    task {
        if message.StartsWith("ping") then
            // Respond with pong
            let pong = message.Replace("ping", "pong")
            let bytes = System.Text.Encoding.UTF8.GetBytes(pong)
            do! ws.SendAsync(
                System.ArraySegment<byte>(bytes),
                WebSocketMessageType.Text,
                true,
                CancellationToken.None)
            return true
        else
            return false
    }

// Full connection with heartbeat
let handleWithHeartbeat (ws: WebSocket) =
    task {
        let cts = new CancellationTokenSource()
        let pingPong = PingPongManager(ws)
        
        // Start heartbeat in background
        let heartbeatTask = pingPong.StartHeartbeat 30 60 cts
        
        let buffer = Array.zeroCreate<byte> 4096
        let mutable running = true
        
        while running && ws.State = WebSocketState.Open && pingPong.IsAlive do
            let! result = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), cts.Token)
            
            match result.MessageType with
            | WebSocketMessageType.Text ->
                let msg = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
                let! isPong = handlePingPong ws msg
                if not isPong then
                    printfn $"Message: {msg}"
                    // Process regular message...
            
            | WebSocketMessageType.Close ->
                running <- false
            
            | _ -> ()
        
        cts.Cancel()
    }
```

---

## 8. Disconnect Handling

```fsharp
open System.Net.WebSockets
open System.Threading

type DisconnectReason =
    | NormalClose
    | ClientDisconnected
    | ServerError of exn
    | Timeout
    | NetworkError of exn

// Robust disconnect handling
let handleDisconnect (ws: WebSocket) (reason: DisconnectReason) =
    task {
        printfn $"Disconnecting: {reason}"
        
        match reason with
        | NormalClose ->
            if ws.State = WebSocketState.Open then
                do! ws.CloseAsync(
                    WebSocketCloseStatus.NormalClosure,
                    "Normal closure",
                    CancellationToken.None)
        
        | ClientDisconnected ->
            // Client already disconnected, just clean up
            printfn "Client disconnected unexpectedly"
        
        | ServerError ex ->
            printfn $"Server error: {ex.Message}"
            if ws.State = WebSocketState.Open then
                do! ws.CloseAsync(
                    WebSocketCloseStatus.InternalServerError,
                    "Server error",
                    CancellationToken.None)
        
        | Timeout ->
            if ws.State = WebSocketState.Open then
                do! ws.CloseAsync(
                    WebSocketCloseStatus.PolicyViolation,
                    "Connection timeout",
                    CancellationToken.None)
        
        | NetworkError ex ->
            printfn $"Network error: {ex.Message}"
            // Nothing to do, connection is already broken
    }

// Connection wrapper with auto-disconnect handling
let safeHandle (ws: WebSocket) (handler: WebSocket -> Task) =
    task {
        try
            do! handler ws
            do! handleDisconnect ws NormalClose
        with
        | :? WebSocketException as ex ->
            do! handleDisconnect ws (NetworkError ex)
        | :? System.OperationCanceledException ->
            do! handleDisconnect ws ClientDisconnected
        | ex ->
            do! handleDisconnect ws (ServerError ex)
    }
```

---

## 9. Broadcasting to Multiple Clients

```fsharp
open System.Net.WebSockets
open System.Collections.Concurrent
open System.Text
open System.Threading
open System.Threading.Tasks

// Connection hub for broadcasting
type ConnectionHub() =
    let connections = ConcurrentDictionary<string, WebSocket>()
    let subscriptions = ConcurrentDictionary<string, ResizeArray<string>>()  // channel -> connection IDs
    
    member _.AddConnection (id: string) (ws: WebSocket) =
        connections.[id] <- ws
        printfn $"Hub: Added {id} (Total: {connections.Count})"
    
    member _.RemoveConnection (id: string) =
        connections.TryRemove(id) |> ignore
        // Remove from all subscriptions
        for kvp in subscriptions do
            kvp.Value.Remove(id) |> ignore
        printfn $"Hub: Removed {id} (Total: {connections.Count})"
    
    member _.Subscribe (connectionId: string) (channel: string) =
        let subs = subscriptions.GetOrAdd(channel, fun _ -> ResizeArray<string>())
        if not (subs.Contains(connectionId)) then
            subs.Add(connectionId)
        printfn $"Hub: {connectionId} subscribed to {channel}"
    
    member _.Unsubscribe (connectionId: string) (channel: string) =
        match subscriptions.TryGetValue(channel) with
        | true, subs -> subs.Remove(connectionId) |> ignore
        | _ -> ()
    
    // Broadcast to all connections
    member _.BroadcastAll (message: string) =
        task {
            let bytes = Encoding.UTF8.GetBytes(message)
            let tasks =
                connections.Values
                |> Seq.filter (fun ws -> ws.State = WebSocketState.Open)
                |> Seq.map (fun ws ->
                    ws.SendAsync(
                        System.ArraySegment<byte>(bytes),
                        WebSocketMessageType.Text,
                        true,
                        CancellationToken.None)
                    :> Task)
                |> Seq.toArray
            
            do! Task.WhenAll(tasks)
        }
    
    // Broadcast to specific channel subscribers
    member _.BroadcastChannel (channel: string) (message: string) =
        task {
            match subscriptions.TryGetValue(channel) with
            | false, _ -> ()
            | true, subscriberIds ->
                let bytes = Encoding.UTF8.GetBytes(message)
                let tasks =
                    subscriberIds.ToArray()
                    |> Array.choose (fun id ->
                        match connections.TryGetValue(id) with
                        | true, ws when ws.State = WebSocketState.Open -> Some ws
                        | _ -> None)
                    |> Array.map (fun ws ->
                        ws.SendAsync(
                            System.ArraySegment<byte>(bytes),
                            WebSocketMessageType.Text,
                            true,
                            CancellationToken.None)
                        :> Task)
                
                do! Task.WhenAll(tasks)
        }
    
    // Send to specific connection
    member _.SendTo (connectionId: string) (message: string) =
        task {
            match connections.TryGetValue(connectionId) with
            | false, _ -> ()
            | true, ws when ws.State = WebSocketState.Open ->
                let bytes = Encoding.UTF8.GetBytes(message)
                do! ws.SendAsync(
                    System.ArraySegment<byte>(bytes),
                    WebSocketMessageType.Text,
                    true,
                    CancellationToken.None)
            | _ -> ()
        }
    
    member _.ConnectionCount = connections.Count
    member _.ChannelCount (channel: string) =
        match subscriptions.TryGetValue(channel) with
        | true, subs -> subs.Count
        | _ -> 0
```

---

## 10. Chat Application Example

```fsharp
// Full chat application

// Message types
type ChatMessageType =
    | Join
    | Leave
    | Message
    | DirectMessage
    | SystemNotice

type ChatMessage = {
    Type: string
    From: string
    To: string option
    Content: string
    Room: string
    Timestamp: System.DateTime
    MessageId: string
}

// Chat hub
type ChatHub() =
    inherit ConnectionHub()
    
    let usernames = System.Collections.Concurrent.ConcurrentDictionary<string, string>()
    
    member this.SetUsername (connectionId: string) (username: string) =
        usernames.[connectionId] <- username
    
    member this.GetUsername (connectionId: string) =
        match usernames.TryGetValue(connectionId) with
        | true, name -> Some name
        | _ -> None
    
    member _.RemoveUser (connectionId: string) =
        usernames.TryRemove(connectionId) |> ignore
    
    member this.GetAllUsers () =
        usernames.Values |> Seq.toList
    
    member this.BroadcastChatMessage (msg: ChatMessage) =
        task {
            let json = System.Text.Json.JsonSerializer.Serialize(msg)
            do! this.BroadcastAll json
        }
    
    member this.SendToUser (username: string) (msg: ChatMessage) =
        task {
            let targetId = 
                usernames
                |> Seq.tryFind (fun kvp -> kvp.Value = username)
                |> Option.map (fun kvp -> kvp.Key)
            
            match targetId with
            | None -> ()
            | Some id ->
                let json = System.Text.Json.JsonSerializer.Serialize(msg)
                do! this.SendTo id json
        }

let chatHub = ChatHub()

// Chat WebSocket handler
let handleChatConnection (ctx: Microsoft.AspNetCore.Http.HttpContext) =
    task {
        if ctx.WebSockets.IsWebSocketRequest then
            let! ws = ctx.WebSockets.AcceptWebSocketAsync()
            let connectionId = ctx.Connection.Id
            
            chatHub.AddConnection connectionId ws
            
            let mutable username = $"User_{connectionId[..7]}"
            chatHub.SetUsername connectionId username
            
            // Notify all users
            do! chatHub.BroadcastChatMessage {
                Type = "join"
                From = "system"
                To = None
                Content = $"{username} joined the chat"
                Room = "general"
                Timestamp = System.DateTime.UtcNow
                MessageId = System.Guid.NewGuid().ToString("N")
            }
            
            // Send current users list
            let usersMsg = {
                Type = "userList"
                From = "system"
                To = Some connectionId
                Content = System.Text.Json.JsonSerializer.Serialize(chatHub.GetAllUsers())
                Room = "general"
                Timestamp = System.DateTime.UtcNow
                MessageId = System.Guid.NewGuid().ToString("N")
            }
            do! chatHub.SendTo connectionId (System.Text.Json.JsonSerializer.Serialize(usersMsg))
            
            // Message loop
            let buffer = Array.zeroCreate<byte> 4096
            let mutable running = true
            
            while running && ws.State = WebSocketState.Open do
                try
                    let! result = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), System.Threading.CancellationToken.None)
                    
                    match result.MessageType with
                    | WebSocketMessageType.Text ->
                        let text = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
                        
                        // Parse incoming message
                        try
                            let incoming = System.Text.Json.JsonSerializer.Deserialize<{| type': string; content: string; to': string option |}>(text)
                            
                            match incoming.``type'`` with
                            | "setName" ->
                                let oldName = username
                                username <- incoming.content
                                chatHub.SetUsername connectionId username
                                
                                do! chatHub.BroadcastChatMessage {
                                    Type = "rename"
                                    From = "system"
                                    To = None
                                    Content = $"{oldName} changed name to {username}"
                                    Room = "general"
                                    Timestamp = System.DateTime.UtcNow
                                    MessageId = System.Guid.NewGuid().ToString("N")
                                }
                            
                            | "message" ->
                                do! chatHub.BroadcastChatMessage {
                                    Type = "message"
                                    From = username
                                    To = None
                                    Content = incoming.content
                                    Room = "general"
                                    Timestamp = System.DateTime.UtcNow
                                    MessageId = System.Guid.NewGuid().ToString("N")
                                }
                            
                            | "dm" ->
                                match incoming.``to'`` with
                                | Some targetUser ->
                                    do! chatHub.SendToUser targetUser {
                                        Type = "dm"
                                        From = username
                                        To = Some targetUser
                                        Content = incoming.content
                                        Room = "dm"
                                        Timestamp = System.DateTime.UtcNow
                                        MessageId = System.Guid.NewGuid().ToString("N")
                                    }
                                | None -> ()
                            
                            | _ -> ()
                        with ex ->
                            printfn $"Parse error: {ex.Message}"
                    
                    | WebSocketMessageType.Close ->
                        running <- false
                    
                    | _ -> ()
                with ex ->
                    printfn $"WebSocket error: {ex.Message}"
                    running <- false
            
            // Cleanup
            chatHub.RemoveUser connectionId
            chatHub.RemoveConnection connectionId
            
            do! chatHub.BroadcastChatMessage {
                Type = "leave"
                From = "system"
                To = None
                Content = $"{username} left the chat"
                Room = "general"
                Timestamp = System.DateTime.UtcNow
                MessageId = System.Guid.NewGuid().ToString("N")
            }
        else
            ctx.Response.StatusCode <- 400
    }
```

---

## 11. WebSocket with Giraffe

```fsharp
open Giraffe
open Microsoft.AspNetCore.Http
open System.Net.WebSockets

// Giraffe WebSocket handler
let wsHandler : HttpHandler =
    fun next ctx ->
        task {
            if ctx.WebSockets.IsWebSocketRequest then
                let! ws = ctx.WebSockets.AcceptWebSocketAsync()
                do! processWebSocket ws ctx
                return! next ctx
            else
                return! (setStatusCode 400 >=> text "WebSocket connections only") next ctx
        }

let processWebSocket (ws: WebSocket) (ctx: HttpContext) =
    task {
        let buffer = Array.zeroCreate<byte> 4096
        
        while ws.State = WebSocketState.Open do
            let! result = ws.ReceiveAsync(System.ArraySegment<byte>(buffer), System.Threading.CancellationToken.None)
            
            if result.MessageType = WebSocketMessageType.Text then
                let msg = System.Text.Encoding.UTF8.GetString(buffer, 0, result.Count)
                let response = System.Text.Encoding.UTF8.GetBytes($"Giraffe received: {msg}")
                do! ws.SendAsync(System.ArraySegment<byte>(response), WebSocketMessageType.Text, true, System.Threading.CancellationToken.None)
            elif result.MessageType = WebSocketMessageType.Close then
                do! ws.CloseAsync(WebSocketCloseStatus.NormalClosure, "Closing", System.Threading.CancellationToken.None)
    }

// Register WebSocket route in Giraffe
let webApp : HttpHandler =
    choose [
        route "/ws" >=> wsHandler
        route "/chat" >=> wsHandler
        GET >=> route "/" >=> text "WebSocket server"
    ]

// Use with Giraffe app
let configureApp (app: Microsoft.AspNetCore.Builder.IApplicationBuilder) =
    app.UseWebSockets() |> ignore
    app.UseGiraffe(webApp)
```

---

## 12. SignalR Alternative

SignalR เป็น higher-level abstraction สำหรับ real-time communication:

```bash
dotnet add package Microsoft.AspNetCore.SignalR
```

```fsharp
open Microsoft.AspNetCore.SignalR
open System.Threading.Tasks

// SignalR Hub
type NotificationHub() =
    inherit Hub()
    
    // Client calls this on server
    override this.SendMessage(user: string, message: string) =
        // Broadcast to all clients
        this.Clients.All.SendAsync("ReceiveMessage", user, message)
    
    // Send to specific user
    override this.SendToUser(userId: string, message: string) =
        this.Clients.User(userId).SendAsync("ReceiveMessage", "Server", message)
    
    // Join group (room/channel)
    member this.JoinGroup(groupName: string) =
        task {
            do! this.Groups.AddToGroupAsync(this.Context.ConnectionId, groupName)
            return! this.Clients.Group(groupName).SendAsync("UserJoined", this.Context.ConnectionId)
        }
    
    // Leave group
    member this.LeaveGroup(groupName: string) =
        task {
            do! this.Groups.RemoveFromGroupAsync(this.Context.ConnectionId, groupName)
            return! this.Clients.Group(groupName).SendAsync("UserLeft", this.Context.ConnectionId)
        }
    
    // Broadcast to group
    member this.SendToGroup(groupName: string, message: string) =
        this.Clients.Group(groupName).SendAsync("ReceiveMessage", "Server", message)
    
    // Connection events
    override this.OnConnectedAsync() =
        printfn $"Connected: {this.Context.ConnectionId}"
        base.OnConnectedAsync()
    
    override this.OnDisconnectedAsync(exception: exn) =
        printfn $"Disconnected: {this.Context.ConnectionId}"
        base.OnDisconnectedAsync(exception)

// Configure SignalR
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Builder

let configureSignalR (services: IServiceCollection) =
    services.AddSignalR(fun opts ->
        opts.EnableDetailedErrors <- true
        opts.KeepAliveInterval <- System.TimeSpan.FromSeconds(15)
        opts.ClientTimeoutInterval <- System.TimeSpan.FromSeconds(30))
    |> ignore

let configureSignalRApp (app: IApplicationBuilder) =
    app.UseRouting() |> ignore
    app.UseEndpoints(fun endpoints ->
        endpoints.MapHub<NotificationHub>("/hubs/notifications") |> ignore) |> ignore

// Send from anywhere using IHubContext
type NotificationService(hubContext: IHubContext<NotificationHub>) =
    member _.SendToAll(message: string) =
        hubContext.Clients.All.SendAsync("ReceiveNotification", message)
    
    member _.SendToUser(userId: string, message: string) =
        hubContext.Clients.User(userId).SendAsync("ReceiveNotification", message)
    
    member _.SendToGroup(group: string, message: string) =
        hubContext.Clients.Group(group).SendAsync("ReceiveNotification", message)
```

---

## สรุป

WebSocket ใน F# / ASP.NET Core:

1. **Setup**: ต้องเรียก `UseWebSockets()` ก่อน
2. **Accept**: ตรวจสอบ `IsWebSocketRequest` แล้วเรียก `AcceptWebSocketAsync()`
3. **Message loop**: รับ-ส่ง messages ใน loop จนกว่าจะ Close
4. **Broadcast**: ใช้ `ConnectionHub` หรือ library เช่น SignalR
5. **Heartbeat**: ส่ง ping/pong เพื่อ detect dead connections
6. **Cleanup**: remove connections เมื่อ disconnect

เลือก WebSocket เมื่อ:
- ต้องการ full bidirectional communication
- Low latency สำคัญ
- Frequent small messages
- Custom protocol

เลือก SignalR เมื่อ:
- ต้องการ abstraction สูง
- Auto-reconnection
- Fallback mechanisms
- Group/user targeting ง่าย
