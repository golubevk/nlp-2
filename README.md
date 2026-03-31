# Банковский консультант — RAG-система

RAG-система (Retrieval-Augmented Generation) для автоматизации ответов клиентам банка «Альфа-Финанс» по продуктам: кредиты, ипотека, депозиты.

## Стек

- **LLM:** OpenRouter API (совместим с OpenAI API)
- **Эмбеддинги:** `intfloat/multilingual-e5-large` (1024-dim)
- **Векторная БД:** PostgreSQL 16 + pgvector
- **Фреймворк:** LangChain
- **Оценка:** Ragas
- **Python:** 3.13+

## Структура проекта

```
├── stage1_data_preparation.ipynb   # Этап 1: данные, чанкинг, эмбеддинги, pgvector
├── stage2_retrieval.ipynb          # Этап 2: ретриверы, гибридный поиск, метрики
├── stage3_llm_integration.ipynb    # Этап 3: LLM, RAG-цепочка, оптимизация ответов
├── stage4_analysis.ipynb           # Этап 4: Ragas-оценка, бенчмарки, оптимизация
├── rag_config.json                 # Конфиг и метрики (передаётся между этапами)
├── requirements.txt                # Зависимости
├── docker-compose.yml              # PostgreSQL + pgvector
├── initdb/01-init.sql              # Автоинициализация БД (расширение vector)
└── .env                            # OPENROUTER_API_KEY
```

## Быстрый старт

### 1. Поднять PostgreSQL с pgvector

```bash
docker compose up -d
```

Контейнер автоматически:
- создаёт базу `bank_rag`
- включает расширение `vector`

### 2. Установить зависимости

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

### 3. Настроить API-ключ

Создать файл `.env` в корне проекта:

```
OPENROUTER_API_KEY=sk-or-v1-ваш-ключ
```

### 4. Запустить ноутбуки последовательно

Ноутбуки выполняются строго по порядку — каждый следующий зависит от результатов предыдущего через `rag_config.json` и данные в pgvector.

```
stage1_data_preparation.ipynb  →  stage2_retrieval.ipynb  →  stage3_llm_integration.ipynb  →  stage4_analysis.ipynb
```


## Этапы проекта

### Этап 1. Подготовка данных и база знаний

- 5 синтетических документов банка «Альфа-Финанс» (кредиты, ипотека, депозиты, требования к заёмщикам, FAQ)
- Очистка и нормализация текстов
- 3 стратегии чанкинга с сравнением: `CharacterTextSplitter`, `NLTKTextSplitter`, `RecursiveCharacterTextSplitter` (выбран финальным)
- Эмбеддинги `intfloat/multilingual-e5-large` с обоснованием выбора
- Сохранение 31 чанка в pgvector

### Этап 2. Система ретрива

- Similarity Search (косинусное сходство)
- MMR (Maximum Marginal Relevance, λ=0.6)
- Гибридный поиск: EnsembleRetriever (BM25 50% + Vector 50%)
- Фильтрация по метаданным (`product_type`)
- Контекстное сжатие (EmbeddingsFilter, порог 0.75)
- 20 тестовых вопросов, метрики Hit Rate@1/3/5 и MRR

### Этап 3. Интеграция с LLM

- Системный промпт с 7 правилами (запрет ответов вне контекста, цитирование источников, конкретные цифры)
- `BankingRAGChain` — RAG-цепочка с историей диалога (до 6 ходов) и graceful degradation
- Multi-Query Retriever (3 перефразировки запроса)
- Self-Query Retriever (автоизвлечение фильтра `product_type` из текста)
- Реранкинг (EmbeddingReranker)
- Answer Grounding — LLM-судья оценивает соответствие ответа контексту (0–1)
- `OptimizedBankingRAGChain` — финальный пайплайн (Multi-Query + дедупликация + реранкинг + grounding)

### Этап 4. Анализ и оптимизация

- Ragas evaluation на 10 тестовых вопросах

## Подключение к БД

```
Host:     localhost
Port:     5432
Database: bank_rag
User:     postgres
Password: admin
```

Connection string: `postgresql+psycopg2://postgres:admin@localhost:5432/bank_rag`

## Сброс данных

Для полной переинициализации БД:

```bash
docker compose down -v
docker compose up -d
```
