# Part 153: Crystal กับ MongoDB

## บทนำ

MongoDB เป็นฐานข้อมูล NoSQL ที่จัดเก็บข้อมูลในรูปแบบ BSON (Binary JSON) ซึ่งยืดหยุ่นมากกว่าฐานข้อมูลแบบ Relational เหมาะสำหรับข้อมูลที่มีโครงสร้างหลากหลาย เช่น บันทึกกิจกรรม, ข้อมูลผลิตภัณฑ์ที่มี attributes ต่างกัน, หรือ document-based data

## การติดตั้ง MongoDB Shard

### เพิ่ม Dependency ใน shard.yml

```yaml
# shard.yml
name: crystal_mongo_app
version: 1.0.0

dependencies:
  mongo:
    github: elbywan/cryomongo
    version: ~> 0.4.0
```

### ติดตั้ง Shards

```bash
shards install
```

### ตรวจสอบ MongoDB ทำงานอยู่

```bash
# ตรวจสอบ MongoDB service
sudo systemctl status mongod

# หรือใช้ Docker
docker run -d -p 27017:27017 --name mongodb mongo:7.0
```

## การเชื่อมต่อ MongoDB

### Connection พื้นฐาน

```crystal
# src/database/mongo_connection.cr
require "cryomongo"

module MongoConnection
  # Connection URI
  MONGO_URI = ENV.fetch("MONGODB_URI", "mongodb://localhost:27017")
  DB_NAME   = ENV.fetch("MONGODB_DB", "myapp_development")
  
  @@client : Mongo::Client? = nil
  
  def self.client : Mongo::Client
    @@client ||= Mongo::Client.new(MONGO_URI)
  end
  
  def self.database : Mongo::Database
    client[DB_NAME]
  end
  
  def self.collection(name : String) : Mongo::Collection
    database[name]
  end
  
  def self.close
    @@client.try(&.close)
  end
  
  # ทดสอบการเชื่อมต่อ
  def self.ping
    result = database.run_command({"ping" => 1})
    puts "MongoDB ping: #{result}"
    true
  rescue ex
    puts "MongoDB เชื่อมต่อล้มเหลว: #{ex.message}"
    false
  end
end
```

### Connection สำหรับ Production

```crystal
# src/config/mongo_config.cr
module MongoConfig
  # MongoDB URI พร้อม options
  def self.uri : String
    host     = ENV.fetch("MONGO_HOST", "localhost")
    port     = ENV.fetch("MONGO_PORT", "27017")
    username = ENV["MONGO_USER"]?
    password = ENV["MONGO_PASSWORD"]?
    
    if username && password
      "mongodb://#{URI.encode_path(username)}:#{URI.encode_path(password)}@#{host}:#{port}/" \
      "?authSource=admin" \
      "&maxPoolSize=20" \
      "&minPoolSize=5" \
      "&connectTimeoutMS=5000" \
      "&socketTimeoutMS=30000" \
      "&serverSelectionTimeoutMS=5000" \
      "&retryWrites=true" \
      "&w=majority"
    else
      "mongodb://#{host}:#{port}/?maxPoolSize=20"
    end
  end
end
```

## BSON Documents

### การทำงานกับ BSON

```crystal
# src/examples/bson_examples.cr
require "cryomongo"

class BSONExamples
  def self.demonstrate
    # สร้าง BSON document
    doc = BSON.new({
      "name" => "สมชาย ใจดี",
      "age" => 30,
      "email" => "somchai@example.com",
      "active" => true,
      "score" => 9.5,
      "tags" => ["crystal", "programming", "mongodb"],
      "address" => {
        "street" => "123 ถนนสุขุมวิท",
        "city" => "กรุงเทพ",
        "zip" => "10110"
      },
      "created_at" => Time.utc
    })
    
    # อ่านค่าจาก BSON
    name = doc["name"].as(String)
    age  = doc["age"].as(Int32)
    puts "ชื่อ: #{name}, อายุ: #{age}"
    
    # อ่าน nested document
    address = doc["address"].as(BSON)
    city = address["city"].as(String)
    puts "เมือง: #{city}"
    
    # อ่าน array
    tags = doc["tags"].as(Array(BSON::Value))
    tags.each { |tag| puts "  Tag: #{tag}" }
    
    # แปลงเป็น JSON
    puts "JSON: #{doc.to_json}"
  end
end
```

