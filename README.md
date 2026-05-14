# VPN auto-subscription

Генерирует xray-подписку с автовыбором лучшего сервера из публичного списка VLESS-конфигов на GitHub. Обновляется раз в час через GitHub Actions, бесплатно.

## Что внутри

- `generate.py` — скрипт. Тянет источник, парсит VLESS-ссылки, пингует все сервера TCP-коннектом, отфильтровывает мёртвые, сортирует по latency, генерирует xray JSON с `burstObservatory` (постоянный мониторинг в реальном времени) и `balancer` со стратегией `leastLoad` (автовыбор быстрейшего).
- `.github/workflows/update.yml` — cron-job раз в час + при push'е, коммитит `docs/`.
- `docs/sub.json` — финальный xray JSON-конфиг **(основная подписка)**.
- `docs/sub.txt` — plain-список `vless://...` (для клиентов которые не понимают xray JSON).
- `docs/sub` — то же самое в Base64 (стандартный формат подписок для большинства клиентов).
- `docs/stats.json` — сколько серверов живо, лучший пинг, время обновления.

## Setup (3 шага)

### 1. Создай репо на GitHub

Залей все файлы в новый публичный репо. Например `username/vpn-sub`.

### 2. Включи GitHub Pages

- Settings → Pages
- Source: **Deploy from a branch**
- Branch: **main** / folder: **/docs**
- Save

Через минуту подписка будет доступна по адресам:

```
https://USERNAME.github.io/vpn-sub/sub.json   ← xray JSON (Happ, v2rayN, v2rayNG)
https://USERNAME.github.io/vpn-sub/sub        ← Base64 (универсальный)
https://USERNAME.github.io/vpn-sub/sub.txt    ← plain list
https://USERNAME.github.io/vpn-sub/stats.json ← мониторинг
```

### 3. Включи Actions

Actions → разреши запуск workflows.

Первый раз запусти руками: **Actions → Update subscription → Run workflow**. Через 30-60 секунд в `docs/` появятся файлы и закоммитятся.

Дальше — само раз в час, по cron.

## Какую подписку вставлять в клиент

| Клиент | URL |
|--------|-----|
| **Happ (iOS)** | `sub.json` или `sub` |
| **v2rayN (Windows)** | `sub.json` или `sub` |
| **v2rayNG (Android)** | `sub` (Base64) или `sub.json` |
| **Streisand (iOS)** | `sub.json` |
| **NekoBox (Android)** | `sub` |
| **sing-box-клиенты** | `sub.json` через конвертер, либо `sub` |

`sub.json` даёт **автовыбор сервера + автореконнект** — клиент будет постоянно пинговать сервера и переключаться на лучший сам.
`sub` (Base64) даёт **просто список** — переключение между серверами ручное.

## Настройка под себя

В начале `generate.py`:

```python
SOURCE_URL = "https://raw.githubusercontent.com/zieng2/wl/main/vless_lite.txt"
PING_TIMEOUT_SEC = 2.5      # таймаут пинга
PING_WORKERS = 50           # параллельных пингов
MAX_SERVERS = 80            # макс. серверов в финальном конфиге
```

Источников можешь добавить несколько — расширь `fetch_source()` чтобы тянул из списка URL и склеивал.

## Как это работает

1. **Server-side ping** (раз в час, GitHub Actions): отбрасывает мёртвые сервера до того как они попадут в твою подписку.
2. **Client-side ping** (постоянно, в самом клиенте через `burstObservatory`): пингует живые сервера каждые 3 минуты HTTP-запросом на `gstatic.com/generate_204`, оценивает реальную скорость, балансировщик `leastLoad` выбирает быстрейший.
3. **Автореконнект**: если активный сервер начинает лагать или падает — `burstObservatory` это замечает, балансировщик переключает трафик на следующий по скорости. Это встроенная фича xray.
4. **Обновление списка**: раз в час Actions тянет свежий файл с гитхаба источника, перегенерирует подписку. Клиент обновляет подписку по своему расписанию (в Happ это обычно настраивается отдельно — раз в сутки/час).

## Возможные доработки

- Добавить блок-листы (anti-AD, anti-tracker) в `routing.rules`
- Добавить GeoIP-роутинг (русские домены — direct, остальное — через VPN)
- Тянуть из нескольких источников и дедуплицировать по `host:port`
- Делать ICMP-пинг (требует root, не подойдёт на GitHub Actions без хаков; TCP-пинга на порт сервера достаточно)
- Telegram-уведомления через webhook когда живых серверов < N

## Локальный тест

```bash
python3 generate.py
# смотрим что получилось
cat docs/stats.json
```
