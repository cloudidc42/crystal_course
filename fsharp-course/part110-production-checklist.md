# Part 110 - Production Checklist สำหรับ F# Applications

## บทนำ (Introduction)

Production Checklist เป็นรายการตรวจสอบที่ครอบคลุมทุกด้านของการนำ F# application ขึ้น production อย่างปลอดภัยและมั่นคง

---

## 1. Security Checklist

### Authentication & Authorization

```fsharp
// ✅ ใช้ JWT tokens พร้อม proper expiration
// ✅ Implement token refresh mechanism
// ✅ ใช้ HTTPS เสมอ
// ✅ Validate tokens ทุก request

// Authentication middleware
open Microsoft.AspNetCore.Authentication.JwtBearer
open System.IdentityModel.Tokens.Jwt

let configureAuth (services: IServiceCollection) (config: IConfiguration) =
    services
        .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
        .AddJwtBearer(fun options ->
            options.TokenValidationParameters <- TokenValidationParameters(
                ValidateIssuer = true,
                ValidateAudience = true,
                ValidateLifetime = true,              // ✅ ตรวจ expiry
                ValidateIssuerSigningKey = true,
                ValidIssuer = config["Jwt:Issuer"],
                ValidAudience = config["Jwt:Audience"],
                IssuerSigningKey = SymmetricSecurityKey(
                    Encoding.UTF8.GetBytes(config["Jwt:SecretKey"])),
                ClockSkew = System.TimeSpan.Zero      // ✅ ไม่ tolerate clock drift
            ))
    |> ignore

// ✅ Role-based authorization
[<Authorize(Roles = "Admin,Manager")>]
type AdminController() =
    inherit ControllerBase()
    
    [<HttpDelete("{id}")>]
    member _.Delete(id: int) = 
        // Only admins and managers can delete
        ()

// ✅ Resource-based authorization
let authorizeUserAccess (userId: Guid) (httpContext: HttpContext) =
    let currentUserId = httpContext.User.FindFirst("sub")?.Value |> Option.ofObj
    match currentUserId with
    | Some id when id = userId.ToString() -> true
    | _ -> 
        httpContext.User.IsInRole("Admin")

// ✅ Validate all inputs
let validateInput (input: string) =
    if System.String.IsNullOrWhiteSpace(input) then
        Error "Input cannot be empty"
    elif input.Length > 1000 then
        Error "Input too long"
    elif input.Contains("<script") then
        Error "Invalid characters"
    else
        Ok input
```

### Input Validation และ Sanitization

```fsharp
// ✅ Validate ทุก input จาก user
// ✅ Sanitize HTML content
// ✅ ป้องกัน SQL injection ด้วย parameterized queries
// ✅ ป้องกัน XSS

// SQL injection prevention - ใช้ parameterized queries เสมอ
let getUserSafe (conn: IDbConnection) (email: string) =
    // ✅ ถูกต้อง: parameterized query
    conn.QueryFirstOrDefault<User>(
        "SELECT * FROM users WHERE email = @email",
        {| email = email |})

// ❌ ผิด: string concatenation
// conn.Execute(sprintf "SELECT * FROM users WHERE email = '%s'" email)

// HTML sanitization
open HtmlSanitizer

let sanitizeHtml (html: string) =
    let sanitizer = HtmlSanitizer()
    sanitizer.AllowedTags.Add("b")
    sanitizer.AllowedTags.Add("i")
    sanitizer.AllowedTags.Add("p")
    sanitizer.Sanitize(html)

// CORS configuration - ระบุ origins ที่อนุญาต
let configureCors (services: IServiceCollection) =
    services.AddCors(fun options ->
        options.AddPolicy("ProductionPolicy", fun builder ->
            builder
                .WithOrigins([| "https://myapp.com"; "https://www.myapp.com" |])  // ✅ ระบุชัดเจน
                .AllowCredentials()
                .AllowAnyHeader()
                .AllowAnyMethod()
            |> ignore))
    |> ignore
```

### Secrets Management

