# Part 198: Final Project Part 1 - Full-Stack Web App (Setup and Models)

## บทนำ

ถึงเวลาแล้วที่เราจะนำทุกสิ่งที่เรียนมาตลอดคอร์สมาสร้าง Full-Stack Web Application จริงๆ เราจะสร้าง **BlogCMS** ซึ่งเป็นระบบบล็อก/CMS ที่สมบูรณ์ มีทั้ง REST API, Authentication, File Upload และอื่นๆ อีกมาก

## โครงสร้างโปรเจกต์

```
blogcms/
├── shard.yml
├── shard.lock
├── .env
├── .env.example
├── docker-compose.yml
├── Makefile
├── src/
│   ├── blogcms.cr              # Entry point
│   ├── config/
│   │   ├── database.cr         # Database configuration
│   │   ├── settings.cr         # App settings
│   │   └── cors.cr             # CORS settings
│   ├── models/
│   │   ├── base_model.cr       # Base model
│   │   ├── user.cr             # User model
│   │   ├── post.cr             # Post model
│   │   ├── category.cr         # Category model
│   │   ├── tag.cr              # Tag model
│   │   ├── comment.cr          # Comment model
│   │   ├── attachment.cr       # File attachment
│   │   └── session.cr          # User session
│   ├── repositories/
│   │   ├── base_repository.cr  # Base repository
│   │   ├── user_repository.cr
│   │   ├── post_repository.cr
│   │   └── comment_repository.cr
│   ├── services/
│   │   ├── auth_service.cr     # Authentication
│   │   ├── post_service.cr     # Post management
│   │   ├── upload_service.cr   # File uploads
│   │   └── email_service.cr    # Email sending
│   ├── validators/
│   │   ├── user_validator.cr
│   │   └── post_validator.cr
│   ├── middleware/
│   │   ├── auth_middleware.cr   # JWT auth
│   │   ├── rate_limit.cr        # Rate limiting
│   │   └── request_logger.cr    # Request logging
│   ├── handlers/
│   │   ├── auth_handler.cr
│   │   ├── posts_handler.cr
│   │   ├── users_handler.cr
│   │   └── upload_handler.cr
│   └── utils/
│       ├── slug_generator.cr
│       ├── password_hasher.cr
│       └── jwt_helper.cr
├── db/
│   └── migrations/
│       ├── 001_create_users.sql
│       ├── 002_create_posts.sql
│       ├── 003_create_categories.sql
│       ├── 004_create_tags.sql
│       ├── 005_create_comments.sql
│       └── 006_create_attachments.sql
├── uploads/
│   ├── images/
│   └── documents/
└── spec/
    ├── spec_helper.cr
    ├── models/
    └── handlers/
```

## shard.yml

```yaml
# shard.yml
name: blogcms
version: 1.0.0
description: Full-stack Blog/CMS built with Crystal

authors:
  - Your Name <your@email.com>

crystal: ">= 1.10.0"

dependencies:
  pg:
    github: will/crystal-pg
    version: ~> 0.28.0
  
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
  
  redis:
    github: stefanwille/crystal-redis
    version: ~> 2.9.0
  
  jwt:
    github: crystal-community/jwt
    version: ~> 1.6.0
  
  kemal:
    github: kemalcr/kemal
    version: ~> 1.4.0
  
  kemal-session:
    github: kemalcr/kemal-session
    version: ~> 0.4.0
  
  crypto:
    github: crystal-lang/crystal
    version: "*"

development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: ~> 1.6.0

targets:
  blogcms:
    main: src/blogcms.cr
  
  migrate:
    main: src/migrate.cr
  
  seed:
    main: src/seed.cr
```

## Database Migrations

### SQL Migrations

