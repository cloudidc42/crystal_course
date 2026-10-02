# Part 86 - gRPC กับ F#

## บทนำ (Introduction)

gRPC คือ modern, high-performance RPC framework จาก Google ที่ใช้ HTTP/2 และ Protocol Buffers สำหรับ serialization

### ข้อดีของ gRPC เทียบกับ REST:
- **Performance**: Binary protocol เร็วกว่า JSON
- **Streaming**: Server/Client/Bidirectional streaming
- **Type Safety**: Generated code จาก .proto files
- **Multiplexing**: หลาย requests บน single connection

## 1. Protocol Buffers (.proto files)

```protobuf
// product.proto
syntax = "proto3";

option csharp_namespace = "ProductService.Grpc";

package product.v1;

// Service definition
service ProductCatalog {
  // Unary RPC
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  
  // Server-side streaming - client receives stream of responses
  rpc ListProducts (ListProductsRequest) returns (stream ProductResponse);
  
  // Client-side streaming - client sends stream of requests
  rpc BatchCheckStock (stream CheckStockRequest) returns (BatchStockResponse);
  
  // Bidirectional streaming
  rpc WatchPrices (stream WatchPriceRequest) returns (stream PriceUpdate);
}

// Messages
message GetProductRequest {
  string product_id = 1;
}

message ListProductsRequest {
  string category = 1;
  int32 page_size = 2;
  string page_token = 3;
  bool active_only = 4;
}

message CheckStockRequest {
  string product_id = 1;
  int32 requested_quantity = 2;
}

message BatchStockResponse {
  repeated StockCheckResult results = 1;
}

message StockCheckResult {
  string product_id = 1;
  bool available = 2;
  int32 current_stock = 3;
  string message = 4;
}

message WatchPriceRequest {
  string product_id = 1;
}

message PriceUpdate {
  string product_id = 1;
  double old_price = 2;
  double new_price = 3;
  google.protobuf.Timestamp updated_at = 4;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  string description = 3;
  double price = 4;
  int32 stock = 5;
  string category = 6;
  bool is_available = 7;
  repeated string tags = 8;
  map<string, string> attributes = 9;
}

// Status codes
enum ProductStatus {
  PRODUCT_STATUS_UNSPECIFIED = 0;
  PRODUCT_STATUS_ACTIVE = 1;
  PRODUCT_STATUS_DISCONTINUED = 2;
  PRODUCT_STATUS_OUT_OF_STOCK = 3;
}

// User service
// user.proto
syntax = "proto3";
option csharp_namespace = "UserService.Grpc";
package user.v1;

service UserService {
  rpc GetUser (GetUserRequest) returns (UserResponse);
  rpc CreateUser (CreateUserRequest) returns (UserResponse);
  rpc UpdateUser (UpdateUserRequest) returns (UserResponse);
  rpc DeleteUser (DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers (ListUsersRequest) returns (stream UserResponse);
}

message GetUserRequest {
  string user_id = 1;
}

message CreateUserRequest {
  string username = 1;
  string email = 2;
  string password = 3;
}

message UpdateUserRequest {
  string user_id = 1;
  optional string username = 2;
  optional string email = 3;
}

message DeleteUserRequest {
  string user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
  string role_filter = 3;
}

message UserResponse {
  string id = 1;
  string username = 2;
  string email = 3;
  string role = 4;
  bool is_active = 5;
}
```

## 2. Project Setup

```xml
<!-- ProductService.fsproj -->
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Grpc.AspNetCore" Version="2.59.0" />
    <PackageReference Include="Google.Protobuf" Version="3.25.1" />
    <PackageReference Include="Grpc.Tools" Version="2.59.0" PrivateAssets="All" />
  </ItemGroup>

  <ItemGroup>
    <!-- Proto files -->
    <Protobuf Include="Protos\product.proto" GrpcServices="Server" />
    <Protobuf Include="Protos\user.proto" GrpcServices="Client" />
  </ItemGroup>

  <ItemGroup>
    <Compile Include="Services\ProductGrpcService.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
</Project>
```

## 3. Unary RPC Implementation

