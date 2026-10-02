# Part 138: Lucky Framework Introduction - เริ่มต้นกับ Lucky

## บทนำ

Lucky เป็น full-stack web framework สำหรับ Crystal ที่เน้น type safety, convention over configuration และ developer productivity Lucky แตกต่างจาก Kemal ตรงที่มีโครงสร้างชัดเจนกว่าและมี ORM (Avram) รวมมาด้วย

## Lucky vs Kemal

| Feature | Lucky | Kemal |
|---------|-------|-------|
| Type | Full-stack | Micro framework |
| ORM | Avram (built-in) | ไม่มี |
| Type Safety | สูงมาก | ปานกลาง |
| Learning Curve | สูง | ต่ำ |
| Convention | มาก | น้อย |
| Use Case | Full app | API/Micro |

## การติดตั้ง

```bash
# ติดตั้ง Lucky CLI
brew install luckyframework/homebrew-lucky/lucky  # macOS
# หรือ
curl -sSL https://lucky-cli.fly.dev/install.sh | bash  # Linux

# สร้าง project ใหม่
lucky init my_app
cd my_app
shards install

# รัน development server
lucky dev
```

## โครงสร้าง Lucky Project

```
my_app/
├── src/
│   ├── actions/         <- HTTP Action classes
│   │   ├── browser_action.cr
│   │   ├── api_action.cr
│   │   └── home/
│   │       └── index.cr
│   ├── models/          <- Avram models
│   ├── operations/      <- SaveOperations (form handling)
│   ├── pages/           <- Page components (HTML)
│   │   ├── main_layout.cr
│   │   └── home/
│   │       └── index_page.cr
│   ├── queries/         <- Database queries
│   ├── components/      <- Reusable components
│   ├── app.cr
│   └── server.cr
├── db/
│   └── migrations/
├── config/
├── spec/
└── shard.yml
```

## Actions (Routes)

```crystal
# src/actions/home/index.cr
class Home::Index < BrowserAction
  get "/" do
    render IndexPage
  end
end

# API Action
class Api::Users::Index < ApiAction
  get "/api/users" do
    users = UserQuery.new.order_by_created_at(:desc).select
    json UserSerializer.for_collection(users)
  end
end

# Action with path parameters
class Api::Users::Show < ApiAction
  get "/api/users/:user_id" do
    user = UserQuery.find(user_id)
    json UserSerializer.new(user)
  end
end
```

## BrowserAction

```crystal
# src/actions/browser_action.cr
abstract class BrowserAction < Lucky::Action
  include Lucky::SecureHeaders::DisableFLoC
  include Lucky::ProtectFromForgery
  include CleverQueryParams
  
  accepted_formats [:html]
  
  # Shared helpers
  include AuthHelpers
  
  def current_user : User
    # จาก session
    @current_user ||= UserQuery.find(session.get(:user_id).to_i64)
  rescue
    raise NotSignedIn.new
  end
end
```

## Pages (HTML Components)

```crystal
# src/pages/home/index_page.cr
class Home::IndexPage < MainLayout
  def content
    h1 "ยินดีต้อนรับสู่ Lucky!"
    
    div class: "features" do
      render_features
    end
    
    link "เข้าสู่ระบบ", to: SignIns::New, class: "btn btn-primary"
  end
  
  private def render_features
    [
      {name: "Type-safe", desc: "Type safety ทุกที่"},
      {name: "Fast", desc: "เร็วเหมือน C"},
      {name: "ORM", desc: "Avram ORM"},
    ].each do |feature|
      div class: "feature-card" do
        h3 feature[:name]
        p feature[:desc]
      end
    end
  end
end
```

## MainLayout

```crystal
# src/pages/main_layout.cr
abstract class MainLayout < Lucky::HTMLPage
  abstract def content
  
  def page_title : String
    "My Lucky App"
  end
  
  def render
    html_doctype
    
    html lang: "th" do
      head do
        utf8_charset
        title page_title
        css_link asset("css/app.css")
        js_link asset("js/app.js"), defer: "true"
      end
      
      body do
        render_nav
        
        main class: "container" do
          content
        end
        
        render_footer
      end
    end
  end
  
  private def render_nav
    nav class: "navbar" do
      a "Lucky App", href: "/", class: "navbar-brand"
      
      div class: "navbar-links" do
        link "หน้าแรก", to: Home::Index
        link "เกี่ยวกับ", to: About::Index
      end
    end
  end
  
  private def render_footer
    footer do
      p "© 2024 Lucky App"
    end
  end
end
```

## Models (Avram)

```crystal
# src/models/user.cr
class User < BaseModel
  table do
    column email : String
    column name : String
    column hashed_password : String
    column role : String, default: "user"
    column active : Bool, default: true
    column verified_at : Time?
    
    has_many posts : Post
    has_many comments : Comment
  end
  
  # Validations ใน Operations
end

# Migration: db/migrations/00000000000001_create_users.cr
class CreateUsers::V00000000000001 < Avram::Migrator::Migration::V1
  def migrate
    create table_for(User) do
      primary_key id : Int64
      add_timestamps
      add email : String, unique: true
      add name : String
      add hashed_password : String
      add role : String, default: "user"
      add active : Bool, default: true
      add verified_at : Time?
    end
  end
  
  def rollback
    drop table_for(User)
  end
end
```

## Queries

