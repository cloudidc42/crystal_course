# Part 84 - Microservices กับ F#

## บทนำ (Introduction)

Microservices architecture คือการแบ่งแอปพลิเคชันออกเป็น services ขนาดเล็กที่ทำงานอิสระจากกัน แต่ละ service มี:
- Business capability ของตัวเอง
- Database ของตัวเอง (Database per Service)
- การสื่อสารผ่าน well-defined APIs

## 1. โครงสร้าง Microservices

```
order-service/          ← จัดการ orders
user-service/           ← จัดการ users/auth
product-service/        ← จัดการ products/inventory
notification-service/   ← ส่ง emails/SMS
payment-service/        ← จัดการ payments
api-gateway/           ← Entry point สำหรับ clients
```

## 2. Service Communication: REST

```fsharp
// ===== Product Service =====
// Product.Service/Program.fs

open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection
open System

// Domain Types
type ProductId = Guid
type Product = {
    Id: ProductId
    Name: string
    Price: decimal
    Stock: int
    IsAvailable: bool
}

// In-memory storage (use real DB in production)
let private productStore = System.Collections.Concurrent.ConcurrentDictionary<ProductId, Product>()

// Seed data
let seedProducts () =
    let products = [
        { Id = Guid.Parse("11111111-0000-0000-0000-000000000001"); Name = "Widget A"; Price = 99.99m; Stock = 100; IsAvailable = true }
        { Id = Guid.Parse("11111111-0000-0000-0000-000000000002"); Name = "Widget B"; Price = 149.99m; Stock = 50; IsAvailable = true }
        { Id = Guid.Parse("11111111-0000-0000-0000-000000000003"); Name = "Gadget X"; Price = 299.99m; Stock = 0; IsAvailable = false }
    ]
    for p in products do
        productStore.[p.Id] <- p

// DTOs
type ProductDto = {
    Id: string
    Name: string
    Price: decimal
    Stock: int
    IsAvailable: bool
}

type CheckStockRequest = {
    ProductId: string
    Quantity: int
}

type CheckStockResponse = {
    ProductId: string
    Available: bool
    CurrentStock: int
}

// Minimal API endpoints
let configureProductEndpoints (app: WebApplication) =
    
    app.MapGet("/api/products", fun () ->
        let products = productStore.Values |> Seq.map (fun p -> {
            Id = p.Id.ToString()
            Name = p.Name
            Price = p.Price
            Stock = p.Stock
            IsAvailable = p.IsAvailable
        }) |> Seq.toList
        products) |> ignore
    
    app.MapGet("/api/products/{id}", fun (id: string) ->
        match Guid.TryParse(id) with
        | false, _ -> 
            Microsoft.AspNetCore.Http.Results.BadRequest("Invalid ID")
        | true, guid ->
            match productStore.TryGetValue(guid) with
            | true, p ->
                let dto = { Id = p.Id.ToString(); Name = p.Name; Price = p.Price; Stock = p.Stock; IsAvailable = p.IsAvailable }
                Microsoft.AspNetCore.Http.Results.Ok(dto)
            | _ ->
                Microsoft.AspNetCore.Http.Results.NotFound()) |> ignore
    
    app.MapPost("/api/products/check-stock", fun (req: CheckStockRequest) ->
        match Guid.TryParse(req.ProductId) with
        | false, _ -> 
            Microsoft.AspNetCore.Http.Results.BadRequest("Invalid product ID")
        | true, guid ->
            match productStore.TryGetValue(guid) with
            | true, p ->
                let response = {
                    ProductId = req.ProductId
                    Available = p.IsAvailable && p.Stock >= req.Quantity
                    CurrentStock = p.Stock
                }
                Microsoft.AspNetCore.Http.Results.Ok(response)
            | _ ->
                Microsoft.AspNetCore.Http.Results.NotFound()) |> ignore

let builder = WebApplication.CreateBuilder()
builder.Services.AddEndpointsApiExplorer() |> ignore
builder.Services.AddSwaggerGen() |> ignore

let app = builder.Build()
app.UseSwagger() |> ignore
app.UseSwaggerUI() |> ignore
configureProductEndpoints app
seedProducts()
// app.Run("http://localhost:5001")
```

