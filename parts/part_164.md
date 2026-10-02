# Part 164: Test Coverage ใน Crystal

## บทนำ

Test Coverage วัดว่า code ของเราถูก test มากแค่ไหน เป็นตัวช่วยระบุส่วนที่ขาด tests แต่ coverage 100% ไม่ได้หมายความว่า code ไม่มี bugs

## ประเภทของ Coverage

```
Coverage Types:
1. Line Coverage   - กี่บรรทัดถูกรัน
2. Branch Coverage - กี่ branch (if/else) ถูกทดสอบ
3. Method Coverage - กี่ method ถูกเรียก
4. Statement Coverage - กี่ statements ถูกรัน
```

## Crystal Coverage ด้วย --coverage

Crystal มี built-in support สำหรับ code coverage:

```bash
# รัน tests พร้อม coverage
crystal spec --coverage

# ระบุ coverage output directory
crystal spec --coverage --coverage-dir=coverage

# รัน coverage แบบ LLVM
crystal build --coverage src/app.cr -o app_coverage
./app_coverage
llvm-profdata merge -sparse default.profraw -o default.profdata
llvm-cov report app_coverage -instr-profile=default.profdata
```

## ตั้งค่า Coverage ใน Project

```crystal
# spec/spec_helper.cr
require "spec"

# เปิด coverage tracking
Spec.before_suite do
  puts "เริ่มต้น tests..."
end

Spec.after_suite do
  puts "Tests เสร็จสิ้น"
end
```

```yaml
# .github/workflows/test.yml
- name: Run tests with coverage
  run: crystal spec --coverage

- name: Upload coverage
  uses: codecov/codecov-action@v3
  with:
    files: ./coverage/lcov.info
```

## Code ที่ต้องการ Coverage

```crystal
# src/order_calculator.cr
class OrderCalculator
  TAX_RATE = 0.07_f64
  FREE_SHIPPING_THRESHOLD = 500.0_f64
  SHIPPING_COST = 50.0_f64

  def calculate(items : Array(OrderItem), discount_code : String? = nil) : OrderSummary
    subtotal = calculate_subtotal(items)
    discount = calculate_discount(subtotal, discount_code)
    shipping = calculate_shipping(subtotal - discount)
    tax = calculate_tax(subtotal - discount)
    total = subtotal - discount + shipping + tax

    OrderSummary.new(
      subtotal:  subtotal,
      discount:  discount,
      shipping:  shipping,
      tax:       tax,
      total:     total
    )
  end

  private def calculate_subtotal(items : Array(OrderItem)) : Float64
    items.sum { |item| item.price * item.quantity }
  end

  private def calculate_discount(subtotal : Float64, code : String?) : Float64
    return 0.0 if code.nil? || code.empty?

    case code.upcase
    when "SAVE10"
      subtotal * 0.10
    when "SAVE20"
      subtotal * 0.20
    when "HALF"
      subtotal * 0.50
    else
      0.0  # invalid code
    end
  end

  private def calculate_shipping(subtotal : Float64) : Float64
    if subtotal >= FREE_SHIPPING_THRESHOLD
      0.0  # ส่งฟรี
    else
      SHIPPING_COST
    end
  end

  private def calculate_tax(subtotal : Float64) : Float64
    subtotal * TAX_RATE
  end
end

record OrderItem, price : Float64, quantity : Int32
record OrderSummary,
  subtotal : Float64,
  discount : Float64,
  shipping : Float64,
  tax : Float64,
  total : Float64
```

## Spec ที่ครอบคลุม Coverage

