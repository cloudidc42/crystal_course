# Part 199: Final Project Part 2 - API and Frontend Integration

## บทนำ

ใน Part 2 เราจะสร้าง REST API layer ที่สมบูรณ์สำหรับ BlogCMS รวมถึง Middleware, JSON serialization ที่ดี, Redis caching, Search functionality และ Test suite

## Middleware

### Authentication Middleware

```crystal
# src/middleware/auth_middleware.cr
require "kemal"
require "../utils/jwt_helper"
require "../repositories/user_repository"

class AuthMiddleware < Kemal::Handler
  EXCLUDE_PATHS = [
    {method: "POST", path: "/api/auth/login"},
    {method: "POST", path: "/api/auth/register"},
    {method: "POST", path: "/api/auth/refresh"},
    {method: "GET",  path: "/api/posts"},
    {method: "GET",  path: /^\/api\/posts\/.+$/},
    {method: "GET",  path: "/api/categories"},
    {method: "GET",  path: "/api/tags"},
  ]
  
  def initialize(@user_repo : UserRepository)
  end
  
  def call(context : HTTP::Server::Context)
    # ตรวจสอบว่าต้อง auth หรือไม่
    unless requires_auth?(context)
      call_next(context)
      return
    end
    
    # ดึง token จาก header
    auth_header = context.request.headers["Authorization"]?
    unless auth_header && auth_header.starts_with?("Bearer ")
      context.response.status_code = 401
      context.response.content_type = "application/json"
      context.response.print({"error" => "กรุณา login ก่อน"}.to_json)
      return
    end
    
    token = auth_header.lstrip("Bearer ").strip
    
    # ตรวจสอบ token
    user = @user_repo.find_by_id(JWTHelper.extract_user_id(token) || 0_i64)
    
    unless user && user.active?
      context.response.status_code = 401
      context.response.content_type = "application/json"
      context.response.print({"error" => "Token ไม่ถูกต้องหรือหมดอายุ"}.to_json)
      return
    end
    
    # บันทึกข้อมูล user ลง context
    context.set("current_user", user)
    call_next(context)
  end
  
  private def requires_auth?(context : HTTP::Server::Context) : Bool
    path = context.request.path
    method = context.request.method
    
    EXCLUDE_PATHS.none? do |excluded|
      method_match = excluded[:method] == method
      path_match = case excluded[:path]
                   when String then excluded[:path] == path
                   when Regex  then path.matches?(excluded[:path])
                   else false
                   end
      method_match && path_match
    end
  end
end

# Helper method สำหรับ Handler
module AuthHelper
  def current_user(env : HTTP::Server::Context) : User
    env.get("current_user").as(User)
  end
  
  def current_user?(env : HTTP::Server::Context) : User?
    env.get?("current_user").try(&.as(User))
  end
  
  def require_role(env : HTTP::Server::Context, *roles : String)
    user = current_user(env)
    unless roles.includes?(user.role)
      halt env, status_code: 403, response: {"error" => "ไม่มีสิทธิ์"}.to_json
    end
  end
end
```

### Rate Limiting Middleware

```crystal
# src/middleware/rate_limit.cr
require "kemal"
require "redis"

class RateLimitMiddleware < Kemal::Handler
  # Limits per endpoint type
  LIMITS = {
    "auth"   => {requests: 10, window: 60},   # 10 requests/minute
    "api"    => {requests: 100, window: 60},  # 100 requests/minute
    "upload" => {requests: 20, window: 60},   # 20 requests/minute
    "search" => {requests: 30, window: 60}    # 30 requests/minute
  }
  
  def initialize(@redis : Redis::PooledClient)
  end
  
  def call(context : HTTP::Server::Context)
    path = context.request.path
    ip = get_client_ip(context)
    
    limit_type = determine_limit_type(path)
    limit = LIMITS[limit_type]
    
    key = "rate_limit:#{limit_type}:#{ip}"
    current = @redis.get(key).try(&.to_i) || 0
    
    if current >= limit[:requests]
      context.response.status_code = 429
      context.response.content_type = "application/json"
      context.response.headers["Retry-After"] = limit[:window].to_s
      context.response.headers["X-RateLimit-Limit"] = limit[:requests].to_s
      context.response.headers["X-RateLimit-Remaining"] = "0"
      context.response.print({"error" => "Too Many Requests", "retry_after" => limit[:window]}.to_json)
      return
    end
    
    # อัปเดต counter
    if current == 0
      @redis.setex(key, limit[:window], "1")
    else
      @redis.incr(key)
    end
    
    remaining = limit[:requests] - current - 1
    context.response.headers["X-RateLimit-Limit"] = limit[:requests].to_s
    context.response.headers["X-RateLimit-Remaining"] = [remaining, 0].max.to_s
    
    call_next(context)
  end
  
  private def get_client_ip(context : HTTP::Server::Context) : String
    context.request.headers["X-Forwarded-For"]?.try(&.split(",").first.strip) ||
    context.request.headers["X-Real-IP"]? ||
    context.request.remote_address.try(&.to_s) ||
    "unknown"
  end
  
  private def determine_limit_type(path : String) : String
    case path
    when /^\/api\/auth/ then "auth"
    when /^\/api\/upload/ then "upload"
    when /^\/api\/search/ then "search"
    else "api"
    end
  end
end
```

