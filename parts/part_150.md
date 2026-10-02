# Part 150: Full-Stack Web App - แอปพลิเคชันเว็บแบบครบชุด

## บทนำ

ในบทนี้เราจะสร้าง full-stack web application โดยใช้ Kemal + ECR + PostgreSQL + Authentication + REST API + Static Files รวมทุกอย่างที่เรียนมา

## โครงสร้างโปรเจค

```
taskapp/
├── src/
│   ├── app.cr              <- Main application
│   ├── models/
│   │   ├── user.cr
│   │   └── task.cr
│   ├── handlers/           <- HTTP handlers
│   │   ├── auth_handler.cr
│   │   ├── task_handler.cr
│   │   └── api_handler.cr
│   └── middlewares/
│       ├── auth.cr
│       └── security.cr
├── views/
│   ├── layouts/
│   │   └── main.ecr
│   ├── tasks/
│   │   ├── index.ecr
│   │   ├── new.ecr
│   │   └── show.ecr
│   └── auth/
│       ├── login.ecr
│       └── register.ecr
├── public/
│   ├── css/
│   │   └── app.css
│   └── js/
│       └── app.js
└── shard.yml
```

## shard.yml

```yaml
name: taskapp
version: 0.1.0

dependencies:
  kemal:
    github: kemalcr/kemal
    version: ~> 1.4.0
  pg:
    github: will/crystal-pg
    version: ~> 0.28.0
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
  crypto:
    github: crystal-lang/crystal-bcrypt
    version: ~> 0.2.0
```

## Models

```crystal
# src/models/user.cr
require "db"
require "pg"
require "crypto/bcrypt/password"

struct User
  include DB::Serializable
  
  property id : Int64
  property email : String
  property name : String
  property hashed_password : String
  property role : String
  property active : Bool
  property created_at : Time
  
  def admin? : Bool
    role == "admin"
  end
  
  def verify_password(password : String) : Bool
    Crypto::Bcrypt::Password.new(hashed_password).verify(password)
  rescue
    false
  end
  
  def self.find_by_email(db, email : String) : User?
    db.query_one?(
      "SELECT * FROM users WHERE email = $1 AND active = true",
      email,
      as: User
    )
  end
  
  def self.find_by_id(db, id : Int64) : User?
    db.query_one?("SELECT * FROM users WHERE id = $1", id, as: User)
  end
  
  def self.create(db, email : String, name : String, password : String, role : String = "user") : User?
    hashed = Crypto::Bcrypt::Password.create(password).to_s
    db.query_one?(
      <<-SQL,
        INSERT INTO users (email, name, hashed_password, role, active, created_at)
        VALUES ($1, $2, $3, $4, true, NOW())
        RETURNING *
      SQL
      email, name, hashed, role,
      as: User
    )
  rescue DB::Error
    nil
  end
end

# src/models/task.cr
struct Task
  include DB::Serializable
  
  property id : Int64
  property title : String
  property description : String?
  property status : String  # todo, in_progress, done
  property priority : Int32
  property user_id : Int64
  property due_date : Time?
  property created_at : Time
  property updated_at : Time
  
  def done? : Bool
    status == "done"
  end
  
  def self.for_user(db, user_id : Int64, status : String? = nil) : Array(Task)
    if status
      db.query_all(
        "SELECT * FROM tasks WHERE user_id = $1 AND status = $2 ORDER BY priority DESC, created_at DESC",
        user_id, status,
        as: Task
      )
    else
      db.query_all(
        "SELECT * FROM tasks WHERE user_id = $1 ORDER BY priority DESC, created_at DESC",
        user_id,
        as: Task
      )
    end
  end
  
  def self.create(db, user_id : Int64, title : String, description : String?, priority : Int32 = 1, due_date : Time? = nil) : Task?
    db.query_one?(
      <<-SQL,
        INSERT INTO tasks (title, description, status, priority, user_id, due_date, created_at, updated_at)
        VALUES ($1, $2, 'todo', $3, $4, $5, NOW(), NOW())
        RETURNING *
      SQL
      title, description, priority, user_id, due_date,
      as: Task
    )
  end
  
  def self.update_status(db, id : Int64, user_id : Int64, status : String) : Task?
    db.query_one?(
      "UPDATE tasks SET status = $1, updated_at = NOW() WHERE id = $2 AND user_id = $3 RETURNING *",
      status, id, user_id,
      as: Task
    )
  end
  
  def self.delete(db, id : Int64, user_id : Int64) : Bool
    result = db.exec("DELETE FROM tasks WHERE id = $1 AND user_id = $2", id, user_id)
    result.rows_affected > 0
  end
end
```

