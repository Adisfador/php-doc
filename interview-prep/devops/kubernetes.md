# Kubernetes (K8s)

Платформа для автоматизации развёртывания, масштабирования и управления контейнеризированными приложениями.

---

## ☸️ Что такое Kubernetes

**Kubernetes (K8s)** - система оркестрации контейнеров с открытым исходным кодом, изначально разработанная Google.

### Зачем нужен Kubernetes

**Проблемы при ручном управлении контейнерами:**
- Как запустить 100 контейнеров на 10 серверах?
- Что делать если контейнер упал?
- Как масштабировать при увеличении нагрузки?
- Как обновлять без downtime?
- Как распределять нагрузку?

**Kubernetes решает:**
- ✅ **Orchestration** - автоматическое размещение контейнеров
- ✅ **Self-healing** - перезапуск упавших контейнеров
- ✅ **Auto-scaling** - горизонтальное масштабирование
- ✅ **Rolling updates** - обновление без downtime
- ✅ **Load balancing** - распределение нагрузки
- ✅ **Service discovery** - автоматическое обнаружение сервисов
- ✅ **Secret & config management** - управление конфигурацией

### Docker Swarm vs Kubernetes

| Feature | Docker Swarm | Kubernetes |
|---------|--------------|------------|
| Сложность | Простой | Сложный |
| Learning curve | Низкая | Высокая |
| Масштаб | Небольшой/средний | Любой |
| Ecosystem | Ограниченный | Огромный |
| Auto-scaling | Ручной | Автоматический (HPA) |
| Self-healing | Базовый | Продвинутый |
| Community | Меньше | Огромное |
| Use case | Простые проекты | Production-grade |

---

## 🏗️ Архитектура Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│                     Kubernetes Cluster                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────────────────────────────────────────┐      │
│  │              Control Plane (Master)              │      │
│  │                                                  │      │
│  │  ┌──────────────┐  ┌──────────────┐            │      │
│  │  │ API Server   │  │  Scheduler   │            │      │
│  │  └──────────────┘  └──────────────┘            │      │
│  │                                                  │      │
│  │  ┌──────────────┐  ┌──────────────┐            │      │
│  │  │   etcd       │  │  Controller  │            │      │
│  │  │ (key-value)  │  │   Manager    │            │      │
│  │  └──────────────┘  └──────────────┘            │      │
│  └──────────────────────────────────────────────────┘      │
│                           │                                 │
│                           │ kubectl commands                │
│                           ▼                                 │
│  ┌──────────────────────────────────────────────────┐      │
│  │                Worker Nodes                      │      │
│  │                                                  │      │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────┐ │      │
│  │  │   Node 1    │  │   Node 2    │  │  Node 3 │ │      │
│  │  │             │  │             │  │         │ │      │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │┌──────┐ │ │      │
│  │  │ │  Pod 1  │ │  │ │  Pod 3  │ │  ││ Pod 5│ │ │      │
│  │  │ └─────────┘ │  │ └─────────┘ │  │└──────┘ │ │      │
│  │  │ ┌─────────┐ │  │ ┌─────────┐ │  │         │ │      │
│  │  │ │  Pod 2  │ │  │ │  Pod 4  │ │  │         │ │      │
│  │  │ └─────────┘ │  │ └─────────┘ │  │         │ │      │
│  │  │             │  │             │  │         │ │      │
│  │  │  kubelet    │  │  kubelet    │  │ kubelet │ │      │
│  │  │  kube-proxy │  │  kube-proxy │  │kube-proxy│ │     │
│  │  └─────────────┘  └─────────────┘  └─────────┘ │      │
│  └──────────────────────────────────────────────────┘      │
└─────────────────────────────────────────────────────────────┘
```

### Control Plane Components

**1. API Server (kube-apiserver)**
- REST API для всех операций
- Frontend для Kubernetes
- Все компоненты общаются через API Server

**2. etcd**
- Распределённое key-value хранилище
- Хранит состояние кластера (desired state)
- Highly available

**3. Scheduler (kube-scheduler)**
- Решает, на какую node запустить pod
- Учитывает ресурсы (CPU, RAM), constraints, affinity

**4. Controller Manager (kube-controller-manager)**
- Запускает контроллеры (control loops)
- Следит чтобы actual state = desired state
- Примеры: ReplicaSet Controller, Deployment Controller

### Node Components

**1. kubelet**
- Агент на каждой node
- Запускает pods
- Следит за здоровьем pods
- Общается с API Server

**2. kube-proxy**
- Network proxy на каждой node
- Управляет сетевыми правилами
- Реализует Service abstraction

**3. Container Runtime**
- Docker, containerd, CRI-O
- Запускает контейнеры

---

## 🎯 Основные концепции

### Pod

**Pod** - минимальная единица развёртывания в K8s, группа из одного или нескольких контейнеров.

```yaml
# pod.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx:1.21
    ports:
    - containerPort: 80
