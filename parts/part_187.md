# Part 187: Publishing Shards ใน Crystal

## บทนำ

การ publish shard ช่วยให้ developer อื่นๆ ใช้ library ของเราได้ Crystal community มี shards.info เป็น directory หลักสำหรับค้นหา shards

## สร้าง Shard

```bash
# สร้าง shard project ใหม่
crystal init lib my_awesome_shard
cd my_awesome_shard

# โครงสร้างจะเป็น:
# my_awesome_shard/
# ├── .gitignore
# ├── README.md
# ├── LICENSE
# ├── shard.yml
# ├── src/
# │   └── my_awesome_shard.cr
# └── spec/
#     ├── my_awesome_shard_spec.cr
#     └── spec_helper.cr
```

## shard.yml สำหรับ Publishing

```yaml
# shard.yml - complete สำหรับ published shard
name: my_awesome_shard
version: 1.0.0

description: |
  A concise description (1-2 sentences) of what this shard does.
  Appears in shards.info directory listing.

authors:
  - Full Name <email@example.com>

# Homepage (usually GitHub repo)
homepage: https://github.com/username/my_awesome_shard

# Documentation URL (optional)
documentation: https://username.github.io/my_awesome_shard

license: MIT

crystal: ">= 1.0.0"

# Minimal runtime dependencies
dependencies:
  # Only add what's truly necessary

# Dev dependencies
development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5"
```

## โครงสร้าง Shard ที่ดี

```crystal
# src/my_awesome_shard.cr - main entry point
# ทำแค่ require ไฟล์อื่น

require "./my_awesome_shard/version"
require "./my_awesome_shard/cache"
require "./my_awesome_shard/store"
require "./my_awesome_shard/errors"

# src/my_awesome_shard/version.cr
module MyAwesomeShard
  VERSION = "1.0.0"
end

# src/my_awesome_shard/errors.cr
module MyAwesomeShard
  class Error < Exception; end
  class NotFoundError < Error; end
  class ValidationError < Error; end
  class TimeoutError < Error; end
end

# src/my_awesome_shard/cache.cr
module MyAwesomeShard
  # Main class ของ shard
  class Cache(K, V)
    getter hits : Int64 = 0_i64
    getter misses : Int64 = 0_i64

    def initialize(@max_size : Int32 = 1000, @ttl : Time::Span? = nil)
      @store = {} of K => {value: V, inserted_at: Time}
    end

    def get(key : K) : V?
      if entry = @store[key]?
        if ttl = @ttl
          if Time.local - entry[:inserted_at] > ttl
            @store.delete(key)
            @misses += 1
            return nil
          end
        end
        @hits += 1
        entry[:value]
      else
        @misses += 1
        nil
      end
    end

    def set(key : K, value : V) : V
      evict_if_needed
      @store[key] = {value: value, inserted_at: Time.local}
      value
    end

    def delete(key : K) : Bool
      @store.delete(key) != nil
    end

    def size : Int32
      @store.size
    end

    def clear
      @store.clear
    end

    def hit_rate : Float64
      total = @hits + @misses
      total > 0 ? @hits.to_f / total : 0.0
    end

    private def evict_if_needed
      return if @store.size < @max_size
      oldest_key = @store.min_by { |_, v| v[:inserted_at] }.first
      @store.delete(oldest_key)
    end
  end
end
```

## Documentation ด้วย Crystal Docs

```crystal
# Crystal generate docs อัตโนมัติจาก comments

module MyAwesomeShard
  # A thread-safe, TTL-aware in-memory cache.
  #
  # Supports LRU eviction when the cache reaches its maximum size.
  #
  # ## Example
  #
  # ```
  # cache = MyAwesomeShard::Cache(String, User).new(max_size: 100, ttl: 5.minutes)
  # cache.set("user:1", user)
  # cache.get("user:1")  # => user (or nil if expired)
  # ```
  class Cache(K, V)
    # Gets a value from the cache.
    #
    # Returns `nil` if the key doesn't exist or has expired.
    #
    # ```
    # cache = Cache(String, String).new
    # cache.set("key", "value")
    # cache.get("key")  # => "value"
    # cache.get("missing")  # => nil
    # ```
    def get(key : K) : V?
      # implementation
    end

    # Sets a value in the cache.
    #
    # If the cache is full, the oldest entry is evicted.
    # Returns the value that was set.
    def set(key : K, value : V) : V
      # implementation
    end

    # The current cache hit rate (0.0 to 1.0).
    #
    # Returns 0.0 if no lookups have been performed yet.
    def hit_rate : Float64
      # implementation
    end
  end