```sql
-- db/migrations/001_create_users.sql
CREATE TABLE IF NOT EXISTS users (
  id            BIGSERIAL PRIMARY KEY,
  email         VARCHAR(255) NOT NULL UNIQUE,
  username      VARCHAR(50) NOT NULL UNIQUE,
  password_hash VARCHAR(255) NOT NULL,
  full_name     VARCHAR(100),
  bio           TEXT,
  avatar_url    VARCHAR(500),
  role          VARCHAR(20) NOT NULL DEFAULT 'author',
  active        BOOLEAN NOT NULL DEFAULT TRUE,
  email_verified BOOLEAN NOT NULL DEFAULT FALSE,
  verification_token VARCHAR(100),
  reset_token   VARCHAR(100),
  reset_token_expires_at TIMESTAMP WITH TIME ZONE,
  last_login_at TIMESTAMP WITH TIME ZONE,
  created_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_users_email ON users (email);
CREATE INDEX idx_users_username ON users (username);
CREATE INDEX idx_users_role ON users (role);
CREATE INDEX idx_users_active ON users (active) WHERE active = TRUE;

-- db/migrations/002_create_posts.sql
CREATE TABLE IF NOT EXISTS posts (
  id            BIGSERIAL PRIMARY KEY,
  user_id       BIGINT NOT NULL REFERENCES users(id) ON DELETE SET NULL,
  category_id   BIGINT REFERENCES categories(id) ON DELETE SET NULL,
  title         VARCHAR(500) NOT NULL,
  slug          VARCHAR(500) NOT NULL UNIQUE,
  excerpt       TEXT,
  content       TEXT NOT NULL,
  featured_image_url VARCHAR(500),
  status        VARCHAR(20) NOT NULL DEFAULT 'draft',
  visibility    VARCHAR(20) NOT NULL DEFAULT 'public',
  views         BIGINT NOT NULL DEFAULT 0,
  likes         BIGINT NOT NULL DEFAULT 0,
  reading_time  INT,
  published_at  TIMESTAMP WITH TIME ZONE,
  created_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at    TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
  deleted_at    TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_posts_user_id ON posts (user_id);
CREATE INDEX idx_posts_category_id ON posts (category_id);
CREATE INDEX idx_posts_status ON posts (status);
CREATE INDEX idx_posts_published_at ON posts (published_at DESC);
CREATE INDEX idx_posts_slug ON posts (slug);

-- Full-text search index (PostgreSQL)
ALTER TABLE posts ADD COLUMN search_vector tsvector;
CREATE INDEX idx_posts_search ON posts USING GIN (search_vector);

CREATE OR REPLACE FUNCTION update_post_search_vector()
RETURNS TRIGGER AS $$
BEGIN
  NEW.search_vector :=
    setweight(to_tsvector('english', COALESCE(NEW.title, '')), 'A') ||
    setweight(to_tsvector('english', COALESCE(NEW.excerpt, '')), 'B') ||
    setweight(to_tsvector('english', COALESCE(NEW.content, '')), 'C');
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER posts_search_vector_update
  BEFORE INSERT OR UPDATE ON posts
  FOR EACH ROW EXECUTE FUNCTION update_post_search_vector();

-- db/migrations/003_create_categories.sql
CREATE TABLE IF NOT EXISTS categories (
  id          BIGSERIAL PRIMARY KEY,
  name        VARCHAR(100) NOT NULL UNIQUE,
  slug        VARCHAR(100) NOT NULL UNIQUE,
  description TEXT,
  parent_id   BIGINT REFERENCES categories(id),
  sort_order  INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_categories_parent_id ON categories (parent_id);

-- db/migrations/004_create_tags.sql
CREATE TABLE IF NOT EXISTS tags (
  id         BIGSERIAL PRIMARY KEY,
  name       VARCHAR(50) NOT NULL UNIQUE,
  slug       VARCHAR(50) NOT NULL UNIQUE,
  created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE IF NOT EXISTS post_tags (
  post_id BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  tag_id  BIGINT NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
  PRIMARY KEY (post_id, tag_id)
);

CREATE INDEX idx_post_tags_post_id ON post_tags (post_id);
CREATE INDEX idx_post_tags_tag_id ON post_tags (tag_id);

-- db/migrations/005_create_comments.sql
CREATE TABLE IF NOT EXISTS comments (
  id          BIGSERIAL PRIMARY KEY,
  post_id     BIGINT NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  user_id     BIGINT REFERENCES users(id) ON DELETE SET NULL,
  parent_id   BIGINT REFERENCES comments(id) ON DELETE CASCADE,
  author_name VARCHAR(100),
  author_email VARCHAR(255),
  content     TEXT NOT NULL,
  status      VARCHAR(20) NOT NULL DEFAULT 'pending',
  created_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at  TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_comments_post_id ON comments (post_id);
CREATE INDEX idx_comments_user_id ON comments (user_id);
CREATE INDEX idx_comments_status ON comments (status);

-- db/migrations/006_create_attachments.sql
CREATE TABLE IF NOT EXISTS attachments (
  id           BIGSERIAL PRIMARY KEY,
  user_id      BIGINT REFERENCES users(id) ON DELETE SET NULL,
  post_id      BIGINT REFERENCES posts(id) ON DELETE SET NULL,
  filename     VARCHAR(255) NOT NULL,
  original_name VARCHAR(255) NOT NULL,
  file_path    VARCHAR(500) NOT NULL,
  file_size    BIGINT NOT NULL,
  content_type VARCHAR(100) NOT NULL,
  width        INT,
  height       INT,
  created_at   TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_attachments_user_id ON attachments (user_id);
CREATE INDEX idx_attachments_post_id ON attachments (post_id);
```

## Models

### Base Model

```crystal
# src/models/base_model.cr
abstract class BaseModel
  macro property_getter(name, type)
    def {{name.id}} : {{type}}
      @{{name.id}}.not_nil!
    end
    
    def {{name.id}}? : {{type}}?
      @{{name.id}}
    end
  end
  
  def to_json_hash : Hash(String, JSON::Any)
    {} of String => JSON::Any
  end
  
  def to_json(io : IO) : Nil
    to_json_hash.to_json(io)
  end
  
  def to_json : String
    String.build { |io| to_json(io) }
  end
end
```

