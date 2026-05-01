# Звіт про виявлені помилки та виправлення — AI-сервер

**Дата:** 2026-05-01  
**Сервер:** Ubuntu 25.04 | AMD Ryzen 9000x | 2× RTX PRO 4000 Blackwell | 59 GB RAM  
**Стек:** Ollama 0.22.0 · Paperclip 2026.428.0 · OpenClaw · Qdrant 1.17.1 · FastAPI RAG

---

## Помилка №1 — Paperclip: SKILL.md читав `.json` як JSON-об'єкт

### Симптом
Агент CEO падав з помилкою авторизації при кожному wake event. У transcript run CMP-24 було видно:
```
401 Unauthorized
```
Auth-запити до `http://localhost:3100/api/...` не проходили.

### Причина
У файлі `~/.openclaw/workspace/skills/paperclip/SKILL.md` було прописано:

```bash
# НЕПРАВИЛЬНО
PAPERCLIP_API_KEY=$(cat ~/.openclaw/workspace/paperclip-claimed-api-key.json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

Але файл `paperclip-claimed-api-key.json` містить **просто рядок токена**, не JSON-об'єкт:
```
pcp_37af21c4551c37616d0226c0b3d0056902f937ee217dce7c
```

Тому `json.load()` кидав `JSONDecodeError` → `PAPERCLIP_API_KEY` залишався порожнім → `401`.

Справжній JSON-файл з полем `token` — це `paperclip-claimed-api-key.full.json`:
```json
{
  "keyId": "9dd0991a-...",
  "token": "pcp_37af21c4...",
  "paperclipApiUrl": "http://localhost:3100",
  ...
}
```

### Виправлення
```bash
# ПРАВИЛЬНО — читати з full.json через jq
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
PAPERCLIP_URL=$(jq -r '.paperclipApiUrl' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
```

**Файли змінено:**
- `~/.openclaw/workspace/skills/paperclip/SKILL.md` — виправлено команду читання ключа

---

## Помилка №2 — Системний SKILL.md: приклад "Correct" вказував на неправильний файл

### Симптом
Системний файл `~/.openclaw/skills/paperclip/SKILL.md` містив правило:

```
Wrong:
PAPERCLIP_API_KEY=$(cat ~/.openclaw/workspace/paperclip-claimed-api-key.json)

Correct:
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.json)
```

Але "Correct" також використовував `.json` (не JSON-файл) — тому `jq` повертав `null`.

### Виправлення
```
Correct:
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
PAPERCLIP_API_URL=$(jq -r '.paperclipApiUrl' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
```

Додано правило: для адаптера `openclaw_gateway` **`PAPERCLIP_API_KEY` авто-інжектується** як short-lived JWT — вручну встановлювати не потрібно взагалі.

**Файли змінено:**
- `~/.openclaw/skills/paperclip/SKILL.md` — виправлено приклад та додано правило авто-інжекції

---

## Помилка №3 — Агент описував кроки замість виконання

### Симптом
При кожному wake event агент (CEO, модель `qwen2.5:32b`) генерував текстовий опис процедури замість її виконання:

```
Step 1: Load Environment Variables...
Step 2: Retrieve API Key...
Step 3: Get Agent Details...
Tool Call: exec to GET /api/agents/me
[Streaming]
run completed status=ok
```

Фактична задача (наприклад, написати код для ESP32) ніколи не виконувалась. Continuation summary містив лише опис кроків без пробілів — ознака що агент скопіював свій же текстовий output в коментар.

### Причина
SKILL.md не містив явної заборони на опис кроків у тексті. Модель `qwen2.5:32b` схильна описувати процедуру, а не виконувати її через exec-інструмент.

Також агент зайво робив `GET /api/agents/me` на кожному wake event, хоча у wake-контексті вже є `PAPERCLIP_TASK_ID` — і можна одразу йти до задачі.

### Виправлення
Workspace SKILL.md повністю переписаний з двома критичними правилами на початку:

```
⚠️ КРИТИЧНО: НЕ встановлюй PAPERCLIP_API_KEY вручну
⚠️ КРИТИЧНО: Виконуй задачу — не описуй процедуру
```

Алгоритм скорочено: якщо є `$PAPERCLIP_TASK_ID` — пропускати `GET /agents/me`, одразу до задачі.

**Файли змінено:**
- `~/.openclaw/workspace/skills/paperclip/SKILL.md` — повністю переписаний

---

## Помилка №4 — Paperclip та OpenClaw не мали systemd автозапуску

### Симптом
Після перезавантаження сервера `paperclip` та `openclaw-gateway` не запускались автоматично. Запуск був виключно ручним через термінал.

### Причина
`paperclipai` не був встановлений глобально — запускався через `npx` з кешованим нестабільним шляхом:
```
node /home/leo/.npm/_npx/43414d9b790239bb/node_modules/.bin/paperclipai onboard --yes
```
Шлях `43414d9b790239bb` може змінитися після оновлення пакету.

Systemd user-юніти для цих сервісів були відсутні.

### Виправлення

1. Встановлено `paperclipai` глобально:
```bash
npm install -g paperclipai
# → /home/leo/.nvm/versions/node/v22.22.2/bin/paperclipai
```

2. Створено systemd user-юніти:

**`~/.config/systemd/user/paperclip.service`**
```ini
[Unit]
Description=Paperclip AI control plane
After=network.target

[Service]
Type=simple
Environment="PATH=/home/leo/.nvm/versions/node/v22.22.2/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/home/leo/.nvm/versions/node/v22.22.2/bin/paperclipai onboard --yes
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

**`~/.config/systemd/user/openclaw-gateway.service`**
```ini
[Unit]
Description=OpenClaw Gateway
After=network.target paperclip.service
Wants=paperclip.service

[Service]
Type=simple
Environment="PATH=/home/leo/.nvm/versions/node/v22.22.2/bin:/usr/local/bin:/usr/bin:/bin"
WorkingDirectory=/home/leo
ExecStart=/home/leo/.nvm/versions/node/v22.22.2/bin/openclaw gateway --port 18789
Restart=on-failure
RestartSec=10

[Install]
WantedBy=default.target
```

3. Увімкнено автозапуск (linger вже був активний):
```bash
systemctl --user daemon-reload
systemctl --user enable paperclip.service openclaw-gateway.service
```

---

## Підсумок виправлень

| # | Компонент | Помилка | Статус |
|---|-----------|---------|--------|
| 1 | `workspace/skills/paperclip/SKILL.md` | `python3 json.load()` на не-JSON файлі → порожній API ключ | ✅ Виправлено |
| 2 | `skills/paperclip/SKILL.md` (системний) | "Correct" приклад вказував на `.json` замість `.full.json` | ✅ Виправлено |
| 3 | CEO агент (qwen2.5:32b) | Описував кроки замість виконання, не доходив до задачі | ✅ Виправлено |
| 4 | systemd | Немає автозапуску після ребуту | ✅ Виправлено |

## Поточний стан системи

```
paperclip.service       active (running) — enabled
openclaw-gateway.service active (running) — enabled  
ollama.service          active (running) — enabled
Qdrant :6333            OK (v1.17.1)
Paperclip API :3100     200 OK
Gateway ws://127.0.0.1:18789  reachable
```
