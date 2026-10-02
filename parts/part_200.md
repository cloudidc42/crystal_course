# Part 200: Final Project Part 3 - Deployment และ Production

## บทนำ

ยินดีด้วย! นี่คือบทสุดท้ายของคอร์ส Crystal Programming Language ที่ครบถ้วน ในบทนี้เราจะ Deploy BlogCMS ไปยัง Production ด้วย Docker, Nginx, SSL/TLS, CI/CD Pipeline และระบบ Monitoring ที่สมบูรณ์

## Docker Configuration

### Dockerfile สำหรับ Crystal App

```dockerfile
# Dockerfile
# Stage 1: Build
FROM crystallang/crystal:1.13-build AS builder

WORKDIR /app

# คัดลอก dependency files
COPY shard.yml shard.lock ./

# ติดตั้ง dependencies (cached layer)
RUN shards install --production

# คัดลอก source code
COPY src/ ./src/
COPY db/ ./db/

# Build แบบ static binary
RUN crystal build src/blogcms.cr \
    --release \
    --static \
    -o /app/blogcms \
    --no-debug

# Build migration tool
RUN crystal build src/migrate.cr \
    --release \
    --static \
    -o /app/migrate

# Stage 2: Runtime (minimal image)
FROM alpine:3.19

WORKDIR /app

# ติดตั้ง runtime dependencies เท่าที่จำเป็น
RUN apk add --no-cache \
    libssl3 \
    libcrypto3 \
    tzdata \
    ca-certificates \
    curl

# สร้าง non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# คัดลอก binary จาก builder
COPY --from=builder /app/blogcms /app/blogcms
COPY --from=builder /app/migrate /app/migrate
COPY --from=builder /app/db/migrations /app/db/migrations

# สร้างโฟลเดอร์ที่ต้องการ
RUN mkdir -p /app/uploads && chown -R appuser:appgroup /app

USER appuser

# Port ที่ app ใช้
EXPOSE 4000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:4000/health || exit 1

# Start command
CMD ["/app/blogcms"]
```

### Multi-stage Dockerfile สำหรับ Development

```dockerfile
# Dockerfile.dev
FROM crystallang/crystal:1.13

WORKDIR /app

# ติดตั้ง development tools
RUN apt-get update && apt-get install -y \
    postgresql-client \
    redis-tools \
    curl \
    && rm -rf /var/lib/apt/lists/*

# ติดตั้ง crscry (Crystal formatter/linter)
RUN crystal install ameba

COPY shard.yml shard.lock ./
RUN shards install

COPY . .

EXPOSE 4000

# Start ด้วย hot reload (ใช้ watchexec)
CMD ["crystal", "run", "src/blogcms.cr"]
```

## Docker Compose

### docker-compose.yml สำหรับ Production

```yaml
# docker-compose.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile
    image: blogcms:${VERSION:-latest}
    container_name: blogcms_app
    restart: unless-stopped
    environment:
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
      - REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379
      - JWT_SECRET=${JWT_SECRET}
      - PORT=4000
      - UPLOAD_DIR=/app/uploads
      - ALLOWED_ORIGIN=${ALLOWED_ORIGIN:-*}
    volumes:
      - uploads_data:/app/uploads
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 128M

  db:
    image: postgres:16-alpine
    container_name: blogcms_db
    restart: unless-stopped
    environment:
      - POSTGRES_USER=${DB_USER}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_DB=${DB_NAME}
      - PGDATA=/var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init:/docker-entrypoint-initdb.d
    networks:
      - app_network
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 256M

  redis:
    image: redis:7-alpine
    container_name: blogcms_redis
    restart: unless-stopped
    command: >
      redis-server
      --requirepass ${REDIS_PASSWORD}
      --maxmemory 128mb
      --maxmemory-policy allkeys-lru
      --save 60 1
      --loglevel warning
    volumes:
      - redis_data:/data
    networks:
      - app_network
    healthcheck:
      test: ["CMD", "redis-cli", "-a", "${REDIS_PASSWORD}", "ping"]
      interval: 10s
      timeout: 5s
      retries: 3

  nginx:
    image: nginx:alpine
    container_name: blogcms_nginx
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
      - uploads_data:/var/www/uploads:ro
      - nginx_logs:/var/log/nginx
    depends_on:
      - app
    networks:
      - app_network
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  migrate:
    build:
      context: .
      dockerfile: Dockerfile
    container_name: blogcms_migrate
    command: /app/migrate
    environment:
      - DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}
    depends_on:
      db:
        condition: service_healthy
    networks:
      - app_network
    restart: "no"

  # Monitoring
  prometheus:
    image: prom/prometheus:latest
    container_name: blogcms_prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      - app_network
    restart: unless-stopped

  grafana:
    image: grafana/grafana:latest
    container_name: blogcms_grafana
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_USER:-admin}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD}
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
      - ./monitoring/grafana/datasources:/etc/grafana/provisioning/datasources
    ports:
      - "3001:3000"
    networks:
      - app_network
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
  uploads_data:
  nginx_logs:
  prometheus_data:
  grafana_data:

networks:
  app_network:
    driver: bridge
```

