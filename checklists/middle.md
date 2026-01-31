# Middle PHP Developer Checklist

## PHP Core

### Основы языка
- [ ] Типы данных: scalar, compound, special
- [ ] Type juggling и strict types
- [ ] References и copy-on-write
- [ ] Области видимости: global, local, static
- [ ] Anonymous functions и closures
- [ ] Generators и yield
- [ ] Error handling: exceptions, try-catch-finally
- [ ] include vs require, _once варианты

### ООП
- [ ] Классы и объекты
- [ ] Наследование и полиморфизм
- [ ] Абстрактные классы vs интерфейсы
- [ ] Traits: использование, конфликты
- [ ] Magic methods: __get, __set, __call, __toString
- [ ] Visibility: public, protected, private
- [ ] Static methods и properties
- [ ] Constructor property promotion (PHP 8+)
- [ ] Late static binding (static::)
- [ ] Anonymous classes

### Типизация
- [ ] Type hints: scalar, object, array
- [ ] Return types
- [ ] Nullable types (?type)
- [ ] Union types (PHP 8+)
- [ ] Mixed type
- [ ] Void и never types

### SPL
- [ ] Iterators: Iterator, IteratorAggregate
- [ ] Data structures: SplStack, SplQueue, SplHeap
- [ ] ArrayObject и ArrayIterator
- [ ] SPL Exceptions
- [ ] SPL File Handling

## Базы данных

### SQL
- [ ] SELECT: JOIN, GROUP BY, HAVING, ORDER BY
- [ ] Подзапросы: scalar, column, row, table
- [ ] CTE (Common Table Expressions)
- [ ] Window functions: ROW_NUMBER, RANK, LAG, LEAD
- [ ] Агрегатные функции: COUNT, SUM, AVG, GROUP_CONCAT
- [ ] UNION vs UNION ALL
- [ ] CASE expressions

### Индексы
- [ ] B-Tree индексы
- [ ] Unique и composite индексы
- [ ] Covering indexes
- [ ] Index selectivity
- [ ] EXPLAIN и query analysis
- [ ] Когда индексы не используются

### Транзакции
- [ ] ACID свойства
- [ ] BEGIN, COMMIT, ROLLBACK
- [ ] Уровни изоляции: READ UNCOMMITTED, READ COMMITTED, REPEATABLE READ, SERIALIZABLE
- [ ] Phantom reads, dirty reads, non-repeatable reads
- [ ] Deadlocks и их решение

### Производительность
- [ ] N+1 query problem
- [ ] Eager loading vs lazy loading
- [ ] Query optimization
- [ ] Connection pooling
- [ ] Database caching

## Laravel

### Основы
- [ ] Request lifecycle
- [ ] Service Container и dependency injection
- [ ] Service Providers
- [ ] Facades vs dependency injection
- [ ] Configuration и environment variables

### Routing & Controllers
- [ ] Route definition и parameters
- [ ] Route groups и middleware
- [ ] Resource controllers
- [ ] Form requests и validation
- [ ] API Resources

### Eloquent ORM
- [ ] Models и conventions
- [ ] Relationships: hasOne, hasMany, belongsTo, belongsToMany
- [ ] Eager loading: with(), load()
- [ ] Query scopes: local и global
- [ ] Accessors и mutators
- [ ] Casting attributes
- [ ] Soft deletes

### Database
- [ ] Query Builder
- [ ] Migrations и schema building
- [ ] Seeders и factories
- [ ] Raw queries и bindings
- [ ] Transactions в Laravel

### Jobs & Queues
- [ ] Job classes
- [ ] Queue drivers: database, redis, sqs
- [ ] Job chaining и batching
- [ ] Failed jobs handling
- [ ] Rate limiting

### Middleware
- [ ] Global middleware
- [ ] Route middleware
- [ ] Middleware groups
- [ ] Terminable middleware

### Caching
- [ ] Cache drivers
- [ ] Cache::remember pattern
- [ ] Cache tags (Redis)
- [ ] Query caching
- [ ] View caching

### Events & Listeners
- [ ] Event definition
- [ ] Listener registration
- [ ] Event discovery
- [ ] Observers
- [ ] Queue listeners

## API Development

### REST API
- [ ] HTTP methods: GET, POST, PUT, PATCH, DELETE
- [ ] Status codes: 2xx, 3xx, 4xx, 5xx
- [ ] RESTful resource naming
- [ ] Pagination: offset, cursor
- [ ] Filtering и sorting
- [ ] HATEOAS принципы
- [ ] Versioning: URI, header, content negotiation