```fsharp
// ✅ ไม่ commit secrets ลง git
// ✅ ใช้ environment variables หรือ secret managers
// ✅ Rotate secrets regularly
// ✅ ใช้ different secrets per environment

// Development: User Secrets
// dotnet user-secrets set "ConnectionStrings:Db" "..."

// Production: Environment Variables หรือ Azure Key Vault

// ✅ ใช้ IConfiguration ดึง secrets
let getConnectionString (config: IConfiguration) =
    config.GetConnectionString("DefaultConnection")
    |> Option.ofObj
    |> Option.defaultWith (fun () -> failwith "Connection string not configured")

// ✅ Azure Key Vault integration
let configureKeyVault (config: IConfigurationBuilder) =
    let vaultUri = System.Uri(System.Environment.GetEnvironmentVariable("KEYVAULT_URI"))
    config.AddAzureKeyVault(vaultUri, DefaultAzureCredential())
    |> ignore

// ✅ Never log secrets
let logUserAction (userId: string) (action: string) (logger: ILogger) =
    // ✅ ถูกต้อง
    logger.LogInformation("User {UserId} performed {Action}", userId, action)
    // ❌ ผิด: อย่า log password หรือ tokens
    // logger.LogInformation("User {UserId} with password {Password}", userId, password)
```

---

## 2. Performance Checklist

### Database Performance

```fsharp
// ✅ ใช้ async/await ทุก I/O operation
// ✅ Connection pooling
// ✅ Indexes ที่เหมาะสม
// ✅ N+1 query ต้องหลีกเลี่ยง

// ✅ Async database calls
let getOrdersAsync (conn: IDbConnection) (customerId: Guid) =
    async {
        let! orders = 
            conn.QueryAsync<Order>(
                "SELECT * FROM orders WHERE customer_id = @id ORDER BY created_at DESC",
                {| id = customerId |})
            |> Async.AwaitTask
        return orders |> Seq.toList
    }

// ✅ Avoid N+1: ใช้ JOIN หรือ batch loading
let getOrdersWithItems (conn: IDbConnection) (customerId: Guid) =
    async {
        let sql = """
            SELECT o.*, i.product_id, i.quantity, i.unit_price
            FROM orders o
            LEFT JOIN order_items i ON o.id = i.order_id
            WHERE o.customer_id = @id
        """
        let! results = conn.QueryAsync(sql, {| id = customerId |}) |> Async.AwaitTask
        // Group by order
        let orders = 
            results
            |> Seq.groupBy (fun r -> r?id)
            |> Seq.map (fun (id, rows) ->
                let first = Seq.head rows
                { Id = id; Items = rows |> Seq.map (fun r -> { ProductId = r?product_id; Quantity = r?quantity }) |> Seq.toList })
            |> Seq.toList
        return orders
    }

// ✅ Pagination
let getPagedResults (conn: IDbConnection) (page: int) (pageSize: int) =
    async {
        let offset = (page - 1) * pageSize
        let! results =
            conn.QueryAsync<Product>(
                "SELECT * FROM products ORDER BY name LIMIT @limit OFFSET @offset",
                {| limit = pageSize; offset = offset |})
            |> Async.AwaitTask
        
        let! totalCount =
            conn.QueryFirstAsync<int>("SELECT COUNT(*) FROM products")
            |> Async.AwaitTask
        
        return {
            Items = results |> Seq.toList
            Total = totalCount
            Page = page
            PageSize = pageSize
            TotalPages = (totalCount + pageSize - 1) / pageSize
        }
    }
```

### Caching

```fsharp
open Microsoft.Extensions.Caching.Distributed
open System.Text.Json

// ✅ Cache frequently accessed data
let getCachedUser (cache: IDistributedCache) (id: Guid) (getFromDb: Guid -> Async<User option>) =
    async {
        let key = sprintf "user:%s" (id.ToString("N"))
        
        let! cached = cache.GetStringAsync(key) |> Async.AwaitTask
        
        if not (isNull cached) then
            return Some (JsonSerializer.Deserialize<User>(cached))
        else
            let! user = getFromDb id
            
            match user with
            | Some u ->
                let json = JsonSerializer.Serialize(u)
                let options = DistributedCacheEntryOptions(
                    AbsoluteExpirationRelativeToNow = System.TimeSpan.FromMinutes 5.0)
                do! cache.SetStringAsync(key, json, options) |> Async.AwaitTask
                return Some u
            | None ->
                return None
    }

// ✅ Cache invalidation
let invalidateUserCache (cache: IDistributedCache) (userId: Guid) =
    async {
        let key = sprintf "user:%s" (userId.ToString("N"))
        do! cache.RemoveAsync(key) |> Async.AwaitTask
    }

// ✅ Memory cache สำหรับ static data
open Microsoft.Extensions.Caching.Memory

let getCachedConfig (cache: IMemoryCache) =
    cache.GetOrCreate("app-config", fun entry ->
        entry.AbsoluteExpirationRelativeToNow <- System.TimeSpan.FromHours 1.0
        loadConfigFromDb())  // Load once, cache for 1 hour
```

