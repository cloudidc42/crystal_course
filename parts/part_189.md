# Part 189: CI/CD - GitHub Actions ใน Crystal

## บทนำ

GitHub Actions เป็นเครื่องมือ CI/CD ที่ integrate กับ GitHub โดยตรง ช่วย automate การ test, build, และ deploy Crystal applications

## .github/workflows/ci.yml พื้นฐาน

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

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Install Crystal
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: latest

      - name: Cache shards
        uses: actions/cache@v4
        with:
          path: ~/.cache/shards
          key: ${{ runner.os }}-shards-${{ hashFiles('shard.yml') }}
          restore-keys: |
            ${{ runner.os }}-shards-

      - name: Install dependencies
        run: shards install

      - name: Run linter (ameba)
        run: ./bin/ameba

      - name: Check formatting
        run: crystal tool format --check

      - name: Run tests
        env:
          DATABASE_URL: postgres://postgres:password@localhost:5432/myapp_test
          REDIS_URL: redis://localhost:6379
          APP_ENV: test
        run: crystal spec --error-trace

      - name: Build release
        run: crystal build --release --no-debug src/main.cr -o /dev/null
```

## Matrix Build - Test Multiple Crystal Versions

```yaml
# .github/workflows/matrix.yml
name: Matrix CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 6 * * 1'  # Weekly on Monday at 6 AM

jobs:
  test:
    name: Test Crystal ${{ matrix.crystal }}
    runs-on: ${{ matrix.os }}

    strategy:
      fail-fast: false
      matrix:
        crystal: ["1.10.0", "1.11.0", "1.12.0", "latest", "nightly"]
        os: [ubuntu-latest, macos-latest]
        exclude:
          # nightly ใช้แค่ ubuntu
          - crystal: nightly
            os: macos-latest

    steps:
      - uses: actions/checkout@v4

      - name: Install Crystal ${{ matrix.crystal }}
        uses: crystal-lang/install-crystal@v1
        with:
          crystal: ${{ matrix.crystal }}

      - name: Install dependencies
        run: shards install

      - name: Run specs
        run: crystal spec
        continue-on-error: ${{ matrix.crystal == 'nightly' }}
```

## Coverage Reporting

```yaml
# .github/workflows/coverage.yml
name: Coverage

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  coverage:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: crystal-lang/install-crystal@v1
        with:
          crystal: latest

      - name: Install lcov
        run: sudo apt-get install -y lcov

      - name: Install dependencies
        run: shards install

      - name: Run tests with coverage
        run: |
          crystal spec --error-trace \
            --compiler-flags=--debug \
            2>&1 | tee spec_output.txt

      # Upload to Codecov
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v4
        with:
          token: ${{ secrets.CODECOV_TOKEN }}
          fail_ci_if_error: false
```

## Build และ Push Docker Image

```yaml
# .github/workflows/docker.yml
name: Docker Build & Push

on:
  push:
    branches: [main]
    tags:
      - 'v*'

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=ref,event=branch
            type=ref,event=pr
            type=semver,pattern={{version}}
            type=semver,pattern={{major}}.{{minor}}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: ${{ github.event_name != 'pull_request' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

## Deployment Action

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    tags:
      - 'v*'
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ github.event.inputs.environment || 'production' }}

    steps:
      - uses: actions/checkout@v4

      - name: Deploy to server
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.SERVER_HOST }}
          username: ${{ secrets.SERVER_USER }}
          key: ${{ secrets.SERVER_SSH_KEY }}
          script: |
            cd /app
            docker pull ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            docker-compose up -d --no-deps app
            docker system prune -f
```

## Parallel Test Sharding

```yaml
# .github/workflows/parallel-tests.yml
name: Parallel Tests

on:
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        shard_index: [0, 1, 2, 3]
        shard_total: [4]

    steps:
      - uses: actions/checkout@v4
      - uses: crystal-lang/install-crystal@v1

      - name: Install dependencies
        run: shards install

      - name: Run tests (shard ${{ matrix.shard_index }})
        env:
          SPEC_SHARD_INDEX: ${{ matrix.shard_index }}
          SPEC_SHARD_TOTAL: ${{ matrix.shard_total }}
        run: |
          # Crystal spec ไม่รองรับ shard โดยตรง แต่เราทำเองได้
          crystal run scripts/shard_runner.cr -- \
            ${{ matrix.shard_index }} ${{ matrix.shard_total }}
```

## แบบฝึกหัด

1. สร้าง CI workflow ที่ test Crystal app กับ PostgreSQL และ Redis
2. เพิ่ม Docker build ที่ push image ไปยัง GitHub Container Registry
3. สร้าง matrix build สำหรับ Crystal versions 1.10, 1.11, latest
4. เพิ่ม caching สำหรับ shards และ build artifacts ลด CI time

## สรุป

GitHub Actions สำหรับ Crystal:
- **crystal-lang/install-crystal@v1**: action สำหรับ install Crystal
- **services**: run PostgreSQL/Redis สำหรับ integration tests
- **cache**: cache shards และ build artifacts ลด CI time
- **matrix**: test หลาย Crystal versions พร้อมกัน
- **docker/build-push-action**: build และ push Docker images
- **environment**: deploy สำหรับ staging/production พร้อม approval
- **workflow_dispatch**: trigger manual deployments
- **tags trigger**: auto-deploy เมื่อ create git tag
