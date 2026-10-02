# Part 169: CI Testing ใน Crystal

## บทนำ

CI (Continuous Integration) คือการ run tests อัตโนมัติทุกครั้งที่มี code change เพื่อตรวจสอบว่า code ยังทำงานได้ถูกต้อง

## GitHub Actions สำหรับ Crystal

### CI พื้นฐาน

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: latest

      - name: Cache shards
        uses: actions/cache@v4
        with:
          path: |
            ~/.cache/shards
            lib
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.lock') }}
          restore-keys: |
            ${{ runner.os }}-shards-

      - name: Install dependencies
        run: shards install --frozen

      - name: Check formatting
        run: crystal tool format --check

      - name: Run tests
        run: crystal spec --order=random

      - name: Build (ensure it compiles)
        run: crystal build src/main.cr --no-codegen
```

## Test Matrix (Multiple Crystal Versions)

```yaml
# .github/workflows/matrix.yml
name: Test Matrix

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 0 * * 0'  # ทุกอาทิตย์

jobs:
  test:
    name: Crystal ${{ matrix.crystal }} on ${{ matrix.os }}
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false  # รัน matrix ทั้งหมดแม้มี failure
      matrix:
        os: [ubuntu-latest, macos-latest]
        crystal: [latest, nightly]
        include:
          - os: ubuntu-latest
            crystal: "1.12.0"
          - os: ubuntu-latest
            crystal: "1.11.0"
        exclude:
          - os: macos-latest
            crystal: nightly

    steps:
      - uses: actions/checkout@v4

      - name: Install Crystal ${{ matrix.crystal }}
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: ${{ matrix.crystal }}

      - name: Install dependencies
        run: shards install

      - name: Run tests
        run: crystal spec

      - name: Type check
        run: crystal build src/main.cr --no-codegen
```

## Database Tests ใน CI

```yaml
# .github/workflows/ci-with-db.yml
name: CI with Database

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

      redis:
        image: redis:7-alpine
        ports:
          - 6379:6379
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
      REDIS_URL: redis://localhost:6379
      APP_ENV: test

    steps:
      - uses: actions/checkout@v4

      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1

      - name: Cache shards
        uses: actions/cache@v4
        with:
          path: ~/.cache/shards
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.lock') }}

      - name: Install dependencies
        run: shards install

      - name: Setup database
        run: crystal run db/migrate.cr -- up

      - name: Run tests
        run: crystal spec --order=random

      - name: Upload coverage report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/
```

## Coverage Reporting to Codecov

```yaml
# .github/workflows/coverage.yml
name: Coverage

on:
  push:
    branches: [main]
  pull_request:

jobs:
  coverage:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    env:
      DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
      APP_ENV: test

    steps:
      - uses: actions/checkout@v4

      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1

      - name: Install LLVM tools
        run: |
          sudo apt-get update
          sudo apt-get install -y llvm lcov

      - name: Install dependencies
        run: shards install

      - name: Setup database
        run: crystal run db/migrate.cr -- up

      - name: Run tests with coverage
        run: |
          crystal spec --coverage --coverage-dir=coverage
          # หรือ ถ้าใช้ LLVM coverage:
          # crystal build --coverage spec/all_spec.cr -o spec_runner
          # LLVM_PROFILE_FILE=spec.profraw ./spec_runner
          # llvm-profdata merge -sparse spec.profraw -o spec.profdata
          # llvm-cov export spec_runner -instr-profile=spec.profdata -format=lcov > coverage/lcov.info

      - name: Upload to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          files: ./coverage/lcov.info
          flags: unittests
          name: crystal-coverage
          fail_ci_if_error: false
          verbose: true

      - name: Check coverage threshold
        run: |
          COVERAGE=$(lcov --summary coverage/lcov.info 2>&1 | grep "lines" | grep -oP '\d+\.\d+%' | head -1 | tr -d '%')
          echo "Coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below required 80%"
            exit 1
          fi