### docker-compose.dev.yml

```yaml
# docker-compose.dev.yml
version: '3.9'

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - /app/lib  # ไม่ override lib directory
    environment:
      - DATABASE_URL=postgres://postgres:devpassword@db:5432/blogcms_dev
      - REDIS_URL=redis://redis:6379
      - JWT_SECRET=dev-secret-not-for-production
      - PORT=4000
    ports:
      - "4000:4000"

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=devpassword
      - POSTGRES_DB=blogcms_dev
    ports:
      - "5432:5432"
    volumes:
      - dev_postgres:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  dev_postgres:
```

## Nginx Configuration

### Main Nginx Config

```nginx
# nginx/nginx.conf
user nginx;
worker_processes auto;
worker_rlimit_nofile 65535;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    
    # Logging
    log_format main '$remote_addr - $remote_user [$time_local] '
                    '"$request" $status $body_bytes_sent '
                    '"$http_referer" "$http_user_agent" '
                    '$request_time';
    
    access_log /var/log/nginx/access.log main buffer=16k flush=5s;
    error_log  /var/log/nginx/error.log warn;
    
    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    keepalive_requests 1000;
    
    # Compression
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_types text/plain text/css application/json application/javascript
               text/xml application/xml image/svg+xml;
    
    # Security headers
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;
    
    # Hide server version
    server_tokens off;
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=100r/m;
    limit_req_zone $binary_remote_addr zone=auth:10m rate=10r/m;
    
    # File upload size
    client_max_body_size 10M;
    
    include /etc/nginx/conf.d/*.conf;
}
```

### Virtual Host Config

```nginx
# nginx/conf.d/blogcms.conf
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    
    # Redirect HTTP to HTTPS
    return 301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    server_name yourdomain.com www.yourdomain.com;
    
    # SSL Configuration
    ssl_certificate     /etc/nginx/ssl/yourdomain.com.crt;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.com.key;
    
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:50m;
    ssl_session_tickets off;
    
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;
    
    # HSTS
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
    
    # CSP
    add_header Content-Security-Policy "default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'; img-src 'self' data: https:; font-src 'self';" always;
    
    root /var/www/html;
    
    # API proxy
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        
        proxy_pass http://app:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_cache_bypass $http_upgrade;
        
        proxy_connect_timeout 10s;
        proxy_send_timeout 30s;
        proxy_read_timeout 30s;
        
        proxy_buffering on;
        proxy_buffer_size 4k;
        proxy_buffers 8 4k;
    }
    
    # Auth endpoints - stricter rate limit
    location /api/auth/ {
        limit_req zone=auth burst=5 nodelay;
        
        proxy_pass http://app:4000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
    
    # Uploaded files
    location /uploads/ {
        alias /var/www/uploads/;
        expires 30d;
        add_header Cache-Control "public, immutable";
        
        # Security: ป้องกัน execution
        location ~* \.(php|cgi|pl|py)$ {
            deny all;
        }
    }
    
    # Health check
    location /health {
        proxy_pass http://app:4000;
        access_log off;
    }
    
    # Frontend (SPA)
    location / {
        try_files $uri $uri/ /index.html;
        expires 1h;
        add_header Cache-Control "public, must-revalidate";
    }
    
    # Assets caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
    }
}
```

