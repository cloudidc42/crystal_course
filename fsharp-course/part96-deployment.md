# Part 96 - การ Deploy (Deployment and DevOps)

## บทนำ

การ Deploy F# applications สามารถทำได้หลายวิธี บทนี้จะครอบคลุมตั้งแต่ Docker, Kubernetes ไปจนถึง CI/CD pipelines และ Cloud deployments

---

## 1. Docker กับ F# Applications

### Dockerfile พื้นฐาน

```dockerfile
# Dockerfile สำหรับ F# / ASP.NET Core application

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

# Copy project files and restore
COPY ["MyApp/MyApp.fsproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.fsproj"

# Copy source and build
COPY . .
WORKDIR "/src/MyApp"
RUN dotnet build "MyApp.fsproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.fsproj" -c Release -o /app/publish \
    --no-restore \
    -p:PublishSingleFile=true \
    -p:SelfContained=false

# Runtime stage (minimal image)
FROM mcr.microsoft.com/dotnet/aspnet:9.0 AS final
WORKDIR /app

# Security: run as non-root
RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

COPY --from=publish /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=60s --retries=3 \
    CMD curl -f http://localhost:8080/health || exit 1

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### Multi-stage Build สำหรับ Self-contained

```dockerfile
# Self-contained single file deployment
FROM mcr.microsoft.com/dotnet/sdk:9.0 AS build
WORKDIR /src

COPY ["MyApp/MyApp.fsproj", "MyApp/"]
RUN dotnet restore "MyApp/MyApp.fsproj" --runtime linux-x64

COPY . .
WORKDIR "/src/MyApp"
RUN dotnet publish "MyApp.fsproj" \
    -c Release \
    -r linux-x64 \
    --self-contained true \
    -p:PublishSingleFile=true \
    -p:PublishTrimmed=true \
    -o /app/publish

# Use minimal base image for self-contained
FROM debian:bullseye-slim AS final
WORKDIR /app

RUN groupadd -r appgroup && useradd -r -g appgroup appuser
USER appuser

COPY --from=build /app/publish .
RUN chmod +x ./MyApp

