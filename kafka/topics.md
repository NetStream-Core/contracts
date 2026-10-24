# Kafka topics

| Топик | Содержимое | Кодировка | Партиции | Retention |
|---|---|---|---|---|
| `netstream.events.v1` | OTLP-логи: `netstream.flow`, `netstream.dns.query`, `netstream.blocklist.hit` | `otlp_proto` | 3 | 7 дней (`retention.ms=604800000`) |
| `netstream.metrics.v1` | OTLP-метрики агента (`netstream_*` — накладные расходы, см. `deploy/README.md`) | `otlp_proto` | 3 | 7 дней |
| `netstream.detections.v1` | Результат оценки каждого закрывшегося окна онлайн-детектором: `host_id`, `src_ip`, `window_start`, `window_seconds`, `score`, `is_attack`, `latency_ms` | `JSONEachRow` (не OTLP — не телеметрия, а результат уже обработанных данных) | 3 | 7 дней (`retention.ms=604800000`) |

Все три топика создаются идемпотентно (`--create --if-not-exists`) сервисом `kafka-init` в `deploy/compose.yml`; `max.message.bytes=4194304` (4 МиБ, запас на батч OTLP-записей/детекций).

`netstream.events.v1` создаётся с тремя партициями, и Kafka не даёт порядок между разными партициями — только в пределах одной. `otel-edge` включает `partition_logs_by_resource_attributes: true` (`deploy/collector/edge.yaml`, `edge-tls.yaml`), так что партиция считается хэшем ресурсных атрибутов батча (`host.id`, `service.instance.id`) — все записи одного сенсора за время жизни одного процесса агента стабильно попадают в одну партицию, что нужно `WindowAccumulator` (и `ml`'s `OnlineDetector`, и будущим его аналогам) для корректного закрытия окон по порядку прибытия.

## Производители и потребители

- **Producer `netstream.events.v1`/`netstream.metrics.v1`**: `otel-edge` (принимает OTLP от агентов, пишет в оба топика по типу сигнала).
- **Consumer `netstream.events.v1`**: два независимых consumer'а, каждый со своей группой — ни один не зависит от другого и не блокирует его:
  - `otel-gateway`, группа `netstream-gateway` — пишет в ClickHouse (`otel_logs`, `otel_metrics_gauge`/`_sum`/`_histogram`).
  - `ml`'s онлайн-детектор (`netstream_ml.detector`, `detect run`), группа задаётся явно через `--consumer-group` (по умолчанию `netstream-detector`) — кормит `WindowAccumulator`, оценивает закрывшиеся окна обученной моделью, публикует результат в `netstream.detections.v1`.
- **Consumer `netstream.metrics.v1`**: `otel-gateway` (та же группа `netstream-gateway`).
- **Producer `netstream.detections.v1`**: `ml`'s онлайн-детектор. Входной офсет `netstream.events.v1` коммитится вручную, только после того как публикация в `netstream.detections.v1` подтверждена брокером — см. `ml/docs/online-detector.md`, раздел «Надёжность: ручной commit, идемпотентный producer, дедупликация детекций».
- **Consumer `netstream.detections.v1`**: ClickHouse — напрямую, через таблицу с движком `Kafka` (группа `clickhouse-detections`) и materialized view в `detections` (`deploy/clickhouse/migrations/015_detections.sql`), без промежуточного сервиса.

Разделение `netstream.events.v1`/`netstream.metrics.v1` по типу сигнала (а не один общий топик) позволяет масштабировать/настраивать consumer'ов независимо для логов и метрик, если один поток станет узким местом — сейчас оба идут в один и тот же `otel-gateway`, разделение уже сделано на будущее. `netstream.detections.v1` — отдельный топик с другим форматом сообщений (не OTLP), поскольку это уже не телеметрия, а результат её обработки.

## Пока не существует (план)

`netstream.alerts.v1` — появится вместе с коррелятором инцидентов (группировка повторных/связанных детекций в один инцидент, таблица `incidents` в PostgreSQL `backend` уже создана, обработчиков ещё нет); заранее не заводится, чтобы не плодить пустой топик без потребителя.
