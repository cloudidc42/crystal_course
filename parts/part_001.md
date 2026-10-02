# Part 001: แนะนำ Crystal Programming Language

## Crystal คืออะไร?

Crystal เป็นภาษาโปรแกรมสมัยใหม่ที่พัฒนาโดย Manas Technology Solutions เริ่มต้นในปี 2011 โดย Ary Borenszweig และทีมงาน โดยมีเป้าหมายเพื่อสร้างภาษาที่:

- **มีไวยากรณ์คล้าย Ruby** - เขียนง่าย อ่านง่าย
- **ทำงานเร็วเหมือน C** - compile เป็น native machine code
- **Type-safe** - ตรวจสอบ type errors ตั้งแต่ compile time
- **Null-safe** - ป้องกัน NullPointerException ด้วยระบบ type
- **Concurrent** - รองรับ concurrent programming ด้วย Fibers

---

## ประวัติและพัฒนาการ

```
2011 - Ary Borenszweig เริ่มพัฒนา Crystal
2014 - Crystal เปิดตัว Alpha version
2016 - Crystal 0.18.0 - เสถียรภาพเพิ่มขึ้นมาก
2019 - Crystal 0.31.0 - Multi-threading support
2021 - Crystal 1.0.0 - เวอร์ชันเสถียรแรก
2022 - Crystal 1.4.0 - พัฒนา performance และ stdlib
2023 - Crystal 1.10.0 - ปรับปรุง concurrency
2024 - Crystal 1.13+ - ยังพัฒนาต่อเนื่อง
```

---

## ทำไมต้องเรียน Crystal?

### 1. ความเร็ว (Speed)
Crystal ถูก compile เป็น native binary ด้วย LLVM ทำให้ทำงานได้เร็วมาก เปรียบเทียบ:

| ภาษา | ความเร็วสัมพัทธ์ |
|------|-----------------|
| C | 1x (baseline) |
| Crystal | ~1.2x |
| Go | ~1.5x |
| Java | ~2x |
| Python | ~50x ช้ากว่า C |
| Ruby | ~100x ช้ากว่า C |

### 2. ไวยากรณ์ที่สวยงาม (Beautiful Syntax)
```crystal
# Crystal
names = ["Alice", "Bob", "Charlie"]
names.each { |name| puts "Hello, #{name}!" }

# เปรียบเทียบกับ Java
// Java
String[] names = {"Alice", "Bob", "Charlie"};
for (String name : names) {
    System.out.println("Hello, " + name + "!");
}
```

### 3. Type Safety โดยไม่ต้องระบุ Type ตลอด
```crystal
# Crystal อนุมาน type ให้อัตโนมัติ
name = "Alice"      # String
age = 25            # Int32
height = 1.75       # Float64
active = true       # Bool

# แต่ยังตรวจสอบ type ตั้งแต่ compile time
name + 1  # Error: no overload matches 'String#+' with type Int32
```

### 4. Null Safety
```crystal
# ต้องประกาศชัดเจนว่าค่าอาจเป็น nil
name : String | Nil = nil
name : String? = nil     # เหมือนกัน แต่สั้นกว่า

# ต้องตรวจสอบก่อนใช้
if name
  puts name.upcase   # ปลอดภัย - รู้ว่าไม่ nil
end
```

---

## การเปรียบเทียบกับภาษาอื่น

### Crystal vs Ruby
| คุณสมบัติ | Crystal | Ruby |
|-----------|---------|------|
| ความเร็ว | ~100x เร็วกว่า | baseline |
| Type System | Static (inferred) | Dynamic |
| Null Safety | มี | ไม่มี |
| Compile | ใช่ | ไม่ใช่ (interpreted) |
| Syntax | คล้ายกันมาก | ต้นแบบ |
| Ecosystem | เล็กกว่า | ใหญ่กว่า |