```fsharp
// ===== ProductGrpcService.fs =====
namespace ProductService.Grpc

open System
open System.Threading.Tasks
open Grpc.Core
open ProductService.Grpc

// Data store (use real DB in production)
module ProductData =
    type Product = {
        Id: string
        Name: string
        Description: string
        Price: decimal
        Stock: int
        Category: string
        IsAvailable: bool
        Tags: string list
        Attributes: Map<string, string>
    }
    
    let private store = 
        [
            { Id = "P001"; Name = "Widget A"; Description = "High quality widget"; Price = 99.99m; 
              Stock = 100; Category = "Electronics"; IsAvailable = true; 
              Tags = ["popular"; "sale"]; Attributes = Map.ofList [("color", "red"); ("size", "medium")] }
            { Id = "P002"; Name = "Widget B"; Description = "Premium widget"; Price = 149.99m; 
              Stock = 50; Category = "Electronics"; IsAvailable = true; 
              Tags = ["new"]; Attributes = Map.ofList [("color", "blue"); ("size", "large")] }
            { Id = "P003"; Name = "Gadget X"; Description = "Advanced gadget"; Price = 299.99m; 
              Stock = 0; Category = "Electronics"; IsAvailable = false; 
              Tags = ["premium"]; Attributes = Map.empty }
        ] |> List.map (fun p -> p.Id, p) |> Map.ofList
    
    let findById id = store |> Map.tryFind id
    let findAll () = store |> Map.values |> Seq.toList
    let findByCategory cat = store |> Map.values |> Seq.filter (fun p -> p.Category = cat) |> Seq.toList

// Convert domain to proto response
let toProductResponse (p: ProductData.Product) =
    let response = ProductResponse(
        Id = p.Id,
        Name = p.Name,
        Description = p.Description,
        Price = float p.Price,
        Stock = p.Stock,
        Category = p.Category,
        IsAvailable = p.IsAvailable
    )
    for tag in p.Tags do
        response.Tags.Add(tag)
    for kvp in p.Attributes do
        response.Attributes.[kvp.Key] <- kvp.Value
    response

// gRPC Service Implementation
type ProductCatalogService() =
    inherit ProductCatalog.ProductCatalogBase()
    
    // Unary RPC: Get single product
    override _.GetProduct(request: GetProductRequest, context: ServerCallContext) : Task<ProductResponse> =
        task {
            match ProductData.findById request.ProductId with
            | None ->
                // Throw RpcException for proper gRPC error handling
                raise (RpcException(Status(StatusCode.NotFound, sprintf "Product %s not found" request.ProductId)))
                return ProductResponse()  // Unreachable but needed for type inference
            | Some product ->
                return toProductResponse product
        }
    
    // Server streaming: List products
    override _.ListProducts(request: ListProductsRequest, responseStream: IServerStreamWriter<ProductResponse>, context: ServerCallContext) : Task =
        task {
            let products = 
                if String.IsNullOrEmpty(request.Category) then
                    ProductData.findAll()
                else
                    ProductData.findByCategory request.Category
            
            let filtered =
                if request.ActiveOnly then
                    products |> List.filter (fun p -> p.IsAvailable)
                else
                    products
            
            let pageSize = if request.PageSize > 0 then request.PageSize else 10
            let page = filtered |> List.truncate pageSize
            
            for product in page do
                // Check if client cancelled
                if context.CancellationToken.IsCancellationRequested then
                    ()
                else
                    do! responseStream.WriteAsync(toProductResponse product)
                    // Simulate some processing time
                    do! Task.Delay(50)
        }
    
    // Client streaming: Batch check stock
    override _.BatchCheckStock(requestStream: IAsyncStreamReader<CheckStockRequest>, context: ServerCallContext) : Task<BatchStockResponse> =
        task {
            let results = ResizeArray<StockCheckResult>()
            
            let! hasNext = requestStream.MoveNext()
            let mutable continueReading = hasNext
            
            while continueReading do
                let request = requestStream.Current
                
                let result = 
                    match ProductData.findById request.ProductId with
                    | None ->
                        StockCheckResult(
                            ProductId = request.ProductId,
                            Available = false,
                            CurrentStock = 0,
                            Message = "Product not found"
                        )
                    | Some product ->
                        let available = product.IsAvailable && product.Stock >= request.RequestedQuantity
                        StockCheckResult(
                            ProductId = request.ProductId,
                            Available = available,
                            CurrentStock = product.Stock,
                            Message = if available then "In stock" 
                                      elif product.Stock = 0 then "Out of stock"
                                      else sprintf "Only %d available" product.Stock
                        )
                
                results.Add(result)
                
                let! next = requestStream.MoveNext()
                continueReading <- next
            
            let response = BatchStockResponse()
            for r in results do
                response.Results.Add(r)
            return response
        }
    
    // Bidirectional streaming: Watch prices
    override _.WatchPrices(requestStream: IAsyncStreamReader<WatchPriceRequest>, responseStream: IServerStreamWriter<PriceUpdate>, context: ServerCallContext) : Task =
        task {
            let watchedProducts = System.Collections.Generic.HashSet<string>()
            let random = Random()
            
            // Process incoming requests in background
            let processRequests = 
                task {
                    let! hasNext = requestStream.MoveNext()
                    let mutable continueReading = hasNext
                    while continueReading && not context.CancellationToken.IsCancellationRequested do
                        watchedProducts.Add(requestStream.Current.ProductId) |> ignore
                        let! next = requestStream.MoveNext()
                        continueReading <- next
                }
            
            // Simulate price updates
            let sendUpdates =
                task {
                    while not context.CancellationToken.IsCancellationRequested do
                        for productId in watchedProducts |> Seq.toList do
                            match ProductData.findById productId with
                            | Some product ->
                                // Simulate price change
                                let change = (random.NextDouble() - 0.5) * 10.0
                                let update = PriceUpdate(
                                    ProductId = productId,
                                    OldPrice = float product.Price,
                                    NewPrice = float product.Price + change
                                )
                                do! responseStream.WriteAsync(update)
                            | None -> ()
                        do! Task.Delay(1000)  // Update every second
                }
            
            do! Task.WhenAll(processRequests, sendUpdates)
        }
```

