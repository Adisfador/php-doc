# PHP Middle/Senior Interview Preparation

Комплексный план подготовки к собеседованиям. Акцент на глубокое понимание PHP и баз данных.

---

## 🗄️ Базы данных (deep dive)
- 🔴 [SQL Синтаксис](databases/sql-syntax.md) - SELECT, JOIN, GROUP BY, HAVING, подзапросы, CTE, оконные функции, триггеры, процедуры
- 🔴 [SQL базы данных: PostgreSQL & MySQL](databases/sql-databases.md) - синтаксис, конструкции, диалекты
- 🔴 [Физическое хранение данных](databases/physical-storage.md) - страницы, файлы, B-Tree структуры
- 🔴 [Индексы](databases/indexes.md) - все типы, внутреннее устройство, оптимизация
- 🔴 [Транзакции и изоляция](databases/transactions.md) - ACID, уровни изоляции, аномалии, блокировки
- 🔴 [Оптимизация и обслуживание](databases/optimization.md) - VACUUM, ANALYZE, статистика, производительность
- 🟡 [NoSQL базы данных](databases/nosql.md) - Redis, MongoDB, ClickHouse, Elasticsearch
- 🟡 [NewSQL базы данных](databases/newsql.md) - CockroachDB, Google Spanner, YugabyteDB - SQL + горизонтальное масштабирование
- 🟡 [Концепции БД](databases/concepts.md) - OLTP/OLAP, CAP теорема, репликация, шардирование
- 🔴 [Нормализация БД](databases/normalization.md) - нормальные формы 1NF-6NF, BCNF, денормализация, практические примеры
- 📚 [Обучающие ресурсы](databases/resources.md) - курсы, видео, книги по БД

---

## 🐘 PHP (deep dive)
- 🔴 [Основы языка](php/core-basics.md) - типы, операторы, функции, области видимости
- 🔴 [ООП в PHP](php/oop.md) - классы, интерфейсы, трейты, магические методы, позднее статическое связывание
- 🔴 [Типизация](php/types.md) - strict_types, type hints, union/intersection types, mixed, never
- 🔴 [SPL (Standard PHP Library)](php/spl.md) - итераторы, структуры данных, исключения
- 🔴 [Streams (Потоки)](php/streams.md) - stream wrappers, php://, фильтры, контексты, файлы
- 🔴 [Внутреннее устройство](php/internals.md) - Zend Engine, OpCache, JIT, memory management
- 🟢 [Composer и автозагрузка](php/composer.md) - PSR-4, версионирование, semver
- 🔴 [PSR стандарты](php/psr-standards.md) - PSR-1, 2, 3, 4, 6, 7, 11, 12, 15, 18
- 🔴 [Безопасность](php/security.md) - XSS, CSRF, SQL Injection, OWASP Top 10
- 🟡 [Производительность и профилирование](php/performance.md) - Xdebug, Blackfire, оптимизация
- 📚 [Обучающие ресурсы](php/resources.md) - курсы, видео, книги по PHP

---

## 🔴 Laravel (deep dive)
- 🔴 [Архитектура и жизненный цикл](laravel/architecture.md) - Request Lifecycle, Service Container, Service Providers
- 🔴 [Основные компоненты](laravel/core-components.md) - Routing, Middleware, Controllers, Requests
- 🔴 [Collections](laravel/collections.md) - Collection API, методы map/filter/pluck, lazy collections, higher order messages, macros
- 🔴 [Blade Templates](laravel/blade.md) - шаблонизатор, директивы, компоненты, slots, layouts, формы
- 🔴 [Authentication & Authorization](laravel/authentication.md) - Session auth, Sanctum API tokens, JWT, Guards, Policies, Gates
- 🔴 [Eloquent ORM](laravel/eloquent.md) - модели, отношения, eager loading, N+1 problem
- 🔴 [Database](laravel/database.md) - Query Builder, миграции, сиды, транзакции
- 🔴 [Очереди и Jobs](laravel/queues.md) - queue workers, failed jobs, job batching
- 🟡 [Кеширование](laravel/caching.md) - cache drivers, cache tags, events
- 🟡 [События и слушатели](laravel/events.md) - observers, listeners, broadcasting
- 📚 [Обучающие ресурсы](laravel/resources.md) - курсы, видео, книги по Laravel
- 🔴 [Тестирование](laravel/testing.md) - PHPUnit, Pest, Feature/Unit tests, мокирование

