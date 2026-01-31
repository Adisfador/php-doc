# Grafana - визуализация и дашборды

## Основы Grafana

### Что такое Grafana
- Open-source платформа для визуализации метрик
- Поддержка множества data sources (Prometheus, MySQL, PostgreSQL, Elasticsearch, Loki)
- Интерактивные дашборды
- Alerting и notifications
- Templating и переменные

### Компоненты
- **Dashboard** - набор панелей
- **Panel** - отдельный график/таблица
- **Data Source** - источник данных
- **Query** - запрос к data source
- **Variable** - переменная для фильтрации
- **Alert** - правило алертинга

## Типы визуализаций

### Time Series (Graph)
- Линейные графики
- Area charts
- Bars
- Используется для: metrics over time, trends

### Stat / Single Stat
- Одно значение
- С sparkline
- Thresholds и цветовая индикация
- Используется для: current values, KPIs

### Gauge
- Циферблатная визуализация
- Процентное значение
- Min/max thresholds
- Используется для: CPU, memory, disk usage

### Bar Chart / Bar Gauge
- Горизонтальные или вертикальные столбцы
- Сравнение значений
- Используется для: top N items

### Table
- Табличное представление
- Сортировка, фильтрация
- Форматирование колонок
- Используется для: logs, detailed metrics

### Heatmap
- Тепловая карта
- Распределение значений
- Используется для: latency distribution, histograms

### Pie Chart / Donut
- Круговая диаграмма
- Доли от целого
- Используется для: distribution breakdown

## Интеграция с Prometheus

### Настройка Data Source
```yaml
# grafana provisioning
apiVersion: 1
datasources:
  - name: Prometheus
    type: prometheus
    access: proxy
    url: http://prometheus:9090
    isDefault: true
```

### Примеры PromQL запросов
```promql
# Request rate по статусам
sum by (status) (rate(http_requests_total[5m]))

# 95-й перцентиль latency
histogram_quantile(0.95, 
  rate(http_request_duration_seconds_bucket[5m])
)

# Error rate percentage
sum(rate(http_requests_total{status=~"5.."}[5m])) / 
sum(rate(http_requests_total[5m])) * 100

# Memory usage
php_memory_usage_bytes / 1024 / 1024
```

## Variables (Переменные)

### Query Variable
```promql
# Список инстансов
label_values(http_requests_total, instance)

# Список роутов
label_values(http_requests_total{instance="$instance"}, route)
```

### Custom Variable
```
prod, staging, dev
```

### Interval Variable
```
$__auto_interval_count
```

### Использование в запросах
```promql
rate(http_requests_total{instance="$instance", route="$route"}[5m])
```

## Dashboard Design

### Структура дашборда
```
┌─────────────────────────────────────────┐
│  KPIs (Single Stats)                    │
│  Request Rate | Error Rate | P95 Latency│
├─────────────────────────────────────────┤
│  Main Graphs                            │
│  Requests/sec | Response Time           │
├─────────────────────────────────────────┤
│  Details                                │
│  Requests by Route | Errors by Type    │
├─────────────────────────────────────────┤
│  Resources                              │
│  Memory | CPU | Connections             │
└─────────────────────────────────────────┘
```

### Best Practices дизайна
- Важные метрики наверху
- Логическая группировка панелей
- Консистентные цвета
- Понятные названия
- Annotations для деплоев
- Row collapse для деталей

## Alerting в Grafana

### Alert Rule
```yaml
# На панели
Conditions:
  WHEN avg() OF query(A, 5m, now) IS ABOVE 0.1
  
# Notification
Send to: slack-channel
Message: High error rate: {{ $value }}%
```

### Alert States
- **OK** - все в порядке
- **Pending** - условие выполнено, но не достаточно долго
- **Alerting** - alert активен
- **No Data** - нет данных
- **Error** - ошибка в запросе

### Notification Channels
- Email
- Slack
- Telegram
- PagerDuty
- Webhook
- Discord

## Provisioning (as Code)

### Dashboard JSON
```json
{
  "dashboard": {
    "title": "PHP Application",
    "panels": [
      {
        "title": "Request Rate",
        "targets": [{
          "expr": "rate(http_requests_total[5m])"
        }]
      }
    ]
  },
  "overwrite": true
}
```

### Provisioning файлы
```
/etc/grafana/provisioning/
  ├── datasources/
  │   └── prometheus.yml
  ├── dashboards/
  │   ├── dashboard.yml
  │   └── dashboards/
  │       └── app.json
  └── notifiers/
      └── slack.yml
```

## Annotations

### Deployment annotations
```promql
# Из Prometheus
ALERTS{alertname="Deployment"}

# Manual annotation
Запись события деплоя вручную
```

### Использование
- Корреляция событий с метриками
- Debugging performance issues
- Понимание причин аномалий

## Templating Examples

### PHP Application Dashboard
```
Variables:
- environment: prod, staging, dev
- instance: label_values(http_requests_total, instance)
- route: label_values(http_requests_total, route)

Panels:
1. Request Rate
   sum(rate(http_requests_total{env="$environment", instance=~"$instance"}[5m]))

2. Error Rate %
   sum(rate(http_requests_total{status=~"5..", env="$environment"}[5m])) / 
   sum(rate(http_requests_total{env="$environment"}[5m])) * 100

3. P95 Latency
   histogram_quantile(0.95, 
     rate(http_request_duration_seconds_bucket{env="$environment"}[5m])
   )

4. Memory Usage
   php_memory_usage_bytes{env="$environment", instance=~"$instance"}

5. Active Connections
   php_pdo_connections_active{env="$environment"}
```

## Plugins

### Популярные плагины
- **Pie Chart** - круговые диаграммы
- **Worldmap Panel** - географическая визуализация
- **Clock Panel** - отображение времени
- **Boom Table** - расширенные таблицы
- **Polystat** - множественные stat панели

### Установка
```bash
grafana-cli plugins install grafana-piechart-panel
grafana-cli plugins install grafana-worldmap-panel
```

## Best Practices

### Организация
- Папки для группировки дашбордов
- Теги для категоризации
- Naming convention: `[Team] Service - Purpose`
- Dashboard для каждого сервиса

### Performance
- Ограничение time range
- Использование recording rules
- Кеширование запросов
- Optimization query intervals

### Maintenance
- Version control для dashboards (JSON в git)
- Provisioning вместо manual config
- Regular review и cleanup
- Documentation в dashboard description
