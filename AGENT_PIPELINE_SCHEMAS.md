# Схемы работы агента Scira

Этот документ содержит полное описание всех пайплайнов работы агента, схем вызова промптов и передачи данных между компонентами.

**Дата создания:** 2025-11-11
**Версия:** Актуальная версия в репозитории

---

## Оглавление

1. [Общая архитектура](#1-общая-архитектура)
2. [Extreme Search Pipeline](#2-extreme-search-pipeline)
3. [Мультизапросные пайплайны](#3-мультизапросные-пайплайны)
4. [Специализированные пайплайны](#4-специализированные-пайплайны)
5. [Передача данных между этапами](#5-передача-данных-между-этапами)
6. [Управление группами и маршрутизация](#6-управление-группами-и-маршрутизация)

---

## 1. Общая архитектура

### 1.1. Группо-ориентированная архитектура

Система использует **группы (SearchGroupId)** для определения, какие инструменты и промпты использовать:

```
User Request
     ↓
┌────────────────────────────────────┐
│  API Endpoint: /api/search         │
│  POST { group, messages, ... }     │
└────────────────────────────────────┘
     ↓
┌────────────────────────────────────┐
│  getGroupConfig(group)             │
│  ├─ tools: массив доступных tools  │
│  └─ instructions: системный промпт │
└────────────────────────────────────┘
     ↓
┌────────────────────────────────────┐
│  streamText({                      │
│    model,                          │
│    tools: baseTools + groupTools,  │
│    activeTools: group-specific,    │
│    system: instructions            │
│  })                                │
└────────────────────────────────────┘
     ↓
   Response
```

### 1.2. Доступные группы и их инструменты

```
┌─────────────┬─────────────────────────────────────────────────┐
│   Группа    │              Активные инструменты               │
├─────────────┼─────────────────────────────────────────────────┤
│ web         │ web_search, code_interpreter, get_weather_data, │
│             │ retrieve, text_translate, nearby_places_search, │
│             │ track_flight, movie_or_tv_search, datetime      │
├─────────────┼─────────────────────────────────────────────────┤
│ academic    │ academic_search, code_interpreter, datetime     │
├─────────────┼─────────────────────────────────────────────────┤
│ youtube     │ youtube_search, datetime                        │
├─────────────┼─────────────────────────────────────────────────┤
│ code        │ code_context                                    │
├─────────────┼─────────────────────────────────────────────────┤
│ reddit      │ reddit_search, datetime                         │
├─────────────┼─────────────────────────────────────────────────┤
│ x           │ x_search                                        │
├─────────────┼─────────────────────────────────────────────────┤
│ stocks      │ stock_chart, currency_converter, datetime       │
├─────────────┼─────────────────────────────────────────────────┤
│ crypto      │ coin_data, coin_ohlc, coin_data_by_contract,    │
│             │ datetime                                        │
├─────────────┼─────────────────────────────────────────────────┤
│ extreme     │ extreme_search                                  │
├─────────────┼─────────────────────────────────────────────────┤
│ memory      │ search_memories, add_memory, datetime           │
├─────────────┼─────────────────────────────────────────────────┤
│ connectors  │ connectors_search, datetime (Pro only)          │
├─────────────┼─────────────────────────────────────────────────┤
│ chat        │ (нет инструментов - только чат)                 │
└─────────────┴─────────────────────────────────────────────────┘
```

---

## 2. Extreme Search Pipeline

**Группа:** `extreme`
**Файл:** `/home/user/scira/lib/tools/extreme-search.ts`
**Назначение:** Автономный агент глубоких исследований с планированием и параллельными поисками

### 2.1. Общая схема пайплайна

```
User Query
    ↓
┌─────────────────────────────────────────────────────────┐
│ ЭТАП 1: ПЛАНИРОВАНИЕ ИССЛЕДОВАНИЯ                       │
│ Model: scira-grok-4-fast-think                          │
│ Mode: generateObject                                    │
├─────────────────────────────────────────────────────────┤
│ Input:                                                  │
│   - Запрос пользователя                                 │
│   - Сегодняшняя дата                                    │
│                                                         │
│ Task:                                                   │
│   Разбить тему на plan с подзадачами                    │
│                                                         │
│ Output Schema:                                          │
│   {                                                     │
│     plan: [                                             │
│       {                                                 │
│         title: "Research Theme",                        │
│         todos: ["subtask1", "subtask2", ...]            │
│       }                                                 │
│     ]                                                   │
│   }                                                     │
│                                                         │
│ Constraints:                                            │
│   - 1-5 тем максимум                                    │
│   - 3-5 подзадач на тему                                │
│   - Максимум 15 действий всего                          │
│                                                         │
│ Stream Event:                                           │
│   data-extreme_search {                                 │
│     kind: 'plan',                                       │
│     status: { title },                                  │
│     plan: [...]                                         │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ЭТАП 2: АВТОНОМНЫЙ АГЕНТ ИССЛЕДОВАНИЯ                   │
│ Model: scira-grok-4-fast-think                          │
│ Mode: generateText                                      │
├─────────────────────────────────────────────────────────┤
│ Configuration:                                          │
│   - stopWhen: stepCountIs(totalTodos)                   │
│   - maxRetries: 10                                      │
│   - Parallel tool calls: false                          │
│                                                         │
│ Available Tools:                                        │
│   ┌─────────────────────────────────────────┐           │
│   │ 1. webSearch(query, category?,          │           │
│   │              includeDomains?)           │           │
│   │                                         │           │
│   │ 2. xSearch(query, startDate?,           │           │
│   │            endDate?, xHandles?,         │           │
│   │            maxResults?)                 │           │
│   │                                         │           │
│   │ 3. codeRunner(title, code)              │           │
│   └─────────────────────────────────────────┘           │
│                                                         │
│ Strategy (from system prompt):                          │
│   - 95% поиск, 5% код                                   │
│   - Broad → Specific → Recent → Expert                  │
│   - Queries: 5-15 words max                             │
│   - Продолжение с целевыми запросами                    │
│   - Cross-validation источников                         │
└─────────────────────────────────────────────────────────┘
    ↓
    ├─────────────────────┬─────────────────────┬─────────────────────┐
    ↓                     ↓                     ↓                     ↓
┌─────────┐       ┌─────────┐         ┌─────────┐          ┌─────────┐
│webSearch│       │xSearch  │         │codeRunner│         │ Цикл    │
│(детали  │       │(детали  │         │(детали   │         │повторяет│
│ниже)    │       │ниже)    │         │ниже)     │         │ся       │
└─────────┘       └─────────┘         └─────────┘          └─────────┘
    ↓                     ↓                     ↓                     ↓
    └─────────────────────┴─────────────────────┴─────────────────────┘
                                    ↓
┌─────────────────────────────────────────────────────────┐
│ ЭТАП 3: СБОРКА РЕЗУЛЬТАТОВ                              │
├─────────────────────────────────────────────────────────┤
│ Aggregation:                                            │
│   - toolResults: все вызовы инструментов                │
│   - allSources: дедупликация по URL                     │
│   - Content trimming: max 3000 символов на source       │
│   - charts: извлечение из codeRunner                    │
│                                                         │
│ Stream Event:                                           │
│   data-extreme_search {                                 │
│     kind: 'plan',                                       │
│     status: { title: 'Research completed' }             │
│   }                                                     │
│                                                         │
│ Final Output:                                           │
│   {                                                     │
│     text: "Agent's final synthesis",                    │
│     toolResults: [...],                                 │
│     sources: [{                                         │
│       title, url, content,                              │
│       publishedDate, favicon                            │
│     }],                                                 │
│     charts: [...]                                       │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
  User
```

### 2.2. Детальная схема инструмента webSearch

```
webSearch(query, category?, includeDomains?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: START                                     │
│   data-extreme_search {                                 │
│     kind: 'query',                                      │
│     queryId: nanoid(),                                  │
│     query: string,                                      │
│     status: 'started'                                   │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОИСК ЧЕРЕЗ EXA API                                     │
├─────────────────────────────────────────────────────────┤
│ exa.searchAndContents(query, {                          │
│   numResults: 8,                                        │
│   type: 'auto',                                         │
│   category?: category,                                  │
│   include_domains?: includeDomains                      │
│ })                                                      │
│                                                         │
│ Returns: [{                                             │
│   title, url, text,                                     │
│   publishedDate, favicon                                │
│ }]                                                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ STREAM: Sources Found                                   │
│                                                         │
│ Для каждого результата:                                 │
│   data-extreme_search {                                 │
│     kind: 'source',                                     │
│     queryId,                                            │
│     source: { title, url, favicon }                     │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОЛУЧЕНИЕ ПОЛНОГО КОНТЕНТА                              │
├─────────────────────────────────────────────────────────┤
│ Stream Event:                                           │
│   data-extreme_search {                                 │
│     kind: 'query',                                      │
│     queryId,                                            │
│     status: 'reading_content'                           │
│   }                                                     │
│                                                         │
│ Primary: exa.getContents(urls, {                        │
│   livecrawl: 'preferred',                               │
│   maxCharacters: 3000                                   │
│ })                                                      │
│                                                         │
│ Fallback (если Exa не вернул content):                  │
│   firecrawl.scrapeUrl(url, {                            │
│     formats: ['markdown'],                              │
│     onlyMainContent: true,                              │
│     maxCharacters: 3000                                 │
│   })                                                    │
│                                                         │
│ Stream: Content Retrieved                               │
│   data-extreme_search {                                 │
│     kind: 'content',                                    │
│     queryId,                                            │
│     content: { title, url, text, favicon }              │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: COMPLETED                                 │
│   data-extreme_search {                                 │
│     kind: 'query',                                      │
│     queryId,                                            │
│     status: 'completed'                                 │
│   }                                                     │
│                                                         │
│ Return to Agent:                                        │
│   [{                                                    │
│     title, url, content,                                │
│     publishedDate                                       │
│   }]                                                    │
└─────────────────────────────────────────────────────────┘
```

### 2.3. Детальная схема инструмента xSearch

```
xSearch(query, startDate?, endDate?, xHandles?, maxResults?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: START                                     │
│   data-extreme_search {                                 │
│     kind: 'x_search',                                   │
│     xSearchId: nanoid(),                                │
│     query, startDate, endDate,                          │
│     handles: xHandles,                                  │
│     status: 'started'                                   │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОИСК ЧЕРЕЗ GROK API                                    │
├─────────────────────────────────────────────────────────┤
│ generateText({                                          │
│   model: 'grok-4-fast-non-reasoning',                   │
│   messages: [{ role: 'user', content: query }],         │
│   maxOutputTokens: 10,                                  │
│                                                         │
│   providerOptions: {                                    │
│     xai: {                                              │
│       searchParameters: {                               │
│         mode: 'on',                                     │
│         fromDate: startDate || 15 days ago,             │
│         toDate: endDate || today,                       │
│         maxSearchResults: maxResults || 15,             │
│         returnCitations: true,                          │
│         sources: [{                                     │
│           type: 'x',                                    │
│           includedXHandles?: xHandles                   │
│         }]                                              │
│       }                                                 │
│     }                                                   │
│   }                                                     │
│ })                                                      │
│                                                         │
│ Returns:                                                │
│   {                                                     │
│     text: "minimal response",                           │
│     experimental_providerMetadata: {                    │
│       xai: {                                            │
│         citations: [{ url: tweet_url }]                 │
│       }                                                 │
│     }                                                   │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОЛУЧЕНИЕ ПОЛНЫХ ТВИТОВ                                 │
├─────────────────────────────────────────────────────────┤
│ Для каждого citation.url:                               │
│                                                         │
│   Extract tweetId from URL                              │
│     ↓                                                   │
│   getTweet(tweetId) // Exa Tweet Scraper API            │
│     ↓                                                   │
│   Returns: {                                            │
│     text: "tweet content",                              │
│     link: "https://x.com/...",                          │
│     title: "@handle: preview..."                        │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: COMPLETED                                 │
│   data-extreme_search {                                 │
│     kind: 'x_search',                                   │
│     xSearchId,                                          │
│     status: 'completed',                                │
│     result: {                                           │
│       content, citations,                               │
│       sources: [{ text, link, title }]                  │
│     }                                                   │
│   }                                                     │
│                                                         │
│ Return to Agent:                                        │
│   {                                                     │
│     content: string,                                    │
│     citations: [...],                                   │
│     sources: [{ text, link, title }],                   │
│     dateRange: { fromDate, toDate },                    │
│     handles: xHandles                                   │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
```

### 2.4. Детальная схема инструмента codeRunner

```
codeRunner(title, code)
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: START                                     │
│   data-extreme_search {                                 │
│     kind: 'code',                                       │
│     codeId: nanoid(),                                   │
│     title: string,                                      │
│     code: string,                                       │
│     status: 'running'                                   │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ СОЗДАНИЕ DAYTONA SANDBOX                                │
├─────────────────────────────────────────────────────────┤
│ const sandbox = daytona.create({                        │
│   snapshot: SNAPSHOT_NAME                               │
│ })                                                      │
│                                                         │
│ // Проверка и установка библиотек                       │
│ if (code.includes('import') || code.includes('from')) { │
│   Extract missing libraries                             │
│   ↓                                                     │
│   sandbox.process.executeSync(                          │
│     `pip install ${libs.join(' ')}`                     │
│   )                                                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВЫПОЛНЕНИЕ КОДА                                         │
├─────────────────────────────────────────────────────────┤
│ response = sandbox.process.codeRun(code, {              │
│   language: 'python'                                    │
│ })                                                      │
│                                                         │
│ Returns: {                                              │
│   stdout: string,                                       │
│   stderr: string,                                       │
│   exitCode: number,                                     │
│   artifacts?: {                                         │
│     charts?: Array<{                                    │
│       type: 'chart',                                    │
│       format: 'png',                                    │
│       base64: string  // УДАЛЯЕТСЯ для экономии         │
│     }>                                                  │
│   }                                                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ CLEANUP                                                 │
├─────────────────────────────────────────────────────────┤
│ sandbox.remove()                                        │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ Stream Event: COMPLETED                                 │
│   data-extreme_search {                                 │
│     kind: 'code',                                       │
│     codeId,                                             │
│     status: 'completed',                                │
│     result: {                                           │
│       stdout, stderr, exitCode                          │
│     },                                                  │
│     charts: [{ type, format }]  // без base64           │
│   }                                                     │
│                                                         │
│ Return to Agent:                                        │
│   {                                                     │
│     output: stdout,                                     │
│     error: stderr,                                      │
│     charts: artifacts?.charts                           │
│   }                                                     │
└─────────────────────────────────────────────────────────┘
```

### 2.5. Промпты Extreme Search

**Planning Prompt** (строки 222-241):
```
You are a research planning assistant.

Break down the topic into a research plan:
- 1-5 research themes
- 3-5 specific, diverse search queries per theme
- Max 15 total actions
- Continue with more specific queries based on initial findings
- Add code tasks when data analysis is needed
- Technical and specific focus

Return format:
{
  "plan": [
    {
      "title": "Research Theme",
      "todos": ["specific query 1", "specific query 2", ...]
    }
  ]
}
```

**Research Agent System Prompt** (строки 268-341):
```
PRIMARY FOCUS: Search-based research (95% of your work)
PRIORITY: Search over code execution

Search Strategy:
1. START: Broad foundational searches
2. PROGRESS: Specific technical queries
3. CONTINUE: Recent/current developments
4. FINISH: Expert analyses/reviews

Query Guidelines:
- 5-15 words max
- Use category for news vs general
- Use includeDomains for specific sources
- X search for real-time discussions

Code Usage (minimal):
- Only when data processing needed
- After sufficient search data collected
- For calculations, visualizations, data analysis

Progressive Specificity:
Step 1-2: Broad searches
Step 3+: Targeted queries based on findings

Cross-validation:
- Compare multiple sources
- Verify key claims
- No duplicate searches
```

---

## 3. Мультизапросные пайплайны

Эти пайплайны выполняют параллельные поиски по нескольким запросам одновременно.

### 3.1. Web Search Pipeline

**Группа:** `web`
**Файл:** `/home/user/scira/lib/tools/web-search.ts`
**Провайдеры:** Parallel AI (default), Tavily, Firecrawl, Exa

#### Общая схема

```
webSearchTool(queries[], maxResults[]?, topics[]?, quality[]?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВАЛИДАЦИЯ ПАРАМЕТРОВ                                    │
├─────────────────────────────────────────────────────────┤
│ queries: string[] (3-5 обязательно)                     │
│ maxResults: number[] (default: 10 per query)            │
│ topics: ('general'|'news')[] (default: 'general')       │
│ quality: ('default'|'best')[] (default: 'default')      │
│                                                         │
│ Правило: все массивы должны совпадать по длине          │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВЫБОР СТРАТЕГИИ ПОИСКА (Strategy Pattern)               │
├─────────────────────────────────────────────────────────┤
│ const strategy = searchStrategies[provider]             │
│                                                         │
│ Доступные стратегии:                                    │
│   - ParallelSearchStrategy (default, max 5 queries)     │
│   - TavilySearchStrategy                                │
│   - FirecrawlSearchStrategy                             │
│   - ExaSearchStrategy                                   │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПАРАЛЛЕЛЬНОЕ ВЫПОЛНЕНИЕ ЗАПРОСОВ                        │
│                                                         │
│ Для каждого query параллельно:                          │
│                                                         │
│   ┌─────────────────────────────────────┐               │
│   │ Stream: START                       │               │
│   │   data-query_completion {           │               │
│   │     query, index, total,            │               │
│   │     status: 'started'               │               │
│   │   }                                 │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ ПОИСК (зависит от стратегии)       │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ ПОИСК ИЗОБРАЖЕНИЙ (параллельно)    │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ ДЕДУПЛИКАЦИЯ РЕЗУЛЬТАТОВ            │               │
│   │   - По domain + URL                 │               │
│   │   - Сортировка по relevance         │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ Stream: COMPLETED                   │               │
│   │   data-query_completion {           │               │
│   │     query, status: 'completed',     │               │
│   │     resultsCount, imagesCount       │               │
│   │   }                                 │               │
│   └─────────────────────────────────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   searches: [                                           │
│     {                                                   │
│       query: string,                                    │
│       results: [{                                       │
│         url, title, content,                            │
│         published_date, author                          │
│       }],                                               │
│       images: [{                                        │
│         url, description                                │
│       }]                                                │
│     }                                                   │
│   ]                                                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 3.2. Academic Search Pipeline

**Группа:** `academic`
**Файл:** `/home/user/scira/lib/tools/academic-search.ts`
**Провайдер:** Exa (category: 'research paper')

#### Схема пайплайна

```
academicSearchTool(queries[], maxResults[]?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВАЛИДАЦИЯ                                               │
├─────────────────────────────────────────────────────────┤
│ queries: string[] (1-5, обязательно)                    │
│ maxResults: number[] (default: 20 per query)            │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПАРАЛЛЕЛЬНОЕ ВЫПОЛНЕНИЕ                                 │
│                                                         │
│ Для каждого query параллельно:                          │
│                                                         │
│   exa.searchAndContents(query, {                        │
│     type: 'auto',                                       │
│     numResults: maxResults[i] || 20,                    │
│     category: 'research paper',                         │
│     summary: {                                          │
│       query: 'Abstract of the Paper'                    │
│     }                                                   │
│   })                                                    │
│   ↓                                                     │
│   Returns: [{                                           │
│     url, title, text,                                   │
│     summary, publishedDate,                             │
│     author                                              │
│   }]                                                    │
│   ↓                                                     │
│   ОБРАБОТКА:                                            │
│     - Дедупликация по URL                               │
│     - Очистка summary:                                  │
│       remove "Summary:" prefix                          │
│     - Очистка title:                                    │
│       remove [...] suffix                               │
│   ↓                                                     │
│   { query, results: [...] }                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   searches: [{                                          │
│     query: string,                                      │
│     results: [{                                         │
│       url, title, summary,                              │
│       publishedDate, author                             │
│     }]                                                  │
│   }]                                                    │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 3.3. Reddit Search Pipeline

**Группа:** `reddit`
**Файл:** `/home/user/scira/lib/tools/reddit-search.ts`
**Провайдер:** Tavily (domain: reddit.com)

#### Схема пайплайна

```
redditSearchTool(queries[], maxResults[]?, timeRange[]?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВАЛИДАЦИЯ                                               │
├─────────────────────────────────────────────────────────┤
│ queries: string[] (1-5, max 200 chars each)             │
│ maxResults: number[] (default: 20)                      │
│ timeRange: ('day'|'week'|'month'|'year')[]              │
│   default: week                                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПАРАЛЛЕЛЬНОЕ ВЫПОЛНЕНИЕ                                 │
│                                                         │
│ Для каждого query параллельно:                          │
│                                                         │
│   tvly.search(query, {                                  │
│     maxResults: max(20, maxResults[i]),                 │
│     timeRange: timeRange[i] || 'week',                  │
│     includeRawContent: 'markdown',                      │
│     searchDepth: 'advanced',                            │
│     chunksPerSource: 5,                                 │
│     includeDomains: ['reddit.com']                      │
│   })                                                    │
│   ↓                                                     │
│   Returns: [{                                           │
│     url, title, content,                                │
│     score, published_date                               │
│   }]                                                    │
│   ↓                                                     │
│   ОБРАБОТКА КАЖДОГО РЕЗУЛЬТАТА:                         │
│                                                         │
│     1. Извлечение subreddit из URL:                     │
│        /r/subredditname/ → "subredditname"              │
│                                                         │
│     2. Проверка isRedditPost:                           │
│        URL содержит "/comments/" → true                 │
│                                                         │
│     3. Извлечение комментариев:                         │
│        Парсинг content для блоков комментариев          │
│        Format: "username: comment text"                 │
│                                                         │
│   ↓                                                     │
│   {                                                     │
│     query,                                              │
│     results: [{                                         │
│       url, title, content,                              │
│       score, published_date,                            │
│       subreddit, isRedditPost,                          │
│       comments: string[]                                │
│     }],                                                 │
│     timeRange                                           │
│   }                                                     │
│                                                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   searches: [{                                          │
│     query, timeRange,                                   │
│     results: [{...}]                                    │
│   }]                                                    │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 3.4. X (Twitter) Search Pipeline

**Группа:** `x`
**Файл:** `/home/user/scira/lib/tools/x-search.ts`
**Провайдер:** Grok API с X Search

#### Схема пайплайна

```
xSearchTool(queries[], startDate?, endDate?,
            includeXHandles[]?, excludeXHandles[]?,
            postFavoritesCount?, postViewCount?,
            maxResults[]?)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВАЛИДАЦИЯ И ПОДГОТОВКА                                  │
├─────────────────────────────────────────────────────────┤
│ queries: string[] (1-5)                                 │
│ startDate: YYYY-MM-DD (default: 15 days ago)            │
│ endDate: YYYY-MM-DD (default: today)                    │
│ includeXHandles: string[] (max 10)                      │
│ excludeXHandles: string[] (max 10)                      │
│   ВЗАИМОИСКЛЮЧАЮЩИЕ - только один может быть задан      │
│ postFavoritesCount: number (min likes)                  │
│ postViewCount: number (min views)                       │
│ maxResults: number[] (default: 15, max 100)             │
│                                                         │
│ САНИТИЗАЦИЯ HANDLES:                                    │
│   - Удаление @ префикса                                 │
│   - Lowercase преобразование                            │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПАРАЛЛЕЛЬНОЕ ВЫПОЛНЕНИЕ                                 │
│                                                         │
│ Для каждого query параллельно:                          │
│                                                         │
│   ┌─────────────────────────────────────┐               │
│   │ GROK SEARCH                         │               │
│   │                                     │               │
│   │ generateText({                      │               │
│   │   model: 'grok-4-fast-non-reasoning'│               │
│   │   messages: [{                      │               │
│   │     role: 'user',                   │               │
│   │     content: query                  │               │
│   │   }],                               │               │
│   │   maxOutputTokens: 10,              │               │
│   │   providerOptions: {                │               │
│   │     xai: {                          │               │
│   │       searchParameters: {           │               │
│   │         mode: 'on',                 │               │
│   │         fromDate: startDate,        │               │
│   │         toDate: endDate,            │               │
│   │         maxSearchResults:           │               │
│   │           max(15, maxResults[i]),   │               │
│   │         returnCitations: true,      │               │
│   │         sources: [{                 │               │
│   │           type: 'x',                │               │
│   │           includedXHandles?,        │               │
│   │           excludedXHandles?,        │               │
│   │           postFavoriteCount?,       │               │
│   │           postViewCount?            │               │
│   │         }]                          │               │
│   │       }                             │               │
│   │     }                               │               │
│   │   }                                 │               │
│   │ })                                  │               │
│   │                                     │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ ИЗВЛЕЧЕНИЕ CITATIONS                │               │
│   │                                     │               │
│   │ experimental_providerMetadata       │               │
│   │   ?.xai?.citations                  │               │
│   │                                     │               │
│   │ [{                                  │               │
│   │   url: "https://x.com/.../..."      │               │
│   │ }]                                  │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ ПОЛУЧЕНИЕ ПОЛНЫХ ТВИТОВ             │               │
│   │                                     │               │
│   │ Для каждого citation.url:           │               │
│   │   Extract tweetId                   │               │
│   │   ↓                                 │               │
│   │   getTweet(tweetId) // Exa API      │               │
│   │   ↓                                 │               │
│   │   Returns: {                        │               │
│   │     text: "tweet content",          │               │
│   │     link: "https://x.com/...",      │               │
│   │     title: "@handle: preview..."    │               │
│   │   }                                 │               │
│   └─────────────────────────────────────┘               │
│       ↓                                                 │
│   ┌─────────────────────────────────────┐               │
│   │ РЕЗУЛЬТАТ ДЛЯ QUERY                 │               │
│   │                                     │               │
│   │ {                                   │               │
│   │   content: string,                  │               │
│   │   citations: [...],                 │               │
│   │   sources: [{                       │               │
│   │     text, link, title               │               │
│   │   }],                               │               │
│   │   query,                            │               │
│   │   dateRange: {                      │               │
│   │     fromDate, toDate                │               │
│   │   },                                │               │
│   │   handles: {                        │               │
│   │     included?, excluded?            │               │
│   │   }                                 │               │
│   │ }                                   │               │
│   └─────────────────────────────────────┘               │
│                                                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   searches: [{                                          │
│     content, citations, sources,                        │
│     query, dateRange, handles                           │
│   }],                                                   │
│   dateRange: { fromDate, toDate },                      │
│   handles: { included?, excluded? }                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

---

## 4. Специализированные пайплайны

### 4.1. YouTube Search Pipeline

**Группа:** `youtube`
**Файл:** `/home/user/scira/lib/tools/youtube-search.ts`
**Провайдер:** YouTube Data API v3

```
youtubeSearchTool(query)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОИСК ВИДЕО                                             │
├─────────────────────────────────────────────────────────┤
│ youtube.search.list({                                   │
│   part: ['snippet'],                                    │
│   q: query,                                             │
│   type: ['video'],                                      │
│   maxResults: 10,                                       │
│   relevanceLanguage: user's language                    │
│ })                                                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОЛУЧЕНИЕ ТРАНСКРИПТОВ                                  │
├─────────────────────────────────────────────────────────┤
│ Для каждого video параллельно:                          │
│                                                         │
│   YoutubeTranscript.fetchTranscript(videoId, {          │
│     lang: preferredLanguages                            │
│   })                                                    │
│   ↓                                                     │
│   Returns: [{                                           │
│     text: string,                                       │
│     offset: seconds                                     │
│   }]                                                    │
│   ↓                                                     │
│   Преобразование в segments с timestamp:                │
│   [{                                                    │
│     timestamp: "MM:SS",                                 │
│     text: string                                        │
│   }]                                                    │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   videos: [{                                            │
│     id, title, channelTitle,                            │
│     publishedAt, description,                           │
│     thumbnailUrl,                                       │
│     transcript: [{                                      │
│       timestamp, text                                   │
│     }]                                                  │
│   }]                                                    │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 4.2. Code Context Pipeline

**Группа:** `code`
**Файл:** `/home/user/scira/lib/tools/code-context.ts`
**Провайдер:** Tavily

```
codeContextTool(query)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОИСК ЧЕРЕЗ TAVILY                                      │
├─────────────────────────────────────────────────────────┤
│ tvly.search(query, {                                    │
│   topic: 'general',                                     │
│   maxResults: 10,                                       │
│   searchDepth: 'advanced',                              │
│   includeRawContent: 'markdown'                         │
│ })                                                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ОБРАБОТКА РЕЗУЛЬТАТОВ                                   │
├─────────────────────────────────────────────────────────┤
│ Фильтрация:                                             │
│   - Приоритет официальной документации                  │
│   - Фокус на примеры кода                               │
│   - Сортировка по релевантности                         │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   results: [{                                           │
│     url, title, content,                                │
│     score                                               │
│   }]                                                    │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 4.3. Memory Management Pipeline

**Группа:** `memory`
**Файл:** `/home/user/scira/lib/tools/memory-manager.ts`
**Провайдер:** Supermemory API

```
memoryManagerTool(query)
    ↓
┌─────────────────────────────────────────────────────────┐
│ ПОИСК В ПАМЯТИ                                          │
├─────────────────────────────────────────────────────────┤
│ POST /api/search                                        │
│ {                                                       │
│   query: string,                                        │
│   collection: 'user-memories',                          │
│   limit: 10                                             │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ОБРАБОТКА И РАНЖИРОВАНИЕ                                │
├─────────────────────────────────────────────────────────┤
│ - Сортировка по relevance score                         │
│ - Фильтрация по recency                                 │
│ - Группировка похожих воспоминаний                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ФИНАЛЬНЫЙ РЕЗУЛЬТАТ                                     │
├─────────────────────────────────────────────────────────┤
│ {                                                       │
│   memories: [{                                          │
│     id, content, timestamp,                             │
│     score                                               │
│   }]                                                    │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 4.4. Crypto Data Pipeline

**Группа:** `crypto`
**Файл:** `/home/user/scira/lib/tools/crypto.ts`
**Провайдер:** CoinGecko API

#### Три основных инструмента:

```
1. coin_data(coinId)
   ↓
   GET /coins/{id}
   ↓
   Returns: {
     id, symbol, name,
     market_data: {
       current_price, market_cap,
       total_volume, price_change_24h
     },
     description, links
   }

2. coin_ohlc(coinId, days)
   ↓
   GET /coins/{id}/ohlc?days={days}
   ↓
   Returns: [
     [timestamp, open, high, low, close]
   ]
   ↓
   Преобразование в candlestick формат

3. coin_data_by_contract(contractAddress, platform)
   ↓
   GET /coins/{platform}/contract/{address}
   ↓
   Returns: {
     id, symbol, name,
     contract_address, platforms,
     market_data: {...}
   }
   ↓
   Возвращает coin_id для использования с другими API
```

---

## 5. Передача данных между этапами

### 5.1. Stream-based Communication

```
┌──────────┐
│  Client  │
└────┬─────┘
     │
     │ SSE (Server-Sent Events)
     │
     ↓
┌──────────────────────────────────────┐
│  createUIMessageStream               │
│  (app/api/search/route.ts)           │
├──────────────────────────────────────┤
│  dataStream = new DataStreamWriter() │
└──────┬───────────────────────────────┘
       │
       ├─→ Message Parts:
       │   - text: "response text"
       │   - tool-call: { toolName, args }
       │   - tool-result: { result }
       │
       ├─→ Data Events:
       │   - data-query_completion
       │   - data-extreme_search
       │
       └─→ Metadata:
           - finishReason
           - usage (tokens)
```

### 5.2. Типы данных в стриме

#### data-query_completion (мультизапросные поиски)

```typescript
{
  query: string,           // Поисковый запрос
  index: number,           // Индекс в массиве запросов
  total: number,           // Общее количество запросов
  status: 'started' | 'completed' | 'error',
  resultsCount?: number,   // Количество результатов
  imagesCount?: number,    // Количество изображений
  error?: string           // Сообщение об ошибке
}
```

#### data-extreme_search (extreme search)

```typescript
// Plan events
{
  kind: 'plan',
  status: { title: string },
  plan?: Array<{
    title: string,
    todos: string[]
  }>
}

// Query events
{
  kind: 'query',
  queryId: string,
  query: string,
  status: 'started' | 'reading_content' | 'completed'
}

// Source events
{
  kind: 'source',
  queryId: string,
  source: {
    title: string,
    url: string,
    favicon?: string
  }
}

// Content events
{
  kind: 'content',
  queryId: string,
  content: {
    title: string,
    url: string,
    text: string,
    favicon?: string
  }
}

// Code execution events
{
  kind: 'code',
  codeId: string,
  title: string,
  code: string,
  status: 'running' | 'completed' | 'error',
  result?: {
    stdout: string,
    stderr: string,
    exitCode: number
  },
  charts?: Array<{
    type: 'chart',
    format: 'png'
  }>
}

// X search events
{
  kind: 'x_search',
  xSearchId: string,
  query: string,
  startDate?: string,
  endDate?: string,
  handles?: string[],
  status: 'started' | 'completed',
  result?: {
    content: string,
    citations: any[],
    sources: Array<{
      text: string,
      link: string,
      title: string
    }>
  }
}
```

### 5.3. Передача между инструментами и моделью

```
User Message
    ↓
┌─────────────────────────────────────────────────────────┐
│ streamText({                                            │
│   model,                                                │
│   messages,                                             │
│   tools,                                                │
│   activeTools,                                          │
│   onStepFinish()                                        │
│ })                                                      │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ MODEL ДУМАЕТ И ВЫЗЫВАЕТ ИНСТРУМЕНТ                      │
│                                                         │
│ toolCall = {                                            │
│   toolName: 'web_search',                               │
│   args: { queries: [...] }                              │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВЫПОЛНЕНИЕ ИНСТРУМЕНТА                                  │
│                                                         │
│ tool.execute(args) {                                    │
│   // Поиск данных                                       │
│   dataStream.write({                                    │
│     type: 'data-query_completion',                      │
│     ...                                                 │
│   })                                                    │
│   return results                                        │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ ВОЗВРАТ РЕЗУЛЬТАТА МОДЕЛИ                               │
│                                                         │
│ toolResult = {                                          │
│   toolCallId,                                           │
│   result: { ... }                                       │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ prepareStep() ПРОВЕРКА                                  │
│                                                         │
│ if (toolCalls.length > 0 && toolResults.length > 0) {   │
│   // Уже был один tool call, отключаем инструменты      │
│   return {                                              │
│     toolChoice: 'none',                                 │
│     activeTools: []                                     │
│   }                                                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ MODEL ГЕНЕРИРУЕТ ФИНАЛЬНЫЙ ОТВЕТ                        │
│                                                         │
│ - Использует результаты инструментов                    │
│ - Форматирует согласно системному промпту               │
│ - Добавляет цитирования                                 │
└─────────────────────────────────────────────────────────┘
    ↓
┌─────────────────────────────────────────────────────────┐
│ onFinish() - СОХРАНЕНИЕ В БД                            │
│                                                         │
│ - Сохранение сообщений в chat                           │
│ - Обновление usage statistics                           │
│ - Background tasks (title generation)                   │
└─────────────────────────────────────────────────────────┘
    ↓
  User
```

---

## 6. Управление группами и маршрутизация

### 6.1. Логика выбора группы

```
POST /api/search
  { group, messages, ... }
    ↓
┌─────────────────────────────────────────────────────────┐
│ ОПРЕДЕЛЕНИЕ КОНФИГУРАЦИИ ГРУППЫ                         │
├─────────────────────────────────────────────────────────┤
│ getGroupConfig(group) {                                 │
│   switch(group) {                                       │
│     case 'web':                                         │
│       return {                                          │
│         tools: webTools,                                │
│         instructions: groupInstructions.web             │
│       }                                                 │
│     case 'extreme':                                     │
│       return {                                          │
│         tools: [extremeSearchTool],                     │
│         instructions: groupInstructions.extreme         │
│       }                                                 │
│     case 'memory':                                      │
│       // Требует аутентификации                         │
│       if (!user) throw new Error('Unauthorized')        │
│       return {                                          │
│         tools: memoryTools,                             │
│         instructions: groupInstructions.memory          │
│       }                                                 │
│     case 'connectors':                                  │
│       // Требует Pro подписки                           │
│       if (!isProUser) throw new Error('Pro required')   │
│       return {                                          │
│         tools: connectorsTools,                         │
│         instructions: groupInstructions.connectors      │
│       }                                                 │
│     default:                                            │
│       // Fallback на web                                │
│       return {                                          │
│         tools: webTools,                                │
│         instructions: groupInstructions.web             │
│       }                                                 │
│   }                                                     │
│ }                                                       │
└─────────────────────────────────────────────────────────┘
```

### 6.2. Ограничения и проверки доступа

```
┌─────────────────────────────────────────────────────────┐
│ ПРОВЕРКИ ПЕРЕД ВЫПОЛНЕНИЕМ                              │
├─────────────────────────────────────────────────────────┤
│ 1. Rate Limiting (Upstash)                              │
│    - Неавторизованные: лимит по IP                      │
│    - Free users: 100 сообщений/день                     │
│    - Pro users: безлимит                                │
│                                                         │
│ 2. Extreme Search Limiting                              │
│    - Free users: лимитированное количество              │
│    - Отдельный счетчик для extreme searches             │
│                                                         │
│ 3. Feature Access                                       │
│    - Memory: требует аутентификации                     │
│    - Connectors: требует Pro подписки                   │
│    - Custom instructions: требует Pro подписки          │
│                                                         │
│ 4. Message Pruning                                      │
│    - Если messages.length > 10: обрезка истории         │
│    - Если tokens > 100k: агрессивная обрезка            │
│                                                         │
│ 5. Reasoning Support Check                              │
│    - Если модель не поддерживает reasoning:             │
│      удалить <Thought> теги из истории                  │
└─────────────────────────────────────────────────────────┘
```

### 6.3. Параллельные операции при старте

```
await Promise.all([
  // 1. Получение конфигурации группы
  getGroupConfig(group),
  
  // 2. Проверка rate limits
  checkRateLimit(userId || ipAddress),
  
  // 3. Получение пользовательских настроек
  user ? getUserSettings(userId) : null,
  
  // 4. Проверка доступа к функциям
  group === 'connectors' ? checkProAccess(userId) : null
])
```

---


## 7. Оптимизации и управление производительностью

### 7.1. Кэширование

```
┌─────────────────────────────────────────────────────────┐
│ USAGE COUNT CACHE                                       │
├─────────────────────────────────────────────────────────┤
│ // Redis cache с TTL                                    │
│                                                         │
│ const cacheKey = `usage:${userId}:${date}`              │
│                                                         │
│ // Проверка кэша перед БД запросом                      │
│ let count = await cache.get(cacheKey)                   │
│ if (!count) {                                           │
│   count = await db.getMessageCount(userId, date)        │
│   await cache.set(cacheKey, count, { ttl: 3600 })       │
│ }                                                       │
│                                                         │
│ Преимущества:                                           │
│   - Снижение нагрузки на БД                             │
│   - Быстрая проверка лимитов                            │
│   - Автоматическая инвалидация через TTL                │
└─────────────────────────────────────────────────────────┘
```

### 7.2. Background Operations

```
┌─────────────────────────────────────────────────────────┐
│ VERCEL AFTER() API                                      │
├─────────────────────────────────────────────────────────┤
│ // Выполнение после отправки ответа пользователю        │
│                                                         │
│ after(() => {                                           │
│   // 1. Генерация заголовка чата                        │
│   if (isFirstMessage) {                                 │
│     generateTitle(chatId, userMessage)                  │
│   }                                                     │
│                                                         │
│   // 2. Обновление статистики                           │
│   updateUsageStats(userId, {                            │
│     messageCount: 1,                                    │
│     extremeCount: group === 'extreme' ? 1 : 0           │
│   })                                                    │
│                                                         │
│   // 3. Аналитика                                       │
│   trackSearchEvent({                                    │
│     group, toolsUsed, responseTime                      │
│   })                                                    │
│ })                                                      │
│                                                         │
│ Преимущества:                                           │
│   - Не блокирует ответ пользователю                     │
│   - Снижает latency                                     │
│   - Асинхронная обработка задач                         │
└─────────────────────────────────────────────────────────┘
```

### 7.3. Message Pruning

```
┌─────────────────────────────────────────────────────────┐
│ СТРАТЕГИЯ ОБРЕЗКИ СООБЩЕНИЙ                             │
├─────────────────────────────────────────────────────────┤
│ function pruneMessages(messages) {                      │
│   const tokenCount = estimateTokens(messages)           │
│                                                         │
│   // Aggressive pruning                                 │
│   if (messages.length > 10 || tokenCount > 100000) {    │
│     return [                                            │
│       messages[0],  // Системное сообщение              │
│       ...messages.slice(-4)  // Последние 4 сообщения   │
│     ]                                                   │
│   }                                                     │
│                                                         │
│   // Tool call pruning                                  │
│   // Удаляем старые tool calls, оставляя только         │
│   // последние 2 сообщения с tool results               │
│   return pruneToolCalls(messages, {                     │
│     strategy: 'before-last-2-messages'                  │
│   })                                                    │
│ }                                                       │
│                                                         │
│ Цели:                                                   │
│   - Контроль расхода токенов                            │
│   - Поддержка длинных диалогов                          │
│   - Сохранение контекста                                │
└─────────────────────────────────────────────────────────┘
```

### 7.4. Parallel Tool Calls Control

```
┌─────────────────────────────────────────────────────────┐
│ ОТКЛЮЧЕНИЕ ПАРАЛЛЕЛЬНЫХ ВЫЗОВОВ                         │
├─────────────────────────────────────────────────────────┤
│ // В конфигурации моделей                               │
│                                                         │
│ const modelConfig = {                                   │
│   'scira-grok-4': {                                     │
│     parallel_tool_calls: false                          │
│   },                                                    │
│   'scira-grok-4-fast-think': {                          │
│     parallel_tool_calls: false                          │
│   }                                                     │
│ }                                                       │
│                                                         │
│ Причины:                                                │
│   - Контролируемая последовательность                   │
│   - Предсказуемый поток данных                          │
│   - Упрощение логики prepareStep()                      │
│   - Экономия ресурсов API                               │
│                                                         │
│ Исключение:                                             │
│   - Внутри extreme_search агента                        │
│     (для parallel web/x searches)                       │
└─────────────────────────────────────────────────────────┘
```

---

## 8. Обработка ошибок и восстановление

### 8.1. Tool Repair

```
┌─────────────────────────────────────────────────────────┐
│ АВТОМАТИЧЕСКОЕ ИСПРАВЛЕНИЕ ОШИБОК                       │
├─────────────────────────────────────────────────────────┤
│ experimental_repairToolCall: async ({ toolCall }) => {  │
│   // Использование Grok для исправления                 │
│   const fixed = await generateText({                    │
│     model: 'scira-grok-4-fast-non-reasoning',           │
│     messages: [{                                        │
│       role: 'user',                                     │
│       content: `Fix this tool call: ${toolCall}`        │
│     }]                                                  │
│   })                                                    │
│                                                         │
│   return fixed.toolCall                                 │
│ }                                                       │
│                                                         │
│ Примеры исправлений:                                    │
│   - Неправильный формат массива → исправление           │
│   - Отсутствующие параметры → добавление defaults       │
│   - Неверный тип данных → приведение типа               │
└─────────────────────────────────────────────────────────┘
```

### 8.2. Retry Logic

```
┌─────────────────────────────────────────────────────────┐
│ СТРАТЕГИЯ ПОВТОРОВ                                      │
├─────────────────────────────────────────────────────────┤
│ const streamConfig = {                                  │
│   maxRetries: 10,                                       │
│   abortSignal: AbortSignal.timeout(                     │
│     group === 'extreme' ? 300000 : 60000                │
│   )                                                     │
│ }                                                       │
│                                                         │
│ Таймауты:                                               │
│   - Extreme search: 5 минут                             │
│   - Обычные запросы: 1 минута                           │
│   - WebSearch API: 30 секунд                            │
│   - Tavily API: 30 секунд                               │
│                                                         │
│ Политика повторов:                                      │
│   - Exponential backoff                                 │
│   - Retry на network errors                             │
│   - Не retry на 4xx errors (кроме 429)                  │
│   - Retry на 5xx errors                                 │
└─────────────────────────────────────────────────────────┘
```

### 8.3. Fallback Strategies

```
┌─────────────────────────────────────────────────────────┐
│ ЗАПАСНЫЕ ВАРИАНТЫ                                       │
├─────────────────────────────────────────────────────────┤
│ 1. Content Retrieval Fallback:                          │
│    Exa API → Firecrawl API                              │
│    if (!content || content.length < 100) {              │
│      content = await firecrawl.scrape(url)              │
│    }                                                    │
│                                                         │
│ 2. Search Provider Fallback:                            │
│    Primary Provider → Secondary Provider                │
│    try {                                                │
│      return await primarySearch(query)                  │
│    } catch {                                            │
│      return await fallbackSearch(query)                 │
│    }                                                    │
│                                                         │
│ 3. Model Fallback:                                      │
│    Grok-4 → Grok-4-fast                                 │
│    if (grok4Error && error.code === 'overloaded') {     │
│      return useModel('scira-grok-4-fast-think')         │
│    }                                                    │
└─────────────────────────────────────────────────────────┘
```

---

## 9. Ключевые особенности архитектуры

### 9.1. Разделение ответственности

```
┌─────────────────────────────────────────────────────────┐
│ СЛОИ АРХИТЕКТУРЫ                                        │
├─────────────────────────────────────────────────────────┤
│                                                         │
│  ┌────────────────────────────────────┐                 │
│  │  API Layer (route.ts)              │                 │
│  │  - Аутентификация                  │                 │
│  │  - Rate limiting                   │                 │
│  │  - Маршрутизация по группам        │                 │
│  └────────────────────────────────────┘                 │
│             ↓                                           │
│  ┌────────────────────────────────────┐                 │
│  │  Orchestration Layer               │                 │
│  │  - streamText управление           │                 │
│  │  - prepareStep логика              │                 │
│  │  - Message pruning                 │                 │
│  └────────────────────────────────────┘                 │
│             ↓                                           │
│  ┌────────────────────────────────────┐                 │
│  │  Tools Layer                       │                 │
│  │  - Реализация инструментов         │                 │
│  │  - Stream events                   │                 │
│  │  - External API вызовы             │                 │
│  └────────────────────────────────────┘                 │
│             ↓                                           │
│  ┌────────────────────────────────────┐                 │
│  │  Provider Layer                    │                 │
│  │  - Exa, Tavily, Firecrawl          │                 │
│  │  - Grok, YouTube APIs              │                 │
│  │  - CoinGecko, Supermemory          │                 │
│  └────────────────────────────────────┘                 │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 9.2. Модели принятия решений

```
┌─────────────────────────────────────────────────────────┐
│ ТРИ ОСНОВНЫХ МОДЕЛИ                                     │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 1. scira-grok-4-fast-think                              │
│    - Основная модель для диалогов                       │
│    - С reasoning поддержкой                             │
│    - Использование: 90% запросов                        │
│    - Max tokens: не ограничено                          │
│                                                         │
│ 2. scira-grok-4-fast-non-reasoning                      │
│    - Для X Search и быстрых задач                       │
│    - Без reasoning overhead                             │
│    - Использование: X Search, tool repair               │
│    - Max tokens: 10-100 (минимальный)                   │
│                                                         │
│ 3. scira-grok-4 (full)                                  │
│    - Резерв для сложных запросов                        │
│    - С полным reasoning                                 │
│    - Использование: редко, по требованию                │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 9.3. prepareStep() - Логика управления

```
┌─────────────────────────────────────────────────────────┐
│ КОНТРОЛЬ ПОТОКА ВЫПОЛНЕНИЯ                              │
├─────────────────────────────────────────────────────────┤
│ function prepareStep({ toolCalls, toolResults }) {      │
│                                                         │
│   // ПРАВИЛО: Максимум один вызов инструмента           │
│   // После первого tool call → отключить инструменты    │
│                                                         │
│   if (toolCalls.length > 0 && toolResults.length > 0) { │
│     return {                                            │
│       toolChoice: 'none',                               │
│       activeTools: []                                   │
│     }                                                   │
│   }                                                     │
│                                                         │
│   // Инструменты доступны                               │
│   return {                                              │
│     toolChoice: 'auto',                                 │
│     activeTools: groupTools[currentGroup]               │
│   }                                                     │
│ }                                                       │
│                                                         │
│ Результат:                                              │
│   - Один tool call на запрос                            │
│   - Предсказуемый поток: query → tool → response        │
│   - Контроль расхода токенов                            │
│   - Упрощение отладки                                   │
└─────────────────────────────────────────────────────────┘
```

---

## 10. Резюме и рекомендации

### 10.1. Основные пайплайны

```
1. EXTREME SEARCH (автономное исследование)
   Planning → Research Agent → Multi-tool execution → Synthesis
   Используется для: глубокий анализ, множественные источники

2. WEB SEARCH (мультизапросный)
   Parallel queries → Multiple providers → Deduplication → Results
   Используется для: быстрый поиск, актуальная информация

3. ACADEMIC SEARCH (научный поиск)
   Parallel queries → Exa research papers → Summaries → Results
   Используется для: научные статьи, исследования

4. X SEARCH (Twitter/X поиск)
   Parallel queries → Grok search → Tweet fetching → Results
   Используется для: реалтайм обсуждения, мнения

5. SPECIALIZED (YouTube, Code, Crypto, etc.)
   Single query → Specialized API → Processing → Results
   Используется для: конкретные задачи
```

### 10.2. Паттерны проектирования

```
┌─────────────────────────────────────────────────────────┐
│ ИСПОЛЬЗУЕМЫЕ ПАТТЕРНЫ                                   │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 1. Strategy Pattern                                     │
│    - SearchStrategies (Parallel, Tavily, Exa, etc.)     │
│    - Гибкая замена провайдеров                          │
│                                                         │
│ 2. Observer Pattern                                     │
│    - DataStreamWriter для SSE                           │
│    - Real-time updates клиенту                          │
│                                                         │
│ 3. Chain of Responsibility                              │
│    - prepareStep() → tool execution → response          │
│    - Последовательная обработка                         │
│                                                         │
│ 4. Factory Pattern                                      │
│    - getGroupConfig() создание конфигураций             │
│    - Динамическое создание инструментов                 │
│                                                         │
│ 5. Decorator Pattern                                    │
│    - after() для background tasks                       │
│    - Расширение функциональности                        │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 10.3. Рекомендации по расширению

```
┌─────────────────────────────────────────────────────────┐
│ ПРИ ДОБАВЛЕНИИ НОВОГО ПАЙПЛАЙНА                         │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ 1. Создать новый инструмент в lib/tools/               │
│    - Реализовать execute() функцию                      │
│    - Добавить stream events                             │
│    - Обработать ошибки с fallback                       │
│                                                         │
│ 2. Добавить в группу инструментов                       │
│    - Обновить groupTools mapping                        │
│    - Создать системный промпт                           │
│                                                         │
│ 3. Настроить prepareStep логику                         │
│    - Определить правила вызова                          │
│    - Установить лимиты                                  │
│                                                         │
│ 4. Добавить stream events                               │
│    - Определить структуру данных                        │
│    - Реализовать прогресс индикацию                     │
│                                                         │
│ 5. Тестирование                                         │
│    - Проверить с разными запросами                      │
│    - Валидировать stream события                        │
│    - Тестировать error handling                         │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

### 10.4. Метрики производительности

```
┌─────────────────────────────────────────────────────────┐
│ ЦЕЛЕВЫЕ ПОКАЗАТЕЛИ                                      │
├─────────────────────────────────────────────────────────┤
│                                                         │
│ Web Search (обычный):                                   │
│   - Time to first byte: < 1s                            │
│   - Total response time: 5-15s                          │
│   - Parallel queries: 3-5                               │
│                                                         │
│ Extreme Search:                                         │
│   - Planning phase: 2-5s                                │
│   - Research phase: 30-120s                             │
│   - Max execution time: 5 min                           │
│                                                         │
│ Specialized tools:                                      │
│   - YouTube: 10-20s (with transcripts)                  │
│   - Code context: 5-10s                                 │
│   - Crypto data: 2-5s                                   │
│                                                         │
│ Message overhead:                                       │
│   - Pruning triggered: > 10 messages                    │
│   - Token limit: 100k                                   │
│   - Stream latency: < 100ms                             │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

## Заключение

Система Scira использует сложную многоуровневую архитектуру пайплайнов, оптимизированную для различных типов поиска и исследований. Основные принципы:

1. **Модульность**: Каждый пайплайн независим и может быть заменен
2. **Масштабируемость**: Параллельное выполнение где возможно
3. **Надежность**: Множественные fallback стратегии
4. **Производительность**: Кэширование, background tasks, stream processing
5. **Контроль**: prepareStep() управление, rate limiting, token control

**Дата создания:** 2025-11-11
**Версия документа:** 1.0
**Статус:** Актуальный

