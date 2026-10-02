# Part 199: Crystal Ecosystem Overview

## บทนำ

Crystal มี ecosystem ที่เติบโตขึ้นเรื่อยๆ ด้วย frameworks, tools, และ community resources ที่ครบครัน บทนี้จะ overview สิ่งสำคัญที่ควรรู้

## Web Frameworks

```yaml
# Kemal: framework ที่ popular ที่สุด, Sinatra-inspired
# github: kemalcr/kemal
dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4"
```

```crystal
# Kemal example
require "kemal"

get "/hello/:name" do |env|
  name = env.params.url["name"]
  "Hello, #{name}!"
end

Kemal.run
```

```yaml
# Lucky: full-stack framework, Rails-inspired
# github: luckyframework/lucky
# มี ORM (Avram), ตรวจ type-safe, form objects, testing support

# Amber: MVC framework
# github: amberframework/amber
# มี scaffolding, middleware, pipelines

# Spider-Gazelle: high-performance, ActionController-like
# github: spider-gazelle/spider-gazelle
```

## ORM และ Database

```yaml
# crystal-db: database abstraction layer
# github: crystal-lang/crystal-db

# crystal-pg: PostgreSQL driver
# github: will/crystal-pg

# crystal-sqlite3: SQLite3 driver
# github: crystal-lang/crystal-sqlite3

# granite: ActiveRecord-like ORM
# github: amberframework/granite

# jennifer.cr: flexible ORM with query DSL
# github: imdrasil/jennifer.cr

# micrate: database migrations
# github: amberframework/micrate
```

## HTTP Clients และ Networking

```crystal
# HTTP::Client (built-in)
response = HTTP::Client.get("https://api.example.com/users")
puts response.body

# crest: rest-easy HTTP client
# github: mamantoha/crest
require "crest"
response = Crest.get("https://api.example.com/users")

# halite: HTTP client with more features
# github: icyleaf/halite
require "halite"
response = Halite.get("https://api.example.com/users",
  headers: {content_type: "application/json"},
  params: {page: 1}
)

# WebSocket (built-in)
require "http/web_socket"
ws = HTTP::WebSocket.new(URI.parse("wss://ws.example.com"))
ws.on_message { |msg| puts msg }
ws.run
```

## Background Jobs

```yaml
# mosquito: job queue สำหรับ Crystal
# github: nicholaswasher/mosquito
dependencies:
  mosquito:
    github: nicholaswasher/mosquito
    version: "~> 1.0"
```

```crystal
require "mosquito"

class WelcomeEmailJob < Mosquito::QueuedJob
  params(user_id : Int64)

  def perform
    user = User.find!(user_id)
    Mailer.send_welcome(user.email)
  end
end

# Enqueue
WelcomeEmailJob.new(user_id: 123_i64).enqueue

# Worker runner
Mosquito::Runner.start
```

## Testing Tools

```yaml
# spec (built-in): Crystal's testing framework
# ameba: linter สำหรับ Crystal
# github: crystal-ameba/ameba

# webmock: HTTP mocking
# github: manastech/webmock.cr

# timecop: freeze/travel time ใน tests
# github: waterlink/timecop.cr

# factory_bot-like: ไม่มี official แต่ทำเองได้ง่าย
```

## Popular Shards Directory

```crystal
# shards.info - directory หลัก
# https://shards.info

# หมวดหมู่ที่ popular:

# JSON/YAML parsing (built-in)
require "json"
require "yaml"

# Environment variables
require "dotenv"  # github: gdotdesign/cr-dotenv

# Caching
# github: crystal-community/lru_cache
# github: stefanwille/crystal-redis

# Authentication/Security
# github: crystal-community/jwt
# github: bcrypt/bcrypt.cr (Bcrypt hashing)

# Validation
# github: PixeLInc/validate.cr

# Config management
# github: nicholaswasher/tomlr.cr  (TOML)
# github: crystallabs/crystalconfig

# Task management
# github: nicholaswasher/sam.cr  (Rake-like)
```

