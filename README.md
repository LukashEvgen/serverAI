# AI-сервер для юридичної практики — Звіт про налаштування

## Залізо

| Компонент | Деталі |
|-----------|--------|
| CPU | AMD Ryzen 9000x |
| GPU | 2x NVIDIA RTX PRO 4000 Blackwell (GB203GL, 24 GB VRAM кожна = **48 GB** сумарно) |
| RAM | ~59 GB |
| SSD | 1.6 TB |
| IP | 192.168.50.105 |
| ОС | Ubuntu 25.04 (plucky) |

> Ubuntu 25.04 обрано через несумісність 24.04 з мережевою картою Realtek 8126.

---

## Що встановлено

| Компонент | Версія | Статус | Порт |
|-----------|--------|--------|------|
| Ubuntu 25.04 | 6.14.0-37-generic | ✅ | — |
| NVIDIA Driver | 575.57.08 | ✅ | — |
| CUDA | 12.9 (nvcc 12.8) | ✅ | — |
| Docker | 29.2.1 | ✅ | — |
| Qdrant | 1.17.1 (Docker) | ✅ | 6333 |
| Python | 3.12.9 (скомпільовано з джерел) | ✅ | — |
| venv312 | `~/venv312` | ✅ | — |
| PyTorch | 2.6.0+cu124 | ✅ | — |
| vLLM | 0.8.3 | ⚠️ не використовується (не підтримує Blackwell sm_120) | — |
| Ollama | 0.22.0 (`~/ollama/bin/ollama`) | ✅ | 11434 |
| Node.js | v22.22.2 | ✅ | — |
| Paperclip | 2026.428.0 | ✅ | 3100 |
| OpenClaw | 2026.4.29 | ✅ | 18789 (WS) |
| RAG API | FastAPI (`legal-api.service`) | ✅ | 8080 |
| Claude Code | v2.1.123 | ✅ | — |

---

## Моделі Ollama

| Модель | Розмір | Призначення | Швидкість |
|--------|--------|-------------|-----------|
| `qwen2.5:32b` | 19.9 GB | Юридичний аналіз, RAG, Telegram-бот | ~31 tok/sec |
| `qwen2.5-coder:32b` | 19.9 GB | Код, скрипти | ~29 tok/sec |
| `nomic-embed-text` | 274 MB | Ембединги для векторного пошуку | — |
| `qwen2.5-72b` | 47.4 GB | Резерв (не поміщається повністю в 48 GB VRAM) | — |

> **Чому Ollama замість vLLM:** GPU RTX PRO 4000 Blackwell (sm_120) не підтримується PyTorch 2.6. Ollama підтримує Blackwell нативно.

---

## Архітектура системи

```
Telegram
   ├── @claude_luk_bot  →  OpenClaw agent: main  →  qwen2.5:32b     (юридичний аналіз)
   └── @Codex_lu_bot    →  OpenClaw agent: coder →  qwen2.5-coder:32b (код і скрипти)

OpenClaw Gateway (порт 18789, WebSocket)
   ├── skill: legal-rag  →  RAG API (порт 8080)  →  Qdrant (порт 6333)
   └── skill: paperclip  →  Paperclip (порт 3100)
                ↓
        Ollama (порт 11434)
        ├── qwen2.5:32b        ← юридична модель
        ├── qwen2.5-coder:32b  ← модель для коду
        └── nomic-embed-text   ← ембединги для RAG
```

---

## RAG Pipeline (пошук по судових справах)

### Файли

| Файл | Призначення |
|------|-------------|
| `~/rag/index_cases.py` | Індексація документів (PDF/DOCX/TXT) → Qdrant |
| `~/rag/ask.py` | CLI-запит до бази справ |
| `~/rag/api.py` | FastAPI HTTP сервер (порт 8080) |
| `~/rag/start_api.sh` | Скрипт запуску API |

### Qdrant колекція

- **Назва:** `legal_cases`
- **Вектори:** 768 вимірів (nomic-embed-text)
- **Поточний стан:** 1 тестовий документ (500 реальних справ ще не завантажено)
- **Документи:** `~/cases/`

### Команди

```bash
# Перевірка стану API
curl http://localhost:8080/health

# Запитати базу справ
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Які шанси виграти справу про стягнення боргу?"}'

# Індексація нових справ (покласти файли в ~/cases/ і запустити)
source ~/venv312/bin/activate
python3 ~/rag/index_cases.py
```

---

## OpenClaw — Telegram-боти

### Поточний стан: ✅ Два боти активні

| Параметр | Значення |
|----------|----------|
| Версія | 2026.4.29 |
| Gateway | ws://127.0.0.1:18789 |
| Telegram admin ID | 447256133 |

### Агенти