---

## 3. Reliability Checklist

### Error Handling

```fsharp
// ✅ Handle ทุก error อย่างชัดเจน
// ✅ ไม่ swallow exceptions โดยไม่ log
// ✅ Graceful degradation
// ✅ Circuit breaker pattern

// ✅ Result type สำหรับ domain errors
type AppError =
    | NotFound of resource: string * id: string
    | ValidationError of errors: string list
    | AuthenticationError of string
    | AuthorizationError of string
    | DatabaseError of System.Exception
    | ExternalServiceError of service: string * System.Exception

let handleAppError (error: AppError) (logger: ILogger) =
    match error with
    | NotFound (res, id) ->
        logger.LogWarning("Resource {Resource} with ID {Id} not found", res, id)
        Results.NotFound({| error = sprintf "%s %s not found" res id |})
    
    | ValidationError errors ->
        logger.LogWarning("Validation failed: {Errors}", errors)
        Results.BadRequest({| errors = errors |})
    
    | AuthenticationError msg ->
        logger.LogWarning("Authentication failed: {Message}", msg)
        Results.Unauthorized()
    
    | AuthorizationError msg ->
        logger.LogWarning("Authorization failed: {Message}", msg)
        Results.Forbid()
    
    | DatabaseError ex ->
        logger.LogError(ex, "Database error occurred")
        Results.StatusCode(500)
    
    | ExternalServiceError (service, ex) ->
        logger.LogError(ex, "External service {Service} failed", service)
        Results.StatusCode(503)

// ✅ Circuit Breaker (ใช้ Polly)
open Polly
open Polly.CircuitBreaker

let createCircuitBreaker () =
    Policy
        .Handle<System.Exception>()
        .CircuitBreakerAsync(
            exceptionsAllowedBeforeBreaking = 5,
            durationOfBreak = System.TimeSpan.FromSeconds 30.0)

let callExternalService (circuitBreaker: IAsyncPolicy) (service: unit -> Async<string>) =
    circuitBreaker.ExecuteAsync(fun () ->
        service() |> Async.StartAsTask)
    |> Async.AwaitTask

// ✅ Retry Policy
let retryPolicy =
    Policy
        .Handle<System.Net.Http.HttpRequestException>()
        .WaitAndRetryAsync(
            retryCount = 3,
            sleepDurationProvider = fun attempt -> 
                System.TimeSpan.FromSeconds(System.Math.Pow(2.0, float attempt)))
```

### Health Checks

```fsharp
// ✅ Implement health checks
// ✅ Database connectivity check
// ✅ External service check
// ✅ Disk space check

open Microsoft.Extensions.Diagnostics.HealthChecks

type DatabaseHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context, token) =
            task {
                try
                    use conn = new Npgsql.NpgsqlConnection(connectionString)
                    do! conn.OpenAsync(token)
                    let! result = conn.ExecuteScalarAsync<int>("SELECT 1")
                    if result = 1 then
                        return HealthCheckResult.Healthy("Database is healthy")
                    else
                        return HealthCheckResult.Unhealthy("Database check failed")
                with ex ->
                    return HealthCheckResult.Unhealthy("Cannot connect to database", ex)
            }

let configureHealthChecks (services: IServiceCollection) (config: IConfiguration) =
    services
        .AddHealthChecks()
        .AddCheck<DatabaseHealthCheck>("database")
        .AddUrlGroup(
            System.Uri("https://api.external-service.com/health"),
            "external-api",
            HealthStatus.Degraded)
        .AddDiskStorageHealthCheck(
            setup = (fun diskOptions ->
                diskOptions.AddDrive("C:\\", minimumFreeMegabytes = 500L)),
            name = "disk-space")
    |> ignore

// Health check endpoint
let configureApp (app: IApplicationBuilder) =
    app.UseHealthChecks("/health") |> ignore
    app.UseHealthChecks("/health/ready", HealthCheckOptions(
        Predicate = fun check -> check.Tags.Contains("ready"))) |> ignore
    app.UseHealthChecks("/health/live", HealthCheckOptions(
        Predicate = fun check -> check.Tags.Contains("live"))) |> ignore
```