EXPOSE 8080
ENV ASPNETCORE_URLS=http://+:8080
ENTRYPOINT ["./MyApp"]
```

### .dockerignore

```
**/.git
**/bin
**/obj
**/out
**/.vs
**/.vscode
**/node_modules
**/.env
**/*.user
**/TestResults
**/coverage
Dockerfile*
docker-compose*
**/*.md
```

---

## 2. Docker Compose

```yaml
# docker-compose.yml
version: '3.9'

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
    ports:
      - "8080:8080"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
      - Database__Host=postgres
      - Database__Name=shopdb
      - Database__Username=postgres
      - Database__Password=${DB_PASSWORD}
      - Redis__ConnectionString=redis:6379
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - app-network
    deploy:
      replicas: 2
      resources:
        limits:
          memory: 512M
          cpus: '0.5'
  
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: shopdb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./migrations:/docker-entrypoint-initdb.d
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
  
  redis:
    image: redis:7-alpine
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - app-network
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
    depends_on:
      - api
    networks:
      - app-network

volumes:
  postgres_data:
  redis_data:

networks:
  app-network:
    driver: bridge
```

---

## 3. Kubernetes Basics

```yaml
# kubernetes/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
  namespace: production
  labels:
    app: myapp
    version: "1.0.0"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: myapp
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: myapp
        version: "1.0.0"
    spec:
      serviceAccountName: myapp-sa
      
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
      
      containers:
        - name: myapp
          image: myregistry/myapp:1.0.0
          ports:
            - containerPort: 8080
              name: http
          
          env:
            - name: ASPNETCORE_ENVIRONMENT
              value: "Production"
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: db-password
            - name: REDIS_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: myapp-secrets
                  key: redis-password
          
          envFrom:
            - configMapRef:
                name: myapp-config
          
          resources:
            requests:
              memory: "256Mi"
              cpu: "100m"
            limits:
              memory: "512Mi"
              cpu: "500m"
          
          livenessProbe:
            httpGet:
              path: /health/live
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3
          
          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3
          
          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
      
      volumes:
        - name: config
          configMap:
            name: myapp-config

---
# kubernetes/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: myapp
  namespace: production
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080
      name: http
  type: ClusterIP

---
# kubernetes/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  namespace: production
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
spec:
  tls:
    - hosts:
        - api.example.com
      secretName: myapp-tls
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 80

---
# kubernetes/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: production
data:
  ASPNETCORE_URLS: "http://+:8080"
  Database__Host: "postgres-service"
  Database__Port: "5432"
  Database__Name: "shopdb"
  Redis__Host: "redis-service"
  Redis__Port: "6379"
  Logging__LogLevel__Default: "Information"

---
# kubernetes/hpa.yaml (Horizontal Pod Autoscaler)
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-hpa
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp
  minReplicas: 2
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```

---

## 4. GitHub Actions CI/CD

```yaml
# .github/workflows/ci-cd.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}
  DOTNET_VERSION: '9.0.x'

jobs:
  # Build and Test
  test:
    name: Build and Test
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_DB: testdb
          POSTGRES_USER: test
          POSTGRES_PASSWORD: test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}
      
      - name: Cache NuGet packages
        uses: actions/cache@v4
        with:
          path: ~/.nuget/packages
          key: ${{ runner.os }}-nuget-${{ hashFiles('**/*.fsproj') }}
          restore-keys: |
            ${{ runner.os }}-nuget-
      
      - name: Restore dependencies
        run: dotnet restore
      
      - name: Build
        run: dotnet build --no-restore --configuration Release
      
      - name: Run tests
        run: |
          dotnet test --no-build --configuration Release \
            --logger trx \
            --collect:"XPlat Code Coverage" \
            --results-directory ./TestResults
        env:
          ConnectionStrings__TestDb: "Host=localhost;Port=5432;Database=testdb;Username=test;Password=test"
      
      - name: Upload test results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results
          path: ./TestResults
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./TestResults/**/coverage.cobertura.xml
  
  # Security scan
  security:
    name: Security Scan
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: 'fs'
          scan-ref: '.'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'
      
      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: 'trivy-results.sarif'
  
  # Build Docker image
  build-image:
    name: Build Docker Image
    runs-on: ubuntu-latest
    needs: [test]
    if: github.event_name == 'push'
    
    outputs:
      image-tag: ${{ steps.meta.outputs.tags }}
      image-digest: ${{ steps.build.outputs.digest }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=semver,pattern={{version}}
            type=sha,prefix=sha-
      
      - name: Build and push Docker image
        id: build
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  # Deploy to staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [build-image]
    if: github.ref == 'refs/heads/develop'
    environment: staging
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_STAGING }}
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::8} \
            -n staging
          kubectl rollout status deployment/myapp -n staging
  
  # Deploy to production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [build-image]
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Configure kubectl
        uses: azure/k8s-set-context@v3
        with:
          kubeconfig: ${{ secrets.KUBE_CONFIG_PROD }}
      
      - name: Deploy to Kubernetes
        run: |
          kubectl set image deployment/myapp \
            myapp=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:sha-${GITHUB_SHA::8} \
            -n production
          kubectl rollout status deployment/myapp -n production
      
      - name: Notify Slack
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: "Deployed to production: ${{ github.sha }}"
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
        if: always()
```

---

## 5. Health Checks ใน F#

```fsharp
// Health checks สำหรับ Kubernetes probes

open Microsoft.AspNetCore.Builder
open Microsoft.AspNetCore.Diagnostics.HealthChecks
open Microsoft.Extensions.DependencyInjection
open Microsoft.Extensions.Diagnostics.HealthChecks
open System.Text.Json

// Custom health check
type DatabaseHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context, ct) = task {
            try
                use conn = new Npgsql.NpgsqlConnection(connectionString)
                do! conn.OpenAsync(ct)
                use cmd = conn.CreateCommand()
                cmd.CommandText <- "SELECT 1"
                let! _ = cmd.ExecuteScalarAsync(ct)
                return HealthCheckResult.Healthy("Database connection OK")
            with ex ->
                return HealthCheckResult.Unhealthy("Database connection failed", ex)
        }