### User Model

```crystal
# src/models/user.cr
require "json"
require "crypto/bcrypt/password"

class User < BaseModel
  include JSON::Serializable
  
  @[JSON::Field(ignore: true)]
  property password_hash : String = ""
  
  property id : Int64?
  property email : String
  property username : String
  property full_name : String?
  property bio : String?
  property avatar_url : String?
  property role : String
  property active : Bool
  property email_verified : Bool
  property last_login_at : Time?
  property created_at : Time?
  property updated_at : Time?
  
  ROLES = %w[admin editor author subscriber]
  
  def initialize(
    @email : String,
    @username : String,
    @role : String = "author",
    @active : Bool = true,
    @email_verified : Bool = false,
    @id : Int64? = nil,
    @full_name : String? = nil,
    @bio : String? = nil,
    @avatar_url : String? = nil,
    @last_login_at : Time? = nil,
    @created_at : Time? = nil,
    @updated_at : Time? = nil
  )
  end
  
  # ตั้งรหัสผ่าน
  def password=(plain_password : String)
    raise "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร" if plain_password.size < 8
    @password_hash = Crypto::Bcrypt::Password.create(plain_password).to_s
  end
  
  # ตรวจสอบรหัสผ่าน
  def authenticate(plain_password : String) : Bool
    return false if @password_hash.empty?
    Crypto::Bcrypt::Password.new(@password_hash).verify(plain_password)
  end
  
  # ตรวจสอบสิทธิ์
  def admin? : Bool
    role == "admin"
  end
  
  def editor? : Bool
    %w[admin editor].includes?(role)
  end
  
  def can_publish? : Bool
    %w[admin editor author].includes?(role)
  end
  
  def active? : Bool
    active
  end
  
  # Serialization ปลอดภัย (ไม่รวม sensitive fields)
  def to_public_json : String
    {
      "id"         => id,
      "username"   => username,
      "full_name"  => full_name,
      "bio"        => bio,
      "avatar_url" => avatar_url,
      "role"       => role,
      "created_at" => created_at.try(&.to_rfc3339)
    }.to_json
  end
  
  def to_admin_json : String
    {
      "id"             => id,
      "email"          => email,
      "username"       => username,
      "full_name"      => full_name,
      "bio"            => bio,
      "avatar_url"     => avatar_url,
      "role"           => role,
      "active"         => active,
      "email_verified" => email_verified,
      "last_login_at"  => last_login_at.try(&.to_rfc3339),
      "created_at"     => created_at.try(&.to_rfc3339)
    }.to_json
  end
end
```

### Post Model

```crystal
# src/models/post.cr
require "json"

class Post < BaseModel
  include JSON::Serializable
  
  property id : Int64?
  property user_id : Int64
  property category_id : Int64?
  property title : String
  property slug : String
  property excerpt : String?
  property content : String
  property featured_image_url : String?
  property status : String
  property visibility : String
  property views : Int64
  property likes : Int64
  property reading_time : Int32?
  property published_at : Time?
  property created_at : Time?
  property updated_at : Time?
  
  # Virtual fields (ไม่บันทึก DB)
  @[JSON::Field(ignore: true)]
  property author : User? = nil
  
  @[JSON::Field(ignore: true)]
  property category : Category? = nil
  
  @[JSON::Field(ignore: true)]
  property tags : Array(Tag) = [] of Tag
  
  @[JSON::Field(ignore: true)]
  property comments_count : Int64 = 0_i64
  
  STATUSES = %w[draft published archived scheduled]
  VISIBILITIES = %w[public private unlisted]
  
  def initialize(
    @user_id : Int64,
    @title : String,
    @content : String,
    @slug : String = "",
    @category_id : Int64? = nil,
    @excerpt : String? = nil,
    @featured_image_url : String? = nil,
    @status : String = "draft",
    @visibility : String = "public",
    @views : Int64 = 0_i64,
    @likes : Int64 = 0_i64,
    @reading_time : Int32? = nil,
    @published_at : Time? = nil,
    @id : Int64? = nil,
    @created_at : Time? = nil,
    @updated_at : Time? = nil
  )
    # คำนวณ reading time ถ้าไม่ได้ระบุ
    @reading_time ||= calculate_reading_time(content)
  end
  
  def published? : Bool
    status == "published"
  end
  
  def draft? : Bool
    status == "draft"
  end
  
  def visible_to_public? : Bool
    published? && visibility == "public"
  end
  
  # คำนวณเวลาอ่านโดยประมาณ (นาที)
  private def calculate_reading_time(content : String) : Int32
    words = content.split(/\s+/).size
    (words / 200.0).ceil.to_i  # เฉลี่ย 200 คำต่อนาที
  end
  
  def to_list_json : String
    {
      "id"                 => id,
      "title"              => title,
      "slug"               => slug,
      "excerpt"            => excerpt,
      "featured_image_url" => featured_image_url,
      "status"             => status,
      "views"              => views,
      "likes"              => likes,
      "reading_time"       => reading_time,
      "published_at"       => published_at.try(&.to_rfc3339),
      "created_at"         => created_at.try(&.to_rfc3339),
      "author"             => author.try { |u| {
        "id" => u.id, "username" => u.username, "avatar_url" => u.avatar_url
      }},
      "tags"               => tags.map { |t| {"id" => t.id, "name" => t.name, "slug" => t.slug} },
      "comments_count"     => comments_count
    }.to_json
  end
  
  def to_detail_json : String
    {
      "id"                 => id,
      "title"              => title,
      "slug"               => slug,
      "excerpt"            => excerpt,
      "content"            => content,
      "featured_image_url" => featured_image_url,
      "status"             => status,
      "visibility"         => visibility,
      "views"              => views,
      "likes"              => likes,
      "reading_time"       => reading_time,
      "published_at"       => published_at.try(&.to_rfc3339),
      "created_at"         => created_at.try(&.to_rfc3339),
      "updated_at"         => updated_at.try(&.to_rfc3339),
      "author"             => author.try { |u| {
        "id" => u.id, "username" => u.username, "full_name" => u.full_name, 
        "avatar_url" => u.avatar_url, "bio" => u.bio
      }},
      "category"           => category.try { |c| {"id" => c.id, "name" => c.name, "slug" => c.slug} },
      "tags"               => tags.map { |t| {"id" => t.id, "name" => t.name, "slug" => t.slug} },
      "comments_count"     => comments_count
    }.to_json
  end
end
```

