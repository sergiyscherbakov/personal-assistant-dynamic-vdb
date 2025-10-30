# Personal Assistant with Dynamic VDB - Документація

## Автор
**Розробник:** Сергій Щербаков
**Email:** sergiyscherbakov@ukr.net
**Telegram:** @s_help_2010

### 💰 Підтримати розробку
Задонатити на каву USDT (BINANCE SMART CHAIN):
**`0xDFD0A23d2FEd7c1ab8A0F9A4a1F8386832B6f95A`**

---

## 1. Опис

**Personal Assistant with Dynamic VDB** - це інтелектуальний персональний асистент на базі n8n, який працює через Telegram бот. Він використовує OpenAI GPT-4.1-mini для обробки запитів, підтримує голосові повідомлення (транскрибування через ElevenLabs), зберігає контекст розмов у Redis та використовує Zep для довготривалої пам'яті з динамічною векторною базою даних (Knowledge Graph).

### Основні можливості:
- Прийом текстових та голосових повідомлень через Telegram
- Автоматична транскрибація голосових повідомлень
- Управління користувачами та потоками (threads) в Zep
- Довготривала пам'ять через Redis
- Динамічна векторна база знань (Knowledge Graph) через Zep
- Автоматичне додавання важливої інформації в базу знань
- Пошук інформації з загальної бази знань
- Імпорт документів (PDF) в базу знань

---

## 2. Схема Workflow

```
ОСНОВНИЙ ФЛОВ:
┌─────────────────┐
│ Telegram        │
│ Trigger         │
└────────┬────────┘
         │
         v
┌─────────────────┐
│ Switch          │ (Розподіл: audio/text)
└────┬────────┬───┘
     │        │
     v        v
   audio    text
     │        │
┌────v───┐    │
│Get File│    │
└────┬───┘    │
     │        │
┌────v────┐   │
│Transcribe│  │
└────┬────┘   │
     │        │
┌────v────┐┌──v──────┐
│Edit     ││Edit     │
│Fields   ││Fields1  │
└────┬────┘└──┬──────┘
     │        │
     └────┬───┘
          v
    ┌─────────────┐
    │ setupPrompt │
    └──────┬──────┘
           │
           v
    ┌─────────────┐
    │ Check_User  │
    └──────┬──────┘
           │
           v
    ┌─────────────┐
    │  Switch1    │ (Користувач існує?)
    └─┬─────────┬─┘
      │         │
   no │         │ yes
      v         v
┌─────────┐  ┌──────────────────┐
│add_user │  │get_message_from_ │
└────┬────┘  │thread            │
     │       └────────┬─────────┘
     v                │
┌──────────────┐      │
│create_thread │      │
└──────┬───────┘      │
       │              v
       │       ┌─────────────────┐
       │       │get_message_from_│
       │       │graph            │
       │       └────────┬────────┘
       └────────────┬───┘
                    v
              ┌──────────┐
              │ AI Agent │◄───┐
              │          │    │ (OpenAI + Redis + Tools)
              └────┬─────┘    │
                   │          │
                   v          │
         ┌──────────────────┐ │
         │add_message_to_   │ │
         │grpah             │ │
         └────┬─────────────┘ │
              │               │
              v               │
       ┌─────────────────┐    │
       │Send a text      │    │
       │message          │    │
       └─────────────────┘    │
                              │
TOOLS для AI Agent: ──────────┘
  - get_data_from_db
  - add_info_to_db


СТВОРЕННЯ ГРАФУ:
┌──────────────┐
│Create Graph  │
│DB            │
└──────────────┘


ДОДАВАННЯ ІНФОРМАЦІЇ:
┌────────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│Get a file1 │ --> │Extract   │ --> │Chunking  │ --> │HTTP      │
│(PDF)       │     │from File │     │          │     │Request   │
└────────────┘     └──────────┘     └──────────┘     └──────────┘
```

---

## 3. Покрокова інструкція по нодах

### Нода 1: Telegram Trigger
**Тип:** `n8n-nodes-base.telegramTrigger`
**Що робить:** Отримує вхідні повідомлення з Telegram бота (текстові та голосові/аудіо)

**Параметри:**
- Updates: message (отримує оновлення типу "message")
- Download: true (автоматично завантажує файли)

**Вхідні дані:**
```json
{
  "message": {
    "message_id": 12345,
    "from": {
      "id": 987654321,
      "first_name": "John",
      "last_name": "Doe"
    },
    "chat": {
      "id": 987654321
    },
    "text": "Привіт, асистенте!",
    "voice": {
      "file_id": "AwADBAADXXXXXXX",
      "mime_type": "audio/ogg"
    }
  }
}
```

**Процес обробки:**
1. Отримує webhook від Telegram API
2. Парсить повідомлення користувача
3. Якщо є файли (голосові) - завантажує їх автоматично

