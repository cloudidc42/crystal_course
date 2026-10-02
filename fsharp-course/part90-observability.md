# Part 90 - Observability กับ F#

## บทนำ (Introduction)

Observability คือความสามารถในการเข้าใจว่าระบบกำลังทำอะไรอยู่จากภายนอก โดยมี Three Pillars:

1. **Logs** - เหตุการณ์ที่เกิดขึ้น (What happened?)
2. **Metrics** - ตัวเลขวัดผล (How much/many/often?)
3. **Traces** - เส้นทางของ request (How long/where?)

## 1. Serilog Structured Logging

```fsharp
// ===== NuGet Packages =====
// Serilog
// Serilog.Sinks.Console
// Serilog.Sinks.File
// Serilog.Sinks.Seq
// Serilog.AspNetCore
// Serilog.Extensions.Logging
// Serilog.Enrichers.Thread
// Serilog.Enrichers.Environment
// Serilog.Enrichers.Process

open Serilog
open Serilog.Events
open System

// ===== Basic Serilog Setup =====
let configureSerilog () =
    LoggerConfiguration()
        .MinimumLevel.Debug()
        .MinimumLevel.Override("Microsoft", LogEventLevel.Warning)
        .MinimumLevel.Override("System", LogEventLevel.Warning)
        
        // Enrichers - add context to every log
        .Enrich.FromLogContext()
        .Enrich.WithThreadId()
        .Enrich.WithProcessId()
        .Enrich.WithEnvironmentName()
        .Enrich.WithMachineName()
        
        // Sinks - where to write logs
        .WriteTo.Console(
            outputTemplate = "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}")
        .WriteTo.File(
            path = "logs/app-.log",
            rollingInterval = RollingInterval.Day,
            outputTemplate = "{Timestamp:yyyy-MM-dd HH:mm:ss.fff} [{Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}",
            retainedFileCountLimit = 30)
        
        .CreateLogger()

// ===== Structured Logging Examples =====
let demonstrateStructuredLogging (logger: ILogger) =
    
    // Simple message
    logger.Information("Application started at {StartTime}", DateTime.UtcNow)
    
    // Structured data - use properties
    logger.Information("User {UserId} logged in from {IpAddress}", "U001", "192.168.1.1")
    
    // Complex objects (destructuring with @)
    let user = {| Id = "U001"; Name = "Alice"; Email = "alice@example.com" |}
    logger.Information("User details: {@User}", user)
    
    // Warning
    logger.Warning("Low disk space: {FreeSpaceGB}GB remaining on {Drive}", 1.5, "C:")
    
    // Error with exception
    try
        failwith "Something went wrong"
    with ex ->
        logger.Error(ex, "Failed to process request {RequestId}", "REQ-123")
    
    // Debug (only in debug builds)
    logger.Debug("Processing item {ItemIndex} of {TotalItems}", 5, 100)
    
    // Verbose (most detailed)
    logger.Verbose("Cache hit for key {CacheKey}", "user:U001")

// ===== Log Levels =====
let logLevelExamples (logger: ILogger) =
    logger.Verbose("Verbose: Most detailed, tracking flow")
    logger.Debug("Debug: Development troubleshooting")
    logger.Information("Information: Normal events")
    logger.Warning("Warning: Unexpected but handled situation")
    logger.Error("Error: Failures in processing")
    logger.Fatal("Fatal: Application crash imminent")
```

## 2. Serilog Enrichers

