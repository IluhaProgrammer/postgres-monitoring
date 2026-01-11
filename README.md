# PostgreSQL Monitoring Lab

## 📌 Описание

Этот репозиторий содержит практическую лабораторную работу по **мониторингу PostgreSQL в продакшн-подобной среде** с использованием **Docker, Docker Compose, Prometheus, Grafana и postgres_exporter**.

Лаба сделана как часть обучения DevOps / SRE и нацелена на понимание **как выявлять проблемы с БД до инцидентов**, а не после падения сервиса.


## 🎯 Цели лабораторной

В результате выполнения лабы ты научишься:

- мониторить состояние PostgreSQL
- видеть **slow queries**
- отслеживать **locks / blocking queries**
- контролировать **connections / max_connections**
- отслеживать **replication lag**
- настраивать **алерты в Prometheus**
- понимать, *почему* сработал алерт


## 🧱 Используемые технологии

- PostgreSQL 16
- Docker / Docker Compose
- Prometheus
- Grafana
- postgres_exporter
- pg_stat_statements


## 📁 Структура проекта

```text
postgres-monitoring/
├── docker-compose.yml
├── alermanager/
│   └── alertmanager.yml
├── prometheus/
│   ├── prometheus.yml
│   └── rules/
│       └── rules.yml
└── README.dm
```


## 🚀 Запуск лабораторной

### 1. Клонировать репозиторий

```bash
git clone <repo-url>
cd postgres-monitoring
```

### 2. Запустить сервисы

```bash
docker compose up -d
```

### 3. Проверить, что всё запущено

```bash
docker ps
```

Должны быть запущены контейнеры:
- postgres
- postgres_exporter
- prometheus
- grafana

## 🛠 Инициализация PostgreSQL

Подключиться к БД:

```bash
docker exec -it postgres psql -U postgres -d prod_db
```

Включить расширение:

```sql
CREATE EXTENSION pg_stat_statements;
```

Проверка:

```sql
SELECT * FROM pg_stat_statements LIMIT 1;
```

## Prometheus

<img width="696" height="248" alt="image" src="https://github.com/user-attachments/assets/121f9e7f-c570-4f72-abc4-1f6ca1e43e1c" />

<img width="696" height="226" alt="image" src="https://github.com/user-attachments/assets/d4866e62-98a5-42f4-9d0d-c51c713df97d" />

Проверим, что у нас prometheus мониторит pg_exporter и правила алертов успешно подключены

## 📊 Grafana

### Доступ

- URL: http://localhost:3000
- Login: `admin`
- Password: `admin`

### Подключение Prometheus

- Data Source: **Prometheus**
- URL: `http://prometheus:9090`

### Дашборд

Рекомендуемый официальный дашборд:

- **PostgreSQL Exporter Dashboard**
- Grafana ID: **9628**

<img width="684" height="715" alt="image" src="https://github.com/user-attachments/assets/3a90931c-3d99-4d5a-95e5-d3287d96ffdd" />


## 🔍 Что мониторится

### 🐌 Slow Queries

- `pg_stat_statements_mean_time_ms`
- `pg_stat_statements_total_time_ms`

Пример запроса:

```sql
SELECT query, calls, mean_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 5;
```

<img width="694" height="316" alt="image" src="https://github.com/user-attachments/assets/d708632e-b646-4997-85e2-64b7df414775" />


### 🔒 Locks

Метрика:

- `pg_locks_count`

Проверка вручную:

```sql
SELECT * FROM pg_locks WHERE granted = false;
```
<img width="697" height="114" alt="image" src="https://github.com/user-attachments/assets/4c6c3e68-d170-4863-ac79-5dcc6e588afe" />


### 🔌 Connections

Метрики:

- `pg_stat_activity_count`
- `pg_settings_max_connections`

Цель — держать usage **< 70–80%**


### ⏱ Replication Lag

Метрика:

- `pg_replication_lag_seconds`

Важно для HA и failover сценариев.


## 🚨 Алерты (Prometheus)

### High Connections

```promql
pg_stat_activity_count / pg_settings_max_connections > 0.8
```

### Replication Lag

```promql
pg_replication_lag_seconds > 10
```

Алерты вынесены в файл:

```text
prometheus/alerts.yml
```


## 🧪 Симуляция проблем

### 1. Переполнение connections

```bash
for i in {1..50}; do
  psql postgresql://app:app@localhost:5432/app &
done
```


### 2. Slow Query

```sql
SELECT pg_sleep(3);
```


### 3. Lock

Session 1:
```sql
BEGIN;
UPDATE pg_class SET relname = relname;
```

Session 2:
```sql
UPDATE pg_class SET relname = relname;
```

<img width="295" height="256" alt="image" src="https://github.com/user-attachments/assets/1f7c2115-d7df-48b8-8238-36f4681225b0" />

На графике в Grafana мы видим, что количество locks выросло из за такой транзакции незаврешенный, long transactions, таких моментов стараться не допускать
