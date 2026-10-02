# Part 196: Clean Code ใน Crystal

## บทนำ

Clean Code คือ code ที่อ่านง่าย เข้าใจง่าย และ maintain ง่าย Robert Martin (Uncle Bob) บอกว่า "Code is read far more often than it is written" Crystal ด้วย type system ที่แข็งแกร่งช่วยให้เขียน clean code ได้ง่ายขึ้น

## การตั้งชื่อที่ดี

```crystal
# BAD: ชื่อที่ไม่สื่อความหมาย
def calc(x : Int32, y : Float64) : Float64
  x * y * 1.07
end

def process(d : Array(Hash(String, String))) : Array(Hash(String, String))
  d.select { |r| r["a"] == "1" }.map { |r| {r["n"] => r["e"]} }
end

# GOOD: ชื่อที่สื่อความหมายชัดเจน
TAX_RATE = 0.07

def calculate_total_with_tax(quantity : Int32, unit_price : Float64) : Float64
  quantity * unit_price * (1 + TAX_RATE)
end

def get_active_user_name_email_pairs(users : Array(Hash(String, String))) : Array(Hash(String, String))
  users
    .select { |user| user["status"] == "active" }
    .map { |user| {user["name"] => user["email"]} }
end

# ชื่อ variable ที่ดี
# BAD
t = Time.local
u = User.find(1)
lst = [] of String

# GOOD
current_time = Time.local
current_user = User.find(1)
error_messages = [] of String

# Boolean naming: ใช้ is_, has_, can_, should_
def is_valid? : Bool; end
def has_permission?(action : String) : Bool; end
def can_edit?(user : User) : Bool; end
```

## Functions ที่ดี

```crystal
# SOLID: Single Responsibility Principle
# BAD: function ทำหลายอย่าง
def process_user_registration(email : String, password : String, name : String, plan : String)
  # Validate
  raise "Invalid email" unless email.includes?("@")
  raise "Password too short" if password.size < 8

  # Create user
  user = User.new(email: email, password: hash_password(password), name: name)
  user.save!

  # Send email
  smtp = SMTP::Client.new
  smtp.send_welcome_email(email, name)

  # Charge credit card
  stripe = Stripe::Client.new(API_KEY)
  stripe.create_subscription(user.id, plan)

  # Log analytics
  Analytics.track("user_registered", user_id: user.id, plan: plan)

  user
end

# GOOD: แยกความรับผิดชอบ
def register_user(email : String, password : String, name : String, plan : String) : User
  validate_registration!(email, password)

  user = create_user(email, password, name)

  send_welcome_notification(user)
  setup_billing(user, plan)
  track_registration(user, plan)

  user
end

private def validate_registration!(email : String, password : String)
  UserValidator.validate!(email: email, password: password)
end

private def create_user(email : String, password : String, name : String) : User
  UserRepository.create(
    email: email,
    password_hash: PasswordHasher.hash(password),
    name: name
  )
end

private def send_welcome_notification(user : User)
  WelcomeEmailJob.perform_later(user.id)
end

private def setup_billing(user : User, plan : String)
  BillingService.create_subscription(user, plan)
end

private def track_registration(user : User, plan : String)
  Analytics.track("user_registered", {user_id: user.id, plan: plan})
end
```

## Classes ที่ดี

```crystal
# BAD: God class ที่รู้ทุกอย่าง
class Application
  def initialize
    @db = Database.new
    @cache = Cache.new
    @mailer = Mailer.new
    @users = Hash(Int64, User).new
    @orders = Hash(Int64, Order).new
    @sessions = Hash(String, User).new
  end

  def do_everything(action : String, params : Hash)
    # 500 lines of mixed concerns...
  end
end

# GOOD: classes เล็กๆ ที่มี single responsibility
class UserRepository
  def initialize(@db : Database)
  end

  def find(id : Int64) : User?
    @db.query_one?("SELECT * FROM users WHERE id = $1", id, as: User)
  end

  def create(attrs : NamedTuple) : User
    @db.exec("INSERT INTO users ...", **attrs)
    find(last_insert_id)
  end
end

class UserCache
  def initialize(@cache : Cache, @ttl : Time::Span = 5.minutes)
  end

  def get(id : Int64) : User?
    @cache.get("user:#{id}").try { |data| User.from_json(data) }
  end

  def set(user : User)
    @cache.set("user:#{user.id}", user.to_json, @ttl)
  end

  def invalidate(user_id : Int64)
    @cache.delete("user:#{user_id}")
  end
end

class UserService
  def initialize(
    @repository : UserRepository,
    @cache : UserCache,
    @notifier : UserNotifier
  )
  end

  def find(id : Int64) : User?
    @cache.get(id) || begin
      user = @repository.find(id)
      @cache.set(user) if user
      user
    end
  end

  def update(id : Int64, attrs : Hash) : User
    user = @repository.update(id, attrs)
    @cache.invalidate(id)
    @notifier.notify_updated(user)
    user
  end
end
```