---

## 4. Observability Checklist

### Logging

```fsharp
// ✅ Structured logging
// ✅ Appropriate log levels
// ✅ Correlation IDs ทุก request
// ✅ ไม่ log sensitive data

open Serilog
open Serilog.Events

let configureSerilog () =
    LoggerConfiguration()
        .MinimumLevel.Information()
        .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
        .MinimumLevel.Override("System", LogEventLevel.Warning)
        .Enrich.FromLogContext()
        .Enrich.WithMachineName()
        .Enrich.WithEnvironmentName()
        .WriteTo.Console(
            outputTemplate = "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
        .WriteTo.File(
            path = "logs/app-.log",
            rollingInterval = RollingInterval.Day,
            retainedFileCountLimit = 30)
        .WriteTo.Seq("http://localhost:5341")  // Seq log server
        .CreateLogger()

// ✅ Correlation ID middleware
let correlationMiddleware =
    fun (ctx: HttpContext) (next: RequestDelegate) ->
        task {
            let correlationId = 
                ctx.Request.Headers["X-Correlation-ID"].ToString()
                |> fun id -> if id = "" then System.Guid.NewGuid().ToString("N") else id
            
            ctx.Response.Headers.["X-Correlation-ID"] <- correlationId
            
            use _ = LogContext.PushProperty("CorrelationId", correlationId)
            
            do! next.Invoke(ctx)
        }

// ✅ Request logging
let requestLoggingMiddleware (logger: ILogger<RequestLoggingMiddleware>) =
    fun (ctx: HttpContext) (next: RequestDelegate) ->
        task {
            let sw = System.Diagnostics.Stopwatch.StartNew()
            
            logger.LogInformation(
                "Request {Method} {Path} started",
                ctx.Request.Method,
                ctx.Request.Path)
            
            do! next.Invoke(ctx)
            
            sw.Stop()
            
            logger.LogInformation(
                "Request {Method} {Path} completed with {StatusCode} in {ElapsedMs}ms",
                ctx.Request.Method,
                ctx.Request.Path,
                ctx.Response.StatusCode,
                sw.ElapsedMilliseconds)
        }
```

### Metrics

```fsharp
// ✅ Application metrics
// ✅ Business metrics
// ✅ Infrastructure metrics

open Prometheus

// Define metrics
let requestCounter = 
    Metrics.CreateCounter(
        "http_requests_total",
        "Total HTTP requests",
        CounterConfiguration(
            LabelNames = [| "method"; "path"; "status" |]))

let requestDuration =
    Metrics.CreateHistogram(
        "http_request_duration_seconds",
        "HTTP request duration",
        HistogramConfiguration(
            LabelNames = [| "method"; "path" |],
            Buckets = [| 0.005; 0.01; 0.025; 0.05; 0.1; 0.25; 0.5; 1.0; 2.5; 5.0; 10.0 |]))

let activeConnections =
    Metrics.CreateGauge(
        "active_connections",
        "Number of active connections")

// Business metrics
let ordersCreated =
    Metrics.CreateCounter(
        "orders_created_total",
        "Total orders created",
        CounterConfiguration(LabelNames = [| "status" |]))

let orderValue =
    Metrics.CreateHistogram(
        "order_value_baht",
        "Order value in Thai Baht",
        HistogramConfiguration(
            Buckets = [| 100.0; 500.0; 1000.0; 5000.0; 10000.0; 50000.0 |]))

// Record business metric
let recordOrder (order: Order) =
    ordersCreated.WithLabels("created").Inc()
    orderValue.Observe(float order.TotalAmount)
```

### Distributed Tracing