### Request Logger Middleware

```crystal
# src/middleware/request_logger.cr
require "kemal"

class RequestLoggerMiddleware < Kemal::Handler
  def call(context : HTTP::Server::Context)
    start_time = Time.monotonic
    
    call_next(context)
    
    elapsed = (Time.monotonic - start_time).total_milliseconds
    
    # Color coding สำหรับ status
    status = context.response.status_code
    status_str = case status
                 when 200..299 then "\e[32m#{status}\e[0m"  # Green
                 when 300..399 then "\e[33m#{status}\e[0m"  # Yellow
                 when 400..499 then "\e[31m#{status}\e[0m"  # Red
                 else               "\e[35m#{status}\e[0m"  # Magenta
                 end
    
    STDOUT.puts "[#{Time.utc.to_s("%H:%M:%S")}] #{context.request.method.ljust(7)} #{context.request.path.ljust(40)} #{status_str} #{elapsed.round(1)}ms"
  end
end
```

## API Handlers

### Auth Handler

```crystal
# src/handlers/auth_handler.cr
require "kemal"
require "../services/auth_service"

module AuthHandler
  include AuthHelper
  
  def self.setup(auth_service : AuthService)
    # POST /api/auth/register
    post "/api/auth/register" do |env|
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      
      email     = data["email"]?.try(&.as_s) || ""
      username  = data["username"]?.try(&.as_s) || ""
      password  = data["password"]?.try(&.as_s) || ""
      full_name = data["full_name"]?.try(&.as_s)
      
      # Validation
      errors = [] of String
      errors << "กรุณาระบุอีเมล" if email.empty?
      errors << "กรุณาระบุ username" if username.empty?
      errors << "กรุณาระบุรหัสผ่าน" if password.empty?
      errors << "รูปแบบอีเมลไม่ถูกต้อง" unless email.matches?(/.+@.+\..+/)
      errors << "Username ต้องมี 3-20 ตัวอักษร" unless username.size.in?(3..20)
      
      unless errors.empty?
        env.response.status_code = 422
        next {"errors" => errors}.to_json
      end
      
      result = auth_service.register(email, username, password, full_name)
      
      if result[:success]
        env.response.status_code = 201
        {"message" => "สมัครสมาชิกสำเร็จ", "user" => {
          "id"       => result[:user].try(&.id),
          "username" => result[:user].try(&.username),
          "email"    => result[:user].try(&.email)
        }}.to_json
      else
        env.response.status_code = 422
        {"error" => result[:error]}.to_json
      end
    rescue ex : JSON::ParseException
      env.response.status_code = 400
      {"error" => "JSON ไม่ถูกต้อง"}.to_json
    end
    
    # POST /api/auth/login
    post "/api/auth/login" do |env|
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      
      email    = data["email"]?.try(&.as_s) || ""
      password = data["password"]?.try(&.as_s) || ""
      
      result = auth_service.login(email, password)
      
      if result[:success]
        user = result[:user].not_nil!
        {"message" => "Login สำเร็จ",
         "access_token"  => result[:access_token],
         "refresh_token" => result[:refresh_token],
         "user" => {
           "id"       => user.id,
           "username" => user.username,
           "email"    => user.email,
           "role"     => user.role,
           "avatar_url" => user.avatar_url
         }}.to_json
      else
        env.response.status_code = 401
        {"error" => result[:error]}.to_json
      end
    end
    
    # POST /api/auth/refresh
    post "/api/auth/refresh" do |env|
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      
      refresh_token = data["refresh_token"]?.try(&.as_s) || ""
      
      result = auth_service.refresh(refresh_token)
      
      if result[:success]
        {"access_token" => result[:access_token]}.to_json
      else
        env.response.status_code = 401
        {"error" => result[:error]}.to_json
      end
    end
    
    # POST /api/auth/logout
    post "/api/auth/logout" do |env|
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      refresh_token = data["refresh_token"]?.try(&.as_s) || ""
      
      auth_service.logout(refresh_token)
      {"message" => "Logout สำเร็จ"}.to_json
    end
    
    # GET /api/auth/me
    get "/api/auth/me" do |env|
      user = current_user(env)
      user.to_admin_json
    end
  end
end
```