### Category และ Tag Models

```crystal
# src/models/category.cr
class Category < BaseModel
  include JSON::Serializable
  
  property id : Int64?
  property name : String
  property slug : String
  property description : String?
  property parent_id : Int64?
  property sort_order : Int32
  property created_at : Time?
  
  @[JSON::Field(ignore: true)]
  property children : Array(Category) = [] of Category
  
  @[JSON::Field(ignore: true)]
  property posts_count : Int64 = 0_i64
  
  def initialize(
    @name : String,
    @slug : String,
    @description : String? = nil,
    @parent_id : Int64? = nil,
    @sort_order : Int32 = 0,
    @id : Int64? = nil,
    @created_at : Time? = nil
  )
  end
end

# src/models/tag.cr
class Tag < BaseModel
  include JSON::Serializable
  
  property id : Int64?
  property name : String
  property slug : String
  property created_at : Time?
  
  @[JSON::Field(ignore: true)]
  property posts_count : Int64 = 0_i64
  
  def initialize(
    @name : String,
    @slug : String,
    @id : Int64? = nil,
    @created_at : Time? = nil
  )
  end
end

# src/models/comment.cr
class Comment < BaseModel
  include JSON::Serializable
  
  property id : Int64?
  property post_id : Int64
  property user_id : Int64?
  property parent_id : Int64?
  property author_name : String?
  property author_email : String?
  property content : String
  property status : String
  property created_at : Time?
  property updated_at : Time?
  
  @[JSON::Field(ignore: true)]
  property author : User? = nil
  
  @[JSON::Field(ignore: true)]
  property replies : Array(Comment) = [] of Comment
  
  STATUSES = %w[pending approved spam]
  
  def initialize(
    @post_id : Int64,
    @content : String,
    @user_id : Int64? = nil,
    @parent_id : Int64? = nil,
    @author_name : String? = nil,
    @author_email : String? = nil,
    @status : String = "pending",
    @id : Int64? = nil,
    @created_at : Time? = nil,
    @updated_at : Time? = nil
  )
  end
  
  def approved? : Bool
    status == "approved"
  end
end
```

## Repositories

### Base Repository

```crystal
# src/repositories/base_repository.cr
require "db"
require "pg"

abstract class BaseRepository(T)
  def initialize(@db : DB::Database)
  end
  
  abstract def find_by_id(id : Int64) : T?
  abstract def create(model : T) : T
  abstract def update(model : T) : Bool
  abstract def delete(id : Int64) : Bool
  
  def transaction(&block)
    @db.transaction do |tx|
      yield tx
    end
  end
end
```

### User Repository