## Database Setup

```crystal
# src/database.cr
require "db"
require "pg"

DATABASE_URL = ENV["DATABASE_URL"]? || "postgres://user:password@localhost/taskapp"

def db
  DB.open(DATABASE_URL)
end

def create_tables
  DB.open(DATABASE_URL) do |db|
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS users (
        id BIGSERIAL PRIMARY KEY,
        email VARCHAR(255) UNIQUE NOT NULL,
        name VARCHAR(255) NOT NULL,
        hashed_password VARCHAR(255) NOT NULL,
        role VARCHAR(50) DEFAULT 'user',
        active BOOLEAN DEFAULT true,
        created_at TIMESTAMP DEFAULT NOW()
      )
    SQL
    
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS sessions (
        id VARCHAR(64) PRIMARY KEY,
        user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
        data TEXT DEFAULT '{}',
        expires_at TIMESTAMP NOT NULL,
        created_at TIMESTAMP DEFAULT NOW()
      )
    SQL
    
    db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS tasks (
        id BIGSERIAL PRIMARY KEY,
        title VARCHAR(500) NOT NULL,
        description TEXT,
        status VARCHAR(50) DEFAULT 'todo',
        priority INT DEFAULT 1,
        user_id BIGINT REFERENCES users(id) ON DELETE CASCADE,
        due_date TIMESTAMP,
        created_at TIMESTAMP DEFAULT NOW(),
        updated_at TIMESTAMP DEFAULT NOW()
      )
    SQL
    
    # Create indexes
    db.exec "CREATE INDEX IF NOT EXISTS idx_tasks_user_id ON tasks(user_id)"
    db.exec "CREATE INDEX IF NOT EXISTS idx_sessions_expires_at ON sessions(expires_at)"
    
    puts "Tables created successfully"
  end
end
```

## Main Application

