# Part 134: Kemal Templates (ECR) - การใช้งาน ECR Templates ใน Kemal

## บทนำ

ECR (Embedded Crystal) เป็น template engine ที่รวม Crystal code เข้ากับ HTML คล้ายกับ ERB ใน Ruby ทำให้สามารถสร้าง dynamic HTML pages ได้สะดวก

## ECR พื้นฐาน

### ไฟล์ Template

สร้าง `src/views/hello.ecr`:
```html
<h1>สวัสดี, <%= name %>!</h1>
<p>เวลาปัจจุบัน: <%= Time.local %></p>
```

### การ render template

```crystal
require "kemal"
require "ecr"

# ตัวแปรที่จะส่งไปยัง template ต้องอยู่ใน scope ที่ template เรียก render
get "/hello/:name" do |env|
  name = env.params.url["name"]
  render "src/views/hello.ecr"
end

Kemal.run
```

## ECR Syntax

```html
<!-- src/views/example.ecr -->

<!-- 1. Output value: <%= ... %> -->
<p>ชื่อ: <%= user_name %></p>

<!-- 2. Execute code (no output): <% ... %> -->
<% items.each do |item| %>
  <li><%= item %></li>
<% end %>

<!-- 3. Comment: <%# ... %> -->
<%# This is a comment %>

<!-- 4. Escape HTML: <%== ... %> (unescaped) -->
<p>Raw HTML: <%== raw_html %></p>
```

## Layout System

### Layout file: `src/views/layouts/main.ecr`

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title><%= page_title %></title>
  <link rel="stylesheet" href="/css/style.css">
</head>
<body>
  <nav>
    <a href="/">หน้าแรก</a>
    <a href="/about">เกี่ยวกับ</a>
    <a href="/contact">ติดต่อ</a>
  </nav>
  
  <main>
    <%= content %>
  </main>
  
  <footer>
    <p>&copy; 2024 Crystal App</p>
  </footer>
  
  <script src="/js/app.js"></script>
</body>
</html>
```

### Content template: `src/views/index.ecr`

```html
<div class="hero">
  <h1><%= title %></h1>
  <p><%= description %></p>
</div>

<div class="features">
  <% features.each do |feature| %>
    <div class="feature-card">
      <h3><%= feature["name"] %></h3>
      <p><%= feature["description"] %></p>
    </div>
  <% end %>
</div>
```

### ใช้งาน layout ใน Kemal

```crystal
require "kemal"

get "/" do |env|
  page_title = "หน้าแรก"
  title = "ยินดีต้อนรับสู่ Crystal App"
  description = "สร้างด้วย Crystal + Kemal"
  features = [
    {"name" => "เร็ว", "description" => "เร็วเหมือน C"},
    {"name" => "ปลอดภัย", "description" => "Type safety"},
    {"name" => "สวยงาม", "description" => "Syntax สวยงาม"},
  ]
  
  # Render content
  content = render "src/views/index.ecr"
  
  # Render layout พร้อม content
  render "src/views/layouts/main.ecr"
end

Kemal.run
```

## Partial Templates

### Partial: `src/views/partials/_nav.ecr`

```html
<nav class="navbar">
  <div class="navbar-brand">
    <a href="/"><%= app_name %></a>
  </div>
  <ul class="navbar-menu">
    <% nav_items.each do |item| %>
      <li>
        <a href="<%= item["url"] %>" 
           class="<%= current_path == item["url"] ? "active" : "" %>">
          <%= item["name"] %>
        </a>
      </li>
    <% end %>
  </ul>
</nav>
```

### Partial: `src/views/partials/_user_card.ecr`

```html
<div class="user-card">
  <img src="<%= user_avatar %>" alt="<%= user_name %>">
  <h3><%= user_name %></h3>
  <p><%= user_email %></p>
  <% if is_admin %>
    <span class="badge admin">Admin</span>
  <% end %>
</div>
```

### ใช้งาน partials

```crystal
require "kemal"

# Helper method สำหรับ render partial
def partial(template : String, **vars) : String
  # สร้าง binding context ด้วยตัวแปรที่ส่งมา
  render template
end

get "/dashboard" do |env|
  app_name = "Crystal Dashboard"
  current_path = "/dashboard"
  nav_items = [
    {"name" => "หน้าแรก", "url" => "/"},
    {"name" => "Dashboard", "url" => "/dashboard"},
    {"name" => "Users", "url" => "/users"},
  ]
  
  # Users
  users = [
    {"name" => "สมชาย", "email" => "somchai@example.com", "is_admin" => true},
    {"name" => "สมหญิง", "email" => "somying@example.com", "is_admin" => false},
  ]
  
  render "src/views/dashboard.ecr", layout: "src/views/layouts/main.ecr"