### Crystal vs Go
| คุณสมบัติ | Crystal | Go |
|-----------|---------|-----|
| Syntax | Ruby-like | C-like |
| Generics | มี (ดีกว่า) | 1.18+ |
| Macros | มี | ไม่มี |
| Concurrency | Fibers/CSP | Goroutines/Channels |
| Performance | คล้ายกัน | คล้ายกัน |

### Crystal vs Rust
| คุณสมบัติ | Crystal | Rust |
|-----------|---------|------|
| Learning Curve | ง่ายกว่ามาก | ยากมาก |
| Memory Safety | GC | Ownership System |
| Performance | ดี | ดีกว่านิดหน่อย |
| Syntax | สวยงาม | ซับซ้อน |
| Ecosystem | เล็ก | ใหญ่กว่า |

---

## การติดตั้ง Crystal

### macOS
```bash
# ใช้ Homebrew
brew install crystal

# ตรวจสอบเวอร์ชัน
crystal --version
```

### Ubuntu/Debian Linux
```bash
# เพิ่ม repository
curl -fsSL https://crystal-lang.org/install.sh | sudo bash

# หรือ manual
sudo apt-get install -y curl gnupg
curl -fsSL https://keybase.io/crystal/pgp_keys.asc | sudo gpg --dearmor -o /usr/share/keyrings/crystal.gpg
echo "deb [signed-by=/usr/share/keyrings/crystal.gpg] https://dist.crystal-lang.org/apt crystal main" | sudo tee /etc/apt/sources.list.d/crystal.list
sudo apt-get update
sudo apt-get install -y crystal

# ตรวจสอบ
crystal --version
```

### Fedora/CentOS/RHEL
```bash
# เพิ่ม repository
curl -fsSL https://crystal-lang.org/install.sh | sudo bash

# ตรวจสอบ
crystal --version
```

### Windows
```bash
# ใช้ WSL2 (แนะนำ)
# ติดตั้ง WSL2 ก่อน แล้วทำตาม Ubuntu instructions

# หรือใช้ Scoop
scoop install crystal

# ตรวจสอบ
crystal --version
```

### Docker
```bash
# ใช้ Docker image อย่างเป็นทางการ
docker pull crystallang/crystal:latest

# รันแบบ interactive
docker run -it crystallang/crystal:latest crystal

# หรือรัน script
docker run --rm -v $(pwd):/app -w /app crystallang/crystal:latest crystal run hello.cr
```

### Snapcraft (Linux)
```bash
sudo snap install crystal --classic
crystal --version
```

---

## เครื่องมือที่จำเป็น

### Text Editors และ IDEs

**VS Code (แนะนำ)**
```bash
# ติดตั้ง extension
# เปิด VS Code แล้วไปที่ Extensions
# ค้นหา "Crystal Language"
# ติดตั้ง "crystal-lang.crystal-lang"
```

**คุณสมบัติที่ได้จาก VS Code Extension:**
- Syntax highlighting
- Code completion (autocomplete)
- Go to definition
- Error highlighting
- Format on save

**JetBrains IDEs**
```
ติดตั้ง plugin: "Crystal"
รองรับ RubyMine, IntelliJ IDEA, CLion
```

**Vim/Neovim**
```vim
" ติดตั้ง vim-crystal plugin
Plug 'vim-crystal/vim-crystal'

" หรือใช้ LSP
Plug 'neoclide/coc.nvim'
" แล้ว :CocInstall coc-crystal
```

**Emacs**
```lisp
;; ติดตั้ง crystal-mode
(use-package crystal-mode)
```

### Crystal Language Server
```bash
# ติดตั้ง crystalline (LSP server)
# https://github.com/elbywan/crystalline
shards install  # ใน project
```

---

## โครงสร้าง Crystal Project

```
my_project/
├── shard.yml          # package manifest (เหมือน package.json)
├── shard.lock         # dependency lock file
├── src/
│   ├── my_project.cr  # main source file
│   └── models/
│       ├── user.cr
│       └── post.cr
├── spec/
│   ├── spec_helper.cr
│   └── my_project_spec.cr
├── lib/               # ไฟล์ shards (dependencies)
└── bin/               # compiled binaries
```