## 3. HTTP Client สำหรับ Service-to-Service Communication

```fsharp
// ===== HTTP Client for inter-service calls =====

open System.Net.Http
open System.Text.Json

// Product Service Client
type ProductServiceClient(httpClient: HttpClient) =
    
    let deserialize<'T> (json: string) =
        JsonSerializer.Deserialize<'T>(json, JsonSerializerOptions(PropertyNameCaseInsensitive = true))
    
    member _.GetProduct (productId: string) =
        async {
            try
                let! response = httpClient.GetAsync(sprintf "/api/products/%s" productId) |> Async.AwaitTask
                if response.IsSuccessStatusCode then
                    let! content = response.Content.ReadAsStringAsync() |> Async.AwaitTask
                    let product = deserialize<{| Id: string; Name: string; Price: decimal; Stock: int |}> content
                    return Ok (Some product)
                elif response.StatusCode = System.Net.HttpStatusCode.NotFound then
                    return Ok None
                else
                    return Error (sprintf "HTTP %d" (int response.StatusCode))
            with ex ->
                return Error (sprintf "Connection error: %s" ex.Message)
        }
    
    member _.CheckStock (productId: string) (quantity: int) =
        async {
            try
                let request = {| ProductId = productId; Quantity = quantity |}
                let json = JsonSerializer.Serialize(request)
                let content = new StringContent(json, System.Text.Encoding.UTF8, "application/json")
                let! response = httpClient.PostAsync("/api/products/check-stock", content) |> Async.AwaitTask
                if response.IsSuccessStatusCode then
                    let! body = response.Content.ReadAsStringAsync() |> Async.AwaitTask
                    let result = deserialize<{| Available: bool; CurrentStock: int |}> body
                    return Ok result.Available
                else
                    return Error "Failed to check stock"
            with ex ->
                return Error (sprintf "Connection error: %s" ex.Message)
        }

// Order Service that calls Product Service
type OrderService(productClient: ProductServiceClient) =
    
    member _.CreateOrder (items: (string * int) list) =
        async {
            // Check stock for all items
            let! stockChecks =
                items
                |> List.map (fun (productId, qty) ->
                    async {
                        let! result = productClient.CheckStock productId qty
                        return (productId, qty, result)
                    })
                |> Async.Parallel
            
            let unavailable =
                stockChecks
                |> Array.choose (fun (pid, qty, result) ->
                    match result with
                    | Ok false -> Some (sprintf "Product %s (qty %d) not available" pid qty)
                    | Error e -> Some e
                    | Ok true -> None)
            
            if unavailable.Length > 0 then
                return Error (String.concat "; " unavailable)
            else
                let orderId = System.Guid.NewGuid().ToString()
                printfn "Order %s created with %d items" orderId items.Length
                return Ok orderId
        }
```

## 4. gRPC กับ F#

