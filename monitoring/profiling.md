# Профилирование и анализ производительности PHP

## Инструменты профилирования

### Xdebug Profiler
- Генерация cachegrind файлов
- Анализ с помощью KCachegrind/QCachegrind/WebGrind
- Включение: `xdebug.mode=profile`
- Conditional profiling с trigger
- Метрики: время выполнения, количество вызовов, memory usage

### Blackfire.io
- Production-ready профилировщик
- Автоматические рекомендации по оптимизации
- Сравнение профилей (до/после)
- CI/CD интеграция
- Метрики: CPU time, I/O time, memory, network

### Tideways / XHProf
- Легковесные профилировщики для production
- Минимальный overhead (~1-3%)
- Hierarchical profiling
- Интеграция с APM системами

### SPX (Simple Profiling eXtension)
- Веб-интерфейс в реальном времени
- Flame graphs
- Memory profiling
- Минимальная нагрузка

## Метрики производительности

### Время выполнения
- Wall time - реальное время
- CPU time - время процессора
- I/O time - ожидание ввода/вывода
- Inclusive vs exclusive time

### Память
- Memory usage - текущее потребление
- Peak memory - пиковое значение
- Memory leaks - утечки памяти
- Allocated vs used memory

### Вызовы функций
- Call count - количество вызовов
- Call graph - граф вызовов
- Hot paths - наиболее нагруженные пути
- Bottlenecks - узкие места

## Анализ и оптимизация

### Flame Graphs
- Визуализация стека вызовов
- Ширина = время выполнения
- Интерактивный drill-down
- Поиск горячих путей

### N+1 Query Problem
- Обнаружение через профилировщик БД
- Laravel Debugbar / Telescope
- Clockwork для детального анализа
- Решение: eager loading, query optimization

### OpCode Caching
- OpCache статистика: `opcache_get_status()`
- Hit rate, memory usage, interned strings
- Оптимизация: `opcache.validate_timestamps=0` в production
- Preloading в PHP 7.4+

### Memory Profiling
- `memory_get_usage()` и `memory_get_peak_usage()`
- Valgrind для глубокого анализа
- Поиск циклических ссылок
- Optimization: unset, gc_collect_cycles

## Best Practices

### В разработке
- Профилировать на realistic data sets
- Тестировать с production-like нагрузкой
- Сравнивать до и после оптимизаций
- Документировать результаты

### В production
- Использовать sampling (1-5% запросов)
- Автоматические алерты на деградацию
- Регулярный мониторинг ключевых endpoint'ов
- A/B тестирование оптимизаций

### Типичные проблемы
- N+1 queries
- Отсутствие кеширования
- Избыточные вычисления в циклах
- Неоптимальные алгоритмы (O(n²) вместо O(n))
- Memory leaks в long-running процессах
- Blocking I/O операции

## Интеграция в workflow

### CI/CD
- Автоматический performance regression testing
- Blackfire builds на каждый PR
- Thresholds для critical paths
- Performance budgets

### Мониторинг
- APM (Application Performance Monitoring)
- Real User Monitoring (RUM)
- Synthetic monitoring
- Distributed tracing

### Инструменты Laravel
- Laravel Telescope - queries, jobs, events
- Laravel Debugbar - timeline, queries, views
- Clockwork - детальная информация о request
- Ray - advanced debugging tool