### BSON Types ที่ใช้บ่อย

```crystal
# src/models/mongo_types.cr

# ObjectId - ID หลักของ MongoDB
object_id = BSON::ObjectId.new
puts "ObjectId: #{object_id}"
puts "Timestamp: #{object_id.generation_time}"

# แปลง String เป็น ObjectId
id_string = "507f1f77bcf86cd799439011"
oid = BSON::ObjectId.new(id_string)

# ตรวจสอบความถูกต้อง
def valid_object_id?(str : String) : Bool
  BSON::ObjectId.new(str)
  true
rescue
  false
end
```

## CRUD Operations กับ MongoDB

### Document Model

```crystal
# src/models/article.cr
require "cryomongo"
require "json"

struct Article
  include JSON::Serializable
  
  property _id : BSON::ObjectId?
  property title : String
  property content : String
  property author : String
  property author_id : String
  property tags : Array(String)
  property status : String  # draft, published, archived
  property views : Int32
  property metadata : Hash(String, String)?
  property created_at : Time
  property updated_at : Time
  
  def initialize(
    @title : String,
    @content : String,
    @author : String,
    @author_id : String,
    @tags : Array(String) = [] of String,
    @status : String = "draft",
    @views : Int32 = 0,
    @metadata : Hash(String, String)? = nil,
    @_id : BSON::ObjectId? = nil,
    @created_at : Time = Time.utc,
    @updated_at : Time = Time.utc
  )
  end
  
  def id_string : String
    _id.try(&.to_s) || ""
  end
  
  def published?
    status == "published"
  end
  
  def to_bson : BSON
    doc = BSON.new
    doc["_id"] = _id || BSON::ObjectId.new
    doc["title"] = title
    doc["content"] = content
    doc["author"] = author
    doc["author_id"] = author_id
    doc["tags"] = tags
    doc["status"] = status
    doc["views"] = views
    doc["metadata"] = metadata.to_bson if metadata
    doc["created_at"] = created_at
    doc["updated_at"] = updated_at
    doc
  end
  
  def self.from_bson(doc : BSON) : Article
    new(
      _id: doc["_id"]?.try { |v| BSON::ObjectId.new(v.as(BSON::ObjectId).to_s) },
      title: doc["title"].as(String),
      content: doc["content"].as(String),
      author: doc["author"].as(String),
      author_id: doc["author_id"].as(String),
      tags: doc["tags"]?.try { |v| v.as(Array(BSON::Value)).map(&.as(String)) } || [] of String,
      status: doc["status"].as(String),
      views: doc["views"].as(Int32),
      created_at: doc["created_at"]?.try(&.as(Time)) || Time.utc,
      updated_at: doc["updated_at"]?.try(&.as(Time)) || Time.utc
    )
  end
end
```

### Article Repository