```fsharp
// ===== Custom Enricher =====
open Serilog.Core

type CorrelationIdEnricher() =
    interface ILogEventEnricher with
        member _.Enrich(logEvent: LogEvent, propertyFactory: ILogEventPropertyFactory) =
            // Get correlation ID from AsyncLocal or similar
            let correlationId = 
                System.Threading.Thread.CurrentThread.Name 
                |> Option.ofObj
                |> Option.defaultValue (Guid.NewGuid().ToString("N")[..7])
            
            let property = propertyFactory.CreateProperty("CorrelationId", correlationId)
            logEvent.AddPropertyIfAbsent(property)

type RequestIdEnricher(httpContext: Microsoft.AspNetCore.Http.IHttpContextAccessor) =
    interface ILogEventEnricher with
        member _.Enrich(logEvent, propertyFactory) =
            match httpContext.HttpContext with
            | null -> ()
            | ctx ->
                let traceId = ctx.TraceIdentifier
                logEvent.AddPropertyIfAbsent(propertyFactory.CreateProperty("RequestId", traceId))

// ===== LogContext for per-request enrichment =====
let demonstrateLogContext () =
    let logger = LoggerConfiguration().WriteTo.Console().CreateLogger()
    
    // Push properties to context
    use _ = LogContext.PushProperty("OrderId", "ORD-12345")
    use _ = LogContext.PushProperty("UserId", "U001")
    
    // All logs within this scope will have these properties
    logger.Information("Processing order")
    logger.Information("Validating items")
    logger.Information("Order processed successfully")
    
    // Properties are removed when using blocks exit
    logger.Information("Outside scope - no OrderId/UserId")

// ===== Serilog Enricher Configuration =====
let configureWithEnrichers () =
    LoggerConfiguration()
        .Enrich.FromLogContext()
        .Enrich.With<CorrelationIdEnricher>()
        .WriteTo.Console()
        .CreateLogger()
```

## 3. Serilog Sinks

```fsharp
// ===== Multiple Sinks Configuration =====
let configureMultipleSinks () =
    LoggerConfiguration()
        .MinimumLevel.Debug()
        .Enrich.FromLogContext()
        
        // Console sink - colored output
        .WriteTo.Console(
            theme = Serilog.Sinks.SystemConsole.Themes.AnsiConsoleTheme.Code,
            outputTemplate = "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj}{NewLine}{Exception}")
        
        // File sink - rolling daily
        .WriteTo.File(
            path = "logs/myapp-.log",
            rollingInterval = RollingInterval.Day,
            fileSizeLimitBytes = 100L * 1024L * 1024L,  // 100MB max
            rollOnFileSizeLimit = true,
            retainedFileCountLimit = 7,
            buffered = true)
        
        // JSON file sink - for ELK/parsing
        .WriteTo.File(
            formatter = Serilog.Formatting.Json.JsonFormatter(),
            path = "logs/myapp-json-.log",
            rollingInterval = RollingInterval.Day)
        
        // Seq sink (real-time log server)
        // .WriteTo.Seq("http://localhost:5341")
        
        // Different level for different sinks
        .WriteTo.Logger(fun lc ->
            lc.Filter.ByIncludingOnly(fun e -> e.Level >= LogEventLevel.Error)
              .WriteTo.File("logs/errors-.log", rollingInterval = RollingInterval.Day))
        
        .CreateLogger()

// ===== Application Performance Logging =====
let logPerformance (logger: ILogger) (operationName: string) (operation: Async<'T>) =
    async {
        let sw = System.Diagnostics.Stopwatch.StartNew()
        try
            let! result = operation
            sw.Stop()
            logger.Information("{Operation} completed in {ElapsedMs}ms", operationName, sw.ElapsedMilliseconds)
            return result
        with ex ->
            sw.Stop()
            logger.Error(ex, "{Operation} failed after {ElapsedMs}ms", operationName, sw.ElapsedMilliseconds)
            raise
    }
```

## 4. Microsoft.Extensions.Logging Integration

