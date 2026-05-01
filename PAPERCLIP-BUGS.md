# Paperclip + OpenClaw — Виявлені помилки та виправлення

**Дата діагностики:** 2026-05-01  
**Версії:** Paperclip 2026.428.0 · OpenClaw 2026.4.29 · Ubuntu 25.04  
**Модель агента:** ollama/qwen2.5:32b → ollama/qwen2.5-coder:32b

---

## Помилка №1 — `json.load()` на файлі без JSON → порожній API ключ → 401

### Симптом
Кожен wake event для агента CEO падав з `401 Unauthorized`.  
У Paperclip transcript (run CMP-24) було видно:
```
run completed status=ok   ← але жодного реального API виклику не відбувалось
```

### Причина
У `~/.openclaw/workspace/skills/paperclip/SKILL.md` рядок:
```bash
PAPERCLIP_API_KEY=$(cat ~/.openclaw/workspace/paperclip-claimed-api-key.json \
  | python3 -c "import sys,json; print(json.load(sys.stdin)['token'])")
```

**Файл `paperclip-claimed-api-key.json` містить лише сирий рядок токена**, не JSON:
```
pcp_37af21c4551c37616d0226c0b3d0056902f937ee217dce7c
```

`json.load()` кидав `JSONDecodeError` → `PAPERCLIP_API_KEY=""` → `401`.

Справжній JSON з полем `token` лежить в **`paperclip-claimed-api-key.full.json`**:
```json
{
  "keyId": "9dd0991a-...",
  "token": "pcp_37af21c4...",
  "paperclipApiUrl": "http://localhost:3100"
}
```

### Виправлення
```bash
# ПРАВИЛЬНО — читати з full.json через jq
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
PAPERCLIP_URL=$(jq -r '.paperclipApiUrl' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
```

**Файл:** `~/.openclaw/workspace/skills/paperclip/SKILL.md`

---

## Помилка №2 — Системний SKILL.md: приклад "Correct" вказував на неправильний файл

### Симптом
`~/.openclaw/skills/paperclip/SKILL.md` (системний файл OpenClaw) містив:
```
Wrong:
PAPERCLIP_API_KEY=$(cat ~/.openclaw/workspace/paperclip-claimed-api-key.json)

Correct:
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.json)
```

Приклад "Correct" також використовував `.json` (не JSON файл) — `jq` повертав `null`.

### Виправлення
```
Correct:
PAPERCLIP_API_KEY=$(jq -r '.token' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
PAPERCLIP_API_URL=$(jq -r '.paperclipApiUrl' ~/.openclaw/workspace/paperclip-claimed-api-key.full.json)
```

Також додано правило: для адаптера `openclaw_gateway` `PAPERCLIP_API_KEY`
**авто-інжектується як short-lived JWT** — встановлювати вручну не потрібно.

**Файл:** `~/.openclaw/skills/paperclip/SKILL.md`

---

## Помилка №3 — Агент генерував текстовий опис замість виконання задачі

### Симптом
При кожному wake event агент (`qwen2.5:32b`, адаптер `openclaw_gateway`)
генерував у відповідь текстовий опис кроків:

```
Step 1: Load Environment Variables...
  export PAPERCLIP_RUN_ID=...
Step 2: Retrieve API Key...
  export PAPERCLIP_API_KEY=$(cat ~/.openclaw/workspace/paperclip-claimed-api-key.json)
Step 3: Get Agent Details...
  Tool Call: exec to GET /api/agents/me
[run completed status=ok]
```

Реальна задача (написати код для ESP32) **ніколи не виконувалась**.  
Continuation summary в Paperclip накопичував цей текст злипленим рядком без пробілів,
що отруювало контекст наступних wake event'ів.

### Причина
Два незалежних фактори:
1. **Модель** (`qwen2.5:32b`) має в тренувальних даних цей "Step 1/Step 2/cat .json"
   патерн для Paperclip heartbeat — і генерує його автоматично.
2. **SKILL.md не завантажується автоматично** — OpenClaw повідомляє агенту список
   доступних skills і каже "Use the read tool to load a skill's file when needed".
   Але wake event message від Paperclip містить: _"Run this procedure now.
   Do not guess undocumented endpoints"_ — модель слідує цій директиві
   і не читає SKILL.md.

### Виправлення

**Крок 1** — Додано правило в `SOUL.md` (завантажується першим при кожній сесії):
```markdown
## ⛔ PAPERCLIP WAKE EVENT — АБСОЛЮТНЕ ПРАВИЛО

Коли отримуєш повідомлення що починається з "Paperclip wake event":
1. ПЕРША ДІЯ — прочитай skill файл: ~/.openclaw/workspace/skills/paperclip/SKILL.md
2. ЗАБОРОНЕНО писати "Step 1/Step 2/Step 3..." у тексті відповіді
3. ЗАБОРОНЕНО будь-який `cat paperclip-claimed-api-key` — використовуй $PAPERCLIP_API_KEY
4. ОДРАЗУ виконуй задачу через exec без текстових описів кроків
```

**Крок 2** — Оновлено description SKILL.md щоб тригерити завантаження:
```yaml
description: "ОБОВ'ЯЗКОВО читати цей файл ПЕРШИМ при будь-якому Paperclip wake event"
```

