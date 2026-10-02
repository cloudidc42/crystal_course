# Part 154: SQLite - การใช้งาน SQLite ใน Crystal

## บทนำ

SQLite เป็น embedded database ที่ไม่ต้องติดตั้ง server แยก เหมาะสำหรับ desktop apps, mobile apps, prototypes และ small web apps

## การติดตั้ง

```yaml
# shard.yml
dependencies:
  sqlite3:
    github: crystal-lang/crystal-sqlite3
    version: ~> 0.20.0
  db:
    github: crystal-lang/crystal-db
    version: ~> 0.13.0
```

## เชื่อมต่อ SQLite

```crystal
require "db"
require "sqlite3"

# File-based database
DB.open("sqlite3://./myapp.db") do |db|
  puts "Connected to SQLite"
end

# In-memory database (ข้อมูลหายเมื่อปิด connection)
DB.open("sqlite3://:memory:") do |db|
  puts "Connected to in-memory SQLite"
end

# Persistent connection
DATABASE = DB.open("sqlite3://./data/myapp.db")
at_exit { DATABASE.close }
```

## CRUD พื้นฐาน

```crystal
require "db"
require "sqlite3"

struct Note
  include DB::Serializable

  property id : Int64
  property title : String
  property content : String
  property tags : String  # SQLite ไม่มี array type - ใช้ comma-separated
  property pinned : Bool
  property created_at : Time
  property updated_at : Time
end

DB.open("sqlite3://./notes.db") do |db|

  # Pragmas สำหรับ performance
  db.exec("PRAGMA journal_mode = WAL")
  db.exec("PRAGMA synchronous = NORMAL")
  db.exec("PRAGMA cache_size = -64000")  # 64MB cache
  db.exec("PRAGMA foreign_keys = ON")

  # CREATE TABLE
  db.exec <<-SQL
    CREATE TABLE IF NOT EXISTS notes (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      title TEXT NOT NULL,
      content TEXT NOT NULL DEFAULT '',
      tags TEXT NOT NULL DEFAULT '',
      pinned INTEGER NOT NULL DEFAULT 0,
      created_at DATETIME NOT NULL DEFAULT (datetime('now')),
      updated_at DATETIME NOT NULL DEFAULT (datetime('now'))
    )
  SQL

  db.exec("CREATE INDEX IF NOT EXISTS idx_notes_pinned ON notes(pinned)")

  # INSERT
  result = db.exec(
    "INSERT INTO notes (title, content, tags) VALUES (?, ?, ?)",
    "Crystal Notes", "Crystal is amazing!", "crystal,programming"
  )
  puts "Inserted ID: #{result.last_insert_id}"

  # SELECT
  notes = db.query_all(
    "SELECT * FROM notes ORDER BY pinned DESC, created_at DESC",
    as: Note
  )
  notes.each { |n| puts "#{n.id}: #{n.title}" }

  # SELECT ONE
  note = db.query_one?("SELECT * FROM notes WHERE id = ?", 1_i64, as: Note)
  puts note.try(&.title) || "Not found"

  # UPDATE
  db.exec(
    "UPDATE notes SET title = ?, updated_at = datetime('now') WHERE id = ?",
    "Updated Title", 1_i64
  )

  # DELETE
  db.exec("DELETE FROM notes WHERE id = ?", 99_i64)

end
```

## SQLite Pragmas

```crystal
require "db"
require "sqlite3"

DB.open("sqlite3://./app.db") do |db|

  # Performance pragmas
  db.exec("PRAGMA journal_mode = WAL")      # Write-Ahead Logging - ดีที่สุดสำหรับ web apps
  db.exec("PRAGMA synchronous = NORMAL")    # Balance performance/safety
  db.exec("PRAGMA cache_size = -32000")     # 32MB page cache
  db.exec("PRAGMA temp_store = MEMORY")     # Temp tables in RAM
  db.exec("PRAGMA mmap_size = 268435456")   # Memory-mapped I/O 256MB

  # Security pragmas
  db.exec("PRAGMA foreign_keys = ON")       # Enforce foreign key constraints

  # Read pragmas
  journal = db.query_one("PRAGMA journal_mode", as: String)
  puts "Journal mode: #{journal}"

  page_size = db.query_one("PRAGMA page_size", as: Int32)
  puts "Page size: #{page_size}"

  db_size = db.query_one("PRAGMA page_count", as: Int32) * page_size
  puts "Database size: #{db_size} bytes"

  # Analyze (update query planner statistics)
  db.exec("ANALYZE")

  # Vacuum (defragment database)
  db.exec("VACUUM")

end
```

