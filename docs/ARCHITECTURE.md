# ARCHITECTURE.md — архитектура Tilda RAG Chatbot

## Основные принципы

1. **Опубликованная Tilda = источник истины**. Бот не требует ручного заполнения знаний.
2. **Внешний публичный сайт не является доверенным кодом**. HTML и текст страниц могут содержать скрытые токены, PII или prompt injection.
3. **Public chat ≠ admin MCP**. Посетители не получают вызовов Tilda get_project_settings, edit_record, update_record, публикации, черновиков или CMS-заявок.
4. **Fail closed**. Неудача извлечения, подозрительный секрет или некорректный источник запрещают публикацию соответствующего фрагмента.
5. Браузерный виджет содержит только публичный конфиг (Worker URL, локаль, стиль), без LLM/Тilda секретов.

## Поток индексации

```text
Comfortclinic.kz sitemap.xml
        │ allowlist HTTPS + robots policy
        ▼
Crawler (GitHub Actions, scheduled + workflow_dispatch)
        │ fetch pages, ETag/Last-Modified/hash when available
        ▼
Extractor: visible text, headings, tables, titles, canonical links
        │ excludes scripts, forms, hidden fields, comments, Tilda admin snippets
        ▼
Sanitizer: secrets + PII + prompt-injection patterns + HTML filtering
        ▼
Segmenter + metadata + stable source URLs
        ▼
Validation (schema, link allowlist, no secrets, no duplicate chunks)
        ▼
Versioned index artifact → Cloudflare storage
        │ atomic activation, rollback previous version
        ▼
Retrieval available to read-only Worker
```

### Первичный crawler

- Разрешённые хосты: строго `comfortclinic.kz` и заранее отдельно добавленные публичные поддомены при необходимости. URL задаёт конфиг, а не посетитель/LLM.
- Никаких произвольных URL, даже при ссылках в sitemap.
- Ограничение глубины, количества страниц, bytes/page и общего времени.
- Проверка MIME/кодировки, статуса ответа, редиректов. При переходе на иной домен отказ.
- SSRF-защита: проверка публичного адреса с учётом DNS/redirect и запрет внутренних/private/loopback/link-local IP.
- Не индексировать страницы с авторизацией, приватные или технические endpoints.
- Проверять sitemap и canonical, исключать дубли и URL-параметры вроде токенов/сессий.
- На Tilda Zero Blocks и табах отдельно подтвердить видимый текст, сохранение смысловой структуры, наличие актуальных цен.
- При ошибке публикации сохранить предыдущую версию индекса, но помечать её возраст.

### Обновление

- Плановый запуск, например каждые 30 минут, если позволяет GitHub Actions и правила хостинга; реальное время запуска **не гарантируется**.
- `workflow_dispatch` для ручной переиндексации после публикации важных цен.
- Fetch only modified pages при надёжных validators; иначе content hash.
- Если страница исчезла с sitemap, проверить факт удаления/закрытия, затем исключить из новой версии индекса.
- Если сайт временно недоступен, **не обнулять весь индекс**.
- Подписывать версию, сохранять timestamp, собирать отчёт об удалениях/добавлениях.

## Поток чата

```text
Tilda T123 widget
    │ POST /api/chat: {message, locale, minimal session state}
    ▼
Cloudflare Worker
    │ input validation, CORS allowlist, rate-limit, budget
    ▼
Retriever (read-only sanitized index)
    │ chunks with sourceUrl + score + freshness
    ▼
LLM (instructions separated from untrusted evidence)
    │ concise grounded response (no untrusted tools)
    ▼
Server-side answer checks (source allowlist, citations, price grounding)
    ▼
Widget (text-safe rendering, source links)
```

**Никаких прямых вызовов Tilda MCP из Worker от имени посетителя.** Tilda MCP можно использовать отдельно в админ-разработке, но публичная база строится только по опубликованному сайту.

## API-контракт MVP

`POST /api/chat` принимает JSON с `message` (лимит символов), `locale` (ru/kz) и необязательным коротким session id без PII. Ответ:

```json
{
  "answer": "Ответ на основе опубликованного сайта",
  "sources": [
    {"title": "Чекапы", "url": "https://comfortclinic.kz/checkup"}
  ],
  "indexVersion": "2026-11-07T00:00:00Z",
  "fallback": false
}
```

При недостатке подтверждённых данных `fallback: true`, краткое объяснение и официальный контакт/ссылка. Источники подбираются и проверяются сервером, **не подставляются из непроверенной генерации модели**. Не использовать модельный HTML в интерфейсе.

`GET /api/health`: отвечает только общим статусом, версией/возрастом индекса; не раскрывает ключи или служебные URL.

## Данные и хранение

Документ: `{url,title,language,fetchedAt,contentHash,chunks:[{id,heading,text,sourceUrl}]}`. Хранить сокращённый **очищенный** текст. Никогда не сохранять сырые приватные поля, JWT, bearer, токены Tilda, идентификаторы пациентов, входы администратора.

MVP: компактный индекс в Cloudflare R2/KV, либо иной постоянный storage. Размещение индекса в **публичном GitHub** допустимо только после технической проверки, что все данные действительно безопасны для публикации. Смена модели retrieval или переход на embeddings не должна менять внешний Chat API. Векторная БД — опция, не стартовое обязательство.

## Frontend

- Самодостаточный виджет для Tilda T123; изоляция классов CSS, не менять штатные блоки.
- Стиль Comfort Clinic, акцент `#004341`; адаптивная панель, мобильный режим.
- RU/KZ, клавиатурная навигация, aria-live ответа, доступная кнопка закрытия.
- Статусы загрузки/ошибки, источник рядом с каждым ответом.
- Опционально: кнопки к записи и чекапам, только если ссылки действительно опубликованы.
- Не сохранять историю в localStorage по умолчанию.

## Разделение среды

- GitHub: source code, workflow, tests, docs.
- GitHub Actions: ограниченный crawler/transformer/QA.
- Cloudflare Workers: chat API, server secrets, rate limit, cache access.
- Tilda: виджет + публичный контент.
- Tilda administrative MCP: **только для владельца/оператора вне публичного маршрута**.

## Тестирование

1. Контроль актуализации цены после publish.
2. Удаление страницы и исчезновение из выдачи.
3. Исключение скрытых форм, скриптов и тестовых токенов.
4. Неиндексирование защищённых страниц и чужих доменов.
5. Injection в содержимом страницы и сообщении посетителя.
6. Точное цитирование и отказ при нехватке данных.
7. Rate limiting, бюджет, burst, CORS, XSS.
8. RU/KZ, мобильное отображение, Tilda popup/links.
9. Отказ при недоступном LLM и откат индекса.
