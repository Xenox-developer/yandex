## 1. Общий обзор системы (System Overview)
### Основные роли AI:
1.  **AI-Методист (Offline/Async):** Автоматический разбор материалов, построение графа знаний, генерация диагностик и структуры курса.
2.  **AI-Тьютор (Real-time):** Персонализированная подача контента (адаптация текста, генерация примеров) и ответы на вопросы по контексту (Lecture-Free).
3.  **AI-Тренер (Real-time):** Проведение ролевых симуляций, оценка ответов пользователя по эталонам, трекинг мягких навыков (Кейс-тренажер).

---

## 2. Компоненты архитектуры (High-Level)

Система разделена на 4 функциональных контура:

1.  **Data Ingestion & Processing Pipeline** — Загрузка и векторизация контента.
2.  **Core Generation Engine** — Генерация графа, связей и метаданных.
3.  **Interaction Engine** — Движок взаимодействия со студентом в реальном времени.
4.  **Analytics & Assessment Engine** — Оценка компетенций и аналитика.

---

## 3. Детальное описание пайплайнов

### 3.1. Data Ingestion (Обработка материалов автора)
*Этап происходит при создании курса (PDF стр. 3).*

*   **Inputs:** DOCX, PDF, TXT, Ссылки (Video/Web), РПД.
*   **Pipeline:**
    1.  **Unstructured Loader:** Парсинг текста с сохранением форматирования.
    2.  **ASR Module (Whisper):** Транскрибация видео/аудио материалов в текст.
    3.  **Semantic Chunking:** Разбиение текста на смысловые блоки (с учетом заголовков) для качественного RAG.
    4.  **Embedding Generation:** Преобразование чанков в векторы (модели уровня `text-embedding-3-large`).
    5.  **Vector Store:** Сохранение векторов и метаданных (источник, страница, раздел).

### 3.2. Graph & Content Generation (Генерация курса)
*Асинхронный фоновый процесс (PDF стр. 3-4).*

*   **Graph Extractor Agent:**
    *   Анализирует материалы и РПД.
    *   Извлекает иерархию: `Курс` -> `Кластер` -> `Концепт` -> `Раздел`.
    *   Строит связи (пререквизиты) между нодами графа.
*   **Bloom Classifier:**
    *   Анализирует цели обучения и классифицирует их по Таксономии Блума.
    *   Генерирует подсказки для формулировок компетенций.
*   **Assessment Generator:**
    *   **RAT-Tests:** Генерация вопросов множественного выбора для "Диагностики понимания".
    *   **Verification:** Самопроверка (Self-reflection) модели на корректность сгенерированного ответа и дистракторов (неверных вариантов).

### 3.3. Interaction & Personalization (Взаимодействие со студентом)
*Работает в реальном времени (Inference time) (PDF стр. 6-8).*

*   **Student Profiler:**
    *   Формирует системный промпт на основе: `Стиль обучения (Колб)`, `Интересы`, `Уровень сложности`, `История`.
*   **Adaptive Content Engine (Lecture-Free):**
    *   **Text Adaptation:** LLM переписывает исходный чанк (Лонгрид) под профиль студента (упрощение языка, примеры из сферы интересов).
    *   **Audio-Lector (TTS):** Генерация голоса для режима лекции.
*   **RAG Assistant:**
    *   Контекстный чат-бот. Отвечает на вопросы *только* на основе материалов курса (Strict RAG), цитируя источники.

### 3.4. Case-Trainer (Кейс-тренажер)
*Диалоговая симуляция (PDF стр. 9-10).*

*   **Simulation Agent:**
    *   Stateful-агент, ведущий пользователя по сценарию кейса.
    *   Управляет переходами состояний диалога.
*   **The Judge (Evaluator):**
    *   Сравнивает ответ студента с "Эталонным ответом" (Semantic Similarity + LLM Logic).
    *   **Logic:**
        *   Если ответ верный -> +Score, переход дальше.
        *   Если неверный -> Генерация подсказки (Hint), повторная попытка.
        *   После 2 неудач -> Показ правильного ответа.
*   **Competency Tracker:** Обновляет карту компетенций на основе результатов диалога.

---

## 4. Стек технологий (Рекомендованный)

| Компонент | Технология | Назначение |
| :--- | :--- | :--- |
| **LLM (Complex Logic)** | GPT-4o / Claude 3.5 Sonnet | Генерация графа, оценка кейсов, сложные объяснения. |
| **LLM (Fast/Chat)** | GPT-4o-mini / Llama 3 | Быстрые ответы в чате, UI-подсказки. |
| **Embeddings** | OpenAI text-embedding-3 / E5 | Векторизация материалов для поиска. |
| **Vector DB** | Qdrant / Weaviate / pgvector | Хранение базы знаний (RAG). |
| **Orchestrator** | LangChain / LangGraph | Управление графом диалога и цепочками агентов. |
| **Backend** | Python (FastAPI) | API сервиса. |
| **ASR / TTS** | Whisper / ElevenLabs (or OpenAI) | Работа с голосом (видео -> текст, текст -> лектор). |