**Вихідні дані:**
Передає повний об'єкт message з усіма полями в наступну ноду (Switch)

**Credentials:**
- Тип: Telegram API
- ID: ZWi14sL1REwFLLc8
- Назва: Personal Assistant
- Як отримати:
  1. Створити бота через @BotFather в Telegram
  2. Отримати токен (формат: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`)
  3. Додати credentials в n8n: Credentials → New → Telegram API → вставити токен

---

### Нода 2: Switch
**Тип:** `n8n-nodes-base.switch`
**Що робить:** Розподіляє вхідні повідомлення на два типи: аудіо (голосові) та текстові

**Вхідні дані:**
```json
{
  "message": {
    "text": "текст повідомлення",
    "voice": {
      "mime_type": "audio/ogg"
    },
    "audio": {
      "mime_type": "audio/mpeg"
    }
  }
}
```

**Процес обробки:**
1. **Правило 1 (audio):** Перевіряє чи містить `$json.message.voice.mime_type` або `$json.message.audio.mime_type` текст "audio"
   - Якщо так - направляє в output "audio"
2. **Правило 2 (text):** Перевіряє чи існує `$json.message.text`
   - Якщо так - направляє в output "text"

**Вихідні дані:**
- **Output "audio":** повідомлення з голосом/аудіо → йде до "Get a file"
- **Output "text":** текстові повідомлення → йде до "Edit Fields1"

**Credentials:** Не потрібні

---

### Нода 3: Get a file
**Тип:** `n8n-nodes-base.telegram`
**Що робить:** Отримує файл (голосове/аудіо повідомлення) з Telegram

**Вхідні дані:**
```json
{
  "message": {
    "voice": {
      "file_id": "AwADBAADXXXXXXX"
    }
  }
}
```

**Процес обробки:**
1. Витягує `file_id` з `$json.message.voice.file_id` або `$json.message.audio.file_id`
2. Робить запит до Telegram API для завантаження файлу
3. Повертає бінарні дані файлу

**Вихідні дані:**
```json
{
  "data": "<binary audio data>",
  "mimeType": "audio/ogg",
  "fileName": "voice.ogg"
}
```

**Credentials:**
- Тип: Telegram API
- ID: ZWi14sL1REwFLLc8
- Назва: Personal Assistant

---

### Нода 4: Transcribe audio or video
**Тип:** `@elevenlabs/n8n-nodes-elevenlabs.elevenLabs`
**Що робить:** Транскрибує аудіо файл в текст за допомогою ElevenLabs Speech-to-Text API

**Вхідні дані:**
Бінарні аудіо дані з попередньої ноди

**Процес обробки:**
1. Отримує бінарні дані аудіо
2. Відправляє запит до ElevenLabs API (endpoint: speech-to-text)
3. Повертає розпізнаний текст

**Вихідні дані:**
```json
{
  "text": "розпізнаний текст з голосового повідомлення"
}
```

**Credentials:**
- Тип: ElevenLabs API
- ID: Z6j7Jl6dgDXy5okd
- Назва: 11labs_education
- Як отримати:
  1. Зареєструватися на https://elevenlabs.io/
  2. Перейти до Settings → API Keys
  3. Створити новий API ключ
  4. Додати в n8n: Credentials → New → ElevenLabs API → вставити ключ

---

### Нода 5: Edit Fields
**Тип:** `n8n-nodes-base.set`
**Що робить:** Підготовлює дані з аудіо потоку для подальшої обробки

**Вхідні дані:**
```json
{
  "text": "розпізнаний текст",
  "message": {
    "from": {
      "id": 987654321
    }
  }
}
```

**Процес обробки:**
1. Створює поле `setPrompt` = `$json.text` (текст з транскрибації)
2. Створює поле `userID` = `"edu_" + $json.message.from.id` (наприклад: "edu_987654321")

**Вихідні дані:**
```json
{
  "setPrompt": "розпізнаний текст",
  "userID": "edu_987654321"
}
```

**Credentials:** Не потрібні

---

### Нода 6: Edit Fields1
**Тип:** `n8n-nodes-base.set`
**Що робить:** Підготовлює дані з текстового потоку для подальшої обробки

**Вхідні дані:**
```json
{
  "message": {
    "text": "текст повідомлення користувача",
    "from": {
      "id": 987654321
    }
  }
}
```

**Процес обробки:**
1. Створює поле `setPrompt` = `$json.message.text` (текст повідомлення)
2. Створює поле `userID` = `"edu_" + $json.message.from.id`

**Вихідні дані:**
```json
{
  "setPrompt": "текст повідомлення користувача",
  "userID": "edu_987654321"
}
```

**Credentials:** Не потрібні

---

### Нода 7: setupPrompt
**Тип:** `n8n-nodes-base.set`
**Що робить:** Об'єднує потоки (аудіо і текст) та підготовлює фінальні дані

**Вхідні дані:**
```json
{
  "setPrompt": "запит користувача",
  "userID": "edu_987654321"
}
```

**Процес обробки:**
1. Просто передає дані далі без змін
2. Є точкою злиття двох потоків (audio та text)

**Вихідні дані:**
```json
{
  "setPrompt": "запит користувача",
  "userID": "edu_987654321"
}
```

**Credentials:** Не потрібні

---

### Нода 8: Check_User
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Перевіряє чи існує користувач в Zep системі

**Вхідні дані:**
```json
{
  "userID": "edu_987654321"
}
```

**Процес обробки:**
1. Робить GET запит до Zep API: `https://api.getzep.com/api/v2/users/{{ $json.userID }}`
2. Якщо користувач існує - повертає його дані
3. Якщо не існує - повертає помилку (onError: continueRegularOutput)

**Вихідні дані (якщо користувач існує):**
```json
{
  "user_id": "edu_987654321",
  "first_name": "John",
  "last_name": "Doe",
  "created_at": "2024-01-15T10:30:00Z"
}
```

**Вихідні дані (якщо користувача немає):**
```json
{
  "error": "User not found"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account
- Як отримати:
  1. Зареєструватися на https://www.getzep.com/
  2. Отримати API ключ з Dashboard
  3. Додати в n8n: Credentials → New → HTTP Header Auth
  4. Header Name: `Authorization`
  5. Header Value: `Bearer YOUR_API_KEY`

---

### Нода 9: Switch1
**Тип:** `n8n-nodes-base.switch`
**Що робить:** Визначає чи потрібно створювати нового користувача

**Вхідні дані:**
```json
{
  "user_id": "edu_987654321"
}
```

**Процес обробки:**
1. **Правило 1 (no user):** Перевіряє чи `$json.user_id` НЕ співпадає з ID користувача з Telegram
   - Якщо не співпадає або немає - направляє в output "no user" → створення користувача
2. **Правило 2 (user exist):** Перевіряє чи існує поле `$json.user_id`
   - Якщо існує - направляє в output "user exist" → отримання повідомлень

**Вихідні дані:**
- **Output "no user":** → йде до "add_user"
- **Output "user exist":** → йде до "get_message_from_thread"

**Credentials:** Не потрібні

---

### Нода 10: add_user
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Створює нового користувача в Zep системі

**Вхідні дані:**
Дані з Telegram Trigger

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/users`
2. Відправляє JSON body з даними користувача:

**JSON Body:**
```json
{
  "user_id": "edu_987654321",
  "first_name": "John",
  "last_name": "Doe"
}
```

**Вихідні дані:**
```json
{
  "user_id": "edu_987654321",
  "first_name": "John",
  "last_name": "Doe",
  "created_at": "2024-10-30T12:00:00Z"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 11: create_thread
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Створює новий thread (потік розмови) для користувача в Zep

**Вхідні дані:**
Дані з Telegram Trigger

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/threads`
2. Відправляє JSON body:

**JSON Body:**
```json
{
  "thread_id": "edu_987654321",
  "user_id": "edu_987654321"
}
```

**Вихідні дані:**
```json
{
  "thread_id": "edu_987654321",
  "user_id": "edu_987654321",
  "created_at": "2024-10-30T12:00:00Z"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 12: get_message_from_thread
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Отримує останні 50 повідомлень з історії розмови користувача

**Вхідні дані:**
```json
{
  "userID": "edu_987654321"
}
```

**Процес обробки:**
1. Робить GET запит до `https://api.getzep.com/api/v2/threads/{{ "edu_"+$('Telegram Trigger').item.json.message.chat.id }}/messages?lastn=50`
2. Повертає останні 50 повідомлень з історії

**Вихідні дані:**
```json
{
  "messages": [
    {
      "role": "user",
      "content": "Привіт!",
      "created_at": "2024-10-30T10:00:00Z"
    },
    {
      "role": "assistant",
      "content": "Вітаю! Як можу допомогти?",
      "created_at": "2024-10-30T10:00:05Z"
    }
  ]
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 13: get_message_from_graph
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Шукає релевантні зв'язки (edges) в Knowledge Graph на основі запиту користувача

**Вхідні дані:**
```json
{
  "setPrompt": "запит користувача",
  "userID": "edu_987654321"
}
```

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/graph/search`
2. Відправляє JSON body:

**JSON Body:**
```json
{
  "user_id": "edu_987654321",
  "query": "запит користувача",
  "scope": "edges",
  "limit": 20
}
```

**Вихідні дані:**
```json
{
  "edges": [
    {
      "fact": "AI-агенти можуть автоматизувати бізнес-процеси",
      "source": "ai-automation-doc",
      "relevance": 0.92
    },
    {
      "fact": "n8n підтримує інтеграцію з 300+ сервісами",
      "source": "n8n-docs",
      "relevance": 0.85
    }
  ]
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 14: AI Agent
**Тип:** `@n8n/n8n-nodes-langchain.agent`
**Що робить:** Головний AI агент, який обробляє запити користувача використовуючи OpenAI GPT-4.1-mini

**Вхідні дані:**
```json
{
  "setPrompt": "запит користувача"
}
```

**System Prompt:**
```
You are a helpful assistant

Для пошуку інформації із загальної бази знань використовуй інстурмент "get_data_from_db"

Використовуй наступний контекст
ВЗАЄМОЗВЯЗКИ
{{ $('get_message_from_graph').item.json.edges }}
ІНФОРМАЦІЯ
{{ $('get_message_from_thread').item.json.messages }}

ЗАВЖДИ використовуй інстурмент "add_info_to_db" для додавання ключової та унікальної і важливої інформацїі в векторну базу даних
```

**Процес обробки:**
1. Отримує запит користувача через `$('setupPrompt').item.json.setPrompt`
2. Використовує контекст з історії повідомлень (get_message_from_thread)
3. Використовує контекст з Knowledge Graph (get_message_from_graph)
4. Може викликати tools:
   - `get_data_from_db` - для пошуку в загальній базі знань
   - `add_info_to_db` - для додавання важливої інформації
5. Генерує відповідь через OpenAI GPT-4.1-mini

**Вихідні дані:**
```json
{
  "output": "відповідь асистента на запит користувача"
}
```

**Credentials:** Не потрібні (використовує підключені компоненти)

---

### Нода 15: OpenAI Chat Model
**Тип:** `@n8n/n8n-nodes-langchain.lmChatOpenAi`
**Що робить:** Підключає модель OpenAI GPT-4.1-mini до AI Agent

**Параметри:**
- Model: gpt-4.1-mini

**Credentials:**
- Тип: OpenAI API
- ID: r1mXL5JVPJilJ6rh
- Назва: OpenAi account
- Як отримати:
  1. Зареєструватися на https://platform.openai.com/
  2. Перейти до API Keys
  3. Створити новий API ключ
  4. Додати в n8n: Credentials → New → OpenAI API → вставити ключ

---

### Нода 16: Redis Chat Memory
**Тип:** `@n8n/n8n-nodes-langchain.memoryRedisChat`
**Що робить:** Зберігає короткотривалу пам'ять розмови в Redis

**Параметри:**
- Session Key: `$('Telegram Trigger').item.json.message.chat.id` (унікальний для кожного чату)
- Context Window Length: 10 (зберігає останні 10 повідомлень)

**Процес обробки:**
1. Зберігає кожне повідомлення в Redis під ключем chat_id
2. Підтримує контекст останніх 10 повідомлень
3. Автоматично видаляє старі повідомлення

**Credentials:**
- Тип: Redis
- ID: laEVTxNKVymKxvhh
- Назва: Redis EduUpStash
- Як отримати:
  1. Зареєструватися на https://upstash.com/
  2. Створити Redis database
  3. Отримати connection string
  4. Додати в n8n: Credentials → New → Redis

---

### Нода 17: add_message_to_grpah
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Додає пару повідомлень (користувач + асистент) в Zep Knowledge Graph для довготривалої пам'яті

**Вхідні дані:**
```json
{
  "setPrompt": "запит користувача",
  "output": "відповідь асистента"
}
```

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/threads/{{ userID }}/messages`
2. Відправляє JSON body:

**JSON Body:**
```json
{
  "messages": [
    {
      "role": "user",
      "name": "HUMAN",
      "content": "запит користувача"
    },
    {
      "role": "assistant",
      "name": "AI",
      "content": "відповідь асистента"
    }
  ]
}
```

**Вихідні дані:**
```json
{
  "success": true,
  "thread_id": "edu_987654321"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 18: Send a text message
**Тип:** `n8n-nodes-base.telegram`
**Що робить:** Відправляє відповідь асистента користувачу в Telegram

**Вхідні дані:**
```json
{
  "output": "відповідь асистента"
}
```

**Процес обробки:**
1. Отримує chat_id з `$('Telegram Trigger').item.json.message.chat.id`
2. Отримує текст з `$('AI Agent').item.json.output`
3. Відправляє повідомлення через Telegram API

**Вихідні дані:**
```json
{
  "message_id": 67890,
  "chat": {
    "id": 987654321
  },
  "text": "відповідь асистента"
}
```

**Credentials:**
- Тип: Telegram API
- ID: ZWi14sL1REwFLLc8
- Назва: Personal Assistant

---

### Нода 19: Create Graph DB
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Створює новий Knowledge Graph в Zep (виконується один раз на початку)

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/graph/create`
2. Відправляє JSON body:

**JSON Body:**
```json
{
  "graph_id": "ai-agent-edu-knowledge",
  "name": "AI Agent Knowledge",
  "description": "Docs from Google Drive"
}
```

**Вихідні дані:**
```json
{
  "graph_id": "ai-agent-edu-knowledge",
  "created_at": "2024-10-30T12:00:00Z"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 20: Get a file1
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Завантажує PDF файл з інтернету для додавання в базу знань

**Параметри:**
- URL: https://ai-2027.com/ai-2027.pdf

**Процес обробки:**
1. Робить GET запит до вказаного URL
2. Завантажує PDF файл

**Вихідні дані:**
Бінарні дані PDF файлу

**Credentials:** Не потрібні

---

### Нода 21: Extract from File
**Тип:** `n8n-nodes-base.extractFromFile`
**Що робить:** Витягує текст з PDF файлу

**Вхідні дані:**
Бінарні дані PDF файлу

**Процес обробки:**
1. Парсить PDF файл
2. Витягує текстовий контент
3. Повертає метадані файлу

**Вихідні дані:**
```json
{
  "text": "повний текст з PDF файлу...",
  "metadata": {
    "xmp:modifydate": "2024-10-30T12:00:00Z",
    "pdf:producer": "Adobe PDF",
    "dc:title": "AI 2027"
  },
  "info": {
    "Title": "AI 2027 - Future of AI"
  }
}
```

**Credentials:** Не потрібні

---

### Нода 22: Chunking
**Тип:** `n8n-nodes-base.code`
**Що робить:** Розбиває великий текст на частини (chunks) по 2000 символів

**Вхідні дані:**
```json
{
  "text": "дуже довгий текст...",
  "info": {
    "Title": "AI 2027"
  },
  "metadata": {
    "xmp:modifydate": "2024-10-30"
  }
}
```

**JavaScript код:**
```javascript
const s = $json.text || "";
const chunks = [];
const SIZE = 2000;  // безпечна довжина
for (let i=0; i<s.length; i+=SIZE) {
  chunks.push({
    type: "text",
    data: s.slice(i, i+SIZE),
    source_description: $input.first().json.info.Title,
    created_at: $json.modifiedTime
  });
}
return chunks.map(e => ({ json: e }));
```

**Процес обробки:**
1. Отримує текст з попередньої ноди
2. Розбиває на частини по 2000 символів
3. Створює окремий об'єкт для кожного чанку

**Вихідні дані:**
```json
[
  {
    "type": "text",
    "data": "перші 2000 символів...",
    "source_description": "AI 2027",
    "created_at": "2024-10-30"
  },
  {
    "type": "text",
    "data": "наступні 2000 символів...",
    "source_description": "AI 2027",
    "created_at": "2024-10-30"
  }
]
```

**Credentials:** Не потрібні

---

### Нода 23: HTTP Request
**Тип:** `n8n-nodes-base.httpRequest`
**Що робить:** Додає кожен чанк тексту в Zep Knowledge Graph

**Вхідні дані:**
```json
{
  "type": "text",
  "data": "частина тексту",
  "source_description": "AI 2027",
  "created_at": "2024-10-30"
}
```

**Процес обробки:**
1. Робить POST запит до `https://api.getzep.com/api/v2/graph`
2. Для кожного чанку відправляє JSON body:

**JSON Body:**
```json
{
  "graph_id": "ai-agent-edu-knowledge",
  "type": "text",
  "data": "частина тексту",
  "source_description": "",
  "created_at": "2024-10-30T12:00:00Z"
}
```

**Вихідні дані:**
```json
{
  "success": true,
  "node_id": "abc123"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 24: get_data_from_db (Tool)
**Тип:** `n8n-nodes-base.httpRequestTool`
**Що робить:** AI Tool для пошуку інформації в загальній базі знань

**Tool Description:** "Запускай даний інстурмент коли необхідно занйти інфомрацію із загальної бази заннь"

**Процес обробки:**
1. AI Agent викликає цей tool коли потрібно знайти інформацію
2. Робить POST запит до `https://api.getzep.com/api/v2/graph/search`
3. Використовує параметр `$fromAI("ai_response_for_database_search")` як запит

**JSON Body:**
```json
{
  "query": "запит від AI агента",
  "graph_id": "ai-agent-edu-knowledge",
  "limit": 5
}
```

**Вихідні дані:**
```json
{
  "results": [
    {
      "content": "знайдена інформація 1",
      "relevance": 0.95
    },
    {
      "content": "знайдена інформація 2",
      "relevance": 0.87
    }
  ]
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

### Нода 25: add_info_to_db (Tool)
**Тип:** `n8n-nodes-base.httpRequestTool`
**Що робить:** AI Tool для додавання важливої інформації в базу знань

**Tool Description:** "Використовуй даний інстурмент для додавання ключовї унікальної інформації в векторну базу"

**Процес обробки:**
1. AI Agent викликає цей tool коли потрібно зберегти важливу інформацію
2. Робить POST запит до `https://api.getzep.com/api/v2/graph`
3. Використовує параметр `$fromAI("summary_of_communication", "передавай сюди ключову інформацію з комунікації, якої ще немає в твої базі")`

**JSON Body:**
```json
{
  "graph_id": "ai-agent-edu-knowledge",
  "type": "text",
  "data": "ключова інформація для збереження"
}
```

**Вихідні дані:**
```json
{
  "success": true,
  "node_id": "xyz789"
}
```

**Credentials:**
- Тип: HTTP Header Auth
- ID: YVtwOVo5MDifVjzb
- Назва: Zep DEV Account

---

## 4. Умови спрацювання

### Для основного флоу:
**Спрацьовує коли:**
- Користувач відправляє текстове повідомлення боту
- Користувач відправляє голосове повідомлення боту
- Користувач відправляє аудіо файл боту

**Приклади запитів що викликають workflow:**
- "Розкажи про AI агентів"
- "Що таке n8n?"
- "Збережи цю інформацію: n8n - це платформа автоматизації"
- (Голосове повідомлення): "Привіт, як справи?"

**Приклади що НЕ викликають workflow:**
- Фото або відео (не обробляються)
- Документи (крім аудіо)
- Стікери, GIF-ки

### Для Tool "get_data_from_db":
**Спрацьовує коли:**
- Користувач запитує інформацію, якої немає в контексті розмови
- AI агент визначає, що потрібно шукати в загальній базі знань

**Приклади запитів:**
- "Що написано в документації про AI 2027?"
- "Знайди інформацію про автоматизацію бізнесу"
- "Шукай в базі дані про інтеграції"

**НЕ викликає tool:**
- "Привіт, як справи?" (загальна розмова)
- "Скільки буде 2+2?" (простий розрахунок)

### Для Tool "add_info_to_db":
**Спрацьовує коли:**
- Користувач явно просить зберегти інформацію
- В розмові з'являється унікальна важлива інформація

**Приклади запитів:**
- "Запам'ятай: мій улюблений колір - синій"
- "Збережи: наш проект стартує 1 листопада"
- "Додай в базу: API ключ для сервісу X"

**НЕ викликає tool:**
- "Привіт" (немає важливої інформації)
- "Дякую" (немає що зберігати)

---

## 5. Налаштування Credentials

### 1. Telegram API (Personal Assistant)
**Де використовується:**
- Telegram Trigger
- Get a file
- Send a text message

**Як отримати:**
1. Відкрийте Telegram
2. Знайдіть бота @BotFather
3. Напишіть `/newbot`
4. Введіть назву бота (наприклад: "My Personal Assistant")
5. Введіть username бота (має закінчуватися на "bot", наприклад: "my_personal_assistant_bot")
6. Отримаєте токен у форматі: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`

**Додавання в n8n:**
1. Перейдіть до Credentials → New
2. Виберіть "Telegram API"
3. Вставте токен в поле "Access Token"
4. Назвіть credentials: "Personal Assistant"
5. Збережіть

---

### 2. ElevenLabs API (11labs_education)
**Де використовується:**
- Transcribe audio or video

**Як отримати:**
1. Зареєструйтеся на https://elevenlabs.io/
2. Перейдіть до Settings → API Keys
3. Натисніть "Create API Key"
4. Скопіюйте згенерований ключ

**Додавання в n8n:**
1. Перейдіть до Credentials → New
2. Виберіть "ElevenLabs API"
3. Вставте API ключ в поле "API Key"
4. Назвіть credentials: "11labs_education"
5. Збережіть

---

### 3. OpenAI API (OpenAi account)
**Де використовується:**
- OpenAI Chat Model

**Як отримати:**
1. Зареєструйтеся на https://platform.openai.com/
2. Перейдіть до API Keys: https://platform.openai.com/api-keys
3. Натисніть "Create new secret key"
4. Назвіть ключ (наприклад: "n8n-assistant")
5. Скопіюйте ключ (він показується лише один раз!)

**Додавання в n8n:**
1. Перейдіть до Credentials → New
2. Виберіть "OpenAI API"
3. Вставте API ключ в поле "API Key"
4. Назвіть credentials: "OpenAi account"
5. Збережіть

---

### 4. Redis (Redis EduUpStash)
**Де використовується:**
- Redis Chat Memory

**Як отримати:**
1. Зареєструйтеся на https://upstash.com/
2. Створіть новий Redis database
3. Виберіть регіон (ближче до вас = швидше)
4. Скопіюйте connection string

**Додавання в n8n:**
1. Перейдіть до Credentials → New
2. Виберіть "Redis"
3. Вставте connection string або заповніть поля:
   - Host: (з Upstash dashboard)
   - Port: (зазвичай 6379)
   - Password: (з Upstash dashboard)
   - Database: 0
4. Назвіть credentials: "Redis EduUpStash"
5. Збережіть

---

### 5. HTTP Header Auth (Zep DEV Account)
**Де використовується:**
- Check_User
- add_user
- create_thread
- get_message_from_thread
- get_message_from_graph
- add_message_to_grpah
- Create Graph DB
- HTTP Request (chunking)
- get_data_from_db
- add_info_to_db

**Як отримати:**
1. Зареєструйтеся на https://www.getzep.com/
2. Перейдіть до Dashboard
3. Знайдіть секцію "API Keys"
4. Створіть новий API ключ
5. Скопіюйте ключ

**Додавання в n8n:**
1. Перейдіть до Credentials → New
2. Виберіть "HTTP Header Auth"
3. Заповніть поля:
   - Header Name: `Authorization`
   - Header Value: `Bearer YOUR_ZEP_API_KEY` (замініть YOUR_ZEP_API_KEY на ваш ключ)
4. Назвіть credentials: "Zep DEV Account"
5. Збережіть

---

## 6. Troubleshooting

### Помилка: "Telegram bot not responding"
**Причина:** Невірний токен бота або бот не активований

**Рішення:**
1. Перевірте токен в @BotFather через команду `/token`
2. Переконайтеся, що бот не був видалений
3. Оновіть credentials в n8n з правильним токеном

---

### Помилка: "ElevenLabs API error 401"
**Причина:** Невірний або застарілий API ключ

**Рішення:**
1. Перевірте API ключ на https://elevenlabs.io/settings
2. Створіть новий ключ якщо потрібно
3. Оновіть credentials в n8n

---

### Помилка: "OpenAI API error 429 - Rate limit exceeded"
**Причина:** Перевищено ліміт запитів до OpenAI API

**Рішення:**
1. Зачекайте кілька хвилин
2. Перевірте ваші ліміти на https://platform.openai.com/account/limits
3. Додайте більше коштів на акаунт якщо потрібно
4. Розгляньте можливість використання черги для запитів

---

### Помилка: "Redis connection timeout"
**Причина:** Проблеми з підключенням до Redis

**Рішення:**
1. Перевірте чи працює Redis сервер на Upstash dashboard
2. Перевірте connection string
3. Переконайтеся, що firewall не блокує з'єднання
4. Перезапустіть Redis database в Upstash

---

### Помилка: "Zep API error 404 - User not found"
**Причина:** Користувач не створений в Zep

**Рішення:**
Це нормально для нових користувачів - workflow автоматично створить користувача. Якщо помилка повторюється:
1. Перевірте credentials для Zep
2. Переконайтеся, що API ключ має права на створення користувачів
3. Вручну створіть користувача через Zep Dashboard

---

### Помилка: "Graph not found"
**Причина:** Knowledge Graph не створений

**Рішення:**
1. Запустіть окремо ноду "Create Graph DB"
2. Переконайтеся, що graph_id = "ai-agent-edu-knowledge"
3. Перевірте на Zep Dashboard чи граф створений

---

### Помилка: "Cannot read property 'json' of undefined"
**Причина:** Помилка в expressions при доступі до даних попередньої ноди

**Рішення:**
1. Перевірте чи всі попередні ноди виконалися успішно
2. Використайте Test Workflow для перевірки даних на кожному кроці
3. Додайте перевірки існування даних: `$json?.field ?? "default"`

---

### Помилка: "AI Agent returns empty response"
**Причина:** Проблеми з промптом або моделлю

**Рішення:**
1. Перевірте System Prompt в AI Agent ноді
2. Переконайтеся, що OpenAI API працює
3. Перевірте чи передаються дані в AI Agent:
   ```javascript
   // В setupPrompt ноді переконайтеся що:
   $json.setPrompt // не пусте
   ```
4. Збільште timeout для AI Agent

---

### Помилка: "Chunking fails with large PDF"
**Причина:** PDF файл занадто великий

**Рішення:**
1. Зменшіть SIZE константу в Chunking ноді (наприклад, до 1000)
2. Додайте фільтрацію порожніх чанків:
   ```javascript
   const chunks = [];
   for (let i=0; i<s.length; i+=SIZE) {
     const chunk = s.slice(i, i+SIZE).trim();
     if (chunk.length > 0) {
       chunks.push({...});
     }
   }
   ```
3. Розгляньте можливість обробки PDF частинами

---

### Помилка: "Workflow execution timeout"
**Причина:** Workflow виконується занадто довго

**Рішення:**
1. Збільште timeout в Settings → Workflow Settings
2. Оптимізуйте workflow:
   - Зменшіть кількість повідомлень з історії (lastn=50 → lastn=20)
   - Зменшіть context window в Redis (10 → 5)
3. Розділіть workflow на кілька менших workflows

---

## 7. Приклади використання

### Приклад 1: Текстовий запит
**Користувач відправляє:** "Що таке AI агенти?"

**Потік виконання:**
1. Telegram Trigger отримує повідомлення
2. Switch направляє в "text" вихід
3. Edit Fields1 створює `setPrompt = "Що таке AI агенти?"`
4. setupPrompt передає далі
5. Check_User перевіряє користувача (існує)
6. get_message_from_thread отримує історію
7. get_message_from_graph шукає релевантну інформацію
8. AI Agent обробляє запит з контекстом
9. add_message_to_grpah зберігає розмову
10. Send a text message відправляє відповідь

**Відповідь бота:** "AI агенти - це автономні програми, які використовують штучний інтелект для виконання завдань без постійного втручання людини. Вони можуть аналізувати дані, приймати рішення та взаємодіяти з іншими системами..."

---

### Приклад 2: Голосове повідомлення
**Користувач відправляє:** (Голосове) "Розкажи про n8n автоматизацію"

**Потік виконання:**
1. Telegram Trigger отримує голосове повідомлення
2. Switch направляє в "audio" вихід
3. Get a file завантажує аудіо
4. Transcribe audio or video розпізнає: "Розкажи про n8n автоматизацію"
5. Edit Fields створює `setPrompt = "Розкажи про n8n автоматизацію"`
6. Далі як у прикладі 1...

---

### Приклад 3: Додавання інформації в базу
**Користувач відправляє:** "Запам'ятай: наш офіс працює з 9 до 18 години"

**Потік виконання:**
1-8. Стандартний потік...
9. AI Agent викликає tool `add_info_to_db` з даними: "Офіс працює з 9 до 18 години"
10. Tool додає інформацію в Knowledge Graph
11. add_message_to_grpah зберігає розмову
12. Send a text message: "Зрозумів, я запам'ятав що ваш офіс працює з 9 до 18 години"

---

### Приклад 4: Пошук в базі знань
**Користувач відправляє:** "Що ти знаєш про AI в 2027 році?"

**Потік виконання:**
1-8. Стандартний потік...
9. AI Agent викликає tool `get_data_from_db` з запитом: "AI 2027"
10. Tool знаходить релевантну інформацію з PDF документу
11. AI Agent формує відповідь на основі знайденої інформації
12. add_message_to_grpah зберігає розмову
13. Send a text message: "Згідно з документом AI 2027, очікується що..."

---

## 8. Статистика Workflow

- **Назва:** Personal assistent with Dynamic VDB
- **Кількість нод:** 25
- **Активний:** false (вимкнений)
- **Типи нод:**
  - Triggers: 1 (Telegram Trigger)
  - Actions: 18 (HTTP Requests, Edit Fields, Transcribe, Extract, тощо)
  - AI Nodes: 3 (AI Agent, OpenAI Chat Model, Redis Chat Memory)
  - Tools: 2 (get_data_from_db, add_info_to_db)
  - Utility: 1 (Switch, Chunking Code)

---

## 9. Вимоги та залежності

### Зовнішні сервіси:
1. **Telegram** - для бота
2. **ElevenLabs** - для транскрибації аудіо
3. **OpenAI** - для AI моделі GPT-4.1-mini
4. **Redis (Upstash)** - для короткотривалої пам'яті
5. **Zep** - для довготривалої пам'яті та Knowledge Graph

### Мінімальні вимоги:
- n8n версія: 1.0+
- Node.js: 18+
- Інтернет з'єднання для всіх API

### Вартість (орієнтовно):
- Telegram API: безкоштовно
- ElevenLabs: від $5/міс (залежно від використання)
- OpenAI GPT-4.1-mini: ~$0.15 за 1M input tokens, ~$0.60 за 1M output tokens
- Upstash Redis: безкоштовний план (10,000 команд/день)
- Zep: від $0 (є безкоштовний план)

---

## 10. Додаткова інформація

### Як імпортувати workflow:
1. Відкрийте n8n
2. Натисніть "Import from File" або "Import from URL"
3. Виберіть файл "Personal assistent with Dynamic VDB -15.json"
4. Налаштуйте всі credentials (дивіться розділ 5)
5. Активуйте workflow

### Як тестувати:
1. Переконайтеся, що всі credentials налаштовані
2. Спочатку запустіть ноду "Create Graph DB" окремо
3. Активуйте workflow
4. Відправте тестове повідомлення боту в Telegram
5. Перевірте логи в n8n для діагностики

### Розширення функціоналу:
- Додайте підтримку зображень
- Інтеграція з Google Drive для автоматичного імпорту документів
- Додайте інші AI tools (наприклад, калькулятор, погода, тощо)
- Налаштуйте кастомні промпти для різних типів запитів

---

**Дата створення документації:** 2024-10-30
**Версія workflow:** 1.0
**Автор документації:** Сергій Щербаков