```fsharp
// ✅ OpenTelemetry tracing
// ✅ Trace ทุก external calls
// ✅ ใส่ meaningful span names

open OpenTelemetry.Trace

let configureTracing (services: IServiceCollection) =
    services
        .AddOpenTelemetry()
        .WithTracing(fun tracing ->
            tracing
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                .AddSqlClientInstrumentation()
                .AddOtlpExporter(fun opts ->
                    opts.Endpoint <- System.Uri("http://localhost:4317"))
            |> ignore)
    |> ignore

// ใช้ tracing ใน code
let processOrderWithTracing (tracer: Tracer) (order: Order) =
    async {
        use span = tracer.StartActiveSpan("ProcessOrder")
        span.SetAttribute("order.id", order.Id.ToString())
        span.SetAttribute("order.amount", float order.TotalAmount)
        
        try
            // Process order...
            span.SetStatus(Status.Ok)
            return Ok ()
        with ex ->
            span.SetStatus(Status.Error, ex.Message)
            span.RecordException(ex)
            return Error ex.Message
    }
```

---

## 5. Testing Checklist

### Test Coverage

```fsharp
// ✅ Unit tests สำหรับ business logic ทั้งหมด
// ✅ Integration tests สำหรับ database operations
// ✅ API tests (end-to-end)
// ✅ Performance tests
// ✅ Security tests
// ✅ ≥ 80% code coverage

// Unit test coverage report
// dotnet test --collect:"XPlat Code Coverage"
// reportgenerator -reports:"**/coverage.cobertura.xml" -targetdir:"coveragereport" -reporttypes:Html

// ===== Test Pyramid =====

// Layer 1: Many unit tests (fast, isolated)
[<Fact>]
let ``Unit: Order total calculation is correct`` () =
    let items = [
        { ProductId = 1; Quantity = 2; UnitPrice = 100.0m }
        { ProductId = 2; Quantity = 1; UnitPrice = 250.0m }
    ]
    Assert.Equal(450.0m, calculateTotal items)

// Layer 2: Some integration tests
[<Fact>]
let ``Integration: Order persisted to database`` () =
    async {
        use db = new TestDatabase()
        let service = OrderService(db.Connection)
        let order = TestOrders.pendingOrder (System.Guid.NewGuid())
        
        let! result = service.CreateOrder(order)
        
        match result with
        | Ok savedOrder ->
            let! found = service.GetOrder(savedOrder.Id)
            Assert.Equal(Some savedOrder, found)
        | Error err -> Assert.Fail(err)
    } |> Async.RunSynchronously

// Layer 3: Few E2E tests
[<Fact>]
let ``E2E: Place order via API`` () =
    async {
        use server = new TestServer()
        let client = server.CreateClient()
        
        let token = await server.GetAuthToken("test@example.com", "password")
        client.DefaultRequestHeaders.Authorization <- 
            Headers.AuthenticationHeaderValue("Bearer", token)
        
        let body = JsonContent.Create({| productId = 1; quantity = 2 |})
        let! response = client.PostAsync("/api/orders", body) |> Async.AwaitTask
        
        Assert.Equal(HttpStatusCode.Created, response.StatusCode)
    } |> Async.RunSynchronously
```

---

## 6. Deployment Checklist

### Environment Configuration

```fsharp
// ✅ Environment-specific configuration
// ✅ Infrastructure as Code
// ✅ Zero-downtime deployment
// ✅ Database migration strategy

// appsettings.json (ไม่มี secrets)
(*
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft": "Warning"
    }
  },
  "AllowedHosts": "*"
}
*)

// appsettings.Production.json
(*
{
  "Logging": {
    "LogLevel": {
      "Default": "Warning"
    }
  }
}
*)

// Dockerfile ที่ดี
(*
FROM mcr.microsoft.com/dotnet/sdk:7.0 AS build
WORKDIR /app
COPY *.sln .
COPY src/ src/
RUN dotnet restore
RUN dotnet publish src/MyApp -c Release -o /publish --no-restore

FROM mcr.microsoft.com/dotnet/aspnet:7.0 AS runtime
WORKDIR /app
COPY --from=build /publish .

# ✅ Non-root user
RUN adduser --disabled-password --gecos '' appuser
USER appuser

# ✅ Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD curl -f http://localhost:8080/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
*)

// ✅ Kubernetes deployment
(*
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0    # Zero-downtime
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        livenessProbe:
          httpGet:
            path: /health/live
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 30
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 10
*)
```

