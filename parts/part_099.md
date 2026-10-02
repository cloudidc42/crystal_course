# Part 99: Recursive Types ใน Crystal

## บทนำ

Recursive types คือ types ที่อ้างอิงถึงตัวเองในการนิยาม Crystal รองรับ recursive types ผ่าน forward declarations และ class/struct ที่มี self-referential fields

## Forward Declarations

```crystal
# Forward declaration - ประกาศ type ก่อนนิยาม
class Node; end
class LinkedList; end

class Node
  property value : Int32
  property next : Node?

  def initialize(@value, @next = nil)
  end
end

class LinkedList
  property head : Node?

  def initialize
    @head = nil
  end

  def prepend(value : Int32)
    @head = Node.new(value, @head)
  end

  def to_array : Array(Int32)
    result = Array(Int32).new
    current = @head
    while node = current
      result << node.value
      current = node.next
    end
    result
  end
end

list = LinkedList.new
list.prepend(3)
list.prepend(2)
list.prepend(1)
puts list.to_array.inspect  # => [1, 2, 3]
```

## Self-Referential Types

```crystal
# Binary Tree
class BinaryTree(T)
  getter value : T
  property left : BinaryTree(T)?
  property right : BinaryTree(T)?

  def initialize(@value : T, @left = nil, @right = nil)
  end

  def insert(val : T)
    if val < @value
      if left = @left
        left.insert(val)
      else
        @left = BinaryTree(T).new(val)
      end
    else
      if right = @right
        right.insert(val)
      else
        @right = BinaryTree(T).new(val)
      end
    end
  end

  def inorder : Array(T)
    result = Array(T).new
    @left.try { |l| result.concat(l.inorder) }
    result << @value
    @right.try { |r| result.concat(r.inorder) }
    result
  end

  def height : Int32
    left_h = @left.try(&.height) || 0
    right_h = @right.try(&.height) || 0
    1 + [left_h, right_h].max
  end

  def size : Int32
    1 + (@left.try(&.size) || 0) + (@right.try(&.size) || 0)
  end
end

tree = BinaryTree(Int32).new(5)
[3, 7, 1, 4, 6, 8, 2].each { |v| tree.insert(v) }

puts tree.inorder.inspect  # => [1, 2, 3, 4, 5, 6, 7, 8]
puts tree.height           # => 4
puts tree.size             # => 8
```

## Doubly Linked List

```crystal
# Doubly linked list - mutual references
class DNode(T)
  property value : T
  property next : DNode(T)?
  property prev : DNode(T)?

  def initialize(@value : T, @next = nil, @prev = nil)
  end
end

class DoublyLinkedList(T)
  property head : DNode(T)?
  property tail : DNode(T)?
  getter size : Int32

  def initialize
    @head = nil
    @tail = nil
    @size = 0
  end

  def push_back(value : T)
    node = DNode(T).new(value)
    if tail = @tail
      tail.next = node
      node.prev = tail
      @tail = node
    else
      @head = node
      @tail = node
    end
    @size += 1
  end

  def push_front(value : T)
    node = DNode(T).new(value)
    if head = @head
      node.next = head
      head.prev = node
      @head = node
    else
      @head = node
      @tail = node
    end
    @size += 1
  end

  def pop_back : T?
    if tail = @tail
      value = tail.value
      @tail = tail.prev
      if new_tail = @tail
        new_tail.next = nil
      else
        @head = nil
      end
      @size -= 1
      value
    end
  end

  def to_array : Array(T)
    result = Array(T).new
    current = @head
    while node = current
      result << node.value
      current = node.next
    end
    result
  end

  def to_array_reverse : Array(T)
    result = Array(T).new
    current = @tail
    while node = current
      result << node.value
      current = node.prev
    end
    result
  end
end

dll = DoublyLinkedList(String).new
dll.push_back("B")
dll.push_back("C")
dll.push_front("A")
dll.push_back("D")

puts dll.to_array.inspect          # => ["A", "B", "C", "D"]
puts dll.to_array_reverse.inspect  # => ["D", "C", "B", "A"]
puts dll.pop_back.inspect          # => "D"
puts dll.to_array.inspect          # => ["A", "B", "C"]
```

## Tree Nodes สำหรับ AST