### Posts Handler

```crystal
# src/handlers/posts_handler.cr
require "kemal"
require "../repositories/post_repository"
require "../services/post_service"

module PostsHandler
  include AuthHelper
  
  def self.setup(
    post_repo : PostRepository,
    post_service : PostService,
    cache : CacheService
  )
    # GET /api/posts - List published posts
    get "/api/posts" do |env|
      page     = env.params.query["page"]?.try(&.to_i) || 1
      per_page = env.params.query["per_page"]?.try(&.to_i) || 10
      category = env.params.query["category"]?
      tag      = env.params.query["tag"]?
      author   = env.params.query["author"]?.try(&.to_i64)
      
      page = [page, 1].max
      per_page = per_page.clamp(1, 50)
      
      cache_key = "posts:list:#{page}:#{per_page}:#{category}:#{tag}:#{author}"
      
      cached = cache.get(cache_key)
      if cached
        env.response.headers["X-Cache"] = "HIT"
        next cached
      end
      
      posts = post_repo.find_published(
        category_id: category.try(&.to_i64),
        tag_slug: tag,
        author_id: author,
        limit: per_page,
        offset: (page - 1) * per_page
      )
      
      total = post_repo.count_published(
        category_id: category.try(&.to_i64),
        tag_slug: tag,
        author_id: author
      )
      
      response = {
        "posts"    => posts.map { |p| JSON.parse(p.to_list_json) },
        "total"    => total,
        "page"     => page,
        "per_page" => per_page,
        "pages"    => (total.to_f / per_page).ceil.to_i
      }.to_json
      
      cache.set(cache_key, response, 5.minutes)
      env.response.headers["X-Cache"] = "MISS"
      response
    end
    
    # GET /api/posts/:slug - Get single post
    get "/api/posts/:slug" do |env|
      slug = env.params.url["slug"]
      
      cache_key = "post:#{slug}"
      cached = cache.get(cache_key)
      if cached
        env.response.headers["X-Cache"] = "HIT"
        # Increment views async
        spawn { post_service.increment_views_by_slug(slug) }
        next cached
      end
      
      post = post_repo.find_by_slug(slug)
      
      unless post && post.visible_to_public?
        env.response.status_code = 404
        next {"error" => "ไม่พบบทความ"}.to_json
      end
      
      # Increment views
      spawn { post_service.increment_views(post.id.not_nil!) }
      
      response = post.to_detail_json
      cache.set(cache_key, response, 5.minutes)
      env.response.headers["X-Cache"] = "MISS"
      response
    end
    
    # POST /api/posts - Create post (requires auth)
    post "/api/posts" do |env|
      user = current_user(env)
      
      unless user.can_publish?
        env.response.status_code = 403
        next {"error" => "ไม่มีสิทธิ์สร้างบทความ"}.to_json
      end
      
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      
      title      = data["title"]?.try(&.as_s) || ""
      content    = data["content"]?.try(&.as_s) || ""
      excerpt    = data["excerpt"]?.try(&.as_s)
      status     = data["status"]?.try(&.as_s) || "draft"
      category_id = data["category_id"]?.try(&.as_i64)
      tag_names  = data["tags"]?.try(&.as_a.map(&.as_s)) || [] of String
      
      errors = [] of String
      errors << "กรุณาระบุหัวข้อ" if title.empty?
      errors << "กรุณาระบุเนื้อหา" if content.empty?
      errors << "สถานะไม่ถูกต้อง" unless Post::STATUSES.includes?(status)
      
      unless errors.empty?
        env.response.status_code = 422
        next {"errors" => errors}.to_json
      end
      
      result = post_service.create_post(
        user_id: user.id.not_nil!,
        title: title,
        content: content,
        excerpt: excerpt,
        status: status,
        category_id: category_id,
        tag_names: tag_names
      )
      
      if result[:success]
        env.response.status_code = 201
        cache.invalidate_pattern("posts:list:*")
        {"message" => "สร้างบทความสำเร็จ", "post_id" => result[:post_id], "slug" => result[:slug]}.to_json
      else
        env.response.status_code = 422
        {"error" => result[:error]}.to_json
      end
    end
    
    # PUT /api/posts/:id - Update post
    put "/api/posts/:id" do |env|
      user = current_user(env)
      post_id = env.params.url["id"].to_i64
      
      post = post_repo.find_by_id(post_id)
      
      unless post
        env.response.status_code = 404
        next {"error" => "ไม่พบบทความ"}.to_json
      end
      
      # ตรวจสอบสิทธิ์
      unless post.user_id == user.id || user.editor?
        env.response.status_code = 403
        next {"error" => "ไม่มีสิทธิ์แก้ไขบทความนี้"}.to_json
      end
      
      body = env.request.body.try(&.gets_to_end) || ""
      data = JSON.parse(body)
      
      # อัปเดตเฉพาะ fields ที่ส่งมา
      post.title   = data["title"]?.try(&.as_s) || post.title
      post.content = data["content"]?.try(&.as_s) || post.content
      post.excerpt = data["excerpt"]?.try(&.as_s) || post.excerpt
      post.status  = data["status"]?.try(&.as_s) || post.status
      
      if data["status"]?.try(&.as_s) == "published" && !post.published?
        post.published_at = Time.utc
      end
      
      if post_repo.update(post)
        cache.delete("post:#{post.slug}")
        cache.invalidate_pattern("posts:list:*")
        {"message" => "อัปเดตบทความสำเร็จ"}.to_json
      else
        env.response.status_code = 500
        {"error" => "อัปเดตบทความล้มเหลว"}.to_json
      end
    end
    
    # DELETE /api/posts/:id - Delete post
    delete "/api/posts/:id" do |env|
      user = current_user(env)
      post_id = env.params.url["id"].to_i64
      
      post = post_repo.find_by_id(post_id)
      
      unless post
        env.response.status_code = 404
        next {"error" => "ไม่พบบทความ"}.to_json
      end
      
      unless post.user_id == user.id || user.admin?
        env.response.status_code = 403
        next {"error" => "ไม่มีสิทธิ์ลบบทความนี้"}.to_json
      end
      
      if post_repo.delete(post_id)
        cache.delete("post:#{post.slug}")
        cache.invalidate_pattern("posts:list:*")
        {"message" => "ลบบทความสำเร็จ"}.to_json
      else
        env.response.status_code = 500
        {"error" => "ลบบทความล้มเหลว"}.to_json
      end
    end
  end
end
```

