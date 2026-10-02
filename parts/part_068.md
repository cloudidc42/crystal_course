# Part 68: Template Strings

## บทนำ

Template strings ใน Crystal มีหลายรูปแบบตั้งแต่ string interpolation พื้นฐาน heredoc ไปจนถึง ECR (Embedded Crystal) ซึ่งเป็น template engine ที่ built-in มาใน Crystal

---

## 1. String Interpolation เชิงลึก

```crystal
# Interpolation พื้นฐาน
name = "Crystal"
puts "Hello, #{name}!"

# Expression ที่ซับซ้อน
data = {name: "Alice", scores: [90, 85, 92]}
puts "#{data[:name]}'s average: #{data[:scores].sum / data[:scores].size}"

# Method calls
puts "Time: #{Time.local.to_s("%H:%M:%S")}"

# Conditional expression
age = 20
puts "Status: #{age >= 18 ? "adult" : "minor"}"

# Multi-line expression
result = "Total: #{
  items = [10, 20, 30, 40, 50]
  items.select { |i| i > 15 }.sum
}"
puts result  # => "Total: 140"

# Nested interpolation
outer = "world"
inner = "hello"
puts "#{outer.upcase} says: '#{inner.capitalize}!'"
# => "WORLD says: 'Hello!'"

# interpolation ใน regex
pattern = "world"
puts "hello world" =~ /#{pattern}/  # => 6

# interpolation กับ heredoc
name = "Crystal"
version = "1.10"
doc = <<-HEREDOC
  # #{name} Programming
  Version: #{version}
  Released: #{Time.local.year}
  HEREDOC

puts doc
```

---

## 2. Heredoc ขั้นสูง

```crystal
# Heredoc พื้นฐาน
text = <<-TEXT
  Line 1
  Line 2
  Line 3
  TEXT
puts text  # indentation ถูก strip ออก

# Heredoc ที่ไม่มี interpolation (ใช้ 'MARKER')
raw = <<-'RAW'
  This has #{no interpolation}
  And \n is literal
  RAW
puts raw  # แสดง literal #{no interpolation}

# Heredoc กับ method chaining
html = <<-HTML.strip.gsub("World", "Crystal")
  Hello, World!
  HTML
puts html  # => "Hello, Crystal!"

# Heredoc เป็น method argument
def process(text : String) : String
  text.upcase
end

result = process(<<-TEXT)
  hello world
  TEXT
puts result  # => "HELLO WORLD\n"

# หลาย heredoc ในบรรทัดเดียว (ไม่ค่อยแนะนำ)
a = <<-A
first
A
b = <<-B
second
B
puts "#{a.strip}, #{b.strip}"  # => "first, second"

# ใช้ heredoc สำหรับ SQL
def build_query(user_id : Int32, limit : Int32 = 10) : String
  <<-SQL
    SELECT u.id, u.name, u.email,
           COUNT(p.id) as post_count,
           MAX(p.created_at) as last_post
    FROM users u
    LEFT JOIN posts p ON p.user_id = u.id
    WHERE u.id = #{user_id}
      AND u.active = true
    GROUP BY u.id, u.name, u.email
    ORDER BY last_post DESC
    LIMIT #{limit}
    SQL
end

puts build_query(42, 5)
```

---

## 3. ECR (Embedded Crystal) Templates

```crystal
# ECR เป็น built-in template engine ของ Crystal
require "ecr"

# ECR syntax:
# <%= expression %> - output expression result
# <% code %>       - execute code (no output)
# <%- code -%>     - execute and trim whitespace
# <%# comment %>   - comment

# Template string ง่ายๆ
name = "Alice"
age = 25

template = ECR.render(%(
  Name: <%= name %>
  Age: <%= age %>
  Adult: <%= age >= 18 ? "Yes" : "No" %>
))
puts template

# ECR กับ loops
items = ["Crystal", "Ruby", "Go", "Rust"]
list_html = ECR.render(%(
  <ul>
  <% items.each do |item| %>
    <li><%= item %></li>
  <% end %>
  </ul>
))
puts list_html

# ECR กับ conditional
score = 85
grade = ECR.render(%(
  Score: <%= score %>
  Grade: <%= 
    case score
    when 90..100 then "A"
    when 80..89  then "B"
    when 70..79  then "C"
    else              "F"
    end
  %>
))
puts grade.strip
```

