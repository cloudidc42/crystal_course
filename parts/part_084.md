# ตอนที่ 84: XML ใน Crystal

## บทนำ

XML (eXtensible Markup Language) เป็นรูปแบบข้อมูลที่ใช้กันอย่างแพร่หลายใน web services, configuration files และการแลกเปลี่ยนข้อมูลระหว่างระบบ Crystal มี built-in support สำหรับ XML ผ่าน `require "xml"`

---

## 1. การ require "xml"

```crystal
require "xml"
```

---

## 2. XML.parse - การแปลง XML String เป็น Document

`XML.parse` แปลง XML string เป็น `XML::Node` object ที่แทน document ทั้งหมด:

```crystal
require "xml"

xml_string = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
  <person>
    <name>สมชาย</name>
    <age>30</age>
    <city>กรุงเทพ</city>
  </person>
  XML

doc = XML.parse(xml_string)
puts doc.class  # => XML::Node
puts doc        # แสดง XML ทั้งหมด
```

### การ parse จาก IO:

```crystal
require "xml"

# จาก String IO
xml_io = IO::Memory.new(<<-XML)
  <root>
    <item>value1</item>
    <item>value2</item>
  </root>
  XML

doc = XML.parse(xml_io)
puts doc.class
```

---

## 3. XML::Node - Node Types

XML ประกอบด้วย node หลายประเภท:

```crystal
require "xml"

xml = <<-XML
  <?xml version="1.0"?>
  <!-- ความคิดเห็น -->
  <root id="1" class="main">
    <child>ข้อความ</child>
  </root>
  XML

doc = XML.parse(xml)

# วิธีเข้าถึง root node
root = doc.root
puts root          # => <root id="1" class="main">...</root>
puts root.name     # => root
puts root.class    # => XML::Node
```

### ประเภทของ Node:

| Node Type          | ค่า | ความหมาย                |
|--------------------|-----|------------------------|
| XML::Node::Type::DOCUMENT_NODE    | 9   | Document node (root)   |
| XML::Node::Type::ELEMENT_NODE     | 1   | Element `<tag>`        |
| XML::Node::Type::TEXT_NODE        | 3   | Text content           |
| XML::Node::Type::ATTRIBUTE_NODE   | 2   | Attribute `attr="val"` |
| XML::Node::Type::COMMENT_NODE     | 8   | `<!-- comment -->`     |
| XML::Node::Type::CDATA_SECTION_NODE | 4 | `<![CDATA[...]]>`      |

```crystal
require "xml"

xml = <<-XML
  <root>
    <!-- ความคิดเห็น -->
    <child id="1">ข้อความ</child>
  </root>
  XML

doc = XML.parse(xml)
root = doc.root

puts "Document type: #{doc.type}"  # => DOCUMENT_NODE
puts "Root type: #{root.type}"     # => ELEMENT_NODE
puts "Root name: #{root.name}"

root.children.each do |child|
  puts "Child type: #{child.type}, name: #{child.name}"
end
```

---

## 4. Element Nodes

```crystal
require "xml"

xml = <<-XML
  <catalog>
    <book id="1" category="fiction">
      <title>Crystal Adventures</title>
      <author>John Doe</author>
      <price>350.00</price>
      <year>2024</year>
    </book>
    <book id="2" category="programming">
      <title>Learn Crystal</title>
      <author>Jane Smith</author>
      <price>450.00</price>
      <year>2023</year>
    </book>
  </catalog>
  XML

doc = XML.parse(xml)
catalog = doc.root

puts "Root element: #{catalog.name}"

# วนซ้ำผ่าน child elements
catalog.children.each do |node|
  # ข้ามข้อความว่าง (whitespace text nodes)
  next unless node.element?

  puts "\nBook:"
  node.children.each do |child|
    next unless child.element?
    puts "  #{child.name}: #{child.inner_text}"
  end
end
```

---

## 5. Text Nodes

```crystal
require "xml"

xml = <<-XML
  <message>
    <greeting>สวัสดี</greeting>
    <body>นี่คือ<em>ข้อความ</em>สำคัญ</body>
  </message>
  XML

doc = XML.parse(xml)
root = doc.root

# เข้าถึง text ใน element
greeting = root.children.find { |n| n.name == "greeting" }
if greeting
  puts "Greeting element text: #{greeting.inner_text}"
  # inner_text รวม text ทั้งหมดใน subtree
end

body = root.children.find { |n| n.name == "body" }
if body
  puts "Body inner_text: #{body.inner_text}"

  # วนซ้ำ child nodes ทั้งหมด (รวม text nodes)
  body.children.each do |child|
    if child.text?
      puts "Text node: '#{child.content}'"
    elsif child.element?
      puts "Element node: <#{child.name}>#{child.inner_text}</#{child.name}>"
    end
  end
end
```