```fsharp
// ===== ASP.NET Core Logging =====
open Microsoft.Extensions.Logging
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.DependencyInjection

// Setup in Program.fs
let configureLogging (builder: WebApplicationBuilder) =
    builder.Host.UseSerilog(fun ctx services configuration ->
        configuration
            .ReadFrom.Configuration(ctx.Configuration)
            .ReadFrom.Services(services)
            .Enrich.FromLogContext()
            .WriteTo.Console()
        |> ignore)

// Using ILogger<T> in services
type OrderService(logger: ILogger<OrderService>) =
    
    member _.PlaceOrder (orderId: string) (customerId: string) =
        async {
            using (logger.BeginScope(dict ["OrderId", orderId :> obj; "CustomerId", customerId :> obj])) (fun _ ->
                logger.LogInformation("Starting order placement")
                
                // Business logic...
                
                logger.LogInformation("Order {OrderId} placed successfully for customer {CustomerId}", 
                                      orderId, customerId))
            
            return Ok orderId
        }

// appsettings.json configuration
(*
{
  "Serilog": {
    "MinimumLevel": {
      "Default": "Information",
      "Override": {
        "Microsoft": "Warning",
        "System": "Warning"
      }
    },
    "WriteTo": [
      {
        "Name": "Console",
        "Args": {
          "outputTemplate": "[{Timestamp:HH:mm:ss} {Level:u3}] {Message:lj} {Properties:j}{NewLine}{Exception}"
        }
      },
      {
        "Name": "File",
        "Args": {
          "path": "logs/app-.log",
          "rollingInterval": "Day"
        }
      }
    ],
    "Enrich": ["FromLogContext", "WithThreadId", "WithMachineName"]
  }
}
*)
```

## 5. OpenTelemetry

```fsharp
// ===== OpenTelemetry Setup =====
// NuGet packages:
// OpenTelemetry
// OpenTelemetry.Extensions.Hosting
// OpenTelemetry.Instrumentation.AspNetCore
// OpenTelemetry.Instrumentation.Http
// OpenTelemetry.Exporter.Jaeger
// OpenTelemetry.Exporter.Otlp

open System.Diagnostics

// Activity Source for this service
let activitySource = new ActivitySource("MyApp.OrderService", "1.0.0")

// ===== Creating Traces =====
let processOrderWithTracing (orderId: string) =
    async {
        // Start a new activity (span)
        use activity = activitySource.StartActivity("ProcessOrder")
        
        // Add tags (attributes)
        activity |> Option.iter (fun a ->
            a.SetTag("order.id", orderId) |> ignore
            a.SetTag("service.name", "order-service") |> ignore)
        
        // Simulate work
        do! Async.Sleep 10
        
        // Child span
        use validateActivity = activitySource.StartActivity("ValidateOrder")
        validateActivity |> Option.iter (fun a ->
            a.SetTag("validation.step", "items") |> ignore)
        
        do! Async.Sleep 5
        
        // Add events to span
        activity |> Option.iter (fun a ->
            a.AddEvent(ActivityEvent("ValidationCompleted")) |> ignore)
        
        // Another child span
        use paymentActivity = activitySource.StartActivity("ProcessPayment")
        paymentActivity |> Option.iter (fun a ->
            a.SetTag("payment.method", "credit_card") |> ignore
            a.SetTag("payment.amount", 99.99) |> ignore)
        
        do! Async.Sleep 20
        
        // Set status on error
        // activity |> Option.iter (fun a -> a.SetStatus(ActivityStatusCode.Error, "Payment failed"))
        
        // Success
        activity |> Option.iter (fun a ->
            a.SetStatus(ActivityStatusCode.Ok) |> ignore)
        
        printfn "Order %s processed with distributed tracing" orderId
        return Ok orderId
    }

// ===== OpenTelemetry Configuration =====
let configureOpenTelemetry (services: IServiceCollection) (serviceName: string) =
    services
        .AddOpenTelemetry()
        .WithTracing(fun builder ->
            builder
                // Service info
                .SetResourceBuilder(
                    OpenTelemetry.Resources.ResourceBuilder.CreateDefault()
                        .AddService(serviceName, serviceVersion = "1.0.0")
                        .AddAttributes([| 
                            System.Collections.Generic.KeyValuePair("deployment.environment", "production" :> obj) 
                        |]))
                
                // Auto-instrumentation
                .AddAspNetCoreInstrumentation(fun opts ->
                    opts.RecordException <- true)
                .AddHttpClientInstrumentation()
                .AddSqlClientInstrumentation(fun opts ->
                    opts.SetDbStatementForText <- true)
                
                // Custom instrumentation
                .AddSource(activitySource.Name)
                
                // Exporters
                .AddJaegerExporter(fun opts ->
                    opts.AgentHost <- "localhost"
                    opts.AgentPort <- 6831)
                
                // OTLP exporter (for Grafana Tempo, etc.)
                // .AddOtlpExporter(fun opts -> opts.Endpoint <- Uri("http://localhost:4317"))
            |> ignore)
    |> ignore
    services
```

