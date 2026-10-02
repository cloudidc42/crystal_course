# Part 200: Final Project - World-Class Crystal Application

## บทนำ

ในบทสุดท้ายนี้ เราจะสร้าง production-ready Crystal application ที่รวมทุกสิ่งที่เรียนมาตลอด 200 ตอน: REST API, WebSocket, PostgreSQL, Redis, Docker, CI/CD, Monitoring, Security, และ Performance

## Project: TaskFlow - Real-time Task Management API

```
สถาปัตยกรรม:
┌─────────────────────────────────────────────────┐
│                   Client (Web/Mobile)            │
└────────────┬──────────────────────┬─────────────┘
             │ REST API             │ WebSocket
┌────────────▼──────────────────────▼─────────────┐
│              Crystal App (Kemal)                 │
│  - Authentication (JWT)                          │
│  - REST endpoints                                │
│  - WebSocket real-time updates                   │
│  - Rate limiting                                 │
│  - Request logging                               │
└────────┬──────────────┬──────────────────────────┘
         │              │
┌────────▼────┐   ┌─────▼──────┐
│ PostgreSQL  │   │   Redis    │
│ - Users     │   │ - Sessions │
│ - Tasks     │   │ - Cache    │
│ - Projects  │   │ - Pub/Sub  │
└─────────────┘   └────────────┘
```

## Project Structure

```
taskflow/
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── deploy.yml
├── db/
│   └── migrations/
│       ├── 001_create_users.sql
│       ├── 002_create_projects.sql
│       └── 003_create_tasks.sql
├── src/
│   ├── taskflow.cr          # entry point
│   ├── config.cr            # configuration
│   ├── database.cr          # DB connection
│   ├── models/
│   │   ├── user.cr
│   │   ├── project.cr
│   │   └── task.cr
│   ├── handlers/
│   │   ├── auth_handler.cr
│   │   ├── users_handler.cr
│   │   ├── projects_handler.cr
│   │   ├── tasks_handler.cr
│   │   └── websocket_handler.cr
│   ├── middleware/
│   │   ├── auth_middleware.cr
│   │   ├── rate_limiter.cr
│   │   ├── request_logger.cr
│   │   └── security_headers.cr
│   └── services/
│       ├── auth_service.cr
│       ├── task_service.cr
│       └── notification_service.cr
├── spec/
│   ├── spec_helper.cr
│   ├── unit/
│   └── integration/
├── Dockerfile
├── docker-compose.yml
├── shard.yml
└── shard.lock
```

## shard.yml

```yaml
name: taskflow
version: 1.0.0
description: Real-time Task Management API

authors:
  - Developer <dev@example.com>

license: MIT
crystal: ">= 1.10.0"

dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4"
  db:
    github: crystal-lang/crystal-db
    version: "~> 0.13"
  pg:
    github: will/crystal-pg
    version: "~> 0.28"
  redis:
    github: stefanwille/crystal-redis
    version: "~> 2.8"
  jwt:
    github: crystal-community/jwt
    version: "~> 1.6"
  dotenv:
    github: gdotdesign/cr-dotenv
    version: "~> 1.0"

development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5"
  webmock:
    github: manastech/webmock.cr
    version: "~> 0.3"
```

## src/taskflow.cr - Entry Point

```crystal
require "dotenv"
require "kemal"
require "log"

Dotenv.load

require "./config"
require "./database"
require "./models/*"
require "./services/*"
require "./middleware/*"
require "./handlers/*"

# Configure logging
Log.setup do |c|
  level = Config.production? ? Log::Severity::Info : Log::Severity::Debug
  c.bind("*", level, Log::IOBackend.new)
end

# Add middleware
add_handler AuthMiddleware.new
add_handler RateLimiter.new(max_requests: 100, window: 1.minute)
add_handler SecurityHeaders.new
add_handler RequestLogger.new

# Configure Kemal
Kemal.config.port = Config.port
Kemal.config.env = Config.environment

Log.info { "Starting TaskFlow on port #{Config.port}" }
Kemal.run
```

## src/config.cr

```crystal
module Config
  DATABASE_URL = ENV["DATABASE_URL"]? || raise "DATABASE_URL required"
  REDIS_URL    = ENV["REDIS_URL"]? || "redis://localhost:6379"
  JWT_SECRET   = ENV["JWT_SECRET"]? || raise "JWT_SECRET required"
  PORT         = (ENV["PORT"]? || "8080").to_i
  ENVIRONMENT  = ENV["APP_ENV"]? || "development"

  def self.port : Int32
    PORT
  end

  def self.environment : String
    ENVIRONMENT
  end

  def self.production? : Bool
    ENVIRONMENT == "production"
  end

  def self.development? : Bool
    ENVIRONMENT == "development"
  end

  def self.test? : Bool
    ENVIRONMENT == "test"
  end
end
```