end

# Generate docs
# crystal docs
# Output: docs/ directory ที่ serve เป็น HTML
```

## GitHub Release Process

```bash
# 1. ตรวจสอบทุกอย่างก่อน release
crystal spec
crystal tool format --check
ameba

# 2. อัปเดต version
# แก้ไข shard.yml: version: "1.1.0"
# แก้ไข src/my_shard/version.cr: VERSION = "1.1.0"

# 3. Commit และ tag
git add shard.yml src/my_shard/version.cr
git commit -m "Release v1.1.0"
git tag v1.1.0
git push origin main --tags

# 4. สร้าง GitHub Release
gh release create v1.1.0 \
  --title "v1.1.0" \
  --notes "$(cat CHANGELOG.md | head -30)"

# 5. Users สามารถใช้ได้ทันที
# dependencies:
#   my_shard:
#     github: username/my_shard
#     version: "~> 1.1.0"
```

## CHANGELOG.md

```markdown
# Changelog

All notable changes to this project will be documented in this file.

## [Unreleased]

## [1.1.0] - 2025-01-15

### Added
- TTL support for cache entries
- `hit_rate` method for cache statistics
- Thread-safe operations with mutex

### Changed
- `Cache#get` now returns `nil` for expired entries (was raising error)
- Improved eviction algorithm (LRU instead of random)

### Fixed
- Fixed race condition in concurrent access

## [1.0.0] - 2025-01-01

### Added
- Initial release
- Basic get/set/delete operations
- Max size enforcement with oldest-first eviction
```

## README.md Template

```markdown
# MyAwesomeShard

[![GitHub release](https://img.shields.io/github/release/username/my_awesome_shard.svg)](https://github.com/username/my_awesome_shard/releases)
[![CI](https://github.com/username/my_awesome_shard/actions/workflows/ci.yml/badge.svg)](https://github.com/username/my_awesome_shard/actions)

A brief description of what this shard does and why someone would want to use it.

## Installation

Add the dependency to your `shard.yml`:

```yaml
dependencies:
  my_awesome_shard:
    github: username/my_awesome_shard
    version: "~> 1.0"
```

Run `shards install`.

## Usage

```crystal
require "my_awesome_shard"

cache = MyAwesomeShard::Cache(String, User).new(max_size: 100, ttl: 5.minutes)

# Set a value
cache.set("user:1", user)

# Get a value
if user = cache.get("user:1")
  puts "Found: #{user.name}"
else
  puts "Not found or expired"
end

puts "Hit rate: #{(cache.hit_rate * 100).round(1)}%"
```

## Contributing

1. Fork it (<https://github.com/username/my_awesome_shard/fork>)
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Add tests for your changes
4. Commit your changes (`git commit -am 'Add some feature'`)
5. Push to the branch (`git push origin my-new-feature`)
6. Create a new Pull Request

## License

MIT License. See [LICENSE](LICENSE) for details.
```

## แบบฝึกหัด

1. สร้าง shard ที่ implement simple rate limiter แล้ว publish ไปยัง GitHub
2. เขียน docs ครบถ้วนสำหรับ public API ของ shard
3. สร้าง CI pipeline ที่ test shard ใน multiple Crystal versions
4. เพิ่ม shard ของคุณไปยัง shards.info

## สรุป

Publishing Shards:
- **crystal init lib**: สร้าง shard project structure
- **shard.yml**: ระบุ name, version, description, authors, license
- **VERSION constant**: แยก version ออกมาใน version.cr
- **Crystal Docs**: comment ด้วย `#` สำหรับ auto-generated docs
- **GitHub tags**: ใช้ v1.0.0 format สำหรับ releases
- **CHANGELOG.md**: document changes ทุก version
- **README.md**: installation + usage + contributing guide
- **shards.info**: directory สำหรับ Crystal shards
- **Minimal dependencies**: ดีสำหรับ libraries ไม่ควรมี runtime deps เยอะ
