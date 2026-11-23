# Техническое задание под ИИ-агента (Cursor / Claude Code)

> **Проект:** Backend-сервис трекера и LLM-консультанта по индивидуальной образовательной траектории в ИТ на основе графа навыков с оптимизацией производительности и расхода токенов.  
> **Основание:** Материалы ЛР 2 (`docs/research/report.md`) и ЛР 3 (`docs/design/report.md`).  
> **Назначение документа:** Самодостаточная спецификация для поэтапной генерации кода, тестов и бенчмарков ИИ-агентом без архитектурных расхождений.

---

## 1. Контекст и неизменяемые архитектурные инварианты

1. **Гибридная архитектура (Strict Separation of Concerns):** выбор следующего шага обучения ВСЕГДА выполняется детерминированным алгоритмом обхода графа навыков в PostgreSQL (`Graph Engine`), без вызова LLM. LLM используется исключительно для объяснения рассчитанных шагов и ответов на вопросы.
2. **Нулевая терпимость к галлюцинациям узлов:** любая ссылка на навык/тему в ответе консультанта должна проходить пост-валидацию по множеству допустимых `node_id` из локального подграфа. Узлы вне графа отфильтровываются, а метрика `graph_aligned_ratio` фиксируется в телеметрии.
3. **Измеримая экономия токенов и latency:** система обязана поддерживать два переключаемых режима для сравнительных бенчмарков:
   - `baseline` — передача полного графа навыков и полной истории прогресса в промпт без кэша и сжатия;
   - `optimized` — передача только минимального k-hop подграфа и краткой сводки прогресса + кэширование в Redis/БД + сжатие контекста + маршрутизация моделей.
4. **Отказоустойчивость (Graceful Degradation):** при таймауте (`LLM_TIMEOUT_SEC`) или ответе 5xx от LLM API сервис НЕ возвращает 500 клиенту, а отдаёт `200 OK` с детерминированными `next_nodes`, базовым описанием из `SKILL_NODE.description` и флагом `"fallback": true`.
5. **Безопасность (RBAC и изоляция):** все пользовательские эндпоинты требуют JWT-аутентификации; пользователь с ролью `learner` имеет доступ только к собственным данным (`user_id == request.user.id`); управление узлами графа и просмотр агрегированной телеметрии доступны только роли `admin`.

---

## 2. Технологический стек и структура проекта

- **Язык и фреймворк:** Python 3.12+, Django 5.1+, Django REST Framework (DRF), `djangorestframework-simplejwt`, `drf-spectacular` (OpenAPI 3.0).
- **Хранилища:** PostgreSQL 16 (основная БД), Redis 7 (кэш пояснений LLM и горячих подграфов).
- **HTTP-клиенты и валидация:** `httpx` (с явными таймаутами и retry для синхронизации), `pydantic` v2 для контрактов внешних API.
- **Тестирование и качество:** `pytest`, `pytest-django`, `factory-boy`, `respx` (моки внешних HTTP/LLM API), `ruff`, `mypy`.
- **Контейнеризация:** `Dockerfile`, `docker-compose.yml` (`web`, `db`, `redis`).

```text
backend/
├── config/
│   ├── settings/
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── test.py
│   ├── urls.py
│   └── wsgi.py
├── apps/
│   ├── users/               # Модель User (role, external_model_user_id), JWT auth
│   ├── skills/              # SkillNode, SkillDependency, загрузчик карт Proglib/roadmap.sh
│   ├── tracker/             # LearningGoal, UserProgress, GraphEngine (обход DAG)
│   ├── integration/         # ExternalModelAdapter, ExternalSyncLog (pull/push sync)
│   ├── consultant/          # LLMProviderAdapter (GigaChat/YandexGPT/Mock), Cache, Compressor, Router, Validator
│   └── telemetry/           # RequestTelemetryLog, сервис асинхронной записи, эндпоинты аналитики
├── benchmarks/
│   ├── synthetic_profiles.py # Генератор синтетических профилей пользователей
│   └── run_benchmark.py      # Сравнение baseline vs optimized (CSV/JSON отчёт)
├── fixtures/
│   └── proglib_webdev_graph.json # Начальный граф навыков (Proglib + roadmap.sh)
├── tests/
│   ├── test_graph_engine.py
│   ├── test_consultant_pipeline.py
│   ├── test_fallback_and_validation.py
│   ├── test_external_sync.py
│   └── test_rbac_and_api.py
├── manage.py
├── pyproject.toml
└── docker-compose.yml
```

