# Kafka topics

| Топик | Содержимое | Кодировка | Партиции | Retention |
|---|---|---|---|---|
| `netstream.events.v1` | OTLP-логи: `netstream.flow`, `netstream.dns.query`, `netstream.blocklist.hit` | `otlp_proto` | 3 | 7 дней (`retention.ms=604800000`) |
| `netstream.metrics.v1` | OTLP-метрики агента (`netstream_*` — накладные расходы, см. `deploy/README.md`) | `otlp_proto` | 3 | 7 дней |

Оба топика создаются идемпотентно (`--create --if-not-exists`) сервисом `kafka-init` в `deploy/compose.yml`; `max.message.bytes=4194304` (4 МиБ, запас на батч OTLP-записей).

## Производители и потребители

- **Producer**: `otel-edge` (принимает OTLP от агентов, пишет в оба топика по типу сигнала).
- **Consumer**: `otel-gateway`, группа `netstream-gateway` — читает оба топика, пишет в ClickHouse (`otel_logs`, `otel_metrics_gauge`/`_sum`/`_histogram`).

Разделение на два топика по типу сигнала (а не один общий) позволяет масштабировать/настраивать consumer'ов независимо для логов и метрик, если один поток станет узким местом — сейчас оба идут в один и тот же `otel-gateway`, разделение уже сделано на будущее.

## Пока не существует (план)

`netstream.detections.v1`, `netstream.alerts.v1` — появятся вместе с онлайн-детектором (M16) и коррелятором инцидентов; заранее не заводятся, чтобы не плодить пустые топики без потребителя.