```

**Характеристики:**
- Один pod = одно приложение (обычно 1 контейнер)
- Контейнеры в pod делят network namespace (localhost)
- Контейнеры в pod делят storage volumes
- Pod = ephemeral (может быть удалён и пересоздан)

**Multi-container pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: my-app:1.0
    ports:
    - containerPort: 8000
  - name: sidecar-logger
    image: logger:1.0
    volumeMounts:
    - name: logs
      mountPath: /var/log
  volumes:
  - name: logs
    emptyDir: {}
```

### ReplicaSet

**ReplicaSet** - обеспечивает заданное количество реплик pod.

```yaml
# replicaset.yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
spec:
  replicas: 3  # Всегда 3 экземпляра
  selector:
    matchLabels:
      app: nginx
  template:  # Pod template
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.21
        ports:
        - containerPort: 80
```

**Self-healing:**
```
Desired: 3 pods
Actual: 3 pods  ✅

Pod 2 crashes ❌
Actual: 2 pods

ReplicaSet Controller видит:
  Desired (3) != Actual (2)
  
Создаёт новый Pod 4
Actual: 3 pods  ✅
```

### Deployment

**Deployment** - декларативное управление pods и ReplicaSets.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  labels:
    app: laravel
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      containers:
      - name: laravel
        image: my-laravel-app:1.0
        ports:
        - containerPort: 9000
        env:
        - name: APP_ENV
          value: production
        - name: DB_HOST
          value: mysql-service
```

**Rolling Update:**
```bash
# Обновление образа
kubectl set image deployment/laravel-app laravel=my-laravel-app:2.0

# Процесс:
# 1. Создаёт новые pods с версией 2.0
# 2. Ждёт пока новые pods станут Ready
# 3. Удаляет старые pods с версией 1.0
# 4. Повторяет пока все pods не обновлены

# Zero downtime! ✅
```

**Rollback:**
```bash
# Откатиться к предыдущей версии
kubectl rollout undo deployment/laravel-app

# История
kubectl rollout history deployment/laravel-app

# Откат к конкретной ревизии
kubectl rollout undo deployment/laravel-app --to-revision=2
```

### Service

**Service** - абстракция для доступа к pods, load balancing.

**Проблема без Service:**
- Pods имеют динамические IP
- Pods могут быть пересозданы с новым IP
- Как обращаться к группе pods?

**Service решает:**
- Постоянный IP и DNS имя
- Load balancing между pods
- Service discovery

#### ClusterIP (default)

**Доступ только внутри кластера.**

```yaml
# service-clusterip.yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
spec:
  type: ClusterIP
  selector:
    app: laravel  # Pods с этим label
  ports:
  - port: 80       # Service port
    targetPort: 9000  # Pod port
```

```
┌────────────────────────────────────┐
│         laravel-service            │
│         ClusterIP: 10.0.0.5        │
└────────────┬───────────────────────┘
             │ Load balancing
    ┌────────┼────────┐
    ▼        ▼        ▼
 ┌─────┐ ┌─────┐ ┌─────┐
 │Pod 1│ │Pod 2│ │Pod 3│
 └─────┘ └─────┘ └─────┘
```

#### NodePort

**Доступ извне через порт на каждой node.**

```yaml
# service-nodeport.yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-nodeport
spec:
  type: NodePort
  selector:
    app: laravel
  ports:
  - port: 80
    targetPort: 9000
    nodePort: 30080  # 30000-32767
