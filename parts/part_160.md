# Part 160: Database Optimization ใน Crystal

## บทนำ

Database optimization เป็นเรื่องสำคัญมากสำหรับ application ที่ต้องการประสิทธิภาพสูง ในบทนี้จะครอบคลุม indexes, N+1 queries, eager loading, connection pooling, และการวิเคราะห์ queries

## Indexes

### ประเภทของ Index

```sql
-- B-tree Index (default) - เหมาะสำหรับ equality, range, sorting
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- Hash Index - เหมาะสำหรับ equality เท่านั้น
CREATE INDEX idx_sessions_token ON sessions USING HASH (token);

-- GIN Index - สำหรับ array, full-text search
CREATE INDEX idx_products_tags ON products USING GIN (tags);
CREATE INDEX idx_products_fts ON products USING GIN (
  to_tsvector('english', name || ' ' || COALESCE(description, ''))
);

-- GiST Index - สำหรับ geometric data, range types
CREATE INDEX idx_events_period ON events USING GIST (period);

-- BRIN Index - สำหรับ large tables ที่ข้อมูลเรียงตาม physical order
CREATE INDEX idx_logs_created ON logs USING BRIN (created_at);
```

### เมื่อไหร่ควรสร้าง Index

```crystal
require "db"
require "pg"

# วิเคราะห์ slow queries ด้วย EXPLAIN ANALYZE
DB.open(ENV["DATABASE_URL"]) do |db|
  # ก่อน index
  result = db.query_all(
    "EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = $1",
    42_i64,
    as: String
  )
  puts "ก่อน index:"
  result.each { |row| puts row }

  # สร้าง index
  db.exec("CREATE INDEX IF NOT EXISTS idx_orders_user_id ON orders(user_id)")

  # หลัง index
  result = db.query_all(
    "EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = $1",
    42_i64,
    as: String
  )
  puts "\nหลัง index:"
  result.each { |row| puts row }
end
```

### Composite Index

```sql
-- ลำดับของ columns ใน composite index สำคัญมาก
-- ใช้ index ได้เมื่อ query ใช้ columns ตั้งแต่ซ้ายสุด

CREATE INDEX idx_orders_user_status ON orders(user_id, status, created_at);

-- ใช้ index ได้: user_id
-- ใช้ index ได้: user_id + status
-- ใช้ index ได้: user_id + status + created_at
-- ใช้ index ไม่ได้: status เพียงอย่างเดียว
-- ใช้ index ไม่ได้: status + created_at (ข้าม user_id)
```

```crystal
# Index advisor - วิเคราะห์ว่าควรสร้าง index อะไร
class IndexAdvisor
  def initialize(@db : DB::Database)
  end

  def analyze_missing_indexes
    # ดู slow queries จาก pg_stat_statements
    slow_queries = @db.query_all(<<-SQL,
      SELECT
        query,
        calls,
        mean_exec_time,
        total_exec_time
      FROM pg_stat_statements
      WHERE mean_exec_time > 100  -- มากกว่า 100ms
      ORDER BY total_exec_time DESC
      LIMIT 20
    SQL
      as: {query: String, calls: Int64, mean_exec_time: Float64, total_exec_time: Float64}
    )

    slow_queries.each do |q|
      puts "Query: #{q[:query]}"
      puts "Calls: #{q[:calls]}, Mean: #{q[:mean_exec_time].round(2)}ms"
      puts "---"
    end
  end

  def analyze_index_usage
    # ดู indexes ที่ไม่ค่อยได้ใช้
    unused = @db.query_all(<<-SQL,
      SELECT
        schemaname,
        tablename,
        indexname,
        idx_scan,
        idx_tup_read,
        idx_tup_fetch
      FROM pg_stat_user_indexes
      WHERE idx_scan < 100
      ORDER BY idx_scan
    SQL
      as: {schemaname: String, tablename: String, indexname: String,
           idx_scan: Int64, idx_tup_read: Int64, idx_tup_fetch: Int64}
    )

    puts "Indexes ที่แทบไม่ได้ใช้:"
    unused.each do |idx|
      puts "  #{idx[:tablename]}.#{idx[:indexname]}: #{idx[:idx_scan]} scans"
    end
  end

  def table_bloat
    # ดู tables ที่ต้องการ VACUUM
    @db.query_all(<<-SQL,
      SELECT
        schemaname,
        relname AS tablename,
        n_live_tup,
        n_dead_tup,
        last_autovacuum
      FROM pg_stat_user_tables
      WHERE n_dead_tup > 10000
      ORDER BY n_dead_tup DESC
    SQL
      as: {schemaname: String, tablename: String, n_live_tup: Int64,
           n_dead_tup: Int64, last_autovacuum: Time?}
    )
  end
end
```