## Caching Service

### Redis Cache

```crystal
# src/services/cache_service.cr
require "redis"
require "json"

class CacheService
  def initialize(@redis : Redis::PooledClient)
  end
  
  # ดึงค่าจาก cache
  def get(key : String) : String?
    @redis.get(key)
  rescue Redis::Error
    nil  # ถ้า Redis ล้มเหลว ให้ทำงานต่อโดยไม่มี cache
  end
  
  # บันทึกค่าใน cache
  def set(key : String, value : String, ttl : Time::Span = 5.minutes)
    @redis.setex(key, ttl.total_seconds.to_i, value)
  rescue Redis::Error
    # ไม่ทำให้ app พัง ถ้า cache ล้มเหลว
    nil
  end
  
  # ลบ cache key
  def delete(key : String)
    @redis.del(key)
  rescue Redis::Error
    nil
  end
  
  # ลบ cache หลาย keys ด้วย pattern
  def invalidate_pattern(pattern : String)
    keys = @redis.keys(pattern)
    @redis.del(*keys.map(&.as(String))) unless keys.empty?
  rescue Redis::Error
    nil
  end
  
  # Cache-aside pattern
  def fetch(key : String, ttl : Time::Span = 5.minutes, &block : -> String) : String
    cached = get(key)
    return cached if cached
    
    value = yield
    set(key, value, ttl)
    value
  end
  
  # Cache statistics
  def stats : Hash(String, String)
    info = @redis.info
    {
      "connected_clients" => info.split("\n").find { |l| l.starts_with?("connected_clients") }.try(&.split(":").last.strip) || "unknown",
      "used_memory_human" => info.split("\n").find { |l| l.starts_with?("used_memory_human") }.try(&.split(":").last.strip) || "unknown",
      "keyspace_hits"     => info.split("\n").find { |l| l.starts_with?("keyspace_hits") }.try(&.split(":").last.strip) || "0",
      "keyspace_misses"   => info.split("\n").find { |l| l.starts_with?("keyspace_misses") }.try(&.split(":").last.strip) || "0"
    }
  rescue Redis::Error
    {} of String => String
  end
end
```

## Post Service

### Post Business Logic