## 4. Interceptors (Middleware สำหรับ gRPC)

```fsharp
// ===== Authentication Interceptor =====
open Grpc.Core
open Grpc.Core.Interceptors
open Microsoft.AspNetCore.Authorization

// Server-side interceptor
type AuthenticationInterceptor(tokenValidator: string -> Result<{| UserId: string; Role: string |}, string>) =
    inherit Interceptor()
    
    let getToken (context: ServerCallContext) =
        let authorization = context.RequestHeaders.GetValue("authorization")
        if String.IsNullOrEmpty(authorization) then
            None
        elif authorization.StartsWith("Bearer ") then
            Some (authorization.[7..])
        else
            None
    
    override _.UnaryServerHandler<'TReq, 'TRes>(
        request: 'TReq, 
        context: ServerCallContext,
        continuation: UnaryServerMethod<'TReq, 'TRes>) =
        task {
            match getToken context with
            | None ->
                raise (RpcException(Status(StatusCode.Unauthenticated, "Authorization token required")))
                return Unchecked.defaultof<'TRes>
            | Some token ->
                match tokenValidator token with
                | Error msg ->
                    raise (RpcException(Status(StatusCode.Unauthenticated, msg)))
                    return Unchecked.defaultof<'TRes>
                | Ok claims ->
                    // Add claims to context
                    context.UserState.["UserId"] <- claims.UserId
                    context.UserState.["Role"] <- claims.Role
                    return! continuation.Invoke(request, context)
        }

// Logging Interceptor
type LoggingInterceptor(logger: string -> unit) =
    inherit Interceptor()
    
    override _.UnaryServerHandler<'TReq, 'TRes>(
        request: 'TReq,
        context: ServerCallContext,
        continuation: UnaryServerMethod<'TReq, 'TRes>) =
        task {
            let startTime = DateTime.UtcNow
            let methodName = context.Method
            
            try
                let! response = continuation.Invoke(request, context)
                let duration = (DateTime.UtcNow - startTime).TotalMilliseconds
                logger (sprintf "[gRPC] %s - OK (%.0fms)" methodName duration)
                return response
            with
            | :? RpcException as ex ->
                let duration = (DateTime.UtcNow - startTime).TotalMilliseconds
                logger (sprintf "[gRPC] %s - %A (%.0fms): %s" methodName ex.Status.StatusCode duration ex.Status.Detail)
                raise
            | ex ->
                let duration = (DateTime.UtcNow - startTime).TotalMilliseconds
                logger (sprintf "[gRPC] %s - INTERNAL (%.0fms): %s" methodName duration ex.Message)
                raise (RpcException(Status(StatusCode.Internal, "Internal server error")))
                return Unchecked.defaultof<'TRes>
        }

// Rate Limiting Interceptor
type RateLimitInterceptor(maxRequestsPerSecond: int) =
    inherit Interceptor()
    
    let requests = System.Collections.Concurrent.ConcurrentDictionary<string, ResizeArray<DateTime>>()
    
    let isRateLimited (clientId: string) =
        let now = DateTime.UtcNow
        let windowStart = now.AddSeconds(-1.0)
        
        let clientRequests = requests.GetOrAdd(clientId, fun _ -> ResizeArray<DateTime>())
        
        lock clientRequests (fun () ->
            // Remove old requests
            clientRequests.RemoveAll(fun t -> t < windowStart) |> ignore
            
            if clientRequests.Count >= maxRequestsPerSecond then
                true
            else
                clientRequests.Add(now)
                false)
    
    override _.UnaryServerHandler<'TReq, 'TRes>(
        request: 'TReq,
        context: ServerCallContext,
        continuation: UnaryServerMethod<'TReq, 'TRes>) =
        task {
            let clientId = context.Peer
            
            if isRateLimited clientId then
                raise (RpcException(Status(StatusCode.ResourceExhausted, "Rate limit exceeded. Try again later.")))
                return Unchecked.defaultof<'TRes>
            else
                return! continuation.Invoke(request, context)
        }
```