---

## 7. Code Quality Checklist

### Code Standards

```fsharp
// ✅ Consistent naming conventions
// ✅ No magic numbers/strings
// ✅ Functions ไม่ยาวเกินไป (≤ 30 lines)
// ✅ ไม่มี dead code
// ✅ ใช้ Option แทน null

// ✅ Constants สำหรับ magic values
module Constants =
    [<Literal>]
    let MaxOrderItems = 100
    
    [<Literal>]
    let VipDiscountRate = 0.10m
    
    [<Literal>]
    let TokenExpiryMinutes = 60
    
    [<Literal>]
    let MaxRetries = 3

// ✅ Domain types ป้องกัน invalid state
type Email private (value: string) =
    member _.Value = value
    
    static member Create (email: string) =
        if System.String.IsNullOrWhiteSpace(email) then
            Error "Email cannot be empty"
        elif not (email.Contains("@")) then
            Error "Invalid email format"
        else
            Ok (Email email)

type Money private (amount: decimal, currency: string) =
    member _.Amount = amount
    member _.Currency = currency
    
    static member Create (amount: decimal) (currency: string) =
        if amount < 0m then Error "Amount cannot be negative"
        elif System.String.IsNullOrWhiteSpace(currency) then Error "Currency required"
        else Ok (Money(amount, currency))
    
    static member (+) (a: Money, b: Money) =
        if a.Currency <> b.Currency then
            Error "Cannot add different currencies"
        else
            Ok (Money(a.Amount + b.Amount, a.Currency))

// ✅ Prefer composition over inheritance
type OrderProcessor = {
    Validate: Order -> Result<Order, string list>
    Calculate: Order -> Order
    Save: Order -> Async<Result<Order, string>>
    Notify: Order -> Async<unit>
}

let processOrder (processor: OrderProcessor) (order: Order) =
    async {
        return!
            order
            |> processor.Validate
            |> Result.map processor.Calculate
            |> function
               | Error errs -> async.Return(Error (String.concat ", " errs))
               | Ok calculated ->
                   async {
                       let! saved = processor.Save calculated
                       match saved with
                       | Ok savedOrder ->
                           do! processor.Notify savedOrder
                           return Ok savedOrder
                       | Error err ->
                           return Error err
                   }
    }
```

---

## 8. SLA/SLO/SLI

### การกำหนด Service Level

```fsharp
// ===== SLI (Service Level Indicators) =====
// ตัวชี้วัดที่วัดได้จริง

// 1. Availability = uptime / total time
// 2. Latency = response time (P50, P95, P99)
// 3. Error Rate = errors / total requests
// 4. Throughput = requests per second

// ===== SLO (Service Level Objectives) =====
// เป้าหมายที่เราตั้ง

(*
Service: Order API
SLOs:
  - Availability: 99.9% (≤ 8.76 hours downtime/year)
  - Latency P50: ≤ 100ms
  - Latency P95: ≤ 500ms
  - Latency P99: ≤ 2000ms
  - Error Rate: < 0.1%
*)

// ===== Error Budget =====

// Error budget = 100% - SLO
// Error budget สำหรับ availability 99.9%:
// = 0.1% = 8.76 hours/year = 43.8 minutes/month

let calculateErrorBudget (sloPercentage: float) (period: System.TimeSpan) =
    let errorBudgetPercentage = 100.0 - sloPercentage
    let totalMinutes = period.TotalMinutes
    let budgetMinutes = totalMinutes * errorBudgetPercentage / 100.0
    
    printfn "SLO: %.3f%%" sloPercentage
    printfn "Error Budget: %.3f%% = %.1f minutes over %.0f days"
        errorBudgetPercentage
        budgetMinutes
        period.TotalDays

calculateErrorBudget 99.9 (System.TimeSpan.FromDays 30.0)
```

---

## 9. Runbook Template

### Runbook สำหรับ Incident Response