```crystal
# src/repositories/article_repository.cr
require "cryomongo"
require "../models/article"

class ArticleRepository
  COLLECTION = "articles"
  
  def initialize(@db : Mongo::Database)
    @collection = @db[COLLECTION]
    setup_indexes
  end
  
  # CREATE - สร้างบทความใหม่
  def create(article : Article) : Article
    doc = article.to_bson
    result = @collection.insert_one(doc)
    
    article._id = result.inserted_id.as(BSON::ObjectId)
    article
  end
  
  # CREATE MANY - สร้างหลายบทความพร้อมกัน
  def create_many(articles : Array(Article)) : Int32
    docs = articles.map(&.to_bson)
    result = @collection.insert_many(docs)
    result.inserted_count
  end
  
  # READ - ค้นหาด้วย ID
  def find_by_id(id : String) : Article?
    oid = BSON::ObjectId.new(id)
    filter = BSON.new({"_id" => oid})
    
    doc = @collection.find_one(filter)
    doc ? Article.from_bson(doc) : nil
  rescue ArgumentError
    nil  # ID ไม่ถูกต้อง
  end
  
  # READ ALL - ดึงบทความทั้งหมด
  def find_all(
    status : String? = nil,
    limit : Int32 = 20,
    skip : Int32 = 0,
    sort_by : String = "created_at",
    sort_dir : Int32 = -1
  ) : Array(Article)
    filter = BSON.new
    filter["status"] = status if status
    
    options = Mongo::Collection::FindOptions.new(
      limit: limit,
      skip: skip,
      sort: BSON.new({sort_by => sort_dir})
    )
    
    articles = [] of Article
    @collection.find(filter, options: options).each do |doc|
      articles << Article.from_bson(doc)
    end
    articles
  end
  
  # UPDATE - อัปเดตบทความ
  def update(id : String, updates : Hash(String, BSON::Value)) : Bool
    oid = BSON::ObjectId.new(id)
    filter = BSON.new({"_id" => oid})
    
    update_doc = BSON.new
    set_doc = BSON.new
    updates.each { |k, v| set_doc[k] = v }
    set_doc["updated_at"] = Time.utc
    update_doc["$set"] = set_doc
    
    result = @collection.update_one(filter, update_doc)
    result.modified_count > 0
  end
  
  # UPDATE MANY - อัปเดตหลายรายการ
  def update_many(filter_hash : Hash(String, BSON::Value), updates : Hash(String, BSON::Value)) : Int32
    filter = BSON.new
    filter_hash.each { |k, v| filter[k] = v }
    
    update_doc = BSON.new
    set_doc = BSON.new
    updates.each { |k, v| set_doc[k] = v }
    set_doc["updated_at"] = Time.utc
    update_doc["$set"] = set_doc
    
    result = @collection.update_many(filter, update_doc)
    result.modified_count
  end
  
  # DELETE - ลบบทความ
  def delete(id : String) : Bool
    oid = BSON::ObjectId.new(id)
    filter = BSON.new({"_id" => oid})
    result = @collection.delete_one(filter)
    result.deleted_count > 0
  end
  
  # FIND BY TAGS - ค้นหาด้วย tags
  def find_by_tags(tags : Array(String), limit : Int32 = 20) : Array(Article)
    filter = BSON.new({"tags" => BSON.new({"$in" => tags})})
    options = Mongo::Collection::FindOptions.new(
      limit: limit,
      sort: BSON.new({"created_at" => -1})
    )
    
    articles = [] of Article
    @collection.find(filter, options: options).each do |doc|
      articles << Article.from_bson(doc)
    end
    articles
  end
  
  # SEARCH - ค้นหา Full-Text
  def search(query : String, limit : Int32 = 20) : Array(Article)
    filter = BSON.new({"$text" => BSON.new({"$search" => query})})
    
    projection = BSON.new({
      "score" => BSON.new({"$meta" => "textScore"})
    })
    
    options = Mongo::Collection::FindOptions.new(
      limit: limit,
      projection: projection,
      sort: BSON.new({"score" => BSON.new({"$meta" => "textScore"})})
    )
    
    articles = [] of Article
    @collection.find(filter, options: options).each do |doc|
      articles << Article.from_bson(doc)
    end
    articles
  end
  
  # INCREMENT VIEWS
  def increment_views(id : String)
    oid = BSON::ObjectId.new(id)
    filter = BSON.new({"_id" => oid})
    update = BSON.new({"$inc" => BSON.new({"views" => 1})})
    @collection.update_one(filter, update)
  end
  
  # COUNT
  def count(filter_hash : Hash(String, BSON::Value) = {} of String => BSON::Value) : Int64
    filter = BSON.new
    filter_hash.each { |k, v| filter[k] = v }
    @collection.count_documents(filter)
  end
  
  private def setup_indexes
    # Index สำหรับ text search
    @collection.create_index(
      BSON.new({"title" => "text", "content" => "text"}),
      BSON.new({"name" => "text_search_idx"})
    )
    
    # Index สำหรับ status
    @collection.create_index(
      BSON.new({"status" => 1}),
      BSON.new({"name" => "status_idx"})
    )
    
    # Index สำหรับ tags
    @collection.create_index(
      BSON.new({"tags" => 1}),
      BSON.new({"name" => "tags_idx"})
    )
    
    # Compound index
    @collection.create_index(
      BSON.new({"author_id" => 1, "created_at" => -1}),
      BSON.new({"name" => "author_date_idx"})
    )
  end
end
```

