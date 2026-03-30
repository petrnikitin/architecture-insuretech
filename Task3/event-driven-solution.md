# Event-Driven архитектура InsureTech: Решение

## 1. Внедрение Event Streaming (Kafka)

Добавить Kafka для асинхронной передачи событий между сервисами.

**Компоненты:**
- Kafka Cluster — брокер сообщений
- Schema Registry — управление схемами событий (Avro/Protobuf)
- Kafka Connect — коннекторы для интеграции

---

## 2. Переход на Event-Driven взаимодействия

### 2.1. ins-product-aggregator → core-app, ins-comp-settlement

**Было:** Синхронные REST вызовы каждые 15 мин / раз в сутки

**Станет:**

**ins-product-aggregator:**
- Фоновый процесс опрашивает каждую страховую компанию независимо и асинхронно (каждые 5 мин)
- Публикует события в Kafka только при изменениях: `insurance.products.updated`, `insurance.tariffs.updated`
- Недоступность одной компании не блокирует другие (partial availability)

**core-app, ins-comp-settlement:**
- Подписываются на Kafka топики
- Обновляют локальные реплики в реальном времени
- Устраняется polling

**Результат:** Near real-time синхронизация, отказоустойчивость, снижение нагрузки на внешние API, loose coupling

---

### 2.2. core-app → ins-comp-settlement (оформленные страховки)

**Было:** Синхронный REST запрос раз в сутки для получения всех страховок

**Станет:**

**core-app:**
- Публикует событие `insurance.policies.created` при оформлении каждой страховки

**ins-comp-settlement:**
- Подписывается на топик
- Накапливает данные инкрементально в течение дня
- Ночью формирует реестр

**Результат:** Устранение тяжёлого batch-запроса, масштабируемость, обработка в реальном времени

---

## 3. Transactional Outbox

**Использовать для core-app:** ДА

**Цель:** Гарантировать атомарность сохранения страховки в БД и публикации события в Kafka.

**Реализация:**
- Создать таблицу `outbox_events` в БД core-app
- При оформлении страховки в одной транзакции сохранять запись в `policies` и событие в `outbox_events`
- Отдельный процесс (Outbox Publisher) читает события из `outbox_events` и публикует в Kafka
- Альтернатива: Debezium (Change Data Capture) — мониторит WAL PostgreSQL и автоматически публикует изменения в Kafka

**Для ins-product-aggregator Transactional Outbox не требуется** — допустимо редкое дублирование событий (idempotent consumers).

---

## 4. Дополнительные улучшения

- **Dead Letter Queue (DLQ)** — изоляция проблемных сообщений
- **Event Versioning** — управление версиями схем через Schema Registry
- **Idempotent Consumers** — дедупликация событий по `event_id`
- **Monitoring** — Consumer Lag, Event Tracing, Alerting

---

## Топики Kafka

1. **insurance.products.updated** (Publisher: ins-product-aggregator → Subscribers: core-app, ins-comp-settlement)
2. **insurance.tariffs.updated** (Publisher: ins-product-aggregator → Subscribers: core-app, ins-comp-settlement)
3. **insurance.policies.created** (Publisher: core-app → Subscribers: ins-comp-settlement)

---

## Результаты внедрения

| Проблема | Решение |
|----------|---------|
| Медленные синхронные запросы к 10 компаниям | Асинхронный опрос, независимая публикация событий |
| Cascading failures | Partial availability |
| Polling каждые 15 мин/сутки | Push-модель через события |
| Устаревшие данные | Near real-time синхронизация |
| Tight coupling | Loose coupling через Kafka |
| Отсутствие retry | Kafka retry + DLQ |
| Тяжёлый batch-запрос страховок | Инкрементальная передача |
| Отсутствие audit trail | Kafka retention, возможность replay |