```markdown
# Runbook: [Service Name] - [Incident Type]

## Overview
Brief description of what this runbook covers.

## Severity Levels
- **P1 (Critical)**: Complete service outage, data loss
- **P2 (High)**: Degraded performance, partial outage  
- **P3 (Medium)**: Non-critical feature unavailable
- **P4 (Low)**: Minor issue, no customer impact

## Escalation Path
1. On-call engineer (PagerDuty)
2. Team lead
3. Engineering manager
4. CTO

## Symptoms
- Error rate > 1%
- P99 latency > 5 seconds
- Health check failing
- Database connection errors

## Investigation Steps

### Step 1: Check Service Status
```bash
kubectl get pods -n production
kubectl logs -n production deployment/myapp --tail=100
```

### Step 2: Check Metrics
- Grafana dashboard: https://grafana.example.com/d/service-overview
- Focus on: error rate, latency, CPU, memory

### Step 3: Check Database
```bash
kubectl exec -it db-pod -- psql -U app -c "SELECT COUNT(*) FROM connections;"
```

## Resolution Procedures

### Database Connection Issues
1. Check connection pool metrics
2. Restart connection pool: `kubectl rollout restart deployment/myapp`
3. Check DB health: `psql -c "SELECT 1"`

### High CPU/Memory
1. Check for memory leaks in profiler
2. Scale horizontally: `kubectl scale deployment/myapp --replicas=5`
3. Check for infinite loops or excessive logging

### Deployment Issues
1. Check recent deployments: `kubectl rollout history deployment/myapp`
2. Rollback if needed: `kubectl rollout undo deployment/myapp`

## Post-Incident
1. Document in incident tracker
2. Schedule post-mortem within 48 hours
3. Create action items for prevention
```

---

## 10. Post-Mortem Process

### Template สำหรับ Post-Mortem

```fsharp
// ===== Post-Mortem Structure =====

type IncidentSeverity = P1 | P2 | P3 | P4

type ActionItem = {
    Description: string
    Owner: string
    DueDate: System.DateTime
    Priority: string
}

type PostMortem = {
    IncidentDate: System.DateTime
    DetectedAt: System.DateTime
    ResolvedAt: System.DateTime
    Severity: IncidentSeverity
    Title: string
    Summary: string
    CustomerImpact: string
    Timeline: (System.DateTime * string) list
    RootCauses: string list
    ContributingFactors: string list
    WentWell: string list
    WentPoorly: string list
    ActionItems: ActionItem list
}

// ===== Blameless Post-Mortem Principles =====
(*
1. ไม่ blame บุคคล - focus ที่ process และ system
2. ทุกคนทำดีที่สุดตามข้อมูลที่มีในขณะนั้น
3. เป้าหมายคือ learn and improve ไม่ใช่ punish
4. เปิดเผยและ transparent
5. Action items ต้องชัดเจน, มี owner, มี due date
*)

let generatePostMortemReport (pm: PostMortem) =
    let duration = pm.ResolvedAt - pm.DetectedAt
    let timeToDetect = pm.DetectedAt - pm.IncidentDate
    
    printfn """
# Post-Mortem Report: %s
Date: %s | Severity: %A

## Summary
%s

## Customer Impact
%s

## Timeline
Duration: %.0f minutes
Time to Detect: %.0f minutes

Key Events:
%s

## Root Causes
%s

## What Went Well
%s

## What Can Be Improved
%s

## Action Items
%s
""" pm.Title
        (pm.IncidentDate.ToString("yyyy-MM-dd"))
        pm.Severity
        pm.Summary
        pm.CustomerImpact
        duration.TotalMinutes
        timeToDetect.TotalMinutes
        (pm.Timeline |> List.map (fun (dt, event) -> sprintf "  [%s] %s" (dt.ToString("HH:mm")) event) |> String.concat "\n")
        (pm.RootCauses |> List.mapi (fun i rc -> sprintf "  %d. %s" (i+1) rc) |> String.concat "\n")
        (pm.WentWell |> List.map (fun w -> sprintf "  - %s" w) |> String.concat "\n")
        (pm.WentPoorly |> List.map (fun w -> sprintf "  - %s" w) |> String.concat "\n")
        (pm.ActionItems |> List.map (fun ai -> sprintf "  - [%s] %s (Owner: %s, Due: %s)" ai.Priority ai.Description ai.Owner (ai.DueDate.ToString("yyyy-MM-dd"))) |> String.concat "\n")
```

---

## 11. Production Monitoring Dashboard

### Key Metrics ที่ต้อง Monitor