## Aggregation Pipeline

### ตัวอย่าง Aggregation ต่างๆ

```crystal
# src/analytics/article_analytics.cr
class ArticleAnalytics
  def initialize(@db : Mongo::Database)
    @collection = @db["articles"]
  end
  
  # นับบทความแต่ละสถานะ
  def count_by_status : Array(NamedTuple(status: String, count: Int64))
    pipeline = [
      BSON.new({
        "$group" => BSON.new({
          "_id" => "$status",
          "count" => BSON.new({"$sum" => 1})
        })
      }),
      BSON.new({
        "$sort" => BSON.new({"count" => -1})
      })
    ]
    
    results = [] of NamedTuple(status: String, count: Int64)
    @collection.aggregate(pipeline).each do |doc|
      results << {
        status: doc["_id"].as(String),
        count: doc["count"].as(Int64)
      }
    end
    results
  end
  
  # บทความยอดนิยมตาม tag
  def top_articles_by_tag(tag : String, limit : Int32 = 10) : Array(NamedTuple(title: String, views: Int32))
    pipeline = [
      BSON.new({
        "$match" => BSON.new({
          "tags" => tag,
          "status" => "published"
        })
      }),
      BSON.new({
        "$sort" => BSON.new({"views" => -1})
      }),
      BSON.new({
        "$limit" => limit
      }),
      BSON.new({
        "$project" => BSON.new({
          "_id" => 0,
          "title" => 1,
          "views" => 1
        })
      })
    ]
    
    results = [] of NamedTuple(title: String, views: Int32)
    @collection.aggregate(pipeline).each do |doc|
      results << {
        title: doc["title"].as(String),
        views: doc["views"].as(Int32)
      }
    end
    results
  end
  
  # สถิติรายเดือน
  def monthly_stats(year : Int32) : Array(NamedTuple(month: Int32, count: Int64, total_views: Int64))
    pipeline = [
      BSON.new({
        "$match" => BSON.new({
          "created_at" => BSON.new({
            "$gte" => Time.utc(year, 1, 1),
            "$lt"  => Time.utc(year + 1, 1, 1)
          })
        })
      }),
      BSON.new({
        "$group" => BSON.new({
          "_id" => BSON.new({
            "$month" => "$created_at"
          }),
          "count" => BSON.new({"$sum" => 1}),
          "total_views" => BSON.new({"$sum" => "$views"})
        })
      }),
      BSON.new({
        "$sort" => BSON.new({"_id" => 1})
      })
    ]
    
    results = [] of NamedTuple(month: Int32, count: Int64, total_views: Int64)
    @collection.aggregate(pipeline).each do |doc|
      results << {
        month: doc["_id"].as(Int32),
        count: doc["count"].as(Int64),
        total_views: doc["total_views"].as(Int64)
      }
    end
    results
  end
  
  # Top tags
  def top_tags(limit : Int32 = 20) : Array(NamedTuple(tag: String, count: Int64))
    pipeline = [
      BSON.new({
        "$match" => BSON.new({"status" => "published"})
      }),
      BSON.new({
        "$unwind" => "$tags"
      }),
      BSON.new({
        "$group" => BSON.new({
          "_id" => "$tags",
          "count" => BSON.new({"$sum" => 1})
        })
      }),
      BSON.new({
        "$sort" => BSON.new({"count" => -1})
      }),
      BSON.new({
        "$limit" => limit
      })
    ]
    
    results = [] of NamedTuple(tag: String, count: Int64)
    @collection.aggregate(pipeline).each do |doc|
      results << {
        tag: doc["_id"].as(String),
        count: doc["count"].as(Int64)
      }
    end
    results
  end
  
  # Average views per author
  def author_performance : Array(NamedTuple(author: String, articles: Int64, avg_views: Float64))
    pipeline = [
      BSON.new({
        "$match" => BSON.new({"status" => "published"})
      }),
      BSON.new({
        "$group" => BSON.new({
          "_id" => "$author",
          "article_count" => BSON.new({"$sum" => 1}),
          "total_views" => BSON.new({"$sum" => "$views"}),
          "avg_views" => BSON.new({"$avg" => "$views"})
        })
      }),
      BSON.new({
        "$sort" => BSON.new({"avg_views" => -1})
      })
    ]
    
    results = [] of NamedTuple(author: String, articles: Int64, avg_views: Float64)
    @collection.aggregate(pipeline).each do |doc|
      results << {
        author: doc["_id"].as(String),
        articles: doc["article_count"].as(Int64),
        avg_views: doc["avg_views"].as(Float64)
      }
    end
    results
  end
end
```