---

## 6. Attribute Nodes

```crystal
require "xml"

xml = <<-XML
  <product
    id="P001"
    name="แล็ปท็อป"
    price="35000"
    category="electronics"
    in-stock="true"
  />
  XML

doc = XML.parse(xml)
product = doc.root

# เข้าถึง attribute ด้วย []
id       = product["id"]
name     = product["name"]
price    = product["price"]
category = product["category"]
in_stock = product["in-stock"]

puts "ID: #{id}"
puts "Name: #{name}"
puts "Price: #{price}"
puts "Category: #{category}"
puts "In Stock: #{in_stock}"
```

### ใช้ []? สำหรับ optional attributes:

```crystal
require "xml"

xml = <<-XML
  <items>
    <item id="1" name="สินค้า A" discount="10"/>
    <item id="2" name="สินค้า B"/>
    <item id="3" name="สินค้า C" discount="20"/>
  </items>
  XML

doc = XML.parse(xml)
doc.root.children.each do |node|
  next unless node.element?

  id       = node["id"]
  name     = node["name"]
  discount = node["discount"]?  # อาจไม่มี

  if discount
    puts "#{name}: ลด #{discount}%"
  else
    puts "#{name}: ไม่มีส่วนลด"
  end
end
```

---

## 7. การ Traverse - children, parent, next_sibling

```crystal
require "xml"

xml = <<-XML
  <family>
    <person id="1">
      <name>พ่อ</name>
      <role>parent</role>
    </person>
    <person id="2">
      <name>แม่</name>
      <role>parent</role>
    </person>
    <person id="3">
      <name>ลูกคนแรก</name>
      <role>child</role>
    </person>
    <person id="4">
      <name>ลูกคนที่สอง</name>
      <role>child</role>
    </person>
  </family>
  XML

doc = XML.parse(xml)
root = doc.root

puts "=== Children ==="
root.children.each do |node|
  next unless node.element?
  puts "  #{node["id"]}: #{node.xpath_node("name/text()").try(&.content)}"
end

# first_element_child
first = root.first_element_child
puts "\nFirst child: #{first.try { |n| n["id"] }}"

# next_sibling
if first
  second = first.next_sibling
  # ข้ามข้อความว่าง
  while second && !second.element?
    second = second.next_sibling
  end
  puts "Second child: #{second.try { |n| n["id"] }}"
end
```

### การหา parent:

```crystal
require "xml"

xml = <<-XML
  <document>
    <section id="s1">
      <paragraph id="p1">เนื้อหา</paragraph>
    </section>
  </document>
  XML

doc = XML.parse(xml)

# หา paragraph
paragraph = doc.xpath_node("//paragraph")
if paragraph
  puts "Paragraph: #{paragraph["id"]}"

  # หา parent
  parent = paragraph.parent
  puts "Parent: #{parent.try(&.name)} (#{parent.try { |p| p["id"] }})"

  # หา grandparent
  grandparent = parent.try(&.parent)
  puts "Grandparent: #{grandparent.try(&.name)}"
end
```

---

## 8. inner_text - การดึง Text Content

```crystal
require "xml"

xml = <<-XML
  <article>
    <title>Crystal Programming Language</title>
    <content>
      Crystal เป็นภาษาโปรแกรมที่
      <strong>เร็ว</strong> และ
      <em>ปลอดภัย</em>
    </content>
    <tags>
      <tag>programming</tag>
      <tag>crystal</tag>
      <tag>systems</tag>
    </tags>
  </article>
  XML

doc = XML.parse(xml)
root = doc.root

# inner_text รวม text ทั้งหมดใน element และ children
title   = root.xpath_node("title")
content = root.xpath_node("content")

puts "Title: #{title.try(&.inner_text.strip)}"
puts "Content: #{content.try(&.inner_text.strip.gsub(/\s+/, " "))}"

# เก็บ tags ทั้งหมด
tags = root.xpath_nodes("tags/tag").map(&.inner_text)
puts "Tags: #{tags.join(", ")}"
```