end

Kemal.run
```

## Passing Data ให้ Templates

```crystal
require "kemal"

# วิธีที่ 1: Local variables
get "/users" do |env|
  users = [
    {id: 1, name: "สมชาย", email: "somchai@example.com", role: "admin"},
    {id: 2, name: "สมหญิง", email: "somying@example.com", role: "user"},
  ]
  
  total = users.size
  page = env.params.query["page"]?.try(&.to_i?) || 1
  
  render "src/views/users/index.ecr"
end

# วิธีที่ 2: Instance variables (ไม่แนะนำ แต่ใช้ได้)
class AppController
  property users : Array(NamedTuple(id: Int32, name: String))
  property current_user : String
  
  def initialize
    @users = [] of NamedTuple(id: Int32, name: String)
    @current_user = ""
  end
  
  def index
    @users = [
      {id: 1, name: "สมชาย"},
      {id: 2, name: "สมหญิง"},
    ]
    @current_user = "Admin"
    
    ECR.render("src/views/users/index.ecr")
  end
end
```

## Template Helpers

```crystal
require "kemal"
require "html"

# Helper methods ที่ใช้ใน templates
module ViewHelpers
  def self.h(text : String) : String
    HTML.escape(text)
  end
  
  def self.truncate(text : String, length : Int32 = 100) : String
    text.size > length ? "#{text[0, length]}..." : text
  end
  
  def self.format_date(time : Time, format : String = "%d/%m/%Y") : String
    time.to_s(format)
  end
  
  def self.format_number(n : Number) : String
    # เพิ่ม comma separator
    n.to_s.reverse.chars.each_slice(3).map(&.join).join(",").reverse
  end
  
  def self.pluralize(count : Int32, singular : String, plural : String? = nil) : String
    if count == 1
      "#{count} #{singular}"
    else
      "#{count} #{plural || "#{singular}s"}"
    end
  end
  
  def self.active_class(current : String, target : String) : String
    current == target ? "active" : ""
  end
  
  def self.ago(time : Time) : String
    diff = Time.local - time
    
    case diff
    when .< 1.minute  then "เมื่อกี้"
    when .< 1.hour    then "#{diff.total_minutes.to_i} นาทีที่แล้ว"
    when .< 1.day     then "#{diff.total_hours.to_i} ชั่วโมงที่แล้ว"
    when .< 1.week    then "#{diff.total_days.to_i} วันที่แล้ว"
    else format_date(time)
    end
  end