## Indexing

### การสร้างและจัดการ Indexes

```crystal
# src/database/index_manager.cr
class IndexManager
  def initialize(@db : Mongo::Database)
  end
  
  def setup_all_indexes
    setup_article_indexes
    setup_user_indexes
    setup_comment_indexes
  end
  
  private def setup_article_indexes
    col = @db["articles"]
    
    # Single field indexes
    col.create_index(BSON.new({"status" => 1}))
    col.create_index(BSON.new({"author_id" => 1}))
    col.create_index(BSON.new({"created_at" => -1}))
    
    # Compound indexes
    col.create_index(
      BSON.new({"status" => 1, "created_at" => -1}),
      BSON.new({"name" => "status_date_idx"})
    )
    
    col.create_index(
      BSON.new({"author_id" => 1, "status" => 1}),
      BSON.new({"name" => "author_status_idx"})
    )
    
    # Multikey index สำหรับ array
    col.create_index(BSON.new({"tags" => 1}))
    
    # Text index สำหรับ full-text search
    col.create_index(
      BSON.new({
        "title" => "text",
        "content" => "text",
        "tags" => "text"
      }),
      BSON.new({
        "name" => "full_text_idx",
        "weights" => BSON.new({
          "title" => 10,
          "tags" => 5,
          "content" => 1
        })
      })
    )
    
    puts "สร้าง indexes สำหรับ articles สำเร็จ"
  end
  
  private def setup_user_indexes
    col = @db["users"]
    
    # Unique index
    col.create_index(
      BSON.new({"email" => 1}),
      BSON.new({"unique" => true, "name" => "email_unique_idx"})
    )
    
    # TTL index - ลบ session หลัง 24 ชั่วโมง
    col.create_index(
      BSON.new({"last_active" => 1}),
      BSON.new({
        "expireAfterSeconds" => 86400,
        "name" => "session_ttl_idx"
      })
    )
  end
  
  private def setup_comment_indexes
    col = @db["comments"]
    
    col.create_index(BSON.new({"article_id" => 1}))
    col.create_index(BSON.new({"user_id" => 1}))
    col.create_index(
      BSON.new({"article_id" => 1, "created_at" => -1}),
      BSON.new({"name" => "article_comment_idx"})
    )
  end
  
  # แสดง indexes ทั้งหมด
  def list_indexes(collection_name : String)
    col = @db[collection_name]
    puts "\nIndexes ของ #{collection_name}:"
    col.list_indexes.each do |idx|
      puts "  - #{idx["name"]}: #{idx["key"]}"
    end
  end
end
```

## GridFS สำหรับไฟล์

### การอัปโหลดและดาวน์โหลดไฟล์ด้วย GridFS