---

## 9. XPath - การค้นหา Nodes

Crystal's XML รองรับ XPath expressions สำหรับค้นหา nodes:

```crystal
require "xml"

xml = <<-XML
  <?xml version="1.0"?>
  <bookstore>
    <book category="fiction">
      <title>The Crystal Tower</title>
      <author>Alice Johnson</author>
      <price>350</price>
    </book>
    <book category="programming">
      <title>Crystal in Action</title>
      <author>Bob Smith</author>
      <price>450</price>
    </book>
    <book category="fiction">
      <title>Midnight Crystal</title>
      <author>Carol White</author>
      <price>280</price>
    </book>
  </bookstore>
  XML

doc = XML.parse(xml)

# หา node แรกที่ match
first_book = doc.xpath_node("//book")
puts "First book: #{first_book.try { |b| b.xpath_node("title").try(&.inner_text) }}"

# หา nodes ทั้งหมดที่ match
all_titles = doc.xpath_nodes("//title")
puts "\nAll titles:"
all_titles.each { |t| puts "  - #{t.inner_text}" }

# หา node ตาม attribute
fiction_books = doc.xpath_nodes("//book[@category='fiction']")
puts "\nFiction books:"
fiction_books.each do |book|
  title = book.xpath_node("title").try(&.inner_text)
  puts "  - #{title}"
end

# หาหนังสือที่ราคาสูงกว่า 300
expensive = doc.xpath_nodes("//book[price>300]")
puts "\nBooks over 300 baht:"
expensive.each do |book|
  title = book.xpath_node("title").try(&.inner_text)
  price = book.xpath_node("price").try(&.inner_text)
  puts "  #{title}: #{price} บาท"
end
```

### XPath Functions:

```crystal
require "xml"

xml = <<-XML
  <data>
    <items>
      <item>a</item>
      <item>b</item>
      <item>c</item>
    </items>
    <name>Hello World</name>
  </data>
  XML

doc = XML.parse(xml)

# นับ nodes
count = doc.xpath_float("count(//item)")
puts "Item count: #{count}"  # => 3.0

# string value
name_str = doc.xpath_string("//name")
puts "Name: #{name_str}"  # => Hello World

# boolean
has_items = doc.xpath_bool("count(//item) > 0")
puts "Has items: #{has_items}"  # => true
```

---

## 10. XML::Builder - การสร้าง XML

`XML::Builder` ใช้สร้าง XML string แบบ programmatic:

```crystal
require "xml"

xml_string = XML.build do |xml|
  xml.element("root") do
    xml.element("person", {"id" => "1"}) do
      xml.element("name") { xml.text "สมชาย" }
      xml.element("age")  { xml.text "30" }
      xml.element("city") { xml.text "กรุงเทพ" }
    end
    xml.element("person", {"id" => "2"}) do
      xml.element("name") { xml.text "มานี" }
      xml.element("age")  { xml.text "25" }
      xml.element("city") { xml.text "เชียงใหม่" }
    end
  end
end

puts xml_string
```

### XML.build พร้อม declaration:

```crystal
require "xml"

xml = XML.build(indent: "  ") do |xml|
  xml.element("catalog") do
    products = [
      {id: 1, name: "กาแฟ", price: 250, category: "beverages"},
      {id: 2, name: "ชา", price: 180, category: "beverages"},
      {id: 3, name: "ขนมปัง", price: 45, category: "bakery"},
    ]

    products.each do |p|
      xml.element("product", {
        "id"       => p[:id].to_s,
        "category" => p[:category],
      }) do
        xml.element("name")  { xml.text p[:name] }
        xml.element("price") { xml.text p[:price].to_s }
      end
    end
  end
end

puts xml
```

---

## 11. XML Builder ขั้นสูง

```crystal
require "xml"

xml = XML.build(indent: "  ") do |xml|
  xml.element("rss", {
    "version" => "2.0",
    "xmlns:content" => "http://purl.org/rss/1.0/modules/content/",
  }) do
    xml.element("channel") do
      xml.element("title") { xml.text "Crystal News" }
      xml.element("link")  { xml.text "https://crystal-lang.org" }
      xml.element("description") { xml.text "Latest Crystal news" }

      # Comment
      xml.comment "News items below"

      items = [
        {title: "Crystal 1.11 Released", date: "2024-01-15"},
        {title: "New stdlib additions", date: "2024-01-20"},
      ]

      items.each do |item|
        xml.element("item") do
          xml.element("title")   { xml.text item[:title] }
          xml.element("pubDate") { xml.text item[:date] }
        end
      end
    end
  end
end

puts xml
```