```

**Доступ:**
```
http://<NodeIP>:30080
```

#### LoadBalancer

**Создаёт внешний load balancer (AWS ELB, GCP LB).**

```yaml
# service-loadbalancer.yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-lb
spec:
  type: LoadBalancer
  selector:
    app: laravel
  ports:
  - port: 80
    targetPort: 9000
```

**Cloud provider создаёт:**
- External Load Balancer
- Публичный IP
- Направляет трафик на nodes

### Ingress

**Ingress** - HTTP(S) routing в сервисы, единая точка входа.

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: laravel-service
            port:
              number: 80
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**С TLS:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress-tls
spec:
  tls:
  - hosts:
    - example.com
    secretName: tls-secret
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: laravel-service
            port:
              number: 80
```

**Ingress Controller:**
```bash
# Установка Nginx Ingress Controller
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.0/deploy/static/provider/cloud/deploy.yaml
```

---

## 📦 ConfigMap и Secret

### ConfigMap

**ConfigMap** - хранение конфигурации (не секретной).

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: production
  APP_DEBUG: "false"
  LOG_LEVEL: info
  app.conf: |
    server {
      listen 80;
      server_name example.com;
    }
```

**Использование в Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: laravel-pod
spec:
  containers:
  - name: laravel
    image: my-laravel-app:1.0
    envFrom:
    - configMapRef:
        name: app-config
    # или отдельные переменные:
    env:
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV
```

### Secret

**Secret** - хранение чувствительных данных (base64 encoded).

```yaml
# secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  DB_PASSWORD: c2VjcmV0cGFzc3dvcmQ=  # base64 encoded
  API_KEY: YWJjMTIzNDU2Nzg5  # base64 encoded
```

**Создание Secret через kubectl:**
```bash
# Из literal
kubectl create secret generic db-secret \
  --from-literal=DB_PASSWORD=secretpassword \
  --from-literal=API_KEY=abc123456789

# Из файла
kubectl create secret generic db-secret \
  --from-file=password.txt

# TLS secret
kubectl create secret tls tls-secret \
  --cert=path/to/cert.pem \
  --key=path/to/key.pem
```

**Использование:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: laravel-pod
spec:
  containers:
  - name: laravel
    image: my-laravel-app:1.0
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: DB_PASSWORD
```

---

## 💾 Volumes и PersistentVolume

### emptyDir

**Временное хранилище, удаляется с pod.**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-with-emptydir
spec:
  containers:
  - name: app
    image: my-app
    volumeMounts:
    - name: cache
      mountPath: /var/cache
  volumes:
  - name: cache
    emptyDir: {}
```

### hostPath

**Монтирует директорию с node.**

```yaml
volumes:
- name: data
  hostPath:
    path: /data
    type: Directory
```

### PersistentVolume (PV) и PersistentVolumeClaim (PVC)

**PV** - ресурс хранилища в кластере (admin создаёт).
**PVC** - запрос на хранилище (user использует).

```yaml
# persistentvolume.yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mysql-pv
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /data/mysql
```

```yaml
# persistentvolumeclaim.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: standard
```

**Использование PVC:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: mysql-pod
spec:
  containers:
  - name: mysql
    image: mysql:8.0
    volumeMounts:
    - name: mysql-storage
      mountPath: /var/lib/mysql
  volumes:
  - name: mysql-storage
    persistentVolumeClaim:
      claimName: mysql-pvc
```

---

## 📊 Horizontal Pod Autoscaler (HPA)

**HPA** - автоматическое масштабирование pods на основе метрик.

```yaml
# hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: laravel-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: laravel-app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70  # 70% CPU
```

**Как работает:**
```
1. Metrics Server собирает метрики (CPU, RAM)
2. HPA Controller проверяет метрики каждые 15 сек
3. Если CPU > 70%:
   - Увеличить replicas (до maxReplicas)
4. Если CPU < 70%:
   - Уменьшить replicas (до minReplicas)
```

**Установка Metrics Server:**
```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

---

## 🎮 kubectl - основные команды

### Управление ресурсами