---

## 5. Алгоритмы оценки (Evaluation Logic)

### 5.1. Оценка в кейсах (Case-Study)
1.  **Input:** Сообщение студента.
2.  **Intent Analysis:** Является ли сообщение ответом на вопрос кейса или отвлеченным вопросом?
3.  **Comparison:**
    ```json
    {
      "student_answer": "...",
      "reference_answer": "...",
      "grading_rubric": "Checking for empathy and technical correctness"
    }
    ```
4.  **LLM Verdict:** Модель возвращает JSON с полями `is_correct` (bool), `score` (1-5), `feedback` (str).
5.  **State Update:** Система решает, двигаться дальше или дать подсказку.

### 5.2. Генерация профиля (Onboarding)
1.  **Сбор данных:** Тест Колба (20-30 вопросов) + Ввод интересов.
2.  **Clustering:** Определение психотипа (напр., "Аккомодатор").
3.  **Prompt Injection:** Добавление инструкции в системный промпт компаньона: *"Ты — наставник. Объясняй материал, используя практические эксперименты, так как студент — Аккомодатор."*

---

## 6. Модель данных (Сущности)

### Граф знаний (Graph Schema)
```json
{
  "concept_id": "UUID",
  "title": "Название темы",
  "prerequisites": ["list_of_concept_ids"],
  "content_chunks": ["chunk_id_1", "chunk_id_2"],
  "bloom_level": "Analysis",
  "diagnostics": ["test_id_1", "case_id_1"]
}
```
---

## 7. Визуализация архитектуры

```mermaid
graph TD
    %% Стили узлов
    classDef actor fill:#f9f,stroke:#333,stroke-width:2px,color:black;
    classDef process fill:#fff,stroke:#333,stroke-width:2px,color:black;
    classDef ai fill:#d4edda,stroke:#28a745,stroke-width:2px,color:black;
    classDef storage fill:#fff3cd,stroke:#ffc107,stroke-width:2px,color:black;

    %% Акторы
    Author((Автор)):::actor
    Student((Студент)):::actor

    %% Группа: Обработка данных (Offline)
    subgraph Ingestion_Pipeline [1. Data Ingestion & Generation (Offline)]
        direction TB
        Files[Файлы / РПД / Видео]
        Parser[Парсинг + Whisper ASR]:::process
        Chunker[Семантическая нарезка]:::process
        Embedder[Embedding Model]:::ai
        GraphAgent[AI: Graph Extractor]:::ai
        QuizGen[AI: Test Generator]:::ai
    end

    %% Группа: Хранение данных
    subgraph Data_Layer [Data Layer]
        VDB[(Vector DB\nQdrant)]:::storage
        SQL[(PostgreSQL\nGraph, Users, Logs)]:::storage
    end

    %% Группа: Движок взаимодействия (Online)
    subgraph Interaction_Engine [2. Interaction & Inference (Real-time)]
        Profiler[Student Profiler\n(Колб, Интересы, История)]:::process
        
        subgraph Mode_Lecture [Lecture-Free / AI-Assistant]
            Retriever[RAG Retriever]:::process
            Adapter[AI: Content Adaptor]:::ai
            TTS[Audio Lector]:::ai
        end
        
        subgraph Mode_Case [Case-Trainer Loop]
            SimAgent[AI: Simulation Agent]:::ai
            Judge[AI: Evaluator / Judge]:::ai
            Tracker[Competency Tracker]:::process
        end
    end

    %% Связи: Загрузка данных
    Author --> Files
    Files --> Parser
    Parser --> Chunker & GraphAgent & QuizGen
    Chunker --> Embedder
    Embedder -- Векторы --> VDB
    GraphAgent -- Структура графа --> SQL
    QuizGen -- Тесты/Квизы --> SQL

    %% Связи: Студент и Профиль
    Student --> Profiler
    Profiler -- Контекст персонализации --> Adapter
    Profiler -- Контекст персонализации --> SimAgent

    %% Связи: Lecture-Free (RAG)
    Adapter -- 1. Запрос контекста --> Retriever
    Retriever -- 2. Поиск чанков --> VDB
    VDB -.-> Retriever
    Retriever -- 3. Контекст --> Adapter
    Adapter -- 4. Персонализированный текст --> Student
    Adapter --> TTS
    TTS -- Аудио --> Student

    %% Связи: Case-Trainer (Loop)
    Student -- 1. Ответ --> SimAgent
    SimAgent -- 2. Анализ ответа --> Judge
    Judge -- 3. Оценка и фидбек --> SimAgent
    Judge -- 4. Метрики --> Tracker
    Tracker -- Обновление профиля --> SQL
    SimAgent -- 5. Реакция персонажа --> Student
```