```

## PR Checks และ Status

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  lint:
    name: Code Style
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1
      - name: Check formatting
        run: crystal tool format --check

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1
      - name: Install dependencies
        run: shards install
      - name: Security audit
        run: |
          # ตรวจสอบ known vulnerabilities ใน dependencies
          shards check

  test:
    name: Tests
    runs-on: ubuntu-latest
    needs: [lint]  # รัน test หลัง lint ผ่านเท่านั้น

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1

      - name: Cache shards
        uses: actions/cache@v4
        with:
          path: ~/.cache/shards
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.lock') }}

      - name: Install dependencies
        run: shards install

      - name: Setup test database
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
        run: crystal run db/migrate.cr -- up

      - name: Run unit tests
        run: crystal spec spec/unit --order=random

      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test
        run: crystal spec spec/integration --order=random

  build:
    name: Build
    runs-on: ubuntu-latest
    needs: [test]

    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1
      - name: Install dependencies
        run: shards install
      - name: Build release
        run: crystal build --release src/main.cr -o bin/app
      - name: Upload artifact
        uses: actions/upload-artifact@v4
        with:
          name: app-binary
          path: bin/app
```

## Crystal Spec ใน CI

```crystal
# spec/spec_helper.cr - CI-aware configuration
require "spec"

# CI-specific settings
if ENV["CI"]? == "true"
  # รัน tests แบบ verbose ใน CI
  Spec.override_default_formatter(Spec::VerboseFormatter.new)

  # ตั้งค่า database จาก environment variables
  DB_URL = ENV["DATABASE_URL"]? || "postgresql://localhost/test"
else
  DB_URL = "postgresql://localhost/myapp_test"
end

# Load support files
require "./support/**"

# Setup/teardown hooks
Spec.before_suite do
  DatabaseHelper.run_migrations!
end

Spec.after_suite do
  DatabaseHelper.cleanup!
end
```

## Parallel Test Execution

```yaml
# .github/workflows/parallel.yml
name: Parallel Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest

    strategy:
      matrix:
        shard_index: [0, 1, 2, 3]
        shard_total: [4]

    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB: myapp_test_${{ matrix.shard_index }}
          POSTGRES_USER: postgres
          POSTGRES_PASSWORD: password
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1

      - name: Install dependencies
        run: shards install

      - name: Setup shard database
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test_${{ matrix.shard_index }}
        run: crystal run db/migrate.cr -- up

      - name: Run test shard ${{ matrix.shard_index }}/${{ matrix.shard_total }}
        env:
          DATABASE_URL: postgresql://postgres:password@localhost:5432/myapp_test_${{ matrix.shard_index }}
          TEST_SHARD_INDEX: ${{ matrix.shard_index }}
          TEST_SHARD_TOTAL: ${{ matrix.shard_total }}
        run: |
          # แบ่ง spec files ตาม shard_index
          SPEC_FILES=$(ls spec/**/*_spec.cr | awk "NR % ${{ matrix.shard_total }} == ${{ matrix.shard_index }}")
          crystal spec $SPEC_FILES

      - name: Upload results
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-results-${{ matrix.shard_index }}
          path: test-results/
```

## Scheduled Tests

```yaml
# .github/workflows/scheduled.yml
name: Scheduled Tests

on:
  schedule:
    - cron: '0 2 * * *'  # ทุกวัน 02:00 UTC
  workflow_dispatch:  # รันด้วย manual trigger ได้

jobs:
  regression-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1

      - name: Install dependencies
        run: shards install

      - name: Run all tests including slow ones
        run: crystal spec --tag=all --order=random

      - name: Run performance benchmarks
        run: crystal run benchmarks/main.cr

      - name: Notify on failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          channel: '#alerts'
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK }}
```

## สรุป

CI Testing สำหรับ Crystal:
- **GitHub Actions**: `.github/workflows/*.yml` สำหรับ CI/CD
- **Matrix builds**: ทดสอบหลาย Crystal versions พร้อมกัน
- **Services**: PostgreSQL, Redis ใน CI environment
- **Coverage**: ส่ง coverage report ไปยัง Codecov
- **PR Checks**: lint, security, tests, build
- **Parallel**: แบ่ง test suite เป็น shards รันพร้อมกัน

CI ที่ดีทำให้ทุก PR มั่นใจได้ว่า code quality ไม่ลดลง