```bash
# Создать из файла
kubectl apply -f deployment.yaml
kubectl apply -f .  # все yaml в директории

# Удалить
kubectl delete -f deployment.yaml
kubectl delete deployment laravel-app
kubectl delete pod nginx-pod

# Получить список
kubectl get pods
kubectl get deployments
kubectl get services
kubectl get nodes

# С дополнительной информацией
kubectl get pods -o wide
kubectl get pods -o yaml
kubectl get pods -o json

# Watch (обновление в реальном времени)
kubectl get pods -w

# Все ресурсы
kubectl get all

# В определённом namespace
kubectl get pods -n kube-system

# Все namespaces
kubectl get pods --all-namespaces
kubectl get pods -A
```

### Просмотр деталей

```bash
# Детали ресурса
kubectl describe pod nginx-pod
kubectl describe deployment laravel-app
kubectl describe service laravel-service

# Логи pod
kubectl logs nginx-pod
kubectl logs nginx-pod -f  # follow
kubectl logs nginx-pod --tail=100
kubectl logs nginx-pod -c container-name  # multi-container pod

# Логи предыдущего контейнера (после crash)
kubectl logs nginx-pod --previous
```

### Выполнение команд

```bash
# Выполнить команду в pod
kubectl exec nginx-pod -- ls -la
kubectl exec nginx-pod -- cat /etc/nginx/nginx.conf

# Интерактивный shell
kubectl exec -it nginx-pod -- bash
kubectl exec -it nginx-pod -- sh

# В multi-container pod
kubectl exec -it pod-name -c container-name -- bash
```

### Копирование файлов

```bash
# Из pod на локальную машину
kubectl cp nginx-pod:/var/log/nginx/access.log ./access.log

# На pod
kubectl cp ./config.php nginx-pod:/var/www/html/config.php
```

### Масштабирование

```bash
# Изменить количество replicas
kubectl scale deployment laravel-app --replicas=5

# Autoscale
kubectl autoscale deployment laravel-app --min=2 --max=10 --cpu-percent=70
```

### Обновление

```bash
# Обновить образ
kubectl set image deployment/laravel-app laravel=my-laravel-app:2.0

# Статус rollout
kubectl rollout status deployment/laravel-app

# История
kubectl rollout history deployment/laravel-app

# Откат
kubectl rollout undo deployment/laravel-app
kubectl rollout undo deployment/laravel-app --to-revision=2

# Пауза/возобновление rollout
kubectl rollout pause deployment/laravel-app
kubectl rollout resume deployment/laravel-app
```

### Port forwarding

```bash
# Проброс порта из pod на локальную машину
kubectl port-forward pod/nginx-pod 8080:80
# Теперь http://localhost:8080 → pod:80

kubectl port-forward service/laravel-service 8000:80
```

### Namespaces

```bash
# Список namespaces
kubectl get namespaces

# Создать namespace
kubectl create namespace production

# Удалить namespace
kubectl delete namespace production

# Установить default namespace
kubectl config set-context --current --namespace=production
```

### Контекст и кластеры

```bash
# Текущий контекст
kubectl config current-context

# Список контекстов
kubectl config get-contexts

# Переключить контекст
kubectl config use-context minikube
kubectl config use-context production-cluster

# Просмотр конфигурации
kubectl config view
```

---

## 🚀 Пример: Laravel приложение в K8s

### Структура проекта

```
k8s/
├── deployment.yaml
├── service.yaml
├── ingress.yaml
├── configmap.yaml
├── secret.yaml
└── mysql/
    ├── deployment.yaml
    ├── service.yaml
    └── pvc.yaml
```

### MySQL

```yaml
# k8s/mysql/pvc.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
---
# k8s/mysql/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mysql
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mysql
  template:
    metadata:
      labels:
        app: mysql
    spec:
      containers:
      - name: mysql
        image: mysql:8.0
        env:
        - name: MYSQL_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mysql-secret
              key: password
        - name: MYSQL_DATABASE
          value: laravel
        ports:
        - containerPort: 3306
        volumeMounts:
        - name: mysql-storage
          mountPath: /var/lib/mysql
      volumes:
      - name: mysql-storage
        persistentVolumeClaim:
          claimName: mysql-pvc
---
# k8s/mysql/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: mysql
spec:
  selector:
    app: mysql
  ports:
  - port: 3306
    targetPort: 3306
  clusterIP: None  # Headless service
```

### Laravel App