## 6. Metrics กับ OpenTelemetry

```fsharp
// ===== Custom Metrics =====
open System.Diagnostics.Metrics

// Create meter for this service
let meter = new Meter("MyApp.OrderService", "1.0.0")

// Define metrics
let ordersPlacedCounter = meter.CreateCounter<int64>(
    "orders.placed",
    unit = "orders",
    description = "Total number of orders placed")

let orderValueHistogram = meter.CreateHistogram<decimal>(
    "order.value",
    unit = "THB",
    description = "Distribution of order values")

let activeOrdersGauge = meter.CreateObservableGauge<int>(
    "orders.active",
    fun () ->
        // Real: query DB
        let count = 42  // Current active orders
        [| Measurement<int>(count) |],
    unit = "orders",
    description = "Currently active orders")

let orderProcessingDuration = meter.CreateHistogram<double>(
    "order.processing.duration",
    unit = "ms",
    description = "Time to process an order")

// ===== Using Metrics =====
let recordOrderPlaced (orderId: string) (amount: decimal) (customerId: string) =
    // Counter - increment
    ordersPlacedCounter.Add(1L, 
        KeyValuePair("customer.tier", "gold"),
        KeyValuePair("payment.method", "credit_card"))
    
    // Histogram - record distribution
    orderValueHistogram.Record(amount,
        KeyValuePair("currency", "THB"))

let measureOrderProcessing (orderId: string) (processOrder: unit -> Async<unit>) =
    async {
        let sw = System.Diagnostics.Stopwatch.StartNew()
        try
            do! processOrder()
            sw.Stop()
            orderProcessingDuration.Record(float sw.ElapsedMilliseconds,
                KeyValuePair("status", "success"))
        with ex ->
            sw.Stop()
            orderProcessingDuration.Record(float sw.ElapsedMilliseconds,
                KeyValuePair("status", "error"))
            raise
    }

// ===== Prometheus Metrics Export =====
let configurePrometheusMetrics (services: IServiceCollection) =
    services
        .AddOpenTelemetry()
        .WithMetrics(fun builder ->
            builder
                .SetResourceBuilder(
                    OpenTelemetry.Resources.ResourceBuilder.CreateDefault()
                        .AddService("order-service"))
                
                // Collect .NET runtime metrics
                .AddRuntimeInstrumentation()
                .AddProcessInstrumentation()
                .AddAspNetCoreInstrumentation()
                .AddHttpClientInstrumentation()
                
                // Custom metrics
                .AddMeter(meter.Name)
                
                // Export to Prometheus
                .AddPrometheusExporter()
        |> ignore)
    |> ignore
    services

// Add Prometheus endpoint in app configuration:
// app.MapPrometheusScrapingEndpoint() // /metrics endpoint
```

## 7. Distributed Tracing

```fsharp
// ===== Distributed Tracing Pattern =====
// Tracing across multiple services

// Trace Context Propagation
open System.Diagnostics
open System.Net.Http

// When making HTTP calls, trace context is automatically propagated
// by OpenTelemetry HttpClient instrumentation

// Manual propagation example
let makeTracedHttpCall (httpClient: HttpClient) (url: string) =
    async {
        use activity = activitySource.StartActivity("HttpCall")
        
        // OpenTelemetry automatically injects trace headers:
        // traceparent: 00-{traceId}-{spanId}-{flags}
        
        try
            let! response = httpClient.GetAsync(url) |> Async.AwaitTask
            
            activity |> Option.iter (fun a ->
                a.SetTag("http.status_code", int response.StatusCode) |> ignore
                a.SetTag("http.url", url) |> ignore)
            
            return response
        with ex ->
            activity |> Option.iter (fun a ->
                a.SetStatus(ActivityStatusCode.Error, ex.Message) |> ignore)
            raise
    }

// Correlating logs with traces
let logWithTraceContext (logger: Serilog.ILogger) (message: string) =
    let currentActivity = Activity.Current
    match currentActivity with
    | null ->
        logger.Information(message)
    | activity ->
        // Add trace context to log
        use _ = Serilog.Context.LogContext.PushProperty("TraceId", activity.TraceId.ToString())
        use _ = Serilog.Context.LogContext.PushProperty("SpanId", activity.SpanId.ToString())
        logger.Information(message)
```