---

## 🏗️ Архитектура и паттерны
- 🔴 [Паттерны проектирования GoF](patterns/design-patterns.md) - 23 паттерна Gang of Four, применение в PHP
- 🔴 [GRASP принципы](patterns/grasp.md) - General Responsibility Assignment Software Patterns
- 🔴 [SOLID принципы](patterns/solid.md) - детальный разбор с примерами
- 🔴 [Архитектурные паттерны](patterns/architectural.md) - MVC, Repository, Service Layer, CQRS, Event Sourcing
- 🟡 [Hexagonal Architecture (Ports & Adapters)](patterns/hexagonal.md) - изоляция бизнес-логики
- 🟡 [Domain-Driven Design (DDD)](patterns/ddd.md) - bounded contexts, aggregates, value objects, entities
- 🟡 [Porto Architecture](patterns/porto.md) - архитектура для крупных Larave
- 📚 [Обучающие ресурсы](patterns/resources.md) - курсы, видео, книги по паттернамl проектов
- 🟢 [Чистый код](patterns/clean-code.md) - принципы, рефакторинг, именование

---

## 🌐 API и интеграции
- [REST API](api/rest.md) - принципы, HTTP методы, stateless, HATEOAS
- [GraphQL](api/graphql.md) - схемы, запросы, мутации, N+1 problem
- [gRPC](api/grpc.md) - protobuf, streaming, использование с PHP
- [Аутентификация](api/auth.md) - JWT, OAuth 2.0, API tokens

---

## 📮 Очереди и брокеры сообщений
- 🟡 [RabbitMQ](messaging/rabbitmq.md) - exchanges, queues, routing, паттерны
- 🟡 [Apache Kafka](messaging/kafka.md) - topics, partitions, consumer groups
- 🟡 [Паттерны работы с очередями](messaging/patterns.md) - retry, DLQ, idempotency

---

## 🌐 Сети и протоколы
- 🔴 [TCP/IP и OSI](networking/protocols.md) - все уровни, от физического до прикладного
- 🔴 [HTTP/HTTPS](networking/http.md) - методы GET/POST/PUT/PATCH/DELETE, статус коды, заголовки, HTTP/2 multiplexing, HTTP/3 QUIC, HTTPS TLS handshake, кеширование ETag/Cache-Control, CORS
- 🟢 [DNS и доменные имена](networking/dns.md) - что происходит при вводе URL
- 🟡 [SSH и SSL](networking/ssh-ssl.md) - ключи, сертификаты, handshake
- 🟡 [WebSockets](networking/websockets.md) - real-time коммуникация, upgrade от HTTP, авторизация

---

## 🐳 DevOps и инфраструктура
- 🔴 [Linux](devops/linux.md) - команды, файловая система, процессы, права, сеть
- 🔴 [Docker](devops/docker.md) - контейнеризация vs виртуализация, images, volumes, networks
- 🟡 [Kubernetes](devops/kubernetes.md) - pods, services, deployments, основы оркестрации
- 🔴 [Веб-серверы](devops/web-servers.md) - Nginx, Apache, устройство, отличия
- 🔴 [CGI и PHP серверы](devops/php-servers.md) - CGI, FastCGI, PHP-FPM, mod_php
- 🟡 [Swoole, RoadRunner, Laravel Octane](devops/swoole-octane-roadrunner.md) - persistent PHP процессы, application servers, FrankenPHP
- 🟡 [Git](devops/git.md) - продвинутые команды, стратегии ветвления, rebase vs merge
- 🟡 [CI/CD](devops/cicd.md) - пайплайны, автоматизация, best practices
- 📚 [Обучающие ресурсы](devops/resources.md) - курсы, видео, книги по DevOps