```fsharp
// ===== .proto file =====
// product.proto:
(*
syntax = "proto3";

option csharp_namespace = "ProductService.Grpc";

package product;

service ProductCatalog {
  rpc GetProduct (GetProductRequest) returns (ProductResponse);
  rpc GetProducts (GetProductsRequest) returns (stream ProductResponse);
  rpc CheckStock (CheckStockRequest) returns (CheckStockResponse);
}

message GetProductRequest {
  string product_id = 1;
}

message GetProductsRequest {
  repeated string product_ids = 1;
}

message ProductResponse {
  string id = 1;
  string name = 2;
  double price = 3;
  int32 stock = 4;
  bool is_available = 5;
}

message CheckStockRequest {
  string product_id = 1;
  int32 quantity = 2;
}

message CheckStockResponse {
  bool available = 1;
  int32 current_stock = 2;
}
*)

// ===== gRPC Server Implementation =====
open Grpc.Core

// Generated types from proto (simplified representation)
type GetProductRequest = { ProductId: string }
type ProductResponse = { Id: string; Name: string; Price: float; Stock: int; IsAvailable: bool }
type CheckStockRequest = { ProductId: string; Quantity: int }
type CheckStockResponse = { Available: bool; CurrentStock: int }

// gRPC Service implementation
// In real F# code with Grpc.AspNetCore:
(*
type ProductCatalogService(productRepo: IProductRepository) =
    inherit ProductCatalog.ProductCatalogBase()
    
    override _.GetProduct(request: GetProductRequest, context: ServerCallContext) =
        task {
            let! product = productRepo.FindById(request.ProductId)
            match product with
            | None ->
                context.Status <- Status(StatusCode.NotFound, sprintf "Product %s not found" request.ProductId)
                return ProductResponse()
            | Some p ->
                return ProductResponse(
                    Id = p.Id,
                    Name = p.Name,
                    Price = float p.Price,
                    Stock = p.Stock,
                    IsAvailable = p.IsAvailable
                )
        }
    
    override _.GetProducts(request: GetProductsRequest, responseStream: IServerStreamWriter<ProductResponse>, context: ServerCallContext) =
        task {
            for productId in request.ProductIds do
                let! product = productRepo.FindById(productId)
                match product with
                | Some p ->
                    do! responseStream.WriteAsync(ProductResponse(
                        Id = p.Id,
                        Name = p.Name,
                        Price = float p.Price,
                        Stock = p.Stock,
                        IsAvailable = p.IsAvailable
                    ))
                | None -> ()
        }
*)

// gRPC Client usage
let grpcClientExample () =
    // Real usage with Grpc.Net.Client:
    (*
    use channel = GrpcChannel.ForAddress("https://localhost:5001")
    let client = ProductCatalog.ProductCatalogClient(channel)
    
    // Unary call
    let response = client.GetProduct(GetProductRequest(ProductId = "P001"))
    printfn "Product: %s, Price: %.2f" response.Name response.Price
    
    // Server streaming
    let streamingCall = client.GetProducts(GetProductsRequest())
    streamingCall.RequestStream.WriteAllAsync([...]) |> ignore
    
    while streamingCall.ResponseStream.MoveNext() do
        let product = streamingCall.ResponseStream.Current
        printfn "Product: %s" product.Name
    *)
    printfn "gRPC client example (requires real gRPC setup)"
```

## 5. Message Queues: RabbitMQ

```fsharp
// ===== RabbitMQ Integration =====
// Requires: RabbitMQ.Client NuGet package

open System.Text
open System.Text.Json

// Message types
type OrderCreatedMessage = {
    OrderId: string
    CustomerId: string
    Items: {| ProductId: string; Quantity: int; Price: decimal |} list
    TotalAmount: decimal
    CreatedAt: System.DateTime
}

type PaymentProcessedMessage = {
    OrderId: string
    PaymentId: string
    Amount: decimal
    Success: bool
    FailureReason: string option
    ProcessedAt: System.DateTime
}

type NotificationMessage = {
    UserId: string
    Channel: string  // "email" | "sms" | "push"
    Subject: string
    Body: string
    Priority: int
}

// RabbitMQ abstraction
type IMessageBus =
    abstract member Publish: 'T -> string -> Async<unit>
    abstract member Subscribe: string -> ('T -> Async<unit>) -> unit

// Simplified RabbitMQ implementation
type RabbitMQMessageBus(connectionString: string) =
    
    // In real code using RabbitMQ.Client:
    (*
    let factory = ConnectionFactory(Uri = Uri(connectionString))
    let connection = factory.CreateConnection()
    let channel = connection.CreateModel()
    
    let declareQueue queueName =
        channel.QueueDeclare(
            queue = queueName,
            durable = true,
            exclusive = false,
            autoDelete = false,
            arguments = null)
    *)
    
    interface IMessageBus with
        member _.Publish<'T> (message: 'T) (queueName: string) =
            async {
                let json = JsonSerializer.Serialize(message)
                let body = Encoding.UTF8.GetBytes(json)
                // Real: channel.BasicPublish(...)
                printfn "[RabbitMQ] Published to %s: %s..." queueName (json.[..min 50 (json.Length-1)])
            }
        
        member _.Subscribe<'T> (queueName: string) (handler: 'T -> Async<unit>) =
            // Real: set up consumer on channel
            printfn "[RabbitMQ] Subscribed to queue: %s" queueName

// Event handlers (Consumers)
let handleOrderCreated (notificationBus: IMessageBus) (message: OrderCreatedMessage) =
    async {
        printfn "[OrderHandler] Processing order %s" message.OrderId
        
        // Send notification
        let notification = {
            UserId = message.CustomerId
            Channel = "email"
            Subject = sprintf "Order %s Confirmed" message.OrderId
            Body = sprintf "Your order for %.2f has been received" message.TotalAmount
            Priority = 1
        }
        
        do! notificationBus.Publish notification "notifications"
        printfn "[OrderHandler] Notification queued for order %s" message.OrderId
    }

let handlePaymentProcessed (orderBus: IMessageBus) (message: PaymentProcessedMessage) =
    async {
        if message.Success then
            printfn "[PaymentHandler] Payment successful for order %s" message.OrderId
            // Trigger order fulfillment
        else
            printfn "[PaymentHandler] Payment failed for order %s: %s" 
                    message.OrderId 
                    (message.FailureReason |> Option.defaultValue "Unknown")
    }
```

