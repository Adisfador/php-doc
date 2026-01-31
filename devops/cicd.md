# CI/CD - Непрерывная интеграция и доставка

## Содержание
- [Основы CI/CD](#основы-cicd)
- [Принципы и практики](#принципы-и-практики)
- [GitLab CI/CD](#gitlab-cicd)
- [GitHub Actions](#github-actions)
- [Jenkins](#jenkins)
- [Docker в CI/CD](#docker-в-cicd)
- [Тестирование в CI/CD](#тестирование-в-cicd)
- [Деплой стратегии](#деплой-стратегии)
- [Мониторинг и метрики](#мониторинг-и-метрики)
- [Безопасность в CI/CD](#безопасность-в-cicd)
- [Оптимизация пайплайнов](#оптимизация-пайплайнов)
- [Лучшие практики](#лучшие-практики)
- [Вопросы на собеседовании](#вопросы-на-собеседовании)

## Основы CI/CD

### Что такое CI/CD?

**CI (Continuous Integration)** - непрерывная интеграция
- Автоматическая сборка и тестирование кода при каждом коммите
- Раннее обнаружение проблем
- Частая интеграция изменений

**CD (Continuous Delivery)** - непрерывная доставка
- Автоматическая подготовка к релизу
- Код всегда готов к деплою
- Ручное подтверждение деплоя

**CD (Continuous Deployment)** - непрерывное развертывание
- Полностью автоматический деплой
- Каждое изменение автоматически попадает в production

### Преимущества CI/CD

```
Без CI/CD:
Dev → Manual Build → Manual Tests → Manual Deploy → Production
      (часы/дни)     (часы/дни)      (часы/дни)

С CI/CD:
Dev → Push → Auto Build → Auto Test → Auto Deploy → Production
             (минуты)     (минуты)     (минуты)
```

**Преимущества:**
- Быстрая обратная связь
- Уменьшение количества багов
- Ускорение time-to-market
- Повышение качества кода
- Автоматизация рутинных задач
- Уверенность в стабильности

### Жизненный цикл CI/CD

```
1. Source         → Commit/Push в репозиторий
2. Build          → Компиляция/сборка проекта
3. Test           → Запуск автотестов
4. Package        → Создание артефактов (Docker образы)
5. Deploy Staging → Развертывание на staging
6. Test Staging   → Тестирование на staging
7. Deploy Prod    → Развертывание на production
8. Monitor        → Мониторинг и алерты
```

## Принципы и практики

### 12-факторное приложение (12-Factor App)

```yaml
1. Codebase:
   - Один репозиторий для приложения
   - Множественные деплои из одной кодовой базы

2. Dependencies:
   - Явное объявление зависимостей (composer.json)
   - Изоляция зависимостей (vendor/)

3. Config:
   - Конфигурация в переменных окружения
   - Не хранить секреты в коде

4. Backing Services:
   - База данных, кеш, очереди как подключаемые ресурсы
   - Легкая замена сервисов

5. Build, Release, Run:
   - Строгое разделение стадий
   - Build (composer install) → Release (env) → Run

6. Processes:
   - Stateless процессы
   - Состояние в backing services

7. Port Binding:
   - Экспорт сервисов через порты
   - Self-contained приложение

8. Concurrency:
   - Горизонтальное масштабирование
   - Процессы как граждане первого класса

9. Disposability:
   - Быстрый старт и graceful shutdown
   - Устойчивость к сбоям

10. Dev/Prod Parity:
    - Минимальные различия между окружениями
    - Использование Docker

11. Logs:
    - Логи как потоки событий
    - Централизованное логирование

12. Admin Processes:
    - Одноразовые задачи как процессы
    - Миграции, консольные команды
```

### Версионирование (Semantic Versioning)

```
MAJOR.MINOR.PATCH (например, 2.4.1)

MAJOR - несовместимые изменения API (breaking changes)
MINOR - новая функциональность (обратно совместимая)
PATCH - исправления багов (обратно совместимые)

Pre-release: 1.0.0-alpha, 1.0.0-beta.1, 1.0.0-rc.2
Build metadata: 1.0.0+20230101
```

```bash
# Автоматическое версионирование с git tags
git tag -a v1.0.0 -m "Release version 1.0.0"
git push origin v1.0.0

# Semantic Release (автоматическое версионирование)
# Анализирует commit messages и автоматически создает версии
feat: добавить новую функцию    → MINOR
fix: исправить баг              → PATCH
feat!: breaking change          → MAJOR
```

## GitLab CI/CD

### .gitlab-ci.yml структура

```yaml
# Основная структура .gitlab-ci.yml

# Стадии выполнения
stages:
  - build
  - test
  - deploy

# Глобальные переменные
variables:
  MYSQL_ROOT_PASSWORD: secret
  MYSQL_DATABASE: testdb
  APP_ENV: testing

# Кеширование для ускорения
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - node_modules/

# Before script для всех jobs
before_script:
  - php -v
  - composer install --no-interaction --prefer-dist

# Job: Сборка приложения
build:
  stage: build
  image: php:8.2
  script:
    - composer install
    - npm install
    - npm run build
  artifacts:
    paths:
      - public/build/
      - vendor/
    expire_in: 1 hour

# Job: Тестирование
test:unit:
  stage: test
  image: php:8.2
  services:
    - mysql:8.0
  script:
    - cp .env.testing .env
    - php artisan key:generate
    - php artisan migrate
    - vendor/bin/phpunit --coverage-text --colors=never
  coverage: '/^\s*Lines:\s*\d+.\d+\%/'

test:feature:
  stage: test
  image: php:8.2
  script:
    - vendor/bin/pest --coverage

# Code Quality
code_quality:
  stage: test
  image: php:8.2
  script:
    - vendor/bin/phpstan analyse --error-format=gitlab > phpstan.json
    - vendor/bin/php-cs-fixer fix --dry-run --diff
  artifacts:
    reports:
      codequality: phpstan.json

# Деплой на staging
deploy:staging:
  stage: deploy
  image: alpine:latest
  before_script:
    - apk add --no-cache openssh-client
    - eval $(ssh-agent -s)
    - echo "$SSH_PRIVATE_KEY" | tr -d '\r' | ssh-add -
    - mkdir -p ~/.ssh
    - chmod 700 ~/.ssh
  script:
    - ssh user@staging.example.com "cd /var/www && git pull && composer install --no-dev && php artisan migrate --force"
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

# Деплой на production
deploy:production:
  stage: deploy
  image: alpine:latest
  script:
    - ssh user@production.example.com "cd /var/www && git pull && composer install --no-dev --optimize-autoloader && php artisan migrate --force && php artisan optimize:clear"
  environment:
    name: production
    url: https://example.com
  only:
    - main
  when: manual  # Ручное подтверждение
```

### Расширенные возможности GitLab CI

```yaml
# Использование includes для переиспользования конфигураций
include:
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/test.yml'
  - template: Security/SAST.gitlab-ci.yml

# Динамические environments
deploy:review:
  stage: deploy
  script:
    - deploy review-app
  environment:
    name: review/$CI_COMMIT_REF_NAME
    url: https://$CI_COMMIT_REF_SLUG.review.example.com
    on_stop: stop_review
  only:
    - branches
  except:
    - main

stop_review:
  stage: deploy
  script:
    - destroy review-app
  environment:
    name: review/$CI_COMMIT_REF_NAME
    action: stop
  when: manual

# Parallel jobs для ускорения
test:parallel:
  stage: test
  parallel: 5
  script:
    - vendor/bin/phpunit --group=slow --testdox

# Matrix builds для тестирования разных версий
test:matrix:
  stage: test
  parallel:
    matrix:
      - PHP_VERSION: ['8.1', '8.2', '8.3']
        LARAVEL_VERSION: ['10', '11']
  image: php:${PHP_VERSION}
  script:
    - composer require "laravel/framework:${LARAVEL_VERSION}.*"
    - vendor/bin/phpunit

# Needs для оптимизации зависимостей
deploy:fast:
  stage: deploy
  needs:
    - build
    - test:unit
  # Не ждет остальные test jobs
  script:
    - deploy.sh

# Rules для сложной логики запуска
deploy:production:
  stage: deploy
  script: deploy-prod.sh
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
    - if: '$CI_COMMIT_TAG =~ /^v\d+\.\d+\.\d+$/'
      when: on_success
    - when: never

# Retry для нестабильных jobs
test:flaky:
  stage: test
  script: flaky-test.sh
  retry:
    max: 2
    when:
      - script_failure
      - unknown_failure

# Triggers для Multi-project pipelines
trigger:microservice:
  stage: deploy
  trigger:
    project: company/microservice
    branch: main
  only:
    - main
```

### GitLab Runner конфигурация

```toml
# /etc/gitlab-runner/config.toml

concurrent = 4  # Количество параллельных jobs

[[runners]]
  name = "docker-runner"
  url = "https://gitlab.com/"
  token = "PROJECT_TOKEN"
  executor = "docker"
  
  [runners.docker]
    tls_verify = false
    image = "php:8.2"
    privileged = false
    disable_cache = false
    volumes = ["/cache", "/var/run/docker.sock:/var/run/docker.sock"]
    shm_size = 0
    
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "s3.amazonaws.com"
      BucketName = "gitlab-runner-cache"
      BucketLocation = "us-east-1"

[[runners]]
  name = "shell-runner"
  url = "https://gitlab.com/"
  token = "PROJECT_TOKEN"
  executor = "shell"
```

## GitHub Actions

### Workflows структура

```yaml
# .github/workflows/ci.yml

name: CI/CD Pipeline

# Триггеры запуска
on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]
  schedule:
    - cron: '0 2 * * *'  # Каждый день в 2:00 AM
  workflow_dispatch:  # Ручной запуск

# Переменные окружения
env:
  PHP_VERSION: '8.2'
  NODE_VERSION: '18'

# Jobs
jobs:
  # Job 1: Build
  build:
    name: Build Application
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, xml, ctype, json, pdo, pdo_mysql
          coverage: xdebug
      
      - name: Cache Composer dependencies
        uses: actions/cache@v3
        with:
          path: vendor
          key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
          restore-keys: |
            ${{ runner.os }}-composer-
      
      - name: Install dependencies
        run: composer install --no-interaction --prefer-dist --optimize-autoloader
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install Node dependencies
        run: npm ci
      
      - name: Build assets
        run: npm run build
      
      - name: Upload artifacts
        uses: actions/upload-artifact@v3
        with:
          name: build-artifacts
          path: |
            public/build
            vendor
          retention-days: 1

  # Job 2: Tests
  test:
    name: Run Tests
    runs-on: ubuntu-latest
    needs: build
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testdb
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
      
      redis:
        image: redis:alpine
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s --health-retries=3
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
          extensions: mbstring, xml, ctype, json, pdo, pdo_mysql, redis
          coverage: xdebug
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
      
      - name: Prepare Laravel Application
        run: |
          cp .env.example .env
          php artisan key:generate
          php artisan migrate --force
      
      - name: Run PHPUnit Tests
        run: vendor/bin/phpunit --coverage-clover coverage.xml
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          file: ./coverage.xml
          fail_ci_if_error: true

  # Job 3: Code Quality
  code-quality:
    name: Code Quality Checks
    runs-on: ubuntu-latest
    needs: build
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ env.PHP_VERSION }}
      
      - name: Download artifacts
        uses: actions/download-artifact@v3
        with:
          name: build-artifacts
      
      - name: Run PHPStan
        run: vendor/bin/phpstan analyse
      
      - name: Run PHP CS Fixer
        run: vendor/bin/php-cs-fixer fix --dry-run --diff
      
      - name: Run Psalm
        run: vendor/bin/psalm --output-format=github

  # Job 4: Security Scan
  security:
    name: Security Scanning
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Security Checker
        uses: symfonycorp/security-checker-action@v5
      
      - name: Run Snyk
        uses: snyk/actions/php@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}

  # Job 5: Deploy Staging
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test, code-quality]
    if: github.ref == 'refs/heads/develop'
    environment:
      name: staging
      url: https://staging.example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Deploy to Staging Server
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.STAGING_HOST }}
          username: ${{ secrets.STAGING_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            cd /var/www/staging
            git pull origin develop
            composer install --no-dev --optimize-autoloader
            php artisan migrate --force
            php artisan optimize:clear
            php artisan queue:restart

  # Job 6: Deploy Production
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [test, code-quality, security]
    if: github.ref == 'refs/heads/main'
    environment:
      name: production
      url: https://example.com
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Create Release
        uses: actions/create-release@v1
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        with:
          tag_name: v${{ github.run_number }}
          release_name: Release ${{ github.run_number }}
          draft: false
          prerelease: false
      
      - name: Deploy to Production (Blue-Green)
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.PRODUCTION_HOST }}
          username: ${{ secrets.PRODUCTION_USER }}
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            /opt/deploy/blue-green-deploy.sh
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-deploy.yml
name: Reusable Deploy Workflow

on:
  workflow_call:
    inputs:
      environment:
        required: true
        type: string
      php-version:
        required: false
        type: string
        default: '8.2'
    secrets:
      deploy-key:
        required: true
      host:
        required: true

jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: ${{ inputs.environment }}
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: ${{ inputs.php-version }}
      
      - name: Deploy
        run: |
          echo "Deploying to ${{ inputs.environment }}"
          # Deploy logic here

# .github/workflows/main.yml - использование reusable workflow
name: Main Pipeline

on: [push]

jobs:
  deploy-staging:
    uses: ./.github/workflows/reusable-deploy.yml
    with:
      environment: staging
      php-version: '8.2'
    secrets:
      deploy-key: ${{ secrets.STAGING_KEY }}
      host: ${{ secrets.STAGING_HOST }}
```

### Composite Actions

```yaml
# .github/actions/setup-laravel/action.yml
name: 'Setup Laravel'
description: 'Setup PHP, Composer, and Laravel application'

inputs:
  php-version:
    description: 'PHP version'
    required: false
    default: '8.2'

runs:
  using: "composite"
  steps:
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ inputs.php-version }}
        extensions: mbstring, xml, ctype, json, pdo, pdo_mysql
    
    - name: Install Composer dependencies
      run: composer install --no-interaction --prefer-dist
      shell: bash
    
    - name: Prepare Laravel
      run: |
        cp .env.example .env
        php artisan key:generate
      shell: bash

# Использование в workflow
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-laravel
        with:
          php-version: '8.2'
      - run: vendor/bin/phpunit
```

## Jenkins

### Jenkinsfile (Declarative Pipeline)

```groovy
// Jenkinsfile

pipeline {
    agent any
    
    environment {
        PHP_VERSION = '8.2'
        COMPOSER_HOME = "${WORKSPACE}/.composer"
        APP_ENV = 'testing'
    }
    
    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timeout(time: 30, unit: 'MINUTES')
        timestamps()
        ansiColor('xterm')
    }
    
    triggers {
        pollSCM('H/5 * * * *')  // Каждые 5 минут
        cron('H 2 * * *')       // Каждый день в 2:00
    }
    
    stages {
        stage('Checkout') {
            steps {
                checkout scm
                sh 'git log -1'
            }
        }
        
        stage('Build') {
            parallel {
                stage('PHP Dependencies') {
                    steps {
                        sh """
                            composer install \
                                --no-interaction \
                                --prefer-dist \
                                --optimize-autoloader
                        """
                    }
                }
                
                stage('Node Dependencies') {
                    steps {
                        sh 'npm ci'
                        sh 'npm run build'
                    }
                }
            }
        }
        
        stage('Code Quality') {
            parallel {
                stage('PHPStan') {
                    steps {
                        sh 'vendor/bin/phpstan analyse --error-format=checkstyle > phpstan-report.xml || true'
                        recordIssues(
                            tools: [phpStan(pattern: 'phpstan-report.xml')],
                            qualityGates: [[threshold: 10, type: 'TOTAL', unstable: true]]
                        )
                    }
                }
                
                stage('PHP CS Fixer') {
                    steps {
                        sh 'vendor/bin/php-cs-fixer fix --dry-run --format=checkstyle > cs-fixer-report.xml || true'
                    }
                }
                
                stage('Psalm') {
                    steps {
                        sh 'vendor/bin/psalm --output-format=checkstyle > psalm-report.xml || true'
                    }
                }
            }
        }
        
        stage('Test') {
            parallel {
                stage('Unit Tests') {
                    steps {
                        sh """
                            cp .env.testing .env
                            php artisan key:generate
                            vendor/bin/phpunit \
                                --coverage-html coverage \
                                --coverage-clover coverage.xml \
                                --log-junit junit.xml
                        """
                    }
                    post {
                        always {
                            junit 'junit.xml'
                            publishHTML([
                                reportDir: 'coverage',
                                reportFiles: 'index.html',
                                reportName: 'Code Coverage'
                            ])
                        }
                    }
                }
                
                stage('Feature Tests') {
                    steps {
                        sh 'vendor/bin/pest --parallel'
                    }
                }
            }
        }
        
        stage('Security Scan') {
            steps {
                sh 'composer audit'
                sh 'npm audit'
            }
        }
        
        stage('Build Docker Image') {
            when {
                branch 'main'
            }
            steps {
                script {
                    def image = docker.build("myapp:${env.BUILD_NUMBER}")
                    docker.withRegistry('https://registry.example.com', 'docker-credentials') {
                        image.push()
                        image.push('latest')
                    }
                }
            }
        }
        
        stage('Deploy Staging') {
            when {
                branch 'develop'
            }
            steps {
                sshagent(['staging-ssh-key']) {
                    sh """
                        ssh user@staging.example.com '
                            cd /var/www/staging &&
                            git pull origin develop &&
                            composer install --no-dev --optimize-autoloader &&
                            php artisan migrate --force &&
                            php artisan optimize:clear
                        '
                    """
                }
            }
        }
        
        stage('Deploy Production') {
            when {
                branch 'main'
            }
            steps {
                input message: 'Deploy to Production?', ok: 'Deploy'
                
                sshagent(['production-ssh-key']) {
                    sh """
                        ssh user@production.example.com '
                            /opt/deploy/blue-green-deploy.sh
                        '
                    """
                }
            }
        }
    }
    
    post {
        always {
            cleanWs()
        }
        
        success {
            slackSend(
                color: 'good',
                message: "Build Successful: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
        }
        
        failure {
            slackSend(
                color: 'danger',
                message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
            )
            emailext(
                subject: "Build Failed: ${env.JOB_NAME}",
                body: "Build ${env.BUILD_NUMBER} failed. Check console output.",
                to: 'team@example.com'
            )
        }
    }
}
```

### Jenkinsfile (Scripted Pipeline)

```groovy
// Более гибкий, но сложнее

node {
    try {
        stage('Checkout') {
            checkout scm
        }
        
        stage('Build') {
            sh 'composer install'
            sh 'npm ci && npm run build'
        }
        
        stage('Test') {
            parallel(
                'Unit Tests': {
                    sh 'vendor/bin/phpunit'
                },
                'Static Analysis': {
                    sh 'vendor/bin/phpstan analyse'
                }
            )
        }
        
        if (env.BRANCH_NAME == 'main') {
            stage('Deploy') {
                input 'Deploy to production?'
                sh './deploy.sh production'
            }
        }
        
        currentBuild.result = 'SUCCESS'
    } catch (err) {
        currentBuild.result = 'FAILURE'
        throw err
    } finally {
        // Cleanup
        cleanWs()
        
        // Notifications
        if (currentBuild.result == 'SUCCESS') {
            slackSend color: 'good', message: 'Build successful'
        } else {
            slackSend color: 'danger', message: 'Build failed'
        }
    }
}
```

## Docker в CI/CD

### Multi-stage Dockerfile для CI/CD

```dockerfile
# syntax=docker/dockerfile:1

# Базовый образ с PHP и расширениями
FROM php:8.2-fpm-alpine AS base

RUN apk add --no-cache \
    postgresql-dev \
    zip \
    libzip-dev \
    libpng-dev \
    libjpeg-turbo-dev \
    freetype-dev \
    icu-dev

RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install pdo pdo_pgsql gd zip intl opcache

WORKDIR /var/www/html


# Стадия для установки зависимостей
FROM base AS dependencies

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

COPY composer.json composer.lock ./

RUN composer install \
    --no-dev \
    --no-scripts \
    --no-autoloader \
    --prefer-dist \
    --optimize-autoloader


# Стадия для сборки фронтенда
FROM node:18-alpine AS frontend

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .
RUN npm run build


# Стадия для тестирования (используется в CI)
FROM base AS test

COPY --from=composer:2 /usr/bin/composer /usr/bin/composer

# Установка dev зависимостей для тестов
COPY composer.json composer.lock ./
RUN composer install --prefer-dist

COPY . .

# Запуск тестов
RUN vendor/bin/phpunit
RUN vendor/bin/phpstan analyse
RUN vendor/bin/php-cs-fixer fix --dry-run


# Production образ
FROM base AS production

# Оптимизации PHP для production
RUN echo "opcache.enable=1" >> /usr/local/etc/php/conf.d/opcache.ini && \
    echo "opcache.memory_consumption=256" >> /usr/local/etc/php/conf.d/opcache.ini && \
    echo "opcache.max_accelerated_files=20000" >> /usr/local/etc/php/conf.d/opcache.ini && \
    echo "opcache.validate_timestamps=0" >> /usr/local/etc/php/conf.d/opcache.ini

# Копирование зависимостей
COPY --from=dependencies /var/www/html/vendor ./vendor

# Копирование собранного фронтенда
COPY --from=frontend /app/public/build ./public/build

# Копирование кода приложения
COPY . .

# Генерация autoloader для production
RUN composer dump-autoload --optimize --classmap-authoritative

# Оптимизация Laravel
RUN php artisan config:cache && \
    php artisan route:cache && \
    php artisan view:cache

# Установка прав
RUN chown -R www-data:www-data /var/www/html/storage /var/www/html/bootstrap/cache

USER www-data

EXPOSE 9000

CMD ["php-fpm"]
```

### Docker Compose для CI/CD

```yaml
# docker-compose.ci.yml

version: '3.8'

services:
  app:
    build:
      context: .
      target: test
      cache_from:
        - registry.example.com/myapp:latest
    environment:
      APP_ENV: testing
      DB_CONNECTION: pgsql
      DB_HOST: postgres
      REDIS_HOST: redis
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    volumes:
      - .:/var/www/html
    command: vendor/bin/phpunit

  postgres:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: testdb
      POSTGRES_USER: testuser
      POSTGRES_PASSWORD: testpass
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U testuser"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 5s
      retries: 5

# Использование в CI
# docker-compose -f docker-compose.ci.yml up --abort-on-container-exit
```

### BuildKit и кеширование

```yaml
# .github/workflows/docker.yml

name: Docker Build

on: [push]

jobs:
  build:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3
      
      - name: Login to Registry
        uses: docker/login-action@v3
        with:
          registry: registry.example.com
          username: ${{ secrets.REGISTRY_USERNAME }}
          password: ${{ secrets.REGISTRY_PASSWORD }}
      
      - name: Build and push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: |
            registry.example.com/myapp:latest
            registry.example.com/myapp:${{ github.sha }}
          cache-from: type=registry,ref=registry.example.com/myapp:cache
          cache-to: type=registry,ref=registry.example.com/myapp:cache,mode=max
          build-args: |
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
            VCS_REF=${{ github.sha }}
```

## Тестирование в CI/CD

### Автоматизация тестов

```yaml
# .gitlab-ci.yml - комплексное тестирование

test:unit:
  stage: test
  script:
    - vendor/bin/phpunit --testsuite=Unit --coverage-text
  coverage: '/^\s*Lines:\s*\d+.\d+\%/'

test:feature:
  stage: test
  services:
    - mysql:8.0
    - redis:alpine
  script:
    - php artisan migrate --seed
    - vendor/bin/phpunit --testsuite=Feature

test:integration:
  stage: test
  services:
    - mysql:8.0
    - redis:alpine
    - mailhog/mailhog:latest
  script:
    - vendor/bin/pest --group=integration

test:e2e:
  stage: test
  image: cypress/browsers:node18.12.0-chrome107
  script:
    - npm ci
    - npm run build
    - npm run start &  # Запуск приложения
    - npx wait-on http://localhost:8000
    - npx cypress run --record --key $CYPRESS_RECORD_KEY
  artifacts:
    when: on_failure
    paths:
      - cypress/screenshots
      - cypress/videos

test:performance:
  stage: test
  image: grafana/k6
  script:
    - k6 run --vus 100 --duration 30s tests/performance/load-test.js

test:security:
  stage: test
  script:
    - composer audit
    - npm audit
    - vendor/bin/psalm --taint-analysis
```

### Parallel Testing

```php
// phpunit.xml - параллельное тестирование

<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true"
         processIsolation="false"
         stopOnFailure="false"
         cacheDirectory=".phpunit.cache">
    
    <testsuites>
        <testsuite name="Unit">
            <directory>tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory>tests/Feature</directory>
        </testsuite>
    </testsuites>
    
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
    </php>
    
    <extensions>
        <bootstrap class="ParaTest\Extension"/>
    </extensions>
</phpunit>
```

```yaml
# GitHub Actions - параллельные тесты

test:
  strategy:
    matrix:
      test-suite: [Unit, Feature, Integration]
      php-version: ['8.1', '8.2', '8.3']
  
  steps:
    - uses: actions/checkout@v4
    
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ matrix.php-version }}
    
    - name: Run tests
      run: vendor/bin/paratest --testsuite=${{ matrix.test-suite }} --processes=4
```

## Деплой стратегии

### Blue-Green Deployment

```bash
#!/bin/bash
# blue-green-deploy.sh

set -e

BLUE_CONTAINER="app-blue"
GREEN_CONTAINER="app-green"
NGINX_CONFIG="/etc/nginx/conf.d/app.conf"

# Определяем текущий активный контейнер
if docker ps | grep -q $BLUE_CONTAINER; then
    ACTIVE=$BLUE_CONTAINER
    INACTIVE=$GREEN_CONTAINER
    ACTIVE_COLOR="blue"
    INACTIVE_COLOR="green"
else
    ACTIVE=$GREEN_CONTAINER
    INACTIVE=$BLUE_CONTAINER
    ACTIVE_COLOR="green"
    INACTIVE_COLOR="blue"
fi

echo "Current active: $ACTIVE_COLOR"
echo "Deploying to: $INACTIVE_COLOR"

# Деплой в неактивный контейнер
docker-compose up -d $INACTIVE

# Проверка здоровья
echo "Health check..."
for i in {1..30}; do
    if curl -f http://localhost:808${INACTIVE_COLOR:0:1}/health; then
        echo "Health check passed"
        break
    fi
    if [ $i -eq 30 ]; then
        echo "Health check failed"
        docker-compose stop $INACTIVE
        exit 1
    fi
    sleep 2
done

# Smoke tests
echo "Running smoke tests..."
./tests/smoke-tests.sh http://localhost:808${INACTIVE_COLOR:0:1}

# Переключение nginx
echo "Switching traffic to $INACTIVE_COLOR"
sed -i "s/$ACTIVE_COLOR/$INACTIVE_COLOR/g" $NGINX_CONFIG
nginx -s reload

# Даем время на graceful shutdown
sleep 10

# Остановка старого контейнера
docker-compose stop $ACTIVE

echo "Deployment complete. $INACTIVE_COLOR is now active"
```

### Canary Deployment

```yaml
# kubernetes canary deployment

apiVersion: v1
kind: Service
metadata:
  name: myapp
spec:
  selector:
    app: myapp
  ports:
    - port: 80
      targetPort: 8080

---
# Stable deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-stable
spec:
  replicas: 9
  selector:
    matchLabels:
      app: myapp
      version: stable
  template:
    metadata:
      labels:
        app: myapp
        version: stable
    spec:
      containers:
      - name: myapp
        image: myapp:v1.0.0
        ports:
        - containerPort: 8080

---
# Canary deployment (10% traffic)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-canary
spec:
  replicas: 1  # 10% от stable
  selector:
    matchLabels:
      app: myapp
      version: canary
  template:
    metadata:
      labels:
        app: myapp
        version: canary
    spec:
      containers:
      - name: myapp
        image: myapp:v1.1.0
        ports:
        - containerPort: 8080
```

### Rolling Update

```yaml
# GitLab CI - rolling update

deploy:rolling:
  stage: deploy
  script:
    - |
      # Обновление по одному серверу за раз
      for server in server1 server2 server3; do
        echo "Deploying to $server"
        
        # Вывод из балансировщика
        ssh lb "remove_from_pool $server"
        sleep 5
        
        # Деплой
        ssh $server "cd /var/www && git pull && composer install --no-dev"
        ssh $server "php artisan migrate --force"
        ssh $server "php artisan optimize:clear"
        ssh $server "sudo systemctl restart php-fpm"
        
        # Проверка
        if ssh $server "curl -f http://localhost/health"; then
          echo "$server healthy"
        else
          echo "$server unhealthy - rollback"
          ssh $server "git checkout HEAD~1"
          exit 1
        fi
        
        # Возврат в pool
        ssh lb "add_to_pool $server"
        
        echo "$server deployed successfully"
        sleep 10  # Даем время на прогрев
      done
```

### Feature Flags в CI/CD

```php
// app/Services/FeatureFlagService.php

namespace App\Services;

class FeatureFlagService
{
    public function isEnabled(string $feature, ?User $user = null): bool
    {
        // Проверка через переменные окружения (для CI/CD)
        if (config("features.$feature.enabled") === true) {
            return true;
        }
        
        // Проверка через процент пользователей (Canary)
        $percentage = config("features.$feature.percentage", 0);
        if ($percentage > 0 && $user) {
            $hash = crc32($user->id . $feature);
            return ($hash % 100) < $percentage;
        }
        
        // Проверка через LaunchDarkly/Unleash и т.д.
        return app('launchdarkly')->isEnabled($feature, $user);
    }
}

// Использование в коде
if (app(FeatureFlagService::class)->isEnabled('new_checkout', auth()->user())) {
    return view('checkout.new');
}

return view('checkout.old');
```

```yaml
# CI/CD с feature flags

deploy:canary:
  stage: deploy
  script:
    - |
      # Включаем новую фичу для 10% пользователей
      kubectl set env deployment/myapp \
        FEATURE_NEW_CHECKOUT_ENABLED=true \
        FEATURE_NEW_CHECKOUT_PERCENTAGE=10

deploy:full:
  stage: deploy
  when: manual
  script:
    - |
      # После проверки метрик - включаем для всех
      kubectl set env deployment/myapp \
        FEATURE_NEW_CHECKOUT_PERCENTAGE=100
```

## Мониторинг и метрики

### Healthcheck endpoints

```php
// routes/api.php

Route::get('/health', function () {
    $checks = [
        'database' => checkDatabase(),
        'redis' => checkRedis(),
        'queue' => checkQueue(),
        'storage' => checkStorage(),
    ];
    
    $healthy = !in_array(false, $checks, true);
    $status = $healthy ? 200 : 503;
    
    return response()->json([
        'status' => $healthy ? 'healthy' : 'unhealthy',
        'timestamp' => now()->toIso8601String(),
        'checks' => $checks,
        'version' => config('app.version'),
    ], $status);
});

Route::get('/ready', function () {
    // Readiness probe для Kubernetes
    // Проверяет готовность принимать трафик
    
    try {
        DB::connection()->getPdo();
        Cache::get('test');
        
        return response()->json(['status' => 'ready']);
    } catch (\Exception $e) {
        return response()->json([
            'status' => 'not ready',
            'error' => $e->getMessage()
        ], 503);
    }
});

function checkDatabase(): bool
{
    try {
        DB::connection()->getPdo();
        return true;
    } catch (\Exception $e) {
        Log::error('Database health check failed', ['error' => $e->getMessage()]);
        return false;
    }
}

function checkRedis(): bool
{
    try {
        Redis::ping();
        return true;
    } catch (\Exception $e) {
        Log::error('Redis health check failed', ['error' => $e->getMessage()]);
        return false;
    }
}
```

### Метрики деплоя

```yaml
# Prometheus metrics для деплоя

apiVersion: v1
kind: ConfigMap
metadata:
  name: deployment-metrics
data:
  metrics.sh: |
    #!/bin/bash
    
    # Метрики времени деплоя
    START_TIME=$(date +%s)
    
    # Деплой
    ./deploy.sh
    
    END_TIME=$(date +%s)
    DURATION=$((END_TIME - START_TIME))
    
    # Отправка метрик в Prometheus
    echo "deployment_duration_seconds $DURATION" | \
      curl --data-binary @- http://pushgateway:9091/metrics/job/deployment
    
    # Отправка события в Grafana
    curl -X POST https://grafana.com/api/annotations \
      -H "Authorization: Bearer $GRAFANA_TOKEN" \
      -H "Content-Type: application/json" \
      -d '{
        "text": "Deployment completed",
        "tags": ["deployment", "production"],
        "time": '$(($END_TIME * 1000))'
      }'
```

### Уведомления о деплое

```yaml
# .gitlab-ci.yml notifications

deploy:production:
  stage: deploy
  script:
    - ./deploy.sh
  after_script:
    # Slack notification
    - |
      curl -X POST $SLACK_WEBHOOK \
        -H 'Content-Type: application/json' \
        -d '{
          "text": "🚀 Production Deployment",
          "attachments": [{
            "color": "good",
            "fields": [
              {"title": "Environment", "value": "Production", "short": true},
              {"title": "Version", "value": "'$CI_COMMIT_TAG'", "short": true},
              {"title": "Deployed by", "value": "'$GITLAB_USER_NAME'", "short": true},
              {"title": "Pipeline", "value": "'$CI_PIPELINE_URL'", "short": false}
            ]
          }]
        }'
    
    # Sentry release tracking
    - |
      curl https://sentry.io/api/0/organizations/myorg/releases/ \
        -X POST \
        -H "Authorization: Bearer $SENTRY_TOKEN" \
        -H 'Content-Type: application/json' \
        -d '{
          "version": "'$CI_COMMIT_SHA'",
          "projects": ["myapp"],
          "refs": [{
            "repository": "'$CI_PROJECT_PATH'",
            "commit": "'$CI_COMMIT_SHA'"
          }]
        }'
    
    # DataDog deployment event
    - |
      curl -X POST "https://api.datadoghq.com/api/v1/events" \
        -H "DD-API-KEY: $DATADOG_API_KEY" \
        -H "Content-Type: application/json" \
        -d '{
          "title": "Production Deployment",
          "text": "Deployed version '$CI_COMMIT_TAG' to production",
          "tags": ["environment:production", "service:myapp"]
        }'
```

## Безопасность в CI/CD

### Управление секретами

```yaml
# GitLab CI - использование секретов

variables:
  # Никогда не храните секреты в коде!
  # Используйте CI/CD variables

deploy:
  script:
    # Секреты из GitLab CI/CD variables
    - echo $DATABASE_PASSWORD | docker login -u $DATABASE_USER --password-stdin
    
    # Секреты из HashiCorp Vault
    - export DATABASE_PASSWORD=$(vault kv get -field=password secret/database)
    
    # Секреты из AWS Secrets Manager
    - export API_KEY=$(aws secretsmanager get-secret-value --secret-id api-key --query SecretString --output text)
    
    # Секреты из .env.encrypted (encrypted with git-crypt)
    - git-crypt unlock /path/to/key
    - cp .env.encrypted .env
```

### Сканирование на уязвимости

```yaml
# .gitlab-ci.yml - security scanning

include:
  - template: Security/SAST.gitlab-ci.yml
  - template: Security/Dependency-Scanning.gitlab-ci.yml
  - template: Security/Container-Scanning.gitlab-ci.yml
  - template: Security/Secret-Detection.gitlab-ci.yml

# Custom security scans

security:composer:
  stage: test
  script:
    - composer audit
  allow_failure: false

security:npm:
  stage: test
  script:
    - npm audit --audit-level=high
  allow_failure: false

security:snyk:
  stage: test
  image: snyk/snyk:php
  script:
    - snyk test --severity-threshold=high
    - snyk monitor
  only:
    - main

security:trivy:
  stage: test
  image: aquasec/trivy
  script:
    - trivy image --severity HIGH,CRITICAL myapp:latest

security:owasp:
  stage: test
  image: owasp/zap2docker-stable
  script:
    - zap-baseline.py -t https://staging.example.com -r owasp-report.html
  artifacts:
    paths:
      - owasp-report.html
```

### Image signing

```yaml
# Docker Content Trust

deploy:
  variables:
    DOCKER_CONTENT_TRUST: 1
  before_script:
    - echo $DOCKER_TRUST_PRIVATE_KEY | base64 -d > key.pem
    - docker trust key load key.pem
  script:
    - docker build -t myapp:$CI_COMMIT_SHA .
    - docker push myapp:$CI_COMMIT_SHA
    - docker trust sign myapp:$CI_COMMIT_SHA
```

## Оптимизация пайплайнов

### Кеширование

```yaml
# GitLab CI - эффективное кеширование

cache:
  key:
    files:
      - composer.lock
      - package-lock.json
  paths:
    - vendor/
    - node_modules/
  policy: pull-push

build:
  stage: build
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - vendor/
      - .composer-cache/
  script:
    - composer install --prefer-dist --no-progress
  artifacts:
    paths:
      - vendor/
    expire_in: 1 hour

test:
  stage: test
  cache:
    key: $CI_COMMIT_REF_SLUG
    paths:
      - vendor/
    policy: pull  # Только чтение кеша
  dependencies:
    - build
  script:
    - vendor/bin/phpunit
```

### Артефакты

```yaml
# GitHub Actions - artifacts

build:
  steps:
    - name: Build
      run: |
        composer install
        npm run build
    
    - name: Upload artifacts
      uses: actions/upload-artifact@v3
      with:
        name: build-${{ github.sha }}
        path: |
          vendor/
          public/build/
        retention-days: 1

test:
  needs: build
  steps:
    - name: Download artifacts
      uses: actions/download-artifact@v3
      with:
        name: build-${{ github.sha }}
    
    - name: Run tests
      run: vendor/bin/phpunit
```

### Параллелизация

```yaml
# GitLab CI - parallel jobs

test:
  stage: test
  parallel: 5
  script:
    - vendor/bin/paratest --processes=4

# Matrix builds
test:matrix:
  stage: test
  parallel:
    matrix:
      - PHP: ['8.1', '8.2', '8.3']
        LARAVEL: ['10', '11']
  image: php:${PHP}
  script:
    - composer require "laravel/framework:^${LARAVEL}"
    - vendor/bin/phpunit
```

## Лучшие практики

### Принципы эффективного CI/CD

```yaml
# 1. Fail Fast - быстрое обнаружение проблем
stages:
  - lint      # Самые быстрые проверки первыми
  - test      # Тесты
  - build     # Сборка
  - deploy    # Деплой

lint:
  stage: lint
  script:
    - vendor/bin/phpstan analyse --error-format=raw --no-progress
    - vendor/bin/php-cs-fixer fix --dry-run --diff
  # Если lint не прошел - прерываем весь pipeline

# 2. Идемпотентность - повторный запуск дает тот же результат
deploy:
  script:
    # ❌ Плохо - добавляет строку при каждом запуске
    - echo "server_name example.com;" >> nginx.conf
    
    # ✅ Хорошо - idempotent
    - sed -i 's/old_server/new_server/g' nginx.conf
    - ansible-playbook deploy.yml  # Ansible идемпотентен

# 3. Прозрачность - понятные логи и статусы
test:
  script:
    - echo "Starting unit tests..."
    - vendor/bin/phpunit --testdox
    - echo "Unit tests completed successfully"

# 4. Безопасность - никаких секретов в логах
deploy:
  script:
    - echo "Deploying to production..."  # ✅
    - deploy.sh
    # ❌ НЕ выводите: echo "Password: $SECRET_PASSWORD"
  
  # Скрытие чувствительных переменных
  variables:
    SECRET_KEY:
      value: "secret"
      masked: true
```

### Структура репозитория

```
project/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml              # Основной CI pipeline
│   │   ├── deploy-staging.yml  # Деплой на staging
│   │   └── deploy-prod.yml     # Деплой на production
│   └── actions/
│       └── setup-laravel/      # Custom action
│           └── action.yml
├── .gitlab/
│   └── ci/
│       ├── build.yml
│       ├── test.yml
│       └── deploy.yml
├── docker/
│   ├── Dockerfile
│   ├── Dockerfile.dev
│   ├── docker-compose.yml
│   └── docker-compose.ci.yml
├── scripts/
│   ├── deploy.sh
│   ├── health-check.sh
│   └── rollback.sh
├── tests/
│   ├── Unit/
│   ├── Feature/
│   ├── E2E/
│   └── performance/
├── .gitlab-ci.yml
├── Jenkinsfile
└── README.md
```

### Pipeline как код - best practices

```yaml
# DRY - переиспользование конфигурации

.base_job:  # Template job
  image: php:8.2
  before_script:
    - composer install
  cache:
    key: ${CI_COMMIT_REF_SLUG}
    paths:
      - vendor/

# Использование template
test:unit:
  extends: .base_job
  script:
    - vendor/bin/phpunit --testsuite=Unit

test:feature:
  extends: .base_job
  script:
    - vendor/bin/phpunit --testsuite=Feature

# Якоря для переиспользования (YAML anchors)
.deploy_script: &deploy_script
  - ssh user@server "cd /var/www && git pull"
  - ssh user@server "composer install --no-dev"
  - ssh user@server "php artisan migrate --force"

deploy:staging:
  script: *deploy_script
  environment: staging

deploy:production:
  script: *deploy_script
  environment: production
  when: manual
```

## Вопросы на собеседовании

### Базовые вопросы

**Q: Что такое CI/CD и в чем разница между CI и CD?**
A:
- **CI (Continuous Integration)** - автоматическая сборка и тестирование при каждом коммите
- **CD (Continuous Delivery)** - автоматическая подготовка к релизу, ручной деплой
- **CD (Continuous Deployment)** - полностью автоматический деплой в production

**Q: Какие этапы обычно включает CI/CD pipeline?**
A:
1. Source - получение кода из репозитория
2. Build - сборка приложения (composer install, npm build)
3. Test - запуск тестов (unit, integration, e2e)
4. Quality Gates - статический анализ, code coverage
5. Package - создание артефактов (Docker images)
6. Deploy - развертывание на окружения
7. Monitor - мониторинг и алерты

**Q: Как кешировать зависимости в CI/CD?**
A:
```yaml
# GitLab CI
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - node_modules/

# GitHub Actions
- uses: actions/cache@v3
  with:
    path: vendor
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
```

### Средние вопросы

**Q: Объясните разницу между артефактами и кешем в CI/CD**
A:
- **Cache** - для зависимостей, которые не меняются часто (vendor/, node_modules/)
  - Оптимизация скорости сборки
  - Может быть удален без последствий
  
- **Artifacts** - для результатов сборки, которые нужны другим jobs
  - Передача данных между jobs
  - Важны для pipeline

```yaml
# Cache - для зависимостей
cache:
  paths:
    - vendor/

# Artifacts - для результатов сборки
artifacts:
  paths:
    - public/build/
    - coverage/
  expire_in: 1 day
```

**Q: Как организовать деплой на несколько окружений?**
A:
```yaml
.deploy_template:
  script:
    - deploy.sh $ENVIRONMENT
  only:
    - main
    - develop

deploy:staging:
  extends: .deploy_template
  environment:
    name: staging
    url: https://staging.example.com
  only:
    - develop

deploy:production:
  extends: .deploy_template
  environment:
    name: production
    url: https://example.com
  when: manual
  only:
    - main
```

**Q: Как обрабатывать секреты в CI/CD?**
A:
1. **CI/CD Variables** - хранить в настройках CI/CD
2. **Vault** - HashiCorp Vault, AWS Secrets Manager
3. **Encrypted files** - git-crypt, SOPS
4. **Environment variables** - никогда не логировать!

```yaml
deploy:
  variables:
    DB_PASSWORD: "secret"  # ❌ Плохо!
  script:
    - echo $DB_PASSWORD    # ❌ Попадет в логи!

deploy:
  script:
    - export DB_PASSWORD=$(vault kv get secret/db)  # ✅ Хорошо
    - deploy.sh  # Использует DB_PASSWORD внутри
```

### Продвинутые вопросы

**Q: Объясните Blue-Green и Canary deployment. Когда что использовать?**
A:

**Blue-Green:**
- Две идентичные среды (blue и green)
- Переключение происходит мгновенно
- Легкий откат
- Дорого (требует 2x ресурсов)

```bash
# Текущий: blue активен
# 1. Деплой в green
# 2. Тестирование green
# 3. Переключение трафика на green
# 4. Blue становится неактивным
```

**Canary:**
- Постепенное перенаправление трафика на новую версию
- Мониторинг метрик
- Меньше рисков
- Сложнее настройка

```bash
# 1. Деплой canary (10% пользователей)
# 2. Мониторинг 30 минут
# 3. Увеличение до 50%
# 4. Полный переход 100%
```

**Q: Как оптимизировать время выполнения CI/CD pipeline?**
A:

1. **Параллелизация:**
```yaml
test:
  parallel: 5
  script:
    - vendor/bin/paratest --processes=4
```

2. **Кеширование:**
```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - vendor/
    - .composer-cache/
```

3. **Docker layer caching:**
```dockerfile
FROM php:8.2

# Сначала зависимости (меняются редко)
COPY composer.json composer.lock ./
RUN composer install

# Потом код (меняется часто)
COPY . .
```

4. **Fail fast:**
```yaml
stages:
  - lint        # 30 секунд
  - test        # 5 минут
  - build       # 10 минут
  - deploy      # только если все прошло
```

5. **Needs для DAG:**
```yaml
deploy:
  needs: [build, test:unit]  # Не ждать test:e2e
```

**Q: Как реализовать GitOps подход?**
A:

GitOps - все конфигурации инфраструктуры в Git:

```yaml
# Infrastructure as Code
infrastructure/
├── terraform/
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── kubernetes/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── ingress.yaml
└── ansible/
    └── playbook.yml

# ArgoCD для автоматического деплоя при изменении в Git
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: myapp
spec:
  source:
    repoURL: https://github.com/user/repo
    path: kubernetes/
    targetRevision: main
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

**Q: Как тестировать сам CI/CD pipeline?**
A:

1. **Локальное тестирование:**
```bash
# GitLab CI - запуск локально
gitlab-runner exec docker test

# GitHub Actions - act
act -j test

# Docker Compose
docker-compose -f docker-compose.ci.yml up
```

2. **Feature branches:**
```yaml
# Полный pipeline только для main/develop
# Упрощенный для feature branches
test:full:
  script: run-all-tests.sh
  only:
    - main
    - develop

test:quick:
  script: run-unit-tests.sh
  except:
    - main
    - develop
```

3. **Staged rollouts:**
```yaml
deploy:canary:
  script: deploy-10-percent.sh
  
deploy:full:
  when: manual  # Ручное подтверждение
  script: deploy-100-percent.sh
```

**Q: Как обеспечить compliance и аудит в CI/CD?**
A:

1. **Логирование всех деплоев:**
```yaml
deploy:
  after_script:
    - |
      curl -X POST $AUDIT_API/deployments \
        -d '{
          "user": "'$CI_COMMIT_AUTHOR'",
          "commit": "'$CI_COMMIT_SHA'",
          "environment": "production",
          "timestamp": "'$(date -Iseconds)'"
        }'
```

2. **Approval gates:**
```yaml
deploy:production:
  when: manual
  environment:
    name: production
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
      allow_failure: false
```

3. **Immutable artifacts:**
```yaml
build:
  script:
    - docker build -t myapp:${CI_COMMIT_SHA} .
    - docker push myapp:${CI_COMMIT_SHA}
  # Используем SHA, а не latest
```

4. **Audit trail:**
```yaml
stages:
  - audit-start
  - build
  - test
  - deploy
  - audit-end

audit-start:
  script:
    - log-audit-event "Pipeline started"

audit-end:
  script:
    - log-audit-event "Pipeline completed"
  when: always
```

---

## Заключение

CI/CD - это не просто инструменты, а культура и практики непрерывной доставки качественного софта. Ключевые принципы:

1. **Автоматизация** - минимум ручных операций
2. **Скорость** - быстрая обратная связь
3. **Надежность** - уверенность в каждом деплое
4. **Безопасность** - проверки на каждом этапе
5. **Мониторинг** - видимость всего процесса

Эффективный CI/CD pipeline позволяет команде фокусироваться на разработке, а не на рутинных задачах развертывания.