## 8. Health Checks

```fsharp
// ===== Health Checks =====
open Microsoft.Extensions.Diagnostics.HealthChecks
open System.Threading
open System.Threading.Tasks

// Custom health checks
type DatabaseHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(ctx: HealthCheckContext, ct: CancellationToken) =
            task {
                try
                    // Real: try to connect and run a simple query
                    // use conn = new Npgsql.NpgsqlConnection(connectionString)
                    // do! conn.OpenAsync(ct)
                    // let! _ = conn.ExecuteScalarAsync("SELECT 1")
                    
                    return HealthCheckResult.Healthy(
                        "Database connection successful",
                        dict ["server", connectionString.[..30] :> obj])
                with ex ->
                    return HealthCheckResult.Unhealthy(
                        sprintf "Database connection failed: %s" ex.Message,
                        ex)
            }

type RedisHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(ctx, ct) =
            task {
                try
                    // Real: StackExchange.Redis.ConnectionMultiplexer
                    return HealthCheckResult.Healthy("Redis connection OK")
                with ex ->
                    return HealthCheckResult.Degraded(
                        sprintf "Redis unavailable: %s" ex.Message)
            }

type ExternalApiHealthCheck(httpClient: HttpClient, url: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(ctx, ct) =
            task {
                try
                    let sw = Diagnostics.Stopwatch.StartNew()
                    let! response = httpClient.GetAsync(url, ct)
                    sw.Stop()
                    
                    if response.IsSuccessStatusCode then
                        return HealthCheckResult.Healthy(
                            sprintf "API responded in %dms" sw.ElapsedMilliseconds,
                            dict ["responseTime", sw.ElapsedMilliseconds :> obj])
                    else
                        return HealthCheckResult.Degraded(
                            sprintf "API returned %d" (int response.StatusCode))
                with ex ->
                    return HealthCheckResult.Unhealthy(sprintf "API unreachable: %s" ex.Message)
            }

// ===== Health Check Configuration =====
let configureHealthChecks (services: IServiceCollection) =
    services
        .AddHealthChecks()
        .AddCheck<DatabaseHealthCheck>("database", tags = [|"db"; "critical"|])
        .AddCheck<RedisHealthCheck>("redis", tags = [|"cache"|])
        // Built-in checks
        // .AddSqlServer(connectionString, tags = [|"db"|])
        // .AddNpgSql(connectionString)
        // .AddRedis("localhost:6379")
        .AddCheck("self", fun () -> HealthCheckResult.Healthy())
    |> ignore
    services

// Health check endpoints
// app.MapHealthChecks("/health")
// app.MapHealthChecks("/health/ready", HealthCheckOptions(Predicate = fun c -> c.Tags.Contains("critical")))
// app.MapHealthChecks("/health/live", HealthCheckOptions(Predicate = fun _ -> false))

// Custom response format
open Microsoft.AspNetCore.Diagnostics.HealthChecks
open System.Text.Json

let healthCheckResponseWriter (ctx: Microsoft.AspNetCore.Http.HttpContext) (report: HealthReport) =
    task {
        ctx.Response.ContentType <- "application/json"
        let response = {|
            Status = report.Status.ToString()
            Duration = report.TotalDuration.TotalMilliseconds
            Checks = 
                report.Entries 
                |> Seq.map (fun kvp -> {|
                    Name = kvp.Key
                    Status = kvp.Value.Status.ToString()
                    Description = kvp.Value.Description
                    Duration = kvp.Value.Duration.TotalMilliseconds
                    Tags = kvp.Value.Tags
                |}) |> Seq.toList
        |}
        let json = JsonSerializer.Serialize(response)
        do! ctx.Response.WriteAsync(json)
    }
```