## 6. Health Checks

```fsharp
// ===== Health Checks with ASP.NET Core =====

open Microsoft.Extensions.Diagnostics.HealthChecks
open System.Threading
open System.Threading.Tasks

// Custom health check
type DatabaseHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context: HealthCheckContext, cancellationToken: CancellationToken) =
            task {
                try
                    // Real: check DB connection
                    // use conn = new SqlConnection(connectionString)
                    // do! conn.OpenAsync(cancellationToken)
                    return HealthCheckResult.Healthy("Database connection OK")
                with ex ->
                    return HealthCheckResult.Unhealthy(sprintf "Database unavailable: %s" ex.Message)
            }

type RabbitMQHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context, cancellationToken) =
            task {
                try
                    // Real: check RabbitMQ connection
                    return HealthCheckResult.Healthy("RabbitMQ connection OK")
                with ex ->
                    return HealthCheckResult.Unhealthy(sprintf "RabbitMQ unavailable: %s" ex.Message)
            }

type ExternalServiceHealthCheck(httpClient: HttpClient, serviceUrl: string, serviceName: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context, cancellationToken) =
            task {
                try
                    let! response = httpClient.GetAsync(sprintf "%s/health" serviceUrl, cancellationToken)
                    if response.IsSuccessStatusCode then
                        return HealthCheckResult.Healthy(sprintf "%s is healthy" serviceName)
                    else
                        return HealthCheckResult.Degraded(sprintf "%s returned %d" serviceName (int response.StatusCode))
                with ex ->
                    return HealthCheckResult.Unhealthy(sprintf "%s unreachable: %s" serviceName ex.Message)
            }

// Register health checks in Program.fs:
(*
builder.Services
    .AddHealthChecks()
    .AddCheck<DatabaseHealthCheck>("database", tags = [|"db"|])
    .AddCheck<RabbitMQHealthCheck>("rabbitmq", tags = [|"messaging"|])
    .AddCheck("product-service", 
        fun sp -> 
            let client = sp.GetRequiredService<ProductServiceClient>()
            // check it
            HealthCheckResult.Healthy()) |> ignore

// Map health check endpoints
app.MapHealthChecks("/health") |> ignore
app.MapHealthChecks("/health/ready", HealthCheckOptions(
    Predicate = fun check -> check.Tags.Contains("db"))) |> ignore
app.MapHealthChecks("/health/live", HealthCheckOptions(
    Predicate = fun _ -> false)) |> ignore
*)
```

## 7. Circuit Breaker กับ Polly