## In-Memory Database

```crystal
require "db"
require "sqlite3"

# In-memory database เหมาะสำหรับ testing และ caching
class InMemoryCache
  def initialize
    @db = DB.open("sqlite3://:memory:")
    setup_schema
  end

  def finalize
    @db.close
  end

  private def setup_schema
    @db.exec <<-SQL
      CREATE TABLE cache_entries (
        key TEXT PRIMARY KEY,
        value TEXT NOT NULL,
        expires_at INTEGER,
        created_at INTEGER NOT NULL DEFAULT (strftime('%s', 'now'))
      )
    SQL
  end

  def set(key : String, value : String, ttl_seconds : Int32? = nil)
    expires_at = ttl_seconds ? Time.local.to_unix + ttl_seconds : nil

    @db.exec(
      <<-SQL,
        INSERT INTO cache_entries (key, value, expires_at)
        VALUES (?, ?, ?)
        ON CONFLICT(key) DO UPDATE SET
          value = excluded.value,
          expires_at = excluded.expires_at,
          created_at = strftime('%s', 'now')
      SQL
      key, value, expires_at
    )
  end

  def get(key : String) : String?
    now = Time.local.to_unix

    entry = @db.query_one?(
      "SELECT value, expires_at FROM cache_entries WHERE key = ?",
      key,
      as: {value: String, expires_at: Int64?}
    )

    return nil unless entry

    # ตรวจสอบ expiration
    if exp = entry[:expires_at]
      if now > exp
        delete(key)
        return nil
      end
    end

    entry[:value]
  end

  def delete(key : String)
    @db.exec("DELETE FROM cache_entries WHERE key = ?", key)
  end

  def cleanup_expired
    @db.exec(
      "DELETE FROM cache_entries WHERE expires_at IS NOT NULL AND expires_at < ?",
      Time.local.to_unix
    )
  end

  def stats : NamedTuple(total: Int64, expired: Int64)
    now = Time.local.to_unix
    total = @db.query_one("SELECT COUNT(*) FROM cache_entries", as: Int64)
    expired = @db.query_one(
      "SELECT COUNT(*) FROM cache_entries WHERE expires_at IS NOT NULL AND expires_at < ?",
      now, as: Int64
    )
    {total: total, expired: expired}
  end
end

# ใช้งาน
cache = InMemoryCache.new

cache.set("user:1", {"name" => "สมชาย", "role" => "admin"}.to_json, ttl_seconds: 300)
cache.set("config:theme", "dark")  # ไม่มี TTL

if value = cache.get("user:1")
  user = JSON.parse(value)
  puts "User: #{user["name"]}"
end

stats = cache.stats
puts "Cache: #{stats[:total]} entries, #{stats[:expired]} expired"
```

## Transactions และ WAL Mode