## N+1 Query Problem

```crystal
require "db"
require "pg"

# N+1 Query Problem - ปัญหาที่พบบ่อยที่สุด
DB.open(ENV["DATABASE_URL"]) do |db|
  # แย่! - N+1 queries
  # 1 query หา orders
  orders = db.query_all(
    "SELECT id, user_id, total FROM orders WHERE status = $1 LIMIT $2",
    "completed", 10,
    as: {id: Int64, user_id: Int64, total: Float64}
  )

  # N queries หา users (1 per order!)
  orders.each do |order|
    user = db.query_one?(
      "SELECT email FROM users WHERE id = $1",
      order[:user_id],
      as: {email: String}
    )
    puts "Order #{order[:id]}: #{user.try(&.[:email])} - ฿#{order[:total]}"
  end
  # รวม: 1 + 10 = 11 queries!
end
```

### แก้ด้วย JOIN

```crystal
DB.open(ENV["DATABASE_URL"]) do |db|
  # ดี! - 1 query ด้วย JOIN
  results = db.query_all(<<-SQL,
    SELECT
      o.id AS order_id,
      o.total,
      u.email AS user_email,
      u.username
    FROM orders o
    INNER JOIN users u ON o.user_id = u.id
    WHERE o.status = $1
    LIMIT $2
  SQL
    "completed", 10,
    as: {order_id: Int64, total: Float64, user_email: String, username: String}
  )

  results.each do |r|
    puts "Order #{r[:order_id]}: #{r[:user_email]} - ฿#{r[:total]}"
  end
  # รวม: 1 query!
end
```

### แก้ด้วย IN Query (Batch Loading)

```crystal
DB.open(ENV["DATABASE_URL"]) do |db|
  # หา orders ก่อน
  orders = db.query_all(
    "SELECT id, user_id, total FROM orders WHERE status = $1 LIMIT $2",
    "completed", 100,
    as: {id: Int64, user_id: Int64, total: Float64}
  )

  # หา users ทั้งหมดในครั้งเดียวด้วย IN
  user_ids = orders.map(&.[:user_id]).uniq

  if user_ids.any?
    placeholders = user_ids.map_with_index { |_, i| "$#{i + 1}" }.join(", ")
    users_map = db.query_all(
      "SELECT id, email, username FROM users WHERE id IN (#{placeholders})",
      args: user_ids,
      as: {id: Int64, email: String, username: String}
    ).index_by(&.[:id])

    orders.each do |order|
      user = users_map[order[:user_id]]?
      puts "Order #{order[:id]}: #{user.try(&.[:email])} - ฿#{order[:total]}"
    end
  end
  # รวม: 2 queries เท่านั้น!
end
```

## Eager Loading