## Environment Configuration

### .env.example

```bash
# .env.example - Copy to .env and fill in values

# Application
APP_ENV=production
PORT=4000
JWT_SECRET=change-this-to-a-long-random-string-in-production
ALLOWED_ORIGIN=https://yourdomain.com

# Database
DB_USER=blogcms
DB_PASSWORD=change-this-password
DB_NAME=blogcms_production
DATABASE_URL=postgres://${DB_USER}:${DB_PASSWORD}@db:5432/${DB_NAME}

# Redis
REDIS_PASSWORD=change-this-redis-password
REDIS_URL=redis://:${REDIS_PASSWORD}@redis:6379

# File Upload
UPLOAD_DIR=/app/uploads
MAX_FILE_SIZE=10485760  # 10MB in bytes

# Email (optional)
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=your@email.com
SMTP_PASSWORD=app-specific-password
FROM_EMAIL=noreply@yourdomain.com

# Monitoring
GRAFANA_USER=admin
GRAFANA_PASSWORD=change-this-grafana-password

# Backup (optional)
BACKUP_S3_BUCKET=my-backups
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
```

## CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: blogcms_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: "1.13.0"
      
      - name: Cache shards
        uses: actions/cache@v3
        with:
          path: ~/.cache/shards
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.lock') }}
          restore-keys: |
            ${{ runner.os }}-shards-
      
      - name: Install dependencies
        run: shards install
      
      - name: Format check
        run: crystal tool format --check
      
      - name: Run ameba (linter)
        run: bin/ameba
        continue-on-error: true
      
      - name: Run migrations
        env:
          TEST_DATABASE_URL: postgres://postgres:testpassword@localhost/blogcms_test
        run: crystal run src/migrate.cr -- up
      
      - name: Run tests
        env:
          TEST_DATABASE_URL: postgres://postgres:testpassword@localhost/blogcms_test
          TEST_REDIS_URL: redis://localhost:6379/1
        run: crystal spec --order random
  
  build:
    name: Build Docker Image
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=sha,prefix={{branch}}-
            type=raw,value=latest,enable={{is_default_branch}}
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
  
  deploy:
    name: Deploy to Production
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    environment: production
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.PROD_HOST }}
          username: ${{ secrets.PROD_USER }}
          key: ${{ secrets.PROD_SSH_KEY }}
          script: |
            cd /opt/blogcms
            
            # ดึง image ใหม่
            docker-compose pull app
            
            # รัน migrations
            docker-compose run --rm migrate
            
            # Deploy ด้วย zero-downtime
            docker-compose up -d --no-deps --scale app=2 app
            sleep 15
            docker-compose up -d --no-deps --scale app=1 app
            
            # ล้าง old images
            docker image prune -f
            
            echo "Deploy สำเร็จ! เวอร์ชัน: ${{ github.sha }}"
      
      - name: Notify Slack
        if: always()
        uses: rtCamp/action-slack-notify@v2
        env:
          SLACK_WEBHOOK: ${{ secrets.SLACK_WEBHOOK }}
          SLACK_STATUS: ${{ job.status }}
          SLACK_TITLE: "BlogCMS Deploy"
          SLACK_MESSAGE: "Deploy ${{ job.status == 'success' && 'สำเร็จ' || 'ล้มเหลว' }}: ${{ github.sha }}"
```

## Monitoring

### Prometheus Configuration

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'blogcms'
    static_configs:
      - targets: ['app:4000']
    metrics_path: '/metrics'
    scheme: http
  
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres_exporter:9187']
  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis_exporter:9121']
  
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx_exporter:9113']
  
  - job_name: 'node'
    static_configs:
      - targets: ['node_exporter:9100']

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - /etc/prometheus/alerts/*.yml
```

### Alert Rules

```yaml
# monitoring/alerts/app.yml
groups:
  - name: blogcms_alerts
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.1
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate สูงผิดปกติ"
          description: "Error rate: {{ $value | humanize }}/s"
      
      - alert: HighLatency
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Response time ช้า (P95 > 2s)"
      
      - alert: DatabaseDown
        expr: up{job="postgres"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "PostgreSQL หยุดทำงาน!"
      
      - alert: RedisDown
        expr: up{job="redis"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Redis หยุดทำงาน!"
      
      - alert: HighMemoryUsage
        expr: container_memory_usage_bytes{name="blogcms_app"} / container_spec_memory_limit_bytes > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Memory usage สูง (>90%)"
```

