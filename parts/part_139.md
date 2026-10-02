# Part 139: Lucky Pages and Actions - Pages และ Actions ใน Lucky

## บทนำ

Lucky ใช้ระบบ type-safe สำหรับทั้ง HTTP Actions และ HTML Pages แทนที่จะใช้ string-based routes และ template files

## Actions ใน Lucky

### โครงสร้าง Action

```crystal
# src/actions/home/index.cr
class Home::Index < BrowserAction
  # HTTP method และ path
  get "/" do
    render IndexPage
  end
end

# POST action
class Users::Create < BrowserAction
  post "/users" do
    # handle form submission
    SaveUser.create(params) do |operation, user|
      if user
        redirect to: Users::Show.with(user_id: user.id)
      else
        render Users::NewPage, operation: operation
      end
    end
  end
end
```

### Action Parameters

```crystal
# Path parameters
class Users::Show < BrowserAction
  get "/users/:user_id" do
    user_id  # Int64 - auto-parsed จาก path
    user = UserQuery.find(user_id)
    render Users::ShowPage, user: user
  end
end

# Nested path parameters
class Posts::Comments::Show < BrowserAction
  get "/posts/:post_id/comments/:comment_id" do
    post_id     # Int64
    comment_id  # Int64
    # ...
  end
end
```

### Query Parameters

```crystal
class Users::Index < BrowserAction
  # กำหนด allowed query params
  param search : String = ""
  param page : Int32 = 1
  param per_page : Int32 = 20
  param sort : String = "created_at"
  param order : String = "desc"
  
  get "/users" do
    # ใช้ typed params
    query = UserQuery.new
    query = query.search(search) unless search.empty?
    
    total = query.select_count
    users = query
      .limit(per_page)
      .offset((page - 1) * per_page)
      .select
    
    render Users::IndexPage,
      users: users,
      search: search,
      page: page,
      per_page: per_page,
      total: total
  end
end
```

### Action Responses

```crystal
class Products::Show < BrowserAction
  get "/products/:product_id" do
    product = ProductQuery.find(product_id)
    
    # Render HTML page
    render Products::ShowPage, product: product
  end
end

class Api::Users::Show < ApiAction
  get "/api/users/:user_id" do
    user = UserQuery.find(user_id)
    
    # JSON response
    json({
      id: user.id,
      name: user.name,
      email: user.email
    })
  end
end

class Downloads::File < BrowserAction
  get "/downloads/:filename" do
    filepath = "uploads/#{filename}"
    
    # Send file
    send_file filepath, filename: filename
  end
end
```

### Redirects

```crystal
class Sessions::Create < BrowserAction
  post "/sign_in" do
    if authenticated?
      # Redirect ไปยัง named route
      redirect to: Dashboard::Index
    else
      redirect to: Sessions::New, status: 302
    end
  end
end

# Redirect พร้อม flash message (ใช้ notice/alert keys)
redirect to: Users::Index, notice: "สร้างบัญชีสำเร็จ!"
redirect to: Home::Index, alert: "เกิดข้อผิดพลาด"
```

## Pages ใน Lucky

### Page พื้นฐาน

```crystal
# src/pages/home/index_page.cr
class Home::IndexPage < MainLayout
  def content
    h1 "หน้าแรก"
    p "ยินดีต้อนรับ"
    
    div class: "container" do
      render_hero
      render_features
    end
  end
  
  private def render_hero
    section class: "hero" do
      h2 "สร้าง App ด้วย Lucky"
      p "Type-safe, fast, and fun!"
      link "เริ่มต้นใช้งาน", to: Users::New, class: "btn-primary"
    end
  end
  
  private def render_features
    ul class: "features" do
      ["Type Safety", "Fast", "ORM"].each do |f|
        li f
      end
    end
  end
end
```

### Page ที่รับ Data

```crystal
# src/pages/users/show_page.cr
class Users::ShowPage < MainLayout
  needs user : User
  needs posts : PostQuery::SelectResult
  
  def content
    div class: "user-profile" do
      render_user_info
      render_posts
    end
  end
  
  private def render_user_info
    div class: "profile-header" do
      h1 user.name
      p user.email
      
      if user.admin?
        span "Admin", class: "badge badge-admin"
      end
      
      if current_user? == user
        link "แก้ไขโปรไฟล์", to: Users::Edit.with(user_id: user.id)
      end
    end
  end
  
  private def render_posts
    h2 "บทความของ #{user.name}"
    
    if posts.empty?
      p "ยังไม่มีบทความ"
    else
      posts.each do |post|
        render_post_card(post)
      end
    end
  end
  
  private def render_post_card(post : Post)
    article class: "post-card" do
      a href: Posts::Show.path(post_id: post.id) do
        h3 post.title
      end
      p post.excerpt
      time post.published_at.to_s, datetime: post.published_at.to_rfc3339
    end
  end
end
```

