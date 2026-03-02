# Под ключ: мост между двумя Telegram-ботами (VPS Linux + Windows)

Ниже схема, которая обходит ограничение Telegram `bot -> bot` в группе без правок логики самих ботов.

## 1) Архитектура

```text
Telegram Group
   | (сообщения от людей)
   v
Bot A (VPS) ---------------------> Relay API (VPS)
   ^                                   |
   |                                   v
Bot B (Windows) <---------------- Redis Streams (VPS)
```

- Боты **не общаются напрямую** в Telegram.
- Каждый бот получает только сообщения от людей в группе (как обычно).
- `Relay API` выступает шиной событий: принимает события от одного окружения и доставляет второму.
- Используем поля `origin_bot`, `event_id`, `ttl`, чтобы исключить зацикливание.

## 2) Что разворачиваем

На VPS (Linux):
1. Redis
2. Небольшой `relay`-сервис (FastAPI)
3. Nginx + HTTPS (опционально, но рекомендуется)

На Windows:
1. Лёгкий агент `bridge-client` (Python), который слушает поток Redis для Bot B
2. Agent публикует события от Bot B в relay

## 3) Формат события

```json
{
  "event_id": "uuid-v4",
  "origin_bot": "bot_a|bot_b",
  "chat_id": -100123,
  "thread_id": 0,
  "text": "...",
  "reply_to_message_id": 123,
  "created_at": "2026-03-02T12:34:56Z",
  "ttl": 1
}
```

Правила:
- `event_id` уникален (dedup 5 минут).
- `origin_bot` обязателен.
- `ttl` при пересылке уменьшается, при `ttl <= 0` событие отбрасывается.

## 4) Минимальные эндпоинты relay

- `POST /publish` — принять событие и положить в Redis Stream целевого бота.
- `GET /health` — проверка здоровья.

Потоки Redis:
- `bridge:to:bot_a`
- `bridge:to:bot_b`
- `bridge:dedup` (ключи `event_id` с TTL)

## 5) Запуск на VPS (Ubuntu)

### 5.1 Установка

```bash
sudo apt update
sudo apt install -y redis-server python3-venv nginx
sudo systemctl enable --now redis-server
```

### 5.2 Relay (FastAPI)

```bash
mkdir -p /opt/bot-bridge && cd /opt/bot-bridge
python3 -m venv .venv
source .venv/bin/activate
pip install fastapi uvicorn redis pydantic
```

Создать `relay.py`:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from redis import Redis
import json

app = FastAPI()
r = Redis(host="127.0.0.1", port=6379, decode_responses=True)

class Event(BaseModel):
    event_id: str
    origin_bot: str
    chat_id: int
    thread_id: int | None = None
    text: str
    reply_to_message_id: int | None = None
    created_at: str
    ttl: int = 1

@app.get("/health")
def health():
    return {"ok": True}

@app.post("/publish")
def publish(e: Event):
    if e.origin_bot not in ("bot_a", "bot_b"):
        raise HTTPException(400, "origin_bot must be bot_a|bot_b")

    if e.ttl <= 0:
        return {"skipped": "ttl"}

    dedup_key = f"dedup:{e.event_id}"
    if not r.setnx(dedup_key, 1):
        return {"skipped": "duplicate"}
    r.expire(dedup_key, 300)

    target = "bridge:to:bot_b" if e.origin_bot == "bot_a" else "bridge:to:bot_a"
    payload = e.model_dump()
    payload["ttl"] = e.ttl - 1

    r.xadd(target, {"event": json.dumps(payload, ensure_ascii=False)})
    return {"ok": True, "target": target}
```

### 5.3 systemd юнит

`/etc/systemd/system/bot-bridge.service`

```ini
[Unit]
Description=Telegram Bot Bridge Relay
After=network.target redis-server.service

[Service]
User=root
WorkingDirectory=/opt/bot-bridge
ExecStart=/opt/bot-bridge/.venv/bin/uvicorn relay:app --host 0.0.0.0 --port 8088
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now bot-bridge
curl http://127.0.0.1:8088/health
```

## 6) Windows-агент (для Bot B)

```bash
py -m venv C:\bot-bridge\.venv
C:\bot-bridge\.venv\Scripts\pip install redis requests
```

`C:\bot-bridge\agent_b.py`:

```python
import json
import uuid
import requests
from redis import Redis

REDIS_HOST = "<VPS_IP>"
REDIS_PORT = 6379
RELAY_URL = "https://<YOUR_DOMAIN>/publish"
BOT_B_INBOX = "bridge:to:bot_b"

r = Redis(host=REDIS_HOST, port=REDIS_PORT, decode_responses=True)
last_id = "$"

while True:
    rows = r.xread({BOT_B_INBOX: last_id}, block=5000, count=10)
    for stream, messages in rows:
        for msg_id, fields in messages:
            last_id = msg_id
            event = json.loads(fields["event"])
            # TODO: вызвать локальный интерфейс Bot B (CLI/HTTP), чтобы он отправил ответ в группу
            # После получения ответа от Bot B:
            response_text = f"[bot_b processed] {event['text']}"
            out = {
                "event_id": str(uuid.uuid4()),
                "origin_bot": "bot_b",
                "chat_id": event["chat_id"],
                "thread_id": event.get("thread_id"),
                "text": response_text,
                "reply_to_message_id": event.get("reply_to_message_id"),
                "created_at": "2026-03-02T00:00:00Z",
                "ttl": 1
            }
            requests.post(RELAY_URL, json=out, timeout=10)
```

> Аналогичный агент делается для Bot A, если нужен двунаправленный обмен.

## 7) Как подключить без правки кода самих ботов

Вариант A (предпочтительно):
- Если nanobot уже имеет HTTP/API или CLI-вызов "отправь сообщение" — агент просто вызывает этот интерфейс.
- Это не требует изменения исходников бота.

Вариант B (полностью внешняя обвязка):
- Ставим middleware перед webhook каждого бота.
- Middleware дублирует входящие апдейты в relay и может отправлять "инъекции" апдейтов в локальный бот-процесс.

## 8) Анти-цикл и надёжность (обязательно)

1. Dedup по `event_id` (TTL 300 сек)
2. `ttl = 1` (или 2 максимум)
3. Проверка `origin_bot` (не отправлять обратно туда же)
4. Retry с backoff на сетевые ошибки
5. Логи: `event_id`, `source`, `target`, `result`

## 9) Быстрый план внедрения за 20 минут

1. Поднять Redis + relay на VPS
2. Проверить `POST /publish` вручную через `curl`
3. Запустить Windows-агент
4. Привязать агент к способу отправки сообщений через Bot B
5. Добавить systemd/Task Scheduler автозапуск
6. Проверить цикл A->relay->B и B->relay->A

## 10) Пример ручной проверки

```bash
curl -X POST http://127.0.0.1:8088/publish \
  -H 'content-type: application/json' \
  -d '{
    "event_id":"11111111-1111-1111-1111-111111111111",
    "origin_bot":"bot_a",
    "chat_id":-100123,
    "text":"hello from a",
    "created_at":"2026-03-02T12:00:00Z",
    "ttl":1
  }'
```

Ожидаемо: событие окажется в `bridge:to:bot_b`, а бот B ответит в группу уже как самостоятельный бот.
