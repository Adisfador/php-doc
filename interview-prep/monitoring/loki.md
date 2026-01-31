# Loki - логирование и анализ логов

## Основы Loki

### Что такое Loki
- Log aggregation система от Grafana Labs
- "Prometheus для логов"
- Индексирует только метаданные (labels), не содержимое
- Хранит логи в сжатом виде
- Интеграция с Grafana

### Архитектура
- **Promtail** - агент для сбора логов
- **Loki** - сервер хранения и обработки
- **Grafana** - UI для поиска и визуализации
- **LogQL** - язык запросов

### Отличия от Elasticsearch
- Индексирует только labels, не full-text
- Меньше требований к ресурсам
- Проще в настройке и поддержке
- Нет сложных поисковых запросов
- Дешевле в эксплуатации

## LogQL - язык запросов

### Log Stream Selector
```logql
{job="php-app"}                           # все логи приложения
{job="php-app", env="production"}         # с фильтрами
{job="php-app", env=~"prod|staging"}      # regex
{job="php-app", level!="debug"}           # исключение
```

### Line Filter
```logql
{job="php-app"} |= "error"                # содержит
{job="php-app"} != "debug"                # не содержит
{job="php-app"} |~ "error|fatal"          # regex contains
{job="php-app"} !~ "debug|trace"          # regex не содержит
```

### Parser
```logql
# JSON parser
{job="php-app"} | json

# Logfmt parser
{job="php-app"} | logfmt

# Pattern parser
{job="php-app"} | pattern `<level> <msg>`

# Regexp parser
{job="php-app"} | regexp `(?P<level>\w+): (?P<msg>.*)`
```

### Label Filter (после parsing)
```logql
{job="php-app"} 
  | json 
  | level="error"
  | duration > 1000

{job="php-app"} 
  | json 
  | user_id="123" 
  | status_code >= 500
```

### Aggregation Functions
```logql
# Count
count_over_time({job="php-app"} |= "error" [5m])

# Rate
rate({job="php-app"} [5m])

# Sum extracted values
sum by (route) (
  rate({job="php-app"} | json | unwrap duration [5m])
)

# Quantiles
quantile_over_time(0.95, 
  {job="php-app"} | json | unwrap duration [5m]
)
```

## Promtail конфигурация

### Сбор из файлов
```yaml
server:
  http_listen_port: 9080

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: php-app
    static_configs:
      - targets:
          - localhost
        labels:
          job: php-app
          env: production
          __path__: /var/log/php/*.log
    
    pipeline_stages:
      # Parse JSON logs
      - json:
          expressions:
            level: level
            message: message
            context: context
            timestamp: timestamp
      
      # Extract labels
      - labels:
          level:
          
      # Timestamp parsing
      - timestamp:
          source: timestamp
          format: RFC3339
```

### Docker logs
```yaml
scrape_configs:
  - job_name: containers
    docker_sd_configs:
      - host: unix:///var/run/docker.sock
    relabel_configs:
      - source_labels: ['__meta_docker_container_name']
        target_label: 'container'
      - source_labels: ['__meta_docker_container_log_stream']
        target_label: 'stream'
```

### Kubernetes
```yaml
scrape_configs:
  - job_name: kubernetes-pods
    kubernetes_sd_configs:
      - role: pod
    relabel_configs:
      - source_labels: [__meta_kubernetes_pod_label_app]
        target_label: app
      - source_labels: [__meta_kubernetes_namespace]
        target_label: namespace
```

## Laravel интеграция

### Structured logging
```php
// config/logging.php
'channels' => [
    'loki' => [
        'driver' => 'monolog',
        'handler' => StreamHandler::class,
        'formatter' => JsonFormatter::class,
        'with' => [
            'stream' => '/var/log/php/laravel.log',
        ],
    ],
],

// Использование
Log::channel('loki')->error('Payment failed', [
    'user_id' => $user->id,
    'amount' => $amount,
    'error_code' => $errorCode,
]);
```

### Contextual logging
```php
Log::withContext([
    'request_id' => Str::uuid(),
    'user_id' => auth()->id(),
]);

Log::info('Request processed', [
    'duration' => $duration,
    'route' => request()->route()->getName(),
]);
```

### Custom formatter
```php
class LokiFormatter extends JsonFormatter
{
    public function format(array $record): string
    {
        return json_encode([
            'timestamp' => $record['datetime']->format('c'),
            'level' => $record['level_name'],
            'message' => $record['message'],
            'context' => $record['context'],
            'extra' => $record['extra'],
        ]) . "\n";
    }
}
```

## Queries для мониторинга

### Error tracking
```logql
# Errors per minute
sum(rate({job="php-app", level="error"} [1m]))

# Errors by type
sum by (exception) (
  count_over_time({job="php-app", level="error"} | json [5m])
)

# Top error messages
topk(10,
  sum by (message) (
    count_over_time({job="php-app", level="error"} [1h])
  )
)
```

### Performance monitoring
```logql
# Slow requests (>1s)
{job="php-app"} 
  | json 
  | duration > 1000
  
# P95 request duration
quantile_over_time(0.95,
  {job="php-app"} | json | unwrap duration [5m]
) by (route)

# Request rate by route
sum by (route) (
  rate({job="php-app"} | json [5m])
)
```

### Business metrics
```logql
# Payment failures
count_over_time(
  {job="php-app"} |= "Payment failed" [1h]
)

# Registrations
count_over_time(
  {job="php-app"} |= "User registered" [1h]
)

# Conversion funnel
sum by (step) (
  count_over_time({job="php-app"} | json | step="checkout" [1h])
)
```

## Grafana Dashboard

### Explore mode
- Ad-hoc поиск логов
- Live tail
- Переключение между logs и metrics
- Context switching (logs → traces)

### Dashboard panels
```
# Log Panel
{job="php-app", level=~"error|critical"}
  | json
  
# Stat Panel - Error Count
sum(count_over_time({job="php-app", level="error"} [5m]))

# Graph Panel - Error Rate
rate({job="php-app", level="error"} [5m])

# Table Panel - Top Errors
topk(10,
  sum by (message) (count_over_time({job="php-app", level="error"} [1h]))
)
```

### Variables
```
environment: label_values(job, env)
level: label_values(job, level)
```

## Alerting

### Alert rules
```yaml
groups:
  - name: php_app_logs
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate({job="php-app", level="error"} [5m])) > 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate in PHP app"
          
      - alert: CriticalError
        expr: |
          count_over_time({job="php-app", level="critical"} [1m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Critical error detected"
```

## Best Practices

### Labeling
- Используй мало labels (cardinality!)
- Хорошие labels: job, env, level, instance
- Плохие labels: user_id, request_id, trace_id (в context)
- Labels для фильтрации, context для деталей

### Log format
- Используй JSON для structured logs
- Включай: timestamp, level, message, context
- Добавляй request_id для трейсинга
- Consistent naming в context

### Performance
- Ограничивай time range
- Используй label filters перед line filters
- Избегай regex где можно использовать =
- Cache часто используемые запросы

### Retention
```yaml
# loki config
limits_config:
  retention_period: 30d

table_manager:
  retention_deletes_enabled: true
  retention_period: 30d
```

### Cost optimization
- Compress logs
- Minimal indexing
- Retention policy
- Log sampling для high-volume