## 5. gRPC Server Setup (Program.fs)

```fsharp
// ===== Program.fs =====
open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Hosting
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Hosting

let builder = WebApplication.CreateBuilder()

// Add gRPC services
builder.Services.AddGrpc(fun opts ->
    opts.EnableDetailedErrors <- true
    opts.MaxReceiveMessageSize <- 16 * 1024 * 1024  // 16MB
    opts.MaxSendMessageSize <- 16 * 1024 * 1024) |> ignore

// Add gRPC reflection (for tools like grpcurl)
builder.Services.AddGrpcReflection() |> ignore

// Register interceptors
builder.Services.AddSingleton<LoggingInterceptor>(fun _ ->
    LoggingInterceptor(printfn "%s")) |> ignore

builder.Services.AddSingleton<RateLimitInterceptor>(fun _ ->
    RateLimitInterceptor(100)) |> ignore

// Register services
builder.Services.AddScoped<ProductCatalogService>() |> ignore

let app = builder.Build()

// Configure HTTP/2 for gRPC
app.UseRouting() |> ignore

// Map gRPC services
app.MapGrpcService<ProductCatalogService>() |> ignore
app.MapGrpcReflectionService() |> ignore

// Health check endpoint
app.MapGet("/health", fun () -> "OK") |> ignore

// app.Run("https://localhost:5001")
printfn "gRPC server configured"
```

## 6. gRPC Client Implementation