```crystal
require "db"
require "sqlite3"

DB.open("sqlite3://./app.db") do |db|

  db.exec("PRAGMA journal_mode = WAL")
  db.exec("PRAGMA foreign_keys = ON")

  # Transaction พื้นฐาน
  db.transaction do |tx|
    cnn = tx.connection

    user_id = cnn.query_one(
      "INSERT INTO users (name, email) VALUES (?, ?) RETURNING id",
      "สมชาย", "somchai@example.com",
      as: Int64
    )

    cnn.exec(
      "INSERT INTO user_profiles (user_id, bio) VALUES (?, ?)",
      user_id, ""
    )

    puts "Created user #{user_id}"
  end

  # Exclusive transaction สำหรับ heavy write
  db.exec("BEGIN EXCLUSIVE")
  begin
    db.exec("UPDATE counter SET value = value + 1 WHERE key = 'visits'")
    db.exec("COMMIT")
  rescue ex
    db.exec("ROLLBACK")
    puts "Error: #{ex.message}"
  end

  # Deferred transaction (default)
  db.exec("BEGIN DEFERRED")
  begin
    rows = db.query_all("SELECT id FROM items WHERE needs_update = 1", as: Int64)
    rows.each do |id|
      db.exec("UPDATE items SET processed = 1 WHERE id = ?", id)
    end
    db.exec("COMMIT")
  rescue
    db.exec("ROLLBACK")
  end

end
```

## Full-text Search ด้วย FTS5

```crystal
require "db"
require "sqlite3"

DB.open("sqlite3://./search.db") do |db|

  # สร้าง FTS5 virtual table
  db.exec <<-SQL
    CREATE VIRTUAL TABLE IF NOT EXISTS articles_fts USING fts5(
      title,
      content,
      tokenize = "unicode61"
    )
  SQL

  # Insert
  db.exec(
    "INSERT INTO articles_fts (rowid, title, content) VALUES (?, ?, ?)",
    1_i64, "Crystal Programming", "Crystal is a fast, compiled programming language"
  )

  db.exec(
    "INSERT INTO articles_fts (rowid, title, content) VALUES (?, ?, ?)",
    2_i64, "Ruby vs Crystal", "Crystal has Ruby-like syntax but compiles to native code"
  )

  # Search
  results = db.query_all(
    "SELECT rowid, title, snippet(articles_fts, 1, '<b>', '</b>', '...', 20) as excerpt FROM articles_fts WHERE articles_fts MATCH ? ORDER BY rank",
    "crystal programming",
    as: {rowid: Int64, title: String, excerpt: String}
  )

  results.each do |r|
    puts "#{r[:title]}: #{r[:excerpt]}"
  end

  # Phrase search
  phrase_results = db.query_all(
    "SELECT rowid, title FROM articles_fts WHERE articles_fts MATCH ? ORDER BY rank",
    '"crystal language"',  # ใส่ quotes สำหรับ phrase search
    as: {rowid: Int64, title: String}
  )

  # Prefix search
  prefix_results = db.query_all(
    "SELECT rowid, title FROM articles_fts WHERE articles_fts MATCH ?",
    "crys*",  # prefix
    as: {rowid: Int64, title: String}
  )

end
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: Local Task Manager

```crystal
require "db"
require "sqlite3"
require "json"

struct Task
  include DB::Serializable

  property id : Int64
  property title : String
  property description : String
  property status : String
  property priority : Int32
  property due_date : String?
  property tags : String
  property created_at : Time
  property completed_at : Time?

  def tag_list : Array(String)
    tags.split(",").map(&.strip).reject(&.empty?)
  end
end

