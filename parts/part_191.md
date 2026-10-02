# Part 191: Deployment Strategies ใน Crystal

## บทนำ

การ deploy Crystal applications มีหลายวิธีขึ้นอยู่กับ requirements ด้าน uptime, scaling, และ infrastructure Crystal มีข้อได้เปรียบพิเศษคือ single binary ทำให้ deploy ง่ายมาก

## Binary Deployment

```bash
# Crystal build static binary
crystal build --release --static src/main.cr -o myapp

# Copy ไปยัง server
scp myapp user@server:/app/

# หรือ rsync สำหรับ large files
rsync -avz --progress myapp user@server:/app/

# Run บน server
ssh user@server "/app/myapp"

# Run เป็น background
ssh user@server "nohup /app/myapp > /var/log/myapp.log 2>&1 &"
```

## systemd Service

```ini
# /etc/systemd/system/myapp.service
[Unit]
Description=My Crystal App
After=network.target postgresql.service redis.service
Requires=postgresql.service

[Service]
Type=simple
User=myapp
Group=myapp
WorkingDirectory=/app

# Environment variables
Environment="DATABASE_URL=postgres://myapp:password@localhost:5432/myapp_prod"
Environment="REDIS_URL=redis://localhost:6379"
Environment="PORT=8080"
Environment="APP_ENV=production"
Environment="SECRET_KEY=your-production-secret-key"
EnvironmentFile=-/app/.env  # Optional .env file (- หมายถึง optional)

# Binary path
ExecStart=/app/myapp

# Restart policy
Restart=always
RestartSec=5

# Logging
StandardOutput=journal
StandardError=journal
SyslogIdentifier=myapp

# Security
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
ReadWritePaths=/app/data

[Install]
WantedBy=multi-user.target
```

```bash
# ติดตั้ง systemd service
sudo cp myapp.service /etc/systemd/system/
sudo systemctl daemon-reload
sudo systemctl enable myapp
sudo systemctl start myapp

# ดู status
sudo systemctl status myapp

# ดู logs
sudo journalctl -u myapp -f

# Deploy script สำหรับ systemd
#!/bin/bash
# deploy.sh

set -e

APP_DIR="/app"
APP_NAME="myapp"
SERVICE_NAME="${APP_NAME}.service"
BINARY="dist/myapp"

echo "Deploying ${APP_NAME}..."

# Copy binary
sudo cp "${BINARY}" "${APP_DIR}/${APP_NAME}.new"
sudo chmod +x "${APP_DIR}/${APP_NAME}.new"

# Atomic swap
sudo mv "${APP_DIR}/${APP_NAME}.new" "${APP_DIR}/${APP_NAME}"

# Restart service
sudo systemctl restart "${SERVICE_NAME}"

# Wait and check
sleep 3
if sudo systemctl is-active --quiet "${SERVICE_NAME}"; then
  echo "Deployment successful!"
else
  echo "Deployment failed! Check: journalctl -u ${SERVICE_NAME}"
  exit 1
fi
```

## Rolling Updates

```crystal
# Crystal app ที่รองรับ graceful shutdown สำหรับ rolling updates
require "signal"

class Server
  @server : HTTP::Server
  @shutting_down = false
  @active_requests = Atomic(Int32).new(0)

  def initialize
    @server = HTTP::Server.new([handler])
  end

  def start
    Signal::TERM.trap { graceful_shutdown }
    Signal::INT.trap { graceful_shutdown }

    puts "Server starting on port #{PORT}"
    @server.listen("0.0.0.0", PORT)
  end

  private def handler : HTTP::Handler
    HTTP::Handler.new do |context|
      @active_requests.add(1)
      begin
        if @shutting_down
          context.response.status_code = 503
          context.response.print "Service Unavailable"
          next
        end
        # Handle request...
        process_request(context)
      ensure
        @active_requests.sub(1)
      end
    end
  end

  private def graceful_shutdown
    return if @shutting_down
    @shutting_down = true

    puts "Graceful shutdown initiated..."
    puts "Waiting for #{@active_requests.get} active requests..."

    # รอสูงสุด 30 วินาทีสำหรับ requests ที่ค้างอยู่
    30.times do |i|
      active = @active_requests.get
      break if active == 0
      puts "Still #{active} requests active, waiting... (#{30 - i}s remaining)"
      sleep(1)
    end

    puts "Shutting down server..."
    @server.close
  end

  private def process_request(context : HTTP::Server::Context)
    # Request handling logic
    context.response.print "OK"
  end
end

server = Server.new
server.start
```

## Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

SERVER="user@production-server"
APP_DIR="/app"
NGINX_CONF="/etc/nginx/sites-enabled/myapp"

# ตรวจว่าตอนนี้ใช้ blue หรือ green
CURRENT=$(ssh $SERVER "cat /app/current 2>/dev/null || echo blue")