```crystal
# Abstract Syntax Tree node
abstract class Expr
  abstract def evaluate : Float64
  abstract def to_s : String
end

class NumberExpr < Expr
  def initialize(@value : Float64)
  end

  def evaluate : Float64
    @value
  end

  def to_s : String
    @value.to_s
  end
end

class BinaryOp < Expr
  def initialize(@left : Expr, @op : String, @right : Expr)
  end

  def evaluate : Float64
    l = @left.evaluate
    r = @right.evaluate
    case @op
    when "+" then l + r
    when "-" then l - r
    when "*" then l * r
    when "/" then
      raise "Division by zero" if r == 0.0
      l / r
    else raise "Unknown op: #{@op}"
    end
  end

  def to_s : String
    "(#{@left} #{@op} #{@right})"
  end
end

class UnaryOp < Expr
  def initialize(@op : String, @operand : Expr)
  end

  def evaluate : Float64
    case @op
    when "-" then -@operand.evaluate
    when "abs" then @operand.evaluate.abs
    else raise "Unknown unary op: #{@op}"
    end
  end

  def to_s : String
    "(#{@op} #{@operand})"
  end
end

# สร้าง expression: (3 + 4) * (2 - 1)
expr = BinaryOp.new(
  BinaryOp.new(NumberExpr.new(3.0), "+", NumberExpr.new(4.0)),
  "*",
  BinaryOp.new(NumberExpr.new(2.0), "-", NumberExpr.new(1.0))
)

puts expr.to_s       # => ((3.0 + 4.0) * (2.0 - 1.0))
puts expr.evaluate   # => 7.0

# Negative number: -(5 + 3)
neg_expr = UnaryOp.new("-",
  BinaryOp.new(NumberExpr.new(5.0), "+", NumberExpr.new(3.0))
)
puts neg_expr.evaluate  # => -8.0
```

## Mutual Recursion

```crystal
# Mutual recursion - A อ้างถึง B และ B อ้างถึง A

# Forward declarations จำเป็น
class EvenChecker; end
class OddChecker; end

class EvenChecker
  def even?(n : Int32) : Bool
    if n == 0
      true
    else
      OddChecker.new.odd?(n - 1)
    end
  end
end

class OddChecker
  def odd?(n : Int32) : Bool
    if n == 0
      false
    else
      EvenChecker.new.even?(n - 1)
    end
  end
end

# ทดสอบ (ไม่ effective สำหรับตัวเลขใหญ่ แต่แสดง mutual recursion)
checker = EvenChecker.new
puts checker.even?(4)  # => true
puts checker.even?(7)  # => false

# Better example: JSON-like structure (mutual recursion)
alias JsonVal = Nil | Bool | Int64 | Float64 | String | JsonArr | JsonObj

class JsonArr < Array(JsonVal)
  def to_crystal_s : String
    "[#{map { |v| format_val(v) }.join(", ")}]"
  end

  private def format_val(v : JsonVal) : String
    case v
    when JsonArr then v.to_crystal_s
    when JsonObj then v.to_crystal_s
    when String  then "\"#{v}\""
    when Nil     then "nil"
    else              v.to_s
    end
  end
end

class JsonObj < Hash(String, JsonVal)
  def to_crystal_s : String
    pairs = map { |k, v| "#{k.inspect} => #{format_val(v)}" }
    "{#{pairs.join(", ")}}"
  end

  private def format_val(v : JsonVal) : String
    case v
    when JsonArr then v.to_crystal_s
    when JsonObj then v.to_crystal_s
    when String  then "\"#{v}\""
    when Nil     then "nil"
    else              v.to_s
    end
  end
end

# สร้าง nested structure
obj = JsonObj.new
arr = JsonArr.new
arr << 1_i64
arr << "hello"
arr << true

nested = JsonObj.new
nested["name"] = "Crystal"
nested["version"] = 1_i64

obj["data"] = arr
obj["config"] = nested

puts obj.to_crystal_s
```

## Recursive Type กับ Visitor Pattern

```crystal
# Visitor pattern สำหรับ recursive types
abstract class FileSystemItem
  abstract def accept(visitor : FileSystemVisitor)
  abstract def name : String
end

abstract class FileSystemVisitor
  abstract def visit_file(file : RegularFile)
  abstract def visit_directory(dir : Directory)
end

class RegularFile < FileSystemItem
  getter name : String
  getter size : Int64

  def initialize(@name, @size = 0_i64)
  end

  def accept(visitor : FileSystemVisitor)
    visitor.visit_file(self)
  end
end

class Directory < FileSystemItem
  getter name : String
  getter children : Array(FileSystemItem)

  def initialize(@name)
    @children = Array(FileSystemItem).new
  end

  def add(item : FileSystemItem) : self
    @children << item
    self
  end

  def accept(visitor : FileSystemVisitor)
    visitor.visit_directory(self)
  end
end

# Concrete visitors
class SizeCalculator < FileSystemVisitor
  getter total_size : Int64

  def initialize
    @total_size = 0_i64
  end

  def visit_file(file : RegularFile)
    @total_size += file.size
  end

  def visit_directory(dir : Directory)
    dir.children.each(&.accept(self))
  end
end

class TreePrinter < FileSystemVisitor
  def initialize(@indent = 0)
  end

  def visit_file(file : RegularFile)
    puts " " * @indent + "📄 #{file.name} (#{file.size} bytes)"
  end

  def visit_directory(dir : Directory)
    puts " " * @indent + "📁 #{dir.name}/"
    printer = TreePrinter.new(@indent + 2)
    dir.children.each(&.accept(printer))
  end
end

# สร้าง filesystem tree
root = Directory.new("project")
  .add(RegularFile.new("README.md", 1024_i64))
  .add(RegularFile.new("shard.yml", 256_i64))
  .add(
    Directory.new("src")
      .add(RegularFile.new("main.cr", 4096_i64))
      .add(RegularFile.new("helper.cr", 2048_i64))
  )
  .add(
    Directory.new("spec")
      .add(RegularFile.new("main_spec.cr", 1536_i64))
  )

printer = TreePrinter.new
root.accept(printer)

calculator = SizeCalculator.new
root.accept(calculator)
puts "Total size: #{calculator.total_size} bytes"
```