---

## 4. ECR File Templates

```crystal
# ใช้ ECR จาก file
require "ecr"

# สมมุติว่ามีไฟล์ templates/greeting.ecr:
# Hello, <%= @name %>!
# You have <%= @messages.size %> messages.

class GreetingView
  def initialize(@name : String, @messages : Array(String))
  end
  
  def render : String
    # ECR.render_file เมื่อมีไฟล์ template จริง
    # แต่ในตัวอย่างนี้ใช้ string template แทน
    ECR.render(%(
Hello, <%= @name %>!
You have <%= @messages.size %> message<%= @messages.size == 1 ? "" : "s" %>.
<% unless @messages.empty? %>
Your messages:
<% @messages.each_with_index do |msg, i| %>
  <%= i + 1 %>. <%= msg %>
<% end %>
<% end %>
    ))
  end
end

view = GreetingView.new("Alice", ["Hello!", "Meeting at 3pm", "Don't forget lunch"])
puts view.render

# Template เป็น class
class EmailTemplate
  def initialize(
    @to : String,
    @from : String,
    @subject : String,
    @body : String,
    @name : String
  )
  end
  
  def render : String
    ECR.render(%(
From: <%= @from %>
To: <%= @to %>
Subject: <%= @subject %>

Dear <%= @name %>,

<%= @body %>

Best regards,
The Team
    ))
  end
end

email = EmailTemplate.new(
  to: "alice@example.com",
  from: "noreply@example.com",
  subject: "Welcome!",
  body: "Thank you for joining our service.",
  name: "Alice"
)
puts email.render
```

---

## 5. Custom Template Engine

```crystal
# Simple Mustache-like template engine
class MustacheTemplate
  def initialize(@template : String)
  end
  
  def render(context : Hash(String, String | Array(Hash(String, String)) | Bool)) : String
    result = @template
    
    # Variables: {{variable}}
    result = result.gsub(/\{\{(\w+)\}\}/) do
      key = $~[1]
      case value = context[key]?
      when String then value.as(String)
      when Bool   then value.to_s
      else ""
      end
    end
    
    # Sections: {{#section}}...{{/section}}
    result = result.gsub(/\{\{#(\w+)\}\}(.*?)\{\{\/\1\}\}/m) do
      key = $~[1]
      block = $~[2]
      
      case value = context[key]?
      when Array
        items = value.as(Array(Hash(String, String)))
        items.map do |item|
          block.gsub(/\{\{(\w+)\}\}/) { |_| item[$~[1]]? || "" }
        end.join
      when Bool
        value.as(Bool) ? block : ""
      else ""
      end
    end
    
    # Inverted sections: {{^section}}...{{/section}}
    result = result.gsub(/\{\{\^(\w+)\}\}(.*?)\{\{\/\1\}\}/m) do
      key = $~[1]
      block = $~[2]
      
      case value = context[key]?
      when Bool then value.as(Bool) ? "" : block
      when Array then value.as(Array).empty? ? block : ""
      when Nil then block
      else ""
      end
    end
    
    result
  end
end

# ทดสอบ
template = MustacheTemplate.new(<<-TMPL)
  Hello, {{name}}!
  {{#items}}
  - {{title}} by {{author}}
  {{/items}}
  {{^items}}
  No items found.
  {{/items}}
  TMPL

puts template.render({
  "name" => "Alice",
  "items" => [
    {"title" => "Crystal Book", "author" => "Matz"},
    {"title" => "Ruby Way", "author" => "Fulton"},
  ] of Hash(String, String),
} of String => String | Array(Hash(String, String)) | Bool)

puts "\n---\n"

puts template.render({
  "name" => "Bob",
  "items" => [] of Hash(String, String),
} of String => String | Array(Hash(String, String)) | Bool)
```

---

## 6. Pattern สำหรับ HTML Templates