### Metrics ใน Crystal App

```crystal
# src/metrics/metrics.cr
require "http/server"

module Metrics
  @@request_count = {} of String => Int64
  @@request_duration = {} of String => Array(Float64)
  @@mutex = Mutex.new
  
  def self.increment(metric : String, labels : Hash(String, String) = {} of String => String)
    key = "#{metric}{#{labels.map { |k, v| "#{k}=\"#{v}\"" }.join(",")}}"
    @@mutex.synchronize { @@request_count[key] = (@@request_count[key]? || 0_i64) + 1 }
  end
  
  def self.observe(metric : String, value : Float64, labels : Hash(String, String) = {} of String => String)
    key = "#{metric}{#{labels.map { |k, v| "#{k}=\"#{v}\"" }.join(",")}}"
    @@mutex.synchronize do
      @@request_duration[key] ||= [] of Float64
      @@request_duration[key] << value
      # เก็บแค่ 1000 samples ล่าสุด
      @@request_duration[key] = @@request_duration[key].last(1000)
    end
  end
  
  def self.to_prometheus_format : String
    lines = [] of String
    
    @@request_count.each do |key, count|
      lines << "# TYPE #{key.split("{").first} counter"
      lines << "#{key} #{count}"
    end
    
    @@request_duration.each do |key, values|
      metric_name = key.split("{").first
      lines << "# TYPE #{metric_name} summary"
      
      sorted = values.sort
      count = sorted.size
      
      if count > 0
        p50 = sorted[(count * 0.5).to_i]
        p90 = sorted[(count * 0.9).to_i]
        p99 = sorted[(count * 0.99).to_i]
        sum = sorted.sum
        
        labels_part = key.includes?("{") ? ",#{key[key.index("{")...-1]}" : ""
        
        lines << "#{metric_name}{quantile=\"0.5\"#{labels_part}} #{p50}"
        lines << "#{metric_name}{quantile=\"0.9\"#{labels_part}} #{p90}"
        lines << "#{metric_name}{quantile=\"0.99\"#{labels_part}} #{p99}"
        lines << "#{metric_name}_count #{count}"
        lines << "#{metric_name}_sum #{sum}"
      end
    end
    
    lines.join("\n")
  end
end

# Metrics middleware
class MetricsMiddleware < Kemal::Handler
  def call(context : HTTP::Server::Context)
    start_time = Time.monotonic
    
    call_next(context)
    
    elapsed = (Time.monotonic - start_time).total_seconds
    status = context.response.status_code.to_s
    method = context.request.method
    path = normalize_path(context.request.path)
    
    Metrics.increment(
      "http_requests_total",
      {"method" => method, "path" => path, "status" => status}
    )
    
    Metrics.observe(
      "http_request_duration_seconds",
      elapsed,
      {"method" => method, "path" => path}
    )
  end
  
  private def normalize_path(path : String) : String
    # Normalize paths ที่มี ID เพื่อ group metrics
    path
      .gsub(/\/\d+/, "/:id")
      .gsub(/\/[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}/, "/:uuid")
  end
end
```

## Production Checklist

### Security Checklist

```bash
#!/bin/bash
# scripts/security_check.sh
echo "=== Security Checklist ==="

# 1. ตรวจสอบ Environment Variables
check_env() {
  local var=$1
  if [ -z "${!var}" ]; then
    echo "FAIL: $var ไม่ได้ตั้งค่า"
    return 1
  fi
  echo "OK: $var ตั้งค่าแล้ว"
}

check_env "JWT_SECRET"
check_env "DB_PASSWORD"
check_env "REDIS_PASSWORD"

# 2. ตรวจสอบ JWT Secret ความยาว
if [ ${#JWT_SECRET} -lt 32 ]; then
  echo "FAIL: JWT_SECRET สั้นเกินไป (ต้องการ 32+ ตัวอักษร)"
else
  echo "OK: JWT_SECRET ความยาวเหมาะสม"
fi

# 3. ตรวจสอบว่าไม่ใช้ default passwords
if [ "$DB_PASSWORD" = "password" ] || [ "$DB_PASSWORD" = "changeme" ]; then
  echo "FAIL: DB_PASSWORD เป็น default value!"
fi

# 4. ตรวจสอบ SSL certificate
if [ -f "/etc/nginx/ssl/yourdomain.com.crt" ]; then
  expiry=$(openssl x509 -enddate -noout -in /etc/nginx/ssl/yourdomain.com.crt | cut -d= -f2)
  echo "OK: SSL Certificate หมดอายุ: $expiry"
else
  echo "FAIL: ไม่พบ SSL Certificate"
fi

echo "=== Security Check เสร็จสิ้น ==="
```

