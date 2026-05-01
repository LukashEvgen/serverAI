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

> Ubuntu 25.04 обрано через несумісність 24.04 з Realtek 8126 ethernet.

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
| vLLM | 0.8.3 | ✅ | — |
| Ollama | 0.22.0 (`~/ollama/bin/ollama`) | ✅ | 11434 |
| Node.js | v22.22.2 | ✅ | — |
| Paperclip | 2026.428.0 | ✅ | 3101 |
| OpenClaw | 2026.4.29 | ✅ | 18789 (WS) |
| RAG API | FastAPI (legal-api.service) | ✅ | 8080 |

---

## Моделі (Ollama)

| Модель | Розмір | Призначення | Швидкість |
|--------|--------|-------------|-----------|
| `qwen2.5:32b` | 19.9 GB | Юридичний аналіз, RAG | ~31 tok/sec |
| `qwen2.5-coder:32b` | 19.9 GB | Код, скрипти | ~29 tok/sec |
| `nomic-embed-text` | 274 MB | Ембединги для векторного пошуку | — |
| `qwen2.5-72b` | 47.4 GB | Резерв (не поміщається в VRAM повністю) | — |

> GPU: 2x RTX PRO 4000 (sm_120, Blackwell). PyTorch 2.6 не підтримує sm_120 нативно — тому використовується Ollama (підтримує Blackwell нативно), а не vLLM.

---

## Архітектура системи

```
Telegram
   ↓
OpenClaw (порт 18789, WS)
   ├── skill: legal-rag  →  RAG API (порт 8080)  →  Qdrant (порт 6333)
   └── skill: paperclip  →  Paperclip UI (порт 3101)
                                    ↓
                          Ollama (порт 11434)
                          ├── qwen2.5:32b
                          ├── qwen2.5-coder:32b
                          └── nomic-embed-text
```

---

## RAG Pipeline

### Файли

| Файл | Призначення |
|------|-------------|
| `~/rag/index_cases.py` | Індексація документів (PDF/DOCX/TXT) → Qdrant |
| `~/rag/ask.py` | Прямий CLI-запит до бази справ |
| `~/rag/api.py` | FastAPI HTTP сервер на порту 8080 |
| `~/rag/start_api.sh` | Скрипт запуску API |

### Qdrant колекція

- **Назва:** `legal_cases`
- **Вектори:** 768 вимірів (nomic-embed-text)
- **Зараз:** 1 тестовий документ (500 реальних справ ще не завантажено)

### Команди

```bash
# Запуск Ollama
OLLAMA_MODELS=~/models/ollama OLLAMA_HOST=0.0.0.0:11434 ~/ollama/bin/ollama serve

# Перевірка API
curl http://localhost:8080/health

# Запитати базу справ
curl -X POST http://localhost:8080/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Які шанси виграти справу про стягнення боргу?"}'

# Індексація нових справ (покласти файли в ~/cases/)
source ~/venv312/bin/activate
python3 ~/rag/index_cases.py
```

---

## OpenClaw — Telegram-бот

### Налаштування

- **Telegram user ID адміністратора:** 447256133
- **OpenClaw Gateway URL:** `ws://127.0.0.1:18789/`
- **LLM:** `qwen2.5:32b` через Ollama (OpenAI-сумісний API на порту 11434)

### Skills

| Skill | Статус | Призначення |
|-------|--------|-------------|
| `legal-rag` | ✅ Ready | Пошук по судових справах, юридичні відповіді |
| `paperclip` | ✅ Ready | Керування агентами через Paperclip |

### Конфігурація файлів

- `~/.openclaw/workspace/TOOLS.md` — інструкції для агента (RAG API, Ollama)
- `~/.openclaw/workspace/skills/legal-rag/SKILL.md` — юридичний skill

---

## Paperclip

- **URL (локально через SSH тунель):** `http://localhost:3101`
- **SSH тунель з MacBook:** `ssh -L 3101:localhost:3101 leo@192.168.50.105 -N`
- **Компанія:** Юридична практика
- **Агент OpenClaw:** підключено через `openclaw_gateway`
- **API ключ:** збережено в `~/.openclaw/workspace/paperclip-claimed-api-key.json`

### Відома проблема

OpenClaw Gateway показує статус `error` з Paperclip через несумісність:
```
INVALID_REQUEST: invalid agent params: at root: unexpected property 'paperclip'
```
Paperclip 2026.428.0 надсилає поле `paperclipApiUrl` у WebSocket-запиті, яке OpenClaw 2026.4.29 відхиляє через жорстку schema validation. Потребує оновлення одного з компонентів або патча schema.

---

## Автозапуск сервісів

### legal-api (systemd user service)

```bash
# Статус
systemctl --user status legal-api

# Перезапуск
systemctl --user restart legal-api

# Логи
journalctl --user -u legal-api -f
```

### Ollama (запускається вручну або потрібен systemd)

```bash
OLLAMA_MODELS=~/models/ollama OLLAMA_HOST=0.0.0.0:11434 \
  nohup ~/ollama/bin/ollama serve > ~/ollama.log 2>&1 &
```

### Перевірка всіх сервісів

```bash
# Ollama
curl -s http://localhost:11434/api/tags

# RAG API
curl -s http://localhost:8080/health

# Qdrant
curl -s http://localhost:6333/health

# Paperclip
curl -s http://localhost:3101/api/health

# OpenClaw
openclaw gateway status
```

---

## Що залишилось зробити

- [ ] **Завантажити 500 судових справ** з OneDrive на сервер (`~/cases/`)
  - Встановити `rclone`: `sudo apt install rclone`
  - Налаштувати OneDrive: `rclone config`
  - Синхронізувати: `rclone sync onedrive:Справи ~/cases/`
- [ ] **Переіндексувати базу** після завантаження справ: `python3 ~/rag/index_cases.py`
- [ ] **Виправити OpenClaw ↔ Paperclip** (несумісність schema)
- [ ] **Автозапуск Ollama** при перезавантаженні (systemd service)
- [ ] **Встановити Tesseract** для OCR сканованих PDF: `sudo apt install tesseract-ocr tesseract-ocr-ukr`

---

## Команди для швидкого старту після перезавантаження

```bash
# 1. Запустити Ollama
OLLAMA_MODELS=~/models/ollama OLLAMA_HOST=0.0.0.0:11434 \
  nohup ~/ollama/bin/ollama serve > ~/ollama.log 2>&1 &

# 2. RAG API запускається автоматично (systemd)
systemctl --user status legal-api

# 3. Запустити Paperclip (якщо не запущений)
npx paperclipai onboard --yes

# 4. OpenClaw Gateway запускається автоматично (systemd)
openclaw gateway status
```

---

*Дата: 30 квітня — 1 травня 2026*  
*Сервер: leo@192.168.50.105*