```yaml
# k8s/secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: laravel-secret
type: Opaque
data:
  APP_KEY: base64_encoded_key
  DB_PASSWORD: base64_encoded_password
---
# k8s/configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: laravel-config
data:
  APP_ENV: production
  APP_DEBUG: "false"
  DB_CONNECTION: mysql
  DB_HOST: mysql
  DB_PORT: "3306"
  DB_DATABASE: laravel
  CACHE_DRIVER: redis
  QUEUE_CONNECTION: redis
  REDIS_HOST: redis
---
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: laravel-app
  labels:
    app: laravel
spec:
  replicas: 3
  selector:
    matchLabels:
      app: laravel
  template:
    metadata:
      labels:
        app: laravel
    spec:
      initContainers:
      - name: migrations
        image: my-laravel-app:latest
        command: ['php', 'artisan', 'migrate', '--force']
        envFrom:
        - configMapRef:
            name: laravel-config
        env:
        - name: APP_KEY
          valueFrom:
            secretKeyRef:
              name: laravel-secret
              key: APP_KEY
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: laravel-secret
              key: DB_PASSWORD
      containers:
      - name: laravel
        image: my-laravel-app:latest
        ports:
        - containerPort: 9000
        envFrom:
        - configMapRef:
            name: laravel-config
        env:
        - name: APP_KEY
          valueFrom:
            secretKeyRef:
              name: laravel-secret
              key: APP_KEY
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: laravel-secret
              key: DB_PASSWORD
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 9000
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /ready
            port: 9000
          initialDelaySeconds: 5
          periodSeconds: 5
---
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: laravel-service
spec:
  selector:
    app: laravel
  ports:
  - port: 80
    targetPort: 9000
  type: ClusterIP
---
# k8s/ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: laravel-ingress
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  tls:
  - hosts:
    - example.com
    secretName: laravel-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: laravel-service
            port:
              number: 80
---
# k8s/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: laravel-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: laravel-app
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

### Деплой

```bash
# Создать namespace
kubectl create namespace production

# Создать secrets
kubectl create secret generic mysql-secret \
  --from-literal=password=secretpassword \
  -n production

kubectl create secret generic laravel-secret \
  --from-literal=APP_KEY=base64:... \
  --from-literal=DB_PASSWORD=secretpassword \
  -n production

# Применить манифесты
kubectl apply -f k8s/ -n production

# Проверка
kubectl get all -n production
kubectl get pods -n production -w

# Логи
kubectl logs -f deployment/laravel-app -n production

# Доступ к приложению
kubectl port-forward service/laravel-service 8000:80 -n production
# http://localhost:8000
```

---

## 🎓 Для собеседования: ключевые точки

1. **Архитектура** - Control Plane (API Server, etcd, Scheduler, Controller Manager) + Worker Nodes (kubelet, kube-proxy, container runtime)
2. **Pod** - минимальная единица, один или несколько контейнеров, ephemeral, контейнеры в pod делят network/storage
3. **Deployment** - управление ReplicaSets и Pods, rolling updates, rollback, declarative updates
4. **Service** - абстракция для доступа к pods, ClusterIP (внутри кластера), NodePort (на node), LoadBalancer (external LB)
5. **Ingress** - HTTP(S) routing, единая точка входа, SSL/TLS termination, host-based/path-based routing
6. **ConfigMap/Secret** - ConfigMap для конфигурации (не секретной), Secret для чувствительных данных (base64)
7. **PV/PVC** - PersistentVolume (admin создаёт storage), PersistentVolumeClaim (user запрашивает), разделение ответственности
8. **HPA** - Horizontal Pod Autoscaler, масштабирование на основе CPU/RAM, minReplicas/maxReplicas
9. **kubectl** - CLI для K8s, apply/get/describe/logs/exec, rolling update (set image), rollback (rollout undo)
10. **Self-healing** - если pod упал, ReplicaSet Controller создаёт новый, desired state vs actual state

**Главное:** Понимай архитектуру (master/worker nodes), разницу Pod/ReplicaSet/Deployment, Service для доступа к pods (типы ClusterIP/NodePort/LoadBalancer), Ingress для HTTP routing, автоматическое масштабирование (HPA), declarative configuration через YAML манифесты.
