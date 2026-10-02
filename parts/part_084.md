# Part 84: XML

## บทนำ

XML (eXtensible Markup Language) ยังใช้งานอย่างแพร่หลายใน configuration, SOAP web services, และ data exchange Crystal มี standard library `xml` สำหรับ parse และ build XML

---

## 1. require "xml" และ XML.parse

```crystal
require "xml"

# parse XML string
xml_str = "<root><name>Alice</name><age>30</age></root>"
doc = XML.parse(xml_str)

# access root element
root = doc.root!
puts root.name   # => root

# access child elements
name_el = root.xpath_node("name")
puts name_el.try(&.content)  # => Alice

age_el = root.xpath_node("age")
puts age_el.try(&.content)   # => 30

# Parse complex XML
xml_str = <<-XML
  <?xml version="1.0" encoding="UTF-8"?>
  <catalog>
    <book id="1">
      <title>Crystal Programming</title>
      <author>Alice Smith</author>
      <price>49.99</price>
      <tags>
        <tag>programming</tag>
        <tag>crystal</tag>
      </tags>
    </book>
    <book id="2">
      <title>Web Development with Crystal</title>
      <author>Bob Jones</author>
      <price>39.99</price>
    </book>
  </catalog>
  XML

doc = XML.parse(xml_str)
root = doc.root!
puts root.name  # => catalog

# วนลูปผ่าน children
root.children.each do |node|
  next unless node.element?  # skip text nodes
  puts "Book: #{node.name}"
end
```

---

## 2. XML::Node - Traversal

```crystal
require "xml"

xml_str = <<-XML
  <library>
    <section name="Fiction">
      <book id="1">
        <title>The Crystal Story</title>
        <author>Alice</author>
      </book>
      <book id="2">
        <title>Adventures in Crystal</title>
        <author>Bob</author>
      </book>
    </section>
    <section name="Technical">
      <book id="3">
        <title>Crystal Programming Guide</title>
        <author>Carol</author>
      </book>
    </section>
  </library>
  XML

doc = XML.parse(xml_str)
root = doc.root!

# traverse children
root.children.each do |section|
  next unless section.element?
  
  section_name = section["name"]  # attribute
  puts "Section: #{section_name}"
  
  section.children.each do |book|
    next unless book.element?
    
    book_id = book["id"]
    title = book.xpath_node("title").try(&.content) || ""
    author = book.xpath_node("author").try(&.content) || ""
    
    puts "  [#{book_id}] #{title} by #{author}"
  end
end

# Node properties
node = doc.root!
puts node.type     # => XML::Node::Type::ELEMENT_NODE
puts node.element? # => true
puts node.text?    # => false

# Parent/sibling traversal
sections = root.children.select(&.element?)
first_section = sections.first
puts first_section.name          # => section
puts first_section.parent.try(&.name)  # => library
```

---

## 3. XPath Queries

```crystal
require "xml"

xml_str = <<-XML
  <employees>
    <employee id="1" dept="Engineering">
      <name>Alice</name>
      <salary>75000</salary>
      <skills>
        <skill>Crystal</skill>
        <skill>Ruby</skill>
      </skills>
    </employee>
    <employee id="2" dept="Design">
      <name>Bob</name>
      <salary>55000</salary>
      <skills>
        <skill>Figma</skill>
      </skills>
    </employee>
    <employee id="3" dept="Engineering">
      <name>Carol</name>
      <salary>85000</salary>
      <skills>
        <skill>Crystal</skill>
        <skill>Go</skill>
      </skills>
    </employee>
  </employees>
  XML

doc = XML.parse(xml_str)
root = doc.root!

# xpath_nodes - คืน NodeSet
all_employees = root.xpath_nodes("employee")
puts "Total employees: #{all_employees.size}"

# xpath กับ attribute filter
engineers = root.xpath_nodes("employee[@dept='Engineering']")
puts "Engineers: #{engineers.size}"
engineers.each do |emp|
  name = emp.xpath_node("name").try(&.content) || ""
  puts "  #{name}"
end

# xpath descendant
all_skills = root.xpath_nodes("//skill")
skill_names = all_skills.map { |s| s.content }.uniq.sort
puts "All skills: #{skill_names.inspect}"

# xpath string value
first_name = root.xpath_string("employee[1]/name")
puts "First employee: #{first_name}"

# xpath number
total_salary = root.xpath_float("sum(employee/salary)")
puts "Total salary: #{total_salary}"

# xpath boolean
has_designer = root.xpath_bool("boolean(employee[@dept='Design'])")
puts "Has designer: #{has_designer}"
```

---

## 4. XML Attributes