## 9. Application Insights

```fsharp
// ===== Application Insights =====
// NuGet: Microsoft.ApplicationInsights.AspNetCore

open Microsoft.ApplicationInsights
open Microsoft.ApplicationInsights.DataContracts
open Microsoft.ApplicationInsights.Extensibility

// Setup in Program.fs
let configureApplicationInsights (services: IServiceCollection) (connectionString: string) =
    services.AddApplicationInsightsTelemetry(fun opts ->
        opts.ConnectionString <- connectionString) |> ignore
    services

// Manual telemetry
type TelemetryService(telemetryClient: TelemetryClient) =
    
    // Track custom event
    member _.TrackOrderPlaced (orderId: string) (amount: decimal) (customerId: string) =
        let properties = dict [
            "OrderId", orderId
            "CustomerId", customerId
        ]
        let metrics = dict [
            "OrderAmount", float amount
        ]
        telemetryClient.TrackEvent("OrderPlaced", properties, metrics)
    
    // Track dependency (external call)
    member _.TrackPaymentGateway (operation: Async<Result<string, string>>) =
        async {
            let startTime = DateTimeOffset.UtcNow
            let sw = Diagnostics.Stopwatch.StartNew()
            
            let! result = operation
            sw.Stop()
            
            let dependency = DependencyTelemetry(
                "HTTP",
                "payment-gateway",
                "ChargeCard",
                startTime,
                sw.Elapsed,
                Result.isOk result)
            
            match result with
            | Error e -> dependency.ResultCode <- e
            | Ok txnId -> dependency.Data <- txnId
            
            telemetryClient.TrackDependency(dependency)
            return result
        }
    
    // Track exception
    member _.TrackException (ex: exn) (properties: Map<string, string>) =
        let props = dict (properties |> Map.toSeq)
        telemetryClient.TrackException(ex, props)
    
    // Track metric
    member _.TrackMetric (name: string) (value: float) =
        telemetryClient.TrackMetric(name, value)
    
    // Flush on shutdown
    member _.Flush () =
        telemetryClient.Flush()
```

## 10. Complete Observability Example