```crystal
# src/services/post_service.cr
require "../repositories/post_repository"
require "../utils/slug_generator"

class PostService
  def initialize(
    @post_repo : PostRepository,
    @db : DB::Database
  )
  end
  
  # สร้างบทความใหม่
  def create_post(
    user_id : Int64,
    title : String,
    content : String,
    excerpt : String? = nil,
    status : String = "draft",
    category_id : Int64? = nil,
    tag_names : Array(String) = [] of String,
    featured_image_url : String? = nil
  ) : NamedTuple(success: Bool, post_id: Int64?, slug: String?, error: String?)
    # สร้าง slug
    base_slug = SlugGenerator.generate(title)
    slug = ensure_unique_slug(base_slug)
    
    # สร้าง or หา tags
    tags = tag_names.map { |name| find_or_create_tag(name) }
    
    # สร้าง excerpt อัตโนมัติถ้าไม่มี
    auto_excerpt = excerpt || generate_excerpt(content)
    
    post = Post.new(
      user_id:            user_id,
      title:              title,
      slug:               slug,
      content:            content,
      excerpt:            auto_excerpt,
      status:             status,
      category_id:        category_id,
      featured_image_url: featured_image_url
    )
    post.tags = tags
    post.published_at = Time.utc if status == "published"
    
    saved = @post_repo.create(post)
    {success: true, post_id: saved.id, slug: saved.slug, error: nil}
  rescue ex
    {success: false, post_id: nil, slug: nil, error: ex.message}
  end
  
  # เพิ่มจำนวนการดู
  def increment_views(post_id : Int64)
    @post_repo.increment_views(post_id)
  end
  
  def increment_views_by_slug(slug : String)
    post = @post_repo.find_by_slug(slug)
    increment_views(post.id.not_nil!) if post
  end
  
  # ค้นหาบทความ (Full-text search)
  def search(
    query : String,
    page : Int32 = 1,
    per_page : Int32 = 10
  ) : NamedTuple(posts: Array(Post), total: Int64)
    offset = (page - 1) * per_page
    
    posts = [] of Post
    total = 0_i64
    
    @db.query(
      "SELECT id, user_id, category_id, title, slug, excerpt, content,
              featured_image_url, status, visibility, views, likes,
              reading_time, published_at, created_at, updated_at,
              ts_rank(search_vector, query) AS rank
       FROM posts,
            plainto_tsquery('english', $1) query
       WHERE search_vector @@ query
         AND status = 'published'
         AND visibility = 'public'
         AND deleted_at IS NULL
       ORDER BY rank DESC
       LIMIT $2 OFFSET $3",
      query, per_page, offset
    ) do |rs|
      rs.each do
        post = Post.new(
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
        # rs.read(Float32)  # rank - skip
        posts << post
      end
    end
    
    total = @db.query_one(
      "SELECT COUNT(*) FROM posts,
              plainto_tsquery('english', $1) query
       WHERE search_vector @@ query
         AND status = 'published'
         AND visibility = 'public'
         AND deleted_at IS NULL",
      query,
      as: Int64
    )
    
    {posts: posts, total: total}
  end
  
  # Popular posts
  def popular_posts(limit : Int32 = 5) : Array(Post)
    posts = [] of Post
    @db.query(
      "SELECT id, user_id, category_id, title, slug, excerpt, content,
              featured_image_url, status, visibility, views, likes,
              reading_time, published_at, created_at, updated_at
       FROM posts
       WHERE status = 'published' AND visibility = 'public' AND deleted_at IS NULL
       ORDER BY views DESC
       LIMIT $1",
      limit
    ) do |rs|
      rs.each do
        posts << map_post_from_rs(rs)
      end
    end
    posts
  end
  
  private def ensure_unique_slug(base_slug : String) : String
    count = @db.query_one(
      "SELECT COUNT(*) FROM posts WHERE slug LIKE $1",
      "#{base_slug}%",
      as: Int64
    )
    count == 0 ? base_slug : "#{base_slug}-#{count + 1}"
  end
  
  private def find_or_create_tag(name : String) : Tag
    slug = SlugGenerator.generate(name)
    
    tag = @db.query_one?(
      "SELECT id, name, slug FROM tags WHERE slug = $1",
      slug
    ) do |rs|
      Tag.new(id: rs.read(Int64), name: rs.read(String), slug: rs.read(String))
    end
    
    return tag if tag
    
    result = @db.query_one(
      "INSERT INTO tags (name, slug) VALUES ($1, $2) RETURNING id",
      name, slug,
      as: Int64
    )
    
    Tag.new(id: result, name: name, slug: slug)
  end
  
  private def generate_excerpt(content : String, max_length : Int32 = 200) : String
    # ลบ markdown syntax
    plain = content
      .gsub(/#{.*?}/, "")
      .gsub(/\[.*?\]\(.*?\)/, "")
      .gsub(/[#*_`>]/, "")
      .gsub(/\n+/, " ")
      .strip
    
    return plain if plain.size <= max_length
    plain[0, max_length].rpartition(" ").first + "..."
  end
  
  private def map_post_from_rs(rs : DB::ResultSet) : Post
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

## File Upload Handler