### Deployment Script

```bash
#!/bin/bash
# scripts/deploy.sh
set -e

echo "=== เริ่ม Deploy BlogCMS ==="

# ตรวจสอบ git status
if [[ -n $(git status -s) ]]; then
  echo "ERROR: มี uncommitted changes"
  exit 1
fi

# ดึง tag version
VERSION=$(git describe --tags --always)
echo "Deploy เวอร์ชัน: $VERSION"

# Build image
echo "Building Docker image..."
docker build -t blogcms:$VERSION -t blogcms:latest .

# Tag สำหรับ registry
docker tag blogcms:latest ghcr.io/yourusername/blogcms:$VERSION
docker tag blogcms:latest ghcr.io/yourusername/blogcms:latest

# Push to registry
echo "Pushing to registry..."
docker push ghcr.io/yourusername/blogcms:$VERSION
docker push ghcr.io/yourusername/blogcms:latest

# Deploy to server
echo "Deploying to server..."
ssh deploy@yourserver.com << EOF
  cd /opt/blogcms
  
  # Backup database
  docker exec blogcms_db pg_dump -U \$DB_USER \$DB_NAME > /backup/blogcms_\$(date +%Y%m%d_%H%M%S).sql
  
  # Pull new image
  docker-compose pull app
  
  # Run migrations
  docker-compose run --rm migrate
  
  # Rolling deployment
  docker-compose up -d --no-deps app
  
  # Verify deployment
  sleep 10
  curl -f http://localhost:4000/health || { echo "Health check ล้มเหลว!"; exit 1; }
  
  echo "Deploy สำเร็จ!"
EOF

echo "=== Deploy เสร็จสิ้น ==="
```

## Backup Strategy

### Automated Backup

```bash
#!/bin/bash
# scripts/backup.sh - รัน daily ด้วย cron: 0 2 * * *
BACKUP_DIR="/backup"
DATE=$(date +%Y%m%d_%H%M%S)
RETENTION_DAYS=30

echo "=== เริ่ม Backup ==="

# Database backup
echo "Backup database..."
docker exec blogcms_db pg_dump \
  -U $DB_USER \
  -Fc \
  $DB_NAME > "$BACKUP_DIR/db_$DATE.dump"

# Compress และ encrypt
gzip "$BACKUP_DIR/db_$DATE.dump"
openssl enc -aes-256-cbc -pbkdf2 \
  -in "$BACKUP_DIR/db_$DATE.dump.gz" \
  -out "$BACKUP_DIR/db_$DATE.dump.gz.enc" \
  -pass env:BACKUP_ENCRYPTION_KEY
rm "$BACKUP_DIR/db_$DATE.dump.gz"

# Upload files backup
echo "Backup uploaded files..."
tar czf "$BACKUP_DIR/uploads_$DATE.tar.gz" /app/uploads/

# Upload to S3
if command -v aws &> /dev/null; then
  aws s3 cp "$BACKUP_DIR/db_$DATE.dump.gz.enc" \
    "s3://$BACKUP_S3_BUCKET/database/"
  aws s3 cp "$BACKUP_DIR/uploads_$DATE.tar.gz" \
    "s3://$BACKUP_S3_BUCKET/uploads/"
  echo "อัปโหลด backup ไป S3 สำเร็จ"
fi

# ลบ backups เก่าเกิน retention period
find "$BACKUP_DIR" -name "*.dump.gz.enc" -mtime +$RETENTION_DAYS -delete
find "$BACKUP_DIR" -name "*.tar.gz" -mtime +$RETENTION_DAYS -delete

echo "=== Backup เสร็จสิ้น ==="
```