```crystal
# สร้าง HTML builder ที่ safe (auto-escaping)
module HTML
  def self.escape(str : String) : String
    str
      .gsub("&", "&amp;")
      .gsub("<", "&lt;")
      .gsub(">", "&gt;")
      .gsub("\"", "&quot;")
      .gsub("'", "&#39;")
  end
  
  class Builder
    def initialize
      @io = IO::Memory.new
      @indent = 0
    end
    
    def tag(name : String, attrs : Hash(String, String) = {} of String => String, &block)
      indent_str = "  " * @indent
      attr_str = attrs.map { |k, v| " #{k}=\"#{HTML.escape(v)}\"" }.join
      @io << "#{indent_str}<#{name}#{attr_str}>\n"
      @indent += 1
      block.call
      @indent -= 1
      @io << "#{indent_str}</#{name}>\n"
    end
    
    def text(content : String)
      @io << "  " * @indent << HTML.escape(content) << "\n"
    end
    
    def raw(content : String)
      @io << content
    end
    
    def comment(text : String)
      @io << "  " * @indent << "<!-- #{text} -->\n"
    end
    
    def to_s : String
      @io.to_s
    end
  end
end

# ใช้ HTML builder
builder = HTML::Builder.new

builder.tag("html") do
  builder.tag("head") do
    builder.tag("title") do
      builder.text("My Page")
    end
  end
  builder.tag("body") do
    builder.comment("Main content")
    builder.tag("h1", {"class" => "title"}) do
      builder.text("Hello <World>!")  # auto-escaped
    end
    builder.tag("ul") do
      ["Crystal", "Ruby", "Python"].each do |lang|
        builder.tag("li") do
          builder.text(lang)
        end
      end
    end
  end
end

puts builder.to_s
```

---

## 7. Template กับ Partial และ Layout

```crystal
# Layout system
class LayoutTemplate
  def initialize(@layout : String)
  end
  
  def render(content : String, vars : Hash(String, String) = {} of String => String) : String
    result = @layout
    vars.each { |k, v| result = result.gsub("{{#{k}}}", v) }
    result.gsub("{{yield}}", content)
  end
end

class PageTemplate
  def initialize(@content : String)
  end
  
  def render(vars : Hash(String, String) = {} of String => String) : String
    result = @content
    vars.each { |k, v| result = result.gsub("{{#{k}}}", v) }
    result
  end
end

# Layout
layout = LayoutTemplate.new(<<-LAYOUT)
  <!DOCTYPE html>
  <html>
  <head>
    <title>{{title}} - My Site</title>
  </head>
  <body>
    <nav>Home | About | Contact</nav>
    {{yield}}
    <footer>© 2024 My Site</footer>
  </body>
  </html>
  LAYOUT

# Pages
home_page = PageTemplate.new(<<-PAGE)
  <main>
    <h1>Welcome to {{site_name}}!</h1>
    <p>{{description}}</p>
  </main>
  PAGE

# Render
content = home_page.render({
  "site_name" => "Crystal Land",
  "description" => "A place for Crystal programming",
})

full_page = layout.render(content, {"title" => "Home"})
puts full_page

# Partial templates
class TemplateEngine
  def initialize
    @partials = {} of String => String
  end
  
  def register_partial(name : String, template : String)
    @partials[name] = template
  end
  
  def render(template : String, context : Hash(String, String)) : String
    result = template
    
    # Replace variables
    context.each { |k, v| result = result.gsub("{{#{k}}}", v) }
    
    # Include partials: {{> partial_name}}
    result = result.gsub(/\{\{>\s*(\w+)\s*\}\}/) do
      partial_name = $~[1]
      partial = @partials[partial_name]? || ""
      render(partial, context)
    end
    
    result
  end
end

engine = TemplateEngine.new

engine.register_partial("header", <<-PARTIAL)
  <header>
    <h1>{{site_name}}</h1>
    <nav>{{nav_links}}</nav>
  </header>
  PARTIAL

engine.register_partial("footer", <<-PARTIAL)
  <footer>© {{year}} {{site_name}}</footer>
  PARTIAL

template = <<-TMPL
  {{> header}}
  <main>
    <h2>{{page_title}}</h2>
    <p>{{content}}</p>
  </main>
  {{> footer}}
  TMPL

puts engine.render(template, {
  "site_name" => "Crystal Blog",
  "nav_links" => "Home | Posts | About",
  "page_title" => "Welcome",
  "content" => "Hello from Crystal!",
  "year" => "2024",
})
```

---

## 8. I18n Template Pattern