```crystal
# spec/order_calculator_spec.cr
require "./spec_helper"
require "../src/order_calculator"

describe OrderCalculator do
  @calc : OrderCalculator? = nil

  before_each do
    @calc = OrderCalculator.new
  end

  private def calc
    @calc.not_nil!
  end

  private def items(price : Float64 = 100.0, qty : Int32 = 1)
    [OrderItem.new(price: price, quantity: qty)]
  end

  describe "#calculate" do
    context "ไม่มี discount code" do
      it "คำนวณ subtotal ถูกต้อง" do
        result = calc.calculate([
          OrderItem.new(price: 100.0, quantity: 2),
          OrderItem.new(price: 50.0, quantity: 3)
        ])
        result.subtotal.should eq(350.0)
      end

      it "ไม่มี discount" do
        result = calc.calculate(items(100.0, 1))
        result.discount.should eq(0.0)
      end
    end

    context "มี discount code" do
      it "SAVE10 ลด 10%" do
        result = calc.calculate(items(100.0, 1), discount_code: "SAVE10")
        result.discount.should eq(10.0)
      end

      it "SAVE20 ลด 20%" do
        result = calc.calculate(items(100.0, 1), discount_code: "SAVE20")
        result.discount.should eq(20.0)
      end

      it "HALF ลด 50%" do
        result = calc.calculate(items(100.0, 1), discount_code: "HALF")
        result.discount.should eq(50.0)
      end

      it "code ไม่ถูกต้องไม่มี discount" do
        result = calc.calculate(items(100.0, 1), discount_code: "INVALID")
        result.discount.should eq(0.0)
      end

      it "case insensitive" do
        result = calc.calculate(items(100.0, 1), discount_code: "save10")
        result.discount.should eq(10.0)
      end
    end

    context "shipping" do
      it "คิดค่าส่ง 50 บาท เมื่อ subtotal น้อยกว่า 500" do
        result = calc.calculate(items(100.0, 1))
        result.shipping.should eq(50.0)
      end

      it "ส่งฟรีเมื่อ subtotal มากกว่าหรือเท่ากับ 500" do
        result = calc.calculate(items(500.0, 1))
        result.shipping.should eq(0.0)
      end

      it "ส่งฟรีเมื่อ subtotal มากกว่า 500" do
        result = calc.calculate(items(600.0, 1))
        result.shipping.should eq(0.0)
      end

      it "คำนวณ shipping หลังหัก discount" do
        # subtotal=1000, discount=50% = 500, ส่งฟรี
        result = calc.calculate(items(1000.0, 1), discount_code: "HALF")
        result.shipping.should eq(0.0)

        # subtotal=1000, discount=50% = 500, ส่งฟรี (ที่ threshold พอดี)
        result2 = calc.calculate(items(200.0, 1), discount_code: "HALF")
        # 200 - 100 = 100, น้อยกว่า 500 ต้องเสียค่าส่ง
        result2.shipping.should eq(50.0)
      end
    end

    context "tax" do
      it "คำนวณ tax 7%" do
        result = calc.calculate(items(100.0, 1))
        result.tax.should be_close(7.0, 0.01)
      end
    end

    context "total" do
      it "total = subtotal - discount + shipping + tax" do
        result = calc.calculate(items(100.0, 1))
        expected = 100.0 - 0.0 + 50.0 + 7.0
        result.total.should be_close(expected, 0.01)
      end
    end
  end
end
```

## Coverage Reports

### ตัวอย่าง Coverage Output

```
Files                            Lines   Branches
src/order_calculator.cr          97.8%   92.3%
  ├── calculate                  100%
  ├── calculate_subtotal         100%
  ├── calculate_discount         100%    88.9%
  │   └── else branch (line 32)  ✓
  ├── calculate_shipping         100%    100%
  └── calculate_tax              100%

Overall: 97.8% lines, 92.3% branches
```

### สร้าง Coverage Report

```crystal
# tools/coverage_reporter.cr
require "json"

struct CoverageData
  include JSON::Serializable

  property file : String
  property lines : Hash(Int32, Int32)  # line_number => hit_count

  def coverage_pct : Float64
    covered = lines.values.count { |hits| hits > 0 }
    return 100.0 if lines.empty?
    covered.to_f / lines.size * 100
  end

  def uncovered_lines : Array(Int32)
    lines.select { |_, hits| hits == 0 }.keys.sort
  end
end

class CoverageReporter
  def initialize(@data : Array(CoverageData))
  end

  def report
    total_lines = 0
    covered_lines = 0

    puts "=" * 60
    puts "Coverage Report"
    puts "=" * 60

    @data.sort_by(&.file).each do |file_data|
      pct = file_data.coverage_pct
      total_lines += file_data.lines.size
      covered_lines += file_data.lines.values.count { |h| h > 0 }

      status = case pct
      when 90.0..100.0 then "✓"
      when 70.0...90.0 then "~"
      else "✗"
      end

      puts "#{status} #{file_data.file.ljust(50)} #{pct.round(1).to_s.rjust(6)}%"

      if pct < 100.0
        uncovered = file_data.uncovered_lines
        unless uncovered.empty?
          puts "  ไม่ได้ test: lines #{uncovered.join(", ")}"
        end
      end
    end

    overall = total_lines > 0 ? (covered_lines.to_f / total_lines * 100) : 100.0
    puts "-" * 60
    puts "Total: #{covered_lines}/#{total_lines} lines = #{overall.round(1)}%"

    if overall < 80.0
      puts "\n⚠️  Coverage ต่ำกว่า 80%! ควรเพิ่ม tests"
      exit(1)
    end
  end
end
```

## Coverage Goals

```crystal
# spec/spec_helper.cr
require "spec"

# กำหนด coverage threshold
COVERAGE_THRESHOLD = 80.0

Spec.after_suite do
  if ENV["CHECK_COVERAGE"]? == "true"
    coverage = load_coverage_data
    if coverage < COVERAGE_THRESHOLD
      STDERR.puts "Coverage #{coverage}% < required #{COVERAGE_THRESHOLD}%"
      exit(1)
    else
      puts "Coverage: #{coverage}% ✓"
    end
  end
end

def load_coverage_data : Float64
  # อ่านจาก coverage output file
  if File.exists?("coverage/lcov.info")
    parse_lcov_coverage
  else
    0.0
  end
end

def parse_lcov_coverage : Float64
  total_lines = 0
  covered_lines = 0

  File.each_line("coverage/lcov.info") do |line|
    case line
    when /^DA:(\d+),(\d+)/
      total_lines += 1
      covered_lines += 1 if $2.to_i > 0
    end
  end

  return 100.0 if total_lines == 0
  covered_lines.to_f / total_lines * 100
end
```