---

## 3. Схема базы данных (Django ORM Models)

Агент обязан реализовать все 8 таблиц в точном соответствии с ER-моделью из ЛР 3:

1. **`User` (`apps/users/models.py`, наследование от `AbstractUser`):**
   - `email` (`EmailField`, `unique=True`)
   - `role` (`CharField`, choices: `learner`, `admin`, default=`learner`)
   - `external_model_user_id` (`CharField(max_length=128, null=True, blank=True, db_index=True)`)
2. **`SkillNode` (`apps/skills/models.py`):**
   - `slug` (`SlugField`, `unique=True`)
   - `title` (`CharField(max_length=255)`)
   - `description` (`TextField`)
   - `source_map` (`CharField`, choices: `proglib`, `roadmap_sh`)
   - `difficulty_level` (`PositiveSmallIntegerField`, validators: 1..5)
   - `is_active` (`BooleanField`, default=`True`)
3. **`SkillDependency` (`apps/skills/models.py`):**
   - `prerequisite_node` (`ForeignKey(SkillNode, related_name="dependent_links", on_delete=CASCADE)`)
   - `dependent_node` (`ForeignKey(SkillNode, related_name="prerequisite_links", on_delete=CASCADE)`)
   - `relation_type` (`CharField`, choices: `required`, `recommended`, default=`required`)
   - `Meta.constraints`: `UniqueConstraint(fields=["prerequisite_node", "dependent_node"])`, запрет самоссылок (`CheckConstraint`) + проверка отсутствия циклов при сохранении.
4. **`LearningGoal` (`apps/tracker/models.py`):**
   - `user` (`ForeignKey(User, on_delete=CASCADE)`)
   - `target_node` (`ForeignKey(SkillNode, on_delete=PROTECT)`)
   - `status` (`CharField`, choices: `active`, `achieved`, `paused`)
   - `created_at`, `updated_at`
5. **`UserProgress` (`apps/tracker/models.py`):**
   - `user` (`ForeignKey(User, on_delete=CASCADE)`)
   - `node` (`ForeignKey(SkillNode, on_delete=CASCADE)`)
   - `status` (`CharField`, choices: `not_started`, `in_progress`, `done`, `skipped`)
   - `mastery_score` (`FloatField`, 0.0..1.0, default=0.0)
   - `updated_at`, `synced_at` (`DateTimeField(null=True, blank=True)`)
   - `Meta.constraints`: `UniqueConstraint(fields=["user", "node"])`.
6. **`LLMCache` (`apps/consultant/models.py`):**
   - `cache_key` (`CharField(max_length=64, primary_key=True)`) — SHA-256
   - `subgraph_hash` (`CharField(max_length=64, db_index=True)`)
   - `model_name` (`CharField(max_length=64)`)
   - `compressed_prompt` (`TextField`)
   - `response_text` (`TextField`)
   - `validated_node_ids` (`JSONField(default=list)`)
   - `created_at`, `expires_at` (`DateTimeField(db_index=True)`)
7. **`RequestTelemetryLog` (`apps/telemetry/models.py`):**
   - `user` (`ForeignKey(User, on_delete=SET_NULL, null=True)`)
   - `endpoint` (`CharField(max_length=128)`)
   - `mode` (`CharField`, choices: `baseline`, `optimized`, default=`optimized`)
   - `cache_hit` (`BooleanField(default=False)`)
   - `model_used` (`CharField(max_length=64, blank=True)`)
   - `prompt_tokens` (`PositiveIntegerField(default=0)`)
   - `completion_tokens` (`PositiveIntegerField(default=0)`)
   - `latency_ms` (`PositiveIntegerField`)
   - `graph_aligned_ratio` (`FloatField(default=1.0)`)
   - `fallback_triggered` (`BooleanField(default=False)`)
   - `created_at` (`DateTimeField(auto_now_add=True, db_index=True)`)