### CDATA section:

```crystal
require "xml"

xml = XML.build(indent: "  ") do |xml|
  xml.element("root") do
    xml.element("script") do
      xml.cdata "if (x < 5 && y > 3) { console.log('hello'); }"
    end
    xml.element("content") do
      xml.cdata "<p>HTML content with <strong>bold</strong></p>"
    end
  end
end

puts xml
```

---

## 12. Namespaces

XML namespaces ใช้แก้ปัญหา name conflicts:

```crystal
require "xml"

xml_with_ns = <<-XML
  <?xml version="1.0"?>
  <root
    xmlns="http://default.example.com"
    xmlns:dc="http://purl.org/dc/elements/1.1/"
    xmlns:atom="http://www.w3.org/2005/Atom">

    <title>Document Title</title>
    <dc:creator>John Doe</dc:creator>
    <dc:date>2024-01-15</dc:date>
    <atom:link href="https://example.com" rel="alternate"/>
  </root>
  XML

doc = XML.parse(xml_with_ns)
root = doc.root

puts "Root name: #{root.name}"

root.children.each do |node|
  next unless node.element?
  ns_prefix = node.namespace.try { |ns| ns.prefix ? "#{ns.prefix}:" : "" } || ""
  puts "Element: #{ns_prefix}#{node.name} = #{node.inner_text}"
end
```

### สร้าง XML พร้อม namespaces:

```crystal
require "xml"

xml = XML.build(indent: "  ") do |xml|
  xml.element("soap:Envelope", {
    "xmlns:soap" => "http://schemas.xmlsoap.org/soap/envelope/",
    "xmlns:tns"  => "http://example.com/service",
  }) do
    xml.element("soap:Body") do
      xml.element("tns:GetUser") do
        xml.element("tns:UserId") { xml.text "12345" }
      end
    end
  end
end

puts xml
```

---

## 13. การ Parse XML จาก Web Services

```crystal
require "xml"

# จำลอง SOAP response
soap_response = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
  <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
    <soap:Body>
      <GetWeatherResponse xmlns="http://weather.example.com/">
        <Location>กรุงเทพมหานคร</Location>
        <Temperature unit="celsius">32</Temperature>
        <Humidity unit="percent">75</Humidity>
        <Condition>มีเมฆบางส่วน</Condition>
        <Forecast>
          <Day date="2024-01-16" high="34" low="26" condition="แดดจัด"/>
          <Day date="2024-01-17" high="31" low="25" condition="มีฝน"/>
          <Day date="2024-01-18" high="29" low="24" condition="มีเมฆ"/>
        </Forecast>
      </GetWeatherResponse>
    </soap:Body>
  </soap:Envelope>
  XML

doc = XML.parse(soap_response)

# ค้นหาด้วย XPath (namespace-aware)
location = doc.xpath_node("//*[local-name()='Location']")
temp     = doc.xpath_node("//*[local-name()='Temperature']")
humidity = doc.xpath_node("//*[local-name()='Humidity']")
condition = doc.xpath_node("//*[local-name()='Condition']")

puts "สภาพอากาศ: #{location.try(&.inner_text)}"
puts "อุณหภูมิ: #{temp.try(&.inner_text)}°C"
puts "ความชื้น: #{humidity.try(&.inner_text)}%"
puts "สภาพ: #{condition.try(&.inner_text)}"

puts "\nพยากรณ์ 3 วัน:"
doc.xpath_nodes("//*[local-name()='Day']").each do |day|
  date      = day["date"]
  high      = day["high"]
  low       = day["low"]
  day_cond  = day["condition"]
  puts "  #{date}: #{low}°C - #{high}°C, #{day_cond}"
end
```

---

## 14. การแปลง XML เป็น Crystal Structs

