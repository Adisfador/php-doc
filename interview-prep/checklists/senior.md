# Senior PHP Developer Checklist

## Advanced PHP

### Internals
- [ ] Zend Engine архитектура
- [ ] OpCode и компиляция
- [ ] OpCache: конфигурация, preloading
- [ ] JIT compilation (PHP 8+)
- [ ] Memory management: reference counting, garbage collection
- [ ] Copy-on-write механизм
- [ ] Cycle collector для circular references

### Advanced OOP
- [ ] Late static binding детально
- [ ] Reflection API
- [ ] Attributes (PHP 8+)
- [ ] Covariance и contravariance
- [ ] Object cloning: __clone, deep vs shallow
- [ ] Serialization: Serializable interface, __serialize, __unserialize
- [ ] Anonymous classes use cases

### Type System
- [ ] Union types (string|int)
- [ ] Intersection types (Countable&Traversable)
- [ ] Mixed vs object
- [ ] Never type
- [ ] Readonly properties (PHP 8.1+)
- [ ] Enums (PHP 8.1+)
- [ ] Generic types через PHPDoc

### Performance
- [ ] Profiling: Xdebug, Blackfire, Tideways
- [ ] Memory profiling
- [ ] CPU profiling
- [ ] Flame graphs анализ
- [ ] Optimization strategies
- [ ] Micro-optimizations когда имеют смысл

### PSR Standards
- [ ] PSR-1, PSR-12: Coding standards
- [ ] PSR-3: Logger Interface
- [ ] PSR-4: Autoloading
- [ ] PSR-6, PSR-16: Caching
- [ ] PSR-7, PSR-15: HTTP Message, Middleware
- [ ] PSR-11: Container Interface
- [ ] PSR-14: Event Dispatcher
- [ ] PSR-18: HTTP Client

## Database Advanced

### Query Optimization
- [ ] EXPLAIN ANALYZE детальный анализ
- [ ] Query execution plans
- [ ] Index strategies: covering, partial, expression
- [ ] Partition tables
- [ ] Materialized views
- [ ] Query rewriting techniques
- [ ] Database-specific optimizations (PostgreSQL vs MySQL)

### Advanced Indexing
- [ ] B-Tree vs Hash indexes
- [ ] GiST, GIN indexes (PostgreSQL)
- [ ] Full-text search indexes
- [ ] Partial indexes
- [ ] Expression indexes
- [ ] Multi-column index ordering
- [ ] Index maintenance: VACUUM, ANALYZE, REINDEX

### Transactions Deep Dive
- [ ] MVCC (Multi-Version Concurrency Control)
- [ ] Lock types: row, table, advisory
- [ ] Lock modes: shared, exclusive
- [ ] Deadlock detection и prevention
- [ ] Savepoints
- [ ] Two-Phase Commit (2PC)
- [ ] Isolation levels implementation details

### Replication & Scaling
- [ ] Master-slave replication
- [ ] Master-master replication
- [ ] Read replicas
- [ ] Replication lag handling
- [ ] Sharding strategies: hash, range, geographic
- [ ] Partitioning: horizontal, vertical
- [ ] Connection pooling: PgBouncer, ProxySQL

### NoSQL
- [ ] Redis: data structures, persistence, clustering
- [ ] MongoDB: document model, aggregation pipeline
- [ ] Elasticsearch: full-text search, aggregations
- [ ] When to use NoSQL vs SQL
- [ ] Polyglot persistence

## Laravel Advanced

### Architecture
- [ ] Service Container deep dive: binding, resolution, contextual binding
- [ ] Macros и extending framework
- [ ] Package development
- [ ] Custom service providers
- [ ] Deferred providers
- [ ] Octane: Swoole, RoadRunner

### Eloquent Advanced
- [ ] Polymorphic relationships
- [ ] Many-to-many polymorphic
- [ ] Has-many-through
- [ ] Eloquent performance optimization
- [ ] Custom Eloquent builders
- [ ] Global scopes implementation
- [ ] Eloquent collections customization

### Advanced Features
- [ ] Broadcasting: Pusher, Socket.io, Redis
- [ ] Horizon для queue management
- [ ] Telescope для debugging
- [ ] Sanctum vs Passport
- [ ] Rate limiting strategies
- [ ] Task scheduling: cron, overlapping prevention
- [ ] Notifications: channels, customization