```crystal
# src/handlers/upload_handler.cr
require "kemal"
require "file_utils"
require "digest/md5"

module UploadHandler
  include AuthHelper
  
  ALLOWED_IMAGE_TYPES = %w[image/jpeg image/png image/gif image/webp image/svg+xml]
  ALLOWED_DOC_TYPES   = %w[application/pdf application/msword text/plain]
  MAX_FILE_SIZE = 10 * 1024 * 1024  # 10MB
  UPLOAD_DIR = ENV.fetch("UPLOAD_DIR", "./uploads")
  
  def self.setup(db : DB::Database)
    # POST /api/upload/image
    post "/api/upload/image" do |env|
      user = current_user(env)
      
      files = env.params.files
      file = files["file"]?
      
      unless file
        env.response.status_code = 400
        next {"error" => "กรุณาเลือกไฟล์"}.to_json
      end
      
      # ตรวจสอบประเภทไฟล์
      content_type = file.headers["Content-Type"]? || "application/octet-stream"
      unless ALLOWED_IMAGE_TYPES.includes?(content_type)
        env.response.status_code = 422
        next {"error" => "ประเภทไฟล์ไม่ได้รับอนุญาต"}.to_json
      end
      
      # ตรวจสอบขนาดไฟล์
      file_data = file.tempfile.gets_to_end.to_slice
      if file_data.size > MAX_FILE_SIZE
        env.response.status_code = 422
        next {"error" => "ไฟล์ใหญ่เกินไป (สูงสุด 10MB)"}.to_json
      end
      
      # สร้างชื่อไฟล์ที่ไม่ซ้ำ
      ext = File.extname(file.filename || "").downcase
      filename = "#{Time.utc.to_unix}_#{Random::Secure.hex(8)}#{ext}"
      
      # สร้างโฟลเดอร์ตามวันที่
      date_dir = Time.utc.to_s("%Y/%m")
      dir_path = File.join(UPLOAD_DIR, "images", date_dir)
      FileUtils.mkdir_p(dir_path)
      
      file_path = File.join(dir_path, filename)
      
      # บันทึกไฟล์
      File.write(file_path, file_data)
      
      # บันทึกข้อมูลใน database
      attachment_id = db.query_one(
        "INSERT INTO attachments (user_id, filename, original_name, file_path, file_size, content_type)
         VALUES ($1, $2, $3, $4, $5, $6) RETURNING id",
        user.id, filename, file.filename, file_path, file_data.size, content_type,
        as: Int64
      )
      
      public_url = "/uploads/images/#{date_dir}/#{filename}"
      
      env.response.status_code = 201
      {
        "id"          => attachment_id,
        "filename"    => filename,
        "url"         => public_url,
        "content_type" => content_type,
        "size"        => file_data.size
      }.to_json
    end
    
    # DELETE /api/upload/:id
    delete "/api/upload/:id" do |env|
      user = current_user(env)
      attachment_id = env.params.url["id"].to_i64
      
      # ตรวจสอบสิทธิ์
      row = db.query_one?(
        "SELECT id, user_id, file_path FROM attachments WHERE id = $1",
        attachment_id
      ) do |rs|
        {id: rs.read(Int64), user_id: rs.read(Int64?), file_path: rs.read(String)}
      end
      
      unless row
        env.response.status_code = 404
        next {"error" => "ไม่พบไฟล์"}.to_json
      end
      
      unless row[:user_id] == user.id || user.admin?
        env.response.status_code = 403
        next {"error" => "ไม่มีสิทธิ์ลบไฟล์นี้"}.to_json
      end
      
      # ลบไฟล์จาก disk
      File.delete(row[:file_path]) if File.exists?(row[:file_path])
      
      # ลบจาก database
      db.exec("DELETE FROM attachments WHERE id = $1", attachment_id)
      
      {"message" => "ลบไฟล์สำเร็จ"}.to_json
    end
  end
end
```

## Search Handler

```crystal
# src/handlers/search_handler.cr
require "kemal"
require "../services/post_service"
require "../services/cache_service"

module SearchHandler
  def self.setup(post_service : PostService, cache : CacheService)
    # GET /api/search?q=query
    get "/api/search" do |env|
      query    = env.params.query["q"]? || ""
      page     = env.params.query["page"]?.try(&.to_i) || 1
      per_page = env.params.query["per_page"]?.try(&.to_i) || 10
      
      if query.empty? || query.size < 2
        env.response.status_code = 422
        next {"error" => "คำค้นหาต้องมีอย่างน้อย 2 ตัวอักษร"}.to_json
      end
      
      page = [page, 1].max
      per_page = per_page.clamp(1, 50)
      
      cache_key = "search:#{Digest::MD5.hexdigest(query)}:#{page}:#{per_page}"
      
      response = cache.fetch(cache_key, 2.minutes) do
        result = post_service.search(query, page, per_page)
        
        {
          "query"    => query,
          "posts"    => result[:posts].map { |p| JSON.parse(p.to_list_json) },
          "total"    => result[:total],
          "page"     => page,
          "per_page" => per_page,
          "pages"    => (result[:total].to_f / per_page).ceil.to_i
        }.to_json
      end
      
      response
    end
    
    # GET /api/search/suggestions?q=prefix
    get "/api/search/suggestions" do |env|
      prefix   = env.params.query["q"]? || ""
      limit    = env.params.query["limit"]?.try(&.to_i) || 8
      
      if prefix.size < 2
        next {"suggestions" => [] of String}.to_json
      end
      
      # ดึง titles ที่ขึ้นต้นด้วย prefix
      # จาก PostgreSQL
      env.response.content_type = "application/json"
      {"suggestions" => [] of String}.to_json  # Placeholder
    end
  end
end
```

