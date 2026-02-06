# Pinecone + n8n Workflows

> Два workflow для работы с Pinecone векторной базой через n8n: загрузка текстов и семантический поиск (RAG).

---

## Содержание

- [Быстрый старт](#-быстрый-старт)
- [Структура проекта](#-структура-проекта)
- [Настройка](#-настройка)
- [Использование](#-использование)
- [Параметры и настройки](#-параметры-и-настройки)
- [Структура workflows](#-структура-workflows)
- [Примеры кода](#-примеры-кода)
- [Решение проблем](#-решение-проблем)
- [Лицензия](#-лицензия)

---

## Быстрый старт

```bash
# 1. Запустить n8n
docker-compose up -d

# 2. Открыть в браузере
# http://localhost:5678

# 3. Импортировать workflow1-pinecone-upload.json и workflow2-pinecone-search.json
# 4. Настроить Credentials (OpenRouter + Pinecone)
# 5. Включить workflows (кнопка Active)
```

После настройки: загрузка через webhook `/webhook/pinecone-upload`, поиск — `/webhook/pinecone-search`.

---

## Структура проекта

| Файл | Описание |
|------|----------|
| `workflow1-pinecone-upload.json` | Workflow загрузки текстов в Pinecone |
| `workflow2-pinecone-search.json` | Workflow семантического поиска по Pinecone |
| `upload_files_to_pinecone.py` | Python-скрипт загрузки файлов из папки `/data` |
| `test_search_api.py` | Клиент для тестирования поиска через API |
| `docker-compose.yml` | Запуск n8n в Docker |
| `OPENROUTER_SETUP.md` | Подробная настройка OpenRouter |

---

## Настройка

### 1. Запуск n8n

```bash
docker-compose up -d
```

Интерфейс: **http://localhost:5678**

---

### 2. Импорт workflows

1. Откройте n8n: http://localhost:5678  
2. **Workflows** → **Import from File**  
3. Импортируйте `workflow1-pinecone-upload.json`  
4. Импортируйте `workflow2-pinecone-search.json`

---

### 3. Credentials

#### OpenRouter (embeddings)

| Параметр | Значение |
|----------|----------|
| Тип | HTTP Header Auth |
| Header Name | `Authorization` |
| Header Value | `Bearer YOUR_OPENROUTER_API_KEY` |

- Используется в узлах: **Get Embeddings**, **Get Query Embedding**
- Ключ: https://openrouter.ai/keys  
- Модель в workflow: `openai/text-embedding-3-large`

#### Pinecone

| Параметр | Значение |
|----------|----------|
| Тип | HTTP Header Auth |
| Header Name | `Api-Key` |
| Header Value | `YOUR_PINECONE_API_KEY` |

- Используется в узлах: **Upsert to Pinecone**, **Query Pinecone**

---

### 4. Переменные окружения

В `docker-compose.yml` или в n8n (**Settings** → **Environment Variables**):

```yaml
environment:
  - PINECONE_ENDPOINT=https://your-index.svc.region.pinecone.io
```

---

### 5. Активация workflows

В каждом workflow нажмите **Active** в правом верхнем углу.

---

## Использование

### Workflow 1: Загрузка данных

#### Вариант A: Python-скрипт

1. Положите `.txt` файлы в папку `data/`
2. Выполните:

```bash
python upload_files_to_pinecone.py
```

С переменными окружения:

```bash
export DATA_DIR="/path/to/your/data"
export N8N_UPLOAD_WEBHOOK_URL="http://localhost:5678/webhook/pinecone-upload"
export PINECONE_NAMESPACE="my-namespace"
python upload_files_to_pinecone.py
```

#### Вариант B: API (curl)

```bash
curl -X POST http://localhost:5678/webhook/pinecone-upload \
  -H "Content-Type: application/json" \
  -d '{
    "files": [
      {"fileName": "document1.txt", "text": "Содержимое первого документа..."},
      {"fileName": "document2.txt", "text": "Содержимое второго документа..."}
    ],
    "namespace": "default"
  }'
```

**Тело запроса:**

```json
{
  "files": [
    { "fileName": "document1.txt", "text": "Текст документа..." }
  ],
  "namespace": "default"
}
```

**Пример ответа:**

```json
{
  "status": "success",
  "totalFiles": 2,
  "totalUpserted": 150,
  "fileResults": [
    { "fileIndex": 1, "upserted": 75 },
    { "fileIndex": 2, "upserted": 75 }
  ]
}
```

---

### Workflow 2: Поиск

#### Python-клиент

```bash
python test_search_api.py
```

#### API (curl)

```bash
curl -X POST http://localhost:5678/webhook/pinecone-search \
  -H "Content-Type: application/json" \
  -d '{"query": "ваш поисковый запрос", "namespace": "default"}'
```

**Тело запроса:**

```json
{
  "query": "текст запроса",
  "namespace": "default"
}
```

**Пример ответа:**

```json
[
  {
    "text": "найденный текст чанка",
    "score": 0.95,
    "source": "document1",
    "chunk_index": 0,
    "id": "chunk_document1_1234567890_0"
  }
]
```

---

## Параметры и настройки

### Чанки (узёл «Chunk Text»)

| Параметр | По умолчанию | Описание |
|----------|--------------|----------|
| `chunkSize` | 600 | Размер чанка (символов) |
| `overlap` | 100 | Перекрытие между чанками |

### Embeddings (узлы «Get Embeddings»)

| Параметр | Значение |
|----------|----------|
| Модель | `openai/text-embedding-3-large` (OpenRouter) |
| Размерность | `3072` (должна совпадать с Pinecone-индексом) |
| Endpoint | `https://openrouter.ai/api/v1/embeddings` |

> **Важно:** Размерность в workflow должна совпадать с размерностью индекса в Pinecone.

### Поиск (узёл «Query Pinecone»)

| Параметр | По умолчанию |
|----------|--------------|
| `topK` | 5 (число возвращаемых результатов) |

---

## Структура workflows

**Workflow 1 (Upload):**

```
Webhook → Parse Input → Chunk Text → Prepare Batch →
Get Embeddings → Format for Pinecone → Upsert to Pinecone →
Aggregate Results → Respond to Webhook
```

**Workflow 2 (Search):**

```
Webhook → Process Query → Get Query Embedding →
Query Pinecone → Format Results → Respond to Webhook
```

---

## Примеры кода

### Загрузка документации проекта

```python
import os
import requests

files_data = []
for filename in os.listdir('./docs'):
    if filename.endswith('.txt'):
        with open(f'./docs/{filename}', 'r') as f:
            files_data.append({'fileName': filename, 'text': f.read()})

response = requests.post(
    'http://localhost:5678/webhook/pinecone-upload',
    json={'files': files_data, 'namespace': 'documentation'}
)
print(response.json())
```

### Поиск по документации

```python
import requests

response = requests.post(
    'http://localhost:5678/webhook/pinecone-search',
    json={'query': 'How to install?', 'namespace': 'documentation'}
)

for result in response.json():
    print(f"Score: {result['score']:.4f}")
    print(f"Source: {result['source']}")
    print(f"Text: {result['text'][:200]}...")
    print('-' * 80)
```

---

## Решение проблем

| Ошибка | Решение |
|--------|--------|
| **Cannot find module 'fs'** | Используйте webhook-вариант (текущая версия), не executeCommand. |
| **Blocks are not connected** | Закройте workflow → удалите старый → импортируйте заново → обновите страницу (F5). |
| **Docker pull висит** | `docker-compose down` → `docker pull n8nio/n8n:latest` → `docker-compose up -d` |
| **Vector dimension does not match the index** | Размерность embeddings не совпадает с индексом Pinecone. |

### Vector dimension does not match

1. Проверьте размерность индекса в консоли Pinecone.  
2. В узле **Get Embeddings** (в обоих workflow) задайте тот же `dimensions`.  
3. Для `text-embedding-3-large`: допустимы **256**, **1024**, **3072**.  
4. В workflow загрузки и поиска должна быть **одна и та же** размерность.

---

## Лицензия

MIT