```crystal
require "xml"

struct Employee
  property id : String
  property name : String
  property department : String
  property salary : Float64
  property skills : Array(String)

  def initialize(@id, @name, @department, @salary, @skills)
  end
end

def parse_employee(node : XML::Node) : Employee
  id         = node["id"]? || ""
  name       = node.xpath_node("name").try(&.inner_text) || ""
  department = node.xpath_node("department").try(&.inner_text) || ""
  salary     = node.xpath_node("salary").try(&.inner_text.to_f) || 0.0
  skills     = node.xpath_nodes("skills/skill").map(&.inner_text)

  Employee.new(id, name, department, salary, skills)
end

xml = <<-XML
  <employees>
    <employee id="E001">
      <name>สมชาย ใจดี</name>
      <department>Engineering</department>
      <salary>85000</salary>
      <skills>
        <skill>Crystal</skill>
        <skill>Ruby</skill>
        <skill>PostgreSQL</skill>
      </skills>
    </employee>
    <employee id="E002">
      <name>มานี สวยงาม</name>
      <department>Marketing</department>
      <salary>62000</salary>
      <skills>
        <skill>Marketing</skill>
        <skill>Analytics</skill>
      </skills>
    </employee>
  </employees>
  XML

doc = XML.parse(xml)
employees = doc.xpath_nodes("//employee").map { |node| parse_employee(node) }

employees.each do |emp|
  puts "#{emp.id}: #{emp.name} (#{emp.department})"
  puts "  เงินเดือน: #{emp.salary.format(decimal_places: 0)} บาท"
  puts "  ทักษะ: #{emp.skills.join(", ")}"
end
```

---

## 15. XML Validation และ Error Handling

```crystal
require "xml"

def safe_parse_xml(xml_string : String) : XML::Node?
  XML.parse(xml_string)
rescue XML::Error => e
  puts "XML parse error: #{e.message}"
  nil
end

# XML ที่ถูกต้อง
valid_xml = "<root><child>value</child></root>"
doc = safe_parse_xml(valid_xml)
puts "Valid XML parsed: #{!doc.nil?}"

# XML ที่ไม่ถูกต้อง
invalid_xml = "<root><child>unclosed"
doc2 = safe_parse_xml(invalid_xml)
puts "Invalid XML parsed: #{!doc2.nil?}"
```

### ตรวจสอบโครงสร้าง XML:

```crystal
require "xml"

def validate_user_xml(xml_string : String) : Array(String)
  errors = [] of String

  doc = XML.parse(xml_string)
  root = doc.root

  unless root.name == "user"
    errors << "Root element ต้องเป็น <user>"
    return errors
  end

  unless root.xpath_node("name")
    errors << "ต้องมี element <name>"
  end

  unless root.xpath_node("email")
    errors << "ต้องมี element <email>"
  end

  age_node = root.xpath_node("age")
  if age_node
    age = age_node.inner_text.to_i?
    errors << "อายุต้องเป็นตัวเลข" unless age
    errors << "อายุต้องมากกว่า 0" if age && age <= 0
  else
    errors << "ต้องมี element <age>"
  end

  errors
end

good_xml = <<-XML
  <user>
    <name>สมชาย</name>
    <email>somchai@example.com</email>
    <age>30</age>
  </user>
  XML

bad_xml = <<-XML
  <user>
    <name>มานี</name>
    <age>invalid</age>
  </user>
  XML

puts "Good XML:"
errs = validate_user_xml(good_xml)
puts errs.empty? ? "  ถูกต้อง" : errs.map { |e| "  - #{e}" }.join("\n")

puts "\nBad XML:"
errs = validate_user_xml(bad_xml)
puts errs.empty? ? "  ถูกต้อง" : errs.map { |e| "  - #{e}" }.join("\n")
```

---

## 16. RSS Feed Parser

```crystal
require "xml"

rss_xml = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
  <rss version="2.0">
    <channel>
      <title>Crystal Programming Blog</title>
      <link>https://crystal-blog.example.com</link>
      <description>Latest Crystal programming news and tutorials</description>
      <item>
        <title>Crystal 1.11: What's New</title>
        <link>https://crystal-blog.example.com/crystal-1-11</link>
        <description>Overview of Crystal 1.11 features</description>
        <pubDate>Mon, 15 Jan 2024 10:00:00 +0700</pubDate>
        <author>editor@crystal-blog.example.com</author>
      </item>
      <item>
        <title>Getting Started with Crystal Shards</title>
        <link>https://crystal-blog.example.com/shards-tutorial</link>
        <description>How to use Crystal's package manager</description>
        <pubDate>Fri, 12 Jan 2024 09:00:00 +0700</pubDate>
        <author>editor@crystal-blog.example.com</author>
      </item>
    </channel>
  </rss>
  XML

struct RSSItem
  property title : String
  property link : String
  property description : String
  property pub_date : String

  def initialize(@title, @link, @description, @pub_date)
  end
end

doc = XML.parse(rss_xml)

channel_title = doc.xpath_string("//channel/title")
puts "Feed: #{channel_title}"
puts "=" * 40

items = doc.xpath_nodes("//item").map do |item|
  title       = item.xpath_string("title")
  link        = item.xpath_string("link")
  description = item.xpath_string("description")
  pub_date    = item.xpath_string("pubDate")
  RSSItem.new(title, link, description, pub_date)
end

items.each do |item|
  puts "\n#{item.title}"
  puts "  #{item.pub_date}"
  puts "  #{item.link}"
  puts "  #{item.description}"
end
```