```crystal
# src/queries/user_query.cr
class UserQuery < User::BaseQuery
  def active
    active(true)
  end
  
  def admin
    role("admin")
  end
  
  def by_email(email : String)
    email(email)
  end
  
  def search(q : String)
    name.ilike("%#{q}%").or(&.email.ilike("%#{q}%"))
  end
  
  def recent(limit : Int32 = 10)
    order_by_created_at(:desc).limit(limit)
  end
end

# ใช้งาน
users = UserQuery.new.active.recent(5).select
admin = UserQuery.new.admin.first?
user = UserQuery.new.by_email("user@example.com").first?
```

## Operations (Form Handling)

```crystal
# src/operations/save_user.cr
class SaveUser < User::SaveOperation
  attribute password : String
  attribute password_confirmation : String
  
  permit_columns email, name
  
  # Validations
  before_save do
    validate_required email, name, password
    validate_uniqueness_of email
    validate_format_of email, with: /.+@.+\..+/, message: "รูปแบบอีเมลไม่ถูกต้อง"
    validate_minimum_length password, 8
    validate_confirmation_of password, with: password_confirmation
    hash_password
  end
  
  private def hash_password
    if password.valid? && (pwd = password.value)
      hashed_password.value = Crypto::Bcrypt::Password.create(pwd).to_s
    end
  end
end

# ใช้งานใน Action
class Users::Create < BrowserAction
  post "/users" do
    SaveUser.create(params) do |operation, user|
      if user
        redirect to: Home::Index, notice: "สร้างบัญชีสำเร็จ!"
      else
        render Users::NewPage, operation: operation
      end
    end
  end
end
```

## JSON API

```crystal
# src/actions/api_action.cr
abstract class ApiAction < Lucky::Action
  accepted_formats [:json]
  
  include Lucky::RequestExpectsJson
  
  # Error handling
  rescue_from Avram::RecordNotFoundError do
    json({error: "ไม่พบข้อมูล"}, status: 404)
  end
end

# src/actions/api/v1/users/index.cr
class Api::V1::Users::Index < ApiAction
  get "/api/v1/users" do
    page = params.get?(:page).try(&.to_i) || 1
    per_page = 20
    
    users = UserQuery.new
      .active
      .order_by_created_at(:desc)
      .limit(per_page)
      .offset((page - 1) * per_page)
      .select
    
    total = UserQuery.new.active.select_count
    
    json({
      data: users.map { |u| serialize_user(u) },
      meta: {
        page: page,
        per_page: per_page,
        total: total,
        pages: (total.to_f / per_page).ceil.to_i,
      }
    })
  end
  
  private def serialize_user(user : User)
    {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      created_at: user.created_at,
    }
  end
end
```

## Pipes (Middleware)

```crystal
# src/actions/mixins/require_sign_in.cr
module RequireSignIn
  macro included
    before sign_in_required
  end
  
  private def sign_in_required
    if current_user?
      continue
    else
      if request.headers["Accept"]?.try(&.includes?("application/json"))
        json({error: "Authentication required"}, status: 401)
      else
        redirect to: SignIns::New
      end
    end
  end
  
  private def current_user? : User?
    @current_user ||= begin
      if user_id = session.get?(:user_id)
        UserQuery.find?(user_id.to_i64)
      end
    end
  end
end

# ใช้งาน
class Dashboard::Index < BrowserAction
  include RequireSignIn
  
  get "/dashboard" do
    render Dashboard::IndexPage, current_user: current_user
  end
end
```

## Components

```crystal
# src/components/user_avatar_component.cr
class UserAvatarComponent < Lucky::Component
  needs user : User
  needs size : Int32 = 40
  
  def render
    div class: "user-avatar" do
      img(
        src: gravatar_url,
        alt: user.name,
        width: size,
        height: size,
        class: "avatar-img"
      )
    end
  end
  
  private def gravatar_url : String
    hash = Digest::MD5.hexdigest(user.email.downcase.strip)
    "https://www.gravatar.com/avatar/#{hash}?s=#{size}&d=identicon"
  end
end

# ใช้งานใน Page
mount UserAvatarComponent, user: current_user
mount UserAvatarComponent, user: current_user, size: 80
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Blog API ด้วย Lucky

```crystal
# src/actions/api/v1/posts/index.cr
class Api::V1::Posts::Index < ApiAction
  get "/api/v1/posts" do
    tag = params.get?(:tag)
    author_id = params.get?(:author_id).try(&.to_i64?)
    
    query = PostQuery.new.published.recent
    query = query.by_tag(tag) if tag
    query = query.by_author(author_id) if author_id
    
    posts = query.preload_author.select
    
    json({
      posts: posts.map { |p|
        {
          id: p.id,
          title: p.title,
          slug: p.slug,
          excerpt: p.excerpt,
          author: {id: p.author.id, name: p.author.name},
          published_at: p.published_at,
          tags: p.tags,
        }
      }
    })
  end
end

# src/queries/post_query.cr
class PostQuery < Post::BaseQuery
  def published
    published_at.lt(Time.local).published(true)
  end
  
  def recent
    order_by_published_at(:desc)
  end
  
  def by_tag(tag : String)
    tags.includes(tag)
  end
  
  def by_author(author_id : Int64)
    author_id(author_id)
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Lucky vs Kemal**: เปรียบเทียบ framework
2. **Project Structure**: โครงสร้าง Lucky project
3. **Actions**: HTTP request handlers
4. **Pages**: type-safe HTML generation
5. **Layouts**: MainLayout และ page structure
6. **Models**: Avram model definition
7. **Queries**: type-safe database queries
8. **Operations**: form handling และ validation
9. **JSON API**: API actions
10. **Pipes**: before/after hooks
11. **Components**: reusable UI components

Lucky เหมาะกับ full-stack web apps ที่ต้องการ type safety และ structure ชัดเจน