```crystal
# ระบบ Eager Loading ใน Crystal
module EagerLoader
  # โหลด associations ล่วงหน้า
  def self.load_orders_with_users(db : DB::Database, status : String, limit : Int32 = 20)
    # Step 1: โหลด orders
    orders = db.query_all(<<-SQL,
      SELECT id, user_id, total, status, created_at
      FROM orders
      WHERE status = $1
      ORDER BY created_at DESC
      LIMIT $2
    SQL
      status, limit,
      as: {id: Int64, user_id: Int64, total: Float64, status: String, created_at: Time}
    )

    return [] of typeof(orders.first) if orders.empty?

    # Step 2: Collect unique user_ids
    user_ids = orders.map(&.[:user_id]).uniq

    # Step 3: Load all users at once
    placeholders = user_ids.each_with_index.map { |_, i| "$#{i + 1}" }.join(", ")
    users = db.query_all(
      "SELECT id, email, username, first_name FROM users WHERE id IN (#{placeholders})",
      args: user_ids,
      as: {id: Int64, email: String, username: String, first_name: String?}
    ).each_with_object({} of Int64 => typeof(users.first)) do |u, hash|
      hash[u[:id]] = u
    end

    # Step 4: Combine
    orders.map do |order|
      user = users[order[:user_id]]?
      {order: order, user: user}
    end
  end

  # Recursive eager loading สำหรับ nested associations
  def self.load_posts_with_comments_and_authors(db : DB::Database, limit : Int32 = 10)
    # Load posts
    posts = db.query_all(
      "SELECT id, title, user_id FROM posts ORDER BY created_at DESC LIMIT $1",
      limit,
      as: {id: Int64, title: String, user_id: Int64}
    )
    return [] of NamedTuple if posts.empty?

    # Load authors
    author_ids = posts.map(&.[:user_id]).uniq
    ph = author_ids.map_with_index { |_, i| "$#{i+1}" }.join(", ")
    authors = db.query_all(
      "SELECT id, username FROM users WHERE id IN (#{ph})",
      args: author_ids,
      as: {id: Int64, username: String}
    ).index_by(&.[:id])

    # Load comments for all posts
    post_ids = posts.map(&.[:id])
    ph2 = post_ids.map_with_index { |_, i| "$#{i+1}" }.join(", ")
    comments = db.query_all(
      "SELECT id, post_id, content, user_id FROM comments WHERE post_id IN (#{ph2}) ORDER BY created_at",
      args: post_ids,
      as: {id: Int64, post_id: Int64, content: String, user_id: Int64}
    ).group_by(&.[:post_id])

    # Load comment authors
    comment_user_ids = comments.values.flatten.map(&.[:user_id]).uniq
    ph3 = comment_user_ids.map_with_index { |_, i| "$#{i+1}" }.join(", ")
    comment_authors = if comment_user_ids.any?
      db.query_all(
        "SELECT id, username FROM users WHERE id IN (#{ph3})",
        args: comment_user_ids,
        as: {id: Int64, username: String}
      ).index_by(&.[:id])
    else
      {} of Int64 => NamedTuple(id: Int64, username: String)
    end

    # Assemble
    posts.map do |post|
      post_comments = (comments[post[:id]]? || [] of typeof(comments.values.first.first)).map do |c|
        {comment: c, author: comment_authors[c[:user_id]]?}
      end
      {post: post, author: authors[post[:user_id]]?, comments: post_comments}
    end
  end
end
```

## Connection Pooling

```crystal
require "db"
require "pg"

# Crystal DB มี connection pooling built-in
module Database
  @@pool : DB::Database? = nil

  def self.pool : DB::Database
    @@pool ||= DB.open(connection_url, pool_settings)
  end

  private def self.connection_url
    ENV["DATABASE_URL"]? || "postgresql://localhost/myapp"
  end

  private def self.pool_settings
    DB::ConnectParams.new(
      max_pool_size:   ENV["DB_POOL_SIZE"]?.try(&.to_i) || 10,
      initial_pool_size: 2,
      max_idle_pool_size: 5,
      checkout_timeout: 10.0,
      retry_attempts: 3,
      retry_delay: 0.5
    )
  end

  def self.with_connection(&block : DB::Database -> _)
    block.call(pool)
  end

  def self.query_one?(sql : String, *args)
    pool.query_one?(sql, *args)
  end

  def self.query_all(sql : String, *args, as type)
    pool.query_all(sql, *args, as: type)
  end

  def self.exec(sql : String, *args)
    pool.exec(sql, *args)
  end

  def self.stats
    {
      size:      pool.pool_size,
      available: pool.pool_available,
    }
  end
end

# ใช้งาน connection pooling
users = Database.query_all(
  "SELECT id, email FROM users WHERE active = $1 LIMIT $2",
  true, 10,
  as: {id: Int64, email: String}
)

# ใช้หลาย fibers พร้อมกัน
channel = Channel(Array(String)).new

5.times do |i|
  spawn do
    emails = Database.query_all(
      "SELECT email FROM users LIMIT $1",
      10,
      as: String
    )
    channel.send(emails)
  end
end

5.times do
  emails = channel.receive
  puts "Got #{emails.size} emails"
end

puts "Pool stats: #{Database.stats}"
```