```fsharp
// ===== Circuit Breaker with Polly =====
// Requires: Polly NuGet package

open Polly
open Polly.CircuitBreaker

// Policy configurations
let createRetryPolicy () =
    Policy
        .Handle<System.Net.Http.HttpRequestException>()
        .WaitAndRetryAsync(
            retryCount = 3,
            sleepDurationProvider = fun retryAttempt ->
                System.TimeSpan.FromSeconds(float (System.Math.Pow(2.0, float retryAttempt))),
            onRetry = fun (exception: System.Exception) timespan retryCount ctx ->
                printfn "[Retry] Attempt %d after %A: %s" retryCount timespan exception.Message)

let createCircuitBreakerPolicy () =
    Policy
        .Handle<System.Net.Http.HttpRequestException>()
        .CircuitBreakerAsync(
            exceptionsAllowedBeforeBreaking = 5,
            durationOfBreak = System.TimeSpan.FromSeconds(30.0),
            onBreak = fun _ duration ->
                printfn "[CircuitBreaker] OPEN for %A" duration,
            onReset = fun () ->
                printfn "[CircuitBreaker] CLOSED",
            onHalfOpen = fun () ->
                printfn "[CircuitBreaker] HALF-OPEN")

let createTimeoutPolicy () =
    Policy.TimeoutAsync(System.TimeSpan.FromSeconds(5.0))

// Combine policies
let createResiliencePolicy () =
    let retry = createRetryPolicy()
    let circuitBreaker = createCircuitBreakerPolicy()
    let timeout = createTimeoutPolicy()
    
    // Timeout wraps circuit breaker wraps retry
    Policy.WrapAsync(timeout, circuitBreaker, retry)

// Resilient HTTP client
type ResilientHttpClient(httpClient: System.Net.Http.HttpClient) =
    let policy = createResiliencePolicy()
    
    member _.GetAsync<'T> (url: string) =
        async {
            try
                let! result = 
                    policy.ExecuteAsync(fun () ->
                        task {
                            let! response = httpClient.GetAsync(url)
                            response.EnsureSuccessStatusCode() |> ignore
                            let! content = response.Content.ReadAsStringAsync()
                            return System.Text.Json.JsonSerializer.Deserialize<'T>(content)
                        }) |> Async.AwaitTask
                return Ok result
            with
            | :? BrokenCircuitException ->
                return Error "Service unavailable (circuit breaker open)"
            | :? System.TimeoutException ->
                return Error "Request timed out"
            | ex ->
                return Error (sprintf "Request failed: %s" ex.Message)
        }
```

## 8. Distributed Tracing กับ OpenTelemetry

```fsharp
// ===== OpenTelemetry Distributed Tracing =====
// Requires: OpenTelemetry.* NuGet packages

open System.Diagnostics

// Activity source for this service
let activitySource = new ActivitySource("OrderService")

// Trace an operation
let processOrderWithTracing (orderId: string) (items: string list) =
    async {
        use activity = activitySource.StartActivity("ProcessOrder")
        match activity with
        | null -> ()
        | act ->
            act.SetTag("order.id", orderId) |> ignore
            act.SetTag("order.items.count", items.Length) |> ignore
        
        // Simulate processing steps
        use validationActivity = activitySource.StartActivity("ValidateOrder")
        match validationActivity with
        | null -> ()
        | act -> act.SetTag("validation.items", items.Length) |> ignore
        
        do! Async.Sleep 10  // Simulate work
        
        use paymentActivity = activitySource.StartActivity("ProcessPayment")
        match paymentActivity with
        | null -> ()
        | act ->
            act.SetTag("payment.method", "credit_card") |> ignore
            act.SetTag("payment.amount", 100.0) |> ignore
        
        do! Async.Sleep 20  // Simulate payment
        
        printfn "Order %s processed with tracing" orderId
        return Ok orderId
    }

// Setup OpenTelemetry (in Program.fs):
(*
open OpenTelemetry.Trace
open OpenTelemetry.Resources

builder.Services
    .AddOpenTelemetry()
    .WithTracing(fun builder ->
        builder
            .SetResourceBuilder(
                ResourceBuilder.CreateDefault()
                    .AddService("OrderService", serviceVersion = "1.0.0"))
            .AddAspNetCoreInstrumentation()
            .AddHttpClientInstrumentation()
            .AddSource("OrderService")
            .AddJaegerExporter(fun opts ->
                opts.AgentHost <- "localhost"
                opts.AgentPort <- 6831)
        |> ignore)
    |> ignore
*)
```

## 9. API Gateway Pattern