type RedisHealthCheck(connectionString: string) =
    interface IHealthCheck with
        member _.CheckHealthAsync(context, ct) = task {
            try
                let connection = StackExchange.Redis.ConnectionMultiplexer.Connect(connectionString)
                let db = connection.GetDatabase()
                let! _ = db.PingAsync()
                return HealthCheckResult.Healthy("Redis connection OK")
            with ex ->
                return HealthCheckResult.Unhealthy("Redis connection failed", ex)
        }

// Configure health checks
let configureHealthChecks (services: IServiceCollection) (config: IConfiguration) =
    services
        .AddHealthChecks()
        .AddCheck<DatabaseHealthCheck>(
            "database",
            tags = [| "db"; "critical" |])
        .AddCheck<RedisHealthCheck>(
            "redis",
            tags = [| "cache" |])
        .AddDiskStorageHealthCheck(
            (fun opts -> opts.AddDrive("/", minimumFreeMegabytes = 512L)),
            "disk_storage",
            tags = [| "infrastructure" |])
    |> ignore

// Configure health check endpoints
let configureHealthEndpoints (app: WebApplication) =
    // Liveness probe - is the process running?
    app.MapHealthChecks("/health/live", HealthCheckOptions(
        Predicate = fun _ -> false))  // no checks, just alive
    |> ignore
    
    // Readiness probe - can it serve traffic?
    app.MapHealthChecks("/health/ready", HealthCheckOptions(
        Predicate = fun check -> check.Tags.Contains("critical"),
        ResponseWriter = fun context report -> task {
            let result = {|
                Status = report.Status.ToString()
                Checks = 
                    report.Entries 
                    |> Seq.map (fun kv -> {|
                        Name = kv.Key
                        Status = kv.Value.Status.ToString()
                        Description = kv.Value.Description
                        Duration = kv.Value.Duration.TotalMilliseconds
                    |})
                    |> Seq.toList
                TotalDuration = report.TotalDuration.TotalMilliseconds
            |}
            context.Response.ContentType <- "application/json"
            do! context.Response.WriteAsync(JsonSerializer.Serialize(result))
        }))
    |> ignore
    
    // Full health check (for monitoring tools)
    app.MapHealthChecks("/health", HealthCheckOptions())
    |> ignore
```

---

## 6. Configuration Management

```fsharp
// Configuration management ใน F#

open Microsoft.Extensions.Configuration
open Microsoft.Extensions.Hosting

// appsettings.json
// {
//   "Application": {
//     "Name": "MyApp",
//     "Version": "1.0.0"
//   },
//   "Database": {
//     "Host": "localhost",
//     "Port": 5432,
//     "Name": "mydb"
//   }
// }

// Strongly-typed configuration
type DatabaseConfig = {
    Host: string
    Port: int
    Name: string
    Username: string
    Password: string
    MaxPoolSize: int
}

type ApplicationConfig = {
    Name: string
    Version: string
    Environment: string
}

type AppSettings = {
    Application: ApplicationConfig
    Database: DatabaseConfig
}