## เทคนิคเพิ่ม Coverage

### 1. Table-Driven Tests

```crystal
describe "OrderCalculator discount" do
  test_cases = [
    {code: "SAVE10",  pct: 0.10, desc: "10% discount"},
    {code: "SAVE20",  pct: 0.20, desc: "20% discount"},
    {code: "HALF",    pct: 0.50, desc: "50% discount"},
    {code: "INVALID", pct: 0.00, desc: "invalid code"},
    {code: "",        pct: 0.00, desc: "empty code"},
    {code: "save10",  pct: 0.10, desc: "lowercase code"},
  ]

  test_cases.each do |tc|
    it "#{tc[:desc]}" do
      result = OrderCalculator.new.calculate(
        [OrderItem.new(price: 100.0, quantity: 1)],
        discount_code: tc[:code]
      )
      result.discount.should eq(100.0 * tc[:pct])
    end
  end
end
```

### 2. Property-based Coverage

```crystal
describe "OrderCalculator properties" do
  it "total เสมอมากกว่าหรือเท่ากับ subtotal - discount" do
    1000.times do
      price = rand(1.0..10000.0)
      qty = rand(1..10)
      items = [OrderItem.new(price: price, quantity: qty)]
      code = ["SAVE10", "SAVE20", nil, "INVALID"].sample

      result = OrderCalculator.new.calculate(items, discount_code: code)

      result.total.should be >= result.subtotal - result.discount - 0.01
    end
  end

  it "discount ไม่เกิน subtotal" do
    items = [OrderItem.new(price: 100.0, quantity: 1)]
    ["SAVE10", "SAVE20", "HALF"].each do |code|
      result = OrderCalculator.new.calculate(items, discount_code: code)
      result.discount.should be <= result.subtotal
    end
  end
end
```

### 3. Edge Case Coverage

```crystal
describe "Edge Cases" do
  it "รายการว่าง" do
    result = OrderCalculator.new.calculate([] of OrderItem)
    result.subtotal.should eq(0.0)
    result.total.should eq(0.0)
  end

  it "ราคาเป็นศูนย์" do
    items = [OrderItem.new(price: 0.0, quantity: 5)]
    result = OrderCalculator.new.calculate(items)
    result.total.should eq(50.0)  # แค่ค่าส่ง
  end

  it "จำนวนมาก" do
    items = [OrderItem.new(price: 0.01, quantity: 1000000)]
    result = OrderCalculator.new.calculate(items)
    result.subtotal.should be_close(10000.0, 0.01)
  end

  it "ราคาทศนิยม" do
    items = [OrderItem.new(price: 33.33, quantity: 3)]
    result = OrderCalculator.new.calculate(items)
    result.subtotal.should be_close(99.99, 0.01)
  end
end
```

## Coverage Badge สำหรับ README

```bash
# สร้าง coverage badge
crystal spec --coverage 2>&1 | tee coverage.log
COVERAGE=$(grep "Overall" coverage.log | awk '{print $NF}')
echo "Coverage: $COVERAGE"

# อัปโหลดไปยัง shields.io หรือ codecov
```

## CI/CD Coverage Integration

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: latest

      - name: Install dependencies
        run: shards install

      - name: Run tests with coverage
        run: crystal spec --coverage --coverage-dir=coverage

      - name: Check coverage threshold
        run: |
          COVERAGE=$(cat coverage/summary.json | jq '.total.lines.pct')
          echo "Coverage: $COVERAGE%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "Coverage $COVERAGE% is below threshold 80%"
            exit 1
          fi

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          files: coverage/lcov.info
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: true
```

## แบบฝึกหัด

1. เขียน tests สำหรับ `BankAccount` class ให้ได้ coverage 100% บน branch และ line
2. วิเคราะห์ code ที่มีอยู่แล้วหา uncovered lines แล้วเขียน tests เพิ่ม
3. ตั้งค่า CI ที่ fail เมื่อ coverage ต่ำกว่า 85%
4. สร้าง coverage report ที่แสดง uncovered lines พร้อม code snippet

## สรุป

Test Coverage ใน Crystal:
- **crystal spec --coverage**: สร้าง coverage report
- **Line Coverage**: กี่บรรทัดถูกทดสอบ
- **Branch Coverage**: กี่ branches (if/else) ถูกทดสอบ
- **Coverage Goals**: ตั้ง threshold เช่น 80% สำหรับ enforce ใน CI
- **Table-driven tests**: ทดสอบหลาย cases อย่างมีระเบียบ
- **Edge cases**: อย่าลืม null, empty, boundary values

Coverage เป็น metric ที่ useful แต่ไม่ใช่เป้าหมายสูงสุด ควรเน้น test quality มากกว่า quantity