## src/models/task.cr

```crystal
require "json"

class Task
  include JSON::Serializable

  enum Status
    Todo
    InProgress
    Done
    Cancelled

    def to_s
      super.downcase.gsub("_", "-")
    end
  end

  enum Priority
    Low
    Medium
    High
    Critical
  end

  property id : Int64?
  property title : String
  property description : String?
  property status : Status = Status::Todo
  property priority : Priority = Priority::Medium
  property project_id : Int64
  property assignee_id : Int64?
  property created_by : Int64
  property due_date : Time?
  property created_at : Time?
  property updated_at : Time?

  def initialize(
    @title, @project_id, @created_by,
    @description = nil, @assignee_id = nil, @due_date = nil,
    @status = Status::Todo, @priority = Priority::Medium
  )
  end

  def self.from_db(row) : Task
    task = Task.new(
      title: row[1].as(String),
      project_id: row[3].as(Int64),
      created_by: row[6].as(Int64)
    )
    task.id = row[0].as(Int64)
    task.description = row[2].as(String?)
    task.status = Status.from_value(row[4].as(Int32))
    task.priority = Priority.from_value(row[5].as(Int32))
    task.assignee_id = row[7].as(Int64?)
    task.due_date = row[8].as(Time?)
    task.created_at = row[9].as(Time?)
    task.updated_at = row[10].as(Time?)
    task
  end
end
```

## src/handlers/tasks_handler.cr

```crystal
require "kemal"

# GET /api/tasks
get "/api/tasks" do |env|
  auth = require_auth(env)
  project_id = env.params.query["project_id"]?.try(&.to_i64)

  tasks = TaskService.list_tasks(
    user_id: auth[:user_id],
    project_id: project_id,
    status: env.params.query["status"]?,
    page: (env.params.query["page"]? || "1").to_i,
    per_page: [(env.params.query["per_page"]? || "20").to_i, 100].min
  )

  env.response.content_type = "application/json"
  {data: tasks, meta: {total: tasks.size}}.to_json
end

# POST /api/tasks
post "/api/tasks" do |env|
  auth = require_auth(env)
  body = parse_json_body(env)

  task = TaskService.create_task(
    title: body["title"].as_s,
    project_id: body["project_id"].as_i64,
    created_by: auth[:user_id],
    description: body["description"]?.try(&.as_s?),
    assignee_id: body["assignee_id"]?.try(&.as_i64?),
    priority: body["priority"]?.try(&.as_s?) || "medium"
  )

  # Real-time notification
  NotificationService.broadcast_task_created(task)

  env.response.status_code = 201
  env.response.content_type = "application/json"
  task.to_json
rescue ex : ValidationError
  halt(env, status_code: 422, response: {error: ex.message}.to_json)
end

# PATCH /api/tasks/:id
patch "/api/tasks/:id" do |env|
  auth = require_auth(env)
  task_id = env.params.url["id"].to_i64
  body = parse_json_body(env)

  task = TaskService.update_task(
    id: task_id,
    user_id: auth[:user_id],
    attrs: body
  )

  NotificationService.broadcast_task_updated(task)

  env.response.content_type = "application/json"
  task.to_json
rescue ex : NotFoundError
  halt(env, status_code: 404, response: {error: "Task not found"}.to_json)
rescue ex : ForbiddenError
  halt(env, status_code: 403, response: {error: "Access denied"}.to_json)
end
```

## src/handlers/websocket_handler.cr

```crystal
# WebSocket สำหรับ real-time updates
class WebSocketManager
  @connections : Hash(Int64, Array(HTTP::WebSocket)) = {} of Int64 => Array(HTTP::WebSocket)
  @mutex = Mutex.new

  def self.instance : self
    @@instance ||= new
  end

  def register(user_id : Int64, socket : HTTP::WebSocket)
    @mutex.synchronize do
      @connections[user_id] ||= [] of HTTP::WebSocket
      @connections[user_id] << socket
    end
    Log.debug { "WebSocket registered for user #{user_id}" }
  end

  def unregister(user_id : Int64, socket : HTTP::WebSocket)
    @mutex.synchronize do
      @connections[user_id]?.try(&.delete(socket))
      @connections.delete(user_id) if @connections[user_id]?.try(&.empty?)
    end
  end

  def broadcast_to_project(project_id : Int64, event : String, data : Hash)
    message = {event: event, data: data}.to_json
    project_user_ids = ProjectRepository.get_member_ids(project_id)

    @mutex.synchronize do
      project_user_ids.each do |user_id|
        @connections[user_id]?.try do |sockets|
          sockets.each do |socket|
            socket.send(message) rescue nil
          end
        end
      end
    end
  end
end

ws_manager = WebSocketManager.instance

ws "/ws" do |socket, env|
  # Authenticate WebSocket connection
  token = env.params.query["token"]?
  claims = token ? TokenService.verify(token) : nil

  unless claims
    socket.send({error: "Unauthorized"}.to_json)
    socket.close
    next
  end

  user_id = claims["user_id"].as_i64
  ws_manager.register(user_id, socket)

  socket.on_message do |msg|
    # Handle ping/pong
    parsed = JSON.parse(msg) rescue nil
    if parsed && parsed["type"]?.try(&.as_s?) == "ping"
      socket.send({type: "pong"}.to_json)
    end
  end

  socket.on_close do
    ws_manager.unregister(user_id, socket)
    Log.debug { "WebSocket closed for user #{user_id}" }
  end
end
```