### Connection Pool Monitor

```crystal
class ConnectionPoolMonitor
  def initialize(@db : DB::Database)
  end

  def monitor
    loop do
      check_pool_health
      sleep 60
    end
  end

  private def check_pool_health
    stats = {
      pool_size:      @db.pool_size,
      pool_available: @db.pool_available,
      in_use:         @db.pool_size - @db.pool_available,
    }

    utilization = (stats[:in_use].to_f / stats[:pool_size] * 100).round(1)

    if utilization > 80
      Log.warn { "Connection pool: #{utilization}% utilized" }
    end

    # ตรวจสอบ long-running connections
    @db.query_all(<<-SQL,
      SELECT
        pid,
        now() - pg_stat_activity.query_start AS duration,
        query,
        state
      FROM pg_stat_activity
      WHERE (now() - pg_stat_activity.query_start) > interval '5 minutes'
        AND state != 'idle'
    SQL
      as: {pid: Int32, duration: PG::Interval?, query: String, state: String?}
    ).each do |conn|
      Log.warn { "Long-running query (pid: #{conn[:pid]}): #{conn[:query][0..100]}" }
    end
  end
end
```

## Query Analysis

```crystal
require "db"
require "pg"

class QueryAnalyzer
  def initialize(@db : DB::Database)
  end

  # วิเคราะห์ query plan
  def explain(sql : String, *params)
    rows = @db.query_all(
      "EXPLAIN ANALYZE #{sql}",
      *params,
      as: String
    )
    rows.each { |row| puts row }
  end

  # หา Sequential Scans (ไม่ใช้ index)
  def find_seq_scans
    @db.query_all(<<-SQL,
      SELECT
        schemaname,
        relname AS tablename,
        seq_scan,
        seq_tup_read,
        idx_scan,
        seq_scan - idx_scan AS more_seq_than_idx
      FROM pg_stat_user_tables
      WHERE seq_scan > idx_scan
        AND n_live_tup > 1000
      ORDER BY more_seq_than_idx DESC
    SQL
      as: {schemaname: String, tablename: String, seq_scan: Int64,
           seq_tup_read: Int64, idx_scan: Int64, more_seq_than_idx: Int64}
    )
  end

  # หา slow queries (ต้องเปิด pg_stat_statements)
  def slow_queries(threshold_ms : Float64 = 100.0)
    @db.query_all(<<-SQL,
      SELECT
        substring(query, 1, 200) AS short_query,
        calls,
        round(mean_exec_time::numeric, 2) AS avg_ms,
        round(total_exec_time::numeric, 2) AS total_ms,
        round(stddev_exec_time::numeric, 2) AS stddev_ms
      FROM pg_stat_statements
      WHERE mean_exec_time > $1
      ORDER BY mean_exec_time DESC
      LIMIT 20
    SQL
      threshold_ms,
      as: {short_query: String, calls: Int64, avg_ms: Float64,
           total_ms: Float64, stddev_ms: Float64}
    )
  end

  # ประสิทธิภาพของ indexes
  def index_efficiency
    @db.query_all(<<-SQL,
      SELECT
        t.schemaname,
        t.relname AS table_name,
        ix.relname AS index_name,
        ROUND(
          100.0 * idx_scan / NULLIF(idx_scan + seq_scan, 0), 2
        ) AS index_use_pct
      FROM
        pg_stat_user_tables t
        JOIN pg_stat_user_indexes i USING (relid)
        JOIN pg_index pi ON i.indexrelid = pi.indexrelid
        JOIN pg_class ix ON ix.oid = i.indexrelid
      WHERE NOT pi.indisunique
      ORDER BY index_use_pct NULLS LAST
    SQL
      as: {schemaname: String, table_name: String, index_name: String, index_use_pct: Float64?}
    )
  end

  # วิเคราะห์ cache hit ratio
  def cache_hit_ratio
    @db.query_one(<<-SQL,
      SELECT
        sum(heap_blks_read) AS heap_read,
        sum(heap_blks_hit) AS heap_hit,
        ROUND(
          100.0 * sum(heap_blks_hit) /
          NULLIF(sum(heap_blks_hit) + sum(heap_blks_read), 0),
          2
        ) AS ratio
      FROM pg_statio_user_tables
    SQL
      as: {heap_read: Int64?, heap_hit: Int64?, ratio: Float64?}
    )
  end
end

# ใช้งาน
DB.open(ENV["DATABASE_URL"]) do |db|
  analyzer = QueryAnalyzer.new(db)

  # ดู execution plan
  analyzer.explain(
    "SELECT u.*, COUNT(o.id) FROM users u LEFT JOIN orders o ON u.id = o.user_id GROUP BY u.id",
  )

  # หา tables ที่ไม่ใช้ index
  puts "\n=== Sequential Scans ==="
  analyzer.find_seq_scans.each do |t|
    puts "#{t[:tablename]}: #{t[:seq_scan]} seq scans vs #{t[:idx_scan]} idx scans"
  end

  # Cache hit ratio
  ratio = analyzer.cache_hit_ratio
  puts "\nCache hit ratio: #{ratio[:ratio]}%"
  puts "(ควรมากกว่า 99%)"
end
```