```fsharp
// ===== Complete Observability Setup =====

open System
open System.Diagnostics
open Serilog
open Serilog.Events

// Metrics
let appMeter = new Diagnostics.Metrics.Meter("ShopApp")
let requestCounter = appMeter.CreateCounter<int64>("http.requests.total")
let requestDuration = appMeter.CreateHistogram<double>("http.request.duration.ms")
let errorCounter = appMeter.CreateCounter<int64>("http.errors.total")
let ordersTotal = appMeter.CreateCounter<int64>("orders.total")

// Tracing
let appActivitySource = new ActivitySource("ShopApp")

// Logging
let appLogger = 
    LoggerConfiguration()
        .MinimumLevel.Debug()
        .Enrich.FromLogContext()
        .WriteTo.Console(
            outputTemplate = "[{Timestamp:HH:mm:ss} {Level:u3}] [{TraceId}] {Message:lj}{NewLine}{Exception}")
        .CreateLogger()

// ===== Instrumented Service =====
type InstrumentedOrderService() =
    
    member _.ProcessOrder (orderId: string) (customerId: string) (amount: decimal) =
        async {
            // Start trace
            use activity = appActivitySource.StartActivity("ProcessOrder")
            
            // Add trace context to logs
            let traceId = 
                match activity with
                | null -> "no-trace"
                | a -> a.TraceId.ToString()
            
            use _ = Serilog.Context.LogContext.PushProperty("TraceId", traceId)
            use _ = Serilog.Context.LogContext.PushProperty("OrderId", orderId)
            
            let sw = Diagnostics.Stopwatch.StartNew()
            
            try
                appLogger.Information("Processing order {OrderId} for customer {CustomerId}, amount {Amount}", 
                                       orderId, customerId, amount)
                
                // Set trace attributes
                activity |> Option.iter (fun a ->
                    a.SetTag("order.id", orderId) |> ignore
                    a.SetTag("customer.id", customerId) |> ignore
                    a.SetTag("order.amount", float amount) |> ignore)
                
                // Simulate processing steps
                use validateActivity = appActivitySource.StartActivity("ValidateOrder")
                do! Async.Sleep 10
                appLogger.Debug("Order validation completed")
                
                use paymentActivity = appActivitySource.StartActivity("ProcessPayment")
                paymentActivity |> Option.iter (fun a ->
                    a.SetTag("payment.amount", float amount) |> ignore)
                do! Async.Sleep 50
                appLogger.Information("Payment processed for order {OrderId}", orderId)
                
                use inventoryActivity = appActivitySource.StartActivity("UpdateInventory")
                do! Async.Sleep 20
                appLogger.Debug("Inventory updated")
                
                sw.Stop()
                
                // Record metrics
                ordersTotal.Add(1L,
                    KeyValuePair("status", "success"),
                    KeyValuePair("customer.tier", "regular"))
                requestDuration.Record(float sw.ElapsedMilliseconds,
                    KeyValuePair("operation", "process_order"))
                
                // Mark trace success
                activity |> Option.iter (fun a ->
                    a.SetStatus(ActivityStatusCode.Ok) |> ignore)
                
                appLogger.Information("Order {OrderId} processed successfully in {ElapsedMs}ms", 
                                       orderId, sw.ElapsedMilliseconds)
                
                return Ok orderId
                
            with ex ->
                sw.Stop()
                
                // Log error
                appLogger.Error(ex, "Failed to process order {OrderId} after {ElapsedMs}ms", 
                                    orderId, sw.ElapsedMilliseconds)
                
                // Record error metrics
                errorCounter.Add(1L, KeyValuePair("operation", "process_order"))
                
                // Mark trace failure
                activity |> Option.iter (fun a ->
                    a.SetStatus(ActivityStatusCode.Error, ex.Message) |> ignore)
                
                return Error ex.Message
        }

// ===== Middleware for HTTP Request Observability =====
type RequestObservabilityMiddleware(next: Microsoft.AspNetCore.Http.RequestDelegate, logger: ILogger) =
    
    member _.InvokeAsync(ctx: Microsoft.AspNetCore.Http.HttpContext) =
        task {
            let sw = Diagnostics.Stopwatch.StartNew()
            let requestId = ctx.TraceIdentifier
            
            use _ = Serilog.Context.LogContext.PushProperty("RequestId", requestId)
            use _ = Serilog.Context.LogContext.PushProperty("Method", ctx.Request.Method)
            use _ = Serilog.Context.LogContext.PushProperty("Path", ctx.Request.Path.Value)
            
            logger.Information("HTTP {Method} {Path} started", ctx.Request.Method, ctx.Request.Path)
            
            try
                do! next.Invoke(ctx)
                sw.Stop()
                
                let statusCode = ctx.Response.StatusCode
                
                requestCounter.Add(1L,
                    KeyValuePair("method", ctx.Request.Method),
                    KeyValuePair("status", string statusCode))
                requestDuration.Record(float sw.ElapsedMilliseconds,
                    KeyValuePair("method", ctx.Request.Method))
                
                logger.Information("HTTP {Method} {Path} completed {StatusCode} in {ElapsedMs}ms",
                                    ctx.Request.Method, ctx.Request.Path,
                                    statusCode, sw.ElapsedMilliseconds)
                
            with ex ->
                sw.Stop()
                
                errorCounter.Add(1L,
                    KeyValuePair("method", ctx.Request.Method))
                
                logger.Error(ex, "HTTP {Method} {Path} failed in {ElapsedMs}ms",
                              ctx.Request.Method, ctx.Request.Path, sw.ElapsedMilliseconds)
                raise
        }

// ===== Demo: Run with observability =====
let runObservabilityDemo () =
    async {
        appLogger.Information("=== Observability Demo ===")
        
        let service = InstrumentedOrderService()
        
        // Process several orders
        for i in 1..3 do
            let orderId = sprintf "ORD-%03d" i
            let customerId = sprintf "CUST-%03d" (i % 5 + 1)
            let amount = decimal (i * 100)
            
            let! result = service.ProcessOrder orderId customerId amount
            match result with
            | Ok id -> appLogger.Information("✓ Order {OrderId} done", id)
            | Error e -> appLogger.Error("✗ Order {OrderId} failed: {Error}", orderId, e)
        
        appLogger.Information("All orders processed. Check logs, metrics, and traces.")
        
        // Flush logs
        Log.CloseAndFlush()
    }

runObservabilityDemo() |> Async.RunSynchronously
```