## SOLID Principles

```crystal
# O: Open/Closed Principle - เปิดสำหรับ extension, ปิดสำหรับ modification
abstract class Discount
  abstract def apply(price : Float64) : Float64
  abstract def description : String
end

class PercentageDiscount < Discount
  def initialize(@percent : Float64)
  end

  def apply(price : Float64) : Float64
    price * (1 - @percent / 100)
  end

  def description : String
    "#{@percent}% off"
  end
end

class FixedDiscount < Discount
  def initialize(@amount : Float64)
  end

  def apply(price : Float64) : Float64
    [price - @amount, 0.0].max
  end

  def description : String
    "฿#{@amount} off"
  end
end

# เพิ่ม discount ใหม่โดยไม่ต้องแก้ Cart
class Cart
  def apply_discount(discount : Discount) : Float64
    total = subtotal
    discounted = discount.apply(total)
    puts "Applied: #{discount.description}"
    discounted
  end
end

# D: Dependency Inversion - depend on abstractions, not concretions
abstract class EmailProvider
  abstract def send(to : String, subject : String, body : String)
end

class SendGridProvider < EmailProvider
  def send(to : String, subject : String, body : String)
    # SendGrid API call
  end
end

class SMTPProvider < EmailProvider
  def send(to : String, subject : String, body : String)
    # SMTP send
  end
end

# ขึ้นกับ abstraction EmailProvider ไม่ใช่ concrete class
class NotificationService
  def initialize(@email_provider : EmailProvider)
  end

  def notify_order_shipped(user_email : String, order_id : String)
    @email_provider.send(
      user_email,
      "Your order #{order_id} has shipped!",
      "Track your order..."
    )
  end
end
```

## Refactoring Code Smells

```crystal
# Smell: Long Method -> Extract Method
# BAD
def generate_report(users : Array(User))
  output = ""
  total_revenue = 0.0

  output += "User Report\n"
  output += "=" * 40 + "\n"

  users.each do |user|
    output += "User: #{user.name}\n"
    output += "  Email: #{user.email}\n"
    orders = Order.where(user_id: user.id)
    user_total = orders.sum(&.total)
    total_revenue += user_total
    output += "  Orders: #{orders.size}\n"
    output += "  Total spent: #{user_total}\n"
  end

  output += "\nTotal Revenue: #{total_revenue}\n"
  output
end

# GOOD: Extracted methods
def generate_report(users : Array(User)) : String
  String.build do |sb|
    append_header(sb)
    total_revenue = 0.0
    users.each do |user|
      user_total = append_user_section(sb, user)
      total_revenue += user_total
    end
    append_summary(sb, total_revenue)
  end
end

private def append_header(io : IO)
  io << "User Report\n"
  io << "=" * 40 << "\n"
end

private def append_user_section(io : IO, user : User) : Float64
  orders = Order.where(user_id: user.id)
  user_total = orders.sum(&.total)
  io << "User: #{user.name}\n"
  io << "  Email: #{user.email}\n"
  io << "  Orders: #{orders.size}\n"
  io << "  Total: #{user_total}\n"
  user_total
end

private def append_summary(io : IO, total : Float64)
  io << "\nTotal Revenue: #{total}\n"
end
```

## แบบฝึกหัด

1. Refactor God class ใหม่ให้แยก concerns ออกเป็น focused classes
2. เพิ่ม Open/Closed principle ให้ payment processing system รองรับ providers ใหม่
3. สร้าง UserService ที่ depend on interfaces ไม่ใช่ concrete implementations
4. Identify code smells ใน codebase ของคุณและ refactor ทีละตัว

## สรุป

Clean Code ใน Crystal:
- **Naming**: ชื่อที่สื่อความหมาย, boolean prefix (is_/has_/can_)
- **Small functions**: ทำสิ่งเดียว ยาวไม่เกิน 20 บรรทัด
- **SRP**: class มีเหตุผลเดียวที่จะเปลี่ยน
- **OCP**: เปิดสำหรับ extension ผ่าน inheritance/modules
- **DIP**: depend on abstractions (abstract class/module)
- **Extract method**: แยก code block ออกเป็น named method
- **Avoid magic numbers**: ใช้ named constants
- Crystal's type system ช่วย enforce clean interfaces ผ่าน abstract classes