```crystal
# src/repositories/user_repository.cr
require "./base_repository"
require "../models/user"

class UserRepository < BaseRepository(User)
  def find_by_id(id : Int64) : User?
    @db.query_one?(
      "SELECT id, email, username, password_hash, full_name, bio, avatar_url, 
              role, active, email_verified, last_login_at, created_at, updated_at 
       FROM users WHERE id = $1 AND deleted_at IS NULL",
      id
    ) { |rs| map_user(rs) }
  end
  
  def find_by_email(email : String) : User?
    @db.query_one?(
      "SELECT id, email, username, password_hash, full_name, bio, avatar_url,
              role, active, email_verified, last_login_at, created_at, updated_at
       FROM users WHERE email = $1 AND deleted_at IS NULL",
      email
    ) { |rs| map_user(rs) }
  end
  
  def find_by_username(username : String) : User?
    @db.query_one?(
      "SELECT id, email, username, password_hash, full_name, bio, avatar_url,
              role, active, email_verified, last_login_at, created_at, updated_at
       FROM users WHERE username = $1 AND deleted_at IS NULL",
      username
    ) { |rs| map_user(rs) }
  end
  
  def find_all(
    role : String? = nil,
    active : Bool? = nil,
    limit : Int32 = 20,
    offset : Int32 = 0
  ) : Array(User)
    conditions = ["deleted_at IS NULL"]
    params = [] of DB::Any
    param_idx = 1
    
    if role
      conditions << "role = $#{param_idx}"
      params << role
      param_idx += 1
    end
    
    unless active.nil?
      conditions << "active = $#{param_idx}"
      params << active
      param_idx += 1
    end
    
    params << limit
    params << offset
    
    query = "SELECT id, email, username, password_hash, full_name, bio, avatar_url,
                    role, active, email_verified, last_login_at, created_at, updated_at
             FROM users 
             WHERE #{conditions.join(" AND ")} 
             ORDER BY created_at DESC 
             LIMIT $#{param_idx} OFFSET $#{param_idx + 1}"
    
    users = [] of User
    @db.query(query, *params) do |rs|
      rs.each { users << map_user(rs) }
    end
    users
  end
  
  def create(user : User) : User
    result = @db.query_one(
      "INSERT INTO users (email, username, password_hash, full_name, bio, avatar_url, role, active, email_verified) 
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9) 
       RETURNING id, created_at, updated_at",
      user.email, user.username, user.password_hash, user.full_name,
      user.bio, user.avatar_url, user.role, user.active, user.email_verified
    ) do |rs|
      {id: rs.read(Int64), created_at: rs.read(Time), updated_at: rs.read(Time)}
    end
    
    user.id = result[:id]
    user.created_at = result[:created_at]
    user.updated_at = result[:updated_at]
    user
  end
  
  def update(user : User) : Bool
    raise "User ไม่มี ID" unless user.id
    
    result = @db.exec(
      "UPDATE users 
       SET email = $1, username = $2, full_name = $3, bio = $4, avatar_url = $5,
           role = $6, active = $7, email_verified = $8, updated_at = NOW()
       WHERE id = $9 AND deleted_at IS NULL",
      user.email, user.username, user.full_name, user.bio, user.avatar_url,
      user.role, user.active, user.email_verified, user.id
    )
    result.rows_affected > 0
  end
  
  def update_password(id : Int64, new_hash : String) : Bool
    result = @db.exec(
      "UPDATE users SET password_hash = $1, updated_at = NOW() WHERE id = $2",
      new_hash, id
    )
    result.rows_affected > 0
  end
  
  def update_last_login(id : Int64)
    @db.exec(
      "UPDATE users SET last_login_at = NOW() WHERE id = $1",
      id
    )
  end
  
  def delete(id : Int64) : Bool
    result = @db.exec(
      "UPDATE users SET deleted_at = NOW(), active = FALSE WHERE id = $1",
      id
    )
    result.rows_affected > 0
  end
  
  def count(role : String? = nil, active : Bool? = nil) : Int64
    conditions = ["deleted_at IS NULL"]
    params = [] of DB::Any
    param_idx = 1
    
    if role
      conditions << "role = $#{param_idx}"
      params << role
      param_idx += 1
    end
    
    unless active.nil?
      conditions << "active = $#{param_idx}"
      params << active
    end
    
    @db.query_one(
      "SELECT COUNT(*) FROM users WHERE #{conditions.join(" AND ")}",
      *params,
      as: Int64
    )
  end
  
  private def map_user(rs : DB::ResultSet) : User
    user = User.new(
      id:             rs.read(Int64),
      email:          rs.read(String),
      username:       rs.read(String),
      full_name:      rs.read(String?),
      bio:            rs.read(String?),
      avatar_url:     rs.read(String?),
      role:           rs.read(String),
      active:         rs.read(Bool),
      email_verified: rs.read(Bool),
      last_login_at:  rs.read(Time?),
      created_at:     rs.read(Time?),
      updated_at:     rs.read(Time?)
    )
    user.password_hash = rs.read(String)
    user
  end
end
```

### Post Repository