```fsharp
// ===== API Gateway =====
// Simple reverse proxy / aggregator pattern

open System.Net.Http

type ServiceRegistry = {
    UserServiceUrl: string
    OrderServiceUrl: string
    ProductServiceUrl: string
    PaymentServiceUrl: string
}

// API Gateway aggregates calls from multiple services
type ApiGateway(registry: ServiceRegistry, httpClientFactory: IHttpClientFactory) =
    
    let getClient serviceName = httpClientFactory.CreateClient(serviceName)
    
    // Aggregate dashboard data from multiple services
    member _.GetUserDashboard (userId: string) =
        async {
            let userClient = getClient "user-service"
            let orderClient = getClient "order-service"
            
            // Parallel calls to multiple services
            let! userTask = 
                userClient.GetAsync(sprintf "%s/api/users/%s" registry.UserServiceUrl userId)
                |> Async.AwaitTask
            
            let! ordersTask =
                orderClient.GetAsync(sprintf "%s/api/orders?userId=%s&limit=5" registry.OrderServiceUrl userId)
                |> Async.AwaitTask
            
            if not userTask.IsSuccessStatusCode then
                return Error "User not found"
            else
            
            let! userContent = userTask.Content.ReadAsStringAsync() |> Async.AwaitTask
            let! ordersContent = ordersTask.Content.ReadAsStringAsync() |> Async.AwaitTask
            
            return Ok {|
                User = userContent
                RecentOrders = ordersContent
                Timestamp = System.DateTime.UtcNow
            |}
        }
    
    // Route and transform requests
    member _.RouteRequest (path: string) (method: string) =
        let (serviceUrl, newPath) =
            if path.StartsWith("/users") then
                (registry.UserServiceUrl, path)
            elif path.StartsWith("/orders") then
                (registry.OrderServiceUrl, path)
            elif path.StartsWith("/products") then
                (registry.ProductServiceUrl, path)
            elif path.StartsWith("/payments") then
                (registry.PaymentServiceUrl, path)
            else
                ("", path)
        
        if serviceUrl = "" then
            Error "Service not found"
        else
            Ok (sprintf "%s%s" serviceUrl newPath)
```

## 10. Docker Containerization

```dockerfile
# Dockerfile สำหรับ F# Microservice
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy project files
COPY ["OrderService/OrderService.fsproj", "OrderService/"]
RUN dotnet restore "OrderService/OrderService.fsproj"

# Copy source
COPY . .
WORKDIR "/src/OrderService"
RUN dotnet build "OrderService.fsproj" -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "OrderService.fsproj" -c Release -o /app/publish

FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS final
WORKDIR /app
COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080

ENTRYPOINT ["dotnet", "OrderService.dll"]
```

```yaml
# docker-compose.yml สำหรับ microservices
version: '3.8'

services:
  # Message Broker
  rabbitmq:
    image: rabbitmq:3-management
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      RABBITMQ_DEFAULT_USER: admin
      RABBITMQ_DEFAULT_PASS: password
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "check_port_connectivity"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Database
  postgres:
    image: postgres:15
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: microservices
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U admin"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Tracing
  jaeger:
    image: jaegertracing/all-in-one:latest
    ports:
      - "6831:6831/udp"  # Agent
      - "16686:16686"     # UI
      - "14268:14268"     # Collector

  # Services
  user-service:
    build:
      context: ./UserService
      dockerfile: Dockerfile
    ports:
      - "5001:8080"
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=users;Username=admin;Password=password
      - RabbitMQ__ConnectionString=amqp://admin:password@rabbitmq:5672
      - Otlp__Endpoint=http://jaeger:4317
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  order-service:
    build:
      context: ./OrderService
      dockerfile: Dockerfile
    ports:
      - "5002:8080"
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=orders;Username=admin;Password=password
      - RabbitMQ__ConnectionString=amqp://admin:password@rabbitmq:5672
      - Services__UserServiceUrl=http://user-service:8080
      - Services__ProductServiceUrl=http://product-service:8080
      - Otlp__Endpoint=http://jaeger:4317
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy
      user-service:
        condition: service_started

  product-service:
    build:
      context: ./ProductService
      dockerfile: Dockerfile
    ports:
      - "5003:8080"
    environment:
      - ConnectionStrings__DefaultConnection=Host=postgres;Database=products;Username=admin;Password=password
      - RabbitMQ__ConnectionString=amqp://admin:password@rabbitmq:5672
    depends_on:
      postgres:
        condition: service_healthy
      rabbitmq:
        condition: service_healthy

  notification-service:
    build:
      context: ./NotificationService
      dockerfile: Dockerfile
    ports:
      - "5004:8080"
    environment:
      - RabbitMQ__ConnectionString=amqp://admin:password@rabbitmq:5672
      - Email__SmtpServer=smtp.gmail.com
      - Email__Port=587
    depends_on:
      rabbitmq:
        condition: service_healthy

  api-gateway:
    build:
      context: ./ApiGateway
      dockerfile: Dockerfile
    ports:
      - "5000:8080"
    environment:
      - Services__UserService=http://user-service:8080
      - Services__OrderService=http://order-service:8080
      - Services__ProductService=http://product-service:8080
    depends_on:
      - user-service
      - order-service
      - product-service

volumes:
  postgres_data:
```