```fsharp
// ===== Golden Signals =====

(*
1. Latency - เวลาที่ใช้ handle request
   - P50, P95, P99 response times
   - ต้องแยก successful vs failed requests

2. Traffic - จำนวน demand บน system
   - Requests per second
   - Active users
   - Data throughput

3. Errors - rate ของ requests ที่ fail
   - 5xx error rate
   - Business error rate (validation, etc.)
   - Slow requests ที่ user ยังรอ

4. Saturation - fullness ของ service
   - CPU utilization
   - Memory usage
   - Disk I/O
   - Network bandwidth
*)

// ===== Alert Rules =====

(*
High Priority Alerts (PagerDuty immediate):
  - Availability < 99%
  - Error rate > 5%
  - P99 latency > 10s
  - Database connections exhausted
  - Health check failing

Medium Priority Alerts (Email/Slack):
  - Error rate > 1%
  - P99 latency > 5s
  - CPU > 80% for 10 minutes
  - Memory > 90%
  - Disk usage > 80%

Low Priority (Slack only):
  - Error rate > 0.5%
  - Slow query detected
  - Certificate expiring in 30 days
*)

// Prometheus alert rules
(*
groups:
  - name: application
    rules:
    - alert: HighErrorRate
      expr: rate(http_requests_total{status=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
      for: 2m
      labels:
        severity: critical
      annotations:
        summary: "High error rate detected"
        description: "Error rate is {{ $value | humanizePercentage }}"
    
    - alert: SlowResponseTime
      expr: histogram_quantile(0.99, rate(http_request_duration_seconds_bucket[5m])) > 10
      for: 5m
      labels:
        severity: warning
      annotations:
        summary: "Slow response time"
        description: "P99 latency is {{ $value }}s"
*)
```

---

## 12. Complete Pre-deployment Checklist

```markdown
## Pre-deployment Checklist

### Code Quality
- [ ] Code review approved by 2+ team members
- [ ] All tests passing (unit, integration, E2E)
- [ ] Code coverage ≥ 80%
- [ ] No critical security vulnerabilities (dependency scan)
- [ ] No linting errors
- [ ] Performance tests passing (no regression)

### Configuration
- [ ] Environment variables configured for production
- [ ] Secrets stored in secret manager (not hardcoded)
- [ ] Feature flags configured correctly
- [ ] Rate limits configured
- [ ] CORS configured correctly

### Database
- [ ] Database migrations tested on staging
- [ ] Rollback migration prepared
- [ ] Database backups verified
- [ ] Indexes created for new queries

### Infrastructure
- [ ] Load balancer health checks updated
- [ ] Kubernetes resource limits set
- [ ] Auto-scaling configured
- [ ] CDN cache rules updated if needed

### Monitoring
- [ ] Alerts configured for new features
- [ ] Dashboard updated
- [ ] Runbook updated
- [ ] On-call engineer notified

### Communication
- [ ] Change request submitted
- [ ] Stakeholders notified
- [ ] Maintenance window scheduled (if needed)
- [ ] Rollback plan documented

### Post-deployment
- [ ] Monitor error rate for 30 minutes
- [ ] Verify key user journeys working
- [ ] Check performance metrics
- [ ] Notify team of successful deployment
```

---

## สรุป (Summary)

Production Checklist สำหรับ F# Applications ครอบคลุม:

1. **Security**: Authentication, authorization, input validation, secrets management
2. **Performance**: Database optimization, caching, async operations
3. **Reliability**: Error handling, circuit breakers, health checks
4. **Observability**: Structured logging, metrics, distributed tracing
5. **Testing**: Unit, integration, E2E tests, performance tests
6. **Deployment**: Zero-downtime deployment, containerization, K8s
7. **Code Quality**: Naming conventions, domain types, composition
8. **SLA/SLO/SLI**: Define and measure service levels
9. **Runbooks**: Operational procedures for common incidents
10. **Post-Mortem**: Blameless learning from incidents
11. **Monitoring**: Golden signals, alert rules

การนำ F# application ขึ้น production ต้องพิจารณาทุกด้านเหล่านี้เพื่อให้บริการมีคุณภาพสูง, ปลอดภัย, และ reliable

---

*จบ Series F# Course ครบ 110 Parts*
