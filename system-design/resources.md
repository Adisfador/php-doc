# System Design - Обучающие ресурсы

Лучшие ресурсы для изучения проектирования систем и подготовки к System Design интервью.
https://www.youtube.com/watch?v=iyxd_wmWnVA&t  - Производительность в System Design: tail latency, throughput и масштабирование
https://www.youtube.com/watch?v=wOIJjCa-UZE - System Design: от расчёта нагрузки к масштабированию БД
---

## 📚 Книги

### Must-Read
- **[System Design Interview Vol 1 & 2](https://www.amazon.com/System-Design-Interview-insiders-Second/dp/B08CMF2CQF)** - Alex Xu
  - Лучшая книга для подготовки к интервью
  - 15+ реальных систем (Twitter, YouTube, Google Drive)
  - Диаграммы и пошаговые решения
  
- **[Designing Data-Intensive Applications](https://dataintensive.net/)** - Martin Kleppmann
  - Глубокое понимание distributed systems
  - Replication, partitioning, transactions
  - Must-read для Senior

### Дополнительные
- **[Web Scalability for Startup Engineers](https://www.amazon.com/Scalability-Startup-Engineers-Artur-Ejsmont/dp/0071843655)** - Artur Ejsmont
- **[Building Microservices](https://www.oreilly.com/library/view/building-microservices-2nd/9781492034018/)** - Sam Newman
- **[Release It!](https://pragprog.com/titles/mnee2/release-it-second-edition/)** - Michael Nygard (Production readiness)

---

## 🎓 Онлайн курсы

### Платные (рекомендую)
- **[Grokking the System Design Interview](https://www.educative.io/courses/grokking-the-system-design-interview)** - Educative
  - 13 систем с детальным разбором
  - Интерактивные диаграммы
  - $79 (или Educative Unlimited)
  
- **[Grokking the Advanced System Design](https://www.educative.io/courses/grokking-adv-system-design-intvw)** - Educative
  - Distributed transactions, consensus
  - Для Senior уровня
  
- **[AlgoExpert System Design](https://www.algoexpert.io/systems/fundamentals)** - AlgoExpert
  - Video explanations
  - 30+ fundamentals topics

### Бесплатные
- **[MIT 6.824: Distributed Systems](https://pdos.csail.mit.edu/6.824/)** - MIT
  - Университетский курс (readings + labs)
  - Raft, MapReduce, distributed transactions
  
- **[System Design Primer](https://github.com/donnemartin/system-design-primer)** - GitHub
  - Огромный репозиторий с примерами
  - 200K+ stars

---

## 📺 YouTube каналы

### System Design
- **[ByteByteGo](https://www.youtube.com/@ByteByteGo)** - Alex Xu (автор книги)
  - 5-10 минут на систему
  - Анимированные диаграммы
  - 700K+ subscribers

- **[Gaurav Sen](https://www.youtube.com/c/GauravSensei)**
  - Design Uber, WhatsApp, Netflix
  - Очень подробные объяснения
  
- **[System Design Interview](https://www.youtube.com/@SystemDesignInterview)**
  - Mock interviews
  - Real-time problem solving

### Смежные темы
- **[Hussein Nasser](https://www.youtube.com/@hnasr)** - Backend engineering, databases
- **[Arpit Bhayani](https://www.youtube.com/@ArpitBhayani)** - Deep dives

---

## 💻 Практические платформы

### Mock Interviews
- **[Pramp](https://www.pramp.com/)** - Бесплатные peer-to-peer mock interviews
- **[Interviewing.io](https://interviewing.io/)** - Mock interviews с инженерами из FAANG
- **[Exponent](https://www.tryexponent.com/)** - System Design + Behavioral

### Диаграммы
- **[Excalidraw](https://excalidraw.com/)** - Sketching diagrams
- **[Draw.io](https://app.diagrams.net/)** - Professional diagrams
- **[Lucidchart](https://www.lucidchart.com/)** - Architecture diagrams

---

## 📖 Статьи и блоги

### Engineering Blogs (Must Read)
- **[High Scalability](http://highscalability.com/)** - Case studies реальных систем
- **[Netflix Tech Blog](https://netflixtechblog.com/)**
- **[Uber Engineering](https://eng.uber.com/)**
- **[Instagram Engineering](https://instagram-engineering.com/)**
- **[Airbnb Engineering](https://medium.com/airbnb-engineering)**
- **[Meta Engineering](https://engineering.fb.com/)**

### Specific Topics
- **[CAP Theorem Explained](https://www.ibm.com/topics/cap-theorem)** - IBM
- **[Raft Consensus](https://raft.github.io/)** - Raft protocol
- **[AWS Architecture Blog](https://aws.amazon.com/blogs/architecture/)**

---

## 🛠️ Инструменты и технологии

### Load Balancers
- **[Nginx Documentation](https://nginx.org/en/docs/)**
- **[HAProxy Documentation](http://www.haproxy.org/#docs)**

### Caching
- **[Redis Documentation](https://redis.io/docs/)**
- **[Memcached Wiki](https://github.com/memcached/memcached/wiki)**

### Message Queues
- **[RabbitMQ Tutorials](https://www.rabbitmq.com/getstarted.html)**
- **[Apache Kafka Documentation](https://kafka.apache.org/documentation/)**

### Databases
- **[PostgreSQL Documentation](https://www.postgresql.org/docs/)**
- **[MongoDB University](https://learn.mongodb.com/)** - Free courses

---

## 🎯 Практика: реальные примеры систем

### Tier 1 (начальный)
- URL Shortener (bit.ly)
- Pastebin
- Key-Value Store

### Tier 2 (средний)
- Twitter (timeline, following)
- Instagram (photo upload, feed)
- Uber (matching, pricing)
- WhatsApp (messaging, delivery)

### Tier 3 (продвинутый)
- YouTube (video streaming, recommendations)
- Netflix (CDN, encoding)
- Google Drive (file sync, sharing)
- Distributed Cache (like Memcached)

---

## 📋 Чек-листы для подготовки

### 1-2 месяца до интервью
- [ ] Прочитать System Design Interview Vol 1 (Alex Xu)
- [ ] Пройти Grokking the System Design Interview
- [ ] Решить 15+ систем самостоятельно
- [ ] Посмотреть 20+ видео ByteByteGo

### 2-4 недели до интервью
- [ ] Mock interviews на Pramp (5+ sessions)
- [ ] Повторить все fundamentals (CAP, consistency, etc.)
- [ ] Прочитать engineering blogs (Netflix, Uber, Instagram)
- [ ] Практиковать drawing диаграмм

### 1 неделя до интервью
- [ ] Повторить capacity estimations (back of envelope)
- [ ] Просмотреть свои решения 15+ систем
- [ ] Mock interview с другом/коллегой
- [ ] Отдохнуть 😊

---

## 🎓 Roadmap изучения

### Новичок (0-3 месяца)
1. Основы: CAP, consistency, availability
2. Базовые компоненты: load balancer, cache, database
3. Простые системы: URL shortener, Pastebin

### Intermediate (3-6 месяцев)
1. Scaling: sharding, replication, CDN
2. Messaging: queues, pub/sub
3. Средние системы: Twitter, Instagram

### Advanced (6-12 месяцев)
1. Distributed systems: consensus, transactions
2. Microservices patterns
3. Сложные системы: YouTube, Google Drive

---

## 💡 Советы для успеха

### Do's
✅ Начинай с простого, постепенно усложняй  
✅ Рисуй диаграммы (архитектурные блоки)  
✅ Обсуждай trade-offs (consistency vs availability)  
✅ Используй реальные цифры (capacity estimations)  
✅ Думай вслух (объясняй свои решения)  

### Don'ts
❌ Не прыгай сразу в детали реализации  
❌ Не молчи (интервьюер хочет видеть процесс мышления)  
❌ Не игнорируй requirements (functional + non-functional)  
❌ Не пытайся запомнить решения наизусть (понимай концепции)  

---

## 📊 Прогресс трекинг

```
[ ] Fundamentals (CAP, ACID, BASE, consistency models)
[ ] Scalability (load balancing, caching, DB scaling)
[ ] Common patterns (rate limiting, pagination, feed generation)
[ ] Tier 1 systems (3/3 решены)
[ ] Tier 2 systems (5/5 решены)
[ ] Tier 3 systems (3/3 решены)
[ ] 10+ mock interviews
```

---

## 🔗 Полезные ссылки

- [System Design Primer GitHub](https://github.com/donnemartin/system-design-primer)
- [Awesome Scalability](https://github.com/binhnguyennus/awesome-scalability)
- [System Design Interview Checklist](https://github.com/checkcheckzz/system-design-interview)