end
```

## Complete Blog Template Example

### Layout: `src/views/layouts/blog.ecr`

```html
<!DOCTYPE html>
<html lang="th">
<head>
  <meta charset="UTF-8">
  <title><%= page_title %> - Crystal Blog</title>
  <style>
    * { box-sizing: border-box; margin: 0; padding: 0; }
    body { font-family: "Sarabun", sans-serif; background: #f5f5f5; }
    header { background: #2563eb; color: white; padding: 1rem 2rem; }
    header h1 { font-size: 1.5rem; }
    nav a { color: white; text-decoration: none; margin-left: 1rem; }
    main { max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
    .post-card { background: white; border-radius: 8px; padding: 1.5rem; margin-bottom: 1rem; box-shadow: 0 1px 3px rgba(0,0,0,0.1); }
    .post-card h2 { color: #1e40af; }
    .meta { color: #6b7280; font-size: 0.875rem; margin: 0.5rem 0; }
    .tag { background: #dbeafe; color: #1e40af; padding: 0.25rem 0.75rem; border-radius: 9999px; font-size: 0.75rem; margin-right: 0.5rem; }
    footer { text-align: center; padding: 2rem; color: #6b7280; }
  </style>
</head>
<body>
  <header>
    <h1>Crystal Blog</h1>
    <nav>
      <a href="/">หน้าแรก</a>
      <a href="/posts">บทความ</a>
      <a href="/about">เกี่ยวกับ</a>
    </nav>
  </header>
  
  <main>
    <%= content %>
  </main>
  
  <footer>
    <p>สร้างด้วย Crystal + Kemal + ECR</p>
  </footer>
</body>
</html>
```

### Post List: `src/views/posts/index.ecr`

```html
<h2>บทความทั้งหมด (<%= total_posts %> บทความ)</h2>

<% if posts.empty? %>
  <p>ยังไม่มีบทความ</p>
<% else %>
  <% posts.each do |post| %>
    <article class="post-card">
      <h2><a href="/posts/<%= post["id"] %>"><%= ViewHelpers.h(post["title"].to_s) %></a></h2>
      <div class="meta">
        <span>โดย <%= post["author"] %></span>
        <span> · </span>
        <span><%= ViewHelpers.format_date(Time.parse_rfc3339(post["created_at"].to_s)) %></span>
      </div>
      <p><%= ViewHelpers.truncate(post["content"].to_s, 150) %></p>
      <div class="tags">
        <% post["tags"].as_a.each do |tag| %>
          <span class="tag"><%= tag %></span>
        <% end %>
      </div>
    </article>
  <% end %>
<% end %>

<div class="pagination">
  <% if page > 1 %>
    <a href="/posts?page=<%= page - 1 %>">&laquo; หน้าก่อน</a>
  <% end %>
  <span>หน้า <%= page %> จาก <%= total_pages %></span>
  <% if page < total_pages %>
    <a href="/posts?page=<%= page + 1 %>">หน้าถัดไป &raquo;</a>
  <% end %>
</div>
```

```crystal
require "kemal"

struct BlogPost
  property id : Int32
  property title : String
  property content : String
  property author : String
  property tags : Array(String)
  property created_at : Time
  
  def initialize(@id, @title, @content, @author, @tags = [] of String)
    @created_at = Time.local
  end
end

posts = [
  BlogPost.new(1, "Crystal Programming Language", "Crystal เป็นภาษาที่ compile แล้วเร็วมาก...", "สมชาย", ["crystal", "programming"]),
  BlogPost.new(2, "Kemal Web Framework", "Kemal ทำให้การสร้าง web app ง่ายขึ้นมาก...", "สมหญิง", ["crystal", "web", "kemal"]),
  BlogPost.new(3, "ECR Templates", "ECR เป็น template engine ที่ใช้ง่าย...", "สมชาย", ["crystal", "ecr", "templates"]),
]

get "/posts" do |env|
  page = env.params.query["page"]?.try(&.to_i?) || 1
  per_page = 10
  
  total_posts = posts.size
  total_pages = (total_posts.to_f / per_page).ceil.to_i
  start_idx = (page - 1) * per_page
  paged_posts = posts[start_idx, per_page]? || [] of BlogPost
  page_title = "บทความ"
  
  # แปลงเป็น format สำหรับ template
  posts_data = paged_posts.map do |p|
    {
      "id" => p.id,
      "title" => p.title,
      "content" => p.content,
      "author" => p.author,
      "tags" => p.tags,
      "created_at" => p.created_at.to_rfc3339,
    }
  end
  
  content = render "src/views/posts/index.ecr"
  render "src/views/layouts/blog.ecr"
end

Kemal.run
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Admin Dashboard Template

สร้าง `src/views/admin/dashboard.ecr`:

```html
<div class="dashboard">
  <h1>Admin Dashboard</h1>
  
  <div class="stats-grid">
    <div class="stat-card">
      <h3>ผู้ใช้ทั้งหมด</h3>
      <div class="stat-number"><%= ViewHelpers.format_number(total_users) %></div>
      <div class="stat-change <%= user_growth > 0 ? "positive" : "negative" %>">
        <%= user_growth > 0 ? "▲" : "▼" %> <%= user_growth.abs %>% จากเดือนที่แล้ว
      </div>
    </div>
    
    <div class="stat-card">
      <h3>รายได้วันนี้</h3>
      <div class="stat-number">฿<%= ViewHelpers.format_number(today_revenue) %></div>
    </div>
    
    <div class="stat-card">
      <h3>คำสั่งซื้อใหม่</h3>
      <div class="stat-number"><%= new_orders %></div>
    </div>
  </div>
  
  <div class="recent-activity">
    <h2>กิจกรรมล่าสุด</h2>
    <table>
      <thead>
        <tr>
          <th>เวลา</th>
          <th>ผู้ใช้</th>
          <th>Action</th>
          <th>รายละเอียด</th>
        </tr>
      </thead>
      <tbody>
        <% recent_activities.each do |activity| %>
          <tr>
            <td><%= ViewHelpers.ago(activity.time) %></td>
            <td><%= ViewHelpers.h(activity.user) %></td>
            <td><span class="badge <%= activity.type %>"><%= activity.type %></span></td>
            <td><%= ViewHelpers.truncate(activity.detail, 50) %></td>
          </tr>
        <% end %>
      </tbody>
    </table>
  </div>
</div>
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **ECR Syntax**: `<%= %>` output, `<% %>` execute, `<%# %>` comment
2. **render**: render template file
3. **Layout System**: แยก layout และ content
4. **Partials**: template ย่อยสำหรับ reuse
5. **Passing Data**: ส่งตัวแปรไปยัง templates
6. **View Helpers**: helper methods สำหรับ templates
7. **HTML Escaping**: ป้องกัน XSS ด้วย `ViewHelpers.h()`
8. **Complete Blog**: ตัวอย่าง blog templates

ECR เหมาะสำหรับ server-side rendering ที่ต้องการ Crystal's type safety