```crystal
# src/storage/gridfs_storage.cr
require "cryomongo"

class GridFSStorage
  def initialize(@db : Mongo::Database)
    @bucket = Mongo::GridFS::Bucket.new(@db)
  end
  
  # อัปโหลดไฟล์
  def upload_file(
    filename : String,
    content : Bytes,
    content_type : String,
    metadata : Hash(String, String) = {} of String => String
  ) : String
    # สร้าง metadata BSON
    meta_bson = BSON.new
    meta_bson["content_type"] = content_type
    meta_bson["original_name"] = filename
    metadata.each { |k, v| meta_bson[k] = v }
    
    # อัปโหลด
    upload_stream = @bucket.open_upload_stream(
      filename,
      metadata: meta_bson
    )
    
    upload_stream.write(content)
    upload_stream.close
    
    upload_stream.file_id.as(BSON::ObjectId).to_s
  end
  
  # อัปโหลดจาก IO
  def upload_from_io(filename : String, io : IO, content_type : String) : String
    meta = BSON.new({"content_type" => content_type})
    
    upload_stream = @bucket.open_upload_stream(filename, metadata: meta)
    
    buffer = Bytes.new(64 * 1024)  # 64KB buffer
    loop do
      bytes_read = io.read(buffer)
      break if bytes_read == 0
      upload_stream.write(buffer[0, bytes_read])
    end
    
    upload_stream.close
    upload_stream.file_id.as(BSON::ObjectId).to_s
  end
  
  # ดาวน์โหลดไฟล์
  def download_file(file_id : String) : Tuple(Bytes, String)
    oid = BSON::ObjectId.new(file_id)
    
    # หาข้อมูลไฟล์
    file_info = @bucket.find(BSON.new({"_id" => oid})).first?
    raise "ไม่พบไฟล์: #{file_id}" unless file_info
    
    content_type = file_info["metadata"]?.try(&.as(BSON)["content_type"]?.try(&.as(String))) || "application/octet-stream"
    
    # ดาวน์โหลด
    io = IO::Memory.new
    download_stream = @bucket.open_download_stream(oid)
    
    buffer = Bytes.new(64 * 1024)
    loop do
      bytes_read = download_stream.read(buffer)
      break if bytes_read == 0
      io.write(buffer[0, bytes_read])
    end
    
    download_stream.close
    {io.to_slice, content_type}
  end
  
  # ลบไฟล์
  def delete_file(file_id : String) : Bool
    oid = BSON::ObjectId.new(file_id)
    @bucket.delete(oid)
    true
  rescue
    false
  end
  
  # แสดงรายการไฟล์
  def list_files(prefix : String? = nil, limit : Int32 = 50) : Array(NamedTuple(
    id: String,
    filename: String,
    size: Int64,
    content_type: String,
    upload_date: Time
  ))
    filter = BSON.new
    filter["filename"] = BSON.new({"$regex" => "^#{prefix}"}) if prefix
    
    results = [] of NamedTuple(id: String, filename: String, size: Int64, content_type: String, upload_date: Time)
    
    @bucket.find(filter, limit: limit).each do |file|
      results << {
        id: file["_id"].as(BSON::ObjectId).to_s,
        filename: file["filename"].as(String),
        size: file["length"].as(Int64),
        content_type: file["metadata"]?.try(&.as(BSON)["content_type"]?.try(&.as(String))) || "unknown",
        upload_date: file["uploadDate"].as(Time)
      }
    end
    
    results
  end
end
```

## Real-World App: Content Management System

### CMS Service