```crystal
# src/app.cr
require "kemal"
require "json"
require "./models/user"
require "./models/task"
require "./database"

# Initialize database
create_tables

# ===== MIDDLEWARE =====

# Security headers
before_all do |env|
  env.response.headers["X-Content-Type-Options"] = "nosniff"
  env.response.headers["X-Frame-Options"] = "SAMEORIGIN"
  env.response.headers["X-XSS-Protection"] = "1; mode=block"
end

# Auth helper
def current_user(env) : User?
  session_id = env.request.cookies["session_id"]?.try(&.value)
  return nil unless session_id
  
  DB.open(DATABASE_URL) do |db|
    session = db.query_one?(
      "SELECT user_id FROM sessions WHERE id = $1 AND expires_at > NOW()",
      session_id,
      as: {user_id: Int64}
    )
    
    if session
      User.find_by_id(db, session[:user_id])
    end
  end
end

def require_auth(env)
  user = current_user(env)
  halt env, status_code: 401, response: "Unauthorized" unless user
  user
end

# ===== AUTH ROUTES =====

get "/login" do |env|
  render "views/auth/login.ecr", layout: "views/layouts/main.ecr"
end

post "/login" do |env|
  email = env.params.body["email"]? || ""
  password = env.params.body["password"]? || ""
  
  user = DB.open(DATABASE_URL) do |db|
    User.find_by_email(db, email.downcase)
  end
  
  if user && user.verify_password(password)
    # สร้าง session
    session_id = Random::Secure.hex(32)
    
    DB.open(DATABASE_URL) do |db|
      db.exec(
        "INSERT INTO sessions (id, user_id, expires_at) VALUES ($1, $2, NOW() + INTERVAL '7 days')",
        session_id, user.id
      )
    end
    
    env.response.cookies << HTTP::Cookie.new(
      name: "session_id",
      value: session_id,
      path: "/",
      http_only: true,
      expires: Time.local + 7.days
    )
    
    env.redirect "/"
  else
    error_msg = "อีเมลหรือรหัสผ่านไม่ถูกต้อง"
    render "views/auth/login.ecr", layout: "views/layouts/main.ecr"
  end
end

get "/register" do |env|
  render "views/auth/register.ecr", layout: "views/layouts/main.ecr"
end

post "/register" do |env|
  email = env.params.body["email"]? || ""
  name = env.params.body["name"]? || ""
  password = env.params.body["password"]? || ""
  
  errors = [] of String
  errors << "กรุณาใส่อีเมล" if email.empty?
  errors << "กรุณาใส่ชื่อ" if name.empty?
  errors << "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร" if password.size < 8
  
  unless errors.empty?
    render "views/auth/register.ecr", layout: "views/layouts/main.ecr"
    next
  end
  
  user = DB.open(DATABASE_URL) do |db|
    User.create(db, email.downcase, name, password)
  end
  
  if user
    env.redirect "/login"
  else
    errors << "อีเมลนี้มีในระบบแล้ว"
    render "views/auth/register.ecr", layout: "views/layouts/main.ecr"
  end
end

post "/logout" do |env|
  if session_id = env.request.cookies["session_id"]?.try(&.value)
    DB.open(DATABASE_URL) do |db|
      db.exec("DELETE FROM sessions WHERE id = $1", session_id)
    end
    
    env.response.cookies << HTTP::Cookie.new(
      name: "session_id",
      value: "",
      expires: Time.epoch(0)
    )
  end
  
  env.redirect "/login"
end

# ===== TASK ROUTES =====

get "/" do |env|
  user = current_user(env)
  unless user
    env.redirect "/login"
    next
  end
  
  status_filter = env.params.query["status"]?
  
  tasks = DB.open(DATABASE_URL) do |db|
    Task.for_user(db, user.id, status_filter)
  end
  
  todo_count = tasks.count { |t| t.status == "todo" }
  in_progress_count = tasks.count { |t| t.status == "in_progress" }
  done_count = tasks.count { |t| t.status == "done" }
  
  render "views/tasks/index.ecr", layout: "views/layouts/main.ecr"
end

get "/tasks/new" do |env|
  user = require_auth(env)
  render "views/tasks/new.ecr", layout: "views/layouts/main.ecr"
end

post "/tasks" do |env|
  user = require_auth(env)
  
  title = env.params.body["title"]? || ""
  description = env.params.body["description"]?
  priority = env.params.body["priority"]?.try(&.to_i?) || 1
  due_date_str = env.params.body["due_date"]?
  due_date = due_date_str.try { |d| Time.parse(d, "%Y-%m-%d", Time::Location.local) rescue nil }
  
  unless title.size >= 3
    errors = ["ชื่อ task ต้องมีอย่างน้อย 3 ตัวอักษร"]
    render "views/tasks/new.ecr", layout: "views/layouts/main.ecr"
    next
  end
  
  task = DB.open(DATABASE_URL) do |db|
    Task.create(db, user.id, title, description, priority, due_date)
  end
  
  env.redirect "/" if task
end

patch "/tasks/:id/status" do |env|
  user = require_auth(env)
  id = env.params.url["id"].to_i64? || 0_i64
  status = env.params.body["status"]? || ""
  
  valid_statuses = ["todo", "in_progress", "done"]
  
  unless valid_statuses.includes?(status)
    halt env, status_code: 400, response: "Invalid status"
  end
  
  task = DB.open(DATABASE_URL) do |db|
    Task.update_status(db, id, user.id, status)
  end
  
  env.response.content_type = "application/json"
  task ? task.to_json : halt(env, status_code: 404, response: "Not found")
end

delete "/tasks/:id" do |env|
  user = require_auth(env)
  id = env.params.url["id"].to_i64? || 0_i64
  
  deleted = DB.open(DATABASE_URL) do |db|
    Task.delete(db, id, user.id)
  end
  
  env.response.status_code = deleted ? 204 : 404
  ""
end

# ===== API ROUTES =====

get "/api/tasks" do |env|
  user = require_auth(env)
  
  tasks = DB.open(DATABASE_URL) do |db|
    Task.for_user(db, user.id)
  end
  
  env.response.content_type = "application/json"
  tasks.to_json
end

# ===== ERROR HANDLERS =====

error 404 do |env|
  env.response.content_type = "text/html"
  "<h1>404 - ไม่พบหน้าที่ต้องการ</h1>"
end

error 500 do |env|
  env.response.content_type = "text/html"
  "<h1>500 - เกิดข้อผิดพลาดบน Server</h1>"
end

# ===== START SERVER =====
Kemal.config.port = ENV["PORT"]?.try(&.to_i?) || 3000
Kemal.run
```