## 11. Service Discovery

```fsharp
// ===== Service Discovery =====
// Using Consul or Kubernetes DNS

type ServiceEndpoint = {
    ServiceName: string
    Host: string
    Port: int
    IsHealthy: bool
}

type IServiceDiscovery =
    abstract member Register: ServiceEndpoint -> Async<unit>
    abstract member Deregister: string -> Async<unit>
    abstract member Resolve: string -> Async<ServiceEndpoint option>
    abstract member GetAll: string -> Async<ServiceEndpoint list>

// Simple round-robin load balancer
type RoundRobinLoadBalancer(discovery: IServiceDiscovery) =
    let counters = System.Collections.Concurrent.ConcurrentDictionary<string, int>()
    
    member _.GetEndpoint (serviceName: string) =
        async {
            let! endpoints = discovery.GetAll serviceName
            let healthy = endpoints |> List.filter (fun e -> e.IsHealthy)
            
            if healthy.IsEmpty then
                return None
            else
                let counter = counters.AddOrUpdate(serviceName, 0, fun _ c -> (c + 1) % healthy.Length)
                return Some healthy.[counter]
        }

// Kubernetes-based service discovery (via DNS)
type KubernetesServiceDiscovery(namespace': string) =
    interface IServiceDiscovery with
        member _.Register endpoint =
            async {
                // In K8s, services are registered via manifests
                printfn "Service %s registered at %s:%d" endpoint.ServiceName endpoint.Host endpoint.Port
            }
        
        member _.Deregister serviceName =
            async {
                printfn "Service %s deregistered" serviceName
            }
        
        member _.Resolve serviceName =
            async {
                // K8s DNS: <service-name>.<namespace>.svc.cluster.local
                let host = sprintf "%s.%s.svc.cluster.local" serviceName namespace'
                return Some { ServiceName = serviceName; Host = host; Port = 8080; IsHealthy = true }
            }
        
        member _.GetAll serviceName =
            async {
                let host = sprintf "%s.%s.svc.cluster.local" serviceName namespace'
                return [{ ServiceName = serviceName; Host = host; Port = 8080; IsHealthy = true }]
            }
```

## 12. สรุป Microservices กับ F#

```fsharp
(*
Microservices Patterns ที่สำคัญ:

1. Service Communication
   - REST: ง่าย, HTTP-based, human-readable
   - gRPC: เร็วกว่า, binary protocol, streaming support
   - Message Queue: async, decoupled

2. Resilience Patterns (Polly)
   - Retry: ลอง request ใหม่เมื่อ fail
   - Circuit Breaker: หยุด call service ที่ down ชั่วคราว
   - Timeout: จำกัดเวลา wait
   - Bulkhead: จำกัด concurrent requests

3. Observability
   - Logs: Serilog structured logging
   - Metrics: Prometheus + Grafana
   - Traces: OpenTelemetry + Jaeger

4. Health Checks
   - /health/live: Is service running?
   - /health/ready: Is service ready to accept traffic?

5. Containerization
   - Docker: package service + dependencies
   - docker-compose: local development
   - Kubernetes: production orchestration

F# Advantages for Microservices:
✓ Immutable messages = no accidental mutation
✓ Discriminated Unions = explicit state machines
✓ Type-safe serialization
✓ Async workflows built-in
✓ Small, focused services fit F# modules well
*)

printfn "Microservices with F# - Complete!"
```

---

**สรุป**: F# เหมาะมากสำหรับ microservices เพราะ type system ทำให้ messages และ domain logic ชัดเจน ส่วน Polly, OpenTelemetry, และ ASP.NET Core ให้ tools ที่จำเป็นสำหรับ production microservices