## Production Checklist สรุป

```
Pre-Deployment Checklist:
========================
[ ] ตั้งค่า Environment Variables ทั้งหมด
[ ] สร้าง SSL/TLS certificate (Let's Encrypt หรือ commercial)
[ ] เปลี่ยน default passwords ทั้งหมด
[ ] ทดสอบ migrations ใน staging environment
[ ] รัน test suite ทั้งหมดผ่าน
[ ] ตรวจสอบ Dockerfile ไม่มี secrets
[ ] ตั้งค่า firewall rules
[ ] ตั้งค่า backup schedule

Post-Deployment Checklist:
==========================
[ ] ตรวจสอบ health check endpoint (/health)
[ ] ตรวจสอบ logs ไม่มี errors
[ ] ทดสอบ login/register
[ ] ทดสอบ file upload
[ ] ตรวจสอบ SSL grade (ssllabs.com)
[ ] ตั้งค่า monitoring alerts
[ ] ทดสอบ backup restore procedure

Ongoing Maintenance:
===================
[ ] อัปเดต Crystal version ทุก release
[ ] อัปเดต shards ทุกสัปดาห์
[ ] ตรวจสอบ security advisories
[ ] Review logs ทุกวัน
[ ] Test backups ทุกเดือน
[ ] Load testing ทุกไตรมาส
```

## สรุป Course ทั้งหมด

### สิ่งที่ได้เรียนในคอร์สนี้

```
Part 1-20:     พื้นฐาน Crystal - Types, Variables, Control Flow
Part 21-40:    Functions, Closures, Blocks, Procs
Part 41-60:    Object-Oriented Programming - Classes, Modules, Mixins
Part 61-80:    Generics, Macros, Metaprogramming
Part 81-100:   Concurrency - Fibers, Channels, Async
Part 101-120:  Standard Library - JSON, HTTP, File I/O
Part 121-140:  Web Development - Kemal, Lucky Framework
Part 141-160:  Databases - PostgreSQL, MySQL, MongoDB
Part 161-180:  Advanced Topics - Optimization, Profiling
Part 181-196:  Ecosystem - Testing, CLI, Deployment
Part 197:      Crystal Ecosystem และ Community
Part 198-200:  Final Project - BlogCMS Full-Stack App
```

### ก้าวต่อไปหลังจบคอร์ส

```
1. Contribute to Crystal ecosystem
   - สร้าง shard ของคุณเอง
   - Report bugs และ fix issues ใน crystal-lang/crystal
   - เขียน blog post เกี่ยวกับ Crystal

2. โปรเจกต์ที่แนะนำ
   - สร้าง CLI tool ที่ใช้จริง
   - สร้าง REST API service
   - สร้าง real-time chat application ด้วย WebSocket
   - สร้าง Discord/Telegram bot

3. ทรัพยากรเพิ่มเติม
   - crystal-lang.org/docs
   - forum.crystal-lang.org
   - awesome-crystal (GitHub)
   - DEV.to Crystal tag

4. เชื่อมต่อ Community
   - Discord: discord.gg/YS7YvQy
   - Twitter: @CrystalLanguage
   - GitHub: github.com/crystal-lang
```

## สุดท้าย

ขอแสดงความยินดีที่เรียนจบคอร์ส Crystal Programming Language ครบ 200 บท! คุณได้เรียนรู้ภาษา Crystal อย่างครบถ้วนตั้งแต่พื้นฐานไปจนถึงการ Deploy Production Application

Crystal เป็นภาษาที่มีอนาคตสดใส มีความเร็วระดับ C/Go แต่มีไวยากรณ์ที่สวยงามคล้าย Ruby การเรียนรู้ Crystal จะช่วยให้คุณเขียน code ที่มีประสิทธิภาพสูง ปลอดภัยจาก type errors และ null pointer exceptions

**ขอให้สนุกกับการเขียน Crystal!** 🔮

---

*คอร์สนี้ครอบคลุม Crystal Programming Language อย่างครบถ้วน ตั้งแต่ Hello World จนถึง Production Deployment*