## Layout Template

```html
<!-- views/layouts/main.ecr -->
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Task App</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: 'Sarabun', sans-serif; background: #f3f4f6; color: #1f2937; }
    .navbar { background: #2563eb; color: white; padding: 1rem 2rem; display: flex; justify-content: space-between; align-items: center; }
    .navbar a { color: white; text-decoration: none; }
    .container { max-width: 1000px; margin: 2rem auto; padding: 0 1rem; }
    .card { background: white; border-radius: 8px; padding: 1.5rem; box-shadow: 0 1px 3px rgba(0,0,0,0.1); margin-bottom: 1rem; }
    .btn { padding: 0.5rem 1rem; border: none; border-radius: 4px; cursor: pointer; text-decoration: none; display: inline-block; }
    .btn-primary { background: #2563eb; color: white; }
    .btn-danger { background: #ef4444; color: white; }
    .btn-success { background: #10b981; color: white; }
    input, textarea, select { width: 100%; padding: 0.5rem; border: 1px solid #d1d5db; border-radius: 4px; margin-top: 0.25rem; }
    label { font-weight: 600; display: block; margin-top: 1rem; }
    .error { color: #ef4444; font-size: 0.875rem; }
    .status-badge { padding: 0.25rem 0.75rem; border-radius: 9999px; font-size: 0.75rem; font-weight: 600; }
    .status-todo { background: #fef3c7; color: #92400e; }
    .status-in_progress { background: #dbeafe; color: #1e40af; }
    .status-done { background: #d1fae5; color: #065f46; }
  </style>
</head>
<body>
  <nav class="navbar">
    <a href="/" style="font-size: 1.25rem; font-weight: bold;">📋 Task App</a>
    <% if user = current_user(env) %>
      <div>
        <span>สวัสดี, <%= user.name %></span>
        <form method="POST" action="/logout" style="display: inline; margin-left: 1rem;">
          <button type="submit" style="background: none; border: 1px solid white; color: white; padding: 0.25rem 0.5rem; cursor: pointer; border-radius: 4px;">ออกจากระบบ</button>
        </form>
      </div>
    <% else %>
      <div>
        <a href="/login" style="margin-right: 1rem;">เข้าสู่ระบบ</a>
        <a href="/register">สมัครสมาชิก</a>
      </div>
    <% end %>
  </nav>
  
  <div class="container">
    <%= content %>
  </div>
</body>
</html>
```

## Task Index Template

