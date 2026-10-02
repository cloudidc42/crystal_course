# Part 177: Shared Libraries ใน Crystal

## บทนำ

Shared Libraries (.so บน Linux, .dylib บน macOS, .dll บน Windows) เป็น libraries ที่ loaded ตอน runtime ทำให้หลาย programs ใช้ code เดียวกันได้ ลด binary size และ update library ได้โดยไม่ต้อง recompile

## Shared Library พื้นฐาน

### สร้าง Shared Library ใน C

```c
/* mylib.c */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

/* Export ฟังก์ชัน */
__attribute__((visibility("default")))
int add(int a, int b) {
    return a + b;
}

__attribute__((visibility("default")))
double multiply(double a, double b) {
    return a * b;
}

__attribute__((visibility("default")))
char* greet(const char *name) {
    static char buffer[256];
    snprintf(buffer, sizeof(buffer), "Hello, %s!", name);
    return buffer;
}

/* Constructor/Destructor */
__attribute__((constructor))
void library_init(void) {
    printf("Library initialized\n");
}

__attribute__((destructor))
void library_cleanup(void) {
    printf("Library cleaned up\n");
}
```

```bash
# Compile เป็น shared library
gcc -shared -fPIC -O2 mylib.c -o libmylib.so

# macOS
gcc -shared -fPIC -dynamiclib -O2 mylib.c -o libmylib.dylib

# ดู exported symbols
nm -D libmylib.so | grep " T "
# หรือ
readelf --symbols libmylib.so | grep FUNC
```

### Linking กับ Shared Library

```crystal
# Link กับ .so/.dylib
@[Link("mylib")]
@[Link(ldflags: "-L./lib -Wl,-rpath,./lib")]
lib LibMyLib
  fun add(a : Int32, b : Int32) : Int32
  fun multiply(a : Float64, b : Float64) : Float64
  fun greet(name : UInt8*) : UInt8*
end

puts LibMyLib.add(3, 4)          # 7
puts LibMyLib.multiply(2.5, 4.0) # 10.0

greeting = LibMyLib.greet("Crystal")
puts String.new(greeting)  # Hello, Crystal!
```

## Dynamic Loading ด้วย libdl

```crystal
# Dynamic loading - โหลด library ตอน runtime
lib LibDl
  RTLD_LAZY   = 1
  RTLD_NOW    = 2
  RTLD_GLOBAL = 256
  RTLD_LOCAL  = 0

  fun dlopen(filename : UInt8*, flag : Int32) : Void*
  fun dlclose(handle : Void*) : Int32
  fun dlsym(handle : Void*, symbol : UInt8*) : Void*
  fun dlerror() : UInt8*
end

class DynamicLibrary
  class Error < Exception; end

  def initialize(path : String)
    @handle = LibDl.dlopen(path, LibDl::RTLD_LAZY)
    if @handle.null?
      err = LibDl.dlerror
      raise Error.new("Cannot load #{path}: #{err ? String.new(err) : "unknown"}")
    end
    @path = path
  end

  def close
    LibDl.dlclose(@handle)
  end

  def symbol(name : String) : Void*
    ptr = LibDl.dlsym(@handle, name)
    if ptr.null?
      err = LibDl.dlerror
      raise Error.new("Symbol '#{name}' not found: #{err ? String.new(err) : "unknown"}")
    end
    ptr
  end

  # Get typed function pointer
  def func(name : String, type : T.class) : T forall T
    ptr = symbol(name)
    ptr.as(T)
  end

  def finalize
    close unless @handle.null?
  end
end

# ใช้งาน dynamic loading
lib = DynamicLibrary.new("./lib/libmylib.so")

# โหลด function pointer
add_fn = lib.func("add", Proc(Int32, Int32, Int32))
puts add_fn.call(10, 20)  # 30

# โหลด another function
greet_fn = lib.func("greet", Proc(UInt8*, UInt8*))
result = greet_fn.call("Dynamic")
puts String.new(result)

lib.close
```

## Plugin System ด้วย Shared Libraries

```crystal
# ระบบ plugin ที่ load dynamic libraries
module Plugin
  # Interface ที่ทุก plugin ต้องใช้
  abstract class Base
    abstract def name : String
    abstract def version : String
    abstract def execute(input : String) : String
  end
end

# สร้าง plugin factory
alias PluginFactoryFn = -> Plugin::Base

class PluginLoader
  @plugins : Hash(String, DynamicLibrary) = {} of String => DynamicLibrary
  @instances : Hash(String, Plugin::Base) = {} of String => Plugin::Base

  def load(path : String) : Plugin::Base
    lib = DynamicLibrary.new(path)
    @plugins[path] = lib

    # เรียก factory function จาก shared library
    factory = lib.func("create_plugin", PluginFactoryFn)
    plugin = factory.call

    @instances[path] = plugin
    puts "Loaded plugin: #{plugin.name} v#{plugin.version}"
    plugin
  end

  def unload(path : String)
    @instances.delete(path)
    if lib = @plugins.delete(path)
      lib.close
    end
  end

  def get(path : String) : Plugin::Base?
    @instances[path]?
  end
end

loader = PluginLoader.new
plugin = loader.load("./plugins/processor.so")
result = plugin.execute("hello world")
puts "Plugin result: #{result}"
```