## Development Tools

```bash
# Crystal compiler
crystal --version     # ดู version
crystal build         # compile
crystal run           # compile + run
crystal spec          # run tests
crystal docs          # generate docs
crystal tool format   # format code
crystal tool hierarchy  # class hierarchy
crystal tool implementations  # ดู implementations ของ method

# Shards package manager
shards install        # install dependencies
shards update         # update dependencies
shards list           # list installed shards
shards check          # verify installations

# Ameba linter
./bin/ameba           # lint codebase
./bin/ameba --all     # lint รวม autocorrectable

# Language Server Protocol (LSP)
# crystalline: LSP server สำหรับ Crystal
# github: elbywan/crystalline
# ใช้กับ VS Code, Neovim, Emacs

# VS Code Extension
# github: crystal-lang-tools/vscode-crystal-lang
```

## Community Resources

```
# Official
- https://crystal-lang.org           - Official website
- https://crystal-lang.org/docs      - Documentation
- https://crystal-lang.org/api       - API reference
- https://github.com/crystal-lang    - Source code

# Community
- https://forum.crystal-lang.org     - Official forum
- https://discord.gg/crystal-lang    - Discord server
- https://reddit.com/r/crystal_programming - Reddit
- https://fosstodon.org/@crystal_lang    - Mastodon

# Learning
- https://www.crystalforrubyists.com - Tutorial สำหรับ Ruby developers
- https://exercism.org/tracks/crystal - Exercises
- https://github.com/veelenga/awesome-crystal - Curated list

# Ecosystem
- https://shards.info                - Shard directory
- https://crystalshards.xyz          - Alternative directory
```

## ตัวอย่าง Real-world Projects

```crystal
# Amaranth: web application framework components
# Lucky Framework: full production app example
# Ambassador: Kubernetes API gateway ใช้ Crystal

# Production crystal ที่ใช้จริง:
# - 84codes (RabbitMQ cloud): Crystal microservices
# - Manas Technology: various Crystal projects
# - DeisLabs (Microsoft): Helm 3 uses Go แต่ Crystal ทำงาน alongside
```

## สร้าง Crystal Project ใหม่

```bash
# สร้าง web app
mkdir my_crystal_app
cd my_crystal_app
crystal init app .

# shard.yml
cat > shard.yml << 'EOF'
name: my_crystal_app
version: 0.1.0
description: My Crystal App

authors:
  - Your Name <you@example.com>

license: MIT
crystal: ">= 1.10.0"

dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4"

development_dependencies:
  ameba:
    github: crystal-ameba/ameba
    version: "~> 1.5"
EOF

shards install

# สร้าง basic app
cat > src/my_crystal_app.cr << 'EOF'
require "kemal"

get "/" do
  "Hello, Crystal!"
end

Kemal.run
EOF

# Run
crystal run src/my_crystal_app.cr
```

## แบบฝึกหัด

1. สำรวจ shards.info และเลือก 5 shards ที่น่าสนใจพร้อมอธิบายว่าทำอะไร
2. ลอง Lucky framework สร้าง simple CRUD application
3. เข้าร่วม Crystal Discord และถามคำถามเกี่ยวกับ project ของคุณ
4. Contribute to an open-source Crystal project (แก้ bug, เพิ่ม docs, หรือ issue)

## สรุป

Crystal Ecosystem:
- **Kemal/Lucky/Amber**: web frameworks สำหรับทุก use case
- **crystal-db + drivers**: database layer ที่ standardized
- **Granite/Jennifer**: ORM options ทั้งแบบ simple และ flexible
- **mosquito**: background job processing
- **ameba**: Crystal linter
- **crystalline**: LSP สำหรับ IDE support
- **shards.info**: discover 2000+ available shards
- **forum.crystal-lang.org**: active community สำหรับ questions
- **crystal-lang.org**: official docs ครบครัน
- Crystal ecosystem เล็กกว่า Ruby/Go แต่เติบโตเร็ว และมีคุณภาพสูง
