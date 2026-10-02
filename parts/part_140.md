# Part 140: Lucky Avram ORM - การใช้งาน Avram ORM

## บทนำ

Avram เป็น ORM (Object-Relational Mapper) สำหรับ Lucky framework รองรับ PostgreSQL มี type-safe queries และ migrations

## การตั้งค่า

```yaml
# shard.yml
dependencies:
  lucky:
    github: luckyframework/lucky
  avram:
    github: luckyframework/avram
  carbon:
    github: luckyframework/carbon
```

```crystal
# config/database.cr
Avram::Database.configure do |settings|
  settings.url = ENV["DATABASE_URL"]? || "postgres://user:pass@localhost/myapp_dev"
end
```

## Models

```crystal
# src/models/base_model.cr
abstract class BaseModel < Avram::Model
  def self.database : Class
    AppDatabase
  end
end

# src/models/user.cr
class User < BaseModel
  table do
    # Primary key (auto Int64)
    primary_key id : Int64
    timestamps  # created_at, updated_at
    
    # Columns
    column email : String
    column name : String
    column hashed_password : String
    column role : String
    column bio : String?       # Nullable
    column active : Bool
    column age : Int32?
    column score : Float64
    column metadata : JSON::Any?
    column tags : Array(String)
    column avatar_url : String?
    column verified_at : Time?
    
    # Associations
    has_many posts : Post
    has_many comments : Comment
    belongs_to team : Team?
  end
end

class Post < BaseModel
  table do
    primary_key id : Int64
    timestamps
    
    column title : String
    column slug : String
    column content : String
    column excerpt : String?
    column published : Bool
    column published_at : Time?
    column view_count : Int64
    column tags : Array(String)
    
    belongs_to author : User
    has_many comments : Comment
    has_many post_tags : PostTag
    has_many tags_through : Tag, through: :post_tags
  end
end
```

## Migrations

```crystal
# db/migrations/00000000000001_create_users.cr
class CreateUsers::V00000000000001 < Avram::Migrator::Migration::V1
  def migrate
    create table_for(User) do
      primary_key id : Int64
      add_timestamps
      
      add email : String, unique: true
      add name : String
      add hashed_password : String
      add role : String, default: "user"
      add bio : String?, comment: "Optional biography"
      add active : Bool, default: true
      add age : Int32?
      add score : Float64, default: 0.0
      add tags : Array(String), default: [] of String
      add avatar_url : String?
      add verified_at : Time?
      
      add_index :email, unique: true
      add_index :role
    end
  end
  
  def rollback
    drop table_for(User)
  end
end

# db/migrations/00000000000002_create_posts.cr
class CreatePosts::V00000000000002 < Avram::Migrator::Migration::V1
  def migrate
    create table_for(Post) do
      primary_key id : Int64
      add_timestamps
      
      add title : String
      add slug : String, unique: true
      add content : String
      add excerpt : String?
      add published : Bool, default: false
      add published_at : Time?
      add view_count : Int64, default: 0
      add tags : Array(String), default: [] of String
      
      # Foreign key
      add author_id : Int64
      add_index :author_id
      add_index :slug
    end
  end
  
  def rollback
    drop table_for(Post)
  end
end

# db/migrations/00000000000003_add_team_to_users.cr
class AddTeamToUsers::V00000000000003 < Avram::Migrator::Migration::V1
  def migrate
    alter table_for(User) do
      add team_id : Int64?
      add_index :team_id
    end
  end
  
  def rollback
    alter table_for(User) do
      remove :team_id
    end
  end
end
```

## Queries

```crystal
# src/queries/user_query.cr
class UserQuery < User::BaseQuery
  # Scopes
  def active
    active(true)
  end
  
  def inactive
    active(false)
  end
  
  def admin
    role("admin")
  end
  
  def verified
    verified_at.is_not_nil
  end
  
  def unverified
    verified_at.is_nil
  end
  
  def search(q : String)
    name.ilike("%#{q}%").or { |scope|
      scope.email.ilike("%#{q}%")
    }
  end
  
  def by_team(team_id : Int64)
    self.team_id(team_id)
  end
  
  def recent(n : Int32 = 10)
    order_by_created_at(:desc).limit(n)
  end
  
  def paginate(page : Int32 = 1, per_page : Int32 = 20)
    offset((page - 1) * per_page).limit(per_page)
  end
  
  def order_alphabetically
    order_by_name(:asc)
  end
  
  # Complex queries
  def with_post_count
    join_posts
      .group_by_id
      .select_count_of_posts
  end
end

# ใช้งาน
users = UserQuery.new.active.admin.recent(10).select
total = UserQuery.new.active.select_count
user = UserQuery.new.email("test@example.com").first?
admins = UserQuery.new.admin.order_alphabetically.select
```

## CRUD Operations

```crystal
# CREATE ด้วย SaveOperation
class SaveUser < User::SaveOperation
  attribute password : String
  attribute password_confirmation : String
  
  permit_columns email, name, role, bio
  
  before_save do
    validate_required email, name, password
    validate_uniqueness_of email
    hash_password
  end
  
  private def hash_password
    if (pwd = password.value) && !pwd.empty?
      hashed_password.value = Crypto::Bcrypt::Password.create(pwd).to_s
    end
  end
end

# Create new user
SaveUser.create(params) do |op, user|
  if user
    puts "Created: #{user.id}"
  else
    puts "Errors: #{op.errors}"
  end
end

# Create with explicit values
SaveUser.create!(email: "test@example.com", name: "Test", password: "secret123")

# UPDATE
SaveUser.update(existing_user, params) do |op, user|
  # ...
end

SaveUser.update!(user, name: "New Name")

# DELETE
user.delete

# DELETE with query
UserQuery.new.inactive.delete
```

