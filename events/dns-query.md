# `netstream.dns.query`

Один DNS-запрос, разобранный агентом на входе (XDP, UDP) и на выходе (TC, UDP и TCP/53). Несёт общий [envelope](envelope.md).

| Атрибут | Тип | Колонка ClickHouse |
|---|---|---|
| `network.io.direction` | `receive` или `transmit` | `direction` |
| `network.transport` | `tcp` или `udp` | `transport` |
| `source.address`, `destination.address` | IPv4 | `src_ip`, `dst_ip` |
| `dns.question.name` | string, нижний регистр | `qname` |
| `netstream.dns.question.type` | `A`, `TXT`, `NULL`, `AAAA`, … | `qtype` |
| `netstream.dns.qname.length`, `.labels`, `.longest_label` | int | `qname_length`, `label_count`, `longest_label` |
| `netstream.dns.qname.entropy`, `.digit_ratio` | double | `entropy`, `digit_ratio` |
| `netstream.dns.unique_subdomains` | int | `unique_subdomains` |

`unique_subdomains` считает уникальные поддомены **зарегистрированного** домена (по публичному списку суффиксов, `PUBLIC_SUFFIX_LIST_FILE`), а не по «последним двум меткам»: для `a.b.example.co.uk` это `example.co.uk`, а не `co.uk`. Список — только раздел ICANN; для доменов вроде `github.io`/`herokuapp.com`, не входящих в него, группировка совпадает со старым поведением (по последним двум меткам общего домена платформы, а не по фактически зарегистрированному домену конкретного пользователя).

## Ограничение DNS-over-TCP

Разбирается только запрос, целиком поместившийся в один TCP-сегмент (2-байтовая длина + сообщение по RFC 1035 §4.2.2). Запрос, разбитый на несколько сегментов, тихо пропускается — пересборка TCP-потока в eBPF не реализована.