## Application Entry Point

```crystal
# src/blogcms.cr
require "kemal"
require "pg"
require "db"
require "redis"
require "jwt"

require "./config/settings"
require "./models/*"
require "./repositories/*"
require "./services/*"
require "./middleware/*"
require "./handlers/*"
require "./utils/*"

# Configuration
DB_URL    = ENV.fetch("DATABASE_URL", "postgres://postgres:password@localhost/blogcms_dev")
REDIS_URL = ENV.fetch("REDIS_URL", "redis://localhost:6379")

# Initialize connections
db    = DB.open(DB_URL)
redis = Redis::PooledClient.new(url: REDIS_URL)

# Initialize repositories
user_repo    = UserRepository.new(db)
post_repo    = PostRepository.new(db)

# Initialize services
cache        = CacheService.new(redis)
auth_service = AuthService.new(user_repo, redis)
post_service = PostService.new(post_repo, db)

# Setup middlewares
add_handler AuthMiddleware.new(user_repo)
add_handler RateLimitMiddleware.new(redis)
add_handler RequestLoggerMiddleware.new

# CORS configuration
before_all do |env|
  env.response.headers["Access-Control-Allow-Origin"]  = ENV.fetch("ALLOWED_ORIGIN", "*")
  env.response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, PATCH, DELETE, OPTIONS"
  env.response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization, X-Requested-With"
  env.response.headers["Content-Type"] = "application/json"
end

options "/*" do |env|
  env.response.headers["Access-Control-Allow-Origin"]  = "*"
  env.response.headers["Access-Control-Max-Age"] = "86400"
  200
end

# Setup routes
AuthHandler.setup(auth_service)
PostsHandler.setup(post_repo, post_service, cache)
UploadHandler.setup(db)
SearchHandler.setup(post_service, cache)

# Serve uploaded files
public_folder "uploads"

# Health check
get "/health" do |env|
  {
    "status"    => "ok",
    "timestamp" => Time.utc.to_rfc3339,
    "version"   => "1.0.0"
  }.to_json
end

# Static pages (for SPA)
get "/" do
  send_file("public/index.html")
end

# 404 handler
error 404 do |env|
  env.response.content_type = "application/json"
  {"error" => "Not Found"}.to_json
end

# 500 handler
error 500 do |env|
  env.response.content_type = "application/json"
  {"error" => "Internal Server Error"}.to_json
end

# Shutdown hook
at_exit do
  puts "ปิด connections..."
  db.close
  redis.close
end

port = ENV.fetch("PORT", "4000").to_i
puts "BlogCMS API กำลังทำงานที่ http://localhost:#{port}"
Kemal.run(port: port)
```

## Tests

### Spec Helper

```crystal
# spec/spec_helper.cr
require "spec"
require "db"
require "pg"
require "redis"
require "../src/models/*"
require "../src/repositories/*"
require "../src/services/*"
require "../src/utils/*"

TEST_DB_URL    = ENV.fetch("TEST_DATABASE_URL", "postgres://postgres:password@localhost/blogcms_test")
TEST_REDIS_URL = ENV.fetch("TEST_REDIS_URL", "redis://localhost:6379/1")

Spec.before_each do
  # ล้างข้อมูลก่อน test แต่ละ case
  db = DB.open(TEST_DB_URL)
  db.exec("TRUNCATE TABLE comments, post_tags, attachments, posts, tags, categories, users RESTART IDENTITY CASCADE")
  db.close
end
```

### Model Tests