```crystal
require "xml"

xml_str = <<-XML
  <config version="2.0" env="production">
    <database host="localhost" port="5432" name="myapp">
      <option key="pool_size" value="10"/>
      <option key="timeout" value="30"/>
    </database>
    <server host="0.0.0.0" port="8080" ssl="true"/>
  </config>
  XML

doc = XML.parse(xml_str)
root = doc.root!

# อ่าน attributes
puts root["version"]   # => 2.0
puts root["env"]       # => production
puts root["missing"]?  # => nil

# วนลูปผ่าน attributes
root.attributes.each do |attr|
  puts "  #{attr.name} = #{attr.content}"
end

# Database config
db = root.xpath_node("database")
if db
  puts "DB: #{db["host"]}:#{db["port"]}/#{db["name"]}"
  
  # Options
  options = {} of String => String
  db.xpath_nodes("option").each do |opt|
    options[opt["key"]] = opt["value"]
  end
  puts "Options: #{options.inspect}"
end

# Server config
server = root.xpath_node("server")
if server
  ssl = server["ssl"]? == "true"
  puts "Server: #{server["host"]}:#{server["port"]} (SSL: #{ssl})"
end
```

---

## 5. XML::Builder

```crystal
require "xml"

# สร้าง XML ด้วย XML::Builder
output = IO::Memory.new
xml = XML::Builder.new(output)

xml.document("1.0", "UTF-8") do
  xml.element("catalog") do
    xml.element("book", id: "1") do
      xml.element("title") { xml.text "Crystal Programming" }
      xml.element("author") { xml.text "Alice Smith" }
      xml.element("price") { xml.text "49.99" }
    end
    xml.element("book", id: "2") do
      xml.element("title") { xml.text "Web Dev with Crystal" }
      xml.element("author") { xml.text "Bob Jones" }
      xml.element("price") { xml.text "39.99" }
    end
  end
end

puts output.to_s

# สร้าง RSS feed
def build_rss(title : String, link : String, items : Array(Hash(String, String))) : String
  output = IO::Memory.new
  xml = XML::Builder.new(output)
  
  xml.document("1.0", "UTF-8") do
    xml.element("rss", version: "2.0") do
      xml.element("channel") do
        xml.element("title") { xml.text title }
        xml.element("link") { xml.text link }
        xml.element("description") { xml.text "#{title} RSS Feed" }
        
        items.each do |item|
          xml.element("item") do
            xml.element("title") { xml.text item["title"] }
            xml.element("link") { xml.text item["link"] }
            xml.element("description") { xml.text item["description"] }
            xml.element("pubDate") { xml.text item["date"] }
          end
        end
      end
    end
  end
  
  output.to_s
end

rss = build_rss("My Blog", "https://myblog.com", [
  {"title" => "Post 1", "link" => "https://myblog.com/1", "description" => "First post", "date" => "Mon, 01 Jan 2024"},
  {"title" => "Post 2", "link" => "https://myblog.com/2", "description" => "Second post", "date" => "Tue, 02 Jan 2024"},
])
puts rss
```

---

## 6. XML Namespaces

```crystal
require "xml"

# Parse XML กับ namespaces
xml_str = <<-XML
  <?xml version="1.0"?>
  <root xmlns="http://default.ns" xmlns:app="http://app.ns">
    <app:user id="1">
      <name>Alice</name>
      <app:role>admin</app:role>
    </app:user>
  </root>
  XML

doc = XML.parse(xml_str)
root = doc.root!

# XPath กับ namespace
# (ต้อง register namespace ก่อน)
namespaces = {"app" => "http://app.ns"}

# หา app:user elements
users = root.xpath_nodes("//app:user", namespaces)
users.each do |user|
  puts "User ID: #{user["id"]}"
  name = user.xpath_node("*[local-name()='name']").try(&.content)
  puts "  Name: #{name}"
end

# Namespace-aware builder
def build_soap_envelope(action : String, body : String) : String
  output = IO::Memory.new
  xml = XML::Builder.new(output)
  
  xml.document("1.0", "UTF-8") do
    xml.element("soap:Envelope",
                "xmlns:soap": "http://schemas.xmlsoap.org/soap/envelope/",
                "xmlns:xsi": "http://www.w3.org/2001/XMLSchema-instance") do
      xml.element("soap:Header") do
        xml.element("Action") { xml.text action }
      end
      xml.element("soap:Body") do
        xml.text body  # simplified - real code would nest properly
      end
    end
  end
  
  output.to_s
end
```

---

## 7. Read/Write XML Files

```crystal
require "xml"

# อ่าน XML file
def read_xml_file(path : String) : XML::Node
  XML.parse(File.read(path))
end

# เขียน XML file
def write_xml_file(path : String, &block : XML::Builder ->)
  File.open(path, "w") do |file|
    xml = XML::Builder.new(file)
    xml.document("1.0", "UTF-8") do
      block.call(xml)
    end
  end
end

# สร้าง config XML
write_xml_file("/tmp/config.xml") do |xml|
  xml.element("configuration") do
    xml.element("appSettings") do
      [
        {"key" => "server", "value" => "localhost"},
        {"key" => "port", "value" => "8080"},
        {"key" => "debug", "value" => "false"},
      ].each do |setting|
        xml.element("add", key: setting["key"], value: setting["value"])
      end
    end
    xml.element("connectionStrings") do
      xml.element("add",
                  name: "DefaultConnection",
                  connectionString: "Host=localhost;Database=myapp",
                  providerName: "Npgsql")
    end
  end
end

puts File.read("/tmp/config.xml")

# โหลด config กลับมา
doc = read_xml_file("/tmp/config.xml")
root = doc.root!

settings = {} of String => String
root.xpath_nodes("//appSettings/add").each do |node|
  settings[node["key"]] = node["value"]
end

puts "\nSettings:"
settings.each { |k, v| puts "  #{k} = #{v}" }

File.delete("/tmp/config.xml")
```