```fsharp
// ===== gRPC Client =====
// ProductServiceClient.fsproj requires:
// <PackageReference Include="Grpc.Net.Client" Version="2.59.0" />
// <Protobuf Include="Protos\product.proto" GrpcServices="Client" />

open Grpc.Net.Client
open Grpc.Core
open System
open System.Threading

type ProductServiceGrpcClient(serverAddress: string) =
    
    let createChannel () =
        let options = GrpcChannelOptions()
        options.MaxRetryAttempts <- 3
        options.HttpHandler <- new System.Net.Http.HttpClientHandler()
        GrpcChannel.ForAddress(serverAddress, options)
    
    let addAuthHeader (token: string) =
        Metadata()
        |> fun m -> m.Add("authorization", sprintf "Bearer %s" token); m
    
    // Unary call
    member _.GetProduct (productId: string) (token: string) =
        async {
            use channel = createChannel()
            let client = ProductCatalog.ProductCatalogClient(channel)
            let headers = addAuthHeader token
            
            try
                let! response = 
                    client.GetProductAsync(
                        GetProductRequest(ProductId = productId),
                        headers = headers,
                        deadline = DateTime.UtcNow.AddSeconds(5.0)
                    ).ResponseAsync |> Async.AwaitTask
                
                return Ok {|
                    Id = response.Id
                    Name = response.Name
                    Price = response.Price
                    Stock = response.Stock
                    IsAvailable = response.IsAvailable
                |}
            with
            | :? RpcException as ex when ex.StatusCode = StatusCode.NotFound ->
                return Error (sprintf "Product %s not found" productId)
            | :? RpcException as ex when ex.StatusCode = StatusCode.DeadlineExceeded ->
                return Error "Request timed out"
            | :? RpcException as ex ->
                return Error (sprintf "gRPC error: %A - %s" ex.StatusCode ex.Status.Detail)
        }
    
    // Server streaming
    member _.ListProducts (category: string) (pageSize: int) =
        async {
            use channel = createChannel()
            let client = ProductCatalog.ProductCatalogClient(channel)
            
            let request = ListProductsRequest(
                Category = category,
                PageSize = pageSize,
                ActiveOnly = true
            )
            
            use streamingCall = client.ListProducts(request)
            
            let products = ResizeArray()
            let mutable continueReading = true
            
            while continueReading do
                try
                    let! hasNext = streamingCall.ResponseStream.MoveNext(CancellationToken.None) |> Async.AwaitTask
                    if hasNext then
                        let product = streamingCall.ResponseStream.Current
                        products.Add({|
                            Id = product.Id
                            Name = product.Name
                            Price = product.Price
                        |})
                    else
                        continueReading <- false
                with _ ->
                    continueReading <- false
            
            return products |> Seq.toList
        }
    
    // Client streaming
    member _.BatchCheckStock (items: (string * int) list) =
        async {
            use channel = createChannel()
            let client = ProductCatalog.ProductCatalogClient(channel)
            
            use streamingCall = client.BatchCheckStock()
            
            // Send all requests
            for (productId, quantity) in items do
                do! streamingCall.RequestStream.WriteAsync(
                    CheckStockRequest(ProductId = productId, RequestedQuantity = quantity)
                ) |> Async.AwaitTask
            
            do! streamingCall.RequestStream.CompleteAsync() |> Async.AwaitTask
            
            // Get response
            let! response = streamingCall.ResponseAsync |> Async.AwaitTask
            
            return response.Results
                   |> Seq.map (fun r -> {|
                       ProductId = r.ProductId
                       Available = r.Available
                       Stock = r.CurrentStock
                       Message = r.Message
                   |})
                   |> Seq.toList
        }
    
    // Bidirectional streaming example
    member _.WatchPrices (productIds: string list) (onUpdate: {| ProductId: string; OldPrice: float; NewPrice: float |} -> unit) =
        async {
            use channel = createChannel()
            let client = ProductCatalog.ProductCatalogClient(channel)
            
            use cts = new CancellationTokenSource()
            use streamingCall = client.WatchPrices(cancellationToken = cts.Token)
            
            // Send product IDs to watch
            let sendRequests =
                async {
                    for productId in productIds do
                        do! streamingCall.RequestStream.WriteAsync(
                            WatchPriceRequest(ProductId = productId)
                        ) |> Async.AwaitTask
                }
            
            // Receive price updates
            let receiveUpdates =
                async {
                    try
                        let mutable continueReading = true
                        while continueReading do
                            let! hasNext = streamingCall.ResponseStream.MoveNext(cts.Token) |> Async.AwaitTask
                            if hasNext then
                                let update = streamingCall.ResponseStream.Current
                                onUpdate {|
                                    ProductId = update.ProductId
                                    OldPrice = update.OldPrice
                                    NewPrice = update.NewPrice
                                |}
                            else
                                continueReading <- false
                    with :? OperationCanceledException -> ()
                }
            
            // Run both concurrently for 5 seconds
            let timeout = async {
                do! Async.Sleep 5000
                cts.Cancel()
            }
            
            do! Async.Parallel [sendRequests; receiveUpdates; timeout] |> Async.Ignore
        }
```