// Configuration loading
let loadConfig (configuration: IConfiguration) =
    let dbSection = configuration.GetSection("Database")
    let appSection = configuration.GetSection("Application")
    
    {
        Application = {
            Name = appSection["Name"] |> Option.ofObj |> Option.defaultValue "MyApp"
            Version = appSection["Version"] |> Option.ofObj |> Option.defaultValue "1.0.0"
            Environment = configuration["ASPNETCORE_ENVIRONMENT"] |> Option.ofObj |> Option.defaultValue "Production"
        }
        Database = {
            Host = dbSection["Host"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Host required")
            Port = dbSection["Port"] |> Option.ofObj |> Option.bind (fun s -> match System.Int32.TryParse(s) with true, n -> Some n | _ -> None) |> Option.defaultValue 5432
            Name = dbSection["Name"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Name required")
            Username = dbSection["Username"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Username required")
            Password = dbSection["Password"] |> Option.ofObj |> Option.defaultWith (fun () -> failwith "Database:Password required")
            MaxPoolSize = dbSection["MaxPoolSize"] |> Option.ofObj |> Option.bind (fun s -> match System.Int32.TryParse(s) with true, n -> Some n | _ -> None) |> Option.defaultValue 20
        }
    }

// Register configuration
let registerConfiguration (services: IServiceCollection) (config: IConfiguration) =
    let appSettings = loadConfig config
    services.AddSingleton(appSettings) |> ignore
    services.AddSingleton(appSettings.Database) |> ignore
    services.AddSingleton(appSettings.Application) |> ignore
```

---

## 7. AWS Lambda กับ F#

```fsharp
// AWS Lambda F# handler

// <PackageReference Include="Amazon.Lambda.Core" Version="2.1.0" />
// <PackageReference Include="Amazon.Lambda.APIGatewayEvents" Version="2.7.0" />
// <PackageReference Include="Amazon.Lambda.Serialization.SystemTextJson" Version="2.4.0" />

open Amazon.Lambda.Core
open Amazon.Lambda.APIGatewayEvents
open System.Text.Json

[<assembly: LambdaSerializer(typeof<Amazon.Lambda.Serialization.SystemTextJson.DefaultLambdaJsonSerializer>)>]
do ()

// Simple Lambda function
let handler (request: APIGatewayProxyRequest) (context: ILambdaContext) : APIGatewayProxyResponse =
    context.Logger.LogInformation($"Processing request: {request.Path}")
    
    let body = {| 
        Message = "Hello from F# Lambda!"
        Path = request.Path
        Method = request.HttpMethod
    |}
    
    APIGatewayProxyResponse(
        StatusCode = 200,
        Body = JsonSerializer.Serialize(body),
        Headers = dict ["Content-Type", "application/json"])

// Async Lambda handler
let asyncHandler (request: APIGatewayProxyRequest) (context: ILambdaContext) =
    async {
        context.Logger.LogInformation($"Processing: {request.Path}")
        
        // Do async work
        do! Async.Sleep 10
        
        let response = {| Status = "ok"; Timestamp = System.DateTime.UtcNow |}
        
        return APIGatewayProxyResponse(
            StatusCode = 200,
            Body = JsonSerializer.Serialize(response))
    }
    |> Async.StartAsTask

// Lambda with DI (using IHostBuilder)
open Microsoft.Extensions.Hosting
open Microsoft.Extensions.DependencyInjection

type FunctionHandler(myService: IMyService) =
    member _.Handle(request: APIGatewayProxyRequest, context: ILambdaContext) =
        task {
            let! result = myService.ProcessAsync(request.Body)
            return APIGatewayProxyResponse(
                StatusCode = 200,
                Body = JsonSerializer.Serialize(result))
        }

// serverless.yml equivalent (for AWS SAM)
// template.yaml:
// AWSTemplateFormatVersion: '2010-09-09'
// Transform: AWS::Serverless-2016-10-31
// 
// Resources:
//   MyFunction:
//     Type: AWS::Serverless::Function
//     Properties:
//       CodeUri: ./
//       Handler: MyApp::MyApp.LambdaHandler::handler
//       Runtime: dotnet8
//       MemorySize: 256
//       Timeout: 30
//       Environment:
//         Variables:
//           ENVIRONMENT: production
//       Events:
//         Api:
//           Type: Api
//           Properties:
//             Path: /api/{proxy+}
//             Method: ANY
```

---

## 8. Azure App Service Deployment

```fsharp
// Azure App Service ด้วย GitHub Actions

// .github/workflows/azure-deploy.yml:
// name: Deploy to Azure App Service
// 
// on:
//   push:
//     branches: [main]
// 
// jobs:
//   deploy:
//     runs-on: ubuntu-latest
//     steps:
//       - uses: actions/checkout@v4
//       
//       - name: Setup .NET
//         uses: actions/setup-dotnet@v4
//         with:
//           dotnet-version: '9.0.x'
//       
//       - name: Build and publish
//         run: |
//           dotnet publish MyApp/MyApp.fsproj \
//             -c Release \
//             -o ./publish
//       
//       - name: Deploy to Azure Web App
//         uses: azure/webapps-deploy@v3
//         with:
//           app-name: 'my-fsharp-app'
//           publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
//           package: './publish'

// Program.fs สำหรับ Azure App Service
module Program =
    open Microsoft.AspNetCore.Builder
    open Microsoft.Extensions.DependencyInjection
    open Microsoft.Extensions.Hosting
    
    [<EntryPoint>]
    let main args =
        let builder = WebApplication.CreateBuilder(args)
        
        // Azure-specific configuration
        builder.Configuration
            .AddEnvironmentVariables()  // Azure App Settings
            .AddAzureAppConfiguration(fun opts ->
                opts.Connect(builder.Configuration.GetConnectionString("AppConfig"))
                opts.ConfigureRefresh(fun refresh ->
                    refresh.Register("Settings:Sentinel", refreshAll = true)
                           .SetCacheExpiration(System.TimeSpan.FromSeconds(30.0))
                    |> ignore)
                |> ignore)
            |> ignore
        
        // Application Insights
        builder.Services.AddApplicationInsightsTelemetry() |> ignore
        
        // Compression
        builder.Services.AddResponseCompression() |> ignore
        
        let app = builder.Build()
        
        if app.Environment.IsDevelopment() then
            app.UseDeveloperExceptionPage() |> ignore
        
        app.UseResponseCompression() |> ignore
        app.UseHttpsRedirection() |> ignore
        app.UseAuthentication() |> ignore
        app.UseAuthorization() |> ignore
        app.MapControllers() |> ignore
        
        app.Run()
        0
```

---

## 9. Helm Charts

```yaml
# helm/Chart.yaml
apiVersion: v2
name: myapp
description: My F# Application
type: application
version: 0.1.0
appVersion: "1.0.0"
```

```yaml
# helm/values.yaml
replicaCount: 2

image:
  repository: ghcr.io/myorg/myapp
  pullPolicy: IfNotPresent
  tag: "latest"

service:
  type: ClusterIP
  port: 80
  targetPort: 8080

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
  hosts:
    - host: api.example.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: myapp-tls
      hosts:
        - api.example.com

resources:
  requests:
    memory: "256Mi"
    cpu: "100m"
  limits:
    memory: "512Mi"
    cpu: "500m"

autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 70

env:
  ASPNETCORE_ENVIRONMENT: Production

secrets:
  DB_PASSWORD: ""
  REDIS_PASSWORD: ""

database:
  host: postgres-service
  port: "5432"
  name: mydb
  username: postgres
```

```yaml
# helm/templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "myapp.fullname" . }}
  labels:
    {{- include "myapp.labels" . | nindent 4 }}
spec:
  {{- if not .Values.autoscaling.enabled }}
  replicas: {{ .Values.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "myapp.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "myapp.selectorLabels" . | nindent 8 }}
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
          env:
            {{- range $key, $value := .Values.env }}
            - name: {{ $key }}
              value: {{ $value | quote }}
            {{- end }}
            - name: Database__Host
              value: {{ .Values.database.host | quote }}
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: {{ include "myapp.fullname" . }}-secrets
                  key: db-password
          resources:
            {{- toYaml .Values.resources | nindent 12 }}
          livenessProbe:
            httpGet:
              path: /health/live
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /health/ready
              port: {{ .Values.service.targetPort }}
            initialDelaySeconds: 10
            periodSeconds: 5
```

---

## สรุป

การ Deploy F# Applications:

1. **Docker**: ใช้ multi-stage builds เพื่อ image ขนาดเล็ก
2. **Docker Compose**: สำหรับ local development และ simple deployments
3. **Kubernetes**: สำหรับ production-scale deployments
4. **Helm**: Package manager สำหรับ Kubernetes
5. **GitHub Actions**: CI/CD automation
6. **Health Checks**: Essential สำหรับ container orchestration
7. **AWS Lambda / Azure App Service**: Serverless และ PaaS options

Best Practices:
- ใช้ non-root user ใน containers
- เพิ่ม health checks
- ใช้ secrets management (ไม่ hardcode secrets)
- ตั้ง resource limits
- Implement graceful shutdown
- ใช้ rolling updates

---

*ต่อไป: Part 97 - Cloud Services กับ F#*