```html
<!-- views/tasks/index.ecr -->
<div style="display: flex; justify-content: space-between; align-items: center; margin-bottom: 1.5rem;">
  <h1>Tasks ของฉัน</h1>
  <a href="/tasks/new" class="btn btn-primary">+ เพิ่ม Task</a>
</div>

<div style="display: flex; gap: 1rem; margin-bottom: 1.5rem;">
  <div class="card" style="flex: 1; text-align: center;">
    <div style="font-size: 2rem; font-weight: bold; color: #92400e;"><%= todo_count %></div>
    <div>รอดำเนินการ</div>
  </div>
  <div class="card" style="flex: 1; text-align: center;">
    <div style="font-size: 2rem; font-weight: bold; color: #1e40af;"><%= in_progress_count %></div>
    <div>กำลังดำเนินการ</div>
  </div>
  <div class="card" style="flex: 1; text-align: center;">
    <div style="font-size: 2rem; font-weight: bold; color: #065f46;"><%= done_count %></div>
    <div>เสร็จแล้ว</div>
  </div>
</div>

<div style="margin-bottom: 1rem;">
  <a href="/" class="btn <%= status_filter.nil? ? "btn-primary" : "" %>" style="margin-right: 0.5rem; background: <%= status_filter.nil? ? "#2563eb" : "#e5e7eb" %>; color: <%= status_filter.nil? ? "white" : "#374151" %>">ทั้งหมด</a>
  <a href="/?status=todo" class="btn" style="margin-right: 0.5rem; background: <%= status_filter == "todo" ? "#92400e" : "#fef3c7" %>; color: <%= status_filter == "todo" ? "white" : "#92400e" %>">รอดำเนินการ</a>
  <a href="/?status=in_progress" class="btn" style="margin-right: 0.5rem; background: <%= status_filter == "in_progress" ? "#1e40af" : "#dbeafe" %>; color: <%= status_filter == "in_progress" ? "white" : "#1e40af" %>">กำลังทำ</a>
  <a href="/?status=done" class="btn" style="background: <%= status_filter == "done" ? "#065f46" : "#d1fae5" %>; color: <%= status_filter == "done" ? "white" : "#065f46" %>">เสร็จแล้ว</a>
</div>

<% if tasks.empty? %>
  <div class="card" style="text-align: center; padding: 3rem;">
    <p style="color: #6b7280;">ยังไม่มี tasks</p>
    <a href="/tasks/new" class="btn btn-primary" style="margin-top: 1rem;">สร้าง Task แรก</a>
  </div>
<% else %>
  <% tasks.each do |task| %>
    <div class="card" style="display: flex; justify-content: space-between; align-items: flex-start;">
      <div style="flex: 1;">
        <div style="display: flex; align-items: center; gap: 0.5rem; margin-bottom: 0.5rem;">
          <span class="status-badge status-<%= task.status %>"><%= task.status.gsub("_", " ") %></span>
          <strong><%= task.title %></strong>
        </div>
        <% if task.description %>
          <p style="color: #6b7280; font-size: 0.875rem;"><%= task.description %></p>
        <% end %>
        <% if task.due_date %>
          <p style="color: #9ca3af; font-size: 0.75rem; margin-top: 0.5rem;">ครบกำหนด: <%= task.due_date.not_nil!.to_s("%d/%m/%Y") %></p>
        <% end %>
      </div>
      <div style="display: flex; gap: 0.5rem; margin-left: 1rem;">
        <% if task.status != "done" %>
          <button onclick="updateStatus(<%= task.id %>, '<%= task.status == "todo" ? "in_progress" : "done" %>')" 
                  class="btn btn-success" style="font-size: 0.75rem;">
            <%= task.status == "todo" ? "เริ่มทำ" : "เสร็จแล้ว" %>
          </button>
        <% end %>
        <button onclick="deleteTask(<%= task.id %>)" class="btn btn-danger" style="font-size: 0.75rem;">ลบ</button>
      </div>
    </div>
  <% end %>
<% end %>

<script>
async function updateStatus(id, status) {
  const res = await fetch(`/tasks/${id}/status`, {
    method: 'PATCH',
    headers: {'Content-Type': 'application/x-www-form-urlencoded'},
    body: `status=${status}`
  });
  if (res.ok) location.reload();
}

async function deleteTask(id) {
  if (!confirm('ต้องการลบ task นี้หรือไม่?')) return;
  const res = await fetch(`/tasks/${id}`, {method: 'DELETE'});
  if (res.ok) location.reload();
}
</script>
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Add Tags to Tasks

```crystal
# เพิ่ม tag functionality
# ALTER TABLE tasks ADD COLUMN tags VARCHAR[] DEFAULT '{}';

# ใน Task model
def self.by_tag(db, user_id : Int64, tag : String) : Array(Task)
  db.query_all(
    "SELECT * FROM tasks WHERE user_id = $1 AND $2 = ANY(tags) ORDER BY created_at DESC",
    user_id, tag,
    as: Task
  )
end
```

## สรุป

ในบทนี้เราได้สร้าง full-stack web app ที่ประกอบด้วย:

1. **Database**: PostgreSQL + migrations
2. **Models**: User, Task ด้วย DB::Serializable
3. **Authentication**: session-based login/register/logout
4. **Authorization**: ตรวจสอบว่า task เป็นของ user
5. **CRUD**: create, read, update status, delete tasks
6. **ECR Templates**: layout + pages
7. **CSS**: inline styles สำหรับ UI
8. **JavaScript**: fetch API สำหรับ async operations
9. **API Routes**: JSON endpoints
10. **Error Handling**: 404, 500

นี่คือ pattern พื้นฐานที่ใช้ใน real-world web applications