---

## 📊 Мониторинг и observability
- 🟡 [Профилирование](monitoring/profiling.md) - Xdebug, Blackfire, графы вызовов, анализ производительности
- 🟡 [Prometheus](monitoring/prometheus.md) - метрики, PromQL, алерты
- 🟡 [Grafana](monitoring/grafana.md) - дашборды, визуализация
- 🟡 [Loki](monitoring/loki.md) - логирование, запросы LogQL
- 📚 [Обучающие ресурсы](monitoring/resources.md) - курсы, видео, книги по мониторингу

---

## 🧮 Алгоритмы и структуры данных
- 🟢 [Big O и сложность](algorithms/complexity.md) - асимптотическая сложность, анализ алгоритмов
- � [PHP Helper Functions](algorithms/php-helpers.md) - полезные встроенные функции, сортировки (usort, uasort), примеры для алгоритмов
- �🟡 [LeetCode паттерны](algorithms/patterns.md) - Two Pointers, Sliding Window, DFS/BFS, DP шаблоны
- 🟡 [Структуры данных](algorithms/data-structures.md) - Array, LinkedList, Tree, Graph, Heap на PHP
- 📚 [Ресурсы для практики](algorithms/resources.md) - LeetCode, Blind 75, NeetCode 150, курсы

---

## ✅ Тестирование
- [PHPUnit](testing/phpunit.md) - unit tests, assertions, data providers
- [Pest](testing/pest.md) - современный синтаксис, expectations
- [Моки и стабы](testing/mocks.md) - doubles, spy, mock, stub
- [TDD](testing/tdd.md) - Test-Driven Development, best practices

---

## 🏗️ System Design и распределённые системы
- 🔴 [Fundamentals](system-design/fundamentals.md) - CAP теорема, consistency models, ACID vs BASE, availability, latency vs throughput
- 🔴 [Scalability](system-design/scalability.md) - load balancing, auto-scaling, caching layers, database scaling, CDN, connection pooling
- 🔴 [Interview Guide](system-design/interview-guide.md) - framework REASCH, примеры систем, capacity estimations, trade-offs
- 📚 [Обучающие ресурсы](system-design/resources.md) - книги, курсы, YouTube, платформы для практики

---

## 📋 Чек-листы
- [ ] [Middle Developer Checklist](checklists/middle.md)
- [ ] [Senior Developer Checklist](checklists/senior.md)

---

## 📝 Backlog и новые темы

**[📋 Backlog](backlog.md)** - файл для добавления новых тем, вопросов и концепций, которые появляются в процессе подготовки. Используй этот файл чтобы:
- Записывать вопросы с собеседований
- Добавлять темы для изучения
- Отмечать пробелы в знаниях
- Планировать дополнение документации

---

## 📌 Как использовать этот гайд

1. **Регулярное повторение**: проходи по темам раз в неделю
2. **Практика**: для каждой темы пиши код или рисуй схемы
3. **Глубина понимания**: не просто запоминай термины, понимай "почему так работает"
4. **Актуальность**: следи за обновлениями PHP и Laravel
5. **Backlog**: записывай новые темы в [backlog.md](backlog.md) и периодически переноси в основную документацию
6. **Ресурсы**: добавляй полезные курсы и видео в [resources.md](resources.md)

**Легенда уровней погружения:**
- 🔴 **Deep dive** - максимальная глубина, детальное понимание внутреннего устройства
- 🟡 **Conceptual** - концепции и практическое применение, как это работает и когда использовать
- 🟢 **Basics** - базовое понимание, общие принципы