### Page สำหรับ Forms

```crystal
# src/pages/users/new_page.cr
class Users::NewPage < MainLayout
  needs operation : SaveUser
  
  def content
    h1 "สร้างบัญชีใหม่"
    
    form_for Users::Create do
      render_errors
      
      div class: "form-group" do
        label_for operation.name_param, "ชื่อ"
        text_input operation.name, class: "form-control", placeholder: "ชื่อ-นามสกุล"
        error_for operation.name
      end
      
      div class: "form-group" do
        label_for operation.email_param, "อีเมล"
        email_input operation.email, class: "form-control"
        error_for operation.email
      end
      
      div class: "form-group" do
        label_for operation.password_param, "รหัสผ่าน"
        password_input operation.password, class: "form-control"
        error_for operation.password
      end
      
      div class: "form-group" do
        label_for operation.password_confirmation_param, "ยืนยันรหัสผ่าน"
        password_input operation.password_confirmation, class: "form-control"
        error_for operation.password_confirmation
      end
      
      submit "สร้างบัญชี", class: "btn btn-primary"
    end
    
    para do
      text "มีบัญชีแล้ว? "
      link "เข้าสู่ระบบ", to: SignIns::New
    end
  end
  
  private def render_errors
    if operation.errors.any?
      div class: "alert alert-danger" do
        h5 "กรุณาแก้ไขข้อผิดพลาด:"
        ul do
          operation.errors.each do |field, messages|
            messages.each { |msg| li "#{field}: #{msg}" }
          end
        end
      end
    end
  end
end
```

## Needs System

```crystal
# ใน Action - ส่ง data ไปยัง Page
class Dashboard::Index < BrowserAction
  include RequireSignIn
  
  get "/dashboard" do
    stats = {
      users: UserQuery.new.select_count,
      posts: PostQuery.new.published.select_count,
      views: PageViewQuery.new.today.select_count,
    }
    
    recent_users = UserQuery.new.order_by_created_at(:desc).limit(5).select
    
    render Dashboard::IndexPage,
      current_user: current_user,
      stats: stats,
      recent_users: recent_users
  end
end

# ใน Page - รับ data
class Dashboard::IndexPage < MainLayout
  needs current_user : User
  needs stats : NamedTuple(users: Int64, posts: Int64, views: Int64)
  needs recent_users : UserQuery::SelectResult
  
  def content
    h1 "Dashboard"
    
    div class: "stats-grid" do
      render_stat("ผู้ใช้", stats[:users])
      render_stat("บทความ", stats[:posts])
      render_stat("ยอดชม", stats[:views])
    end
    
    render_recent_users
  end
  
  private def render_stat(label : String, value : Int64)
    div class: "stat-card" do
      p label, class: "stat-label"
      span value.to_s, class: "stat-value"
    end
  end
  
  private def render_recent_users
    h2 "ผู้ใช้ล่าสุด"
    recent_users.each do |user|
      div class: "user-row" do
        strong user.name
        text " - "
        span user.email
      end
    end
  end
end
```

## HTML Helpers

```crystal
# Lucky HTML helpers พร้อมใช้งาน
class ExamplesPage < MainLayout
  def content
    # Tags พื้นฐาน
    h1 "ชื่อเรื่อง"
    h2 "หัวข้อย่อย", class: "subtitle"
    p "ย่อหน้า"
    span "inline text"
    
    # Links
    link "ข้อความ", to: Home::Index
    link "ข้อความ", to: Users::Show.with(user_id: 1)
    a "external", href: "https://example.com", target: "_blank"
    
    # Forms
    form_for Users::Create do
      text_input operation.name
      email_input operation.email
      password_input operation.password
      number_input operation.age
      checkbox_input operation.active
      select_input operation.role, [{label: "Admin", value: "admin"}, {label: "User", value: "user"}]
      textarea_input operation.bio
      file_input operation.avatar
      submit "บันทึก"
    end
    
    # Lists
    ul do
      li "Item 1"
      li "Item 2"
    end
    
    ol do
      ["a", "b", "c"].each { |i| li i }
    end
    
    # Table
    table do
      thead do
        tr do
          th "ชื่อ"
          th "อีเมล"
        end
      end
      tbody do
        tr do
          td "สมชาย"
          td "somchai@example.com"
        end
      end
    end
    
    # Conditionals
    if current_user?
      span "Welcome #{current_user.name}!"
    else
      link "เข้าสู่ระบบ", to: SignIns::New
    end
    
    # Raw HTML (ระวัง XSS!)
    raw "<strong>Raw HTML</strong>"
    
    # Text (auto-escaped)
    text user_input  # safe
  end
end
```