8. **`ExternalSyncLog` (`apps/integration/models.py`):**
   - `user` (`ForeignKey(User, on_delete=CASCADE)`)
   - `sync_direction` (`CharField`, choices: `pull`, `push`)
   - `http_status` (`PositiveSmallIntegerField(null=True)`)
   - `error_message` (`TextField(blank=True)`)
   - `synced_at` (`DateTimeField(auto_now_add=True)`)

---

## 4. Контракт REST API (`/api/v1/`)

| Метод | Эндпоинт | Роль | Описание |
| :--- | :--- | :--- | :--- |
| `POST` | `/api/v1/auth/token/` | Public | Получение пары JWT-токенов (access / refresh) |
| `GET` | `/api/v1/skills/graph/` | `learner`, `admin` | Получение графа навыков (поддержка `?source_map=proglib`) |
| `POST/PATCH` | `/api/v1/skills/nodes/` | `admin` | Создание и редактирование узлов и зависимостей графа |
| `GET/POST` | `/api/v1/tracker/goals/` | `learner` | Просмотр и установка целевого узла `target_node_id` |
| `PATCH` | `/api/v1/tracker/progress/{node_id}/` | `learner` | Обновление статуса узла (`done`, `in_progress`, `skipped`) и `mastery_score` |
| `POST` | `/api/v1/integration/sync/` | `learner` | Принудительная синхронизация с внешней моделью данных |
| `GET` | `/api/v1/recommendations/next/` | `learner` | Детерминированный расчёт следующих шагов (0 обращений к LLM) |
| `POST` | `/api/v1/recommendations/explain/` | `learner` | Расчёт шага + пояснение LLM-консультанта (поддержка `mode`: `optimized` / `baseline`) |
| `GET` | `/api/v1/telemetry/summary/` | `admin` | Сводная статистика токенов, p50/p95 latency, `cache_hit_rate`, `graph_aligned_ratio` |

### Пример контракта `POST /api/v1/recommendations/explain/`

**Request:**
```json
{
  "question": "Почему мне стоит изучить Docker перед деплоем Django-приложения?",
  "mode": "optimized",
  "max_next_nodes": 3
}
```

**Response (`200 OK`):**
```json
{
  "next_nodes": [
    {"id": 14, "slug": "docker-basics", "title": "Основы Docker", "difficulty_level": 2}
  ],
  "explanation": "Перед развёртыванием Django рекомендуется освоить узел [14: Основы Docker]...",
  "graph_refs": [14],
  "meta": {
    "mode": "optimized",
    "cache_hit": false,
    "model_used": "gigachat-lite",
    "prompt_tokens": 185,
    "completion_tokens": 140,
    "latency_ms": 620,
    "graph_aligned_ratio": 1.0,
    "fallback": false
  }
}
```

---

## 5. Спецификация ключевых алгоритмов

### 5.1. Детерминированный расчёт следующего шага (`GraphEngine.get_next_steps`)

1. Загрузить активную `LearningGoal` пользователя (`target_node_id`) и все активные `SkillNode` / `SkillDependency` (одним батчевым SQL-запросом без N+1).
2. Построить обратный подграф достижимости `A(target)` — множество всех предков `target_node_id` (включая сам целевой узел).
3. Из `A(target)` исключить узлы, имеющие у пользователя статус `done` или `skipped`.
4. Отобрать «готовые к изучению» узлы (frontier): те, у которых **все** `required`-пререквизиты уже имеют статус `done` (или `skipped`).
5. Отсортировать frontier по приоритету:
   - сначала узлы со статусом `in_progress`;
   - затем узлы с минимальным `difficulty_level` и минимальной дистанцией до `target_node_id`;
   - вернуть top-`N` (`max_next_nodes`, по умолчанию 3).
6. Сформировать **минимальный локальный подграф** (`subgraph`): выбранные `next_nodes` + их непосредственные `prerequisite`-узлы (1 hop назад) + непосредственные `dependent`-узлы на пути к цели (1 hop вперёд), а также компактную сводку прогресса (`done_count / total_on_path`, текущая цель).

### 5.2. Конвейер оптимизации и валидации LLM (`ConsultantService.explain`)

1. **Нормализация интента и ключ кэша:** вычислить `subgraph_hash = sha256(canonical_json(subgraph))` и `intent_hash = sha256(normalized_question)`. Сформировать `cache_key = sha256(f"{subgraph_hash}:{intent_hash}")`.
2. **Проверка кэша (только в режиме `optimized`):**
   - `GET cache_key` из Redis; при промахе — проверка `LLMCache` в PostgreSQL (`expires_at > now()`).
   - При попадании (`cache_hit=True`) вернуть кэшированный ответ (`prompt_tokens=0`, `completion_tokens=0`).