if [ "$CURRENT" = "blue" ]; then
  DEPLOY_TO="green"
  DEPLOY_PORT="8081"
  CURRENT_PORT="8080"
else
  DEPLOY_TO="blue"
  DEPLOY_PORT="8080"
  CURRENT_PORT="8081"
fi

echo "Deploying to ${DEPLOY_TO} (port ${DEPLOY_PORT})..."

# Deploy ไปยัง inactive slot
ssh $SERVER "
  sudo cp /tmp/myapp ${APP_DIR}/myapp-${DEPLOY_TO}
  sudo chmod +x ${APP_DIR}/myapp-${DEPLOY_TO}
  sudo systemctl restart myapp-${DEPLOY_TO}
  sleep 2
  sudo systemctl is-active myapp-${DEPLOY_TO}
"

# Health check บน new deployment
echo "Health check on ${DEPLOY_TO}..."
for i in $(seq 1 10); do
  STATUS=$(ssh $SERVER "curl -s -o /dev/null -w '%{http_code}' http://localhost:${DEPLOY_PORT}/health")
  if [ "$STATUS" = "200" ]; then
    echo "Health check passed!"
    break
  fi
  echo "Attempt ${i}: status ${STATUS}, retrying..."
  sleep 2
done

# Switch traffic
echo "Switching traffic to ${DEPLOY_TO}..."
ssh $SERVER "
  sudo sed -i 's/proxy_pass http:\/\/localhost:${CURRENT_PORT}/proxy_pass http:\/\/localhost:${DEPLOY_PORT}/' ${NGINX_CONF}
  sudo nginx -t && sudo nginx -s reload
  echo ${DEPLOY_TO} > ${APP_DIR}/current
"

echo "Deployment complete! Now running ${DEPLOY_TO} on port ${DEPLOY_PORT}"
echo "Old ${CURRENT} still running on ${CURRENT_PORT} (can rollback)"
```

## Docker Deployment

```bash
#!/bin/bash
# docker-deploy.sh

IMAGE="registry.example.com/myapp"
TAG="${1:-latest}"
CONTAINER_NAME="myapp"

echo "Deploying ${IMAGE}:${TAG}..."

# Pull new image
docker pull "${IMAGE}:${TAG}"

# Zero-downtime update
docker service update \
  --image "${IMAGE}:${TAG}" \
  --update-parallelism 1 \
  --update-delay 10s \
  --update-failure-action rollback \
  "${CONTAINER_NAME}"

# ตรวจสอบ health
echo "Waiting for service to stabilize..."
sleep 10
docker service ps "${CONTAINER_NAME}"
```

## Deployment Checklist

```crystal
# scripts/pre-deploy.cr - ตรวจสอบก่อน deploy
require "http/client"

class PreDeployChecker
  CHECKS = [
    "Database migrations current",
    "Config values present",
    "Health endpoint responsive",
  ]

  def self.run(env : String)
    puts "=== Pre-Deploy Checks for #{env} ==="
    all_pass = true

    # Check 1: DB Connection
    begin
      DB.open(ENV["DATABASE_URL"]) do |db|
        db.query_one("SELECT 1", as: Int32)
      end
      puts "✓ Database connection OK"
    rescue ex
      puts "✗ Database connection FAILED: #{ex.message}"
      all_pass = false
    end

    # Check 2: Required ENV vars
    required_vars = %w[DATABASE_URL SECRET_KEY PORT]
    required_vars.each do |var|
      if ENV[var]?
        puts "✓ #{var} is set"
      else
        puts "✗ #{var} is MISSING"
        all_pass = false
      end
    end

    unless all_pass
      puts "\nPre-deploy checks FAILED. Aborting."
      exit(1)
    end

    puts "\nAll checks passed. Proceeding with deployment."
  end
end

PreDeployChecker.run(ARGV[0]? || "production")
```

## แบบฝึกหัด

1. สร้าง systemd service file สำหรับ Crystal web server พร้อม environment file
2. Implement graceful shutdown ที่รอให้ active requests เสร็จก่อน shutdown
3. สร้าง blue-green deployment script สำหรับ simple binary app
4. เขียน pre-deploy health check script ที่ตรวจ database, Redis, และ required config

## สรุป

Deployment Strategies สำหรับ Crystal:
- **Static binary**: deploy ง่ายที่สุด scp single file
- **systemd**: production-grade service management, auto-restart, logging
- **Graceful shutdown**: Signal::TERM handler รอ active requests
- **Blue-Green**: zero-downtime ด้วย 2 identical environments
- **Rolling update**: อัปเดตทีละ instance
- **Docker**: containerized deployment, ง่ายสำหรับ scaling
- **Pre-deploy checks**: ตรวจ dependencies ก่อน deploy เสมอ
- **Crystal advantage**: single static binary ไม่ต้อง install runtime