```crystal
# src/services/cms_service.cr
require "../repositories/article_repository"
require "../storage/gridfs_storage"
require "../analytics/article_analytics"

class CMSService
  def initialize(@db : Mongo::Database)
    @articles = ArticleRepository.new(@db)
    @storage  = GridFSStorage.new(@db)
    @analytics = ArticleAnalytics.new(@db)
  end
  
  # สร้างบทความพร้อมรูปภาพ
  def create_article_with_image(
    title : String,
    content : String,
    author : String,
    author_id : String,
    tags : Array(String),
    image_path : String? = nil
  ) : NamedTuple(success: Bool, article_id: String?, error: String?)
    image_id = nil
    
    # อัปโหลดรูปภาพถ้ามี
    if image_path
      begin
        File.open(image_path, "rb") do |file|
          ext = File.extname(image_path).downcase
          content_type = case ext
                         when ".jpg", ".jpeg" then "image/jpeg"
                         when ".png"          then "image/png"
                         when ".gif"          then "image/gif"
                         when ".webp"         then "image/webp"
                         else "application/octet-stream"
                         end
          
          filename = "article_#{Time.utc.to_unix}#{ext}"
          image_id = @storage.upload_from_io(filename, file, content_type)
        end
      rescue ex
        return {success: false, article_id: nil, error: "อัปโหลดรูปภาพล้มเหลว: #{ex.message}"}
      end
    end
    
    # สร้างบทความ
    metadata = {} of String => String
    metadata["cover_image_id"] = image_id if image_id
    
    article = Article.new(
      title: title,
      content: content,
      author: author,
      author_id: author_id,
      tags: tags,
      metadata: metadata.empty? ? nil : metadata
    )
    
    saved = @articles.create(article)
    {success: true, article_id: saved._id.try(&.to_s), error: nil}
  rescue ex
    {success: false, article_id: nil, error: ex.message}
  end
  
  # เผยแพร่บทความ
  def publish_article(article_id : String) : Bool
    @articles.update(article_id, {
      "status" => "published" as BSON::Value,
      "published_at" => Time.utc as BSON::Value
    })
  end
  
  # Dashboard สถิติ
  def dashboard_stats : Hash(String, Object)
    stats = {} of String => Object
    
    # จำนวนบทความแต่ละสถานะ
    @analytics.count_by_status.each do |item|
      stats["#{item[:status]}_count"] = item[:count]
    end
    
    # Top tags
    stats["top_tags"] = @analytics.top_tags(10)
    
    # Author performance
    stats["author_performance"] = @analytics.author_performance
    
    stats
  end
  
  # ค้นหาบทความ
  def search(query : String, page : Int32 = 1, per_page : Int32 = 10) : NamedTuple(
    articles: Array(Article),
    total: Int64,
    page: Int32,
    per_page: Int32
  )
    articles = @articles.search(query, limit: per_page)
    total = @articles.count({"status" => "published" as BSON::Value})
    
    {
      articles: articles,
      total: total,
      page: page,
      per_page: per_page
    }
  end
end
```

### ตัวอย่างการใช้งาน

```crystal
# src/main.cr
require "cryomongo"
require "./database/mongo_connection"
require "./repositories/article_repository"
require "./services/cms_service"

# เชื่อมต่อ
MongoConnection.ping

db = MongoConnection.database
cms = CMSService.new(db)

# สร้างบทความ
puts "=== สร้างบทความ ==="
result = cms.create_article_with_image(
  title: "บทความทดสอบ Crystal กับ MongoDB",
  content: "เนื้อหาบทความ...",
  author: "สมชาย ใจดี",
  author_id: "user123",
  tags: ["crystal", "mongodb", "programming"]
)

if result[:success]
  puts "สร้างบทความสำเร็จ! ID: #{result[:article_id]}"
  
  # เผยแพร่
  cms.publish_article(result[:article_id].not_nil!)
  puts "เผยแพร่บทความแล้ว"
else
  puts "ข้อผิดพลาด: #{result[:error]}"
end

# ค้นหา
puts "\n=== ค้นหาบทความ ==="
search_result = cms.search("MongoDB")
puts "พบ #{search_result[:total]} บทความ"
search_result[:articles].each do |article|
  puts "  - #{article.title} (#{article.author})"
end

# สถิติ
puts "\n=== สถิติ ==="
cms.dashboard_stats.each do |key, value|
  puts "#{key}: #{value}"
end

MongoConnection.close
```

## สรุป

ในบทนี้เราได้เรียนรู้:
- การติดตั้งและใช้งาน `cryomongo` shard
- การทำงานกับ BSON documents
- CRUD operations ครบทุกรูปแบบ
- Aggregation Pipeline สำหรับวิเคราะห์ข้อมูล
- การสร้าง Indexes ที่มีประสิทธิภาพ
- GridFS สำหรับจัดเก็บไฟล์ขนาดใหญ่
- ตัวอย่าง CMS จริงๆ ที่ใช้ MongoDB

## ขั้นตอนต่อไป

ใน **Part 154** เราจะเรียนรู้การใช้งาน Crystal กับ **Elasticsearch** สำหรับการทำ Full-Text Search และการวิเคราะห์ข้อมูลขนาดใหญ่