| Агент | Telegram-бот | Модель | Призначення |
|-------|-------------|--------|-------------|
| `main` (default) | `@claude_luk_bot` | `qwen2.5:32b` | Юридичний аналіз, RAG, загальні питання |
| `coder` | `@Codex_lu_bot` | `qwen2.5-coder:32b` | Код, скрипти, автоматизація |

### Skills (агент main)

| Skill | Статус | Призначення |
|-------|--------|-------------|
| `legal-rag` | ✅ Ready | Пошук по судових справах, юридичні відповіді |
| `paperclip` | ✅ Ready | Керування агентами через Paperclip |

### Workspaces

| Агент | Workspace |
|-------|-----------|
| main | `~/.openclaw/workspace/` |
| coder | `~/.openclaw/workspace-coder/` |

### Виправлені проблеми

**Проблема 1 (30 квітня):** після оновлення OpenClaw зник npm-пакет `grammy`.
```bash
cd ~/.nvm/versions/node/v22.22.2/lib/node_modules/openclaw
npm install grammy
openclaw gateway stop && openclaw gateway start
```

**Проблема 2 (1 травня):** `invalid agent params: unexpected property 'paperclip'` — OpenClaw відхиляв з'єднання від Paperclip через жорстку schema validation.  
**Рішення:** патч schema + скрипт для повторного застосування після оновлень.
```bash
~/fix-openclaw-schema.sh
openclaw gateway stop && openclaw gateway start
```

---

## Paperclip

### Поточний стан: ✅ Працює, агент підключений

| Параметр | Значення |
|----------|----------|
| Версія | 2026.428.0 |
| Порт | 3100 |
| Компанія | Юридична практика |
| Агент OpenClaw | ✅ idle (підключений та працює) |

### Підключення з MacBook

```bash
# Термінал 1 — SSH тунель (тримати відкритим)
ssh -L 3100:127.0.0.1:3100 leo@192.168.50.105 -N

# Браузер
http://localhost:3100
```

---

## Автозапуск сервісів

| Сервіс | Автозапуск | Команда перевірки |
|--------|-----------|-------------------|
| Qdrant | ✅ Docker (`restart: always`) | `curl http://localhost:6333/health` |
| Ollama | ✅ systemd (`ollama.service`) | `systemctl --user status ollama` |
| RAG API | ✅ systemd (`legal-api.service`) | `systemctl --user status legal-api` |
| OpenClaw | ✅ systemd (`openclaw-gateway.service`) | `openclaw gateway status` |
| Paperclip | ✅ systemd (`paperclip.service`) | `curl http://localhost:3100/api/health` |

> Всі сервіси запускаються автоматично після перезавантаження.  
> Порядок залежностей: Ollama → RAG API + OpenClaw → Paperclip

### Перевірка після перезавантаження

```bash
systemctl --user status ollama paperclip legal-api openclaw-gateway
```

### Після оновлення OpenClaw — перевірити патч schema

```bash
~/fix-openclaw-schema.sh && openclaw gateway stop && openclaw gateway start
```

---

## Claude Code на сервері

```bash
# Запуск
claude

# Статус лайн (налаштований у ~/.claude/settings.json):
# claude-sonnet-4-6  [====        ]  34%  |  45k/200k  |  no-git  |  leo
```

---

## Що залишилось зробити

- [ ] **Завантажити 500 судових справ** з OneDrive → `~/cases/`
  ```bash
  sudo apt install rclone
  rclone config        # налаштувати OneDrive
  rclone sync onedrive:Справи ~/cases/
  ```
- [ ] **Переіндексувати Qdrant** після завантаження справ
  ```bash
  source ~/venv312/bin/activate && python3 ~/rag/index_cases.py
  ```
- [x] **Автозапуск Ollama** при перезавантаженні (systemd `ollama.service`)
- [x] **Автозапуск Paperclip** при перезавантаженні (systemd `paperclip.service`)
- [ ] **Tesseract OCR** для сканованих PDF
  ```bash
  sudo apt install tesseract-ocr tesseract-ocr-ukr tesseract-ocr-rus poppler-utils
  ```

---

## Швидка перевірка всього стеку

```bash
echo "=Ollama=" && curl -s http://localhost:11434/api/tags | \
  python3 -c "import sys,json; [print(' ✓', m['name']) for m in json.load(sys.stdin)['models']]"
echo "=Qdrant=" && curl -s http://localhost:6333/collections/legal_cases | \
  python3 -c "import sys,json; print(' ✓ vectors:', json.load(sys.stdin)['result']['points_count'])"
echo "=RAG API=" && curl -s http://localhost:8080/health
echo ""
echo "=OpenClaw=" && openclaw gateway status 2>/dev/null | grep "Runtime\|Telegram"
echo "=Paperclip=" && curl -s http://localhost:3100/api/health
```

---

*Оновлено: 1 травня 2026 (ніч)*  
*Сервер: `leo@192.168.50.105`*