### Authentication
- [ ] Session-based auth
- [ ] Token-based auth
- [ ] JWT: структура, claims, signature
- [ ] OAuth 2.0: flows, tokens
- [ ] API tokens (Sanctum)
- [ ] Rate limiting

### API Best Practices
- [ ] Request validation
- [ ] Error handling и consistent responses
- [ ] API documentation (OpenAPI/Swagger)
- [ ] CORS configuration
- [ ] Security headers

## Паттерны проектирования

### SOLID
- [ ] Single Responsibility Principle
- [ ] Open/Closed Principle
- [ ] Liskov Substitution Principle
- [ ] Interface Segregation Principle
- [ ] Dependency Inversion Principle

### Основные паттерны GoF
- [ ] Singleton
- [ ] Factory Method
- [ ] Abstract Factory
- [ ] Builder
- [ ] Strategy
- [ ] Observer
- [ ] Decorator
- [ ] Adapter
- [ ] Repository

### Архитектурные паттерны
- [ ] MVC (Model-View-Controller)
- [ ] Repository Pattern
- [ ] Service Layer
- [ ] DTO (Data Transfer Object)

## Тестирование

### PHPUnit
- [ ] Unit tests
- [ ] Assertions
- [ ] Data providers
- [ ] setUp и tearDown
- [ ] Mocking: createMock, createStub
- [ ] Code coverage

### Feature Testing
- [ ] HTTP tests
- [ ] Database assertions
- [ ] Authentication testing
- [ ] JSON response testing

### Best Practices
- [ ] AAA pattern (Arrange-Act-Assert)
- [ ] Test isolation
- [ ] Fast tests
- [ ] Meaningful test names

## Git

### Основы
- [ ] git add, commit, push, pull
- [ ] git status, diff, log
- [ ] Branching и checkout
- [ ] Merge vs rebase
- [ ] Stash
- [ ] Cherry-pick

### Workflow
- [ ] Git Flow
- [ ] Feature branches
- [ ] Pull requests
- [ ] Code review process
- [ ] Merge conflicts resolution

## DevOps Basics

### Docker
- [ ] Контейнеры vs виртуальные машины
- [ ] Dockerfile: FROM, RUN, COPY, CMD
- [ ] docker build, run, exec
- [ ] Docker Compose
- [ ] Volumes и networking
- [ ] Multi-stage builds

### Linux
- [ ] Навигация: cd, ls, pwd
- [ ] Файлы: cat, grep, find, chmod
- [ ] Процессы: ps, top, kill
- [ ] Permissions: rwx, chown, chmod
- [ ] Environment variables

### Web Servers
- [ ] Nginx: конфигурация, location blocks
- [ ] Apache: .htaccess, mod_rewrite
- [ ] Reverse proxy
- [ ] Load balancing basics
- [ ] SSL/TLS certificates

## Безопасность

### OWASP Top 10
- [ ] SQL Injection защита
- [ ] XSS (Cross-Site Scripting) предотвращение
- [ ] CSRF protection
- [ ] Authentication и session management
- [ ] Sensitive data exposure
- [ ] Security headers: CSP, X-Frame-Options

### PHP Security
- [ ] Input validation и sanitization
- [ ] Prepared statements
- [ ] Password hashing: bcrypt, argon2
- [ ] File upload security
- [ ] HTTPS enforcement

## Инструменты

### Composer
- [ ] require vs require-dev
- [ ] Autoloading: PSR-4
- [ ] Semantic versioning
- [ ] composer.lock назначение
- [ ] Scripts

### Debugging
- [ ] Xdebug: breakpoints, step debugging
- [ ] dd(), dump() в Laravel
- [ ] Laravel Telescope
- [ ] Log levels и channels

## Soft Skills

### Код
- [ ] Clean code principles
- [ ] Meaningful naming
- [ ] Code comments когда нужны
- [ ] DRY principle
- [ ] KISS principle

### Работа в команде
- [ ] Code review participation
- [ ] Documentation writing
- [ ] Task estimation
- [ ] Communication in tickets/PRs

## Performance

### PHP
- [ ] OpCache использование
- [ ] Memory management
- [ ] Profiling: Xdebug, Blackfire
- [ ] Avoiding N+1 queries

### Caching
- [ ] Redis basics
- [ ] Cache strategies: cache-aside, write-through
- [ ] Cache invalidation
- [ ] CDN для static assets

## Оценка готовности
- ✅ 80%+ - готов к middle позиции
- 🟡 60-79% - нужна подготовка
- ❌ <60% - сфокусируйся на основах
