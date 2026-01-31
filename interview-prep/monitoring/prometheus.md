# Prometheus - метрики и мониторинг

## Основы Prometheus

### Что такое Prometheus
- Open-source система мониторинга и алертинга
- Time-series database
- Pull-based модель сбора метрик
- PromQL для запросов
- Service discovery для динамических сред

### Архитектура
- Prometheus Server - сбор и хранение метрик
- Exporters - экспорт метрик из приложений
- Pushgateway - для short-lived jobs
- Alertmanager - управление алертами
- Grafana - визуализация

## Типы метрик

### Counter (счётчик)
- Монотонно растущее значение
- Например: количество запросов, ошибок
- Операции: `inc()`, reset при рестарте
- Функции: `rate()`, `increase()`

### Gauge (датчик)
- Может расти и падать
- Например: температура, память, количество активных пользователей
- Операции: `set()`, `inc()`, `dec()`
- Используется как есть или с `avg_over_time()`

### Histogram
- Распределение значений по buckets
- Например: latency, response time
- Автоматически создаёт: `_bucket`, `_sum`, `_count`
- Расчёт квантилей на сервере

### Summary
- Похож на histogram, но квантили на клиенте
- Менее гибкий, но точнее для малых наборов
- Не агрегируется между инстансами
- Используется реже чем histogram

## PromQL основы

### Селекторы
```promql
http_requests_total                    # все метрики
http_requests_total{status="200"}      # с label
http_requests_total{status=~"2.."}     # regex
```

### Операторы
```promql
rate(http_requests_total[5m])          # скорость за 5 минут
increase(http_requests_total[1h])      # прирост за час
sum by (status) (http_requests_total)  # группировка
http_requests_total offset 1h          # смещение времени
```

### Функции агрегации
- `sum`, `avg`, `min`, `max`, `count`
- `topk`, `bottomk` - топ N значений
- `quantile` - квантили
- `stddev`, `stdvar` - стандартное отклонение

### Временные функции
- `rate()` - скорость изменения counter
- `irate()` - мгновенная скорость
- `delta()` - разница для gauge
- `increase()` - абсолютный прирост
- `avg_over_time()`, `max_over_time()`

## Интеграция с PHP

### PHP Prometheus Client
```php
// Composer: promphp/prometheus_client_php
use Prometheus\CollectorRegistry;
use Prometheus\Storage\Redis;

$registry = new CollectorRegistry(new Redis());

// Counter
$counter = $registry->getOrRegisterCounter(
    'app', 'http_requests_total', 'Total HTTP requests',
    ['method', 'status']
);
$counter->inc(['GET', '200']);

// Gauge
$gauge = $registry->getOrRegisterGauge(
    'app', 'memory_usage_bytes', 'Memory usage'
);
$gauge->set(memory_get_usage());

// Histogram
$histogram = $registry->getOrRegisterHistogram(
    'app', 'request_duration_seconds', 'Request duration',
    ['route'],
    [0.1, 0.5, 1, 2, 5] // buckets
);
$histogram->observe(0.3, ['api.users']);
```

### Endpoint для метрик
```php
// metrics.php
$renderer = new RenderTextFormat();
echo $renderer->render($registry->getMetricFamilySamples());
```

### Laravel middleware для метрик
```php
class PrometheusMiddleware
{
    public function handle($request, Closure $next)
    {
        $start = microtime(true);
        $response = $next($request);
        $duration = microtime(true) - $start;
        
        app(CollectorRegistry::class)
            ->getOrRegisterHistogram(...)
            ->observe($duration, [
                $request->route()->getName(),
                $response->getStatusCode()
            ]);
            
        return $response;
    }
}
```

## Service Discovery

### Static config
```yaml
scrape_configs:
  - job_name: 'php-app'
    static_configs:
      - targets: ['app1:9090', 'app2:9090']
```

### File-based SD
```yaml
scrape_configs:
  - job_name: 'php-app'
    file_sd_configs:
      - files: ['/etc/prometheus/targets/*.json']
```

### Kubernetes SD
```yaml
scrape_configs:
  - job_name: 'kubernetes-pods'
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
        action: keep
        regex: true
```

## Alerting

### Правила алертов
```yaml
groups:
  - name: php_app
    rules:
      - alert: HighErrorRate
        expr: rate(http_requests_total{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate on {{ $labels.instance }}"
          
      - alert: SlowRequests
        expr: histogram_quantile(0.95, http_request_duration_seconds_bucket) > 2
        for: 5m
        labels:
          severity: warning
```

### Recording rules
```yaml
groups:
  - name: php_app_rules
    rules:
      - record: job:http_requests:rate5m
        expr: rate(http_requests_total[5m])
```

## Best Practices

### Именование метрик
- `{namespace}_{subsystem}_{name}_{unit}`
- Примеры: `http_requests_total`, `php_memory_usage_bytes`
- Counter всегда с суффиксом `_total`
- Единицы измерения в имени: `_seconds`, `_bytes`

### Labels
- Использовать умеренно (cardinality!)
- Не использовать user_id, email как labels
- Хорошие labels: method, status, route, environment
- Избегать high-cardinality labels (timestamp, uuid)

### Производительность
- Redis storage для high-traffic приложений
- APCu storage для low-traffic
- Batch operations где возможно
- Кеширование registry

### Retention и storage
- По умолчанию 15 дней
- Конфигурация: `--storage.tsdb.retention.time=30d`
- Remote write для долгосрочного хранения
- Thanos/Cortex для масштабирования