---

## Crystal Compiler Commands

```bash
# คอมไพล์และรันทันที
crystal run hello.cr

# คอมไพล์เป็น binary
crystal build hello.cr
./hello

# คอมไพล์แบบ optimized
crystal build --release hello.cr

# ตรวจสอบ syntax โดยไม่รัน
crystal check hello.cr

# ดู AST (Abstract Syntax Tree)
crystal tool hierarchy hello.cr

# Format code
crystal tool format hello.cr

# ดู dependencies
crystal tool dependencies hello.cr

# สร้าง documentation
crystal docs

# รัน tests
crystal spec
```

---

## ระบบ Shards (Package Manager)

```yaml
# shard.yml
name: my_app
version: 1.0.0
description: My Crystal application

authors:
  - Your Name <your@email.com>

crystal: ">= 1.0.0"

license: MIT

dependencies:
  kemal:
    github: kemalcr/kemal
    version: "~> 1.4"
  
  granite:
    github: amberframework/granite
    version: "~> 0.27"

development_dependencies:
  webmock:
    github: manastech/webmock.cr
```

```bash
# ติดตั้ง dependencies
shards install

# อัพเดท dependencies
shards update

# ดู dependencies
shards list
```

---

## โปรแกรมแรก: Hello World

```crystal
# สร้างไฟล์ hello.cr
puts "Hello, World!"
```

```bash
# รัน
crystal run hello.cr
# Output: Hello, World!
```

### Hello World แบบเต็ม
```crystal
# hello.cr

# ประกาศ module
module HelloWorld
  VERSION = "1.0.0"
  
  # Method หลัก
  def self.greet(name : String) : String
    "Hello, #{name}! Welcome to Crystal #{VERSION}"
  end
  
  # Method แบบ generic
  def self.greet_all(names : Array(String))
    names.each_with_index do |name, index|
      puts "#{index + 1}. #{greet(name)}"
    end
  end
end

# เรียกใช้
HelloWorld.greet_all(["Alice", "Bob", "Charlie"])
```

```bash
crystal run hello.cr
# Output:
# 1. Hello, Alice! Welcome to Crystal 1.0.0
# 2. Hello, Bob! Welcome to Crystal 1.0.0
# 3. Hello, Charlie! Welcome to Crystal 1.0.0
```

---

## Crystal Playground

คุณสามารถทดลอง Crystal ออนไลน์ได้ที่:
- **crystal-lang.org/play** - Official playground
- **carc.in** - Crystal online compiler

---

## คุณสมบัติพิเศษของ Crystal

### 1. Compile-time Metaprogramming
```crystal
# Macros ทำงานตอน compile time
macro define_getter(name, type)
  def {{name}} : {{type}}
    @{{name}}
  end
end

class Person
  def initialize(@name : String, @age : Int32)
  end
  
  define_getter name, String
  define_getter age, Int32
end

person = Person.new("Alice", 25)
puts person.name  # Alice
puts person.age   # 25
```

### 2. Type Inference
```crystal
# ไม่ต้องระบุ type หลายๆ ครั้ง
result = [1, 2, 3].map { |x| x * 2 }.select { |x| x > 2 }
# Crystal รู้ว่า result เป็น Array(Int32)
puts result  # [4, 6]
```

### 3. Nil Safety
```crystal
# ฟังก์ชันที่อาจ return nil
def find_user(id : Int32) : String?
  users = {"1" => "Alice", "2" => "Bob"}
  users[id.to_s]?
end

user = find_user(1)
# ต้องตรวจสอบก่อนใช้
if user
  puts user.upcase  # ALICE
end

# หรือใช้ safe navigation operator
puts user&.upcase   # ALICE หรือ nil
```