class TaskManager
  def initialize(db_path : String = "./tasks.db")
    @db = DB.open("sqlite3://#{db_path}")
    setup
  end

  def finalize
    @db.close
  end

  private def setup
    @db.exec("PRAGMA journal_mode = WAL")
    @db.exec("PRAGMA foreign_keys = ON")

    @db.exec <<-SQL
      CREATE TABLE IF NOT EXISTS tasks (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        title TEXT NOT NULL,
        description TEXT NOT NULL DEFAULT '',
        status TEXT NOT NULL DEFAULT 'todo' CHECK(status IN ('todo','in_progress','done','cancelled')),
        priority INTEGER NOT NULL DEFAULT 2 CHECK(priority BETWEEN 1 AND 5),
        due_date TEXT,
        tags TEXT NOT NULL DEFAULT '',
        created_at DATETIME NOT NULL DEFAULT (datetime('now')),
        completed_at DATETIME
      )
    SQL

    @db.exec("CREATE INDEX IF NOT EXISTS idx_tasks_status ON tasks(status)")
    @db.exec("CREATE INDEX IF NOT EXISTS idx_tasks_priority ON tasks(priority DESC)")
  end

  def add(title : String, description : String = "", priority : Int32 = 2, tags : Array(String) = [] of String, due_date : String? = nil) : Task
    result = @db.exec(
      "INSERT INTO tasks (title, description, priority, tags, due_date) VALUES (?, ?, ?, ?, ?)",
      title, description, priority, tags.join(","), due_date
    )
    @db.query_one("SELECT * FROM tasks WHERE id = ?", result.last_insert_id, as: Task)
  end

  def list(status : String? = nil, priority : Int32? = nil) : Array(Task)
    conditions = [] of String
    args = [] of DB::Any

    if status
      conditions << "status = ?"
      args << status
    end

    if priority
      conditions << "priority = ?"
      args << priority
    end

    where_clause = conditions.empty? ? "" : "WHERE #{conditions.join(" AND ")}"

    @db.query_all(
      "SELECT * FROM tasks #{where_clause} ORDER BY priority DESC, created_at DESC",
      *args,
      as: Task
    )
  end

  def complete(id : Int64) : Bool
    result = @db.exec(
      "UPDATE tasks SET status = 'done', completed_at = datetime('now') WHERE id = ? AND status != 'done'",
      id
    )
    result.rows_affected > 0
  end

  def delete(id : Int64) : Bool
    result = @db.exec("DELETE FROM tasks WHERE id = ?", id)
    result.rows_affected > 0
  end

  def stats : NamedTuple(total: Int64, todo: Int64, in_progress: Int64, done: Int64)
    row = @db.query_one(
      <<-SQL,
        SELECT
          COUNT(*) as total,
          SUM(CASE WHEN status = 'todo' THEN 1 ELSE 0 END) as todo,
          SUM(CASE WHEN status = 'in_progress' THEN 1 ELSE 0 END) as in_progress,
          SUM(CASE WHEN status = 'done' THEN 1 ELSE 0 END) as done
        FROM tasks
      SQL
      as: {total: Int64, todo: Int64, in_progress: Int64, done: Int64}
    )
    row
  end

  def search(query : String) : Array(Task)
    pattern = "%#{query}%"
    @db.query_all(
      "SELECT * FROM tasks WHERE title LIKE ? OR description LIKE ? OR tags LIKE ? ORDER BY priority DESC",
      pattern, pattern, pattern,
      as: Task
    )
  end
end

# ใช้งาน
mgr = TaskManager.new

t1 = mgr.add("เรียน Crystal", "อ่าน documentation", priority: 5, tags: ["learning", "crystal"])
t2 = mgr.add("สร้าง API", "REST API ด้วย Kemal", priority: 4, due_date: "2026-12-31")
t3 = mgr.add("ทำ homework", priority: 3, tags: ["school"])

puts "Tasks:"
mgr.list.each { |t| puts "  [#{t.status}] P#{t.priority} - #{t.title}" }

mgr.complete(t1.id)

stats = mgr.stats
puts "\nStats: #{stats[:total]} total, #{stats[:todo]} todo, #{stats[:done]} done"

puts "\nSearch 'crystal':"
mgr.search("crystal").each { |t| puts "  #{t.title}" }
```

## สรุป

ในบทนี้เราได้เรียนรู้:

1. **crystal-sqlite3**: shard สำหรับ SQLite
2. **Connection String**: `sqlite3://./file.db` หรือ `sqlite3://:memory:`
3. **Pragmas**: journal_mode=WAL, foreign_keys, cache_size
4. **AUTOINCREMENT**: primary key auto-increment
5. **In-Memory DB**: database ใน RAM สำหรับ testing/caching
6. **Transactions**: BEGIN/COMMIT/ROLLBACK, EXCLUSIVE
7. **FTS5**: full-text search virtual table
8. **ON CONFLICT**: handle duplicates (upsert)
9. **WAL Mode**: Write-Ahead Logging สำหรับ performance
10. **Embedded**: ไม่ต้องมี server แยก

SQLite เหมาะสำหรับ: local apps, testing, prototypes, small web apps, caching