**Крок 3** — Додано правила в `TOOLS.md` (резервна інструкція):
```markdown
## ⚠️ PAPERCLIP — ОБОВ'ЯЗКОВІ ПРАВИЛА
`PAPERCLIP_API_KEY` авто-інжектується. Не встановлюй вручну.
Завжди подвійні лапки: -H "Authorization: Bearer $PAPERCLIP_API_KEY"
```

**Файли:** `SOUL.md`, `skills/paperclip/SKILL.md`, `TOOLS.md`

---

## Помилка №4 — Персистентна сесія накопичувала хибний патерн

### Симптом
Після першого запуску з хибним "Step 1/Step 2" патерном кожен наступний
wake event для тієї ж задачі (CMP-24) повторював його — навіть після виправлення SKILL.md.

### Причина
OpenClaw зберігає conversation history сесій у `.jsonl` файлах.
Сесія `agent:main:paperclip:issue:6430cfca-...` накопичила 5 запусків
з однаковим хибним патерном. Модель бачила цю history і генерувала аналогічну
відповідь ("продовжувала розмову").

Файли сесій:
```
~/.openclaw/agents/main/sessions/48d150a3-*.jsonl
~/.openclaw/agents/main/sessions/sessions.json
```

### Виправлення
Видалено всі 9 paperclip-сесій з `sessions.json` та `.jsonl` файли:
```python
# Видалено ключі типу:
# agent:main:paperclip
# agent:main:paperclip:issue:6430cfca-...
# (+ 7 інших issue-сесій)
```
Перезапущено `openclaw-gateway.service`.

---

## Помилка №5 — curl команди з одинарними лапками не розкривали змінні

### Симптом
У exec tool calls модель генерувала:
```bash
curl ... -H 'Authorization: Bearer $PAPERCLIP_API_KEY'
```
В bash одинарні лапки **блокують** розкриття змінних — `$PAPERCLIP_API_KEY`
надсилалось би буквально.

### Чому не ламало роботу
Exec tool OpenClaw виконує команди через `bash -c "..."` (подвійні лапки на зовнішньому
рівні). Тому навіть `$(cat ...)` в одинарних лапках всередині **розкривається**
до передачі bash — curl отримував правильний токен.

### Виправлення
Додано в `SOUL.md` та `TOOLS.md` правило про подвійні лапки:
```bash
# ПРАВИЛЬНО
curl -H "Authorization: Bearer $PAPERCLIP_API_KEY"

# НЕПРАВИЛЬНО (але технічно працює через exec tool)
curl -H 'Authorization: Bearer $PAPERCLIP_API_KEY'
```

---

## Помилка №6 — Відсутній systemd автозапуск

### Симптом
Після перезавантаження сервера `paperclip` та `openclaw-gateway` не стартували.
`paperclipai` запускався через `npx` з нестабільним кешованим шляхом.

### Виправлення

Встановлено глобально:
```bash
npm install -g paperclipai
```

Створено user systemd units (`~/.config/systemd/user/`):

**`paperclip.service`**
```ini
[Unit]
Description=Paperclip AI control plane
After=network.target

[Service]
Environment="PATH=/home/leo/.nvm/versions/node/v22.22.2/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/home/leo/.nvm/versions/node/v22.22.2/bin/paperclipai onboard --yes
Restart=on-failure
RestartSec=10
```

**`openclaw-gateway.service`**
```ini
[Unit]
Description=OpenClaw Gateway
After=network.target paperclip.service

[Service]
Environment="PATH=/home/leo/.nvm/versions/node/v22.22.2/bin:/usr/local/bin:/usr/bin:/bin"
ExecStart=/home/leo/.nvm/versions/node/v22.22.2/bin/openclaw gateway --port 18789
Restart=on-failure
RestartSec=10
```

```bash
systemctl --user enable paperclip.service openclaw-gateway.service
```

---

## Підсумок

| # | Компонент | Помилка | Результат |
|---|-----------|---------|-----------|
| 1 | `workspace/skills/paperclip/SKILL.md` | `python3 json.load()` на сирому рядку → 401 | ✅ Виправлено |
| 2 | `skills/paperclip/SKILL.md` (системний) | "Correct" приклад → `.json` замість `.full.json` | ✅ Виправлено |
| 3 | CEO агент (wake event) | Текстовий опис замість виконання задачі | ✅ Виправлено |
| 4 | Session persistence | Хибний патерн в conversation history | ✅ Очищено |
| 5 | curl в exec tool | Одинарні лапки для змінних | ✅ Виправлено |
| 6 | systemd | Немає автозапуску після ребуту | ✅ Виправлено |

## Поточний стан після виправлень

```
paperclip.service        active (running) — enabled
openclaw-gateway.service active (running) — enabled
ollama.service           active (running) — enabled
Paperclip API :3100      200 OK
Gateway ws://127.0.0.1:18789  reachable
Qdrant :6333             OK (v1.17.1)
```

Агент CEO тепер при wake event:
- Читає SKILL.md (перша дія)
- Одразу виконує задачу через exec (без текстових описів)
- Пише коментарі українською мовою
- Коректно оновлює статус задачі