## Dockerfile

```dockerfile
FROM crystallang/crystal:1.13.0-alpine AS builder

RUN apk add --no-cache \
    openssl-dev openssl-static \
    libpq-dev \
    zlib-dev zlib-static \
    yaml-static

WORKDIR /build
COPY shard.yml shard.lock ./
RUN shards install --production

COPY src/ ./src/

RUN crystal build --release --static \
    --no-debug \
    -o /build/taskflow \
    src/taskflow.cr

FROM alpine:3.19

RUN apk add --no-cache ca-certificates tzdata && \
    addgroup -S app && adduser -S app -G app

WORKDIR /app
COPY --from=builder /build/taskflow .
RUN chown app:app taskflow

USER app
EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD wget -qO- http://localhost:8080/health || exit 1

CMD ["./taskflow"]
```

## docker-compose.yml

```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      DATABASE_URL: postgres://postgres:password@db:5432/taskflow
      REDIS_URL: redis://redis:6379
      JWT_SECRET: ${JWT_SECRET:-change-me-in-production}
      APP_ENV: production
      PORT: "8080"
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: taskflow
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: password
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./db/migrations:/docker-entrypoint-initdb.d:ro
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 3

volumes:
  postgres_data:
  redis_data:
```

## .github/workflows/ci.yml

```yaml
name: CI/CD

on:
  push:
    branches: [main]
    tags: ['v*']
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: taskflow_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports: ["5432:5432"]
        options: --health-cmd pg_isready --health-interval 5s --health-retries 5
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
        options: --health-cmd "redis-cli ping" --health-interval 5s --health-retries 3
    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1
        with:
          crystal: latest
      - name: Cache shards
        uses: actions/cache@v4
        with:
          path: ~/.cache/shards
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.yml') }}
      - run: shards install
      - run: crystal tool format --check
      - run: ./bin/ameba
      - run: crystal spec
        env:
          DATABASE_URL: postgres://postgres:password@localhost:5432/taskflow_test
          REDIS_URL: redis://localhost:6379
          JWT_SECRET: test-secret
          APP_ENV: test

  docker:
    needs: test
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:${{ github.sha }}
```

## แบบฝึกหัด

1. สร้าง TaskFlow ทั้งหมด: REST API + WebSocket + PostgreSQL + Redis แบบ complete
2. เพิ่ม rate limiting ต่างกันสำหรับ auth endpoints vs regular endpoints
3. สร้าง comprehensive test suite รวม unit, integration, และ WebSocket tests
4. Deploy ไปยัง cloud (Railway, Fly.io, หรือ VPS) พร้อม monitoring

## สรุป Final Project

TaskFlow ครอบคลุมทุกแง่มุมของ production Crystal development:

**Architecture**: REST API + WebSocket real-time, microservices-ready

**Database**: PostgreSQL พร้อม migrations, indexes, connection pooling

**Caching**: Redis สำหรับ sessions, rate limiting, pub/sub

**Security**: JWT authentication, CSRF protection, rate limiting, security headers, input validation, parameterized queries

**Performance**: struct สำหรับ value objects, connection pooling, lazy evaluation, pre-allocated collections

**DevOps**: Docker multi-stage build, docker-compose, GitHub Actions CI/CD

**Observability**: structured logging, health check endpoints, Prometheus metrics

**Code Quality**: ameba linting, format checking, comprehensive tests

Crystal เป็นภาษาที่ **fast as C, slick as Ruby** - ใช้งานได้จริงใน production ด้วย binary เล็ก, memory ต่ำ, และ performance สูง เหมาะอย่างยิ่งสำหรับ high-performance web services, CLIs, และ system programming

**ขอแสดงความยินดีที่เรียนจบ 200 ตอนของ Crystal Programming Language!**