### Testing Advanced
- [ ] Mocking facades
- [ ] HTTP client faking
- [ ] Time manipulation в тестах
- [ ] Database transactions vs RefreshDatabase
- [ ] Browser testing: Dusk
- [ ] Parallel testing
- [ ] CI/CD integration

## Architectural Patterns

### Domain-Driven Design
- [ ] Bounded contexts
- [ ] Aggregates и aggregate roots
- [ ] Entities vs Value Objects
- [ ] Domain events
- [ ] Repositories pattern
- [ ] Domain services
- [ ] Application services
- [ ] Anti-corruption layer

### Hexagonal Architecture
- [ ] Ports и adapters
- [ ] Dependency rule
- [ ] Application core изоляция
- [ ] Infrastructure layer
- [ ] Implementing use cases

### CQRS & Event Sourcing
- [ ] Command Query Responsibility Segregation
- [ ] Event store
- [ ] Projections
- [ ] Eventual consistency
- [ ] Saga pattern
- [ ] Compensating transactions

### Microservices
- [ ] Service decomposition strategies
- [ ] API Gateway pattern
- [ ] Service discovery
- [ ] Circuit breaker
- [ ] Distributed transactions
- [ ] Distributed tracing
- [ ] Service mesh basics

## Message Queues & Async

### RabbitMQ
- [ ] Exchanges: direct, fanout, topic, headers
- [ ] Queues: durable, exclusive, auto-delete
- [ ] Bindings и routing keys
- [ ] Message acknowledgment
- [ ] Dead Letter Exchanges
- [ ] Priority queues
- [ ] Publisher confirms

### Kafka
- [ ] Topics и partitions
- [ ] Consumer groups
- [ ] Offset management
- [ ] Replication factor
- [ ] Retention policies
- [ ] Exactly-once semantics
- [ ] Kafka Streams basics

### Async Patterns
- [ ] Idempotency
- [ ] Retry strategies: exponential backoff
- [ ] Circuit breaker pattern
- [ ] Bulkhead pattern
- [ ] Saga pattern
- [ ] Outbox pattern
- [ ] Event-driven architecture

## API Design

### GraphQL Advanced
- [ ] Schema design best practices
- [ ] N+1 problem solutions: DataLoader
- [ ] Pagination: cursor-based
- [ ] Subscriptions
- [ ] Federation
- [ ] Error handling
- [ ] Security: depth limiting, query complexity

### gRPC
- [ ] Protocol Buffers
- [ ] Streaming: server, client, bidirectional
- [ ] Interceptors
- [ ] Deadlines и timeouts
- [ ] Error handling
- [ ] Load balancing

### API Security
- [ ] OAuth 2.0 flows: authorization code, client credentials, PKCE
- [ ] JWT: best practices, refresh tokens
- [ ] API keys management
- [ ] Rate limiting algorithms: token bucket, leaky bucket
- [ ] API versioning strategies
- [ ] API documentation: OpenAPI, AsyncAPI

## DevOps & Infrastructure

### Docker Advanced
- [ ] Multi-stage builds optimization
- [ ] Layer caching strategies
- [ ] Docker networking: bridge, host, overlay
- [ ] Docker volumes: named, bind mounts
- [ ] Docker Compose: depends_on, healthchecks
- [ ] Docker security: non-root user, scanning
- [ ] Registry management

### Kubernetes
- [ ] Pods, ReplicaSets, Deployments
- [ ] Services: ClusterIP, NodePort, LoadBalancer
- [ ] ConfigMaps и Secrets
- [ ] Persistent Volumes
- [ ] Ingress controllers
- [ ] Horizontal Pod Autoscaling
- [ ] Helm charts
- [ ] kubectl essential commands

### CI/CD
- [ ] Pipeline stages: build, test, deploy
- [ ] Artifact management
- [ ] Blue-green deployment
- [ ] Canary releases
- [ ] Rolling updates
- [ ] Rollback strategies
- [ ] Infrastructure as Code: Terraform basics

### Monitoring & Observability