```crystal
# Internationalization template
class I18n
  alias Translation = Hash(String, String)
  
  def initialize(@locale : String = "en")
    @translations = {} of String => Translation
  end
  
  def add_locale(locale : String, translations : Translation)
    @translations[locale] = translations
  end
  
  def translate(key : String, **vars) : String
    text = @translations[@locale]?.try(&.[key]?) || key
    
    # Interpolate variables
    vars.each do |k, v|
      text = text.gsub("%{#{k}}", v.to_s)
    end
    
    text
  end
  
  def t(key : String, **vars) : String
    translate(key, **vars)
  end
  
  def locale=(locale : String)
    @locale = locale
  end
end

i18n = I18n.new("en")
i18n.add_locale("en", {
  "greeting" => "Hello, %{name}!",
  "farewell" => "Goodbye, %{name}!",
  "count_items" => "You have %{count} item(s).",
})
i18n.add_locale("th", {
  "greeting" => "สวัสดี, %{name}!",
  "farewell" => "ลาก่อน, %{name}!",
  "count_items" => "คุณมี %{count} รายการ",
})

puts i18n.t("greeting", name: "Alice")   # => "Hello, Alice!"
puts i18n.t("count_items", count: 5)     # => "You have 5 item(s)."

i18n.locale = "th"
puts i18n.t("greeting", name: "Alice")   # => "สวัสดี, Alice!"
puts i18n.t("count_items", count: 5)     # => "คุณมี 5 รายการ"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
สร้าง simple code generator ที่รับ class definition (name, properties) และสร้าง Crystal struct code

### แบบฝึกหัดที่ 2
สร้าง markdown renderer ที่แปลง headers, bold, italic, code เป็น HTML

### แบบฝึกหัดที่ 3
สร้าง email template engine ที่รองรับ layout, partials, และ variable interpolation

### เฉลย

```crystal
# แบบฝึกหัดที่ 1
def generate_struct(name : String, properties : Array({String, String})) : String
  String.build do |io|
    io << "struct #{name}\n"
    
    properties.each do |prop_name, prop_type|
      io << "  property #{prop_name} : #{prop_type}\n"
    end
    
    io << "\n  def initialize(\n"
    properties.each_with_index do |(prop_name, prop_type), i|
      io << "    @#{prop_name} : #{prop_type}"
      io << "," unless i == properties.size - 1
      io << "\n"
    end
    io << "  )\n"
    io << "  end\n"
    
    io << "\n  def to_json : String\n"
    io << "    JSON.build do |json|\n"
    io << "      json.object do\n"
    properties.each do |prop_name, _|
      io << "        json.field \"#{prop_name}\", @#{prop_name}\n"
    end
    io << "      end\n"
    io << "    end\n"
    io << "  end\n"
    io << "end\n"
  end
end

puts generate_struct("User", [
  {"name", "String"},
  {"age", "Int32"},
  {"email", "String"},
  {"active", "Bool"},
])

# แบบฝึกหัดที่ 2
def markdown_to_html(md : String) : String
  md
    .gsub(/^### (.+)$/, "<h3>\\1</h3>")
    .gsub(/^## (.+)$/, "<h2>\\1</h2>")
    .gsub(/^# (.+)$/, "<h1>\\1</h1>")
    .gsub(/\*\*(.+?)\*\*/, "<strong>\\1</strong>")
    .gsub(/\*(.+?)\*/, "<em>\\1</em>")
    .gsub(/`(.+?)`/, "<code>\\1</code>")
    .gsub(/\[(.+?)\]\((.+?)\)/, "<a href=\"\\2\">\\1</a>")
end

md = """
# Title
## Section
Some **bold** and *italic* text.
Check `code` and [link](https://example.com).
"""
puts markdown_to_html(md)
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **String Interpolation** ขั้นสูงกับ expressions ซับซ้อน
2. **Heredoc** รูปแบบต่างๆ และการใช้งาน
3. **ECR** (Embedded Crystal) built-in template engine
4. **Custom Template Engine** สร้าง Mustache-like templates
5. **HTML Builder** pattern สำหรับสร้าง HTML อย่างปลอดภัย
6. **Layout และ Partials** สำหรับ reusable templates
7. **I18n** pattern สำหรับ internationalization

Templates เป็นส่วนสำคัญของ web development และ code generation ใน Crystal
