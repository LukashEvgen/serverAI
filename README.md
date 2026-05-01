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
| `qwen2.5:32b` | 19 GB | Юридичний аналіз, RAG | ~31 tok/sec |
| `qwen2.5-coder:32b` | 19 GB | Код, скрипти | ~29 tok/sec |
| `nomic-embed-text` | 274 MB | Ембединги для RAG | — |
| `qwen2.5-72b` | 47 GB | Резерв | — |
| `deepseek-v4-pro:cloud` | cloud | Складне юридичне reasoning | — |
| `qwen3-coder:480b-cloud` | cloud | Складна архітектура/код | — |
| `deepseek-v3.1:671b-cloud` | cloud | Резерв | — |
| `gpt-oss:120b-cloud` | cloud | Резерв | — |

> **Чому Ollama замість vLLM:** GPU RTX PRO 4000 Blackwell (sm_120) не підтримується PyTorch 2.6. Ollama підтримує Blackwell нативно.

---

## Архітектура системи

```
Telegram
   ├── @claude_luk_bot  →  OpenClaw agent: main  →  qwen2.5:32b     (юридичний)
   └── @Codex_lu_bot    →  OpenClaw agent: coder →  qwen2.5-coder:32b (код)

OpenClaw Gateway (порт 18789, WebSocket)
   ├── skill: legal-rag  →  RAG API (порт 8080)  →  Qdrant (порт 6333)
   │                        └── fallback: web_search (DuckDuckGo)
   └── skill: paperclip  →  Paperclip (порт 3100)
                ↓
        Ollama (порт 11434)
        ├── qwen2.5:32b           ← юридична модель (default)
        ├── qwen2.5-coder:32b     ← модель для коду
        ├── deepseek-v4-pro:cloud ← складне reasoning (cloud)
        ├── qwen3-coder:480b-cloud← складна архітектура (cloud)
        └── nomic-embed-text      ← ембединги для RAG
```

---

## Routing моделей

| Тип задачі | Агент | Модель |
|------------|-------|--------|
| Юридичний аналіз, RAG, документи | main (@claude_luk_bot) | `qwen2.5:32b` |
| Складне юридичне reasoning | main | `deepseek-v4-pro:cloud` (явна вказівка) |
| Пошук в інтернеті | main | `web_search` → DuckDuckGo (auto через legal-rag) |
| Код, скрипти, дебаг | coder (@Codex_lu_bot) | `qwen2.5-coder:32b` |
| Складна архітектура | coder | `qwen3-coder:480b-cloud` (явна вказівка) |

**Веб-пошук:** DuckDuckGo — безкоштовно, без API-ключа. Автоматично запускається через legal-rag skill коли локальна база порожня або питання про актуальне законодавство.

---

## RAG Pipeline (пошук по судових справах)

### Файли

| Файл | Призначення |
|------|-------------|
| `~/rag/index_cases.py` | Індексація документів (PDF/DOCX/TXT) → Qdrant |
| `~/rag/ask.py` | CLI-запит до бази справ |
| `~/rag/api.py` | FastAPI HTTP сервер (порт 8080) |

### Qdrant колекція

- **Назва:** `legal_cases`
- **Вектори:** 768 вимірів (nomic-embed-text)
- **Документи:** `~/cases/`

### Команди

```bash
# Перевірка стану API
curl http://localhost:8080/health

# Запитати базу справ
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Які шанси виграти справу про стягнення боргу?"}'

# Індексація нових справ
source ~/venv312/bin/activate
python3 ~/rag/index_cases.py
```

---

## OpenClaw — Telegram-боти

| Параметр | Значення |
|----------|----------|
| Версія | 2026.4.29 |
| Gateway | ws://127.0.0.1:18789 |
| Telegram admin ID | 447256133 |

### Агенти

| Агент | Telegram-бот | Модель | Workspace |
|-------|-------------|--------|-----------|
| `main` | `@claude_luk_bot` | `qwen2.5:32b` | `~/.openclaw/workspace/` |
| `coder` | `@Codex_lu_bot` | `qwen2.5-coder:32b` | `~/.openclaw/workspace-coder/` |
| `browser` | — | `deepseek-v4-pro:cloud` | `~/.openclaw/workspace-browser/` |

### Skills (агент main)

| Skill | Статус | Призначення |
|-------|--------|-------------|
| `legal-rag` | ✅ | Пошук по справах + fallback до web_search |
| `paperclip` | ✅ | Керування агентами |

### Виправлені проблеми

**Проблема (30 квітня):** після оновлення OpenClaw зник npm-пакет `grammy`.
```bash
cd ~/.nvm/versions/node/v22.22.2/lib/node_modules/openclaw
npm install grammy
openclaw gateway stop && openclaw gateway start
```

**Проблема (1 травня):** `invalid agent params: unexpected property 'paperclip'` — schema validation.
```bash
~/fix-openclaw-schema.sh
openclaw gateway stop && openclaw gateway start
```

**Проблема (1 травня):** веб-пошук не працював — `provider: "ollama"`.
**Рішення:** змінено на `provider: "duckduckgo"` в `~/.openclaw/openclaw.json`, увімкнено DuckDuckGo plugin.

---

## Paperclip

| Параметр | Значення |
|----------|----------|
| Версія | 2026.428.0 |
| Порт | 3100 |
| Агент OpenClaw | ✅ підключений |

### Підключення з MacBook

```bash
# Термінал 1 — SSH тунель (тримати відкритим)
ssh -L 3100:127.0.0.1:3100 leo@192.168.50.105 -N

# Браузер
http://localhost:3100
```

---

## Автозапуск сервісів

| Сервіс | Systemd unit | Перевірка |
|--------|-------------|-----------|
| Qdrant | Docker (`restart: always`) | `curl http://localhost:6333/health` |
| Ollama | `ollama.service` (user) | `systemctl --user status ollama` |
| RAG API | `legal-api.service` (user) | `systemctl --user status legal-api` |
| OpenClaw | `openclaw-gateway.service` (user) | `openclaw gateway status` |
| Paperclip | `paperclip.service` (user) | `curl http://localhost:3100/api/health` |

> Порядок залежностей: Qdrant → Ollama → RAG API + OpenClaw → Paperclip

```bash
# Перевірка всіх після перезавантаження
systemctl --user status ollama paperclip legal-api openclaw-gateway
```

### Після оновлення OpenClaw

```bash
~/fix-openclaw-schema.sh && openclaw gateway stop && openclaw gateway start
```

---

## Claude Code на сервері

```bash
# Запуск
claude
```

---

## Швидка перевірка всього стеку

```bash
echo "=Ollama=" && curl -s http://localhost:11434/api/tags | \
  python3 -c "import sys,json; [print(' ✓', m['name']) for m in json.load(sys.stdin)['models']]"
echo "=Qdrant=" && curl -s http://localhost:6333/collections/legal_cases | \
  python3 -c "import sys,json; print(' ✓ vectors:', json.load(sys.stdin)['result']['points_count'])"
echo "=RAG API=" && curl -s http://localhost:8080/health
echo "=Web search=" && openclaw capability web providers 2>/dev/null | grep '"selected": true' | head -1
echo "=OpenClaw=" && openclaw gateway status 2>/dev/null | grep "Runtime\|Telegram"
echo "=Paperclip=" && curl -s http://localhost:3100/api/health
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
- [ ] **Tesseract OCR** для сканованих PDF
  ```bash
  sudo apt install tesseract-ocr tesseract-ocr-ukr tesseract-ocr-rus poppler-utils
  ```

---

*Оновлено: 1 травня 2026*
*Сервер: `leo@192.168.50.105`*
