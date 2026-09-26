# `netstream.blocklist.hit`

Срабатывание блок-листа доменов — DNS-запрос совпал с записью в `malware_domains.txt` (или доне обновлённой через hot-reload). Несёт общий [envelope](envelope.md).

| Атрибут | Тип | Колонка ClickHouse |
|---|---|---|
| `source.address` | IPv4 | `src_ip` |
| `netstream.hit.domain` | string | `domain` |
| `netstream.hit.action` | `observed`, `dropped`, `quarantined` | `action` |

`action` зависит от `RESPONSE_MODE` агента на момент срабатывания:

| `RESPONSE_MODE` | `action` | Поведение |
|---|---|---|
| `monitor` (по умолчанию) | `observed` | Запрос пропускается, только фиксируется событие |
| `enforce` | `dropped` | Запрос дропается на месте |
| `gateway` | `quarantined` | Запрос дропается, источник ставится в карантин на `QUARANTINE_TTL_SECS` (по умолчанию 60 с) — все его пакеты дропаются, пока запись не истечёт или адрес не окажется в `quarantine_allowlist` |

Снятие карантина раньше TTL сейчас не поддерживается: убрать домен из блок-листа останавливает **новые** срабатывания (подхватывается hot-reload'ом за `RELOAD_POLL_MS`, без перезапуска агента), но уже наложенный карантин на конкретный IP продолжает действовать до истечения TTL или явного присутствия адреса в `QUARANTINE_ALLOWLIST` (требует перезапуска агента — список не hot-reload'ится).