## Caching Strategies

```crystal
require "redis"

class CachedRepository
  def initialize(@db : DB::Database, @cache : Redis::Client)
    @ttl = 300  # 5 minutes
  end

  def find_user(id : Int64)
    key = "user:#{id}"

    # ลองหาจาก cache ก่อน
    if cached = @cache.get(key)
      return JSON.parse(cached)
    end

    # ไม่มีใน cache ดึงจาก DB
    user = @db.query_one?(
      "SELECT id, email, username, role FROM users WHERE id = $1",
      id,
      as: {id: Int64, email: String, username: String, role: String}
    )

    if user
      # เก็บใน cache
      @cache.setex(key, @ttl, user.to_json)
    end

    user
  end

  def find_products(category : String)
    key = "products:#{category}"

    if cached = @cache.get(key)
      return Array(NamedTuple).from_json(cached)
    end

    products = @db.query_all(
      "SELECT id, name, price FROM products WHERE category = $1 AND active = true",
      category,
      as: {id: Int64, name: String, price: Float64}
    )

    @cache.setex(key, @ttl, products.to_json)
    products
  end

  def invalidate_user(id : Int64)
    @cache.del("user:#{id}")
  end

  def invalidate_category(category : String)
    @cache.del("products:#{category}")
  end

  # Cache-aside with automatic refresh
  def cached(key : String, ttl : Int32 = 300, &block : -> String) : String
    cached_value = @cache.get(key)
    return cached_value if cached_value

    fresh_value = block.call
    @cache.setex(key, ttl, fresh_value)
    fresh_value
  end
end
```

## Batch Processing