## 7. REST vs gRPC Comparison

```fsharp
// ===== Comparison Example =====

// REST approach
module RestApproach =
    open System.Net.Http
    open System.Text.Json
    
    type ProductClient(baseUrl: string) =
        let client = new HttpClient(BaseAddress = Uri(baseUrl))
        
        member _.GetProduct productId =
            async {
                let! response = client.GetAsync(sprintf "/api/products/%s" productId) |> Async.AwaitTask
                if response.IsSuccessStatusCode then
                    let! json = response.Content.ReadAsStringAsync() |> Async.AwaitTask
                    let product = JsonSerializer.Deserialize<{| Id: string; Name: string; Price: float |}>(json)
                    return Ok product
                else
                    return Error (sprintf "HTTP %d" (int response.StatusCode))
            }

// gRPC approach - more code upfront but better in practice
module GrpcApproach =
    // Already shown above - client.GetProductAsync() is strongly typed
    // No JSON serialization/deserialization
    // No manual HTTP status code handling
    // Automatic retry, deadline, cancellation
    ()

// Performance comparison
(*
Benchmark (1000 requests):
- REST (JSON): ~250ms
- gRPC (Protobuf): ~80ms
- Savings: ~68%

Payload size comparison:
- REST: {"id":"P001","name":"Widget A","price":99.99} = 50 bytes
- gRPC: Binary protobuf = ~20 bytes (60% smaller)

When to use REST:
✓ Public APIs (browser-accessible)
✓ Simple CRUD operations
✓ When JSON is needed
✓ Wide tooling support

When to use gRPC:
✓ Internal microservices communication
✓ High-performance requirements
✓ Streaming needs
✓ Strongly-typed contracts
✓ Multiple languages in use
*)
```

## 8. Authentication ใน gRPC

```fsharp
// ===== JWT Authentication with gRPC =====

// Client: Add token to every request
let createAuthenticatedChannel (serverAddress: string) (tokenProvider: unit -> string) =
    let credentials = CallCredentials.FromInterceptor(fun context metadata _ ->
        let token = tokenProvider()
        metadata.Add("authorization", sprintf "Bearer %s" token)
        Threading.Tasks.Task.CompletedTask)
    
    // Combine with SSL credentials
    let compositeCredentials = ChannelCredentials.Create(SslCredentials(), credentials)
    GrpcChannel.ForAddress(serverAddress, GrpcChannelOptions(Credentials = compositeCredentials))

// Server: Validate token in interceptor
// (Already shown in AuthenticationInterceptor above)

// Per-call credentials (for different tokens per call)
let callWithToken (client: ProductCatalog.ProductCatalogClient) (token: string) (productId: string) =
    async {
        let headers = Metadata()
        headers.Add("authorization", sprintf "Bearer %s" token)
        
        let callOptions = CallOptions(headers = headers, deadline = Nullable(DateTime.UtcNow.AddSeconds(5.0)))
        
        let! response = 
            client.GetProductAsync(
                GetProductRequest(ProductId = productId),
                callOptions
            ).ResponseAsync |> Async.AwaitTask
        
        return response
    }
```

## 9. Error Handling ใน gRPC