---

## 8. XML Transformation

```crystal
require "xml"

# แปลง XML เป็น Hash
def xml_to_hash(node : XML::Node) : Hash(String, _)
  result = {} of String => String | Array(Hash(String, _)) | Hash(String, _)
  
  node.attributes.each do |attr|
    result["@#{attr.name}"] = attr.content
  end
  
  children_by_name = {} of String => Array(XML::Node)
  node.children.each do |child|
    next unless child.element?
    children_by_name[child.name] ||= [] of XML::Node
    children_by_name[child.name] << child
  end
  
  children_by_name.each do |name, children|
    if children.size == 1 && children.first.children.none?(&.element?)
      # leaf node
      result[name] = children.first.content
    else
      converted = children.map { |c| xml_to_hash(c) }
      result[name] = converted.size == 1 ? converted.first : converted
    end
  end
  
  result
end

xml_str = <<-XML
  <person id="1">
    <name>Alice</name>
    <age>30</age>
    <address>
      <city>Bangkok</city>
      <country>Thailand</country>
    </address>
  </person>
  XML

doc = XML.parse(xml_str)
hash = xml_to_hash(doc.root!)
puts hash.inspect

# XML error handling
def safe_parse_xml(xml_str : String) : XML::Node?
  XML.parse(xml_str)
rescue XML::Error => e
  puts "XML parse error: #{e.message}"
  nil
end

puts safe_parse_xml("<valid>xml</valid>").inspect
puts safe_parse_xml("<invalid>xml").inspect
```

---

## แบบฝึกหัด

### แบบฝึกหัดที่ 1
เขียน `XMLConfig` class ที่โหลด config จาก XML file รองรับ XPath queries

### แบบฝึกหัดที่ 2
เขียน function ที่แปลง XML เป็น JSON

### แบบฝึกหัดที่ 3
เขียน `SitemapBuilder` ที่สร้าง sitemap.xml ตาม sitemap protocol

### เฉลย

```crystal
require "xml"
require "json"

# แบบฝึกหัดที่ 3: SitemapBuilder
struct SitemapEntry
  property url : String
  property last_modified : Time?
  property change_freq : String?
  property priority : Float64?
  
  def initialize(@url, @last_modified = nil, @change_freq = nil, @priority = nil)
  end
end

class SitemapBuilder
  def initialize(@base_url : String)
    @entries = [] of SitemapEntry
  end
  
  def add(path : String, last_modified : Time? = nil, change_freq : String? = nil, priority : Float64? = nil)
    url = "#{@base_url}#{path}"
    @entries << SitemapEntry.new(url, last_modified, change_freq, priority)
    self
  end
  
  def build : String
    output = IO::Memory.new
    xml = XML::Builder.new(output)
    
    xml.document("1.0", "UTF-8") do
      xml.element("urlset",
                  "xmlns": "http://www.sitemaps.org/schemas/sitemap/0.9") do
        @entries.each do |entry|
          xml.element("url") do
            xml.element("loc") { xml.text entry.url }
            
            if lm = entry.last_modified
              xml.element("lastmod") { xml.text lm.to_s("%Y-%m-%d") }
            end
            
            if cf = entry.change_freq
              xml.element("changefreq") { xml.text cf }
            end
            
            if p = entry.priority
              xml.element("priority") { xml.text p.to_s }
            end
          end
        end
      end
    end
    
    output.to_s
  end
  
  def save(path : String)
    File.write(path, build)
  end
end

sitemap = SitemapBuilder.new("https://example.com")
  .add("/", last_modified: Time.utc, change_freq: "daily", priority: 1.0)
  .add("/about", change_freq: "monthly", priority: 0.8)
  .add("/blog", last_modified: Time.utc, change_freq: "weekly", priority: 0.9)
  .add("/contact", change_freq: "yearly", priority: 0.5)

puts sitemap.build
```

---

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **XML.parse** - parse XML string เป็น document
2. **XML::Node** - node traversal
3. **XPath queries** - xpath_nodes, xpath_node, xpath_string, xpath_float
4. **XML attributes** - อ่านและ iterate attributes
5. **XML::Builder** - สร้าง XML document
6. **XML namespaces** - default และ prefixed namespaces
7. **File operations** - read/write XML files
8. **Transformation** - XML to Hash, error handling

XML เป็น format ที่ยังใช้อย่างแพร่หลายใน enterprise systems, SOAP APIs, และ configuration files