## 11. Grafana Dashboard Configuration

```yaml
# ===== prometheus.yml =====
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'shopapp'
    static_configs:
      - targets: ['localhost:5000']
    metrics_path: '/metrics'
```

```json
// ===== Grafana Dashboard (simplified) =====
{
  "title": "ShopApp Observability",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(http_requests_total[5m])",
          "legendFormat": "{{method}} {{status}}"
        }
      ]
    },
    {
      "title": "Request Duration P95",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, rate(http_request_duration_ms_bucket[5m]))",
          "legendFormat": "P95 latency"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "rate(http_errors_total[5m]) / rate(http_requests_total[5m]) * 100",
          "legendFormat": "Error %"
        }
      ]
    },
    {
      "title": "Orders per Minute",
      "type": "graph",
      "targets": [
        {
          "expr": "rate(orders_total[1m]) * 60"
        }
      ]
    }
  ]
}
```

## 12. สรุป Observability กับ F#

```fsharp
(*
Observability Three Pillars in F#:

1. LOGS - Serilog
   Structured logging with properties
   Multiple sinks: Console, File, Seq, ELK
   Enrichers for automatic context
   Log levels: Verbose → Debug → Info → Warning → Error → Fatal
   
   Key practices:
   - Use structured logging: "User {UserId} logged in" not string concat
   - Enrich with correlation/trace IDs
   - Different levels for different environments
   - Avoid logging sensitive data (passwords, PII)

2. METRICS - OpenTelemetry + Prometheus
   Counters: Total requests, total orders, total errors
   Histograms: Request durations, order values, latency
   Gauges: Active connections, queue depth, memory
   
   Key metrics (RED method):
   - Rate: Requests per second
   - Errors: Error rate/count  
   - Duration: Latency percentiles (P50, P95, P99)

3. TRACES - OpenTelemetry + Jaeger/Zipkin
   Distributed traces across services
   Activities = Spans (parent-child relationships)
   Automatic propagation via HTTP headers
   
   Key practices:
   - Add semantic attributes (http.method, db.statement, etc.)
   - Set appropriate status codes
   - Correlate with logs using trace/span IDs

Tools:
- Serilog: Structured logging
- OpenTelemetry .NET: Standard traces + metrics
- Prometheus: Metrics collection
- Grafana: Visualization
- Jaeger/Zipkin: Trace visualization
- Seq: Log exploration
- ELK Stack: Log aggregation

F# Advantages for Observability:
✓ Immutable data = safer logging (no mutation during log)
✓ DUs = exhaustive case handling = fewer unhandled exceptions
✓ Type system catches bugs before they become incidents
✓ Async workflows integrate naturally with async tracing
*)

printfn "Observability with F# - Complete!"
printfn "Logs + Metrics + Traces = Full Observability"
```

---

**สรุป**: Observability ใน F# ใช้ Serilog สำหรับ structured logging, OpenTelemetry สำหรับ distributed tracing และ metrics, ร่วมกับ Prometheus/Grafana สำหรับ visualization ทำให้เข้าใจพฤติกรรมของระบบใน production ได้อย่างสมบูรณ์
