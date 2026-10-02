# Part 188: Docker ใน Crystal

## บทนำ

Docker ช่วยให้ Crystal applications deploy ได้ง่ายและ reproducible มากขึ้น Crystal มีข้อได้เปรียบพิเศษคือ compile เป็น static binary ทำให้ Docker image เล็กมากได้

## Dockerfile พื้นฐาน

```dockerfile
# Dockerfile - single stage (development)
FROM crystallang/crystal:1.13.0

WORKDIR /app

# Copy dependency files ก่อน
COPY shard.yml shard.lock ./

# Install dependencies
RUN shards install --production

# Copy source
COPY src/ ./src/

# Build application
RUN crystal build --release -o app src/main.cr

# Run
CMD ["./app"]
```

## Multi-Stage Build (Production)

```dockerfile
# Dockerfile - multi-stage build สำหรับ production
# Stage 1: Build
FROM crystallang/crystal:1.13.0-alpine AS builder

# ติดตั้ง build dependencies
RUN apk add --no-cache \
    openssl-dev \
    openssl-static \
    libpq-dev \
    zlib-dev \
    yaml-static

WORKDIR /build

# Copy และ install dependencies
COPY shard.yml shard.lock ./
RUN shards install --production

# Copy source code
COPY src/ ./src/

# Build static binary
RUN crystal build --release --static \
    -o /build/app \
    src/main.cr

# Stage 2: Runtime (เล็กมาก!)
FROM scratch
# หรือ: FROM alpine:3.19 ถ้าต้องการ shell/utilities

# Copy เฉพาะ binary
COPY --from=builder /build/app /app

# Copy config ถ้าจำเป็น
# COPY --from=builder /build/config/ /config/

# เพิ่ม non-root user (security best practice)
# ไม่ได้ใช้ scratch ถ้าต้องการ user management

EXPOSE 8080

ENTRYPOINT ["/app"]
```

## Dockerfile กับ Alpine

```dockerfile
# Dockerfile สำหรับ Crystal + Alpine (เล็กกว่า Debian)
FROM crystallang/crystal:1.13.0-alpine AS builder

RUN apk add --no-cache \
    musl-dev \
    openssl-dev \
    openssl-static \
    zlib-dev \
    zlib-static

WORKDIR /build
COPY shard.yml shard.lock ./
RUN shards install --production

COPY src/ ./src/

RUN crystal build \
    --release \
    --static \
    --no-debug \
    -o /build/server \
    src/server.cr

# Minimal runtime
FROM alpine:3.19

RUN addgroup -S crystal && adduser -S crystal -G crystal

# ca-certificates สำหรับ HTTPS calls
RUN apk add --no-cache ca-certificates tzdata

WORKDIR /app
COPY --from=builder /build/server .
RUN chown crystal:crystal server

USER crystal

EXPOSE 8080
CMD ["./server"]
```

## docker-compose.yml

```yaml
# docker-compose.yml สำหรับ development
version: "3.9"

services:
  app:
    build:
      context: .
      dockerfile: Dockerfile.dev
    ports:
      - "8080:8080"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_dev
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=development_secret
    volumes:
      - ./src:/app/src:ro  # Mount source สำหรับ hot reload
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - app_network

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp_dev
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/init.sql:/docker-entrypoint-initdb.d/init.sql:ro
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app_network

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis_data:/data
    command: redis-server --appendonly yes
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3
    networks:
      - app_network

  # Background job worker
  worker:
    build:
      context: .
      dockerfile: Dockerfile.dev
    command: ["./worker"]
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/myapp_dev
      - REDIS_URL=redis://redis:6379
    depends_on:
      - db
      - redis
    networks:
      - app_network

volumes:
  postgres_data:
  redis_data:

networks:
  app_network:
    driver: bridge
```

## Dockerfile สำหรับ Development

```dockerfile
# Dockerfile.dev - สำหรับ development (hot reload)
FROM crystallang/crystal:1.13.0

RUN apt-get update && apt-get install -y \
    libssl-dev \
    libpq-dev \
    inotify-tools \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /app

COPY shard.yml shard.lock ./
RUN shards install

# Development: mount source จาก host
VOLUME ["/app/src"]

EXPOSE 8080

# Watch and rebuild สำหรับ development
COPY docker-entrypoint-dev.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```bash
#!/bin/bash
# docker-entrypoint-dev.sh

set -e

echo "Starting development server..."

# Initial build
crystal build src/main.cr -o app

# Start and watch for changes
./app &
APP_PID=$!

while inotifywait -r -e modify,create,delete src/; do
    echo "Changes detected, rebuilding..."
    kill $APP_PID 2>/dev/null || true
    crystal build src/main.cr -o app
    ./app &
    APP_PID=$!
done
```

## Crystal Application Code

```crystal
# src/main.cr - web app สำหรับ Docker example
require "kemal"
require "db"
require "pg"

DB_URL = ENV["DATABASE_URL"] rescue "postgres://localhost:5432/myapp"
PORT = (ENV["PORT"]? || "8080").to_i

# Database connection pool
DB_POOL = DB.open(DB_URL)

get "/health" do |env|
  env.response.content_type = "application/json"
  begin
    DB_POOL.query_one("SELECT 1", as: Int32)
    {status: "ok", db: "connected"}.to_json
  rescue ex
    env.response.status_code = 503
    {status: "error", db: ex.message}.to_json
  end
end

get "/api/users" do |env|
  env.response.content_type = "application/json"
  users = DB_POOL.query_all(
    "SELECT id, name, email FROM users ORDER BY id",
    as: {id: Int64, name: String, email: String}
  )
  users.to_json
end

Kemal.config.port = PORT
Kemal.run
```

## แบบฝึกหัด

1. สร้าง multi-stage Docker build สำหรับ Crystal app ที่ใช้ PostgreSQL และวัด image size
2. สร้าง docker-compose.yml ที่ include Crystal app + PostgreSQL + Redis + Nginx
3. เขียน health check endpoint และทดสอบด้วย Docker healthcheck
4. สร้าง Dockerfile สำหรับ background job worker ที่แยกจาก web server

## สรุป

Docker สำหรับ Crystal:
- **Multi-stage build**: builder stage (Alpine + Crystal) + runtime stage (scratch/alpine)
- **--static**: compile เป็น static binary ทำให้ใช้ scratch base image ได้
- **FROM scratch**: image เล็กที่สุด เหมาะสำหรับ static binaries
- **FROM alpine**: เล็กแต่มี shell/utilities สำหรับ debugging
- **docker-compose**: orchestrate multiple services (app, db, redis, worker)
- **healthcheck**: ตรวจสอบ service health ก่อน dependent services start
- **Volumes**: mount source code สำหรับ development hot-reload
- **non-root user**: security best practice ใน production
- **ca-certificates**: ต้องใช้สำหรับ HTTPS calls จาก Alpine/scratch