## macOS Framework Linking

```crystal
# macOS Frameworks
@[Link(framework: "Foundation")]
lib LibFoundation
  # NSString equivalents...
end

@[Link(framework: "CoreFoundation")]
lib LibCoreFoundation
  CFStringRef = Void
  CFAllocatorRef = Void

  CFAllocatorDefault = Pointer(Void).null

  fun CFStringCreateWithCString(
    alloc : CFAllocatorRef*,
    cStr : UInt8*,
    encoding : UInt32
  ) : CFStringRef*

  fun CFStringGetLength(theString : CFStringRef*) : Int64
  fun CFStringGetCStringPtr(theString : CFStringRef*, encoding : UInt32) : UInt8*
  fun CFRelease(cf : Void*) : Void
end

@[Link(framework: "IOKit")]
lib LibIOKit
  # IOKit functions สำหรับ hardware access
end

@[Link(framework: "Security")]
lib LibSecurity
  # Keychain access, code signing, etc.
end
```

### macOS System Information

```crystal
@[Link(framework: "IOKit")]
@[Link(framework: "CoreFoundation")]
lib LibMacInfo
  IOOptionBits = UInt32
  io_iterator_t = UInt32
  io_object_t = UInt32
  io_service_t = UInt32

  kIOReturnSuccess = 0

  fun IOServiceGetMatchingServices(
    mainPort : UInt32,
    matching : Void*,
    existing : io_iterator_t*
  ) : Int32

  fun IOIteratorNext(iterator : io_iterator_t) : io_object_t
  fun IOObjectRelease(object : io_object_t) : Int32
end

# ดู hardware info บน macOS
{% if flag?(:darwin) %}
  def mac_serial_number : String?
    # ใช้ IOKit เพื่อดู serial number
    # (simplified example)
    cmd_output = `ioreg -l | grep IOPlatformSerialNumber`
    match = cmd_output.match(/\"(.+)\"$/)
    match.try(&.[1])
  end

  puts mac_serial_number || "Unknown"
{% end %}
```

## Shared Library Versioning

```bash
# สร้าง versioned shared library
gcc -shared -fPIC -Wl,-soname,libmylib.so.1 \
    -o libmylib.so.1.0.0 mylib.c

# สร้าง symlinks
ln -s libmylib.so.1.0.0 libmylib.so.1
ln -s libmylib.so.1 libmylib.so

# อัปเดต ldconfig
ldconfig

# ดู library dependencies
ldd mybinary
```

```crystal
# Crystal link กับ versioned library
@[Link("mylib")]  # ใช้ libmylib.so.1 (latest)
lib LibMyLib
  fun version() : UInt8*
end

version = LibMyLib.version
puts "Library version: #{String.new(version)}"
```

## Build Script สำหรับ Shared Library

```makefile
# Makefile
CC = gcc
CFLAGS = -Wall -O2 -fPIC
LDFLAGS = -shared -Wl,-soname,$(LIB_SONAME)

LIB_NAME = mylib
LIB_VERSION = 1.0.0
LIB_SONAME = lib$(LIB_NAME).so.1
LIB_REALNAME = lib$(LIB_NAME).so.$(LIB_VERSION)

SRCS = $(wildcard src/*.c)
OBJS = $(SRCS:src/%.c=build/%.o)

.PHONY: all clean install

all: lib/$(LIB_REALNAME)

build/%.o: src/%.c
	mkdir -p build
	$(CC) $(CFLAGS) -c $< -o $@

lib/$(LIB_REALNAME): $(OBJS)
	mkdir -p lib
	$(CC) $(LDFLAGS) -o $@ $^
	ln -sf $(LIB_REALNAME) lib/$(LIB_SONAME)
	ln -sf $(LIB_SONAME) lib/lib$(LIB_NAME).so

clean:
	rm -rf build lib

install: all
	install -d $(DESTDIR)/usr/lib
	install -m 755 lib/$(LIB_REALNAME) $(DESTDIR)/usr/lib/
	ldconfig
```

## แบบฝึกหัด

1. สร้าง plugin system ที่ load/unload plugins แบบ hot-reload โดยไม่ต้อง restart application
2. สร้าง shared library ที่มี C API สำหรับ Calculator แล้วเรียกใช้จาก Crystal
3. เปรียบเทียบ performance ระหว่าง static linking และ dynamic linking
4. สร้าง version-aware loader ที่ load library version ที่ดีที่สุดที่มีอยู่

## สรุป

Shared Libraries ใน Crystal:
- **@[Link("name")]**: link กับ shared library ปกติ
- **@[Link(framework: "Name")]**: macOS-specific framework
- **-Wl,-rpath**: ระบุ runtime library path
- **dlopen/dlsym**: dynamic loading ตอน runtime
- **Plugin systems**: load/unload modules แบบ dynamic
- **Versioning**: soname, symlinks สำหรับ ABI compatibility

Shared libraries เหมาะสำหรับ plugin architectures และ systems ที่ต้องการ update library โดยไม่ recompile application