#### Metrics (Prometheus/Grafana)
- [ ] Metric types: counter, gauge, histogram, summary
- [ ] PromQL queries
- [ ] Alerting rules
- [ ] Grafana dashboards
- [ ] Service Level Indicators (SLI)
- [ ] Service Level Objectives (SLO)

#### Logging (ELK/Loki)
- [ ] Structured logging
- [ ] Log aggregation
- [ ] LogQL / Lucene queries
- [ ] Log retention policies
- [ ] Correlation IDs

#### Tracing
- [ ] Distributed tracing concepts
- [ ] OpenTelemetry
- [ ] Jaeger / Zipkin
- [ ] Trace sampling strategies

## Security Advanced

### Application Security
- [ ] OWASP Top 10 deep dive
- [ ] Security headers: CSP, HSTS, CORP, COOP
- [ ] JWT security: алгоритмы, key management
- [ ] Secrets management: HashiCorp Vault
- [ ] Dependency scanning
- [ ] SAST и DAST tools
- [ ] Penetration testing basics

### Infrastructure Security
- [ ] Network segmentation
- [ ] Firewall rules
- [ ] VPC и security groups
- [ ] TLS/SSL: certificates, handshake, cipher suites
- [ ] SSH hardening
- [ ] Container security scanning
- [ ] Secrets in CI/CD

## System Design

### Scalability
- [ ] Horizontal vs vertical scaling
- [ ] Load balancing: round-robin, least connections, IP hash
- [ ] Stateless applications design
- [ ] Session management: sticky sessions, distributed sessions
- [ ] CDN для static content
- [ ] Database scaling strategies

### High Availability
- [ ] Redundancy
- [ ] Failover strategies
- [ ] Health checks
- [ ] Circuit breakers
- [ ] Graceful degradation
- [ ] Chaos engineering principles

### Performance
- [ ] Caching layers: browser, CDN, application, database
- [ ] Cache invalidation strategies
- [ ] Database connection pooling
- [ ] Async processing
- [ ] Read-through, write-through caching
- [ ] Content compression

### Reliability
- [ ] Error budgets
- [ ] Incident response
- [ ] Post-mortems
- [ ] Monitoring и alerting
- [ ] SLA, SLO, SLI
- [ ] Disaster recovery planning

## Design Patterns Advanced

### Behavioral
- [ ] Chain of Responsibility
- [ ] Command
- [ ] Iterator
- [ ] Mediator
- [ ] Memento
- [ ] State
- [ ] Template Method
- [ ] Visitor

### Structural
- [ ] Composite
- [ ] Proxy
- [ ] Flyweight
- [ ] Bridge
- [ ] Facade

### Creational
- [ ] Prototype
- [ ] Object Pool
- [ ] Dependency Injection

### Enterprise Patterns
- [ ] Unit of Work
- [ ] Identity Map
- [ ] Data Mapper
- [ ] Active Record
- [ ] Table Data Gateway
- [ ] Row Data Gateway

## Soft Skills

### Leadership
- [ ] Code review: конструктивная критика
- [ ] Mentoring junior developers
- [ ] Technical decision making
- [ ] Architecture documentation
- [ ] Technical debt management

### Communication
- [ ] Writing technical proposals (RFC)
- [ ] Presenting to stakeholders
- [ ] Cross-team collaboration
- [ ] Conflict resolution

### Process
- [ ] Agile methodologies
- [ ] Estimation: story points, planning poker
- [ ] Sprint planning participation
- [ ] Retrospectives facilitation

## Interview Preparation

### Coding Challenges
- [ ] Algorithm complexity: O(n), O(log n), O(n²)
- [ ] Data structures: arrays, linked lists, trees, graphs, hash tables
- [ ] Sorting: quick, merge, heap
- [ ] Searching: binary search, DFS, BFS
- [ ] Dynamic programming basics
- [ ] Common interview patterns

### System Design
- [ ] Requirements gathering
- [ ] Capacity estimation
- [ ] API design
- [ ] Database schema design
- [ ] Trade-offs discussion
- [ ] Scalability planning

## Оценка готовности
- ✅ 75%+ - сильный senior уровень
- 🟡 60-74% - senior с пробелами
- 🟠 50-59% - strong middle, требуется рост
- ❌ <50% - нужна серьёзная подготовка
