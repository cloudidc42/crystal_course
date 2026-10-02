# Part 190: CI/CD - GitLab CI ใน Crystal

## บทนำ

GitLab CI เป็น alternative ที่ทรงพลังสำหรับ CI/CD โดยเฉพาะสำหรับ self-hosted environments หรือ enterprises ที่ใช้ GitLab

## .gitlab-ci.yml พื้นฐาน

```yaml
# .gitlab-ci.yml
image: crystallang/crystal:latest

variables:
  POSTGRES_DB: myapp_test
  POSTGRES_USER: postgres
  POSTGRES_PASSWORD: password
  DATABASE_URL: "postgres://postgres:password@postgres:5432/myapp_test"
  REDIS_URL: "redis://redis:6379"

stages:
  - lint
  - test
  - build
  - deploy

# Cache shards ระหว่าง jobs
cache:
  key:
    files:
      - shard.yml
      - shard.lock
  paths:
    - .cache/shards/
    - lib/

before_script:
  - shards install --production
```

## Lint Stage

```yaml
# Linting jobs
lint:format:
  stage: lint
  script:
    - crystal tool format --check
  rules:
    - if: $CI_MERGE_REQUEST_IID

lint:ameba:
  stage: lint
  script:
    - shards install  # ต้องการ dev dependencies
    - ./bin/ameba --config .ameba.yml
  rules:
    - if: $CI_MERGE_REQUEST_IID
  allow_failure: true  # อย่า fail build เพราะ style warnings

lint:security:
  stage: lint
  script:
    # ตรวจ vulnerable dependencies
    - crystal run scripts/security_check.cr
  rules:
    - if: $CI_PIPELINE_SOURCE == "schedule"
    - if: $CI_MERGE_REQUEST_IID
```

## Test Stage

```yaml
# Test jobs
test:unit:
  stage: test
  script:
    - crystal spec spec/unit/ --error-trace
  coverage: '/Lines:\s+(\d+\.\d+)%/'

test:integration:
  stage: test
  services:
    - name: postgres:16-alpine
      alias: postgres
      variables:
        POSTGRES_DB: myapp_test
        POSTGRES_USER: postgres
        POSTGRES_PASSWORD: password
    - name: redis:7-alpine
      alias: redis
  script:
    - crystal spec spec/integration/ --error-trace
  rules:
    - if: $CI_MERGE_REQUEST_IID
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

test:e2e:
  stage: test
  services:
    - name: postgres:16-alpine
      alias: postgres
    - name: redis:7-alpine
      alias: redis
  script:
    - crystal build --release src/main.cr -o server
    - ./server &
    - sleep 2
    - crystal run spec/e2e/runner.cr
  after_script:
    - kill $(cat server.pid) 2>/dev/null || true
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

# Parallel test jobs
test:parallel:
  stage: test
  parallel: 4
  script:
    - crystal spec spec/unit/ --error-trace
  variables:
    SPEC_SHARD_INDEX: $CI_NODE_INDEX
    SPEC_SHARD_TOTAL: $CI_NODE_TOTAL
```

## Build Stage

```yaml
# Build jobs
build:linux:
  stage: build
  image: crystallang/crystal:latest-alpine
  before_script:
    - apk add --no-cache openssl-dev openssl-static libpq-dev zlib-dev
    - shards install --production
  script:
    - crystal build --release --static src/main.cr -o dist/app-linux-amd64
  artifacts:
    name: "$CI_JOB_NAME-$CI_COMMIT_REF_SLUG"
    paths:
      - dist/
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

build:docker:
  stage: build
  image: docker:24
  services:
    - docker:dind
  variables:
    DOCKER_BUILDKIT: "1"
  before_script:
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD $CI_REGISTRY
  script:
    - |
      docker build \
        --cache-from $CI_REGISTRY_IMAGE:latest \
        --tag $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA \
        --tag $CI_REGISTRY_IMAGE:latest \
        .
    - docker push $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA
    - docker push $CI_REGISTRY_IMAGE:latest
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG
```

## Deploy Stage

```yaml
# Deploy jobs
deploy:staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$STAGING_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $STAGING_HOST >> ~/.ssh/known_hosts
  script:
    - |
      ssh $STAGING_USER@$STAGING_HOST "
        cd /app &&
        docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_SHA &&
        docker-compose up -d --no-deps app &&
        docker system prune -f
      "
  environment:
    name: staging
    url: https://staging.myapp.com
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH

deploy:production:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$PROD_SSH_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $PROD_HOST >> ~/.ssh/known_hosts
  script:
    - |
      ssh $PROD_USER@$PROD_HOST "
        cd /app &&
        docker pull $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG &&
        docker tag $CI_REGISTRY_IMAGE:$CI_COMMIT_TAG $CI_REGISTRY_IMAGE:stable &&
        docker-compose up -d --no-deps app
      "
  environment:
    name: production
    url: https://myapp.com
  when: manual  # ต้องกด manual
  rules:
    - if: $CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/
```

## GitLab CI Templates

```yaml
# Reusable templates ใน .gitlab-ci.yml
.crystal_base:
  image: crystallang/crystal:latest
  cache:
    key:
      files: [shard.yml]
    paths:
      - lib/
      - .cache/
  before_script:
    - shards install

.deploy_base:
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client curl
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh && chmod 700 ~/.ssh
    - ssh-keyscan $DEPLOY_HOST >> ~/.ssh/known_hosts

# Extend templates
test:unit:
  extends: .crystal_base
  stage: test
  script:
    - crystal spec spec/unit/

deploy:staging:
  extends: .deploy_base
  stage: deploy
  variables:
    DEPLOY_HOST: $STAGING_HOST
  script:
    - ./scripts/deploy.sh staging
```

## แบบฝึกหัด

1. สร้าง GitLab CI pipeline ที่ run tests แบบ parallel กับ PostgreSQL service
2. เพิ่ม Docker build และ push ไปยัง GitLab Container Registry
3. สร้าง manual deploy job สำหรับ production พร้อม environment protection
4. เพิ่ม scheduled pipeline สำหรับ nightly builds และ security scans

## สรุป

GitLab CI สำหรับ Crystal:
- **.gitlab-ci.yml**: configuration ไฟล์หลัก
- **stages**: ลำดับ execution (lint, test, build, deploy)
- **services**: PostgreSQL/Redis สำหรับ integration tests
- **parallel**: แบ่ง test job ออกเป็นหลาย runners
- **artifacts**: เก็บ build output ระหว่าง stages
- **environment**: staging/production environments พร้อม URL
- **when: manual**: ต้องการ manual approval สำหรับ production deploy
- **rules**: กำหนดว่า job จะรันเมื่อไหร่
- **CI_COMMIT_TAG**: trigger deployment เมื่อมี version tag
- **GitLab Container Registry**: built-in registry สำหรับ Docker images