---

## 17. SVG Generation

XML Builder สามารถสร้าง SVG ได้:

```crystal
require "xml"

def generate_bar_chart(data : Array(Tuple(String, Int32))) : String
  width    = 400
  height   = 300
  margin   = 50
  bar_width = (width - 2 * margin) / data.size

  max_value = data.max_of { |_, v| v }

  XML.build(indent: "  ") do |xml|
    xml.element("svg", {
      "xmlns"   => "http://www.w3.org/2000/svg",
      "width"   => width.to_s,
      "height"  => height.to_s,
      "viewBox" => "0 0 #{width} #{height}",
    }) do
      # Background
      xml.element("rect", {"width" => "100%", "height" => "100%", "fill" => "#f5f5f5"})

      # Bars
      data.each_with_index do |(label, value), i|
        bar_height = (value.to_f / max_value * (height - 2 * margin)).to_i
        x = margin + i * bar_width + bar_width / 4
        y = height - margin - bar_height
        w = bar_width / 2

        # Bar rectangle
        xml.element("rect", {
          "x"      => x.to_s,
          "y"      => y.to_s,
          "width"  => w.to_s,
          "height" => bar_height.to_s,
          "fill"   => "#4A90D9",
          "rx"     => "3",
        })

        # Value label
        xml.element("text", {
          "x"           => (x + w / 2).to_s,
          "y"           => (y - 5).to_s,
          "text-anchor" => "middle",
          "font-size"   => "12",
          "fill"        => "#333",
        }) { xml.text value.to_s }

        # X label
        xml.element("text", {
          "x"           => (x + w / 2).to_s,
          "y"           => (height - margin + 15).to_s,
          "text-anchor" => "middle",
          "font-size"   => "11",
          "fill"        => "#666",
        }) { xml.text label }
      end
    end
  end
end

data = [
  {"มกราคม", 120},
  {"กุมภาพันธ์", 155},
  {"มีนาคม", 98},
  {"เมษายน", 175},
]

svg = generate_bar_chart(data)
puts svg
# File.write("chart.svg", svg)
```

---

## 18. ตัวอย่างโปรแกรมสมบูรณ์ - XML Config System