```crystal
# spec/models/user_spec.cr
require "../spec_helper"

describe User do
  describe "#authenticate" do
    it "คืนค่า true ถ้ารหัสผ่านถูกต้อง" do
      user = User.new(email: "test@test.com", username: "testuser")
      user.password = "password123"
      user.authenticate("password123").should be_true
    end
    
    it "คืนค่า false ถ้ารหัสผ่านผิด" do
      user = User.new(email: "test@test.com", username: "testuser")
      user.password = "password123"
      user.authenticate("wrongpassword").should be_false
    end
  end
  
  describe "#admin?" do
    it "คืนค่า true สำหรับ admin role" do
      user = User.new(email: "admin@test.com", username: "admin", role: "admin")
      user.admin?.should be_true
    end
    
    it "คืนค่า false สำหรับ role อื่น" do
      user = User.new(email: "user@test.com", username: "user", role: "author")
      user.admin?.should be_false
    end
  end
  
  describe "#can_publish?" do
    it "admin สามารถ publish ได้" do
      User.new(email: "x@x.com", username: "x", role: "admin").can_publish?.should be_true
    end
    
    it "author สามารถ publish ได้" do
      User.new(email: "x@x.com", username: "x", role: "author").can_publish?.should be_true
    end
    
    it "subscriber ไม่สามารถ publish ได้" do
      User.new(email: "x@x.com", username: "x", role: "subscriber").can_publish?.should be_false
    end
  end
end
```

### Repository Tests

```crystal
# spec/repositories/user_repository_spec.cr
require "../spec_helper"

describe UserRepository do
  db = DB.open(TEST_DB_URL)
  repo = UserRepository.new(db)
  
  describe "#create" do
    it "สร้างผู้ใช้ใหม่ได้" do
      user = User.new(email: "new@test.com", username: "newuser")
      user.password = "password123"
      
      saved = repo.create(user)
      
      saved.id.should_not be_nil
      saved.email.should eq("new@test.com")
      saved.created_at.should_not be_nil
    end
    
    it "ป้องกัน email ซ้ำ" do
      user1 = User.new(email: "dup@test.com", username: "user1")
      user1.password = "password123"
      repo.create(user1)
      
      user2 = User.new(email: "dup@test.com", username: "user2")
      user2.password = "password123"
      
      expect_raises(Exception) { repo.create(user2) }
    end
  end
  
  describe "#find_by_email" do
    it "ค้นหาผู้ใช้ด้วยอีเมลได้" do
      user = User.new(email: "find@test.com", username: "finduser")
      user.password = "password123"
      repo.create(user)
      
      found = repo.find_by_email("find@test.com")
      found.should_not be_nil
      found.not_nil!.username.should eq("finduser")
    end
    
    it "คืนค่า nil ถ้าไม่พบ" do
      repo.find_by_email("notexist@test.com").should be_nil
    end
  end
end
```

### Auth Service Tests

```crystal
# spec/services/auth_service_spec.cr
require "../spec_helper"

describe AuthService do
  db = DB.open(TEST_DB_URL)
  user_repo = UserRepository.new(db)
  service = AuthService.new(user_repo)
  
  describe "#register" do
    it "สร้างผู้ใช้ใหม่ได้" do
      result = service.register("register@test.com", "reguser", "password123")
      result[:success].should be_true
      result[:user].should_not be_nil
    end
    
    it "ไม่อนุญาตอีเมลซ้ำ" do
      service.register("dup@test.com", "user1", "password123")
      result = service.register("dup@test.com", "user2", "password123")
      result[:success].should be_false
      result[:error].should_not be_nil
    end
  end
  
  describe "#login" do
    before_each do
      service.register("login@test.com", "loginuser", "password123")
    end
    
    it "login สำเร็จด้วย credentials ที่ถูกต้อง" do
      result = service.login("login@test.com", "password123")
      result[:success].should be_true
      result[:access_token].should_not be_nil
      result[:refresh_token].should_not be_nil
    end
    
    it "login ล้มเหลวด้วยรหัสผ่านผิด" do
      result = service.login("login@test.com", "wrongpassword")
      result[:success].should be_false
    end
    
    it "login ล้มเหลวถ้าไม่พบอีเมล" do
      result = service.login("notexist@test.com", "password123")
      result[:success].should be_false
    end
  end
end
```

## สรุป Part 2

ใน Part 2 เราได้สร้าง API Layer ที่สมบูรณ์:
- **Middleware**: Authentication, Rate Limiting, Request Logging
- **Handlers**: Auth, Posts, Upload, Search
- **Caching**: Redis cache-aside pattern พร้อม invalidation
- **Post Service**: Business logic, Full-text search
- **File Upload**: Secure file handling
- **Tests**: Unit tests สำหรับ Models, Repositories, Services

## ขั้นตอนต่อไป

ใน **Part 200** เราจะ Deploy BlogCMS ไปยัง Production ด้วย Docker, Nginx, SSL/TLS, CI/CD และ Monitoring ซึ่งเป็นบทสุดท้ายของคอร์สนี้