```crystal
# src/repositories/post_repository.cr
require "./base_repository"
require "../models/post"

class PostRepository < BaseRepository(Post)
  def find_by_id(id : Int64) : Post?
    post = @db.query_one?(
      "SELECT p.id, p.user_id, p.category_id, p.title, p.slug, p.excerpt, p.content,
              p.featured_image_url, p.status, p.visibility, p.views, p.likes,
              p.reading_time, p.published_at, p.created_at, p.updated_at
       FROM posts p
       WHERE p.id = $1 AND p.deleted_at IS NULL",
      id
    ) { |rs| map_post(rs) }
    
    load_post_relations(post) if post
    post
  end
  
  def find_by_slug(slug : String) : Post?
    post = @db.query_one?(
      "SELECT p.id, p.user_id, p.category_id, p.title, p.slug, p.excerpt, p.content,
              p.featured_image_url, p.status, p.visibility, p.views, p.likes,
              p.reading_time, p.published_at, p.created_at, p.updated_at
       FROM posts p
       WHERE p.slug = $1 AND p.deleted_at IS NULL",
      slug
    ) { |rs| map_post(rs) }
    
    load_post_relations(post) if post
    post
  end
  
  def find_published(
    category_id : Int64? = nil,
    tag_slug : String? = nil,
    author_id : Int64? = nil,
    limit : Int32 = 10,
    offset : Int32 = 0
  ) : Array(Post)
    conditions = ["p.status = 'published'", "p.visibility = 'public'", "p.deleted_at IS NULL"]
    params = [] of DB::Any
    param_idx = 1
    
    joins = ""
    
    if category_id
      conditions << "p.category_id = $#{param_idx}"
      params << category_id
      param_idx += 1
    end
    
    if tag_slug
      joins += " JOIN post_tags pt ON p.id = pt.post_id JOIN tags t ON pt.tag_id = t.id"
      conditions << "t.slug = $#{param_idx}"
      params << tag_slug
      param_idx += 1
    end
    
    if author_id
      conditions << "p.user_id = $#{param_idx}"
      params << author_id
      param_idx += 1
    end
    
    params << limit
    params << offset
    
    posts = [] of Post
    @db.query(
      "SELECT p.id, p.user_id, p.category_id, p.title, p.slug, p.excerpt, p.content,
              p.featured_image_url, p.status, p.visibility, p.views, p.likes,
              p.reading_time, p.published_at, p.created_at, p.updated_at
       FROM posts p #{joins}
       WHERE #{conditions.join(" AND ")}
       ORDER BY p.published_at DESC
       LIMIT $#{param_idx} OFFSET $#{param_idx + 1}",
      *params
    ) do |rs|
      rs.each { posts << map_post(rs) }
    end
    
    # Load relations
    load_batch_relations(posts)
    posts
  end
  
  def create(post : Post) : Post
    result = @db.query_one(
      "INSERT INTO posts (user_id, category_id, title, slug, excerpt, content, 
                          featured_image_url, status, visibility, reading_time, published_at)
       VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11)
       RETURNING id, created_at, updated_at",
      post.user_id, post.category_id, post.title, post.slug, post.excerpt, post.content,
      post.featured_image_url, post.status, post.visibility, post.reading_time, post.published_at
    ) do |rs|
      {id: rs.read(Int64), created_at: rs.read(Time), updated_at: rs.read(Time)}
    end
    
    post.id = result[:id]
    post.created_at = result[:created_at]
    post.updated_at = result[:updated_at]
    
    # บันทึก tags
    save_tags(post)
    
    post
  end
  
  def update(post : Post) : Bool
    raise "Post ไม่มี ID" unless post.id
    
    result = @db.exec(
      "UPDATE posts 
       SET title = $1, category_id = $2, excerpt = $3, content = $4,
           featured_image_url = $5, status = $6, visibility = $7,
           reading_time = $8, published_at = $9, updated_at = NOW()
       WHERE id = $10 AND deleted_at IS NULL",
      post.title, post.category_id, post.excerpt, post.content,
      post.featured_image_url, post.status, post.visibility,
      post.reading_time, post.published_at, post.id
    )
    
    # อัปเดต tags
    save_tags(post)
    
    result.rows_affected > 0
  end
  
  def delete(id : Int64) : Bool
    result = @db.exec(
      "UPDATE posts SET deleted_at = NOW() WHERE id = $1",
      id
    )
    result.rows_affected > 0
  end
  
  def increment_views(id : Int64)
    @db.exec("UPDATE posts SET views = views + 1 WHERE id = $1", id)
  end
  
  def count_by_status : Hash(String, Int64)
    counts = {} of String => Int64
    @db.query("SELECT status, COUNT(*) FROM posts WHERE deleted_at IS NULL GROUP BY status") do |rs|
      rs.each { counts[rs.read(String)] = rs.read(Int64) }
    end
    counts
  end
  
  private def save_tags(post : Post)
    post_id = post.id.not_nil!
    
    # ลบ tags เก่า
    @db.exec("DELETE FROM post_tags WHERE post_id = $1", post_id)
    
    # เพิ่ม tags ใหม่
    post.tags.each do |tag|
      next unless tag_id = tag.id
      @db.exec("INSERT INTO post_tags (post_id, tag_id) VALUES ($1, $2) ON CONFLICT DO NOTHING",
        post_id, tag_id)
    end
  end
  
  private def load_post_relations(post : Post)
    load_batch_relations([post])
  end
  
  private def load_batch_relations(posts : Array(Post))
    return if posts.empty?
    
    post_ids = posts.map(&.id.not_nil!).join(",")
    user_ids = posts.map(&.user_id).uniq.join(",")
    
    # Load authors
    authors = {} of Int64 => User
    @db.query(
      "SELECT id, username, full_name, bio, avatar_url, role, created_at 
       FROM users WHERE id IN (#{user_ids})"
    ) do |rs|
      rs.each do
        id = rs.read(Int64)
        authors[id] = User.new(
          id: id,
          email: "",  # ไม่แสดง email ใน public
          username: rs.read(String),
          full_name: rs.read(String?),
          bio: rs.read(String?),
          avatar_url: rs.read(String?),
          role: rs.read(String),
          created_at: rs.read(Time?)
        )
      end
    end
    
    # Load tags
    post_tags = Hash(Int64, Array(Tag)).new { |h, k| h[k] = [] of Tag }
    @db.query(
      "SELECT pt.post_id, t.id, t.name, t.slug 
       FROM post_tags pt JOIN tags t ON pt.tag_id = t.id 
       WHERE pt.post_id IN (#{post_ids})"
    ) do |rs|
      rs.each do
        post_id = rs.read(Int64)
        tag = Tag.new(
          id: rs.read(Int64),
          name: rs.read(String),
          slug: rs.read(String)
        )
        post_tags[post_id] << tag
      end
    end
    
    # Load comment counts
    comment_counts = {} of Int64 => Int64
    @db.query(
      "SELECT post_id, COUNT(*) FROM comments 
       WHERE post_id IN (#{post_ids}) AND status = 'approved' 
       GROUP BY post_id"
    ) do |rs|
      rs.each { comment_counts[rs.read(Int64)] = rs.read(Int64) }
    end
    
    # Assign relations
    posts.each do |post|
      post.author = authors[post.user_id]?
      post.tags = post_tags[post.id.not_nil!]? || [] of Tag
      post.comments_count = comment_counts[post.id.not_nil!]? || 0_i64
    end
  end
  
  private def map_post(rs : DB::ResultSet) : Post
    Post.new(
      id:                 rs.read(Int64),
      user_id:            rs.read(Int64),
      category_id:        rs.read(Int64?),
      title:              rs.read(String),
      slug:               rs.read(String),
      excerpt:            rs.read(String?),
      content:            rs.read(String),
      featured_image_url: rs.read(String?),
      status:             rs.read(String),
      visibility:         rs.read(String),
      views:              rs.read(Int64),
      likes:              rs.read(Int64),
      reading_time:       rs.read(Int32?),
      published_at:       rs.read(Time?),
      created_at:         rs.read(Time?),
      updated_at:         rs.read(Time?)
    )
  end
end
```