## Linked List Types

```crystal
# Functional linked list (recursive algebraic type)
abstract class FList(T)
  def self.empty : Empty(T)
    Empty(T).new
  end

  def self.cons(head : T, tail : FList(T)) : Cons(T)
    Cons(T).new(head, tail)
  end

  abstract def empty? : Bool
  abstract def head : T
  abstract def tail : FList(T)
  abstract def size : Int32
  abstract def to_array : Array(T)

  def prepend(value : T) : Cons(T)
    FList(T).cons(value, self)
  end

  def map(&block : T -> U) : FList(U) forall U
    if empty?
      FList(U).empty
    else
      FList(U).cons(block.call(head), tail.map(&block))
    end
  end

  def filter(&pred : T -> Bool) : FList(T)
    if empty?
      self
    elsif pred.call(head)
      FList(T).cons(head, tail.filter(&pred))
    else
      tail.filter(&pred)
    end
  end

  def fold(initial : U, &op : U, T -> U) : U forall U
    if empty?
      initial
    else
      tail.fold(op.call(initial, head), &op)
    end
  end
end

class Empty(T) < FList(T)
  def empty? : Bool
    true
  end

  def head : T
    raise "Empty list has no head"
  end

  def tail : FList(T)
    raise "Empty list has no tail"
  end

  def size : Int32
    0
  end

  def to_array : Array(T)
    Array(T).new
  end

  def to_s : String
    "[]"
  end
end

class Cons(T) < FList(T)
  def initialize(@head : T, @tail : FList(T))
  end

  def empty? : Bool
    false
  end

  def head : T
    @head
  end

  def tail : FList(T)
    @tail
  end

  def size : Int32
    1 + @tail.size
  end

  def to_array : Array(T)
    [@head] + @tail.to_array
  end

  def to_s : String
    "[#{to_array.join(" -> ")}]"
  end
end

# สร้าง list
list = FList(Int32).empty
  .prepend(5)
  .prepend(4)
  .prepend(3)
  .prepend(2)
  .prepend(1)

puts list          # => [1 -> 2 -> 3 -> 4 -> 5]
puts list.size     # => 5

doubled = list.map { |x| x * 2 }
puts doubled       # => [2 -> 4 -> 6 -> 8 -> 10]

evens = list.filter { |x| x.even? }
puts evens         # => [2 -> 4]

sum = list.fold(0) { |acc, x| acc + x }
puts sum           # => 15
```

## แบบฝึกหัด

### แบบฝึกหัดที่ 1: N-ary Tree
สร้าง N-ary tree ที่:
- แต่ละ node มี children ไม่จำกัด
- Breadth-first traversal
- Depth-first traversal
- Path finding

### แบบฝึกหัดที่ 2: Trie
Implement Trie data structure:
- Insert words
- Search words
- Prefix search
- Delete words

### แบบฝึกหัดที่ 3: Expression Tree
สร้าง math expression tree ที่:
- รองรับ variables
- Simplification rules
- Derivative computation
- Pretty printing

### แบบฝึกหัดที่ 4: Directed Graph
สร้าง recursive directed graph:
- สร้าง cycles detection
- Topological sort
- Strongly connected components

## สรุป

Recursive Types ใน Crystal:
- **Forward declarations**: ประกาศ type ก่อนเพื่อ reference กันเอง
- **Nilable references**: ใช้ `T?` สำหรับ optional links
- **Mutual recursion**: ใช้ forward declarations สำหรับ A ↔ B
- **Abstract classes**: เหมาะสำหรับ algebraic data types

Patterns ที่ใช้บ่อย:
1. Linked lists (singly, doubly)
2. Tree structures (BST, trie, AST)
3. Graph representations
4. Recursive algebraic types (functional style)
