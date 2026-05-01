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
| vLLM | 0.8.3 | ✅ (не використовується — не підтримує Blackwell) | — |
| Ollama | 0.22.0 (`~/ollama/bin/ollama`) | ✅ | 11434 |
| Node.js | v22.22.2 | ✅ | — |
| Paperclip | 2026.428.0 | ✅ | 3101 |
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
   ↓
OpenClaw (порт 18789, WebSocket)
   ├── skill: legal-rag  →  RAG API (порт 8080)  →  Qdrant (порт 6333)
   └── skill: paperclip  →  Paperclip (порт 3101)
                ↓
        Ollama (порт 11434)
        ├── qwen2.5:32b        ← основна модель
        ├── qwen2.5-coder:32b
        └── nomic-embed-text   ← ембединги
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

## OpenClaw — Telegram-бот

### Поточний стан: ✅ Працює

| Параметр | Значення |
|----------|----------|
| Версія | 2026.4.29 |
| Telegram | ON / OK |
| Модель | qwen2.5:32b (33k context) |
| Gateway | ws://127.0.0.1:18789 |
| Telegram admin ID | 447256133 |

### Skills

| Skill | Статус | Призначення |
|-------|--------|-------------|
| `legal-rag` | ✅ Ready | Пошук по судових справах, юридичні відповіді |
| `paperclip` | ✅ Ready | Керування агентами через Paperclip |

### Виправлена проблема (1 травня 2026)

**Причина:** після оновлення OpenClaw 2026.4.29 зник npm-пакет `grammy` (Telegram-бібліотека).  
**Рішення:**
```bash
cd ~/.nvm/versions/node/v22.22.2/lib/node_modules/openclaw
npm install grammy
openclaw gateway stop && openclaw gateway start
```

---

## Paperclip

| Параметр | Значення |
|----------|----------|
| Версія | 2026.428.0 |
| Порт | 3101 (доступний тільки локально) |
| Компанія | Юридична практика |
| Агент | OpenClaw (openclaw_gateway) |

**SSH тунель з MacBook:**
```bash
ssh -L 3101:localhost:3101 leo@192.168.50.105 -N
# Відкрити у браузері: http://localhost:3101
```

**Відома проблема:** OpenClaw Gateway іноді показує `error` в Paperclip через несумісність schema (`unexpected property 'paperclip'`). Telegram-бот при цьому продовжує працювати нормально.

---

## Автозапуск сервісів

| Сервіс | Автозапуск | Команда перевірки |
|--------|-----------|-------------------|
| Qdrant | ✅ Docker | `curl http://localhost:6333/health` |
| RAG API | ✅ systemd (`legal-api.service`) | `systemctl --user status legal-api` |
| OpenClaw | ✅ systemd (`openclaw-gateway.service`) | `openclaw gateway status` |
| Ollama | ❌ вручну | `curl http://localhost:11434/api/tags` |
| Paperclip | ❌ вручну | `curl http://localhost:3101/api/health` |

### Запуск після перезавантаження

```bash
# 1. Ollama (обов'язково, без неї бот не відповідає)
OLLAMA_MODELS=~/models/ollama OLLAMA_HOST=0.0.0.0:11434 \
  nohup ~/ollama/bin/ollama serve > ~/ollama.log 2>&1 &

# 2. Paperclip (якщо потрібен UI)
npx paperclipai onboard --yes

# 3. RAG API та OpenClaw стартують автоматично
systemctl --user status legal-api
openclaw gateway status
```

---

## Claude Code на сервері

```bash
# Запуск
claude

# Статус лайн показує: модель | контекст | git-гілка | проект
# Налаштований у ~/.claude/settings.json
```

---

## Що залишилось зробити

- [ ] **Завантажити 500 судових справ** з OneDrive → `~/cases/`
  ```bash
  sudo apt install rclone
  rclone config   # налаштувати OneDrive
  rclone sync onedrive:Справи ~/cases/
  ```
- [ ] **Переіндексувати Qdrant** після завантаження справ
  ```bash
  source ~/venv312/bin/activate && python3 ~/rag/index_cases.py
  ```
- [ ] **Автозапуск Ollama** при перезавантаженні (systemd service)
- [ ] **Tesseract OCR** для сканованих PDF
  ```bash
  sudo apt install tesseract-ocr tesseract-ocr-ukr tesseract-ocr-rus poppler-utils
  ```
- [ ] **Виправити Paperclip ↔ OpenClaw** (schema несумісність при `paperclipApiUrl`)

---

## Перевірка всього стеку

```bash
# Один рядок — все одразу
echo "=Ollama=" && curl -s http://localhost:11434/api/tags | python3 -c "import sys,json; [print(' ✓', m['name']) for m in json.load(sys.stdin)['models']]" && \
echo "=Qdrant=" && curl -s http://localhost:6333/collections/legal_cases | python3 -c "import sys,json; print(' ✓ vectors:', json.load(sys.stdin)['result']['points_count'])" && \
echo "=RAG API=" && curl -s http://localhost:8080/health && \
echo "" && echo "=OpenClaw=" && openclaw gateway status 2>/dev/null | grep "Runtime\|Telegram"
```

---

*Оновлено: 1 травня 2026*  
*Сервер: `leo@192.168.50.105`*