3. **Сжатие контекста:** удалить избыточные описания из непрофильных узлов, сериализовать локальный подграф в компактный формат (`id:title(status)->[deps]`), ограничить системный промпт жёстким бюджетом токенов.
4. **Маршрутизация модели (`ModelRouter`):**
   - простые пояснения («что это», «зачем учить шаг») → легковесная/дешёвая модель (`lite`);
   - составные вопросы (сравнение технологий, план на несколько шагов) → старшая модель (`pro`).
5. **Вызов провайдера через единый интерфейс `BaseLLMProvider`:**
   - реализации: `GigaChatProvider`, `YandexGPTProvider`, `MockLLMProvider` (для CI и оффлайн-тестов).
   - таймаут `LLM_TIMEOUT_SEC` (по умолчанию 8 с).
6. **Пост-валидация (`GraphResponseValidator`):**
   - модель инструктируется ссылаться на темы в структурированном формате (JSON или теги `[node_id]`);
   - валидатор извлекает все упомянутые ссылки на навыки, проверяет их вхождение в множество `allowed_node_ids` локального подграфа;
   - вычисляет `graph_aligned_ratio = valid_refs / max(total_refs, 1)`; недопустимые ссылки удаляются/заменяются.
7. **Graceful Degradation:** при исключении `TimeoutError` / `HTTPStatusError` сформировать детерминированный ответ по шаблону на основе `SkillNode.description`, выставить `fallback_triggered=True`.
8. **Телеметрия:** сохранить запись в `RequestTelemetryLog` (не блокируя возврат HTTP-ответа при ошибке записи лога).

---

## 6. Порядок поэтапной реализации для ИИ-агента

Агент должен выполнять разработку строго инкрементально, запуская `pytest` после каждого этапа:

1. **Этап 1 (Каркас и БД):** настроить Django + DRF + PostgreSQL + Redis + Docker Compose; создать модели и миграции; добавить management-команду `load_skill_graph` для загрузки `fixtures/proglib_webdev_graph.json`.
2. **Этап 2 (Трекер и Graph Engine):** реализовать эндпоинты целей/прогресса, RBAC-права, алгоритм `GraphEngine` и покрыть граничные случаи (циклы, пропущенные узлы, выполненная цель).
3. **Этап 3 (Внешняя интеграция):** реализовать `ExternalModelAdapter` (с конфигурируемым `EXTERNAL_MODEL_BASE_URL` и mock-режимом) + запись `ExternalSyncLog`.
4. **Этап 4 (LLM-консультант и оптимизация):** реализовать `BaseLLMProvider` (GigaChat, YandexGPT, Mock), кэширование Redis+DB, сжатие подграфа, `ModelRouter`, `GraphResponseValidator` и `fallback`.
5. **Этап 5 (Телеметрия и бенчмарк):** реализовать `RequestTelemetryLog`, сводный эндпоинт `/api/v1/telemetry/summary/` и скрипт `benchmarks/run_benchmark.py`, прогоняющий ≥50 синтетических профилей в режимах `baseline` и `optimized` с выгрузкой таблицы сравнения.

---

## 7. Критерии приёмки (Definition of Done)

- Все миграции применяются на чистой PostgreSQL без ошибок; `docker compose up` поднимает рабочий стек с OpenAPI-документацией `/api/docs/`.
- Покрытие тестами `apps/tracker` и `apps/consultant` не ниже 80%; все тесты (`pytest`) проходят без доступа к реальному интернету (на `respx`/моках).
- Проверены сценарии: `Cache Hit` (0 токенов), `Cache Miss` + валидация, срабатывание `fallback` при таймауте LLM, изоляция данных между разными `learner`-пользователями (403/404), запрет редактирования графа для `learner` (403).
- Скрипт `benchmarks/run_benchmark.py` формирует воспроизводимый отчёт, подтверждающий снижение среднего числа `prompt_tokens` и медианы `latency_ms` в режиме `optimized` относительно `baseline` при `graph_aligned_ratio >= 0.95`.