```crystal
require "db"

class BatchProcessor
  def initialize(@db : DB::Database, @batch_size : Int32 = 1000)
  end

  # ประมวลผล records ทีละ batch
  def process_in_batches(table : String, &block : Array(Hash(String, DB::Any)) -> Nil)
    offset = 0

    loop do
      rows = @db.query_all(
        "SELECT * FROM #{table} ORDER BY id LIMIT $1 OFFSET $2",
        @batch_size, offset
      )

      break if rows.empty?

      block.call(rows)
      offset += @batch_size

      puts "Processed #{offset} rows..."
    end
  end

  # Bulk update
  def bulk_update(table : String, updates : Array({id: Int64, data: Hash(String, String)}))
    return if updates.empty?

    updates.each_slice(@batch_size) do |batch|
      params = [] of DB::Any
      param_idx = 0

      # Build CASE expression for bulk update
      id_list = batch.map do |item|
        param_idx += 1
        params << item[:id]
        "$#{param_idx}"
      end

      sql = "UPDATE #{table} SET updated_at = NOW() WHERE id IN (#{id_list.join(", ")})"
      @db.exec(sql, args: params)
    end
  end

  # Bulk insert with conflict handling
  def bulk_upsert(table : String, rows : Array(Hash(String, DB::Any)), conflict_key : String)
    rows.each_slice(@batch_size) do |batch|
      columns = batch.first.keys
      params = [] of DB::Any
      param_idx = 0

      value_groups = batch.map do |row|
        placeholders = columns.map do |_|
          param_idx += 1
          "$#{param_idx}"
        end
        row.each_value { |v| params << v }
        "(#{placeholders.join(", ")})"
      end

      update_sets = columns
        .reject { |c| c == conflict_key }
        .map { |c| "#{c} = EXCLUDED.#{c}" }

      sql = <<-SQL
        INSERT INTO #{table} (#{columns.join(", ")})
        VALUES #{value_groups.join(", ")}
        ON CONFLICT (#{conflict_key})
        DO UPDATE SET #{update_sets.join(", ")}
      SQL

      @db.exec(sql, args: params)
    end
  end
end
```

## Partitioning

```sql
-- Table Partitioning สำหรับ large datasets
-- ตัวอย่าง: partition orders ตามปี

CREATE TABLE orders (
  id BIGSERIAL,
  user_id BIGINT NOT NULL,
  total DECIMAL(10,2),
  status VARCHAR(50),
  created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
) PARTITION BY RANGE (created_at);

-- สร้าง partitions
CREATE TABLE orders_2024 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');

CREATE TABLE orders_2025 PARTITION OF orders
  FOR VALUES FROM ('2025-01-01') TO ('2026-01-01');

CREATE TABLE orders_2026 PARTITION OF orders
  FOR VALUES FROM ('2026-01-01') TO ('2027-01-01');

-- Index บน partition
CREATE INDEX ON orders_2024(user_id);
CREATE INDEX ON orders_2024(status);
CREATE INDEX ON orders_2025(user_id);
CREATE INDEX ON orders_2025(status);
```

## แบบฝึกหัด

1. วิเคราะห์ application ที่มี N+1 query problems แล้วแก้ด้วย JOIN หรือ batch loading
2. สร้าง connection pool monitor ที่ส่ง alert เมื่อ utilization สูงเกิน 80%
3. เปรียบเทียบ query performance ก่อนและหลังสร้าง index ด้วย EXPLAIN ANALYZE
4. สร้าง caching layer สำหรับ product catalog ที่ invalidate cache เมื่อข้อมูลเปลี่ยน

## สรุป

Database Optimization ใน Crystal:
- **Indexes**: สร้างให้เหมาะสม ไม่มากเกินไป composite index ต้องระวังลำดับ
- **N+1 Queries**: แก้ด้วย JOIN หรือ batch loading (2 queries แทน N+1)
- **Eager Loading**: โหลด associations ล่วงหน้าในขั้นตอนเดียว
- **Connection Pooling**: Crystal DB มี built-in pool, ตั้งค่าให้เหมาะกับ load
- **Query Analysis**: EXPLAIN ANALYZE, pg_stat_statements, cache hit ratio
- **Caching**: Redis สำหรับ frequently accessed data
- **Batch Processing**: ประมวลผลข้อมูลจำนวนมากทีละ chunk

การ optimize database ต้องวัดผลก่อนและหลังเสมอ อย่า optimize โดยไม่มีข้อมูล