## Associations

```crystal
# has_many
user = UserQuery.find(1_i64)

# Access has_many (lazy loading)
# Lucky ไม่ support lazy loading แบบ Rails
# ต้อง preload
users = UserQuery.new.preload_posts.select
users.each do |u|
  u.posts.each { |p| puts p.title }
end

# belongs_to
post = PostQuery.find(1_i64)
# ต้อง preload ด้วย
posts = PostQuery.new.preload_author.select
posts.each { |p| puts "#{p.title} by #{p.author.name}" }
```

## Raw SQL

```crystal
# ใช้ exec สำหรับ queries ที่ซับซอน
AppDatabase.query("SELECT * FROM users WHERE id = $1", 1_i64) do |rs|
  rs.each do
    id = rs.read(Int64)
    name = rs.read(String)
    email = rs.read(String)
    puts "#{id}: #{name} (#{email})"
  end
end

# exec สำหรับ data modification
AppDatabase.exec("UPDATE users SET active = $1 WHERE id = $2", false, 1_i64)

# scalar query
count = AppDatabase.query_one("SELECT COUNT(*) FROM users WHERE active = true", as: Int64)
```

## Transactions

```crystal
# Transaction ง่ายๆ
AppDatabase.transaction do
  user = SaveUser.create!(email: "new@example.com", name: "New User", password: "pass123")
  profile = SaveProfile.create!(user_id: user.id, bio: "Hello!")
  # ถ้า error เกิดขึ้น จะ rollback อัตโนมัติ
end

# Transaction แบบ explicit
AppDatabase.transaction do |t|
  begin
    # operations...
    t.commit
  rescue ex
    t.rollback
    raise ex
  end
end
```

## Validations

```crystal
class SavePost < Post::SaveOperation
  permit_columns title, content, published, tags
  
  before_save validate_data
  
  private def validate_data
    validate_required title, content
    validate_minimum_length title, 5
    validate_maximum_length title, 200
    validate_minimum_length content, 50
    
    validate_uniqueness_of slug
    
    # Custom validation
    if published.value == true && content.value.try(&.size).try(&.< 100)
      content.add_error "เนื้อหาต้องมีอย่างน้อย 100 ตัวอักษรก่อนตีพิมพ์"
    end
    
    # Generate slug
    if title.changed?
      slug.value = title.value.try { |t| t.downcase.gsub(/[^a-z0-9]+/, "-") }
    end
  end
end
```

## Callbacks

```crystal
class SaveUser < User::SaveOperation
  before_save do
    # ทำงานก่อน save
    validate_required email, name
    normalize_email
  end
  
  after_save do |user|
    # ทำงานหลัง save สำเร็จ
    spawn { send_welcome_email(user) }
  end
  
  private def normalize_email
    if email_value = email.value
      email.value = email_value.downcase.strip
    end
  end
  
  private def send_welcome_email(user : User)
    # Send email...
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Blog System

```crystal
# Models
class Post < BaseModel
  table do
    primary_key id : Int64
    timestamps
    
    column title : String
    column slug : String
    column content : String
    column excerpt : String?
    column published : Bool
    column published_at : Time?
    column view_count : Int64
    
    belongs_to author : User
    has_many comments : Comment
  end
  
  def increment_view!
    AppDatabase.exec(
      "UPDATE posts SET view_count = view_count + 1 WHERE id = $1",
      id
    )
  end
end

# Query
class PostQuery < Post::BaseQuery
  def published
    published(true).published_at.lte(Time.local)
  end
  
  def draft
    published(false)
  end
  
  def by_slug(slug : String)
    self.slug(slug)
  end
  
  def recent
    order_by_published_at(:desc)
  end
  
  def popular
    order_by_view_count(:desc)
  end
  
  def by_tag(tag : String)
    tags.includes(tag)
  end
end

# Operation
class SavePost < Post::SaveOperation
  permit_columns title, content, excerpt, published, tags
  
  before_save do
    validate_required title, content
    validate_minimum_length title, 3
    validate_minimum_length content, 50
    
    # Auto-generate slug
    if title.changed? || slug.value.nil?
      raw_slug = (title.value || "").downcase.gsub(/[^a-z0-9ก-๛]+/, "-")
      slug.value = raw_slug
    end
    
    # Auto-generate excerpt
    if excerpt.value.nil? && content.value
      excerpt.value = (content.value || "")[0, 200]
    end
    
    # Set published_at
    if published.value == true && published_at.value.nil?
      published_at.value = Time.local
    end
  end
end

# Action
class Posts::Create < BrowserAction
  include RequireSignIn
  
  post "/posts" do
    SavePost.create(params, author_id: current_user.id) do |op, post|
      if post
        redirect to: Posts::Show.with(post_id: post.id), notice: "สร้างบทความสำเร็จ!"
      else
        render Posts::NewPage, operation: op
      end
    end
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Models**: กำหนด columns, associations
2. **Migrations**: สร้าง/แก้ไข database schema
3. **Queries**: type-safe database queries
4. **Scopes**: reusable query conditions
5. **SaveOperations**: create/update พร้อม validation
6. **Associations**: has_many, belongs_to, preload
7. **Raw SQL**: สำหรับ complex queries
8. **Transactions**: atomic operations
9. **Validations**: built-in validators
10. **Callbacks**: before_save, after_save

Avram เน้น explicitness - ไม่มี magic, ทุกอย่าง type-safe