### 4. Powerful Blocks
```crystal
# Blocks คล้าย Ruby แต่ type-safe
[1, 2, 3, 4, 5]
  .select { |n| n.odd? }        # [1, 3, 5]
  .map { |n| n * n }            # [1, 9, 25]
  .reduce(0) { |sum, n| sum + n } # 35
  .tap { |result| puts result } # 35
```

### 5. Native C Bindings
```crystal
# เรียกใช้ C functions โดยตรง
@[Link("c")]
lib LibC
  fun printf(format : LibC::Char*, ...) : LibC::Int
end

LibC.printf("Hello from C! %d\n", 42)
```

---

## Crystal Standard Library Overview

Crystal มี standard library ที่ครบครัน:

```
Crystal Standard Library
├── Core Types: Int, Float, String, Bool, Char, Symbol
├── Collections: Array, Hash, Set, Tuple, NamedTuple, Deque
├── IO: File, IO, STDIN, STDOUT, STDERR
├── Networking: HTTP, Socket, DNS
├── Concurrency: Fiber, Channel, Mutex
├── Crypto: OpenSSL, Digest, Base64
├── Data Formats: JSON, YAML, CSV, XML
├── System: Process, Signal, ENV
├── Testing: Spec
└── Utilities: Time, UUID, Random, Regex
```

---

## ตัวอย่าง Real-world Crystal Code

### HTTP Server
```crystal
require "http/server"

server = HTTP::Server.new do |context|
  context.response.content_type = "text/plain"
  context.response.print "Hello world, got #{context.request.path}!"
end

address = server.bind_tcp 8080
puts "Listening on http://#{address}"
server.listen
```

### JSON API
```crystal
require "json"
require "http/server"

struct User
  include JSON::Serializable
  
  property id : Int32
  property name : String
  property email : String
  
  def initialize(@id, @name, @email)
  end
end

server = HTTP::Server.new do |context|
  case context.request.path
  when "/user"
    user = User.new(1, "Alice", "alice@example.com")
    context.response.content_type = "application/json"
    context.response.print user.to_json
  else
    context.response.status = HTTP::Status::NOT_FOUND
    context.response.print "Not Found"
  end
end

server.bind_tcp(8080)
server.listen
```

### Concurrent Programming
```crystal
channel = Channel(String).new

# Spawn fibers
5.times do |i|
  spawn do
    sleep (rand * 0.1).seconds
    channel.send "Message from fiber #{i}"
  end
end

# Receive messages
5.times do
  puts channel.receive
end
```

---

## สรุป Part 001

ในบทนี้เราได้เรียนรู้:

1. **Crystal คืออะไร** - ภาษาที่มีไวยากรณ์คล้าย Ruby แต่เร็วเหมือน C
2. **ประวัติ** - เริ่มพัฒนา 2011, เวอร์ชัน 1.0 ในปี 2021
3. **ทำไมต้องเรียน** - ความเร็ว, syntax สวย, type safety, null safety
4. **การติดตั้ง** - รองรับ macOS, Linux, Windows (WSL2)
5. **เครื่องมือ** - VS Code + Crystal extension แนะนำ
6. **โครงสร้าง Project** - src/, spec/, shard.yml
7. **Crystal Commands** - run, build, check, spec
8. **คุณสมบัติพิเศษ** - Macros, Type Inference, Nil Safety, Blocks, C Bindings

---

## แบบฝึกหัด

1. ติดตั้ง Crystal และตรวจสอบเวอร์ชัน
2. สร้างไฟล์ `hello.cr` และรัน
3. ลองใช้ Crystal playground ออนไลน์
4. ลองคอมไพล์เป็น binary และรัน

---

## ขั้นตอนต่อไป

ไปที่ [Part 002](part_002.md) เพื่อเรียนรู้:
- Hello World แบบละเอียด
- โครงสร้างพื้นฐานของ Crystal Program
- การใช้ puts, print, p
- Comments
