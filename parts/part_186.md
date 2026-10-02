# Part 186: Shards - Package Manager ใน Crystal

## บทนำ

Shards คือ package manager ของ Crystal ใช้สำหรับจัดการ dependencies ของ Crystal projects โดยใช้ `shard.yml` เป็น configuration file

## shard.yml Structure

```yaml
# shard.yml - ไฟล์หลักสำหรับ Crystal project
name: my_awesome_app
version: 1.0.0

description: |
  A Crystal application for doing awesome things

authors:
  - Your Name <you@example.com>

license: MIT

crystal: ">= 1.0.0"

# Runtime dependencies
dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4.0"

  db:
    github: crystal-lang/crystal-db
    version: "~> 0.13.0"

  pg:
    github: will/crystal-pg
    version: "~> 0.28.0"

  redis:
    github: stefanwille/crystal-redis
    version: "~> 2.8.0"

  jwt:
    github: crystal-community/jwt
    version: "~> 1.6.0"

  dotenv:
    github: gdotdesign/cr-dotenv
    version: "~> 1.0.0"

# Development-only dependencies
development_dependencies:
  webmock:
    github: manastech/webmock.cr
    version: "~> 0.3.0"

  timecop:
    github: waterlink/timecop.cr
    version: "~> 0.0.1"

  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5.0"
```

## Semantic Versioning

```yaml
# Version constraints ใน shard.yml

# Exact version
version: "= 1.0.0"

# Minimum version (>= จะ compatible กับ major เดิม)
version: ">= 1.0.0"

# Pessimistic constraint (~>): ใช้บ่อยที่สุด
# ~> 1.4.0 หมายถึง >= 1.4.0, < 1.5.0
# ~> 1.4   หมายถึง >= 1.4.0, < 2.0.0
version: "~> 1.4.0"

# Less than
version: "< 2.0.0"

# Range
version: ">= 1.0.0, < 2.0.0"

# Any version (ไม่แนะนำ)
# ไม่ระบุ version หมายถึง latest

# Git ref
dependencies:
  my_lib:
    github: user/my_lib
    branch: main        # หรือ specific branch

  another_lib:
    github: user/lib
    tag: v2.0.0         # git tag

  dev_lib:
    github: user/lib
    commit: abc123def   # specific commit
```

## shards install และ update

```bash
# ติดตั้ง dependencies
shards install

# อัปเดต dependencies (เคารพ version constraints)
shards update

# อัปเดต specific shard
shards update kemal

# ดู installed shards
shards list

# ตรวจสอบ outdated shards
shards outdated

# ดู info ของ specific shard
shards info kemal

# สร้าง shard.lock จาก shard.yml
shards install --frozen  # ไม่อัปเดต lock file

# Install ด้วย lock file (reproducible builds)
shards install --frozen

# ดู dependencies tree
shards tree
```

## shard.lock

```yaml
# shard.lock - สร้างอัตโนมัติ, commit เข้า git เสมอ
version: 2.0
shards:
  db:
    git: https://github.com/crystal-lang/crystal-db.git
    version: 0.13.0

  kemal:
    git: https://github.com/kemalcr/kemal.git
    version: 1.4.0

  pg:
    git: https://github.com/will/crystal-pg.git
    version: 0.28.0
```

## Path Dependencies

```yaml
# ใช้ local path สำหรับ development
dependencies:
  my_local_lib:
    path: ../my_local_lib

  # Shared library ใน monorepo
  shared:
    path: ../../shared
```

## Conditional Dependencies

```crystal
# shard.yml ไม่รองรับ conditional dependencies โดยตรง
# แต่สามารถใช้ Crystal macros

{% if flag?(:linux) %}
  require "platform_linux"
{% elsif flag?(:darwin) %}
  require "platform_macos"
{% end %}

# หรือใช้ optional เพื่อ skip missing shards
{% begin %}
  require "optional_shard"
{% rescue %}
  # shard ไม่มี, ใช้ fallback
  module OptionalShard
    def self.feature; nil; end
  end
{% end %}
```

## Custom Task ใน shard.yml

```yaml
# shard.yml tasks (ไม่ค่อยใช้ แต่มีได้)
scripts:
  postinstall: make build
  preinstall: which gcc
```

## ตัวอย่าง project จริง

```yaml
# shard.yml สำหรับ web API project
name: task_api
version: 0.1.0

description: Task management REST API

authors:
  - Developer <dev@company.com>

license: MIT

crystal: ">= 1.10.0"

dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4.0"

  db:
    github: crystal-lang/crystal-db
    version: "~> 0.13.0"

  pg:
    github: will/crystal-pg
    version: "~> 0.28.0"

  redis:
    github: stefanwille/crystal-redis
    version: "~> 2.8.0"

  json_mapping:
    github: crystal-lang/json
    version: "~> 0.9.0"

  jwt:
    github: crystal-community/jwt
    version: "~> 1.6.0"

  dotenv:
    github: gdotdesign/cr-dotenv
    version: "~> 1.0.0"

  cryolog:
    github: Sija/cryolog.cr
    version: "~> 0.1.0"

development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5.0"

  webmock:
    github: manastech/webmock.cr
    version: "~> 0.3.0"
```

## shard.yml สำหรับ Library

```yaml
# shard.yml สำหรับ reusable library
name: crystal_cache
version: 2.1.0

description: |
  Thread-safe in-memory cache for Crystal applications
  Supports TTL, LRU eviction, and statistics

authors:
  - Author Name <author@example.com>

homepage: https://github.com/author/crystal_cache
repository: https://github.com/author/crystal_cache
documentation: https://author.github.io/crystal_cache

license: MIT

crystal: ">= 1.0.0"

dependencies:
  # ไม่มี dependencies จะดีที่สุดสำหรับ libraries
  # ถ้าต้องการ ใช้ version ranges กว้างๆ

development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5"
```

## แบบฝึกหัด

1. สร้าง `shard.yml` สำหรับ web scraper project ที่ใช้ HTTP client และ HTML parser
2. ทดลอง path dependency โดยสร้าง local shard แล้วใช้ใน project หลัก
3. เปรียบเทียบ `shards install` กับ `shards install --frozen` สำหรับ reproducible builds
4. สร้าง Makefile ที่ integrate กับ shards commands

## สรุป

Shards Package Manager:
- **shard.yml**: ไฟล์ configuration หลักของ Crystal project
- **shard.lock**: lock versions สำหรับ reproducible builds, commit เข้า git
- **dependencies**: runtime dependencies
- **development_dependencies**: ใช้แค่ dev/test
- **~> 1.4.0**: pessimistic constraint (>= 1.4.0, < 1.5.0)
- **shards install**: ติดตั้งตาม constraints
- **shards update**: อัปเดตตาม constraints
- **path**: local dependencies สำหรับ development/monorepo
- **github/git/path**: sources ของ dependencies