```fsharp
// ===== gRPC Error Handling =====

// Status codes
(*
OK = 0              - Success
CANCELLED = 1       - Client cancelled
UNKNOWN = 2         - Unknown error
INVALID_ARGUMENT = 3  - Bad request
NOT_FOUND = 5       - Resource not found
ALREADY_EXISTS = 6  - Resource exists
PERMISSION_DENIED = 7 - No permission
RESOURCE_EXHAUSTED = 8 - Rate limit
FAILED_PRECONDITION = 9 - Bad state
UNIMPLEMENTED = 12  - Not implemented
INTERNAL = 13       - Server error
UNAVAILABLE = 14    - Service down
DEADLINE_EXCEEDED = 4 - Timeout
*)

// Server-side error handling
let throwNotFound (id: string) (entityType: string) =
    raise (RpcException(Status(StatusCode.NotFound, sprintf "%s with ID %s not found" entityType id)))

let throwInvalidArgument (message: string) =
    raise (RpcException(Status(StatusCode.InvalidArgument, message)))

let throwPermissionDenied () =
    raise (RpcException(Status(StatusCode.PermissionDenied, "Insufficient permissions")))

// With rich error details (using Google.Rpc.Status)
let throwWithDetails (code: StatusCode) (message: string) (details: string) =
    let status = Status(code, message)
    // Add error details to trailers
    raise (RpcException(status))

// Client-side error handling
let handleGrpcError (call: Async<'T>) =
    async {
        try
            return! call |> Async.map Ok
        with
        | :? RpcException as ex ->
            match ex.StatusCode with
            | StatusCode.NotFound ->
                return Error (sprintf "Not found: %s" ex.Status.Detail)
            | StatusCode.InvalidArgument ->
                return Error (sprintf "Invalid request: %s" ex.Status.Detail)
            | StatusCode.DeadlineExceeded ->
                return Error "Request timed out"
            | StatusCode.Unavailable ->
                return Error "Service temporarily unavailable"
            | StatusCode.Unauthenticated ->
                return Error "Authentication required"
            | StatusCode.PermissionDenied ->
                return Error "Permission denied"
            | _ ->
                return Error (sprintf "gRPC error (%A): %s" ex.StatusCode ex.Status.Detail)
        | ex ->
            return Error (sprintf "Unexpected error: %s" ex.Message)
    }
```

## 10. Complete Example: Order gRPC Service

