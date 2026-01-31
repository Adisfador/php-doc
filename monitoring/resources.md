# Обучающие ресурсы: Мониторинг и Observability

## 🎯 Основные материалы

### Видео
- [Loki, Prometheus, Grafana - полный разбор](https://www.youtube.com/watch?v=2JIyHNskK-c) - стек мониторинга и логирования, как настроить и использовать

---

## 📖 Дополнительно

### Документация

### Prometheus
- [Prometheus Official Docs](https://prometheus.io/docs/introduction/overview/) - официальная документация
- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/) - best practices
- [PromQL Cheat Sheet](https://promlabs.com/promql-cheat-sheet/) - шпаргалка по PromQL

### Grafana
- [Grafana Documentation](https://grafana.com/docs/) - официальная документация
- [Grafana Tutorials](https://grafana.com/tutorials/) - интерактивные туториалы
- [Grafana Dashboards](https://grafana.com/grafana/dashboards/) - готовые дашборды

### Loki
- [Grafana Loki Docs](https://grafana.com/docs/loki/latest/) - документация Loki
- [LogQL Cheat Sheet](https://grafana.com/docs/loki/latest/logql/log_queries/) - язык запросов Loki

### Профилирование

#### Xdebug
- [Xdebug Documentation](https://xdebug.org/docs/) - официальная документация
- [Xdebug Profiling](https://xdebug.org/docs/profiler) - профилирование с Xdebug

#### Blackfire
- [Blackfire Docs](https://blackfire.io/docs/introduction) - официальная документация
- [Blackfire Academy](https://blackfire.io/docs/cookbooks/index) - туториалы и примеры

## Инструменты для мониторинга PHP

### APM (Application Performance Monitoring)
- [New Relic APM](https://docs.newrelic.com/docs/apm/) - мониторинг приложений
- [Datadog APM](https://docs.datadoghq.com/tracing/) - distributed tracing
- [Elastic APM](https://www.elastic.co/guide/en/apm/guide/current/index.html) - APM от Elastic

### Error Tracking
- [Sentry](https://docs.sentry.io/) - error tracking и мониторинг
- [Bugsnag](https://docs.bugsnag.com/) - error monitoring
- [Rollbar](https://docs.rollbar.com/) - error tracking

### Laravel-специфичные
- [Laravel Telescope](https://laravel.com/docs/11.x/telescope) - локальная отладка
- [Laravel Horizon](https://laravel.com/docs/11.x/horizon) - мониторинг очередей
- [Spatie Laravel Log Viewer](https://github.com/spatie/laravel-log-viewer) - просмотр логов

## Практика

### Настройка стека мониторинга
1. Prometheus + Node Exporter для метрик сервера
2. Grafana для визуализации
3. Loki для сбора логов
4. Alertmanager для алертов

### Метрики для отслеживания
- Response time (P50, P95, P99)
- Error rate
- Throughput (requests/sec)
- Database query time
- Queue processing time
- Memory usage
- CPU usage

### Примеры дашбордов
- [Laravel Dashboard](https://grafana.com/grafana/dashboards/11529) - метрики Laravel
- [Nginx Dashboard](https://grafana.com/grafana/dashboards/12708) - метрики Nginx
- [PostgreSQL Dashboard](https://grafana.com/grafana/dashboards/9628) - метрики PostgreSQL

## Статьи и гайды

- [The Four Golden Signals](https://sre.google/sre-book/monitoring-distributed-systems/) - что мониторить
- [Observability Engineering](https://www.oreilly.com/library/view/observability-engineering/9781492076438/) - книга от Honeycomb
- [PHP Profiling Guide](https://tideways.com/profiler/blog/essential-guide-to-php-profiling-like-a-boss) - гайд по профилированию PHP

## YouTube каналы

- [Grafana Labs](https://www.youtube.com/@Grafana) - официальный канал
- [Prometheus Monitoring](https://www.youtube.com/c/PrometheusIo) - туториалы по Prometheus

## Инструменты для анализа профайлов

- [KCachegrind](https://kcachegrind.github.io/) - визуализация профайлов Xdebug
- [Webgrind](https://github.com/jokkedk/webgrind) - веб-интерфейс для профайлов
- [phpspy](https://github.com/adsr/phpspy) - low-overhead профайлер

## Best Practices

### Логирование
- Структурированные логи (JSON)
- Уровни логирования (DEBUG, INFO, WARNING, ERROR)
- Correlation ID для трейсинга
- Не логировать sensitive данные

### Метрики
- RED метод (Rate, Errors, Duration)
- USE метод (Utilization, Saturation, Errors)
- Naming conventions для метрик

### Алерты
- Не слишком много алертов (alert fatigue)
- Actionable алерты (можешь что-то сделать)
- Документация для каждого алерта (runbook)