## Authentication System

### JWT Helper

```crystal
# src/utils/jwt_helper.cr
require "jwt"
require "json"

module JWTHelper
  SECRET = ENV.fetch("JWT_SECRET", "your-super-secret-key-change-in-production")
  ALGORITHM = JWT::Algorithm::HS256
  
  ACCESS_TOKEN_TTL  = 1.hour
  REFRESH_TOKEN_TTL = 30.days
  
  def self.generate_access_token(user_id : Int64, role : String) : String
    payload = {
      "sub"  => user_id.to_s,
      "role" => role,
      "type" => "access",
      "iat"  => Time.utc.to_unix,
      "exp"  => (Time.utc + ACCESS_TOKEN_TTL).to_unix
    }
    JWT.encode(payload, SECRET, ALGORITHM)
  end
  
  def self.generate_refresh_token(user_id : Int64) : String
    payload = {
      "sub"  => user_id.to_s,
      "type" => "refresh",
      "iat"  => Time.utc.to_unix,
      "exp"  => (Time.utc + REFRESH_TOKEN_TTL).to_unix
    }
    JWT.encode(payload, SECRET, ALGORITHM)
  end
  
  def self.decode(token : String) : Hash(String, JSON::Any)?
    payload, _ = JWT.decode(token, SECRET, ALGORITHM)
    
    # ตรวจสอบว่า expire หรือไม่
    if exp = payload["exp"]?.try(&.as_i64)
      return nil if Time.utc.to_unix > exp
    end
    
    payload.as_h.transform_values { |v| JSON::Any.new(v.raw) }
  rescue JWT::ExpiredSignatureError, JWT::VerificationError, JWT::DecodeError
    nil
  end
  
  def self.extract_user_id(token : String) : Int64?
    claims = decode(token)
    claims.try { |c| c["sub"]?.try(&.as_s.to_i64) }
  end
  
  def self.token_type(token : String) : String?
    claims = decode(token)
    claims.try { |c| c["type"]?.try(&.as_s) }
  end
end
```

### Auth Service