```fsharp
// ===== Complete Order gRPC Service =====

// order.proto (conceptual):
(*
service OrderService {
  rpc CreateOrder (CreateOrderRequest) returns (OrderResponse);
  rpc GetOrder (GetOrderRequest) returns (OrderResponse);
  rpc ListOrders (ListOrdersRequest) returns (stream OrderResponse);
  rpc TrackOrder (TrackOrderRequest) returns (stream OrderStatusUpdate);
}
*)

// Simulated generated types
type CreateOrderRequest = { CustomerId: string; Items: (string * int) list; Total: decimal }
type OrderResponse = { Id: string; CustomerId: string; Status: string; Total: decimal; CreatedAt: DateTime }
type GetOrderRequest = { OrderId: string }
type ListOrdersRequest = { CustomerId: string; StatusFilter: string }
type TrackOrderRequest = { OrderId: string }
type OrderStatusUpdate = { OrderId: string; Status: string; Message: string; UpdatedAt: DateTime }

// Service implementation
type OrderGrpcService() =
    let orders = System.Collections.Concurrent.ConcurrentDictionary<string, OrderResponse>()
    let statusEvents = System.Collections.Concurrent.ConcurrentDictionary<string, ResizeArray<OrderStatusUpdate>>()
    
    // Simulate order status progression
    let simulateOrderProgression (orderId: string) =
        async {
            let statuses = [
                "Processing", "Order received and being processed"
                "PaymentConfirmed", "Payment has been confirmed"
                "Preparing", "Items being prepared for shipment"
                "Shipped", "Order has been shipped"
                "Delivered", "Order delivered successfully"
            ]
            
            for status, message in statuses do
                do! Async.Sleep 500  // Simulate each step taking time
                let update = { 
                    OrderId = orderId
                    Status = status
                    Message = message
                    UpdatedAt = DateTime.UtcNow 
                }
                
                let events = statusEvents.GetOrAdd(orderId, fun _ -> ResizeArray())
                events.Add(update)
                
                if orders.ContainsKey(orderId) then
                    orders.[orderId] <- { orders.[orderId] with Status = status }
        }
    
    member _.CreateOrder (request: CreateOrderRequest) =
        async {
            let orderId = Guid.NewGuid().ToString()
            let order = {
                Id = orderId
                CustomerId = request.CustomerId
                Status = "Created"
                Total = request.Total
                CreatedAt = DateTime.UtcNow
            }
            orders.[orderId] <- order
            statusEvents.[orderId] <- ResizeArray([{
                OrderId = orderId
                Status = "Created"
                Message = "Order created"
                UpdatedAt = DateTime.UtcNow
            }])
            
            // Start progression simulation
            Async.Start (simulateOrderProgression orderId)
            
            return Ok order
        }
    
    member _.GetOrder (orderId: string) =
        async {
            match orders.TryGetValue(orderId) with
            | true, order -> return Ok order
            | _ -> return Error (sprintf "Order %s not found" orderId)
        }
    
    member _.TrackOrder (orderId: string) (onUpdate: OrderStatusUpdate -> unit) =
        async {
            let mutable lastEventIndex = 0
            let startTime = DateTime.UtcNow
            let timeout = startTime.AddSeconds(30.0)
            
            while DateTime.UtcNow < timeout do
                let events = statusEvents.GetOrAdd(orderId, fun _ -> ResizeArray())
                
                while lastEventIndex < events.Count do
                    onUpdate events.[lastEventIndex]
                    lastEventIndex <- lastEventIndex + 1
                
                // Check if order is in final state
                match orders.TryGetValue(orderId) with
                | true, order when order.Status = "Delivered" ->
                    ()  // Done
                | _ ->
                    do! Async.Sleep 100  // Poll every 100ms
        }

// Usage example
let orderServiceDemo () =
    async {
        printfn "=== Order gRPC Service Demo ==="
        
        let service = OrderGrpcService()
        
        // Create an order
        let! createResult = service.CreateOrder {
            CustomerId = "CUST-001"
            Items = [("P001", 2); ("P002", 1)]
            Total = 349.97m
        }
        
        match createResult with
        | Error e -> printfn "Error creating order: %s" e
        | Ok order ->
        
        printfn "Order created: %s (Status: %s)" order.Id order.Status
        
        // Track the order (streaming updates)
        printfn "\nTracking order..."
        
        let trackTask =
            service.TrackOrder order.Id (fun update ->
                printfn "[%s] %s: %s" 
                    (update.UpdatedAt.ToString("HH:mm:ss"))
                    update.Status
                    update.Message)
        
        do! trackTask
        
        // Get final state
        let! finalResult = service.GetOrder order.Id
        match finalResult with
        | Ok final -> printfn "\nFinal status: %s" final.Status
        | Error e -> printfn "Error: %s" e
    }

orderServiceDemo() |> Async.RunSynchronously
```

## 11. สรุป gRPC กับ F#

```fsharp
(*
gRPC with F# Summary:

1. Protocol Buffers
   - Define .proto files with services and messages
   - Generate F#/C# code with Grpc.Tools
   - Versioning with message evolution

2. Service Types
   - Unary: Request/Response (like REST)
   - Server Streaming: Server sends multiple responses
   - Client Streaming: Client sends multiple requests
   - Bidirectional: Both sides stream

3. Interceptors
   - Authentication
   - Logging
   - Rate limiting
   - Error handling
   - Metrics collection

4. Error Handling
   - Use StatusCode for appropriate errors
   - RpcException carries status + detail
   - Rich error details with Google.Rpc.Status

5. Performance Tips
   - Use connection pooling (GrpcChannel is reusable)
   - Set appropriate deadlines
   - Use compression for large payloads
   - Consider load balancing

gRPC vs REST Summary:
- Use gRPC for: internal microservices, streaming, performance-critical
- Use REST for: public APIs, browser clients, simple CRUD
*)

printfn "gRPC with F# - Complete!"
printfn "Binary protocol, strongly-typed, high-performance RPC"
```

---

**สรุป**: gRPC ใน F# ให้ประสิทธิภาพสูงและ type safety ผ่าน Protocol Buffers ทำให้เหมาะสำหรับ internal microservice communication โดยเฉพาะเมื่อต้องการ streaming หรือ performance สูง