```crystal
require "xml"

struct DatabasePool
  property min_connections : Int32
  property max_connections : Int32
  property timeout : Int32

  def initialize(@min_connections, @max_connections, @timeout)
  end
end

struct DatabaseConfig
  property host : String
  property port : Int32
  property name : String
  property username : String
  property pool : DatabasePool

  def initialize(@host, @port, @name, @username, @pool)
  end
end

struct CacheConfig
  property enabled : Bool
  property host : String
  property port : Int32
  property ttl : Int32

  def initialize(@enabled, @host, @port, @ttl)
  end
end

struct AppConfig
  property name : String
  property version : String
  property environment : String
  property debug : Bool
  property database : DatabaseConfig
  property cache : CacheConfig

  def initialize(@name, @version, @environment, @debug, @database, @cache)
  end
end

config_xml = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
  <configuration>
    <application>
      <name>CrystalApp</name>
      <version>2.0.0</version>
      <environment>production</environment>
      <debug>false</debug>
    </application>
    <database>
      <host>db.crystalapp.com</host>
      <port>5432</port>
      <name>crystalapp_prod</name>
      <username>app_user</username>
      <pool>
        <minConnections>5</minConnections>
        <maxConnections>50</maxConnections>
        <timeout>30</timeout>
      </pool>
    </database>
    <cache>
      <enabled>true</enabled>
      <host>cache.crystalapp.com</host>
      <port>6379</port>
      <ttl>3600</ttl>
    </cache>
  </configuration>
  XML

def get_text(node : XML::Node, path : String) : String
  node.xpath_node(path).try(&.inner_text) || ""
end

def get_int(node : XML::Node, path : String) : Int32
  get_text(node, path).to_i
end

def get_bool(node : XML::Node, path : String) : Bool
  get_text(node, path) == "true"
end

doc = XML.parse(config_xml)
root = doc.root

app = root.xpath_node("application").not_nil!
db  = root.xpath_node("database").not_nil!
pool = db.xpath_node("pool").not_nil!
cache = root.xpath_node("cache").not_nil!

config = AppConfig.new(
  name:    get_text(app, "name"),
  version: get_text(app, "version"),
  environment: get_text(app, "environment"),
  debug:   get_bool(app, "debug"),
  database: DatabaseConfig.new(
    host:     get_text(db, "host"),
    port:     get_int(db, "port"),
    name:     get_text(db, "name"),
    username: get_text(db, "username"),
    pool: DatabasePool.new(
      min_connections: get_int(pool, "minConnections"),
      max_connections: get_int(pool, "maxConnections"),
      timeout:         get_int(pool, "timeout")
    )
  ),
  cache: CacheConfig.new(
    enabled: get_bool(cache, "enabled"),
    host:    get_text(cache, "host"),
    port:    get_int(cache, "port"),
    ttl:     get_int(cache, "ttl")
  )
)

puts "Application: #{config.name} v#{config.version}"
puts "Environment: #{config.environment}"
puts "Debug: #{config.debug}"
puts ""
puts "Database: #{config.database.host}:#{config.database.port}/#{config.database.name}"
puts "  Pool: #{config.database.pool.min_connections}-#{config.database.pool.max_connections} connections"
puts ""
puts "Cache: #{config.cache.host}:#{config.cache.port} (TTL: #{config.cache.ttl}s)"
puts "  Enabled: #{config.cache.enabled}"
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: XML Book Catalog
สร้างโปรแกรมที่:
1. Parse XML catalog ที่มีหนังสือหลายรายการ
2. Filter หนังสือตาม category
3. Sort ตาม price
4. Generate XML output ใหม่พร้อม summary

### แบบฝึกหัดที่ 2: XML Builder
สร้าง function `build_invoice` ที่รับข้อมูล invoice และสร้าง XML:

```crystal
struct InvoiceItem
  property name : String
  property qty : Int32
  property price : Float64
end

def build_invoice(invoice_no : String, items : Array(InvoiceItem)) : String
  # TODO: สร้าง XML invoice
  # <invoice number="...">
  #   <items>
  #     <item name="..." qty="..." price="..." total="..."/>
  #   </items>
  #   <total>...</total>
  # </invoice>
end
```

### แบบฝึกหัดที่ 3: XPath Query
สร้าง XML document ที่มีข้อมูลพนักงาน และเขียน XPath queries:
1. หาพนักงานทุกคนใน "Engineering" department
2. หาพนักงานที่เงินเดือน > 80000
3. นับจำนวนพนักงานทั้งหมด
4. หาพนักงานที่อาวุโสที่สุด (years_of_service มากที่สุด)

---

## สรุป

ในตอนนี้เราได้เรียนรู้:

1. **`require "xml"`** - การนำเข้า XML library
2. **`XML.parse`** - การแปลง XML string เป็น `XML::Node` document
3. **`XML::Node`** - node types (element, text, attribute, comment)
4. **Element nodes** - การเข้าถึงและ traverse elements
5. **Text nodes** - การดึง text content ด้วย `inner_text` และ `content`
6. **Attribute nodes** - การอ่าน attributes ด้วย `[]` และ `[]?`
7. **children/parent/next_sibling** - การ traverse XML tree
8. **`inner_text`** - การดึง text content ทั้งหมดใน subtree
9. **XPath** - การค้นหา nodes ด้วย `xpath_node`, `xpath_nodes`, `xpath_string`
10. **`XML::Builder`** - การสร้าง XML แบบ programmatic
11. **Namespaces** - การทำงานกับ XML namespaces
12. **Error handling** - การจัดการ parse errors

XML ใน Crystal เหมาะสำหรับการทำงานกับ web services (SOAP, RSS), configuration files แบบ XML, และการ generate XML documents