## JSON Responses

```crystal
# src/actions/api/v1/users/index.cr
class Api::V1::Users::Index < ApiAction
  # Query params
  param search : String = ""
  param page : Int32 = 1
  param per_page : Int32 = 20
  
  get "/api/v1/users" do
    query = UserQuery.new.active
    query = query.search(search) unless search.empty?
    
    total = query.select_count
    users = query.limit(per_page).offset((page - 1) * per_page).select
    
    json({
      data: users.map { |u| serialize(u) },
      meta: pagination_meta(total)
    })
  end
  
  private def serialize(user : User)
    {
      id: user.id,
      email: user.email,
      name: user.name,
      role: user.role,
      created_at: user.created_at.to_rfc3339,
    }
  end
  
  private def pagination_meta(total : Int64)
    {
      page: page,
      per_page: per_page,
      total: total,
      pages: (total.to_f / per_page).ceil.to_i,
    }
  end
end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Product Catalog

```crystal
# Action
class Products::Index < BrowserAction
  param search : String = ""
  param category : String = ""
  param min_price : Float64 = 0.0
  param max_price : Float64 = 99999.0
  param sort : String = "name"
  param page : Int32 = 1
  
  get "/products" do
    query = ProductQuery.new.active
    query = query.search(search) unless search.empty?
    query = query.by_category(category) unless category.empty?
    query = query.price_range(min_price, max_price)
    
    total = query.select_count
    products = query.sorted_by(sort).paginate(page, 20).select
    categories = CategoryQuery.new.all.select
    
    render Products::IndexPage,
      products: products,
      categories: categories,
      total: total,
      page: page,
      search: search,
      category: category
  end
end

# Page
class Products::IndexPage < MainLayout
  needs products : ProductQuery::SelectResult
  needs categories : CategoryQuery::SelectResult
  needs total : Int64
  needs page : Int32
  needs search : String
  needs category : String
  
  def content
    div class: "product-catalog" do
      render_filters
      render_products
      render_pagination
    end
  end
  
  private def render_filters
    form_for Products::Index do
      text_input search_param, value: search, placeholder: "ค้นหาสินค้า..."
      select_input category_param, value: category, options: category_options
      submit "กรอง"
    end
  end
  
  private def category_options
    [["ทั้งหมด", ""]] + categories.map { |c| [c.name, c.slug] }
  end
  
  private def render_products
    if products.empty?
      p "ไม่พบสินค้าที่ตรงกับการค้นหา"
    else
      div class: "products-grid" do
        products.each { |p| render_product_card(p) }
      end
    end
  end
  
  private def render_product_card(product : Product)
    div class: "product-card" do
      img src: product.image_url, alt: product.name
      h3 product.name
      p "฿#{product.price}", class: "price"
      link "ดูรายละเอียด", to: Products::Show.with(product_id: product.id)
    end
  end
  
  private def render_pagination
    # Pagination links
    div class: "pagination" do
      if page > 1
        link "« ก่อนหน้า", to: Products::Index.with(page: page - 1)
      end
      text " หน้า #{page} "
      link "ถัดไป »", to: Products::Index.with(page: page + 1)
    end
  end
end
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **Actions**: HTTP handlers พร้อม type-safe params
2. **Path Parameters**: auto-parsed เป็น typed values
3. **Query Parameters**: `param` macro ใน Action
4. **Responses**: render, json, redirect, send_file
5. **Pages**: type-safe HTML generation
6. **needs**: ส่งและรับ data ระหว่าง Action และ Page
7. **HTML Helpers**: Lucky's DSL สำหรับ HTML
8. **Forms**: type-safe form helpers
9. **JSON API**: ApiAction และ json response

Lucky บังคับ type safety ทุกขั้นตอนตั้งแต่ URL params จนถึง HTML output