```crystal
# src/services/auth_service.cr
require "../repositories/user_repository"
require "../utils/jwt_helper"
require "crypto/bcrypt/password"
require "random/secure"

class AuthService
  def initialize(
    @user_repo : UserRepository,
    @redis : Redis::PooledClient? = nil
  )
  end
  
  # Login
  def login(email : String, password : String) : NamedTuple(
    success: Bool,
    user: User?,
    access_token: String?,
    refresh_token: String?,
    error: String?
  )
    user = @user_repo.find_by_email(email)
    
    return {success: false, user: nil, access_token: nil, refresh_token: nil, error: "ไม่พบผู้ใช้"} unless user
    return {success: false, user: nil, access_token: nil, refresh_token: nil, error: "บัญชีถูกระงับ"} unless user.active?
    return {success: false, user: nil, access_token: nil, refresh_token: nil, error: "รหัสผ่านไม่ถูกต้อง"} unless user.authenticate(password)
    
    access_token  = JWTHelper.generate_access_token(user.id.not_nil!, user.role)
    refresh_token = JWTHelper.generate_refresh_token(user.id.not_nil!)
    
    # บันทึก refresh token ใน Redis (ถ้ามี)
    if redis = @redis
      redis.setex(
        "refresh_token:#{refresh_token}",
        (30 * 24 * 3600),
        user.id.to_s
      )
    end
    
    # อัปเดต last_login
    @user_repo.update_last_login(user.id.not_nil!)
    
    {
      success:       true,
      user:          user,
      access_token:  access_token,
      refresh_token: refresh_token,
      error:         nil
    }
  end
  
  # Register
  def register(
    email : String,
    username : String,
    password : String,
    full_name : String? = nil
  ) : NamedTuple(success: Bool, user: User?, error: String?)
    # ตรวจสอบว่า email ซ้ำไหม
    if @user_repo.find_by_email(email)
      return {success: false, user: nil, error: "อีเมลนี้ถูกใช้แล้ว"}
    end
    
    # ตรวจสอบว่า username ซ้ำไหม
    if @user_repo.find_by_username(username)
      return {success: false, user: nil, error: "Username นี้ถูกใช้แล้ว"}
    end
    
    # ตรวจสอบรหัสผ่าน
    if password.size < 8
      return {success: false, user: nil, error: "รหัสผ่านต้องมีอย่างน้อย 8 ตัวอักษร"}
    end
    
    user = User.new(
      email:     email,
      username:  username,
      full_name: full_name,
      role:      "author"
    )
    user.password = password
    
    saved_user = @user_repo.create(user)
    {success: true, user: saved_user, error: nil}
  rescue ex
    {success: false, user: nil, error: ex.message}
  end
  
  # Refresh Token
  def refresh(refresh_token : String) : NamedTuple(
    success: Bool,
    access_token: String?,
    error: String?
  )
    claims = JWTHelper.decode(refresh_token)
    
    unless claims && claims["type"]?.try(&.as_s) == "refresh"
      return {success: false, access_token: nil, error: "Token ไม่ถูกต้อง"}
    end
    
    user_id = claims["sub"]?.try(&.as_s.to_i64)
    return {success: false, access_token: nil, error: "Token ไม่ถูกต้อง"} unless user_id
    
    # ตรวจสอบใน Redis
    if redis = @redis
      stored_id = redis.get("refresh_token:#{refresh_token}")
      return {success: false, access_token: nil, error: "Token หมดอายุแล้ว"} unless stored_id
    end
    
    user = @user_repo.find_by_id(user_id)
    return {success: false, access_token: nil, error: "ไม่พบผู้ใช้"} unless user
    return {success: false, access_token: nil, error: "บัญชีถูกระงับ"} unless user.active?
    
    new_access_token = JWTHelper.generate_access_token(user_id, user.role)
    {success: true, access_token: new_access_token, error: nil}
  end
  
  # Logout
  def logout(refresh_token : String)
    if redis = @redis
      redis.del("refresh_token:#{refresh_token}")
    end
  end
  
  # ตรวจสอบ token
  def verify_token(token : String) : User?
    user_id = JWTHelper.extract_user_id(token)
    return nil unless user_id
    
    @user_repo.find_by_id(user_id)
  end
end
```

## สรุป Part 1

ใน Part 1 เราได้สร้างพื้นฐานของ BlogCMS:
- **โครงสร้างโปรเจกต์** ที่ครบถ้วน
- **Database Schema** ด้วย PostgreSQL migrations
- **Models** ครบทุกตาราง (User, Post, Category, Tag, Comment)
- **Repositories** สำหรับ data access layer
- **Authentication** ด้วย JWT tokens
- **Services** หลักอย่าง AuthService

## ขั้นตอนต่อไป

ใน **Part 199** เราจะสร้าง:
- REST API handlers ที่สมบูรณ์
- JSON serialization
- Caching ด้วย Redis
- Search functionality
- Testing
