# Настройка OpenRouter для n8n Workflows

## Проблема
Ошибка: `The model "api/v1" is not available` возникает, когда workflow пытается использовать неправильный URL или формат модели.

## Решение

Workflows уже обновлены для работы с OpenRouter. Вам нужно только настроить credentials.

### Шаг 1: Получите API ключ OpenRouter

1. Зарегистрируйтесь на https://openrouter.ai/
2. Перейдите в раздел Keys: https://openrouter.ai/keys
3. Создайте новый API ключ
4. Скопируйте ключ (он начинается с `sk-or-...`)

### Шаг 2: Настройте Credentials в n8n

1. Откройте n8n: http://localhost:5678
2. Перейдите в **Credentials** (в меню слева)
3. Найдите или создайте credential с именем **"OpenRouter API"**
4. Выберите тип: **HTTP Header Auth**
5. Заполните:
   - **Header Name**: `Authorization`
   - **Header Value**: `Bearer YOUR_OPENROUTER_API_KEY` (замените на ваш ключ)
6. Сохраните

### Шаг 3: Проверьте настройки в Workflows

Workflows уже настроены на:
- **URL**: `https://openrouter.ai/api/v1/embeddings`
- **Модель**: `openai/text-embedding-3-large`
- **Заголовки**: 
  - `HTTP-Referer: http://localhost:5678`
  - `X-Title: n8n Pinecone Workflow`

### Шаг 4: Импортируйте обновленные workflows

Если вы уже импортировали workflows ранее:

1. Удалите старые workflows из n8n
2. Импортируйте обновленные файлы:
   - `workflow1-pinecone-upload.json`
   - `workflow2-pinecone-search.json`
3. Убедитесь, что в узлах "Get Embeddings" используется credential "OpenRouter API"

### Альтернативные модели

Если хотите использовать другую модель через OpenRouter, измените в JSON body узла "Get Embeddings":

```json
{
  "model": "openai/text-embedding-3-large",  // или другая модель
  "input": "...",
  "dimensions": 1024
}
```

Доступные модели для embeddings через OpenRouter:
- `openai/text-embedding-3-large` (1024 dimensions)
- `openai/text-embedding-3-small` (1536 dimensions)
- `openai/text-embedding-ada-002` (1536 dimensions)

**Важно:** Убедитесь, что размерность embeddings соответствует размерности вашего Pinecone индекса!

## Проверка работы

После настройки попробуйте запустить workflow. Если всё настроено правильно, вы не должны видеть ошибку `The model "api/v1" is not available`.

## Troubleshooting

### Ошибка: "Invalid API key"
- Проверьте, что в Header Value указан полный ключ: `Bearer sk-or-...`
- Убедитесь, что ключ активен на OpenRouter

### Ошибка: "Model not found"
- Проверьте формат модели: должно быть `openai/text-embedding-3-large`, а не просто `text-embedding-3-large`
- Убедитесь, что модель доступна на OpenRouter

### Ошибка: "HTTP-Referer required"
- Workflows уже настроены с заголовком HTTP-Referer
- Если ошибка сохраняется, проверьте, что заголовок добавлен в узле HTTP Request
